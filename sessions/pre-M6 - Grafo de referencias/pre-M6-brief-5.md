# pre-M6 — Brief 5: aristas por nombre (DRD nombrados sin cita)

> Rama `pre-m6-grafo`. Una variable: **resolver nombres de DRD** que el texto menciona sin citar su
> estándar ("software configuration file", "SCF", "technical budget") y traer ese anexo. **Una
> pasada real autorizada: ≤ 1,00 USD**, solo si la comprobación sin generación da R ≥ 3.
>
> Brief 4 cerrado con «no se adopta» (aceptado por el chat; sin UAT, correcto). Su lección guía este
> brief: **el problema es de selección, no de alcance**. Por eso aquí el destino es un anexo
> concreto, no un documento entero, y se reutiliza la puerta vigente sin abrir más profundidad.

---

## 0. Contexto

L2: 3 positivos `by_name` `pointed_not_resolved`: q24-g1 y q24-g2 (E-ST-40C §5.5.3.1 nombra el SCF →
M-ST-40C §E.2.1 y §E.1.2) y q25-g1 (E-ST-40C §5.3.8.1 nombra los technical budgets → E-ST-10C
§I.1.2). Negativo del tipo: q29-g1 (nombre ambiguo, "acceptance" → E-ST-10-03C), hoy `not_followed`.

## 1. Decisiones

| # | Decisión | Porqué |
|---|---|---|
| B1 | **Diccionario nombre → anexo DRD**, derivado y versionado (con `input_fingerprint`), construido de forma determinista desde los **títulos de anexo DRD** del corpus ya parseado ("Annex E (normative) Software configuration file (SCF) – DRD"): nombre completo y, si el título lo trae entre paréntesis, su acrónimo. Se construye fuera de `corpus/` (en `app/` o como artefacto derivado en `data/processed/`) | Sin LLM, sin lista a mano, sin tocar `corpus/` |
| B2 | **Detección** en el texto `hop = 0` (como P2): nombre completo case-insensitive con límites de palabra; acrónimo solo en mayúsculas exactas y como palabra. Formas plurales simples ("technical budgets") cuentan | Acrónimos de 3 letras en minúscula son ruido |
| B3 | **No se sigue:** nombre que resuelve a más de un anexo (`ambiguous`); anexo del mismo documento que la cláusula de origen cuando ya está en contexto (`already_injected`); términos genéricos fuera del diccionario (q29 "acceptance" no es título de DRD) | q29 debe seguir `not_followed`; sin heurísticas nuevas |
| B4 | **Qué se trae:** el mecanismo de punteros a documento (brief 3) con el **ámbito reducido al subárbol del anexo** (cláusulas `E`, `E.x`, `E.x.y`): misma consulta, misma puerta **0,3758**, plazas propias `RETRIEVAL_NAME_MAX_NAMES = 3`, `..._MAX_CHUNKS_PER_NAME = 3`, `..._TOTAL_TOKENS = 2000` | Una sola maquinaria de seguimiento; la puerta y las plazas ya medidas |
| B5 | Canal `origin = graph_name`, `edge_kind = by_name`, `hop = 1`, precedencia **5** (tras `graph_document`); `from_clause` y el nombre detectado (`mention`) en el contrato. Degradación `name_over_limit`, no pintada | Trazabilidad del camino |
| B6 | Flag `RETRIEVAL_NAMES: off \| on`, `off` por defecto hasta medir | Una variable |

## 2. Guardas (v2, preregistradas)

`control` = defaults actuales; `variant` = `RETRIEVAL_NAMES=on`. R = positivos `by_name` ganados
frente a control en L2. **Techo: R ≤ 3, así que N = 3: los tres ganados deben resolverse en ≥ 2/3.**
Si R < 3, no se paga: porqué no por aspecto y parada. Guarda 1b sobre los ya resueltos por control
(todos los tipos), 2–4 como siempre, **5 sobre los cuatro negativos** (q10-a1, q27-g1, q28-g1,
q29-g1). Riesgo declarado: 2 de los 3 son de q24.

## 3. Comprobación sin generación

1. Diccionario: tamaño, entradas ambiguas, y los nombres detectados en el contexto de las 29
   consultas (cuántos por consulta, cuántos seguidos). Si una consulta dispara > 3 nombres, reportar
   cuáles.
2. L2 de la variante, R y N.
3. Digest de `defaults` igual al del brief 3 (la variante está apagada).
4. Tamaño de contexto control frente a variante.

## 4. Se reporta (no decide)

Latencia, coste, tokens, `not_found`, `delegated` por clase, citas fuera de su afirmación (con y sin
cambio de contexto), cláusulas solo del grafo citadas.

## 5. UAT y cierre

Si se adopta: por API q24 y q25 (por evidencia `graph_name`: `from_clause` contiene la mención, el
destino es el anexo del diccionario, distancia ≤ umbral) y captura del render de q24. Si no se
adopta: sin UAT. `pre-M6-decisions-5.md`, `pre-M6-uat-5.md` si aplica, docs y commit. Si pasan las
guardas, `RETRIEVAL_NAMES=on` pasa a default.
