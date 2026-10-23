# S12 · Decisiones — Brief 4 (mini): `bounded` por defecto y verificación, protocolo 1.2

> Decisiones tomadas al implementar `S12-brief-4.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Rama `s12-agentes`. Sin llamadas reales.

## 1. `bounded` por defecto — excepción registrada

- **E1.** `RETRIEVAL_EXPANSION` por defecto pasa de `orphans` a **`bounded`**, con los topes de
  600 / 2.000. El valor cambia en `app/config.py`, en `.env.example` y en `.env` local, así que no
  queda ningún override a `orphans`. Test ajustado: el que fija el default
  (`test_retrieval_defaults_without_optional_variables`); los que prueban `orphans` y `off` como
  modos siguen igual y en verde.
- **E2 — [DOMINIO] la excepción.** Frente a `orphans`, ninguna guarda medida empeora (recall 0,90 →
  0,91, precisión 0,86 → 0,90, `verified` 1,00, latencia mediana 12,9 → 12,7 s, coste ×1,12). Con el
  criterio de S11 el empate habría dado `orphans` («gana lo simple»). Se adopta porque la métrica de
  `same_clause` (cita literal) estaba mal especificada (0 de 2; 2 de 2 citados por `[n]`) y porque
  la UAT muestra una mejora clara en un estrato que el golden set no tiene: preguntas explicativas o
  de contenido de un DRD. En el SDP, de 4 requisitos y un `not_found` se pasa a 11 requisitos sin
  `not_found`. **Condición: confirmarlo cuando el golden set tenga ese estrato.** Registrada en
  `ARCHITECTURE.md` §5 (junto a la excepción de la precisión de S11), en §7 y en `CLAUDE.md`.
- **E3 — referencia sin pasada nueva.** La referencia del arnés es la configuración `bounded` de
  `generation_20261019-224307`. `compare.py` ya lee una configuración de un artefacto
  (`--baseline-config bounded`), así que **no hace falta derivar un artefacto propio**: se
  referencia el que hay. Cifras de la referencia: recall 0,91, precisión 0,90, `verified` 1,00 (1
  cita `not_found`), citadas / inyectadas 0,68, mediana 12,7 s, 0,0019 USD por turno.

## 2. Verificación, protocolo 1.2 — [AGNÓSTICO]

- **P1, respuestas principales.** `in_corpus.primary_answers` (como mucho 3; el esquema rechaza 4)
  y `related`. B1 solo si el destino del puntero coincide con una **principal** (misma sección,
  antecesora o descendiente). Un puntero que solo lleva a una relacionada no da B1, y `check` lo
  lista aparte («literal pointers … that do not lead to a primary answer»).
- **P2, puntero literal, comprobado por código.** `is_literal` aplica a la `quote` del puntero el
  **mismo** `extract` y `normalize_standard` de `app/services/references.py` (importados, sin
  duplicar ni extraer a otro módulo: `corpus/groundtruth/` ya importa de `app/`). Las referencias
  internas se resuelven con el estándar de la cláusula citada. Las de tabla, con las etiquetas que
  `check` calcula de `data/processed`, el mismo cálculo que la ingesta. Es literal si alguna
  referencia de la cita resuelve a `points_to_doc` y, cuando `points_to_section` no es nulo, a esa
  sección, una antecesora o una descendiente. Si no, `inferred`: `check` lo lista y no da B1. Un
  puntero por título («Software configuration file [DDF, SCF…]», «SCMP») no contiene ninguna
  referencia: `inferred`. Los fixtures son citas reales de V3 y V6.
- **P3, `answered_as_phrased: yes | partly | no`.** Con `no`, la lectura es C, o A si
  `outside_corpus` nombra el documento, aunque haya respuestas principales.
- **P4, versión 1.2.** `VERIFY.md`/`protocol.md` actualizados. `check` rechaza 1.1 sin reescribir
  su informe (comprobado en PowerShell con V2). V1 (1.0) y V2–V7 (1.1) se conservan.
- **P5, V8–V13 creados** (`new`, solo `question.md`), mismos aspectos y mismo orden que V2–V7.

## 3. Clasificación de q12-a3, decidida por el chat

**B1-standard**: el contexto apuntaba a M-ST-80C, y allí está la delegación de los criterios a la
política de riesgos del proyecto. Sustituye la corrección a C de la UAT del brief 1 y coincide con lo
que leyó V2. No cambia ningún recuento.

## 4. Tabla de V8–V13 (la completa la UAT)

| Caso | Aspecto | Diagnóstico (corregido en las UAT) | Lectura 1.1 (V2–V7, histórico) | Lectura 1.2 | ¿Coincide? |
|---|---|---|---|---|---|
| V8 | q12-a3 | **B1-standard** (decisión del chat) | V2: B1-standard | B1 | sí |
| V9 | q05-a1 | B4 | V3: B1-standard | B2? | sin decidir → B4 (rango medido) |
| V10 | q09-a2 | C | V4: B1-clause | C | sí |
| V11 | q13-a2 | B4 | V5: B2? | B2? (tras la corrección del esquema, brief 5) | sin decidir → B4 (rango medido) |
| V12 | q04-a3 | B4 (UAT brief 1: de B2) | V6: B1-clause | B2? | sin decidir → B4 (rango medido) |
| V13 | q05-a2 | B4 | V7: B1-clause | B2? | sin decidir → B4 (rango medido) |

Esperado según el brief: desaparecen o quedan explicadas las discrepancias de V3, V4, V6 y V7, y V9
y V12 ya no salen B1 salvo con puntero literal.

**Resultado (UAT del brief 4, y V11 tras el brief 5).** Las cuatro discrepancias de 1.1 desaparecen:
ninguno de los cinco aspectos «sin puntero» sale B1 en 1.2. V10 coincide (C) y V8 coincide (B1). V9,
V11, V12 y V13 dan `B2?`: la fase 1 elige el documento de la respuesta, así que el verificador dice
que un plan que leyera la pregunta iría allí. Que eso baste para ser B2 lo decide un rango, y el
diagnóstico ya lo midió: las variantes con palabras de la pregunta no llevan la respuesta al top 10
(q04-a3, rango 27; q05-a1, 33; q05-a2 y q13-a2, fuera del top 50). **Siguen en B4.** El «same
category: no» de `check` frente a B4 es esa lectura sin decidir, no una discrepancia.

## 5. Tests

**1.407 → 1.417** (+10 en `test_verification_cases.py`: literal con cláusula y con estándar,
inferido por destino no nombrado y por título, interno resuelto con el estándar citado, tabla por
etiqueta, B1 solo con respuesta principal, como mucho 3 principales, `answered_as_phrased=no` → C/A,
inferido listado aparte). Modificados: los fixtures del mismo fichero pasan a 1.2 (cita de puntero
que nombra su destino, `primary_answers`, `answered_as_phrased`), el rechazo de versión pasa a 1.1, y
el test del default de expansión en `test_config.py`. Borrados: 0.
