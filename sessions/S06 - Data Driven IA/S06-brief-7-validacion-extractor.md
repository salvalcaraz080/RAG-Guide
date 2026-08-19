# S06 — Brief 7: Tests y validación del extractor de referencias

> **Tipo:** tests automáticos (CC) + arnés de validación contra ground truth (CC construye el arnés,
> el usuario construye el ground truth y ejecuta). Cierra S06 y desbloquea la decisión grafo-vs-metadata
> de S07: no se construye retrieval sobre las referencias hasta saber si son fiables.
>
> **Alcance:** el **extractor de referencias del grafo** (las que salen del texto de los requisitos vía
> regex + ventana de asociación). **NO** las 281 referencias de tabla (Mecanismo D del Brief 5) — su
> destino no está decidido, validarlas sería especulativo; quedan como input medido para S07.
>
> **Frontera CC:** CC escribe tests de mecánica (mockeados) + el arnés de scoring (código que compara
> extractor contra un ground truth etiquetado). **Construir el ground truth y ejecutar la validación
> sobre el corpus real es UAT del usuario** — el etiquetado es juicio humano, irreductible.

---

## §0 — Por qué dos tipos de "test", y por qué uno no puede ser automático

Dos cosas distintas se llaman test aquí, y el extractor necesita ambas:

- **Tests de mecánica (automáticos, CC):** ¿el extractor hace lo que su código dice? Regex captura los
  patrones, la ventana de asociación empareja referencia con cláusula, la canonicalización normaliza.
  Deterministas, fixtures sintéticos, `uv run pytest`. Prueban **consistencia interna**.
- **Validación contra ground truth (manual, usuario):** ¿lo que el código dice es lo **correcto** sobre
  el corpus real? **No puede ser automática**, porque no existe oráculo automático de «esta referencia
  es correcta» — la única verdad es un humano leyendo el texto y juzgando. Un test de mecánica escrito
  con la misma lógica del extractor comparte sus errores de concepto y no los detecta (lección Brief 4:
  «coinciden dos implementaciones» no es corrección). Solo la verdad externa —humano sobre el
  documento— valida el concepto.

Este brief produce las dos capas. La segunda tiene un paso manual **irreductible**: el etiquetado.

---

## §1 — Tests de mecánica (CC, automáticos)

Sobre fixtures de texto sintético (sin `data/`, sin red): que el extractor, dado texto controlado,
produce las referencias esperadas. Cubrir al menos:

- Referencia normativa explícita (`ECSS-E-ST-40C clause 5.2`) → `(E-ST-40C, 5.2)`.
- Referencia a documento sin cláusula (`per ECSS-E-ST-10C`) → target documento, sin `target_clause_id`
  (el caso que hoy es ~49% — que se capture *como tal*, documento-sin-cláusula, no que se descarte ni
  se invente cláusula).
- La ventana de asociación: una cláusula con una referencia dentro de la ventana → asociadas; fuera de
  la ventana → no asociadas (el ±N caracteres, el parámetro cuya calibración el ground truth medirá).
- Canonicalización: `Rev.N` quitado en el `doc_id` **topológico** del grafo (coherente con Brief 4 —
  el grafo usa identidad sin revisión; solo la cita conserva revisión).
- `mention_type` (normative / informative / intra-doc) asignado según el patrón.

Estos tests fijan el comportamiento; **no** miden acierto sobre el corpus. `uv run pytest` verde.

---

## §2 — El arnés de validación (CC construye, usuario alimenta)

CC verifica/completa el código que, **dado un ground truth etiquetado, calcula precisión y recall** del
extractor contra él. Probablemente `score_groundtruth.py` ya existe — el brief confirma que funciona y
lo ejercita sobre un ground truth **sintético** (mockeado) en los tests. El arnés es código; el ground
truth que consume es dato humano.

El arnés produce, dado el etiquetado: precisión, recall, y la lista de fallos (falsos positivos y
falsos negativos concretos) para inspección. No un número solo — los casos, para poder mirarlos.

---

## §3 — Ground truth de PRECISIÓN (muestra existente, sobre la salida)

**Qué mide:** de las referencias que el extractor **sacó**, cuántas son correctas (ruido / falsos
positivos).

**Instrumento:** el `precision_sample.csv` que ya existe (~49 referencias muestreadas de la salida del
extractor, con 5 columnas de juicio vacías: `referencia_existe`, `target_doc_ok`, `clausula_target_ok`,
`tipo_ok`, `notas`).

**Flujo de etiquetado (asistido por LLM, verificado por humano):**
1. Un LLM juzga cada una de las 49 (¿la referencia existe en el texto fuente? ¿target correcto?
   ¿cláusula correcta? ¿tipo correcto?) — primer pase.
2. **El usuario revisa donde el LLM y su criterio discrepan**, y arbitra. Aquí «revisar discrepancias»
   **es suficiente**, porque el universo es cerrado: solo se juzgan las 49 que el extractor ya sacó, no
   hay referencia que pueda «faltar» de esta lista. (El caso ciego del recall —§4— no existe aquí.)
3. El `precision_sample.csv` queda etiquetado → el arnés calcula precisión.

**El LLM asiste; el humano decide.** El LLM no es el ground truth — propone, el humano verifica. Su
output es una hipótesis a revisar, no un veredicto (misma disciplina que el diagnóstico de tablas).

---

## §4 — Ground truth de RECALL (documento completo, sobre la fuente)

**Qué mide:** de las referencias que **existen** en el corpus, cuántas encontró el extractor (omisión /
falsos negativos). **Es la ausencia silenciosa** — el modo de fallo dominante de la sesión, aplicado al
extractor.

**Por qué no se puede derivar de la precisión:** precisión muestrea la *salida* del extractor; recall
tiene que muestrear la *fuente*, independientemente de lo que el extractor (o un LLM) haya propuesto.
Son dos instrumentos distintos, no uno más grande.

**El punto ciego que hay que evitar (crítico).** Cuatro casos por referencia: (1) extractor y LLM la
encuentran; (2) solo extractor; (3) solo LLM; (4) **ninguno la encuentra**. «Revisar las discrepancias»
cubre (2) y (3), pero **(4) es invisible** — no genera discrepancia, no aparece en ninguna lista, y es
exactamente el falso negativo compartido que el recall debe cazar. Si el etiquetado de recall solo
revisa discrepancias LLM-vs-extractor, **hereda el recall combinado de ambos** y es ciego al caso 4.

**Flujo de etiquetado (LLM asiste el barrido, humano lee las omisiones):**
0. **Paso previo de conteo (CC):** reportar referencias-por-documento y tamaño, para que el usuario
   **elija el documento de recall con datos delante** (no a ciegas). Recomendación: un CORE de densidad
   alta y tamaño manejable. **Un solo documento** para empezar; ampliar solo si el recall sale sospechoso.
1. El LLM hace un primer barrido del documento elegido y propone todas las referencias que ve →
   lista de candidatos (punto de partida, no lista a filtrar).
2. El usuario verifica los candidatos (tacha alucinaciones, confirma correctos).
3. **El usuario lee el documento completo buscando lo que el LLM y el extractor se saltaron ambos** —
   el paso irreductible. Es lo que captura el caso 4. Sin este paso, el recall es el del LLM, optimista
   en una cantidad desconocida.
4. El resultado es un **inventario exhaustivo hecho a mano** de las referencias de ese documento →
   el arnés calcula recall del extractor contra él.

**La diferencia operativa con precisión:** en precisión el humano *filtra* una lista cerrada; en recall
el humano puede *añadir* lo que ninguna máquina propuso. El flujo de recall debe permitir añadir, no
solo confirmar/tachar. Si solo permite filtrar candidatos, no mide recall — mide acuerdo.

---

## §5 — Qué decide el resultado (contexto, no trabajo de CC)

- **Precisión alta + recall alto:** el extractor es fiable; S07 puede construir sobre las referencias
  (grafo o metadata, la otra decisión).
- **Precisión baja:** el extractor mete ruido; la ventana de asociación o el regex necesitan ajuste
  antes de confiar en las referencias.
- **Recall bajo:** el extractor pierde referencias en silencio; un grafo/metadata construido encima
  está incompleto en una cantidad ahora **medida** (no supuesta). Esto **cambia la decisión
  grafo-vs-metadata de S07**: infraestructura de grafo sobre referencias con recall del 60% no es
  infraestructura, es deuda con forma de feature.
- **El número entra en la decisión de S07, no la toma este brief.** S06 mide; S07 decide con la medida.

---

## Qué entrega CC

1. **Tests de mecánica** del extractor (fixtures sintéticos, sin `data/`) — §1. `uv run pytest` verde.
2. **El arnés de scoring** verificado/completado — calcula precisión y recall dado un ground truth
   etiquetado, y lista los fallos concretos. Ejercitado en tests contra un ground truth **sintético**.
3. **Paso de conteo** (§4.0): comando que reporta referencias-por-documento y tamaño, para que el
   usuario elija el documento de recall.
4. **Plantilla/estructura del ground truth de recall** — el formato en que el usuario inventaría las
   referencias de un documento completo (análogo al `precision_sample.csv` pero orientado a
   exhaustividad sobre un documento, no a muestreo de salida). Con las columnas de juicio que el arnés
   consume.
5. **Guion UAT** — los dos flujos de etiquetado (precisión sobre las 49; recall sobre el documento
   elegido, con el paso de lectura de omisiones explícito), y la ejecución del arnés sobre ambos ground
   truths etiquetados → precisión y recall reportados con sus listas de fallos.
6. **`S06-decisions.md`** (append), etiquetado agnóstico/dominio.

## Qué NO hace CC

- No etiqueta el ground truth (juicio humano — UAT del usuario). Puede entregar el *prompt* para el
  primer barrido LLM, pero no sustituye la verificación humana.
- No valida las 281 referencias de tabla (S07).
- No decide grafo-vs-metadata (S07) — entrega la medida.
- No modifica el extractor para «mejorar» los números — este brief **mide**; ajustar el extractor con
  la medida delante es trabajo posterior, decidido con los fallos concretos a la vista.
- No ejecuta sobre `data/` real (UAT del usuario).

## Criterio de "hecho"

- Tests de mecánica verdes; el arnés calcula precisión y recall sobre ground truth sintético en tests.
- El paso de conteo permite elegir el documento de recall con datos.
- Los dos ground truths tienen su estructura definida; el de recall **permite añadir referencias que
  ninguna máquina propuso** (no solo filtrar candidatos) — la propiedad que lo hace medir recall y no
  acuerdo.
- Guion UAT entregado con el paso de lectura de omisiones explícito en el flujo de recall.
- Tras el UAT del usuario: precisión y recall del extractor **medidos** (no supuestos), con las listas
  de falsos positivos/negativos concretos, listos para que S07 decida grafo-vs-metadata con la cifra.
