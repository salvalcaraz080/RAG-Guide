# S08 · Brief 3 — Ingesta del artefacto de S07 en pgvector

> **Producido por:** Claude Chat (decisión). **Ejecuta:** Claude Code.
> **Alcance:** cargar `chunks.json` (ya con `requirement_uid`, Brief 2) + `embeddings.npy` en las
> tablas del Brief 1. Es un módulo/script standalone. **No** toca el camino de petición, **no**
> crea endpoint (Brief 4), **NO re-embebe**, **no** escanea `data/processed/`.
>
> **Renumeración recordatorio:** este es el Brief 3 (lo que el `decisions` del Brief 1 llamaba
> "brief 2"). `/search` es el Brief 4.

---

## 0. Por qué

La capa de datos existe (Brief 1) y el artefacto está completo y verificado (Brief 2:
`embed_text` invariante, `uid` poblado, vectores intactos). Este brief los une: **una pasada que
carga los 13.876 chunks y sus vectores en Postgres**, idempotente y con guardas que se niegan a
cargar un artefacto incoherente. Sigue en **régimen exacto** — no se crea índice vectorial.

El módulo de ingesta es standalone (p. ej. `scripts/ingest.py` o `app/ingest/…`); nadie del camino
de petición lo importa. El seam `get_context`/`context/reference.py` sigue intacto hasta el Brief 4.

---

## 1. Objetivo

1. Migración: añadir `chunks.oversized` (columna tipada — §3).
2. Módulo de ingesta que puebla `documents` y `chunks` desde `chunks.json` + sidecar + manifiesto.
3. Guardas de obsolescencia que **abortan ruidosamente antes de escribir** si los artefactos no
   son un par coherente (§5).
4. Idempotente: correr dos veces deja el mismo estado (§6).
5. Tests unitarios (sin DB) de la lógica pura + script UAT propuesto para la carga real (§8).

---

## 2. Fuera de alcance (no lo hagas)

- Endpoint `/search`, camino de petición, seam (Brief 4).
- Re-embeber / tocar `embeddings.npy` o el sidecar.
- Escanear `data/processed/` (heredaría el defecto de `S08-brief-2-decisions.md` §5.1 — ver §4).
- Índice vectorial, `halfvec`, tabla/columna de refs poblada.
- Refactor del manifiesto de los 6 módulos (§5.1 del Brief 2) — diferido a brief propio tras
  Brief 4. **No lo abras aquí.**
- Arreglar el `[0.26.0]` duplicado, el huérfano `rag-ecss-redis`, `EMBEDDING_PROVIDER`.
- Borrar filas huérfanas en re-ingesta (ver nota en §6).

---

## 3. Migración: `chunks.oversized` tipada

Nueva revisión Alembic (escrita a mano; `autogenerate` necesita DB viva): `ALTER TABLE chunks ADD
COLUMN oversized BOOLEAN NOT NULL DEFAULT false`.

**Por qué tipada y no JSONB**, saltándome la regla "tipado = load-bearing para la cita": `oversized`
no sostiene la cita (el `cite_text` verbatim está intacto se recupere o no) — sostiene la
**fidelidad del vector**: marca chunks cuyo embedding puede no representar todo el `cite_text`. En
un sistema cuyo valor es la trazabilidad, un defecto conocido del artefacto no debe vivir blando en
un blob donde un análisis futuro pueda no comprobarlo; tipado es visible en `\d chunks` e
imposible de olvidar, y **consultable** justo donde hará falta (el fallo de retrieval B/C está
abierto para S09: poder preguntar "¿algún miss de recall cae en un chunk con embedding degradado?"
sin gimnasia JSONB es diagnóstico real).

**Confírmalo antes de dar por buena la semántica:** doy por hecho que `oversized` significa "el
vector no cubre todo el `cite_text`". No está verificado (solo sé que son 2 casos marcados, Brief 1
§1.2). **Lee qué marca realmente ese flag en el artefacto, y documéntalo.** Si resultara benigno
(p. ej. se dividió sin pérdida), la columna sigue costando ~0; si es degradación real, se gana el
sitio. Reporta cuál es.

---

## 4. De dónde sale cada tabla — `documents` desde `chunks.json`, NO desde el directorio

**`documents`:** deriva el conjunto de las **identidades distintas presentes en `chunks.json`**
(cada chunk trae canónica `ChunkMetadata.standard` + cita `CiteRef.standard`). Es la lista de docs
que **de verdad tienen contenido**, y evita escanear `data/processed/` — así la ingesta no hereda
el defecto de §5.1 (directorio que es entrada de un stage y salida de otro).

- Verifica **canónica→cita 1:1** al derivar (Brief 1 §1.4 lo comprobó en los 26; re-comprúebalo,
  no lo asumas). Si una canónica apuntara a dos citas → **aborta**: rompería `UNIQUE(canonical_doc_id)`
  y la integridad de la cita.
- `source_path` sale del **manifiesto** (join por `canonical_doc_id`). Si un doc de `chunks.json`
  no está en el manifiesto → **aborta** (integridad).
- `corpus_version` sale del **sidecar** (es la etiqueta del conjunto que produjo estos vectores);
  mismo valor en las 26 filas.

**`chunks`:** una fila por chunk de `chunks.json`. El vector se empareja **por orden de fila** con
`embeddings.npy` (Brief 2 verificó que las 13.876 emparejan). Resuelve `document_id` desde el mapa
`canonical_doc_id → documents.id` poblado en el paso anterior (documents antes que chunks, por la FK).

---

## 5. Guardas de obsolescencia — ANTES de escribir una sola fila

La ingesta debe **negarse a cargar un artefacto incoherente**, ruidosamente, sin escribir nada. Es
el eco en la ingesta de "se mide, no se asume". Lee el sidecar y `chunks.json` y comprueba:

1. **Par coherente:** el `input_fingerprint` de `chunks.json` coincide con el `chunks_fingerprint`
   del sidecar (`ab3fce8b5cb1a0d4` hoy — Brief 2 §4). Si no → abortar.
2. **Conteo:** `embeddings.npy.shape[0] == len(chunks)` de `chunks.json`. Si no → abortar.
3. **Dimensión y modelo:** `shape[1] == 1536` y el modelo/dims del sidecar son los esperados
   (`text-embedding-3-small` @ 1536). Si no → abortar.

Cualquier fallo aborta con un mensaje que diga **qué** no cuadró (valores esperados vs vistos), no
un stack trace. Nada se escribe si una guarda salta.

---

## 6. Idempotencia

Upsert por claves naturales: `documents` por `ON CONFLICT (canonical_doc_id) DO UPDATE`, `chunks`
por `ON CONFLICT (chunk_id) DO UPDATE`. `BIGSERIAL` no se regenera en conflicto → los `id` quedan
estables y las FK no se rompen al re-correr. Correr dos veces → 26 documents / 13.876 chunks, sin
duplicados.

**Límite conocido, no lo resuelvas aquí:** el upsert cubre alta y actualización, **no borra filas
que desaparezcan** de un artefacto futuro (corpus mutado). Hoy el corpus es estático dentro de una
`corpus_version`, así que no aplica; cuando el corpus mute (borrado de chunks entre versiones)
habrá que reconciliar por presencia. Anótalo, no lo construyas.

---

## 7. Reglas de poblado por columna

- **`requirement_uid`:** ya viene en `chunks.json` (Brief 2). Cárgalo tal cual; respeta `NULL`
  donde no hay (Brief 2 usó `or None` para no confundir `""` con "tiene UID vacío"). No re-derives.
- **`embed_text` / `cite_text`:** dos columnas, cada una de su campo. `cite_text` **jamás** del
  `embed_text`. (Invariante del Brief 1.)
- **`embedding`:** por orden de fila. Hoy las 13.876 tienen vector (1:1), así que ninguna queda
  `NULL`; el nullable no se ejercita hoy pero **el código no debe asumir 1:1** — si tras la guarda
  de conteo (que hoy pasa) faltara un vector, esa fila entra con `NULL`, no revienta.
- **`oversized`:** desde el flag del artefacto (§3).
- **`token_count`:** **no se persiste** (propiedad de ventana, re-derivable — Brief 1 §1.2).
- **`revision`:** vive en `documents.doc_id_cite`, **no** en el chunk.
- **`metadata` JSONB (chunks y documents):** solo enriquecimiento **genuino** que no sea ya una
  columna tipada ni esté prohibido. **Prohibido explícitamente:** `outgoing_refs` (gateado hasta
  validar el extractor), `anchor` (es `chunk_type`), `revision`/`corpus_version` (van a `documents`),
  `token_count`. Si hoy no queda nada que califique, déjalo `{}` — **no inventes contenido del que
  aún no hay contrato**. Reporta qué metiste (probablemente nada).

---

## 8. Frontera de test

**Unit (pytest, sin DB — tuyo):**
- Guardas de §5: pasan con par coherente; abortan con fingerprint desalineado / conteo distinto /
  dim o modelo distintos (entradas sintéticas).
- Derivación de `documents`: canónica→cita 1:1; **aborta** si una canónica sintética apunta a dos
  citas; aborta si un doc no está en el manifiesto.
- Transformación chunk→fila: `requirement_uid or None`, `oversized`, separación `embed_text`/
  `cite_text`, JSONB sin campos prohibidos.
- Emparejamiento vector↔chunk por orden de fila (set sintético pequeño).
- Construcción del upsert (sesión mockeada), no ejecución real.

**UAT (Docker, Salva):**
- `alembic upgrade head` aplica la migración de `oversized`.
- Correr la ingesta contra pgvector real.
- Conteos: 26 documents, 13.876 chunks, 13.876 embeddings no-nulos, dim 1536.
- Split authority en DB: 10.320 normative / 3.556 informative; conteos por ancla como Brief 1.
- `oversized`: los N que confirmes (2 esperados) en `true`, resto `false`.
- **Idempotencia:** correr dos veces → mismos conteos, cero duplicados.
- **Guarda:** apuntar a un fingerprint/artefacto desalineado → aborta sin escribir.
- Spot check: un chunk `requirement` conocido → `requirement_uid`, `section_number`, `authority`,
  y `doc_id_cite` por JOIN, correctos.

Entrega el script UAT con estos comandos.

---

## 9. Qué NO hacer (resumen)

- No re-embeber; no tocar `embeddings.npy`/sidecar.
- No escanear `data/processed/`; deriva `documents` de `chunks.json`.
- No crear endpoint ni tocar el camino de petición.
- No índice vectorial, no `halfvec`, no refs pobladas.
- No inventar contenido de JSONB.
- No borrar huérfanos en re-ingesta (solo anotar el límite).
- No abrir el refactor del manifiesto ni la deuda diferida.
- No commitear (redacta el mensaje; commitea Salva). No stagear `.env`, artefactos, embeddings.

---

## 10. Docs

- **`ARCHITECTURE.md`:** módulo de ingesta; las guardas de obsolescencia como invariante; columna
  `oversized` y qué marca; y la decisión de derivar `documents` de `chunks.json` (no del directorio)
  con su porqué (§5.1). Régimen exacto sigue vigente (sin índice).
- **`CHANGELOG.md`:** entrada nueva (siguiente versión; no repitas el `[0.26.0]` duplicado).
- **`CLAUDE.md`:** estado presente si cambia (ahora hay datos ingeribles en pgvector). Sin diario.

---

## 11. Entregables + aceptación

1. Migración `oversized` (a mano) aplicable.
2. Módulo de ingesta con guardas (§5), derivación de `documents` desde `chunks.json` (§4), upsert
   idempotente (§6), reglas de poblado (§7).
3. `uv run pytest` verde sobre lo testeable sin DB.
4. Script UAT (§8) + mensaje de commit propuesto.
5. Docs (§10).
6. `S08-brief-3-decisions.md`: decisiones crudas etiquetadas agnóstico/dominio, **qué marca
   realmente `oversized`**, qué (si algo) entró al JSONB, y cualquier sorpresa. Súbelo al chat para
   la extracción (paso 4).
