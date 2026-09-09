# Brief — Auditoría de estado (chat puente S08 → S09), Paso A

**Tipo:** auditoría **read-only**. Andamiaje de continuidad entre sesiones, no un brief de
sesión del máster ni un paso del ciclo de seis pasos. Excepción declarada al ciclo.

**Rol de CC en este paso:** investigar y **reportar hechos**. No decide disposiciones. La
clasificación de cada deuda (fantasma / futura / transversal) se hace en Claude Chat sobre este
informe. CC aporta la evidencia; el chat decide qué se hace con ella.

---

## Qué es este paso y qué NO es

Estamos cerrando S08 (capa de datos vectorial) antes de entrar en S09 (pipeline RAG). Antes de
tocar nada, auditamos el estado real del proyecto y el estado real de la deuda documentada.

Esto es un **diagnóstico**, y el diagnóstico y la reparación son tareas distintas. La salida de
este paso **es la especificación** de los pasos siguientes (actualización de docs, briefs
transversales, auditoría de código). Por eso aquí no se repara nada.

### En alcance (SOLO lectura + informe)

- Leer libremente: `app/`, `corpus/`, `frontend/`, `migrations/`, `tests/`, `scripts/`,
  `sesiones/*`, `CHANGELOG.md`, y los artefactos de `data/processed/` (metadatos, conteos —
  no hace falta volcar contenido).
- Ejecutar herramientas **read-only** para establecer hechos: `uv run pytest`, y las diagnósticas
  que **ya existen** en `corpus/diagnostics/` (`verify_parser`, `audit_raw_corpus`,
  `diagnose_numbering`, `diagnose_tables`, `inspect_graph`, etc.). **Reutilizar lo que ya mide;
  no escribir tooling nuevo** para esto.
- Producir **un único informe**: `sesiones/bridge-S08-S09/audit-report.md`.

### Fuera de alcance (NO hacer en este paso)

- **No escribir ni editar** ningún doc (`CLAUDE.md`, `ARCHITECTURE.md`, `CHANGELOG.md`),
  ningún código, ninguna migración. El único archivo que CC crea es el informe.
- **No clasificar** deuda como transversal/futura/fantasma. Reportar estado fáctico, no
  disposición (ver vocabulario abajo).
- **No cazar** code smells, refactors ni mejoras de forma abierta. La auditoría de calidad de
  código es un paso posterior (Paso D) y se hará **después** de corregir la deuda conocida, para
  no auditar código que vamos a cambiar. Aquí solo se lee código para **verificar afirmaciones de
  los docs y estado de deuda**, no para buscar trabajo nuevo.
- **No commitear** (regla estándar del proyecto).

---

## Tarea 1 — Chequeo de realidad de la deuda documentada

Recorrer **todos** los items de deuda documentados y reportar el estado fáctico de cada uno. El
texto completo de cada item está en los docs; CC lo lee ahí, no hace falta transcribirlo. Fuentes,
por ubicación:

- **(A)** `.claude/CLAUDE.md` → sección **"Decisiones pendientes"** (todos los items `- [ ]`).
- **(B)** `ARCHITECTURE.md` → §7 **"Qué no está resuelto todavía"**.
- **(C)** `.claude/CLAUDE.md` → bloques finales **"Grafo de referencias"** / **"Deuda conocida del
  extractor"** (incluye la "Deuda viva" del extractor sin validar contra ground truth).

Varios items de (B) y (C) son el mismo que uno de (A) contado desde otro sitio (p. ej. recomponer
cláusula, fan-out de citas, las 189 tablas, el grafo sin validar). **Reconciliarlos: un item por
deuda real, anotando en qué docs aparece.** Si dos entradas describen la misma deuda con estados
distintos, eso mismo es un hallazgo (ver Tarea 2).

Para **cada** item, el informe da:

| Campo | Contenido |
|---|---|
| **Ref** | Dónde está documentado (doc + sección/línea; todas las apariciones si es más de una) |
| **Deuda (1 línea)** | Qué afirma el doc que está pendiente |
| **Estado** | Uno de los cuatro de abajo — nada más |
| **Evidencia** | El hecho que sostiene el estado: `archivo:línea`, resultado de test/diagnóstica, conteo de `grep`, salida de una herramienta. Sin evidencia no hay estado |

**Vocabulario de estado (exactamente estos cuatro, sin sinónimos, sin disposición):**

- `VIGENTE` — el código/artefactos confirman que la deuda sigue existiendo tal como se describe.
- `RESUELTA` — ya está corregida de facto en el repo; el doc quedó desactualizado.
- `NO_APLICA` — nunca fue deuda real, o dejó de aplicar por un cambio posterior. Decir **por qué**.
- `INDETERMINADA` — no se puede establecer el estado sin una decisión o información que CC no
  tiene. Decir **qué falta** para resolverlo.

**Prohibido** en este informe cualquier etiqueta de disposición ("transversal", "futuro",
"diferible", "de S09", "de M3"). Eso lo decide el chat. Si CC cree que un item pertenece
claramente a una sesión futura, lo deja en `VIGENTE` y lo dice en observaciones — pero no lo
clasifica.

---

## Tarea 2 — Deriva docs ↔ repo

Aparte del estado de la deuda, reportar **discrepancias entre lo que los docs afirman y lo que el
repo hace de verdad**. Dos direcciones:

1. **El doc afirma algo que el código contradice** (afirmación obsoleta o falsa).
2. **El código hace algo estructural que los docs no reflejan** (deuda o estado no documentado).
   No enumerar cada función — solo afirmaciones estructurales: capas, seams, endpoints, flujo de
   petición, contratos, estado de módulos.

**Pista concreta ya detectada** (confirmar, no asumir): `ARCHITECTURE.md` §7 dice *"Ingesta y
`/search` sin implementar … `services/retrieval.py` no existe todavía"*, pero `CLAUDE.md` describe
S08 cerrada con ingesta cargada (26 docs / 13.876 chunks), `retrieval.py` con modos vector+híbrida
y `/search` expuesto. Si eso es así, `ARCHITECTURE.md` §7 (y posiblemente §5 y la tabla de módulos
de §2, que marca M2 "en curso") van **por detrás** de `CLAUDE.md`. Verificar contra el código cuál
es el estado real y **listar todas las afirmaciones de `ARCHITECTURE.md` que hayan quedado atrás**,
no solo esta.

Para establecer el estado positivo real (lo que los docs dicen que SÍ está hecho), ejecutar
`uv run pytest` y reportar pass/fail y cobertura de suites (contrato app / capa de datos / corpus).
Un test verde también es un hecho.

---

## Formato del informe

`sesiones/bridge-S08-S09/audit-report.md`, con estas secciones:

1. **Resumen** — conteos: nº de items por estado (`VIGENTE` / `RESUELTA` / `NO_APLICA` /
   `INDETERMINADA`), y nº de derivas docs↔repo.
2. **Estado de la suite de tests** — salida de `uv run pytest` (pass/fail por suite).
3. **Tabla de deuda** (Tarea 1) — un item por fila, las cuatro columnas de arriba, reconciliada.
4. **Deriva docs ↔ repo** (Tarea 2) — dirección (1 o 2), afirmación, ubicación, hecho que la
   contradice.
5. **Observaciones** — cualquier cosa que CC necesite señalar sin clasificar (incluye aquí
   corazonadas de "esto es de una sesión futura" o "esto huele a artefacto muerto", como nota, no
   como veredicto).

---

## Transporte

Chat → repo: este brief baja a `sesiones/bridge-S08-S09/`.
CC → chat: `audit-report.md` se sube al chat cuando esté. Con ese informe clasificamos aquí
(Paso B) y de ahí salen la actualización de docs y los briefs transversales (Paso C).
