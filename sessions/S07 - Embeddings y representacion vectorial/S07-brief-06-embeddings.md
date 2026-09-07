# S07 · Brief 6 — Generación de embeddings (objetivo de la sesión)

> **Para:** Claude Code. **Produce:** los vectores de los 13.876 chunks, generados con
> `text-embedding-3-small` a través de un *seam* de embeddings, persistidos como **artefacto
> derivado** atado a `corpus_version` + fingerprint del stage, con un sanity check de discriminación
> de dominio. **NO produce:** carga en vector store / pgvector (eso es S08), embedding de queries
> (S09+), abstracción multi-proveedor completa (Phase 4).
>
> **Contrato de trabajo:** CC no decide arquitectura; marca `# TODO: pendiente de decisión en
> Claude Chat` y sigue. Tests de contrato mockeados (sin red). La **generación real sobre el corpus
> y el sanity check corren con la API real → UAT del usuario** (CC entrega los scripts, el usuario
> los ejecuta). Commits los hace el usuario.

---

## 0. Contexto — el objetivo de temario de S07

Los cuatro briefs previos dejaron el corpus indexable y citación-correcto: 13.876 chunks con
`embed_text` (header + fuente) y `cite_ref` (verbatim + coordenada). Este brief cumple el objetivo
nominal de la sesión: **convertir esos `embed_text` en vectores.**

Decisiones ya tomadas (no reabrir):
- **Modelo: `text-embedding-3-small`, dimensiones por defecto (1536).** Vigencia verificada a
  septiembre 2026 (sigue siendo el default sensato, sin reemplazo OpenAI más nuevo). Reversible por
  construcción: vive detrás del seam + fingerprint.
- **Seam, no wrapper completo.** La abstracción multi-proveedor es Phase 4; aquí basta la función
  frontera que aísla al proveedor (patrón de Phase 1 de la Guide).
- **Corte con S08:** este brief genera los vectores como **artefacto derivado**; S08 los carga en
  pgvector. Este brief **no toca pgvector**.

---

## Task 1 — El *seam* de embeddings

Una función frontera que aísla al proveedor, análoga al seam de LLM de Phase 1:

```python
def embed_texts(texts: list[str]) -> list[list[float]]:
    """Frontier for corpus embedding. Isolates text-embedding-3-small.
    NOT a multi-provider abstraction (that's Phase 4) — one provider, one place."""
```

- Toda llamada a la API de embeddings pasa por aquí; nada de `client.embeddings.create(...)`
  disperso. Es el punto de inserción de la abstracción futura — se rellena el cuerpo, no se mueve.
- **Es el mismo seam que usará el embedding de query** (S09+): embebe *texto*, sin saber si es
  corpus o query. No construir el lado query aquí, pero no cerrarle la puerta.
- Esto es un **build batch**, no el hot-path del servidor. La corrección async de Phase 1 (cliente
  nativo async) **no aplica** aquí — un script de build secuencial/por lotes está bien; lo que
  importa es el batching (Task 2), no la concurrencia del event loop.

---

## Task 2 — Script de generación (batch sobre los chunks)

- Lee los chunks del artefacto del Brief 4; embebe el campo `embed_text` de cada uno. Uniforme
  sobre **todas** las anclas (requirement/section/table/note/glossary_term) — el embedding es
  agnóstico al ancla; el ancla es metadata que viaja con el vector, no cambia la generación.
- **Batching:** la API acepta arrays de inputs por request. Agrupar chunks en lotes, no una llamada
  por chunk (13.876 llamadas sería lento y chocaría con rate limits). Respetar el **cap de 8.191
  tokens por input**: un `embed_text` que lo exceda es una **señal a reportar** (no debería ocurrir
  —las secciones grandes ya se sub-parten en Brief 4— pero si aparece, se marca, no se trunca en
  silencio).
- **Batch API (opción, no obligación):** 50% de descuento, ventana 24h, ideal para indexado inicial.
  A nuestra escala el coste es de céntimos, así que el Batch importa por rate limits, no por ahorro.
  Dejarlo como flag; el default puede ser síncrono dado el volumen.

---

## Task 3 — Artefacto derivado, atado a su versión

Persistir los vectores como artefacto derivado (p. ej. `data/processed/embeddings.*`). Formato a
criterio de CC con estas restricciones (documéntalo):
- **Eficiente para floats:** 13.876 × 1536 float32 ≈ 85 MB. JSON de floats es inviable (tamaño y
  parseo); usar binario (`.npy`/parquet) para los vectores + un sidecar que mapee `chunk_id` ↔ fila.
  Que S08 pueda cargarlo a pgvector con facilidad.
- **Estampado de versión** (el encadenamiento del fingerprint, decisión B):
  - `corpus_version` (etiqueta legible del conjunto).
  - el `input_fingerprint` de los chunks que consumió (enlaza estos vectores a *qué* chunks).
  - `model` (`text-embedding-3-small`), `dimensions` (1536), y cualquier param.
  - su propio **fingerprint de stage** = digest sobre (`embed_text`s consumidos + model + params).
- `is_stale()` para embeddings: `True` si el digest de los `embed_text` actuales ≠ el registrado, o
  si model/params cambiaron; `None` si no hay artefacto o no lleva fingerprint (el "no lo sé" no se
  disfraza de "está bien", como en Brief 4).

---

## Task 4 — Contrato de forma y norma (y la honestidad sobre el determinismo)

- **Forma:** todo vector tiene longitud 1536; hay exactamente un vector por chunk; el mapeo
  `chunk_id` ↔ vector es total y sin huecos (mismo principio de conservación que "sin fuga
  processed→chunks": ningún chunk se queda sin vector, ningún vector sin chunk).
- **Norma ≈ 1:** `3-small` devuelve vectores normalizados. Aseverarlo (candidato "true by
  construction → true by regression test"). Como usamos dimensiones por defecto, **no** hay truncado
  Matryoshka ni renormalización manual; si algún día se trunca, habría que renormalizar — no aquí.
- **⚠️ El determinismo NO es como el del chunking.** El chunk se testea por igualdad (mismo input →
  mismo output). La API de embeddings **puede devolver vectores con variación numérica mínima entre
  llamadas**, así que **no** se escribe un test de igualdad de vectores entre corridas — fallaría en
  verde. La garantía honesta es: *estos vectores salieron de este input (fingerprint), con este
  modelo+params.* El fingerprint atestigua la **identidad del input**, no la igualdad del output.
  Lo que sí es determinista y testeable es el fingerprint (mismo `embed_text`s → mismo fingerprint).

---

## Task 5 — Sanity check de discriminación de dominio (diagnóstico, lo corre el usuario)

**No** es evaluación formal de retrieval (eso es S11: recall@k, NDCG). Es el "verificar antes de
construir encima" aplicado al embedding: confirmar que el modelo **no es ciego a nuestra distinción
crítica** antes de dar los vectores por buenos y montar retrieval encima.

- CC entrega un script diagnóstico (`corpus/diagnostics/embedding_sanity.py`) que embebe un conjunto
  pequeño de **pares del dominio** y reporta la similitud coseno de cada uno. El usuario lo corre
  con la API real y revisa (no bloquea merge — es de la *suite de evaluación*, no de la de contrato;
  necesita modelo real y juicio humano, à la Guide).
- **El par trampa es obligatorio:** contenido de "criticidad B" vs "criticidad C" (superficie casi
  idéntica, cláusulas distintas). Construirlo de **chunks reales** (las cláusulas de niveles de
  criticidad de `ECSS-Q-ST-80`). La bandera roja: si ese par sale a similitud ~0.99, el modelo no
  los separa y el retrieval no podrá preferir B sobre C — es justo lo que queremos ver **antes** de
  embeber a ciegas, no después.
- Resto del conjunto (media docena en total): un par de pares que **deberían** ser similares
  (paráfrasis del mismo requisito) y un par que **debería** ser lejano (dos requisitos de temas
  distintos), para tener referencia de qué rango da el modelo en nuestro dominio. La lectura no es
  el número absoluto (maldición de la dimensionalidad: 0.2–0.5 entre no-relacionados es normal) sino
  la **estructura**: ¿el par trampa queda por debajo de los pares-similares, o colapsa con ellos?
- Si el par trampa colapsa, **es decisión de Chat** qué hacer (reconsiderar modelo, o apoyarse en
  híbrida/BM25 de S10 para la discriminación exacta de identificadores) — CC lo reporta, no lo
  resuelve.

---

## Tests (contrato, mockeados) vs sanity (evaluación, usuario)

Dos suites soberanas, como la Guide fija:
- **Contrato (CC, mockeado, sin red, en CI):** forma (1536, mapeo total chunk↔vector), lógica de
  batching (agrupa bien, respeta el cap de 8.191 con un input mock sobredimensionado), cómputo del
  fingerprint (mismo `embed_text`s mock → mismo digest; cambiar uno lo cambia), `is_stale` tri-estado.
  **La respuesta de la API se mockea** — no se llama al modelo en el contrato.
- **Evaluación (usuario, API real, no bloquea):** la generación real sobre el corpus, la norma ≈ 1
  observada, y el sanity check de dominio.

No escribir un test de contrato que compare vectores reales entre corridas (Task 4).

---

## UAT (usuario, con la API real)

- Correr la generación sobre los 13.876 chunks; confirmar mapeo total (0 chunks sin vector), normas
  ≈ 1, y el artefacto estampado con `corpus_version` + fingerprints + model + dims.
- Correr `embedding_sanity.py` y revisar: el par trampa criticidad B/C no colapsa con los
  pares-similares; la estructura (similares > trampa > lejanos) se sostiene.
- Reportar el resultado del par trampa a Chat (es el dato que decide si algo hay que reconsiderar).

---

## Al cerrar (CC)

- `S07-decisions-6-embeddings.md`: log crudo, etiquetado agnóstico/dominio. Sube al chat.
- Deriva `CHANGELOG.md` y **reescribe el estado presente** en `CLAUDE.md`/`ARCHITECTURE.md`: el seam
  de embeddings, el artefacto de vectores y su estampado de versión, el fingerprint de stage.
- Guion de UAT (generación real + sanity).
- Cualquier `# TODO`.

---

## Fuera de scope (no implementar)

- **Carga en vector store / pgvector** — S08. Este brief deja los vectores como artefacto derivado
  listo para que S08 los cargue.
- **Embedding de queries y normalización de query a inglés** — S09+ (el seam servirá, no se
  construye aquí).
- **Abstracción multi-proveedor completa** — Phase 4; aquí solo el seam de un proveedor.
- **Evaluación formal de retrieval (recall@k, NDCG, RAGAS)** — S11; el sanity check no es eso.
- **Batch API como obligación** — es opción; el coste es despreciable a esta escala.
- **Población del slot de refs** — sigue gateada por validación del extractor.
