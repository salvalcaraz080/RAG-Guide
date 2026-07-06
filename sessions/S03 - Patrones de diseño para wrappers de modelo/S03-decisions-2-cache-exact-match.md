# S03-decisions-2-cache-exact-match.md — Implementación (Claude Code), brief 2/5

Decisiones y fricciones surgidas al implementar `S03-brief-2-cache-exact-match.md` sobre el
repo `RAG-ECSS`. Etiquetadas agnóstico (candidata a Guide) vs. dominio (solo ECSS).

## Verificaciones hechas antes de implementar

1. **[agnóstico] Cliente Redis async.** `uv add redis` instaló `redis==8.0.1`. Confirmado
   `redis.asyncio.Redis.from_url(url, decode_responses=True, ...)` y las firmas de
   `get(name)` / `set(name, value, ex=None, ...)` / `ping()` — todas devuelven
   `Awaitable` cuando el cliente se usa en modo async (import `redis.asyncio as redis`, no
   el cliente síncrono).
2. **[agnóstico] `fakeredis` async.** No estaba instalado, y añadirlo solo para tests
   habría sido una dependencia nueva para un problema que ya tenía solución con lo que
   había: `LLMCache` acepta un `client` inyectable en el constructor, así que los tests
   usan un doble en memoria propio (`FakeRedisClient` en `tests/app/conftest.py`) que
   implementa solo `get`/`set`/`ping`. Más simple que verificar la interfaz async de un
   paquete externo.

## Registradas por el brief

- **[agnóstico] `LLMCache` propia sobre `redis.asyncio`**, en `app/services/cache.py`,
  **por encima** del wrapper (no dentro de `llm_wrapper.py`). Un hit corta en
  `llm_service.answer_query` ANTES de tocar `_call_model`/`call_llm` — el wrapper nunca
  fabrica un `LLMResult` que finja una llamada que no ocurrió.
- **[agnóstico] Clave con slots explícitos** (`build_cache_key`): `question`, `model`,
  `system_prompt`, `context`, `corpus_version`, serializados con
  `json.dumps(..., sort_keys=True)` antes del sha256. `context` y `corpus_version` son
  slots separados de `system_prompt` a propósito — la razón de hacer la clave propia en
  vez de usar la caché integrada de LiteLLM.
- **[dominio] Refactor de `build_system_prompt`** en `llm_service.py`: se separó en
  `_STATIC_INSTRUCTIONS` (rol/grounding/citación/formato, constante) + `_build_context_block`
  (el material de referencia, antes iba todo junto). `build_system_prompt(fragments)` se
  mantiene con el mismo comportamiento observable (mismo string final) — el refactor es
  puramente para poder exponer `system_prompt` y `context` como slots independientes de la
  clave de caché, no cambia el prompt que ve el modelo ni el orden (rol → grounding →
  citación → formato → material, estático antes que variable).
- **[dominio] `CORPUS_VERSION = "m1-static"`** añadida en `context/reference.py` junto a
  `ECSS_REFERENCE`. Constante en M1 a propósito: existe para que M2, cuando el corpus se
  versione de verdad, invalide todas las claves de caché cambiando un solo valor.
- **[agnóstico] `REDIS_URL` no fail-fast**, igual que `OPENAI_API_KEY` en el brief 1:
  `LLMCache` degrada con warning (`cache_disabled_redis_unavailable` /
  `cache_get_failed` / `cache_set_failed` / `cache_ping_failed`) en vez de lanzar. Nunca
  `except: pass` — cada warning lleva el error real.
- **[agnóstico] `cache_hit` expuesto en `QueryResponse`** (recomendación del brief seguida):
  barato de exponer, útil para el test de usuario y para el brief 5 (Streamlit).
- **[agnóstico] `redis.asyncio`, nunca el cliente síncrono** — mismo principio que
  `router.acompletion()` en el brief 1: un cliente síncrono bloquearía el event loop de
  FastAPI igual que un LLM síncrono.

## Decisión no explícita en el brief, tomada durante la implementación

- **[dominio] `depends_on` sin `condition: service_healthy` en `docker-compose.yml`.** El
  brief presentaba dos opciones ("la app espera a que Redis esté healthy, o arranca igual y
  degrada"). Se eligió la segunda, coherente con el principio repetido en todo el brief
  ("Redis degradable, no bloqueante"): si `depends_on` esperara a `service_healthy` y Redis
  fallara al arrancar, el contenedor de la API tampoco arrancaría — contradice justo el
  diseño que el brief pedía. `depends_on: [redis]` (forma simple) solo ordena la creación
  de contenedores; `LLMCache` se encarga de degradar por su cuenta si Redis no responde.
- **[agnóstico] Startup ping vía `lifespan` de FastAPI**, no `@app.on_event("startup")`
  (deprecado). `main.py` llama a `cache.ping()` una vez al arrancar solo para dejar
  constancia en el log de arranque de si Redis está disponible — el resultado no bloquea
  nada, `ping()` ya loguea su propio warning si falla.
- **[agnóstico] `/health` expone `cache: "up"|"degraded"`** (opcional según el brief,
  implementado): útil para diagnóstico rápido y para el test de usuario, sin que un Redis
  caído tumbe el healthcheck del contenedor (`status` sigue siendo `"healthy"` siempre que
  el proceso esté vivo).

## Fricción de implementación no prevista por el brief

- **[dominio] Colisión con el Redis de OTRO proyecto en la misma máquina.** `REDIS_URL`
  por defecto (`redis://localhost:6379/0`) coincide con el puerto donde un contenedor
  `redis` de un proyecto completamente distinto (imagen `redis/redis-stack`) ya estaba
  escuchando en esta máquina. Al ejecutar `uv run pytest` antes de aislar los tests, la
  suite escribió 2 claves reales `llmcache:*` en ese Redis ajeno (namespaced, no
  colisionaron con nada de ese proyecto, pero tocaron un recurso compartido sin permiso).
  **Corrección aplicada:**
  1. Se limpiaron esas 2 claves con confirmación explícita del usuario
     (`docker exec redis redis-cli del ...`).
  2. Se añadió `tests/app/conftest.py` con una fixture `autouse=True`
     (`isolate_cache_from_real_redis`) que fuerza `llm_service.cache._client = None` en
     TODOS los tests de `tests/app/` por defecto — ningún test toca un Redis real salvo que
     pida explícitamente `fake_cache` (que inyecta un `FakeRedisClient` en memoria).
  3. En `docker-compose.yml`, el servicio `redis` de este proyecto **no publica puerto al
     host** (`ports:` omitido) — solo `api` lo alcanza, por nombre de servicio dentro de la
     red interna de Compose. Evita que este proyecto vuelva a chocar con el puerto 6379 de
     otro, en cualquier dirección.
- **[agnóstico] `redis.Redis.from_url(...)` no conecta en el constructor.** Es perezoso: el
  `try/except` alrededor de `from_url` en `LLMCache.__init__` solo captura errores de URL
  malformada, no de conectividad real (eso ocurre en el primer `get`/`set`/`ping`, ya
  cubierto por sus propios `try/except`). Documentado en el docstring de `LLMCache` para
  que no se asuma que el constructor ya valida la conexión.

## Verificación en Docker (test de usuario del brief)

`docker compose up --build -d` (servicios `rag-ecss-api` + `rag-ecss-redis`, sin puerto de
Redis publicado al host) → `/health` → `{"status": "healthy", "cache": "up"}`.

Pregunta ECSS real dos veces por `POST /api/v1/query`:
- 1ª llamada: `cache_hit: false`, ~4.8s (llamada real al modelo).
- 2ª llamada (idéntica): `cache_hit: true`, ~0.18s — **hit visible e instantáneo**.
- Pregunta distinta: `cache_hit: false` de nuevo (la clave cambia con la pregunta).
- `docker exec rag-ecss-redis redis-cli --scan --pattern "llmcache:*"` → 2 claves, en el
  Redis **propio** de este proyecto, no en el de terceros.

## Tests automáticos

`uv run pytest` — 188 pasan, 2 deselected (`real_llm`, sin cambios respecto al brief 1-fix).
Nuevos:
- `tests/app/conftest.py` — `FakeRedisClient` + fixtures `isolate_cache_from_real_redis`
  (autouse) y `fake_cache` (opt-in).
- `tests/app/test_cache.py` — unit tests de `LLMCache`/`build_cache_key`: determinismo,
  sensibilidad a cada slot (parametrizado, obligatorio #3), roundtrip get/set, degradación
  sin cliente y con cliente que lanza (obligatorio #4), operaciones async (obligatorio #5).
- `tests/app/test_llm_service_cache.py` — integración vía `answer_query`: miss llama y
  guarda (#1), hit no llama (#2), sensibilidad de la clave a cada slot vía `answer_query`
  real (pregunta, modelo, system_prompt, context, corpus_version — #3), degradación sin
  Redis (#4), async (#5).
- `tests/app/test_query_integration.py` — nuevo test HTTP end-to-end
  (`test_cache_hit_is_visible_over_http_and_skips_the_model`): dos llamadas por
  `POST /api/v1/query`, la 2ª es hit y el wrapper real solo se llama una vez (`spy` con
  `AsyncMock(wraps=...)`).
- `tests/app/test_queries.py` — aserción `cache_hit is False` añadida a la regresión
  existente (#6, contrato HTTP no roto más allá del campo nuevo).

Ruff limpio sobre `app/` y `tests/app/`.

## No implementado (fuera de alcance de este brief) — nada que reportar

Prompt caching de proveedor (`cache_control`), cache semántico, y logging de `cache_hit`
como coste evitado (brief 4) quedan fuera, tal como el brief especificaba. El valor
cacheado conserva el `usage` original completo para que el brief 4 pueda calcular ese
ahorro sin cambios adicionales aquí.

## Corrección post-implementación: `model` retirado de la clave de caché

**Revisión del usuario tras cerrar el brief:** el brief especificaba `model` como slot de
la clave ("misma pregunta, mismo modelo, mismo contexto"). El usuario cuestionó esa
decisión: la respuesta debería depender de la pregunta y el contexto, no de qué modelo la
generó.

**Análisis antes de aplicar el cambio (ambas caras):**
- **A favor de mantener `model`:** protege contra el caso real de cambiar `LLM_MODEL` en
  `.env` (ya ha pasado dos veces en esta sesión, al verificar vigencia de modelos) — sin
  ese slot, las respuestas cacheadas con el modelo anterior siguen sirviéndose hasta que
  expiran por `CACHE_TTL` (hasta 24h), silenciosamente.
- **A favor de quitarlo (razón del usuario):** en la práctica de M1, `LLM_MODEL` casi nunca
  cambia en caliente; mantenerlo añade una superficie de invalidación que rara vez se
  ejercita. Además —matiz que se hizo explícito antes de decidir— el slot `model` en la
  clave era `settings.LLM_MODEL` (el modelo **configurado**), no `model_used` (quién
  **realmente** respondió). Es decir, ni siquiera con `model` en la clave se evitaba que una
  respuesta servida por el fallback de OpenAI se sirviera desde caché en una consulta
  posterior donde Anthropic ya estuviera sano de nuevo — ese problema requeriría
  `model_used` en la clave, que el brief no pedía. Mantener `model` no resolvía el riesgo
  más serio de contaminación por fallback; solo protegía el caso de cambio deliberado de
  config.

**Decisión: se retira `model` de la clave.** Slots finales: `question`, `system_prompt`,
`context`, `corpus_version`. Cambios:
- `cache.py::build_cache_key` — parámetro `model` eliminado; docstring documenta el límite
  aceptado explícitamente.
- `llm_service.py::answer_query` — ya no pasa `model=settings.LLM_MODEL` a
  `build_cache_key`.
- `tests/app/test_cache.py` — `BASE_KWARGS` y el test parametrizado de sensibilidad por
  slot ya no incluyen `model`.
- `tests/app/test_llm_service_cache.py` — `test_key_changes_with_model` (que exigía miss
  al cambiar el modelo) sustituido por `test_model_change_does_not_invalidate_cache`, que
  **fija a propósito** el comportamiento contrario: cambiar `LLM_MODEL` NO invalida una
  entrada ya cacheada. Así, si alguien reintroduce el slot sin darse cuenta, un test falla
  y obliga a que sea una decisión consciente, no un accidente.

**Límite aceptado y documentado (no una laguna descubierta después):** tras un cambio de
`LLM_MODEL`, las respuestas ya cacheadas con el modelo anterior siguen sirviéndose hasta
expirar por `CACHE_TTL`. Si en el futuro esto importa (p. ej. se detecta una regresión de
calidad real por servir respuestas de un modelo retirado), la mitigación más simple es
bajar `CACHE_TTL` o vaciar el namespace `llmcache:*` a mano tras el cambio de modelo — no
hace falta reintroducir el slot para eso.

Re-verificado tras el cambio: `uv run pytest` — 187 pasan (188 → 187, un test sustituido
por otro), ruff limpio, Docker rebuild con `docker compose up --build -d`: hit visible
(`cache_hit: false` → `cache_hit: true`) sigue funcionando idéntico.
