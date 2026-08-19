# S06 — Decisions log, brief 7: tests y validación del extractor de referencias

> `[agnóstico]` (aplica a cualquier sistema que extraiga estructura de texto) / `[dominio]` (RAG-ECSS).

---

## 1. Un test escrito con la lógica del extractor no valida el extractor `[agnóstico]`

Los tests de mecánica que entrega este brief fijan comportamiento: dado este texto, sale esta
referencia. Son útiles —evitan regresiones, documentan la ventana de asociación— y son
**incapaces** de decir si el extractor acierta. Comparten el concepto con el código que prueban;
si el concepto está mal, coinciden los dos en el mismo error.

No es una precaución teórica. En el Brief 4, el parser y el diagnóstico de numeración se dieron
la razón mutuamente durante todo un diagnóstico: la columna «coinciden» estaba llena y ambos
estaban equivocados. Lo destapó una persona mirando el render, no un test.

De ahí la separación que estructura el brief: **mecánica automática, acierto humano**. Y de ahí
que el arnés se ejercite en tests sobre ground truth **sintético** —donde la respuesta se conoce
de antemano— y nunca sobre el corpus: sobre el corpus no hay oráculo.

## 2. Medir recall exige muestrear la fuente, no la salida `[agnóstico]`

El instrumento de recall que existía (`recall_sample.csv`) muestreaba requisitos «cuyo texto
contiene un disparador de referencia (`ECSS-` o `clause`)» — el mismo disparador que usa el
extractor. Su propio docstring lo decía: *«References phrased with no such token are out of
scope for a regex extractor and are not counted as misses»*.

Eso **define fuera del universo** exactamente el fallo que se quería medir. Lo que produce ese
instrumento es acuerdo con el extractor, no recall: por construcción no puede bajar del 100%
por una referencia redactada sin token. Queda superseded, y el arnés lo dice en voz alta cuando
lo encuentra en lugar de ignorarlo —descartar en silencio ha sido el modo de fallo de toda la
sesión.

El instrumento nuevo (`scripts/recall_groundtruth.py`) invierte la dirección: parte del
documento, no de la salida, y **permite añadir filas**. La columna `origen`
(`extractor` / `llm` / `humano`) es lo que hace medible el caso 4 del brief —la referencia que
ni el extractor ni el LLM ven—: si sólo se pudieran confirmar o tachar candidatos, ese caso no
tendría dónde aparecer y el número reportado sería el recall combinado de las dos máquinas,
optimista en una cantidad desconocida.

## 3. Dos recalls, porque hay dos fallos distintos y un solo número los confunde `[dominio]`

Al preparar el instrumento salió un hecho que cambia la lectura de cualquier cifra de recall:
**el extractor sólo ve texto de requisito**. `Section` no guarda cuerpo; `processed/` no contiene
prosa. Un documento sin requisitos da cero referencias por construcción, y así aparecen los seis
handbooks — pero también `ECSS-E-ST-40-07C Rev.1`, con **596 requisitos, 169k de texto y cero
menciones `ECSS-` en ellos**, y `ECSS-E-ST-70-41C`, con 3185 requisitos y 18 referencias.

Un recall medido contra el documento renderizado mezclaría entonces dos fallos que se arreglan
distinto:

- la referencia que **estaba en un requisito** y el regex no vio → fallo del extractor;
- la referencia que estaba en prosa, tabla o anexo y el extractor **no podía ver** → techo del
  parser, que ningún ajuste del regex sube.

Por eso la plantilla lleva `en_texto_de_requisito` y el arnés reporta las dos cifras. El coste
para quien etiqueta es una columna; el beneficio es no mandar a arreglar el regex por un
problema que no es del regex.

## 4. El arnés devuelve fallos, no sólo porcentajes `[agnóstico]`

Un 82% no dice qué arreglar. El falso positivo concreto —su texto de mención, su sección, qué
eje falló— sí. El arnés lista falsos positivos y falsos negativos con procedencia, y el CLI
recorta a 10 con `--all` para verlos todos.

Consecuencia de diseño: las funciones de scoring **devuelven datos** (`PrecisionScore`,
`RecallScore`) y sólo el CLI imprime. Eso es lo que permite ejercitarlas en tests; un arnés que
sólo imprime no se puede verificar más que leyéndolo.

Detalle que se corrigió al probarlo: una fila **en blanco no es una fila etiquetada**. La versión
anterior normalizaba el vacío a «no» y lo contaba como referencia alucinada — habría reportado
precisión 0% sobre un CSV sin tocar. Sin juicio no hay dato.

## 5. Elegir el documento de recall es una decisión con coste, y ahora tiene datos `[dominio]`

El paso de conteo (`--survey`) reporta referencias, requisitos, secciones y **kilo-caracteres de
texto de requisito** por documento. El último es el que importa: el coste real del recall es
leer el documento entero a mano, y ese coste se mide en texto, no en número de secciones.

Los CORE y su densidad, sobre los 26 documentos (**329 referencias**, frente a las 273 del grafo
construido sobre 14):

| documento | refs | reqs | texto | refs/req |
|---|---:|---:|---:|---:|
| ECSS-E-ST-10C Rev.1 | 55 | 356 | 69k | 0.15 |
| ECSS-E-ST-40C Rev.1 | 44 | 782 | 128k | 0.06 |
| ECSS-Q-ST-80C Rev.2 | 43 | 347 | 61k | 0.12 |
| ECSS-Q-ST-40C Rev.1 | 35 | 329 | 60k | 0.11 |
| ECSS-E-ST-70-41C | 18 | 3185 | 661k | 0.01 |
| ECSS-E-ST-40-07C Rev.1 | **0** | 596 | 169k | 0.00 |

`ECSS-Q-ST-80C Rev.2` es la recomendación: CORE, densidad alta y el texto más corto de los CORE.
`ECSS-E-ST-40C Rev.1` es el otro CORE, con el doble de lectura.

## 6. La muestra de precisión se regeneró sobre el corpus de hoy `[dominio]`

Estaba dibujada cuando el universo eran menos documentos y 273 referencias. Medir la precisión
del extractor de hoy contra una muestra de su salida de ayer daría una cifra que no describe a
ninguno de los dos. Como no tenía ni una fila etiquetada, regenerarla no costó trabajo humano —
50 referencias, estratificadas 30/12/8 (con cláusula / sin cláusula / intra-doc), seed fija.

De paso, el sampler dejó de escribir el `recall_sample.csv` superseded cuando no se le pide
muestra de recall: una cabecera huérfana con el nombre del instrumento equivocado invita a
rellenar justo lo que no hay que rellenar.

## 7. Un instrumento sin procedimiento escrito no es reproducible `[agnóstico]`

La validación entró en `corpus/RUNBOOK.md` como **paso 8**, con la misma forma que el resto del
pipeline: qué lee, qué escribe, precondición, comandos exactos y criterio de verificación
observable. Incluye los criterios de etiquetado columna a columna y las reglas de desempate.

El motivo no es orden documental. El etiquetado es juicio humano, y un juicio humano sin criterio
escrito **no es comparable consigo mismo**: la tanda de dentro de tres meses mediría otra cosa que
la de hoy y la diferencia se leería como una regresión del extractor. Las reglas de desempate
—la falta de letra de revisión no es error, los ejes no se juzgan si la referencia no existe, la
cláusula que coincide con la de la sección fuente es sospechosa— existen para eso.

El paso 8 se declara **fuera del pipeline de transformación**: no produce datos, mide. Y sus tres
herramientas viven en `scripts/` y no en `corpus/diagnostics/` porque **escriben en `data/`**, que
es justo la frontera que define `diagnostics/`.

## 8. Los dos fallos que el repaso encontró estaban donde no miraban los tests `[agnóstico]`

Ambos en el camino de impresión del arnés, y ninguno visible en la suite:

- **El listado de fallos salía sin destino.** `_fail` leía `target_doc` / `target_clause`, que son
  los nombres del inventario de recall; la muestra de precisión los llama `target_doc_id` /
  `target_clause_id`. Los fallos se listaban como `(sin target)`, obligando a volver al CSV para
  saber de qué hablaban — que es exactamente lo que el listado existe para evitar. Lo tapaba un
  *fixture* que usaba los nombres de la otra columna: el test pasaba porque el test estaba mal.
- **El arnés reventaba justo cuando había algo que decir.** La consola de Windows es cp1252 y el
  listado imprimía `→` (U+2192), que no existe en esa página de códigos. Con cero fallos, ninguna
  ejecución lo tocaba; con el primer falso positivo real, `UnicodeEncodeError` en mitad del
  informe. Los tests no lo veían porque las funciones devuelven datos y no imprimen — la misma
  decisión que hace el arnés verificable deja su salida fuera de cobertura.

Tercer arreglo, latente: el lector saltaba **toda** línea que empezara por `#`, no sólo la cabecera
de instrucciones. El texto de requisito viaja en celdas multilínea, así que una línea suya que
empezara por `#` habría partido la fila en silencio. Hoy no hay ninguna en el corpus; el filtro se
acotó al bloque inicial de todos modos, porque «hoy no ocurre» no es una defensa.

## 9. Este brief mide; no toca el extractor `[agnóstico]`

Con la lista de fallos delante da la tentación de ajustar la ventana de ±80 caracteres o el
`break` que empareja por orden en vez de por cercanía. No se ha tocado nada. Ajustar antes de
medir convierte la medida en la de un extractor que ya no existe, y deja sin saber cuál era el
problema real. El orden es: medir, mirar los fallos concretos, y **entonces** decidir qué ajustar
—con la deuda conocida del extractor (documentada desde el diseño del grafo) como hipótesis
principal, no como certeza.

## 10. Qué desbloquea, y qué no `[dominio]`

La cifra entra en la decisión grafo-vs-metadata de S07; no la toma este brief. Lo que sí queda
fijado es que la decisión se tomará **con una medida**, no con una impresión: infraestructura de
grafo sobre referencias con recall del 60% no es infraestructura, es deuda con forma de feature.

Fuera de alcance y anotado: las **281 referencias de tabla** del Brief 5 (el `2 Normative
references` de cada documento) no se validan aquí. Su destino no está decidido y validarlas sería
especular sobre un uso que nadie ha definido.
