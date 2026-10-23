# S12 — Brief 1: diagnóstico de forma sobre los `not_found` de la referencia

> Primer brief de contenido de S12. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar.
> Rama: `s12-agentes` (ya contiene pre-S12). Los commits los hace Salva.
>
> **Regla del brief:** medir, no construir. Este brief no añade tools, bucles, expansión de
> referencias ni fan-out al producto. Produce el **instrumento** que decide si S12 tiene algo que
> orquestar y de qué tipo. Si durante el análisis una solución parece evidente, se anota como
> hipótesis en `S12-decisions.md`; no se implementa.

---

## 0. Contexto y objetivo

**Por qué este brief.** M5 (S12, agentes) plantea si el pipeline fijo de RAG-ECSS
(`retrieve → generate_structured → verify`) necesita una capa de decisión. La teoría de la sesión
deja este criterio (ver `S12-resumen.md` §1 y §8):

- **Forma variable pero predecible al leer la pregunta** (comparativa, aplicabilidad) → workflow con
  plan y fan-out en código. Sin bucle.
- **Forma reactiva** (el siguiente paso depende de lo que se observa: cadena de referencias cruzadas)
  → expansión determinista de referencias o, solo si no basta, bucle con tools.
- **Ninguna de las dos** → no hay nada que orquestar.

**Qué sabemos de la referencia `generation_20261019-141324`.** El recall de 0,91 no mide la forma:
es *relevante citado / relevante inyectado*, es decir, omisiones del generador sobre su propio
contexto. La señal de forma está en otro sitio: **33 de 42 ejecuciones se sirven como `partial`** y
hay **65 aspectos `not_found`**. Una lectura provisional del chat los reparte en tres grupos, sin
verificar contra el corpus:

- **A.** fuera del corpus;
- **B.** en el corpus pero no inyectado;
- **C.** ECSS no lo dice.

**Objetivo.** Clasificar cada aspecto `not_found` contra el corpus, **con evidencia**, para que el
chat decida el brief 2 con datos.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| D1 | La clasificación la hace CC con búsqueda dirigida en el corpus; Salva la verifica en la UAT | Decidido en el chat. Es un juicio, así que cada clasificación lleva evidencia literal verificable |
| D2 | La unidad es el **aspecto canónico por consulta**, no la línea del informe. Los aspectos parafraseados entre las tres ejecuciones de una consulta se agrupan, y se conserva el mapeo a las líneas originales | Las tres ejecuciones repiten el mismo hueco con palabras distintas; contar líneas infla la señal |
| D3 | Taxonomía cerrada (§2), con una categoría para lo que **sí estaba inyectado** | Un `not_found` sobre algo que estaba en el contexto es omisión del generador, no un problema de forma; mezclarlo contaminaría la decisión |
| D4 | Herramientas **deterministas**: retrieval con variantes, búsqueda léxica en `data/processed`, consulta por ID de cláusula como script. Sin llamadas de generación | Medir la forma con un LLM introduciría la variable que se quiere medir |
| D5 | La regla de decisión (§4) queda **fijada antes de ver los datos** | Disciplina de criterio preregistrado: el chat no puede ajustar el umbral al resultado |
| D6 | `MAX_HISTORY_TURNS` exige `ge=1` en `config.py` | Decidido tras pre-S12. Con 0, `history[-0:]` devolvía la lista entera. Para trabajar sin historial ya existe la pregunta suelta |

---

## 2. Taxonomía

Cada aspecto canónico recibe **una** categoría, una confianza (`alta` / `media` / `baja`) y
evidencia.

| Cat. | Significado | Evidencia mínima |
|---|---|---|
| **G** | **Estaba inyectado.** El contenido que responde el aspecto estaba en el contexto de esa ejecución | `chunk_id` inyectado + cita literal que lo responde |
| **B1** | En el corpus, no inyectado, **alcanzable por referencia cruzada explícita** desde un chunk que sí se inyectó (el chunk inyectado nombra la cláusula, el anexo o el estándar donde está la respuesta) | Chunk inyectado que referencia + cláusula destino + cita literal de ambos |
| **B2** | En el corpus, no inyectado, **alcanzable por cobertura por estándar o faceta**: aparece al recuperar con la misma pregunta restringida a otro estándar, o con una subconsulta por faceta (p. ej. "categoría C" por separado de "categoría B") | Variante de retrieval que lo trae (filtro, subconsulta) + rango + cita literal |
| **B3** | En el corpus, no inyectado, **problema de ranking**: lo trae la misma consulta y el mismo filtro con `top_k` ampliado (p. ej. rango 11–50) | Rango con la consulta original + cita literal |
| **B4** | En el corpus, no inyectado, y no lo alcanza ninguna de las anteriores (vocabulario muy distinto, tabla, chunk mal partido…) | Cláusula + cita literal + por qué no lo alcanza |
| **A** | **Fuera del corpus.** La respuesta está en un estándar que no forma parte del subconjunto | Idealmente, una cláusula del corpus que remite a ese estándar (p. ej. "see ECSS-Q-ST-30-02"). Si no la hay, la confianza baja |
| **C** | **ECSS no lo dice.** Las cláusulas que tratan el tema no lo especifican | Las 1–3 cláusulas que cubren el tema y no lo dicen + por qué el aspecto excede lo que el estándar regula |

Reglas de desempate:
- Si un aspecto es B1 **y** B2, cuenta como **B1**: la referencia explícita es la señal más
  específica.
- Si el aspecto está **parcialmente** cubierto (el corpus responde una parte), se clasifica por la
  parte no cubierta y se anota la parcialidad.
- Si el propio aspecto es dudoso (el modelo pidió algo que la pregunta no requería), se marca
  `aspecto_espurio: true` y se clasifica igualmente. Esto también es señal para el consolidado.

---

## 3. Trabajo

1. **Fuente.** `data/groundtruth/generation/runs/generation_20261019-141324.json` (artefacto
   inmutable, no se modifica). De ahí salen, por ejecución, los aspectos `not_found` y los
   `chunk_id` inyectados. Si los inyectados no estuvieran en el artefacto, reconstrúyelos re-ejecutando
   **solo el retrieval** con los defaults vigentes y regístralo como desviación.
2. **Agrupación (D2).** Aspectos canónicos por consulta, con su mapeo a las líneas originales y en
   cuántas de las 3 ejecuciones aparece cada uno.
3. **Clasificación (§2)**, por este orden en cada aspecto, porque cada paso es más barato que el
   siguiente:
   1. G (¿estaba en lo inyectado?);
   2. búsqueda léxica y por cláusula en el corpus (¿existe?) → si no existe, A o C;
   3. si existe: B1 (¿lo nombra algún chunk inyectado?) → B3 (`top_k` ampliado, misma consulta) →
      B2 (filtro por otro estándar o subconsulta por faceta) → B4.

   Las variantes de retrieval de B2 se registran literalmente (consulta, filtro, `top_k`), para poder
   repetirlas.
4. **Consulta por ID de cláusula** (para B1 y la evidencia): como **función de script** en
   `scripts/` o `app/eval/`, no como capacidad del producto. Si sale útil, el chat decide en el brief 2
   si se promociona a primitiva (`get_clause`).
5. **D6:** `MAX_HISTORY_TURNS: int = Field(10, ge=1)` (o la forma que use `config.py`), con un test de
   que 0 se rechaza al arrancar.
6. **Salida:**
   - `data/groundtruth/generation/diagnostics/notfound_20261019-141324.json`: aspecto canónico →
     categoría, confianza, evidencia (IDs + citas literales), variantes de retrieval, líneas
     originales;
   - un `.md` hermano legible, con:
     - **tabla resumen** por categoría (aspectos y **consultas distintas** afectadas);
     - la misma tabla por estrato (`comparative`, `cross_standard`, `names_standard_xref`…);
     - lista de estándares ausentes que aparecen en A (señal para una futura ampliación del corpus,
       que sigue congelado);
     - detalle por consulta.
   - **Inmutable** como los demás artefactos: si se rehace, archivo nuevo con sello de tiempo.

---

## 4. Regla de decisión (preregistrada; la aplica el chat, no CC)

Sobre **consultas distintas** afectadas (no sobre aspectos), con las 14 consultas del golden set v2:

| Resultado | Lectura | Brief 2 candidato |
|---|---|---|
| **B1 afecta a ≥ 3 consultas** | Hay señal de forma reactiva | Expansión determinista de referencias cruzadas + `get_clause`; el bucle solo si la expansión no basta. Medido con el arnés |
| **B2 afecta a ≥ 3 consultas** | Hay señal de forma predecible | Plan + fan-out por estándar/faceta en código (orchestrator-workers). Medido con el arnés |
| Ambas ≥ 3 | Las dos | Se prioriza la que afecte a más consultas; la otra queda candidata |
| **B3 domina** (B3 ≥ B1 + B2) | No es forma, es ranking | No hay brief agéntico de producto. Vuelven los diferidos de S10 (reranker, `top_k`) con esta señal |
| **G domina** | Omisión del generador | No hay brief agéntico. Señal de prompt o de modelo, que se mide aparte |
| B1 y B2 < 3 cada una | Sin señal suficiente | S12 cierra en producto con lo hecho (pre-S12 + instrumento). La capa de decisión queda diferida hasta tener la ampliación del golden set (protocolo 1.2) con estratos de referencia cruzada y comparativa |

El umbral de 3 consultas (~20 % del set) es deliberadamente bajo, porque el set es pequeño (2
consultas por estrato). Pasarlo es una señal para construir y medir, no una prueba. La aceptación de
cualquier solución sigue siendo el arnés con un criterio preregistrado.

**CC no aplica la regla ni recomienda el brief 2**: reporta la tabla con evidencia. Si alguna
clasificación le parece frágil, la marca con confianza `baja` en lugar de forzarla.

---

## 5. Llamadas reales y coste

- **Generación: ninguna.**
- **Embeddings de consulta** para las variantes de retrieval (si el embedding es por API): autorizadas.
  Coste despreciable (decenas de consultas cortas). Registra el número.
- Si algo exigiera una llamada de generación, **para y reporta**.

---

## 6. Tests de contrato (sin red)

- `uv run pytest` en verde. Recuento frente a los **1296** de partida.
- Test de D6 (`MAX_HISTORY_TURNS=0` se rechaza).
- Si la consulta por ID de cláusula o la agrupación se escriben como funciones reutilizables, sus
  tests unitarios con fixtures del corpus procesado (sin BD, o con la excepción de red declarada
  según pre-S12 si fuera imprescindible: regístralo).
- El análisis en sí no es un test: es un script que produce el artefacto.

---

## 7. UAT (usuario)

Propón el script de UAT. El núcleo es que Salva **verifique la clasificación**, no el código:

1. **Todas las B1 y B2** (las que pueden disparar trabajo): abrir la evidencia y confirmar que
   (a) la cláusula responde el aspecto y (b) la ruta (referencia o variante) es real.
2. **Todas las de confianza `baja`.**
3. **Una muestra de A, C y G**: al menos una por consulta afectada o 5 por categoría, lo que sea
   menos.
4. Para cada desacuerdo, anotar la categoría que pondría Salva. El chat aplica la regla de §4 con la
   clasificación **corregida**.

Formato sugerido: una tabla por aspecto con la columna "¿De acuerdo? / categoría corregida", para que
la corrección se lea directamente.

---

## 8. Cierre

- `S12-decisions.md` en `RAG-Guide/sessions/S12 - Agentes/`: desviaciones (si hubo que reconstruir
  inyectados), criterio de agrupación, decisiones de clasificación no obvias, hipótesis de solución
  anotadas sin implementar, recuento de tests, etiquetado agnóstico/dominio.
- `S12-uat.md` con el script de UAT.
- Docs: `ARCHITECTURE.md` / `.claude/CLAUDE.md` solo si cambia algo del estado del repo (el script
  de diagnóstico, D6). `CHANGELOG.md` derivado.
