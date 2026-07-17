# S05 — Brief 1: Conversación multi-turn (historial de propiedad del servidor)

> Instrucciones de implementación para Claude Code. Una funcionalidad, testeable de forma
> independiente. CC implementa dentro del contrato de este brief; no toma decisiones de
> arquitectura — las que faltan van marcadas `# TODO: pendiente de decisión en Claude Chat`.

---

## 1. Objetivo

Dar soporte a conversaciones multi-turn en RAG-ECSS, donde una pregunta de seguimiento
(*"¿y para criticidad B?"*) es ininteligible sin los turnos anteriores. El estado de la
conversación (el historial) es **propiedad del servidor**, no del cliente.

Esto es **historial puro** (resolución de anáfora/elipsis), NO memoria de hechos destilados:
no hay extractor LLM, ni `ProjectMetadata`, ni políticas de olvido. RAG-ECSS no construye un
estado de dominio negociado turno a turno; el usuario pregunta y la respuesta ya existe en el
corpus.

---

## 2. Alcance

**SÍ entra:**
- Creación de sesión en el servidor (`POST /api/v1/sessions`).
- `session_id` opcional en `/query` y `/query/stream`; cuando se provee, se inyecta el
  historial de esa sesión en la llamada al modelo y se anexa el nuevo turno.
- Almacén de sesiones en memoria del proceso (dict), con la atadura de "un solo worker"
  documentada en el módulo.
- Compuerta de elegibilidad de cache por presencia de historial.
- Semántica de `session_id` provisto-pero-ausente → **error explícito**, nunca single-turn
  silencioso.
- Ventana deslizante simple sobre el historial (acotar crecimiento).
- Tests de contrato (familia 1, mockeados, sin red) de toda la lógica anterior.

**NO entra (diferido, no implementar):**
- Persistencia y federación de sesiones (Redis/PG). Es despliegue → S06/M6. Solo dict en memoria.
- TTL de sesión / job de archivado. Diferido a S06/M6.
- Extracción de memoria / hechos destilados. Descartado por diseño (ver §1).
- Condensación de la pregunta de seguimiento en pregunta autónoma. Es M2/S10; hoy el modelo
  resuelve la anáfora él solo con el array de mensajes (CAG, contexto fijo).
- Perilla de disciplina/corpus en foco. Es M2 (necesita retrieval que filtrar).
- Cualquier cambio en la clave de cache para meter el historial. **El historial NO es slot de
  clave** (ver §5).

---

## 3. Almacén de sesiones — `app/services/sessions.py` (nuevo)

Un `SessionStore` propio sobre un `dict` en memoria, construido una sola vez a nivel de módulo
(mismo patrón que el Router de LiteLLM y `LLMCache`: singleton de módulo, no por request).

**Tipos:**
- `Message` (Pydantic): `role: Literal["user", "assistant"]`, `content: str`. Es el turno
  mínimo que se reinyecta al modelo — solo rol y texto, sin citas ni metadata (esas serían ruido
  en el contexto del modelo).
- `Session` (Pydantic o dataclass): `session_id: str`, `history: list[Message]`,
  `created_at: datetime`. `history` es la lista cronológica de turnos (user, assistant, user,
  assistant, …).

**API del store:**
- `create() -> str`: crea una sesión vacía, devuelve su `session_id` (uuid4 como string).
- `get(session_id: str) -> Session | None`: devuelve la sesión o `None` si no existe.
- `append_turn(session_id: str, user_msg: Message, assistant_msg: Message) -> None`: anexa el
  par (user, assistant) a la sesión y aplica la ventana deslizante (§6).

**Atadura de un solo worker (documentar en el docstring del módulo, no asumir):** el dict vive
en la memoria del proceso. Con varios workers (`uvicorn --workers N`), el turno 1 crea la sesión
en el worker A y el turno 2 puede aterrizar en el B, que no la tiene → sesión-no-encontrada en
operación normal. Correcto solo con **un worker**. Escalar workers es exactamente el momento en
que se necesita un store compartido (S06/M6). Esto NO es un defecto a arreglar aquí; es una
condición a escribir.

**Excepción tipada:** define `SessionNotFoundError` (en este módulo o en un módulo de errores del
servicio) para el caso provisto-pero-ausente. NO se traduce a single-turn.

---

## 4. Ciclo de vida de la sesión — endpoints

**`POST /api/v1/sessions`** (router nuevo `app/routers/sessions.py`, thin: recibe, delega,
devuelve):
- Sin body (o body vacío). Delega en `session_store.create()`.
- Respuesta: `SessionResponse{session_id: str}` (schema nuevo en `app/schemas/query.py` o un
  `app/schemas/session.py` nuevo — a criterio de CC, coherente con la estructura actual).

No hay endpoint de reset explícito en este brief: reiniciar una conversación = crear una sesión
nueva y usar el nuevo `session_id`. (`DELETE /sessions/{id}` queda fuera de alcance; si se quiere,
`# TODO` para una sesión futura.)

---

## 5. Cambios en el flujo de `llm_service.py`

`session_id` es **opcional** en `answer_query` y `stream_answer_query`. La lógica de sesión vive
en el service (compartida por `/query` y `/query/stream`), no en los routers.

**Contrato de request:** `QueryRequest` gana `session_id: str | None = None`. Los routers pasan
`session_id` al service. `include_context` sigue igual.

**Flujo (pseudocódigo, no es la implementación — CC escribe el código real):**

```
answer_query(question, *, session_id=None, include_context=False):

    # 1) Moderación SIEMPRE, sobre la pregunta ACTUAL (invariante S04 intacto).
    #    El historial NO se re-modera: cada turno pasó por guardrails cuando fue la
    #    pregunta actual, y los turnos assistant los generó el servidor (confiables).
    run_input_guards(question)

    # 2) Resolver historial desde la sesión (si la hay).
    history = []
    session = None
    if session_id is not None:
        session = session_store.get(session_id)
        if session is None:
            raise SessionNotFoundError(session_id)   # → NUNCA single-turn silencioso
        history = session.history

    # 3) Compuerta de elegibilidad de cache: elegible SOLO si NO se inyecta historial.
    #    Turno 1 de una sesión (history vacío) se comporta como single-turn: elegible.
    #    Turnos 2+ (history no vacío) → bypass de get Y set.
    cache_eligible = (len(history) == 0)

    # 4) Camino single-turn: intento de cache (comportamiento M1 intacto).
    if cache_eligible:
        hit = _cache_lookup(question, ...)     # get
        if hit is not None:
            # Si hay sesión (turno 1 con hit), el turno IGUAL se anexa al historial,
            # usando el TEXTO de la respuesta cacheada como turno assistant.
            if session is not None:
                session_store.append_turn(
                    session_id,
                    Message(role="user", content=question),
                    Message(role="assistant", content=hit.answer),
                )
            return hit

    # 5) Miss o multi-turn: llamada real al modelo, con historial.
    context_block = _build_context_block(ECSS_REFERENCE)
    system_prompt = _compose_system_prompt(context_block)
    result = _call_model(system_prompt, question, history=history)   # ver §7
    response = _assemble_response(result, context_block, include_context, ...)

    # 6) Solo el camino single-turn escribe en cache.
    if cache_eligible:
        _cache_set(...)

    # 7) Si hay sesión, anexar el turno (user actual + assistant generado).
    if session is not None:
        session_store.append_turn(
            session_id,
            Message(role="user", content=question),
            Message(role="assistant", content=result.text),
        )

    return response
```

**Puntos que NO deben perderse:**
- **`cache_eligible = (len(history) == 0)`**, no `session_id is None`. El turno 1 de una sesión
  (historial vacío) ES elegible y comparte entrada de cache con la misma pregunta hecha en
  single-turn (misma pregunta + mismo contexto + misma corpus_version → misma clave → misma
  respuesta). Esto preserva el valor FAQ. Solo los seguimientos (prefijo único) hacen bypass.
- **El turno se anexa incluso en un cache HIT de turno 1**, usando `hit.answer` como texto
  assistant — si no, el turno 2 no tendría contexto del turno 1.
- **La clave de cache NO cambia**: sigue siendo `question + system_prompt + context +
  corpus_version`. El historial no entra en la clave; es una compuerta, no un slot.
- El texto assistant que se guarda es el que el modelo dijo (incluida cualquier frase-marcador de
  rechazo como `Out of scope:`). Es el historial honesto.
- La detección de rechazo y la construcción de citas (`_refusal_kind`, `_citations`) operan sobre
  la respuesta ACTUAL, sin cambios. El historial es solo contexto previo para el modelo.

**`stream_answer_query`:** misma lógica de sesión e historial. Consecuencia: una petición
multi-turn hace bypass de cache → es siempre un "miss" → siempre streamea de verdad. El turno 1
de una sesión puede aún ser un hit y emitir el único `event: done` (con `cache_hit: true`). El
turno se anexa igual que en `answer_query` (en el hit y en el miss).

---

## 6. Ventana deslizante (acotar crecimiento del historial)

`append_turn` mantiene solo los últimos **N turnos** (pares user+assistant); descarta los más
antiguos. Config: `MAX_HISTORY_TURNS: int` en `Settings` (ver `.env.example`).

**Sobre el valor por defecto:** es un tope de seguridad, no un número investigado. Ponlo
explícito (p. ej. `10`) y coméntalo como knob a calibrar, NO como constante con autoridad — no
inventamos números con cara de verdad. La estrategia es la más simple (ventana deslizante) a
propósito: en RAG-ECSS no hay memoria de hechos que perder al truncar, y la anáfora suele
referirse al turno inmediatamente anterior.

No hay resumen acumulativo ni híbrida-con-anclas en este brief (son estrategias de S02 para
cuando importe; hoy no).

---

## 7. Historial en el seam del modelo — `llm_wrapper.py`

`call_llm` y `stream_llm` ganan un parámetro **opcional** `history: list[Message] | None = None`
(o `list[dict]`, a criterio de CC — mantener el tipo coherente con `Message` de `sessions.py`,
sin acoplar el wrapper a la lógica de sesión). Composición del array de mensajes:

```
messages = [{"role": "system", "content": system_prompt}]
         + [ {"role": m.role, "content": m.content} for m in (history or []) ]
         + [{"role": "user", "content": user_message}]
```

Cambio **retrocompatible**: `history=None` → `[]` → comportamiento single-turn idéntico al
actual. El wrapper sigue sin saber qué es una sesión; solo compone y normaliza (su único trabajo).
No mover la responsabilidad de resolver la sesión al wrapper — eso vive en `llm_service`.

Alternativa considerada y descartada (para que CC no la reabra): pasar el array de mensajes
completo ya montado desde el service. Se descarta por ser un refactor mayor que toca ambos
caminos (stream y no-stream) sin beneficio sobre el parámetro opcional; el wrapper ya recibe
`system_prompt` + `user_message` por separado y basta con intercalar el historial.

---

## 8. Mapeo de errores en routers

- `/query` con `session_id` provisto-pero-ausente → `SessionNotFoundError` → **HTTP 404** con
  mensaje legible ("session not found or expired; create a new session"). NO 200 single-turn.
- `/query/stream` con `session_id` provisto-pero-ausente → **`event: error`** tipado (reusar el
  mismo evento SSE que ya existe para toxicidad/fallo de proveedor a mitad de stream), sin ningún
  `chunk` previo.
- Sin `session_id` → comportamiento M1 intacto (single-turn, cache-elegible).

---

## 9. Tests de contrato a escribir (`tests/app/`, familia 1, mockeados, sin red)

Todos con la respuesta del LLM y la moderación mockeadas (fixtures `autouse` de `conftest.py`
que aíslan Redis y Moderation siguen aplicando; verificar que se mantienen si se toca ese fichero).

- `session_id` ausente → single-turn, cache-elegible, comportamiento M1 intacto (regresión).
- `session_id` provisto-pero-ausente → `SessionNotFoundError` → 404 en `/query`; `event: error`
  en `/query/stream`. **Nunca** single-turn.
- Multi-turn (historial no vacío) → **bypass de cache**: ni `get` ni `set` se ejecutan
  (mock del cache asertando 0 llamadas, o el equivalente).
- Turno 1 con sesión (historial vacío) → cache-elegible; el turno se anexa al historial **también
  en un cache HIT**, con `hit.answer` como texto assistant.
- El array de mensajes que llega al seam lleva los turnos en orden correcto
  (`system, u1, a1, …, u_actual`). Testear contra `call_llm`/`stream_llm` mockeado, inspeccionando
  el `messages` compuesto.
- Moderación corre sobre la pregunta ACTUAL únicamente (no sobre el historial).
- Ventana deslizante: tras `MAX_HISTORY_TURNS` superados, solo sobreviven los últimos N pares.

Recordatorio del contrato de CC: el único paso de verificación es `uv run pytest` (mockeado, sin
red, sin Redis). La evaluación de si el modelo *resuelve bien la anáfora* multi-turn es cualitativa
y va al guion de UAT (§10), no a `pytest`.

---

## 10. UAT propuesto (CC entrega el guion; lo ejecuta el usuario)

Guion para que el usuario lo corra en la UI de Streamlit contra el backend real:

1. Crear sesión (`POST /api/v1/sessions`), obtener `session_id`.
2. Turno 1: pregunta que toque el fragmento ECSS de M1 → respuesta con cita. Verificar
   `cache_hit` según corresponda.
3. Turno 2 en la misma sesión: pregunta de seguimiento con anáfora (*"¿y para el caso X?"*) →
   verificar que el modelo la interpreta con el contexto del turno 1 (no pide aclaración, no
   responde out-of-material por no entender la referencia).
4. Repetir la pregunta del turno 1 en una sesión NUEVA → verificar que es cache-elegible (mismo
   comportamiento que single-turn).
5. Enviar un `session_id` inventado → verificar 404 legible (no una respuesta single-turn).
6. (Opcional) Enviar una pregunta tóxica como turno de seguimiento → verificar que la moderación
   la bloquea igual (invariante S04 sobre la pregunta actual).

**Nota importante para el usuario:** el frontend Streamlit hoy tiene el historial en
`st.session_state` de forma **presentacional** (S03, no se reenvía al modelo). Este brief mueve
la propiedad del historial al servidor. Adaptar el frontend para (a) crear la sesión y (b) mandar
el `session_id` en cada `/query` es parte del cableado de la demo — decidir en el propio brief si
CC toca `frontend/` o si queda como paso manual del usuario. **`# TODO: pendiente de decisión en
Claude Chat`** si CC ve que el alcance del frontend no está claro: por defecto, este brief es
backend; el frontend se aborda por separado para mantener los briefs granulares.

---

## 11. Convenciones (recordatorio)

- Type hints y docstrings (Google, inglés) en todo lo nuevo.
- Comentarios inline en español, explicando QUÉ y POR QUÉ (este proyecto es material de
  aprendizaje): explicar la primera aparición de mecanismos (uuid4, ventana deslizante, singleton
  de módulo, etc.).
- Nada de `print()`; `structlog`. Nada de `except: pass`.
- Pydantic/dataclasses, nunca dicts crudos para estructuras.
- 100 chars por línea, nombres en inglés.
- Al cerrar: `SNN-decisions.md` con el log de decisiones (etiquetadas agnóstico/dominio), derivar
  `CHANGELOG.md`, y actualizar `CLAUDE.md`/`ARCHITECTURE.md` reescribiendo el ESTADO PRESENTE (no
  diario). Entregar el guion de UAT (§10).
