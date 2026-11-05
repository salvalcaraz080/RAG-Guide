# pre-M6 — Decisiones del brief 1: instrumento del grafo

> Entregadas las fases A, B y C, las guardas y los pendientes. Golden set **v3** importado sin
> rechazos: 29 consultas y 40 aspectos. **L2 y L3 son definitivos; L1 sale INVALID** (229 pares sin
> juzgar) y sus cifras son cota inferior hasta el modo B, que necesita una sesión ciega. Las únicas
> llamadas al proveedor fueron los embeddings de las consultas nuevas.

## 1. Qué encontró G7 *(dominio)*

**Expected output: ya está.** 588 de 588 entradas de `requirements[].expected_output` aparecen
literales en el texto de su cláusula: E-ST-40C 282, Q-ST-80C 278, E-ST-80C 28. No hizo falta añadir
nada; el protocolo le dice al anotador dónde mirarlas.

**Lo que no está para el brief 4: el ancla del DRD.** El parse da la etiqueta (`[TS, SRS; PDR]`), no
la arista. El título del anexo sí permite derivarla (`Software requirements specification (SRS) -
DRD`). Con una regex simple de etiqueta, primera aproximación y no cifra final:

| Documento | Expected output | Con etiqueta de documento | Con DRD en el mismo documento |
|---|---|---|---|
| E-ST-40C | 282 | 159 | 145 |
| Q-ST-80C | 278 | 118 | 60 |
| E-ST-80C | 28 | 0 | 0 |

Hay etiquetas cuyo DRD está en otro estándar: SRF o SRS desde Q-ST-80C caen en anexos de E-ST-40C, y
SCF en M-ST-40C. SSAR no tiene anexo «- DRD» en el corpus con esta lectura. El diccionario del brief 5
y la arista DRD del brief 4 se solapan aquí.

**Matrices: están en el texto, pero como una cláusula enorme.** Seis, con una fila por requisito:

| Cláusula | Forma de la fila | Cláusulas de la matriz | Unidas a una cláusula |
|---|---|---|---|
| E-ST-40C R.2 | `Requirement identification:` | 146 | 146 |
| Q-ST-80C D.2 | `Clause:` | 321 | 264 |
| Q-ST-40C F | `Clause:` | 151 | 121 |
| Q-ST-20C 6 | `5.2.6e \| X \| …` | 79 | 79 |
| E-ST-10C 7 | `ECSS req. #:` | 32 | 32 |
| E-ST-10-02C 6 | `ECSS req. #:` | 38 | 38 |

Las no unidas son encabezados sin texto propio (`5`, `5.1`…): no hay cláusula en el export.
El export las muestra bajo cada cláusula (1.122 filas en 680 cláusulas) con el número del requisito
como única clave. Se leen del texto del chunk, sin extraer nada nuevo de `corpus/`. R.2 cruza además
requisito y expected output (`5.2.2.1a eo b`). Dos límites para el brief 4: el chunking corta filas
(solo se conserva la forma completa si el solape la trae), y las tablas de E-ST-10-03C y el resto de
matrices sin número de requisito no entran. F.2–F.7 de Q-ST-80C y Q.* de E-ST-40C son «qué documento
en qué revisión», no aplicabilidad.

## 2. Space-to-Ground ICD *(dominio)*

**Entró como negativo** (q28, forma `out_of_corpus`): el anotador lo citó en E-ST-70-41C § 1. El
enlace está en contexto y no se sigue, como debe. Era el ejemplo que el protocolo proponía, así que es
elección guiada, no hallazgo independiente del anotador.

## 3. Decisiones de implementación

| # | Decisión | Porqué |
|---|---|---|
| 1 | `reached` exige que la **cita** esté en un chunk de `answer_clause`, no la cláusula | El brief decía «llega `answer_clause`», pero una cláusula puede llegar sin su tabla: es el caso `same_clause`. Regla de acierto por evidencia. *(agnóstico)* |
| 2 | Aspecto con **`alternatives`** | 4 de los 14 aspectos tienen varias respuestas válidas (q06-a1, q08-a2, q11-a1, q12-a2). Con una sola se contaría como fallo una respuesta buena. La principal es la que nombra el puntero |
| 3 | `from_clause` de q11-a1: **E-ST-40C 5.4.2.1** | Ni el diagnóstico ni `aspects.py` lo traían; es el puntero «see ECSS-Q-ST-80 clause 7.2» que encontró V1 |
| 4 | `not_retrieved` cuando no hay otra cláusula que lleve a la respuesta; `not_pointed` cuando la hay y no llegó | Las definiciones del brief solapaban: si `from_clause` no llega, la respuesta tampoco |
| 5 | Cláusula presente sin su cita → `over_limit` | Solo un tope deja fuera un chunk de una cláusula entregada |
| 6 | `link_quote` se valida también contra las filas de matriz exportadas | En `applicability` el enlace es la fila, que vive en la matriz. En L2, `applicability` y `hierarchy` cuentan el enlace como presente si llega `from_clause` |
| 7 | Negativos con destino en `answer_clause`, `form` y columna `link in context` | El brief no fijaba el destino y sin él no se distingue «seguido». Un negativo cuyo enlace nunca llegó es trivial: se nombra en el informe. `followed` solo mira la expansión, no lo que trajo el retrieval por sí mismo |
| 8 | **La forma de cortesía exige un estándar del índice** | El ejemplo del brief, ECSS-S-ST-00, **no está en el corpus**: coincidiría con la forma `out_of_corpus` |
| 9 | Una consulta sin camino se sustituye en fase 2, con aprobación | Sin leer el corpus no se sabe si hay camino, y la fase 1 es ciega. Queda marcado en `note` |
| 10 | `graph_targets import` amplía golden y targets a la vez; `golden extend` se niega si trae aspectos | «Rechaza el fichero entero» no se cumple con dos pasos separados |
| 11 | Los documentos de D1 salen del **export**, no de la tabla `documents` | Comprobado igual (26) y deja el informe re-puntuable sin base. Desviación menor del brief |
| 12 | Las guardas **no citan el brief** en el código | `CLAUDE.md` lo prohíbe y el brief pedía «este brief como fuente»: `GUARDS_VERSION = 1`, y este documento y el CHANGELOG hacen de enlace |
| 13 | Workspace nuevo, `ecss-annotation-v3` | El anterior contiene `annotation.json` y pares, que el protocolo prohíbe ver |
| 14 | Un enlace que el parser lee y que está en contexto es `pointed_not_followed` aunque la traza no lo registre | `xref` no vuelve a leer lo que trae la expansión: q16 (B.2.2, traída por `xref`) salía `pointed_not_resolved`, «el parser no lo lee», y sí lo lee. Fallo del clasificador hallado en la fase B, con 2 tests que lo codificaban. Artefacto re-puntuado, sin consultar la base |

Una **limitación de medida** de `hops`: el aspecto guarda solo el último eslabón, así que L2 daría
`not_pointed` para un 2 saltos sin saber en cuál se rompió. No hay ningún caso: los 40 son de 1 salto.

## 4. Línea base (L1–L3)

**Artefactos** (`data/groundtruth/retrieval/measurements/`): `measurement_20261031-165719.md`,
`reach_20261031-165749.json` y su re-puntuación `reach_20261031-165749_20261031-165918.md`.

### L2–L3: alcanzabilidad

`defaults` (`filter`, `bounded`, `clause`, top 10), 40 aspectos: 36 positivos y 4 negativos.

| `edge_kind` | n | reached | not_pointed | pointed_not_resolved | pointed_not_followed | over_limit | not_followed | followed |
|---|---|---|---|---|---|---|---|---|
| `explicit_clause` | 11 | 9 | 0 | 0 | 0 | 2 | 0 | 0 |
| `explicit_document` | 14 | 1 | 0 | 0 | 10 | 0 | 3 | 0 |
| `drd` | 3 | 3 | 0 | 0 | 0 | 0 | 0 | 0 |
| `applicability` | 3 | 1 | 1 | 0 | 0 | 1 | 0 | 0 |
| `hierarchy` | 3 | 3 | 0 | 0 | 0 | 0 | 0 | 0 |
| `by_name` | 6 | 2 | 0 | 3 | 0 | 0 | 1 | 0 |

Ninguno `not_retrieved`. **Todos con `hops` 1:** no hay caso de 2 saltos.

Lectura (el orden de los briefs lo decide el chat):

- **`reached` no significa resuelto:** es techo de contexto; depende después de la generación.
- **Puntero a documento: 10 de 11 positivos sin alcanzar.** q08, q12 ×4, q14, q15 ×2 y q16 ×2. Es el
  mayor techo, el del brief 3. `clause_standard` midió 1 de 6 aspectos objetivo: techo no es ganancia.
- **Por nombre: 3 de 5 positivos sin resolver** (q24 ×2 hacia el SCF de M-ST-40C, q25 hacia E-ST-10C),
  sin parser que lea esa arista.
- **Estructurales, techo casi nulo:** `drd` y `hierarchy` llegan ya, 6 de 6, sin arista nueva. De los 9
  aspectos solo fallan 2, ambos `applicability`: q20-g1 es un tope y q20-g2 no trajo el enlace. **n = 3
  por arista: indicativo.** Con estos datos, el brief 4 no tiene dónde ganar en DRD ni jerarquía.
- **`over_limit` (3):** q06-a2 y q20-g1 (R.2 presente y cortada) y q09-a3, cuya subcláusula 6.2.7.8 se
  quedó sin plaza con 6.2.7 expandida y truncada. Ampliar topes es otra variable, no el grafo.
- **Negativos: 4 de 4 sin seguir.** Solo 2 prueban algo: en q10-a1 y q28-g1 el enlace estaba en
  contexto. En q27-g1 y q29-g1 nunca llegó, así que son triviales.

### L1: retrieval sobre v3 — INVALID

| Consultas | n | p@k | R@k | R@candidatos | Sin juzgar |
|---|---|---|---|---|---|
| v1 | 10 | 0,66 | 0,32 | 0,53 | 30 |
| v2 | 14 | 0,54 | 0,26 | 0,42 | 63 |
| v3 | 29 | 0,45 | 0,32 | 0,50 | 229 |

| Estrato nuevo | n | p@k | R@k | Sin juzgar |
|---|---|---|---|---|
| `graph_document_pointer` | 2 | 0,21 | 0,16 | 22 |
| `graph_drd` | 3 | 0,45 | 0,58 | 20 |
| `graph_applicability` | 2 | 0,14 | 0,24 | 33 |
| `graph_hierarchy` | 2 | 0,52 | 0,71 | 22 |
| `graph_by_name` | 3 | 0,31 | 0,29 | 22 |
| `graph_negative` | 3 | 0,45 | 0,29 | 47 |

**Superado:** L1 válido en `pre-M6-decisions-2.md` §0, tras juzgar estos pares.

## 5. `delegated` hacia el corpus (D3)

**`gaps` de `generation_20261024-140241`: 15 huecos**, los 15 del resumen (14 en `runs` y 1 en
`turn_runs`).

| Clase | Huecos | Casos |
|---|---|---|
| `in_corpus_not_injected` | 5 | q12 ×3 → M-ST-80C; q14 ×2 → E-ST-70-01C |
| `in_corpus_injected` | 3 | q08 ×3 → Q-ST-30-02C |
| `out_of_corpus` | 0 | |
| `unresolved` | 7 | q04 ×5 (guía ISVV, CSW-ESAISVV-2022-GBK-02897), q13 ×1 (SAVOIR), q14 turno 1 ×1 (Space-to-Ground ICD) |

**UAT de S14: 4 huecos.** 2 `in_corpus_not_injected` (M-ST-80C, Q-HB-80-03A) y 2 `out_of_corpus`
(Q-ST-10-04, Q-ST-10-09).

Dos lecturas que el número solo no da:

- **Los 3 de q08 están mal clasificados en el fondo.** El documento estaba en contexto solo por su
  § 2, referencias normativas; la respuesta (§ 4.1) no, y L2 los da `pointed_not_followed`. Para un
  puntero a documento entero, «el destino estaba» es indecidible. El informe nombra la cláusula
  presente. **El fallo de retrieval real es 8 de 15, no 5.**
- **Los 7 `unresolved` son en realidad externos.** Todos nombran un documento no ECSS, que el parser
  no lee. Ninguno oculta un hueco de retrieval.

## 6. Pendientes del brief

- `S14-2-panel-look`: **ya estaba** en `partial` por la 0.68.1; sin cambios.
- Compose: Postgres publicado en `127.0.0.1`; contenedor recreado, 13.695 chunks intactos y API 200.
- Notas del anotador: **Q-ST-80C 6.2.3.1 no es un hueco del parse.** Existe en el parse con sus
  requisitos como `<<deleted>>` (texto retirado en la Rev.2): marcadores no citables que la carga
  descarta, así que no hay cláusula en el export. q22 remite a Q-ST-20C 5.4.1, que solo tiene texto en
  5.4.1.1–5.4.1.3: q22-g3 llega igualmente (`reached`).

## 7. Qué falta

1. Nada de este brief: el modo B y L1 válido se cerraron en `pre-M6-decisions-2.md` §0.

UAT de este brief: revisar las secciones 4 y 5. Es juicio del usuario o del chat, y no hay render.
