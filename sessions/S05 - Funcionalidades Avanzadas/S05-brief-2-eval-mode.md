# S05 — Brief 2: Modo evaluación (`LLM_EVAL_MODE`) y reencuadre del contrato de testing

> Instrucciones de implementación para Claude Code. Funcionalidad separada del brief 1 (config +
> documentación, casi sin lógica), testeable de forma independiente. CC implementa dentro del
> contrato; no reabre las decisiones marcadas como cerradas.

---

## 1. Objetivo

Introducir el **modo evaluación** como pieza de primera clase del sistema: una configuración con
nombre único que, al activarse, pone al sistema en la disposición necesaria para evaluar la
calidad del output del LLM. Y reencuadrar la regla de testing del proyecto para reflejar que hay
**dos suites soberanas**, no una con excepciones.

Contexto de por qué (breve): evaluar un componente probabilístico es una función del producto, no
una actividad externa. La cache existe para *suprimir* la varianza que la evaluación existe para
*medir* — son opuestas. Sin un modo que desactive la cache (entre otras cosas), un test de
consistencia mediría la cache y no el modelo: verde y midiendo la dimensión equivocada.

---

## 2. Alcance

**SÍ entra:**
- `LLM_EVAL_MODE: bool = False` en `Settings`.
- Guardarraíl de arranque: **fail-fast** si `LLM_EVAL_MODE` está activo en entorno de producción.
- Cableado mínimo que S05 necesita del modo: **bypass total de cache** (get y set) cuando el modo
  está activo.
- `.env.example`: documentar la variable.
- `CLAUDE.md`: reencuadrar la regla de tests (dos suites) y documentar el modo.
- Tests de contrato (familia 1) del guardarraíl y del bypass.

**NO entra (diferido, no implementar):**
- La suite de evaluación en sí (familias 2 y 3: consistencia, LLM-as-judge, golden dataset). Es
  S11/S15 — necesita retrieval y fidelidad que evaluar. En M1 (un fragmento estático) casi no hay
  qué evaluar.
- DeepEval / RAGAS como dependencia. Se evalúan cuando llegue la evaluación de calidad real, no
  ahora.
- Logging de contenido (prompt/respuesta completos) bajo modo eval, sampling, etc. Son propiedades
  que el modo *derivará* cuando existan; hoy no existen. Dejar un `# TODO` comentado en el punto
  donde el modo se consulta, señalando que aquí se añadirán futuras propiedades derivadas.
- Cualquier desactivación por **flag de request**. La activación es por **entorno/modo**, no por
  petición (ver §4).

---

## 3. `LLM_EVAL_MODE` en `Settings` + guardarraíl de arranque

Añadir a `app/config.py` (`Settings`, pydantic-settings):

- `LLM_EVAL_MODE: bool = False`. Por defecto desactivado: producción y desarrollo normal no lo
  tocan.

**Guardarraíl (fail-fast, `model_validator`):** el sistema **se niega a arrancar** si
`LLM_EVAL_MODE` está activo bajo entorno de producción. Un modo que apaga protecciones y
optimizaciones es catastrófico activado por accidente en producción; la barrera de arranque es lo
que hace que el modo, siendo más potente, sea también seguro.

- Condición exacta a rechazar: `LLM_EVAL_MODE is True` **y** `APP_ENV == "production"` → lanzar
  error de configuración claro al construir `Settings` (mismo espíritu fail-fast que los secretos
  obligatorios sin default: fallar al arrancar, no en la primera request).
- `# TODO: pendiente de decisión en Claude Chat` si CC detecta que los valores de `APP_ENV` en uso
  no son exactamente `development`/`production` y hay ambigüedad sobre qué entornos deben permitir
  el modo. Por defecto: solo se prohíbe `production`; el resto lo permite.

---

## 4. Cableado del bypass de cache

Cuando `settings.LLM_EVAL_MODE` está activo, la cache se **desactiva por completo** (get y set),
globalmente, para todas las peticiones.

Reutiliza el mismo punto de decisión que la compuerta de historial del brief 1. La elegibilidad de
cache pasa a ser:

```
cache_eligible = (len(history) == 0) and (not settings.LLM_EVAL_MODE)
```

Es decir: el bypass del modo eval se compone con el bypass del multi-turn — cualquiera de los dos
desactiva la cache. Un solo sitio de decisión, no dos ramas paralelas.

**Por qué por entorno/modo y no por flag de request** (decisión cerrada, no reabrir): la
cache-on/off es una propiedad del **despliegue** en que corres, no de la pregunta concreta. Un flag
de request que desactive la cache desde fuera sería superficie de ataque (forzar bypass y disparar
coste) y contradiría que la elegibilidad la decide el sistema, no el cliente. El modo por entorno no
añade superficie al contrato público.

**Relación con el seam de S03:** el mecanismo de bypass no es nuevo — la cache ya es degradable, con
un camino de "servir sin cache" existente (S03, tolerancia a fallos de Redis). El modo eval reutiliza
ese camino con otra intención (aislar el modelo bajo medida). Un buen seam paga dos veces; no hay que
inventar un mecanismo de bypass, solo activarlo desde el modo.

---

## 5. `.env.example`

Añadir, documentada:

```
# Modo evaluación: desactiva cache (y, en el futuro, activa logging de contenido y
# sampling para medir la calidad del LLM). NO activar en producción — el arranque falla
# si LLM_EVAL_MODE=true con APP_ENV=production. Uso: suite de evaluación del desarrollador,
# NO la suite de contrato de CC.
LLM_EVAL_MODE=false
```

---

## 6. Reencuadre de la regla de tests en `CLAUDE.md`

Esto es **edición de estado del repo**, no una nota de Guide. La regla actual de `CLAUDE.md` dice,
tajante:

> "Los tests deben ejecutarse sin red y sin base de datos — usar fixtures y mocks. El único paso de
> verificación de CC es `uv run pytest` (mockeado, sin red)."

Esa frase **no se elimina ni se afloja** — se **acota**: sigue siendo la regla de la *suite de
contrato* y de *CC*. Lo que falta es nombrar que existe una **segunda suite soberana**. Reescribir
la sección de tests para reflejar las dos, dejando claro que el mundo de CC no cambia:

**Redacción a introducir (adaptar al estilo del documento):**

- **Suite de contrato (la de CC).** Verifica que el *código* respeta el contrato. Veredicto binario
  (falla/no falla), determinista, **mockeada, sin red ni Redis ni modelo real**. Es `uv run pytest`
  y es el único paso de verificación de CC. Familia 1 (validez de schema, campos, rangos,
  invariantes, orden de ensamblado, compuertas de cache) con la respuesta del LLM mockeada. **CC no
  toca red, ni Docker, ni la suite de evaluación.**
- **Suite de evaluación (la del desarrollador).** Verifica que el *LLM* produce output de calidad.
  Veredicto en gradiente (score, distribución, tendencia), **no determinista**, requiere modelo real
  y **la cache desactivada** (`LLM_EVAL_MODE`). Vive **fuera** de la suite por defecto (marca
  `real_llm`, ya existente desde S03), la corre el desarrollador a demanda/periódicamente, y su
  resultado requiere análisis estadístico. **No bloquea merge, no es responsabilidad de CC.** Su
  contenido serio (consistencia, LLM-as-judge, golden dataset, faithfulness) es S11/S15; en M1 casi
  no hay qué evaluar.
- **Nota de invariante:** los tests de contrato **no pueden depender de qué proveedor respondió**
  (hay fallback; fijar la salida de un proveedor rompería el test cuando responde el otro, que es
  camino normal). Se testean propiedades del sistema, no salidas concretas del modelo.

Añadir también, en la sección de variables de entorno / decisiones, la entrada de `LLM_EVAL_MODE`
(qué activa, el guardarraíl de producción, que pertenece a la suite de evaluación).

Recordatorio de disciplina de escritura de `CLAUDE.md`: reescribir el estado presente en su sitio,
no añadir entrada de diario. El histórico va a `CHANGELOG.md`.

---

## 7. Tests de contrato a escribir (`tests/app/`, familia 1, mockeados)

- `LLM_EVAL_MODE=true` → la cache hace bypass (ni `get` ni `set`), incluso para una petición
  single-turn que normalmente sería cache-elegible.
- `LLM_EVAL_MODE=false` (default) → comportamiento de cache intacto (regresión).
- Composición de bypasses: `LLM_EVAL_MODE=true` **o** historial no vacío → bypass; solo
  `eval=false` **y** historial vacío → elegible.
- Guardarraíl: construir `Settings` con `LLM_EVAL_MODE=true` + `APP_ENV=production` → **falla al
  arrancar** (error de configuración). Con `APP_ENV=development` → arranca.

Todos deterministas, mockeados, sin red. Entran en `uv run pytest`.

---

## 8. UAT propuesto (CC entrega el guion; lo ejecuta el usuario)

1. Arrancar el backend con `LLM_EVAL_MODE=false` (normal) → una pregunta repetida da `cache_hit:
   true` en la segunda llamada (cache activa).
2. Arrancar con `LLM_EVAL_MODE=true` (en entorno no-producción) → la misma pregunta repetida da
   `cache_hit: false` siempre (cache desactivada, cada llamada va al modelo).
3. Intentar arrancar con `LLM_EVAL_MODE=true` y `APP_ENV=production` → el backend **no arranca**,
   error de configuración legible.

---

## 9. Convenciones (recordatorio)

- Type hints, docstrings (Google, inglés), comentarios inline en español (QUÉ + POR QUÉ; explicar
  `model_validator` de Pydantic la primera vez que aparezca si no está ya explicado).
- Nada de `print()` / `except: pass`. 100 chars. Nombres en inglés.
- Al cerrar: `SNN-decisions.md`, derivar `CHANGELOG.md`, actualizar `CLAUDE.md` (§6) reescribiendo
  estado presente. Entregar guion de UAT (§8).
