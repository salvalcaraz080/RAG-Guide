# S08 · Decisiones — Brief 4 (capa de retrieval y `/search`)

> Decisiones crudas tomadas al implementar, etiquetadas **[AGNÓSTICO]** (transferible a
> cualquier sistema RAG) o **[DOMINIO]** (específico de ECSS / de este proyecto).
>
> **Verificación completa:** `uv run pytest` **809 verde** (25 tests nuevos), lint limpio, y la
> UAT ejecutada entera contra la pila real (`S08-brief-4-uat.md`). Los números de §8 están
> **medidos**, no estimados.

---

## 1. El operador se elige hoy aunque hoy no cambie nada — **[AGNÓSTICO]**

`<=>` (coseno) en el `ORDER BY`, nunca `<->` ni `<#>`. Sin índice, los tres darían el mismo seq
scan y el mismo coste, así que la elección **no tiene efecto medible ahora**. Se hace igualmente
por lo que pasa después:

Un índice `vector_cosine_ops` sirve consultas escritas con `<=>` y **no** las escritas con `<->`.
Cuando no coinciden, Postgres **no da error ni aviso**: ignora el índice y cae en seq scan. El
síntoma es una degradación de dos órdenes de magnitud que aparece meses más tarde, en un sistema
que «tiene índice», y que nadie relaciona con una letra del operador.

Es la misma clase de decisión que separar `embed_text` de `cite_text`: coste cero hoy, y cierra
por construcción un fallo que más adelante sería caro de diagnosticar. Hay un test que lo fija
(`test_orders_by_the_cosine_operator`) y que además comprueba que los otros dos operadores **no**
aparecen.

Coseno y no producto interno también por robustez: no dependemos de que los vectores estén
normalizados.

---

## 2. Los filtros dentro del top-k, y por qué eso es *el* argumento de pgvector — **[AGNÓSTICO]**

`authority`, `chunk_type` y `canonical_doc_id` van en el `WHERE` de **la misma sentencia** que el
`ORDER BY <=> … LIMIT k` y el `JOIN documents`. No es una preferencia de estilo: es la diferencia
entre dos semánticas.

- **Filtrar dentro:** el top-5 se calcula sobre el conjunto filtrado. Pides 5 chunks informativos
  y recibes los 5 informativos más cercanos.
- **Filtrar fuera (top-k y luego descartar):** pides 5 y puedes recibir 1, porque los otros 4 eran
  normativos. Silenciosamente peor cuanto más selectivo el filtro.

Que lo primero sea **una sola sentencia con ACID** es lo que justificó meter los vectores en
Postgres en vez de en un store dedicado al lado: cualquier alternativa exige mantener dos sistemas
de acuerdo sobre los mismos metadatos.

Nota sobre lo que NO se hizo: **no hay `iterative_scan`**, y no es un olvido. Ese parámetro existe
para HNSW con filtros selectivos, donde el índice devuelve k candidatos y el filtro los diezma. En
régimen exacto el seq scan ve todas las filas ya filtradas y el problema no puede darse.

---

## 3. Un filtro imposible se rechaza; no devuelve vacío — **[AGNÓSTICO]**

`authority="mandatory"` no existe en el corpus (el CHECK de la tabla admite dos valores). La
opción fácil es dejar que la query devuelva 0 filas. Se rechaza con 400 y el mensaje nombra el
valor.

El motivo es que **un top-k vacío es ambiguo**: significa igual «tu filtro está mal escrito» que
«no hay nada parecido en el corpus», y son dos situaciones que piden acciones opuestas. Un
retrieval que no distingue esas dos cosas enseña a su usuario a desconfiar de los ceros.

---

## 4. La guarda de dimensión va en el sitio del error — **[AGNÓSTICO]**

La query se embebe por `app/services/embedding.py`, el mismo seam que embebió el corpus, así que
cae en el mismo espacio **por construcción** y no por disciplina. Aun así hay un
`len(vector) == 1536` antes de consultar.

No es redundante: si alguien cambiara `EMBEDDING_MODEL` en `.env`, el corpus seguiría en el espacio
viejo y la query saldría en el nuevo. Las distancias serían **ruido con forma de resultado** —
cinco hits, ordenados, plausibles y sin relación con la pregunta. No hay ninguna señal aguas abajo
que delate eso, así que la guarda tiene que estar donde se produce, no donde se nota.

Es la misma familia que la quinta guarda de la ingesta (brief 3 §3): proteger del fallo que no
hace ruido.

---

## 5. `embed_text` no sale, y no sale porque no se selecciona — **[AGNÓSTICO]**

El contrato de `SearchHit` no tiene campo `embed_text`, pero además **la columna no está en el
`SELECT`**: no llega a salir de la base. Hay dos tests, uno sobre el SQL compilado y otro sobre la
respuesta HTTP.

Poner la defensa en el borde del API habría bastado hoy. Ponerla en la query significa que el
header sintético no está disponible para que nadie lo devuelva por error más adelante — que es la
diferencia entre una convención y un invariante.

Por el mismo criterio, `distance` sale tal cual la calculó el operador y **no se convierte en un
`score` de similitud**. Un `1 - distance` sería una escala inventada por nosotros, y la tentación
siguiente es umbralizar sobre ella sin haber medido qué significa.

---

## 6. Dónde vive cada pieza

- `app/services/retrieval.py` — la capa medible. **No la llama nadie de la generación**:
  `get_context` sigue devolviendo el fragmento estático. Cablearlo es S09, y el brief lo excluye
  explícitamente para no mezclar el objetivo de datos de S08 con el de RAG de S09.
- `app/routers/search.py` — **el primer punto del camino de petición que toca la base**. Por eso
  `get_session` se cablea aquí y no en el seam de generación: la dependencia entra por donde se
  usa.
- `scripts/query_examples.py` — arnés de demostración. Va a `scripts/` porque es el mismo género
  que `recall_groundtruth.py` y compañía: mide una sesión concreta, no es una etapa del pipeline.
  `output_examples.txt` **sí se commitea**, porque su valor es ser la foto contra la que el brief 5
  compara.

---

## 7. Sorpresa: el `DATABASE_URL` del contenedor dejó de ser inocuo — **[AGNÓSTICO]**

`S08-brief-1-decisions.md` §4.2 anotó que el contenedor `api` recibe `DATABASE_URL` apuntando a
`localhost` (correcto para alembic y la ingesta, que corren fuera) y que **dentro de Compose el
host es `db`**. Se dejó sin tocar con un argumento explícito: «añadir un override para una
configuración que nadie lee sería afirmar algo que el sistema no hace todavía».

Este brief es exactamente el momento en que deja de ser cierto: `/api/v1/search` es el primer
endpoint que consulta la base. Añadido el override en `docker-compose.yml`, más `depends_on: db`
sin condición de salud (si la base tarda, la API arranca igual y falla por petición; `/query` no
la necesita en absoluto).

**Consecuencia operativa que hay que recordar al ejecutar la UAT:** Compose **congela `env_file` al
crear el contenedor**, así que un `docker compose up -d api` sobre el contenedor existente no
recoge el cambio. Hace falta `--force-recreate`. Es la misma trampa que tumbó el `api` en el brief
1, vista desde el otro lado.

Lección transferible: *una anotación de deuda con un disparador nombrado («importará cuando X»)
vale más que un arreglo preventivo — pero solo si alguien comprueba el disparador. Aquí lo
comprobó el propio brief que lo activaba.*

---

## 8. Lo medido

### 8.1 `EXPLAIN ANALYZE` — `Seq Scan on chunks`, como debe ser

Cero índices sobre `embedding`. Lo que además enseña el plan, y es el punto de §2:

```
sin filtro:              Seq Scan on chunks  ... rows=13876
authority=normative:     Seq Scan on chunks  ... rows=10320
```

El filtro **reduce las filas dentro del scan**, no después del `LIMIT`. Es la demostración
operativa de que el top-k se calcula sobre el conjunto filtrado.

### 8.2 Baseline de latencia — la predicción se cumplió, y por eso hay dos números

| | base de datos (`Execution Time`) | extremo a extremo (`/search`) |
|---|---|---|
| p50 | **65,1 ms** | **266,7 ms** |
| p99 | **69,3 ms** | **298,5 ms** |
| min / max | 59,3 / 73,4 | 236,0 / 1203,2 |

Con `authority=normative` (10.320 filas en vez de 13.876) la mediana de base baja a **45,2 ms**:
el coste escala con las filas escaneadas, que es exactamente lo que un seq scan debe hacer.

**Los ~200 ms de diferencia son la llamada de embedding de la query por red**, tal como se
predijo. Eso convierte la separación de los dos números en una decisión operativa, no en un
detalle de presentación: **el disparador exacto→HNSW se lee del número de base (65/69 ms), no del
extremo a extremo**. Mirar solo el segundo llevaría a añadir un índice vectorial para arreglar una
latencia que es de red y que el índice no toca.

Contra el SLO orientativo de 100-200 ms: la base está **holgadamente por debajo** y el disparador
no salta. A escala de corpus completo (~60-100k, ×5-7) el seq scan rondaría los 350-450 ms y **sí**
lo cruzaría — pero eso es una proyección lineal, no una medida, y el criterio del proyecto es
medir.

Nota sobre el `max` de 1203 ms en el extremo a extremo: es la primera petición, cola de arranque
del proceso y de la conexión. No contamina p50/p99 y no se ha limpiado del dato.

### 8.3 Cross-check SQL vs numpy — idéntico en las cinco queries

Orden idéntico en el top-10 de las cinco, con `max|Δd|` entre **5,6e-08 y 1,5e-07** —
float32-vs-float64, no desacuerdo. **Recall 1.0 verificado contra la verdad**, y verificación
independiente, desde el otro extremo del camino, de la quinta guarda de la ingesta: si un chunk
tuviera el vector de otro, estos rankings no coincidirían.

### 8.4 El arquetipo 5 (B vs C): más interesante que un simple fallo — **[DOMINIO]**

El resultado **no** es «el vector puro falla». Es más útil que eso.

Top-5 de *"difference between software criticality category B and C"*:

| # | d | chunk |
|---|---|---|
| 1 | 0,2593 | `ECSS-E-HB-40A §4.2.4.2` — «Critical software is defined … as category A, B or C» |
| 2 | 0,2623 | `ECSS-Q-ST-40C §6.5.6.3` — «La criticidad (A,B,C) se asignará…» |
| **3** | **0,2641** | **`ECSS-Q-ST-40C §6.5.6.3` [table] — la tabla que SÍ distingue B de C** |
| 4 | 0,2711 | `ECSS-Q-ST-80C §3.2.8` [glossary] — «software of criticality category A, B or C» |
| 5 | 0,2820 | `ECSS-E-ST-40C §3.2.7` [glossary] — idéntico al anterior, otro documento |

Comprobado leyendo el chunk 3 en la base: **contiene el contenido que responde** («Criticality
category B if, in addition, at least one of the following compensating provisions is available…»).
O sea, **el retrieval SÍ lo alcanza** — está en el top-5 con k=5.

Lo que falla es el **orden**: dos chunks que solo repiten la enumeración «A, B o C» lo adelantan.
Y el **rango de distancia del top-5 es 0,0227, el más estrecho de los cinco arquetipos**: es casi
un empate. El vector está ordenando por parecido temático con «criticality category», no por cuál
de los chunks lleva el contenido discriminante — que es justo lo que S07 predijo al medir que B y
C difieren en 2 tokens de 42.

**Consecuencia para el brief 5 y S09:** la justificación de la híbrida sigue en pie, pero el
problema medido es de **ranking**, no de recall. Un boost léxico sobre los tokens discriminantes
—o un reranker— tendría dónde morder, porque el candidato correcto ya está en el conjunto
recuperado. Es una conclusión más fuerte que «el vector no lo encuentra», y distinta.

### 8.5 Hallazgo no pedido: **la distancia no es una señal de confianza** — **[AGNÓSTICO]**

Comparando los cinco arquetipos:

| arquetipo | rango de distancia del top-5 |
|---|---|
| 3 · **fuera de dominio** | 0,4731 – 0,5062 |
| 4 · ambigua (en dominio) | 0,4321 – 0,4685 |
| 1 · match directo | 0,3466 – 0,4003 |
| 2 · reformulación | 0,3784 – 0,4149 |
| 5 · B vs C (falla el orden) | **0,2593 – 0,2820** |

Dos cosas que se leen de ahí:

1. **Fuera de dominio y ambigua-en-dominio se solapan** (0,47-0,51 vs 0,43-0,47). No hay un umbral
   de distancia que separe «el corpus no tiene esto» de «el corpus lo tiene y es genérico».
2. **El arquetipo con peor comportamiento semántico tiene las distancias MÁS BAJAS de los cinco.**
   La distancia mide parecido con lo recuperado, no si lo recuperado responde.

**Por eso no se convirtió `distance` en un `score` de similitud** (§5): habría invitado a
umbralizar sobre una escala que estos números demuestran que no soporta esa lectura. Para S09
significa que un gate de «no lo sé» basado en distancia **no funcionaría**, y hay que buscarlo en
otro sitio (el generador rechazando por scope, como ya hace M1).

### 8.6 Nota sobre el arquetipo 1

El brief predecía que casaría «casi literal con una cláusula de ECSS-E-ST-40». **No ocurrió**: el
top-1 es una sección informativa de `ECSS-E-HB-40A` (el handbook), y el primer requisito normativo
aparece en el puesto 3, de `ECSS-E-ST-70-01C`. No es un fallo del retriever —los cinco hits hablan
de memoria— pero sí un recordatorio de que el corpus tiene 26 documentos y el más parecido a una
pregunta no tiene por qué ser el que uno esperaba.

---

## 9. Lo que se dejó sin hacer, según el brief

- **Sin cablear el seam**: `get_context`, `context/reference.py` y `llm_service` intactos;
  `/query` responde con el fragmento estático.
- **Sin híbrida ni léxica** (brief 5), sin rerank, sin expansión por grafo, sin normalización de
  query a inglés.
- **Sin índice vectorial**, sin `halfvec`.
- **Sin abrir la deuda diferida**: acoplamiento offline↔`ANTHROPIC_API_KEY`, refactor del
  manifiesto de los seis módulos, `[0.26.0]` duplicado, huérfano `rag-ecss-redis`.
