# S10 · Brief 2 — UAT

> **Por qué es tuya:** caso 1 de la frontera (juzgar relevancia no tiene oráculo) y caso 2 (qué
> configuración pasa a default lo decide Claude Chat con la tabla). Código, migración, estadísticas
> y mediciones los ejecutó CC.

## 1. Modo B sobre la exportación agrupada

Las 11 configuraciones medidas dejaron **24 pares** sin juzgar, en un solo fichero. CC ya lo copió
al workspace: `C:\Users\salca\proyectos\ecss-annotation\pairs_20261012-214119.json`.

Abre una sesión **nueva** de Claude Code (Opus) en la ventana de VS Code sobre `ecss-annotation`
y pega:

```
Read protocol.md and follow Mode B for pairs_20261012-214119.json. Write pair_judgments_20261012-214119.json.
```

Cuando termine:

```powershell
Set-Location C:\Users\salca\proyectos\RAG-ECSS
uv run python -m corpus.groundtruth.retrieval.golden pairs C:\Users\salca\proyectos\ecss-annotation\pair_judgments_20261012-214119.json
```

**Qué mirar:** `imported 24 judgments` y `review list: 0`. Si sale `rejected:`, devuelve los motivos
a la sesión de anotación. Después avisa a CC, que vuelve a medir hasta que todo salga `VALID`.

## 2. Leer la tabla y la recomendación

En `S10-decisions-2.md`, cuando CC la complete: la tabla por estrato y la recomendación (no
aplicada) de qué configuración pasaría a default.
