# S03-consolidado.md — Parte 1: hito backend (briefs 1-4)

> Paso 5 del flujo. Resumen destilado + aprendizajes de **implementar** los briefs 1-4 de S03
> (wrapper+fallback, cache, streaming, observabilidad). **Histórico: no se reescribe.** El
> `S03-resumen.md` (teoría, paso 2) no se borra; este documento registra qué confirmó, qué divergió y
> qué añadió la implementación respecto a aquella teoría. Es la fuente desde la que se gradúa
> `RAG-GUIDE.md` (paso 6). El segundo consolidado (Streamlit, brief 5) cierra el segundo hito.

Estado al cierre del hito: `POST /api/v1/query` (no-stream) y `POST /api/v1/query/stream` (SSE),
ambos con abstracción de proveedor + fallback, cache exact-match, y traza operacional. 205 tests
verdes (2 `real_llm` deselected). Docker: app + redis, `/health` con `cache: up|degraded`.

---

## Bloque 1 — Abstracción de proveedor + fallback (LiteLLM Router)

**Decidido (teoría) y confirmado (implementación).** LiteLLM Router detrás del seam `_call_model`,
async-nativo, fallback secuencial Anthropic→OpenAI. Se eligió Router completo desde el primer brief
(no `acompletion` simple) para tener el fallback montado y testeable con mock desde ya.

**Confirmado por implementación:**
- **async ≠ sync no es solo rendimiento — cambia la lógica de fallback.** Verificado leyendo
  `litellm/router.py` en la instalación real: el camino síncrono y el async llevan lógicas de
  fallback distintas. `router.acompletion()` es requisito duro, no preferencia. (En teoría lo
  apoyábamos en la doc; la implementación lo confirmó en fuente.)
- `usage` (`prompt_tokens`/`completion_tokens`) viene poblado y uniforme para ambos proveedores sin
  mapeo adicional — una incógnita de la teoría que se cerró a favor de la simplicidad.
- El fallback se **validó end-to-end** (no solo mock): con key real, forzando el fallback, `gpt-5.4-mini`
  respondió a una query ECSS real citando correctamente, `finish_reason="stop"`, `usage` poblado.

**Aprendizajes nuevos (fricción de implementación, no estaban en teoría):**
- **Gotcha de testing del Router:** `mock_testing_fallbacks=True` **por sí solo dispara una llamada de
  red real** al fallback (fuerza un error en el primario y rota al proveedor real). Para tests sin red
  hay que combinarlo con `mock_response` en los `litellm_params` de cada deployment del `model_list`
  (monkeypatched). Así se testea el Router de producción tal cual está cableado —mismos `fallbacks`/
  `retry_policy`— sin red ni keys. No está en la doc pública; se descubrió en fuente.
- **Derivar el proveedor por substring del nombre de modelo no escala.** La primera implementación usó
  `"claude"`/`"gpt"` como heurística (necesaria porque `response.model` normalizado pierde el prefijo
  `anthropic/`/`openai/`). Se corrigió (brief 1-fix) a `litellm.get_llm_provider()`. Regla: con N
  proveedores, resolver el proveedor con el resolvedor nativo, no con substrings.

**Decisiones de diseño que emergieron:**
- **`LLMResult` normalizado** (`text`, `model_used`, `provider`, `tokens_in/out`, `finish_reason`),
  con `model_used`/`finish_reason`/`**provider_params` presentes desde ya aunque el brief no los
  consuma — puertas abiertas para cache (passthrough), streaming (`finish_reason`) y observabilidad
  (`fallback_used`). Diseñar los campos del contrato pensando en los consumidores futuros evitó
  rediseñar el wrapper en cada brief posterior.
- **El fallback es observable por el cliente HTTP, no solo por el log:** `QueryResponse.model`/
  `provider` reflejan quién **realmente** respondió (`model_used`), no el primario configurado.
- **System prompt agnóstico de proveedor** (brief 1-fix) como requisito de diseño, con test de
  regresión que valida citación correcta por ambos proveedores. Límite honesto: "agnóstico" es
  **comprobado** contra los dos proveedores cableados, no **garantizado** universalmente — LiteLLM
  uniformiza el envío, pero cada modelo reacciona distinto; con un tercero, se re-valida.

**Decisiones de dominio:** `OPENAI_API_KEY` opcional (degradable con warning, no fail-fast); modelo de
respaldo `gpt-5.4-mini` (verificado vigente; el `gpt-4o-mini` del material está stale);
`retry_policy` diferenciado (auth=0, timeout=2, rate-limit=2, server=1).

**Deuda/riesgo señalado:** LiteLLM tuvo un compromiso de supply-chain en PyPI (marzo 2026) → fijar
versión y verificar hashes. Aplica a cualquier dependencia.

---

## Bloque 2 — Cache exact-match (`LLMCache` propia sobre Redis)

**Decidido (teoría) y confirmado (implementación).** `LLMCache` propia sobre `redis.asyncio`, **por
encima** del wrapper (un hit corta en `answer_query` antes de tocar el wrapper — no se fabrica un
`LLMResult` que finja una llamada). Se eligió implementación propia sobre la caché integrada de
LiteLLM por el **control de la clave**.

**El núcleo — clave con slots explícitos desde M1:**
`question + system_prompt + context + corpus_version`, serializados canónicamente (`sort_keys=True`)
antes del sha256.
- `context` y `corpus_version` son slots **separados** aunque en M1 sean estáticos (el fragmento ECSS
  y `"m1-static"`). Razón: en M2, pasar a contexto dinámico de retrieval es **rellenar el slot**, no
  rediseñar la clave. Si el contexto cambia, la clave cambia → no se sirve una respuesta vieja sobre
  contexto nuevo (**cita fantasma servida desde caché**, el modo de fallo que el sistema existe para
  evitar). Requirió refactorizar `build_system_prompt` en `_STATIC_INSTRUCTIONS` + `_build_context_block`
  (mismo prompt final, componentes expuestos por separado).
- `corpus_version` existe para que M2 invalide todas las claves cambiando un solo valor cuando el
  corpus se versione (invalidación por versión de corpus, preferida sobre TTL puro para corpus
  normativo).

**Divergencia teoría→implementación (decisión revisada tras cerrar el brief): `model` NO es slot.**
El brief lo especificaba; el usuario lo cuestionó. Análisis: el slot habría sido `settings.LLM_MODEL`
(el modelo **configurado**), no `model_used` (quién **realmente** respondió) — por tanto **nunca
protegió del riesgo real** (una respuesta del fallback cacheada y servida cuando el primario ya está
sano). Solo protegía el cambio deliberado de `LLM_MODEL`, caso menor. Se retiró. Un test invertido
(`test_model_change_does_not_invalidate_cache`) fija el comportamiento a propósito: reintroducir el
slot rompe un test, obligando a que sea decisión consciente. **Límite aceptado y documentado:** tras
cambiar `LLM_MODEL`, las entradas viejas se sirven hasta expirar por `CACHE_TTL`; mitigación simple si
importa (bajar TTL o vaciar namespace), sin reintroducir el slot.

**Aprendizaje nuevo (fricción — el más valioso del bloque, agnóstico pese a nacer en dominio):**
- **Contaminación cruzada de recursos entre proyectos en la misma máquina.** `REDIS_URL` por defecto
  (`localhost:6379`) coincidió con el Redis de **otro proyecto** en la misma máquina; la suite escribió
  claves reales en ese recurso ajeno antes de aislar. Corrección de tres capas: (1) fixture `autouse`
  que fuerza `cache._client = None` en todos los tests por defecto (opt-in al recurso real, no
  opt-out); (2) el Redis del proyecto **no publica puerto al host** en `docker-compose` (solo lo
  alcanza `api` por la red interna); (3) limpieza de las claves con confirmación. Regla general:
  **cualquier store con puerto por defecto compartido es un riesgo de contaminación cruzada; los tests
  deben aislarse por defecto y el servicio no debe exponer puerto si solo lo consume otro contenedor.**

**Decisiones de diseño:** Redis degradable, no fail-fast (`REDIS_URL` no tumba el arranque; `LLMCache`
degrada con warning explícito — nunca `except: pass`). `depends_on: [redis]` sin `condition:
service_healthy` (esperar a healthy contradiría el diseño degradable). `/health` expone
`cache: up|degraded` sin que un Redis caído tumbe el healthcheck. `cache_hit` en `QueryResponse`. El
valor cacheado conserva el `usage` original (para el coste evitado del brief 4).

**Deuda diferida (nodo con señal disparadora):** contaminación de cache por fallback — una respuesta
del respaldo se cachea y se sirve cuando el primario ya está sano. Resolverla exige `model_used` en la
clave/valor. No se hace en M1 (riesgo bajo: dos proveedores validados como equivalentes, exact-match
estático). Señal: si sube el volumen de fallbacks o se detecta regresión de calidad, entonces toca.

---

## Bloque 3 — Streaming SSE (`POST /api/v1/query/stream`)

**Decidido (teoría) y confirmado (implementación).** SSE encima del path no-stream, que es la fuente
de verdad. `answer_query` y `stream_answer_query` comparten `_cache_lookup` y `_assemble_response`/
`_citations` — el streaming no tiene lógica de negocio propia, solo cambia cómo entrega. Verificado
con test que compara las citas de ambos paths. Dos tramos separados: A (`stream_llm`, LiteLLM
proveedor→backend, no sabe de FastAPI) y B (`query_stream`, endpoint→cliente, no sabe del prompt/cache).

**El cache decide si hay stream** (corrección del usuario en teoría, confirmada en implementación): un
hit no llama al modelo → no hay generación → no se streamea. El endpoint consulta cache primero; miss
→ SSE con chunks de texto + `event: done` tipado (citas, usage, finish_reason); hit → respuesta
instantánea. El cache no se "mezcla" con el stream: decide si el request entra siquiera en el camino
de streaming.

**Divergencia teoría→implementación (mecanismo — material de Guide explícito): `fastapi.sse` decide
el modo SSE por RUTA, no por petición.** El brief asumía que se podía decidir por petición (hit →
JSON plano, miss → abrir SSE instanciando `EventSourceResponse` a mano en la rama miss). **Falso:** la
serialización `ServerSentEvent`→texto vive en la capa de routing de FastAPI, que solo activa SSE si la
ruta se declara con `response_class=EventSourceResponse` **y** el endpoint es un generador. Intentar
instanciarlo a mano lanza `AttributeError: 'ServerSentEvent' object has no attribute 'encode'`.
Resolución (consultada): en vez de reinventar el formateo SSE con `StreamingResponse` (lo que el brief
pedía NO reinventar), **el hit también abre SSE** pero emite un único `event: done` sin chunks previos
(~0.19s, cero llamadas al LLM, sin diferencia observable para el usuario). Además simplificó el
endpoint (no hace falta `anext()` para adelantar el primer evento). **Lección para la Guide:** con
`fastapi.sse`, el modo stream es propiedad de la ruta; si necesitas alternar JSON/SSE por petición,
o son dos rutas, o formateas SSE a mano — no hay punto medio con este mecanismo.

**Aprendizajes nuevos (fricción):**
- **`stream_options={"include_usage": True}` es obligatorio** para que LiteLLM incluya un chunk con
  `usage` poblado durante el streaming. Sin él no hay token count que cachear ni reportar. Verificado
  empíricamente.
- **`fastapi.sse` es un submódulo, no atributo top-level.** `hasattr(fastapi, "EventSourceResponse")`
  da falso negativo; hay que `from fastapi.sse import ...`. Real desde `fastapi>=0.135` (instalada
  0.136.1). El material afirmaba soporte nativo 0.135.0 — resultó cierto, contra pronóstico cauto.
- El "estimador del máster" es un proyecto **hermano** (LIDR), no una versión anterior de este repo —
  `git grep` sobre el historial de RAG-ECSS dio cero rastro de streaming. Verificar de dónde se porta
  un patrón antes de asumir que existe en el repo actual.

**Dominio:** `truncated` (`finish_reason=="length"`) marcado en el `done` como fallo de integridad, no
cosmético (respuesta posiblemente cortada antes de su cita). Error a mitad de stream → `event: error`
tipado, nunca conexión colgada ni `except: pass`. Descartada la estrategia del material "limitar
salida por prompt a N palabras" (compite con completitud normativa).

---

## Bloque 4 — Observabilidad operacional (structlog)

**Alcance (confirmado):** traza **operacional** (modelo, tokens, coste, latencia, cache_hit,
fallback_used, finish_reason), **no** traza de cita (qué cláusula fundamentó qué afirmación — S11). No
mezclar las dos acepciones de "trazabilidad".

**Auditoría previa (paso 0 del brief) — resultado:** `structlog>=24.0` instalado y ya en uso puntual
(`cache.py`, `llm_wrapper.py` emitían warnings), pero **nunca configurado** (`structlog.configure`
ausente en todo el repo). Corriendo con defaults: salía legible pero sin filtrar por `LOG_LEVEL`, sin
JSON para producción, sin `request_id`. Escenario "configura desde cero". Los módulos que ya usaban
`get_logger()` recogen la config automáticamente (structlog es perezoso), sin tocarlos.

**Divergencia teoría→implementación (corrección de mi brief): `cost_per_token`, no `completion_cost`.**
El brief especificaba `completion_cost()`; la función correcta para coste sobre tokens ya contados es
`litellm.cost_per_token(model, prompt_tokens, completion_tokens)`. `completion_cost` espera un
`ModelResponse` completo o recalcula tokens desde texto — redundante teniendo los contadores exactos
del `usage`. Corregido en implementación.

**Aprendizaje que refuerza la regla del coste:** `cost_per_token` **lanza excepción** para un modelo
no mapeado (no devuelve `0` silencioso). Simplifica la regla "nunca 0 engañoso": `try/except` +
tratar excepción y `==0` como `None`. Ambos modelos cableados tienen precio plausible en el map de
LiteLLM (no hace falta derivar coste offline por ahora).

**Decisiones de diseño:**
- `app/observability.py::configure_logging(settings)` dual: dev `ConsoleRenderer` / resto
  `JSONRenderer`; `wrapper_class` con filtrado real por `LOG_LEVEL`. `RequestIDMiddleware` vincula
  `request_id` por request vía `contextvars` → todos los logs de una request lo llevan sin pasarlo a
  mano, y permite reconstruir la traza completa filtrando por él.
- **`configure_logging()` se llama en `main.py` ANTES de importar los routers**, no en `lifespan`:
  `lifespan` corre tras los imports, y un warning a nivel de módulo (`openai_fallback_disabled`) debe
  salir ya con la config final. Sutil pero correcto.
- Traza operacional en `llm_service` (el seam, no el wrapper — que no sabe de cache ni de request),
  compartida por ambos paths sin duplicar (mismo principio que `_cache_lookup`). El evento `cache_hit`
  vive dentro de `_cache_lookup` (único sitio que decide qué es un hit), con `cost_avoided_usd`
  (coste que *habría* tenido la llamada, no cero) y `fallback_used` derivado retroactivamente.
- **`fallback_used` observable junto a `cache_hit`** en la misma traza: hace la deuda de contaminación
  de cache por fallback (bloque 2) **observable en análisis** (cruzando ambos campos) sin resolverla en
  runtime. Decisión de alcance: observar la deuda, no mecanizar su solución todavía.
- `ttft_ms` (time-to-first-token) añadido en el path de stream vía `**extra` — aterriza el
  forward-reference del brief 3. Separa el tiempo al primer token del total: es la métrica que
  justifica el streaming (mide la UX que promete).

**Dominio/nodo agnóstico anotado:** superficie de exposición del log — no se loguea texto de prompts/
respuestas por defecto, solo metadatos. Para ECSS (corpus público) no muerde; en un corpus con datos
sensibles, el log es superficie de exposición (GDPR, S06). Contenido para debug solo a nivel DEBUG y
consciente.

---

## Divergencias teoría-vs-implementación del hito (resumen para graduar a Guide)

Dos correcciones de mecanismo sobre briefs de este chat, ambas descubiertas por contacto con el
runtime/API real — material de Guide sobre "el material (y el diseño en papel) simplifica mecanismos":

1. **`fastapi.sse` es por-ruta, no por-petición** (bloque 3). Un diseño que asuma alternar JSON/SSE
   dentro del mismo endpoint es irrealizable con este mecanismo.
2. **`cost_per_token` vs `completion_cost`** (bloque 4). La función depende de si ya tienes los tokens
   contados (sí → `cost_per_token`) o un `ModelResponse` completo (→ `completion_cost`).

Ambas confirman el principio del proyecto: los snippets/diseños en papel se contrastan contra la API
real antes de darlos por buenos.

---

## Estado abierto al cierre del hito backend

- **Prompt caching de proveedor** (`cache_control` de Anthropic): diferido a M2 (prefijo M1 no llega a
  1024 tokens). Gate real cuando el retrieval infle el prefijo; ahí verificar passthrough vía LiteLLM.
- **Clave de cache sobre retrieval dinámico** (M2): poblar el slot `context` con los chunks
  recuperados / hash de IDs.
- **Contaminación de cache por fallback:** nodo de deuda con señal disparadora (arriba, bloque 2/4).
- **Cache semántico** (S07/S08) + sospecha de dominio (criticidad B vs C casi idénticas en embedding).
- **Herramienta de observabilidad visual** (Logfire/Langfuse): nodo de escalado, no MVP.
- **Streamlit** (brief 5): abre el segundo hito. Consumirá `/query/stream` con cliente SSE
  (`sseclient-py`), pintando texto en vivo + citas del `event: done` al final.
