# S09 — Brief 3: frase de rechazo verificable y seguimientos cortos

> Continuación de `S09-brief-2.md`, sobre `S09-decisions-2.md` §2.2 y §3.1. Dos cambios mínimos.
> Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar.

---

## 0. Alcance

**Dentro:** sustituir la frase marcador de "no encontrado"; permitir preguntas cortas; que el
frontend muestre los 4xx como lo que son.

**Fuera (no implementar, diferido con anclaje):**

- Más iteraciones de prompt sobre respuestas parciales (E6) → S11 (contrato de salida tipado).
- Recuperar el estándar nombrado en la query → S10 (ponderación híbrida / filtro por ID).
- Detectar o medir frases sin cita (relleno con conocimiento de preentrenamiento) → S11.

Si algo parece exigir esto: `# TODO: pendiente de decisión en Claude Chat`.

---

## 1. Decisiones

| # | Decisión | Porqué |
|---|---|---|
| F1 | Frase marcador `Not covered by the indexed ECSS standards:` → **`I could not find this in the indexed ECSS standards:`** | La anterior afirma una ausencia en el corpus entero, pero el modelo solo ve los k chunks recuperados: cada fallo de retrieval se convertía en una afirmación falsa (medido: "E-ST-40C is not included in the indexed material"). La nueva habla de la búsqueda, que es verificable. Cierta por construcción, en voz de producto |
| F2 | Las reglas de `system.md` no permiten afirmar que un estándar, cláusula o tema **no está** en el corpus | Misma causa que F1, en el cuerpo de la respuesta y no solo en la frase marcador |
| F3 | `QueryRequest.question`: `min_length` de 20 → **2** | El mínimo se puso en M1 para no gastar una llamada en preguntas triviales; en una conversación, los seguimientos cortos ("And for C?") son uso normal. 2 solo descarta lo vacío o de un carácter |
| F4 | El frontend distingue **4xx** de fallo de conexión | Hoy un 422 se pinta como "Could not reach the backend…", un mensaje falso |

---

## 2. Trabajo

### 2.1 Frase marcador (F1, F2)

- Cambia la frase en `system.md`, en `_refusal_kind` y en los tests. El valor interno
  `out_of_material` no cambia (continuidad de logs). `Out of scope:` no cambia.
- La comparación sigue siendo `startswith` exacto y case-sensitive; la frase nueva empieza por
  `I`, así que comprueba que ninguna otra regla del prompt induce respuestas normales que empiecen
  igual (si la hubiera, regístralo en decisiones; no rediseñes la detección).
- Regla nueva en `system.md`, en voz dirigida al modelo: no afirmar que un estándar, cláusula o
  tema no existe o no está en los estándares indexados; decir solo que no se encontró.
- No toques las reglas 6–8 de respuesta parcial más allá de lo que exija la frase nueva.

### 2.2 Preguntas cortas (F3)

- `min_length=2` en `QueryRequest`. `max_length`, si existe, no cambia.
- Ajusta los tests que fijaban 20.

### 2.3 Errores en el frontend (F4)

- En `/query/stream` y `/query`: un 4xx muestra un mensaje que dice que la petición fue rechazada,
  con el `detail` del backend si viene; un 5xx dice que el servicio falló; solo el fallo de
  conexión o timeout dice "Could not reach the backend".
- Factorízalo en `sse_client.py` (o junto a él), sin importar Streamlit, para poder testearlo.

---

## 3. Tests de contrato

- `_refusal_kind` reconoce la frase nueva y no la anterior.
- `QueryRequest` acepta una pregunta de 2 caracteres y rechaza una de 1.
- Clasificación de errores del cliente: 422 → mensaje de petición rechazada; 503 → fallo del
  servicio; `ConnectionError` → backend inalcanzable.

## 4. UAT propuesta

CC redacta el guion. Mínimo:

1. Caso 5 de la UAT anterior (E-ST-40C, memoria dinámica): ninguna afirmación de que el estándar
   "no está indexado"; si rechaza, con la frase nueva.
2. Caso 6 (vacío térmico de batería): frase nueva; ninguna afirmación de ausencia en el corpus.
3. En una sesión abierta, "And for C?" tras la pregunta B/C: responde, sin error.
4. Forzar un 4xx (p. ej. pregunta de 1 carácter): mensaje de petición rechazada, no de conexión.

## 5. Cierre

- `S09-decisions-3.md`, etiquetado agnóstico/dominio.
- `CHANGELOG.md`, `CLAUDE.md` y `ARCHITECTURE.md` según sus reglas. En `ARCHITECTURE.md` §7, deja
  anotados los tres diferidos de §0 con su sesión (S10, S11, S11).
- `uv run pytest` en verde y mensaje de commit propuesto.
