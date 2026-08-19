# S06 — Brief 5-pre: Diagnóstico de tablas del corpus

> **Tipo:** diagnóstico / instrumento de medición. **No** extrae tablas, **no** toca el parser de
> producción. Precede al Brief 5 (extracción de tablas) porque la naturaleza de las tablas decide el
> mecanismo de extracción, y hoy no la conocemos.
>
> **Por qué diagnóstico primero (patrón de la sesión):** cada brief estructural (conversión,
> numeración) pagó por diagnosticar antes de tocar. Las tablas son el territorio **más ad-hoc** del
> módulo — el máster no lo cubre, y no sabemos qué tipos de tabla usa ECSS. Escribir un brief de
> extracción sin caracterizar las tablas sería pedirle a CC que resuelva una decisión de arquitectura
> (¿cómo se trata cada tipo?) que no hemos tomado.
>
> **Frontera CC:** CC escribe el script + tests sobre DOCX sintético (sin `data/`). **Ejecutar sobre
> los 26 reales es UAT del usuario**; sube el output para que Claude Chat decida el mecanismo del
> Brief 5.

---

## Principio de diseño: el diagnóstico mide, no clasifica

Lección del diagnóstico de numeración: **el instrumento puede equivocarse con confianza.** Allí el
riesgo era de cálculo (el `Annex %1`); aquí es de **juicio** — «qué tipo de tabla es esto» es
interpretación, y un clasificador automático etiquetaría mal sin avisar.

Por eso este diagnóstico **extrae señales objetivas y muestra ejemplos**, pero **no decide el tipo**
de cada tabla. La clasificación en tipos (requisito / parámetros / ilustrativa / matriz…) la hace el
humano mirando el output, no un heurístico. El script reporta lo medible (dimensiones, celdas
combinadas, densidad de texto, cláusula contenedora, tipo de documento) y deja lo interpretable a la
vista. Misma disciplina que «el resolver devuelve vacío en vez de adivinar»: medir lo medible, no
fabricar juicio.

*(El script puede proponer una clasificación tentativa como conveniencia, pero siempre junto a las
señales crudas y una muestra, para que el humano la contraste — nunca como veredicto que sustituya la
mirada.)*

---

## Qué mide el diagnóstico

Recorre los 26 `.docx` de `parse_ready/` (los mismos que el parser ya procesa) y reporta, **sin
modificar nada**:

### 1. Inventario de tablas por documento, ordenado por densidad

Por documento: número de tablas, ordenado de más a menos. Recupera y actualiza el número del recon
del Brief 2 (569 sobre 14; ahora sobre 26, con los 12 convertidos que pueden traer las suyas).
Distingue explícitamente:

- **tablas en estándares** (contenido potencialmente **normativo** — requisito, transcripción literal
  en la cita), vs
- **tablas en handbooks** (contenido **informativo** — recomendación, síntesis permitida).

El tipo de autoridad de una tabla hereda del documento (decisión del Brief 4). Esto importa para la
extracción: una tabla-requisito y una tabla-recomendación se tratan distinto aguas abajo, así que el
diagnóstico tiene que separarlas desde el recuento.

### 2. Ubicación: bajo qué `clause_id` vive cada tabla

Ahora que la numeración es fiable (Brief 4), cada tabla cuelga de una sección con identidad correcta.
Por tabla, reportar la **cláusula contenedora** (`standard#section` + revisión). Doble propósito:

- Es lo que hace **citable** la tabla (su cita es la de su sección).
- **Valida que la reparación de numeración era prerequisito real:** si las tablas cuelgan de cláusulas
  con identidad correcta, la extracción puede anclarlas; si alguna cae bajo un rótulo o una sección
  sin número, es un caso que la extracción tendrá que resolver (y un dato de que quedan huecos de
  identidad).

### 3. Señales estructurales (para que el humano juzgue el tipo)

Por tabla, medir y reportar — **señales crudas, no etiquetas**:

- **Dimensiones:** filas × columnas.
- **Celdas combinadas** (`gridSpan`, `vMerge`): presencia y cuántas. Es la señal que separa «tabla
  aplanable a texto en orden» de «matriz con estructura relacional que aplanar destruiría».
- **Densidad de texto por celda:** ¿celdas con frases/párrafos (contenido) o celdas con tokens
  cortos/números (parámetros)? Distingue tabla-de-prosa de tabla-de-datos.
- **Fila/columna de cabecera:** ¿hay header declarado (`tblHeader`) o inferible?
- **Anidamiento:** ¿tablas dentro de celdas? (señal de maquetación o de estructura compleja).
- **Proporción texto-tabla vs texto-libre en la sección contenedora:** una sección cuyo cuerpo *es*
  la tabla (tabla-requisito) vs una tabla incrustada en prosa (tabla-de-apoyo).
- **Señal de maquetación:** tablas sin bordes, de 1 columna, o que envuelven un bloque de texto — 
  candidatas a **tabla-como-layout** (ruido, no contenido tabular). No decidir que lo son; señalarlo.

### 4. Muestras representativas para inspección humana

Para cada patrón estructural que emerja (cada combinación distinta de señales), **volcar 2-3 tablas
de ejemplo en forma legible** (texto de celdas en una grid ASCII/markdown) para que el humano vea qué
son realmente. El recuento sin muestra no permite juzgar el tipo; la muestra es lo que convierte
«hay 389 tablas con estas señales» en «ah, son matrices de criticidad / son requisitos numerados / 
son cajas de maquetación».

Priorizar muestras de: el documento tabla-pesado (`ECSS-E-ST-70-41C`), los handbooks convertidos, y
cualquier documento cuyas señales sean atípicas respecto al resto.

---

## Qué NO hace este diagnóstico

- No extrae tablas al corpus, no modifica el parser ni los JSON.
- No decide el tipo de cada tabla (lo hace el humano con el output).
- No decide la unidad de tabla (¿tabla=chunk? ¿fila=chunk? ¿tabla dentro del chunk de sección?) — eso
  es el Brief 5 + S07, informado por esto.
- No toca `Citation` ni el grafo.

## Qué entrega CC

1. **Script de diagnóstico** (`corpus/diagnose_tables.py`, análogo a `diagnose_numbering.py`) —
   recorre `parse_ready/`, produce el informe de las 4 partes. Sin efectos sobre datos.
2. **Tests** sobre DOCX sintético (sin `data/`): una tabla simple 2×N → señales correctas; una con
   `gridSpan`/`vMerge` → detectada como combinada; una tabla de 1 columna envolviendo texto →
   señalada como candidata a layout; una tabla bajo una cláusula conocida → `clause_id` correcto
   reportado; una tabla en handbook vs en estándar → autoridad correcta. `uv run pytest` verde.
3. **Guion UAT** — el usuario ejecuta sobre los 26 reales y sube: el inventario por documento
   (parte 1, con separación estándar/handbook), la ubicación por `clause_id` (parte 2), las señales
   estructurales (parte 3) y las muestras (parte 4). El guion pide explícitamente muestras de
   `ECSS-E-ST-70-41C` y de al menos un handbook.
4. **`S06-decisions.md`** (append), etiquetado agnóstico/dominio.

## Criterio de "hecho"

- El script inventaría las tablas de los 26 documentos, ordenadas por densidad, separando
  estándar/handbook (normativo/informativo).
- Reporta la cláusula contenedora de cada tabla, y señala las que caen bajo secciones sin identidad
  limpia (si las hay).
- Reporta las señales estructurales crudas por tabla, sin sustituir el juicio del tipo por un
  heurístico.
- Incluye muestras legibles de cada patrón estructural, suficientes para que el humano clasifique.
- Tests verdes sobre DOCX sintético.
- Output real subido, listo para que Claude Chat decida —con el usuario— la taxonomía de tablas y el
  mecanismo de extracción del Brief 5.

---

## Qué decidiremos con este output (contexto, no trabajo de CC)

- **La taxonomía real de tablas del corpus** (qué tipos hay, cuántas de cada uno) — juicio humano
  sobre las muestras.
- **El mecanismo de extracción por tipo:** aplanar a texto en orden (tablas simples), preservar
  estructura relacional (matrices), descartar (maquetación), o marcar-y-no-extraer (lo irreducible,
  con traza — la red de seguridad que decidimos para lo que la extracción no cubra).
- **La unidad de tabla** (tabla/fila/sección) y cómo hereda la cita de su sección — conecta con las
  cuatro unidades (recuperar/inyectar/citar/embeber) y con el chunking de S07.
- **El tratamiento por autoridad:** tabla-requisito (transcripción, cita-identidad) vs
  tabla-recomendación (síntesis, cita-procedencia) — hereda del documento.
