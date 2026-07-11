# S04 — Brief 3: pipeline de input-guardrails (moderación de toxicidad, antes del cache)

> **Para:** Claude Code. **Sesión:** S04 (reapertura tras consolidación — ver §9).
> **Tipo:** dominio (RAG-ECSS), con aprendizaje agnóstico.
> Si encuentras una decisión no cubierta: `# TODO: pendiente de decisión en Claude Chat`.
> `.claude/CLAUDE.md` manda si hay contradicción.

---

## 1. Objetivo

Introducir un **pipeline de input-guardrails**: un punto único que evalúa la pregunta del usuario
**antes de** tocar el cache o el LLM. Primer (y único, por ahora) inquilino: **moderación de
contenido tóxico** vía la Moderation API de OpenAI. El andamiaje deja hueco explícito para una capa
futura de detección de prompt injection (M2+), que **no** se implementa aquí.

**Por qué andamiaje y no solo la moderación suelta:** se sabe que vendrán más capas de input
(injection). Montar el pipeline ahora —una secuencia ordenada de checks, cada uno con su política de
fallo declarada— evita reescribir el flujo cuando llegue la segunda capa. Moderación entra como el
primer check; injection será el segundo, en su sitio, sin rework.

---

## 2. El orden es la decisión central (no negociable)

```
request
  → INPUT GUARDRAILS PIPELINE        ← nuevo (este brief)
      → moderación de toxicidad
      → [hueco: injection, M2+]
  → cache lookup                     ← existente (S03 brief 2)
      → HIT: devuelve respuesta cacheada
      → MISS: build prompt → LLM → cachea → responde
```

**Los input guardrails corren ANTES del cache lookup.** No es optimización, es un invariante de
seguridad: **el guardrail decide si una petición merece entrar al sistema; el cache decide cómo
responder a las que sí entran.** Si la moderación fuera después del lookup, un HIT se serviría sin
haber pasado moderación — creando una clase de peticiones (las que aciertan cache) que nunca se
moderan. En exact-match esto hoy es prácticamente inalcanzable (un input distinto genera clave
distinta → miss → sí se modera), pero **la corrección del guardrail no debe depender de qué
algoritmo de cache haya debajo**: el día que entre normalización de clave o **caché semántica**
(nodo ya previsto), "input distinto que acierta el mismo hit" deja de ser improbable y el bypass se
abriría solo, en silencio. Poner el guardrail antes del lookup no cuesta nada y cierra eso por
diseño. **No lo muevas después del lookup bajo ninguna optimización.**

---

## 3. Decisiones ya tomadas (no re-decidas)

1. **Orden:** input guardrails → cache lookup → LLM. (§2.)
2. **Alcance del inquilino:** solo moderación de **toxicidad**. Honestidad sobre qué cubre: la
   Moderation API está entrenada para contenido tóxico (acoso, violencia, sexual, self-harm), **no**
   para prompt injection. La injection la mitiga hoy parcialmente la robustez de prompt del brief 1;
   una capa dedicada es trabajo futuro (hueco en el pipeline, no implementado).
3. **Política de fallo: `exception`.** Toxicidad clara → se aborta con `400` (o `422`) y mensaje
   claro. **No** se degrada, **no** se reintenta, **no** se cachea. La respuesta de moderación no
   entra en el cache de respuestas.
4. **Sin cachear el veredicto de moderación.** Decisión explícita: la Moderation API es gratis y de
   baja latencia; se acepta que corra en **toda** request (incluidos los hits) sin cachear su
   resultado. Consecuencia aceptada: un hit deja de ser instantáneo (~0.18s) y pasa a llevar una
   llamada de red a OpenAI (~50-100ms) por delante. Trade-off elegido: seguridad/higiene del input
   por encima de la latencia del hit.
5. **Moderación degradable, no fail-fast** — ver §5 (la sub-decisión fail-open/closed).

---

## 4. Implementación

### 4.1 Estructura del pipeline
- Un módulo/función que representa el pipeline de input-guardrails, p. ej.
  `app/services/input_guards.py` con una función `run_input_guards(question: str) -> None` que
  ejecuta los checks **en orden** y lanza una excepción tipada al primer fallo.
- Cada check es una unidad con su **política declarada** (hoy solo `moderate_toxicity`, política
  `exception`). El hueco de injection se deja como comentario explícito
  (`# TODO: injection guardrail (M2+) — segundo check del pipeline, aquí`), no como código muerto.
- Excepción tipada propia, p. ej. `InputGuardrailError` (con el motivo: qué check disparó), que el
  router traduce a `400`/`422`.
- Comentarios inline en español (qué es el pipeline, por qué corre antes del cache, qué es la
  política `exception`, por qué la moderación es una llamada de red y no local). Type hints +
  docstrings Google en inglés.

### 4.2 El check de moderación
- Llama a la Moderation API de OpenAI (cliente OpenAI ya presente como dependencia del proyecto).
  Usa el modelo de moderación vigente — **verifícalo en la doc de OpenAI antes de cablearlo**, no
  asumas el nombre de un snippet (misma disciplina que "verify model currency" del proyecto).
- Si el resultado viene `flagged=True` → lanza `InputGuardrailError`. Si no, retorna sin efecto.
- **No** loguees el texto completo del input por defecto (superficie de exposición; principio "log
  metadata, not content"). Loguea el evento de moderación (disparó / no disparó, categorías) como
  structlog, sin el texto — coherente con la observabilidad de S03 brief 4.

### 4.3 Integración en el flujo
- `run_input_guards(question)` se invoca en `llm_service` (o en el router, delegando) **antes** de
  `_cache_lookup`, tanto en `answer_query` como en `stream_answer_query` — punto compartido, no
  duplicado por endpoint.
- En `/query`: un fallo se traduce a `HTTPException(400/422)`.
- En `/query/stream`: un fallo se traduce al evento SSE `error` tipado ya existente (brief 3 de S03),
  **antes** de abrir cualquier stream de texto — no se emite ningún `chunk`. Coherente con "el cache
  decide si hay stream": aquí el guardrail decide, aún más arriba, si hay algo que procesar.

### 4.4 Lo que NO cambia
- `build_cache_key` / la clave de cache: sin cambios. La moderación no añade slots.
- El contrato de respuesta en el camino feliz: sin cambios.
- `context/reference.py`, el prompt, las citas: sin cambios.

---

## 5. Sub-decisión de seguridad: fail-open vs fail-closed

La Moderation API es de **OpenAI**, y en este proyecto `OPENAI_API_KEY` es **opcional** (sin ella el
Router arranca solo-Anthropic). Además la API puede fallar o dar timeout. Hay que decidir qué pasa
cuando **no se puede moderar** (sin key, o la llamada falla):

- **fail-open** (si no se puede moderar, se deja pasar): no bloquea el desarrollo solo-Anthropic ni
  tumba el servicio si OpenAI está caído; coste: en esos momentos no hay moderación.
- **fail-closed** (si no se puede moderar, se rechaza): garantiza que nada pasa sin moderar; coste:
  un despliegue solo-Anthropic o un fallo de OpenAI deja el servicio **inservible**.

**Recomendación: fail-open con warning explícito y observable** (mismo patrón que Redis degradable y
`OPENAI_API_KEY` opcional del proyecto: una optimización/capa de seguridad que degrada con log
explícito, no un fail-fast que rompe el servicio). Loguea `moderation_unavailable` (nivel warning)
cuando se degrade, para que sea visible en la traza. Si CC lo ve de otro modo, que lo marque
`# TODO: pendiente de decisión en Claude Chat` en vez de decidir en silencio.

> **Deuda registrada:** fail-open significa que la moderación es una garantía **condicional a que
> OpenAI esté disponible**. Aceptable para una demo; un despliegue a cliente real con requisito duro
> de moderación necesitaría fail-closed o un proveedor de moderación independiente del LLM. Anótalo,
> no lo implementes.

---

## 6. Tests

Levanta Docker y prueba la imagen ANTES de la suite. No rompas la fixture `autouse` de
`tests/app/conftest.py`.

### Sin API (en CI, mockeando la Moderation API)
- **Orden:** con un mock de la Moderation API que devuelve `flagged=True`, una request **no** llega
  al cache lookup ni al LLM — se rechaza antes. Verifícalo (el mock de `_call_model` / cache no se
  invoca).
- **Guardrail antes del hit:** precachea una respuesta para una pregunta; luego manda un input que el
  mock de moderación marca como tóxico; verifica que se rechaza aunque hubiera dado hit — el guardrail
  corre antes del lookup. **Este es el test que documenta el invariante** (§2).
- **Camino feliz:** input limpio (mock `flagged=False`) → sigue al cache/LLM como hoy, sin regresión.
- **Política exception:** un flag tóxico produce `400`/`422` en `/query` y `event: error` en
  `/query/stream`, sin `chunk`.
- **Degradación (§5):** si la Moderation API lanza (o no hay key), el comportamiento es el decidido
  (fail-open con warning) — testéalo mockeando el fallo.

### `real_llm` (marcado, fuera de la suite por defecto)
- Opcional: un test que pegue a la Moderation API real con un input claramente tóxico y uno limpio,
  para confirmar el cableado end-to-end. Marcado `real_llm`, no en la suite por defecto.

---

## 7. Tests de usuario (contra Docker)
- Input tóxico evidente → `400`/`422` con mensaje claro; no se llama al LLM.
- Input limpio repetido → sigue dando hit (con la latencia añadida de la moderación, esperada).
- Streamlit: un input tóxico muestra el error, no abre stream.

---

## 8. Docs a actualizar
- **`S04-decisions-3-...md`** (lo escribes tú, sube al chat): el orden y su porqué (el guardrail no
  depende del cache — invariante, no optimización), la honestidad toxicidad≠injection, la decisión
  de no cachear el veredicto (y su coste en latencia de hit), la sub-decisión fail-open/closed, y el
  hueco de injection.
- **`ARCHITECTURE.md`**: nuevo pipeline de input-guardrails antes del cache lookup; su lugar en el
  flujo de request (§3 del ARCHITECTURE); nota del acoplamiento a OpenAI para moderación.
- **`CLAUDE.md`**: reflejar `input_guards.py` en la estructura; el pipeline y su política; el hueco
  de injection en "Decisiones pendientes".
- **`CHANGELOG.md`**: entrada `feat`.

---

## 9. Nota de proceso (para el chat, no para CC)

Este brief **reabre S04, que ya estaba consolidada y graduada** — es una excepción al orden 5→6.
Al cerrarlo, el chat debe **backfillear**:
- **Addendum** al `S04-consolidado.md` (no reescribir el existente): la decisión de moderación, su
  aprendizaje agnóstico (la Moderation API es una llamada de red, no local, y no ahorra llamadas sino
  que añade una; cubre toxicidad, no injection; el orden input-guardrails-antes-del-cache es
  invariante de seguridad, no eficiencia), y la sub-decisión fail-open.
- **Ajuste a la Guide (Phase 5, nodo de guardrail tooling):** hoy dice que la moderación es
  "near-irrelevant for a normative technical corpus". Esta decisión lo corrige: la relevancia no la
  decide el dominio del corpus sino la **superficie de input abierto** — cualquier endpoint público
  que acepta texto de usuario justifica moderar el input, con independencia de la temática. Y añadir,
  en el nodo de guardrails↔cache, que el "input antes del lookup" es invariante que **no debe
  depender del algoritmo de cache** (se rompe con caché semántica si se coloca después del hit).

---

## 10. Criterios de cierre
- [ ] Pipeline de input-guardrails en un punto compartido, **antes** del cache lookup, con hueco de
      injection comentado.
- [ ] Moderación de toxicidad (OpenAI), política `exception` → `400`/`422` (`/query`) y `event:
      error` (`/query/stream`), sin `chunk`.
- [ ] Test del invariante: input tóxico que habría dado hit se rechaza igualmente.
- [ ] Camino feliz sin regresión; veredicto de moderación no cacheado (por diseño).
- [ ] Degradación fail-open con warning observable (o `# TODO` si CC discrepa).
- [ ] Modelo de moderación vigente verificado en la doc de OpenAI, no asumido de un snippet.
- [ ] Docker levanta; `/health` responde; `uv run pytest` verde.
- [ ] Docs actualizadas; commit lo hace el usuario.
- [ ] (Chat) Backfill del consolidado + ajuste de la Guide tras el cierre (§9).
