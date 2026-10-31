# S14 · Decisiones — Brief 1: taxonomía del «sin respuesta normal»

> §1 es el **preregistro**: escrito el 2026-10-24 **antes** de la pasada del arnés, con el código
> ya implementado y la suite en verde. §2 en adelante, después de medir.

---

## 1 · Preregistro

### 1.1 Inventario de degradaciones técnicas (parte A)

Eventos del camino de `/query` que cambian lo que recibe el usuario. «Hoy» es antes de este brief.

| Evento | Dónde | Hoy en el contrato | Decisión |
|---|---|---|---|
| Fallback de proveedor | Router de LiteLLM (`llm_wrapper`) | Sí: `fallback_used`, `model`, `provider` | Se queda ahí (T2) |
| Recorte por presupuesto: cláusulas | `_fit_budget` | Sí: `input_budget.trimmed_clauses` | Se queda ahí (T2) |
| Recorte por presupuesto: historial | `_fit_budget` | Sí: `input_budget.trimmed_history_turns` | Se queda ahí (T2) |
| Reintento por esquema | `_answer` | Sí: `schema_attempts`; el coste incluye el descartado | Se queda ahí (T2) |
| Salida cortada por el tope | proveedor | Sí: `truncated`, `finish_reason` (o 502 si no parsea) | Se queda ahí (T2) |
| Coste no calculable | `llm_wrapper._cost_usd` | Sí: `cost_usd = null` | Se queda ahí; no altera la respuesta |
| Proveedor no resoluble | `llm_wrapper` | Sí: `provider = "unknown"` | Ídem |
| **Moderación caída por red** | `input_guards` (fail-open) | **No** (solo log `moderation_unavailable`) | **`moderation_unavailable`**, `stage=guardrails`, `detail=["api_error"]` |
| Moderación sin clave | `input_guards` | No (log en cada petición) | **No entra**: es configuración del despliegue, constante; avisaría en todas las respuestas. Su sitio es el arranque |
| **Filtro de routing sin resultados** | `context._rank` | **No** (log `routing_filter_empty`) | **`routing_filter_empty`**, `stage=retrieval`, detalle: estándares nombrados |
| **Expansión `bounded` cortada por tope** | `expansion.expand` | **No** (solo el recuento en `expansion_applied`) | **`expansion_truncated`**, detalle: cláusulas |
| **`xref`: puntero sin plaza** (`over_limit`, cláusula o estándar) | `references.fill` / `_follow` | **No** (solo `references_expanded`) | **`xref_over_limit`**, detalle: cláusulas destino o estándares |
| **`xref`: cláusula seguida pero cortada por tope** | `references.fill` / `_follow` | **No** | **`xref_truncated`**, detalle: cláusulas o estándares |
| **Etiquetas de tabla inservibles** (huella distinta) | `retrieval.load_table_labels` | **No** (log `table_labels_unavailable`) | **`table_labels_unavailable`**: ninguna referencia a tabla se puede seguir |
| Expansión «por tiempo» | — | — | **No existe**: ninguna etapa tiene plazo; no hay nada que inventariar |
| Ventana de historial (`MAX_HISTORY_TURNS`) | `SessionStore` | No | **No entra**: es la política configurada, no un fallo |
| Reintentos internos (LiteLLM, SDK de OpenAI) | ver 1.2 | No | **No entran**: no cambian el contenido, solo la latencia. Reporte en 1.2 |

Forma: `QueryResponse.degradations: list[{stage, kind, detail}]`, siempre presente, vacía si nada
se degradó; `kind` es un `Literal` cerrado. Una entrada por tipo y etapa, con todos los afectados
en `detail`. `RetrievedContext` transporta las de retrieval (la costura no cambia de forma: un campo
con default) y `run_input_guards` devuelve las suyas.

**Aviso de frecuencia, antes de medir:** con `bounded` (600 tokens por cláusula), es probable que
`expansion_truncated` aparezca en buena parte de las respuestas. Por eso la UI la pinta colapsada
(«⚙️ Technical limits on this answer (n)»), nunca como aviso.

### 1.2 Capas de reintento (reporte, sin cambios)

Leído del código instalado (LiteLLM 1.106.0, `openai` 2.41.0); no medido.

| Capa | Dónde | Predicado | Cuántos | ¿Se ve en el contrato? |
|---|---|---|---|---|
| Router de LiteLLM | `llm_wrapper._router` | Por tipo (`RetryPolicy`): timeout 2, rate limit 2, error 5xx 1, autenticación 0; el resto, `num_retries` del Router (default = `openai.DEFAULT_MAX_RETRIES` = 2). `should_retry_this_error`: no reintenta 4xx no reintentables ni `NotFound`, ni un rate limit sin despliegue sano si hay fallback | 0–2 por despliegue | **No.** Solo latencia; `query_stages` lo ve como un único intento `llm` |
| Fallback | Router, `fallbacks` | Fallo del primario tras sus reintentos | 1 despliegue (con sus propios reintentos) | Sí: `fallback_used` |
| SDK bajo el Router | `default_litellm_params.max_retries = 0` | — | 0 | — (el Router lo anula) |
| Reintento por esquema | `llm_service._answer` | La salida no valida y no está cortada | 1 | Sí: `schema_attempts` |
| SDK de OpenAI, embeddings | `embedding._default_client` (`AsyncOpenAI`, default) | Conexión, 408, 409, 429, 5xx | 2 | **No.** Tras ellos, 503 (fail-closed) |
| SDK de OpenAI, moderación | `input_guards._moderation_client` (default) | Ídem | 2 | Tras ellos, ahora sí: `moderation_unavailable` |
| Base de datos | `create_async_engine(pool_pre_ping=True)` | Conexión del pool muerta | Reconecta 1 vez al sacar del pool | No |

**Qué enmascaran:** las capas del Router y de los SDK convierten un fallo transitorio en latencia,
sin rastro en el contrato. **Peor caso del camino del modelo:** (1 + 2) intentos del primario + (1 +
2) del fallback = 6 llamadas, ×2 por el reintento de esquema = **12 llamadas** a proveedor en una
petición. Ninguna capa enmascara una degradación de contenido: el fallback y el reintento por esquema
ya llegan al contrato. Decide el chat.

### 1.3 Comprobación previa sin generación (§2.4)

`scripts/gap_kinds_precheck.py` (salida en `scripts/output_gap_kinds_precheck.txt`): el proveedor de
producción (`bounded` + `clause`) sobre la query de retrieval de cada turno, como la compone
`llm_service`. 3 embeddings, ninguna llamada al modelo.

| Caso | Turno | Evidencia (cita que el hueco tipado tendría que llevar) | ¿Inyectada? |
|---|---|---|---|
| q14 turno 1 → `premise_contradicted` | 1 | «all services are optional» | **Sí**, [7] ECSS-E-ST-70-41C § 1, `#1::s1` (normativo, sección, **traído por expansión**) |
| q14-a1 → `delegated` | 3 | «refer to ECSS-E-ST-70-01» | **Sí**, [3] ECSS-E-ST-70-41C § 6.18.1.1, `#6.18.1.1::s0`; también en las 3 ejecuciones de la referencia |
| q12-a3 → `delegated` | 1 | «Risk management for software shall be performed by cross-reference to the project risk policy, as specified in ECSS-M-ST-80» | **Sí**, [1] ECSS-Q-ST-80C § 5.3.1; también en las 3 ejecuciones de la referencia |

- **Los tres casos sobreviven.** Con tres casos, el criterio 1 pide 2 de 3.
- **Matiz sobre q12-a3:** 5.3.1 delega la gestión de riesgos *entera* a M-ST-80; no nombra los
  criterios de aceptación (q12-a3 en el diagnóstico: «M-ST-80C no da criterios: exige que la política
  de riesgos los describa»). La misma cláusula delega también q12-a4 (contenido de la política). Se
  mantiene en el objetivo porque la evidencia está inyectada y sí es una remisión; un `delegated` a
  5.3.1 en q12 puede corresponder a cualquiera de los dos aspectos, y el informe lista el texto del
  aspecto para que el chat lo juzgue.
- q14 turno 2 no tiene caso objetivo.
- **No se ajusta nada** a la vista de esto.

### 1.4 Instrumento de la pasada

- **Configuraciones:** `control` es `xref+bounded` (la configuración de la referencia, con la
  opción **apagada** de forma explícita); `gaps` es `xref+bounded` con la opción encendida. El
  nombre `control` ya lo usa el arnés para la configuración de S12 (`orphans`, sin `xref`), así que
  el control de este brief se pasa como `--config xref+bounded --config gaps`. El `compare.py` de
  siempre lo enfrenta a la referencia con `--baseline-config xref+bounded`.
- **q14 turno 1 se mide 3 veces.** El arnés solo respondía una vez los turnos previos (para fabricar
  el historial). `REPEATED_TURNS = {"q14": (1,)}` lo ejecuta además 3 veces por configuración sobre
  el historial anterior a él (vacío, al ser el primero). Son 3 llamadas más por configuración; no se
  puntúan contra el golden set.
- **Regla de acierto por caso** (en `corpus/groundtruth/generation/gaps.py`, fijada ahora): en una
  ejecución, el caso se cumple si se **sirve** un hueco del `kind` esperado con cita `verified` o
  `misattributed` (reasignada) **y sus refs servidas incluyen la cláusula de evidencia** de 1.3. El
  caso tiene éxito con ≥ 2 de 3 ejecuciones.
- **Por qué no, por ejecución** (el primero que aplique): `error` (no servida) ·
  `evidence_not_injected` · `downgraded_quote_not_verified` (declarado con el tipo esperado y servido
  como `not_in_material`; se lista también el motivo concreto) · `other_evidence` (tipo esperado,
  otra cláusula) · `declared_kind_wrong` (declarado el otro tipo) · `declared_not_in_material` ·
  `no_gap`. Los cuatro del brief más tres que el brief no cubre: `error`, `other_evidence` y `no_gap`.
- **Huecos tipados fuera del objetivo:** todo hueco tipado servido que no sea el que cumple un caso,
  con su consulta, turno, ejecución, aspecto, cita y cláusula. Para revisión del chat (criterio 2).
- **Criterios 3 y 4** salen del informe del arnés y de `compare.py` (pareado `control` frente a
  `gaps`, y `control` frente a la referencia para la condición de ruido de §3.1).
- **Coste previsto:** 49 llamadas por configuración (42 juzgadas, 4 de turnos previos, 3 repetidas)
  más reintentos; ≈ 0,27 USD. Antes, **una llamada de humo** con la opción encendida sobre una
  pregunta **fuera del golden set**, solo para comprobar que el proveedor acepta el esquema
  expandido (≈ 0,003 USD).

### 1.5 Decisiones de implementación que el chat debe ver

1. **`gap_kind_downgraded` va en `verification`, no en `degradations`.** `degradations` es técnico
   por definición (T1); una cita que no prueba el tipo es un juicio de contenido. Mezclarlos
   reintroduce el antipatrón que el brief corrige. Lo cuenta `gaps.py` por ejecución.
2. **Un rechazo servido no lleva hueco tipado.** Si el estado normalizado es `out_of_material` (el
   modelo declaró solo huecos, sin afirmaciones), los huecos tipados se sirven como
   `not_in_material` con motivo `refusal`: así se mantiene «ningún rechazo sale con citas» sin tocar
   la normalización (T7). Marcado `TODO: pending decision in Claude Chat`.
3. **Las refs de un hueco tipado entran en `citations`.** Su `[n]` aparece en el texto, y todo `[n]`
   del texto está en `citations` (la UI numera así la evidencia). Consecuencia para la medición: un
   hueco tipado puede añadir una cláusula citada y mover recall y precisión.
4. **Las citas de los huecos no cuentan en los veredictos de `verification`** (`verified`,
   `modal_edit`…): el criterio 3 es sobre afirmaciones. El veredicto de cada hueco va en el propio
   hueco (`gaps[].verdict`).
5. **Contrato:** `not_found: list[str]` sigue igual (todos los aspectos, de cualquier tipo: T7) y
   `gaps: list[{aspect, kind, declared_kind, refs, declared_refs, quote, verdict}]` es aditivo; con
   la opción apagada, cada aspecto sale `not_in_material` con `declared_kind = null`.
6. **Ref fuera de rango en un hueco tipado → degradado** (`invalid_ref`), sin buscar la cita en
   otras cláusulas (el §4 del brief lo pide así). Una ref válida con la cita en otra cláusula sí se
   reasigna (`misattributed`).
7. **`not_in_material` declarado con refs o cita:** se sirven nulos, sin registrarlo.
8. **Prompt con `on`:** la regla va al final de las instrucciones estáticas (`gap_kinds.md`); el
   mensaje de usuario y el recordatorio no cambian. Prefijo cacheado distinto del de `off`.
9. **Composición:** un bloque por tipo tras las afirmaciones, en este orden: «❗ The indexed ECSS
   standards contradict what the question assumes:», «↪️ The indexed ECSS standards leave this to
   another document:» y el de siempre, «⚠️ I could not find…». Cada viñeta: `aspecto — “cita” [n]`.
   Con `off`, el texto es el de antes byte a byte.

---

## 2 · Resultados

Pasada única: `generation_20261024-140241` (`xref+bounded` = control, `gaps`), 14 × 3 + q14 turno
1 × 3 por configuración. 98 llamadas, **0,2645 USD**, 0 fallback, 0 errores, 0 esquemas inválidos,
0 recortes, 0 rechazos con citas. Informes: `gaps_20261024-140316.md` (huecos),
`comparison_20261024-140304.md` (control frente a la referencia),
`comparison_20261024-140310.md` (`gaps` frente a control) y el `aspects.py` de siempre (métricas
absolutas). Las 63 parejas sin juicio son las mismas de la referencia `xref+bounded` (ya las tenía:
«INVALID — 63 unjudged pairs»); no es un fallo del instrumento. Antes, 1 llamada de humo (0,002
USD): el proveedor acepta el esquema tipado.

### 2.1 Control frente a la referencia (§3.1)

| | referencia `generation_20261019-224307` | control hoy | Δ |
|---|---|---|---|
| recall condicionado | 0,861 | 0,835 | **−0,026** |
| precisión | 0,876 | 0,892 | +0,016 |
| `verified` | 1,00 | 1,00 | 0 |

**Dentro del ruido (±0,03), así que se compara.** Una observación aparte: las citas fuera de su
afirmación suben de 25/309 a 43/331 en el control, sin la opción; `compare.py` lo marca FAIL frente a
la referencia. No lo causa este brief (la configuración es la de la referencia).

### 2.2 Tabla del criterio (§3.2), sin aplicarlo

| # | Criterio | Medido | Lectura literal |
|---|---|---|---|
| 1 | Cada caso objetivo con su `kind` y cita `verified` en ≥ 2 de 3; con tres casos, ≥ 2 casos | q14 turno 1 **2/3** · q14-a1 **2/3** · q12-a3 **3/3** → **3 de 3 casos** (control: 0/3, 0/3, 0/3). **Pero ver 2.3, q12-a3** | cumple; con la lectura estricta de q12-a3, 2 de 3 casos: cumple igual |
| 2 | Ningún `premise_contradicted` servido fuera del objetivo que el chat juzgue espurio | **0** fuera del objetivo (los 2 servidos son de q14 turno 1). `delegated` fuera del objetivo: **10**, listados en 2.4 | cumple; los `delegated`, a revisar |
| 3 | `verified` = 1,00 sobre las afirmaciones | 1,00 en las dos | cumple |
| 4 | Recall y precisión no bajan > 0,03; relevantes citadas en absoluto no bajan | recall condicionado +0,008 · recall sobre relevantes recuperadas 0,85 → 0,85 · **precisión −0,028** (0,892 → 0,864) · relevantes citadas 237 → 238 | cumple, con la precisión a 0,002 del umbral |

Sin ser guarda:

| | control | `gaps` |
|---|---|---|
| `gap_kind_downgraded` | — | **0** |
| huecos declarados | 66 (sin tipo) | 40 `not_in_material`, 2 `premise_contradicted`, 15 `delegated` |
| huecos por ejecución (juzgadas) | 1,48 | 1,21 |
| estados servidos (juzgadas, `aspects.py`) | answered 10, partial 32 | answered 13, partial 29 |
| latencia mediana / p95 | 16,1 / 27,4 s | **18,3 / 28,0 s** (+2,2 s, +14 %; `llm` 15,0 → 17,2 s) |
| tokens de salida (mediana) | 3.866 | 4.282 (+11 %) |
| tokens de entrada (mediana) / leídos de caché | 10.478 / 2.651 | 11.288 / 3.401 (prefijo mayor, cacheado) |
| coste por ejecución | 0,00268 USD | 0,00299 USD (+12 %) |
| citas fuera de su afirmación | 43 / 331 | 6 / 323 |
| precisión sin cláusulas solo de expansión | 0,93 | 0,92 |

La bajada de precisión se concentra en q05, q13 y q14 (−0,14 a −0,17 por consulta, pareado); el
resto se compensa. Con N = 3, no se puede atribuir a la opción por consulta.

### 2.3 Casos uno a uno

**q14 turno 1 → `premise_contradicted`: 2/3.**
- Ejecuciones 2 y 3: `services the on-board software is obliged to / must support — “all services
  are optional” [7]` (E-ST-70-41C § 1). Es exactamente el caso de S13-uat-2.
- Ejecución 1, por qué no: `declared_not_in_material` («which specific services must be supported
  and which are optional»).
- La ejecución 3 añade además un `delegated` dudoso (ver 2.4).

**q14-a1 (turno 3) → `delegated`: 2/3.**
- Ejecuciones 1 y 3: `delegated` con «compliant with and complements the spacecraft on-board control
  procedures standard (refer to ECSS-E-ST-70-01)» [3] (§ 6.18.1.1). Aspectos: «detailed rules for
  executing on-board control procedures» y «requirements the flight software must meet to execute
  on-board control procedures».
- Ejecución 2, por qué no: `declared_not_in_material` («flight software capabilities needed to
  execute on-board control procedures»).

**q12-a3 → `delegated`: 3/3 por la regla preregistrada, con un matiz que el chat debe juzgar.**
- Las 3 ejecuciones sirven `delegated` con la cita completa de 5.3.1 («Risk management for software
  shall be performed by cross-reference to the project risk policy, as specified in ECSS-M-ST-80.»)
  [1], `verified`.
- **El aspecto no es el de q12-a3.** En las tres es «how software risks feed into / are integrated
  into the project's overall risk management process» — la segunda mitad de la pregunta —, no «los
  criterios de aceptación de riesgos». La regla del preregistro casa por cláusula de evidencia, no
  por aspecto, y ya avisaba de que 5.3.1 delega varios aspectos (1.3).
- Con la lectura estricta por aspecto, q12-a3 es 0/3 (por qué no: el modelo no declara ese aspecto;
  ningún hueco nombra los criterios de aceptación). El criterio 1 se cumpliría igual con 2 de 3
  casos. Lo que sí se ve: la delegación que el modelo tipa es la que la cláusula hace.

### 2.4 Huecos tipados fuera del objetivo (para revisión del chat)

Todos `delegated`, todos con cita `verified`. Ningún `premise_contradicted`.

| Consulta · ejecuciones | Aspecto | Cita | Cláusula | Lectura de CC |
|---|---|---|---|---|
| q04 · 1, 2 | criterios para decidir cuándo hace falta ISVV y a qué nivel | «The ISVV Level is depending on two factors (ref ISVV guide annex E for detail):» | E-HB-40A § 5.6.2.2.2 | remisión real (a la guía de ISVV) |
| q04 · 1, 2, 3 | propósito, alcance y actividades de ISVV en detalle | «Guidance on ISVV purpose, scope, and activities is provided in CSW-ESAISVV-2022-GBK-02897.» | Q-ST-80C § 6.2.6.13 | remisión real |
| q08 · 1, 2, 3 | procedimiento paso a paso de la FMEA de software | «[ECSS-Q-30-02] provides implementation requirements and a step-by-step procedure for … FMEA» | Q-HB-80-03A § 6.2.2.1 | remisión real |
| q13 · 2 | procedimientos detallados de mantenimiento en vuelo | «More information on software maintenance in flight is given in the SAVOIR document» | E-HB-40A § 5.10.2.2.1 | remisión real |
| q14 turno 1 · 3 | qué servicios debe soportar cada proceso de aplicación | «Each mission usually documents the results of this design and selection process in a "Space-to-Ground Interface Control Document".» | E-ST-70-41C § 1 | **dudosa**: describe una práctica de la misión, no que otro documento lo regule |

Sin ningún `downgrade`: ninguna cita de hueco tipado falló la verificación en 15 + 2.

### 2.5 Parte A: degradaciones en la pasada

En las ejecuciones juzgadas (las dos configuraciones, mismo contexto): `expansion_truncated` 36,
`xref_truncated` 36, `xref_over_limit` 18. **13 de 14 consultas** llevan alguna; solo q02 sale
limpia. Lo que el preregistro anticipaba: la degradación por tope es lo normal con `bounded` +
`clause`, y por eso se pinta plegada. Ninguna de moderación, routing vacío ni etiquetas.

### 2.6 UAT automatizada (fuera del preregistro)

`S14-uat.md`, por HTTP con la opción encendida, 2 ejecuciones de q14 y q12: q14 turno 1, 2 de 2;
**q14-a1, 0 de 2**; 5.3.1 en q12, 1 de 2. Acumulado con la pasada: q14 turno 1 **5 de 6** (incluye la
respuesta que fabrica el historial), q14-a1 **2 de 5**, 5.3.1 en q12 **4 de 5**. No cambia la tabla
de 2.2, que es la del preregistro, pero matiza su lectura: q14-a1 pasa el umbral por poco.
Además, q02 salió `partial` con un `delegated` real, cuando en la pasada fue `answered` sin huecos.

### 2.7 Recuento de tests

**1533 verde** (1477 + 56): `test_degradations.py` (11), `test_gap_kinds.py` (27),
`test_generation_gaps.py` (12), 4 del arnés y 2 de presentación. La suite no llama a ningún
proveedor.

---

## 3 · Etiquetado agnóstico / dominio

- **Agnóstico:** error ≠ revisión en el contrato (degradaciones técnicas en un campo propio, con un
  vocabulario cerrado, frente a juicios de contenido en la verificación); «el modelo declara un tipo,
  el código lo prueba o lo degrada»; una categoría en el esquema en vez de un párrafo de reglas cuando
  el tipo solo se conoce tras leer el contexto; inventario de capas de reintento (qué enmascara cada
  una); preregistro con una regla de acierto que case por evidencia **y por aspecto** (lección de
  q12-a3).
- **Dominio:** la premisa contradicha y la delegación son tipos naturales en un corpus normativo
  (opcionalidad en E-ST-70-41C, remisiones entre estándares y a guías externas); la degradación por
  tope es casi universal con `bounded` + `clause` en este corpus.
