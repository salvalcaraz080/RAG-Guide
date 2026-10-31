# S14 — Brief 3: UAT estructurado y credencial de solo lectura del retrieval

> Independiente de los briefs 1 y 2. Rama `s14-multiagente`. Los commits los hace Salva.
> **Sin llamadas al proveedor.** No se toca el camino de `/query`.
>
> **Regla del brief:** dos piezas pequeñas, una de proceso y una de verificación. La parte B
> **solo reporta**: si la credencial no es de solo lectura, no se refactoriza, se reporta.

---

## 0. Contexto y objetivo

**Parte A — UAT estructurado** (`S14-resumen.md` §9, decisión 2b). Del material: el par
*lo que el sistema propuso / lo que el humano decidió* es el dato más valioso que produce el sistema,
y es irrecuperable si no se guarda. Nosotros ya lo producimos en cada `SNN-uat.md`, pero en prosa:
no se puede consultar, agregar ni promover al golden set. Desde S14 las respuestas de la UAT ya se
guardan en `data/groundtruth/generation/runs/uat_s14_20261030.json`; falta el veredicto ligado a
ellas. Objetivo: que cada caso de UAT sea **un registro** con la respuesta y el juicio, materia prima
del golden set 1.2 (diferido).

**Parte B — credencial de solo lectura** (`S14-resumen.md` §6): el mínimo privilegio efectivo vive en
la credencial, no en la lista de tools. El retrieval solo lee; ¿su conexión puede escribir?

---

## 1. Decisiones (parte A)

| # | Decisión | Porqué |
|---|---|---|
| U1 | **Un registro por caso y ejecución**, en JSONL versionado junto al resto de ground truth (`corpus/groundtruth/uat/`, o donde encaje con lo existente; justifícalo) | Formato agregable y diffeable; vive con el golden set al que alimenta |
| U2 | Campos mínimos: `session` (p. ej. `S14`), `brief`, `case_id`, `turn`, `run`, `query`, `config` (flags que cambian la respuesta + modelo + hash del prompt), huella del corpus (la que ya use el arnés, p. ej. `input_fingerprint`), referencia a la respuesta guardada (fichero + índice; no se duplica), `premise` (qué se espera y contra qué contexto determinista se comprobó), `verdict ∈ ok \| ko \| partial \| spurious \| pending`, `judge ∈ user \| chat \| cc`, `rationale`, `failure_kind` (vocabulario abierto, texto corto), `golden_candidate: bool` | Lo necesario para reproducir el caso y para promoverlo, nada más. Si falta un identificador de petición en las respuestas guardadas, añádelo al **guardado de la UAT**, no al contrato |
| U3 | **Validación con Pydantic** al escribir y al leer; un registro inválido rompe | El registro es un contrato, no notas |
| U4 | **CC escribe los registros** al ejecutar la UAT con `verdict = pending`; al cierre, transcribe el veredicto desde el `SNN-uat.md` (juicio del usuario o del chat) | El juicio sigue redactándose en prosa donde se razona; el registro es su forma consultable |
| U5 | **Backfill solo de S14** (brief 1: la pasada no, la UAT sí, con el juicio de `S14-uat.md` §6). Las UAT anteriores: reporta cuántas serían recuperables y con qué coste, **sin hacerlo** | Empezar a guardar es lo que importa; el histórico se decide con el dato del coste |
| U6 | Un comando mínimo de consulta (resumen por sesión, veredicto y `failure_kind`; lista de `golden_candidate`) | Si no se consulta, es otra prosa |
| U7 | Sin juez LLM, sin UI | Fuera de alcance |

### Para el backfill de S14 (juicio del chat, `S14-uat.md` §6)

- q14 t1 ❗ → `ok`, `golden_candidate: true` (premisa contradicha con evidencia estable).
- q14-a1 ↪️ → las dos ejecuciones de la UAT `ko` (`failure_kind: not_typed`); `golden_candidate: true`.
- q12 ↪️ → `ok` las dos (las remisiones servidas son reales); `golden_candidate: true`.
- Hueco real (caso 3) → `ok`.
- q02 → `ok` (delegación real a Q-ST-10-09; `partial` más honesto que `answered`).
- Degradaciones (caso 4) → `ok` en contrato; render del desplegable `pending` (lo juzga Salva en el
  brief 2).
- Render de los bloques ❗ y ↪️ → `ok`, `judge: user` (Salva, 2026-10-30).
- El ↪️ «Space-to-Ground ICD» es de la **pasada**, no de la UAT: no entra como registro de UAT;
  mencionarlo en `S14-decisions-3.md` como candidato a caso negativo del golden set.

---

## 2. Parte B — credencial del retrieval (solo reporte)

- Qué rol/usuario de base de datos usa el retrieval (y el resto de la app) según `DATABASE_URL` y
  Compose; qué privilegios tiene sobre las tablas que lee (`SELECT` solo, o más); si hay un rol
  distinto para escritura (ingesta, sesiones, migraciones).
- Qué haría falta para que el retrieval use un rol de solo lectura (en una frase: rol nuevo,
  segunda URL, coste estimado). **No se implementa.**
- Si encuentras otra superficie de escritura alcanzable desde el camino de `/query` (además de las
  sesiones, que son legítimas), lístala.

---

## 3. Tests (sin red)

- `uv run pytest` en verde. Recuento frente al cierre del brief 2.
- Esquema del registro: válido, inválido (veredicto fuera de vocabulario, referencia a respuesta
  inexistente), ida y vuelta JSONL.
- Backfill de S14: los registros existen, apuntan a respuestas existentes y su veredicto casa con
  §1 de este brief.
- Comando de consulta sobre un fichero sintético.

---

## 4. UAT

Mínima, por CC: ejecutar el comando de consulta sobre los registros de S14 y pegar la salida en
`S14-uat-3.md`. Es la primera prueba de que el formato sirve: el chat la juzga al consolidar.

---

## 5. Cierre

- `S14-decisions-3.md`: ubicación y forma del registro (y por qué), coste del backfill histórico,
  reporte de la credencial, recuento de tests, etiquetado agnóstico/dominio.
- `S14-uat-3.md`.
