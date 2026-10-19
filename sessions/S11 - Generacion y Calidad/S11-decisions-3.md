# S11 · Decisiones — Brief 3: modelo generador, presupuesto de entrada, caché de prefijo y cita visible

> Decisiones tomadas al implementar `S11-brief-3.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
>
> **Verificación:** `uv run pytest` **1263 verde** (1240 + 23), `ruff` igual que antes (el E402
> conocido). Pasada del arnés **VÁLIDA** (`generation_20261018-212343`): 42/42 comparables, 0
> fallback, 0 errores, 0 esquemas inválidos, 0 recortes, 0 pares sin juicio. **El criterio
> preregistrado NO pasa: la precisión de cita baja 0,07** (umbral 0,05); recall y `verified` pasan
> (§3). API de Docker reconstruida con LiteLLM 1.106.0: q06 responde sin 502.

---

## 1. Comprobación de LiteLLM (M2) y vía de salida estructurada

| | LiteLLM 1.90.3 (fijada) | LiteLLM 1.106.0 (adoptada) |
|---|---|---|
| Precio de `claude-haiku-5-5` y `gpt-6-luna` | Sí, **solo porque descarga el mapa de modelos de GitHub (`main`) al arrancar**; el mapa empaquetado de versiones de hace dos semanas no trae Haiku 5.5 | Sí, en el mapa empaquetado (incluida la tarifa >100k de Haiku 5.5) |
| Vía de Haiku 5.5 | **Herramienta forzada** (una lista fija de modelos «nativos» que no incluye Haiku); Haiku 5.5 la acepta pero se salta el pensamiento | **Nativa** (`output_config.format`, decodificación restringida), decidida por la capacidad `supports_native_structured_output` del mapa |
| Vía de `gpt-6-luna` | `json_schema` strict | `json_schema` strict |

- **[AGNÓSTICO] Datos verificados en la documentación de cada proveedor.** `claude-haiku-5-5`:
  $0,10 / $0,50 por MTok hasta 100k tokens de prompt ($0,50 / $2,50 por encima), caché: lectura
  0,1× ($0,01), escritura 5 min 1,25× ($0,125), TTL por defecto 5 min, mínimo cacheable 512 tokens,
  1M de contexto, 128K de salida, salida estructurada nativa, **pensamiento adaptativo activo por
  defecto con esfuerzo `medium`**, parámetros de muestreo no por defecto → 400, tokenizador ~30 %
  más denso que Haiku 4.5. `gpt-6-luna`: $0,10 / $0,50 (caché leída $0,01, escrita $0,125),
  922k de entrada, 128K de salida, más caro por encima de 272k, salida estructurada soportada,
  razonamiento con esfuerzo `medium` por defecto. LiteLLM tiene los mismos precios.
- **[AGNÓSTICO] Actualización a 1.106.0, decidida por el usuario** tras exponerle el riesgo: es una
  versión publicada el mismo día (LiteLLM tuvo versiones comprometidas en PyPI) y cambia a la vez la
  vía y el pensamiento. Inspeccioné los wheels de 1.106.0, 1.104.0 y 1.100.5 sin ejecutarlos antes
  de proponerla. El lock añade como dependencias **boto3, botocore, s3transfer, jmespath, h2,
  hpack, hyperframe, python-dateutil y six**, y sube aiohttp y yarl. La suite pasa sin cambios.
- **[AGNÓSTICO] Coste del snapshot fechado.** `gpt-6-luna-2026-09-22` no está en el mapa de OpenAI y
  `get_llm_provider` no lo resuelve: con el código anterior el fallback habría salido con proveedor
  `unknown` y `cost_usd: null`. El wrapper toma ahora el proveedor del despliegue que respondió
  (`_hidden_params["custom_llm_provider"]`) y el coste de `litellm.completion_cost` sobre la
  respuesta, que resuelve el nombre fechado y aplica las tarifas de caché y de prompt largo. En la
  sonda OpenAI respondió como `gpt-6-luna`, sin fecha.
- **[AGNÓSTICO] `MAX_OUTPUT_TOKENS`: 2.000 → 16.000.** La sonda cortó q06 en los dos modelos
  (`finish_reason=length`, JSON roto): el razonamiento cuenta dentro de la salida. Con 16.000, q06
  usó 3.549 (Haiku) y 2.673 (Luna) y validó. Solo se factura lo generado. Lo cambié como límite de
  salida que exige el modelo nuevo (M2); el esfuerzo de razonamiento queda por defecto (§8).
- **[AGNÓSTICO] Sonda del fallback (§2.1):** 4 llamadas forzando `gpt-6-luna` y 3 al primario
  (unos 0,01 USD). El esquema valida en los dos con el mismo formato; la marca de caché del system
  no rompe la llamada a OpenAI, cuya caché automática leyó 1.292 y 4.081 tokens.

## 2. Implementación

- **[AGNÓSTICO] M1, por configuración.** `LLM_MODEL=claude-haiku-5-5`,
  `LLM_FALLBACK_MODEL=gpt-6-luna` (en `config.py`, `.env` y `.env.example`); el Router los toma de
  ahí. Los mocks de los tests que fingen al primario se actualizan: `_fallback_used` compara con el
  nombre configurado.
- **[AGNÓSTICO] M7, orden del prompt corregido.** Antes el contexto recuperado iba **dentro del
  system**, detrás de las instrucciones, y el historial después: el prefijo no era idéntico entre
  peticiones. Ahora `system` = instrucciones estáticas solas (byte a byte iguales siempre); el
  historial va detrás; el último mensaje de usuario lleva el bloque de fuentes, la pregunta y el
  recordatorio. `build_system_prompt` se sustituye por `build_prompt(chunks, question) → (system,
  user)`. Un test fija que el system de dos peticiones distintas es idéntico.
- **[AGNÓSTICO] M3–M4, presupuesto (`services/budget.py`, puro).** `MAX_INPUT_TOKENS=30.000`
  estimados como **caracteres / 2,5**: lo fijo (instrucciones, esquema, pregunta, recordatorio) más
  el bloque y el historial. Si no cabe, salen las cláusulas de mayor `[n]` enteras (los refs de las
  que quedan no cambian), luego los pares de historial más antiguos; se registra
  `context_budget_trimmed` con los `chunk_id` y las cláusulas descartadas, y la respuesta lleva
  `input_budget`. Una pregunta que sola no cabe se sirve y se registra `context_budget_exceeded`
  (`# TODO: pending decision in Claude Chat` — rechazarla, y con qué estado, no está decidido).
  **Por qué ÷2,5 y no ÷3:** la sonda midió Haiku 5.5 a ~2,9 caracteres por token en q06 (6.331
  estimados con ÷3 frente a 6.521 reales); ÷3 subestimaba.
- **[AGNÓSTICO] M5.** `llm_call_completed` lleva siempre `tokens_in` (del proveedor) y
  `estimated_input_tokens`; `prompt_tokens_high` (warning) por encima de `PROMPT_TOKENS_ALERT=50.000`.
  Las dos son configuración con nombre y su porqué al lado.
- **[AGNÓSTICO] M6, caché condicionada.** `_CACHE_STATIC_PREFIX` se decide al cargar: el prefijo
  (instrucciones + esquema) se estima **a la baja** (÷4: aquí el error caro es pedir caché para un
  prefijo que el proveedor no cachea) y se compara con 512. Estimado: 1.584; **real, 2.493 tokens**
  del proveedor (`cache_creation_input_tokens`). El wrapper marca el system con `cache_control` solo
  si se le pide (`cache_system`); un test fija los dos lados del mínimo y que el parámetro llega.
- **[AGNÓSTICO] M8, la cita verificada se ve.** Una cita que no está en el texto de su afirmación
  se verifica igual; si es literal (`verified`, o `misattributed` ya reasignada a donde está) se
  compone detrás de la afirmación con su propio marcador: `Afirmación [2] — “tramo exacto” [1].`
  Una `not_found` o `modal_edit` fuera del texto no se presenta (ni se marca como no literal en la
  UI, porque no aparece). Los recuentos por veredicto siguen contando solo las citas dentro de su
  afirmación — comparables con la referencia — y las de fuera llevan `quotes_outside_statement` y
  `quotes_shown_apart`. `VerifiedQuote` gana `in_text`; `quote_outside_text` desaparece.
- **[AGNÓSTICO] M9.** ARCHITECTURE decía que el prompt caching «suprime varianza»: es falso. La
  caché de prefijo ahorra cómputo de entrada y no cambia la distribución de la salida; no interfiere
  con la evaluación ni pide `LLM_EVAL_MODE`. Corregido.
- **[AGNÓSTICO] Contrato y UI.** `usage` gana `cache_write_tokens` y `cache_read_tokens`;
  `QueryResponse.input_budget`. El panel lateral muestra modelo, tokens de entrada, caché, coste y
  presupuesto. El arnés registra tokens reales y estimados, caché, recorte (una ejecución recortada
  se marca), citas declaradas, fuera y mostradas aparte, y coste por turno; `compare.py` acepta
  también una referencia estructurada.

## 3. Comparación preregistrada — [DOMINIO]

Referencia `generation_20261018-182613` (Haiku 4.5) frente a `generation_20261018-212343` (Haiku 5.5).

| métrica | referencia | nuevo | Δ global | Δ emparejado | veredicto |
|---|---|---|---|---|---|
| recall condicionado | 0,72 [0,71–0,72] | **0,91** [0,89–0,93] | +0,20 | +0,20 | PASA |
| precisión de cita | 0,91 [0,89–0,93] | **0,85** [0,84–0,86] | **−0,07** | **−0,07** | **NO PASA** |
| tasa `verified` | 0,98 [0,97–1,00] | **1,00** [1,00–1,00] | +0,02 | +0,03 | PASA |
| rechazos servidos con citas | 0 | **0** | | | PASA |

| consulta | estrato | Δ recall | Δ precisión | Δ `verified` |
|---|---|---|---|---|
| q01 | names_standard | −0,11 | +0,17 | +0,00 |
| q02 | names_standard | +0,00 | +0,04 | +0,00 |
| q03 | generic | +0,11 | +0,00 | +0,00 |
| q04 | generic | +0,14 | +0,00 | +0,00 |
| q05 | paraphrase | +0,19 | −0,00 | +0,00 |
| q06 | comparative | +0,33 | −0,17 | n/a |
| q07 | comparative | +0,67 | +0,00 | +0,00 |
| q08 | cross_standard | +0,21 | +0,00 | n/a |
| q09 | cross_standard | +0,22 | +0,00 | +0,33 |
| q10 | multi_turn | +0,29 | +0,00 | +0,00 |
| q11 | names_standard_xref | +0,22 | +0,03 | +0,00 |
| q12 | names_standard_xref | +0,00 | **−0,41** | +0,00 |
| q13 | paraphrase | +0,33 | −0,14 | +0,00 |
| q14 | multi_turn | +0,17 | **−0,50** | +0,00 |

**Los casos de la caída de precisión** (cláusulas citadas que el golden set juzga no relevantes, en
las 3 ejecuciones):
- **q12** (riesgos software en Q-ST-80C): Haiku 5.5 cita **9–10 de las 10** cláusulas inyectadas;
  5–6 son no relevantes y son siempre las mismas — Q-ST-80C §6.2.2.3, §5.3.2.1, §5.3.2.2, §6.2.10.2,
  §6.2.4.8 (y §5.7.3.1). Haiku 4.5 citaba 4–6, con 0–2 no relevantes.
- **q14** (servicios PUS y OBCP): cita 4 cláusulas, entre ellas ECSS-E-ST-70-41C **§1** (alcance) y
  §5.4.3.1; Haiku 4.5 citaba 1–2, todas relevantes.
- **q06**: añade Q-ST-80C §D.1 en 2 de 3 ejecuciones. **q13**: E-ST-80C §5.4.2 o E-ST-10-02C
  §5.2.4.5.

**Lectura:** el modelo nuevo cita mucho más de lo que recibe (recall +0,20, por encima de cualquier
ruido medido) a costa de citar también cláusulas de contexto general o tangenciales. La precisión
global (0,85) sigue alta, pero la caída supera el umbral. **No he iterado el prompt** (§2.5 del
brief); la decisión es de Claude Chat. Una precisión más baja con un recall mucho más alto también
puede leerse contra el golden set: relevante ≠ imprescindible, y la alcance (§1) o una cláusula de
proceso general pueden ser útiles; eso es una revisión de juicios, no una conclusión de este brief.

## 4. Métricas reportadas — [DOMINIO]

| | Haiku 4.5 (`182613`) | Haiku 5.5 (`212343`) |
|---|---|---|
| Esquema inválido / reintentos / 502 | 5 / 3 / 1 (q06) | **0 / 0 / 0** |
| Citas declaradas / fuera de su afirmación | 188 / 76 (no mostradas) | 249 / 46 (**las 46 mostradas aparte**) |
| Citas literales que ve el usuario | 112 | **249** (203 dentro + 46 aparte) |
| Tokens de salida, mediana / máx | 978 / 1.682 | **2.998 / 16.000** |
| Tokens de entrada (proveedor), mediana / máx | — | 6.466 / 8.909 |
| Latencia del turno juzgado, mediana / p95 | 10,3 s / 16,3 s | **13,7 s / 19,4 s** |
| Coste por turno juzgado (mediana) | 0,0103 USD | **0,0019 USD** |
| Ausencias / violaciones de voz | 1 / 0 | 0 / 0 |
| Reasignaciones, `modal_edit`, no literales, sin fuente | 0 / 0 / 2 / 0 | 0 / 0 / 0 / 0 |

- **[DOMINIO] Estados declarado → servido:** answered → answered 3, answered → partial 5,
  partial → partial 34. Ningún `out_of_*` declarado.
- **[DOMINIO] Razonamiento desbocado en q08 ej. 3:** 16.000 tokens de salida (el tope), 44 s,
  `finish_reason=length`; el JSON final aún validó y se sirvió marcado como truncado. Es la única
  ejecución por encima de 8.000. Con el tope anterior habría sido un 502; con este, una respuesta
  lenta y truncada. El esfuerzo de razonamiento es la palanca (§8).
- **[DOMINIO] El coste por turno baja ~5×** aunque se generen ~3× más tokens: el precio por token
  de Haiku 5.5 es la décima parte del de Haiku 4.5. Coste de la pasada: 0,091 USD.

## 5. Presupuesto de entrada — [DOMINIO]

- **Estimación (÷2,5) frente a `prompt_tokens` real, 42 ejecuciones:** cociente mínimo **1,00**,
  mediana 1,14, máximo 1,26. La estimación nunca quedó por debajo del real, pero en el peor caso
  coincidió: el margen es justo y ÷3 habría subestimado.
- **Máximo real observado: 8.909 tokens** de entrada, a un 30 % del presupuesto (30.000) y a un 9 %
  del umbral de precio (100k). **Recortes: 0.**

## 6. Caché de prefijo — [DOMINIO]

- **Prefijo estático: 2.493 tokens del proveedor** (instrucciones + esquema) frente a un mínimo de
  512: la caché **se activó**. En la pasada las 42 ejecuciones **leyeron** el prefijo (0 escrituras:
  la sonda previa lo dejó caliente y las llamadas seguidas lo mantuvieron dentro del TTL de 5 min).
- **Coste calculado sobre la misma pasada** (tarifas: entrada $0,10, lectura $0,01, escritura
  $0,125 por MTok):

| patrón | coste de las 42 | por llamada frente a sin caché |
|---|---|---|
| Sin caché | 0,0930 USD | — |
| Arnés: llamadas seguidas, todo lectura | **0,0836 USD** | **−10,1 %** (−0,00022 USD) |
| Usuario espaciado más allá del TTL: todo escritura | **0,0956 USD** | **+2,8 %** (+0,00006 USD) |

  **Con tráfico espaciado, la caché encarece un 2,8 %.** Dejarla activa en producción lo decide
  Claude Chat con estos números. OpenAI cachea solo, sin coste de escritura: en la sonda del
  fallback leyó de caché sin pedirlo.

## 7. M8 — citas que pasan a ser visibles — [DOMINIO]

**46 citas** verificadas que el modelo dejó fuera del texto de su afirmación se muestran ahora junto
a ella; las 46 eran literales (ninguna `not_found` o `modal_edit` fuera del texto en la pasada). Con
Haiku 4.5 eran 76 de 188 y no se mostraban. Ejemplo de composición:
`…requirements [2] — “The customer shall specify the software safety” [1].`

## 8. Para Claude Chat

1. **Precisión −0,07, por debajo del umbral** (§3): el modelo cita casi todo lo que recibe. Casos
   arriba. Opciones a valorar: una regla en el prompt sobre citar solo lo que sostiene cada
   afirmación, revisar los juicios del golden set para esas cláusulas, o aceptar el cambio por el
   recall.
2. **Esfuerzo de razonamiento**: por defecto `medium` en los dos modelos. Sube latencia (+33 %) y
   tokens, y produjo una ejecución de 16.000 tokens / 44 s. Bajar el esfuerzo es una decisión de
   producto que no he tomado; tampoco he ejecutado la comparación.
3. **El mapa de modelos de LiteLLM se descarga de GitHub (`main`) en cada arranque**, y en 1.106.0
   decide la **vía** de salida estructurada además del precio. Un cambio remoto puede cambiar el
   comportamiento sin desplegar nada. Con 1.106.0 el mapa empaquetado ya tiene los dos modelos, así
   que fijarlo (`LITELLM_LOCAL_MODEL_COST_MAP=True`) es posible. No lo he cambiado.
4. **Caché en producción con tráfico espaciado** (+2,8 %, §6).
5. **Pregunta que sola excede el presupuesto** (`# TODO` en `_fit_budget`).
6. **Tolerancia del parseo a arrays como texto** (§9.2 de decisions-2): con la vía nativa, **0
   salidas inválidas** en 42 + 4 llamadas; los datos del modelo nuevo no la piden.

## 9. Recuento de tests y coste de la sesión

| fichero | antes | después | qué |
|---|---|---|---|
| `test_budget.py` (nuevo) | — | 6 | estimación, sin recorte, cláusulas de menor rango primero y enteras, refs estables, historial más antiguo, no cabe |
| `test_input_budget_and_cache.py` (nuevo) | — | 10 | recorte de punta a punta con log de ids, sin recorte, `prompt_tokens_high`, mínimo de caché a los dos lados, `cache_system` que llega al wrapper (×2), marca solo en el system, tokens de caché de los dos proveedores, modelos por defecto, nombres que llegan al Router |
| `test_composition.py` | 9 | 13 | M8: cita aparte con marcador, sin duplicar, no literal no se presenta, historial sin marcas |
| `test_verification.py` | 25 | 26 | cita fuera del texto: verificada y mostrada / no literal y no mostrada |
| `test_system_prompt_snapshot.py` | 1 | 2 | system y mensaje de usuario |
| `test_presentation.py` | 4 | 5 | cita no literal fuera del texto no se marca |
| **Total** | **1240** | **1263** | |

Además: los tests de prompt pasan a `build_prompt` (system estático idéntico entre peticiones; el
usuario lleva fuentes, pregunta y recordatorio), los mocks de `_call_model` llevan coste y tokens de
caché, y los tests de coste pasan al wrapper (incluido el snapshot fechado del fallback).

**Coste real de la sesión ≈ 0,10 USD:** pasada 0,091 USD, sondas (7 llamadas) ~0,010 USD, smoke
test 0,002 USD.
