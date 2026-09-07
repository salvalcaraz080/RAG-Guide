# S07 · Brief 5 — Completar identidad del parser (antes de embeddings)

> **Para:** Claude Code. **Produce:** el cierre de los defectos de identidad de cita que la UAT
> del Brief 4 destapó — enrutar `Definition1` como cláusula (acoplado al arreglo del contador de
> lista sobre borrados), atar `aim*` a su requisito, auditar la familia "estructural-como-prosa",
> y declarar los números de cláusula repetidos. **NO produce:** embeddings, wrapper, vector store.
>
> **Contrato de trabajo:** CC no decide arquitectura; marca `# TODO: pendiente de decisión en
> Claude Chat` y sigue. Tests mockeados. La validación contra el render es UAT del usuario.
> Commits los hace el usuario.
>
> **⚠️ Este brief toca `corpus/numbering.py`, el módulo que S06 validó contra el render.** Es el
> punto más peligroso del pipeline. El control de riesgo es la validación exhaustiva contra el
> render (PDF como oráculo) + `diff_reparse` con `exit 1` ante cualquier cambio de identidad no
> previsto.

---

## 0. Contexto — por qué esto va antes de los embeddings

El Brief 4 dejó el corpus indexado (13.888 chunks) pero con defectos de identidad conocidos:

- **`Definition1` no se enruta como cláusula:** 482 términos de glosario citan la sección
  contenedora (`3.2`) en vez de su propia cláusula (`3.2.1 acceptance test`…). Es el defecto de
  `Annex4/5` del Brief 3, en otra familia de estilo.
- **`aim*` desacoplado de su requisito:** 24 párrafos en 11 secciones de `ECSS-E-ST-40C` (CORE),
  4 secciones donde varios AIM se **funden** y se pierde a qué requisito pertenece cada uno.
- **4 números de cláusula repetidos** en 3 handbooks (ambigüedad de cita).

Por qué **antes** de embeddings: por el principio que este proyecto lleva cinco briefs sosteniendo
—*una cita precisa-pero-incorrecta es peor que una gruesa-pero-correcta*— y porque el texto y la
coordenada de un chunk son lo que se embebe y se cita; cambiarlos tras generar vectores obliga a
re-embeber. Cerrar identidad ahora evita ese re-trabajo.

Este es el último brief del arco chunking+identidad. Tras su UAT, se consolida S07 (paso 5) y se
gradúa la Guide (paso 6), antes de abrir el movimiento de embeddings.

---

## Task 0 — Auditoría de estilos estructurales-como-prosa (diagnóstico, gatea Task 2)

`Annex4/5` (Brief 3) y `Definition1` (Brief 4) son el mismo defecto: un estilo **estructural**
(lleva numeración de cláusula propia) que cae a la rama de prosa y nunca llega a ser `Section`.
El detector de "sin clasificar" **no lo caza** porque el estilo *sí* está clasificado —como
prosa—; lo que está mal es el rol, no el reconocimiento. Es un punto ciego, y lo hemos descubierto
uno por UAT. Este brief cierra el patrón en vez de perseguirlo.

- **Entregable:** un diagnóstico read-only (`corpus/diagnostics/audit_structural_styles.py`) que
  enumera **todo estilo de párrafo que porta numeración de lista propia** (el resolver ya la
  calcula) y reporta, por estilo: ¿se enruta hoy como `Section`/heading, o cae a `body`? Un estilo
  numerado que cae a prosa es un candidato a defecto de identidad.
- **Gate:** el usuario lo corre en UAT y reporta la lista. Los casos **inequívocos** (numeración
  de cláusula, es una cláusula: `Definition1` es el caso claro) entran en Task 2. Cualquier caso
  **ambiguo** (porta número pero no está claro que sea cláusula citable) se marca
  `# TODO: pendiente de decisión en Claude Chat` — **no se enruta a ciegas**, que sería fabricar
  identidad. Mejor gruesa-pero-correcta que precisa-e-inventada.

---

## Task 1 — Descontar borrados en el contador de lista (`numbering.py`, render-validado)

**Prerrequisito de Task 2, y de mayor riesgo.** `ListResolver.advance()` corre sobre *todos* los
párrafos, incluidos los que el control de cambios borró por completo. En el documento aceptado un
ítem borrado **no consume número**; en nuestro resolver sí. Medido: **137 párrafos borrados
consumen número en 8 documentos**.

- **Arreglo:** `advance()` no cuenta un párrafo totalmente borrado (sin texto tras aceptar
  `w:del`). Consistente con `accepted_text`, que ya descarta el borrado — el contador debe seguir
  la misma semántica de documento-aceptado que el texto.
- **Por qué es peligroso y benigno a la vez hoy:** hoy los 12 *headings* afectados están en
  anexos que el documento marca `<<deleted>>` (`ECSS-E-ST-10C Rev.1`), anexos muertos que no
  desplazan hermanos vivos — por eso S06 validó bien y las 205 secciones del Brief 3 cuadraron. El
  arreglo **no debe romper eso**: `diff_reparse` debe mostrar que las 205 secciones del Brief 3
  siguen con sus números y que ninguna cláusula viva fuera del glosario cambia de número.
- **Efecto invisible fuera del glosario:** los borrados en `require:level2`/`list:level2` afectan
  a marcadores de lista *dentro* de requisitos, que hoy no se reproducen (deuda diferida). Ahí el
  arreglo corrige un contador que no se muestra — sin efecto visible, pero correcto para cuando se
  reproduzcan.
- **Oportunista:** ya que se está en `numbering.py`, medir de paso la **frecuencia de referencias
  a sub-ítem** (`5.4.2.2(b)`) en el corpus. Si es alta, informa si los marcadores literales dejan
  de poder diferirse. Solo medir y reportar; no implementar marcadores aquí.

---

## Task 2 — Enrutar `Definition1` (+ hallazgos claros de Task 0) como cláusula

Con Task 1 hecho (números correctos), enrutar `Definition1` como heading/cláusula, igual que
`Annex4/5`. Los párrafos `Definition1` ya llevan numeración propia; solo hay que enrutarlos para
que lleguen a ser `Section`.

- Los 482 términos de glosario pasan de citar `3.2` a citar su cláusula (`3.2.5 configurable
  code`). Es el arreglo: la cita se vuelve tan precisa como el documento.
- **Interacción con el agrupado de glosario del Brief 4:** hoy el `SectionChunker` agrupa
  `Definition1 + paragraph + NOTE` en un chunk de término anclado a `3.2`. Al volverse `Definition1`
  una cláusula propia, el término se ancla a su cláusula (`3.2.5`) y su definición+NOTE cuelgan de
  ella. CC verifica que el agrupado sigue produciendo **un chunk por término**, ahora con la
  coordenada correcta. Cita = identidad (verbatim), sin cambio.
- **Render-validado:** los 482 números resueltos coinciden con el PDF. En concreto, los términos
  tras un término borrado (p. ej. "configurable code" = `3.2.5`, no `3.2.6`) — el caso que
  demostró que Task 1 y Task 2 son inseparables.

---

## Task 3 — Atar `aim*` a su requisito (como EXPECTED OUTPUT)

Mismo patrón que EXPECTED OUTPUT en el Brief 2: `aim*` **cualifica un requisito** y, tratado como
prosa de sección, se desacopla (y funde cuando hay varios).

- El parser ata `aim*` al requisito **precedente**, en `Requirement.aim` (paralelo a
  `expected_output`), con disciplina `accepted_text`. Un `aim*` sin requisito abierto cae a `body`
  (mismo fallback que EXPECTED OUTPUT).
- El `RequirementChunker` incluye `aim` en el chunk del requisito, verbatim, delimitado, bajo la
  coordenada del requisito — como `expected_output` (Brief 4, Task 4).
- Resuelve el desacople **y** la fusión: cada AIM viaja con su requisito, ninguno se funde con el
  del vecino.
- Riesgo bajo (asociación, no toca numeración). Render-validar los 24 casos: cada AIM queda con su
  requisito correcto.

---

## Task 4 — Declarar los 4 números de cláusula repetidos

Dos cláusulas distintas comparten número en 3 handbooks (`4.1.1 Introduction` y `4.1.1 Process
definitions` en `Q-HB-80-02 Part 2A`). La ambigüedad **es del documento fuente**, no del parser.

- **No se fabrica desambiguador** (colgar el título cambiaría la coordenada respecto a lo que el
  documento imprime — identidad inventada). El breadcrumb de títulos del header + el verbatim
  desambiguan para el lector, como los sub-chunks que comparten coordenada de sección.
- Se **declaran los 4 casos** en el reporte como ambigüedad de cita conocida (visible, no tapada).
  Son informativos y son 4; se revisa solo si el retrieval lo sufre.
- El `chunk_id` ya no colisiona (Brief 4, ordinales por número de cláusula). Esto es solo la parte
  de *cita*, no de *clave interna*.

---

## Validación contra el render — el control central

Toca `numbering.py`, así que la validación es la disciplina de S06, exhaustiva, con el **PDF como
oráculo** (es el artefacto publicado y aceptado, numeración sin ambigüedad — el que el usuario usó
para encontrar el defecto). Guion de UAT:

- **`diff_reparse` contra el parse previo:** debe mostrar exactamente el delta previsto — 482
  términos re-anclados a su cláusula, 24 AIM movidos al requisito, y **cero** cambios de identidad
  fuera de eso. Las 205 secciones del Brief 3 intactas. `exit 1` si algo más toca identidad.
- **Glosario contra PDF (exhaustivo):** los 482 números coinciden con el PDF, incluidos los que
  siguen a un término borrado.
- **AIM contra render:** cada uno de los 24 con su requisito correcto.
- **Regresión de números vivos:** ninguna cláusula viva fuera del glosario cambió de número por
  el arreglo del contador.

Si algún caso falla contra el render, CC **no lo fuerza**: lo marca y lo sube. Precisa-e-incorrecta
es peor que gruesa-pero-correcta.

---

## Regeneración y fingerprint

- Re-parseo → `chunks.json` invalidado (borrado ruidoso, ya implementado) y regenerado con la
  identidad corregida. `input_fingerprint` se actualiza (detecta la staleness, como se diseñó).
- **Referencias:** si algún requisito con AIM/EXPECTED atado, o alguna cláusula re-numerada, es
  *origen* de referencias en `references.json`, CC lo **señala** (no regenera — sigue gateado por
  validación del extractor). El enrutado de `Definition1` puede afectar a refs que apunten a
  términos de glosario; verificar y reportar.

---

## Tests (mockeados, sin red, sin datos reales)

1. **Contador sobre borrados:** una lista mock con un ítem borrado entre dos vivos numera los
   vivos `1, 2` (no `1, 3`).
2. **Definition1 → Section:** un `Definition1` mock con numeración propia produce una cláusula
   `3.2.N`, y los términos bajo `3.2` se anclan a su cláusula, no a `3.2`.
3. **Glosario tras el enrutado:** un término sigue siendo un chunk (definición+NOTE agrupadas),
   ahora anclado a su cláusula.
4. **AIM atado:** un `aim` tras un requisito va a `requirement.aim`, no a `body`; dos AIM de dos
   requisitos no se funden.
5. **No regresión Annex:** los headings `Annex4/5` del Brief 3 conservan sus números tras el
   arreglo del contador.
6. **Repetidos declarados:** dos cláusulas con el mismo número producen ids distintos y aparecen
   en el reporte de ambigüedad.

Fixtures: lista con ítem borrado, cláusula de glosario con término borrado en medio, requisito con
AIM, sección con dos requisitos y dos AIM, un `Annex4` (regresión), dos secciones con número
repetido.

---

## Al cerrar (CC)

- `S07-decisions-5-identidad-final.md`: log crudo, etiquetado agnóstico/dominio. Sube al chat.
- Deriva `CHANGELOG.md` y **reescribe el estado presente** en `CLAUDE.md`/`ARCHITECTURE.md`:
  `Definition1` enrutado, contador de lista descuenta borrados, `Requirement.aim`, números
  repetidos declarados. Sin diario.
- Guion de UAT (render, exhaustivo sobre lo afectado, PDF como oráculo).
- Reporte de la auditoría de Task 0 (input para Chat si aparecen estilos más allá de `Definition1`).
- Reporte de frecuencia de refs a sub-ítem (Task 1, oportunista).
- Cualquier `# TODO`.

---

## Fuera de scope (no implementar)

- **Marcadores literales de lista** (`1.`, `(a)`) — se mide su necesidad (Task 1) pero no se
  implementan aquí; siguen gateados.
- **Estilos ambiguos de la auditoría** (si los hay) — se reportan, se deciden en Chat, no se
  enrutan a ciegas.
- **Desambiguador de números repetidos** — se declaran, no se fabrica coordenada.
- **Regeneración de `references.json`/grafo** — solo se señala el acoplamiento; sigue gateada.
- **Embeddings, wrapper, vector store** — movimiento siguiente, tras consolidar S07.
