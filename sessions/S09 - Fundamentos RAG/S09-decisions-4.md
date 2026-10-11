# S09 · Decisiones — Brief 4 (historial sin formato de cita)

> Decisiones tomadas al implementar `S09-brief-4.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> G1 y G2 no se repiten.
>
> **Verificación:** `uv run pytest` **1017 verde** (1011 + 6 netos). Contra la pila real: el caso
> de `S09-decisions-3.md` §2.1 en 6 sesiones nuevas (≈0,03 USD), con el contenedor confirmado sobre
> el código nuevo (`strip_markers` presente, `rewrite_markers` ausente). Sin UAT del usuario: nada
> de lo encontrado requiere juicio de producto.

---

## 1. Verificación: B/C → «And for C?», 6 sesiones nuevas

**Criterio de aceptación (formato de cita): cumplido.**

| Ejecución | Marcadores `[n]` | Citas naturales `[ECSS-…]` | `citations` | Atribución (dato, no criterio) |
|---|---|---|---|---|
| 1 | 4 | 0 | 1 | Correcta y completa |
| 2 | 4 | 0 | 2 | **Errónea**: mete en la C la fila «compensating provisions for category II», que es de la B, y omite la de cat. III |
| 3 | 5 | 0 | 1 | Correcta y completa |
| 4 | 3 | 0 | 1 | Correcta y completa |
| 5 | 5 | 0 | 1 | Correcta y completa |
| 6 | 2 | 0 | 1 | Correcta, incompleta (omite la fila de cat. III) |

- **Formato:** **6/6 con marcadores `[n]` válidos y 0/6 con citas naturales.** Con D11 había sido
  1 de 6 con formato natural y sin citas.
- **Atribución, comparada con S09-3:**

  | | S09-3 (historial con citas naturales) | S09-4 (historial sin marcas) |
  |---|---|---|
  | Correcta y completa | 2/6 | 4/6 |
  | Correcta pero incompleta | 2/6 | 1/6 |
  | Atribución errónea | 2/6 | 1/6 |

  La mejora es compatible con que el historial sin marcas distraiga menos, pero **con 6 ejecuciones
  por lado no es una diferencia medida**: la atribución sigue siendo un problema abierto (soporte,
  aplanado de tablas → S10/S11), y la fila de la B atribuida a la C reaparece.
- **Nuevo, menor:** en la ejecución 6 dos citas literales de la lista no llevan marcador propio (solo
  el `[5]` del encabezado y del cierre). Ya lo cuenta `markers_without_quote`; no es efecto de este
  cambio.

---

## 2. Decisiones de implementación

- **[AGNÓSTICO] `strip_markers` sustituye a `rewrite_markers`, que se elimina** junto con su
  marcador `[?]` para refs fuera de rango: borrar no necesita distinguir válido de inválido, así que
  la función ya no recibe los grupos.
- **[AGNÓSTICO] Una racha de marcas seguidas se trata como una sola** (`[1] [3]`, `[1][3]`), con el
  espacio horizontal que la rodea. El separador se decide una vez según lo que sigue: nada si va
  delante de puntuación, al final de la línea o al principio; un espacio si separa dos palabras.
  Tratar cada marca por separado dejaba «word .» en «word [1] [3].».
- **[AGNÓSTICO] Solo espacio horizontal**: el borrado nunca consume saltos de línea, así que listas
  y párrafos del turno guardado conservan su estructura.
- **[DOMINIO] Cita natural = corchete que empieza por `ECSS-`.** Cubre `[ECSS-… §…]`, listas con
  `;` y variantes sin `§`. Otros corchetes del texto (`[a]`, `[5-1]`) se conservan: son contenido,
  no formato de cita.
- **[AGNÓSTICO] El test del turno 2 captura una copia del historial en el momento de la llamada**:
  el mock guarda la lista por referencia y `append_turn` la muta después.

---

## 3. Lo que queda abierto

Nada nuevo. El ítem «el modelo imita el formato de cita del historial» sale de `ARCHITECTURE.md` §7.
Siguen diferidos, tal como los dejó el brief 3: respuestas parciales con frase de rechazo (S11),
recuperar el estándar nombrado (S10), frases sin cita y ausencias en el cuerpo (S11), y el soporte
de la cita, con 1/6 de atribución errónea en esta corrida (S11).
