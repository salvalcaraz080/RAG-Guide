# pre-M6 — UAT del brief 3

Por API contra el stack recreado con los defaults nuevos (`RETRIEVAL_XREF=clause_standard`,
`RETRIEVAL_DOC_POINTER_MAX_DISTANCE=0.3758`; comprobado con `printenv` dentro del contenedor).
Respuestas en `data/groundtruth/generation/runs/uat_pre-m6_3_20261102.json`; registros
`pre-M6-3-q12-pointer` y `pre-M6-3-q14-pointer`, `verdict = pending` (`records check`: 23 válidos).
**Juzga el chat.** Coste: 0,0104 USD por API y 0,0028 USD de la consulta de la captura.

## Caso 1: q12 (turno único)

*How does Q-ST-80C expect software-related risks to be identified and handled, and how are they
supposed to feed into the project's overall risk management process?*

Respuesta `partial`, `claude-haiku-5-5`, 53 chunks (50 en el brief 2):

| `origin` | `edge_kind` | `hop` | Chunks |
|---|---|---|---|
| `retrieval` | — | 0 | 10 |
| `expansion` | — | 0 | 26 |
| `xref` | `explicit_clause` | 1 | 14 |
| `graph_document` | `explicit_document` | 1 | 3 |

Por evidencia `graph_document`:

| Evidencia | `from_clause` | ¿El origen nombra el destino? | Distancia | ¿≤ 0,3758? | Citada |
|---|---|---|---|---|---|
| M-ST-80C § E.3 | Q-ST-80C § 5.3.1 | sí («as specified in ECSS-M-ST-80») | 0,2983 | sí | sí |
| M-ST-80C § 6.3 | Q-ST-80C § 5.3.1 | sí | 0,3435 | sí | sí |
| M-ST-80C § 6.1 | Q-ST-80C § 5.3.1 | sí | 0,3495 | sí | sí |

**Comprobado por CC:** `from_clause` está en contexto como texto `hop = 0` y su texto contiene un
puntero que resuelve a M-ST-80C; el destino es el documento nombrado; las tres distancias quedan
bajo el umbral. Las tres cláusulas salen citadas. Degradaciones: `expansion_truncated` y
`xref_truncated`, ninguna `doc_pointer_over_limit`.

**Render** (Playwright, FRONTEND_ENV=development), en `data/groundtruth/uat/screens/`, no commiteado:
- `pre-M6-3-q12-answer.png`: la respuesta completa.
- `pre-M6-3-q12-trace.png`: las tres evidencias desplegadas. Aparecen como `[21]`–`[23]` en
  «Cited» con «➕ by cross-reference», cada átomo con «normative · section · added by
  cross-reference» y su distancia (modo dev). El panel no se rompe.

**Para el juicio:** la UI no distingue un puntero a documento de una referencia a cláusula: las dos
se marcan «by cross-reference». Es coherente con la decisión de pintar `origin` agrupado, pero el
usuario no ve que lo trajo la puerta por relevancia y no una remisión literal a esa cláusula.

## Caso 2: q14 (turno 3)

*And if the spacecraft is also supposed to run on-board control procedures, what does the flight
software have to provide to execute them?* (Los turnos 1–2 se reprodujeron en la misma sesión.)

Respuesta `partial`, 52 chunks: `retrieval` 10, `expansion` 1, `xref` 41. **Ninguna evidencia
`graph_document`.** El log del API para esta petición dice `doc_pointers_followed pointers=1
followed=0`: el puntero a E-ST-70-01C se lee y la puerta lo para. Coincide con L2 (`gated`) y con
el arnés: la cláusula respuesta §5.4.4.2 está a 0,4618, sobre el umbral.

**Comprobado por CC:** no hay evidencia del canal y, por tanto, nada que verificar por evidencia. El
caso muestra la puerta cerrando. La respuesta sigue declarando el hueco de E-ST-70-01C como
`delegated` (`in_corpus_not_injected`), igual que con `gaps`.

**Para el juicio:** el brief esperaba que q14 bajara su `in_corpus_not_injected`. No baja, por una
decisión de diseño (puerta absoluta) y no por un fallo. Ver `pre-M6-decisions-3.md` §7.

## Veredicto (chat, 2026-11-02)

- `pre-M6-3-q12-pointer` → **ok**, `judge: chat`. Las tres evidencias `graph_document` (M-ST-80C
  §E.3, §6.3, §6.1) son contenido real de gestión de riesgos al que remite Q-ST-80C §5.3.1, bajo el
  umbral, y salen citadas. Render (captura `pre-M6-3-q12-trace.png`) correcto: el panel no se rompe.
  Observación de UI para la sesión de UI/UX, no fallo: «by cross-reference» no distingue la remisión
  a documento (seleccionada por relevancia) de la remisión literal a cláusula; conviene mostrar el
  `from_clause` («via Q-ST-80C 5.3.1 → ECSS-M-ST-80»).
- `pre-M6-3-q14-pointer` → **ok**, `judge: chat`. La puerta cierra como está diseñada y el hueco se
  declara `delegated`, que es la respuesta honesta. Que q14-a1 no se resuelva es un límite del
  diseño (puerta absoluta), registrado como diferido, no un fallo de la UAT.

## Decisión sobre el default (chat)

**Se confirma `RETRIEVAL_XREF=clause_standard` por defecto.** Las reservas de `pre-M6-decisions-3.md`
§5 son ciertas, pero no cambian la decisión:
1. La ganancia real es 2 de 11, no 3. Es un fallo **del preregistro** (contaba lo alcanzado, no lo
   ganado): permitió pagar la pasada con un techo menor del previsto. No respalda adoptar; lo que
   respalda adoptar es que lo servido no empeora (recall 0,86 = 0,86, precisión 0,85 → 0,87 sin lo
   añadido, `verified` 1,00, negativos 100 %), con coste y latencia en el ruido. Por error
   asimétrico, dos aspectos ganados sin pérdida medible bastan. La regla se corrige desde el brief 4.
2. Las citas fuera de su afirmación: 25 de 33 en ejecuciones con contexto idéntico al de control →
   varianza del generador, no efecto del canal. Sigue como diferido de generación (ya lo era en S14).
