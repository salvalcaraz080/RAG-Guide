# S09 — Brief: cablear retrieval → generación

> Funcionalidad única: `/api/v1/query` y `/api/v1/query/stream` responden sobre el corpus
> completo en pgvector en lugar del fragmento estático de M1, con citas por cláusula.
> Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar: este brief asume su contexto.

---

## 0. Objetivo y alcance

**Objetivo.** Sustituir el fragmento estático que devuelve `get_context` por evidencia
recuperada del corpus, ensamblada por cláusula, y que la respuesta cite solo lo que el modelo
referencia.

**Dentro de alcance:** seam de contexto inyectable, proveedor respaldado por retrieval,
agrupación por cláusula, bloque de contexto, prompt, marcadores de cita y fan-out, historial
con citas mapeadas, validación de citas literales en modo observación, logging de retrieval,
render numerado en el frontend, diagnóstico de cobertura de cláusula.

**Fuera de alcance (no implementar):** reformulación de query con LLM, threshold por chunk o
puerta de abstención por distancia, ponderación de la fusión híbrida, reranking, expansión por
grafo, expansión de cláusula con chunks no recuperados, filtros por área, rate limiting,
idempotencia, caché, política de reintento ante cita inválida. Si una pieza parece exigir algo
de esta lista: `# TODO: pendiente de decisión en Claude Chat` y seguir.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| D1 | `llm_service` recibe un **proveedor de contexto inyectado**; no importa `db`, `models` ni `retrieval` | Módulos con un único cometido. El servicio LLM no conoce la base |
| D2 | El retrieval ocurre **dentro de `_prepare`**, en el mismo punto que hoy `get_context`: después de guardrails y de resolver sesión | Invariante *guardrails antes del primer efecto*. Recuperar en el router antes de llamar a `llm_service` lo rompería: embedding y consulta son efectos |
| D3 | El proveedor **gestiona su propia sesión de BD**, corta, por llamada (`async with` sobre el sessionmaker) | No depender de una sesión request-scoped cuyo ciclo de vida frente a SSE es frágil |
| D4 | Modo de retrieval para generación: **`vector`**, configurable | S08 midió que RRF sin pesos degrada las queries genéricas; ponderar es S10 |
| D5 | Unidad de contexto y de cita = **cláusula**: los hits se agrupan por `(canonical_doc_id, section_number)` | Resuelve clónicos y da la cita que promete el producto |
| D6 | **Sin expansión**: un grupo contiene solo los chunks recuperados de esa cláusula | Hay cláusulas con hasta 59 requisitos; recomponer completa sin medir es arriesgado. El brief mide (§3.8) |
| D7 | **Sin threshold ni puerta por distancia** | Medido en S08: la distancia no separa "fuera del corpus" de "genérico". La abstención sigue en el generador (frases marcador existentes) |
| D8 | Salida en **prosa streameada** con marcadores numéricos `[n]` y citas literales entre comillas; **no** structured output | Structured output rompe el streaming token a token. Los marcadores se procesan en `_finish`, sobre el texto completo |
| D9 | `[n]` es el handle opaco: `n` = posición de la cláusula en el bloque de contexto de **esta** petición | El modelo no puede citar una cláusula que no se le pasó |
| D10 | `citations` = **solo las cláusulas referenciadas** por marcadores válidos | Fan-out: hoy se citan todos los fragmentos inyectados, cite el modelo lo que cite |
| D11 | El turno que se guarda en el historial lleva los `[n]` **reescritos a identificadores naturales** | Un `[1]` del turno anterior apuntaría a otra cláusula en el turno siguiente |
| D12 | Validación de cita literal **normalizada, en modo observación**: se registra, no altera la respuesta | La política ante fallo se decide en S11 con la tasa medida |
| D13 | Fallo de embedding en `/query` → **fail-closed** (503 / `event: error`) | Sin contexto no hay respuesta anclada. A diferencia de la moderación, aquí no cabe fail-open |

---

## 2. Contexto del código actual (lo relevante)

- `llm_service._prepare(question, session_id)`: guardrails → `_resolve_session` →
  `get_context(question, history)` → `_build_context_block` → `_compose_system_prompt`.
  `answer_query` y `stream_answer_query` comparten `_prepare` y `_finish`.
- `services/retrieval.py`: coseno exacto + canal léxico + RRF; hits con `doc_id_cite`,
  `canonical_doc_id`, `section_number`, `chunk_type`, `authority`, `requirement_uid`,
  `cite_text`, `distance`, rangos por canal. Excepciones `RetrievalError`,
  `QuerySpaceMismatch`, `EmbeddingCredentialsMissing`.
- `schemas/evidence.py::EvidenceChunk` viaja en `retrieved_context`; `schemas/query.py::Citation`
  transporta `standard` + `clause`.
- `_citations(text, fragments)` devuelve hoy todos los fragmentos; vacía en rechazo.
- `prompts/query/system.md`: rol, grounding, citación, formato, scope con frases marcador
  `Out of scope:` / `Not covered by the provided material:`.

---

## 3. Trabajo

### 3.1 Seam de contexto inyectable (D1, D2, D3)

Define en `app/services/` un tipo para el proveedor, por ejemplo:

```python
class ContextProvider(Protocol):
    async def __call__(self, question: str, history: list[Message]) -> list[EvidenceChunk]: ...
```

- `answer_query` / `stream_answer_query` reciben el proveedor como parámetro; `_prepare` lo invoca
  donde hoy llama a `get_context`. Ningún import de `db`, `models` o `retrieval` en
  `llm_service.py`.
- Nuevo módulo (p. ej. `app/services/context.py`) con el proveedor respaldado por retrieval: abre
  su propia sesión por llamada, llama a `retrieval` con el modo y `top_k` de configuración y
  devuelve los hits como `EvidenceChunk` en orden de rango.
- `routers/queries.py` obtiene el proveedor por `Depends` y lo pasa. El router **no** recupera
  nada ni toca la sesión de BD.
- El fragmento estático de M1 sale del camino de producción. Si `context/reference.py` queda sin
  uso fuera de tests, elimínalo y usa fakes en los tests; si algún test de contrato lo necesita
  como fixture, muévelo a `tests/`.
- Settings nuevos: `RETRIEVAL_MODE` (default `vector`, valores `vector|hybrid`) y
  `RETRIEVAL_TOP_K` (default `10`, knob sin calibrar). A `.env` y `.env.example`.
- Errores (D13): `EmbeddingCredentialsMissing` y fallos de red del embedding → 503 en `/query`,
  `event: error` tipado en el stream. `QuerySpaceMismatch` → 500. Guardrails siguen corriendo
  antes que todo esto (test que lo fije).

### 3.2 Agrupación por cláusula (D5, D6)

Función pura (sin BD), testeable:

- Entrada: hits ordenados por rango. Salida: grupos ordenados por el mejor rango de sus chunks.
- Clave: `(canonical_doc_id, section_number)`.
- Dentro del grupo, orden de documento si es derivable de forma estable (p. ej. ordinal del
  `chunk_id`); si no lo es, conserva el orden de rango y deja `TODO`.
- Cada chunk conserva `authority`, `chunk_type` y `requirement_uid`: **una cláusula puede mezclar
  normativo e informativo, y esa distinción no se pierde al agrupar.**
- Límite conocido: 4 números de cláusula repetidos dentro de un mismo documento (handbooks) se
  fusionarían. Se acepta; documéntalo.

### 3.3 Bloque de contexto (D9)

Sustituye la construcción actual en `_build_context_block` por un formato por cláusula:

```xml
<source id="1" standard="ECSS-E-ST-40C" clause="5.8.3.5">
  <item authority="normative" type="requirement" uid="E40-123">verbatim…</item>
  <item authority="informative" type="note">verbatim…</item>
</source>
```

- `id` = posición 1..n del grupo. `standard` = `doc_id_cite` (con revisión).
- El texto es `cite_text`, **nunca** `embed_text`.
- Sin `distance` ni rangos en el bloque.
- Sin truncamiento: con `top_k` chunks de ≤512 tokens (dos *oversized* de ~700) el bloque está
  acotado por construcción. Registra su tamaño en tokens (§3.7).
- El bloque se construye **una vez** por petición y de él salen prompt, mapa `n → cláusula`,
  citas y evidencia (principio de procedencia única ya vigente).

### 3.4 Prompt (`prompts/query/system.md`)

Mantén rol, grounding, scope y las dos frases marcador **exactas**. Añade o reescribe:

1. **Citación:** toda afirmación basada en una fuente termina con su marcador `[n]`, usando solo
   los `id` presentes en `<source>`. Nunca citar por número de cláusula de memoria.
2. **Cita literal:** al enunciar el contenido de un requisito, incluir el tramo **mínimo**
   literal entre comillas dobles, seguido de su marcador: `"…" [n]`. Sin elipsis dentro de las
   comillas; sin parafrasear dentro de ellas.
3. **Peso normativo:** "shall" = requisito; "should" = recomendación; "may" = permiso; los
   `item` con `authority="informative"` (NOTE, prosa informativa) no son vinculantes y no se
   presentan como requisito.
4. **Cobertura:** si se enumeran requisitos, indicar que la lista se limita a las fuentes
   proporcionadas.
5. **Insuficiencia:** "Not covered by the provided material:" se refiere a las fuentes
   recibidas, no a afirmar que ECSS no regula el tema.

Recordatorio breve de las reglas 1 y 2 al final del mensaje de usuario, si el compositor actual
lo permite sin romper el orden estático→variable del prompt. Si no, déjalo en el system y anótalo.

### 3.5 Marcadores, fan-out e historial (D10, D11)

En `_finish` (no en el tramo SSE):

- Extrae los marcadores `[n]` del texto final (admite `[1]`, `[1, 3]`, `[1][3]`).
- `citations` = cláusulas con marcador válido, deduplicadas, en orden de primera aparición.
  `Citation` gana `ref: int` (el `n` del marcador) además de `standard` y `clause`.
- Marcadores fuera de rango (`[7]` con 6 fuentes): no generan cita; se registran como inválidos.
- Rechazo (frases marcador) → `citations = []`, como hoy. `retrieved_context` viaja siempre.
- Antes de `session_store.append_turn`, reescribe en el texto del asistente cada `[n]` válido a
  `[<standard> §<clause>]`. El texto **devuelto al cliente** conserva los `[n]`.

### 3.6 Validación de citas literales — modo observación (D12)

Función pura:

- Extrae pares `"cita" [n]` (comillas rectas y tipográficas).
- Normaliza **ambos lados** igual: Unicode NFKC, espacios colapsados, guiones de partición
  unidos, comillas y guiones tipográficos unificados, casefold.
- Resultado por par: `verified` (substring de algún `item` del grupo `n`), `not_found`,
  `invalid_ref`. Además, conteo de marcadores sin cita literal.
- **No modifica la respuesta.** Solo log estructurado (§3.7).

### 3.7 Logging

Un evento de retrieval y uno de validación por petición, con `request_id` (ya vía contextvars):

- `retrieval_completed`: modo, `top_k`, lista de `{chunk_id, rank, distance}`, cláusulas
  resultantes tras agrupar, tokens del bloque de contexto, latencia de embedding y de BD
  por separado.
- `citations_validated`: marcadores válidos / inválidos, cláusulas citadas vs inyectadas,
  `verified` / `not_found` / `invalid_ref`, marcadores sin cita literal.
- Sin texto de la pregunta ni de la respuesta (principio vigente: metadata, no contenido).

### 3.8 Diagnóstico de cobertura de cláusula (insumo de la próxima decisión)

Script en `scripts/` (investigación de sesión, no herramienta permanente). Sobre un conjunto fijo
de queries — el de S08 si existe, más las tres preguntas típicas del producto —, ejecuta el
retrieval de generación y reporta por cláusula recuperada:

- chunks recuperados / chunks totales de la cláusula, por `chunk_type`;
- tokens del grupo recuperado vs tokens de la cláusula completa;
- si la cláusula tiene chunk `section` no recuperado.

Agregado: % de cláusulas recuperadas incompletas y distribución de tamaño de la cláusula
completa. Gasta embeddings reales: pregunta antes de ejecutarlo. Comprueba además en qué idioma
está el conjunto de queries de S08 y repórtalo.

### 3.9 Frontend

- Pintar `citations` como lista numerada por `ref`, coherente con los `[n]` del texto.
- La traza de usuario (`retrieved_context`) sigue igual, pero ahora **lo citado y lo recuperado
  difieren**: distingue visualmente las cláusulas citadas de las solo recuperadas.
- Sin lógica nueva en el frontend más allá de proyectar el contrato.

---

## 4. Tests de contrato (sin red, sin BD)

Como mínimo:

- `llm_service` no importa `db`/`models`/`retrieval` (test de import o de dependencia).
- Orden: una pregunta tóxica nunca invoca al proveedor de contexto; un `session_id` inválido
  tampoco (ambos caminos, `/query` y stream).
- Agrupación: clónicos colapsan; orden por mejor rango; autoridad preservada por item.
- Bloque de contexto: usa `cite_text`; ids 1..n; sin `distance`.
- Marcadores: válidos, múltiples, fuera de rango, rechazo → `[]`, deduplicación y orden.
- Historial: se guarda con identificadores naturales; la respuesta conserva `[n]`.
- Validación literal: normalización (guiones de partición, ligaduras, espacios, comillas
  tipográficas), `verified` / `not_found` / `invalid_ref`.
- Errores: embedding ausente → 503 y `event: error`; mismo veredicto con solo
  `ANTHROPIC_API_KEY` en el entorno (proveedor de contexto mockeado).

## 5. UAT propuesta (para el usuario)

CC redacta el script de UAT en Streamlit con, al menos: pregunta con identificador explícito
(E-ST-40C, memoria dinámica), par de criticidad B/C de Q-ST-80, pregunta enumerativa (qué
estándares aplican a software de vuelo crítico), seguimiento anafórico ("and in Q-ST-80?"),
fuera de dominio y fuera de material. En cada caso, qué mirar: marcadores ↔ lista de citas,
citas literales visibles, autoridad en la traza, y los dos eventos de log.

## 6. Cierre

- `S09-decisions.md` con cada decisión tomada durante la implementación + porqué, etiquetada
  **agnóstico/dominio**. Incluye los resultados del diagnóstico §3.8.
- `CHANGELOG.md`, `CLAUDE.md` y `ARCHITECTURE.md` según sus reglas (presente, no diario). En
  `ARCHITECTURE.md`: el flujo de petición de §3 pasa a M2; §7 pierde "retriever no cableado" y
  "fan-out sin decidir", y gana lo que quede abierto (expansión de cláusula, política de cita no
  verificada, seguimientos conversacionales).
- `uv run pytest` en verde y mensaje de commit propuesto. El commit lo hace el usuario.
