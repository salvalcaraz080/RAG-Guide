# S11 — Brief 2: salida estructurada, contrato tipado y política de verificación

> Sobre `S11-decisions.md` (referencia `generation_20261018-164057`) y las decisiones A–D tomadas en
> Claude Chat tras leerlo. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar. Rama
> `s11-generacion-calidad`.
> **Este brief cambia lo que el usuario recibe**, y se valida con el arnés del brief 1 contra la
> referencia. Regla del brief: el modelo **declara**, el código **valida y compone**. Todo lo que se
> pueda decidir en código no se le pide al modelo.

---

## 0. Objetivo y alcance

**Objetivo.** Sustituir la prosa con `[n]` por una salida estructurada que el código valida antes de
servir, con un estado de respuesta tipado y una política explícita por veredicto. Se eliminan tres
heurísticas que existían por la prosa: parseo de marcadores por regex, segmentación en frases
(`statements.py`) y clasificación del rechazo por prefijo de texto.

**Dentro de alcance:** esquema de salida (H1), validación y estado (H2–H3), política por veredicto
(H4–H6), relajación de `normalize` (H7), composición de la respuesta en código (H8), prompt reescrito
(H9), contrato HTTP y UI (H10–H11), re-medición con el arnés (H12).

**Fuera de alcance:** juez LLM, reintento por calidad de cita, ampliar el golden set, `bounded`,
cambiar de modelo generador, `corpus/`. Si algo lo exige: `# TODO: pending decision in Claude Chat`.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| H1 | **Salida estructurada** validada con Pydantic (esquema en §2), pedida al proveedor como `response_format` a través del wrapper | Decisión A. Una frase sin fuente pasa a ser violación de esquema, no una heurística; el estado se declara, no se infiere de un prefijo |
| H2 | El modelo **declara** el estado; el código lo **normaliza** con reglas deterministas (§3) | 4/42 respuestas de la referencia empiezan con frase de rechazo y citan: el modelo confunde parcial con rechazo. La regla lo corrige sin preguntar al modelo |
| H3 | Salida que no valida el esquema → **un reintento**; si vuelve a fallar → **502** tipado, sin turno en sesión | Mismo criterio que respuesta vacía y fallo de proveedor (pre-S11): el proveedor devolvió algo inservible |
| H4 | `misattributed` → **se reasigna** la cita a todos los grupos de `found_in`, sin nueva llamada | Decisión C (UAT de brief 1): el usuario ve la cláusula de la que salió la cita |
| H5 | `not_found` cosmético → **se sirve** con la cita marcada como **no literal** | Decisión B. Ningún `not_found` de la referencia es fabricación; no se rechaza lo que es una elisión |
| H6 | Nuevo veredicto **`modal_edit`** (la cita solo coincide con la fuente ignorando los modales) → **se sirve con `needs_review`** visible; sin reintento | Decisión B. La inserción de «shall» en q04 es inflación de autoridad; 1/280, no justifica +6 s de reintento |
| H7 | `normalize` unifica `'`/`"` e ignora el espacio junto a `/` y a guiones, **en los dos lados** | 7/16 `not_found` de la referencia son falsos negativos. Relajar es monótono: solo acepta más, nunca rechaza una cita antes aceptada, así que el importador del golden set no se rompe |
| H8 | **La respuesta se compone en código** a partir del objeto validado: marcadores, frase de "no encontrado", texto de rechazo | El código garantiza la voz (reglas 5, 6, 8, 9 del prompt actual) en vez de pedírsela al modelo |
| H9 | Prompt reescrito para la salida estructurada, incluidas las reglas de voz y de cita literal | Decisión D. Se mide contra la referencia |
| H10 | `QueryResponse` gana `status`, `statements`, `not_found`, `verification`, `needs_review`; `refusal` se sustituye por `status` | Contrato tipado: el cliente decide por campos, nunca por texto |
| H11 | UI: el bloque "retrieved but not cited" se reorganiza en dos ejes | Anclaje de S10 |
| H12 | Re-medición con el arnés y comparación preregistrada (§7) | Medir el cambio, no suponerlo |

---

## 2. Esquema de salida (lo que el modelo devuelve)

Punto de partida; CC puede ajustar nombres si lo documenta, no la semántica.

```python
class Quote(BaseModel):
    text: str                 # exact span copied from one <item>; must also appear inside Statement.text
    refs: list[int]           # source ids the quote was copied from (min 1)

class Statement(BaseModel):
    text: str                 # one claim, no markers; may contain the quote in double quotes
    refs: list[int]           # source ids this claim is based on (min 1)
    quote: Quote | None = None

class Section(BaseModel):
    heading: str | None = None
    as_list: bool = False     # render statements as bullets
    statements: list[Statement]

class GenerationOutput(BaseModel):
    status: Literal["answered", "partial", "out_of_domain", "out_of_material"]
    sections: list[Section]
    not_found: list[str] = []     # aspects of the question not found (short noun phrases)
    refusal_reason: str | None = None   # only for out_of_* statuses, one sentence
```

- **Toda afirmación lleva `refs` (mínimo 1).** No hay frases de transición sin fuente; lo que no se
  encontró va a `not_found`, no a una frase libre.
- **`not_found` son frases nominales**, no oraciones: la frase que las presenta la compone el código
  (H8). Así la única forma de afirmar ausencia es "no encontrado en los estándares indexados", y la
  ausencia normativa ("ECSS no exige X") queda fuera del canal previsto.
- Compatibilidad del esquema con los dos proveedores (Anthropic y fallback OpenAI) vía LiteLLM:
  verificar con la versión fijada; si una construcción (p. ej. `Literal`, opcionales anidados) no se
  soporta en uno de los dos, simplificar el esquema, no añadir un camino por proveedor.

---

## 3. Validación y estado (en código, tras parsear)

Orden fijo, todo determinista:
1. **Refs válidos.** Un ref fuera del rango de grupos se quita de su `Statement`/`Quote` y se registra
   `invalid_ref`. Si una afirmación se queda sin refs → se sirve marcada **sin fuente** y
   `needs_review = true` (no se borra texto en silencio).
2. **`quote.refs ⊆ statement.refs`** tras el paso 4; si el modelo no los incluyó, se añaden.
3. **`quote.text` dentro de `statement.text`** (normalizado). Si no está, la cita no se puede
   presentar como literal de esa frase: se trata como afirmación sin cita literal y se registra.
4. **Verificación literal** de cada `quote` contra sus grupos, con la precedencia del brief 1 más
   `modal_edit`: `verified` > `misattributed` > `modal_edit` > `not_found`.
   - `modal_edit`: no se encuentra tal cual, pero sí tras eliminar los modales (`shall`, `should`,
     `may`, `must`, `will`, `can`) **de los dos lados**, en el grupo citado o en otro (si en otro, se
     reasigna además, como H4).
5. **Estado normalizado** (el declarado por el modelo se registra aparte):

| Declarado | Condición | Estado servido |
|---|---|---|
| `out_of_material` | hay afirmaciones con refs | `partial` |
| `answered` | `not_found` no vacío | `partial` |
| `partial` | `not_found` vacío | `answered` |
| `out_of_domain` / `out_of_material` | sin afirmaciones | se mantiene |
| `answered` / `partial` | sin afirmaciones | `out_of_material` |

6. **`needs_review`** = algún `modal_edit`, algún `invalid_ref` o alguna afirmación sin fuente.
   `verification` = recuentos por veredicto, reasignaciones (`from` → `to`) y citas no literales.

---

## 4. Política por veredicto

| Veredicto | Qué recibe el usuario |
|---|---|
| `verified` | La cita, con su marcador |
| `misattributed` | La cita con el marcador **reasignado** a `found_in` (todos si son varios). Los `refs` de la afirmación no se tocan salvo para añadir `found_in`: el modelo pudo apoyar la afirmación en la cláusula original aunque la cita no saliera de ella |
| `not_found` (cosmético) | La cita, marcada como **no literal** |
| `modal_edit` | La cita, marcada como no literal **y** `needs_review` |
| `invalid_ref` | Ref retirado; si la afirmación queda sin fuente, marcada y `needs_review` |

Ningún veredicto provoca un reintento ni una nueva llamada al modelo (el reintento de H3 es solo por
esquema inválido).

---

## 5. Composición en código (H8)

- Secciones → encabezado (si hay) + afirmaciones como párrafos o viñetas; cada afirmación termina con
  sus marcadores `[n]` (`[1][3]`), en orden de primera aparición.
- `not_found` no vacío → una frase de cierre fija del producto (p. ej. "Not found in the indexed ECSS
  standards: X; Y."). La formulación es constante y la define el código.
- `out_of_domain` / `out_of_material` → texto de rechazo fijo del producto + `refusal_reason` si
  existe. Las frases marcador (`Out of scope:` / `I could not find…`) dejan de ser un mecanismo de
  clasificación; si se conservan como texto visible es decisión de presentación.
- **Historial de sesión**: se guarda la composición **sin marcadores**, generada directamente (no
  quitando marcadores con regex después).
- `citations` = unión de refs tras validación, en orden de primera aparición. El fan-out sigue
  cerrado.

---

## 6. Qué se elimina o se reduce

- Parseo de `[n]` por regex en `_citations`, `strip_markers`, `_refusal_kind` por prefijo: fuera si
  nada más los usa.
- `statements.py`: la segmentación y el detector de frases sin marcador dejan de tener objeto (una
  afirmación sin refs es inválida por esquema). **Se conserva lo que sigue midiendo algo**, aplicado
  a `Statement.text`: el contador de **ausencia** (detecta ausencias normativas que el modelo meta en
  una afirmación) y un detector nuevo de **reglas de voz 6–7** ("consult the full standard",
  "material available to me", "provided material"…), los dos solo como métrica y log.
- Las reglas del prompt que pasan a garantizarse en código (formulación de "no encontrado", frases
  marcador, formato de marcadores) salen del prompt.

---

## 7. Re-medición (H12) y comparación preregistrada

1. **Re-puntuar la referencia sin llamadas**: aplicar `normalize` nuevo (H7) y el veredicto
   `modal_edit` al artefacto `generation_20261018-164057`. Esa es la línea base de la comparación
   (aísla el efecto del cambio de salida del efecto del validador).
2. **Ejecutar el sistema nuevo**: 14 consultas × 3 ejecuciones, defaults vigentes (≈46 llamadas,
   ≈0,30–0,40 USD por el sobrecoste de tokens del JSON): **autorizadas**. Si la primera pasada revela
   un fallo del instrumento, una segunda pasada completa también está autorizada (como en el brief 1).
3. **Arnés adaptado** a la salida estructurada: mismas métricas de recall condicionado, precisión y
   veredictos (con `modal_edit` y citas no literales); "frases sin marcador" se sustituye por:
   afirmaciones por respuesta, afirmaciones sin fuente tras validación, `not_found` por respuesta,
   ausencias y violaciones de voz detectadas, distribución de estados declarado → servido, tasa de
   esquema inválido y reintentos, tokens de salida y latencia.
4. **Criterio** (global y pareado por consulta, no por estrato — el suelo de ruido por estrato con 2
   consultas no decide nada):
   - Recall condicionado y precisión de cita: **no empeoran más de 0,05** (≈ el ancho del ruido
     global) frente a la línea base re-puntuada.
   - Tasa `verified`: no empeora más de 0,05.
   - **Cero** respuestas `out_of_*` con citas en el estado servido; cero violaciones de esquema
     servidas.
   - Latencia: se reporta; sin umbral (se juzga en Claude Chat).
   - Si algo empeora más allá del umbral: **no iterar el prompt a ciegas**; reportar con los casos y
     decide Claude Chat.

---

## 8. Contrato HTTP y UI (H10–H11)

- `QueryResponse`: `answer` (composición con marcadores, para presentar), `status` (servido),
  `statements` (estructura validada, con veredicto por cita y reasignaciones), `not_found`,
  `verification`, `needs_review`; se mantienen `citations`, `retrieved_context`, `finish_reason`,
  `truncated`, `latency_ms`, etc. `refusal` desaparece: lo cubre `status`.
- Truncado con JSON roto (`finish_reason == "length"`): no se puede parsear → 502 tipado como
  truncado (no como esquema inválido) y sin reintento ciego. Comprobar que el `max_tokens` vigente
  deja margen al sobrecoste del JSON (reportar tokens de salida de la referencia frente a los nuevos).
- UI de evidencia en dos ejes: **citado / no citado** × **recuperado / añadido por expansión (➕)**.
  Etiquetas propuestas: "Cited", "In context, not cited"; el ➕ se mantiene como marca del átomo.
  Cita no literal → distintivo discreto junto a la cita; `needs_review` → aviso visible en la
  respuesta; reasignaciones → solo en el panel de diagnóstico.
- `status = partial` → la UI no lo presenta como rechazo (hoy el aviso de "does not cite any source"
  depende de `refusal`).

---

## 9. Tests de contrato (sin red, sin BD, sin modelo real)

- Esquema: salida válida; ref fuera de rango; afirmación sin refs → inválida → reintento → 502;
  `quote.text` no contenido en `statement.text`.
- Normalización de estado: las cinco filas de la tabla del §3.
- Veredictos: `verified`, `misattributed` (uno y varios `found_in`), `modal_edit` (modal añadido,
  quitado y cambiado; en el grupo citado y en otro), `not_found`, `invalid_ref`; precedencia.
- `normalize`: `'`/`"`, espacio junto a `/` y a guiones; test de monotonía (toda cita verificada con
  la normalización anterior sigue verificada) sobre las citas del golden set.
- Composición: marcadores en orden, frase de cierre de `not_found`, rechazo, historial sin marcadores.
- Truncado con JSON roto → 502 tipado como truncado.
- Contrato HTTP: campos nuevos presentes; `refusal` ausente; frontend: `partial` no se pinta como
  rechazo.
- Recuento final frente a 1217.

---

## 10. UAT (usuario)

Script para la UI: una consulta `answered` (criticidad B vs C en Q-ST-80C), una que la referencia
dejaba como "rechazo que cita" (q01) y que ahora debe salir `partial` con la frase de cierre, una
fuera de dominio, un seguimiento multi-turn, y — si la ejecución del arnés produce alguno — un caso
con cita reasignada o `needs_review` para ver cómo se presenta. Revisar los nuevos rótulos de
evidencia.

---

## 11. Cierre

`S11-decisions-2.md` con: esquema final y lo que hubo que simplificar por compatibilidad de
proveedores; tabla del §7 (línea base re-puntuada vs nuevo, global y por consulta, con el ruido);
efecto aislado de H7 (cuántos `not_found` pasan a `verified`); distribución declarado → servido;
todas las reasignaciones, `modal_edit` y afirmaciones sin fuente con su contexto; tasa de esquema
inválido; ausencias y violaciones de voz con ejemplos; tokens y latencia; recuento de tests; qué se
eliminó (§6). `S11-uat-2.md` con el script. `CHANGELOG.md`, `ARCHITECTURE.md`, `.claude/CLAUDE.md`.
