# S12 · Decisiones — Brief 1: diagnóstico de forma sobre los `not_found` de la referencia

> Decisiones tomadas al implementar `S12-brief.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Rama `s12-agentes`. Sin llamadas de generación; 24 embeddings de consulta (variantes de retrieval,
> todos en caché).

## 1. Resultado, sin aplicar la regla

La regla de §4 la aplica el chat. Esta es la tabla tal como sale, **antes** de la UAT de Salva. Hay
**27 aspectos canónicos** de 65 líneas, en **13 consultas** (todas salvo q02, la única sin
`not_found`).

| Categoría | Aspectos | Consultas distintas | Consultas |
|---|---|---|---|
| G | 2 | 2 | q07, q08 |
| **B1** | 16 | **8** | q01, q03, q06, q08, q09, q10, q12, q14 |
| **B2** | 2 | **2** | q04, q11 |
| B3 | 1 | 1 | q13 |
| B4 | 3 | 2 | q05, q13 |
| A | 2 | 1 | q04 |
| C | 1 | 1 | q09 |

**Sensibilidad que el chat debería mirar antes de aplicar el umbral** (es un dato, no una
recomendación):

- **B1 por tipo de puntero.** `same_clause` 2 aspectos (q01, q03), `clause` 6 (q01, q06, q08, q09),
  `standard` 7 (q08, q10, q12, q14), `name` 1 (q14). Solo con punteros a cláusula (`clause` o
  `same_clause`) quedan **5 consultas**: q01, q03, q06, q08, q09. Solo con punteros a estándar
  entero, 4: q08, q10, q12, q14.
- **Seguir el puntero no siempre basta.** Con `clause` y `same_clause` la respuesta está en la
  cláusula nombrada: leerla la trae. Con `standard`, la consulta original filtrada a ese estándar
  pone la respuesta en el top 10 en 4 de 7 aspectos (q12-a1, a2 y a4; q14-a1). En q08-a2 y q10 queda
  en el rango 15, y en q12-a3 fuera del top 50.
- **Las 2 B2 son frágiles en direcciones distintas.** q04-a3 (confianza baja) solo llega con una
  subconsulta que ya nombra la «matriz de aplicabilidad». q11-a1 (alta) llega en el rango 1, pero
  elegir E-ST-10-06C como estándar destino no sale de la pregunta: sale de un chunk de E-ST-40C que
  no se inyectó (B.1.2).
- **Las dos `same_clause` (q01-a2, q03) son tablas de una cláusula de la que se inyectó el
  texto, pero no la tabla.** Es lo mismo que mide la expansión de cláusula `bounded`, pendiente
  desde S10, no una referencia cruzada.
- Un aspecto **espurio** (q10: cifras objetivo, cuando el turno pregunta qué hace obligatorio
  Q-ST-80, y Q-ST-80 lo responde sin cifras). Una B1 de confianza **baja** (q14-a2, puntero por
  título). Sin esas dos, B1 sigue en 8 consultas.

## 2. Desviaciones — [AGNÓSTICO]

- **Inyectados sin `chunk_id`.** El artefacto guarda, por chunk inyectado, la cláusula, la autoridad
  y el `cite_text`, pero no el id. No hizo falta re-ejecutar el retrieval: el id se recupera
  emparejando cláusula y texto contra `data/processed/chunks.json` (`injected()`), y falla en voz
  alta si un texto no aparece. Los 14 pools se recuperaron completos. Como comprobación cruzada, la
  consulta original re-ejecutada contra la base pone exactamente los inyectados en su top 10 (q01).
- **El contexto inyectado es idéntico en las 3 ejecuciones de cada consulta** (retrieval
  determinista). G y B se juzgan una vez por consulta, no por ejecución.

## 3. Criterio de agrupación (D2) — [AGNÓSTICO]

Un aspecto canónico por **hueco de contenido**, no por frase. Las variantes de una misma faceta se
juntan aunque una sea más concreta («la documentación de la SRR» y «la de la PDR» van con «la
documentación de cada revisión»). Se separan cuando la respuesta está en **sitios distintos** del
corpus: en q09, «la redacción vinculante» y «la cualificación» comparten puntero, pero «lo
específico de vuelo» es una C, y por eso son tres. Cada canónico conserva sus `(run, índice)`
originales y en cuántas ejecuciones aparece. El script comprueba que las 65 líneas están asignadas
exactamente una vez.

## 4. Decisiones de clasificación no obvias — [AGNÓSTICO] salvo indicación

- **B2 solo cuenta si la variante pone la respuesta en el top 10.** Es la profundidad que inyecta
  producción: una subconsulta de un fan-out que deja la respuesta en el rango 27 no la habría
  servido. B3 es el rango 11–50 con la consulta y el filtro originales. El script lo valida.
- **Las variantes B2 se escriben con palabras de la pregunta.** Una subconsulta con el vocabulario
  de la respuesta («software configuration file contents») la encuentra siempre, y eso no mide lo
  que un plan sacaría de leer la pregunta. Cuando solo el vocabulario de la respuesta llega, es B4
  (q05 a1 y a2, q13-a2), y las variantes probadas que fallaron quedan en el artefacto (`tried`).
- **B1 exige que seguir el puntero lleve a la respuesta**, además de que el puntero exista.
  Si el puntero nombra un estándar entero, se re-ejecuta la consulta original filtrada a ese
  estándar y se registra el rango, aunque quede fuera del top 10: el chat ve lo débil que es.
- **El orden B1 → B3 se respetó aunque el aspecto también sea B3.** q01-a1 (rango 18) y q14-a2
  (rango 31) llevan la B3 como `alternative`, también re-verificada.
- **[DOMINIO] Las guías ESA no son estándares ECSS del subconjunto.** q04 a1 y a2 son A con
  confianza alta, porque el handbook inyectado remite literalmente al anexo E y al anexo D de la guía
  ISVV de ESA.
- **[DOMINIO] «Específico de vuelo» es C.** Los requisitos de reutilización de ECSS discriminan por
  criticidad, no por segmento.
- **Parcialidad.** Ocho aspectos están cubiertos en parte por lo inyectado (campo `partial`). Se
  clasifican por la parte que falta, como pide el brief.

## 5. Hipótesis anotadas, sin implementar

Para el chat. Ninguna está medida: son lo que la evidencia sugiere.

- **H-a. Expansión de referencias por cláusula, determinista.** 5 de las 8 consultas B1 tienen
  punteros a una cláusula o tabla concreta («Clause 6.2.7 in ECSS-Q-ST-80», «Table D-2», «Table
  R-1», «requirement 4.6.f», «see Table A-1»). Sale con una regex sobre el texto inyectado más
  `get_clause`: sin LLM y sin bucle. Los formatos son varios (`[ECSS-Q-80] Table D-2`, `ECSS-Q-ST-80
  Clause 6.2.7`, `ECSS-E-ST-40 | 5.3.3.2b`), y eso es trabajo de normalización, no de
  orquestación.
- **H-b. Los punteros a estándar entero necesitan una subconsulta dentro del estándar destino.**
  Reaprovechar la consulta original con el filtro llega al top 10 en 4 de 7 aspectos. Reformular
  dentro del estándar es justo el paso que depende de lo observado: el único sitio de este
  diagnóstico donde un bucle aportaría algo que el código fijo no da.
- **H-c. Las tablas de una cláusula ya inyectada** (q01-a2, q03) son de la expansión de cláusula
  `bounded`, ya pendiente; no son referencia cruzada.
- **H-d. q12 y q11 son el coste conocido del filtro de routing** (`names_standard_xref`,
  pendiente desde S10): el estándar al que remite el nombrado queda fuera. q12 lo nombra el texto
  inyectado (B1); q11 solo un chunk sin inyectar.
- **H-e. Las B4 son de vocabulario** (estrato `paraphrase`, q05 y q13): el usuario no dice
  «configuration control board» ni «maintainer». Encaja más con reescribir la consulta que con
  orquestar.
- **Observado, sin relación con la forma:** la tabla 6-3 de Q-ST-40C (6.5.6.3::t0) solo trae las
  filas de las funciones I y II. Es la deuda de tablas aplanadas que ya está en la lista.

## 6. D6 — `MAX_HISTORY_TURNS` ≥ 1 — [AGNÓSTICO]

`MAX_HISTORY_TURNS: int = Field(default=10, ge=1)`. Con 0 el arranque falla nombrando la
variable (test). Sigue valiendo el 1 que usa la UAT de pre-S12.

## 7. Recuento de tests

**1.296 → 1.308.** Nuevos: 1 de D6 (`test_config`) y 11 del script
(`tests/app/test_notfound_diagnosis_script.py`: literalidad, cláusula y subcláusulas, recuperación
de ids inyectados, y que el validador **rechaza** un G no inyectado, una cita no literal, una B1 sin
referencia inyectada o con un puntero que no llega, una B inyectada, B2 frente a B3, una B2 fuera del
top 10 y una A dentro del corpus). Todos sobre un corpus sintético de tres chunks: sin base, sin
red, sin `data/`. Modificados y borrados: 0.

## 8. Artefactos

- `data/groundtruth/generation/diagnostics/notfound_20261019-141324.json` y `.md` (inmutables:
  `build` se niega si ya existen).
- `notfound_judgments_20261019-141324.json`: los juicios escritos a mano, la entrada de `build`.
- `query_embeddings_cache.json`: 24 embeddings de consulta. Repetir una variante no llama a la API.
- `scripts/notfound_diagnosis.py`: `aspects`, `clause`, `grep`, `rank`, `original`, `check`,
  `build`. `clause` es la consulta por id de cláusula (§3.4 del brief) y es función de script, no
  capacidad del producto: si se promociona a `get_clause`, lo decide el brief 2.

## 9. Verificación independiente — [AGNÓSTICO]

q11-a1 se verificó en una sesión aparte, sin el repo y a ciegas respecto a la clasificación (montaje
y resultado en `S12-uat.md` §5). Corrige **q11-a1 de B2 a B1**: el contexto inyectado apuntaba a
Q-ST-80 7.2 y CC no lo vio. Lección de método: buscar punteros solo hacia la respuesta ya
sospechada sesga hacia B2/B4. Un aspecto se clasifica como «sin puntero» solo después de leer
**todos** los punteros del contexto. El artefacto `notfound_20261019-141324` no se reescribe
(inmutable); la clasificación corregida vive en la UAT.
