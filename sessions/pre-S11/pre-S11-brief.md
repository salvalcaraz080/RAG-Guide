# pre-S11 — Brief: retirar el streaming

> Brief de infraestructura previo a S11 (no es contenido de S11). Lee `.claude/CLAUDE.md` y
> `ARCHITECTURE.md` antes de empezar. Rama: `s11-generacion-calidad` desde `main`; este brief es su
> primer bloque de trabajo. Los commits los hace Salva.
> **Regla del brief:** quitar, no rediseñar. Toda decisión que el streaming condicionó se revisa, y
> solo se cambia si el cambio **simplifica**. Lo que exija diseño nuevo no se toca aquí:
> `# TODO: pendiente de decisión en Claude Chat`.

---

## 0. Objetivo y alcance

**Objetivo.** Un solo camino de respuesta: `POST /api/v1/query`, respuesta completa. Motivo (decidido
en S11, paso 2): la política ante citas no verificadas que llega en `S11-brief-2` necesita poder
rechazar o reintentar **antes de servir**, y con streaming el texto ya está en pantalla cuando la
validación termina. Para un producto cuyo valor es la trazabilidad, verificar después de servir no
vale. El streaming se diseñó como capa de entrega borrable; este brief la borra.

**Dentro de alcance:** eliminar `/query/stream` y todo lo que solo existe para él (backend, wrapper,
schemas, frontend, tests, dependencias, docs); migrar al camino no-stream cualquier invariante que
hoy solo esté testeado por el stream; nueva referencia de **latencia total** que sustituye al TTFT.

**Fuera de alcance:**
- **Cambiar el formato de salida** (prosa con `[n]` → structured output). El streaming era la razón
  declarada de la prosa, pero pasar a structured output es diseño nuevo, no simplificación, y es
  precisamente el contrato tipado de `S11-brief-2`. Aquí la salida sigue siendo prosa con `[n]`.
- **La frase marcador de rechazo** (`Out of scope:` / `I could not find…`). Funciona; su debilidad
  (clasificación por prefijo, "puente, no solución definitiva") se resuelve con el contrato tipado →
  `S11-brief-2`.
- Política de citas, `misattributed`, golden set de generación, `corpus/`.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| P1 | Se elimina `POST /api/v1/query/stream` y `llm_service.stream_answer_query` | Un solo camino; habilita verificar antes de servir |
| P2 | Se elimina el código, no se deja latente tras un flag | Código muerto que hay que mantener y testear; el historial de git lo conserva |
| P3 | `llm_wrapper.stream_llm`, `StreamChunk`, `StreamDoneEvent`, `StreamErrorEvent`, `ttft_ms` y la validación SSE en el borde desaparecen | Solo existen para el stream |
| P4 | Los invariantes que vivían duplicados ("su gemelo en el camino de streaming") quedan con una sola implementación en `/query` | Guardrails antes de cualquier efecto, sesión desconocida → 404, respuesta vacía → 502 sin turno en sesión, fallo de proveedor → error tipado: todos siguen vigentes, ahora en un sitio |
| P5 | Frontend: `sse_client.py` y `sseclient-py` fuera; Streamlit llama a `/query` con un indicador de espera (`st.spinner` o similar, una sola frase) | Cliente HTTP fino; no se inventan fases que el servidor no comunica |
| P6 | Latencia total como métrica de referencia (servidor: `latency_ms` en log y respuesta, si no existe ya; cliente: la que ya mide el sidebar) | Sin stream, lo que el usuario espera es la respuesta entera; el TTFT de S10 (mediana 1,76 s) deja de ser la referencia |
| P7 | Timeout del cliente explícito y documentado, holgado frente a la latencia total medida | Sin stream no llegan bytes hasta el final; un timeout por defecto corto rompería respuestas largas |

---

## 2. Inventario: decisiones condicionadas por el streaming

Revisar cada una y registrar en `pre-S11-decisions.md` qué se hizo y por qué. Lista de partida
(sacada de `ARCHITECTURE.md`; si encuentras más, añádelas):

| Decisión actual | Origen streaming | Acción esperada |
|---|---|---|
| Prosa, no structured output | "el structured output rompe el streaming token a token" | **No tocar** (fuera de alcance). Reescribir el porqué en docs: la restricción desaparece; la decisión queda pendiente de `S11-brief-2` |
| Frase marcador natural de rechazo | "no hay que stripearla… habría exigido bufferizar el arranque del stream" | **No tocar.** Actualizar el porqué en docs (el motivo streaming ya no aplica; sigue siendo puente hasta el contrato tipado) |
| Marcadores leídos en `_finish` sobre texto completo ("un `[1]` puede llegar partido") | Defensa frente a deltas | Se mantiene (sigue siendo correcto); quitar la justificación de deltas |
| Respuesta vacía → `event: error` en stream / 502 en `/query` | Doble camino | Solo 502 |
| `stream_options={"include_usage": True}` y detección de truncado en stream | Wrapper stream | Fuera; la detección de truncado vía `finish_reason` en no-stream se mantiene |
| `ttft_ms` en observabilidad y sidebar | Métrica de stream | Fuera; sustituida por latencia total (P6) |
| Limitación `EventSourceResponse` (modo SSE por endpoint) | SSE | Desaparece con el endpoint |
| Router de Streamlit: consumo SSE, repintado | SSE | Llamada `requests.post` a `/query` |

Si alguna otra decisión del repo existe **solo** por el streaming y quitarla simplifica, quítala y
regístrala. Si quitarla exige diseño, `TODO` y regístrala como pendiente.

---

## 3. Trabajo

1. **Auditoría de cobertura antes de borrar.** Recorre `tests/app/test_streaming.py` y clasifica cada
   test: (a) comportamiento solo de stream → se borra; (b) invariante compartido que **solo** está
   probado por el stream → se migra a `test_queries.py` (u otro) contra `/query` **antes** de borrar.
   Lista la clasificación en `pre-S11-decisions.md`. Ningún invariante pierde su test.
2. Backend: router, `llm_service`, `llm_wrapper`, schemas, observabilidad (P1–P4, P6).
3. Frontend (P5, P7).
4. Dependencias: `sseclient-py` y cualquier otra que solo usara el stream (grupo `frontend` y
   principal). `uv lock` actualizado.
5. Docs: `ARCHITECTURE.md` y `.claude/CLAUDE.md` describen el estado actual (un camino, sin
   narrativa histórica); los porqués del §2 reescritos. `CHANGELOG.md` derivado de
   `pre-S11-decisions.md`. Los decisions históricos (`S03-decisions-3-streaming-sse.md`) no se tocan.
6. **Latencia de referencia** (`scripts/`): mismo protocolo que el TTFT de S10 — 14 consultas del
   golden set, defaults vigentes, último turno en multi-turn — midiendo latencia total (mediana y
   p95) a `scripts/output_latency_baseline.txt`. Son ~14 llamadas reales (≈0,10 USD, como en S10):
   autorizadas; si prefieres no ejecutarlas, deja el script listo y lo corre Salva.

---

## 4. Tests de contrato (sin red, sin BD, sin modelo real)

- `uv run pytest` en verde; registrar el recuento final frente a los 1179 de partida, con el
  desglose: borrados (solo stream), migrados, nuevos.
- Nuevos o confirmados en `/query`: guardrail rechaza antes de retrieval y de leer sesión; sesión
  desconocida → 404; respuesta vacía → 502 y sin turno guardado; error de proveedor → error tipado;
  `latency_ms` presente.
- Test de ausencia: la ruta `/api/v1/query/stream` responde 404/405 (el endpoint no existe).
- Ningún import de `sseclient` ni de `fastapi.sse` queda en el repo.

---

## 5. UAT (usuario)

Propón un script de UAT para la UI de Streamlit (Docker lo levanta Salva). Mínimo:
1. Pregunta puntual (p. ej. criticidad SW en ECSS-Q-ST-80): respuesta completa con citas `[n]`,
   indicador de espera visible mientras tanto, latencia en el sidebar.
2. Pregunta fuera de dominio: rechazo con su frase marcador; `refusal` correcto en el sidebar.
3. Seguimiento corto con anáfora (multi-turn): sigue funcionando sin stream.
4. Pregunta larga/enumerativa: no corta por timeout.
5. Evidencia ("retrieved but not cited" y ➕) sigue visible igual que antes (el renombrado es de
   `S11-brief-2`, aquí no se toca).

---

## 6. Cierre

- `pre-S11-decisions.md` en `RAG-Guide/sessions/pre-S11/`: auditoría de cobertura (§3.1), inventario
  del §2 resuelto fila a fila, recuento de tests, latencia de referencia (o script pendiente),
  etiquetado agnóstico/dominio.
- `pre-S11-uat.md` con el script de UAT.
- `CHANGELOG.md`, `ARCHITECTURE.md`, `.claude/CLAUDE.md` actualizados.
