# S10 — Brief 1: golden set de retrieval y arnés de medición

> Funcionalidad única: una verdad de referencia versionada para el retrieval y un arnés que mide
> cualquier configuración contra ella. **No cambia el pipeline de producción.**
> Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md` antes de empezar: este brief asume su contexto.
> Documento hermano: `S10-protocolo-anotacion.md` (lo ejecuta **otra** sesión, no tú).

---

## 0. Objetivo y alcance

**Objetivo.** Que cada decisión de retrieval pendiente (filtro por estándar nombrado, ponderación
de la fusión, BM25, reranker, expansión de cláusula) se tome contra una tabla — calidad por
estrato + latencia — y no contra 5 arquetipos leídos a ojo. Primera tabla: **vector vs híbrida
actual**.

**Cómo se reparte el trabajo.**
- **Tú (esta sesión):** export de cláusulas para anotar, importador/validador del golden set,
  métricas, arnés re-ejecutable, export/import incremental, primera medición, verificaciones baratas.
- **Sesión de anotación separada** (Claude Code, nueva, sin contexto de esta): genera las
  consultas y anota los relevantes siguiendo `S10-protocolo-anotacion.md`. Lee solo el export de
  cláusulas; no usa la API ni ve resultados de ninguna configuración. **No la simules ni la
  sustituyas**: si te falta su salida, te detienes y lo dices.

**Fuera de alcance (no implementar):** cualquier cambio en el pipeline de petición (filtros,
pesos, BM25, reranker, expansión, `RETRIEVAL_MODE`, `RETRIEVAL_TOP_K`), juez LLM por API,
evaluación de la generación (S11), CI. Si algo parece exigirlo: `# TODO: pendiente de decisión en
Claude Chat` y seguir.

---

## 1. Decisiones que este brief implementa (y su porqué)

| # | Decisión | Porqué |
|---|---|---|
| D1 | **Unidad de relevancia = cláusula** `(canonical_doc_id, section_number)`, nunca `chunk_id` | Es la unidad de contexto y de cita; sobrevive al re-chunk pendiente |
| D2 | **Relevancia binaria**, criterio literal en el golden set: *"A clause is relevant if an ECSS engineer would use it to answer the question, whether normative or informative. Authority is metadata, not relevance."* | Lo informativo útil cuenta (decisión de producto). La autoridad ya es metadato: se cruza al medir, no se anota |
| D3 | **Exactamente 10 consultas**, congeladas, estratificadas: nombra estándar (2), genérica/conceptual (2), paráfrasis (1), comparativa (2), enumerativa/cross-estándar (2), multi-turn con cambio de tema (1, como secuencia de turnos; se juzga el último) | Número fijo = puntuación comparable siempre. Con 1–2 consultas por estrato, el estrato es **indicativo**, no concluyente |
| D4 | **Anotación externa e independiente del retriever** (sesión separada, suscripción, sin coste de API): busca relevantes en el corpus completo, no en un pool de nuestras configuraciones | Elimina el sesgo del pooling: el conjunto de relevantes no depende de lo que nuestro sistema ya encuentra |
| D5 | **Anotación incremental por protocolo**: cláusulas del top-k de cualquier configuración sin juicio → el arnés las exporta → la sesión de anotación las juzga con el mismo protocolo → se importan. Nunca se sobrescribe un juicio en silencio | Sin esto, lo no juzgado cuenta como irrelevante y penaliza a la configuración que encuentra algo nuevo |
| D6 | **Evidencia literal obligatoria** en cada juicio positivo, **validada por código** contra el texto de la cláusula con la normalización del validador de citas existente | El usuario no domina ECSS: su revisión se limita a lo que el validador marque. Ataja la alucinación de cláusulas y citas |
| D7 | **Repetible por protocolo, no por modelo**: el protocolo vive versionado en el repo; cada juicio registra modelo declarado, fecha y versión de protocolo; el golden set registra criterio e `input_fingerprint` | Mismo grado de repetibilidad que un anotador humano, con la vara escrita |
| D8 | Se mide **lo que llega al LLM**: el camino del proveedor de contexto de producción (query con historial, agrupación por cláusula, dedup), no `/search` crudo | p@k sobre chunks infla por duplicados y mide algo que el generador no recibe |
| D9 | **Ubicación del arnés, del protocolo y del golden set: la decide CC** con la frontera de `CLAUDE.md`. Restricciones: golden set y protocolo **versionados** (fuera de `data/`); arnés **re-ejecutable** con cada técnica | Delegado por el usuario; el porqué va a `S10-decisions.md` |

---

## 2. Métricas (por consulta, por estrato y media)

Sea *k* = número de cláusulas que el pipeline entrega al LLM con la configuración medida.

- **precision@k** — fracción de las k cláusulas entregadas que son relevantes.
- **recall@k** — de las relevantes conocidas, cuántas llegan en las k.
- **recall@k normativo** — mismo cálculo restringido a relevantes normativas (detecta lo
  normativo desplazado por handbooks).
- **recall@candidatos** — de las relevantes conocidas, cuántas aparecen entre los candidatos
  amplios (top-50 chunks → cláusulas). Techo: alto aquí + bajo p@k = margen para reranker; bajo
  aquí = problema de recall.
- **sin juzgar@k** — cláusulas del top-k sin juicio. Si > 0 la medición se marca **inválida** y el
  arnés exporta la lista a juzgar (D5).
- **Latencia de retrieval** — en caliente (se descarta la primera ejecución), mediana de 5,
  embedding de la query incluido.

Salida: tabla estrato × configuración + detalle por consulta. La media se reporta; **la tabla por
estrato es el resultado**.

---

## 3. Trabajo

1. **Export de cláusulas para anotar.** Un fichero (o conjunto) legible y buscable con grep, una
   entrada por cláusula: `canonical_doc_id`, `section_number`, título si existe, autoridad y el
   texto completo de la cláusula reconstruido a partir de sus chunks, en orden. Más un índice de
   los 26 documentos (ID + título) **sin contenido**, para la fase 1 del protocolo. Debe
   regenerarse de forma determinista desde `data/processed/` y registrar el `input_fingerprint`.
   Señala en el export las cláusulas con `section_number` repetido dentro de un documento
   (problema de identidad abierto): el anotador las verá y el importador las tratará con cuidado.
2. **Importador/validador del golden set.** Rechaza (con mensaje claro) si: el número de
   consultas ≠ 10; falta criterio, versión de protocolo o fingerprint; un estrato no cuadra con
   D3; una cláusula no existe en el export; una cita no aparece literal (normalizada) en el texto
   de su cláusula; hay duplicados. Lo que no es rechazo duro pero requiere ojo humano (cita
   encontrada solo tras normalización agresiva, cláusula con número repetido) va a una **lista de
   revisión**.
3. **Métricas y arnés** (§2, D8). Configuraciones de partida: `vector` e `hybrid` tal como están.
   El arnés debe aceptar configuraciones futuras sin reescribirse.
4. **Export/import incremental** (D5): genera el fichero de pares (consulta, cláusula) a juzgar en
   el formato del modo B del protocolo, y lo importa con las mismas validaciones.
5. **Tests de contrato** (§4).
6. **Verificaciones baratas** (informan el brief 2; no cambian código de producción). Ejecútalas
   ya, no dependen de la anotación:
   - **Identificador como contenedor:** para queries que nombran un estándar (usa las 2 de los
     arquetipos actuales si el golden set aún no existe), ¿las cláusulas que trae `hybrid` son
     *del* estándar nombrado o *citan* ese estándar desde otro? Contarlo.
   - **Tokenización de IDs:** `ts_debug('english', …)` sobre `ECSS-E-ST-40C`, `E-ST-40C`,
     `Q-ST-80C Rev.2`, `5.3.5.1`. ¿Qué lexemas salen? ¿`e`/`st` casan medio corpus?
   - **Hipótesis IDF:** arquetipo genérico 1, top-10 léxicos con la frecuencia de cada término de la
     query en el chunk y su frecuencia documental. ¿Gana quien repite términos comunes?
   - **Volumen y versiones:** chunks por documento y si hay más de una revisión del mismo estándar.
7. **Pausa.** Con 1–6 hechos y la suite en verde, te detienes y reportas. El usuario ejecuta la
   sesión de anotación. Cuando el golden set exista: importar, resolver la lista de revisión con el
   usuario, medir `vector` vs `hybrid`, y si hay `sin juzgar@k > 0`, ciclo incremental antes de dar
   la tabla por válida.

---

## 4. Tests de contrato (sin red, sin BD)

Como mínimo:
- Métricas sobre casos construidos: cero relevantes, k mayor que lo devuelto, cláusula repetida vía
  varios chunks, recall normativo, sin juzgar@k → medición inválida.
- Importador: cada rechazo de §3.2 y cada caso de la lista de revisión.
- Validación de cita literal: reutiliza la normalización existente y sus casos límite.
- Incremental: exporta solo lo no juzgado; nunca sobrescribe un juicio; el import respeta D6.
- Export de cláusulas: determinista (mismo input → mismo output), orden de chunks dentro de la
  cláusula, marcado de números repetidos.
- Ningún veredicto depende de variables de entorno opcionales.

La ejecución real (BD) es **evaluación**, no contrato: fuera de la suite por defecto.

---

## 5. UAT (usuario) — y por qué es suya

Caso 1 de la frontera (no hay oráculo) y caso 2 (decisión del usuario):
1. Ejecutar la sesión de anotación con el protocolo (fase 1 → aprobar las 10 consultas → fase 2).
2. Resolver la lista de revisión del importador (lectura: ¿la cita existe y responde?).
3. Leer la tabla vector vs híbrida por estrato.

CC deja en el script de UAT el comando exacto para lanzar la sesión de anotación con el protocolo y
el export, y qué mirar en cada paso.

---

## 6. Cierre

- `S10-decisions.md`: cada decisión + porqué, etiquetada **agnóstico/dominio**. Incluye: ubicación
  elegida (D9) y razonamiento, tamaño del export, resultados de las cuatro verificaciones de §3.6 y
  — cuando exista — la tabla vector vs híbrida por estrato, el tamaño de la lista de revisión y,
  si se hizo, el acuerdo entre sesiones de anotación.
- `CHANGELOG.md`, `CLAUDE.md` y `ARCHITECTURE.md` según sus reglas (presente, no diario).
- `uv run pytest` en verde y mensaje de commit propuesto. El commit lo hace el usuario.
