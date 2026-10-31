# S14 — Brief 2: adopción de los huecos tipados y filtro de degradaciones en la UI

> Requiere el brief 1 cerrado (`S14-decisions.md`, `S14-uat.md` §6). Rama `s14-multiagente`.
> Los commits los hace Salva. **Sin llamadas al proveedor**: lo que se adopta ya está medido.
>
> **Regla del brief:** solo lo que dice §1. No se toca prompt, esquema, validación, composición,
> retrieval ni contrato. Si algo más pareciera necesario: `TODO: pendiente de decisión en Claude
> Chat` y se reporta.

---

## 0. Contexto

Juicio de producto del chat (`S14-uat.md` §6) sobre la pasada `generation_20261024-140241`:
- **`gaps` cumple el criterio preregistrado** (1–4) y la precisión de tipados servidos es 16/17, con
  0 premisas espurias. El error es asimétrico: un tipado que falta devuelve el comportamiento de
  antes; uno espurio dice algo falso. Se adopta.
- **El desplegable de degradaciones sale en 13 de 14 consultas**, casi siempre por
  `expansion_truncated` / `xref_truncated`, que son la política `bounded` configurada (misma categoría
  que la ventana de historial, que el brief 1 ya excluyó). Un aviso casi universal se ignora.

---

## 1. Decisiones que este brief implementa

| # | Decisión | Porqué |
|---|---|---|
| D1 | `GENERATION_GAP_KINDS` **default `on`** (config, `.env.example`, docs). `off` sigue existiendo | Criterio cumplido y juicio de producto favorable. `off` queda para comparar |
| D2 | **Nueva referencia del arnés:** la configuración `gaps` de `generation_20261024-140241`. La anterior (`xref+bounded` de `generation_20261019-224307`) queda como histórica | El default cambia; la referencia sigue al default. No hace falta pasada nueva: ya está medida |
| D3 | **UI:** el desplegable «⚙️ Technical limits» muestra solo `xref_over_limit`, `routing_filter_empty`, `moderation_unavailable` y `table_labels_unavailable`. Si no hay ninguno de esos, no hay desplegable. El recuento `(n)` cuenta solo los mostrados | Solo lo que cambia cuánto fiarse de la respuesta. Las truncaciones ya se ven en los átomos marcados ➕ |
| D4 | **El contrato no cambia:** `degradations` sigue llevando todos los tipos, incluidos los truncados | El arnés y cualquier cliente siguen viéndolo todo; el filtro es de presentación |
| D5 | La lista de tipos mostrados vive en **un solo sitio** de la capa de presentación, no repartida por la UI | Cambiarla es una línea |

---

## 2. Trabajo

- D1–D2: config, `.env.example`, `compare.py`/arnés si tienen la referencia cableada, y docs.
- D3–D5: presentación (`degradation_notes` o donde viva el filtro). Ningún cambio en `app/` fuera de
  presentación.
- Docs: `ARCHITECTURE.md` (default nuevo, referencia nueva, regla del desplegable) y
  `.claude/CLAUDE.md` si aplica. `CHANGELOG.md` derivado.

---

## 3. Tests (sin red)

- `uv run pytest` en verde. Recuento frente a 1533.
- Default `on`; con `off`, la equivalencia byte a byte del brief 1 (T8) sigue valiendo.
- Presentación: solo truncados → sin desplegable; un tipo mostrado + truncados → desplegable con
  `(1)`; los cuatro tipos mostrados se pintan; `degradations` del contrato intacta en todos los casos.

---

## 4. UAT (usuario: render del desplegable)

El render de los bloques ❗ y ↪️ ya lo validó Salva (2026-10-30, con el brief 1): OK. Queda solo el
desplegable con el filtro nuevo, **con el default nuevo, sin tocar `.env`**:
1. q14, turno 3 (la de OBCP del brief 1): desplegable presente **solo si** hay `xref_over_limit`
   (lo hubo en la pasada), con esa línea y sin las de truncado.
2. q02 (no conformidades): sin desplegable.
3. Una respuesta con solo truncados (cualquiera de q04, q08, q13 que no tenga `xref_over_limit`):
   sin desplegable.

## 5. Cierre

- `S14-decisions-2.md` (breve: qué cambió, recuento de tests, etiquetado agnóstico/dominio).
- `S14-uat-2.md` con el script; el juicio de render lo pone Salva.
