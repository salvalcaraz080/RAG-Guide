# Fix-brief F3 + F4 — Smalls de cierre

**Últimos del ciclo** (F1 ✓ → F2 ✓ → **F3+F4**). Dos bloques independientes, **encadenables como
commits separados**. Cierran los seis hallazgos triviales del Paso D. Van sobre el árbol **con F2 ya
commiteado**; ninguno pisa terreno de F2 (líneas y ficheros distintos).

**Insumo:** `pasoD-diagnostic-report.md`. **Naturaleza:** fix-brief, CC implementa + verifica +
docs. No commitea. Todos son correctitud latente barata o limpieza — sin re-embed, sin tocar
identidad de chunk.

---

## Bloque F3 — capa app

### T1-5 · Separar culpa de cliente y de servidor en `RetrievalError`

**Ubicación:** `app/routers/search.py:31-33` · `app/services/retrieval.py:93-97`

Hoy un filtro con valor imposible (culpa del cliente) y «la query se embebió a N dims y la columna es
`vector(1536)`» (misconfiguración del servidor) salen ambos como **400**. La segunda es la guarda
puesta «en el sitio del error» para que un cambio de modelo no falle en silencio — pero reportada
como 400, monitorización la cuenta como petición malformada y nadie mira la config.

**Fix:** distinguir las dos causas (subclase de `RetrievalError` o flag) para que el router mapee el
**desajuste de dimensiones → 500** (culpa de servidor) y el **filtro imposible → 400** (culpa de
cliente). `EmbeddingCredentialsMissing` ya está bien en **503**, no tocar.

**Verificación:** un test por rama — dim-mismatch devuelve 500, filtro imposible 400.

### T1-6 · Desempate de RRF con `or` sobre un entero

**Ubicación:** `app/services/retrieval.py:260`

```python
key=lambda h: (-h["rrf_score"], h["vector_rank"] or 10**6, h["chunk_id"]),
```

`0 or 10**6 → 10**6`: hoy inerte porque los rangos vienen de `enumerate(hits, start=1)`, pero si
alguien pasa a 0-based, **el mejor hit vectorial se desempata como el peor** y el resultado sigue
siendo plausible. Defecto latente enmascarado por una precondición accidental (nodo Phase 2.5):
correcto por `start=1`, no por construcción.

**Fix:** la intención es «None → al final», que se escribe explícita:
```python
h["vector_rank"] if h["vector_rank"] is not None else 10**6
```

**Verificación:** un test con un hit de `vector_rank = 0` que hoy se desempataría como el peor y tras
el fix se ordena en su sitio.

---

## Bloque F4 — corpus y config

### T1-7 · `production_status` por prefijo de string + `break`

**Ubicación:** `corpus/scraper.py:362`

`ECSS-E-ST-10-02C` casa con la entrada `ECSS-E-ST-10` y hereda su estado; el `break` toma la primera
coincidencia en orden de iteración del dict, no la más específica. Contradice el principio ya graduado
en `CLAUDE.md`: *«nunca matching difuso — `ECSS-E-ST-10` no resuelve a `ECSS-E-ST-10-02C`»*. Latente:
`production_status` se escribe al inventario y **nadie aguas abajo lo lee** (verificado), así que se
arregla por higiene de principio, no por efecto presente.

**Fix:** resolver al match **exacto / más específico**, nunca a un prefijo más corto. **No confundir**
con `classify_software_relevance` (`:205`), cuyo tipo de regla se llama `number_prefix` y ahí el
prefijo es la intención — esa no se toca.

**Verificación:** un test en que `ECSS-E-ST-10-02C` **no** hereda el estado de `ECSS-E-ST-10`. No hace
falta re-scrapear (el campo no se lee).

### T1-9 · Quitar `EMBEDDING_PROVIDER` (muerta)

**Ubicación:** `.env`, `.env.example`, `.claude/CLAUDE.md`

Decidido: quitar, por simetría con `LLM_PROVIDER` (su gemela exacta, que C1 ya retiró). Nadie la lee
(verificado sobre `app/`, `corpus/`, `frontend/`). Sacarla de `.env` local y de `.env.example`
(commiteado), y **retirar de `CLAUDE.md` la línea que la reconocía** («hoy no la lee nadie»). Cuando
se construya el seam de provider de embedding, vuelve con lector — misma lógica que aplicó C1.

### T1-10 · `prose_chars` es código muerto

**Ubicación:** `corpus/report.py:72-74`

`@property` de `DocumentResult` sin un solo uso en el repo (verificado). El único código muerto real
que el audit encontró. **Fix:** quitarla.

### T1-11 · `zip()` sin `strict=` en el chequeo de sobredimensión de `embed_all`

**Ubicación:** `corpus/embed_all.py:75` (`for cid, text in zip(self.chunk_ids, self.texts)`)

Es el chequeo cuyo docstring promete que ningún texto se trunca en silencio; si las dos listas
divergieran en longitud, `zip` truncaría **el propio chequeo** al más corto. Hoy se construyen juntas
→ hardening, no bug, pero es el stage de más riesgo. **Fix:** `strict=True`.

**No tocar** los otros dos `zip` que ruff señala: `chunking.py:277` es el idiom pairwise
(`zip(pieces, pieces[1:])`, longitudes distintas a propósito) y `diff_reparse.py:102` está guardado
por un check de longitud tres líneas antes. El audit los verificó como no-defectos.

---

## Docs y cierre

- `.claude/CLAUDE.md`: retirar la línea de `EMBEDDING_PROVIDER`; anotar los fixes de F3/F4 donde
  corresponda (los latentes que se cerraron).
- `CHANGELOG.md`: entradas de F3 y F4 (separadas si van en commits separados).
- `sessions/Auditoria pre M4/F3-F4-decisions.md`: log crudo etiquetado agnóstico/dominio.

**Fuera de alcance:** los cuatro items de Tier 2 (T2-1 build_rows, T2-2 doble check_guards, T2-3
herencia de Settings, T2-4 predicado aguas arriba) siguen diferidos como juicio/documentados — F3/F4
no los tocan. Ningún re-embed, ningún cambio de identidad de chunk.
