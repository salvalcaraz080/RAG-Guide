# S07-decisions-2-prosa.md — Implementación (Claude Code)

Decisiones y fricciones al implementar `S07-brief-02-prosa-parser.md`.
Etiquetadas **[agnóstico]** (candidata a Guide) vs **[dominio]** (solo ECSS).

---

## Resultado sobre el corpus real

```
prosa extraída        2.273.068 chars   (59 estilos, 3.064 de 5.151 secciones con cuerpo)
excluida                140.057 chars   (con motivo declarado)
sin clasificar           10.448 chars   (8 estilos, extraídos igualmente y nombrados)
                      ─────────────────
total contabilizado   2.413.125 chars

EXPECTED OUTPUT          33.419 chars atados a 457 requisitos
requisitos                  7.922        (idéntico a antes: este brief añade, no altera)
```

Los `2.413.125` cuadran con los `2.403.239` del inventario de la Tarea 0 más los `9.886` de
estilos estructurales que aquel inventario excluía por empezar por `require`/`Annex`. La
diferencia está explicada al carácter, no aproximada.

**Fidelidad verificada contra el DOCX:** 147/147 párrafos de la cláusula 3.2 de
`ECSS-E-ST-40C` y 200/200 de un muestreo global aleatorio aparecen **literalmente** en el
documento fuente. Sin corrupción por control de cambios, sin párrafos partidos.

**Glosario:** la cláusula 3.2 sale con sus tres estilos contiguos y en orden
(`Definition1` → `paragraph` → `NOTE`), que era la precondición del agrupado del brief 3.

---

## El mecanismo de visibilidad cazó un defecto en su primera corrida

`Annex4`, `Annex5`, `require:level4` y `require` aparecieron como **sin clasificar**: 9.886
chars en 5 documentos. No son prosa desconocida — son **estilos estructurales que el parser
nunca ha reconocido**:

| estilo | chars | qué es | documentos |
|---|---:|---|---|
| `Annex4` | 8.615 | heading de anexo a profundidad 4 | `E-ST-70-31C`, `E-ST-70-41C`, `Q-HB-80-03A`, `Q-HB-80-04A` |
| `Annex5` | 1.096 | heading de anexo a profundidad 5 | `E-ST-70-31C` |
| `require:level4` | 165 | continuación de requisito, 4º nivel | `E-ST-40-07C`, `E-ST-70-01C` |
| `require` | 10 | cuerpo de requisito sin nivel | `E-ST-10C` |

`_HEADING_STYLES` llega hasta `Annex3` y `_REQUIRE_CONTINUATION_STYLES` hasta `require:level3`.
Los ejemplos son inequívocos: «Telecommand acceptance», «Execution reporting», «Deleting
telecommands over a time period» son títulos de cláusula, no prosa.

- **[dominio] Esto NO es un hueco de contenido, es uno de IDENTIDAD.** Un heading que no se
  enruta no llega a ser `Section`, así que las cláusulas que cuelgan de él se anclan a su
  ancestro y su cita queda más gruesa que el documento. Y las 14 continuaciones
  `require:level4` son texto que le falta a su requisito.
- **[dominio] No se arregla en este brief y es deliberado.** Añadir `Annex4/5` a los headings
  crea secciones nuevas y re-ancla requisitos: toca la identidad de cita, que es justo lo que
  S06 fijó y validó contra el render del documento. Hacerlo de refilón dentro de un brief de
  prosa sería exactamente el tipo de cambio silencioso que este brief combate. Va al reporte
  con su propio epígrafe ⚠️, separado de los estilos de prosa desconocidos, porque **la acción
  que pide es otra**.
- **[agnóstico]** Es la validación del diseño: la categoría «sin clasificar» no es burocracia,
  es un detector. Encontró en su primera ejecución un defecto que llevaba en el corpus desde
  S06 y que ninguna suite habría señalado, porque todo lo que el parser declaraba seguir se
  cumplía.

---

## Decisiones de implementación

- **[agnóstico] Clasificación por familia normalizada, no por nombre exacto.** El corpus trae
  65 estilos de párrafo, y las variantes de caso son reales: existen `EXPECTED OUTPUT` **y**
  `Expected Output` (2.018 chars), `table of figures` **y** `Table of Figures` (3.931 chars).
  Un match exacto se habría comido el gemelo en silencio — el mismo modo de fallo que este
  brief corrige, reintroducido en su propia corrección. Se clasifica sobre el nombre
  normalizado (minúsculas, espacios colapsados) por prefijo de familia; `Paragraph.style`
  conserva el original, porque es sobre él que el chunker decide.
- **[agnóstico] Un estilo sin clasificar se extrae igualmente, no se retiene.** El brief dice
  «no se descarta: se reporta». Se leyó como extraer-con-etiqueta: retenerlo sería que el
  parser juzgue qué es contenido, que es exactamente lo que §2.5 le prohíbe. Lo que aporta la
  marca es visibilidad, no filtrado. Con eso, los estilos de prosa realmente sin clasificar
  del corpus son **562 chars en 4 estilos** — ruido.
- **[dominio] Front-matter sin sección contenedora se excluye con motivo**
  (`no_containing_section`, 66.119 chars de `paragraph` más `Document:Title`, `Published`,
  `Subtitle`). Sin heading previo no hay ancla, y no se le inventa una — mismo criterio que S06
  aplicó a una tabla sin cláusula contenedora. Queda declarado y contado, no perdido.
  → Nota para Chat: ahí dentro está `Document:Title` (4.483 chars), que es el título de
  documento que el header del chunker pedía en el brief 1 y no tenía. Recuperarlo es una
  decisión aparte (no es prosa de sección, es identidad del documento).
- **[agnóstico] `EXPECTED OUTPUT` se acumula mientras el requisito está abierto.** El flush
  del requisito lo consume; un `EXPECTED OUTPUT` sin requisito abierto no se pierde, cae a
  `body` con su estilo. 457 requisitos llevan salida esperada.
- **[agnóstico] `parse_all` ya no borra `chunks.json` en silencio.** Un re-parseo invalida los
  chunks —sobrevivirlos sería peor, se embeberían como si fueran del parse actual— pero el
  borrado ahora se nombra en el log. Es el mismo principio del brief aplicado al propio
  pipeline: no es que no se pueda borrar, es que no se puede borrar callando.

---

## Lo que este brief NO tocó (y por qué se nota)

- **El chunker no consume nada de esto todavía.** `SectionChunker` sigue leyendo solo tablas
  aplanadas, así que los 2,27M chars de prosa están en `data/processed/` y **no** en el índice.
  Es el brief 3. `chunks.json` quedó invalidado por el re-parseo y se regenera allí.
- **El grafo no se regeneró y no hace falta:** las referencias se extraen del texto de los
  requisitos, que es idéntico (7.922 requisitos, mismo texto). `references.json` y
  `graph.graphml` siguen vigentes.

---

## Pendiente de decisión en Claude Chat

1. **Versionado de derivados (§4 del brief).** `corpus_version` identifica el conjunto de
   documentos, no la lógica que los procesó. Este brief cambió el output del parser sin
   cambiar el conjunto: unos chunks de antes y otros de después llevan
   `2026-08-04.91bc8116` siendo distintos. Rompe «un derivado se ata a su versión», que es lo
   que sostendrá el determinismo de los embeddings. `# TODO` puesto en `corpus/chunk_all.py`,
   en el punto donde se estampa. **Debe resolverse antes del brief de embeddings.**
2. **Los cuatro estilos estructurales sin enrutar** (`Annex4`, `Annex5`, `require:level4`,
   `require`). Afecta a identidad de cita, no a cobertura de prosa.
3. **`Document:Title`** — recuperarlo daría al header del chunker la materia del estándar que
   el brief 1 pedía y no pudo poner.
4. **Autoridad de la familia `NOTE*` y agrupado del glosario**: ya decidido en Chat, lo aplica
   el chunker en el brief 3. El parser solo preservó estilo y orden, que es lo que hacía falta.
