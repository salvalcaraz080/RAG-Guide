# S04 — Brief 2/2: exponer prompt activo + contexto CAG (cierre del anclaje S03)

> **Para:** Claude Code. **Sesión:** S04. **Tipo:** dominio (RAG-ECSS).
> Si encuentras una decisión no cubierta: `# TODO: pendiente de decisión en Claude Chat`.
> `.claude/CLAUDE.md` manda si hay contradicción.

---

## 1. Objetivo

Cerrar el anclaje que S03 dejó puesto: el sidebar de diagnóstico de Streamlit (S03 brief 5) tiene
dos `# TODO` — *system prompt activo* y *contexto CAG activo* — porque el backend no los exponía.
Este brief los expone, **cada uno según su naturaleza**:

- **Instrucciones activas** (rol, grounding, citación, formato, scope) → **`GET` dedicado**. Son
  estáticas en M1 *y* en M2 (el contrato con el modelo no depende de la query) → un GET global es
  estable, no se tira en M2.
- **Contexto CAG inyectado** → **campo opt-in en la respuesta de `/query`** (y `/query/stream`).
  Es estático en M1 (fragmento fijo) pero **dinámico por-query en M2** (retrieval). Un GET global
  no encajaría en M2 (no hay *un* contexto global, hay "lo recuperado para *esta* pregunta"); que
  viaje con la respuesta es estable M1→M2.

**Por qué separado y no un blob:** el sidebar los conceptualizó como dos cosas, y la anatomía del
brief 1 ya los separa (instrucciones = contrato en archivo; contexto = dato en `reference.py`).
Exponerlos por vías distintas respeta que tienen ciclos de vida distintos.

---

## 2. Decisiones ya tomadas (no re-decidas)

1. **Instrucciones por `GET`; contexto por campo en la respuesta de `/query`.** (Decisión de
   arquitectura tomada en Claude Chat.)
2. **Contexto opt-in bajo flag, no en cada respuesta.** El contexto es *contenido*; en M2 serán
   *k* chunks. Meterlo en toda respuesta de producción infla el payload sin que el consumidor
   normal lo pida. Un flag en el request lo activa solo para diagnóstico. Coherente con "log
   metadata, not content, by default" (Guide, Phase 4).
3. **Fidelidad por envoltura, no por copia.** El GET y el campo salen de las **mismas fuentes** que
   `build_system_prompt` usa (loader / `_STATIC_INSTRUCTIONS` y `_build_context_block`), nunca de
   una reconstrucción. Reimplementar el prompt sería el acoplamiento-por-duplicación que el
   thin-client rule prohíbe: el diagnóstico dejaría de ser fiel en cuanto una copia divergiera.
4. **Sin versionado de prompt en el shape.** Git es el histórico de versiones (brief 1). No se
   añade un id/versión de prompt al contrato en este brief (`# TODO` si algún día hace falta).
5. **Nota de exposición, no bloqueo:** exponer el system prompt públicamente es deseable **para
   esta demo** (transparencia = feature; corpus público). En un despliegue a cliente real este GET
   podría querer auth/gating — decisión futura, fuera de alcance aquí. Anótalo, no lo implementes.

---

## 3. Implementación — backend

### 3.1 Schema
- **`PromptResponse`** (nuevo, `schemas/query.py`): `{ instructions: str }`.
- **`QueryRequest`**: nuevo campo `include_context: bool = False` (opt-in del contexto en la
  respuesta). Default `False` → comportamiento actual intacto para consumidores existentes.
- **`QueryResponse`** y **`StreamDoneEvent`**: nuevo campo `injected_context: str | None = None`.
  Poblado **solo** si `include_context=True`; `None` en caso contrario. Nombre `injected_context`
  (neutral: en M1 es el fragmento estático, en M2 los chunks recuperados — mismo campo).

### 3.2 `GET /api/v1/prompt`
- Router **delgado** (`routers/queries.py` o un router de diagnóstico): recibe, delega, devuelve.
  No construye nada.
- Delega en una función de service (p. ej. `llm_service.get_active_instructions() -> str`) que
  devuelve **exactamente** `_STATIC_INSTRUCTIONS` (el mismo valor cargado por el loader que
  `build_system_prompt` compone). **No** re-leer el archivo por separado ni reconstruir — devolver
  la constante ya cargada, para que sea imposible que diverja del prompt real.
- Comentarios inline en español (qué es un `GET`, por qué delega en service, por qué devuelve la
  constante y no re-lee el archivo). Type hints + docstring Google en inglés.

### 3.3 Contexto en la respuesta de `/query` y `/query/stream`
- `answer_query` y `stream_answer_query` ya construyen el contexto vía `_build_context_block`.
  Cuando `include_context=True`, incluir ese **mismo** string en `injected_context` del resultado.
- Hazlo en el punto de ensamblado compartido (`_assemble_response`), para que **ambos** endpoints
  lo cubran sin duplicar. En el stream va en el `event: done` (donde ya viajan citas/usage).
- **Fidelidad:** el `injected_context` devuelto debe ser el **mismo** string que se compuso en el
  prompt enviado al modelo, no una segunda derivación de `ECSS_REFERENCE`. Si hay riesgo de
  divergencia, captura el contexto una vez durante el ensamblado y reúsalo para el prompt y para el
  campo.

### 3.4 Consecuencias que NO ocurren (verificar)
- **La clave de caché no cambia.** `include_context` es un flag de *presentación de diagnóstico*,
  no afecta al prompt enviado ni a la respuesta generada → **no** debe entrar en `build_cache_key`.
  Dos requests idénticas que difieren solo en `include_context` deben seguir dando el mismo hit de
  caché (el contexto se adjunta al servir, no cambia qué se generó/cacheó). Verifícalo con un test.
- El `GET /prompt` no toca caché ni modelo — es lectura de una constante.

---

## 4. Implementación — frontend (cierre del anclaje; capa separable)

> Esta sección **consume** lo anterior por HTTP. Es front-end-only (thin-client). Si prefieres
> tratarla como un mini-brief aparte, es escindible — pero es pequeña y cierra el `# TODO`.

- `frontend/streamlit_app.py`: el sidebar deja de tener los dos `# TODO`.
  - **Instrucciones activas:** una llamada `GET /api/v1/prompt` (vía el cliente HTTP existente,
    sin importar nada del backend) y se muestran en el sidebar.
  - **Contexto CAG activo:** las llamadas a `/query` / `/query/stream` pasan `include_context=True`
    y el sidebar muestra `injected_context` de la última respuesta.
- Respeta el desacople por imagen: el frontend solo habla HTTP, no importa `llm_service` ni nada
  del backend (ya enforced por el grupo de dependencias `frontend`).
- El sidebar puede mostrarlas separadas (instrucciones / contexto) o concatenadas; es presentación,
  libre. En M1, instrucciones + contexto reproducen el prompt de esa query (contexto fijo); en M2
  el contexto será el de la última query (per-query), y sigue siendo fiel.

---

## 5. Tests

Levanta Docker y prueba la imagen ANTES de la suite. No rompas la fixture `autouse` de
`tests/app/conftest.py`.

### Backend (sin API, en CI)
- **Fidelidad del GET:** `GET /api/v1/prompt` devuelve un `instructions` **idéntico** a
  `_STATIC_INSTRUCTIONS` (el mismo que `build_system_prompt` compone). Este test es el que garantiza
  que el diagnóstico no diverge del prompt real.
- **Opt-in del contexto:** `/query` con `include_context=True` → `injected_context` presente e
  **igual** al bloque que `_build_context_block` produce; `/query` sin el flag (o `False`) →
  `injected_context is None`.
- **Fidelidad del contexto:** el `injected_context` devuelto coincide con el contexto realmente
  compuesto en el prompt (no una segunda derivación divergente).
- **Caché insensible al flag:** dos requests iguales que solo difieren en `include_context` →
  mismo `cache_hit` behaviour (el segundo es hit); `include_context` **no** está en la clave.
- **Capas:** el router del GET no construye el prompt (delega en service). El contexto se adjunta en
  el punto compartido (un test que verifique que `/query/stream` también trae `injected_context` en
  `done` bajo el flag).

### Frontend
- Verificación manual contra Docker (como en S03 brief 5): el sidebar muestra instrucciones y
  contexto tras una query, y ambos coinciden con lo que el backend envió.

---

## 6. Tests de usuario (contra Docker)
- `GET /api/v1/prompt` en Swagger/curl → devuelve las instrucciones activas (con la sección de
  scope del brief 1).
- `/query` con `include_context=true` → respuesta trae `injected_context`; sin el flag, no.
- Streamlit: el sidebar ya no tiene `# TODO`; muestra instrucciones + contexto de la última query.

---

## 7. Docs a actualizar
- **`S04-decisions-2-...md`** (lo escribes tú, sube al chat): por qué instrucciones-GET vs
  contexto-en-respuesta (naturaleza estática vs dinámica-en-M2), por qué opt-in (metadata-not-
  content + payload M2), la fidelidad por envoltura, y la nota de exposición/gating futuro.
- **`ARCHITECTURE.md`**: nuevo `GET /api/v1/prompt`; campo `injected_context` en el contrato;
  `include_context` en `QueryRequest`; nota de que el flag no entra en la clave de caché.
- **`CLAUDE.md`**: **quitar** de "Decisiones pendientes" el ítem del endpoint que expone system
  prompt/contexto CAG (queda resuelto); reflejar el nuevo endpoint y campo en la estructura.
- **`CHANGELOG.md`**: entrada `feat`.

---

## 8. Criterios de cierre
- [ ] `GET /api/v1/prompt` devuelve `instructions` idéntico a `_STATIC_INSTRUCTIONS` (fidelidad).
- [ ] `include_context` en `QueryRequest` (default `False`); `injected_context` en `QueryResponse`
      y `StreamDoneEvent`, poblado solo bajo el flag, fiel al contexto compuesto.
- [ ] `include_context` **no** está en `build_cache_key`; caché insensible al flag (test verde).
- [ ] Router del GET delgado (delega en service); contexto adjuntado en el punto compartido (cubre
      ambos endpoints).
- [ ] Sidebar de Streamlit sin los dos `# TODO`; consume GET + campo por HTTP, sin importar backend.
- [ ] Docker levanta; `/health` responde; `uv run pytest` verde.
- [ ] Sin versionado de prompt en el shape; nota de gating futuro anotada, no implementada.
- [ ] Docs actualizadas (incl. retirar el `# TODO` de decisiones pendientes de `CLAUDE.md`).
- [ ] Commit lo hace el usuario.
