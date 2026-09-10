# Auditoría de estado — chat puente S08 → S09, Paso A

**Tipo:** informe de diagnóstico, read-only. Ninguna disposición: los estados son fácticos y la
clasificación (fantasma / futura / transversal) se hace en Claude Chat sobre este informe.

**Fecha:** 2026-09-09 · **Rama:** `s08-bdd-vectoriales` · **HEAD:** `59002b5` · **Árbol:** limpio

**Nota de ruta:** el brief pide el informe en `sesiones/bridge-S08-S09/`. En el repo `RAG-Guide`
el directorio real es `sessions/` (inglés), y ahí vive el brief. El informe va a
`RAG-Guide/sessions/bridge-S08-S09/audit-report.md`.

**Entorno de la auditoría:** la pila de `docker-compose` estaba levantada (`db`, `api`,
`frontend` healthy), así que los hechos de base de datos y de endpoint son medidos contra el
sistema vivo, no inferidos del código.

---

## 1 · Resumen

### Deuda documentada (Tarea 1)

| Estado | Items |
|---|---:|
| `VIGENTE` | 29 |
| `RESUELTA` | 1 |
| `NO_APLICA` | 1 |
| `INDETERMINADA` | 1 |
| **Total (reconciliado)** | **32** |

Los 32 salen de reconciliar 24 items de `CLAUDE.md` § "Decisiones pendientes", 9 de
`ARCHITECTURE.md` §7 y 4 de los bloques de grafo/extractor de `CLAUDE.md`. Cinco deudas
aparecen en dos documentos a la vez (D09, D20, D22, D23, D29) y se cuentan una sola vez.

### Deriva docs ↔ repo (Tarea 2)

| Dirección | Hallazgos |
|---|---:|
| 1 — el doc afirma algo que el código contradice | 12 |
| 2 — el código hace algo estructural que los docs no reflejan | 3 |
| **Total** | **15** |

**Lo más grave de las dos tareas, en una línea cada uno:**

- `ARCHITECTURE.md` §7 declara sin implementar la ingesta, `/search` y `retrieval.py`; los tres
  existen y funcionan (R1), y §3 se contradice con §5 dentro del mismo documento (R2).
- La deuda de la lista de exclusión (D13) no es latente: `corpus.graph` y
  `corpus.diagnostics.inspect_graph` **fallan hoy** con `KeyError: 'sections'`, así que el grafo
  de referencias no se puede regenerar y el paso 7 del `RUNBOOK` está roto (R8, R11).
- El arnés de ground truth del extractor existe y está commiteado, pero el etiquetado está sin
  hacer: **1 de 50** filas de precisión, **0** de recall; el gate da FAIL con precisión 0,0% (D29, R14).

---

## 2 · Estado de la suite de tests

`uv run pytest` — **837 passed, 8 deselected, 3 warnings en 45,45 s**. Verde.

| Suite | Comando | Resultado |
|---|---|---|
| App (contrato) | `uv run pytest tests/app` | **182 passed**, 8 deselected |
| Corpus | `uv run pytest tests/corpus` | **655 passed** |
| Capa de datos (subconjunto de app) | `tests/app/test_data_layer.py test_ingestion.py test_search.py` | **96 passed** |
| Evaluación (`real_llm`) | `uv run pytest -m real_llm --collect-only` | **8 recolectados**, no ejecutados |

Los 8 `real_llm` son 2 de `test_input_guards.py` (Moderation API real) y 6 de
`test_system_prompt_provider_agnostic.py` (citación y rechazo contra proveedor real). Excluidos
por `addopts = "-m 'not real_llm'"`, como documenta `CLAUDE.md`.

Los 3 warnings son un `RuntimeWarning` de LiteLLM (`Logging.async_success_handler was never
awaited`) en `test_llm_wrapper.py` y `test_query_integration.py`. Ruido de librería, no del
código del proyecto.

### Hechos de estado positivo establecidos por ejecución

| Afirmación de los docs | Verificación | Resultado |
|---|---|---|
| 26 documentos, 13.876 chunks en la base | `psql` contra `rag-ecss-db` | 26 / 13.876 ✅ |
| Ningún chunk sin vector | `count(*) WHERE embedding IS NULL` | 0 ✅ |
| Régimen exacto: sin índice vectorial | `pg_indexes` sobre `chunks` | 8 índices, **ninguno** HNSW/IVFFlat ✅ |
| GIN sobre los dos `metadata` + GIN sobre `cite_tsv` | `pg_indexes` | `ix_chunks_metadata`, `ix_documents_metadata`, `ix_chunks_cite_tsv` ✅ |
| Migración en `head` | `SELECT * FROM alembic_version` | `0003_chunks_lexical_channel` ✅ |
| `oversized` = 2 · `requirement_uid` = 2.536 | `psql` | 2 / 2.536 ✅ |
| `/api/v1/search` responde en modo híbrido | `curl` contra `localhost:8000` | 2 hits, `mode=hybrid` ✅ |
| Endpoints activos | `GET /openapi.json` | `/health`, `/api/v1/{prompt,query,query/stream,search,sessions}` ✅ |
| M1 sigue sobre 1 fragmento estático | `from app.context.reference import ECSS_REFERENCE` | 1 fragmento, `ECSS-E-ST-40C` §5.2.4.8, 230 chars ✅ |
| Artefactos coherentes (par fingerprint, conteo, dimensión, orden) | comparación `chunks.json` ↔ sidecar ↔ `.npy` | `(13876, 1536) float32`, fingerprints iguales, orden fila→chunk idéntico ✅ |
| Cobertura del corpus 100% | `corpus.diagnostics.verify_parser` | 26 declarados / 26 parseados ✅ |
| Numeración: 0 rótulos con número fabricado | `corpus.diagnostics.diagnose_numbering` | 41 rótulos, **0** fabricados ✅ |
| Tablas 988/8/55/28 = 1.079 | agregado sobre `data/processed/` | exacto ✅ |
| Sin cache, sin Redis, sin LangChain/LlamaIndex/RAGAS | `pyproject.toml` + grep | ninguna dependencia presente ✅ |

---

## 3 · Tabla de deuda (Tarea 1)

Fuentes: **(A)** `.claude/CLAUDE.md` § "Decisiones pendientes" · **(B)** `ARCHITECTURE.md` §7 ·
**(C)** `.claude/CLAUDE.md` bloques de grafo / deuda del extractor.

### Parser e identidad de cita

| # | Ref | Deuda (1 línea) | Estado | Evidencia |
|---|---|---|---|---|
| D01 | A · `CLAUDE.md:626-632` | Los marcadores literales de lista (`1.`, `(a)`) no se reproducen en el texto de requisito | `VIGENTE` | `corpus/parser.py:246-251`: comentario + `# TODO: pendiente de decisión en Claude Chat` y `_CONTINUATION_INDENT = "  "`. `corpus/numbering.py:60-64`: `ResolvedNumber` sigue sin campo de sufijo. El join sigue siendo por sangría (`parser.py:561-565`) |
| D02 | A · `CLAUDE.md:633-635` | 2 párrafos «None.» con estilo `require` desnudo quedan en prosa sin clasificar | `VIGENTE` | `prose_coverage.unclassified` de `data/processed/ECSS-E-ST-10C_Rev.1.json` → `{"require": 10}`. La unidad es **chars** (`parser.py:77-85`), y 10 chars = 2 × `"None."`. Ningún otro documento del corpus tiene `require` sin clasificar |
| D03 | A · `CLAUDE.md:636-639` | El contenido de los cuadros de texto flotantes no se captura | `VIGENTE` | `corpus/parser.py:347` (`_W_TXBX = qn("w:txbxContent")`) y `:370` («Text boxes are **excluded**»). No hay camino alternativo de captura |
| D04 | A · `CLAUDE.md:640-643` | Un heading numerado con estilo `Normal` no llega a ser cláusula | `VIGENTE` | `corpus.diagnostics.audit_structural_styles`: `{"style": "Normal", "paragraphs": 1, "documents": 1, "routed_as_heading": false, "verdict": "defect", "samples": [["", "3.4", "Nomenclature"]]}`. Resumen: `"defects": ["Normal"]` |
| D05 | A · `CLAUDE.md:644-649` | 4 números de cláusula identifican a más de una sección dentro de su propio documento | `VIGENTE` | Agregado sobre `data/processed/`: `ECSS-Q-HB-80-01A` §4 y §5 (2 cada uno), `ECSS-Q-HB-80-02_Part_1A` §4, `ECSS-Q-HB-80-02_Part_2A` §4.1.1 («Introduction» / «Process definitions»). Total **4**, en 3 handbooks — exactamente lo documentado |

### Chunking

| # | Ref | Deuda (1 línea) | Estado | Evidencia |
|---|---|---|---|---|
| D06 | A · `CLAUDE.md:650-653` | La autoridad del chunk de requisito se hereda de su sección, frente a forzarla a normativa | `VIGENTE` | `corpus/chunking.py:333` (`authority_of = {s.number: s.authority …}`) y `:346-349`, con el comentario «`Requirement` no lleva authority propia: se hereda de su sección» |
| D07 | A · `CLAUDE.md:654-658` | El ordinal dentro de la sección hace la identidad del `chunk_id` parcialmente posicional | `VIGENTE` | `corpus/chunking.py:337-344` (requisitos), `:434-438` (tablas), `:460-464` (glosario), `:488-505` (secciones y notas). El comentario de `:159` nombra el problema: «ahí un requisito sin UID lleva el ordinal `r0`, que es posición y no identidad» |
| D08 | A · `CLAUDE.md:659-662` | Ventana y solapamiento son knobs sin investigar | `VIGENTE` | `corpus/chunking.py:38` `DEFAULT_MAX_TOKENS = 512`, `:43` `SECTION_OVERLAP_RATIO = 0.12`. Ambos siguen siendo constantes de módulo, sin calibración documentada |
| D22 | A · `CLAUDE.md:720-731` · B · `ARCHITECTURE.md:1307-1310` | Si al recuperar un chunk de requisito conviene arrastrar el de sección de su misma cláusula | `VIGENTE` | Anclas en el corpus: `requirement` 7.922, `section` 2.583, `table` 1.481, `note` 1.410, `glossary_term` 480. Ningún chunk contiene la cláusula entera y `app/services/retrieval.py` no hace ninguna expansión por coordenada: devuelve el top-k tal cual |

### Tablas y referencias

| # | Ref | Deuda (1 línea) | Estado | Evidencia |
|---|---|---|---|---|
| D09 | A · `CLAUDE.md:663-667` · B · `ARCHITECTURE.md:1317-1320` | 189 tablas cuya fila de anotación (`repeated N times`) pierde alcance al aplanar | `VIGENTE` | Agregado sobre `data/processed/`: **189** tablas con mecanismo `flatten` cuyo `content` contiene «repeated». Muestra real de `ECSS-E-ST-70-41C`: `"repeated N times\nN \| packet field 1 \| packet field 2…"`. **El desglose por documento de los docs es incorrecto**, ver deriva R6 |
| D10 | A · `CLAUDE.md:668-672` | 281 referencias extraídas de tablas viven fuera del grafo, sin decidir qué se hace con ellas | `VIGENTE` | Agregado sobre `ParsedDocument.table_references`: **281** registros en 26 documentos. `references.json` tiene **329** registros y no las incluye: son dos artefactos separados |
| D11 | A · `CLAUDE.md:673-677` | Qué inyectar cuando una referencia inter-documento apunta al documento entero | `VIGENTE` | Medido sobre `references.json` (329 registros): 231 inter-doc, **163 aterrizan** en el corpus, de las cuales **48 nombran cláusula** y **115 apuntan al documento entero**. Reproduce los números documentados al dígito |
| D29 | C · `CLAUDE.md:962-963` y `:990-996` · B · `ARCHITECTURE.md:1321-1322` | La precisión del extractor de referencias sigue sin validar contra ground truth | `VIGENTE` | `uv run python scripts/score_groundtruth.py` → «etiquetadas: 1 (49 en blanco)», «alucinadas 1/1», «PRECISIÓN 0,0% (0/1)», recall «sin filas confirmadas todavía», y el gate cierra en `FAIL`. El instrumento existe y está cableado; **lo que falta es el etiquetado humano** |
| D30 | C · `CLAUDE.md:992-993` | Ventana de asociación cláusula↔doc laxa (±80 chars, simétrica) | `VIGENTE` | `corpus/graph.py:158` `_ASSOC_WINDOW = 80`; `:245` la aplica en las dos direcciones (`abs(cm.start() - dm.end()) <= _ASSOC_WINDOW or abs(dm.start() - cm.end()) <= _ASSOC_WINDOW`) |
| D31 | C · `CLAUDE.md:994` | El `break` empareja la primera cláusula en orden de texto, no la más cercana | `VIGENTE` | `corpus/graph.py:244-248`: el bucle recorre `clause_matches` en orden de texto y corta en el primer match dentro de ventana. El propio comentario dice `# use the nearest clause only`, que es justo lo que **no** hace |
| D32 | C · `CLAUDE.md:995` | Una misma cláusula puede emparejarse con varios documentos | `VIGENTE` | `corpus/graph.py:243-247`: `paired_clauses.add(ci)` marca la cláusula pero no la retira de `clause_matches`, y el bucle exterior sobre menciones de documento vuelve a recorrer la misma lista |

### Capa de datos y retrieval (S08)

| # | Ref | Deuda (1 línea) | Estado | Evidencia |
|---|---|---|---|---|
| D12 | A · `CLAUDE.md:678-681` | Qué campos de `Chunk` van al `metadata` JSONB | `VIGENTE` | `psql`: `count(*) WHERE metadata <> '{}'::jsonb` = **0** en `chunks` y **0** en `documents`. Los cuatro candidatos siguen fuera de columna: `token_count`, `oversized` (tiene columna propia desde 0002), `outgoing_refs` y `metadata.revision` |
| D13 | A · `CLAUDE.md:682-688` | El conjunto de documentos se deriva de una lista de exclusión, no del manifiesto | `VIGENTE` | Solo `corpus/chunk_all.py:50` excluye los tres (`_NOT_DOCUMENTS`). Los otros cinco excluyen únicamente `references.json`: `graph.py:403`, `report.py:226-229`, `verify_parser.py:292`, `audit_raw_corpus.py:177`, `inspect_graph.py:45`. Efecto medido: 29 `.json` en `data/processed/`, **28** bajo la exclusión corta y **26** bajo la larga. Consecuencias reales, cada una ejecutada: `verify_parser` reporta «EN processed/ SIN FILA ACTIVA (2): chunks, embeddings»; `audit_raw_corpus` reporta «activos con DOCX: 26 \| parseados: 28 \| **cobertura: 107%**»; `report.py` degrada bien (26/26, los nombra en `unexpected`); **`inspect_graph` y `graph.py` CRASHEAN** (`KeyError: 'sections'` en `parser.py:797`). Ver deriva R8 |
| D14 | A · `CLAUDE.md:689-694` | La fusión RRF no pondera, y eso degrada la query genérica | `VIGENTE` | `app/services/retrieval.py:55-58` (`RRF_C = 60`, «NO es un knob afinado») y `:230-260` (`fuse_rrf`, sin parámetro de peso; el docstring dice «needs neither normalisation nor weights»). Lo medido está commiteado en `scripts/output_examples.txt` |
| D15 | A · `CLAUDE.md:695-698` | 103 chunks sin contenido indexable por el canal léxico | `VIGENTE` | `psql`: `count(*) WHERE cite_tsv = to_tsvector('english','')` = **103**. Desglose: `-` (51), `<<deleted>>` (47), `<<deleted>>.` (3), `<<deleted and incorporated in 5.4.2.3a.>>.` (1), `<<deleted and moved modified as Note to 5.1a>>` (1). Los cuatro últimos suman los 52 «deleted» que documenta el doc |
| D16 | A · `CLAUDE.md:699-701` | El schema de S08 es el inicial, no el final | `VIGENTE` | `alembic_version` = `0003_chunks_lexical_channel`; solo dos tablas (`documents`, `chunks`); ni tabla de referencias ni índice vectorial en `pg_indexes` |
| D17 | A · `CLAUDE.md:702-712` | La representación densa no separa las categorías de criticidad | `VIGENTE` | `corpus.diagnostics.embedding_sanity` reproducido contra la API real: par trampa B vs C = **0,9818**; pares «similares» = 0,7044 y 0,49; «distantes» = 0,2924 y 0,2977; `trap_below_similar: false`. **La coletilla del item («pendiente de medir el lado query … es S09») está desactualizada**, ver deriva R7 |

### Camino de petición y generación

| # | Ref | Deuda (1 línea) | Estado | Evidencia |
|---|---|---|---|---|
| D20 | A · `CLAUDE.md:715-717` · B · `ARCHITECTURE.md:1295-1296` | Trazabilidad de cita / verificación de grounding: las citas son candidatas, no verificadas | `VIGENTE` | `app/services/llm_service.py:236-258`: `_citations` documenta «NOT extracted from the model's answer, not verified — see S11» y devuelve los fragmentos inyectados. No hay comparación contra el texto generado |
| D23 | A · `CLAUDE.md:732-735` · B · `ARCHITECTURE.md:1314-1316` | Fan-out de citas: se devuelven todos los fragmentos inyectados como candidatos | `VIGENTE` | `app/services/llm_service.py:258`: `return [{"standard": f["standard"], "clause": f["clause"]} for f in fragments]`. Sin filtro por mención real. Hoy da 1 cita porque `get_context` devuelve 1 fragmento; con *k* chunks dará *k* |
| D24 | A · `CLAUDE.md:736-741` | Guardrail de prompt injection: hueco comentado, no implementado | `VIGENTE` | `app/services/input_guards.py:94-97`: `await _moderate_toxicity(question)` y a continuación `# TODO: injection guardrail (M2+) — second check of the pipeline, here`. Cabecera del módulo (`:6-9`) confirma que el único check es toxicidad |
| D25 | B · `ARCHITECTURE.md:1297-1300` — **solo ahí** | Sesiones multi-turn en memoria, un solo worker, sin TTL ni federación | `VIGENTE` | `app/services/sessions.py:76` `self._sessions: dict[str, Session] = {}`; `:93` el comentario deja el TTL «para S06/M6»; `:8` «Escalar exige un store compartido (Redis/PG), diferido a S06/M6». `docker-compose.yml:61` arranca `uvicorn` sin `--workers`, así que hoy no se manifiesta |
| D26 | B · `ARCHITECTURE.md:1301-1305` — **solo ahí** | Prompt caching de proveedor (`cache_control`) sin activar | `VIGENTE` | Ninguna aparición de `cache_control` en el código de `app/` salvo el comentario de `llm_wrapper.py:133` que explica que el wrapper lo admitiría. El orden del prompt está preparado; el breakpoint no está puesto |
| D27 | B · `ARCHITECTURE.md:1311-1313` | «Ingesta y `/search` sin implementar … `services/retrieval.py` no existe todavía» | **`RESUELTA`** | Las tres partes son falsas hoy: `app/services/retrieval.py` existe con dos modos (`fuse_rrf` en `:230`); la base tiene 26 documentos y 13.876 chunks; `POST /api/v1/search` está registrado (`app/main.py:28`), aparece en `openapi.json` y responde en vivo. El doc quedó desactualizado — ver R1 |
| D28 | B · `ARCHITECTURE.md:1306` | «Sin cache de respuestas» listado bajo «Qué no está resuelto todavía» | **`NO_APLICA`** | No es deuda: es una **decisión consolidada** y deliberada. `CLAUDE.md:379-380` la registra como tal («Sin cache de respuestas ni Redis — retirada antes de M2. No se reintroduce sin hit rate observado»), y el propio `ARCHITECTURE.md:393` la documenta en §3 como decisión. Aparecer en §7 la reencuadra como pendiente |

### Módulos futuros y producto

| # | Ref | Deuda (1 línea) | Estado | Evidencia |
|---|---|---|---|---|
| D18 | A · `CLAUDE.md:713` | Framework de agentes y orquestación (M3) | `VIGENTE` | Ninguna dependencia de agentes en `pyproject.toml`; ningún módulo de orquestación en `app/` |
| D19 | A · `CLAUDE.md:714` | Stack de evaluación — RAGAS u otras | `VIGENTE` | `pyproject.toml` no declara `ragas` ni equivalente. La única infraestructura de evaluación existente es la marca `real_llm` (8 tests) y `embedding_sanity` |
| D21 | A · `CLAUDE.md:718-719` | Wedge de producto: Requirements Engineering Assistant vs Compliance Matrix Generator vs Verification & Traceability Helper | **`INDETERMINADA`** | No es verificable contra el repo: el propio item declara que depende de **customer discovery**, insumo que CC no tiene y que no se puede derivar del código. Para resolverlo hace falta el resultado de esa investigación |

---

## 4 · Deriva docs ↔ repo (Tarea 2)

### Dirección 1 — el doc afirma algo que el código contradice

**R1 · `ARCHITECTURE.md:1311-1313` — la pista del brief, confirmada y peor de lo esperado.**
El doc dice: «**Ingesta y `/search` sin implementar** (S08 briefs 2 y 3): el schema y la
infraestructura existen (§5) pero están vacíos, y `services/retrieval.py` no existe todavía».
Las tres afirmaciones son falsas:

| Afirmación | Hecho |
|---|---|
| «el schema … está vacío» | 26 documentos y 13.876 chunks en la base, 0 sin vector (`psql`) |
| «`services/retrieval.py` no existe todavía» | Existe, con dos modos y fusión RRF (`app/services/retrieval.py:230-298`) |
| «`/search` sin implementar» | `app/main.py:28` lo registra; aparece en `GET /openapi.json`; responde en vivo (probado con `curl`, `mode=hybrid`, 2 hits) |

**R2 · `ARCHITECTURE.md:110-113` se contradice con `ARCHITECTURE.md:1178`, dentro del mismo
documento.** §3 afirma: «La capa de datos … está construida pero **fuera de ese flujo**: ningún
router ni service la importa todavía. Se enchufará cuando `services/retrieval.py` exista». §5
afirma lo contrario y es lo correcto: «`POST /api/v1/search` es **el primer punto del camino de
petición que toca la base**, y por eso `get_session` se cablea en ese router». El hecho:
`app/routers/search.py` importa la dependencia de sesión y consulta la base.

**R3 · `ARCHITECTURE.md:76-104` — el árbol de `app/` omite cinco módulos que existen:**
`routers/search.py`, `services/retrieval.py`, `services/ingestion.py`, `services/embedding.py`
y `schemas/search.py`. Los cuatro primeros están descritos en prosa más abajo (§5), así que el
desfase es del árbol, no del documento entero.

**R4 · `ARCHITECTURE.md:657-660` y el diagrama de `ARCHITECTURE.md:16-22`.** §5 dice que
`corpus/` «solo se conecta a `app/`, cuando `services/retrieval.py` empiece a leer de
`data/processed/`». Doble desfase: `retrieval.py` ya existe, y **no lee `data/processed/`** —
lee la base de datos. El puente real es `app/services/ingestion.py`, un batch job. El diagrama
de §1 (`data/processed/*.json ◄──(consume)── app/`) describe por tanto un acoplamiento que hoy
ya no es el del camino de petición.

**R5 · `ARCHITECTURE.md:624-651` — el árbol de `corpus/` omite `embed_all.py` y
`diagnostics/embedding_sanity.py`.** Ambos existen, ambos están documentados en prosa en el
mismo §5 (línea 882) y en `corpus/RUNBOOK.md` (pasos 10 y 11).

**R6 · `CLAUDE.md:663-667` y `ARCHITECTURE.md:1317-1320` — el desglose por documento de las 189
tablas es incorrecto.** Los docs dicen «120 en `ECSS-E-ST-70-41C`, 49 en `ECSS-E-ST-70-31C`».
Medido hoy sobre `data/processed/`: **187 en `ECSS-E-ST-70-41C`, 1 en `ECSS-E-HB-40-01A`, 1 en
`ECSS-E-HB-40A`, y 0 en `ECSS-E-ST-70-31C`**. El total (189) coincide; la atribución no. El
número parece de antes del re-parseo de S06 brief 4b.

**R7 · `CLAUDE.md:710-712` — «Pendiente de medir el lado query … y es S09» ya no es cierto.**
S08 brief 5 lo midió y el resultado está commiteado en `scripts/output_examples.txt:120-152`
(arquetipo 5, query «difference between software criticality category B and C»): el chunk
discriminante está en el puesto 3 en modo vector y sube al 1 en híbrida, y la tabla D-1 real
entra en el puesto 2 desde fuera del top-5. El propio `CLAUDE.md` lo recoge 600 líneas más
arriba, en «Estado actual» y en el item D14 — el item D17 quedó sin actualizar.

**R8 · `CLAUDE.md:682-688` subestima la severidad de la deuda que describe.** El item dice que
cinco módulos «verían hoy 28 o 29 documentos donde hay 26». Dos de ellos no ven de más: **fallan
en seco**. `uv run python -m corpus.diagnostics.inspect_graph` termina en
`KeyError: 'sections'` (`corpus/parser.py:797`, al hacer `load_parsed` sobre `chunks.json`), y
`corpus/graph.py:403-404` tiene exactamente el mismo patrón, así que
`uv run python -m corpus.graph` falla igual. **Consecuencia operativa: `references.json` y
`graph.graphml` no se pueden regenerar hoy.**

**R9 · `CLAUDE.md:144-151` — el árbol de `app/services/` omite `embedding.py`**, que existe y
está documentado en «Decisiones consolidadas» del mismo archivo («Un seam de embeddings, no un
wrapper multi-proveedor»).

**R10 · `CLAUDE.md:226-228` — el árbol de `scripts/` está incompleto.** Lista solo
`query_examples.py` y `output_examples.txt`. Existen además `measure_divergent_impact.py`,
`sample_for_groundtruth.py`, `recall_groundtruth.py` y `score_groundtruth.py`. Los tres últimos
son el arnés de ground truth del extractor — es decir, el instrumento de la deuda D29, invisible
en el árbol que debería mostrarlo.

**R11 · `corpus/RUNBOOK.md:266-275` documenta un comando que falla.** El paso 7 es
`uv run python -m corpus.graph`, y hoy termina en `KeyError: 'sections'` (misma causa que R8).
El RUNBOOK es el manual de operación del pipeline, así que la primera persona que siga la
secuencia se estrella ahí.

**R12 · `corpus/diagnostics/diagnose_numbering.py:18-19` — docstring anterior a la decisión que
describe.** Dice «nothing here is production numbering. It resolves the list to compare, not to
replace the parser (that decision is Brief 5)». El brief 5 decidió a favor del resolver, que
está en producción: `corpus/parser.py` numera desde `corpus/numbering.py`.

### Dirección 2 — el código hace algo estructural que los docs no reflejan

**R13 · La ingesta a pgvector no está en el `RUNBOOK`.** `corpus/RUNBOOK.md:12-27` enumera la
secuencia 0→11 y termina en `embedding_sanity`. El paso que mete el corpus en la base de datos
(`uv run python -m app.services.ingestion`) no aparece. Está documentado solo en
`ARCHITECTURE.md:1121`. `CLAUDE.md` presenta el RUNBOOK como «el comando exacto de cada paso,
qué lee y escribe, y cómo verificar que salió bien», así que el hueco es estructural: el paso
que cruza la frontera `corpus/` → `app/` es justo el que no tiene entrada.

**R14 · El arnés de ground truth del extractor existe y ningún doc de estado lo menciona.**
`scripts/sample_for_groundtruth.py`, `scripts/recall_groundtruth.py`,
`scripts/score_groundtruth.py`, `tests/corpus/test_groundtruth_harness.py` y
`data/groundtruth/` (dos CSV: `precision_sample.csv`, `recall_ECSS-Q-ST-80C_Rev.2.csv`) están
commiteados y cableados, y `RUNBOOK.md:306-521` documenta el procedimiento completo (§8.1–8.4).
Pero tanto `CLAUDE.md:995-996` como `ARCHITECTURE.md:1321-1322` redactan la deuda como «validar
con ground truth etiquetado» sin decir que el instrumento ya está construido y que **lo único
que falta es el etiquetado humano**. Estado real medido: 1 de 50 filas de precisión etiquetada
(49 en blanco), 0 filas de recall confirmadas, gate en FAIL con precisión 0,0%.

**R15 · `.env` conserva tres variables muertas: `CACHE_TTL`, `REDIS_URL` y `LLM_PROVIDER`.**
Ninguna está en `.env.example`, ninguna está declarada en `app/config.py`, y ninguna se lee en
el código. Las tres contradicen afirmaciones explícitas de los docs («No hay `LLM_PROVIDER`»,
`CLAUDE.md:869`; «Sin cache de respuestas ni Redis», `CLAUDE.md:379`). Son residuo inerte —
`Settings` declara `extra="ignore"` (`app/config.py:15`), así que no rompen nada — pero
`.env.example` y `.env` han divergido.

---

## 5 · Observaciones

Sin clasificar y sin veredicto. Cosas que conviene tener delante al decidir en el Paso B.

**O1 · Un hallazgo soft de `verify_parser` que ningún doc registra.** La corrida deja
`25/26 sin hallazgos`, con `ECSS-E-ST-10C_Rev_1.docx` marcado:
`FAIL INV5(soft): 1 requirements start lowercase (check for truncation)`. El exit code es 0 y
el invariante es blando a propósito, pero es un aviso de posible truncado en el documento que
también carga los 10 chars de `require` desnudo de D02 y las 344 recuperaciones de control de
cambios más altas del corpus. Puede no ser nada; hoy nadie lo ha mirado.

**O2 · `diagnose_numbering` reporta 127 divergencias contador-vs-XML donde los docs hablan de
469.** `CLAUDE.md:411-414` dice «Contar daba **469** cláusulas con número erróneo y 41 rótulos
numerados sin serlo; resolver da 0 y 0». La corrida de hoy da **41 rótulos** ✅ y **0
fabricados** ✅ —las dos afirmaciones vivas—, pero **127** divergencias del contador simulado, no
469. La cifra 469 es una medición histórica de S06 sobre otro estado del parser, y está
redactada en pasado, así que no es una afirmación falsa; conviene saber que el número no se
reproduce con la herramienta actual.

**O3 · `CHANGELOG.md` tiene dos entradas `## [0.26.0]`**, en las líneas 249 (2026-09-01) y 303
(2026-09-04). Un número de versión duplicado en el único documento que es el histórico.

**O4 · No hay bloque `Unreleased` en `CHANGELOG.md`**, y el árbol está limpio con S08 cerrada en
`[0.31.0]`. Coherente con el estado, pero el checklist de fin de sesión de `CLAUDE.md:923` pide
que «el bloque **Unreleased** refleje el estado real» — hoy no existe bloque que reflejar.

**O5 · D13, D29 y R11/R13 se tocan entre sí.** La deuda de la lista de exclusión (D13) es la que
rompe `graph.py`; `graph.py` roto es lo que impide regenerar `references.json`; y
`references.json` es el artefacto cuya precisión mide el arnés de ground truth sin etiquetar
(D29). Tres entradas de deuda documentadas por separado que hoy forman una sola cadena rota
alrededor del grafo. **Corazonada, no veredicto:** cualquiera que sea la disposición, tienen
orden entre sí.

**O6 · D17 y D14 son la misma medición contada dos veces, con signos opuestos.** D17 dice que
el vector no separa B de C y deja pendiente medir el lado query; D14 dice que la híbrida sí lo
arregla pero degrada la genérica. La medición de S08 brief 5 responde a los dos. **Corazonada:**
puede que sean un item, no dos.

**O7 · D25 (sesiones en memoria, un solo worker) no se manifiesta hoy y podría hacerlo sin
aviso.** `docker-compose.yml:61` y `Dockerfile:65` arrancan `uvicorn` sin `--workers`, así que
el multi-turn funciona. Añadir `--workers N` en un despliegue lo rompería en silencio: no hay
guardarraíl de arranque que lo impida, al contrario que con `LLM_EVAL_MODE` + `APP_ENV=production`.

**O8 · El régimen exacto se sostiene y está verificado.** `pg_indexes` sobre `chunks` y
`documents` da 8 índices, ninguno vectorial: dos PK, dos UNIQUE de clave natural, la FK
`ix_chunks_document_id`, los dos GIN de `metadata` y el GIN de `cite_tsv`. Es exactamente lo que
los docs describen; lo anoto porque es la afirmación más fácil de erosionar sin darse cuenta.

**O9 · Ningún item de la deuda documentada resultó `RESUELTA` salvo D27, y D27 es de
`ARCHITECTURE.md`, no de `CLAUDE.md`.** Las 24 entradas de `CLAUDE.md` § "Decisiones pendientes"
siguen vigentes una por una. Dicho de otro modo: la deriva de este proyecto no está en la lista
de pendientes —que está bien mantenida— sino en `ARCHITECTURE.md`, en los árboles de directorio
de ambos archivos y en el `RUNBOOK`.

---

## 6 · Qué NO se hizo, y por qué

Conforme al alcance del brief:

- **No se editó ningún doc, código ni migración.** El único archivo creado es este informe.
- **No se clasificó ninguna deuda** como transversal / futura / fantasma. Los estados son los
  cuatro del vocabulario, sin sinónimos.
- **No se buscaron code smells ni refactors.** El código se leyó solo para verificar
  afirmaciones de los docs y estado de deuda.
- **No se commiteó nada.**
- **No se ejecutó `corpus/graph.py`, `corpus/report.py` ni `corpus/parse_all.py`** aunque
  habrían dado hechos útiles: los tres **escriben** artefactos (`references.json`,
  `graph.graphml`, `processing-report.md`, `data/processed/*.json`). Para `report.py` se llamó
  a `build_report()` en memoria, sin `save()`, y así se estableció el hecho sin tocar el
  artefacto commiteado. Para `graph.py` bastó con leer el código: `:403-404` es idéntico al de
  `inspect_graph.py:45-46`, que sí se ejecutó y falló.
- **Se ejecutó `corpus.diagnostics.embedding_sanity`, que llama a la API real** (media docena de
  textos cortos con `text-embedding-3-small`, coste despreciable). Es un diagnóstico existente y
  read-only, y era la única forma de reproducir D17 en vez de citarlo de oídas. Se reprodujo al
  dígito.
- **No se llamó a `POST /api/v1/query`** para no gastar una llamada de LLM: el hecho que
  interesaba (M1 sigue sobre 1 fragmento estático) se estableció importando `ECSS_REFERENCE`
  directamente.

### Lo que queda para el usuario

Un único punto, **caso 1 de la frontera** (`CLAUDE.md` § Tests): lo que CC no puede ejecutar por
sí mismo.

- **El etiquetado del ground truth del extractor de referencias (D29).** No existe oráculo de
  «esta referencia es correcta»: el insumo es una persona leyendo el documento. El instrumento
  está listo y el procedimiento documentado en `corpus/RUNBOOK.md:306-521`. Estado actual: 1 de
  50 filas de precisión etiquetada, 0 de recall. **Esto es un hecho de la auditoría, no una
  petición**: si se etiqueta o no, y cuándo, es disposición del Paso B.

No hay ningún punto de caso 2 (decisión o input del usuario) pendiente para cerrar este paso:
la auditoría está completa.
