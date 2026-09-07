# S07 · Brief 1 — Chunking: chunks ref-ready por ancla de contenido

> **Para:** Claude Code. **Produce:** el módulo de chunking del corpus ECSS, que transforma
> los `data/processed/*.json` (26 docs, S06) en objetos `Chunk` listos para embeber.
> **NO produce:** embeddings, wrapper de proveedor, escritura en vector store, población de
> referencias salientes. Esos son briefs posteriores o trabajo gateado (ver §Fuera de scope).
>
> **Contrato de trabajo (recordatorio):** CC no toma decisiones de arquitectura; si aparece
> una, la marca `# TODO: pendiente de decisión en Claude Chat` y sigue sin bloquear. Tests
> mockeados, sin red, sin Docker, sin tocar `data/raw/`. Commits los hace el usuario. La
> ejecución del pipeline sobre datos reales es UAT del usuario.

---

## 0. Contexto mínimo (por qué este diseño)

El sistema debe citar **texto normativo verbatim** con trazabilidad total: cada respuesta
anclada a `standard#clause`. Eso impone tres invariantes que gobiernan todo el brief:

1. **El texto citable es el texto fuente, sin reescribir.** Nada de parafrasear, renderizar ni
   "proposicionalizar" un requisito. Lo que se cita debe ser byte-a-byte lo que dice el DOCX.
2. **Lo que se embebe ≠ lo que se cita.** Al vector le añadimos contexto para que sea
   recuperable; a la cita no le añadimos nada. Son dos campos distintos del mismo `Chunk`.
3. **El ancla de contenido es mixta dentro de un documento.** Un standard normativo tiene
   requisitos *y* secciones con tablas aplanadas; 6 de los 26 docs no tienen ningún requisito
   (591 secciones). No se enruta por tipo de documento — se chunkea por ancla de contenido.

---

## 0-bis. TAREA 0 — Verificación previa (obligatoria, antes de implementar)

Antes de escribir el chunker, CC entrega un **script de inspección read-only** que el usuario
correrá en UAT sobre `data/processed/`. No muta nada; imprime un reporte. Objetivo: confirmar
tres supuestos de los que depende el diseño. Si alguno falla, CC **para y marca** el punto
como decisión pendiente en vez de asumir.

Ubicación sugerida: `corpus/diagnostics/inspect_chunk_inputs.py` (encaja con la convención de
`diagnostics/`: mide, no escribe en `data/`).

Debe reportar:

1. **Shape de la salida de secciones/requisitos por documento.** Para un `data/processed/*.json`
   de muestra: campos disponibles por sección y por requisito, y en concreto si cada unidad
   trae de forma fiable su `section_number` resuelto, su texto `accepted_text`, y su
   `authority` (normativo/informativo). Esto fija de qué campos leen los chunkers.
2. **`section_number` es prefix-addressable.** Confirmar que los IDs de sección/cláusula son
   jerárquicos por prefijo (`5.2.3.1` bajo `5.2`, anexos `B.2.1` bajo `B`), de modo que "todos
   los chunks de la sección 5.2" = filtro por prefijo del ID. Reportar cualquier caso que rompa
   la propiedad (p. ej. rótulos sin número, front-matter `Heading 0`).
3. **`table_references`: existencia, shape y vigencia.** Localizar el artefacto (28 tablas →
   281 pares esperados). Reportar: (a) si cada par carga `source_clause_id` de la cláusula
   contenedora, *joinable* al schema de lista plana de `references.json`
   (`{source_clause_id, source_doc_id, source_uid, target_doc_id, target_clause_id,
   mention_text, mention_type}`); (b) si está a 26 docs o stale a 14. **Este punto solo
   informa un brief posterior** (la población de refs); no bloquea el chunking, pero se mide
   ahora porque el usuario lo pidió y evita sorpresas.

Entregable de la Tarea 0: el script + un guion de UAT de una línea por punto para que el
usuario lo corra y pegue el reporte. CC no ejecuta contra datos reales.

---

## 1. Interfaz y modelo de datos

### 1.1 El objeto `Chunk`

Un `dataclass` (o modelo Pydantic si prefieres validación) con, como mínimo:

```python
@dataclass(frozen=True)
class Chunk:
    chunk_id: str            # identidad estable y trazable (ver §1.3)
    embed_text: str          # header contextual + texto fuente — lo que SE EMBEBE
    cite_ref: CiteRef        # coordenada + texto verbatim limpio — lo que SE CITA
    anchor: str              # "requirement" | "section"
    metadata: ChunkMetadata  # filtrable, fuera del vector (§1.4)
    outgoing_refs: list      # SLOT ref-ready — vacío en este brief (§3)
    token_count: int         # vía tokenizer abstraído (§2.3), no tiktoken directo
```

`embed_text` y `cite_ref` son **campos separados a propósito**. El invariante 2 se rompe si en
algún punto la extracción de cita lee `embed_text`: el header sintético acabaría "citado". Un
test debe fijar que `cite_ref.text` no contiene el header.

`CiteRef` lleva: `standard`, `clause` (el `section_number` resuelto), `revision`,
`corpus_version`, y `text` (verbatim). Es la coordenada + el texto que el usuario ve citado.

### 1.2 Despacho por ancla de contenido

Interfaz común y despacho que corre **ambos** extractores sobre cada `ParsedDocument`:

```python
class Chunker(ABC):
    @abstractmethod
    def chunk(self, doc: ParsedDocument) -> list[Chunk]: ...

class RequirementChunker(Chunker):
    """Un requisito = un chunk. Overlap 0. cite = identidad (normativo)."""

class SectionChunker(Chunker):
    """Una sección = un chunk. Cubre docs sin requisitos + contenido de sección
    (incluidas tablas aplanadas de standards normativos). Overlap intra-sección
    admitido; fallback recursive-dentro-de-sección si excede ventana."""
```

**No enrutar por tipo de documento.** Ambos chunkers procesan cada documento; cada uno emite
los chunks de su ancla. Un mismo standard producirá chunks `requirement` (de sus requisitos) y
chunks `section` (de sus secciones con tabla aplanada). Que un documento no tenga requisitos es
un caso normal, no un error: el `RequirementChunker` devuelve `[]` y el `SectionChunker` hace
el trabajo.

> **Decisión de diseño a respetar:** el grado de autoridad (`normative`/`informative`) **viaja
> con el contenido**, tomado de lo que el parser ya etiquetó en S06 (la tabla lleva la
> `authority` de su sección). No se deduce del tipo de documento ni se hardcodea por chunker.

### 1.3 `chunk_id` — identidad estable

Formato trazable y determinista, derivado de la coordenada, no de un contador ni un hash de
posición. Sugerencia: `{canonical_doc_id}#{section_number}` para unidad sección, y para unidad
requisito el mismo con el UID como sufijo cuando exista (`{canonical_doc_id}#{section_number}::{uid}`),
cayendo al `section_number` cuando el requisito no tiene UID (~mitad del corpus no lo tiene).
Debe ser **prefix-addressable** (§3) y **reproducible** entre corridas con el mismo
`corpus_version`.

### 1.4 `ChunkMetadata` — filtrable, fuera del vector

```
standard          # canonical_doc_id, p.ej. "ECSS-E-ST-40C"
clause            # section_number resuelto
revision          # letra de revisión
corpus_version    # ata el chunk (y su futuro embedding) a la versión del corpus
authority         # "normative" | "informative"
anchor            # "requirement" | "section"
```

Estos campos **no entran en `embed_text`** (son ciegos al embedding: IDs y letras no aportan
señal semántica, añaden ruido). Sirven para filtrar en el retrieval (S08+) y para construir la
cita. `corpus_version` es load-bearing: un embedding derivado debe poder atarse a la versión que
lo produjo (cobra el anclaje de S06 "`corpus_version` no viaja a los JSON").

---

## 2. `embed_text` — cómo se construye

### 2.1 Header contextual estático + texto fuente verbatim

```
[Standard: ECSS-E-ST-40C — Software | Section: 5.4 Software design]
<texto fuente verbatim del requisito o de la sección>
```

El header carga contexto **semántico** (materia del standard, título de la sección/tema) para
que el chunk sea recuperable aun aislado. **No** metas en el header identificadores crudos
(clause-ID, revisión, `corpus_version`): van en metadata, no en el vector.

El header es **estático y determinista** — construido de campos que el parser ya nos da, sin
llamar a ningún LLM. Esto no es solo más barato que Contextual Retrieval con LLM: es
**reproducible** (mismo `corpus_version` → mismo `embed_text` → mismo embedding). El
enriquecimiento por LLM queda fuera de scope precisamente porque metería no-determinismo en el
índice.

### 2.2 `cite_ref.text` — verbatim, limpio

El texto fuente que va a `cite_ref.text` es el `accepted_text` del parser (conserva `w:ins`,
descarta `w:del` — los DOCX de ECSS llegan con control de cambios sin aceptar). **Sin header,
sin reescritura.** Un test fija que `cite_ref.text` es substring exacto del texto de la unidad
en el `ParsedDocument`, y que no contiene el prefijo `[Standard: ...]`.

### 2.3 Conteo de tokens — tokenizer abstraído

El fallback de sub-partición (§2.4) necesita medir tokens. **No uses `tiktoken` directo** — es
OpenAI-específico y el modelo está sin decidir. Define una interfaz `TokenCounter` con una
implementación por defecto, inyectable. El chunking no debe acoplarse al tokenizer de un
proveedor concreto.

### 2.4 Unidad requisito vs unidad sección

- **Requisito:** un requisito = un chunk. **Overlap 0** (un requisito es un átomo acotado;
  solapar arrastraría cola del requisito vecino y contaminaría la cita). No sub-partir: un
  requisito que excede ventana es una **señal a reportar**, no un corte (partir un requisito
  rompe cita = identidad). Marca y reporta ese caso; no lo trocees.
- **Sección:** una sección = un chunk. **Overlap intra-sección admitido** (prosa continua
  dentro de la sección). Si una sección excede la ventana, **fallback recursive-dentro-de-
  sección** (jerarquía de separadores sobre la prosa de esa sección). Bajo cite = procedencia
  esto es tolerable; deja `# TODO: pendiente de decisión en Claude Chat` sobre **qué se cita
  cuando una sección se sub-parte** (¿la sección entera?, ¿el sub-chunk?) — es decisión de
  arquitectura, no la resuelvas tú.

---

## 3. Slot ref-ready + ID prefix-addressable

El chunk expone `outgoing_refs` como **slot vacío** en este brief. NO lo pobles: la población
lee de `references.json` / `table_references`, cuyo extractor está sin validar y cuyo artefacto
puede estar stale — poblar ahora arriesga citas fantasma. La población es un paso downstream
gateado por el re-run + validación del extractor (fuera de scope).

Lo que **sí** cablea este brief: que el `chunk_id`/`clause` sea **prefix-addressable**, de modo
que un consumidor futuro pueda resolver "referencia a sección 5.2" como "todos los chunks cuyo
clause empieza por `5.2`" sin un índice de padres físico. Un test fija la propiedad: dado un
conjunto de chunks, filtrar por prefijo de `clause` devuelve exactamente los descendientes
esperados.

Esto deja el chunk **agnóstico a la forma de consumo**: sirva luego el retrieval por adyacencia,
por metadata o por baking-en-embedding, el material del chunk es el mismo. La decisión
grafo-vs-metadata **no se toca aquí**.

---

## 4. Pipeline, IO y salida

- **Entrada:** `data/processed/*.json` (los 26 docs re-parseados en S06). Léelos como
  `ParsedDocument`; no re-parsees DOCX.
- **Salida:** un artefacto derivado de chunks — p. ej. `data/processed/chunks.json` (o
  `chunks/*.json` por documento; elige y documenta). Debe llevar `corpus_version` a nivel de
  artefacto y por chunk, para que cualquier derivado (embeddings, S08) se ate a su versión.
- **Naturaleza:** es un **build batch sobre nuestro corpus**, no un endpoint cara-a-cliente.
  Idempotente: misma entrada + mismo `corpus_version` → misma salida. No hay `document_type`
  en payload (el ancla se decide por contenido, no por un llamante).
- **Módulo:** vive del lado corpus/ingesta, no en el hot-path del servidor. Sugerencia:
  `corpus/chunking.py` (biblioteca) + un CLI de build (`python -m corpus.chunk_all` análogo a
  `parse_all`), coherente con el patrón de `corpus/`. Confírmalo contra la estructura existente;
  si `app/services/` encaja mejor por la dirección de M2, marca la duda.

---

## 5. Tests (mockeados, sin red, sin datos reales)

Pipeline por unidad: implementación → tests automáticos → (UAT del usuario) → docs.

Invariantes que los tests deben fijar (no exhaustivo):

1. **Verbatim:** `cite_ref.text` es substring exacto del texto de la unidad en el
   `ParsedDocument` mockeado; no contiene el header.
2. **embed ≠ cite:** `embed_text` contiene el header y el texto fuente; `cite_ref.text` no
   contiene el header.
3. **Overlap 0 en requisito:** dos chunks-requisito consecutivos no comparten texto.
4. **Sin sub-partir requisito:** un requisito sobre-dimensionado se marca/reporta, no se trocea.
5. **Fallback de sección:** una sección sobre-dimensionada se sub-parte; una normal no.
6. **Ancla mixta:** un documento mock con requisitos + una sección con tabla aplanada produce
   chunks de ambas anclas; un documento sin requisitos produce solo chunks `section` (y no
   lanza error).
7. **Authority del contenido:** un chunk hereda la `authority` etiquetada en su unidad de
   origen, no una deducida del tipo de documento.
8. **Prefix-addressable:** filtrar chunks por prefijo de `clause` devuelve los descendientes
   correctos.
9. **Determinismo:** dos corridas sobre la misma entrada + `corpus_version` producen `chunk_id`
   y `embed_text` idénticos.
10. **Slot vacío:** `outgoing_refs == []` en todos los chunks de este brief.

Usa `ParsedDocument` mockeados que cubran: un standard con requisitos y tablas aplanadas, un
doc sin requisitos (solo secciones), un requisito sin UID, una sección sobre-dimensionada, un
requisito sobre-dimensionado, un anexo (`B.2.1`).

---

## 6. Al cerrar (CC)

- `SNN-decisions.md` = `S07-decisions.md`: log crudo de decisiones + porqué, etiquetado
  agnóstico/dominio. Sube al chat.
- Deriva `CHANGELOG.md` (entrada de sesión) y **reescribe el estado presente** en
  `CLAUDE.md`/`ARCHITECTURE.md` (no diario). En concreto: registrar el módulo de chunking, el
  contrato `Chunk` (embed ≠ cite, slot ref-ready), y el despacho por ancla.
- Guion de UAT propuesto para el usuario (incluye la corrida del script de la Tarea 0).
- Marca cualquier `# TODO: pendiente de decisión en Claude Chat` que haya surgido (esperado al
  menos uno: qué se cita cuando una sección se sub-parte).

---

## Fuera de scope de este brief (no implementar)

- **Población de `outgoing_refs`** — gateada por re-run + validación del extractor a 26 docs
  (universo efectivo 20). Poblar desde artefactos stale/no-validados = riesgo de cita fantasma.
- **Wrapper de embeddings / generar vectores** — Brief 2, parcialmente bloqueado por elección
  de modelo (abierta).
- **Escritura en vector store / pgvector** — S08.
- **Contenido de tabla como destino recuperable/citable** — diferido (el "gap marcado" ya
  existe vía `content_extractable: false`).
- **Contextual Retrieval con LLM; propositional; late chunking** — diferidos.
- **Elección de modelo de embedding y tokenizer concreto** — se mide aparte; aquí todo es
  abstracto/inyectable.
- **Normalización de query a inglés; fan-out y profundidad del grafo; grafo-vs-metadata** —
  M4/S09-10, decisión abierta.
