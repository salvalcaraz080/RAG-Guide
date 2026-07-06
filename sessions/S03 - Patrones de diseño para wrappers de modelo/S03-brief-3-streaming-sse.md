# S03 · Brief 3 — Streaming SSE (`POST /api/v1/query/stream`)

> Instrucciones para Claude Code. Este chat decide, CC implementa. Al cerrar: `S03-decisions.md`
> (append), derivar `CHANGELOG.md`, editar `CLAUDE.md`/`ARCHITECTURE.md`.
> **Brief 3 de 5.** Depende de los briefs 1 (wrapper) y 2 (cache) ya implementados. El brief 5
> (Streamlit) consumirá este endpoint. El brief 4 (observabilidad) logueará también el path de stream.

## Objetivo

Añadir un endpoint de streaming `POST /api/v1/query/stream` que emite la respuesta del modelo por
**SSE** (`text/event-stream`) token a token, para que el cliente vea el texto escribiéndose en vez de
esperar a la respuesta completa. Mejora de **UX de latencia percibida**, no de correctitud.

## Principio rector (no negociable): el stream es UX descartable encima de la fuente de verdad

El endpoint no-stream `POST /api/v1/query` es el **camino principal y la fuente de verdad**:
construcción de citas, grounding, validación y cache viven ahí (`answer_query`). `/query/stream` es
una capa de presentación **encima**, no un reemplazo. Consecuencias duras:
- **No** meter lógica de negocio que solo exista en el path de streaming. Cache, grounding y citas se
  deciden con la MISMA lógica que `/query`; el stream solo cambia **cómo se entrega** el resultado.
- Si el streaming diera problemas, se elimina sin tocar `/query`. Ese es el criterio de diseño:
  mantenerlo barato de quitar.
- `POST /api/v1/query` **no cambia** en este brief.

## Las dos capas de streaming (no confundir)

Hay dos tramos de red distintos. LiteLLM cubre uno; el otro es tuyo:

- **Tramo A — proveedor → backend.** `router.acompletion(stream=True)` devuelve un `async for` de
  chunks ya **normalizados** (LiteLLM unifica las APIs divergentes de Anthropic `text_stream` y
  OpenAI `delta.content`). Esto es lo que "LiteLLM soporta SSE" significa, y es real: no hay que
  normalizar las dos APIs a mano. **Este es el único tramo que LiteLLM resuelve.**
- **Tramo B — backend → cliente.** Emitir `text/event-stream` por HTTP desde `/query/stream` es
  código de **tu router FastAPI**, no de LiteLLM. LiteLLM entrega el iterador async (tramo A);
  convertirlo en eventos SSE que salen por el endpoint es tramo B.

El wrapper (`llm_wrapper.py`) gana una capacidad de streaming en el tramo A; el router gana el tramo B.

## Flujo del endpoint — el cache decide si hay stream

⚠️ El cache **no se mezcla** con el stream: el cache es una capa por encima del wrapper (brief 2) que
decide si se llama al modelo. Un hit no llama al modelo → no hay generación → no hay nada que
streamear. Por tanto el cache decide, antes de abrir ningún SSE, si este request siquiera entra en el
camino de streaming:

```
POST /api/v1/query/stream {"question": "..."}
1. Consulta cache (misma lógica que answer_query: build_cache_key + get).
2. HIT  → devuelve la respuesta entera, JSON de una vez (igual que /query). NO se abre SSE.
          Instantáneo, no hay generación que escribir token a token.
3. MISS → abre SSE (text/event-stream):
            a. Llama al wrapper en modo stream (tramo A: router.acompletion(stream=True)).
            b. Emite cada chunk de texto como evento SSE `data:` (tramo B).
            c. Al terminar la generación: construye citas (desde reference.py, misma lógica
               que answer_query) + usage + finish_reason.
            d. Emite un evento final tipado `event: done` con {citations, usage, model,
               provider, finish_reason, cache_hit: false}.
            e. Guarda en cache (set) la respuesta completa, igual que answer_query en el path
               no-stream — para que una repetición de esta pregunta sea un hit (y por tanto no
               se streamee).
```

Un hit sale por el mismo tipo de respuesta que `/query` (JSON completo, `cache_hit: true`). El SSE
solo existe en el camino miss.

## Qué transporta el stream (decidido)

- **Solo el texto** de la respuesta va token a token, en eventos SSE `data:`. Es lo único que
  mejora la latencia percibida.
- **Las citas y métricas NO se streamean**: viajan en un **único evento final tipado**
  `event: done` con el JSON de {citations, usage, model, provider, finish_reason}. Un solo
  round-trip; el cliente pinta el texto en vivo desde los `data:` y renderiza las citas al recibir
  `done`. Razón: la propiedad central es que la respuesta *lleve* la cita verificable, no *cuándo*
  aparece en pantalla; nadie audita una cláusula en tiempo real mientras se escribe. Streamear las
  citas como eventos tipados intermedios sería complejidad sin ganancia.

## Factorización de `llm_service` (recomendación — CC decide el diseño concreto)

Hoy `answer_query` devuelve la respuesta entera. El camino miss-con-streaming necesita el generador
de chunks del wrapper, no el texto ya ensamblado — pero SIN duplicar la lógica de cache/grounding/
citas. Recomendación de partición (CC ajusta):

- **Camino común de decisión** (compartido por `/query` y `/query/stream`): construir clave de cache,
  consultar cache. Hit → respuesta ensamblada. Este tramo es idéntico para ambos endpoints y no debe
  duplicarse.
- **Cola A — ensamblar entero** (lo que hoy hace `answer_query` en el miss): llama al wrapper
  (no-stream), arma la respuesta completa, cachea, devuelve. La usan `/query` siempre y `/query/stream`
  en el hit.
- **Cola B — emitir stream** (nueva, solo `/query/stream` en el miss): llama al wrapper en modo
  stream, va emitiendo chunks, y al cerrar arma citas+usage (reusando la MISMA construcción de citas
  que la cola A, no una copia) y cachea la respuesta completa.

El objetivo es que la construcción de citas y la clave de cache existan en **un** sitio, consumidos
por las dos colas. Si CC ve una factorización más limpia que respete "una sola lógica de decisión y
de citas, dos formas de entregar", adelante — el invariante es no duplicar grounding ni cache.

## `finish_reason` — fallo de integridad, no cosmético (dominio)

Si el stream termina con `finish_reason="length"` (truncamiento por límite de tokens), la respuesta
puede haberse cortado **antes de su cita** → afirmación normativa sin ancla de trazabilidad. En un
sistema de trazabilidad obligatoria esto es un **fallo de integridad**, no un aviso de UX:
- El evento `done` lleva `finish_reason`; si es `"length"`, marcarlo explícitamente (campo tipo
  `truncated: true` o equivalente) para que el cliente NO lo presente como respuesta normativa fiable.
- **No** aplicar la estrategia del material de "instruir al modelo a limitar la salida a N palabras":
  compite con la completitud normativa (el modelo podría omitir requisitos aplicables para caber).
  `MAX_OUTPUT_TOKENS` generoso + detección, no prevención por prompt.

## Tramo B — mecanismo SSE de FastAPI (verificar en el repo, no asumir)

El repo del proyecto del máster (estimador) ya resolvió el tramo B con un patrón que funcionó
(frontend consumía con `sseclient-py` un `text/event-stream` emitido por el endpoint). **Portar ese
patrón, no reinventarlo.** CC verifica en el repo/historial qué mecanismo exacto usó para emitir:
- Si fue `EventSourceResponse` de `sse-starlette` → reusar (`uv add sse-starlette` si no está).
- Si fue `StreamingResponse` con `media_type="text/event-stream"` y formateo manual de eventos →
  reusar.
- El material afirma soporte nativo `fastapi.sse`/`EventSourceResponse` desde FastAPI 0.135.0;
  verificar contra la versión instalada antes de fiarse. Prioridad: reusar lo que ya funcionó en el
  estimador sobre adoptar algo nuevo no verificado.

## Async

El generador de chunks es `async for` sobre `router.acompletion(stream=True)`. El endpoint emite de
forma async. No introducir iteración síncrona sobre el stream (mismo principio de los briefs 1 y 2:
no bloquear el event loop). El `set` de cache al cerrar el stream es async (`redis.asyncio`).

## Tests obligatorios (`tests/app/`, sin red — mock del wrapper en modo stream)

1. **Miss streamea texto + evento done.** Mockear el wrapper para que emita una secuencia de chunks;
   verificar que el endpoint emite los `data:` de texto en orden y cierra con `event: done` que lleva
   citations + usage + finish_reason.
2. **Hit no streamea.** Con la respuesta ya en cache, `/query/stream` devuelve JSON entero
   `cache_hit: true` y **no** abre SSE ni llama al wrapper (verificar con el mock que no se invocó el
   modo stream).
3. **Miss cachea la respuesta.** Tras un miss streameado, la entrada queda en cache: una segunda
   petición idéntica es hit (test 2).
4. **Citas coherentes con el path no-stream.** Para la misma pregunta, las citas del `event: done`
   coinciden con las que `/query` devuelve — misma lógica de construcción, no una copia divergente.
5. **Truncamiento marcado.** Si el stream cierra con `finish_reason="length"`, el evento `done` lo
   marca como truncado/no-fiable.
6. **Async.** El endpoint y el generador son async; no bloquean bajo `pytest-asyncio`.
7. **Regresión.** `POST /api/v1/query` (no-stream) sigue idéntico; su contrato no cambió.

Sin `except: pass`. Un error a mitad de stream (el proveedor corta, timeout) debe cerrar el SSE de
forma limpia con un evento de error tipado, no dejar la conexión colgada ni tragarse la excepción.

## Dependencias hacia adelante

- **Brief 4 (observabilidad):** el path de stream también se loguea (latencia hasta primer token,
  tokens, `finish_reason`, `cache_hit`). No implementar aquí; garantizar que los datos existen al
  cerrar el stream.
- **Brief 5 (Streamlit):** consumirá este endpoint con un cliente SSE (`sseclient-py` u
  equivalente), pintando los `data:` en vivo y renderizando las citas del `event: done` al final.

## Cierre (CC)

Implementación → `uv run pytest` → imagen Docker (`docker compose up --build`, app + redis, `/health`
responde) → test de usuario (consulta ECSS por `/query/stream`: ver el texto escribiéndose en vivo +
citas al final; repetir la misma pregunta: hit instantáneo sin stream) → docs (`S03-decisions.md`
etiquetado agnóstico/dominio, `CHANGELOG.md`, `CLAUDE.md`/`ARCHITECTURE.md`). El commit lo hace el
usuario.
