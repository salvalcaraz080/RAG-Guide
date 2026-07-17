# S05 — Decisions log, brief 1: conversación multi-turn (historial de propiedad del servidor)

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier proyecto
> LLM) / `[dominio]` (específico de RAG-ECSS).

---

## 1. Historial PURO, no memoria de hechos destilados `[dominio]`

Decisión de alcance central del brief, no un recorte de implementación. RAG-ECSS soporta
resolución de anáfora/elipsis ("¿y para criticidad B?" solo tiene sentido con el turno
anterior), reinyectando los turnos previos tal cual se dijeron. NO hay extractor LLM, ni
`ProjectMetadata`, ni políticas de olvido. Motivo de dominio: el usuario pregunta y la
respuesta ya existe en el corpus — no se negocia un estado de dominio turno a turno que
haya que destilar y recordar. Un extractor de memoria sería complejidad especulativa sobre
un problema que este producto no tiene.

## 2. El historial es propiedad del SERVIDOR, no del cliente `[agnóstico]`

El estado de la conversación vive en el servidor (`SessionStore`), no viaja en cada request
desde el cliente. El cliente solo lleva un `session_id` opaco. Motivo: (a) el cliente no
puede manipular ni corromper el historial que se reinyecta al modelo; (b) los turnos
`assistant` los generó el servidor y son de confianza (relevante para el §5, la
no-remoderación); (c) desacopla el tamaño del payload del cliente del tamaño del historial.

Contraste con el frontend actual: Streamlit tenía el historial en `st.session_state` de
forma **presentacional** (S03, no se reenviaba al modelo). Este brief mueve la PROPIEDAD del
historial al servidor. Adaptar el frontend es cableado de demo, se aborda por separado
(este brief es backend-only) para mantener los briefs granulares.

## 3. Compuerta de cache por presencia de historial, no por presencia de sesión `[agnóstico]`

La decisión más sutil del brief. `cache_eligible = len(history) == 0`, **no**
`session_id is None`. Consecuencia buscada:

- **Turno 1 de una sesión** (historial vacío) ES cache-elegible y comparte entrada de cache
  con la misma pregunta hecha en single-turn (misma pregunta + mismo contexto + misma
  `corpus_version` → misma clave → misma respuesta). Preserva el valor FAQ: la primera
  pregunta de una conversación es tan cacheable como cualquier consulta suelta.
- **Turnos 2+** (historial no vacío) hacen **bypass de get Y set**. El prefijo del prompt
  (system + historial + pregunta) es único por conversación, así que cachearlo casi nunca
  daría hit y ensuciaría Redis con entradas de un solo uso.

El historial **NO es un slot de la clave de cache**. `build_cache_key` no cambia. Es una
compuerta (entra/no entra al cache), no una dimensión más de la clave. Meterlo como slot
habría hecho crecer el espacio de claves sin beneficio (cada conversación tiene su propio
prefijo → colisión cero de todas formas).

## 4. El turno se anexa incluso en un cache HIT de turno 1 `[agnóstico]`

Si el turno 1 acierta cache, la respuesta viene de Redis sin llamar al modelo — pero el
turno IGUAL se anexa al historial, usando el TEXTO de la respuesta cacheada (`hit.answer`)
como turno `assistant`. Si no, el turno 2 no tendría contexto del turno 1 y la anáfora se
rompería justo en el caso más común (primera pregunta cacheada → seguimiento). Es el detalle
que hace que la compuerta de cache y el historial coexistan sin agujeros.

## 5. La moderación corre SIEMPRE sobre la pregunta actual; el historial NO se re-modera `[agnóstico]`

Invariante de S04 intacto: `run_input_guards(question)` corre sobre la pregunta ACTUAL, antes
de resolver la sesión y antes del cache. El historial no se vuelve a moderar porque: (a) cada
turno `user` ya pasó guardrails cuando fue la pregunta actual; (b) los turnos `assistant` los
generó el servidor (confiables, §2). Re-moderar el historial entero en cada turno sería
gastar N llamadas de red para re-verificar lo ya verificado. Test que lo fija:
`test_moderation_runs_on_current_question_only` (espía `run_input_guards`, aserta una sola
llamada con la pregunta actual).

## 6. `session_id` provisto-pero-ausente → error explícito, nunca single-turn silencioso `[agnóstico]`

Un `session_id` que no existe (inventado, o expirado en un futuro con TTL) es un **error**,
no una señal de "trátalo como turno suelto". `SessionNotFoundError` tipada → **404** en
`/query`, **`event: error`** tipado en `/query/stream` (reusa el mismo evento SSE que
toxicidad/fallo de proveedor). Motivo: el cliente que mandó un `session_id` espera su
contexto de conversación; responder como single-turn perdería ese contexto **en silencio** —
el modelo respondería una pregunta de seguimiento como si fuera la primera, produciendo una
respuesta plausible pero desalineada, sin ninguna señal de que algo falló. Fallar ruidoso es
correcto aquí.

## 7. Ventana deslizante simple, con valor por defecto HONESTO `[agnóstico]`

`MAX_HISTORY_TURNS = 10`, expuesto como `Settings`/`.env`. Comentado explícitamente como
**tope de seguridad, knob a calibrar, NO número investigado** — no se inventan constantes con
cara de verdad. La estrategia es la más simple a propósito (ventana deslizante que descarta
los pares más antiguos): en RAG-ECSS no hay memoria de hechos que perder al truncar (§1), y
la anáfora suele referirse al turno inmediatamente anterior. Se descartan explícitamente para
este brief el resumen acumulativo y la híbrida-con-anclas (estrategias de S02 para cuando
importe; hoy no aporta).

Implementación: `history[:] = history[-2N:]` (reasignación EN SITIO del contenido de la
lista, no un atributo nuevo) — evita disparar validación de asignación de Pydantic y mantiene
la referencia estable.

## 8. Historial en el seam del wrapper vía parámetro opcional, no array pre-montado `[agnóstico]`

`call_llm`/`stream_llm` ganan `history: list[Message] | None = None`. El wrapper compone
`[system, *history, current_user]` (helper `_compose_messages`) y sigue sin saber qué es una
sesión — su único trabajo es componer y normalizar. `history=None` → `[]` → shape single-turn
byte-idéntico al de antes de S05 (retrocompatible).

**Alternativa descartada** (para que no se reabra): pasar el array de mensajes completo ya
montado desde el service. Es un refactor mayor que toca ambos caminos (stream y no-stream)
sin beneficio sobre el parámetro opcional — el wrapper ya recibe `system_prompt` +
`user_message` por separado y basta con intercalar el historial en medio. La responsabilidad
de RESOLVER la sesión se queda en `llm_service`, no se mueve al wrapper.

Nota de acoplamiento: el wrapper importa el tipo `Message` de `sessions.py`. Es una
importación de un tipo de DATOS Pydantic (no de la lógica del store), sin ciclo de import
(`sessions` solo importa `config`). Coherente con "mantener el tipo de `history` alineado con
`Message`" sin duplicar la definición.

## 9. Almacén en memoria del proceso: atadura de un solo worker, escrita como condición `[dominio]`

El `dict` de sesiones vive en la memoria del proceso. Con `uvicorn --workers N`, el turno 1
puede crear la sesión en el worker A y el turno 2 aterrizar en el B → `SessionNotFoundError`
en operación normal. **Correcto solo con un worker.** Esto NO es un defecto a arreglar en
este brief; es una condición documentada en el docstring del módulo. Escalar workers es
exactamente el momento en que se necesita un store COMPARTIDO (Redis/PG) — diferido a S06/M6
junto con persistencia, federación y TTL de sesión. Mismo patrón "singleton de módulo" que el
Router y `LLMCache`, con la misma consecuencia de estado por-proceso.

## 10. Aislamiento en tests: fixture autouse para el singleton de sesiones `[dominio]`

`session_store` es un singleton de módulo sobre un `dict` que persiste todo el proceso — sin
limpiarlo, una sesión creada en un test se filtraría al siguiente (y los ids son uuid4
aleatorios, difíciles de rastrear). Fixture `autouse` `isolate_session_store` en
`tests/app/conftest.py` limpia `_sessions` antes Y después de cada test. Mismo patrón que
`isolate_cache_from_real_redis` e `isolate_input_guards_from_real_moderation` ya existentes.
Mutar el `_sessions` privado en teardown de test es aceptable (código de test, no producción
alcanzando internos).

## 11. Qué NO se tocó `[agnóstico]`

- `build_cache_key`: sin cambios (el historial no es slot, §3).
- El contrato de respuesta en el camino feliz single-turn: sin cambios (regresión M1 fijada
  por `test_absent_session_id_is_single_turn`).
- `_refusal_kind`/`_citations`: operan sobre la respuesta ACTUAL, sin cambios — el historial
  es solo contexto previo para el modelo.
- `context/reference.py`, el prompt: sin cambios.
- Frontend Streamlit: sin tocar (backend-only, §2).
