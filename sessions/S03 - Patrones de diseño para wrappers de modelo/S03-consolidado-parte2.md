
---

# S03-consolidado.md — Parte 2: hito frontend (brief 5)

> Paso 5, segundo hito. Append histórico a la parte 1 (backend). Aprendizajes de **implementar** el
> frontend Streamlit. Cierra la sesión S03 completa. Fuente para la segunda pasada de `RAG-GUIDE.md`.

Estado al cierre: tres servicios Docker (`api` + `redis` + `frontend`), frontend Streamlit
consumiendo `/query` y `/query/stream` como cliente HTTP puro. 205 tests backend verdes. Sesión S03
completa en los seis pasos, ambos hitos.

## Bloque 5 — Frontend conversacional (Streamlit)

**Decidido (teoría) y confirmado (implementación).** Streamlit como **cliente HTTP fino**: sin
importar `llm_service`/`litellm`/SDKs/Redis, hablando con el backend solo por HTTP. Desacople
**enforced a nivel de imagen** (grupo de dependencias `frontend` sin `fastapi`/`litellm`/`openai`/
`anthropic`/`redis`; el `Dockerfile` instala solo ese grupo) — el frontend no *puede* importar el
backend aunque quisiera. Patrón portado del proyecto hermano LIDR. Chat single-turn (historial visible
en `st.session_state`, no enviado al LLM; multi-turn real es S05). Sin API keys en el frontend (no
llama a proveedores). Sin async en el frontend (Streamlit síncrono; `sseclient-py` bloquea dentro de
su run, correcto — no hay event loop que proteger).

**Corrección sobre lo asumido en teoría (qué heredó LIDR y qué no).** En la revisión teórica se asumió
que el proyecto hermano ya consumía SSE en ruta cliente-HTTP y que solo se portaba. La implementación
lo matiza: LIDR tenía la **dependencia** `sseclient-py` y un endpoint `/estimate/stream` "conservado,
no usado" — su UI real llamaba al endpoint **no-stream**. Así que el **patrón cliente-HTTP desacoplado
sí se heredó**, pero el **consumo SSE real es código nuevo** verificado desde cero contra el backend
(`SSEClient(response)` acepta directamente el `requests.Response` con `stream=True`, sin `response.raw`
ni iteración manual). Lección: distinguir "heredé el patrón" de "heredé el código que ejerce el
patrón" — la dependencia presente no implica el camino validado.

**El hallazgo más valioso de la sesión — bug del hit sin texto, escondido dos briefs:**
Un cache hit vía `/query/stream` no tenía **ninguna vía** para entregar el texto de la respuesta al
cliente. Cadena causal: (1) brief 3 diseñó `event: done` sin texto (en un miss el texto ya viaja por
`chunk`); (2) la corrección consultada del brief 3 hizo que un hit también respondiera por SSE con un
único `done` — pero ese `done` no llevaba texto, y un hit no manda ningún `chunk`; (3) `StreamDoneEvent`
no declaraba `answer`, así que Pydantic lo descartaba en silencio. Resultado: hit servido por streaming
= respuesta vacía en el cliente.
- **Por qué sobrevivió dos briefs:** los tests comprobaban `cache_hit: true` y ausencia de `chunk`
  (propiedades observables **de servidor**), pero ninguno comprobaba que el cliente pudiera **leer el
  texto** de un hit (propiedad observable **de cliente**). Solo apareció al construir el consumidor
  real.
- **Corrección:** `StreamDoneEvent.answer: str | None = None`; el hit lo puebla (única vía posible), el
  miss lo excluye a propósito (el texto ya viajó por `chunk`, no duplicar payload). Frontend maneja el
  caso: si no llegó ningún `chunk` antes de `done`, pinta `done.answer`. Tests actualizados para
  aseverar ambos lados.
- **Nota de flujo (honesta):** el bug nació de una cadena de dos correcciones de diseño en papel
  (mi diseño original hit-por-JSON era irrealizable con `fastapi.sse` → corrección hit-por-SSE →
  hueco del texto). Solo la implementación del consumidor real lo cerró. Es el argumento del proyecto,
  en vivo: la Guide se gradúa desde la implementación, no desde el diseño en papel.

**Decisiones de diseño:**
- `cost_usd`/`fallback_used`/`ttft_ms` **promovidos del log al contrato HTTP** (`QueryResponse`,
  `StreamDoneEvent`) para que el sidebar los muestre sin recalcular ni leer logs. `ttft_ms` es `None`
  donde no aplica (`/query`, y hit de `/query/stream`).
- **System prompt y contexto CAG del sidebar → `# TODO`, no duplicados.** El backend no los expone hoy;
  reconstruirlos en el frontend sería el acoplamiento-por-reimplementación que el principio rector
  prohíbe. Mejor pendiente que acoplado.

**Nodo abierto que deja el brief:** el backend no expone el system prompt ni el contexto activo. El
sidebar los quiere (visibilidad = parte de la trazabilidad que vende el producto). Candidato a endpoint
futuro — engancha con S04 (abstracción de prompts) / S05.

## Divergencia teoría-vs-implementación del hito frontend (para graduar)

- **Validar un contrato de streaming exige construir el consumidor real.** Las aserciones de servidor
  ("¿entró en el camino correcto?") no atrapan un fallo de entrega al cliente ("¿puede el cliente leer
  la respuesta en todos los caminos?"). El bug del hit es el caso testigo.
- **Dependencia heredada ≠ camino validado** (LIDR tenía `sseclient-py` pero no consumía SSE).

## Estado de la Guide tras S03

Sesión completa. Segundo hito añade a Phase 4 (o a una sección de frontend) el nodo de **elección de
framework de UI** y la práctica de **frontend como cliente HTTP desacoplado enforced por imagen**. El
resto de S03 ya se graduó en la primera pasada (hito backend).
