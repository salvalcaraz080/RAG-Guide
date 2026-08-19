# S06 — Decisions log, brief 5-pre: diagnóstico de tablas

> `[agnóstico]` (aplica a cualquier extracción de documentos ofimáticos) / `[dominio]`
> (RAG-ECSS y el corpus ECSS).
>
> Diagnóstico puro: no extrae tablas, no toca el parser. Su producto es un informe que el
> humano lee para decidir la taxonomía del Brief 5.

---

## 1. La agrupación por «firma» existe para muestrear, no para clasificar `[agnóstico]`

El brief prohíbe clasificar automáticamente, y con razón: «matriz de criticidad» vs «caja de
maquetación» es juicio, y un heurístico se equivocaría en silencio. Pero sin ninguna agrupación
el informe serían 600 tablas en fila y nadie las miraría.

La salida fue una **firma estructural gruesa** (`combinada · datos · pequeña`) que solo sirve
para **agrupar muestras**, y que el propio informe rotula como *«agrupación solo para
muestrear, no es taxonomía»*. La firma se compone de señales ya reportadas por separado, así
que el humano puede ignorarla y mirar los números crudos. Un nombre de grupo no es un veredicto
mientras el material para contradecirlo esté al lado.

`layout_candidate` sigue la misma regla: se llama *candidate*, se señala, y la tabla **se mide
y se muestra igual** — hay un test que lo fija, porque la tentación de filtrar el ruido antes
de que nadie lo haya mirado es exactamente cómo se pierde contenido sin enterarse.

## 2. La cláusula contenedora se cuenta, no se recalcula `[agnóstico]`

Para saber bajo qué cláusula vive una tabla hacía falta recorrer el cuerpo del documento en
orden, y ahí había dos caminos: reimplementar el seguimiento de secciones del parser, o
apoyarse en lo que el parser ya produjo.

Se eligió lo segundo: se recorre el cuerpo contando **headings numerados** y ese contador indexa
en `parsed.sections`. La numeración no se recalcula en ningún momento, así que este diagnóstico
**no puede divergir del parser** — que es justo lo que pasó en el diagnóstico de numeración,
donde dos implementaciones equivocadas igual se daban la razón.

Se comprueba además que el recorrido y el parser coinciden en número de headings; si no, el
documento se marca con `error` y no se reporta una ubicación posiblemente falsa.

Sub-decisión de fidelidad: el resolver **solo avanza con párrafos de nivel de cuerpo**, no con
los de dentro de celdas — porque `doc.paragraphs`, que es lo que ve el parser, tampoco los
incluye. Es una desviación deliberada respecto a Word, y se prefiere porque mantener la
alineación con el parser vale más aquí que la fidelidad absoluta.

## 3. Los glifos de Wingdings son señal, no basura `[dominio]`

El primer humo sobre un documento real **reventó**: una celda con ``, glifo de Wingdings,
que la consola cp1252 de Windows no sabe codificar. Un informe muriéndose a mitad del volcado de
muestras, que es justo su razón de ser.

Lo fácil habría sido silenciar la excepción. Pero mirando qué era ese carácter aparece lo
interesante: **es la marca de aplicabilidad de una matriz ECSS** — el «✓» de una tabla de
criticidad. O sea, no es texto (su significado depende de la fuente instalada) pero tampoco es
ruido: es contenido, y de los que distinguen una matriz de una tabla de prosa.

Resultado: se cuenta como señal propia (`symbol_cells`, rotulada en el informe como *huella de
matriz de aplicabilidad*), se pinta como `[sym]` en las muestras, y el `stdout` se reconfigura
con `errors="replace"` como red de seguridad. Tres capas para que el informe no pueda morir por
no saber pintar un carácter.

## 4. Umbrales gruesos y declarados `[agnóstico]`

`PROSE_CELL_CHARS = 60` separa «párrafo» de «token». Es arbitrario y el comentario lo dice: no
pretende clasificar, solo dar al humano un eje ordenable. Lo mismo con «grande» = más de 10
filas y «prosa» = más del 40% de celdas largas.

La disciplina que sí importa: esos umbrales **solo afectan al agrupamiento de muestras**, nunca
a qué se mide ni a qué se conserva. Cambiar el umbral cambia cómo se ordena el informe, no los
datos.

## 5. Qué NO se hizo `[dominio]`

- **No se extrae ninguna tabla** al corpus, ni se toca el parser ni los JSON.
- **No se decide el tipo** de ninguna tabla — el informe no tiene columna «tipo».
- **No se decide la unidad** (tabla/fila/sección) ni cómo hereda la cita. Es Brief 5 + S07.

## Tensiones abiertas

- **`layout_candidate` puede estar tragándose contenido real.** Una tabla de una columna con 309
  caracteres es maquetación con casi total seguridad; una de una columna con datos cortos podría
  no serlo. El informe las agrupa juntas y por eso las muestra: el filtro es del humano.
- **Las tablas anidadas se cuentan pero no se recorren.** Una tabla dentro de una celda aparece
  como señal del contenedor, sin señales propias. Si el corpus tiene anidamiento significativo,
  el Brief 5 necesitará decidir si son una unidad o dos.
- **Una tabla huérfana (sin cláusula contenedora) no tiene identidad que citar.** El informe las
  cuenta y las muestra; qué hacer con ellas es decisión del Brief 5, y el número dirá si es un
  caso de borde o un problema.
