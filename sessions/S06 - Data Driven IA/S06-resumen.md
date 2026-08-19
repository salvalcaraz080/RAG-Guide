# S06 — Resumen de sesión (provisional)

> **Módulo 3, Sesión 06 — Análisis, formateo y normalización de datos.**
> Resumen pre-implementación: teoría del máster destilada, agnóstica de proyecto. No es
> consolidado (ese llega tras implementar, con la fricción incorporada) ni Guide (solo lo
> graduado). Aquí: qué enseña la sesión, qué simplifica u omite, y qué queda como candidato a
> graduar. Sin decisiones concretas de implementación — esas van al consolidado.

---

## Encuadre de la sesión

Cinco artículos + ejercicio pre-sesión. La sesión es el **puente CAG→RAG por el lado de los
datos**: no monta retrieval (eso es S07+), sino que establece por qué el CAG se queda corto y qué
estado debe tener un corpus **antes** de vectorizar. El módulo entero se queda en la capa de datos;
el espacio vectorial se difiere explícitamente a S07.

El hilo conductor, repetido en los cinco artículos: **vectorizar es lo último, no lo primero.** El
antipatrón que la sesión combate es "vectorizar para ver resultados rápido y luego pasar seis meses
entendiendo por qué el sistema responde mal de forma intermitente".

---

## 1. El techo del CAG es compuesto, no binario

La contribución conceptual más fuerte de la sesión. El límite del CAG no es "no cabe en el
contexto" — son **cuatro restricciones simultáneas**, y basta que una falle para que la arquitectura
no sea viable en producción:

1. **Context window** — restricción binaria y obvia (¿cabe técnicamente?).
2. **Coste por consulta** — continua; escala linealmente con tokens de entrada. Suele ser la que
   mata el proyecto antes que la primera.
3. **Latencia** — depende del producto, no de la arquitectura (batch nocturno la tolera; asistente
   síncrono no).
4. **Degradación de atención sobre contexto largo** — la más subestimada. *Lost in the middle*: la
   información en la mitad del contexto se recupera peor que en los extremos. Aunque el corpus
   quepa, no se procesa con la misma fidelidad que un fragmento relevante aislado.

**Lectura crítica:** las tres primeras se rellenan con una calculadora; la cuarta
(`quality_holds_with_load`) exige un **instrumento de evaluación** — no es un booleano que se marca
a ojo. Rellenarla por intuición es lo que produce el falso positivo ("parecía funcionar con queries
de prueba") que el propio artículo describe después. La simetría de los cuatro booleanos en el
código de ejemplo es engañosa: tres son medición barata, uno es medición cara.

**Candidato a graduar:** el marco de las cuatro restricciones como test de viabilidad CAG en
Fase 0. Refuerza (no crea) el nodo "corpus bounded+stable → CAG viable" — lo hace defendible ante un
stakeholder con cuatro ejes en vez de uno.

---

## 2. La decisión CAG / RAG / híbrido / fine-tuning

Árbol de decisión sobre cuatro ejes: **volumen relativo al context window, frecuencia de
actualización, requisito de trazabilidad, sensibilidad de los datos.**

**Lo que la sesión acierta:** materializar el árbol como criterios explícitos que un ingeniero debe
poder defender ante un director de producto. Y dos correcciones honestas al hype:

- **Fine-tuning no es alternativa a RAG**, es una capa encima cuando el retrieval ya trae lo
  correcto pero el modelo lo *presenta* mal (estilo, terminología, formato). Usarlo como sustituto
  de un retrieval mal diseñado es "enseñar al modelo a memorizar lo que debería estar buscando".
- **RAG no es más barato que CAG.** Añade coste de embeddings, de base vectorial, de operación del
  pipeline de indexación, de re-indexaciones. En corpus que caben en contexto, RAG puede ser *más
  caro*. La elección se hace por **viabilidad y funcionalidad, no por coste**.

**Lo que la sesión equivoca (importante):** el árbol afirma *"trazabilidad obligatoria → RAG puro,
porque CAG no puede atribuir a fragmentos concretos"*. **Es falso.** La atribución no depende de
CAG-vs-RAG; depende de que el material inyectado lleve **coordenadas estructurales**. Un CAG con
contrato de cita las tiene. La afirmación confunde dos ejes ortogonales.

Lo que **sí** es cierto, enterrado debajo y mejor formulado: **la precisión de la cita es
inversamente proporcional al tamaño del material inyectado.** Con un fragmento, la cita candidata es
informativa; con el corpus entero inyectado, "cité estas N mil unidades" no discrimina nada. Ese es
el argumento honesto por el que la trazabilidad *fina* empuja hacia retrieval (seleccionar el
subconjunto) — no porque CAG no pueda citar, sino porque citar sobre demasiado material es no citar.

**Candidato a graduar:** la corrección — *trazabilidad y CAG/RAG son ejes ortogonales; lo que la
trazabilidad fina exige es reducir el material citable por respuesta, y eso es lo que empuja a
retrieval*. También los dos trade-offs honestos (fine-tuning como capa, RAG no-más-barato).

---

## 3. Offline vs online: la línea que parte la arquitectura

El pipeline RAG canónico (ingest → parse → chunk → embed → retrieve → generate) **no es un flujo
lineal en tiempo real.** Se parte en dos pipelines con restricciones distintas:

- **Offline (ingest→parse→chunk→embed):** background, disparado por eventos (subida de documento,
  refresco programado). Presupuesto de latencia: minutos u horas. Puede usar modelos pesados
  (OCR, embeddings grandes, validadores estrictos).
- **Online (retrieve→generate):** síncrono, con usuario esperando. Presupuesto estricto (típ. <3 s).
  Quirúrgico: búsqueda vectorial + construcción de prompt + llamada al LLM.

Materializar esta separación es la primera decisión estructural del módulo. Mezclar responsabilidades
(indexar mientras se atiende una query) es un antipatrón que cuelga el servicio.

**Lectura crítica — contradicción en el material:** el artículo advierte del servicio que se cuelga
indexando mientras responde, y a renglón seguido propone `BackgroundTasks` de FastAPI para el
pipeline offline. Pero `BackgroundTasks` corre **en el mismo proceso y el mismo event loop** — un
parseo o embedding CPU-bound bloquea el endpoint de query igual. La separación *real* exige **proceso
distinto** (worker con cola, o script fuera del servicio), no una background task. El ejemplo
traiciona su propio principio.

**Candidato a graduar:** el nodo offline/online como separación arquitectónica de primer orden, con
la advertencia de que "background task en el mismo proceso" **no** es separación real — la frontera
es de proceso, no de función async.

---

## 4. Calidad del dato como variable de control

Tesis central del módulo: *no amount of clever chunking or fancy architecture can fix fundamentally
bad data.* Un RAG no genera información, la recupera y la presenta: si recupera ruido, presenta ruido
bien formateado; si recupera info obsoleta, presenta desinformación con cara de respuesta
autorizada.

**Tres modos de fallo del antipatrón "vectorizar primero":**

- **Mezcla silenciosa de versiones** — dos versiones contradictorias del mismo registro coexisten;
  el sistema recupera la que el chunker indexó primero. Diagnosticar esto en producción lleva
  semanas porque *no se rompe*: da respuestas inconsistentes que parecen "ruido del LLM".
- **Fuentes podridas** — documentos válidos en forma pero con contenido obsoleto; generan respuestas
  seguras sobre información incorrecta.
- **Gaps invisibles** — el sistema responde bien donde hay datos y **genera información plausible y
  falsa donde no los hay**, porque nadie le dijo que esa categoría de pregunta no está cubierta.

Los tres comparten raíz: nadie miró los datos antes de procesarlos. Y el problema es traicionero
porque **el sistema parece funcionar al principio** — con corpus pequeño y queries elegidas por el
equipo. La degradación aparece cuando el corpus crece y llegan preguntas no anticipadas.

**Candidato a graduar (transversal, fuerte):** los tres modos de fallo son instancias de un mismo
principio que ya vive en la Guide por otro nombre — **corrección degradada en silencio**. El más
peligroso de los tres (gaps invisibles) es el que un sistema con contrato de rechazo/grounding debe
convertir en fallo *declarado* en vez de *silencioso*. La sesión da vocabulario de negocio
("silent version mixing", "invisible gaps") a un nodo que ya teníamos.

---

## 5. Inventario y catálogo: auditar antes de tocar

El primer paso operativo no es vectorizar, es **construir el censo** de fuentes: qué existe, dónde
vive, quién la mantiene, formato, volumen, y — la pareja reveladora — **periodicidad declarada vs
observada** (una fuente "mensual" cuya última modificación es de hace siete meses no es mensual, es
abandonada).

Sobre el censo, una **evaluación de calidad por cuatro dimensiones** (completitud, consistencia,
actualidad, fiabilidad), con una regla **no-compensatoria**: no se promedian. Una dimensión en 1-2
envenena el retrieval aunque las otras sean excelentes (completitud=5 + fiabilidad=1 = "datos
completos que pueden ser mentira", lo peor para RAG). Cada dimensión es condición necesaria.

Todo se materializa en un **catálogo YAML versionado** que *el pipeline lee al arrancar* para saber
qué procesar, qué excluir y qué metadatos propagar — no es documentación muerta, es artefacto de
software (validado con schema, versionado en git, con decisión de inclusión/exclusión por fuente y
motivo registrado).

**Lectura crítica — dependencia del tipo de corpus:** todo este aparato presupone un corpus
**heterogéneo, multi-fuente, sucio, vivo**. Su valor es proporcional a esa heterogeneidad. Para un
corpus de fuente única, homogénea y estable, la mayor parte del formulario (owners, método de
acceso, periodicidad observada) se desinfla por infraestructura ausente — pero **los tres modos de
fallo y la regla no-compensatoria sobreviven**, trasladando la unidad de evaluación de *fuente* a
*documento/tipo post-parseo*.

**Trade-offs honestos que la sesión da bien:**
- Catálogo YAML en repo vs plataforma profesional (DataHub, Collibra…): el YAML es correcto hasta
  que las fuentes crecen "en serio" o llega un mandato regulatorio. Saltar a la plataforma sin pasar
  por el YAML da "una herramienta cara y vacía".
- Auditoría **suficiente para arrancar** vs exhaustiva: las fuentes son móviles; auditar solo lo que
  entra en el primer release. El catálogo crece con el proyecto, no antes.
- **Excluir deliberadamente** con motivo registrado es disciplina, no desidia. En RAG las fuentes
  malas no se manifiestan como ruido aleatorio (fácil de detectar) sino como respuestas seguras a
  info incorrecta.

**Candidato a graduar:** la regla no-compensatoria de calidad (las dimensiones no se promedian);
el catálogo/manifiesto como artefacto que la ingesta lee (no documento muerto); "auditoría suficiente
para arrancar, no exhaustiva"; exclusión-con-motivo como higiene.

---

## 6. Pipeline de extracción: el Document canónico y las tres capas

Ante múltiples formatos, el patrón no es "todo a `unstructured` y a correr", sino una **arquitectura
modular con un contrato común**: un objeto canónico `Document{content, metadata}` que todo parser,
sea cual sea el formato de entrada, debe producir. Chunking/embedding/retrieval operan **solo** sobre
`Document`, sin saber si vino de PDF o de JSON.

**Tres capas con fronteras claras:**
- **Loaders** — "cómo llego al fichero" (paths, URLs, auth, S3). No saben qué hay dentro.
- **Parsers** — "qué hay dentro" (eligen librería por formato, producen representación intermedia
  *específica del parser*, no el canónico todavía).
- **Normalizers** — "cómo convierto mi salida al contrato canónico".

La razón de tres capas y no dos (parser→Document directo) es **testabilidad**: los parsers son
lógica pesada dependiente de librerías; testearlos contra su representación intermedia (fácil de
mockear) y los normalizers contra el contrato canónico mantiene cada test enfocado.

**Metadatos de triple origen**, combinados por el orquestador: del **catálogo** (antes de tocar el
documento: fuente, owner, sensibilidad), del **parser** (después: título, autor, página, sección),
del **pipeline** (en el momento: timestamp de ingesta, versión de parser, config). Los del catálogo
se aplican **después** del parser (defensa en profundidad: un parser buggeado no puede falsificar el
`source_name`).

**Estrategias por formato — lo útil, agnóstico:**
- **JSON**: no es "extraer texto" sino *decidir qué representación textual* entra al RAG. `json.dumps`
  entero genera embeddings ruidosos (mezcla claves técnicas con valores); solo valores pierde
  contexto. Mejor: **renderizar a markdown estructurado** — un parser que conoce el schema.
- **TXT**: esconde trampa. Una transcripción no es texto plano homogéneo; tratarla como bolsa de
  texto pierde la señal *quién dijo qué*. Parser consciente del formato → turnos enriquecidos con
  speaker/timestamp.
- **XLSX**: "parece tabular pero rara vez lo es de verdad" (celdas combinadas, fórmulas, múltiples
  tablas por hoja, formato condicional que codifica info). Tabla pura → markdown; estructura compleja
  → caso especial o fuera del corpus.
- **DOCX**: amable — estructura semántica explícita (estilos, headings) que preserva jerarquía. Un
  parser por-heading emite un Document por sección con el heading como `section_title`.
- **PDF**: "el infierno" — formato de presentación, no de contenido; estructura implícita en
  posiciones/fuentes. Tres niveles: texto plano rápido (`pypdf`/`pdfplumber`, pierde tablas y
  layout) → mejor layout (`pymupdf`) → visión por computador (`unstructured hi_res`, detecta tablas y
  escaneos, orden de magnitud más lento y caro). Decisión **por fuente en el catálogo, no por
  documento**.

**`unstructured` como navaja suiza — trade-off honesto:** unifica interface y soporta 20+ formatos,
pero mete cientos de MB en la imagen (Tesseract, PyTorch, modelos), `hi_res` es un orden de magnitud
más lento, y es **opaco** (cuando falla una detección, depurar es difícil porque decide un modelo
neuronal). Regla operativa: **parsers nativos para formatos predecibles (<5-6, bien entendidos),
`unstructured` reservado para PDF cuando lo necesita (tablas/escaneo) y como fallback de formatos
exóticos.** No convertirlo en punto único de dependencia que oscurece todo el pipeline.

**Candidato a graduar:** el Document canónico como contrato que homogeneiza downstream (con la
condición de aplicabilidad: su valor escala con la heterogeneidad de formatos — para fuente única,
aplanar puede ser destructivo si la representación intermedia es más rica que `content: str`); las
tres capas por testabilidad; metadata de triple origen con catálogo-sobrescribe-parser como defensa
en profundidad; la regla native-vs-`unstructured` por conteo de formatos; **la unidad de recuperación
no tiene por qué ser la de inyección ni la de citación ni la de embedding** — cuatro unidades
potencialmente distintas que se colapsan por defecto si no se nombran (JSON→markdown y
TXT→turnos-enriquecidos son casos donde *lo que se embebe ≠ el dato crudo*).

---

## 7. Limpieza, normalización y validación

El contrato de forma (Pydantic: hay `content`, hay `metadata`) **no es** el contrato de contenido.
Dos Documents pueden cumplir el schema y ser incompatibles para el RAG.

**Cuatro familias de "suciedad":**
- **Heterogeneidad de formato** — la misma cosa escrita de N maneras (fechas, monedas, IDs de
  cliente). Veneno para embeddings: dos fragmentos del mismo concepto caen en regiones distantes del
  espacio vectorial. *El retrieval pierde la señal que el equipo asumía obvia.*
- **Duplicados con divergencias** — mismo registro, valores distintos; el sistema recupera el que se
  indexó primero, sin control del equipo.
- **Nulos disfrazados** — `"N/A"`, `"TBD"`, `"-"`, cadena vacía: técnicamente válidos, se vectorizan
  como si fueran contenido real, el RAG los presenta con autoridad.
- **Fuera de rango** — negativos donde no cabe, fechas fin < inicio. Impacto asimétrico: rara vez se
  recuperan, pero cuando lo hacen generan respuestas de alta confianza sobre afirmaciones absurdas.

**Dónde va la limpieza:** módulo **único y separado**, no parcheada en chunker/embedder/retriever.
Repartirla la hace no-auditable (reglas dispersas en condicionales), no-testeable (mockear
validaciones de otra capa) y sin punto único donde un fallo pueda detener el pipeline. Posición
natural: **entre parser y normalizer**.

**Herramienta (para datos tabulares): Pandera** — valida DataFrames columna a columna y fila a fila,
lo que Pydantic hace por instancia. Al fallar devuelve un reporte de *qué filas fallan y por qué*
(con `lazy=True`, todos los errores juntos), que es justo lo que la capa siguiente necesita para
decidir. Permite checks cross-column (`si status=signed, total>0`) que un schema por-instancia no
expresa. `strict=True` (rechaza columnas no declaradas) + `coerce=False` (la limpieza ya coercionó) =
separación de responsabilidades. El schema Pandera es **artefacto vivo versionado**, equivalente
para datos del catálogo YAML para fuentes.

**Separación limpieza / validación:** la limpieza *transforma lo transformable* (coerciones
permisivas `errors="coerce"` → NaN); la validación *decide qué hacer con lo que quedó*. Tres
políticas de fallo explícitas:
- **Reparar** — recuperable sin pérdida semántica (fecha con otro formato parseable, currency fuera
  del mapa pero inequívoca).
- **Cuarentena** — grave pero rescatable por revisión humana; no entra al RAG, se preserva con motivo
  en tabla separada. El limbo: ni dentro ni fuera, esperando arbitraje.
- **Descartar** — contaminación clara sin valor; se elimina con log detallado.

**Lectura crítica:** todo el aparato Pandera opera sobre **datos tabulares de campos**. Para corpus
de prosa (sin `total_amount` que pueda ser negativo, sin `currency` con cinco grafías) la máquina no
aplica por ausencia del tipo de dato. Lo que **sí** es transversal y gradúa es el **eje forma vs
contenido** y la **tríada de políticas de fallo** (misma familia que las políticas de guardrail:
exception/fix-with-retry/filter). Con una asimetría fina: en la ingesta el contrato de contenido es
**decidible determinísticamente** (¿total≥0? sí/no); en la generación de un RAG con trazabilidad, el
contrato de contenido (¿la cita es fiel?) **no** es determinista — misma dicotomía, distinta
decidibilidad según la capa del pipeline.

**Trade-offs honestos:** Pandera (ligero, inline, schema-como-código) vs Great Expectations (pesado,
datadocs, profiling, para escala con stakeholders no técnicos); **strict desde el día 1, lo que se
relaja ante fallos es la política, no el contrato**; **normalizar con bisturí, no con motosierra**
(bajar todo a minúsculas resuelve variantes de mayúsculas pero borra "Apple" empresa vs "apple"
fruta) — normalizar primero lo claramente accidental, dejar la señal semántica dudosa para después o
nunca.

**Candidato a graduar:** contrato forma-vs-contenido como eje **transversal a todo el pipeline** (no
solo en output), con la asimetría determinista(ingesta)/no-determinista(generación); las tres
políticas de fallo reparar/cuarentena/descartar; "strict el contrato, relaja la política"; bisturí-
no-motosierra en normalización; la capa de limpieza como módulo único (mismo motivo que "eligibility
predicate en un solo sitio").

---

## 8. PII, anonimización y GDPR: defensa antes del embedding

El control de acceso a nivel de aplicación **no protege el espacio vectorial** — y el motivo es
**estructural, no de implementación**. En una BBDD relacional el atacante necesita una query que
apunte a la columna protegida; en RAG hace preguntas en lenguaje natural, el sistema busca
semánticamente y recupera los chunks que *contienen* el dato sensible en el texto. **El vector no
sabe qué es sensible.** Por tanto la protección tiene que ocurrir **antes del embedding**, no como
filtro en la respuesta.

**Tres modos de filtración:**
- **Directa** — el dato está literalmente en los chunks recuperados. Trivial de explotar, trivial de
  prevenir con anonimización en su sitio.
- **Por agregación** — cada query es inocua; el atacante combina varias para reconstruir. Defenderse
  exige pensar en superficie de información agregada, no en chunks individuales.
- **Por inferencia** — la más peligrosa: ocurre **incluso tras anonimización ingenua**. Reemplazar
  `Juan García, CEO de Acme` por `[PERSON], CEO de [ORG]` no basta: el contexto circundante (sector,
  fechas, importes, geografía) reidentifica. La defensa no es solo anonimizar, es **reducir la
  combinatoria de pistas** alrededor del individuo.

Las tres no requieren acceso administrativo — bastan credenciales legítimas y lenguaje natural.

**GDPR mínimo aplicado al pipeline:** datos personales (definición amplia — también identificadores
indirectos y *combinaciones* que reducen la población a uno); **anonimización irreversible vs
pseudonimización reversible** (mapping table separada); **derecho al olvido** (art. 17 — imposible sin
un mapeo explícito "qué chunks mencionan a X", porque los chunks están vectorizados y dispersos);
**minimización** (¿necesita el sistema los nombres reales, o solo los patrones? — si no los necesita,
pseudonimizar no pierde nada útil y elimina riesgo).

**Herramienta: Presidio** (analyzer detecta + anonymizer transforma), con recognizers custom por
dominio (`PatternRecognizer` sobre regex, con `score` para resolver solapamientos) y
**pseudonimización consistente con Faker + mapping table**.

**El hallazgo transversal más valioso de este artículo (aunque hable de PII):** la
**pseudonimización con token genérico (`[PERSON]`) degrada el retrieval 15-25%** frente a pseudónimo
consistente (`Juan García`→`Carlos Martínez` *siempre*, en todo el corpus), porque el token genérico
**destruye la estructura semántica** — dos chunks del mismo referente caen en regiones distantes del
espacio vectorial. *Es exactamente el mismo fallo que la "heterogeneidad de formato" del punto 7.*
Generalizado y despojado de PII: **la consistencia de la representación superficial gobierna la
coherencia del espacio vectorial.** Cualquier entidad recurrente (un nombre, un identificador, una
referencia estructural) representada de dos formas distintas fractura el retrieval. Esto tiene
evidencia cuantitativa aquí, y es un ancla directa para las decisiones de *qué string representa a
una entidad en el texto que se embebe* (S07).

**Trade-offs honestos:** irreversible (simple, deja de ser dato personal, pero degrada embeddings)
vs pseudonimización (preserva señal, pero la mapping table sigue siendo dato personal); Presidio
funciona **peor en español** (falsos positivos en nombres comunes: "Mar", "Sol", "Cruz") — mitigar
con umbral de score, blacklist, o NER custom; el ruido de la pseudonimización es despreciable cuando
el retrieval opera sobre *patrones* y no sobre *identidad nominal* — cuantificar con benchmarks si la
identidad importa.

**Candidato a graduar:** el nodo **defensa de PII antes del embedding** — condición de aplicabilidad
"corpus contiene datos personales/sensibles Y sistema multiusuario/expuesto"; principio estructural
"app-level access control no protege el espacio vectorial"; los tres modos de filtración con "la
anonimización ingenua no cierra la inferencia". **Y, separado del contexto PII: la consistencia de
representación superficial como determinante de la coherencia vectorial** (15-25% de evidencia) — nodo
agnóstico que ancla la decisión de embedding de S07.

---

## Cierre del módulo (teoría)

El estado-objetivo del corpus al final de M3, antes de vectorizar: **inventariado (censo + calidad),
extraído (Document canónico multi-formato), limpio y validado (forma + contenido, con política de
fallo explícita), y — si aplica — anonimizado (defensa pre-embedding).** Cada decisión versionada,
cada exclusión con motivo, cada invariante con un schema que lo hace cumplir. "No es código
brillante; es un corpus que un equipo de producción podría defender ante cualquier interlocutor:
legal, comercial, técnico, regulatorio."

La vectorización propiamente dicha (embeddings, chunking, modelos, espacio vectorial) se difiere a
**S07 (M3 continúa)**; las BBDD vectoriales y las arquitecturas RAG completas, a M4.

---

## Patrón meta de la sesión (para el consolidado)

Una observación que no es de un artículo concreto sino del conjunto, y que conviene arrastrar al
consolidado: **la sesión está escrita para un corpus heterogéneo/sucio/multi-fuente/privado/vivo.**
Su maquinaria (catálogo de 15 campos, Pandera sobre DataFrames, Presidio + mapping table) es
proporcional a esa suciedad. Para un corpus del perfil opuesto (fuente única, homogéneo, público,
autoritativo, congelado), la mayoría de la máquina **se desinfla por infraestructura ausente** — el
mismo movimiento que "N/A por backend inexistente". Pero en cada artículo **sobrevive un principio**
que no depende del perfil de corpus (modos de fallo, no-compensatoriedad, forma-vs-contenido,
consistencia de representación, defensa pre-embedding) — y varios **renacen más delgados** por la
lente de la trazabilidad en vez de por la del dato sucio. Esa disciplina — *distinguir la máquina
que se desinfla del principio que sobrevive* — es lo que el consolidado tendrá que registrar cuando
la implementación concrete cuál de las dos cosas era cada nodo.
