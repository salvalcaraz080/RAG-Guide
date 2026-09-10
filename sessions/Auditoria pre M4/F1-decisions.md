# F1 — Log de decisiones (filtro de marcadores no citables + oráculo de completitud)

Log crudo de qué se cambió y por qué. Etiquetado **agnóstico** (vale para cualquier proyecto) /
**dominio** (específico de RAG-ECSS).

**Estado:** las tres tareas cerradas y verificadas. `uv run pytest` → **891 passed, 8 deselected**
(desde 854: +37 tests). Base recargada: **26 documentos, 13.695 chunks**. Árbol listo, **sin
commitear**.

**Cierra:** T1-1, T1-3, T1-8 y O-2 del `pasoD-diagnostic-report.md`.

---

## 0 · El diagnóstico primero, y encontró una trampa que el brief no podía saber

Antes de escribir nada, se midió el universo real de formas marcador-like en `chunks.json`. **26
formas distintas, 182 chunks** — y una de las 182 **no es un marcador**:

```
'Unnecessary tools shall be removed from working areas.'
```

Cayó en la red porque contiene **«moved» dentro de «removed»**. Es normativa real de pleno derecho.
Los marcadores reales son **181**, y ese único falso positivo fijó la primera decisión de diseño.

**[agnóstico] El predicado compara el verbatim ENTERO, nunca por subcadena.** El brief ya lo pedía
(«ojo con el sobre-emparejamiento»); el corpus dio el caso concreto que lo demuestra, y quedó como
test (`test_real_clauses_are_not_markers`).

---

## 1 · Tarea 1 — el predicado

**[dominio] Tres categorías nombradas**, en vez de un booleano: `bare_dash` (51), `no_content`
(55, los `None`/`None.` de O-2), `withdrawn` (75). El código dice *por qué* descarta, y el log
reporta el desglose. Total **181**.

**[dominio] Renombrado** `is_retracted`/`drop_retracted` → `is_non_citable_marker`/
`drop_non_citable`. «Retracted» sub-declaraba: los 55 `None.` no son cláusulas retiradas, son
cláusulas sin contenido, y el nombre tiene que nombrar lo que distingue.

**[dominio] Desviación del brief, medida: NO se exige el `>>` de cierre.** El brief decía
«encerrado en `<<…>>` o `>>…>>`». El corpus trae un marcador **sin cerrar**:

```
ECSS-Q-ST-20C#5.8.7::r0  →  '<<deleted, requirement moved to 5.8.4.4a.'
```

Exigir el cierre lo habría dejado fuera — es decir, habría reproducido T1-1 con otra forma. El
predicado exige **apertura** (`<<` o `>>`, tolerante al espaciado) más una palabra clave. Va con
test y con comentario en el código nombrando ese chunk.

---

## 2 · Tarea 2 — el oráculo, y la vocabulario que el brief sugería no servía

**[agnóstico] Fuera `EXPECTED_RETRACTED = 121` y su gate.** El conteo se loguea por categoría, no
se asserta. Esto elimina de paso la atadura documental a `corpus_version` que el código nunca leía:
sin número mágico no hay atadura que mantener.

**[agnóstico] Calibración contra datos, no contra la intuición.** El brief proponía como léxico de
marcador `deleted, moved, none, covered by, see, refer to…`. Se midió qué cláusulas **reales**
cortas tiene el corpus antes de fijarlo:

```
  4  'A-A-'
 10  'See source'
 11  'See 5.3.5.2'
 13  'See also 4.2.'
 10  'SID S3 SID'
```

Con `see` en el léxico de marcador, las cuatro con «See» se marcan → **la allow-list no quedaría
vacía**, que es justo el objetivo que el brief fija. Y `'A-A-'` cae si la regla es «solo relleno».

**[agnóstico] La corrección que hace el oráculo utilizable: exige al menos una palabra
DISTINTIVA.** No basta con «no tiene contenido»; tiene que **decir** algo de marcador
(`deleted`, `moved`, `modified`, `incorporated`, `covered`, `none`, `withdrawn`) y que todo lo
demás sea relleno, coordenada de cláusula (token con dígito) o letra suelta de sub-ítem. Con eso:

- `'A-A-'` → no hay palabra distintiva → **no se marca** ✅
- `'See also 4.2.'` → no hay palabra distintiva → **no se marca** ✅
- `'None of the requirements in clause 5 have been deleted from the plan.'` → tiene distintivas,
  pero también `requirements`, `plan` → sustantivo → **no se marca** ✅

**[agnóstico] Es más ancho que el predicado, y ahí está su valor.** No exige el `<<` de apertura:
`'deleted, moved to 5.4.1'` sin corchetes **lo caza el oráculo y no el predicado**. Eso es lo que
lo hace un instrumento distinto y no un eco — hay test explícito.

**Auto-test, con el código real (no con la sonda de diseño):**

```
AUTO-TEST 1 · predicado OFF -> el oráculo ve 181/181   OK
AUTO-TEST 2 · predicado ON  -> 0 supervivientes marcados (allow-list vacía)   OK
```

**[agnóstico] La allow-list existe pero está vacía, y esa es la señal.** Se relaja por **id y a
mano**, nunca aflojando la heurística: aflojarla reabre el agujero para todos. Que esté vacía dice
que el predicado no se deja nada.

---

## 3 · Tarea 3 — ningún documento se queda a cero

**[agnóstico] `assert_no_document_emptied(before, after)`**, comparando el conjunto de documentos
antes y después del filtro. Falla nombrando los vaciados, **antes de tocar la base**.

Convierte el `KeyError` opaco de `build_rows` en un error que se lee, y cubre el caso peor —el
documento que se carga sin filas sin que nada lo nombre—, que el `KeyError` ni siquiera alcanzaba.
Todo-o-nada aplicado al documento.

**Margen real tras F1:** el documento más afectado, `ECSS-E-HB-40A`, pasa de 497 a **446** chunks.
Sigue lejos de cero, pero al ampliar el filtro el margen bajó y ahora está asertado.

---

## 4 · Verificación

| Comprobación | Antes de F1 | Ahora |
|---|---|---|
| `/search` «requirement moved deleted» (hybrid, k=5) | **4 marcadores** en el top-5, `requirement/normative` | **5 cláusulas reales** |
| Marcadores de retirada en base | 5 | **0** |
| `-` / `None` / `None.` en base | 106 | **0** |
| `cite_tsv` vacío | 0 | **0** |
| Chunks en base | 13.755 | **13.695** |
| Documentos | 26 | 26 |
| Oráculo sobre supervivientes | — | **0**, allow-list vacía |
| Auto-test del oráculo | — | **181/181** sin predicado, **0** con predicado |
| `input_fingerprint` | `ab3fce8b5cb1a0d4` | `ab3fce8b5cb1a0d4` (sin re-embedding) |
| Suite | 854 | **891** |

**Prueba de que las regresiones muerden.** El predicado anterior, contra los casos nuevos:

```
'<< deleted, requirement moved to 5.8.1a >>'  viejo=SE LA DEJA  nuevo=withdrawn
'>>deleted>>'                                 viejo=SE LA DEJA  nuevo=withdrawn
'None.'                                       viejo=SE LA DEJA  nuevo=no_content
```

**Re-ingesta:** `TRUNCATE TABLE chunks` + recarga, mismo motivo que en C2 — el upsert es
idempotente pero no borra, así que un conjunto cargable que se **reduce** deja huérfanos.

---

## 5 · Nota para quien retome el fix aguas arriba (T2-4, diferido)

**El oráculo NO es andamiaje desechable, es el test de aceptación de ese fix.**

Hoy `assert_no_marker_survivors` corre sobre los supervivientes del filtro de carga y significa
«el predicado no se dejó nada». El día que los marcadores se traten en `parser`/`chunking` para que
**no lleguen a ser chunks**, el mismo oráculo —sin reescribirlo— pasa a significar «no se produce
ningún chunk-marcador». Solo cambia dónde se le llama: sobre `chunks.json` recién construido en vez
de sobre los supervivientes de la carga.

Está anotado en el docstring de la propia función para que no se borre por parecer temporal.

Recordatorio del coste que mantiene ese fix diferido: **dispara re-chunking y re-embedding** del
corpus entero, porque los 181 siguen en `chunks.json`, tienen `embed_text` y entran en el
`input_fingerprint`.

---

## 6 · Ficheros tocados

**Código** — `app/services/ingestion.py` (predicado, oráculo, guarda de documento vaciado, reporte
por categoría; fuera `EXPECTED_RETRACTED`).

**Tests** — `tests/app/test_ingestion.py`: reemplazado el bloque tautológico por
`TestNonCitablePredicate`, `TestCompletenessOracle`, `TestDropNonCitable`,
`TestNoDocumentIsEmptied`.

**Docs** — `.claude/CLAUDE.md` (cifras de estado a 13.695; **cerrado** el pendiente de los «2
párrafos `None.`», que resultaron ser 55 y ya se filtran; item del filtro reformulado a predicado +
oráculo; **retirado** el pendiente de `EXPECTED_RETRACTED`, cuya constante ya no existe; decisión
consolidada nueva sobre los dos instrumentos). `CHANGELOG.md` (entrada `[0.33.0]`).

**Lista de pendientes: de 24 a 22.** Salen dos, no entra ninguno.

**Sin commitear.** Mensaje propuesto en la respuesta de CC.
