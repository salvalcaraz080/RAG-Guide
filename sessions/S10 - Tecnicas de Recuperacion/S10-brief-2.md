# S10 — Brief 2: canal léxico, routing por estándar nombrado y recall con techo

> Continuación de `S10-brief.md`, sobre `S10-decisions.md` §6–§7. Lee `.claude/CLAUDE.md` y
> `ARCHITECTURE.md` antes de empezar.
> Todo lo nuevo entra en el camino de producción **detrás de configuración, con el default actual
> intacto** (`RETRIEVAL_MODE=vector`, sin routing). Qué pasa a default lo decide Claude Chat con la
> tabla delante.

---

## 0. Objetivo y alcance

**Objetivo.** Las verificaciones de §6 mostraron que la híbrida no está mal *ponderada*: está rota
en dos sitios (pre-tokenizado que destruye los identificadores y `ts_rank` sin IDF), y que un
estándar nombrado en la pregunta es un **contenedor**, no un término. Este brief arregla el canal
léxico, añade routing determinista por estándar nombrado, corrige la métrica de recall y mide todo
como configuraciones del arnés.

**Dentro de alcance:** recall con techo (§1 E1), tokenización léxica (E2), ranking léxico con
longitud e IDF (E3), detector de estándar nombrado + filtro y cuota (E4), medición de las
configuraciones de §2, ciclo incremental de anotación.

**Fuera de alcance:** pesos en RRF, reranker, expansión de cláusula o de consulta, cambiar
defaults, tocar `corpus/` o re-embeber. Si la tabla final sugiere ponderar RRF, se **reporta**, no
se implementa. Lo que parezca exigir otra cosa: `# TODO: pendiente de decisión en Claude Chat`.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| E1 | **Recall con techo** junto al actual: `R̂@k = aciertos / min(|R|, k)` y `R̂@cand = aciertos / min(|R|, cláusulas_candidatas)`. Las columnas existentes se mantienen | Con 16–55 relevantes por consulta y ~25–35 cláusulas en 50 chunks, R@cand mezcla incapacidad real con techo estructural. Sin separarlo no se puede decidir si un reranker tiene margen |
| E2 | **La query llega al canal léxico tokenizada por Postgres**, con la misma configuración que `cite_tsv`, en **OR** como hoy. De cada compuesto con guiones se conserva **el compuesto y se descartan sus partes** (tipos `*hword_*part` de `ts_debug`); números tipo `5.3.5.1` se conservan como token `version` | Simetría query/documento garantizada por construcción. Las partes (`e`, `st`, `rev`, `2`) son ruido en OR; el compuesto (0,7 % df) es la señal. Es la corrección de un defecto, no una decisión de diseño: se mide aparte solo para cuantificarlo |
| E3 | **Ranking léxico con IDF:** BM25 calculado en SQL con una tabla de estadísticas de términos (lexema → df) y longitud por chunk, `k1=1.2`, `b=0.75` (valores estándar, **sin ajustar**). Además, como configuración barata de contraste: `ts_rank` con normalización por longitud (CC elige la máscara y justifica) | `ts_rank` no tiene IDF: gana quien repite términos comunes en chunks largos (§6.3). El corpus está congelado → la tabla de df se calcula una vez con la ingesta, sin extensión ni infraestructura nueva. Si una extensión BM25 ya está disponible en la imagen sin coste, CC puede proponerla en decisiones, pero no cambiar la imagen en este brief |
| E4 | **Routing determinista por estándar nombrado:** detector en el lado de la consulta que reutiliza las formas de nombrar un documento de `corpus/identity.py` y devuelve una **lista** de `canonical_doc_id` (vacía si ninguno). Se aplica sobre la **query de retrieval compuesta** (la de producción, con historial) | Nivel 1 de la escalera de routing: gratis, determinista, auditable. Lista = varios estándares nombrados se buscan todos. Operar sobre la query compuesta mantiene D8 del brief 1 (medir lo que usa producción) y hace que "and in Q-ST-80?" herede el estándar del turno anterior |
| E5 | Dos modos de routing, ambos medidos: **filtro** (`WHERE canonical_doc_id IN (…)` en las ramas de búsqueda) y **cuota** (las primeras `ceil(q·k)` plazas de cláusula salen de la búsqueda filtrada, el resto de la no filtrada, deduplicado; `q=0.7` configurable) | Filtro: máxima precisión, rompe referencias cruzadas. Cuota: deja hueco a la cláusula de otro estándar que la respuesta necesite. `q=0.7` es un número de partida declarado, no calibrado |
| E6 | El routing **registra en log** qué estándares detectó y la **cardinalidad tras filtrar** | Un filtro que vacía el resultado en silencio es el fallo más difícil de depurar. En régimen exacto el `WHERE` es correcto; anotar en `ARCHITECTURE.md` que con un índice HNSW dejaría de serlo |
| E7 | **Sin detección → sin cambio**: si el detector devuelve lista vacía, la petición sigue exactamente el camino actual | El routing no puede degradar las 8 consultas que no nombran estándar |

---

## 2. Configuraciones a medir

Todas contra el golden set congelado, mismas métricas que el brief 1 + E1.

| Config | Qué es |
|---|---|
| `vector` | Baseline actual |
| `hybrid` | Baseline actual (rota) |
| `hybrid_tok` | `hybrid` + E2 (cuantifica el defecto) |
| `hybrid_tok_len` | E2 + `ts_rank` normalizado por longitud |
| `hybrid_tok_bm25` | E2 + BM25 (E3) |
| `vector_filter` / `vector_quota` | `vector` + routing E4/E5 |
| `best_filter` / `best_quota` | Routing sobre la mejor de las híbridas arregladas, **solo si** esa híbrida supera a `vector` en p@k global |

Además del canal fusionado, el informe muestra para las híbridas el **ranking léxico solo** en el
arquetipo genérico 1 y en `q01`: si con E2+E3 sigue ganando la prosa de HB y las tablas DRD, se
dice.

---

## 3. Trabajo

1. **E1** en las métricas e informe.
2. **E2** en el canal léxico; mostrar en decisiones el antes/después de la tsquery para
   `ECSS-E-ST-40C`, `E-ST-40C`, `Q-ST-80C Rev.2`, `5.3.5.1`.
3. **E3**: tabla de estadísticas de términos (dónde vive y cuándo se regenera: CC decide y lo
   justifica; debe quedar ligada al `input_fingerprint`) y la consulta BM25 sobre los chunks que
   casan.
4. **E4–E7**: detector, filtro, cuota, logging; flags de configuración con default apagado.
5. **Medición** de §2. Cada configuración que deje cláusulas sin juzgar en su top-k → export de
   pares en modo B → **pausa** para que el usuario ejecute la sesión de anotación → import →
   re-medir. Agrupa los pares de todas las configuraciones en **una sola exportación** para que la
   sesión de anotación se haga una vez.
6. **Tests de contrato** (§4).

---

## 4. Tests de contrato (sin red, sin BD)

Como mínimo:
- E1: techo con |R| < k, |R| > k, |R| = 0 (`n/a`).
- E2: compuesto conservado y partes descartadas; `version` conservado; texto sin identificadores
  produce los mismos lexemas que hoy (salvo los compuestos); OR preservado.
- E3: BM25 sobre un corpus de juguete con valores calculados a mano (df, longitud media, k1, b);
  un término raro supera a uno común con igual tf.
- E4: detector sobre las formas de `identity.py` (con y sin prefijo `ECSS-`, con revisión, varios
  en una frase, ninguno); **sin falsos positivos** con "ST", "E", "Q" sueltos, números de cláusula o
  texto que contenga `-ST-` fuera de un ID.
- E5: cuota con menos resultados filtrados que plazas (rellena con el resto), deduplicación entre
  ambas listas, `q` en los extremos.
- E7: lista vacía → mismas llamadas que el camino actual.
- Ningún veredicto depende de variables de entorno opcionales.

---

## 5. UAT (usuario)

Caso 1 (no hay oráculo) y caso 2 (decisión):
1. Sesión de anotación en modo B sobre la exportación agrupada (mismo workspace, protocolo 1.0).
2. Leer la tabla final por estrato y la recomendación de CC.

---

## 6. Cierre

- `S10-decisions-2.md`: decisiones + porqué, etiquetadas **agnóstico/dominio**; antes/después de
  E2; dónde vive la tabla de df; tabla completa de §2 por estrato con E1; nº de pares juzgados en
  modo B y cuántos relevantes; y una **recomendación razonada** (no aplicada) de qué configuración
  pasaría a default y si RRF ponderado sigue haciendo falta.
- `CHANGELOG.md`, `CLAUDE.md`, `ARCHITECTURE.md` según sus reglas. En `CLAUDE.md` §Decisiones
  pendientes: "Recuperar el estándar nombrado" y "Ponderar la fusión híbrida" se reescriben con el
  estado medido, no se borran hasta que Claude Chat decida el default.
- `uv run pytest` en verde y mensaje de commit propuesto. El commit lo hace el usuario.
