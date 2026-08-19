# S06 — Decisions log, brief 3-pre: diagnóstico de numeración de sección

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier pipeline que
> lea documentos ofimáticos) / `[dominio]` (específico de RAG-ECSS y del corpus ECSS).
>
> Diagnóstico, no producción: nada de esto numera cláusulas de verdad. Resuelve la lista de
> Word **para comparar** contra el contador del parser. La versión de producción es Brief 5.

---

## 1. Resolver la lista de Word es reimplementar un motor de numeración `[agnóstico]`

El brief pide comparar «el número que el parser cuenta» contra «el número que se resuelve desde
el XML». La segunda mitad suena a lectura y no lo es: `numbering.xml` no contiene números,
contiene **definiciones de contador**. Para saber que un heading imprime `8.6.2` hay que
recorrer el documento en orden manteniendo el estado de la lista — start, incremento, reinicio
de niveles profundos, formato por nivel — igual que hace Word al renderizar.

Es decir, el diagnóstico contiene un motor de numeración en miniatura (`ListResolver`). Eso
tiene una consecuencia incómoda que conviene tener presente al leer el output: **el informe
compara dos implementaciones, no una implementación contra la verdad**. Si el resolver tuviera
un fallo, aparecería como «divergencia» y se leería como culpa del parser.

Mitigación: el resolver se testea contra XML sintético donde la respuesta correcta se conoce por
construcción (composición de niveles, reinicio, alfabético de anexo, `startOverride`), y las
asunciones de fidelidad quedan escritas en el código, no en la cabeza de nadie. Ver §3.

## 2. Los contadores viven en el `abstractNum`, no en el `numId` `[agnóstico]`

La asunción de fidelidad más importante, y la que más fácilmente falsearía el informe.

En WordprocessingML, un párrafo apunta a un `numId`; cada `numId` referencia un
`abstractNumId`, que es donde están las definiciones de nivel. Varios `numId` pueden compartir
el mismo `abstractNum`. La pregunta es si eso son **una secuencia o varias**: si son varias, dos
`numId` hermanos empiezan cada uno por 1; si son una, el segundo continúa donde lo dejó el
primero.

Se implementó **una secuencia por `abstractNumId`**, que es el comportamiento de Word y lo que
un documento ECSS necesita para que los anexos numeren `A, B, C…` de corrido aunque el DOCX los
haya troceado en varias instancias de lista. Está fijado en `test_num_ids_sharing_an_abstract_continue_one_sequence`.

Si el output real muestra divergencias sistemáticas y alineadas (todo un anexo desplazado una
posición), **ésta es la primera hipótesis a revisar**, antes de culpar al parser.

## 3. El éxito no se mide por lo que resuelve, sino por lo que no inventa `[agnóstico]`

Un resolver incompleto puede fallar de dos formas: no dar número, o dar uno equivocado. La
primera es inocua para este diagnóstico (el heading queda fuera de la comparación); la segunda
contamina el dato que se quiere medir.

Por eso todas las rutas dudosas devuelven `""` en vez de adivinar: formato no numérico
(`bullet`, `none`), `numId` desconocido, nivel sin `lvlText`. Un heading sin número resuelto
**no cuenta como divergencia** — se clasifica como rótulo o se ignora, pero nunca produce una
divergencia falsa. Es la misma disciplina que el proyecto ya aplica a los DRD: no numerar es
mejor que numerar mal.

## 4. Distinguir divergencia «ruidosa» de «silenciosa» por el componente cero `[dominio]`

El hallazgo que el brief persigue no es la divergencia cualquiera, es la que **hoy no deja
traza**. Hacía falta un criterio operativo para separarlas.

Se usa: una divergencia es **ruidosa** si el número contado tiene algún componente `0`
(`E.0.1`), y **silenciosa** en caso contrario (`E.3.1` donde el XML dice `E.2.1`). El criterio
no es arbitrario — el componente cero es exactamente lo que hace fallar `INV4` en
`verify_parser.py`, porque el padre `E.0` no existe entre las secciones. O sea, la clasificación
reproduce «¿lo pillaría el invariante que ya tenemos?».

Las silenciosas se imprimen **primero** en las muestras de cada documento, por delante de las
ruidosas. Un informe que empieza listando los `E.0.1` que ya conocemos entierra el hallazgo
nuevo bajo el viejo.

## 5. Un rótulo es un heading sin numeración de lista **y** sin número en el texto `[dominio]`

La parte 3 cuenta headings que el parser numera y que no deberían llevar número. La definición
ingenua —«no tiene `w:numPr`»— produce falsos positivos: un documento donde el número esté
escrito como caracteres en el texto tampoco tiene `numPr`, y sin embargo su heading sí es una
cláusula citable.

Por eso `is_label` exige las dos condiciones: sin numeración de lista **y** sin número literal
al principio del texto. El detector de literales (`literal_number_in_text`) existe casi solo
para esto, aunque el brief lo pida además como mecanismo de la parte 1.

## 6. La numeración ligada a estilo obliga a resolver la cadena `w:basedOn` `[agnóstico]`

Un heading puede estar numerado sin llevar `w:numPr` propio: lo hereda de su estilo, y el estilo
puede heredarlo a su vez de otro vía `w:basedOn`. Leer solo el párrafo clasificaría esos
headings como **rótulos**, inflando la parte 3 con documentos enteros y haciendo el informe
inútil justo donde más importa.

`resolve_numpr` sigue la cadena con tope de profundidad y detección de ciclo
(`test_based_on_cycle_does_not_hang`). Que esto sea necesario es, en sí, un dato para el
Brief 5: la resolución de producción tendrá que hacer lo mismo.

## 7. La comparación se alinea por posición, y se comprueba `[agnóstico]`

Para comparar contador contra XML hace falta emparejar cada heading del XML con su `Section` del
parser. Ambos recorren `doc.paragraphs` en orden con el mismo filtro (estilo en
`_HEADING_STYLES`, texto no vacío), así que van 1:1 por índice.

Eso es una asunción sobre código ajeno que puede romperse en silencio si el parser cambia su
filtro. En vez de confiar, `analyse_document` **compara las longitudes** y, si no cuadran,
marca el documento con `error` y omite la comparación en vez de emparejar mal y reportar
divergencias inventadas. Un informe que miente es peor que uno incompleto.

Sub-decisión: los párrafos **vacíos** consumen elemento de lista en Word aunque el parser los
salte, así que el resolver los avanza igualmente. Sin esto, cualquier heading tras un párrafo
numerado vacío saldría desplazado.

## 8. Un documento que revienta no puede llevarse el informe `[agnóstico]`

Aplicado desde el principio, no aprendido aquí: `main` envuelve cada documento y registra el
error en su fila. Es la lección de `M-ST-80C` en el Brief 2, donde un `AttributeError` en el
documento 16 de 26 mató el proceso y con él el informe de cobertura.

En un instrumento de medición esto importa el doble: el valor está en el agregado sobre los 26,
y perderlo por un documento roto anula el trabajo entero.

## 9. Qué NO se hizo `[dominio]`

- **No se toca el parser.** El diagnóstico no corrige un solo número.
- **No se decide el mecanismo del Brief 5.** El informe dimensiona la decisión; no la toma.
- **No se resuelven tablas, `Citation`, ni canonicalización de revisión.**
- **No se ejecuta sobre `parse_ready/`**: es UAT del usuario.

## 10. La primera ejecución encontró dos bugs — en el instrumento, no en el parser `[agnóstico]`

El primer output real dio **604 divergencias sobre 5151 cláusulas (11,7%), 600 silenciosas**. La
disciplina que §1 dejaba escrita —«si sale masivo y uniforme, sospecha del resolver»— se aplicó
antes de interpretar nada, y encontró dos defectos propios.

**Bug 1: el prefijo literal del `lvlText` contaba como parte del número.** ECSS numera sus
anexos con `lvlText = "Annex %1"`, así que el resolver rendereaba `Annex B` donde la identidad
de cláusula del proyecto es `B`. Resultado: **cada anexo de cada documento salía como
divergencia silenciosa**. Son 143 de las 604 — el 24% del informe era ruido mío, y ruido
concentrado justo en el tipo de cláusula (anexos DRD) que más importa para la cita.

La corrección no es recortar `"Annex "` por casos: es quedarse **solo con el tramo entre el
primer y el último marcador `%N`** del `lvlText`, descartando literales de delante y de detrás.
Eso vale para cualquier prefijo (`Annex`, `Chapter`, `Part`) y para sufijos (`%1)`), sin lista
de casos especiales. El literal se devuelve aparte (`ResolvedNumber.prefix`) porque es
informativo: que ECSS imprima «Annex B» y el proyecto cite «B» es un dato del Brief 5, no basura.

**Bug 2: `is_label` preguntaba por el `numPr`, no por el número.** La definición original —«el
párrafo no tiene numeración de lista»— dio **0 rótulos en los 26 documentos**, lo que contradecía
frontalmente la premisa verificada a mano del brief (`E-HB-40A` usa `Heading 6` como rótulos).

La causa: esos headings **sí** están enganchados a una lista, heredada del estilo; lo que pasa es
que su nivel tiene `numFmt="none"` u otra forma que no imprime nada. Word no les pinta número,
pero el `numPr` existe. La pregunta correcta no es «¿está en una lista?» sino **«¿imprime Word un
número?»**. Con la definición arreglada, los ~40 headings de `E-HB-40A` que el informe contaba
como «sin resolver» aparecen como lo que son: rótulos a los que el parser fabrica un número.

Lección transversal, y es la misma de `soffice.com`: **un instrumento de medición necesita su
propia validación antes de creerse su output.** Aquí la validación fue barata —cruzar el reparto
de divergencias contra el número de anexos por documento, que ya conocíamos de
`verify_parser`— y cambió por completo la lectura: de «604 divergencias por todas partes» a «461
reales, 455 de ellas concentradas en 5 handbooks».

## 11. La validación a mano cerró la tensión principal `[agnóstico]`

§1 dejaba abierto que el informe compara dos implementaciones y no una contra la verdad. Se
cerró de la forma barata: **cuatro comprobaciones manuales contra el documento renderizado**,
una por cada población de divergencia (desplazamiento global, patrón handbook, divergencia
silenciosa en estándar, `INV4`). Las cuatro dieron la razón al resolver.

Elegir las comprobaciones importó tanto como hacerlas: cada una valida un grupo entero, no un
caso suelto. Cuatro miradas al documento cubren las 461 divergencias. Y el argumento
complementario es igual de fuerte y no costó nada: el resolver **coincide** con el parser en
4.690 cláusulas seguidas, incluida toda la numeración de anexos de los 21 estándares — una
implementación rota no acierta 4.690 veces y falla justo en 461 formando patrones coherentes.

Resultados en `S06-diagnostico-3-numeracion.md`.

## Tensiones abiertas

- ~~El resolver es una segunda implementación, no un oráculo~~ — **cerrada** (§11): validado
  4/4 contra el render, más 4.690 coincidencias.
- **`lvlOverride` con redefinición de nivel se detecta pero no se aplica.** El informe marca los
  documentos que lo usan; el resolver ignora la redefinición. Si algún documento la tiene, sus
  divergencias no son de fiar y hay que leerlas con esa advertencia. Se prefirió detectar y
  avisar antes que implementar un caso que puede no existir en el corpus.
- **La comparación no cubre los headings que el XML no numera** (§3). Si el parser fabrica un
  número para un heading que Word tampoco numera, eso sale en la parte 3 como rótulo, no como
  divergencia. Son dos hallazgos distintos y conviene no sumarlos.
