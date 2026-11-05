# pre-M6 — Grafo de referencias · Consolidado

> Sesión propia del proyecto, puerta antes de M6. Cinco briefs más uno de proceso (brief 0, modo
> autónomo). Rama `pre-m6-grafo`, pendiente de merge a `main`. Coste real total ≈ 1,07 USD (dos
> pasadas: briefs 3 y 4) más embeddings. Suite: 1.684 en verde.
>
> **Resultado:** la puerta queda resuelta. Punteros a documento **adoptados** con puerta de
> relevancia; aristas estructurales **sin señal** (DRD y jerarquía ya llegan por retrieval);
> aristas por nombre **implementadas y apagadas** (R = 1); más profundidad en el destino **no
> adoptada**. El resumen (`pre-M6-resumen.md`) se conserva: la divergencia es material de Guide.

---

## 1. Qué se construyó, brief a brief

| Brief | Qué | Resultado |
|---|---|---|
| 0 | Modo autónomo de CC (Docker, una pasada, UAT con capturas, commit por brief, paradas) | Aplicado |
| 1 | Instrumento: golden set v3 (protocolo 1.2, 15 consultas `graph_*` con camino), mapa de alcanzabilidad sin generar (L2), métrica `delegated` → corpus, guardas preregistradas | 40 aspectos, VALID |
| 2 | Acumulador único por `ClauseKey`, orden declarativo, precedencia de `origin`, trail (`origin`, `edge_kind`, `from_clause`, `hop`) en el contrato, `MAX_HOPS = 1` | Refactor puro: 165 digests idénticos |
| 3 | Punteros a documento: retrieval acotado al destino, puerta por distancia calibrada (0,3758), prioridad explícita (autoridad normativa → distancia), plazas propias | **Adoptado**: +2 aspectos, sin pérdida medible |
| 4 | Profundidad en el destino (10 candidatos; 5 y luego 10 chunks) | **No adoptado**: G1 1/3, G5 falla |
| 5 | Aristas por nombre de DRD (diccionario derivado de títulos de anexo, sin LLM) | **Parada sin pagar**: R = 1; queda apagado |

## 2. Lo que cambió respecto al resumen

| Resumen (antes de implementar) | Implementación |
|---|---|
| Tres tipos de arista a construir | Solo uno tenía techo: los **estructurales** (DRD, jerarquía) ya llegaban por retrieval en 6 de 6; aplicabilidad, 2 fallos con n = 3. El mapa de alcanzabilidad lo dijo antes de escribir código |
| Cinco briefs de técnica previstos | Tres de técnica; el mapa reordenó y podó |
| `delegated` → `in_corpus` como métrica | Confirmado como métrica de arnés; útil para separar fallo de retrieval (`in_corpus_not_injected`) de fallo de generación (`in_corpus_injected`) |
| Un salto por defecto, el segundo si hay casos | Los 40 aspectos son de 1 salto. Señal nueva: una arista por nombre puede **sustituir** un segundo salto (q16-g2) |
| La puerta de relevancia como requisito | Imprescindible y, a la vez, el **límite** de la sesión: separa mal destino útil de destino de cortesía cuando todo el documento está cerca de la consulta |

## 3. Aprendizajes (agnóstico / dominio)

1. **Instrumento antes que técnica, y el instrumento decide el orden.** *(agnóstico)* Un mapa de
   alcanzabilidad por tipo de arista, sin generar (¿llega la respuesta al contexto? si no, ¿por qué:
   no recuperada, origen no llega, arista no detectada, detectada y no seguida, sin plaza?), podó
   dos de cinco briefs antes de escribir código.
2. **Un golden set para grafo necesita el camino, no solo la relevancia.** *(agnóstico)* Cada
   aspecto lleva origen, destino, tipo de arista, saltos y **dos** citas literales (respuesta y
   enlace). Las negativas también: una de cortesía, una fuera del corpus, una de nombre ambiguo.
   Negativas cuyo enlace nunca llega al contexto no prueban nada (2 de 4 aquí).
3. **Preregistrar lo ganado frente a control, no lo alcanzado.** *(agnóstico)* El techo contado como
   "lo que alcanza la variante" incluyó un aspecto que control ya resolvía y permitió pagar una
   pasada con menos margen del previsto. Corregido (guardas v2) y añadida una guarda que protege lo
   que control ya resuelve.
4. **Un refactor de canales se valida por digest del contexto.** *(agnóstico)* Antes de añadir
   canales, un acumulador único con orden declarativo; aceptación = contexto byte a byte idéntico
   en todas las configuraciones (165 digests). Hace medibles los briefs siguientes.
5. **Seguir un puntero a documento funciona con puerta y plazas propias.** *(agnóstico)* El mismo
   concepto que S13 no adoptó pasó al darle (a) umbral de distancia calibrado con la regla "lo
   admitido no puede ser menos preciso que lo que ya se sirve", (b) prioridad explícita y (c)
   plazas no compartidas. Ganancia modesta (2 de 11), sin coste medible.
6. **Más profundidad en el destino no es la palanca: el problema pasa a ser selección.**
   *(agnóstico)* Con más chunks por puntero, lo relevante entra y el generador no lo usa (10
   cláusulas del mismo documento compiten) y lo de cortesía entra y sí lo usa. La primera
   configuración ni siquiera midió lo que quería: subir candidatos sin subir el tope de chunks deja
   el tope vinculante intacto. Revisar qué tope es el vinculante antes de abrir otro.
7. **Una puerta absoluta calibrada sobre documentos no sirve igual para destinos estrechos.**
   *(agnóstico)* Por nombre, la detección acierta los tres casos y la puerta para dos. Un anexo es un
   destino más fiable que un documento entero; si se reabre, con **su propia** puerta.
8. **Calibrar in-sample y decirlo.** *(agnóstico)* El umbral se ajustó sobre las mismas 29 consultas
   con que se mide; la curva no es monótona y el umbral queda en el borde. Además, los embeddings de
   consulta varían en la cuarta cifra entre ejecuciones: un umbral está definido a ±0,0001.
9. **Anotación ciega delegada a un subagente.** *(proceso)* El chat ejecuta la anotación con un
   subagente nuevo que solo ve el workspace: 15 consultas + 4 rondas de modo B (229, 65, 81 pares)
   sin intervención de Salva. Riesgo residual declarado: no se puede garantizar que el subagente no
   reciba el contexto general del proyecto; la validación de citas es por código.
10. **Diccionario de nombres sin LLM desde títulos de anexo.** *(dominio)* 82 nombres y 41
    acrónimos de 83 anexos DRD; dos ambiguos (SMP). Solapes: gana la mención más larga.
11. **Diferencias de autoridad reales.** *(dominio)* E-ST-40C R.2 y Q-ST-80C D.2 discrepan sobre
    pruebas unitarias en categoría D; es un caso de prueba del producto, no un error del golden set.

## 4. Estado y defaults

`RETRIEVAL_XREF=clause_standard` (punteros a documento con puerta 0,3758, 5 candidatos, 3 chunks,
2.000 tokens, 3 punteros), `RETRIEVAL_GRAPH_MAX_HOPS=1`, `RETRIEVAL_NAMES=off`. Variante `deep`
(brief 4) y canal de nombres (brief 5) quedan como instrumento. Contrato con `origin`,
`edge_kind`, `from_clause`, `hop`, `origins`, `mention`, `distance` (solo `graph_document`).

## 5. Diferidos

- **Selección del contexto añadido:** puerta relativa al documento o segundo peldaño (cross-encoder
  sobre lo añadido); tratamiento propio de punteros de cortesía (q10-a1). Casos: q14-a1 (rango 2 a
  0,46), q24-g1, q25-g1.
- **Uso del contexto por el generador:** q12-a1 y q12-a4 (traído y no citado). Junto con la deriva de
  citas fuera de su afirmación (varianza del generador: 25 de 33 en contextos idénticos).
- **Puerta propia para nombres** con pares de anexo; nombres del propio documento (¿ruido?); q16-g2
  como sustituto de un segundo salto.
- **Aplicabilidad:** 2 fallos (tope y enlace no traído), n = 3.
- **Q-ST-80C 6.2.3.1** ausente del export (¿hueco de parse o no citable?).
- **UI:** distinguir remisión a documento de remisión a cláusula en la traza (mostrar `from_clause`).
- **Para M6:** rol de solo lectura en la base, quitar el superusuario, capas de reintento; 3 tests
  que fallan sin `.env`.
