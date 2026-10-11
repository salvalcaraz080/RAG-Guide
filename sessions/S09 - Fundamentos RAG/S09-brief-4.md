# S09 — Brief 4: historial sin formato de cita

> Continuación de `S09-brief-3.md`, sobre `S09-decisions-3.md` §2.1. Un solo cambio.
> Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar.

---

## 0. Alcance

**Dentro:** el turno del asistente que se guarda en la sesión deja de llevar marcas de cita.

**Fuera (no implementar):** soporte de la cita, aplanado de tablas, respuestas parciales (E6),
afirmaciones de ausencia en el cuerpo. Siguen diferidos a S10/S11 tal como los dejó el brief 3.

---

## 1. Decisión

| # | Decisión | Porqué |
|---|---|---|
| G1 | El texto del asistente que se guarda en el historial **no lleva ninguna marca de cita**: se eliminan los `[n]` (y `[1, 3]`, `[1][3]`) en vez de reescribirlos a `[<standard> §<clause>]`. Sustituye a D11 | Todo lo que va en el historial es un ejemplo de formato para el modelo. Con D11 el modelo imitó `[ECSS-Q-ST-80C Rev.2 §D.1]` en vez de usar `[n]` (1 de 6 en `S09-decisions-3.md` §2.1): sin marcadores válidos, la respuesta pierde citas y validación. Quitar las marcas resuelve los dos problemas a la vez: no hay `[1]` antiguo que apunte a otra cláusula y no hay formato alternativo que imitar |
| G2 | También se eliminan del turno guardado las citas en formato natural entre corchetes (`[ECSS-… §…]`) si el modelo las hubiera escrito | Si una respuesta ya salió con ese formato, guardarla tal cual reintroduce el ejemplo en el turno siguiente |

**Qué se pierde, dicho en voz alta:** el turno siguiente ya no sabe qué cláusula respaldaba cada
frase del turno anterior. Se acepta: el retrieval de cada turno trae sus propias fuentes (y desde
E1 la query lleva las preguntas previas), así que el modelo vuelve a tener la evidencia con
marcadores válidos de esta petición.

---

## 2. Trabajo

- Sustituye la reescritura de marcadores del historial por un borrado. Al borrar, normaliza los
  espacios que queden (`"…" [3].` → `"…".`, sin doble espacio ni espacio antes de puntuación).
  Las citas literales entre comillas **se conservan**: son texto, no formato de cita.
- El texto **devuelto al cliente** no cambia: conserva sus `[n]`.
- Ninguna otra parte del flujo cambia (`citations`, validación, `retrieved_context`, logs).
- Si la función reescritora queda sin uso, elimínala; no dejes código muerto.

## 3. Tests de contrato

- El turno guardado no contiene `[n]`, `[1, 3]`, `[1][3]` ni `[ECSS-… §…]`; las comillas se conservan.
- Sin dobles espacios ni espacio antes de puntuación tras el borrado.
- La respuesta al cliente conserva los `[n]`.
- El mensaje que se envía al modelo en el turno 2 no contiene ningún corchete de cita del turno 1.
- Ajusta o retira los tests que fijaban D11.

## 4. Verificación (CC, con coste mínimo)

Repite el caso de `S09-decisions-3.md` §2.1 (B/C → «And for C?») en 6 sesiones nuevas (≈0,03 USD)
y reporta: cuántas respuestas usan `[n]` válidos (esperado 6/6) y, como dato, la tabla de
atribuciones igual que en §2.1. La calidad de atribución **no** es criterio de aceptación de este
brief; solo el formato de cita.

UAT del usuario: no hace falta salvo que CC encuentre algo que requiera juicio de producto.

## 5. Cierre

- `S09-decisions-4.md`, etiquetado agnóstico/dominio, con la tabla de la verificación.
- `CHANGELOG.md`, `CLAUDE.md` y `ARCHITECTURE.md` según sus reglas. En `ARCHITECTURE.md`: la
  descripción del historial pasa a «turno guardado sin marcas de cita» con el porqué en una línea;
  §7 pierde el ítem del formato de cita del historial.
- `uv run pytest` en verde y mensaje de commit propuesto.
