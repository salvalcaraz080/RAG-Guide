# S07 · Brief 4 — Consumición en chunking: la prosa entra al índice

> **Para:** Claude Code. **Produce:** el `SectionChunker` consumiendo la prosa real del parser
> (los 2,26M chars que hoy están en `data/processed/` y no en el índice), el agrupado de
> glosario, la autoridad por chunk, EXPECTED OUTPUT en el chunk de requisito, `Subtitle` como
> materia del header, y el versionado por fingerprint de contenido (decisión B de Chat).
> Regenera `chunks.json`. **NO produce:** embeddings, wrapper de proveedor, vector store.
>
> **Contrato de trabajo:** CC no decide arquitectura; marca `# TODO: pendiente de decisión en
> Claude Chat` y sigue. Tests mockeados, sin red, sin Docker, sin `data/raw/`. La validación
> sobre corpus real es UAT del usuario. Commits los hace el usuario.

---

## 0. Contexto — qué desbloquea este brief

Tras los Briefs 2 y 3, `data/processed/` tiene por fin la prosa de cuerpo (2,26M chars, tipada y
ordenada en `Section.body`), la identidad de sección completa (`Annex4/5` enrutados), EXPECTED
OUTPUT atado a 457 requisitos, y `document_title`/`Subtitle` a nivel documento. Pero el
`SectionChunker` del Brief 1 **sigue comiendo solo tablas aplanadas** — toda esa prosa está en
disco y **no en el índice**. Este brief cierra el hueco: es donde el corpus se vuelve
consultable. Es el último brief antes de embeddings.

Riesgo asimétrico, tareas separadas: Task 0 (Subtitle) es aditiva; Tasks 1–4 tocan qué se
embebe y qué se cita — el corazón del sistema. Task 6 (versionado) es la red que hace detectable
la staleness que los Briefs 2/3 introdujeron.

---

## Task 0 — `Subtitle` como materia (parser, ~5 líneas, con gate de reporte)

El Brief 3 descubrió que `Document:Title` es la **rama** ECSS (`Space engineering`, 3 valores,
ya implícita en la letra del doc_id → cero señal nueva). La **materia** que el header del Brief 1
pedía (`Software`, `Verification`…) está en `Subtitle`.

- Capturar `Subtitle` a `ParsedDocument.document_subject` (mismo patrón que `document_title`,
  con la corrección `w:txbxContent` del Brief 3 ya aplicada).
- **Gate:** reportar los 26 valores de `Subtitle` para que Chat verifique que es materia usable
  en todos, no solo en los 4 vistos. Si alguno está vacío o es basura, se declara; el header
  cae al doc_id canónico ahí (§Task 5).

---

## Task 1 — `SectionChunker` consume `Section.body` (prosa real)

Reescribir el `SectionChunker` para que chunkee el **cuerpo de sección** (unidad = sección), no
solo tablas aplanadas. Las tablas aplanadas ya viven dentro de `Section.body` como parte del
contenido, así que dejan de ser un camino aparte.

- **Unidad = sección.** Una sección → un chunk cuando cabe en ventana.
- **Sección sobre-dimensionada → fallback recursive-dentro-de-sección** (jerarquía de
  separadores sobre la prosa de esa sección), con **overlap modesto (10–15%) entre sub-chunks**.
  El overlap aquí sí tiene sentido (prosa continua), a diferencia del overlap 0 de requisito.
- **Cita cuando una sección se sub-parte** (resuelve el `# TODO` del Brief 1): la **coordenada**
  de cita es la sección (los sub-cortes no tienen clause-ID propio); el **texto** citado es el
  sub-chunk verbatim recuperado. Varios sub-chunks comparten coordenada de sección; el texto
  verbatim distingue cuál fundamentó la respuesta. El usuario ve "sección 6.5" + el fragmento
  exacto.
- **embed ≠ cite** igual que en requisito: `embed_text` = header contextual + prosa; `cite_ref`
  = prosa verbatim limpia + coordenada. La extracción de cita nunca agarra el header.
- **Fidelidad de estructura, no solo de texto** (lección del Brief 3): la prosa ya llega con su
  anidamiento preservado por la sangría del Brief 3. El chunker **no debe re-aplanar** listas al
  construir el texto — conservar los saltos y la sangría que el parser dejó. Aplanar aquí
  reintroduciría el defecto de afirmación que el Brief 3 arregló.

---

## Task 2 — Agrupado de glosario (cláusula 3)

El Brief 2 verificó que la cláusula 3.x sale con `Definition1` → `paragraph` → `NOTE` contiguos
y en orden. El chunker los **agrupa en una unidad por término**, no en chunks sueltos (un
término sin su definición no sirve).

- **Detección:** una sección cuyo cuerpo contiene `Definition1` es glosario → el chunker cambia
  a agrupado por término. Patrón: `Definition1` (término) + `paragraph`(s) (cuerpo) +
  `NOTE`(s) (procedencia) forman **un** chunk, hasta el siguiente `Definition1`.
- **Cita = identidad (verbatim).** Una definición se cita textual, nunca parafraseada
  ("ECSS-E-ST-40C define 'software' como…"). El chunk de término es cite=identidad.
- **La NOTE de glosario NO se separa.** Es procedencia de la definición ("Adapted from
  ISO/IEC…"), parte de la unidad. El agrupado de glosario **tiene precedencia** sobre la regla
  de NOTE standalone (Task 3). Coordenada de cita = la cláusula del término (3.x).

---

## Task 3 — Autoridad por chunk: ningún chunk mezcla autoridades

La autoridad (`normative`/`informative`) es propiedad **por chunk**, tomada de lo que el parser
etiquetó (`Section.authority`), refinada por el estilo del párrafo. Principio rector: **un chunk
no mezcla autoridades** — o el generador presentaría una nota no vinculante como vinculante.

- **Chunk de sección:** autoridad = `Section.authority`.
- **NOTE standalone** (familia `NOTE*` que no es parte de una unidad de glosario):
  - en sección **normativa** → sale como **chunk propio informativo** (para que nunca se cite
    como identidad normativa), con la coordenada de su sección para trazabilidad.
  - en sección **informativa** (handbook) → **se queda** en el chunk de sección (el conjunto ya
    es informativo; separar no aporta y fragmenta).
- **Chunk de requisito:** normativo (cite=identidad), como Brief 1. Sin cambio.
- El estilo original (`Paragraph.style`) es lo que el chunker lee para decidir esto; el parser
  ya lo preservó (Brief 2, §2.5). El parser no decidió autoridad — la decide aquí el chunker,
  como se acordó.

---

## Task 4 — EXPECTED OUTPUT en el chunk de requisito

El parser ató `Requirement.expected_output` (457 requisitos). Decisión de Chat: es **contenido
citable atado al requisito**. El `RequirementChunker` lo incorpora **al chunk del requisito**,
no como chunk suelto:

- `embed_text` y `cite_ref` del requisito incluyen su expected_output, **verbatim**, delimitado
  (p. ej. una línea `Expected output:` antes del bloque), para que quede claro que son dos
  partes de la misma cláusula.
- Coordenada de cita = la del requisito. Ambas partes viven bajo la misma cláusula, así que
  citarlas juntas bajo esa coordenada es fiel.
- Autoridad = normativa (es parte de la especificación del requisito).

---

## Task 5 — Header contextual con la materia real

El header del Brief 1 caía al doc_id porque no tenía materia. Ahora usa `document_subject`
(Task 0):

```
[Standard: ECSS-E-ST-40C — Software | Section: 5.4 Software design]
```

- Materia = `document_subject` (`Subtitle`). Donde falte o sea basura (según el reporte de
  Task 0), **cae al doc_id canónico** — el hueco declarado, no inventado.
- El header sigue siendo **estático y determinista** (sin LLM): mismo input → mismo header →
  mismo embedding. Identificadores crudos (clause-ID, revisión, `corpus_version`) siguen **fuera**
  del header, en metadata.

---

## Task 6 — Versionado por fingerprint de contenido (decisión B)

Cablear donde CC dejó el `# TODO` (`corpus/chunk_all.py`). `corpus_version` se queda como
**etiqueta legible del conjunto** (para cita y manifiesto); la invalidación se ancla a un
**fingerprint del contenido real consumido**, no a ese proxy.

- El artefacto de chunks estampa un `input_fingerprint` = digest sobre el contenido procesado
  que el chunker de verdad consumió (`data/processed/` en su estado actual). Un chunk está stale
  ⟺ el fingerprint del `data/processed/` actual ≠ el registrado. Esto hace **detectable** la
  deriva silenciosa que los Briefs 2/3 introdujeron (mismo `corpus_version`, contenido distinto).
- **Encadenado:** el fingerprint del stage de embeddings (digest de los `embed_text` + modelo +
  params) se implementa en el brief de embeddings. Se diseña aquí la forma para que encaje: cada
  stage fingerprintea su propio input; como el input de un stage es el output del anterior, un
  cambio de lógica aguas arriba se propaga como input cambiado aguas abajo. No hay bump manual
  (que sería "true by discipline", justo lo que evitamos).
- El borrado ruidoso de `chunks.json` en `parse_all` (Brief 2) se mantiene; el fingerprint es la
  versión que **detecta** en vez de solo **borrar**.

---

## Regeneración y reporte

- Regenerar `chunks.json` con todo lo anterior. La prosa (2,26M chars) entra al índice por
  primera vez.
- **Reporte de cobertura** (visible, la disciplina de siempre): nº de chunks por ancla
  (requisito / sección / término-glosario / NOTE-informativa), chars indexados vs chars en
  `data/processed/` (para confirmar que no se pierde prosa entre processed y chunks — un
  `processed → chunks` con fuga sería el hueco del 60% otra vez, un nivel más abajo), y delta
  contra el `chunks.json` anterior (que era solo tablas + requisitos).
- El objetivo es poder aseverar: "todo lo que está en `data/processed/` como contenido llegó al
  índice, o se excluyó con motivo". Sin fugas silenciosas processed→chunks.

---

## Tests (mockeados, sin red, sin datos reales)

1. **Prosa indexada:** una sección mock con prosa produce chunk(s) de sección con la prosa en
   `embed_text` y `cite_ref`.
2. **embed ≠ cite:** `cite_ref.text` no contiene el header en ningún tipo de chunk.
3. **Sub-partición de sección:** sección sobre-dimensionada → varios sub-chunks con overlap,
   coordenada de cita = la sección, texto = sub-chunk verbatim.
4. **Estructura preservada:** una lista anidada mock no se re-aplana al chunkear (los niveles
   sobreviven).
5. **Glosario agrupado:** `Definition1`+`paragraph`+`NOTE` → un chunk por término, cite=identidad,
   la NOTE dentro (no separada).
6. **Autoridad por chunk:** NOTE standalone en sección normativa → chunk informativo propio;
   en sección informativa → dentro del chunk de sección. Ningún chunk con dos autoridades.
7. **EXPECTED OUTPUT:** el chunk de requisito con expected_output lo incluye verbatim, delimitado,
   bajo la coordenada del requisito.
8. **Header con materia:** `document_subject` presente → aparece en el header; ausente → cae al
   doc_id.
9. **Fingerprint:** dos corridas sobre el mismo `data/processed/` dan el mismo `input_fingerprint`;
   cambiar el contenido processed lo cambia.
10. **Sin fuga processed→chunks:** todo contenido de las secciones mock llega a algún chunk o se
    excluye con motivo contado.

Fixtures: sección de prosa normal, sección sobre-dimensionada, cláusula 3 de glosario, sección
normativa con NOTE, sección informativa con NOTE, requisito con expected_output, doc sin
`Subtitle`, lista anidada.

---

## UAT (usuario, sobre corpus real)

- Un muestreo **dirigido** (no solo aleatorio — lección del Brief 3): revisar chunks de sección
  con listas anidadas contra el render, confirmando que la estructura sobrevive al chunking.
- Glosario de una cláusula 3 real: cada término es un chunk con su definición y su NOTE.
- Una NOTE dentro de una cláusula normativa real: quedó como chunk informativo, no citable como
  identidad.
- El reporte de cobertura cuadra: chars processed → chars en chunks, sin fuga.

---

## Al cerrar (CC)

- `S07-decisions-4-consumicion.md`: log crudo, etiquetado agnóstico/dominio. Sube al chat.
- Deriva `CHANGELOG.md` y **reescribe el estado presente** en `CLAUDE.md`/`ARCHITECTURE.md`:
  `SectionChunker` consume prosa, agrupado de glosario, autoridad por chunk, EXPECTED OUTPUT en
  requisito, `document_subject` en header, `input_fingerprint` en el artefacto de chunks.
- Guion de UAT (muestreo dirigido).
- Reporte de valores de `Subtitle` por documento (gate de Task 0 para Chat).
- Cualquier `# TODO`.

---

## Fuera de scope (no implementar)

- **Embeddings, wrapper de proveedor, vector store** — briefs/sesión siguientes. El fingerprint
  del stage de embeddings se diseña aquí (forma), se implementa allí.
- **Contextual Retrieval con LLM; propositional; late chunking** — diferidos.
- **Población del slot `outgoing_refs`** — sigue gateada por validación del extractor.
- **Marcadores literales de lista** (`1.`, `(a)`) — deuda declarada del Brief 3, gateada por
  medir la frecuencia de refs a sub-ítem; no se toca `numbering.py` aquí.
- **Contenido de cuadros de texto / tablas como destino** — caminos deliberados diferidos, no
  scavenging.
