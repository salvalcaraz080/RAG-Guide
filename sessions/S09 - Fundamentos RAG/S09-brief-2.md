# S09 — Brief 2: retrieval conversacional y contrato de rechazo

> Continuación de `S09-brief.md`, sobre el resultado de su UAT (`S09-decisions.md` §5).
> Cuatro cambios pequeños y acotados. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar.

---

## 0. Alcance

**Dentro:** composición de la query de retrieval con historial, retirada de la regla "un rechazo
vacía `citations`", voz de producto en prompt y frase marcador, aviso de respuesta sin citas.

**Fuera (no implementar):** reescritura de la pregunta con LLM, expansión de cláusula, notas
huérfanas, ponderación híbrida o filtro por identificador, política ante cita no verificada.
Si algo parece exigirlo: `# TODO: pendiente de decisión en Claude Chat`.

---

## 1. Decisiones

| # | Decisión | Porqué |
|---|---|---|
| E1 | La query de retrieval = **pregunta actual + las últimas N preguntas del usuario** de la sesión, concatenadas. Sin LLM | El seguimiento anafórico falla en retrieval, no en generación. El uso esperado encadena preguntas sobre el mismo tema, así que la contaminación por cambio de tema es rara y, con un retrieval sesgado a recall, barata |
| E2 | Solo preguntas del usuario, **nunca** respuestas del asistente | Las respuestas son largas: diluyen el vector hacia un centroide |
| E3 | N = `RETRIEVAL_HISTORY_QUESTIONS`, default **2** | Con N=1 un seguimiento encadenado pierde el tema si la pregunta anterior también era anafórica. Knob sin calibrar |
| E4 | `citations` = marcadores válidos **siempre**, también en un rechazo | La regla "rechazo → `citations=[]`" protegía contra la cita fantasma de M1, cuando las citas salían de lo inyectado. Desde D10 salen de lo que el modelo marca: la cita fantasma ya no es posible por construcción y la regla solo produce `[n]` sin fuente debajo |
| E5 | Frase marcador `Not covered by the provided material:` → **`Not covered by the indexed ECSS standards:`** | La frase llega al usuario tal cual; "provided material" es vocabulario de la tubería. La nueva sigue siendo honesta: delimita la cobertura al corpus indexado |
| E6 | Respuestas parciales: contestar lo cubierto, con citas, y nombrar el hueco **al final**. La frase marcador solo cuando no hay nada relevante | Hoy una respuesta parcial empieza por la frase de rechazo y se clasifica como rechazo |
| E7 | Respuesta no rechazada sin ningún marcador válido → **aviso visible** "sin fuentes citadas" | Hoy se presenta como anclada sin estarlo |

---

## 2. Trabajo

### 2.1 Query de retrieval con historial (E1–E3)

- La composición ocurre en el camino de `_prepare`, donde ya se tienen pregunta e historial
  resueltos, y se pasa al proveedor de contexto. El proveedor sigue sin saber qué es una sesión:
  recibe un texto de query.
- Orden: pregunta actual primero, después las anteriores de la más reciente a la más antigua,
  separadas por un salto de línea. Sin sesión o con historial vacío: solo la pregunta actual
  (comportamiento actual, byte-idéntico).
- La moderación sigue corriendo solo sobre la pregunta actual.
- `RETRIEVAL_HISTORY_QUESTIONS` en `Settings`, `.env` y `.env.example`; rango `0..5`. `0` desactiva
  la composición.
- `/api/v1/search` no cambia.
- Log: añade a `retrieval_completed` cuántas preguntas previas entraron en la query. Sin texto.

### 2.2 Retirar el vaciado de citas en rechazo (E4)

- `_citations` deja de vaciar en rechazo: devuelve las cláusulas de marcadores válidos en
  cualquier caso. `_refusal_kind` sigue clasificando, y su resultado sigue yendo al log.
- Frontend: un rechazo con citas pinta sus citas y su traza como cualquier otra respuesta.
- Actualiza `ARCHITECTURE.md` en la sección de la cita fantasma: la excepción desaparece y se
  explica en una línea por qué ya no hace falta (las citas salen de los marcadores, no de lo
  inyectado).

### 2.3 Voz de producto (E5, E6)

- Cambia la frase marcador en `system.md`, en `_refusal_kind` y en los tests. `Out of scope:` no
  cambia.
- Reglas de voz en `system.md`:
  - No hablar al usuario de "provided material", "sources you received" ni "context". La
    referencia es "the indexed ECSS standards".
  - Nunca pedir al usuario que aporte documentos o cláusulas.
  - Respuesta parcial: contestar lo cubierto con sus `[n]` y cerrar con una frase que nombre lo
    que no se encontró en los estándares indexados. Sin frase marcador al principio.
- Las reglas internas que describen el bloque `<source>` pueden seguir hablando de "sources":
  van dirigidas al modelo, no al usuario.

### 2.4 Aviso sin citas (E7)

- Si la respuesta no es rechazo y no tiene ningún marcador válido, el frontend muestra un aviso
  explícito de que la respuesta no cita fuentes.
- Si `QueryResponse` / `StreamDoneEvent` no transportan ya el tipo de rechazo, añade un campo
  tipado (p. ej. `refusal: Literal["out_of_scope", "not_covered"] | None`). El frontend decide por
  ese campo, nunca por substring del texto.

---

## 3. Tests de contrato

- Composición: sin sesión → solo pregunta actual; con historial → orden y N correctos; `N=0`;
  nunca incluye turnos del asistente; la moderación no ve las preguntas previas.
- Un rechazo con marcadores válidos devuelve sus citas; sin marcadores, `[]`.
- `_refusal_kind` reconoce la frase nueva y deja de reconocer la antigua.
- El campo de rechazo viaja en `/query` y en `done` del stream.

## 4. UAT propuesta

CC redacta el guion. Mínimo:

1. Seguimiento anafórico del S09: "And what does ECSS-Q-ST-80C say about it?" tras la pregunta B/C.
2. Seguimiento encadenado: un tercer turno anafórico ("And for category C?").
3. Cambio de tema autocontenido en la misma sesión, para observar la contaminación.
4. El caso 1 de la UAT anterior (identificador explícito): esperado, respuesta parcial con citas y
   el hueco al final, sin "provided material".
5. Fuera de material total: frase marcador nueva, sin citas.
6. Comprobar en logs `retrieval_completed` con el número de preguntas previas.

## 5. Cierre

- `S09-decisions-2.md` con lo decidido al implementar, etiquetado agnóstico/dominio.
- `CHANGELOG.md`, `CLAUDE.md` y `ARCHITECTURE.md` según sus reglas. En `ARCHITECTURE.md` §7:
  "seguimientos conversacionales" pasa a resuelto con su límite (cambio de tema) y aparece
  "reescritura con LLM" como alternativa diferida, con su señal disparadora: contaminación
  observada en uso real.
- `uv run pytest` en verde y mensaje de commit propuesto.
