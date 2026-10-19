# pre-S11 · Decisiones — retirar el streaming

> Decisiones tomadas al implementar `pre-S11-brief.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
>
> **Verificación:** `uv run pytest` **1169 verde** (1179 − 21 borrados + 11 nuevos), `ruff` igual que
> antes (el E402 conocido de `app/main.py`). La API en marcha (bind mount + `--reload`) sirve ya el
> contrato nuevo: `/query/stream` responde 404 y el OpenAPI de `QueryResponse` lleva `finish_reason`,
> `truncated` y `latency_ms`. Latencia de referencia medida con 14 llamadas reales autorizadas.

---

## 1. Decisiones de implementación

- **[AGNÓSTICO] P1–P3 aplicadas tal cual.** Fuera `POST /api/v1/query/stream`,
  `llm_service.stream_answer_query`, `llm_wrapper.stream_llm`, `StreamChunk`, `StreamDoneEvent`,
  `StreamErrorEvent`, `ttft_ms` y la validación SSE en el borde. Sin flag ni código latente: el
  historial de git lo conserva.
- **[AGNÓSTICO] P4: dos invariantes vivían solo en el stream, y `/query` los gana.**
  - **Truncado.** `QueryResponse` no llevaba `truncated` ni `finish_reason`: el aviso de respuesta
    cortada solo existía en el `done`. Ahora viajan en `/query` (`truncated = finish_reason ==
    "length"`). Es mover un campo del contrato que se borra al que queda, no diseño nuevo.
  - **Fallo del proveedor.** En `/query` la excepción escapaba como un 500 sin tipar. Ahora
    `answer_query` la registra (`llm_call_failed`) y la relanza como **`LLMCallFailed`**, que el
    router traduce a **502** con mensaje genérico (el detalle del proveedor va al log, no al
    cliente; mismo criterio que el 503 de retrieval). 502 y no 500, como la respuesta vacía: la
    petición estaba bien formada y quien falló fue el proveedor. No deja turno en la sesión.
- **[AGNÓSTICO] P6: `latency_ms` = tiempo de servidor de la petición entera** (guardrails,
  retrieval y modelo), en la respuesta y en un evento nuevo `query_completed`. El `latency_ms` que
  ya había en `llm_call_completed` se queda como estaba —solo la llamada al modelo— y su docstring
  lo dice. Son dos medidas distintas con su sitio.
- **[AGNÓSTICO] P5: `frontend/backend_client.py` sustituye a `sse_client.py`** (`create_session`,
  `query`, `get_active_instructions`, sin Streamlit). La UI hace una llamada con `st.spinner` y una
  frase fija; el sidebar muestra la latencia del servidor y la del cliente en lugar del TTFT.
  Lleva tests propios (`tests/frontend/test_backend_client.py`, 3).
- **[AGNÓSTICO] P7: `QUERY_TIMEOUT_SECONDS = 120`**, constante con su porqué junto a ella y un
  test que impide que vuelva a un default corto. Frente a la latencia medida (p95 7,4 s, máximo
  7,8 s) es unas 16 veces el p95: margen para los reintentos (2 por timeout) y el fallback del
  Router. Sin variable de entorno nueva: nadie la ha pedido.
- **[AGNÓSTICO] Dependencias:** `sseclient-py` fuera del grupo `frontend`; `uv lock` lo retira y
  no cambia nada más. El suelo `fastapi[standard]>=0.135` existía por `fastapi.sse`: **se
  mantiene** y pierde su comentario. Bajarlo no simplifica nada (el lock no se mueve) y sería
  cambiar una dependencia sin motivo.
- **[AGNÓSTICO] `scripts/ttft_baseline.py` se borra** (mide un endpoint que ya no existe) y
  `scripts/latency_baseline.py` lo sustituye con el mismo protocolo. `output_ttft_baseline.txt` se
  queda como registro de la medición de S10.

## 2. Auditoría de cobertura antes de borrar (§3.1)

(a) = comportamiento solo de stream → se borra. (b) = invariante que **solo** probaba el stream →
se migra a `/query` antes de borrar. En el resto, el invariante ya estaba probado en `/query`.

| Test | Clase | Destino |
|---|---|---|
| `test_streaming::test_streams_chunks_then_done` | (a) | borrado |
| `test_streaming::test_every_request_reaches_the_wrapper` | cubierto | `test_query_integration` (dos peticiones → dos llamadas al wrapper) y `test_observability` (dos trazas) |
| `test_streaming::test_stream_citations_match_non_stream_path` | (a) paridad entre caminos | borrado |
| `test_streaming::test_truncation_is_marked` | **(b)** | **migrado:** `test_queries::TestTruncation` (2) + campo en `QueryResponse` |
| `test_streaming::test_stream_endpoint_and_generator_are_async` | **(b)** en su mitad `/query` | **migrado:** `test_queries::test_the_endpoint_and_the_service_are_async` |
| `test_streaming::test_non_stream_query_unchanged` | duplicado | `test_queries::test_query_returns_only_the_cited_clause` |
| `test_streaming::test_stream_done_carries_the_same_evidence_as_query` | (a) paridad | evidencia en `/query`: `test_query_carries_structured_evidence_unconditionally` |
| `test_streaming::test_stream_refusal_has_no_citations` | cubierto | `TestRefusalField::test_out_of_scope_is_typed` |
| `test_streaming::test_multiturn_stream_appends_generated_turn` | cubierto | `test_multi_turn::test_multiturn_appends_generated_turn` |
| `test_streaming::test_provider_error_mid_stream_emits_typed_error_event` | **(b)** | **migrado:** `test_queries::TestProviderFailure` (2): 502 con detalle genérico, sin turno en sesión |
| `test_streaming::test_empty_stream_closes_as_error…` | cubierto | `test_query_502s_instead_of_answering_with_no_words`, `test_empty_completion_leaves_no_turn_in_the_session` |
| `test_streaming::test_stream_refusal_still_carries_the_retrieved_evidence` | cubierto | `test_query_grounding_not_covered_does_not_fabricate_citation` |
| `test_streaming::test_a_marker_split_across_deltas_is_still_cited` | (a) | borrado |
| `test_streaming::test_retrieval_unavailable_emits_error_before_any_chunk` | cubierto | `TestRetrievalFailures::test_missing_embedding_is_a_503` |
| `test_streaming::test_multiturn_stream_stores_no_citation_marks` | cubierto | `test_history_stores_no_citation_marks_and_the_response_keeps_markers` |
| `test_streaming::test_done_carries_the_refusal_field` | cubierto | `TestRefusalField::test_a_refusal_that_cites_keeps_its_citations` |
| `test_streaming::test_done_of_a_grounded_answer_has_no_refusal` | cubierto | `TestRefusalField::test_a_grounded_answer_has_no_refusal` |
| `test_input_guards::test_flagged_input_stream_emits_error_with_no_chunks` | cubierto | `test_flagged_input_returns_400_and_never_calls_the_model` |
| `test_input_guards::test_flagged_input_never_reaches_the_context_provider` | mixto | se queda su mitad `/query` |
| `test_multi_turn::test_provided_but_absent_session_stream_emits_error_event` | cubierto | `test_provided_but_absent_session_returns_404` |
| `test_multi_turn::test_moderation_before_session_stream_emits_toxicity_error` | cubierto | `test_moderation_runs_before_session_resolution` (el orden se prueba en el servicio) |
| `test_multi_turn::test_unknown_session_never_reaches_the_context_provider` | mixto | se queda su mitad `/query` |
| `test_observability::test_stream_error_logs_llm_call_failed` | cubierto | `test_error_logs_llm_call_failed`, que ahora espera `LLMCallFailed` con la excepción del proveedor como causa |

**Ningún invariante pierde su test.**

## 3. Inventario: decisiones que condicionó el streaming

| Decisión | Qué se hizo |
|---|---|
| Prosa, no structured output | **No se toca.** El porqué en `ARCHITECTURE.md` §3 se reescribe: la restricción (el streaming token a token) ya no existe y la decisión queda pendiente del contrato tipado (`S11-brief-2`). En `CLAUDE.md`, «Contrato de salida tipado» entra en pendientes. |
| Frase marcador natural de rechazo | **No se toca.** Desaparece el motivo «bufferizar el arranque del stream»; queda como puente hasta el contrato tipado. |
| Marcadores leídos en `_finish` sobre texto completo | Se mantiene; fuera la justificación de los deltas partidos (y su test, que era solo de stream). |
| Respuesta vacía → `event: error` / 502 | Solo 502. |
| `stream_options={"include_usage": True}` y truncado en stream | Fuera con `stream_llm`. El truncado vía `finish_reason` sigue en no-stream y ahora llega al cliente. |
| `ttft_ms` en observabilidad y sidebar | Fuera; sustituido por `latency_ms` total (P6). |
| Limitación de `EventSourceResponse` | Desaparece con el endpoint (sección «Streaming SSE» de ARCHITECTURE retirada). |
| Router de Streamlit: consumo SSE | Una llamada `requests.post` a `/query` (P5). |

**Encontradas además, y quitadas porque simplifican:**

- **[AGNÓSTICO] Sesión perdida detectada por substring.** El frontend descartaba el `session_id`
  si el mensaje del `event: error` contenía `"session not found"`, deuda anotada porque el evento
  no tenía código tipado. Con `/query` la señal tipada ya existe: el **404**. Se discrimina por
  código de estado y la deuda desaparece.
- **[AGNÓSTICO] `**extra` en `_finish` y `_log_llm_call_completed`** existía para que el stream
  añadiera `ttft_ms`. Fuera.
- **[AGNÓSTICO] Pendiente de ARCHITECTURE §7 «`finish_reason` puede venir sin decir nada».** Lo
  causaba el default `"unknown"` del wrapper de stream, cuando el proveedor cerraba el stream sin
  `finish_reason`. Con la respuesta completa, `finish_reason` llega en la propia respuesta; el
  pendiente se retira.
- **[AGNÓSTICO] Docstring obsoleto de `EmptyCompletionError`** («el wrapper defaultea un stream
  silencioso a `stop`»): reescrito para el camino que queda.
- **[AGNÓSTICO] Menciones a `/query/stream`** en `schemas/session.py`, `services/sessions.py` y
  `.env.example`, retiradas.

**Nada exigió diseño nuevo**, así que no queda ningún `# TODO: pending decision in Claude Chat` de
este brief. (El marcador del brief está en español; el código ya usa la forma inglesa.)

## 4. Recuento de tests

| | Tests |
|---|---|
| Partida | 1179 |
| Borrados (solo stream o ya cubiertos en `/query`) | −21 (17 de `test_streaming.py`, 1 de `test_input_guards`, 2 de `test_multi_turn`, 1 de `test_observability`) |
| Nuevos en `/query` | +8: truncado (2), fallo de proveedor (2), `latency_ms`, async, ruta de stream inexistente (404/405), ningún import de `sseclient`/`fastapi.sse` |
| Nuevos en el frontend | +3 (`test_backend_client.py`) |
| **Final** | **1169** |

De los nuevos, 4 son **migraciones** de invariantes que solo probaba el stream (truncado ×2,
fallo de proveedor ×2) y 1 la mitad `/query` del test de async; los demás los pide el §4 del
brief. Dos tests se quedaron con su mitad `/query` y uno cambió su expectativa (`LLMCallFailed`).

## 5. Latencia de referencia

`scripts/latency_baseline.py` → `scripts/output_latency_baseline.txt`. Mismo protocolo que el TTFT
de S10: las 14 consultas del golden set v2, defaults vigentes, último turno en multi-turn, en serie
contra la API en marcha. 14/14 sin error.

| Medida | Mediana | p95 | n |
|---|---|---|---|
| **[DOMINIO]** Latencia total, cliente | **6,31 s** | 7,40 s | 14 |
| **[DOMINIO]** `latency_ms` de servidor | 6,30 s | 7,39 s | 14 |

- **[DOMINIO]** Cliente y servidor difieren en ~10 ms: la red local no cuenta; lo que pesa es la
  generación. El rango es de 2,8 s (`q14`, multi-turn, 130 tokens de salida) a 7,8 s (`q01`).
- **[AGNÓSTICO]** Sustituye al TTFT de S10 (mediana 1,76 s) como referencia: sin stream, el
  usuario espera la respuesta entera. La comparación del reranker en `CLAUDE.md` y ARCHITECTURE §7
  pasa a hacerse contra esta cifra (1,6–7 s es de +25 % a algo más del doble).

## 6. Para el usuario

- **UAT:** `pre-S11-uat.md`.
- **`.env` local:** un comentario sigue mencionando `/query/stream`. No lo he tocado: es tu fichero,
  con claves; `.env.example` ya está corregido.
