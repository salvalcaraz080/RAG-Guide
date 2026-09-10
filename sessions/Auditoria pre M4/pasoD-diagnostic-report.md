# Paso D — Diagnóstico de calidad de código

**Read-only.** CC recorrió el código, reporta y propone. No se tocó nada, no se commiteó nada.
El único archivo creado es este informe.

**Árbol auditado:** `57d8a27` (C2 y C1 ya commiteados, árbol limpio). El chequeo de conformidad usa
como baseline `CLAUDE.md` y `ARCHITECTURE.md` **post-C1** y la Guide graduada.

**Sin etiquetas de disposición.** El orden es (tier × severidad); la clasificación es del chat.

---

## 1 · Resumen

| Tier | Alta | Media | Baja | Total |
|---|---:|---:|---:|---:|
| **Tier 1 · Objetivo** | 2 | 2 | 7 | **11** |
| **Tier 2 · Juicio** | — | 2 | 2 | **4** |
| | | | | **15** |

Más **3 punteros** en la bandeja de futuro y **4 observaciones**.

**Los dos de severidad alta comparten una propiedad incómoda:** los 854 tests pasan **y hay un test
que afirma cubrir exactamente lo que falla**. No son huecos descubiertos, son verdes falsos.

- **T1-1** — el filtro de cláusulas retiradas de C2 **no caza 5 marcadores que están hoy en la base**,
  y `/api/v1/search` los devuelve como `requirement/normative`. El gate de 121 no puede detectarlo
  porque un gate de conteo es ciego a que su propio predicado se quede corto.
- **T1-2** — la llamada de embedding de la query es **bloqueante dentro de `async def search`**.
  Medido: `/health` pasa de 22,7 ms a 237,4 ms (**10,5×**) mientras un `/search` está en vuelo. El
  principio está graduado *en el propio código* (`llm_wrapper.py:136`: «hard requirement, not
  optional»).

---

## 2 · Tier 1 · Objetivo

### 🔴 T1-1 · El filtro de cláusulas retiradas deja pasar 5 marcadores, y están servidos hoy

**Ubicación:** [`app/services/ingestion.py:196`](../../../RAG-ECSS/app/services/ingestion.py#L196)
· test que afirma cubrirlo: `tests/app/test_ingestion.py:244-251`

**Qué es.** El predicado es `text == "-" or text.startswith("<<deleted")`. El corpus contiene
variantes de la misma familia que no encajan: `<< deleted, …>>` (con espacio tras `<<`) y
`>>deleted>>` (typo del documento fuente). No se descartan en la carga, así que **están en la base
y son citables**.

**Evidencia.** Las cinco filas, en la base ahora mismo, todas indexables por el canal léxico:

```
ECSS-E-ST-10-03C#6.5.7.1::…0750367  | >>deleted>>                                | lexico=t
ECSS-Q-ST-20C#5.8.1.2.1::r0         | << deleted, requirement moved to 5.8.1a >> | lexico=t
ECSS-Q-ST-20C#5.8.1.2.1::r1         | << deleted, requirement moved to 5.8.1b >> | lexico=t
ECSS-Q-ST-20C#5.8.1.2.1::r2         | << deleted, requirement moved to 5.8.1c >> | lexico=t
ECSS-Q-ST-20C#5.8.6::r1             | << deleted, requirement moved to 5.8.4.3b>> | lexico=t
```

Y no es teórico — `POST /api/v1/search`, `mode=hybrid`, query «requirement moved deleted» devuelve
**4 de los 5 primeros hits** siendo marcadores retirados, etiquetados `requirement/normative`:

```
1. ECSS-Q-ST-20C Rev.2 §5.8.6      [requirement/normative]  '<< deleted, requirement moved to 5.8.4.3b>>'
2. ECSS-Q-ST-20C Rev.2 §5.8.1.2.1  [requirement/normative]  '<< deleted, requirement moved to 5.8.1c >>'
3. ECSS-Q-ST-20C Rev.2 §5.8.1.2.1  [requirement/normative]  '<< deleted, requirement moved to 5.8.1b >>'
4. ECSS-Q-ST-20C Rev.2 §5.8.1.2.1  [requirement/normative]  '<< deleted, requirement moved to 5.8.1a >>'
```

Es exactamente el fallo que el filtro existía para impedir, y con el agravante que el propio
docstring del filtro nombra: las variantes con texto **producen lexemas**, así que el canal léxico
no las tolera, las **prefiere**.

**Condición de disparo.** Ya disparada. Cualquier query cuyo `ts_rank` favorezca esos términos.
Se vuelve visible al usuario en cuanto S09 cablee retrieval a generación.

**Test que afirma falsamente cubrirlo.** `test_markers_are_recognised` parametriza sobre
`"-", " - ", "<<deleted>>", "<<deleted>>.", "<<deleted, covered by clause 5.4.1.2>>",
"<<deleted, requirement moved to 5.8.4.2b>>"`. **Los seis ejemplos están tomados de lo que el
predicado ya caza**: el test es tautológico respecto al defecto. Ninguno lleva espacio tras `<<`.
Verde, y ciego a la variante que falla.

---

### 🔴 T1-2 · Llamada bloqueante dentro del camino async de `/search`

**Ubicación:** [`app/services/retrieval.py:287`](../../../RAG-ECSS/app/services/retrieval.py#L287)
(`vector = embed_query(query_text, client=client)` dentro de `async def search`) →
[`app/services/embedding.py:122`](../../../RAG-ECSS/app/services/embedding.py#L122)
(`OpenAI(...).embeddings.create(...)`, cliente **síncrono**)

**Qué es.** Violación de un principio graduado, no de estilo. El proyecto decidió cliente async
nativo o `to_thread`, y lo tiene escrito en su propio código:

| Seam | Cliente | Async |
|---|---|---|
| `input_guards.py:11,65` | `AsyncOpenAI` | `await …moderations.create` ✅ |
| `llm_wrapper.py:141,204` | LiteLLM Router | `await router.acompletion` ✅ — el docstring de `:136` dice *«Async note (hard requirement, not optional)»* |
| `embedding.py:85,122` | `OpenAI` (sync) | **llamada bloqueante** ❌ |

**Evidencia medida.** `/health` es un endpoint trivial; si el loop está libre responde igual esté o
no corriendo otra petición. Disparando `/health` 150 ms después de lanzar un `/search`:

```
/health a solas            :   22.7 ms
/health durante un /search :  237.4 ms      <- 10,5x
/search total              :  485.1 ms
```

El loop queda bloqueado durante toda la ida y vuelta del embedding. Y `ARCHITECTURE.md` ya
documenta que esa llamada es **la componente dominante** de la latencia de `/search` («unas cuatro
veces el scan»): es decir, la operación más lenta del endpoint es justo la que serializa a todos
los demás.

**Condición de disparo.** Dos peticiones concurrentes cualesquiera. Hoy invisible porque el uso es
mono-usuario, y **por eso mismo entra**: es un defecto presente, no una preocupación de escala
futura, y `WEB_CONCURRENCY=1` (C2) impide la salida fácil de añadir workers.

**Test que afirma falsamente cubrirlo.** `tests/app/test_search.py:422-461` sí ejercita
`await search(...)`, pero **siempre con `client=_fake_embedding_client()`**, que devuelve al
instante. La suite ejercita la corrección funcional del camino async y **nunca su comportamiento
async**. Es literalmente el caso de la Guide: *«a test that mocks the frontier does not test the
wiring up to the frontier»*.

---

### 🟠 T1-3 · Un gate de conteo no puede detectar que su predicado se quede corto

**Ubicación:** [`app/services/ingestion.py:183`](../../../RAG-ECSS/app/services/ingestion.py#L183)

**Qué es.** `EXPECTED_RETRACTED = 121` compara `dropped != 121`. Ese gate detecta un cambio de
corpus y un predicado que **sobre**-empareja. Es estructuralmente ciego a un predicado que
**sub**-empareja: si el predicado deja de ver una familia, `dropped` sigue siendo 121 y el gate pasa.
Es el mecanismo por el que T1-1 pudo colarse con la ingesta en verde.

**Evidencia.** Los 5 de T1-1 existen y `dropped == 121` exactamente.

**Segundo hueco, del mismo tipo.** El comentario de `:180` ancla la cifra a
`corpus_version 2026-08-04.91bc8116`, pero `drop_retracted` **nunca lee
`artefacts.corpus_version`**. La atadura es documental, no verificada: otro corpus cuyo conteo
coincida en 121 pasa el gate en silencio.

*Nota metodológica:* la Guide ya registró la mitad de esto como lección de C2 («comprobar que el
instrumento del diagnóstico y el criterio del gate son la misma pregunta», el episodio 103→121). Lo
que este hallazgo añade es que **un gate de conteo no es el instrumento correcto para un criterio de
pertenencia**, ni siquiera con el número corregido.

---

### 🟠 T1-4 · `embedding._client` es el tercer global de módulo y el único sin aislamiento

**Ubicación:** [`app/services/embedding.py:78`](../../../RAG-ECSS/app/services/embedding.py#L78)
· `tests/app/conftest.py:6-38`

**Qué es.** El proyecto ha reconocido este patrón **dos veces** y lo ha cerrado con fixtures
`autouse`, ambas documentadas en `CLAUDE.md` § Tests:

| Global | Fixture autouse | Riesgo declarado |
|---|---|---|
| `input_guards._moderation_client` | ✅ `isolate_input_guards_from_real_moderation` | llamada real a la Moderation API en cada test |
| `sessions.session_store` | ✅ `isolate_session_store` | sesiones que se filtran entre tests |
| `embedding._client` | ❌ **ninguna** | llamada real a la Embeddings API |

**Evidencia.** No hay fixture. La invariante «la suite de contrato no hace llamadas reales» se
sostiene hoy **porque todos los tests recuerdan pasar `client=`** —verificado: los 6 puntos de
llamada lo inyectan— y porque el único test que toca `_default_client()`
(`tests/corpus/test_embeddings.py:118-126`) monkeypatchea `_client` y las settings antes.

Eso es disciplina, no garantía, y `OPENAI_API_KEY` **sí está en `.env`**. Un test futuro que llame a
`search()` o `embed_texts()` sin inyectar construye un cliente real y hace una llamada real: pasaría
igualmente, solo más lento y con coste. Es la forma exacta que la Guide señala — *una invariante que
descansa sobre una propiedad accidental de otro subsistema*.

**Condición de disparo.** El primer test que olvide inyectar el cliente.

---

### 🟡 T1-5 · `RetrievalError` mapea a 400 dos causas de culpa opuesta

**Ubicación:** `app/routers/search.py:31-33` · `app/services/retrieval.py:93-97`

Un filtro con valor imposible (culpa del cliente) y «la query se embebió a N dimensiones y la
columna es `vector(1536)`» (mala configuración del servidor) salen ambos como **400**. La segunda es
la guarda que `ARCHITECTURE.md` describe como puesta «en el sitio del error» para que un cambio de
modelo no falle en silencio — pero al reportarse como error de cliente, un panel de monitorización
la cuenta como petición malformada y nadie mira la config. `EmbeddingCredentialsMissing` sí está bien
separada (503).

---

### 🟡 T1-6 · Desempate de RRF con `or` sobre un entero

**Ubicación:** [`app/services/retrieval.py:260`](../../../RAG-ECSS/app/services/retrieval.py#L260)

```python
key=lambda h: (-h["rrf_score"], h["vector_rank"] or 10**6, h["chunk_id"]),
```

`0 or 10**6` → `10**6`. Hoy inerte porque los rangos vienen de `enumerate(hits, start=1)`. Si
alguien pasa a rangos 0-based, **el mejor hit vectorial se desempata como el peor** y el resultado
sigue siendo plausible. La intención es «None → al final», que se escribe
`10**6 if h["vector_rank"] is None else h["vector_rank"]`.

Defecto latente enmascarado por una precondición accidental (nodo Phase 2.5): correcto por
`start=1`, no por construcción.

---

### 🟡 T1-7 · `production_status` se asigna por prefijo de string y con `break`

**Ubicación:** [`corpus/scraper.py:362`](../../../RAG-ECSS/corpus/scraper.py#L362)

```python
if ecss_number.upper().startswith(status_number.upper()):
```

`ECSS-E-ST-10-02C` casa con la entrada `ECSS-E-ST-10` y hereda su estado; el `break` toma **la
primera coincidencia en orden de iteración del dict**, no la más específica. Es el mismo par de
defectos que la deuda documentada del extractor de referencias (ventana laxa + `break` por orden de
texto), y contradice el principio ya graduado en `CLAUDE.md`: *«Nunca es matching difuso —
`ECSS-E-ST-10` no resuelve a `ECSS-E-ST-10-02C`»*.

**Latente:** `production_status` se escribe al inventario y **nadie aguas abajo lo lee** (verificado
sobre todo `corpus/`). Se activa el día que alguien lo consuma.

*No confundir con* `classify_software_relevance` (`:205`), cuyo tipo de regla se llama literalmente
`number_prefix` y solo puede promover de tier: ahí el prefijo es la intención.

---

### 🟡 T1-8 · `derive_documents` supone que ningún documento pierde todos sus chunks

**Ubicación:** `app/services/ingestion.py:229` (`derive_documents(loadable.chunks, …)`) →
`build_rows` en `:361` (`ids[chunk["metadata"]["standard"]]`)

Tras el filtro, `documents` se deriva de los chunks **supervivientes**. Un documento cuyos chunks
fueran todos marcadores desaparecería de `ids` y `build_rows` reventaría con `KeyError` — o peor, el
documento se cargaría sin filas sin que nada lo nombre.

Verificado que hoy no pasa (el más afectado, `ECSS-E-HB-40A`, va de 497 a 446). Precondición
accidental otra vez: el filtro es pequeño hoy.

---

### 🟡 T1-9 · `EMBEDDING_PROVIDER` sobrevivió a la limpieza de variables muertas

**Ubicación:** `.env` y `.env.example`

C1 retiró `CACHE_TTL`, `REDIS_URL` y `LLM_PROVIDER` por muertas. `EMBEDDING_PROVIDER` es de la
misma clase —**nadie la lee**, verificado sobre `app/`, `corpus/` y `frontend/`— y sigue en los dos
ficheros. `CLAUDE.md` lo reconoce («hoy no la lee nadie») pero la conserva, así que la limpieza
quedó a medias por criterio, no por descuido.

---

### 🟡 T1-10 · `prose_chars` es código muerto

**Ubicación:** [`corpus/report.py:72-74`](../../../RAG-ECSS/corpus/report.py#L72)

`@property` de `DocumentResult` sin un solo uso en todo el repo (verificado sobre `app/`, `corpus/`,
`tests/`, `scripts/`, `frontend/`). Es el único código muerto que la búsqueda encontró: el resto de
candidatos resultaron ser properties, handlers de FastAPI o llamadas desde el `main()` del módulo.

---

### 🟡 T1-11 · `zip()` sin `strict=` en el chequeo de sobredimensión de `embed_all`

**Ubicación:** [`corpus/embed_all.py:75`](../../../RAG-ECSS/corpus/embed_all.py#L75)

```python
for cid, text in zip(self.chunk_ids, self.texts)
```

Es el chequeo cuyo docstring promete que ningún texto se trunca en silencio. Si `chunk_ids` y
`texts` divergieran en longitud, `zip` truncaría **el propio chequeo** al más corto: la verificación
dejaría de cubrir la cola sin decirlo. Hoy las dos listas se construyen juntas, así que es
hardening, no bug — pero es el stage de más riesgo y el candidato natural a `strict=True`.

*Los otros dos `zip` sin `strict` que ruff señala **no** son defectos:* `chunking.py:277` es el
idiom pairwise (`zip(pieces, pieces[1:])`, longitudes distintas a propósito) y
`diff_reparse.py:102` está guardado por un check de longitud tres líneas antes.

---

## 3 · Tier 2 · Juicio

Propuestas con trade-offs. Sin veredicto.

### 🟠 T2-1 · `build_rows` materializa 769 MB para cargar 84 MB de vectores

**Coste medido** (`tracemalloc`, corrida real sobre el corpus completo):

```
drop_retracted (copia de la matriz):     85.2 MB pico
build_rows (todas las filas en RAM):    768.9 MB residentes
filas construidas: 13755
```

**Qué cambiaría.** `load_artefacts` abre la matriz con `mmap_mode="r"` y el comentario de `:113`
justifica esa decisión: *«no se cargan 85 MB en RAM para leerlos fila a fila»*. Dos pasos después,
`build_rows` construye las 13.755 filas de golpe con `vector.tolist()`, que convierte cada float32 en
un float de Python: **769 MB, ~9× lo que el mmap ahorraba**. `DEFAULT_BATCH_SIZE = 500` trocea las
*sentencias*, no la construcción. Construir por lotes alinearía el código con su propia intención.

**Trade-offs.** A favor: recupera el ahorro del mmap, hace la ingesta viable en una máquina modesta y
elimina una contradicción interna. En contra: `build_rows` es hoy una función pura y trivial de
testear (`TestVectorPairing` la ejercita sin base de datos); hacerla generadora o por lotes la acopla
al bucle de escritura y complica ese test. Y **769 MB no duele hoy**: es un batch job manual que corre
en segundos en una máquina de desarrollo.

**Coste de no hacerlo.** Ninguno mientras el corpus sea este. Escala lineal: a 60-100k chunks
(la proyección que `ARCHITECTURE.md` maneja) son 3,3-5,6 GB, y ahí sí decide si la ingesta corre.

---

### 🟠 T2-2 · `check_guards` corre dos veces en cada ingesta real

**Ubicación:** `app/services/ingestion.py:396` (`_run`) y `:384` (`ingest`)

`_run` llama a `check_guards(artefacts)` y después a `ingest(...)`, que vuelve a llamarla. La guarda
4 construye una lista de 13.876 `chunk_id` para comparar con el sidecar, así que se paga dos veces.

**Trade-offs.** Quitarla de `_run` deja `ingest` como único guardián, pero `_run` la llama para
poder distinguir «guarda rechazó» (exit 1) antes de abrir conexión. Quitarla de `ingest` deja la
función pública sin protección si alguien la llama desde otro sitio. La tercera vía —un flag
`already_checked`— es peor que la duplicación. **Es más una decisión de dónde vive el contrato que
una optimización**: el coste real es despreciable.

---

### 🟡 T2-3 · Los dos `Settings` son deliberados, pero la asimetría de aislamiento no

`EmbeddingSettings` (`embedding.py:45`) existe por una razón buena y documentada: un build offline no
debe exigir `ANTHROPIC_API_KEY`. Eso no está en cuestión. Lo que sí es criterio: el proyecto tiene
ahora **dos fuentes de settings, dos caches `lru_cache`, y solo una de las dos rutas config→seam
está cubierta por fixtures**. Alternativas: dejarlo como está y cerrar solo T1-4; o hacer que
`Settings` herede de `EmbeddingSettings` para que haya un solo sitio donde se declara una variable.

---

### 🟡 T2-4 · El predicado de retiradas vive en `ingestion`, y el fenómeno es del parser

Ubicar `is_retracted` en la capa de carga fue la decisión explícita de C2 (fix barato, el robusto
diferido). La consecuencia estructural: **el conocimiento de qué es un marcador de retirada está en
`app/services/`, y el que los produce está en `corpus/`**. Cualquier corrección de T1-1 se aplica en
el sitio equivocado por diseño. Trade-off conocido y ya aceptado; se anota porque T1-1 lo vuelve a
poner sobre la mesa y porque el fix aguas arriba dispara re-chunking + re-embedding, que es
justamente lo que se quiso evitar.

---

## 4 · Bandeja de futuro

Punteros, **no propuestas**. Cosas que CC vio y que son nodo de diseño de una sesión futura:

- **S09** — el fan-out de citas (`_citations` devuelve todos los fragmentos inyectados) se vuelve
  observable en cuanto `get_context` devuelva *k* chunks. Ya está en pendientes; se anota que T1-1
  lo hace más grave, porque un marcador retirado recuperado sería además citado.
- **S10** — la ponderación de la fusión RRF. `CANDIDATE_POOL` y `RRF_C = 60` son los otros dos knobs
  sin calibrar del mismo nodo.
- **M6** — nada de este informe entra aquí salvo la consecuencia de T1-2: el día que se quiera
  `WEB_CONCURRENCY > 1` harán falta a la vez el store de sesiones compartido **y** el cliente de
  embedding async.

---

## 5 · Observaciones

**O-1 · El orden canónico del accesor de C2 está bien, y lo verifiqué en vez de asumirlo.**
`expected_document_ids` ordena por *clave* y el fingerprint digiere `path.name` (`{clave}.json`).
Son dos funciones de orden distintas y pueden divergir si una clave es prefijo estricto de otra y el
siguiente carácter es menor que `.` (p. ej. `ECSS-E-ST-10` junto a `ECSS-E-ST-10-02C`). **Para este
corpus coinciden** — probado: el fingerprint sigue siendo `ab3fce8b5cb1a0d4`. No es un defecto,
porque lo que el fingerprint necesita es un orden *estable*, no uno concreto; se anota solo porque un
corpus futuro con esa forma cambiaría el fingerprint sin que cambiara contenido, y el síntoma sería
un re-embedding aparentemente injustificado.

**O-2 · 55 chunks con `cite_text` = `None.` / `None`.** No los toqué porque no está claro que sean
la misma clase que los marcadores de retirada: `CLAUDE.md` tiene un pendiente abierto sobre «2
párrafos *None.* bajo un heading *Special remarks* de un DRD», y el criterio ahí era que enrutarlos
crearía identidad citable para contenido que dice que no hay contenido. Pero **son 55, no 2**, y hoy
están en la base y son citables. Es dato para el chat, no hallazgo: la pregunta «¿es `None.` un
marcador o una afirmación?» es de dominio.

**O-3 · La disciplina de logging y la finura de los routers están limpias.** Cero `print()` en
código de producción; `import logging` aparece solo en los dos módulos que configuran structlog
(`app/observability.py`, `corpus/log_config.py`); `search.py` es un router delgado con envelope de
error consistente con `queries.py`. Lo anoto porque el brief pedía chequearlo y un chequeo que pasa
también es un hecho.

**O-4 · Los `ERA001` y `B008` que ruff reporta son falsos positivos.** Los dos «commented-out code»
son un separador de sección (`build_parse_ready.py:181`) y un comentario que documenta la forma de
una tupla (`scraper.py:79`); el `B008` es `Depends()` en argumento por defecto, que es el idiom de
FastAPI. No son hallazgos y no deberían llegar a un fix-brief.

---

## 6 · Alcance: lo que CC dejó fuera a propósito

- **No se propuso rediseño de retrieval, reranking ni generación.** T1-1 y T1-2 entran porque son
  defectos presentes en lo ya construido; la ponderación de la fusión y el fan-out se quedan como
  punteros, no como propuestas.
- **No se re-auditó la deuda ya dispuesta en A/B.** Lo único que la roza es T1-3, y entra por ser
  **regresión sobre un cambio de C2 sin commitear-y-revisar**, que el brief admite explícitamente.
- **Ninguna optimización sin coste medido.** T2-1 cita 768,9 MB medidos con `tracemalloc`; T2-2 se
  presenta como decisión de diseño y **no** como optimización, porque su coste real es despreciable
  y no tengo medida que lo justifique de otro modo.
- **No se corrió la suite para «confirmar que pasa».** Se ejecutaron `/search`, `/health`, consultas
  a la base y `tracemalloc` **para producir evidencia**, no para validar el verde.
