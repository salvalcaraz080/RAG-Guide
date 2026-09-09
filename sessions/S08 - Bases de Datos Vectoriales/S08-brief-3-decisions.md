# S08 · Decisiones — Brief 3 (ingesta del artefacto en pgvector)

> Decisiones crudas tomadas al implementar, etiquetadas **[AGNÓSTICO]** (transferible a
> cualquier sistema RAG) o **[DOMINIO]** (específico de ECSS / de este proyecto).
> **Verificación:** `uv run pytest` (784 tests) **y la carga real ejecutada contra Postgres** —
> 26 documentos, 13.876 chunks, conteos leídos de la base, idempotencia y guardas comprobadas
> sobre los artefactos reales. No queda UAT pendiente.

---

## 1. Qué marca realmente `oversized` — la pregunta del brief, respondida

**El brief suponía «el vector puede no representar todo el `cite_text`». Es falso**, y la
diferencia importa porque cambia para qué sirve la columna.

Lo que hace el chunker: `oversized = counter.count(text) > max_tokens`, con `max_tokens = 512`.
Marca que el chunk **excede la ventana de chunking y no se partió** — y no se parte a propósito,
porque partir un requisito rompe la identidad que lo hace citable.

Medido sobre los dos casos del corpus:

| chunk | tokens `cite_text` | tokens `embed_text` |
|---|---|---|
| `ECSS-E-ST-10-03C#5.5.1.2::…_0750117` | 555 | 604 |
| `ECSS-E-ST-40-07C#5.3.7::…_1630261` | 646 | 688 |

El tope del embedder es **8.191 tokens**. Los dos están un orden de magnitud por debajo, así que
**sus vectores cubren el texto entero**. Y no es casualidad de estos dos: el `embed_text` más
largo de todo el corpus mide 2.206 tokens, y `corpus/embed_all.py` reporta —nunca trunca— un input
que exceda el tope, con 0 casos.

**Lo que la bandera sí señala es dilución, no truncamiento.** Más contenido promediado en un vector
de tamaño fijo es una representación menos afilada. Eso la convierte en una señal de **calidad de
recuperación**, no en un defecto de fidelidad — y sigue justificando la columna tipada por el
motivo que daba el brief (poder preguntar «¿este miss de recall cae en un chunk diluido?» sin
gimnasia JSONB), solo que con la semántica correcta escrita en el modelo, la migración y
`ARCHITECTURE.md`.

Lección transferible: *una bandera heredada de otra capa hay que leerla en su origen antes de darle
significado. «Excede la ventana» y «el vector no cubre el texto» son afirmaciones distintas, y solo
una de las dos es cierta aquí.*

---

## 2. Qué entró al `metadata` JSONB: **nada**, en las dos tablas — **[AGNÓSTICO]**

Repasado campo a campo lo que sobra del artefacto:

| Campo | Por qué no entra |
|---|---|
| `token_count` | re-derivable del texto; el brief 1 §1.2 ya lo excluyó |
| `anchor` | ya es la columna `chunk_type` |
| `metadata.revision` | vive en `documents.doc_id_cite` |
| `metadata.corpus_version` | vive en `documents.corpus_version` |
| `outgoing_refs` | gateado hasta validar el extractor |
| `metadata.standard` / `clause` | son `documents.canonical_doc_id` y `section_number` |

No queda nada que califique, así que el blob va `{}` y está afirmado por un test. **Un JSONB con
contenido inventado es un contrato que nadie acordó**, y el coste de inventarlo no se paga hoy sino
cuando alguien lo consulte creyendo que significa algo.

---

## 3. Una quinta guarda que el brief no pedía — **[AGNÓSTICO]**

El brief listaba tres (fingerprint, conteo, dimensión/modelo). Añadí una cuarta comprobación
—que el sidecar declare las mismas dimensiones que la matriz— y sobre todo **una quinta: que el
orden de `sidecar["chunk_ids"]` sea el de `chunks.json`**.

El motivo es que esa es **la única de todas cuyo fallo es indetectable aguas abajo**. El sidecar
de S07 declara en su propio docstring que *el orden de esa lista ES el mapeo* fila→chunk. Si ese
orden se rompiera, cada chunk recibiría el vector de otro y el resultado sería una base **bien
formada y falsa**: conteos correctos, dimensiones correctas, fingerprints correctos, y retrieval
devolviendo el requisito equivocado con una cita impecable. Las otras cuatro guardas protegen de
fallos ruidosos; esta protege del silencioso.

Cuesta una comparación de listas y corre una vez por carga. Verificada contra los artefactos reales
invirtiendo la lista en memoria.

---

## 4. `documents` desde `chunks.json`, y el join con el manifiesto por identidad de CITA

Ejecutado como pedía el brief, con un matiz de implementación: el join con el manifiesto va por la
**identidad de cita** (`ECSS-E-ST-40C Rev.1`), no por la canónica. No es una desviación sino la
forma exacta de hacer el mismo join: `ManifestEntry.doc_id` **es** la identidad con revisión, así
que emparejar por ahí es una comparación directa, mientras que hacerlo por la canónica exigiría
canonicalizar los 26 doc_id del manifiesto en cada corrida — derivar para comparar, cuando la clave
ya está delante.

De ahí sale `source_path`, como `data/processed/{entry.key}.json`: `key` es la forma reversible con
la que `parse_all` nombró el fichero, así que no se re-deriva un nombre que otro stage ya fijó.

Las dos abortadas se re-comprueban en cada corrida, no se asumen del brief 1: **canónica→cita 1:1**
(sigue siéndolo en los 26) y **todo documento con chunks está declarado en el manifiesto**.

---

## 5. Dónde vive el módulo, y la tensión que arrastra — **[AGNÓSTICO]**

`app/services/ingestion.py`, que es donde `CLAUDE.md` ya lo tenía planificado. Es un batch job
standalone: nada del camino de petición lo importa, y él no importa nada de ese camino.

**La tensión, anotada y no resuelta:** importa `app.db`, que lee `app.config.Settings`, que exige
`ANTHROPIC_API_KEY`. Un job offline que solo necesita `DATABASE_URL` acaba pidiendo una credencial
de LLM — exactamente el acoplamiento que `app/services/embedding.py` evitó declarando su propio
`Settings`, y el mismo que `corpus/log_config.py` evitó para el logging. Hoy no molesta (Salva
tiene la clave en `.env`) y arreglarlo tocaría el `Settings` que el brief 1 fijó por instrucción
explícita, así que se deja nombrado. El módulo sí usa `corpus.log_config` en vez de
`app.observability`, que es la mitad del problema ya evitada.

---

## 6. Detalles de implementación que resultaron ser decisiones

- **`mmap_mode="r"` al cargar `embeddings.npy`** — **[AGNÓSTICO]**. Son 85 MB que se leen fila a
  fila; cargarlos enteros en RAM para eso no aporta nada.
- **Lotes de 500 filas** — **[AGNÓSTICO]**. No es una constante de negocio sino un tope de memoria
  del servidor: 13.876 vectores en una sentencia obligarían a materializar los 85 MB de golpe. Por
  eso es un flag, no una constante. La carga completa tardó **57 s**.
- **`build_rows` no asume 1:1** aunque hoy lo sea: un chunk sin vector entra con `NULL`, no
  revienta. La guarda de conteo es la que decide si esa situación llega. La fila es la unidad
  citable y existe sin su vector — es la razón por la que la columna es nullable.
- **Todo en una transacción** (`session.begin()`): documentos y chunks entran juntos o no entra
  nada. Con la FK de por medio, una carga parcial dejaría chunks apuntando a documentos que otra
  corrida podría cambiar.
- **La ingesta compara lo almacenado con lo ingerido** y avisa si sobran filas. No las borra —eso
  es el límite conocido del upsert— pero lo dice, en vez de que un conteo raro aparezca meses
  después sin explicación.

---

## 7. Sorpresas

### 7.1 El test de deriva modelo↔migración saltó en rojo, y estuvo bien que saltara

El test que escribí en el brief 1 exige que toda columna del modelo aparezca en la migración —
pero leía **solo** `0001`. Al llegar `oversized` en la `0002`, rojo. Es un falso positivo, pero
uno que solo podía aparecer la primera vez que hubiera una segunda revisión: el test estaba atado
a un fichero cuando su premisa era «alguna revisión la migra». Corregido a recorrer todas.

Lección transferible: *un test que se ata a un artefacto concreto en vez de a la propiedad caduca
en cuanto el artefacto se multiplica. Que caduque en rojo, y no en silencio, es lo único que lo
salva.*

### 7.2 Una guarda que reventaba justo al explicarse — **[AGNÓSTICO]**

El mensaje de la guarda de orden llevaba una flecha Unicode (`→`). La consola de Windows es cp1252
y **no puede codificar ese carácter**: la guarda habría muerto con `UnicodeEncodeError` en el
momento exacto de decir por qué se negaba, convirtiendo una negativa limpia en el stack trace que
el brief pedía evitar. Apareció al ejecutar las guardas contra los artefactos reales — un test
unitario no lo habría visto nunca, porque pytest captura la salida y no la codifica al terminal.

Arreglado dejando en ASCII las cadenas imprimibles, y comprobado con un recorrido del AST sobre
todos los argumentos de `IngestionError` y de `log.*`.

Lección transferible: *el camino de error también se ejecuta. Un mensaje de fallo que no se ha
impreso nunca en el terminal real es código sin probar, y falla en el peor momento posible.*

---

## 8. Lo que se dejó sin hacer, según el brief

- **No se borran huérfanos** en re-ingesta: el upsert da de alta y actualiza. Hoy el corpus es
  estático dentro de una `corpus_version`; cuando mute habrá que reconciliar por presencia.
  Anotado en `CLAUDE.md` y avisado en el log, no construido.
- **Sin endpoint, sin tocar el camino de petición**, sin índice vectorial, sin `halfvec`, sin refs
  pobladas, sin re-embeber (`embeddings.npy` y sidecar sin abrir para escritura).
- **No se abrió el refactor del manifiesto** de los seis módulos (brief 2 §5.1), ni el `[0.26.0]`
  duplicado, ni el huérfano `rag-ecss-redis`, ni `EMBEDDING_PROVIDER`.

---

## 9. Nota de operación

La base la habías parado (`docker compose stop db`); la levanté para poder ejecutar la carga y
**la dejo corriendo, con los datos dentro**, que es lo que el brief 4 necesita encontrarse.
`docker compose stop db` la para conservando el volumen.
