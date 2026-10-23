# S12 — Agentes: anatomía, patrones, function calling y coste

> Resumen provisional (paso 2). Teoría de los seis artículos + lectura sobre RAG-ECSS + señal extraída
> de la referencia `generation_20261019-141324` + sesión en vivo.
> Pre-implementación: nada de lo que aquí figura como propuesta está validado en código ni medido.

---

## 1. De pipeline a agente: la decisión raíz

**Tres niveles** (Anthropic, *Building Effective Agents*; Barry Zhang):
- **Tarea** — una llamada al modelo.
- **Workflow** — varias llamadas en un flujo de control **que escribes tú**. El modelo rellena huecos.
- **Agente** — el modelo **elige el siguiente paso** en un bucle a partir de lo que observa. Tú
  defines el objetivo y las barreras.

La diferencia es una sola línea (`model.decide`): quién controla la fontanería. Coste, latencia,
testabilidad y observabilidad se derivan de ahí.

**Qué compra un agente:** solo **orquestación adaptativa**. No compra mejor retrieval, ni mejor
generación, ni "más inteligencia". Si la forma del problema es fija, no hay nada que comprar.

**El precio:** latencia (N vueltas), coste (exploración), no-determinismo (testing y reproducción más
difíciles), errores que se componen (un mal paso 2 contamina los siguientes) y deuda de
observabilidad (hay que trazar decisiones, no solo entradas y salidas).

**Criterios del material:**
1. ¿Se puede mapear el árbol de decisión de antemano? → workflow.
2. ¿La forma es variable de maneras que no se pueden enumerar? → territorio de agente.
3. ¿El valor por tarea justifica el multiplicador? (alto volumen y bajo valor → workflow).
4. ¿Cuánto cuesta un error y se puede verificar? (solo lectura, validación de salida, humano en puntos
   críticos).
5. ¿El modelo domina el dominio? Si no, la agencia solo produce fallos más elaborados.

Caso canónico donde el agente se gana el sitio: agentes de código (problema ambiguo, valor obvio,
modelos competentes, resultado verificable con tests).

**Patrón de adopción:**
- El pipeline sigue siendo el camino por defecto.
- El agente entra como capa de decisión **por encima**, no como sustituto: los pasos del pipeline se
  promocionan a tools.
- Un router barato decide la vía para cada petición.
- Todo queda dentro del servicio IA, detrás del mismo contrato.

**Lectura crítica: falta el escalón intermedio.** El material plantea una dicotomía entre pipeline
fijo y agente. El caso estrella (descomponer en N componentes, buscar cada uno, consolidar) es
**orchestrator-workers**, que la propia guía de Anthropic clasifica como *workflow*: una llamada con
salida estructurada devuelve el plan (`subqueries[]`), el código hace el fan-out en paralelo y una
llamada final sintetiza. La forma varía según la entrada, pero la fija **una sola** llamada y la
ejecuta el código.

**Criterio refinado (candidato a la Guide):** que la forma sea variable no basta. **El bucle agéntico
se justifica cuando el siguiente paso depende de observar el resultado del anterior.** Si el plan se
puede fijar al principio, es un workflow de forma dinámica. "Si la búsqueda devuelve poco, reformula"
también es expresable como regla determinista (umbral → reformular una vez).

---

## 2. Anatomía del bucle

**Esqueleto ReAct** (Yao et al., 2023): Thought → Action → Observation, en bucle, con **condición de
parada** (respuesta final, máximo de pasos o presupuesto agotado). La parada es parte del esqueleto.

**Órganos:**
- **Razonamiento** — en los modelos de razonamiento, el Thought pasa a estar *dentro* del modelo (no
  se escribe ni se ve). Para observabilidad quedan los *reasoning summaries*.
- **Planificación** — por adelantado (auditable, rígida) o continua (adaptable, difícil de
  presupuestar). Forzar un plan explícito es una palanca de diseño.
- **Acción** — function calling. El modelo emite una intención y el código decide si la ejecuta.
  Mínimo privilegio. Las acciones irreversibles se validan o pasan por un humano.
- **Observación** — su calidad gobierna la siguiente decisión. Lo justo, con IDs estables. **Los
  errores son observaciones** y deben ser informativos ("1 coincidencia débil, baja confianza").
- **Handover** — a un humano (`needs_review` en el contrato, que el backend enruta) o a otro agente.
  Necesita un contrato explícito: qué estado se transfiere y quién decide.
- **Estado** — lo mantienes tú y es la traza. **Engorda**: la vuelta *n* reenvía las *n−1*
  observaciones anteriores.

**Lectura crítica:**
- **Estado del agente ≠ historial de la conversación.** Dentro de un turno del usuario hay *k* pares
  tool_call/tool_result. Consecuencias:
  - (a) la ventana del historial se mide en turnos anclados al mensaje del usuario, no en mensajes;
  - (b) truncar nunca puede separar un par tool_call/tool_result (el proveedor lo rechaza);
  - (c) el tráfico de tools de turnos pasados probablemente no debe viajar en el historial: basta con
    la respuesta final y sus citas.
- **Un reasoning summary no es auditoría.** No garantiza que refleje la decisión real. Lo auditable
  son las acciones, las observaciones y la salida estructurada. En RAG la trazabilidad vive en las
  **citas verificadas**, no en el razonamiento.
- **El razonamiento no es portable entre proveedores.** En bucles con tools, los proveedores exigen
  devolver su artefacto de razonamiento en el turno siguiente. Un fallback a mitad del bucle mezcla
  estados incompatibles. Detalles exactos por API: verificar en el brief, no de memoria.
- **Handover declarado por el modelo frente a decidido por el código.** La confianza que declara un
  modelo está mal calibrada. **Si existe una señal determinista para escalar, el handover lo decide el
  código.** La autodeclaración del modelo complementa, nunca sustituye.
- **Observación delgada frente a cita literal (específico de RAG).** Una observación de solo IDs
  rompe la verificación literal, porque el texto citado tiene que estar en contexto. Hay dos diseños:
  `search` devuelve chunks completos, o `search` devuelve cabeceras y `read_clause(id)` trae el texto
  (más fino, una vuelta más, y el modelo decide qué entra en el contexto citable, lo que afecta al
  recall).
- **Contexto citable en un agente** = unión de lo recuperado en todas las vueltas. La invariante
  "todo lo citado fue inyectado" sigue siendo exigible, y la traza debe poder decir en qué vuelta
  entró cada chunk.

---

## 3. Function calling: el contrato

**El modelo no ejecuta nada.** Cuatro tiempos: declarar tools → el modelo pide (nombre + argumentos)
→ el código ejecuta → se devuelve el resultado. Es una interfaz tipada cuyo cliente es un modelo.

**Anatomía de una tool:** nombre (con namespace cuando el conjunto crece), **descripción** (la palanca
principal), parámetros en JSON Schema (enums, campos requeridos, descripción por campo) y `strict`
(forma garantizada, sentido no).

**Mecánica:**
- La salida es una lista de items tipados.
- El `call_id` empareja cada petición con su resultado.
- **Llamadas paralelas:** una respuesta puede traer varias. Se ejecutan todas (`gather`) y se
  devuelven juntas en una sola continuación. Suponer que siempre hay exactamente una es un bug clásico.
- El mismo contrato tiene distinta forma según el proveedor: `function_call`/`function_call_output`
  (OpenAI Responses), `tool_use`/`tool_result` (Anthropic), y en Chat Completions las tools van
  anidadas bajo `function`. La diferencia de transporte se aísla en un adaptador o un agregador.

**Diseño de tools:**
- La descripción es la interfaz: cuándo usarla, cuándo no, con qué granularidad y qué devuelve.
- Resultados de alto valor (no volcados).
- Errores informativos.
- Validar los argumentos de lo que puede hacer daño.
- Pocas tools con fronteras nítidas. Señal de alarma: tener que explicar "no uses esta, usa la otra".

**Lectura crítica:**
- **Estado en el servidor (`previous_response_id`) frente a estado en el cliente.** El primero ata la
  conversación a un proveedor y es incompatible con el fallback. Nodo de la Guide: para un wrapper
  agnóstico con fallback, el estado vive en el cliente y el coste del reenvío lo mitiga el caché.
- **Nuestro adaptador ya existe: LiteLLM**, que normaliza al formato de Chat Completions (rol `tool`,
  `tool_call_id`). Se copia el contrato, no el snippet (que usa la Responses API).
- **Function calling frente a salida estructurada.** El material no trata cómo conviven. Dos vías:
  - **`submit_answer`**: la respuesta final como una tool cuyo schema es la salida estructurada,
    forzada en la última vuelta. Es portable y sirve además como salida degradada al llegar a
    `MAX_STEPS`.
  - **Tools más `response_format` a la vez**: el artículo 4 lo hace en OpenAI. La paridad en Anthropic
    y a través de LiteLLM está por verificar.
- **Las descripciones de tools son prompt:** se versionan, se miden con el arnés y forman parte del
  **prefijo cacheado** (cambiarlas o reordenarlas invalida el caché, así que deben serializarse de
  forma determinista).
- **Un enum que enseña, generado desde el catálogo**, no escrito a mano. Los IDs de cláusula no caben
  en un enum: `pattern` + validación contra el índice + error informativo con las cláusulas más
  cercanas.
- **El resultado de una tool es una entrada no confiable** (*prompt injection* indirecta). Condición
  de aplicabilidad: relevante si las tools devuelven contenido de terceros no controlado. Bajo en ECSS
  (corpus congelado y de confianza). Relevante en proyectos que procesan documentos de clientes.

---

## 4. El bucle hecho a mano

**Piezas:** tools + registro nombre → función (desacopla el catálogo del bucle) + modelo orquestador
+ estado/traza. El despacho captura los fallos y los devuelve como observación. Bucle: recoger todas
las `function_call` → `gather` → devolver las salidas con su `call_id` → repetir hasta que no haya
llamadas o se llegue a `MAX_STEPS` (`for/else` → `max_steps_exceeded`). La salida final se ciñe a un
modelo Pydantic: **la no-determinación se queda dentro del bucle y el contrato de salida es
determinista.** El agente vive detrás de un endpoint y el backend solo ve `status` + resultado.

Lo que enseña hacerlo a mano: el manejo de errores es tuyo, la observabilidad no viene gratis y sabes
qué te automatizaría un framework (tema de S13).

**Lectura crítica:**
- **Antipatrón: el prompt impone una invariante.** "Always run validate_estimate before the final
  answer" deja la validación en manos del modelo, que a veces se la salta. **Lo invariante va en
  código, después del bucle.** (Reaparece en el artículo 5 con "only call this after…". Ver §5.)
- **`MAX_STEPS` limita las vueltas, no el trabajo.** Una vuelta con 12 llamadas paralelas cuenta como
  un paso. Límites necesarios: vueltas + tool calls totales + **tokens acumulados** + tiempo total.
- **El `except Exception` convierte los bugs en observaciones** (degradación silenciosa, mensajes
  internos que llegan al modelo). Tres clases de error:

  | Clase | Ejemplo | Tratamiento |
  |---|---|---|
  | Transitorio | timeout, 429 | Reintenta el código; el modelo no lo ve. |
  | Error del modelo | argumento inválido, cláusula inexistente | Observación informativa redactada. |
  | Bug interno | excepción no prevista | Falla en voz alta: se loguea y se aborta o degrada. |

  Además, en el snippet `json.loads(arguments)` queda fuera del `try`.
- **`max_steps_exceeded` sin respuesta.** Tras N vueltas de evidencia no se entrega nada. En RAG es
  mejor degradar: forzar `submit_answer` con lo acumulado y que `not_found[]`/`status` reflejen lo
  que faltó.
- **La traza no es parte del contrato público.** Va a la observabilidad, no a la API. En RAG la
  traza que ve el usuario son las citas.
- **Testing del bucle:** con un **modelo falso que responde con un guion** (tool calls, errores,
  agotamiento), determinista y sin red. Con un bucle, una llamada real accidental en tests se
  multiplica por N vueltas.
- **El propio ejemplo se resuelve como workflow:** `calculate_estimate` es determinista y la
  reformulación es una regla. Al modelo solo le queda descomponer, que es una llamada de plan.

---

## 5. Patrones y diseño de tools

**Tres ejes de forma** (el material admite que no son independientes; son lentes, no un catálogo):
- un solo paso o iterativo;
- reactivo (decide a la vista de la última observación) o proactivo (anticipa hacia el objetivo);
- plan fijo (auditable, rígido) o planificación dinámica (adaptable, imprevisible).

**La forma se decide por petición:** un router barato manda lo simple al camino corto y lo complejo
al iterativo.

**Las tools dirigen el comportamiento.** Si el agente se porta mal, la causa suele estar en una
descripción o en las fronteras entre tools. Las descripciones se iteran como prompts. Bucle de mejora:
un conjunto representativo de entradas → leer trazas → localizar la causa → arreglar → re-ejecutar.

**Lectura crítica:**
- **"Arreglar y comprobar que la traza mejora" sobre un puñado de casos es iterar el prompt a
  ciegas.** Leer trazas sirve para **diagnosticar** (formular hipótesis). Aceptar el cambio requiere
  el arnés sobre el set completo, un criterio preregistrado y una mejora por encima del ruido. Si no,
  se sobreajusta a la anécdota. Corolario: cada pasada del arnés debe grabar la **versión de las
  definiciones de tools**.
- **Invariantes en código, preferencias en las descripciones** (segunda aparición del antipatrón,
  candidato a principio). Una precondición se comprueba en la propia tool, que devuelve un error
  informativo. La descripción puede mencionarla para ahorrar vueltas. Si la precondición es "todo lo
  anterior hecho", probablemente no es una tool sino el paso de código posterior al bucle.
- **"Casi nunca es culpa del modelo" es una generalización excesiva.** El orden de diagnóstico tiene
  sentido (primero lo barato), pero la competencia del modelo en el dominio es una causa real con un
  orquestador pequeño y jerga especializada (contradice el criterio 5 del artículo 1).
- **Los ejes dependen del caso.** Para transcripciones con sorpresas, el material elige reactivo con
  plan ligero. Para preguntas ECSS la forma suele leerse en la propia pregunta (ver §8).
- **Hueco del temario: los patrones canónicos no aparecen** (prompt chaining, routing,
  parallelization, orchestrator-workers, **evaluator-optimizer**). Evaluator-optimizer es nuestro
  anclaje Actor/Critic/Boss y no se trata en ningún artículo. Lo cubrimos nosotros en §7.

---

## 6. Coste de un agente

**Cuatro fuentes de sobrecoste:** más llamadas · **contexto que crece en cada vuelta (dominante)** ·
razonamiento repetido en cada vuelta · exploración y reintentos.

Cuenta ilustrativa del material: pipeline ~8k tokens frente a agente ~48k (≈6×). Casi todo es entrada
reenviada. Regla mental del ecosistema: ~0,10 USD ≈ 30–50k tokens (depende del modelo).

**Medición:**
- registro de costes que acumula `usage` en cada vuelta (el razonamiento ya va incluido en la salida;
  no se suma aparte);
- coste por paso (revela el crecimiento del contexto);
- **distribución (p95)**, no solo la media: la cola larga la causa un agente confundido;
- **ratio frente al pipeline** con las mismas entradas;
- **atribución por tool** (qué observación infla el contexto).

**Palancas, por orden de impacto:** enrutar · adelgazar el contexto · acotar la cola (presupuesto por
ejecución → revisión) · ajustar modelo y razonamiento a cada sub-tarea · cachear resultados
deterministas.

**Lectura crítica:**
- **Falta el caché de prefijo**, la palanca natural del coste dominante: en un bucle cada vuelta es la
  anterior más un sufijo, y lo reenviado se factura a una fracción del precio. Cifras por proveedor:
  verificar.
- **Tensión entre compactar y cachear** (no aparece en el material). Resumir o descartar
  observaciones antiguas reescribe el historial y rompe el prefijo cacheado desde ese punto. Con buen
  caché conviene crecer solo por el final. Compactar compensa cuando el contexto se acerca al límite o
  las observaciones son enormes.
- **El crecimiento es cuadrático:** con observaciones de tamaño parecido, la entrada acumulada crece
  con el cuadrado de las vueltas. Duplicar `MAX_STEPS` cuadruplica el peor caso de entrada, así que el
  límite por tokens acumulados importa más que el de vueltas.
- **El registro de costes está atado a OpenAI y no contempla el caché:** en Anthropic, escribir en
  caché cuesta más que la entrada normal y leer de él, mucho menos. Ignorarlo da costes erróneos. Con
  LiteLLM, el coste sale del mapa de modelos local, que hay que mantener al día.
- **Ignora la latencia.** Cada vuelta es una llamada completa con razonamiento. Las tools paralelas
  solo recortan el tiempo de las tools. Las métricas de coste (por paso, p95, ratio) aplican igual a la
  latencia.
- **Coste bruto frente a coste por calidad.** "¿El valor justifica el multiplicador?" pasa a ser regla
  de aceptación con un criterio preregistrado: ganancia de calidad pareada > ruido, sin violar la
  asimetría de omisión, y multiplicador de coste/latencia por debajo de un **techo fijado antes de
  medir**.
- **Encuadre de volumen.** El ejemplo de millones de tareas no aplica a un RAG de pocas consultas y
  mucho valor por respuesta. En ese caso el coste rara vez bloquea; bloquean la latencia y la falta de
  evidencia.

---

## 7. Lo que el temario no cubre y añadimos

| Tema | Posición propuesta |
|---|---|
| **Evaluator-optimizer / Actor-Critic-Boss** | Se mantiene el anclaje de S11. Critic y Boss siguen en código (verificación determinista). Un crítico LLM solo entra con una señal de errores de soporte que la capa literal no ve; en 125 respuestas no hubo ninguno. Lo mismo vale para la reflexión o el auto-refinado: cada vuelta extra se paga en latencia. |
| **Evaluación de agentes** | El arnés de generación puntúa la respuesta final. Con un bucle hacen falta además **métricas de trayectoria**: vueltas, tool calls, coste y latencia por pregunta (mediana y p95), tasa de no convergencia y en qué vuelta entra cada chunk citado. Las trayectorias se graban como artefactos inmutables y re-puntuables. |
| **Escalón workflow dinámico** | Orchestrator-workers / plan-and-execute como paso intermedio obligatorio en el árbol de decisión (§1). |
| **Portabilidad** | Estado en el cliente. El fallback solo puede ocurrir antes de la primera vuelta (o reinicia el bucle). Respuesta final vía `submit_answer`. |
| **Historial** | Ventana en turnos anclados al usuario con pares tool atómicos (cobra la deuda `-2N`, haya bucle o no, en cuanto exista el rol `tool`). |

---

## 8. Lectura sobre RAG-ECSS

**Hoy RAG-ECSS es un workflow puro:** `retrieve(routing filter) → generate_structured → verify →
compose`. Tres órganos del agente ya existen y son deterministas:
- handover (`needs_review` decidido por veredicto);
- observaciones informativas (`not_found[]`);
- validación de la salida.

Lo que falta es exactamente lo que el material dice que solo se compra si hace falta: **la decisión
del siguiente paso**.

**Tipos de pregunta ECSS según su forma:**
- **Puntual / un estándar** → forma fija. Pipeline actual.
- **Comparativa** (E-40 frente a Q-80 sobre X) y **aplicabilidad** ("¿qué estándares aplican a…?") →
  forma variable pero **predecible al leer la pregunta**. Workflow con plan por adelantado y fan-out
  por estándar o faceta. Sin bucle.
- **Cadena de referencias cruzadas** (la cláusula A remite a B, en otro estándar) → el único caso de
  verdad **reactivo**. Dos soluciones:
  - (a) expansión determinista de referencias en el retrieval (línea `orphans` / grafo diferido de S10);
  - (b) bucle con `search` + `get_clause`.

  Por sencillez, (a) va primero salvo evidencia de lo contrario.

**Router ya disponible:** el routing `filter` detecta qué estándares menciona la pregunta. Número de
estándares ≥ 2 o marcadores de comparación → camino de plan. Es una señal determinista que ya existe,
sin llamada extra.

**Primitiva valiosa con o sin agente:** `get_clause(standard, clause_id)`, una consulta directa por ID
sin embeddings. La usarían tanto la expansión determinista como una tool.

**Coste frente a latencia:** una pasada del arnés con agente seguiría costando céntimos. Lo que
bloquea es la **latencia** (mediana 12,8 s y p95 20 s sin bucle) y la **evidencia**.

### Señal en la referencia `generation_20261019-141324`

**El recall de 0,91 no mide la forma.** Es *relevante citado / relevante inyectado*: sus fallos son
omisiones del generador sobre lo que ya tenía en contexto, no búsquedas que faltaron. Una capa de
decisión no lo mejora.

**La señal de forma está en el estado servido y en los `not_found`:**
- 33 de 42 ejecuciones se sirven como `partial` (31 declaradas y servidas así, 2 degradadas por el
  código);
- 65 aspectos `not_found`.

Lectura **provisional** de esos aspectos (hipótesis, no verificadas contra el corpus):

| Grupo | Consultas | Hipótesis | ¿Lo arregla orquestar? |
|---|---|---|---|
| **A. Fuera del corpus** | q08 (ECSS-Q-ST-30-02C, SFMEA), q12 (política de riesgos de proyecto), posiblemente q11 (redacción de requisitos) | El contenido vive en estándares que no están en el subconjunto | No. La abstención es correcta. Señal para ampliar el corpus, no para orquestar. |
| **B. En el corpus pero no recuperado** | q01 (revisiones y entregables por revisión), q14 (ejecución de OBCP, E-ST-70-41), q06 (cat. B frente a C), q09 (reutilización en Q-80 y E-40), q13 (carga en órbita) | Segundo salto (referencia cruzada) o cobertura por faceta/estándar insuficiente | **Candidato.** Fan-out (q06, q09) o referencias cruzadas (q01, q14). |
| **C. Probablemente ECSS no lo dice** | q05 (versión exacta por build), q10 (umbrales numéricos) | El estándar no lo especifica | No. `not_found` correcto. |

**Conclusión provisional:** hay señal candidata (grupo B), pero mezclada con abstenciones correctas
(A, C). Sin clasificar cada aspecto contra el corpus, cualquier capa de decisión se aceptaría a ciegas.
Además, el golden set v2 tiene solo dos consultas por estrato y ninguna de cadena de referencias
explícita: no basta para preregistrar un criterio con potencia.

---

## 9. Decisiones S12

### Decididas
- **Primero `pre-S12`** (brief escrito en `sessions/pre-S12/`): historial por turnos y bloqueo de red
  en la suite de contrato. Sin rol `tool` ni modelo falso con guion: llegan solo si hay bucle.
- **La clasificación A/B/C de los `not_found` la hace CC** (búsqueda dirigida en el corpus). Salva la
  comprueba en la UAT de ese brief.
- **Brief 1 cerrado (diagnóstico):** con la clasificación corregida en la UAT, B1 = 9 consultas (8 sin el
  espurio de q10) y B2 = 0. Por la regla preregistrada → **expansión determinista de referencias +
  `get_clause`**. Fan-out por estándar descartado (sin señal).
- **Brief 2:** solo punteros a cláusula + medición de `bounded` en la misma pasada. Punteros a
  estándar entero fuera (el bucle o el filtro por estándar se deciden tras medir). Criterio sobre
  aspectos resueltos del diagnóstico (≥ 3/6 en ≥ 2/3 ejecuciones) con guardas de recall (−0,03),
  `verified` 1,00, latencia +20 % y coste ×1,5. La precisión se reporta, no es guarda. Tope de 5
  cláusulas añadidas, marcadas ➕, las primeras en descartarse por presupuesto.
- **Resultado del brief 2:** ninguna configuración pasa el criterio preregistrado. `xref`
  implementado y **apagado** (recall −0,08 por inflación del denominador; no se reinterpreta a
  posteriori). **`bounded` pasa a default como excepción registrada** (sin empeorar ninguna guarda,
  mejora clara en la UAT del SDP, estrato fuera del golden set; se confirma cuando exista ese
  estrato). Sin bucle agéntico en S12: el contenido llega por la vía determinista; lo que falló fue
  la métrica y los topes, no la falta de decisión del modelo.
- **q12-a3 = B1-standard** (decisión del chat tras la verificación V2).
- **Brief 4:** `bounded` por defecto + verificación, protocolo 1.2 (respuestas principales frente a
  relacionadas, puntero literal comprobado con el parser del brief 2, `answered_as_phrased`).
- **Las métricas corregidas del arnés** ("resuelto" por `[n]` con la cita literal aparte, recall
  sobre relevantes recuperadas + recuentos absolutos) se preregistran para el brief de ampliación del
  golden set.
- **`xref` encendido por defecto (`clause`, sobre `bounded`), decisión de Salva tras la UAT 4:** la
  calidad de respuesta va antes que la latencia. Es una excepción por prioridad de producto: el
  techo de latencia deja de ser guarda y pasa a objetivo, y se ataca por otras vías (UI de progreso,
  razonamiento `low`, caché). Brief 5.
- **Grafo de referencias: diferido con puerta obligatoria antes de M6.** Tres tipos de arista con
  casos: (1) referencias textuales explícitas (hoy `xref`, al vuelo, un salto); (2) estructurales
  implícitas (fila de la matriz de aplicabilidad D.2 ↔ cláusula, V12; padre/hermana, V9; requisito ↔
  DRD); (3) por nombre (SCF → M-ST-40C anexo E, SCMP → anexo A; V3/V7). Más la preferencia por
  autoridad normativa frente a handbook (q09/V10). Punto de partida: los artefactos de S06
  (`data/processed/references.json`, `graph.graphml`) y el parser de `xref`.
- **`MAX_HISTORY_TURNS` ≥ 1** (decidido tras pre-S12; implementado en el brief 1).

### Diferidos con señal (nuevos en S12)
- **Prompt por tipo de pregunta (explicativa frente a requisito).** UAT de pre-S12: las respuestas a
  preguntas abiertas citan en vez de sintetizar, y algunas viñetas `not_found` parecen forzadas (1
  espuria de 27 en el diagnóstico: q10). Se mide aparte para no confundirlo con la expansión, y
  necesita antes un estrato de preguntas explicativas en el golden set.
- **La consulta de retrieval arrastra las preguntas previas aunque cambie el tema.** En la UAT de
  pre-S12, "What is the SDP…" tras una pregunta de seguridad no recuperó el anexo O (DRD del SDP),
  que sí está en el corpus. Candidato: detectar el cambio de tema antes de componer la consulta con
  el historial.
- **Punteros a estándar entero** (q08, q10, q12, q14): el filtro por estándar llega al top 10 en 4
  de 7. El bucle con tools solo tendría sentido para el resto.
- **Herramienta de validación externa** (en construcción por CC): verificador sin acceso al código
  que contrasta juicios contra los documentos. Sustituye la parte pesada de la UAT de clasificación.

### Propuestas (pendientes de confirmar antes del brief)
- **Brief `pre-S12` (infraestructura, independiente del agente):**
  - historial por turnos anclados al usuario con pares tool atómicos (cobra la deuda `-2N`);
  - bloqueo de red en la suite de contrato + modelo falso con guion para el control de flujo.

  Pasa de diferido a prerrequisito si hay bucle.
- **Brief 1 — diagnóstico de forma (sin llamadas reales o mínimas):** clasificar cada aspecto
  `not_found` de la referencia en A/B/C contra el corpus. Para los B, decir si se alcanzan por
  referencia cruzada desde lo ya recuperado o por fan-out por estándar/faceta. Es el instrumento que
  decide si hay algo que orquestar.
- **Si B pesa:** primitiva `get_clause` + la solución más simple por tipo:
  - fan-out con plan para comparativa/aplicabilidad;
  - expansión determinista de referencias para las cadenas.

  El bucle con tools solo si lo determinista no basta.
- **Ampliación del golden set** (protocolo 1.2) con estratos de cadena de referencias, comparativa y
  aplicabilidad. Probablemente sale del diferido porque es el instrumento de M5.
- **Si hay bucle:** estado en el cliente, `submit_answer` como salida final y degradada, fallback solo
  antes de la primera vuelta, límites multidimensionales (vueltas, tool calls, tokens acumulados,
  tiempo), tres clases de error, verificación literal fuera del bucle sobre la unión de lo recuperado,
  métricas de trayectoria en el arnés y techo de latencia/coste preregistrado.

### Abiertas
- ¿El filtro de estándares lo sigue eligiendo el código (routing `filter`) o pasa a ser un argumento
  de tool que elige el modelo? Inclinación: código, por el principio de §2.
- ¿`search` devuelve chunks completos o cabeceras + `read_clause`? (Tensión entre observación delgada
  y cita literal.)
- Tools + `response_format` simultáneos: ¿paridad en Anthropic y en LiteLLM 1.106.0? Verificar antes
  de descartar `submit_answer`.
- ¿El arnés separa ya los tokens de escritura y lectura de caché en el coste? La columna `cache w/r`
  sugiere que sí. Confirmar con CC.
- ¿Fan-out por estándar dentro del retrieval (determinista, sin LLM) o con una llamada de plan
  (subconsultas reformuladas)?

### Diferidas
- Framework frente a orquestación interna, estado en grafo → S13.
- Multi-agente, human-in-the-loop operativo, sandboxing → S14.
- Ampliar el corpus con los estándares del grupo A (Q-ST-30-02, gestión de riesgos…) → `corpus/`
  sigue congelado. Se registra como señal.
- Compactación del historial en bucles largos → solo si aparece presión de contexto (tensión con el
  caché).

---

## 10. Sesión en vivo

**Ideas destacadas.**
- **No pasar a agéntico por hype.** Solo cuando resuelve algo que de otra forma es incómodo, difícil o
  caro en tiempo humano. Si ya hay una solución tradicional simple y eficaz, se queda. (Coincide con
  la decisión raíz de §1.)
- **La decisión no es solo técnica:** cuenta también cómo prefieren interactuar los usuarios.
  (Matiz para RAG-ECSS: un ingeniero que consulta normativa espera una respuesta citada en segundos,
  no un proceso en varias vueltas. Refuerza que la latencia sea un criterio de aceptación.)
- **Validación humana en procesos sensibles** y **diseño conservador para reducir alucinaciones.**
  (En RAG-ECSS ya existe: `needs_review` decidido por veredicto y política estricta de citas.)
- **Ni un agente que lo haga todo ni demasiados agentes** (coste de mantenimiento). Avance de S14.
- **Hacer el bucle a mano como ejercicio** para entender qué abstraen los frameworks. Avance de S13.
- Primero los conceptos, después la implementación.

**Preguntas y respuestas.**
- *¿Context engineering, loop engineering y graph engineering son enfoques distintos?* — Se solapan y
  conviven en el mismo agente: un agente en grafo también gestiona contexto y bucles. La terminología
  cambia rápido; importa qué aporta cada mecanismo, no la etiqueta. Los grafos se ven en S13.
- *¿Qué se usa en producción real?* — Más habitual: servicios agénticos en Python con LangChain y
  LangGraph, integrados como parte de una solución de software, más que grandes sandboxes de
  agentes. Hermes, de forma limitada y en equipos avanzados. (Es la opinión del mentor, no un dato de
  mercado. Framework frente a orquestación interna se decide en S13 con nuestros criterios.)
- *¿Python es requisito?* — No, es una recomendación por el ecosistema de datos/ML y de librerías.
- *¿Cómo se aíslan los permisos de los agentes?* — Dependen de las tools y recursos que se le den.
  Una opción es un contenedor por agente con permisos y recursos limitados. Se enlaza con la
  orquestación en LangGraph. (Es el principio de mínimo privilegio del artículo 2 llevado a
  infraestructura. Sandboxing: S14. En RAG-ECSS, todas las tools candidatas son de solo lectura sobre
  un corpus propio.)
- *¿Siempre merece la pena pasar de un flujo determinista a uno agéntico?* — No (ver ideas
  destacadas). Fue el mensaje central de la sesión.
- El repositorio del proyecto del máster es público; el mentor tiene un fork con guiones y agentes
  adicionales que podría compartir.
