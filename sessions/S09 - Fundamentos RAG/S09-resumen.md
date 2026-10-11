# S09 — Fundamentos RAG + retrieval + capa de datos como servicio

> Resumen provisional (paso 2). Teoría de los cinco artículos + decisiones tomadas en la revisión.
> Pre-implementación: lo que aquí figura como decisión todavía no está validado en código.

---

## 1. Del CAG al flujo RAG

**Anatomía.** El flujo RAG se describe en cuatro etapas: **Query → Retrieval → Augmentation →
Generation**. Es convención industrial, no la nomenclatura de Lewis et al. (2020), cuyo RAG
entrena conjuntamente retriever (DPR) y generador (BART). Antes de las cuatro hay una **etapa
cero, la indexación** (parsing, chunking, embeddings): el retrieval está acotado por ella.

Las variantes modernas no siempre "sofistican" una etapa: Self-RAG y Agentic RAG **reorganizan**
el flujo — el retrieval pasa a ser condicional e iterativo, decidido por el modelo. Pipeline
lineal frente a bucle es una decisión propia (M5).

**CAG vs RAG — cinco ejes.**

| Eje | CAG | RAG |
|---|---|---|
| Frescura | Cambiar el conocimiento = editar prompt y redesplegar | Insertar documentos sin tocar código |
| Techo de corpus | La ventana (y el contexto *efectivo*, menor) | Sin techo; solo sube lo relevante |
| Latencia / coste | Una llamada, prompt caching | Embedding + consulta + LLM (+ reformulación opcional). Con corpus grande puede salir más barato que un CAG enorme |
| Trazabilidad | Se conoce el contexto, no qué parte se usó | Conjunto candidato menor; la trazabilidad real la dan **citas + validación** |
| Alucinación | La instrucción de grounding aplica igual | Menos distractores, menos *lost in the middle* |

- Chan et al. (2024), *Don't Do RAG*: con corpus pequeño y estable, CAG iguala o supera a RAG.
  El umbral de ~100k tokens es orden de magnitud, no regla.
- **Crecimiento ≠ rotación del corpus.** Crecer justifica RAG por techo; rotar justifica RAG por
  frescura y arrastra una decisión de versionado (¿conviven revisiones?).
- **Híbrido CAG + RAG:** lo estable y transversal va fijo en el prompt; lo variable se recupera.
  En un corpus normativo, lo fijo no es contenido sino **reglas de lectura** del corpus.

**"Retrieval domina".** El techo de calidad lo fija lo que llega al contexto. Orden de
depuración: **indexación** (¿existe un chunk con la respuesta entera?) → retrieval →
augmentation → prompt → modelo.

**Fronteras del chunk — dos ejes.** El contexto queda incompleto en vertical (requisito partido
de su cláusula: *recomposición*) y en horizontal (la cláusula remite a otra que no se
recuperó: *referencias cruzadas*). Las referencias cruzadas son una debilidad estructural de
RAG, no un argumento a su favor: un CAG con todo el corpus sí ve ambas cláusulas.

## 2. Reformulación de queries

**Problema que resuelve:** entradas largas y ruidosas (el vector resultante es un centroide),
keywords ahogadas por ruido conversacional, anáforas a contexto ausente.

**Cinco familias:**

| Técnica | Qué hace | Cuándo encaja |
|---|---|---|
| Query rewriting | LLM reescribe como consulta concisa | Entrada corta y mal formulada |
| Sub-query decomposition | N sub-queries + fusión (RRF) | Sub-temas ortogonales (p. ej. comparativa entre fuentes) |
| Step-back | Pregunta más abstracta | QA sobre conocimiento estructurado |
| HyDE | Embebe una respuesta hipotética | Corpus narrativo y dominio conocido por el modelo; riesgo: sesga el retrieval hacia el *prior* del modelo |
| Extracción estructurada | JSON validado → texto a embeber + filtros de metadata | Hay campos útiles como filtro; produce un artefacto inspeccionable |

- La reformulación es una capa de primera clase: prompt y esquema versionados, instrumentación
  (tasa de fallback como señal de salud) y tests.
- **Condición de aplicabilidad:** entrada larga/ruidosa, conversacional, multi-tema ortogonal o
  con desajuste de idioma. Si nada de eso se cumple, no se reformula hasta que la medición lo pida.
- Problemas de query propios de corpus técnicos: **identificadores exactos** (se resuelven en
  retrieval — canal léxico o filtro determinista —, no con un LLM), **seguimientos
  conversacionales** (reescritura condicionada al historial) y **comparativas** (descomposición).
- Un **filtro duro** convierte un error del extractor en recall cero. Filtrar solo con evidencia
  explícita, o usar boost.
- Structured output `strict` (OpenAI) exige todas las propiedades en `required` y
  `additionalProperties: false`: un modelo Pydantic con defaults no es válido tal cual.

## 3. Retrieval: top-K, threshold, filtros

- **Dos fallos de top-K a secas:** resultados *clónicos* (varios chunks del mismo origen) y
  *mediocres* (K resultados, ninguno bueno).
- **Contrato del retriever:** K como techo; el número que llega al contexto es emergente; **cero
  resultados es información** y se propaga hacia arriba (soft-fail), no se genera sobre vacío.
- **Threshold:** se calibra contra etiquetas de relevancia, nunca se copia. Absoluto solo si la
  distribución de distancias lo permite; si varía por query, relativo (gap con el top-1).
  **No aplica sobre scores de fusión por rango (RRF)**, que no tienen escala absoluta.
  Separar dos funciones: *filtrar chunks* (necesita score calibrado por pareja query–chunk; llega
  con reranking) y *decidir si hay evidencia* (por query).
- **Clónicos:** se combaten con diversidad (MMR, máximo por documento) o, en corpus jerárquicos,
  cambiando la unidad de contexto (agrupar por la unidad citable).
- **Filtros en pgvector:** pre-filtering (filtro selectivo), post-filtering (filtro poco selectivo,
  con `wide_k`), in-query. Los *iterative index scans* llegan en **pgvector 0.8.0** y vienen
  **desactivados por defecto** (`hnsw.iterative_scan`); sin ellos, un filtro selectivo sobre HNSW
  devuelve menos de K sin avisar (límite `ef_search`).
- **Escala:** por debajo de ~100k vectores, el KNN exacto es milisegundos y recall 1.0; la
  sintonía de HNSW con filtros es un problema de escalado.
- `(:x IS NULL OR …)` con prepared statements puede producir *generic plans* malos; construir el
  `WHERE` dinámicamente es igual de simple.
- **Anti-patrones:** subir K para arreglar calidad; "el LLM filtrará"; omitir el criterio de
  abstención; mezclar tipos de chunk sin intención.
- **Recall vs precision** se decide por la asimetría de errores del dominio, y puede variar por
  tipo de query (enumerativa vs puntual). Principio: **el retrieval asegura cobertura, la
  generación asegura corrección.**
- En dominio normativo, "no encontrado" se formula **sobre el corpus indexado, nunca sobre la
  norma**: ausencia de evidencia no es evidencia de ausencia.

## 4. Augmentation y generación

- **Tres patologías del `"\n\n".join`:** citas inventadas, mezcla entre fuentes, respuesta que
  ignora el contexto. Causa común: el modelo no sabe qué es el bloque de texto.
- **Delimitadores XML con metadata** como atributos (`<source id=…>`): unidades atribuibles.
  JSON como contenedor tiende a leerse como dato, no como referencia.
- **Lost in the middle** (Liu et al., 2023): curva en U por posición. Con K pequeño basta
  orden por relevancia; con K grande, patrón en U. Alternativa para corpus jerárquicos: orden de
  documento dentro de una misma fuente.
- **Truncar por unidad completa**, contando el envoltorio. Con unidades de tamaño variable,
  saltar la que no cabe (`continue`), no cortar la lista (`break`). El presupuesto de contexto
  se fija por calidad y coste, no por la ventana.
- **Prompt de generación — contratos:** solo contexto; citar cada afirmación; no inventar
  identificadores; política de insuficiencia; evidencia separada de asunción. Repetir la regla
  crítica al final del user prompt.
- **Validación post-generación:** comprobar que lo citado pertenece al conjunto recuperado
  (pertenencia ≠ soporte; la fidelidad es S11). Reintento acotado como política por defecto.
- **Identificadores de cita:** handles locales opacos impiden citar lo no recuperado cuando el
  modelo conoce el corpus de preentrenamiento; chocan con la memoria conversacional si no se
  mapean antes de guardar el turno.
- En corpus normativos, la **cita literal** del requisito permite una validación determinista de
  soporte (substring sobre texto normalizado).
- Exponer la distancia al modelo no aporta: no calibra números y el orden ya transmite el rango.

## 5. La capa de datos como servicio

- Retrieval y generación son dos servicios lógicos: radio de impacto de cambios, coste muy
  asimétrico, granularidad de credenciales. Contratos asimétricos: retrieval expone palancas;
  generación encapsula.
- **Modelo de autenticación según consumidor:** servicio→servicio (API key), usuario final
  (sesión/JWT), malla de servicios (mTLS). `secrets.compare_digest` siempre; rotación con dos
  claves válidas.
- **Securizar la capa de datos de un RAG = autorización en el retrieval**, aplicada dentro de la
  consulta (corpus con permisos o multi-tenant). Las API keys protegen el endpoint, no los datos.
- **Rate limiting** gana su sitio con exposición pública o varios consumidores; antes, un tope de
  gasto en el proveedor cubre el riesgo de coste. En memoria escala mal con varios workers.
- **Idempotencia** para operaciones caras; debe cubrir **peticiones en vuelo** (estado "en
  proceso"), o un reintento por timeout duplica justo la operación cara.
- **Logging por etapa con `request_id`**; en retrieval registrar **qué se recuperó** (IDs, rango,
  score), no conteos.
- Caché de respuestas en RAG: la versión del corpus entra en la clave.

---

## 6. Decisiones S09

### Cerradas
- Producto **solo en inglés**.
- `llm_service` **no conoce la sesión de BD**: recibe un proveedor de contexto inyectado.
- **Unidad de contexto y de cita = cláusula** (agrupación por `(documento, cláusula)`).
- **Retrieval sesgado a recall**: omitir un requisito aplicable es peor que citar uno irrelevante.
- **Sin threshold por chunk** en S09.
- **Handles opacos** en el prompt; identificadores naturales al usuario y en el historial.
- **Cita literal con validación normalizada, en modo observación**; la política ante fallo se
  decide en S11 con la tasa medida.
- **Reglas de lectura normativa** en el system prompt (peso de "shall/should/may", NOTE
  informativa, cobertura limitada a las fuentes, insuficiencia sobre el corpus indexado).
- **Sin rate limiting** en S09; tope de gasto en la consola del proveedor. Sin URL pública.
- **Sin reformulación LLM** en S09: queries cortas y en inglés.
- **Sin caché.**

### Diferidas
- Filtro por **área de conocimiento** (taxonomía propia, metadata a nivel documento) — cuando el
  corpus salga de software.
- **Expansión de cláusula** (arrastrar chunks no recuperados de la misma cláusula) — se decide
  con la medición que produce el brief S09.
- Seguir **referencias cruzadas** (grafo sin validar) — S10+.
- Política ante cita no verificada — S11.
- `/search` como funcionalidad de producto — hipótesis; señal: usuarios que ignoran la respuesta
  generada y van a las citas.
- Reescritura de seguimientos conversacionales — si la UAT muestra fallo en anáforas.

---

## 7. Sesión en vivo

Pendiente: no se ha revisado material de la sesión en vivo.
