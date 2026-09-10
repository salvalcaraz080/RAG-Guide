# Brief — Auditoría de código pre-M2 (doble-check + auditoría independiente)

**Tipo de sesión:** solo lectura. Esta sesión NO modifica código, NO refactoriza, NO corrige
nada de lo que encuentre — ni siquiera lo trivial. Produce un único entregable:
`AUDIT-PRE-M2.md` en la raíz del repo. Toda decisión que un hallazgo sugiera se marca como
`pendiente de decisión en Claude Chat`, según el flujo de CLAUDE.md.

**Contexto:** se ha completado una auditoría arquitectónica sobre la documentación
(RAG-GUIDE.md, CLAUDE.md, ARCHITECTURE.md) sin acceso al código. Esta sesión tiene dos
mandatos: (A) verificar contra el código real cada afirmación de esa auditoría — la
documentación puede divergir de la implementación en ambos sentidos; (B) auditar de forma
independiente lo que solo el código puede revelar. Decisión ya tomada que condiciona la
lectura: **la cache exact-match se aparca en M2** (no se configura Redis, sale del camino
caliente; el código no se borra). No auditar la cache como si fuera a adaptarse — auditarla
como artefacto que debe quedar aparcado limpiamente.

**Formato del entregable:** por cada punto, veredicto explícito —
`CONFIRMADO` / `REFUTADO` / `MATIZADO` (explicar el matiz) / `NO VERIFICABLE` (explicar por
qué) — con evidencia `archivo:línea` o nombre de test. Para la Parte B: hallazgos con
severidad (alta/media/baja), evidencia, y consecuencia concreta (no "esto es mejorable" sino
qué se rompe o qué cuesta). Sin relleno: si un área está limpia, una línea que lo diga.

---

## Parte A — Doble-check de los hallazgos de la auditoría documental

### A1. Estado actual

- **A1.1 — Recreación de sesión en el frontend.** La auditoría afirma: ante `event: error`
  de "session not found", `frontend/streamlit_app.py` descarta el `session_id` y recrea
  sesión en la siguiente pregunta, potencialmente sin avisar al usuario de que el contexto
  se perdió. Verificar en el código del frontend: ¿se muestra al usuario un aviso explícito
  y visible de que la conversación se ha reiniciado, o la recuperación es silenciosa?
  Transcribir el fragmento exacto del manejo de ese error.
- **A1.2 — `LLM_PROVIDER` vestigial.** Verificar con grep exhaustivo: ¿algún código fuera de
  `config.py` lee `settings.LLM_PROVIDER`? ¿Tiene algún efecto en el Router de
  `llm_wrapper.py` o en cualquier rama? Listar todos los puntos de uso. Si es config muerta,
  confirmarlo; si tiene un efecto no documentado en ARCHITECTURE.md, describirlo.
- **A1.3 — Flujo real de una petición.** Reconstruir desde el código (no desde la doc) el
  orden real de `answer_query` y `stream_answer_query`: guards → resolución de sesión →
  compuerta de cache → lookup → prompt → LLM → citas → set/append. Confirmar que coincide
  con los 12 pasos de ARCHITECTURE.md §3 o listar cada divergencia. Atención especial:
  ¿los guards corren de verdad antes de `_resolve_session` (es decir, antes de CUALQUIER
  efecto, como exige la versión afilada del invariante), o solo antes del cache lookup?
- **A1.4 — Aislamiento de tests.** Confirmar que las tres fixtures autouse de
  `tests/app/conftest.py` (`isolate_cache_from_real_redis`,
  `isolate_input_guards_from_real_moderation`, `isolate_session_store`) existen, son
  autouse, y cubren lo que dicen cubrir. Verificar además que ningún test las desactiva
  ni las esquiva (override de fixture, monkeypatch posterior).

### A2. Afirmaciones sobre invariantes que la doc da por implementados

- **A2.1 — Citas: construcción y supresión en rechazo.** Localizar `_citations` /
  `_refusal_kind`. Confirmar: (a) las citas se construyen desde `ECSS_REFERENCE`, nunca
  parseando la respuesta del modelo; (b) un rechazo detectado devuelve `[]`; (c) la
  detección es `startswith` case-sensitive sobre exactamente las dos frases marcador que
  `system.md` instruye — comparar byte a byte las frases del código contra las del prompt
  (una divergencia de una tilde o mayúscula rompe la detección en silencio); (d) el punto
  de ensamblado es realmente compartido por `/query` y `/query/stream` (mismo código, no
  dos copias).
- **A2.2 — Compuerta de cache.** Confirmar que `_cache_eligible(history)` es literalmente
  `len(history) == 0 and not settings.LLM_EVAL_MODE`, que es el ÚNICO sitio que decide
  elegibilidad (grep de accesos a `cache.get`/`cache.set` fuera de ese camino), y que en un
  hit de turno 1 el turno se anexa a la sesión con `hit.answer`.
- **A2.3 — Fidelidad por envoltura.** Confirmar que `GET /api/v1/prompt` devuelve la misma
  constante que compone el prompt real (no relee el archivo), y que `injected_context` se
  puebla desde el MISMO string usado para el prompt de esa request (no una segunda
  derivación). Buscar cualquier segunda derivación del contexto o del prompt en el repo.
- **A2.4 — Wrapper y fallback.** Confirmar: `llm_service` no importa litellm; solo
  `_call_model` cruza el seam; se usa `router.acompletion` (nunca `.completion`); la
  derivación de proveedor usa `litellm.get_llm_provider()` y no heurística de substring;
  `stream_options={"include_usage": True}` está en el camino de streaming.
- **A2.5 — Test-cemento en el punto crítico.** La Guide documenta el patrón "test que
  asertaba la fabricación como esperada". Revisar los tests actuales de citación y rechazo:
  ¿asertan el comportamiento correcto (rechazo → citations == []) o hay algún test que
  asertó comportamiento vigente sin cuestionarlo? Listar los tests que protegen el contrato
  de citación por nombre y qué asertan exactamente.

### A3. Verificación de los riesgos M1→M2 contra el código real

- **A3.1 — Puntos de contacto del swap de contexto.** Enumerar TODOS los sitios del código
  que conocen `ECSS_REFERENCE` o `context/reference.py` (import, uso, tests). El plan dice
  que M2 solo toca `llm_service`; verificar si hay acoplamientos no documentados (¿algún
  test, el frontend, `build_cache_key`, el endpoint de prompt?).
- **A3.2 — Estado real de los placeholders M2.** ¿`services/ingestion.py`, `embedding.py`,
  `retrieval.py`, `routers/ingestion.py`, `schemas/document.py`, `models/` existen? ¿Vacíos,
  con esqueleto, con código muerto? ¿Algo importa de ellos ya? Un placeholder con firma
  equivocada condiciona M2 más que ninguno.
- **A3.3 — Multi-turn: qué llega exactamente al modelo.** Confirmar en `_compose_messages`
  el shape real `[system, *history, current_user]` y que los `Message` del historial son
  solo `{role, content}` (sin citas ni metadata). Relevante para la decisión pendiente de
  condensación: verificar qué tendría disponible `retrieval.py` en el punto donde se
  insertará (¿recibe la pregunta cruda, tiene acceso al historial resuelto?).
- **A3.4 — Grafo y corpus.** Ejecutar SOLO los verificadores existentes de lectura
  (`python -m corpus.graph` si es idempotente de lectura, `verify_parser` si no toca red ni
  reescribe artefactos — si regeneran archivos, NO ejecutarlos y decirlo). Confirmar que las
  stats documentadas (273 referencias, 326 nodos, 247 aristas, 14 JSON) coinciden con
  `data/processed/` actual. Verificar que `references.json` tiene el shape que el contrato
  de runtime de M2 asume (keyed por `(source, target)`, con `mention_text`/`mention_type`/
  `source_uid`).
- **A3.5 — Tablas omitidas: dimensionar la deuda.** Sin arreglar nada: ¿el parser deja
  alguna traza de las tablas que omite (conteo, warning, campo en el JSON)? Si es posible con
  un script de solo lectura en `scripts/`, estimar cuántas tablas/secciones-con-tabla hay en
  los 14 documentos procesados, para dimensionar R2.6 con un número en vez de una intuición.

---

## Parte B — Auditoría independiente (lo que la doc no puede mostrar)

Buscar activamente; no limitarse a confirmar que "parece estar bien". Áreas, por prioridad:

- **B1 — Corrección async.** La Guide la marca como failure mode, no preferencia. Grep de:
  llamadas síncronas bloqueantes dentro de `async def` (SDK síncrono, `requests`, I/O de
  archivo pesado, `time.sleep`); iteración síncrona de streams; uso del cliente de
  moderación (¿`AsyncOpenAI` de verdad?). El singleton de módulo del Router y del loader de
  prompts: ¿algo bloqueante en import time que penalice el arranque o un worker?
- **B2 — Manejo de errores real vs declarado.** Grep de `except` desnudos, `except: pass`,
  excepciones capturadas y logueadas pero con retorno ambiguo. Especial atención a los tres
  caminos degradables (Redis, moderación, fallback): ¿la degradación devuelve un estado
  inequívoco o hay valores centinela (`None`, string vacío) que un llamador podría
  malinterpretar? ¿`SessionNotFoundError` puede escaparse por algún camino como 500 en vez
  de 404/error tipado?
- **B3 — Estado global y concurrencia.** Inventariar todos los singletons de módulo (Router,
  LLMCache, session_store, cliente de moderación, prompt cargado). Con un solo worker pero
  event loop concurrente: ¿`append_turn` sobre el dict de sesiones es seguro ante dos
  requests concurrentes de la misma sesión (interleaving de lectura-modificación)? ¿La
  ventana deslizante muta en sitio o reasigna (nota de la Guide sobre re-validación
  Pydantic)? ¿Alguna cardinalidad implícita tipo `-2N`?
- **B4 — Divergencias config/doc.** Diff de tres vías: campos de `Settings` en `config.py`
  vs `.env.example` vs la sección de variables de CLAUDE.md. Listar: variables en un sitio y
  no en otro, defaults que difieren, y cualquier `Settings` field que ningún código lea
  (config muerta más allá de A1.2).
- **B5 — Calidad de la suite de contrato.** (a) ¿Hay tests que dependan de qué proveedor
  respondió (violando el invariante declarado)? (b) ¿Hay asserts débiles (`len > 0`,
  `is not None`) donde el contrato promete propiedades concretas? (c) ¿Los caminos de error
  de streaming (`event: error` por toxicidad, por sesión ausente, por fallo de proveedor a
  mitad de stream) tienen test de que el CLIENTE puede leer el error, según la lección del
  bug del cache-hit sin canal? (d) ¿Cobertura del camino degradado de Redis y de moderación
  caída, o solo del camino feliz?
- **B6 — Consistencia del prompt como artefacto.** Comparar `app/prompts/query/system.md`
  contra lo que ARCHITECTURE.md dice que contiene (los 5 puntos + frases marcador). ¿Existe
  el snapshot de caracterización del prompt compuesto que la Guide describe como patrón?
  ¿Hay algún resto de las instrucciones como constante en código (la versión pre-S04) que
  haya quedado muerto?
- **B7 — Higiene general con ojo de revisor externo.** Código muerto o inalcanzable; TODOs
  sin la etiqueta de decisión pendiente; imports del backend en `frontend/` (debería ser
  imposible por imagen, pero verificar el código fuente, no solo el Dockerfile); comentarios
  que describen comportamiento que ya no es cierto (deuda de la regla presente-vs-histórico
  filtrándose al código); logs que violen "metadata, not content" (¿algún log emite la
  pregunta o la respuesta completa fuera de nivel DEBUG?).
- **B8 — Preparación real del aparcamiento de la cache.** Con la decisión tomada: ¿qué pasa
  hoy, en código, si `REDIS_URL` no está definida (no solo si Redis no responde)? ¿Arranca
  limpio con warning único, o hay ruido por request? ¿Algún test asume Redis configurado?
  Listar qué haría falta tocar (sin tocarlo) para que "cache aparcada" sea un estado
  soportado de primera clase y no una degradación permanente ruidosa.
- **B9 — Oportunidades que solo el código muestra.** Duplicaciones entre `answer_query` y
  `stream_answer_query` que el swap de M2 obligaría a tocar dos veces; funciones de
  `llm_service` que ya hacen demasiadas cosas y donde `retrieval.py` va a insertar presión;
  cualquier seam adicional barato de dejar preparado ANTES de M2 (nombrarlo como propuesta
  con trade-off, no implementarlo).

---

## Cierre de la sesión

1. `AUDIT-PRE-M2.md` en la raíz: Parte A con veredictos + evidencia, Parte B con hallazgos
   por severidad. Al final, una tabla-resumen de máximo 15 filas: hallazgo, severidad,
   ¿bloquea M2?, ¿requiere decisión en Claude Chat?
2. Ningún cambio en el árbol de trabajo aparte del propio informe (confirmar con
   `git status` al cerrar y pegarlo al final del informe).
3. NO actualizar CLAUDE.md ni CHANGELOG.md en esta sesión: el informe se revisa primero en
   Claude Chat y las decisiones que salgan seguirán el flujo normal.
4. Guion de UAT: no aplica (sesión de solo lectura); en su lugar, listar los comandos de
   verificación ejecutados y su salida resumida.
