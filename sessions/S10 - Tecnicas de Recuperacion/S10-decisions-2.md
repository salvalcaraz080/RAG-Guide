# S10 · Decisiones — Brief 2 (canal léxico, routing por estándar nombrado, recall con techo)

> Decisiones tomadas al implementar `S10-brief-2.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> E1–E7 se implementan como dice el brief; aquí va lo que hubo que decidir, dónde se apartó la
> implementación y la tabla final con la recomendación.
>
> **Verificación:** `uv run pytest` **1127 verde** (1069 + 58 netos). Contra la pila real: migración
> `0004` aplicada, estadísticas léxicas calculadas (13.695 chunks, 10.550 lexemas, longitud media
> 41,5), 11 configuraciones medidas tres veces y una ronda de modo B de **24 pares** (2
> relevantes). Todo nuevo va detrás de configuración y **los defaults no cambian**.

---

## 1. Decisiones de implementación

### E1 — recall con techo

- **[AGNÓSTICO] Dos columnas nuevas, las de siempre intactas:** `R@k cap` = aciertos /
  min(|R|, k) y `R@cand cap` = aciertos / min(|R|, cláusulas candidatas).
- **[DOMINIO] En este golden set `R@k cap` es idéntico a p@k**: toda consulta tiene más relevantes
  (16–55) que cláusulas entregadas (4–10), así que min(|R|, k) = k. La columna informativa es
  `R@cand cap`: separa el techo estructural del pool (25–40 cláusulas) de la incapacidad real.
- **[AGNÓSTICO] Cabeceras en ASCII** (`R@k cap`, no `R̂`): el combinante no se imprime en la consola
  cp1252 de Windows y el arnés murió al volcar el informe.

### E2 — tokenización léxica por Postgres

`ts_debug` con la misma configuración que `cite_tsv`; se descartan los tipos `hword_part`,
`hword_numpart` y `hword_asciipart`; las stopwords caen solas (sin lexema); los lexemas se pasan
**entrecomillados** a la tsquery, que así no los vuelve a normalizar; OR como antes.

| Query | Antes (`legacy`) | Después |
|---|---|---|
| `ECSS-E-ST-40C` | `'ecss' \| 'e' \| 'st' \| '40c'` | `'ecss-e-st-40c'` |
| `E-ST-40C` | `'e' \| 'st' \| '40c'` | `'e-st-40c'` |
| `Q-ST-80C Rev.2` | `'q' \| 'st' \| '80c' \| 'rev' \| '2'` | `'q-st-80c' \| 'rev.2'` |
| `5.3.5.1` | `'5' \| '3' \| '5' \| '1'` | `'5.3.5.1'` |
| «requirements for dynamic memory management in flight software» | 6 lexemas | **los mismos 6** |

- **[AGNÓSTICO] Casts explícitos** a `regconfig` y `text`: asyncpg manda los parámetros como
  `varchar` y `ts_debug(varchar, varchar)` no existe. Solo se vio contra la base real.

### E3 — BM25 y `ts_rank` normalizado

- **[AGNÓSTICO] Dónde vive la tabla de df: en la base**, migración `0004`: `lexical_stats` (lexema →
  df), `lexical_corpus` (`input_fingerprint`, N, longitud media) y `chunks.lexical_length`
  (lexemas con repeticiones). Derivadas **solo** de `cite_tsv`, en SQL.
- **[AGNÓSTICO] Cuándo se regenera: al final de cada ingesta, en su misma transacción**, así que no
  pueden describir otra carga; y a mano con `uv run python -m app.services.lexical_stats` sobre una
  base ya cargada. Se niega si los chunks llevan más de un fingerprint. El arnés se niega a medir
  BM25 si el fingerprint de `lexical_corpus` no es el del golden set, y en producción unas
  estadísticas ausentes dan `ContextMisconfigured` (500), no un ranking vacío.
- **[AGNÓSTICO] Desviación del brief: la fórmula se calcula en Python, no en SQL.** SQL devuelve
  (chunk, lexema, tf, longitud) de los chunks que casan, con los filtros del canal; `bm25_score`
  puntúa. Así la fórmula es **una** función pura que un test de contrato comprueba con valores
  calculados a mano; en SQL solo la habría verificado una base real. Coste: ~110 ms más que
  `tokenized` en la mediana (dos viajes extra y el cálculo sobre los chunks que casan).
- **[AGNÓSTICO] IDF de Lucene, `ln(1 + (N − df + 0,5)/(df + 0,5))`**: nunca negativo, a diferencia
  del Robertson clásico con términos en más de la mitad de los chunks. `k1 = 1,2`, `b = 0,75`.
- **[AGNÓSTICO] `ts_rank` normalizado: máscara 1** (÷ 1 + log longitud). La 2 (÷ longitud) premia
  los chunks diminutos —términos de glosario, NOTE de una línea—; la 1 es la penalización suave.
- **[DOMINIO] Extensión BM25 en la imagen** (`pg_search`, `pg_textsearch`): no se propone. La de
  `pgvector/pgvector:pg16` no trae ninguna, y con el corpus congelado la tabla de df cuesta una
  sentencia por ingesta.

### E4–E7 — routing por estándar nombrado

- **[AGNÓSTICO] El detector no importa `corpus/identity.py`.** La imagen solo copia `app/`, así que
  importarlo rompería el contenedor (mismo criterio que `app/models/chunk.py`, que repite las
  constantes del chunker). Su vocabulario son las identidades canónicas de `documents` —las que
  produjo `identity.py`, leídas una vez por proceso—, y los tests lo comprueban contra **todas las
  formas** que genera `identity.py` (`doc_id`, canónica, `document_key`, `sanitize_filename`).
  `# TODO: pendiente de decisión en Claude Chat` — compartir `identity` con la imagen va con M6.
- **[DOMINIO] Qué reconoce:** prefijo `ECSS-` opcional, letra de versión opcional, revisión detrás,
  guiones tipográficos, mayúsculas indistintas, `Part 1A` con espacio o `_`. **Qué no:** `ST`/`E`/`Q`
  sueltos, números de cláusula, `-ST-` fuera de un ID, otra letra de versión (`E-ST-40B`), ni un
  prefijo de otro número (`E-ST-10` no es `E-ST-10-02C`). Límite conocido: `Q-HB-80-02` sin `Part`
  no casa con ninguna de sus dos partes.
- **[AGNÓSTICO] La cuota opera sobre las plazas de chunk de `top_k`, no de cláusula**: el proveedor
  devuelve chunks y la agrupación ocurre después, en `llm_service`. `ceil(q·k)` del filtrado, el
  resto del no filtrado, deduplicado por `chunk_id`, relleno si el filtrado trae menos.
- **[AGNÓSTICO] E6:** log `routing_applied` (estándares, hits tras filtrar) y `routing_filter_empty`
  como warning si el filtro vacía el resultado. E7: sin detección, **las mismas llamadas** que el
  camino actual —lo fija un test— y con routing apagado ni siquiera se leen los documentos.
- **[AGNÓSTICO] Configuración:** `RETRIEVAL_LEXICAL` (`legacy` | `tokenized` | `tokenized_length` |
  `bm25`), `RETRIEVAL_ROUTING` (`off` | `filter` | `quota`), `RETRIEVAL_ROUTING_QUOTA` (0,7). Defaults:
  `legacy` y `off`. En `.env` y `.env.example`.
- **[DOMINIO] Detección sobre el golden set:** q01 y q02 (`names_standard`) y **q10** (`multi_turn`:
  «And what does Q-ST-80 actually make mandatory?»). Las otras 7 no nombran nada y miden lo mismo
  con o sin routing.

---

## 2. Anotación incremental

Una sola exportación con las 11 configuraciones: **24 pares**, 2 relevantes. Golden set: **377
juicios, 315 relevantes**, lista de revisión vacía.

Queda **1 par** sin juzgar (`q09 · ECSS-Q-ST-80C §7.3.1`, solo en `vector_quota`), y no por una
configuración nueva: q09 no nombra estándar, así que `vector_quota` sigue el camino de `vector`, y
el embedding de la query cambió la última cláusula entre ejecuciones (el arnés marca q09 como no
estable). Acota el efecto: como mucho ±0,01 en la p@k global de `vector_quota`. Exportado en
`pairs_20261012-222009.json`.

---

## 3. Tabla final

`top_k=10`, `history_questions=2`, cuota 0,7. Todas válidas salvo `vector_quota` (1 par, §2).
Con 1–2 consultas por estrato la tabla es **indicativa**.

### p@k por estrato

| Configuración | names_std (2) | generic (2) | paraphrase (1) | comparative (2) | cross_std (2) | multi_turn (1) | **todas** |
|---|---|---|---|---|---|---|---|
| `vector` | 0,41 | **1,00** | **0,70** | **0,75** | 0,74 | 0,25 | 0,68 |
| `hybrid` | 0,29 | 0,88 | 0,44 | **0,75** | 0,89 | 0,33 | 0,64 |
| `hybrid_tok` | 0,27 | 0,88 | 0,44 | **0,75** | 0,89 | 0,33 | 0,63 |
| `hybrid_tok_len` | 0,53 | 0,76 | 0,50 | 0,68 | **0,95** | 0,38 | 0,67 |
| `hybrid_tok_bm25` | 0,53 | 0,88 | 0,56 | 0,68 | 0,90 | 0,38 | 0,69 |
| **`vector_filter`** | 0,74 | **1,00** | **0,70** | **0,75** | 0,74 | **1,00** | **0,82** |
| `vector_quota` | 0,46 | **1,00** | **0,70** | **0,75** | 0,74 | 0,86 | 0,75 |
| `hybrid_tok_len_filter` | 0,73 | 0,76 | 0,50 | 0,68 | **0,95** | 0,88 | 0,76 |
| `hybrid_tok_len_quota` | 0,56 | 0,76 | 0,50 | 0,68 | **0,95** | 0,75 | 0,71 |
| **`hybrid_tok_bm25_filter`** | **0,81** | 0,88 | 0,56 | 0,68 | 0,90 | **1,00** | 0,81 |
| `hybrid_tok_bm25_quota` | 0,61 | 0,88 | 0,56 | 0,68 | 0,90 | 0,75 | 0,74 |

### Global

| Configuración | p@k | R@k | R@k norm. | R@cand | R@cand cap | Latencia ms |
|---|---|---|---|---|---|---|
| `vector` | 0,68 | 0,19 | 0,10 | 0,47 | 0,51 | 318 |
| `hybrid` | 0,64 | 0,18 | 0,07 | 0,40 | 0,43 | 412 |
| `hybrid_tok` | 0,63 | 0,18 | 0,07 | 0,41 | 0,44 | 347 |
| `hybrid_tok_len` | 0,67 | 0,21 | 0,11 | 0,44 | 0,47 | 347 |
| `hybrid_tok_bm25` | 0,69 | 0,21 | 0,11 | 0,46 | 0,49 | 452 |
| **`vector_filter`** | **0,82** | 0,23 | 0,15 | 0,53 | 0,57 | **303** |
| `vector_quota` | 0,75 | 0,21 | 0,13 | 0,52 | 0,56 | 327 |
| `hybrid_tok_len_filter` | 0,76 | 0,24 | 0,16 | 0,53 | 0,56 | 329 |
| `hybrid_tok_len_quota` | 0,71 | 0,22 | 0,13 | 0,52 | 0,55 | 351 |
| **`hybrid_tok_bm25_filter`** | 0,81 | **0,26** | **0,17** | **0,55** | **0,59** | 444 |
| `hybrid_tok_bm25_quota` | 0,74 | 0,23 | 0,14 | 0,55 | 0,58 | 493 |

### El canal léxico solo (top-10 antes de fundir)

- **Sonda genérica** («requirements for dynamic memory management in flight software»): con
  `legacy` y `tokenized`, 8 de 10 son prosa de `E-HB-40A` y el resto tablas/NOTE de anexo de
  `E-ST-40C` (§Q.2, §P.2.1 de 8.698 caracteres), **~2.000 caracteres de media**. Con
  `tokenized_length` y `bm25` el top pasa a **requisitos de 82–400 caracteres**. Pero BM25 pone
  arriba `E-ST-70-41C §6.6.4.3`, el servicio PUS de *memory management*: coincidencia léxica exacta y
  tema equivocado. El IDF arregla la longitud, no la polisemia.
- **q01** (revisiones de E-ST-40C): `legacy` trae **8 filas de la tabla B de `Q-HB-80-02 Part 2A`**
  (las partes `e`/`st`/`40c` del ID), 1 relevante. Solo con E2 desaparecen, y `tokenized` ya trae 5
  relevantes; `tokenized_length` y `bm25` traen 6, con requisitos de `E-ST-40C §5.3.3.x` arriba.

---

## 4. Lectura

1. **El routing por filtro es la palanca grande, y es barato.** `vector_filter`: p@k 0,68 → **0,82**,
   latencia **más baja** (303 ms; el filtro escanea menos filas). Solo toca las 3 consultas que
   nombran estándar —las otras 7 miden exactamente igual (E7 confirmado en datos)— y en ellas:
   `names_standard` 0,41 → 0,74 y la `multi_turn` 0,25 → 1,00.
2. **La cuota pierde frente al filtro en todos los estratos que toca** (0,75 frente a 0,82). Las
   plazas que deja libres se las llevan cláusulas de otros estándares no relevantes: en q02 la cuota
   mete tres cláusulas de `E-ST-40C` y p@k cae de 0,89 a 0,67. El caso para el que existe —el
   estándar nombrado remite a otro— **no está en el golden set**, así que su beneficio no está medido.
3. **E2 sola no arregla la híbrida** (0,63 frente a 0,64): quitar las partes del ID limpia q01 en el
   canal léxico, pero la fusión sigue arrastrando prosa larga. **El arreglo real es E3**: con
   longitud o BM25 la híbrida pasa de 0,64 a 0,67 / 0,69.
4. **Arreglada, la híbrida empata con `vector`, no le gana** (0,69 frente a 0,68): cambia estratos.
   Gana en `names_standard`, `cross_standard` y `multi_turn`; pierde en `generic`, `paraphrase` y
   `comparative`. Con 1–2 consultas por estrato eso es ruido de una cláusula.
5. **Con routing, `hybrid_tok_bm25_filter` ≈ `vector_filter` en precisión** (0,81 frente a 0,82) y
   **mejor en recall** (R@k 0,26 frente a 0,23; normativo 0,17 frente a 0,15; R@cand 0,55 frente a
   0,53), a +140 ms.
6. **El techo de recall sigue ahí:** R@cand cap ≈ 0,5–0,6 en todas. La mitad de lo relevante que
   cabe en el pool no está en él: un reranker sobre 50 chunks tiene poco que rescatar; el margen
   está antes (pool, expansión de cláusula, consulta).

---

## 5. Recomendación (no aplicada)

**Default: `RETRIEVAL_MODE=vector` + `RETRIEVAL_ROUTING=filter`.** Es la mayor mejora medida (+0,14 de
p@k global, y de 0,25–0,41 a 0,74–1,00 justo donde el producto fallaba), la configuración más rápida,
determinista y auditable, y no depende de estadísticas que haya que mantener. Riesgos a vigilar: un
estándar nombrado que remite a otro (sin caso en el golden set) y un falso positivo del detector,
que el log `routing_filter_empty` hace visible.

**`hybrid_tok_bm25_filter` como alternativa, no como default**: precisión empatada, recall algo
mejor, +140 ms y una dependencia operativa más (las estadísticas). La diferencia de recall (+0,03)
cabe en una o dos cláusulas sobre 10 consultas: hace falta un golden set mayor para elegirla.

**RRF ponderado: no hace falta hoy.** La híbrida arreglada empata con `vector` intercambiando
estratos; un peso afinado sobre 10 consultas se ajustaría a ellas, no al producto. La palanca era el
routing, no el peso. Se reabre si un golden set mayor confirma que los estratos que gana la híbrida
son sistemáticos.

**La cuota no se recomienda** mientras su caso de uso no tenga una consulta en el golden set.
