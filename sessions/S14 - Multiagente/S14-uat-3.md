# S14 · UAT — Brief 3: consulta de los registros

> **Ejecutada por CC** (2026-10-30). Salida literal del comando, sin llamadas. La juzga el chat al
> consolidar: es la primera prueba de que el formato sirve.

```powershell
uv run python -m corpus.groundtruth.uat.records check
uv run python -m corpus.groundtruth.uat.records summary --session S14
uv run python -m corpus.groundtruth.uat.records candidates
```

## check

```
20 records valid, every response found.
```

## summary

## S14

| brief | case | runs | verdicts | failure kinds | golden |
|---|---|---|---|---|---|
| 1 | S14-1-q14-t1 | 2 | ok 2 | — | yes |
| 1 | S14-1-q14-t2 | 2 | ok 2 | — |  |
| 1 | S14-1-q14-a1 | 2 | ko 2 | not_typed | yes |
| 1 | S14-1-q12 | 2 | ok 2 | — | yes |
| 1 | S14-1-q05 | 1 | ok 1 | — |  |
| 1 | S14-1-q02 | 1 | ok 1 | — |  |
| 1 | S14-1-render-blocks | 2 | ok 2 | — |  |
| 1 | S14-1-degradations | 2 | ok 2 | — |  |
| 2 | S14-2-panel | 5 | ok 5 | — |  |
| 2 | S14-2-panel-look | 1 | pending 1 | — |  |

Records 20 · ko 2, ok 17, pending 1 · failure kinds: not_typed 2

## candidates

- S14 S14-1-q14-t1 turn 1 run 1 · ok · Our on-board software has to implement the packet utilization standard. Which services are we obliged to support and which are optional? · uat_s14_20261030.json#first/q14/0
- S14 S14-1-q14-t1 turn 1 run 2 · ok · Our on-board software has to implement the packet utilization standard. Which services are we obliged to support and which are optional? · uat_s14_20261030.json#repeat/q14/0
- S14 S14-1-q14-a1 turn 3 run 1 · ko (not_typed) · And if the spacecraft is also supposed to run on-board control procedures, what does the flight software have to provide to execute them? · uat_s14_20261030.json#first/q14/2
- S14 S14-1-q14-a1 turn 3 run 2 · ko (not_typed) · And if the spacecraft is also supposed to run on-board control procedures, what does the flight software have to provide to execute them? · uat_s14_20261030.json#repeat/q14/2
- S14 S14-1-q12 turn 1 run 1 · ok · How does Q-ST-80C expect software-related risks to be identified and handled, and how are they supposed to feed into the project's overall risk management process? · uat_s14_20261030.json#first/q12/0
- S14 S14-1-q12 turn 1 run 2 · ok · How does Q-ST-80C expect software-related risks to be identified and handled, and how are they supposed to feed into the project's overall risk management process? · uat_s14_20261030.json#repeat/q12/0

**Lectura de CC:** el formato responde sin abrir la prosa qué falló y por qué (q14-a1, `not_typed`,
2 de 2), qué queda por juzgar (el aspecto del desplegable) y qué promover al golden set. Lo único
`pending` es el registro `S14-2-panel-look`.
