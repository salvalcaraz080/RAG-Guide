# pre-S12 · UAT — historial por turnos y suite sin red

> **Por qué es tuyo:** los casos 1–3 son caso 2 de la frontera (coste real: CC no tenía autorizada
> ninguna llamada al modelo) y solo se juzgan leyendo la respuesta. El caso 4 necesita cortar la red
> de tu máquina. CC ha verificado el contrato (1.296 tests, equivalencia byte a byte del array de
> mensajes y de la query de retrieval) y que la API recarga el código nuevo sin errores.
>
> **Coste:** 7 preguntas a Haiku 5.5, unos 0,015 USD. Cada respuesta tarda 10–20 s.

## 0 · Preparación

```powershell
docker compose ps        # api, db y frontend en marcha
```

La API monta `app/` y recarga sola: ya corre el código de la rama. Abre `http://localhost:8501`.

Para ver cuántas preguntas previas entraron en la query de retrieval de cada turno (la usan los
casos 1 y 2):

```powershell
docker compose logs api --since 10m 2>&1 | Select-String -Pattern "history_questions(?:\x1b\[[0-9;]*m)*=(?:\x1b\[[0-9;]*m)*(\d+)" | ForEach-Object { "history_questions=" + $_.Matches[0].Groups[1].Value }
```

---

## 1 · Seguimiento anafórico (3 turnos)

**🔄 New conversation**, y una detrás de otra:

```
What does ECSS-E-ST-40C require for software safety and dependability requirements?
```
```
And in ECSS-Q-ST-80C?
```
```
Which of those clauses are normative?
```

**Esperado:** el turno 2 responde sobre seguridad y confiabilidad del software **en Q-ST-80C** (no
sobre Q-ST-80C en general); el turno 3 habla de las cláusulas de los turnos anteriores. Como antes
del cambio. En los logs: `history_questions` = **0, 1, 2**.

## 2 · Más allá de la ventana

Baja la ventana a un turno: en `.env`, `MAX_HISTORY_TURNS=1` (a mano en el editor), y:

```powershell
docker compose up -d --force-recreate api
```

Recarga la página, **🔄 New conversation**, y:

```
What does ECSS-E-ST-40C say about software safety requirements?
```
```
What is the Software Development Plan in ECSS-E-ST-40C?
```
```
Going back to my first question: what does ECSS-Q-ST-80C add?
```

**Esperado:**
- Los tres turnos responden, sin error ni aviso de sesión perdida.
- En los logs: `history_questions` = **0, 1, 1**. Con un turno de ventana, al tercero solo le queda
  el segundo, aunque `RETRIEVAL_HISTORY_QUESTIONS` sea 2.
- El turno 3 **no** sabe que la primera pregunta era sobre seguridad: responde sobre el SDP en
  Q-ST-80C, o dice que no lo encuentra. Si respondiera sobre seguridad, el turno 1 seguiría
  influyendo: anótalo.

**Deshaz el cambio:** `MAX_HISTORY_TURNS=10` en `.env` y otra vez
`docker compose up -d --force-recreate api`.

## 3 · Pregunta suelta, sin historial

**🔄 New conversation**:

```
What does ECSS-E-ST-40C require for software safety and dependability requirements?
```

**Esperado:** respuesta con citas («Cited»), sin aviso de revisión inesperado, en el orden de la
referencia: mediana 12,8 s, p95 20 s (la latencia está en la barra lateral de desarrollo).

## 4 · La suite sin red

Corta la red (Wi-Fi apagada o cable fuera) y, en la raíz del repo:

```powershell
uv run --offline pytest -q
```

**Esperado:** `1296 passed, 8 deselected`. `--offline` hace que `uv` no intente sincronizar
dependencias por red; la suite no lo necesita. Vuelve a conectar la red al terminar.

---

**Si algo falla:** en el caso 1 o 3, copia la pregunta, la respuesta y la salida del comando de
logs. En el caso 4, las primeras líneas del primer fallo: si dice `NetworkBlockedError`, el guard ha
encontrado una salida que la suite no conocía.

**Resultado:** Nada falla, todos los tests da el resultado esperado.

**Feedback adicional:** Aunque los tests pasan, las respuestas no me dejan del todo convencido. Especialmente las parciales, a veces da la sensación que el modelo fuerza la respuesta parcial y enumera cosas "que no encontró" que no son relevantes o no se han preguntado explícitamente. Por ejemplo, a la pregunta:
```
What is the Software Development Plan in ECSS-E-ST-40C?
```
El modelo responde:
```
"The software development plan (SDP) is called from the normative provisions summarized in Table O-1." [1] Table O-1 gives the SDP's traceability to ECSS-E-ST-40 and ECSS-Q-ST-80 clauses, as "SDP traceability to ECSS-E-ST-40 and ECSS-Q-ST-80 clauses" [1].

⚠️ I could not find in the indexed ECSS standards:

- definition of the software development plan
- contents of the software development plan
```
Para empezar, la respuesta en sí es muy pobre para todo el contexto que tiene. Se limita a citar en lugar de desarrollar una respuesta. Es una pregunta abierta que requiere de razonamiento y sintesis de varias clausulas, no es una pregunta sobre requerimientos específica, tal vez debamos hacer esa distinción.

En segundo lugar, las viñetas de lo que no pudo encontrar no son cosas específicamente preguntadas y parecen forzadas.