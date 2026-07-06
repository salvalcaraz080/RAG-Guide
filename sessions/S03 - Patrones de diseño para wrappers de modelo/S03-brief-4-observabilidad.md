# S03 · Brief 4 — Observabilidad operacional (structlog)

> Instrucciones para Claude Code. Este chat decide, CC implementa. Al cerrar: `S03-decisions.md`
> (append), derivar `CHANGELOG.md`, editar `CLAUDE.md`/`ARCHITECTURE.md`.
> **Brief 4 de 5.** Cierra el hito backend (wrapper + cache + streaming + observabilidad). Depende de
> los briefs 1–3. El brief 5 (Streamlit) mostrará algunas de estas métricas en su sidebar.

## Encuadre (leer antes de implementar)

Este brief **no "añade logging desde cero"**. Los briefs 1–3 ya emiten eventos de log dispersos
(`openai_fallback_disabled` en el 1; `cache_disabled_redis_unavailable`, `cache_get_failed`,
`cache_set_failed`, `cache_ping_failed` en el 2; el `event: error` del stream en el 3). El objetivo
real es **auditar lo que ya existe y unificarlo** en una capa de observabilidad coherente, y añadir la
traza operacional por llamada al LLM que hoy falta.

**Alcance: traza OPERACIONAL, no traza de cita.** "Observabilidad" aquí = qué modelo respondió,
tokens, coste, latencia, si hubo cache hit, si hubo fallback, `finish_reason`. NO es la trazabilidad
de cita (qué cláusula fundamentó qué afirmación) — eso es dominio de S11, fuera de este brief. No
mezclar.

## Paso 0 obligatorio — auditoría del estado actual de structlog

`structlog>=24.0` está en `pyproject.toml`, pero su uso es incierto. **Antes de configurar nada**, CC
audita y reporta en `S03-decisions.md`:
1. ¿Existe algún `configure_logging()` / `structlog.configure(...)` en el repo? (¿En `main.py`,
   `config.py`, algún módulo de logging?)
2. ¿Qué está logueando cada módulo hoy? Buscar `structlog.get_logger()`, `logging.getLogger()`,
   `logger.`, y `print(` en `app/`. Clasificar: structlog / logging estándar / print suelto.
3. ¿Los warnings de los briefs 1–3 (arriba) se emiten con structlog o con `logging`? ¿Con qué formato
   salen hoy?

Según lo que la auditoría encuentre, este brief:
- **Configura desde cero** si structlog está instalado pero sin configurar.
- **Impone config coherente** si hay `get_logger()` sueltos corriendo con defaults.
- **Se acopla a lo existente** si S02 ya dejó configuración — sin duplicarla.

El resultado de la auditoría condiciona el resto del brief; no asumir un escenario.

## Configuración de structlog

Config **dual** (patrón estándar, adaptado a lo que la auditoría revele):
- **Desarrollo** (`APP_ENV=development` / `LOG_LEVEL=DEBUG`): `ConsoleRenderer` — salida legible y
  coloreada en terminal.
- **Producción**: `JSONRenderer` — logs estructurados para ingestión (Elasticsearch/Loki/CloudWatch).

Procesadores compartidos: nivel de log, timestamp ISO-8601, y el renombrado de campos que convenga.
`wrapper_class` con filtrado por nivel desde `LOG_LEVEL` (ya en `Settings`).

**Contexto vinculado (`bind`).** Un `request_id` único por request entrante, vía **middleware** de
FastAPI, de modo que todos los logs de una misma request lo lleven sin repetirlo en cada llamada.
Esto es lo que permite reconstruir la traza completa de una consulta (HTTP → cache → LLM → respuesta)
filtrando por `request_id`.

Config vía `lifespan` (no `@app.on_event`, deprecado — ya usado en el brief 2 para el ping de cache).

## La traza operacional por llamada al LLM

El patrón es log-al-inicio / log-al-completar / log-en-error, sobre el punto donde se llama al modelo
(`llm_service` / el seam). Campos:

**Al iniciar** (evento `llm_call_started`): `request_id` (del bind), modelo solicitado (nombre
lógico), si la consulta se resolvió desde cache (`cache_hit` — si es hit, NO hay llamada al modelo y
esto se loguea distinto, ver abajo).

**Al completar** (`llm_call_completed`): `model_used` (quién realmente respondió), `provider`,
`tokens_in`, `tokens_out`, `cost_usd` (ver "Coste"), `latency_ms`, `finish_reason`, `truncated`
(brief 3), `fallback_used` (ver "Fallback observable"), `cache_hit: false`.

**En error** (`llm_call_failed`): tipo de error (timeout / rate-limit / auth / server), proveedor que
falló, si se intentará fallback, número de reintento. LiteLLM ya diferencia estos por su
`retry_policy` (brief 1); el log los hace visibles.

**Cache hit — caso especial.** Un hit no llama al modelo (brief 2: el cache corta por encima del
wrapper). Por tanto NO se emite `llm_call_started/completed` como si hubiera habido llamada. Se emite
un evento propio (`cache_hit`) con: `request_id`, `model`/`provider` de la respuesta **original**
cacheada, y el `usage` original. Punto clave para el coste (abajo): en un hit, el coste de esa
respuesta ya se pagó una vez — se registra como **coste evitado**, no como coste cero ni como coste
nuevo.

## Coste (`completion_cost` de LiteLLM)

Calcular `cost_usd` en runtime vía `litellm.completion_cost(completion_response=...)` (o equivalente
vigente), que usa el price map embebido de LiteLLM.

**Verificación obligatoria (los modelos de esta sesión son recientes):** confirmar que
`completion_cost` devuelve un valor **no nulo y plausible** para AMBOS modelos cableados
(`claude-haiku-4-5-20251001` y `gpt-5.4-mini-...`). Si el price map no tiene el precio o lo tiene a 0:
- Loguear `cost_usd: null` **explícito**, NO un `0` engañoso. Un coste desconocido no es coste cero
  (misma lógica que el `usage` del hit del brief 2).
- Registrar en `S03-decisions.md` qué modelos tienen precio y cuáles no, para saber si el coste es
  fiable o hay que derivarlo offline desde los tokens más adelante.

**Coste evitado en el hit.** Para el evento `cache_hit`, calcular el coste que *habría* tenido la
llamada (desde el `usage` original guardado) y loguearlo como `cost_avoided_usd`. Esto es lo que
convierte el cache de "cache_hit: true" en una métrica de ahorro real — la métrica que el artículo de
cache (brief 2) aparcó "para la capa de logging". Aquí aterriza.

## Fallback observable (alcance decidido: medio)

- **Sí:** loguear `fallback_used` (bool) en cada `llm_call_completed`, derivado de comparar el modelo
  solicitado con `model_used` (ya disponible desde el brief 1; la derivación de provider ya es robusta
  tras el brief 1-fix con `get_llm_provider`). Un fallback deja de ser silencioso: el log dice cuándo
  el primario cayó y respondió el respaldo.
- **Sí (barato, alto valor):** registrar `fallback_used` **en la misma traza que `cache_hit`**, de
  modo que un análisis posterior pueda cruzar ambos y responder "¿qué entradas se cachearon durante
  una ventana de fallback?" — haciendo **observable** la deuda de contaminación de cache por fallback
  (registrada en el brief 2) SIN resolverla en runtime.
- **No:** NO implementar el marcado/invalidación de entradas de cache servidas por el fallback (que
  requeriría `model_used` en la clave o en el valor + lógica de detección en el hit). Es peso real
  para un riesgo bajo en M1 (dos proveedores validados como equivalentes en citación, exact-match
  sobre fragmento estático). Queda como **nodo de deuda con señal disparadora**: si el volumen de
  fallbacks sube, o se detecta una regresión de calidad por servir respuestas del respaldo desde
  cache, ENTONCES toca meter `model_used` en la clave. No antes.

## Superficie de exposición del log (nodo agnóstico, anotar)

El structured logging captura prompts y, potencialmente, respuestas. Para ECSS (corpus público) no
hay problema de datos sensibles. Pero es un nodo a dejar anotado en el decisions como **agnóstico**:
en un corpus/queries con datos sensibles (p. ej. el proyecto de localización), el log es superficie de
exposición (GDPR, S06). Decisión para este brief: NO loguear el texto completo de prompts/respuestas
por defecto (loguear metadatos: tokens, modelo, latencia, no el contenido). Si se quiere el contenido
para debug, que sea a nivel DEBUG y consciente.

## Tests obligatorios (`tests/app/`, sin red)

1. **Traza de llamada completa.** Una consulta miss emite `llm_call_started` + `llm_call_completed`
   con los campos esperados (capturar logs con el capsys/caplog de structlog o un procesador de test).
2. **Traza de hit.** Una consulta hit emite el evento `cache_hit` con `cost_avoided_usd`, y NO emite
   `llm_call_completed` (no hubo llamada).
3. **`fallback_used` correcto.** Cuando el fallback rota (mock), `llm_call_completed` lleva
   `fallback_used: true` y el `model_used` del respaldo; sin fallback, `false`.
4. **Coste nulo explícito.** Si `completion_cost` devuelve 0/None para un modelo, se loguea
   `cost_usd: null`, no `0`.
5. **`request_id` propagado.** Todos los logs de una misma request llevan el mismo `request_id`
   (bind por middleware).
6. **Error logueado.** Un fallo del wrapper emite `llm_call_failed` con tipo de error y proveedor, sin
   `except: pass`.
7. **Regresión.** `/query` y `/query/stream` siguen respondiendo igual; el logging no altera el
   contrato ni rompe los paths existentes.

Los warnings ya existentes de los briefs 1–3 deben seguir emitiéndose, ahora por el logger unificado y
con formato coherente (verificar que la auditoría del paso 0 los reconcilió).

## Cierre (CC)

Auditoría (paso 0) → configuración → traza operacional → `uv run pytest` → imagen Docker
(`docker compose up --build`, app + redis, `/health`) → test de usuario (una consulta miss y una hit;
inspeccionar los logs: en dev, consola legible con la traza completa y el `cost_avoided_usd` en el
hit) → docs (`S03-decisions.md` con la auditoría del paso 0 + etiquetado agnóstico/dominio,
`CHANGELOG.md`, `CLAUDE.md`/`ARCHITECTURE.md`). El commit lo hace el usuario.

## Nota de cierre de hito

Con este brief cerrado, el hito backend de S03 está completo (wrapper + fallback + cache + streaming +
observabilidad). El siguiente paso del flujo es la **consolidación del hito** (`S03-consolidado.md`,
parte backend) y la primera pasada de `RAG-GUIDE.md` — trabajo de este chat, no de CC. El brief 5
(Streamlit) abre el segundo hito.
