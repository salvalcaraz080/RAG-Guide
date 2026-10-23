# pre-S12 — Brief: historial por turnos y suite de contrato sin red

> Brief de infraestructura previo a S12 (no es contenido de S12). Lee `.claude/CLAUDE.md` y
> `ARCHITECTURE.md` antes de empezar. Rama: `s12-agentes` desde `main`; este brief es su primer
> bloque de trabajo. Los commits los hace Salva.
>
> **Regla del brief:** preparar, no adelantar. Nada de tools, bucles, rol `tool` ni modelo falso con
> guion: eso llega solo si S12 decide que hay bucle. Lo que exija diseño nuevo no se toca aquí:
> `# TODO: pendiente de decisión en Claude Chat`.

---

## 0. Objetivo y alcance

**Objetivo.** Quitar dos deudas que S11 dejó señaladas para M5 y que no dependen de la decisión
agente/workflow:

1. **El historial asume dos mensajes por turno (`-2N`).** La unidad atómica del historial pasa a ser
   el **turno** (pregunta del usuario + respuesta servida), no el mensaje. Hoy la suposición está
   repartida en tres sitios que fallarían en silencio en cuanto un turno tenga más de dos mensajes,
   que es justo lo que trae cualquier function calling:
   - `SessionStore.append_turn`: `max_messages = self._max_history_turns * 2`;
   - `fit_budget`: `turns = turns[2:]` ("history goes in pairs");
   - `Session.history`: lista plana de `Message` que "crece por pares".
2. **La suite de contrato puede hacer llamadas reales.** Ya hubo una accidental. Con un bucle, una
   fuga se multiplicaría por N vueltas. La suite tiene que **fallar** ante cualquier intento de red,
   no simplemente evitarlo por convención.

**Dentro de alcance:** modelo de datos del historial, ventana deslizante, recorte por presupuesto,
serialización al array de mensajes del modelo, composición de la consulta de retrieval con
historial, el bloqueo de red en tests y la documentación.

**Fuera de alcance:**
- El rol `tool`, las tools, el bucle, `submit_answer` y el modelo falso con guion (sería código
  para un bucle no decidido).
- **Qué parte de un turno con tools se guardaría en el historial** (todo el tráfico o solo la
  respuesta final). La estructura nueva no debe impedir ninguna de las dos opciones, pero no se
  decide aquí.
- Cambios de prompt, modelo, retrieval o del contrato HTTP. `corpus/` sigue congelado.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| H1 | La sesión guarda una lista de **turnos** (`Turn`: pregunta del usuario + respuesta del asistente), no una lista plana de mensajes | El turno es la unidad de la conversación. Contar mensajes es un detalle de serialización que cambia en cuanto un turno tenga mensajes intermedios |
| H2 | La ventana deslizante (`MAX_HISTORY_TURNS`) cuenta turnos directamente; desaparece el `* 2` | El nombre del ajuste ya decía "turnos"; el código multiplicaba para traducirlo a mensajes |
| H3 | `fit_budget` descarta **turnos enteros**, del más antiguo al más reciente; desaparece el `turns[2:]` | Recortar a mitad de un turno deja un mensaje huérfano. Con un par tool_call/tool_result, el proveedor rechaza la petición (400) |
| H4 | El aplanado de turnos a `[{role, content}, …]` vive en **un solo sitio** y es la única puerta hacia el modelo | Cuando un turno tenga más mensajes, solo cambia esa función |
| H5 | **Equivalencia byte a byte:** con el historial actual (solo user/assistant), el array de mensajes que llega al modelo y la consulta de retrieval son idénticos a los de antes del refactor | Es un refactor estructural: no debe mover ninguna métrica. Este test sustituye a una pasada del arnés |
| H6 | La suite **bloquea la red por defecto** y falla en voz alta ante cualquier intento de conexión saliente | Evitar llamadas por convención ya falló una vez; tiene que ser una garantía |
| H7 | Las excepciones al bloqueo (si algún test necesita localhost, por ejemplo una BD de integración) se declaran **explícitamente por test o por marcador**, nunca de forma global | Que cada excepción se vea en el código del test |

---

## 2. Trabajo

### 2.1 Historial por turnos

1. **Inventario primero.** Busca todos los sitios que leen o escriben `Session.history` o que
   suponen pares (`* 2`, `[2:]`, `[-2:]`, índices pares/impares, `zip` de user/assistant…). Los tres
   de §0 son el punto de partida, no la lista cerrada. Regístralo en `pre-S12-decisions.md`.
2. Modelo `Turn` y `Session.history: list[Turn]`. La forma exacta (campos, si `Message` se mantiene
   dentro de `Turn`) la decides tú y la registras. El criterio: **H4 tiene que poder ampliarse a
   mensajes intermedios sin tocar la ventana ni el presupuesto.**
3. `append_turn` recibe y guarda un `Turn`. La ventana se aplica por turnos (H2).
4. `fit_budget` y `_history_tokens` trabajan sobre turnos (H3). `dropped_turns` sigue significando
   turnos, ahora literalmente.
5. `_compose_retrieval_query` sigue usando las preguntas de usuario de los turnos previos. Mismo
   resultado (H5).
6. Observabilidad: los campos de log que hablan de historial (`trimmed_history_turns`,
   `history_questions`…) mantienen su significado. Si alguno contaba mensajes, regístralo y
   corrígelo.

### 2.2 Bloqueo de red en la suite de contrato

Dos opciones; elige la más simple que cumpla H6/H7 y registra el porqué:
- **(a)** Bloqueo a nivel de socket (por ejemplo `pytest-socket` con `--disable-socket` y
  `--allow-hosts` o un marcador para las excepciones). Cubre cualquier cliente (LiteLLM, httpx,
  embeddings…). Añade una dependencia de dev.
- **(b)** Un fixture `autouse` en `conftest.py` que parchea `socket.socket.connect` (o las entradas de
  LiteLLM) para lanzar un error explícito. Sin dependencia, pero hay que mantenerlo.

Requisitos en ambos casos:
- El error dice qué test intentó conectar y a dónde.
- El modo de evaluación real (el arnés, `LLM_EVAL_MODE` o lo que hoy permita llamadas reales) **no**
  queda afectado: el bloqueo es de la suite de contrato, no del código de producción ni de los
  scripts del arnés.
- Si hoy algún test hace red sin saberlo, **saldrá en rojo al activar el bloqueo**. No se silencia:
  se mockea o se marca como excepción justificada, y se registra en `pre-S12-decisions.md`. Es el
  hallazgo más valioso del bloque.

### 2.3 Docs

`ARCHITECTURE.md` y `.claude/CLAUDE.md` describen el estado actual (historial por turnos, red
bloqueada en la suite, cómo declarar una excepción), sin narrativa histórica. `CHANGELOG.md` se
deriva de `pre-S12-decisions.md`.

---

## 3. Tests de contrato (sin red, sin BD, sin modelo real)

- `uv run pytest` en verde. Recuento final frente a los **1280** de partida, desglosado en
  modificados, nuevos y borrados.
- **Equivalencia (H5):** para un historial de N turnos user/assistant, el array de mensajes hacia el
  modelo y la consulta de retrieval son idénticos a los de la implementación anterior. Se fijan como
  valores esperados literales antes del refactor, no se recalculan con el código nuevo.
- **Ventana (H2):** con `MAX_HISTORY_TURNS = k`, tras k+3 turnos quedan exactamente los k más
  recientes, completos.
- **Presupuesto (H3):** cuando el presupuesto obliga a recortar, se descartan turnos enteros desde
  el más antiguo; nunca queda un mensaje suelto; `dropped_turns` coincide.
- **Atomicidad a prueba de futuro:** un test con un `Turn` que contiene un mensaje intermedio
  (construido en el test, sin rol nuevo en el contrato público) demuestra que la ventana y el
  presupuesto tratan el turno como bloque. Si la forma elegida en §2.1.2 no lo permite sin
  introducir el rol `tool`, se deja `TODO` y se registra; no se adelanta el rol.
- **Red (H6):** un test comprueba que un intento de conexión dentro de la suite falla con el error
  explícito. Otro comprueba que una excepción declarada funciona (si existe alguna).

---

## 4. Llamadas reales

**Ninguna autorizada ni necesaria.** H5 se demuestra con equivalencia byte a byte, lo que es más
fuerte y más barato que una pasada del arnés. Si por algún motivo H5 no pudiera garantizarse
(cambio intencionado en el array de mensajes), **para y reporta** antes de seguir: ese cambio exigiría
una medición con el arnés contra `generation_20261019-141324` y un criterio preregistrado.

---

## 5. UAT (usuario)

Propón un script de UAT corto para la UI de Streamlit (Docker lo levanta Salva). Mínimo:
1. Conversación de 3–4 turnos con anáfora ("¿y en Q-80?"): el seguimiento resuelve como antes.
2. Conversación que supere `MAX_HISTORY_TURNS` (o con el valor bajado temporalmente por env): sigue
   respondiendo y los turnos más antiguos dejan de influir.
3. Una pregunta puntual sin historial: la respuesta, las citas y la latencia son del mismo orden que
   la referencia.
4. Ejecutar `uv run pytest` en la máquina de Salva sin red (o con la red cortada): en verde.

---

## 6. Cierre

- `pre-S12-decisions.md` en `RAG-Guide/sessions/pre-S12/`: inventario de §2.1.1, forma elegida de
  `Turn` y porqué, opción de bloqueo de red y porqué, tests que hacían red sin saberlo (si los hubo),
  recuento de tests y etiquetado agnóstico/dominio.
- `pre-S12-uat.md` con el script de UAT.
