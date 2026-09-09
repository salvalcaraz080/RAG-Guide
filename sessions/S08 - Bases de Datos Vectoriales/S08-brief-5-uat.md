# S08 · Brief 5 — Guion de UAT (ejecutado)

> **Ejecutado íntegramente por CC el 2026-09-09** contra la pila real. Todos los bloques verdes.
> Se conserva como guion de re-ejecución.

Ejecutar desde `C:\Users\salca\proyectos\RAG-ECSS`.

---

## 1. Migración del canal léxico

```powershell
docker compose start db
uv run alembic upgrade head
uv run alembic current
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT count(*) AS chunks, count(cite_tsv) AS con_tsv, count(*) FILTER (WHERE cite_tsv = '') AS vacios FROM chunks;"
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT indexname FROM pg_indexes WHERE tablename='chunks' ORDER BY 1;"
```

**Obtenido:** `0003_chunks_lexical_channel (head)`; `13876 | 13876 | 103`; e `ix_chunks_cite_tsv`
entre los índices de `chunks`.

La columna es **generada**, así que los 13.876 chunks ya ingeridos la tienen sin re-ingerir nada.
Los **103 vacíos** no son un fallo de la migración — ver §6.

---

## 2. El canal léxico usa el GIN (aquí SÍ esperamos índice)

No confundir con el régimen exacto: aquél es sobre el índice **vectorial**.

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "EXPLAIN ANALYZE SELECT c.chunk_id, ts_rank(c.cite_tsv, websearch_to_tsquery('english', 'difference or between or software or criticality or category or b or c')) AS rank FROM chunks c JOIN documents d ON d.id = c.document_id WHERE c.cite_tsv @@ websearch_to_tsquery('english', 'difference or between or software or criticality or category or b or c') ORDER BY rank DESC LIMIT 50;"
```

**Obtenido:**

```
->  Bitmap Heap Scan on chunks c        (actual rows=3199)
      ->  Bitmap Index Scan on ix_chunks_cite_tsv   (actual time=0.656 rows=3199)
Execution Time: 16.936 ms
```

**El GIN se usa**, y el canal léxico cuesta **16,9 ms** contra los ~65 ms del seq scan vectorial.
De paso se ve la tsquery real: `'differ' | 'softwar' | 'critic' | 'categori' | 'b' | 'c'` — la
config `english` se comió `between` como stopword, y **`b` y `c` sobreviven como lexemas**, que es
la condición para que este caso pueda funcionar.

---

## 3. `mode=hybrid` responde y respeta los filtros

```powershell
curl -s -X POST http://localhost:8000/api/v1/search -H "Content-Type: application/json" -d '{\"query\":\"criticality category\",\"k\":4,\"mode\":\"hybrid\",\"authority\":\"informative\"}'
curl -s -X POST http://localhost:8000/api/v1/search -H "Content-Type: application/json" -d '{\"query\":\"criticality\",\"mode\":\"hybrid\",\"authority\":\"mandatory\"}'
curl -s -X POST http://localhost:8000/api/v1/search -H "Content-Type: application/json" -d '{\"query\":\"criticality\",\"mode\":\"lexical\"}'
```

**Obtenido:** 4 hits, **todos** `informative`; filtro imposible → **400** con el mensaje que nombra
el valor; modo inexistente → **400** igualmente. Un hit excluido por filtro en un canal no se cuela
por el otro.

> Ojo con una trampa al probar: `{"query":"x"}` da **422**, no 400 — lo rechaza la validación de
> longitud mínima del schema antes de llegar al servicio. Hay que usar una query válida para ver
> el 400 del filtro.

---

## 4. La medición: ¿mueve la fusión el ranking?

```powershell
uv run python -m scripts.query_examples
Get-Content scripts\output_examples.txt
```

**Obtenido — el veredicto del arquetipo 5, leído del propio informe:**

```
>> discriminante: vector=3 hybrid=1  ->  sube
```

Y el top-2 híbrido trae además `ECSS-Q-ST-80C Rev.2 §D.1 [table]` (v#6 l#5), la tabla D-1 real que
el vector puro dejaba **fuera del top-5**.

### 4.1 El efecto por arquetipo, medido y no a ojo

```powershell
# script en el scratchpad: solape top-5, dónde cae el mejor hit del vector, y distancia media
```

**Obtenido:**

| arquetipo | solape 5 | top-1 vector cae a | solo-léxico | d media vector | d media híbrida |
|---|---|---|---|---|---|
| 1 match directo | 3/5 | 2 | 1 | 0,3822 | 0,3958 |
| 2 reformulación | 3/5 | 1 | 1 | 0,3985 | 0,3997 |
| 3 fuera de dominio | 3/5 | 1 | 2 | 0,4939 | 0,4858 |
| **4 ambigua** | **1/5** | **4** | 1 | 0,4488 | **0,4932** |
| 5 identificadores | 2/5 | 3 | 0 | 0,2678 | 0,2926 |

El arquetipo 4 es el daño: solo **1 de 5** resultados sobrevive y el mejor hit del vector cae al
puesto 4.

---

## 5. Latencia: qué cuesta el segundo canal

```powershell
# script en el scratchpad: 30 peticiones por modo sobre las 5 queries
```

**Obtenido:**

| modo | p50 | p99 |
|---|---|---|
| vector | 245,1 ms | 356,1 ms |
| hybrid | **290,7 ms** | 389,1 ms |

~45 ms de más, coherente con una segunda query de 17 ms más la fusión. Sigue dominado por el
embedding de la query por red.

---

## 6. Los 103 tsvector vacíos — qué son

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT DISTINCT left(cite_text, 40) AS muestra, count(*) OVER (PARTITION BY cite_text) AS veces FROM chunks WHERE cite_tsv = '' ORDER BY 2 DESC LIMIT 12;"
```

**Obtenido:** `-` (51 veces) y `<<deleted>>` (52 entre sus variantes). Son marcadores de cláusula
retirada en revisión que el parser conservó como contenido. **Tienen embedding**, así que el canal
vectorial puede devolverlos; el léxico los excluye solo porque no producen lexemas. Es un hallazgo
de chunking, no un fallo de esta migración.

---

## 7. `/query` sigue intacto

```powershell
curl -s -X POST http://localhost:8000/api/v1/query -H "Content-Type: application/json" -d '{\"question\":\"What does ECSS say about software design documentation?\"}'
```

**Obtenido:** *«Not covered by the provided material…»* con `citations` vacío — el guardrail de
scope sobre el fragmento estático de M1. La generación sigue sin ver los 13.876 chunks.

---

## 8. Cerrar (opcional)

```powershell
docker compose stop db api
```
