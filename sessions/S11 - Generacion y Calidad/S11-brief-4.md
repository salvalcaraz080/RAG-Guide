# S11 — Brief 4: cierre — cita sin repetición, "no encontrado" legible, rechazo limpio, reproducibilidad

> Sobre `S11-decisions-3.md` y `S11-uat-3.md`. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de
> empezar. Rama `s11-generacion-calidad`. Último brief de implementación de S11.
> Referencia de la comparación: `generation_20261018-212343` (Haiku 5.5).

---

## 0. Objetivo y alcance

**Objetivo.** Corregir los tres defectos de presentación que encontró la UAT conjunta, cerrar las
preguntas abiertas de `S11-decisions-3.md` §8 y dejar una medición final de S11.

**Fuera de alcance:** ampliación del golden set (diferida: nivel "esencial", consultas condicionales
y de abstención, revisión de juicios de q12/q14), esfuerzo de razonamiento (se queda en `medium`),
juez LLM, `bounded`, `corpus/`. Si algo lo exige: `# TODO: pending decision in Claude Chat`.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| K1 | **Regla de prompt: cita o paráfrasis, no las dos.** Si una afirmación solo transmite lo que dice un requisito, su `text` **es** el tramo literal entre comillas (y `quote` es ese mismo tramo). La paráfrasis queda para cuando la afirmación sintetiza, compara o combina fuentes; entonces `quote` (si hay) es el tramo que la sostiene | UAT caso 2b: «The supplier shall ensure… as stated: "The supplier shall ensure…" [8]». Criterio del usuario: si la respuesta exige síntesis, que el modelo la elabore; si solo va a reformular la cita, que ponga la cita |
| K2 | **Red de seguridad en la composición:** si una cita verificada iría aparte y es prácticamente la misma que el texto de su afirmación (criterio determinista y documentado, p. ej. el texto normalizado está contenido en la cita o comparten casi todos los tokens), se compone **solo la cita** con su marcador | Que una regla de prompt no se cumpla siempre no debe volver a mostrar la repetición. Umbral conservador: ante la duda, se muestran las dos |
| K3 | **"No encontrado" como bloque aparte:** «I could not find in the indexed ECSS standards:» y una viñeta por elemento, separado del cuerpo | UAT caso 4: la frase en línea con elementos separados por `;` se leía como contenido de los estándares |
| K4 | **Regla de prompt para `not_found`:** cada elemento nombra **un aspecto de la pregunta** que no se encontró, en términos de la pregunta del usuario (p. ej. «requirements that apply only to category C»), nunca como un hecho sobre el contenido del estándar | Un elemento como «Category C-specific rows in the tailoring matrix» se lee como ausencia normativa, justo lo que el canal `not_found` debía evitar |
| K5 | **Rechazo limpio:** con estado servido `out_of_domain` u `out_of_material`, la vista principal muestra solo el mensaje de rechazo; nada de «Cited» ni «In context, not cited». La evidencia sigue en el panel de diagnóstico (modo desarrollo) y en el contrato | UAT caso 3 |
| K6 | **Mapa de modelos de LiteLLM local** (`LITELLM_LOCAL_MODEL_COST_MAP=True` o equivalente en 1.106.0), en la imagen y en local | Hoy se descarga de GitHub `main` en cada arranque y decide precio **y vía de salida estructurada**: un cambio remoto alteraría el comportamiento y rompería la comparabilidad de las mediciones sin desplegar nada |
| K7 | **Longitud máxima de la pregunta en la validación de entrada** → 422 antes de cualquier efecto (como los guardrails). Valor coherente con `MAX_INPUT_TOKENS` y documentado. Cierra el `TODO` de `_fit_budget` (`context_budget_exceeded`) | Una pregunta que sola excede el presupuesto no es un caso de uso; se rechaza en el borde, no a mitad del pipeline |
| K8 | **Métrica de vigilancia en el arnés: cláusulas citadas / inyectadas**, por consulta y global | El recall condicionado se dispara si el modelo lo cita todo (q12: 9–10 de 10). La razón tiene que estar a la vista para no premiar esa estrategia |
| K9 | **Caché de prefijo: se queda activa en producción** (solo documentar) | +2,8 % con tráfico espaciado (~0,00006 USD/llamada) frente a −10 % en ráfagas; las sesiones multi-turn son ráfagas, y una sola configuración para arnés y producción es más simple |
| K10 | **La caída de precisión del brief 3 se acepta como excepción explícita** (solo documentar en `ARCHITECTURE.md`/`CHANGELOG.md` donde corresponda) | Criterio preregistrado simétrico frente a la asimetría de producto (peor omitir que citar de más): +0,20 de recall por −0,07 de precisión. Decidido en Claude Chat |

---

## 2. Medición final

1. **Pasada del arnés**: 14 × 3 con los defaults y el prompt nuevo (≈46 llamadas, ≈0,10 USD,
   **autorizadas**; segunda pasada autorizada si falla el instrumento).
2. **Criterio preregistrado** contra `generation_20261018-212343`, global y pareado:
   - recall condicionado: no empeora más de 0,05;
   - precisión de cita: no empeora más de 0,05;
   - tasa `verified`: no empeora más de 0,05; cero rechazos servidos con citas; cero esquemas
     inválidos servidos;
   - **citas fuera de su afirmación: bajan** (referencia: 46 de 249).
3. **Se reporta además:** citadas / inyectadas (K8), cuántas veces actuó la red de K2, elementos
   `not_found` por respuesta con ejemplos (para juzgar K4), estados declarado → servido, tokens,
   latencia, coste.
4. Si algo empeora más allá del umbral: no iterar el prompt; reportar con casos.

---

## 3. Tests de contrato (sin red, sin BD, sin modelo real)

- K2: cita casi igual al texto → solo la cita; cita distinta → afirmación y cita; cita dentro del
  texto → sin cambios.
- K3: composición del bloque «I could not find…» con viñetas; sin `not_found` → sin bloque.
- K5: estados de rechazo → la presentación no incluye evidencia; `partial` y `answered` sí.
- K6: el arranque usa el mapa local (test o comprobación de configuración, lo que sea verificable
  sin red).
- K7: pregunta por encima del máximo → 422, sin llamada al modelo, sin retrieval, sin turno.
- K8: la razón citadas / inyectadas en el artefacto y el informe.
- Snapshot del prompt actualizado (K1, K4).
- Recuento final frente a 1263.

---

## 4. UAT (usuario)

`S11-uat-4.md`, corta: repetir q02 (caso 2b de la UAT anterior: ya no se repite cita y paráfrasis),
el seguimiento multi-turn de categoría C (caso 4: bloque «I could not find…» legible), fuera de
dominio (caso 3: solo el mensaje), y una pregunta enorme pegada en el chat (422 con mensaje claro en
la UI).

---

## 5. Cierre

`S11-decisions-4.md` con: tabla del criterio (global y por consulta), K8 por consulta, actuaciones de
K2, ejemplos de `not_found` antes/después, longitud máxima elegida y su porqué, confirmación del mapa
local, recuento de tests, coste. `S11-uat-4.md`. `CHANGELOG.md`, `ARCHITECTURE.md`,
`.claude/CLAUDE.md`.
