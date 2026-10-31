# S14 — Brief 1: taxonomía del "sin respuesta normal" en el contrato

> Primer brief de S14. Requiere `s13-orquestacion-agentes` mergeada a `main`; rama nueva
> `s14-multiagente`. Los commits y el merge los hace Salva.
>
> **Regla del brief:** una variable cada vez. La **parte A** no toca el array de mensajes (solo
> contrato y trazas). La **parte B** cambia esquema de salida y prompt **detrás de una opción
> apagada por defecto**. No cambia retrieval, routing, expansión, `xref`, topes, modelo ni límites.
> Si algo de eso pareciera necesario: `TODO: pendiente de decisión en Claude Chat` y se reporta.

---

## 0. Contexto y objetivo

**De dónde sale** (`S14-resumen.md` §4, §7 y §9). El antipatrón de HITL del material —tratar un
error como una revisión— tiene su traducción en RAG: el contrato mezcla, bajo `not_found` y bajo
estados genéricos, cosas de naturaleza distinta.

- **Técnico** (el sistema no pudo hacer su trabajo): fallback de proveedor, recorte por presupuesto,
  plazas de `xref` agotadas, expansiones que no entran. Lo detecta el **código**.
- **De contenido** (el sistema hizo su trabajo y el resultado no es una respuesta normal). Hoy todo
  sale como `not_found`, aunque son tres cosas:

| Caso | Ejemplo con señal (golden set v2) | Hoy | Debería |
|---|---|---|---|
| El material no lo trata | huecos reales | `not_found` | `not_in_material` |
| El material dice lo contrario de lo que la pregunta presupone | **q14 turno 1**: afirma «all services are optional» y declara `not_found` «which services are mandatory» (S13-uat-2, clase C `answered_as_phrased`) | `not_found` | `premise_contradicted` + cita |
| El material remite a otro documento | **q14-a1**: cita «…(refer to ECSS-E-ST-70-01)» y declara `not_found` sobre lo que ese estándar complementa; **q12-a3** (M-ST-80C no fija los criterios) | `not_found` | `delegated` + cita |

**Por qué no clasificar antes de generar** (decidido en chat): el tipo depende de lo que diga el
contexto recuperado; q14 turno 1 parece una pregunta normal hasta leer la cláusula. Ni plantilla
por tipo ni agentes: **una categoría en el esquema sustituye a un párrafo de reglas en el prompt**, y
la verificación de citas existente la ancla.

**Objetivo.** (A) Hacer visible en el contrato toda degradación técnica. (B) Tipar los huecos de
contenido como opción de generación apagada, y medirla contra un criterio fijado aquí.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| T1 | **Parte A:** `QueryResponse.degradations: list[{stage, kind, detail}]`, siempre presente (vacía si nada se degradó). Recoge todo evento técnico que hoy solo está en logs | La degradación se ve en el contrato, no en el estado interno (candidato de Guide S13). El cliente decide por campos |
| T2 | Parte A **no cambia** los campos existentes (`fallback_used`, `truncated`, `input_budget`…) ni el array de mensajes | Aditivo: el frontend y el arnés no se rompen |
| T3 | **Parte B:** opción `GENERATION_GAP_KINDS: off \| on`, **default `off`**. Con `on`, cada aspecto `not_found` del `GenerationOutput` pasa a `{aspect, kind, refs, quote}` con `kind ∈ not_in_material \| premise_contradicted \| delegated` | Ampliar la estructura que existe, no un campo global nuevo. Apagada para medir pareado |
| T4 | `premise_contradicted` y `delegated` exigen `refs` a cláusulas inyectadas y **cita literal**; `not_in_material` lleva `refs` y `quote` nulos | Una afirmación de este tipo le dice al ingeniero "tu premisa es falsa" o "esto lo regula otro": decirlo sin prueba es peor que un hueco |
| T5 | **El código valida y degrada:** la cita pasa por `check_quote` con la misma política que las afirmaciones; solo `verified` (o `misattributed` reasignada) sostiene el tipo. Si no, el hueco se sirve como `not_in_material` y se registra `gap_kind_downgraded` (con el `kind` declarado) en traza y en `verification` | El modelo declara, el código valida. Ningún tipo sin prueba llega al usuario |
| T6 | **El código compone** la frase de cada tipo (frase fija, como la de "no encontrado" hoy), con el marcador `[n]` de la cita | Nada que el código pueda decidir se le pide al modelo |
| T7 | La **normalización del estado no cambia**: un hueco tipado cuenta como `not_found` a efectos de `answered`/`partial` | Una variable cada vez. Si el estado debería tratarlos distinto, se decide en chat con los datos |
| T8 | El prompt añade la regla de los tres tipos **solo con `on`**. Con `off`, array de mensajes y esquema **idénticos byte a byte** a la referencia | La opción no puede mover la referencia apagada. Se acepta que con `on` cambie el hash del prefijo cacheado; se reporta |

---

## 2. Trabajo

### 2.1 Parte A — inventario y contrato de degradación (sin LLM)

1. **Inventario** de todo evento técnico del camino de `/query` que altera lo que recibe el usuario:
   fallback de proveedor, recortes de `_fit_budget` (cláusulas y historial), plazas `over_limit` de
   `xref`, expansiones que no entran por tope o tiempo, reintentos por esquema, moderación de
   guardrails que falla por red, y los que encuentres. Para cada uno: dónde ocurre, si hoy llega al
   contrato, y por qué campo.
2. Los que **no** llegan, entran en `degradations` (T1). `kind` en vocabulario cerrado
   (`Literal`); `detail` con los IDs afectados (`chunk_id`, estándar).
3. **Capas de reintento (solo reporte, sin cambios):** cuántas capas reintentan hoy (LiteLLM,
   reintento por esquema, otras), con qué predicado, y si alguna enmascara una degradación. Es el
   diferido de S13; el chat decide después.
4. UI: mostrar `degradations` de forma discreta en la respuesta. Propón la presentación; nada que
   bloquee la lectura.

### 2.2 Parte B — huecos tipados (opción apagada)

- Esquema (T3), validación (T4–T5), composición (T6), prompt (T8). El esquema sigue viajando
  expandido, sin `$ref` ni defaults, para los dos proveedores.
- Traza: por hueco, `kind` declarado, `kind` servido, veredicto de la cita y refs.
- UI: los tres tipos distinguibles visualmente; `premise_contradicted` y `delegated` con su cita.

### 2.3 Arnés

- **Conjunto objetivo** (casos con señal): **q14 turno 1** → `premise_contradicted`; **q14-a1** →
  `delegated`; **q12-a3** → `delegated`.
- **Métricas nuevas:** por ejecución, huecos por `kind` declarado y servido, degradaciones
  `gap_kind_downgraded`, y la lista completa de huecos tipados **fuera** del conjunto objetivo con
  su cita (para revisión, no para puntuar).
- Las de siempre: recall condicionado, sobre relevantes recuperadas y absolutos, precisión con y sin
  expansión, `verified`, estados servidos, latencia con desglose, coste, tokens.

### 2.4 Comprobación previa sin generación

Sobre el **contexto determinista** de q12, q14 (todos sus turnos) y la referencia
`generation_20261019-224307`:
- ¿Está **inyectada** la cláusula que niega la premisa en q14 turno 1, y la que remite a
  E-ST-70-01 en q14-a1?
- ¿Está inyectada alguna cláusula que **delegue** lo de q12-a3? Si **no** lo está, q12-a3 sale del
  conjunto objetivo (no se puede declarar `delegated` sin la evidencia) y se reporta como tal.

Se reporta antes de la pasada y **no se ajusta nada** a la vista de ella.

### 2.5 Docs

`ARCHITECTURE.md` (contrato: `degradations`, huecos tipados y su validación; prompt: la regla
condicionada) y `.claude/CLAUDE.md` si aplica. `CHANGELOG.md` derivado.

---

## 3. Medición (preregistrada)

### 3.1 Configuraciones

Golden set v2, 14 consultas × 3 ejecuciones, mismo modelo, retrieval y límites que la referencia
(`xref+bounded`):

| Config | `GENERATION_GAP_KINDS` |
|---|---|
| `control` | `off` |
| `gaps` | `on` |

`control` repite los defaults el mismo día. Si se sale de la referencia más que el ruido (±0,03) en
recall o precisión, **para y reporta** antes de comparar.

### 3.2 Criterio de aceptación (fijado antes de medir)

**`gaps` se acepta como nuevo default si cumple todo esto, pareado con `control`:**
1. **Éxito:** cada caso del conjunto objetivo que sobreviva a §2.4 se sirve con su `kind` esperado
   y cita `verified` en **≥ 2 de 3** ejecuciones. Con tres casos, hacen falta al menos 2 de 3
   casos; con dos, los dos.
2. **Sin falsos tipados servidos:** ningún `premise_contradicted` servido fuera del conjunto
   objetivo que la revisión del chat juzgue espurio. (Los `delegated` fuera del objetivo se revisan
   igual; uno espurio no tumba el criterio, se registra.)
3. **`verified`** sigue en 1,00 sobre las afirmaciones.
4. **Recall** (condicionado y sobre relevantes recuperadas) y **precisión** no bajan más de 0,03, y
   las relevantes citadas en absoluto no bajan.

Se reportan sin ser guarda: `gap_kind_downgraded` por ejecución, distribución de estados servidos,
número de huecos por consulta, latencia (mediana, p95, desglose), coste por turno, tokens de salida
(la regla nueva puede alargarla).

**Con 2–3 casos, el resultado es indicativo.** La confirmación llega con el golden set 1.2.

**CC no aplica el criterio ni cambia defaults:** reporta la tabla y los casos. Si `gaps` no pasa,
**no se itera** (ni prompt, ni esquema, ni validación): se reporta por caso **por qué no**
(`declared_not_in_material`, `declared_kind_wrong`, `downgraded_quote_not_verified`,
`evidence_not_injected`).

### 3.3 Llamadas reales autorizadas

- **Una pasada** de 2 configuraciones ≈ **0,25–0,30 USD** (referencia S13: ≈ 0,25 por dos
  configuraciones).
- **Segunda pasada** completa solo si falla el instrumento (no el resultado): `control` fuera del
  ruido, errores de proveedor, artefacto incompleto.
- La parte A no requiere llamadas.

---

## 4. Tests de contrato (sin red)

- `uv run pytest` en verde. Recuento frente a 1477.
- **T8:** con `off`, equivalencia byte a byte del array de mensajes y del esquema frente a hoy.
- **T1–T2:** cada evento del inventario produce su entrada en `degradations`; sin degradación, la
  lista existe y está vacía; los campos previos no cambian.
- **T4–T5:** `premise_contradicted`/`delegated` con cita `verified` → servido; cita no literal,
  ref no inyectada o cita ausente → `not_in_material` + `gap_kind_downgraded`; `misattributed` →
  reasignado y servido.
- **T6:** composición por tipo con marcador correcto; `not_in_material` sin marcador.
- **T7:** estado servido igual que hoy con huecos tipados.
- Esquema expandido válido para los dos proveedores con `on`.

---

## 5. UAT (usuario)

Propón el script. Cada caso con su **premisa comprobada contra el contexto determinista** y lo
esperado en **todos** los turnos de la consulta (fallo de script de S13). Mínimo, con
`GENERATION_GAP_KINDS=on`:
1. q14 turno 1 (PUS): el hueco "servicios obligatorios" se sirve como premisa contradicha, con la
   cita de la opcionalidad.
2. q14-a1: el hueco se sirve como delegación a E-ST-70-01, con la cita de la remisión.
3. Un hueco real (`not_in_material`) que no cambia.
4. Una respuesta con degradación técnica visible (forzada si hace falta, p. ej. recorte de
   presupuesto) y otra sin ninguna.

---

## 6. Cierre

- `S14-decisions.md` en `RAG-Guide/sessions/S14 - Multiagente/`: §1 preregistro (inventario de
  degradaciones, comprobación previa) escrito **antes** de medir; resultados después (tabla del
  criterio sin aplicarlo, casos uno a uno con su "por qué no", lista de huecos tipados fuera del
  objetivo con su cita, reporte de capas de reintento), recuento de tests, etiquetado
  agnóstico/dominio.
- `S14-uat.md` con el script de UAT.
