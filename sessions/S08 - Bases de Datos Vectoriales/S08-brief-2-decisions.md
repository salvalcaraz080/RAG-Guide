# S08 · Decisiones — Brief 2 (propagación de `requirement_uid` al chunk)

> Decisiones crudas tomadas al implementar, etiquetadas **[AGNÓSTICO]** (transferible a
> cualquier sistema RAG) o **[DOMINIO]** (específico de ECSS / de este proyecto).
> **Verificación:** `uv run pytest` (760 tests) + comparación del artefacto viejo contra el
> nuevo sobre los 13.876 chunks reales, 14/14 comprobaciones verdes. **Cero llamadas al
> embedder.**

---

## 1. La rama de fingerprint que resultó: **A**, pero no por el motivo que el brief anticipaba

El brief planteaba dos ramas: (A) el fingerprint digiere `embed_text` → no se mueve; (B) digiere
el registro completo del chunk → hay que estrechar la función. **La realidad es una tercera, y es
mejor que A: cada stage digiere SU PROPIA ENTRADA.**

| Fingerprint | Qué digiere | ¿Lo mueve añadir `requirement_uid`? |
|---|---|---|
| `chunk_all.input_fingerprint` | los ficheros `ParsedDocument` de `data/processed/` | No: los documentos no cambiaron |
| `embed_all.stage_fingerprint` | los `embed_text` + modelo + dimensiones | No: `embed_text` es byte-idéntico |

Ninguno digiere el registro del chunk, así que **no hubo nada que estrechar**. Verificado
empíricamente, no deducido: `ab3fce8b5cb1a0d4` antes y después, y emparejando con el
`chunks_fingerprint` del sidecar, que no se tocó.

**Por qué esto es más fuerte que la rama A del brief.** Que `chunk_all` digiera su *entrada* y no
su *salida* significa que un cambio en la **lógica** del chunker tampoco mueve el fingerprint — lo
cual suena a defecto pero no lo es, porque el fingerprint de este stage responde «¿cambiaron los
documentos que consumí?». Quien responde «¿debo re-embeber?» es el `stage_fingerprint` de
`embed_all`, y ese sí digiere `embed_text` y **sí** se habría movido si el UID hubiera contaminado
el texto. Las dos preguntas tienen dueños distintos, y esa separación es la que hizo este brief
barato.

Lección transferible: *una clave de invalidación digiere lo que su consumidor consume. Si un stage
tiene dos consumidores con preguntas distintas, necesita dos claves, no una compartida.*

---

## 2. Transportar vs re-derivar — **[AGNÓSTICO]**, y la trampa concreta

Vía (a) del brief, ejecutada tal cual: `Requirement.uid` → `Chunk.requirement_uid`. La razón por
la que (b) era una trampa se puede enseñar con dos ids reales del corpus:

```
ECSS-E-ST-10-03C#4.1::ECSS-E-ST-10-03_0750001    ← el sufijo ES el UID
ECSS-E-HB-40A#C.2.1::r0                          ← el sufijo es el ORDINAL, no un UID
```

En ese sufijo **conviven dos cosas de naturaleza distinta**, y solo el sitio que lo construyó sabe
cuál es cuál. Parsearlo habría exigido una regla del tipo «si empieza por `r` y sigue un número,
no es UID» — una heurística sobre un espacio de nombres que no controlamos, en un corpus donde el
68% de los requisitos no tiene UID. Leer de `Requirement.uid` da el `None` **de forma natural**,
sin regla que mantener.

Verificado que no hay fuga: **0 valores con forma `r<n>`** en los 13.876 chunks.

---

## 3. `or None` en vez de `""` — **[AGNÓSTICO]**

El parser deja `Requirement.uid = ""` cuando el documento no trae UID. Se propaga como
`req.uid or None`. Un `""` en la columna afirmaría «este requisito tiene un UID, y es la cadena
vacía»; `NULL` dice «no tiene». La distinción importa porque la columna es `TEXT NULL` y el
`COUNT(requirement_uid)` de cualquier informe futuro contaría los `""` como presentes.

---

## 4. La garantía central: `embed_text` byte-idéntico, verificado contra el artefacto anterior

Antes de regenerar se copió `chunks.json` y `embeddings.json`. La comparación vieja↔nueva sobre
los 13.876 chunks:

| Comprobación | Resultado |
|---|---|
| `embed_text` byte-idéntico | **0 diferencias** / 13.876 |
| `cite_ref` sin cambios | 0 diferencias |
| `chunk_id` sin cambios | mismo conjunto de ids |
| Único campo que cambia | `requirement_uid` (y solo ese) |
| Requisitos con UID | **2.536** de 7.922 (confirmadas las cifras del brief 1) |
| Requisitos sin UID | 5.386 en `NULL` |
| Fuga de ordinal | 0 |
| Otras cuatro anclas con UID | 0 |
| UID ↔ sufijo del id, donde lo hay | 0 discrepancias |
| `input_fingerprint` ↔ sidecar | `ab3fce8b5cb1a0d4` en los tres sitios |
| Sidecar y `embeddings.npy` | intactos, MD5 y mtime sin cambiar |
| Orden fila→chunk | las 13.876 filas emparejan |

Conteos por ancla y autoridad idénticos al brief 1 (requirement 7.922 · section 2.583 ·
table 1.481 · note 1.410 · glossary_term 480; 10.320 normative / 3.556 informative), 0 colisiones,
0 fugas. **Regenerar no reestructuró nada**, que era el punto.

**Esa comparación no es un test de la suite**, y deliberadamente: compara contra una copia que
solo existió en esta sesión. Lo que sí quedó en `tests/corpus/` es su forma **generalizable** —
construir el mismo chunk con y sin UID y exigir `embed_text` igual. Un test que necesita el
artefacto de ayer no es un test, es una foto.

---

## 5. Sorpresa: un bug latente bloqueaba la regeneración — **[AGNÓSTICO]**

Al primer intento de regenerar, `chunk_all` murió con `KeyError: 'sections'`. Causa:
**`data/processed/` es a la vez la entrada de este build y el buzón de salida de los stages
siguientes**. `chunk_all` excluye por nombre `references.json` y `chunks.json`, pero S07 brief 6
dejó ahí un tercero, `embeddings.json`, **después** del último build de chunks. Nadie lo había
notado porque nadie había vuelto a correr el chunker.

Los dos daños, y el segundo es peor que el primero:
1. El build revienta (ruidoso, se arregla).
2. **De no reventar, el artefacto intruso habría entrado en el digest como si fuera corpus** —
   invalidando los chunks sin que hubiera cambiado ni un solo documento. Silencioso.

Arreglado añadiéndolo a la lista, con test de regresión que mete los tres intrusos en un
`processed/` sintético y exige que el conjunto de documentos y el fingerprint no se muevan.

### 5.1 El mismo defecto vive en otros cinco módulos — **no arreglado, a propósito**

Medido hoy sobre `data/processed/` (26 documentos reales, 29 ficheros `.json`):

| Módulo | Documentos que vería | De más |
|---|---|---|
| `graph.py` | 28 | `chunks.json`, `embeddings.json` |
| `inspect_graph.py` | 28 | idem |
| `verify_parser.py` | 28 | idem |
| `audit_raw_corpus.py` | 28 | idem |
| `report.py` | **29** | los tres |
| `chunk_all.py` (arreglado) | 26 | — |

Sus artefactos actuales son anteriores a que aparecieran los intrusos, así que **hoy están bien**;
la siguiente corrida de cualquiera de ellos no lo estaría. No se han tocado por dos razones: está
fuera del alcance de este brief, y **la corrección buena no es añadir el nombre en seis sitios**
sino derivar el conjunto del **manifiesto**, que ya es la fuente de identidad de `parse_all`. Eso
toca seis módulos y cambia firmas: es decisión de Claude Chat, no de CC. Anotado en `CLAUDE.md`
como pendiente con la medida incluida.

Lección transferible: *un directorio que es a la vez entrada de un stage y salida de otro
convierte «qué hay aquí» en una pregunta ambigua. Una lista de exclusión la responde mal por
omisión; un manifiesto la responde por declaración.*

---

## 6. Lo que se dejó sin hacer, según el brief

- **Cero re-embed.** No se ejecutó el embedder. `embeddings.npy` y el sidecar sin tocar, MD5
  verificado.
- **Nada ingerido en Postgres** (es el brief 3 tras la renumeración).
- **Schema sin tocar** — la columna ya existía del brief 1.
- **Sidecar sin re-estampar**: no hizo falta, y de haber hecho falta la instrucción era estrechar
  la función, no parchear el dato.
- **No se arreglaron los cinco módulos de §5.1** ni el `[0.26.0]` duplicado del changelog.

---

## 7. Nota de alcance

Se corrigió de paso una cifra desfasada en `ARCHITECTURE.md` («13.888 chunks» → **13.876**), la
misma que `S08-brief-1-decisions.md` §4.1 ya había señalado. Es la documentación del artefacto que
este brief regenera, así que entraba en el alcance; se anota para que no parezca deriva.
