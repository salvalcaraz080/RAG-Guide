# S07-decisions-6-embeddings.md — Implementación (Claude Code)

Decisiones y fricciones al implementar `S07-brief-06-embeddings.md`.
Etiquetadas **[agnóstico]** (candidata a Guide) vs **[dominio]** (solo ECSS).

---

## UAT — generación correcta, y el par trampa COLAPSA

### Generación: verde

```
chunks 13.876 · oversized 0 · dimensions 1536 · model text-embedding-3-small
norm_min 0.99935 · norm_max 1.00062          (normalizados, sin truncado Matryoshka)
corpus_version 2026-08-04.91bc8116
chunks_fingerprint ab3fce8b5cb1a0d4  ·  stage_fingerprint c54e2f865e3fb1a6
```

Mapeo total (13.876 vectores para 13.876 chunks), ningún input sobre el tope, normas ≈ 1 como
el brief anticipaba. El artefacto queda listo para que S08 lo cargue.

### Sanity check: el par trampa queda POR ENCIMA de los pares similares

```
0.9818   trap      criticidad B vs C
0.7044   similar   validación de software, dos estándares
0.4900   similar   dos requisitos de la misma cláusula
0.2977   distant   criticidad vs ensayo de humedad
0.2924   distant   software vs ensayo de vibración

trap_below_similar = False
```

La estructura que el brief quería ver —similares > trampa > lejanos— **no se sostiene**. El par
trampa no es que colapse *con* los similares: los supera con holgura.

### Por qué, y por qué no es un fallo del modelo

Los dos textos, verbatim del corpus:

```
B: Software involved in category  I functions AND: at least one of the following compensating
   provisions is available, meeting the requirements defined in ECSS-Q-ST-30 clause 5.4 and
   ECSS-Q-ST-40 clause 6.5.6.3: - A hardware implementation - A software implementation; this
   software implementation shall be classified as criticality A - An operational procedure

C: Software involved in category II functions AND: … shall be classified as criticality B …
```

**Difieren en dos tokens de cuarenta y dos.** `I` → `II` y `A` → `B`. Solapamiento léxico del
**93%** (39 de 42 tokens compartidos).

- **[agnóstico] El 0,98 es la respuesta correcta a la pregunta que se le hizo.** Un embedding
  denso comprime *significado*, y el significado compartido por esas dos frases es casi todo lo
  que dicen; lo que las distingue son dos **identificadores**. Pedirle a un vector de 1536
  dimensiones que separe dos textos idénticos al 93% por un numeral romano es pedirle algo que
  no hace ningún modelo denso — cambiar a `3-large` daría ~0,98 también.
- **[dominio] La conclusión operativa no es «otro modelo», es «otro mecanismo».** Los tokens
  discriminantes (`II`, `B`) son de coincidencia exacta, que es exactamente donde la búsqueda
  léxica es fuerte y la densa débil. Esto convierte la búsqueda híbrida de más adelante de
  «buena práctica que se asume» en **requisito medido**, con el caso concreto que la justifica.
- **[agnóstico] El valor del probe no fue el número, fue mirarlo antes de indexar.** Si esto
  aparece en S11 durante una evaluación de retrieval, se lee como «el retrieval falla» y se
  gastan días afinando k, reranking y prompts. Visto aquí, se lee como lo que es: una propiedad
  del texto fuente que la representación densa no puede capturar.

### Límite honesto de mi propio probe

El par que construí son **dos filas de una tabla**, que es el caso más extremo posible: puro
boilerplate compartido. Una consulta real no es «fila B contra fila C», sino «pregunta contra
filas». La pregunta que de verdad decide es **si una consulta por la categoría C recupera C por
encima de B**, y eso es embedding de query — fuera del alcance de este brief (S09).

Mi probe demuestra que la **representación** no separa B de C. No demuestra todavía que el
**retrieval** vaya a equivocarse, aunque lo hace muy probable. Medirlo cuesta una llamada más y
lo dejo ofrecido, no hecho.

### El resto de la tabla D-1 dice lo mismo

`A`, `B`, `C` y `D` son la misma frase con el numeral y la letra cambiados (salvo `A`, que es
más corta porque no lleva provisiones compensatorias). No es una peculiaridad del par B/C: es
cómo está escrita la tabla entera, y por extensión buena parte de las tablas normativas de ECSS.

---

## Dónde vive el seam, y por qué no lee `Settings`

`app/services/embedding.py`, no `corpus/`. El brief dice que es el mismo seam que usará el
embedding de query (S09+), y `CLAUDE.md` ya reservaba ese fichero para M2.

- **[agnóstico] No construye `app.config.Settings`.** Ese exige `ANTHROPIC_API_KEY`, y un build
  offline de corpus no tiene por qué necesitar credencial de LLM. Es el razonamiento que en S06
  dio a `corpus/` su propio `log_config`, un piso más arriba. Declara sus dos variables
  (`OPENAI_API_KEY`, `EMBEDDING_MODEL`) en un `EmbeddingSettings` propio, sobre pydantic-settings
  como el resto del proyecto.
- **[agnóstico] Fricción real: mi primera versión leía `os.environ` y falló al ejecutarla.** Este
  proyecto guarda sus variables en `.env` y **nunca las exporta al entorno del proceso** — las lee
  pydantic-settings. El SDK de OpenAI no encontraba la clave, y peor: `active_model()` devolvía
  siempre el default ignorando `EMBEDDING_MODEL` de `.env`. Como el valor coincidía, **no se
  notaba**; habría fallado en silencio el día que alguien cambiase el modelo, con el fingerprint
  registrando uno y el build usando otro. Lo encontró la ejecución del usuario, no mis tests,
  porque los tests inyectan el cliente y nunca tocan la configuración real.
  → Lección: **un test que mockea la frontera no prueba el cableado hasta la frontera.** Hay ya
  dos tests nuevos para eso — que el modelo salga de la configuración, y que la clave ausente
  falle con nombre en vez de en las tripas del SDK.
- **[agnóstico] El seam es la frontera de UNA petición, no del batching.** `embed_texts(texts)`
  = una request. Cuántos textos por request es un compromiso con los rate limits, no una
  propiedad del seam, así que lo decide el llamante (`corpus/embed_all.py`) y es un flag.
- **[agnóstico] Cliente inyectable**, que es lo que hace testeable el contrato sin red.

---

## El detalle que habría corrompido el artefacto en silencio

`embed_texts` **ordena la respuesta por `index`** en vez de fiarse del orden de llegada.

- **[agnóstico]** Si un lote volviera desordenado, cada chunk recibiría el vector de otro. El
  artefacto seguiría siendo consistente en forma —13.876 vectores, mapeo total, normas ≈ 1— y
  falso en contenido, y **nada aguas abajo podría detectarlo**: el retrieval devolvería
  cláusulas equivocadas con citas perfectamente bien formadas. Es la misma familia de fallo que
  el prefijo `startswith` del brief 1 o el emparejamiento por contenido del brief 3: plausible,
  silencioso, y solo visible si lo buscas antes de que ocurra. Hay un test con un cliente que
  invierte el orden a propósito.

---

## Tres versiones, tres trabajos

El artefacto estampa `corpus_version` (etiqueta del conjunto declarado), `chunks_fingerprint`
(qué chunks lo produjeron) y su propio `stage_fingerprint` = digest sobre los `embed_text`
consumidos + modelo + dimensiones.

- **[agnóstico] El modelo y las dimensiones entran en el digest** porque el mismo texto bajo
  otro modelo es otro vector: sin ellos, cambiar de modelo dejaría los vectores viejos pasando
  por frescos.
- **[agnóstico] El encadenado funciona sin bump manual.** El input de este stage es el output
  del anterior, así que un cambio del chunker —o del parser aguas arriba— cambia los
  `embed_text` y con ellos este fingerprint. Los briefs 2, 3 y 5 cambiaron el output tres veces
  bajo la misma `corpus_version`; esa clase de deriva ya no puede pasar inadvertida.
- `is_stale()` tri-estado, como en el brief 4: `None` cuando no se puede saber.

---

## La honestidad sobre el determinismo

**No hay ningún test que compare vectores entre dos corridas.** La API puede devolver variación
numérica mínima entre llamadas, así que un test de igualdad fallaría en verde algún día, y un
test que falla por motivos ajenos al código es peor que no tenerlo.

- **[agnóstico] Lo que el fingerprint atestigua es la identidad del *input*, no la igualdad del
  *output*.** Es una garantía más débil y es la única que se sostiene. Lo determinista y
  testeable es el digest: mismos textos → mismo digest, y cambiar uno lo cambia.
- Contrasta con el chunking, que sí se testea por igualdad porque sí es determinista. Dos stages
  del mismo pipeline con garantías distintas, y decirlo en voz alta evita que alguien asuma la
  fuerte donde solo hay la débil.

---

## El formato del artefacto

`embeddings.npy` (matriz float32) + `embeddings.json` (sidecar).

- **[dominio] Binario para los floats.** 85 MB como JSON de texto no sería ni cargable ni
  parseable a velocidad razonable. `float32` y no `float64`: la mitad de tamaño y la precisión
  que pgvector usa igualmente.
- **El orden de `chunk_ids` en el sidecar ES el mapeo fila → chunk.** S08 carga filas a pgvector
  sin re-derivar nada.

---

## Task 5 — el par trampa se construye del corpus, no se escribe

`corpus/diagnostics/embedding_sanity.py` embebe media docena de pares **de texto real** y
reporta su coseno.

- **[dominio] El par trampa sale de la tabla D-1 de `ECSS-Q-ST-80C`**, extraído con regex del
  chunk real: las filas de categoría B y C son la misma frase con la misma lista de provisiones
  compensatorias, y solo cambian el numeral de la función y la letra de criticidad. Es
  exactamente la distinción que el sistema tiene que poder hacer.
- **[agnóstico] Todo el conjunto es texto del corpus, no prosa mía.** Un probe escrito a mano
  mediría el modelo contra mi manera de escribir, no contra los documentos de los que el
  sistema tiene que responder.
- Los otros pares: dos «deberían estar cerca» (validación de software en E-ST-40 y en Q-ST-80;
  dos requisitos de la misma cláusula) y dos «deberían estar lejos» (software frente a ensayo de
  vibración y de humedad), para tener la referencia de qué rango da el modelo en este dominio.
- **La lectura es la estructura, no el número.** En dimensión alta un 0,2–0,5 entre textos sin
  relación es normal. Lo que importa: ¿el par trampa queda por debajo de los pares-similares, o
  colapsa con ellos? El script lo reporta como `trap_below_similar`.
- **Sale con exit 0 siempre.** Informa un juicio, no dictamina: que el par trampa colapse es un
  hallazgo para Claude Chat —reconsiderar modelo, o apoyarse en la híbrida más adelante—, no un
  fallo de build.

---

## Pendiente

1. **Qué hacer con el par trampa.** El dato está: 0,9818, por encima de los pares similares.
   Mi lectura es que no pide otro modelo sino otro mecanismo —la discriminación vive en dos
   tokens de coincidencia exacta— pero la decisión es de Chat.
2. **Medir el lado query** (si C recupera C por encima de B) cerraría la pregunta de verdad.
   Es S09 y cuesta una llamada; ofrecido, no hecho.
3. Siguen abiertos del brief 5: el `Normal` numerado de `ECSS-E-ST-70-41C`, y del brief 4 el
   ancla `table` y la autoridad del chunk de requisito.
