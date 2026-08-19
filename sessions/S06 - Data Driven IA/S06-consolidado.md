# S06 — Consolidado

> **Paso 5 del ciclo.** Resumen provisional (`S06-resumen.md`) **+ lo aprendido al implementar.**
> No reescribe el resumen ni lo sustituye: la divergencia entre lo que la teoría del máster decía y lo
> que la implementación reveló **es** el material de valor. Histórico — no se reescribe en sesiones
> futuras. De aquí se gradúa a `RAG-GUIDE.md` (paso 6) solo lo que sobreviva la destilación.
>
> **A diferencia del resumen (teoría pura, agnóstica), el consolidado registra también lo de dominio**
> (RAG-ECSS), porque muchos aprendizajes solo aparecieron al chocar la teoría con el corpus real.

---

## 0. El arco de la sesión, y la divergencia teoría-vs-implementación

**Lo que el resumen anticipó (teoría del máster, S06):** la sesión trataba «análisis, formateo y
normalización de datos» — auditar antes de vectorizar, calidad del dato como variable de control,
los tres modos de fallo (mezcla de versiones, fuentes podridas, gaps invisibles), Document canónico,
limpieza/validación con política de fallo, PII pre-embedding. Y el patrón meta que ya se veía: la
máquina del máster (catálogo de 15 campos, Pandera, Presidio) **se desinfla** para un corpus
público/homogéneo/congelado, pero **el principio sobrevive**.

**Lo que la implementación reveló, y que la teoría no podía anticipar:** el módulo menos RAG-like del
máster (origen y tratamiento de datos, intrínsecamente ad-hoc) resultó ser donde el corpus estaba
**más roto de lo que sabíamos**, y donde casi todos los defectos eran del mismo tipo — el que el
propio resumen nombró en abstracto («gaps invisibles») y que la implementación materializó una y otra
vez: **corrección degradada en silencio.** La sesión no fue «normalizar datos»; fue **descubrir, medir
y hacer visible cuánto se perdía en silencio en cada etapa del pipeline.**

La divergencia central: el resumen trataba los tres modos de fallo como *riesgos a prevenir en un
corpus sucio*. La implementación mostró que **nuestro corpus limpio/público/congelado ya los sufría
todos**, por vías que la teoría no contempla: no por datos sucios, sino por un parser que descartaba
en silencio (tablas, documentos `.doc`, referencias tabulares) y por una numeración fabricada que
ningún invariante detectaba. El corpus «limpio» mentía por omisión tanto como el «sucio» del máster —
solo que sin avisar, porque no había suciedad visible que levantara sospecha. **La ausencia de
suciedad evidente hizo el fallo silencioso más peligroso, no menos.**

El resultado en una línea: el corpus pasó de *«14 documentos, numeración fabricada, tablas descartadas
sin traza, referencias sin validar»* a *«26 documentos, identidad de cita fiel y versionada, tablas
extraídas por tipo con conservación comprobada, extractor de referencias con recall del 48% conocido
con precisión»*. No se arregló todo — pero **se sabe qué no está arreglado y cuánto**, que es la
diferencia que define el proyecto frente a «ejecutar».

---

## 1. El nodo raíz: la ausencia no es observable desde lo presente

Casi todos los hallazgos de S06 son instancias de un solo principio, que CC formuló en el Brief 6:
**contar lo que hay nunca dice lo que falta.** `verify_parser` glob-eando `*.docx` veía 14 y reportaba
OK — sin mencionar los 12 `.doc` que ni miró. El grafo se construyó sobre 14 sin notar que faltaban
12. Las tablas se descartaban sin registrar que existían.

**Formulación graduable (agnóstica):** *un sistema que solo inspecciona lo que tiene no puede conocer
sus propias ausencias. Detectar lo que falta requiere una declaración externa de lo que debería estar
— iterar el manifiesto de lo declarado, no el directorio de lo presente. Las ausencias silenciosas son
el modo de fallo dominante en pipelines de datos, y son invisibles por construcción para cualquier
verificación que no compare contra una lista declarada.*

De este nodo cuelgan, como casos: C-15 (12 `.doc` invisibles), la reconciliación de cobertura, el
`parse_all` que itera el inventario, el reporte que se reconcilia contra el manifiesto, y el caso 4 del
recall (referencia que ni extractor ni LLM ven → invisible para «revisar discrepancias»). **Es el hilo
conductor de la sesión entera.** Todo lo demás es este nodo aplicado a una etapa distinta.

---

## 2. Nodos de método (cómo se construye)

**2.1 — Diagnosticar antes de tocar.** Cada brief estructural (conversión, numeración, tablas) empezó
por un diagnóstico que caracterizaba el problema antes de repararlo. No es ceremonia: el diagnóstico de
conversión destapó `soffice.com`; el de numeración destapó que el problema era 502 cláusulas, no 4; el
de tablas destapó que «combinada» son tres problemas. *Escribir el brief de reparación sin el
diagnóstico habría dimensionado mal cada uno.* Agnóstico.

**2.2 — El instrumento de medición es software y puede tener el bug.** Tres instancias en la sesión:
el resolver de numeración contaba `Annex %1` como número (143 falsos positivos); el de tablas confundía
glifos Wingdings y tenía una ventana de muestra mala. **La disciplina: valida el instrumento contra una
verdad externa (render, muestra a ojo) antes de creer su agregado, y sospecha primero del instrumento
cuando el resultado es masivo y uniforme.** Agnóstico. CC lo internalizó — en el Brief 5 lo nombró solo
(«es la tercera vez que un diagnóstico necesita corregirse antes de fiarse de él»).

**2.3 — «Coinciden dos implementaciones» ≠ corrección.** El nodo epistemológico central, y el que más
caro habría salido ignorar. Dos implementaciones pueden acordar **en el error** si comparten hipótesis.
- *Numeración (Brief 4):* el parser y el resolver decían `E.1.1` los dos — coincidían porque compartían
  el mismo error. Las 4.690 «coincidencias» incluían acuerdos-en-lo-correcto y acuerdos-en-lo-incorrecto,
  indistinguibles. El bug (22 citas ambiguas) solo salió porque el paso de medición pidió un número que
  resultó ser 0 y **CC no aceptó el 0**.
- *Recall (Brief 7):* confirmado empíricamente. 7 de 47 referencias reales las fallaron extractor y LLM
  **a la vez**. Un flujo que solo revisara discrepancias habría medido 85% en vez del 62% real sobre ese
  documento. El LLM y el extractor **comparten sesgo** (ambos leen prosa, ambos ojean tablas por encima),
  así que sus errores se solapan.

**Formulación graduable (agnóstica):** *la única evidencia de corrección es la comparación contra una
verdad externa a ambas implementaciones. El acuerdo entre dos falibles no es corrección — es hipótesis
compartida, incluidas las falsas. El falso positivo peligroso no es el desacuerdo (ruidoso, se
investiga) sino el acuerdo-en-el-error (silencioso, se archiva como éxito). Corolario: un número que
sale redondo o nulo cuando esperabas algo es una señal, no un alivio.*

**2.4 — Medir ≠ arreglar.** Auditoría separada de reparación (downloader: Brief 1 audita, 1b repara);
diagnóstico separado de extracción (numeración, tablas); Brief 7 mide el extractor y **no lo toca** —
deja los 7 casos concretos como especificación de la reparación futura. *Un brief de alcance abierto
(«audita y arregla lo que encuentres») es lo que la granularidad evita; el diagnóstico especifica la
reparación con datos, en vez de reparar a ciegas.* Agnóstico.

**2.5 — Un brief que hereda invariantes debe verificarse contra ellos, no describir el resultado.**
*(Autocrítica de método.)* El Brief 4b, tal como lo escribí, tenía dos instrucciones que habrían roto
invariantes del Brief 4: «borrar `processed/`» (habría destruido el grafo que declaramos stale) y
«parsear los `.docx` del directorio» (habría reintroducido C-4, identidad desde nombre de fichero).
CC las atrapó. **El error tenía una forma: escribí el brief pensando en el resultado deseado sin trazar
cada instrucción contra los invariantes que los briefs previos establecieron.** Es la misma disciplina
que le pedimos a CC (tests de caracterización que afirman el comportamiento nuevo) aplicada a escribir
briefs. Nodo de proceso, agnóstico.

---

## 3. El meta-nodo de datos: verifica el artefacto, no el proceso

Toda la sesión converge en un principio que aparece en cinco capas distintas:

| Capa | «No confíes en…» | «Verifica…» | Brief |
|---|---|---|---|
| Red | el `Content-Type` del servidor | magic bytes del fichero | 1b |
| Conversión | el exit 0 de LibreOffice | que el `.docx` exista y sea válido | 2 |
| Numeración | el contador del parser | el número que Word imprime (`numbering.xml`) | 4 |
| Validación | «coinciden las dos implementaciones» | la verdad externa (render, ground truth humano) | 4,7 |
| Ingesta | que un LLM transcriba fiel | — (no se usa LLM en ingesta) | (discusión) |

**Formulación graduable (agnóstica):** *cuando delegas en un proceso cuyo reporte de éxito no controlas
—un servidor, una herramienta externa, otra implementación, un LLM— verifica el artefacto producido
contra una verdad independiente, no el código de retorno ni el auto-reporte del proceso. El éxito lo
decide el artefacto, no quien dice haberlo producido.* Es el nodo del que «form is not truth» era una
instancia; ahora generalizado a todo el pipeline de datos.

**Corolario, extraído de la discusión sobre LLM en la ingesta (dominio + agnóstico):** *un LLM puede
tocar lo que se verifica después (generación con cita auditable contra el corpus) o lo que no es fuente
de verdad (diagnóstico offline que un humano revisa), pero no puede SER el proceso que crea la fuente de
verdad. Un LLM en la ingesta mueve la fabricación al cimiento — donde es permanente (contamina el corpus,
no una respuesta) e indetectable (produce texto plausible sin garantía de fidelidad, en el punto donde
la fidelidad importa más y se detecta menos).*

---

## 4. Nodos de arquitectura de datos

**4.1 — Construction-time guarantee > runbook check.** En vez de un invariante que hay que recordar
comprobar, una estructura donde el caso malo no existe. Instancias de la sesión:
- El rótulo se modela como **no-Section**, no como Section con `number=""` → `sections` significa
  «cláusulas citables» sin excepción que filtrar aguas abajo.
- La `Table` **no tiene identificador propio** → es imposible citarla como si fuera cláusula.
- El reporte generado **se auto-sobrescribe** al regenerar → editarlo a mano no es «desaconsejado», es
  un error que el pipeline deshace.
- Verificación por **conservación**: la suma de tablas por mecanismo iguala el total → descartar en
  silencio es imposible, nada se pierde sin aparecer en algún cubo.

*Formulación (agnóstica): prefiere la garantía por construcción a la instrucción en el runbook. Una
precondición que alguien se va a saltar no es una precondición, es un bug con buenos modales.* (La frase
es de CC, Brief 2.)

**4.2 — El daño aplazado por la arquitectura, no reparado por ella.** Las 455 divergencias de numeración
en handbooks eran invisibles **no porque estuvieran arregladas** sino porque otra decisión (cita cuelga
del requisito, y los handbooks no tienen requisitos) impedía que se manifestaran. *Formulación
(agnóstica): un defecto latente enmascarado por una precondición arquitectónica accidental se activa el
día que la precondición cambia — y entonces contamina retroactivamente todo lo construido encima.* El
día que S07 chunkee por sección (para rescatar los handbooks), las 455 se activan. Por eso el Brief 4
tuvo que repararlas antes, aunque hoy fueran inocuas.

**4.3 — Un defecto latente bajo una precondición accidental (caso menor del anterior).** El
`rename`→`replace`: el bug de Windows dormía porque el downloader solo bajaba lo ausente (destino nunca
existía); la feature de re-descarga lo despertó. *Cuando añades una capacidad que ejercita un camino
antes inalcanzable, sospecha de las APIs cuya corrección dependía de que ese camino estuviera muerto.*
Agnóstico.

**4.4 — Fuente única de verdad: migrar es eliminar, no duplicar.** El centinela `GRAPH_STALE.md` no se
dejó junto a la marca en el reporte — `parse_all` lo *borra* si lo encuentra. *Dejar dos fuentes de la
misma afirmación es exactamente cómo una se queda obsoleta sin que nadie lo note.* Agnóstico.

**4.5 — Todo-o-nada en artefactos derivados-completos.** `parse_all`: se sustituye `processed/` solo si
los 26 declarados produjeron JSON; 25 de 26 «no es casi bien, es un corpus más pequeño ocupando el sitio
de uno completo». *La política ante fallo parcial en un artefacto que se asume completo es todo-o-nada,
no parcial-con-log — un subconjunto ocupando el sitio del conjunto es un fallo silencioso aunque el log
lo diga.* Agnóstico.

**4.6 — Manifiesto declarativo vs reporte generado: ciclo de vida opuesto.** Dos artefactos, no uno: el
manifiesto se **edita** cuando cambia el corpus (fuente de verdad, commiteado); el reporte se **regenera**
con cada pipeline (medición, cabecera de no-editar, commiteado por su *trayectoria* — el `git log` como
historial de regresiones, à la lockfile). *Mezclarlos produce el fichero que nadie sabe si editar o dejar
regenerar.* La cobertura es la reconciliación entre ambos. Agnóstico.

**4.7 — `corpus_version` identifica el conjunto declarado, no su contenido.** `{fecha}.{digest8 sobre
doc_ids ordenados}`: re-scrapear sin cambios da la misma versión; añadir/revisar un documento la cambia;
pero **no rastrea contenido** (si ECSS republica bajo el mismo número, no se entera). *Nombrar la versión
por lo que de verdad detecta —el conjunto declarado, no el corpus— es la honestidad que evita confiar de
más en ella.* Constante-sin-autoridad-falsa aplicado a un identificador. Agnóstico.

**4.8 — Una alarma que no se puede apagar entrena a ignorarla.** No se declara como fallo lo que no tiene
arreglo accionable: un ADJACENT PDF-only permanente no deja la cobertura roja para siempre; un `.doc`
inprocesable no se declara. *No declares como fallo lo que no tiene resolución accionable — la alarma
permanente se aprende a ignorar, y con ella las que sí importan.* Agnóstico.

---

## 5. Nodos de dominio (RAG-ECSS / trazabilidad)

**5.1 — Dos grados de autoridad citable.** *(El mejor nodo de dominio de la sesión, del usuario, no de
ningún artículo.)* El corpus citable tiene dos grados, y el grado gobierna dos cosas — si la cita es
identidad o procedencia, y cuánta fidelidad literal exige:
- **Requisito** (estándar): cita = **identidad** de la sustancia; **transcripción literal** (parafrasear
  una norma es alterarla); puede responder solo.
- **Recomendación** (handbook): cita = **procedencia** de una orientación; **síntesis permitida** (un
  consejo resumido sigue siendo el consejo); **también puede responder solo** (el handbook es fuente
  autónoma, no siempre subordinada al requisito).

*Refina el nodo previo «no parafrasear bajo cita», que era demasiado fuerte: no parafrasear bajo cita
**normativa**; sí sintetizar bajo cita **orientativa**. Pero la síntesis relaja la fidelidad del
CONTENIDO, nunca la de la COORDENADA — una recomendación resumida necesita procedencia tan exacta como
un requisito necesita identidad exacta.* Ancla: S07 (chunking ramifica por tipo), S11 (generación
ramifica el tratamiento; `Citation` gana eje de autoridad, que el grafo ya tiene como `mention_type`).

**5.2 — La autoridad es de la sección, no del documento.** Refutado empíricamente por el diagnóstico de
tablas: 253 tablas informativas dentro de estándares (anexos `(informative)`), 1 tabla normativa dentro
de un handbook (`C.1.1`). *El eje de autoridad se resuelve por sección/anexo, no por tipo de documento.
Lo que el documento DECLARA sobre una sección (`(normative)`/`(informative)`) es más autoritativo que lo
que se INFIERE de su tipo.* Es «resolver desde lo que la fuente declara» (mismo espíritu que numeración
desde `numbering.xml`). Ya implementado en `Section.authority` (Brief 4), heredado por las tablas
(Brief 5). Dominio.

**5.3 — Out-of-material es conjuntivo.** Con el handbook como fuente autónoma (5.1), el rechazo se
endurece: *«no cubierto» requiere ausencia de requisito Y de recomendación, no solo de requisito.* Una
pregunta de guía pura tiene respuesta si hay handbook, aunque no haya norma. El guardrail de scope (S11)
debe consultar ambas autoridades antes de rechazar. Dominio (ancla S11).

**5.4 — Resolver desde la fuente estructural, no contar.** La numeración de sección **no está en el
texto** (la genera Word desde `numbering.xml`); contarla es la fuente del bug (502 cláusulas mal, 457
silenciosas). Resolverla desde el XML produce el número que el ingeniero ve al abrir el DOCX — fiel por
construcción, verificable contra el render. *Con la disciplina: toda ruta dudosa devuelve «sin número»,
no un número adivinado — no numerar es mejor que numerar mal.* Dominio, pero con núcleo agnóstico (*deriva
la identidad de la representación canónica de la fuente, no de un cálculo propio que no puedes verificar
contra ella*). Y §4 del Brief 5-decisions afinó: el nivel jerárquico también sale del número resuelto, no
del estilo — el estilo solo dice «esto es un heading».

**5.5 — La prueba de irreducibilidad es el fracaso de la transformación, no un juicio sobre la forma.**
Para decidir si una tabla es aplanable, CC no *juzga* si lo es — **intenta desplegarla y ve si sale una
rejilla consistente**. Si tras resolver los merges las filas no cuadran, no hay orden lineal fiel →
irreducible (`ragged`). Esto reveló una clase que no habíamos previsto. *Para decidir si una transformación
es fiel, intenta la transformación y verifica el resultado — no juzgues la entrada. El fracaso de la
transformación es la prueba, y a veces revela una categoría no anticipada.* Agnóstico (es 2.3 aplicado a
la clasificación).

**5.6 — La unidad de descarte importa.** Descartar una *columna* de ruido (relleno `Verified`) no es
descartar la *tabla* útil que la contiene; no tener ancla no es motivo para *inventar* una (change-log de
portada → descarte con motivo, no colgarlo de la sección siguiente). *«Descartar ruido» tiene que
especificar la unidad — el ruido dentro de algo útil no convierte lo útil en ruido.* Agnóstico.

**5.7 — Marcar-sin-extraer: el tercer resultado, ni cita ni rechazo.** Para contenido irreducible
(diagramas, anidamiento), el sistema no transcribe ni rechaza: marca `content_extractable: false` y da la
coordenada. Produce (S11) «la cláusula X define esto mediante una estructura que no se reproduce; consúltese
el documento». *Honesto por construcción: existe (no es rechazo) + coordenada exacta + no transcrito (no es
cita infiel). En trazabilidad, «existe, aquí está dónde, no puedo mostrarlo fielmente» es correcto; una
linealización infiel mentiría sobre qué dice la estructura.* Dominio (materializa el principio no-fabricar).

**5.8 — LLM para ground truth: asistente sí, decisor no.** *(De la discusión + confirmado por el recall.)*
El LLM como **asistente de etiquetado** (propone, el humano verifica y **lee las omisiones**) acelera; como
**ground truth autónomo** contamina, porque comparte sesgo con lo medido (7 falsos negativos compartidos,
confirmado). *El ground truth debe ser más fiable que lo medido, no su par. Para recall, «revisar
discrepancias» hereda el recall combinado de los dos falibles y es ciego al caso donde ambos fallan — el
humano debe poder AÑADIR lo que ninguna máquina propuso, no solo filtrar candidatos.* Agnóstico.

**5.9 — Precisión y recall no pesan igual; el peso lo da el dominio.** Extractor: precisión 89% (buena),
recall 48% (mala) — *y no son igual de graves. Una referencia de más es ruido que el reranking filtra; una
de menos es una conexión que el sistema no sabe que existe.* *En un sistema de trazabilidad el falso
negativo (omisión) es el modo de fallo caro, así que el recall manda la decisión.* Es el hilo
ruidoso-vs-silencioso cuantificado: el falso negativo ES la ausencia silenciosa. Dominio con núcleo
agnóstico (*el error más caro depende del dominio; mídelo y deja que el caro gobierne*).

---

## 6. Refuerzos de nodos existentes de la Guide

- **Techo del CAG compuesto (4 restricciones)** — del resumen (teoría). Valida retroactivamente que
  nunca justificamos M2 solo por tamaño. Sin cambio.
- **Structural anchors > positional coordinates** — confirmado más general: el `clause_id` sirve a citar,
  a gobernar/borrar fragmentos dispersos (art. 5 del resumen), y ahora a **anclar tablas** a su sección.
- **Contrato forma/contenido** — confirmado transversal, con la asimetría determinista(ingesta)/
  no-determinista(generación) del resumen, reforzada por 2.3 (la validación de contenido puede necesitar
  verdad externa, no basta el shape).
- **«Combinada» codifica tres relaciones distintas** (Brief 5) — *una señal estructural uniforme puede
  codificar semánticas distintas; contar la señal no es entender el caso.* Refuerza «diagnóstico antes de
  brief». Agnóstico.
- **Rechazar-y-documentar** — S06 rechazó: LLM en ingesta, LLM como ground truth autónomo, MarkItDown
  como reemplazo del pipeline, resolver numeración por conteo. Cada uno con su condición de aplicabilidad
  (no rechazo universal — MarkItDown/unstructured/LLM sirven para heterogéneo-sin-trazabilidad).

---

## 7. Estado que S06 entrega a S07 (anclajes)

- **Grafo-vs-metadata reorientada por datos:** el extractor tiene recall 48%, y el 40% de las pérdidas
  son **referencias tabulares** (las del Mecanismo D, Brief 5) que el extractor de prosa nunca leyó. La
  pregunta ya no es «¿grafo o metadata?» sino «el extractor pierde la mitad, la mayoría por no leer tablas
  — ¿qué se arregla antes de decidir la estructura?». Cualquier estructura de S07 debe alimentarse también
  de las referencias tabulares o nace con el 48%.
- **Chunking ramifica por tipo:** estándar → requisito (transcripción, cita-identidad); handbook →
  sección (síntesis, cita-procedencia). Las 455 latentes se activan al chunkear handbooks por sección —
  ya reparadas por el Brief 4.
- **Las cuatro unidades** (recuperar/inyectar/citar/embeber) siguen abiertas; la de embedding con la
  evidencia del resumen (consistencia de representación gobierna el retrieval, 15-25%).
- **`corpus_version` no viaja aún a los JSON** (tensión Brief 6) — cualquier derivado (embeddings,
  respuestas) debe poder atarse a la versión de la que salió. Ancla S07.
- **Deudas menores:** `--check` del manifiesto no lo ejecuta nadie solo; el estado del grafo en el reporte
  está escrito a mano (caduca en silencio cuando S07 toque el grafo); `ragged` puede marcar de más (0 casos
  en 2 docs probados, sin evidencia sobre los otros 24); el aplanado posicional pierde legibilidad en el
  69% sin cabecera (deliberado, fiel>útil, reconsiderar con datos si molesta).

---

## 8. Nota de proceso sobre la sesión

S06 fue la sesión más larga y menos «teórica» del máster hasta ahora, y la que más trabajo real produjo
— siete briefs (diez artefactos con diagnósticos), todos disparados por **una auditoría que no estaba
planificada** (el usuario pidió repasar el downloader «por si acaso»). Esa auditoría destapó C-15, que
reordenó la sesión entera. *Aprendizaje de proceso: el paso barato de «auditar lo que parece que funciona»
pagó más que ningún brief planificado — porque el corpus limpio escondía el fallo silencioso justo por
parecer limpio.* Es la validación empírica del primer artículo del máster («vectorizar es lo último, no
lo primero») llevada un paso más atrás: **antes de tratar los datos, auditar que son los que crees que
son.**
