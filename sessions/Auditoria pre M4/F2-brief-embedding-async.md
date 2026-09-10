# Fix-brief F2 — Embedding async nativo + aislamiento del cliente en tests

**Segundo de cuatro** (F1 ✓ → **F2** → F3 → F4). Cierra **T1-2** (la llamada de embedding de la query
es bloqueante dentro de `async def search`; 10,5× medido) y **T1-4** (`embedding._client` es el único
global de módulo sin fixture de aislamiento).

**Insumo:** `pasoD-diagnostic-report.md`, T1-2 y T1-4.

**Naturaleza:** fix-brief. CC implementa, verifica, escribe decisiones + CHANGELOG + estado. **No**
re-embed: la salida del batch de embedding debe quedar **byte-idéntica** (mismo modelo, mismas
entradas → `input_fingerprint` sigue `ab3fce8b5cb1a0d4`). No commitea.

**Ficheros:** `app/services/embedding.py`, `app/services/retrieval.py`, `tests/app/conftest.py`,
`tests/app/` (test de comportamiento async), y `corpus/embed_all.py` **solo si** el camino batch lo
exige. Docs: `.claude/CLAUDE.md`, `CHANGELOG.md`, `sessions/Auditoria pre M4/F2-decisions.md`.

---

## Tarea 1 — El camino de petición no bloquea el loop (T1-2)

**Qué es.** `retrieval.search` (async) llama a `embed_query`, que usa un cliente `OpenAI` **síncrono**
(`embedding.py:122`). El loop queda bloqueado toda la ida y vuelta del embedding —la componente
dominante de la latencia de `/search` según `ARCHITECTURE.md`—, así que la operación más lenta del
endpoint serializa a todas las demás. Es violación de un principio graduado escrito en el propio
código (`llm_wrapper.py:136`: «hard requirement, not optional»), no de estilo.

**Dirección (decidida en el chat):** conformar el seam de embedding al patrón **async nativo** que ya
usan los otros dos seams del proyecto — `input_guards` (`AsyncOpenAI`) y `llm_wrapper` (LiteLLM
async). `embedding` es hoy el único síncrono; nativo cierra esa inconsistencia de raíz (es la misma
que T1-4 señala desde el lado de los tests). En concreto:

- El cliente del **camino de petición** (`embed_query` → `retrieval.search`) pasa a `AsyncOpenAI`, y
  `retrieval.search` lo **`await`**-ea.
- El **camino batch** (`corpus/embed_all.py`) **no debe regresionar**. Cómo se estructure es decisión
  de CC contra el código real: un cliente async compartido invocado desde el batch vía `asyncio.run`,
  o mantener el batch síncrono aparte. Lo que no vale es que el batch cambie su salida.
- **Fallback explícito, solo si** la reestructuración nativa resulta invasiva contra el código real:
  `asyncio.to_thread` alrededor de la llamada síncrona (el fallback que la propia Guide admite). Si se
  usa el fallback en vez de lo nativo, **reportar por qué** — la preferencia es nativo.

**Verificación — la sonda real, no el mock.** El test actual mockea la frontera con un fake instantáneo:
ejercita la corrección **funcional** del camino async y **nunca su comportamiento async**. Es
literalmente el caso de la Guide: *«un test que mockea la frontera no prueba el cableado hasta la
frontera»*. Por eso:

- **Reproducir la sonda del informe:** disparar `/health` ~150 ms después de lanzar un `/search`.
  Antes: 22,7 ms a solas → 237,4 ms bajo `/search` (**10,5×**). Después: `/health` bajo `/search`
  debe quedar **cerca del baseline**, no multiplicado. Reportar los tres números.
- **Añadir un test que falle contra la implementación bloqueante** (regresión que muerde): inyectar un
  fake async que registre que fue **`await`**-eado / que el loop siguió libre — no un fake instantáneo
  que solo comprueba que el valor de retorno es correcto. El test tiene que ejercitar el
  comportamiento async, no solo la corrección funcional.
- **El fingerprint del batch no cambia:** `embed_all` produce los mismos vectores → `ab3fce8b5cb1a0d4`.
  Si cambia, algo tocó la identidad y hay que parar.

---

## Tarea 2 — Aislar `embedding._client` en la suite (T1-4)

**Qué es.** El proyecto ya cerró este patrón **dos veces** con fixtures `autouse` documentadas en
`CLAUDE.md` § Tests — `input_guards._moderation_client` y `sessions.session_store`. `embedding._client`
es el tercer global y el único sin fixture. La invariante «la suite de contrato no hace llamadas
reales» se sostiene hoy solo porque **todos los tests recuerdan pasar `client=`**, y `OPENAI_API_KEY`
está en `.env`: el primer test que olvide inyectar construye un cliente real y hace una llamada real
—pasaría, solo más lento y con coste—. Es la forma exacta que la Guide señala: *una invariante que
descansa sobre una propiedad accidental de otro subsistema*.

**Qué implementar.** Fixture `autouse` que aísle `embedding._client` (y el cliente async que añada la
Tarea 1), simétrica a las dos existentes. La forma correcta no es solo inyectar un fake por defecto,
sino que **un cliente NO inyectado falle de forma ruidosa** (o se construya como fake), de modo que un
test futuro olvidadizo se **cace**, no que pase más lento. Documentarla en `CLAUDE.md` § Tests como la
tercera de la familia.

**Verificación.** Un test que llame a `search()` / `embed_texts()` **sin** inyectar cliente **no**
hace llamada real (la fixture lo impide o lo hace fallar). Los ~6 puntos de llamada que hoy inyectan
siguen verdes.

---

## Fuera de alcance

- **No** re-embed, **no** cambiar el modelo de embedding, **no** tocar `chunks.json`/`embeddings.npy`.
  El fingerprint del batch es la prueba de que no se tocó identidad.
- **No** tocar la lógica de retrieval más allá del sitio de la llamada al embedding, ni generación.
- **No** entrar en el refactor de herencia de `Settings` (T2-3): F2 cierra solo la mitad de T1-4 (la
  fixture); la unificación de las dos fuentes de settings sigue diferida como juicio.

## Docs (CC al cerrar)

- `CLAUDE.md`: `embedding` es ahora seam async nativo (o, si fallback, `to_thread` y por qué);
  fixture de aislamiento documentada como la tercera de la familia en § Tests.
- `CHANGELOG.md`: entrada de F2.
- `F2-decisions.md`: log crudo etiquetado agnóstico/dominio; en particular, **qué mecanismo async se
  eligió y por qué** (nativo o fallback), y la medición antes/después de la sonda `/health`.
