# S02 — Primeros pasos de arquitectura CAG

## CAG: qué es y cuándo usarla

**CAG (Cache-Augmented Generation)**: precarga todo el conocimiento relevante en la ventana de contexto del modelo en cada llamada, en lugar de recuperarlo en tiempo real (RAG). Elimina el paso de retrieval: sin base de datos vectorial, sin embeddings, sin pipeline de indexación.

| | RAG | CAG |
|---|---|---|
| Cómo alimenta conocimiento | Búsqueda + recuperación en tiempo real | Todo precargado en el prompt |
| Infraestructura | Vector DB, embeddings, pipeline de indexación | Ninguna |
| Latencia | Incluye tiempo de retrieval | Mínima (sin búsqueda) |
| Escala de datos | Ilimitada (millones de documentos) | Limitada al tamaño de la ventana de contexto |
| Riesgo principal | Selección de documentos incorrecta/incompleta | Corpus no cabe en contexto |

**Condición de aplicabilidad de CAG:**
- Base de conocimiento acotada — cabe en la ventana de contexto (orientativo: ~200-250 páginas para 128K tokens).
- Datos relativamente estáticos (no cambian cada hora/minuto).
- Prioridad en latencia mínima y simplicidad arquitectónica sobre escala.

**Cuándo NO usar CAG:** corpus grande y creciente, datos que exigen precisión de selección (mezclar contenido irrelevante genera "context distraction"), o volumen de queries alto donde el coste por token sea crítico (matizable con prompt caching — ver más abajo).

**CAG y RAG no son excluyentes** — se entienden mejor como fases de madurez de un mismo sistema: CAG para validar rápido, RAG cuando el corpus crece o la precisión de selección se vuelve crítica.

## Componentes de una arquitectura CAG

1. **Fuente de conocimiento** — el dataset que el modelo necesita (en el proyecto vehículo del máster: presupuestos históricos).
2. **Capa de preprocesamiento** — selección de campos relevantes, normalización de formatos, cálculo de campos derivados, anonimización. Transforma datos crudos en contexto listo para inyectar.
3. **Constructor de prompts** — ensambla system prompt + contexto + mensaje de usuario.
4. **Servicio de llamada al LLM** — gestión de claves, errores, reintentos, rate limits, extracción de la respuesta.
5. **Postprocesamiento** — parseo a JSON, validación de restricciones, extracción de datos estructurados, verificación de coherencia (p. ej. detectar que "10 horas" con coste de "50.000€" es incoherente).

## Estructura del proyecto (FastAPI)

Por qué FastAPI: modelo async (ASGI) — mientras una llamada al LLM está en curso (2-30s típico), el event loop libera el hilo para atender otras peticiones. Frameworks síncronos con modelo de threads bloquean un worker completo por cada llamada en curso.

**Nota de implementación:** usar cliente async del SDK del proveedor (o `asyncio.to_thread` si no existe variante async) — envolver una llamada síncrona en `async def` no la hace no-bloqueante; el event loop se congela igual durante la llamada.

Estructura por capas:

```
app/
├── main.py          ← punto de entrada, registra routers
├── config.py         ← Pydantic BaseSettings, falla al arrancar si falta config
├── routers/          ← capa de transporte HTTP, "delgada": recibe, delega, devuelve
├── services/          ← lógica de negocio: construcción de prompt, llamada LLM, postproceso
├── schemas/            ← contratos Pydantic (request/response)
└── context/             ← datos de referencia para CAG (punto de sustitución hacia RAG en M3-M4)
```

Principio: los routers no conocen la lógica de prompts ni el proveedor LLM; los servicios no conocen HTTP. `context/` es estático hoy (diccionario en código); se sustituye por un servicio de búsqueda semántica en RAG sin tocar el resto del sistema — siempre que el servicio de negocio no dependa del origen de los datos, solo del formato.

Versionado de API (`/api/v1`) desde el principio: permite evolucionar el contrato sin romper a consumidores existentes.

## Gestión de contexto

**Presupuesto de tokens** — la ventana de contexto reparte: system prompt + contexto inyectado + mensaje de usuario + espacio para la respuesta. Todo debe caber, y "caber" no es suficiente: la atención del modelo se degrada con contexto más largo ("lost in the middle" — más atención a principio y final, menos en el medio).

**Qué incluir:**
- Ejemplos completos con desglose (no resúmenes de una línea) — el modelo replica la granularidad que ve.
- Patrones de precios/dedicación reales de la empresa, para no dejar que el modelo invente cifras de mercado genérico.
- Un ejemplo del formato de output esperado (más efectivo que describirlo en texto).

**Qué evitar:** detalle que no aporta al patrón (historial de comunicaciones, campos administrativos), información contradictoria (épocas/precios muy distintos entre ejemplos), redundancia entre ejemplos muy similares (mejor diversidad de tipos de proyecto que repetición del mismo).

**Formato:** texto plano estructurado (más eficiente en tokens, bueno para narrativa) vs. JSON (bueno para jerarquía, más caro en tokens) vs. Markdown (punto intermedio, formato por defecto del proyecto). Regla práctica: el formato de los ejemplos debe parecerse al formato de output esperado — el modelo tiende a replicar patrones que ve en el contexto. Usar delimitadores explícitos entre ejemplos múltiples para evitar que el modelo mezcle información entre ellos.

**Orden por atención posicional:**
```
1. System prompt + formato esperado     ← principio (máxima atención)
2. Ejemplos de referencia (por relevancia)
3. Restricciones/reglas específicas     ← cerca del final
4. Mensaje del usuario (consulta real)  ← final (máxima atención)
```

**System prompt efectivo — cuatro dimensiones:** rol y expertise concreto (no "eres un asistente"), tarea concreta, cómo debe usar el contexto de referencia (¿son ejemplos de formato? ¿de calibración de precios?), formato exacto del output esperado.

**Cuántos ejemplos:** 2-3 mínimo viable; 5-7 punto dulce (diversidad de tipos sin diluir atención); >10 rendimientos decrecientes — señal de que toca migrar a RAG.

## Arquitectura de conversaciones

Los LLMs son **stateless**: cada llamada envía la conversación completa como array de mensajes (`system`/`user`/`assistant`). El modelo no recuerda nada entre llamadas — la ilusión de memoria conversacional vive en el cliente, que reenvía todo el historial cada vez.

- **`system`** — comportamiento global, se envía en cada llamada (coste multiplicado por número de turnos si es largo).
- **`user`** — entradas del humano.
- **`assistant`** — respuestas previas del modelo; el desarrollador es responsable de guardarlas e incluirlas en llamadas siguientes, el proveedor no lo hace automáticamente.

**Single-turn vs. multi-turn:** decisión de producto, no técnica. Single-turn (transaccional: una transcripción → una estimación) es más simple, sin estado. Multi-turn (conversacional: refinar iterativamente) da mejor UX pero acumula tokens con cada turno.

**Gestión de historial cuando crece** (tres estrategias, de simple a sofisticada):
1. **Ventana deslizante** — conservar solo los últimos N turnos + system prompt. Simple, pero pierde información temprana relevante.
2. **Resumen acumulativo** — comprimir turnos antiguos en un resumen generado por el propio LLM. Conserva lo esencial, cuesta una operación extra.
3. **Híbrida con anclas** — turnos marcados como clave (decisiones, aprobaciones) nunca se descartan; el resto sigue ventana deslizante. Mejor resultado en producción, requiere criterio para marcar anclas.

Para la fase actual, ventana deslizante es suficiente.

## Diferencias entre proveedores (relevantes para el wrapper de S03)

| Aspecto | OpenAI | Anthropic |
|---|---|---|
| System prompt | Mensaje más dentro del array | Parámetro separado (`system=`), fuera del array |
| Rol de resultados de tools | Rol `tool` explícito | Bloques `tool_result` dentro de un mensaje `user` (no hay rol `tool` separado) |
| Campo de respuesta | `response.choices[0].message.content` | `response.content[0].text` |
| Alternancia de roles | Algunos modelos exigen alternancia estricta user/assistant | — |

`max_tokens` (o equivalente) debe fijarse explícitamente, no depender del default del proveedor.

## Prompt caching (verificado independientemente, no cubierto en el material de S02)

Los cuatro artículos de S02 tratan el coste de CAG como proporcional al volumen de tokens enviados en cada llamada, sin mencionar el prompt caching que ofrecen los proveedores — mecanismo directamente relevante porque el patrón de CAG (prefijo estático grande + parte variable pequeña) es el caso de uso ideal para caching.

- **Anthropic**: marcado explícito vía `cache_control`. Escritura a 1.25x (TTL 5 min) o 2x (TTL 1h) del precio normal de input; lectura a 0.10x (descuento del 90%). Mínimo 1024 tokens cacheables.
- **OpenAI**: caching automático, sin marcado explícito, descuento del 50%. Mismo mínimo de 1024 tokens.

El orden de contexto recomendado por posición atencional (system + ejemplos + restricciones antes del mensaje de usuario) coincide con la estructura óptima para un cache breakpoint: todo lo estático antes de la parte variable. Diseñar pensando en ambos criterios a la vez (atención posicional + frontera de prefijo cacheable) es más eficiente que tratarlos como preocupaciones separadas.

## Decisiones tomadas en S02

- Ninguna decisión de arquitectura CAG/RAG para RAG-ECSS se ha tomado en esta sesión: el contenido de S02 desarrolla el proyecto vehículo del máster (estimador de presupuestos), que cumple la condición de aplicabilidad de CAG (corpus acotado y estable). RAG-ECSS no cumple esa condición (corpus grande, jerárquico, con requisito de trazabilidad por cita) — CAG no aplicará directamente a ECSS, pero el eje de decisión ("¿corpus acotado y estable?") es candidato a nodo de la Guide.
- Estructura por capas (routers/services/schemas/context) adoptada como patrón base, pendiente de validar con fricción real de implementación.

## Abierto para sesiones futuras

- **S03 (wrapper de proveedor):** el wrapper debe normalizar las diferencias de proveedor listadas arriba, exponer el control de prompt caching como parte de la abstracción (no como detalle oculto de un proveedor), y usar cliente async nativo (o `to_thread`) para no romper el modelo de concurrencia de FastAPI.
- **S03:** la interacción entre estrategia de truncamiento de historial y prompt caching — truncar en cada turno invalida el cache de prefijo; resumen acumulativo o anclas son más compatibles con caching por mantener el prefijo estable entre turnos.
- **S12 (function calling):** confirmar cómo se estructura `tool_result` en Anthropic vs. rol `tool` en OpenAI al implementar tools.
- **Diseño de interacción para RAG-ECSS:** decidir transaccional vs. conversacional — probablemente pesa más lo conversacional que en el estimador de ejemplo, dado que un ingeniero consultando estándares plausiblemente encadena preguntas relacionadas.
