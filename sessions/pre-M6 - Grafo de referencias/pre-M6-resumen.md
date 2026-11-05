# pre-M6 — Grafo de referencias · Resumen provisional

> Sesión propia del proyecto (no pertenece al máster). Puerta obligatoria antes de M6.
> Sin material externo: la revisión parte de los artefactos de S06 (`references.json`,
> `graph.graphml`, `corpus/graph.py`), del parser de `xref` de runtime y de los aprendizajes de
> S13–S14. Documento provisional, previo a implementar.

---

## 1. Qué problema resuelve el grafo

El retrieval semántico trae lo que **se parece** a la pregunta. En un corpus normativo, buena parte
de lo que una respuesta necesita no se parece a la pregunta: está **referido** por lo que sí se
recuperó (una cláusula remite a otro estándar, un requisito exige un DRD, una tabla de
aplicabilidad decide si el requisito aplica). El grafo convierte esas relaciones en un segundo
canal de recuperación, determinista y auditable.

Base medida: recall 48 % en S06, ~40 % de las pérdidas son referencias tabulares. En S14, 15 huecos
tipados `delegated`, varios hacia estándares **indexados** (M-ST-80C, E-ST-70-01C): el destino
estaba en el corpus y el sistema no lo trajo.

## 2. Dos familias de "GraphRAG"

| | Grafo extraído por LLM | Grafo estructural / de citas |
|---|---|---|
| Ejemplos | Microsoft GraphRAG, LightRAG | RAG legal, patentes, normas |
| Aristas | Entidades y relaciones extraídas por un LLM; resúmenes de comunidad | Ya existen en el documento: referencias, jerarquía, anexos, nombres |
| Pregunta típica | Global ("¿qué temas recorren el corpus?") | Local ("¿qué más obliga a leer esta cláusula?") |
| Coste de construcción | Alto (LLM sobre todo el corpus) | Bajo (parser determinista) |
| Trazabilidad | Introduce texto generado (resúmenes) | Cada arista apunta a texto literal |

**Decisión:** RAG-ECSS usa la familia estructural. La extraída por LLM queda contraindicada por el
principio de trazabilidad (resúmenes de comunidad = texto normativo no verificado en el prompt).

Otras técnicas de recorrido, como teoría no ejercitada: personalized PageRank sobre el grafo
(HippoRAG), recorrido guiado por LLM (*think-on-graph*), embeddings de nodo. Útiles cuando el grafo
es grande y denso; con cientos de aristas y saltos acotados no aportan sobre un recorrido acotado
explícito.

## 3. Tipos de arista en ECSS

1. **Textuales explícitas.** "in accordance with ECSS-M-ST-10", "see clause 5.3.1", "Table D-1".
   Hoy: parser de `xref` en runtime sobre el texto inyectado, un salto. `references.json` (S06)
   tiene 329 aristas: 166 `normative_ref`, 98 `intra_doc_ref`, 52 `definition_ref`,
   13 `informative_ref`; 170 con cláusula destino, el resto apuntan a un documento entero.
2. **Estructurales implícitas.** No están escritas como referencia, se deducen de la estructura:
   cláusula padre/hermana, fila de la matriz de aplicabilidad de la cláusula, requisito ↔ DRD
   ("Expected output" del requisito → anexo DRD).
3. **Por nombre.** El texto nombra un artefacto sin citar su estándar: "SCF" → M-ST-40C anexo E.
   Requiere un diccionario nombre → ancla.

Además, **autoridad**: ante el mismo contenido, la fuente normativa prevalece sobre el handbook. No
es una arista; es una regla de precedencia del acumulador.

## 4. Piezas del diseño

- **Almacén.** Índice de adyacencia en memoria, derivado de artefactos versionados
  (`corpus_version`). Sin base de datos de grafos. El `nx.DiGraph` sigue fuera del camino de
  petición (herramienta de análisis).
- **Acumulador por `clause_id`.** Todo canal (retrieval, expansión, `xref`, aristas nuevas) entrega
  candidatos a un único acumulador que deduplica por `clause_id`, ordena de forma estable (nunca por
  orden de llegada: el orden fija la posición en el prompt) y resuelve qué `origin` gana cuando una
  cláusula llega por dos caminos.
- **`origin` en el contrato.** Cada evidencia declara por qué entró (retrieval, expansión, tipo de
  arista) y desde dónde (trail).
- **Presupuesto de saltos y trail.** Profundidad máxima explícita; cada evidencia añadida por grafo
  lleva el camino que la trajo.
- **Prioridad entre punteros.** Explícita (tipo de arista, autoridad, relevancia), no el orden de
  aparición en el texto (aprendizaje 9 de S13).
- **Puerta de relevancia.** Seguir una referencia no garantiza traer algo relevante: un retrieval
  filtrado sin umbral siempre devuelve algo (aprendizaje 8 de S13). Escalera: umbral de distancia →
  cross-encoder → juez LLM, este último solo donde haya ambigüedad real y evaluado offline contra el
  golden set.

## 5. Punteros a documento entero

Un puntero sin cláusula destino **no trae el documento entero**. Se sigue como retrieval acotado al
documento destino con la consulta, detrás de la puerta de relevancia y con plaza según la prioridad
explícita.

Antecedente: es el concepto de `clause_standard` (S13), implementado y no adoptado. Falló la guarda 1
(resolver aspectos objetivo) por la **política de plazas** (orden de aparición, plazas compartidas)
y la falta de puerta, no por el concepto: recall, precisión y `verified` mejoraban o se mantenían.
Se reabre dentro del acumulador nuevo, con criterio preregistrado nuevo; no se relee el de S13.

## 6. `delegated` hacia el corpus

Un hueco `delegated` cuyo destino está indexado es señal de hueco de retrieval. Resolver el destino
contra el corpus es determinista con el parser de `xref` (marca `in_corpus`). Dos usos posibles:

- **Métrica** del arnés: `delegated` con `in_corpus=true` mide lo que el grafo debería haber traído
  antes de generar.
- **Disparador en runtime**: generar → detectar → traer → regenerar. Duplica el coste del turno.

## 7. Riesgos

- **Explosión multi-salto:** cada salto sube el recall y baja la precisión del contexto. El
  presupuesto de saltos es el parámetro principal.
- **Inundación estructural:** padre/hermanas y matrices de aplicabilidad crecen rápido; necesitan
  tope propio.
- **Evaluación:** sin métricas preregistradas, cualquier tipo de arista "parece ayudar". Casos con
  señal: q14-a1 (OBCP → E-ST-70-01, 2 de 5), q12 (5.3.1 → M-ST-80).

## 8. Estado de las decisiones

**Decidido**
- Familia estructural; sin extracción por LLM ni resúmenes de comunidad.
- Índice en memoria desde artefactos versionados; sin base de grafos.
- Puntero a documento = retrieval acotado al destino con puerta, nunca el documento entero.
- Los tres tipos de arista y la preferencia de autoridad se resuelven en esta sesión, un brief por
  funcionalidad, una variable cada vez.

**Abierto**
- Orden y granularidad de los briefs.
- Tipo de puerta de relevancia de partida (umbral de distancia como primer peldaño).
- `delegated` + `in_corpus`: métrica o disparador.
- Profundidad máxima (1 o 2 saltos) por tipo de arista.
- Dónde se construyen las aristas (2) y (3) sin tocar `corpus/` (congelado).
