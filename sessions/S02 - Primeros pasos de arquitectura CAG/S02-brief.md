# S02-brief.md — Scaffolding FastAPI nativo-ECSS (CAG v0)

> **Destino:** repo RAG-ECSS. **Ejecuta:** Claude Code.
> **Este chat decidió; CC implementa.** Al cerrar, CC escribe `sesiones/S02-decisions.md`,
> deriva `CHANGELOG.md` y edita `CLAUDE.md`/`ARCHITECTURE.md`.

---

## 0. Objetivo y encuadre

Construir la estructura base de RAG-ECSS como servicio FastAPI con un endpoint que reciba una
**pregunta en lenguaje natural** sobre un estándar ECSS y devuelva una **respuesta anclada a un
fragmento de referencia inyectado en el prompt** (arquitectura CAG), con **citas** (documento /
cláusula / página) en la respuesta.

**Esto NO es un port 1:1 del ejercicio del máster.** El ejercicio del máster construye un estimador
de presupuestos, cuyo conocimiento (unos pocos ejemplos) cabe en contexto → CAG válido. El corpus
ECSS completo **no cabe** en contexto y exige trazabilidad → es un proyecto RAG, no CAG. Por eso
esta sesión implementa CAG **sobre una rebanada acotada** de un estándar: es CAG legítimo (la
rebanada sí cabe y es estable) y sirve para levantar la arquitectura y el contrato de citación desde
el día uno, prefigurando el límite que justificará RAG en M3+.

Diferencia conceptual respecto al estimador, que CC debe tener presente: en el estimador el contexto
estático son **ejemplos que calibran formato** (el modelo generaliza a un proyecto nuevo). Aquí el
contexto estático es **la base de conocimiento en sí** (el modelo responde *sobre* ese fragmento, no
generaliza). El sistema debe responder desde el fragmento y citarlo; si la pregunta no se puede
responder con el fragmento inyectado, debe decirlo, no inventar.

---

## 1. Fuera de alcance (NO implementar — pertenece a sesiones futuras)

- **Sin retrieval, sin vector store, sin embeddings, sin persistencia** (M3+). El conocimiento viaja
  en el prompt.
- **NO cablear `corpus/`.** El pipeline de `corpus/` está congelado y se cablea en S06/S07. Para el
  fragmento de referencia (§4) se **copia a mano** un excerpt de un artefacto ya procesado; eso NO es
  importar ni ejecutar el pipeline.
- **NO construir el wrapper de proveedor** (fallback, cache, streaming, registro multi-proveedor,
  normalización de formatos de respuesta entre proveedores). Eso es S03. Aquí solo un **seam fino**
  (§5).
- **NO implementar prompt caching** (`cache_control`, TTL, breakpoints). Es S03. Aquí solo se
  **estructura el prompt** de forma que el prefijo estático quede antes de la parte variable, para
  que S03 pueda insertar el breakpoint sin reordenar. Estructurar ≠ activar.
- **NO verificación/grounding automático de citas ni métricas de alucinación** (RAGAS, etc.). Es S11.
  Aquí las citas se **emiten** a partir de los metadatos del fragmento inyectado; no se verifican.
- **NO multi-turn / gestión de historial.** Endpoint transaccional single-turn. (Multi-turn se
  evaluará para ECSS más adelante; ver `S02-resumen.md`.)

Si CC detecta que para cumplir algo de §1 haría falta adelantar trabajo de una sesión futura, **para
y lo registra en `S02-decisions.md`** en lugar de implementarlo.

---

## 2. Git

1. Taggear el estado actual antes de tocar nada: `git tag pre-rebuild` (recupera el scaffolding del
   estimador si hiciera falta).
2. Reconstruir **solo `app/`** nativo-ECSS. **Preservar intactos** `corpus/`, `data/processed/`,
   `tests/corpus/`. `corpus/` es ortogonal a `app/`; viaja en la línea de trabajo sin merge.
3. Trabajar en rama dedicada; no mezclar con `corpus/`.

---

## 3. Estructura de proyecto (nativa ECSS)

```
rag-ecss/
├── app/
│   ├── __init__.py
│   ├── main.py               ← entrypoint, /health, metadata Swagger
│   ├── config.py             ← Pydantic BaseSettings
│   ├── routers/
│   │   ├── __init__.py
│   │   └── queries.py        ← POST /api/v1/query
│   ├── services/
│   │   ├── __init__.py
│   │   └── llm_service.py    ← construcción de prompt + seam de proveedor
│   ├── schemas/
│   │   ├── __init__.py
│   │   └── query.py          ← QueryRequest / QueryResponse / Citation
│   └── context/
│       ├── __init__.py
│       └── reference.py      ← fragmento(s) ECSS estáticos + metadatos de cita
├── corpus/                   ← CONGELADO, no tocar
├── data/processed/           ← preservar, no tocar
├── tests/
│   ├── corpus/               ← preservar, no tocar
│   └── app/                  ← tests nuevos de esta sesión
├── .env  .env.example  .gitignore  pyproject.toml  README.md
```

Dependencias (`uv`): `fastapi`, `uvicorn[standard]`, `pydantic-settings`, `anthropic`,
`python-dotenv`. (Añadir `openai` solo si se decide cablear ese proveedor en su lugar.)
Nombres de endpoint/schema en **inglés** (dominio del producto); comentarios en el idioma que CC
prefiera.

---

## 4. Capa de contexto: `context/reference.py`

Corazón de la CAG de esta sesión. Contiene **1–2 fragmentos reales y acotados** de un estándar ECSS
**CORE** (preferible ECSS-E-ST-40C o ECSS-Q-ST-80), copiados a mano desde `data/processed/`
(NO importados de `corpus/`).

Requisitos del fragmento:
- **Autocontenido y pequeño** — una cláusula o sub-sección con sentido propio (p. ej. un requisito
  con su ID), no un volcado de páginas.
- **Con sus coordenadas de cita** — cada fragmento lleva `standard`, `clause`, `page`. Estas
  coordenadas son las que poblarán el campo `citations` de la respuesta; sin ellas el contrato de
  trazabilidad no cierra ni en esta versión de juguete.
- **Representativo del reto** — idealmente una cláusula con ID de requisito y/o una referencia
  cruzada a otro estándar, para que se vea el tipo de contenido que RAG tendrá que manejar.

Estructura sugerida (CC ajusta nombres):

```python
ECSS_REFERENCE = [
    {
        "standard": "ECSS-E-ST-40C",
        "clause": "5.4.3.1",          # ID/nº real de la cláusula copiada
        "page": 47,                    # página real en el PDF fuente
        "text": "<texto literal del requisito/cláusula copiado de data/processed>",
    },
    # opcional: un segundo fragmento, p. ej. con referencia cruzada
]
```

CC debe registrar en `S02-decisions.md` **de qué artefacto de `data/processed/` salió cada
fragmento** (trazabilidad del propio dato de prueba).

---

## 5. Capa de servicio: `services/llm_service.py`

Dos responsabilidades: **construir el prompt** (formato + orden) y **llamar al modelo a través de un
seam fino**.

### 5.1 Seam fino de proveedor (Fork 2)

NO es una abstracción de proveedor (S03). Es **una única función de frontera** que aísla la llamada
al SDK, para que S03 tenga un punto limpio de inserción y para no escribir acoplamiento que sabemos
que tiraremos. Regla dura: **la llamada debe ser async-correcta**. Envolver una llamada síncrona en
`async def` NO la hace no-bloqueante — congela el event loop durante los 2–30 s de la llamada y
tira por tierra la razón de usar FastAPI. Usar el cliente async nativo del SDK.

```python
from anthropic import AsyncAnthropic
from app.config import get_settings

settings = get_settings()
_client = AsyncAnthropic(api_key=settings.ANTHROPIC_API_KEY)

async def _call_model(system: str, user: str) -> dict:
    """Seam de frontera. Único punto que conoce el SDK del proveedor.
    S03 sustituirá el cuerpo por el wrapper agnóstico."""
    resp = await _client.messages.create(
        model=settings.LLM_MODEL,
        max_tokens=settings.MAX_OUTPUT_TOKENS,
        system=system,                       # Anthropic: system va fuera del array
        messages=[{"role": "user", "content": user}],
    )
    return {
        "text": resp.content[0].text,
        "input_tokens": resp.usage.input_tokens,
        "output_tokens": resp.usage.output_tokens,
    }
```

Nota para S03 (no implementar ahora, solo dejar constancia en el código con un comentario): el manejo
de `system` como parámetro separado y de `resp.content[0].text` es **específico de Anthropic**; el
wrapper de S03 tendrá que normalizar esto si entra OpenAI.

### 5.2 Construcción del prompt (orden importa dos veces)

Orden del contenido — sirve **a la vez** para atención posicional y para futura frontera de cache
(todo lo estático antes de lo variable):

```
system (estático):
  1. Rol: experto en normativa ECSS.
  2. Instrucción de grounding: responde SOLO con la información del material de referencia;
     si no está, dilo explícitamente ("La referencia proporcionada no cubre esto"). No inventes.
  3. Instrucción de citación: toda afirmación normativa debe citar standard + clause + page
     del fragmento usado.
  4. Formato de salida esperado.
  5. ===== MATERIAL DE REFERENCIA ECSS ===== <fragmentos de context/reference.py, con delimitadores>
user (variable):
  6. La pregunta del ingeniero.
```

Formato de los fragmentos: texto estructurado con delimitadores explícitos entre fragmentos y con las
coordenadas de cita visibles al modelo (para que pueda citarlas). Markdown ligero es suficiente.

`build_system_prompt()` debe ser una función testeable sin red (se testea el string resultante, ver §8).

### 5.3 Entrada de servicio

```python
async def answer_query(question: str) -> dict:
    system = build_system_prompt(ECSS_REFERENCE)
    result = await _call_model(system, question)
    return {
        "answer": result["text"],
        "citations": <derivadas de los fragmentos inyectados; ver nota>,
        "model": settings.LLM_MODEL,
        "provider": settings.LLM_PROVIDER,
        "usage": {"input_tokens": result["input_tokens"],
                  "output_tokens": result["output_tokens"]},
    }
```

**Nota sobre `citations` (Fork 3, mínimo):** en esta versión las citas se pueblan a partir de los
metadatos de los fragmentos **inyectados** (candidatas de cita), NO extrayéndolas ni verificándolas
contra el texto de la respuesta. Es decir: el campo existe y transporta `standard/clause/page` de lo
que se metió en contexto. La verificación real (que la respuesta cite lo que efectivamente usó, y que
no alucine cláusulas) es S11 y queda fuera. Dejar comentario en el código diciendo esto, para que en
S11 no se dé por hecho que ya está resuelto.

---

## 6. Schemas: `schemas/query.py`

```python
from pydantic import BaseModel, Field

class Citation(BaseModel):
    standard: str          # p. ej. "ECSS-E-ST-40C"
    clause: str            # p. ej. "5.4.3.1"
    page: int | None = None

class QueryRequest(BaseModel):
    question: str = Field(..., min_length=10,
                          description="Pregunta en lenguaje natural sobre el corpus ECSS")

class QueryResponse(BaseModel):
    answer: str
    citations: list[Citation]     # mínimo viable; verificación → S11
    model: str
    provider: str
    usage: dict | None = None
```

`min_length` evita llamadas triviales. El contrato de citación vive en código desde v0, aunque su
cumplimiento fuerte llegue en S11.

---

## 7. Router y main

- `routers/queries.py`: `POST /api/v1/query`, `response_model=QueryResponse`, delega en
  `answer_query`. Router delgado: recibe, delega, devuelve. No construye prompt ni conoce el SDK.
- `main.py`: crea la app (título "RAG-ECSS", versión "0.1.0", descripción), incluye el router con
  prefijo `/api/v1`, añade `GET /health` → `{"status": "healthy"}`. Mantener `main.py` breve.

---

## 8. Config: `config.py`

Pydantic `BaseSettings`, carga desde `.env`, falla al arrancar si falta lo obligatorio:

```python
ANTHROPIC_API_KEY: str            # obligatoria, sin default
LLM_PROVIDER: str = "anthropic"
LLM_MODEL: str = "claude-haiku-4-5"   # ver §9
MAX_OUTPUT_TOKENS: int = 2000
APP_ENV: str = "development"
LOG_LEVEL: str = "INFO"
```

`get_settings()` con `@lru_cache`. `.env` en `.gitignore`; `.env.example` con las claves sin valores.
Las API keys nunca en código.

---

## 9. Modelo (verificado a 2026-07-03)

- Default: **`claude-haiku-4-5`** (API ID `claude-haiku-4-5-20251001`) — modelo económico de
  generación actual, $1/$5 por MTok, vigente. El `claude-haiku-4-5` del máster NO está stale.
- El `gpt-4o-mini` del ejercicio del máster **SÍ está stale**: es legacy. Si se prefiere OpenAI,
  usar **GPT-5.4 mini** ($0.75/$4.50), no `gpt-4o-mini`. (Cambiaría el seam a `AsyncOpenAI` y la
  extracción de respuesta a `resp.choices[0].message.content`.)
- Fijar `LLM_MODEL` vía config, nunca hardcodeado en el servicio.

---

## 10. Testing (pipeline: implementación → tests auto → tests usuario → docs)

Tests automáticos en `tests/app/` (no tocar `tests/corpus/`):

1. `GET /health` → 200 `{"status":"healthy"}`.
2. `build_system_prompt()` — sin red: el string contiene el rol, la instrucción de grounding, la de
   citación, los delimitadores de referencia y el texto de los fragmentos. Orden correcto (referencia
   antes que — bueno, la pregunta va en `user`, así que verificar que el system termina con el
   material de referencia).
3. `POST /api/v1/query` con el SDK **mockeado** (sin llamada real): valida que `QueryResponse`
   serializa, que `citations` se puebla con las coordenadas de los fragmentos inyectados, y que
   `question` < `min_length` da 422.
4. **Test de grounding (mock):** si la respuesta simulada no cubre la pregunta, el sistema no
   inventa cita (comportamiento de "no cubierto"). Verificable con un mock que devuelva la frase de
   "no cubierto".

Test de usuario (manual, 1 llamada real): levantar `uv run uvicorn app.main:app --reload`, lanzar una
pregunta cuya respuesta esté en el fragmento inyectado (p. ej. sobre la cláusula que se copió), y
verificar que (a) responde desde el fragmento, (b) cita `standard/clause/page` correctos, (c) ante una
pregunta fuera del fragmento responde "no cubierto" en vez de inventar. Registrar el resultado.

Docs: `README.md` con setup (`uv sync`, `.env`), arranque, ejemplo de `curl` contra `/api/v1/query`,
y una nota de que la respuesta está anclada a un fragmento estático (CAG v0), no al corpus completo.

---

## 11. Qué registrar en `S02-decisions.md` (para la extracción posterior del chat)

Etiquetar cada decisión **agnóstico** (candidata a Guide) vs **dominio** (solo ECSS):

- [agnóstico] CAG sobre rebanada acotada como forma de levantar arquitectura + contrato de salida
  antes de tener retrieval. Condición de aplicabilidad y límite (cuándo deja de servir → RAG).
- [agnóstico] Seam fino de proveedor con llamada async-correcta como paso previo al wrapper; el bug
  de "cliente síncrono en `async def`" evitado explícitamente.
- [agnóstico] Orden de prompt que sirve a atención y a futura frontera de cache simultáneamente.
- [agnóstico] Campo de citación en el schema desde v0 aunque la verificación sea posterior.
- [dominio] Qué fragmento(s) ECSS se eligieron, de qué artefacto de `data/processed/` salieron, y por
  qué (representatividad).
- Cualquier fricción de implementación que contradiga la teoría del máster (material para la
  divergencia teoría-vs-implementación del consolidado).
```

