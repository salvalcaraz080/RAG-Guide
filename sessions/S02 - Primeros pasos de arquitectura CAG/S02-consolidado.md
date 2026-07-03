# S02-consolidado.md — Registro histórico de la sesión 02

> Documento **histórico, no se reescribe**. Captura lo que la teoría del máster planteó, lo que la
> implementación reveló, y — sobre todo — **dónde divergieron**. El resumen (`S02-resumen.md`) es
> pre-implementación; la Guide destila solo lo graduado. Este documento es el único que conserva
> el *cómo se llegó*, incluida la fricción.

Sesión 02 — "Primeros pasos de arquitectura CAG". Módulo 2. Rama de implementación:
`feat/s02-cag-scaffolding` sobre RAG-ECSS.

---

## 1. Qué cubría la sesión (teoría)

Cuatro artículos: fundamentos de CAG (vs. RAG), estructura FastAPI escalable, gestión de contexto
en CAG, y arquitectura de conversaciones (roles system/user/assistant, single/multi-turn, gestión
de historial). El detalle destilado vive en `S02-resumen.md`; aquí solo lo necesario para situar
la divergencia.

Tesis central del máster en S02: CAG (precargar todo el conocimiento en el prompt, sin retrieval)
es el punto de partida correcto cuando los datos de referencia son acotados y estables; se levanta
un servicio FastAPI por capas y se itera sobre calidad de prompt y contexto antes de migrar a RAG.

---

## 2. Divergencias teoría-vs-implementación

El núcleo del documento. Cada entrada: qué decía el material, qué reveló implementarlo (o
verificarlo), y qué se hizo.

### 2.1 El corpus real rompe la premisa de CAG — CAG como andamio, no destino

- **Teoría (máster):** desarrolla CAG sobre el proyecto vehículo del máster (estimador de
  presupuestos), cuyo conocimiento cabe en contexto → CAG válido como arquitectura. Presenta CAG y
  RAG como fases de madurez del *mismo* sistema.
- **Implementación (ECSS):** el corpus ECSS **no cumple** la condición de aplicabilidad de CAG
  (grande, jerárquico, con trazabilidad por cláusula). Un port 1:1 del ejercicio era imposible por
  diseño: el corpus no cabe. La resolución no fue abandonar CAG, sino usarlo **sobre una rebanada
  acotada** de un estándar como *andamio arquitectónico*: levanta el contrato de salida y el corte
  vertical completo (schema, grounding, citación, seam de proveedor) con las mínimas piezas
  móviles, sabiendo de antemano su fecha de caducidad.
- **Aprendizaje graduado:** CAG-como-andamio con condición de aplicabilidad explícita y señal de
  salida conocida. → Guide, Fase 3 (decisión marquesina).

### 2.2 El snippet del máster tenía un bug de concurrencia

- **Teoría (máster):** el artículo de arquitectura FastAPI argumenta correctamente *por qué* async
  (llamadas al LLM largas e I/O-bound; el event loop libera el hilo mientras espera). Pero el
  snippet de servicio que ofrece usa el **cliente síncrono** de OpenAI (`client.chat.completions
  .create(...)`) dentro de una función `async def`, sin `await` real sobre I/O asíncrono.
- **Implementación:** eso **contradice la propia tesis del artículo** — una llamada síncrona
  bloqueante dentro de `async def` congela el event loop durante los 2–30 s de la llamada,
  reproduciendo exactamente el problema que el artículo dice resolver. Se implementó con el cliente
  async nativo (`AsyncAnthropic`) tras el seam.
- **Aprendizaje graduado:** async correctness como best-practice con modo de fallo concreto; los
  snippets de tutorial/curso frecuentemente lo hacen mal. → Guide, Fase 1.

### 2.3 Coste de CAG presentado sin su mitigación natural (prompt caching)

- **Teoría (máster):** los cuatro artículos tratan el coste de CAG como proporcional al volumen de
  tokens enviados en cada llamada (ejemplo recurrente: 10K tokens de contexto × 1.000 llamadas/día
  = 10M tokens/día). **No mencionan prompt caching** en ninguno de los cuatro.
- **Verificación independiente:** el patrón de CAG (prefijo estático grande + parte variable
  pequeña, repetido en cada llamada) es el caso de uso de manual para prompt caching. Con caching
  de Anthropic (`cache_control`), el coste efectivo de ese bloque repetido cae a ~10-15% del
  original (lectura a 0.10x tras una escritura a 1.25-2x). OpenAI: caching automático, 50%.
- **Decisión de alcance:** no se implementó caching en S02 (es S03, tema explícito del wrapper).
  Pero el **orden del prompt** se estructuró para dejarle sitio (todo lo estático antes de lo
  variable) — que coincide con el orden óptimo por atención posicional. La coincidencia de ambos
  criterios (atención + frontera de prefijo cacheable) apuntando al mismo diseño es el aprendizaje.
- **Aprendizaje graduado:** el *orden de prompt de doble propósito* (no el caching en sí) → Guide,
  Fase 3. Prompt caching como técnica queda pendiente de graduar en S03.

### 2.4 La cita por página choca con el formato de origen — anclas estructurales > posicionales

- **Teoría (máster / brief):** el contrato de trazabilidad se formuló como documento / sección /
  cláusula / **página**, con `page` como "página real en el PDF fuente".
- **Implementación (fricción no prevista):** al ir a poblar el fragmento de referencia, se descubrió
  que **el parser de `corpus/` no captura página, y no puede**: extrae de DOCX, donde la paginación
  no es un dato del XML sino un cálculo de layout en render; tampoco hay PDFs fuente en `data/raw/`.
  `data/processed/*.json` no tiene campo de página en ninguno de los 14 documentos.
- **Decisión (Claude Chat):** **no instrumentar el parser.** Contar saltos de página en DOCX da
  precisión falsa; alinear contra PDF oficial es trabajo de pipeline (S06), prematuro; y tocar
  `corpus/` reabre tooling congelado por ganancia marginal. Se reencuadró la limitación como
  principio: para un corpus normativo, el **ID de cláusula** (`ECSS-E-ST-40C §5.2.4.8`) es un ancla
  de cita *superior* al número de página — semántico, verificable, independiente de la edición. Un
  número de página es específico de la impresión. `Citation.page` queda `int | None` y absorbe el
  `None` sin bloquear. Revisitar en S06 solo si aparece necesidad real de página, alineando contra
  PDFs oficiales.
- **Aprendizaje graduado:** anclas estructurales > anclas posicionales; verificar que el formato de
  origen contiene la coordenada que el contrato de salida promete. → Guide, Fase 2.

### 2.5 Emitir citas ≠ verificar citas (frontera de alcance que el máster no marca)

- **Teoría (máster):** trata la citación como una funcionalidad, sin distinguir niveles.
- **Implementación:** se implementó el campo `citations` en el schema desde v0, pero poblado solo
  con las coordenadas de los fragmentos **inyectados** (candidatas), sin extraerlas ni verificarlas
  contra el texto generado. Se comentó explícitamente en `llm_service.py` que la verificación real
  (grounding automático, detectar cláusulas alucinadas) es S11, para que esa sesión no asuma que ya
  está resuelto.
- **Aprendizaje graduado:** distinción *emitir vs. verificar* citas, y la práctica de fijar el
  contrato de salida (con trazabilidad) desde v0 aunque la verificación llegue después. → Guide,
  Fase 2. Es el gancho explícito donde S11 engancha.

---

## 3. Lo que funcionó sin fricción (confirma la teoría)

- **Grounding a la primera.** El system prompt con instrucción de grounding ("responde solo desde
  el material; si no está, dilo; no inventes") funcionó sin iterar: la pregunta cubierta por el
  fragmento se respondió citando correctamente, y la pregunta fuera del fragmento (empuje del
  Ariane 6) devolvió "la referencia proporcionada no cubre esto" sin fabricar. La teoría del máster
  sobre calidad de system prompt se sostuvo en la práctica.
- **Separación por capas y seam de proveedor.** La estructura routers/services/schemas/context se
  implementó sin fricción; el seam fino (`_call_model`) aísla el SDK en una función, dando a S03 un
  punto de inserción limpio sin haber construido el wrapper.
- **Modelo verificado vigente.** `claude-haiku-4-5` (`claude-haiku-4-5-20251001`) confirmado como
  económico de generación actual a 2026-07-03. Nota: el `gpt-4o-mini` del ejercicio del máster está
  **stale** (legacy); el económico OpenAI actual sería GPT-5.4 mini. Se defaulteó a Anthropic haiku.

---

## 4. Decisiones de dominio (solo ECSS, no graduan a Guide)

- Fragmento de referencia elegido: `ECSS-E-ST-40C §5.2.4.8` ("Software safety and dependability
  requirements") de `data/processed/ECSS-E-ST-40C_Rev.1.json`. Motivo: autocontenido (un requisito
  con sentido propio) y con referencia cruzada explícita a `ECSS-Q-ST-80` (cláusulas 5.4.4, 6.2.2,
  6.2.3) — cubre representatividad sin necesitar un segundo fragmento.
- `.env` corregido: heredaba `openai`/`gpt-4o-mini` del scaffolding del estimador; actualizado a
  `anthropic`/`claude-haiku-4-5-20251001` + `MAX_OUTPUT_TOKENS=2000`.
- `pre-rebuild` taggeado antes de reconstruir `app/`; `corpus/`, `data/processed/`, `tests/corpus/`
  preservados.

---

## 5. Estado al cerrar S02

- **Implementado y validado:** servicio FastAPI nativo-ECSS con `POST /api/v1/query` (single-turn),
  CAG sobre rebanada acotada, seam de proveedor async-correcto, contrato de salida con citación
  mínima, grounding funcional. `/health`, Swagger, config fail-fast, tests (health, prompt sin red,
  query mockeado, grounding).
- **Graduado a la Guide:** Fases 0–3 (framing, esqueleto de servicio, contrato de salida primero,
  rebanada CAG).
- **Pendiente / abierto para sesiones futuras:** wrapper multi-proveedor y prompt caching activado
  (S03); verificación de citas / RAGAS (S11); multi-turn para ECSS (a evaluar); decisión de página
  revisitable en S06. Cableado de `corpus/` al pipeline (S06/S07).
