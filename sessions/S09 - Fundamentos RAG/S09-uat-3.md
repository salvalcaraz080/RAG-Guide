# S09 · Brief 3 — UAT

> **Estado:** los casos 1–3 los verificó CC contra la pila real y **no necesitan tu input** (los
> resultados están abajo). Solo queda **el caso 4: mirar en el navegador** dos mensajes que CC no
> puede ver, porque no maneja Streamlit. Detalle y porqués en `S09-decisions-3.md` §1–§2.

Ejecutar desde `C:\Users\salca\proyectos\RAG-ECSS`, en PowerShell.

```powershell
docker compose up -d
docker compose ps --format "{{.Name}} {{.Status}}"
```

---

## 1. Identificador explícito — verificado por CC

> What does ECSS-E-ST-40C require for dynamic memory management in flight software?

**Resultado:** frase nueva `I could not find this in the indexed ECSS standards:`, y ya **no** dice
que E-ST-40C «no está indexado». Pero el cuerpo afirma «the indexed materials… do not contain
specific requirements from ECSS-E-ST-40C» y remite al usuario a «consult that standard directly».
**F2 a medias; regla 7 incumplida.** Diferido a S11.

## 2. Batería en vacío térmico — verificado por CC

> What are the ECSS requirements for the thermal vacuum testing of a satellite battery?

**Resultado:** frase nueva y cierre correcto («I could not find battery-specific… in the indexed ECSS
standards»), pero en medio «The indexed ECSS standards… do not contain specific requirements
dedicated to satellite battery…». **F2 a medias.** Diferido a S11.

## 3. Seguimiento corto — verificado por CC, 6 ejecuciones

> What is the difference between software criticality category B and C?
> And for C?

Cada ejecución en una sesión nueva. Atribuciones comprobadas contra la tabla D-1 de
`ECSS-Q-ST-80C` en la base.

| Ejecución | Resultado de «And for C?» |
|---|---|
| Smoke original | Responde, pero mete en la C la fila «compensating provisions for **category II**», que es de la **B**, y omite la de cat. III |
| 1 | **Correcta y completa** (3 filas de la C) |
| 2 | Correcta, **incompleta** (omite la fila de cat. III) |
| 3 | **Falla de tres formas** (abajo) |
| 4 | **Correcta y completa** |
| 5 | Correcta, **incompleta** |

**F3 cumplido:** 6/6 responden sin error. **Calidad de la respuesta:** 2/6 completas, 2/6
incompletas y **2/6 con atribución errónea**.

La ejecución 3 es la más grave:

1. **No usa marcadores `[n]`**: escribe las citas como `[ECSS-Q-ST-80C Rev.2 §D.1]`, el formato con
   el que el historial guarda el turno anterior. El modelo lo **copió** del historial. Resultado:
   `citations=[]` y la validación no puede comprobar nada (`valid_markers=0` en el log).
2. Mete en la C la fila de cat. III con provisiones compensatorias, que en la tabla es de la **D**.
3. Atribuye a `Q-ST-80C §D.1` la frase «the software product is the sole means to implement the
   function», que en la base solo está en `Q-ST-40C §6.5.6.3` y en `Q-HB-80-03A §5.2.3.2`.

## 4. Mensajes en la interfaz — **pendiente de ti**

CC comprobó que la lógica produce el texto correcto: el 422 real del backend pasado por el
clasificador del frontend da «The backend rejected the request (HTTP 422): Message too short», y un
backend inalcanzable da «Could not reach the backend…». Lo que no puede ver es **cómo lo pinta
Streamlit**.

> El primer mensaje repetía el texto de Pydantic («String should have at least 2 characters»). Tras
> la UAT se resumió en «Message too short», a petición del usuario.

**4a. Petición rechazada.** En <http://localhost:8501>, escribe:

> x

Esperado: un recuadro de error con **«The backend rejected the request (HTTP 422)…»**, y **no**
«Could not reach the backend».

Resultado: pinta el mensaje esperado.

**4b. Backend inalcanzable** (opcional):

```powershell
docker compose stop api
```

Haz cualquier pregunta. Esperado: **«Could not reach the backend at http://api:8000…»**. Después:

```powershell
docker compose start api
```

---

## Qué devolver

- ¿4a muestra el mensaje de petición rechazada?
- (Opcional) ¿4b muestra el de backend inalcanzable?

Todo lo demás está decidido o medido; lo que queda abierto va a Claude Chat, no a UAT.
