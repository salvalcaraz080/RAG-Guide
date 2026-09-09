# S08 — Consolidado de sesión (histórico, no se reescribe)

> **Paso 5 del ciclo.** Combina el resumen provisional (paso 2) con los aprendizajes de implementar
> los cinco briefs. La **divergencia teoría↔medida** se conserva aquí a propósito: es el material de
> Guide. Es el registro histórico de S08; no se reescribe. La graduación a `RAG-GUIDE.md` (paso 6)
> se hace **desde** este documento.
>
> **Sesión:** M3 / S08 — Bases de datos vectoriales. **Cerrada con:** capa de datos pgvector +
> retrieval vector exacto + híbrida básica, medibles. Seam sin cablear (S09).

---

## 1. El arco de S08

Entramos a decidir sobre la marcha (sin importar defaults del máster) cuatro cosas: si añadir un
store vectorial y cuál, con qué índice, qué schema, y —traído desde el ejercicio— si construir ya la
híbrida. Salimos con: **pgvector** (por joins+híbrida+trazabilidad en un solo sistema con ACID, no
por el stack del máster); **régimen exacto** de arranque (recall 1.0 por trazabilidad, baseline de
medición, menos complejidad operativa) en divergencia consciente con el HNSW del máster; **schema de
dos tablas con columnas tipadas load-bearing para la cita**; e **híbrida básica** (léxico + RRF)
traída a S08 porque la evidencia ya existía y el léxico-en-el-mismo-sistema es la razón de elegir
pgvector. La implementación confirmó el grueso, refinó algunas cosas y **refutó dos predicciones** —
que es donde está el valor.

---

## 2. Decisiones contrastadas con la implementación

### Confirmadas

- **Store = pgvector.** El JOIN de filtros tipados + vector en una sentencia atómica se ejecutó y se
  midió (§8.1 del Brief 4: el filtro reduce filas *dentro* del scan). Es *el* argumento, ahora
  operativo.
- **Régimen exacto correcto a nuestra escala.** Recall 1.0 verificado por cross-check contra numpy
  brute-force (Δd 5,6e-8–1,5e-7, float32/64), y baseline **65/69 ms** en base, holgadamente bajo el
  SLO. El `Seq Scan` es correcto, no bug.
- **Coseno + `vector_cosine_ops` + `<=>`** por robustez frente a des-normalización; operador elegido
  hoy aunque hoy no cambie nada, con test que verifica que los otros dos no aparecen.
- **`embed≠cite`** materializado tres veces: dos columnas en el schema, ausente del `SELECT` de
  `/search`, y el índice léxico sobre `cite_text` (no `embed_text`). De convención a invariante.
- **HNSW-no-IVFFlat si/cuando salgamos de exacto** (degradación silenciosa de IVF con corpus
  creciente). No se ejerció; queda como nodo con disparador.

### Refinadas

- **Disparador exacto→HNSW, ahora con número.** Se lee del tiempo de **base** (65/69 ms), no del
  extremo a extremo (266/298 ms) — los ~200 ms de diferencia son la llamada de embedding por red, que
  un índice no toca. Proyección a corpus completo (~60-100k): el seq scan rondaría 350-450 ms y **sí**
  cruzaría el SLO — pero es proyección lineal, no medida.
- **Fingerprint: no es "por embed_text" (rama A), es "cada stage digiere su propia entrada".**
  `chunk_all` digiere los `ParsedDocument`; `embed_all` digiere `embed_text`+modelo+dims. Añadir
  `requirement_uid` no movió ninguno. La pregunta "¿re-embebo?" tiene dueño propio.
- **`oversized` no es fidelidad de vector — es dilución.** Medido: los 2 casos (555/646 tok) están un
  orden de magnitud bajo el tope del embedder (8.191); sus vectores cubren el texto entero. La
  bandera señala representación menos afilada, no truncamiento. Señal de *calidad de recuperación*,
  con la semántica correcta en el modelo.

### Refutadas / corregidas por la medida

- **"El problema B/C es de ranking, no de recall" (Brief 4) — parcialmente falso.** Cierto para el
  discriminante de 40C (estaba en top-5, mal ordenado); **falso para la tabla D-1 de 80C**, que el
  vector dejaba en rango 6, fuera del top-5. Fue **recall** para ese ítem. Lección: en un mismo caso
  conviven fallo de ranking y de recall; hay que medir cuál, porque piden fixes distintos.
- **"La híbrida básica no discriminará B/C" (hipótesis del Brief 5) — refutada.** Sí discriminó:
  discriminante 40C 3→1, tabla D-1 80C recuperada de rango 6 al puesto 2. **Por qué**, contra el
  pronóstico: las tablas son largas y repiten los términos; `ts_rank` pondera por frecuencia y premió
  el desglose fila-a-fila. **La densidad de repetición resultó ser un proxy léxico de "esto desglosa
  el tema"** — la señal estructural que dábamos por no capturable tenía correlato léxico.
- **"El reranker es el fix probable de B/C" (Brief 4) — revisado.** El léxico bastó para B/C. El
  reranker queda para lo que el léxico **no** resuelve, que la medida reveló ser la **query
  genérica** (ver abajo).
- **La híbrida ayuda Y estorba.** Sube identificadores/tablas (B/C), pero degrada la genérica
  ("verification requirements": conserva 1/5 del vector; su nuevo #1 lo rankeaba el vector el 47).
  Causa: RRF sin pesos da al léxico el mismo voto siempre, y en una query genérica `ts_rank` ordena
  por densidad, que no correlaciona con relevancia. **El fix no es otro mecanismo, es modular el
  peso** — afinado de fusión, el encargo exacto de S10, ahora con evidencia medida.

---

## 3. Números medidos (que el resumen provisional no tenía)

- **Corpus:** 13.876 vectores medidos (26 docs, ~534/doc). Proyección corpus completo ~60-100k (×5-7;
  estimación, no medida). Árbol ECSS activo: ~121-139 estándares + handbooks/TMs.
- **Baseline latencia (régimen exacto):** base p50/p99 = **65,1 / 69,3 ms**; e2e 266,7 / 298,5 ms (gap
  ≈ embedding por red). `authority=normative` (10.320 filas) → base 45,2 ms (escala con filas).
- **Cross-check recall 1.0:** top-10 idéntico SQL vs numpy en las 5 queries; Δd 5,6e-8–1,5e-7.
- **Cobertura UID:** 2.536/7.922 requisitos (32%).
- **Composición:** authority 10.320 normative / 3.556 informative (~74/26). Anclas: requirement 7.922
  · section 2.583 · table 1.481 · note 1.410 · glossary_term 480. Fingerprint corpus `ab3fce8b5cb1a0d4`.
- **B/C híbrida:** discriminante 40C rango 3→1; tabla D-1 80C **6→2 (recuperada)**. Genérica: solape
  1/5, nuevo #1 = rango vectorial 47.
- **Distancia ≠ calidad (medido dos veces):** fuera-de-dominio (0,47-0,51) y genérico-en-dominio
  (0,43-0,47) se **solapan**; el peor arquetipo semántico (B/C) tiene las distancias **más bajas**; y
  en B/C la híbrida acierta mientras su distancia media **empeora** (0,2678→0,2926).
- **Datos:** 103 chunks con tsvector vacío (`cite_text` = `-` ×51, `<<deleted>>`-variantes ×52).
- Suite: 754 → 837 tests a lo largo de los 5 briefs. Carga completa 57 s.

---

## 4. El hilo conductor de S08: fallos silenciosos y guardas en el origen

S08 fue, en buena parte, una caza de **resultados bien-formados-y-falsos** — el modo de fallo más
peligroso en un sistema cuyo valor es la trazabilidad, porque no hace ruido. Inventario de lo que
apareció y cómo se cerró:

- **Emparejamiento fila↔vector** (Brief 3): si el orden se rompe, cada chunk recibe el vector de otro
  → base bien-formada-y-falsa, con citas impecables y equivocadas. Guarda de orden explícita; la única
  cuyo fallo era silencioso.
- **Espacio de embedding de la query** (Brief 4): otro modelo/dims → "ruido con forma de resultado".
  Guarda de dim en el origen.
- **Filtrar fuera del top-k** (Brief 4): pides 5, recibes 1 sin aviso. Filtro dentro de la sentencia.
- **AND por defecto en el léxico** (Brief 5): léxico devuelve 0 → RRF degenera a vector → híbrida
  idéntica a vector *pareciendo que corrió*. OR + descartar and/or/not de la frase del usuario.
- **Directorio compartido entre stages** (Brief 2): artefacto intruso entra en el digest como corpus.
  Detectado ruidoso por suerte; el modo silencioso habría invalidado chunks sin cambiar un documento.
- **La distancia como criterio** (Briefs 4-5): no está calibrada; usarla como gate o como aceptación
  concluiría cosas falsas.

Patrón: **la guarda va donde se produce el fallo, no donde se nota**, y **un segundo canal audita al
primero** (el léxico encontró los 103 marcadores basura del índice vectorial).

---

## 5. Abiertos y anclas para S09 / S10

- **S09 — gate "no lo sé":** no puede basarse en distancia (§3, medido). Va en el generador (como ya
  hace M1).
- **S09 — ¿el mal ranking es load-bearing?** Si la generación recibe top-k y el chunk correcto está
  dentro, el desorden puede quedar absorbido. Medir si el generador responde bien B/C pese al orden —
  decide cuánta inversión en híbrida/rerank justifica el producto de generación (vs búsqueda pura).
- **S09 — normalización de query a inglés:** diferida; el baseline se midió con queries en inglés.
- **S10 — afinado de fusión (peso del léxico):** el encargo, ahora con evidencia: léxico ayuda en
  identificadores/tablas, estorba en genéricas. RRF sin pesos es un suelo, no la respuesta.
- **S10 — rol del reranker:** abierto. El léxico bastó para B/C; el reranker es para lo que no resuelve.
- **S10 — config de text-search / criticidad tipada:** no hizo falta para B/C con `english` básico;
  candidata si aparece un caso que la densidad no cubra.
- **S09/S10 — grafo-vs-metadata:** sigue abierto; con la variante `references` como tabla Postgres
  join-expandible. El contenido discriminante de B/C estaba en chunks `table` — engancha con el hilo
  de referencias tabulares de S06.
- **Congelado:** `corpus/` salvo lo tocado en Brief 2; `halfvec` e índice HNSW (nodos con disparador).

---

## 6. Deuda

- **Dominio:** 103 chunks marcador de cláusula retirada (`-`/`<<deleted>>`) son recuperables por el
  canal vectorial. Decisión de chunking pendiente: ¿debe existir un chunk cuyo texto dice que no hay
  texto?
- **Endurecimiento (brief propio, tras S08):** (a) refactor del manifiesto en los 6 módulos que
  escanean `data/processed/` con lista de exclusión (§5.1 Brief 2); (b) acoplamiento de `ingestion.py`
  —job offline— a `ANTHROPIC_API_KEY` vía `app.config.Settings` (§5 Brief 3). Principio: un job offline
  no debe depender de una credencial online.
- **Proceso (¿regla para CC?):** dos fallos en la **frontera del terminal** — comillas de PowerShell
  (Brief 1 §4.5) y `→`/Unicode en cp1252 (Brief 3 §7.2). pytest captura salida y no ejerce el terminal
  real, así que el camino de error se escapa a UAT. Candidato a regla: smoke-test de rutas de salida/
  error en el shell objetivo antes de entregar.
- **Higiene menor:** `[0.26.0]` duplicado en CHANGELOG; huérfano `rag-ecss-redis`; `EMBEDDING_PROVIDER`
  sin lectores. Ninguna de este brief; anotadas.

---

## 7. Candidatos a la Guide (feedstock para el paso 6 — graduar con filtro de altitud)

Ordenados por fuerza; algunos se fundirán o caerán al graduar.

1. **Exacto-vs-ANN con disparador medido.** Condición: escala; señal: p99 de base > SLO (leído del
   número de base, no del e2e). A baja escala + requisito de recall total → exacto. Alternativa (ANN)
   descartada con porqué. **HNSW-no-IVFFlat** si el corpus crece (degradación silenciosa de IVF).
2. **Store dedicado vs Postgres bajo trazabilidad:** cuando la coherencia vector↔metadato es
   correctitud (no solo rendimiento), el ACID de un solo sistema pesa más que el filtrado superior de
   un dedicado. Filtrar **dentro** del top-k (no después) es el corolario operativo, con su modo de fallo.
3. **La distancia no es confianza calibrada** — ni puntual ni agregada. No hay umbral que separe
   fuera-de-dominio de genérico-en-dominio; el gate va en el generador. Corolario: no convertir
   distancia en `score`; fundir por **rango** (RRF), no por esa escala.
4. **Recall-en-k vs ranking vs calidad de respuesta** son tres cosas distintas; medir cuál falla
   (un mismo caso puede tener recall y ranking rotos a la vez) porque piden fixes distintos.
5. **Híbrida:** la densidad de repetición léxica es un proxy sorprendente de "este fragmento desglosa
   el tema" en corpus tabular; y la híbrida ayuda donde el vector falla y estorba donde acertaba →
   RRF sin pesos es un suelo; el valor está en **modular el peso**.
6. **Fingerprints por-entrada-de-stage:** cada pregunta de invalidación (¿re-parseo/re-chunkeo/
   re-embebo?) con su propia clave que digiere lo que su consumidor consume.
7. **Manifiesto sobre lista-de-exclusión** para un directorio compartido entre stages (responde "qué
   hay aquí" por declaración, no por omisión).
8. **Meta — guardas de fallo silencioso en el origen:** en un sistema de trazabilidad el enemigo es el
   resultado bien-formado-y-falso; la guarda va donde se produce, no donde se nota; un segundo canal de
   recuperación audita al primero.
9. **`embed≠cite` como invariante propagado** (schema, borde del API, índice léxico) y la
   diferencia entre convención e invariante (quitarlo del `SELECT`, no solo del contrato).
10. **OR-no-AND en el canal léxico** como fallo silencioso concreto (AND → 0 filas → híbrida
    disfrazada de vector).
11. **Menores:** operador index-ready; columna generada STORED (deriva imposible por construcción);
    filtro imposible → 400 no vacío (un cero ambiguo enseña a desconfiar de los ceros); test atado a
    propiedad y no a artefacto.
