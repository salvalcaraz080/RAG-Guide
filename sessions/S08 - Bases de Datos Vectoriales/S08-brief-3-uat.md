# S08 · Brief 3 — Guion de UAT (ya ejecutado)

> **Ejecutado íntegramente por CC el 2026-09-08.** Docker estaba disponible, y `CLAUDE.md` dice
> que CC corre todo lo que esté a su alcance, así que no queda UAT pendiente: los ocho bloques
> están verdes contra Postgres real, y los comandos están verificados **en PowerShell 5.1**, que
> es el shell donde se ejecutan.
>
> Se conserva como guion de re-ejecución, por ejemplo tras regenerar el corpus.

Ejecutar desde `C:\Users\salca\proyectos\RAG-ECSS`.

---

## 1. Base levantada y migración aplicada

```powershell
docker compose start db      # o `docker compose up -d db` si no existe el contenedor
uv run alembic upgrade head
uv run alembic current
```

**Obtenido:** `0002_chunks_oversized (head)`.

---

## 2. La columna `oversized` existe y es NOT NULL con default

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "\d chunks"
```

**Obtenido:** `oversized | boolean | not null | false`, y el resto del schema del brief 1
intacto (`embedding vector(1536)` nullable, `embed_text`/`cite_text` separadas, los dos CHECK).

---

## 3. Ensayo en seco: las guardas pasan y los conteos cuadran, sin escribir

```powershell
uv run python -m app.services.ingestion --dry-run
```

**Obtenido:** 26 documents · 13.876 chunks · 13.876 con vector · 0 sin vector · 2 oversized ·
2.536 con `requirement_uid` · `input_fingerprint=ab3fce8b5cb1a0d4`.

---

## 4. La carga real

```powershell
uv run python -m app.services.ingestion
```

**Obtenido:** `ingestion_complete` con `stored_documents=26`, `stored_chunks=13876`. **57
segundos** en lotes de 500 filas.

---

## 5. Conteos leídos de la base, no del log de la ingesta

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT (SELECT count(*) FROM documents) AS docs, (SELECT count(*) FROM chunks) AS chunks, (SELECT count(embedding) FROM chunks) AS con_vector, (SELECT count(*) FROM chunks WHERE embedding IS NULL) AS sin_vector, (SELECT count(*) FROM chunks WHERE oversized) AS oversized, (SELECT count(requirement_uid) FROM chunks) AS con_uid, (SELECT vector_dims(embedding) FROM chunks WHERE embedding IS NOT NULL LIMIT 1) AS dims;"
```

**Obtenido:** `26 | 13876 | 13876 | 0 | 2 | 2536 | 1536`.

### 5.1 Split por ancla y autoridad

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT chunk_type, count(*), count(*) FILTER (WHERE authority='normative') AS normative, count(*) FILTER (WHERE authority='informative') AS informative FROM chunks GROUP BY chunk_type ORDER BY 2 DESC;"
```

**Obtenido**, idéntico a los conteos del artefacto:

| chunk_type | total | normative | informative |
|---|---|---|---|
| requirement | 7.922 | 7.922 | 0 |
| section | 2.583 | 1.246 | 1.337 |
| table | 1.481 | 760 | 721 |
| note | 1.410 | 0 | 1.410 |
| glossary_term | 480 | 392 | 88 |

Totales 10.320 normative / 3.556 informative. Las 1.410 notas informativas al 100% son el
invariante «ningún chunk mezcla autoridades» visible en la base.

---

## 6. Idempotencia

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -t -c "SELECT min(id)||'-'||max(id) FROM documents;"
uv run python -m app.services.ingestion
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT (SELECT count(*) FROM documents) AS docs, (SELECT count(*) FROM chunks) AS chunks, (SELECT min(id)||'-'||max(id) FROM documents) AS rango_ids, (SELECT count(DISTINCT chunk_id) FROM chunks) AS ids_unicos;"
```

**Obtenido:** 26 / 13.876 / rango de ids **sin moverse** (`4-29` antes y después) / 13.876 ids
únicos. El `DO UPDATE` no regenera el `BIGSERIAL`, así que las FK sobreviven al re-run.

---

## 7. Las guardas se niegan contra los artefactos reales

Los artefactos se desalinean **en memoria**, así que los ficheros de `data/processed/` no se
tocan. El script está en el scratchpad de la sesión; su forma es:

```powershell
$py = @'
import dataclasses, sys
sys.path.insert(0, ".")
import numpy as np
from app.services.ingestion import IngestionError, check_guards, load_artefacts

real = load_artefacts()
check_guards(real)                      # el par real pasa

def refuses(label, artefacts):
    try:
        check_guards(artefacts)
        print("FAIL", label)
    except IngestionError as e:
        print("PASS", label, "->", e)

refuses("fingerprint", dataclasses.replace(real, sidecar={**real.sidecar, "chunks_fingerprint": "0"*16}))
refuses("conteo", dataclasses.replace(real, chunks=real.chunks[:-1]))
refuses("dimension", dataclasses.replace(real, vectors=np.zeros((len(real.chunks), 768), dtype=np.float32), sidecar={**real.sidecar, "dimensions": 768}))
refuses("modelo", dataclasses.replace(real, sidecar={**real.sidecar, "model": "text-embedding-3-large"}))
refuses("orden", dataclasses.replace(real, sidecar={**real.sidecar, "chunk_ids": list(reversed(real.sidecar["chunk_ids"]))}))
'@
$path = Join-Path $env:TEMP "uat_guards.py"
Set-Content -Path $path -Value $py -Encoding utf8
uv run python $path
```

**Obtenido:** el par real pasa; las cinco desalineaciones se niegan, cada una con esperado y
visto. Ejemplo:

```
los artefactos no son un par coherente: chunks.json input_fingerprint='ab3fce8b5cb1a0d4'
vs sidecar chunks_fingerprint='0000000000000000'. Re-generar embeddings tras el chunking.
```

---

## 8. Spot check: la cita se ensambla por JOIN

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT d.doc_id_cite, c.section_number, c.requirement_uid, c.authority, c.chunk_type, c.oversized, left(c.cite_text, 60) AS cita FROM chunks c JOIN documents d ON d.id = c.document_id WHERE c.chunk_id = 'ECSS-E-ST-10-03C#4.1::ECSS-E-ST-10-03_0750001';"
```

**Obtenido:** `ECSS-E-ST-10-03C Rev.1 | 4.1 | ECSS-E-ST-10-03_0750001 | normative | requirement |
f | A coherent test programme shall be established, encompassing…`

La revisión viaja en `doc_id_cite` (no en el chunk), el UID está poblado, y el `cite_text` es
verbatim.

### 8.1 Los dos `oversized`, y la prueba de que `embed ≠ cite` sobrevivió a la carga

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT chunk_id, length(cite_text) AS chars FROM chunks WHERE oversized;"
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT count(*) AS embed_igual_a_cite FROM chunks WHERE embed_text = cite_text;"
```

**Obtenido:** los 2 esperados (`ECSS-E-ST-10-03C#5.5.1.2::…_0750117` con 2.220 chars y
`ECSS-E-ST-40-07C#5.3.7::…_1630261` con 2.583), y **0** filas donde el texto embebido coincida
con el citable — las dos columnas siguen siendo dos.

---

## 9. Cerrar (opcional)

```powershell
docker compose stop db        # conserva los datos ingeridos
```
