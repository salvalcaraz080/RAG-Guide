# S05 — Consolidado de sesión

> Resumen + aprendizajes de implementar. Histórico, no se reescribe. Es la fuente desde la que
> se gradúa `RAG-GUIDE.md` (paso 6). El `S05-resumen.md` (pre-implementación) NO se borra: la
> divergencia teoría-vs-implementación es material de Guide y se recoge aquí.
>
> Estructura: §1 teoría destilada (referencia al resumen, no se repite entera) · §2 lo que se
> IMPLEMENTÓ y su estado · §3 decisiones que se mantuvieron del resumen al código · §4
> aprendizajes NUEVOS de implementar (divergencias, detalles que solo aparecen al escribir el
> código) · §5 nodos nuevos surgidos en revisión · §6 anclajes y diferidos · §7 hilo conductor ·
> §8 candidatos a graduar a la Guide.

**Hito consolidado:** S05 completa, dos briefs (multi-turn + modo evaluación). Agrupación por
hito, no por brief, según cadencia acordada.

---

## 1. Teoría de la sesión (destilada — detalle en `S05-resumen.md`)

Cinco bloques: (1.1) contexto dinámico externo y sus tres mecanismos —adjuntos, búsqueda web,
BBDD de negocio— con la regla "input, no programa" y el eje quién-decide/determinismo; (1.2)
historial vs memoria conversacional, y la propiedad de que la memoria sobrevive al truncado;
(1.3) prompts por tier —adaptar la estructura de salida, no el tono—; (1.4) testing por
propiedades con las tres familias (hard/soft/judge) y el golden dataset; (1.5) separación
generación/verificación (Actor-Critic-Boss) y por qué tres roles y no dos.

Para RAG-ECSS, la sesión resultó **teóricamente ancha, prácticamente estrecha**: el único trabajo
implementable fue multi-turn (historial servidor-dueño) + el reencuadre de testing con el modo
evaluación. El resto se graduó a la Guide como nodos con su condición de aplicabilidad, sin tocar
código de M1.

---

## 2. Lo que se implementó (estado tras S05)

**Brief 1 — multi-turn (backend):**
- `app/services/sessions.py` (nuevo): `SessionStore` singleton de módulo sobre `dict` en memoria;
  tipos `Message{role, content}` y `Session{session_id, history, created_at}`; API
  `create()`/`get()`/`append_turn()`; `SessionNotFoundError` tipada.
- `POST /api/v1/sessions` (`app/routers/sessions.py`, nuevo) → `SessionResponse{session_id}`.
- `session_id: str | None` en `QueryRequest`; `/query` y `/query/stream` lo aceptan.
- `llm_service`: resolución de sesión, inyección de historial, compuerta de cache por presencia de
  historial, anexado de turno (incluso en cache HIT), moderación sobre la pregunta actual.
- `llm_wrapper`: `call_llm`/`stream_llm` ganan `history` opcional; helper `_compose_messages`.
- Ventana deslizante (`MAX_HISTORY_TURNS`, default 10) en `append_turn`.
- Tests de contrato (familia 1, mockeados) + fixture `autouse` `isolate_session_store`.
- Frontend: **sin tocar** (backend-only; el cableado Streamlit se aborda por separado).

**Brief 2 — modo evaluación + reencuadre de testing:**
- `LLM_EVAL_MODE: bool = False` en `Settings`; guardarraíl fail-fast
  (`model_validator(mode="after")`) que rechaza arrancar con el modo activo bajo
  `APP_ENV == "production"`.
- Helper único `_cache_eligible(history)` que compone el bypass de multi-turn con el de eval.
- Reencuadre de la regla de tests en `CLAUDE.md` (dos suites soberanas); `.env.example`
  documentado.
- Tests de contrato del guardarraíl y del bypass.

---

## 3. Decisiones que se mantuvieron del resumen al código

Confirmadas en implementación sin cambio (detalle de razonamiento en `S05-resumen.md` §2):

- **Multi-turn = historial puro, no memoria.** No hay extractor, ni `ProjectMetadata`, ni olvido.
- **Servidor dueño del historial**, por la forja de turnos `assistant` (no por "control" genérico).
- **Historial = compuerta de cache, no slot de clave.** `build_cache_key` sin cambios.
- **Compuerta fina** (`len(history) == 0`), no por presencia de sesión: turno 1 de sesión es
  cache-elegible y comparte entrada con la misma pregunta single-turn (preserva valor FAQ).
- **El turno se anexa incluso en cache HIT** de turno 1, con `hit.answer` como texto assistant.
- **`session_id` provisto-pero-ausente → error explícito** (404 / `event: error`), nunca
  single-turn silencioso.
- **Política de fallo = propiedad del caller.** Test de degradabilidad: la ausencia cambia el
  coste → fail-open; cambia la respuesta → fail-loud. Sesión = fail-loud.
- **Activación del modo eval por entorno, no por flag de request** (superficie de ataque; la
  elegibilidad la decide el sistema, no el cliente).
- **Dos suites soberanas**; la regla "sin red, mockeado" se acota a la de contrato, no se afloja.
- **Los tests de contrato no dependen de qué proveedor respondió** (fallback).

---

## 4. Aprendizajes NUEVOS de implementar (divergencias y detalles que solo aparecen en el código)

Esto es el valor que el consolidado añade sobre el resumen: lo que no se sabía hasta escribir el
código.

### 4.1 Mutación de la ventana IN-SITU para no re-validar Pydantic `[agnóstico, con detalle técnico]`

El brief pedía la ventana deslizante pero no CÓMO mutarla. CC eligió `history[:] = history[-2N:]`
(reasignación en sitio del contenido de la lista) en vez de `session.history = ...`. Motivo: una
asignación de atributo en un modelo Pydantic dispara la validación de asignación del modelo entero
en cada turno; la mutación en sitio evita ese coste y mantiene la referencia estable. Aprendizaje
graduable en forma agnóstica: **mutar una colección dentro de un modelo Pydantic en el camino
caliente prefiere la mutación in-place sobre la reasignación de atributo, para no re-validar el
modelo en cada operación.**

### 4.2 Deuda conocida: la ventana asume 2 mensajes por turno `[agnóstico]`

`-2N` codifica el supuesto "un turno = un par (user, assistant)". Es cierto hoy. El día que un
turno incluya un tercer rol (`tool`/`function`, M5), `-2N` cortaría el historial **a mitad de
turno**, en silencio. Deuda identificada en revisión, no en el brief. Acción acordada: `# TODO`
en `append_turn` que nombre **la condición que la dispara** (un turno deja de ser un par), no un
"arreglar algún día":

```
# TODO (M5): la ventana asume 2 mensajes por turno (user+assistant). Si un turno pasa a
# incluir un rol tool/function, `-2N` corta el historial a mitad de turno. Revisar al
# introducir function calling.
```

*(Pendiente de confirmar que el TODO aterrizó en código; si no, entra en el próximo commit que
toque `sessions.py`.)* Lección agnóstica: **una constante que codifica una cardinalidad implícita
(aquí, mensajes-por-turno) es deuda silenciosa; se anota con su condición de disparo, no como nota
vaga.**

### 4.3 El helper `_cache_eligible` como único sitio de decisión `[agnóstico]`

El brief 2 pedía componer el bypass de eval con el de multi-turn "en un solo sitio". CC lo
materializó como:

```python
def _cache_eligible(history: list[Message]) -> bool:
    return len(history) == 0 and not settings.LLM_EVAL_MODE
```

Ambos caminos (`answer_query`, `stream_answer_query`) llaman a este helper. Aprendizaje: **cuando
varias condiciones independientes pueden desactivar una optimización, factorizar la elegibilidad
en un único predicado —no ramas paralelas en cada camino— hace que (a) añadir una condición futura
sea un cambio en un sitio y (b) exista un punto natural donde crezcan las propiedades derivadas**
(el `# TODO` de logging de contenido / sampling del modo eval vive justo ahí).

### 4.4 El guardarraíl valida una RELACIÓN entre dos campos, no un campo `[agnóstico]`

`model_validator(mode="after")` corre después de parsear todos los campos y puede validar la
relación `LLM_EVAL_MODE` × `APP_ENV` — algo que un validador por-campo no ve. Aprendizaje: **un
fail-fast sobre una combinación prohibida de config (no sobre un valor individual) necesita un
validador post-parseo del modelo completo; es la herramienta correcta para "esta config es válida
por campos pero inválida en conjunto".** Mismo espíritu fail-fast que un secreto obligatorio sin
default, elevado a relación entre campos.

### 4.5 "Un buen seam paga dos veces" — confirmado, con instancia concreta `[agnóstico]`

El bypass del modo eval no necesitó mecanismo nuevo: reutilizó el camino "servir sin cache" que ya
existía desde S03 por la degradabilidad de Redis. El cambio real del brief 2 fue minúsculo (una
condición en `_cache_eligible` + el flag + el guardarraíl) precisamente porque el seam ya estaba.
Aprendizaje: **un seam construido para un propósito (tolerancia a fallos) absorbe un segundo
propósito no anticipado (aislar el modelo bajo evaluación) sin rediseño — evidencia de que el seam
estaba en el sitio correcto.**

### 4.6 Verificación de acoplamiento del import, no asunción `[agnóstico]`

El wrapper importa `Message` de `sessions.py`. CC verificó que es un import de un tipo de DATOS
(no de la lógica del store) y que no hay ciclo (`sessions` solo importa `config`). Es la diligencia
"verifica el path, no la presencia" aplicada a un import. Menor, pero es el patrón correcto.

---

## 5. Nodos nuevos surgidos en revisión (no estaban en ningún brief)

### 5.1 Invariante de orden: guardrails antes de tocar el estado, no solo antes del cache `[agnóstico]`

El invariante de S04 era "guardrails antes del cache". Multi-turn introduce una línea nueva
(`session_store.get`) entre moderación y cache, así que el invariante se **extiende**: la
moderación corre antes de **cualquier** trabajo con estado (antes de resolver la sesión, antes del
cache). Motivo: un `session_id` malicioso o malformado no debe disparar trabajo antes de que la
pregunta pase moderación. Antes de esta sesión era cierto "por construcción" (frágil ante
refactor); ahora hay un **test que lo fija** (implementado por el usuario), pasando de garantía
accidental a garantía de regresión. Generalización graduable: **el punto de guardrail va antes de
todo efecto —cache, lectura de estado, cualquier trabajo—, y ese orden se protege con un test, no
se confía a la disposición actual del código.**

### 5.2 Interacción rechazo-en-historial `[dominio, con lección agnóstica]`

En un cache HIT de turno 1, el texto anexado como `assistant` puede ser un **rechazo**
(`Out of scope: …`) — que en M1, con un solo fragmento, es el caso *común*. El turno 2 llega
entonces con un historial cuyo turno assistant es un rechazo. El código es correcto (el historial
honesto incluye los rechazos, sin cita por S04), pero abre una pregunta de **calidad
conversacional** que solo la evaluación responde: *¿un rechazo previo en contexto degrada el
seguimiento?* Es el primer sitio donde la compuerta de cache (brief 1) y el corte-de-cita-en-rechazo
(S04) se cruzan. Acción: el UAT prueba un seguimiento **después de un rechazo**, no solo después de
una respuesta exitosa. Lección agnóstica: **cuando dos subsistemas correctos por separado se cruzan
(aquí: anexado-de-historial × señalización-de-rechazo), la corrección de código no garantiza
corrección de producto; la intersección se prueba explícitamente.**

### 5.3 Contraindicación del resumen acumulativo en corpus con trazabilidad `[dominio → agnóstico]`

Pregunta surgida en revisión: si la conversación supera la ventana, ¿no perdemos info? Respuesta y
nodo:

La ventana simple es defendible mientras (a) **la fuente de verdad sea el corpus, no el historial**
—los hechos normativos son re-recuperables re-preguntando, no se pierden al truncar— y (b) la
**anáfora sea de corto alcance** (apunta al turno anterior, no al origen; la conversación deriva).
En RAG-ECSS ambas se cumplen: el corpus es la memoria persistente, el historial es solo
desambiguación de corto alcance.

El resumen acumulativo se **contraindica no por coste sino por trazabilidad**: resumir turnos con
un LLM es **generar texto no verificado que entra al contexto como si fuera fiel**. Un resumen que
comprime mal —confunde criticidad B con C, atribuye a 40C algo de 10C— mete una **afirmación
normativa fabricada en el historial**, contaminando todos los turnos siguientes. Es el mismo vector
de fabricación del extractor de memoria (art. 2) y de la caché semántica, por la puerta del resumen.

Reabrir solo si el producto exige **síntesis multi-turno de largo alcance** (decisión de wedge:
Requirements Assistant hace preguntas encadenadas-a-corto; Compliance Helper teje). Y aun entonces,
preferir **anclas de turnos íntegros** (la "híbrida con anclas" con el ancla como turno completo,
no resumido) sobre resumen generado — para no introducir texto no verificado. Nodo agnóstico: **la
elección de estrategia de historial en un sistema de trazabilidad tiene un eje que el material
general no ve — comprimir con LLM material citable es introducir fabricación; la ventana simple es
la única estrategia que no genera texto no verificado.**

---

## 6. Anclajes y diferidos (actualizado desde el resumen)

Sin cambios de fondo respecto a `S05-resumen.md §3`; se re-listan con el estado tras implementar:

- **Perilla de disciplina/corpus en foco → M2.** Es perilla (no extracción). Cuando exista, SÍ es
  slot de clave (cambia qué se recupera). Tensión: filtrar el punto de entrada del retrieval, no el
  conjunto alcanzable por grafo (no romper multi-hop de referencias cruzadas).
- **Conflicto Redis-degradable vs estado-de-sesión-no-degradable → S06/M6.** Hoy no existe (dict en
  memoria no toca Redis). Salida probable que lo disuelve: PG como hogar durable del estado (fail-
  loud transaccional), Redis como cache delante (degradable). Redis se queda cache puro.
- **Atadura de un solo worker** (dict en memoria): escalar workers exige store compartido — es el
  mismo momento en que el conflicto Redis-sesión se vuelve real.
- **Condensación de query → M2/S10.** Multi-turn en CAG es la mitad fácil (el modelo resuelve la
  anáfora con el array de mensajes); en RAG hay que decidir sobre qué texto se recupera. La
  condensación restaura determinismo Y cacheabilidad (el seguimiento condensado re-entra al cache).
  Se sabe dónde iría; no se implementa.
- **Verificación de cita vía separación generación/verificación → S11 (forma de M5).** "Criticar el
  output" en recuperación *es* "verificar la cita"; Actor-Critic-Boss es el candidato. `answer` y
  `citations[]` ya separados → compatible sin deshacer nada.
- **Evaluación estadística real + golden dataset + DeepEval → S11/S15.** El aparato se define ahora
  (`LLM_EVAL_MODE`, dos suites); se puebla cuando haya retrieval que evaluar. Dos sentidos de
  "estadístico": dispersión intra-pregunta (para nosotros, **estabilidad de la cita**, no del texto)
  vs agregación sobre golden dataset.
- **Hueco de injection → M2+.** Condición correcta: "entra contenido no confiable de tamaño
  arbitrario", no "hay input de usuario". La propiedad-del-servidor sobre el historial cerró el
  vector del historial forjado; el de adjuntos sigue abierto.
- **Adjuntos como interacción central → producto maduro, atado al wedge.** Con ellos, la asimetría
  autoridad (corpus, citable) vs sujeto (documento del usuario, evaluado, NO citable).

---

## 7. Hilo conductor de la sesión

Los cinco artículos instancian una misma tesis, en versión estática (arts. 1-4) y procedural
(art. 5), y la implementación la confirmó en cada punto de fricción:

> En un sistema de trazabilidad hay una **frontera dura** entre *el material del que se responde y
> se cita* y *todo lo demás que rodea a la pregunta*; y una frontera análoga entre *generar* y
> *verificar*. Casi todo lo que el máster resuelve dejando que el LLM **infiera, adapte, recuerde,
> resuma o se autovalide**, RAG-ECSS lo resuelve con un **parámetro explícito**, un **invariante
> determinista** o una **separación de responsabilidades** — porque el producto no puede permitir
> que el LLM introduzca texto no verificado que afecte a la fidelidad.

Instancias observadas: autoridad-vs-sujeto (contexto externo) · historial-no-es-memoria +
perilla-no-extracción (estado conversacional) · tier-adapta-salida-no-cita (perfil) ·
propiedad-del-sistema-no-salida-del-modelo + dos-suites (testing) · generación-separada-de-
verificación (Actor-Critic-Boss) · y ahora **ventana-no-resumen** (§5.3): la única estrategia de
historial que no comprime material citable en texto no verificado.

En la Guide esto es **una entrada vertebradora con múltiples encarnaciones**, no seis entradas
sueltas. La formulación candidata: *"la frontera material-citable / contexto-de-la-pregunta, y su
corolario generar≠verificar, gobierna qué se delega al LLM y qué se hace explícito/determinista."*

---

## 8. Candidatos a graduar a `RAG-GUIDE.md` (paso 6)

El paso 6 es de juicio, no automático — esta lista es el material, no la edición. Marcados por
fase de la Guide donde encajan y si extienden un nodo existente o abren uno nuevo.

**Extienden Phase 4 (hardening: cache, robustez):**
- **Historial = compuerta de elegibilidad de cache, no slot de clave.** El historial desambigua la
  pregunta, no cambia la respuesta → no pertenece a la clave. Compuerta fina por presencia de
  historial (turno 1 elegible, comparte FAQ; turnos 2+ bypass). Tras condensación (M2/S10) el
  seguimiento condensado re-entra al cache. *(Extiende el nodo de exact-match cache.)*
- **La política de fallo es propiedad del caller, no del datastore; test de degradabilidad**
  (ausencia cambia coste → fail-open; cambia respuesta → fail-loud). Nunca compartir la envoltura
  de acceso entre inquilino degradable y no-degradable. *(Extiende "degradable vs fail-fast".)*
- **La cache es adversaria del instrumento de evaluación** (suprime la varianza que la evaluación
  mide) → debe desactivarse para evaluar, por entorno/modo no por flag de request. *(Nuevo,
  cuelga del nodo de cache.)*

**Nuevo bloque — estado conversacional (¿Phase 4 o phase propia de multi-turn?):**
- **Historial vs memoria; en un sistema de recuperación con trazabilidad, el multi-turn suele ser
  historial puro** (la fuente es el corpus, no un estado negociado). La "memoria" candidata es una
  **perilla explícita**, no extracción por LLM.
- **Propiedad del historial: servidor, por forja de turnos `assistant`** (el vector de amenaza es
  el decisor, no "control"). Coste: el backend deja de ser stateless. Atadura de un-solo-worker con
  dict en memoria, documentada como condición.
- **Ventana simple vs resumen acumulativo en corpus con trazabilidad** (§5.3): el resumen se
  contraindica porque comprime material citable en texto no verificado; reabrir solo con síntesis
  de largo alcance, y preferir anclas íntegras sobre resumen. *(Nuevo, importante.)*

**Nuevo bloque / extiende Phase 5 (producto, guardrails, testing):**
- **Dos suites soberanas: contrato (determinista, mockeada, CI, bloquea) vs evaluación
  (probabilística, modelo real, desarrollador, a demanda, análisis estadístico).** El eje
  determinista/no-determinista es ortogonal a la pirámide hard/soft/judge y decide qué corre en CI.
- **Evaluar un componente probabilístico es función de primera clase del producto**, materializada
  como un modo con nombre único (`LLM_EVAL_MODE`) que declara intención y deriva la mecánica; flags
  sueltos convierten el modo en un ritual memorizado cuyo olvido produce evaluación inválida en
  silencio. Guardarraíl fail-fast contra producción (validación de relación entre campos).
- **En un sistema de trazabilidad, la métrica de consistencia es la estabilidad de la cita, no la
  del texto.** Y con cache determinista, la consistencia de preguntas cacheadas es un invariante,
  no una propiedad estadística.
- **Tier adapta salida generada, no material citado** — en un sistema de recuperación el tier solo
  toca la glosa/presentación, nunca la cita. *(Nodo por negación.)*
- **"Perilla explícita" > "que el LLM lo infiera"** como patrón recurrente; el vector de amenaza de
  una perilla es proporcional a lo que desbloquea (presentación → cliente-dueño defendible; coste/
  datos → no).
- **Guardrail antes de todo efecto** (cache, lectura de estado, cualquier trabajo), protegido por
  test. *(Extiende "guardrails antes del cache" de S04.)*
- **Trigger de Jinja2 afinado**: parciales reutilizados entre variantes O ramas reales dentro del
  bloque estático — no "hay multi-turn" ni "hay tier" per se.

**Nuevo — generación/verificación (madurez M5/producción, condición no cumplida en M1/M2):**
- **El techo de una sola pasada es estructural: cuando el fallo es de verificación y no de
  instrucción, separar generación de evaluación** (Self-Refine). Tres roles, no dos (separar
  evaluación de decisión rompe bucle-infinito y sesgo-de-confirmación). En recuperación, el Critic
  útil *es* la verificación de grounding de S11. Feedback estructurado, iteraciones acotadas.

**Detalles de implementación agnósticos (menores, graduables como notas):**
- Mutación in-place de colección Pydantic en el camino caliente (no re-validar).
- Constante que codifica cardinalidad implícita = deuda silenciosa; anotar con su condición de
  disparo.
- Elegibilidad factorizada en un predicado único cuando varias condiciones la desactivan.
- Un buen seam absorbe un segundo propósito no anticipado sin rediseño.

**La entrada vertebradora (§7):** la frontera material-citable / contexto-de-la-pregunta + corolario
generar≠verificar, como principio del que las anteriores son instancias.
