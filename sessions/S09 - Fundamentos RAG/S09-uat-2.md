# S09 · Brief 2 — Guion de UAT (Streamlit)

> Para el usuario. CC ya corrió la sesión encadenada, el cambio de tema y los dos casos de
> respuesta parcial (`S09-decisions-2.md` §1–§2). Esta UAT es el juicio de producto: **¿se siente
> como una conversación, y la voz inspira confianza?** Gasta llamadas reales (céntimos).

Ejecutar desde `C:\Users\salca\proyectos\RAG-ECSS`, en PowerShell.

```powershell
docker compose up -d
docker compose ps --format "{{.Name}} {{.Status}}"
```

Abrir <http://localhost:8501> y pulsar **🔄 New conversation** antes de empezar. Logs, en otra
consola:

```powershell
docker logs rag-ecss-api --since 15m | Select-String "retrieval_completed|citations_validated"
```

En cada `retrieval_completed`, mirar `history_questions`.

---

## Casos 1–3: una sola conversación

1. > What is the difference between software criticality category B and C?

   Línea base: cita `ECSS-Q-ST-80C §D.1`. `history_questions=0`.

2. > And what does ECSS-Q-ST-80C say about it?

   **Esperado: responde** con la tabla D-1 (en S09-1 rechazaba). `history_questions=1`.

3. > And what about category C then?

   **Esperado: responde** sobre categoría C. `history_questions=2`.
   Prueba también «And for category C?»: **CC lo vio fallar con un error** por tener menos de 20
   caracteres, y Streamlit lo pinta como «Could not reach the backend». Juzga cuánto molesta
   (`S09-decisions-2.md` §3.1).

## Caso 4: cambio de tema en la misma conversación

> What are the requirements for software configuration management?

Esperado: fuentes de gestión de configuración, ninguna de criticidad. `history_questions=2`: las
dos preguntas de criticidad entran en la query, y lo que se mira es si **contaminan** lo recuperado
(abre «Retrieved but not cited»). Si te apetece, prueba un salto más brusco (p. ej. a telemetría).

## Caso 5: identificador explícito (conversación nueva)

> What does ECSS-E-ST-40C require for dynamic memory management in flight software?

**Ya visto por CC:** responde con citas `[3]` `[4]` y ahora sí las pinta en «Sources», pero **sigue
empezando por `Not covered by the indexed ECSS standards:`** y puede afirmar que E-ST-40C «no está
indexado», **lo cual es falso** (`S09-decisions-2.md` §2.2). Juzga cómo lo leería un ingeniero.

## Caso 6: fuera de material total

> What are the ECSS requirements for the thermal vacuum testing of a satellite battery?

Esperado según el brief: frase marcador nueva, sin citas. **CC lo vio responder en parte** (vacío
térmico general, sin baterías) bajo la frase de rechazo, a veces con «you would need to consult…».
Mirar: frase nueva y no la antigua; ninguna mención a «provided material»; si cita, que pinte sus
fuentes.

## Caso 7: aviso sin citas

No hay forma fiable de forzarlo. Si alguna respuesta sale **sin `[n]` y sin frase de rechazo**,
tiene que aparecer el aviso «This answer does not cite any source…». Un `Out of scope:` (prueba
«What's a good recipe for a chocolate cake?») **no** lo muestra.

---

## Qué devolver a Claude Chat

- ¿El seguimiento conversacional se siente resuelto (casos 2–3)?
   > Sí, la conversación sigue natural.
- ¿Hay contaminación visible en el cambio de tema (caso 4)?
   > No, ha cambiado de el tema sin referenciar lo anterior.
- La voz en los casos 5–6: ¿la frase de rechazo delante de una respuesta con citas confunde? ¿Y la
  afirmación falsa de que un estándar no está indexado?
  > La respuesta 5 es extremadamente confusa, primero dice que el estándar no está indexado, pero luego le responde con lo que el modelo ya sabe (probablemente de su entrenamiento), necesitamos trabajar estos casos.

  > La respuesta 6 dice que que no está cubierto, pero luego cita texto sobre tests de vacio térmico. También se ve rara.
- ¿Cuánto molesta el 422 en seguimientos cortos?
   > No he visto ningún 422.
