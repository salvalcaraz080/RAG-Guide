# AUDIT-PRE-M2 — Auditoría de código pre-M2

**Fecha:** 2026-07-19 · **Sesión:** solo lectura (brief `RAG-Guide/sessions/brief-auditoria-pre-M2.md`)
**Método:** lectura completa de `app/`, `frontend/`, `tests/`, greps exhaustivos, inspección
read-only de `data/processed/` y ejecución de la suite de contrato (`uv run pytest`).
No se ha modificado ningún archivo salvo este informe. Toda decisión sugerida por un hallazgo
queda **pendiente de decisión en Claude Chat**.

Nota previa: el árbol de trabajo ya contenía modificaciones sin commitear **antes** de esta
sesión (los mismos 30 ficheros listados en el `git status` final). Esta auditoría se ha hecho
sobre ese estado del working tree, no sobre `HEAD`.

---

## Parte A — Doble-check de la auditoría documental

### A1. Estado actual

#### A1.1 — Recreación de sesión en el frontend · `MATIZADO`

La recuperación **no es silenciosa**, pero el aviso no dice que el contexto se perdió.
Fragmento exacto ([frontend/streamlit_app.py:160-175](frontend/streamlit_app.py#L160-L175)):

```python
elif event["event"] == "error":
    error_message = event["data"]["error"]
    # Si el backend perdió la sesión (reinicio, historial en memoria), se
    # descarta el `session_id` local para que la próxima pregunta cree otra.
    if "session not found" in error_message.lower():
        st.session_state.session_id = None
...
if error_message:
    st.error(error_message)
    st.session_state.messages.append(
        {"role": "assistant", "content": f"⚠️ Error: {error_message}"}
    )
```

- Sí visible: `st.error(...)` muestra el mensaje del backend ("session not found or expired;
  create a new session") y queda anclado al historial visible.
- El matiz: el mensaje es el genérico del backend; **no** dice explícitamente "la conversación
  se ha reiniciado y la siguiente pregunta empezará sin contexto". Además el chat visible
  (`st.session_state.messages`) sigue pintado, lo que sugiere continuidad que el servidor ya
  no tiene. La detección por substring `"session not found"` acopla el frontend al texto del
  mensaje de `SessionNotFoundError` ([app/services/sessions.py:63](app/services/sessions.py#L63)) —
  si ese texto cambia, la recuperación deja de dispararse en silencio.

#### A1.2 — `LLM_PROVIDER` vestigial · `CONFIRMADO`

Grep exhaustivo sobre `**/*.py`: la única aparición es su definición en
[app/config.py:25](app/config.py#L25). **Nadie la lee.** El Router hardcodea los prefijos de
proveedor en `model_list` ([app/services/llm_wrapper.py:29,42](app/services/llm_wrapper.py#L29)):
el primario es siempre `anthropic/{LLM_MODEL}` y el fallback siempre
`openai/{LLM_FALLBACK_MODEL}`, independientemente de `LLM_PROVIDER`. Es config muerta,
también listada en `.env.example:16` y CLAUDE.md como si tuviera efecto.

#### A1.3 — Flujo real de una petición · `CONFIRMADO` (con 1 matiz de nomenclatura)

Orden real reconstruido de `answer_query` ([app/services/llm_service.py:373-460](app/services/llm_service.py#L373-L460)):

1. `run_input_guards(question)` (L398)
2. `_resolve_session(session_id)` (L400)
3. `cache_eligible = _cache_eligible(history)` (L402)
4. Si elegible: `_cache_lookup` → HIT: `_append_session_turn(...)` + return (L405-415)
5. Si no elegible: `_build_context_block(ECSS_REFERENCE)` (L417)
6. `_compose_system_prompt(context_block)` + `_call_model(system, question, history)` (L420-424)
7. `_assemble_response` (citas incluidas) (L442)
8. `cache.set` solo si elegible (L452-453)
9. `_append_session_turn` (L455) → return

`stream_answer_query` es idéntico en orden (L510-521: guards → resolve, convertidos a
`event: error` en vez de raise). **Coincide con los 12 pasos de ARCHITECTURE.md §3.**

Punto de atención del brief: **los guards corren de verdad antes de `_resolve_session`** —
antes de CUALQUIER efecto — en ambos caminos, y el orden está cementado por test con
discriminador de tipo de excepción:
`test_multi_turn.py::test_moderation_runs_before_session_resolution` (+ gemelo de stream).

Matiz: el paso 8 de la doc nombra `build_system_prompt(ECSS_REFERENCE)`, pero el camino de
producción usa `_compose_system_prompt(context_block)` (L420, L542); `build_system_prompt`
([llm_service.py:58](app/services/llm_service.py#L58)) produce el mismo string pero hoy solo
lo llaman los tests (ver B7).

#### A1.4 — Aislamiento de tests · `CONFIRMADO`

Las tres fixtures existen en [tests/app/conftest.py](tests/app/conftest.py), las tres
`autouse=True`, y cubren lo que dicen:

- `isolate_cache_from_real_redis` (L28-39): fuerza `llm_service.cache._client = None`.
- `isolate_input_guards_from_real_moderation` (L51-65): fuerza `_moderation_client = None`.
- `isolate_session_store` (L68-83): limpia `_sessions` antes Y después de cada test.

Ningún test las desactiva hacia un recurso real dentro de la suite por defecto. Los dos
únicos opt-outs son deliberados y seguros: `fake_cache` (doble en memoria, no Redis real) y
`real_moderation_client` ([test_input_guards.py:244-252](tests/app/test_input_guards.py#L244-L252)),
que restaura el cliente real **solo** para los dos tests `@pytest.mark.real_llm`, excluidos
por `addopts = "-m 'not real_llm'"` ([pyproject.toml:54](pyproject.toml#L54)).

### A2. Invariantes que la doc da por implementados

#### A2.1 — Citas: construcción y supresión en rechazo · `CONFIRMADO`

- (a) `_citations` deriva exclusivamente de `ECSS_REFERENCE`
  ([llm_service.py:277](app/services/llm_service.py#L277)); nunca parsea la respuesta.
- (b) Rechazo → `[]` ([llm_service.py:275-276](app/services/llm_service.py#L275-L276)).
- (c) Comparación byte a byte: las constantes
  `_OUT_OF_DOMAIN_MARKER = "Out of scope:"` y
  `_OUT_OF_MATERIAL_MARKER = "Not covered by the provided material:"`
  ([llm_service.py:238-239](app/services/llm_service.py#L238-L239)) son **idénticas** a las
  frases que `system.md` instruye ([app/prompts/query/system.md:18-19](app/prompts/query/system.md#L18-L19)).
  `startswith` case-sensitive, cementado por `TestRefusalKind::test_is_case_sensitive`.
  ⚠️ Pero ver **B6**: el propio `system.md` contiene una segunda frase de rechazo (pre-S04)
  que NO es marcador reconocido.
- (d) Punto de ensamblado único: `_assemble_response` se usa en L442 (no-stream) y L588
  (stream); no hay copia. Verificado además por
  `test_streaming.py::test_stream_citations_match_non_stream_path`.

#### A2.2 — Compuerta de cache · `CONFIRMADO`

`_cache_eligible` es literalmente
`return len(history) == 0 and not settings.LLM_EVAL_MODE`
([llm_service.py:370](app/services/llm_service.py#L370)). Grep de `cache.get(`/`cache.set(`
en `app/`: `get` solo en `_cache_lookup` (L218); `set` en L453 y L599, ambos tras
`if cache_eligible`. El único otro acceso es `cache.ping()` en
[app/main.py:20,43](app/main.py#L20) (lifespan/health — conectividad, no elegibilidad).
El HIT de turno 1 anexa con `cached["answer"]` (L410, L528), cementado por
`test_multi_turn.py::test_turn1_appends_even_on_cache_hit`.

#### A2.3 — Fidelidad por envoltura · `CONFIRMADO`

`GET /api/v1/prompt` → `get_active_instructions()` devuelve la **misma constante de módulo**
`_STATIC_INSTRUCTIONS` que usa `_compose_system_prompt` (L70-77 y L55); no relee el archivo
(test: `test_get_active_instructions_matches_module_constant`). `injected_context` se puebla
en los 4 puntos de retorno (L414, L459, L536, L612) desde el MISMO `context_block` que
compuso el prompt de esa request. El camino cache-inelegible reconstruye `context_block` con
la misma función pura sobre el mismo dato (L417, L540) — en M1 es el mismo string por
construcción; ARCHITECTURE.md ya marca este punto como revisable si el contexto de M2 deja
de ser determinista. No hay ninguna otra derivación del contexto ni del prompt en el repo.

#### A2.4 — Wrapper y fallback · `MATIZADO`

- ✅ Solo `_call_model` cruza el seam de **completions**; `router.acompletion` en ambos
  caminos ([llm_wrapper.py:141, 204](app/services/llm_wrapper.py#L141)) — `.completion`
  síncrono no aparece.
- ✅ `_provider_from_model` usa `litellm.get_llm_provider()` (L254), no substring.
- ✅ `stream_options={"include_usage": True}` en el camino de streaming (L209).
- ❌ El matiz: **`llm_service.py` SÍ importa litellm**
  ([llm_service.py:4](app/services/llm_service.py#L4)) — para `litellm.cost_per_token` en
  `_cost_usd` (L125). La afirmación de ARCHITECTURE.md §3 ("`llm_service` nunca importa
  LiteLLM directamente") es literalmente falsa desde S03 brief 4. El seam de proveedor se
  respeta para las llamadas al modelo; el cálculo de coste lo perfora. Consecuencia: un
  cambio de librería de routing en M3 tocaría `llm_service`, y la doc sobrevende el
  aislamiento.

#### A2.5 — Test-cemento en el punto crítico · `CONFIRMADO` (limpio)

Tests que protegen el contrato de citación, por nombre y aserto:

| Test | Qué aserta |
|---|---|
| `test_queries.py::test_query_grounding_not_covered_does_not_fabricate_citation` | rechazo → `citations == []` vía HTTP |
| `test_llm_service.py::TestCitationsCutOnRefusal` (2 tests) | `_citations(rechazo) == []` para ambos marcadores |
| `test_llm_service.py::TestRefusalKind` (4 tests) | clasificación por prefijo exacto, case-sensitive |
| `test_streaming.py::test_stream_refusal_has_no_citations` | mismo corte en `done` del stream |
| `test_system_prompt_provider_agnostic.py` (6 tests, `real_llm`) | modelo real emite el marcador y `_citations` real corta |
| `test_queries.py::test_query_returns_populated_citations` | respuesta normal → citas == coordenadas de `ECSS_REFERENCE` |

Todos asertan el comportamiento **correcto** (rechazo sin cita), no el vigente-por-inercia.
El único test que cementa deliberadamente una decisión discutible es
`test_llm_service_cache.py::test_model_change_does_not_invalidate_cache`, y su docstring
declara que es un pin intencionado de la decisión post-brief-2 (model no es slot). No se ha
encontrado ningún "test que aserta la fabricación como esperada".

### A3. Riesgos M1→M2 contra el código real

#### A3.1 — Puntos de contacto del swap de contexto · `CONFIRMADO` con inventario

En **producción**, `ECSS_REFERENCE`/`CORPUS_VERSION` solo los conoce
`app/services/llm_service.py` — pero en **4 sitios internos**, no 1:

- import (L8); `_cache_lookup` → `_build_context_block(ECSS_REFERENCE)` + `CORPUS_VERSION` (L211, L216)
- `_citations` — deriva las citas de `ECSS_REFERENCE` global (L277)
- `answer_query` camino inelegible (L417)
- `stream_answer_query` camino inelegible (L540)

Ni el frontend, ni los routers, ni `build_cache_key` (recibe strings), ni el endpoint de
prompt (solo instrucciones) lo conocen. En **tests**, 8 ficheros lo importan o monkeypatchean
(`test_llm_service.py`, `test_queries.py`, `test_prompt_endpoint.py`, `test_streaming.py`,
`test_llm_service_cache.py` — que monkeypatchea `ECSS_REFERENCE` y `CORPUS_VERSION` —,
`test_query_integration.py`, `test_system_prompt_provider_agnostic.py`,
`test_system_prompt_snapshot.py` — golden del prompt completo, fragmento incluido).
El plan "M2 solo toca `llm_service`" se sostiene a nivel de módulo, pero dentro del módulo
hay 3 call sites de contexto duplicados + `_citations` acoplada al global (ver B9), y el
golden snapshot habrá que partirlo.

#### A3.2 — Placeholders M2 · `REFUTADO` (la doc los inventa)

**No existen.** Glob completo de `app/**/*.py`: no hay `services/ingestion.py`,
`services/embedding.py`, `services/retrieval.py`, `routers/ingestion.py`,
`schemas/document.py` ni directorio `models/`. Nada los importa. El árbol de carpetas de
CLAUDE.md los dibuja como existentes ("(M2)"), lo cual es aspiracional, no descriptivo.
Consecuencia positiva: **cero riesgo de placeholder con firma equivocada** — M2 parte de
hoja en blanco. Consecuencia menor: la doc debe dejar de listarlos como presentes.

#### A3.3 — Multi-turn: qué llega al modelo · `CONFIRMADO`

`_compose_messages` ([llm_wrapper.py:94-116](app/services/llm_wrapper.py#L94-L116)) produce
exactamente `[{system}, *history, {user actual}]`; `Message` es `{role, content}` estricto
(`Literal["user","assistant"]`, [sessions.py:20-33](app/services/sessions.py#L20-L33)) — sin
citas ni metadata. Cementado por `test_sessions.py::TestHistoryComposition`.

Para el punto de inserción de retrieval: en ambos caminos la sesión se resuelve **antes** de
construir el contexto (L400 antes de L405/417), y `_cache_lookup` recibe la pregunta cruda
(L406). Es decir, en el punto donde M2 insertará retrieval están disponibles **la pregunta
cruda Y el historial ya resuelto** — la decisión pendiente de condensación (¿recuperar con
la pregunta sola o condensada con historial?) no está condicionada por el código: ambas
opciones son alcanzables sin reordenar el flujo.

#### A3.4 — Grafo y corpus · `MATIZADO`

No se ejecutó `python -m corpus.graph`: su `__main__` **reescribe**
`references.json` y `graph.graphml` ([corpus/graph.py:297-300](corpus/graph.py#L297-L300)),
así que no es un verificador de solo lectura. Tampoco `verify_parser`. En su lugar,
inspección read-only de los artefactos:

- **Stats: COINCIDEN.** `references.json`: 273 referencias. `graph.graphml`: 326 nodos,
  247 aristas. `data/processed/`: 14 JSON de documento (+ `references.json` + `graph.graphml`).
- **Shape: MATIZADO.** `references.json` es una **lista** de registros
  `{source_clause_id, source_doc_id, source_uid, target_doc_id, target_clause_id,
  mention_text, mention_type}` — **no** un dict keyed por `(source, target)` como sugiere la
  doc ("procedencia rica … keyed por (source, target)"). El índice de adyacencia
  `{source: [targets]}` del contrato de runtime M2 es construible trivialmente desde la
  lista, pero el key no existe hoy como tal.
- **Dato nuevo relevante para M2:** **135 de 273 referencias (49%) tienen
  `target_clause_id` vacío** (mención a documento entero, sin cláusula). La expansión a
  1 salto solo puede aterrizar en cláusula concreta en la mitad de los casos; en la otra
  mitad el target es un documento completo — esto condiciona el diseño del enriquecimiento
  de contexto (¿qué se inyecta cuando el target es un doc entero?) y debería entrar en la
  decisión de chunking. También: 134/273 sin `source_uid` (consistente con "~mitad del
  corpus sin UID"). Tipos: 146 normative_ref, 70 intra_doc_ref, 45 definition_ref,
  12 informative_ref.

#### A3.5 — Tablas omitidas: la deuda, dimensionada · hecho

- El parser **no deja ninguna traza** de las tablas que omite: grep de `table|Table` en
  `corpus/parser.py` → 0 resultados. Ni conteo, ni warning, ni campo en el JSON — la omisión
  es totalmente silenciosa.
- Conteo read-only (python-docx, `len(Document(...).tables)`) sobre los 14 DOCX crudos de
  los documentos procesados: **569 tablas en total, los 14 documentos tienen al menos una**.
  Distribución: ECSS-E-ST-70-41C **389** (su contenido es esencialmente definiciones
  tabulares de paquetes TM/TC — un documento RELATED casi ilegible sin tablas),
  ECSS-E-ST-40C 39, ECSS-E-ST-40-07C 24, ECSS-E-ST-10-03C 19, ECSS-E-HB-40-01A 18,
  ECSS-Q-ST-80C 18, resto entre 4 y 13. R2.6 no es deuda marginal: para E-ST-70-41C es
  la mayor parte del contenido normativo.

---

## Parte B — Auditoría independiente

### B1 — Corrección async · limpio

- Cliente de moderación: `AsyncOpenAI` real ([input_guards.py:23](app/services/input_guards.py#L23)),
  llamada `await ... .moderations.create(...)`.
- Sin `time.sleep`, sin `requests`, sin SDK síncrono en `app/` (greps negativos).
- Streams iterados con `async for` en ambos tramos.
- Único trabajo síncrono en camino caliente: `litellm.cost_per_token` (lookup de price map
  local, CPU), `sha256`/`json.dumps` de la clave — negligible, no I/O.
- Import time: lectura de `system.md` (~2 KB), construcción del Router y de `Settings` —
  una vez por proceso, sin red (LiteLLM no conecta al construir el Router). Nada bloqueante
  reseñable.

### B2 — Manejo de errores real vs declarado

- **(media) `SessionNotFoundError` puede abortar el stream SSE sin evento tipado.**
  `_append_session_turn` corre **fuera** del `try` que protege el stream del proveedor
  ([llm_service.py:551-567](app/services/llm_service.py#L551-L567) protege; el append está
  en L601, y en el HIT en L528). Si la sesión desaparece entre `_resolve_session` y el
  append (hoy solo por borrado concurrente; con TTL de sesión en S06 la ventana se vuelve
  real), `append_turn` lanza ([sessions.py:123-125](app/services/sessions.py#L123-L125)) y
  la excepción escapa del generador → conexión SSE cortada sin `event: error`. En `/query`
  no-stream el router sí la captura → 404, pero un 404 emitido **después** de generar y
  cachear la respuesta es semánticamente engañoso (coste gastado, respuesta descartada).
  Sin test que cubra este camino.
- **(baja) Mensajes de excepción crudos hacia el cliente.** `str(exc)` del proveedor viaja
  en `event: error` (L513, L521, L566) y en `detail` de los HTTPException. Un error de
  LiteLLM puede arrastrar detalles internos (URLs, request-ids del proveedor). Aceptable en
  demo; a revisar antes de exponer a terceros.
- Limpio en lo demás: cero `except:` desnudos y cero `except: pass` en `app/`; los tres
  caminos degradables devuelven estados inequívocos (`None` de `cache.get` = miss — un
  llamador no puede confundirlo porque un hit siempre es dict; moderación degrada con
  `return` explícito y warning; `_cost_usd`/`_provider_from_model` degradan a
  `None`/`"unknown"` documentados y logueados).

### B3 — Estado global y concurrencia

Inventario de singletons de módulo: `get_settings()` (lru_cache),
`llm_service.cache` (LLMCache), `llm_wrapper._router`, `input_guards._moderation_client`,
`sessions.session_store`, `llm_service._STATIC_INSTRUCTIONS`. Todos construidos en import,
coherente con el patrón documentado.

- `append_turn` es **síncrono de principio a fin** (sin `await` interno) → atómico respecto
  al event loop: dos requests concurrentes de la misma sesión no pueden intercalar los dos
  `append` de un mismo par; el historial mantiene siempre la forma `[u,a,u,a,…]`.
- La ventana deslizante **reasigna en sitio** (`history[:] = history[-max:]`,
  [sessions.py:133](app/services/sessions.py#L133)) — evita la re-validación de asignación
  de Pydantic, como anota la Guide. Cardinalidad correcta: `max_messages = turnos × 2`.
- **(baja) `_resolve_session` devuelve la lista viva, no una copia**
  ([llm_service.py:331](app/services/llm_service.py#L331)). En la práctica la composición de
  mensajes ocurre sin ceder el event loop entre resolución y composición (el camino
  multi-turn no tiene `await` entre ambas), así que cada request ve un snapshot coherente.
  El riesgo residual real es semántico, no estructural: dos requests concurrentes de la
  misma sesión generan ambas sin ver el turno de la otra, y los pares aterrizan en orden de
  llegada. No hay corrupción; con un solo worker y un cliente (Streamlit) es teórico. Nota:
  el propio test `test_multiturn_forwards_history_to_seam` documenta la mutación
  ("Se captura una COPIA del historial: la lista real se muta luego").

### B4 — Divergencias config / doc

Diff de tres vías `Settings` ([app/config.py](app/config.py)) vs `.env.example` vs CLAUDE.md:

| Variable | Settings | .env.example | CLAUDE.md | Nota |
|---|---|---|---|---|
| `LLM_PROVIDER` | ✅ default `anthropic` | ✅ | ✅ | **config muerta** (A1.2) — en los 3 sitios como si tuviera efecto |
| `LOG_LEVEL` | default `INFO` | `DEBUG` | `DEBUG` | valor activo ≠ default; sin efecto real, pero CLAUDE.md lo presenta como "activo" sin mencionar el default |
| `EMBEDDING_*`, `DATABASE_URL` | ausentes | ✅ | ✅ (M2+) | coherente: `extra="ignore"` las tolera |
| resto (12 campos) | ✅ | ✅ | ✅ | alineados, defaults idénticos |

Ningún otro campo de `Settings` está sin lector (todos verificados). Extra encontrado:
**(baja)** `pyproject.toml:4` describe el proyecto como *"AI-powered software estimation
with RAG over ECSS space standards"* — descripción heredada del proyecto hermano
(estimador LIDR); este proyecto no es de estimación.

### B5 — Calidad de la suite de contrato

- (a) **Sin dependencia del proveedor vivo**: los asserts de `provider == "anthropic"/"openai"`
  en `test_llm_wrapper.py`/`test_query_integration.py` son sobre deployments con
  `mock_response` determinista — testean el mecanismo de fallback, no la varianza de quién
  respondió. El invariante se respeta.
- **(media) La suite de contrato exige `OPENAI_API_KEY` presente para pasar.** La fixture
  `mock_deployments` ([test_llm_wrapper.py:20-26](tests/app/test_llm_wrapper.py#L20-L26))
  hace `_deployment_for(_FALLBACK_MODEL_NAME)`, que lanza `AssertionError` si el deployment
  no existe — y sin `OPENAI_API_KEY` el Router arranca sin fallback
  ([llm_wrapper.py:37-56](app/services/llm_wrapper.py#L37-L56)). En un clone fresco con solo
  `ANTHROPIC_API_KEY`, 3 tests de `test_llm_wrapper.py` fallan. No toca red, pero
  contradice "la suite corre sin depender del entorno": la presencia de una key opcional
  cambia el veredicto. (Los tests podrían skippearse con un `pytest.mark.skipif` sobre la
  config, o inyectar el deployment — pendiente de decisión.)
- (b) Asserts débiles: puntuales y de bajo riesgo (`assert data["model"]` en
  [test_queries.py:49](tests/app/test_queries.py#L49)); los contratos serios (citas, orden,
  compuertas) se asertan con igualdad estricta.
- (c) **Caminos de error de streaming legibles por el cliente: cubiertos los tres** —
  toxicidad (`test_flagged_input_stream_emits_error_with_no_chunks`), sesión ausente
  (`test_provided_but_absent_session_stream_emits_error_event`), fallo de proveedor a mitad
  de stream (`test_provider_error_mid_stream_emits_typed_error_event`) — todos vía
  TestClient parseando SSE real, según la lección del bug del cache-hit. El hueco es el
  hallazgo B2: el append fallido post-stream no tiene test (coherente: es un camino hoy sin
  manejo).
- (d) Degradación cubierta: Redis caído (`test_degrades_without_redis`,
  `test_degrades_when_client_raises`), moderación caída/ausente
  (`test_no_openai_key_fails_open`, `test_moderation_api_error_fails_open`, + end-to-end).

### B6 — Consistencia del prompt como artefacto

- **(media) Contradicción interna en `system.md`: dos instrucciones de rechazo distintas.**
  La línea 3 (bloque pre-S04) instruye: *"If the question cannot be answered with that
  material, say so explicitly («The provided reference does not cover this»)"*, mientras el
  bloque S04 (líneas 11-21) exige empezar exactamente con `Not covered by the provided
  material:`. Si el modelo obedece la frase de la línea 3 (que el prompt le da entre
  comillas, como cita literal), `_refusal_kind` **no** detecta el rechazo → `_citations`
  adjunta la cita candidata → exactamente la cita fantasma que el fix S04-brief-1 existe
  para evitar. Mitigado porque la instrucción del marcador es posterior y más específica, y
  los 6 tests `real_llm` validan que ambos proveedores emiten el marcador — pero es una
  contradicción latente sensible a cambio de modelo, y el golden snapshot cementa **ambas**
  frases (pin del estado, no corrección). Las líneas 1-8 y 9-23 son en general dos bloques
  redundantes (rol y grounding se enuncian dos veces).
- El resto coincide con ARCHITECTURE.md: los 5 puntos presentes, marcadores exactos, el
  snapshot de caracterización existe (`test_system_prompt_snapshot.py`), y no queda ninguna
  constante pre-S04 con las instrucciones en código (solo el `.md`).

### B7 — Higiene general

- **(baja) `StreamErrorEvent` es código muerto**: definido en
  [app/schemas/query.py:87-90](app/schemas/query.py#L87-L90), cero usos — el router emite el
  evento `error` como dict crudo sin validarlo ([routers/queries.py:71-72](app/routers/queries.py#L71-L72)),
  mientras `done` sí se valida contra `StreamDoneEvent`. O se usa (simetría de validación) o
  sobra.
- **(baja) `build_system_prompt` no tiene llamadores de producción** — solo tests y
  docstrings; el camino real usa `_compose_system_prompt` (A1.3). No es muerto (es la API de
  composición que los tests ejercitan), pero la doc lo presenta como el paso del flujo.
- **(baja) ARCHITECTURE.md §4 desactualizado**: *"System prompt y contexto CAG activo se
  marcan `# TODO` — el backend no los expone hoy"* — falso desde S04 brief 2; el propio §3
  del mismo documento describe la implementación, y el sidebar los muestra
  ([streamlit_app.py:79-103](frontend/streamlit_app.py#L79-L103)). Deuda de la regla
  presente-vs-histórico dentro de ARCHITECTURE.md.
- TODOs en código: exactamente 2 ([input_guards.py:103](app/services/input_guards.py#L103),
  [llm_service.py:368](app/services/llm_service.py#L368)); ambos apuntan a brief futuro,
  ninguno usa la etiqueta literal `pendiente de decisión en Claude Chat` — son huecos
  planificados, no decisiones abiertas; aceptable.
- Frontend: **cero imports del backend en el código fuente** (solo
  `requests`/`sseclient`/`streamlit` + sus módulos locales) — el desacople se cumple en
  fuente, no solo por imagen.
- Logs "metadata, not content": limpio — ningún log de `app/` emite la pregunta ni la
  respuesta; moderación loguea solo categorías. Vigilar (info): `error=str(exc)` en
  `llm_call_failed` podría arrastrar contenido si un proveedor lo incluyera en su mensaje.
- `print()`: cero en `app/`/`frontend/`.

### B8 — Preparación real del aparcamiento de la cache

Estado hoy si `REDIS_URL` **no está definida**: `Settings` aplica el default
`redis://localhost:6379/0` ([config.py:34](app/config.py#L34)) — no existe un estado
"sin cache configurada". `redis.Redis.from_url` es perezoso ([cache.py:67-74](app/services/cache.py#L67-L74)),
así que `_client` casi nunca es `None` en producción; el `except` del constructor es
prácticamente inalcanzable. Consecuencia por request con Redis ausente:

- `cache.get` intenta conectar (connect timeout 1 s) → warning `cache_get_failed`;
- cada miss añade `cache_set_failed` → **2 warnings por request**, más el ping de `/health`;
- la latencia añadida depende del modo de fallo: `ECONNREFUSED` local es inmediato, pero un
  hostname no resoluble (p. ej. `redis` en Compose sin el servicio) puede pagar el timeout
  completo en **cada** get y set.

**No arranca "limpio con warning único": es una degradación permanente ruidosa.** Los tests
no asumen Redis (aislamiento por fixture) — y de hecho el estado "aparcado" que usan
(`_client = None`) es justo el que producción no puede alcanzar por configuración.

Qué haría falta (sin tocarlo, pendiente de decisión en Claude Chat) para que "cache
aparcada" sea un estado de primera clase:

1. `REDIS_URL: str | None = None` (o cadena vacía) en `Settings` → `LLMCache` con
   `_client = None` desde el arranque, un único log `cache_disabled` en startup.
2. Decidir qué reporta `/health` en ese estado (`"disabled"` explícito vs el actual
   `"degraded"`, que sugiere avería).
3. `.env.example`/CLAUDE.md documentando el estado apagado; `docker-compose.yml` sin el
   servicio `redis` (o comentado).
4. Nada que borrar: `cache.py` y sus tests quedan intactos como artefacto aparcado.

### B9 — Oportunidades que solo el código muestra (propuestas, no implementadas)

1. **Duplicación `answer_query` / `stream_answer_query`** — el preludio (guards → resolve →
   compuerta → lookup/contexto → prompt) y el cierre (coste/fallback/log/assemble/set/append)
   están duplicados con variaciones mínimas (raise vs yield). El swap M2 del contexto tocaría
   hoy los dos caminos en espejo. Propuesta: extraer un helper compartido de preparación
   (guards+sesión+compuerta+contexto+prompt) que ambos consuman. Trade-off: una indirección
   más en un módulo ya denso, a cambio de que M2 inserte retrieval en UN punto.
2. **`_citations()` acoplada al global** ([llm_service.py:277](app/services/llm_service.py#L277)):
   deriva de `ECSS_REFERENCE`, no de los fragmentos de ESA request. En M2 las citas deben
   salir de los chunks recuperados para esa pregunta → la firma tendrá que cambiar a
   `_citations(text, fragments)`. Decidir ese shape ANTES de M2 es barato; descubrirlo en
   medio del swap, no. Trade-off: ninguno real — es un parámetro más.
3. **`_build_context_block` en 3 call sites** (L211, L417, L540): el punto natural para el
   seam de retrieval es UNA función `get_context(question, history) -> fragments` que los
   tres consuman (se solapa con la propuesta 1; cualquiera de las dos lo resuelve).
4. **Golden snapshot del prompt completo** (`test_system_prompt_snapshot.py`) incluye el
   fragmento estático: con contexto dinámico en M2 habrá que partirlo en snapshot de
   instrucciones + test de composición. Anticipable al planificar M2, no urgente.

---

## Tabla-resumen

| # | Hallazgo | Severidad | ¿Bloquea M2? | ¿Decisión en Claude Chat? |
|---|---|---|---|---|
| 1 | `system.md` contiene dos instrucciones de rechazo contradictorias; la pre-S04 no es marcador reconocido → cita fantasma latente (B6) | media | no | sí (limpiar el bloque pre-S04 + regenerar snapshot) |
| 2 | 49% de las referencias del grafo sin `target_clause_id` (target = doc entero) — condiciona la expansión a 1 salto (A3.4) | media | condiciona diseño | sí (política de expansión cuando el target es doc) |
| 3 | 569 tablas omitidas silenciosamente en los 14 docs; 389 solo en E-ST-70-41C (A3.5) | media | condiciona chunking | sí (ya estaba pendiente; ahora con número) |
| 4 | Cache no aparcable limpiamente: sin `REDIS_URL` definida degrada con 2 warnings/request y posible timeout por operación (B8) | media | no, pero ensucia M2 | sí (estado "cache off" de primera clase) |
| 5 | `SessionNotFoundError` en el append post-stream escapa del generador SSE sin `event: error` (B2) | media | no | sí (decidir manejo; ventana crece con TTL de sesión S06) |
| 6 | Suite de contrato falla sin `OPENAI_API_KEY` presente (fixture `mock_deployments`) (B5) | media | no | sí (skipif vs inyección de deployment) |
| 7 | `llm_service` importa litellm (`cost_per_token`) — la doc del seam sobrevende el aislamiento (A2.4) | baja | no | doc: corregir; código: opcional |
| 8 | `LLM_PROVIDER` es config muerta en código, `.env.example` y CLAUDE.md (A1.2/B4) | baja | no | sí (eliminar vs cablear) |
| 9 | Placeholders M2 del árbol de CLAUDE.md no existen (A3.2) | baja | no | no (corregir doc) |
| 10 | ARCHITECTURE.md §4 afirma que el backend no expone prompt/contexto (falso desde S04 b2) (B7) | baja | no | no (corregir doc) |
| 11 | `references.json` es lista, no dict keyed por `(source,target)` como dice la doc (A3.4) | baja | no | no (ajustar doc o el builder M2) |
| 12 | `StreamErrorEvent` sin uso; evento `error` sin validar en el router (B7) | baja | no | opcional |
| 13 | Frontend: recuperación de sesión visible pero sin aviso de pérdida de contexto; detección por substring del mensaje (A1.1) | baja | no | opcional (UX) |
| 14 | Duplicación answer/stream + `_citations` acoplada al global — seams baratos pre-M2 (B9) | propuesta | facilita M2 | sí (aprobar refactor pre-M2) |
| 15 | `pyproject.toml` describe el proyecto como "software estimation" (B4) | baja | no | no (corregir) |

Áreas verificadas y limpias (una línea cada una): corrección async (B1); aislamiento de
tests (A1.4); invariante guards→sesión→cache con cemento de tests (A1.3/A2.2); fidelidad
prompt/contexto por envoltura (A2.3); contrato de citación y su batería de tests (A2.1/A2.5);
canales de error SSE testeados de extremo a extremo (B5c); logs sin contenido (B7); frontend
sin imports del backend (B7); stats del grafo coinciden con la doc (A3.4).

---

## Comandos de verificación ejecutados

| Comando | Resultado resumido |
|---|---|
| `uv run pytest` | **271 passed, 8 deselected** (real_llm), 3 warnings (RuntimeWarning de litellm logging_worker), 8.8 s |
| Inspección read-only `references.json` + `graph.graphml` (script inline) | 273 refs / 326 nodos / 247 aristas — coinciden con la doc; 135 refs sin `target_clause_id`, 134 sin `source_uid` |
| Conteo de tablas (python-docx, read-only, 14 DOCX) | 569 tablas totales; máx. ECSS-E-ST-70-41C = 389 |
| `ls data/processed/` | 14 JSON de documento + references.json + graph.graphml |
| Greps: `LLM_PROVIDER`, `cache.get/set`, `except:`/`print(`/`time.sleep`, imports de proveedor, `TODO`, `StreamErrorEvent`, `build_system_prompt` | Evidencia citada en A1.2, A2.2, B1, B2, B7 |
| `python -m corpus.graph` / `verify_parser` | **NO ejecutados**: el `__main__` de `graph.py` reescribe `references.json`/`graph.graphml` (graph.py:297-300) — no son de solo lectura |

## `git status` al cierre

El único archivo nuevo de esta sesión es este informe (`AUDIT-PRE-M2.md`, sin trackear).
Las 30 modificaciones listadas ya estaban en el árbol **antes** de iniciar la sesión
(idénticas al snapshot de apertura); esta auditoría no ha tocado ninguna de ellas.

```
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
	modified:   .claude/CLAUDE.md
	modified:   app/config.py
	modified:   app/context/reference.py
	modified:   app/main.py
	modified:   app/observability.py
	modified:   app/prompts/loader.py
	modified:   app/routers/queries.py
	modified:   app/routers/sessions.py
	modified:   app/schemas/query.py
	modified:   app/schemas/session.py
	modified:   app/services/cache.py
	modified:   app/services/input_guards.py
	modified:   app/services/llm_service.py
	modified:   app/services/llm_wrapper.py
	modified:   app/services/sessions.py
	modified:   frontend/config.py
	modified:   frontend/sse_client.py
	modified:   frontend/streamlit_app.py
	modified:   tests/app/conftest.py
	modified:   tests/app/test_health.py
	modified:   tests/app/test_input_guards.py
	modified:   tests/app/test_llm_service.py
	modified:   tests/app/test_llm_wrapper.py
	modified:   tests/app/test_multi_turn.py
	modified:   tests/app/test_prompt_endpoint.py
	modified:   tests/app/test_queries.py
	modified:   tests/app/test_query_integration.py
	modified:   tests/app/test_sessions.py
	modified:   tests/app/test_system_prompt_provider_agnostic.py
	modified:   tests/app/test_system_prompt_snapshot.py

Untracked files:
	AUDIT-PRE-M2.md
```
