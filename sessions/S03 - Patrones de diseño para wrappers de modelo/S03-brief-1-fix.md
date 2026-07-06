# S03 · Brief 1-fix — Correcciones tras la implementación del brief 1

> Instrucciones para Claude Code. Mini-brief de corrección, no de funcionalidad nueva. Cierra la
> deuda técnica registrada en `S03-decisions-1-abstraccion-fallback.md` y convierte un hallazgo n=1
> en requisito de diseño. Se cierra **antes** del brief 2 (cache).
> Al terminar: actualizar `S03-decisions.md` (mismo archivo, sección de correcciones), derivar
> `CHANGELOG.md`, editar `CLAUDE.md`/`ARCHITECTURE.md` si aplica. El commit lo hace el usuario.

## Contexto

El brief 1 quedó implementado y validado (Router + fallback Anthropic→OpenAI, async-correcto,
`LLMResult` normalizado, fallback validado end-to-end). La implementación dejó tres cosas a corregir
antes de construir cache encima. Ninguna es bloqueante del funcionamiento actual; todas son deuda que
se agrava si se arrastra al resto de la sesión.

---

## Fix 1 — Derivación de proveedor sin heurística de substring

**Problema.** `_provider_from_model` en `app/services/llm_wrapper.py` deriva el proveedor por el
prefijo del nombre del modelo (`"claude"` → anthropic, `"gpt"` → openai). Necesario porque tras la
normalización `response.model` pierde el prefijo de proveedor (`anthropic/`/`openai/`) y porque tras
un fallback el modelo pedido y el respondido difieren. Pero la heurística por substring es frágil:
con un tercer proveedor (Gemini, Mistral, etc.) se convierte en un `if/elif` que hay que ampliar a
mano y que rompe en cuanto un nombre de modelo no encaje en el patrón esperado.

**Corrección.** Usar el resolvedor nativo de LiteLLM en vez de la heurística:
`litellm.get_llm_provider(model)` devuelve el proveedor a partir del string del modelo. 

**Verificar antes de dar por buena la corrección** (esto es el punto delicado): confirmar que
`get_llm_provider` resuelve correctamente sobre el `response.model` **normalizado** — es decir, sobre
el identificador SIN el prefijo `anthropic/`/`openai/` (`"claude-haiku-4-5-20251001"`,
`"gpt-5.4-mini-2026-03-17"`), que es exactamente el punto donde perdíamos el prefijo. Si
`get_llm_provider` necesita el prefijo para resolver y `response.model` no lo trae, la vía correcta es
derivar el proveedor del **deployment que respondió** (el `model_list` sí conserva el string físico
con prefijo), no del `response.model` pelado. CC decide cuál de las dos aplica según lo que la
verificación muestre, y lo documenta en el decisions.

**Objetivo de la corrección:** eliminar la fragilidad de raíz, no parchearla — que añadir un tercer
proveedor al `model_list` no obligue a tocar la lógica de derivación de proveedor.

---

## Fix 2 — Test de integración del path real

**Problema.** El test 6 (regresión de grounding/citación) se apoya en el mock de `_call_model`. Eso
confirma que el contrato interno no se rompió, pero **no ejercita el wrapper real** en el path
`POST /api/v1/query` → `llm_service` → `llm_wrapper` → Router. La cobertura de ese camino es por
contrato, no por integración.

**Corrección.** Añadir un test de integración que ejercite el path completo con el mock puesto a
**nivel de `router.acompletion`** (no de `_call_model`), sin red ni keys reales. Reutilizar el patrón
ya descubierto en el brief 1: `mock_response` en los `litellm_params` de los deployments del
`model_list` (monkeypatched), que permite ejercitar el Router de producción tal cual está cableado
—mismos `fallbacks`/`retry_policy`— sin llamada de red.

El test debe verificar que una petición a `/api/v1/query` con una pregunta ECSS devuelve respuesta
con citas construidas desde `reference.py` y con `model`/`provider` reflejando el deployment que
respondió. El test 6 actual se mantiene (cubre el contrato); este lo complementa cubriendo el path.

**No** hace falta un test de integración con red real: la validación end-to-end con keys reales ya se
hizo manualmente en el brief 1 y quedó registrada. Este test es para la suite automática sin red.

---

## Fix 3 — System prompt agnóstico de proveedor (requisito de diseño, no solo fix)

**Contexto.** El brief 1 observó que el system prompt de citación —redactado pensando en Anthropic—
funcionó también con OpenAI (el fallback siguió la instrucción de citar). Eso fue un hallazgo n=1
(una sola query), no una garantía. Se eleva a **requisito de diseño**: el system prompt no debe
asumir el proveedor primario, porque cualquier petición puede acabar servida por el fallback.

**Precisión técnica (para no redactar contra un fantasma).** LiteLLM no impone un formato de system
prompt propio: normaliza al formato OpenAI (`{"role": "system", ...}`) y traduce al de cada proveedor
por debajo. "Agnóstico de proveedor" NO significa "escrito para LiteLLM" — significa que el
**contenido** del prompt no dependa de construcciones afinadas para un proveedor concreto (ni fraseos
específicos de Claude, ni supuestos sobre cómo un proveedor maneja el system, ni nada que un modelo
de otro proveedor pudiera interpretar distinto).

**Corrección.**
1. Revisar `build_system_prompt` en `app/services/llm_service.py` y neutralizar cualquier
   construcción específica de proveedor en el system prompt (rol, grounding, citación, formato). El
   orden del prompt (estático antes que variable, preparado para el cache breakpoint del brief 2) NO
   cambia — esto es sobre el contenido, no sobre la estructura.
2. **Añadir validación como test de regresión** (esto es lo que convierte n=1 en requisito): un test
   que verifique que el **mismo** system prompt produce respuesta con citación correcta tanto por el
   primario (Anthropic) como por el fallback (OpenAI). Con `mock_response` que simule la salida de
   cada proveedor citando correctamente, o —si se quiere señal real— como test de integración marcado
   que golpee ambos proveedores con key real (fuera de la suite sin-red por defecto). CC elige; la
   suite automática por defecto va sin red.

**Límite honesto a dejar escrito en el decisions:** "agnóstico" es un requisito **comprobado contra
los dos proveedores que tenemos**, no garantizado universalmente. LiteLLM uniformiza el envío, pero
cada modelo reacciona distinto al mismo prompt. Que Anthropic y OpenAI lo respeten no asegura que un
tercer proveedor lo haga; con un tercero, se re-valida. El test protege contra regresiones en los dos
proveedores actuales, no contra lo desconocido.

---

## Cierre (CC)

Correcciones → `uv run pytest` (los 167 previos siguen verdes + los nuevos de Fix 2 y Fix 3) →
test de usuario si aplica (no imprescindible aquí: no hay cambio de contrato HTTP, solo robustez
interna y cobertura) → actualizar `S03-decisions.md` con una sección de correcciones (etiquetando
agnóstico/dominio), derivar `CHANGELOG.md`, tocar `CLAUDE.md`/`ARCHITECTURE.md` solo si algo del
estado actual cambió. El commit lo hace el usuario.
