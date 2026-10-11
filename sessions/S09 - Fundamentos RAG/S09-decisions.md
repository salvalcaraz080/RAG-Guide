# S09 · Decisiones — cablear retrieval → generación

> Decisiones tomadas al implementar el brief, etiquetadas **[AGNÓSTICO]** (transferible a cualquier
> sistema RAG) o **[DOMINIO]** (específico de ECSS / de este proyecto). Las D1–D13 del brief no se
> repiten: aquí va lo que el brief dejaba abierto y lo que la implementación obligó a decidir.
>
> **Verificación:** `uv run pytest` **990 verde** (923 al empezar, +67 netos), sin red ni base. Contra la pila
> real: diagnóstico §3.8 ejecutado (§1), smoke end-to-end de cuatro preguntas (§2) y fail-closed
> con la base parada (§2.3).

---

## 1. Diagnóstico de cobertura de cláusula (§3.8)

`scripts/clause_coverage.py`, salida en `scripts/output_clause_coverage.txt`. Ocho queries: los
cinco arquetipos de S08 más las tres preguntas típicas del producto, con el retrieval de
generación tal cual (`vector`, `top_k=10`). **El conjunto de S08 está en inglés** (los cinco
arquetipos de `scripts/query_examples.py`), igual que las tres de producto.

| Medida | Valor |
|---|---|
| Cláusulas recuperadas (pares query–cláusula) | 54 |
| **Incompletas** (falta algún chunk de la cláusula) | **38 (70,4 %)** |
| Cláusulas únicas / incompletas | 39 / 29 (74,4 %) |
| Con chunk `section` no recuperado | 14 |
| Tamaño de la cláusula **completa** (tokens, únicas) | mediana 319 · p75 872 · p90 1.737 · máx 2.615 |
| Tamaño del grupo **recuperado** (tokens) | mediana 82 · p75 400 · p90 487 · máx 1.182 |
| Chunks por cláusula completa | mediana 5 · p90 18 · máx 50 |

**Lectura [DOMINIO].** Lo normal es recibir una cláusula a trozos, y el tipo de trozo importa:

1. **Notas huérfanas.** En `ECSS-E-ST-70-41C §6.6.4.2` y `§6.6.3.2` lo único recuperado es la NOTE
   («Refer also to clause 5.4.3.3.», 8 tokens) sin ninguno de sus 3–4 requisitos. Al modelo le
   llega una remisión sin el contenido al que acompaña. Pasa en las dos queries de memoria
   dinámica y en la de producto con identificador.
2. **La cola larga es de anexos DRD.** Las cláusulas de más de 2.000 tokens son anexos de
   documentos requeridos (`ECSS-E-ST-40C §B.2.1`: 47 requisitos, 2.615 tokens; `§D.2.1`: 49,
   2.244). Expandir a la cláusula entera sin tope metería una de ellas por cada requisito suelto
   que se recupere de ahí.
3. **Las cláusulas pequeñas suelen llegar completas** (glosario, secciones de un chunk, tablas
   cortas): de las 16 completas, casi todas tienen 1–2 chunks.

**Insumo para la decisión de expansión:** expandir siempre a la cláusula completa no cabe (la
mediana se multiplicaría por ~4 y la cola por ~5), pero una expansión **acotada** sí parece
justificada, y el caso más barato con más valor es el de la nota huérfana (arrastrar el requisito
o la sección a la que la nota acompaña). La decisión es de Claude Chat.

**Hallazgo lateral [DOMINIO].** La pregunta con identificador explícito («What does ECSS-E-ST-40C
require for dynamic memory management…») no recupera **ninguna** cláusula de `ECSS-E-ST-40C`: el
canal vectorial trae `70-41C`, `70-01C` y el handbook. Es el problema de identificadores de S08,
y pertenece a la ponderación de la fusión o a un filtro determinista, no al generador.

---

## 2. Smoke end-to-end contra Docker

Cuatro preguntas reales (Haiku, ~0,016 USD en total).

| # | Pregunta | Resultado |
|---|---|---|
| 1 | Identificador explícito (E-ST-40C, memoria dinámica) | `Not covered by the provided material:`. Rechazo honesto: dice que las fuentes son de 70-41C, 70-01C y un handbook informativo, y no lo presenta como requisito. `citations=[]` |
| 2 | Criticidad B/C (con sesión) | 7 cláusulas inyectadas, **1 citada** (`[5]` = `ECSS-Q-ST-80C Rev.2 §D.1`). **5 citas literales, 5 `verified`** |
| 3 | Seguimiento «And what does ECSS-Q-ST-80C say about it?» (stream) | `Not covered…`: el retrieval recibe solo la pregunta de seguimiento y el «it» no recupera nada útil (§2, §3, glosarios). El historial guardado sí lleva la cita natural `[ECSS-Q-ST-80C Rev.2 §D.1]` |
| 4 | Fuera de dominio | `Out of scope:`, `citations=[]` |

**Latencias medidas:** embedding 177–697 ms (el primero, en frío), BD 75–185 ms. Bloque de
contexto: 954–3.298 tokens estimados.

### 2.1 Lo que confirman

- **El fan-out funciona** (1 de 7 citada) y la validación literal da señal limpia en el caso bueno.
- **El seguimiento anafórico falla en retrieval, no en generación.** Es justo la decisión diferida
  «reescritura de seguimientos conversacionales»: la UAT del usuario lo tiene que ver, y aquí ya
  hay un caso reproducible.

### 2.2 Un rechazo puede citar

La pregunta 1 rechazó **citando** una frase literal del handbook (`"…no dynamic memory mechanism
implementation…" [4]`, `verified`). `citations` sale vacía por política (cita fantasma), pero
`citations_validated` registra el marcador. Es correcto así: el rechazo usó una fuente para
justificar que no puede responder, y esa señal solo se ve en el log.

### 2.3 Fail-closed con la base parada

`docker stop rag-ecss-db` → `/query` **503**, stream `event: error`, mensaje genérico al cliente y
`retrieval_failed stage=database error_type=gaierror` en el log. El `gaierror` es un `OSError`, no
un `SQLAlchemyError`: **sin capturar `OSError` habría salido un 500 y un stream roto**. El test de
contrato cubre el `OperationalError`; esta corrida cubre el otro camino.

---

## 3. Decisiones de implementación

### Seam y capas

- **[AGNÓSTICO] El proveedor devuelve `RetrievedContext`, no una lista suelta.** Lleva los chunks
  más `mode`, `top_k` y las dos latencias. Así el evento `retrieval_completed` es **uno** por
  petición y se emite en `llm_service`, que es quien conoce los grupos y el tamaño del bloque, sin
  que `llm_service` sepa cómo se recupera.
- **[AGNÓSTICO] `RetrievedEvidence(EvidenceChunk)` añade `chunk_id` y `canonical_doc_id`.** Agrupar
  necesita el canónico y loguear necesita el id; ninguno es evidencia. `SearchHit` pasa a extender
  `RetrievedEvidence` (sigue habiendo un solo modelo de evidencia), y la respuesta proyecta a
  `EvidenceChunk` (`_evidence` excluye los dos campos).
- **[AGNÓSTICO] `retrieval.search` se parte en `embed_query` + `rank`.** El proveedor embebe primero
  y abre la sesión solo para rankear: cada latencia se mide por separado, la sesión dura lo que la
  consulta y un fallo de embedding no llega a abrirla. `/search` sigue llamando a `search`, que
  compone las dos.
- **[AGNÓSTICO] El seam vive en un módulo puro (`grounding.py`)**: Protocol, `RetrievedContext`,
  errores y agrupación. `llm_service` importa de ahí; `context.py` (el que sí toca `db` y
  `retrieval`) solo lo importa el router. El test de capas corre en un **subproceso**, porque en el
  proceso de la suite `app.main` ya lo ha importado todo.
- **[AGNÓSTICO] `context_provider` es keyword obligatorio** en `answer_query`/`stream_answer_query`.
  Un default obligaría a importar el proveedor real en `llm_service`, que es justo lo que D1 prohíbe.
  En tests, una fixture autouse sustituye la dependencia del router y entrega el mismo doble a
  quien llama al servicio directamente.

### Errores

- **[AGNÓSTICO] Dos errores tipados en el seam:** `ContextUnavailable` (503) y
  `ContextMisconfigured` (500), los dos subclases de `ContextRetrievalError`, que es lo que el
  stream captura. El mensaje al cliente es genérico; qué credencial falta o qué host no responde va
  al log (`retrieval_failed`).
- **[AGNÓSTICO] La base caída también es 503.** El brief nombraba solo el embedding; sin base
  tampoco hay respuesta anclada, y sin capturarlo el stream se rompía a medias. Se capturan
  `SQLAlchemyError` y `OSError` (ver §2.3).

### Agrupación y bloque

- **[DOMINIO] Dentro del grupo, orden de rango, con `TODO`.** El orden de documento no se puede
  derivar de forma estable: los ordinales del `chunk_id` cuentan **por tipo** (`r0`, `s0`, `n0`…),
  no son posición en la cláusula.
- **[DOMINIO] `standard` del grupo = `doc_id_cite` del primer chunk.** Canónico ↔ cita es 1:1 en el
  corpus (26/26), así que no hay conflicto posible hoy.
- **[DOMINIO] Límite aceptado:** los 4 números de cláusula repetidos dentro de un handbook se
  fusionan en un grupo (documentado en el código).
- **[AGNÓSTICO] Se escapan los atributos del bloque, no el texto.** El texto es el verbatim que el
  modelo tiene que poder citar literal: un `&amp;` en el prompt acabaría dentro de la cita y la
  validación fallaría por algo que no es del modelo.
- **[AGNÓSTICO] La evidencia sale en el orden del bloque** (cláusula a cláusula, en orden de `[n]`),
  no en orden de rango: misma procedencia que el mapa `n → cláusula`.
- **[AGNÓSTICO] Los tokens del bloque se estiman como el chunker** (~4 chars/token), replicado y no
  importado: el servidor no depende de `corpus/`.

### Prompt

- **[AGNÓSTICO] El recordatorio va al final del mensaje de usuario.** El compositor lo permite sin
  romper el orden estático→variable (va detrás de la pregunta). La sesión guarda la pregunta pelada.
- **[DOMINIO] El system prompt explica el bloque** (qué es `<source>`, `<item>`, `authority`,
  `type`) antes de las reglas, y la regla 5 se refuerza junto a la frase marcador: «Not covered…
  refers to the sources you received, not to the ECSS standards as a whole». Las dos frases marcador
  quedan exactas.

### Marcadores, historial y validación

- **[AGNÓSTICO] En el historial, un ref fuera de rango se reescribe a `[?]`.** Dejar `[7]` crudo
  reintroduce el problema de D11 (en el turno siguiente puede existir una fuente 7); inventarle una
  cita es peor. `[1, 3]` pasa a `[A §x; B §y]`.
- **[AGNÓSTICO] Un par `"…" [1, 3]` es `verified` si la cita está en cualquiera de sus grupos
  válidos**, e `invalid_ref` solo si ninguno lo es.
- **[AGNÓSTICO] Se quita la puntuación final (`.,;:`) de la cita antes de comparar:** el punto
  dentro de las comillas suele ser del modelo, no de la fuente.
- **[AGNÓSTICO] Guion de partición = letra + guion + espacio + minúscula**, aplicado igual a los dos
  lados. Un guion suspendido legítimo («safety- and») se une en ambos y sigue casando.
- **[AGNÓSTICO] `citations_validated` cuenta marcadores también en un rechazo**, con el campo
  `refusal` al lado (ver §2.2). Los conteos son por aparición; `cited_clauses`, por cláusula.

### Fixture, frontend y configuración

- **[DOMINIO] `app/context/` se elimina.** El fragmento de M1 sobrevive como fixture en
  `tests/app/fakes.py`, junto a dos chunks más (otra cláusula, y una nota de la misma) para que
  agrupación y fan-out se ejerciten de punta a punta. El texto de la nota es sintético.
- **[AGNÓSTICO] Frontend:** las cláusulas citadas se pintan como `[n]` en orden de `ref`, cada una
  plegable con sus átomos; lo recuperado y no citado va en un desplegable aparte, rotulado como
  tal. Una respuesta sin citas (rechazo o sin marcadores) sigue sin traza de usuario: la decisión
  de render no cambia.
- **[AGNÓSTICO] `RETRIEVAL_MODE` es `Literal`**, así que un modo inventado falla al arrancar.
  `RETRIEVAL_TOP_K` va de 1 a 50, el mismo rango que el `k` de `/search`.

---

## 4. Lo que queda abierto (va a `ARCHITECTURE.md` §7)

- **Seguimientos conversacionales**, ahora **prioritario** (§5.1): el retrieval recibe solo la
  pregunta actual.
- **Rechazos con contenido** (§5.2): una respuesta parcial con la frase de rechazo delante pierde
  sus citas.
- **La voz del producto** (§5.3): el modelo habla del material recibido, no del corpus.
- **Expansión de cláusula**, con los números de §1 y la causa de las notas huérfanas (§5.4).
- **Política ante una cita literal no verificada** (S11), con `citations_validated` midiendo ya y
  una primera lectura de soporte satisfactoria (§5.5).
- **Queries con identificador en modo `vector`** (§1, hallazgo lateral): sigue en la ponderación de
  la fusión.

---

## 5. Resultado de la UAT del usuario

Guion y respuestas literales en `S09-uat.md`. Seis casos en Streamlit. Los casos 2 (par B/C), 5
(fuera de dominio) y 6 (fuera de material) dan el resultado esperado. Los datos de log y de base
de esta sección los añadió CC, sin coste.

### 5.1 El seguimiento anafórico es un fallo de producto [AGNÓSTICO]

«And what does ECSS-Q-ST-80C say about it?» rechaza; la variante explícita («…about software
criticality category B?») responde bien. Veredicto del usuario: **si un pronombre no funciona, el
producto no es un chat**, sino un asistente de preguntas aisladas. La causa está aislada (el
retrieval no ve el historial), así que la reescritura de seguimientos pasa de diferida a candidata
prioritaria. La técnica la decide Claude Chat.

### 5.2 Rechazos con contenido rompen el modelo binario [AGNÓSTICO]

En los logs de la UAT, **4 de los 6 rechazos `out_of_material` llevaban citas literales
`verified`**. Son respuestas parciales con la frase de rechazo delante: el caso 1 aporta lo que sí
hay (`ECSS-E-ST-70-01C` [3], el handbook [4]) y aun así empieza por `Not covered…`. Como un
rechazo vacía `citations`, **el usuario ve `[3]` y `[4]` en el texto sin fuentes debajo**. La
decisión es de contrato: un tercer estado («respuesta parcial»), o que un rechazo que cita
conserve sus citas.

### 5.3 La voz deja ver la tubería [DOMINIO]

La muletilla «based on the provided material» (caso 3) y el «I would need access to the actual
normative clauses» (caso 1) tienen el mismo origen: el modelo habla del material que recibió, no
del corpus. Le pide al usuario algo que es trabajo del retrieval. Lo empujan las reglas 4 y 5 del
prompt y la frase marcador `Not covered by the provided material:`, que llega tal cual al
usuario. Lectura del usuario: que el retrieval se quedó corto es señal **para el developer** (está
en los logs), y mostrarla erosiona la confianza. El aviso de cobertura sigue siendo honesto si se
formula sobre el corpus indexado y no sobre el prompt. Cambiar la frase marcador toca
`_refusal_kind` y su contrato.

### 5.4 Notas huérfanas arriba del ranking: el header sintético [DOMINIO]

En el caso 1, los dos hits más cercanos son notas «Refer also to clause 5.4.3.3.». Su `embed_text`
es casi todo cabecera («…ST[06] memory management > … Memory accessibility») más 29 caracteres de
texto: **el vector representa el título de la sección, no la nota**, y el título casa con «memory
management». Afecta a todo chunk diminuto. Dos vías: fusionar la nota con su requisito en el
chunking (re-chunkea y re-embebe; agrupable con el tokenizador real) o que la nota huérfana
arrastre su requisito en la expansión de cláusula (§1).

### 5.5 Pertenencia, literalidad y soporte de las citas [AGNÓSTICO]

| Nivel | Cómo | Resultado |
|---|---|---|
| Pertenencia (`[n]` existe) | Automático | 0 marcadores inválidos en toda la UAT |
| Literalidad (comillas en la fuente) | Automático | 0 `not_found` en toda la UAT |
| Soporte (la fuente respalda la frase) | El usuario, leyendo las fuentes de los casos 2 y 3 | **Satisfactorio**: el texto reproduce o parafrasea con precisión el verbatim; ningún `[n]` atribuye a una cláusula algo que no dice |

Alcance: veredicto cualitativo sobre dos respuestas, no una tasa frase a frase. La síntesis final
sin marcador de una respuesta queda fuera de este método. Para S11 es una primera señal de que el
modelo no fuerza las fuentes; la política ante una cita no soportada necesitará una muestra
mayor y medida.

### 5.6 Menores

- **Caso 3, handbooks omitidos** pese a haberse recuperado: coherente con la regla 3 (la pregunta
  es qué estándares *aplican*, y un handbook no obliga). Mostrarlos como guía sería decisión de
  producto.
- **Anomalía sin identificar:** una respuesta no rechazada **sin ningún marcador**
  (`refusal=None`, `valid_markers=0`, 4 cláusulas inyectadas), presentada como anclada y sin
  «Sources». Sin timestamp no se ha podido asociar a una pregunta.
