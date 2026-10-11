# S09 — Consolidado de sesión (histórico, no se reescribe)

> **Paso 5 del ciclo.** Combina el resumen provisional (`S09-resumen.md`, paso 2) con lo aprendido al
> implementar los cuatro briefs (`S09-decisions*.md`, `S09-uat*.md`). La **divergencia teoría↔medida**
> se conserva a propósito: es el material de Guide. La graduación a `RAG-GUIDE.md` (paso 6) se hace
> **desde** este documento.
>
> **Sesión:** M4 / S09 — Fundamentos RAG, retrieval y capa de datos como servicio. **Cerrada con:**
> M2 cableado — `/query` y `/query/stream` responden sobre el corpus completo en pgvector, con citas
> por cláusula, fan-out real, validación literal en modo observación y retrieval conversacional.
> `uv run pytest` 1017 verde (923 al empezar).

---

## 1. El arco de S09

Entramos con M2 construido y medido pero sin cablear, y cuatro decisiones: por dónde entra el
retrieval en la generación, cómo se citan k chunks, si se recompone la cláusula y qué modo de
retrieval alimenta al generador. Salimos con un seam de contexto inyectable que respeta el
invariante de guardrails, la **cláusula como unidad de contexto y de cita**, handles opacos `[n]`
con fan-out por marcador, citas literales verificadas por substring y la query de retrieval
compuesta con el historial.

El brief principal cumplió el objetivo. Los tres siguientes salieron de la UAT, y cada uno corrigió
algo que una decisión anterior había roto o dejado corto: el historial no llegaba al retrieval (E1),
la regla anti-cita-fantasma sobrevivió a su premisa (E4), la frase de rechazo nueva afirmaba
ausencias falsas (F1) y el formato de cita del historial se convirtió en un ejemplo que el modelo
imitaba (G1). **Cuatro de los aprendizajes más valiosos son correcciones a decisiones propias.**

---

## 2. Decisiones contrastadas con la implementación

### Confirmadas

- **`llm_service` no conoce la base.** El proveedor de contexto se inyecta por `Depends` en el router
  y se invoca dentro de `_prepare`, después de guardrails y sesión. Test de capas en subproceso
  (en el proceso de la suite todo ya está importado).
- **Fail-closed sin contexto.** Base parada → 503 / `event: error`. La verificación contra la pila
  real encontró que `gaierror` es `OSError`, no `SQLAlchemyError`: el test con mock no lo habría
  visto.
- **Cláusula como unidad.** Agrupar colapsa clónicos y da la cita que promete el producto.
- **Fan-out por marcador.** 1 de 7 cláusulas inyectadas citada en el caso B/C. `citations` y
  `retrieved_context` dejan de coincidir, como preveía el contrato.
- **Validación literal normalizada.** 0 `not_found` en toda la UAT; la normalización (NFKC, guiones
  de partición, comillas tipográficas, puntuación final) no produjo falsos positivos.
- **`vector` al generador.** No se reabrió.
- **Concatenar preguntas previas sin LLM (E1, N=2).** Seguimientos anafóricos y encadenados
  resueltos; sin contaminación observada en el cambio de tema (un caso de CC y la UAT del usuario).

### Refinadas

- **Recomponer la cláusula: medido y no aplicado.** 70 % de las cláusulas recuperadas llegan
  incompletas. Expandir siempre multiplicaría la mediana del bloque ×4 y la cola ×5 (anexos DRD de
  hasta 2.615 tokens). La expansión **acotada** queda justificada, con un caso prioritario: la
  **nota huérfana** (una remisión de 8 tokens sin el requisito al que acompaña).
- **Seam: el proveedor recibe una query, no `(question, history)`.** La composición vive en
  `llm_service`; el proveedor no sabe que existe una conversación.
- **`RetrievedContext` en vez de una lista.** Lleva modo, `top_k` y dos latencias, para emitir un
  solo `retrieval_completed` desde quien conoce los grupos.
- **Embedding y ranking separados** (`embed_query` + `rank`): cada latencia se mide aparte y un
  fallo de embedding no abre sesión de base.

### Refutadas o corregidas por la medida

- **"Reformulación diferida: queries cortas y en inglés" — equivocada.** El resumen dejaba fuera la
  condición *entrada conversacional*. La UAT lo demostró: «And what does Q-ST-80C say about it?» no
  recuperaba nada porque el retrieval no veía el historial. Diseñado en M1 para el generador, el
  historial en RAG tiene que llegar **también** al retriever.
- **"Un rechazo vacía `citations`" — regla puente que sobrevivió a su premisa.** Protegía de la
  cita fantasma cuando las citas salían de lo inyectado. Con el fan-out por marcadores ya no hacía
  falta, y producía `[3]` en el texto sin fuente debajo (4 de 6 rechazos de la UAT citaban).
- **E5 "Not covered by the indexed ECSS standards" — falsa por construcción.** Afirma ausencia en el
  corpus entero cuando el modelo solo ve k chunks. Medido: «E-ST-40C is not included in the indexed
  material», con E-ST-40C indexado. F1 lo sustituye por una frase sobre **la búsqueda**.
- **D11 "historial con citas naturales" — corregido por G1.** Evitaba que un `[1]` antiguo apuntara
  a otra cláusula, pero el modelo imitó el formato y abandonó `[n]` (1/6, respuesta sin
  trazabilidad). Borrar las marcas resuelve los dos problemas: 6/6 con `[n]` válidos tras G1.
- **E6 "respuesta parcial por prompt" — no se cumple.** Dos redacciones; Haiku sigue abriendo con la
  frase de rechazo cuando falta el estándar nombrado. Iteración cortada en una, como se acordó.

---

## 3. Números medidos

- **Cobertura de cláusula** (8 queries, `vector`, top_k 10): 54 pares query–cláusula, **70,4 %
  incompletos**. Cláusula completa: mediana 319 tokens, p90 1.737, máx 2.615. Grupo recuperado:
  mediana 82, p90 487.
- **Latencias:** embedding 177–697 ms (frío el primero), base 75–185 ms. Bloque de contexto 954–3.298
  tokens estimados.
- **Validación literal:** 0 marcadores inválidos, 0 `not_found` en la UAT de S09-1.
- **Soporte (tabla D-1, «And for C?»):**

  | | S09-3 (historial con citas) | S09-4 (historial sin marcas) |
  |---|---|---|
  | Correcta y completa | 2/6 | 4/6 |
  | Correcta, incompleta | 2/6 | 1/6 |
  | Atribución errónea | 2/6 | 1/6 |

  Con 6 por lado no es una diferencia medida. Lo que sí está medido: **hay atribuciones falsas que
  pasan pertenencia y literalidad.**
- **Coste:** smokes y repeticiones de los cuatro briefs ≈ 0,12 USD en total.
- **Suite:** 923 → 1017 tests.

---

## 4. Aprendizajes agnósticos (candidatos a Guide)

1. **En RAG multi-turn, el historial alimenta al retriever, no solo al generador.** Condición:
   producto conversacional. Primera opción: concatenar preguntas previas del usuario (nunca
   respuestas). Señal para escalar a condensación con LLM: contaminación observada por cambio de
   tema.
2. **Pertenencia ≠ literalidad ≠ soporte.** Tres niveles de validación de cita; los dos primeros son
   deterministas y baratos, el tercero no. Una cita literal y bien referenciada puede estar atribuida
   a la entidad equivocada (filas casi idénticas de una tabla).
3. **El historial también es prompt.** Cualquier formato guardado en turnos previos es un ejemplo
   que el modelo imita. Guardar el turno sin artefactos de formato que no deban repetirse.
4. **Una afirmación de ausencia solo es honesta sobre lo que el sistema verificó.** Puede hablar de
   la búsqueda, nunca del corpus: su veracidad depende del recall, y el recall no es 1.
5. **Las reglas puente se revisan cuando cambia su premisa.** Una protección diseñada para un
   mecanismo (citas desde lo inyectado) se vuelve un bug con el mecanismo siguiente (citas desde
   marcadores).
6. **Con modelos pequeños, una instrucción de formato exacto gana a una regla de juicio**, y las
   prohibiciones se cumplen en la letra. Una iteración de prompt medida es el límite útil; después,
   la palanca es estructural (contrato de salida).
7. **Unidad de contexto = unidad citable** en corpus jerárquicos. Resuelve los clónicos sin MMR.
8. **Recomposición: medir antes de expandir.** La expansión completa a la unidad padre puede
   multiplicar el contexto; la acotada (p. ej. nota huérfana → su requisito) es la candidata.
9. **Seam de contexto inyectable dentro del punto de preparación**, no retrieval en el router:
   conserva el invariante "guardrails antes del primer efecto" y desacopla el servicio LLM de la
   base. El proveedor gestiona su propia sesión corta.
10. **Un seam fail-closed se verifica contra la pila real.** Los errores de red de un driver pueden
    no heredar de la jerarquía de la librería.
11. **El header contextual tiene coste en chunks diminutos:** el vector representa el título de la
    sección, no el contenido. Condición: chunks muy cortos bajo títulos ricos.
12. **Escapar atributos, no el texto citable**, si la cita se valida contra el verbatim.
13. **La voz del prompt se filtra al producto.** Lo que el usuario puede leer se redacta en términos
    del producto, no de la tubería.
14. **Seguridad de la capa de datos de un RAG = autorización en el retrieval** (teoría, no
    implementado: corpus público). Rate limiting y autenticación según exposición y consumidor; en un
    MVP sin exposición, un tope de gasto en el proveedor cubre el riesgo.

## 5. Aprendizajes de dominio (ECSS)

- **Peso normativo como regla de lectura en el system prompt** (shall/should/may; NOTE no
  vinculante): el CAG dentro del RAG son reglas de lectura, no contenido.
- **Los identificadores de estándar no se recuperan en modo `vector`** (E-ST-40C no aparece para una
  pregunta que lo nombra). Es la causa raíz de los casos 5/6 de la UAT.
- **Tablas de filas casi idénticas** (D-1: «Software criticality category: X. Definition: …")
  favorecen la atribución cruzada. Pista para el ítem abierto de aplanado de tablas, sin verificar.
- **Los handbooks se recuperan pero no se citan** en preguntas sobre qué *aplica*: coherente con la
  regla de autoridad.

---

## 6. Abierto, con anclaje

| Ítem | Sesión |
|---|---|
| Recuperar el estándar nombrado en la query (ponderación híbrida / filtro por ID) | S10 |
| Expansión de cláusula acotada; nota huérfana | S10 |
| Referencias cruzadas (grafo, sin validar) | S10+ |
| Respuestas parciales con frase de rechazo delante (E6): contrato de salida tipado | S11 |
| Frases sin cita y afirmaciones de ausencia en el cuerpo (relleno de preentrenamiento) | S11 |
| Soporte de la cita y política ante cita no soportada (1–2/6 en D-1) | S11 |
| Aplanado de tablas: la etiqueta de fila se separa de su definición | Con S10/S11 |
| Condensación de seguimientos con LLM | Sin sesión; señal: contaminación en uso real |
| Filtro por área de conocimiento | Cuando el corpus salga de software |
| `/search` como funcionalidad de producto | Hipótesis; señal: usuarios que van directos a las citas |

## 7. Sesión en vivo

No revisada en esta sesión.

## 8. Proceso

- **Transporte directo:** desde S09 el chat lee y escribe en `RAG-ECSS` y `RAG-Guide` a través de la
  app de escritorio. Incorporado a las instrucciones del proyecto.
- **Criterio de corte de una sesión:** se arregla ahora lo que hace que el producto diga algo falso o
  lo que es trivial e introducido por la sesión; lo estructural va a la sesión del máster que lo
  cubre.
