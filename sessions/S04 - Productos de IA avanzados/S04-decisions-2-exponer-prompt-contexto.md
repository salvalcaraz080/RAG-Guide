# S04 — Decisions log, brief 2/2: exponer prompt activo + contexto CAG

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier proyecto
> LLM) / `[dominio]` (específico de RAG-ECSS). Cierra el anclaje de S03 brief 5 (los dos
> `# TODO` del sidebar de Streamlit) y S04 queda cerrado con este brief.

---

## 1. GET dedicado para instrucciones vs. campo en la respuesta para contexto `[dominio]`

Decisión de arquitectura ya tomada en Claude Chat, implementada tal cual: las
instrucciones estáticas (rol, grounding, citación, formato, scope) van por un `GET`
global; el contexto CAG inyectado va como campo opt-in en la respuesta de `/query`/
`/query/stream`.

**Por qué esta división y no un blob único**: la anatomía del brief 1 ya separaba
instrucciones (contrato con el modelo, archivo) de contexto (dato del corpus,
`reference.py`) — el sidebar de Streamlit las conceptualizó igual, como dos cosas
distintas. Pero la razón de fondo es de **ciclo de vida**:

- Las instrucciones son estáticas tanto en M1 como en M2 — el contrato con el modelo no
  depende de la pregunta ni de qué corpus haya detrás. Un `GET` global es estable y no se
  tira al pasar de módulo.
- El contexto es estático en M1 (un fragmento fijo) pero será **dinámico por-query en
  M2** (retrieval). Un `GET` global no tiene sentido ahí — no hay *un* contexto, hay "lo
  recuperado para *esta* pregunta". Que viaje con la respuesta de la query es lo único
  que es estable M1→M2.

## 2. Opt-in bajo flag, no en cada respuesta `[agnóstico]`

`injected_context` solo se puebla si `QueryRequest.include_context=True` (default
`False`). El contexto es **contenido**, no metadata operacional — en M2 serán *k* chunks
de retrieval, potencialmente varios KB. Meterlo en toda respuesta de producción infla el
payload de cada consumidor sin que lo haya pedido. Un flag lo activa solo para
diagnóstico (el sidebar de Streamlit). Mismo principio que ya guiaba la observabilidad de
S03 brief 4: log metadata, not content, by default.

## 3. Fidelidad por envoltura, no por copia `[agnóstico]`

El riesgo real de este brief no era técnico (es una función y un campo) sino de
**divergencia**: que lo que el endpoint de diagnóstico muestra deje de coincidir con lo
que el modelo REALMENTE recibió, en cuanto alguien tocara una de las dos copias sin
tocar la otra.

Se resolvió factorizando `_compose_system_prompt(context_block: str) -> str` a partir de
`build_system_prompt(fragments)` (que ahora es solo `_compose_system_prompt(
_build_context_block(fragments))`), y haciendo que `_cache_lookup` devuelva el
`context_block` que ya construye para la clave de caché. `answer_query`/
`stream_answer_query` reutilizan ESE MISMO string tanto para componer el prompt como
para `injected_context` — nunca hay una segunda llamada a `_build_context_block` en el
mismo ciclo de request. `GET /api/v1/prompt` sigue el mismo principio: delega en
`get_active_instructions()`, que devuelve literalmente `_STATIC_INSTRUCTIONS` (la
constante de módulo cargada por el loader del brief 1), no una relectura del archivo.

Es técnicamente cierto que en M1 `_build_context_block(ECSS_REFERENCE)` es una función
pura y determinista — llamarla dos veces daría el mismo string igualmente. Pero el punto
no es "en M1 no hay riesgo real", es **no depender de esa propiedad**: en M2, cuando el
contexto venga de retrieval (potencialmente no determinista, o simplemente costoso de
recalcular), una segunda derivación sí podría divergir o desperdiciar trabajo. Diseñar
la reutilización ahora, cuando es gratis, evita tener que re-descubrir el problema en M2.

## 4. `include_context` no es slot de caché `[dominio]`

El flag es puramente de **presentación** — no cambia qué se le pide al modelo ni qué se
genera, solo si el servidor ECHOEA el contexto de vuelta. Por tanto NO entra en
`build_cache_key`/`_cache_lookup`. Verificado con test: dos requests idénticas que solo
difieren en `include_context` dan el mismo `cache_hit` (la segunda es HIT, no dispara
una llamada nueva al modelo).

Consecuencia de diseño no trivial: en un HIT, `injected_context` se adjunta **fresco**,
tomado del `context_block` que `_cache_lookup` acaba de recomputar para ESA request — no
se guarda `injected_context` dentro del blob cacheado en Redis. Esto significa que el
HIT respeta el flag de la request ACTUAL, no el de la request que originalmente generó
la entrada cacheada (una entrada se puede cachear con `include_context=False` y servirse
después con `include_context=True`, o viceversa, sin ningún problema). Correcto en M1
porque `context_block` es puro y determinista respecto a `ECSS_REFERENCE` — recomputarlo
en el momento de servir un HIT da exactamente el mismo string que se usó para generar la
respuesta cacheada originalmente. **Advertencia para M2**: si el contexto pasa a
depender de retrieval no determinista (o de un corpus que cambia entre el momento de la
generación cacheada y el momento del HIT), recomputar en el momento de servir podría
devolver un contexto distinto del que realmente se usó para generar la respuesta
cacheada — en ese caso, fidelidad exigiría guardar `context_block` DENTRO de la entrada
cacheada. No se implementa aquí porque no aplica a M1. `# TODO: pendiente de decisión en
Claude Chat` si M2 lo requiere.

## 5. Sin versionado de prompt en el shape `[agnóstico]`

Git es el histórico de versiones (decisión ya tomada en brief 1). No se añade un
id/versión del prompt al contrato HTTP en este brief — sería prematuro sin evals que lo
necesiten.

## 6. Nota de exposición, no bloqueo `[dominio]`

`GET /api/v1/prompt` es público, sin auth, deliberadamente — para esta demo la
transparencia del prompt es una feature (el corpus ECSS también es público). Se anota,
no se implementa, que un despliegue a cliente real podría querer gating/auth en este
endpoint: fuera de alcance de este brief, decisión futura si el proyecto llega a ese
punto.

## 7. Qué NO se tocó

- `context/reference.py`: sin cambios.
- Contrato de citación (`Citation`, `_citations`): sin cambios — brief ortogonal al 1-fix.
- No se añadió versión/id de prompt al shape.
- El frontend sigue siendo un cliente HTTP fino: `sse_client.py` gana dos funciones
  (`stream_query` con `include_context`, `get_active_instructions` nueva), ningún import
  del backend.
