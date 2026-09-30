# pre-M4 · ronda 2 · Traza estructurada en el contrato de respuesta — log de decisiones

> Unidad **off-cycle** (repaso de frontend + cambio de contrato), ejecutada entre la limpieza
> la ronda 2 y el cableado de S09. **Numeración resuelta:** no lleva `SNN`, vive en la ronda 2
> de la auditoría pre-M4 (ver el `README.md` de esta carpeta).
>
> Etiquetado: **[A]** agnóstico (candidato a la Guide) · **[D]** de dominio.

---

## 1. La sub-decisión que el brief dejó abierta (§3.2) — **resuelta**

**Pregunta:** ¿el schema de `/search` es un modelo de evidencia limpio, o mezcla evidencia con
diagnóstico de fusión?

**Hallazgo:** mezcla. `SearchHit` llevaba trece campos de tres naturalezas distintas:

| Naturaleza | Campos | ¿Viaja en una respuesta? |
|---|---|---|
| Cita verificable | `doc_id_cite`, `section_number`, `authority`, `chunk_type`, `requirement_uid`, `cite_text`, `distance`, `oversized` | **Sí** — es la evidencia |
| Direccionamiento | `chunk_id`, `canonical_doc_id` | No — sirven para filtrar y direccionar, no para citar |
| Diagnóstico de fusión | `vector_rank`, `lexical_rank`, `rrf_score` | No — dicen cómo se ordenó la búsqueda, no qué fundamentó la respuesta |

**Resuelto por factorización, no rebotado:** salió limpia en una línea de herencia. El núcleo
`EvidenceChunk` (8 campos) vive en `app/schemas/evidence.py` y `SearchHit` lo **extiende** con
los otros cinco. Ningún campo se movió de sitio en el contrato de `/search`; lo único que cambia
ahí es de dónde se heredan.

**Dos detalles que no eran obvios:**

- **El módulo propio importaba.** Poner el núcleo dentro de `search.py` habría hecho que
  `schemas/query.py` importase `app.services.retrieval` y `app.models.chunk` para transportar
  ocho campos de texto. El contrato de M1 no debe arrastrar la capa de datos por import. **[A]**
- **`oversized` se estrecha en la subclase.** En el núcleo es `bool | None` (el fragmento
  estático de M1 nunca pasó por el chunker, así que la pregunta no aplica); en `SearchHit` se
  redeclara obligatorio, porque un hit **siempre** salió del chunker. La subclase promete más
  que el núcleo, no menos. **[A]**

---

## 2. Decisiones de contrato

### 2.1 Un contrato de API no depende del entorno **[A]**

La respuesta es byte a byte la misma en dev y en producción. Esconder ruido de diagnóstico es
responsabilidad de la **capa de render**, no del backend.

Tres razones, y la tercera es la que decide:

1. No hay nada que proteger: `distance` es un coseno y `oversized` un bool. Ruido, no secreto.
2. Bifurcar por `APP_ENV` deja a los tests validando, en dev, una forma que producción no envía —
   la suite dejaría de ser evidencia sobre el sistema desplegado.
3. La UI que proyecta se puede cambiar sin tocar el backend; un contrato que se bifurca obliga a
   desplegar el backend para cambiar qué se enseña.

### 2.2 Un modelo compartido, nunca un DTO gemelo **[A]**

`/query` y `/search` describen la misma cosa —el chunk sobre el que se sostiene el sistema— así
que comparten modelo. Dos modelos con los mismos campos no son redundancia inofensiva: son dos
sitios que divergen, y el que se lee primero gana sin que nadie lo decida. Es el mismo principio
de «un hogar por decisión» aplicado a los tipos en vez de a la documentación.

### 2.3 Lo que no se midió va a `None`, nunca a un número **[A]**

`distance` es nulo donde la evidencia no se contrastó contra el vector de una query (el fragmento
estático de M1; los hits que solo trajo el canal léxico de la híbrida). Un `0.0` no sería
«desconocido»: se leería como «idéntico a la query», que es la lectura *opuesta*. Mismo criterio
que ya gobierna `cost_usd` cuando el modelo no está en el price map.

### 2.4 Un rechazo vacía las citas, no la evidencia **[A]**

> **Revisada el 2026-09-30, tras la UAT.** La decisión original fue la contraria —`_evidence`
> aplicaba el mismo corte que `_citations`, así que un rechazo no llevaba ni citas ni evidencia—
> y resultó estar mal. Se deja escrita porque el motivo del cambio es el valor del registro.

`citations` y `retrieved_context` son **dos hechos distintos, y solo uno depende de la
respuesta**:

- `citations` = lo que la respuesta cita. Se vacía en un rechazo: adjuntar una cláusula a un
  texto que dice «no lo cubro» es la cita fantasma que el guardrail existe para evitar.
- `retrieved_context` = lo que se recuperó y se inyectó. Un **hecho del sistema**, independiente
  de lo que el modelo hiciera con ello. Viaja siempre.

**Qué destapó el error.** La UAT pidió mirar el modo dev y ahí se vio: con un rechazo, el sidebar
de diagnóstico se queda mudo. Antes `injected_context` estaba siempre, así que se podía inspeccionar
qué se había inyectado; suprimirlo dejaba ciego el caso que más interesa inspeccionar.

**El argumento que decide es S09.** Con el retriever cableado, «trajo estos 5 chunks y el modelo
aun así dijo *not covered*» es la señal principal para diagnosticar la calidad del retrieval.
Suprimirla habría dejado ciego justo el trabajo que viene.

**Y la supresión no desaparece: cambia de capa.** No pintar la traza de un rechazo sigue siendo
correcto —no hay nada que respaldar— pero es decisión de **render**, y vive en el frontend
(`grounded=bool(citations)`). El contrato sigue mandando lo mismo siempre, que es 2.1 intacto.

**Lección agnóstica [A]:** cuando una regla se aplica «igual» a dos salidas, conviene comprobar
que las dos responden a la misma pregunta. Aquí no: una era «¿en qué se apoya esta respuesta?» y
la otra «¿qué se recuperó?». Colapsarlas parecía consistencia y era pérdida de información.

### 2.4-bis Una respuesta vacía no se entrega: se nombra **[A]**

Defecto encontrado **durante la UAT**, ajeno al split de render. Traza real del incidente:

```
tokens_in=0  tokens_out=0  ttft_ms=None  latency_ms=21915.6  finish_reason=stop  cost_usd=None
```

El proveedor cerró el stream sin un solo delta tras 22 s. `ttft_ms=None` lo delata. Y el sistema
lo entregó **como respuesta correcta**: `answer: ""`, `truncated=False`, y su traza de evidencia
detrás. En un producto cuyo valor ES la cita verificable, eso es evidencia respaldando un texto
que no existe.

**Causa raíz:** los defaults del wrapper de streaming. `finish_reason = "stop"` es una
*afirmación* —«el modelo terminó con normalidad»— que nadie verificó, y si el `async for` no
itera ni una vez, sale tal cual. Aguas abajo es indistinguible de un éxito.

**Arreglado en tres sitios:**

1. Un stream sin deltas cierra con `event: error`, no con `done`. El docstring de
   `StreamErrorEvent` ya decía que es «un evento tipado para cada forma en que un stream puede
   acabar sin respuesta»; esto es exactamente eso.
2. `/query` devuelve **502**: la petición estaba bien formada y quien devolvió algo inservible fue
   el proveedor. Ni 500 (culpa nuestra) ni 200 mintiendo.
3. El default del wrapper pasa de `"stop"` a `"unknown"`.

Y no deja turno en la sesión: un turno de asistente vacío se re-inyectaría en la siguiente
pregunta.

**Lecciones agnósticas [A]:** (1) **el valor por defecto de un campo de estado no debe afirmar el
caso bueno** — si el proveedor no lo dijo, el default tiene que decir «no se sabe», porque un
default optimista convierte un fallo en un éxito silencioso; (2) **una salida vacía es un
resultado, no una respuesta**.

**Queda abierto:** `truncated` sigue siendo `finish_reason == "length"`, así que con `"unknown"`
dice «no truncada» cuando en realidad no se sabe. Hacerlo tri-estado cambia contrato y render.

### 2.5 La autoridad se declara en el dato, no se infiere en el código **[D]**

Para dar forma de átomo al fragmento estático hacían falta `authority` y `chunk_type`, que
`ECSS_REFERENCE` no llevaba. Se añadieron **al dato**, copiados a mano del documento publicado
igual que ya lo estaba el texto: son propiedades que el estándar imprime.

El código los lee **sin default** (`f["authority"]`, no `f.get(...)`), así que un fragmento que no
los declare rompe en vez de entrar en la respuesta con una autoridad fabricada. Es la contrapartida
de una regla vieja del proyecto: una NOTE informativa presentada como normativa es el fallo que el
chunking gastó un brief entero en evitar.

### 2.6 Granularidad = chunk; la cláusula queda diferida **[D]**

La agrupación por `(doc_id_cite, section_number)` que pinta el frontend es **de vista**: sale del
propio hit set y no trae nada nuevo. La recomposición de cláusula —traer los hermanos que el
retrieval no devolvió— sigue en §7 de `ARCHITECTURE.md`, y el átomo ya carga la coordenada por la
que esa capa entrará.

---

## 3. Decisiones de frontend

- **`FRONTEND_ENV` es del frontend.** Otro servicio, otra imagen: este proceso no puede leer el
  `APP_ENV` del backend, y fingir que sí sería el acoplamiento que la frontera prohíbe. **[D]**
- **Se compara contra `"development"`, no contra `"production"` [A].** Con la comparación
  invertida, una errata al desplegar (`prod`, `PRODUCTION`) habría dejado el diagnóstico visible.
  Así, un valor desconocido **oculta**. El default de un interruptor de exposición se elige por
  qué pasa cuando alguien se equivoca, no por qué pasa cuando acierta.
- **`authority` es campo de usuario, no de diagnóstico [D].** Un ingeniero necesita saber si lo
  que lee es normativo o una NOTE informativa; `distance` y `oversized`, en cambio, no le dicen
  nada y solo se pintan en dev.
- **La traza sustituye al expander de citas**, que era un subconjunto estricto (las mismas
  coordenadas, sin el verbatim que las hace verificables). Hoy `citations` y `retrieved_context`
  describen el mismo conjunto **salvo en un rechazo**, que es justo el caso que motivó separarlas
  (2.4); cuando el filtro de fan-out estreche las citas dejarán de coincidir también en el caso
  normal y la UI tendrá que distinguirlas — anotado en §7.
- **`FRONTEND_ENV` tenía que ser sobreescribible desde el host** (`${FRONTEND_ENV:-development}`).
  Estaba fijada a pelo en el compose, así que el modo producción no se podía levantar sin editar
  el fichero — y una instrucción de UAT que exportaba la variable habría enseñado el modo dev
  haciéndolo pasar por prod. **[A]** Un interruptor que no se puede accionar sin editar código no
  es un interruptor.

---

## 4. Lo que este brief NO hizo

- No tocó `get_context`: el contexto sigue siendo el fragmento estático. Cablear el retriever es
  S09.
- No implementó recomposición ni agrupación real por cláusula.
- No tocó `GET /api/v1/prompt` (su exposición en producción es securización de endpoint).
- No reintrodujo ningún flag opt-in ni ninguna lectura de `APP_ENV` en la respuesta.
- No añadió `FRONTEND_ENV` a `.env`/`.env.example`: `BACKEND_URL` tampoco está ahí, porque el
  frontend no lee ese fichero. Declararla en los dos sitios fingiría una fuente que no existe, y
  la excepción queda escrita en `CLAUDE.md` junto a la regla.

---

## 5. Candidatos agnósticos para la Guide — **anotados, no graduados**

1. **Un contrato de API no depende del entorno.** Proyectar es del render; el backend manda lo
   mismo siempre. Corolario: un contrato que se bifurca por entorno hace que los tests dejen de
   ser evidencia sobre lo desplegado.
2. **Un modelo compartido, nunca un DTO gemelo** — «un hogar por decisión», aplicado a los tipos.
3. **Lo que no se midió se admite, no se rellena.** Un valor por defecto en un campo de medida no
   dice «desconocido»: dice otra cosa, y normalmente la contraria.
4. **Una subclase puede prometer más que su base, nunca menos** — así se factoriza un núcleo sin
   aflojar el contrato de quien ya lo cumplía.
5. **El default de un interruptor de exposición se elige por el caso del error**, no por el del
   acierto.
6. **Cuando falta un dato para dar forma a algo, se declara en la fuente y se lee sin default.**
   El `get(...)` con valor por defecto es donde se cuelan los datos inventados.

---

## 6. Verificación ejecutada

- `uv run pytest`: **919 verde**, 8 deselected (913 antes; 6 del schema nuevo y los reescritos de
  contrato).
- Contrato comprobado **contra el contenedor real** (`docker compose up -d --build api frontend`)
  leyendo `/openapi.json`: `QueryRequest` sin `include_context`, `QueryResponse` con
  `retrieved_context`, `EvidenceChunk` con sus 8 campos y `SearchHit` con los 13.
- `ruff check`: limpio salvo un `E402` preexistente en `app/main.py`, ajeno a esta unidad.
- **UAT visual cerrada el 2026-09-30.** Dev pinta expander + etiqueta de autoridad + átomo con
  `distance`/`oversized` en `null`; prod oculta el bloque `Diagnostics` y la línea de diagnóstico
  del expander, conservando la traza; `FRONTEND_ENV=PRODUCTION` (errata) se comporta como prod,
  así que el guardarraíl falla hacia ocultar. Un falso positivo durante la UAT resultó ser una
  pregunta mal elegida —el fragmento dice *que* el cliente debe especificar, no *cuáles* son, así
  que el guardrail de scope rechazaba correctamente— y el otro fue el defecto de 2.4-bis.
- **Suite tras el cierre: 923 verde.** Los tres caminos verificados contra el contenedor real:
  pregunta respondible, rechazo (citas vacías, evidencia presente) y stream vacío.
