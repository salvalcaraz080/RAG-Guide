# Brief C2 — Fixes transversales de código (chat puente S08 → S09, Paso C)

**Primero de dos briefs del Paso C.** C2 (este) toca **código, RUNBOOK y CHANGELOG**. C1 (después)
reconcilia `CLAUDE.md`, `ARCHITECTURE.md` y `.env` con la realidad **ya arreglada por C2**. Orden
C2 → C1 fijado: el código se corrige, luego los docs describen el presente corregido.

**Insumo obligatorio:** `sessions/bridge-S08-S09/audit-report.md`. Es la especificación. Cada tarea
cita el/los item(s) del informe; la evidencia está ahí, no se repite aquí.

**Naturaleza:** brief de **reparación**, no de auditoría. El diagnóstico ya está hecho. No buscar
trabajo nuevo, no cazar code smells, no refactorizar fuera de lo listado. Las tareas son
independientes entre sí → pueden ser commits separados. CC **no commitea**: redacta mensajes, el
usuario commitea.

## Fuera de alcance (líneas que no se cruzan)

- **No tocar** `.claude/CLAUDE.md`, `ARCHITECTURE.md` ni el `.env` local — son de C1.
- **No tocar identidad ni texto de chunk.** Nada aquí dispara re-embedding. Los 13.876 vectores no
  se regeneran.
- **No afinar el extractor de referencias** (D30/D31/D32): quedan documentados como están. C2 hace
  que `graph.py` **corra**, no que acierte más. Validar/afinar es S09+ con ground truth (D29).
- **Ficheros que C2 SÍ puede tocar:** `corpus/*.py` y `corpus/diagnostics/*.py` (los listados),
  `app/services/ingestion.py`, `app/services/sessions.py`, `app/config.py`, `app/main.py`,
  `.env.example`, `corpus/RUNBOOK.md`, `CHANGELOG.md`, y el log de decisiones
  `sessions/bridge-S08-S09/C2-decisions.md`.

---

## Tarea 1 — D13: conjunto de documentos derivado del manifiesto (accesor único)

**Ref:** informe D13, R8, R11. **Prioridad: primera.** Es un bug vivo, no latente: `corpus.graph`
e `inspect_graph` crashean hoy (`KeyError: 'sections'` en `parser.py:797`) porque hacen
`load_parsed` sobre `chunks.json`, que no es un `ParsedDocument`. Consecuencia: `references.json` y
`graph.graphml` no se pueden regenerar y el paso 7 del RUNBOOK está roto.

**Decisión de arquitectura (tomada en Claude Chat, no negociable en la implementación):** el
conjunto de documentos del corpus se deriva **del manifiesto** (`corpus/manifest.yaml`, que ya es
la fuente de identidad de `parse_all`), expuesto por **un único accesor** que consumen todos los
módulos del pipeline. Nada de parchear seis listas de exclusión — eso deja seis fuentes de verdad
que vuelven a divergir.

**Qué implementar:**

1. Un accesor único (sugerencia: en `corpus/manifest.py`, algo como
   `expected_document_ids()` / `document_json_paths(processed_dir)`) que devuelva **solo los JSON
   por-documento declarados en el manifiesto**. Los artefactos derivados (`chunks.json`,
   `embeddings.npy/.json`, `references.json`, `graph.graphml`) quedan fuera **por construcción**,
   no por lista negra.
2. Migrar los seis módulos a ese accesor donde hoy escanean `data/processed/*.json`:
   `graph.py:403`, `report.py:226-229`, `diagnostics/verify_parser.py:292`,
   `diagnostics/audit_raw_corpus.py:177`, `diagnostics/inspect_graph.py:45`, y **también**
   `chunk_all.py:50` (hoy usa `_NOT_DOCUMENTS`; migrarlo unifica la fuente aunque ya funcione).
3. **Distinción que no se puede perder:** el accesor gobierna *qué documentos son el corpus*. Los
   módulos que además leen artefactos derivados por nombre (`report.py` lee `references.json`;
   `inspect_graph` lee el grafo) **siguen leyéndolos explícitamente** — no confundir "no es un
   documento del corpus" con "no se lee nunca".

**Verificación (correr, reportar):**
- `python -m corpus.graph` completa sin crash y regenera `references.json` + `graph.graphml`.
- `python -m corpus.diagnostics.inspect_graph` completa sin crash.
- `verify_parser` → 26/26, sin la fila fantasma «EN processed/ SIN FILA ACTIVA (2): chunks, embeddings».
- `audit_raw_corpus` → parseados **26**, cobertura **100%** (no 107%).
- `report` (`build_report()`) → 26/26 sin `unexpected` espurios.
- **Diff de `references.json` regenerado vs commiteado:** reportarlo. Si cambia de forma material,
  es señal (el commiteado venía de un estado anterior del parser) — no lo escondas, no lo commitees
  tú.

---

## Tarea 2 — D15: filtro de chunks basura en ingesta

**Ref:** informe D15. 103 chunks cuyo `cite_text` es un marcador de cláusula retirada (`-` ×51,
`<<deleted…>>` ×52). Tienen embedding y son recuperables por el canal vectorial — un `<<deleted>>`
citado como fuente rompería la trazabilidad en cuanto S09 cablee retrieval a generación.

**Fix acordado (el barato):** filtrarlos **en ingesta**, sin re-chunkear ni re-embeber. La solución
robusta (que no lleguen a ser chunks: tratar los marcadores de borrado aguas arriba en
parser/chunking) **queda como deuda técnica** — la documenta C1, no se implementa aquí.

**Qué implementar en `app/services/ingestion.py`:**

1. Predicado de descarte: `cite_text.strip() == "-"` **o** `cite_text.strip().startswith("<<deleted")`.
2. **Alineación por posición, sin desincronizar (crítico).** El filtro opera sobre el par
   `(chunk, vector)` **por índice**: cuando se descarta un chunk, se descarta su vector en la misma
   posición. Nunca filtrar chunks sin filtrar el vector correspondiente — desincronizar da a cada
   chunk el vector de su vecino (fallo shape-consistent/content-false).
3. **El chequeo de coherencia del par (fingerprint/conteo/dimensión/orden) corre ANTES del filtro,
   sobre los artefactos completos.** El filtro es una política de carga, no una edición de artefacto.
4. **Auto-chequeo (la cifra es la especificación):** si el filtro descarta un número **distinto de
   103**, parar y reportar — el predicado sobre- o sub-empareja. 103 exacto o no sigas.

**Verificación:**
- La BD termina con **13.773** chunks (13.876 − 103). Si `ingestion` no es idempotente (append),
  truncar antes de recargar; reportar qué hizo.
- `count(*) WHERE cite_tsv = to_tsvector('english','')` = **0** tras recargar.
- Spot-check: un chunk conocido conserva su vector correcto (alineación intacta).

---

## Tarea 3 — O7: guardarraíl barato de sesiones multi-worker

**Ref:** informe O7 / D25. Las sesiones viven en memoria de un proceso; `--workers N>1` rompe el
multi-turn en silencio. Hoy no se manifiesta (`docker-compose` arranca `uvicorn` sin `--workers`).

**Fix acordado (barato, honesto sobre su límite):** fail-fast de arranque análogo al de
`LLM_EVAL_MODE` + `APP_ENV=production`.

**Qué implementar:**
1. Campo en `Settings` (`app/config.py`), p. ej. `web_concurrency: int = 1`, que el despliegue fija
   para declarar cuántos workers corre.
2. `model_validator` que **falla al arrancar** si `web_concurrency > 1` (porque las sesiones son
   en-memoria / mono-proceso hoy). Mensaje que nombre la causa y apunte a la mejora futura.
3. Documentar el campo en `.env.example` con default `1` y un comentario de una línea.

**Límite declarado (ponerlo en el mensaje de error y en el `C2-decisions.md`):** es una convención,
no es hermético — caza al operador que **declara** la concurrencia, no al que pasa `--workers 4` sin
declararla, porque un worker no puede contar a sus hermanos. El fix robusto (store de sesiones
compartido, Redis/PG) queda diferido a M6 y lo documenta C1.

**Verificación:** un test que afirme que el arranque falla con `web_concurrency=2` y arranca normal
con `=1`.

---

## Tarea 4 — R12: docstring de `diagnose_numbering`

**Ref:** informe R12. `corpus/diagnostics/diagnose_numbering.py:18-19` dice que la resolución es
«to compare, not to replace the parser (that decision is Brief 5)». El brief 5 **decidió a favor**
del resolver, hoy en producción (`parser.py` numera desde `numbering.py`). Actualizar el docstring
para que describa la herramienta como diagnóstico del resolver en producción, no como una decisión
pendiente. Cambio de comentario, nada de lógica.

---

## Tarea 5 — RUNBOOK (R11 + R13)

**Ref:** informe R11, R13. `corpus/RUNBOOK.md`.
- **R11:** el paso 7 (`corpus.graph`) crasheaba; tras la Tarea 1 corre. Verificar que la entrada del
  paso 7 es correcta con el pipeline arreglado.
- **R13:** falta el paso que mete el corpus en la BD (`python -m app.services.ingestion`), el puente
  `corpus/` → `app/`. Añadirlo tras `embed_all`/`embedding_sanity`, con nota de su IO y del filtro
  de la Tarea 2 (descarta 103 marcadores de borrado; BD final 13.773).

---

## Tarea 6 — CHANGELOG (entrada de C2 + O3)

`CHANGELOG.md`. Redactar la entrada del trabajo de C2 (derivada de `C2-decisions.md`) y, de paso,
corregir **O3**: hay dos `## [0.26.0]` (líneas 249 y 303). Es el único fichero del histórico y no
puede tener una versión duplicada. C2 es el único brief que toca `CHANGELOG.md`.

---

## Salida de CC → chat

- `sessions/bridge-S08-S09/C2-decisions.md`: log crudo de qué se cambió y por qué, etiquetado
  agnóstico/dominio, con las dos notas de deuda que C1 debe documentar: **(a)** fix robusto de los
  chunks basura aguas arriba (parser/chunking), **(b)** store de sesiones compartido (M6). Este log
  y el CHANGELOG son el insumo de C1.
- Reportar el resultado de cada verificación. Lo que no se pueda cerrar (p. ej. si algo exige
  ground truth) se marca y se deja para su sitio, no se fuerza.
