# pre-M6 — Brief 2: acumulador de evidencia por cláusula (sin cambio de comportamiento)

> Rama `pre-m6-grafo`. Requisito previo: brief 0 aplicado (`CLAUDE.md` en modo autónomo) y el
> brief 1 commiteado. **Sin pasada de generación.** Llamadas al proveedor: solo embeddings de las
> 29 consultas del golden set para la comprobación de digest (despreciable).
>
> **Regla del brief:** refactor puro. Con cualquier configuración existente, el contexto que llega
> al proveedor es **idéntico byte a byte** al de antes. Si no se puede conseguir sin cambiar el
> orden, CC para y reporta la diferencia.

---

## 0. Contexto

La línea base del brief 1 (`pre-M6-decisions.md` §4) fija el orden de la sesión:

- **Brief 3 — punteros a documento:** 10 de 11 positivos `pointed_not_followed`. Mayor techo.
- **Brief 4 — aristas por nombre:** 3 positivos `pointed_not_resolved`.
- **Estructurales (DRD, jerarquía):** 6 de 6 ya `reached` sin arista nueva. **No se implementan**
  en esta sesión: sin señal. Aplicabilidad: 2 fallos (un tope, un enlace no traído), n = 3 —
  insuficiente para un brief; se registra como diferido.

Los briefs 3 y 4 añaden canales que pueden traer una cláusula que ya llegó por otro camino. Hoy cada
canal (retrieval, expansión, `xref`) deduplica y ordena a su manera. Antes de añadir canales hace
falta **un único punto** que decida qué entra, en qué orden y por qué. Este brief lo crea sin que
nada cambie.

## 1. Decisiones

| # | Decisión | Porqué |
|---|---|---|
| K1 | **Acumulador único por `ClauseKey`** `(canonical_doc_id, section_number)`. Todos los canales le entregan candidatos; él deduplica, ordena y asigna plazas. Los topes por canal actuales (`RETRIEVAL_XREF_MAX_CLAUSES`, tokens de expansión y `xref`) se respetan tal cual | Un solo lugar para las reglas que los briefs 3–4 tocarán |
| K2 | **Orden estable y explícito.** CC documenta el orden que hoy produce el sistema y lo codifica como regla declarativa (no como efecto del orden de llamadas ni de llegada). Nunca por orden de llegada: el orden fija la posición en el prompt | Reproducibilidad; prerequisito para medir cualquier cambio posterior |
| K3 | **Precedencia de `origin`** cuando una cláusula llega por dos caminos: gana el canal de mayor precedencia, hoy `retrieval > expansion > xref`; los demás se conservan en `origins`. Debe reproducir el resultado actual | La regla de autoridad del brief 3 se enchufa aquí |
| K4 | **`origin` y trail en el contrato**, campo aditivo por evidencia: `origin ∈ retrieval \| expansion \| xref`, `edge_kind` (para `xref`: `explicit_clause`), `from_clause` (la cláusula cuyo texto la nombró) y `hop`. `origins` con los caminos secundarios. La UI no cambia | Requisito heredado; sin él no se puede auditar por qué entró algo |
| K5 | **Presupuesto de saltos** `RETRIEVAL_GRAPH_MAX_HOPS = 1`, aplicado en el acumulador: lo que llega por un salto no dispara otro. Es el comportamiento actual, ahora explícito | L3: los 40 aspectos son de 1 salto. El 2 se plantea solo con casos |
| K6 | `corpus_version`: nada se persiste en este brief → no aplica. Anotarlo en decisions | Requisito heredado condicionado a persistencia |
| K7 | Las degradaciones (`xref_over_limit`, `xref_truncated`, …) se siguen emitiendo igual, ahora desde el acumulador | Contrato estable |

## 2. Comprobación (criterio de aceptación)

- **Digest del contexto del proveedor** para las 29 consultas del golden set v3 (todos los turnos),
  antes y después, con estas configuraciones: `defaults`; `xref=off`; `expansion=off`;
  `expansion=orphans`; `clause_standard` (apagado por defecto, debe seguir funcionando igual).
  **Todos idénticos.** Sin generación: el arnés de retrieval/contexto basta.
- `graph_reach rescore` sobre el artefacto nuevo da la misma tabla L2 que el brief 1.
- `uv run pytest` en verde; tests nuevos: dedupe por `ClauseKey`, precedencia K3, orden K2 estable
  ante permutación de la entrada, `origin`/`from_clause`/`hop` en el contrato, K5.

## 3. UAT

No hay cambio visible. UAT mínima por API: una consulta con `xref` (q12 turno 1) devuelve las
evidencias con `origin`, `from_clause` y `hop` coherentes. Registro en `records.jsonl` con
`verdict = pending`; lo juzga el chat.

## 4. Cierre

`pre-M6-decisions-2.md` con la regla de orden documentada (K2), los digests y cualquier diferencia
encontrada. Docs al día y commit, según el brief 0.

## 5. Antes de empezar: cerrar L1 del brief 1

Los 229 pares ya están juzgados en modo B por una sesión ciega:
`data/groundtruth/retrieval/annotation/v3-ws/pair_judgments_20261031-165719.json` (15 relevantes,
214 no). Impórtalos (`golden pairs`, `RUNBOOK.md` §13) y repite el harness `defaults` hasta VALID.
Si sale INVALID otra vez, exporta los pares nuevos y para. Las tablas L1 válidas van a
`pre-M6-decisions-2.md` §0 (y la línea de `CLAUDE.md` que dice INVALID se actualiza).
