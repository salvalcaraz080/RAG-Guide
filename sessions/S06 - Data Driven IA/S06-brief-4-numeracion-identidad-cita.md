# S06 — Brief 4: Numeración e identidad de cita fiel al documento

> **Tipo:** reparación estructural del parser + extensión del contrato de cita. Cruza la frontera
> `corpus/` ↔ `app/` (el identificador nace en el parser, el contrato `Citation` vive en
> `app/schemas/`). Es el brief que construye **un `clause_id` verificable contra el documento** — el
> cimiento de toda cita, del que cuelgan las tablas (Brief 5) y el chunking de S07.
>
> **Por qué ahora, antes que tablas:** una tabla hereda el `clause_id` de su sección; si la
> numeración de la sección está mal, la cita de la tabla nace fantasma. La numeración fiable es
> prerequisito de todo lo que cita. Y es prerequisito del chunking de handbooks en S07 (ver §0).
>
> **Frontera CC:** CC escribe el código de producción + tests mockeados (XML sintético, sin `data/`,
> sin red). **Ejecutar sobre los 26 DOCX reales y verificar las citas resultantes es UAT del
> usuario.** El resolver de numeración ya existe como prototipo validado en
> `corpus/diagnose_numbering.py` — este brief lo **promueve a producción**, no lo reinventa.

---

## §0 — Contexto: por qué la identidad de cita es más que un número

El diagnóstico (`S06-diagnostico-3-numeracion.md`) estableció que el parser **fabrica identidad de
cita incorrecta en 502 cláusulas**, 457 de ellas silenciosas (número plausible pero equivocado, que
ningún invariante detecta). El daño está **latente**: 455 divergencias están en handbooks con 0
requisitos, y con el modelo actual (cita cuelga del requisito) no pueden producir cita fantasma hoy
— pero se activan en cuanto el chunking use la sección como unidad, que es la única forma de que los
handbooks aporten al corpus (S07). Este brief repara la numeración **antes** de que esa activación
ocurra.

**Modelo de autoridad de cita (decidido en Claude Chat, gobierna el diseño del identificador):** el
corpus citable tiene **dos grados de autoridad**, y la unidad citable debe llevar cuál es:

- **Requisito** (viene de un estándar): la cita es **identidad** de la sustancia. Fidelidad literal
  (se transcribe, no se parafrasea). Puede ser toda la respuesta.
- **Recomendación** (viene de un handbook): la cita es **procedencia** de una orientación. Síntesis
  permitida (se resume). **También puede ser toda la respuesta** — un handbook es fuente autónoma, no
  siempre subordinada a un requisito.

Ambos son citables de primera clase. Consecuencia para *este* brief: la coordenada de cita de un
handbook (`handbook X, sección Y`) tiene que ser **tan exacta como la de un requisito**, porque
sostiene una respuesta por sí sola — la síntesis relaja la fidelidad del *contenido*, nunca la de la
*coordenada*. Por eso «se resuelven todas las secciones», no solo las que llevan requisito.

*(El tratamiento diferenciado en generación —transcribir requisitos, sintetizar recomendaciones— es
S11. Aquí solo se prepara el terreno: que la identidad lleve el grado de autoridad consigo.)*

---

## Paso de medición (primero, barato) — dimensionar el daño realizado hoy

Antes de reparar, cerrar el único número que el diagnóstico dejó pendiente (§6 del diagnóstico):
**cuántos requisitos cuelgan exactamente de las 6 secciones divergentes de los estándares**
(`L.4`/`L.5` de `M-ST-40C Rev.1`, `E.1.x`/`F.1.x` de `Q-ST-30-02C`). Esas 6 son la **munición real
hoy** — las únicas divergencias en documentos con requisitos, capaces de producir cita fantasma en
el modelo actual. Reportar el número. Es medición, no reparación; informa la urgencia y sirve de caso
de prueba UAT (esas cláusulas concretas deben quedar correctas tras la reparación).

---

## Sub-tarea 1 — Resolver la numeración desde `numbering.xml` (el cimiento)

Promover el `ListResolver` del diagnóstico a código de producción del parser. La numeración de
sección deja de **contarse** (fuente del bug) y pasa a **resolverse** desde la numeración automática
de Word — el número que el ingeniero ve al abrir el DOCX, fiel por construcción.

**El mecanismo es UNO SOLO para todo el corpus** — el diagnóstico lo probó (validado en los 26, dos
familias, con y sin `startOverride`; la clasificación «18 mixtos / 8 uniformes» era espejismo del
render). No hay fallback por familias. Pero hereda los **cinco detalles no-opcionales** que el
diagnóstico descubrió, cada uno con su test de por qué:

- Seguir la cadena `w:basedOn` para numeración ligada a estilo (sin esto, documentos enteros parecen
  no numerados).
- Quedarse solo con el tramo entre el primer y último marcador `%N` del `lvlText` (sin esto, todo
  anexo ECSS sale `Annex B` en vez de `B`). El literal (`Annex`, `Part`) se conserva **aparte** como
  dato informativo, no se tira — puede importar en la presentación de S11.
- Contadores por `abstractNumId`, no por `numId` (varios `numId` que comparten abstract son **una**
  secuencia — es lo que hace que los anexos numeren `A, B, C…` de corrido aunque el DOCX los trocee).
- Avanzar la lista también en párrafos vacíos (consumen elemento aunque el parser los salte).
- `startOverride` (lo usan 24 de 26 documentos).

**Cabo suelto a cerrar aquí (el diagnóstico lo detectó pero no lo implementó):** `lvlOverride` con
redefinición de nivel, presente en `E-HB-40-01A`. El diagnóstico lo detecta y avisa; producción debe
**aplicarlo**. Es el único documento con esa característica, así que es un caso acotado — pero sin
cubrirlo, las 163 divergencias de `E-HB-40-01A` no son de fiar.

**Disciplina heredada del diagnóstico (no negociable):** toda ruta dudosa devuelve «sin número», NO
un número adivinado. Formato no numérico, `numId` desconocido, nivel sin `lvlText` → sin número, que
se trata como rótulo (sub-tarea 2), nunca como cláusula con identidad inventada. *No numerar es mejor
que numerar mal* — la misma regla que el proyecto ya aplica a los DRD.

**Validación por construcción, no por parche:** el resolver produce el número que Word imprime. La
forma de verificarlo es la del diagnóstico — comparar contra el render en una muestra por población.
El UAT lo cubre; las 6 cláusulas del paso de medición y las 4 verificadas a mano en el diagnóstico
(`E-HB-40-01A Scope`=1, `Q-HB-80-01A`=3.5, `M-ST-40C`=L.5, `Q-ST-30-02C`=E.1.1) son casos de
regresión concretos.

## Sub-tarea 2 — Rótulos: heredar identidad, no fabricarla

Los **41 rótulos** (40 en `E-HB-40A`, 1 en `E-HB-40-01A`) — headings de estilo (`Overview`,
`Approach`, `Technical requirements`) que Word **no numera** (su nivel tiene `numFmt="none"` u otra
forma sin número) y que el parser hoy numera fabricando un `clause_id`.

**Tratamiento:** un rótulo **no recibe número de sección propio**. Hereda el `clause_id` de su padre
numerado (el heading numerado inmediatamente superior), y su título viaja como **contexto** dentro de
esa cláusula (patrón DRD ya existente: `subsection_path`). Una cita a contenido bajo «Overview» apunta
a la cláusula real que lo contiene, no a un «Overview» inventado.

**Definición correcta de rótulo (el diagnóstico la corrigió — importa):** un rótulo es un heading
**sin número que Word imprima** — NO «un heading sin `w:numPr`». Los rótulos de `E-HB-40A` *sí* tienen
`numPr` (heredado del estilo), pero su nivel no imprime número. La pregunta es «¿imprime Word un
número?», no «¿está en una lista?». Un heading con el número escrito como texto literal (raro en este
corpus, pero posible) tampoco es rótulo — es cláusula citable. `is_label` exige: sin número resuelto
por el `ListResolver` **y** sin número literal en el texto.

## Sub-tarea 3 — Revisión en el identificador de cita (el C-4 original)

Separar los **dos usos del `doc_id`** que hoy están colapsados:

- **`doc_id` topológico** (grafo): quita `Rev.N` para unificar source y target de una arista. Correcto
  para el grafo — se mantiene como está en `graph.py`.
- **`doc_id` de cita** (`Citation`): **conserva la revisión**. `ECSS-E-ST-40C Rev.1`, no
  `ECSS-E-ST-40C` — porque `ECSS-E-ST-40C` a secas es el superseded de 2009 en el inventario, y
  colapsarlos es citar norma derogada con cara de vigente.

**Y cerrar la reversibilidad nombre-fichero ↔ `doc_id` (C-4 de la auditoría, la otra mitad del brief).**
Hoy la identidad se reconstruye desde el nombre de fichero vía `sanitize_filename`, que es **lossy**
(`ECSS-E-ST-40C Rev.1` → `ECSS-E-ST-40C_Rev_1`), y el diagnóstico ya tuvo que normalizar tres
deletreos distintos del mismo documento para reconciliar cobertura (§7 de sus decisions) — un parche
que sólo existe porque no hay clave canónica. La solución: **una identidad de documento canónica que
viaje desde el inventario/manifiesto** (Brief 6) hasta la cita, en vez de re-derivarse frágilmente de
un nombre de fichero. Este brief define esa clave canónica y hace que `Citation` la use; el manifiesto
(Brief 6) es donde se persiste.

*Nota de dependencia con Brief 6:* la clave canónica la **define** este brief (es identidad de cita);
el manifiesto la **registra**. Si el orden obliga, este brief puede establecer la clave y dejar un
`# TODO` en el punto donde el manifiesto la consumirá — pero la forma de la clave se decide aquí,
porque es contrato de cita.

## Sub-tarea 4 — La identidad lleva el grado de autoridad

Propagar hasta la unidad citable el eje de autoridad del §0, para que S07 (chunking) y S11
(generación) puedan ramificar el tratamiento:

- `document_type: standard | handbook` ya existe en el corpus (CLAUDE.md) — asegurar que **viaja con
  la sección/requisito** hasta ser accesible en la unidad citable, no solo a nivel de documento.
- Considerar el eje explícito `authority: normative | informative` en la identidad. El grafo **ya
  tiene** esta distinción como `mention_type` (`normative_ref` vs `informative_ref`) — reutilizar el
  vocabulario, no inventar uno nuevo.

**Alcance estricto:** esto es *preparar el terreno* — que el dato esté presente y accesible. **NO** se
implementa el tratamiento diferenciado (transcribir vs sintetizar, cita-identidad vs cita-procedencia)
— eso es S11. Aquí: que la unidad citable *sepa de qué grado de autoridad es*. Un `# TODO` apuntando a
S11 donde el tratamiento se ramificará.

**Toca `app/schemas/Citation`:** si `Citation` gana un campo de autoridad, es cambio de contrato en
`app/`. Mantenerlo mínimo — `Citation` es contrato de producto, no metadata de parser. La pregunta de
si `Citation` necesita el campo *ya* o basta con que el dato exista en la unidad de recuperación hasta
S11 es de detalle; resolverla del lado de menos cambio de contrato ahora (probablemente: el dato viaja
en la unidad de recuperación, `Citation` se extiende en S11 cuando el tratamiento lo use).

---

## Orden de implementación (dependencia dura)

`medición → sub-tarea 1 (numeración) → sub-tarea 2 (rótulos) → sub-tarea 3 (revisión) → sub-tarea 4
(autoridad)`. Los rótulos son un caso de la numeración; la revisión ensambla sobre secciones ya
numeradas; la autoridad viaja con la identidad ya construida. No alterar el orden.

## Qué entrega CC

1. **El `ListResolver` en producción** dentro de `corpus/parser.py` (o módulo que el parser importe),
   reemplazando el contador de numeración. Con los cinco detalles + `lvlOverride`. La numeración de
   `Section`/`Requirement` pasa a ser la resuelta, no la contada.
2. **Tratamiento de rótulos** — `is_label` con la definición correcta (sin número impreso Y sin
   literal); rótulo hereda `clause_id` del padre, título a `subsection_path`.
3. **`doc_id` de cita con revisión** + clave canónica de documento reversible; `graph.py` mantiene su
   `doc_id` topológico sin revisión (los dos usos separados, documentados).
4. **Autoridad propagada** hasta la unidad citable; `# TODO`→S11 en el punto de tratamiento.
5. **Tests** (XML sintético, sin `data/`): resolución de numeración (composición de niveles, reinicio,
   alfabético de anexo, `startOverride`, `basedOn`, `lvlOverride`, párrafo vacío); rótulo detectado y
   heredando padre; un caso de cada una de las 4 poblaciones de divergencia del diagnóstico como
   regresión; `doc_id` de cita conserva `Rev.N` mientras el topológico no; autoridad presente en la
   unidad citable. `uv run pytest` verde.
6. **Actualización de `verify_parser.py`:** el `INV4` de un rótulo legítimo deja de ser fallo (es
   estado esperado); un `INV4` por numeración mal resuelta sigue siendo fallo. La herramienta distingue
   los dos. La cobertura de identidad correcta (¿cuántas secciones tienen `clause_id` resuelto vs sin
   número?) pasa a ser salida de la verificación.
7. **Guion UAT** — el usuario ejecuta el parser reparado sobre los 26 reales y verifica: las 4
   cláusulas del diagnóstico dan el número que Word imprime; las 6 divergencias de estándares quedan
   correctas; los 41 rótulos no reciben número propio; una cita a un handbook (`E-HB-40A sección X`)
   apunta a la sección que Word muestra; `ECSS-E-ST-40C Rev.1` se cita con revisión, no como
   `ECSS-E-ST-40C`.
8. **`S06-decisions.md`** (append), etiquetado agnóstico/dominio.

## Qué NO hace CC

- No implementa el tratamiento diferenciado requisito/recomendación en generación (S11) — solo
  propaga el dato.
- No parsea tablas (Brief 5).
- No construye el manifiesto (Brief 6) — define la clave canónica de cita, no la persiste.
- No toca el `doc_id` topológico del grafo (se mantiene sin revisión, a propósito).
- No regenera el grafo (Brief 7 / validación aparte).
- No ejecuta sobre `data/` real (UAT del usuario).

## Criterio de "hecho"

- La numeración de sección se resuelve desde `numbering.xml`; las 461 divergencias del diagnóstico
  quedan resueltas (verificable: re-ejecutar el diagnóstico contra el parser reparado debe dar ~0
  divergencias, salvo `lvlOverride` si quedara algún caso — declarado).
- Los 41 rótulos no reciben `clause_id` propio; heredan el del padre.
- La cita conserva la revisión; el grafo mantiene su identidad topológica sin ella; los dos usos están
  separados y documentados.
- La unidad citable lleva su grado de autoridad (standard/handbook, normative/informative).
- `verify_parser.py` distingue `INV4`-rótulo-legítimo de `INV4`-numeración-rota, y reporta cobertura de
  identidad.
- Tests verdes sobre XML sintético; guion UAT entregado con las verificaciones concretas contra render.
- El número de requisitos bajo las 6 divergencias de estándares está medido (paso de medición).
