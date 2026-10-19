# S11 — Brief 3: modelo generador, presupuesto de entrada, caché de prefijo y cita visible

> Sobre `S11-decisions-2.md` (referencia estructurada `generation_20261018-182613`) y lo decidido en
> Claude Chat tras leerlo. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar. Rama
> `s11-generacion-calidad`.
> **La UAT del brief 2 se pospone y se une a la de este brief** (excepción al orden habitual: con
> Haiku 4.5 el primer paso de esa UAT, q06, sale 502). `S11-uat-3.md` cubre los dos.

---

## 0. Objetivo y alcance

**Objetivo.** Cuatro cambios, cada uno medido o probado por separado dentro de la misma ejecución:
1. Generador **`claude-haiku-5-5`** y fallback **`gpt-6-luna`** (más capaces y más baratos que la
   configuración actual, según el usuario; verificar en la documentación de cada proveedor).
2. **Presupuesto de entrada** con tope duro: Haiku 5.5 multiplica el precio por encima de 100k tokens
   de entrada (tokenizador de Anthropic).
3. **Caché del prefijo estático**, solo si compensa (§4).
4. **La cita verificada se muestra** aunque el modelo la parafrasee en la afirmación
   (`S11-decisions-2.md` §5 y §9.1).

**Fuera de alcance:** tolerancia del parseo a arrays serializados como texto (§9.2 de decisions-2:
se decide con los datos del modelo nuevo), iterar el prompt, juez LLM, golden set, `bounded`,
`corpus/`. Si algo lo exige: `# TODO: pending decision in Claude Chat`.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| M1 | Cambio de modelos por configuración, sin tocar el seam del wrapper | El modelo es una pieza intercambiable; si cambiarlo exige código, el seam está mal |
| M2 | **Comprobación previa**: que LiteLLM (versión fijada) conoce los dos modelos — precio (`cost_usd`), salida estructurada **nativa o por herramienta**, límites de contexto y de salida. Si no, actualizar LiteLLM y registrar qué cambió | Con la versión actual, `cost_usd` podría salir vacío o mal, y la vía de salida estructurada decide la tasa de esquema inválido (el fallo de `$ref` vino de la ruta por herramienta forzada de Haiku 4.5) |
| M3 | Presupuesto de entrada `MAX_INPUT_TOKENS` (default **30.000**), aplicado **antes** de llamar al modelo sobre una estimación conservadora (caracteres / 3 o similar, documentada) | Hoy la mediana es ~4.000 tokens; el riesgo está en los extremos (tablas sin partir, multi-turn largo, una expansión futura). El margen hasta 100k es tan grande que no hace falta el endpoint de conteo exacto (latencia extra) |
| M4 | Si no cabe: se descartan **las cláusulas de menor rango primero** (el orden `[n]`), nunca a mitad de cláusula, y se registran (`context_budget_trimmed`, ids descartados). Si aun sin contexto no cabe (historial), se recorta el historial más antiguo y se registra | Etapa "presupuesto" del artículo 1: toda etapa que tira información registra qué tiró |
| M5 | Conteo exacto **a posteriori**: `usage.prompt_tokens` del proveedor se registra siempre; alerta (`prompt_tokens_high`, nivel warning) por encima de **50.000** | Calibra la estimación de M3 con el tokenizador real, gratis |
| M6 | Caché de prefijo **condicionada**: se activa solo si el prefijo estático (instrucciones + esquema) supera el **mínimo cacheable** de Haiku 5.5. Si no lo supera, no se activa y se registra el porqué | Por debajo del mínimo es un no-op. Además, en Anthropic escribir caché cuesta más que la entrada normal y el TTL es corto: con tráfico de un usuario espaciado podría encarecer. Se decide con números (§4) |
| M7 | Orden del prompt: **estático primero** (instrucciones, esquema), luego historial, luego contexto recuperado y pregunta | Solo se cachea el prefijo idéntico. Verificar el orden actual; corregir si no lo cumple |
| M8 | Cita verificada fuera de su afirmación → **se compone visible** junto a la afirmación, con su marcador (forma exacta a criterio de CC: p. ej. la afirmación y, a continuación, `— "cita" [n]`, o una línea de cita debajo) | 76 de 188 citas verificadas no llegaban al usuario porque el modelo parafrasea en `text`. Determinista, sin pedir nada al modelo. Solo `verified`: una cita `not_found` o `modal_edit` no se presenta como literal |
| M9 | `ARCHITECTURE.md` (§ modo evaluación): corregir que el prompt caching "suprime varianza" | La caché de prefijo ahorra cómputo de entrada; no cambia la distribución de la salida. No interfiere con la evaluación ni requiere `LLM_EVAL_MODE` |

---

## 2. Medición del cambio de modelo

1. **Sonda del fallback** antes de la pasada: unas pocas llamadas forzando `gpt-6-luna` con el
   esquema real (≈5 llamadas, autorizadas). El esquema tiene que validar en los dos sin camino por
   proveedor, como en el brief 2.
2. **Pasada del arnés**: 14 consultas × 3 ejecuciones con `claude-haiku-5-5` (≈46 llamadas,
   autorizadas; con caché activa si M6 se cumple — la caché no cambia la salida, así que una sola
   pasada mide calidad y coste). Si la primera pasada revela un fallo del instrumento, una segunda
   pasada completa también está autorizada.
3. **Criterio preregistrado** contra `generation_20261018-182613`, global y pareado (mismo de §7 del
   brief 2): recall condicionado, precisión y tasa `verified` no empeoran más de 0,05; cero
   rechazos servidos con citas.
4. **Además se reporta** (sin umbral, decide Claude Chat): tasa de esquema inválido y reintentos
   (referencia: 5 inválidas, 1 × 502 en q06), tokens de entrada y salida, latencia mediana/p95, coste
   por consulta, citas declaradas y cuántas quedan fuera de su afirmación.
5. Si algo empeora más allá del umbral: no iterar el prompt; reportar con casos.

---

## 3. Presupuesto de entrada (M3–M5)

- `MAX_INPUT_TOKENS` y el umbral de alerta como configuración con nombre y su porqué al lado
  (sin entornos nuevos más allá de lo necesario).
- Reportar en decisions: estimación frente a `prompt_tokens` real sobre las 42 ejecuciones (error de
  la estimación, máximo real observado) — para saber cuánto margen deja la heurística.
- El arnés registra `prompt_tokens` y si hubo recorte; una ejecución con recorte no se compara sin
  marcarla.

---

## 4. Caché de prefijo (M6–M7)

- Medir en tokens del proveedor el prefijo estático (instrucciones + esquema) y compararlo con el
  mínimo cacheable documentado para Haiku 5.5.
- Si se activa: el coste con y sin caché se **calcula** sobre la misma pasada a partir de los tokens
  de lectura/escritura de caché que devuelve el proveedor (sin pasada extra). Reportar para dos
  patrones: el del arnés (llamadas seguidas) y el de un usuario con consultas espaciadas más allá del
  TTL (todo escritura). Si en el segundo patrón la caché encarece, decirlo: la decisión de dejarla
  activa en producción la toma Claude Chat con esos números.
- OpenAI cachea de forma automática (sin coste de escritura): solo reportar si aparece en el uso del
  fallback.

---

## 5. Tests de contrato (sin red, sin BD, sin modelo real)

- Presupuesto: contexto que no cabe → se descartan las de menor rango, nunca a mitad de cláusula,
  y se registran; historial recortado cuando el contexto no basta; `prompt_tokens` alto → warning.
- Orden del prompt: el prefijo estático es idéntico entre dos peticiones distintas (precondición de
  la caché).
- Caché: con prefijo por debajo del mínimo no se pide; por encima, sí (parámetros que llegan al
  wrapper).
- Composición M8: cita verificada fuera del texto → visible con marcador; cita dentro → sin
  duplicar; `not_found`/`modal_edit` → no se presentan como literales.
- Configuración: los nombres de modelo llegan al Router; un test fija el fallback.
- Recuento final frente a 1240.

---

## 6. UAT conjunta (brief 2 + brief 3)

`S11-uat-3.md` sustituye a `S11-uat-2.md`. Mínimo: q06 (criticidad B vs C) `answered` sin 502; q01
`partial` con frase de cierre; fuera de dominio; multi-turn; una respuesta con cita verificada
compuesta visible (M8); rótulos de evidencia en dos ejes (brief 2); si aparece, un `needs_review` o
una reasignación. Sidebar: modelo nuevo, tokens de entrada, coste por consulta.

---

## 7. Cierre

`S11-decisions-3.md` con: comprobación de LiteLLM (M2) y si hubo actualización; vía de salida
estructurada de cada modelo; tabla del §2 (criterio + métricas reportadas, global y por consulta);
presupuesto (estimación vs real, máximo, recortes); caché (tamaño del prefijo, mínimo, si se activó,
coste en los dos patrones); M8 (cuántas citas pasan a ser visibles); recuento de tests; coste real de
la sesión. `S11-uat-3.md`. `CHANGELOG.md`, `ARCHITECTURE.md`, `.claude/CLAUDE.md`.
