# S03-decisions-3-streaming-sse.md — Implementación (Claude Code), brief 3/5

Decisiones y fricciones surgidas al implementar `S03-brief-3-streaming-sse.md` sobre el
repo `RAG-ECSS`. Etiquetadas agnóstico (candidata a Guide) vs. dominio (solo ECSS).

## Verificación hecha antes de implementar (§ "Tramo B — verificar en el repo")

El brief pedía comprobar en el repo/historial qué mecanismo SSE usó el "estimador" antes de
reinventarlo. **Este repo (`RAG-ECSS`) no tiene ningún rastro de streaming**, ni siquiera en
el tag `pre-rebuild` (verificado con `git grep`/`git log -S` sobre `StreamingResponse`,
`EventSourceResponse`, `sse-starlette` en todo el historial — cero resultados). El "estimador
del máster" al que se refiere el brief es un proyecto **hermano** (`C:\Users\salca\proyectos\
LIDR`), no una versión anterior de este repo. Localizado ahí:
`app/routers/estimations.py::create_estimation_stream`, usando
`from fastapi.sse import EventSourceResponse, ServerSentEvent`.

**[agnóstico] `fastapi.sse` es real y está en la versión instalada.** Confirmado
`fastapi==0.136.1` expone `fastapi.sse.EventSourceResponse`/`ServerSentEvent` como
submódulo (no como atributo top-level de `fastapi` — de ahí que una comprobación ingenua
con `hasattr(fastapi, "EventSourceResponse")` diera negativo al principio; hay que probar
el import directo `from fastapi.sse import ...`). No se necesitó añadir `sse-starlette` ni
ninguna dependencia nueva. `pyproject.toml` sube el mínimo de `fastapi` a `>=0.135`
(antes `>=0.110`) para reflejar esta dependencia real.

## Fricción de implementación no prevista por el brief (la más importante de este brief)

**[agnóstico] `fastapi.sse.EventSourceResponse` no se puede instanciar a mano dentro de un
endpoint normal.** El diseño original de este brief asumía —siguiendo el flujo literal del
brief— que se podía decidir *por petición* entre devolver un JSON plano (hit) o abrir SSE
(miss), construyendo `EventSourceResponse(generador)` manualmente solo en la rama miss. Al
probarlo, Starlette lanzó `AttributeError: 'ServerSentEvent' object has no attribute
'encode'`: la serialización de `ServerSentEvent` → texto SSE **no** vive en la clase
`EventSourceResponse` (que es solo un marcador + `media_type`), vive en la capa de routing
de FastAPI (`fastapi/routing.py::get_request_handler`), y esa capa **solo** activa el modo
SSE cuando la ruta se declaró con `response_class=EventSourceResponse` **y** la propia
función del endpoint es un generador async (`yield`). Es una decisión **por ruta, no por
petición** — no hay forma de alternar JSON/SSE dentro del mismo endpoint reutilizando este
mecanismo.

**Decisión (consultada con el usuario, no tomada en solitario):** en vez de abandonar
`fastapi.sse` y formatear SSE a mano con `StreamingResponse` (la alternativa que sí
permitiría alternar JSON/SSE por petición, pero reinventando el formateo que el brief pedía
explícitamente NO reinventar), se mantiene `fastapi.sse` tal cual funciona: **el HIT también
abre SSE**, pero `stream_answer_query` no produce ningún evento `chunk` antes del `done` —
la conexión abre y cierra casi al instante con un único mensaje. Verificado en Docker:
~0.19s para un hit (vs. varios segundos de un miss real). Sin diferencia de comportamiento
para el usuario (0 llamadas al LLM, respuesta instantánea en ambos casos); la única
diferencia es de encuadre en el cable (`event: done\ndata: {...}` en vez de un body JSON
plano). El endpoint quedó mucho más simple gracias a esto: no hace falta `anext()` para
adelantar el primer evento y decidir la rama — el propio generador de `stream_answer_query`
ya solo produce un evento en el hit, y el endpoint se limita a reemitir cada evento tal cual.

## Registradas por el brief

- **[agnóstico] Principio rector respetado:** `answer_query` y `stream_answer_query`
  comparten `_cache_lookup` (clave + `cache.get`) y `_assemble_response`/`_citations`
  (misma construcción de citas). El streaming no tiene lógica de negocio propia — solo
  cambia cómo se entrega el resultado. Verificado con test explícito
  (`test_stream_citations_match_non_stream_path`) que compara las citas de `/query` y de
  `/query/stream` para la misma pregunta.
- **[agnóstico] Las dos capas de streaming, separadas tal como pedía el brief:**
  - Tramo A (`llm_wrapper.stream_llm`): `router.acompletion(stream=True,
    stream_options={"include_usage": True})`, normaliza a `StreamChunk` (texto o resumen
    final). Sigue sin saber que existe FastAPI.
  - Tramo B (`routers/queries.py::query_stream`): reemite los eventos de
    `stream_answer_query` como `ServerSentEvent`. Sigue sin saber cómo se construye el
    prompt ni cómo se decide el cache.
- **[agnóstico] `stream_options={"include_usage": True}` es obligatorio.** Verificado
  empíricamente (mock y llamada real): sin este parámetro, LiteLLM no incluye ningún chunk
  con `usage` poblado durante el streaming — no habría token count que cachear ni
  reportar. Confirmado también en el patrón ya usado en LIDR
  (`app/services/llm_wrapper.py::stream_structured`), que documentaba el mismo requisito.
- **[agnóstico] Solo el texto se streamea; citas/métricas van en un único `event: done`**
  tipado (`StreamDoneEvent`), tal como especificaba el brief — sin eventos intermedios
  para las citas.
- **[dominio] `truncated` (finish_reason == "length") marcado explícitamente** en el
  `done` — fallo de integridad, no cosmético, en un sistema de trazabilidad obligatoria.
- **[agnóstico] Error a mitad de stream:** `stream_answer_query` captura cualquier
  excepción del wrapper explícitamente (nunca `except: pass`) y emite `{"type": "error",
  "error": str(exc)}`, cerrando el generador limpiamente. El endpoint lo reemite como
  `event: error`. Verificado con test (`test_provider_error_mid_stream_emits_typed_error_event`)
  que fuerza un `ConnectionError` a mitad de stream.
- **[agnóstico] `**provider_params` heredado del brief 1** se mantiene en `stream_llm`
  (misma firma que `call_llm`), sin usarse todavía — sigue sin cerrar esa puerta.

## Tests automáticos

`uv run pytest` — 195 pasan, 2 deselected (`real_llm`, sin cambios). Nuevo:
`tests/app/test_streaming.py` (8 tests), mockeando `llm_service.stream_llm` con un
generador controlable (`_fake_stream_llm`) — nunca toca el Router real ni Redis real
(reutiliza `fake_cache` de `tests/app/conftest.py`).

Cobertura de los 7 tests obligatorios + el requisito de error explícito:
1. Miss streamea texto + evento done — `test_miss_streams_chunks_then_done`.
2. Hit no streamea (verificado con un mock que falla si se invoca) —
   `test_hit_does_not_stream`.
3. Miss cachea la respuesta — `test_miss_then_repeat_is_a_hit`.
4. Citas coherentes con `/query` — `test_stream_citations_match_non_stream_path`.
5. Truncamiento marcado — `test_truncation_is_marked`.
6. Async (endpoint + generador) — `test_stream_endpoint_and_generator_are_async`.
7. Regresión de `/query` — `test_non_stream_query_unchanged`.
8. (No numerado en el brief, exigido en prosa) Error a mitad de stream —
   `test_provider_error_mid_stream_emits_typed_error_event`.

Ruff limpio sobre `app/` y `tests/app/`.

## Verificación en Docker (test de usuario del brief)

`docker compose up --build -d` → `/health` → `{"status": "healthy", "cache": "up"}`.

`curl -N -X POST /api/v1/query/stream` con una pregunta ECSS nueva: chunks de texto en
vivo, cierre con `event: done` (citas correctas, `finish_reason: "stop"`,
`truncated: false`, `cache_hit: false`). Misma pregunta otra vez: un único
`event: done` con `cache_hit: true`, ~0.19s (vs. varios segundos del miss). `POST
/api/v1/query` (no-stream) verificado sin cambios de contrato.

## No implementado (fuera de alcance de este brief) — nada que reportar

Logging de latencia hasta primer token / `cache_hit` en el path de stream (brief 4) y el
cliente SSE de Streamlit (brief 5) no se adelantaron. Los datos que esos briefs necesitan
(`finish_reason`, `truncated`, `cache_hit`, `usage`) ya existen en el evento `done`.
