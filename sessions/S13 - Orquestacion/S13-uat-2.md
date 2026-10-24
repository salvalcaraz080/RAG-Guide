# S13 · UAT — Brief 2: punteros a un estándar entero

> **Por qué es tuyo:** se juzga leyendo la respuesta (si lo que cita de M-ST-80C responde a la
> pregunta) y cambia temporalmente la configuración del contenedor. CC ya ha comprobado el
> contrato (1.470 tests, `clause` byte a byte igual), la comprobación previa sin generación y la
> pasada del arnés (`S13-decisions-2.md` §2).
>
> **Coste:** tres preguntas a Haiku 5.5 (< 0,01 USD), unos 60 s.
>
> **No hace falta la verificación independiente.** Solo q12-a2 se resuelve, y en el caso 1 se ve
> leyendo una cita.

## 0 · Encender `clause_standard`

Abre `.env` y cambia **una línea**, `RETRIEVAL_XREF=clause` → `RETRIEVAL_XREF=clause_standard`.
Después:

```powershell
docker compose up -d --force-recreate api
docker compose exec api python -c "from app.config import get_settings; print(get_settings().RETRIEVAL_XREF)"
```

**Esperado:** `clause_standard`.

## 1 · q12: cita M-ST-80C donde antes había `not_found`

En la UI, con **🔄 New conversation**:

```
How does Q-ST-80C expect software-related risks to be identified and handled, and how are they supposed to feed into the project's overall risk management process?
```

**Esperado:**

- La respuesta cita **ECSS-M-ST-80C** (6.1, 6.3, 5.2.2.1 o E.3). En la pasada, las 3 ejecuciones
  citaban 6.1: «The results of risk management are considered in the routine project management
  process…».
- En la evidencia, esas cláusulas salen con **«➕ by cross-reference»**.
- Sigue habiendo algún `not_found`: los criterios de aceptación (q12-a3) y el contenido de la
  política (q12-a4) no entran (`S13-decisions-2.md` §2.3).

**Juicio:** ¿lo que cita de M-ST-80C responde a «cómo alimentan los riesgos de software la
gestión de riesgos del proyecto»?

**Resultado:** OK

## 2 · q14: **no** cambia, y eso es lo esperado

Tres turnos en la misma conversación:

```
Our on-board software has to implement the packet utilization standard. Which services are we obliged to support and which are optional?
```
```
How do we document the subset of services and subtypes that our mission actually uses?
```
```
And if the spacecraft is also supposed to run on-board control procedures, what does the flight software have to provide to execute them?
```

**Esperado:** el tercer turno **no** cita ECSS-E-ST-70-01C. El puntero queda `over_limit`: las 5
plazas ya las llenan punteros a cláusula de E-ST-70-41C que aparecen antes. Para confirmarlo en
el log:

```powershell
docker compose logs api --since 10m | Select-String "references_expanded" | Select-String "E-ST-70-01C"
```

Ahí debe verse `'doc': 'ECSS-E-ST-70-01C', 'status': 'over_limit'`.

Esto **difiere del brief**, que esperaba la cita: es el efecto de W4 que mostró la comprobación
previa.

**Resultado:** KO. Los dos primeros mensajes dicen "I could not find this...". Y el tercer mensaje cita 3 clausulas de ECSS-E-ST-70-41C. Repite tú esta prueba y analiza el resultado.

**Repetida por CC** (2026-10-23): los tres turnos en una sesión contra `POST /api/v1/query`, con
`clause_standard` activo en el contenedor. El resultado es el mismo que viste.

- **Turno 3: es el comportamiento medido, no un fallo de esta implementación.** Cita E-ST-70-41C
  6.18.1.1, 5.4.3.1 y 4.2.4. El log muestra `ECSS-E-ST-70-01 → ECSS-E-ST-70-01C, over_limit`: las
  5 plazas ya las ocupan punteros a cláusula de E-ST-70-41C que aparecen antes. Es lo que predijo
  la comprobación previa y lo que dio la pasada (q14-a1 `over_limit` ×3).
  - El coste para el producto se ve en la propia respuesta. Cita «…complements the spacecraft
    on-board control procedures standard (refer to ECSS-E-ST-70-01)» y, a la vez, declara
    `not_found` «requirements of the on-board control procedures standard that the OBCP service
    complements». El sistema ve el puntero y no lo sigue, por W4.
  - El fallo del script es que lo marcaba como «esperado» sin explicar que, para el usuario, eso
    es un hueco.
- **Turnos 1 y 2: no los causa este brief.** El bloque `not_found` ya salía en esos dos turnos en
  la referencia (`generation_20261019-224307`) y en `control` (`xref+bounded`) de la pasada de hoy.
  El script no decía qué esperar de ellos: es un fallo del script, no del sistema.
  - Lo que sí se ve en el turno 1 (anterior a este brief): dice «all services are optional» y
    declara `not_found` «which services are mandatory». La pregunta da por hecho algo que el
    estándar niega, y la respuesta no lo dice así: lo trata como un hueco (clase C,
    `answered_as_phrased`).

**Veredicto CC:** el comportamiento es el previsto por el diseño (W4), y el KO apunta a dos
cosas:

1. El coste de producto del orden de aparición con el tope compartido. Es una decisión del chat,
   y está en `S13-decisions-2.md` §2.4.
2. Un script de UAT que no describía los turnos 1 y 2.

## 3 · Una consulta sin punteros a estándar no cambia

```
What does ECSS-Q-ST-80C require for software criticality classification?
```

**Esperado:** respuesta del mismo tipo que con `clause`. En la evidencia no hay ninguna cláusula
«➕ by cross-reference» de un documento distinto de los que ya salían. Si el texto nombra algún
estándar entero, el log lo muestra en `standards` con su estado.

**Resultado:** Repite tú esta prueba y compara.

**Repetida por CC** (2026-10-23): la misma pregunta con `clause_standard` y, tras recrear `api`,
con `clause`. Se compara el contexto, que es determinista, no el texto de la respuesta, que no lo
es.

- **La premisa del caso estaba mal elegida: esta consulta sí tiene punteros a estándar.** El
  contexto de Q-ST-80C incluye 5.3.1, que nombra «ECSS-M-ST-80».
  - Con `clause_standard`, M-ST-80C se sigue y añade **7 chunks** (E.3, E.4, A.1.1, C.1.1,
    5.2.1.2, B.1.1 y 7.1), anexos de gestión de riesgos que no tienen que ver con la criticidad.
  - En el log de la UAT hubo otra petición que siguió E-ST-40C y M-ST-40C y registró más de 60
    entradas `already_searched` de «ECSS-Q-ST-80», una por mención.
    - **Corrección (brief 3): esa petición no era este caso.** La traza de esta consulta,
      repetida solo con retrieval, no tiene ninguna entrada `already_searched`.
    - No se puede saber qué consulta era: los logs se perdieron al recrear `api` (el log no
      guarda la pregunta).
    - El fenómeno (una entrada por mención) sí se reproduce en el artefacto del brief 2
      (`S13-decisions-3.md`).
- **Lo que no cambia:** los 27 chunks de `clause` están todos en `clause_standard`, en el mismo
  orden; lo nuevo va detrás. Ninguna de las dos respuestas cita M-ST-80C.
- **Lo que cambió en esta pareja:**

  | | `clause` | `clause_standard` |
  |---|---|---|
  | Estado | `answered` | `partial` (dos `not_found` sobre criterios por categoría) |
  | Latencia | 6,6 s | 18,2 s |
  | Coste | 0,0013 USD | 0,0032 USD |
  | Tokens de entrada | 9.464 | 10.494 |

  Es una ejecución de cada: no se puede atribuir a la opción. Generar no es determinista, y la
  pasada (42 frente a 42) da +1,2 s de mediana, no +11 s.
- **Dato agregado de la pasada.** Con `clause_standard`, lo que solo trajo un estándar seguido
  entra en 10 de 14 consultas: 207 cláusulas en 30 ejecuciones. Se citan 100: 71 relevantes según
  el golden set, 15 sin juicio y 14 no relevantes. En q01 y q03 se añade y no se cita nunca (12 y
  21 cláusulas).

**Veredicto CC:** OK en lo que el caso puede comprobar: el contexto de `clause` queda intacto y lo
nuevo va al final. Pero el caso no sirve para «sin punteros», y deja dos observaciones para el
chat:

1. **Seguir un puntero no implica que el destino sea relevante.** El retrieval filtrado siempre
   devuelve algo, porque no hay umbral de distancia, y un puntero genérico (la remisión de 5.3.1 a
   M-ST-80) se sigue en cualquier consulta que inyecte esa cláusula.
2. **Ruido en la traza.** `already_searched` y `out_of_corpus` se registran una vez por mención,
   sin deduplicar por documento. No afecta al contexto. El arreglo, solo en la traza, es pequeño:
   CC lo propone y no lo ha hecho.

## 4 · Latencia (de la pasada, no de la UI)

Del desglose en `S13-decisions-2.md` §2.5:

- domina `llm` (93–94 % del turno);
- `xref` pasa de 60 a 128 ms de mediana con `clause_standard` (+68 ms);
- el total sube 1,2 s, casi todo en `llm` (más contexto y más salida).

Nada que ejecutar: confirma que lo has leído.

**Resultado:** OK.

## 5 · Volver al default

Deja `.env` otra vez en `RETRIEVAL_XREF=clause` y:

```powershell
docker compose up -d --force-recreate api
docker compose exec api python -c "from app.config import get_settings; print(get_settings().RETRIEVAL_XREF)"
```

**Esperado:** `clause`.

**Resultado (CC):** hecho al repetir el caso 3. `.env` ya estaba en `clause`, `api` recreado, y
`get_settings().RETRIEVAL_XREF` devuelve `clause`.
