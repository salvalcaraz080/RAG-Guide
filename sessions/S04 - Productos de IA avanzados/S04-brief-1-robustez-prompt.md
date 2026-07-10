# S04 — Brief 1/2: Robustez de prompt (instrucciones como artefacto + scope + "no sé")

> **Para:** Claude Code. **Sesión:** S04 (productos avanzados sobre CAG).
> **Tipo:** dominio (RAG-ECSS). **No** decidas arquitectura por tu cuenta: si encuentras una
> decisión pendiente no cubierta aquí, márcala con `# TODO: pendiente de decisión en Claude Chat`
> y continúa sin bloquear.
> **Fuente de verdad:** `.claude/CLAUDE.md` manda si hay contradicción con este brief.

---

## 1. Objetivo

Dos cosas, en este orden:

1. **Mover las instrucciones estáticas del system prompt a un artefacto de archivo separado**
   (`app/prompts/query/system.md`) cargado por un loader trivial. **Sin motor de plantillas**
   (nada de Jinja2). El **contexto CAG se queda donde está** (`app/context/reference.py`).
2. **Reforzar el contenido** de esas instrucciones con una sección de *scope* explícita que
   distingue **dos tipos de rechazo** ("fuera de dominio" vs "fuera del material") y formaliza el
   permiso para decir "no sé".

El movimiento (1) es un refactor que **no debe cambiar el prompt final ni un solo carácter**. El
refuerzo (2) **sí** cambia el prompt, deliberadamente, y ese cambio es el objeto de revisión.

**Por qué esta separación importa:** sacar el prompt a un archivo lo convierte en un artefacto
versionable y revisable — una PR que solo cambia el prompt tiene un diff legible sin entender el
código Python. Ese es el beneficio, y por eso hacemos el paso (1) como refactor puro antes de
tocar contenido: para que el diff del paso (2) sea *solo* el cambio de instrucciones.

---

## 2. Alcance

### Entra
- `app/prompts/query/system.md` — las instrucciones estáticas (rol, grounding, citación, formato,
  **+scope reforzado**).
- `app/prompts/loader.py` — loader trivial que lee el archivo una vez a nivel de módulo.
- Refactor de `build_system_prompt` / `_STATIC_INSTRUCTIONS` para consumir el loader.
- Tests: caracterización de refactor, estructural, y `real_llm` (3 escenarios × 2 proveedores).
- Docs: `CLAUDE.md`, `ARCHITECTURE.md`, `CHANGELOG.md`, `S04-decisions.md`.

### NO entra (no lo implementes; si aparece, `# TODO`)
- Motor de plantillas (Jinja2). El prompt en M1 no tiene condicionales; no se gana la dependencia.
- Versionado por directorios (`v1/`, `v2/`). No hay evals todavía; git *es* el histórico de
  versiones. Estructura plana: `app/prompts/query/system.md`, sin `v1/`.
- Structured output / forzar JSON en la respuesta. Es otra decisión (S11 / M2).
- Nuevos campos en `QueryResponse` para señalar out-of-scope. Ver §7 (anclaje).
- El endpoint que expone el prompt activo. Es el **brief 2/2** de esta sesión.
- Cualquier cambio al **contexto CAG** (`reference.py`) o a su bloque en el prompt.

---

## 3. Decisiones de arquitectura ya tomadas (no re-decidas)

1. **Instrucciones a archivo; contexto se queda en `reference.py`.** La anatomía es
   *instrucciones estáticas* (contrato con el modelo → archivo) vs *contexto* (dato del corpus →
   `reference.py`). `build_system_prompt` sigue componiendo ambos. El fragmento ECSS es un dato,
   no una instrucción; no se mueve.
2. **Sin motor de plantillas.** Loader = leer un `.md` a un string. Nada más.
3. **No se toca el estilo de delimitación existente.** El system prompt actual se validó
   *provider-agnostic* contra Anthropic y OpenAI en S03. Sea cual sea el estilo de delimitador que
   uses hoy (separadores `=====`, XML tags, o lo que haya), **la sección nueva de scope usa el
   mismo estilo que las secciones existentes**. No mezcles estilos. No "mejores" el delimitador.
4. **La política de fallo del guardrail de grounding/scope es `filter` (degradar), no
   `exception` ni `retry`.** En M1 se implementa **como comportamiento instruido en el prompt**
   (el modelo degrada con un mensaje claro), no como código de validación post-respuesta. No hay
   validador Python de grounding en este brief: la verificación real (comprobar que el modelo no
   alucinó una cláusula) es S11. Declararlo explícito en docs.
5. **No se fabrican guardrails artificiales.** En M1 hay *un* guardrail (grounding/scope) con una
   política (filter). Las otras dos políticas (exception, retry) son el marco para guardrails de
   input (M2) y structured output (S11) que aún no existen. Se *declaran* en docs como diseño, no
   se *implementan* inventando disparadores.

---

## 4. Implementación

### 4.1 Estructura de archivos

```
app/prompts/
├── __init__.py
├── query/
│   └── system.md        # instrucciones estáticas (rol, grounding, citación, formato, scope)
└── loader.py            # carga trivial del .md
```

El subdirectorio `query/` es **organizativo** (deja sitio a futuros casos de uso: `ingestion/`,
etc.), **no** versionado. Sin `v1/`.

### 4.2 El archivo `system.md`

Contiene **todo lo que hoy es `_STATIC_INSTRUCTIONS`**, trasladado literalmente, **más** la
sección de scope reforzada. El bloque de contexto ECSS **no** va aquí (lo compone
`build_system_prompt` desde `reference.py`).

La sección de scope reforzada tiene este **contenido** (intégralo con el estilo de delimitación
vigente en el prompt; el texto es en inglés, como el resto del prompt y las queries):

> You answer questions about ECSS (European Cooperation for Space Standardization) normative
> standards, grounded strictly in the reference material provided below.
>
> Two situations require an explicit refusal instead of an answer:
>
> 1. **Out of domain** — the question is not about ECSS standards or aerospace normative
>    engineering at all. State briefly that it falls outside what this assistant covers, and do
>    not attempt to answer it.
>
> 2. **Out of material** — the question is about ECSS standards, but the reference material below
>    does not contain what is needed to answer it. State explicitly that the provided material
>    does not cover it. Do not answer from your own general knowledge, and never invent a
>    standard, clause, or requirement to fill the gap.
>
> When the reference material does answer the question, ground every normative claim in it and
> cite it (standard + clause) as instructed above. If you are unsure whether the material
> supports a claim, treat it as *out of material* rather than guessing.

**Por qué dos tipos y no uno:** el estimator solo tiene "fuera de dominio" (reforma de baño vs
software). Nosotros tenemos además "dentro de dominio pero fuera del material inyectado" — una
pregunta legítima sobre ECSS que este fragmento concreto no cubre. Son rechazos distintos con
mensajes distintos, y la distinción es la esencia del grounding en un sistema de trazabilidad: el
"out of material" es el que impide la cita fantasma. Ambos son política *filter*.

### 4.3 El loader (`loader.py`)

Requisitos:
- Lee `query/system.md` **una vez a nivel de módulo** (igual patrón que el Router de LiteLLM en
  `llm_wrapper.py`: se construye una vez al importar, no por request). Cambiar el prompt requiere
  reiniciar el proceso — correcto para un prompt que cambia por deploy, no por request.
- Path **relativo al módulo** (`Path(__file__).parent / "query" / "system.md"`), no absoluto ni
  dependiente del cwd.
- Encoding **UTF-8 explícito**.
- Expón el resultado como una constante de módulo (p. ej. `QUERY_SYSTEM_INSTRUCTIONS: str`) o una
  función `load_query_system_instructions() -> str` que se llama una vez. Elige lo que encaje con
  cómo `llm_service.py` consume hoy `_STATIC_INSTRUCTIONS`, minimizando el cambio en el llamador.
- Comentarios inline en español explicando el **qué** y el **porqué** (este proyecto es también
  material de aprendizaje; ver convención en `CLAUDE.md`): explica qué es `Path(__file__)`, por
  qué se lee a nivel de módulo y no por request, qué hace `encoding="utf-8"`.
- Type hints y docstring Google en inglés.

### 4.4 Refactor de `build_system_prompt`

- `_STATIC_INSTRUCTIONS` deja de ser una constante literal en el código y pasa a venir del loader.
- `build_system_prompt` **sigue componiendo** instrucciones (del loader) + bloque de contexto (de
  `_build_context_block`, sin cambios). El orden del prompt no cambia.
- `_build_context_block` y `reference.py` **no se tocan**.

### 4.5 Consecuencia sobre la caché (verificar, no implementar)

- Tras el **movimiento puro** (§4.2 sin el refuerzo aún): el prompt final compuesto es
  **byte-idéntico** al actual → el hash del slot `system_prompt` en `build_cache_key` no cambia →
  la caché no se invalida. **Esto es un test, ver §5.1.**
- Tras el **refuerzo** (con la sección de scope): el prompt final cambia → el hash cambia → las
  entradas cacheadas con el prompt viejo quedan huérfanas y expiran por `CACHE_TTL`. **Esto es
  correcto y deseado**: es la invalidación por versión de prompt funcionando (el mismo mecanismo
  que `corpus_version`). No hay que hacer nada especial, solo ser consciente y anotarlo.

---

## 5. Tests

Recuerda: **levanta Docker y prueba la imagen ANTES de correr la suite** (`docker compose up
--build -d`, `curl http://localhost:8000/health`), no como paso final. Aislamiento de Redis por la
fixture `autouse` de `tests/app/conftest.py` — no la rompas.

### 5.1 Test de caracterización del refactor (transitorio → luego regresión)
- **Antes de tocar nada**, captura el prompt final actual (`build_system_prompt` con el contexto
  ECSS de M1) como *golden snapshot* en un fichero de test.
- Tras el **movimiento** (§4.2 sin refuerzo), verifica que el prompt sigue **idéntico** al
  snapshot. Si difiere aunque sea en un `\n` final, el refactor introdujo un cambio no
  intencionado — arréglalo.
- Tras el **refuerzo**, **actualiza el snapshot deliberadamente** al nuevo prompt. El diff del
  snapshot en la PR *es* la revisión del cambio de instrucciones (ese es el beneficio de tener el
  prompt como artefacto). El snapshot queda como **test de regresión** permanente.

### 5.2 Test estructural (sin API, en CI)
- `build_system_prompt` con el contexto ECSS produce un string que contiene: el rol, la regla de
  grounding, la regla de citación, el formato, **la nueva sección de scope con sus dos casos**
  ("out of domain" / "out of material"), y el bloque de contexto ECSS.
- Más robusto a wording que el snapshot exacto: asserts sobre presencia de marcadores/frases
  clave, no sobre el texto completo.

### 5.3 Tests `real_llm` (marcados, fuera de la suite por defecto)
Marca con `@pytest.mark.real_llm` (excluidos de `uv run pytest` por `addopts`; se corren con
`uv run pytest -m real_llm`). Validan comportamiento contra **ambos** proveedores reales
(Anthropic y OpenAI) — es la re-verificación provider-agnostic que exige tocar el prompt (ver
Guide/S03: "agnóstico verificado contra los proveedores que testeas, no garantizado").

Tres escenarios, cada uno contra los dos proveedores:
1. **Respondible por el fragmento** — pregunta cubierta por el fragmento ECSS inyectado → el
   modelo responde y emite cita(s). Assert: `answer` no vacío, `citations` no vacío.
2. **Out of domain** — pregunta no relacionada con ECSS/aeroespacial (p. ej. algo cotidiano) → el
   modelo rechaza como fuera de scope. Assert: `citations` vacío y el `answer` no inventa una
   cláusula ECSS. Aserción **laxa** sobre el wording (es cualitativo por naturaleza).
3. **Out of material** — pregunta legítima sobre ECSS pero **no cubierta** por el fragmento
   inyectado (p. ej. sobre un estándar distinto del que está en `reference.py`) → el modelo dice
   que el material no lo cubre, **sin inventar**. Assert: `citations` vacío y el `answer` no
   fabrica un estándar/cláusula. Aserción laxa sobre wording.

**Nota de expectativa (no es un bug):** con un solo fragmento en M1, el caso 3 ("out of material")
se disparará para *casi cualquier* pregunta que no toque ese fragmento concreto. Es honesto y
correcto. El refuerzo es sobre todo **preparación para M2** (cuando haya corpus real y el "out of
material" se reduzca a lo que ni el corpus completo cubre), más que una mejora tangible hoy. No
intentes "arreglar" que se dispare mucho.

---

## 6. Tests de usuario (contra Docker, no `uvicorn` suelto)
Propón al usuario probar en Swagger/curl contra el contenedor:
- Una pregunta respondible → respuesta con citas.
- Una pregunta claramente fuera de dominio → rechazo de scope, sin cláusulas inventadas.
- Una pregunta ECSS fuera del fragmento → "no está en el material", sin cláusulas inventadas.

---

## 7. Docs a actualizar al cerrar
- **`S04-decisions.md`** (lo escribes tú, sube al chat): log crudo de decisiones + porqué,
  etiquetado agnóstico/dominio. Incluye: la anatomía instrucciones-vs-contexto, por qué sin Jinja,
  la distinción de los dos tipos de rechazo, la política *filter* instruida-en-prompt (≠
  validación S11), la consecuencia de caché (huérfanas por refuerzo), y el resultado de la
  re-verificación provider-agnostic.
- **`CLAUDE.md`**: reflejar `app/prompts/` en la estructura; nota de que las instrucciones son
  artefacto de archivo y el contexto sigue en `reference.py`; el guardrail de scope y su política.
- **`ARCHITECTURE.md`**: actualizar §3 (orden del prompt / `build_system_prompt`) con la nueva
  fuente de las instrucciones y la sección de scope.
- **`CHANGELOG.md`**: entrada `feat` para el trabajo de la sesión; `Unreleased` refleja el estado.

**Anclaje que este brief deja puesto (para el brief 2/2 y más allá):**
- Si el frontend necesita distinguir un *rechazo* (out of domain / out of material) de una
  *respuesta* de forma programática (no por inspeccionar texto), eso es un **campo nuevo en
  `QueryResponse`** — no incluido aquí a propósito. `# TODO: pendiente de decisión en Claude Chat`
  si surge. Candidato a resolverse cuando entre structured output (S11) o al diseñar el endpoint
  del brief 2.

---

## 8. Criterios de cierre
- [ ] `app/prompts/query/system.md` + `loader.py` creados; instrucciones movidas; contexto intacto
      en `reference.py`.
- [ ] Sección de scope reforzada integrada **con el estilo de delimitación existente**.
- [ ] Snapshot de caracterización: idéntico tras el movimiento, actualizado tras el refuerzo.
- [ ] Test estructural verde en CI.
- [ ] Tests `real_llm` verdes contra Anthropic **y** OpenAI (3 escenarios).
- [ ] `docker compose up --build` levanta; `/health` responde.
- [ ] `uv run pytest` verde (suite por defecto, sin `real_llm`).
- [ ] Sin Jinja, sin `v1/`, sin cambios de contrato en `QueryResponse`.
- [ ] Docs actualizadas; commit lo hace el usuario.
