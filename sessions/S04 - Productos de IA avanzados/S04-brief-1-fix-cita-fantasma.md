# S04 — Brief 1-fix: cortar la cita fantasma en rechazos

> **Para:** Claude Code. **Sesión:** S04. **Tipo:** dominio (RAG-ECSS).
> **Es un fix del brief 1** (`S04-brief-1-robustez-prompt`), no una funcionalidad nueva.
> Si encuentras una decisión no cubierta aquí: `# TODO: pendiente de decisión en Claude Chat`.
> `.claude/CLAUDE.md` manda si hay contradicción.

---

## 1. Problema que corrige

Tras el brief 1, el sistema **adjunta una cita fantasma a cada rechazo**. `_citations()` deriva la
cita candidata de los metadatos del fragmento inyectado **siempre**, respondiera o rechazara el
modelo (diseño desde S03: las citas no se extraen del texto generado). Ahora que el modelo rechaza
("out of domain" / "out of material"), un rechazo sale con `citations: [ECSS-E-ST-40C 5.2.4.8]`
pegada: preguntas algo no-ECSS, el modelo dice "fuera de scope", y el sistema le cuelga una cita
ECSS.

**Por qué es grave y no diferible:** en M1, con un solo fragmento, los rechazos son el caso
**común** en una demo, no el raro — casi cualquier pregunta que no toque ese fragmento cae en "out
of material". Para un artefacto cuya carta de presentación es la trazabilidad ante ingenieros
ECSS, una cita colgada de una negativa es exactamente la cita fantasma que el sistema entero existe
para evitar. Irónicamente, cuanto mejor rechaza el guardrail del brief 1, más citas fantasma
produce.

**Qué NO es este fix:** no es verificación de grounding (¿la respuesta cita lo que realmente usó?
¿alucinó una cláusula al *responder*?). Eso sigue siendo S11. Este fix resuelve solo el caso
binario y frecuente: **si el modelo rechazó, no hay cita.**

---

## 2. Mecanismo (frase de rechazo fija + detección en el ensamblado)

El rechazo empieza con una **frase natural fija y legible** (mismo patrón que el estimator del
máster, que marca out-of-scope con `summary` empezando por `"Out of scope:"`). El ensamblado
detecta el prefijo y devuelve `citations = []`.

**Por qué una frase natural y no un marcador técnico** (p. ej. `OUT_OF_MATERIAL:`): un marcador
técnico habría que strippearlo del texto, y en `/query/stream` se streamearía token a token al
cliente antes de poder limpiarlo → obligaría a bufferizar el arranque del stream, que es
lógica-solo-en-streaming (prohibida por S03). Una frase natural **es** parte del mensaje legible:
no se strippea, se streamea presentable, y la detección es un `startswith` en el punto compartido.

---

## 3. Decisiones ya tomadas (no re-decidas)

1. **Dos frases, una por tipo de rechazo** (mantiene la distinción del brief 1):
   - Out of domain → la respuesta empieza **exactamente** con `Out of scope:`
   - Out of material → empieza **exactamente** con `Not covered by the provided material:`
2. **La frase se conserva en el `answer`** (es parte del mensaje legible; el usuario ve
   `"Out of scope: This question is not about ECSS standards."`). No se strippea.
3. **La detección vive en el punto de ensamblado compartido** (`_citations()` / `_assemble_response`
   en `llm_service.py`), que ya usan **ambos** paths (`answer_query` y `stream_answer_query`, ver
   `ARCHITECTURE.md` §3). El fix vive en **un solo sitio** y cubre los dos endpoints sin tocar el
   mecanismo de streaming.
4. **Sin campo nuevo en `QueryResponse`/`StreamDoneEvent`.** El fix es cortar la cita, no señalar
   el rechazo programáticamente con un campo tipado — eso sigue anclado (brief 1 §7, candidato a
   structured output / S11). En M1, `citations == []` es de facto la señal de rechazo (una
   respuesta normal siempre lleva la cita candidata).
5. **Es un puente, no la solución definitiva.** Detección por prefijo de frase = frágil si el
   modelo no emite la frase exacta. Se mitiga con instrucción explícita en el prompt y se jubila
   cuando structured output (S11) dé un campo tipado. Documentar el límite (§7).

---

## 4. Implementación

### 4.1 Ajuste de `app/prompts/query/system.md`
En la sección de scope del brief 1, instruir **explícitamente** que:
- un rechazo *out of domain* **debe empezar con el texto exacto** `Out of scope:` seguido del
  mensaje;
- un rechazo *out of material* **debe empezar con el texto exacto**
  `Not covered by the provided material:` seguido del mensaje.

Redacción sugerida (intégrala con el estilo de delimitación vigente, sin cambiarlo):

> When you refuse, begin your response with an exact marker phrase so the system can recognise it:
> - Out of domain → start with exactly: `Out of scope:`
> - Out of material → start with exactly: `Not covered by the provided material:`
>
> Use these exact phrases verbatim at the very start of the response. Then continue with a brief,
> readable explanation. Do not add any text before the marker phrase.

### 4.2 Detección compartida
- Añade una helper, p. ej. `_refusal_kind(text: str) -> str | None`, que devuelve
  `"out_of_domain"`, `"out_of_material"`, o `None`, según el prefijo exacto del texto (comparación
  `startswith`, **case-sensitive**).
- Comentarios inline en español (qué hace `startswith`, por qué case-sensitive, por qué vive aquí
  y no en el path de streaming). Type hints + docstring Google en inglés.

### 4.3 Cortar la cita
- En el ensamblado (`_citations()` o donde se construye el resultado final), si `_refusal_kind(...)`
  no es `None` → `citations = []`. Si es `None` → la cita candidata como hoy.
- Debe aplicar **igual** en `answer_query` y en el cierre (`event: done`) de `stream_answer_query`,
  por vivir en el punto compartido. Verifica que **ambos** paths devuelven `citations: []` en un
  rechazo.

### 4.4 Consecuencia de caché (verificar, no implementar)
Cambiar `system.md` (añadir las frases de rechazo) vuelve a cambiar el prompt final → el snapshot
de caracterización del brief 1 **se actualiza otra vez** y las entradas cacheadas con el prompt
previo quedan huérfanas y expiran por `CACHE_TTL`. Es la invalidación por versión de prompt
funcionando; nada especial que hacer, solo anotarlo.

---

## 5. Tests

Levanta Docker y prueba la imagen ANTES de la suite. No rompas la fixture `autouse` de
`tests/app/conftest.py`.

### 5.1 Estructural (sin API)
- El prompt renderizado contiene las dos frases marcador exactas.
- `_refusal_kind` clasifica correctamente: texto que empieza con `Out of scope:` → `out_of_domain`;
  con `Not covered by the provided material:` → `out_of_material`; cualquier otro → `None`.
- **Sin API:** llama a `_assemble_response`/`_citations` con un texto de rechazo simulado y un texto
  de respuesta normal simulado, y verifica `citations == []` en el primero y la cita candidata en
  el segundo. Esto testea el corte sin pegar al modelo.

### 5.2 `real_llm` (marcados, fuera de la suite por defecto)
Actualiza los tests de rechazo del brief 1 para **aseverar sobre `citations`, no sobre el texto**
(el fix lo hace posible y es más limpio que escarbar el texto buscando cláusulas inventadas):
- Out of domain (Anthropic y OpenAI) → `answer` empieza con `Out of scope:` **y** `citations == []`.
- Out of material (Anthropic y OpenAI) → `answer` empieza con `Not covered by the provided
  material:` **y** `citations == []`.
- Respondible por el fragmento → `citations` **no** vacío (la cita candidata sigue saliendo).

Si esto permite que los tests `real_llm` llamen al pipeline completo (`answer_query`) en vez de a
`call_llm` directamente (limitación anotada en `S04-decisions-1` §8), mejor — hazlo así.

### 5.3 Regresión de la cita fantasma
Un test explícito que documente el bug corregido: una pregunta de rechazo (simulada, sin API)
produce `citations == []`. Es el test que habría cazado el problema.

---

## 6. Tests de usuario (contra Docker)
- Pregunta fuera de dominio → `answer` legible con `Out of scope:` al inicio, **sin** citas.
- Pregunta ECSS fuera del fragmento → `answer` con `Not covered by the provided material:`, **sin**
  citas.
- Pregunta respondible → respuesta con su cita candidata (sin regresión).
- Repetir contra `/query/stream` (Streamlit): confirmar que el rechazo se muestra legible y sin
  cita en el sidebar.

---

## 7. Docs a actualizar
- **`S04-decisions-1-...md`** (mismo log o uno nuevo `-fix`): registrar el bug de cita fantasma en
  rechazos, por qué era frecuente en M1, el mecanismo de frase fija (y por qué frase natural y no
  marcador técnico — el problema de streaming), y el **límite conocido**: si el modelo no emite la
  frase exacta, la cita no se corta; se jubila con structured output (S11).
- **`ARCHITECTURE.md`** §3 (contrato de citación): las citas candidatas se suprimen en rechazos
  detectados por prefijo; puente hasta S11.
- **`CLAUDE.md`**: nota breve en el estado de M1.
- **`CHANGELOG.md`**: entrada `fix` (corrige código ya del brief 1 de esta sesión).

**Anclaje que deja puesto:** el campo tipado para señalar rechazo programáticamente (no por prefijo
de texto) sigue pendiente — `# TODO: pendiente de decisión en Claude Chat`, candidato a resolverse
con structured output (S11) o al diseñar el endpoint (brief 2).

---

## 8. Criterios de cierre
- [ ] `system.md` instruye las dos frases marcador exactas.
- [ ] `_refusal_kind` (o equivalente) clasifica por prefijo, case-sensitive, en el punto compartido.
- [ ] Rechazo → `citations == []` en **ambos** paths (`/query` y `/query/stream`).
- [ ] Respuesta normal → cita candidata sin regresión.
- [ ] Snapshot de caracterización actualizado; test de regresión de cita fantasma verde.
- [ ] Tests `real_llm` actualizados a aseverar sobre `citations`, verdes contra Anthropic y OpenAI.
- [ ] Docker levanta; `/health` responde; `uv run pytest` verde.
- [ ] Sin campo nuevo de contrato, sin buffering en streaming, sin tocar el mecanismo SSE.
- [ ] Docs actualizadas; commit lo hace el usuario.
