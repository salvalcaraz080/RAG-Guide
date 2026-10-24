# S13 — Brief 2: punteros a un estándar entero (arista condicional determinista)

> Segundo brief de S13. Requiere el brief 1 cerrado (la pasada de este brief produce también el
> desglose de latencia). Rama `s13-orquestacion`. Los commits los hace Salva.
>
> **Regla del brief:** una variable cada vez. Este brief añade **una** capacidad de retrieval
> (seguir los punteros a un estándar entero desde lo inyectado) **apagada por defecto**. No cambia
> el prompt, el modelo, el esquema de salida, el routing ni los topes existentes. Si algo de eso
> pareciera necesario, `TODO: pendiente de decisión en Claude Chat` y se reporta.

---

## 0. Contexto y objetivo

**De dónde sale.** El diagnóstico de S12 dejó un único hueco reactivo sin cubrir: los punteros a un
**estándar entero** («as specified in ECSS-M-ST-80»). Hoy el parser de `xref` los detecta y los
registra con motivo `whole_standard`, pero no los sigue. Es la hipótesis H-b/H-h de S12 y el último
candidato a bucle.

**Lo que decidió el chat** (`S13-resumen.md` §7 y §9), con el criterio de la Guide §3.8:
- El siguiente paso depende de lo observado (qué estándar nombra el texto inyectado), pero **lo
  resuelve una regla**: puntero a cláusula → `get_clause` (como hoy); puntero a estándar entero →
  **retrieval con la misma consulta filtrado a ese estándar**. Es una arista condicional, no un
  bucle: sin LLM, sin tools, sin reformular la consulta.
- Reformular la subconsulta dentro del estándar destino (una llamada más) queda **fuera**. Se decide
  después de medir esta vía, con el desglose de "por qué no".
- Dato de partida (diagnóstico S12): la consulta original filtrada al estándar destino pone la
  respuesta en el **rango 2–4** en q12-a1, q12-a2 y q14-a1, en el **9** en q12-a4, en el **15** en
  q08-a2 y fuera del top 50 en q12-a3.

**Objetivo.** Implementar el seguimiento de punteros a estándar entero como opción de retrieval
apagada por defecto, y medirla con el arnés de generación contra un criterio fijado en este brief.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| W1 | Nuevo valor de `RETRIEVAL_XREF`: `off \| clause \| clause_standard`, **default `clause`** (sin cambio) | Misma opción, mismo mecanismo, un paso más. `clause_standard` incluye todo lo de `clause` |
| W2 | Un puntero `whole_standard` que resuelve a un documento del corpus se sigue con **la consulta de retrieval de la petición** (el mismo embedding, sin llamada nueva) filtrada a ese documento | Reutiliza el embedding: el coste es una consulta a la base. Sin reformular: lo simple primero |
| W3 | **No se sigue** si el estándar destino ya está en el filtro de routing activo de la petición; se registra (`already_searched`) | La misma consulta sobre un estándar ya buscado no aporta nada nuevo. El hueco real es el estándar al que remite el nombrado (`names_standard_xref`, H-d de S12) |
| W4 | Cada estándar seguido ocupa **una plaza** del tope existente de 5 (`RETRIEVAL_XREF_MAX_CLAUSES`) y entra con el tope por plaza (600 tokens) y el total (2.000) **ya vigentes**, en orden de aparición junto a los punteros a cláusula | Sin topes nuevos que ajustar al resultado. Se acepta de antemano que el rango 9 y el 15 probablemente no entren |
| W5 | Dentro de la plaza, los chunks en **orden de rango** del retrieval filtrado, excluyendo los ya inyectados, hasta el tope por plaza | Determinista. El rango es la señal que da el retrieval; no hay otra mejor sin LLM |
| W6 | Lo añadido entra con `origin="expansion"` y la misma presentación al modelo, al final del contexto | Cero cambios de prompt (hashes iguales a la referencia). `origin` explícito es otro candidato de S13, no este brief |
| W7 | Con `RETRIEVAL_XREF=clause`, el array de mensajes y el contexto son **idénticos byte a byte** a los actuales | La opción no puede mover la referencia cuando está apagada |

---

## 2. Trabajo

### 2.1 Resolución

- Reutiliza el parser y la normalización de `references.py`. Un `whole_standard` que **resuelve a un
  único documento del corpus** es candidato; fuera del corpus o ambiguo sigue sin seguirse, con su
  motivo actual.
- Los punteros por **título** sin identificador (caso q14-a2, V11) siguen fuera.
- Traza: por cada puntero a estándar, `followed` / `already_searched` / `over_limit` / motivo de no
  resolución, los `chunk_id` añadidos con su rango en el retrieval filtrado, y los tokens. En
  `RetrievedContext.xref` y en `references_expanded`, como hoy.

### 2.2 Arnés: aspectos objetivo

Métrica de `aspects.py`, con las **métricas corregidas que S12 dejó preregistradas**:

- **Conjunto objetivo `standard`** (aspectos B1 con puntero a estándar entero, clasificación
  corregida en S12): **q08-a2, q12-a1, q12-a2, q12-a3, q12-a4, q14-a1** (6 aspectos). q10-a1 se
  excluye por espurio.
- **Resuelto** en una ejecución = el chunk respuesta del aspecto (`notfound_20261019-141324.json`)
  **citado por `[n]`** en la respuesta, y que la referencia no tenía. La lectura con cita literal
  (`verified` estricta) se reporta al lado, no decide.
- **Recall sobre relevantes recuperadas** (las que trajo el retrieval por rango, no la expansión) y
  **recuentos absolutos** de relevantes citadas, junto al recall condicionado de siempre. Si el
  arnés no las calcula todavía como métrica de primera clase, añádelas aquí (son las de S12
  §5.4).
- **"Por qué no"**, por ejecución no resuelta: `not_detected`, `already_searched`, `over_limit`,
  `not_in_slot` (el chunk respuesta no entró en la plaza por rango o tope), `brought_not_cited`,
  `trimmed_by_budget`.

### 2.3 Comprobación previa sin generación

Con `clause_standard`, solo retrieval (sin generación), sobre q08, q12 y q14: qué estándares se
siguen, qué chunks entran y si el chunk respuesta de cada aspecto objetivo está entre ellos. Se
**reporta** antes de la pasada y **no se ajusta nada** a la vista de ella.

### 2.4 Docs

`ARCHITECTURE.md` (expansión por referencia: el valor nuevo, W2–W5, que el default no cambia) y
`.claude/CLAUDE.md` si aplica. `CHANGELOG.md` derivado.

---

## 3. Medición (preregistrada)

### 3.1 Configuraciones

Golden set v2, 14 consultas × 3 ejecuciones, mismo modelo, prompt y límites que la referencia:

| Config | Expansión de cláusula | `RETRIEVAL_XREF` |
|---|---|---|
| `control` | `bounded` | `clause` |
| `standard` | `bounded` | `clause_standard` |

`control` repite los defaults el mismo día. Si se sale de la referencia (`xref+bounded` de
`generation_20261019-224307`) más que el ruido (±0,03) en recall o precisión, **para y reporta**
antes de comparar. Las dos configuraciones guardan el desglose de latencia del brief 1.

### 3.2 Criterio de aceptación (fijado antes de medir)

**`standard` se acepta como nuevo default si cumple todo esto, pareado con `control`:**
1. **Éxito:** ≥ 3 de los 6 aspectos del conjunto `standard` resueltos (§2.2) en ≥ 2 de las 3
   ejecuciones.
2. **Recall sobre relevantes recuperadas** no baja más de 0,03, y las **relevantes citadas en
   absoluto** no bajan.
3. **`verified`** sigue en 1,00 (ninguna cita no literal ni mal atribuida nueva).
4. **Precisión sin cláusulas solo de expansión** no baja más de 0,03.

Se reportan sin ser guarda: latencia (mediana, p95 y el desglose por etapa), coste por turno, tokens
de entrada, `not_found` por ejecución, estado servido, precisión estándar, recall condicionado y
citas sobre chunks añadidos. Si la mediana de tokens de entrada se acerca a `MAX_INPUT_TOKENS`,
reporta.

**Con 6 aspectos en 3 consultas, el resultado es indicativo.** La confirmación llega con el golden
set 1.2 (estrato de referencias cruzadas, ya diferido con señal).

**CC no aplica el criterio ni cambia defaults:** reporta la tabla y los casos. Si `standard` no
pasa, **no se itera** (ni los topes, ni el orden, ni la consulta): se reporta el desglose de "por
qué no". Ese desglose decide si la reformulación (una llamada más) tiene caso.

### 3.3 Llamadas reales autorizadas

- **Una pasada** de 2 configuraciones ≈ **0,25 USD** (≈ 0,11–0,12 por configuración más turnos
  previos).
- **Una segunda pasada** completa solo si falla el instrumento (no el resultado): `control` fuera
  del ruido, errores de proveedor, artefacto incompleto o desglose de latencia que no cuadra
  (`unaccounted_ms` desproporcionado).
- Embeddings de consulta: los que hagan falta; registra el número (W2 no debería añadir ninguno).

---

## 4. Tests de contrato (sin red)

- `uv run pytest` en verde. Recuento frente al cierre del brief 1.
- **W7:** con `RETRIEVAL_XREF=clause`, equivalencia byte a byte del array de mensajes y del contexto
  frente a hoy. Con `off`, la de X8 sigue valiendo.
- Resolución: estándar del corpus → se sigue; fuera del corpus, ambiguo o por título → no; ya en el
  filtro de routing → `already_searched`.
- Plazas: un estándar ocupa una plaza; mezcla de punteros a cláusula y a estándar respeta el orden
  de aparición y el tope de 5; tope por plaza y total.
- Selección dentro de la plaza: orden de rango, sin duplicados ni ya inyectados.
- Sin embedding nuevo: el retrieval filtrado reutiliza el de la petición.
- Arnés: puntuación por `[n]` sobre artefactos sintéticos (resuelto / citado sin `[n]` al chunk
  respuesta / cita a otra cláusula / aspecto ausente) y las métricas de §2.2.

---

## 5. UAT (usuario)

Propón el script. Mínimo, con `RETRIEVAL_XREF=clause_standard` en la UI:
1. q12 (riesgos de software): la respuesta cita M-ST-80C (marcado ➕) donde antes había `not_found`.
2. q14 (OBCP): ídem con E-ST-70-01C.
3. Una consulta sin punteros a estándar: la respuesta no cambia frente a `clause`.
4. Del desglose de latencia de la pasada: qué etapa domina y cuánto añade la etapa `xref` con
   `clause_standard`.

Si la herramienta de verificación independiente aplica (comprobar que lo traído responde cada
aspecto), el script puede delegarle la comprobación pesada y dejar a Salva las discrepancias más
2–3 al azar (protocolo 1.2).

---

## 6. Cierre

- `S13-decisions-2.md` en `RAG-Guide/sessions/S13 - Orquestacion/`: §1 preregistro (resolución,
  plazas, comprobación previa) escrito **antes** de medir; resultados después (tabla del criterio
  sin aplicarlo, aspectos uno a uno, desglose de "por qué no", desglose de latencia por etapa de
  las dos configuraciones), recuento de tests, etiquetado agnóstico/dominio.
- `S13-uat-2.md` con el script de UAT.
