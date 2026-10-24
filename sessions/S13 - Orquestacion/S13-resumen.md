# S13 — Orquestación: frameworks frente a orquestación interna, LangGraph, estado y errores

> Resumen provisional (paso 2). Teoría de los seis artículos + lectura sobre RAG-ECSS + sesión en vivo.
> Pre-implementación: nada de lo que aquí figura como propuesta está validado en código ni medido.
> Parte de los anclajes de S12 (`S12-consolidado.md` §4 y §6): **no hay bucle agéntico que
> orquestar** (B1 = 9 resuelto con traversal determinista, B2 = 0).

---

## 1. La decisión raíz: qué forma tiene el flujo

**Tres modelos de orquestación (2026):**
- **Grafo** — LangGraph (1.0 en octubre de 2025). Flujo explícito y definido de antemano: control
  determinista y depuración sencilla a cambio de diseño previo.
- **Roles** — CrewAI. Un *manager* delega en especialistas. Prototipado rápido, menos control fino.
- **Conversacional** — linaje AutoGen; hoy **Microsoft Agent Framework** (fusión de AutoGen y
  Semantic Kernel, 1.0 en abril de 2026). Natural en Azure/.NET.
- En el radar de un equipo Python: **Google ADK**, **Pydantic AI** (tipado, inyección de
  dependencias al estilo FastAPI) y los SDK de agente de cada proveedor (bucle de agente único).

La novedad de todos está acotada: resuelven variaciones del mismo problema (coordinar pasos,
mantener estado, recuperarse de errores) con abstracciones distintas.

**La pregunta correcta no es "¿framework sí o no?", sino "¿qué forma tiene mi flujo?".**

**Escalera de decisión:**

| Forma del flujo | Respuesta |
|---|---|
| Una llamada al modelo (+ formato de salida, quizá una cola) | Nada que orquestar. Sin framework. |
| Agente único que razona y llama tools en bucle (ReAct) | Bucle a mano bien instrumentado, o `create_agent`. |
| Grafo de verdad: dependencias, ramas, paralelismo, persistir/reanudar, aprobación humana | `StateGraph`. |

**Impuesto de complejidad:** dependencias, capas de indirección, andamiaje y migraciones de API
(`create_react_agent` → `create_agent`). Se paga solo si se recibe algo a cambio.

**Mide la línea base antes de decidir.** El bucle (o pipeline) que ya existe es la referencia
honesta: si al reexpresarlo como grafo no se gana nada medible (claridad, recuperación,
observabilidad), el framework sobra.

**El contrato no cambia.** El grafo vive dentro del servicio IA; el cliente sigue enviando una
petición y recibiendo una salida estructurada con `status`. El framework orquesta; **no sustituye la
capa de llamada al LLM.**

**Patrón dominante en la práctica:** híbrido (framework para lo estándar, código propio para el
diferencial de dominio). Consejo dominante: empezar simple, instrumentar mucho, añadir complejidad
solo donde los datos la exijan. El error típico es pasarse un escalón de sofisticación.

### Matices para producción

- **Qué justifica un framework y qué no.** Varias razones de la escalera no lo requieren:
  - ramas condicionales → un `if`;
  - paralelismo → `asyncio.gather` / `TaskGroup`.

  Lo que de verdad cuesta reimplementar bien (los **disparadores reales**):
  - **persistir el estado y reanudar** tras un fallo (ejecución duradera);
  - **pausar a la espera de un humano** y retomar;
  - **observabilidad por nodo** sin trabajo adicional.
- **Workflow frente a agente.** Un grafo con aristas fijas es un *workflow*, no un agente. Todo
  pipeline ya es un grafo implícito en el código: **tener un grafo no implica necesitar un framework
  de grafos.**
- **Agnóstico del modelo, con letra pequeña.** `StateGraph` con nodos que llaman a nuestro wrapper sí
  lo es. `create_agent` usa las abstracciones de chat model de LangChain, que duplicarían el wrapper
  con fallback. Si algún día entra LangGraph, sería solo en la capa de grafo.
- **Ejecución duradera fuera del ecosistema de agentes** (Temporal y similares) es la alternativa
  cuando lo único que se necesita es reanudar.
- **El dato del "60 % de incidentes por gestión de estado"** procede del informe de LangChain (fuente
  interesada, no verificable) y solo aplica a sistemas con estado de larga duración.

---

## 2. LangGraph: cuatro primitivas

LangGraph separa **trabajo** (nodos), **control** (aristas) y **dato** (estado compartido).

1. **Estado** — `TypedDict` (o Pydantic/dataclass; uno y consistente). **Es la decisión de más
   peso**: todo se lee y escribe contra él.
   - Por defecto, una actualización **sobrescribe** el campo (`status`, `estimate`).
   - Con un **reducer** (`Annotated[list[X], operator.add]`) **acumula**: varias ramas pueden escribir
     a la vez sin pisarse.
2. **Nodo** — función que recibe el estado y devuelve **solo los campos que cambia**. No muta lo que
   recibe. Reutiliza la lógica de dominio existente; no orquesta.
3. **Arista** — directa (secuencia fija) o **condicional**: una función que lee el estado y devuelve
   el nombre del siguiente nodo. `START` y `END` son los centinelas. La función de enrutado **no
   trabaja ni llama al modelo**: solo decide.
4. **Checkpointer** — persiste el estado tras cada nodo (§3).

`StateGraph(Schema)` → `add_node` → `add_edge` / `add_conditional_edges` → `compile()` produce un
grafo ejecutable (síncrono, asíncrono, streaming, persistible).

**Disciplina del grafo:**
- reducers solo donde de verdad se acumula;
- aristas condicionales solo en decisiones reales;
- estado mínimo: todo se serializa en cada transición.

### Matices para producción

- **"Funciones puras" se refiere al trato del estado**, no a la función: los nodos llaman a retrieval
  y al LLM (efectos laterales). Siguen haciendo falta mocks para testearlos.
- **`TypedDict` no valida en tiempo de ejecución**, solo en el análisis estático. Validar exige
  Pydantic, con coste en cada transición.
- **Coste oculto del estado compartido.** Cada nodo ve todo el estado, así que sus dependencias
  reales no aparecen en su firma. Una función `chunks → context` declara su contrato; un nodo
  `EstimationState → dict` no. Compensa cuando el orden de nodos es variable; en una cadena fija es
  acoplamiento sin beneficio.
- **Separar trabajo y control es buena práctica sin framework:** que las funciones de cada etapa no
  decidan qué va después y que la secuencia viva en un solo sitio.

---

## 3. Estado y persistencia

**Reducers al reanudar.** Si al reanudar desde un checkpoint se pasan campos acumuladores en el
estado inicial, el reducer los **concatena con lo guardado y se duplican**. Regla del material: al
reanudar, pasar solo las entradas nuevas.

**Checkpointer.** Persiste el estado tras cada nodo; habilita pausar, reanudar, inspeccionar paso a
paso y la aprobación humana, sin escribir una capa de persistencia.
- Backends: `InMemorySaver` (tests), `SqliteSaver` (un servidor), `PostgresSaver` /
  `AsyncPostgresSaver` (producción, varias instancias; la asíncrona casa con FastAPI + asyncpg).
- **`thread_id`** ata cada ejecución a su historia: mismo `thread_id` → se reanuda desde el último
  checkpoint; distinto → ejecución limpia.

**Memoria corta frente a memoria larga.**

| | Corta | Larga |
|---|---|---|
| Qué es | Estado del *thread* | Dato de negocio durable entre sesiones |
| Dónde vive | Checkpointer | Almacén de negocio |
| Propósito | Operativo: reanudar, inspeccionar, aprobar | Contexto para ejecuciones futuras |

**El checkpointer no es la base de datos de producto.** Mezclar los dos problemas ensucia los dos.

**El tamaño del estado es rendimiento.** Estados de cientos de KB o MB convierten la escritura del
checkpoint (ms → cientos de ms) en el cuello de botella. Guardar IDs, resultados destilados y
banderas; lo voluminoso, en el ámbito de la función.

### Matices para producción

- **Reducers idempotentes por construcción.** "Pasa solo lo nuevo" es un ritual que alguien
  olvidará, y entonces los datos se duplican en silencio (mismo patrón que motivó `LLM_EVAL_MODE`).
  En producción el acumulador **une por clave** (deduplica) en lugar de concatenar. `operator.add`
  tampoco es idempotente ante el reintento de un nodo.
- **Memoria corta = memoria del *thread*, no de una ejecución.** En la documentación de LangGraph el
  caso canónico es una conversación de varios turnos. Con esa definición, un historial por turnos en
  el servidor ya es memoria corta (`session_id` ≡ `thread_id`).
- **"Reutiliza el Postgres" oculta dos costes:**
  - **Dominio de fallo y carga compartidos:** las escrituras de checkpoint compiten por conexiones
    con el retrieval. Además, compilado dentro del grafo, un fallo de escritura del checkpointer
    tumba la ejecución: algo que debería ser **degradable** (cambia el coste) pasa a **no degradable**
    (cambia la respuesta) por accidente.
  - **Retención:** las tablas de checkpoints crecen sin límite y guardan las entradas del usuario.
    Sin política de retención es retención de datos personales sin base (RGPD, S06).
- **IDs persistidos en un corpus versionado.** Un `clause_id` guardado solo es reproducible junto a
  su `corpus_version`: reanudar tras reconstruir el corpus puede resolver el mismo ID a otro texto.

---

## 4. Ejecución paralela y enrutado condicional

**Fan-out con la API `Send`.** Una función de despacho emite un `Send("worker", payload)` por
elemento; el trabajador procesa uno solo y devuelve únicamente el campo acumulador. El reducer hace
el **fan-in**; el nodo siguiente se ejecuta una vez, con todo reunido. La latencia del paso pasa de
la suma de las ramas a la de la más lenta.

**Aristas condicionales** para bifurcar según el estado (validación → `END` o revisión). Solo en
puntos de decisión reales.

**Ciclos acotados, en dos capas:**
- **Estrategia:** contador de intentos en el estado y función de enrutado que, superado el tope,
  desvía a una salida limpia.
- **Red de seguridad:** `recursion_limit` del framework. Es la última línea, no la estrategia.

**Coste del paralelismo: el merge del estado.**
- Campos con escrituras concurrentes → acumuladores (un campo de sobrescritura depende de qué rama
  terminó última).
- Cada rama devuelve la forma que espera el agregador.
- Trabajador mínimo, confinado al campo acumulador.

### Matices para producción

- **La condición de carrera existe porque hay estado compartido.** Con `gather`/`TaskGroup` cada
  tarea devuelve su resultado y se recoge una lista: sin reducer ni carreras.
- **Orden de los resultados.** `gather` preserva el orden de despacho por construcción. Con el
  reducer hay que comprobar la garantía de orden antes de depender de ella. En RAG el orden no es
  cosmético: el rango de retrieval fija la numeración `[n]` de las citas.
- **Límites de tasa (S02):** N llamadas simultáneas a embeddings o LLM chocan con el *rate limit*.
  Hace falta un semáforo que acote la concurrencia.
- **"Latencia = la rama más lenta" solo vale para trabajo de E/S.** En CPU dentro del proceso, el GIL
  se come buena parte de la ganancia.
- **Fallo parcial de una rama:** con `gather`, `return_exceptions=True` y una política explícita para
  los huecos.

---

## 5. Errores y recuperación

**Cada tipo de fallo pide una respuesta distinta:**

| Fallo | Respuesta | Mecanismo en LangGraph |
|---|---|---|
| Transitorio (pico de latencia, corte de red) | Reintento con backoff exponencial | `RetryPolicy(max_attempts, backoff_factor)` por nodo |
| Dependencia caída de forma persistente | Ruta alternativa + *circuit breaker* (enfriamiento, ir directo al camino degradado) | Lógica en el nodo |
| Excepción en un nodo | Reanudar desde el último checkpoint | Checkpointer |
| Baja confianza / ambigüedad | Parar y preguntar a un humano | `interrupt` + `Command(resume=…)` |

**Timeouts en el nodo, con degradación elegante:** el nodo que conoce la operación pone el timeout;
si expira, anota el hueco en un acumulador `errors` en vez de propagar una excepción que tumbe todo.

**`interrupt`:** pausa a mitad del nodo, persiste el estado y expone un valor a quien invocó; espera
indefinidamente. Requiere checkpointer. **Al reanudar, el nodo se re-ejecuta desde el principio**:
lo que va antes de `interrupt` tiene que ser barato e idempotente; lo caro, después.

**Puerta humana proporcional al coste del error:** ni automatizarlo todo ni una puerta en cada paso.
Una sola, en el punto de alto coste y baja certeza.

### Matices para producción (contrastados con las tres clases de error de S12)

S12 dejó (provisional, sin graduar) tres clases: **transitorio** (reintenta el código, el modelo no
lo ve), **error del modelo** (observación informativa redactada) y **bug interno** (falla en voz
alta: se loguea y se aborta o degrada). Cruzado con el material:

- **"Excepción → reanudar desde checkpoint" mezcla dos causas.** Reanudar re-ejecuta el mismo nodo
  con el mismo estado: sirve si la causa fue externa (caída del proceso, despliegue); con un bug o un
  dato malformado vuelve a fallar igual. La clase **bug interno** de S12 ya cubre ese caso: ni
  reintentar ni reanudar, fallar en voz alta. Las salidas inválidas del LLM son un caso intermedio
  (estocásticas: re-muestrear puede arreglarlas, mejor devolviendo el error al modelo), que encaja
  en **error del modelo**.
- **`RetryPolicy` necesita un predicado** que acote qué excepciones se reintentan; sin él reintenta
  también los bugs.
- **Amplificación de reintentos.** Si el router LLM ya reintenta y hace fallback, añadir reintento por
  nodo multiplica las llamadas (3 × 3 = 9 por fallo). **Los reintentos viven en una sola capa.**
- **La degradación tiene que verse en el contrato.** Que el campo `errors` se quede en el estado
  interno produce respuestas incompletas que parecen completas. Cuando omitir es el peor error
  (asimetría de RAG-ECSS), el hueco llega al cliente vía `status` / `origin`.
- **`confidence` sin origen definido.** La confianza autodeclarada por el modelo está mal calibrada
  (S12); una señal determinista (verificación de citas) es mejor disparador de la puerta.
- **En un producto conversacional, la puerta humana suele ser un turno.** Ante una pregunta ambigua,
  se devuelve una petición de aclaración y el historial lleva el contexto al turno siguiente: sin
  `interrupt` ni checkpointer. El checkpointer se gana su sitio cuando la pausa ocurre **a mitad del
  pipeline, tras trabajo caro que no debe repetirse**, o cuando **quien decide no es quien
  preguntó** (un aprobador distinto del usuario).

---

## 6. Observabilidad

**Span y traza.** Un span es un tramo con nombre, inicio, fin y atributos; los spans forman un árbol
padre/hijo (la traza): petición → grafo → nodo → llamada al modelo / consulta a la base. Métricas:
**latencia por nodo, tasa de éxito por nodo, coste por ejecución.** En un grafo, cada nodo es una
unidad natural de medida.

**Dos herramientas de referencia:**
- **LangSmith** (LangChain): traza y evaluación nativas de agentes; se activa con variables de
  entorno (`LANGSMITH_TRACING`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`). Encaje natural dentro de
  LangChain.
- **Logfire** (Pydantic): sobre OpenTelemetry, observa toda la aplicación
  (`instrument_fastapi`, `instrument_asyncpg`, `instrument_httpx`, una línea cada uno). Spans
  consultables por SQL, de modo que el coste por ejecución es una consulta.

**Criterio del material: solo LLM frente a toda la pila.** Las herramientas centradas en el LLM
(LangSmith, Langfuse, Arize) no ven lo que pasa entre la llamada a una herramienta y su resultado
(una consulta lenta, un problema de conexión), y ahí suelen estar los problemas.

### Matices para producción

- **La decisión que perdura es instrumentar con OpenTelemetry** (portable) y elegir backend después:
  Logfire, Langfuse (ingiere OTel), o Jaeger / Grafana Tempo autoalojados.
- **Privacidad y cumplimiento.** Un backend SaaS de trazas recibe prompts, texto recuperado y
  preguntas del usuario. En sectores con información sensible (proyectos del cliente, control de
  exportación en aeroespacial), SaaS frente a autoalojado es una decisión de cumplimiento.
- **El coste por ejecución hay que escribirlo:** la consulta presupone un atributo `llm_cost_usd` por
  span; registrar ese atributo es el trabajo real.
- **La observabilidad es lo que hace posible "medir antes de decidir"** en todas las decisiones
  anteriores (adoptar framework, paralelizar, qué hacer robusto).

---

## 7. Lectura sobre RAG-ECSS

**El pipeline expresado como grafo:**

```
START → route → retrieve → expand(bounded) → xref(clause) → generate(structured) → verify ─┬─ verified → END
                                                                                            └─ degraded → END
```

Una cadena con una sola arista condicional al final. Petición-respuesta de ~17 s, sin pausas ni
reanudación, sin humano a mitad. **Por la escalera del §1, primer peldaño: no hay nada que
orquestar.**

**Las cuatro primitivas, contra lo que ya existe:**

| Primitiva | En RAG-ECSS | Comentario |
|---|---|---|
| Estado tipado | Sí, como datos que pasan entre etapas + contrato tipado de salida | Cada etapa declara lo que consume y produce; en una cadena fija es mejor que un estado compartido. |
| Nodos | Sí: routing, retrieval, expansión, xref, generación, verificación | Testeados por separado (suite 1419). |
| Aristas condicionales | Sí, como `if`: filtro de routing, fallback de proveedor, verificado → degradado | No hay ciclos en producción. |
| Checkpointer | **No** | Lo más parecido es el historial de sesión, que persiste **entre turnos**, no **entre pasos de una petición**. Reanudar a mitad no ahorra nada: la petición es corta y barata, y todo salvo la generación es determinista. |

**Lo que falta es justo lo que no se necesita.** Esa es la señal de que el framework sobra hoy.

**Candidatos a bucle, con esta lente:**
- **Punteros a un estándar entero (q08, q10, q12, q14).** Una arista condicional más: puntero a
  cláusula → `xref` (como ahora); puntero a documento → retrieval filtrado a ese documento con la
  misma consulta. El siguiente paso depende de lo observado, pero lo resuelve una regla: por §3.8 no
  es bucle. Si hubiera que reformular la subconsulta para el destino, sería una llamada más, no un
  bucle, y sin checkpointer. (Dato de S12: el filtro por estándar llega al top 10 en 4 de 7.)
- **Recorrido de varios saltos del grafo de referencias.** BFS acotado por profundidad y presupuesto
  de tokens, determinista y de segundos. Si podar vecinos exigiera juzgar relevancia, sería un
  rerank por salto, no un agente. **Requisito de diseño:** el acumulador de cláusulas une por
  `clause_id` y respeta el presupuesto (la misma cláusula llega por dos caminos).
- **Regenerar una vez si la verificación falla** (ciclo acotado, nuevo frente a S12). Sí es bucle por
  §3.8 (el siguiente paso depende de qué citas fallan), pero **sin señal**: `verified` = 1,00 en la
  referencia. Si algún día entra, el reintento se registra como evento; si no, la tasa de primera
  pasada desaparece de las métricas y el reintento enmascara la degradación del generador.

**Paralelismo: medir antes de paralelizar.** Hay trabajo independiente (hasta 5 cláusulas por
`xref`, índices normativo y handbook), pero la mediana de 16,8 s probablemente la domina la
generación con razonamiento. Sin desglose de latencia por etapa, paralelizar retrieval podría ahorrar
milisegundos sobre 17 s. El desglose sale de structlog (duración por etapa con `request_id`), sin
árbol de spans.

**Errores, sobre lo que hay:**
- El router LiteLLM ya reintenta y hace fallback: es la capa de reintento. No añadir otra encima.
- Una `xref` o una expansión que expire en silencio produce una respuesta incompleta que parece
  completa. La degradación tiene que reflejarse en el contrato.

**Observabilidad:** elegir herramienta y montar trazas es materia de S15 (LLMOps). Lo accionable ya
es el desglose de latencia por etapa sobre structlog.

---

## 8. Sesión en vivo

**Preguntas y respuestas.**
- *¿LangGraph y LangChain son lo mismo?* — No. Mismo ecosistema, productos distintos; LangGraph es la
  orquestación mediante grafos.
- *¿Cuándo pasar a multiagente en lugar de usar un modelo más potente?* — Cuando la cantidad y
  diversidad de tareas consume demasiado contexto: especializar agentes y modelos por función en vez
  de resolverlo todo con más potencia. (Avance de S14.)
- *¿Una tool puede ser directamente un subagente?* — No conceptualmente: un agente dispone de tools,
  pero una tool no equivale a un subagente.
- *¿Ante una caída persistente del proveedor se puede retomar con otro proveedor manteniendo el
  estado?* — Sí, con un fallback diseñado de antemano: el sistema conserva el estado y continúa con el
  proveedor alternativo. (Matiz con S12: lo portable es **el estado propio** —el del grafo o el del
  cliente—, por eso se puede reanudar con otro proveedor **en la frontera de un nodo**. Lo que no es
  portable es el artefacto de razonamiento del proveedor **dentro de un bucle con tools**: a mitad de
  ese bucle el fallback reinicia el bucle, no lo continúa. Las dos afirmaciones son compatibles.)
- *¿Habéis trabajado observabilidad en sistemas de IA?* — Experiencia en el grupo con trazas y
  reportería básica (sin monitorización continua) y con LangSmith. El mentor: la observabilidad es
  esencial a partir de esta sesión.

**Consejos del mentor.**
- Empezar siempre con un solo agente; añadir complejidad solo cuando los datos lo justifiquen.
- El manejo del estado es la pieza central de una arquitectura de agentes.
- Checkpointer para recuperar ejecuciones; no confundirlo con memoria de largo plazo.
- Límites de iteraciones contra bucles infinitos y costes descontrolados.
- Reintentos con backoff exponencial, fallback entre proveedores y circuit breakers.
- Human-in-the-loop cuando el agente no resuelve con suficiente confianza.
- Pasar entre agentes solo el estado necesario, no historiales completos ni contexto irrelevante.
- Observabilidad desde el principio: latencia, costes, errores y dónde merece la pena subdividir el
  sistema.
- Para desarrollo y revisión de código, usar modelos o proveedores distintos para aumentar la
  probabilidad de detectar errores. (Enlaza con el protocolo de verificación de S12: la independencia
  es de contexto, no de modelo; la diversidad de modelo es una capa adicional, no un sustituto del
  ancla humana.)

---

## 9. Decisiones S13

### Decididas
- *(Ninguna todavía: se cierran al preparar el brief.)*

### Propuestas (pendientes de confirmar antes del brief)
- **No adoptar LangGraph ni otro framework de orquestación en RAG-ECSS.** El pipeline es una cadena
  con una arista condicional, petición-respuesta, sin pausas; de las cuatro primitivas solo falta el
  checkpointer, que no tiene caso. Se registra la **condición** (forma del flujo) y la **señal** que
  cambiaría la decisión (necesitar persistencia **entre pasos**, no entre turnos: humano en el bucle
  con aprobador distinto del usuario, o trabajos largos reanudables).
- **Candidatos a brief** (se elige en el paso 3):
  1. **Desglose de latencia por etapa** sobre structlog (duración por etapa con `request_id`,
     agregado en el arnés). Instrumento, no técnica nueva. Responde al diferido de latencia y a si
     paralelizar retrieval/`xref` merece la pena.
  2. **Punteros a estándar entero** (q08, q10, q12, q14): arista condicional determinista (puntero a
     documento → retrieval filtrado a ese documento). Con criterio preregistrado derivado de lo que
     el producto acepta y métricas que la técnica no mueva.
  3. **Auditoría de errores y degradación:** una sola capa de reintento (LiteLLM), predicado de
     reintento, y que todo hueco de expansión/`xref` (timeout, tope) sea visible en el contrato.
  4. **`origin` explícito en el contrato** (`xref | bounded | …`), prerrequisito del grafo de
     referencias; hoy la UI lo deduce.
  5. **Separación trabajo/control en el orquestador actual** (comprobación por CC, no refactor por
     defecto).

### Abiertas
- ¿La arista de punteros a estándar entero usa la misma consulta o una reformulada (una llamada
  más)? Inclinación: misma consulta primero, por sencillez.
- ¿El desglose de latencia va como brief propio o dentro del brief que se elija?
- ¿`origin` entra ya en S13 o con el brief del grafo de referencias?

### Diferidas
- Multi-agente, human-in-the-loop operativo, sandboxing → S14.
- Elección de herramienta de observabilidad (OTel + backend; SaaS frente a autoalojado por
  cumplimiento) → S15.
- Regenerar si falla la verificación → solo con señal (`verified` < 1,00).
- Checkpointer, reducers idempotentes, política de retención y `corpus_version` en IDs persistidos →
  solo si aparece persistencia entre pasos.
- Grafo de referencias (recorrido de varios saltos, acumulador por `clause_id`) → puerta antes de M6.

---

## 10. Candidatos a Guide (provisionales, no graduados)

- **Nodo framework frente a orquestación interna.** Condición: la forma del flujo. Los disparadores
  reales de un framework son persistir/reanudar, pausar para un humano y observabilidad por nodo; no
  las ramas ni el paralelismo. Señal: necesitar persistencia entre pasos dentro de una ejecución.
- **Memoria corta frente a larga.** La corta es incondicional en un RAG conversacional (se decide
  *dónde* vive). La larga tiene condición: se descarta en corpus normativos con trazabilidad
  obligatoria.
- **Checkpointer.** Condición: reanudar a mitad vale más que repetir entero, o hay pausa con aprobador
  distinto del usuario. Requisitos al adoptarlo: reducers idempotentes, política de retención,
  dominio de fallo que no convierta lo degradable en no degradable, IDs con versión del corpus.
- **Reintentos en una sola capa**, con predicado.
- **La degradación se ve en el contrato**, no en el estado interno.
- **En producto conversacional, la puerta humana es un turno** salvo pausa a mitad tras trabajo caro
  o aprobador distinto.
- **Paralelizar solo con desglose de latencia por etapa delante.**
- **Observabilidad: instrumentar con OpenTelemetry; el backend es una decisión posterior y de
  cumplimiento.**
