# Auditoría pre-M4 — saneamiento entre S08 y S09

Todo el trabajo hecho entre el cierre de S08 (Módulo 3, bases vectoriales) y el arranque de S09
(Módulo 4, pipeline RAG). No es una sesión del máster: es andamiaje para llegar a S09 con el
terreno despejado.

Ocurrió en **dos rondas**, y cada una tiene su propia serie de etiquetas `A/B/C/D`.

> ⚠️ **Las etiquetas de las dos rondas chocan.** «C1» es un brief de docs en las dos, pero son
> **dos briefs distintos**; «C2» y «F2» significan cosas sin relación entre una ronda y otra.
> Las etiquetas no se renumeraron porque ya son historia commiteada en el `CHANGELOG.md` de
> RAG-ECSS. **Al citar una etiqueta de
> este tramo, di la ronda:** «C1 de la ronda 2», no «C1».

---

## Ronda 1 — la auditoría (9–10 sep)

Ciclo **A → B → C2 → C1 → D → F1 → F2 → F3 → F4**, cerrado en `CHANGELOG` 0.36.0.

| Paso | Qué | Archivos |
|---|---|---|
| **A** | Auditoría de estado read-only: CC reporta hechos, no decide | `initial-audit-pre-M4-audit-brief.md` · `initital-audit-pre-M4-report.md` |
| **B** | Clasificación de la deuda en Claude Chat, sobre el informe de A | *(sin archivo aquí)* |
| **C2** | Fixes transversales de código | `pre-M4-brief-codigo.md` · `pre-M4-decisions-codigo.md` |
| **C1** | Reconciliación de docs con la realidad ya arreglada por C2 | `pre-M4-brief-docs.md` |
| **D** | Diagnóstico y triaje de lo que quedaba | `pasoD-brief-diagnostico.md` · `pasoD-diagnostic-report.md` |
| **F1** | Filtro de marcadores no citables + oráculo de completitud | `F1-brief-filtro-oraculo.md` · `F1-decisions.md` |
| **F2** | Seam de embeddings async nativo | `F2-brief-embedding-async.md` · `F2-decisions.md` |
| **F3–F4** | Smalls de la capa app, y de corpus y config | `F3-F4-brief-smalls.md` · `F3-F4-decisions.md` |

C2 va antes que C1 a propósito: los docs se reconcilian contra el código **ya arreglado**.

## Ronda 2 — refactor, limpieza y traza (11–30 sep)

Archivos con prefijo **`ronda2-`**. Empezó como una sesión de refactor con su propio informe
(bloques A, B y el hallazgo F2), y siguió con los briefs de Claude Chat que ese informe dejó
pendientes.

| Paso | Qué | Archivos |
|---|---|---|
| **A**, **B** | Informe de simplificación y sus dos bloques: rutas, identidad, texto DOCX y preparación compartida en `llm_service` | *(el informe se entregó en el chat; el detalle está en `CHANGELOG` 0.37–0.38)* |
| **F2** | La conservación de tablas se medía contra sí misma; ahora contra el DOCX crudo | *(en el chat; `CHANGELOG` 0.39)* |
| **C2–C7, D1, D2** | Limpieza de lo ya decidido | `ronda2-limpieza-decisions.md` |
| **C1** | Reorganización de docs: un hogar por decisión | `ronda2-docs-routing-map.md` |
| — | Traza estructurada en el contrato de respuesta, y cierre de su UAT | `ronda2-traza-decisions.md` |
| — | Resumen de cierre para Claude Chat | `ronda2-resumen-para-chat.md` |

**Si solo vas a leer uno, lee `ronda2-resumen-para-chat.md`**: es autocontenido y cubre el tramo
entero, incluidas las decisiones que el cable de S09 activa.

---

## Qué queda abierto al cerrar

Las tres decisiones que S09 activa (fan-out de citas, recomposición de cláusula, modo de fusión)
y el resto de pendientes, con sus cifras, están en `ARCHITECTURE.md` §7 de RAG-ECSS. Los
candidatos agnósticos a graduar están en `ronda2-resumen-para-chat.md` §6.
