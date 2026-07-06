# S03 · Brief 2 — Cache exact-match (`LLMCache` propia sobre Redis)

> Instrucciones para Claude Code. Este chat decide, CC implementa. Al cerrar: `S03-decisions.md`
> (append a la sección de S03), derivar `CHANGELOG.md`, editar `CLAUDE.md`/`ARCHITECTURE.md`.
> **Brief 2 de 5.** Depende del brief 1 (wrapper) ya implementado y corregido. El brief 4
> (observabilidad) consumirá el `cache_hit` que este brief produce; el brief 3 (streaming) hereda de
> aquí una bifurcación (un hit no se streamea). Ver "Dependencias hacia adelante".

## Objetivo

Añadir **cache exact-match** de respuestas: una consulta idéntica (misma pregunta,
mismo contexto) devuelve la respuesta guardada sin volver a llamar al modelo. Implementación **propia**
(`LLMCache`) sobre **Redis**, ubicada en `llm_service` **por encima** del wrapper.

Alcance cerrado: **solo exact-match.** El prompt caching de proveedor (`cache_control` de Anthropic)
queda diferido a M2 (el prefijo M1 plausiblemente no alcanza el mínimo de 1024 tokens; ver
`S03-resumen.md`). El cache semántico queda diferido a S07/S08. Este brief no los toca.

## Por qué así (decisiones ya tomadas)

- **`LLMCache` propia, no la caché integrada de LiteLLM.** Razón: control total de la **clave**. En
  M1 cualquiera de las dos serviría, pero en M2 el contexto lo inyectará retrieval dinámicamente y la
  clave deberá incluirlo, o se servirían respuestas viejas generadas sobre chunks que retrieval ya no
  devolvería (una **cita fantasma servida desde caché** — el modo de fallo que el producto entero
  existe para evitar). Controlar la clave desde M1 hace esa transición segura.
- **Cache en `llm_service`, por encima del wrapper.** El wrapper (`llm_wrapper.call_llm`) queda puro
  "hablar con proveedores". Un cache hit corta **antes** de llamar al wrapper → no se fabrica un
  `LLMResult` que finja una llamada que no ocurrió.
- **Redis degradable, no bloqueante.** Coherente con cómo el brief 1 dejó `OPENAI_API_KEY` (opcional,
  warning, no tumba el arranque). El cache es optimización, no correctitud: si Redis no responde, se
  loguea y se sirve sin cache. Tumbar el servicio por falta de la optimización sería frágil.

## Redis en el stack (servicio nuevo)

- **`docker-compose`:** añadir servicio `redis` (imagen oficial). Healthcheck propio. El servicio de
  la app espera a que Redis esté healthy (o arranca igual y degrada — ver abajo).
- **`Settings` (`config.py`):** `REDIS_URL` (p. ej. `redis://localhost:6379/0`). A diferencia de
  `ANTHROPIC_API_KEY` (fail-fast), `REDIS_URL` **no** debe tumbar el arranque si Redis no está: la
  conexión se intenta y, si falla, el cache se marca deshabilitado con warning
  (`cache_disabled_redis_unavailable`). Decidir: ¿fallo de conexión perezoso (al primer uso) o al
  arranque con degradación? Recomendado: intentar al arranque, degradar con warning, reintentar
  conexión de forma perezosa en usos posteriores.
- **Dependencia:** `uv add redis` (cliente async: `redis.asyncio`, para no romper el modelo async del
  wrapper — ver "Async").
- **`CLAUDE.md` (tests):** la imagen debe levantar y `/health` responder antes de correr tests. Con
  Redis añadido, `docker compose up --build` ahora levanta app **+ redis**; verificar que ambos
  suben y que `/health` responde. Considerar reflejar en `/health` el estado del cache (opcional:
  `cache: "up" | "degraded"`), útil para el test de usuario.

## La clave de caché — el núcleo del brief

El material hashea `prompt + system_prompt`. Para M1 basta, pero **por una razón frágil**: el
contexto ECSS está *dentro* del system prompt (fragmento estático), así que hashear el system prompt
captura el contexto por accidente. Eso enseñaría una lección equivocada para M2. 

**Diseñar la clave con slots explícitos desde M1**, aunque algunos sean estáticos ahora:

```
cache_key = sha256(canonical_json({
    "question":       <str>,   # la pregunta del usuario
    "model":          <str>,   # nombre lógico o físico del modelo (afecta la respuesta)
    "system_prompt":  <str>,   # rol/grounding/citación/formato (sin el contexto, ver abajo)
    "context":        <str>,   # M1: el fragmento ECSS estático. M2: los chunks recuperados.
    "corpus_version": <str>,   # M1: constante ("m1-static" o similar). M2: versión real del corpus.
}))
```

Dos slots que el material no tiene y que son la razón de hacer la clave propia:

- **`context` como componente separado del `system_prompt`.** En M1 se puede poblar con el mismo
  fragmento estático (extraído de `context/reference.py`), pero vive en su **propio slot**. Así, en
  M2, pasar de contexto estático a contexto dinámico de retrieval es **rellenar el slot con los
  chunks recuperados** (o un hash de sus IDs), no rediseñar la clave. Si el contexto cambia, la clave
  cambia, y no se sirve una respuesta vieja sobre contexto nuevo.
- **`corpus_version`.** En M1 es constante (el "corpus" es un fragmento fijo). Existe para que, cuando
  el corpus se versione en M2, un cambio de estándar (nueva revisión ECSS) cambie **todas** las
  claves automáticamente → las entradas viejas dejan de acertar y expiran por TTL. Es la
  "invalidación por versión de corpus" que decidimos preferir sobre TTL puro para corpus normativo.

**Serialización canónica:** `json.dumps(..., sort_keys=True)` antes del hash, para que el orden de las
claves no altere el hash. Namespace de la clave en Redis: prefijo claro (`llmcache:` o similar).

**Qué se guarda como valor** (no un `LLMResult` que finja una llamada): la respuesta observable de la
consulta original — `text`, `citations`, `model`, `provider`, `tokens_in`, `tokens_out`,
`finish_reason` — más un marcador de que procede de caché al recuperarla. En un hit, `usage` es el de
la llamada **original** (no hubo tokens nuevos); esto le importa al brief 4 (coste evitado, no coste
cero). Ver "Dependencias hacia adelante".

## TTL e invalidación

- **TTL** por defecto configurable (`CACHE_TTL`, p. ej. 24h como punto de partida). Para el corpus
  normativo el TTL no es el mecanismo principal de frescura (la respuesta correcta cambia cuando
  cambia el corpus, no con el tiempo) — pero es la red de seguridad. La invalidación real la da
  `corpus_version` en la clave.
- **No** implementar invalidación por evento explícita en M1 (borrado de tags/namespaces): con
  `corpus_version` en la clave, el versionado del corpus ya invalida implícitamente. Anotarlo como
  suficiente para M1; reevaluar en M2 si hiciera falta borrado activo.

## Ubicación e integración

- **`app/services/cache.py`** (nuevo): clase `LLMCache` con `get(key)` / `set(key, value, ttl)` async
  sobre `redis.asyncio`. Aislada, testeable con un Redis mockeado o `fakeredis`.
- **`app/services/llm_service.py`:** `answer_query` consulta el cache **antes** de construir el prompt
  completo y llamar al wrapper:
  1. Construye la clave (pregunta + modelo + system_prompt + context + corpus_version).
  2. `get(key)` → si hit: devuelve la respuesta guardada marcada como `cache_hit=True`, sin tocar el
     wrapper.
  3. Si miss: llama al wrapper (`call_llm`), construye la respuesta (con sus citas desde
     `reference.py`), `set(key, ...)`, y devuelve con `cache_hit=False`.
- El campo `cache_hit` viaja en la respuesta interna hacia `QueryResponse`. Decidir si se expone en el
  contrato HTTP público o solo internamente para el log del brief 4. Recomendado: exponerlo (barato,
  y útil para el test de usuario y el sidebar de Streamlit del brief 5).

## Async

`redis.asyncio` (no el cliente síncrono). El `get`/`set` del cache va en el path async de
`answer_query`; un cliente Redis síncrono bloquearía el event loop igual que un LLM síncrono lo haría
(misma lección del brief 1). No romper el modelo de concurrencia.

## Dependencias hacia adelante (no implementar aquí, dejar la puerta abierta)

- **Brief 4 (observabilidad):** `cache_hit=True` significa "tokens ya pagados una vez, no ahora". El
  log de coste debe contarlo como **coste evitado**, no coste cero. El valor cacheado conserva el
  `usage` original justamente para poder calcular ese ahorro. No loguear aquí; solo garantizar que el
  dato está.
- **Brief 3 (streaming):** un cache **hit no se streamea** (no hay generación que escribir token a
  token; se devuelve entero, instantáneo). El endpoint de streaming tendrá que ramificar: consultar
  cache primero; si hit → responder entero sin SSE; si miss → streamear la generación real. Este brief
  no toca streaming, pero deja constancia de la bifurcación para que el brief 3 la herede.

## Tests obligatorios (`tests/app/`, sin red real — `fakeredis` o mock)

1. **Miss llama y guarda.** Primera consulta: cache miss → llama al wrapper (mockeado) → guarda en
   Redis → `cache_hit=False`.
2. **Hit no llama.** Segunda consulta idéntica: cache hit → **no** se invoca el wrapper (verificar con
   el mock que no fue llamado) → `cache_hit=True` → respuesta idéntica a la primera.
3. **La clave incluye todos los slots.** Cambiar cualquiera de {pregunta, modelo, system_prompt,
   context, corpus_version} produce una clave distinta → cache miss. Un test por slot (o parametrizado)
   que confirme que variar ese componente rompe el hit. **Especialmente** `context` y `corpus_version`:
   son la garantía anti-cita-fantasma de M2.
4. **Degradación sin Redis.** Si el cliente Redis no está disponible, `answer_query` sigue
   respondiendo (llama al wrapper, no cachea) y loguea el warning — no lanza excepción al usuario.
5. **Async.** Las operaciones de cache son coroutines; no bloquean bajo `pytest-asyncio`.
6. **Regresión.** El path `POST /api/v1/query` sigue devolviendo respuesta con citas desde
   `reference.py`; el contrato HTTP de M1 no cambia (más allá del posible campo `cache_hit`).

Sin `except: pass`. Errores de Redis (conexión, timeout) se capturan explícitamente y degradan a
"sin cache", no se silencian ni se propagan como 500 al usuario.

## Verificaciones que CC debe hacer

1. **Cliente Redis async.** Confirmar el import y API vigentes de `redis.asyncio` en la versión que
   `uv add redis` instale (la API ha cambiado entre versiones de `redis-py`).
2. **`fakeredis` async.** Si se usa `fakeredis` para tests, confirmar que soporta la interfaz async
   (`fakeredis.aioredis` o equivalente vigente); si no, mockear el cliente.

## Cierre (CC)

Implementación → `uv run pytest` (los previos verdes + los nuevos) → **imagen Docker con Redis**
(`docker compose up --build`, app + redis suben, `/health` responde) → test de usuario (consulta
ECSS dos veces: la 2ª debe ser un hit visible — instantánea, `cache_hit=True`) → docs
(`S03-decisions.md` con etiquetado agnóstico/dominio, `CHANGELOG.md`, `CLAUDE.md`/`ARCHITECTURE.md`).
El commit lo hace el usuario.
