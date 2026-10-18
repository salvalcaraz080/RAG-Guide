# S10 · Brief 3 — UAT

> **Por qué es tuya:** caso 1 de la frontera (las consultas nuevas y sus relevantes no tienen
> oráculo) y caso 2 (aprobar las consultas, leer las tablas, juzgar si una respuesta se lee mejor).
> Defaults, curva de profundidad, expansión y su medición los ejecutó CC.

Todo en una **sola sesión de anotación** (Opus, nueva) sobre el workspace de siempre, con una
pausa en medio para que CC mida.

## 1. Fase 1 de ampliación — 4 consultas nuevas

CC ya dejó en `C:\Users\salca\proyectos\ecss-annotation` el `protocol.md` **1.1** y
`existing_queries.json` (las 10 consultas actuales, sin juicios). Abre una sesión **nueva** de
Claude Code (Opus) en la ventana de VS Code sobre `ecss-annotation` y pega:

```
Read protocol.md (version 1.1) and run the Extension of Mode A, Phase 1 only. The existing queries are in existing_queries.json; the document index is INDEX.md; do not open clauses/ yet. Write only the 4 new queries, ids q11 to q14, to annotation_v2.json and stop.
```

**Qué mirar:** exactamente 4 consultas: **2 `names_standard_xref`** (nombran un estándar y su
respuesta necesita otro al que remite), **1 `paraphrase`** y **1 `multi_turn`**. Que no repitan
el tema de las 10 existentes. Edítalas o pide cambios hasta aprobarlas.

## 2. Fase 2 sobre las nuevas

En la **misma sesión**:

```
The new queries are approved. Run Phase 2 of protocol.md for q11 to q14 only and append their judgments to annotation_v2.json.
```

Cuando termine:

```powershell
Set-Location C:\Users\salca\proyectos\RAG-ECSS
uv run python -m corpus.groundtruth.retrieval.golden extend C:\Users\salca\proyectos\ecss-annotation\annotation_v2.json
```

**Qué mirar:** `imported N judgments` y `review list: 0`. Si sale `rejected:`, devuelve los motivos
a la sesión. **No cierres la sesión de anotación:** avisa a CC, que mide y deja el fichero de pares.

## 3. Modo B, en la misma sesión

CC te dará el nombre del fichero (`pairs_<stamp>.json`, ya copiado al workspace). Pega:

```
Now follow Mode B of protocol.md for pairs_<stamp>.json. Write pair_judgments_<stamp>.json.
```

```powershell
Set-Location C:\Users\salca\proyectos\RAG-ECSS
uv run python -m corpus.groundtruth.retrieval.golden pairs C:\Users\salca\proyectos\ecss-annotation\pair_judgments_<stamp>.json
```

## 4. Leer

En `S10-decisions-3.md`, cuando CC la complete: la curva de profundidad (§2), la tabla v2 filtro
frente a cuota (§3) y la tabla de expansión (§4), con la recomendación.

## 5. Una respuesta real con expansión, en Streamlit

**Pregunta** (q02 del golden set): sin expansión, de `ECSS-Q-ST-80C §6.2.5.4` llega solo la NOTE «See
also software problem reporting described in clause 5.2.5.», sin su requisito («Process metrics shall
be used within the supplier's organization and reported to the customer…»).

```
What does Q-ST-80 require for handling software problem reports and nonconformances?
```

Cada vuelta cambia `RETRIEVAL_EXPANSION` en `.env` (sin BOM: `Set-Content` de PowerShell 5.1 lo
añadiría y rompería la primera variable) y recrea la API para que lo lea. Gasta una llamada al LLM
por pregunta (céntimos).

```powershell
Set-Location C:\Users\salca\proyectos\RAG-ECSS
function Set-Expansion($mode) {
    $p = "C:\Users\salca\proyectos\RAG-ECSS\.env"
    $t = [IO.File]::ReadAllText($p) -replace '(?m)^RETRIEVAL_EXPANSION=.*$', "RETRIEVAL_EXPANSION=$mode"
    [IO.File]::WriteAllText($p, $t)
    docker compose up -d --force-recreate api
}
Set-Expansion orphans
```

Abre http://localhost:8501, **New conversation**, haz la pregunta y abre la fuente de
`ECSS-Q-ST-80C · clause 6.2.5.4` si la cita (o en lo recuperado sin citar). Después:

```powershell
Set-Expansion bounded
```
Respuesta del LLM:
```
[...]
Documentation and Reporting

"The supplier shall ensure that nonconformances and software problem reports detected during testing are properly documented and reported to those concerned" [1]. Furthermore, "any discovered problems shall be documented in nonconformance reports" [3].
[...]
```

Expandible ECSS-Q-ST-80C · clause 6.2.5.4 (retrieved but not cited):
```
Brought by retrieval and shown to the model; the answer does not use them.

📄 ECSS-Q-ST-80C Rev.2 · clause 6.2.5.4

ℹ️ informative · note

See also software problem reporting described in clause 5.2.5.

distance: 0.3606903359293143 · oversized: False

⚖️ normative · requirement · ➕ added to complete the clause

Requirement: ECSS-Q-ST-80_0720116

Process metrics shall be used within the supplier’s organization and reported to the customer, including: number of problems detected during verification; number of problems detected during integration and validation testing and use. Expected output: Software product assurance reports [PAF, -; -]. Software product assurance reports [PAF, -; -].
```

**New conversation** y la misma pregunta. Para comparar con la respuesta sin expansión, `off`:

```powershell
Set-Expansion off
```
Respuesta del LLM:
```
[...]
Documentation and Reporting

"The supplier shall ensure that nonconformances and software problem reports detected during testing are properly documented and reported to those concerned." [1] Additionally, "any discovered problems shall be documented in nonconformance reports." [3]
[...]
```

Expandible ECSS-Q-ST-80C · clause 6.2.5.4 (retrieved but not cited):
```
Brought by retrieval and shown to the model; the answer does not use them.

📄 ECSS-Q-ST-80C Rev.2 · clause 6.2.5.4

ℹ️ informative · note

See also software problem reporting described in clause 5.2.5.
```

**New conversation**, la misma pregunta, y al terminar deja el default (`orphans` desde el brief 4):

```powershell
Set-Expansion orphans
```
Respuesta del LLM:
```
[...]
Documentation and Reporting

"The supplier shall ensure that nonconformances and software problem reports detected during testing are properly documented and reported to those concerned." [1] Additionally, "Any discovered problems shall be documented in nonconformance reports." [3]
[...]
```

Expandible ECSS-Q-ST-80C · clause 6.2.5.4 (retrieved but not cited):
```
Brought by retrieval and shown to the model; the answer does not use them.

📄 ECSS-Q-ST-80C Rev.2 · clause 6.2.5.4

ℹ️ informative · note

See also software problem reporting described in clause 5.2.5.

distance: 0.3606903359293143 · oversized: False

⚖️ normative · requirement · ➕ added to complete the clause

Requirement: ECSS-Q-ST-80_0720116

Process metrics shall be used within the supplier’s organization and reported to the customer, including: number of problems detected during verification; number of problems detected during integration and validation testing and use. Expected output: Software product assurance reports [PAF, -; -]. Software product assurance reports [PAF, -; -].
```

**Qué mirar:** con `orphans`, el requisito de §6.2.5.4 aparece en la fuente marcado `➕ added to
complete the clause`; con `bounded`, más cláusulas completas (más ➕). ¿Se lee mejor la respuesta
que sin expansión? ¿La traza deja claro qué se añadió y qué encontró la búsqueda?

**Intepretación**: en los 3 casos la respuesta del LLM fue la misma, un resultado sorprendemente determinista. Sin embargo, sí es verdad que al ir a consultar las fuentes, es deseable que la cláusula a la que hace referencia la note, aparezca para ser leída. Tanto en bounded como en orphans la respuesta fue idéntica. En las 3, la cláusula aparece también como retrieved but not used by the model
