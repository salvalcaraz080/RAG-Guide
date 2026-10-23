# S12 — Brief 4 (mini): `bounded` por defecto y verificación, protocolo 1.2

> Mini-brief de cierre de S12. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md`. Rama `s12-agentes`. Los
> commits los hace Salva.
>
> **Regla del brief:** dos cambios cerrados por el chat. Nada de prompt, modelo ni métricas del arnés
> (las métricas corregidas van al brief de ampliación del golden set).

---

## 0. Contexto

- **Brief 2:** ninguna configuración pasó el criterio preregistrado. `xref` queda implementado y
  **apagado** (recall −0,08, aunque se explique por el denominador: no se reinterpreta a posteriori).
  `bounded` no empeoró ninguna guarda medida (recall 0,90 → 0,91, precisión 0,86 → 0,90, `verified`
  1,00, latencia ≈, coste ×1,12). Solo falló la métrica estricta de cita literal de `same_clause`
  (0/2; 2/2 por `[n]`), que el chat considera mal especificada.
- **UAT del brief 2, caso SDP:** con `orphans`, 4 requisitos y `not_found`. Con `bounded`, respuesta
  desarrollada, 11 requisitos del DRD y sin `not_found`. Es un estrato (preguntas explicativas o de
  contenido de DRD) que el golden set no tiene.
- **Verificación V2–V7 (protocolo 1.1):** la lectura mecánica daba B1/B2 en los 5 casos "sin
  puntero", pero los `assessment` del verificador coinciden casi siempre con el diagnóstico. Las
  discrepancias vienen de la traducción, por tres huecos (§2).

---

## 1. `bounded` como default (excepción registrada)

| # | Decisión | Porqué |
|---|---|---|
| E1 | `RETRIEVAL_EXPANSION` por defecto pasa de `orphans` a `bounded`, con los topes actuales (600 / 2.000) | Ninguna guarda medida empeora; la UAT muestra una mejora clara en un estrato que el instrumento no ve |
| E2 | Se registra como **excepción al criterio de S11** (empate → gana lo simple), con la condición de **confirmarlo** cuando el golden set tenga el estrato explicativo | Mismo patrón que la precisión aceptada fuera del criterio en S11 brief 3 |
| E3 | La nueva referencia del arnés es la configuración `bounded` del artefacto `generation_20261019-224307`, **sin pasada nueva** | Ya está medida con el instrumento validado (`control` dentro del ruido). Si `compare.py` necesita un artefacto de referencia propio, deriva la vista de ese artefacto sin llamar al modelo y regístralo |

Trabajo: cambiar el default en la configuración; ajustar los tests que fijen `orphans` como default
(no los que prueban `orphans` como modo); `ARCHITECTURE.md` y `.claude/CLAUDE.md` (default, porqué,
la excepción y la condición de confirmación); `CHANGELOG.md`.

---

## 2. Verificación independiente, protocolo 1.2

| # | Decisión | Porqué |
|---|---|---|
| P1 | `in_corpus` separa `primary_answers` (como mucho 3: las que responden al hueco **tal como se formuló**) de `related` (lo demás). **B1 solo si el destino del puntero coincide con una respuesta principal** (misma sección, antecesora o descendiente) | V6: un puntero a E-ST-40 5.8 coincidía con una de las muchas cláusulas listadas, aunque el verificador escribía "nothing in the context points to Table D-2" |
| P2 | **Puntero literal, comprobado por código:** el parser de referencias del brief 2 se aplica a la `quote` del puntero (y al estándar de la cláusula citada, para las referencias internas). Si la referencia que se extrae no resuelve a `points_to_doc` (y a `points_to_section` cuando no es nulo), el puntero se marca `inferred` y **no da B1**. `check` lista los inferidos aparte | V3 y V7: el destino lo infería el verificador ("Software configuration file [DDF, SCF…]" no nombra M-ST-40C) |
| P3 | Nuevo campo `answered_as_phrased: yes \| partly \| no`: ¿existe en el corpus lo que se pidió **con su calificativo** (p. ej. "específico de vuelo")? Con `no`, la lectura es **C** (o **A** si se nombra un documento externo), aunque haya contenido relacionado | V4: el verificador escribió "the flight-specific part does not exist as such", pero `found=true` por los requisitos generales de reutilización |
| P4 | Protocolo **1.2**: `VERIFY.md`/`protocol.md` actualizados. `check` rechaza otras versiones. V1 (1.0) y las respuestas 1.1 de V2–V7 se conservan como histórico | Cambia lo que significa una verificación |
| P5 | Los casos V2–V7 se **rehacen en 1.2** como casos nuevos (V8–V13, mismos aspectos y mismo orden), con el flujo `new → fase 1 → reveal → fase 2 → check` | Comprobar que las lecturas 1.2 cuadran con los `assessment` |

Detalles de P2:
- Reutiliza el parser y la normalización del brief 2 (mismo código, sin duplicarlo). Si hace falta
  extraerlo a un módulo común entre `app/` y `corpus/groundtruth/`, hazlo y regístralo.
- Un puntero "por título" (nombra un documento o DRD sin estándar ni número, como "SCMP") no es
  literal: `inferred`. Se puede listar, pero no da B1.

Trabajo: modelos, `mechanical_reading`, `check`, protocolo, workspace y docs. Los casos V8–V13 los
crea CC (`new`). Las sesiones de verificación las abre Salva (UAT).

---

## 3. Tests (sin red)

- `uv run pytest` en verde; recuento frente al de cierre del brief 3.
- E1: el default es `bounded`; `orphans` y `off` siguen funcionando como modos.
- P1: puntero a una respuesta principal → B1; puntero solo a una `related` → no B1.
- P2: puntero cuya cita nombra el destino → literal; cita que no lo nombra (caso V3, "Software
  configuration file [DDF, SCF…]") → `inferred`, no B1; referencia interna resuelta con el estándar
  de la cláusula citada → literal; por título → `inferred`.
- P3: `answered_as_phrased=no` → C, o A con documento externo, aunque haya `primary_answers`.
- P4: `check` rechaza 1.1.
- Regresión: re-leer las respuestas 1.1 de V2–V7 con la lógica nueva **no** está permitido
  (versión distinta). La comprobación es con V8–V13.

---

## 4. Llamadas reales

Ninguna del producto. Las sesiones de verificación las abre Salva.

---

## 5. UAT (usuario)

1. **`bounded` por defecto:** con `.env` sin override de `RETRIEVAL_EXPANSION`, repetir la pregunta
   del SDP. El resultado debe ser el de la UAT del brief 2 con `bounded`.
2. **Verificación 1.2:** correr V8–V13 (`new` ya hecho por CC → sesión → `reveal` → sesión →
   `check`). Tabla final: aspecto · diagnóstico (corregido en las UAT) · lectura 1.2 · ¿coincide?
   Esperado: las discrepancias de V3, V4, V6 y V7 desaparecen o quedan explicadas, y V12 (q04-a3) y
   V9 (q05-a1) ya no salen B1 salvo puntero literal. **Salva revisa a mano solo las discrepancias
   que queden, más 2–3 al azar.**

**Clasificación de q12-a3 (V2), decidida por el chat:** **B1-standard** (el contexto apuntaba a
M-ST-80C y allí está la delegación de los criterios a la política de riesgos del proyecto). Sustituye
a la corrección C de la UAT del brief 1. No cambia ningún recuento.

---

## 6. Cierre

`S12-decisions-4.md` (E1–E3 con la excepción explícita, P1–P5, tabla de V8–V13) y `S12-uat-4.md`.
