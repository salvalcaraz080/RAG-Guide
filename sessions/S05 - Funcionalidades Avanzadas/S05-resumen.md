# S05 — Resumen de sesión

> Notas provisionales de S05 (multi-turn / conversación). Documento limpio de referencia:
> teoría destilada + decisiones tomadas para RAG-ECSS. Pre-implementación. La crítica al
> material del máster vive en la conversación, no aquí. Se consolida (paso 5) tras la
> implementación; no se reescribe.

**Tema nominal de la sesión:** multi-turn (historial, memoria, contexto dinámico externo,
prompts por tier, testing). **Tema real del brief para RAG-ECSS:** conversación multi-turn con
historial de propiedad del servidor, más un reencuadre del contrato de testing. El resto de la
sesión es teoría que se gradúa a la Guide con su condición de aplicabilidad, pero que no toca el
código de M1 (ver §3).

---

## 1. Teoría de la sesión

Cinco bloques. Se resumen en positivo (qué aporta cada uno), con su ubicación en la vida de un
proyecto RAG.

### 1.1 Contexto dinámico desde fuentes externas

El contexto que viaja al LLM se parte en dos naturalezas. **Estático**: vive en código o en
parámetros tipados, es predecible, versionable y testeable. **Dinámico**: se obtiene en runtime en
respuesta a una petición concreta, vive en sistemas externos (archivos del usuario, web, BBDD).

Tres reglas operativas del contexto dinámico:

1. **Es input, no programa.** Contenido que entra desde fuera del sistema se delimita
   explícitamente y nunca se le concede al LLM la capacidad de interpretarlo como instrucciones
   (prompt injection clásico).
2. **Tiene coste real por petición.** El estático se descuenta vía prompt caching; el dinámico se
   reincluye y consume tokens nuevos en cada llamada.
3. **Introduce latencia observable.** Procesar un documento, una búsqueda web o un round-trip a
   BBDD añade tiempo que el usuario nota.

Tres mecanismos canónicos:

- **Archivos adjuntos.** Dos caminos: (A) multimodal directo — el documento viaja al LLM, cero
  código de extracción, se paga lock-in con el proveedor multimodal y no hay control fino sobre
  qué entra; (B) extracción local — solo el texto viaja, independiente del proveedor, control fino,
  y es la primera pieza de un pipeline de chunking. No se implementan los dos: es decisión
  arquitectónica, no característica acumulable.
- **Búsqueda web.** Tres aproximaciones: herramienta nativa del proveedor (simple, lock-in),
  servicio especializado para LLMs (Tavily/Exa/Firecrawl — independiente, calidad orientada a IA),
  SERP tradicional (máximo control, máxima carga). Se activa solo cuando el modelo no puede
  responder con su propio conocimiento y la pregunta es sensible al tiempo.
- **Consulta a BBDD de negocio.** El servicio IA no accede a la BBDD operacional directamente
  (acoplamiento de schema, permisos amplios, lógica duplicada). El patrón correcto es function
  calling contra un endpoint HTTP autenticado del backend de negocio, que resuelve la consulta
  aplicando sus propias reglas. Distinción clave: **la BBDD vectorial es la base de conocimiento
  del servicio IA; la BBDD operacional es del negocio.** Se hablan por contrato HTTP, nunca por
  BBDD compartida.

Disciplinas transversales cuando coexisten varios mecanismos: **budget de tokens** por turno con
truncado/resumen al superarlo, y **trazabilidad** (cada herramienta invocada es un span
observable). Regla de fondo: arrancar con el mínimo contexto, medir, y añadir contexto dinámico
solo con evidencia de que el sistema lo necesita.

### 1.2 Historial vs memoria conversacional

Dos estructuras distintas con responsabilidades distintas:

- **Historial**: el array de mensajes (system/user/assistant) que viaja a la API en cada llamada.
  Estructura bruta, cronológica. Su gestión (cuántos turnos sobreviven, cómo se trunca/resume) es
  materia de S02.
- **Memoria**: el conjunto de hechos destilados sobre el dominio de la conversación. No son turnos,
  son afirmaciones. Cada hecho tiene un origen (un turno) pero es independiente de él: **sobrevive
  al truncado del historial.**

Tres razones para mantenerlas separadas: coste/latencia (el historial crece lineal, la memoria no),
resistencia al truncado (el hecho sobrevive a la ventana), y auditabilidad (los hechos asumidos son
inspeccionables).

Materialización canónica (en el material): la memoria es una estructura tipada (Pydantic) que se
inyecta en el system prompt vía template y se actualiza tras cada turno. Dos formas de actualizar:
heurística simple (coste cero, frágil) o LLM extractor (robusto, coste real + latencia + riesgo de
fabricar un hecho falso que persiste). El olvido necesita políticas explícitas: revisión por el
usuario, TTL de sesión, reset explícito.

Tres estrategias de gestión de historial (recordatorio de S02): ventana deslizante, resumen
acumulativo, híbrida con anclas. Separar memoria de historial permite usar la estrategia más simple
(ventana deslizante) sin perder los hechos que importan.

### 1.3 Prompts adaptativos por perfil (patrón "tier")

Adaptar la salida a distintos perfiles de usuario no requiere maquinaria de modelo (fine-tuning,
RLHF por rol): es desarrollo de aplicación. Tres capas:

1. **Persistencia**: el tier vive como una columna en la BBDD de negocio. Es dimensión de producto
   (qué experiencia recibe el usuario), no rol de autorización (qué puede hacer).
2. **Propagación**: el tier viaja al servicio IA por un canal que el cliente final no puede
   manipular (claim de JWT firmado, o header en red privada de confianza).
3. **Materialización**: el tier selecciona template Jinja2 **y** schema Pydantic de salida.

Regla central: **adaptar a un perfil significa adaptar la estructura de salida, no solo el tono.**
Un solo schema con "responde en tono X" es lock-in cosmético; el valor de la adaptación vive en la
estructura. Disciplina de templates: bloques compartidos en parciales (`include`) para no divergir.
El schema por tier actúa como segundo guardrail (fuerza estructura aunque el template falle).

El patrón escala desde "misma pipeline, distinto template" hasta "pipeline completamente distinta"
(ejemplo: un modo de investigación profunda que invoca otro modelo, habilita herramientas y opera
en background). El punto de despacho pasa entonces de template a pipeline.

### 1.4 Testing y evaluación de sistemas con LLM

El test unitario clásico (`assert == expected`) no aplica a outputs probabilísticos: produce falsos
negativos masivos (la suite roja con el sistema sano) o falsos positivos silenciosos (la suite verde
sin mirar la dimensión correcta). En su lugar, **los tests verifican propiedades**, no igualdad.

Tres familias, con costes y propósitos distintos (pirámide: mucha base, poca cima):

- **Hard determinista.** La propiedad no depende del modelo: validez de schema, campos presentes,
  rangos numéricos plausibles. Barato, rápido, primera capa.
- **Soft determinista (estadístico).** Ejecuta N veces el mismo input y verifica la forma de la
  distribución (p. ej. consistencia: la varianza del output no supera un umbral). Más caro.
- **Subjetivo (LLM-as-judge).** Una segunda llamada al LLM evalúa una propiedad genuinamente
  subjetiva (coherencia, comprensibilidad). Modos pointwise o pairwise. El juez también se equivoca
  y hay que calibrarlo; el umbral es una decisión de producto; se reserva para lo que no se puede
  capturar de otra forma.

**Golden dataset**: conjunto curado de casos representativos, cada uno anotado con sus criterios de
éxito (no una lista de inputs, una lista de inputs con su metadata de éxito). Es la base de la
evaluación seria; construirlo es inversión, no coste, y se revisa periódicamente. Herramienta de
entrada: DeepEval sobre pytest (nativa, sin infra externa, métricas listas). Lo que queda para más
adelante (LLMOps, producción): métricas específicas de RAG (RAGAS), regresión continua en CI/CD,
monitoring en producción, red teaming, datasets sintéticos.

### 1.5 Separación generación/verificación (Actor-Critic-Boss)

Hay un techo de calidad que no se rompe refinando el prompt: cuando el fallo es de **verificación**
y no de **instrucción**, la solución es estructural — separar la generación de la evaluación en
llamadas distintas (Self-Refine, Madaan et al. 2023, ~20% de mejora absoluta por separar generación
de feedback).

Tres roles, cada uno una llamada con su propio criterio de éxito:

- **Actor**: genera un candidato (la llamada que el sistema ya hace).
- **Critic**: evalúa el candidato contra criterios explícitos y produce **feedback estructurado**
  (no texto libre): issues con categoría, severidad, campo afectado.
- **Boss**: recibe candidato + feedback y **decide** — acepta, devuelve al actor para iterar con
  instrucciones, o sintetiza. Acota el número de iteraciones.

**Por qué tres y no dos**: si el crítico también decide sobre su propio feedback, aparecen dos modos
de fallo opuestos — bucle infinito por insatisfacción crónica (siempre encuentra algo que mejorar) y
sesgo de confirmación temprana (acepta su propia respuesta). Separar evaluación de decisión rompe
ambos. Anclaje en literatura: compone *evaluator-optimizer* y *orchestrator-workers* (Anthropic,
*Building Effective Agents*). Compensa cuando el coste del error es alto, los criterios de evaluación
son claros y la latencia extra es tolerable; se aplica a caminos críticos, no a toda llamada.

---

## 2. Decisiones de S05 para RAG-ECSS

Lo cerrado en esta sesión. Cada punto es una decisión de arquitectura tomada aquí.

### 2.1 Multi-turn en RAG-ECSS es historial, no memoria

Un sistema de recuperación con trazabilidad no construye un estado de dominio negociado turno a
turno (como haría un co-autor: "el proyecto se llama X", "el equipo son 3"). El usuario **pregunta**,
y la respuesta ya existe fija en el corpus. No hay `ProjectMetadata` análogo que destilar.

Lo que el multi-turn aporta aquí es **resolución de anáfora/elipsis**: *"¿y para criticidad B?"* es
ininteligible sin el turno anterior. Eso es **historial puro**, no memoria de hechos.

El único candidato a "memoria" (un dato que debe sobrevivir al truncado) es la **disciplina/estándar
en foco** (ingeniero de software vs de motores → qué corpus se consulta). Pero no es un hecho a
extraer: es una **perilla** — un parámetro explícito, tipado, declarado por el usuario/sesión, que
viaja como dato de request y sobrevive al truncado por construcción (no vive en el historial). No
necesita extractor LLM, ni riesgo de fabricación, ni maquinaria de memoria, ni políticas de olvido.
Su **maquinaria** (pre-filtrar el retrieval) es M2, no S05 (§3).

### 2.2 El servidor es dueño del historial

El backend asigna `session_id` y guarda el estado; el cliente no envía la lista de mensajes. Razón
decisiva: con el cliente dueño, un atacante puede **forjar turnos `assistant`** (inyectar una
respuesta falsa que el modelo tomaría como propia y construiría encima). El servidor, que generó las
respuestas, sabe qué se dijo de verdad.

Coste asumido: el backend deja de ser stateless (hoy cada `/query` es independiente y escala
horizontal gratis). Para S05 esto se acota a un **dict en memoria del proceso** — un solo worker, se
pierde al reiniciar. Persistencia y federación de sesiones son despliegue (S06/M6), no esta sesión.

**Atadura documentada, no asumida:** el dict en memoria solo es correcto con **un único worker**. Con
varios workers, el turno 1 crea la sesión en el worker A y el turno 2 puede aterrizar en el B, que no
la tiene. Escalar workers es exactamente el momento en que se necesita el store compartido.

### 2.3 El historial es una compuerta de elegibilidad de cache, no un slot de clave

El historial **no cambia la respuesta**, solo **desambigua la pregunta** — por eso no pertenece a la
clave de cache. Operacionalización:

- Petición **sin** `session_id` (single-turn) → **cache-elegible**, comportamiento M1 intacto.
- Petición **con** historial (turnos 2+) → **bypass de cache** (get y set).

Esto *revierte* el anclaje de S04 ("el slot de historial entra sin reingeniería"): entra
mecánicamente, sí, pero metería en la clave el prefijo entero de la conversación → el hit-rate
colapsa hacia cero conforme la conversación avanza, y la propiedad de auditabilidad (misma pregunta →
misma respuesta+cita) se degrada (pasaría a depender del prefijo). Se preserva el valor FAQ (la misma
primera pregunta la hace todo el mundo) sin pagar por seguimientos de prefijo único.

**Caducidad conocida:** en M2, con **condensación de query** (reescribir *"¿y para criticidad B?"* +
historial → pregunta autónoma), el seguimiento recupera una identidad cacheable — su forma
condensada, que puede coincidir con la primera pregunta de otro usuario. Entonces los seguimientos
re-entran al cache, keyed sobre la pregunta condensada. "Cache solo la pregunta inicial" es
correcto-para-M1 y temporal.

### 2.4 La política de fallo es propiedad del caller, no del datastore

Que un datastore se caiga es un evento; **qué se hace con ese evento** (degradar con warning, o
abortar) lo decide el código que lo accede, no el store. Un mismo Redis físico puede servir a un
inquilino degradable (cache) y a uno no degradable (estado de sesión) con políticas opuestas, porque
la política vive en la envoltura de acceso.

**Test de degradabilidad:** una dependencia es degradable **si y solo si su ausencia cambia el
*coste* de la respuesta, no la *respuesta***. Si su ausencia cambia la respuesta, debe fallar
ruidosamente.

- Cache caído → misma respuesta, más lenta → **degradable** (fail-open). *(Ya implementado, S03.)*
- Estado de sesión caído → un seguimiento se respondería sin su contexto → cambia la respuesta →
  **no degradable** (fail-loud).
- Moderación caída → no cambia la respuesta a una pregunta legítima, cambia si una tóxica se bloquea
  → se decidió fail-open **como excepción consciente y documentada** (deuda aceptada, S04), no porque
  el test lo dicte.

**Regla de higiene:** nunca compartir la envoltura de acceso entre un inquilino degradable y uno no
degradable, aunque compartan el store físico (evita que el estado de sesión herede por accidente el
handler fail-open del cache).

**Decisión viva en S05:** `session_id` **ausente-pero-provisto** (sesión perdida/expirada tras
reinicio) → **error explícito** ("tu conversación expiró"), nunca tratar como single-turn en
silencio. Distinto de **sin** `session_id` → single-turn legítimo, cache-elegible. Definir esta
semántica ahora hace que, cuando la persistencia sustituya el dict (S06/M6), no herede el fail-open
del cache por defecto.

### 2.5 Dos suites de testing soberanas, no una con excepciones

La actividad de "testing" se parte en dos productos distintos según un segundo eje —
determinista/no-determinista— ortogonal a la pirámide hard/soft/judge:

| | **Suite de contrato** | **Suite de evaluación** |
|---|---|---|
| Verifica | que *el código* respeta el contrato | que *el LLM* produce calidad |
| Veredicto | binario (falla/no falla) | gradiente (score, distribución, tendencia) |
| Determinismo | obligatorio (mockeado) | imposible (el modelo es la caja negra) |
| Red / modelo real | prohibidos | requeridos |
| Cache | irrelevante (respuesta mockeada) | debe poder desactivarse |
| Quién la corre | CC, tras cada implementación | el desarrollador, a demanda / periódico |
| Dónde vive | `uv run pytest`, bloquea merge | fuera de la suite por defecto |

La regla vigente "sin red, mockeado, `uv run pytest`" **se acota**: sigue siendo cierta para la suite
de contrato y para CC. No es la regla universal del proyecto — es la regla de *una* de las dos suites.
La familia 1 (hard) es la única que es contrato puro y entra en CI **con la respuesta del LLM
mockeada**; las familias 2 y 3 son evaluación (no CI, las corre el desarrollador). El mecanismo ya
existe desde S03: `@pytest.mark.real_llm`, excluido por defecto.

Reencuadres para nuestro dominio:
- **Los tests de contrato no pueden depender de qué proveedor respondió** (tenemos fallback; fijar la
  salida de un proveedor rompe el test cuando responde el otro, que es camino normal).
- **Con cache determinista, la consistencia (familia 2) es un invariante para preguntas cacheadas, no
  una propiedad estadística** — la cache es reproducibilidad de la respuesta citada, no solo latencia.
- **La métrica de consistencia de un sistema de trazabilidad es la estabilidad de la *cita*, no la del
  texto.**
- **Familia 1 verifica *emisión* de cita (bien formada, apunta a material inyectado); familia 3
  verifica *fidelidad* (la respuesta usó lo que cita).** Es la frontera emitir≠verificar de S11.

### 2.6 `LLM_EVAL_MODE`: la evaluación como función de primera clase

La configuración de evaluación es un **modo con nombre único** (`LLM_EVAL_MODE`), no un conjunto de
flags independientes. Un modo declara **intención** ("estoy evaluando"); el sistema **deriva** la
mecánica (bypass total de cache, y — cuando existan — logging de contenido para el análisis, sampling,
etc.). Flags sueltos convierten "estar en modo evaluación" en un ritual memorizado cuyo olvido produce
**evaluación silenciosamente inválida** (medir a través de una cache medio activa). Mismo principio que
el fail-fast de config y que "perilla explícita > inferencia del LLM".

**Por qué la cache debe desactivarse:** la cache existe para *suprimir* la varianza que la suite de
evaluación existe para *medir* — son teleológicamente opuestas. Con cache activa, un test de
consistencia obtiene hits y reporta varianza ≈ 0 midiendo la cache, no el modelo. Desactivable es
correctitud del instrumento. La desactivación es **por entorno/modo, no por flag de request** (la
cache-on/off es propiedad del despliegue, no de la pregunta; un flag de request abriría superficie de
ataque: forzar bypass y disparar coste).

**Postura de fondo:** evaluar un componente probabilístico es una **función del producto**, no una
actividad externa — es la única forma de saber si el producto funciona, y "funciona" no es binario.
Para un RAG de trazabilidad, medir fidelidad *es* saber si el sistema no fabrica. Por eso el sistema
soporta nativamente el modo evaluación, no "se truca" para evaluarlo.

**Guardarraíl que el modo exige:** fail-fast si `LLM_EVAL_MODE` coincide con entorno de producción. Un
modo que apaga protecciones y optimizaciones es catastrófico activado por accidente en producción;
la potencia del modo es justo lo que obliga a la barrera de arranque.

---

## 3. Anclajes y diferidos (lo abierto, con su destino)

Lo que la sesión deja explícitamente para cobrar más adelante. Registrado, no olvidado.

- **Perilla de disciplina/corpus en foco → M2.** La perilla es real y es la forma correcta (no
  extracción por LLM), pero no significa nada hasta que haya retrieval que filtrar. Tensión a resolver
  entonces: el filtro por disciplina debe acotar el **punto de entrada** del retrieval, no el
  **conjunto alcanzable** por expansión de grafo — un pre-filtro duro suprimiría las referencias
  cruzadas (multi-hop) que son el diferenciador del producto. Cuando exista, la perilla **sí es slot
  de clave** (cambia qué material se recupera → cambia la verdad de la respuesta), a diferencia del
  historial.

- **Conflicto Redis-degradable vs estado-de-sesión-no-degradable → S06/M6.** En S05 no existe (el dict
  en memoria no toca Redis). Cuando la persistencia sea requisito real, la forma probable que
  **disuelve** la tenencia conflictiva: Postgres (que ya se necesita para pgvector en M2) como hogar
  durable del estado de sesión (transaccional → fail-loud gratis), y cualquier Redis por delante como
  cache de la sesión (degradable, cae a PG). Redis se queda como cache puro para siempre.

- **Condensación de query → M2/S10.** El multi-turn honesto en M1 (CAG) es la mitad fácil: se manda el
  array de mensajes y el modelo resuelve la anáfora, porque el contexto es fijo. En RAG, alguien tiene
  que decidir **sobre qué texto se recupera**, y *"¿y para criticidad B?"* no es una query de
  retrieval. La condensación (reescribir el seguimiento en pregunta autónoma, materia de S10) restaura
  a la vez el determinismo y la cacheabilidad. No se implementa en S05; sí se sabe **dónde iría** para
  no cerrar la puerta.

- **Verificación de cita vía separación generación/verificación → S11 (con forma de M5).** "Criticar el
  output" en un sistema de recuperación *es* "verificar la cita". Actor-Critic-Boss es el candidato
  canónico para la verificación de grounding que hoy dejamos como candidata-no-verificada: Actor genera
  respuesta + citas, Critic verifica el respaldo de cada afirmación, Boss suprime las citas no
  verificables. Nuestro contrato ya separa `answer` de `citations[]`, así que es compatible sin
  deshacer nada. Es M5 por temario y S11 por dependencia.

- **Evaluación estadística real + golden dataset + DeepEval → S11/S15.** Dos sentidos de "estadístico":
  dispersión intra-pregunta (temprano, barato — para nosotros, estabilidad de la cita) vs agregación
  sobre golden dataset anotado (evaluación seria). El aparato se **define** conceptualmente ahora
  (`LLM_EVAL_MODE`, dos suites) y se **puebla** cuando haya retrieval que evaluar. DeepEval se evalúa
  como dependencia cuando lleguemos a evaluación de calidad real, no antes; en S05 solo tocaríamos
  familia 1 (pytest + Pydantic pelado, sin DeepEval).

- **Hueco de injection → M2+.** El `# TODO: injection guardrail` de `input_guards.py` deja de ser
  decorativo en cuanto entra **contenido no confiable de tamaño arbitrario** en el prompt — no cuando
  "hay input de usuario". Hoy nuestro único input no confiable es la pregunta (corta, moderada). Un
  historial de propiedad del cliente (turnos `assistant` forjables) o un adjunto (documento ajeno,
  largo) invierten eso. La propiedad-del-servidor sobre el historial (§2.2) cierra el vector del
  historial forjado; el de adjuntos sigue abierto para cuando adjuntos entren.

- **Adjuntos como interacción central → producto maduro, atado al wedge.** Si la customer discovery
  mueve el wedge a Compliance Matrix Generator / Verification Helper, el adjunto pasa de accesorio a
  input primario, y su clasificación de madurez hay que releerla. Con él llega la asimetría
  **autoridad vs sujeto**: el corpus ECSS es autoridad citable; el documento del usuario es sujeto
  evaluado, NO citable como norma. Si el prompt no los separa, el modelo citaría la especificación del
  usuario como si fuera cláusula ECSS.

---

## 4. Ediciones de estado que S05 dispara (para el brief de CC)

Distinto de la Guide: esto modifica el estado del repo y se ejecuta vía brief a CC.

- **`CLAUDE.md` — regla de tests.** Acotar la regla "sin red, mockeado, `uv run pytest`" a la suite de
  contrato y a CC, y nombrar la existencia de la suite de evaluación (soberana, del desarrollador,
  fuera de CI). El mundo de CC no cambia: sigue sin tocar red, sin Docker, sin evaluación.
- **`CLAUDE.md` / `.env.example` — `LLM_EVAL_MODE`.** Documentar el modo (atómico, deriva la mecánica,
  fail-fast contra producción). En S05 se cablea el mínimo que la sesión consuma (bypass de cache); se
  puebla el resto cuando llegue la evaluación real.
- **Multi-turn.** Estado de sesión servidor-dueño (dict en memoria, un worker), compuerta de cache por
  historial, semántica de `session_id` ausente-pero-provisto → error. Tests de esta lógica: familia 1,
  mockeados, `uv run pytest`.

---

## 5. Hilo conductor de la sesión (nota para el consolidado)

Los cinco artículos instancian una misma tesis, en versión estática (arts. 1-4) y procedural (art. 5):

> En un sistema de trazabilidad, hay una **frontera dura** entre *el material del que se responde y se
> cita* y *todo lo demás que rodea a la pregunta*; y una frontera análoga entre *generar* y *verificar*.
> Casi todo lo que el máster resuelve dejando que el LLM **infiera, adapte, recuerde o se autovalide**,
> nuestro producto lo resuelve con un **parámetro explícito**, un **invariante determinista** o una
> **separación de responsabilidades**.

Instancias: autoridad-vs-sujeto (contexto externo), historial-no-es-memoria + perilla-no-extracción
(estado conversacional), tier-adapta-salida-no-cita (perfil), propiedad-del-sistema-no-salida-del-modelo
+ dos-suites (testing), generación-separada-de-verificación (Actor-Critic-Boss). En el consolidado
probablemente sea **una** entrada de Guide con múltiples encarnaciones, no seis entradas sueltas.
