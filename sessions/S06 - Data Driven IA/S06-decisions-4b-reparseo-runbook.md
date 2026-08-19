# S06 — Decisions log, brief 4b: re-parseo del corpus + RUNBOOK

> `[agnóstico]` (aplica a cualquier pipeline de datos) / `[dominio]` (RAG-ECSS y el corpus ECSS).
>
> El análisis previo encontró dos puntos donde el brief, implementado a la letra, habría
> deshecho trabajo del Brief 4. Ambos están resueltos aquí.

---

## 1. Borrar `processed/` habría destruido el grafo, no declarado stale `[dominio]`

El brief pedía dos cosas incompatibles sin verlo: «regenerar `processed/` desde cero borrando el
contenido anterior» (decisión 1) y «el grafo no se regenera, queda stale-declarado» (decisión 2).
`data/processed/` contiene **también** `references.json` y `graph.graphml`.

Borrar no es declarar obsoleto: es destruir el artefacto que S07 tiene que mirar para decidir si
el grafo debe existir. Se perderían las 273 referencias extraídas y la decisión abierta se
tomaría sin material.

**Resuelto:** `_replace_document_json` borra solo los JSON por documento y deja intactos los del
grafo, que produce otro paso. `parse_all` escribe además `GRAPH_STALE.md` a su lado, y
`graph.load_graph()` avisa si lo encuentra.

Sub-decisión de dónde vive la marca: la escribe **el comando que invalida el grafo**, no un paso
manual. Quien rompe un invariante es quien todavía se acuerda de que lo ha roto.

## 2. La identidad no puede salir del nombre de fichero — es C-4 otra vez `[agnóstico]`

El paso 3 del brief dice «parsea cada `.docx` de `parse_ready/`». Al pie de la letra, eso
**reintroduce el bug que el Brief 4 acababa de cerrar**, y en el peor sitio posible: el paso que
escribe los artefactos.

```
parse_ready/  ECSS-Q-ST-20C_Rev_2.docx     ← nombre del downloader, lossy
processed/    ECSS-Q-ST-20C_Rev.2.json     ← identidad correcta, con el punto
```

`sanitize_filename` colapsa espacio y punto sobre `_`, así que del stem saldría
`ECSS-Q-ST-20C Rev 2` — un documento que no existe — y `save_parsed` renombraría **todos** los
JSON con la identidad corrupta.

**Resuelto:** `build_parse_plan` itera el **inventario**, no el directorio. La dirección
`ecss_number → sanitize_filename → stem` está bien definida; la inversa no lo está y por eso no
se usa. Tres beneficios, dos de ellos no buscados:

- La identidad que llega al JSON es la canónica, con revisión.
- La cobertura sale de comparar contra lo **declarado**, que es la pregunta correcta («¿está
  todo lo que debería?») en vez de «¿cuántos ficheros había?».
- Un fichero suelto en `parse_ready/` sin fila activa no puede convertirse en documento con
  identidad inventada. Queda reportado, no parseado.

Consecuencia para el RUNBOOK: el paso 3 necesita `data/inventory/`, precondición que el brief no
listaba.

Esto **adelanta parte del manifiesto** (Brief 6) sin construirlo: `parse_all` ya usa el
inventario como fuente de identidad, que es lo que el manifiesto formalizará.

## 3. Todo-o-nada, y un test lo obligó `[agnóstico]`

El brief pedía borrar y reconstruir, «sin `--force` porque no hay nada que forzar». Correcto en
principio: `processed/` es derivable. Pero si el parseo revienta a mitad te quedas con menos JSON
de los que tenías, y eso es lo que el Brief 1b ya había prohibido en el downloader (*una descarga
fallida nunca destruye la copia buena*).

Se implementó staging en directorio temporal… y **el primer test lo tumbó**: el intercambio corría
igualmente, así que un fallo total borraba lo anterior y no ponía nada. El staging estaba, la
condición no.

La política quedó en **todo-o-nada**: se sustituye solo si todos los documentos declarados
produjeron JSON. Un fallo deja `processed/` intacto y sale con código ≠ 0.

Se consideró la alternativa —sustituir lo que sí parseó y reportar el resto— y se descartó: 25 de
26 no es «casi bien», es un corpus más pequeño ocupando el sitio de uno completo. La brecha no
queda oculta (exit code + resumen), pero tampoco se materializa. Si un documento está
permanentemente roto, el arreglo es arreglarlo o quitarlo del inventario, no aceptar en silencio
un corpus menor.

## 4. El RUNBOOK documenta el criterio observable, no el exit code `[agnóstico]`

Para cada paso, el RUNBOOK dice **cómo saber que salió bien** mirando el artefacto, no el código
de salida. Es la lección acumulada de la sesión: LibreOffice devuelve 0 sin producir fichero; el
downloader daba éxito con un HTML de login; el diagnóstico daba «coinciden» cuando ambos lados
fallaban igual.

Por eso el paso 3 se verifica con `identity_complete: true` y no con «terminó sin excepción», y
el 2 con «el `.docx` existe y tiene magic bytes de Word».

También queda escrita ahí, de una vez, la frontera CC/usuario: **ejecutar cualquier paso de este
runbook es del usuario**, porque todos tocan datos reales.

## 5. Qué NO se hizo `[dominio]`

- **No se regenera el grafo.** Stale-declarado; su futuro es S07.
- **No se resuelve grafo-vs-metadata-de-chunk.** Queda registrada y anclada a S07: depende de la
  estrategia de chunking y de validar el extractor, y ninguna de las dos existe. Decidir antes
  sería pre-emptar el nodo otra vez.
- **No se tocan tablas, manifiesto, `raw/` ni `parse_ready/`.**

## Tensiones abiertas

- **`parse_all` usa el inventario como fuente de identidad, y el inventario no está versionado.**
  Un re-scrapeo lo sobrescribe en sitio. Si cambia entre una descarga y un re-parseo, la
  identidad de los JSON cambia sin que nada lo registre. Es exactamente el hueco que el
  manifiesto (Brief 6) tiene que cerrar.
- **Todo-o-nada puede bloquear.** Un documento permanentemente irreparable impide regenerar
  `processed/` entero. Es deliberado —fuerza a decidir sobre ese documento— pero si ocurre, la
  salida es quitarlo del inventario, y eso hoy es una edición manual de un CSV generado.
- **El centinela de stale es un fichero en `data/`, que no se commitea.** Un clon limpio no lo
  tendría, y tampoco tendría el grafo, así que no miente — pero la marca vive junto al artefacto
  que describe, no en el código. Si el manifiesto pasa a registrar estado del grafo, ahí es donde
  debería mudarse.
