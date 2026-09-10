# C2 — Log de decisiones (fixes transversales de código)

**Insumo de C1.** Log crudo de qué se cambió y por qué. Etiquetado **agnóstico** (vale para
cualquier proyecto) / **dominio** (específico de RAG-ECSS).

**Estado:** las seis tareas cerradas y verificadas. `uv run pytest` → **854 passed, 8 deselected**
(desde 837: +17 tests nuevos). Árbol listo, **sin commitear**.

**Nota de ruta:** el brief escribe a `sessions/bridge-S08-S09/`; ese directorio se renombró a
`sessions/Auditoria pre M4/`, que es donde está este log y el informe de auditoría
(`initital-audit-pre-M4-report.md`).

---

## 0 · Cuatro correcciones al brief, aprobadas antes de implementar

| # | El brief decía | El repo dice | Resolución |
|---|---|---|---|
| a | Filtrar **103** chunks, y «103 exacto o no sigas» | El predicado del propio brief caza **121** | Gate a **121**. Ver §2 |
| b | «Si `ingestion` no es idempotente (append), truncar» | **Sí** es idempotente; por eso los huérfanos sobreviven | `TRUNCATE` + recarga. Ver §2 |
| c | «Diff de `references.json` regenerado vs **commiteado**» | `data/processed/` está en `.gitignore`: no hay versión commiteada | Copia previa + diff contra ella. Ver §1 |
| d | Sin lista negra, en todos los módulos | El `unexpected` de `report.py` **necesita** saber qué es un artefacto legítimo | Una constante central. Ver §1 |

---

## 1 · Tarea 1 — el conjunto de documentos sale del manifiesto (D13, R8, R11)

### Qué se cambió

- **Accesor único** en `corpus/manifest.py`: `expected_document_ids(manifest)` y
  `document_json_paths(manifest, processed_dir)`, más la constante `DERIVED_ARTEFACTS`.
- **Seis módulos migrados**, ninguno escanea ya el directorio para saber qué es el corpus:
  `chunk_all.py`, `graph.py`, `report.py`, `diagnostics/verify_parser.py`,
  `diagnostics/audit_raw_corpus.py`, `diagnostics/inspect_graph.py`.
- Eliminada la lista `_NOT_DOCUMENTS` de `chunk_all.py` y las cinco exclusiones sueltas
  (`!= "references.json"`) de los demás.
- `graph.py` y `inspect_graph.py` ganan manejo de `ManifestError`: sin manifiesto salen con código
  2 y un mensaje, no con un traceback.

### Decisiones tomadas durante la implementación

**[agnóstico] El accesor recibe el manifiesto explícito, no lo carga por su cuenta.** Sigue el
estilo que ya tenía `report.build_report(manifest, processed_dir)`. Efecto lateral necesario: los
tests pueden inyectar un manifiesto sintético y seguir corriendo sin `data/` ni red, que es
invariante del proyecto. La alternativa (que el accesor hiciera `load()` internamente) habría
hecho que un test con `tmp_path` leyese el manifiesto real de 26 documentos.

**[agnóstico] El accesor ORDENA, en vez de heredar el orden de declaración del manifiesto.** Es la
decisión con más consecuencia y la menos visible. `chunk_all.input_fingerprint` digiere esas rutas
**en orden**, así que un manifiesto reordenado habría cambiado el fingerprint sin que cambiara ni
un byte de contenido → el par con `embeddings.json` se rompe → la guarda 1 de la ingesta refusa →
re-embedding de los 13.876 vectores. Ordenar en el accesor lo hace imposible por construcción. El
orden es propiedad del accesor, no del fichero de configuración.

*Verificado antes de tocar nada:* el orden de declaración del manifiesto coincide **hoy** con el
alfabético, y ambos reproducen `ab3fce8b5cb1a0d4`. Era una coincidencia, y ahora no hace falta que
se mantenga.

**[agnóstico] `report.py` sigue escaneando el directorio, a propósito.** Su `unexpected` hace la
pregunta **contraria** a la del accesor: «¿qué hay aquí que el manifiesto no declara?». Contra un
conjunto derivado del manifiesto esa respuesta sería siempre vacía y la señal moriría. Filtra por
`DERIVED_ARTEFACTS`, una constante central en `corpus/manifest.py`.

> Desviación consciente de la letra del brief («por construcción, no por lista negra»). Lo que la
> decisión compra es **una** fuente de verdad en lugar de las seis que divergieron; para el
> conjunto del corpus se cumple literalmente (los artefactos derivados no pueden ser nombrados
> porque el manifiesto no los declara), y para la señal inversa hace falta un complemento. **C1
> debería documentar `DERIVED_ARTEFACTS` como el sitio donde se registra un artefacto nuevo de
> `processed/`.**

**[dominio] Cambio semántico deliberado, con test.** Un JSON que el manifiesto no declara ya **no
invalida** los chunks: antes, cualquier fichero caído en `processed/` entraba en el digest. El test
`test_stale_chunks_are_detected` fallaba justamente por esto; se reescribió para expresar la
staleness como la define ahora el corpus (el documento nuevo entra **declarándose**), y se añadió
`test_undeclared_json_does_not_invalidate_the_build` para cubrir la propiedad nueva.

### Verificación

| Comprobación | Antes | Ahora |
|---|---|---|
| `python -m corpus.graph` | `KeyError: 'sections'` | 26 docs, 329 refs, 384 nodos, 299 aristas |
| `python -m corpus.diagnostics.inspect_graph` | `KeyError: 'sections'` | `OK - all checks passed` |
| `verify_parser` | «EN processed/ SIN FILA ACTIVA (2)» | «Sin brecha», 26/26, 100% |
| `audit_raw_corpus` | parseados 28, **cobertura 107%** | parseados 26, cobertura 100% |
| `build_report()` | `unexpected: ['chunks','embeddings']` | `unexpected: []` |
| `input_fingerprint` | `ab3fce8b5cb1a0d4` | `ab3fce8b5cb1a0d4` (sin re-embedding) |

**Diff de artefactos regenerados:** `references.json` **idéntico** —329 registros, mismo conjunto y
mismo orden— y `graph.graphml` con el **mismo SHA-256** (`787fac487414162a`). El fix hace que el
stage **corra**; no cambia lo que produce. Copia previa en el scratchpad de la sesión.

---

## 2 · Tarea 2 — las cláusulas retiradas no entran en la base (D15)

### Qué se cambió

En `app/services/ingestion.py`: `is_retracted(chunk)`, `drop_retracted(artefacts)`, la constante
`EXPECTED_RETRACTED = 121` y el campo `retracted` en `IngestionReport`. Llamado desde `ingest()` y
desde la rama `--dry-run`, en ambos casos **después** de `check_guards`.

### La corrección del número: 103 → 121

**[dominio] El brief tomó el número de un instrumento distinto del criterio que argumenta.**

- **103** = chunks con `cite_tsv` vacío. Mide *ceguera del canal léxico*. Es lo que midió la
  auditoría, y con la herramienta con la que lo midió (`psql`) es correcto.
- **121** = chunks cuyo verbatim es un marcador de retirada. Es lo que mide el **predicado** del
  brief, y lo que corresponde al motivo declarado de la tarea: «un `<<deleted>>` citado como fuente
  rompería la trazabilidad».

Verificado en BD: `103 ⊂ 121`, contención exacta, **0 elementos de 103 fuera de 121**. Los 18 de
diferencia son variantes con texto (`<<deleted, covered by clause 5.4.1.2>>`,
`<<deleted, requirement covered by 5.2.7.1a>>` ×6…) que **sí producen lexemas**: el canal léxico
puede recuperarlas activamente. Son las **más** peligrosas, no las más inocuas — que es
exactamente lo contrario de lo que sugería el número más bajo.

Revisados los 121 por longitud: el más largo son 58 caracteres y es marcador puro. **No se pierde
contenido real.** Ningún documento se queda sin chunks (el más afectado, `ECSS-E-HB-40A`, pasa de
497 a 446). Reparto: `ECSS-E-HB-40A` 51, `ECSS-E-ST-10C` 19, `ECSS-E-ST-10-02C` 19,
`ECSS-Q-ST-20C` 13, `ECSS-E-ST-10-03C` 12, `ECSS-Q-ST-40C` 4, `ECSS-Q-ST-80C` 3.

### Decisiones tomadas durante la implementación

**[agnóstico] El filtro va después de las guardas y no toca los ficheros.** Es política de
**carga**. Hay una razón mecánica además de la conceptual: la guarda 4 compara
`sidecar["chunk_ids"]` con la lista de chunks, así que filtrar antes la haría saltar. El sidecar
sigue describiendo el artefacto entero; hay un test que lo fija.

**[agnóstico] Se filtra por índice, chunk y vector juntos.** `build_rows` empareja
`artefacts.vectors[index]` con `chunks[index]`, así que descartar un lado daría a cada
superviviente el vector de su vecino: una base bien formada y falsa, el único fallo que nada aguas
abajo detecta. El test `test_surviving_chunks_keep_their_own_vector` usa vectores donde cada fila
lleva su índice como valor, para que un desplazamiento se vea en el número.

**[agnóstico] La cifra es una especificación, no un contador.** Si el filtro descartase otra
cantidad, la ingesta **para** y dice qué mirar. El coste conocido: `EXPECTED_RETRACTED` hay que
actualizarlo cuando cambie el corpus. Se aceptó porque el fallo es ruidoso y auto-explicativo, y
porque encaja con la disciplina del módulo («se niega antes de escribir»). Va anclado en
comentario a `corpus_version 2026-08-04.91bc8116`.

**[dominio] `TRUNCATE` + recarga, no borrado dirigido.** El brief suponía que la ingesta podía no
ser idempotente; **lo es**, y por eso mismo el upsert no borra y las 121 filas ya cargadas habrían
sobrevivido a la recarga — la BD se habría quedado en 13.876 con los marcadores intactos, que es
justo lo que la tarea quiere evitar. Se descartó añadir un `DELETE` a `ingestion.py` porque sería
implementar por la puerta de atrás la «reconciliación por presencia» que está **diferida** hasta
que el corpus deje de ser estático. `TRUNCATE` operativo mantiene esa decisión diferida y es
barato: la BD es derivada y se reconstruye del artefacto sin llamadas a API.

### Verificación

| | Antes | Ahora |
|---|---|---|
| chunks | 13.876 | **13.755** |
| documents | 26 | 26 |
| `cite_tsv` vacío | 103 | **0** |
| marcadores de retirada | 121 | **0** |
| sin vector | 0 | 0 |
| aviso `stale_rows_present` | — | no salta (`stored` = `ingested`) |

**Alineación intacta:** cinco vectores de control, tres de ellos tomados **justo detrás** de
marcadores retirados en el orden del artefacto (filas 347, 834 y 13.609), con `md5(embedding)`
byte-idéntico antes y después.

---

## 3 · Tarea 3 — guardarraíl de concurrencia (O7 / D25)

**Qué se cambió:** campo `WEB_CONCURRENCY: int = 1` en `Settings`, `field_validator` que rechaza
`> 1` al arrancar, entrada en `.env.example`, y 4 tests en `tests/app/test_config.py`.

**[agnóstico] Dos desviaciones del brief, ambas al alza:**

1. **`WEB_CONCURRENCY` en mayúsculas**, no `web_concurrency`. Todos los campos de `Settings` son
   mayúsculas; minúsculas rompía el estilo.
2. **Ese nombre exacto hace el guardarraíl mejor de lo que el brief asume.** `WEB_CONCURRENCY` es
   la variable que **uvicorn lee de verdad** para fijar `--workers` cuando no se le pasa el flag
   (verificado en la 0.47.0 instalada, `uvicorn/config.py:339-340`). Así que un contenedor que
   declare `WEB_CONCURRENCY=4` arranca 4 workers reales **y** revienta al arrancar. Deja de ser
   una convención que solo caza al operador honesto: caza la vía estándar de despliegue.

**[agnóstico] El límite que queda, y va en el mensaje de error:** `uvicorn --workers 4` en la línea
de comandos no pasa por `Settings`, porque un worker no puede contar a sus hermanos. El mensaje lo
dice explícitamente — un guardarraíl que no explica su propio alcance es uno que el operador
desactiva.

Hay un test (`test_the_refusal_names_the_cause_not_just_the_rule`) que fija que el mensaje nombra
la causa, no solo la regla.

---

## 4 · Tareas 4, 5 y 6 — documentación operativa

**[dominio] R12** — docstring de `corpus/diagnostics/diagnose_numbering.py`. Decía que resolver la
lista era «to compare, not to replace the parser (that decision is Brief 5)»; el brief 5 decidió a
favor y el resolver está en producción. Reescrito para decir que el contador que simula es el
**baseline** que justifica el coste del resolver, y que por eso sigue ahí. Sin cambio de lógica.

**[dominio] R11** — paso 7 del RUNBOOK: dejaba de ser cierto que lee `data/processed/*.json`.
Ahora dice que el conjunto sale del manifiesto, con la precondición corregida (pasos **3 y 4**, no
solo el 4) y una nota de por qué el glob mataba el paso.

**[agnóstico] R13** — **paso 12 (`ingestion`)** añadido al RUNBOOK, y a la secuencia de cabecera.
Es el único paso que cruza la frontera `corpus/` → `app/` y no estaba en ningún runbook. Documenta
IO, precondiciones, las cinco guardas, el filtro de retiradas, la verificación esperada
(`stored_chunks=13755`, `retracted=121`) y —esto es lo que faltaba operativamente— **el límite del
upsert**: no borra, así que reducir el conjunto cargable exige `TRUNCATE` antes. El comando de
verificación `psql` se probó tal y como está escrito.

**[agnóstico] O3** — el CHANGELOG tenía **dos `## [0.26.0]`** y además desordenadas: S07 brief 6
(09-04) estaba **debajo** del brief 5 (09-01) en un fichero newest-first. El brief 6 pasa a
`0.26.1` y sube a su sitio. Se descartó renumerar en cascada (brief 6 → 0.27.0 y correr las cinco
entradas de S08) por no reescribir seis entradas del histórico. Comprobado antes: no hay tags de
versión y nadie cita `0.26.0` fuera del CHANGELOG. Resultado: **33 entradas, sin duplicados,
estrictamente descendentes.**

**Entrada `[0.32.0]`** redactada con el trabajo de C2.

---

## 5 · Las dos notas de deuda que C1 debe documentar

**(a) [dominio] El filtro de retiradas es el fix barato; el robusto sigue pendiente.** Los 121
marcadores **siguen siendo chunks**: están en `chunks.json`, tienen `embed_text`, tienen vector en
`embeddings.npy` y ocupan sitio en el fingerprint. Lo único que cambia es que no se cargan. El fix
estructural es que no lleguen a ser chunks —tratar los marcadores de borrado en
`corpus/parser.py` / `corpus/chunking.py`—, y **eso sí dispararía re-chunking y re-embedding**, que
es exactamente por lo que se difirió. Documentar como deuda con esa consecuencia explícita: no es
«pendiente de limpiar», es «pendiente de una operación cara».

**(b) [agnóstico] El guardarraíl de workers es una convención reforzada, no un cerrojo.** Caza la
concurrencia declarada por entorno (que es la vía estándar, y de ahí que sirva), no `--workers N`
en línea de comandos. El fix real es un **store de sesiones compartido** (Redis/PG), diferido a
**M6**. Documentar el guardarraíl como mitigación con alcance nombrado, no como solución de D25 —
D25 sigue `VIGENTE`.

---

## 6 · Hallazgo lateral, no reparado

**[dominio] Dos fuentes de «qué debería existir» en la misma reconciliación.** `verify_parser` y
`audit_raw_corpus` reconcilian contra el **inventario** (`load_active_documents(INVENTORY_PATH)`),
mientras `parse_all` y `report` lo hacen contra el **manifiesto**. Tras C2, el lado «parseado» de
esos dos diagnósticos sale del manifiesto y el lado «esperado» sigue saliendo del inventario.

Funciona hoy porque ambos listan los mismos 26 documentos, y la cadena de identidad del proyecto
sitúa el inventario **antes** de la derivación humana, así que puede ser deliberado para
`audit_raw_corpus` (que audita `data/raw/`) y discutible para la sección de cobertura de
`verify_parser`. Añadido: `data/inventory/` está en `.gitignore`, así que esos dos diagnósticos
solo corren en una máquina que haya scrapeado.

**No estaba en el informe de auditoría ni en el brief. No se ha tocado.** Es decisión de Claude
Chat si merece un item propio.

---

## 7 · Ficheros tocados

**Código** — `corpus/manifest.py`, `corpus/chunk_all.py`, `corpus/graph.py`, `corpus/report.py`,
`corpus/diagnostics/verify_parser.py`, `corpus/diagnostics/audit_raw_corpus.py`,
`corpus/diagnostics/inspect_graph.py`, `corpus/diagnostics/diagnose_numbering.py`,
`app/services/ingestion.py`, `app/config.py`.

**Tests** — `tests/corpus/test_chunk_consumption.py`, `tests/app/test_ingestion.py`,
`tests/app/test_config.py`. *(El brief no listaba `tests/` entre los ficheros tocables, pero
mandaba un test nuevo en la Tarea 3 y cambiar una firma obliga a actualizar sus tests. Se
interpretó como omisión, no como prohibición.)*

**Docs y config** — `corpus/RUNBOOK.md`, `CHANGELOG.md`, `.env.example`.

**No tocados, por estar fuera de alcance:** `.claude/CLAUDE.md`, `ARCHITECTURE.md`, el `.env`
local (son de C1), `app/main.py` (estaba permitido pero ninguna tarea lo necesitaba), y el
extractor de referencias (D30/D31/D32: C2 hace que `graph.py` **corra**, no que acierte más).

**Sin commitear.** Mensajes propuestos en la respuesta de CC, uno por tarea independiente.
