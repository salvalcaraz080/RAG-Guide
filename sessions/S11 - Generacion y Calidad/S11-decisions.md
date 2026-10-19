# S11 · Decisiones — Brief 1: arnés de generación con métricas deterministas

> Decisiones tomadas al implementar `S11-brief.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
>
> **Verificación:** `uv run pytest` **1217 verde** (1169 + 48), `ruff` igual que antes (el E402
> conocido de `app/main.py`). Ejecución de referencia **VÁLIDA**: 42/42 ejecuciones comparables,
> 0 fallback, 0 errores, 0 pares sin juicio. Artefacto de referencia:
> `data/groundtruth/generation/runs/generation_20261018-164057.{json,md}`.

---

## 1. Decisiones de implementación

- **[AGNÓSTICO] G1 tal cual.** `validate_quotes` devuelve un veredicto por par (`QuoteCheck`: cita,
  refs escritas, veredicto, `found_in`) y los recuentos salen de ahí, así que los tests y el log
  existentes no cambian de forma. Si la cita no está en el grupo citado, se busca en **todos los
  demás** grupos inyectados con la misma normalización; `found_in` lista todos los que la contienen.
  Precedencia `verified` > `misattributed` > `not_found`; un marcador sin ref válida es `invalid_ref`
  aunque la cita exista en otro grupo (el marcador ya está roto, no hay atribución que juzgar). El
  log lleva `misattributions: [{cited, found_in}]` — refs, nunca el texto citado, como el resto del
  log (metadata, no contenido). Un test existente cambió su expectativa: «cita de otra cláusula →
  `not_found`» pasa a `misattributed`, que es exactamente el caso que G1 separa.
- **[AGNÓSTICO] G2 en un módulo propio, `app/services/statements.py`,** junto a `citations` y
  usando su `contains_marker`. Las dos frases marcador de rechazo se mueven de `llm_service` a
  `citations`, porque el detector las necesita y `citations` no puede importar `llm_service` (ciclo).
  Cableado en `citations_validated` (`factual_statements`, `unmarked_statements`,
  `absence_statements`) y expuesto al arnés. Reglas en §4.
- **[AGNÓSTICO] G3: el arnés lee la respuesta del producto, no su interior.** Llama a
  `llm_service.answer_query` (parámetro inyectable; un test fija que el default es esa función) y
  todo sale del dict que devuelve: `citations` y `retrieved_context`. Los grupos `[n]` se
  reconstruyen agrupando la evidencia, que viaja en orden de `[n]`, con la misma `group_by_clause`;
  cada cita se mapea por la tabla `documents` (`doc_id_cite` → `canonical_doc_id`) y se comprueba
  contra el grupo de su `ref`. Una cita que no mapea, que discrepa de su grupo o que no está entre
  las inyectadas es un `HarnessError`/`CitationOutsideContext`, no una cita irrelevante.
  Alternativa descartada: envolver el proveedor de contexto para capturar los grupos — habría
  medido lo que el servicio vio en vez de lo que el producto entrega, y el mapeo por `documents` que
  pide el brief habría sobrado.
- **[AGNÓSTICO] Multi-turn con entrada idéntica entre ejecuciones.** Una consulta multi-turn
  contesta sus turnos previos **una vez**, en una sesión real, y antes de cada una de las 3
  ejecuciones abre una sesión nueva con ese mismo historial; solo varía el turno juzgado, que es lo
  que define el suelo de ruido. Coste: **4 llamadas más que las 42 autorizadas** (2 consultas × 2
  turnos previos), ~0,02 USD; regenerar el historial en cada ejecución habría costado 12 y mezclado
  dos fuentes de variación. Consecuencia: entre dos pasadas, el historial de las multi-turn sí cambia.
- **[AGNÓSTICO] Agregación por índice de ejecución.** Para cada ejecución `i` se calcula la cifra
  del estrato (media macro sobre consultas para recall y precisión, como el arnés de retrieval;
  recuento conjunto para las tasas de `verified` y de frases sin marcador, que son conteos) y se
  reporta la media y el rango de esas 3 cifras. Una ejecución con fallback sale del índice en que
  ocurrió; el índice agrega las consultas que quedan. Por consulta, el informe da cada ejecución.
- **[AGNÓSTICO] Artefacto autosuficiente.** Además de lo que pide el brief (respuesta, citas,
  cláusulas inyectadas, veredictos, frases, `finish_reason`, tokens, `latency_ms`, proveedor,
  fallback), cada cláusula inyectada lleva su juicio y el **texto de sus chunks**: cualquier
  veredicto se comprueba sin la base, y se puede re-puntuar sin volver a llamar al modelo. Cabecera:
  `input_fingerprint`, SHA-256 del prompt de sistema **y del mensaje de usuario** (lleva el
  recordatorio de citas, que también cambia la generación), modelo y fallback, configuración,
  versión del golden set y del protocolo, `LLM_EVAL_MODE`, fecha. Se escribe con modo `x`: si el
  fichero existe, falla en vez de sobrescribir.
- **[AGNÓSTICO] La exportación de pares se reutiliza, no se duplica.** `unjudged_pairs` del arnés de
  retrieval se parte en dos: `pairs_file(pairs, golden)` construye el fichero del modo B desde
  `(query_id, cláusula)`, y los dos arneses lo llaman. Los tests de retrieval no cambian.
- **[AGNÓSTICO] Configuraciones como lista; solo `defaults`.** `defaults` usa
  `get_context_provider()`, el mismo proveedor que sirve `/query`. `bounded` será una entrada más.
- **[AGNÓSTICO] Una segunda pasada, autorizada.** La primera ejecución (`generation_20261018-163222`)
  salió válida, pero al leer sus frases sin marcador aparecieron dos fallos del detector (§4). Se
  corrigieron con tests y se repitió la ejecución entera (autorizado: 46 llamadas más) para tener un
  solo artefacto coherente con el código. La primera se conserva: para recall, precisión y
  veredictos, que el detector no toca, es una segunda muestra del ruido (§2).
- **[AGNÓSTICO] `data/groundtruth/generation/` a `.gitignore`.** Solo estaba ignorado
  `data/groundtruth/retrieval/`; los artefactos salían como no rastreados.

## 2. Suelo de ruido por estrato — [DOMINIO]

Defaults: `vector` + `filter` + `tokenized_length` + `orphans`, `top_k=10`, `claude-haiku-4-5`.
Cada celda: media [mín–máx] de las 3 ejecuciones (`generation_20261018-164057`).

| estrato | n | recall condicionado | precisión de cita | tasa `verified` | frases sin marcador |
|---|---|---|---|---|---|
| names_standard | 2 | 0,83 [0,83–0,83] | 0,94 [0,83–1,00] | 0,94 [0,92–1,00] | 0,17 [0,15–0,21] |
| names_standard_xref | 2 | 0,68 [0,67–0,71] | 0,58 [0,58–0,58] | 0,95 [0,95–0,95] | 0,19 [0,13–0,25] |
| generic | 2 | 0,64 [0,61–0,69] | 1,00 [1,00–1,00] | 0,93 [0,87–1,00] | 0,13 [0,06–0,19] |
| paraphrase | 2 | 0,68 [0,52–0,76] | 0,87 [0,76–0,93] | 0,89 [0,80–0,93] | 0,24 [0,20–0,32] |
| comparative | 2 | 0,52 [0,43–0,60] | 1,00 [1,00–1,00] | 1,00 [1,00–1,00] | 0,34 [0,29–0,41] |
| cross_standard | 2 | 0,49 [0,42–0,65] | 1,00 [1,00–1,00] | 0,90 [0,83–1,00] | 0,19 [0,12–0,31] |
| multi_turn | 2 | 0,69 [0,61–0,86] | 1,00 [1,00–1,00] | 0,96 [0,88–1,00] | 0,25 [0,22–0,27] |
| **todas** | 14 | **0,65** [0,63–0,68] | **0,91** [0,90–0,93] | **0,94** [0,92–0,95] | **0,21** [0,19–0,26] |

**Las 6 ejecuciones de las dos pasadas** (recall, precisión y `verified`; la primera pasada tiene
otro historial en multi-turn):

| estrato | recall | ancho | precisión | ancho | `verified` | ancho |
|---|---|---|---|---|---|---|
| names_standard | 0,85 [0,77–1,00] | 0,23 | 0,95 [0,83–1,00] | 0,17 | 0,92 [0,89–1,00] | 0,11 |
| names_standard_xref | 0,73 [0,67–0,83] | 0,17 | 0,58 [0,50–0,62] | 0,12 | 0,95 [0,95–0,96] | 0,01 |
| generic | 0,61 [0,54–0,69] | 0,15 | 1,00 | 0,00 | 0,95 [0,87–1,00] | 0,13 |
| paraphrase | 0,64 [0,52–0,76] | 0,24 | 0,80 [0,68–0,93] | 0,25 | 0,81 [0,69–0,93] | 0,24 |
| comparative | 0,61 [0,43–0,90] | **0,47** | 0,98 [0,88–1,00] | 0,12 | 0,99 [0,92–1,00] | 0,08 |
| cross_standard | 0,47 [0,42–0,65] | 0,23 | 1,00 | 0,00 | 0,92 [0,80–1,00] | 0,20 |
| multi_turn | 0,70 [0,61–0,86] | 0,25 | 1,00 | 0,00 | 0,96 [0,86–1,00] | 0,14 |
| **todas** | 0,66 [0,63–0,69] | **0,05** | 0,90 [0,88–0,93] | **0,06** | 0,93 [0,90–0,95] | **0,04** |

**Lectura:**
- **[AGNÓSTICO] El suelo de ruido global es ~±0,03** en las tres métricas comparables (ancho 0,04–0,06
  en 6 ejecuciones). Una diferencia A/B global menor que ~0,05 no se distingue del ruido con N=3.
- **[AGNÓSTICO] Por estrato, 3 ejecuciones subestiman el ruido.** El rango dentro de una pasada
  llega a 0,25; con 6 ejecuciones, a 0,47 (recall de `comparative`: q07 cita 1 de 5 relevantes en
  unas ejecuciones y 4 de 5 en otras). Con 2 consultas por estrato, **un estrato aislado no decide
  nada**: el criterio preregistrado de `bounded` («gana en omisión en el estrato normativo») debería
  pedir la comparación pareada por consulta y más ejecuciones, o leerse sobre el agregado.
- **[DOMINIO] El recall condicionado es el eje que más se mueve; la precisión casi no.** El modelo
  rara vez cita algo irrelevante (0,91), pero deja sin citar un tercio de lo relevante que recibe
  (0,65). La excepción en precisión es `names_standard_xref` (0,58, estable): en q11 y q12 cita
  cláusulas que el golden set juzga no relevantes. `cross_standard` tiene el recall más bajo (0,49):
  con 6–8 relevantes inyectadas, la respuesta cita 2–5.
- **[DOMINIO] Las métricas leen mal la atribución falsa entre relevantes**: los 2 `misattributed`
  (§3) citan una cláusula relevante (`[1]`) con texto de otra relevante (`[5]`). La precisión no lo
  ve, porque `[1]` es relevante. El recall sí penaliza que `[5]` quede sin citar (q13 ej. 3: 0,33),
  pero lo cuenta como **omisión**, cuando el contenido de `[5]` sí se usó, mal atribuido. Es la razón
  para mantener el veredicto literal como métrica aparte.

## 3. Veredictos literales — [DOMINIO]

| | verified | misattributed | not_found | invalid_ref | marcadores sin cita literal |
|---|---|---|---|---|---|
| Referencia (42 respuestas) | 262 | 2 | 16 | 0 | 21 |
| Primera pasada (42 respuestas) | 234 | 7 | 14 | 0 | 47 |

Tasa sobre pares cita–marcador (referencia): `verified` 0,94, `misattributed` 0,007, `not_found`
0,057.

**Todos los `misattributed`** — los 9 de las dos pasadas son el mismo caso, q13 (paráfrasis: «fix
bugs in the onboard code and send up a corrected version»), y la misma pareja de cláusulas hermanas
de `ECSS-E-HB-40A`, **las dos juzgadas relevantes**:

| pasada | ejecución | cita | citada | donde está |
|---|---|---|---|---|
| referencia | 3 | "The patch mechanism design and development is part of the initial software development process and can be validated as such during the qualification process" | [1] §5.10.2.2.1 | [5] §5.10.2.2.2 |
| referencia | 3 | "It is highly recommended during validation of the modification to exercise the patch procedure itself to verify its robustness and efficiency" | [1] §5.10.2.2.1 | [5] §5.10.2.2.2 |
| primera | 1, 2, 3 | las mismas dos frases (×2 por ejecución) | [1] §5.10.2.2.1 | [5] §5.10.2.2.2 |
| primera | 3 | "the correct mapping in the patch application i.e. not patched zone needs to have an ISO-mapping demonstrated. The patch needs to be correctly linked in the hosting application" | [1] §5.10.2.2.1 | [5] §5.10.2.2.2 |

Contexto: `§5.10.2.2.1` (rango 1) es la sección de recarga de software en vuelo, cuatro chunks
informativos; `§5.10.2.2.2` (rango 5) es la de parches, y contiene las tres frases. El modelo
organiza la respuesta en torno a la cláusula de rango 1 y le atribuye el texto de la hermana. Patrón
para D-1: **la atribución falsa aparece entre cláusulas contiguas del mismo documento y del mismo
tema**, donde la etiqueta `[n]` se pega a la primera fuente del tema. Comprobado sobre el texto del
artefacto: las tres frases están en `[5]` y en ningún chunk de `[1]`.

**Los 16 `not_found` de la referencia, uno a uno** (comparando la cita con su fuente):

| tipo | n | casos |
|---|---|---|
| Límite de la normalización (la cita es literal salvo espacios o tipo de comilla) | 7 | q02 ×2: `'as-is'` frente a `"as-is"` en la fuente; q05 ×3: la fuente tiene `information /documentation`; q09: `'commercial off-the-shelf'` frente a `"…"`; q09: `phase – definition` donde la fuente dice `phase -definition` |
| Elisión dentro de las comillas | 4 | q11 ×3: omite «(5.2.4.7)» a mitad de cita; q03: omite «(ECSS-Q-ST-80C Annex D)» |
| Inserción o cambio dentro de las comillas | 4 | q03: «[the components]» por «them»; q08: `ECSS-Q-HB-80-03A` donde la fuente dice `ECSS-Q-HB-80-03`; q10: un punto entre dos ítems pasa a punto y coma; **q04: inserta «shall»** («and *shall* not fully or partly replace» donde la fuente dice «and not fully or partly replace») |
| Reordenación | 1 | q09: «ECSS-E-ST-40 clause 5.4.3.7 requires» por «5.4.3.7 of ECSS-E-ST-40 requires» |

- **[DOMINIO] Ningún `not_found` es fabricación pura**: en los 16 la fuente citada contiene la
  frase o casi. Para la política del brief 2, `not_found` mezcla dos cosas: falsos negativos del
  validador (7, arreglables unificando `'`/`"` e ignorando el espacio junto a `/` y a guiones en
  `normalize`, en los dos lados) y **ediciones del modelo dentro de las comillas** (9), que el
  prompt prohíbe. Un rechazo automático por `not_found` hoy rechazaría 7 citas literalmente
  correctas.
- **[DOMINIO] La inserción de «shall» en q04 es inflación de autoridad detectada por la capa
  literal**: el texto citado pasa de descriptivo a obligación. Es un argumento para que la política
  no trate igual todos los `not_found`: una edición que añade o quita una palabra modal no es una
  elisión cosmética. No se implementa aquí (`S11-brief-2`).
- **[AGNÓSTICO]** Cambiar `normalize` ahora movería las mediciones de S09–S10 y el importador del
  golden set, que comparten la función; queda como propuesta para el brief 2, medible re-puntuando
  este artefacto sin coste.

## 4. Detector de frases sin marcador (G2) — reglas finales y ejemplos

**Reglas** (docstring de `app/services/statements.py`):
1. Una cita entre comillas que ocupa varias líneas se une en una línea (hasta una línea en blanco,
   que cierra una comilla suelta): es una cita, no una frase por línea.
2. Un ítem de lista (`-`, `*`, `•`, `1.`, `1)`, `a.`, `a)`) es **una** frase, tenga las oraciones que
   tenga. El resto de líneas se parten en oraciones en `.`, `!` o `?` (con los marcadores que le
   sigan) seguidos de mayúscula, **nunca dentro de comillas**.
3. **Con marcador** = contiene un `[n]` en cualquiera de sus formas (`[1]`, `[1, 3]`, `[1][3]`). Una
   cita natural `[ECSS-… §…]` no cuenta.
4. **Excluidas** (fuera del denominador): la oración que abre con una frase marcador de rechazo
   (solo al principio de la respuesta); la que dice que la búsqueda **no encontró** algo («could not
   find», «did not find», «was not found») — la forma que el prompt pide en vez de afirmar ausencia —;
   encabezados (`#…` o línea entera en negrita); oraciones que terminan en `:` (introducen lista);
   conectores sueltos de una lista cerrada («In summary», «However», «Additionally»…). Ante la duda,
   factual.
5. Filas separadoras de tabla fuera; el resto de filas son frases.
6. **Ausencia** (contador aparte, sobre frases factuales, con o sin marcador): «does/do/did not
   (explicitly) require/specify/define/mandate/address/cover/…», «is/are not (explicitly)
   specified/defined/required/…», «no (specific/explicit/such/additional/formal/particular/detailed)
   requirement/guidance/obligation/mention», «not explicitly». Se busca **fuera de las comillas**.

**Afinado tras leer las frases de la primera pasada** (con test cada uno):
- Los ítems `a.`, `b.`… no se reconocían como lista y la segmentación dejaba «a.», «b.» como frases
  sueltas sin marcador (q11: 20 falsos «sin marcador»).
- Una cita literal de varias líneas (la lista de contenidos del informe de problemas de q02) se
  troceaba en 5 fragmentos sin marcador.
- Dos falsos positivos de ausencia: «with no compensating provisions» (texto de la cláusula, fuera de
  comillas) y «requirements are not covered by test» (paráfrasis de la cláusula). Se restringe `no …`
  a una lista cerrada de calificadores y se ignora lo entrecomillado.

Efecto sobre las mismas 42 respuestas: sin marcador 98 → 73, ausencias 22 → 20. En la referencia:
**342 frases factuales, 74 sin marcador (0,21), 12 de ausencia**; excluidas: 128 encabezados, 24
introducciones de lista, 4 frases marcador de rechazo, 2 «could not find».

**Ejemplos reales contados como sin marcador** (referencia):
- Síntesis o marco sin fuente: «The criticality category of a software item is determined through
  a system-level analysis based on the functions it implements and the presence of compensating
  provisions.» (q03) · «Once identified, risks are addressed through targeted engineering
  measures.» (q12) · «In waterfall development, reviews mark discrete phase transitions with formal
  milestones.» (q07) · «Your choice depends on trade-offs made early in design.» (q13).
- Comparaciones derivadas: «Category C software allows flexibility in code coverage targets (to be
  negotiated with the customer) and does not require independent verification and validation by a
  third party.» (q06) — contenido de dos cláusulas citadas antes, reformulado sin marcador.
- Remisiones al usuario, **que incumplen la regla de voz 7**: «To obtain the complete definition …
  you would need to consult the full standard, particularly Annex A (Table A-1)…» (q01, las 3
  ejecuciones) · «To find those details, you would need to consult ECSS-E-ST-70-01…» (q14).
- Menciones del contexto, **que incumplen la regla 6**: «…which is referenced but not fully indexed
  in the material available to me.» (q14) · «…is not contained in the indexed ECSS standards
  provided to me…» (q09).
- Frase de título que el detector no aparta: «According to ECSS-E-ST-40C, the software technical
  specification should contain a comprehensive range of software requirements.» (q11) — es un
  resumen de lo que sigue; cuenta como factual por la regla conservadora.

**Ejemplos reales excluidos:**
- `list_intro`: «According to the indexed ECSS standards, Q-ST-80C Rev.2 makes the following
  mandatory:» (q10) · «Three rules apply to how each requirement shall be written:» (q11).
- `not_found`: «I could not find in the indexed ECSS standards more detailed information on the
  specific mechanisms for escalating software risks into the project-level risk register…» (q12).
- `refusal_marker`: «I could not find this in the indexed ECSS standards:» (q01 ×3, q14 ×1).
- `heading`: «**Design and validation of the patch mechanism:**» (q13) · «## Rules for Individual
  Software Requirements» (q11).

**Ejemplos reales de ausencia** (12 en la referencia):
- Sin marcador, sobre el contexto: «However, the indexed ECSS standards do not specify what concrete
  capabilities or functional requirements the flight software must provide to execute these
  procedures.» (q14, las 3 ejecuciones) · «The indexed ECSS standards do not specify the exact
  composition of authorities or approval bodies…» (q05).
- Sobre el contenido de una cláusula: «Category C software … does not require independent
  verification and validation by a third party.» (q06) — afirma una ausencia normativa, no de
  búsqueda: es la extrapolación desde el silencio que S09 anotó.

**Límites conocidos (para que Claude Chat juzgue si mide lo que dice):**
- **[AGNÓSTICO] El recuento de ausencias es una cota inferior.** No ve «is not present in the
  indexed material», «is notably absent», «has no fixed mandatory targets», «does not apply», «are
  not fully covered» (todas en la referencia, contadas solo como factuales).
- **[AGNÓSTICO]** Una ausencia sobre la búsqueda («the indexed standards do not specify…») y una
  ausencia normativa («Category C does not require…») cuentan igual; la segunda es la peligrosa.
  Distinguirlas pide mirar el sujeto, que la heurística no hace.
- **[AGNÓSTICO]** La partición en oraciones no corta tras una inicial («category A. For …» se
  partiría bien; «e.g. The» no se parte, a propósito). Una frase de síntesis con un marcador al final
  cuenta como marcada aunque solo parte de ella venga de la fuente: el detector mide marcadores, no
  soporte.

## 5. Otras señales de la referencia — [DOMINIO]

- **Respuestas parciales con frase de rechazo delante:** q01 empieza por `I could not find this in
  the indexed ECSS standards:` en las 3 ejecuciones de las dos pasadas y aun así cita 2–3 cláusulas
  (relevantes). q14 lo hace en 1 de 3. Es el pendiente conocido del contrato tipado; ahora con tasa:
  4 de 42 respuestas llevan `refusal` y las 4 citan.
- **Las reglas de voz 6 y 7 se incumplen de forma estable** en las consultas cuya respuesta está
  fuera de lo recuperado (q01, q09, q14): remisión a «consult the full standard» y mención del
  «material available to me». Detectable con un `if`, como `misattributed`.

## 6. Pares sin juicio, fallback, coste y latencia

| | primera pasada | referencia |
|---|---|---|
| Pares sin juicio | 0 | 0 |
| Ejecuciones con fallback | 0 | 0 |
| Errores | 0 | 0 |
| Llamadas al modelo | 46 | 46 |
| Coste real (LiteLLM) | 0,2525 USD | 0,2562 USD |
| Latencia del turno juzgado (`latency_ms`) | mediana 5,57 s · p95 6,89 s | mediana 5,63 s · p95 7,55 s |

- **[DOMINIO]** 0 pares sin juicio en las dos pasadas, como se esperaba: con los defaults, el
  retrieval es determinista y el arnés de S10 ya juzgó lo inyectado.
- **[DOMINIO]** Coste total de la sesión: **0,51 USD** (dos pasadas). Por pasada, ~0,0055 USD por
  llamada, con ~4.070 tokens de entrada en mediana.
- **[DOMINIO]** La latencia queda algo por debajo de la referencia de pre-S11 (6,3 s), que medía
  `/query` por HTTP con multi-turn sin sesión; aquí es `latency_ms` de servidor del turno juzgado.

## 7. Recuento de tests

| | Tests |
|---|---|
| Partida | 1169 |
| `test_citations.py` | +5 (G1: en el grupo citado y en otro → `verified`; en ninguno → `not_found`; fuera de rango → `invalid_ref`; misma normalización; varios `found_in`) y 1 renombrado con expectativa nueva |
| `test_observability.py` | +1 (`misattributed`, `misattributions` y frases en `citations_validated`, sin texto citado) |
| `test_statements.py` (nuevo) | +22 (marcadores, listas, citas multilínea, exclusiones, ausencia) |
| `test_generation_harness.py` (nuevo) | +20 (métricas y `n/a`, mapeo `doc_id_cite` → canónico, cláusula sin juicio → inválida y par exportado, agregación y rango, fallback fuera, `answer_query` como default, multi-turn con historial repetido, fallo de proveedor registrado, artefacto e informe) |
| **Final** | **1217** |

## 8. Para el usuario

- **Comprobación del instrumento, hecha por CC:** recuento independiente de las 42 ejecuciones,
  sin importar nada del arnés, contra `golden_set.json` y el texto de las respuestas → 42/42
  coinciden con el informe, y los 2 `misattributed` se confirman sobre el texto (detalle en
  `S11-uat.md`). No es UAT: es mecánico y ejecutable.
- **UAT:** solo un juicio de política sobre el `misattributed` de q13 (caso 2: decisión del
  usuario), en `S11-uat.md`, con los textos delante. No bloquea el cierre del brief.
- **[DOMINIO] Decisión del usuario tras la UAT → insumo para `S11-brief-2`: un `misattributed` se
  reasigna.** «El usuario debe ver la cláusula correcta de donde salió cada cita.» Las dos cláusulas
  de q13 son necesarias para la pregunta (§5.10.2.2.1 enmarca la modificación en vuelo, §5.10.2.2.2
  concreta los parches); la cita mal dirigida le quita al usuario justo la subsección específica. El
  marcador se mueve a la cláusula donde está la cita (`found_in`), sin nueva llamada al modelo. No
  implementado: este brief es solo de observación. Queda para Claude Chat al concretar la política:
  - **`found_in` con más de un grupo** (la cita está en varias cláusulas): a cuál se reasigna, o a
    todas.
  - **Marcador compartido**: si el `[1]` mal dirigido también sostiene otra frase que sí está en
    `[1]`, reasignar el marcador entero rompería esa otra cita; la unidad de reasignación es el par
    cita–marcador, no el marcador.
  - **Efecto en `citations`**: la cláusula de `found_in` entra en `citations`, y la de origen sale
    solo si ningún otro marcador la sostiene.
  - **Medida**: el arnés tendría que puntuar después de reasignar; el recall de q13 pasaría de
    contar omisión a contar la cita bien dirigida.
- **Ningún `# TODO: pending decision in Claude Chat` nuevo**: nada exigió diseño fuera de alcance.
  Las propuestas para el brief 2 (unificar `'`/`"` en `normalize`, tratar aparte una edición que
  toca un modal, detectar las reglas de voz 6–7, ampliar el patrón de ausencia) están arriba, sin
  implementar.
