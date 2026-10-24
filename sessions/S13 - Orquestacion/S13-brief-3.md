# S13 — Brief 3 (mini): traza de punteros deduplicada por documento

> Mini-brief de cierre. Rama `s13-orquestacion-agentes`. Los commits los hace Salva. **Sin llamadas
> reales.** Solo traza: no cambia el contexto, el prompt ni ningún default.

## 0. Contexto

Decisión tras el brief 2: `clause_standard` **no se adopta** (falla la guarda 1, 1 de 6; sin
excepción de producto). Queda implementado y apagado. La traza de punteros será el instrumento del
grafo de referencias, y en la UAT 2 (caso 3) registró más de 60 entradas `already_searched` de
«ECSS-Q-ST-80», una por mención.

## 1. Decisiones

| # | Decisión | Porqué |
|---|---|---|
| T1 | En `RetrievedContext.xref` y `references_expanded`, las entradas **sin efecto en el contexto** (`already_searched`, `out_of_corpus`, `ambiguous`, y `whole_standard` con `clause`) se agrupan **por documento** con un contador `mentions` y la primera `source` | Una entrada por mención no aporta información y entierra la que sí la aporta |
| T2 | Las entradas que **sí** deciden (`followed`, `over_limit`, `truncated`, `already_injected`, `no_hits` y todas las de cláusula) no cambian de forma | El orden de aparición y las plazas se leen de ellas; agruparlas perdería por qué un puntero quedó fuera |
| T3 | El contexto es **idéntico byte a byte** en `off`, `clause` y `clause_standard` | Solo traza |

Si el arnés o `aspects.py` leen las entradas agrupadas (motivos de "por qué no"), que sigan
puntuando igual sobre `generation_20261023-191136`: re-puntúa y compara.

## 2. Docs

`ARCHITECTURE.md` (expansión por referencia): `clause_standard` implementado y **no adoptado**, con
el resultado del brief 2 en una línea y la referencia a `S13-decisions-2.md`; la forma de la traza.
`CHANGELOG.md` derivado.

## 3. Tests (sin red)

- `uv run pytest` en verde, recuento frente a los **1470**.
- Varias menciones del mismo documento → una entrada con `mentions`.
- T2: las entradas que deciden no cambian.
- T3: digests del contexto en los tres modos.
- Re-puntuación de `aspects.py` sobre el artefacto del brief 2 sin cambios en las cifras.

## 4. Cierre

`S13-decisions-3.md`. Sin UAT de usuario: CC comprueba la traza de la consulta del caso 3 de la
UAT 2 en un log y lo anota en el mismo fichero.
