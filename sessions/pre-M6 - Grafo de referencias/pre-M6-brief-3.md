# pre-M6 — Brief 3: punteros a documento con puerta de relevancia

> Rama `pre-m6-grafo`. Una variable: **seguir punteros a documento entero** (`explicit_document`).
> **Una pasada real autorizada: ≤ 1,00 USD** (estimado ~0,70: 33 turnos × 3 ejecuciones × 2
> configuraciones). Antes de pagarla, comprobación sin generación (§3) con parada si no hay techo.
> Primer paso: transcribir a `records.jsonl` el veredicto del chat de `pre-M6-uat-2.md`.

---

## 0. Contexto

L2: 10 de 11 positivos `explicit_document` son `pointed_not_followed` (q08, q12 ×4, q14, q15 ×2,
q16 ×2). `clause_standard` (S13) ya hacía el retrieval acotado al destino y no se adoptó por la
**política de plazas** (orden de aparición, plazas compartidas con `xref`) y la falta de **puerta**,
no por el concepto. Además, la UAT del brief 2 muestra contextos de ~50 chunks antes de seguir nada.

## 1. Decisiones

| # | Decisión | Porqué |
|---|---|---|
| P1 | **Canal nuevo en el acumulador**, `origin = graph_document`, `edge_kind = explicit_document`, `hop = 1`, precedencia **3** (tras `xref`). Sustituye a la implementación de `clause_standard`: el modo `RETRIEVAL_XREF=clause_standard` pasa a usar esta política; la versión sin puerta se elimina (sus artefactos quedan) | Una sola forma de seguir punteros a documento |
| P2 | **Qué se lee:** punteros a documento en el texto de evidencia `hop = 0` (retrieval y expansión). Lo traído por `hop = 1` no se lee (K5) | q16 (B.2.2) llega por expansión; con `hop ≤ 1` sigue siendo un salto |
| P3 | **Resolución:** destino contra la tabla `documents` con el parser actual. `out_of_corpus` / `ambiguous` / `own_document` no se siguen y quedan en la traza | Determinista; q28 debe quedar `not_followed` |
| P4 | **Retrieval acotado:** la consulta de producción (con historial) contra los chunks del documento destino, top `RETRIEVAL_DOC_POINTER_CANDIDATES = 5` por puntero | Mismo vector que el retrieval principal: sin llamada extra |
| P5 | **Puerta de relevancia, primer peldaño: umbral de distancia** `RETRIEVAL_DOC_POINTER_MAX_DISTANCE`. Calibrado **una vez**, antes de generar, sobre los candidatos de P4 en las 29 consultas con los juicios del golden set v3: el umbral más permisivo con el que la precisión de lo admitido sea **≥ la p@k de `retrieval` en `defaults`** (L1: 0,48). Se reporta como calibrado *in-sample* (29 consultas) | Regla de producto: no servir por grafo nada peor que lo que ya se sirve. Sin cross-encoder ni juez LLM mientras el umbral baste |
| P6 | **Prioridad entre punteros, explícita:** (1) autoridad de la cláusula de origen, normativa antes que handbook; (2) mejor distancia admitida del puntero; (3) `ClauseKey` de origen como desempate. Nunca orden de aparición | Aprendizaje 9 de S13; preferencia de autoridad |
| P7 | **Plazas propias**, no compartidas con `xref`: `RETRIEVAL_DOC_POINTER_MAX_POINTERS = 3`, `..._MAX_CHUNKS_PER_POINTER = 3`, tope de tokens `..._TOTAL_TOKENS = 2000`. Degradación `doc_pointer_over_limit` en el contrato (no se pinta en la UI) | Lo que tumbó a `clause_standard`: q14-a1 se quedó fuera estando en rango 2 |
| P8 | **Autoridad en el destino:** si un puntero lleva a un handbook y en contexto ya hay una cláusula normativa con la misma `ClauseKey` destino… no aplica (claves distintas). La preferencia se ejerce solo vía P6. Anotar en decisions si aparece un caso real | No inventar regla sin caso |
| P9 | **Default tras la medición:** si pasan las guardas, `RETRIEVAL_XREF=clause_standard` pasa a default. Si no, queda apagado con el porqué no | Adoptar por error asimétrico |

## 2. Guardas (preregistradas en el brief 1, `graph_guards.py`)

Pareado `control` (`defaults`) frente a `variant` (`clause_standard` nuevo), 29 consultas / 33
turnos, 3 ejecuciones. Objetivo del brief: los 11 positivos `explicit_document`.

**N se fija antes de generar con esta regla:** `R` = positivos `explicit_document` `reached` por la
variante en la comprobación sin generación (§3). `N = max(3, ⌈R/2⌉)`. Si `R < 3`, **no se paga la
pasada**: se reporta el porqué no por aspecto y se para.

Guardas 2–5 como en el brief 1. Negativos del tipo: q10-a1, q27-g1, q28-g1 → `not_followed` o nada
citado.

## 3. Comprobación sin generación (antes de pagar)

1. Calibrar P5 y fijar el valor en `config.py` y en decisions.
2. `graph_reach run` con la variante: tabla L2 completa; `R` y `N`.
3. Digest de `defaults` igual al del brief 2 (la variante está apagada por defecto).
4. Tamaño de contexto: mediana y p95 de chunks y tokens de entrada, `control` frente a `variant`.

## 4. Se reporta (no decide)

Latencia mediana/p95, coste por turno, tokens de entrada, `not_found` por ejecución, `delegated` por
clase (D1: se espera que bajen los `in_corpus_not_injected` de q12 y q14), citas fuera de su
afirmación (deriva vigilada: 25 → 43 en S14).

## 5. UAT

Por API con la variante: q12 (turno 1) y q14 (turno 3). Por evidencia `graph_document`: `from_clause`
contiene el puntero, el destino es el documento nombrado, distancia ≤ umbral. Captura del render
(Playwright) de q12: la traza de evidencias muestra las nuevas sin romper el panel. Registros con
`verdict = pending`; juzga el chat.

## 6. Cierre

`pre-M6-decisions-3.md` (umbral y su curva de calibración, L2 de la variante, guardas con el porqué no
de cada aspecto objetivo no resuelto, lo reportado, decisión de default), `pre-M6-uat-3.md`, docs y
commit.
