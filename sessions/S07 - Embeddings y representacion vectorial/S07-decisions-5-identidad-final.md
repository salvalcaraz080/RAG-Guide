# S07-decisions-5-identidad-final.md — Implementación (Claude Code)

Decisiones y fricciones al implementar `S07-brief-05-identidad-final.md`.
Etiquetadas **[agnóstico]** (candidata a Guide) vs **[dominio]** (solo ECSS).

---

## Resultado

```
secciones     5.356 → 5.836   (+480, exactamente los términos de glosario)
requisitos    7.922 → 7.922   (sin cambio)
chunks       13.888 → 13.876   · 0 colisiones · 0 fugas
  glossary_term 480 (anclados ya a su propia cláusula) · note 1.410 · section 2.583
AIM atados: 24 párrafos en 14 requisitos
input_fingerprint: 7119bdf34b60029c → ab3fce8b5cb1a0d4   (la staleness se detectó sola)
```

**La prueba que demuestra que las tareas 1 y 2 eran inseparables**, sobre el corpus real:

```
3.2.1 acceptance test · 3.2.2 automatic code generation · 3.2.3 code coverage
3.2.4 condition       · 3.2.5 configurable code
```

`configurable code` sale **3.2.5**, que es lo que imprime el documento. Antes del arreglo del
contador el resolver daba 6, porque el término borrado «competent assessor» consumía el 4.

---

## Lo que el XML decidió, y no una heurística

`Definition1` no comparte lista con los headings (`abstractNumId` 15 frente a 16), pero su
`lvlText` es **`'3.2.%1'`**: el `3.2.` va **literal en la definición de la lista**, así que Word
imprime `3.2.1 acceptance test` y el número impreso ES la cláusula. No hubo que juzgarlo.

- **[agnóstico] Un prefijo puramente numérico no es presentación, es la parte alta del número.**
  `_render` devolvía `prefix` aparte para no convertir `Annex B` en identidad `Annex B` en vez
  de `B`. Pero `3.2.` no es un rótulo: es el número. La regla nueva —prefijo de solo dígitos y
  puntos se pliega dentro del número— afecta **exclusivamente** a `Definition1`: se inventarió
  el corpus entero y los demás prefijos son `Annex`, `NOTE`, `<`, `(`, `EXPECTED OUTPUT:` y
  viñetas, todos presentación real.

---

## Tarea 1 — el contador y la semántica de documento aceptado

`ListResolver.advance()` recibe ahora `deleted: bool`. Un párrafo que el control de cambios
borró del todo **no consume número**.

- **[agnóstico] Es la misma semántica que `accepted_text`, aplicada al contador.** El texto ya
  seguía el documento aceptado desde S06; el contador seguía el documento con marcas. Dos
  semánticas distintas dentro del mismo parser es precisamente el tipo de incoherencia que
  produce números plausibles y falsos.
- **[dominio] Cuidado con no romper el caso vecino:** un párrafo **vacío** sí existe y sí
  consume número — S06 lo documentó como detalle no opcional del resolver. La condición es
  específicamente «sin texto tras aceptar **y** con `delText`», no «sin texto». Hay un test por
  cada uno de los dos casos, porque confundirlos rompe la numeración entera.
- Los 137 borrados numerados siguen ahí; ahora ninguno cuenta.

---

## Tarea 3 — `aim` atado, y por qué sin delimitador

`Requirement.aim`, paralelo a `expected_output`, con el mismo fallback a `body` si no hay
requisito abierto. El `RequirementChunker` lo incluye en el chunk del requisito.

- **[dominio] Sin encabezado sintético, al contrario que `EXPECTED OUTPUT`.** Los párrafos
  fuente ya abren con `AIM:`, que es el marcador que el propio documento escribe; añadir uno
  duplicaría lo que ya está. `EXPECTED OUTPUT` sí lo lleva porque su texto no se anuncia solo.
- Resuelve las dos mitades del defecto: el desacople del requisito **y** la fusión entre AIM de
  requisitos distintos.

---

## Tarea 0 — la auditoría cierra el patrón, y encuentra un caso más

`corpus/diagnostics/audit_structural_styles.py` enumera todo estilo cuyo párrafo **porta número
resuelto** y reporta si se enruta como heading. Es la generalización de haber encontrado
`Annex4/5` y `Definition1` de uno en uno por UAT.

Resultado sobre los 26 documentos: **37 estilos numerados, 11 enrutados**.

- **`DRD1/2/3` → `drd_marker`, no defecto.** Primera versión del veredicto los marcó como
  defecto porque imprimen `5.5.1`. Falso positivo mío: su `lvlText` es `<%6.%7>`, o sea el
  documento imprime `<5.5.1>` — un marcador de item DRD, y S06 decidió deliberadamente **no**
  numerarlos porque el marcador lo genera Word y no está en el texto. El veredicto distingue
  ahora por el prefijo `<`.
- **`Normal` → defecto real, 1 párrafo en 1 documento.** En `ECSS-E-ST-70-41C` hay un heading
  «Nomenclature» numerado `3.4` **con estilo `Normal`**. Es una cláusula que no llega a serlo.
  → `# TODO: pendiente de decisión en Claude Chat`. **No lo enruto**, y no por prudencia
  genérica: `Normal` es el estilo de prosa por defecto, así que enrutarlo por estilo convertiría
  en cláusula cualquier párrafo suelto que porte número. Es un defecto **del documento**, y
  arreglarlo pide una regla por documento o una excepción nombrada, no una entrada en la tabla
  de estilos.
- Los 23 `review` restantes son marcadores de lista (`a`, `1`, `(1)`) y captions: numeran ítems,
  no cláusulas. No se enrutan.

---

## Tarea 4 — los repetidos se declaran, no se desambiguan

`corpus/processing-report.md` gana la sección **Ambigüedad de cita conocida**, con los 4 casos
medidos (no hardcodeados): `Q-HB-80-01A` cláusulas `4` y `5`, `Q-HB-80-02 Part 1A` cláusula `4`,
`Part 2A` cláusula `4.1.1`.

- **[agnóstico] No se fabrica desambiguador.** Colgar el título de la coordenada haría que la
  cita difiera de lo que el documento imprime, que es identidad inventada. La ambigüedad es del
  documento fuente y se declara como tal; el breadcrumb del header y el verbatim la resuelven
  para quien lee.

---

## Fricción de proceso: perdí el delta de `diff_reparse`

El brief pedía correr `diff_reparse` **contra el parse previo** para ver el delta exacto
(482 términos re-anclados, 24 AIM movidos, cero cambios fuera de eso). Re-parseé antes de
capturarlo, así que esa comparación ya no se puede hacer: `data/processed/` quedó sobrescrito y
`diff_reparse` ahora da 0, que solo dice que el disco está al día con el parser.

- **[agnóstico] El error de método:** una herramienta que compara el estado en disco contra un
  parse fresco solo sirve **antes** de escribir. Debí correrla primero. Es la segunda vez esta
  sesión que cierro una verificación por el orden equivocado.
- **Evidencia equivalente, que sí tengo:** el crecimiento de secciones es **exactamente 480** y
  los términos de glosario son **exactamente 480** — es decir, lo único que se añadió fue el
  glosario, y ninguna otra sección apareció ni desapareció. Los requisitos siguen siendo 7.922.
  No sustituye a ver el delta enumerado, pero acota lo que pudo cambiar.
- **Queda para la UAT** lo que de verdad importa y que ninguna aritmética sustituye: que los 480
  números coincidan con el PDF.

---

## Pendiente de decisión en Claude Chat

1. **`Normal` numerado en `ECSS-E-ST-70-41C`** (1 caso, «Nomenclature» = 3.4). Cláusula que no
   se enruta, y que no se puede enrutar por estilo sin romper la prosa.
2. Sigue abierto del brief 4: ancla `table` separada, autoridad del chunk de requisito.

---

## La medición oportunista: el corpus no cita sub-ítems

El brief pedía, dentro de la Tarea 1, medir la frecuencia de referencias a sub-ítem
(`5.4.2.2(b)`) para informar si la deuda de marcadores literales puede seguir diferida.

```
referencias a cláusula ("clause N"):        1.455
referencias a SUB-ÍTEM (5.4.2.2(b)):            0
```

El regex encontró 5 candidatos y los cinco son **falsos positivos**: `para. 3.6.2 (mod)` en
`ECSS-E-ST-80C`, que son citas a CCSDS con una anotación entre paréntesis, no referencias a un
ítem de lista.

- **[dominio] Conclusión: la deuda de marcadores literales puede seguir diferida.** Nada en el
  corpus cita un sub-ítem por su marcador impreso, así que no reproducir `1.` / `(a)` / `(1)` no
  rompe ninguna referencia existente. La sangría del brief 3 —que sí conserva el anidamiento—
  cubre lo que hay.
- **[agnóstico]** Merece decirse cómo se llegó: el conteo bruto daba 5 y la conclusión honesta
  exigía mirarlos. Un «5 de 1.455, despreciable» habría sido la misma decisión por el motivo
  equivocado, y habría dejado en pie la idea de que el patrón existe aunque sea raro.