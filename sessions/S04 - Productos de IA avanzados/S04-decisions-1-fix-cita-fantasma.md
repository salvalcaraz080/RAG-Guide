# S04 — Decisions log, brief 1-fix: cortar la cita fantasma en rechazos

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier proyecto
> LLM) / `[dominio]` (específico de RAG-ECSS). Fix sobre el brief 1
> (`S04-decisions-1-robustez-prompt.md`), no una funcionalidad nueva.

---

## 1. El bug `[dominio]`

`_citations()` derivaba la cita candidata de `ECSS_REFERENCE` **incondicionalmente**,
respondiera o rechazara el modelo (diseño original de S02/S03: las citas nunca se extraen
del texto generado, siempre de los metadatos del fragmento inyectado). El brief 1 de esta
sesión introdujo un guardrail de scope que hace que el modelo rechace explícitamente
("out of domain" / "out of material"). Pero un rechazo seguía saliendo con
`citations: [ECSS-E-ST-40C 5.2.4.8]` pegada — el sistema decía "no cubro esto" y a la vez
adjuntaba una cláusula ECSS como si la respaldara.

**Por qué era grave y no diferible**: con un solo fragmento en M1, "out of material" es el
caso **común**, no el raro — casi cualquier pregunta que no toque ese fragmento concreto
cae ahí. Cuanto mejor rechazaba el guardrail del brief 1, más citas fantasma producía. Para
un artefacto cuya carta de presentación es la trazabilidad ante ingenieros ECSS, esto era
exactamente el fallo que el sistema entero existe para evitar.

**Qué NO es este fix**: no es verificación de grounding (¿la respuesta cita lo que
realmente usó al *responder*? ¿alucinó una cláusula?). Eso sigue siendo S11. Este fix
resuelve solo el caso binario y frecuente: si el modelo rechazó, no hay cita.

## 2. Frase natural fija, no marcador técnico `[agnóstico]`

Se instruyen dos frases marcador **exactas** al inicio de un rechazo (`Out of scope:` /
`Not covered by the provided material:`), detectadas por `startswith` en
`_refusal_kind()`.

Se descartó un marcador técnico tipo `OUT_OF_MATERIAL:` porque habría que stripearlo del
texto antes de mostrarlo al usuario. En `POST /api/v1/query` no es un problema (se
stripea antes de devolver la respuesta completa), pero en `/query/stream` el texto se
emite token a token conforme llega del proveedor (`event: chunk`) — para poder limpiar un
marcador técnico habría que **bufferizar el arranque del stream** hasta tener suficiente
texto para decidir si hay que stripear algo, lo cual es lógica exclusiva del path de
streaming que S03 prohibió explícitamente (el streaming es una capa de entrega sobre la
misma fuente de verdad, no un camino con reglas propias). Una frase natural resuelve esto
sin tocar el mecanismo: **es** parte del mensaje legible, se streamea tal cual, y la
detección ocurre en el punto de ensamblado compartido, no en el texto que ya viajó al
cliente.

## 3. Detección en el punto de ensamblado compartido, no en cada endpoint `[dominio]`

`_refusal_kind`/`_citations` viven en `llm_service.py`, en el mismo sitio que ya
compartían `answer_query` (`/query`) y `stream_answer_query` (`/query/stream`) desde S03
brief 3 (`_assemble_response`). El fix es una función y una llamada extra en un solo
sitio — no requirió tocar `routers/queries.py` ni el mecanismo SSE de `fastapi.sse`.
Verificado explícitamente en ambos paths:
- `/query` con un mock de `_call_model` devolviendo el marcador → `citations: []`
  (`test_queries.py`)
- `/query/stream` con un mock de `stream_llm` devolviendo el marcador → `citations: []`
  en el evento `done` (`test_streaming.py::test_stream_refusal_has_no_citations`)
- Verificado también contra Docker con las tres preguntas reales del brief (chocolate
  cake / ECSS-E-ST-70-41 / pregunta respondible), no solo mockeado

## 4. Case-sensitive a propósito `[agnóstico]`

`_refusal_kind` compara con `startswith`, sensible a mayúsculas. La instrucción del
prompt pide la frase **verbatim**; si el modelo la emite con una variación de mayúsculas,
eso ya es una señal de que no siguió la instrucción al pie de la letra — no se quiere
enmascarar esa desviación con una comparación case-insensitive que "arregle" el síntoma
sin que nadie se entere.

## 5. Sin campo nuevo en el contrato HTTP `[dominio]`

Igual que el brief 1, este fix no añade un campo tipado a `QueryResponse`/
`StreamDoneEvent` para señalar el rechazo programáticamente. `citations == []` sigue
siendo la señal de facto (en M1, una respuesta normal siempre lleva al menos la cita
candidata del único fragmento). Sigue anclado como pendiente para cuando entre structured
output (S11) o al diseñar el endpoint del brief 2.

## 6. Es un puente, no la solución definitiva `[agnóstico]`

Detección por prefijo de frase es frágil: si el modelo no emite la frase exacta (fallo de
instrucción, o un tercer proveedor no verificado), `_refusal_kind` devuelve `None` y la
cita candidata vuelve a colarse. Se mitiga con la instrucción explícita y verbatim en
`system.md`, verificada `real_llm` contra Anthropic y OpenAI (los 6 tests de
`test_system_prompt_provider_agnostic.py` pasan, incluyendo las 4 aserciones de rechazo
que ahora comprueban `citations == []` en vez de escarbar el texto con regex buscando
cláusulas inventadas). Se jubila cuando structured output (S11) dé un campo tipado en vez
de inferir el rechazo del texto.

## 7. Consecuencia de caché `[dominio]`

Añadir las frases marcador a `system.md` cambia el prompt final otra vez → el snapshot de
caracterización del brief 1 se actualiza otra vez → las entradas cacheadas con el prompt
previo (sin las frases marcador) quedan huérfanas y expiran por `CACHE_TTL`. Misma
invalidación por versión de prompt que el brief 1 y `corpus_version` — nada especial que
hacer.

## 8. Regresión que documenta el bug `[dominio]`

`tests/app/test_queries.py::test_query_grounding_not_covered_does_not_fabricate_citation`
(de S02) afirmaba literalmente el comportamiento que este fix corrige: mockeaba un
rechazo sin frase marcador ("The provided reference does not cover this.") y aseveraba
`citations == [...]` (la cita candidata) como si fuera correcto. Se reescribió para usar
el marcador real (`Not covered by the provided material:`) y aseverar `citations == []`
— el test pasaba mecánicamente antes del fix (el texto viejo nunca activaba
`_refusal_kind`), pero su intención documentada era exactamente la cita fantasma. Es el
test que, de haber existido con la intención correcta, habría cazado el problema antes.
