# S08 · Brief 2 — Propagación de `requirement_uid` al chunk (corpus)

> **Producido por:** Claude Chat (decisión). **Ejecuta:** Claude Code.
> **Espacio:** `corpus/` (dominio) — regenera el artefacto de S07, aguas arriba de la ingesta.
> **Alcance:** SOLO propagar `Requirement.uid` al `Chunk` y regenerar `chunks.json`. **No** ingesta,
> **no** endpoint, **no** cambios de schema (la columna `chunks.requirement_uid` ya existe del
> Brief 1), y **NO se re-embebe**.
>
> **Renumeración:** este es el Brief 2. La ingesta (que `S08-brief-1-decisions.md` llamaba "brief 2")
> pasa a **Brief 3**; `/search` a **Brief 4**.

---

## 0. Por qué, y por qué ahora

El schema del Brief 1 declaró `chunks.requirement_uid TEXT NULL` pero el `Chunk` de S07 no lleva
el campo (marcado en `S08-brief-1-decisions.md` §1.1). Decisión de producto: **lo poblamos ya** —
se necesitará citación a nivel de requisito. Poblarlo **antes** de la ingesta evita una segunda
pasada (ingestar NULLs y re-`UPDATE` después). Vía elegida: **(a)** propagar desde `Requirement.uid`,
NO **(b)** parsear el sufijo del `chunk_id`. Mismo criterio que separó `embed_text` de `cite_text`:
un dato de identidad se transporta desde el origen, no se re-deriva de una cadena — y `::r0` es
ordinal posicional, no un UID, así que (b) tiene una trampa que (a) no tiene.

---

## 1. La propiedad de seguridad — `embed_text` invariante, CERO re-embed

`requirement_uid` es identidad; **no forma parte de `embed_text`** (el texto que consumió el
embedder = header contextual + verbatim). Por tanto, tras regenerar `chunks.json`:

- **`embed_text` debe quedar byte-idéntico para los 13.876 chunks.** Esa es la garantía de que
  los vectores de `embeddings.npy` siguen siendo los correctos.
- **NO se ejecuta el embedder.** Si te encuentras re-embebiendo, para: `embed_text` no cambió, los
  vectores no cambian, re-embeber solo gasta dinero e introduce no-determinismo. `embeddings.npy` y
  su sidecar **no se tocan** (salvo, quizá, el valor del fingerprint — ver §2).

**Criterio de aceptación central:** diff de `embed_text` viejo↔nuevo sobre los 13.876 chunks = 0
diferencias. Si aparece **una sola**, la propagación tocó algo que no debía; para y repórtalo.

---

## 2. Fingerprint — verifícalo primero, luego actúa (paso 0)

`chunks.json` y el sidecar comparten `chunks_fingerprint = ab3fce8b5cb1a0d4`. **Antes de tocar
nada, lee la función de fingerprint** y determina qué digiere:

- **Rama A — digiere `embed_text` (o el contenido consumido por el embedder):** añadir `uid` no lo
  mueve. Regeneras `chunks.json`, el fingerprint recalculado sigue siendo `ab3fce8b5cb1a0d4`,
  empareja con el sidecar sin tocarlo. Fin. Es la rama esperada por el principio de S07.
- **Rama B — digiere el registro completo del chunk:** añadir un campo cambia el valor → mismatch
  con el sidecar. Esto **no es un problema de este brief, es una señal de que el fingerprint está
  mal escopado**: su trabajo es responder "¿debo re-embeber?", y para eso debe digerir exactamente
  lo que el embedder consume (`embed_text`), no la identidad. En esta rama la corrección es
  **estrechar el fingerprint a `embed_text`** — así una edición de identidad futura (otra columna,
  otro campo) nunca vuelve a disparar un mismatch espurio. Tras estrecharlo, el valor sobre
  `embed_text` invariante coincide con el del sidecar y todo empareja.
  - Documenta el cambio de escopado en `ARCHITECTURE.md` (es un invariante del pipeline).
  - **No** re-estampes el sidecar con un valor nuevo derivado del registro completo: eso perpetúa
    el mal escopado. Estrecha la función; no parchees el dato.

En ambas ramas el resultado final es el mismo: sidecar intacto en sus vectores, fingerprint que
vuelve a emparejar, cero re-embed.

---

## 3. Implementación

1. Propaga `Requirement.uid` al `Chunk` en el chunker (`corpus/chunking.py` o donde se construya
   el `Chunk`). El dato existe en `Requirement.uid`; el chunker no lo propagaba.
2. **Regla de poblado, estricta:**
   - Solo chunks `chunk_type == 'requirement'` pueden llevar `requirement_uid`.
   - `requirement_uid = Requirement.uid` **si existe, si no NULL/ausente**. **Nunca** el ordinal
     `r0`/`::rN`. Leer de `Requirement.uid` da NULL de forma natural cuando no hay UID — no
     re-derives del `chunk_id`.
   - Los chunks no-requirement (section/table/note/glossary_term) **no** llevan `requirement_uid`.
3. Regenera `chunks.json`.
4. `chunk_id` **no cambia** — el UID ya vivía en su sufijo; solo lo estamos elevando a campo. La
   clave natural del upsert (Brief 3) sigue intacta.

---

## 4. Verificación (pytest, sin DB — está a tu alcance)

- **`embed_text` byte-idéntico** viejo↔nuevo sobre los 13.876 chunks (el criterio de §1). 0 diffs.
- **Cobertura del UID:** exactamente **2.536** requisitos con `requirement_uid` no-nulo de 7.922
  totales; los otros 5.386 en NULL. (Cifras de `S08-brief-1-decisions.md` §1.1 — confírmalas
  medidas, no las des por buenas.)
- **Cero fuga de ordinal:** ningún `requirement_uid` con forma `r<n>`/`::r…`.
- **No-requirement todos NULL:** ningún chunk de las otras cuatro anclas lleva UID.
- **Fingerprint:** comportamiento según la rama de §2 (A: valor idéntico; B: función estrechada y
  valor coincidente sobre `embed_text`). Test que ate el emparejamiento `chunks.json` ↔ sidecar.
- Los conteos de anclas y el split de authority **no cambian** respecto al Brief 1 (regenerar no
  reestructura: requirement 7.922 · section 2.583 · table 1.481 · note 1.410 · glossary_term 480;
  10.320 normative / 3.556 informative).

---

## 5. Qué NO hacer

- **No re-embeber.** No ejecutar el embedder. No tocar `embeddings.npy`.
- No ingestar nada en Postgres (Brief 3).
- No tocar el schema (la columna ya existe) ni el camino de petición.
- No re-derivar el UID del `chunk_id`; no meter el ordinal como UID.
- No re-estampar el sidecar con un fingerprint de registro completo (rama B: estrechar la función,
  no parchear el valor).
- No commitear (redacta el mensaje; commitea Salva).

---

## 6. Docs

- **`ARCHITECTURE.md`:** el `Chunk` ahora transporta `requirement_uid` (identidad de requisito,
  poblada solo donde existe). Si tocaste el escopado del fingerprint (rama B), documéntalo como
  invariante del pipeline: la clave de invalidación digiere `embed_text`, no el registro.
- **`CHANGELOG.md`:** entrada nueva (`[0.28.0]` — nota: `S08-brief-1-decisions.md` §4.7 avisa de
  un `[0.26.0]` duplicado histórico; no lo arregles aquí, solo no repitas número).
- **`CLAUDE.md`:** solo si el estado presente cambia (p. ej. si el escopado del fingerprint pasa a
  ser un invariante que CC deba respetar). Sin narrativa de diario.

---

## 7. Entregables + aceptación

1. `Chunk` propaga `requirement_uid`; `chunks.json` regenerado.
2. `embed_text` invariante probado (0 diffs) → vectores válidos, sin re-embed.
3. Fingerprint emparejando (rama A idéntico / rama B estrechado), con test que lo ata.
4. Cobertura y no-fugas verificadas (§4).
5. `uv run pytest` verde.
6. Docs (§6) + mensaje de commit propuesto.
7. `S08-brief-2-decisions.md` — decisiones crudas etiquetadas agnóstico/dominio, **la rama de
   fingerprint que resultó (A o B) y por qué**, y cualquier sorpresa. Súbelo al chat para la
   extracción (paso 4).
