# S07 — Consolidado: Embeddings, chunking e identidad de la capa de datos

> **Estado:** histórico. No se reescribe. Registra lo que S07 decidió *y* lo que enseñó
> implementar. La divergencia teoría-vs-implementación es material de Guide y aquí se conserva.
> El resumen (`S07-resumen.md`) tiene la teoría destilada pre-implementación; este documento la
> asume y se centra en lo que la implementación añadió, contradijo o descubrió.

---

## 0. Tesis de la sesión

S07 tiene un hilo único del que cuelga casi todo lo demás, y que la generación de embeddings —el
último brief— generalizó a su forma más amplia:

> **Mide la propiedad real de cada capa antes de construir la siguiente encima. Cada capa tiene un
> modo de fallo silencioso que solo un probe dirigido —no una suite en verde— caza.**

En la capa de parseo, esa propiedad es la fidelidad **estructural**: un parser puede tener todos
sus invariantes en verde y aun así afirmar algo distinto de lo que el documento dice, porque el
documento significa con estructura visual (numeración, sangría, anidamiento, qué párrafo cualifica
a cuál) que el texto plano no lleva. Verificar contra el render no es verificar contra el texto.
Los cinco briefs de identidad fueron, en el fondo, la persecución de esa brecha: el hueco del 60%
de prosa, los estilos estructurales tratados como prosa, las listas aplanadas, los cuadros de
texto, el contador de lista sobre borrados. Ninguno lo habría cazado un test de igualdad de
cadenas.

En la capa de embedding, esa propiedad es la **discriminación de la representación**: un embedding
puede generarse sin errores —forma correcta, normas ≈ 1, mapeo total— y aun así ser ciego a la
distinción que el sistema tiene que hacer. El par trampa lo midió *antes* de indexar: "criticidad
B" vs "criticidad C", que difieren en 2 tokens de 42, salieron a 0.98 — no colapsaron *con* los
pares similares, los superaron. El denso comprime significado y descarta la identidad léxica
exacta; nuestra distinción crítica vive justo ahí. Ese probe, hecho antes del retrieval, separó
"el modelo no puede" de "mi retrieval está mal configurado" — dos diagnósticos que en S11 se
habrían confundido y costado semanas.

Dos verdades de encuadre que el consolidado declara sin adornos:

1. **S07 cumplió su objetivo de temario (embeddings generados), pero estuvo dominada por la capa de
   datos.** El material prometía embeddings y selección de modelo; la realidad del corpus obligó a
   arreglar los cimientos del parseo antes de poder embeber nada honesto — cinco briefs de fidelidad
   e identidad antes del sexto, que era el objetivo. Material de Guide: *en un corpus normativo real,
   la calidad del chunking está gateada por la fidelidad del parseo, y esa fidelidad puede ser un
   agujero invisible que consume la sesión entera.*
2. **El hallazgo más consecuente de S07 para la arquitectura no fue una decisión de chunking, fue el
   par trampa:** convirtió la búsqueda híbrida de "buena práctica que se asume" en **requisito
   medido, con el caso concreto que lo justifica** (S10). La teoría dice por qué (el denso descarta
   el léxico exacto); el par trampa lo confirma sobre nuestro texto real.

---

## 1. Teoría destilada (marco; detalle en el resumen)

Los cuatro artículos aportaron el marco, no las decisiones. Lo esencial que sobrevive:

- Embedding = texto → vector; el significado está en direcciones/distancias, no en dimensiones.
  La geometría emerge del entrenamiento contrastivo; los positivos determinan qué es "similar".
- **La métrica no es decisión de arquitectura** — la fija la model card (coseno = dot en vectores
  normalizados; euclídea ordena igual).
- Los embeddings son **ciegos a identificadores y números**, débiles en palabra rara exacta
  (→ híbrida, S10), y sufren la maldición de dimensionalidad (→ calibrar umbrales sobre datos
  propios).
- **El chunking domina la calidad** tanto o más que el modelo (Vectara/NAACL 2025). El mejor
  chunking depende del tipo de contenido.
- **Estructural + metadata enriquecido = mayor ROI** (Azure +15-25). Contextual Retrieval
  (enriquecer antes de embeber, no lo devuelto). Parent-child = arquitectura de retrieval.
- Selección de modelo: cinco ejes; benchmark sobre datos propios, no MTEB genérico. Matryoshka
  como palanca futura.

---

## 2. Decisiones fundamentadas en S07 (nodos de decisión)

Lo que la sesión decidió y dejó firme. Candidatas a Guide como **nodos del árbol**.

**Embedding y espacio.**
- Métrica no se delibera (model card). **Modelo fijado: `text-embedding-3-small`, 1536 dim**
  (vigencia verificada a sept 2026). No se montó torneo de modelos — sobreingeniería a esta fase;
  el modelo vive detrás del seam + fingerprint, reversible. Lo que **sí** se hizo es el sanity
  check de dominio (no elige modelo, verifica que no es ciego a la distinción crítica).
- **Idioma canónico = inglés hacia el modelo**, sea cual sea el idioma de la query. Fijarlo saca
  el requisito multilingüe del embedder. Mecanismo de normalización de query: abierto.
- **Seam, no wrapper completo.** La abstracción multi-proveedor es Phase 4; en S07 basta la función
  frontera que aísla `3-small` (patrón Phase 1). El seam es la frontera de **una petición**; el
  batching es política del llamante, no del seam. Cliente inyectable (contrato testeable sin red).
- **Determinismo de embedding ≠ de chunking.** El chunk se testea por igualdad (determinista); la
  API de embeddings puede variar numéricamente entre llamadas, así que la garantía es
  **identidad-de-input** (fingerprint), no igualdad-de-output. No se escribe test de igualdad de
  vectores. Dos stages del mismo pipeline con garantías distintas, dicho en voz alta.
- **Artefacto de vectores estampa tres versiones:** `corpus_version` (conjunto), `chunks_fingerprint`
  (qué chunks lo produjeron), y `stage_fingerprint` = digest sobre (`embed_text`s + **modelo +
  dimensiones**). Modelo y dims entran en el digest porque el mismo texto bajo otro modelo es otro
  vector. `.npy` float32 + sidecar; el orden de `chunk_ids` ES el mapeo fila→chunk (S08 carga sin
  re-derivar).

**Retrieval (nodo de primer orden, medido en S07 aunque se implemente en S10).**
- **Búsqueda híbrida (densa + léxica) = requisito, no opción, para este corpus.** El par trampa lo
  midió: "criticidad B" vs "C" difieren en 2 tokens de 42 y salen a 0.98 en denso — por encima de
  los pares similares. No es fallo del modelo (`3-large` daría igual): el denso comprime significado
  y descarta la identidad léxica exacta, y **la distinción crítica de ECSS vive sistemáticamente en
  identificadores de coincidencia exacta** (toda la tabla D-1: A/B/C/D es la misma frase con el
  numeral cambiado). El mecanismo que separa esos tokens es la búsqueda léxica (BM25), fuerte justo
  donde el denso es ciego. Condición de aplicabilidad del nodo: *corpus cuya distinción crítica vive
  en identificadores/jerga de coincidencia exacta*. Señal disparadora: *el par trampa denso no queda
  por debajo de los pares similares*. Límite honesto: el probe midió la **representación** (colapsa
  B/C); que el **retrieval** falle es altísimamente probable pero se confirma con embedding de query
  en S09.

**Chunking.**
- **Despacho por ancla de contenido, no por tipo de documento.** El ancla es mixta dentro de un
  documento (un standard normativo tiene requisitos *y* secciones con tablas aplanadas; 6 docs no
  tienen requisitos). Contenido portador-de-requisito → unidad requisito; contenido anclado-a-
  sección → unidad sección. Ambos chunkers corren sobre cada documento.
- **embed ≠ cite** como campos separados: `embed_text` (header + fuente) ≠ `cite_ref` (verbatim
  limpio + coordenada + `corpus_version`). La extracción de cita nunca agarra el header.
- **Header contextual estático y determinista** (sin LLM): gana en coste *y* reproducibilidad
  (mismo `corpus_version` → mismo embedding). Materia = `Subtitle` del documento; identificadores
  fuera del vector, en metadata.
- **Overlap 0 para requisito** (átomo acotado; solapar contamina la cita); **overlap intra-sección
  12%** solo entre sub-chunks de la misma sección (prosa continua).
- **Cita de un sub-chunk de sección** = coordenada de la sección + texto verbatim del sub-chunk
  (los sub-cortes no tienen clause-ID propio; fabricárselo sería inventar identidad).
- **Ancla `table` separada de `section`** por procedencia (linealización de rejilla ≠ prosa
  escrita; el retrieval debe distinguirlas).

**Autoridad.**
- **Ningún chunk mezcla autoridades.** La autoridad viaja con el contenido, **no se deduce del
  tipo**: un requisito en anexo informativo no es normativo por ser requisito. NOTE standalone en
  sección normativa → chunk informativo propio; en sección informativa → se queda. La NOTE de
  glosario es excepción (procedencia de la definición, viaja en la unidad).
- **Cluster chunking excluido** (rompe trazabilidad). **Propositional excluido para normativo**
  (fabricación), permisible para informativo.

**Referencias e identidad.**
- **Chunks ref-ready, agnósticos al consumo:** slot de refs salientes + clause-ID prefix-
  addressable. La **población** del slot está gateada por el re-run + validación del extractor —
  no se puebla desde artefactos stale/no-validados (riesgo de cita fantasma).
- **Direccionamiento multi-nivel por prefijo de clause-ID** (comparación por **componentes**, no
  prefijo de string: `5.20` no cuelga de `5.2`). Da parent-child barato sin índice de padres.
- **Grafo-objeto (`nx.DiGraph`) descartado** a favor de índice de adyacencia desde
  `references.json`. **Grafo-vs-metadata** (cómo se *recupera* lo referenciado: adyacencia vs
  metadata vs baking-en-embedding) **sigue abierto**.
- **EXPECTED OUTPUT y AIM atados a su requisito** (contenido citable ligado; asociación
  estructural en el parser, viajan en el chunk de requisito).

**Versionado.**
- **Invalidación por fingerprint de contenido real** (decisión B): `corpus_version` = etiqueta
  legible del conjunto (cita, manifiesto); `input_fingerprint` = digest sobre lo que el stage de
  verdad consume. Cada stage fingerprintea su input; un cambio aguas arriba se propaga como input
  cambiado aguas abajo, sin bump manual. `is_stale()` tri-estado: `None` cuando no se puede saber.

**Parseo.**
- **Extraer-todo-con-etiqueta; exclusiones declaradas y contadas, nunca silenciosas.** Un estilo
  desconocido se reporta como no-clasificado, no se descarta.

---

## 3. Aprendizajes de implementación (principios de método)

El grueso del valor de S07, y lo que las instrucciones predicen: nacido de la fricción, no de la
teoría. Candidatos a Guide como **principios de método**. Organizados bajo la tesis.

### 3.1 Verificación: contra el render, no contra el texto

- **[agnóstico] Fidelidad de contenido ≠ fidelidad de significado.** Dos defectos sobrevivieron a
  "200/200 párrafos literales" porque el contenido estaba y era literal; lo que faltaba era la
  *relación entre las partes*. Aplanar una lista jerárquica **cambia lo que afirma** (seis estados
  hermanos vs dos con anidamiento — a "¿qué estados soporta un OBCP?" el texto plano fundamenta
  una respuesta incorrecta; 21% de los requisitos). Los cuadros de texto eran texto *apareciendo*
  donde no estaba. La estructura visual es parte del significado.
- **[agnóstico] Un parser escrito para un consumidor no es el input de otro.** El hueco del 60% de
  prosa fue invisible mientras el único consumidor era quien definió el parser (requisitos);
  apareció cuando el chunking pidió todo el cuerpo. **La ausencia no es observable desde una suite
  en verde** — los tests solo aseveran lo especificado. Verificar el shape del input antes de
  construir encima es la contramedida (y cazó el 60% en la Tarea 0 del primer brief).
- **[agnóstico] "Sin clasificar" es un detector, no burocracia.** Hacer visible lo desconocido por
  construcción cazó, en su primera corrida, estilos estructurales que llevaban en el corpus desde
  S06. Pero **el detector mismo se depura**: su primer veredicto marcó `DRD1` en falso (imprime
  `<5.5.1>`, marcador de ítem que S06 eligió no numerar) y halló un defecto real (`Normal`
  numerado) que correctamente rehusó auto-enrutar.
- **[agnóstico] El muestreo aleatorio no encuentra defectos raros y localizados.** 14 casos sobre
  15.000 párrafos → 200 muestras tenían ~17% de cazar uno. Los encontró la **inspección dirigida
  por dimensión** (mirar un campo documento a documento). La UAT dirigida > la aleatoria para
  identidad.
- **[agnóstico] Cuando dos mecanismos de una spec se contradicen, manda la verificación.** El
  Brief 4 afirmaba (falso) que las tablas viven en `Section.body`; el mismo brief exigía un chequeo
  de fuga. Seguir la aserción borraba 1,46M chars en silencio; el chequeo lo impidió. La
  verificación tiene autoridad sobre la aserción de conveniencia — incluida la del autor del brief.
- **[agnóstico] Resolver una discrepancia ≠ pasar la prueba.** Explicar por qué algo *parece* mal
  no confirma que esté bien; una verificación que se detiene cuando la anomalía queda "explicada"
  cierra un control sin ejecutarlo.
- **[agnóstico] Algunas verificaciones son sensibles al orden.** `diff_reparse` compara disco vs
  parse fresco: solo sirve *antes* de escribir. Re-parsear primero la anula en silencio (pasó dos
  veces en S07). El error de método se declaró, no se tapó.
- **[agnóstico] "5 de 1455, despreciable" es la decisión correcta por el motivo equivocado.** Los
  5 candidatos a refs de sub-ítem eran falsos positivos; el conteo real es 0. *Raro* e
  *inexistente* llevan a futuros distintos — un conteo pequeño aún exige inspección.

### 3.2 Identidad: sin fabricar, sin mezclar semánticas

- **[agnóstico] Una clave derivada del contenido inventa correspondencias en corpus repetitivo, y
  en la dirección que más asusta.** Emparejar requisitos por `(uid, texto)` reportó 901 re-anclajes
  falsos; por título, 24/26 docs "tocando identidad". Real: 0. Donde hay orden estructural fiable,
  **la posición es mejor clave que el contenido** (misma raíz que el 64% de `chunk_id` duplicados).
- **[agnóstico] La canonicalización topológica no se propaga al contrato de cita.** `cite_ref`
  conserva el doc_id con revisión; `chunk_id`/metadata usan el canónico. Dos identidades, dos
  propósitos — misma familia que embed≠cite y recuperar≠citar.
- **[dominio→agnóstico] Sin identidad fabricada.** "None." ×2 no se vuelve requisito citable;
  números de cláusula repetidos no se desambiguan colgando el título (la ambigüedad es del
  documento, se declara); `Normal` numerado no se enruta por estilo (convertiría toda prosa
  numerada en cláusula).
- **[agnóstico] Precisa-e-incorrecta es peor que gruesa-pero-correcta.** El principio que atravesó
  cinco briefs y forzó el acople contador↔enrutado: enrutar `Definition1` sin descontar los
  borrados habría convertido 482 citas gruesas-pero-correctas en 482 precisas-e-incorrectas.
- **[agnóstico] Un prefijo puramente numérico es la parte alta del número, no presentación.** El
  XML lo codifica (`lvlText='3.2.%1'`); `Annex`/`NOTE`/`(` sí son presentación. Distinguir
  identidad de presentación por lo que el formato codifica, no por lo que parece.
- **[agnóstico] Dos semánticas dentro del mismo parser producen output plausible-y-falso.** El
  contador de lista seguía el documento *con marcas*; el texto, el *aceptado*. Alinear el contador
  a la semántica del texto (borrado no consume número) lo arregla. Y la **condición de borde no se
  confunde**: vacío ≠ borrado (un párrafo vacío sí consume número; uno borrado no) — un test por
  cada caso.

### 3.3 Versionado y proceso

- **[agnóstico] El fingerprint se ancla a inputs reales, no a un proxy ni a una proyección.**
  `corpus_version` (proxy del conjunto) falló porque no captura contenido ni lógica — los Briefs 2
  y 3 cambiaron el output del parser tres veces bajo la misma etiqueta. Se hashean los ficheros
  almacenados, no una proyección re-derivada (que habría que sincronizar). Conservador: avisa de
  más, nunca de menos. Un bump manual es "true by discipline"; el fingerprint encadenado es "true
  by construction".
- **[agnóstico] Un "no lo sé" no se disfraza de "está bien".** `is_stale()` devuelve `None` cuando
  no hay artefacto o no lleva fingerprint.
- **[agnóstico] Arregla la mitad barata antes de que el derivado caro la congele.** La sangría
  (nesting) cuesta 5 líneas y se hizo; los marcadores literales tocan `numbering.py` y se
  difirieron. El texto del requisito es lo que se embebe: cambiarlo tras generar embeddings obliga
  a re-embeber. El coste de regeneración aguas abajo dicta qué se arregla antes.
- **[agnóstico] Nombrar por el ancla, no por el accidente temporal.** `SectionChunker`, no
  `TableChunker`, aunque hoy solo coma tablas.
- **[agnóstico] Un derivado invalidado se borra en voz alta.** `parse_all` nombra el borrado de
  `chunks.json` en el log — no es que no se pueda borrar, es que no se puede borrar callando.
- **[agnóstico] Clasificar por familia normalizada, preservar el original.** Existen `EXPECTED
  OUTPUT` y `Expected Output`; un match exacto se come el gemelo en silencio — el fallo que se
  corrige, reintroducido en la corrección. Normalizar para clasificar; el estilo original es sobre
  el que el consumidor decide.

### 3.4 Embedding: medir la representación antes del retrieval

- **[agnóstico] El valor del probe no fue el número, fue mirarlo antes de indexar.** El 0.98 del par
  trampa, visto en S11 durante una evaluación de retrieval, se lee como "el retrieval falla" y se
  gastan días afinando k, reranking y prompts — persiguiendo un síntoma. Visto antes de indexar, se
  lee como lo que es: una propiedad del *texto fuente* que la representación densa no puede capturar
  y que ningún tuning de retrieval arregla. **Medir la propiedad de la representación antes de
  construir retrieval separa "el modelo no puede" de "mi retrieval está mal configurado"** — dos
  diagnósticos que se confunden y llevan a semanas distintas.
- **[agnóstico] Un probe se construye del corpus, no se escribe a mano.** El par trampa se extrajo
  de la tabla D-1 real; un probe escrito a mano mediría el modelo contra la forma de escribir del
  autor, no contra los documentos de los que el sistema responde. Y su límite se declara: mide la
  representación, no el retrieval (eso es query-side, S09).
- **[agnóstico] Un test que mockea la frontera no prueba el cableado hasta la frontera.** El seam
  leía `os.environ`, pero el proyecto lee `.env` vía pydantic-settings y nunca exporta; el modelo se
  ignoraba y devolvía el default — y como *coincidía*, no se notaba (habría fallado en silencio al
  cambiar de modelo, con el fingerprint registrando uno y el build usando otro). Los tests no lo
  cazaron porque inyectan el cliente y nunca tocan la config real. El mock de la frontera deja sin
  probar el tramo *hasta* la frontera; ese tramo necesita su propio test.
- **[agnóstico] Ordenar la respuesta por `index`, no fiarse del orden de llegada.** Un lote
  desordenado daría a cada chunk el vector de otro: artefacto consistente en forma (mapeo total,
  normas ≈ 1) y **falso en contenido**, indetectable aguas abajo (citas bien formadas, cláusulas
  equivocadas). Misma familia que el `startswith` del Brief 1 y el emparejamiento por contenido del
  Brief 3: plausible, silencioso, solo visible si lo buscas antes. Test con cliente que invierte el
  orden a propósito.
- **[agnóstico] El seam es la frontera de una petición, no del batching.** Cuántos textos por
  request es un compromiso con rate limits del llamante, no una propiedad del seam. Separarlos
  mantiene el seam limpio para reusarlo en el embedding de query (S09).

---

## 4. Estado real al cerrar S07

**Objetivo de temario cumplido — code-complete y validado:**
- **Embeddings generados:** 13.876 vectores · 1536 dim · `text-embedding-3-small` · normas ≈ 1 ·
  mapeo total. Artefacto `.npy` + sidecar estampado con las tres versiones, listo para que S08 lo
  cargue en pgvector. Sanity check corrido; par trampa medido (0.98).
- **Chunking + identidad:** 13.876 chunks · 0 colisiones · 0 fugas · citación-correcto validado
  contra el render (PDF como oráculo). Prosa (2,26M chars) indexada. Glosario anclado a su cláusula
  (480 términos, validados). AIM y EXPECTED OUTPUT en su requisito.
- **Parser:** prosa de cuerpo tipada y ordenada; `Annex4/5`, `Definition1`, `require:level4`
  enrutados; contador de lista con semántica de documento aceptado; `document_title`/`document_subject`.

**Abierto (movimientos siguientes, no de S07):**
- **Vector store (pgvector):** S08 — carga el artefacto de vectores que S07 dejó listo.
- **Búsqueda híbrida:** S10 — **requisito medido** por el par trampa, no opción (ver §2).
- **Embedding de query + normalización a inglés + confirmar el fallo de retrieval B/C:** S09.
- **Población del slot de refs:** gateada por validación del extractor contra ground truth.
- **Grafo-vs-metadata:** abierto (el mapa creció con baking-en-embedding).

**Deuda declarada (visible, no silenciosa):**
- Marcadores literales de lista — **diferible confirmado por medición** (0 refs a sub-ítem en el
  corpus); la sangría cubre el anidamiento.
- Contenido de tabla como destino recuperable/citable; contenido de cuadros de texto — caminos
  deliberados diferidos, no scavenging.
- `Normal` numerado (1 caso, "Nomenclature" 3.4) y números de cláusula repetidos (4 casos) —
  declarados en el reporte, gruesos-pero-correctos.
- Contextual Retrieval con LLM; propositional para informativo; late chunking (acoplado a modelo).

**Diferido con condición de salida cumplida o pendiente:** ver §2 (ref-ready, grafo-vs-metadata).

---

## 5. Candidatos a Guide (para el paso 6, etiquetados)

**Nodo de decisión de primer orden (agnóstico):** **búsqueda híbrida obligatoria cuando la
distinción crítica del corpus vive en identificadores/jerga de coincidencia exacta.** Condición
(estática): el discriminante del dominio es léxico exacto, no semántico. Señal (dinámica): el par
trampa denso no queda por debajo de los pares similares. Es el hallazgo más consecuente de S07 para
la arquitectura, y llega con prueba, no con supuesto.

**Nodos de decisión (agnósticos):** métrica la fija la model card; **modelo estándar sin torneo a
menos que la medición lo pida** (sanity de dominio, no benchmark competitivo); seam de proveedor
(una petición) vs abstracción completa (fase posterior); determinismo de embedding ≠ de chunking
(garantía input-identity, no output-equality); artefacto derivado estampa conjunto + input +
(modelo+params); embed≠cite como campos separados; despacho por ancla de contenido; header estático
determinista vs Contextual Retrieval con LLM; overlap por tipo de unidad; autoridad por chunk sin
mezclar, ligada al contenido; chunks ref-ready agnósticos al consumo; direccionamiento por prefijo
por componentes; invalidación por fingerprint de inputs reales; grafo-objeto vs índice de
adyacencia; cluster/propositional excluidos por trazabilidad/autoridad.

**Principios de método (agnósticos):** **medir la propiedad real de cada capa antes de construir la
siguiente** (separa "el modelo/parser no puede" de "mi capa superior está mal configurada"); un
probe se construye del corpus, no se escribe a mano; un test que mockea la frontera no prueba el
cableado *hasta* la frontera; ordenar por índice, no por orden de llegada (corrupción de contenido
plausible y silenciosa); verificar contra el render ≠ contra el texto; la ausencia no es observable
desde una suite verde; hacer visible lo desconocido como detector; muestreo aleatorio no caza
defectos raros → inspección dirigida; la verificación manda sobre la aserción de la spec; resolver ≠
verificar; verificaciones sensibles al orden; raro ≠ inexistente; clave por contenido inventa
correspondencias → posición donde hay orden; sin identidad fabricada; precisa-e-incorrecta peor que
gruesa-pero-correcta; dos semánticas en un parser dan output plausible-y-falso; fingerprint sobre
inputs reales, tri-estado honesto; arregla lo barato antes de que el caro lo congele; nombrar por el
ancla; borrar en voz alta; normalizar para clasificar preservando el original.

**Meta-principio (agnóstico), el de mayor alcance:** cada capa del pipeline (parseo → chunking →
embedding → retrieval) descansa sobre una propiedad de la capa inferior que la teoría da por
supuesta y el corpus real puede no cumplir — y esa propiedad hay que **medirla, no asumirla**, antes
de construir encima. En el parseo fue la fidelidad estructural (agujero del 60%, listas aplanadas);
en el embedding fue la discriminación de la representación (par trampa). El chunking "domina la
calidad" (teoría) solo si el parseo bajo él es fiel; el retrieval denso "encuentra por significado"
solo si la distinción vive en el significado. La fidelidad no es que el dato esté — es que la
propiedad que lo hace útil para la capa siguiente esté, y se demuestre.

**Dominio (ECSS, no Guide agnóstica):** la mecánica concreta de estilos (`Definition1`, `Annex4/5`,
`require:level*`, `NOTE*`, `aim`, `EXPECTED OUTPUT`), `numbering.xml`, el DOCX como fuente y el PDF
como oráculo de validación, los números de cláusula repetidos, el ancla `table`.

---

## 6. Nota de proceso

Seis briefs de implementación en una sesión (lo normal es uno): cinco de fidelidad e identidad
(2-5) más el objetivo (6, embeddings). No fue dispersión: fue un único defecto de fidelidad (el
hueco del 60%) que, al destaparse, arrastró la cadena de identidad de cita que había debajo. Cada
brief cerró una capa y su UAT destapó la siguiente, hasta agotar la familia con la auditoría de
estilos.

**Corrección de proceso (compromiso vivo, a incorporar a las instrucciones).** A mitad de sesión
propuse un corte equivocado: consolidar y graduar la Guide tras cerrar el arco chunking+identidad,
*sin* generar embeddings — el objetivo de temario. El usuario lo corrigió: **una sesión no se
consolida ni gradúa hasta que su objetivo de temario está cumplido; los briefs intermedios son
medios, no sustitutos.** La razón es de sistema: el temario es una máquina de estados con
precondiciones entre sesiones — S08 asume que existen vectores que cargar. Consolidar sin el
objetivo habría propagado un desfase a S08 y graduado conocimiento incompleto. El orden correcto no
es solo consolidado→Guide (5→6); es **objetivo-de-sesión-cumplido → 5 → 6**. Este consolidado se
escribió como borrador de trabajo tras el arco de identidad y se **congeló solo aquí**, con el
objetivo cumplido y el mineral de embeddings dentro.

El arco tiene cierre: de los estilos estructurales que el detector destapó, quedan 2 párrafos
"None." (prosa, deliberado) y 1 `Normal` numerado (declarado). S07 cumplió su objetivo —vectores
generados— sobre un corpus que por fin es fiel *y* citación-correcto, la precondición que la teoría
daba por supuesta y el corpus real no cumplía. S08 arranca cargando el artefacto de vectores;
S10 arranca con la híbrida ya justificada por medición.
