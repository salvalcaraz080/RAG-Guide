# S08 · Brief 1 — Infraestructura pgvector + schema de la capa de datos

> **Producido por:** Claude Chat (decisión). **Ejecuta:** Claude Code.
> **Sesión:** S08 (M3 Data-driven, última sesión — levanta la capa de datos vectorial que el
> pipeline RAG de S09+ consumirá). **M1 CAG v0 hecho; S07 chunking+embeddings hecho.**
> **Alcance:** SOLO infra + schema. La ingesta de vectores es el Brief 2; el endpoint `/search`
> es el Brief 3. No implementes ninguno de los dos aquí.

---

## 0. Contexto y por qué (no solo el qué)

S08 introduce la persistencia vectorial. El artefacto de S07 (`embeddings.npy` + sidecar,
`text-embedding-3-small` @ 1536, fp32, **13.876 vectores**) está listo; este brief **no lo
carga** — solo prepara el sitio donde vivirá: Postgres + pgvector arriba, capa de acceso de
datos, y el schema `documents`/`chunks`.

**Decisión de arranque que gobierna todo el brief — régimen EXACTO, sin índice vectorial.**
A nuestra escala (13.876 hoy; proyección a corpus completo ~60-100k) la búsqueda exacta
(sequential scan, coseno) da recall 1.0 en milisegundos, y recall 1.0 es un requisito de
**trazabilidad**, no una preferencia: un vecino perdido por aproximación puede ser un requisito
perdido. Además el exacto es el baseline contra el que se medirá cualquier índice futuro.
Por tanto **este schema NO crea índice HNSW/IVFFlat**, y eso es correcto, no un "todavía no".
El paso a HNSW es un nodo futuro con disparador medido (p99 del seq scan > SLO), fuera de S08.

Consecuencia para ti: cuando más adelante ejecutes `EXPLAIN ANALYZE` sobre una query vectorial
(Brief 3), ver `Seq Scan` **es lo esperado y correcto**. No lo trates como bug ni añadas un
índice para "arreglarlo".

---

## 1. Objetivo del brief

Dejar el repo con:
1. Un servicio Postgres + pgvector en `docker-compose`, con la extensión `vector` habilitable.
2. Capa de acceso de datos: **SQLAlchemy async + Alembic + `pgvector.sqlalchemy`** (engine,
   sessionmaker, base declarativa, dependencia de sesión).
3. Modelos y migración inicial para el schema `documents`/`chunks` de §3.
4. `DATABASE_URL` cableada vía `Settings` (el patrón ya existe en `config.py`).
5. Docs (`CLAUDE.md`/`ARCHITECTURE.md`/`CHANGELOG.md`) actualizados.

**No** toca el camino de petición: `routers/`, `services/llm_service.py`, el seam
`get_context` y `context/reference.py` quedan **intactos**. La capa de datos se construye
desacoplada; se enchufará al seam en el Brief 3 (vía `services/retrieval.py`, que no existe aún).

---

## 2. Decisión de stack ya tomada (no re-abrir)

- **ORM/migraciones:** SQLAlchemy async + Alembic + `pgvector.sqlalchemy`. Razón: disciplina de
  migraciones sobre un schema que **crecerá** (tabla de refs, índice HNSW/halfvec, columnas
  nuevas); tipo `vector` soportado de fábrica; repositorio mockeable para los tests sin-Docker.
  Esto **no** contradice "cada capa desde cero" — ese principio rechaza frameworks RAG
  (LangChain/LlamaIndex), no un ORM de infraestructura.
- **Motor async:** `create_async_engine` + `async_sessionmaker` + driver `asyncpg`
  (`postgresql+asyncpg://...`). Coherente con FastAPI/ASGI del resto de `app/`.
- **Postgres:** imagen con pgvector (p. ej. `pgvector/pgvector:pg16` o equivalente). La extensión
  se habilita en la **migración inicial** (`CREATE EXTENSION IF NOT EXISTS vector`), no a mano.

---

## 3. Schema (esto es la decisión; tú lo mapeas a los dataclasses reales de S07)

Dos tablas, relación 1—N, `ON DELETE CASCADE`. Los nombres de columna de abajo son la
**intención**; mapéalos a los campos reales de `Chunk`/`ChunkMetadata`/`CiteRef` de S07. **Si
algún campo no mapea limpio, NO adivines: márcalo en `S08-decisions.md` y déjalo explícito.**

### `documents` — identidad a nivel documento (una fila por documento ECSS)

| Columna | Tipo | Notas |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `canonical_doc_id` | TEXT NOT NULL **UNIQUE** | direccionar/filtrar/grafo (`ECSS-E-ST-40C`) ← `ChunkMetadata.standard` (canónico) |
| `doc_id_cite` | TEXT NOT NULL | **citar**, con revisión (`ECSS-E-ST-40C Rev.1`) ← `CiteRef.standard` |
| `corpus_version` | TEXT NOT NULL | etiqueta del conjunto declarado |
| `source_path` | TEXT NOT NULL | procedencia (`data/processed/…json`) |
| `ingested_at` | TIMESTAMPTZ NOT NULL DEFAULT NOW() | |
| `metadata` | JSONB NOT NULL DEFAULT '{}' | enriquecimiento flexible a nivel doc |

**Por qué las dos identidades van aquí y no en `chunks`:** canónico y revisión son propiedades
del documento, no del chunk. Duplicarlas en cada uno de los ~13.876 chunks es exactamente la
duplicación que el modelo de dos tablas existe para evitar. La cita se ensambla por JOIN:
`documents.doc_id_cite` + `chunks.section_number`.

### `chunks` — la unidad recuperable y citable

| Columna | Tipo | Notas |
|---|---|---|
| `id` | BIGSERIAL PK | |
| `document_id` | BIGINT NOT NULL REFERENCES documents(id) **ON DELETE CASCADE** | |
| `chunk_id` | TEXT NOT NULL **UNIQUE** | id de S07 con ordinal (0 colisiones). **Clave natural para ingesta idempotente** (Brief 2 hará upsert por aquí) |
| `chunk_type` | TEXT NOT NULL | uno de: `requirement`,`section`,`table`,`note`,`glossary_term`. CHECK o enum PG |
| `authority` | TEXT NOT NULL | `normative`\|`informative`. CHECK o enum. **Es por chunk, no por doc** (una NOTE normativa sale como chunk informativo) |
| `section_number` | TEXT NOT NULL | coordenada de cita (`5.4.3.2`, `B.2.1`) — el `clause` del contrato de citación |
| `requirement_uid` | TEXT NULL | UID de requisito donde exista (68% del corpus no tiene) |
| `embed_text` | TEXT NOT NULL | **lo que se embebió**: header contextual + verbatim |
| `cite_text` | TEXT NOT NULL | **verbatim fuente, citable** ← `CiteRef.text`. **NUNCA derivar de `embed_text` ni de otro texto** |
| `embedding` | `vector(1536)` **NULL** | nullable a propósito (ver §4) |
| `input_fingerprint` | TEXT NOT NULL | invalidación anclada al contenido consumido (S07) |
| `metadata` | JSONB NOT NULL DEFAULT '{}' | enriquecimiento (tags, tecnologías inferidas…) |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT NOW() | |

### `embed_text` y `cite_text` son DOS columnas por construcción — invariante

Es la materialización en schema del principio `embed≠cite` de S07: `embed_text` lleva el header
(estándar + ruta de secciones) para que el chunk sea recuperable aislado; `cite_text` es el texto
normativo verbatim, sin nada añadido. **Deben ser columnas separadas y `cite_text` debe salir del
verbatim (`CiteRef.text`), jamás del `content`/`embed_text`.** Si colapsas ambas o derivas la cita
del texto embebido, reintroduces el vector de cita fantasma que S07 cerró. Guardamos las dos:
`cite_text` para citar/anclar, `embed_text` para reproducibilidad (re-embeber si cambia el modelo).

### Índices en este brief

- **GIN** sobre `chunks.metadata` y `documents.metadata` (filtros sobre JSONB).
- Los de PK/FK/UNIQUE que el ORM genera.
- **NINGÚN índice vectorial** (régimen exacto, §0).
- **No** añadas B-tree sobre `authority`/`chunk_type`/`section_number` todavía: los patrones de
  filtro se miden en S09/S10; añadirlos ahora es adivinar. Sencillez > sobreingeniería.

### Fuera del schema de S08 (no lo añadas)

- Tabla/columna de **referencias** (`outgoing_refs` sigue vacío hasta validar el extractor). El
  slot de refs es un brief posterior, gateado.
- `halfvec` (optimización del índice HNSW; sin índice no aplica). El schema de hoy **no lo
  cierra**: la columna es `vector(1536)` y halfvec viviría en el `CREATE INDEX` futuro.

---

## 4. `embedding` nullable — por qué, y qué NO asumir

Nullable a propósito: **la fila-chunk es la unidad citable y existe con su cláusula, su authority
y su verbatim aunque el embedding falte o aún no se calcule.** La identidad de cita no depende del
vector. Esto también prepara ingesta asíncrona futura.

Aviso para el Brief 2 (no lo implementes aquí, solo tenlo presente al diseñar el schema): las
docs citan "13.888 chunks" en un sitio y **13.876 vectores** (row-count real del `.npy`) en otro.
Si hay más chunks que vectores, esos chunks entran con `embedding = NULL` — la ingesta **debe
reconciliar conteo de chunks vs vectores y no asumir 1:1**. El schema nullable ya lo soporta.

---

## 5. Frontera CC / usuario (contrato de test)

Tu contrato es `uv run pytest` **mockeado, sin Docker, sin red**. Eso parte el trabajo en dos:

**Lo que TÚ verificas (pytest, sin DB):**
- Los modelos importan y se definen correctamente (columnas, tipos, constraints, relación
  1—N, cascade declarado).
- `Settings` carga `DATABASE_URL` (y falla al arrancar si falta, según el patrón existente).
- La sesión/engine se construyen sin tocar una DB real (mockea el engine si hace falta).
- **Alembic `autogenerate` NO está a tu alcance** (compara modelos contra una DB viva, que no
  tienes). **Escribe la migración inicial a mano** desde los modelos. No dejes un autogenerate a
  medias.

**Lo que queda para UAT del usuario (Docker, red — responsabilidad de Salva):**
- `docker compose up` levanta Postgres+pgvector.
- `alembic upgrade head` aplica la migración: extensión `vector` habilitada, tablas `documents`/
  `chunks` creadas con sus constraints, GIN presentes, **sin índice vectorial**.
- Verificación de que `\d chunks` muestra `embedding vector(1536)` nullable.

Entrega al final un **script UAT propuesto** (comandos exactos que Salva ejecutará) para validar
lo de arriba.

---

## 6. Docs a actualizar (parte del cierre, no opcional)

- **`CLAUDE.md`:** reescribe el **estado presente** (hoy dice "Estado actual: Módulo 1 / Sesión
  S06" — está desfasado). Debe reflejar: M1 CAG v0 hecho; S07 chunking+embeddings hecho; S08 en
  curso construyendo la capa de datos vectorial (pgvector, régimen exacto). **Reescribe en su
  sitio, no añadas narrativa de diario** (tu checklist lo prohíbe). El histórico va a `CHANGELOG.md`.
- **`ARCHITECTURE.md`:** añade la fila **ORM/capa de datos** al stack §6 (SQLAlchemy async +
  Alembic + `pgvector.sqlalchemy` + asyncpg); documenta el schema `documents`/`chunks` y la
  decisión de régimen exacto (sin índice, con su porqué); corrige el §7 que aún dice "estrategia
  de chunking sin decidir" (S07 la decidió).
- **Reconciliación menor:** `CLAUDE.md` numera módulos como el máster (RAG=M4) y `ARCHITECTURE §2`
  usa numeración propia (RAG=M2). Únifica o al menos deja una nota para que no se lean como
  contradicción. Baja prioridad.
- **`CHANGELOG.md`:** entrada para el trabajo de este brief.

---

## 7. Qué NO hacer (resumen)

- No cargar embeddings ni chunks (Brief 2).
- No crear endpoint `/search` ni tocar el seam `get_context` (Brief 3).
- No crear índice vectorial (HNSW/IVFFlat), ni halfvec.
- No crear tabla/columna de referencias poblada.
- No añadir B-tree de filtro especulativos.
- No colapsar `embed_text`/`cite_text` ni derivar `cite_text` del texto embebido.
- No duplicar la identidad de documento en `chunks`.
- No commitear (redacta el mensaje; commitea Salva). No stagear `.env`, DOCXs, CSVs, embeddings.

---

## 8. Entregables + criterios de aceptación

1. `docker-compose` con servicio Postgres+pgvector.
2. Capa de datos: engine async, sessionmaker, base, dependencia de sesión, modelos `documents`/
   `chunks`, migración inicial **escrita a mano** (extensión + tablas + constraints + GIN, sin
   índice vectorial).
3. `DATABASE_URL` en `Settings`.
4. `uv run pytest` verde sobre lo testeable sin DB (§5).
5. Docs actualizadas (§6).
6. Script UAT propuesto (§5) + mensaje de commit propuesto (`feat: …`).
7. `S08-decisions.md` con las decisiones crudas etiquetadas agnóstico/dominio, y **cualquier
   campo de S07 que no mapeara limpio al schema marcado explícitamente**.

Al terminar, sube `S08-decisions.md` al chat para la extracción (paso 4 del ciclo).
