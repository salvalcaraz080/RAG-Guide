# S14 — Multi-agente, human-in-the-loop y privilegio de acciones

> Resumen provisional (paso 2). Teoría de los seis artículos + lectura sobre RAG-ECSS + sesión en vivo.
> Pre-implementación: nada de lo que aquí figura como propuesta está validado en código ni medido.
> Parte de los anclajes de S13 (`S13-consolidado.md`): sin framework de orquestación, sin bucle
> agéntico, la puerta humana es un turno, y el grafo de referencias como puerta antes de M6.
> El sandboxing de runtime (contenedores, red, límites de recursos, secretos) **no** entra en S14:
> el material lo difiere a M6.

---

## 1. La frontera: workflow, agente, multi-agente

**Quién es dueño del control flow.** Lo que separa un workflow de un sistema agéntico no es el número
de nodos, sino quién decide qué se ejecuta después: el código (workflow) o el modelo en runtime
(agente). Un "multi-agente" que solo renombra funciones es teatro de arquitectura, y se paga en
latencia, tokens y depuración.

**El grafo lineal no es un borrador.** Un flujo con secuencia estable se resuelve mejor con un grafo
determinista: predecible, barato de trazar, fácil de testear.

**Señales del techo del grafo único:**
1. **Prompt que acumula reglas de dominios distintos** — tocar una regla rompe un caso no relacionado
   (acoplamiento).
2. **Muchas tools en un único espacio de decisión** — la tasa de elección incorrecta sube con el
   número de opciones.
3. **El orden no se conoce de antemano** — el camino óptimo depende del caso.
4. **Responsabilidades que evolucionan a ritmos distintos** — ejes de cambio distintos piden
   componentes distintos.

**Cuándo no hacerlo:** si el flujo es fijo (un supervisor con política fija es una arista condicional
cara); si el problema real es un prompt malo (repartirlo da N prompts malos + coordinación); si no hay
observabilidad por nodo.

**Costes de multi-agente:** llamadas de enrutado que no producen trabajo, pérdida de contexto en los
traspasos, no-determinismo del camino, más superficie de fallo, coste cognitivo para quien mantiene.

### Matices para producción

- **Dos decisiones en serie, no una.** La señal 3 (orden desconocido) justifica dar el control al
  modelo, pero lo resuelve **un solo agente** con tools en bucle. Las señales 1 y 2 (saturación de
  contexto o de tools) son las que justifican **partir en varios agentes**. Nodo 1: ¿workflow o
  agente? (dueño del control flow). Nodo 2: ¿uno o varios agentes? (saturación). Afina Guide §3.8.
- **Componente ≠ agente.** La señal 4 pide módulos separados (prompts en ficheros distintos, nodos
  distintos en un grafo determinista), no una llamada al LLM más con su propio bucle.
- **Antes de partir por exceso de tools:** filtrado o carga dinámica de tools según la consulta, o
  agentes-como-tools bajo un agente principal.

---

## 2. El supervisor

**Tres funciones:** descomponer, delegar, consolidar. **No hace trabajo de dominio ni tiene tools de
negocio**; si las tiene, es otro nodo sobrecargado.

**El digest.** El supervisor no ve el historial: ve una proyección compacta del estado, construida a
mano, que responde solo a lo que necesita para enrutar ("¿ya buscó?", no "¿qué encontró?"). Coste
constante por decisión; cada dato que se añade es una decisión explícita.

**Decisión tipada.** `Literal` cerrado de destinos + `reason`. El mismo `Literal` tipa
`Command[AgentName]`, así que el contrato del modelo y la topología del grafo son el mismo objeto.
`Command(update=..., goto=...)` actualiza estado y mueve el control en un solo retorno; las aristas
viven dentro de los nodos.

**Piezas obligatorias:**
- **Presupuesto de enrutado** (`MAX_ROUTING_STEPS`) desde la primera línea: en un grafo enrutado por
  un modelo, el bucle es el comportamiento por defecto ante una instrucción ambigua.
- **`routing_trail`** en el estado (memoria de decisiones, inspeccionable desde tests).
- **Span con la decisión** (`next_agent`, `reason`), no con la respuesta.

**Impuesto de enrutado.** Topología plana: cada salto de trabajo lleva su salto de enrutado (×2
llamadas). Solo compensa si la ruta varía de verdad.

**Supervisor híbrido (la opción por defecto).** Las precondiciones se resuelven por reglas; el modelo
solo enruta donde hay ambigüedad real. Corolario: si al escribirlo no queda ninguna ambigüedad real,
no hace falta supervisor con modelo.

**Testing:** se testean resultado e invariantes (precondiciones respetadas, techo no superado), no el
camino.

**Plano frente a jerárquico:** jerarquía (supervisores de equipo) solo cuando el plano empiece a
fallar por número de opciones. Cada nivel es un salto de enrutado más.

**Abstracciones (`create_supervisor`):** según el material, la recomendación actual de LangChain es
implementar el patrón a mano para conservar el control del contexto y del enrutado (no verificado).

### Matices para producción

- **Reglas de precondición en un system prompt** convierten una garantía en una probabilidad. El primer
  snippet del material es ese antipatrón; el híbrido lo corrige.
- **Camino de fallo del parse:** excepción, negativa o `output_parsed=None` necesitan ruta de
  degradación; si no, tumban el grafo.
- **Agotar el presupuesto → `finalize`** solo es correcto si `finalize` cierra con estado incompleto
  y la degradación llega al contrato.
- **`reason`** es racionalización generada con la decisión: sirve para depurar, no para auditar. Si
  va **antes** que `next_agent` en el schema, el modelo razona antes de decidir.
- **Un digest de conteos pierde la calidad** ("3 matches" irrelevantes). El diseño del digest fija qué
  decisiones puede tomar el router: es ingeniería de contexto aplicada al enrutado.

---

## 3. Comunicación entre agentes: una escalera

| Patrón | Qué es | Gana | Paga | Cuándo |
|---|---|---|---|---|
| **Estado compartido (blackboard)** | Cada agente escribe su parcial en un estado común; nadie habla con nadie | Desacoplamiento entre agentes (solo al esquema), trazabilidad, coste | Estado que crece hacia objeto Dios; todos leen todo | **Por defecto** |
| **Handoff (swarm)** | El agente que termina pasa el testigo con una tool que devuelve `Command` (`graph=Command.PARENT`) | 2 llamadas en lugar de 4 frente a un supervisor con LLM | Acoplamiento topológico (estrella → malla), trazas repartidas | Impuesto de enrutado **medido** |
| **Mensajes / eventos** | Publicar hechos en un bus; nadie llama a nadie | Desacoplamiento máximo, escalado independiente | Consistencia eventual, `correlation_id`, infraestructura | Agentes que son **servicios** (ciclo de vida o equipo propio) |

Son combinables: la pregunta es qué patrón corresponde a cada frontera del sistema.

**Reducers:** para cada campo, decidir conscientemente si acumula o sobrescribe. Un campo que debería
acumular y sobrescribe pierde datos en silencio. **Proyecciones** contra el objeto Dios: cada agente
recibe su vista.

**Qué viaja en un traspaso** (`task_brief`): historial completo (coste y ruido heredados) frente a
brief redactado (contexto limpio, pero lo omitido no existe para el siguiente). Decisión explícita;
el default de casi todas las librerías (historial completo) es el que peor escala.

### Matices para producción

- **El ahorro del handoff se calcula contra el supervisor equivocado.** Frente al supervisor híbrido
  (enrutado por reglas, sobrecoste ~0) el handoff no ahorra llamadas y conserva sus costes. Orden:
  supervisor híbrido primero; handoff solo si el enrutado residual en el LLM sigue siendo caro.
- **El handoff reintroduce el nodo con dos responsabilidades** (trabajo + enrutado + una tool por
  vecino), lo que el supervisor quería evitar.
- **Acoplamiento al esquema incluye semántica y calidad del campo.** Contratos y validación también en
  la lectura.
- **Reducers en paralelo y orden.** `operator.add` concatena en orden no determinista. Si el consumidor
  depende del orden (posición en un prompt), la salida varía sin que cambie la entrada. Reducer
  correcto: combinar, deduplicar por clave, ordenar estable.
- **Evolución del esquema de un estado persistido:** checkpoints con esquema viejo se rompen al
  cambiarlo. Versionar si se persiste.

---

## 4. Human-in-the-loop

**La pausa es persistencia.** Una revisión puede durar días: la ejecución muere y resucita en el mismo
punto. El checkpointer ya lo resuelve. `interrupt(payload)` lanza una excepción de control, persiste
y termina; `ainvoke(Command(resume=decision), config)` reanuda con el mismo `thread_id`.

- **`thread_id` = ID de dominio**, el mismo en servicio IA, backend, frontend y trazas.
- **El payload es la interfaz del revisor**, no un log: lo necesario para decidir.

**Señales de disparo = booleanos evaluables sobre el estado.** Si no se escribe como booleano, es una
intuición. En el ejemplo: confianza baja, fuera de banda histórica (aritmética), sin precedente
(ningún análogo sobre umbral).

**Antipatrones:**
- **Errores al humano.** Error = el sistema no pudo hacer su trabajo (reintento, fallback,
  degradación). Revisión = lo hizo y el resultado necesita juicio.
- **Revisar por si acaso.** Si todo se revisa, nada se revisa: un gate que dispara en el 5 % vale más
  que uno al 60 %.
- **Reglas de negocio duras en el grafo.** Son flujos de aprobación y viven en el backend. El gate del
  servicio IA es para "el sistema sabe que no sabe".

**Contrato entre capas.** Un valor nuevo en `status` (`awaiting_human_review`) + endpoint `/resume`.
Autorización, notificación, caducidad e histórico de aprobaciones son del backend de negocio.

**Lo que muerde:**
- **El nodo se reejecuta entero al reanudar.** El nodo que interrumpe solo interrumpe: nada de efectos
  laterales ni llamadas al modelo antes del `interrupt()`.
- **Pausas huérfanas** — política de plazo, escalado o caducidad (decisión de negocio).
- **Doble reanudación** — endpoint idempotente o bloqueo en backend.
- **El par propuesta–decisión humana es el dato más valioso del sistema.** Guardarlo desde el día uno.

### Matices para producción

- **Prioridad de señales: deterministas primero.** La confianza autodeclarada por el LLM calibra mal;
  solo es usable calibrada contra decisiones humanas acumuladas.
- **Precisión sin recall.** Hace falta muestreo de auditoría sobre los casos **no** marcados para saber
  si el gate es selectivo o ciego.
- **Un grafo reanudado puede volver a interrumpir**: `/resume` debe reutilizar la rama de
  `__interrupt__`, no devolver siempre `completed`.
- **Obsolescencia del checkpoint:** tras días de pausa, el histórico o el corpus pueden haber cambiado.
  Versionar la evidencia (`corpus_version`).
- **Cuándo no hace falta checkpointer** (omitido por el material): su caso lo justifica porque el
  aprobador es **distinto** de quien pidió y hay **trabajo caro** antes de la pausa. Si quien revisa es
  el mismo usuario, la pausa es un turno.

---

## 5. Competición y síntesis

**Problema:** un estimador único no mide su propia fragilidad, y la confianza autodeclarada
correlaciona poco con el acierto.

**Cooperación frente a competición.** Cooperación: piezas ortogonales, una pasada, riesgo de cadena
(un eslabón débil contamina lo siguiente). Competición: varios agentes atacan la misma tarea con
criterios opuestos y un sintetizador decide; ×3 coste, ×~2 latencia.

**Implementación:**
- **Propuestas con esquema**: número + `assumptions` + `risks`. Los supuestos son la carga útil: la
  diferencia se convierte en una pregunta resoluble.
- **Paralelo con fan-in por reducer** (`proposals: Annotated[list, operator.add]`); sin anotar, un
  competidor desaparece en silencio.
- **La divergencia se calcula, no se opina**: `(hi−lo)/hi`, cero tokens. Asimetría: convergencia →
  cierre automático; divergencia → juicio humano. Alimenta el gate de HITL.
- **El sintetizador no promedia**: rango, supuestos que lo mueven, `open_questions`.

**Cuándo es un fraude:**
- **Trampa de la correlación**: mismo modelo, mismo contexto, prompts que difieren en un adjetivo →
  ilusión de segunda opinión y falsa señal de confianza. Diversidad real: criterios sustantivos,
  **evidencia distinta** (la palanca más fuerte), idealmente modelos distintos.
- **No hay juicio que hacer**: si un experto no sabría defender las dos posturas, los agentes tampoco.
- **N > 2**: retornos decrecientes con coste lineal.

**Muestreo repetido (self-consistency)** como alternativa barata. Según el material mide el ruido del
modelo; la competición, la incertidumbre del dominio.

**Decisión:** depende de lo que cueste equivocarse.

### Matices para producción

- **Converger no es acertar.** Los competidores comparten los requisitos del mismo extractor; si este
  omite algo, convergen sobre el mismo error y la divergencia baja lo presenta como confianza alta.
  La convergencia mide robustez a los supuestos variados, no corrección. Diversificar también la
  entrada.
- **Incoherencia interna del material:** calcula la divergencia y luego pide al sintetizador que
  "reporte su confianza". Lo coherente: confianza y disparo de HITL derivados de señales
  deterministas; el LLM solo explica.
- **Self-consistency no es solo ruido.** En tareas con una única respuesta correcta, la dispersión
  entre muestras predice error (base de detectores de alucinación tipo SelfCheckGPT). El orden de
  preferencia se invierte en problemas factuales.
- **Modelos distintos de calidad desigual** inflan la divergencia con error, no con incertidumbre.
- **Sesgo de posición en la síntesis:** ordenar las propuestas de forma estable (p. ej. por `stance`).

---

## 6. Mínimo privilegio, validación de acciones y auditoría

**El prompt no es un mecanismo de seguridad**: es una instrucción que el modelo pondera. El modelo es
el cliente; lo que propone se valida en una capa que no controla.

**Tres capas:**
1. **Mínimo privilegio en el reparto de tools.** Riesgo explícito por tool (`PURE`, `READ`, `WRITE`,
   `EXTERNAL`). Grants como dato, verificados al arrancar (un agente con una tool no concedida rompe el
   despliegue). Escritura concentrada en un agente pequeño: la superficie peligrosa cabe en un fichero.
2. **Guardia de acciones.** Código determinista entre intención y ejecución; valida argumentos, no solo
   permiso. La comprobación clave ata cada acción a la ejecución en curso (`estimation_id` =
   `thread_id`). **Acciones irreversibles → gate humano** (la misma pausa, otra señal).
3. **Auditoría.** Toda acción con efectos se registra, **incluidas las denegadas** (la tasa de
   denegación es una alarma temprana). Indexada por ID de dominio, con datos sensibles redactados: se
   audita la forma de la acción, no el contenido.

**Dos fronteras:** aplicación (qué puede hacer un agente dentro de la lógica; esta sesión) y runtime
(qué daño puede hacer el proceso: aislamiento, red, recursos, secretos; M6). Se complementan.

### Matices para producción

- **El privilegio efectivo vive en las credenciales**, no en la lista de tools: usuario de BD de solo
  lectura por rol, seguridad a nivel de fila, credenciales con el alcance justo. El dict de grants es
  documentación verificable; la credencial es la restricción.
- **Leer no es inofensivo.** Una tool de lectura sin acotar por cliente/tenant filtra datos ajenos.
  Las lecturas también se acotan al contexto de la ejecución (enlaza con la securización de la capa
  de datos de S09).
- **Inyección indirecta:** todo lo que el modelo lee es vector, incluidos los documentos recuperados.
  En un RAG, el corpus es superficie de ataque.
- **Invariantes en la propia tool** (su esquema), guardia central solo para lo transversal (privilegio
  y ámbito de la ejecución).
- **Auditoría de verdad:** append-only, a prueba de manipulación, con retención. La alarma de
  denegaciones necesita línea base.

---

## 7. Lectura sobre RAG-ECSS

**El producto se queda en el primer peldaño de cada escalera, por falta de señal:**

| Decisión | RAG-ECSS hoy | Qué la haría subir |
|---|---|---|
| ¿Workflow o agente? | Workflow (control en el código) | Orden no conocido de antemano |
| ¿Uno o varios agentes? | — | Saturación de contexto o de tools |
| Enrutado | Reglas (`filter`) | Ambigüedad real a mitad del flujo |
| Comunicación entre etapas | Estado compartido de facto (contrato estructurado; la generación recibe una proyección: el contexto) | Impuesto de enrutado medido / fronteras de proceso |
| Puerta humana | Turno | Aprobador distinto o pausa tras trabajo caro |
| Competición | No aplica | Juicio legítimo con error caro |
| Seguridad de acciones | No aplica (sin tools `WRITE`/`EXTERNAL`) | La primera tool con efectos |

**Competición.** Las preguntas típicas son factuales-normativas, ancladas en texto. El único candidato
con componente de juicio sería tailoring/aplicabilidad, pero el producto cita y no decide el
tailoring por el ingeniero. Además el LLM ya es el 93–94 % de la latencia. Para salidas ancladas en
fuente, la señal de incertidumbre preferente es la verificación determinista contra la fuente (ya
tenemos verificación de citas) y, después, muestreo repetido.

**HITL.** El revisor es el propio ingeniero que pregunta: el anclaje de S13 se confirma (turno, sin
checkpointer). Lo que sí transfiere sin maquinaria:
- **"El sistema sabe que no sabe" = abstención o aclaración**, con señales deterministas análogas
  ("sin precedente" → ninguna cláusula sobre umbral; "fuera de rango" → la pregunta nombra un
  estándar o cláusula inexistente en el corpus). Conecta con el diferido de premisas falsas (q14).
- **Error ≠ revisión en el contrato**: `status` debería distinguir fallo técnico/degradación de
  evidencia insuficiente. Conecta con el diferido de degradación visible.
- **El par respuesta–feedback del ingeniero** como materia prima del golden set. Desconocido si hoy se
  captura.

**Privilegio.** Única capacidad: leer de un corpus público. Aplica en versión débil: credencial de
solo lectura para el retrieval (estado desconocido desde S09) y el corpus como superficie de
inyección (riesgo bajo: fuente de confianza). En un corpus privado o multi-cliente (proyecto de
localización) el filtrado de lecturas por tenant sube a prioridad.

**Aportes a la puerta del grafo de referencias (anotados, no decididos):**

| Patrón del módulo | Requisito heredado |
|---|---|
| `MAX_ROUTING_STEPS` | Presupuesto de saltos al seguir punteros |
| `routing_trail` | `origin` en el contrato |
| Reducer con semántica explícita | Acumulador por `clause_id`: deduplicar por clave, **orden estable** (por relevancia u `origin`, no por llegada; fija la posición en el prompt), regla de qué `origin` gana si llega por dos caminos. Sin determinismo, el golden set no es reproducible. |
| Supervisor híbrido | Reglas para xref explícitos; juez LLM solo donde haya ambigüedad real de relevancia |

---

## 8. Sesión en vivo

**Preguntas y respuestas.**
- *¿Se usa ACP para comunicar agentes?* — Existen protocolos agente-a-agente; importan cuando agentes
  de distintos runtimes, empresas o servicios colaboran. Dentro de un único sistema no se usan.
  (Matiz: hay colisión de nombres. "Agent Client Protocol" conecta editores con agentes de código; el
  protocolo agente-a-agente de referencia es A2A. Es el escalón "mensajes" del §3 llevado entre
  organizaciones.)
- *¿Tools y Skills son equivalentes?* — No. Una tool da capacidad de actuar; una skill es un saber
  hacer: receta o procedimiento que el agente sigue.
- *¿Agentes en Docker/VM orquestados desde Python?* — Sí, pero ya es arquitectura fuera de LangGraph:
  levantar el contenedor, pasar configuración y permisos, recoger el output, registrar. Los principios
  de privilegio, auditoría y comunicación siguen aplicando. (Es la frontera de runtime del §6, M6.)
- *¿Píldoras de determinista frente a agéntico?* — Se compartirán (pendientes).
- *¿Límite entre Claude + MCP/documentación/memoria y orquestar con LangGraph?* — No hay frontera
  universal; depende del stack y capacidades que ya tenga la empresa. (Matiz: la escalera de S13 da el
  criterio operativo: forma del flujo y necesidad de persistencia entre pasos.)
- *¿Cómo reproducir una ejecución fallida de hace días?* — Estado y trazas quedan historificados;
  LangGraph permite partir de un checkpoint anterior. (Matiz: reanudar desde un checkpoint reproduce
  el **estado**, no la **salida del modelo**, que no es determinista. Reproducir de verdad exige
  guardar las salidas del LLM en la traza o reejecutar con ellas inyectadas.)

**Consejos del mentor.**
- No confundir un pipeline secuencial con orquestación agéntica: si el flujo está prefijado, el control
  es del workflow.
- Supervisor cuando haya que decidir dinámicamente qué agente actúa y en qué orden.
- HITL con confianza baja o estimaciones muy discrepantes entre políticas.
- Privilegios y guardarraíles para operaciones persistentes o irreversibles.
- Limitar reintentos y comprobar estado contra bucles.
- Concurrencia explícita: que varios agentes no sobrescriban el mismo estado.
- IDs de ejecución para no mezclar procesos paralelos.
- Trazar y loguear las decisiones del supervisor.

---

## 9. Decisiones S14

### Decididas
- **No introducir multi-agente, supervisor, handoff, competición ni checkpointer en RAG-ECSS.**
  Ninguna señal de §1–§5 está presente. Se registran condición y señal de cada nodo para la Guide.
- **Prompt por tipo de pregunta: ni agentes (A) ni plantilla por tipo (C).** El tipo de "no respuesta
  normal" (premisa falsa, delegación, hueco real) solo se conoce después de leer el contexto
  recuperado (q14 turno 1 parece una pregunta normal), así que no se puede clasificar antes de
  generar. Se resuelve como **B aplicado al contrato**: una categoría en el esquema de salida
  sustituye a reglas acumuladas en el prompt, anclada por la verificación de citas existente.
- **Brief 1 — taxonomía del "sin respuesta normal"** (`S14-brief.md`): (A) degradaciones técnicas
  visibles en el contrato, sin tocar mensajes; (B) huecos tipados `not_in_material` /
  `premise_contradicted` / `delegated`, con cita obligatoria verificada por código, tras opción
  apagada y medidos contra criterio preregistrado. Casos con señal: q14 turno 1, q14-a1, q12-a3.
  (Corrige el planteamiento inicial del candidato 1 como "solo señales deterministas": lo técnico
  lo detecta el código; lo de contenido lo declara el modelo y lo prueba una cita.)
- **Brief 2 — UAT estructurado (2b):** cada veredicto de UAT como registro ligado a `request_id`,
  consulta, respuesta y `corpus_version`, como materia prima del golden set. Sin llamadas al LLM.
  Incluye la **comprobación de credencial de solo lectura en el retrieval** (verificación, sin
  refactor).
- **Feedback en el producto (2a): no ahora.** Hoy no se captura ningún feedback; con un solo usuario
  el retorno es bajo. Señal para revisarlo: usuarios distintos del desarrollador.

### Abiertas
- Si la normalización del estado debe tratar los huecos tipados distinto de `not_found` (T7 del
  brief 1 la deja igual a propósito): se decide con los datos de la pasada.

### Diferidas
- Sandboxing de runtime (aislamiento, red, recursos, secretos) → M6.
- Determinismo del acumulador (dedupe por `clause_id`, orden estable, regla de `origin`), presupuesto
  de saltos, juez LLM solo con ambigüedad → puerta del grafo de referencias.
- Filtrado de lecturas por tenant → proyecto con corpus privado (fuera de ECSS).
- Capas de reintento: el brief 1 solo las inventaría; la decisión, con su reporte.

---

## 10. Candidatos a Guide (provisionales, no graduados)

- **Workflow frente a agente** (dueño del control flow) y **uno frente a varios agentes**
  (saturación de contexto/tools) como **dos nodos en serie**. Componente ≠ agente.
- **Supervisor híbrido por defecto:** reglas para precondiciones, modelo solo en ambigüedad real.
  Presupuesto de enrutado y trail obligatorios. Sin ambigüedad real → sin supervisor.
- **Escalera de comunicación:** estado compartido → handoff (impuesto medido, frente al híbrido) →
  mensajes (frontera de proceso). Por frontera, no global.
- **Reducers:** política explícita por campo; en paralelo, combinar + deduplicar + ordenar estable.
- **HITL:** señales deterministas primero; error ≠ revisión; precisión y recall del gate; checkpointer
  solo con aprobador distinto o pausa tras trabajo caro (si no, turno); nodo de interrupción sin
  efectos laterales; guardar el par propuesta–decisión.
- **Señal de incertidumbre según el tipo de salida:** juicio con error caro → competición con evidencia
  distinta; factual/anclada en fuente → verificación determinista, luego self-consistency; error
  barato → ninguna. Converger ≠ acertar. *(Competición: not exercised here.)*
- **Seguridad de acciones** (condición: existe una tool con efectos; señal: se añade la primera):
  privilegio en credenciales, guardia determinista ligado a la ejecución, irreversibles al humano,
  auditoría con denegaciones. Leer tampoco es inofensivo en corpus multi-cliente. *(Not exercised
  here.)*
