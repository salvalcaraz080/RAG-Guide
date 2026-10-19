# S11 — Generación y calidad: augmentation, síntesis, citación, alucinaciones, índice, evaluación

> Resumen provisional (paso 2). Teoría de los seis artículos + lectura sobre RAG-ECSS + sesión en vivo.
> Pre-implementación: nada de lo que aquí figura como propuesta está validado en código ni medido.

---

## 1. Content augmentation: preparar el contexto antes de generar

**Qué es.** La capa entre recuperación y generación. No mejora el retrieval ni el prompt: mejora el
material. Convierte chunks crudos en contexto destilado — lo justo, en el orden correcto, sin perder
trazabilidad.

**Por qué el ruido es caro.** Tres costes medibles:
- **Tokens** — se paga cada token de entrada; el boilerplate es coste sin señal.
- **Atención** — *lost in the middle* (Liu et al. 2023): lo del centro de un contexto largo pesa menos.
  Efecto real pero dependiente de modelo y tamaño de contexto — hipótesis a medir, no default.
- **Alucinación** — el ruido (sobre todo numérico) es combustible: más cifras irrelevantes, más
  probabilidad de atribuir la equivocada.

**Pipeline componible.** Etapas con contrato uniforme (fragmentos entran, evidencia sale), cada una
activable, desactivable y medible aislada: *comprimir → extraer puntos clave → ordenar → ajustar al
presupuesto*. La firma lleva **qué se va a generar** (`target`), no solo la consulta: la destilación
es específica de la salida.

**Invariantes de implementación.**
- El `chunk_id` de origen sobrevive a toda transformación; evidencia sin id = trazabilidad rota antes
  de generar.
- Toda etapa que tira información registra **qué** tiró (`was_compressed`, `dropped_chunk_ids`).
- Una compresión que deja vacío devuelve el original, no vacío (fallo silencioso de recall).

**Extractiva vs abstractiva.**
- **Extractiva** — selecciona texto original sin reescribir. Barata, sin llamada, **no puede
  inventar**. Riesgo: tirar lo relevante por coincidencia léxica ("cobros" vs "pagos") — recall
  degradado en augmentation, difícil de diagnosticar porque el retrieval parecía bien.
- **Abstractiva** — resumen con LLM. Flexible, natural para prosa narrativa. Introduce **un segundo
  punto de generación**: si resume mal, la alucinación entra con apariencia de fuente. Contención
  mínima: esquema estricto con `None` explícito. Coste: una llamada por chunk, puede superar el
  ahorro.
- **Enriquecer también es generar** — una cabecera sintética escrita por LLM tiene el mismo riesgo.
  Alternativa sin riesgo: **cabecera determinista desde metadatos**.

**Orden y presupuesto.** Cargar los extremos (lo más fuerte al principio y al final). Orden correcto
de las operaciones: **ajustar al presupuesto sobre la lista ordenada por relevancia** (descartar lo
más débil primero) y **después** reordenar por extremos — al revés se descarta por posición, no por
fuerza. El punto óptimo de compresión no se adivina: se mide contra la calidad final.

**Lectura sobre RAG-ECSS.**
- La abstractiva queda **excluida por principio** (graduado: texto normativo generado = vector de
  fabricación), no por coste.
- En ECSS el chunk ya es la cláusula: comprimir *dentro* de ella separa requisito de NOTE/AIM →
  deja de ser augmentation y pasa a ser re-chunking. Excepción: las 272 tablas sin partir (filas) —
  grupo re-chunk.
- Lo que sobrevive previsiblemente: **deduplicación** (duplicados del parser), **presupuesto**,
  **orden**, **cabecera determinista** (ya existe: `<source>`/`<item authority type uid>`).
- Recencia como señal de orden no aplica; su análogo (revisión vigente vs sustituida) es filtro.

---

## 2. Síntesis de fuentes que se contradicen

**La discrepancia es información.** Si las fuentes coinciden, no hay nada que sintetizar. Cuando
discrepan, la primera pregunta es *por qué*: casi siempre revela una variable oculta que el usuario
no mencionó. Una síntesis buena conserva esa señal ("X si A; Y si B"), no la aplana en un número.

**Tres síntesis malas** — promedio silencioso, quedarse con la primera, número intermedio inventado.
Las tres plausibles, ninguna defendible.

**Separar aritmética de juicio** (patrón central). El código calcula un **ancla determinista**
(mediana ponderada, rango, señal de contradicción); el LLM razona sobre ella sin inventar la
aritmética. Esqueleto auditable + juicio donde la aritmética no llega. Frente a la pasada única
(simple, pero el número sale de una caja negra).

**Ponderar con pocas señales defendibles.** Si no puedes justificar un coeficiente en una frase, no
estaba listo. Pesar no arregla señales malas: las **amplifica**. Contradicción = dispersión alta
**entre fuentes fuertes** (no dispersión a secas; si no, cada outlier débil dispara alertas). El
umbral es política, no verdad.

**Salida honesta.** Rangos cuando hay desacuerdo real, `contested`, `rationale`, fuentes por
componente; "si la evidencia es pobre, dilo". La confianza de la salida refleja la calidad de la
entrada, no un valor por defecto. **Instrucción ≠ enforcement**: lo que el código pueda comprobar tras
generar (ids citados ⊆ proporcionados, rango dentro del ancla) se comprueba en código.

**Lectura sobre RAG-ECSS.** La ponderación por score **no se traslada**: una cláusula no vota, obliga
o no. La autoridad es estructural (normativa vs informativa, revisión, aplicabilidad) → reglas de
precedencia deterministas. El score de retrieval mide coincidencia con la consulta, no verdad.
Tipología de "síntesis" en normativa:

| Tipo | Ejemplo | Qué hace la síntesis |
|---|---|---|
| Acumulativa | E-40 y Q-80 imponen requisitos sobre el mismo tema | Unión; el riesgo es **omitir** |
| Condicional | Aplica según criticidad (B vs C) o tailoring | **Exponer la condición** — análogo directo de la "variable oculta" |
| De autoridad | Handbook parafrasea la norma | La norma gana por regla |
| Contradicción normativa real | Cláusulas o revisiones incompatibles | Exponer, nunca resolver |

La condicional es la de más valor de producto y la más fácil de aplanar.

---

## 3. Citación y atribución verificable

**Un id no es una citación.** Una cita verificable cumple tres propiedades comprobables:
- **Resuelve** — apunta a algo que estuvo en el contexto.
- **Localiza** — la línea/tramo concreto, no el documento.
- **Trazable** — el lector puede llegar a la fuente.

La granularidad del localizador **se paga en la ingesta**: si no se capturó, no se improvisa — se
cita a nivel inferior con honestidad.

**Integridad referencial en código, tras generar.** Cita colgante = el fallo más peligroso porque
tiene el mismo aspecto que una legítima. Política explícita: rechazar y reintentar / degradar /
bloquear; nunca ignorar. La integridad es **estructural, no semántica**: confirma que la fuente existe,
no que diga lo afirmado.

**Estructura primero, presentación después.** La cita estructurada es la fuente de verdad; inline,
notas al pie o chips son render encima. La capa de IA emite datos neutros (`document_id`, locator);
la capa de negocio resuelve enlaces y permisos.

**Lectura sobre RAG-ECSS** (estado verificado contra `ARCHITECTURE.md`):

| Propiedad | RAG-ECSS |
|---|---|
| Resuelve | Marcadores posicionales `[n]` resueltos en servidor → **cita colgante imposible por construcción**; `[n]` fuera de rango = `invalid_ref` |
| Localiza | Cláusula (ancla estructural) + cita literal `"…" [n]` como localizador fino |
| Trazable | `retrieved_context` chunk a chunk; sin enlace al DOCX |
| Estructura/presentación | `citations` (lo que el texto marca) ≠ `retrieved_context` (lo inyectado) |
| Política ante fallo | **Pendiente** — validación literal en modo observación |

Nodo para la Guide: **hacer imposible antes que detectar** — no pedir al modelo que copie ids opacos.
Granularidad: el material cita por componente para no cansar; en normativa una afirmación sin marcador
es inverificable → la respuesta a "cansa" está en la presentación, no en citar menos.
Restricción propia que el material no trata: **la política choca con el streaming** — la validación
completa llega con el texto ya en pantalla; bufferizar destruye TTFT; el streaming es capa borrable
(graduado) → la política opera en el camino no-stream y llega al stream como veredicto posterior.

---

## 4. Detección y mitigación de alucinaciones

**Tres formas de alucinar.** Todas plausibles — ese es el problema.
- **Fabricación** — afirmación sin fuente.
- **Atribución falsa** — el dato existe, pero no en la fuente citada ("alucinación con coartada").
- **Extrapolación no fundamentada** — salto lógico más allá de las fuentes.

**Embudo de detección, barato → caro.** Lo que un `if` descarta no gasta una llamada.
1. Anclaje determinista (aritmético/literal).
2. Juez LLM estricto: veredicto booleano estrecho, modelo distinto al generador, sesgo a "no
   soportado". Circular por naturaleza: reduce el riesgo, no lo elimina.
3. Consistencia entre N muestras: cara; confunde "adivina" con "las fuentes discrepan".

**Mitigación.** *Prevenir* (prompt + esquema; rendimientos decrecientes) → *validar* (decisión
graduada por línea) → *abstenerse* (acierto del sistema, no fallo). Abstenerse de más vuelve el
sistema inútil: el umbral se calibra. Estado mixto (señales que discrepan) ≠ "fundamentado": necesita
estado propio (`needs_review`). La `confidence` autoasignada por un LLM no está calibrada — el
veredicto booleano con razón es lo defendible. Guardarraíl por respuesta ≠ medida del sistema.

**Lectura sobre RAG-ECSS.**

| Tipo | Forma en normativa | Detección |
|---|---|---|
| Fabricación | Contenido fuera de los grupos inyectados; riesgo propio: **conocimiento paramétrico de ECSS** | `not_found`; frases sin marcador son fabricación indetectable |
| Atribución falsa | D-1: cita literal exacta de otra cláusula/fila/columna; `[n]` al grupo equivocado | Parcialmente determinista (ver abajo) |
| Extrapolación | Ampliar alcance; **inflar autoridad** (NOTE como requisito, *shall*↔*should*); **afirmaciones de ausencia** | Mayormente semántica |

Embudo de RAG-ECSS:

| Capa | Comprobación | Coste | Estado |
|---|---|---|---|
| 0 | `invalid_ref` | nulo | ✅ hecho (S09) |
| 1 | literal `verified` / `not_found` | nulo | ✅ hecho (S09), observación, sin política |
| 1 | literal `misattributed` (la cita está en otro grupo inyectado) | nulo | ❌ nuevo |
| 2 | frase factual sin marcador | nulo | ❌ diferido desde S09 |
| 3 | juez LLM / NLI afirmación ↔ cláusula citada | alto | abierto |

`misattributed`: si la cita no está en el grupo citado, buscarla en los demás → separa fabricación
(en ningún grupo) de atribución falsa (en otro). Cubre parte de D-1 sin LLM; no cubre la cita literal
válida usada para sostener otra cosa (columna/fila equivocada) — eso es semántico.
Consistencia: descartada para RAG-ECSS (N generaciones sobre respuesta streameada).
El anclaje por rango del material es el modo laxo; el modo estricto (literal) es el nuestro.
El mismo verificador sirve dos veces: guardarraíl online y métrica offline.

---

## 5. Ciclo de vida del índice: reindexación y versionado

**El fallo que no da error.** Dos vías: **deriva de contenido** (cambia el documento, el vector no) y
**mezcla de versiones** (vectores de dos modelos comparados: el coseno devuelve un número sin
significado).

**Defensas.** Clave de versión por vector (modelo, dimensiones, normalización, preprocesamiento);
**scope obligatorio en consulta** (`WHERE embedding_version = :current`) como garantía de corrección,
no optimización. ¿Cambió el documento → incremental por hash? ¿Cambió el proceso → migración
completa? Migración **blue/green**: construir sombra, verificar, promover atómicamente, borrar lo viejo
solo tras éxito. Disparar por evento, no por calendario. "Nunca cambiaremos de modelo" es una frase
cara: la columna de versión es un seguro barato.

**Lectura sobre RAG-ECSS.** Detección de obsolescencia ya resuelta en S07 (`input_fingerprint`,
`chunks_fingerprint`, `stage_fingerprint` con modelo y dimensiones; código 2 = "no se puede saber").
Sin verificar: si la consulta filtra por versión (probable que no haga falta hoy: una sola versión,
carga que reemplaza). Hueco que el material no trata: **el golden set también está versionado** — sus
juicios referencian chunks bajo un `input_fingerprint`; un re-chunk los invalida. La verificación
correcta antes de promover un índice es el arnés por estrato, no "consultas sanas". Los veredictos de
fidelidad de S11 deben registrar versión de corpus, prompt y modelo.

---

## 6. Evaluación de la generación (RAGAS)

**Cuatro métricas, dos ejes.**
- Recuperación: **context precision** (relevantes y bien posicionados), **context recall** (se trajo
  todo lo necesario; requiere referencia).
- Generación: **faithfulness** (proporción de afirmaciones inferibles del contexto), **answer
  relevancy** (la respuesta aborda la pregunta).

Leídas juntas diagnostican: fidelidad baja con contexto bueno → problema de generación; fidelidad alta
con recall bajo → el modelo se porta bien con lo poco que le llega.

**El golden set es el techo.** Un set pobre produce números igual de confiados y sin significado. Debe
cubrir el espectro real: casos claros, ambiguos, **contradictorios** y **de abstención** — sin caso de
abstención se premia contestar siempre.

**Dos modos.** Offline con referencia como **puerta de regresión** antes de desplegar; en producción,
sin referencia (solo faithfulness/relevancy sobre muestra) + señales operativas de guardarraíles.
Alertar por deriva, no por valor absoluto; la deriva puede ser cambio de tráfico, no regresión.

**Disciplina.** El juez es un LLM: números comparables (A vs B), no notas absolutas. Goodhart:
optimizar fidelidad sola empuja a abstenerse y hunde la utilidad. Evaluar es un lote presupuestado,
no un guardarraíl en línea. API de RAGAS inestable entre versiones: fijar versión.

**Lectura sobre RAG-ECSS.**
- **Retrieval ya se mide mejor** que con RAGAS (642 juicios humanos, protocolo versionado, arnés por
  estrato): las métricas de contexto de RAGAS serían una medida más ruidosa de lo mismo.
- **Faithfulness de RAGAS no ve la atribución falsa**: comprueba cada afirmación contra *todo* el
  contexto, no contra la cláusula que la cita.
- **Faithfulness está sesgada a favor de la expansión**: más contexto = más fácil encontrar apoyo. La
  dilución no se manifiesta como infidelidad, sino como **omisión**, **pérdida de foco** (NOTEs ➕ en
  lugar del requisito) y **más atribución falsa**. Decidir `bounded` con faithfulness mide lo que no
  importa.
- **Answer relevancy castiga la abstención correcta** (genera preguntas desde la respuesta y compara
  por embeddings).
- **Referencia como conjunto de cláusulas obligatorias** por consulta (derivable de los juicios v2),
  no prosa → métricas **deterministas**: recall de citas obligatorias (captura omisión), precisión de
  citas, tasa de citas literalmente verificadas (+ `misattributed`), tasa de frases factuales sin
  marcador. El juez LLM queda para lo que el código no ve (implicación, exposición de la condición).
- N=14: comparación **pareada** (`orphans` vs `bounded`, misma consulta), lectura **por estrato**,
  criterio de mejora **fijado antes** de ejecutar.

---

## 7. Anclajes de S10 / diferidos de S09 frente a S11

| Anclaje | Qué aporta la teoría |
|---|---|
| `bounded` vs `orphans` | Es una pregunta de augmentation (dilución), no de retrieval. Se decide con evaluación pareada de generación y métricas de citas; faithfulness solo como apoyo. Tercera opción además de sí/no: incluir ➕ marcados como contexto de apoyo, no evidencia del mismo rango |
| Contrato tipado (parcial/rechazo) | Estado de respuesta (completa / parcial / condicional / rechazo) alimentado por veredictos, entregado al final del stream. Unidad = afirmación ligada a cláusula |
| Frases sin cita / afirmaciones de ausencia | Capa 2 del embudo (determinista). Ausencia = extrapolación desde el silencio del contexto recuperado |
| Política ante cita no soportada (D-1) | Embudo 0–3; `misattributed` cubre parte de D-1 sin LLM; política graduada por veredicto, compatible con streaming |
| UI "retrieved but not cited" | Dos ejes visibles: citado / no citado × recuperado / añadido por expansión (➕) |

---

## 8. Decisiones S11

### Decididas
- **`orphans` sigue como default** de expansión. `bounded` no se adopta; queda reabrible como
  experimento cuando exista el arnés de generación (comparación pareada, criterio preregistrado:
  gana en omisión en el estrato normativo, no empeora atribución ni precisión de citas, coste
  asumible; empate → gana lo simple).
- **Se elimina el streaming.** Respuesta tradicional (completa y verificada antes de servirse). Se
  revisan las decisiones condicionadas por el streaming **solo si simplifican**. Motivo: el streaming
  impide la política más estricta (rechazar/reintentar antes de servir) y para un producto de
  trazabilidad verificar después de servir no es aceptable. Coste: latencia percibida — medir latencia
  total como nueva referencia (sustituye al TTFT 1,76 s de S10).
- **Métrica de fidelidad propia, por fases, sin RAGAS.** Fase 1: solo deterministas (recall de
  cláusulas obligatorias, precisión de citas, verificadas literal con `not_found`/`misattributed`
  separados, frases factuales sin marcador). Fase 2, solo si hace falta: juez LLM **por cita**
  (afirmación ↔ cláusula citada, no ↔ contexto entero), reutilizable como capa 3 en línea. RAGAS
  descartado: faithfulness ciega a atribución falsa y sesgada a favor de la expansión; answer
  relevancy castiga el rechazo correcto; API inestable; configuración de LLM paralela al wrapper.

### Propuestas (pendientes de confirmar antes del brief)
- **Golden set de generación como versión nueva** (no editar v2): referencia = cláusulas obligatorias
  + estratos de síntesis condicional y abstención.
- Añadir veredicto **`misattributed`** a la validación literal.
- Política ante veredictos **graduada** (rechazo/reintento, degradar estado, servir con estado) —
  sin streaming, todas las opciones están disponibles; concretar en el brief.
- Augmentation en RAG-ECSS limitada a **deduplicar, presupuestar, ordenar, cabecera determinista**;
  abstractiva excluida.

### Abiertas
- ¿Los chunks ➕ se distinguen hoy en el prompt de los recuperados, o entran como evidencia indistinta?
  (verificar con CC).
- ¿Qué tipos de síntesis (acumulativa / condicional / autoridad / contradicción) y qué casos de
  abstención cubre el golden set v2?
- Inflación de autoridad (NOTE como requisito, *shall*↔*should*): ¿determinista con `authority`,
  `type` y la cita literal (capa 1) o semántica (capa 3)?
- Juez LLM vs modelo NLI especializado para la capa 3/fase 2 (coste CPU, como el reranker).
- ¿La consulta de retrieval filtra por versión de embedding? (seguro barato; verificar con CC).
- Umbrales de abstención distintos por tipo de consulta (puntual vs enumerativa), por la asimetría
  omisión > irrelevancia.

### Diferidas
- Compresión extractiva por filas en tablas → grupo re-chunk.
- Re-anclaje del golden set ante re-chunk: juicios a nivel de cláusula vs versión nueva con mapeo →
  grupo re-chunk.
- Consistencia entre muestras → descartada para MVP (coste N× sobre stream).
- Monitorización en producción sin referencia → cuando haya exposición pública.
- Pesos de recencia → no aplica (revisión = filtro).

---

## 9. Sesión en vivo

**Ideas destacadas.**
- La evaluación de un RAG es un **ciclo continuo medir → mejorar → volver a medir**, no un test
  determinista; un 100 % no es objetivo realista.
- La evaluación avanzada **llega más tarde** que otras piezas: primero pipeline y datos, después
  evaluación sofisticada para subir calidad de forma progresiva. (En RAG-ECSS la medición de retrieval
  llegó en S10, antes que la de generación: coherente con esta secuencia.)
- **Mantenimiento del corpus**: si la fuente cambia, la base vectorial también; para proyectos grandes,
  arquitecturas orientadas a eventos y versionado de documentos/chunks.
- **Fuentes contradictorias** sin respuesta cerrada en clase; el material del programa (artículo 2) era
  la respuesta.

**Preguntas y respuestas.**
- *¿Encaja el patrón critic o solo se valida el resultado?* — Es testing del RAG: valida la relación
  recuperación–generación, no amplía funcionalidad. La recuperación se comprueba directamente; la
  generación necesita un LLM para evaluar. (Matiz para RAG-ECSS: parte de la generación sí se
  comprueba sin LLM — citas literales, marcadores, cláusulas obligatorias.)
- *¿Rangos, dado el no determinismo?* — Existen rangos y tolerancias; las métricas son agregadas sobre
  varias consultas. Umbrales por proyecto: no hay valor universal de faithfulness ni de recall.
- *¿Evaluar con un modelo (DeepSeek) y producir con otro?* — Posible y potencialmente más barato, pero
  los modelos están entrenados distinto y los resultados pueden cambiar. (Corolario: el juez es parte
  de la versión de la medida; cambiarlo invalida la comparación con ejecuciones previas.)
- *¿Documentos que se contradicen?* — Sin respuesta cerrada en clase; se remitió al material del
  programa y a seguimiento posterior.
