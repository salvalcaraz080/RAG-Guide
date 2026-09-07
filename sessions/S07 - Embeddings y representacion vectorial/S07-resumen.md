# S07 — Resumen de sesión: Embeddings, chunking y espacio vectorial

> **Estado:** provisional, pre-implementación. Notas de teoría + decisiones tomadas en revisión teórica.
> Documento de referencia limpio; la lectura crítica del material vive en la conversación, no aquí.
> La divergencia teoría-vs-implementación se consolidará en `S07-consolidado.md` tras el brief.

---

## 1. Teoría destilada

### 1.1 Embeddings: fundamentos

- Un embedding es una función texto → vector de dimensión fija. El significado no está en las dimensiones individuales (no interpretables), sino en **direcciones y distancias relativas**.
- La geometría emerge del **entrenamiento contrastivo** (ancla / positivo / negativo). La naturaleza de los positivos determina qué entiende el modelo por "similar" → un modelo entrenado en paráfrasis de inglés general no discrimina jerga densa de un dominio especializado.
- **La métrica de similitud no es una decisión de arquitectura.** Para vectores normalizados, coseno = dot product, y la euclídea ordena igual. La fija la *model card*, no nosotros. (Matiz aguas abajo: el índice ANN de la BBDD vectorial sí se optimiza por métrica — se re-acopla en S08, no antes.)

### 1.2 Lo que un embedding no resuelve (lo más relevante para nosotros)

- **Ciego a identificadores y números.** Los IDs y cifras son tokens cualesquiera; el modelo no aplica aritmética ni distingue con fiabilidad `ECSS-E-ST-40C` de `ECSS-Q-ST-80C`. → Filtrado sobre **metadata estructurada**, no similitud vectorial. (Mismo modo de fallo que el cache semántico rechazado en M2: "criticality B" ≈ "criticality C" en el espacio, cláusulas distintas.)
- **Débil en coincidencia exacta de palabra rara.** BM25 la recupera mejor → búsqueda híbrida (S10). Relevante: ECSS está saturado de tokens raros exactos (IDs de cláusula/requisito, terminología específica).
- **Maldición de la dimensionalidad.** Similitudes de 0.2–0.5 entre textos no relacionados son normales. → **Calibrar umbrales sobre dataset propio**, nunca hardcodear `0.7`. Coste asimétrico: umbral bajo → inyectas cláusula irrelevante (fabricación); umbral alto → pierdes el requisito que gobierna.
- **La elección de modelo pesa más que la de métrica.**

### 1.3 Selección de modelo: ejes y método

- **Método correcto:** benchmark sobre datos propios con consultas reales. MTEB es filtro grueso para descartar modelos claramente débiles, no oráculo (mide genérico; es target gameado; la varianza de chunking iguala o supera la de modelo — Vectara/NAACL 2025).
- **Cinco ejes:** dimensionalidad, idioma del corpus, dominio, hosting/coste, licencia. Su prioridad no es universal; se identifica cuál pesa más en el contexto y la decisión se simplifica.
- **Matryoshka (MRL):** las primeras dimensiones cargan más información; se puede truncar conservando calidad. Palanca de optimización futura, con condición de activación (millones de vectores + presión de storage/latencia). *Gotcha:* al truncar manualmente se pierde la norma unitaria → renormalizar.

### 1.4 Chunking: por qué domina y cómo se organiza

- **El chunking domina la calidad del RAG**, tanto o más que la elección de modelo. Invertir en el chunker rinde más que cazar 1-2 puntos de MTEB.
- **Cuatro familias:** mecánicas (fixed-size, recursive, sentence-window), estructurales (document-based, parent-child), semánticas (semantic, cluster, propositional/LLM), avanzadas/contextuales (late, agentic, query-dependent, Contextual Retrieval).
- **Regla de oro:** el mejor chunking depende del **tipo de documento**; corpus heterogéneo se beneficia de estrategias distintas por tipo.
- **Estructural + metadata enriquecido = mayor ROI conocido** (Azure: +15–25 pts de accuracy solo con headers contextuales). Cuando la estructura del documento es fiable, respetarla conserva contexto sin análisis semántico.
- **Parent-child es arquitectura de retrieval, no solo chunking:** decide qué se indexa (hijos) vs qué se devuelve (padres). Anticipa la separación **recuperar ≠ inyectar**.
- **Contextual Retrieval (Anthropic):** prepende contexto generado al chunk **antes de embeber e indexar en BM25** — no como texto devuelto. Reduce fallos de retrieval de forma robusta.

### 1.5 Chunking aplicado: patrones transferibles

- **Dos chunkers especializados sobre interfaz común** > un genérico con condicionales internos (simplicidad).
- **Contextual header estático** (info del padre que ya tenemos, sin LLM) = versión barata y **determinista** de Contextual Retrieval.
- **Regla texto-vs-metadata:** si la info cambia el significado semántico para una query natural → al texto embebido; si es discreta y para filtrar → a metadata; a veces a ambos con roles distintos.
- **IDs trazables** (`{padre}::{hijo}`): críticos para citar, auditar e invalidar chunks cuando el padre cambia de versión.
- Etapas distintas del pipeline pueden usar modelos distintos legítimamente (p. ej. modelo local barato para detectar fronteras, modelo de índice para embeber).

---

## 2. Decisiones RAG-ECSS

### 2.1 Fundamentadas en S07

Estas ya estaban decididas en principio (S06) o se derivan directamente de la teoría; S07 les da base y las deja firmes.

- **Idioma canónico = inglés hacia el modelo.** Independientemente del idioma de la consulta del usuario, toda la información se pasa al modelo en inglés. Colapsa el retrieval cross-lingual a mono-lingüe y **saca el requisito multilingüe del embedder** (se traslada a un paso de normalización de query). *El mecanismo de normalización queda abierto — ver 2.2.*
- **La métrica no se delibera:** la fija la model card del modelo que se elija.
- **Despacho por ancla de contenido, no por tipo de documento.** (Corregido contra el corpus real, S06.) El criterio no es "standard→un chunker, handbook→otro", porque el ancla de contenido es **mixta dentro de un mismo documento**: un standard normativo tiene requisitos *y* secciones con tablas aplanadas. Medición de S06: 6 de 26 documentos no tienen ningún requisito (591 secciones), y el contenido tabular se ancla a la sección — un chunker anclado al requisito deja fuera ambos. Regla: **contenido portador-de-requisito → unidad requisito; contenido anclado-a-sección (los 6 docs sin requisitos + tablas aplanadas de cualquier documento) → unidad sección**. Ambos chunkers corren sobre cada documento, cada uno extrae su ancla. El grado de autoridad viaja con el contenido (ya etiquetado en S06: la tabla lleva la `authority` de su sección), no se deduce del tipo de documento.
- **Contenido portador-de-requisito → estructural, unidad = requisito**, desde los clause-IDs (numeración resuelta en S06). No recursive: recursive es para prosa sin estructura; aquí la estructura ES el significado y las fronteras están declaradas.
- **Contenido anclado-a-sección → estructural, unidad = sección.** **No semántico:** las secciones ECSS tienen numeración declarada (resuelta en S06); las fronteras no hay que descubrirlas por caída de similitud. El chunker de sección **no es "el de handbooks"** — cubre todo lo anclado a sección, incluidas tablas aplanadas dentro de standards normativos. (Único resquicio semántico: sub-partir una sección anómalamente larga como *fallback* recursive-dentro-de-sección.)
- **Overlap = 0 para unidad requisito.** El overlap existe para no perder ideas en fronteras de prosa continua; sobre un átomo acotado (requisito) contamina la cita (arrastra cola del requisito vecino). (Overlap intra-sección para unidad sección: admitido — ver más abajo.)
- **Identificadores → metadata, no embebidos.** `standard-ID`, `clause-ID`, revisión y `corpus_version` viajan como metadata filtrable. Son ciegos al embedding (§1.2) y meterlos en el vector añade ruido, no señal. El header contextual carga contexto **semántico** (materia del standard, tema de la cláusula), no identificadores.
- **Header contextual estático para standards.** Gana en **dos** ejes sobre la versión con LLM: coste y **reproducibilidad** (mismo `corpus_version` → mismos embeddings; el LLM metería no-determinismo en el índice).
- **embed ≠ cite.** El texto embebido lleva header + requisito verbatim; el texto citado apunta al **requisito verbatim limpio**, sin header sintético. La extracción de cita no puede agarrar `chunk.text` a lo bruto.
- **Cluster chunking: excluido.** Rompe la trazabilidad (un chunk clusterizado no tiene lugar en el documento original). Innegociable bajo cita obligatoria.
- **Propositional / LLM chunking: excluido para unidad requisito.** Descomponer un requisito en "proposiciones autocontenidas" parafrasea el texto normativo — mismo vector de fabricación que resumen de historial y cache semántico. (Permisible sobre contenido informativo, donde cita = procedencia — ver 2.2.)
- **Alcance de S07: despacho por ancla, 1 espacio.** Cubre ambas unidades (requisito y sección) sobre **interfaz común → un pipeline de embedding → un espacio → filtro por grado-de-autoridad en metadata**. No excluir la unidad sección — es fuente autónoma (los 6 docs sin requisitos) y su omisión deja el sistema funcionalmente incompleto; además, construir solo la unidad requisito nunca ejercería cite = procedencia y forzaría un retrofit posterior de la maquinaria embed ≠ cite. Espacios separados = multi-índice/routing (S10), solo si la medición muestra que el compartido se queda corto. El modelo (abierto) es uno para ambas unidades.
- **Orden de implementación: unidad requisito primero.** Secuenciar requisitos (átomo acotado, cite = identidad, sin unidad sobredimensionada) antes que secciones. Valida el pipeline base en el caso simple; las secciones añaden variabilidad (tamaño, fallback de sub-partición) sobre esa base. Es orden, no recorte de alcance.
- **Overlap intra-sección admitido.** A diferencia del overlap 0 de la unidad requisito, la prosa dentro de una sección admite overlap. La unidad sección necesita además un *fallback* de sub-partición para secciones que exceden ventana (recursive-dentro-de-sección), con la pregunta asociada de qué se cita entonces — más tolerable bajo cite = procedencia, pero a decidir en el brief.
- **Chunks ref-*ready*, agnósticos al consumo.** (Decisión del repaso de retrieval, ajustada por staleness.) Cada chunk expone un **slot de referencias salientes** + su **clause-ID exacto prefix-addressable**. El chunker cablea la *estructura* (slot + esquema de ID); la **población** del slot es un paso downstream, **gateado por el re-run + validación del extractor a 26 docs** — no se acopla al `references.json` stale/no-validado. Requisito **invariante a la forma de retrieval**: grafo-adyacencia, metadata y baking-en-embedding necesitan el mismo material. Desbloquea el chunking sin cerrar grafo-vs-metadata ni depender del extractor sin validar.
- **Direccionamiento multi-nivel por prefijo de clause-ID (Opción B).** El destino de una referencia lo dicta la referencia, no nosotros — luego el direccionamiento debe resolver a **cualquier** granularidad. Los clause-IDs ECSS ya son jerárquicos (nodo del grafo = `{canonical_doc_id}#{section_number}`): ref a requisito → match exacto; ref a sección → match por **prefijo** de ID. Esto da **parent-child en su versión barata** (por-prefijo, sin índice de padres físico ni duplicación de chunks). Se rechaza normalizar todo a sección (Opción A): sobre-recupera y degrada la precisión-de-cláusula que es el diferenciador.
- **Referencias tabulares: unión verificada, no asumida.** S06 dejó el artefacto `table_references` (28 tablas → 281 pares, aparte del contenido citable) y, por construcción, **toda tabla retenida lleva las coordenadas y la `authority` de su sección contenedora** (las tablas sin cláusula contenedora se descartan con motivo, no reciben ancla inventada). El "gap marcado, nunca silencioso" **ya está implementado**: 8 tablas con `content_extractable: false` + nota. Lo que queda por verificar se estrecha a: (a) shape exacto de `table_references` — ¿cada par carga `source_clause_id` *joinable* al schema de lista plana de `references.json`?; (b) vigencia (¿26 docs o stale a 14?). Verificación = script de inspección read-only de CC, corrida por el usuario en UAT.
- **Regenerar ≠ validar (gate de población de citas).** Regenerar `references.json` a 26 docs es cobertura completa; **no** es fidelidad validada. El extractor tiene deuda conocida sin medir (ventana cláusula↔doc ±80 chars → falsos positivos; empareja primera cláusula en orden de texto, no la más cercana) y un falso positivo es **cita fantasma** — rompe la trazabilidad, el diferenciador. Universo efectivo al regenerar = **20, no 26** (los 6 docs sin requisitos aportan cero refs de prosa; el extractor deriva refs del texto de requisitos). El artefacto regenerado alimenta desarrollo (chunk ref-ready, diseño de adyacencia), pero **poblar citas visibles al usuario desde él sin validar el extractor está prohibido**. El reporte debe declarar explícitamente "cobertura-completa / fidelidad-sin-validar", o una cobertura reciente se leerá como confiable sin que nadie midió la fidelidad (el modo de fallo que el manifiesto de S06 existe para prevenir).

### 2.2 Abiertas — a decidir con medición en el brief

- **Modelo de embedding.** Abierto, *contra* la recomendación del artículo (que lo cierra en `text-embedding-3-small`). Cerrarlo ahora sería importar decisión pre-tomada. Se resuelve con benchmark sobre **pares adversariales del dominio**, incluyendo el **par trampa "criticidad B" vs "criticidad C"**. Prioridad de ejes para RAG-ECSS: **dominio > (licencia/privacidad, condicionada al cliente) > idioma > resto**. Nota: fijar inglés canónico (2.1) reduce la presión multilingüe sobre el embedder.
- **Instrumento de selección de modelo.** El harness del material mide plumbing (latencia, dimensiones, norma), no discriminación de dominio. Falta el instrumento real: pares adversariales del dominio con el par trampa. (Esto es sanity, no calidad formal de retrieval — eso es S11: recall@k, NDCG.)
- **Mecanismo de normalización de query a inglés.** ¿Traducir la query antes de embeber? ¿Con qué? Riesgo propio de drift sobre la consulta. Decisión de estrategia diferida explícitamente.
- **Contextual Retrieval con LLM sobre la unidad *embeber*.** Condicionado a que embed ≠ cite esté cableado. Puede **erosionar parcialmente** grafo-vs-metadata: inyectar en el contexto de embedding la conciencia de cross-refs ("este requisito referencia las definiciones de criticidad de ECSS-Q-ST-80") hace el chunk recuperable por queries sobre lo referenciado sin traversar grafo (aunque no permite *recorrer* referencias en query-time).
- **Propositional para handbooks.** Deja de estar prohibido (autoridad = procedencia); si *compensa* es coste/beneficio a medir.
- **Late chunking.** Resuelve "el chunk pierde contexto al aislarse" sin reescribir texto (cero fabricación) — más seguro que Contextual Retrieval en ese sentido. Pero exige modelo de contexto largo + token-level embeddings → **acopla con la decisión de modelo (abierta)**. Probablemente diferir; acoplamiento anotado.
- **Contenido de tabla como destino recuperable/citable.** La tabla como *arista* se resuelve ahora (§2.1); cómo su *contenido* se embebe/renderiza para ser findable y citable se difiere. No es cosmético: respuestas reales viven solo en tablas ("¿qué requisitos aplican a criticidad B?"). Diferido con la disciplina de "gap marcado, nunca silencioso" (§2.1).
- **Fan-out y profundidad del grafo (lado retrieval, no chunking).** Expandir un target-ID de sección a sus chunks-hijos por prefijo, control de profundidad del traversal (las refs ECSS pueden ser cíclicas; el cierre transitivo puede explotar el contexto), y mezcla de lo traído por grafo con lo traído por similitud. Diseño de M4/S09-10; no condiciona dónde se parte el texto.

### 2.3 Anclajes heredados aún abiertos (de S06)

- **Grafo-vs-metadata** (decisión grande). Se **resuelve un sub-nodo**: grafo-*objeto* (`nx.DiGraph`) vs índice-de-adyacencia. Gana el índice de adyacencia construido desde `references.json` (dict en memoria) — el `nx.DiGraph` no es dependencia de runtime de M2 (ya documentado); queda para tooling/multi-hop si M3 lo pide, sin acoplarse al camino caliente. **Sigue abierto** el nodo grande, que no es sobre estructura de datos sino sobre *cómo se recupera lo referenciado*: adyacencia vs metadata vs **hornear la conciencia de cross-ref en el embedding** (Contextual Retrieval). Regenerar `references.json` es agnóstico a esa decisión (las tres formas lo consumen). La pregunta sigue: "¿qué se arregla antes de decidir?" — extractor sin validar (recall 48% medido en S06, 40% de pérdidas tabulares).
- **Las cuatro unidades** (recuperar / inyectar / citar / embeber): S07 las volvió concretas tres veces (parent-child por prefijo, Contextual Retrieval, header contextual). Siguen sin cablearse.
- **`corpus_version` no viaja a los JSON.** Reforzado por S07: cualquier derivado (embeddings) debe atarse a su versión; el determinismo del header estático depende de ello.

---

## 3. Principios que gradúan (candidatos a Guide — paso 6, no ahora)

- **Recall ≠ fidelidad de cita.** Los benchmarks de chunking (Chroma, Vectara) optimizan recall/accuracy; ninguno mide que el texto recuperado *sea* el texto fuente verbatim. Bajo trazabilidad, recall es necesario pero no suficiente: una técnica que reescribe texto vale cero aunque maximice recall.
- **La trazabilidad es filtro, no criterio.** Mata familias enteras (cluster, propositional-para-normativo) *antes* de medir. No entra en el trade-off; lo precede.
- **El grado de autoridad decide la técnica.** Una misma técnica (propositional, Contextual Retrieval con LLM) es prohibida para fuente normativa (cita = identidad) y permisible para fuente informativa (cita = procedencia). El modelo de dos autoridades de S06 se paga aquí.
- **La estructura declarada poda el descubrimiento semántico.** Cuando las fronteras están declaradas (IDs de requisito, numeración de sección), no hay que descubrirlas por similitud. Semantic/topic-segmentation son para corpus *sin* estructura.
- **Determinismo del índice.** Para un sistema versionado por `corpus_version`, la entrada de embedding debe ser reproducible → preferir enriquecimiento estático sobre generado por LLM en el camino de embeber.
- **Distintos ejes pueden apuntar al mismo candidato por razones distintas** (licencia/privacidad y "corpus lleno de identificadores" apuntan ambos a modelos como bge-m3 — self-host + dense/sparse/multi-vector — por razones no relacionadas). No cerrar el modelo de forma que cierre la híbrida.

---

## 4. Anclajes que S07 deja para los briefs

S07 se implementa en **más de un brief** (disciplina: un brief por funcionalidad). El chunking es funcionalidad autónoma, testeable con `ParsedDocument` mockeado, sin depender del modelo (abierto) ni del vector store (S08). El wrapper de embeddings es funcionalidad aparte y además está parcialmente bloqueada por la elección de modelo. Por eso:

### Brief 1 — Chunking: chunks ref-ready por ancla de contenido

Produce objetos `Chunk` listos para embeber, con la maquinaria embed ≠ cite y el slot ref-ready. Orden interno: unidad requisito primero, luego unidad sección. Puntos a fijar:

0. **Verificación previa (script read-only de CC; lo corre el usuario en UAT):** shape y vigencia de `table_references` (¿cada par carga `source_clause_id` joinable al schema de `references.json`? ¿26 docs o stale a 14?), y confirmar que `section_number` es prefix-addressable. No muta `data/`; reporta.
1. **Despacho por ancla de contenido sobre interfaz común.** Unidad requisito: desde clause-IDs, overlap 0. Unidad sección: desde numeración, overlap intra-sección admitido, con *fallback* recursive-dentro-de-sección para secciones que exceden ventana (conteo de tokens vía **tokenizer abstraído**, no `tiktoken` directo). Ambas corren sobre cada documento. Texto embebible = header contextual semántico + **texto fuente verbatim**.
2. **Separación embed ≠ cite** cableada: `embed_text` (header + texto fuente) distinto de `cite_ref` (texto verbatim limpio + clause-ID + `corpus_version`). La extracción de cita nunca agarra `embed_text` a lo bruto. Validar contra **ambos** modos: identidad (unidad requisito) y procedencia (unidad sección).
3. **Slot ref-ready + ID prefix-addressable.** El chunk expone el slot de refs salientes (vacío en este brief) y su clause-ID prefix-addressable. La **población** del slot NO entra en este brief — es downstream, gateada por el re-run + validación del extractor. El chunk queda agnóstico al consumo (grafo/metadata/embedding sin decidir).
4. **Metadata filtrable** (fuera del vector): standard-ID, clause-ID, revisión, `corpus_version`, **grado de autoridad** (normativo/informativo, tomado del contenido), ancla (requisito/sección), slot de refs.
5. **`corpus_version` propagado** a los chunks/JSON derivados (cobra el anclaje de S06).

### Brief 2 — Wrapper de embeddings (posterior; parcialmente bloqueado por elección de modelo)

6. **Provider-abstraction para embeddings** (no `client.embeddings.create` desnudo): el wrapper debe saber qué modelo para qué etapa, y — si se deja la puerta al self-host — abstraer también backend local.
7. **Determinismo por regresión test:** aseverar que mismo `corpus_version` + mismo input → mismos embeddings (candidato "true by construction → true by regression test"). Si se trunca (Matryoshka), aseverar norma ≈ 1.

### Fuera de scope de S07 (diferido, marcado)

Población del slot de refs (gateada por re-run + validación del extractor a 26 docs, universo efectivo 20); contenido de tabla como destino recuperable/citable; Contextual Retrieval con LLM; propositional sobre contenido informativo; late chunking; elección de modelo (se mide aparte, sobre pares adversariales con par trampa); mecanismo de normalización de query a inglés; fan-out, profundidad del grafo y mezcla grafo-similitud (M4/S09-10); decisión grafo-vs-metadata (sigue abierta, no comprometida por el chunk ref-ready).

> **Nota de proceso:** el material es de mayo 2026 (precios/modelos se mueven cada trimestre). Cuando toque *seleccionar* modelo de verdad, verificar panorama actual con búsqueda por mes/año antes de comprometer.

> **FYI en curso (usuario):** regeneración de `references.json` a 26 docs solicitada a CC (no arrastrar el stale de 14). Es regeneración de cobertura, **no** validación de fidelidad — ver §2.1 "regenerar ≠ validar". El reporte debe declarar "cobertura-completa / fidelidad-sin-validar".
