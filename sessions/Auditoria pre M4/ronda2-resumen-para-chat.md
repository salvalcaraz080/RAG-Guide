# RAG-ECSS · Resumen para Claude Chat — cierre de la ronda 2 de pre-M4

> **Para qué es este archivo:** ponerte al día de todo lo hecho desde el cierre de S08 hasta hoy,
> y dejar sobre la mesa lo que queda por decidir antes de S09. Autocontenido: no hace falta abrir
> el repo para leerlo.
>
> **Fecha:** 2026-09-30 · **Rama:** `main` · **Suite:** 923 verde, 8 deselected
> **`input_fingerprint`:** `ab3fce8b5cb1a0d4` — sin mover en todo el tramo, así que **no se ha
> re-chunkeado ni re-embebido nada**.

---

## 1. Dónde está el sistema

**M1 (CAG) responde en producción. M2 (RAG) está construido y sin cablear.**

El corpus está parseado, chunkeado (13.876 chunks), embebido y **cargado en pgvector** (13.695
filas: la carga descarta 181 marcadores no citables). Hay una capa de retrieval que consulta la
base con dos modos —coseno exacto e híbrida con canal léxico fundido por RRF— y un endpoint
`POST /api/v1/search` que la expone.

**Lo que falta es un cable, literalmente uno.** `llm_service.get_context()` sigue devolviendo el
fragmento estático de `ECSS-E-ST-40C 5.2.4.8`. Sustituir ese `return` por recuperación semántica
es S09, y toca **un solo punto** porque los dos caminos de entrega (`/query` y `/query/stream`)
comparten preparación desde el refactor.

---

## 2. Qué se hizo en este tramo (5 commits)

Todo fue **preparación**, no funcionalidad nueva. El objetivo era llegar a S09 con el terreno
despejado, y se cumplió.

| Commit | Qué |
|---|---|
| `b2ce0ce` | Rutas centralizadas (`corpus/paths.py`), código muerto fuera, y **fix: un re-parseo ya no borra `embeddings.json`** dejando el `.npy` huérfano |
| `3b37595` | `identity.py` y `docx_text.py` como módulos propios (rompen dos ciclos de import); `_prepare`/`_finish` compartidos en `llm_service` — **el punto único donde entra S09** |
| `f9c8204` | **Fix: la conservación de tablas se medía contra sí misma.** El chequeo comparaba la salida del parser con la salida del parser, así que no podía detectar su propia omisión. Ahora se mide contra el `w:tbl` del DOCX crudo |
| `f4fc2e5` | Un hogar por decisión: `CLAUDE.md` 1.133 → **263 líneas**, estándar de docstring de tres campos (265 → 0 fuera del estándar), **227 → 0** citas a briefs en el código |
| `3d83fe4` | Traza estructurada en el contrato de respuesta (ver §3) |

Además, hoy: cierre de la UAT de la traza y **un defecto de streaming que destapó** (ver §4).
Eso está **sin commitear**, esperando tu revisión de este resumen.

---

## 3. El contrato de traza (commiteado)

`injected_context: str` (blob opt-in de diagnóstico) se sustituyó por
**`retrieved_context: list[EvidenceChunk]`**, que viaja en `QueryResponse` y en el evento `done`
del stream.

**El átomo es un solo modelo.** `EvidenceChunk` (8 campos: `doc_id_cite`, `section_number`,
`authority`, `chunk_type`, `requirement_uid`, `cite_text`, `distance`, `oversized`) y `SearchHit`
lo **extiende** con lo que solo tiene sentido midiendo retrieval: `chunk_id`, `canonical_doc_id`
y los rangos por canal de la fusión. Esos cinco son diagnóstico de `/search`, no evidencia, y por
eso no viajan en una respuesta.

**Principios que se aplicaron** (los tres primeros son candidatos a la Guide, §6):

- **Un contrato de API no depende del entorno.** Viaja siempre, sin flag y sin mirar `APP_ENV`.
  Esconder ruido de diagnóstico es de la capa de render.
- **Un modelo compartido, nunca un DTO gemelo.** Dos modelos con los mismos campos son dos sitios
  que divergen.
- **Lo que no se midió va a `None`, nunca a un número.** `distance` es nulo en el fragmento
  estático y en los hits solo-léxicos; un `0.0` se leería como «idéntico a la query».
- **La UI proyecta**, gobernada por `FRONTEND_ENV` —variable **del frontend**, no el `APP_ENV` del
  backend— y comparada contra `"development"` para que una errata **oculte** en vez de filtrar.

---

## 4. La UAT de hoy: pasó, y encontró un defecto aparte

**El split dev/prod pasa.** Dev pinta expander + `⚖️ normative · requirement` + el átomo con
`distance`/`oversized` en `null`; prod oculta el bloque `Diagnostics` y la línea de diagnóstico
del expander, conservando la traza; y `FRONTEND_ENV=PRODUCTION` (errata en mayúsculas) se
comporta como prod, confirmando que el guardarraíl falla hacia ocultar.

Dos incidentes durante la UAT, y solo uno era un bug:

**(a) Falso positivo — mi pregunta estaba mal.** Pedí preguntar «qué requisitos de seguridad debe
especificar el cliente». El fragmento dice *que* el cliente debe especificarlos y en qué debe
basarse, **no cuáles son** — así que el modelo respondió `Not covered by the provided material:` y
el guardrail de scope hizo lo correcto. Sin citas y sin traza es el comportamiento esperado ahí.

**(b) Defecto real, ajeno al split.** Traza del incidente:

```
tokens_in=0  tokens_out=0  ttft_ms=None  latency_ms=21915.6  finish_reason=stop  cost_usd=None
```

El proveedor cerró el stream **sin un solo delta de texto** tras 22 segundos. Y el sistema lo
entregó *como respuesta correcta*: `answer: ""`, `truncated=False`, y **su traza de evidencia
detrás**. En un producto cuyo valor es la cita verificable, eso es evidencia respaldando un texto
que no existe.

**Causa raíz:** los defaults del wrapper de streaming. `finish_reason = "stop"` es una
*afirmación* que nadie verificó —«el modelo terminó con normalidad»— y si el bucle del stream no
itera ni una vez, sale tal cual. Aguas abajo es indistinguible de un éxito.

---

## 5. Las cuatro decisiones que aprobaste, ya implementadas

Sin commitear todavía. Suite **923 verde** y los tres caminos verificados contra el contenedor
real.

**1. `citations` y `retrieved_context` con semánticas separadas.** `citations` se vacía en un
rechazo (cita fantasma evitada); **la evidencia viaja igual**, porque lo recuperado es un hecho
del sistema, independiente de lo que el modelo hiciera con ello. *El argumento que decidió: en
S09, «el retriever trajo estos 5 chunks y el modelo aun así dijo not covered» es la señal
principal para diagnosticar retrieval.* No pintar la traza de un rechazo sigue siendo correcto,
pero pasa a ser **decisión de render** y vive en el frontend. Verificado en vivo: rechazo con
`citations: []` y 1 átomo presente.

**2. Una respuesta vacía no se entrega: se nombra.** Stream sin deltas → `event: error`, no
`done`. `/query` → **502** (petición bien formada, proveedor devolvió algo inservible; ni 500 ni
200 mintiendo). Tampoco deja turno en la sesión: un turno de asistente vacío se re-inyectaría en
la pregunta siguiente.

**3. El wrapper deja de afirmar normalidad.** Default de `finish_reason`: `"stop"` → `"unknown"`.

**4. Numeración y graduación** — resueltas como «lo más recomendable», ver §6 y §7.

---

## 6. Lo que te toca decidir: qué se gradúa a la Guide

Del tramo salieron **13 candidatos agnósticos anotados y ninguno graduado** (graduar es paso de
juicio tuyo, no mío). Mi recomendación es graduar **tres** y dejar el resto anotado, porque estos
tres se ganaron el sitio apareciendo **más de una vez y en capas distintas**:

1. **Un instrumento que se mide consigo mismo no puede detectar su propia omisión.** Apareció en
   el gate de conteo de marcadores, en la conservación de tablas (`f9c8204`) y en cualquier
   chequeo derivado de la misma fuente que verifica. Es el más fuerte del tramo.
2. **Un contrato de API no depende del entorno.** Proyectar es del render. Corolario operativo:
   un contrato que se bifurca por entorno hace que los tests dejen de ser evidencia sobre lo
   desplegado.
3. **El valor por defecto de un campo de estado no debe afirmar el caso bueno.** Si el proveedor
   no lo dijo, el default tiene que decir «no se sabe»: un default optimista convierte un fallo
   en un éxito silencioso. Es el defecto de §4(b) en una frase.

Los otros diez, anotados para más adelante: un hogar por decisión · la conservación se mide contra
la fuente, no contra la salida · un chequeo que sale rojo siempre deja de leerse · cuando dos
grafías no se reconcilian se traduce hacia la lossy · colapsar una abstracción retira deuda pero
no cierra la pregunta que la motivó · validar antes de gastar · un modelo compartido nunca un DTO
gemelo · lo que no se midió se admite, no se rellena · una subclase puede prometer más que su
base, nunca menos · un interruptor que no se puede accionar sin editar código no es un
interruptor.

**Sobre la numeración del off-cycle: resuelta, sin `SNN`.** Había dos carpetas para el mismo
objetivo —`Auditoria pre M4` (ronda 1, 9–10 sep) y una `pre-S09` que abrí yo después— y se han
consolidado en la primera, que es la establecida y la paralela a `Auditoria pre M3`. Todo lo del
off-cycle vive ahí como **ronda 2**, con prefijo `ronda2-`.

**Ojo con las etiquetas, y es la razón del prefijo:** las dos rondas usan series `A/B/C/D`
propias que **chocan**. «C1» es reconciliación de docs en la ronda 1 y reorganización de docs en
la ronda 2 —parecen lo mismo y son dos briefs distintos—; «C2» es fixes de código en una y
colapsar `TokenCounter` en la otra; y «F2» es el seam de embeddings async en la ronda 1 y la
conservación de tablas en la ronda 2. No se reescribieron las etiquetas (son historia commiteada en
el CHANGELOG); el `README.md` de la carpeta mapea cada una a su ronda. **Cuando cites una etiqueta
de este tramo, di la ronda.**

---

## 7. Lo que queda abierto, y lo que S09 activa

**Tres decisiones que el cable de S09 activa de golpe.** No son de implementación: cambian qué ve
el generador y qué cita la respuesta.

1. **Fan-out de citas.** Hoy `_citations` devuelve *todos* los fragmentos como candidatos. Con *k*
   chunks son *k* citas por respuesta, cite el modelo lo que cite. La firma ya admite el filtro;
   el filtro no existe. **Tiene consecuencia de UI:** en cuanto estreche las citas, «lo citado» y
   «lo recuperado» dejarán de coincidir y la traza tendrá que distinguirlos.
2. **Recomponer la cláusula en retrieval.** El chunk es el **requisito**, no la cláusula, así que
   ningún chunk contiene la cláusula entera. Medido: el **67 %** de las cláusulas con requisitos
   tienen 2 o más (máximo 59), el **59 %** contiene una NOTE que mezclaría autoridad, y el **11 %**
   excedería la ventana como chunk único. Si al recuperar un requisito conviene arrastrar el chunk
   de sección de su misma cláusula es decisión de retrieval.
3. **Qué modo alimenta al generador: `vector` o `hybrid`.** La híbrida sin pesos **arregla la
   query con identificadores y degrada la genérica**: en el par trampa criticidad B/C sube el
   chunk correcto del puesto 3 al 1 y saca la tabla D-1 real al 2; en «verification requirements»
   el mejor hit del vector cae del 1 al 4, sobrevive 1 de 5 resultados y la distancia media empeora
   de 0,449 a 0,493. Ponderar la fusión era trabajo de S10, pero si S09 cablea la híbrida sin
   pesos hereda la degradación.

**Un fleco nuevo del fix de hoy:** `truncated` sigue siendo `finish_reason == "length"`, así que
con el default `"unknown"` dice «no truncada» cuando en realidad **no se sabe**. Hacerlo
tri-estado cambia contrato y render, así que queda anotado, no hecho.

**Deuda de doc conocida:** la pasada de estilo sobre `ARCHITECTURE.md` quedó parcial (§3 y §5
siguen densas, 1.511 líneas). El resto de lo abierto —marcadores no citables aguas arriba,
tokenizador real, expansión por grafo, las 189 tablas, los flecos de identidad, las dos fuentes
de «qué debería existir»— sigue como estaba y con sus cifras en `ARCHITECTURE.md` §7.

---

## 8. Estado del árbol ahora mismo

`main` limpio hasta `3d83fe4`. **Sin commitear**, esperando tu ok a este resumen: los cambios de
§5 en `app/services/llm_service.py`, `app/services/llm_wrapper.py`, `app/routers/queries.py`,
`frontend/streamlit_app.py`, `docker-compose.yml`, los dos ficheros de test, y la documentación
(`CLAUDE.md`, `ARCHITECTURE.md`, `CHANGELOG.md` con la entrada `0.43.0`).

El repo de la Guide sigue **sin commitear a propósito**, con los cuatro archivos de
`sessions/Auditoria pre M4/` (ronda 2).
