# S10 — Brief 3: defaults, profundidad del pool, golden set v2 y expansión de cláusula acotada

> Continuación de `S10-brief-2.md`, sobre `S10-decisions-2.md`. Último brief de S10.
> Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar.

---

## 0. Objetivo y alcance

**Objetivo.** Cerrar S10: aplicar los defaults decididos, medir si el reranker tendría margen con un
pool más profundo, ampliar el golden set para cubrir el riesgo del filtro, y resolver el último
anclaje de S09 (cláusulas que llegan incompletas, nota huérfana como caso prioritario).

**Dentro de alcance:** §1 (defaults), §2 (profundidad), §3 (golden set v2), §4 (expansión).

**Fuera de alcance:** reranker (se diagnostica, no se implementa), pesos en RRF, expansión de
consulta, grafo de referencias, tablas, evaluación de la generación (S11), tocar `corpus/` o
re-embeber. Lo que parezca exigirlo: `# TODO: pendiente de decisión en Claude Chat`.

---

## 1. Defaults decididos en Claude Chat

| # | Decisión | Porqué |
|---|---|---|
| F1 | `RETRIEVAL_MODE=vector` + `RETRIEVAL_ROUTING=filter` pasan a **default** | p@k 0,68 → 0,82, más rápido, determinista; las consultas sin estándar nombrado no cambian |
| F2 | `RETRIEVAL_LEXICAL` default → **`tokenized_length`** | `legacy` tiene un defecto conocido y `/search?mode=hybrid` lo sigue sirviendo. `tokenized_length` ≈ BM25 en precisión (0,67 vs 0,69) sin estadísticas que mantener y ~100 ms más rápido |
| F3 | `bm25` y `quota` **se quedan como opciones** configurables; `legacy` se conserva solo si el arnés lo necesita para reproducir mediciones pasadas (CC decide y lo dice) | Alternativas medidas, sin evidencia para default |

Actualiza `.env.example`, `CLAUDE.md` (estado actual y decisiones pendientes: "Recuperar el estándar
nombrado" y "Ponderar la fusión híbrida" pasan a resueltas, con su porqué) y `ARCHITECTURE.md`
(incluida la nota de que el filtro es correcto en régimen exacto y dejaría de serlo con HNSW).

---

## 2. Diagnóstico de profundidad del pool (decide el futuro del reranker)

**Pregunta:** las relevantes que no están en los 50 primeros chunks, ¿están más abajo o no están?

- `R@cand` y `R@cand cap` con pool de **50, 100, 200 y 400** chunks, para `vector_filter` y
  `hybrid_tok_bm25_filter`, por estrato.
- Para cada profundidad: cláusulas candidatas únicas y latencia mediana de retrieval.
- No requiere anotación: R@cand solo cuenta relevantes conocidas.

**Salida:** curva recall-profundidad. Lectura que se espera en decisiones: si a 200 el R@cand cap
sube de forma sustancial, el reranker tiene margen (se diseña cuando toque); si se aplana, el límite
es de representación (embedding/chunking) y el reranker queda diferido **con esa señal registrada**.
Nada se implementa aquí.

---

## 3. Golden set v2

El tamaño fijo de 10 era arbitrario; se amplía donde la v1 dejaba huecos. La v1 **no se toca**: la
v2 la contiene y añade 4 consultas → **14**, número fijo a partir de ahora.

| Estrato | v1 | v2 | Por qué |
|---|---|---|---|
| `names_standard_xref` (nuevo) | 0 | **2** | Nombra un estándar cuya respuesta completa **necesita una cláusula de otro** al que remite. Es el riesgo del filtro (F1) y el caso de uso de la cuota, sin medir hasta ahora |
| `paraphrase` | 1 | 2 | Un solo punto por estrato no dice nada |
| `multi_turn` | 1 | 2 | Ídem; además es donde el routing cambió más (0,25 → 1,00) |

- **Protocolo 1.1** (en su sitio del repo): añade el estrato `names_standard_xref` con esta
  definición: *"Explicitly names one standard, and a complete answer requires at least one clause
  from a different standard that the named one refers to or depends on."* Y una regla de fase 1
  para ampliaciones: el anotador ve las consultas existentes (para no duplicar) y escribe **solo**
  las nuevas. El resto del criterio no cambia.
- El importador acepta una v2 que extiende la v1 (mismas 10 consultas y juicios, 4 nuevas) y
  registra la versión del golden set en cada medición.
- **Re-medir sobre v2** como mínimo: `vector`, `vector_filter`, `vector_quota`,
  `hybrid_tok_bm25_filter`. El informe da la tabla v1 (comparable con lo anterior) y la v2.
- **Lectura esperada en decisiones:** en `names_standard_xref`, ¿el filtro pierde la cláusula del
  otro estándar? ¿La cuota la recupera, y a qué coste en p@k? Si la cuota gana ahí sin perder en el
  resto, se reporta; el cambio de default lo decide Claude Chat.

---

## 4. Expansión de cláusula acotada

**Problema (S09):** el 70 % de las cláusulas entregadas llega incompleta; el caso prioritario es la
**nota huérfana** — una NOTE ("Refer also to clause X", aclaraciones) entregada sin el requisito de
su cláusula. La expansión completa se descartó (×4 tokens en mediana, cola de anexos DRD).

**Es una etapa de aumento**, no de selección: va después del último corte (las cláusulas ya
elegidas) y antes de construir el bloque de contexto. **No cambia qué cláusulas llegan**, solo
cuánto de cada una. Por eso el golden set no la mide: necesita métricas propias.

### 4.1 Configuraciones

| Config | Qué hace |
|---|---|
| `X0` | Sin expansión (actual) |
| `X1` — huérfanas | Si una cláusula entregada **tiene** chunks normativos pero **ninguno** entregado, se añaden sus chunks normativos |
| `X2` — completar acotado | Para cada cláusula entregada, se añaden sus chunks no entregados: **primero normativos**, luego el resto, hasta un **tope por cláusula** y un **tope global** de tokens extra |

Topes de partida (declarados, no calibrados): **600 tokens por cláusula**, **2.000 tokens extra
por petición**, contados con el contador vigente. Si no cabe todo, se priorizan las cláusulas por su
orden en el contexto.

### 4.2 Métricas (sobre las consultas del golden set v2 con la config default F1)

- **Completitud**: fracción de los chunks de cada cláusula entregada que llegan (media y mediana).
- **Completitud normativa**: ídem, solo chunks normativos.
- **Tasa de huérfanas**: cláusulas entregadas con chunks normativos existentes y ninguno entregado.
- **Coste**: tokens de contexto por petición (mediana y p95) frente a `X0`.
- **Cláusulas truncadas por tope**: cuántas quedan incompletas por el tope, y en qué documentos
  (¿anexos DRD?).

### 4.3 Requisitos de diseño

- La traza (`retrieved_context`) distingue **recuperado** de **añadido por expansión** en cada
  chunk. Sin esto, auditar una respuesta mezcla lo que el retrieval encontró con lo que se arrastró.
- Los chunks añadidos son parte de su cláusula: mismo `[n]`, citables, y la validación literal opera
  sobre el texto ampliado.
- Orden dentro de la cláusula: el del artefacto (limitación ya conocida, no se resuelve aquí).
- Detrás de configuración (`off` | `orphans` | `bounded`), **default `off`** hasta decidir con la tabla.
- Dónde vive la etapa: CC decide según el seam actual (proveedor vs `llm_service`) y lo justifica.

---

## 5. Tests de contrato (sin red, sin BD)

Como mínimo:
- F1–F3: defaults efectivos sin variables opcionales; `legacy` (si se conserva) solo accesible
  explícitamente.
- §2: el arnés acepta profundidad de pool configurable.
- §3: importador de v2 (extiende v1 sin alterar juicios existentes; rechaza si una consulta v1
  cambia); estrato nuevo reconocido; versión del golden set en el informe.
- §4: X1 añade solo normativos de cláusulas huérfanas; X2 respeta ambos topes y la prioridad
  normativa; ninguna configuración cambia el conjunto de cláusulas; la traza marca lo añadido; la
  validación literal casa con texto de un chunk añadido; `off` = camino actual.

---

## 6. UAT (usuario)

Caso 1 (sin oráculo) y caso 2 (decisión):
1. Sesión de anotación (protocolo 1.1): fase 1 de **ampliación** (4 consultas nuevas, apruebas),
   fase 2 sobre las nuevas, y modo B de los pares sin juzgar de §3 — CC agrupa todo para hacerlo en
   una sola sesión si es posible.
2. Leer: curva de profundidad (§2), tabla v2 filtro vs cuota (§3), tabla de expansión (§4).
3. Una respuesta real en Streamlit con `X1` y otra con `X2` sobre una pregunta con nota huérfana
   (CC elige la pregunta del golden set): ¿se lee mejor? ¿la traza deja claro qué se añadió?

---

## 7. Cierre

- `S10-decisions-3.md`: decisiones + porqué, etiquetadas **agnóstico/dominio**, con las tres tablas
  y una **recomendación razonada** (no aplicada) sobre: default de expansión y topes, filtro vs
  cuota a la vista de `names_standard_xref`, y la señal para el reranker.
- `CHANGELOG.md`, `CLAUDE.md`, `ARCHITECTURE.md` según sus reglas.
- `uv run pytest` en verde y mensaje de commit propuesto. El commit lo hace el usuario.
