# S10 · Decisiones — Brief 4 (reranker como experimento, solo en el arnés)

> Decisiones tomadas al implementar `S10-brief-4.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
>
> **Verificación:** `uv run pytest` **1179 verde** (1157 + 22 netos), **sin** el grupo de
> dependencias `rerank` instalado. Contra la pila real: 7 configuraciones medidas dos veces (antes y
> después del modo B), TTFT de referencia con 14 llamadas autorizadas (~0,10 USD), y una ronda de modo
> B de **82 pares** (7 relevantes). Golden set: **642 juicios, 466 relevantes**. **El reranker no entra
> en el camino de petición.**

---

## 1. Decisiones de implementación

- **[AGNÓSTICO] G1:** `RETRIEVAL_EXPANSION` pasa a `orphans` por defecto en `Settings`, `.env` y
  `.env.example`, con un test que lo fija sin variables opcionales. La API recreada lo usa.
- **[AGNÓSTICO] Modelo:** `cross-encoder/ms-marco-MiniLM-L6-v2` @ `233902d25c44…`. El nombre del brief
  (`…MiniLM-L-6-v2`) **redirige** en el hub a este, el actual: se fija el canónico y su commit.
- **[AGNÓSTICO] Dependencias: grupo `rerank` de uv** (`sentence-transformers`, `psutil`), que
  `uv sync` no instala. Se usa con `uv run --group rerank …`. La suite de contrato pasa sin él, y un
  test en proceso limpio fija que ni `app/` ni el arnés cargan `sentence_transformers` ni `torch`.
- **[AGNÓSTICO] Dónde vive el wrapper: `corpus/groundtruth/retrieval/rerank.py`**, junto al arnés y
  fuera de `app/`, porque es un experimento del arnés y no una etapa de producción. Si entrara en
  producción, la mudanza a `app/` iría con la decisión de llevar PyTorch a la imagen.
- **[AGNÓSTICO] Forma del wrapper (G6):** `Reranker` carga el modelo una vez y de forma perezosa
  (en el primer score, no al construirse), y devuelve `RerankedEvidence`, una **subclase de
  `RetrievedEvidence`** con `rerank_score`. Así el resto del camino (agrupación, traza) no cambia.
  `RerankingProvider` es un `ContextProvider`: envuelve el pool, se queda `top_k` y registra, por
  llamada, el tiempo del reranker y **el pool entero puntuado**. Latencia y separabilidad se leen de
  las mismas ejecuciones que miden la calidad.
- **[AGNÓSTICO] G3–G5:** unidad = chunk; el pool es el retrieval por defecto (`vector` + filtro) a
  50/200/400 chunks, se reordena y se quedan los 10 de `RETRIEVAL_TOP_K`. La query es la de retrieval
  de producción (pregunta + historial). `header` = `"<doc_id_cite> · <cláusula> · <título>\n" +
  cite_text`, con el título del export de cláusulas; sin título, la línea queda sin él.
- **[AGNÓSTICO] Bug encontrado por los tests:** el umbral «que conserva el 90 % de los relevantes»
  se calculaba con `int((1 − 0,9) · n)`, que por coma flotante cae un puesto abajo. Corregido con
  `round`, antes de la medición válida.

## 2. Tabla por estrato (golden set v2, todas válidas)

p@k (`top_k = 10` chunks → cláusulas):

| Configuración | names_std | xref | generic | paraphrase | comparative | cross_std | multi_turn | **todas** |
|---|---|---|---|---|---|---|---|---|
| **`vector_filter`** (baseline) | 0,74 | 0,39 | **1,00** | **0,56** | **0,75** | 0,74 | 0,64 | 0,69 |
| `rerank50_plain` | 0,69 | 0,30 | 0,94 | 0,38 | 0,60 | 0,82 | 0,29 | 0,57 |
| `rerank50_header` | 0,83 | **0,45** | 0,94 | 0,40 | 0,69 | 0,82 | **0,75** | 0,70 |
| `rerank200_plain` | 0,56 | 0,16 | 0,94 | 0,49 | 0,50 | 0,78 | 0,25 | 0,52 |
| `rerank200_header` | **1,00** | 0,44 | 0,94 | 0,44 | 0,60 | **0,89** | 0,62 | **0,71** |
| `rerank400_plain` | 0,61 | 0,21 | 0,94 | 0,41 | 0,50 | 0,78 | 0,30 | 0,53 |
| `rerank400_header` | **1,00** | 0,39 | 0,94 | 0,50 | 0,60 | **0,89** | 0,62 | **0,71** |

Global:

| Configuración | p@k | R@k | R@k norm. | R@cand | Latencia total ms |
|---|---|---|---|---|---|
| `vector_filter` | 0,69 | 0,19 | 0,13 | 0,42 | 322 |
| `rerank50_header` | 0,70 | 0,21 | 0,14 | 0,42 | 1.886 |
| `rerank200_header` | 0,71 | 0,21 | 0,16 | 0,62 | 4.384 |
| `rerank400_header` | 0,71 | 0,20 | 0,15 | 0,71 | 7.713 |
| `rerank50/200/400_plain` | 0,57 / 0,52 / 0,53 | 0,18 / 0,16 / 0,17 | 0,12–0,13 | = | 1.772 / 4.005 / 7.405 |

- **[DOMINIO] Sin cabecera, el cross-encoder es peor que el vector** (−0,12 a −0,17 de p@k), y sobre
  todo en multi-turn (0,64 → 0,25–0,30): sin saber a qué documento pertenece un chunk, no puede
  premiar el del estándar del que se habla. **La variante `header` es obligatoria.**
- **[DOMINIO] Con cabecera, empata globalmente** (0,70–0,71 frente a 0,69: una cláusula en 14
  consultas) **cambiando estratos**: gana en `names_standard` (0,74 → 1,00 con pool ≥ 200) y en
  `cross_standard` (0,74 → 0,89); pierde en `generic` (1,00 → 0,94), `paraphrase` (0,56 → 0,40–0,50)
  y `comparative` (0,75 → 0,60–0,69).
- **[DOMINIO] El margen del pool no se convierte en precisión.** R@cand sube de 0,42 a 0,71 entre 50 y
  400 chunks, pero la p@k de `header` apenas se mueve (0,70 → 0,71): este modelo no sabe elegir mejor
  entre más candidatos. La señal del brief 3 era necesaria, no suficiente.

## 3. Latencia, carga y memoria

Hardware: Intel64 Family 6 Model 154 (12.ª gen.), 16 CPUs lógicas, Windows, **CPU sin GPU**.

| Pool | Reranker, mediana | p95 | Frente al TTFT (1,76 s) |
|---|---|---|---|
| 50 | 1,52–1,58 s | 1,7 s | **+86–90 %** |
| 200 | 3,7–4,1 s | 6,0 s | **×2,1–2,3** |
| 400 | 7,0–7,3 s | 10,9 s | **×4,0–4,1** |

- **Carga del modelo:** 28 s y 51 s en las dos ejecuciones (la caché del hub degrada sin symlinks en
  Windows). **RAM residente:** +403–448 MB.
- **TTFT de referencia** (`/query/stream`, defaults, 14 consultas, solo el último turno en
  multi-turn): **mediana 1,76 s**, p95 3,19 s (`scripts/output_ttft_baseline.txt`). El reranker
  cuesta entre casi otro TTFT entero (pool 50) y cuatro (pool 400), por delante del primer token.

## 4. Truncado silencioso y contador de tokens (G7)

| Variante | Pares (query, chunk) | Sobre 512 tokens | Par más largo |
|---|---|---|---|
| `plain` | 5.552 | **428 (7,7 %)** | 1.566 |
| `header` | 5.598 | **462 (8,3 %)** | 1.588 |

- **[DOMINIO] El modelo puntúa por el prefijo en 1 de cada 12 pares**, sin avisar.
- **[DOMINIO] Hallazgo sobre el chunker, no sobre el reranker:** sobre el artefacto completo, **287 de
  13.876 chunks superan los 512 tokens reales** —272 tablas, 10 NOTE, 4 secciones, 1 requisito— y
  **204 de ellos ya superan los 512 tokens *estimados***, la ventana del propio chunker, sin estar
  marcados `oversized` (que hoy solo marca 2 requisitos). El caso extremo es la NOTE
  `ECSS-E-ST-40C §P.2.1::n0`, de 2.175 tokens estimados. Las tablas y las NOTE largas no se parten
  ni se marcan. `# TODO: pendiente de decisión en Claude Chat` — va con la deuda de ventana y
  tokenizador.
- **[DOMINIO] El contador ~4 chars/token contra un tokenizador real** (WordPiece del modelo, sobre
  `cite_text`): mediana **0,82** (el contador **sobreestima** en prosa: ~4,9 chars/token), p95
  **1,26**, y en tablas llega a **×2,3** (`Q-HB-80-02 Part 2A §C`: 568 estimados, 1.319 reales). Es un
  tokenizador distinto del de `text-embedding-3-small`, así que el dato acota la deuda, no la cierra.

## 5. Separabilidad del score (anclaje del threshold)

| Configuración | Relevantes / irrelevantes juzgados | AUC global | AUC por consulta (mediana) |
|---|---|---|---|
| `rerank50_plain` | 298 / 163 | 0,61 | 0,46 |
| `rerank50_header` | 298 / 163 | **0,63** | **0,57** |
| `rerank200_plain` | 631 / 291 | 0,53 | 0,28 |
| `rerank200_header` | 631 / 291 | 0,52 | 0,48 |
| `rerank400_plain` | 833 / 349 | 0,50 | 0,28 |
| `rerank400_header` | 833 / 349 | 0,51 | 0,45 |

Umbral fijo que conserva el 90 % de los relevantes: elimina el **7–8 %** de los irrelevantes con
`plain` y el **15–21 %** con `header`. Para quitar un tercio hay que perder el 20 % de lo relevante
(`rerank50_header`, umbral −2,47).

- **[DOMINIO] No hay base para un threshold.** La AUC global está entre 0,50 y 0,63, así que un umbral
  fijo entre consultas apenas separa. La mediana por consulta queda por **debajo de 0,5** en 5 de 6
  configuraciones: dentro de una consulta, el score ordena los juzgados peor que el azar.
- **[DOMINIO] Matiz que lo hace pesimista, no falso:** los irrelevantes juzgados son en su mayoría
  cláusulas que **algún sistema ya puso en su top-10** (los pares del modo B), es decir, negativos
  difíciles. Contra irrelevantes al azar la AUC sería mayor, pero el umbral tendría que operar
  justo sobre estos.

---

## 6. Recomendación (no aplicada)

**El reranker no entra en producción.** Con cabecera empata en precisión (0,71 frente a 0,69, una
cláusula en 14 consultas), cuesta entre 1,6 s (pool 50) y 7 s (pool 400) en CPU —de casi un TTFT
entero a cuatro, antes del primer token— y añade PyTorch, ~400 MB de RAM y medio minuto de carga. Las
ganancias son de estrato y se compensan: lo que gana en `names_standard` y `cross_standard` lo pierde
en `generic`, `paraphrase` y `comparative`.

- **Pool y variante, si se reabre:** `header` siempre; **pool 50** es el único que cabe en un
  presupuesto razonable (+90 % de TTFT), y con él se obtiene lo mismo que con 200–400 (0,70 frente a
  0,71). El pool más hondo no compra precisión con este modelo.
- **¿Cabe en el presupuesto de TTFT?** No en CPU. Reabrirlo pide otra cosa medible: un modelo más
  fuerte (MiniLM-L12, bge-reranker), GPU, o un reranker hospedado, con el mismo arnés y las mismas
  configuraciones: cambiar de cross-encoder local es cambiar `MODEL_ID` y su revisión.
- **Threshold por chunk: sin base.** AUC 0,50–0,63; ningún umbral fijo útil. El anclaje sigue abierto,
  ahora con evidencia en contra para este modelo.
- **Lo que sí deja:** la cabecera estándar · cláusula · título es la variable que más pesa (≈ +0,15
  de p@k sobre `plain`), y el hallazgo de chunks sobre la ventana se agrupa con la deuda de
  tokenizador real.
