# F3 + F4 — Log de decisiones (smalls de cierre)

Log crudo de qué se cambió y por qué. Etiquetado **agnóstico** / **dominio**.

**Estado:** los seis hallazgos cerrados y verificados. `uv run pytest` → **905 passed, 8
deselected** (desde 894: +11 tests). Sin re-embed, sin tocar identidad de chunk. Árbol listo,
**sin commitear**, en dos bloques encadenables como commits separados.

**Cierra:** T1-5, T1-6 (F3) · T1-7, T1-9, T1-10, T1-11 (F4). **Con esto termina el ciclo de
saneamiento pre-M4.**

---

## Bloque F3 — capa app

### T1-5 · La culpa la nombra el tipo, no el sitio que lanza

**[agnóstico] Subclase, no flag.** `QuerySpaceMismatch(RetrievalError)`. Se eligió la subclase
sobre un flag por una propiedad concreta: **ningún `except RetrievalError` existente se rompe**.
Lo que cambia es que ahora se *puede* distinguir, no que haya que hacerlo.

**[agnóstico] Y esa misma propiedad crea la trampa, así que hay test.** Al ser subclase, **el
orden de los `except` en el router decide el resultado**: si alguien pone `except RetrievalError`
primero, las dos culpas vuelven a colapsar en 400 y nada falla a gritos.
`test_the_two_causes_do_not_collapse_into_one_code` ejercita las dos ramas en la misma corrida y
asserta `{"cliente": 400, "servidor": 500}` — es un test sobre el *orden*, no sobre cada rama por
separado.

**[dominio] `EmbeddingCredentialsMissing` sigue en 503, sin tocar.** Ya estaba bien: no es culpa
del cliente ni configuración incoherente, es una dependencia ausente.

### T1-6 · El rango 0 es un rango, no una ausencia

**[agnóstico] El fix de una línea no era suficiente: había que hacerlo testeable.** Escribir
`rank if rank is not None else NO_VECTOR_RANK` cierra el defecto, pero el desempate era un
**closure dentro de `fuse_rrf`** y la API pública **nunca produce un rango 0**
(`enumerate(..., start=1)`) — que es exactamente lo que hacía el defecto latente e inalcanzable
desde fuera. Un test contra `fuse_rrf` no puede llegar a él.

Se extrajo a **`rrf_sort_key`, a nivel de módulo**, y el docstring dice por qué está ahí y no
dentro. Así el test le pasa un rango 0 y ejercita **la función de producción**.

**[agnóstico] Iba a escribir un test tautológico y lo corregí a tiempo.** La primera versión
copiaba la lambda del desempate dentro del test y la evaluaba: habría probado la copia contra sí
misma —el mismo defecto que la auditoría del Paso D marcó como «test que afirma falsamente
cubrirlo»—. La extracción a nivel de módulo es lo que permitió testear de verdad.

**[agnóstico] `NO_VECTOR_RANK` como constante nombrada**, no un `10**6` suelto en dos sitios: el
número es «al final del desempate», y eso se dice mejor con un nombre.

---

## Bloque F4 — corpus y config

### T1-7 · Match exacto para el estado de producción

**[dominio] El `startswith` + `break` fallaba dos veces a la vez**, y la segunda es la que no
estaba nombrada en el hallazgo:

1. `ECSS-E-ST-10-02C` heredaba el estado de `ECSS-E-ST-10`.
2. **Cuál de los candidatos ganaba dependía del orden de iteración del dict**, no de cuál era más
   específico.

Verificado sobre la implementación anterior:

```
ECSS-E-ST-10-02C  VIEJO -> 'under revision'   <- hereda del prefijo
ECSS-E-ST-10-02C  NUEVO -> 'published'
¿depende del orden del dict?  VIEJO: True  |  NUEVO: False
```

Hay un test por cada uno de los dos fallos, incluido
`test_the_result_does_not_depend_on_dict_order`.

**[dominio] Se resolvió a match EXACTO, no a «el más específico».** El brief admitía cualquiera de
los dos. Exacto es lo correcto aquí: el estado de producción es una propiedad **del documento**, y
que una familia herede el de su prefijo no es una aproximación razonable, es una afirmación
distinta. «Más específico» habría mantenido viva la idea de herencia.

**[dominio] Extraído a `resolve_production_status`** en vez de arreglarlo en línea: era lo que lo
hacía testeable sin montar una página HTML falsa.

**[dominio] `classify_software_relevance` no se toca**, como pedía el brief: su tipo de regla se
llama literalmente `number_prefix` y ahí el prefijo *es* la intención.

### T1-9 · `EMBEDDING_PROVIDER` fuera

**[agnóstico] Simetría con lo que ya hizo C1.** Gemela exacta de `LLM_PROVIDER`. Retirada de
`.env`, `.env.example` y de la línea de `CLAUDE.md` que la reconocía («sigue en `.env` pero hoy no
la lee nadie»), que era el reconocimiento explícito de que sobraba. Los dos ficheros de entorno
quedan con **las mismas 12 claves**, y `Settings` sigue cargando.

### T1-10 · `prose_chars` fuera

**[agnóstico]** `@property` de `DocumentResult` sin un solo uso en el repo. El único código muerto
real que encontró la auditoría — el resto de candidatos resultaron ser properties en uso, handlers
de FastAPI o llamadas desde el `main()` del módulo.

### T1-11 · `zip(..., strict=True)`

**[agnóstico] Hardening en el sitio de más riesgo.** Es el chequeo cuyo docstring promete que
ningún texto se trunca en silencio: si las dos listas divergieran, `zip` truncaría **el propio
chequeo** al más corto y dejaría de cubrir la cola sin decirlo — una verificación que deja de
verificar. Hoy se construyen juntas, así que es endurecimiento, no bug.

**[agnóstico] Los otros dos `zip` que ruff señala NO se tocan**, como pedía el brief y como
verificó la auditoría: `chunking.py` es el idiom pairwise (`zip(pieces, pieces[1:])`, longitudes
distintas a propósito) y `diff_reparse.py` está guardado por un check de longitud tres líneas
antes.

---

## Verificación

| | Antes | Ahora |
|---|---|---|
| Query fuera del espacio vectorial | **400** (culpa del cliente) | **500** (culpa del servidor) |
| Filtro imposible | 400 | 400 |
| Sin credencial | 503 | 503 |
| Desempate con `vector_rank = 0` | `10**6` → **al final** | `0` → **en su sitio** |
| `ECSS-E-ST-10-02C` con `ECSS-E-ST-10` declarado | hereda «under revision» | **''** (no hereda) |
| ¿el estado depende del orden del dict? | **sí** | no |
| `.env` vs `.env.example` | 13 claves, una muerta | **12, idénticas** |
| Suite | 894 | **905** |

**Los tres fixes de correctitud tienen test que muerde**, comprobado contra la implementación
anterior (no solo contra la nueva). Los otros tres son borrados, y su verificación es que la suite
sigue verde y nadie los referencia.

---

## Ficheros tocados

**F3** — `app/services/retrieval.py` (`QuerySpaceMismatch`, `NO_VECTOR_RANK`, `rrf_sort_key`),
`app/routers/search.py` (mapeo de culpa a código), `tests/app/test_search.py` (+6 tests).

**F4** — `corpus/scraper.py` (`resolve_production_status`), `corpus/report.py` (borrado),
`corpus/embed_all.py` (`strict=True`), `tests/corpus/test_scraper_parsing.py` (+5 tests),
`.env`, `.env.example`.

**Docs** — `.claude/CLAUDE.md` (los latentes cerrados anotados donde corresponde; línea de
`EMBEDDING_PROVIDER` retirada), `CHANGELOG.md` (entradas `[0.35.0]` F3 y `[0.36.0]` F4, separadas
porque los bloques van en commits separados).

**Sin commitear.** Dos mensajes propuestos en la respuesta de CC, uno por bloque.

---

## Estado del triaje del Paso D al cerrar

Los **11 hallazgos Tier 1** están cerrados: T1-1/T1-3/T1-8 (F1), T1-2/T1-4 (F2), T1-5/T1-6 (F3),
T1-7/T1-9/T1-10/T1-11 (F4).

Los **4 de Tier 2** siguen diferidos como juicio, tal y como marcaba el alcance: T2-1
(`build_rows` con 769 MB residentes), T2-2 (doble `check_guards`), T2-3 (herencia de `Settings`),
T2-4 (predicado de marcadores aguas arriba, cuyo test de aceptación ya dejó escrito F1).
