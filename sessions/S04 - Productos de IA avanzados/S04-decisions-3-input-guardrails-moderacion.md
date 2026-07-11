# S04 — Decisions log, brief 3: pipeline de input-guardrails (moderación de toxicidad)

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier proyecto
> LLM) / `[dominio]` (específico de RAG-ECSS). Este brief **reabre S04**, ya consolidada
> tras el brief 2 — ver §9 del brief para el backfill pendiente en Claude Chat
> (addendum a `S04-consolidado.md` + ajuste de la Guide, Phase 5).

---

## 1. El orden es la decisión central, no un detalle `[agnóstico]`

`run_input_guards(question)` corre **antes** de `_cache_lookup`, en `answer_query` y en
`stream_answer_query`. No es una optimización de latencia (de hecho es lo contrario:
añade una llamada de red incluso en lo que sería un hit instantáneo) — es un
**invariante de seguridad**: el guardrail decide si una petición MERECE entrar al
sistema; el cache decide cómo responder a las que sí entran. Si la moderación corriera
después del lookup, un HIT se serviría sin haber pasado moderación, creando una clase de
peticiones — las que aciertan cache — que nunca se moderan.

**Por qué esto importa incluso siendo hoy "inalcanzable"**: con cache exact-match, un
input distinto genera una clave distinta → siempre es un miss → siempre se modera. El
bypass es hoy teórico. Pero la corrección del guardrail **no debe depender de qué
algoritmo de cache haya debajo** — el día que entre normalización de clave (p. ej.
lowercase + trim antes de hashear) o **caché semántica** (nodo ya previsto en la
arquitectura de RAG-ECSS), "input distinto que acierta el mismo hit" deja de ser
improbable, y el bypass se abriría solo, en silencio, sin que nadie lo decidiera
explícitamente. Poner el guardrail antes del lookup no cuesta nada hoy y cierra esa
puerta por diseño, de una vez, en vez de tener que acordarse de revisarlo cuando llegue
la caché semántica.

**Test que fija esto como regresión** (no solo como intención en un comentario):
`test_input_guards.py::test_toxic_input_rejected_even_if_it_would_have_hit_cache` —
precachea una respuesta, repite la MISMA pregunta (que daría HIT) pero con un veredicto
de moderación distinto (flagged), y verifica que se rechaza con `400` en vez de servir
el hit. Es el test que documenta el invariante, no solo lo comprueba de pasada.

## 2. Honestidad: toxicidad ≠ prompt injection `[agnóstico]`

La Moderation API de OpenAI está entrenada para contenido tóxico (acoso, violencia,
sexual, self-harm) — **no** para prompt injection (intentos de manipular las
instrucciones del sistema vía el input del usuario). Son amenazas distintas con
detectores distintos. Este brief monta el **andamiaje del pipeline** (una secuencia
ordenada de checks, cada uno con política declarada) precisamente para que la segunda
capa (injection, M2+) entre en su sitio sin reescribir el flujo — pero NO la implementa.
El hueco queda como comentario explícito en `run_input_guards`
(`# TODO: injection guardrail (M2+)`), no como código muerto ni como una promesa vaga.

Hoy, la única mitigación de injection que existe en el sistema es indirecta: la robustez
de prompt del brief 1 (scope reforzado, frases marcador de rechazo) hace más difícil que
una instrucción inyectada en el input logre que el modelo se salga de su contrato — pero
eso es un efecto secundario de otro brief, no un guardrail dedicado.

## 3. No cachear el veredicto de moderación `[dominio]`

Decisión explícita, no un descuido: la Moderation API es gratuita y de baja latencia
(~50-100ms), así que se acepta que corra en **toda** request, incluidos los hits.
Consecuencia aceptada: un hit deja de ser prácticamente instantáneo (~0.18s medido en
S03 brief 2) y pasa a llevar esa llamada de red por delante. El trade-off es deliberado:
seguridad/higiene del input por encima de la latencia del hit. Cachear el veredicto
habría reintroducido exactamente el mismo riesgo del §1 (un hit que se sirve sin pasar
moderación de verdad, esta vez porque el veredicto cacheado podría quedar obsoleto si
las políticas de moderación de OpenAI cambian) a cambio de un ahorro de latencia que no
compensa.

## 4. Fail-open vs. fail-closed `[agnóstico]`

Sub-decisión explícita, no dada por hecha. Dos opciones:

- **Fail-open** (elegida): si no se puede moderar (sin `OPENAI_API_KEY`, o la llamada
  falla), la pregunta pasa, con un warning explícito (`moderation_unavailable`,
  structlog). No bloquea un despliegue solo-Anthropic ni tumba el servicio si OpenAI
  está caído.
- **Fail-closed** (descartada): si no se puede moderar, se rechaza. Garantiza que nada
  pasa sin moderar, pero un despliegue solo-Anthropic (`OPENAI_API_KEY` no configurada,
  escenario ya soportado desde S03 brief 1) dejaría el servicio **inservible** — todas
  las preguntas rechazadas por falta de moderación, no por su contenido.

Se eligió fail-open siguiendo el mismo patrón de degradación ya establecido en el
proyecto: Redis degradable (S03 brief 2, cache es optimización, no correctitud) y el
fallback opcional de OpenAI (S03 brief 1, el Router arranca solo-Anthropic sin la key).
Una capa de seguridad/optimización degrada con log explícito y observable, no hace
fail-fast y se lleva el servicio entero por delante.

**Deuda registrada, no oculta**: fail-open significa que la moderación es una garantía
**condicional** a que OpenAI esté disponible — no absoluta. Aceptable para esta demo; un
despliegue a cliente real con requisito duro de moderación necesitaría fail-closed, o
un proveedor de moderación independiente del LLM primario (para no acoplar la
disponibilidad de la moderación a la disponibilidad del proveedor de completions).

## 5. Cliente OpenAI directo, no vía el Router de LiteLLM `[dominio]`

`input_guards.py` construye su propio `AsyncOpenAI` (SDK de `openai`, ya dependencia del
proyecto), en vez de enrutar la moderación a través del `litellm.Router` de
`llm_wrapper.py` (que sí soporta `amoderation()`). Motivo: el Router existe para
abstraer **completions** con fallback secuencial entre proveedores de chat
(`ecss-primary`/`ecss-fallback`) — la moderación no es una completion, no tiene
fallback, y mezclarla en el mismo `model_list` habría acoplado dos responsabilidades
(elegir qué LLM responde vs. vetar qué preguntas entran) sin necesidad real. Aislarla en
su propio cliente mantiene `llm_wrapper.py` enfocado en una sola cosa.

## 6. Verificación del modelo vigente `[agnóstico]`

`omni-moderation-latest` — verificado en la documentación oficial de OpenAI en el
momento de implementar (no asumido de un snippet de ejemplo, misma disciplina que
"verify model currency" ya aplicada a `LLM_MODEL`/`LLM_FALLBACK_MODEL`). Es el modelo de
moderación multimodal (texto + imagen) que sustituyó a `text-moderation-latest`
(retirado); en este proyecto solo se modera texto. Es gratuito.

## 7. No se loguea el texto de la pregunta `[agnóstico]`

Ni en el veredicto positivo (`moderation_passed`, sin cuerpo) ni en el negativo
(`moderation_flagged`, solo las categorías disparadas: `["harassment", ...]`, nunca el
texto que las disparó). Mismo principio "log metadata, not content" que ya regía la
observabilidad operacional de S03 brief 4 — el input del usuario es la superficie de
exposición más sensible del sistema; no hay razón para persistirlo en logs de
diagnóstico, ni siquiera cuando el propio contenido es lo que se está evaluando.

## 8. Aislamiento en tests: por qué hizo falta una fixture nueva `[dominio]`

`OPENAI_API_KEY` SÍ está configurada en `.env` de este proyecto (para el fallback del
Router, S03 brief 1) — así que sin aislamiento explícito, `uv run pytest` habría hecho
una llamada real a la Moderation API en **cada test** que pasa por `answer_query`/
`stream_answer_query` (la mayoría de la suite). Se añadió una fixture `autouse` en
`tests/app/conftest.py` (`isolate_input_guards_from_real_moderation`) que fuerza
`input_guards._moderation_client = None` — toma la ruta fail-open (sin red) por
defecto, igual que `isolate_cache_from_real_redis` ya hacía con Redis. Los tests que
SÍ quieren ejercitar moderación real la sustituyen localmente (un doble en memoria para
los tests mockeados; el cliente real reconstruido a mano solo en los dos tests
marcados `real_llm`, que necesitan deshacer explícitamente el efecto de la fixture
autouse).

## 9. Qué NO se tocó

- `build_cache_key`: sin cambios, la moderación no añade slots.
- El contrato de respuesta en el camino feliz: sin cambios.
- `context/reference.py`, el prompt, las citas: sin cambios.
- Ningún guardrail de injection implementado (solo el hueco comentado).

## 10. Pendiente para Claude Chat (backfill, no CC)

Este brief reabrió S04 tras su consolidación. Al cerrarlo, falta:
- Addendum a `S04-consolidado.md` (no reescribirlo) con esta decisión y su aprendizaje
  agnóstico: la Moderation API es una llamada de red que AÑADE latencia, no la ahorra;
  cubre toxicidad, no injection; el orden input-guardrails-antes-del-cache es invariante
  de seguridad, no eficiencia.
- Ajuste a la Guide (Phase 5, nodo de guardrail tooling): corregir que la moderación sea
  "near-irrelevant for a normative technical corpus" — la relevancia la decide la
  superficie de input abierto (cualquier endpoint público que acepta texto de usuario),
  no el dominio del corpus. Añadir también, en el nodo guardrails↔cache, que el orden
  "input antes del lookup" es invariante independiente del algoritmo de cache (se rompe
  con caché semántica si se coloca después del hit).
