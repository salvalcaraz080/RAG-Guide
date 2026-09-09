# S08 · Brief 5 — Búsqueda híbrida (léxico + vector)

> **Producido por:** Claude Chat (decisión). **Ejecuta:** Claude Code.
> **Alcance:** canal léxico full-text sobre `cite_text` + fusión con el canal vectorial exacto vía
> RRF, expuesto como `mode=hybrid` en `/api/v1/search`. Extiende `query_examples.py` a comparar
> vector vs híbrida. **Sigue sin cablear el seam, sin re-embeber, sin índice vectorial.**
> Último brief de implementación de S08.
>
> **Reencuadre vs versión previa:** el Brief 4 midió que el problema B/C es de **ranking, no de
> recall** — el chunk discriminante ya está en el top-5, mal ordenado. §0, §6 y §7 lo reflejan.

---

## 0. Por qué traemos la híbrida a S08 (y no a S10)

Tres razones alineadas: (1) el ejercicio real de S08 la construye en su directo — y no diferimos
algo solo porque el máster lo mande al directo; (2) el canal léxico **en el mismo sistema** es *la
razón* por la que elegimos pgvector sobre Qdrant/Weaviate — no ejercerlo sería no cobrar la
decisión; (3) es una capacidad general útil (matches exactos de IDs, términos, siglas), no solo un
parche para B/C.

**Lo que S08 hace y lo que queda para S10.** S08 entrega el **canal léxico básico + fusión por
defecto (RRF)**. S10 sigue siendo el titular de **tuning de fusión, reranking y expansión**.
Traemos la capacidad, no su afinado.

**Encuadre del caso B/C, corregido por lo que midió el Brief 4 (§8.4).** El problema **no es de
recall**: el chunk que distingue B de C (`ECSS-Q-ST-40C §6.5.6.3 [table]`) ya sale en el top-5 con
k=5, en el puesto 3. Lo que falla es el **orden** — dos chunks que solo enumeran "A, B o C" lo
adelantan, con las cinco distancias casi empatadas (rango 0,0227). Así que la tarea de la híbrida
aquí es **reordenar**, no recuperar. Este brief **mide** si el canal léxico lo consigue; puede que
no (§7), y eso sería un resultado válido, no un fallo.

**Nota sobre "régimen exacto".** Esa decisión es sobre el **índice vectorial** (no HNSW/IVFFlat). El
canal léxico usa un índice **GIN full-text**, que es otra cosa y es lo apropiado: barato, estándar,
no depende de HNSW. El lado vectorial sigue siendo seq-scan coseno exacto.

---

## 1. Objetivo

1. Migración: columna `tsvector` generada desde `cite_text` + índice GIN full-text (§3).
2. Canal léxico en `retrieval.py` (`ts_rank` sobre la tsvector, con los mismos filtros que el vector).
3. Fusión **RRF** de los dos rankings → top-k (§4).
4. `mode` en `search()` y en `/api/v1/search`: `vector` (default) | `hybrid` (§5).
5. Extender `query_examples.py`: cada query en ambos modos, con el **rango del chunk discriminante**
   como foco en el arquetipo 5 (§6).
6. Tests unitarios sin DB de la fusión + script UAT que **mida el efecto sobre el ranking B/C** (§7).

---

## 2. Fuera de alcance (no lo hagas)

- **Tuning de fusión** (pesos, `k` de RRF afinado, score normalizado): S10.
- **Reranking y expansión por referencias/grafo:** S10 / S09. (Nota: el Brief 4 §8.4 apunta al
  **reranker** como el fix más probable del ranking B/C, dado que el candidato ya está recuperado.
  Este brief mide antes si el canal léxico —más barato— basta; el reranker es S10.)
- **Config de text-search fina para separar tokens compartidos** (criticidad "B"/"C"): S10 — este
  brief mide con `english` básico; el diseño del arreglo si no basta es S10 (§7).
- **Criticidad como columna tipada:** posible refinamiento S10+, requiere extracción de dominio; no
  la construyas aquí.
- Cablear el seam / generación (S09). Índice vectorial, `halfvec`. Normalización de query (S09).
- Re-embeber. Deuda diferida (acoplamiento offline, manifiesto, `[0.26.0]`, redis huérfano).

---

## 3. Canal léxico — `tsvector` generada desde `cite_text` + GIN

Migración Alembic nueva (a mano; `autogenerate` necesita DB):

- Columna generada `STORED`: `to_tsvector('english', cite_text)`.
- Índice **GIN** sobre esa columna.

**Por qué desde `cite_text` y no `embed_text`.** El léxico debe casar el **verbatim citable**, no el
header contextual. Si indexáramos `embed_text`, el header (estándar + ruta de secciones) inyectaría
matches léxicos falsos: una query con un número de sección casaría por el header con todo chunk bajo
esa sección. `embed≠cite` también aquí. El match por IDs/sección se sirve con las columnas tipadas
(`canonical_doc_id`, `section_number`) y filtros, no con full-text sobre el header.

**Config `english`** como punto de partida. Si separa o no los tokens compartidos por los candidatos
(B/C) es lo que se **mide** en §7, no se asume.

---

## 4. Fusión — RRF por defecto

Dos rankings (vector exacto top-N, léxico `ts_rank` top-N, **ambos con los mismos filtros**) se
funden por **Reciprocal Rank Fusion**: `score(d) = Σ 1/(c + rank_i(d))` sobre los canales donde
aparece `d`, con `c` constante estándar (60), documentada.

**Por qué RRF y no combinación lineal ponderada.** El coseno-distancia y el `ts_rank` viven en
escalas incomparables; combinarlos por score exige normalizar y elegir pesos — el **tuning que
diferimos a S10**. RRF es **basado en rango**, no en score: robusto, sin normalización, sin pesos.
Y el Brief 4 §8.5 lo refuerza: midió que la escala de distancia **no está calibrada** (fuera-de-
dominio y genérico-en-dominio se solapan), así que fundir por rango en vez de por esa escala evita
construir sobre un número que no soporta esa lectura. RRF es la fusión "que funciona sin tuning".

**Implementación — dos queries por canal + fusión en `retrieval.py`, no un SQL monstruo.** Cada
canal es su propia query SQL (con sus filtros, atómica en sí misma); la fusión RRF se hace en Python
sobre los dos rankings. Más simple de leer y testear que un full-outer-join con RRF en SQL, y **no
sacrifica filtrar-en-SQL** (cada canal filtra en su WHERE). Sencillez > sobreingeniería. Fusión en
un solo SQL queda como optimización diferida.

- La fusión es **lógica pura** → unit-testeable sin DB.
- Los filtros se aplican **igual en ambos canales** (un hit filtrado en vector no debe colarse por
  léxico), reusando la validación del Brief 4 §3 (filtro imposible → 400, no vacío).

---

## 5. `mode` en `/search` — vector (default) | hybrid

- `search(query_text, k, filters, mode="vector")`; el endpoint acepta `mode`.
- **Default `vector`** — mantiene el baseline limpio y **preserva el arco diagnóstico**: S09 corre
  vector-puro para *diagnosticar*; `hybrid` es la *intervención* que se mide. Ambos de primera clase
  al cerrar S08.
- `hybrid` reutiliza el mismo embedding de query y la guarda `dim==1536` del Brief 4.

---

## 6. `query_examples.py` — extender a vector vs híbrida

Cada una de las 5 queries en **ambos modos**, lado a lado en `output_examples.txt`. El foco del
arquetipo 5 (B/C) es **el rango del chunk discriminante** (`ECSS-Q-ST-40C §6.5.6.3 [table]`, puesto
3 en vector puro según Brief 4): ¿la híbrida lo sube por encima de los chunks que solo enumeran
"A, B o C"? El resto de arquetipos sirven para verificar que la híbrida **no degrada** los casos que
el vector ya resolvía bien (match directo, reformulación).

---

## 7. La pregunta que este brief debe medir (no asumir) — reencuadrada a ranking

**¿El canal léxico + RRF sube el chunk discriminante B/C por encima de los enumerativos?** Camino:

1. Vector puro sobre el arquetipo 5 → reproducir el orden del Brief 4 (discriminante en puesto 3).
2. Híbrida (léxico `english` sobre `cite_text` + RRF) → ¿el discriminante sube?
   - **Hipótesis informada de que puede NO subir:** los chunks que lo adelantan **también contienen
     los tokens "b", "c", "criticality", "category"** — comparten el vocabulario, así que un
     `ts_rank` básico puede no distinguirlos mejor que el vector. Lo que hace único al discriminante
     no es "tiene B y C" (todos lo tienen) sino que **explica la diferencia** (es una tabla con
     filas por categoría) — señal estructural que ni el coseno ni el `ts_rank` capturan bien.
   - **Si sube:** el canal léxico ya paga para reordenar; documenta el delta de rango.
   - **Si no sube** (probable): **no lo arregles aquí.** Documenta *qué* no discriminó y *por qué*
     (tokens compartidos), que es el enunciado exacto del problema que S10 resuelve con **reranker**
     (lee el contenido y juzga cuál responde, no cuál comparte tokens) o config de text-search a
     medida. Es la conclusión medida más valiosa del brief, suba o no.

En ambos casos, material de Guide: o "la híbrida básica reordena tokens discriminantes cortos", o
"no basta porque los distractores comparten el vocabulario — el reordenado por contenido es
reranking, no léxico".

---

## 8. Frontera de test

**Unit (pytest, sin DB — tuyo):**
- **Fusión RRF:** dos rankings sintéticos → orden fusionado correcto; documento en un canal vs en
  ambos; `c` respetado.
- Construcción de la query léxica (sesión mockeada): `ts_rank`, filtros que componen igual que
  vector.
- Ruteo de `mode`: `vector` no toca el canal léxico; `hybrid` invoca ambos y funde.
- Coherencia de filtros entre canales; filtro imposible → 400 en ambos modos.

**UAT (Docker con datos — Salva):**
- `alembic upgrade head` aplica la columna generada + GIN.
- `EXPLAIN ANALYZE` de la query léxica → usa el **GIN full-text** (aquí sí esperamos índice; es el
  GIN, no un índice vectorial — no confundir con el régimen exacto del lado vector).
- `mode=hybrid` devuelve resultados sensatos; filtros respetados; no degrada arquetipos 1/2.
- **Medición B/C (§7):** rango del chunk discriminante en vector vs híbrida, con veredicto
  documentado (sube / no sube, y por qué).
- `query_examples.py` → `output_examples.txt` con las 5 queries en ambos modos.

---

## 9. Qué NO hacer (resumen)

- No tuning de fusión, no pesos, no rerank, no expansión (S10).
- No config de text-search a medida ni criticidad tipada (S10) — solo medir con `english` básico.
- No cablear seam/generación. No índice vectorial, no `halfvec`.
- No indexar `embed_text` para léxico — solo `cite_text`.
- No fundir en un SQL monstruo si dos queries + RRF en Python es más claro.
- No forzar que la híbrida "arregle" B/C — **medir** y documentar el resultado, sea cual sea.
- No abrir deuda diferida. No commitear (redacta el mensaje).

---

## 10. Docs

- **`ARCHITECTURE.md`:** canal léxico (tsvector generada desde `cite_text` + GIN), fusión RRF,
  `mode` en `/search`; aclarar que "régimen exacto" es sobre el índice **vectorial** y que el GIN
  full-text es ortogonal y esperado; el seam sigue estático.
- **`CHANGELOG.md`:** entrada nueva (no repitas `[0.26.0]`).
- **`CLAUDE.md`:** estado presente — S08 cierra con retrieval vector exacto + híbrida básica
  medible; el problema B/C es de ranking (no recall) y su fix probable (reranker) es S10; seam sin
  cablear (S09). Sin diario.

---

## 11. Entregables + aceptación

1. Migración (columna `tsvector` generada + GIN) aplicable a mano.
2. Canal léxico + fusión RRF en `retrieval.py`; `mode` en `search()` y en `/api/v1/search`.
3. `query_examples.py` extendido a vector vs híbrida → `output_examples.txt`.
4. `uv run pytest` verde sobre lo testeable sin DB (fusión RRF incluida).
5. Script UAT (§8) con la **medición del ranking B/C** documentada (sube / no sube, y por qué).
6. Mensaje de commit propuesto + docs (§10).
7. `S08-brief-5-decisions.md`: decisiones crudas etiquetadas, **el veredicto B/C medido** (efecto
   sobre el rango del chunk discriminante, y el porqué), y cualquier sorpresa. Con él cerramos la
   extracción de S08 → consolidado (paso 5) y Guide (paso 6).
