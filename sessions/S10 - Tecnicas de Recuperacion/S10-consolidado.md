# S10 — Consolidado de sesión (histórico, no se reescribe)

> **Paso 5 del ciclo.** Combina el resumen provisional (`S10-resumen.md`, paso 2) con lo aprendido al
> implementar los cuatro briefs (`S10-decisions*.md`, `S10-uat*.md`). La **divergencia teoría↔medida**
> se conserva a propósito: es el material de Guide. La graduación a `RAG-GUIDE.md` (paso 6) se hace
> **desde** este documento.
>
> **Sesión:** M4 / S10 — Técnicas de recuperación (reranking, medición, híbrida, expansión, routing,
> filtrado). **Cerrada con:** golden set de retrieval v2 (14 consultas, 642 juicios) y arnés
> re-ejecutable; defaults nuevos — `vector` + routing por estándar nombrado (`filter`) + léxico
> `tokenized_length` + expansión `orphans`; reranker medido y descartado para producción.
> `uv run pytest` 1179 verde (1017 al empezar).

---

## 1. El arco de S10

S09 dejó cinco anclajes, todos formulados como técnicas a aplicar: ponderar la fusión híbrida, filtrar
por ID de estándar, expandir la cláusula, decidir el rol del reranker, mirar tablas. La revisión
teórica cambió la pregunta antes que la respuesta: **ninguno de esos anclajes se podía decidir sin una
vara**, y la conclusión de S09 que los motivaba ("RRF arregla IDs y degrada genéricas") salía de cinco
arquetipos leídos a ojo.

1. **Brief 1 — la vara.** Golden set de retrieval anotado por una sesión de Opus aislada del repo
   (independiente del retriever, sin coste de API), arnés que mide lo que llega al LLM, y cuatro
   verificaciones baratas. Las verificaciones reformularon el problema entero: el estándar nombrado es
   un **contenedor**, el canal léxico tenía un **defecto** de tokenizado y `ts_rank` **no tiene IDF**.
2. **Brief 2 — arreglar antes de ponderar.** Tokenizado por Postgres, BM25 propio, `ts_rank`
   normalizado, routing determinista (filtro y cuota). El routing fue la palanca grande (p@k 0,68 →
   0,82); la híbrida arreglada empata con `vector`; RRF ponderado deja de hacer falta.
3. **Brief 3 — cerrar huecos.** Defaults aplicados, golden set v2 con el estrato que mide el riesgo
   del filtro (`names_standard_xref`), curva de profundidad del pool y expansión de cláusula medida
   con métricas propias.
4. **Brief 4 — el reranker, con datos.** Cross-encoder local en el arnés: empata en precisión, cuesta
   de 1 a 4 TTFT en CPU, su score no sirve como threshold. No entra. Deja dos hallazgos más valiosos
   que él: la cabecera de contexto y chunks que exceden la ventana.

---

## 2. Decisiones contrastadas con la implementación

### Confirmadas

- **Medir antes de decidir** (art. 2): todas las decisiones de la sesión se tomaron contra una tabla
  por estrato. Sin ella, el brief 2 habría ponderado RRF sobre un canal roto.
- **Tabla por estrato, no media** (resumen §2): la híbrida arreglada empata en la media (0,69 vs
  0,68) intercambiando estratos; la cuota pierde en `names_standard` lo que gana en `xref`. La media
  sola habría escondido ambas cosas.
- **El estándar nombrado es un contenedor → routing determinista** (arts. 3, 5, 6): la híbrida traía
  **0** cláusulas de E-ST-40C y 3 que lo citaban. Detector nivel 1 + filtro: `names_standard` 0,41 →
  0,74, `multi_turn` 0,25 → 1,00, y **más rápido** (el filtro escanea menos).
- **`ts_rank` no tiene IDF** (resumen §3, hipótesis): confirmada. El término discriminante
  (`dynam`, df 0,8 %) aparecía en 1 de 10 hits léxicos; ganaban prosa de handbook y tablas DRD que
  repiten `requir` hasta 20 veces.
- **Autoridad como metadato, no como relevancia** (decisión de producto): lo informativo útil cuenta;
  la autoridad se cruza al medir (`R@k normativo`) sin anotarla.
- **Unidad de relevancia = cláusula; deduplicar por cláusula** (resumen §2): adoptado en el arnés.
- **Reranker: la señal correcta es "relevantes en el pool pero no arriba"** (art. 1): se midió en dos
  partes (profundidad del pool; precisión tras reordenar) y la segunda no se cumplió.
- **Cross-encoder ve solo el texto que se le pasa** (resumen §1): sin cabecera, el reranker es **peor
  que el vector** (−0,12 a −0,17 de p@k).
- **Truncado silencioso a 512 tokens** (resumen §1): medido, 7,7–8,3 % de los pares.
- **Trampa de filtro con HNSW no aplica en régimen exacto** (art. 6): confirmado; anotado en
  `ARCHITECTURE.md` para el día que haya índice.

### Refinadas

- **Golden set: de juez LLM por API a anotación externa por protocolo.** El brief 1 preveía un juez
  fijado por API + calibración humana a ciegas. Se sustituyó por una sesión de Opus por suscripción:
  sin coste marginal, **independiente del retriever** (busca en el corpus, no anota un pool) y
  repetible **por protocolo versionado**, no por versión de modelo. La calibración humana se descartó
  porque el anotador humano no tiene dominio ECSS: la revisión humana quedó limitada a lo que el
  validador de citas literales marque.
- **El anotador, aislado del repo** (lo detectó CC): una sesión de Claude Code abierta en el repo
  carga `CLAUDE.md`, que describe el retriever, sus defaults y sus fallos — rompe la independencia
  antes de empezar. Workspace externo con solo protocolo, índice y export.
- **Pooling sustituido, no eliminado:** la fase 2 da los relevantes independientes del retriever; el
  modo B juzga lo que las configuraciones traen sin juicio. El modo B encontró relevantes que la
  fase 2 no vio en todas las rondas (23 %, 8 %, 7 %, 9 %): el anotador tampoco es exhaustivo.
- **Golden set de tamaño fijo → versionado:** 10 fijas al principio; v2 añadió 4 donde había huecos
  (estrato nuevo `names_standard_xref`, segundo `paraphrase` y `multi_turn`), conservando la v1
  intacta para comparar.
- **Recall con techo** (brief 2): con 16–55 relevantes por consulta y k = 4–10 cláusulas, `R@k cap`
  resultó idéntico a p@k; la columna informativa fue `R@cand cap`.
- **"Ponderar la fusión" → "arreglar el canal y re-medir":** el anclaje de S09 estaba mal formulado.
- **Expansión de cláusula:** el anclaje se partió en dos. `orphans` resuelve que la nota llegue sin su
  requisito (6 → 0, sin coste); pero 4 de las 6 notas huérfanas **remiten a otra cláusula**, y eso no
  es expansión de cláusula sino seguir referencias (UAT del brief 3).

### Refutadas o corregidas por la medida

- **"RRF sin pesos degrada las genéricas → hay que ponderar"** (S09): la causa no era RRF, eran el
  pre-tokenizado de `lexical_terms` (partía `ECSS-E-ST-40C` en `ecss│e│st│40c` antes de llegar a
  Postgres) y `ts_rank` sin IDF. Arreglado el canal, ponderar no hace falta.
- **"El tokenizado léxico es la causa de que fallen los IDs"** (resumen §3): era un defecto real,
  pero arreglarlo solo no movió la híbrida (0,64 → 0,63). El ranking (E3) y el routing (E4) eran la
  palanca.
- **"El problema dominante es de recall"** (brief 1): en parte era un tope estructural de la métrica.
  Con techo y con profundidad, la lectura cambió: lo que falta **está más abajo** (R@cand cap 0,46 a
  50 chunks → 0,70 a 400), no ausente.
- **"Con margen en el pool, un reranker tendrá algo que rescatar"** (decisions-3): necesaria, no
  suficiente. El pool a 400 contiene el 71 % de lo relevante, y el reranker no lo convierte en
  precisión (p@k 0,70 → 0,71 de 50 a 400).
- **"El reranker sería el primer score calibrable por par → threshold"** (anclaje S09): AUC global
  0,50–0,63, mediana por consulta < 0,5 en 5 de 6 configuraciones. Sin base con este modelo.
- **"La cuota protege las referencias cruzadas"** (resumen §5): las protege (xref 0,39 → 0,49) pero
  pierde más en `names_standard` (0,74 → 0,46). No gana en ningún balance.
- **Contador de tokens ~4 chars/token:** sobreestima en prosa (mediana 0,82) e infraestima hasta ×2,3
  en tablas. 287 chunks superan 512 tokens reales; 204 superan incluso el estimado sin estar marcados
  `oversized`.

---

## 3. Números medidos

**Golden set:** v2, 14 consultas (2 por estrato en 7 estratos), **642 juicios, 466 relevantes**,
lista de revisión vacía, anotado por `claude-opus-5-5` con protocolo 1.1. Export: 4.433 cláusulas, 26
documentos. Un solo anotador, sin drift check.

**Configuraciones (p@k global, golden set v2):**

| Configuración | p@k | R@cand (pool 50) | Latencia retrieval |
|---|---|---|---|
| `vector` | 0,60 | — | ~300 ms |
| **`vector_filter` (default)** | **0,69** | 0,42 | ~300 ms |
| `vector_quota` | 0,65 | — | ~330 ms |
| `hybrid_tok_bm25_filter` | 0,68 | 0,44 | ~440 ms |
| `rerank50_header` | 0,70 | 0,42 | +1,5 s |
| `rerank400_header` | 0,71 | 0,71 | +7 s |

**Por estrato** (v1, 10 consultas, `vector` → `vector_filter`): `names_standard` 0,41 → 0,74,
`multi_turn` 0,25 → 1,00, resto idéntico. **xref** (v2): `vector` 0,49 · filtro 0,39 · cuota 0,49.

**Canal léxico:** df de `ecss` 9,6 %, `st` 7,5 %, `e` 6,1 %; compuesto `ecss-e-st-40c` 0,7 %.
Estadísticas BM25: 10.550 lexemas, longitud media 41,5.

**Corpus:** `ECSS-E-ST-70-41C` = **35,2 %** de los chunks (vs 7,9 % E-ST-40C); inundaba la pregunta
sobre memoria dinámica en E-ST-40C (6 de 9 cláusulas). Sin revisiones duplicadas.

**Profundidad** (`vector_filter`, R@cand cap): 50 → 0,46 · 100 → 0,52 · 200 → 0,62 · 400 → 0,70; sin
coste de latencia (régimen exacto). Planos: `names_standard_xref` (0,17 → 0,20) y `paraphrase` (0,26
→ 0,49).

**Expansión** (106 cláusulas entregadas): `off` completitud normativa 0,59, 6 huérfanas, 2.340 tokens
mediana · `orphans` 0,68, 0 huérfanas, **mismos tokens** · `bounded` 0,95, +36 % tokens; los topes
cortan anexos DRD.

**Reranker** (CPU, 12.ª gen., sin GPU): 1,5 s (pool 50) a 7 s (400); carga 28–51 s; +400 MB RAM. TTFT
de referencia 1,76 s mediana. Truncado 7,7–8,3 %. AUC 0,50–0,63. Cabecera ≈ +0,15 de p@k.

**Suite:** 1017 → 1179.

---

## 4. Aprendizajes agnósticos (candidatos a Guide)

1. **Construye la vara antes de la primera técnica.** Un golden set pequeño, estratificado y anotado
   con criterio escrito convierte cada técnica en un experimento de una línea. Sin él, S09 llegó a
   una conclusión ("pondera RRF") cuya causa era otra.
2. **Verificaciones baratas antes de la técnica cara.** Cuatro comprobaciones de minutos (qué
   recupera realmente el canal, cómo tokeniza, si hay IDF, cómo se reparte el corpus) cambiaron el
   brief siguiente entero. Mirar el mecanismo es más barato que medir la solución.
3. **Distingue identificador-contenido de identificador-contenedor.** Si el término nombra el
   documento al que pertenece el texto, no es un término de búsqueda: es un filtro de metadatos.
   Ninguna búsqueda léxica lo arregla, porque el documento rara vez se nombra a sí mismo.
4. **Routing determinista primero.** Detectar con reglas lo que el usuario ya dijo es gratis,
   auditable y fue la mayor mejora de la sesión. El LLM clasificador queda para lo que las reglas no
   resuelven.
5. **Filtro duro: corta las referencias cruzadas.** Medido como un estrato propio. La cuota no es la
   respuesta (pierde más de lo que salva); el remedio estructural es ampliar el filtro con lo que el
   documento referencia.
6. **"Ponderar la fusión" puede ser un síntoma, no una causa.** Antes de afinar pesos de RRF, verifica
   que cada canal funciona solo: tokenizado simétrico query/documento y un ranking léxico con IDF.
7. **`ts_rank` no es BM25 y la diferencia no es ruido en corpus de vocabulario compartido.** Sin IDF,
   gana la repetición de términos comunes en textos largos. IDF + longitud lo arreglan; la polisemia
   (mismo término, otro tema) no la arregla nadie en el canal léxico.
8. **El reranker se justifica con dos medidas, no una:** margen en el pool (profundidad) **y**
   capacidad del modelo para convertirlo en precisión. La primera sin la segunda es latencia comprada
   a cambio de nada.
9. **El reranker (y cualquier modelo que puntúe pares) solo sabe lo que le pasas.** La cabecera
   documento · sección · título fue la variable más importante del experimento (≈ +0,15 de p@k).
   Corolario no probado: el embedding del chunk probablemente también la necesita.
10. **Un score de cross-encoder no es un threshold.** Logits sin calibrar, separación pobre frente a
    negativos difíciles. Si se quiere umbral, se mide la AUC dentro y entre consultas antes de fijarlo.
11. **Mide la latencia contra el TTFT, no en absoluto.** 1,5 s parece poco hasta que es otro TTFT
    entero delante del primer token.
12. **Un anotador LLM no es independiente si hereda la descripción del sistema.** Las instrucciones de
    proyecto (CLAUDE.md, memoria) se cargan solas; aísla el workspace.
13. **Repetibilidad por protocolo:** con un anotador LLM por suscripción, la vara estable es el
    protocolo versionado + cita literal verificada por código, no la versión del modelo.
14. **Verificación de cita literal como sustituto de dominio.** Si quien revisa no domina el campo,
    restringe su trabajo a lectura: ¿la cita existe y responde? El código valida la existencia.
15. **El anotador también se deja relevantes.** Un modo incremental que juzga lo que las
    configuraciones traen sin juicio es obligatorio; sin él, lo no juzgado penaliza a quien encuentra
    algo nuevo.
16. **Expansión de contexto ≠ selección:** se mide con métricas propias (completitud, huérfanas,
    tokens) porque no cambia qué unidades llegan. Y "completar la unidad" ≠ "seguir sus referencias".
17. **Un UAT que no discrimina también informa:** si la unidad expandida no se cita, la respuesta no
    puede cambiar. Diseña la pregunta de UAT para que dependa de lo que se prueba.
18. **El volumen de un documento puede inundar el top-k** (35 % del corpus en uno). El filtro lo
    resuelve para preguntas que nombran; para el resto, vigilar.

---

## 5. Aprendizajes de dominio (ECSS)

- Las cláusulas de un estándar no contienen su propio identificador; los que lo nombran son handbooks
  y estándares que lo citan.
- `ECSS-E-ST-70-41C` (PUS) domina el corpus en volumen y semánticamente compite con todo lo que hable
  de memoria, telemetría o servicios; BM25 lo sube por coincidencia léxica exacta en "memory
  management" (servicio PUS) cuando la pregunta es de software de vuelo.
- Los anexos DRD (tablas largas con "requirements" repetido) son el peor enemigo del ranking léxico
  sin normalización y los que más tokens cuestan al expandir.
- Notas informativas que remiten a otra cláusula ("See also… clause 5.2.5") son frecuentes: 4 de 6
  huérfanas.
- Preguntas que nombran un estándar a menudo necesitan otro: en q11, 17 de 34 relevantes eran de
  `E-ST-10-06C`; en q12, 17 de 23 de `M-ST-80C`.
- La paráfrasis (lenguaje cotidiano frente a vocabulario ECSS) es el estrato más bajo a cualquier
  profundidad de pool: límite de representación.
- Formas de citar un estándar que el detector reconoce: con y sin `ECSS-`, con letra de versión, con
  revisión, `Part 1A`. Límite: `Q-HB-80-02` sin `Part` no casa con ninguna de sus partes.

---

## 6. Abierto, con anclaje

| Tema | Señal medida | Condición para reabrir | Dónde |
|---|---|---|---|
| **Filtro ampliado por referencias / expansión por grafo** | xref: filtro 0,49 → 0,39; 4 de 6 notas huérfanas remiten a otra cláusula (UAT brief 3) | Validar el grafo (gate en FAIL, 1/50) | Sesión posterior |
| **Cabecera de contexto en el embedding** | Cabecera ≈ +0,15 en el reranker | Grupo re-chunk + re-embed | Con `corpus/` |
| **Chunks sobre la ventana** | 287 > 512 reales; 272 tablas sin partir; 204 sin marcar `oversized` | Mismo grupo + tokenizador real | Con `corpus/` |
| **Duplicados del parser** ("Expected output…" ×2) | UAT brief 3 | Mismo grupo | Con `corpus/` |
| **Reranker** | Empata 0,71 vs 0,69; +1,5–7 s en CPU; margen de pool 0,42 → 0,71 | Modelo más fuerte, GPU o hospedado; mismo arnés | Cuando haya presupuesto |
| **Threshold por chunk** | AUC 0,50–0,63 con MiniLM | Otro modelo con AUC medida | Con el reranker |
| **Expansión de consulta / HyDE** | `paraphrase` 0,49 a 400 chunks | Más consultas de paráfrasis que confirmen | — |
| **`bounded` como default** | Normativa 0,59 → 0,95, +36 % tokens | Evaluación de la generación | **S11** |
| **Etiqueta de traza** ("retrieved but not cited" incluye chunks ➕) | UAT brief 3 | Al tocar el frontend | S11 |
| **Drift check del anotador** | No hecho | Antes de usar el golden set para una decisión fina | — |
| **Compartir `identity` con la imagen** | El detector replica vocabulario | Empaquetado | M6 |

---

## 7. Sesión en vivo

- **Idioma:** canal semántico y léxico alineados en el mismo idioma. RAG-ECSS solo inglés,
  `tsvector` en inglés — alineado; la sesión no lo tocó.
- **Tablas (Docling, MinerU):** relevante para las 272 tablas que exceden la ventana y para D-1. Sigue
  siendo candidato para el re-chunk agrupado.
- **Marcar qué tiene respaldo y qué es generado:** conecta con el contrato de salida tipado (S11).
- **Imágenes:** fuera de alcance.

---

## 8. Proceso

- **Cuatro briefs encadenados**, cada uno decidido con la tabla del anterior. El 4 no estaba
  planificado: se abrió porque la señal del 3 era positiva y el reranker es contenido central de S10.
- **Coste:** casi todo por suscripción (anotación); API solo para embeddings y 14 llamadas de TTFT
  (~0,10 USD).
- **Protocolo de anotación** versionado en el repo (1.0 → 1.1), workspace externo, cuatro rondas de
  modo B. El importador rechaza con todas las razones a la vez y nunca sobrescribe juicios.
- **Lo que hizo CC sin que el brief lo pidiera y fue correcto:** aislar el workspace del anotador,
  añadir `criterion` e `input_fingerprint` a la cabecera del protocolo, conservar `legacy` solo para
  reproducir tablas, calcular BM25 en Python para poder testear la fórmula, el contraste
  `hybrid_tok_bm25_quota`.
