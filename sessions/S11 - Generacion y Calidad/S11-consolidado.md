# S11 — Consolidado de sesión (histórico, no se reescribe)

> **Paso 5 del ciclo.** Combina el resumen provisional (`S11-resumen.md`, paso 2) con lo aprendido al
> implementar el brief previo y los cuatro briefs (`pre-S11-decisions.md`, `S11-decisions*.md`,
> `S11-uat*.md`). La **divergencia teoría↔medida** se conserva a propósito: es el material de Guide.
> La graduación a `RAG-GUIDE.md` (paso 6) se hace **desde** este documento.
>
> **Sesión:** M4 / S11 — Generación y calidad (augmentation, síntesis, citación, alucinaciones,
> ciclo de vida del índice, evaluación). **Cerrada con:** respuesta no streameada; salida
> estructurada validada en código (afirmación → refs → cita literal), estado tipado
> (`answered`/`partial`/`out_of_domain`/`out_of_material`) normalizado por reglas; política por
> veredicto sin llamadas extra; arnés de generación con métricas deterministas sobre el golden set v2;
> generador `claude-haiku-5-5` (fallback `gpt-6-luna`) con salida estructurada nativa; presupuesto de
> entrada; caché de prefijo; mapa de modelos de LiteLLM fijado. `uv run pytest` **1280** verde (1179
> al empezar).

---

## 1. El arco de S11

S10 dejó cuatro anclajes: decidir `bounded`, el contrato tipado (parcial/rechazo), la política ante
una cita literal válida que atribuye mal (D-1) y el rótulo "retrieved but not cited". La revisión
teórica (seis artículos) dio el vocabulario; la implementación cambió el orden y, en dos sitios, la
respuesta.

0. **pre-S11 — quitar el streaming.** La política estricta (rechazar o corregir antes de servir) es
   imposible si el texto ya está en pantalla. Se eliminó `/query/stream` entero. Al auditar los tests
   aparecieron **dos invariantes que solo existían en el camino de stream** (aviso de truncado; fallo
   de proveedor tipado): `/query` los ganó.
1. **Brief 1 — la vara de la generación.** Arnés sobre el camino real (`answer_query`), cuatro
   métricas deterministas, veredicto `misattributed`, detector de frases sin marcador. Primer
   entregable: el **suelo de ruido**. Diagnóstico: el generador omite (recall condicionado 0,65), no
   inventa (precisión 0,91; ningún `not_found` era fabricación).
2. **Brief 2 — salida estructurada y contrato tipado.** El modelo declara, el código valida y
   compone. Desaparecen tres heurísticas de prosa. Pasa el criterio, pero con tres salvedades: ~10 %
   de esquema inválido (vía de herramienta forzada de Haiku 4.5), la cita literal se pierde en
   paráfrasis (76/188), latencia ×1,8.
3. **Brief 3 — modelo, presupuesto, caché, cita visible.** Haiku 5.5 con salida nativa: 0 inválidas,
   recall +0,20, coste por turno ÷5. La precisión cae 0,07 (no pasa el criterio simétrico): se acepta
   como excepción explícita por la asimetría de producto.
4. **Brief 4 — cierre.** Cita o paráfrasis (no las dos), "no encontrado" en términos de la pregunta y
   en bloque aparte, rechazo sin evidencia en la vista, mapa de LiteLLM local, longitud máxima de
   pregunta. Criterio preregistrado: pasa entero.

---

## 2. Decisiones contrastadas con la implementación

### Confirmadas

- **Separar aritmética de juicio / el modelo declara, el código decide** (art. 2): generalizó más
  allá de las cifras. Estado, refs, composición, frase de ausencia y reasignación son deterministas;
  el modelo solo redacta afirmaciones y elige fuentes.
- **Hacer imposible antes que detectar** (art. 3, ya graduado en S09 para `[n]`): con `refs`
  obligatorios por afirmación, la "frase sin fuente" pasó de heurística medida (0,21 de las frases) a
  violación de esquema.
- **Embudo barato → caro** (art. 4): todo lo observado en la sesión lo cazaron las capas
  deterministas (`invalid_ref`, literal, `misattributed`, `modal_edit`). La capa 3 (juez/NLI) no tuvo
  señal que la justificara.
- **`misattributed` sin LLM** (resumen §4): 2/280 pares en la referencia, siempre entre **cláusulas
  hermanas contiguas del mismo documento y tema** (q13, E-HB-40A §5.10.2.2.1 ↔ §5.10.2.2.2). Recall y
  precisión no lo ven (las dos cláusulas son relevantes); por eso el veredicto literal es métrica
  aparte.
- **Inflación de autoridad detectable en la capa 1** (pregunta abierta del resumen): la inserción de
  «shall» en q04 se detecta comparando cita y fuente sin modales (`modal_edit`). No hace falta juez.
- **RAGAS fuera** (resumen §6): las métricas deterministas de citas diagnosticaron todo lo que
  importaba (omisión, atribución, ediciones dentro de comillas, rechazos que citan). El argumento del
  sesgo de *faithfulness* a favor de la expansión sigue siendo **razonamiento, no medida**: no se
  ejecutó RAGAS para comprobarlo.
- **El golden set por cláusula sobrevive a un re-chunk** (art. 5, hueco anotado en el resumen): los
  juicios son `(canonical_doc_id, section_number)`, no ids de chunk. El riesgo era menor de lo
  anotado.
- **Presupuesto de tokens del contexto** (art. 1, una de las etapas que sobrevivían): implementado
  con descarte por rango y registro; 0 recortes en la medición (máximo real 8.909 frente a 30.000).
- **Abstractiva excluida** (art. 1): no se tocó; la augmentation de RAG-ECSS quedó en orden,
  presupuesto y cabecera determinista.

### Refinadas

- **De "política compatible con streaming" a "sin streaming".** El resumen proponía entregar el
  veredicto al final del stream. La implementación mostró que eso dejaba fuera justo la política
  estricta; se eliminó el streaming. La latencia percibida (6,3 s → 12,8 s mediana con razonamiento)
  resultó aceptable en la UAT.
- **Referencia de generación: de "cláusulas obligatorias derivadas del v2" a recall condicionado.**
  El v2 tiene 16–55 relevantes por consulta frente a `top_k=10` y relevancia binaria amplia: no es
  una lista de obligatorias. Se midió **relevantes citadas / relevantes inyectadas** (aísla al
  generador; el retrieval lo mide el arnés de S10). El nivel "esencial" exige anotación nueva.
- **Criterio de `bounded`.** El resumen lo formuló por estrato ("gana en el normativo"). Con 2
  consultas por estrato el ruido llega a 0,47: se reformuló como **pareado por consulta sobre el
  agregado**, con más ejecuciones. `bounded` no se midió: `orphans` sigue de default.
- **Reasignación (`misattributed`).** Se corrige la fuente de la **cita**, no las fuentes de la
  **afirmación** (el modelo pudo apoyarla en la cláusula original). Más simple y conservador que
  "retirar la de origen si nada más la sostiene".
- **`not_found`, de veredicto único a taxonomía.** De 16 en la referencia: 5 falsos negativos del
  validador (comillas, espacios junto a `/` y guiones; se relajó `normalize`, cambio **monótono**),
  1 `modal_edit`, 10 ediciones del modelo dentro de las comillas. Ninguna fabricación. Rechazar por
  `not_found` habría rechazado citas correctas.
- **Cita visible.** Con salida estructurada el modelo rellenaba `quote` con el tramo exacto y
  parafraseaba en `text` (76/188 citas invisibles). Primero se compuso la cita aparte (brief 3); la
  UAT reveló repetición; se resolvió con regla de prompt **y** red determinista (cita sola si la
  afirmación no añade palabras propias). De 38 actuaciones de la red, 35 eran "el texto ya era la cita
  sin comillas".

### Refutadas o corregidas por la medida

- **"La precisión no puede bajar más de 0,05" (criterio simétrico).** Contradecía la asimetría de
  producto (peor omitir que citar de más). +0,20 de recall por −0,07 de precisión se aceptó como
  excepción documentada; el criterio tendría que haberla codificado desde el principio.
- **"El prompt caching suprime varianza y exige `LLM_EVAL_MODE`"** (`ARCHITECTURE.md` desde S03):
  falso. La caché de prefijo ahorra cómputo de entrada y no cambia la distribución de la salida.
  Corregido.
- **"Una sonda corta valida el esquema"**: la sonda inicial del brief 2 pasó con un caso trivial y la
  primera pasada real dio 26 salidas inválidas (`$ref` sin `#` generado por LiteLLM en la ruta de
  herramienta forzada). La sonda tiene que usar un caso que ejercite el esquema entero.
- **"El contexto recuperado en el system"**: el orden histórico ponía las fuentes dentro del system,
  tras las instrucciones; el prefijo no era idéntico entre peticiones y la caché era imposible. Se
  movieron al último mensaje de usuario.
- **`MAX_OUTPUT_TOKENS=2000`**: con un modelo de razonamiento, el pensamiento cuenta en la salida; q06
  se truncaba en los dos modelos. Subido a 16.000 (solo se factura lo generado).

---

## 3. Números medidos

**Generación (14 consultas × 3 ejecuciones; recall condicionado / precisión de cita / `verified`):**

| configuración | recall | precisión | `verified` | inválidas | latencia med. / p95 | coste por turno |
|---|---|---|---|---|---|---|
| Prosa, Haiku 4.5 (brief 1) | 0,65 | 0,91 | 0,94 (280 citas) | — | 5,6 / 7,6 s | ~0,0055 USD |
| Estructurada, Haiku 4.5 (brief 2) | 0,72 | 0,91 | 0,98 (112 citas) | ~10 %, 1 × 502 | 10,3 / 16,3 s | 0,0103 USD |
| Estructurada, Haiku 5.5 (brief 3) | 0,91 | 0,85 | 1,00 | 0 | 13,7 / 19,4 s | 0,0019 USD |
| Final (brief 4) | 0,91 | 0,86 | 1,00 | 0 | 12,8 / 20,0 s | 0,0018 USD |

- **Suelo de ruido** (6 ejecuciones, prosa): global ±0,03 (ancho 0,04–0,06); por estrato hasta 0,47.
- **Rechazos servidos con citas:** 4/42 (prosa) → 0 desde el brief 2.
- **Citas fuera de su afirmación:** 76/188 (Haiku 4.5) → 46/249 (Haiku 5.5) → **26/233** (regla K1).
- **Citadas / inyectadas:** 0,73 global; 0,97–1,00 en q07, q10, q12.
- **`not_found` por respuesta:** 2,40 → 1,55 tras la regla K4.
- **Tokens:** entrada mediana ~6.600 (máx 8.909); salida mediana ~2.900 (máx observado 16.000 en una
  ejecución del brief 3, 7.118 en el final). Estimación ÷2,5 frente a real: 1,00–1,26.
- **Caché:** prefijo estático 2.493 tokens (mínimo 512). −10 % de coste con llamadas seguidas; +2,8 %
  con tráfico espaciado (~0,00006 USD/llamada).
- **Latencia sin streaming, prosa** (pre-S11): mediana 6,31 s, p95 7,40 s (sustituye al TTFT 1,76 s).
- **Coste de API de la sesión:** ≈ 2,2 USD; **1,4 USD solo en el brief 2**, casi todo depuración
  (tres pasadas, ~0,35 USD de salidas inválidas). Una pasada estable del arnés cuesta ~0,08 USD.

---

## 4. Aprendizajes agnósticos (candidatos a Guide)

1. **El modelo declara, el código valida y compone.** Todo lo que puede decidirse en código (estado,
   frase de ausencia, marcadores, reasignación) no se le pide al modelo. Corolario: la voz del
   producto ("no encontrado en…") la escribe el código, no el prompt.
2. **Salida estructurada como contrato, no como formato.** Convierte heurísticas sobre prosa
   (segmentación, marcadores por regex, rechazo por prefijo) en un esquema validado. Condición: que el
   streaming no sea requisito. Señal: estás escribiendo detectores sobre el texto libre del modelo.
3. **La vía de salida estructurada decide la fiabilidad.** Nativa (decodificación restringida): 0 %
   inválidas. Emulada por herramienta forzada: ~10 %. Comprueba qué vía usa tu capa de abstracción
   para cada modelo; puede decidirlo una tabla de capacidades, no tu código.
4. **El streaming no es neutro: decide qué políticas de verificación son posibles.** Si necesitas
   corregir o rechazar antes de servir, no hay streaming (o hay buffering, que lo anula). Y dos caminos
   de entrega divergen: los invariantes acaban viviendo solo en el más usado.
5. **Mide el suelo de ruido antes de cualquier A/B.** Con generación no determinista, el rango entre
   ejecuciones idénticas es la vara de la vara. Con pocas consultas por estrato, un estrato aislado no
   decide nada: compara pareado sobre el agregado.
6. **El criterio preregistrado codifica la asimetría del dominio.** Un umbral simétrico en precisión
   y recall contradice un producto donde omitir es peor que citar de más. Escríbelo antes de medir, y
   si se rompe, acéptalo o recházalo como excepción explícita, nunca moviendo el umbral en silencio.
7. **Vigila la estrategia degenerada.** Un recall condicionado alto se consigue citándolo todo; la
   razón citadas / inyectadas tiene que estar a la vista junto al recall.
8. **Métricas deterministas de citas antes que un juez LLM o RAGAS** cuando hay juicios humanos de
   relevancia por unidad citable. RAGAS: faithfulness contra el contexto entero no ve la atribución
   falsa; answer relevancy castiga la abstención correcta.
9. **Recall condicionado = relevantes citadas / relevantes inyectadas.** Separa el fallo del
   generador del del retrieval. Sin nivel "esencial" no tiene techo en 1: léelo comparativo.
10. **La atribución falsa aparece entre fuentes hermanas** (contiguas, mismo documento y tema): la
    etiqueta se pega a la primera fuente del tema. Detectable buscando la cita en las demás fuentes
    inyectadas; corregible reasignando sin nueva llamada.
11. **Relaja la normalización de la verificación literal solo de forma monótona** (acepta más, nunca
    rechaza lo aceptado) y compruébalo sobre las citas ya verificadas. Así no rompes mediciones ni
    importadores que comparten la función.
12. **Cita o paráfrasis, no las dos.** Con salida estructurada, el modelo tiende a copiar la cita en
    un campo y parafrasear en otro. Regla de prompt + red determinista (si la afirmación no añade
    palabras propias, se sirve la cita sola).
13. **La ausencia se formula como aspecto de la pregunta**, nunca como hecho sobre la fuente
    ("requirements that apply only to category C", no "Category C rows in the matrix"). Canal propio
    (`not_found`) y frase compuesta por el código.
14. **El modelo es una palanca de primer orden.** Cambiar de generación de modelo dio más (+0,20 de
    recall, ÷5 de coste, 0 inválidas) que cualquier técnica de la sesión. Antes de diseñar alrededor
    de las limitaciones de un modelo, comprueba si hay uno más nuevo; mídelo con el mismo arnés.
15. **Modelos de razonamiento: el pensamiento cuenta en los tokens de salida.** Dimensiona
    `max_tokens` con margen y vigila ejecuciones desbocadas (16.000 tokens / 44 s en una).
16. **Fija el mapa de capacidades/precios de la capa de abstracción** (LiteLLM lo descargaba de
    `main` en cada arranque, y decidía precio **y** vía de salida). Un cambio remoto no puede alterar
    el comportamiento ni la comparabilidad de las mediciones.
17. **Prompt caching: tres condiciones antes de activarlo.** Prefijo por encima del mínimo cacheable;
    patrón de tráfico (escribir cuesta más que leer, TTL corto: con tráfico espaciado encarece);
    orden del prompt (estático primero, dinámico al final). No interfiere con la evaluación.
18. **Presupuesto de entrada con estimación conservadora calibrada** contra el conteo real que
    devuelve el proveedor (gratis). Descarta unidades enteras por rango y registra qué tiró. El caso
    "la pregunta sola no cabe" se cierra en la validación de entrada (longitud máxima), no a mitad del
    pipeline.
19. **Una sonda tiene que ejercitar el caso real.** La sonda trivial validó un esquema que falló en
    26 de las primeras respuestas reales. El coste de una sesión lo domina la depuración, no el uso.
20. **La suite de contrato tiene que impedir la red, no confiar en los mocks.** Un test sin simular
    el modelo hizo una llamada real; se detectó por el tiempo. (Propuesta, no implementada.)

---

## 5. Aprendizajes de dominio (ECSS)

- **Tipología de "síntesis" en normativa** (resumen §2): acumulativa, condicional, de autoridad,
  contradicción real. La condicional (criticidad, tailoring) es la de más valor y no está en el golden
  set: no se ha medido si el sistema la aplana.
- **El generador omite más de lo que inventa:** antes del cambio de modelo, un tercio de lo relevante
  inyectado quedaba sin citar; ninguna cita fallida era fabricación.
- **Las ediciones dentro de comillas existen y algunas cambian la fuerza normativa** (insertar
  «shall»): la cita literal no es solo un localizador, es la garantía de que la obligación es la del
  estándar.
- **Reglas de voz que el modelo incumplía en prosa** ("consult the full standard", "material available
  to me") desaparecieron con la salida estructurada y la composición en código (0 violaciones).
- **Precisión de `names_standard_xref` baja y estable** (0,58 con Haiku 4.5; q12 cita 9–10 de 10 con
  Haiku 5.5): puede ser juicio estricto del golden set más que error del modelo. A revisar.
- **Tablas aplanadas ilegibles en la traza** (UAT 4): el verbatim es correcto; falta render.

---

## 6. Abierto, con anclaje

| Tema | Señal medida | Condición para reabrir | Dónde |
|---|---|---|---|
| **Ampliación del golden set (protocolo 1.2)**: nivel "esencial", consultas condicionales y de abstención, revisión de juicios q12/q14 y `xref` | Recall sin techo; precisión de q12/q14; abstención y síntesis condicional sin medir | Antes de la próxima decisión que dependa de omisión o abstención | Diferida (decisión del usuario) |
| **`bounded` como default** | Retrieval normativa 0,59 → 0,95, +36 % tokens; criterio corregido (pareado, agregado, ≥5 ejecuciones) | Arnés de generación ya disponible (~0,10 USD por pasada) | Diferida con señal |
| **Esfuerzo de razonamiento `low`** | Latencia 12,8 s mediana; una ejecución desbocada | Si la latencia molesta en uso real | — |
| **Capa 3 (juez/NLI por cita)** | 0 `misattributed` y 0 `modal_edit` en las pasadas estructuradas; D-1 semántico no observado | Aparición de atribución falsa que la capa literal no ve | — |
| **Ausencia normativa dentro de una afirmación** | Contador = cota inferior; no distingue ausencia de búsqueda y normativa | Si el contador sube o la UAT la encuentra | — |
| **Bloqueo de red en la suite de contrato** | Una llamada real accidental (brief 4) | Siguiente brief que toque tests | Próxima sesión |
| **Render de chunks en la traza** (saltos de línea, tablas con `|`) | UAT 4 | Al tocar el frontend | `ARCHITECTURE.md` §7 |
| Diferidos de S10 sin cambios: grafo/filtro ampliado, re-chunk + re-embed (tokenizador real, tablas, duplicados, marcadores), reranker, HyDE | Ver `S10-consolidado.md` §6 | Los suyos | — |

---

## 7. Sesión en vivo

- **Evaluación como ciclo medir → mejorar → medir**, sin objetivo de 100 %: es lo que hizo la sesión
  (cuatro briefs, cada uno decidido con la medida del anterior).
- **La evaluación avanzada llega tarde** (primero pipeline y datos): coherente con RAG-ECSS, que midió
  retrieval en S10 y generación en S11.
- **Patrón critic vs testing:** en clase, "la generación necesita un LLM para evaluarse". Aquí, la
  mayor parte de la evaluación de la generación fue determinista (citas, refs, estados).
- **Evaluar con un modelo y producir con otro:** corolario aplicado — el juez (o el generador) forma
  parte de la versión de la medida; cambiar el generador obligó a una referencia nueva.
- **Corpus que cambia → versionado de documentos/chunks:** ya resuelto en S07 por fingerprints; el
  golden set por cláusula lo aguanta.
- **Fuentes contradictorias:** la respuesta del material (art. 2) entró como tipología y estado
  `partial`/condicional; la condicional sigue sin medir.

---

## 8. Proceso

- **Un brief previo y cuatro de sesión**, cada uno decidido con la tabla del anterior. El brief 3 no
  estaba planificado: lo abrió el usuario al revisar costes y modelos, y fue el de mayor impacto.
- **Excepciones al orden del flujo, registradas:** la UAT del brief 2 se pospuso y se unió a la del
  brief 3 (con Haiku 4.5 el primer caso daba 502); la caída de precisión del brief 3 se aceptó fuera
  del criterio preregistrado; `bounded` se dejó en `orphans` sin medirlo.
- **Decisiones del usuario que cambiaron el diseño:** eliminar el streaming, salida estructurada,
  reasignar la atribución falsa, cambiar de modelo, cita o paráfrasis, diferir la ampliación del
  golden set.
- **Coste:** ≈ 2,2 USD de API, dominado por la depuración del brief 2. Anotación: ninguna nueva.
- **Lo que hizo CC sin que el brief lo pidiera y fue correcto:** reconstruir los grupos `[n]` desde
  la respuesta del producto (medir lo que se entrega, no lo que el servicio vio); historial idéntico
  entre ejecuciones multi-turn; artefacto autosuficiente (re-puntuable sin llamar al modelo); reutilizar
  la exportación de pares; extender la red K2 a la cita dentro del texto; tomar el proveedor real del
  despliegue para el coste del snapshot fechado; inspeccionar los wheels de LiteLLM antes de
  proponerlo.
