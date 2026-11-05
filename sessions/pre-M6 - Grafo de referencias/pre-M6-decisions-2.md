# pre-M6 — Decisiones del brief 2: acumulador de evidencia por cláusula

> Refactor sin cambio de contexto: **165 digests idénticos** antes y después, y la tabla L2 igual. El
> contrato gana el camino de cada evidencia. Coste real: embeddings de las consultas de la comprobación
> (despreciable) y 0,0035 USD de la UAT. **Sin diferencias que reportar.**

## 0. L1 válido (cierra el brief 1)

229 pares importados (15 relevantes, 214 no). Golden set v3: 29 consultas, 1.057 juicios, **VALID** a
la primera, 0 sin juzgar. Artefacto: `measurement_20261101-192515.md`.

| Consultas | n | p@k | R@k | R@k norm | R@candidatos |
|---|---|---|---|---|---|
| v1 | 10 | 0,66 | 0,32 | 0,31 | 0,53 |
| v2 | 14 | 0,54 | 0,26 | 0,25 | 0,42 |
| v3 | 29 | 0,48 | 0,34 | 0,36 | 0,51 |

| Estrato | n | p@k | R@k | R@candidatos |
|---|---|---|---|---|
| `names_standard` | 2 | 0,59 | 0,29 | 0,53 |
| `names_standard_xref` | 2 | 0,21 | 0,17 | 0,15 |
| `generic` | 2 | 0,67 | 0,53 | 0,66 |
| `paraphrase` | 2 | 0,54 | 0,13 | 0,22 |
| `comparative` | 2 | 0,59 | 0,29 | 0,49 |
| `cross_standard` | 2 | 0,68 | 0,22 | 0,48 |
| `multi_turn` | 2 | 0,51 | 0,22 | 0,41 |
| `graph_document_pointer` | 2 | 0,38 | 0,24 | 0,18 |
| `graph_drd` | 3 | 0,51 | 0,61 | 0,83 |
| `graph_applicability` | 2 | 0,14 | 0,24 | 0,52 |
| `graph_hierarchy` | 2 | 0,55 | 0,71 | 0,71 |
| `graph_by_name` | 3 | 0,41 | 0,33 | 0,63 |
| `graph_negative` | 3 | 0,49 | 0,34 | 0,53 |

Estratos de 2–3 consultas: indicativos. **Corrijo lo que dije en el brief 1:** llamé cota inferior a la
p@k de v2, y con todo juzgado sigue en 0,54. El 0,69 que consta en `CLAUDE.md` era `filter` sin
expansiones; `defaults` entrega más cláusulas por expansión y `xref`. Ese es el candidato evidente de la
diferencia, **sin aislar**: no medí las dos configuraciones sobre el mismo golden set. `CLAUDE.md`
queda con las cifras de `defaults`.

## 1. Regla de orden (K2)

Lo que producía el sistema era `retrieval + expansión + xref` por concatenación. Ahora es una regla:

**Los chunks se ordenan por `(precedencia del canal, rango de la plaza, posición en la plaza)`.**

| Clave | Valor |
|---|---|
| Precedencia | `retrieval` 0, `expansion` 1, `xref` 2 |
| Rango, `retrieval` | Rango del chunk en el retrieval |
| Rango, `expansion` | Posición de la cláusula entre las recuperadas, por su primer chunk |
| Rango, `xref` | Orden de aparición de la referencia en el texto inyectado |
| Posición, `bounded` | Normativos primero; dentro, orden del artefacto |
| Posición, `orphans` | Normativos, orden del artefacto |
| Posición, cláusula `xref` | Normativos primero; dentro, distancia a la consulta |
| Posición, estándar seguido | Rango del retrieval filtrado |

`group_by_clause` numera por primer chunk, así que esta regla fija los `[n]`.

## 2. Comprobación

Digest por turno de: bloque del prompt, chunks en orden, traza `xref` y degradaciones. `origin` queda
fuera porque cambia a propósito y no llega al prompt. Vectores guardados en la primera ejecución, así
que antes y después leen los mismos. Dos ejecuciones sobre el código anterior ya salían iguales.

| Configuración | Turnos | Difieren | Digest |
|---|---|---|---|
| `defaults` | 33 | 0 | `1195d38dedad9761` |
| `xref=off` | 33 | 0 | `8400b4948faaa23c` |
| `expansion=off` | 33 | 0 | `ce628cab6e752fd7` |
| `expansion=orphans` | 33 | 0 | `7fb9ff2435c3bf99` |
| `clause_standard` | 33 | 0 | `ca15f676ea43647b` |

- **L2:** `graph_reach run` y `rescore` dan la misma tabla, aspecto por aspecto, que el brief 1.
- **Hashes fijados de `test_references.py`:** **no se re-fijaron.** El test proyecta al formato anterior
  (sin los campos nuevos, con `retrieved`) y reproduce los valores antiguos. El hash de `QueryResponse`
  sí se re-fijó: el esquema cambia a propósito.
- **Suite:** 1.626 verde (17 nuevos). Ruff: un E402 en `app/main.py`, anterior a la sesión.

## 3. Decisiones de implementación

| # | Decisión | Porqué |
|---|---|---|
| 1 | **`origin` no es solo aditivo.** `retrieved` pasa a `retrieval`, y lo traído por referencia pasa de `expansion` a `xref` | Lo pide K4 con esos valores. Afecta al contrato y a `/search`. La UI y `aspects.py` aceptan `expansion` y `xref`; los artefactos antiguos se siguen puntuando igual |
| 2 | Los topes viven en `admit`, un solo bucle | `expand`, `fill` y `_follow` repetían el mismo, con otra escritura. Los topes por canal son los de antes. `expand` y `fill` reciben el acumulador: cambio de firma |
| 3 | Plaza con `rank` y lote entero en `admit` | Sin eso, los topes dependerían del orden de llamada y K2 no aguantaría una permutación |
| 4 | `origins` es **por cláusula**: los otros caminos a la cláusula del chunk | Hoy un chunk solo llega por un camino; lo que se repite es la cláusula. Incluye `xref` cuando una referencia nombra una cláusula ya inyectada |
| 5 | Un chunk de canal menor se sustituye si lo ofrece uno mayor | K3. Hoy no ocurre, porque los canales corren en orden. Solo lo ejercen los tests; lo necesita un canal nuevo |
| 6 | `edge_kind="explicit_document"` en lo seguido por `clause_standard` | El brief solo nombraba `explicit_clause`. Es apagado por defecto, pero `explicit_clause` sería falso |
| 7 | La expansión de cláusula lleva `hop=0` y sin `from_clause` | Completa una cláusula entregada: no da un paso del grafo |
| 8 | `RETRIEVAL_GRAPH_MAX_HOPS` admite solo 0 y 1 | No hay segunda pasada: un 2 no haría nada y parecería que sí |
| 9 | `xref` sigue sin leer lo que ella misma trae | K5: es lo que dejó q16 sin alcance en el brief 1. Si el brief 3 lo cambia, el sitio es `readable()` |

K6: nada se persiste, así que `corpus_version` no aplica. K7: las degradaciones salen igual y están
dentro del digest.

## 4. UAT

`pre-M6-uat-2.md`: q12, `verdict = pending`, juzga el chat.
