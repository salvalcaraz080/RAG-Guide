# S08 — Resumen de sesión (provisional, pre-implementación)

> **Paso 2 del ciclo.** Notas provisionales: teoría destilada de los 5 artículos + decisiones
> tomadas en la revisión. **NO** incluye aprendizajes de implementar (eso es `S08-consolidado.md`,
> paso 5). No se borra al consolidar: la divergencia teoría↔implementación es material de Guide.
>
> **Sesión:** M3 / S08 — Bases de datos vectoriales. **Punto de partida:** artefacto de vectores
> de S07 (`embeddings.npy` + sidecar, `text-embedding-3-small` @ 1536, fp32) listo para cargar.

---

## 1. Teoría destilada (lo valioso, no la crítica al material)

**Por qué existe la categoría (art. 1).** La búsqueda por similitud en alta dimensión es un
problema computacional distinto al de la búsqueda por valor exacto: un B-tree no indexa
"cercanía en 1536D". KNN exacto (seq scan) garantiza recall 1.0 pero paga coste lineal; ANN
(HNSW, IVFFlat) rompe la linealidad aceptando recall <1.0. La clave conceptual: **el recall del
índice se mide contra el baseline exacto, no se asume** — mecánica nueva que no existía en el
mundo relacional. Cuatro propiedades separan un prototipo (array numpy en memoria) de producción:
persistencia, concurrencia, joins con datos relacionales, transaccionalidad.

**Mercado y pgvector (art. 2).** Cinco opciones dominantes (pgvector, Qdrant, Weaviate, Milvus,
Pinecone) sobre cuatro ejes: modelo operativo, escala práctica, funcionalidades nativas, modelo de
coste. La ventaja estructural de pgvector: cruzar búsqueda vectorial con datos relacionales en una
**query SQL atómica con ACID** — lo que los dedicados no dan sin coordinar dos sistemas.

**Anatomía del índice (art. 3).** Seq scan (recall 100%, determinista, recomendado para
auditoría/evaluación) → IVFFlat (particiona en celdas Voronoi; necesita training, **degrada en
silencio con inserciones**) → HNSW (grafo multicapa, log(n), absorbe inserciones, recall >95%,
cuesta memoria) → DiskANN (horizonte, escala >varios M). Umbral clave del texto: el seq scan deja
de ser "interactivo" al entrar en **decenas de miles de vectores**.

**Schema y distancias (art. 4).** Dos tablas 1—N (`documents`/`chunks`, `ON DELETE CASCADE`).
Tres métricas: coseno `<=>`, L2 `<->`, inner product `<#>`. Antipatrón central: **el operador de
la query debe coincidir con la operator class del índice**, o Postgres cae a seq scan silencioso
(sin error, latencia ×1000). Verificación: `EXPLAIN ANALYZE` → buscar `Index Scan using...`.

**Prototipo→producción (art. 5).** Sizing de memoria gobierna el 80% del rendimiento HNSW (¿cabe
el índice en RAM?). `halfvec` (fp16 en el índice, mitad de almacenamiento, recall >99%).
Monitorización (`pg_stat_user_indexes`, `idx_scan=0` delata el antipatrón; `pg_stat_statements`).
Tres señales objetivas de migrar fuera de pgvector: índice/RAM >70%, p99 sostenido > SLO,
funcionalidad nativa ausente. El propio artículo avisa: nada de esto aplica a los volúmenes del
programa.

---

## 2. Nuestras decisiones

### Cerradas

- **Store = pgvector.** Convergemos con el máster, pero **por nuestras razones**: joins
  relacionales + híbrida + trazabilidad en un solo sistema con ACID — no por su stack (Rails sobre
  Postgres). Reformula sobre qué eje reconsideraríamos: saldríamos si el join relacional+híbrido
  dejara de importar o si la escala explotara, no por cambiar de framework de backend.
- **Descartado: store dedicado (Qdrant/Weaviate).** Ambos reintroducen dos fuentes de verdad
  (vectores en el store, metadatos/cita en otro sistema, coherencia a tu cargo). Para el máster es
  un filtro que se desincroniza; **para nosotros es rotura de trazabilidad** — si el vector y su
  metadato de cita divergen, la cita fantasma vuelve por la puerta de atrás. El ACID de un solo
  sistema vale más aquí que en el máster, precisamente por la trazabilidad total. El "cuarto caso"
  del art. 1 (match exacto sobre IDs) que empuja a Weaviate, en pgvector se resuelve con `tsvector`
  en el mismo sistema.
- **Operator class = coseno + `vector_cosine_ops` + `<=>`.** Por **robustez frente a
  des-normalización**, no por convención. Con vectores normalizados de OpenAI, `<#>` daría el mismo
  orden ahorrando el cálculo de normas — pero ata la corrección a una propiedad accidental (norma=1)
  que un fallback a un embedder que no normalice rompería en silencio. Mismo principio S07:
  la corrección no depende de propiedades accidentales de otro subsistema. (Retirada la sugerencia
  de `<#>` que barajé en la revisión.)
- **Schema base (transfiere del máster):** dos tablas 1—N, `ON DELETE CASCADE`, `vector(1536)`
  hardcodeado (cambiar dimensión = reembedear todo = migración por columna nueva; encaja con el
  fingerprinting de S07: la dimensión es parte de la identidad del artefacto), `embedding` nullable
  (la fila-chunk existe con su cita aunque el embedding falte).
- **Divergencia de schema nuestra:** `authority_grade`, `canonical_doc_id`, `clause_id`/
  `section_number` van en **columnas tipadas**, no en JSONB. Son la identidad de la cita
  (load-bearing) y se filtran constantemente. JSONB queda solo para enriquecimiento flexible (tags,
  tecnologías inferidas). El máster mete todo lo "de documento" en JSONB porque para él la metadata
  no sostiene nada; para nosotros sí.
- **Índice de arranque = exacto / seq scan.** Divergencia explícita con el máster (ver abajo).

### Divergencia explícita con el máster

El programa arranca S08 **en HNSW** (m=16, ef_construction=128, ef_search=40). **Nosotros
arrancamos en exacto / seq scan.** Razones:

1. Su proyecto (estimaciones) nace en el rango donde HNSW gana; el nuestro no — a nuestra escala
   (miles hoy, decenas de miles a corpus completo) el exacto son milisegundos con recall 1.0.
2. Recall 1.0 = **trazabilidad**: un vecino perdido por aproximación puede ser un requisito
   perdido. El "98% ≈ 100%" del art. 1 es afirmación de RAG general; no transfiere a un corpus
   normativo con trazabilidad obligatoria.
3. Exacto **es** el baseline contra el que se medirá cualquier índice futuro — lo necesitamos de
   todas formas.
4. Exacto elimina de golpe tres familias de complejidad operativa que el art. 5 dedica páginas a
   domar: sizing de memoria, construcción de índice, mantenimiento de índice. Sencillez >
   sobreingeniería, cuantificado.
5. Bonus diagnóstico: con recall 1.0 garantizado, si el par trampa B/C sigue fallando en S09, el
   fallo es **semántico** (el espacio colapsa B/C), no artefacto de la aproximación. Exacto separa
   las dos causas; ANN las mezcla.

**Si/cuando salgamos de exacto → HNSW, no IVFFlat.** El corpus crece por diseño (subset software →
estándares completos + handbooks). IVFFlat degrada recall en silencio con inserciones — modo de
fallo que la trazabilidad no tolera. m=16, ef_construction=128 como punto de partida informado (no
nodo de decisión propio; convergencia de la comunidad para 1536D).

### Abiertas (marcadas como tales)

- **Grafo-vs-metadata** (cómo se recupera el contenido referenciado a query-time). Reformulado por
  el artículo: si el store es Postgres, `references.json` podría vivir como **tabla relacional** y
  la expansión a 1 salto sería un JOIN en la misma query atómica. Pasa a ser en parte "¿referencias
  como filas Postgres join-expandibles o como grafo separado?". → **S09/S10.** El schema de S08 no
  debe cerrarlo.
- **Calibración de `ef_search`.** Query-time, se tunea empíricamente. Para nosotros depende de la
  híbrida + rerank (S10): si la léxica pilla el par trampa B/C que el semántico colapsa, el
  presupuesto de recall del vector cambia. No se fija ahora. → **S10.**
- **`halfvec`.** Optimización del índice HNSW (mitad de almacenamiento/tiempo de construcción del
  índice), no de la columna. Sin índice no compra nada hoy. Verificado que **nuestro schema de S08
  no lo cierra**: la columna es `vector(1536)` en ambos casos, `halfvec` vive en el `CREATE INDEX`.
  Deuda de schema: cero. → **fase HNSW.**
- **`hnsw.iterative_scan`.** Resuelve HNSW + filtros muy selectivos que devuelven <k. Somos el caso
  (filtramos por authority/doc/cláusula), pero solo existe con índice — en exacto el seq scan ya ve
  todas las filas post-filtro. → **fase HNSW.**

---

## 3. Disparadores (nodos con métrica, sin fecha)

Dos disparadores encadenados, ambos medidos, ninguno por calendario:

- **exacto → HNSW:** p99 del seq scan cruza el SLO de RAG interactivo (~100–200 ms), medido con
  `EXPLAIN ANALYZE`, no estimado. A 6 KB/vector, decenas de miles de vectores son ~cientos de MB;
  el seq scan de eso es milisegundos-bajos, muy por debajo del SLO → **exacto aguanta el corpus
  completo con margen**. El disparador no es "corpus completo" per se, sino "p99 medido > SLO".
- **HNSW → DiskANN/dedicado:** las tres señales del art. 5 (índice/RAM >70%; p99 sostenido > SLO
  tras descartar tuning/antipatrón/bloat; funcionalidad nativa ausente).

---

## 4. Deuda a nombrar en el brief (para CC)

- **Criterio de aceptación en régimen exacto = latencia baseline + recall 1.0, NO `Index Scan`.**
  El criterio del máster asume índice desde el día uno; si el brief no invierte el check, CC marcará
  como fallo un `Seq Scan` que es correcto. El brief debe declarar explícitamente que estamos en
  régimen exacto y por qué no hay `CREATE INDEX`.
- **Observabilidad de query en `structlog`, NO Logfire.** El art. 5 asume Logfire en el stack del
  programa; nosotros usamos structlog (S03). Copiar "habilita Logfire" rompe nuestra capa de
  observabilidad. La necesidad (latencia baseline del seq scan, detección de regresiones) es válida
  y aplica hoy; se satisface con structlog + timing en el endpoint de búsqueda.
- **Materializar `embed≠cite` en la columna de cita.** Si `clause_id`/`canonical_doc_id` son
  columnas tipadas y sostienen la cita, deben salir del **ancla de cita del `Chunk`** (S07), no
  derivarse del `content`. Sin decirlo, CC podría derivar la cita del texto — justo lo que S07
  separó.

---

## 5. Candidatos a la Guide (agnóstico) — provisional, gradúa en paso 6 tras implementar

- **Nodo "¿store vectorial sí/no?":** persistencia ≠ base de datos; las cuatro propiedades deciden
  el store, la escala decide el ANN. Son **ejes independientes** — el artículo los conflaciona.
- **Nodo "exacto vs ANN":** condición = escala; señal = p99 medido > SLO. Postura por defecto a
  baja escala + requisito de recall total = exacto. Con alternativa descartada (ANN) y su porqué.
- **Nodo "HNSW vs IVFFlat":** corpus creciente + recall auditado → HNSW (IVFFlat degrada en
  silencio). Incondicional para RAG con corpus que crece.
- **Nodo "operator class":** con embeddings normalizados, elegir por **robustez** (coseno) sobre
  eficiencia marginal (`<#>`) — no acoplar corrección a la normalización.
- **Nodo "store dedicado vs Postgres bajo trazabilidad":** cuando la coherencia vector↔metadato es
  una restricción de correctitud (no solo de rendimiento), el ACID de un solo sistema pesa más que
  el filtrado superior de un dedicado. Alternativa descartada (Qdrant/Weaviate) + porqué.
- **Regla operativa:** `EXPLAIN ANALYZE` + alineamiento operador/operator class como criterio de
  aceptación, no como nota. Enlaza con "el estado se mide, no se declara".

---

## 6. Dato bloqueante antes del brief

**Row-count real del sidecar de `embeddings.npy`: actual + proyección a corpus completo.** Es lo
que sitúa el subset frente a la frontera "decenas de miles" del art. 3 y confirma con número (no
con estimación) que arrancamos exacto con margen. Primer dato a fijar antes de redactar
`S08-brief.md`.
