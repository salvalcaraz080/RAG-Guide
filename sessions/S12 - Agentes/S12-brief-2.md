# S12 — Brief 2: expansión determinista de referencias cruzadas (+ medición de `bounded`)

> Segundo brief de S12. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar. Rama:
> `s12-agentes` (contiene pre-S12 y el brief 1). Los commits los hace Salva.
>
> **Regla del brief:** una variable cada vez. Este brief añade **una** capacidad de retrieval
> (seguir referencias a cláusula desde lo inyectado) **apagada por defecto**, y la mide junto con
> `bounded` (que ya existe). No cambia el prompt, el modelo, el esquema de salida ni el routing. Si
> algo de eso pareciera necesario, `TODO: pendiente de decisión en Claude Chat` y se reporta.

---

## 0. Contexto y objetivo

**De dónde sale.** El brief 1 clasificó los `not_found` de la referencia
`generation_20261019-141324`. Con la regla preregistrada y la clasificación corregida en la UAT,
**B1 afecta a 9 consultas** (8 sin el espurio de q10) y **B2 a 0**. Hay señal de forma **reactiva**:
el contexto inyectado nombra dónde está lo que falta, pero el pipeline no lo sigue. No hay señal de
forma predecible (fan-out por estándar), así que ese camino se descarta.

**Lo que decidió el chat** (ver `S12-resumen.md` §1 y §8):
- La respuesta a una señal reactiva empieza por lo **determinista**: detectar la referencia en el
  texto inyectado y traer la cláusula nombrada. Sin LLM, sin bucle, sin tools.
- **Solo punteros a cláusula** (cláusula, tabla, anexo, requisito con ID). Los punteros a un
  estándar entero (q08, q10, q12, q14) quedan **fuera**: la búsqueda filtrada al estándar destino
  solo llegaba al top 10 en 4 de 7 aspectos, y es el único sitio donde un bucle aportaría algo.
  Se decide después de esta medición.
- **`bounded` se mide en la misma pasada.** Cubre los aspectos `same_clause` del diagnóstico (tablas
  de una cláusula de la que se inyectó el texto pero no la tabla: q01-a2, q03-a1) y está pendiente
  desde S10.

**Objetivo.** Implementar la expansión de referencias a cláusula como opción de retrieval apagada
por defecto, y medir con el arnés de generación cuatro configuraciones contra un criterio fijado en
este brief.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| X1 | Nueva opción de retrieval **ortogonal** a la expansión de cláusula: `RETRIEVAL_XREF = off \| clause`, **default `off`** | Combinable con `orphans` y `bounded` sin mezclar mecanismos. El default no cambia hasta que el chat lo adopte con la medición |
| X2 | Las referencias se detectan **al consultar**, sobre el texto de los chunks **inyectados**, con un parser determinista | `outgoing_refs` está vacío y bloqueado hasta validar el extractor, y `corpus/` está congelado. Escanear solo lo inyectado es exactamente la definición de B1 del diagnóstico |
| X3 | Solo se expanden referencias que **resuelven a una cláusula concreta del corpus** (cláusula, subcláusula, tabla, anexo, requisito con ID). Las referencias a un estándar entero, a documentos fuera del corpus o que no resuelven se **registran y no se expanden** | Alcance decidido en el chat. Registrar lo no resuelto mide lo que un paso siguiente (bucle o filtro por estándar) podría cubrir |
| X4 | La cláusula destino se trae con una **consulta por ID** (`get_clause`) promocionada desde el script del brief 1 a función del servicio de datos | Es la primitiva útil con o sin agente. Ahora tiene un consumidor de producto |
| X5 | **Tope de 5 cláusulas añadidas por respuesta** y un tope de tokens por cláusula y total (análogos a los de `bounded`; valores de partida en §2.4) | Acotar el contexto que crece. Mismo patrón que `bounded`, para no inventar otro |
| X6 | Los chunks añadidos por referencia entran **marcados como añadidos por expansión**, igual que hoy los de `orphans`/`bounded`, y con la **misma presentación al modelo** | Cero cambios de prompt: el hash del system prompt y del template de usuario debe ser el mismo que en la referencia. La UI ya distingue lo añadido |
| X7 | En el presupuesto, lo añadido por referencia es **lo primero que se descarta** | Lo recuperado por similitud tiene prioridad sobre lo inferido por referencia |
| X8 | Con `RETRIEVAL_XREF=off`, el array de mensajes y el contexto son **idénticos byte a byte** a los actuales | La opción no puede mover la referencia cuando está apagada |

---

## 2. Trabajo

### 2.1 Parser de referencias

Formatos observados en el diagnóstico (punto de partida; amplía con lo que encuentres en los
chunks inyectados del golden set y regístralo):

- `ECSS-Q-ST-80 Clause 6.2.7`, `ECSS-Q-ST-80 clause 7.2`, `(see ECSS-Q-ST-80 clause 7.2)`
- `as per … ECSS-Q-ST-80, 7.2.1.3`
- `[ECSS-Q-80] Table D-2`, `Table R-1`, `see Table A-1`
- `requirement 4.6.f`, `ECSS-E-ST-40 | 5.3.3.2b`
- Referencias **internas** sin estándar (`see clause 5.4`, `Table D-2`): se resuelven contra el
  estándar del propio chunk.

Normalización:
- Identificador de estándar sin letra de revisión ni sufijo (`ECSS-Q-ST-80`, `[ECSS-Q-80]`) → la
  versión del corpus (`ECSS-Q-ST-80C`). Si hay ambigüedad, no se expande y se registra.
- `Table X-n` → la cláusula o chunk de tabla correspondiente, según la convención de IDs del corpus.
- Requisito con letra (`4.6.f`, `5.3.3.2b`) → la cláusula (`4.6`, `5.3.3.2`). La letra se conserva en
  el log, no hace falta para traerla.
- Una referencia a la **misma cláusula** de un chunk ya inyectado no es una referencia cruzada: no
  se expande por X1 (para eso está `bounded`).

### 2.2 Resolución y selección

1. Recorre los chunks inyectados **por orden de rango**; extrae y normaliza referencias; quita
   duplicados y las que apunten a cláusulas ya inyectadas.
2. Resuelve contra el corpus con `get_clause`. Lo que no resuelve → log (`unresolved`, con el motivo:
   fuera del corpus, estándar entero, ambiguo, formato no reconocido).
3. Toma las primeras 5 resueltas (X5), por orden de aparición.

### 2.3 Qué chunks de la cláusula destino entran

Una cláusula puede tener muchos chunks (el anexo O.2.1 de E-ST-40C tiene docenas). Criterio de
partida, **normativos primero** como en `bounded`, hasta el tope de tokens por cláusula. Si
dentro de una cláusula grande hace falta ordenar, el orden por similitud con la consulta es
aceptable siempre que sea determinista. Regístralo.

### 2.4 Topes de partida

`RETRIEVAL_XREF_MAX_CLAUSES=5`, y por cláusula y total los mismos valores de partida que `bounded`
(600 / 2000 tokens) salvo que encuentres un motivo para otros. En ese caso, regístralo **antes** de
medir. Si la mediana de tokens de entrada se acerca a `MAX_INPUT_TOKENS`, reporta.

### 2.5 Observabilidad y traza

Por respuesta, en log y en el artefacto del arnés: referencias detectadas, resueltas, expandidas
(con `chunk_id`), no resueltas con motivo, y tokens añadidos. Debe poder decirse qué cita se apoya en
un chunk añadido por referencia.

### 2.6 Arnés: puntuación de aspectos resueltos

Nueva métrica, **determinista y re-puntuable** sobre los artefactos:

- **Conjunto objetivo `xref`** (aspectos B1 con puntero a cláusula, según la clasificación corregida
  en `S12-uat.md`): **q01-a1, q06-a1, q06-a2, q09-a1, q09-a3, q11-a1** (6 aspectos).
- **Conjunto objetivo `same_clause`**: **q01-a2, q03-a1** (2 aspectos).
- Un aspecto está **resuelto** en una ejecución si la respuesta tiene al menos una afirmación con
  **cita verificada** (`verified`) a un chunk de la **cláusula respuesta** que registró el diagnóstico
  para ese aspecto (`notfound_20261019-141324.json`).
- Se reporta además, por configuración: número de `not_found`, estado servido (`partial` frente a
  `answered`), tokens de entrada, latencia (mediana y p95), coste y citas a chunks añadidos por
  expansión.
- **Precisión de cita en dos cifras:** la estándar, y la misma excluyendo las citas a chunks añadidos
  por expansión. El golden set v2 no juzga esas cláusulas, así que la cifra estándar sale sesgada a la
  baja.

### 2.7 Docs

`ARCHITECTURE.md` y `.claude/CLAUDE.md`: estado actual (la opción, su default, sus topes, `get_clause`
como primitiva del servicio de datos). `CHANGELOG.md` derivado.

---

## 3. Medición (preregistrada)

### 3.1 Configuraciones

Golden set v2, 14 consultas × 3 ejecuciones, mismo modelo, prompt y límites que la referencia:

| Config | Expansión de cláusula | `RETRIEVAL_XREF` |
|---|---|---|
| `control` | `orphans` | `off` |
| `xref` | `orphans` | `clause` |
| `bounded` | `bounded` | `off` |
| `xref+bounded` | `bounded` | `clause` |

`control` repite los defaults el mismo día: sirve de control contemporáneo y de comprobación de que
el instrumento sigue en el ruido de la referencia (±0,03). Si `control` se sale de la referencia más
que el ruido en recall o en precisión, **para y reporta** antes de comparar.

### 3.2 Criterio de aceptación (fijado antes de medir)

**`xref` se acepta como nuevo default si cumple todo esto, comparado de forma pareada con `control`:**
1. **Éxito:** ≥ 3 de los 6 aspectos del conjunto `xref` resueltos en ≥ 2 de las 3 ejecuciones.
2. **Recall** agregado no baja más de 0,03.
3. **`verified`** sigue en 1,00 (ninguna cita no literal ni mal atribuida nueva).
4. **Latencia** mediana sube como mucho un 20 %.
5. **Coste** por turno juzgado se multiplica como mucho por 1,5.

La precisión se reporta en sus dos cifras (§2.6) pero **no es guarda**.

**`bounded`:** criterio de S11 (gana en omisión en el estrato normativo, no empeora atribución ni
precisión de citas, coste asumible; empate → gana lo simple), más el conjunto `same_clause`
(≥ 1 de 2 resuelto en ≥ 2 de 3 ejecuciones). Con 2 aspectos es **indicativo**, no concluyente.

**`xref+bounded`:** solo se considera si `xref` y `bounded` pasan por separado. Si las dos pasan pero
la combinación viola alguna guarda, se queda la que más aspectos resuelva.

**CC no aplica el criterio ni cambia defaults:** reporta la tabla con `compare.py` y los casos. Si una
configuración no pasa, **no se itera** (ni el parser para "llegar", ni los topes): se reporta qué
aspectos no se resolvieron y por qué (no detectado, no resuelto, recortado por presupuesto, traído
pero no citado). Ese desglose es lo que decide el siguiente paso.

### 3.3 Llamadas reales autorizadas

- **Una pasada** de 4 configuraciones ≈ **0,35 USD** (≈ 0,08–0,09 por configuración).
- **Una segunda pasada** completa solo si falla el instrumento (no el resultado): `control` fuera
  del ruido, errores de proveedor, artefacto incompleto.
- Embeddings de consulta: los que hagan falta; registra el número.

---

## 4. Tests de contrato (sin red)

- `uv run pytest` en verde. Recuento frente a los **1308** de partida.
- **X8:** con `RETRIEVAL_XREF=off`, equivalencia byte a byte del array de mensajes y del contexto
  frente a hoy (mismo patrón que la equivalencia de pre-S12).
- Parser: cada formato de §2.1 (y los que añadas), referencias internas, normalización de revisión,
  requisito con letra, ambigüedad → no expande, referencia a la misma cláusula → no expande.
- Resolución y selección: duplicados, ya inyectado, tope de 5, orden por rango, no resueltas
  registradas con motivo.
- Presupuesto: lo añadido por referencia se descarta antes que lo recuperado (X7).
- Marcado: los chunks añadidos llegan marcados igual que los de `orphans` (X6).
- Arnés: la puntuación de aspectos resueltos sobre artefactos sintéticos (resuelto / cita no
  verificada / cita a otra cláusula / aspecto ausente).

---

## 5. UAT (usuario)

Propón el script. Mínimo, con `RETRIEVAL_XREF=clause` en la UI:
1. Una consulta del conjunto `xref` (p. ej. q09 o q06): la respuesta incluye lo que antes era
   `not_found`, citando la cláusula traída (marcada como añadida por expansión).
2. Una consulta sin referencias: la respuesta no cambia frente a `off` en contenido ni en latencia
   apreciable.
3. Tu caso del SDP (`What is the Software Development Plan in ECSS-E-ST-40C?`, conversación nueva):
   anotar qué pasa con `bounded` frente a `orphans`. Es indicativo: no está en el golden set.

Si para entonces está lista la herramienta de validación externa, el script de UAT puede delegarle
la comprobación pesada (que lo traído responde de verdad cada aspecto) y dejar a Salva una muestra.

---

## 6. Cierre

- `S12-decisions-2.md` en `RAG-Guide/sessions/S12 - Agentes/`: formatos de referencia encontrados,
  normalización, criterio de chunks por cláusula, topes finales, resultados de la medición (tabla de
  `compare.py`, aspectos resueltos por configuración, desglose de los no resueltos), recuento de
  tests, etiquetado agnóstico/dominio.
- `S12-uat-2.md` con el script de UAT.
