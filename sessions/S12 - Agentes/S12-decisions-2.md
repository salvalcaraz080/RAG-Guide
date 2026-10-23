# S12 · Decisiones — Brief 2: expansión determinista de referencias cruzadas (+ `bounded`)

> Decisiones tomadas al implementar `S12-brief-2.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Rama `s12-agentes`. **§1–§4 se escribieron antes de lanzar la medición**; §5 en adelante, después.

## 1. Preregistro: lo que no cambia después de medir

### 1.1 Formatos de referencia reconocidos — [DOMINIO]

Se sacaron de un barrido de los chunks inyectados en las 14 consultas de la referencia (117
referencias distintas) y de los formatos del brief:

| Forma | Ejemplo | Lectura |
|---|---|---|
| estándar + `clause`/`requirement`/`section` + número(s) | `ECSS-Q-ST-80 clause 7.2`, `[ECSS-Q-80] clauses 6.2.2.1, 6.2.2.2 and 6.2.3.1` | cada número, en ese estándar |
| estándar + número con punto | `ECSS-Q-ST-80, 7.2.1.3`, `ECSS-Q-ST-80C 6.3.5.2`, `[ECSS-E-ST-40C 5.8.3.5.b` | ídem |
| `clause N in/of` estándar | `Clause 6.2.7 in ECSS-Q-ST-80`, `requirements 6.2.6.5 and 6.2.6.6 of ECSS-Q-ST-80` | ídem |
| tabla de trazabilidad | `ECSS-E-ST-40 \| 5.3.3.2b` | ídem |
| estándar + `Table X-n` / `Table n.m` | `[ECSS-Q-80] Table D-2`, `[ECSS-Q-30] Table 5.3` | la cláusula que contiene esa tabla |
| `Annex X` de un estándar | `Annex N of ECSS-E-ST-40` | la cláusula X y sus subcláusulas |
| interna: `see/as per/…` + número, `clause N`, `Table X-n`, `Annex X` | `see clause 6.2.7.4`, `see Table A-1`, `in Annex A` | en el estándar del propio chunk |
| solo el estándar | `as specified in ECSS-M-ST-80` | **no se expande** (`whole_standard`) |
| tabla sin guion | `Table 63` (la 6-3 de Q-ST-40C, mal aplanada) | **no se expande** (`unrecognized`) |

- **Letra de requisito** (`5.3.3.2b`, `4.6.f`, `5.8.3.5.c`): se trae la cláusula; la letra queda en
  la referencia registrada.
- **Una referencia a la cláusula del propio chunk o a una que lo contiene** («This clause 5
  defines…» dentro de 5.1) es `own_clause`: no es cruzada.
- **Límite conocido:** «requirement 4.6.f for the system FMEA contained in [ECSS-Q-30-02]»: el
  estándar va lejos del número y el parser lo lee como interno. No afecta a ningún objetivo (q08-a3
  pasó a G en la UAT).

### 1.2 Normalización del estándar — [DOMINIO]

`[ECSS-Q-80]` → `ECSS-Q-ST-80`; sin letra de versión → la versión del corpus (`ECSS-Q-ST-80C`); se
quitan los sufijos de revisión. Se resuelve con el mismo detector que el routing
(`detect_standards`). Si ningún documento casa, se prueba por prefijo. **Ambiguo** cuando hay más de
uno (`ECSS-Q-HB-80-02` → Part 1A y Part 2A); **fuera del corpus** cuando no hay ninguno
(`ECSS-Q-ST-30`). Ninguno de los dos se expande.

### 1.3 Tablas: etiqueta → cláusula en la base — [AGNÓSTICO] (decisión de Salva)

**Desviación del brief:** el brief daba por hecho que «Table X-n» se resolvía con la convención de
ids del corpus, y no es así. El número de tabla no está en los chunks (el parser descarta el campo
de la leyenda), y una cláusula puede tener varias tablas. Medido: la regla «la n-ésima cláusula con
tablas» falla en **659 de 1.024** etiquetas.

- **Regla:** ECSS numera las tablas dentro de cada cláusula de primer nivel o anexo, en orden de
  documento (`D-2` = la segunda del anexo D). Se cuentan las tablas de los documentos procesados,
  descartadas incluidas, porque conservan su número en el documento.
- **Validación independiente:** la lista de tablas del índice de cada DOCX («Table A-1 : …») unida a
  la línea de leyenda que sí sobrevive en los chunks («: …»). **43 de 43** etiquetas coinciden; son
  todas las verificables (los 31 de E-ST-40C y los 5 de Q-ST-80C entre ellas). 69 etiquetas no se
  pueden verificar así, porque sus documentos no tienen esa lista.
- **Dónde vive (decidido por Salva):** tabla `table_labels` en la base, mismo patrón que las
  estadísticas BM25. Migración 0005 escrita a mano, etapa de ingesta en la misma transacción, CLI
  para rellenarla sin re-ingestar (`uv run python -m app.services.table_labels`), sello de
  `input_fingerprint`. Si el sello no casa con el de los chunks, ninguna referencia a tabla resuelve
  (`table_unknown`) y se registra un warning. Aplicada en la base local: 1.035 etiquetas de 26
  documentos.

### 1.4 Selección y chunks por cláusula — [AGNÓSTICO]

- Se recorren los chunks en orden: los recuperados por rango y luego los de la expansión de
  cláusula. Se quitan los duplicados, las cláusulas ya inyectadas y las propias.
- Se resuelven contra la base con **`get_clause`** (nueva primitiva del servicio de datos), que trae
  la cláusula **y sus subcláusulas**: «clause 6.2.7» la responden 6.2.7.1 a 6.2.7.11, y 6.2.7 puede no
  tener chunk propio. Una candidata sin ningún chunk en el corpus (`clause_not_found`) no ocupa
  plaza.
- **Dentro de cada cláusula:** normativos primero (como `bounded`) y, dentro de cada autoridad, por
  distancia coseno a la consulta, con empate por orden de chunk. Determinista. Motivo: un anexo o
  una cláusula padre tienen docenas de chunks, y el orden de documento traería siempre los primeros.
- Los añadidos van **al final de todo**, con `origin="expansion"` y sin distancia, igual que los de
  `orphans`/`bounded` (X6). Al ir al final reciben los `[n]` más altos, y el presupuesto de entrada
  descarta las cláusulas de mayor `[n]` primero: **X7 se cumple sin tocar `budget.py`**.

### 1.5 Topes — [AGNÓSTICO]

`RETRIEVAL_XREF_MAX_CLAUSES=5` y **600 / 2.000 tokens** por cláusula y en total, los de partida del
brief, **sin cambiar**. En la prueba sin generación ya se vio que un chunk de tabla (~500 tokens)
deja poco más sitio en la cláusula (q06: D.2 entra solo con su chunk de sección; R.2 con un chunk de
tabla). No es motivo para cambiarlos: subirlos para que lleguen los objetivos sería ajustar al
resultado. Se reporta como causa cuando lo sea (`truncated`).

### 1.6 Definición de «resuelto» — [AGNÓSTICO]

**Desviación del brief, fijada antes de medir:** un aspecto está resuelto en una ejecución si una
cita **`verified`** está en un chunk de su cláusula respuesta **que la referencia no tenía**.

- **Por qué:** en el conjunto `same_clause` la cláusula ya estaba inyectada (el texto sí, la tabla
  no). Con la definición literal («cita verificada a la cláusula respuesta») la propia referencia
  resolvía q03-a1 en 1 de 3 ejecuciones, citando D.1::s0 («Table D-1 describes…»), que no responde a
  las funciones III y IV. Con la nueva regla la referencia da **0 en los 8 aspectos**.
- **Cláusula respuesta de q11-a1:** la del diagnóstico (E-ST-10-06C) **más Q-ST-80C 7.2.1.2**, la que
  encontró la verificación independiente V1. Es la razón por la que q11 está en el conjunto `xref`:
  con la lectura literal no podría resolverse nunca por referencia, porque ningún puntero lleva a
  10-06C.
- **Se reporta también la lectura literal** al lado de cada cifra, para que el chat vea las dos.
- **Precisión en dos cifras:** la estándar, y la misma sin las citas a cláusulas que **solo** trajo
  la expansión (ningún chunk recuperado). El golden set no las juzga, y cuentan como no relevantes.
- **«Por qué no»**, por ejecución no resuelta: `brought_not_cited`, `trimmed_by_budget`, la razón
  de la traza (`truncated`, `over_limit`, `clause_not_found`, `already_injected`, …),
  `not_detected` o `xref_off`.

### 1.7 Herramientas — [AGNÓSTICO]

- Configuraciones del arnés: `control`, `xref`, `bounded` y `xref+bounded`. Difieren solo en
  expansión y referencias; `control` es `defaults` construido explícitamente.
- Cada ejecución guarda en el artefacto la traza de referencias de su contexto (`xref`).
- `compare.py --baseline-config/--new-config` compara configuraciones dentro de un mismo artefacto.
- `corpus/groundtruth/generation/aspects.py`: la métrica de §1.6, re-puntuable sobre artefactos.

## 2. Comprobación previa sin generación

Con `RETRIEVAL_XREF=clause`, solo retrieval (embeddings de consulta), sobre q01, q03, q06, q09 y
q11: el pipeline detecta, resuelve y trae en el contexto las cláusulas objetivo E-ST-40C 5.3.3.2
(q01-a1), R.2::t11 con la fila de verificación independiente (q06-a1), Q-ST-80C 6.2.7.3 (q09-a1),
Q-ST-80C 7.2.1.2 (q11-a1) y Q-ST-80C D.1::t0 (q03-a1). No llegan por los topes: D.2::t4 (q06-a1, la
otra vía), R.2::t12 (q06-a2) y Q-ST-80C 6.2.7.8 (q09-a3). Que el modelo los **cite** es lo que mide
el arnés.

## 3. Tests de contrato

**1.324 → 1.380** (+56: 41 de referencias, 3 de etiquetas de tabla, 9 de aspectos, 3 del arnés).
Modificado 1: `test_data_layer` (la lista de tablas registradas incluye `table_labels`, renombrado
a `test_every_table_is_registered`). Borrados: 0.

## 4. Llamadas reales

Una pasada de 4 configuraciones × 14 consultas × 3 ejecuciones, autorizada (≈ 0,35 USD).
**Gastado: 0,383 USD** en 184 llamadas (168 ejecuciones juzgadas y 16 turnos previos), y 184
embeddings de consulta. No hubo segunda pasada: el instrumento no falló (§5.1).

---

*De aquí en adelante, escrito después de medir.*

## 5. Resultados — artefacto `generation_20261019-224307`

### 5.1 El instrumento — [AGNÓSTICO]

- **168 de 168 ejecuciones** servidas por el primario, sin errores, sin fallback y sin recortes por
  presupuesto (la mediana de entrada máxima, 10.546 tokens en `xref+bounded`, queda lejos de los
  30.000).
- **`control` frente a la referencia** (`comparison_20261019-224409`): recall 0,91 → 0,90, precisión
  0,86 → 0,86, `verified` 1,00 → 1,00. Dentro del ruido (±0,03): **se puede comparar**.
- La salida del arnés es 3 (cláusulas inyectadas sin juicio: 159 en `xref`, 189 en `xref+bounded`);
  los pares quedan exportados en `pairs_generation_20261019-224307.json` para el modo B, si el chat
  lo pide.

### 5.2 Tabla del criterio (§3.2 del brief) — CC no lo aplica

Pareado con `control`, mismo día, misma pasada (`compare.py --baseline-config control`):

| | `control` | `xref` | `bounded` | `xref+bounded` |
|---|---|---|---|---|
| **Aspectos `xref` resueltos ≥ 2/3** (de 6) | 0 | **2** | 0 | **2** |
| Aspectos `same_clause` resueltos ≥ 2/3 (de 2) | 0 | 0 | **0** | 0 |
| Recall (Δ global / pareado) | 0,90 | 0,82 (**−0,08** / −0,08) | 0,91 (+0,01 / +0,01) | 0,86 (−0,04 / −0,04) |
| Precisión estándar | 0,86 | 0,86 | 0,90 | 0,88 |
| Precisión sin cláusulas solo de expansión | 0,86 | 0,87 | 0,90 | 0,90 |
| `verified` | 1,00 | 1,00 | 1,00 [0,99–1,00] | 1,00 |
| Citas `misattributed` / `modal_edit` / `not_found` | 0 / 0 / 0 | 0 / 0 / 0 | 0 / 0 / **1** | 0 / 0 / 0 |
| Latencia mediana / p95 (s) | 12,9 / 19,6 | 13,9 / 22,8 (**+8 %**) | 12,7 / 22,9 | 16,8 / 24,0 (+30 %) |
| Coste por ejecución (USD) | 0,00179 | 0,00198 (**×1,11**) | 0,00200 (×1,12) | 0,00258 (×1,44) |
| Tokens de entrada (mediana) | 6.624 | 8.658 | 8.192 | 10.546 |
| `not_found` por ejecución | 1,43 | 1,40 | 1,43 | 1,40 |
| Estado servido (answered / partial) | 9 / 33 | 11 / 31 | 12 / 30 | 8 / 34 |
| Recall sobre cláusulas relevantes normativas | 0,96 | 0,89 | 0,98 | 0,92 |
| Citas verificadas sobre chunks añadidos | 10 | 83 | 87 | 165 |

**`xref` contra sus cinco guardas:**
1. Éxito ≥ 3 de 6: **2** (q01-a1 y q09-a1, 3/3 cada una). No llega.
2. Recall no baja más de 0,03: **−0,08**. No se cumple (lectura en §5.4).
3. `verified` 1,00 y ninguna cita no literal o mal atribuida nueva: **sí**.
4. Latencia mediana +20 % como mucho: **+8 %**. Sí.
5. Coste ×1,5 como mucho: **×1,11**. Sí.

**`bounded`** (criterio de S11 + `same_clause`): `same_clause` 0 de 2. Recall agregado +0,01 y
precisión +0,04, dentro del ruido. El «estrato normativo» de S11 no está definido; la lectura más
cercana medible, el recall sobre cláusulas relevantes normativas, va de 0,96 a 0,98. Una cita
`not_found` nueva (0 → 1). Coste ×1,12, tokens +24 %.

**`xref+bounded`:** solo se considera si `xref` y `bounded` pasan por separado.

### 5.3 Aspectos, uno a uno — [DOMINIO]

Ejecuciones resueltas (de 3) / literal / citado por `[n]` con el chunk respuesta, con o sin cita
literal (*post hoc*, fuera del criterio, §5.5):

| Aspecto | `xref` | `xref+bounded` | Por qué no, donde no |
|---|---|---|---|
| q01-a1 | **3** / 3 / 3 | **3** / 3 / 3 | — |
| q06-a1 | 0 / 0 / 3 | 1 / 1 / 3 | **la fila de R.2 llega y se cita por `[n]`, parafraseada** |
| q06-a2 | 0 / 0 / 0 | 1\* / 1 / 0 | R.2 llega sin su chunk respuesta (t12): **tope por cláusula** |
| q09-a1 | **3** / 3 / 3 | **3** / 3 / 3 | — |
| q09-a3 | 0 / 0 / 0 | 0 / 0 / 0 | 6.2.7.8 no entra: **`truncated`** (tope por cláusula) |
| q11-a1 | 0 / 0 / 3 | 1 / 0 / 3 | **7.2.1.2 llega y se cita por `[n]` en 6 de 6, parafraseada** |
| q01-a2 (`same_clause`) | 0 / 0 / 0 | 0 / 0 / 3 | con `bounded`: tabla A-1 traída y citada sin cita literal |
| q03-a1 (`same_clause`) | 0 / 1 / 3 | 0 / 1 / 3 | D.1::t0 traído y citado sin cita literal |

\* **Falso positivo de la regla preregistrada:** la única ejecución que «resuelve» q06-a2 cita la fila
de verificación independiente de R.2, que responde a q06-a1. Los dos aspectos comparten cláusula y
la regla es por cláusula. No mueve ningún recuento ≥ 2/3.

Comprobadas a mano, las resoluciones reales: q01-a1 cita «…with as a minimum SRR, PDR, CDR, QR and
AR» (5.3.3.2); q09-a1, «The existing software shall be assessed…» (Q-ST-80C 6.2.7.3); q06-a1, la fila
«Independent software verification plan» de la Tabla R-1; q11-a1, «The software requirements shall
be:» seguida de la lista.

**Desglose de lo no resuelto en `xref` (lo que el brief pide para decidir el paso siguiente):**
- **Traído y usado, sin cita literal:** q06-a1 y q11-a1 (y q03-a1 con `bounded`). El contenido llega,
  el modelo lo cita con su `[n]` y lo parafrasea. Ejemplo q11: «Software requirements shall be
  correct, unambiguous, complete, consistent, verifiable and traceable. [7.2.1.2]». La definición
  preregistrada exige cita `verified`, y el prompt admite «cita o paráfrasis».
- **Traído en parte, por los topes:** q06-a2 (R.2 entra con su chunk más cercano, no con t12) y
  q09-a3 (6.2.7 entra por similitud y 6.2.7.8 no cabe). Es el tope por cláusula de 600 tokens frente
  a chunks de tabla de ~500.
- **No detectado o no resuelto:** ninguno de los 6.

### 5.4 Por qué baja el recall con `xref` — [AGNÓSTICO]

No es que el modelo omita más: **la expansión trae cláusulas que el golden set sí juzga, y juzga
relevantes**, así que el denominador crece.

| | `control` | `xref` |
|---|---|---|
| relevantes en el contexto (denominador) | 216 | 297 (+38 %) |
| relevantes citadas (numerador) | 196 | 233 (+19 %) |
| … de ellas, de las que trajo el retrieval | 196 | 174 (−11 %) |
| cláusulas inyectadas / sin juicio | 318 / 0 | 567 / 159 |

El modelo cita 37 relevantes más en términos absolutos. El coste real es la **sustitución**: deja de
citar 22 relevantes recuperadas y cita en su lugar las añadidas (q09: de 5 a 1,3 recuperadas citadas,
porque cambia los handbooks por la normativa de Q-ST-80C 6.2.7; q01: de 2,3 a 1,0). El recall
condicionado mide «citadas / relevantes inyectadas», y castiga traer más contexto relevante aunque
se use más. La guarda 2 lo cuenta como pérdida tal como está escrita.

### 5.5 Lo que el chat necesita para la regla — sin recomendación

- **La regla y sus guardas, tal como están escritas, no las pasa ninguna configuración.** `xref`
  falla las guardas 1 (2 de 6) y 2 (−0,08). `bounded` no resuelve ninguno de `same_clause`.
- **Hay dos efectos de definición que el chat debe ver antes de decidir**, sin cambiar el criterio a
  posteriori:
  - Con la lectura *post hoc* «citado por `[n]` con el chunk respuesta en contexto», `xref` daría
    **4 de 6** (q01, q06-a1, q09-a1, q11) y `bounded` 2 de 2. La diferencia entre «llega» y
    «resuelto» es, sobre todo, paráfrasis frente a cita literal.
  - La caída de recall es sobre todo inflación del denominador (§5.4). Una métrica sobre las
    relevantes recuperadas, o en absoluto, contaría otra cosa.
- **Lo que no es de definición:** los topes cortan dos objetivos (q06-a2, q09-a3), y la sustitución
  de 22 citas recuperadas es real. Ninguno de los dos se ha iterado (§3.2 del brief).
- **Hipótesis anotadas, sin implementar:**
  - **H-f.** Un tope por cláusula pensado para filas (traer la fila de tabla, no el chunk entero)
    cubriría q06-a2. Depende de las tablas aplanadas, ya en la lista.
  - **H-g.** El prompt podría pedir cita literal cuando la afirmación transmite un requisito, como ya
    hace para las listas. Es cambio de prompt, fuera de este brief.
  - **H-h.** Las referencias a estándar entero (q08, q10, q12, q14) siguen sin seguirse; es el sitio
    del bucle que el chat dejó para después.

