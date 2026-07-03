# S02-decisions.md — Implementación (Claude Code)

Decisiones y fricciones surgidas al implementar el scaffolding descrito en `S02-brief.md`,
sobre el repo `RAG-ECSS` (rama `feat/s02-cag-scaffolding`). Etiquetadas agnóstico (candidata a
Guide) vs. dominio (solo ECSS).

## Registradas por el brief (§11)

- **[agnóstico]** CAG sobre una rebanada acotada de conocimiento es una forma legítima de
  levantar arquitectura + contrato de salida (schemas, citación, grounding) antes de tener
  retrieval. Condición de aplicabilidad: la rebanada debe caber en contexto y ser estable.
  Deja de servir en cuanto el corpus real (ECSS completo) no cumple esa condición — que es
  exactamente lo que motiva M2 (RAG). No se ha "hecho CAG con intención de quedarse"; es
  puente arquitectónico deliberado.
- **[agnóstico]** Seam fino de proveedor (`_call_model`) como paso previo al wrapper agnóstico
  de S03: una única función de frontera que aísla el SDK, usando el cliente async nativo
  (`AsyncAnthropic`). Se evitó explícitamente el bug de envolver una llamada síncrona en
  `async def` (congelaría el event loop de FastAPI durante toda la llamada al LLM).
- **[agnóstico]** Orden del prompt (`build_system_prompt`): rol → grounding → citación →
  formato → material de referencia, todo en `system` (estático); la pregunta va en `user`
  (variable). Este orden sirve simultáneamente a la atención posicional del modelo y a la
  futura frontera de prompt caching de S03 (prefijo estático cacheable sin reordenar nada).
- **[agnóstico]** Campo `citations` en el schema de respuesta desde v0, aunque en esta versión
  solo transporta las coordenadas de los fragmentos **inyectados** (candidatas de cita), sin
  extraerlas ni verificarlas contra el texto generado por el modelo. La verificación real
  (grounding automático, detectar cláusulas alucinadas) es S11. Comentado explícitamente en
  `app/services/llm_service.py` para que S11 no asuma que esto ya está resuelto.

## Fricción de implementación no prevista por el brief

- **[dominio/técnico] El parser de `corpus/` no captura número de página.** El brief pedía
  fragmentos con coordenadas de cita `standard/clause/page`, con `page` como "página real en
  el PDF fuente" (ejemplo: `47`). Al inspeccionar `data/processed/*.json` no existe ningún
  campo de página en ninguno de los 14 documentos — el parser (`corpus/parser.py`) solo
  captura jerarquía estructural (`section_number`, `section_path`), no paginación, porque
  extrae de DOCX (donde la paginación no es un dato del XML sino un cálculo de layout de
  Word/Adobe en render). Tampoco hay PDFs fuente en `data/raw/` (solo DOCX/DOC), así que ni
  siquiera hay un render de referencia contra el que contar páginas a mano de forma fiable.
  **Decisión consolidada (post-implementación):** en vez de instrumentar el parser para
  aproximar página (contar saltos de página en el DOCX, o alinear contra un PDF oficial
  descargado aparte), se descartó `page` como campo de cita por ser un recurso frágil —
  cualquier aproximación sin un PDF de referencia fiable sería una cifra inventada, justo lo
  que el contrato de citación pretende evitar. `Citation` (schemas/query.py) y
  `ECSS_REFERENCE` (context/reference.py) se actualizaron para no transportar `page` en
  absoluto; la trazabilidad queda en `standard`/`clause`, suficiente para localizar la
  cláusula exacta en el documento fuente. Esto sustituye la decisión pendiente inicial.
- **[dominio]** Fragmento elegido: `ECSS-E-ST-40_0860015` de
  `data/processed/ECSS-E-ST-40C_Rev.1.json`, sección `5.2.4.8`
  ("Software safety and dependability requirements"). Motivo: autocontenido (un único
  requisito con sentido propio, sin continuaciones), y contiene una referencia cruzada
  explícita a otro estándar CORE (`ECSS-Q-ST-80` clauses 5.4.4, 6.2.2, 6.2.3) — cubre el
  requisito de representatividad del brief (§4) sin necesitar un segundo fragmento aparte,
  así que se usó solo uno (`ECSS_REFERENCE` es una lista de un elemento; añadir un segundo
  fragmento habría sido especular sin necesidad para este alcance).
- **[dominio]** Se corrigió `.env` / `.env.example`: el repo traía `LLM_PROVIDER=openai` /
  `LLM_MODEL=gpt-4o-mini` de la sesión anterior (heredado del scaffolding del estimador). Se
  actualizó a `LLM_PROVIDER=anthropic` / `LLM_MODEL=claude-haiku-4-5-20251001` (verificado
  vigente a 2026-07-03 en el brief §9) y se añadió `MAX_OUTPUT_TOKENS=2000`, ausente hasta
  ahora aunque `config.py` ya lo requería con default.

## Verificación manual (test de usuario, §10)

Servidor levantado con `uv run uvicorn app.main:app`, dos llamadas reales:
1. Pregunta cubierta por el fragmento (sobre 5.2.4.8) → responde citando
   `ECSS-E-ST-40C 5.2.4.8` correctamente, sin inventar contenido fuera del fragmento.
2. Pregunta fuera del fragmento (empuje del Ariane 6) → responde
   "La referencia proporcionada no cubre esto" y explica por qué, sin fabricar una respuesta.

Sin fricción: el comportamiento de grounding funcionó de primeras con el system prompt tal
como está redactado, no hizo falta iterar el prompt.

## No implementado (fuera de alcance, §1) — nada que reportar

No se detectó ninguna necesidad de adelantar trabajo de sesiones futuras (retrieval, wrapper
multi-proveedor, prompt caching activado, verificación de citas, multi-turn) para completar
el alcance de S02.
