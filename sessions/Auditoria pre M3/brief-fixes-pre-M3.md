# Brief — Sesión de fixes pre-M2

**Tipo de sesión:** modificación de código. Alcance cerrado: solo los puntos de este brief.
Ningún cambio toca `corpus/`, `data/processed/`, ni el diseño de retrieval — eso es M2. Si
un fix parece invitar a "ya que estoy" tocar corpus o retrieval, PARAR y anotarlo, no hacerlo.

**Origen:** hallazgos de `AUDIT-PRE-M2.md` (auditoría de código pre-M2). Cada punto cita el
hallazgo que lo motiva. **Decisión ya tomada aguas arriba: la feature de cache y la dependencia
de Redis se ELIMINAN por completo** (no se aparcan). El bloque 1 de este brief es ese borrado.

**Regla de cierre por cada fix:** la suite verde es el testigo. Ningún fix se da por bueno si
`uv run pytest` no queda en verde. Ojo: el borrado de la cache CAMBIA la línea base de conteo
(desaparecen tests de cache y degradación) — eso es esperado y se documenta en el cierre, no es
una regresión. Los fixes que cambian comportamiento de test esperado (snapshot, `LLM_PROVIDER`)
actualizan el test en el mismo commit lógico y explican por qué. El commit final lo hace el
usuario, no esta sesión (flujo de CLAUDE.md).

**Orden de ejecución (importa):** 0 → 1 → 2 → 3 → 4. El borrado de cache (1) va ANTES del
refactor de seams (4) a propósito: borrar simplifica `answer_query`/`stream_answer_query`
(fuera compuerta y lookup), y el refactor parte de un módulo ya más limpio. Hacerlo al revés
significa tocar el módulo denso dos veces con diffs que pelean entre sí. El paso 0 (Guide) va
primero porque extrae el aprendizaje ANTES de destruir el código que lo generó. Trabajar en
commits lógicos separados por bloque para que el usuario revise y commitee por partes.

---

## 0. Extraer el aprendizaje de la cache a la Guide — ANTES de borrar

**Por qué primero:** el valor de la cache en este proyecto nunca estuvo en los hits (un
usuario, consultas mayoritariamente únicas). Estuvo en lo que forzó: la separación
`_STATIC_INSTRUCTIONS`/`_build_context_block`, el invariante guardrails-antes-de-lookup, el
gate de historial, el modo eval, y varios nodos de la Guide. Ese valor sobrevive al borrado
SOLO si se captura antes. Borrar el código sin este paso pierde el aprendizaje junto con la
feature.

**Fix:** en `RAG-GUIDE.md`, actualizar el nodo de cache (hoy presentado como decisión viva) a
un nodo cerrado que registre la lección — genuinamente agnóstica y validada por este proyecto:

- La cache exact-match es MVP-cheap de construir pero NO de arrastrar: su condición de
  aplicabilidad (hit rate observado >~20%) se re-evalúa en cada cambio de fase, y en un producto
  mono-usuario el único escenario con hit rate real es la demo y la suite de evaluación, no el
  tráfico. Construirla en M1 fue correcto por lo que forzó a separar (contexto estático vs
  instrucciones, orden guardrail→efecto), no por los hits. En la transición a retrieval dinámico
  su relación esfuerzo/beneficio se invirtió y se eliminó.
- Corregir la sobregeneralización del nodo de clave de caché (G4.2 de la auditoría documental):
  "populating the `context` slot is a fill, not a redesign" era cierto para llegar hasta M1,
  pero en la fase RAG el slot habría sido redundante-o-dañino y el orden del flujo se invertía
  (retrieve-then-lookup). Registrar esto como caso de "la Guide convergió desde la evidencia de
  M1 y generalizó más allá de ella" — el patrón que el charter de la Guide existe para vigilar.

**Verificar:** el nodo queda como conocimiento cerrado con su procedencia, no como feature
activa. Tras esto, ningún lector de la Guide creería que hay una cache en el sistema.

**Trade-off:** ninguno; es escribir 2-3 párrafos antes de borrar código. Es el paso que hace
el borrado una decisión de ingeniería y no una pérdida de memoria.

---

## 1. Eliminar la feature de cache y la dependencia de Redis (B8 + decisión aguas arriba)

**Alcance:** eliminación total. No queda compuerta, ni seam, ni vestigio. Redis desaparece del
proyecto. Trabajar de dentro (código de servicio) hacia fuera (infra y tests) para que la suite
guíe el borrado.

**Puntos de eliminación (inventario del informe — verificar cada uno con grep, no asumir):**

Código de servicio:
- `app/services/cache.py` — borrar el módulo entero (`LLMCache`, `build_cache_key`, conexión
  `redis.Redis.from_url`).
- `app/services/llm_service.py`: eliminar el singleton `cache`, `_cache_eligible`,
  `_cache_lookup`, los dos `cache.set` (L453 no-stream, L599 stream), y la rama de HIT completa
  en ambos caminos (`answer_query` L405-415, `stream_answer_query` la equivalente). El flujo
  queda: guards → resolve → contexto → prompt → LLM → assemble → append. Más simple que antes.
- `app/main.py`: quitar `cache.ping()` del lifespan y del `/health` (L20, L43). Decidir qué
  reporta `/health` sin cache: simplemente dejar de reportar el estado de cache (no sustituirlo
  por "disabled" — no hay cache que describir). El health queda sobre lo que exista (proceso,
  y en su caso moderación/router).

Config e infra:
- `app/config.py`: eliminar `REDIS_URL`.
- `.env.example`: eliminar `REDIS_URL`.
- `docker-compose.yml`: eliminar el servicio `redis` y cualquier `depends_on`/red que lo
  referencie.
- Dependencias: quitar `redis` de `pyproject.toml` (y regenerar el lock con `uv lock` — SIN
  instalar nada más, solo reflejar la baja). Verificar que ningún otro módulo importa `redis`.
- CLAUDE.md: eliminar la cache del árbol de servicios, de la sección de variables, y de la
  descripción de arquitectura. Registrar en el CHANGELOG/histórico que la feature se retiró en
  esta fase con puntero al nodo de la Guide (paso 0) para la razón — pero el CHANGELOG lo
  consolida el usuario, así que aquí solo dejar el texto propuesto, no editarlo.

Tests:
- `tests/app/test_llm_service_cache.py` — borrar el fichero entero (incluido el pin
  `test_model_change_does_not_invalidate_cache`; ya no hay clave que pinear).
- `tests/app/conftest.py`: eliminar la fixture `isolate_cache_from_real_redis` y el doble
  `fake_cache`.
- Tests de degradación de Redis: `test_degrades_without_redis`, `test_degrades_when_client_raises`
  y cualquier assert de cache-hit/miss en `test_llm_service.py`, `test_multi_turn.py`
  (`test_turn1_appends_even_on_cache_hit` — el APPEND en turno 1 debe seguir cubierto por un
  test sin cache: el turno 1 se anexa siempre; conservar esa aserción reescrita sin cache),
  `test_streaming.py`, `test_health.py` (el estado de cache en health). Reescribir, no solo
  borrar, los tests cuyo comportamiento asertado sigue siendo válido sin cache (el append de
  turno 1 es el caso claro).

**Cuidado — no tirar cobertura válida con el borrado:** algunos tests de la suite de cache
asertan propiedades que siguen importando sin cache. El más importante: **turno 1 se anexa a la
sesión** (antes se probaba "incluso en cache hit"; ahora es simplemente "turno 1 se anexa").
Antes de borrar cada test, preguntar: ¿esto asertaba algo sobre la cache, o algo sobre el flujo
que la cache no cambiaba? Lo primero se borra; lo segundo se reescribe sin la cache.

**Verificar:** grep final de `cache`, `redis`, `Redis`, `REDIS` en `app/`, `frontend/`,
`tests/`, `pyproject.toml`, `.env.example`, `docker-compose.yml` → cero resultados salvo, si
acaso, menciones históricas en CHANGELOG. `uv run pytest` verde con el nuevo conteo (menor que
271; documentar cuántos tests desaparecen y confirmar que ninguno era cobertura de flujo no-cache
que debía reescribirse). Arranque de la app sin Redis y sin warnings.

**Trade-off:** decidido aguas arriba. La irreversibilidad es conocida: si en producción con
usuarios reales aparece hit rate que justifique cache, se reconstruye desde el nodo de la Guide
(paso 0) — que es exactamente por qué el paso 0 va primero.

---

## 2. Contradicción interna de `system.md` — cita fantasma latente (B6)

**Problema:** `app/prompts/query/system.md` contiene DOS instrucciones de rechazo. La del
bloque pre-S04 (línea ~3) dice al modelo que responda, entre comillas como cita literal, algo
del tipo «The provided reference does not cover this». El bloque S04 (líneas ~11-21) exige que
un rechazo empiece exactamente con el marcador `Not covered by the provided material:`, que es
lo único que `_refusal_kind` detecta. Si el modelo obedece la frase pre-S04, la detección
falla y `_citations` adjunta la cita candidata → la cita fantasma que el fix S04 existe para
evitar. Hoy lo tapan los tests `real_llm` con los dos proveedores cableados, pero es sensible a
cambio de modelo, y el golden snapshot cementa el estado contradictorio.

**Fix:** eliminar del `system.md` el bloque de rechazo pre-S04 redundante (y, si las líneas
1-8 y 9-23 duplican rol/grounding como anota B6, consolidar esa redundancia — sin cambiar la
semántica del marcador ni las 5 reglas). Debe quedar UNA sola instrucción de rechazo: la del
marcador S04. Regenerar `test_system_prompt_snapshot.py` (el snapshot debe reflejar el prompt
limpio, no pinear el contradictorio). Revalidar que los 6 tests `real_llm`
(`test_system_prompt_provider_agnostic.py`) siguen verdes con los dos proveedores — esos son
la red de seguridad de que el modelo sigue emitiendo el marcador correcto tras la limpieza.

**Cuidado:** no tocar las constantes marcador de `llm_service.py` (`_OUT_OF_DOMAIN_MARKER`,
`_OUT_OF_MATERIAL_MARKER`); son correctas y están cementadas byte a byte contra el prompt. El
fix es en el `.md`, no en el detector. Verificar tras el cambio que las frases del `.md`
siguen siendo idénticas byte a byte a esas constantes.

**Trade-off:** ninguno real; es eliminar una instrucción muerta y contradictoria. El único
coste es regenerar el snapshot, que es mecánico.

---

## 3. Menudencias de código (no-corpus, no-doc)

Todas pequeñas, ninguna toca corpus ni decisiones M2. Un commit lógico por punto o agrupadas,
a criterio.

- **3a — Eliminar `LLM_PROVIDER` (A1.2/B4).** Config muerta: nadie la lee (el Router hardcodea
  `anthropic/`/`openai/` en `model_list`). Quitarla de `Settings` (`config.py`),
  `.env.example` y CLAUDE.md. Verificar con grep que no queda ninguna referencia. (Si el
  usuario prefiere cablearla en vez de eliminarla, eso es cambio de comportamiento y sale de
  esta sesión; por defecto, eliminar.)
- **3b — `StreamErrorEvent` sin uso (B7).** Definido en `schemas/query.py`, cero usos; el
  router emite el evento `error` como dict crudo mientras `done` sí se valida contra
  `StreamDoneEvent`. Cerrar la asimetría: validar el evento `error` contra `StreamErrorEvent`
  en `routers/queries.py` (preferido — simetría de validación). Si al hacerlo aparece fricción
  con el shape actual del error, la alternativa es borrar el schema muerto; elegir la primera
  salvo obstáculo real.
- **3c — Suite de contrato depende de `OPENAI_API_KEY` (B5).** Sin la key, 3 tests de
  `test_llm_wrapper.py` fallan porque el Router arranca sin deployment de fallback y la fixture
  `mock_deployments` lo exige. Contradice la soberanía de la suite. Fix preferido: inyectar el
  deployment de fallback en la fixture (con `mock_response`) de modo que el mecanismo de
  fallback se teste SIEMPRE, con o sin key en el entorno. Alternativa más barata:
  `pytest.mark.skipif` sobre la presencia de la key (pero deja el fallback sin testear en
  clones sin key, peor). Elegir la inyección salvo que resulte desproporcionada.
- **3d — Frontend: aviso de contexto perdido + detección robusta (A1.1).** Dos cosas en
  `frontend/streamlit_app.py`: (i) cuando se recrea sesión tras "session not found", mostrar un
  aviso explícito de que la conversación se reinició y la siguiente pregunta empieza sin
  contexto (hoy solo muestra el error genérico del backend, y el chat pintado sugiere una
  continuidad que el servidor ya no tiene); (ii) la detección por substring `"session not
  found"` acopla el frontend al texto del mensaje del backend — si se puede, discriminar por un
  campo estructurado del `event: error` (código/tipo) en vez de por substring. Si el evento de
  error no lleva hoy un campo tipo, dejar (ii) anotado como deuda menor y hacer solo (i); no
  añadir un campo al contrato del backend en esta sesión sin decidirlo.

---

## 4. Seams de contexto y citación para M2 (B9) — refactor con red, DESPUÉS del borrado

Los seams que M2 va a tocar están hoy duplicados o acoplados al global. Prepararlos ahora, con
la suite verde de testigo, para que M2 inserte retrieval en UN punto. Va después del borrado de
la cache (bloque 1) porque ese borrado ya simplifica los dos caminos; refactorizar sobre el
módulo limpio es más seguro. Es refactor puro: **el comportamiento no cambia, la suite debe
seguir verde sin editar asserts** (salvo los que asertan la firma interna que cambia).

**4a — Seam de contexto único.** Tras el borrado, `answer_query` y `stream_answer_query`
todavía construyen el contexto por separado (`_build_context_block(ECSS_REFERENCE)` en cada
camino). Extraer a UNA función `get_context(question, history) -> fragments` que ambos consuman
(hoy devuelve el bloque estático desde `ECSS_REFERENCE`; en M2 será el seam de retrieval).
Opcionalmente extraer el resto del preludio compartido si queda natural; no forzar la fusión de
los dos caminos (raise vs yield es diferencia real).

**4b — `_citations` recibe los fragmentos de la request (B9.2).** Cambiar la firma de
`_citations()` (hoy deriva del global `ECSS_REFERENCE`, L277) a `_citations(text, fragments)`,
donde `fragments` es lo que `get_context` devolvió para ESA request. En M1 el resultado es
idéntico, así que la suite de citación no cambia de veredicto — es el testigo de comportamiento
nulo. En M2 ese parámetro permite que las citas salgan de los chunks recuperados.

**Nota deliberada:** este refactor deja la FIRMA lista pero NO implementa el filtro por mención
de cláusula (fan-out de citas) — decisión de diseño de M2 pendiente en Claude Chat. Aquí solo
se cambia de dónde vienen los `fragments`, no cómo se filtran.

**Verificar:** `uv run pytest` verde sin relajar ningún assert de citación, orden o compuerta
(la compuerta de cache ya no existe; el resto sigue). Ajustar los tests que importaban
`ECSS_REFERENCE` al nuevo seam es parte del refactor; el veredicto que asertan no cambia.

**Trade-off:** una indirección más a cambio de que M2 toque un punto y no dos. Si el helper del
preludio completo queda más enrevesado que los dos caminos, quedarse en el seam mínimo
(`get_context` + `_citations(text, fragments)`) y dejar el resto duplicado — 80% del valor, 20%
del riesgo.

---

## 5. Correcciones de doc (batch — confirmar inclusión con el usuario)

Triviales y sin dependencia de M2. Ninguna roza el corpus ni retrieval, así que son seguras; se
incluyen salvo que el usuario prefiera aparcarlas. NO son cambios de código.

- **5a — Placeholders M2 inexistentes (A3.2):** el árbol de CLAUDE.md dibuja
  `services/ingestion.py`, `embedding.py`, `retrieval.py`, etc. como presentes. No existen.
  Marcarlos como "(M2 — no creado aún)" o retirarlos del árbol.
- **5b — Seam de litellm (A2.4):** ARCHITECTURE.md §3 afirma que `llm_service` nunca importa
  litellm; sí lo hace para `cost_per_token`. Corregir a "el seam de completions se respeta; el
  cálculo de coste importa litellm directamente".
- **5c — ARCHITECTURE.md §4 (B7):** dice que el backend no expone prompt/contexto; falso desde
  S04 b2. Actualizar.
- **5d — `references.json` shape (A3.4):** la doc lo describe como dict keyed por
  `(source, target)`; es una lista de registros. Corregir la descripción.
- **5e — Etiquetas de estado (P1.3):** las tablas de módulos marcan M1 como "✅ (S02)" cuando
  M1 abarca S03-S05. Actualizar el puntero.
- **5f — `pyproject.toml` (B4):** la descripción dice "AI-powered software estimation",
  heredada del proyecto hermano. Corregir al proyecto real (RAG sobre ECSS).

---

## 6. Deuda anotada para M2 — NO se toca en esta sesión

Registrar estos tres puntos donde alimenten la decisión de diseño de M2 (idealmente en el
brief de diseño de M2, no como `# TODO` suelto en código). Son **input de decisión**, no fixes:

- **D1 — Tablas omitidas silenciosamente (A3.5).** El parser no deja traza de las tablas que
  descarta. 569 tablas en total, los 14 documentos tienen al menos una, ECSS-E-ST-70-41C
  concentra 389 (es esencialmente tablas: definiciones de paquetes TM/TC). Consecuencia M2:
  indexar sin parsing de tablas produce documentos presentes-pero-vacíos y un rechazo "no
  cubierto" que miente. Decisión pendiente: política de tablas (parsear vs excluir documentos
  tabla-pesados vs marcar secciones con tablas no indexadas) + si el parser debe dejar traza.
- **D2 — 49% de referencias sin `target_clause_id` (A3.4).** 135 de 273 referencias apuntan a
  documento entero, sin cláusula. La expansión por grafo a 1 salto solo aterriza en cláusula
  concreta en la mitad de los casos. Decisión pendiente: qué se inyecta cuando el target es un
  documento completo. Alimenta la política de expansión y la de chunking.
- **D3 — Shape de `references.json` (A3.4).** Es una lista de registros, no un dict keyed por
  `(source, target)`. El builder de M2 debe saber que parte de una lista. Nota de
  implementación, no deuda de corrección.

---

## Cierre de la sesión

1. Suite verde (`uv run pytest`) tras cada bloque. El conteo base BAJA por el borrado de la
   cache — el mensaje de cierre lista cuántos tests desaparecieron, cuáles se reescribieron sin
   cache (el append de turno 1 como mínimo), y confirma que ninguna cobertura de flujo no-cache
   se perdió en el proceso.
2. NO commitear (flujo de CLAUDE.md: el commit lo hace el usuario). Dejar el trabajo en el
   árbol, organizado por bloques lógicos, y pegar `git status` + resumen de qué toca cada bloque.
3. NO editar CHANGELOG.md (se consolida con el usuario); dejar el texto propuesto de la entrada
   de retirada de la cache en el mensaje de cierre.
4. Nada de `corpus/`, `data/processed/`, ni creación de archivos M2. Si algún fix pareció
   requerirlo, es señal de fix mal delimitado: parar y anotarlo en la sección 6, no ejecutarlo.
5. Guion de UAT para el usuario: arrancar la app sin Redis (que ya no debería ni mencionarse) y
   confirmar arranque limpio sin warnings; y el aviso de contexto perdido en el frontend — los
   cambios con efecto visible fuera de la suite.
6. Verificación específica del borrado: pegar la salida del grep final de
   `cache|redis|Redis|REDIS` sobre todo el repo, para que el usuario confirme de un vistazo que
   no queda vestigio.
