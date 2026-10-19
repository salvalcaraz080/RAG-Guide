# S11 · UAT — Briefs 2 y 3 (conjunta): la respuesta estructurada con Haiku 5.5, en la interfaz

> **Sustituye a `S11-uat-2.md`.**
>
> **Por qué es tuyo** (caso 1 de la frontera): CC ha verificado el contrato (1263 tests), el arnés
> (42 ejecuciones reales con Haiku 5.5) y la API de Docker por HTTP (q06 responde sin 502), pero no
> puede ver la interfaz renderizada: si los rótulos, avisos y la cita mostrada aparte se leen bien es
> un juicio de quien la usa.
>
> **Coste:** cada pregunta es una llamada real a Haiku 5.5 (~0,002 USD). Unas 7 llamadas, menos de
> 0,02 USD. **Tiempo:** cada respuesta tarda unos 10–20 s (el modelo razona antes de responder).

## 0 · Preparación

```powershell
docker compose ps        # api, db y frontend en marcha
```

La API ya está reconstruida con LiteLLM 1.106.0 y el `.env` nuevo. Abre `http://localhost:8501`
(recarga si estaba abierta). Pulsa **🔄 New conversation** antes de cada caso salvo el 4.

---

## 1 · Criticidad B vs C (q06): responde, sin 502

```
What are the differences between software criticality category B and category C in terms of required verification, testing and coverage?
```

**Esperado:**
- Una respuesta (no un error 502). Con Haiku 4.5 este caso fallaba; con Haiku 5.5, 0 de 3 fallos
  en el arnés y el smoke test respondió `partial`.
- Secciones o viñetas con `[n]`; bajo la respuesta, **«Cited»** y **«In context, not cited»**.
- Si es `partial`, cierra con **«Not found in the indexed ECSS standards: …»**.
- Panel lateral: **Model** `claude-haiku-5-5`, **Tokens** (~6.500 de entrada), **Prompt cache**
  (~2.493 leídos, o escritos si es la primera llamada en 5 minutos), **Cost** (~0,002 USD),
  **Input budget** (~7.600 / 30000).

**Resultado:** Se cumple todo lo esperado. Incluida la respuesta parcial con "Not found in the indexed ECSS standards:..."

## 2 · El antiguo «rechazo que cita» (q01): `partial` con frase de cierre

```
Which project reviews does ECSS-E-ST-40C define for software, and what documentation is expected to be delivered at each of them?
```

**Esperado:**
- No empieza por «I could not find this…»: contesta lo que encuentra, con `[n]`.
- Termina con **«Not found in the indexed ECSS standards: …»**.
- **No** aparece el aviso «This answer does not cite any source».
- **Cita mostrada aparte (brief 3):** esta consulta produjo 2–4 en cada ejecución del arnés.
  Busca afirmaciones con la forma `Afirmación [n] — “tramo exacto” [m].`: el tramo entre comillas
  es literal del estándar, verificado, aunque la afirmación lo parafrasee. Si en tu ejecución no
  sale ninguna, prueba el caso 2b.

**Resultado:** Se cumple todo lo esperado.

### 2b · (si hace falta) otra con citas aparte (q02)

```
What does Q-ST-80 require for handling software problem reports and nonconformances?
```

Produjo 3–4 citas aparte en cada ejecución del arnés.

**Revisa:** ¿se entiende que el tramo entre comillas con su `[m]` es la cita literal y la frase de
delante la afirmación? ¿El guion largo (` — `) funciona, o prefieres otra forma (línea debajo,
paréntesis)?

**Resultado:** Sí se entiende, no obstante, la respuesta se ve repetitiva. Copio-pego literal la respuesta que dio para una de las secciones

```
Problem reporting procedures
- The supplier shall ensure the correct application of problem reporting procedures, as stated: "The supplier shall ensure the correct application of problem reporting procedures." [8]
- Nonconformances and software problem reports detected during testing shall be properly documented and reported to those concerned: "The supplier shall ensure that nonconformances and software problem reports detected during testing are properly documented and reported to those concerned." [4]
- Any discovered problems shall be documented in nonconformance reports: "Any discovered problems shall be documented in nonconformance reports." [7]
```
El modelo parafrasea la cita, y luego además incluye la cita tal cual. Creo que la respuesta debería ser una u otra. Cuando la respuesta exija resumen o síntesis, que el modelo la elabore. Pero si simplemente va a parafrasear la cita, entonces que la incluya ya ya, pero no que se repita de esa manera.

## 3 · Fuera de dominio

```
What's a good recipe for a chocolate cake?
```

**Esperado:** «Out of scope: this question is not about the ECSS standards this assistant covers.»
y quizá una frase de motivo. Sin «Cited» ni aviso de «no cita ninguna fuente».

**Resultado:** El mensaje Out of scope está. Sin embargo, también se muestra el desplegable con "In context, not cited". En un caso de respuesta out of scope, tenemos que mostrar únicamente el mensaje de rechazo.

## 4 · Seguimiento multi-turn

Sin pulsar *New conversation* tras el caso 1:

```
And which of those requirements apply to category C only?
```

**Esperado:** responde sobre la categoría C enlazando con la pregunta anterior; la numeración `[n]`
es la de **esta** respuesta. En el panel, el presupuesto incluye ahora el historial.

**Resultado:** Responde específicamente sobre la categoría C. Sin embargo, en este caso la respuesta parcial se ve rara, lo que incluye en "Not found..." parece de hecho sacado de los estándares:
```
Not found in the indexed ECSS standards: A requirement that applies to category C software only; Category C-specific verification or testing requirements beyond the coverage values; Category C-specific rows in the ECSS-Q-ST-80 tailoring matrix.
```
No se entiende muy bien esta parte.

## 5 · Rótulos de evidencia (brief 2)

En cualquiera de las respuestas, abre una cláusula de «Cited» y el desplegable «In context, not
cited».

**Revisa:**
- ¿Se entiende la diferencia entre **«Cited»** e **«In context, not cited»**?
- Si alguna cláusula lleva **«➕ N added»** y átomos «➕ added to complete the clause», ¿se entiende
  que los añadió el sistema y no la búsqueda?
- La autoridad (⚖️ normative / ℹ️ informative) sigue en cada átomo.
- **Haiku 5.5 cita más cláusulas que antes** (la precisión baja, `S11-decisions-3.md` §3): mira si
  en «Cited» aparecen cláusulas que te parezcan de contexto general más que de la respuesta. Tu
  impresión es un dato para esa decisión.

**Resultado:** Se cumple todo lo esperado. Y sí, mete más citas. Aunque algunas son de contexto más general, sí parecen relevantes (a vigilar). 

## 6 · `needs_review`, reasignación o «≈ Not verbatim»

La pasada del arnés **no produjo ninguno** (0 reasignaciones, 0 `modal_edit`, 0 no literales). Si
en los casos anteriores aparece **«⚠️ Needs review…»**, **«≈ Not verbatim…»** o, en el panel,
**«Reassigned [a] → [b]»**, anota la pregunta: es un caso real.

**Resultado:** No vi nada de esto.

---

## Qué reportar a Claude Chat

- Casos 1–4: ¿coincide con lo esperado? (y si q06 dio 502, con su hora).
- Caso 2: ¿la cita mostrada aparte se lee bien? ¿Otra forma?
- Caso 5: ¿los rótulos se entienden? ¿Te sobran cláusulas en «Cited»?
- Tiempo de espera: ¿10–20 s es aceptable en la interfaz?

Los tiempos de espera son aceptables.