# S10 · Brief 4 — UAT

> **Por qué es tuya:** caso 2 (coste real: autorizaste las 14 llamadas del TTFT, ya hechas) y caso 1
> (los pares que el reranker trajo sin juicio no tienen oráculo). Wrapper, configuraciones,
> latencias, truncado y separabilidad los midió CC.

## 1. TTFT de referencia — hecho

14 llamadas a `/query/stream` con los defaults: **mediana 1,76 s**, p95 3,19 s
(`scripts/output_ttft_baseline.txt`).

## 2. Modo B sobre la exportación agrupada

Las 6 configuraciones `rerankN_*` dejaron **82 pares** sin juzgar en un solo fichero, ya copiado al
workspace: `pairs_20261013-094955.json`. Abre una sesión **nueva** de Claude Code (Opus) en la
ventana de VS Code sobre `ecss-annotation` y pega:

```
Read protocol.md (version 1.1) and follow Mode B for pairs_20261013-094955.json. Write pair_judgments_20261013-094955.json.
```

Cuando termine:

```powershell
Set-Location C:\Users\salca\proyectos\RAG-ECSS
uv run python -m corpus.groundtruth.retrieval.golden pairs C:\Users\salca\proyectos\ecss-annotation\pair_judgments_20261013-094955.json
```

**Qué mirar:** `imported 82 judgments` y `review list: 0`. Avisa a CC: vuelve a medir (unos 40 min:
el reranker de pool 400 tarda ~7 s por llamada en CPU) hasta que todo salga `VALID`.

## 3. Leer

`S10-decisions-4.md`, cuando CC la complete: la tabla por estrato, las latencias frente al TTFT, el
truncado y la separabilidad del score, con la recomendación.

## 4. Pendiente del brief 3

La UAT de Streamlit de `S10-uat-3.md` §5 (`orphans` / `bounded`). Hazla **después** de la medición
del reranker: recrear la API y lanzar consultas mete ruido en sus latencias. Ojo: el default ya es
`orphans`, así que la comparación útil es `off` frente a `orphans` frente a `bounded`.
