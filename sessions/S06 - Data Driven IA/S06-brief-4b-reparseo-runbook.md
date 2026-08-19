# S06 — Brief 4b: Re-parseo del corpus completo + RUNBOOK del pipeline

> **Tipo:** comando de pipeline + documentación de ejecución. Cierra la brecha que dejó el Brief 4:
> la numeración reparada vive en el parser, pero `data/processed/` conserva los 14 JSON viejos **con
> numeración obsoleta**. La reparación es teórica hasta que llega a los artefactos.
>
> **Doble entregable:** (1) un comando que regenera `data/processed/` desde `parse_ready/` con el
> parser reparado, y (2) `corpus/RUNBOOK.md` — la secuencia de ejecución del pipeline de datos, que
> hasta ahora vivía solo en la cabeza del usuario. Van juntos: un comando sin orden documentado es
> frágil; un runbook sin comando es una lista de deseos.
>
> **Frontera CC:** CC escribe el comando + tests (mockeados, sin `data/`, sin LibreOffice) y redacta
> el RUNBOOK. **Ejecutar el re-parseo sobre los 26 reales y verificar `processed/` es UAT del
> usuario.**

---

## Contexto: por qué ahora, antes del Brief 5 (tablas)

El Brief 4 dejó tres tensiones, y esta las cierra:

- **La cobertura sigue en 54%** — `processed/` tiene 14 JSON, ahora además con numeración obsoleta.
- **No hay comando de re-parseo** — CC lo dijo explícito: la reparación no puede llegar a los
  artefactos porque no hay forma de regenerarlos.
- **El grafo quedará desincronizado** — se construye desde `processed/`; re-parsear con numeración
  nueva lo deja stale.

Se hace **antes que tablas** porque el Brief 5 ancla contenido tabular a `clause_id` de sección: hay
que validar primero que la numeración reparada llega limpia a los JSON sobre el corpus sin tablas, y
luego tablas es un incremento sobre una base ya correcta. Re-parsear después de tablas acoplaría las
dos cosas.

---

## Decisiones ya tomadas (Claude Chat) que el comando implementa

1. **`processed/` se regenera desde cero.** Borrar el contenido anterior y reconstruir los 26 JSON,
   no actualizar in-place. `processed/` es un artefacto derivable y desechable (igual que
   `parse_ready/`): su fuente de verdad es `parse_ready/` + el parser. Nada de valor se pierde
   borrándolo, y borrar evita huérfanos (los 14 JSON viejos con esquema/numeración antiguos no deben
   sobrevivir mezclados con los 26 nuevos).

2. **El grafo NO se regenera aquí — queda stale-declarado.** El grafo (`references.json`,
   `graph.graphml`) se construye desde `processed/` y quedará desincronizado (sus nodos usan la
   numeración vieja). **No se regenera** por dos razones: (a) regenerarlo sobre 20 documentos con el
   extractor **sin validar** amplifica falsos positivos no medidos (recon del Brief 2), y (b) hay una
   decisión de arquitectura abierta sobre si el grafo `nx.DiGraph` debe existir siquiera, o si la
   trazabilidad de referencias se resuelve con metadata de chunk (anclada a S07 — ver «Decisión
   abierta» abajo). Marcar el grafo como stale **explícitamente** — no dejarlo mintiendo en silencio.
   La marca debe ser visible: un fichero centinela, un campo en el manifiesto (Brief 6), o al mínimo
   un `WARNING` al cargar el grafo que diga «construido sobre numeración pre-Brief-4, pendiente de
   regenerar/reevaluar». Stale-declarado sirve a cualquier futuro del grafo (regenerar, sustituir,
   eliminar); stale-silencioso es el pecado de siempre.

3. **Se produce `corpus/RUNBOOK.md`** con la secuencia completa del pipeline.

---

## El comando de re-parseo

Un comando (`python -m corpus.parse_all` o el nombre que CC juzgue coherente con
`corpus.downloader`/`corpus.graph`) que:

1. Verifica que `parse_ready/` existe y tiene contenido (si no, error claro que remite al paso de
   conversión — Brief 2). No re-convierte ni toca `raw/`; asume `parse_ready/` ya construido.
2. **Borra `data/processed/`** (o su contenido) y lo reconstruye.
3. Parsea cada `.docx` de `parse_ready/` con el parser reparado → un JSON por documento en
   `processed/`.
4. **Resumen estructurado** (structlog, coherente con Brief 1b/2): documentos parseados, requisitos
   y secciones por documento, fallos de parseo con motivo. Exit code ≠ 0 si algún documento esperado
   no produjo JSON. Reutiliza el patrón de resumen-como-dato de los briefs anteriores.
5. **Marca el grafo como stale** al terminar (decisión 2): el comando sabe que acaba de invalidar el
   grafo, así que es el sitio natural para dejar la marca — no una acción manual que alguien olvidará.
6. **Idempotente en el sentido correcto:** re-ejecutar regenera desde cero otra vez (no hay estado
   que preservar en `processed/`; es derivado). Sin `--force` porque no hay nada que forzar — siempre
   regenera.

**Verificación integrada:** tras regenerar, el comando (o `verify_parser.py` invocado a
continuación) debe reportar la **cobertura** (26/26 documentos con JSON) y la **cobertura de
identidad** (secciones con número resuelto vs sin número) que el Brief 4 añadió. El re-parseo no está
«hecho» si no deja esa medida visible.

---

## `corpus/RUNBOOK.md` — la secuencia de ejecución

Hasta S06, `corpus/` estuvo congelado y no se ejecutaba; ahora es un pipeline de pasos manuales que
hay que correr en orden, y ese orden solo vive en la cabeza del usuario. El RUNBOOK lo hace
explícito y reproducible. Contenido mínimo:

**La secuencia completa, en orden, con qué produce cada paso y qué lo dispara:**

```
0. corpus/scraper.py      → data/inventory/*.csv      (índice ECSS → inventario)
1. corpus/downloader.py   → data/raw/                 (inventario → DOCX/DOC descargados)
2. corpus/build_parse_ready → data/parse_ready/       (raw → .docx homogéneo; convierte .doc)   [Brief 2]
3. corpus/parse_all       → data/processed/*.json     (parse_ready → JSON, numeración resuelta)  [este brief]
4. corpus/verify_parser   → (reporte)                 (cobertura + cobertura de identidad)
5. corpus/graph           → data/processed/graph.*    (STALE — ver nota; no correr hasta reevaluar) [Brief 7]
```

Para cada paso, el RUNBOOK documenta: **comando exacto**, **qué lee y qué escribe**, **precondición**
(qué paso anterior tiene que haber corrido), **qué es UAT del usuario** (todo lo que toca `data/` real
y LibreOffice), y **cómo verificar que salió bien** (el criterio observable, no el exit code — la
lección de «el éxito lo decide el artefacto»).

**Notas que el RUNBOOK debe llevar:**
- **Requisitos de entorno:** LibreOffice para el paso 2 (con el detalle `soffice.com` en Windows del
  Brief 2), Python/uv. Qué pasos necesitan red (0, 1) y cuáles no.
- **Qué es derivable y desechable:** `parse_ready/` y `processed/` se reconstruyen desde `raw/`;
  `raw/` se reconstruye desde el inventario; el inventario desde el scraper. La única cosa cara de
  recuperar es lo que el servidor pueda haber dejado de ofrecer.
- **El grafo (paso 5) está STALE y no debe correrse** hasta que se reevalúe su arquitectura (S07) y
  se valide el extractor (Brief 7). El RUNBOOK lo marca claramente, no lo lista como paso rutinario.
- **Frontera CC/usuario:** el RUNBOOK es también donde queda escrito, de una vez, qué ejecuta el
  usuario (todo el pipeline sobre datos reales) y qué hace CC (escribir el código, testear mockeado).

El RUNBOOK no es histórico ni diario — es estado presente operativo, como `ARCHITECTURE.md`. Vive en
`corpus/` porque es el manual de operación de ese subsistema.

---

## Decisión abierta que este brief NO toca (anclada a S07)

**Grafo `nx.DiGraph` vs metadata de chunk.** Registrada aquí porque el re-parseo la roza (deja el
grafo stale) pero no la resuelve:

> Fuimos a la infraestructura de grafo por inercia — «hay referencias cruzadas» → «hace falta un
> grafo» — sin razonar si el producto **necesita** razonar sobre referencias o solo **registrar**
> cuáles hay. Producción ya iba a usar solo un índice de adyacencia (`{source: [targets]}`), no el
> `nx.DiGraph` (CLAUDE.md). La pregunta abierta: ¿la trazabilidad de referencias se resuelve con
> **metadata de chunk** (una tabla `chunk_id → cláusulas referenciadas` + índice inverso para
> entrantes) en vez de infraestructura de grafo? El `nx.DiGraph` solo se justifica por multi-hop real
> (caminos, centralidad), que probablemente es M3-hipotético, no M2. **Depende de dos cosas que aún
> no existen:** la estrategia de chunking (S07 — define qué es un chunk y qué metadata lleva) y la
> validación del extractor contra ground truth (Brief 7 — define si las referencias son fiables; 49%
> ya apuntan a documento entero, no a cláusula). Decidir antes de esas dos es volver a pre-emptar el
> nodo. **Reevaluar en S07.**

Este brief solo asegura que el grafo stale no se propague como si fuera válido.

---

## Qué entrega CC

1. **El comando de re-parseo** (`corpus/parse_all` o análogo) — regenera `processed/` desde cero,
   resumen estructurado, exit code correcto, marca el grafo stale.
2. **`corpus/RUNBOOK.md`** — la secuencia completa 0→5 con comando, IO, precondición, frontera
   CC/usuario y criterio de verificación por paso; el grafo marcado STALE.
3. **Tests** (mockeados, sin `data/`): la lógica de «borrar y regenerar» sobre un `processed/`
   sintético; el conteo del resumen; el exit code ante un documento que no produce JSON; que la marca
   de stale del grafo se emite. `uv run pytest` verde.
4. **Guion UAT** — el usuario ejecuta `parse_all` sobre `parse_ready/` real y verifica: 26 JSON en
   `processed/`, cobertura 26/26, cobertura de identidad `N/N` en todos, y —el cierre del Brief 4—
   **re-ejecutar el diagnóstico de numeración contra los JSON nuevos da ~0 divergencias** (con la
   advertencia del Brief 4: eso es cableado, la verdad es el render; incluir 2-3 verificaciones a ojo
   de las cláusulas que estaban rotas, p.ej. `Q-ST-30-02C E.1.1`, `M-ST-40C L.5`, y una de
   `Q-ST-20C Rev.2` que el Brief 4 destapó).
5. **`S06-decisions.md`** (append), etiquetado agnóstico/dominio.

## Qué NO hace CC

- No regenera el grafo (stale-declarado; su futuro es S07/Brief 7).
- No parsea tablas (Brief 5) — este re-parseo es sobre el parser reparado *sin* tablas todavía.
- No construye el manifiesto (Brief 6).
- No toca `raw/` ni re-convierte `parse_ready/` (asume el Brief 2 ya corrido).
- No ejecuta sobre `data/` real (UAT del usuario).

## Criterio de "hecho"

- `processed/` regenerado desde cero con 26 JSON de numeración correcta; los 14 viejos no sobreviven.
- El comando deja resumen estructurado y exit code correcto; marca el grafo stale de forma visible.
- `corpus/RUNBOOK.md` documenta la secuencia 0→5 ejecutable, con frontera CC/usuario y verificación
  por paso; el grafo marcado STALE y fuera de la rutina.
- Tests verdes mockeados.
- Guion UAT entregado: cobertura 26/26, identidad N/N, diagnóstico ~0 divergencias sobre los JSON
  nuevos + verificación a ojo de las cláusulas reparadas.
- La decisión abierta grafo-vs-metadata queda registrada y anclada a S07, no resuelta a la ligera.
