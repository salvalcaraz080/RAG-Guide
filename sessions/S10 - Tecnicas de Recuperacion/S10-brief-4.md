# S10 — Brief 4: reranker como experimento (solo en el arnés)

> Continuación de `S10-brief-3.md`, sobre `S10-decisions-3.md` §2 y §5. Lee `.claude/CLAUDE.md` y
> `ARCHITECTURE.md` antes de empezar.
> **El reranker no entra en el camino de petición.** Se mide en el arnés; si entra en producción, y
> cómo, lo decide Claude Chat con la tabla delante.

---

## 0. Objetivo y alcance

**Objetivo.** Responder con datos: con el margen medido en §2 de decisiones-3 (R@cand cap 0,46 → 0,70
de 50 a 400 chunks), ¿un cross-encoder local sube la precisión de lo que llega al LLM, y a qué coste de
latencia? Y, de paso, ¿su score separa relevante de no relevante lo bastante como para un threshold?
(anclaje abierto desde S09).

**Dentro de alcance:** default de expansión (G1), wrapper de reranker local (G2–G4), configuraciones
del arnés (G5), latencia (G6), separabilidad del score (G7), ronda de modo B.

**Fuera de alcance:** cablear el reranker en `/query`, reranker hospedado o con LLM, cambiar
`RETRIEVAL_TOP_K`, añadir PyTorch a la imagen Docker, tocar `corpus/`. Si algo lo exige:
`# TODO: pendiente de decisión en Claude Chat`.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| G1 | `RETRIEVAL_EXPANSION` default → **`orphans`** | Elimina las notas huérfanas (6 → 0) con 7 chunks en 14 peticiones y sin coste de tokens medible. `bounded` queda como opción: si +36 % de contexto ayuda o diluye es S11 |
| G2 | Modelo: **`cross-encoder/ms-marco-MiniLM-L-6-v2`** vía `sentence-transformers`, CPU, **revisión fijada**. Dependencias en un **grupo de desarrollo/opcional** de uv, importadas de forma perezosa: `app/` no las importa | Corpus y queries en inglés → el monolingüe vale. Ligero: si el más barato no gana, sabemos más antes de pagar por otro. Sin PyTorch en la imagen hasta decidir |
| G3 | **Unidad = chunk**: el reranker reordena los chunks del pool y se quedan los `top_k` (10); después, la agrupación por cláusula de siempre | Los chunks caben (~512 tokens por diseño) en la ventana del modelo; una cláusula entera no. Se mantiene el mismo `top_k` para comparar contra `vector_filter` en igualdad |
| G4 | **Dos variantes del texto que ve el reranker:** `plain` (`cite_text`) y `header` (`"<estándar> · <cláusula> · <título>\n" + cite_text`) | El cross-encoder solo sabe lo que se le pasa: sin cabecera no sabe a qué documento ni cláusula pertenece un chunk |
| G5 | La **query del reranker es la de retrieval de producción** (pregunta + historial compuesto) | Misma regla que D8 del brief 1: medir el camino real. Si en `multi_turn` resulta dañino, se reporta |
| G6 | El wrapper **se carga una vez**, recibe y devuelve el mismo tipo de evidencia, y **conserva el score** en el resultado | Patrón de producción del artículo: etapa componible y trazable. El score es lo que permitiría un threshold |
| G7 | Se cuenta cuántos pares (query, chunk) **superan la longitud máxima del modelo** con su tokenizador real (truncado silencioso) | Mide el riesgo de truncado y, de paso, contrasta el contador ~4 chars/token con un tokenizador real (deuda abierta de S09) |

---

## 2. Configuraciones

Base: el retrieval por defecto (F1: `vector` + `filter`). Expansión irrelevante aquí: no cambia qué
cláusulas llegan.

| Config | Pool | Texto |
|---|---|---|
| `vector_filter` | — | — (baseline, sin reranker) |
| `rerank50_plain` / `rerank50_header` | 50 | plain / header |
| `rerank200_plain` / `rerank200_header` | 200 | plain / header |
| `rerank400_plain` / `rerank400_header` | 400 | plain / header |

Métricas del arnés de siempre sobre el **golden set v2** (14), por estrato, más:

- **Latencia del reranker** (aparte de la del retrieval): mediana de 5 en caliente por tamaño de
  pool, con el hardware anotado. Además: tiempo de carga del modelo y RAM residente.
- **Denominador:** TTFT actual de `/query/stream` con los defaults, mediana sobre las 14 consultas.
  Gasta llamadas reales (pocas): **pregunta al usuario antes**. El reranker se lee como fracción de
  ese TTFT.

---

## 3. Separabilidad del score (anclaje del threshold)

Con los juicios del golden set, para cada configuración `rerankN_*`:

- Distribución del score de chunks de cláusulas **relevantes** frente a **no relevantes** (los
  juzgados), global y por consulta.
- **AUC** global y **mediana de AUC por consulta**: la segunda dice si un umbral *dentro* de una
  consulta separa; la primera, si un umbral *fijo* entre consultas serviría.
- Si existiera un umbral fijo útil: qué fracción de no relevantes eliminaría y cuántos relevantes
  perdería.

Sin decisión aquí: es la evidencia para el anclaje "threshold por chunk".

---

## 4. Trabajo

1. G1 en settings, `.env`, `.env.example`, `CLAUDE.md`, `ARCHITECTURE.md`.
2. Wrapper del reranker (dónde vive: CC decide; debe ser utilizable desde el arnés sin que `app/`
   importe `sentence-transformers`).
3. Configuraciones de §2 en el registro del arnés.
4. Medición → pares sin juzgar de todas las configuraciones en **una sola exportación** → pausa para
   modo B → import → re-medir hasta válidas.
5. TTFT baseline (tras autorización del usuario), latencias, §3.
6. Tests de contrato (§5).

---

## 5. Tests de contrato (sin red, sin BD, sin modelo real)

Como mínimo:
- Wrapper con modelo falso: ordena por score, respeta `top_k`, lista vacía, conserva el score,
  mismo tipo de entrada y salida, una sola carga del modelo.
- Formato `header` exacto; `plain` = `cite_text`.
- Contador de truncado con un tokenizador falso.
- AUC sobre casos construidos (separación perfecta, nula, empates).
- `app/` no importa `sentence_transformers` (test de import); el default de expansión es `orphans`
  sin variables opcionales.
- La suite de contrato pasa **sin** el grupo de dependencias opcional instalado.

---

## 6. UAT (usuario)

Caso 1 (sin oráculo) y caso 2 (coste y decisión):
1. Autorizar las llamadas del TTFT baseline.
2. Sesión de anotación en modo B (protocolo 1.1) sobre la exportación agrupada.
3. Leer la tabla y §3. Pendiente aún de brief 3: la UAT de Streamlit de `X1`/`X2`.

---

## 7. Cierre

- `S10-decisions-4.md`: decisiones + porqué, etiquetadas **agnóstico/dominio**; tabla por estrato de
  §2; latencias (reranker, carga, RAM, TTFT); truncado (G7) y la comparación del contador con el
  tokenizador real; §3; y una **recomendación razonada** (no aplicada): ¿entra el reranker en
  producción?, ¿con qué pool y variante?, ¿cabe en el presupuesto de TTFT?, ¿hay base para un
  threshold?
- `CHANGELOG.md`, `CLAUDE.md`, `ARCHITECTURE.md` según sus reglas.
- `uv run pytest` en verde y mensaje de commit propuesto. El commit lo hace el usuario.
