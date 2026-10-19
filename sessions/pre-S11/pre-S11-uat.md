# pre-S11 · UAT — Streamlit sin streaming

> **Por qué es UAT y no un test:** juzgar la UI en el navegador (indicador de espera, aviso de
> sesión perdida, que una respuesta larga no se corte) requiere una persona mirando, y cada
> pregunta hace una llamada real al modelo (céntimos en total).

## 0. Preparación (PowerShell, desde la raíz del repo)

La API ya sirve el código nuevo (bind mount + `--reload`). El frontend necesita reconstruirse:
su imagen aún instala `sseclient-py`, que ha salido del lock.

```powershell
docker compose up -d --build frontend
docker compose ps
```

Abre http://localhost:8501. El sidebar de diagnóstico se ve porque `FRONTEND_ENV` es
`development` por defecto. Empieza con **🔄 New conversation**.

## 1. Pregunta puntual

Pregunta: `What is the difference between software criticality category B and C in ECSS-Q-ST-80C?`

- [x] Mientras se genera aparece **un** indicador de espera («Searching the standards and writing the answer…») y ningún texto parcial.
- [x] La respuesta aparece **entera de una vez**, con marcadores `[n]`.
- [x] Bajo «Sources», las cláusulas citadas, numeradas como sus `[n]`.
- [x] Sidebar: **Latency (server)** y **Latency (client-measured)** con valores parecidos (en
      torno a 5–8 s); **Truncated: no**; no hay línea de «Time to first token».

## 2. Fuera de dominio

Pregunta: `What is a good recipe for paella?`

- [x] La respuesta empieza con la frase marcador `Out of scope:`.
- [x] No aparece el aviso amarillo de «does not cite any source» (es un rechazo, no una respuesta).
- [x] Sidebar → evidencia de la última consulta: sigue habiendo átomos (lo recuperado viaja igual
      en un rechazo).

## 3. Seguimiento con anáfora (multi-turn)

Sin pulsar «New conversation» tras el paso 1, pregunta: `And for category C?`
(si hiciste el paso 2 en medio, repite antes el paso 1).

- [x] La respuesta habla de la categoría C de software de `ECSS-Q-ST-80C`: el contexto del turno
      anterior se usó.
- [x] El `Session:` del sidebar es el mismo que en el paso 1.

**Sesión perdida (opcional):** con la conversación abierta, `docker compose restart api`, espera a
que esté `healthy` y haz otra pregunta.

- [x] Sale el error con HTTP 404 y el aviso «🔄 This conversation has been reset…».
- [x] La pregunta siguiente funciona y el `Session:` del sidebar es otro.

## 4. Pregunta larga / enumerativa

Pregunta: `List every software development process defined in ECSS-E-ST-40C and, for each one, the main expected outputs.`

- [x] Llega una respuesta completa (puede tardar más que la del paso 1), **sin** error de timeout.
- [x] Si el sidebar marca **Truncated: ⚠️ yes**, en el chat también aparece el aviso de respuesta
      cortada por el límite de tokens. (Que salga o no depende de la longitud; lo que se comprueba
      es que ambos coinciden.)

## 5. Evidencia como antes

En cualquiera de las respuestas con citas:

- [x] El desplegable **«🗂️ Retrieved but not cited»** sigue apareciendo cuando hay cláusulas
      recuperadas no citadas.
- [x] Los átomos añadidos por la expansión siguen marcados con **«➕ added to complete the
      clause»**.

## Resultado

Si algo falla, anota el paso, la pregunta y lo que se ve (captura si es de UI).
