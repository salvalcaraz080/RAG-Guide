# S11 — Brief 1: arnés de generación con métricas deterministas

> Sobre `S11-resumen.md` §8 (decididas) y `pre-S11-decisions.md`. Lee `.claude/CLAUDE.md` y
> `ARCHITECTURE.md` antes de empezar. Rama `s11-generacion-calidad`, encima del commit de pre-S11.
> **Este brief mide; no cambia lo que el usuario recibe.** La política ante citas no verificadas, el
> contrato tipado y el renombrado de la UI son `S11-brief-2`. Todo veredicto nuevo entra en **modo
> observación** (se registra, la respuesta no cambia).

---

## 0. Objetivo y alcance

**Objetivo.** Poder responder con números, por estrato, a "¿esta versión del sistema genera mejor o
peor que la anterior?". El arnés de S10 mide lo que **llega** al modelo; este mide lo que el modelo
**hace** con ello. Primer entregable real: el **suelo de ruido** — cuánto varían las métricas entre
ejecuciones idénticas. Sin él, ninguna comparación A/B posterior (política, `bounded`, prompt) se
puede interpretar.

**Dentro de alcance:** veredicto `misattributed` en la validación literal (G1); detector de frases
factuales sin marcador (G2); arnés de generación sobre el golden set v2 (G3–G6); una ejecución de
referencia con los defaults vigentes (G7).

**Fuera de alcance:** política ante veredictos, contrato tipado, structured output, juez LLM,
ampliar el golden set (consultas condicionales, de abstención, nivel "esencial"), ejecutar
`bounded`, renombrado de la UI, `corpus/` (parser, chunker, export). Si algo lo exige:
`# TODO: pending decision in Claude Chat`.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| G1 | `validate_quotes` gana el veredicto **`misattributed`**: si la cita no está en el grupo `n`, se busca en los demás grupos inyectados de la misma petición | Separa **fabricación** (no está en ningún grupo → `not_found`) de **atribución falsa** (está en otro grupo → `misattributed`, con los `n` donde se encontró). Es la parte de D-1 que se caza con un `if`. Precedencia: `verified` > `misattributed` > `not_found`; `invalid_ref` sin cambios |
| G2 | Detector **determinista** de frases factuales sin marcador `[n]`, función pura, usado en el log de producción (observación) y en el arnés | Una afirmación sin marcador no es verificable por ninguna capa. Es una **heurística**: se documentan sus reglas y se reporta como tal, no como verdad |
| G3 | El arnés ejecuta el **camino real de `/query`** (`llm_service.answer_query`: guardrails, retrieval, prompt, modelo, `_finish`) | Medir el producto, no una réplica. Un camino paralelo para evaluar acabaría divergiendo, igual que divergió el stream (pre-S11) |
| G4 | Cuatro métricas deterministas (§2), sin juez LLM ni RAGAS | Decidido en S11-resumen §8 |
| G5 | **3 ejecuciones** por consulta y configuración; se reporta cada ejecución, la media y el rango (mín–máx) | La generación no es determinista; el rango entre ejecuciones idénticas es el suelo de ruido |
| G6 | Configuración = entrada de una lista, como en el arnés de retrieval. Se ejecuta **solo la de defaults** | `bounded` (u otra) será una entrada más cuando se decida medirla; hoy no se ejecuta |
| G7 | Cada ejecución deja un **artefacto inmutable con fingerprints** | Las métricas solo son comparables a igualdad de corpus, prompt, modelo y configuración |

---

## 2. Métricas (definición exacta)

Identidad de cláusula = `(canonical_doc_id, section_number)`, la del golden set. Las citas del
contrato llevan `standard` = `doc_id_cite` (con revisión): **mapear a la identidad canónica** por la
tabla `documents`, no por manipulación de cadenas. Si alguna cita no mapea, es un error del arnés,
no una cita irrelevante.

| Métrica | Definición | Notas |
|---|---|---|
| **Recall de cita condicionado** | \|citadas ∩ relevantes\| / \|inyectadas ∩ relevantes\| | Denominador 0 → `n/a` (no 0), con recuento. Aísla la omisión del generador dado su contexto; el recall del retrieval ya lo mide el arnés de S10. **No tiene techo en 1** (relevante ≠ imprescindible): se lee comparativo y por estrato |
| **Precisión de cita** | \|citadas ∩ relevantes\| / \|citadas\| | Sin citas → `n/a` |
| **Veredictos literales** | Recuento por `verified` / `misattributed` / `not_found` / `invalid_ref`, más marcadores sin cita literal | Tasa = sobre pares cita–marcador |
| **Frases sin marcador** | Frases factuales sin `[n]` / frases factuales | Heurística G2 |

"Inyectadas" = las cláusulas de los grupos del contexto de esa petición (incluye los átomos ➕ de
expansión: la cláusula es la misma). Como el fan-out es cerrado, citadas ⊆ inyectadas.

**Pares sin juicio.** Si alguna cláusula inyectada no tiene juicio, la medición de esa consulta se
marca **inválida** y los pares se exportan para el modo B, ordenados por cláusula, igual que el
arnés de retrieval (reutiliza su exportación; no la dupliques). Con los defaults vigentes el arnés
de S10 ya juzgó lo inyectado, así que lo esperable es cero pares; si aparecen, es una señal a
registrar (p. ej. no determinismo del retrieval).

---

## 3. Detector de frases sin marcador (G2)

Reglas mínimas; CC puede afinarlas si las documenta:
- Segmentación por frase; cada ítem de lista cuenta como una frase.
- **Con marcador** = contiene al menos un `[n]` (cualquiera de las formas que ya reconoce
  `_citations`).
- **No factuales** (se excluyen del denominador): la frase marcador de rechazo, encabezados, frases
  que terminan en `:` e introducen una lista, y conectores sin contenido. Criterio conservador: ante
  la duda, **cuenta como factual**.
- Contador aparte, también observacional: frases que **afirman ausencia** ("does not require", "no
  requirement", "is not specified/defined"…), con o sin marcador. Es el anclaje "afirmaciones de
  ausencia" de S09; aquí solo se cuentan.

En `S11-decisions.md`: las reglas finales y **ejemplos reales** de frases contadas como sin marcador,
excluidas y de ausencia, para que Claude Chat juzgue si la heurística mide lo que dice.

---

## 4. Trabajo

1. **G1** en `citations.validate_quotes` (función pura) y en el log `citations_validated`. Tests de
   los tres casos y de la precedencia.
2. **G2** como función pura junto a `citations`; cableada en el log (observación) y expuesta al arnés.
3. **Arnés de generación**, junto al de retrieval y con su mismo patrón (módulo bajo
   `corpus/groundtruth/`, p. ej. `generation/`; el congelado de `corpus/` es el pipeline de
   parser/chunker/export, no las herramientas de groundtruth). Reutiliza la carga del golden set, el
   mapeo de identidades y la exportación de pares; no los dupliques.
4. **Artefacto por ejecución** (G7), con marca temporal y nunca sobrescrito: por consulta y ejecución,
   respuesta, citas, cláusulas inyectadas, veredictos, frases sin marcador, `finish_reason`, tokens,
   `latency_ms`, proveedor y si hubo fallback. Cabecera: `input_fingerprint` del corpus, hash del
   prompt de sistema, modelo, configuración, versión del protocolo del golden set, valor de
   `LLM_EVAL_MODE`, fecha. Más un informe legible por estrato.
5. **Fallback.** Una ejecución servida por el proveedor de fallback no es comparable: se marca y se
   reporta aparte; el agregado principal la excluye.
6. **G7: ejecución de referencia.** 14 consultas × 3 ejecuciones = 42 llamadas reales (≈0,30 USD al
   coste del TTFT de S10): **autorizadas**. Defaults vigentes (`vector` + `filter` +
   `tokenized_length` + `orphans`), temperatura y modelo del producto tal cual — se mide el producto,
   no se tunea para medir.
7. Docs: `ARCHITECTURE.md` y `.claude/CLAUDE.md` (estado actual), `CHANGELOG.md` desde
   `S11-decisions.md`.

---

## 5. Tests de contrato (sin red, sin BD, sin modelo real)

- `misattributed`: cita en otro grupo → `misattributed` con los grupos; cita en el grupo citado y en
  otro → `verified`; en ninguno → `not_found`; marcador fuera de rango → `invalid_ref`. Misma
  normalización en los dos lados que ya usa `validate_quotes`.
- Detector G2: frase con y sin marcador, ítems de lista, frase de rechazo excluida, introducción de
  lista excluida, frase de ausencia contada.
- Métricas: recall y precisión con fixtures pequeños, incluidos denominador 0 → `n/a`, mapeo
  `doc_id_cite` → canónico, y cláusula inyectada sin juicio → consulta inválida y par exportado.
- Agregación: 3 ejecuciones → media y rango; una ejecución con fallback queda fuera del agregado.
- El arnés usa `answer_query` (servicio falso inyectado en test), no una réplica del pipeline.
- Recuento final de la suite frente a 1169.

---

## 6. UAT (usuario)

Propón un script para que Salva verifique **a mano** dos consultas del informe (una
`names_standard`, una `cross_standard`): leer la respuesta, contar citas relevantes e inyectadas
contra el golden set y comprobar que el recall y la precisión del informe coinciden; abrir un
veredicto `misattributed` (si hay alguno) y confirmar en el texto de las cláusulas que la cita está
en otro grupo. Es la comprobación de que el instrumento mide lo que dice.

---

## 7. Cierre

`S11-decisions.md` (en `RAG-Guide/sessions/S11 - Generacion y Calidad/`), etiquetado
agnóstico/dominio, con:
- Tabla por estrato: media y rango (mín–máx) de las cuatro métricas → **suelo de ruido**.
- Distribución de veredictos literales y **todos los `misattributed`** con su contexto (material
  directo para la política de `S11-brief-2` y para D-1).
- Reglas finales de G2 con ejemplos (§3) y recuento de frases de ausencia.
- Pares sin juicio (esperado: ninguno), ejecuciones con fallback, coste real, latencia.
- Recuento de tests.

`S11-uat.md` con el script de UAT. `CHANGELOG.md`, `ARCHITECTURE.md`, `.claude/CLAUDE.md`
actualizados.
