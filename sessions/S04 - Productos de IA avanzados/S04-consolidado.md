# S04 — Consolidado

> **Paso 5 del ciclo.** Integra el resumen de sesión (`S04-resumen.md`, teoría + decisiones
> pre-implementación) con los aprendizajes de **implementar** los tres briefs. Histórico: **no se
> reescribe**. Es la base desde la que se gradúa a `RAG-GUIDE.md` (paso 6).
> El resumen **no se borra**: la divergencia teoría-vs-implementación es material de Guide.

> **Sesión:** S04 — productos avanzados sobre CAG (abstracción de prompts, plantillas, extracción
> estructurada, guardrails, cacheo semántico).
> **Foco implementado (M1):** robustez de prompt + endpoint de diagnóstico. El resto de S04
> (structured output, caché semántica, versionado por directorios) **diferido con razón
> registrada** — no aporta en M1 o adelanta temario.
> **Artefactos de la sesión:** `S04-resumen.md` (teoría), `S04-brief-1-robustez-prompt.md`,
> `S04-brief-1-fix-cita-fantasma.md`, `S04-brief-2-exponer-prompt-contexto.md`, + los tres
> `S04-decisions-*.md` de CC. Tres briefs, un hito.

---

## 1. La lente de la sesión (sin cambios respecto al resumen)

S04 gira sobre **¿dónde vive el prompt?** → en el backend, versionado, como artefacto de software.
Para RAG-ECSS es en gran parte **validación retroactiva** (ya lo encarnábamos desde S02/S03), no
novedad. La teoría completa de los cinco bloques está en `S04-resumen.md`; este consolidado no la
repite — se centra en **qué añadió la implementación**.

---

## 2. Lo implementado y lo aprendido al hacerlo

### 2.1 Brief 1 — robustez de prompt (instrucciones a archivo + scope + "no sé")

**Qué se hizo:** las instrucciones estáticas se movieron de constante Python
(`_STATIC_INSTRUCTIONS`) a `app/prompts/query/system.md`, cargadas por un loader trivial (sin
Jinja). El contexto CAG se quedó en `reference.py`. Se reforzó el contenido con una sección de
*scope* que distingue **dos tipos de rechazo**.

**Aprendizajes de implementación (no estaban en el resumen):**

- **El snapshot de caracterización hizo doble trabajo, como se diseñó.** Capturado antes de tocar
  nada: sirvió de red del refactor (el movimiento produjo un prompt **byte-idéntico**) y luego, al
  actualizarlo tras el refuerzo, su diff *fue* la revisión del cambio de prompt. Materializa el
  beneficio "PR de solo-prompt legible" dentro de la suite.
- **`_STATIC_INSTRUCTIONS` se mantuvo como nombre de constante de módulo por compatibilidad de
  test**, no por estética. Un test de caché hace `monkeypatch.setattr(llm_service,
  "_STATIC_INSTRUCTIONS", ...)` para probar la invalidación de clave sin depender del contenido
  real del archivo. El refactor era de *origen* del valor (archivo en vez de literal), no de su
  *forma de consumo* — cambiar el nombre habría roto un test sin necesidad.
- **Distinguir "mencionar el estándar al rechazar" de "alucinar una cláusula".** La primera
  aserción del test `real_llm` de "out of material" falló porque ambos modelos, correctamente,
  **nombran** el estándar preguntado (`ECSS-E-ST-70-41`) al decir que no lo cubren — eso es el
  comportamiento deseado, no una alucinación. La aserción se corrigió para prohibir solo una
  *cláusula inventada* pegada al estándar (nombre + número), no la mención del nombre. Aprendizaje
  agnóstico: al testear rechazos por texto, el modelo *va* a repetir términos de la pregunta; la
  aserción debe apuntar a la fabricación, no a la mención.
- **Limitación que el brief 1 dejó abierta y que forzó el fix:** los tests `real_llm` tuvieron que
  llamar a `call_llm`/`build_system_prompt` **directamente**, no al pipeline `answer_query`, porque
  `_citations()` devolvía la cita candidata **siempre** — así que aseverar "citations vacío en un
  rechazo" no tenía sentido midiendo el campo `citations`. Esto era el síntoma del bug que el fix
  corrigió.

### 2.2 Brief 1-fix — cortar la cita fantasma en rechazos

**El bug (dominio):** `_citations()` derivaba la cita candidata del fragmento inyectado
**incondicionalmente**. El guardrail de rechazo del brief 1 hizo que el modelo rechazara — pero el
rechazo salía con una cita ECSS pegada. **Con un solo fragmento en M1, el rechazo es el caso
común**, así que el sistema producía citas fantasma en la mayoría de interacciones de demo. Ironía
registrada: **cuanto mejor rechazaba el guardrail, más citas fantasma generaba.**

**El fix:** el rechazo empieza con una **frase natural fija** (`Out of scope:` /
`Not covered by the provided material:`), detectada por `startswith` en `_refusal_kind()` en el
punto de ensamblado compartido → `citations = []`.

**Aprendizajes de implementación:**

- **Frase natural, no marcador técnico — decidido por el streaming.** Un marcador técnico
  (`OUT_OF_MATERIAL:`) habría que strippearlo, y en `/query/stream` el texto se emite token a token
  antes de poder limpiarlo → obligaría a **bufferizar el arranque del stream**, que es
  lógica-solo-en-streaming (prohibida por S03: el streaming es capa de entrega, no camino con
  reglas propias). Una frase natural **es** el mensaje legible: se streamea tal cual, la detección
  ocurre en el ensamblado, no en el texto ya emitido. El principio de S03 (streaming descartable
  sobre la fuente de verdad) *dictó* la elección de mecanismo del guardrail.
- **Case-sensitive a propósito.** Si el modelo emite la frase con otra capitalización, es señal de
  que no siguió la instrucción verbatim — no se enmascara con `lower()` un síntoma de desviación.
- **El fix mejoró la testabilidad de rebote.** Los tests `real_llm` de rechazo pasaron a aseverar
  `citations == []` (limpio) en vez de escarbar el texto buscando cláusulas inventadas, y pudieron
  llamar al pipeline completo — cerrando la limitación del brief 1 (§2.1).
- **Un test de S02 blindaba el bug.** `test_query_grounding_not_covered_does_not_fabricate_citation`
  mockeaba un rechazo y aseveraba `citations == [candidata]` **como si fuera correcto**. Pasaba
  mecánicamente (el texto viejo nunca activaba la detección), dando falsa cobertura sobre justo el
  punto roto. El nombre decía "does not fabricate" y el cuerpo afirmaba una fabricación. Se
  reescribió con el marcador real y `citations == []`.

### 2.3 Brief 2 — exponer prompt activo + contexto CAG (cierre del anclaje S03)

**Qué se hizo:** `GET /api/v1/prompt` devuelve las instrucciones activas; el contexto CAG viaja
como `injected_context` opt-in (`include_context` en `QueryRequest`) en la respuesta de
`/query`/`/query/stream`. Cierra los dos `# TODO` del sidebar de S03.

**Aprendizajes de implementación:**

- **Fidelidad resuelta por factorización, no por disciplina.** El riesgo no era técnico sino de
  **divergencia** (que el diagnóstico dejara de coincidir con lo que el modelo recibió). Se
  factorizó `_compose_system_prompt(context_block)` y se hizo que `_cache_lookup` devuelva el
  `context_block` que ya construye para la clave — `answer_query`/`stream_answer_query` reutilizan
  **ese mismo string** para el prompt y para `injected_context`. No hay segunda derivación en el
  mismo request. La fidelidad queda garantizada por construcción, no por "acordarse de mantenerlas
  iguales".
- **En M1 la doble derivación sería inofensiva (función pura) — pero no se depende de eso.**
  Diseñar la reutilización ahora, cuando es gratis, evita re-descubrir el problema en M2 (contexto
  de retrieval, potencialmente no determinista o caro de recalcular).
- **`include_context` NO es slot de caché** — es presentación, no generación. Dos requests que solo
  difieren en el flag dan el mismo HIT. Refuerza la ortogonalidad de la clave (precedente: `model`
  excluido en S03).
- **Hallazgo de fidelidad para M2 (el más valioso del brief):** en un HIT, `injected_context` se
  adjunta **recomputado fresco** para esa request, no se guarda dentro del blob cacheado. Correcto
  en M1 (recomputar da el mismo string, determinista). **Pero en M2**, si el contexto depende de
  retrieval no determinista o de un corpus que cambió entre la generación cacheada y el HIT,
  recomputar al servir devolvería un contexto **distinto del que realmente generó la respuesta
  cacheada** — fidelidad exigiría entonces **guardar `context_block` dentro de la entrada
  cacheada**. Latente hoy, `# TODO` para M2.

---

## 3. Divergencias teoría ↔ implementación (material de Guide)

- **El material empuja a Jinja2 / Instructor / Guardrails AI / redisvl reflexivamente.** La
  implementación confirmó que **ninguna se gana el sitio en M1**: sin condicionales no hay Jinja;
  con dos proveedores y un seam propio, Instructor compite con nuestro Router; Pydantic puro cubre
  los guardrails; la caché semántica está contraindicada (§4). El patrón agnóstico: el material de
  curso alcanza librerías de abstracción por defecto; contrastar contra la implementación propia
  antes de adoptarlas.
- **"Guardrails antes del cache" (material) es más grueso de lo necesario.** Con caché de slots, lo
  correcto es **input guardrails antes del lookup; output guardrails antes del `set`** — así un HIT
  es seguro por construcción y no re-paga validación. El material da la regla gruesa por no tener
  caché con persistencia de validación.
- **El propio ejemplo del material (Stripe) confirma la contraindicación de caché semántica** que
  ya teníamos, sin que el material cierre el punto para dominios normativos.

---

## 4. Candidatos a graduar a la Guide (para el paso 6)

Marcados por eje agnóstico/dominio y por si **refuerzan** una entrada existente o **añaden** nodo.

1. **[agnóstico, añade] Prompt como artefacto de archivo separado sin motor de plantillas** cuando
   el prompt no tiene condicionales. El beneficio (PR de solo-prompt, editable sin código) no
   requiere Jinja; el motor se gana el sitio en la primera rama real. Disparador: condicionales en
   el prompt.
2. **[agnóstico, añade] Snapshot de caracterización como red de refactor + revisión de cambio de
   prompt.** El diff del snapshot ES la revisión del cambio de instrucciones.
3. **[dominio→generalizable, añade] Dos tipos de rechazo: "out of domain" vs "out of material".**
   Para cualquier RAG con grounding + material acotado. "Out of material" es el que impide la cita
   fantasma cuando el corpus no cubre. Refuerza el nodo de grounding de Phase 2.
4. **[agnóstico, añade] Frase natural como puente hasta structured output.** Para señalar un estado
   (rechazo) sin structured output: una frase-marcador natural detectada por prefijo funciona en
   streaming sin buffering (a diferencia de un marcador técnico, que obligaría a bufferizar). Se
   jubila con structured output. El principio de streaming-descartable *dicta* el mecanismo.
5. **[agnóstico, refuerza] Guardrails y caché: input antes del lookup, output antes del `set`.**
   Refinamiento sobre la regla gruesa del material. Nodo nuevo bajo la caché de Phase 4.
6. **[agnóstico, añade] Fidelidad por envoltura, no por copia**, en endpoints de diagnóstico que
   exponen estado del backend. Corolario del thin-client: envolver la fuente real (factorizar el
   builder), nunca reconstruir. Refuerza la entrada "don't reconstruct backend state in the
   frontend" de Phase 4 con la contraparte backend.
7. **[agnóstico, refuerza] Un test puede blindar un bug si codifica la intención equivocada.**
   Tercera instancia (S02 phantom-citation, S03 StreamDoneEvent, ahora S04 el test que afirmaba la
   cita fantasma). La lección converge: un test que afirma el comportamiento actual sin
   cuestionarlo no es red de seguridad, es cemento sobre el error.
8. **[dominio, refuerza] Caché semántica contraindicada en corpus normativo** — confirmada por el
   ejemplo del propio material. Ya en Phase 4; reforzar con el matiz "reabrible solo con anclas
   normativas en parámetros estructurados + retrieval".
9. **[agnóstico, añade] Flags de presentación no entran en la clave de caché** (`include_context`).
   Y su corolario de fidelidad: si un HIT recomputa contenido de diagnóstico, es fiel solo si el
   recompute es determinista; si no, guardar el contenido en la entrada cacheada. Nodo bajo caché.
10. **[agnóstico, refuerza] Structured output: forma ≠ verdad; emisión ≠ verificación.** Refuerza
    Phase 2 desde el ángulo de guardrails (schema válido, contenido roto).

**Nota para el paso 6:** "converge, no engorda". No todo esto entra como nodo nuevo; varios
**refuerzan** entradas existentes (revisar y ajustar, no duplicar). El juicio de qué gradúa —
validado por fricción de implementación, no por lectura— es el paso 6.

---

## 5. Decisión de arquitectura que S04 deja abierta (nuestra, no del temario)

**Posición de RAG-ECSS en el espectro chat↔formulario.** ¿Perillas estructuradas (formato ahora,
scope de estándares en M2) o pregunta abierta pura? No se resolvió porque el foco elegido
(robustez + endpoint) no dependía de ella. Condiciona structured output y la posible reapertura de
caché semántica. A retomar cuando toque — probablemente al entrar M2.

---

## 6. Anclajes que S04 deja puestos

- **S11 (verificación de grounding):** el guardrail de scope es *filter instruido-en-prompt*, no
  verificación de que el modelo no alucinó al *responder*. El puente de frase-marcador se jubila
  cuando structured output dé un campo tipado de rechazo.
- **M2 (fidelidad de contexto en caché):** si el contexto pasa a ser no determinista, guardar
  `context_block` dentro de la entrada cacheada (§2.3).
- **M2 (perillas de scope):** filtrar por estándar como parámetro estructurado, gated por retrieval.
- **Contrato (diferido):** campo tipado en `QueryResponse` para señalar rechazo programáticamente
  (hoy `citations == []` es la señal de facto). Candidato a structured output / S11.
- **Producto (abierto):** posición en el espectro (§5).

---

## 7. Estado de S04

Cerrada, tres briefs implementados y validados (Docker + tests + `real_llm` contra Anthropic y
OpenAI). Backend M1: prompt como artefacto de archivo, scope reforzado con dos rechazos, cita
fantasma cortada en rechazos, `GET /api/v1/prompt` + `injected_context` opt-in. Sidebar de
Streamlit sin `# TODO`. `corpus/` sigue congelado. Pendiente: **paso 6** — graduar a `RAG-GUIDE.md`
los candidatos de §4 con juicio de convergencia.

---

## ADDENDUM — Brief 3: pipeline de input-guardrails (reapertura post-consolidación)

> **Nota de proceso:** este addendum se añade *después* de que S04 se consolidara y graduara. El
> brief 3 reabrió una decisión que yo había cerrado en el repaso teórico (descartar el guardrail de
> toxicidad). No reescribo nada de lo anterior — el consolidado es histórico; esto se añade. La
> divergencia entre "lo descartado en teoría" y "lo reincorporado tras discusión" es, en sí misma,
> material de Guide.

### Qué se reabrió y por qué

En el repaso teórico descarté la moderación de input apoyándome en "nadie mete toxicidad en
preguntas sobre ECSS". Esa era una **suposición sobre el input que el dominio no controla**: un
endpoint público que acepta texto libre de usuario debe moderar el input por higiene, con
independencia de la temática del corpus. El error no fue solo la conclusión, fue el razonamiento
(dejar que el dominio decidiera una propiedad del input abierto).

### Aprendizajes de implementación (material de Guide)

- **[agnóstico] La Moderation API no ahorra llamadas — añade una.** Es una llamada de red a un
  proveedor (OpenAI), no un check local. En el orden correcto (input guardrails **antes** del cache
  lookup) corre en *toda* request, incluidos los hits — le añade una dependencia de red a un camino
  que hoy no tiene ninguna. La justificación honesta es **seguridad/higiene del input**, nunca
  ahorro de coste. Lo que evita no es la llamada al LLM por economía, es *procesar y cachear una
  respuesta a un input que no debíamos atender* (ganancia de seguridad con forma de ahorro).
- **[agnóstico] Toxicidad ≠ prompt injection.** La Moderation API cubre contenido tóxico, no
  injection. Si se adopta "para proteger el input", cubre una amenaza y deja abierta la que más nos
  afecta (subvertir el grounding). Honestidad de alcance: es "el guardrail de toxicidad del input",
  no "el guardrail de input". La injection la mitiga hoy, parcialmente e indirectamente, la robustez
  de prompt del brief 1; una capa dedicada es futura (hueco en el pipeline).
- **[agnóstico] El orden input-guardrails-antes-del-cache es un INVARIANTE de seguridad, no una
  optimización.** El argumento que lo sostiene no es que la colisión de clave sea explotable hoy (en
  exact-match es prácticamente imposible), sino que **un invariante de seguridad no debe apoyarse en
  una propiedad accidental de otro subsistema** (que la clave sea el texto exacto). El día que entre
  normalización o caché semántica, "input distinto que acierta el mismo hit" pasa de imposible a ser
  el mecanismo de diseño, y el bypass se abriría solo, en silencio, por un cambio no relacionado. El
  coste de ponerlo en el orden correcto hoy es cero; la deuda de ponerlo mal es una vulnerabilidad
  latente activada por un cambio futuro. (Misma familia que la lección "un test que funciona por una
  coincidencia del estado actual, no por diseño".) Fijado como test de regresión, no como comentario.
- **[agnóstico] Moderación como preocupación ortogonal a la abstracción de proveedor.** El cliente de
  moderación va **aparte** del Router de completions: la moderación no es una completion, no tiene
  fallback, y mezclarla en el `model_list` acoplaría "elegir qué LLM responde" con "vetar qué entra".
  Aislarla mantiene la capa de proveedor enfocada.
- **[agnóstico] Aislar en tests las APIs de red por defecto (opt-in, nunca opt-out)** — segunda
  instancia del principio ya graduado para datastores (Phase 4). `OPENAI_API_KEY` presente en `.env`
  habría hecho que la suite pegara a la Moderation API real; una fixture `autouse` fuerza el camino
  sin-red por defecto, igual que la de Redis.
- **[agnóstico] Fail-open con warning observable** para una capa de seguridad que depende de un
  proveedor opcional, coherente con el patrón de degradación del proyecto (Redis, fallback opcional).
  Deuda registrada: la moderación queda **condicional** a la disponibilidad de OpenAI; un requisito
  duro exigiría fail-closed o un proveedor de moderación independiente del LLM.

### Corrección a lo graduado en el paso 6

El nodo de guardrail tooling de la Guide (Phase 5) decía que la moderación es "near-irrelevant for a
normative technical corpus". **Se corrige:** la relevancia de la moderación de input no la decide el
dominio del corpus sino la **superficie de input abierto** — cualquier endpoint público que acepta
texto de usuario la justifica. (Ajuste aplicado a `RAG-GUIDE.md` en el mismo backfill.)
