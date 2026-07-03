# S01 — LLMs + setup

## Estructura comparada de las tres APIs principales

| Aspecto | OpenAI (Responses API) | Anthropic (Messages API) | Gemini (google-genai) |
|---|---|---|---|
| System prompt | `instructions` (param. de primer nivel) | `system` (param. de primer nivel) | `system_instruction` dentro de `config` |
| Entrada usuario | `input` (string o array de mensajes) | `messages` (array, roles `user`/`assistant`) | `contents` (string o array de `Content`) |
| Rol de respuesta del modelo | `assistant` | `assistant` | `model` |
| Config de generación | kwargs sueltos | kwargs sueltos | objeto anidado `GenerateContentConfig` |
| `max_tokens` | opcional | **obligatorio** | opcional (modelo decide si se omite) |
| Texto de respuesta | `response.output_text` | `response.content[0].text` (array de bloques) | `response.text` |
| Historial multi-turno | manual, o `previous_response_id` (server-side, requiere `store=True`) | siempre manual (sin alternativa server-side) | manual, o helper `chats.create()` (sigue reenviando todo internamente) |
| Retry automático del SDK | no | sí, 2x por defecto con backoff exponencial | no |
| Conteo de tokens nativo | no (usar `tiktoken` localmente) | sí, `messages.count_tokens()`, gratuito y exacto | sí, `models.count_tokens()`, gratuito |
| Señal de truncado | `status == "incomplete"` + `incomplete_details` | `stop_reason == "max_tokens"` | `finish_reason == "MAX_TOKENS"` |
| Bloqueo por seguridad | — | — | `finish_reason == "SAFETY"` / `"RECITATION"` (200 OK, no excepción) |
| Alternancia de roles | flexible | estricta user↔assistant | flexible |

`tiktoken` **no** es una aproximación válida para Anthropic (infraestima ~15-20%+); usar `count_tokens()` nativo de cada proveedor cuando exista.

## Parámetros en modelos de razonamiento
Los modelos de razonamiento (o-series/GPT-5.x, Claude con extended thinking) bloquean parámetros de muestreo tradicionales y los sustituyen por control de esfuerzo:

- OpenAI: se pierden `temperature`, `top_p`, `frequency_penalty`, `presence_penalty`, `logprobs`, `logit_bias` → se ganan `reasoning.effort` y `text.verbosity`.
- Anthropic con `thinking` activado: se pierden `temperature` y `top_k` → se gana `thinking.budget_tokens`. Además, en Claude 4.5+ no se pueden combinar `temperature` y `top_p` simultáneamente **incluso sin thinking activado**.
- Los tokens de razonamiento (`reasoning_tokens` / `thinking_tokens`) se facturan como tokens de salida aunque no aparezcan en la respuesta visible.
- Esta restricción es por **modelo concreto**, no solo por proveedor — el wrapper necesita saber qué parámetros acepta el modelo activo, no solo qué proveedor es.

## Tokenización
- Los LLMs no procesan texto, procesan secuencias de IDs numéricos (BPE — Byte Pair Encoding es el estándar de facto).
- El español consume ~20-40% más tokens que el inglés para el mismo contenido semántico; japonés/chino pueden duplicar el conteo.
- Los tokens de salida cuestan 3-10x más que los de entrada → optimizar longitud de respuesta pesa más en factura que optimizar el prompt.
- El código se tokeniza de forma relativamente eficiente (keywords frecuentes tienen tokens dedicados).
- Espacios, saltos de línea e indentación consumen tokens — JSON compacto es más barato que JSON con pretty-print.
- Los números se tokenizan de forma inconsistente ("100" puede ser un token, "101" dos) — los LLMs no ven dígitos individuales alineados, de ahí su debilidad en aritmética.
- El coste de una conversación crece de forma acumulativa: cada turno reenvía todo el historial (incluido el system prompt) salvo que se use prompt caching, truncado de historial, o resumen de turnos anteriores.
- Prompt caching: hasta 90% de descuento en cache hits (Anthropic y OpenAI).

## Mercado y pricing (mecánica estable, independiente de precios/modelos concretos)
- Asimetría input/output universal: el output siempre es más caro.
- Prompt caching disponible en todos los proveedores grandes — especialmente relevante en CAG/RAG donde el contexto base se repite en cada llamada.
- Batch API (~50% descuento, async, <24h) — apto para preprocesamiento masivo sin urgencia de tiempo real (p. ej. embeddings o resúmenes de corpus completo, offline).
- Estrategia multi-modelo (barato para routing/clasificación, mid-tier para el grueso del trabajo, premium para casos difíciles) puede reducir coste 60-80% frente a usar siempre el modelo premium.
- Agregadores/routers existen como alternativa a construir el wrapper desde cero:
  - **OpenRouter** — SaaS, acceso a 500+ modelos con una sola API/billing, ~5.5% de markup. Orientado a prototipado.
  - **LiteLLM** — librería open source self-hosted, sin markup, unifica 100+ proveedores en formato compatible OpenAI, soporta fallback automático y tracking de coste por equipo. Orientado a producción.

## Decisiones para RAG-ECSS
Ninguna tomada en esta sesión — sesión teórica/comparativa, sin fricción de implementación todavía.

## Fricciones identificadas para S03 (abstracción de proveedor)
1. Nombres de rol distintos (`assistant` vs. `model`).
2. Shape de configuración distinto (kwargs planos vs. objeto anidado).
3. Política de default para `max_tokens` (el wrapper necesita una política propia, no puede delegar en el comportamiento de cada proveedor).
4. Semántica de "respuesta bloqueada/incompleta" no uniforme entre proveedores.
5. Política de retry no uniforme — cuidado con reintentos anidados si no se desactiva el retry nativo del SDK antes de implementar uno propio.
6. Parámetros válidos dependientes del modelo concreto (razonamiento vs. no-razonamiento), no solo del proveedor.
7. Conteo de tokens: usar el endpoint nativo de cada proveedor, no `tiktoken` como aproximación universal.
8. Bifurcación de diseño: construir el wrapper a mano vs. adoptar LiteLLM.
9. Manual vs. server-side state para historial de conversación — relevante para ECSS por trazabilidad.

## Candidatas anotadas para sesiones específicas
- **S11 (citación):** extraer IDs de cláusula ECSS del metadato del chunk recuperado, nunca generarlos como texto libre del LLM.
- **S02 (coste):** modelar la asimetría idiomática (corpus ECSS en inglés, consultas probablemente en español) como factor de coste.
- **M4 (inyección de contexto RAG):** preferir formato compacto (JSON sin indentación) al inyectar metadatos estructurados como contexto.

## Próximo paso
M2 CAG — S02: fundamentos CAG, API, tokens/contexto/parámetros/tools, conversación, claves/rate-limit/errores, costes.
