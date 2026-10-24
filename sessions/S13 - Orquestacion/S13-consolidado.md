# S13 — Consolidado de sesión (histórico, no se reescribe)

> **Paso 5 del ciclo.** Combina el resumen provisional (`S13-resumen.md`, paso 2) con lo aprendido al
> implementar los tres briefs (`S13-decisions*.md`, `S13-uat*.md`). La **divergencia teoría↔medida**
> se conserva a propósito: es el material de Guide. La graduación a `RAG-GUIDE.md` (paso 6) se hace
> **desde** este documento.
>
> **Sesión:** M5 / S13 — Orquestación: frameworks frente a orquestación interna, LangGraph, estado,
> errores. **Cerrada con:** sin framework de orquestación; desglose de latencia por etapa como
> instrumento permanente; punteros a estándar entero implementados y **no adoptados**
> (`clause_standard`, apagado); traza de punteros agrupada por documento. Sin bucle agéntico: el
> último candidato (reformular dentro del estándar destino) se queda sin casos. `uv run pytest`
> **1477** verde (1419 al empezar).

---

## 1. El arco de S13

La teoría (seis artículos) justifica un framework de grafos para un flujo que no tenemos. La sesión
lo confirmó y midió los dos huecos que quedaban abiertos de S12.

1. **Teoría — ¿qué forma tiene el flujo?** El pipeline es una cadena con una arista condicional al
   final, petición-respuesta de ~17 s, sin pausas. De las cuatro primitivas de LangGraph ya tiene
   tres (estado tipado, nodos, aristas condicionales como `if`); le falta el checkpointer, que no
   tiene caso. Por la propia escalera del material: nada que orquestar.
2. **Brief 1 — medir antes de paralelizar.** Desglose de latencia por etapa sobre structlog, fuera
   del contrato. Sin llamadas reales; lo midió la pasada del brief 2.
3. **Brief 2 — la arista condicional que faltaba.** Puntero a cláusula → `get_clause` (ya existía);
   puntero a estándar entero → retrieval con la misma consulta filtrado a ese estándar. Una
   comprobación sin generación anticipó que el criterio no se podía cumplir (techo 2 de 6); la pasada
   dio 1 de 6. **No se adopta**, sin excepción.
4. **Brief 3 — instrumento del grafo.** La traza de punteros se agrupa por documento en la
   presentación, sin tocar la selección.

---

## 2. Decisiones contrastadas con la implementación

### Confirmadas

- **Sin framework de orquestación.** Ningún dato de la sesión pidió persistencia entre pasos,
  pausa ni reanudación. La condición es la forma del flujo; la señal que lo cambiaría es necesitar
  persistencia **entre pasos de una petición** (no entre turnos).
- **Lo determinista primero.** La arista de estándar entero no necesita LLM: reutiliza el vector de
  la petición (0 embeddings nuevos, comprobado por test) y cuesta **+68 ms** de mediana en `xref`.
- **Las métricas corregidas de S12 funcionan.** Recall sobre relevantes recuperadas (0,84 → 0,87) y
  recuentos absolutos (238 → 295 relevantes citadas) cuentan lo que el recall condicionado
  (0,836 → 0,846) no distingue.
- **Preregistro + "CC no itera".** La pasada no se ajustó a la comprobación previa ni al resultado;
  el desglose de "por qué no" decidió el paso siguiente.

### Refinadas

- **"Medir antes de paralelizar" con cifras.** `llm` es el **93–94 %** del turno (15,3 s de 16,6 s
  de mediana). Retrieval, expansión y `xref` juntos no llegan a 0,3 s. Paralelizarlos ahorraría
  cientos de ms sobre 17 s. Las palancas de latencia están en la generación: razonamiento `low`,
  longitud de salida, UI de progreso.
- **Seguir un puntero no implica que el destino sea relevante.** El retrieval filtrado siempre
  devuelve algo (no hay umbral de distancia). Una remisión genérica (Q-ST-80C 5.3.1 → M-ST-80) se
  sigue en cualquier consulta que inyecte esa cláusula. En la pasada: 207 cláusulas añadidas solo
  por estándares en 10 de 14 consultas, 100 citadas (71 relevantes, 14 no, 15 sin juicio); en q01 y
  q03 no se cita ninguna.
- **Plazas compartidas en orden de aparición matan de hambre a los punteros tardíos.** q14-a1 tiene
  la respuesta en el rango 2 de E-ST-70-01C, pero E-ST-70-41C nombra cinco cláusulas antes que el
  estándar. El brief aceptaba de antemano que el rango 9 y el 15 no entrarían; **q14 no estaba
  previsto**.

### Refutadas o corregidas por la medida

- **"La reformulación dentro del estándar es el sitio del bucle"** (H-b/H-h de S12): sin casos. De
  5 aspectos no resueltos, 3 son política de plazas, 1 de generación y 1 de consulta (q12-a3), que
  es una delegación clasificada C en S12. **No queda candidato a bucle en el producto.**
- **"`guardrails` + `embedding` son el 25–30 % de la latencia"** (UAT del brief 1, N=2): arranque en
  frío. En la pasada son 480 ms y 258 ms de mediana. El candidato a paralelizar guardrails con
  embedding (que además rozaba el invariante "guardrails antes de cualquier efecto") se descarta.
- **`db_ms` cambió de significado.** Antes medía todo el bloque de base (ranking + expansión +
  referencia); desde el brief 1 mide solo el ranking. Un instrumento que reutiliza un nombre
  existente tiene que decir que lo redefine.
- **UAT 2, caso 3:** la premisa "consulta sin punteros a estándar" era falsa (Q-ST-80C 5.3.1 nombra
  M-ST-80). Y CC atribuyó a esa consulta 60 entradas de traza que eran de otra petición, corregido
  en el brief 3. No se pudo identificar cuál: el log guarda metadatos, no la pregunta, y se perdió al
  recrear el contenedor.

---

## 3. Números medidos

**Pasada `generation_20261023-191136`** (Haiku 5.5, 14 × 3 por configuración; 84/84 comparables,
0 fallback, 0 errores, 0 reintentos de esquema, 0 recortes, 0 rechazos con citas; 0,249 USD).

`control` (`xref+bounded`) frente a la referencia `generation_20261019-224307`: recall 0,861 →
0,836, precisión 0,876 → 0,862, `verified` 1,00. Dentro del ruido.

**Criterio del brief 2** (pareado `standard` frente a `control`):

| # | Guarda | `control` | `standard` | ¿Cumple? |
|---|---|---|---|---|
| 1 | ≥ 3 de 6 aspectos resueltos por `[n]` en ≥ 2/3 | 0 | 1 (q12-a2) | no |
| 2a | Recall sobre relevantes recuperadas, −0,03 como mucho | 0,84 | 0,87 | sí |
| 2b | Relevantes citadas en absoluto no bajan | 238 | 295 | sí |
| 3 | `verified` 1,00 | 1,00 | 1,00 | sí |
| 4 | Precisión sin solo-expansión, −0,03 como mucho | 0,88 | 0,92 | sí |

Reportado: latencia mediana / p95 16,6 / 23,7 s → 17,8 / 27,2 s; coste por turno +7 %; tokens de
entrada 10.546 → 10.570; `not_found` por ejecución 1,19 → 1,50; precisión estándar 0,86 → 0,86.

**Aspectos:** q12-a2 3/3 (cita M-ST-80C 6.1); q12-a1 1/3 (`brought_not_cited`); q12-a3 y q08-a2
`not_in_slot` (rango > 50 y 15); q12-a4 `not_in_slot` (rango 9, la plaza se llena en el 5); q14-a1
`over_limit` (rango 2). La comprobación previa sin generación marcaba el mismo techo (2 de 6).

**Latencia por etapa** (mediana, 42 ejecuciones por configuración):

| Etapa | `control` | `standard` |
|---|---|---|
| `llm` | 15.297 ms (93,1 %) | 16.873 ms (93,8 %) |
| `guardrails` | 480 ms (p95 1.543) | 484 ms |
| `embedding` | 258 ms | 264 ms |
| `db` | 133 ms | 124 ms |
| `xref` | 60 ms (p95 92) | 128 ms (p95 347) |
| `expansion`, `prompt`, `validation`, `compose` | < 5 ms | < 5 ms |
| `unaccounted` | 4,0 ms | 4,7 ms |

Los +1,2 s del total salen de `llm` (+1,6 s: más contexto y más salida), no de la etapa nueva.

**Otros:** tests 1419 → 1449 → 1470 → 1477. Traza agrupada: 1.200 → 992 entradas en el artefacto,
misma puntuación. Coste de llamadas reales de la sesión ≈ 0,26 USD.

---

## 4. Aprendizajes agnósticos (candidatos a Guide)

1. **Framework de orquestación: decidir por la forma del flujo.** Los disparadores reales son
   persistir y reanudar, pausar para un humano y observabilidad por nodo. Ramas (un `if`) y
   paralelismo (`gather`) no lo son. Un grafo con aristas fijas es un workflow, y todo pipeline ya
   es un grafo implícito: tener un grafo no implica necesitar un framework de grafos.
2. **Checkpointer: persistencia entre pasos, no entre turnos.** El historial por turnos ya es
   memoria corta (`session_id` ≡ `thread_id`). El checkpointer se gana su sitio cuando reanudar a
   mitad vale más que repetir, o cuando hay una pausa a mitad tras trabajo caro o con un aprobador
   distinto del usuario. En un producto conversacional, la puerta humana suele ser un turno.
3. **Memoria larga, condicionada al corpus.** En corpus normativos con trazabilidad obligatoria se
   descarta (texto no verificado en el prompt).
4. **Desglose de latencia por etapa como instrumento base.** Un cronómetro por petición en un
   `ContextVar` (las costuras inyectadas marcan sus etapas sin cambiar su protocolo); un evento
   emitido **siempre** desde un `finally`, también en fallo; `unaccounted_ms` para comprobar que
   suma; la llamada al modelo desglosada por intento; fuera del contrato público.
5. **El arranque en frío falsea una UAT de latencia.** N pequeño con el proceso recién levantado no
   mide; mide la pasada.
6. **Medir antes de paralelizar.** En un RAG con generación de razonamiento, el modelo es > 90 % del
   turno: el paralelismo de retrieval no es palanca de latencia.
7. **Comprobación sin generación antes de pagar la pasada.** Ejecutar solo el retrieval de la
   técnica sobre los casos objetivo acota el éxito alcanzable antes de generar. Si el techo está por
   debajo del criterio, se sabe antes; se mide igual para las guardas y el "por qué no", sin ajustar.
8. **Seguir una referencia no es lo mismo que traer algo relevante.** Un retrieval filtrado sin
   umbral siempre devuelve algo, y una remisión genérica se sigue en cualquier consulta que la
   inyecte. Toda expansión por referencia necesita una puerta de relevancia.
9. **Plazas compartidas en orden de aparición sesgan contra los punteros tardíos.** La prioridad
   entre punteros es una decisión explícita (tipo de arista, autoridad, relevancia), no un efecto
   del orden del texto.
10. **El "por qué no" desglosado decide el paso siguiente.** Separar política de plazas, consulta y
    generación mostró que la llamada extra (reformular) no tenía casos.
11. **Juez de relevancia LLM = paso de workflow, no agente.** Una llamada con salida estructurada
    que filtra. Separa reunir candidatos de decidir qué entra, pero falla descartando en silencio,
    que choca con la asimetría de omisión, y suma una llamada al modelo antes de generar. Escalera:
    umbral de distancia → cross-encoder → LLM, evaluados **offline** contra los juicios de
    relevancia del golden set antes de pagarlos en producción.
12. **Agrupar en la presentación, nunca en la lógica que decide.** Solo se agrupa lo que no tuvo
    efecto; el contexto se fija por digest antes de tocar la traza, y se comprueba que el instrumento
    que la lee puntúa igual.
13. **La premisa de cada caso de UAT se comprueba contra el contexto determinista** antes de
    escribir el caso. Y una atribución en la UAT tiene que poder reconstruirse: si el log no guarda
    la pregunta, la evidencia vive en el artefacto, no en el log del contenedor.

**Teoría sin implementar (no se gradúa todavía):** reducers idempotentes por clave; política de
retención de checkpoints; dominio de fallo del checkpointer (degradable que pasa a no degradable);
`corpus_version` junto a IDs persistidos; `interrupt` (el nodo se re-ejecuta); reintentos en una sola
capa con predicado; degradación visible en el contrato; OpenTelemetry y backend como decisión de
cumplimiento; fallback con estado compatible en la frontera de un nodo, no dentro de un bucle con
tools. Se cruzan con las tres clases de error de S12, que se mantienen (el "error determinista" del
material es la clase **bug interno**).

---

## 5. Aprendizajes de dominio (ECSS)

- **M-ST-80C queda fuera del routing de Q-ST-80C** (H-d de S12). Seguir el puntero resuelve q12-a2
  (M-ST-80C 6.1: los riesgos de software alimentan la gestión de riesgos del proyecto).
- **E-ST-70-41C nombra muchas cláusulas antes que E-ST-70-01** (el estándar de OBCP). Con plazas en
  orden de aparición, el puntero al estándar que responde llega tarde.
- **Remisiones genéricas:** Q-ST-80C 5.3.1 remite a M-ST-80 y arrastra anexos de gestión de riesgos
  a cualquier pregunta que inyecte esa cláusula (p. ej., criticidad).
- **Referencias fuera del corpus recurrentes:** «ECSS-Q-ST-30 clause 5.4» (seis menciones en una
  consulta), ECSS-Q-ST-10-04.
- **Premisa falsa en PUS:** "¿qué servicios son obligatorios?" — en E-ST-70-41C todos son opcionales;
  el sistema lo trata como hueco (`not_found`) en vez de corregir la premisa.
- **q12-a3 (criterios de aceptación de riesgos)** confirma la delegación: ningún retrieval lo trae
  porque M-ST-80C no lo fija.

---

## 6. Abierto, con anclaje

- **Grafo de referencias — puerta antes de M6.** Requisitos nuevos de S13:
  - **puerta de relevancia** antes de seguir un puntero (umbral, cross-encoder o juez LLM; evaluación
    offline primero, §4.11);
  - **prioridad entre punteros** que no sea solo el orden de aparición (tipo de arista, autoridad
    normativa frente a handbook);
  - acumulador de cláusulas **por `clause_id`** con presupuesto (la misma cláusula por dos caminos);
  - `origin` explícito en el contrato (prerrequisito de S12);
  - `corpus_version` junto a IDs si aparece persistencia entre pasos.

  Partida: `clause_standard` implementado y apagado, su traza agrupada y el artefacto
  `generation_20261023-191136` con el "por qué no" por aspecto.
- **Latencia:** palancas en la generación (razonamiento `low`, longitud de salida, UI de progreso).
  `guardrails` es la única otra etapa con p95 > 0,5 s (1,5 s, moderación por red).
- **Candidatos de S13 no ejecutados:** auditoría de capas de reintento (LiteLLM como única capa,
  predicado), degradación visible en el contrato, separación trabajo/control en el orquestador.
- **Diferidos de S12 que siguen vivos:** golden set 1.2 (estrato de referencias cruzadas, donde se
  confirmaría cualquier expansión nueva); prompt por tipo de pregunta, delegaciones y premisas falsas
  (q14 turno 1); topes por cláusula frente a tablas; contaminación de la consulta por cambio de tema.
- **Para S14:** human-in-the-loop operativo. En RAG-ECSS la puerta humana es un turno; el
  checkpointer solo con aprobador distinto del usuario.

---

## 7. Sesión en vivo

LangGraph y LangChain son productos distintos del mismo ecosistema. Multiagente frente a modelo más
potente: especializar cuando la cantidad y diversidad de tareas consume demasiado contexto (S14). Una
tool no es un subagente. El fallback ante caída persistente del proveedor conserva el estado y
continúa: compatible con S12 si el cambio ocurre en la frontera de un nodo, porque el estado propio es
portable y el razonamiento del proveedor no. Consejos: empezar con un agente y añadir complejidad con
datos; el estado es la pieza central; checkpointer no es memoria larga; límites de iteraciones;
reintentos con backoff, fallback y circuit breakers; human-in-the-loop con baja confianza; pasar solo
el estado necesario; observabilidad desde el principio; modelos distintos para desarrollar y revisar
(con nuestro protocolo: la independencia es de contexto, y la diversidad de modelo es una capa más,
no un sustituto del ancla humana).

---

## 8. Proceso

- **Decisión sin excepción:** `clause_standard` no se adopta. Falla la guarda 1 por diseño (W4), no
  por un criterio mal especificado como en S12; las guardas 2–4 a favor no bastan con una señal de
  calidad ambigua (precisión estándar igual, `not_found` al alza). Adoptarlo por calidad habría sido
  releer el criterio.
- **La comprobación previa y el "CC no itera" funcionaron:** el techo se conocía antes de generar y
  no se tocó nada para alcanzarlo.
- **Errores de la UAT, corregidos en ambas direcciones:** el chat dio por buena una premisa (caso 3)
  sin comprobarla contra el contexto; CC atribuyó entradas de traza a la petición equivocada y lo
  corrigió en el brief 3. El script de q14 no describía qué esperar de los turnos 1 y 2.
- **Rama:** `s13-orquestacion-agentes` (no `s13-orquestacion`, como decía el brief 1).
- **Brief de infraestructura:** no hubo `pre-S13`; los tres briefs son de la sesión.
