# S11 · Decisiones — Brief 4: cierre — cita sin repetición, «no encontrado» legible, rechazo limpio, reproducibilidad

> Decisiones tomadas al implementar `S11-brief-4.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
>
> **Verificación:** `uv run pytest` **1280 verde** (1263 + 17), `ruff` igual que antes (el E402
> conocido). Medición final **VÁLIDA** (`generation_20261019-141324`): 42/42 comparables, 0
> fallback, 0 errores, 0 esquemas inválidos, 0 recortes, 0 truncadas. **El criterio preregistrado
> pasa entero** (`comparison_20261019-141336.md`). En la API de Docker: el 422 llega en 50 ms y el
> mapa de modelos es el local.

---

## 1. Implementación

- **[AGNÓSTICO] K1 — cita o paráfrasis, en el prompt.** La regla 2 de `system.md` pasa a «Quote or
  paraphrase, never both»: si una afirmación solo transmite lo que dice un requisito, su texto **es**
  el tramo literal entre comillas; palabras propias solo para sintetizar, comparar o combinar. Las
  descripciones de `text` y `not_found` en el esquema dicen lo mismo, porque el modelo también las
  lee.
- **[AGNÓSTICO] K2 — la red, también para la cita dentro del texto.** El brief la acota a «si una
  cita verificada iría aparte», pero el caso de la UAT (`…, as stated: "…" [8]`) era una cita
  **dentro** del texto escrita por el modelo, que esa red no habría cazado. Se aplica a los dos casos
  con un solo criterio, `verification.adds_nothing`: las palabras propias de la afirmación (el texto
  entero si la cita va aparte; el texto sin sus tramos entrecomillados si va dentro), normalizadas y
  sin palabras de relleno (artículos, preposiciones, auxiliares, **modales** — manda la redacción de
  la fuente — y conectores como «as stated», «according to»), tienen que estar en la cita en al menos
  un **90 %**. Si es así, la afirmación se sirve como la cita sola con su marcador (`quote_only`,
  contado en `quotes_collapsed`), y los refs servidos pasan a ser los de la cita. Ante la duda se
  muestran las dos: sin palabras de contenido propias, o por debajo del umbral, no actúa; tampoco con
  una cita no literal. Los tres ejemplos de la UAT colapsan; «Category B requires 100 % statement
  coverage while category C…» frente a la fila de la tabla, no.
- **[AGNÓSTICO] K3 — «no encontrado» como bloque.** `I could not find in the indexed ECSS
  standards:` y una viñeta por aspecto, en bloque aparte tras el cuerpo; sin `not_found`, sin
  bloque. El historial guarda el mismo bloque, sin marcas.
- **[AGNÓSTICO] K4 — `not_found` en términos de la pregunta**, en el prompt: «naming what the user
  asked for in the user's own terms (for example "requirements that apply only to category C"),
  never describing the content or structure of a standard».
- **[AGNÓSTICO] K5 — rechazo limpio.** `presentation.show_evidence(status)`: solo `answered` y
  `partial` pintan «Cited» / «In context, not cited». La evidencia sigue en el contrato y en el panel
  de diagnóstico.
- **[AGNÓSTICO] K6 — mapa de modelos empaquetado.** `app/__init__.py` hace
  `os.environ.setdefault("LITELLM_LOCAL_MODEL_COST_MAP", "True")` antes de que nada importe LiteLLM:
  vale en la imagen (bind mount de `app/`) y en local (`uv run`, arnés incluido), y un valor explícito
  en el entorno sigue mandando. Comprobado con `get_model_cost_map_source_info()`: origen `local`,
  forzado por la variable, sin URL, revisión `d9ea43c5…`; en la API: `local True`. Precio y vía de
  los dos modelos, intactos (Haiku 5.5 sigue nativo).
- **[AGNÓSTICO] K7 — máximo de la pregunta: 4.000 caracteres.** `QueryRequest.question` con
  `max_length=MAX_QUESTION_CHARS`: 422 de validación antes de guardrails, sesión, retrieval o modelo.
  **Por qué 4.000:** ~1.600 tokens estimados (÷2,5), 20 veces el turno más largo del golden set (183
  caracteres), y con el prompt fijo (2.613 estimados) suma ~4.200, muy por debajo de cualquier
  presupuesto válido. Para que «una pregunta aceptada siempre cabe» valga en toda configuración, el
  suelo de `MAX_INPUT_TOKENS` sube de 1.000 a **10.000**; un test fija el invariante. Con eso la
  rama `context_budget_exceeded` y su `TODO` desaparecen de `_fit_budget`. El frontend explica el 422
  con el límite que devuelve el backend («Message too long: the limit is 4,000 characters. Ask about
  one topic at a time.»).
- **[AGNÓSTICO] K8 — citadas / inyectadas en el arnés:** columna por ejecución, tabla por consulta
  (media de ejecuciones) y global (sumadas), y en `compare.py` junto al criterio nuevo de citas fuera
  de su afirmación (tienen que bajar). El arnés lista además los aspectos `not_found` de cada
  respuesta y cuenta las actuaciones de K2.
- **[DOMINIO] K9 y K10, documentados.** La caché de prefijo se queda activa en producción; la caída
  de precisión del brief 3 queda en `ARCHITECTURE.md` como excepción explícita al criterio
  preregistrado, con su porqué.
- **[AGNÓSTICO] Un error mío, ya corregido:** el primer borrador del test «pregunta en el límite se
  acepta» no simulaba el modelo e hizo **una llamada real** a Haiku 5.5 (4.000 «x», <0,001 USD) desde
  la suite de contrato. Lo detecté por el tiempo (4,2 s frente a 0,07 s); el test ya simula
  `_call_model` y comprueba que se llamó una vez.

## 2. Criterio preregistrado — [DOMINIO]

Referencia `generation_20261018-212343` (Haiku 5.5, brief 3) frente a `generation_20261019-141324`.

| métrica | referencia | final | Δ global | Δ emparejado | veredicto |
|---|---|---|---|---|---|
| recall condicionado | 0,91 [0,89–0,93] | **0,91** [0,91–0,92] | −0,00 | −0,00 | PASA |
| precisión de cita | 0,85 [0,84–0,86] | **0,86** [0,85–0,87] | +0,01 | +0,01 | PASA |
| tasa `verified` | 1,00 | **1,00** | +0,00 | +0,00 | PASA |
| rechazos servidos con citas | 0 | **0** | | | PASA |
| esquemas inválidos servidos | 0 | **0** | | | PASA |
| citas fuera de su afirmación | 46 de 249 | **26 de 233** | | | PASA (bajan) |

| consulta | estrato | Δ recall | Δ precisión | Δ `verified` | citadas / inyectadas |
|---|---|---|---|---|---|
| q01 | names_standard | −0,11 | +0,00 | +0,00 | 0,47 |
| q02 | names_standard | +0,00 | +0,00 | +0,00 | 0,89 |
| q03 | generic | +0,11 | +0,00 | +0,00 | 0,78 |
| q04 | generic | +0,00 | +0,00 | +0,00 | 0,86 |
| q05 | paraphrase | +0,00 | +0,03 | +0,00 | 0,83 |
| q06 | comparative | +0,00 | +0,08 | +0,00 | 0,56 |
| q07 | comparative | +0,00 | +0,00 | +0,00 | 1,00 |
| q08 | cross_standard | −0,08 | −0,04 | n/a | 0,85 |
| q09 | cross_standard | +0,06 | +0,00 | +0,00 | 0,47 |
| q10 | multi_turn | +0,00 | +0,00 | +0,00 | 1,00 |
| q11 | names_standard_xref | +0,00 | +0,00 | +0,00 | 0,46 |
| q12 | names_standard_xref | +0,00 | +0,00 | +0,00 | 0,97 |
| q13 | paraphrase | +0,00 | +0,00 | +0,00 | 0,57 |
| q14 | multi_turn | +0,00 | +0,11 | +0,00 | 0,48 |
| **global** | | | | | **0,73** (233 / 318); referencia 0,74 |

**Lectura (K8):** el modelo cita casi todo lo que recibe en q07, q10 y q12 (0,97–1,00) y la mitad
en q01, q09, q11 y q14. El 0,97 de q12 es la consulta cuya precisión cayó en el brief 3: la razón
lo deja a la vista, que es para lo que existe. La global no se mueve (0,74 → 0,73).

## 3. K2: cuántas veces actuó, y qué era — [DOMINIO]

**38 actuaciones en 365 afirmaciones**, pero no son 38 repeticiones:

| tipo | n | ejemplo |
|---|---|---|
| El texto **ya es** la cita, escrita sin comillas (K1 cumplido en el fondo, no en la forma): se sirve entre comillas | **35** | q01 ej. 2: «The applicable technical review process shall be specified by the supplier.» |
| Paráfrasis + la misma cita aparte | 2 | q05 ej. 2: «Changes to information and documentation, and their justification, shall be traceable…» frente a la cita literal |
| Paráfrasis + la misma cita entre comillas en el texto (el caso de la UAT) | 1 | q07 ej. 2: «In agile, PDR can focus on the flexibility and scalability…: "PDR Can serve to focus on…"» |

Las repeticiones reales pasan de ser el patrón de la UAT a **3 en 42 respuestas**, y las tres se
sirven sin repetir. Las 35 de formato no se habrían visto como repetición, pero K2 las mejora: el
usuario ve que es texto literal del estándar.

## 4. `not_found` antes y después (K3, K4) — [DOMINIO]

| | referencia | final |
|---|---|---|
| Aspectos por respuesta (media) | 2,40 | **1,55** |
| Respuestas con algún aspecto | 39 / 42 | 33 / 42 |

Antes (descripción del contenido del estándar) → después (lo que se preguntó):
- q06: «contents of the ECSS-Q-ST-80 Table D-2 applicability matrix for categories B and C» →
  «verification and testing requirements that distinguish category C from category B beyond code
  coverage».
- q01: «the specific documentation expected at each individual review, … as the document requirements
  list in Table A-1 is not in the sources» → «the specific documentation expected at each software
  project review».
- q14: «the requirements of the ECSS-E-ST-70-01 standard for on-board control procedures» →
  «flight software capabilities needed to execute on-board control procedures».

Se compone como bloque aparte con viñetas. Si se lee bien, lo dice la UAT (`S11-uat-4.md`, caso 2).

## 5. Estados, tokens, latencia, coste — [DOMINIO]

- **Declarado → servido:** answered → answered 9, answered → partial 2, partial → partial 31.
- **Tokens:** entrada mediana 6.624, máx 8.478 (0 recortes); salida mediana 2.880, **máx 7.118** (en
  el brief 3 una ejecución agotó los 16.000; ahora ninguna se trunca).
- **Latencia del turno juzgado:** mediana **12,8 s**, p95 20,0 s (brief 3: 13,7 / 19,4).
- **Coste:** 0,0018 USD por turno en mediana; pasada completa 0,081 USD. **Coste de la sesión ≈ 0,08
  USD** (la pasada más la llamada accidental del test, <0,001 USD); sin sondas.

## 6. Recuento de tests

| fichero | antes | después | qué |
|---|---|---|---|
| `test_closing_rules.py` (nuevo) | — | 10 | K2 (la repetición de la UAT, cita aparte igual, afirmación distinta, síntesis, no literal, sin palabras propias), K7 (invariante con el suelo, 422 sin efectos, límite aceptado), K6 (mapa local) |
| `test_composition.py` | 13 | 14 | K3: bloque con viñetas; sin `not_found`, sin bloque |
| `test_generation_harness.py` | 27 | 30 | K8: razón, informe y artefacto; criterio de citas fuera en `compare` |
| `test_errors.py` (frontend) | 7 | 9 | 422 por longitud con el límite y sin código HTTP; otro 422 lo conserva |
| `test_presentation.py` (frontend) | 5 | 6 | K5: un rechazo no muestra evidencia |
| **Total** | **1263** | **1280** | |

Además: snapshot del prompt regenerado (K1, K4) y los asserts de «no encontrado» pasan al bloque.

## 7. Para el usuario

- **UAT:** `S11-uat-4.md` (caso 1 de la frontera: los tres defectos se vieron leyendo la interfaz).
- **Ningún `TODO` nuevo**; el de `_fit_budget` queda cerrado por K7.

## 8. Ajustes tras la UAT (`S11-uat-4.md`)

- **[AGNÓSTICO] Icono ⚠️ delante de `I could not find in the indexed ECSS standards:`** para resaltar
  el bloque. Va en la respuesta servida (`compose(markers=True)`); el turno que guarda la sesión, que
  lee el modelo, va sin él.
- **[AGNÓSTICO] Pregunta demasiado larga, en términos del usuario:** «Your question exceeds the input
  limit of 4000 characters. Please summarize it or ask one topic at a time.», sin «HTTP 422». Lo
  mismo para la demasiado corta («Your question is too short. Please add a few words.»); un 422 que
  no trata de la longitud de la pregunta conserva el estado HTTP.
- **[DOMINIO] Mejora de UI anotada, sin implementar:** los chunks de la traza a veces llegan sin
  saltos de línea y las tablas aplanadas son palabras separadas por `|`. Pendiente en
  `ARCHITECTURE.md` §7: dar formato al render sin tocar el verbatim.
- Casos 1 a 4: OK (el caso 1, q02 sin cita y paráfrasis repetidas, confirmado por el usuario).
