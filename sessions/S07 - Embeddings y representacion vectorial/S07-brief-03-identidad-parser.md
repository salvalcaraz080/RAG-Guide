# S07 · Brief 3 — Identidad del parser: headings estructurales + título de documento

> **Para:** Claude Code. **Produce:** el enrutado de los 4 estilos estructurales que hoy no
> llegan a `Section`/requisito, y la recuperación de `Document:Title` como identidad de
> documento. **NO produce:** cambios en el chunker, versionado de derivados, embeddings.
>
> **Contrato de trabajo:** CC no decide arquitectura; marca `# TODO: pendiente de decisión en
> Claude Chat` y sigue. Tests mockeados, sin red, sin Docker, sin `data/raw/`. La validación de
> identidad de cita contra el render es UAT del usuario. Commits los hace el usuario.

---

## 0. Contexto — por qué existe, y por qué se separó del Brief 2

El mecanismo de visibilidad del Brief 2 (categoría "sin clasificar") cazó en su primera corrida
un defecto que estaba en el corpus desde S06: cuatro estilos **estructurales** que el parser
nunca enrutó — `Annex4` (8.615 ch), `Annex5` (1.096 ch), `require:level4` (165 ch), `require`
(10 ch), en 5 documentos. **No es hueco de contenido, es de identidad:** un heading que no se
enruta no llega a ser `Section`, así que las cláusulas que cuelgan de él se anclan a su ancestro
y **la cita sale más gruesa que el documento**; las 14 continuaciones `require:level4` son texto
que le falta a su requisito.

CC hizo bien en **no** arreglarlo dentro del Brief 2: enrutar `Annex4/5` crea secciones y
**re-ancla requisitos**, que es tocar la identidad de cita que S06 fijó y validó contra el
render. Hacerlo de refilón en un brief de prosa habría sido el cambio silencioso que ese brief
combatía. Va como brief propio, con su propia validación, y **antes de los embeddings** — no se
embebe un ancla gruesa o un requisito incompleto para re-embeber después.

Este brief tiene dos tareas de **riesgo muy distinto**. Se implementan y validan por separado.

---

## Tarea A — `Document:Title` → identidad de documento (bajo riesgo, aditivo)

`Document:Title` (4.483 ch) está hoy en el front-matter excluido como `no_containing_section`.
Es la materia del estándar que el header del chunker del Brief 1 pedía y no tenía (caía al
doc_id canónico). Es identidad de **documento**, no prosa de sección.

- Extraer el párrafo `Document:Title` a un campo nuevo a nivel documento:
  `ParsedDocument.document_title: str | None`. Se toma antes/fuera de la lógica de cuerpo de
  sección (no tiene sección contenedora, por eso el Brief 2 lo excluía).
- **Reportar el valor real hallado en los 26 documentos.** El header quería la *materia*
  (p. ej. "Space engineering — Software"). Antes de confiar en él, CC lista qué string trae
  cada documento, para que Chat verifique que es usable como materia. Si la materia está
  repartida entre `Document:Title` y `Subtitle`, CC lo señala — no lo decide.
- **Cobertura:** qué documentos lo tienen y cuáles no. Donde falte, el consumidor cae al doc_id
  canónico (comportamiento actual del Brief 1). El hueco se declara, no se rellena a la fuerza.

Esta tarea no toca anclaje de nada: es un campo nuevo, additivo. Sus tests son triviales
(presencia/ausencia, valor correcto).

---

## Tarea B — Enrutar los estilos estructurales (alto riesgo: re-ancla cita)

### B.1 `Annex4` / `Annex5` → headings

`_HEADING_STYLES` llega hoy a `Annex3`. Extenderlo a `Annex4` y `Annex5`, de modo que se
conviertan en `Section` a profundidad 4/5. Los ejemplos son inequívocos ("Telecommand
acceptance", "Execution reporting", "Deleting telecommands over a time period"): son títulos de
cláusula.

**El punto crítico — no es solo añadir a una lista:** al volverse `Section`, participan en la
resolución de numeración de S06 (`annex_qualifier`, `number_prefix`, `number` desde
`numbering.xml`). Un heading nuevo cuya numeración se resuelva **mal** da a las cláusulas que
cuelgan de él un ancla *precisa pero incorrecta* — peor que la actual *gruesa pero correcta*.
La numeración de estos headings debe resolverse con el mismo mecanismo que el resto y
**validarse contra el render** (§Validación).

Además: los requisitos que hoy cuelgan del ancestro de profundidad 3 pasarán a colgar del nuevo
heading de profundidad 4/5. Eso **cambia su `section_number`**. Es el arreglo — el punto es que
la cita se vuelva precisa — pero es un cambio de identidad que hay que hacer **visible** (§B.3).

### B.2 `require:level4` + `require` desnudo → texto del requisito

`_REQUIRE_CONTINUATION_STYLES` llega a `require:level3`. Añadir `require:level4`: sus 14
continuaciones son texto que hoy le falta a su requisito y debe unirse a él (con la disciplina
`accepted_text`, como el resto). Esto **cambia el texto citable** de esos requisitos — los
completa.

`require` desnudo (10 ch, 1 doc, `ECSS-E-ST-10C`) es minúsculo: CC lo inspecciona y lo enruta a
donde corresponda (cuerpo de su requisito, con toda probabilidad), y lo **reporta**. Si su
naturaleza no es inequívoca, `# TODO` en vez de adivinar — es identidad de requisito.

### B.3 El re-anclaje se reporta como delta, no se desliza

Principio del Brief 2, aplicado a la identidad: un requisito que cambia de ancla es un **delta
declarado y contado**, no un desplazamiento silencioso. Reportar, por documento:

- secciones nuevas creadas (los `Annex4/5`), con su número resuelto.
- requisitos re-anclados: `{req} : {ancla_vieja} → {ancla_nueva}`.
- requisitos cuyo texto cambió por unir `require:level4`.

Este delta es lo que el usuario revisa en UAT contra el render. Sin él, el re-anclaje sería
invisible — justo lo que no queremos en identidad de cita.

---

## Validación contra el render (UAT del usuario) — el control de riesgo central

Es la disciplina de S06 (la identidad de cita se valida contra cómo se lee el documento, no
contra que el parser no lance excepción). Guion de UAT a entregar:

- Para **cada** sección `Annex4/5` nueva: su número resuelto coincide con el que un humano leería
  en el render del documento (no un número inventado ni desplazado).
- Para **cada** requisito re-anclado (del delta B.3): el nuevo ancla es el que le corresponde en
  el render — la cláusula bajo la que el requisito realmente vive.
- Para los requisitos con `require:level4` unido: el texto completo coincide literalmente con el
  render, sin duplicar ni partir.

Como el volumen es pequeño (9.886 ch, 5 documentos, 14 continuaciones, un puñado de anexos), la
validación puede ser **exhaustiva**, no muestreada. Que sea barata no la hace opcional: es
identidad de cita.

Si cualquier caso falla contra el render, CC **no lo fuerza**: lo deja marcado y lo sube a Chat.
Una cita precisa-pero-incorrecta es peor que una gruesa-pero-correcta.

---

## Interacción con el versionado y los derivados

Como el Brief 2, este brief **cambia el output del parser sin cambiar el conjunto de documentos**
→ `corpus_version` no se mueve, pero los chunks derivados quedan invalidados. El borrado ruidoso
de `chunks.json` en `parse_all` (ya implementado en Brief 2) cubre la invalidación por ahora. El
fingerprint de contenido (decisión B de Chat) se cablea en el brief de consumición, no aquí.
`references.json`/grafo siguen vigentes salvo que el re-anclaje cambie el `section_number` de
requisitos de los que se derivan referencias — **CC verifica** si algún requisito re-anclado es
origen de referencias y, si lo es, lo reporta (puede requerir regenerar `references.json`, que
NO se hace aquí; solo se señala).

---

## Tests (mockeados, sin red, sin DOCX reales)

1. **Document:Title:** un doc mock con `Document:Title` puebla `document_title`; uno sin él lo
   deja `None`.
2. **Annex4/5 → Section:** un heading `Annex4` mock produce una `Section` a profundidad 4, con
   numeración resuelta, y los requisitos bajo él se anclan a ella, no a su ancestro de nivel 3.
3. **Descendencia por componentes se mantiene:** una sección `B.2.1.1` nueva es descendiente de
   `B.2.1` bajo el `is_descendant` por componentes del Brief 1 (no regresión de prefix-address).
4. **require:level4 unido:** una continuación `require:level4` mock se une al requisito abierto,
   con accepted_text; no cae a `body`.
5. **Delta reportado:** un requisito re-anclado aparece en el reporte con ancla vieja → nueva.
6. **Sin regresión:** documentos sin `Annex4/5`/`require:level4` producen exactamente la misma
   salida que antes (el brief añade ramas, no altera las existentes).

Fixtures: un anexo con headings hasta profundidad 5, un requisito con continuación de 4º nivel,
un doc con `Document:Title`, un doc sin ninguno de los dos (invariancia).

---

## Al cerrar (CC)

- `S07-decisions-3-identidad.md`: log crudo, etiquetado agnóstico/dominio. Sube al chat.
- Deriva `CHANGELOG.md` y **reescribe el estado presente** en `CLAUDE.md`/`ARCHITECTURE.md`:
  `_HEADING_STYLES` hasta `Annex5`, `_REQUIRE_CONTINUATION_STYLES` hasta `require:level4`,
  `ParsedDocument.document_title`, y el delta de re-anclaje. Sin diario.
- Guion de UAT (validación contra render, exhaustiva).
- Reporte de valores de `Document:Title` por documento (input para que Chat verifique la materia).
- Cualquier `# TODO` (esperable en `require` desnudo si su naturaleza no es inequívoca).

---

## Fuera de scope (no implementar)

- **Consumición en chunking** (glosario, autoridad NOTE, `SectionChunker` con prosa) — Brief 4.
- **Versionado por fingerprint de contenido (decisión B)** — se cablea en Brief 4, donde los
  chunks se regeneran.
- **Regeneración de `references.json`/grafo** — solo se *señala* si el re-anclaje la afecta; no
  se ejecuta aquí (sigue gateada por validación del extractor).
- **`Subtitle`, `Published` y demás front-matter** — fuera; solo `Document:Title`, salvo que el
  reporte muestre que la materia lo exige (entonces se decide en Chat).
- **Embeddings, wrapper, vector store** — posteriores.
