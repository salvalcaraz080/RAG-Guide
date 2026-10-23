# S12 · Decisiones — Brief 5 (mini): `xref` encendido por defecto y punteros por título

> Decisiones tomadas al implementar `S12-brief-5.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Rama `s12-agentes`. Sin llamadas reales.

## 1. `RETRIEVAL_XREF=clause` por defecto — excepción por prioridad de producto

- **X9.** El default pasa de `off` a **`clause`** en `app/config.py`, `.env.example` y el `.env`
  local, sobre `bounded` y con los topes de 5 cláusulas y 600 / 2.000 tokens. Test ajustado: el
  que fijaba el default (`test_the_default_is_off` → `test_the_default_is_clause`). La equivalencia
  byte a byte de X8 se sigue probando con `RETRIEVAL_XREF=off` explícito
  (`test_off_is_the_current_path_byte_for_byte`), en verde.
- **X10 — [DOMINIO] la excepción.** Registrada como **prioridad de producto**, no como relectura del
  criterio: el criterio del brief 2 queda como está. `xref` falló la guarda 1 (2 de 6 aspectos con
  cita literal; 4 de 6 por `[n]`) y la 2 (recall −0,08, sobre todo por inflación del denominador), y
  `xref+bounded` pasó el techo de latencia (+30 % frente a +20 %). Se enciende porque, apagada, el
  sistema responde desde handbooks informativos que citan por número la normativa sin traerla. En
  q09, con `clause`, la respuesta cita Q-ST-80C 6.2.7.3 en 3 de 3 ejecuciones. **La latencia pasa de
  guarda a objetivo** (decisión de Salva): la calidad de respuesta va antes, y la latencia se ataca
  por otras vías (indicador de progreso en la UI, razonamiento `low`, caché). Registrada en
  `ARCHITECTURE.md` §5, junto a las otras dos excepciones (precisión de Haiku 5.5 y `bounded`).
- **X11 — referencia sin pasada nueva.** La configuración `xref+bounded` de
  `generation_20261019-224307` (`compare.py --baseline-config xref+bounded`), mismo mecanismo que
  E3. Cifras: recall 0,86, precisión 0,88, `verified` 1,00, citadas / inyectadas 0,47, mediana
  16,8 s (p95 24,0 s), 0,0027 USD por turno. El bullet de latencia de referencia de `CLAUDE.md`
  pasa a esas cifras.
- **Etiqueta en la UI — hecha, sin tocar el contrato.** La expansión de cláusula solo completa
  cláusulas que el retrieval entregó, así que una cláusula **sin ningún átomo recuperado** solo
  puede haber llegado por referencia. `ClauseEvidence.by_reference` (función pura en
  `frontend/presentation.py`) lo decide al pintar: sus átomos llevan «➕ added by cross-reference» y
  la cláusula «➕ by cross-reference». El resto sigue con «➕ added to complete the clause». El
  contrato (`origin`) no cambia: X6 se mantiene. Test en `tests/frontend`.

## 2. Verificación: puntero por título sin identificador (V11) — [AGNÓSTICO]

- `Pointer.points_to_doc` admite `null`, **solo** para un puntero que no nombra ningún documento
  con identificador (V11: «SAVOIR document [Flight Computer Initialisation Sequence]»). Un puntero
  así es siempre `inferred` (`is_literal` devuelve falso sin leer la cita), no da B1, y `check` lo
  lista como `untitled (…)`.
- **Sin subir la versión:** es una relajación que no cambia el significado de ninguna respuesta
  válida en 1.2. `protocol.md` lo registra como «1.2, correction of 2026-10-21», e indica qué poner
  en ese caso. Workspace refrescado.
- **`check V11` sobre la respuesta existente**, sin rehacer la sesión: válida, 22/22 citas literales,
  lectura `B2?`, el puntero SAVOIR como `untitled` inferido. Añadida a la tabla de
  `S12-decisions-4.md` §4 (B4 por rango medido).

## 3. Tests

**1.417 → 1.419** (+2: `by_reference` en el frontend; puntero sin documento válido e inferido).
Modificado 1: el del default de `xref`. Borrados: 0.
