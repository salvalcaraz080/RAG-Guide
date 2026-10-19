# S11 · UAT — Brief 2: la respuesta estructurada en la interfaz

> **Por qué es tuyo** (caso 1 de la frontera): CC ha verificado el contrato (1240 tests), el arnés
> (42 ejecuciones reales) y la API de Docker por HTTP, pero no puede ver la interfaz renderizada: si
> los rótulos, avisos y distintivos se leen bien es un juicio de quien la usa.
>
> **Coste:** cada pregunta es una llamada real al modelo (~0,01 USD). Unas 6 llamadas en total.

## 0 · Preparación

```powershell
docker compose ps        # api, db y frontend en marcha
```

Abre `http://localhost:8501`. Tanto la API como el frontend recargan solos el código nuevo (bind mount);
si la página estaba abierta, recárgala. Pulsa **🔄 New conversation** antes de cada caso salvo el 4.

---

## 1 · Respuesta `answered` (criticidad B vs C)

Pregunta:

```
What does ECSS-Q-ST-80C require for software criticality category B compared with category C?
```

**Esperado:**
- Respuesta en secciones o viñetas, con `[n]` al final de cada afirmación.
- Bajo la respuesta, **«Cited»** con las cláusulas numeradas como sus `[n]`, y un desplegable
  **«In context, not cited (N clause(s))»**.
- Ningún aviso amarillo, salvo que haya una cita no literal (verías «≈ Not verbatim: “…” [n]»).
- En el panel lateral (modo dev), **Verification**: `answered` o `partial` servido, y los recuentos.

**Si sale un error 502** («could not be read»): es la incidencia conocida — esta consulta falló dos
veces seguidas en las dos pasadas del arnés (`S11-decisions-2.md` §6). Anótalo y repite una vez.

## 2 · El antiguo «rechazo que cita»: q01

```
Which project reviews does ECSS-E-ST-40C define for software, and what documentation is expected to be delivered at each of them?
```

**Esperado:**
- La respuesta **no** empieza por «I could not find this…». Contesta lo que encuentra, con `[n]`.
- Termina con la frase fija **«Not found in the indexed ECSS standards: …»** y los aspectos que
  faltan.
- **No** aparece el aviso «This answer does not cite any source»: un `partial` es una respuesta.
- Panel lateral: estado servido `partial`.

## 3 · Fuera de dominio

```
What's a good recipe for a chocolate cake?
```

**Esperado:** «Out of scope: this question is not about the ECSS standards this assistant covers.»
más, quizá, una frase de motivo. Sin «Cited», sin aviso de «no cita ninguna fuente». El desplegable
«In context, not cited» puede aparecer: es lo que el retriever trajo y el modelo no usó.

## 4 · Seguimiento multi-turn

Sin pulsar *New conversation* tras el caso 1, pregunta:

```
And which of those requirements apply to category C only?
```

**Esperado:** responde sobre categoría C enlazando con la pregunta anterior; la numeración `[n]` es
la de **esta** respuesta (no arrastra la del turno anterior).

## 5 · Rótulos de evidencia

En cualquiera de las respuestas anteriores, abre una cláusula de «Cited» y el desplegable «In
context, not cited».

**Revisa:**
- ¿Se entiende la diferencia entre **«Cited»** e **«In context, not cited»**?
- Si alguna cláusula lleva **«➕ N added»** en el rótulo y átomos con «➕ added to complete the
  clause», ¿se entiende que el sistema añadió esos fragmentos y no los trajo la búsqueda?
- La autoridad (⚖️ normative / ℹ️ informative) sigue en cada átomo.

## 6 · Reasignación o `needs_review`

La ejecución del arnés **no produjo ninguno** (0 reasignaciones, 0 `modal_edit`, 0 afirmaciones sin
fuente), así que no hay un caso reproducible que darte. Si en los casos anteriores aparece el aviso
**«⚠️ Needs review…»** o, en el panel lateral, una línea **«Reassigned [a] → [b]»**, anota la
pregunta: es un caso real para revisar.

---

## Qué reportar a Claude Chat

- Casos 1–4: ¿el comportamiento coincide con lo esperado? (y si el caso 1 dio 502).
- Caso 5: ¿los rótulos se entienden, o propones otros?
- Cualquier aviso o distintivo que te pareciera ruidoso o confuso.
