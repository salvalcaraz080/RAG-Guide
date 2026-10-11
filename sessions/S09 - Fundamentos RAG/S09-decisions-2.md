# S09 · Decisiones — Brief 2 (retrieval conversacional y contrato de rechazo)

> Decisiones tomadas al implementar `S09-brief-2.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Las E1–E7 del brief no se repiten.
>
> **Verificación:** `uv run pytest` **1002 verde** (990 + 12 netos). Contra la pila real: smoke de
> siete peticiones con la sesión encadenada del brief (§1) y una iteración de prompt sobre E6 con
> dos casos más (§2). Coste total ≈ 0,04 USD.

---

## 1. Smoke end-to-end

| # | Pregunta | `history_questions` | Resultado |
|---|---|---|---|
| 1 | B/C (abre la sesión) | 0 | Responde; cita solo `[5]` = `Q-ST-80C §D.1`. **Sin muletilla**: abre con «The indexed ECSS standards define…» |
| 2 | «And what does ECSS-Q-ST-80C say about it?» | 1 | **Resuelto.** Recupera `Q-ST-80C §D.1` y responde con la tabla. En S09-1 rechazaba |
| 3a | «And for category C?» | — | **HTTP 422**: 19 caracteres, por debajo del `min_length=20` de `QueryRequest` (§3.1) |
| 3b | «And what about category C then?» | 2 | **Resuelto.** Responde categoría C desde `§D.1` |
| 4 | Cambio de tema: «requirements for software configuration management» | 2 | **Sin contaminación visible**: las 10 cláusulas recuperadas son de gestión de configuración (`Q-ST-80C §6.2.4.x`, handbooks). Ninguna de criticidad |
| 5 | Identificador E-ST-40C (pregunta suelta) | 0 | Responde con citas `[3]` `[4]` (**E4 funciona**: antes salían vacías), pero **sigue empezando por la frase de rechazo** (§2) |
| 6 | Batería en vacío térmico (pregunta suelta) | 0 | Igual que el 5: cita `[1][2][4]` bajo la frase de rechazo, y además «material available to me» y «you would need to consult the full ECSS-E-ST-10-03C» |

**E1–E4 y E7 funcionan.** Los seguimientos anafóricos y encadenados recuperan sobre su tema, el
cambio de tema no se contamina en este caso, un rechazo conserva sus citas y `refusal` viaja
tipado. **E5 y E6 no se cumplen del todo** (§2).

---

## 2. E6 no se cumple, y E5 abre un riesgo nuevo

### 2.1 Una iteración de prompt, sin efecto

Tras el smoke se endurecieron las reglas 6–8 de `system.md`: una respuesta con cualquier fuente
relevante es parcial y nunca rechazo, «aunque el estándar que nombra el usuario no esté entre las
fuentes»; se prohíbe remitir al usuario a otros documentos y se añade «material available to me»
a las expresiones vetadas. Repetidos los casos 5 y 6:

- **Los dos siguen empezando por `Not covered by the indexed ECSS standards:`.**
- El 6 empeora: vuelve «The reference material provided…» y «I would need access to the clauses».

El ajuste se conserva porque expresa E6 mejor que la primera redacción, pero **no tiene efecto
medido**. La iteración se cortó en una, como se acordó.

**Lectura [AGNÓSTICO].** Con Haiku, la instrucción de rechazo (frase exacta al inicio) pesa más que
la regla de respuesta parcial. Dos causas plausibles, sin verificar: el propio prompt sigue
llamando «reference material» al bloque en sus reglas internas (el modelo lo repite), y la
decisión «¿hay algo relevante?» se resuelve a favor del rechazo en cuanto falta el estándar
nombrado. Más prompt es la vía barata; la estructural es que el rechazo deje de depender de un
prefijo de texto (campo tipado vía structured output, a costa del streaming).

### 2.2 La frase nueva hace afirmar al modelo cosas falsas sobre el corpus [DOMINIO] — hallazgo principal

En el caso 5 repetido, el modelo dice: *«The specific standard ECSS-E-ST-40C is not included in the
indexed material»*. **Es falso:** `ECSS-E-ST-40C Rev.1` está indexado; lo que pasa es que el
retrieval en modo `vector` no trajo ninguna de sus cláusulas para esta pregunta (hallazgo lateral
de S09-1, `S09-decisions.md` §1).

La causa es la propia E5. «Not covered by the provided material» era cierto por construcción: hablaba
de lo que el modelo recibió. «Not covered by the indexed ECSS standards» **habla del corpus
entero**, y el modelo solo ve los k chunks recuperados. Cada fallo de retrieval se convierte en una
afirmación falsa sobre el corpus, dicha al usuario con la voz del producto. El problema de
confianza que E5 quería resolver se invierte: antes la frase sonaba a tubería; ahora puede ser
incorrecta.

Para Claude Chat: la honestidad de la frase depende de la recall del retrieval. Opciones a
valorar: una formulación que no afirme ausencia en el corpus («I could not find this in the
indexed ECSS standards»), o resolver antes el fallo de identificadores para que la afirmación sea
cierta más a menudo.

---

## 3. Hallazgos fuera de alcance

### 3.1 `min_length=20` bloquea seguimientos cortos [AGNÓSTICO]

«And for category C?» (19 caracteres) devuelve **422** antes de llegar al servicio. El mínimo se
puso en M1 para rechazar preguntas triviales sin gastar una llamada; en una conversación, los
seguimientos cortos son lo normal. Además, en Streamlit el 422 de `/query/stream` sale como
`requests.HTTPError` y se pinta como **«Could not reach the backend…»**, un mensaje que engaña.
Se decide en Claude Chat (bajar el mínimo, o permitir preguntas cortas cuando hay sesión).

### 3.2 La contaminación por cambio de tema no está medida

Un solo caso limpio (§1, caso 4) no mide nada. El riesgo que el brief acepta (E1) sigue siendo una
suposición razonable, no un dato.

---

## 4. Decisiones de implementación

- **[AGNÓSTICO] El proveedor recibe un texto de query, no `(question, history)`.** El `Protocol`
  cambia a `__call__(query)`. La composición vive en `llm_service._compose_retrieval_query`, que
  es quien tiene la sesión resuelta; el proveedor no sabe que existe una conversación. El canal
  léxico de `hybrid` recibe la misma query compuesta.
- **[AGNÓSTICO] Las preguntas previas salen del historial guardado**, que ya contiene la pregunta
  pelada: ni el recordatorio de citas ni los turnos del asistente entran en la query.
- **[AGNÓSTICO] `history_questions` en `retrieval_completed`**: cuántas preguntas previas entraron,
  sin texto. Es lo que permitirá correlacionar contaminación con N cuando se mida.
- **[DOMINIO] Los valores de `refusal` son los internos ya existentes**, `out_of_domain` y
  `out_of_material`, no los que sugería el brief (`out_of_scope`, `not_covered`). Ya los usaba el
  log `citations_validated`; renombrarlos rompería la continuidad de los logs de S09-1 sin ganar
  nada. Tipo `Refusal = Literal[...]` compartido por `QueryResponse` y `StreamDoneEvent`.
- **[AGNÓSTICO] El aviso «sin fuentes» se pinta solo en turnos que vinieron de un `done`**: un
  mensaje de error guardado en el chat no lleva `citations` y no debe disparar el aviso al repintar.
  Se decide por `refusal is None and not citations`, nunca por el texto.
- **[DOMINIO] La última regla del prompt cambia de «trátalo como out of material» a «déjalo fuera y
  nómbralo como no encontrado»**: la versión anterior empujaba hacia el rechazo total, lo
  contrario de E6.
- **[DOMINIO] La regla 4 (cobertura de listas) se mantiene, en voz de producto**: «make clear it
  covers what the indexed ECSS standards contain», sin abrir con disclaimer. En el smoke, las
  respuestas ya no abren con «Based on the provided material».

---

## 5. Lo que queda abierto (va a `ARCHITECTURE.md` §7)

- **Reescritura de seguimientos con LLM**, diferida; señal disparadora: contaminación observada en
  uso real.
- **E6 sin cumplir** y **la frase marcador nueva afirmando ausencias falsas** (§2): decisión de
  contrato y de prompt para Claude Chat.
- **`min_length=20` frente a seguimientos cortos**, y el mensaje engañoso del frontend ante un 422
  (§3.1).
