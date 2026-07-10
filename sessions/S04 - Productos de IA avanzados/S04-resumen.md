# S04 — Resumen de sesión

> **Tema del máster:** productos avanzados sobre CAG — abstracción de prompts, plantillas
> dinámicas, extracción estructurada, guardrails, y (bloque extra) cacheo semántico.
> **Estado:** paso 2 del ciclo (resumen provisional, pre-implementación). No graduado a la Guide.
> **Foco elegido para el brief:** anclaje de S03 (prompt como artefacto + endpoint que expone
> prompt/contexto activo) + robustez de prompt (scope explícito, permiso "no sé", políticas de
> fallo declaradas). El resto de S04 se difiere a M2+ con razón registrada.

---

## 0. La lente que ordena toda la sesión

S04 gira sobre una única pregunta arquitectónica: **¿dónde vive el prompt?** La respuesta madura
es "en el backend, versionado, como artefacto de software" — no en el textarea del usuario. Todo
lo demás de la sesión (plantillas, structured output, guardrails, caché) son capas que se apoyan
en esa decisión.

Para RAG-ECSS esto es en gran parte **validación retroactiva**, no novedad: el movimiento "el
prompt vive en el backend" ya lo hicimos en S02/S03 (`build_system_prompt` íntegro en el backend,
el frontend solo manda `question`, thin-client rule en la Guide). S04 nos da el vocabulario y los
corolarios de una decisión que ya encarnábamos.

---

## 1. De interfaz conversacional a interfaz de producto

### Idea valiosa
El chat es un **default heredado de ChatGPT**, no una decisión de diseño. Cuando delegas el
prompting al usuario, la calidad del producto se vuelve función de cómo de bien promptea cada
persona — eso no es un producto, es una herramienta para usuarios avanzados. La alternativa es
**hornear en la interfaz** lo que hay que pedir y cómo.

### Dos movimientos independientes (que conviene no confundir)
- **A — dónde vive el prompt:** sacar instrucciones/scaffolding del frontend al backend. El
  usuario deja de escribir el prompt.
- **B — la forma del input:** sustituir el textarea libre por un formulario de campos finitos.

Son ortogonales. El estimator los cumple a la vez porque su problema es *cerrado*
(`project_type`, `detail_level`, `output_format` son enums). No todo producto puede hacer B.

### El espectro (no es binario chat/formulario)
`chat puro → chat con parámetros → formulario/acción → UI generativa`. La pregunta correcta ante
una feature no es "¿chat sí/no?" sino "¿dónde en el espectro encaja?".

### Posición de RAG-ECSS (decisión de producto, abierta)
Ni chat puro ni formulario: **híbrido — pregunta abierta en lenguaje natural + perillas
estructuradas**. La pregunta (núcleo del QA sobre corpus) es genuinamente abierta y no se reduce
a selectores. Pero alrededor hay parámetros finitos legítimos: formato de salida (narrativa /
lista de requisitos / matriz de cumplimiento), scope de estándares (CORE/RELATED/ADJACENT),
nivel de detalle.

**Matiz propio del dominio, no visible en el estimator:** esos parámetros están **escalonados por
módulo**. El único accionable hoy (M1, CAG estático) es formato de salida. El más valioso —
filtrar scope por estándar — depende de que exista retrieval (M2): no puedes ofrecer un selector
de "buscar solo en ECSS-E-ST-40" sin corpus que filtrar.

---

## 2. Plantillas de prompts / prompting desde backend

### Idea valiosa
El prompt como **artefacto de software** operacionalizado: vive en el repo, se versiona con git,
se testea, se revisa en PR. Anatomía en tres componentes con ciclos de vida distintos:
- **estructura fija** (rol, instrucciones, formato, few-shot) — el contrato con el modelo;
- **variables** (contenido por request: descripción, contexto RAG, historial);
- **parámetros** (modos: nivel de detalle, formato, idioma).

El *template* une los tres; el motor de plantillas (Jinja2 en el material) sustituye y compone.

### Prácticas que sí transfieren (independientes del motor)
- **Prompt en archivo separado del código Python.** Beneficio real: PR de solo-prompt legible,
  editable sin abrir Python. Se obtiene moviendo el prompt a `app/prompts/...` como string —
  **sin necesidad de motor de plantillas**.
- **Fallo ruidoso ante variable ausente** (`StrictUndefined` o equivalente): una variable
  declarada y no provista rompe en render, en vez de renderizar cadena vacía y mandar un prompt
  malformado en silencio. Es la misma política fail-fast de `Settings`.
- **Test estructural del prompt renderizado**: barato, en CI, sin coste de API. Verifica que el
  prompt final contiene lo que debe. Adoptable aunque el prompt sea texto plano.

### XML tags vs Markdown como estructura interna
Anthropic entrena atendiendo a XML tags; OpenAI tiende a Markdown. Ambos entienden ambos. Para
nosotros la restricción es más estricta que la del material: si el delimitador se optimiza para un
proveedor, el prompt deja de ser **provider-agnostic** — principio ya en la Guide, verificado en
S03 contra Anthropic y OpenAI. Cambiar de estilo obliga a **re-verificar**, no a asumir.

---

## 3. Extracción de datos estructurados

### Idea valiosa
El LLM como **función con tipo de retorno**, no como caja de texto. Cuando un producto *consume*
la salida, el texto libre obliga a un parser intermedio frágil que rompe en silencio. Invertir la
lógica: decirle al modelo qué forma devolver (JSON Schema) en vez de adivinar qué devolvió.
Pydantic como **single source of truth** (clase + validación + JSON Schema + doc OpenAPI de una
declaración). Tres mecanismos equivalentes: OpenAI Structured Outputs (nativo), Anthropic tool-use
forzado (idiomático), otros vía agregador. `Instructor` los unifica.

### El límite que importa para nosotros
**Structured output garantiza la *forma*, no la *verdad*.** Un `Citation` con shape perfecto no
dice nada sobre si la cláusula existe o si el modelo se la inventó. Es exactamente la distinción
ya en la Guide (Phase 2): **emitir citas ≠ verificar citas.** Structured output resuelve el
*transporte* de la trazabilidad; la *verificación* sigue siendo S11. El riesgo es que se *sienta*
como si resolviera trazabilidad (devuelve citas tipadas y limpias) cuando solo resolvió emisión.

### Dónde sí aporta, y cuándo
Aporta en el **eje de emisión**, y el valor real aparece en M2. Hoy (M1) nuestras `Citation[]` se
construyen en backend desde los metadatos del fragmento inyectado — sabemos qué citar, structured
output aporta poco. En M2, cuando retrieval devuelva *k* chunks y el modelo use algunos, "adjuntar
citas de todo lo inyectado" sobre-cita; ahí structured output empieza a ganarse el sitio (pedir al
modelo que declare *cuáles* chunks fundamentaron cada afirmación). Aun así solo mueve el problema
de "sobre-citar" a "confiar en lo que el modelo dice que usó" — sigue necesitando S11.

Decisión candidata: **diseñar el contrato `{answer, citations}` quizá ahora; activar el forzado en
M2.**

---

## 4. Guardrails y validación de outputs

### Idea valiosa
**La forma no garantiza el contenido.** Cuatro respuestas con schema perfecto que rompen el
producto: prompt injection, out-of-scope, PII filtrada, **alucinación con forma impecable**. Este
último (la cláusula inventada con cita perfecta) **es el modo de fallo central de RAG-ECSS**.

Matriz de validación: **input/output × sintáctico/semántico**. La fila sintáctica ya la cubre
Pydantic. La fila semántica es trabajo nuevo. **Defense in depth**: capas ordenadas barata→cara
(regex <1ms → Moderation ~50-100ms → retry de modelo 1-3s → LLM-as-judge), cada caso rechazado en
la capa más barata que lo capture.

### Las tres políticas de fallo (lo más accionable)
Cada guardrail declara explícitamente qué hace al disparar:
- **Exception** — aborta con error (violación grave: PII, injection, toxicidad).
- **Fix con retry** — pide corrección y reintenta (error recuperable: JSON malformado, campo
  ausente).
- **Filter / degrade** — devuelve respuesta segura por defecto (out-of-scope, confianza baja).

Regla: *"el error más común no es elegir mal, es no decidir"*. El code review pregunta "¿qué pasa
cuando dispara?" y la respuesta es una de las tres.

### Reparto de prioridades propio del dominio
El estimator carga el peso en Moderation + scope temático. RAG-ECSS invierte: Moderation sobre
corpus normativo es casi irrelevante; lo que importa es **validar que la respuesta está anclada y
no alucina cláusulas** — que el material *aparca* explícitamente para "cuando el RAG entre en
juego". Es decir: **el guardrail central de RAG-ECSS es S11, no S04.** S04 da el marco (matriz,
políticas, defense-in-depth); S11 construye la verificación de grounding.

### Capa 3 (robustez de prompt): lo único plenamente accionable en M1
Dar permiso explícito para decir "no sé", definir scope en el system prompt, exigir evidencia por
afirmación. **Coste de runtime cero.** Ya practicamos parte vía la regla de grounding ("responde
solo desde el material; si no está, dilo") — que *es* el "permiso para decir no sé". S04 la
reconoce como capa nombrada y la refuerza.

### Guardrails ↔ caché (orden crítico)
Refinamiento sobre la regla gruesa del material ("guardrails antes del cache"):
- **Input guardrails antes del lookup.** Si un hit se salta la validación de input, un atacante
  que sabe que cierto contenido está cacheado puede activar el hit y recibir la respuesta sin
  pasar moderación. El input guardrail protege del caché, no solo del LLM.
- **Output guardrails antes del `set`** (no solo "antes del hit"): solo se cachea lo ya validado,
  así un hit es seguro por construcción y no re-paga la validación cara.
- Cuidado con la política *filter/degrade*: decidir si la respuesta degradada se cachea (puede ser
  correcto si es determinista, o un problema si la degradación fue por fallo transitorio).

---

## 5. Cacheo semántico (bloque extra)

### Idea valiosa
Exact-match captura poco cuando la clave es texto natural humano (N formulaciones de una intención
→ N claves). El caché semántico compara *significados* vía embeddings + similitud coseno sobre un
threshold. El threshold es **decisión de producto** (agresivo 0.85 maximiza hits con riesgo de
falso positivo; conservador 0.95 lo elimina perdiendo hits). Disciplina: **log-only primero,
calibra con datos, bloquea después.**

### La cache key compuesta (valida S03)
Solución del material al problema de colisión de parámetros: **parte determinista (bucket) +
parte vectorial (embedding)**. El `build_bucket_key` (`version:project_type:detail_level:
output_format`) es la misma idea que nuestra clave de slots de S03. Confirma dos decisiones
nuestras:
- **`prompt_version` en la parte determinista invalida buckets viejos automáticamente al subir a
  v2** — el cross-link "versión de prompt = slot de invalidación" que ya teníamos.
- **TTL como red de seguridad, no mecanismo primario**; invalidación por versión (de prompt / de
  corpus) como mecanismo real.

### Reserva fuerte: contraindicado en corpus normativo
El caché semántico agrupa por **similitud superficial**, eje **ortogonal** a lo que nos importa
(**identidad de cláusula**). Ejemplo del dominio: *"criticidad B en ECSS-Q-ST-80C"* vs
*"criticidad C en ..."* son casi idénticas en embedding space pero apuntan a cláusulas distintas.
Un hit semántico serviría la norma equivocada, con cita de aspecto perfecto, saltándose la
generación entera (y por tanto cualquier guardrail del camino de generación). Es el escenario 4
de guardrails, introducido por la propia caché. El material lo roza con su propio ejemplo (Stripe)
y con "jurídico/médico/financiero → 0.95+", pero no cierra el punto: para nosotros no es cuestión
de subir el threshold, es contraindicación de raíz.

**Reabrible solo con precondiciones caras:** si las anclas normativas (estándar, criticidad)
migran a parámetros estructurados (el "bucket" determinista) y solo la intención en lenguaje
natural va al embedding, el riesgo baja. Pero eso presupone (a) mover anclas a perillas —la
decisión de espectro del bloque 1— y (b) M2. Hoy no se dan ninguna.

Además, adoptarlo ahora **adelantaría los embeddings (S07)**, rompiendo la regla de no anticipar
temario. El propio material trata su caché semántica como provisional pre-embeddings.

---

## 6. Decisiones de sesión

### Adoptar en M1 (foco del brief)
- **Prompt como artefacto separado del código** (archivo, no Jinja necesariamente).
- **Endpoint que expone el prompt activo + contexto CAG** — cobra el `# TODO` del sidebar de S03.
  Como **anillo alrededor de `build_system_prompt`**, no fuente de verdad nueva; devuelve el
  prompt *renderizado* que se enviaría, no el template.
- **Robustez de prompt (capa 3):** formalizar scope explícito + permiso "no sé" (parcialmente ya
  vía grounding).
- **Las tres políticas de fallo, declaradas explícitamente.** Hoy tenemos una implícita:
  grounding = filter ("si no está, dilo").
- **Test estructural del prompt renderizado.**

### Diferido / con precondiciones
- **Structured output `{answer, citations}`** — diseñar contrato quizá ahora, activar forzado en
  M2. Resuelve emisión, no verificación (S11).
- **Instructor vs. tool-use vía nuestro Router LiteLLM** — build-vs-buy; sesgo a no meter una
  segunda capa de abstracción sobre el seam (con 2 proveedores, la Guide ya inclina a adapters
  propios). Verificar composición con el fallback antes de decidir. Cuidado con doble capa de
  retry (Instructor + RetryPolicy del Router).
- **Versionado por directorios `v1/v2/`** — pattern para cuando haya evals (S11/S15).

### Rechazado / contraindicado (con porqué — material de Guide)
- **Caché semántica** — eje de agrupación ortogonal a identidad de cláusula; contraindicada en el
  diseño actual, reabrible solo con anclas en perillas + M2.
- **Jinja2 por defecto** — no se gana la dependencia con la complejidad de prompt actual (sin
  condicionales en M1); disparador natural = primera necesidad de ramas en el prompt.
- **Guardrails AI / Moderation API** — baja aplicabilidad en corpus normativo; Pydantic puro
  (`@model_validator`) cubre lo nuestro.

### Anclado a sesión futura
- **Verificación de grounding real (¿alucinó cláusulas?)** → **S11**. S04 aporta el marco.
- **Perillas estructuradas de scope** (filtrar por estándar) → **M2** (gated por retrieval).

---

## 7. Decisión de arquitectura abierta (nuestra, no del temario)

**Posición de RAG-ECSS en el espectro chat↔formulario.** ¿Añadimos perillas estructuradas (formato
de salida ahora; scope de estándares en M2) o nos quedamos en pregunta abierta pura? Condiciona
varias de las anteriores (structured output, posible reapertura de caché semántica). A decidir en
el paso de brief.

---

## 8. Cross-links con estado previo

- **Versión de prompt ↔ clave de caché:** si separamos el prompt en piezas, asegurar que el hash
  refleja el prompt *final compuesto*, no solo una pieza. Failure mode fácil de introducir.
- **Structured output ↔ streaming:** nuestra decisión de S03 (streamea texto, entrega estructura
  de golpe en `event: done`) encaja *mejor* con structured output de lo que el material imagina —
  siempre que mantengamos la separación texto-generado / estructura-final. Un shape donde *todo*
  es estructura rompería el modelo de streaming validado.
- **Endpoint de prompt activo ↔ thin-client rule:** exponer el prompt renderizado (no reconstruir
  lógica en el frontend) respeta la regla; reconstruirlo client-side sería el acoplamiento por
  duplicación que la regla prohíbe.

---

## 9. Punto de entrada al paso 3

Brief(s) de S04 sobre el foco elegido. Candidato a **dos briefs granulares** (uno por
funcionalidad, política del proyecto):
1. **Backend — endpoint que expone prompt activo + contexto CAG** (+ prompt movido a artefacto
   separado si se decide en el brief).
2. **Robustez de prompt** — scope explícito + permiso "no sé" formalizados en el system prompt,
   con las políticas de fallo declaradas y test estructural.

A confirmar en el paso 3 si son uno o dos, y el orden respecto a la decisión de espectro abierta.
