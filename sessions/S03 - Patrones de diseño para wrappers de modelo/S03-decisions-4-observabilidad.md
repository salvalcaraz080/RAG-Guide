# S03-decisions-4-observabilidad.md — Implementación (Claude Code), brief 4/5

Decisiones y fricciones surgidas al implementar `S03-brief-4-observabilidad.md` sobre el
repo `RAG-ECSS`. Etiquetadas agnóstico (candidata a Guide) vs. dominio (solo ECSS).

## Paso 0 — Auditoría obligatoria del estado de structlog (antes de tocar nada)

1. **¿Existe `configure_logging()`/`structlog.configure(...)`?** No. `git grep` sobre
   `app/` no encontró ninguna llamada a `structlog.configure` en todo el repo, en ningún
   módulo (`main.py`, `config.py`, ni ningún otro).
2. **¿Qué loguea cada módulo hoy?**
   - `app/services/cache.py` y `app/services/llm_wrapper.py`: **structlog** —
     `logger = structlog.get_logger(__name__)` a nivel de módulo, con `.warning(...)` en
     varios puntos (`cache_disabled_redis_unavailable`, `cache_get_failed`,
     `cache_set_failed`, `cache_ping_failed` en `cache.py`; `openai_fallback_disabled`,
     `provider_resolution_failed` en `llm_wrapper.py`).
   - `app/services/llm_service.py`, `app/routers/queries.py`, `app/main.py`: **nada**. En
     particular, el `except Exception` de `stream_answer_query` (brief 3) capturaba el
     error del proveedor y emitía el evento SSE `error`, pero **no lo logueaba** en el
     servidor — un gap real que cierra este brief.
   - Sin `logging.getLogger` estándar, sin `print(` sueltos en `app/`.
3. **¿Con qué formato salían los warnings de los briefs 1–3?** Con los **defaults de
   structlog sin configurar**: confirmado empíricamente que ya sale con timestamp y nivel
   razonablemente legibles (`2026-07-05 13:16:56 [warning] evento key=val`), pero **sin
   ningún filtrado por nivel** (`logger.debug(...)` se imprime igual que `.warning(...)`,
   ignorando `LOG_LEVEL` de `Settings`) y sin posibilidad de cambiar a JSON en producción
   ni de vincular `request_id`.

**Conclusión de la auditoría → escenario "configura desde cero"** (de los tres que
planteaba el brief): structlog está instalado y ya en uso puntual, pero sin ninguna
configuración explícita. Se implementa `app/observability.py::configure_logging()` y se
llama una vez, en `main.py`, antes de que se importe nada que pueda loguear a nivel de
módulo — sin tocar `cache.py`/`llm_wrapper.py`, que recogen la config automáticamente al
usar `get_logger()` (structlog es perezoso: el proxy que devuelve `get_logger()` no fija
la configuración hasta la primera llamada real de log).

## Verificaciones hechas antes de implementar

1. **[agnóstico] `litellm.cost_per_token(model=, prompt_tokens=, completion_tokens=)`**
   es la función correcta para coste por tokens ya contados (no `completion_cost`, que
   espera un objeto `ModelResponse` completo o recalcula tokens desde texto). Devuelve
   `(prompt_cost, completion_cost)` en USD.
2. **[dominio] Verificado con los dos modelos cableados** (obligatorio según el brief):
   - `claude-haiku-4-5-20251001` → coste no nulo y plausible (p. ej. ~$0.00095 para
     200/150 tokens).
   - `gpt-5.4-mini-2026-03-17` (y `gpt-5.4-mini` sin fecha) → coste no nulo y plausible
     (~$0.000825 para los mismos tokens).
   Ambos modelos SÍ tienen precio en el price map embebido de LiteLLM — no hace falta
   derivar coste offline por ahora.
3. **[agnóstico] `litellm.cost_per_token` LANZA excepción para un modelo no mapeado**
   (verificado con un nombre inventado), no devuelve `0` silenciosamente. Esto simplifica
   la regla "nunca loguear 0 engañoso": basta con `try/except` alrededor de la llamada y
   tratar tanto la excepción como un resultado `== 0` como "desconocido" (`None`).
4. **[agnóstico] Las excepciones tipadas de LiteLLM exponen `llm_provider`/`num_retries`**
   (verificado en `AuthenticationError`, `RateLimitError`, `Timeout`,
   `APIConnectionError`, `InternalServerError`, y con una llamada real fallida contra
   Anthropic con key inválida: `llm_provider='anthropic'` se pobló correctamente;
   `num_retries` quedó `None` porque `AuthenticationErrorRetries=0` en nuestra
   `retry_policy` — no hubo reintento que contar).
5. **[agnóstico] `structlog.testing.capture_logs()`** es la utilidad oficial para
   capturar logs en tests, independiente de la config global activa. Desde
   `structlog==25.5.0` (instalada) acepta un parámetro `processors=` — necesario para
   incluir `structlog.contextvars.merge_contextvars` y así poder aserciones sobre
   `request_id` en tests (por defecto `capture_logs()` no fusiona contextvars).

## Registradas por el brief

- **[agnóstico] `app/observability.py`** (nuevo): `configure_logging(settings)` — dev:
  `ConsoleRenderer`; cualquier otro `APP_ENV`: `JSONRenderer`. `wrapper_class` con
  `structlog.make_filtering_bound_logger(getattr(logging, settings.LOG_LEVEL.upper()))`
  para que `LOG_LEVEL` filtre de verdad. `RequestIDMiddleware` (Starlette
  `BaseHTTPMiddleware`) vincula un `request_id` por request vía
  `structlog.contextvars.bind_contextvars` — todos los logs de esa request lo llevan sin
  pasarlo a mano por cada función.
- **[dominio] `configure_logging()` se llama en `main.py` ANTES de
  `from app.routers import queries`**, no dentro de `lifespan`: `lifespan` solo corre
  cuando Uvicorn ya arrancó, después de que todos los imports ya se ejecutaron — si
  `openai_fallback_disabled` dispara a nivel de módulo (falta `OPENAI_API_KEY`), tiene que
  hacerlo ya con la config final, no con los defaults de structlog sin configurar.
- **[agnóstico] Traza operacional en `llm_service.py`**, en el seam (no en el wrapper,
  que no sabe de cache ni de request): `_log_llm_call_started` /
  `_log_llm_call_completed` / `_log_llm_call_failed`, compartidas por `answer_query`
  (no-stream) y `stream_answer_query` (stream) — no duplicadas entre los dos paths,
  mismo principio que `_cache_lookup`/`_assemble_response` de los briefs 2-3.
- **[dominio] Evento `cache_hit` movido dentro de `_cache_lookup`** (no en cada llamador):
  como esa función ya es el único sitio que decide qué es un hit para ambos endpoints,
  loguear ahí también evita duplicar la construcción del evento. Lleva `cost_avoided_usd`
  (coste que *habría* tenido la llamada, calculado desde el `usage` original guardado —
  nunca coste cero) y `fallback_used` derivado retroactivamente del `model` cacheado.
- **[agnóstico] `fallback_used`** se deriva comparando el modelo que respondió
  (`model_used`, físico sin prefijo) contra `settings.LLM_MODEL` (el primario
  configurado, mismo formato) — funciona igual en la llamada en vivo y retroactivamente
  desde una entrada de caché, porque ambos guardan el mismo formato de string.
- **[dominio] Superficie de exposición del log:** no se loguea el texto completo de
  prompts/respuestas por defecto — solo metadatos (tokens, modelo, latencia, coste). Para
  ECSS (corpus público) no hay problema de datos sensibles hoy, pero queda anotado como
  nodo **agnóstico**: en un corpus con datos sensibles, el log es superficie de exposición
  (GDPR, S06). Si se quisiera loguear contenido para debug, tendría que ser explícito a
  nivel `DEBUG`, no añadido aquí.
- **[agnóstico] Extra no pedido explícitamente pero coherente con el forward-reference
  del brief 3** ("latencia hasta primer token... no implementar aquí, garantizar que los
  datos existen al cerrar el stream"): se añadió `ttft_ms` (time-to-first-token) en
  `llm_call_completed` del path de stream, vía el parámetro `**extra` de
  `_log_llm_call_completed` (no aplica al path no-stream, que no lo recibe).

## Tests automáticos

`uv run pytest` — 205 pasan, 2 deselected (`real_llm`, sin cambios). Nuevo:
`tests/app/test_observability.py` (10 tests), usando `structlog.testing.capture_logs()`
— nunca toca Redis ni LLM reales (reutiliza `fake_cache`/mocks ya existentes).

Cobertura de los 7 tests obligatorios:
1. Traza completa (miss) — `test_miss_emits_started_and_completed_trace`.
2. Traza de hit (`cache_hit` + `cost_avoided_usd`, sin `llm_call_completed`) —
   `test_hit_emits_cache_hit_event_not_llm_call`.
3. `fallback_used` correcto (con y sin rotación) —
   `test_fallback_used_true_when_backup_answered` /
   `test_fallback_used_false_when_primary_answered`.
4. Coste nulo explícito para modelo no mapeado —
   `test_cost_usd_is_explicit_none_for_unmapped_model` (+
   `test_cost_usd_is_a_plausible_positive_number_for_wired_models`, la verificación del
   punto 2 de arriba fijada como test de regresión).
5. `request_id` propagado — `test_request_id_propagated_across_logs` (vía `TestClient`
   real, atravesando `RequestIDMiddleware`).
6. Error logueado — `test_error_logs_llm_call_failed` (+
   `test_stream_error_logs_llm_call_failed` para el path de stream, no listado por
   número en el brief pero cubierto porque el logging se implementó en ambos paths).
7. Regresión — `test_non_stream_query_unchanged_after_observability`; el resto de la
   suite (`test_queries.py`, `test_streaming.py`) sigue en verde sin cambios.

Ruff limpio sobre `app/` y `tests/app/`.

## Verificación en Docker (test de usuario del brief)

`docker compose up --build -d` → `/health` → `{"status": "healthy", "cache": "up"}`.

Una consulta miss por `POST /api/v1/query`:
```
llm_call_started   cache_hit=False model_requested=claude-haiku-4-5-20251001 request_id=955eff0f-...
llm_call_completed cache_hit=False cost_usd=0.000684 fallback_used=False finish_reason=stop
                    latency_ms=1371.1 model_used=claude-haiku-4-5-20251001 provider=anthropic
                    request_id=955eff0f-... tokens_in=229 tokens_out=91 truncated=False
```
Mismo `request_id` en ambos eventos (confirma la propagación). La misma pregunta otra vez:
```
cache_hit  cost_avoided_usd=0.000684 fallback_used=False model=claude-haiku-4-5-20251001
           provider=anthropic request_id=22ca06c1-... usage={'input_tokens': 229, 'output_tokens': 91}
```
`request_id` distinto (nueva request), sin `llm_call_started/completed`. Por
`POST /api/v1/query/stream` con una pregunta nueva: misma traza
`llm_call_started`/`llm_call_completed`, con `ttft_ms=563.6` (vs. `latency_ms=1208.9`
total) — confirma que el tiempo al primer token se captura por separado del total.
Consola coloreada y legible (modo dev, `APP_ENV=development`).

## Nota de cierre de hito

Con este brief cerrado, el hito **backend** de S03 queda completo: wrapper + fallback
(brief 1) + cache exact-match (brief 2) + streaming SSE (brief 3) + observabilidad
(brief 4). La consolidación del hito (`S03-consolidado.md`) y la primera pasada de
`RAG-GUIDE.md` quedan fuera de este documento — trabajo de Claude Chat, no de Claude Code.
El brief 5 (Streamlit) abre el segundo hito.
