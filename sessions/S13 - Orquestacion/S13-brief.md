# S13 — Brief 1: desglose de latencia por etapa

> Primer brief de S13. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar. Rama:
> `s13-orquestacion`, creada desde `main` **después** de que Salva mergee `s12-agentes`. Los commits
> los hace Salva. **Sin llamadas reales**: la medición la hace la pasada del brief 2.
>
> **Regla del brief:** es **instrumento**, no técnica. No cambia el retrieval, el prompt, el modelo,
> el esquema de salida, el contrato público (`QueryResponse`) ni ningún default. Si algo de eso
> pareciera necesario, `TODO: pendiente de decisión en Claude Chat` y se reporta.

---

## 0. Contexto y objetivo

**De dónde sale.** La referencia `xref+bounded` (`generation_20261019-224307`) tarda 16,8 s de
mediana (p95 24,0 s). La latencia es **objetivo**, no guarda (decisión de producto en S12: la
calidad va antes), y se ataca por otras vías: UI de progreso, razonamiento `low`, caché, paralelizar.
Ninguna de esas palancas se puede elegir sin saber **dónde se van los 16,8 s**. Hoy hay piezas
sueltas (`embedding_ms`, `db_ms`, `latency_ms` de la llamada al modelo, `latency_ms` de la petición),
pero no un desglose que sume la petición entera ni un agregado en el arnés.

**Lo que decidió el chat** (`S13-resumen.md` §7 y §9):
- No se adopta un framework de orquestación ni un árbol de spans (OpenTelemetry y backend de trazas
  son de S15). Basta con **structlog + `request_id`**, que ya existe.
- Medir antes de paralelizar: si la generación domina, paralelizar retrieval o `xref` ahorra
  milisegundos sobre 17 s y no se hace.

**Objetivo.** Cada petición registra la duración de cada etapa del pipeline y el arnés agrega el
desglose por etapa (mediana, p95 y fracción del total), de modo que la pasada del brief 2 lo produzca
sin coste adicional.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| L1 | Un único mecanismo de cronometraje (p. ej. un context manager sobre `time.perf_counter`) que acumula duraciones por nombre de etapa en un objeto por petición | Un sitio, no `perf_counter` repartidos. Las etapas no deciden nada: solo se cronometran (separación trabajo/control) |
| L2 | Las duraciones viajan en **un** evento de log por petición (`query_completed` u otro dedicado, a tu criterio, regístralo) con el `request_id`, y en el artefacto del arnés. **No** entran en `QueryResponse` | La traza no es parte del contrato público (S12). `latency_ms` total ya está en el contrato y no cambia |
| L3 | Las etapas suman la petición: se registra también `unaccounted_ms` = total − suma de etapas | Si el desglose no cuadra, se ve. Un desglose que no suma no sirve para decidir |
| L4 | La llamada al modelo se desglosa por **intento** (reintentos de esquema, fallback) y no solo en total | Un reintento de esquema puede duplicar la etapa dominante; tiene que distinguirse de un modelo lento |
| L5 | Cero cambio de comportamiento: array de mensajes y contexto **idénticos byte a byte** con y sin instrumentación | Mismo patrón que pre-S12. Un instrumento que mueve la referencia invalida la medida |

---

## 2. Trabajo

### 2.1 Etapas

Lista de partida (ajústala a cómo está partido el código hoy y **registra la lista final antes del
brief 2**; no partas ni fusiones funciones para que encajen):

| Etapa | Qué cubre |
|---|---|
| `guardrails` | Guardrails de entrada |
| `history` | Carga del historial y composición de la consulta de retrieval |
| `embedding` | Embedding de la consulta (ya existe como `embedding_ms`) |
| `db` | Búsqueda vectorial (ya existe como `db_ms`) |
| `expansion` | Expansión de cláusula (`bounded`) |
| `xref` | Parser, resolución y `get_clause` de la expansión por referencia |
| `prompt` | Construcción del prompt y presupuesto de entrada |
| `llm` | Llamada(s) al modelo, por intento (L4) |
| `validation` | Validación del esquema y verificación de citas |
| `compose` | Composición de la respuesta servida |

Si una etapa del código no encaja en ninguna, añádela. Si alguna ya se cronometra (`embedding_ms`,
`db_ms`), reutiliza el valor; no lo midas dos veces.

### 2.2 Arnés

- Cada ejecución guarda el desglose en su artefacto.
- Nuevo informe (o sección del que ya hay; regístralo): por configuración, **mediana, p95 y fracción
  del total** de cada etapa, más `unaccounted_ms`. Re-puntuable sobre artefactos, sin modelo ni base.
- Los artefactos anteriores no tienen desglose: el informe los trata como "sin desglose", no falla.

### 2.3 Docs

`ARCHITECTURE.md` (observabilidad: el evento, las etapas, que no está en el contrato) y
`.claude/CLAUDE.md` si aplica. `CHANGELOG.md` derivado.

---

## 3. Tests de contrato (sin red)

- `uv run pytest` en verde. Recuento frente a los **1419** de partida.
- **L5:** equivalencia byte a byte del array de mensajes y del contexto con la instrumentación.
- El evento lleva todas las etapas de §2.1 (las que apliquen a la petición) y `unaccounted_ms`.
- Reintento de esquema simulado → dos intentos en `llm` (L4).
- Una petición que falla en retrieval registra las etapas que llegó a ejecutar, sin romper el
  evento `retrieval_failed` existente.
- `QueryResponse` no cambia (test del esquema público).
- Informe del arnés sobre artefactos sintéticos: con desglose, sin desglose y mezclados.

---

## 4. UAT (usuario)

Propón el script. Mínimo: dos consultas en la UI (una sin referencias y la de q09) y comprobar en los
logs del contenedor que el evento sale con el `request_id`, que las etapas cuadran con el total y qué
etapa domina. Es una comprobación del instrumento, no la medición: la medición es la pasada del
brief 2.

---

## 5. Cierre

- `S13-decisions.md` en `RAG-Guide/sessions/S13 - Orquestacion/`: lista final de etapas, dónde se
  emite el evento, cómo se trata el fallback y el reintento de esquema, recuento de tests,
  etiquetado agnóstico/dominio.
- `S13-uat.md` con el script de UAT.
