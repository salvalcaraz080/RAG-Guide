# S07-decisions-3-identidad.md — Implementación (Claude Code)

Decisiones y fricciones al implementar `S07-brief-03-identidad-parser.md`.
Etiquetadas **[agnóstico]** (candidata a Guide) vs **[dominio]** (solo ECSS).

---

## Resultado: el re-anclaje temido no se produjo

```
26 documentos · 6 con cambios · 0 que toquen identidad de cita
  +205 secciones nuevas (Annex4/Annex5)
     0 secciones eliminadas
     0 secciones renumeradas
     0 requisitos re-anclados
     2 requisitos con texto completado (require:level4)

secciones  5.151 → 5.356      requisitos  7.922 → 7.922 (sin cambio)
```

El cambio es **puramente aditivo**. Ninguna cláusula existente cambió de número y ningún
requisito cambió de ancla: los 205 headings nuevos viven en anexos que no contienen requisitos
(tablas de servicios PUS en `E-ST-70-31C`/`E-ST-70-41C`, y rúbricas de handbook en
`Q-HB-80-03A`/`Q-HB-80-04A`).

**Validación de numeración, además del conteo:** las 205 secciones nuevas tienen padre
existente, cero números duplicados, y los 20 padres con hijos de nivel 4 en `E-ST-70-31C` los
numeran en secuencia `1..n`. La jerarquía resuelta se lee bien: `A.2 Telecommand verification
service` → `A.2.3 Service requests and reports` → `A.2.3.1 Telecommand acceptance`.

**El grafo no necesita regenerarse.** Verificado como pedía el brief: 0 requisitos re-anclados,
y los 2 con texto completado no introducen referencias nuevas (`E-ST-40-07C` 0→0,
`E-ST-70-01C` 12→12). `references.json` sigue vigente.

---

## Tarea A — el brief acertó al pedir el valor real antes de confiar en él

`Document:Title` está ahora en `ParsedDocument.document_title`, presente en **26/26**. Pero el
valor **no es la materia** que el header del chunker necesita:

| documento | `Document:Title` | `Subtitle` |
|---|---|---|
| `ECSS-E-ST-40C` | Space engineering | **Software** |
| `ECSS-Q-ST-80C` | Space product assurance | **Software product assurance** |
| `ECSS-E-ST-10-02C` | Space engineering | **Verification** |
| `ECSS-M-ST-80C` | Space project management | **Risk management** |

Solo hay **tres valores distintos** en los 26 documentos: `Space engineering`, `Space product
assurance`, `Space project management`. Es la **rama** ECSS, que ya está codificada en la letra
del doc_id (`E`, `Q`, `M`) — es decir, no aporta señal nueva al vector.

La materia real (`Software`, `Verification`, `Safety`, `Risk management`) está en `Subtitle`, y
el ejemplo del propio brief 1 (`[Standard: ECSS-E-ST-40C — Software | ...]`) usaba exactamente
eso.

- **[dominio] No lo he decidido, como manda el brief** («si la materia está repartida entre
  `Document:Title` y `Subtitle`, CC lo señala — no lo decide»). Lo señalo: **está repartida, y
  la mitad útil es la que el brief dejaba fuera de scope.** Capturar `Subtitle` es el mismo
  patrón, ~5 líneas. La decisión es de Chat.

---

## La Tarea A estaba bloqueada por un defecto de fidelidad que nadie había visto

Al listar los valores, 24 de 26 `Document:Title` salían así:

```
'Space engineeringECSS SecretariatESA-ESTECRequirements & Standards DivisionNoordwijk,
 The NetherlandsECSS SecretariatESA-ESTECRequirements & Standards DivisionNoordwijk,
 The Netherlands'
```

**Causa:** `accepted_text` recorría el XML del párrafo entero, incluidos los `w:txbxContent` —
el contenido de los **cuadros de texto flotantes** anclados a él. Un cuadro está anclado a un
párrafo pero no es su texto.

**Alcance medido: 40 párrafos en los 26 documentos**, y no solo portadas: 24 `Document:Title`,
2 `Subtitle`, y **14 de prosa de cuerpo** (`paragraph` ×10, `Bul1`, `NOTE 1`, `Normal`,
`graphic`) donde las etiquetas de una figura quedaban pegadas dentro de un párrafo ajeno —
p. ej. `"RetrospectiveSprint ReviewpreparationSprint ReviewBacklogrefinement..."`.

- **[agnóstico] Arreglado**: `accepted_text` excluye `w:txbxContent`. Es un defecto de
  fidelidad que corrompía texto, no una decisión de alcance: el brief 2 declaró que nada
  desaparece sin traza, y esto era lo contrario — texto **apareciendo** donde no estaba.
- **Nota honesta sobre el brief 2:** su verificación de fidelidad («200/200 párrafos literales»)
  no lo detectó porque el muestreo aleatorio no tocó ninguno de los 14. Con 14 casos sobre
  ~15.000 párrafos, un muestreo de 200 tenía ~17% de probabilidad de cazar uno. La lección no
  es que el muestreo fuera pequeño, es que **un muestreo aleatorio no encuentra defectos raros
  y localizados**; lo encontró mirar un campo concreto documento a documento.
- **Efecto colateral:** el contenido de los cuadros de texto (etiquetas de diagramas) ya no se
  captura en ningún sitio. Antes tampoco se capturaba de forma útil — estaba pegado a prosa
  ajena, indistinguible de ella. Si esas etiquetas son contenido recuperable, es decisión de
  Chat y necesita su propio camino, no el de colarse en un párrafo vecino.

---

## Decisiones de implementación

- **[dominio] `require` desnudo NO se enruta a requisito, y va con `# TODO`.** El brief pedía
  inspeccionarlo y no adivinar si no era inequívoco. Son 2 párrafos, ambos «None.», ambos
  inmediatamente bajo un heading `Annex3` «Special remarks» de un DRD. Convertirlos en
  requisitos crearía dos requisitos citables cuyo texto completo es «None.» — identidad de
  requisito inventada para contenido que dice que no hay contenido. Se quedan en prosa, donde
  son visibles y no fingen ser normativos.
- **[agnóstico] El delta se mide con una herramienta, no con un script de usar y tirar**
  (`corpus/diagnostics/diff_reparse.py`). Compara `data/processed/` contra un parse fresco y
  reporta secciones añadidas/eliminadas/renumeradas, requisitos re-anclados y textos cambiados,
  con `exit 1` si algo toca identidad. Cualquier cambio futuro del parser tiene ya su red.
- **[agnóstico] Los requisitos se alinean por POSICIÓN, no por (uid, texto).** Mi primera
  medición usó `(uid, texto[:60])` como clave y reportó **901 requisitos re-anclados en
  `E-ST-70-41C` y 12 en `E-ST-10C`** — un documento que no gana ni una sección. Eran todo
  falsos positivos: el 68% del corpus no tiene UID y los textos se repiten, así que la clave
  emparejaba requisitos distintos. La cifra real es **0**.
- **[agnóstico] Solo un título ÚNICO puede testificar una renumeración.** Segundo tropiezo del
  mismo tipo: emparejar secciones por título daba **24 de 26 documentos «tocando identidad»**,
  porque «DRD identification» encabeza cada anexo DRD (`A.1`, `B.1`, … `F.1`). Un título
  repetido no identifica nada. Con títulos únicos: **0**.
  → **[agnóstico] La lección de las dos:** en un corpus con contenido repetido, una clave
  derivada del contenido **inventa correspondencias**, y las inventa en la dirección que más
  asusta — reportando movimiento donde no lo hay. Es el mismo error que el `chunk_id` del
  brief 1 (64% de ids duplicados) visto desde el otro lado. Donde hay orden estructural
  fiable, la posición es mejor clave que el contenido.

---

## Estado tras el brief

```
prosa extraída        2.257.121 chars   (era 2.273.068: -9.711 que ahora son títulos de
                                         sección Annex4/5, y -6.236 de cuadros de texto)
excluida                135.238 chars
sin clasificar              572 chars   (5 estilos, todos ruido de cola larga)
document_title             26/26
```

De los cuatro estilos estructurales que el brief 2 destapó, quedan **10 chars**: los dos
«None.» de `require`. `Annex4`, `Annex5` y `require:level4` están enrutados.

---

## Pendiente de decisión en Claude Chat

1. **`Subtitle` como materia del documento.** `Document:Title` resultó ser la rama (3 valores
   distintos, ya implícita en la letra del doc_id). La materia está en `Subtitle`. Sin ella el
   header del chunker sigue sin lo que el brief 1 pedía.
2. **`require` desnudo** («None.» ×2): ¿requisito o prosa? Hoy prosa, marcado.
3. **Contenido de cuadros de texto** (etiquetas de diagramas): ya no se captura. ¿Es contenido
   recuperable, y por qué camino?
4. **Marcadores literales de lista** (`1.`, `(a)`, `(1)`) en el texto de requisito. La
   sangría ya conserva el anidamiento, que era la mitad con significado; reproducir el
   marcador impreso exige un campo de sufijo en `ResolvedNumber` y tocar la numeración
   validada contra el render en S06. Afecta a 1.638 de 7.922 requisitos (21%).
5. **Versionado de derivados** — sigue abierto del brief 2, y este brief vuelve a cambiar el
   output del parser sin mover `corpus_version`. `chunks.json` quedó invalidado y se regenera
   en el brief 4.

---

## UAT — validación contra el render: PASADA, con un defecto encontrado

El usuario validó de forma exhaustiva las 205 secciones nuevas y los 2 requisitos completados
(checklist en `S07-uat-3-identidad.md`).

**Resultado:**

- ✅ **Los números de las 205 secciones nuevas coinciden exactamente con los que imprime el
  documento.** El enrutado de `Annex4`/`Annex5` queda validado contra el render, que era el
  control de riesgo central del brief.
- ✅ **El contenido de los 2 requisitos completados es el correcto.** `require:level4` une lo
  que faltaba.
- ❌ **Defecto encontrado, ajeno al brief pero destapado por su UAT:** las listas de requisito
  perdían la jerarquía al unirse.

### El defecto: aplanar una lista jerárquica cambia lo que afirma

`ECSS-E-ST-70-01C` cláusula 5.4.2.2, tal como lo extraía el parser:

```
The following OBCP execution states shall be supported:
in store i.e. the OBCP is in the executable form, stored either on-board or on-ground.
in engine i.e. the OBCP is in an on-board memory accessible by the OBCP engine:
inactive i.e. the OBCP is idle;
active i.e. the OBCP is under control of the OBCP engine:
running i.e. the OBCP is executing;
held i.e. the OBCP execution has been placed on hold.
```

Y como se lee en el documento:

```
The following OBCP execution states shall be supported:
1. in store i.e. the OBCP is in the executable form, stored either on-board or on-ground.
2. in engine i.e. the OBCP is in an on-board memory accessible by the OBCP engine:
    (a) inactive i.e. the OBCP is idle;
    (b) active i.e. the OBCP is under control of the OBCP engine:
        (1) running i.e. the OBCP is executing;
        (2) held i.e. the OBCP execution has been placed on hold.
```

- **[dominio] No es formato perdido, es otra afirmación.** El texto plano sostiene que hay
  **seis estados hermanos**; el documento dice que hay **dos de primer nivel**, uno con dos
  sub-estados, uno de los cuales tiene dos sub-sub-estados. A la pregunta «¿qué estados de
  ejecución debe soportar un OBCP?», el texto aplanado fundamenta una respuesta incorrecta.
  Afecta a **1.638 de 7.922 requisitos (21%)**.

### Arreglado a medias, a propósito

- **Hecho:** las continuaciones se unen **con sangría por nivel**.
  `_REQUIRE_CONTINUATION_STYLES` pasa de conjunto a mapa `estilo → profundidad`; el nivel ya
  estaba en el nombre del estilo y solo se estaba descartando. Cambio barato y sin riesgo: no
  toca numeración ni identidad. `diff_reparse` confirma 0 secciones y 0 re-anclajes.
- **Deuda declarada:** no se reproducen los marcadores literales (`1.`, `(a)`, `(1)`). El
  resolver **ya los calcula** —`num='a'`, `prefix='('`— pero `ResolvedNumber` no guarda sufijo
  de cierre y hace `.strip(".")` a propósito, para que un heading salga `5.2.3` y no `5.2.3.`.
  Reproducirlos exige tocar `corpus/numbering.py`, que es el módulo que S06 validó contra el
  render. Ahí está el riesgo, y por eso se separó.
- **[agnóstico] Por qué se hizo la mitad barata ahora y no todo después:** el texto del
  requisito es literalmente lo que se embebe y lo que se cita. Cambiarlo tras generar los
  embeddings obliga a re-embeber el corpus entero; la sangría cuesta cinco líneas.

### La lección de método, que es lo que vale para la Guide

- **[agnóstico] Dos defectos de esta sesión sobrevivieron a la verificación automática y
  cayeron mirando el documento.** Los cuadros de texto (texto *apareciendo* donde no estaba) y
  este (jerarquía *perdida* sin que faltara ni un carácter). Mi comprobación de fidelidad del
  brief 2 —«200/200 párrafos literales»— dio verde en ambos casos, y con razón: el contenido
  estaba y era literal. Lo que no estaba era la **relación entre las partes**, y eso no lo mide
  una comparación de cadenas.
- La regla que se deriva: **verificar contra el render no es verificar contra el texto**. Un
  parser puede tener todos sus invariantes en verde y aun así afirmar algo distinto de lo que
  el documento dice, porque el documento significa con estructura visual (numeración, sangría,
  anidamiento) que el texto plano no lleva. Es el mismo motivo por el que S06 validó la
  numeración contra el render y no contra el parser.