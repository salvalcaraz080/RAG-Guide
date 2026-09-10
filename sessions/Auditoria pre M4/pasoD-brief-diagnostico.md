# Brief Paso D — Diagnóstico de calidad de código (chat puente, read-only)

**Último paso del saneamiento pre-M4.** Cierra el ciclo: A auditoría → B clasificación → C2 código
→ C1 docs → Guide graduada → **D diagnóstico de calidad**.

**Naturaleza:** diagnóstico **read-only**. CC recorre el código **una vez**, reporta y **propone**;
no toca nada. *Medir y reparar son tareas distintas* — un brief que audita **y** arregla lo que
encuentra es open-scope. Los arreglos salen después, como fix-briefs separados que se aprueban en el
chat. **El informe es la especificación de esos fix-briefs.** CC no commitea.

**Precondición:** se audita el **árbol de trabajo con C2 y C1 ya aplicados** (código arreglado +
docs reconciliados, sin commitear). El chequeo de conformidad (Tier 1 · aspecto 2) usa como baseline
los docs **ya reconciliados por C1** y la Guide **ya graduada**. Si C1 no ha bajado aún, ese chequeo
va contra un baseline obsoleto — aplicar C1 primero.

---

## El principio organizador: dos niveles epistémicos

Los hallazgos **no** se listan en plano. Se separan por si tienen oráculo o son juicio, porque eso
es lo que decide qué se hace con cada uno:

- **Tier 1 · Objetivo** — tiene oráculo, baja divergencia. Evidencia verificable → candidato directo
  a fix-brief. **Prioridad alta.**
- **Tier 2 · Juicio** — sin oráculo, alta divergencia. CC **propone con trade-offs**; no es "hallazgo
  a arreglar", es candidato a decisión nuestra. **Prioridad por defecto baja.**

El informe se ordena por **(tier × severidad)**, no por los cinco aspectos. Mezclar juicio con bug
verificable hace que la opinión pese lo mismo que un error, y no lo pesa.

---

## Tier 1 · Objetivo (evidencia → fix-brief)

### 1 · Detección de errores — lo que los 854 tests NO cazan

"Los tests pasan" **no es un hallazgo.** El material es lo que el verde esconde:

- **La clase plausible-pero-falso** (shape-consistent, content-false): desalineación por índice,
  clave derivada de contenido donde debería ser posicional, confianza en el orden de llegada de un
  batch, test de "descendiente" por prefijo de string. La sección Cross-cutting de la Guide (bullets
  5 y 6) **es la checklist**; recorrerla contra el código.
- **Verificaciones que no verifican:** tests que mockean la frontera y dejan sin ejercitar el camino
  config→seam; checks sensibles al orden; "explicar por qué una anomalía parece mal" ≠ pasar el test;
  un check verde que nunca corrió.
- **Edges sin cubrir:** rutas de error, entradas vacías/degeneradas, condiciones de frontera,
  supuestos de concurrencia.
- **Defecto latente enmascarado por una precondición accidental** (nodo Phase 2.5 de la Guide): un
  camino correcto-por-accidente que una capacidad futura despertará. Clase D13.
- **Incluye los cambios de C2/C1 en el árbol** — están sin commitear y sin segundo par de ojos. Mirar
  con lupa la alineación por índice del filtro de ingesta y el orden canónico del accesor.

Por cada hallazgo: el defecto, su **condición de disparo**, y si **algún test afirma falsamente
cubrirlo** (un verde-falso es peor que un hueco descubierto).

### 2 · Coherencia inter-módulo = conformidad contra un baseline explícito

No "coherencia" en abstracto. **Conformidad** contra lo ya decidido, que vive en: `CLAUDE.md` /
`ARCHITECTURE.md` (post-C1) y la Guide graduada. Chequear:

- **Principios de arquitectura decididos:** routers finos (transporte/servicio/schema/contexto), seam
  único de proveedor (nada de `client.create(...)` disperso), corrección async (cliente async nativo
  o `to_thread`, ninguna llamada bloqueante dentro de `async def`), disciplina de fingerprint (cada
  stage digiere su propio input; orden canónico en el accesor), fail-fast de config, "sin LLM en
  ingesta", `embed≠cite`, la frontera citable/contexto.
- **Mismo stack para misma función:** structlog para logs en todos lados (no una mezcla con
  `logging`/`print`), un solo cliente HTTP, una sola fuente de settings, un envelope de error
  consistente, convenciones de test uniformes.

Una divergencia es (a) un módulo que viola un patrón decidido → arreglar, o (b) dos módulos
resolviendo lo mismo de dos formas → elegir una (decisión menor nuestra). **Este chequeo es el de más
valor y menos divergencia, y puede destapar otra violación de principio graduado — como D13.**

### 3a · Refactor objetivo (la parte con oráculo)

Código muerto (inalcanzable, funciones/vars sin uso), imports y dependencias sin usar, duplicación
**real** (misma lógica copiada), artefactos obsoletos (scripts de "investigación puntual" commiteados
que `CLAUDE.md` ya marca como no-duraderos), código comentado, restos de debug. Es dead-or-not:
evidencia → fix-brief de limpieza.

---

## Tier 2 · Juicio (CC propone con trade-offs; decidimos nosotros)

### 3b · Refactor de gusto y optimización

Refactors estructurales que son criterio (extraer esto, colapsar aquello) y oportunidades de
optimización. **Guardarraíl:** una propuesta de optimización **debe citar un coste medido**, no un
hipotético — *medir, no proyectar*. Optimización prematura es anti-simplicidad (*sencillez >
sobreingeniería*), así que sin hotspot medido no es hallazgo. CC propone con trade-offs, no como "a
arreglar".

### 4 · Mejoras de arquitectura general — solo sobre lo ya construido

**Guardarraíl (el que fijamos):** CC solo señala un problema arquitectónico si es **(a) un defecto
presente en el código ya construido, o (b) la violación de un principio ya graduado** (Guide/docs).
**No** propuestas de capacidad futura, **no** adelantar M6 (observabilidad, latencia/coste,
microservicios están en el temario — no traerlos aquí).

### 5 · Arquitectura de sistemas RAG — solo sobre lo ya implementado

**Guardarraíl (el más estricto):** solo sobre la arquitectura RAG **ya construida** — capa de datos,
retrieval vector/híbrido, pipeline de corpus, contrato citable/contexto. **No** rediseños de
retrieval / reranking / generación: esos son nodos del árbol que S09–S11 recorren sesión a sesión, y
proponerlos ahora **importa una decisión pre-tomada y pre-empta el nodo** que la Guide existe para
razonar. El aspecto 5 pregunta "¿lo ya construido es consistente con los principios RAG graduados y
está libre de defectos?", no "¿cómo deberíamos construir retrieval?".

**Test que CC se auto-aplica en 4 y 5** (la misma frontera de la Paso A): *¿corrige algo roto hoy —
defecto presente o violación de principio ya graduado?* → en alcance. *¿decide algo que el temario
decidirá luego — un nodo de diseño de retrieval/generación, o una preocupación de producción de M6?*
→ fuera de alcance; como mucho un **puntero** en la bandeja de futuro, nunca una propuesta.

---

## Formato del informe

`sessions/Auditoria pre M4/pasoD-diagnostic-report.md`:

1. **Resumen** — conteo de hallazgos por (tier × severidad).
2. **Tier 1 · Objetivo** — por severidad. Cada hallazgo: ubicación (`archivo:línea`), qué es,
   **evidencia**, condición de disparo (si aplica), y si un test afirma falsamente cubrirlo.
3. **Tier 2 · Juicio** — por severidad. Cada propuesta: qué cambiaría, **trade-offs**, y el coste de
   **no** hacerlo. Sin veredicto — la decisión es nuestra.
4. **Bandeja de futuro** — lo que CC note que sea genuinamente nodo de diseño de S09/S10/S11/M6:
   listado como **punteros, no propuestas**, para capturarlo sin pre-emptar.
5. **Observaciones** — lo que no encaje sin clasificar.

**Prohibido** en el informe cualquier etiqueta de disposición (transversal / futuro / arreglar-ya):
eso lo clasificamos en el chat, como en la Paso A. Y CC no arregla, no refactoriza, no commitea.

---

## Fuera de alcance

- Escribir, arreglar, refactorizar, commitear. El único archivo que CC crea es el informe.
- Re-auditar la deuda ya dispuesta en A/B (cerrada) — salvo que un cambio de C2/C1 haya introducido un
  defecto nuevo, que **sí** entra (chequeo de regresión sobre el árbol sin commitear).
- Proponer capacidad RAG o arquitectónica futura; adelantar el temario de M6.
- Correr la suite para "confirmar que pasa": ya sabemos que los 854 están verdes. El objetivo es lo
  que ese verde no ve.

## Transporte

Chat → repo: este brief baja a `sessions/Auditoria pre M4/`.
CC → chat: el informe se sube. Con él triamos aquí — Tier 1 → fix-briefs; Tier 2 → caso a caso;
bandeja de futuro → a su sesión.
