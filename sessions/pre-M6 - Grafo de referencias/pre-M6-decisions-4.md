# pre-M6 — Decisiones del brief 4: profundidad dentro del documento destino

> **No se adopta.** La variante de §7 (10 candidatos, 10 chunks, 3.000 tokens, puerta 0,3991) falla
> dos guardas preregistradas: **G1** (1 de 3 ganados resueltos) y **G5** (q10-a1, 0 de 3). Los
> defaults no cambian. Una pasada real: **0,5327 USD** (`generation_20261105-134017`). Sin UAT (§7).

---

## 1. Hecho

- Veredictos de `pre-M6-uat-3.md` transcritos a `records.jsonl` (`ok`, chat).
- **Variantes con nombre** en `corpus/groundtruth/retrieval/depths.py` (`deep`): la calibración, el
  alcance, el tamaño y el arnés leen los mismos números de ahí (`--depth`, configuración
  `doc_pointers_deep`).
- `doc_pointer_check calibrate` exporta los pares sin juicio para el modo B.
- **Guardas v2** (`graph_guards.py`, `GUARDS_VERSION = 2`): R cuenta lo **ganado** frente a
  control, G1 solo lo que control no resuelve y la guarda 1b conserva lo que sí resuelve. **v1
  sigue reproducible**: `graph_report --guards-version 1` da el veredicto del brief 3.
- `context_digest` con la configuración `clause`.

## 2. Primera variante (D1) y parada

D1: 10 candidatos y 5 chunks. Puerta recalibrada con 10 candidatos tras juzgar 81 pares en modo B
(9 relevantes): **0,3991** (97 admitidas, 47 relevantes, 0,485; el siguiente punto, 0,4019, da
0,480). R = 1 (solo q12-a1): **parada sin pagar**. El tope vinculante era el de 5 chunks por
puntero, no los candidatos.

**Nota de medida:** dos calibraciones sin juicios nuevos dieron 0,3727 y 0,3726. Los embeddings de
consulta se recalculan y varían en la cuarta cifra.

## 3. Variante corregida (§7, E1–E5)

**Comprobación sin generación** (`reach_20261105-124322`):

- Ganados frente a control: **q12-a1, q12-a4, q15-g1** → R = 3, N = 3. q12-a3 pasa a `over_limit`:
  comparte cláusula con q12-a4, y su chunk de evidencia no entra. Ningún negativo cambia en L2.
- Digest: `clause` es byte a byte el contexto del brief 2, y `defaults` el del brief 3.
- Tamaño del contexto (33 turnos):

  | | chunks mediana / p95 | tokens mediana / p95 |
  |---|---|---|
  | control (`doc_pointers`) | 37 / 69 | 5.290 / 7.176 |
  | variante (`doc_pointers_deep`) | 39 / 76 | 5.693 / 7.902 |

**Incidencia:** Docker Desktop estaba parado al empezar. La primera repetición de la comprobación
falló contra la base y el diff que salió en pantalla era el del contrafactual anterior. Se levantó
el stack y se repitió; las cifras de arriba son de la ejecución buena.

## 4. Guardas v2 (`graph_guards_*.md` junto al artefacto)

174 ejecuciones comparables, 0 errores, 0 fallback, VALID.

| # | Guarda | Valor | Umbral | |
|---|---|---|---|---|
| 1 | Ganados resueltos ≥ 2/3 | **1** | ≥ 3 | **falla** |
| 1b | Los que control resuelve siguen resueltos (q12-a2, q15-g2, q26-g2) | 1,00 | 1,00 | pasa |
| 2 | Recall sobre relevantes recuperadas | 0,85 | ≥ 0,84 (control 0,87) | pasa |
| 3 | `verified` | 1,00 | 1,00 | pasa |
| 4 | Precisión sin lo que solo trajo el grafo | 0,87 | ≥ 0,83 (control 0,86) | pasa |
| 5 | Negativos | **0,67** | 1,00 | **falla** |

### Porqué no, por caso

| caso | resultado | porqué |
|---|---|---|
| q15-g1 | **3/3** | E-ST-10-02C §5.2.2.1 (rango 10) entra y se cita |
| q12-a1 | 0/3 | **traído y no citado.** M-ST-80C §5.2.2.1 está en contexto en las 3 ejecuciones, juzgada relevante. El modelo cita §6.1, §6.2, §6.3 y §4.2 del mismo documento |
| q12-a4 | 0/3 | **traído y no citado.** §A.2.1 (registro de riesgos) en contexto en las 3. En la ejecución 2 el modelo declara `not_in_material` «how software-specific risks are ranked or recorded in the project risk register» con esa cláusula inyectada |
| q10-a1 (negativo) | **0/3** | El puntero de cortesía a Q-HB-80-04A trae **8 cláusulas** dentro de la puerta (con 3 chunks traía 3). El modelo cita cláusulas no relevantes que solo trajo el canal: §A.3.2 (ejecuciones 1 y 3) y §1 (ejecución 2). Con el brief 3 no citaba nada de ese documento |
| q27-g1, q28-g1 | 3/3 | `out_of_corpus` / no seguidos |

Lectura: **los dos fallos van en direcciones opuestas.** Con más profundidad, en q12 entra lo
relevante y el modelo no lo usa (10 cláusulas de M-ST-80C compiten en el contexto), y en q10 entra
lo no relevante y el modelo sí lo usa. La puerta por distancia no separa un destino útil de uno de
cortesía cuando el documento entero está cerca de la consulta. El problema es de selección y de
generación, no de alcance: el techo de R ya estaba alcanzado.

## 5. Se reporta (no decide)

| | `doc_pointers` | `doc_pointers_deep` |
|---|---|---|
| latencia mediana / p95 | 17,4 / 33,5 s | 17,1 / 30,2 s |
| etapa `doc_pointers` mediana | 8,2 ms | 8,0 ms |
| coste por turno | 0,00288 USD | 0,00296 USD |
| tokens de entrada mediana / p95 | 10.804 / 13.551 | 11.215 / 14.817 |
| `not_found` por ejecución | 1,11 | 1,18 |
| recall condicionado (arnés, global) | 0,86 | **0,81** (−0,05) |
| precisión (arnés, global) | 0,86 | 0,86 |
| citadas / inyectadas | 0,41 | 0,38 |
| citas fuera de su afirmación | 15 (7 con canal, 8 sin) | 33 (13 con canal, 20 sin) |
| cláusulas solo del grafo citadas | 46 | 78 |

- **El recall condicionado cae 0,05** (`compare.py` lo marca FAIL; no es guarda). Es la inflación del
  denominador ya vista con `xref`: más cláusulas relevantes inyectadas sin citar (q12 pasa de 3 a
  10 cláusulas de M-ST-80C). La guarda 2, sobre lo recuperado, la aísla y pasa (−0,02).
- **`delegated` por clase:** igual en `in_corpus_not_injected` (6 → 6) e `in_corpus_injected` (8 →
  8); `out_of_corpus` 3 → 0 y `unresolved` 14 → 16, que es varianza de generación sin cambio de
  contexto que la explique.
- **Citas fuera de su afirmación:** 20 de 33 en ejecuciones sin nada del canal. Sigue siendo sobre
  todo varianza del generador, como en el brief 3.
- La latencia no sube: el modelo domina (95 %) y su varianza tapa los ~400 tokens de más.

## 6. Decisión

**No se adopta** (§7: «si no, se cierra con "no se adopta"»). Defaults sin cambios: `clause_standard`
con 5 candidatos, 3 chunks, 2.000 tokens y puerta 0,3758. La variante `deep` se queda en
`depths.py` como instrumento, con los valores de E1 y su puerta 0,3991, para reproducir esta
medición.

**Sin UAT.** §7 la ataba a la variante y la variante no se sirve: el producto no cambia y no hay
render nuevo que capturar. Si el chat la quiere como documentación del fallo, se ejecuta por API con
`.env` temporal (~0,01 USD).

## 7. Para el chat (sin implementar)

- **La profundidad deja de ser la palanca.** Con 10 chunks el contexto llega y la generación no lo
  usa (q12), y la misma profundidad sirve ruido de un puntero de cortesía (q10). Lo siguiente con
  señal es **selección**: una puerta relativa o un segundo peldaño (ya diferido por q14), o tratar
  aparte los punteros de cortesía (alcance general, «in accordance with», que nombran el documento
  sin apuntar a contenido).
- **q12-a1 y q12-a4 son un caso de generación**, no de grafo: la cláusula está y no se cita. Va con
  el diferido de soporte de afirmación y de cobertura de aspectos.
- E4 se cumplió como riesgo: 2 de los 3 ganados eran de q12, y los dos fallan por la misma causa.
