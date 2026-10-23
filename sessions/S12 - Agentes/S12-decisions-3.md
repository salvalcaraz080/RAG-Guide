# S12 · Decisiones — Brief 3 (mini): verificación independiente, protocolo 1.1

> Decisiones tomadas al implementar `S12-brief-3.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Rama `s12-agentes`. Sin llamadas reales.

## 1. Registro de la excepción: la herramienta se construyó fuera de un brief — [AGNÓSTICO]

**Qué pasó.** Durante la UAT del brief 1, Salva pidió una consulta independiente «como otras
veces» para un aspecto dudoso (q11-a1), y después que la herramienta fuera reutilizable en vez de
montarse a mano en cada ejecución. Se construyó en dos pasos, sin brief, y quedó en el commit
`9711d43`.

**Qué hace (protocolo 1.0).** Un workspace estable fuera del repo
(`C:\Users\salca\proyectos\ecss-verification`) con el protocolo (`VERIFY.md`), sus instrucciones
(`CLAUDE.md`), el export de cláusulas del golden set y una carpeta por caso. Una sesión de Claude Code
abierta allí, sin el repo y sin la clasificación, responde en dos fases. Fase 1: qué documentos
consultaría leyendo solo la pregunta. Fase 2: si el contexto ya lo respondía, si está en el corpus
(con citas literales), qué punteros llevan allí (desde el contexto y desde fuera) y si el hueco se
preguntó. `corpus/groundtruth/verification/cases.py` crea los casos (`new`) y comprueba las
respuestas (`check`): citas con el validador del golden set, lo citado como contexto contra lo
inyectado, el esquema, y una lectura mecánica a la taxonomía del diagnóstico. De qué aspecto viene
cada caso vive solo en `data/groundtruth/verification/cases.json`.

**Por qué.** Sustituir la parte pesada de la UAT de clasificación: un humano no puede releer el
corpus entero por cada aspecto, y CC no puede revisar su propia clasificación con independencia.
La independencia es **de contexto, no de modelo**: la sesión no sabe qué se clasificó ni cómo
funciona el retriever.

**Resultado del caso V1 (q11-a1).** Las 22 citas, literales. Los dos punteros de contexto (E-ST-40C
5.4.2.1 y B.2.1) estaban de verdad en lo inyectado. La fase 1 eligió E-ST-10-06C con solo leer la
pregunta (la ruta B2 era real), y la fase 2 encontró un puntero que CC no había visto, «see
ECSS-Q-ST-80 clause 7.2», hacia la regla propia del software (Q-ST-80C 7.2.1.2). Clasificación
corregida de B2 a **B1**.

**Lección de método.** CC buscó punteros solo hacia donde ya creía que estaba la respuesta, y eso
sesga hacia B2/B4: un aspecto parece «sin puntero» porque no se leyeron todos. Desde entonces, el
protocolo pide leer **todos** los punteros del contexto (P2.3).

**Lo que el chat encontró al revisarla**, y este brief corrige: (a) `points_to` era texto libre y
nadie comprobaba que el destino fuera la respuesta, así que la lectura mecánica daba B1 por la mera
existencia de un puntero inyectado; (b) el orden de fases era una petición del protocolo, no una
garantía.

## 2. Decisiones de este brief — [AGNÓSTICO]

- **V1, destino del puntero.** `points_to_doc` es obligatorio. `points_to_section` es obligatorio
  como campo y nulo para un estándar entero o un título sin número. `points_to` se queda como
  explicación. Una respuesta con un puntero sin destino no cumple el esquema.
- **V2, B1 por destino.** `pointer_kind` da `B1-clause` si un puntero verificado nombra la sección
  de una respuesta, una antecesora o una descendiente («7.2» frente a «7.2.1.2»), y `B1-standard`
  si `points_to_section` es nulo y coincide el documento. `B1-clause` gana cuando hay los dos. Un
  puntero a otra parte no cuenta, y `check` lo lista aparte («pointers in context that do not lead
  to an answer»), porque es información útil para el diagnóstico.
- **V3, orden por el flujo.** `reveal` es la única forma de que exista `context.md`. Se niega si el
  caso no existe, es de otra versión, ya se reveló, no hay `answer.json`, o su fase 1 no es válida
  (ausente, vacía, ítems incompletos o versión distinta). El hash de la fase 1 (JSON canónico) se
  guarda en el registro, **fuera del workspace**, para que no se pueda reescribir junto con la
  respuesta. `check` marca «phase 1 changed after the context was revealed» y «the context was never
  revealed».
- **V4, versión.** `PROTOCOL_VERSION = "1.1"`. `check` rechaza un caso registrado en otra versión y
  una respuesta con otra `protocol_version`. **No escribe** el informe de un caso de otra versión:
  `checks/V1.json` (1.0) se conserva tal cual (comprobado en PowerShell).
- **V5 y los casos V2–V7.** V1 se queda en 1.0, con su respuesta, su informe y el borrador de
  protocolo con que se respondió (`cases/V1/VERIFY_draft.md`). **V2–V7 se rehicieron:** eran de 1.0,
  nadie los había abierto ni respondido, y ya tenían `context.md`, lo que rompe V3. Se retiraron y
  se crearon en 1.1 con los mismos ids y aspectos y en el mismo orden (q12-a3, q05-a1, q09-a2,
  q13-a2, q04-a3, q05-a2). Copia del registro y de las carpetas de 1.0 hecha antes de retirarlos.
- Con la lectura de 1.1, V1 habría dado **`B1-clause`**: su puntero de 5.4.2.1 lleva a Q-ST-80C 7.2,
  antecesora de la respuesta 7.2.1.2. No se re-ejecuta (V5).

## 3. Tests de contrato

**1.380 → 1.403** (+23). `tests/corpus/test_verification_cases.py` pasa de 16 a 39 casos y se
reescribe para el flujo 1.1: los de 1.0 que suponían `context.md` al crear el caso se adaptan a
`new → reveal`. Nuevos: `reveal` sin respuesta y con cuatro fases 1 inválidas, `reveal` una sola vez
con su hash, fase 1 cambiada, contexto nunca revelado, rechazo de versión (respuesta y caso), puntero
sin destino, B1 por destino (cláusula, antecesora, descendiente, estándar, otra sección, otro
estándar), cláusula sobre estándar, y la coincidencia de los dos B1 con el B1 del diagnóstico.

## 4. Uso a partir de aquí

La verificación sustituye la comprobación pesada de la UAT de clasificación. Salva revisa los
casos en que verificador y diagnóstico **discrepan**, más 2–3 al azar: esa muestra es el ancla
humana.

## 5. Corrección tras la UAT — [AGNÓSTICO]

- **La trampa del paso 4 dio `VALID`.** CC lo reprodujo sobre una copia de V2: cambiar la `why` de
  un ítem de `phase1` sí da `INVALID`. Causa más probable: la instrucción decía «cambia una `why`», y
  `answer.json` tiene otra en `gap_asked_for`, fuera de la fase 1. La UAT ahora da los comandos
  exactos, probados en PowerShell, y un test fija que un cambio fuera de `phase1` no cuenta.
- **Fallo real encontrado de paso:** `answer.json` guardado con BOM (`Set-Content -Encoding UTF8` en
  PowerShell 5.1) hacía caer `check` con un traceback, y `reveal` lo tomaba por una fase 1 inválida.
  `read_answer` lee con `utf-8-sig`, y un fichero que no es JSON es un problema reportado. **1.403 →
  1.407** (+4).
- **Primera discrepancia del flujo 1.1:** V2 (q12-a3) da `B1-standard` y la clasificación corregida
  en la UAT del brief 1 es C. Mismos hechos, distinta categoría; la decide Salva.

