# pre-M6 — Brief 0: modo autónomo de Claude Code

> Brief de proceso, sin cambio de producto. Se ejecuta **antes** de la fase B del brief 1, en la
> rama `pre-m6-grafo`. Decidido por Salva el 2026-10-31.

## Objetivo

Salva tiene poco tiempo. CC pasa a ejecutar cada brief de principio a fin. Actualiza `CLAUDE.md`
(y lo que toque de `RUNBOOK.md`) con estas reglas, que sustituyen a las de alcance actuales
("solo código + `uv run pytest` mockeado; Docker y UAT los hace Salva; los commits los hace Salva").

## Reglas nuevas

| # | Regla |
|---|---|
| A1 | **Alcance:** implementación, tests mockeados, Docker (levantar, ingestar, recrear contenedores), arneses contra la base, UAT y docs. |
| A2 | **Llamadas reales:** una pasada por brief sin preguntar, con el coste que declare el brief. Una **segunda** pasada requiere aprobación de Salva: CC para y lo reporta con coste y porqué. "Comprobación sin generación antes de pagar la pasada" sigue vigente. |
| A3 | **UAT:** CC la ejecuta (por API contra el stack levantado) y escribe `SNN-uat*.md` y los registros en `corpus/groundtruth/uat/records.jsonl` con `verdict = pending`. Cuando el brief cambia la UI, CC hace **capturas del render con Playwright** (Chromium ya instalado) y las referencia en `SNN-uat*.md`; se guardan en `data/groundtruth/uat/screens/` (no commiteadas). El juicio lo pone el chat. |
| A4 | **Commits:** uno por brief en la rama de la sesión (mensaje convencional, en español, como hasta ahora). **Nunca** merge a `main`, push forzado ni reescritura de historia: el merge lo hace Salva. |
| A5 | **Paradas obligatorias:** segunda pasada real; guarda preregistrada que falla (no se itera: se reporta el porqué de cada caso); borrar datos o la base; tocar `corpus/` fuera de `corpus/groundtruth/`; decisión de producto no cubierta por el brief. |
| A6 | **Arranque:** cuando Salva diga "siguiente brief" (o nombre uno), CC busca en `C:\Users\salca\proyectos\RAG-Guide\sessions\` el brief más reciente sin `decisions` correspondiente y lo ejecuta entero. |
| A7 | **Cierre de brief:** `SNN-decisions*.md` y `SNN-uat*.md` en la carpeta de la sesión, `CHANGELOG.md`/`CLAUDE.md`/`ARCHITECTURE.md` al día, commit, y un resumen final de ≤ 10 líneas para Salva: qué se hizo, coste real, qué queda parado y por qué. |

## Cambio en el brief 1, fase B

La anotación v3 la hace Claude Chat con un subagente aislado (no Salva). El fichero llega a
`data/groundtruth/retrieval/annotation/v3-ws/annotation_v3.json`. CC ejecuta el resto del §13 de
`RUNBOOK.md` (import, `golden review`, harness `defaults`, `graph_reach run`) y produce L1–L3. Si el
harness sale `INVALID`, CC exporta los pares sin juzgar al mismo workspace y **para**: el chat los
juzga (modo B) con otro subagente y CC repite.

## Tests

Ninguno nuevo. `uv run pytest` en verde antes del commit.

## Notas del anotador para la fase B (2026-10-31)

- `annotation_v3.json` entregado: 15 consultas (q15–q29), sin sustituciones, 233 citas verificadas
  por el anotador contra el export (el importador las vuelve a comprobar).
- **Q-ST-80C 6.2.3.1 no está en el export** (q23 usa la hermana 6.2.3.2). Comprueba si es un hueco
  del parse o una cláusula no citable; repórtalo en decisions, **sin** tocar `corpus/`.
- q22: Q-ST-80C remite a "ECSS-Q-ST-20 clause 5.4.1", que en el export solo existe como
  5.4.1.1–5.4.1.3. Útil como caso de resolución de puntero a cláusula padre ausente en L2.
- q20: E-ST-40C R.2 y Q-ST-80C D.2 discrepan sobre pruebas unitarias en categoría D; ambos juzgados
  relevantes. No es un error del golden set.
