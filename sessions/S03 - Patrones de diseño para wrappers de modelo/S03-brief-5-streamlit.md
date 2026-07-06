# S03 · Brief 5 — Frontend conversacional (Streamlit)

> Instrucciones para Claude Code. Este chat decide, CC implementa. Al cerrar: `S03-decisions.md`
> (append), derivar `CHANGELOG.md`, editar `CLAUDE.md`/`ARCHITECTURE.md`.
> **Brief 5 de 5.** Segundo hito de S03 (frontend). Consume los endpoints de los briefs 1-4 ya
> implementados. No toca el backend salvo, como mucho, CORS si hiciera falta.

## Objetivo

Interfaz web conversacional en **Streamlit** para consultar RAG-ECSS sin curl/Swagger: el ingeniero
escribe una pregunta ECSS y ve la respuesta escribiéndose en vivo, con sus citas verificables y un
panel lateral de diagnóstico. Es la **cara de demo** del producto (carta de presentación ante PLD
Space), no el frontend definitivo.

## Principio rector (define toda la arquitectura del brief): Streamlit es un cliente HTTP fino

El frontend habla con el backend **solo por HTTP**, como cualquier cliente externo. NO importa
`llm_service`, ni `litellm`, ni el SDK de ningún proveedor, ni toca Redis. Consume `POST
/api/v1/query` y `POST /api/v1/query/stream`. Razones (decididas en la revisión teórica):
- **Prueba el contrato real del producto** (la API), no un atajo en proceso.
- **Migración trivial a un frontend maduro** (React/propio) cuando el producto lo requiera: será
  trabajo de frontend, no de backend, porque el acoplamiento es cero. Y es trivial *también* por el
  scope: el chat es single-turn (ver abajo), así que `st.session_state` guarda un log de
  presentación, no lógica de conversación que portar.
- **Desechable por diseño:** si Streamlit se sustituye, el backend no se entera.

**Enforcement del desacople a nivel de imagen** (patrón validado en el proyecto hermano LIDR): el
frontend vive en su propio proceso/contenedor con sus propias dependencias (`streamlit`, `requests`,
`sseclient-py`), y su `Dockerfile`/grupo de dependencias **excluye** `fastapi`/`litellm`/`openai`/
`anthropic`/`redis`. Que el frontend no *pueda* importar el backend aunque quisiera es la garantía
más fuerte del desacople — no una convención que se pueda erosionar. Estructura sugerida: carpeta
`frontend/` con `streamlit_app.py` + su gestión de dependencias propia (grupo `frontend` en
`pyproject.toml`, o un `pyproject`/entorno separado — CC decide según cómo esté montado el repo).

## "Conversacional" es metáfora de UI, no multi-turn (alcance)

El chat mantiene historial **visible** (`st.session_state`, varias preguntas seguidas en pantalla)
pero **NO pasa el historial al LLM**: cada pregunta es una llamada independiente a `/query` con el
mismo system prompt del backend. Mapea limpio sobre el endpoint single-turn actual sin tocar schema
ni prompt. El multi-turn real (pasar turnos previos como contexto, follow-ups) es **S05**, no este
brief. No implementarlo aquí.

## Configuración

- **`BACKEND_URL`** desde entorno (`.env`/`st.secrets`), no hardcodeado. En local apunta al backend
  (`http://localhost:8000`); en Docker Compose, al nombre de servicio del backend en la red interna.
- **NINGUNA API key en el frontend.** El frontend no llama a proveedores — llama al backend, que ya
  tiene las keys. Esto satisface el requisito del ejercicio ("la API key no está en el código") de la
  forma más fuerte posible: el frontend ni siquiera tiene una que exponer.

## Nivel 1 — Chat básico

- `st.chat_message` / `st.chat_input` para la conversación. Historial en `st.session_state`
  (renderizar el existente al recargar, añadir el nuevo turno). Streamlit re-ejecuta el script entero
  en cada interacción — de ahí que el estado deba vivir en `session_state`, no en variables locales
  (explicar esto en un comentario: es el modelo de ejecución de Streamlit, no obvio si no lo conoces).
- Al enviar una pregunta: `POST /api/v1/query` (no-stream para el Nivel 1), renderizar la respuesta
  como mensaje del asistente.
- **Renderizar las citas** (`standard`/`clause`) como algo más que texto plano: el producto ES la
  cita verificable. Un bloque/expander por cita, visualmente separado del cuerpo de la respuesta.
  (No hace falta enlazar a la fuente todavía — solo presentarlas legibles y distinguibles.)

## Nivel 2 — Streaming

Reemplazar la llamada no-stream del Nivel 1 por consumo de `POST /api/v1/query/stream`:
- **Cliente SSE:** `sseclient-py` (o equivalente) sobre `requests.post(..., stream=True)`. `st.write_stream`
  NO consume SSE-por-red directamente — espera un iterador de strings; el cliente SSE traduce el
  `text/event-stream` en ese iterador. (Este adaptador es trabajo real; el material del máster lo
  omite porque asume Streamlit acoplado al LLM en proceso. Patrón validado en LIDR con `sseclient-py`.)
- **Dos tipos de evento a manejar** (recordar el contrato del brief 3):
  - `event: chunk` / `data: {texto}` → alimentar el placeholder de texto en vivo (`st.write_stream`
    o placeholder + delta manual).
  - `event: done` → trae `{citations, usage, model, provider, finish_reason, truncated, cache_hit}`.
    Al recibirlo: renderizar las citas (Nivel 1) y rellenar el sidebar (Nivel 3).
  - `event: error` → mostrar un error legible, no romper la UI.
- **Un cache hit no streamea** (brief 3): el endpoint responde casi al instante con un único
  `event: done` (`cache_hit: true`) y sin chunks de texto previos. El cliente debe manejar ese caso —
  si no llegan `chunk` y llega directamente `done`, pintar la respuesta completa de una vez (la lleva
  el `done`, o se obtiene del campo correspondiente). Verificar cómo viaja el texto de la respuesta en
  el hit por el endpoint de stream y renderizarlo sin asumir que siempre hubo chunks.
- **`truncated: true`** en el `done` → marcar la respuesta visualmente como potencialmente incompleta
  / no fiable (fallo de integridad, brief 3), no presentarla como respuesta normativa válida sin más.

## Nivel 3 — Sidebar de diagnóstico (`st.sidebar`)

Panel con **toda la información útil para debug** (se recorta para la demo más adelante — decisión del
usuario; ahora, completo):
- **System prompt activo** (solo lectura). Obtenerlo del backend si hay endpoint que lo exponga, o
  mostrarlo como texto conocido; NO reconstruirlo duplicando lógica del backend. Si no hay forma
  limpia de obtenerlo sin acoplar, marcarlo como `# TODO: endpoint para exponer system prompt` y
  dejarlo pendiente en vez de duplicar.
- **Contexto CAG inyectado** — el fragmento ECSS que fundamenta la respuesta. Mismo criterio: si el
  backend lo expone (o viaja en la respuesta), mostrarlo; si no, TODO, no duplicar.
- **Métricas de la última llamada**, de la respuesta / `event: done`: `model`, `provider`,
  `tokens_in`, `tokens_out`, `cost_usd` (si el backend lo expone en la respuesta; hoy va al log, no
  necesariamente al JSON — ver nota), `latency` (cronometrada por el cliente), `ttft` si está,
  `finish_reason`/`truncated`, **`cache_hit`**, **`fallback_used`**.

**Nota sobre qué expone hoy la respuesta HTTP vs. el log:** el brief 4 puso parte de la traza
operacional (`cost_usd`, `fallback_used`, `ttft_ms`) en el **log estructurado**, no necesariamente en
el JSON de `QueryResponse`/`StreamDoneEvent`. Para que el sidebar los muestre, deben viajar en la
respuesta. CC verifica qué campos ya están en el contrato de respuesta y cuáles no:
- `model`, `provider`, `usage`, `cache_hit` → ya en `QueryResponse` (briefs 1-2).
- `finish_reason`, `truncated` → ya en `StreamDoneEvent` (brief 3).
- `cost_usd`, `fallback_used`, `ttft_ms` → hoy en el log. **Decisión para este brief:** exponerlos
  también en la respuesta HTTP (son baratos y ya están calculados) para que el sidebar los muestre sin
  que el frontend los recalcule ni lea logs. Esto es un cambio menor de contrato en el backend
  (añadir campos a la respuesta), permitido en este brief; documentarlo. Alternativa si se prefiere no
  tocar el contrato: sidebar muestra solo lo que ya viaja, y `cost_usd`/`fallback_used` quedan como
  TODO. CC elige y documenta; recomendado exponerlos (el sidebar de debug los quiere y el coste de
  exponerlos es trivial).

## Async / ejecución

Streamlit es síncrono por naturaleza (re-ejecuta el script). El consumo SSE con `sseclient-py` es
bloqueante dentro del run de Streamlit — correcto aquí, no rompe nada (no hay event loop de FastAPI
que proteger; es otro proceso). No introducir complejidad async en el frontend: no la necesita.

## Docker / Compose

- Servicio `frontend` en `docker-compose.yml`, con su imagen propia (dependencias del grupo frontend,
  SIN backend/litellm/redis). Publica su puerto al host (Streamlit, típico 8501) — este SÍ, porque es
  la cara que el usuario abre en el navegador.
- Depende del `api` para orden de arranque (`depends_on: [api]`), sin `service_healthy` (coherente con
  el resto: degradar, no bloquear — si el backend tarda, el frontend muestra error de conexión
  legible, no se cae).
- `BACKEND_URL` apunta al servicio `api` por la red interna de Compose.
- Verificar: `docker compose up --build` levanta **tres** servicios (api + redis + frontend); abrir el
  puerto de Streamlit en el navegador y consultar.

## Tests

Streamlit se testea sobre todo **manualmente** (UI). No forzar tests unitarios de la UI. Sí conviene:
- Un test ligero del **cliente HTTP/SSE** si se factoriza la lógica de consumo (parsear `event: done`,
  manejar el caso hit-sin-chunks, manejar `event: error`) en una función testeable separada del
  render de Streamlit — recomendado: aísla la lógica parseable del framework, y esa función sí se
  testea sin levantar Streamlit.
- El backend no gana tests nuevos por este brief salvo que se expongan campos nuevos en la respuesta
  (entonces, actualizar los tests de contrato de `QueryResponse`/`StreamDoneEvent`).

## Test de usuario (manual, imprescindible aquí)

`docker compose up --build` (api + redis + frontend) → abrir Streamlit en el navegador:
1. Pegar una pregunta ECSS → ver el texto escribiéndose en vivo → citas renderizadas al final.
2. Repetir la misma pregunta → hit instantáneo (`cache_hit: true` en el sidebar), sin efecto de
   escritura o casi.
3. Varias preguntas seguidas → historial persiste en pantalla.
4. Sidebar muestra las métricas de la última llamada.
5. (Si se puede forzar) una pregunta con el primario caído → `fallback_used: true` visible en el
   sidebar — la robustez del backend, visible en la demo.

## Cierre (CC)

Implementación → tests (los del backend siguen verdes; test del cliente SSE si se factorizó) → Docker
(tres servicios, consulta manual por navegador) → docs (`S03-decisions.md` etiquetado agnóstico/
dominio — nota: la elección Streamlit-vs-Chainlit-vs-custom y el patrón cliente-HTTP son **agnósticos**,
material del segundo consolidado; `CHANGELOG.md`, `CLAUDE.md`/`ARCHITECTURE.md`). El commit lo hace el
usuario.

## Para el consolidado (recordatorio para el chat, no para CC)

Este brief cierra el segundo hito. Tras su `S03-decisions.md`: extracción → `S03-consolidado.md`
parte 2 (frontend) → segunda pasada de Guide. Nodo de Guide ya perfilado en conversación: *elección de
framework de UI por madurez y forma de la app* — frameworks simples para MVP (Streamlit si necesitas
sidebar/métricas/presentación de citas; Chainlit si el diferenciador es inspeccionar razonamiento de
agente, reevaluable en M5), frontend custom (React/propio) para producto maduro si lo requiere; y
*frontend como cliente HTTP desacoplado* (enforced por imagen) como práctica que hace la migración
trivial.
