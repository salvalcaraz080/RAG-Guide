# S03 — Wrappers: abstracción de proveedor, cache, streaming, observabilidad

> Resumen provisional (paso 2). Teoría destilada + decisiones de esta sesión, **antes** de
> implementar. La divergencia teoría-vs-implementación se captura en el consolidado (paso 5),
> no aquí. Este documento no audita el material del máster: destila lo que aplica a RAG-ECSS y
> registra qué se decidió y qué queda abierto.

Cinco bloques temáticos (abstracción de proveedor + fallback, cache, streaming, observabilidad,
interfaz). El seam `_call_model` en `app/services/llm_service.py` que S02 dejó preparado es el
punto de inserción de casi todo lo de esta sesión; el orden del prompt (estático antes que
variable) ya deja sitio a un breakpoint de cache sin reordenar.

---

## 1. Abstracción de proveedor + fallback

**Qué resuelve.** Desacoplar la lógica de negocio del SDK de un proveedor concreto, de modo que
cambiar de modelo/proveedor sea configuración, no reescritura. Habilita además el fallback
automático: si un proveedor cae, el sistema rota al siguiente sin intervención.

**El principio de abstracción ya está medio resuelto en el repo.** El seam `_call_model` es la
frontera única que aísla el SDK. Lo que S03 decide no es "¿abstraigo?" (ya está), sino "¿con qué
lleno el seam?". Dos caminos reales: librería de agregación (LiteLLM) o adaptadores propios finos
por proveedor.

**Herramientas del ecosistema** (mapa, no receta): LiteLLM (agregador ligero, 100+ modelos, router
con fallback/retry, tracking de coste, rate-limiting), OpenRouter (marketplace con facturación
consolidada — descartado por privacidad: los datos pasan por un tercero), LangChain (framework
completo — sobredimensionado para solo abstraer; descartado también por decisión previa del
proyecto de no usarlo). 

**Decisión: LiteLLM entra en el MVP.**
- El fallback es **obligatorio para este artefacto**, no por frecuencia de caídas sino por
  **severidad**: RAG-ECSS es una demo/carta de presentación ante PLD Space. Una caída de proveedor
  durante la demo en directo es un fallo de máxima severidad aunque de baja probabilidad, y el
  coste de mitigarlo es bajo. Es un seguro barato contra un evento catastrófico-para-el-objetivo.
- LiteLLM se elige sobre adaptador propio porque el fallback + la normalización de streaming entre
  proveedores (ver bloque 3) son fontanería que la librería resuelve, y porque permite cambiar de
  proveedor principal/fallback por precio o capacidad con cambio de configuración. Se valida por
  implementación previa del alumno en el proyecto del máster (funcionó).
- Va **detrás del seam `_call_model`**, con clientes **async-nativos** (`acompletion` /
  `router.acompletion`, `async for` en streaming). Async-correctness es requisito duro: los
  ejemplos síncronos del material congelarían el event loop de FastAPI.

**Estrategias de fallback.** Secuencial (lista ordenada por preferencia; la común) y por tipo de
error (no todo error merece rotar: auth inválida no se reintenta, timeout sí, rate-limit rota). El
routing por complejidad (query simple → modelo barato; compleja → modelo potente) es patrón de
sesiones de agentes, no de aquí.

**Higiene de dependencias.** Fijar versión y verificar hashes en `pyproject.toml`. LiteLLM tuvo un
compromiso de supply-chain en PyPI (marzo 2026, versiones puntuales) — recordatorio de que aplica a
cualquier dependencia, no solo a esta.

**Guide (agnóstico).**
- *Abstracción de proveedor + fallback* — **Madurez:** el fallback cross-proveedor suele ser nodo
  de producción con SLA; **excepción de aplicabilidad:** cuando el artefacto es una demo de alto
  valor, sube a MVP porque el fallo en directo es catastrófico. **Condición:** ≥2 proveedores que
  realmente necesites cablear y validar (un fallback a un proveedor no validado no es fallback).
  **Señal:** caídas de proveedor observadas, o necesidad de rotar por precio/capacidad.
- *Librería de agregación vs adaptador propio* es un nodo del árbol por sí mismo. La librería gana
  con muchos proveedores y cuando quieres retry/coste/rate-limit "gratis"; el adaptador propio gana
  con dos proveedores y cuando la normalización es objetivo de aprendizaje. Tensión estructural a
  registrar: **toda capa de abstracción unificada puede aplanar features específicos de un
  proveedor** (ver bloque 2, `cache_control`).

---

## 2. Cacheo de respuestas

**Tres capas distintas** (no confundir):
1. **Exact-match** — hash determinista del input (+ modelo + system prompt) → store (Redis).
   Evita la llamada entera. Agnóstico de proveedor. Microsegundos.
2. **Semántico** — embedding del input + similitud coseno sobre umbral. Captura reformulaciones.
   Agnóstico de proveedor. Requiere vector store. Diferido a S07/S08.
3. **Prompt caching del proveedor** (`cache_control` de Anthropic) — no cachea la respuesta;
   abarata el procesamiento del **prefijo repetido**. Específico de proveedor. Mínimo 1024 tokens.

**Decisión: exact-match para M1.** Encaje de dominio **más fuerte** que en el estimador del máster:
para un corpus normativo, que la misma pregunta devuelva la misma respuesta no es solo ahorro — es
**auditabilidad**. Una respuesta normativa que varía entre consultas idénticas es un problema de
trazabilidad. El determinismo cacheado es un beneficio de compliance.

**Clave de caché e invalidación** (el material razona sobre CAG estático; conviene mirar más lejos):
- En M1 (fragmento estático en el system prompt) la clave `input + modelo + system_prompt` es
  segura: el contexto va en el system, que ya entra en la clave.
- En **M2** (contexto inyectado por retrieval) esto **deja de ser seguro**: la misma pregunta puede
  recuperar chunks distintos si el índice cambió. Cachear por `pregunta + system` serviría una
  respuesta vieja sobre chunks que retrieval ya no devolvería — una **cita fantasma servida desde
  caché**, el modo de fallo que el sistema entero existe para evitar. La clave en M2 debe incluir el
  contexto recuperado (o un hash de los IDs de chunk).
- **Invalidación:** para corpus versionado, la respuesta correcta cambia cuando cambia el **corpus**
  (nueva revisión de un estándar), no en una ventana temporal. TTL puro encaja mal; la vía correcta
  es invalidación por **versión de corpus** (namespace/tag en la clave). Es la "invalidación por
  evento" que el material trata como excepcional pero que aquí es la principal.

**Prompt caching (capa 3): diferido a M2, con gate claro.** En M1 el prefijo estático (system +
un fragmento ECSS) plausiblemente no alcanza el mínimo de 1024 tokens, así que `cache_control` ni
se activaría — no es funcionalidad que se pierda, es funcionalidad que aún no aplica. Cuando el
retrieval (S06/S07) infle el prefijo con varios chunks, el prefijo cruzará el umbral y **ahí** toca
verificar si LiteLLM pasa `cache_control` de Anthropic sin aplanarlo (ventaja estructural de
Anthropic: hasta 4 breakpoints en cualquier punto de la petición). Verificación diferida a ese
momento.

**Semántico: diferido y bajo sospecha de dominio.** Además de necesitar vector store (S07/S08),
arrastra un riesgo específico normativo: "criticidad B" vs "criticidad C" son casi idénticas en
espacio de embedding pero apuntan a cláusulas distintas. Un hit semántico entre ambas sirve la
norma equivocada con apariencia de correcta. Puede ser **inadecuado** para RAG-ECSS incluso cuando
esté disponible, o exigir umbrales muy altos + validación específica.

**Métricas de caché** (hit rate, coste evitado, latencia hit-vs-miss, tasa de stale) aterrizan en la
capa de observabilidad (bloque 4). Son el instrumento que gradúa las decisiones abiertas: "¿mereció
la pena el cache?" se responde midiendo, no por intuición.

**Guide (agnóstico).**
- *Exact-match cache* — **Condición:** tarea determinista donde la repetición es deseable (no
  generación creativa). En corpus normativo, además, beneficio de auditabilidad. **Señal:** inputs
  repetidos observados (hit rate > ~20% justifica la infra).
- *Clave de caché sobre retrieval dinámico* — **Condición:** el contexto lo inyecta retrieval, no es
  estático → la clave debe incluir el contexto recuperado, o sirve respuestas desancladas. Nodo M2.
- *Invalidación por versión de corpus > TTL* — **Condición:** corpus versionado que cambia en
  eventos discretos y raros, no continuamente.
- *Cache semántico* — nodo del árbol aunque no acabe en el producto: **Condición:** reformulaciones
  frecuentes del mismo intent **y** dominio donde diferencias léxicas mínimas no cambian la
  respuesta correcta (RAG-ECSS falla esta segunda condición).

---

## 3. Streaming

**Qué resuelve.** UX de latencia percibida: el usuario ve la respuesta escribiéndose en vez de un
spinner. El tiempo total no cambia; el primer token llega en ms.

**Mecanismos:** StreamingResponse (chunks crudos, mínimo overhead), **SSE** (eventos estructurados
+ reconexión automática; estándar de facto para LLM web), WebSockets (bidireccional, reservado para
agentes que piden clarificación — fuera de alcance).

**Decisión: SSE en el endpoint, patrón heredado y validado.** El alumno ya montó este patrón en el
proyecto del máster con la arquitectura correcta (**ruta cliente-HTTP**): Streamlit habla HTTP puro
(`requests` + `sseclient-py`) contra un endpoint FastAPI que emite `ServerSentEvent`; LiteLLM vive
solo en el backend; la separación está enforced a nivel de imagen Docker (el frontend no tiene
`litellm` instalado). Para RAG-ECSS se **porta el patrón** (no se copia el código del estimador):
endpoint `POST /api/v1/query/stream`.

**Jerarquía de paths (importante).** El endpoint **no-stream** (`/query`, ya existente) es el camino
principal y la **fuente de verdad**: construcción de citas, grounding y validación viven ahí. El
`/query/stream` es una mejora de UX **encima**, no un reemplazo. Consecuencia: no meter lógica que
solo exista en el path de streaming. Esto hace que **eliminar el streaming sea barato** si da
problemas — es estético y descartable por diseño.

**Alcance del stream.** Solo el **texto** de la respuesta se streamea. Las citas (`standard`/
`clause`) y las métricas se pintan **al cierre** del stream (o en el JSON de cierre), no como eventos
tipados intermedios. Razón: la propiedad central es que la respuesta *lleve* la cita verificable, no
*cuándo* aparece en pantalla; nadie audita una cláusula en tiempo real mientras se escribe.
Streamear las citas como eventos SSE tipados sería complejidad sin ganancia. El patrón heredado ya
admite eventos tipados si en el futuro hiciera falta, pero por ahora no.

**Respuestas largas / truncamiento.** `max_tokens` generoso (`MAX_OUTPUT_TOKENS=2000` ya puesto) +
**detección de `finish_reason`**. En dominio normativo esto **no es cosmético**: un
`finish_reason="length"` puede cortar una respuesta antes de su cita → afirmación normativa sin
ancla de trazabilidad. Es un **fallo de integridad**, no un aviso de UX: se detecta y se marca
(eleva a warning en el log). **Descartada** la estrategia del material de "limita la salida a N
palabras por prompt": compite con la **completitud** normativa (el modelo podría omitir requisitos
aplicables para caber en el límite).

**Adaptador cliente.** `st.write_stream` no consume SSE-por-red directamente; necesita un cliente
SSE (`sseclient-py`) que traduzca el `text/event-stream` al iterador que Streamlit renderiza. Es
poco código pero no es cero (el material lo omite porque asume Streamlit acoplado al LLM en proceso).

**Currency a verificar antes del brief de streaming.** El material afirma soporte SSE nativo en
FastAPI (`fastapi.sse`, `EventSourceResponse`) desde 0.135.0. Verificar; si es inexacto, la vía es
`sse-starlette` o `StreamingResponse` con `media_type="text/event-stream"`. (Nota: el alumno reporta
que LiteLLM moderno ya trae mecanismos SSE propios que simplifican el tramo proveedor→backend; el
tramo backend→cliente sigue siendo responsabilidad del endpoint.)

**Guide (agnóstico).**
- *Streamear lo que mejora la latencia percibida (texto largo), no lo estructuralmente secundario
  y barato de pintar de golpe (metadatos, citas, métricas).* Regla de balance complejidad/ganancia,
  transferible a cualquier RAG.
- *El path de streaming se construye encima del no-stream, que es la fuente de verdad* → el stream
  es descartable sin cirugía. **Madurez:** MVP si el patrón ya está resuelto (como aquí); primer
  candidato a recortar si hubiera que construirlo desde cero.
- *En sistemas con trazabilidad obligatoria, el truncamiento es fallo de integridad, no de UX.*

---

## 4. Observabilidad, logging y trazabilidad

**Distinción de dominio (crítica).** "Trazabilidad" nombra **dos cosas distintas** que comparten
palabra por accidente:
1. **Traza operacional** (este bloque, S03): qué request, qué modelo, tokens, latencia, coste,
   fallback, cache_hit. Telemetría de infraestructura.
2. **Traza de cita** (S11): qué cláusula fundamentó qué afirmación. Propiedad central del producto.
No son la misma capa ni se implementan juntas. S03 entrega (1); (2) es concern de dominio de S11.

**Decisión: structlog para observabilidad operacional en MVP.** Ya está en el stack. Config dual
(consola legible en dev, JSON en prod), procesadores encadenados, `bind()` contextual (request_id en
middleware). Patrón log-al-inicio / log-al-completar / log-en-error.

**Campos por llamada:** modelo, proveedor, tokens_in/out, coste estimado, latencia, `cache_hit`,
`fallback_used` (+ proveedor destino si rotó), `finish_reason`, tipo de error / reintento en fallo.

**Dos enganches que el material no hace explícitos:**
- El log es lo que hace el **fallback auditable**. Un fallback silencioso es casi tan peligroso como
  no tenerlo: `fallback_used` es lo que convierte "el proveedor primario cayó 3 veces durante la
  demo" en información en vez de misterio.
- El log de coste es el **instrumento que gradúa las decisiones abiertas** de S03 (¿el cache valió
  la pena? ¿activar prompt caching en M2?). Sin él se responde por intuición.
- `finish_reason="length"` se loguea como **warning**, no info: en dominio normativo significa
  "respuesta posiblemente sin cita", no "respuesta larga".

**Herramientas de observabilidad = nodo de escalado, no MVP.** structlog basta para un artefacto sin
volumen de producción. Añadir plataforma visual (Logfire / Langfuse) resuelve un problema
—observabilidad a escala, dashboards de coste para stakeholders— que el MVP no tiene.
- **Logfire** tiene el mejor encaje estructural *si* llega el momento (equipo Pydantic, sobre
  OpenTelemetry, instrumentación nativa FastAPI + LiteLLM + Redis).
- **Langfuse** gana si pesa open-source / self-hosted (privacidad).
- **LangSmith** queda fuera por construcción: su valor depende de LangChain, que el proyecto
  descartó. (No heredar del temario "usaremos LangGraph en M5" — es otra decisión abierta.)

**Superficie de exposición (nodo agnóstico).** El structured logging captura prompts y respuestas →
en dominios con datos sensibles, el log es superficie de exposición (GDPR, S06). Para ECSS (corpus
público) no muerde; para un corpus con datos sensibles (p. ej. el proyecto de localización), sí.

**Guide (agnóstico).**
- *structured logging (structlog) con campos LLM-específicos* — incondicional en cualquier proyecto
  con LLMs; la decisión es *qué campos*, no *si*. No lleva condición.
- *Plataforma de observabilidad visual* — **Madurez:** escalado/producción. **Señal:** el volumen de
  llamadas hace insostenible debuggear por terminal, o hacen falta dashboards de coste para
  stakeholders.
- *Trazabilidad operacional ≠ trazabilidad de cita* — distinción de dominio a preservar; no
  fusionar S03 con S11.
- *El log como superficie de exposición* — **Condición:** corpus/queries con datos sensibles.

---

## 5. Interfaz conversacional (Streamlit)

**Rol.** Frontend MVP: pegar una consulta y ver la respuesta ECSS (con citas) sin curl/Postman.
Cliente HTTP fino de los endpoints `/query` y `/query/stream`. Es demo + carta de presentación, no
producto pulido — y Streamlit es débil como producto final pero excelente como demo técnica, que es
justo lo que este artefacto necesita ser.

**Comparativa de frameworks** (valor del material): Streamlit (generalista, mejor ecosistema de
componentes, sidebar nativo, multipage), Gradio (demos rápidas, share link temporal, multimodal),
Chainlit (chat-only, observabilidad de razonamiento de agente out-of-the-box). **Streamlit gana para
RAG-ECSS** por tres ejes independientes: el Nivel 3 pide **sidebar** (solo Streamlit lo tiene
nativo), renderizar **citas** pide riqueza de componentes, y **no hay agentes** en M1 (el
diferenciador de Chainlit es irrelevante ahora).

**"Conversacional" es metáfora de UI, no multi-turn.** El ejercicio mantiene historial visible
(`st.session_state`) pero **no pasa el historial al LLM**: cada consulta es una llamada independiente
con el mismo system prompt. Mapea limpio sobre el endpoint single-turn actual sin tocar schema ni
prompt. El multi-turn real (pasar turnos previos como contexto, follow-ups) es **S05**, no este
ejercicio. Consecuencia: `st.session_state` guarda un log presentacional, nada que un frontend futuro
tenga que "portar" → **la migración a un frontend sólido es trivial, y lo es por el scope
(single-turn) tanto como por el desacople HTTP.**

**El sidebar (Nivel 3) no necesita observabilidad server-side.** `QueryResponse` ya transporta
`model` / `provider` / `usage`; la latencia la cronometra el cliente. El artículo 5 (traza
persistente) es concern ortogonal, no prerequisito del sidebar.

**Guide (agnóstico).**
- *Elección de framework de UI es dependiente de madurez y de forma de la app.* **Condición:**
  ¿necesitas más que chat (sidebar, métricas, tablas)? → Streamlit. ¿demo multimodal compartible? →
  Gradio. ¿agentes con razonamiento multi-paso que inspeccionar? → Chainlit (reabre en S12+).
- *Frontend como cliente HTTP del endpoint, no acoplado en proceso* — desacopla la migración futura
  a puro trabajo de frontend. Enforced a nivel de dependencias/imagen.

---

## Estructura de briefs y secuenciación

Criterio: **agrupar solo lo que se testea junto.** No cinco briefs simétricos — un núcleo acoplado
y dos satélites.

**Brief A — Núcleo LLM** (wrapper + fallback + cache + observabilidad).
Todo lo que se sienta sobre `_call_model` en el path **no-stream**. Se agrupan porque forman una
**unidad de comportamiento** inseparable en el test: el logging registra `fallback_used` y
`cache_hit` (campos que solo existen si fallback y caché existen); la métrica de coste evitado se
calcula en el log. Separarlos obligaría a mockear tanto que el test perdería valor. Sub-tests:
rotación de fallback, hit/miss de caché con clave correcta, emisión de campos de log. Async-correcto.

**Brief B — Streaming** (`/query/stream` + modo stream del wrapper).
Path distinto con superficie de test distinta (emisión de eventos SSE + detección de `finish_reason`,
no valor de retorno). Depende de A pero es un modo de invocación separado. Aislarlo mantiene el
streaming **descartable** sin tocar A.

**Brief C — Streamlit** (frontend cliente HTTP de A y B).
Test manual de UI, no pytest. Última por dependencia (necesita los endpoints vivos).

**Secuenciación real: A → B → C.** No es el orden del temario (1→5); es el orden de dependencia. El
material pone la interfaz primero por motivación pedagógica; en construcción real es la última porque
consume lo que las otras producen.

---

## Estado: decidido vs abierto

**Decidido (a graduar tras implementar):**
- LiteLLM detrás del seam `_call_model`, async-nativo, con fallback secuencial. Fallback en MVP por
  severidad-para-la-demo.
- Cache exact-match para M1 (auditabilidad, no solo ahorro). Caché de LiteLLM vs `LLMCache` propia =
  sub-decisión del brief A (el control de la clave importa en M2).
- SSE en el endpoint, patrón heredado (ruta cliente-HTTP, `sseclient-py`, desacople por Docker).
  Streaming = UX descartable sobre el path no-stream (fuente de verdad). Solo texto en el stream.
- `finish_reason="length"` = fallo de integridad → warning. Descartada la limitación por prompt.
- structlog para observabilidad operacional en MVP. Plataforma visual = nodo de escalado.
- Streamlit como frontend MVP, cliente HTTP fino, el último.
- Distinción trazabilidad operacional (S03) ≠ trazabilidad de cita (S11).

**Abierto (marcado, no forzado):**
- Passthrough de `cache_control` de Anthropic (capa 3) — gate en M2, cuando el prefijo supere 1024
  tokens tras retrieval.
- Clave de caché sobre retrieval dinámico (M2): incluir contexto recuperado / hash de chunk IDs.
- Invalidación por versión de corpus > TTL (M2).
- Cache semántico (S07/S08) + sospecha de dominio (criticidad B vs C).
- Herramienta de observabilidad visual concreta (escalado).
- Currency a verificar en brief B: soporte SSE nativo de FastAPI vs `sse-starlette`.
