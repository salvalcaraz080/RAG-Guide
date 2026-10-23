# S12 — Brief 5 (mini): `xref` encendido por defecto + esquema de punteros por título

> Mini-brief de cierre. Rama `s12-agentes`. Los commits los hace Salva. Sin llamadas reales.

## 1. `RETRIEVAL_XREF=clause` por defecto (excepción por prioridad de producto)

| # | Decisión | Porqué |
|---|---|---|
| X9 | Default `RETRIEVAL_XREF` pasa de `off` a `clause`, sobre `bounded` (topes actuales: 5 cláusulas, 600 / 2.000 tokens) | Decisión de Salva tras la UAT 4 (V10/q09): con `off`, el sistema responde desde handbooks informativos que citan por número la normativa, sin traerla. Con `clause`, q09-a1 se resuelve 3/3 citando Q-ST-80C 6.2.7.3 |
| X10 | Se registra como **excepción por prioridad de producto**, no como relectura del criterio. El criterio del brief 2 no se reescribe: `xref` falló la guarda 1 (2/6 estricto; 4/6 por `[n]`) y la 2 (recall −0,08, por inflación del denominador), y `xref+bounded` superó el techo de latencia (+30 % frente a +20 %). **Salva rebaja la latencia de guarda a objetivo**: la calidad de respuesta va antes, y la latencia se ataca por otras vías (indicador de progreso en la UI, razonamiento `low`, caché) | Disciplina del preregistro: la excepción se ve como lo que es |
| X11 | La referencia del arnés pasa a ser la configuración `xref+bounded` del artefacto `generation_20261019-224307`, sin pasada nueva (mismo mecanismo que E3 del brief 4) | Ya está medida con el instrumento validado |

Trabajo: cambiar el default; ajustar los tests que fijen `off` como default (no los que prueban `off`
como modo); `ARCHITECTURE.md` y `.claude/CLAUDE.md` (default, porqué, excepción X10);
`CHANGELOG.md`. La equivalencia byte a byte de X8 sigue valiendo con `RETRIEVAL_XREF=off` explícito.

**Etiqueta en la UI:** lo traído por referencia sale con la misma marca ➕ que `orphans`/`bounded`,
con el texto «added to complete the clause», que no describe una referencia. Si cambiar el texto es
trivial (p. ej. «➕ added by cross-reference» para `origin` de referencia), hazlo; si exige tocar el
contrato, `TODO` y regístralo.

## 2. Verificación: puntero por título sin identificador (caso V11)

V11 falla la validación: un puntero nombra un documento externo solo por el título («SAVOIR
document [Flight Computer Initialisation Sequence]») y el verificador puso `points_to_doc: null`,
con razón. El esquema lo exige.

- `points_to_doc` admite `null` **solo** cuando el puntero no nombra ningún documento con
  identificador. Un puntero así es siempre `inferred` (no da B1) y `check` lo lista.
- Es una relajación del esquema que no cambia el significado de ninguna respuesta válida en 1.2:
  **no** se sube la versión del protocolo. Regístralo así (1.2 con la corrección y la fecha).
- Repite `check V11` sobre la respuesta existente (no hace falta rehacer la sesión) y añade su
  lectura a la tabla de `S12-decisions-4.md` §4.

## 3. Tests (sin red)

- `uv run pytest` en verde; recuento frente al cierre del brief 4.
- Default `xref` = `clause`; `off` sigue funcionando y mantiene la equivalencia byte a byte.
- Puntero con `points_to_doc: null` → válido, `inferred`, no B1.

## 4. UAT (usuario)

1. Sin overrides en `.env`: comprobar la configuración efectiva
   (`docker compose exec api python -c "from app.config import get_settings; s = get_settings(); print(s.RETRIEVAL_EXPANSION, s.RETRIEVAL_XREF)"`
   → `bounded clause`).
2. La pregunta de q09 («What do ECSS documents require regarding the reuse of existing or
   off-the-shelf software in a flight project?»): la respuesta cita normativa de Q-ST-80C 6.2.7.x
   además de los handbooks.
3. `check V11` → lectura válida.

## 5. Cierre

`S12-decisions-5.md` y `S12-uat-5.md`.
