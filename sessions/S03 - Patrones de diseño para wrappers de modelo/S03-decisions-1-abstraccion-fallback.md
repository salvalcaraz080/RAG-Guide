# S03-decisions.md — Implementación (Claude Code), brief 1/5

Decisiones y fricciones surgidas al implementar `S03-brief-1-abstraccion-fallback.md` sobre
el repo `RAG-ECSS`. Etiquetadas agnóstico (candidata a Guide) vs. dominio (solo ECSS).

## Verificaciones hechas antes de implementar (§ "Verificaciones que CC debe hacer")

1. **[agnóstico] Model string de LiteLLM para Anthropic con fecha.** Confirmado contra la
   instalación real (`litellm==1.90.3`, `litellm.get_model_info(...)`):
   `anthropic/claude-haiku-4-5-20251001` es un modelo reconocido en el cost/model map
   embebido de LiteLLM. El material original (`claude-haiku-4-5` sin prefijo ni fecha) no
   se copió.
2. **[dominio] Modelo OpenAI de respaldo.** `gpt-4o-mini` (del material) está obsoleto.
   Se evaluó `gpt-5.4-mini` — confirmado vigente por dos vías independientes: (a) búsqueda
   web (julio 2026): recomendado por OpenAI como variante rápida/económica de
   producción, GPT-5.2 en retiro; (b) `litellm.get_model_info("openai/gpt-5.4-mini")` lo
   reconoce, y una llamada real (ver verificación end-to-end abajo) respondió con
   `gpt-5.4-mini-2026-03-17`. Fijado en `.env`/`.env.example` como `LLM_FALLBACK_MODEL`,
   no hardcodeado.
3. **[agnóstico] `usage` poblado en ambos proveedores.** Confirmado con llamadas reales
   (Anthropic directo y OpenAI vía fallback forzado): `usage.prompt_tokens` /
   `completion_tokens` vienen poblados en ambos casos, sin necesidad de mapeo adicional en
   el wrapper.

## Registradas por el brief

- **[agnóstico]** `Router` completo desde el primer brief (no `acompletion` simple), con
  `model_list` (nombres lógicos `ecss-primary`/`ecss-fallback` → modelos físicos),
  `fallbacks=[{"ecss-primary": ["ecss-fallback"]}]` y `retry_policy` explícito
  (`AuthenticationErrorRetries=0`, `TimeoutErrorRetries=2`, `RateLimitErrorRetries=2`,
  `InternalServerErrorRetries=1`) — reintentos diferenciados por tipo de error, no un
  número ciego.
- **[agnóstico]** `router.acompletion()`, nunca `.completion()` — verificado en el código
  fuente instalado (`litellm/router.py`) que el camino async y el síncrono llevan lógicas de
  fallback distintas; no es solo una cuestión de no bloquear el event loop.
- **[agnóstico]** `LLMResult` (dataclass) con `text`, `model_used`, `provider`, `tokens_in`,
  `tokens_out`, `finish_reason` — todos poblados desde este brief aunque solo `text`/
  `tokens_*` se consuman en `llm_service.answer_query` por ahora. `model`/`provider` en
  `QueryResponse` pasan a reflejar quién **realmente** respondió (`model_used`/`provider`
  del wrapper), no el primario configurado — necesario para que un fallback sea visible al
  cliente HTTP, no solo al log interno.
- **[agnóstico]** `**provider_params` en `call_llm` se reenvía sin inspeccionar a
  `router.acompletion` — puerta abierta para brief 2 (cache, `cache_control` de Anthropic)
  sin tocar la firma del wrapper.
- **[dominio]** `OPENAI_API_KEY` se dejó **opcional** en `Settings` (`str | None = None`),
  siguiendo la recomendación del brief: sin ella el Router arranca solo con el primario y
  se loguea un warning (`openai_fallback_disabled`) en vez de fallar el arranque. En este
  repo la key ya está cableada en `.env`, así que el fallback está activo en desarrollo.

## Fricción de implementación no prevista por el brief

- **[agnóstico] `mock_testing_fallbacks=True` dispara una llamada real al fallback si no
  se le da `mock_response`.** El brief lo menciona como mecanismo de test, pero por sí solo
  fuerza un error en el primario y dispara el fallback contra el **proveedor real** de
  respaldo (llamada de red genuina). Para tests sin red (`tests/app/test_llm_wrapper.py`)
  fue necesario combinarlo con `mock_response` en `litellm_params` de cada deployment del
  `model_list` (monkeypatched vía `monkeypatch.setitem`), lo que permite testear el Router
  de producción tal cual está cableado (mismos `fallbacks`/`retry_policy`) sin red ni keys
  reales. No documentado explícitamente en la doc pública de LiteLLM consultada; confirmado
  leyendo el código fuente instalado (`router.py::_handle_mock_testing_fallbacks`).
- **[agnóstico] `response.model` tras normalización no lleva el prefijo de proveedor.**
  LiteLLM normaliza `response.model` al identificador físico sin `anthropic/`/`openai/`
  (p. ej. `"claude-haiku-4-5-20251001"`, `"gpt-5.4-mini-2026-03-17"`). `_provider_from_model`
  en `llm_wrapper.py` deriva el proveedor por el prefijo del nombre del modelo (`"claude"` /
  `"gpt"`), no por el nombre lógico pedido — necesario porque tras un fallback ambos
  difieren.
- **[dominio]** `litellm` no era dependencia del proyecto; se añadió con
  `uv add litellm` (arrastra `openai`, ya presente, y ~20 paquetes de soporte:
  `tiktoken`, `jsonschema`, etc.).

## Tarea abierta del brief — CERRADA en esta sesión

El brief marcaba como tarea abierta la validación end-to-end del fallback contra OpenAI con
key real. Se hizo con dos llamadas reales (no mock):

1. `call_llm(..., mock_testing_fallbacks=True)` con una pregunta ECSS real
   ("What safety and dependability requirements must the customer specify for software?"):
   respondió `gpt-5.4-mini-2026-03-17`, citando correctamente `ECSS-E-ST-40C 5.2.4.8` (el
   modelo sigue la instrucción de citación del system prompt aunque sea un proveedor
   distinto al que se diseñó el prompt), `finish_reason="stop"`, `usage` poblado
   (`tokens_in=201`, `tokens_out=91`).
2. La misma pregunta sin forzar fallback respondió con el primario
   (`claude-haiku-4-5-20251001`), como control.

**Conclusión:** el fallback a OpenAI está **validado**, no solo configurado y
mock-testeado — responde aceptablemente a una query ECSS real y su normalización trae
todos los campos esperados de `LLMResult`.

## Verificación manual (test de usuario)

Servidor levantado con `uv run uvicorn app.main:app`. `POST /api/v1/query` con una pregunta
ECSS real (sin forzar fallback): responde citando `ECSS-E-ST-40C 5.2.4.8`, con
`model="claude-haiku-4-5-20251001"`, `provider="anthropic"`, `usage` poblado — el contrato
HTTP de M1 no cambió.

## Tests automáticos

`uv run pytest` — 167 pasan (162 preexistentes + 5 nuevos en `tests/app/test_llm_wrapper.py`
+ 1 en `tests/app/test_config.py`; se ajustaron los mocks de `_call_model` en
`tests/app/test_queries.py` para incluir las claves nuevas `model_used`/`provider`/
`finish_reason` que ahora consume `answer_query`).

Cobertura de los 6 tests obligatorios del brief:
1. Fallback rota al respaldo — `test_fallback_routes_to_backup_on_primary_failure`.
2. Sin fallo, responde el primario — `test_primary_responds_without_fallback`.
3. Normalización completa — `test_llm_result_is_fully_normalized`.
4. Async — `test_call_llm_is_a_coroutine_function`.
5. Config fail-fast — `tests/app/test_config.py::test_settings_fails_fast_without_anthropic_key`.
6. Regresión de grounding/citación — cubierta por los tests preexistentes de
   `tests/app/test_queries.py` (ya validaban esto vía el mock de `_call_model`; siguen
   pasando tras el cambio de contrato interno).

## No implementado (fuera de alcance de este brief) — nada que reportar

Cache (brief 2), streaming (brief 3), logging estructurado de `fallback_used` (brief 4) y
Streamlit (brief 5) no se adelantaron. El wrapper deja las puertas abiertas que esos briefs
necesitan (`**provider_params`, `model_used`, `finish_reason`) sin implementarlas.

---

## Correcciones (`S03-brief-1-fix`)

### Fix 1 — Derivación de proveedor sin heurística de substring

**[agnóstico]** `_provider_from_model` pasó de un `if model_used.startswith("claude"/"gpt")`
a `litellm.get_llm_provider(model=model_used)`. Verificación previa (punto delicado que
pedía el fix): confirmado con `litellm==1.90.3` que `get_llm_provider` resuelve
correctamente **sobre el string normalizado sin prefijo** (`"claude-haiku-4-5-20251001"` →
`("claude-haiku-4-5-20251001", "anthropic", None, None)`; `"gpt-5.4-mini-2026-03-17"` →
`(..., "openai", ...)`), que era exactamente el punto donde la heurística original perdía
información. No hizo falta la vía alternativa (derivar del deployment del `model_list`) —
el resolvedor de LiteLLM ya funciona sobre el string pelado. Se envuelve en `try/except`
(no `except: pass` — se loguea `provider_resolution_failed` con `structlog` y se devuelve
`"unknown"`) porque `get_llm_provider` lanza `BadRequestError` sobre modelos que no
reconoce; en la práctica no debería dispararse (el modelo que llega es siempre uno que
acaba de responder), pero la derivación de proveedor no debe poder tumbar la respuesta.

Efecto: añadir un tercer proveedor al `model_list` no requiere tocar
`_provider_from_model` — el objetivo del fix.

### Fix 2 — Test de integración del path real

**[agnóstico]** Nuevo `tests/app/test_query_integration.py`: ejercita
`POST /api/v1/query` → `llm_service.answer_query` → `llm_wrapper.call_llm` → Router de
producción real, mockeando solo `litellm_params.mock_response` del deployment primario
(mismo patrón de `test_llm_wrapper.py`), sin tocar `_call_model`. Confirma que `citations`
sigue viniendo de `reference.py` y que `model`/`provider` en la respuesta HTTP son los del
deployment que realmente respondió. Los tests preexistentes de `test_queries.py` (mock de
`_call_model`) se mantienen — cubren el contrato interno, este cubre el camino real.

### Fix 3 — System prompt agnóstico de proveedor

**[agnóstico]** Revisión de `build_system_prompt` (`llm_service.py`): no se encontró
ninguna construcción específica de Anthropic (sin XML tags estilo Claude, sin fraseo de
identidad de proveedor, sin asunciones sobre cómo se transporta el rol `system` — eso lo
resuelve LiteLLM). El prompt ya era agnóstico de contenido; **no se modificó código**,
solo se añadió la validación que faltaba.

Se eleva el hallazgo n=1 del brief 1 a requisito de diseño verificado:
`tests/app/test_system_prompt_provider_agnostic.py`, marcado `@pytest.mark.real_llm`
y **excluido de la suite por defecto** (`addopts = "-m 'not real_llm'"` en
`pyproject.toml`; se ejecuta a mano con `uv run pytest -m real_llm`). Pega a las dos APIs
reales con el mismo `build_system_prompt(ECSS_REFERENCE)` — deliberado: mockear el texto de
respuesta no probaría que un proveedor real sigue la instrucción de citación, solo
confirmaría lo que nosotros mismos escribimos como mock. Ejecutado: **ambos proveedores
citan correctamente** (`ECSS-E-ST-40C`/`5.2.4.8`) con el prompt compartido.

**Límite honesto (tal como pedía el brief):** esto es un requisito comprobado contra los
dos proveedores que tenemos cableados (Anthropic, OpenAI), no una garantía universal. Un
tercer proveedor requeriría re-validar con este mismo test.

### Verificación en Docker tras las correcciones

`docker compose up --build -d` (imagen `rag-ecss-api`) → `/health` healthy → `POST
/api/v1/query` con pregunta ECSS real responde con `provider: "anthropic"` (ahora derivado
vía `get_llm_provider`, no por substring) y cita correcta.

### Tests automáticos tras las correcciones

`uv run pytest` — 168 pasan, 2 deselected (`real_llm`, se ejecutan aparte con
`-m real_llm` y también pasan, 2/2). Ruff limpio sobre los ficheros tocados.
