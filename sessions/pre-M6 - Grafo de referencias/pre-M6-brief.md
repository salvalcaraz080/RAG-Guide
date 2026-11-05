# pre-M6 — Brief 1: instrumento del grafo (golden set 1.2 y `delegated` → corpus)

> Primer brief de la sesión del grafo de referencias (`pre-M6-resumen.md`). Rama nueva
> `pre-m6-grafo` desde `main`, **después** de que Salva mergee `s14-sistemas-multi-agente`.
> Los commits los hace Salva.
>
> **Regla del brief:** es el **instrumento**, no la técnica. No se toca el camino de `/query` ni
> `app/`. `corpus/` sigue congelado salvo `corpus/groundtruth/` (editable, como siempre).
> **Llamadas al proveedor:** solo los embeddings de las consultas nuevas en la línea base
> (coste despreciable, < 0,01 USD, autorizado). **Sin pasada de generación.** La anotación la
> ejecuta Salva en una sesión aparte con su suscripción, no CC.

---

## 0. Contexto y objetivo

Los briefs 3–5 añaden tres tipos de arista al retrieval (punteros a documento, estructurales, por
nombre). Sin un instrumento que mida **por tipo de arista**, cualquiera "parece ayudar", y el de S13
(`aspects.py`) solo cubre aristas explícitas con 14 aspectos de un diagnóstico antiguo.

Objetivo: que antes de tocar el retrieval exista (a) un golden set con consultas cuya respuesta
**exige un salto** de cada tipo, con el camino anotado; (b) el mapa de línea base de por qué hoy no
se alcanza cada respuesta; (c) la métrica de `delegated` hacia el corpus; y (d) las guardas comunes
de los briefs 3–5, preregistradas aquí.

Fases: **A** (CC) prepara protocolo, export e importador → **Salva** ejecuta la anotación →
**B** (CC) importa, valida y produce la línea base. Si la anotación no llega, CC entrega A y C y
para.

---

## 1. Fase A — protocolo 1.2 y targets de grafo

| # | Decisión | Porqué |
|---|---|---|
| G1 | **Protocolo 1.2, golden set v3**, modo extensión: solo consultas nuevas; v1/v2 intactas. El criterio de relevancia **no cambia**; lo que se añade es el **camino**. Registra en la cabecera del protocolo qué cambia y por qué | Las mediciones previas conservan su significado |
| G2 | **Estratos nuevos** (consultas a escribir por el anotador): `graph_document_pointer` 2 (los 2 `names_standard_xref` de v2 cuentan para el estrato), `graph_drd` 3, `graph_applicability` 2, `graph_hierarchy` 2, `graph_by_name` 3, `graph_negative` 3 → **15 consultas** | ≥ 3 positivas por tipo de brief (3: puntero; 4: DRD + aplicabilidad + jerarquía; 5: nombre). Las negativas miden que la puerta no siga lo que no aporta |
| G3 | Cada consulta positiva lleva **≥ 1 aspecto de grafo**: `answer_clause` (cláusula con la respuesta), `from_clause` (la que remite o la estructura que la une), `edge_kind ∈ explicit_clause \| explicit_document \| drd \| applicability \| hierarchy \| by_name`, `hops` (1 o 2), `evidence_quote` literal en `answer_clause` y `link_quote` literal en `from_clause` que muestra la arista | La regla de acierto casa por evidencia **y** por aspecto (S14). `hops` decide si el segundo salto tiene casos |
| G4 | Negativas: tres formas, una por consulta — remisión genérica de cortesía ("in accordance with ECSS-S-ST-00"), destino fuera del corpus, nombre ambiguo. Aspecto con `edge_kind`, `from_clause` y `expected: not_followed` | Precisión de la puerta, medible sin generar |
| G5 | El juicio binario por cláusula de las consultas nuevas sigue el protocolo 1.1 sin cambios (corpus entero, cita literal) | Mismo yardstick para recall/precisión |
| G6 | **Targets de grafo** en un fichero propio versionado (`graph_targets.json` junto a `golden_set.json`), que reúne: los aspectos nuevos de G3/G4 y los **14 aspectos existentes** de `aspects.TARGETS` reetiquetados (`xref` → `explicit_clause`, `standard` → `explicit_document`, `same_clause` → `explicit_clause` con `table: true`), con las correcciones de `CORRECTED_ANSWERS`. q10-a1 entra como negativo (ya marcado espurio). El "Space-to-Ground ICD" tipado `delegated` en la pasada de S14 entra como negativo **solo si** el anotador encuentra una consulta que lo reproduzca; si no, se reporta | No se re-anota lo ya diagnosticado; un único sitio para todos los targets de grafo |
| G7 | El export de cláusulas para el anotador incluye por cláusula sus **"Expected output"** y sus filas de matriz de aplicabilidad si el parse las tiene; si no las tiene, reporta qué falta y **no** se extrae nada nuevo de `corpus/` | El anotador necesita ver las aristas (2) para anotarlas; si el parse no las expone, eso ya es hallazgo para el brief 4 |
| G8 | Importador con Pydantic: rechaza el fichero entero si una cita (`evidence_quote` o `link_quote`) no está en su cláusula, si un `answer_clause`/`from_clause` no existe, o si `edge_kind` no es válido | El golden set es contrato |

Procedimiento del anotador: el de `RUNBOOK.md` §13 (workspace fuera del repo, Opus, sin ver el
retriever). Actualiza §13 con el paso a paso de 1.2 para Salva.

---

## 2. Fase B — línea base sin generación (tras importar)

| # | Decisión | Porqué |
|---|---|---|
| L1 | Tabla del arnés de retrieval actual (`defaults`) sobre v3: por estrato, la tabla v1, la v2 y la de todas | Comparabilidad con lo medido antes |
| L2 | **Mapa de alcanzabilidad por aspecto de grafo**, sin generar: para cada aspecto, con los defaults, ¿llega `answer_clause` al contexto? Si no, el **porqué** con vocabulario cerrado: `not_retrieved` (ni la consulta ni nada la trae), `not_pointed` (`from_clause` no llegó), `pointed_not_resolved` (la arista no la detecta el parser actual), `pointed_not_followed` (detectada y no seguida: puntero a documento con `xref=clause`), `over_limit`, `reached`. Para negativos: `not_followed` / `followed` | Es el techo de cada brief antes de pagar una pasada (comprobación sin generación de S13) |
| L3 | Resumen por `edge_kind` y por `hops`: cuántos aspectos `reached` y cuántos en cada porqué | Decide el orden real de los briefs 3–5 y si hay casos de 2 saltos |
| L4 | Artefacto versionado en `data/groundtruth/`, re-puntuable sin base ni modelo | Como el resto del arnés |

---

## 3. Parte C — `delegated` hacia el corpus (independiente de la anotación)

| # | Decisión | Porqué |
|---|---|---|
| D1 | Para cada hueco `delegated` de un artefacto de generación, resolver el destino de su cita con el parser de `xref` contra la tabla `documents`: `out_of_corpus`, `in_corpus_injected` (el destino ya estaba en el contexto: problema de generación), `in_corpus_not_injected` (problema de retrieval: lo que el grafo debía traer), `unresolved` | Separa el fallo de retrieval del de generación; determinista |
| D2 | Métrica de **arnés**, no de runtime: un informe sobre artefactos existentes. Nada en el contrato ni en `/query` | Decidido en el resumen §6: disparador en runtime duplica el turno |
| D3 | Re-puntuar `generation_20261024-140241` (`gaps`) y la UAT de S14; reportar cada `delegated` con su destino y clase | Línea base de la métrica para los briefs 3–5 |

---

## 4. Guardas comunes preregistradas (briefs 3–5)

Se fijan aquí, antes de ver ningún resultado del grafo. Cada brief fija solo su **N** (aspectos
objetivo) a partir del techo de L2, antes de generar. Pasada pareada `control` (defaults) frente a la
variante, ≥ 3 ejecuciones por consulta.

| # | Guarda | Umbral |
|---|---|---|
| 1 | Aspectos objetivo del `edge_kind` del brief resueltos por evidencia **y** aspecto | ≥ N, cada uno en ≥ 2/3 ejecuciones |
| 2 | Recall sobre relevantes recuperadas | −0,03 como mucho frente a `control` |
| 3 | `verified` | 1,00 |
| 4 | Precisión sin lo añadido solo por grafo | −0,03 como mucho |
| 5 | Negativos del tipo: aspecto `not_followed` o, si se sigue, nada de ello citado | 100 % |

**Se reportan, no deciden:** latencia mediana/p95, coste por turno, tokens de entrada, `not_found`
por ejecución, `delegated` por clase (D1), citas fuera de su afirmación (deriva de S14: 25 → 43;
vigilar).

Regla de adopción: adoptar por error asimétrico — si la variante falla y se apaga, el sistema vuelve
al comportamiento de `control`, así que pesa la precisión de lo servido. Si una guarda falla, CC no
itera: reporta el porqué no de cada caso y la excepción queda registrada.

CC deja estas guardas como **constante versionada** del arnés (con este brief como fuente), no en
prosa.

---

## 5. De paso (pendientes sin brief)

- `corpus/groundtruth/uat/records.jsonl`: `S14-2-panel-look` pasa de `pending` a `partial`
  (`rationale`: la lista larga de cláusulas en el desplegable no sirve; se revisa en la sesión de
  UI/UX).
- Compose: publicar Postgres en `127.0.0.1`, no en `0.0.0.0`.

---

## 6. Tests

Mockeados, sin red ni Docker: importador (cita que no está → rechazo; `edge_kind` inválido →
rechazo; reetiquetado de los 14 aspectos y correcciones), clasificador de L2 sobre contextos
sintéticos (un caso por porqué), clasificador D1 (un caso por clase), guardas como datos.

## 7. Cierre

`pre-M6-decisions.md` en la carpeta de la sesión (agnóstico/dominio etiquetado), con: lo que G7
encontró sobre "Expected output" y matrices en el parse; si el Space-to-Ground ICD entró como
negativo; tablas L1–L3 y D3. `CHANGELOG.md`, `CLAUDE.md`, `ARCHITECTURE.md` y `RUNBOOK.md` §13 al
día. UAT de este brief: no hay render; la UAT es la revisión de L2/D3 por Salva o por el chat.
