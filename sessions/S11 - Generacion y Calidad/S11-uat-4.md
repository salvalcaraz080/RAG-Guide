# S11 · UAT — Brief 4: los tres defectos de la UAT conjunta y la pregunta enorme

> **Por qué es tuyo** (caso 1 de la frontera): los tres defectos los encontraste leyendo la
> interfaz, y solo leyéndola se comprueba que están resueltos. CC ha verificado el contrato (tests),
> la medición final y la API por HTTP.
>
> **Coste:** 4 preguntas reales a Haiku 5.5, menos de 0,01 USD; la del caso 4 no llega al modelo.
> Cada respuesta tarda 10–20 s.

## 0 · Preparación

```powershell
docker compose ps        # api, db y frontend en marcha
```

Abre `http://localhost:8501` (recarga si estaba abierta). Pulsa **🔄 New conversation** antes de
los casos 1 y 3.

---

## 1 · Sin cita y paráfrasis repetidas (caso 2b de la UAT anterior)

```
What does Q-ST-80 require for handling software problem reports and nonconformances?
```

**Esperado:** cada requisito aparece **una sola vez** — o como cita literal entre comillas con su
`[n]`, o como frase propia cuando sintetiza o compara —, nunca «paráfrasis…, as stated: "la misma
frase" [n]». Si el modelo aún repite, el sistema sirve la cita sola.

**Si ves una repetición:** cópiala tal cual; es un caso que la red no cubre.

**Resultado:**: OK.

## 2 · «No encontrado» legible (caso 4)

Primero, en la misma conversación:

```
What are the differences between software criticality category B and category C in terms of required verification, testing and coverage?
```

y después:

```
And which of those requirements apply to category C only?
```

**Esperado:** si falta algo, al final aparece un **bloque aparte**:

**Resultado:**: OK.

```
I could not find in the indexed ECSS standards:
- …
- …
```

con cada viñeta formulada como **lo que preguntaste** (p. ej. «requirements that apply only to
category C»), no como un hecho sobre el estándar (no «Category C-specific rows in the tailoring
matrix»).

**Resultado:**: OK. Se podría añadir el icono de atención al inicio de "I could not find in the indexed ECSS standards"? Para resaltar lo que no se encontró.

## 3 · Fuera de dominio: solo el mensaje (caso 3)

```
What's a good recipe for a chocolate cake?
```

**Esperado:** solo «Out of scope: …». **Sin** «Cited» y **sin** el desplegable «In context, not
cited». (En modo desarrollo, el panel lateral sigue mostrando la evidencia recuperada.)

**Resultado:**: OK 

## 4 · Pregunta enorme

Pega en el chat un texto de más de 4.000 caracteres (por ejemplo, varios párrafos de cualquier
documento, o una pregunta repetida muchas veces).

**Esperado:** un error rojo claro: «The backend rejected the request (HTTP 422): Message too long:
the limit is 4,000 characters. Ask about one topic at a time.» No hay espera de 10–20 s: se rechaza
antes de buscar ni llamar al modelo.

**Resultado:**: el esperado, aunque me gustaría modificar el mensaje de error, el usuario no tiene por qué saber que es un error 422. El mensaje podría ser "Your question exceeds the input limit of 4000 characters. Please summarize it or ask one topic at a time."

---

## Qué reportar a Claude Chat

- Casos 1–4: ¿se resuelve cada defecto? Si no, el texto exacto que viste.
- Caso 2: ¿las viñetas del bloque se leen como lo que *no se encontró* y no como contenido?

Feedback a anotar como mejora de UI a futuro: Los chunks recuperados a veces se vuelven ilegibles, les faltan saltos de linea, y las tablas son todo palabras separadas por "|". Tenemos que darles formato de alguna forma.