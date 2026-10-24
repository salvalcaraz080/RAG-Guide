# S13 · Decisiones — Brief 1: desglose de latencia por etapa

> Decisiones tomadas al implementar `S13-brief.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Rama `s13-orquestacion-agentes`. Sin llamadas reales. Es un instrumento: no cambia el retrieval,
> el prompt, el modelo, el esquema de salida, `QueryResponse` ni ningún default.

## 1. Mecanismo (L1) — [AGNÓSTICO]

- **Un solo mecanismo**, en `app/observability.py`: `stage(nombre)` es un context manager sobre
  `perf_counter` que suma la duración al `StageTimer` de la petición en curso. El timer vive en un
  `ContextVar` que abre `answer_query` con `recording(timer)`. `recording` fija además `total_ms`
  al cerrar, también si hay excepción.
- **Por qué un `ContextVar` y no pasar el timer de mano en mano:** las cuatro etapas de retrieval
  ocurren dentro del proveedor de contexto, que es una costura inyectada (`ContextProvider`). Con el
  `ContextVar`, el proveedor marca sus etapas sin que cambie el protocolo ni los fakes. Fuera de una
  petición, `stage` mide y no registra nada. Así `/search` y el arnés de retrieval, que usan el
  mismo proveedor, no cambian.
- **Las etapas no deciden nada** (separación trabajo/control): solo envuelven código que ya
  existía. No se ha partido ni fusionado ninguna función para que encaje. Las etapas no se anidan;
  una etapa que se ejecuta dos veces (`prompt`, `validation`) suma.
- **Medido una vez:** `embedding_ms` y `db_ms` de `retrieval_completed` son ahora las marcas de las
  etapas `embedding` y `db`. Ya no hay un segundo `perf_counter`.

## 2. Lista final de etapas — [AGNÓSTICO] salvo donde se indica

| Etapa | Qué cubre | Dónde |
|---|---|---|
| `guardrails` | `run_input_guards` (scope, toxicidad) | `llm_service._prepare` |
| `history` | Resolver la sesión y componer la consulta de retrieval | `llm_service._prepare` |
| `embedding` | `embed_query` | proveedor |
| `db` | Detección de routing (lectura única de documentos) y ranking (una consulta con `filter`, dos con `quota`) | proveedor |
| `expansion` | `_expand` (`clause_chunks` + `expand`); solo si no es `off` | proveedor |
| `xref` | `_references`: etiquetas de tabla (una vez por proceso), parser y candidatos, `get_clause` y `fill`; solo si no es `off` | proveedor |
| `prompt` | Agrupar por cláusula, presupuesto de entrada, bloque de contexto y mensaje de usuario | `llm_service` |
| `llm` | Cada llamada a `_call_model` (L4: un intento por llamada) | `llm_service._answer` |
| `validation` | Parsear el esquema en cada intento y `verify` | `llm_service` |
| `compose` | `_assemble_response` (incluye `compose`) y añadir el turno a la sesión | `llm_service._finish` |

- **Cambio de significado de `db_ms`:** antes medía todo el bloque de base de datos (ranking,
  expansión y referencia). Ahora mide **solo el ranking**, como dice el propio brief («`db`:
  búsqueda vectorial, ya existe como `db_ms`»). La expansión y la referencia tienen etapa propia.
  `db_ms` solo se usaba en el log.
- **Una etapa apagada no aparece** en `stages_ms` (con `RETRIEVAL_EXPANSION=off` no hay
  `expansion`), así que el informe distingue «no se ejecutó» de «tardó 0».
- **Qué queda en `unaccounted_ms`:** los eventos de log (`retrieval_completed`,
  `citations_validated`, `llm_call_*`), el cierre de la sesión de base de datos (devolver la
  conexión al pool), el ajuste de coste y el propio marco de la petición. Debería ser del orden de
  milisegundos. Si sale grande, es un fallo del instrumento (criterio de segunda pasada del brief 2).
- **Un movimiento de orden:** `citations_validated` se emite ahora justo tras `verify` y antes de
  componer, para que `compose` sea un bloque contiguo. El contenido del evento es el mismo.

## 3. Evento (L2, L3)

- **`query_stages`**, un evento **dedicado** y no campos añadidos a `query_completed`. El motivo:
  `query_completed` solo sale en una petición servida, y el brief pide que una petición que falla
  registre las etapas a las que llegó. `query_stages` sale **siempre**, desde un `finally` de
  `answer_query`, con:
  - `outcome`: `served`, o el nombre de la excepción que terminó la petición (`ContextUnavailable`,
    `LLMCallFailed`, `InvalidStructuredOutput`, `InputGuardrailError`…);
  - `total_ms`, `stages_ms`, `unaccounted_ms` (total − suma de etapas, L3) y `llm_attempts`.

  `query_completed` no cambia y sigue saliendo después. En una petición servida, `total_ms` es
  exactamente el `latency_ms` de la respuesta: los dos salen del mismo timer.
- **`request_id`:** llega por `structlog.contextvars`, como en el resto de eventos. Un test lo
  comprueba a través del router.
- **Fallo en retrieval:** el proveedor sigue emitiendo `retrieval_failed` sin cambios, y después
  sale `query_stages` con las etapas que llegaron a ejecutarse (fallo de embedding: `guardrails`,
  `history`, `embedding`; fallo de base: lo anterior más `db`).
- **Fuera del contrato:** `QueryResponse` no cambia. Un test fija el sha256 de su esquema JSON,
  tomado antes de tocar nada, y otro comprueba que la respuesta servida no trae campos de tiempos.
- **El arnés** recibe el desglose sin pasar por la respuesta: `answer_query(..., timer=)` acepta un
  `StageTimer` del llamador. Sin él, se crea uno nuevo. El router no lo pasa.

## 4. Reintento de esquema y fallback (L4) — [AGNÓSTICO]

- **Un intento por llamada a `_call_model`**, con `attempt`, `ms`, `outcome`, `model_used` y
  `fallback_used`. Valores de `outcome`:
  - `valid`;
  - `invalid`: reintento de esquema;
  - `provider_failed`: la llamada lanzó, sin modelo;
  - `unparsed`: el parseo lanzó, por salida vacía o cortada; el tipo concreto está en el `outcome`
    del evento.

  La etapa `llm` es la suma de los intentos, así que un reintento de esquema se ve como dos
  intentos y no como un modelo lento.
- **El fallback y los reintentos de LiteLLM no se parten:** ocurren dentro de una sola
  `router.acompletion`. El intento los refleja como `fallback_used=True` y en su duración, pero no
  como intentos aparte. Partirlos exigiría instrumentar el router por dentro (callbacks de
  LiteLLM), y la capa de reintento es una sola (`S13-resumen.md` §5). Si el brief 2 muestra
  intentos de fallback lentos, se puede abrir.
  `TODO: pending decision in Claude Chat` solo si hiciera falta ese detalle.

## 5. Arnés — [AGNÓSTICO]

- Cada ejecución juzgada guarda `stages` (el `StageTimer.as_dict()`) en el artefacto, **también las
  ejecuciones con error**. Los turnos previos de las multi-turn no se cronometran: no son el turno
  juzgado.
- **Informe** (una sección del informe del arnés y un CLI aparte):
  `corpus/groundtruth/generation/latency.py`. Por configuración y sobre las ejecuciones
  comparables (la respuesta del fallback cronometra otro modelo), da por etapa:
  - mediana y p95 (inclusivo, el mismo cálculo que la latencia del informe), sobre las ejecuciones
    que ejecutaron la etapa;
  - **fracción del total** = tiempo sumado de la etapa / total sumado;
  - las filas `unaccounted` y `total`;
  - las ejecuciones con más de un intento y los intentos del fallback.
- **Re-puntuable:** `uv run python -m corpus.groundtruth.generation.latency <artefacto.json> …`,
  sin modelo ni base. Los artefactos anteriores salen «Without breakdown». Comprobado sobre
  `generation_20261019-224307.json`: las 4 configuraciones salen sin desglose y nada falla.

## 6. Cero cambio de comportamiento (L5)

- **Array de mensajes:** los digests fijados de `test_history_equivalence.py` (cinco turnos, que
  incluyen el primero de turno único), grabados antes de la instrumentación, siguen en verde.
  Además, el array es idéntico con y sin timer del llamador.
- **Contexto:** el proveedor real, con sus dobles de embedding y base, devuelve los mismos chunks,
  modo, `top_k` y `xref` dentro y fuera de una grabación.
- `QueryResponse`: el esquema está fijado por hash (§3).

## 7. Tests

**1.419 → 1.449** (+30). Modificados 1 y borrados 0.

- **+20 en `tests/app/test_stage_timing.py`:**
  - el timer: suma por etapa, total y remanente, no-op fuera de una grabación, etapa que lanza;
  - el evento: todas las etapas y su suma, etapa apagada ausente, `embedding_ms`/`db_ms` como
    marcas, un intento, reintento de esquema con dos intentos, fallback marcado, fallo de
    proveedor, fallos de embedding y de base con `retrieval_failed` intacto, timer del llamador,
    `request_id`;
  - L5 y el contrato.
- **+10 en `tests/corpus/test_generation_latency.py`:** desglose con, sin y mezclado, orden de
  etapas, ejecuciones no comparables, render, CLI, y el artefacto del arnés con `stages` también en
  las ejecuciones con error.
- **Modificado:** `FakeAnswer` en `test_generation_harness.py` acepta `timer=` y lo cierra como el
  servicio real.

## 8. Etiquetado

- **[AGNÓSTICO]:** todo el brief. El mecanismo de etapas por `ContextVar`, el evento por petición
  servida o no con remanente, el intento por llamada y el informe re-puntuable sirven para
  cualquier pipeline RAG sobre structlog.
- **[DOMINIO]:** solo la lista concreta de etapas (`expansion` y `xref` existen por la expansión de
  cláusula y por referencia de RAG-ECSS).
