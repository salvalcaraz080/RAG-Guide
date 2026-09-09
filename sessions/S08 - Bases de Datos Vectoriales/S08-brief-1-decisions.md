# S08 · Decisiones — Brief 1 (infraestructura pgvector + schema)

> Decisiones crudas tomadas al implementar, etiquetadas **[AGNÓSTICO]** (transferible a
> cualquier sistema RAG) o **[DOMINIO]** (específico de ECSS / de este proyecto).
> Ejecutado por Claude Code sobre el brief de Claude Chat.
> **Verificación completa:** suite de contrato (754 tests) + UAT sobre Postgres real, los 8 pasos
> verdes (`S08-brief-1-uat.md`).

---

## 1. Campos de S07 que NO mapean limpio al schema

Esto es lo que el brief pidió marcar explícitamente en vez de adivinar.

### 1.1 `requirement_uid` no existe como campo en `Chunk` — **[DOMINIO]**

El schema declara `chunks.requirement_uid TEXT NULL`, pero el `Chunk` de S07 **no tiene ese
campo**. El UID solo sobrevive **dentro del `chunk_id`**, y solo cuando el requisito lo lleva:

```
ECSS-E-ST-10-03C#4.1::ECSS-E-ST-10-03_0750001   ← con UID
ECSS-E-HB-40A#C.2.1::r0                          ← sin UID: ordinal posicional
```

Medido sobre los 7.922 requisitos reales: **2.536 con UID (32%)**, 5.386 sin. La columna queda
declarada y nullable; **no se puebla en este brief** (este brief no escribe filas).

Las dos salidas, para el brief 2:
- **(a)** añadir `requirement_uid` al `Chunk` del chunker y re-generar `chunks.json` — el dato
  existe en `Requirement.uid`, el chunker simplemente no lo propaga;
- **(b)** parsear el sufijo del `chunk_id`, que es derivar identidad de una cadena en vez de
  leerla del origen, con la trampa de que `::r0` es ordinal y no UID.

Recomendación de CC: **(a)**. Es el mismo criterio que separó `embed_text` de `cite_text` — un
dato de identidad se transporta, no se re-deriva.

### 1.2 Campos de `Chunk` sin columna propia — **[AGNÓSTICO]**

| Campo de S07 | Estado |
|---|---|
| `token_count` | sin columna. Es una propiedad de ventana, no de identidad |
| `oversized` | sin columna. 2 casos en el corpus, marcados en el artefacto |
| `outgoing_refs` | sin columna, **y así debe seguir**: el brief lo prohíbe hasta validar el extractor |
| `metadata.revision` (`""` o `"Rev.1"`) | sin columna. Es recuperable de `documents.doc_id_cite` |
| `metadata.corpus_version` | sube a `documents.corpus_version`: es propiedad del documento, no del chunk |
| `metadata.anchor` | duplicado de `Chunk.anchor` → una sola columna, `chunk_type` |

El `metadata` JSONB existe para los tres primeros, pero **cuáles entran es decisión del brief 2**,
que es quien escribe filas. Declararlo hoy sería inventar el contrato del que aún no escribe.

### 1.3 `documents.source_path` no está en ningún artefacto — **[AGNÓSTICO]**

No es un campo de `Chunk`: es la procedencia (`data/processed/<doc>.json`), que solo conoce el
proceso de ingesta al recorrer el directorio. Se declara y lo puebla el brief 2. No hay ambigüedad
aquí, se anota para que no parezca un olvido.

### 1.4 Lo que sí mapeó limpio, verificado sobre los artefactos reales

- `canonical_doc_id` → `cite` es **1:1** en los 26 documentos (comprobado, no asumido): ninguna
  identidad canónica apunta a dos identidades de cita, así que `UNIQUE(canonical_doc_id)` es
  satisfacible.
- `authority` toma exactamente dos valores en los 13.876 chunks (10.320 normative / 3.556
  informative) — **cero vacíos**, así que el CHECK no rechaza nada existente.
- `anchor` toma exactamente las cinco anclas declaradas (requirement 7.922 · section 2.583 ·
  table 1.481 · note 1.410 · glossary_term 480).
- `chunk_id` es único en los 13.876 → `UNIQUE` satisfacible, y por tanto sirve de clave natural
  para el upsert del brief 2.

---

## 2. Decisiones de implementación tomadas dentro del margen del brief

### 2.1 `DATABASE_URL` con default, no obligatoria — **[AGNÓSTICO]** ⚠️ divergencia leve del brief

El brief pedía «falla al arrancar si falta, según el patrón existente». Se implementó **con
default** (`postgresql+asyncpg://rag_ecss:rag_ecss@localhost:5432/rag_ecss`, que es literalmente
el servicio `db` de compose) por un invariante escrito en `CLAUDE.md`: *la suite de contrato debe
dar el mismo veredicto en un clone con solo `ANTHROPIC_API_KEY`*. Una `DATABASE_URL` obligatoria
pondría en rojo la suite entera de cualquiera que clone el repo sin ella.

**A cambio, el fail-fast se puso donde de verdad hay algo que puede estar mal:** un
`field_validator` que **rechaza en el arranque una URL síncrona**. `create_async_engine` sobre
`postgresql://` no falla al arrancar sino al conectar — lejos de su causa. El validador cazó de
inmediato el `.env` real del repo, que traía justo esa URL heredada de un `.env.example` de M2.

Lección transferible: *un default correcto no es un valor de relleno; la validación de arranque
vale para lo que puede estar mal, no para lo que puede faltar.*

### 2.2 Engine y sessionmaker perezosos, no globales de módulo — **[AGNÓSTICO]**

`get_engine()` / `get_sessionmaker()` tras `lru_cache`, no `engine = create_async_engine(...)` a
nivel de módulo. Un global obligaría a leer `Settings` en el import, y entonces importar un modelo
—o que Alembic importe `app.models`— exigiría un `.env` completo. Con la construcción diferida, el
schema entero se testea sin Docker y sin `.env`: `MetaData` es una descripción **en proceso**.
`lru_cache` además hace del engine un singleton, que es lo que se quiere: el engine **es** el pool,
y uno por petición anula el pooling.

### 2.3 Las constantes del vocabulario se duplican, y la copia se ata con un test — **[AGNÓSTICO]**

`CHUNK_TYPES` y `AUTHORITIES` se declaran en `app/models/chunk.py` en vez de importarse de
`corpus/chunking.py`. Importar `corpus/` metería python-docx, lxml y networkx en el runtime del
servidor para nombrar siete cadenas. La duplicación se paga con un test que compara
`AUTHORITIES == (NORMATIVE, INFORMATIVE)` contra el original: *duplicar está bien si la copia
tiene una guarda que avisa cuando el original se mueve.*

### 2.4 Cascade declarado por los dos caminos — **[AGNÓSTICO]**

`ON DELETE CASCADE` en la FK **y** `cascade="all, delete-orphan"` + `passive_deletes=True` en la
relación. No es redundancia: el primero cubre el `DELETE` en SQL directo (el que hará una
re-ingesta), el segundo el borrado a través de la sesión ORM. Con solo uno de los dos, el otro
camino deja huérfanos o carga las 13.876 filas en memoria para borrarlas una a una.

### 2.5 La migración inicial, escrita a mano y verificada en modo offline — **[AGNÓSTICO]**

`autogenerate` compara los modelos contra una base viva, que la suite de contrato no tiene. Escrita
a mano, se verificó con `alembic upgrade head --sql`, que **genera el DDL completo sin conectar a
nada** — una verificación que CC sí puede ejecutar. Además se añadió un test de deriva
modelo↔migración: toda columna de `Base.metadata` tiene que aparecer en el fuente de la migración,
porque una columna añadida a un modelo sin migrar no daría error hasta el primer `INSERT` real.

### 2.6 La extensión se habilita dentro de la migración — **[AGNÓSTICO]**

`CREATE EXTENSION IF NOT EXISTS vector` como primera operación del `upgrade()`, no un paso manual
de setup ni un `init.sql` del contenedor. Así **toda base que llegue a `head` tiene el tipo
`vector`**, incluida la de un CI futuro y la de cualquiera que clone. El `downgrade()`
deliberadamente **no** la elimina: puede estar en uso por otro schema de la misma base.

### 2.7 El atributo se llama `meta`, la columna `metadata` — **[AGNÓSTICO]**

`metadata` es un nombre reservado en la base declarativa de SQLAlchemy (es el objeto `MetaData` del
schema). El mapeo explícito `mapped_column("metadata", JSONB, ...)` conserva el nombre de columna
que pedía el brief sin chocar con el ORM.

---

## 3. Lo que se dejó explícitamente sin hacer

- **Ningún índice vectorial**, ni `halfvec`. Es la decisión de régimen exacto del brief, y está
  protegida por un test que falla si aparece un `hnsw`/`ivfflat` en los modelos o en la migración.
- **Ningún B-tree de filtro** sobre `authority`/`chunk_type`/`section_number`.
- **Ninguna tabla ni columna de referencias poblada.**
- **Cero contacto con el camino de petición**: `routers/`, `llm_service.py`, `get_context` y
  `context/reference.py` intactos. La capa de datos no la importa nadie todavía — el test que lo
  verifica es que la suite entera pasa sin que exista una base de datos.

---

## 4. Hallazgos incidentales

### 4.1 Chunks y vectores son 1:1 — el desajuste que avisaba el brief no existe hoy

El brief advertía de «13.888 chunks en un sitio y 13.876 vectores en otro». Medido sobre los
artefactos reales: **13.876 chunks y 13.876 vectores**, mismo `input_fingerprint`
(`ab3fce8b5cb1a0d4`) en `chunks.json` y como `chunks_fingerprint` en el sidecar. El 13.888 es una
cifra desfasada de la documentación, no un desajuste. El schema nullable soporta el caso
igualmente, y el brief 2 debe seguir reconciliando en vez de asumir 1:1 — pero hoy no hay nada que
reconciliar.

### 4.2 El contenedor `api` recibirá una `DATABASE_URL` equivocada en cuanto la use — **[AGNÓSTICO]**

`api` carga `env_file: .env`, cuyo `DATABASE_URL` apunta a `localhost`. Dentro de Compose el host
es `db`. **Hoy no importa** (la API no consulta la base) y por eso no se ha tocado: añadir un
override para una configuración que nadie lee sería afirmar algo que el sistema no hace todavía.
Importará en el brief 3, y entonces es un `environment: DATABASE_URL=...@db:5432/...` de una línea.

### 4.3 `EMBEDDING_PROVIDER` no la lee nadie

Está en `.env` y `.env.example` desde M2. Verificado con grep sobre todo el repo: cero llamantes.
Un solo proveedor detrás de un solo seam no necesita el interruptor. Se deja (no se borra
configuración de nadie por iniciativa propia) pero queda anotado en `CLAUDE.md`.

### 4.4 Un validador nuevo en `Settings` tumba los contenedores ya creados — **[AGNÓSTICO]**

Al arrancar Docker Desktop, `rag-ecss-api` entró en bucle de reinicio. No era el código: Compose
**congela `env_file` en el momento de crear el contenedor**, así que ese contenedor llevaba dentro
el `DATABASE_URL` viejo (`postgresql://`), mientras que el bind-mount `./app:/app/app` le servía el
`config.py` nuevo. Env viejo + validador nuevo = fail-fast, que es literalmente lo que se pidió que
hiciera. Se arregla recreando el contenedor (`docker compose up -d --force-recreate api`), hecho y
verificado (`/health` responde).

Lección transferible: *añadir una validación de arranque a la configuración invalida los
contenedores existentes, no solo el código. Con `env_file` la variable vive en el contenedor, no en
el fichero.*

### 4.5 PowerShell borra las comillas dobles al llamar a un `.exe` — **[AGNÓSTICO]**

La primera versión del paso 7 bis del guion de UAT usaba `python -c` con el script entre
comillas. Al pasar argumentos a un ejecutable nativo, PowerShell **elimina las comillas dobles**,
así que el script llegaba con las cadenas desnudas y moría con `SyntaxError: invalid decimal
literal`. Falló en manos de Salva, no en las de CC: CC había ejecutado ese paso desde bash y
**tradujo el comando a PowerShell sin volver a probarlo**.

Arreglado pasando por fichero (`Set-Content` es un cmdlet, no un `.exe`, y no sufre el problema),
y re-verificados **todos** los bloques del guion en PowerShell 5.1.

Lección transferible, y la de verdad no es la de las comillas: *un comando traducido a otro shell
es un comando sin ejecutar. Se prueba en el shell donde va a correr, o no se entrega.*

### 4.6 Contenedor huérfano `rag-ecss-redis`

Compose avisa en cada comando: sobrevive un contenedor `redis:7-alpine` del servicio que S05
retiró del `docker-compose.yml`. Está corriendo y sin usarlo nadie. **No se ha tocado** — es
infraestructura del usuario, no residuo de este brief. Se limpia con
`docker compose up -d --remove-orphans` o `docker rm -f rag-ecss-redis` cuando Salva quiera.

### 4.7 `CHANGELOG.md` tiene dos entradas distintas etiquetadas `[0.26.0]`

S07 brief 5 (2026-09-01) y S07 brief 6 (2026-09-04) comparten número de versión. No se ha tocado
—es histórico ya commiteado y arreglarlo no es de este brief— pero conviene saberlo. La entrada de
S08 va como `[0.27.0]`.
