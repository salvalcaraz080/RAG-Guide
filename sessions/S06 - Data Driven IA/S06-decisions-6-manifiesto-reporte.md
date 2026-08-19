# S06 — Decisions log, brief 6: manifiesto + reporte de procesamiento

> `[agnóstico]` (aplica a cualquier pipeline de datos versionado) / `[dominio]` (RAG-ECSS).

---

## 1. La cobertura solo significa algo si se calcula contra lo declarado `[agnóstico]`

Es la idea que justifica los dos artefactos, y conviene decirla sin adornos: **una ausencia no
es observable desde el propio directorio**. Contar los JSON de `processed/` dice cuántos hay,
nunca cuántos faltan. Toda la sesión ha estado tropezando con eso — los 12 `.doc` invisibles, el
grafo sobre 14 documentos, las tablas descartadas sin traza — y siempre la salida fue la misma:
comparar contra una lista declarada.

El manifiesto es esa lista, convertida en fichero de primera clase; el reporte es la
comparación. Que sean dos ficheros y no uno no es organización: es que **tienen ciclo de vida
opuesto** y mezclarlos produciría el fichero que nadie sabe si editar o regenerar.

## 2. El manifiesto sustituye al inventario como fuente de identidad `[dominio]`

Cierra la tensión que el propio Brief 4b dejó anotada: `parse_all` tomaba la identidad del
inventario, que un re-scrapeo sobrescribe **en sitio**. Si el inventario cambiaba entre una
descarga y un re-parseo, la identidad de los JSON cambiaba y nada lo registraba.

El manifiesto se deriva del inventario **una vez, por decisión humana**, y se commitea. A partir
de ahí manda él. La diferencia práctica: un cambio de corpus deja de ser un efecto lateral de
volver a ejecutar el scraper y pasa a ser **un diff que alguien revisa en un PR**.

`--check` existe para no perder la conexión: dice si el manifiesto se ha quedado atrás respecto
al inventario, sin escribir nada. Detectar la divergencia es automático; resolverla, no.

## 3. `corpus_version` identifica el conjunto declarado, no su contenido `[agnóstico]`

`{fecha de scrapeo}.{digest8 sobre los doc_id ordenados}`. Dos propiedades buscadas:

- **Re-scrapear sin cambios da la misma versión.** Si el digest saliera del `scraped_at` a
  secas, cada ejecución invalidaría todos los artefactos derivados sin que nada hubiera
  cambiado.
- **Añadir o revisar un documento la cambia**, que es cuando los derivados sí caducan.

Límite declarado en el docstring: **no rastrea el contenido**. Si ECSS republica un fichero bajo
el mismo `ecss_number`, la versión no se entera. Es honesto llamarlo «versión del conjunto
declarado» y no «versión del corpus», y el sitio donde eso se notaría es la comparación de
tamaños que ya hace el guion UAT del downloader.

## 4. Lo que no puede procesarse no se declara `[dominio]`

El manifiesto lista solo los 26 activos con DOCX. Un superseded, o un activo que ECSS publica
solo en PDF, **no entra**.

La razón es la misma que llevó al Brief 1b a no fallar por `skipped_no_docx`: declarar algo que
nadie puede procesar deja la cobertura permanentemente roja por una carencia que no tiene
arreglo, y una alarma que no se puede apagar es una alarma que se aprende a ignorar. El eje
`status` existe en el modelo para poder registrar un superseded conocido si algún día hace
falta, pero hoy no se emite ninguno.

## 5. La severidad sale del tiering, que es lo que faltaba `[dominio]`

El Brief 1b dejó escrito que su exit code no distinguía relevancia «porque no había manifiesto
que lo dijera». Ahora lo hay: el reporte falla si falta un **CORE o RELATED**, y no falla por un
ADJACENT ausente.

Es la primera vez en la sesión que una decisión de severidad se apoya en un dato del corpus en
vez de en un conteo. El reporte **clasifica**; qué hacer con la clasificación es del que lo
llama.

Sub-decisión: la cobertura de identidad degradada **también** falla, sin mirar tier. Una sección
sin número resuelto significa identidad fabricada de vuelta, y eso es incorrecto en cualquier
documento.

## 6. El reporte se commitea por su trayectoria, no por su contenido `[agnóstico]`

Un fichero generado en el repositorio necesita justificarse. La de éste es la misma que la de un
lockfile: **el valor está en el `git log`**. Que la cobertura baje de 26 a 24, o que las tablas
marcadas salten de 8 a 40 entre dos revisiones, es una regresión que ninguna foto suelta enseña
y que el historial del fichero hace evidente.

Y por eso lleva cabecera `GENERATED … do not edit`: es lo único que lo distingue del manifiesto
sin ambigüedad. Hay un test que fija que regenerar **pisa** una edición a mano — editarlo no es
una opción que se desaconseja, es un error que el pipeline deshace.

Formato Markdown y no JSON: se lee en el diff de un PR, que es donde cumple su función. El
manifiesto va en YAML por lo contrario — se edita a mano y admite comentarios.

## 7. El centinela del grafo se retira, no se duplica `[agnóstico]`

El Brief 4b dejó `GRAPH_STALE.md` como fichero suelto en `data/processed/` y anotó la tensión:
«si el manifiesto registra estado del grafo, ahí es donde debería mudarse». Se ha mudado al
reporte, que es su sitio natural — junto al resto de mediciones y reconciliado contra el
manifiesto.

`parse_all` ya no lo escribe: lo **borra** si lo encuentra. Dejar los dos habría creado dos
fuentes de la misma afirmación, que es exactamente cómo una de ellas se queda obsoleta sin que
nadie lo note.

## 8. Qué NO se hizo `[dominio]`

- **No se decide el futuro del grafo** (S07): el reporte solo registra que está obsoleto.
- **No se cruzan las 281 referencias de tabla con el grafo** (S07): se cuentan.
- **No se toca extracción, numeración ni conversión.**

## Tensiones abiertas

- **`--check` detecta la divergencia pero nadie lo ejecuta solo.** Si alguien re-scrapea y no
  vuelve a derivar, el manifiesto queda atrás en silencio hasta que alguien lo mire. El sitio
  natural para automatizarlo sería el arranque de `parse_all` (avisar, no fallar), y no está.
- **`corpus_version` no viaja a los artefactos derivados.** El reporte la lleva, pero los JSON de
  `processed/` no: no se puede mirar un JSON y saber de qué versión del corpus salió. Es barato
  de añadir y probablemente lo pida S07 cuando existan embeddings.
- **El estado del grafo en el reporte está escrito a mano en el render**, no derivado. Dice
  «construido sobre 14 documentos» porque lo sabemos, no porque el grafo lo declare. Cuando S07
  decida su futuro, o el grafo registra su propio `corpus_version` o esa frase caduca sin
  avisar.
