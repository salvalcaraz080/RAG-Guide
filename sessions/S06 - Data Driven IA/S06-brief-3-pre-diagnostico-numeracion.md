# S06 — Brief 3-pre: Diagnóstico de numeración de sección (XML, no render)

> **Tipo:** diagnóstico / instrumento de medición. **No** es código de producción, **no** cambia el
> parser. Precede al Brief 3 (tablas) y al Brief 5 (numeración + identidad de cita) porque su
> resultado decide el mecanismo de ambos.
>
> **Frontera CC:** CC escribe el script de diagnóstico + tests sobre XML sintético (sin `data/`, sin
> red). **Ejecutar sobre los 26 DOCX reales de `parse_ready/` es UAT del usuario**; CC entrega el
> script + guion, el usuario lo corre y sube el output.

---

## Por qué existe este diagnóstico

Hallazgo verificado a mano por el usuario, sobre tres documentos de perfiles distintos
(`E-ST-40C` nativo, `Q-ST-30-02C` convertido con `INV4`, un handbook):

- La numeración de sección impresa (`8.6.2 In case 1:`) **no está en el texto del documento**. Al
  copiar el heading solo sale `In case 1:`; el `8.6.2` lo pinta Word desde su **numeración
  automática de lista multinivel** (`numId`/`ilvl` → `numbering.xml`), resuelta en render.
- Por eso el parser **cuenta** la numeración (su única señal sin resolver la lista de Word), y su
  contador se desincroniza en ciertos puntos → produce `E.0.1` donde Word pinta otro número. Los
  `INV4` son ese síntoma.
- La numeración parece **uniforme** en los tres documentos (siempre lista automática de Word). **Pero
  esa observación es visual, sobre el render.** Que Word pinte consistente NO garantiza que el XML
  subyacente sea uniforme: `8.6.2` idéntico al ojo puede venir de mecanismos distintos (numeración
  ligada a estilo, lista explícita, `startOverride`, listas que reinician). El render oculta esa
  diversidad.

**Este diagnóstico verifica la uniformidad en el XML, no en el render** — antes de comprometer el
Brief 5 a «resolver la numeración desde `numbering.xml`». Si el XML resulta ser tres mecanismos
distintos por debajo, el Brief 5 cambia de forma; mejor saberlo ahora (barato) que a mitad de
implementación (caro). Es el mismo patrón que destapó `soffice.com` en el Brief 2: la propiedad que
importa solo se ve ejecutando/inspeccionando la fuente real, no razonando sobre ella.

---

## Qué mide el diagnóstico

El script recorre los 26 `.docx` de `parse_ready/` y, **sin modificar nada**, reporta:

### 1. Mecanismo de numeración por documento

Para cada documento, cómo está estructurada la numeración de sus headings de sección en el XML:

- ¿Los headings numerados usan **numeración automática de lista** (`w:numPr` con `numId`/`ilvl` en el
  párrafo o heredado del estilo)? ¿O alguno lleva el número **literal en el texto** (`run` con el
  `8.6.2` como caracteres)?
- Si es lista automática: ¿el vínculo es **directo** (el párrafo tiene `w:numPr`) o **ligado a
  estilo** (el estilo `Heading N` define `numId`, el párrafo lo hereda)? ¿Aparecen ambos patrones en
  el mismo documento?
- ¿Hay `w:startOverride`, `w:lvlOverride`, o listas que reinician (`w:start`) — los mecanismos que
  hacen la resolución no-trivial?
- **Clasificación por documento:** `uniforme-directo` / `uniforme-ligado-a-estilo` / `mixto` /
  `numeración-en-texto` / `sin-numeración` (handbooks de rótulos como `E-HB-40A`).

El objetivo de esta parte: responder **¿un solo mecanismo de resolución sirve para los 26, o hay
familias que necesitan tratamiento distinto?** Ese es el dato que dimensiona el Brief 5.

### 2. Divergencia contador-vs-XML (el dato de oro)

Para cada heading de sección, comparar:
- el número que el **parser cuenta hoy** (su `section_number` derivado), contra
- el número que **se resuelve desde el XML de numeración** (`numId`/`ilvl` → nivel de lista →
  número).

Y reportar **dónde divergen**. Esto mide lo que de verdad importa: **cuántas cláusulas tienen
numeración fabricada incorrecta**, no solo las que disparan `INV4` ruidosamente. La hipótesis a
confirmar o refutar:

- Los `INV4` conocidos (`Q-ST-30-02C`, `E-HB-40A`) deben aparecer como divergencias — es su causa.
- **Lo que buscamos es la versión silenciosa:** un heading donde el contador produce un número
  *plausible pero incorrecto* (`E.3.1` donde el XML dice `E.2.1`), que NO dispara `INV4` (no hay
  ceros) y hoy pasa como cita fabricada sin traza. Si existe, es más peligroso que el `INV4` ruidoso
  — es la cita fantasma que no se ve.

Reportar: por documento, nº de headings, nº que coinciden, nº que divergen, y una muestra de las
divergencias (número-contado vs número-XML vs texto del heading) para inspección manual.

### 3. El caso de los rótulos sin número (patrón `E-HB-40A`)

Confirmado por el usuario: `E-HB-40A` usa `Heading 6` como **rótulos** (`Overview`, `Approach`) sin
numeración, subordinados al heading numerado inmediatamente superior. Estos **no deben recibir
número** (no son cláusulas citables; el título viaja como contexto, patrón DRD ya existente).

El diagnóstico cuenta, **en todo el corpus** (no solo en los documentos con `INV4`): headings cuyo
párrafo **no tiene numeración de lista** (`w:numPr` ausente y no heredado) pero que el parser hoy
**sí numera** (les cuenta un `section_number`). Esos son los rótulos, y todo número que el parser
les ponga es fabricado. Buscamos la versión silenciosa: un rótulo bajo una jerarquía sana recibe un
número inventado sin disparar `INV4`. Reportar cuántos hay y en qué documentos.

---

## Qué NO hace este diagnóstico

- No modifica el parser, ni la numeración, ni nada en `data/`.
- No decide el mecanismo de resolución (eso es el Brief 5, con este informe delante).
- No toca tablas (Brief 3), ni `Citation`, ni la canonicalización de revisión.
- No implementa la resolución desde `numbering.xml` como código de producción — solo la calcula *para
  comparar* contra el contador. (La versión de producción, ya validada, es Brief 5.)

## Qué entrega CC

1. **Script de diagnóstico** (`corpus/diagnose_numbering.py` o nombre análogo a `diagnose_doc.py`) —
   recorre `parse_ready/`, produce el informe de las tres partes. Sin efectos secundarios sobre datos.
2. **Tests** sobre XML sintético (mockeado, sin `data/`): un párrafo con `w:numPr` directo → detectado
   como lista automática; uno con numeración ligada a estilo → detectado; uno con número en texto →
   detectado; un heading sin `numPr` que el contador numeraría → detectado como rótulo; un caso de
   divergencia contador-vs-XML construido a mano → reportado. `uv run pytest` en verde.
3. **Guion UAT** — el usuario ejecuta el script sobre los 26 reales y sube el output. El guion pide
   explícitamente: la tabla de clasificación por documento (parte 1), el recuento de divergencias
   (parte 2) con al menos una muestra por documento divergente, y el recuento de rótulos (parte 3).
4. **`S06-decisions.md`** (append) con las decisiones de implementación del diagnóstico.

## Criterio de "hecho"

- El script clasifica los 26 documentos por mecanismo de numeración (parte 1).
- Reporta la divergencia contador-vs-XML por documento, incluyendo (si existe) la versión silenciosa
  —divergencia sin `INV4`— que es el hallazgo que más importa (parte 2).
- Cuenta los rótulos-sin-número que el parser numeraría, en todo el corpus (parte 3).
- Los tests pasan sobre XML sintético, sin datos reales.
- El output real (ejecutado por el usuario) está subido, listo para que Claude Chat decida el
  mecanismo del Brief 5 con datos y no con la hipótesis de uniformidad-visual.

---

## Qué decidiremos con este output (para contexto, no es trabajo de CC)

- **Si el XML es uniforme** (un mecanismo domina): el Brief 5 resuelve la numeración desde
  `numbering.xml` para todos → `clause_id` fiel por construcción.
- **Si es mixto** (familias distintas): el Brief 5 resuelve las familias tratables y define un
  **fallback** para las intratables — probablemente identidad por UID / `subsection_path` (el camino
  que el proyecto ya tomó para los DRD), no por número fabricado.
- **La magnitud de la divergencia silenciosa** decide la urgencia: si son cuatro cláusulas en dos
  documentos, es acotado; si es sistemático, la numeración-por-conteo estaba produciendo citas
  fantasma a escala y el Brief 5 sube de prioridad sobre el Brief 3.
