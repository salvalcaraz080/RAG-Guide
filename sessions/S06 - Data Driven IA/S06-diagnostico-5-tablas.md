# S06 — Diagnóstico de tablas del corpus: resultados

> Ejecutado sobre los 26 documentos de `parse_ready/` (2026-08-04) con
> `corpus/diagnostics/diagnose_tables.py`. Insumo para decidir, en Claude Chat, la taxonomía de
> tablas y el mecanismo de extracción del Brief 5.
>
> **El instrumento mide y muestra; no clasifica.** Los tipos de la §4 son *lectura de las
> muestras*, no salida del script — están para contrastar, no para adoptar. El script no tiene
> columna «tipo» a propósito.

---

## 1. El corpus, en números

**1.079 tablas en 26 documentos.** El recon del Brief 2 contaba 569 sobre 14; el corpus completo
casi las duplica.

| | |
|---|---|
| en estándares (contenido potencialmente normativo) | 870 |
| en handbooks (informativo) | 209 |
| **con celdas combinadas** | **600 (56%)** |
| con fila de cabecera declarada | 335 (31%) |
| candidatas a maquetación | 49 (4,5%) |
| mayoritariamente prosa | 30 (2,8%) |
| con celdas de marca (glifo ✓/✗) | 3 |
| con tablas anidadas | 6 |
| mediana | 6 × 3 |

**Dos documentos concentran el 60%:** `ECSS-E-ST-70-41C` (389) y `ECSS-E-ST-70-31C` (264). Detrás,
los handbooks de SW PA: `Q-HB-80-04A` (60), `Q-HB-80-02 Part 2A` (59), `Part 1A` (40). El
`ECSS-E-ST-40C` que solemos usar de referencia tiene 39 — el 4% del problema. **Calibrar el
mecanismo con él sería calibrarlo con el caso fácil.**

## 2. La ubicación aguanta: la reparación de numeración era prerequisito real

- **1.055 tablas con cláusula contenedora.**
- **24 huérfanas — una por documento**, y las tres inspeccionadas son el *change log* de portada
  (`ECSS-E-70-31A 9 October 2007 | First issue`). Front-matter anterior a cualquier heading
  numerado, sin contenido citable.
- **0 tablas bajo un rótulo sin número.** Ni una en 1.079.

Es la validación que el brief pedía: cada tabla del corpus cuelga de una cláusula con identidad
correcta, así que la extracción puede anclarlas. No queda ningún hueco de identidad que el
Brief 4 no cerrara.

## 3. La autoridad separa DENTRO del documento, no entre documentos

`normative 599 · informative 456 · sin sección 24`.

Los handbooks solo aportan 209 tablas, así que **unas 253 tablas informativas están dentro de
estándares** — en anexos declarados `(informative)`. En `E-ST-70-31C` son mayoría: 141
informativas frente a 122 normativas.

Consecuencia: un tratamiento diferenciado que mire el tipo de documento trataría como norma casi
un tercio de las tablas de los estándares. El eje que importa es el de la sección, que el
Brief 4 ya propaga.

**Anomalía a decidir:** `E-HB-40A` tiene **1 tabla `normative`** — `C.1.1`, un anexo DRD cuya
cabecera declara `(normative)` y sobrescribe el default del handbook. O ECSS pone DRDs normativos
en handbooks, o estamos confiando de más en el rótulo del anexo. Es 1 tabla de 209, acotado y
verificable abriendo el documento.

## 4. «Combinada» no es un caso: son tres, y rompen distinto

El hallazgo que más condiciona el mecanismo. Las 600 tablas combinadas no comparten problema:

| documento | qué codifica la combinación | qué se pierde al aplanar |
|---|---|---|
| `E-ST-70-41C` | disposición del paquete (`gridSpan`) | la estructura binaria |
| `E-HB-40A` | agrupación de filas (`vMerge`) | a qué grupo pertenece cada fila |
| `E-ST-70-31C` | anidamiento de campos | la relación padre-hijo |

El tercero es el menos evidente. `5.5.2.3 User-defined enumerated types`:

```
| Name    | Definition                     | Data Type                     |                  |
| Name    | The name of the enumerated set | Name                          |                  |
| List of | Value                          | A value of the enumerated set | Character String |
```

`List of` abarca hacia abajo y **contiene** el sub-campo `Value`. Aplanar fila a fila daría
`List of | Value | A value… | Character String` y perdería que `Value` es hijo de `List of` — que
es precisamente lo que define el tipo.

Y el de `E-HB-40A`, `Annex A Documentation Requirement List`: 70×7 con **0 `gridSpan` y 69
`vMerge`**. La columna de agrupación abarca decenas de filas; sin ella, 70 filas sueltas sin
contexto.

**Un mecanismo que trate las 600 como un solo caso resolverá bien uno de los tres.**

## 5. Lo que se ve en las muestras (lectura, no veredicto)

Patrones distinguibles, con dónde mirarlos:

1. **Tabla de códigos / clave-valor** — `7.3.4 Unsigned integer` (PFC → format definition),
   `3.3 Abbreviated terms`, `B.7.2 Bit-manipulation functions`. Cabecera declarada, sin combinar.
   **Aplanables sin pérdida.**
2. **Diccionario de datos** — `5.1 Data definition`, `5.5.2.3`. `Name | Definition | Data Type`.
   En `E-ST-70-31C` **el contenido normativo del estándar *es* la tabla**: es un estándar de
   definición de datos, y hoy está en el corpus vacío de su sustancia.
3. **Matriz de conformidad** — `C.2.x ST[02] device access` (13×8, 12 combinadas):
   `system / interface / message type / minimum / by declaration`. Relacional.
4. **Matriz de correspondencia dispersa** — `A.4.3 Service requests and reports` (9×11, 26
   combinadas), con cinco filas vacías antes del contenido.
5. **Diagrama de estructura de paquete** — `7.2.1 Structure diagram`, `8.12.2.26`. El layout *es*
   el contenido; candidato al «marcar y no extraer».
6. **Definición de campo en una columna** — `8.4.2.1` (`reset flag / Boolean / optional`).
   Contenido normativo real, **no maquetación**.
7. **Caja vacía 1×1** — `6.9.4.2`, `6.9.4.3`. Cero caracteres. Contenedores de figura.
8. **Plantilla en blanco** — `4.2.2 Compliance with ECSS-E-ST-40C`: cabecera y dos filas vacías.
9. **Checklist con columna de relleno** — `7.5.2.5`: las preguntas son la guía, la columna
   `Verified` es ruido.
10. **Tabla de prosa con «shall» en un handbook** — ver §6.
11. **Referencias normativas** — ver §7.
12. **Change log** — las 24 huérfanas.

## 6. La trampa de autoridad: handbooks que dicen «shall»

`E-HB-40A`, `4.2.6.3 Software Delivery Modalities` (14×2, 86% prosa) y `4.2.6.6.2 Warranty`:

```
| | All the deliverables shall be written … |
| | The Contractor shall offer a rolling warranty … |
```

Son **cláusulas contractuales modelo** que el handbook ofrece para copiar. Formalmente
informativas, pero el texto dice «shall». Recuperada una de éstas, el modelo presentaría como
obligación lo que es una sugerencia de redacción.

Cuantitativamente es marginal (30 tablas de prosa en todo el corpus). Cualitativamente es el peor
fallo posible del producto: afirmar una norma que no existe. Es exactamente el caso que hace que
la cita-procedencia («esto viene del handbook X») deje de ser un matiz de presentación.

## 7. Las referencias normativas viven en una tabla que el parser descarta

`2 Normative references` es una tabla en `E-ST-70-41C`, en `E-HB-40A` y en `E-ST-70-31C` — los
tres documentos inspeccionados, sin excepción:

```
| ECSS-E-ST-50    | Space engineering - Communications      |
| ECSS-E-ST-70-01 | Space engineering - On board control p… |
```

No es anécdota: **es la lista de referencias cruzadas de cada documento, y el pipeline la tira**.
Conecta con el 49% de referencias del grafo sin `target_clause_id` — parte de ese hueco puede
estar aquí. Merece medirse antes de decidir nada sobre el grafo (S07).

## 8. Límites conocidos del instrumento

El script se corrigió dos veces durante el propio diagnóstico, y queda una limitación abierta:

- **Corregido — glifos de símbolo.** El primer humo reventó con un carácter Wingdings que la
  consola no sabía codificar. Resultó ser la marca de aplicabilidad de las matrices, así que se
  cuenta como señal (`symbol_cells`) y se pinta como `[sym]`. Aparece en 3 tablas del corpus: el
  patrón «matriz con ✓» es marginal, las matrices reales usan texto.
- **Corregido — ventana de muestra.** Con 5 columnas y las 4 primeras filas, `A.4.3` (9×11, con
  cinco filas vacías al inicio) se mostraba **completamente en blanco**. Ahora son 9 columnas y
  se saltan las filas vacías iniciales, informando cuántas.
- **Abierto — `layout_candidate` mezcla dos cosas.** Junta cajas vacías 1×1 (descartables) con
  definiciones de campo de una columna (contenido normativo). Si el Brief 5 descartara ese grupo
  en bloque, tiraría contenido. El criterio que las separa está en el output y es trivial:
  `chars/celda = 0` frente a `chars/celda = 8`. No se implementó porque decidirlo es del humano.

Y el aviso general de la sesión: **es la tercera vez que un diagnóstico necesita corregirse antes
de fiarse de él.** Lo primero que hay que dudar al leer un informe es del instrumento.

## 9. Qué queda por decidir (Claude Chat)

- **La taxonomía**: cuántos tipos se reconocen y cuáles colapsan. La §5 es una propuesta de
  lectura, no una decisión.
- **El mecanismo por tipo**, sabiendo que «combinada» son tres problemas (§4): aplanar,
  preservar relación, descartar, o marcar-y-no-extraer.
- **La unidad** (tabla / fila / dentro del chunk de sección) y cómo hereda la cita. Conecta con
  S07.
- **El caso `C.1.1`**: si un anexo `(normative)` dentro de un handbook debe poder sobrescribir el
  default informativo del documento.
- **Si las tablas de referencias normativas (§7) se extraen como contenido, como referencias
  para el grafo, o ambas.**
