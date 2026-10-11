# S09 · Decisiones — Brief 3 (frase de rechazo verificable y seguimientos cortos)

> Decisiones tomadas al implementar `S09-brief-3.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> Las F1–F4 del brief no se repiten.
>
> **Verificación:** `uv run pytest` **1010 verde** (1002 + 8 netos). Contra la pila real: 422 real
> y backend caído pasados por el clasificador del frontend (sin coste), smoke de cuatro preguntas
> con el modelo (≈0,02 USD) y cinco repeticiones del seguimiento corto (≈0,03 USD, §2.1). Pendiente
> del usuario: ver en el navegador cómo pinta Streamlit los mensajes de error (`S09-uat-3.md` §4).

---

## 1. Smoke end-to-end

| # | Pregunta | Resultado |
|---|---|---|
| 5 | E-ST-40C, memoria dinámica | Frase nueva. **Ya no dice que el estándar «no está indexado»**: dice «The search returned no sources from ECSS-E-ST-40C», que es cierto. Pero añade «the indexed materials… do not contain specific requirements from ECSS-E-ST-40C» y «You would need to consult that standard directly» |
| 6 | Batería en vacío térmico | Frase nueva y cierre correcto («I could not find battery-specific… in the indexed ECSS standards»), pero en medio: **«The indexed ECSS standards… do not contain specific requirements dedicated to satellite battery…»** |
| B/C | Abre la sesión | Responde; cita `Q-ST-80C §D.1` |
| «And for C?» | Seguimiento de 10 caracteres | **Responde, sin 422** (F3). Pero con un **error de soporte** (§2) |

Comprobado sin coste: una pregunta de 1 carácter da 422, y el clasificador del frontend la muestra
como **«The backend rejected the request (HTTP 422): Message too short»**; con el backend
inalcanzable, «Could not reach the backend…» (F4). El usuario confirmó en el navegador que
Streamlit pinta el mensaje de petición rechazada.

### 1.1 Lectura

- **F1 funciona en la frase marcador**: siempre la nueva y siempre cierta por construcción.
- **F2 se cumple solo a medias [DOMINIO].** La afirmación falsa concreta de S09-2 (un estándar «no
  indexado») desaparece, pero el modelo sigue afirmando ausencias **en el cuerpo de la respuesta**
  con otra forma («the indexed ECSS standards do not contain…»). La regla 9 se respeta en la letra
  de «not indexed» y no en su intención. Más iteraciones de prompt están fuera de alcance (S11).
- **E6 sigue sin cumplirse**: los casos 5 y 6 son respuestas parciales con citas que empiezan por la
  frase de rechazo. Diferido a S11, como fija el brief.
- **La regla 7 se incumple** en el caso 5 («You would need to consult that standard directly»).

---

## 2. Error de soporte con literalidad perfecta [AGNÓSTICO] — hallazgo principal

En «And for C?» el modelo lista, como tercer caso de la categoría C:

> «Additionally, category C software can be "Software included in compensating provisions for
> category II functions" [5]»

En la tabla D-1 de `ECSS-Q-ST-80C` (comprobado en la base) esa fila es de la **categoría B**. La
tercera fila real de la C es «…compensating provisions for **category III** functions», que la
respuesta omite. En la pregunta B/C del mismo turno, la C aparece con solo dos casos.

- La cita es **literal** (el validador la marca `verified`) y el `[5]` **pertenece** al contexto.
  Pasa los dos niveles automáticos y es **falsa**: atribuye a una categoría la fila de otra.
- La causa probable es estructural: las filas de la tabla se aplanan como «Software criticality
  category: X. Definition: …», y el modelo separa la cita de la etiqueta de categoría que la
  acompaña. Es una pista para el ítem abierto de tablas, sin verificar.

Es el primer caso medido de lo que la UAT de S09-1 no encontró en su muestra: **pertenencia y
literalidad no implican soporte.** Insumo directo para S11 (política ante cita no soportada), y
argumento para que la validación no se quede en el substring.

### 2.1 Repetición: 6 ejecuciones del mismo caso

Cinco ejecuciones más, cada una en una sesión nueva (≈0,03 USD), con las atribuciones comprobadas
contra la tabla D-1 en la base:

| Resultado de «And for C?» | Ejecuciones |
|---|---|
| Correcta y completa (las 3 filas de la C) | 2 de 6 |
| Correcta pero incompleta (omite la fila de cat. III) | 2 de 6 |
| **Atribución errónea** | **2 de 6** |

La fila de la B atribuida a la C no se repitió. **El otro error aparece en una ejecución que falla
de tres formas:**

1. **[AGNÓSTICO] El modelo copia el formato del historial en vez de usar `[n]`.** Escribe
   `[ECSS-Q-ST-80C Rev.2 §D.1]`, que es como `rewrite_markers` guarda el turno anterior en la sesión.
   Sin marcadores válidos, `citations=[]` (`valid_markers=0` en el log) y la validación literal no
   puede comprobar nada. El aviso «sin fuentes» del frontend salta, que es lo correcto, pero la
   respuesta queda sin trazabilidad. Es un **efecto secundario de D11** (reescribir los `[n]` del
   historial a citas naturales): resolvió que un `[1]` antiguo apuntara a otra cláusula, y abrió un
   ejemplo de formato que el modelo imita.
2. Atribuye a la C la fila «cat. III con provisiones compensatorias», que en la tabla es de la **D**.
3. Atribuye a `Q-ST-80C §D.1` la frase «the software product is the sole means to implement the
   function», que en la base solo está en `Q-ST-40C §6.5.6.3` y en `Q-HB-80-03A §5.2.3.2` (también
   recuperados): **cita de una cláusula, atribuida a otra.**

**Lectura.** Con una tabla de filas casi idénticas («Software criticality category: X.
Definition: …»), **un tercio de las respuestas atribuye mal alguna fila**, y la mitad de las
correctas está incompleta. La muestra es pequeña (6), pero suficiente para descartar que fuera un
caso aislado. Tres ítems abiertos para Claude Chat: la política de soporte (S11), el aplanado de
tablas (la etiqueta de categoría se separa de su definición) y el formato del historial (D11).

---

## 3. Decisiones de implementación

- **[DOMINIO] La regla nueva es la 9, al final de las reglas de voz**, y no una 5b: el brief pedía no
  tocar las reglas 6–8, y renumerar las habría tocado. Dice al modelo que solo ve lo que devolvió
  la búsqueda y le prohíbe afirmar que algo «does not exist, is not indexed, or is not in the indexed
  ECSS standards».
- **[DOMINIO] Las reglas 4, 5 y el apartado «Out of material» pasan a voz de búsqueda** («what you
  found», «you could not find it»). Hablaban de lo que los estándares indexados «contienen» o «no
  cubren», que es la misma afirmación sobre el corpus que F1 retira de la frase marcador.
- **[AGNÓSTICO] Riesgo del `startswith`, revisado como pide el brief.** Ninguna regla induce a abrir
  una respuesta normal con la frase exacta (con «this» y los dos puntos): la regla 8 pide nombrar el
  hueco **al final**. El riesgo real es el ya conocido de E6: si el modelo abre una respuesta parcial
  con la frase, se clasifica como rechazo. Desde S09-2 eso ya no borra sus citas, así que el daño se
  limita al valor de `refusal` (y a que el aviso «sin fuentes» no salte). No se rediseña la
  detección.
- **[AGNÓSTICO] El clasificador de errores vive en `frontend/errors.py`, no en `sse_client.py`**:
  `sse_client` importa `sseclient`, que solo está en el grupo `frontend` y no en el entorno de la
  suite; `errors.py` solo necesita `requests`. Cubre también los fallos de `create_session` y del
  prompt del sidebar, que pasan por el mismo `except`. Un 4xx con `detail` de texto lo muestra tal
  cual (el backend ya lo redacta para el usuario, p. ej. el rechazo de moderación).
- **[AGNÓSTICO] Un 422 de validación no enseña el texto de Pydantic** («String should have at least 2
  characters» es para un desarrollador). Se traduce por tipo de error a una frase corta
  (`string_too_short` → «Message too short») y los tipos no mapeados dan «Invalid request».
  Decisión del usuario tras ver el primer mensaje.
- **[AGNÓSTICO] Los tests del frontend estrenan `tests/frontend/`** y cargan el módulo **por ruta**:
  el frontend importa sus módulos por nombre desnudo desde su carpeta, y meter `frontend/` en
  `sys.path` dejaría un módulo `errors` global en toda la suite.
- **[AGNÓSTICO] Un test de S08 dependía del mínimo de 20** para afirmar que `/query` «no se tocó» y
  que la generación «sigue sin retrieval». Se reescribe a lo que sigue siendo cierto: cada endpoint
  valida su propio body (`test_each_endpoint_validates_its_own_body`).

---

## 4. Lo que queda abierto (va a `ARCHITECTURE.md` §7)

- **Respuestas parciales con frase de rechazo delante (E6)** → S11, contrato de salida tipado.
- **Recuperar el estándar nombrado en la query** → S10, ponderación híbrida o filtro por ID.
- **Frases sin cita y afirmaciones de ausencia en el cuerpo** (relleno de preentrenamiento, F2 a
  medias, regla 7) → S11.
- **Soporte de la cita**, con 2 de 6 respuestas mal atribuidas en el caso de la tabla D-1 (§2) → S11.
- **El modelo imita el formato de cita del historial** (`[standard §clause]` en vez de `[n]`), lo
  que deja la respuesta sin citas: efecto secundario de D11 (§2.1). Sin sesión asignada.
