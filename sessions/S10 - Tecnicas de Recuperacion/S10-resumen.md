# S10 — Técnicas de recuperación: reranking, medición, híbrida, expansión, routing, filtrado

> Resumen provisional (paso 2). Teoría de los seis artículos + lectura sobre RAG-ECSS + sesión en vivo.
> Pre-implementación: nada de lo que aquí figura como propuesta está validado en código ni medido.

---

## 1. Reranking: recall-then-rerank

**Por qué el bi-encoder ordena mal.** Comprime cada texto en un vector único, calculado sin conocer
la consulta. Dos consecuencias: el vector **promedia** (no distingue "trata de X" de "menciona X
entre diez cosas") y consulta y documento **nunca se leen juntos** — el coseno es geometría sobre
dos resúmenes. Resultado típico: los relevantes están en el top-50, pero su orden es poco fiable.

**Cross-encoder.** Concatena consulta y documento y los procesa juntos: atención cruzada entre
ambos, salida = una puntuación de relevancia del par. Más preciso en ordenación; nada
precalculable — cada par es una inferencia.

**Pipeline de dos etapas.**
- **Recall** (bi-encoder / búsqueda, amplio, barato): conjunto de candidatos, p. ej. 50. Su tamaño
  es el **techo**: *el reranking reordena, no recupera*.
- **Precision** (cross-encoder sobre los candidatos): corte final, p. ej. 5. Su tamaño lo dicta el
  **consumidor del contexto**, no el reranker.

**Local vs hospedado.**
- Local (sentence-transformers; `ms-marco-MiniLM` inglés, `bge-reranker-v2-m3` multilingüe): coste
  marginal cero, datos en casa, sin red; a cambio PyTorch en la imagen, RAM permanente, calidad media.
- Hospedado (Cohere Rerank y similares): más calidad, integración trivial; a cambio coste por
  consulta, red en el camino crítico, datos a un tercero.

**Patrones de implementación de producción.**
- Modelo cargado **una vez** (singleton de ciclo de vida); el healthcheck espera a la carga.
- **Mismo tipo de entrada y salida** → etapa opcional, activable por booleano → comparar es un experimento.
- Log de tamaños entrada/salida y modelo.
- Inferencia local fuera del event loop (`asyncio.to_thread`).

**Cuándo no.** Si el orden ya es bueno; si el problema es de recall; si el presupuesto de latencia
no da. **Señal disparadora:** *los relevantes están entre los candidatos, pero no arriba.*

**Lectura sobre RAG-ECSS.**
- El reranker **no resuelve** el anclaje del estándar nombrado (0 cláusulas en el canal vectorial =
  problema de recall).
- Variante a evaluar: **reranker como fusión** — unión de candidatos vectoriales y léxicos, el
  cross-encoder decide el orden; elimina la constante de ponderación de RRF.
- El cross-encoder solo ve el texto que se le pasa: si el texto no lleva estándar/cláusula, no
  puede saber a qué documento pertenece un chunk.
- **Longitud máxima** (512 tokens en MiniLM, truncado silencioso) choca con la cláusula como unidad
  → decidir si se reordenan chunks o cláusulas.
- Los scores de cross-encoder ms-marco son **logits sin acotar**, no comparables entre consultas:
  cualquier threshold exige calibración en el dominio. Si se adopta, el score debe viajar en
  `retrieved_context`.
- ECSS es público: el argumento de privacidad no aplica. Sí aplican imagen Docker,
  `WEB_CONCURRENCY=1` (una inferencia en el loop bloquea todo el servicio) y TTFT con streaming.
- Política de fallo del reranker (degradar a orden previo vs error) es decisión de contrato.
- Desplazamiento de dominio: MS MARCO (web) vs texto normativo — solo la medición lo resuelve.
- Alternativa no cubierta por el material: reranking con LLM (pointwise/listwise).

---

## 2. Medición artesanal: golden set, precision@k, latencia

**El problema.** "Parece que va mejor" falla por diseño: probamos con consultas fáciles,
recordamos lo memorable y comparamos contra una referencia que se mueve.

**Golden set.** 5–20 consultas **reales**, anotadas a mano con lo relevante. Tres decisiones no
técnicas:
- **Qué consultas:** frecuentes, difíciles conocidas, con términos exactos, alguna larga/desordenada.
- **Cuántas:** la representatividad pesa más que el tamaño; empezar pequeño y ampliar.
- **Quién anota:** quien tiene el juicio de dominio, con el **criterio escrito en una frase** y
  relevancia **binaria** (consistencia > expresividad).
- Versionado junto al código: cambiar el golden set cambia el significado de toda medición previa.

**Métricas.**
- **precision@k** con la *k real del sistema*; medir dos k convierte una duda de diseño en dato.
- **recall@k** como complemento si la anotación es exhaustiva.
- Métricas de orden (MRR, nDCG) quedan para la evaluación sistemática.

**Latencia.** En caliente (descartar la primera ejecución), **mediana** de 3–5 ejecuciones.

**Marco de decisión.** Tabla configuración × (precisión, latencia mediana). El coste se lee contra
el **presupuesto de latencia del producto**, no en absoluto. Zona trampa: ganancia pequeña con
coste pequeño — el coste real incluye la complejidad operativa; la tabla permite decir "no".

**Límites.** Sin potencia estadística (solo valen diferencias grandes y consistentes), sesgo del
anotador, y mide recuperación, no generación.

**Lectura sobre RAG-ECSS.**
- No existe golden set de retrieval. `scripts/query_examples.py` compara 5 arquetipos con
  expectativa descrita y un solo chunk anotado; `corpus/groundtruth/` mide el extractor de
  referencias. La conclusión de S09 "RRF arregla IDs y degrada genéricas" es cualitativa.
- **Estratificar**: el problema abierto es un trade-off entre tipos de consulta (ID / genérica /
  paráfrasis / cross-estándar / multi-turn); la media sola lo oculta.
- **Unidad de relevancia = `(estándar, cláusula)`**, no `chunk_id` (sobrevive al re-chunk
  pendiente). Choca con los números de cláusula repetidos dentro de un documento (identidad, abierto).
- **Deduplicar por cláusula** antes de calcular p@k: varios chunks de una cláusula inflan la métrica.
- Anotación exhaustiva inviable (13.695 chunks): **pooling** — anotar la unión de los top-N de las
  configuraciones comparadas. Sesgo conocido: lo que ninguna recupera no se anota; válido para
  comparar, no para recall absoluto.
- **Criterio de anotación** = decisión de producto: ¿cuenta una cláusula informativa de un handbook
  cuando la pregunta pide requisitos?
- Dos medidas por configuración para conectar con la señal del reranker: **recall@pool** (techo) y
  **p@k** (lo que llega al LLM).
- Denominador de latencia: **TTFT**, no tiempo total (streaming).
- El arnés se re-ejecutará con cada técnica de S10/S11: por la frontera del repo no es `scripts/`.

---

## 3. Búsqueda híbrida (léxica + semántica, RRF)

**Puntos ciegos complementarios.** La léxica no entiende paráfrasis; la semántica diluye lo literal
(nombres propios, siglas, normas, códigos). En dominios técnicos ambos tipos de consulta conviven.

**Full-text en PostgreSQL.** `tsvector` como columna generada (`GENERATED ALWAYS AS ... STORED`),
índice GIN, `websearch_to_tsquery` (tolera entrada libre), `ts_rank`. La configuración lingüística
determina stemming y stopwords. Argumento: no añadir un segundo almacén (Elasticsearch/OpenSearch)
hasta que el primero se quede corto. `ts_rank` no es BM25; existen extensiones con BM25.

**Fusión.** Las puntuaciones de ambos canales no son comparables y normalizarlas se descalibra con
la distribución de consultas. **RRF** usa solo posiciones: `score(d) = Σ 1/(k + rank_i(d))`, k=60
(valor robusto del paper original; no es la primera palanca). Premia el consenso. Implementación:
función pura sobre N rankings; ramas en paralelo; mismo contrato que la búsqueda vectorial.

**Dónde gana:** identificadores exactos, consultas cortas y específicas. **Dónde no aporta:**
consultas conceptuales bien parafraseadas.

**Lectura sobre RAG-ECSS** (la híbrida ya existe desde S08: `ts_rank` sobre `cite_tsv`, RRF c=60).
- RRF sin pesos da **voto igual a ambos canales**: 2º en un solo canal ≈ 1º en el otro (1/62 vs
  1/61). En consultas conceptuales el canal léxico es ruido con voto pleno — mecanismo de la
  degradación observada en el arquetipo 1.
- **`ts_rank` no usa IDF** (solo frecuencia y proximidad dentro del documento). En un corpus
  normativo con vocabulario compartido ("software", "memory", "requirements"), hipótesis fuerte de
  la degradación de genéricas. **Sin verificar.**
- **Dos clases de identificador:** término de contenido (el documento habla de X) vs **contenedor**
  (el documento *es* X). "E-ST-40C" en la pregunta suele ser lo segundo; las cláusulas de E-ST-40C
  rara vez contienen su propio ID. Verificar si la híbrida trae cláusulas *de* E-ST-40C o *que
  citan* E-ST-40C.
- Tokenización de IDs ECSS (`ECSS-E-ST-40C`, `5.3.5.1`) por el parser de Postgres: verificar con
  `ts_debug`, no suponer.
- **RRF ponderado** `w_i/(k + rank_i)`: mantiene la base posicional y permite bajar el voto léxico;
  es una constante más que calibrar.
- Ramas en paralelo: una `AsyncSession` no admite consultas concurrentes → sesión por rama. El
  cuello real del canal vectorial es el embedding de la consulta.

---

## 4. Expansión y descomposición de consultas

**Dos problemas del lado de la consulta.** Multitema → embedding promedio ("cerca de todo, cerca de
nada"); lotería de la formulación → la redacción decide los vecinos.

**Dos técnicas no intercambiables.**
- **Expansión (multi-query):** una intención, varias formulaciones → seguro contra la formulación.
- **Descomposición:** varias intenciones → una sub-consulta por tema.
- Criterio: ¿una cosa dicha de muchas maneras, o muchas cosas dichas a la vez? La técnica
  equivocada degrada.

**LLM con la correa corta.** Salida estructurada (Pydantic; límite en el esquema *y* en la
instrucción: "el esquema garantiza, la instrucción orienta"); conservar términos exactos; no
añadir requisitos; modelo más pequeño fiable, elegido por configuración.

**Fusión según el objetivo.** Expansión → consenso (RRF). Descomposición → **cobertura por tema**
(round-robin o cuota por sub-consulta), con deduplicación.

**Coste.** Una llamada al LLM antes de buscar (200 ms–1 s), tokens por consulta, N búsquedas y N×
candidatos aguas abajo. Mitigación principal: **no aplicarlo cuando no toca** (heurística de
entrada, decisión registrada en log). La medición debe incluir consultas difíciles o el veredicto
sale injustamente tibio.

**Lectura sobre RAG-ECSS.**
- Distinto del anclaje "expansión de cláusula" (expansión de **contexto**), que ningún artículo cubre.
- Nuestro perfil: preguntas cortas en inglés. La heurística por longitud no discrimina; el caso
  real de descomposición son las **preguntas comparativas** ("difference between B and C", "E-40 vs
  Q-80"), con señal léxica.
- La query de retrieval de S09 (pregunta + 2 previas concatenadas) **fabrica** el embedding
  multitema ante un cambio de tema. El golden set necesita un caso multi-turn con cambio de tema
  para poder observarlo (desbloquea el diferido de reescritura de seguimientos).
- **Riesgo de dominio:** el LLM conoce ECSS y puede escribir identificadores plausibles e inventados
  en las variantes; el canal léxico los casaría y RRF los subiría. Regla necesaria: ningún
  identificador ausente de la pregunta original.
- **HyDE** (no cubierto): busca con una respuesta hipotética; ataca la asimetría de registro
  pregunta vs "The supplier shall…", con riesgo de alucinación por diseño.
- Composición no resuelta por el material: N consultas × 2 canales (RRF plano destruye la
  cobertura); reranking contra la pregunta original reintroduce el promedio.
- Fallo de la reformulación → seguir con la consulta original (mejora opcional); la llamada pasa por
  el router multi-proveedor. Caché de reformulaciones: misma regla que la caché de respuestas.
- Prioridad provisional baja: ningún anclaje de S09 apunta aquí.

---

## 5. Multi-índice y routing

**Degradación del índice único con familias mezcladas.** Texturas semánticas distintas (lo verboso
queda cerca de todo), el tipo dominante inunda el top-k por volumen, cada familia quiere su
preprocesamiento, y todo reindexado es global.

**Particionado en PostgreSQL.**
- **A — tabla única + columna discriminadora:** familias que son variaciones de lo mismo.
- **B — tabla por familia:** esquemas de metadatos divergentes. Señal: columnas que solo tienen
  sentido para una familia (NULL en el resto) — *los metadatos confiesan el diseño*.

**Escalera de routing por coste.**
- **Nivel 0 — sin router:** el destino va en el contrato de la API (parámetro o endpoint); el
  llamante sabe lo que el servicio tendría que adivinar.
- **Nivel 1 — reglas deterministas:** patrones inequívocos; gratis, transparentes, frágiles.
- **Nivel 2 — LLM clasificador:** `StrEnum` cerrado (no puede inventar destinos), salida como
  **lista** de destinos (la duda = buscar en varios, no un umbral de confianza), campo `reason`
  para auditoría.
- **Fallback:** buscar en todo; nunca peor que el índice único.

**Combinar colecciones.** Nunca por puntuación cruda; por posiciones o cuotas. A menudo, no
fusionar: agrupar por procedencia. La procedencia viaja siempre con el chunk.

**Cuándo no particionar.** Corpus funcionalmente homogéneo; una colección con ~95 % de las
consultas; "para cuando crezcamos". Señal legítima: contaminación entre familias recurrente y medible.

**Lectura sobre RAG-ECSS.**
- El caso equivalente existe: **handbooks (E-HB-40A, informativo) por delante de requisitos
  normativos** en el arquetipo 1. Pero HB y ST comparten esquema → por la regla del propio
  artículo, **opción A**, que ya existe (`authority`, tipo de chunk). Partición física = sobreingeniería.
- El HB no es ruido sino **apoyo de menor autoridad**: routing duro excluiría información legítima.
- **Filtrar vs preferir:** una partición dura rompe las referencias cruzadas (pregunta sobre E-40
  que necesita la cláusula de Q-80 a la que remite). Alternativa: **cuota** (el estándar nombrado
  ocupa la mayoría del top-k; el resto queda abierto).
- Nivel 1 = detección del estándar nombrado por regex; `corpus/identity.py` ya concentra "todas las
  formas de nombrar un documento" — normalizador existente, falta usarlo en el lado de la consulta.
- Nivel 0 = decisión de producto no planteada: selector de alcance (estándares, solo normativo) en
  la interfaz.
- Nivel 2, si alguna vez hace falta: enum cerrado de los 26 documentos (versión segura del uso de
  LLM frente a la expansión).
- Verificar si algún documento grande inunda por volumen (cuenta de chunks por documento).

---

## 6. Filtrado contextual y temporal; orden del pipeline

**Filtros duros.** `WHERE` sobre metadatos antes de la similitud. **Trampa con índices
aproximados:** HNSW filtra *después* de navegar → con filtros selectivos devuelve pocos o ningún
resultado sin error. Mitigaciones: `hnsw.iterative_scan` (pgvector ≥ 0.8), índice parcial.
Hábito: **loguear la cardinalidad tras filtrar**.

**Calidad del metadato = techo.** Extracción una vez, en ingesta (reglas o LLM estructurado). Un
filtro duro sobre un metadato malo excluye con confianza al mejor candidato sin dejar rastro:
**filtro duro solo sobre metadatos fiables; lo dudoso, como mucho, pondera.**

**El tiempo.**
- **Ventana dura:** razón categórica (política, cumplimiento, cambio de era); borde brutal.
- **Decaimiento exponencial:** erosión gradual; parámetro = semivida, con lectura de negocio.
- Combinables: ventana generosa + decaimiento dentro.

**Ponderación dinámica.** Multiplicadores contextuales en configuración. Zona de máximo riesgo de
sobreingeniería: cada peso es un número mágico. Progresión: filtros y tiempo primero; boosts solo
con evidencia medida y documentados.

**Orden del pipeline:** *lo barato y excluyente al principio; lo caro y fino al final; lo blando al
cierre.* Reformulación/routing → filtros duros (dentro de la consulta) → búsqueda (dos ramas) +
fusión → reranking → ponderaciones blandas. Invertirlo: rerankear lo que un filtro iba a tirar, o
ponderar tan pronto que se expulsan candidatos antes del reranker. Cada etapa activable y observable.

**Lectura sobre RAG-ECSS.**
- Régimen exacto sin índice vectorial → `WHERE` correcto; la trampa de HNSW no aplica hoy. Si se
  añade índice, el filtro por estándar vuelve a ser riesgo (anotar en `ARCHITECTURE.md`).
- **Estándar del chunk** (derivado determinista por `identity.py`) → apto para filtro duro.
  **Autoridad** (reglas del parser; caso abierto del requisito en anexo informativo) → solo ponderación.
- En ECSS el tiempo es **supersesión de versiones**, no erosión: una revisión es aplicable o no
  (aplicabilidad contractual) → filtro categórico, probablemente nivel 0. Verificar si el corpus
  contiene varias revisiones del mismo estándar; si no, el eje no aplica hoy.
- Multiplicadores sobre logits de cross-encoder: los logits pueden ser negativos → un boost
  multiplicativo penaliza. Exige puntuaciones no negativas (sigmoide) o ponderar sobre RRF.
- El pipeline del material omite la **expansión de contexto**: va tras el último corte y antes de
  la generación, y compite por el presupuesto de tokens con el top-k.

---

## 7. Anclajes de S09 frente a las técnicas de S10

| Anclaje S09 | Técnica candidata | Estado |
|---|---|---|
| (prerrequisito) | Golden set estratificado + p@k / recall@pool + latencia (TTFT) | Candidato a primer brief |
| Query que nombra estándar → 0 cláusulas | Routing nivel 1 (regex + `identity.py`) → filtro duro y/o cuota | Abierto: filtro vs cuota |
| RRF degrada genéricas | RRF ponderado / BM25 en lugar de `ts_rank` / reranker como fusión | Abierto; verificar antes hipótesis IDF |
| Rol del reranker / threshold por chunk | Cross-encoder si recall@pool alto y p@k bajo | Abierto; depende de la medición |
| Expansión de cláusula acotada | No cubierta por el material; etapa de aumento tras el corte final | Abierto; diseño propio |
| HB por delante de ST | Ponderación por autoridad (no filtro) | Abierto; depende del criterio de anotación |
| Contaminación multi-turn | Descomposición / reescritura | Diferido; el golden set debe poder observarla |
| Grafo de referencias; tablas D-1 | — | Fuera de S10 |

---

## 8. Decisiones S10

### Propuestas (pendientes de confirmar antes del brief)
- **Golden set de retrieval como primer brief**, antes de tocar el pipeline: consultas reales
  estratificadas, unidad `(estándar, cláusula)`, criterio escrito, pooling, deduplicación por cláusula.
- **Sin partición física** del corpus: discriminación por metadatos existentes (opción A).
- **Estándar nombrado en la query = routing determinista**, no un problema de fusión.
- **Autoridad normativa como ponderación, nunca como filtro.**
- El arnés de medición vive fuera de `scripts/` (se re-ejecuta con cada técnica).

### Abiertas
- Criterio de anotación: ¿cuenta lo informativo cuando la pregunta pide requisitos?
- Filtro duro vs cuota para el estándar nombrado.
- Qué ataca la degradación de genéricas: RRF ponderado, BM25 o reranker como fusión.
- Reranker: unidad (chunk vs cláusula), local vs hospedado, política de fallo, score en la traza.
- Expansión de cláusula: qué se arrastra y con qué tope.
- Selector de alcance en la interfaz (nivel 0).
- Quién anota y cómo evitar el anclaje si un LLM pre-anota.

### Diferidas
- Expansión/descomposición de consultas y HyDE — sin anclaje que lo pida; reabrir si el golden set
  muestra fallos de vocabulario/registro o contaminación multi-turn.
- Eje temporal / versiones — si el corpus incorpora revisiones múltiples.
- Agrupar el contexto por procedencia (ST/HB) — S11.
- Normalización de tablas a Markdown/HTML (Docling, MinerU) — candidato para el re-chunk agrupado
  (`corpus/` congelado), no S10.

---

## 9. Sesión en vivo

**Idioma (Q1).** Los embeddings multilingües acercan "perro"/"dog", pero no son idénticos (demo:
0,55 EN vs 0,59 ES con corte 0,6; en ES se perdieron chunks). Requisito: encoder multilingüe.
**Regla crítica: canal semántico y léxico apuntan al mismo idioma**; cambiar uno obliga a cambiar
el otro. El riesgo real es mezclar idiomas dentro de una consulta.
→ RAG-ECSS: producto solo en inglés (decisión S09) y `tsvector` en inglés — alineado. Si algún día
se aceptan preguntas en español, el canal léxico se rompe antes que el vectorial.

**Imágenes (Q2, Q3).** La imagen no se embebe como imagen: un LLM extrae una **descripción breve**
en ingesta, que es lo que se chunkea y embebe; una **descripción extendida** se genera una vez bajo
demanda y se persiste (dos niveles). El prompt de extracción es específico del producto y filtra el
ruido irrelevante para el caso de uso.
→ RAG-ECSS: figuras de los estándares fuera de alcance hoy; patrón a recordar si entran.

**Tablas (Q4).** No pasar PDF directo al RAG: normalizar a Markdown/HTML preservando estructura.
**MinerU** (OpenDataLab; PDF/imagen → Markdown/JSON con tablas y layout; dockerizado; GPU para
volumen; requiere troceado posterior) y **Docling** (IBM Research; PDF/DOCX/PPTX → Markdown/JSON
con tablas y orden de lectura, orientado a RAG).
→ RAG-ECSS: corpus en DOCX parseado con python-docx. Relevante para las tablas de filas casi
idénticas (D-1) y las 189 tablas cuya anotación pierde alcance al aplanar; evaluar contra el parser
propio cuando se abra el re-chunk agrupado.

**Finalidad (Q5).** RAG = el LLM deja de inventar y además **puede marcar qué tiene respaldo y qué
es generado** (ejemplo: 108 tareas, 35 con dato histórico, 3 ya presupuestadas; el resto marcado
para revisión humana). Es RAG + estructura de salida. Dominios grandes: **generar por silos**, no
todo de golpe.
→ RAG-ECSS: conecta con el contrato de salida tipado y la política ante afirmaciones sin cita (S11).
