# S08 · Brief 4 — Capa de retrieval y endpoint `/search`

> **Producido por:** Claude Chat (decisión). **Ejecuta:** Claude Code.
> **Alcance:** `services/retrieval.py` (coseno exacto top-k con filtros) + endpoint
> `POST /api/v1/search` + `query_examples.py`. **NO cablea el seam:** `get_context`/
> `context/reference.py` siguen estáticos — integración retriever↔generador es S09. **NO re-embebe
> corpus, no crea índice vectorial.**

---

## 0. Por qué standalone, y no cablear ya

El temario de S09 es "cómo el retriever se integra con el generador"; cablear `get_context` a
retrieval dinámico adelanta S09 y mezcla el objetivo de datos de S08 con el de RAG de S09. El
entregable de S08 es la **capa de retrieval medible por sí sola**. Sigue en **régimen exacto**: sin
índice HNSW/IVFFlat. `Seq Scan` en `EXPLAIN ANALYZE` es correcto.

---

## 1. Objetivo

1. `services/retrieval.py`: `search(query_text, k, filters) -> hits`. Embebe la query, coseno exacto
   top-k con filtros en una query SQL atómica, mapea a un contrato de resultado con cita.
2. Endpoint `POST /api/v1/search` (additive; `/api/v1/query` intacto).
3. `query_examples.py` + `output_examples.txt`: arnés de demostración sobre 5 queries ECSS (§8).
4. Baseline de latencia medido + cross-check contra verdad numpy (§7).
5. Tests unitarios sin DB + script UAT.

---

## 2. Fuera de alcance (no lo hagas)

- **Cablear el seam:** `get_context`, `context/reference.py`, `llm_service` — intactos. `/query`
  con contexto estático. El retriever existe pero nadie de la generación lo llama aún.
- Normalización de query a inglés (S09 — abierto; UAT con queries en inglés).
- **Híbrida / léxica — es el Brief 5**, no la adelantes aquí. Este brief es solo vector.
- Reranking, expansión, grafo (S10 / S09).
- Índice vectorial, `halfvec`.
- Deuda diferida (acoplamiento offline↔API key, refactor manifiesto, `[0.26.0]`, redis huérfano).

---

## 3. La query exacta — coseno, seq scan, filtros en la misma sentencia atómica

`ORDER BY embedding <=> :qvec LIMIT :k`, filtros en `WHERE`, JOIN a `documents`.

- **Operador `<=>` (coseno), siempre**, aunque hoy no haya índice: cuando se añada el índice
  `vector_cosine_ops`, el operador ya coincide y no se cae en el antipatrón de desalineamiento (seq
  scan silencioso 600×). Coseno por robustez frente a des-normalización.
- **Filtros en la query atómica** — *la* demostración del valor de pgvector: `authority`,
  `canonical_doc_id` (y opcional `chunk_type`) en el `WHERE`, con el top-k vectorial y el JOIN, en
  **una sentencia con ACID**. Un store dedicado necesitaría coordinar dos sistemas.
- **Sin `iterative_scan`** (es problema de HNSW + filtros selectivos; en exacto el seq scan ve todas
  las filas post-filtro). Filtros opcionales; valida valores (`authority ∈ {normative, informative}`).

---

## 4. Embedding de la query — mismo espacio, con guarda

- **Reusa `app/services/embedding.py`** (S07). Única fuente de modelo/dims → la query se embebe con
  `text-embedding-3-small` @ 1536, el mismo espacio que el corpus. Con otro modelo/dims las
  distancias serían basura, y el fallo sería silencioso.
- **Guarda barata:** `len(qvec) == 1536` antes de consultar. Caza el fallo silencioso en el sitio,
  no aguas abajo.

---

## 5. Contrato del resultado — lo que sostiene una cita, no más

Por hit: `canonical_doc_id`, `doc_id_cite` (JOIN `documents`), `section_number`, `requirement_uid`
(nullable), `authority`, `chunk_type`, `cite_text`, `distance`, `oversized`.

- **Cita = `doc_id_cite` (con revisión) + `section_number`**, más `authority` y `requirement_uid`
  donde exista.
- **`cite_text`, nunca `embed_text`.** El verbatim citable sale; el texto embebido (con header) no —
  `embed≠cite` también en el borde del API.
- **`oversized`** viaja: con la semántica correcta (dilución, no truncamiento), S09 puede preguntar
  "¿este hit cae en un chunk diluido?" sin volver a la base.
- **`distance`** es la salida nativa de `<=>` (menor = más cercano). No inventes score de similitud.
- `k` por defecto 5.

---

## 6. Endpoint `POST /api/v1/search`

- Additive, convenio de router existente. `/api/v1/query` no se toca.
- **Primer punto del camino de petición que usa la DB:** cablea `get_session` (Brief 1) **en este
  router**, no en el seam de generación.
- Body: texto de query, `k`, filtros opcionales. Respuesta: lista de hits del §5.

---

## 7. Régimen exacto — criterios (invertidos) y cross-check

- **`EXPLAIN ANALYZE` mostrando `Seq Scan on chunks` es CORRECTO.** No añadas índice para
  "arreglarlo". El criterio no es "veo Index Scan".
- **Baseline de latencia:** p50 y p99 del `/search` sobre las queries del §8. Primer dato real del
  disparador exacto→HNSW (dispara si p99 cruza SLO ~100-200 ms). A 13.876 vectores se esperan ms
  bajos; se **mide**, no se afirma.
- **Cross-check (recall 1.0):** top-k exacto de SQL vs coseno **brute-force en numpy** sobre
  `embeddings.npy` → mismos ids, mismo orden, distancias dentro de tolerancia float. Valida el
  camino ingesta+query contra la verdad (métrica correcta, vectores fieles, guarda de orden), no la
  aproximación (no la hay). Corrección de camino, no evaluación semántica (eso es S09).

---

## 8. `query_examples.py` + `output_examples.txt` — arnés de demostración

Sustituye conceptualmente al `compare.py` de S07. Lanza 5 queries **en inglés** contra `/search`
(top-5), formatea legible, y vuelca a `output_examples.txt`. Cinco arquetipos, adaptados a ECSS
desde el ejercicio del máster para ejercitar el corpus desde ángulos distintos:

1. **Match directo conocido** (sanity): *"requirements for dynamic memory management in flight
   software"* — debe casar casi perfecto con una cláusula de ECSS-E-ST-40.
2. **Reformulación semántica** (¿significado o palabras?): *"rules for allocating and freeing memory
   at runtime in onboard software"* — misma idea, vocabulario distinto al del corpus.
3. **Fuera de dominio** (el corpus es subset software): *"thermal control coatings for external
   spacecraft surfaces"* — distancia alta / irrelevante esperado.
4. **Ambigua/genérica**: *"verification requirements"* — muchos chunks casan parcialmente; observa
   el ranking sin match dominante.
5. **Muy específica, con IDs/criticidad**: *"difference between software criticality category B and
   C"* — **aquí el baseline vector-puro enseñará su límite** (el par trampa B/C, coseno 0.98). Es el
   punto de arranque del diagnóstico de S09 y de la justificación de la híbrida del Brief 5.

Por hit: `chunk_id`/`section_number`, `distance` (4 decimales), `authority`, `doc_id_cite`, y ~120
chars de `cite_text`. Formato libre, legible en terminal.

`query_examples.py` embebe (red) y consulta la DB → correrlo es **UAT**; en pytest se testea el
formateo con respuestas mockeadas. `output_examples.txt` se genera en UAT contra el corpus real —
es la **foto del baseline vector** que el Brief 5 comparará contra híbrida.

---

## 9. Frontera de test

**Unit (pytest, sin DB — tuyo):**
- Construcción de la query (sesión mockeada): `<=>` en ORDER BY, filtros que componen, JOIN, LIMIT k.
- Embedding de query mockeado; guarda `dim == 1536` salta con dim errónea.
- Mapeo fila→hit: `cite_text` presente, `embed_text` ausente, campos de cita, `oversized`, `distance`.
- Validación de filtros; handler del endpoint con retrieval mockeado.
- Formateo de `query_examples.py` con respuestas mockeadas.

**UAT (Docker con datos del Brief 3 — Salva):**
- `EXPLAIN ANALYZE` → `Seq Scan` (correcto).
- Baseline p50/p99 sobre las 5 queries.
- Cross-check SQL vs numpy brute-force → idéntico.
- Query filtrada → solo lo que cumple.
- `query_examples.py` → `output_examples.txt` generado (foto del baseline).

---

## 10. Qué NO hacer (resumen)

- No cablear `get_context`/seam/generación. `/query` intacto.
- No híbrida/léxica (Brief 5). No índice, no `halfvec`, no rerank, no refs/grafo, no normalización.
- No usar `<->`/`<#>` — solo `<=>`.
- No devolver `embed_text`. No inventar score de similitud.
- No abrir deuda diferida. No commitear (redacta el mensaje).

---

## 11. Docs

- **`ARCHITECTURE.md`:** `services/retrieval.py` y `/api/v1/search`; query exacta con filtros
  atómicos como la propiedad que justificó pgvector; régimen exacto con disparador a HNSW + números
  baseline; seam sigue estático.
- **`CHANGELOG.md`:** entrada nueva (no repitas `[0.26.0]`).
- **`CLAUDE.md`:** estado presente — retrieval exacto medible; seam estático; siguiente S09. Sin diario.

---

## 12. Entregables + aceptación

1. `services/retrieval.py` (coseno exacto top-k, filtros atómicos, guarda de dim, contrato §5).
2. `POST /api/v1/search` (usa `get_session`; `/query` intacto).
3. `query_examples.py` + `output_examples.txt` (§8).
4. `uv run pytest` verde sobre lo testeable sin DB.
5. Script UAT (§9): `Seq Scan` correcto, baseline p50/p99, cross-check vs numpy, filtrado,
   `output_examples.txt`.
6. Mensaje de commit propuesto + docs (§11).
7. `S08-brief-4-decisions.md`: decisiones crudas etiquetadas, **números de latencia baseline**,
   resultado del cross-check, y qué mostró el arquetipo 5 (B/C) en vector puro. Súbelo al chat.
