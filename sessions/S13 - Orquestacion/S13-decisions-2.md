# S13 · Decisiones — Brief 2: punteros a un estándar entero

> Decisiones tomadas al implementar `S13-brief-2.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Rama `s13-orquestacion-agentes`. El §1 está escrito **antes** de la pasada y no se ha tocado
> después; los resultados empiezan en el §2.

## 1. Preregistro (escrito antes de medir)

### 1.1 Resolución — [AGNÓSTICO]

- **W1.** `RETRIEVAL_XREF` admite `off | clause | clause_standard`, con `clause` por defecto (sin
  cambio). `clause_standard` hace todo lo que hace `clause` y además sigue los punteros a un
  estándar entero.
- **Qué es candidato.** Una referencia `standard` del parser (`references.extract`, sin tocar)
  pasa por `normalize_standard`. Lo que no resuelve a un único documento del corpus se registra con
  su motivo (`out_of_corpus`, `ambiguous`) y no se sigue. Los punteros por título sin
  identificador (q14-a2) no los lee el parser, así que siguen fuera.
- **W3.** Si el documento está entre los que filtró el routing de la petición, no se sigue y se
  registra como `already_searched`. Con `quota` cuenta igual: el retrieval filtrado a esos
  documentos ya se ejecutó.
- **Un estándar nombrado varias veces es un solo candidato**, en la posición de su primera
  aparición (igual que una cláusula repetida).
- **No hay regla de «estándar propio».** Un chunk que nombra su propio estándar no se excluye
  (W3 solo excluye el filtro de routing). En la comprobación previa no aparece ningún caso.
- **W2.** El retrieval filtrado reutiliza la consulta y el vector de la petición, con el mismo
  modo, `top_k` (10) y canal léxico: `rank(..., canonical_doc_ids=[doc])`. No hay embedding nuevo
  (un test cuenta las llamadas). Cuesta una consulta a la base por estándar candidato.

### 1.2 Plazas y selección — [AGNÓSTICO]

- **W4.** Un estándar seguido ocupa **una plaza** del tope de 5 (`RETRIEVAL_XREF_MAX_CLAUSES`),
  compartido con las cláusulas y **en orden de aparición** de los punteros en el texto inyectado.
  Tiene el mismo tope por plaza (600) y el mismo total (2.000, compartido con las cláusulas). No
  hay ningún tope nuevo.
- **W5.** Dentro de la plaza entran los chunks del retrieval filtrado en orden de rango, sin los
  que ya están en el contexto, hasta que el siguiente no cabe en la plaza o en el total; ahí se
  para. Es la misma regla que corta una cláusula.
  - Un estándar sin nada nuevo que añadir (`already_injected`, o `no_hits` si el filtrado sale
    vacío) **no ocupa plaza**.
  - Uno cuyo primer chunk no cabe ocupa su plaza sin añadir nada y queda marcado `truncated`,
    como una cláusula cortada a cero.
- **W6.** Lo añadido entra con `origin="expansion"` y `distance=None`, como cualquier expansión,
  al final del contexto. El prompt no cambia.
- **W7.** Con `clause`, los chunks y la traza son idénticos byte a byte a los de antes del cambio:
  digests tomados antes de tocar el código, sobre un texto con punteros a cláusula y a estándar.
  Además, los digests del array de mensajes de `test_history_equivalence` siguen en verde.
- **Traza.** `RetrievedContext.xref` y `references_expanded` llevan, solo con `clause_standard`,
  una lista `standards`. Cada puntero aparece con `text`, `source`, `doc`, `status` (`followed`,
  `already_searched`, `over_limit`, `already_injected`, `no_hits`, `out_of_corpus` o
  `ambiguous`), `chunks` con su rango en el retrieval filtrado, `tokens` y `truncated`. En ese modo
  los punteros a estándar ya no entran en `unresolved` como `whole_standard`, sino en `standards`.

### 1.3 Medición — [AGNÓSTICO] salvo donde se indica

- **Configuraciones.** El `control` del brief (`bounded` + `clause`) es la configuración
  **`xref+bounded`** del arnés, la misma que define la referencia. El nombre `control` ya existe
  en el arnés (`orphans` + `off`, el de S12) y renombrarlo rompería la reproducción de los
  artefactos anteriores. La nueva configuración es **`standard`** (`bounded` + `clause_standard`).
  Comando:

  ```
  harness --config xref+bounded --config standard --runs 3
  ```

- **Aspectos objetivo [DOMINIO]:** q08-a2, q12-a1, q12-a2, q12-a3, q12-a4 y q14-a1. q10-a1 queda
  fuera por espurio.
- **Resuelto (decide):** una afirmación cita por `[n]` (sus `refs` o las de su cita) una
  cláusula que contiene un **chunk respuesta del estándar apuntado** (`follow.doc` del
  diagnóstico). Ese chunk se identifica por la cita literal que el diagnóstico registró en él, y
  la referencia (`generation_20261019-141324`) no lo tenía.
  - **Por qué solo el estándar apuntado:** q08-a2 tiene también un chunk respuesta en una
    cláusula hermana del handbook (Q-HB-80-03A 6.2.2.3), que no depende de seguir el puntero.
    Contarlo atribuiría a la técnica algo que puede traer el ranking.
  - **Al lado, sin decidir:** la misma lectura con cualquier chunk respuesta del diagnóstico, y la
    lectura literal (cita `verified` en ese chunk).
- **Recall sobre relevantes recuperadas:** relevantes que el retrieval puso por rango (las
  cláusulas con al menos un chunk no `expansion`) y que se citan, entre relevantes recuperadas.
  Media de las ejecuciones comparables. Al lado, **relevantes citadas en absoluto** y, de ellas,
  las recuperadas.
- **Por qué no** (ejecución no resuelta), en este orden:
  1. `brought_not_cited`: el chunk respuesta estaba en el contexto.
  2. `trimmed_by_budget`.
  3. Según lo que pasó con el puntero al estándar:
     - `not_in_slot`: se siguió, o no tenía nada nuevo, y el chunk respuesta no entró;
     - `already_searched`;
     - `over_limit`;
     - `not_detected`: ningún puntero a ese estándar;
     - `standards_off`: en `control`.
- **Criterio §3.2 del brief, sin cambios.** CC no lo aplica ni cambia defaults: reporta la tabla.
  Si `standard` no pasa, no se itera.

### 1.4 Comprobación previa sin generación (§2.3 del brief) — [DOMINIO]

`scripts/standard_pointer_precheck.py` (salida en `scripts/output_standard_precheck.txt`): el
proveedor de `standard` sobre la consulta de retrieval del turno juzgado (q14 con sus dos
preguntas previas), sin modelo. Se usan 10 embeddings: 3 del proveedor (uno por consulta, el de
la petición) y 7 de diagnóstico, para leer el rango del chunk respuesta en el top 50 filtrado.
No se ha ajustado nada a la vista de ella.

| Aspecto | Estándar | Qué pasa con el puntero | Chunk respuesta en contexto | Rango filtrado |
|---|---|---|---|---|
| q08-a2 | Q-ST-30-02C | `followed`, entran los rangos 2–4 | no | 15 |
| q12-a1 | M-ST-80C | `followed`, entran los rangos 1–5 | **sí** | 4 |
| q12-a2 | M-ST-80C | ídem | **sí** (6.1::s0); 7.3::r1 no | 3; > 50 |
| q12-a3 | M-ST-80C | ídem | no | > 50 |
| q12-a4 | M-ST-80C | ídem | no | 9 |
| q14-a1 | E-ST-70-01C | **`over_limit`** | no | 2 |

- **Como mucho 2 de 6 pueden resolverse** (q12-a1 y q12-a2). El retrieval es determinista y la
  consulta de q14 no depende de las respuestas previas, así que el criterio 1 (≥ 3) **no se puede
  cumplir** antes de generar. El brief pide medir igualmente: la pasada da además las guardas, el
  «por qué no» y el desglose de latencia del brief 1.
- **q14-a1 es efecto de W4.** E-ST-70-41C 6.18.1.1 nombra E-ST-70-01 después de que los punteros
  a cláusula del mismo texto ya hayan llenado las 5 plazas. Con el orden de aparición, el
  estándar llega tarde aunque su respuesta esté en el rango 2.
- **Plazas que no añaden nada.** E-ST-40C en q08 (sin total restante) y Q-HB-80-03A en q12 (el
  primer chunk pasa de 600) se siguen con 0 chunks y ocupan plaza, igual que una cláusula cortada
  a cero. No cambia la selección: solo se añadió `truncated` a la traza para que se vea.
- **q08 sigue cinco estándares** (Q-HB-80-03A, Q-ST-80C, Q-ST-30-02C, Q-ST-40C, E-ST-40C), que
  llenan el total de 2.000. Es la consulta que más contexto añade.
- Latencia de la etapa `xref` en la comprobación: 367 ms (q08, con la carga única de
  `table_labels`), 191 ms (q12) y 126 ms (q14).

## 2. Resultados (después de medir)

Artefacto `generation_20261023-191136` (14 × 3 por configuración, Haiku 5.5): 84 de 84
ejecuciones comparables, 0 fallback, 0 errores, 0 reintentos por esquema, 0 recortes por
presupuesto y 0 rechazos con citas. Coste total 0,249 USD con 92 llamadas. Salida 3 por cláusulas
sin juicio (63 en `xref+bounded`, 80 en `standard`); los pares están exportados en
`pairs_generation_20261023-191136.json` y no deciden nada aquí.

### 2.1 El instrumento vale — [AGNÓSTICO]

`control` (`xref+bounded`) frente a la referencia `generation_20261019-224307`
(`comparison_20261023-191201.md`):

| | Referencia | Hoy | Δ exacta |
|---|---|---|---|
| Recall | 0,861 | 0,836 | −0,025 |
| Precisión | 0,876 | 0,862 | −0,014 |
| `verified` | 1,00 | 1,00 | 0 |

Las dos diferencias están dentro de ±0,03. Las citas fuera de su afirmación bajan de 25 a 15. No
hace falta segunda pasada: el desglose de latencia cuadra (`unaccounted` con mediana de 4–5 ms).

### 2.2 Tabla del criterio §3.2 — sin aplicar

Pareado `standard` frente a `control`, mismo artefacto (`aspects_20261023-191228.md`):

| # | Guarda | `control` | `standard` | ¿Cumple? |
|---|---|---|---|---|
| 1 | ≥ 3 de 6 aspectos resueltos por `[n]` en ≥ 2 de 3 | 0 | **1** (q12-a2) | **no** |
| 2a | Recall sobre relevantes recuperadas no baja > 0,03 | 0,84 | 0,87 | sí (+0,03) |
| 2b | Relevantes citadas en absoluto no bajan | 238 | 295 | sí (+57; recuperadas 178 → 183) |
| 3 | `verified` en 1,00, sin citas no literales ni mal atribuidas nuevas | 1,00 | 1,00 | sí |
| 4 | Precisión sin cláusulas solo de expansión no baja > 0,03 | 0,88 | 0,92 | sí |

**Reportado, sin ser guarda:**

| | `control` | `standard` |
|---|---|---|
| Latencia mediana / p95 | 16,6 / 23,7 s | 17,8 / 27,2 s |
| Coste por turno | 0,00269 USD | 0,00287 USD (+7 %) |
| Tokens de entrada (mediana) | 10.546 | 10.570 (lejos de 30.000) |
| `not_found` por ejecución | 1,19 | 1,50 |
| Estados | 10 `answered`, 32 `partial` | 9 `answered`, 33 `partial` |
| Precisión estándar | 0,86 | 0,86 |
| Recall condicionado | 0,836 | 0,846 |
| Citas verificadas sobre chunks añadidos | 153 | 225 |
| Citas a cláusulas solo de expansión | 80 | 156 |

Embeddings: `clause_standard` no añade ninguno (lo prueba un test). El arnés no los cuenta por
separado: hay uno por petición, 92 en total.

### 2.3 Aspectos uno a uno — [DOMINIO]

| Aspecto | Resuelto por `[n]` | Literal | Por qué no |
|---|---|---|---|
| q08-a2 | 0/3 | 0/3 | `not_in_slot` ×3: la respuesta está en el rango 15 de Q-ST-30-02C y entran los rangos 2–4 |
| q12-a1 | 1/3 | 1/3 | `brought_not_cited` ×2: M-ST-80C 5.2.2.1 entra, pero el modelo no siempre lo cita |
| q12-a2 | **3/3** | 2/3 | — Las tres respuestas citan M-ST-80C 6.1 («The results of risk management are considered in the routine project management process…») |
| q12-a3 | 0/3 | 0/3 | `not_in_slot` ×3: la respuesta queda más allá del rango 50 |
| q12-a4 | 0/3 | 0/3 | `not_in_slot` ×3: rango 9; la plaza se llena en el 5 |
| q14-a1 | 0/3 | 0/3 | `over_limit` ×3: rango 2, pero las 5 plazas ya las ocupan punteros a cláusula anteriores |

Con la lectura de cualquier chunk respuesta, también 1 (q12-a2). Coincide con la comprobación
previa: solo q12-a1 y q12-a2 podían entrar.

**Efecto lateral, sin interpretar:** q11-a1 (conjunto `xref`) pasa de 0/3 a 2/3 resuelto con cita
literal en `standard`. Por `[n]` estaba en 3/3 en las dos configuraciones.

### 2.4 Desglose de «por qué no»: ¿tiene caso la reformulación?

Es una lectura del desglose, no una recomendación. De los 5 aspectos no resueltos:

- **Uno es de consulta.** q12-a3: la misma consulta no llega ni al top 50 de M-ST-80C. Es el único
  caso que pide reformular dentro del estándar (H-b de S12).
- **Tres son de política de plazas:**
  - q14-a1 (rango 2): el orden de aparición frente al tope de 5;
  - q12-a4 (rango 9) y q08-a2 (rango 15): la profundidad que cabe en 600 tokens por plaza.

  Ninguno necesita una llamada más. W4 aceptaba de antemano que el 9 y el 15 no entrarían.
- **Uno es de generación.** q12-a1: llega y no siempre se cita.

### 2.5 Latencia por etapa (las dos configuraciones) — [AGNÓSTICO]

Mediana sobre 42 ejecuciones cada una:

| Etapa | `control` | `standard` |
|---|---|---|
| `llm` | 15.297 ms (**93,1 %**) | 16.873 ms (**93,8 %**) |
| `guardrails` | 480 ms (3,8 %; p95 1.543) | 484 ms (2,7 %) |
| `embedding` | 258 ms (1,8 %) | 264 ms (1,8 %) |
| `db` | 133 ms (0,8 %) | 124 ms (0,7 %) |
| `xref` | 60 ms (p95 92) | **128 ms** (p95 347) |
| `expansion` | 4 ms | 4 ms |
| `prompt`, `validation`, `compose` | < 4 ms | < 4 ms |
| `unaccounted` | 4,0 ms | 4,7 ms |
| Total | 16.600 ms | 17.789 ms |

- **La generación domina** (93–94 %). Paralelizar retrieval o `xref` ahorraría, como mucho, unos
  pocos cientos de ms sobre ~17 s.
- La única etapa que no es el modelo y supera los 0,5 s en p95 es `guardrails` (la llamada a
  moderación).
- Los 2–4 s de `guardrails` + `embedding` de la UAT del brief 1 eran arranque en frío.
- `clause_standard` añade +68 ms de mediana en `xref`. Los +1,2 s del total salen de `llm` (+1,6
  s): más contexto y más salida, no la etapa nueva.

## 3. Tests

**1.449 → 1.470** (+21). Modificados 0 y borrados 0.

- **+12 en `tests/app/test_references.py`:**
  - W7 con digests anteriores al cambio;
  - resolución: del corpus, fuera del corpus, ambiguo, por título, `already_searched`, repetido;
  - sin `standards`, sigue `whole_standard`;
  - selección: orden de rango y exclusión, topes por plaza y total con `truncated`, plaza
    compartida en orden de aparición con `over_limit`, sin nada nuevo sin plaza;
  - en el proveedor: mismo vector sin embedding nuevo, y documento ya filtrado por routing.
- **+9 en `tests/corpus/test_generation_aspects_standard.py`:** resuelto por `[n]`, literal,
  llega sin citarse, cita a otra cláusula, lo que la referencia ya tenía, cada motivo de «por qué
  no», recorte por presupuesto, cualquier chunk respuesta, y recall sobre recuperadas con los
  recuentos.

## 4. Etiquetado

- **[AGNÓSTICO]:**
  - la arista condicional determinista (puntero a cláusula → por id; puntero a documento →
    retrieval filtrado con la misma consulta);
  - plazas compartidas en orden de aparición;
  - las métricas corregidas: resuelto por `[n]`, recall sobre recuperadas, recuentos absolutos;
  - el desglose de «por qué no» como entrada de la decisión siguiente;
  - el desglose de latencia: la generación domina.
- **[DOMINIO]:**
  - los 6 aspectos y sus rangos;
  - el caso de q14 (E-ST-70-41C nombra muchas cláusulas antes que el estándar de OBCP);
  - que M-ST-80C quede fuera del routing de Q-ST-80C (H-d).
