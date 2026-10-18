# S10 · Brief 1 — UAT

> **Por qué es tuya:** caso 1 de la frontera (no hay oráculo: el insumo es un anotador leyendo el
> corpus) y caso 2 (aprobar las consultas y resolver la revisión son decisiones tuyas). Todo lo
> demás —export, arnés, verificaciones— lo ejecutó CC; resultados en `S10-decisions.md`.

Ejecutar desde `C:\Users\salca\proyectos\RAG-ECSS`, en PowerShell, con Docker levantado
(`docker compose up -d`).

---

## 1. Preparar el workspace de anotación (fuera del repo)

Fuera del repo para que la sesión de anotación no cargue `.claude/CLAUDE.md`, que describe el
retriever (`S10-decisions.md` §2).

```powershell
Set-Location C:\Users\salca\proyectos\RAG-ECSS
$ws = "C:\Users\salca\proyectos\ecss-annotation"
New-Item -ItemType Directory -Force $ws | Out-Null
uv run python -m corpus.groundtruth.retrieval.clauses --out $ws
Copy-Item corpus\groundtruth\retrieval\protocol.md $ws
code $ws
```

**Qué mirar:** el export dice `4433 clauses, 26 documents` e `input_fingerprint: ab3fce8b5cb1a0d4`;
el workspace contiene solo `clauses\`, `INDEX.md` y `protocol.md`.

## 2. Fase 1 — las 10 consultas

En la ventana de VS Code que se abre sobre `ecss-annotation`, abre una sesión **nueva** de Claude
Code con **Opus** y pega:

```
Read protocol.md in this directory and follow it in Mode A, Phase 1 only. The document index is
INDEX.md; do not open clauses/ yet. Write the 10 queries to annotation.json and stop.
```

**Qué mirar:** 10 consultas con la estratificación de la tabla del protocolo, realistas, en inglés,
sin estándares que no estén en `INDEX.md`, y una `multi_turn` cuyo último turno cambie de tema o
sea elíptico. Edítalas o pide cambios hasta aprobarlas.

## 3. Fase 2 — cláusulas relevantes

En **la misma sesión**:

```
The queries are approved. Run Mode A, Phase 2 of protocol.md and append the judgments to
annotation.json.
```

**Qué mirar:** que cada consulta tenga su objeto `judgments` con `search_notes`. No hace falta que
compruebes las citas: lo hace el importador.

## 4. Importar y resolver la revisión

```powershell
Set-Location C:\Users\salca\proyectos\RAG-ECSS
uv run python -m corpus.groundtruth.retrieval.golden import C:\Users\salca\proyectos\ecss-annotation\annotation.json
uv run python -m corpus.groundtruth.retrieval.golden review
```

**Qué mirar:** si sale `rejected:`, la lista dice cada motivo — devuélvesela a la sesión de
anotación para que corrija `annotation.json` y repite el import. Si entra, `review` lista lo que
pide ojo humano (cita que solo casa sin puntuación, cláusula con número repetido). Para cada ítem:
¿la cita existe en esa cláusula y responde a la pregunta? Esto lo resolvemos juntos con CC.

## 5. Medir, y completar los juicios que falten

A partir de aquí puede seguir CC (avísale):

```powershell
uv run python -m corpus.groundtruth.retrieval.harness
```

Sale con `INVALID` mientras haya cláusulas entregadas sin juicio (en el humo, ~109 pares). Para
cada `pairs_<stamp>.json` que exporte:

```powershell
Copy-Item data\groundtruth\retrieval\annotation\pairs_<stamp>.json C:\Users\salca\proyectos\ecss-annotation
```

y en una sesión **nueva** de Claude Code (Opus) sobre el workspace:

```
Read protocol.md and follow Mode B for pairs_<stamp>.json. Write pair_judgments_<stamp>.json.
```

```powershell
uv run python -m corpus.groundtruth.retrieval.golden pairs C:\Users\salca\proyectos\ecss-annotation\pair_judgments_<stamp>.json
uv run python -m corpus.groundtruth.retrieval.harness
```

hasta que las dos configuraciones salgan `VALID`.

## 6. Leer la tabla

**Qué mirar:** la tabla estrato × configuración del informe, no la media. Con 1–2 consultas por
estrato es indicativa. Lo que se espera ver, por lo que ya midieron las verificaciones: `hybrid`
mejor en `names_standard` solo si las cláusulas que trae son del estándar (no citas a él), y peor
en `generic`.

## Opcional — drift check

Otro workspace igual (`ecss-annotation-2`, mismos comandos del paso 1), una sesión nueva que hace
la fase 2 de **2** de las consultas aprobadas sin ver la primera (le pasas un `annotation.json` con
las 10 consultas y sin `judgments`), y:

```powershell
uv run python -m corpus.groundtruth.retrieval.golden agreement C:\Users\salca\proyectos\ecss-annotation-2\annotation.json
```
