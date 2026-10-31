# S14 · UAT — Brief 1: huecos tipados y degradaciones visibles

> **Ejecutada por CC** (2026-10-30, por HTTP contra el contenedor; la ejecución queda en
> «Resultado» de cada caso). Primero, 6 preguntas; después, q14 y q12 repetidas una vez, porque una
> sola ejecución no decide en un caso no determinista. 10 turnos, 0,022 USD. Respuestas completas:
> `data/groundtruth/generation/runs/uat_s14_20261030.json`.
>
> **Lo que sigue siendo del usuario** (caso 2 de la frontera: juicio de producto, sin oráculo): si
> los bloques ❗ y ↪️ dicen lo que un ingeniero necesita oír, y si el desplegable se entiende en el
> navegador. CC no ha visto la UI renderizada: comprobó las líneas que pinta
> (`degradation_notes`), no el desplegable.
>
> **Premisas comprobadas contra el contexto determinista** (`scripts/gap_kinds_precheck.py`):
> «all services are optional» se inyecta en q14 turno 1 como [7] E-ST-70-41C § 1; «…(refer to
> ECSS-E-ST-70-01)» en q14 turno 3 como [3] § 6.18.1.1; 5.3.1 de Q-ST-80C en q12 como [1]. Lo que
> genera el modelo no es determinista: cada caso dice cuántas veces salió en la pasada.

## 0 · Encender los huecos tipados

En `.env`, cambia **una línea**: `GENERATION_GAP_KINDS=off` → `GENERATION_GAP_KINDS=on`. Después:

```powershell
docker compose up -d --force-recreate api
docker compose exec api python -c "from app.config import get_settings; print(get_settings().GENERATION_GAP_KINDS)"
(Invoke-RestMethod http://localhost:8000/api/v1/prompt).instructions.Contains("Kinds of not-found aspects")
```

**Esperado:** `on` y `True`.

## 1 · q14, los tres turnos en la misma conversación

En la UI (http://localhost:8501), **🔄 New conversation**, y los tres mensajes seguidos:

```
Our on-board software has to implement the packet utilization standard. Which services are we obliged to support and which are optional?
```
```
How do we document the subset of services and subtypes that our mission actually uses?
```
```
And if the spacecraft is also supposed to run on-board control procedures, what does the flight software have to provide to execute them?
```

**Esperado, turno 1 (caso de la premisa):**
- Un bloque **«❗ The indexed ECSS standards contradict what the question assumes:»** con una viñeta
  del tipo `services the on-board software is obliged to support — “all services are optional” [7]`.
- En «Cited», **[7] ECSS-E-ST-70-41C · clause 1**, con la marca «➕ added to complete the clause»
  en ese átomo.
- En la pasada salió en 3 de las 4 veces que se respondió el turno. La otra lo dejó como «⚠️ I could
  not find…» (`which specific services must be supported and which are optional`): si te sale así,
  repite el turno en una conversación nueva.
- Puede aparecer además un bloque «↪️ … leave this to another document:» que cita el «Space-to-Ground
  Interface Control Document» (1 de 4). CC lo considera **dudoso**: anótalo si sale.

**Esperado, turno 2 (sin caso objetivo):** una respuesta sobre el documento de definición de
utilización de paquetes específico de la misión y las declaraciones por servicio y subservicio,
con un bloque «⚠️ I could not find…» (en la pasada: formato y ubicación de esa documentación,
contenido del Space-to-Ground ICD). **Sin** bloque ❗ ni ↪️.

**Esperado, turno 3 (caso de la delegación):**
- Un bloque **«↪️ The indexed ECSS standards leave this to another document:»** con la cita
  `…complements the spacecraft on-board control procedures standard (refer to ECSS-E-ST-70-01)` y su
  `[n]`, que en «Cited» es **ECSS-E-ST-70-41C · clause 6.18.1.1**. En la pasada: 2 de 3.
- Y un «⚠️ I could not find…» con lo que el software de vuelo tiene que proveer.
- **El sistema sigue sin traer E-ST-70-01C** (eso no ha cambiado: es S13). Lo nuevo es que ahora lo
  dice con su prueba, en vez de un hueco a secas.

**Juicio:** ¿el bloque ❗ del turno 1 y el ↪️ del turno 3 dicen lo que un ingeniero necesita oír, y la
cita lo prueba?

**Resultado (CC):** parcial.
- **Turno 1: OK, 2 de 2.** Bloque ❗ con «all services are optional» sobre E-ST-70-41C § 1, `verified`.
  El tramo citado varía (una vez la frase entera). Sin el `delegated` dudoso.
- **Turno 2: OK, 2 de 2.** Solo «⚠️ I could not find…», sin ❗ ni ↪️.
- **Turno 3: KO, 0 de 2.** Las dos veces `not_in_material`, sin ↪️: el modelo no declaró `delegated`,
  y no hubo ningún degradado. Con la pasada, q14-a1 queda en **2 de 5**. El guion esperaba un 2 de
  3 y lo daba por salir: es más inestable de lo que dijo la pasada.
- **Juicio de producto: pendiente del usuario.**

## 2 · q12: delegación a M-ST-80 (y el matiz de q12-a3)

**🔄 New conversation**:

```
How does Q-ST-80C expect software-related risks to be identified and handled, and how are they supposed to feed into the project's overall risk management process?
```

**Esperado:** un bloque «↪️ …» con «Risk management for software shall be performed by
cross-reference to the project risk policy, as specified in ECSS-M-ST-80.» [1] (Q-ST-80C 5.3.1).
En la pasada: 3 de 3. El aspecto de la viñeta fue siempre «cómo alimentan los riesgos de software la
gestión de riesgos del proyecto», **no** «los criterios de aceptación» (q12-a3): ver
`S14-decisions.md` §2.3.

**Juicio:** ¿es correcto presentar esa parte de la pregunta como «lo regula otro documento»?

**Resultado (CC):** parcial; **5.3.1 sale 1 de 2** (con la pasada, 4 de 5).
- Ejecución 1: **sin 5.3.1**. `delegated` a otra cosa, «critical item control — For critical item
  control, ECSS-Q-ST-10-04 shall apply» (5.3.2.1), y la integración en la gestión de riesgos queda
  como «⚠️ I could not find…».
- Ejecución 2: `delegated` a 5.3.1, con una cita mínima («as specified in ECSS-M-ST-80»), y otro a
  Q-HB-80-03 (6.2.2.3).
- Todos `verified`, y las remisiones son reales. Lo que varía es **cuál** tipa el modelo.
- **Juicio de producto: pendiente del usuario.**

## 3 · Un hueco real que no cambia

**🔄 New conversation**:

```
How do we keep track of exactly which version of every source file and document went into each software build we hand over to the customer, and who is allowed to change it afterwards?
```

**Esperado:** solo el bloque de siempre, «⚠️ I could not find in the indexed ECSS standards:», con
dos viñetas (versión exacta de cada fichero en cada entrega; quién puede cambiarlo después), **sin**
cita ni `[n]` en ellas, y **sin** bloques ❗ ni ↪️. Así salió en 3 de 3.

**Resultado (CC):** OK. Estado `partial`, dos viñetas sin cita ni `[n]`, sin bloques tipados.

## 4 · Degradación técnica visible, y una respuesta sin ninguna

**a.** Vuelve al **turno 3 del caso 1** (ya respondido): bajo la respuesta hay un desplegable
cerrado **«⚙️ Technical limits on this answer (3)»** (en la pasada: clauses included only in part,
cross-references not followed y cross-referenced clauses included only in part, con sus cláusulas).
No hace falta forzar nada: 13 de las 14 consultas del golden set llevan alguna.

**b.** **🔄 New conversation**:

```
What does Q-ST-80 require for handling software problem reports and nonconformances?
```

**Esperado:** **ningún** desplegable «⚙️ Technical limits». Es la única consulta del golden set sin
degradación (3 de 3).

**Juicio:** ¿el desplegable es discreto, y se entiende lo que dice?

**Resultado (CC):** OK en el contrato; el render, pendiente.
- **a:** el turno 3 lleva 3 degradaciones en una ejecución (`expansion_truncated`,
  `xref_over_limit`, `xref_truncated`), y `degradation_notes` da una línea para cada una.
- **b:** q02 sale con `degradations` vacía, así que no hay desplegable.
- **Fuera de lo esperado en q02:** el guion no lo decía, pero la respuesta salió `partial`, con un
  `delegated` real («For software nonconformance handling, ECSS-Q-ST-10-09 shall apply», 5.2.6.1).
  En la pasada fue `answered` sin huecos, 3 de 3.
- En las 10 respuestas, todo `[n]` del texto está en `citations`, y hay 0 degradados, 0 fallback y
  0 `needs_review`.

## 5 · Volver al default

En `.env`, `GENERATION_GAP_KINDS=on` → `GENERATION_GAP_KINDS=off`, y:

```powershell
docker compose up -d --force-recreate api
docker compose exec api python -c "from app.config import get_settings; print(get_settings().GENERATION_GAP_KINDS)"
```

**Esperado:** `off`.

**Resultado (CC):** OK, `off` en el contenedor. La edición con PowerShell añadió un BOM a `.env`;
retirado.

---

## 6 · Juicio de producto (chat, 2026-10-30)

> Delegado por Salva al chat. Sobre la pasada (`generation_20261024-140241`, `S14-decisions.md` §2)
> y las 10 respuestas de esta UAT. **Render de los bloques ❗ y ↪️: OK (Salva, 2026-10-30).** El
> desplegable de degradaciones se juzga con el filtro del brief 2.

**Principio del juicio: el error es asimétrico.** Un hueco que debía tiparse y sale como
`not_in_material` es el comportamiento de antes: no empeora nada. Un hueco tipado sin razón le dice
al ingeniero algo falso («tu premisa es errónea», «esto lo regula otro»). Pesa la **precisión** de los
tipados servidos, no su recall.

| Caso | Juicio | Por qué |
|---|---|---|
| **q14 t1 ❗** «all services are optional» (E-ST-70-41C §1) | **OK** | Es lo que un ingeniero necesita oír: en PUS-C los servicios se seleccionan por misión. Matiz de dominio que el bloque no da: implementar un servicio sí obliga a su capacidad mínima. No invalida el ❗ (la premisa era a nivel de servicio), pero es lo que la respuesta debería añadir si la cláusula está en el contexto. 5 de 6: estable. |
| **q14-a1 ↪️** «refer to ECSS-E-ST-70-01» (§6.18.1.1) | **OK cuando sale; inestable (2 de 5)** | Remisión correcta: los requisitos de ejecución de OBCP están en E-ST-70-01C. Cuando falla, vuelve al hueco de antes: no daña. |
| **q12 ↪️** 5.3.1 → M-ST-80 | **OK** | Correcto presentar la integración en el riesgo del proyecto como «lo regula otro documento»: 5.3.1 lo remite literalmente a la política de riesgos de M-ST-80. Las variantes (Q-ST-10-04 para ítems críticos, Q-HB-80-03) son remisiones reales. |
| **q02 ↪️** 5.2.6.1 → Q-ST-10-09 (no conformidades) | **OK**, y mejor que antes | Q-ST-80C delega de verdad el tratamiento de no conformidades en Q-ST-10-09. Pasar de `answered` a `partial` es más honesto, no una regresión. |
| q04 (guía ISVV, ×2), q08 (Q-ST-30-02 FMEA), q13 (SAVOIR) | **OK** | Remisiones reales a documentos que fijan el detalle. |
| **q14 t1 · ej. 3 ↪️** «Space-to-Ground ICD» | **Espurio** | Describe dónde la misión documenta su selección, no que otro documento regule los servicios. 1 de 17 tipados servidos; ningún `premise_contradicted` espurio (criterio 2 limpio). |
| Hueco real (caso 3) | **OK** | Sin cambio. |
| **Degradaciones visibles** | **OK en contrato; la UI sobra en 2 de 4 tipos** | `expansion_truncated` y `xref_truncated` salen en 13 de 14 consultas: son la política `bounded` configurada, la misma categoría que la ventana de historial que CC excluyó. Un panel en el 93 % de las respuestas se ignora (si todo se avisa, nada se avisa). En la UI solo deben contar los que cambian cuánto fiarse: `xref_over_limit`, `routing_filter_empty`, `moderation_unavailable`, `table_labels_unavailable`. El contrato no cambia. |

**Hallazgo de producto: el destino de la remisión suele estar indexado.** M-ST-80C y E-ST-70-01C
están en el corpus. «↪️ The indexed ECSS standards leave this to another document» manda al
ingeniero a leer un documento que el sistema tiene y no ha traído. Es honesto, pero no es lo mismo
que una remisión a Q-ST-10-04 o a la guía de ISVV (fuera del corpus). Distinguirlo es determinista
(el parser de `xref` ya resuelve identificadores contra el corpus): `delegated` a un estándar
indexado = señal de hueco de retrieval, entrada directa a la puerta del grafo de referencias.

**Veredicto:** `gaps` se adopta como default (criterio 1–4 cumplido; precisión de tipados servidos
16/17, 0 premisas espurias). Ajustes: UI de degradaciones limitada a los cuatro tipos que cambian la
confianza. T7 y el tratamiento de los rechazos (`S14-decisions.md` §1.5.2) se quedan como están: sin
caso observado que pida cambiarlos.
