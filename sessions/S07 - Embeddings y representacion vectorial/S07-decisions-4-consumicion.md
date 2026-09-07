# S07-decisions-4-consumicion.md — Implementación (Claude Code)

Decisiones y fricciones al implementar `S07-brief-04-consumicion-chunking.md`.
Etiquetadas **[agnóstico]** (candidata a Guide) vs **[dominio]** (solo ECSS).

---

## Resultado: el corpus entra al índice, sin fugas

```
13.888 chunks · 13.888 ids únicos · 0 colisiones
por ancla:      requirement 7.922 · section 2.592 · table 1.481 · note 1.411 · glossary_term 482
por autoridad:  normative 10.331 · informative 3.557

chars de origen en processed/   5.244.208
chars contabilizados            5.244.208
FUGA                                    0
```

El chunk anterior del brief 1 eran 9.403 chunks de requisitos y tablas. Ahora entran los
**1,69M chars de prosa de sección**, los **436.921 de NOTE** y los **99.501 de glosario** que
llevaban desde S06 en disco y fuera del índice.

`input_fingerprint`: `7119bdf34b60029c`.

---

## El brief da por hecho algo que no es cierto, y seguirlo habría costado 1,46M chars

> «Las tablas aplanadas ya viven dentro de `Section.body` como parte del contenido, así que
> dejan de ser un camino aparte.»

**No es así.** El parser guarda las tablas en `ParsedDocument.tables`, no en `Section.body`: en
el DOCX son hijas del *body* del documento, no párrafos, y el brief 2 solo enrutó párrafos.
Verificado antes de tocar nada: en `ECSS-E-ST-40C`, `Section.body` son 109.287 chars y las
tablas aplanadas otros 105.686, disjuntos.

- **[dominio] Decisión:** se mantiene el camino de tablas, con **ancla propia `table`** en vez
  de plegarlo dentro de `section`. Eliminarlo como pedía el brief habría sacado **1.460.887
  chars** del índice — y lo habría hecho en silencio, salvo por la comprobación de fuga que el
  propio brief pide, que lo habría cazado. Que dos mecanismos del mismo brief se contradigan es
  la señal de que la comprobación es la que manda.
- **[agnóstico]** El ancla separada no es cosmética: la procedencia de un chunk de tabla es
  distinta de la de un párrafo (una linealización fiel de una rejilla vs. prosa escrita), y el
  retrieval tendrá que poder distinguirlas. Plegarlas habría hecho invisible esa diferencia.

---

## El corpus repite números de cláusula dentro del mismo documento

La primera corrida dio **1 id colisionado**: `ECSS-Q-HB-80-02 Part 2A#4.1.1::s0`, dos veces. La
causa no es el chunker sino el corpus: ese documento tiene **dos secciones numeradas `4.1.1`**,
una titulada «Introduction» y otra «Process definitions». Hay **4 números repetidos en 3
handbooks** (`Q-HB-80-01A` ×2, `Q-HB-80-02 Part 1A`, `Part 2A`).

- **[dominio] Arreglado en el chunker:** los ordinales se llevan por **número de cláusula**, no
  por objeto sección, así que dos secciones que comparten número siguen produciendo ids
  distintos. 0 colisiones sobre los 13.888.
- **[dominio] Pero el hallazgo de fondo no es de este brief y hay que subirlo:** si dos
  cláusulas distintas comparten número, **el número de cláusula no identifica dentro de esos
  documentos**, y eso es identidad de CITA, no de chunk. Una respuesta que cite
  «`ECSS-Q-HB-80-02 Part 2A` 4.1.1» es ambigua: hay dos. Son handbooks (informativos) y son 4
  casos, así que no es urgente — pero es el mismo tipo de problema que S06 resolvió para la
  numeración, un piso más abajo.

---

## Decisiones de implementación

- **[agnóstico] La coordenada de cita de un sub-chunk es la sección; el texto, el sub-chunk.**
  Resuelve el `# TODO` del brief 1. Un sub-corte no tiene identidad de cláusula y fabricársela
  sería inventar identidad; varios sub-chunks comparten coordenada y es el texto verbatim el
  que dice cuál fundamentó la respuesta.
- **[agnóstico] El solapamiento es de sección, nunca de requisito.** 12% de la ventana, y solo
  entre sub-chunks de la MISMA sección. En prosa continua un corte cae a mitad de argumento; en
  un requisito arrastraría cola del vecino y contaminaría la cita.
- **[dominio] Las NOTE consecutivas se agrupan en un chunk, no uno por párrafo.** Una nota con
  viñetas es una nota, no cinco.
- **[dominio] El glosario tiene precedencia sobre la regla de NOTE**, como pedía el brief: la
  NOTE de una definición es su procedencia («Adapted from ISO/IEC 2382-20:1990») y separarla
  dejaría una cita de origen huérfana de lo que origina.
- **[dominio] El requisito sigue heredando autoridad de su sección**, no forzada a normativa.
  El brief decía «chunk de requisito: normativo». Lo dejé como en el brief 1 a propósito: un
  requisito en un anexo informativo no es normativo por ser requisito, y forzarlo desharía lo
  que el brief 1 fijó con test. Sobre el corpus real el efecto es nulo salvo en anexos
  informativos, que es justo donde importa.
- **[agnóstico] La cobertura se mide sobre los chars de ORIGEN, no sobre lo emitido.** El
  solapamiento duplica caracteres: un conteo del lado de salida daría «más contenido del que
  había» y sería inservible como detector de fugas. Primer intento midió lo ensamblado y falló
  por los 18 chars del delimitador `Expected output:`, que es sintético.

---

## Task 6 — el fingerprint, y por qué no es un número de versión

`corpus_version` se queda como **etiqueta legible del conjunto declarado** (va en la cita y en
el manifiesto). La invalidación se ancla a `input_fingerprint`: un digest sobre los documentos
que el chunker realmente consume.

- **[agnóstico] Cada stage fingerprintea su propio input.** Como el input de un stage es el
  output del anterior, un cambio de lógica aguas arriba se propaga como input cambiado aguas
  abajo, **sin bump manual**. Una versión que hay que subir a mano solo es cierta por
  disciplina, y los briefs 2 y 3 demostraron exactamente eso: cambiaron el output del parser
  tres veces bajo la misma `2026-08-04.91bc8116`.
- **[agnóstico] Se hashean los ficheros almacenados, no una proyección re-derivada.** Una
  proyección habría que mantenerla sincronizada con lo que el chunker lee, y un fingerprint que
  deja de cubrir un campo en silencio es peor que uno conservador. Este avisa de más, nunca de
  menos.
- `is_stale()` devuelve `True`/`False`/`None`: **None cuando no se puede saber** (no hay
  artefacto, o no lleva fingerprint). Un «no lo sé» no debe disfrazarse de «está bien».

---

## Task 0 — el gate de `Subtitle` pasa limpio

`document_subject` presente en **26/26**, y en los 26 es materia real: `Software`,
`Verification`, `Testing`, `Safety`, `Risk management`, `Quality assurance`… Ningún valor vacío
ni basura, así que el fallback al doc_id canónico está implementado y probado pero no se activa
en este corpus.

El header queda así:

```
[Standard: ECSS-E-ST-40C — Software | Section: Requirements > Software management process
 > Software project reviews description > Critical design review]
```

---

## Pendiente de decisión en Claude Chat

1. **`Definition1` no se enruta como heading** (S07 brief 4, UAT): cada término del glosario
   es una cláusula numerada (`3.2.1`, `3.2.2`…) y hoy 482 términos citan la sección
   contenedora. Mismo defecto que `Annex4/5`, otra familia de estilo.
2. **Un párrafo borrado consume número de lista**: 137 casos en 8 documentos. Hoy benigno para
   las secciones (los 12 headings afectados están en anexos que el documento marca
   `<<deleted>>`), pero bloquea el punto 1: arreglar uno sin el otro daría números falsos.
3. **`AIM` desacoplado de su requisito** (24 párrafos en 11 secciones de `ECSS-E-ST-40C`,
   4 de ellas con varios que se funden): ¿del requisito, y viaja con él como EXPECTED
   OUTPUT, o de la cláusula, y hay que separarlos entre sí?
4. **Números de cláusula repetidos** en 3 handbooks (4 casos). El número no identifica ahí, y
   eso es identidad de cita.
5. **Ancla `table` separada de `section`** — la implementé así por procedencia distinta; si Chat
   prefiere plegarlas, es un cambio de una línea, pero perdería la distinción en retrieval.
6. **Autoridad del chunk de requisito**: heredada de la sección (como brief 1) frente a forzada
   a normativa (como pedía este brief). Mantuve la herencia.
7. Sigue abierto del brief 3: `require` desnudo, contenido de cuadros de texto, marcadores
   literales de lista.

---

## UAT — resultado, con un defecto de identidad confirmado

El usuario revisó las cinco comprobaciones. Resumen: **2 y 3 pasan, 5 pasa, 1 es una lectura
del contrato del chunker, y 2 destapa un defecto real** de la misma clase que el `Annex4/5` del
brief 3.

### ✅ Pasan: NOTE informativa, EXPECTED OUTPUT y cobertura

Las NOTE de cláusula normativa salen bien clasificadas como chunk informativo propio con su
coordenada. Los requisitos con `Expected output:` traen contenido y delimitación correctos.
El reporte da `leaked: 0` y `colliding_ids: 0`.

### 1 · ✅ Pasa tras verificación directa, y destapa un hallazgo

El usuario observó que el chunk de sección de `ECSS-E-ST-40C#5.3.4.3` solo contiene líneas
`AIM:` y una `NOTE`, y lo contrastó con un texto que resultó ser el de **5.3.4.4** (error de
copia, corregido después por él). Eso invalidó la comparación, no el chunk.

**Nota de método:** CC dio la prueba por cerrada al confirmarse que `Qualification review` es
5.3.4.4. De ahí solo se seguía que el texto citado no era el de 5.3.4.3 — **no** que el chunk de
5.3.4.3 fuese correcto, que era lo que la prueba pedía comprobar y que nadie había mirado
todavía. Resolver la discrepancia no es lo mismo que pasar la prueba, y confundirlo cerró un
control de identidad de cita sin ejecutarlo.

**Verificado ya de verdad**, volcando el tramo del DOCX entre los headings 5.3.4.3 y 5.3.4.4:

```
Heading 4  Critical design review
  UID 0860039 → require   "After completion of the design of software items…"
                aim       "AIM: —To review the design definition file…"
                aim:bul   "— To review the design justification file…"
                EXPECTED OUTPUT  "Approved design definition file… [DDF, DJF; CDR]."
  UID 0860040 → require   "In case the software detailed design is baselined…"
                aim       "AIM: To baseline the software detailed design…"
                EXPECTED OUTPUT  "Approved detailed design… [DDF, DJF; CDR]."
                NOTE      "e.g. in case of complex software products…"
```

Todo el texto llega a algún chunk y cada pieza al ancla que le toca: dos chunks de requisito con
sus EXPECTED OUTPUT dentro, la NOTE como chunk informativo propio, y los `aim` al chunk de
sección. La cláusula 5.3.4.3 se parsea completa y correctamente.

**Hallazgo — los `aim` se desacoplan de su requisito, y entre sí.** Lo que le chirriaba al
usuario era real, aunque la cláusula que citó fuera otra: el chunk de sección concatena el AIM
del requisito `0860039` con el del `0860040`, y ahí se pierde a cuál pertenece cada uno.

- **[dominio]** Es el mismo patrón que `EXPECTED OUTPUT` antes del brief 2: un párrafo que
  **cualifica a un requisito** y que, tratado como prosa de sección, se separa de él. EXPECTED
  OUTPUT se ató al requisito; `aim` sigue suelto, y cuando hay más de uno en la cláusula además
  se funden.
- **Medido: 24 párrafos `aim*` en 11 secciones, todos en `ECSS-E-ST-40C`**, con **4 secciones
  donde se juntan varios**. Pequeño, pero en el estándar CORE.
- **[agnóstico] La decisión no es de CC:** si `AIM` es del requisito, viaja con él como
  EXPECTED OUTPUT; si es de la cláusula, se queda donde está pero hay que separar los AIM entre
  sí para no fundir dos. Va a Chat.

**Y una observación de diseño que la prueba deja a la vista:** una cláusula queda repartida
entre chunks de anclas distintas y **ninguno contiene la cláusula entera**. Quien recupere el
chunk de sección de 5.3.4.3 ve los AIM sin los requisitos. Es coherente con «un requisito es un
átomo citable», pero si el retrieval debe recomponer la cláusula es decisión de M2.

### 2 · ❌ Defecto confirmado: cada término del glosario es una cláusula numerada

El usuario tiene razón. `3.2` es un rótulo de nivel 2 («Terms specific to the present
standard») y **cada término es una subcláusula numerada**: `3.2.1 acceptance test`,
`3.2.2 automatic code generation`, … El chunker los ancla todos a `3.2`.

Comprobado en el XML: los párrafos `Definition1` **llevan numeración de lista propia** y el
resolver ya calcula su número (`1`, `2`, `3`, …). Simplemente no se enrutan como heading, así
que no llegan a ser `Section`.

- **[dominio] Es exactamente el defecto de `Annex4`/`Annex5` del brief 3**, en otra familia de
  estilo: un estilo estructural que cae a la rama de prosa, y la cita sale más gruesa que el
  documento. **482 términos de glosario** citan hoy la sección contenedora en vez de su propia
  cláusula.
- **[dominio] Y arreglarlo a secas daría números FALSOS**, que es peor. Ver el punto 3.

### 3 · ❌ Defecto nuevo: un párrafo borrado sigue consumiendo número de lista

`ListResolver.advance()` corre sobre **todos** los párrafos, incluidos los que el control de
cambios borró por completo. En el documento aceptado un ítem borrado no consume número; en
nuestro resolver sí.

**Medido: 137 párrafos borrados consumen número en 8 documentos.** Por estilo:
`Definition1` 41 · `require:level2` 41 · `list:level2` 12 · `DRD1` 11 · `list:level1` 10 ·
`Annex3` 8 · `require:level1` 7 · `Annex2` 4 · otros 3.

- **Hoy es benigno para las secciones:** los 12 headings afectados están todos en
  `ECSS-E-ST-10C Rev.1`, dentro de los anexos M y Q, que el propio documento marca
  `<<deleted>>`. Consumen `M.1`, `Q.2.1`… dentro de un anexo muerto y no desplazan a ningún
  hermano vivo. Por eso S06 validó bien contra el render y por eso las 205 secciones del
  brief 3 cuadraron.
- **Deja de ser benigno en cuanto se enrute `Definition1`.** En `ECSS-E-ST-40C` los términos
  «competent assessor» (nº 4) y «mission products» (nº 18) están borrados y consumen número:
  el resolver da **6** a `configurable code` mientras el render de Word muestra **3.2.5**. Uno
  de más, por cada término borrado anterior.
- **[agnóstico] Los dos defectos hay que arreglarlos juntos o ninguno.** Enrutar `Definition1`
  sin descontar los borrados convertiría 482 citas gruesas-pero-correctas en 482
  precisas-e-incorrectas, que es el intercambio que este proyecto lleva tres briefs
  rechazando.

### 6 · Sobre volver al PDF

El usuario observa que el DOCX muestra doble numeración (`3.2.5[3.2.4]`) y contenido en azul
subrayado / naranja tachado, y pregunta si reabrir la decisión de usar PDF.

- **[dominio] Lo que se ve en el render NO es lo que extrae el parser.** `accepted_text` ya
  resuelve el control de cambios desde S06: conserva `w:ins`, descarta `w:del`/`w:delText`. El
  texto que llega a `data/processed/` es la versión aceptada, sin el tachado. Los `<<deleted>>`
  que se ven arriba son párrafos *vacíos tras aceptar*, y el parser los salta.
- **[dominio] El problema real de las revisiones es el otro**: no el texto, sino el **contador**
  de lista, que sí cuenta lo borrado. Es un defecto acotado (137 casos), localizado en una
  función, y arreglable.
- **[agnóstico] Cambiar a PDF costaría mucho más de lo que resuelve.** Todo lo que sostiene la
  identidad de cita vive en el XML del DOCX: los estilos (`require:level1` frente a `NOTE`
  frente a `Definition1`), `numbering.xml` (del que S06 resolvió la numeración tras demostrar
  que contarla fabricaba 469 cláusulas falsas), la autoridad por sección y la estructura de
  tablas. En PDF no hay nada de eso: hay cajas de texto con coordenadas, y distinguir un
  requisito de una nota vuelve a ser heurística sobre tipografía — justo lo que este pipeline
  lleva dos sesiones eliminando.
- **Lo que el PDF sí da** —y conviene no perderlo de vista— es que es el artefacto **publicado
  y aceptado**, con una numeración sin ambigüedad. Es el **oráculo de validación** ideal, y de
  hecho es el que el usuario acaba de usar para encontrar este defecto. Recomendación: seguir
  con DOCX como fuente y usar el PDF como referencia de validación, que es el reparto que ya
  está funcionando.