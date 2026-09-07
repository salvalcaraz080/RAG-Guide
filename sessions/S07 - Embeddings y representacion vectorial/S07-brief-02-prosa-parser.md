# S07 · Brief 2 — Extracción de prosa del parser

> **Para:** Claude Code. **Produce:** la extensión de `corpus/parser.py` (y schema de
> `ParsedDocument`) que captura la prosa de cuerpo que hoy se descarta en silencio (~60% del
> corpus), la expone en una forma consumible por el chunker, y ata `EXPECTED OUTPUT` a su
> requisito. **NO produce:** cambios en el chunker, agrupado de glosario, mapeo estilo→autoridad,
> ni embeddings. Esos son el brief siguiente, gateados por la verificación de este.
>
> **Contrato de trabajo:** CC no decide arquitectura; marca `# TODO: pendiente de decisión en
> Claude Chat` y sigue. Tests mockeados, sin red, sin Docker, sin tocar `data/raw/`. La
> verificación de fidelidad sobre DOCX reales es UAT del usuario. Commits los hace el usuario.

---

## 0. Contexto — por qué existe este brief

La Tarea 0 del Brief 1 descubrió que `corpus/parser.py` nunca extrajo la prosa del cuerpo: su
despacho cubre headings, anexos, DRD, `require:level*`, `ECSS_IEPUID` y tablas, pero descarta
en silencio `paragraph`, `NOTE*`, `Bul*`, `list:*`, `Definition1`, `EXPECTED OUTPUT` y otros.
Medido: **2,49M chars de prosa nunca extraída, el 60% del corpus.** Consecuencia: el
`SectionChunker` del Brief 1 está hambriento — los 6 handbooks sin requisitos aportan hoy 591
*títulos*, no contenido. Este brief cierra ese agujero.

**Decisiones ya tomadas en Claude Chat que gobiernan el brief** (no reabrir):

- La prosa de cuerpo (`paragraph`, `Bul1/2/3`, `list:level1/2`, `indentpara1`) es **contenido
  citable**.
- Los índices de navegación (`toc 1/2/3`, `table of figures`) **no son contenido** → se excluyen.
- `EXPECTED OUTPUT` es **contenido citable atado a su requisito** (viaja con él, no como chunk
  suelto).
- La familia `NOTE*` es contenido con **autoridad informativa aun dentro de cláusula normativa**
  — pero **el mapeo estilo→autoridad lo aplica el chunker, no el parser** (ver §4). El parser
  solo preserva el estilo.
- El glosario (`Definition1` + `paragraph` + `NOTE` como unidad) **lo agrupa el chunker** — el
  parser solo preserva estilo y orden (ver §4).

---

## 1. El cambio de schema

Hoy `Section` no tiene campo de texto: por eso la prosa se perdía. El cambio central es exponer
el cuerpo de cada sección como **secuencia ordenada de párrafos tipados**, no como blob.

```python
@dataclass(frozen=True)
class Paragraph:
    style: str        # el nombre de estilo del DOCX, sin normalizar: "paragraph",
                      # "NOTE", "NOTE:numbered", "Bul1", "Definition1", ...
    text: str         # accepted_text (ver §2.1): w:ins conservado, w:del descartado

@dataclass(frozen=True)
class Section:
    # ... campos existentes: number, title, level, annex_qualifier,
    #     number_prefix, authority ...
    body: list[Paragraph]   # NUEVO — prosa de cuerpo, en orden de documento

@dataclass(frozen=True)
class Requirement:
    # ... campos existentes ...
    expected_output: list[str]   # NUEVO — párrafos EXPECTED OUTPUT atados a este requisito
```

**Por qué tipado y ordenado, no un string aplanado:** el chunker necesita el estilo para aplicar
la regla "NOTE = informativa" y para agrupar la unidad de glosario, y necesita el orden para que
`Definition1` + su `paragraph` + su `NOTE` queden contiguos. Aplanar a blob en el parser
destruiría justo la información que el chunker necesita para decidir. El parser preserva; el
chunker decide.

---

## 2. Qué se extrae y cómo

### 2.1 Prosa de cuerpo → `Section.body`

Añadir al despacho del parser las ramas que hoy faltan, emitiendo un `Paragraph{style, text}`
por cada párrafo de cuerpo, en orden, bajo su sección.

**Disciplina de control de cambios, obligatoria:** el texto es `accepted_text` (conserva `w:ins`,
descarta `w:del`), igual que ya se hace para requisitos. Los DOCX de ECSS llegan con revisiones
sin aceptar; `Paragraph.text` de python-docx las ignora y corrompería texto en silencio — el
mismo fallo que S06 arregló para requisitos y que NO se puede reintroducir para la prosa.

### 2.2 `EXPECTED OUTPUT` → `Requirement.expected_output`

`EXPECTED OUTPUT` (y `EXPECTED OUTPUT:CONT`) se **atan al requisito que preceden** en el orden
del documento. Esta asociación es estructural (el parser procesa linealmente y conoce las
fronteras de requisito), por eso vive en el parser y no en el chunker. Se guarda como lista de
strings `accepted_text` en el requisito. La *decisión semántica* de que es contenido citable ya
está tomada (Chat); el parser solo implementa la asociación estructural.

### 2.3 Extraer-todo-con-etiqueta; exclusiones DECLARADAS, no silenciosas

**Principio rector de este brief, derivado del fallo que lo originó:** un párrafo que no se
extrae debe ser una **exclusión declarada y contada**, nunca un descarte silencioso. El agujero
del 60% existió porque los estilos no cubiertos desaparecían sin traza. La contramedida es la
misma disciplina que S06 aplicó a las tablas (`discard` con motivo nombrado):

- El parser reconoce **todos** los estilos de párrafo de cuerpo.
- Los estilos-contenido (`paragraph`, `Bul*`, `list:*`, `indentpara1`, `NOTE*`, `Definition1`,
  `EXPECTED OUTPUT*`, captions — ver §2.4) van a `Section.body`/`Requirement.expected_output`.
- Los estilos-no-contenido (`toc 1/2/3`, `table of figures`) se **excluyen con motivo**, y su
  volumen se **reporta** (§3). No se saltan en silencio.
- Un estilo **desconocido** (no en ninguna de las dos listas) no se descarta: se **reporta como
  no clasificado** para que una revisión futura lo enrute. Silencio = regresión; visibilidad =
  correcto.

### 2.4 Captions — decisión menor, incluida aquí

`CaptionFigure`/`CaptionTable`/`Caption:Annex Table` son la única traza textual de una figura o
tabla que no se extrae. Se **extraen como contenido** (`Section.body`), con su estilo preservado,
para que el chunker pueda responder "hay una figura sobre X" sin poder mostrarla. Es coherente
con el "gap marcado, nunca silencioso": la existencia de la figura queda registrada aunque su
contenido gráfico no.

### 2.5 Lo que el parser NO hace (es del chunker, brief siguiente)

- **No** mapea estilo→autoridad. La regla "NOTE es informativa aun en cláusula normativa" la
  aplica el chunker sobre el estilo que el parser preservó. El parser expone `Section.authority`
  y el `style` de cada párrafo; el chunker computa la autoridad del chunk.
- **No** agrupa la unidad de glosario. El parser deja `Definition1`, `paragraph` y `NOTE`
  contiguos y tipados en `body`; el chunker los agrupa.
- **No** chunkea. Produce `ParsedDocument` enriquecido; el chunking lo consume después.

---

## 3. Cobertura y reporte

La extracción de prosa debe alimentar `corpus/processing-report.md` (el artefacto "qué RESULTÓ"
de S06), para que la cobertura sea **visible y versionada en git** — una cobertura que baja entre
revisiones es una regresión que ninguna foto suelta enseña.

Reportar, por documento y total:

- chars de prosa extraída, por estilo-contenido.
- chars excluidos, por estilo-no-contenido, con motivo.
- estilos **no clasificados** encontrados (con conteo) — la señal de que apareció algo nuevo.
- delta contra el estado previo (prosa extraída pasó de 0 a N).

El objetivo es que, tras este brief, "qué se captura / qué se excluye / qué no se reconoce" sea
un dato declarado, no un supuesto. Es la corrección directa del modo de fallo que originó el
brief.

---

## 4. Decisión que este brief SURFACEA (no resuelve): versionado de derivados

`corpus_version` es `{fecha de scrapeo}.{digest sobre doc_id}` — identifica el **conjunto
declarado**, no el contenido procesado ni la lógica del pipeline. Este brief **cambia el output
procesado sin cambiar `corpus_version`** (mismo conjunto de docs, más prosa extraída). Efecto:
los chunks derivados del parse viejo (sin prosa) y del nuevo (con prosa) llevarían el mismo
`corpus_version` (`2026-08-04.91bc8116`), siendo distintos. Eso rompe la garantía "un derivado
se ata a su versión" — la misma que sostiene el determinismo de embeddings.

`# TODO: pendiente de decisión en Claude Chat`. No bloquea este brief (los chunks se regeneran),
pero **debe resolverse antes del brief de embeddings**. Recomendación a discutir: estampar en los
derivados una **versión de pipeline/parser** junto a `corpus_version`, de modo que un cambio de
lógica de extracción (no solo de conjunto de docs) invalide los derivados. CC no lo implementa
aquí; solo deja el `# TODO` donde corresponda.

---

## 5. Tests (mockeados, sin red, sin DOCX reales)

Invariantes a fijar con DOCX mockeados / fixtures de `python-docx`:

1. **Prosa extraída:** una sección con `paragraph`/`Bul1`/`list:level1` produce `body` con esos
   párrafos, en orden, con su `style` preservado.
2. **accepted_text:** un párrafo con `w:ins`/`w:del` mockeado se extrae con el texto aceptado
   (inserción conservada, borrado descartado).
3. **EXPECTED OUTPUT atado:** un `EXPECTED OUTPUT` tras un requisito aparece en
   `requirement.expected_output`, no en `section.body`.
4. **Exclusión declarada:** un `toc 2` no entra en `body` y se contabiliza como excluido (no
   desaparece sin traza).
5. **Estilo desconocido reportado:** un estilo inventado no se descarta en silencio — se marca
   no-clasificado.
6. **Orden preservado:** `Definition1` seguido de `paragraph` seguido de `NOTE` quedan contiguos
   y en ese orden en `body` (precondición del agrupado de glosario del brief siguiente).
7. **Sin regresión de requisitos:** los requisitos y su identidad/numeración siguen extrayéndose
   igual que antes (este brief añade, no altera lo existente).

Fixtures que cubran: sección de prosa normal, sección de glosario (los tres estilos), requisito
con EXPECTED OUTPUT, sección con solo toc (todo excluido), un handbook sin requisitos, un párrafo
con control de cambios.

---

## 6. Gate de verificación (UAT del usuario) — antes del brief siguiente

Entregar un guion de UAT para que el usuario corra el parser extendido sobre `data/parse_ready/`
reales y confirme, antes de que se construya el chunking encima:

- La prosa extraída cuadra con el inventario de la Tarea 0 (los ~2,4M chars aparecen).
- Un muestreo de secciones de cuerpo sale **fiel** (sin corrupción por control de cambios, sin
  párrafos partidos).
- El glosario de una cláusula 3 (p. ej. `ECSS-E-ST-40C`) sale con sus tres estilos contiguos.
- El reporte de cobertura no deja estilos no-clasificados de volumen relevante.

Solo cuando esto pase se escribe el **Brief 3 — consumición en chunking** (glosario agrupado,
autoridad NOTE, `SectionChunker` con contenido real). Verificar-antes-de-construir es la regla
que originó este brief; se aplica también a su salida.

---

## 7. Al cerrar (CC)

- `S07-decisions-2-prosa.md`: log crudo, etiquetado agnóstico/dominio. Sube al chat.
- Deriva `CHANGELOG.md` y **reescribe el estado presente** en `CLAUDE.md`/`ARCHITECTURE.md`:
  registrar que el parser ahora extrae prosa de cuerpo tipada, el nuevo shape de `Section.body`
  y `Requirement.expected_output`, y la disciplina de exclusión declarada. Sin diario.
- Guion de UAT (§6).
- `# TODO` de §4 (versión de pipeline) puesto donde corresponda.

---

## Fuera de scope (no implementar)

- **Mapeo estilo→autoridad** (NOTE informativa) — Brief 3, lo aplica el chunker.
- **Agrupado de glosario** — Brief 3, lo hace el chunker sobre estilo+orden preservados.
- **Cualquier cambio en el chunker / regeneración de chunks** — Brief 3.
- **Resolver el versionado de pipeline (§4)** — decisión de Chat, antes del brief de embeddings.
- **Embeddings, wrapper de proveedor, vector store** — briefs/sesiones posteriores.
