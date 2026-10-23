# S12 — Consolidado de sesión (histórico, no se reescribe)

> **Paso 5 del ciclo.** Combina el resumen provisional (`S12-resumen.md`, paso 2) con lo aprendido al
> implementar el brief previo y los cinco briefs (`pre-S12-decisions.md`, `S12-decisions*.md`,
> `S12-uat*.md`). La **divergencia teoría↔medida** se conserva a propósito: es el material de Guide.
> La graduación a `RAG-GUIDE.md` (paso 6) se hace **desde** este documento.
>
> **Sesión:** M5 / S12 — Agentes: anatomía, patrones, function calling, coste. **Cerrada con:** sin
> bucle agéntico (no había caso medido); historial por turnos y suite de contrato sin red;
> diagnóstico de forma sobre los `not_found`; expansión determinista de referencias (`xref=clause`) y
> `bounded` como defaults, ambos por **excepción de producto registrada**; primitiva `get_clause`;
> etiquetas de tabla en la base; herramienta de verificación independiente (protocolo 1.2).
> `uv run pytest` **1419** verde (1280 al empezar).

---

## 1. El arco de S12

La teoría (seis artículos) planteaba una pregunta: ¿necesita RAG-ECSS una capa de decisión? La
respuesta de la sesión es **no, todavía**. Pero la forma de llegar a ella dejó más que la propia
respuesta.

0. **pre-S12 — las dos deudas que M5 iba a romper.** El historial pasa a ser una lista de turnos
   (cada turno, una lista de mensajes; un único aplanado hacia el proveedor), demostrado con
   equivalencia byte a byte fijada **antes** del refactor. La suite bloquea la red y **falla en el
   teardown**, porque el fallback y los errores tipados del servicio se tragaban la excepción.
1. **Brief 1 — diagnosticar la forma antes de orquestar.** El recall de la referencia (0,91) no medía
   la forma: es condicionado a lo inyectado. La señal estaba en los 65 `not_found`, que se agruparon
   en 27 aspectos y se clasificaron contra el corpus con evidencia literal (G / B1–B4 / A / C).
   Resultado tras la UAT: **B1 (puntero desde lo inyectado) en 9 consultas; B2 (fan-out) en 0.** Hay
   forma reactiva, de un salto y casi siempre explícita.
2. **Brief 2 — lo determinista primero.** Parser de referencias + `get_clause` + tope de 5
   cláusulas, apagado por defecto, medido en cuatro configuraciones. **Ninguna pasó el criterio
   preregistrado**, por dos errores de especificación del propio criterio (§2) y un coste real
   (latencia +30 % combinado).
3. **Briefs 3–5 — el instrumento y las decisiones.** La verificación independiente (construida fuera
   de brief, registrada como excepción) pasa a 1.1 (orden de fases por flujo) y a 1.2 (puntero
   literal comprobado con el parser, respuesta principal frente a relacionada, `answered_as_phrased`).
   `bounded` y `xref` pasan a ser defaults por **prioridad de producto**, con las excepciones
   escritas, no como relecturas del criterio.

---

## 2. Decisiones contrastadas con la implementación

### Confirmadas

- **El pipeline (workflow) es el estado por defecto.** Ningún dato de la sesión justificó el bucle.
  El contenido que faltaba llega por la vía determinista (q01 y q09 resueltos 3/3; q06-a1 y q11
  llegan y se citan por `[n]`).
- **Invariantes en código, preferencias en el prompt.** El Critic y el Boss siguen siendo código. La
  verificación literal sigue fuera de cualquier bucle futuro.
- **El historial es una lista de turnos, no de mensajes.** El refactor tocó once sitios (no tres) y
  cerró la deuda `-2N` sin mover un byte del array de mensajes.
- **El coste no bloquea un RAG de bajo volumen; la latencia sí.** Una pasada de cuatro
  configuraciones costó 0,38 USD. Lo que pesó en la decisión fue la latencia, y al final la calidad
  pesó más.

### Refinadas

- **"Forma variable" → "el siguiente paso depende de lo observado".** Lo confirma el diagnóstico (B1
  = 9, B2 = 0), con un matiz que la teoría no daba: la dependencia reactiva de **un salto con puntero
  explícito** se resuelve sin bucle. El bucle solo conserva un hueco hipotético: los punteros a un
  estándar entero, donde hay que reformular la búsqueda dentro del destino (H-h, sin medir).
- **El techo de latencia pasa de guarda a objetivo.** Decisión de Salva: en un producto de
  trazabilidad normativa no se mantiene la latencia baja a costa de respuestas peores. La latencia
  se ataca por otras vías (UI de progreso, razonamiento `low`, caché). Referencia nueva: mediana
  16,8 s, p95 24,0 s.
- **La etiqueta ➕ distingue el origen** («by cross-reference» frente a «to complete the clause»)
  sin tocar el contrato, deduciéndolo en la UI. Solo vale mientras haya dos mecanismos de expansión.

### Refutadas o corregidas por la medida

- **"Los misses del recall dirán si hay forma"** (hipótesis del chat en el paso 1): falsa. El recall
  condicionado mide omisiones del generador sobre su propio contexto.
- **El criterio del brief 2 estaba mal especificado en dos puntos:**
  - "Resuelto" exigía cita literal, pero el producto acepta "cita o paráfrasis" con `[n]` desde S11.
    Estricto: 2/6; por `[n]`: 4/6.
  - La guarda de recall usaba el recall condicionado, que **castiga traer contexto relevante**: el
    denominador creció un 38 % y el numerador un 19 % (en absoluto, 37 citas relevantes más). Parte
    de la caída era sustitución real (22 citas a cláusulas recuperadas), que en q09 fue cambiar
    handbooks por la normativa que esos handbooks citaban.
- **Verificación 1.0/1.1: la traducción mecánica a categorías daba falsos B1** en los 5 casos "sin
  puntero". Causas: destinos de puntero inferidos y no literales (V3, V7), listas de respuestas
  sobreinclusivas (V6) y juzgar el tema de fondo en vez del hueco formulado (V4). Los `assessment` en
  prosa coincidían con el diagnóstico. Con 1.2, todos los falsos B1 desaparecen.
- **"Las viñetas `not_found` del SDP son forzadas"** (UAT de pre-S12): solo en parte. Lo pedido (la
  definición y el contenido del SDP) existe en el corpus (anexo O, DRD) y no se recuperó. Además, la
  consulta arrastraba la pregunta anterior sobre seguridad. Con `bounded`, 9–11 requisitos citados y
  sin `not_found` (con `orphans`, 4).
- **`pytest-socket` como bloqueo de red:** no funciona en Windows (el event loop crea su self-pipe
  con un socket TCP por loopback). Medido, no supuesto: 26 fallos y 17 errores. Se usó un fixture
  propio.

---

## 3. Números medidos

**Diagnóstico de forma (referencia `generation_20261019-141324`, 14 consultas × 3).** 33/42
ejecuciones servidas `partial`; 65 líneas `not_found` → 27 aspectos en 13 consultas. Tras las UAT,
por consultas distintas: **B1 9** (8 sin el espurio de q10), B2 0, B3 1, B4 3, G 2, A 1, C 1. Por
tipo de puntero B1: 5 consultas con puntero a cláusula, 4 con puntero a estándar entero.

**Brief 2 (`generation_20261019-224307`, pareado con `control`; `control` dentro del ruido de la
referencia):**

| | `control` | `xref` | `bounded` | `xref+bounded` |
|---|---|---|---|---|
| Aspectos `xref` resueltos ≥ 2/3, estricto / por `[n]` | 0 | 2 / 4 | 0 | 2 / 4 |
| `same_clause` resueltos, estricto / por `[n]` | 0 | 0 | 0 / 2 | 0 / 2 |
| Recall | 0,90 | 0,82 | 0,91 | 0,86 |
| Precisión (sin solo-expansión) | 0,86 | 0,87 | 0,90 | 0,90 |
| `verified` | 1,00 | 1,00 | 1,00 | 1,00 |
| Latencia mediana / p95 (s) | 12,9 / 19,6 | 13,9 / 22,8 | 12,7 / 22,9 | 16,8 / 24,0 |
| Coste por turno (USD) | 0,00179 | 0,00198 | 0,00200 | 0,00258 |
| Tokens de entrada (mediana) | 6.624 | 8.658 | 8.192 | 10.546 |

**Otros:**
- Barrido de referencias en lo inyectado: 117 distintas.
- Etiquetas de tabla: la regla ingenua (n-ésima cláusula con tablas) falla 659/1.024; la regla por
  cláusula de primer nivel valida 43/43 contra los índices de tablas de los DOCX; 1.035 etiquetas
  en la base.
- Verificación 1.2 (V8–V13): coinciden V8 y V10; V9, V11, V12 y V13 salen "sin decidir" (`B2?`,
  que se resuelve como B4 por el rango medido); ningún falso B1.
- Coste total de llamadas reales del producto en la sesión: ≈ 0,40 USD.

---

## 4. Aprendizajes agnósticos (candidatos a Guide)

1. **Árbol tarea → workflow → workflow dinámico → agente.** Falta un escalón en el material: el plan
   lo fija una llamada y lo ejecuta el código (orchestrator-workers). Criterio para el bucle: **el
   siguiente paso depende de observar el resultado del anterior** y la dependencia no se resuelve con
   un salto determinista.
2. **Diagnosticar la forma antes de orquestar.** Clasificar los "no encontrado" contra el corpus con
   una taxonomía que separe *estaba en el contexto* (omisión del generador), *alcanzable por puntero*,
   *por fan-out*, *por ranking*, *no alcanzable*, *fuera del corpus* y *no existe*. Solo B1 y B2
   justifican orquestar; G pide prompt o modelo; B3, ranking.
3. **Una señal reactiva de un salto con puntero explícito se resuelve de forma determinista** (parser
   + consulta por ID), sin LLM ni bucle. Es la primera rama a probar.
4. **Invariantes en código, preferencias en las descripciones.** Apareció como antipatrón en dos
   artículos ("Always run validate…", "only call this after…"): lo que debe ocurrir siempre no se le
   pide al modelo.
5. **Refactor estructural ⇒ equivalencia byte a byte fijada antes de tocar el código.** Sustituye una
   pasada del arnés, es más fuerte y no cuesta nada.
6. **Un bloqueo de red en tests tiene que fallar en el teardown**, no solo lanzar una excepción: un
   sistema tolerante a fallos (fallback, errores tipados) se la traga y el test pasa.
7. **Una métrica condicionada a lo inyectado penaliza cualquier técnica que añada contexto
   relevante.** Para expansión, reranking o grafos: recall sobre lo recuperado más recuentos
   absolutos, no un ratio cuyo denominador mueve la propia técnica.
8. **El criterio de éxito tiene que medir el comportamiento que el producto acepta.** Si es más
   estricto (cita literal cuando se admite paráfrasis), el rechazo lo produce el instrumento.
9. **El preregistro protege contra el sesgo, no contra un criterio mal especificado.** Cuando el
   criterio resulta mal especificado, no se relee: se adopta (o no) por **excepción explícita** con
   su razón, y se preregistran las métricas corregidas para datos nuevos.
10. **Verificación independiente de juicios:** sesión ciega (sin código ni clasificación); el
    verificador da **hechos, no categorías**, y la traducción la hace el código; citas comprobadas
    por código; orden de fases garantizado por el flujo (`reveal`), no pedido; punteros comprobados
    como literales con el mismo parser del producto; respuestas principales frente a relacionadas;
    "¿existe tal como se formuló?". **La independencia es de contexto, no de modelo**: hace falta un
    ancla humana sobre discrepancias más una muestra.
11. **Sesgo de confirmación al clasificar:** buscar punteros solo hacia donde se cree que está la
    respuesta sesga hacia "sin puntero". El protocolo exige leer todos.
12. **Calidad de respuesta antes que latencia** en un producto de trazabilidad (condición de
    aplicabilidad: el valor está en la respuesta correcta y citada, no en la interactividad). La
    latencia se trata como objetivo con sus propias palancas.

**Teoría sin implementar (no se gradúa todavía; queda como provisional para S13):** estado del bucle
en el cliente frente a `previous_response_id`; razonamiento no portable entre proveedores (fallback
solo antes de la primera vuelta); `submit_answer` como salida final y degradada; límites
multidimensionales (vueltas, tool calls, tokens acumulados, tiempo); tres clases de error;
crecimiento cuadrático del contexto; tensión entre caché de prefijo y compactación.

---

## 5. Aprendizajes de dominio (ECSS)

- **Los handbooks citan la normativa por número.** Sin seguir esas referencias, preguntas genéricas
  ("¿qué exige ECSS sobre reutilización?") se responden solo desde guías informativas. Es la
  justificación de producto de `xref`.
- **Numeración de tablas:** dentro de cada cláusula de primer nivel o anexo, en orden de documento
  (`D-2` = la segunda del anexo D), contando también las descartadas.
- **Relaciones implícitas que ningún texto nombra:** cada cláusula de Q-ST-80C tiene su fila en la
  matriz de aplicabilidad D.2 (por categoría A–D), y la cláusula no remite a ella; los requisitos
  remiten a sus DRD por la salida esperada.
- **Nombres de documento como punteros:** SCF → M-ST-40C anexo E, SCMP → anexo A.
- **ECSS gradúa por criticidad, no por segmento:** no hay requisitos "específicos de vuelo" de
  reutilización.
- **Delegaciones:** varios huecos (criterios de aceptación de riesgos) no son ausencias: ECSS los
  delega en un documento del proyecto (M-ST-80C A.2.1). La respuesta útil es "ECSS no lo fija; lo
  delega en X", con cita.
- **Referencias a documentos externos:** la guía ISVV de ESA (anexos D/E), Q-ST-30, SAVOIR; las
  menciones a un "anexo D" pueden ser de otro documento.

---

## 6. Abierto, con anclaje

- **Grafo de referencias — puerta antes de M6** (también en las instrucciones del proyecto). Tres
  tipos de arista: explícitas (hoy `xref`, al vuelo y de un salto), estructurales (fila D.2,
  padre/hermana, requisito ↔ DRD) y por nombre (SCF, SCMP); más la preferencia por autoridad
  normativa frente a handbook. Partida: `references.json` y `graph.graphml` de S06 y el parser de
  `xref`. **Prerrequisito:** el origen de cada chunk añadido tiene que ir explícito en el contrato
  (`origin = xref | structural | name …`); hoy la UI lo deduce y dejará de poder hacerlo.
- **Ampliación del golden set (protocolo 1.2)**, con métricas ya preregistradas: "resuelto" = chunk
  respuesta citado por `[n]` (literal aparte); recall sobre relevantes recuperadas + recuentos
  absolutos. Estratos nuevos: referencias cruzadas, preguntas explicativas, respuestas solo con
  handbook cuando existe normativa citada. Ahí se confirman las excepciones de `bounded` y `xref`.
- **Prompt por tipo de pregunta** (explicativa / enumerativa / requisito), afirmaciones de delegación
  en lugar de `not_found`, y la tensión con la cita literal para requisitos (H-g).
- **Topes por cláusula frente a chunks de tabla** (H-f): cortan q06-a2 y q09-a3.
- **La consulta de retrieval arrastra las preguntas previas aunque cambie el tema.**
- **Latencia:** UI de progreso, razonamiento `low`, caché.
- **Punteros a un estándar entero** (q08, q10, q12, q14): el único hueco donde un bucle aportaría
  algo; candidato natural para el grafo o para S13.
- **Para S13 (orquestación):** con la arquitectura actual no hay bucle que orquestar. La comparación
  framework frente a orquestación interna se evalúa sobre el criterio de §4.1, y sobre si el grafo
  necesita recorrido de varios saltos dirigido por el modelo.

---

## 7. Sesión en vivo

Mensaje central: no convertir un flujo en agéntico por hype; solo cuando aporte una ventaja real
frente a lo determinista, y considerando cómo quieren interactuar los usuarios. Validación humana en
procesos sensibles, diseño conservador, ni un agente que lo haga todo ni demasiados agentes. La
terminología (context, loop y graph engineering) se solapa: importa el mecanismo, no la etiqueta. En
producción, según el mentor: servicios en Python con LangChain/LangGraph (opinión, no dato; se decide
en S13). Aislamiento de permisos por contenedor (mínimo privilegio llevado a infraestructura; S14).

---

## 8. Proceso

- **Excepciones registradas:** la herramienta de verificación se construyó fuera de brief
  (registrada en `S12-decisions-3.md`); `bounded` y `xref` como defaults fuera del criterio
  (`ARCHITECTURE.md` §5, junto a la precisión de Haiku 5.5 en S11).
- **"CC no itera":** se respetó en el brief 2. El reporte con los "por qué no" (llega y se
  parafrasea, recortado por topes, no detectado) fue lo que permitió decidir.
- **El criterio mal especificado fue del chat.** Lección propia: escribir el criterio de éxito desde
  la política de aceptación del producto, y comprobar que la métrica no la mueve la propia técnica.
- **La UAT humana corrigió en las dos direcciones:** Salva detectó la mala calidad de q09 que las
  métricas no veían (V10) y el chat corrigió dos lecturas de la UAT (V9: puntero literal que no lleva
  a la respuesta; V12: el anexo D era de la guía ISVV). El ancla humana sigue siendo necesaria.
- **Flujo de verificación vigente:** `new → fase 1 → reveal → fase 2 → check`; revisión humana de
  las discrepancias más 2–3 al azar.
