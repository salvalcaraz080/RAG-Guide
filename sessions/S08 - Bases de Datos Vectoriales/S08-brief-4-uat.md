# S08 · Brief 4 — Guion de UAT (ejecutado)

> **Ejecutado íntegramente por CC el 2026-09-08** tras arrancar Salva Docker Desktop. Los ocho
> bloques verdes contra la pila real. Se conserva como guion de re-ejecución.
>
> Un paso previo que no estaba en el guion original y resultó obligatorio: **la imagen del `api`
> había que reconstruirla**, no solo recrear el contenedor (§0).

Ejecutar desde `C:\Users\salca\proyectos\RAG-ECSS`.

---

## 0. Levantar la pila, con la imagen del `api` reconstruida

Dos cosas distintas, y hacen falta las dos:

- **`build`**: la imagen del `api` es anterior al brief 1, así que **no tiene SQLAlchemy**. Sin
  reconstruirla el contenedor entra en bucle con `ModuleNotFoundError: No module named
  'sqlalchemy'` — el bind-mount sirve el código nuevo, pero no las dependencias.
- **`--force-recreate`**: Compose congela `env_file` al **crear** el contenedor, así que el
  `DATABASE_URL` corregido (host `db` en vez de `localhost`) no entra en un contenedor existente.

```powershell
docker compose start db
docker compose build api
docker compose up -d --force-recreate api
docker compose ps
```

**Obtenido:** `db` en `running (healthy)`, `api` en `Up`, `/health` respondiendo.

---

## 1. Los datos del brief 3 siguen ahí

```powershell
docker compose exec -T db psql -U rag_ecss -d rag_ecss -c "SELECT (SELECT count(*) FROM documents) AS docs, (SELECT count(embedding) FROM chunks) AS con_vector;"
```

**Obtenido:** `26 | 13876`.

---

## 2. `EXPLAIN ANALYZE` → `Seq Scan` es CORRECTO

Criterio **invertido**: ver `Index Scan` sobre `embedding` sería el fallo.

```powershell
$py = @'
import asyncio, sys
sys.path.insert(0, ".")
from sqlalchemy import text
from app.db import get_sessionmaker
from app.services.retrieval import build_query, embed_query

async def main():
    vector = embed_query("requirements for dynamic memory management in flight software")
    async with get_sessionmaker()() as s:
        for label, st in (("sin filtro", build_query(vector, k=5)),
                          ("authority=normative", build_query(vector, k=5, authority="normative"))):
            sql = st.compile(dialect=s.bind.dialect, compile_kwargs={"literal_binds": True})
            plan = (await s.execute(text("EXPLAIN ANALYZE " + str(sql)))).scalars().all()
            print("==", label); print("\n".join(plan))

asyncio.run(main())
'@
$path = Join-Path $env:TEMP "uat_explain.py"
Set-Content -Path $path -Value $py -Encoding utf8
uv run python $path
```

**Obtenido** (el plan trae el vector literal entero; aquí solo las líneas estructurales):

```
sin filtro
Limit  (actual time=422.317..422.320 rows=5)
  -> Sort  Sort Method: top-N heapsort  Memory: 31kB
     -> Hash Join  (actual time=1.484..414.167 rows=13876)
        -> Seq Scan on chunks  (actual time=0.006..11.213 rows=13876)
        -> Hash -> Seq Scan on documents  (rows=26)
Execution Time: 422.430 ms      <- corrida fría

con filtro authority=normative
     -> Hash Join  (actual time=0.057..42.589 rows=10320)
        -> Seq Scan on chunks  (actual time=0.034..5.058 rows=10320)
Execution Time: 46.820 ms
```

**`Seq Scan on chunks`, cero índices sobre `embedding`.** Y lo que hay que mirar además: con
filtro el scan devuelve **10.320 filas, no 13.876** — el filtro se aplica *dentro*, así que el
top-k se calcula sobre el conjunto ya filtrado.

---

## 3. Baseline de latencia — dos números, no uno

El `Execution Time` del §2 mezcla la corrida fría. Medido en caliente y repetido:

```powershell
# script completo en el scratchpad de la sesión: 10 corridas × 5 queries, descartando la primera
```

**Obtenido, lado base de datos** (n=50, tras descartar la corrida fría):

| | ms |
|---|---|
| p50 | **65,1** |
| p99 | **69,3** |
| min / max | 59,3 / 73,4 |
| con `authority=normative` (10.320 filas) | 45,2 (mediana) |

**Obtenido, extremo a extremo por `/search`** (n=30):

| | ms |
|---|---|
| p50 | **266,7** |
| p99 | **298,5** |
| min / max | 236,0 / 1203,2 |

La diferencia (≈200 ms) es la llamada de embedding de la query por red. **El disparador
exacto→HNSW se decide con el número de la base**, no con este.

---

## 4. Cross-check: SQL exacto vs fuerza bruta en numpy

```powershell
# script completo en el scratchpad: top-10 de cada una de las 5 queries, SQL vs numpy
```

**Obtenido — las cinco queries, orden idéntico:**

```
PASS  requirements for dynamic memory management...   orden identico=True  max|Δd|=1.06e-07
PASS  rules for allocating and freeing memory...      orden identico=True  max|Δd|=9.48e-08
PASS  thermal control coatings...                     orden identico=True  max|Δd|=5.63e-08
PASS  verification requirements                       orden identico=True  max|Δd|=1.00e-07
PASS  difference between software criticality B/C     orden identico=True  max|Δd|=1.48e-07
```

Recall 1.0 confirmado contra la verdad, y la discrepancia de distancia es float32-vs-float64,
no desacuerdo.

---

## 5. Los filtros devuelven solo lo que cumplen

**Obtenido**, sobre `"verification requirements"`, top-5:

| filtro | hits | todos cumplen |
|---|---|---|
| (sin filtro) | 5 | — |
| `authority=informative` | 5 | ✔ |
| `chunk_type=note` | 5 | ✔ |
| `canonical_doc_id=ECSS-Q-ST-80C` | 5 | ✔ |

Cinco hits en todos los casos: el filtro entra **antes** del `LIMIT`, no después.

**Filtro imposible:** `authority=mandatory` → **HTTP 400**, `authority='mandatory' no es válido;
valores: ('normative', 'informative')`. No un 200 con lista vacía.

**Campos del hit:** `authority, canonical_doc_id, chunk_id, chunk_type, cite_text, distance,
doc_id_cite, oversized, requirement_uid, section_number`. **`embed_text` ausente.**

---

## 6. La foto del baseline vector

```powershell
uv run python -m scripts.query_examples
Get-Content scripts\output_examples.txt
```

**Obtenido:** `scripts/output_examples.txt` con los cinco arquetipos. Lectura en
`S08-brief-4-decisions.md` §8 — hay dos hallazgos que no son «pasa/no pasa».

---

## 7. `/query` sigue intacto

```powershell
$body = '{"question": "What does ECSS say about software design documentation?"}'
Invoke-RestMethod -Uri "http://localhost:8000/api/v1/query" -Method Post -Body $body -ContentType "application/json"
```

**Obtenido:** *«Not covered by the provided material: The reference material provided contains
only a single clause…»*, con `citations` vacío. El guardrail de scope rechazando sobre el
fragmento estático de M1 — **la generación no ve los 13.876 chunks**, que es exactamente lo que
este brief debía dejar sin tocar.

---

## 8. Cerrar (opcional)

```powershell
docker compose stop db api
```
