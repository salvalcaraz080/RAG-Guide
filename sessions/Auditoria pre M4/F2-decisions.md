# F2 — Log de decisiones (embedding async nativo + aislamiento del cliente en tests)

Log crudo de qué se cambió y por qué. Etiquetado **agnóstico** / **dominio**.

**Estado:** las dos tareas cerradas y verificadas. `uv run pytest` → **894 passed, 8 deselected**
(desde 891). **No se re-embebió nada:** `is_stale()` = `False`, los tres fingerprints intactos.
Árbol listo, **sin commitear**.

**Cierra:** T1-2 y T1-4 del `pasoD-diagnostic-report.md`.

---

## 1 · Qué mecanismo async se eligió, y por qué

**[agnóstico] Async NATIVO (`AsyncOpenAI`), no el fallback `to_thread`.**

El brief admitía `to_thread` «solo si la reestructuración nativa resulta invasiva contra el código
real». Se midió lo invasivo antes de decidir: el seam tiene **cuatro** llamantes
(`retrieval.embed_query`, `corpus/embed_all.generate`, `diagnostics/embedding_sanity.run`, y los
tests). Ninguno tenía lógica que reestructurar — solo `await` y, en los dos batch, un `asyncio.run`
en su entrada. Eso no es invasivo, y a cambio:

- Cierra la inconsistencia **de raíz**. Los otros dos seams de proveedor ya eran async
  (`input_guards` con `AsyncOpenAI`, `llm_wrapper` con `router.acompletion`, cuyo docstring dice
  «hard requirement, not optional»). `to_thread` habría dejado el seam síncrono envuelto, es decir
  la misma inconsistencia con una tirita.
- Es lo mismo que señala T1-4 desde el lado de los tests: un único cliente, un único patrón.

**[dominio] El batch se adapta, no al revés.** `corpus/embed_all.py` y
`corpus/diagnostics/embedding_sanity.py` abren su propio loop con `asyncio.run` y lo cierran con
un `aclose_client()` nuevo en el seam. Razón concreta: `asyncio.run` cierra su loop al terminar, y
un cliente HTTP async que sobreviva a su loop deja conexiones colgando y avisos de «Event loop is
closed». El servidor no lo necesita —su loop vive lo que vive el proceso—, así que la función es
para quien abre y cierra un loop, y el docstring lo dice.

**[dominio] Los lotes siguen en serie y en orden, y se documentó por qué.** Con `async` la
tentación inmediata es `asyncio.gather` sobre los 109 lotes. **No se hizo:** el orden de las filas
**ES** el mapeo fila→chunk que declara el sidecar, así que paralelizarlos no sería acelerar el
batch, sería cambiar la identidad del artefacto — el fallo «bien formado y falso» que la guarda 4
de la ingesta existe para cazar. Queda anotado en el docstring de `generate` para que nadie lo
«optimice» sin ver la consecuencia.

**[dominio] Fichero fuera del alcance del brief que hubo que tocar:**
`corpus/diagnostics/embedding_sanity.py` llamaba al seam y no estaba listado. Hacer el seam async
obliga a adaptar a **todos** sus llamantes; dejarlo habría roto el diagnóstico.

---

## 2 · Tarea 2 — la fixture envenena el hueco, no lo rellena

**[agnóstico] Un doble por defecto habría sido el error.** El brief lo vio: «la forma correcta no
es solo inyectar un fake por defecto, sino que un cliente NO inyectado falle de forma ruidosa».
Un doble silencioso deja pasar al test olvidadizo —justo el caso que T1-4 describe— y el problema
sigue ahí, ahora enmascarado. La fixture pone en `_client` un objeto cuyo `__getattr__` lanza
nombrando el problema.

**[agnóstico] Va en el conftest RAÍZ, no en `tests/app/`.** El brief decía `tests/app/conftest.py`,
donde viven las otras dos. Pero el riesgo es igual —o mayor— en `tests/corpus/`, que es donde vive
el stage de embeddings del corpus. En la raíz cubre las dos suites.

**[agnóstico] Envenenar en vez de sustituir tiene una propiedad que un doble no tiene:** los tests
que legítimamente necesitan otra cosa **ya funcionaban así**.
`test_missing_credentials_fail_by_name` monkeypatchea `_client = None` a propósito para ejercitar
la construcción real, y su monkeypatch simplemente sobrescribe el de la fixture. Cero fricción.

**Verificado:** con la fixture puesta, los 894 siguen verdes — o sea, los ~6 puntos de inyección
existentes la respetan — y hay un test que comprueba que un llamante sin `client=` se caza.

---

## 3 · El test que casi meto mal (y por qué importa)

**[agnóstico] Primera versión del test de latido: no discriminaba.** Contaba el total de latidos
tras `asyncio.gather(search(), heartbeat())` y asertaba `ticks >= 5`. Lo probé contra un cliente
síncrono simulado antes de darlo por bueno:

```
cliente ASYNC    -> latidos: 20
cliente SINCRONO -> latidos: 20   <- el test pasa igual
```

**El total no mide nada**: el latido acaba llegando a su cuenta final aunque el loop se bloquee,
solo que más tarde. Habría sido un verde que no ejercita — exactamente el defecto que F2 viene a
arreglar, reproducido en su propio test.

**Corrección:** contar los latidos ocurridos **mientras** la llamada está en vuelo (latido como
tarea de fondo, snapshot al volver de `search`, cancelar). Re-probado:

```
cliente ASYNC    -> latidos DURANTE el embedding: 19
cliente SINCRONO -> latidos DURANTE el embedding:  0   <- inanición
```

Ahora sí muerde. Queda el porqué escrito en el docstring del test, porque la versión ingenua es la
que a cualquiera se le ocurre primero.

---

## 4 · Medición antes/después de la sonda `/health`

**Nota honesta sobre la comparación:** la sonda del Paso D fue **una sola muestra** y su baseline
(22,7 ms) estaba inflado. Aquí se muestrea, y además se añade una medida que no depende del
baseline en absoluto.

| | Paso D (bloqueante) | F2 (async) |
|---|---|---|
| `/health` en reposo | 22,7 ms (1 muestra) | **p50 2,9 ms · max 11,6 ms** (50 muestras) |
| `/health` durante un `/search` | **237,4 ms** (1 muestra) | **p50 7,9 ms · max 14,4 ms** (13 muestras) |
| Peticiones servidas durante un `/search` | ninguna hasta que volvía el embedding | **13** |

**El número que no admite interpretación:** durante un solo `/search` se sirvieron **13**
peticiones `/health`, con un techo (14,4 ms) prácticamente igual al de reposo (11,6 ms). Con el
cliente síncrono no se servía ninguna: el loop estaba parado.

**La medida independiente del baseline — concurrencia:**

```
1 /search (mediana de 3)  :  246,6 ms
4 concurrentes            :  556,1 ms   (serializado: ~986,5 ms)  -> 1,8x
8 concurrentes            :  652,0 ms   (serializado: ~1.972,9 ms) -> 3,0x
```

Ocho búsquedas cuestan 652 ms en vez de ~1.973 ms. Y el reparto **encaja con el modelo esperado**:
si el embedding (~180 ms) solapa y el seq scan de la base (~65 ms, según `ARCHITECTURE.md`)
serializa, lo previsto es `180 + 8×65 ≈ 700 ms`. Medido: 652 ms. Es decir, **lo que solapa es
exactamente la llamada de red que antes bloqueaba**, y lo que queda serializando es trabajo real de
la base.

---

## 5 · Identidad del batch — la prueba de que no se tocó nada

| | Valor | |
|---|---|---|
| `stage_fingerprint` recalculado | `c54e2f865e3fb1a6` | = el del sidecar ✅ |
| `chunks_fingerprint` (sidecar) | `ab3fce8b5cb1a0d4` | = `input_fingerprint` de `chunks.json` ✅ |
| `is_stale()` | `False` | no hay que re-embeber ✅ |
| `embed_all --dry-run` | 13.876 chunks, 109 requests, mismo fingerprint | ✅ |

---

## 6 · Lo que F2 NO cierra

**[agnóstico] Siguen existiendo dos fuentes de settings** (`Settings` y `EmbeddingSettings`), y es
deliberado: un build offline no debe exigir `ANTHROPIC_API_KEY`. F2 cierra solo **la mitad** de
T1-4 —la fixture—; unificarlas sigue diferido como juicio (T2-3), tal y como el brief marcaba
fuera de alcance.

---

## 7 · Ficheros tocados

**Código** — `app/services/embedding.py` (async nativo, `AsyncOpenAI`, `aclose_client`),
`app/services/retrieval.py` (`embed_query` async, awaited en `search`), `corpus/embed_all.py`
(`generate` async + `asyncio.run` en la entrada), `corpus/diagnostics/embedding_sanity.py` (ídem;
**no estaba en el alcance del brief**, pero llamaba al seam).

**Tests** — `tests/conftest.py` (fixture `isolate_embedding_client`),
`tests/app/test_search.py` (dobles async, dos tests de comportamiento async, test de la fixture),
`tests/corpus/test_embeddings.py` (dobles async, helpers y tests convertidos).

**Docs** — `.claude/CLAUDE.md` (seam async nativo en decisiones consolidadas; fixture documentada
en § Tests como la tercera de la familia), `CHANGELOG.md` (entrada `[0.34.0]`).

**Sin commitear.** Mensaje propuesto en la respuesta de CC.
