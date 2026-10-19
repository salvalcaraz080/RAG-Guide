# S11 · Decisiones — Brief 2: salida estructurada, contrato tipado y política de verificación

> Decisiones tomadas al implementar `S11-brief-2.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
>
> **Verificación:** `uv run pytest` **1240 verde** (1217 + 23 netos), `ruff` igual que antes (el
> E402 conocido de `app/main.py`). Medición de referencia **VÁLIDA**
> (`generation_20261018-182613`): 41 de 42 ejecuciones comparables, 0 fallback, 1 error, 0 pares sin
> juicio. Comparación preregistrada **PASA** (`comparison_20261018-182631.md`). Smoke test contra
> la API de Docker: q01 sale `partial`, con frase de cierre y sin campo `refusal`; el contenedor del
> frontend carga el módulo nuevo.

---

## 1. Esquema final y compatibilidad de proveedores

**[AGNÓSTICO] El esquema del §2, sin simplificar.** `Quote`, `Statement`, `Section` y
`GenerationOutput` con los nombres y la semántica del brief. Para que valga en los dos proveedores
con un solo camino: todos los campos obligatorios, opcionales como nulos explícitos (`quote`,
`heading`, `refusal_reason`), sin valores por defecto, sin `minItems` y `extra="forbid"` en todos
los objetos. «Al menos un ref» es un validador Pydantic sobre el objeto parseado: no viaja en el
esquema, porque Anthropic lo descarta y OpenAI en `strict` no lo garantiza. `Literal` (estado) se
admite en los dos. Una sonda real con los dos proveedores validó el esquema tal cual.

**[AGNÓSTICO] El esquema viaja expandido, sin `$ref` — el fallo que costó la primera pasada.**
Pasar la clase Pydantic como `response_format` hace que LiteLLM 1.90.3, en la ruta de Anthropic
(que para Haiku 4.5 es una herramienta forzada, no salida restringida nativa), genere
`"$ref": "/$defs/Section"`: sin `#`, irresoluble. El modelo recibía las secciones, afirmaciones y
citas **sin definir** e improvisaba sus claves: 26 salidas inválidas en la primera pasada
(«Extra inputs are not permitted»). La sonda inicial no lo vio porque el caso era trivial.
`response_format()` construye el formato a mano con todas las definiciones expandidas en el sitio;
un test fija que no queda ningún `$ref` y que todo objeto prohíbe claves extra y exige todos sus
campos. No es un camino por proveedor: el mismo dict va a los dos.

**[AGNÓSTICO] Añadidos al contrato, documentados:** `VerifiedStatement.quote_outside_text` (una cita
declarada que no está en su afirmación se conserva, no se tira — §3), `QueryResponse.schema_attempts`
(1 o 2) y `cost_usd` que suma los intentos descartados (se facturan igual).

## 2. Decisiones de implementación

- **[AGNÓSTICO] H3, reintento y 502.** `_parse` distingue tres casos: texto vacío
  (`EmptyCompletionError`, como antes), JSON que no valida con `finish_reason="length"`
  (`TruncatedCompletion`, 502 sin reintento: repetir chocaría con el mismo tope) y JSON que no valida
  sin truncado (un reintento; a la segunda, `InvalidStructuredOutput`, 502). Ninguno deja turno en la
  sesión. Si la salida cortada **sí** valida, se sirve con `truncated: true`. `llm_output_invalid`
  registra intento, número de errores, primer error **y su ruta en el esquema** (añadida tras la
  segunda pasada para poder diagnosticar sin contenido).
- **[AGNÓSTICO] Orden de validación del §3, con tres concreciones:**
  - Una cita cuyos refs son todos inválidos se verifica contra los refs de su afirmación (y, como
    siempre, contra el resto si no está ahí).
  - **«Cita dentro de la afirmación» se compara ignorando comillas en los dos lados.** La primera
    versión era literal y descartaba citas correctas cuando el texto ponía comillas alrededor de una
    palabra del tramo (`"fix", when…` frente a `fix, when…`); en la segunda pasada eso dejó fuera 79
    citas. Las comillas del texto son presentación, no contenido.
  - Un `not_found` declarado se limpia (espacios, punto final) antes de componerse.
- **[DOMINIO] La tabla de estados, extendida a `out_of_domain`.** La tabla del §3 normaliza
  `out_of_material` con afirmaciones a `partial`, pero no dice qué hacer con un `out_of_domain` que
  trae afirmaciones con fuente. El criterio del §7 exige cero rechazos servidos con citas, así que se
  trata igual: `partial`. Un test lo fija. No ocurrió en la medición (ningún `out_of_*` declarado).
- **[AGNÓSTICO] `refusal_reason` solo se sirve con un estado de rechazo**; si el estado se normaliza
  a `partial`, se descarta (queda el estado declarado en `verification`).
- **[AGNÓSTICO] Afirmación sin fuente:** se sirve con la marca `[no source]` en el lugar de los
  marcadores. Si **todas** quedan sin fuente, el estado servido es `out_of_material` (tabla del §3) y
  las afirmaciones se componen detrás del texto de rechazo: no se borra texto.
- **[AGNÓSTICO] Composición (H8).** Encabezado en negrita si lo hay; afirmaciones como párrafo o
  viñetas; marcadores `[1][3]` antes de la puntuación final de cada afirmación. Frases fijas del
  producto: `Not found in the indexed ECSS standards: X; Y.` y, para los rechazos,
  `Out of scope: this question is not about the ECSS standards this assistant covers.` /
  `I could not find this in the indexed ECSS standards.` Las frases marcador se conservan como texto
  visible —es voz de producto— pero ya no clasifican nada. El historial guarda `compose(...,
  markers=False)`, generado así.
- **[AGNÓSTICO] Prompt (H9).** Fuera: formato de marcadores, frases marcador, redacción de «no
  encontrado» y la regla «respuesta parcial con frase al final» (ahora son estado y `not_found`).
  Dentro: grounding, qué es el bloque, una afirmación = una idea con `refs`, cita literal en el texto
  y repetida en `quote.text`, lectura normativa («nunca cambies un modal, dentro o fuera de una
  cita»), lo no cubierto a `not_found`, voz, y la semántica de los cuatro estados. El recordatorio del
  mensaje de usuario se reescribe en los mismos términos. Snapshot regenerado.
- **[AGNÓSTICO] UI (H11).** Las decisiones salen a `frontend/presentation.py`, puro y con tests (el
  `streamlit_app.py` no es testeable porque ejecuta Streamlit al importarse): aviso «does not cite
  any source» solo para `answered`/`partial` sin citas; `partial` nunca se pinta como rechazo;
  `needs_review` → aviso visible; cita no literal → «≈ Not verbatim: “…” [n]» bajo la respuesta (el
  texto lo compone el servidor y el frontend no lo reescribe, así que el distintivo va junto y no
  dentro); reasignaciones, estado declarado → servido y recuentos, en el panel de diagnóstico.
- **[AGNÓSTICO] Arnés (H12).** Lee la estructura validada; las métricas «frases sin marcador» se
  sustituyen por las del §7.3. Un turno previo de una multi-turn que no se sirve marca todas las
  ejecuciones de esa consulta como error: en la primera pasada un 502 en un turno previo abortaba el
  proceso entero sin artefacto (fallo del instrumento, corregido con test).
- **[AGNÓSTICO] `compare.py`.** Re-puntúa un artefacto en prosa sin modelo ni base (lleva cada
  respuesta y el texto de cada chunk): es el único sitio que todavía lee `"…" [n]` de la prosa. La
  regex vive ahí y no en `app/`. Criterio: ninguna métrica peor en más de 0,05, globalmente **ni**
  como media de los cambios por consulta; cero rechazos servidos con citas. Sale con 4 si falla.

## 3. Comparación preregistrada (H12) — [DOMINIO]

Línea base: `generation_20261018-164057` (prosa) re-puntuada con el validador nuevo. Nuevo:
`generation_20261018-182613` (estructurada). Media [mín–máx] entre índices de ejecución; ruido global
de la referencia ~±0,03.

| métrica | línea base | nuevo | Δ global | Δ emparejado (media) | veredicto |
|---|---|---|---|---|---|
| recall condicionado | 0,65 [0,63–0,68] | **0,72** [0,71–0,72] | +0,07 | +0,07 | PASA |
| precisión de cita | 0,91 [0,90–0,93] | **0,91** [0,89–0,93] | +0,00 | +0,00 | PASA |
| tasa `verified` | 0,95 [0,94–0,97] | **0,98** [0,97–1,00] | +0,03 | −0,00 | PASA |
| rechazos servidos con citas | 4 / 42 | **0 / 41** | | | PASA |

**Por consulta** (media sobre ejecuciones, nuevo − base):

| consulta | estrato | Δ recall | Δ precisión | Δ `verified` |
|---|---|---|---|---|
| q01 | names_standard | +0,22 | −0,17 | +0,00 |
| q02 | names_standard | +0,00 | −0,04 | +0,07 |
| q03 | generic | −0,06 | +0,00 | +0,08 |
| q04 | generic | +0,05 | +0,00 | +0,05 |
| q05 | paraphrase | +0,00 | −0,04 | +0,00 |
| q06 | comparative | −0,11 | +0,00 | n/a |
| q07 | comparative | +0,07 | +0,00 | +0,00 |
| q08 | cross_standard | +0,25 | +0,00 | −0,20 |
| q09 | cross_standard | +0,06 | +0,00 | −0,28 |
| q10 | multi_turn | +0,00 | +0,00 | +0,07 |
| q11 | names_standard_xref | +0,11 | +0,06 | +0,07 |
| q12 | names_standard_xref | +0,08 | +0,21 | +0,00 |
| q13 | paraphrase | +0,11 | +0,00 | +0,09 |
| q14 | multi_turn | +0,17 | +0,00 | +0,00 |

**Lectura:**
- **[DOMINIO] El recall sube por encima del ruido (+0,07, rango que no se solapa).** Con una
  afirmación por idea y `refs` obligatorios, el modelo cita más cláusulas relevantes de las que
  recibe; la precisión no se mueve. La pasada intermedia (`181325`, con la contención literal) dio
  0,66 / 0,90 / 1,00: el recall también se mueve entre pasadas, y la conclusión firme es «no
  empeora», no «+0,07».
- **[DOMINIO] El `verified` no es comparable en población.** Se calcula sobre **112** citas frente a
  **280**: con la salida estructurada el modelo cita literalmente mucho menos (§5). Pasar el umbral
  no dice que la cita literal haya mejorado. Las caídas de q08 y q09 son los dos únicos `not_found`
  de la pasada, sobre muy pocas citas por consulta.
- **[DOMINIO] q01 deja de ser un «rechazo que cita»:** `partial` en las 3 ejecuciones, con la frase
  de cierre de `not_found`; sube su recall (+0,22) y baja su precisión (cita más cláusulas, algunas
  juzgadas no relevantes). q14, que en la referencia salía rechazo en 1 de 3, sale `partial` en las 3.

**Efecto aislado de H7** (la referencia re-puntuada, sin llamadas): de los 16 `not_found`, **5** pasan
a `verified` (q05 ×3 por el espacio en `information /documentation`, q09 por `'…'` frente a `"…"`, q09
por el espacio tras un guion), **1** a `modal_edit` (el «shall» insertado de q04) y **10** siguen
`not_found`. `S11-decisions.md` anticipaba 7: en q02 ×2 la comilla era solo el primer defecto;
corregida, aparece una edición (los ítems de una lista cosidos con `;` donde la fuente pone `.`), que
sigue siendo `not_found` con razón. **Monotonía comprobada**: los 262 `verified` siguen `verified`.

## 4. Estados declarado → servido — [DOMINIO]

| declarado | servido | ejecuciones |
|---|---|---|
| answered | answered | 13 |
| answered | partial | 6 |
| partial | partial | 22 |

Ningún `out_of_*` declarado en las 41. La normalización actuó en 6 casos (`answered` con `not_found`
→ `partial`). En la referencia, 4 de 42 respuestas empezaban con la frase de rechazo y citaban.

## 5. Reasignaciones, `modal_edit`, no literales, sin fuente y citas fuera de su afirmación — [DOMINIO]

- **Reasignaciones: 0. `modal_edit`: 0. Afirmaciones sin fuente: 0. Refs retirados: 0.** La
  política está probada en contrato (tests de cada veredicto y de la precedencia) y en la referencia
  re-puntuada (el `modal_edit` de q04), pero la pasada no produjo ninguno.
- **No literales (`not_found`): 2**, los dos en `cross_standard`:
  - q08 ej. 2: «ECSS-Q-30-02 provides implementation requirements and a step-by-step procedure for
    the implementation of a Failure Modes and Effects Analysis.» [3]
  - q09 ej. 3: «provides confidence to the customer and to the suppliers that developed or reused
    software satisfies the requirements throughout the system lifetime» [5]
- **Citas fuera de su afirmación: 76 de 188 declaradas** (se conservan en `quote_outside_text`, no se
  sirven como literales). Es comportamiento del modelo, no de la comprobación: rellena `quote.text`
  con el tramo exacto de la fuente y escribe en la afirmación una paráfrasis sin comillas. Ejemplos:
  - q03 ej. 3 — texto: «If a software product implements different system functions, the one having
    the highest criticality is considered…»; cita: «…the one **among them** having the highest…».
  - q01 ej. 1 — texto: «Review data package documentation shall be according to the deliveries for
    every review in the software life cycle chosen, with requirements detailed in Table A-1…»; cita:
    «…chosen (see Table A-1: ECSS-E-ST-40 and ECSS-Q-ST-80 Do…».
  - q01 ej. 1 — la cita es una lista de tres ítems con saltos de línea; el texto la resume en una
    frase.

  **[DOMINIO] Consecuencia de producto:** la cita literal, que era el localizador fino («localiza»),
  aparece mucho menos en lo que lee el usuario, aunque el modelo sí haya copiado el tramo correcto.
  Es la salvedad principal de este brief y va a Claude Chat (§9).

## 6. Esquema inválido y reintentos — [DOMINIO]

| pasada | salidas inválidas | reintentos que salvaron | 502 | nota |
|---|---|---|---|---|
| 1 (`$ref` sin `#`) | 26 | — | abortó | ~19 respuestas completas; sin artefacto |
| 2 (`181325`) | 7 | 5 | 1 (q06 ej. 3) | contención literal (79 citas fuera) |
| 3 (`182613`, referencia) | 5 | 3 | 1 (q06 ej. 3) | |

- **[DOMINIO] ~10 % de las salidas no validan** con el esquema correcto; casi todas «Input should be
  a valid array» (un campo lista que llega con otra forma; hipótesis: un array serializado como
  texto, típico de la ruta por herramienta de Anthropic — sin confirmar porque el log no guarda
  contenido; desde ahora registra la ruta del campo).
- **[DOMINIO] q06 (criticidad B vs C) falla dos veces seguidas en las dos pasadas** y se sirve como
  502. Es la consulta `answered` del UAT.

## 7. Ausencias y voz — [DOMINIO]

- **Violaciones de voz: 0** (en la referencia en prosa salían en q01, q09 y q14: «consult the full
  standard», «material available to me»).
- **Ausencias dentro de una afirmación: 1** — q06 ej. 1: «…the applicability to category C is not
  specified in the ECSS standards referenced». La vía `not_found` absorbe el resto: 92 aspectos no
  encontrados en 41 respuestas, compuestos con la frase fija.

## 8. Tokens, latencia y coste — [DOMINIO]

| | prosa (referencia) | estructurada |
|---|---|---|
| Tokens de salida, mediana / máx | 394 / 619 | **978 / 1.682** |
| Latencia del turno juzgado, mediana / p95 | 5,6 s / 7,6 s | **10,3 s / 16,3 s** |

- `MAX_OUTPUT_TOKENS = 2000`: el máximo observado deja 318 tokens de margen. Un JSON cortado es un
  502 de truncado.
- Peor caso observado: ~28 s (q09, con un reintento). El timeout del frontend (120 s) sigue holgado.
- **Coste de la sesión ≈ 1,4 USD**: tres pasadas (0,19 + 0,42 + 0,43 USD en respuestas servidas, más
  ~0,35 USD en salidas inválidas que el log de entonces no costeaba), tres sondas y el smoke test.

## 9. Para Claude Chat

1. **La cita literal se pierde** (§5): componer la afirmación con la cita, servir la cita verificada
   aparte, o reforzar la regla en el prompt. No se ha iterado el prompt.
2. **~10 % de esquema inválido y q06 en 502** (§6): ¿tolerar en el parseo un array serializado como
   texto (determinista, en código) o cambiar de vía de salida estructurada?
3. **Latencia ×1,8 y tokens ×2,5** (§8): sin umbral en el brief.
4. **`out_of_domain` con afirmaciones → `partial`**: extensión de la tabla de estados que tomé para
   cumplir el criterio (§2). Confirmar.

## 10. Qué se eliminó (§6 del brief)

- `QueryResponse.refusal` y `_refusal_kind` (clasificación por prefijo de texto) → `status`.
- `resolve_markers`, `strip_markers`, `validate_quotes`, `contains_marker` y las regex de prosa de
  `citations.py` → refs declarados y `check_quote` por cita. La regex `"…" [n]` sobrevive solo en
  `compare.py`, para re-puntuar artefactos en prosa.
- La segmentación en frases y el detector de frases sin marcador de `statements.py` → una afirmación
  sin refs es inválida por esquema. Se conservan, sobre `Statement.text`, el contador de ausencia y
  el detector nuevo de reglas de voz, solo como métrica y log.
- Del prompt: formato de marcadores, frases marcador y redacción de «no encontrado».

## 11. Recuento de tests

| fichero | antes | después | qué |
|---|---|---|---|
| `test_citations.py` | 32 | 25 | fuera marcadores y `strip_markers`; normalización H7, monotonía sobre las citas del golden set, `check_quote` con sus cuatro veredictos y precedencia, `modal_edit` añadido/quitado/cambiado en la citada y en otra |
| `test_verification.py` | — | 25 | esquema (válido, sin refs, claves extra, sin `$ref`), las cinco filas de estado más `out_of_domain`, refs retirados, sin fuente, reasignación a uno y a varios, no literal, `modal_edit`, cita fuera de su afirmación, comillas en el texto |
| `test_composition.py` | — | 9 | marcadores en orden, listas y encabezados, frase de cierre, rechazo, sin fuente, historial sin marcas |
| `test_llm_service.py` | 26 | 19 | fuera `_refusal_kind`; prompt y citas sobre la estructura |
| `test_queries.py` | 23 | 28 | estado tipado, `refusal` ausente, reintento (y su coste), dos inválidas → 502 sin turno, truncado → 502 sin reintento, `response_format` que llega al wrapper |
| `test_statements.py` | 22 | 9 | ausencia y voz |
| `test_generation_harness.py` | 20 | 27 | estructura validada, turno previo no servido, rechazos sin citas, `compare` (re-puntuado, transiciones, criterio) |
| `test_presentation.py` | — | 4 | `partial` no es rechazo, aviso por estado, ejes de evidencia, no literales |
| **Total** | **1217** | **1240** | |

Además, los mocks de prosa pasan a salida estructurada en `test_multi_turn`, `test_observability`,
`test_input_guards`, `test_query_integration` y `test_prompt_endpoint`; la suite de evaluación
(`real_llm`, fuera por defecto) se adapta al contrato nuevo.
