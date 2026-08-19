# S06 — Decisions log, brief 5: extracción de tablas

> `[agnóstico]` (aplica a cualquier extracción de documentos ofimáticos) / `[dominio]`
> (RAG-ECSS y el corpus ECSS).

---

## 1. La rejilla se resuelve al leer, y eso es lo que hace objetivo el enrutado `[agnóstico]`

El brief pide clasificar las combinadas por «¿la relación es desplegable a texto sin inventar
notación?». Esa pregunta no se puede contestar contando `gridSpan` y `vMerge`: hay que
**intentar el despliegue** y ver si sale una rejilla.

Así que `read_signals` no cuenta merges, los **resuelve**: una celda que abarca N columnas se
repite en las N, y una continuación vertical hereda el valor de arriba. Después, si las filas no
tienen la misma anchura, no hay rejilla — y sin rejilla no hay orden lineal fiel. Eso da la
cuarta clase, `ragged`, que el brief no anticipaba y que es la huella de los diagramas de
estructura donde la disposición *es* el contenido.

El resultado: la prueba de irreducibilidad es **el fracaso de la resolución**, no un juicio sobre
qué parece la tabla. `nested` y `ragged` van a marcar; `grid_span` y `v_merge` se despliegan.

Reparto real sobre `E-ST-70-31C` (264 tablas): 138 `grid_span`, 69 `none`, 57 `v_merge`, 0
irreducibles. En `E-HB-40A`: 11 `none`, 5 `v_merge`.

## 2. Desplegar para alinear y saltar al linealizar `[agnóstico]`

La repetición que hace posible la rejilla es exactamente lo que la arruina al escribir texto. La
primera versión producía:

```
Name: Type. Name: Type. Definition: The type of array…
```

Tres veces lo que el documento dice una vez, porque la celda `Name` abarcaba columnas y el
aplanado emitía un par por columna cubierta. **No es un problema estético: es infidelidad.** El
texto extraído afirmaba una repetición que el original no tiene.

La solución separa los dos propósitos: la rejilla guarda las posiciones repetidas (las necesita
para alinear) y marca cuáles son continuación; el aplanado las salta. El nombre de columna de
una celda de datos bajo una cabecera combinada se busca caminando a la izquierda hasta el inicio
del span.

## 3. Cabecera declarada o nada `[dominio]`

Segunda infidelidad, encontrada en la misma pasada. Con `use_header = has_header_row or rows > 1`,
una tabla de gramática de una columna tomaba su primera línea de datos como cabecera y prefijaba
con ella todo el resto:

```
Absolute Time = "absolute time";: Absolute Time Constant = (Year, "-", Month…
```

El brief ya lo decía —*«la cabecera declarada da los nombres de campo; sin cabecera, usar
posición»*— y la heurística que añadí por «ayudar» inventaba una relación que el documento no
declara. Se eliminó: `use_header = signals.has_header_row`, sin más.

El precio es real y aceptado: tablas que a ojo tienen cabecera pero no la declaran (`5.1 Data
definition`) salen posicionales, `Type | The type of array… | Enumerated`. Menos legible, pero
no afirma nada que el documento no diga. **Fiel gana a útil**, que es la regla de la sesión
entera.

## 4. La columna de relleno se cae dentro de A, no manda la tabla a C `[dominio]`

El brief lista «columnas de relleno tipo `Verified`» bajo el Mecanismo C (descartar). Leído al
pie de la letra, descartaría la tabla entera de un checklist — y el propio diagnóstico dice que
en esos checklists **las preguntas son la guía**, o sea contenido.

Interpretación aplicada: una columna íntegramente vacía es un hueco de formulario y se **cae al
aplanar** (`_kept_columns`), conservando las preguntas. Descartar la tabla habría perdido las dos
cosas. Se marca aquí como desviación consciente de la letra del brief, no como descuido.

## 5. Sin cláusula contenedora no se inventa ancla `[dominio]`

Una tabla anterior a cualquier heading numerado —el change-log de portada, 24 en el corpus— no
tiene identidad que heredar. Se registra como descarte con motivo `no_containing_clause` en vez
de colgarla de la sección siguiente o de dejarla sin coordenada.

Consecuencia medida: `E-ST-70-31C` da 2 descartes de ese tipo donde el diagnóstico contaba 1
huérfana. La diferencia es real y correcta — el parser vacía la pila de secciones al encontrar un
`Heading 0` (front-matter), así que una tabla bajo «Change log» tampoco tiene cláusula, mientras
que el diagnóstico no modelaba ese vaciado.

## 6. La autoridad se hereda de la sección, sin recalcular `[dominio]`

`Table.authority` copia `Section.authority`, que el Brief 4 ya resuelve por anexo. No se mira el
tipo de documento en ningún punto de la extracción: si se mirara, las ~253 tablas informativas
que viven en anexos de estándares saldrían normativas, y la `C.1.1` normativa de `E-HB-40A`
saldría informativa.

El «shall» de los handbooks se conserva **literal** —no se toca el texto— y viaja con
`authority: informative`. Neutralizar la trampa es S11; aquí lo único que se garantiza es que la
etiqueta que lo acompaña es fiel.

## 7. La tabla no es unidad citable, y el modelo lo dice `[agnóstico]`

`Table` lleva las coordenadas de su sección (`section_number`, `section_title`, `section_path`,
`authority`) y **no** tiene identificador propio. Es la forma de dato que hace imposible citar
una tabla como si fuera una cláusula, y la que no prejuzga el chunking: quien decida en S07 puede
meterla en el chunk de su sección o sacarla, pero no puede darle identidad porque no la tiene.

Las tablas descartadas y marcadas **siguen en la lista** con su motivo. Descartar en silencio es
el pecado; que la suma de los cuatro mecanismos iguale el total de tablas es lo que `verify_parser`
comprueba ahora (`264 = 261 + 0 + 2 + 1`).

## 8. De paso: `authority` no llegaba al JSON `[dominio]`

`_doc_to_dict` serializaba `number`, `title`, `level` y `annex_qualifier`, pero no `authority` ni
`number_prefix` — los dos campos que el Brief 4 añadió. El eje de autoridad existía en memoria y
se perdía al guardar, así que ningún consumidor de `data/processed/` lo habría visto nunca.
Arreglado aquí porque este brief depende de que llegue.

## 9. Qué NO se hizo `[dominio]`

- **No se decide la presentación** del contenido marcado ni de los handbooks-«shall» (S11).
- **No se toca el grafo ni `references.json`.** Las referencias van a
  `ParsedDocument.table_references`, medidas y aparte.
- **No se decide el chunking** (S07): la tabla se ancla a la sección, que es la forma que no lo
  prejuzga.
- **No se reescribe ningún texto de celda.**

## Tensiones abiertas

- **`ragged` puede estar marcando de más.** Es la prueba de irreducibilidad, pero una tabla con
  una fila final descuadrada por maquetación se marcaría entera aunque el resto fuera regular. En
  los dos documentos probados salieron 0 marcadas, así que no hay evidencia de que ocurra — pero
  tampoco de que no.
- **El aplanado posicional pierde legibilidad** en tablas con cabecera no declarada (§3). Es
  deliberado, pero si resulta que la mayoría del corpus no declara `tblHeader`, conviene
  reconsiderar con datos: el diagnóstico contó 335 de 1.079 con cabecera declarada, o sea que el
  69% saldrá posicional.
- **Las referencias extraídas no se cruzan con nada todavía.** 6 en `E-ST-70-31C`, 18 en
  `E-HB-40A`. Si explican parte del 49% de referencias del grafo sin cláusula destino es
  medición de S07, no de aquí.
