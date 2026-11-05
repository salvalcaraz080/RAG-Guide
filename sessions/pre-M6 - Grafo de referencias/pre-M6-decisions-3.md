# pre-M6 — Decisiones del brief 3: punteros a documento con puerta de relevancia

> Pasan las cinco guardas. `RETRIEVAL_XREF=clause_standard` pasa a default (P9), con dos reservas que
> el chat debe ver antes de dar el brief por bueno (§5). Una pasada real: **0,53 USD** (arnés 0,5153,
> UAT 0,0132). Artefacto: `generation_20261102-111935.json`.

---

## 1. Lo implementado

| # | Qué | Dónde |
|---|---|---|
| P1 | Canal `graph_document` del acumulador, precedencia 4 (tras `xref`), `edge_kind = explicit_document`, `hop = 1`. Sustituye a la implementación de `clause_standard`: el modo conserva el nombre y la versión sin puerta se retira (sus artefactos quedan) | `app/services/doc_pointers.py`, `accumulator.py`, `context.py` |
| P2 | Se leen los punteros del texto con `hop = 0` (`accumulator.readable()`) | `read_pointers` |
| P3 | Resolución con el parser actual; `out_of_corpus`, `ambiguous` y `own_document` quedan en la traza, agrupados | `read_pointers`, `XrefReport` |
| P4 | Consulta de producción contra el documento destino, mismo vector y modo, top `RETRIEVAL_DOC_POINTER_CANDIDATES = 5` | `ContextProvider._doc_pointers` |
| P5 | Puerta `RETRIEVAL_DOC_POINTER_MAX_DISTANCE = 0,3758` (§2) | `config.py`, `.env`, `.env.example` |
| P6 | Prioridad `(origen no normativo, mejor distancia admitida, ClauseKey de origen, documento)`; nunca orden de aparición (test con los dos órdenes) | `follow` |
| P7 | Plazas propias: 3 punteros, 3 chunks por puntero, 2.000 tokens; degradación `doc_pointer_over_limit` en el contrato, no pintada | `follow`, `pointer_degradations`, `presentation.py` |
| P8 | Sin regla nueva. **Ningún caso real** observado: ningún puntero a handbook encontró en contexto una cláusula normativa con su misma clave | — |
| P9 | Default `clause_standard` (§5) | `config.py`, `.env`, `.env.example` |

**Diferencias con el texto del brief, por simplificación:**
- **Sin `already_searched`.** Un puntero al documento que el routing ya filtró se lee como
  `own_document` (si sale del propio documento) o `already_injected` (si lo admitido ya está en
  contexto). La consulta restringida a ese documento devuelve lo mismo que ya filtró el routing, así
  que no hace falta un estado propio.
- **La evidencia `graph_document` conserva su `distance`**, al contrario que `expansion` y `xref`:
  la UAT la necesita para comprobar la puerta, y el canal sí rankea.
- **Lo que trae el canal se elige entre lo admitido que aún no está en contexto.** Un documento cuyos
  mejores chunks ya están dentro aporta los siguientes, y si no queda nada nuevo no ocupa plaza.

**Instrumentos nuevos:**
- `corpus/groundtruth/retrieval/doc_pointer_check.py`: `calibrate` (curva y umbral) y `size`
  (tamaño del contexto).
- `corpus/groundtruth/generation/graph_report.py`: las cinco guardas sobre un artefacto pareado.
- `graph_reach run --xref clause_standard`, ahora con la etiqueta de configuración correcta: decía
  `xref=clause` en la variante.

## 2. Calibración de la puerta (P5)

**Paso previo, modo B.** De las 108 cláusulas admisibles con la puerta abierta, 65 no tenían juicio
(17 consultas). El umbral oscilaba entre 0,297 y 0,632 según cómo se tratara lo no juzgado, así que
la calibración, que es de una sola vez, se paró. El chat juzgó los 65 pares
(`pair_judgments_doc_pointer_20261101-195729.json`: 7 relevantes) y entraron con `golden pairs`. La
curva exploratoria anterior (`doc_pointer_calibration_20261101-195729`) se generó antes de tener los
juicios y no decidió nada.

**Calibración** (`doc_pointer_calibration_20261102-101535.{md,json}`), 108 cláusulas, 0 sin juicio,
objetivo 0,48 (p@k de `retrieval` en `defaults`, L1). La regla toma el umbral más permisivo cuya
precisión de lo admitido sea ≥ 0,48:

| distancia | admitidas | relevantes | precisión |
|---|---|---|---|
| 0,2434 | 1 | 1 | 1,00 |
| 0,3092 | 10 | 4 | 0,40 |
| 0,3184 | 13 | 7 | 0,54 |
| 0,3299 | 22 | 12 | 0,55 |
| 0,3570 | 39 | 22 | 0,56 |
| **0,3758** | **45** | **22** | **0,49** ← elegido |
| 0,3768 | 46 | 22 | 0,478 |
| 0,4042 | 53 | 22 | 0,42 |
| 0,4924 | 95 | 34 | 0,36 |
| 0,6316 | 108 | 37 | 0,34 |

**Calibrado *in-sample*** sobre las 29 consultas con las que después se mide: el umbral está
ajustado a estas consultas. La curva no es monótona: entre 0,3092 y 0,3570 la precisión sube porque
llegan relevantes de q08 y q15. A partir de 0,3570 solo entran no relevantes, y el umbral queda en el
último punto que aún pasa el objetivo, así que un desplazamiento pequeño de la distancia lo mueve.

## 3. Comprobación sin generación

- **L2 de la variante** (`reach_20261102-101852`). Respecto a `defaults`, solo cambian cuatro
  aspectos, todos `explicit_document`:

  | aspecto | `defaults` | variante |
  |---|---|---|
  | q12-a2 | `pointed_not_followed` | **`reached`** |
  | q15-g2 | `pointed_not_followed` | **`reached`** |
  | q08-a2 | `pointed_not_followed` | `over_limit` |
  | q10-a1 (negativo) | `not_followed` | `followed` |

  Tabla `explicit_document` de la variante (14): `reached` 3, `pointed_not_followed` 7,
  `over_limit` 1, `not_followed` 2, `followed` 1. El resto de tipos, idénticos.
- **R = 3, N = max(3, ⌈3/2⌉) = 3.** Literal: los positivos `reached` por la variante son q12-a2,
  q15-g2 y q26-g2. **q26-g2 ya lo alcanzaba `defaults`**, por retrieval (el enlace ni siquiera está
  en contexto): la variante solo aporta **2**. La regla del brief cuenta lo alcanzado, no lo
  ganado, así que se pagó. Ver §5.
- **Digest de `defaults` igual al del brief 2** (`1195d38dedad9761`) antes de cambiar el default.
  También `xref_off`, `expansion_off` y `expansion_orphans`. Tras el cambio, el `defaults` nuevo
  es byte a byte el contexto de la variante medida.
- **Tamaño del contexto**, 33 turnos:

  | | chunks mediana | chunks p95 | tokens mediana | tokens p95 |
  |---|---|---|---|---|
  | control | 35 | 66 | 4.857 | 6.886 |
  | variante | 37 | 69 | 5.290 | 7.176 |

## 4. Guardas (`graph_guards_*.md` junto al artefacto)

Pareado `gaps` (los defaults del momento) frente a `doc_pointers`, 29 consultas, 3 ejecuciones, 174
ejecuciones comparables, 0 errores, 0 fallback, VALID (nada sin juicio).

| # | Guarda | Valor | Umbral | |
|---|---|---|---|---|
| 1 | Aspectos objetivo resueltos por evidencia y aspecto, ≥ 2/3 | 3 | ≥ 3 | pasa |
| 2 | Recall sobre relevantes recuperadas | 0,86 | ≥ 0,83 (control 0,86) | pasa |
| 3 | `verified` | 1,00 | 1,00 | pasa |
| 4 | Precisión sin lo que solo trajo el grafo | 0,87 | ≥ 0,82 (control 0,85) | pasa |
| 5 | Negativos `not_followed` o nada citado | 1,00 | 1,00 | pasa |

**Definición operativa de G1** (la fija `graph_report.py`): un aspecto se resuelve en una ejecución si
una afirmación cita por `[n]` una cláusula inyectada que contiene un chunk con **la cita de evidencia
de ese aspecto** (o la de una alternativa). Es la lección de q12-a3 en S14: la cita de evidencia es lo
que separa dos aspectos de una misma cláusula. Se acepta la paráfrasis con su `[n]`, como en la
corrección de S12. **G5:** un negativo se mantiene si ninguna cláusula de su destino que solo trajo
`graph_document` sale citada.

**Por aspecto objetivo** (ejecuciones resueltas, control → variante), con el porqué de cada uno no
resuelto, cruzado con el rango y la distancia de su cláusula respuesta dentro del documento destino:

| aspecto | control | variante | porqué no |
|---|---|---|---|
| q12-a2 | 0/3 | **3/3** | — |
| q15-g2 | 0/3 | **3/3** | — |
| q26-g2 | 3/3 | 3/3 | — (ya lo resolvía control, por retrieval) |
| q08-a2 | 0/3 | 0/3 | puntero seguido (desde Q-HB-80-03A §6.2.2.1); **la cláusula entra sin el chunk de la respuesta**: Q-ST-30-02C §4.1 llega con dos chunks (rangos 3 y 4), y el que contiene la cita de evidencia está en rango 7 u 11, dentro de la puerta: **tope de 3 chunks por puntero**. `over_limit` en L2 |
| q12-a1 | 0/3 | 0/3 | seguido; M-ST-80C §5.2.2.1 en rango 4 (0,3551, dentro): **tope de 3 chunks por puntero** |
| q12-a3, q12-a4 | 0/3 | 0/3 | seguido; §A.2.1 en rango 9 (0,3726, dentro): **fuera de los 5 candidatos** |
| q15-g1 | 0/3 | 0/3 | seguido; E-ST-10-02C §5.2.2.1 en rango 10 (0,3287): **fuera de los 5 candidatos** |
| q14-a1 | 0/3 | 0/3 | **`gated`**: E-ST-70-01C §5.4.4.2 en rango 2, pero a 0,4618, por encima del umbral |
| q16-g1, q16-g2 | 0/3 | 0/3 | **`not_read`**: el puntero está en Q-ST-80C §B.2.2, que llega por `xref` (`hop = 1`) y no se lee (K5). La premisa de P2 («llega por expansión») era incorrecta |

**Negativos:** q10-a1 se **sigue** (Q-HB-80-04A entra por puerta, pero no su §3.8, que está en rango
11) y en 3 de 3 ejecuciones no se cita nada que solo trajera el canal desde ese documento. q27-g1 y
q28-g1 (`out_of_corpus`) no se siguen. La puerta deja pasar un puntero de cortesía y es la
generación la que no lo usa: G5 mide lo servido, no lo que entra.

## 5. Decisión de default (P9) y sus dos reservas

**Pasa a default `RETRIEVAL_XREF=clause_standard`**, por la regla del brief: pasan las guardas y,
por error asimétrico, la precisión servida no baja (0,85 → 0,86 global, 0,87 sin lo añadido).

Reservas, para que el chat decida si confirma:

1. **G1 pasa justo, y por un aspecto que no depende de la variante.** Con la regla literal, R = 3 y
   N = 3, y q26-g2 cuenta en los dos lados. La ganancia real es 2 de 11 (q12-a2, q15-g2). Si el
   preregistro hubiera contado lo **ganado** frente a control, R = 2 < 3 y la pasada no se habría
   pagado. CC aplicó la regla tal como está escrita y no la reinterpretó.
2. **Las citas fuera de su afirmación suben 10 → 33** (`compare.py` lo marca FAIL; aquí se reporta
   sin decidir). Al desglosarlas, **25 de 33 caen en ejecuciones donde el canal no añadió nada**
   (q17, q19, q28, q29, q04, q13, q20: contexto idéntico al de control), así que son varianza del
   modelo. 8 caen en ejecuciones con `graph_document`: q09 ejecución 2 (7) y q03 (1). Es la misma
   deriva que S14 (25 → 43), sin cambio de contexto que la explique.

Volver atrás es una línea (`RETRIEVAL_XREF=clause`): el contexto con `clause` es byte a byte el del
brief 2.

## 6. Se reporta (no decide)

| | `gaps` | `doc_pointers` |
|---|---|---|
| latencia mediana / p95 | 17,4 / 27,1 s | 18,1 / 29,1 s |
| etapa `doc_pointers` mediana / p95 | — | 21 / 94 ms |
| coste por turno | 0,00279 USD | 0,00280 USD |
| tokens de entrada mediana / p95 | 10.666 / 12.994 | 10.804 / 13.551 |
| `not_found` por ejecución | 1,08 | 1,23 |
| citas fuera de su afirmación | 10 | 33 (§5) |
| cláusulas solo del grafo citadas | — | 43 |
| recall / precisión (arnés, global) | 0,85 / 0,85 | 0,84 / 0,86 |

La diferencia de latencia es casi toda del modelo (llm 16,4 → 17,2 s de mediana), dentro de su ruido.
La etapa nueva cuesta 21 ms.

**`delegated` por clase (D1):**

| clase | `gaps` | `doc_pointers` |
|---|---|---|
| `in_corpus_not_injected` | 10 | 7 |
| `in_corpus_injected` | 5 | 8 |
| `out_of_corpus` | 2 | 4 |
| `unresolved` | 11 | 10 |

Lo esperado se cumple en q12: de 3 `in_corpus_not_injected` a 2 `in_corpus_injected` (M-ST-80C
§E.3, §6.3 y §6.1 ya en contexto). **No se cumple en q14**: sigue en 1 `in_corpus_not_injected`
porque la puerta para su puntero. Q-HB-80-01A, en q09, aparece ahora como `in_corpus_injected`.

## 7. Pendiente, sin implementar

- **Profundidad dentro del documento:** 5 de los 8 no resueltos tienen la respuesta dentro de la
  puerta y fuera de las plazas (q08-a2 y q12-a1 por el tope de 3 chunks; q12-a3, q12-a4 y q15-g1
  por los 5 candidatos). Subir los topes es la palanca
  obvia, pero cambiaría la calibración (que se hizo con 5 candidatos) y engordaría el contexto.
  Decisión del chat.
- **q14-a1:** la puerta por distancia absoluta deja fuera un rango 2 a 0,46. Un umbral relativo al
  documento o un segundo peldaño de la puerta (el brief lo nombra) es decisión del chat.
- **q16:** leer punteros en lo que trae un salto choca con K5.
- **`latency.py`** solo acepta rutas, no el nombre del artefacto como los demás. Es preexistente y
  no se ha tocado.
