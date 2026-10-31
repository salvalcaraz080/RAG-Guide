# S14 — Consolidado de sesión (histórico, no se reescribe)

> **Paso 5 del ciclo.** Combina el resumen provisional (`S14-resumen.md`, paso 2) con lo aprendido al
> implementar los tres briefs (`S14-decisions*.md`, `S14-uat*.md`). La **divergencia teoría↔medida**
> se conserva a propósito: es el material de Guide. La graduación a `RAG-GUIDE.md` (paso 6) se hace
> **desde** este documento.
>
> **Sesión:** M5 / S14 — Multi-agente, human-in-the-loop, competición y privilegio de acciones. **Cerrada
> con:** sin multi-agente, supervisor, handoff, competición ni checkpointer (ninguna señal);
> **huecos tipados adoptados** (`GENERATION_GAP_KINDS=on`: `not_in_material` / `premise_contradicted`
> / `delegated`, con cita verificada por código); **degradaciones técnicas en el contrato**
> (`degradations`), con la UI limitada a las cuatro que cambian la confianza; **UAT estructurado**
> (`corpus/groundtruth/uat/`); credencial del retrieval auditada (superusuario, no corregido).
> `uv run pytest` **1554** verde (1477 al empezar).

---

## 1. El arco de S14

La teoría construye un sistema multi-agente completo para un problema (estimación de software) que
tiene juicio, orden variable y acciones con efectos. RAG-ECSS no tiene ninguna de las tres. La sesión
lo confirmó y extrajo lo que sí transfiere sin la maquinaria: la distinción **error ≠ revisión** del
human-in-the-loop, convertida en contrato.

1. **Teoría — una escalera por decisión.** Workflow/agente, uno/varios agentes, enrutado,
   comunicación, puerta humana, competición, seguridad de acciones. RAG-ECSS se queda en el primer
   peldaño de todas por falta de señal.
2. **El diferido "prompt por tipo de pregunta".** Ni agentes ni plantilla por tipo: el tipo de "no
   respuesta normal" (premisa falsa, delegación, hueco real) solo se conoce **después** de leer el
   contexto. Se resuelve con una categoría en el esquema de salida.
3. **Brief 1 — taxonomía del "sin respuesta normal".** (A) `degradations` técnicas en el contrato,
   sin tocar mensajes. (B) huecos tipados tras opción apagada; el modelo declara, el código verifica
   la cita o degrada. Criterio preregistrado cumplido.
4. **Juicio de producto (chat, delegado por Salva).** Adoptar por **error asimétrico**: un tipado
   ausente es el comportamiento de antes; uno espurio dice algo falso. 16/17 tipados servidos
   correctos, 0 premisas espurias. El desplegable de degradaciones sobra en el 93 % de las respuestas.
5. **Brief 2 — adopción.** Default `on`, nueva referencia del arnés, filtro de presentación.
6. **Brief 3 — UAT estructurado + credencial.** El par propuesta–decisión pasa a registro
   consultable; la credencial del retrieval resulta ser superusuario.

---

## 2. Decisiones contrastadas con la implementación

### Confirmadas

- **Sin multi-agente, supervisor, handoff, competición ni checkpointer.** Ninguna señal en §1–§5 del
  resumen. El anclaje de S13 (sin bucle agéntico) se mantiene.
- **La puerta humana es un turno.** El material justifica su checkpointer porque el aprobador es
  distinto del solicitante y hay trabajo caro antes de la pausa; aquí ninguna de las dos.
- **Una categoría en el esquema en vez de reglas en el prompt** cuando el tipo depende del contexto.
  La medida lo sostuvo: los tres casos con señal pasaron de 0/3 a ≥ 2/3 sin tocar retrieval, y
  `verified` siguió en 1,00.
- **El modelo declara, el código prueba.** `premise_contradicted` y `delegated` exigen cita literal
  verificada; si no, se sirven como `not_in_material`. En la pasada: 0 degradados de 17.
- **Error ≠ revisión.** `gap_kind_downgraded` (juicio de contenido) va a `verification`, no a
  `degradations` (técnico). CC lo separó por su cuenta (decisions §1.5.1), coherente con la tesis del
  brief.

### Refinadas

- **El candidato 1 era "solo señales deterministas"; no lo era.** Lo técnico lo detecta el código; los
  tipos de contenido los declara el modelo y los ancla una cita. Corregido antes del brief.
- **El criterio de éxito debía casar por evidencia y por aspecto.** q12-a3 cumplió 3/3 por cláusula
  (5.3.1) pero el aspecto tipado era otro (integración en el riesgo del proyecto, no criterios de
  aceptación). El criterio pasó igual con 2 de 3 casos; la regla queda como lección de preregistro.
- **La precisión del criterio pesaba más que el recall.** El brief puso umbrales simétricos; el juicio
  de producto añadió la asimetría (fallar = comportamiento anterior). Es lo que decidió la adopción
  con q14-a1 en 2 de 5.
- **Inventario de degradaciones.** De 17 eventos técnicos, 7 ya llegaban al contrato; 7 nuevos
  entraron; 3 se excluyeron por ser **política configurada** (ventana de historial, moderación sin
  clave) o no cambiar el contenido (reintentos internos). La "expansión por tiempo" del brief no
  existe.

### Refutadas o corregidas por la medida

- **"La degradación técnica visible" como aviso al usuario.** Con `bounded` + `clause`, las
  truncaciones salen en 13 de 14 consultas: son política, no fallo. Se quedan en el contrato y salen
  de la UI. Solo se muestran `xref_over_limit`, `routing_filter_empty`, `moderation_unavailable` y
  `table_labels_unavailable`.
- **"Pasar de `answered` a `partial` sería una regresión".** En q02 es más honesto: Q-ST-80C delega
  de verdad las no conformidades en Q-ST-10-09. T7 (normalización sin cambios) se mantiene.
- **"El retrieval solo lee".** Por convención del código, no por la credencial: un único rol
  `rag_ecss`, superusuario, dueño de todas las tablas, misma `DATABASE_URL` para retrieval, ingesta
  y migraciones.

---

## 3. Números medidos

Pasada `generation_20261024-140241`, golden set v2, 14 × 3 + q14 turno 1 × 3, 98 llamadas,
**0,2645 USD**, 0 fallback, 0 errores.

| | `control` (`xref+bounded`) | `gaps` |
|---|---|---|
| recall condicionado | 0,835 | 0,843 |
| precisión | 0,892 | **0,864** (−0,028, umbral 0,03) |
| `verified` | 1,00 | 1,00 |
| q14 t1 / q14-a1 / q12-a3 | 0/3 · 0/3 · 0/3 | **2/3 · 2/3 · 3/3** |
| huecos declarados | 66 sin tipo | 40 `not_in_material`, 2 `premise_contradicted`, 15 `delegated` |
| tipados espurios (juicio del chat) | — | 1/17 (`delegated` al Space-to-Ground ICD) |
| latencia mediana / p95 | 16,1 / 27,4 s | 18,3 / 28,0 s (+14 %) |
| tokens de salida (mediana) | 3.866 | 4.282 (+11 %) |
| coste por turno | 0,00268 USD | 0,00299 USD (+12 %) |
| citas fuera de su afirmación | 43/331 | 6/323 |

- **UAT acumulada:** q14 t1 5 de 6; **q14-a1 2 de 5**; 5.3.1 en q12 4 de 5.
- **Control frente a la referencia de S13:** recall −0,026 (dentro del ruido), pero citas fuera de
  su afirmación 25/309 → 43/331 con la misma configuración. Sin explicar; apunta a deriva del
  proveedor.
- **Degradaciones en la pasada:** `expansion_truncated` 36, `xref_truncated` 36, `xref_over_limit` 18;
  13 de 14 consultas con alguna.
- **Capas de reintento:** peor caso 12 llamadas a proveedor por petición (Router 1+2 primario, 1+2
  fallback, ×2 reintento por esquema). Las del Router y los SDK convierten fallos en latencia sin
  rastro en el contrato.
- **Referencia vigente del arnés:** `gaps` de esa pasada (recall 0,84, precisión 0,86, mediana
  18,3 s, 0,0030 USD/turno).
- **UAT histórica:** ~100 casos con modelo en 28 ficheros; **0** conservan la respuesta.

---

## 4. Aprendizajes agnósticos (candidatos a Guide)

1. **Dos nodos en serie, no uno.** ¿Workflow o agente? lo decide quién es dueño del control flow
   (orden desconocido de antemano → agente, que puede ser **uno solo** con tools en bucle). ¿Uno o
   varios? lo decide la saturación de contexto o de tools. Componente ≠ agente: ejes de cambio
   distintos piden módulos, no llamadas al LLM. *(Teoría; aplicado como no-decisión.)*
2. **Supervisor híbrido por defecto:** reglas para precondiciones, modelo solo con ambigüedad real;
   presupuesto de enrutado y trail desde la primera línea. Sin ambigüedad real, sin supervisor.
   *(Not exercised here.)*
3. **Escalera de comunicación:** estado compartido → handoff (impuesto de enrutado medido **frente al
   supervisor híbrido**) → mensajes (frontera de proceso). Por frontera. *(Not exercised here.)*
4. **Reducers:** política explícita por campo; en paralelo, combinar + deduplicar por clave +
   **ordenar estable**, porque el orden fija la posición en el prompt. *(Teoría; heredado por la
   puerta del grafo.)*
5. **Human-in-the-loop:** señales deterministas antes que confianza autodeclarada; error ≠ revisión;
   precisión **y recall** del gate; checkpointer solo con aprobador distinto o pausa tras trabajo
   caro (si no, turno); nodo de interrupción sin efectos laterales. *(Checkpointer: not exercised
   here.)*
6. **"El sistema sabe que no sabe" en RAG = tipar la no-respuesta.** Cuando el tipo (premisa
   contradicha, remisión, hueco real) solo se conoce tras leer el contexto, va como **categoría en el
   esquema de salida**, no como clasificación previa ni como reglas acumuladas en el prompt. **El
   modelo declara; el código exige cita literal verificada o degrada al tipo neutro.** *(Medido.)*
7. **Adoptar por error asimétrico.** Si fallar devuelve el comportamiento anterior, pesa la
   **precisión de lo servido**, no el recall. *(Medido.)*
8. **Degradación técnica en el contrato, filtrada en la presentación.** Todo evento técnico va a un
   campo propio con vocabulario cerrado; la UI muestra solo lo que cambia la confianza. La política
   configurada (truncados, ventanas) no es degradación para el usuario: un aviso casi universal se
   ignora. *(Medido: 13 de 14.)*
9. **Inventariar las capas de reintento.** Cada capa convierte un fallo en latencia sin rastro; el
   peor caso multiplica (aquí, 12 llamadas por petición). *(Reportado; decisión en M6.)*
10. **Señal de incertidumbre según el tipo de salida:** juicio con error caro → competición con
    **evidencia distinta** (convergencia ≠ acierto; el sintetizador no promedia y la confianza sale de
    la divergencia calculada); factual/anclada en fuente → verificación determinista, luego
    self-consistency; error barato → ninguna. *(Competición: not exercised here.)*
11. **Seguridad de acciones** (condición: hay una tool con efectos; señal: se añade la primera):
    privilegio en la **credencial**, guardia determinista atada a la ejecución, irreversibles a la
    puerta humana, auditoría con denegaciones. **Leer tampoco es inofensivo** en un corpus
    multi-cliente. *(Acciones: not exercised here. Credencial: medido, ver §5.)*
12. **El mínimo privilegio que importa es la credencial.** "Solo lee" por convención del código no
    es una garantía. *(Medido: rol superusuario único.)*
13. **Guardar el par propuesta–decisión desde el día uno.** No se recupera: de ~100 casos históricos,
    0 conservan la respuesta, y regenerarlos no sirve porque el veredicto se emitió sobre otra. El
    juicio se razona en prosa; el registro es su forma consultable. *(Medido.)*
14. **Preregistro: la regla de acierto casa por evidencia y por aspecto.** *(Lección de q12-a3.)*

---

## 5. Aprendizajes de dominio (ECSS)

- **Premisa contradicha y delegación son tipos naturales en un corpus normativo.** Opcionalidad (PUS:
  todos los servicios opcionales; implementar uno obliga a su capacidad mínima) y remisiones entre
  estándares (Q-ST-80C → M-ST-80, Q-ST-10-09, Q-ST-10-04; E-ST-70-41C → E-ST-70-01) y a guías externas
  (ISVV, SAVOIR, Q-ST-30-02).
- **El destino de una remisión suele estar indexado** (M-ST-80C, E-ST-70-01C). Un `delegated` a un
  estándar del corpus es una **señal de hueco de retrieval**, no una respuesta terminal. Distinguirlo
  es determinista con el parser de `xref`.
- **Falso positivo característico:** frases que describen dónde la misión documenta algo («Space-to-
  Ground ICD») no son delegaciones normativas. Candidato a caso negativo del golden set.
- **La truncación es casi universal con `bounded` + `clause` en este corpus.**

---

## 6. Abierto, con anclaje

- **Puerta del grafo de referencias (antes de M6).** Requisitos acumulados: filtro de relevancia
  antes de seguir un puntero; prioridad explícita entre punteros; acumulador por `clause_id` con
  **orden estable** y regla de `origin` cuando llega por dos caminos; presupuesto de saltos y trail;
  `origin` en el contrato; `corpus_version` si hay persistencia; juez LLM solo con ambigüedad real,
  evaluado offline. **Nuevo:** `delegated` a estándar indexado como entrada (marca `in_corpus` y, si
  aplica, seguirlo).
- **Credencial (M6):** rol de solo lectura para el camino de petición (≈ medio día) y que el dueño no
  sea superusuario. Postgres publicado en `0.0.0.0:5432` con contraseña por defecto.
- **Capas de reintento (M6):** decidir con el inventario de §3.
- **Deriva del control:** citas fuera de su afirmación 25 → 43 con la misma configuración. Vigilar en
  la próxima pasada.
- **q14-a1 inestable (2 de 5):** sin acción; volver con el golden set 1.2.
- **Golden set 1.2:** primeros candidatos ya marcados en los registros de UAT (q14 t1, q14-a1, q12) y
  un negativo (Space-to-Ground ICD).
- **UI/UX:** la lista larga de cláusulas del desplegable no sirve al usuario (juicio de Salva,
  "parcial"). Sesión de UI/UX con el producto terminado. El registro `S14-2-panel-look` sigue
  `pending` y debe pasar a `partial`.
- **Deuda de tests:** 3 tests fallan sin `.env` (anterior a S14). La suite fija `GAP_KINDS=off` por
  fixture: el camino del default se cubre solo con tests explícitos.
- **Rechazos con huecos tipados** (TODO en `verification.py`): sin caso observado.
- **Backfill histórico de UAT:** no se hace. Lo que valía ya está en diagnósticos y golden set.

---

## 7. Sesión en vivo

- **Protocolos agente-a-agente** (A2A; "ACP" colisiona de nombre con el protocolo editor–agente): es
  el escalón de mensajes entre organizaciones.
- **Tool ≠ skill:** capacidad de actuar frente a procedimiento que el agente sigue.
- **Agentes en contenedores** desde Python: la frontera de runtime (M6); los principios de privilegio
  y auditoría no cambian.
- **Límite Claude + MCP frente a LangGraph:** sin frontera universal; la escalera de S13 da el
  criterio (forma del flujo, persistencia entre pasos).
- **Reproducir una ejecución fallida:** el checkpoint reproduce el estado, no la salida del modelo;
  hace falta guardar las salidas en la traza.

---

## 8. Proceso

- **Primera UAT con juicio de producto delegado al chat.** Funcionó porque el juicio necesitaba
  conocimiento de dominio (qué es una remisión normativa y qué no). El juicio de render sigue siendo
  del usuario.
- **CC ejecutó la UAT por HTTP y con `AppTest`** sin coste o con coste mínimo (0,022 USD). Lo que
  `AppTest` no ve es el aspecto: separarlo como registro aparte.
- **La comprobación sin generación antes de pagar** confirmó los tres casos objetivo y anticipó el
  matiz de q12-a3 antes de medir.
- **Un brief por funcionalidad** (taxonomía / adopción / registro) dejó cada medición limpia.
