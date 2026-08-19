# S06 — Decisions log, brief 4: numeración e identidad de cita

> Log crudo de decisiones + porqué. `[agnóstico]` (aplica a cualquier pipeline sobre documentos
> ofimáticos) / `[dominio]` (específico de RAG-ECSS y del corpus ECSS).

---

## 1. El paso de medición encontró un bug en el instrumento, no un número `[agnóstico]`

El brief pedía cerrar «cuántos requisitos cuelgan de las 6 secciones divergentes». La primera
respuesta fue **0**, y el cero era falso: la lista de 6 venía del diagnóstico, y el diagnóstico
tenía un fallo.

Lo que destapó el cero fue no aceptarlo. Comprobando de dónde colgaban los requisitos de esos
anexos, aparecieron 19 anclados a `E.1.1` y `F.1.1` — números que el resolver producía **dos
veces en el mismo documento**, para dos headings distintos. Dos cláusulas compartiendo identidad
de cita es peor que una identidad equivocada: la cita es ambigua y nada lo detecta.

Causa: Word **materializa** un nivel intermedio ausente, no solo lo muestra. El anexo E de
`Q-ST-30-02C` no tiene ningún heading de nivel 2, así que su primer nivel 3 imprime `E.1.1` con
un `1` implícito; y cuando después aparece el primer nivel 2 de verdad, Word lo numera **E.2**,
porque ese `1` ya estaba consumido. El resolver mostraba el `1` sin guardarlo, así que volvía a
contar desde 1 y colisionaba.

**Lo grave no es el bug, es dónde estaba escondido.** El diagnóstico marcaba `Expected response`
como *coincidencia*: parser y resolver decían `E.1` los dos. Coincidían porque **compartían el
mismo error**. Es la trampa que el propio diagnóstico dejó escrita (§1 de sus decisions: «compara
dos implementaciones, no una contra la verdad») materializándose — y significa que la columna
«coinciden» nunca fue prueba de acierto.

Consecuencias medidas, todas con verificación manual contra el render de por medio:

| | antes del fix | después |
|---|---|---|
| divergencias | 461 | **469** |
| documentos afectados | 7 | **9** (aparece `Q-ST-20C Rev.2`, que figuraba como limpio) |
| requisitos citando mal | «0» | **22** |

**Regla que queda:** un instrumento de medición necesita validación externa antes de creerse su
propio output, y «las dos implementaciones coinciden» no es esa validación.

## 2. Una sola implementación del resolver, en `corpus/numbering.py` `[agnóstico]`

El prototipo vivía en el diagnóstico. Se podía copiar a producción y dejar dos, o extraer a un
módulo que ambos consuman. Se extrajo — es literalmente la lección de C-4 (dos rutas de identidad
conviviendo sin declarar cuál es la buena), aplicada antes de crear el problema en vez de después.

Efecto colateral honesto: el diagnóstico pasa de comparar dos implementaciones a comprobar el
**cableado** (que el parser use de verdad el número resuelto y lo alinee bien). Es un test de
regresión más débil que antes, y hay que saberlo al leer su «0 divergencias».

## 3. Un rótulo no recibe identidad: no es una Section `[dominio]`

Los 41 rótulos podían modelarse como `Section` con `number=""` o no ser `Section` en absoluto.
Se eligió **lo segundo**: `sections` pasa a significar «cláusulas citables», sin excepciones que
haya que recordar aguas abajo. El título del rótulo viaja como contexto en `subsection_path`,
reutilizando el mecanismo que el proyecto ya tenía para los items DRD.

Beneficio no previsto: eso hace **imposible por construcción** que un rótulo llegue a una cita.
No hay que acordarse de filtrar `number == ""` en el chunker, en el retrieval ni en `Citation`.

Sub-decisión: el contexto sin número lleva `kind` (`label` / `drd`) además de profundidad, porque
un rótulo de heading y un item DRD anidan por ejes distintos y una sola pila los mezclaría.

## 4. El nivel de una sección sale del número, no del estilo `[agnóstico]`

Antes, `Section.level` venía del estilo (`Heading 3` → 3). Ahora sale de la profundidad del
número resuelto (`B.2.1` → 3). No es cosmético: en el anexo E, un heading con estilo `Annex3`
imprime `E.1.1`, que es profundidad 3 — coinciden. Pero donde no coincidan, la jerarquía real es
la del número, porque es la que el lector ve y la que hace que `section_path` sea consistente con
la cita. El estilo pasa a decir solo *qué es un heading*.

## 5. Dos identidades de documento, declaradas como tales `[dominio]`

`doc_id` estaba haciendo dos trabajos incompatibles:

- **Topológica** (`graph.canonical_doc_id`): quita `Rev.N` a propósito, para que los dos extremos
  de una referencia cruzada caigan en el mismo nodo. Se mantiene intacta.
- **De cita** (`parser.document_key`): **conserva la revisión**. `ECSS-E-ST-40C` a secas es el
  superseded de 2009 en el inventario; colapsarlo con `Rev.1` es citar norma derogada con cara de
  vigente.

Hay un test que las contrasta lado a lado sobre el mismo `doc_id`, porque el riesgo no es que una
esté mal sino que alguien las confunda.

**La clave es reversible y eso es el arreglo de C-4.** `sanitize_filename` colapsa espacio, punto
y paréntesis sobre `_`, así que `Rev.1` y `Rev 1` se vuelven indistinguibles. `document_key` mapea
**solo** espacio↔underscore, y como los números ECSS no llevan underscore, la ida y vuelta es
exacta. No se tocan los nombres de `data/raw/`: el manifiesto (Brief 6) es quien mapeará fichero →
identidad, y hasta entonces la clave existe y es correcta.

## 6. La autoridad se hereda del anexo, y por defecto del tipo de documento `[dominio]`

`document_type` **no existía** — `CLAUDE.md` lo daba por implementado y no lo estaba en ningún
sitio. Se deriva del número (`-HB-` → handbook), que es dato del propio identificador y no exige
tocar el inventario.

`Section.authority` (`normative` / `informative`) se resuelve así: un handbook es informativo
siempre; un estándar es normativo salvo que la cabecera de un anexo diga lo contrario, y esa
cabecera la heredan sus sub-secciones. Se reutiliza el vocabulario que el grafo ya usa en
`mention_type`, en vez de inventar otro.

Alcance respetado: el dato viaja hasta la unidad citable y ahí se para. `Citation` **no** cambia —
es contrato de producto, y extenderlo antes de que exista el tratamiento diferenciado sería
cambiar la API por especulación. `# TODO`→S11 en el punto donde ramificará.

## 7. `INV4` cambia de significado, no de umbral `[agnóstico]`

`INV4` («toda sección tiene padre presente») existía para cazar el contador desincronizado. Con
la numeración resuelta ese fallo **es imposible por construcción**, y lo que el invariante detecta
ahora es otra cosa: que Word materializó un nivel sin darle cabecera (`E.1.1` sin ningún `E.1`).
Eso es propiedad del documento y el número es fiel.

Así que deja de ser fallo y pasa a ser un recuento informativo. No se relaja un umbral: se
reconoce que el invariante ya no puede fallar por la razón por la que se escribió. En su lugar
`verify_parser` reporta **cobertura de identidad** (`N/N secciones con número resuelto`), que sí
puede degradarse si alguien vuelve a colar una identidad fabricada.

## 8. Los tests necesitaban documentos con numeración de verdad `[agnóstico]`

Efecto no obvio del cambio: los tests del parser construían DOCX con python-docx, cuya plantilla
no define listas — así que con la numeración resuelta **todos sus headings pasaron a ser
rótulos** y 14 tests se cayeron a la vez. No era un bug: el test estaba ejercitando el caso
degenerado sin saberlo.

Se añadió un helper que inyecta definiciones de lista reales en `numbering.xml` (decimal de 6
niveles para el cuerpo, alfabético + decimal para anexos) y numera los headings como lo hace un
documento ECSS. Ahora los tests ejercitan el camino normal, y el caso rótulo se prueba a
propósito con `_label()`.

## Verificación

Sobre los 26 documentos reales, tras la reparación:

- **0 divergencias** contador-vs-XML (eran 469).
- **41 rótulos, 0 con número fabricado** (eran 41 fabricados).
- **0 requisitos citando un número inexistente** (eran 22).
- `verify_parser`: 25/26 sin hallazgos; el único restante es el `INV5(soft)` de un requisito en
  minúscula, que el propio check declara señal blanda.
- Identidad: `N/N` secciones con número resuelto en todos los documentos.
- 412 tests en verde, sin red ni `data/`.

## Tensiones abiertas

- **«0 divergencias» ya no es evidencia independiente** (§2): parser y diagnóstico comparten
  resolver. La evidencia real es la verificación manual contra el render; el guion UAT la
  concreta.
- **`lvlOverride` se aplica pero no está verificado contra el render.** Solo `E-HB-40-01A` lo usa;
  sus números salen coherentes, pero nadie ha comprobado uno a ojo.
- **La cobertura sigue en 54%**: `data/processed/` conserva los 14 JSON viejos, ahora además con
  numeración obsoleta. Re-parsear el corpus es requisito para que la reparación llegue a los
  artefactos, y no hay todavía comando que lo haga.
- **`Citation` sigue sin revisión.** La identidad correcta existe en el parser; que llegue al
  contrato HTTP es trabajo de M2, cuando `retrieval.py` construya las citas desde el corpus.
