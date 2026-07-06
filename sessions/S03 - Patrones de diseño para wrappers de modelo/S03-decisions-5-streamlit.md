# S03-decisions-5-streamlit.md — Implementación (Claude Code), brief 5/5

Decisiones y fricciones surgidas al implementar `S03-brief-5-streamlit.md` sobre el repo
`RAG-ECSS`. Etiquetadas agnóstico (candidata a Guide) vs. dominio (solo ECSS). Cierra el
segundo hito de S03 (frontend) y la sesión completa.

## Verificación hecha antes de implementar

1. **[agnóstico] Patrón de separación por imagen — validado en LIDR, reutilizado tal
   cual.** El proyecto hermano (`C:\Users\salca\proyectos\LIDR`) ya resolvía esto:
   `pyproject.toml` con un grupo de dependencias `frontend` (`streamlit`, `requests`,
   `sseclient-py`) separado del grupo principal, y `frontend/Dockerfile` con
   `uv sync --frozen --no-install-project --no-default-groups --group frontend` —
   instala SOLO ese grupo, sin `fastapi`/`litellm`/`openai`/`anthropic`/`redis`. Portado
   sin cambios de fondo a `RAG-ECSS`.
2. **[dominio] El consumo de SSE en sí NO estaba validado en LIDR, solo la dependencia.**
   Su `app/routers/estimations.py` sí expone un endpoint `/estimate/stream` con
   `fastapi.sse`, pero está marcado "CONSERVADO, no lo usa la UI" — el
   `frontend/streamlit_app.py` real de LIDR llama solo al endpoint no-stream. La lógica de
   parseo SSE de `sseclient-py` en el frontend (`frontend/sse_client.py` de este repo) es
   código nuevo, no un patrón portado — verificado desde cero contra el backend real
   (`SSEClient(response)` aceptando directamente el objeto `requests.Response` con
   `stream=True`, sin necesitar `response.raw` ni iterar a mano).
3. **[agnóstico] `sseclient-py` acepta el `requests.Response` directamente** como
   `event_source` (itera sobre él vía `__iter__`/`iter_content` por defecto). No hizo
   falta configuración adicional.

## Registradas por el brief

- **[agnóstico] Frontend como cliente HTTP fino**, sin ningún import de `llm_service`,
  `litellm`, SDKs de proveedor, ni Redis. Enforced a nivel de imagen (ver verificación 1),
  no solo por convención.
- **[dominio] "Conversacional" es metáfora de UI, no multi-turn**: `st.session_state`
  guarda el historial visible; cada pregunta es una llamada independiente a
  `/query/stream` con el mismo system prompt fijo del backend. Multi-turn real es S05.
- **[agnóstico] `BACKEND_URL` por entorno, sin API keys en el frontend** — el frontend no
  llama a ningún proveedor, así que no tiene ninguna key que exponer (la forma más fuerte
  de cumplir "la API key no está en el código").
- **[dominio] Nivel 3 (sidebar): system prompt y contexto CAG marcados `# TODO`**, no
  reconstruidos — el backend no expone hoy un endpoint para verlos, y duplicar esa lógica
  en el frontend habría sido exactamente lo que el principio rector prohíbe (acoplamiento
  por reimplementación, no por HTTP).
- **[dominio] `cost_usd`/`fallback_used`/`ttft_ms` expuestos en el contrato HTTP**
  (`QueryResponse`, `StreamDoneEvent`), siguiendo la recomendación del brief: antes solo
  iban al log estructurado (brief 4); el coste de exponerlos es trivial y el sidebar los
  necesita sin recalcular ni leer logs. `ttft_ms` es `None` para `/query` (no aplica) y
  para un hit de `/query/stream` (no hubo streaming que medir).
- **[agnóstico] Streamlit síncrono, sin async en el frontend** — `sseclient-py` bloquea
  dentro del `run` de Streamlit, correcto aquí: no hay event loop de FastAPI que proteger,
  es otro proceso.
- **[dominio] Docker: tres servicios.** `frontend` publica el puerto 8501 al host (es la
  cara que abre el usuario — a diferencia de `redis`, que no publica ninguno).
  `depends_on: [api]` sin `condition: service_healthy`, mismo principio que `api`→`redis`
  del brief 2: degradar (mostrar error de conexión legible), no bloquear el arranque.

## Fricción de implementación no prevista por el brief (la más importante de esta sesión)

**[agnóstico] Bug real encontrado al construir el consumidor SSE real: un cache HIT vía
`/query/stream` no tenía ninguna forma de entregar el texto de la respuesta al cliente.**

Reconstrucción: el brief 3 diseñó `event: done` para NO llevar texto en un MISS (ya viajó
por `chunk`). Al adaptar el diseño (brief 3, corrección consultada) para que un HIT
también respondiera por SSE con un único `done`, nadie verificó que ese `done` necesitaba
llevar el texto completo — porque un HIT nunca manda ningún `chunk`. `StreamDoneEvent` no
declaraba el campo `answer`, así que Pydantic lo descartaba en silencio tanto en el HIT
(donde el dict interno SÍ traía `"answer"`, vía `_assemble_response`) como en el MISS
(irrelevante ahí, ya se había mandado por chunks). El bug estuvo dos briefs sin
detectarse porque ningún test comprobaba explícitamente que el cliente pudiera *leer* el
texto de un hit — solo se comprobaba `cache_hit: true` y ausencia de eventos `chunk`.

**Corrección aplicada:**
- `StreamDoneEvent.answer: str | None = None` (nuevo campo).
- `stream_answer_query` (HIT): el evento `done` ahora incluye `answer` con el texto
  cacheado completo — es la ÚNICA vía posible, al no haber `chunk` previos.
- `stream_answer_query` (MISS): el evento `done` EXCLUYE `answer` a propósito (se
  construye filtrando esa clave de `**response`) — el texto ya viajó entero por `chunk`,
  repetirlo en `done` solo duplicaría payload sin necesidad.
- Tests actualizados en `tests/app/test_streaming.py`: `test_miss_streams_chunks_then_done`
  ahora comprueba `done.get("answer") is None`; `test_hit_does_not_stream` comprueba
  `events[0]["data"]["answer"] == "irrelevant"` (el texto cacheado en el test).
- `frontend/streamlit_app.py` maneja explícitamente el caso: si no llegó ningún `chunk`
  antes de `done`, pinta `done_payload["answer"]` — sin este fix, el placeholder se
  habría quedado vacío en todo hit servido por streaming.

Esta corrección se documenta aquí (brief 5) en vez de retroactivamente en brief 3, porque
solo se hizo evidente al construir el consumidor real — es exactamente el tipo de fallo
que un test de "¿el cliente puede leer la respuesta?" habría atrapado antes, y que un test
de "¿cache_hit es true?" no atrapa.

## Verificación en Docker (test de usuario del brief)

`docker compose up --build -d` → tres contenedores (`rag-ecss-api`, `rag-ecss-redis`,
`rag-ecss-frontend`), los tres `Up`/healthy donde aplica. `http://localhost:8501` responde
200; `http://localhost:8000/health` → `{"status": "healthy", "cache": "up"}`.

Probado manualmente por el usuario en el navegador:
- Pregunta ECSS → texto en vivo (streaming real) → cita renderizada en expander al
  terminar.
- Sidebar con métricas de la última llamada (modelo, proveedor, tokens, coste, latencia
  cliente, ttft, finish_reason, truncada, cache hit, fallback).
- Repetir la misma pregunta → hit instantáneo, `cache_hit: ✅ sí` en el sidebar, **y texto
  completo visible** (el fix de arriba, confirmado en uso real, no solo en test).
- Varias preguntas seguidas → historial persiste en pantalla.

## Tests

`uv run pytest` — 205 pasan (sin cambio en el número de tests del backend salvo las dos
aserciones nuevas en `test_streaming.py` para el fix de `answer`, tal como preveía el
brief: "el backend no gana tests nuevos salvo que se expongan campos nuevos — entonces,
actualizar los tests de contrato"). Ruff limpio sobre `app/`, `tests/app/` y `frontend/`.

Sin tests unitarios de la UI de Streamlit (decisión explícita del brief: se testea
manualmente). `frontend/sse_client.py` quedó factorizado sin ningún import de Streamlit,
disponible para testear como Python plano si se decidiera más adelante — no se añadieron
tests automáticos para él en esta sesión (verificado manualmente contra el backend real,
ver sección de verificación arriba), a petición del usuario.

## Nota para el consolidado (material agnóstico ya perfilado, no acción de CC)

- Elección de framework de UI por madurez y forma de la app: Streamlit para MVP con
  sidebar/métricas/presentación de citas; Chainlit si el diferenciador fuese inspeccionar
  razonamiento de agente; frontend custom (React/propio) para producto maduro.
- Frontend como cliente HTTP desacoplado, enforced por imagen Docker (no por convención),
  como práctica que hace la migración a un frontend maduro trivial.

## No implementado (fuera de alcance de este brief) — nada que reportar

Endpoint para exponer el system prompt/contexto CAG activo (sidebar Nivel 3, marcado
`# TODO` en vez de duplicar), multi-turn real (S05), tests automáticos del cliente SSE
del frontend (decisión explícita: pruebas manuales para esta sesión).
