# S13 · Decisiones — Brief 3 (mini): traza de punteros agrupada por documento

> Decisiones tomadas al implementar `S13-brief-3.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Rama `s13-orquestacion-agentes`. Ninguna llamada al modelo; un solo embedding, en la
> comprobación del §4.

## 1. Qué se agrupa (T1, T2) — [AGNÓSTICO]

- **La agrupación vive en la presentación** (`XrefReport.as_dict`, función `_grouped`). Las listas
  internas del informe y la selección (`candidates`, `fill`) no cambian, así que nada de lo que
  decide el contexto puede moverse.
- **Se agrupan las entradas sin efecto en el contexto:**
  - con `clause`, las `unresolved` con motivo `whole_standard`;
  - con `clause_standard`, las `standards` con estado `already_searched`, `out_of_corpus` o
    `ambiguous`.

  Clave: estado y documento. El documento resuelto cuando existe; si no, el nombre normalizado
  con `standard_name`, extraído de `normalize_standard` sin cambiar su comportamiento. Así
  `[ECSS-Q-30]` y `ECSS-Q-ST-30` son una sola entrada.
- **Forma de una entrada agrupada:** `text` y `source` de la primera mención, y `mentions`. Ocupa
  la posición de la primera mención.
  - Las de `standards` agrupadas pierden `chunks`, `tokens` y `truncated`, que en ellas siempre
    eran vacíos o cero.
  - Las de `unresolved` conservan sus cuatro campos.
- **No se agrupan (T2):** `followed`, `over_limit`, `already_injected` y `no_hits` (con
  `truncated` dentro), y todas las entradas de cláusula, incluidas las `out_of_corpus` y
  `ambiguous` de una referencia a cláusula («ECSS-Q-ST-30 clause 5.4»). Esas siguen siendo una
  por mención.
- **Lectura del brief:** «`out_of_corpus`, `ambiguous`» de T1 se entiende como los estados de
  puntero a estándar, porque T2 dice «todas las de cláusula» no cambian. Si el chat quería
  agrupar también las de cláusula fuera del corpus, es un cambio de una línea.

## 2. El contexto no cambia (T3)

Digests del contexto en los tres modos, tomados **antes** de tocar el código, sobre un texto con
punteros a cláusula y a estándar:

| Modo | Digest |
|---|---|
| `off` | `2dd01073…` |
| `clause` | `e343b922…` (el mismo de W7) |
| `clause_standard` | `e343b922…` |

- Siguen en verde, igual que los digests del array de mensajes de `test_history_equivalence`.
- El digest de la **traza** con `clause` (W7, `edd4b96b…`) cambia **a propósito**: T1 agrupa
  `whole_standard`. Su test pasa a afirmar la traza explícita.

## 3. El arnés puntúa igual

- `aspects.py` solo lee `doc` y `status` de `standards`, y un `target` que en `whole_standard`
  siempre es `None`. Agrupar conserva el conjunto de estados por documento, así que el «por qué
  no» no puede cambiar.
- **Re-puntuación sobre el artefacto real** (`generation_20261023-191136`): convertidas sus trazas
  a la forma agrupada con la misma regla, `aspects.score` da **exactamente** el mismo resultado,
  con 1.200 → 992 entradas de traza. Lo hace CC, no un test, porque la suite no lee `data/`. Un
  test de contrato fija lo mismo sobre un artefacto sintético.
- **El peor caso del artefacto:** q08 con `clause` (`xref+bounded`) pasa de 26 a **8** entradas
  `whole_standard` por ejecución; q09, de 26 a **4**.

## 4. Comprobación de la traza del caso 3 de la UAT 2 — [DOMINIO]

La consulta «What does ECSS-Q-ST-80C require for software criticality classification?», con
`clause_standard` y solo retrieval (proveedor del arnés, 1 embedding, sin modelo), en el log
`references_expanded`:

- **`standards`, 4 entradas:**
  1. `ECSS-Q-ST-10-04`, `out_of_corpus`, `mentions=1`;
  2. M-ST-80C `followed` con 7 chunks (rangos 1–7, 478 tokens, `truncated`);
  3. Q-HB-80-03A `followed` con 0 chunks (`truncated`);
  4. Q-ST-30-02C `over_limit`.
- **`unresolved`, 9 entradas.** Seis son «ECSS-Q-ST-30 clause 5.4» `out_of_corpus`: referencias
  a cláusula, que T2 deja como están. Las otras son una `own_clause` y una `already_injected`.

**Corrección a la UAT 2.** Las más de 60 entradas `already_searched` de «ECSS-Q-ST-80» que motivaron
este brief **no eran de esta consulta**: su traza no tiene ninguna. CC las atribuyó a ella en la
UAT 2, y está corregido en `S13-uat-2.md`. Venían de otra petición de la misma UAT, que ya no se
puede identificar: los logs se perdieron al recrear `api`, y el log no guarda la pregunta. El
fenómeno es real y se reproduce en el artefacto (§3).

**Lo que queda repetido, para el chat:** la repetición que sobrevive está en las referencias a
cláusula fuera del corpus. Es la decisión de T2, y aquí suma 6 entradas sobre 9.

## 5. Tests

**1.470 → 1.477** (+7).

- **Modificado:** el test de W7, que pasa a ser paramétrico. Los digests del contexto en los tres
  modos dan +2.
- **Nuevos:**
  - la traza con `clause` agrupa solo `whole_standard`;
  - varias menciones de un documento son una entrada con `mentions`, incluida la forma corta;
  - con `clause` también se agrupa;
  - las entradas que deciden conservan su forma;
  - en `aspects`, la traza agrupada puntúa igual.

## 6. Etiquetado

- **[AGNÓSTICO]:**
  - agrupar en la presentación, nunca en la lógica que decide;
  - agrupar solo lo que no tiene efecto;
  - fijar el contexto por digest antes de tocar una traza;
  - comprobar que el instrumento que lee la traza puntúa igual.
- **[DOMINIO]:** la forma corta de los handbooks (`[ECSS-Q-30]`) como una misma entrada, y los
  casos de q08, q09 y Q-ST-30.
