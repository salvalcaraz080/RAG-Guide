# S08 · Brief 1 — Guion de UAT

> **Por qué es UAT y no verificación de CC:** caso 1 de la frontera de `CLAUDE.md` — Docker
> Desktop no está arrancado en la máquina y CC no lanza aplicaciones de escritorio por su
> cuenta. Todo lo demás (suite de contrato, lint, DDL de la migración, validez del compose,
> arranque de la app) ya está ejecutado y verde.
>
> **Ejecutado por CC el 2026-09-08** tras arrancar Salva Docker Desktop: los 8 pasos verdes.
> Cada bloque está verificado **en PowerShell 5.1**, que es el shell donde se corre — no
> traducido desde otro shell. Se conserva como guion de re-ejecución.

Ejecutar desde `C:\Users\salca\proyectos\RAG-ECSS`.

---

## 1. Levantar la base de datos

```powershell
docker compose up -d db
docker compose ps db
```

**Esperado:** estado `running (healthy)`. La primera vez se descarga la imagen
`pgvector/pgvector:pg16` (~450 MB).

---

## 2. Aplicar la migración

```powershell
uv run alembic upgrade head
uv run alembic current
```

**Esperado:** `0001_initial_vector_schema (head)`.

---

## 3. La extensión está habilitada

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "\dx vector"
```

**Esperado:** una fila `vector | <versión> | public | vector data type...`.

---

## 4. Las tablas, con sus tipos y constraints

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "\d documents"
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "\d chunks"
```

**Esperado en `chunks`** (lo que hay que mirar con lupa):
- `embedding | vector(1536) |` — **sin** `not null`
- `embed_text | text | not null` y `cite_text | text | not null` como **dos columnas distintas**
- `metadata | jsonb | not null default '{}'`
- Check constraints: `ck_chunks_authority` y `ck_chunks_chunk_type`
- Foreign-key: `fk_chunks_document_id ... ON DELETE CASCADE`

**Esperado en `documents`:** `uq_documents_canonical_doc_id UNIQUE`, y **ninguna** columna
`authority` ni `embedding`.

---

## 5. Los índices: GIN sí, vectorial no

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT indexname, indexdef FROM pg_indexes WHERE schemaname = 'public' ORDER BY indexname;"
```

**Esperado — estos siete del schema, más `alembic_version_pkc` (que es de Alembic, no del
schema). Ni uno más:**

| índice | tipo |
|---|---|
| `ix_chunks_document_id` | btree |
| `ix_chunks_metadata` | **gin** |
| `ix_documents_metadata` | **gin** |
| `pk_chunks`, `pk_documents` | btree (PK) |
| `uq_chunks_chunk_id`, `uq_documents_canonical_doc_id` | btree (UNIQUE) |

**Criterio de fallo:** cualquier `USING hnsw` o `USING ivfflat`. No debe haber ninguno — es el
régimen exacto, no un olvido.

---

## 6. El schema se comporta: cascade, CHECK y el tipo `vector`

Un solo bloque; deja la base como la encontró.

```powershell
# Here-string de PowerShell: dentro de @'...'@ las comillas simples son literales, no hay que
# escaparlas. El cierre '@ va pegado al margen izquierdo, sin sangría.
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c @'
INSERT INTO documents (canonical_doc_id, doc_id_cite, corpus_version, source_path)
VALUES ('UAT-DOC', 'UAT-DOC Rev.1', 'uat', 'data/processed/uat.json');

INSERT INTO chunks (document_id, chunk_id, chunk_type, authority, section_number,
                    embed_text, cite_text, input_fingerprint)
SELECT id, 'uat-1', 'requirement', 'normative', '5.4.3.2',
       'header + verbatim', 'verbatim', 'fp'
FROM documents WHERE canonical_doc_id = 'UAT-DOC';

-- El vector entra y sale: 1536 dimensiones, tipo real, columna nullable.
UPDATE chunks SET embedding = array_fill(0.1::real, ARRAY[1536])::vector WHERE chunk_id = 'uat-1';
SELECT chunk_id, vector_dims(embedding) AS dims, embedding IS NULL AS sin_vector FROM chunks;
'@
```

**Esperado:** `uat-1 | 1536 | f`.

### 6.1 El CHECK rechaza un ancla inventada

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "INSERT INTO chunks (document_id, chunk_id, chunk_type, authority, section_number, embed_text, cite_text, input_fingerprint) SELECT id, 'uat-2', 'paragraph', 'normative', '1.1', 'e', 'c', 'fp' FROM documents WHERE canonical_doc_id = 'UAT-DOC';"
```

**Esperado: ERROR** — `new row for relation "chunks" violates check constraint
"ck_chunks_chunk_type"`. Que falle **es** el resultado correcto.

### 6.2 El cascade borra los chunks del documento

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "DELETE FROM documents WHERE canonical_doc_id = 'UAT-DOC'; SELECT count(*) AS chunks_huerfanos FROM chunks;"
```

**Esperado:** `0`.

---

## 7. La migración es reversible

```powershell
uv run alembic downgrade base
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "\dt"
uv run alembic upgrade head
```

**Esperado:** tras el `downgrade`, solo queda `alembic_version` (la extensión `vector`
**sigue ahí** a propósito: puede estar en uso por otro schema). Tras el `upgrade`, las dos
tablas otra vez.

---

## 7 bis. El engine async conecta de verdad

Lo único que la suite de contrato no puede afirmar, porque no tiene base de datos: que
`asyncpg` + el ORM + el tipo `vector` funcionan de extremo a extremo.

**Por fichero, no con `python -c`.** PowerShell **elimina las comillas dobles** al pasar
argumentos a un ejecutable nativo, así que un script Python inline llega con las cadenas sin
comillas y revienta con `SyntaxError: invalid decimal literal`. Un `Set-Content` no sufre eso
porque es un cmdlet, no un `.exe`.

```powershell
$py = @'
import asyncio
import sys

# El script vive en TEMP, así que la raíz del proyecto no está en sys.path. El cwd SÍ es la
# raíz (de ahí sale también el .env que lee Settings), así que basta con anteponerlo.
sys.path.insert(0, ".")

from sqlalchemy import func, select

from app.db import get_sessionmaker
from app.models import Chunk, Document


async def main():
    async with get_sessionmaker()() as s:
        doc = Document(canonical_doc_id="ECSS-E-ST-40C", doc_id_cite="ECSS-E-ST-40C Rev.1",
                       corpus_version="uat", source_path="data/processed/uat.json")
        doc.chunks.append(Chunk(chunk_id="uat#5.4.3.2::r0", chunk_type="requirement",
                                authority="normative", section_number="5.4.3.2",
                                embed_text="header + verbatim", cite_text="verbatim",
                                input_fingerprint="fp", embedding=[0.1] * 1536))
        s.add(doc)
        await s.commit()
        print("cita por JOIN:",
              (await s.execute(select(Document.doc_id_cite, Chunk.section_number,
                                      Chunk.cite_text).join(Chunk))).one())
        await s.delete(doc)
        await s.commit()
        print("chunks tras borrar:",
              (await s.execute(select(func.count()).select_from(Chunk))).scalar())


asyncio.run(main())
'@
$path = Join-Path $env:TEMP "uat_data_layer.py"
Set-Content -Path $path -Value $py -Encoding utf8
uv run python $path
```

**Esperado** (verificado en PowerShell 5.1, no solo traducido):

```
cita por JOIN: ('ECSS-E-ST-40C Rev.1', '5.4.3.2', 'verbatim')
chunks tras borrar: 0
```

La cita ensamblada por JOIN es el punto del modelo de dos tablas; el `0` es el cascade por el
camino ORM.

---

## 8. Cerrar (opcional)

```powershell
docker compose stop db        # conserva los datos
# docker compose down -v      # los borra: solo si quieres empezar de cero
```
