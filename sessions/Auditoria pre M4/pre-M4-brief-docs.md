# Brief C1 — Reconciliación de docs con la realidad (chat puente, Paso C)

**Segundo y último brief del Paso C. C2 ya está cerrado y verificado** (854 tests, árbol sin
commitear). C1 hace que `CLAUDE.md`, `ARCHITECTURE.md` y el `.env` local describan el estado **real
y ya arreglado**. Es reconciliación de documentación: **cero código, cero tests, cero migraciones.**

**Insumos (la especificación):**
- `sessions/Auditoria pre M4/initital-audit-pre-M4-report.md` — las derivas R1–R14 y las
  recategorizaciones D27/D28.
- `sessions/Auditoria pre M4/pre-M4-decisions-codigo.md` — qué cambió C2 y las dos notas de deuda
  que deja para documentar.

**Ficheros que C1 puede tocar — y solo estos:** `.claude/CLAUDE.md`, `ARCHITECTURE.md`, `.env`
(local). **No tocar** `corpus/RUNBOOK.md` ni `CHANGELOG.md` (los cerró C2), ni `.env.example` (C2 ya
metió ahí `WEB_CONCURRENCY`), ni ningún `.py`.

**Regla de convergencia:** la lista de "Decisiones pendientes" debe salir **más corta**, no más
larga. Salen tres (D13, D27, D28), entran dos (EXPECTED_RETRACTED, §6) y se reformulan tres (D15,
D17/D14, D29). No reescribir sí que está bien; converger, no engordar.

**Verificar contra el repo, no de memoria.** Donde una afirmación sea ambigua, CC la comprueba
contra el código antes de reescribirla. La reescritura tiene que reflejar el repo, no el informe si
el repo hubiera cambiado.

---

## A · `ARCHITECTURE.md`

| # | Ref | Acción |
|---|---|---|
| A1 | R1 · §7 | Quitar **D27** («ingesta / `/search` / `retrieval.py` sin implementar»): las tres existen y funcionan. Reescribir §7 al estado real |
| A2 | D28 · §7 | Quitar **D28** («sin cache de respuestas»): no es deuda, es decisión consolidada y ya vive en §3. Aparecer en §7 la reencuadra como pendiente. Borrarla de §7, dejarla donde está en §3 |
| A3 | R2 · §3 vs §5 | §3 dice que la capa de datos está «fuera del flujo, ningún router la importa». Es falso y se contradice con §5, que es lo correcto (`/search` es el primer punto del camino que toca la BD). Alinear §3 con §5 |
| A4 | R4 · §5 + diagrama §1 | §5 dice que `corpus/` se conecta a `app/` «cuando `retrieval.py` lea `data/processed/`». Doble desfase: `retrieval.py` ya existe y **lee la BD, no `data/processed/`**; el puente real es `ingestion.py` (batch job). Corregir §5 y el diagrama de §1 (el acoplamiento `data/processed/*.json ◄─consume─ app/` ya no es el del camino de petición) |
| A5 | R3 | Árbol de `app/`: añadir `routers/search.py`, `services/retrieval.py`, `services/ingestion.py`, `services/embedding.py`, `schemas/search.py` |
| A6 | R5 | Árbol de `corpus/`: añadir `embed_all.py` y `diagnostics/embedding_sanity.py` |
| A7 | §2 tabla de módulos | M2 figura «en curso (S08 la capa de datos)». S08 está cerrada. Actualizar: capa de datos + retrieval vector/híbrido hechos; **S09 (pipeline RAG) es lo siguiente** |

Al reescribir §7, que quede como estado honesto de lo **realmente** abierto (todo gateado a futuro):
grounding/verificación de cita (S11), sesiones en memoria (M6), prompt caching (diferido),
recomposición de cláusula (M2), fan-out de citas (M2), 189 tablas, extractor de grafo sin validar.

---

## B · `.claude/CLAUDE.md`

### B.1 · Árboles y cifras de estado

| # | Ref | Acción |
|---|---|---|
| B1 | R9 | Árbol de `app/services/`: añadir `embedding.py` |
| B2 | R10 | Árbol de `scripts/`: añadir `measure_divergent_impact.py`, `sample_for_groundtruth.py`, `recall_groundtruth.py`, `score_groundtruth.py` |
| B3 | decisions §2 | «Estado actual» y toda cifra de chunks: **13.876 → 13.755** (121 marcadores de retirada filtrados en ingesta). Reflejar S08 cerrada, S09 siguiente |

### B.2 · Reformular items de la lista de pendientes

| # | Ref | Acción |
|---|---|---|
| B4 | R6 | Atribución de las 189 tablas: **no** es «120 en 70-41C, 49 en 70-31C». Medido: **187 en `ECSS-E-ST-70-41C`, 1 en `ECSS-E-HB-40-01A`, 1 en `ECSS-E-HB-40A`, 0 en `70-31C`**. El total 189 se mantiene; corregir el desglose |
| B5 | R7 · O6 | Quitar la coletilla de **D17** «pendiente de medir el lado query … es S09»: se midió en S08 brief 5 (`scripts/output_examples.txt`). Consolidar el hallazgo de criticidad (D17) con la parte medida de **D14** en **un** item. Preservar como línea abierta **solo** lo que sigue sin resolver de D14: la ponderación de la fusión RRF, que es **S10** |
| B6 | R14 | Reformular **D29**: hoy dice «validar con ground truth etiquetado», como si no existiera nada. La verdad: el instrumento **ya está construido y cableado** (`scripts/*_groundtruth.py`, `data/groundtruth/`, RUNBOOK §8); **lo único que falta es el etiquetado humano** (1/50 precisión, 0 recall, gate en FAIL). Que el item lo diga así |

### B.3 · Mover de pendiente a decidido (C2 los resolvió)

| # | Ref | Acción |
|---|---|---|
| B7 | D13 · decisions §1 | **Sacar D13 de pendientes.** Documentar la resolución en decisiones consolidadas: el conjunto de documentos se deriva del **manifiesto** vía un accesor único en `corpus/manifest.py` (`expected_document_ids` / `document_json_paths`). Documentar `DERIVED_ARTEFACTS` como **la constante central donde se registra un artefacto derivado nuevo** de `processed/` — es el punto único de verdad para la señal inversa de `report.py` |

### B.4 · Reformular D15 y D25 con su nota de deuda (no se resuelven, se mitigan)

| # | Ref | Acción |
|---|---|---|
| B8 | D15 · decisions §5(a) | D15 deja de ser «decisión de chunking pendiente». Ahora: filtro barato aplicado **en ingesta** (121 marcadores descartados en carga; BD 13.755). El fix robusto —tratar los marcadores de borrado en `parser`/`chunking` para que no lleguen a ser chunks— **sigue pendiente**, con la consecuencia explícita: **dispara re-chunking + re-embedding**. No es «pendiente de limpiar», es «pendiente de una operación cara» |
| B9 | D25 · decisions §5(b) | Documentar el guardarraíl `WEB_CONCURRENCY` como **mitigación con alcance nombrado**: caza la concurrencia declarada por entorno (la vía de despliegue estándar, porque uvicorn lee esa misma variable), **no** `--workers N` en línea de comandos. El fix real (store de sesiones compartido, Redis/PG) sigue diferido a **M6**. **D25 sigue `VIGENTE`** — el guard mitiga, no resuelve |

### B.5 · Dos items de deuda nuevos

| # | Ref | Acción |
|---|---|---|
| B10 | decisions §2 | **`EXPECTED_RETRACTED` es un contador mantenido a mano.** Anclado a `corpus_version`, **detiene la ingesta** en cada cambio de corpus hasta que se actualice. Documentar con esa consecuencia: es el precio conocido del gate ruidoso, no un descuido. Revisar cuando el corpus deje de ser estático |
| B11 | decisions §6 | **Dos fuentes de «qué debería existir» en la misma reconciliación.** `verify_parser` y `audit_raw_corpus` reconcilian el lado esperado contra el **inventario**; `parse_all` y `report` contra el **manifiesto**. Misma clase de bug que D13. No muerde hoy (ambas listan los 26) y solo corre en máquinas que hayan scrapeado (`data/inventory/` en `.gitignore`). **Baja prioridad + pregunta de diseño abierta:** qué fuente es autoritativa para la cobertura de `verify_parser`. Documentar, no arreglar |

---

## C · `.env` (local)

| # | Ref | Acción |
|---|---|---|
| C1 | R15 | Quitar las tres variables muertas: `CACHE_TTL`, `REDIS_URL`, `LLM_PROVIDER`. No están en `.env.example` (que es el contrato commiteado y ya está limpio + con `WEB_CONCURRENCY`); están solo en el `.env` local y contradicen los docs. Es higiene local, no cambia el repo commiteado — pero deja de mentir |

---

## Salida de CC → chat

Docs editados + un resumen breve de qué cambió por sección (no hace falta log de decisiones: C1 no
toma decisiones nuevas, aplica las ya tomadas). **Sin commitear.** Señalar cualquier afirmación del
informe que, al ir a reescribirla, resultara ya no coincidir con el repo.
