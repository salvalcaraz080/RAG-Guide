# S03 · Brief 1 — Abstracción de proveedor + fallback (LiteLLM Router)

> Instrucciones para Claude Code. Este chat decide, CC implementa. CC, al cerrar, escribe
> `S03-decisions.md`, deriva `CHANGELOG.md` y edita `CLAUDE.md`/`ARCHITECTURE.md`.
> **Este es el primero de 5 briefs de S03.** Orden de dependencia: **1-abstracción/fallback** →
> 2-cache → 3-streaming → 4-observabilidad → 5-Streamlit. No reordenar: el brief de observabilidad
> loguea `fallback_used`, que solo existe si este brief está implementado.

## Objetivo

Sustituir el cuerpo del seam `_call_model` (en `app/services/llm_service.py`) por un wrapper basado
en **LiteLLM Router** que: (a) abstrae el proveedor tras un nombre lógico, (b) añade **fallback
secuencial** de Anthropic (primario) a OpenAI (respaldo), (c) es **async-correcto**, y (d) devuelve
una **respuesta normalizada** que alimenta `QueryResponse` sin cambiar el contrato HTTP.

Lo que **no** entra en este brief: cache (brief 2), streaming (brief 3), logging estructurado
(brief 4). El wrapper debe quedar preparado para que esos se enganchen encima, sin implementarlos.

## Por qué así (contexto de decisión)

- El seam ya existe desde S02: es el punto de inserción, no hay que crearlo. Este brief cambia su
  *cuerpo*, no su posición.
- Se eligió **Router completo desde ya** (no `acompletion` simple) para que el fallback quede montado
  y testeable con mock desde este primer brief. **Matiz honesto:** montar el Router con fallback
  configurado NO significa que el fallback esté validado end-to-end. La validación real de OpenAI
  como respaldo (que responde aceptablemente a una query ECSS y que su respuesta normalizada trae
  los campos esperados) queda como **tarea abierta** al final de este brief — requiere la API key de
  OpenAI cableada y una prueba real, no solo el mock.
- LiteLLM normaliza la respuesta al **formato OpenAI** para todos los proveedores
  (`choices[0].message.content`), lo que simplifica la extracción. El precio de esa normalización
  (posible aplanamiento de features específicos de proveedor, p. ej. `cache_control` de Anthropic)
  NO muerde en este brief, pero condiciona un requisito de diseño (ver "No cerrar puertas").

## Ubicación y estructura

- **Módulo nuevo:** `app/services/llm_wrapper.py` — construye y expone el Router. Infraestructura,
  aislada de la lógica de negocio.
- **`app/services/llm_service.py`** — `_call_model` pasa a delegar en el wrapper. `llm_service` sigue
  siendo el dueño de la construcción del prompt y de armar `QueryResponse`; NO conoce LiteLLM.
- **`app/config.py`** — `Settings` gana los campos de configuración del Router (abajo).

Regla de dependencia (respetar la de `ARCHITECTURE.md §3`): `routers` → `services` → (`schemas`,
`context`). El wrapper es un detalle de `services`; el router HTTP no lo conoce.

## Configuración (`Settings`, pydantic-settings)

Las API keys y los identificadores de modelo vienen de entorno, **nunca inline** (el snippet del
material los hardcodea — no copiar). Campos:

```
# Ya existentes (S02) — se mantienen:
ANTHROPIC_API_KEY   # obligatoria, sin default → fail-fast si falta
OPENAI_API_KEY      # ahora pasa a ser relevante (fallback)
LLM_PROVIDER        # anthropic (primario en desarrollo)
LLM_MODEL           # claude-haiku-4-5-20251001

# Nuevos para el Router:
LLM_FALLBACK_MODEL  # modelo OpenAI de respaldo — VER verificación (2) antes de fijar
MAX_OUTPUT_TOKENS   # ya existe (2000)
```

`Settings` ya falla al arrancar si falta config obligatoria (patrón S02) — mantener ese
comportamiento. `OPENAI_API_KEY` para el fallback: decidir si es obligatoria (fail-fast) u opcional
(el Router arranca solo con primario y el fallback se activa cuando la key existe). **Recomendación:**
opcional con warning al arrancar si falta, para no bloquear desarrollo sin key de OpenAI; pero
entonces el fallback no está realmente disponible y eso debe quedar claro en el log de arranque.

## Contrato del wrapper

Firma async (esqueleto — CC implementa el cuerpo):

```python
# app/services/llm_wrapper.py

async def call_llm(
    system_prompt: str,
    user_message: str,
    *,
    model: str | None = None,      # override opcional del nombre lógico
    **provider_params,             # ← no cerrar puertas (ver abajo)
) -> LLMResult:
    """Llama al LLM vía Router con fallback. Devuelve respuesta normalizada."""
```

`LLMResult` — dataclass o Pydantic model (nunca dict crudo; `CLAUDE.md`). Campos mínimos que el
resto del sistema necesita:

```
text: str                 # choices[0].message.content normalizado
model_used: str           # modelo que REALMENTE respondió (clave para fallback_used, brief 4)
provider: str             # derivado del modelo usado
tokens_in: int            # usage.prompt_tokens
tokens_out: int           # usage.completion_tokens
finish_reason: str        # "stop" | "length" | ... (clave para brief 3, truncamiento)
```

`llm_service` mapea `LLMResult` → `QueryResponse` (que ya transporta `model`, `provider`, `usage`).
Las **citas** se siguen construyendo en `llm_service` desde `context/reference.py`, NO desde la
respuesta del modelo (contrato de citación de M1, sin cambios).

## Comportamiento requerido

**Abstracción.** El `model_list` del Router mapea un nombre lógico (p. ej. `"ecss-primary"` /
`"ecss-fallback"`, o un único grupo con fallback entre nombres) a los modelos físicos. `llm_service`
usa el nombre lógico; no sabe qué proveedor respondió.

**Fallback secuencial.** Sintaxis correcta verificada contra la doc de LiteLLM (2026):

```python
fallbacks=[{"ecss-primary": ["ecss-fallback"]}]
```

⚠️ **Bug del material a NO reproducir:** el snippet del artículo usaba `model_name="estimator"` pero
`fallbacks=[{"estimador": [...]}]` (con 'a') — las claves no casaban y el fallback no dispararía. Las
claves de `fallbacks` DEBEN casar exactamente con `model_name` del `model_list`.

⚠️ **No confundir `order` con `fallbacks`** (la verificación los separó): `order` prioriza deployments
del *mismo* `model_name` (redundancia intra-nombre); `fallbacks` salta de *un* `model_name` a *otro
distinto*. Para Anthropic→OpenAI (proveedores distintos) el mecanismo es **`fallbacks`**, no `order`.

**Async correctness (requisito duro, no opción).** Usar `router.acompletion()`, nunca
`router.completion()`. La verificación reveló que esto va más allá de no bloquear el event loop:
**parte de la lógica de routing/fallback se comporta distinto en sync vs async** — el camino síncrono
cae a un fallback diferente. Todos los snippets del material son síncronos; NO copiarlos.

**Normalización.** Extraer `text`, `usage` y `finish_reason` de la respuesta normalizada de LiteLLM
(formato OpenAI para todos los proveedores). Verificar que `usage` viene poblado tanto para Anthropic
como para OpenAI (ver verificación 3).

## No cerrar puertas (requisitos de diseño para briefs futuros)

- **`**provider_params` en la firma:** el brief 2 (cache) podría necesitar inyectar `cache_control`
  de Anthropic vía passthrough de params específicos de proveedor. El wrapper debe **aceptar y
  reenviar** params extra al `acompletion`, aunque este brief no los use. No hardcodear la lista de
  params.
- **`model_used` en `LLMResult`:** el brief 4 (observabilidad) deriva `fallback_used` comparando el
  modelo pedido con `model_used`. Este campo debe existir desde ya aunque aquí no se loguee.
- **`finish_reason` en `LLMResult`:** el brief 3 (streaming) lo trata como fallo de integridad
  (`"length"` = respuesta potencialmente sin cita). Debe existir desde ya.

## Tests obligatorios (`tests/app/`, sin red, con mocks)

1. **Fallback rota al respaldo.** Con `mock_testing_fallbacks=True` (o mockeando el primario para que
   lance un error que dispare fallback), verificar que la respuesta llega y que `model_used` es el del
   fallback, no el del primario.
2. **Sin fallo, responde el primario.** `model_used` == modelo primario; no se toca el respaldo.
3. **Normalización completa.** `LLMResult` trae `text`, `model_used`, `provider`, `tokens_in`,
   `tokens_out`, `finish_reason` poblados. Mockear la respuesta del Router con forma OpenAI.
4. **Async.** `call_llm` es coroutine; se ejecuta bajo `pytest-asyncio` sin bloquear.
5. **Config fail-fast.** Si falta `ANTHROPIC_API_KEY`, `Settings` falla al arrancar (regresión del
   patrón S02).
6. **Regresión de grounding/citación.** El path `POST /api/v1/query` existente sigue devolviendo
   respuesta con citas construidas desde `reference.py`. El wrapper no rompe el contrato M1.

Los errores no se silencian (`except: pass` prohibido, `CLAUDE.md`). Tipos de error diferenciados
(auth no reintenta/rota; timeout reintenta; rate-limit/server rota) — LiteLLM Router los maneja vía
`retry_policy`; configurar de forma explícita, no por defecto ciego.

## Verificaciones que CC debe hacer antes/durante la implementación

1. **Model string de LiteLLM para Anthropic con fecha.** Confirmar el identificador exacto (prefijo
   de proveedor + id con fecha, previsiblemente `anthropic/claude-haiku-4-5-20251001`). El material
   usa `claude-haiku-4-5` sin prefijo ni fecha — no fiarse.
2. **Modelo OpenAI de respaldo (`LLM_FALLBACK_MODEL`).** El material usa `gpt-4o-mini` (stale, mismo
   patrón que S02). Elegir un modelo OpenAI vigente y de capacidad comparable a Haiku para el rol de
   respaldo. Verificar vigencia antes de fijar; dejar el valor en `.env`, no hardcodeado.
3. **`usage` poblado en ambos proveedores.** Confirmar que la respuesta normalizada de LiteLLM trae
   `prompt_tokens`/`completion_tokens` tanto para Anthropic como para OpenAI (el mapeo de usage entre
   proveedores a veces difiere). Si algún campo no viene, documentarlo en `S03-decisions.md`.

## Tarea abierta (marcar, no cerrar en este brief)

- **Validación end-to-end de OpenAI como fallback.** El Router queda montado con el fallback
  configurado y testeado con mock. Falta: con `OPENAI_API_KEY` real cableada, una prueba manual que
  dispare el fallback contra una query ECSS real y confirme que la respuesta del respaldo es
  aceptable y que su normalización trae todos los campos. Hasta entonces, el fallback está
  *configurado y mock-testeado*, no *validado*. Registrar el estado en `S03-decisions.md`.

## Pipeline de cierre (CC)

Implementación → tests automáticos (`uv run pytest`) → tests de usuario (proponer prueba manual:
arrancar el servicio, `POST /api/v1/query` con una pregunta ECSS, verificar respuesta con citas;
opcionalmente forzar fallback) → docs (`S03-decisions.md` con etiquetado agnóstico/dominio, derivar
`CHANGELOG.md`, actualizar `CLAUDE.md`/`ARCHITECTURE.md`). El commit lo hace el usuario.
