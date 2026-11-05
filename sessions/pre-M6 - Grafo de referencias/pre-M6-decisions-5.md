# pre-M6 — Decisiones del brief 5: aristas por nombre (DRD nombrados sin cita)

> **Parada sin pagar: R = 1.** De los tres positivos `by_name`, la variante solo gana **q24-g2**. Los
> otros dos los para la puerta, no la detección. `RETRIEVAL_NAMES` se queda en `off`, y el canal
> queda en el código como instrumento. Sin pasada real (coste: solo embeddings de consulta, ~0 USD)
> y sin UAT.

---

## 1. Hecho

- **Diccionario (B1)** en `app/services/drd_names.py`. Se construye en el proveedor la primera vez,
  desde los **títulos de anexo** del corpus cargado: el primer tramo de `Section:` en la cabecera
  de cada chunk de anexo (`load_annex_titles` en `retrieval.py`; sale de la base solo el título,
  nunca `embed_text`). Lleva el `input_fingerprint` de los chunks (`ab3fce8b5cb1a0d4`) y lo registra
  `drd_names_loaded`. No es un artefacto aparte ni toca `corpus/`. Entra un título que acaba en
  `DRD`; da el nombre sin paréntesis y cada acrónimo que trae entre paréntesis (`PUM or UM` da
  dos). **82 nombres y 41 acrónimos** de 83 anexos DRD. **Ambiguos: `SMP` y "software maintenance
  plan"** (E-HB-40A §C y E-ST-40C §T). "Guideline content of Analysis Report" (E-ST-10C §S) no
  entra.
- **Detección (B2):** el nombre completo sin distinguir mayúsculas, con `s` de plural; el acrónimo
  solo tal como se escribe, mayúsculas incluidas; los dos como palabra entera. **Añadido que el
  brief no fijaba:** si dos menciones se solapan, gana la más larga. Sin esto, "software
  configuration file" dispararía también "configuration file" (E-ST-40-07C §C).
- **No se sigue (B3):** `ambiguous`, y `already_injected` para un anexo del documento de la mención
  que ya tiene un chunk en contexto (así cae también la mención dentro de su propio anexo).
- **Seguimiento (B4):** la maquinaria de `doc_pointers.follow`, generalizada. `Pointer` lleva
  `annex` y `mention`, y sus hits se archivan por `target` (`<doc>` o `<doc> §<anexo>`). El
  ranking usa la misma consulta y el mismo vector, `section_prefix` (cláusula y subcláusulas) en
  `rank`, y los candidatos y la puerta (0,3758) de los punteros a documento. Plazas propias:
  `RETRIEVAL_NAME_MAX_NAMES=3`, `_MAX_CHUNKS_PER_NAME=3`, `_TOTAL_TOKENS=2000`.
- **Marcado (B5):** `origin="graph_name"`, `edge_kind="by_name"`, `hop = 1`, precedencia 5;
  `from_clause` y `mention` en `EvidenceChunk`; traza `xref["names"]`; degradación
  `name_over_limit` (etapa `names`), sin pintar. El esquema de `QueryResponse` cambia por el campo
  y los valores nuevos, y su hash fijado se actualiza.
- **Flag (B6):** `RETRIEVAL_NAMES=off` por defecto. Con `off`, contexto y traza son byte a byte
  los de antes.
- Instrumento: `graph_reach run --names`, `doc_pointer_check names` (§3.1) y `size --names`, la
  configuración `doc_pointers_names` del arnés, `graph_name` contado como canal del grafo en
  `graph_report` y como añadido en `aspects` y la UI, y la configuración `names` en
  `context_digest`.
- Suite: **1684 verde**.

## 2. Comprobación sin generación

### 2.1 Nombres en el contexto (`drd_names_20261105-160122.md`)

33 turnos: **30 escriben algún nombre** y **8 siguen alguno**. **16 turnos disparan más de 3
nombres**:

| turno | nombres | de ellos seguidos |
|---|---|---|
| q01 | 17 | 3 (SDP, SRelD, SDD → anexos de E-ST-40C) |
| q26 | 14 | 0 |
| q19 | 13 | 3 (DJF → E-ST-10C §K; SDP y SVS → E-ST-40C) |
| q15 | 12 | 0 |
| q11, q24 | 11 | 3 y 1 |
| q17 | 9 | 0 |
| q25 | 8 | 0 |
| q20 | 7 | 0 |
| q05 | 6 | 0 |
| q12, q18, q29 | 5 | 1, 0 y 1 |
| q16, q22, q27 | 4 | 1, 0 y 1 |

El resto de las menciones se queda en `gated` (la mayoría) o en `already_injected` (de q18 a
q24, decenas de menciones del propio documento ya en contexto). La puerta es la que filtra: los
nombres genéricos ("test report", "TS", "DJF") disparan a menudo y casi nunca pasan.

### 2.2 L2 (`reach_20261105-160146` control, `reach_20261105-160203` variante)

| aspecto | control | variante | porqué no |
|---|---|---|---|
| **q24-g2** (M-ST-40C §E.1.2) | `pointed_not_resolved` | **`reached`** | — (E.1.2 es el rango 1 del anexo, a 0,3664) |
| q24-g1 (M-ST-40C §E.2.1) | `pointed_not_resolved` | `pointed_not_resolved` | El nombre se sigue, pero el chunk de evidencia (E.2.1::r9) es el **rango 9 de 19** del anexo, a **0,4100**. Lo paran la puerta (0,3758) y el número de candidatos (5) |
| q25-g1 (E-ST-10C §I.1.2) | `pointed_not_resolved` | `pointed_not_resolved` | "Technical budget" se detecta y queda `gated`: la evidencia es el **rango 1** del anexo, pero a **0,3971** |

**R = 1 < 3: no se paga** (§2 del brief).

Contrafactual, sin ejecutarlo: con la puerta del brief 4 (0,3991) entraría q25-g1 (0,3971) y no
q24-g1 (0,4100). Ni así se llega a R = 3. Para eso haría falta abrir la puerta hasta 0,41 y leer 9
candidatos del anexo, que es justo la profundidad que el brief 4 no adoptó.

**Fuera del tipo (se reporta, no cuenta):** **q16-g2** (`explicit_document`, Q-ST-10C §A.2.1) pasa
de `pointed_not_followed` a `reached`. Q-ST-80C §5.2.1.1 nombra el "product assurance plan" en texto
con `hop = 0`, y entran A.2.1 r2, r9 y r3. Es el caso de q16 que el brief 4 dejó por K5 (el
puntero a Q-ST-10 llega en lo que trae un salto): por nombre se alcanza sin un segundo salto.
Contando también este, los ganados serían 2.

**Negativos:** sin cambios. q29-g1 sigue `not_followed`: "acceptance" no está en el diccionario, y
en q29 se sigue SSS → E-ST-40C §B, que no es su destino. q10-a1 sigue `followed`, como en control,
por su puntero a documento. q27-g1 y q28-g1 siguen igual.

### 2.3 Digest y tamaño

- `defaults`, igual al servido en el brief 3 (`b3_final` `clause_standard`) y al del brief 4 en las
  seis configuraciones: **33 de 33**. Con `names` cambian los 33 digests: la traza gana la clave
  `names` aunque no entre nada.
- Tamaño (33 turnos):

  | | chunks mediana / p95 | tokens mediana / p95 |
  |---|---|---|
  | control (`defaults`) | 36 / 69 | 5.290 / 7.176 |
  | variante (`names=on`) | 37 / 77 | 5.290 / 7.514 |

## 3. Decisión

**Parada** (§2: «si R < 3, no se paga: porqué no por aspecto y parada»). `RETRIEVAL_NAMES=off`
sigue por defecto, y el canal se queda en el código como instrumento, igual que `deep` en el brief 4.
Sin UAT (§5: solo si se adopta).

## 4. Para el chat (sin implementar)

- **La detección funciona y la puerta decide.** El diccionario sin LLM encuentra los tres nombres
  de los positivos, sin falsos ambiguos, y la puerta deja fuera a dos de los tres. Los nombres
  genéricos disparan mucho (16 turnos con más de 3) y la puerta los para casi todos. El coste de
  tenerlo encendido es pequeño: +1 chunk de mediana y +5 % de tokens en p95.
- **Es la misma frontera que en el brief 4:** el destino correcto está cerca, pero no lo bastante
  para una puerta absoluta calibrada sobre documentos enteros. Un anexo es un destino más estrecho
  y más fiable que un documento. Si se reabre, la pregunta es si los nombres merecen **su propia
  puerta**, calibrada sobre pares de anexo, con la misma regla *in-sample* y pares del modo B. No es
  subir la general.
- **q16-g2 por nombre** es la primera señal de que un nombre puede sustituir a un segundo salto. Con
  un solo caso no decide nada.
- **Nombres del propio documento:** en q01 y q19 se siguen anexos de E-ST-40C (SDP, SDD, SVS) desde
  E-ST-40C. B3 lo permite si el anexo no está en contexto. No se ha medido si eso es ruido. Si se
  reabre, conviene mirarlo con los pares.
