# pre-M6 — UAT del brief 2

Sin cambio visible. Una consulta por API contra el stack reconstruido, con los defaults. Respuesta
guardada en `data/groundtruth/generation/runs/uat_pre-m6_2_20261101.json`; registro `pre-M6-2-trail`,
`verdict = pending`. **Juzga el chat.**

## Caso: q12 (turno único)

*How does Q-ST-80C expect software-related risks to be identified and handled, and how are they
supposed to feed into the project's overall risk management process?*

Respuesta `partial`, `claude-haiku-5-5`, 0,0035 USD, 50 chunks:

| `origin` | `edge_kind` | `hop` | Chunks |
|---|---|---|---|
| `retrieval` | — | 0 | 10 |
| `expansion` | — | 0 | 26 |
| `xref` | `explicit_clause` | 1 | 14 |

Las 14 evidencias `xref` vienen de dos cláusulas, ambas en contexto y con la referencia en su texto:

| `from_clause` | Referencia en su texto | Cláusulas traídas |
|---|---|---|
| Q-ST-80C § 6.2.2.4 | `clause 6.2.3` | 6.2.3.2 a 6.2.3.8 |
| Q-ST-80C § 6.2.9.4 | `clause 6.2.10` | 6.2.10.1, .3 y .4 |

`origins`: 26 evidencias de expansión con `retrieval` y 5 recuperadas con `expansion`, las de cláusulas
completadas. El resto, vacío.

**Comprobado por CC:** campos presentes, `from_clause` coherente en las 14 y registro válido
(`records check`: 21). `retrieved_context` sale agrupado por cláusula, así que el orden plano de K2 no
se ve aquí: lo cubren los digests.

La UI no se capturó: no cambia y sus decisiones sobre `xref` están en test.

## Veredicto (chat, 2026-11-01)

`pre-M6-2-trail` → **ok**, `judge: chat`. Los 14 `xref` tienen `from_clause` cuyo texto contiene la
referencia, `hop` 1, `edge_kind` correcto; la expansión va con `hop` 0 sin `from_clause`; `origins`
coherente con K3. Observación para el brief 3 (no es fallo): 50 chunks para q12, 14 de ellos por dos
remisiones genéricas a cláusula padre ("clause 6.2.3"). El contexto ya es grande antes de seguir
punteros a documento: la puerta de relevancia del brief 3 no es opcional.
