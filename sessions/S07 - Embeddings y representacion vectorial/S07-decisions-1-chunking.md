# S07-decisions-1-chunking.md — Implementación (Claude Code)

Decisiones y fricciones al implementar `S07-brief-01-chunking.md` sobre `RAG-ECSS`.
Etiquetadas **[agnóstico]** (candidata a Guide) vs **[dominio]** (solo ECSS).

---

## Resultado de la Tarea 0 — 2 de 3 supuestos fallan

El brief mandaba parar y marcar si un supuesto de la verificación previa fallaba. Fallaron dos,
y uno cambia el alcance del brief. La verificación se ejecutó sobre los 26 `data/processed/*.json`
reales y sobre los DOCX de `data/parse_ready/`.

### ❌ Supuesto 1 — «cada unidad trae su texto `accepted_text`». **Falso para las secciones.**

`Section` no tiene campo de texto. Sus campos son `number`, `title`, `level`,
`annex_qualifier`, `number_prefix`, `authority`. **La prosa del cuerpo nunca se extrajo.**

El despacho de `corpus/parser.py` cubre headings, anexos, contexto DRD, `require:level1/2/3`,
`ECSS_IEPUID` y tablas. No hay rama para `paragraph`, `NOTE`, `NOTE:numbered`, `NOTE:bul`,
`Bul1`, `Bul2`, `Normal` ni `EXPECTED OUTPUT`: esos párrafos se descartan en silencio.

Medido sobre el corpus entero:

```
párrafos del corpus en los DOCX   4.143.797 chars
prosa nunca extraída              2.494.612 chars   (60%)
```

Por documento: `ECSS-E-ST-40C` pierde el **53%** (160.778 ch — `paragraph` 49k, `NOTE` 18,5k,
`EXPECTED OUTPUT` 13k, `NOTE:numbered` 12,7k, `NOTE:bul` 12,4k, `Bul2` 5,3k);
`ECSS-Q-HB-80-02 Part 2A` pierde el **86%**.

Ejemplo literal, `ECSS-E-ST-40C` cláusula 3 (*Terms and definitions*) — el glosario entero de
un estándar CORE:

```
[Definition1] ❌ descartado  configurable code
[paragraph  ] ❌ descartado  code that is only intended to be executed in certain specific
                            configurations of the software product
[NOTE       ] ❌ descartado  This can be achieved either by use of compilation/link
                            directives, parameter configuration...
```

**Consecuencia para el brief:** el `SectionChunker` estaba diseñado para cubrir los 6 handbooks
sin requisitos y el contenido de sección. Esos 6 handbooks no aportan «591 secciones de
contenido» — aportan **591 títulos**. El chunker existe y funciona, pero hoy solo tiene de qué
tirar donde hay tablas aplanadas.

- **[dominio] Decisión tomada:** implementar todo lo que no depende de la prosa, y llevar el
  hueco del parser a Claude Chat como brief propio. Arrastra una decisión de autoridad que CC
  no debe tomar: **una NOTE dentro de una cláusula normativa es informativa**, y un
  `EXPECTED OUTPUT` puede ser metadato del requisito y no contenido citable. Equivocarse ahí
  es barato hoy y caro después de embeber.
- **[agnóstico]** Un parser escrito para un consumidor (extraer requisitos) no es
  automáticamente el input de otro (chunking para RAG). El hueco no era visible mientras el
  único consumidor fuese el que definió el parser; apareció en cuanto el chunking pidió *todo*
  el cuerpo del documento. Verificar el shape del input antes de construir encima **encontró un
  agujero del 60% que ninguna suite de tests habría señalado**, porque todos los invariantes
  del parser se cumplen: lo que falta nunca se pidió.

### ⚠️ Supuesto 2 — «`section_number` es prefix-addressable». **Cierto el diseño, falsa la implementación ingenua.**

0 secciones sin número (bien: los rótulos no llegan a `Section`, decisión de S06). Pero
`"5.20".startswith("5.2")` es cierto y 5.20 **no** es descendiente de 5.2: **1.328 pares del
corpus** colisionan así.

- **[agnóstico] Decisión:** `is_descendant()` compara **componentes**, nunca prefijos de
  cadena. Es implementación, no decisión de arquitectura, pero la trampa es silenciosa: la
  expansión «dame todo lo que cuelga de 5.2» habría sobre-recogido sin avisar.

### ❌ Supuesto 3 — «`table_references` es joinable a `references.json`». **Falso.**

El shape real es `list[tuple[str, str]]` = `(doc_id, título)`. **No lleva `source_clause_id`**,
así que no se puede unir al schema plano de `references.json`. 281 pares, vigentes a 26 docs
(no stale). Unirlos exigiría re-extraerlos con la cláusula contenedora — trabajo de parser,
no de este brief. Solo informativo, como decía el brief.

---

## Tres cosas más que el brief daba por sabidas y no son así

- **[dominio] El `chunk_id` de §1.3 no identifica.** No es «~mitad del corpus sin UID» sino el
  **68% (5.386/7.922)**. Y como los items DRD no se numeran (decisión de S06: sus títulos van a
  `subsection_path`), muchos requisitos comparten `section_number`: hasta **51 requisitos en
  `ECSS-E-ST-10C#P.2.1`**. El esquema del brief dejaba **5.034 requisitos (64%) con id
  duplicado**.
  Medido: `subsection_path` apenas ayuda (4.741 colisiones); un hash del texto deja 25
  (requisitos literalmente repetidos dentro de una sección) y además es opaco; solo un
  **ordinal dentro de la sección** llega a 0.
  → `# TODO: pendiente de decisión en Claude Chat` — el ordinal es posicional, y el brief pedía
  identidad derivada solo de la coordenada. Con el 68% sin UID no existe coordenada que
  desambigüe. Verificado sobre el corpus real: **9.403 chunks, 9.403 ids únicos, 0 colisiones**.
- **[dominio] `Requirement` no tiene `authority`.** Se hereda de su sección, que es donde el
  parser la etiquetó. El join es total: 0 requisitos huérfanos sobre los 26 documentos.
- **[dominio] `corpus_version` no está en los JSON por documento.** Sale del manifiesto
  (`corpus/manifest.yaml`, hoy `2026-08-04.91bc8116`), que es donde la identidad del corpus se
  fija. Se estampa en el artefacto y en cada chunk.

---

## Decisiones de implementación

- **[agnóstico] `CiteRef.standard` lleva el doc_id COMPLETO con revisión** (`ECSS-E-ST-40C
  Rev.1`); `chunk_id` y `ChunkMetadata.standard` llevan el canónico (`ECSS-E-ST-40C`). El brief
  no distinguía. Lo fija `CLAUDE.md`: la canonicalización existe para topología (que una
  referencia y su destino colapsen en un nodo) y **no debe propagarse al contrato de cita**,
  donde el nombre del documento se conserva entero. Son dos identidades con dos propósitos, y
  mezclarlas haría citar un documento que no es exactamente el que se leyó.
- **[agnóstico] El header no lleva número de cláusula.** El brief lo prohíbe en el texto
  («no metas identificadores crudos») pero su ejemplo incluye `Section: 5.4 Software design`.
  Se siguió la regla, no el ejemplo: el contexto semántico lo dan los **títulos** de la ruta de
  secciones (`Design process > Software design`), y el número vive en metadata.
  → `# TODO` — el brief pide además la MATERIA del estándar (`ECSS-E-ST-40C — Software`), pero
  el título del documento tampoco está en `data/processed/`: el parser no captura el estilo
  `Document:Title`. Va con el mismo hueco de prosa.
- **[agnóstico] `SectionChunker` se escribió contra el ancla, no contra las tablas.** Hoy su
  única fuente de contenido son las tablas aplanadas, pero cuando el parser capture prosa
  alimenta el mismo camino sin cambiar de forma. Un chunker llamado `TableChunker` habría
  hecho permanente un accidente temporal.
- **[dominio] Solo lo aplanado con fidelidad entra al índice.** Las tablas `mark`, `discard` y
  `references` no son texto fiel y no son contenido citable. Una tabla sin cláusula contenedora
  se salta: sin ancla no se le inventa una.
- **[agnóstico] `TokenCounter` inyectable, por defecto heurístico** (~4 chars/token). Nada de
  `tiktoken`: es específico de OpenAI y el modelo de embedding sigue sin decidir. Solo gobierna
  cuándo se sub-parte una sección y cuándo se reporta un requisito sobredimensionado, así que
  una estimación basta; cambiarlo por el tokenizer real es una inyección, no una reescritura.
- **[agnóstico] `outgoing_refs` vacío**, como manda el brief. El extractor de referencias sigue
  sin validar contra ground truth (1 de 50 filas etiquetadas), así que poblarlo hoy arriesga
  citas fantasma.
- **[dominio] El build falla ruidoso si hay ids duplicados** (`exit 1`). La identidad del chunk
  es la del futuro embedding: dos chunks con el mismo id son dos vectores que se pisan.

---

## Salida medida sobre el corpus real

```
9.403 chunks · 9.403 ids únicos · 0 colisiones
por ancla:      requirement 7.922 · section 1.481
por autoridad:  normative 8.682 · informative 721
tokens/chunk:   mediana 83 · p90 220 · máx 663
requisitos sobredimensionados (marcados, NO troceados): 2
```

Los 2 sobredimensionados son procedimientos largos con pasos embebidos
(`ECSS-E-ST-10-03C#5.5.1.2`, 583 tokens; `ECSS-E-ST-40-07C#5.3.7`, 663 tokens). Ninguno se
partió: partir un requisito rompe la identidad que lo hace citable.

**Lectura honesta de estas cifras:** la mediana de 83 tokens es baja porque el 84% de los
chunks son requisitos, que son atómicos y cortos. No mide que el chunking esté bien calibrado
— mide que hoy el corpus indexable es casi solo requisitos. La cifra a vigilar cuando entre la
prosa es la de `section`, hoy mediana 247.

---

## Pendiente de decisión en Claude Chat

1. **El hueco de prosa del parser** (el grande). Qué estilos son contenido citable y con qué
   autoridad: `paragraph`, `NOTE`/`NOTE:numbered`/`NOTE:bul`, `Bul1`/`Bul2`, `Definition1`,
   `Normal`, `EXPECTED OUTPUT`. Una NOTE dentro de cláusula normativa es informativa; un
   EXPECTED OUTPUT puede ser metadato del requisito.
2. **Qué se cita cuando una sección se sub-parte** — ¿la sección entera o el sub-chunk? (el
   TODO que el brief anticipaba). Hoy se cita el sub-chunk, que es lo verbatim recuperado.
3. **El ordinal en el `chunk_id`** — aceptar que la identidad sea parcialmente posicional, o
   pedir al parser un identificador estable por requisito.
4. **Título de documento en el header** — va con el punto 1.

---

## Anexo — inventario de la prosa perdida (input para decidir el punto 1)

Claude Chat no tiene los DOCX ni el código, así que la decisión «qué estilos son contenido
citable y con qué autoridad» necesita ver **qué es cada estilo**. Esto es el inventario del
corpus entero (26 documentos), no de un documento de muestra, con ejemplos literales.

### Volumen por estilo no capturado

`2.403.239` chars de párrafo repartidos así (top 22; el resto es cola larga). La cifra difiere
de los `2.494.612` de arriba en ~91k porque aquélla es `total − requisitos − títulos` e incluye
por tanto los marcadores `ECSS_IEPUID`, que son metadato y no prosa; esta tabla suma solo los
estilos que el parser no consume:

| estilo | chars | % | párrafos | docs |
|---|---:|---:|---:|---:|
| `paragraph` | 1.266.340 | 52,7% | 5.832 | 26 |
| `NOTE` | 232.926 | 9,7% | 2.113 | 24 |
| `NOTE:numbered` | 194.673 | 8,1% | 1.565 | 22 |
| `Bul1` | 182.726 | 7,6% | 1.610 | 26 |
| `list:level1` | 158.506 | 6,6% | 1.244 | 21 |
| `NOTE:bul` | 49.937 | 2,1% | 610 | 16 |
| `indentpara1` | 32.619 | 1,4% | 128 | 7 |
| `Bul2` | 31.942 | 1,3% | 357 | 17 |
| `list:level2` | 26.075 | 1,1% | 366 | 18 |
| `list:level1:indent1` | 25.002 | 1,0% | 200 | 1 |
| `EXPECTED OUTPUT` | 24.489 | 1,0% | 452 | 2 |
| `toc 3` | 21.853 | 0,9% | 664 | 14 |
| `CaptionFigure` | 16.541 | 0,7% | 296 | 1 |
| `toc 2` | 12.836 | 0,5% | 378 | 14 |
| `CaptionTable` | 10.667 | 0,4% | 230 | 14 |
| `Bul3` | 10.279 | 0,4% | 146 | 9 |
| `table of figures` | 9.542 | 0,4% | 167 | 13 |
| `toc 1` | 8.744 | 0,4% | 218 | 14 |
| `Definition1` | 7.681 | 0,3% | 480 | 23 |
| `NOTE:cont` | 7.593 | 0,3% | 53 | 10 |
| `EXPECTED OUTPUT:CONT` | 6.912 | 0,3% | 108 | 2 |
| `Caption:Annex Table` | 4.633 | 0,2% | 113 | 17 |

### Qué es cada estilo, literalmente

```
paragraph      The material in this Handbook is defined in terms of description and
               recommendation how to organize and perform the work of space software
               engineering and product assurance following an Agile approach.

NOTE           A story point is a relative measurement unit used to compare relatively
               user stories, epics (e.g. one ideal working day).

NOTE:numbered  1  It is also known as daily meeting, daily Scrum or roll-call.

Bul1           In clause 6, a reference process model based on Scrum to be used to map
               its elements to relevant clauses of ECSS-E-ST-40.

list:level1    For the purpose of this document, the terms and definitions from
               ECSS-S-ST-00-01 apply, in particular the following terms:

NOTE:bul       re-assessing the relative priority of user stories, assigning estimates
               to user stories which have yet to receive one,

indentpara1    enforces a close analytical look to significant cost, risk and technical
               drivers as well as critical issues and specific constraints.

EXPECTED       System and software observability requirements [RB, SSS; SRR].
OUTPUT         Verification and validation process requirements [RB, SSS; SRR].

CaptionFigure  Figure 85 Successful progress of execution verification report
CaptionTable   Table 54: Key factors for selection of classical or agile lifecycle

toc 2          6.5  Organising the Agile activities and meetings in a project to create
                    a life-cycle compliant to ECSS-E-ST-E-40    44
```

Y una entrada de glosario completa (`ECSS-E-ST-40C`, cláusula 3), que **son tres estilos
distintos formando una unidad**:

```
[Definition1] acceptance test
[paragraph  ] test of a system or functional unit usually performed by the customer on
              his premises after installation, with the participation of the supplier...
[NOTE       ] Adapted from ISO/IEC 2382-20:1990
```

### Lectura de CC — input, no decisión

- **Contenido sin duda (≈80% del volumen):** `paragraph`, `Bul1/2/3`, `list:level1/2`,
  `indentpara1`. Es la prosa del cuerpo y la continuación en viñetas de esa prosa.
- **No es contenido (≈2,2%):** `toc 1/2/3`, `table of figures`. Es la tabla de contenidos —
  indexarla produciría chunks de puro ruido de navegación, con números de página incluidos.
- **Contenido, pero con autoridad propia (≈20%):** la familia `NOTE*`. En ECSS una NOTE es
  informativa **aunque viva dentro de una cláusula normativa**, así que heredar la `authority`
  de la sección sería incorrecto justo aquí: es el único caso del corpus donde la autoridad
  del párrafo no es la de su cláusula.
- **Genuinamente abierto — `EXPECTED OUTPUT`** (24.489 ch, solo 2 documentos). Los ejemplos son
  `System and software observability requirements [RB, SSS; SRR]`: no es prosa, es la salida
  documental esperada de un requisito, con el documento destino y el hito de revisión entre
  corchetes. Puede ser **metadato del requisito** (y entonces viaja con él, no como chunk
  propio) o **contenido citable** (y entonces es lo que respondería a «qué entregables exige la
  cláusula X»). Las dos lecturas son defendibles y la elección cambia el schema.
- **Genuinamente abierto — la unidad del glosario.** `Definition1` es solo el término; su
  cuerpo es un `paragraph` y su procedencia una `NOTE`. Chunkeados por separado, el término
  queda sin definición y la definición sin término. La cláusula 3 de cada estándar es
  glosario, y es material de consulta de primer orden para el RAG.
- **Menor pero decidible — `CaptionFigure`/`CaptionTable`.** La figura no se extrae, así que su
  caption es la única traza textual de que existe. Indexarla permite responder «hay una figura
  sobre X», sin poder mostrarla.
