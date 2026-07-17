# S05 — Decisions log, brief 2: modo evaluación (`LLM_EVAL_MODE`) + reencuadre del contrato de testing

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier proyecto
> LLM) / `[dominio]` (específico de RAG-ECSS).

---

## 1. Evaluar es una función del PRODUCTO, no una actividad externa `[agnóstico]`

Encuadre central del brief. Evaluar un componente probabilístico (el LLM) es parte del
sistema, no algo que se hace "por fuera" con scripts sueltos. De ahí que el modo evaluación
sea una configuración con NOMBRE ÚNICO (`LLM_EVAL_MODE`) de primera clase, no un conjunto de
flags ad-hoc. La razón concreta que lo fuerza: la **cache existe para SUPRIMIR la varianza
que la evaluación existe para MEDIR** — son opuestas. Sin un modo que desactive la cache, un
test de consistencia (misma pregunta N veces, ¿responde consistente?) mediría la cache y no
el modelo: verde y midiendo la dimensión equivocada.

## 2. Activación por ENTORNO/MODO, no por flag de request (decisión cerrada) `[agnóstico]`

La cache-on/off es una propiedad del **despliegue** en que corres, no de la pregunta
concreta. Se descartó explícitamente un flag de request que desactivara la cache desde fuera:

- Sería **superficie de ataque**: un cliente podría forzar bypass en cada petición y disparar
  el coste (cada request iría al modelo, sin cache que lo amortigüe).
- Contradiría el invariante ya establecido en el proyecto de que **la elegibilidad de cache la
  decide el sistema, no el cliente** (mismo principio que `include_context` en S04 brief 2: un
  flag de presentación nunca es slot de la clave de cache).

El modo por entorno no añade ninguna superficie al contrato público de la API — no hay campo
nuevo en `QueryRequest`, no hay nada que un cliente pueda tocar.

## 3. Guardarraíl fail-fast en producción: la barrera de arranque es lo que hace el modo seguro `[agnóstico]`

`model_validator(mode="after")` en `Settings` rechaza construir la configuración si
`LLM_EVAL_MODE is True` y `APP_ENV == "production"`. Decisión de diseño, no defensividad
gratuita: un modo que apaga protecciones y optimizaciones (hoy la cache; mañana, potencialmente,
logging de contenido completo del prompt/respuesta) activado por accidente en producción sería
catastrófico — coste disparado, posible fuga de contenido a logs. La barrera de **arranque**
(no una comprobación por request) es exactamente lo que permite que el modo sea MÁS potente y a
la vez seguro: si arrancó, es que el entorno lo permite; nadie tiene que acordarse de
comprobarlo en el camino caliente.

Mismo espíritu fail-fast que un secreto obligatorio sin default (`ANTHROPIC_API_KEY`): fallar al
construir `Settings`, no en la primera request. `model_validator(mode="after")` corre DESPUÉS de
parsear todos los campos y puede validar la RELACIÓN entre dos (`LLM_EVAL_MODE` + `APP_ENV`),
algo que un validador por-campo no ve.

Sub-decisión de alcance del guardarraíl: **solo se prohíbe `production`**. Cualquier otro
`APP_ENV` (development, test, staging…) permite el modo. No hubo ambigüedad de valores de
`APP_ENV` en este proyecto (`development`/`production` son los usados), así que no hizo falta el
`# TODO: pendiente de decisión` que el brief dejaba como salida.

## 4. Un único sitio de decisión: `_cache_eligible`, no dos ramas paralelas `[agnóstico]`

El bypass del modo eval se COMPONE con el bypass del multi-turn del brief 1 (historial no
vacío). En vez de añadir una segunda condición suelta en cada uno de los dos caminos
(`answer_query` y `stream_answer_query`), se factorizó la decisión en un helper único:

```python
def _cache_eligible(history: list[Message]) -> bool:
    return len(history) == 0 and not settings.LLM_EVAL_MODE
```

Ambas funciones llaman a `_cache_eligible(history)`. Beneficio: la lógica de "cuándo la cache
está permitida" vive en UN sitio (cualquiera de los dos bypasses la desactiva), y ese sitio es
donde el brief pide dejar el `# TODO` de las futuras propiedades derivadas del modo (logging de
contenido, sampling) — para que crezcan desde un punto, no dispersas.

## 5. Reutilizar el seam de bypass, no inventar uno nuevo `[agnóstico]`

El mecanismo de "servir sin cache" ya existía desde S03 brief 2: la cache es degradable, con un
camino de tolerancia a fallos de Redis. El modo eval reutiliza ESE camino con otra intención
(aislar el modelo bajo medida en vez de sobrevivir a un Redis caído). "Un buen seam paga dos
veces": no hubo que construir un mecanismo de bypass, solo activarlo desde la compuerta. El
cambio real de este brief es minúsculo (una condición más en `_cache_eligible` + el flag + el
guardarraíl) precisamente porque el seam ya estaba.

## 6. Dos suites soberanas, no una con excepciones `[agnóstico]`

Reencuadre de la regla de testing del proyecto (edición de estado de `CLAUDE.md`, no nota de
Guide). Hasta ahora la regla era "una suite mockeada + una excepción `real_llm`". El modo eval
obliga a nombrar que son **dos suites soberanas**:

- **Suite de contrato (la de CC):** verifica que el CÓDIGO respeta el contrato. Binaria,
  determinista, mockeada, sin red/Redis/modelo real. Es `uv run pytest`, único paso de
  verificación de CC. La regla "sin red, sin BD" **no se afloja** — se ACOTA a esta suite.
- **Suite de evaluación (la del desarrollador):** verifica que el LLM produce output de
  CALIDAD. Gradiente (score/distribución/tendencia), no determinista, modelo real + cache
  desactivada (`LLM_EVAL_MODE`), marca `real_llm` (ya existente desde S03), fuera de la suite
  por defecto. No bloquea merge, no es responsabilidad de CC. Su contenido serio (consistencia,
  LLM-as-judge, golden dataset, faithfulness) es S11/S15.

**Invariante que se hace explícito:** los tests de contrato NO pueden depender de qué proveedor
respondió (hay fallback; fijar la salida de un proveedor rompería el test cuando responde el
otro, que es camino normal). Se testean propiedades del sistema, no salidas concretas del
modelo — esto ya se seguía en la práctica desde S03, ahora queda escrito.

## 7. Qué NO se implementó (diferido a propósito) `[dominio]`

- La suite de evaluación en sí (familias 2 y 3: consistencia, LLM-as-judge, golden dataset).
  Es S11/S15 — necesita retrieval y fidelidad que evaluar. En M1 (un fragmento estático) casi
  no hay qué evaluar.
- DeepEval / RAGAS como dependencia. Cuando llegue la evaluación de calidad real, no ahora.
- Logging de contenido completo (prompt/respuesta) bajo el modo, sampling, etc. Son propiedades
  que el modo DERIVARÁ cuando existan; hoy solo desactiva la cache. `# TODO` comentado en
  `_cache_eligible` señalando el punto de extensión.

## 8. Qué NO se tocó `[agnóstico]`

- `build_cache_key`: sin cambios (el modo no añade slots; desactiva la cache, no la reconfigura).
- El contrato HTTP (`QueryRequest`/`QueryResponse`): sin campos nuevos — la activación es por
  entorno, no por request (§2).
- La lógica de sesión del brief 1: intacta; el modo solo añade un término a la compuerta que
  ella ya alimentaba.
