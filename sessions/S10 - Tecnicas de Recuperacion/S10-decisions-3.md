# S10 · Decisiones — Brief 3 (defaults, profundidad del pool, golden set v2, expansión acotada)

> Decisiones tomadas al implementar `S10-brief-3.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Último brief de S10.
>
> **Verificación:** `uv run pytest` **1157 verde** (1127 + 30 netos). Contra la pila real: defaults
> efectivos en el contenedor recreado (`/search` híbrido responde con `tokenized_length`), golden set
> ampliado a v2 con una ronda de modo B, 5 configuraciones medidas sobre v2, curva de profundidad y
> expansión sobre v1 y v2. **Pendiente del usuario:** la UAT de Streamlit (§5 de `S10-uat-3.md`).

---

## 1. Defaults (F1–F3)

- **[AGNÓSTICO] Defaults efectivos:** `RETRIEVAL_MODE=vector`, `RETRIEVAL_ROUTING=filter`,
  `RETRIEVAL_LEXICAL=tokenized_length`, en `Settings`, `.env` y `.env.example`. Un test los fija en un
  entorno sin variables opcionales.
- **[AGNÓSTICO] `/search` sigue el mismo knob que la generación**: el router le pasa
  `RETRIEVAL_LEXICAL`, así que `/search?mode=hybrid` deja de servir `legacy`. `rank()` y `search()`
  tienen `DEFAULT_LEXICAL = tokenized_length`.
- **[AGNÓSTICO] `legacy` se conserva, solo por nombre.** El arnés lo necesita: sus configuraciones
  `vector` y `hybrid` lo fijan explícitamente para reproducir las tablas de los briefs 1 y 2 aunque el
  default cambie. Verificado: la tabla v1 de esta sesión reproduce la del brief 2 (`vector` 0,68,
  `vector_filter` 0,82, `hybrid_tok_bm25_filter` 0,81).
- **[AGNÓSTICO] Cuatro tests de `/search` y del proveedor fijaban la mecánica de la híbrida con
  `legacy`**; con el default nuevo la híbrida hace una sentencia más (`ts_debug`). Se adaptaron, y uno
  nuevo fija que `legacy` sigue disponible pidiéndolo.

## 2. Profundidad del pool

- **[AGNÓSTICO] El pool de cada canal de la fusión es `max(50, k)`**: con 400 candidatos, la híbrida
  ya no podía devolver más de 50 + 50. En producción (`top_k=10`) no cambia nada.
- **[AGNÓSTICO] `corpus/groundtruth/retrieval/depth.py`**, aparte del arnés, que pasaba de 500 líneas:
  la profundidad es `pool=` en cualquier configuración del registro. Latencia: mediana entre consultas
  de la mediana de 3 ejecuciones en caliente de la recuperación de candidatos.

### Curva recall-profundidad (golden set v2, 14 consultas)

| Configuración | Profundidad | Cláusulas candidatas (mediana) | R@cand | R@cand cap | Latencia ms |
|---|---|---|---|---|---|
| `vector_filter` | 50 | 36 | 0,42 | 0,46 | 306 |
| | 100 | 64 | 0,52 | 0,52 | 294 |
| | 200 | 124 | 0,62 | 0,62 | 310 |
| | 400 | 237 | **0,70** | **0,70** | 308 |
| `hybrid_tok_bm25_filter` | 50 | 38 | 0,44 | 0,47 | 395 |
| | 100 | 72 | 0,56 | 0,56 | 420 |
| | 200 | 137 | 0,65 | 0,65 | 417 |
| | 400 | 270 | **0,73** | **0,73** | 449 |

Por estrato (`vector_filter`, R@cand cap a 50 → 400): `names_standard` 0,60 → 0,89, `generic` 0,66 →
0,92, `cross_standard` 0,60 → 0,90, `comparative` 0,49 → 0,84, `multi_turn` 0,42 → 0,66,
`paraphrase` 0,26 → 0,49, y **`names_standard_xref` plano: 0,17 → 0,20**. Sobre las 10 de v1, la
curva va de 0,57 a **0,87**.

- **[DOMINIO] Lectura: las relevantes que faltan están más abajo, no ausentes.** De 50 a 200 el
  R@cand cap sube 16 puntos y a 400, 24. El reranker **tiene margen** sobre un pool de 200–400, y
  ampliar el pool **no cuesta latencia**: el régimen exacto ya escanea la tabla entera, y `LIMIT 400`
  frente a `LIMIT 50` no se nota.
- **[DOMINIO] La excepción es la señal del §3:** en `names_standard_xref` la curva no sube, porque el
  filtro excluye el otro estándar en cualquier profundidad.
- **[DOMINIO] La paráfrasis es lo más bajo a cualquier profundidad** (0,49 a 400): ahí sí parece un
  límite de representación, no de pool.

## 3. Golden set v2

- **[AGNÓSTICO] Protocolo 1.1** en `corpus/groundtruth/retrieval/protocol.md`: el estrato
  `names_standard_xref` con la definición literal del brief, un segundo `paraphrase` y `multi_turn`,
  y el modo de **ampliación** (el anotador lee `existing_queries.json`, escribe solo las nuevas y la
  fase 2 cubre solo esas). El criterio no cambia, así que los juicios 1.0 conservan su significado.
- **[AGNÓSTICO] `golden queries`** exporta las consultas existentes sin juicios para el anotador;
  **`golden extend`** admite la ampliación. Una consulta v1 solo puede repetirse **sin cambios**, una
  ampliación **no puede juzgar** consultas v1, la estratificación conjunta debe ser la de v2 y un set
  v2 no se amplía otra vez.
- **[AGNÓSTICO] Versión registrada:** `GoldenSet.version` y `Query.version`, la versión que introdujo
  la consulta. El informe da la versión en cabecera y, con un set v2, **dos tablas**: las 10 consultas
  v1 (comparables con lo anterior) y las 14.
- **[DOMINIO] Anotación:** 4 consultas nuevas (`claude-opus-5-5`): q11 (especificación técnica en
  E-ST-40C → `E-ST-10-06C`), q12 (riesgos de software en Q-ST-80C → `M-ST-80C`), q13 (parchear
  software en órbita, en paráfrasis) y q14 (3 turnos: servicios PUS de `E-ST-70-41C` → procedimientos
  de a bordo de `E-ST-70-01C`). **141** relevantes en fase 2. Modo B: **42 pares**, 3 relevantes.
  Total: **560 juicios, 459 relevantes**, lista de revisión vacía. La anotación confirma el riesgo
  que el estrato busca: en q11, 17 de 34 relevantes son de `E-ST-10-06C`, y en q12, 17 de 23 son de
  `M-ST-80C`.

### Tabla v2: filtro frente a cuota (p@k)

| Configuración | names_std (2) | **xref (2)** | generic (2) | paraphrase (2) | comparative (2) | cross_std (2) | multi_turn (2) | **todas (14)** |
|---|---|---|---|---|---|---|---|---|
| `vector` | 0,41 | **0,49** | 1,00 | 0,56 | 0,75 | 0,74 | 0,27 | 0,60 |
| **`vector_filter`** | **0,74** | 0,39 | 1,00 | 0,56 | 0,75 | 0,74 | 0,64 | **0,69** |
| `vector_quota` | 0,46 | **0,49** | 1,00 | 0,56 | 0,75 | 0,74 | 0,57 | 0,65 |
| `hybrid_tok_bm25_filter` | **0,81** | 0,33 | 0,88 | 0,47 | 0,68 | 0,90 | **0,69** | 0,68 |
| `hybrid_tok_bm25_quota`\* | 0,61 | 0,42 | 0,88 | 0,47 | 0,68 | 0,90 | 0,56 | 0,64 |

\* Contraste añadido por CC, no pedido por el brief: 3 cláusulas sin juzgar en xref; indicativa.

En `names_standard_xref`, R@cand: `vector` **0,38** · `vector_filter` **0,14** · `vector_quota` 0,30.
En la tabla v1 (10 consultas), las cifras del brief 2 se reproducen sin cambios.

- **[DOMINIO] El filtro pierde la cláusula del otro estándar, y está medido:** en xref la p@k cae de
  0,49 a 0,39, y R@cand de 0,38 a 0,14 —el pool filtrado no puede contener `E-ST-10-06C` ni
  `M-ST-80C`—. El peor caso es q12: 0,60 → 0,40.
- **[DOMINIO] La cuota lo recupera** (xref vuelve a 0,49, el nivel de `vector`) **a costa del resto:**
  `names_standard` 0,74 → 0,46 y `multi_turn` 0,64 → 0,57. Globalmente: filtro **0,69**, cuota 0,65,
  `vector` 0,60. **La cuota no gana en xref sin perder en otro estrato**, así que no se reporta como
  candidata a default.

## 4. Expansión de cláusula

- **[AGNÓSTICO] Dónde vive la etapa: en el proveedor de contexto.** Necesita los chunks hermanos de
  cada cláusula, que están en la base, y `llm_service` no conoce la base por diseño. El proveedor ya
  sabe qué cláusulas entrega, porque la clave es la misma de `group_by_clause`. La lógica —qué añadir,
  topes, prioridad— es una función pura (`app/services/expansion.expand`) que el informe usa tal cual.
- **[AGNÓSTICO] La expansión no cambia qué cláusulas llegan, por construcción:** solo añade chunks de
  cláusulas ya entregadas, detrás de los recuperados, así que `group_by_clause` da las mismas
  cláusulas en el mismo orden. Hay un test que lo fija para los tres modos.
- **[AGNÓSTICO] Traza:** `EvidenceChunk.origin` (`retrieved` | `expansion`) y Streamlit marca lo
  añadido («➕ added to complete the clause»), también fuera de dev. Un chunk añadido no lleva
  `distance`: nunca se rankeó contra la query.
- **[AGNÓSTICO] Orden dentro de la cláusula:** los recuperados en orden de rango y los añadidos detrás,
  en orden de artefacto (`chunks.id` = orden de carga). La limitación conocida no se resuelve aquí.
- **[AGNÓSTICO] `bounded`:** normativos primero (orden estable), y la cláusula se corta en el primer
  chunk que no cabe, para que un informativo pequeño no se cuele delante de un normativo grande.
  Tokens con el contador vigente (~4 chars/token), repetido en `app/` porque la imagen no trae
  `corpus/`.
- **[AGNÓSTICO] `orphans` no lleva topes** (el brief no los pide). Medido: no hacen falta, las 6
  huérfanas tienen 1–2 normativos.
- **[AGNÓSTICO] Configuración:** `RETRIEVAL_EXPANSION` (`off` | `orphans` | `bounded`, default `off`),
  `RETRIEVAL_EXPANSION_CLAUSE_TOKENS` (600), `RETRIEVAL_EXPANSION_TOTAL_TOKENS` (2.000).

### Tabla de expansión (v2, retrieval por defecto F1, 14 consultas, 106 cláusulas entregadas)

| Modo | Completitud media / mediana | Completitud normativa media / mediana | Huérfanas | Tokens de contexto mediana / p95 | Chunks añadidos | Incompletas (por tope) |
|---|---|---|---|---|---|---|
| `off` | 0,68 / 1,00 | 0,59 / 0,50 | **6** / 106 | 2.340 / 3.875 | 0 | 52 (—) |
| `orphans` | 0,71 / 1,00 | 0,68 / 1,00 | **0** | **2.340 / 3.875** | 7 | 46 (—) |
| `bounded` | **0,92** / 1,00 | **0,95** / 1,00 | 0 | 3.178 / 5.227 | 189 | 19 (**19**) |

- **[DOMINIO] Las huérfanas son el patrón previsto:** las 6 son NOTE de `Q-ST-80C` / `E-ST-40C` que
  llegan sin su requisito. 4 remiten a otra cláusula o documento («See also software problem
  reporting described in clause 5.2.5.», en §6.2.5.4 para q02; «see clause 5.2.7»; «as identified in
  clause 5.2.2.1»; `ECSS-Q-HB-80-03`) y 2 son aclaraciones («This includes documents to be handed
  over…»; «These measures can include…»). Ninguna dice nada vinculante por sí sola.
- **[DOMINIO] `orphans` las arregla gratis:** 7 chunks en 14 peticiones, mediana y p95 de tokens
  intactos.
- **[DOMINIO] `bounded` completa casi todo** (normativa 0,59 → 0,95) por **+36 % de tokens en
  mediana** (+838) y +35 % en p95. Los topes cortan donde se esperaba: **anexos DRD**, como
  `E-ST-40C §B.2.1` (24 de 48 chunks), `§D.2.1` (20/50) y `Q-ST-80C §B.2.1` (20/38), más cláusulas
  largas de handbooks (`E-HB-40A §5.3.5.2.2`, 2/9).

---

## 5. Recomendación (no aplicada)

**Expansión: default `orphans`.** Elimina la huérfana, el caso prioritario de S09, sin coste medible.
`bounded` mejora mucho la completitud, pero añade un 36 % de contexto. Que eso ayude a la respuesta o
la diluya no lo mide el golden set: es evaluación de la generación (S11). Hasta entonces, `bounded`
queda como opción con sus topes de partida. La UAT de Streamlit da una primera lectura cualitativa.

**Filtro frente a cuota: el filtro sigue de default.** El riesgo que motivó `names_standard_xref` es
real y está medido: −0,10 de p@k y un R@cand que se hunde en el estrato. Pero la cuota no lo resuelve
sin perder más en `names_standard`, y el global favorece al filtro (0,69 frente a 0,65). Lo que el dato
pide no es ni filtro ni cuota, sino un filtro que **incluya los estándares a los que remite el
nombrado**: el grafo de referencias ya los conoce. `# TODO: pendiente de decisión en Claude Chat`:
depende de validar el grafo, que sigue en FAIL.

**Señal para el reranker: positiva, y registrada.** R@cand cap 0,46 → 0,70 de 50 a 400 chunks (0,57 →
0,87 en v1), sin coste de latencia en el retrieval. Un reranker sobre un pool de 200–400 tiene margen;
el coste está en el reranker, no en el pool. Excepciones: la paráfrasis (límite de representación) y
xref (lo excluye el filtro, no la profundidad).
