# pre-S12 · Decisiones — historial por turnos y suite de contrato sin red

> Decisiones tomadas al implementar `pre-S12-brief.md`, etiquetadas **[AGNÓSTICO]** o
> **[DOMINIO]**. Rama `s12-agentes`. Sin llamadas reales: H5 se demuestra byte a byte.

## 1. Inventario: quién suponía pares de mensajes — [AGNÓSTICO]

Los tres sitios del brief y los que aparecieron al buscar. **Ningún campo de log contaba mensajes**:
`dropped_turns`, `trimmed_history_turns` y `dropped_history_turns` ya contaban pares (= turnos), y
`history_questions` cuenta preguntas. Mantienen su significado, ahora literal.

| Sitio | Suposición | Ahora |
|---|---|---|
| `SessionStore.append_turn` | `max_messages = max_history_turns * 2`; recibía `(user_msg, assistant_msg)` | recibe un `Turn`; la ventana cuenta turnos |
| `budget.fit_budget` | `turns = turns[2:]` («history goes in pairs») | `turns[1:]`: un turno entero |
| `budget._history_tokens` | suma sobre mensajes sueltos | suma sobre todos los mensajes de cada turno |
| `Session.history` | `list[Message]` que «crece por pares» | `list[Turn]` |
| `llm_wrapper._compose_messages` | aplanaba `history` en línea | llama a `to_provider_messages`, la única puerta |
| `llm_service._compose_retrieval_query` | filtraba `role == "user"` sobre la lista plana | `turn.question` por turno |
| `llm_service._append_session_turn` | construía dos `Message` | `Turn.exchange(question, answer)` |
| `generation/harness._replay` | `zip(history[::2], history[1::2])` | un `append_turn` por turno |
| `generation/harness.artefact` | `[msg.model_dump() for msg in history]` | `to_provider_messages(history)`: **mismo formato de artefacto**, plano |
| `retrieval/harness.retrieval_query` | historial de solo `Message(role="user")` | `Turn.exchange(pregunta, "")`: retrieval solo lee las preguntas |
| Tests | `history[4:]`, `history[-2]`, pares en `_history`, `_session_with`, `FakeAnswer` | por turnos |

Fuera del backend nadie lee el historial: el frontend solo lleva `session_id`, y `scripts/` no lo
toca.

## 2. Forma de `Turn` — [AGNÓSTICO]

```python
class Turn(BaseModel):
    messages: list[Message]   # min 2; abre con user, cierra con assistant (validado)
    exchange(question, answer) -> Turn   # el turno de hoy, dos mensajes
    question -> str                      # messages[0].content
def to_provider_messages(history: list[Turn]) -> list[dict]   # único aplanado
```

**Porqué una lista de mensajes y no `(question, answer)` con campos fijos.** El criterio del brief
era que H4 se pudiera ampliar a mensajes intermedios sin tocar ventana ni presupuesto. Con campos
fijos, un turno con intermedios necesitaría un campo nuevo y que el presupuesto lo sumara; con una
lista, la ventana cuenta `Turn`, el presupuesto suma `turn.messages` y el aplanado concatena: los
tres ya tratan un turno de cuatro mensajes como bloque. Se demuestra con un `Turn` de tres mensajes
(user, assistant intermedio, assistant final) **sin rol nuevo**: no hizo falta `TODO`.

**`Message` se queda** dentro de `Turn`, igual que antes (`role: user | assistant`, `content`). El
rol `tool` no existe; si S12 decide bucle, se añade al `Literal` y a `to_provider_messages`, y nada
más cambia.

**La validación (abre con user, cierra con assistant)** es lo que hace de `question` algo seguro y
mantiene la query de retrieval igual a la de antes. No impide ninguna de las dos opciones abiertas
(guardar todo el tráfico de un turno con tools o solo pregunta y respuesta final).

**`question` y no «el primer mensaje de rol user».** Hoy dan lo mismo. Si algún día un mensaje
intermedio fuera de rol `user` (como hacen algunos proveedores con los resultados de tools), el
filtro por rol metería basura en la query de retrieval; `question` sigue siendo la pregunta.

**Retrieval harness con respuesta vacía.** `retrieval_query` construye `Turn.exchange(turno, "")`:
el arnés de retrieval no tiene respuestas y `_compose_retrieval_query` no las lee. Alternativa
descartada: cambiar la firma para que reciba preguntas sueltas, que separaría la composición de la
query del historial real que usa producción.

## 3. Equivalencia byte a byte (H5) — [AGNÓSTICO]

`tests/app/test_history_equivalence.py`, **escrito y fijado contra el código anterior antes de tocar
nada**. La conversación entra por `answer_query` y sale en `_router.acompletion`, así que ninguno de
los dos extremos depende de la forma interna del historial y el mismo test corre antes y después:

- 5 turnos con ventana 3: sha256 del array de mensajes de **cada** llamada al router, los mensajes
  de historial de la última en claro, y las 5 queries de retrieval literales.
- 5 turnos con respuestas de 2.000 caracteres y `MAX_INPUT_TOKENS=4.900`: el presupuesto recorta
  (cláusulas y luego turnos); sha256 de cada array y los prefijos de los turnos que sobreviven.

Pasó antes y después sin tocar un valor. No hay cambio intencionado en el array: no hace falta
pasada del arnés.

## 4. Bloqueo de red: opción (b), fixture propio — [AGNÓSTICO]

**Medido, no supuesto:** `pytest-socket --disable-socket` sobre `test_multi_turn.py` +
`test_queries.py` → **26 fallos y 17 errores** en Windows. Bloquea la *creación* de sockets, y el
event loop de asyncio en Windows construye su self-pipe con `socket.socketpair()`, que allí es una
conexión TCP por loopback. Su otro modo (`--allow-hosts`) solo funciona permitiendo `127.0.0.1` a
toda la suite, que es justo la excepción global que H7 prohíbe. La opción (a) no cumple en la
máquina donde corre la suite.

**(b): `network_guard`, autouse en `tests/conftest.py` (raíz: cubre `tests/app` y `tests/corpus`).**
Parchea las cuatro salidas:

| Salida | Por qué hace falta |
|---|---|
| `socket.socket.connect` / `connect_ex` | clientes síncronos y el event loop *selector* (Linux, Docker) |
| `socket.getaddrinfo` | DNS: todo proveedor real es un hostname; corta antes de salir |
| `BaseProactorEventLoop.sock_connect` | el loop *proactor* de Windows conecta con `ConnectEx`, sin pasar por `socket.connect` |
| `socket.socketpair` | se deja pasar **solo** dentro de esa llamada (thread-local): es el self-pipe del loop, no una salida |

- **El error dice quién y a dónde:** `NetworkBlockedError: tests/…::test_x tried to connect to
  host:port. … declare the host with @pytest.mark.allow_network("host")`.
- **Registra antes de lanzar y falla en el teardown.** El hallazgo de diseño del bloque: el
  servicio captura los errores del proveedor (fallback del Router, 502 tipado), así que una
  conexión bloqueada puede tragarse y el test pasar. Verificado con un test temporal que captura
  la excepción: sale `ERROR at teardown … attempted network connections: ['203.0.113.1:443']`.
- **Excepciones (H7):** `@pytest.mark.allow_network("127.0.0.1", …)` en el test, registrado en
  `pyproject.toml`. Hoy **ningún test la usa** salvo el que la prueba.
- **Evaluación intacta:** los tests `real_llm` quedan fuera del guard (es la suite que llama a
  propósito); el arnés y los scripts no pasan por pytest; el código de producción no se toca.
- **Límite conocido:** cubre lo que ocurre durante un test (setup, cuerpo, teardown de fixtures de
  función). No hay fixtures de módulo ni de sesión en la suite; una conexión en tiempo de import o
  de colección no la vería.

**Tests que hacían red sin saberlo: ninguno.** Con el guard activo, los 1.282 tests de entonces
pasaron en verde sin ningún intento registrado. Las fixtures de aislamiento existentes (moderación,
embeddings, proveedor de contexto) ya cubrían todas las salidas reales; la llamada accidental que
motivó el bloque no tiene hoy un camino abierto. Lo que el bloque añade es que la próxima
**no dependa de que alguien se acuerde**: `test_the_real_router_cannot_reach_a_provider` llama al
Router real sin mock y el guard registra el intento.

## 5. Recuento de tests — [AGNÓSTICO]

**1.280 → 1.296** (`uv run pytest`, también con `uv run --offline` en PowerShell).

- **Nuevos (16):** equivalencia H5 (2); red (7: síncrono, DNS, async en el loop, intento tragado,
  Router real, host declarado, loopback no declarado); ventana k+3 y ventana con turno de tres
  mensajes (2); `Turn` — validación, `question`, aplanado (3); presupuesto por turnos enteros con
  `dropped_turns` y turno con intermedio descartado entero (2).
- **Modificados (13)**, sin cambiar lo que afirman: `test_budget` 1, `test_multi_turn` 7,
  `test_observability` 1, `test_sessions` 4 (uno renombrado: `…stores_pair_in_order` →
  `…stores_the_turn`). Más tres helpers (`_history`, `_session_with`, `FakeAnswer`).
- **Borrados:** 0.

## 6. Observado, no tocado

- **`MAX_HISTORY_TURNS=0` no recorta nada:** `history[-0:]` es la lista entera, igual que con el
  `* 2` de antes. Comportamiento preexistente, sin validación en `config.py` (`int` sin `ge`).
  Para Claude Chat: o `ge=1`, o 0 significa «sin historial» y hay que implementarlo.
- `ruff`: el único aviso (`E402` en `app/main.py`) es anterior y de otro fichero.
