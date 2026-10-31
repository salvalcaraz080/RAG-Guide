# S14 · Decisiones — Brief 3: UAT estructurado y credencial del retrieval

Sin llamadas al proveedor. No se toca el camino de `/query`.

## 1 · Parte A: el registro

**Ubicación:** `corpus/groundtruth/uat/records.py` y `records.jsonl`. Va junto al golden set al que
alimenta y, como él, **versionado**. `data/groundtruth/generation/` está en `.gitignore`, así que un
registro guardado ahí no sería diffeable. Las respuestas se quedan en `data/` y se referencian.

**Forma:** un `UatRecord` por caso y ejecución, con los campos de U2. Pydantic estricto: un campo
desconocido o un veredicto fuera de vocabulario rompe al escribir y al leer.

- **Referencia:** `{file, index}`, donde `index` es una ruta dentro del JSON (`first/q14/2`,
  `queries/27/runs/0`). Sirve para los ficheros de UAT y para los artefactos del arnés sin un
  formato propio de cada uno.
- **Sin identificador de petición:** el fichero más la ruta ya identifican la respuesta, y las
  guardadas no llevan `request_id`. No hizo falta tocar el contrato ni añadir uno.
- **La comprobación de referencias es explícita:** `load_records(..., responses_dir=None)` valida
  solo el esquema, porque un clone no tiene `data/`. El comando `check` y `append_records` sí la
  hacen. El test de referencias del backfill se salta sin `data/`; aquí ha corrido.

**Consulta:** `uv run python -m corpus.groundtruth.uat.records summary|candidates|check`.

**Backfill de S14:** `scripts/uat_s14_backfill.py`, 20 registros, con el juicio de `S14-uat.md` §6.

- Añadí lo que §1 del brief no nombra: q14 turno 2 (`ok`, `judge: cc`) y los 5 casos del desplegable
  del brief 2 (`ok`, `judge: cc`), que ejecuté yo con `AppTest`.
- **El render del desplegable sigue `pending` (`judge: user`)**, como un registro aparte: el brief lo
  dejaba para Salva y `AppTest` comprueba qué se pinta, no cómo se ve.
- Como pide el brief, el ↪️ «Space-to-Ground ICD» de la pasada **no** es un registro de UAT:
  queda aquí como **candidato a caso negativo del golden set** (q14 turno 1, ejecución 3:
  describe dónde documenta la misión, no una delegación).

## 2 · Coste del backfill histórico (no hecho)

28 ficheros de UAT anteriores a S14, con unos 174 apartados `##`, de los que ~103 son de S09 en
adelante o de los `pre-S11`/`pre-S12` (lo anterior es sobre todo pipeline, sin respuesta de modelo).
Es una cota: un apartado no siempre es un caso.

- **Recuperable con referencia a la respuesta: 0.** Ninguna UAT anterior guardó sus respuestas.
- **Solo el veredicto:** transcribir la prosa a mano o con CC, sin llamadas: del orden de una sesión.
  Quedarían sin `response`, que hoy es obligatorio.
- **Regenerar las respuestas:** ~100 turnos, ≈ 0,30 USD, pero **no sirve**: el modelo, el prompt y el
  contexto han cambiado (Haiku 4.5 → 5.5, `bounded`, `xref`), y el veredicto se emitió sobre otra
  respuesta.

Recomendación: no hacerlo. Lo que vale de ese histórico ya está en los diagnósticos y en el golden
set. Decide el chat.

## 3 · Parte B: credencial del retrieval (solo reporte)

**Hay un solo rol para todo, y es superusuario.** Consultado en la base viva:

- **Rol:** `rag_ecss`, creado por `POSTGRES_USER` en Compose: `rolsuper`, `rolcreaterole`,
  `rolcreatedb` y `rolbypassrls`, todos a `true`. Es el único rol de aplicación que existe.
- **Privilegios:** dueño de las 6 tablas (`alembic_version`, `chunks`, `documents`, `lexical_corpus`,
  `lexical_stats`, `table_labels`), con SELECT, INSERT, UPDATE, DELETE y TRUNCATE.
- **Sin separación:** `DATABASE_URL` es la misma para retrieval, `/search`, ingesta y migraciones. El
  contenedor `api` solo cambia el host.
- **Lectura:** el retrieval **solo lee, por convención del código, no por la credencial**. Ser
  superusuario va más allá de las tablas: da acceso a toda la instancia y, en Postgres, a capacidades
  sobre el servidor.

**Para un rol de solo lectura:** una migración con un rol `rag_ecss_reader` y `GRANT SELECT` sobre
las tablas, una segunda URL (`DATABASE_URL_READONLY`) para el engine del camino de petición, y la
ingesta y Alembic con la actual. Estimado: medio día, más decidir cómo se entrega la contraseña.
**No implementado.** El arreglo mínimo aparte es que el rol dueño no sea superusuario.

**Otras superficies de escritura desde `/query`:** ninguna en la base.

- Ningún camino de `app/routers` llega a `ingestion`, `lexical_stats` ni `table_labels`, que son
  los únicos que escriben (por comando). Comprobado leyendo imports y buscando escrituras, no
  con una prueba de ejecución.
- `retrieval.py` no monta SQL con `text()`: la query del usuario llega como parámetro.
- En memoria, las sesiones (legítimas). A disco o a terceros: logs de metadatos y las llamadas a
  proveedores.

**De paso, fuera del alcance:** `docker compose ps` muestra Postgres publicado en `0.0.0.0:5432`, con
la contraseña por defecto en `docker-compose.yml`. Para desarrollo vale; antes de M6, no.

## 4 · Tests

**1554 verde** (1537 + 17, `tests/corpus/test_uat_records.py`). Ninguna llamada a proveedor.

## 5 · Etiquetado

- **Agnóstico:** guardar el par propuesta–decisión desde el principio, porque no se recupera: de ~100
  casos históricos con modelo, 0 conservan la respuesta. El juicio se razona en prosa y el registro
  es su forma consultable. Mínimo privilegio: lo que limita es la credencial, no el código que
  «solo lee».
- **Dominio:** el `golden_candidate` alimenta el golden set 1.2; q14 turno 1 aporta un caso positivo
  de premisa contradicha y el Space-to-Ground ICD, uno negativo.
