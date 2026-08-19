# S06 — Brief 5: Extracción de tablas por tipo

> **Tipo:** extracción estructural en el parser (`corpus/`, descongelado). Ancla contenido tabular a
> la cláusula contenedora ya resuelta (Brief 4). Es el trabajo **más ad-hoc** del módulo — sin máster
> detrás; el mapa tipo→mecanismo de abajo es decisión de Claude Chat sobre el diagnóstico, no teoría.
>
> **Precedido de:** `S06-diagnostico-5-tablas.md` (1.079 tablas caracterizadas). Este brief actúa
> sobre esa medición, no re-explora.
>
> **Principio rector (el de toda la sesión):** ante la duda, **no fabricar**. Un hueco honesto (marca
> «hay contenido aquí que no transcribo, consúltese la cláusula X») vale más que una linealización
> infiel. Ninguna tabla se aplana si aplanar puede alterar lo que el documento dice.
>
> **Frontera CC:** CC escribe la extracción + tests con DOCX sintético. **Ejecutar sobre los 26 reales
> y verificar el contenido extraído es UAT del usuario.**

---

## Contexto: por qué esto ancla, no cuelga en el aire

El diagnóstico validó que **1.055 de 1.079 tablas cuelgan de una cláusula con identidad correcta**
(las 24 huérfanas son change-logs de portada, sin contenido citable). La reparación de numeración del
Brief 4 era el prerequisito real: cada tabla hereda el `clause_id` —y la `authority`— de su sección
contenedora, ya resueltos. Este brief extrae el *contenido* de la tabla y lo ancla a esa identidad;
no inventa identidad nueva.

**La autoridad de una tabla es la de su sección, no la de su documento** (diagnóstico §3): 253 tablas
informativas viven dentro de estándares (anexos `(informative)`), y `C.1.1` es normativa dentro de un
handbook. El `Section.authority` del Brief 4 ya lo resuelve por anexo. La extracción **hereda** ese
valor, no lo recalcula por tipo de documento.

---

## Paso 0 — Conteo de las 600 combinadas por clase de combinación

Antes de extraer, clasificar las 600 tablas con celdas combinadas por una **prueba objetiva sobre el
XML** (no juicio), porque decide cuánto se recupera vs cuánto se marca:

- **`gridSpan` horizontal simple** (celda que abarca columnas — cabeceras agrupadas): la relación se
  expresa repitiendo contexto. → **aplanable.**
- **`vMerge` de agrupación** (columna que agrupa N filas — el `Annex A` de `E-HB-40A`, 69 vMerge): la
  relación es 1-a-N, se despliega repitiendo el grupo en cada fila hija. → **aplanable.**
- **Anidamiento** (celda que *contiene* estructura — el `List of → Value` de §4, una celda con tabla/
  jerarquía dentro): no hay orden lineal que preserve la contención sin inventar notación. →
  **irreducible, marcar-sin-extraer.**

El criterio es *«¿la relación es desplegable a texto sin inventar notación?»* — 1-a-N se despliega,
contención no. Es una prueba sobre el XML (`gridSpan`/`vMerge` vs celda-contenedora), aplicable sin
interpretación. **Reportar el reparto** (cuántas de cada clase) — dimensiona el brief y es un dato
para el manifiesto.

---

## El mapa tipo → mecanismo (decisión de Claude Chat, cerrada)

Cada tabla se enruta a **uno** de cuatro mecanismos. El tipo se determina por señales objetivas del
diagnóstico (dimensiones, combinación, densidad de texto, celdas vacías), no por juicio.

### Mecanismo A — Aplanar a texto (dentro del chunk de su sección)

**Para:** clave-valor y campos simples sin combinación problemática (patrones 1, 6 del diagnóstico:
códigos, glosarios, definiciones de campo de una columna), **más** las combinadas clase gridSpan-simple
y vMerge-agrupación del paso 0.

**Cómo:** linealizar a texto fiel. Clave-valor → `Cabecera: valor. Cabecera: valor.` vMerge-agrupación
→ desplegar el grupo en cada fila hija (`Grupo G — campo X: …` repetido). La cabecera declarada
(`tblHeader`, presente en 335 tablas) da los nombres de campo; sin cabecera, usar posición.

**Fidelidad:** copia del texto de celda **sin reescribir** — mismo criterio que la extracción de
requisitos (`accepted_text`, conserva `w:ins`, descarta `w:del`). No se resume, no se parafrasea, no
se «mejora». El texto tabular es tan literal como el de un requisito.

### Mecanismo B — Marcar sin extraer (con traza)

**Para:** anidamiento irreducible (paso 0), diagramas de estructura donde el layout *es* el contenido
(patrón 5: `7.2.1 Structure diagram`), y cualquier tabla cuya linealización fiel no esté garantizada.

**Cómo:** la sección conserva su identidad y su prosa; la tabla **no** se transcribe. Se emite una
**marca en la unidad de contenido**: `content_extractable: false` (o campo análogo) + una nota mínima
de qué hay (`"structured table, N×M, not transcribed"`). El contenido citable de esa cláusula es su
prosa + la marca, no la tabla.

**Por qué es un tercer resultado, ni cita ni rechazo:** produce, aguas abajo (S11), una respuesta del
tipo *«la cláusula X define esto mediante una estructura que no se reproduce aquí; consúltese el
documento en esa cláusula»*. Es honesto por construcción: la cláusula **existe** (no es rechazo) y se
da la coordenada exacta, pero no se transcribe (no es cita normal). En trazabilidad, «existe, aquí
está dónde, no puedo mostrarlo fielmente» es correcto; una linealización infiel sería mentir sobre qué
dice la estructura. **La marca nace aquí (extracción); su tratamiento en la respuesta es S11.**

### Mecanismo C — Descartar (con conteo, nunca silencioso)

**Para:** ruido real sin contenido (patrones 7, 8, 9: cajas vacías 1×1 contenedoras de figura,
plantillas en blanco, columnas de relleno tipo `Verified`).

**Cómo:** no entra al corpus. **Pero se cuenta**: por documento, cuántas tablas descartadas y por qué
(`empty_cell`, `blank_template`, `filler_column`). El conteo va al manifiesto (Brief 6). Descartar es
legítimo; descartar **en silencio** es el pecado — la traza es lo que distingue «ruido eliminado» de
«contenido perdido».

**Ojo con `layout_candidate` (§8 del diagnóstico):** mezcla cajas vacías 1×1 (descartar) con
definiciones de campo de una columna (contenido normativo — Mecanismo A). El criterio que las separa
está en el output y es objetivo: `chars/celda == 0` → descartar; `chars/celda > 0` → es contenido, va
a A. **No descartar el grupo en bloque** — tiraría contenido normativo.

### Mecanismo D — Extraer como referencias (no como contenido)

**Para:** las tablas de `2 Normative references` (diagnóstico §7 — presentes en los tres documentos
inspeccionados, probablemente en todos).

**Cómo:** son pares `doc_referenciado → título`, no respuesta a ninguna pregunta del ingeniero. **No
entran como contenido citable** (nadie pregunta «¿cuáles son las referencias normativas de X?»
esperando la tabla). Se extraen a un **artefacto aparte** — una lista de referencias por documento —
y se **mide cuántas son**. Ese artefacto es input para la decisión grafo-vs-metadata de S07: parte del
49% de referencias del grafo sin `target_clause_id` puede estar aquí, en referencias que nunca se
extrajeron por vivir en una tabla. **El Brief 5 las saca de la invisibilidad y las cuenta; S07 decide
qué hacer con ellas.** No se tocan el grafo ni `references.json` aquí.

---

## Autoridad y el caso «shall» en handbooks (diagnóstico §6)

Las 30 tablas de prosa incluyen cláusulas contractuales modelo en handbooks que dicen «shall»
(`E-HB-40A 4.2.6.3`). Son **informativas** (su sección lo es) aunque el texto suene obligatorio.

Este brief **no** resuelve cómo se presentan (eso es S11) — pero **garantiza que llevan su `authority`
correcta** (`informative`, heredada de la sección). El peligro —presentar una sugerencia de redacción
como norma— lo neutraliza S11 usando ese `authority` para enmarcarlas como recomendación, no como
requisito. Aquí: que el dato viaje correcto. La extracción no altera el texto (el «shall» se conserva
literal); solo asegura que la etiqueta de autoridad que lo acompaña es fiel.

**`C.1.1` (anexo normativo en handbook):** ya resuelto en el Brief 4 — la cabecera `(normative)` del
anexo sobrescribe el default informativo del documento, porque lo que el documento **declara** sobre sí
mismo es más autoritativo que lo que nosotros **inferimos** de su tipo. La extracción hereda ese
`authority` sin recalcularlo. No se revisa el documento a mano: el mecanismo es correcto por principio.

---

## Qué entrega CC

1. **Extracción de tablas en el parser** — enruta cada tabla a A/B/C/D por señales objetivas; el
   contenido extraído (A) o la marca (B) se ancla al `clause_id` + `authority` de su sección.
   Descarte (C) y referencias (D) con sus artefactos/conteos. El conteo del paso 0 incluido.
2. **La unidad de tabla:** el contenido de una tabla aplanada (A) vive **dentro** de la unidad de su
   sección (no como chunk independiente) — la sección es la unidad citable, la tabla es parte de su
   contenido. *(La decisión fina de chunking es S07; aquí la tabla se ancla a la sección, que es la
   forma que no prejuzga el chunking.)*
3. **El flag `content_extractable`** (o análogo) para el Mecanismo B, en la unidad de contenido.
4. **Artefacto de referencias** (Mecanismo D) — lista por documento, con conteo.
5. **Conteos al manifiesto** (Brief 6): tablas por mecanismo por documento (aplanadas / marcadas /
   descartadas-con-motivo / referencias). Es el registro de cuarentena que el manifiesto formaliza.
6. **Tests** (DOCX sintético, sin `data/`): una clave-valor → aplanada fiel; una vMerge-agrupación →
   grupo desplegado en cada fila; una anidada → marcada, no aplanada; una caja 1×1 vacía → descartada;
   una de una columna con texto → NO descartada (va a A); una tabla de referencias → al artefacto D,
   no a contenido; herencia de `authority` de la sección (normativa e informativa). `uv run pytest`
   verde.
7. **Actualización de `verify_parser.py`:** cobertura de tablas (cuántas extraídas/marcadas/descartadas
   vs cuántas hay) como salida — el equivalente-tablas de la cobertura de identidad del Brief 4.
8. **Guion UAT** — el usuario ejecuta sobre los 26 y verifica: el contenido de `E-ST-70-31C` (el
   estándar cuyo contenido *es* la tabla de diccionario de datos) deja de estar vacío; una tabla marcada
   (B) produce marca, no texto inventado; los conteos de descarte cuadran; el artefacto de referencias
   existe con su medida. Re-parsear el corpus tras la extracción (el `parse_all` del Brief 4b ahora
   incluye tablas).
9. **`S06-decisions.md`** (append), etiquetado agnóstico/dominio.

## Qué NO hace CC

- No resuelve la presentación en respuesta del contenido marcado ni de los handbooks-«shall» (S11).
- No toca el grafo ni `references.json` — las referencias (D) van a un artefacto aparte, medido, para
  S07.
- No decide la estrategia de chunking (S07) — ancla la tabla a la sección, forma que no la prejuzga.
- No reescribe, resume ni «mejora» ningún texto de celda — copia fiel, como los requisitos.
- No construye el manifiesto (Brief 6) — le entrega conteos.
- No ejecuta sobre `data/` real (UAT del usuario).

## Criterio de "hecho"

- Cada tabla enrutada a A/B/C/D por prueba objetiva; el reparto del paso 0 (600 combinadas por clase)
  reportado.
- Contenido aplanado (A) fiel al texto de celda, anclado a `clause_id` + `authority` de su sección;
  `E-ST-70-31C` recupera su sustancia tabular.
- Anidamiento e irreducibles (B) marcados, nunca linealizados infielmente.
- Descartes (C) contados con motivo; `layout_candidate` separado por `chars/celda`, no en bloque.
- Referencias (D) en artefacto aparte, medidas, sin entrar como contenido ni tocar el grafo.
- `authority` heredada correctamente (los «shall» de handbook salen informativos).
- `verify_parser` reporta cobertura de tablas; conteos entregados para el manifiesto.
- Tests verdes; guion UAT entregado con la verificación de `E-ST-70-31C` y de una tabla marcada.
