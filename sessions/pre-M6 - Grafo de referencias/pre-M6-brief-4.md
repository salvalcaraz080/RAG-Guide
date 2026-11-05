# pre-M6 — Brief 4: profundidad dentro del documento destino

> Rama `pre-m6-grafo`. Una variable: **cuánto se lee del documento al que apunta un puntero**
> (candidatos y plazas por puntero). **Una pasada real autorizada: ≤ 1,00 USD.** Primer paso:
> transcribir a `records.jsonl` los veredictos del chat de `pre-M6-uat-3.md`.

---

## 0. Contexto

`pre-M6-decisions-3.md` §4/§7: de los 8 aspectos `explicit_document` no resueltos, **5 tienen la
respuesta dentro de la puerta y fuera de las plazas**: q08-a2 y q12-a1 por el tope de 3 chunks por
puntero; q12-a3, q12-a4 y q15-g1 por los 5 candidatos. Es más techo que el de las aristas por
nombre (3), así que va antes.

Fuera de este brief (diferidos, registrados): q14-a1 (puerta absoluta deja fuera un rango 2 a 0,46;
umbral relativo o segundo peldaño), q16 (puntero en texto `hop = 1`, choca con K5), etiqueta de la UI
para punteros a documento (sesión de UI/UX).

## 1. Decisiones

| # | Decisión | Porqué |
|---|---|---|
| D1 | `RETRIEVAL_DOC_POINTER_CANDIDATES` 5 → **10**; `..._MAX_CHUNKS_PER_POINTER` 3 → **5**; `..._TOTAL_TOKENS` 2.000 → **3.000**. `MAX_POINTERS` sigue en 3 | Los cinco casos están en rango ≤ 11 dentro del destino. Las tres perillas son la misma variable (profundidad); el tope de tokens sube para no convertir la profundidad en truncado |
| D2 | **Recalibrar la puerta** con 10 candidatos, misma regla que P5 (umbral más permisivo con precisión de lo admitido ≥ 0,48). Si hay pares sin juzgar, CC los exporta al workspace y **para**: los juzga el chat (modo B) | La calibración de P5 se hizo con 5 candidatos: con más, la curva cambia |
| D3 | Si el umbral recalibrado es **menor** que 0,3758, se usa el nuevo y se reporta qué deja fuera; no se mezclan umbrales | Una regla, un valor |

## 2. Preregistro corregido (aplica desde este brief)

**`R` cuenta lo ganado, no lo alcanzado:** positivos `explicit_document` `reached` por la variante
**y no** por `control` (control = defaults actuales, `clause_standard` con 5/3). `N = max(3,
⌈R/2⌉)`. Si `R < 3`, no se paga la pasada: porqué no por aspecto y parada.

Guardas (pareado `control` frente a `variant`, 29 consultas, 3 ejecuciones):

| # | Guarda | Umbral |
|---|---|---|
| 1 | Aspectos **ganados** resueltos por evidencia y aspecto (solo los no resueltos por control) | ≥ N, cada uno en ≥ 2/3 |
| 1b | Los ya resueltos por control (q12-a2, q15-g2, q26-g2) siguen resueltos | cada uno ≥ 2/3 |
| 2 | Recall sobre relevantes recuperadas | −0,03 como mucho |
| 3 | `verified` | 1,00 |
| 4 | Precisión sin lo añadido solo por grafo | −0,03 como mucho |
| 5 | Negativos (q10-a1, q27-g1, q28-g1) `not_followed` o nada citado | 100 % |

CC actualiza `graph_guards.py` con la definición de R y la guarda 1b (constante versionada, fuente
este brief) y deja el criterio del brief 3 reproducible tal cual.

## 3. Comprobación sin generación

Calibración (D2), L2 de la variante, `R` y `N`, tamaño de contexto (chunks y tokens mediana/p95)
control frente a variante, y digest de `RETRIEVAL_XREF=clause` igual al del brief 2.

## 4. Se reporta (no decide)

Latencia, coste, tokens de entrada, `not_found`, `delegated` por clase, citas fuera de su afirmación
desglosadas como en el brief 3 (en ejecuciones con y sin cambio de contexto).

## 5. UAT y cierre

Por API: q12 (turno 1) y q15. Por evidencia `graph_document`: origen, destino, distancia ≤ umbral.
Captura del render de q12 (traza). Registros `pending`; juzga el chat. Default: si pasan las guardas,
los valores de D1 pasan a default; si no, se quedan los actuales con el porqué no.
`pre-M6-decisions-4.md`, `pre-M6-uat-4.md`, docs y commit.

---

## 7. Decisión del chat tras la parada (2026-11-05)

Lectura de `pre-M6-decisions-4.md`: la parada fue correcta (R = 1 < 3). El fallo es de **D1**, no de
la técnica: subí los candidatos sin subir el tope de chunks, que era el vinculante. Se sigue con
una variante corregida, preregistrada aquí **antes** de generar.

| # | Decisión |
|---|---|
| E1 | Variante `deep` redefinida: **10 candidatos, 10 chunks por puntero, 3.000 tokens**, `MAX_POINTERS` 3. El tope de tokens se queda en 3.000: el contrafactual muestra que no es vinculante y no se abre más contexto del necesario |
| E2 | **Puerta 0,3991** (recalibrada con 10 candidatos y chunks abiertos): se confirma. "Una regla, un valor" aplica también cuando el umbral sube. Sin recalibrar otra vez |
| E3 | Guardas **v2** sin cambios. Con la comprobación sin generación de §5: R = 3 (q12-a1, q12-a4, q15-g1), **N = 3** → los tres ganados deben resolverse en ≥ 2/3. CC repite la comprobación con E1 antes de pagar; si R < 3, para |
| E4 | Riesgo aceptado y declarado: 2 de los 3 ganados son de la misma consulta (q12). Un pase de G1 no generaliza más allá de "remisiones a documento cuya respuesta está profunda en el destino" |
| E5 | Una pasada real, ≤ 1,00 USD (la autorización de este brief; no es segunda pasada: no se pagó ninguna). Reportar además el tamaño de contexto y la latencia frente a control |

UAT y cierre como en §5 (q12 turno 1 y q15 por API, captura de q12; juzga el chat). Si pasan las
guardas, E1 + E2 pasan a default; si no, se cierra con "no se adopta" y el porqué no por aspecto.
