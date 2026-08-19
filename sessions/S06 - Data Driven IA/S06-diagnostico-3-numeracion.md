# S06 — Diagnóstico de numeración de sección: resultados

> Ejecutado sobre los 26 documentos de `parse_ready/` (2026-08-01) con
> `corpus/diagnose_numbering.py`. Insumo para decidir el mecanismo del Brief 5.
>
> **El instrumento está validado**: sus cuatro afirmaciones más cargadas se comprobaron a mano
> contra el documento renderizado, y las cuatro se confirmaron. Lo que sigue no es la salida
> cruda de un script, es una medición contrastada.

---

## 1. Validación del instrumento

La primera ejecución dio 604 divergencias y **143 eran un falso positivo propio** (el prefijo
literal `Annex %1` del `lvlText` contaba como parte del número). Corregido eso, quedan 461, y
antes de interpretarlas se validaron a mano las cuatro poblaciones:

| # | Documento | Heading | Parser dice | Resolver dice | **Word imprime** | Veredicto |
|---|---|---|---|---|---|---|
| A | `E-HB-40-01A` | `Scope` | 2 | 1 | **1** | resolver |
| B | `Q-HB-80-01A` | `Relation to other ECSS Standards` | 4.2 | 3.5 | **3.5** | resolver |
| C | `M-ST-40C Rev.1` | `Semantics of the metadata model` | L.4 | L.5 | **L.5** | resolver |
| D | `Q-ST-30-02C` | `Requirement identification…` (anexo E) | E.0.1 | E.1.1 | **E.1.1** | resolver |

Cuatro de cuatro. Sumado a que el resolver **coincide con el parser en 4.690 cláusulas
consecutivas** —incluyendo toda la numeración alfabética de anexos de los 21 estándares—, la
conclusión es que las divergencias son errores del parser, no artefactos de la medición.

En D, además, el usuario verificó el patrón completo: los seis anexos imprimen `A.1.1`, `B.1.1`,
`C.1.1`, `D.1.1`, `E.1.1`, `F.1.1`. El parser produce `E.0.1` — un número que no existe.

---

## 2. Qué está mal, y dónde

| población | cláusulas | con número erróneo | % |
|---|---|---|---|
| 21 estándares | 4.620 | **6** | 0,13% |
| 5 handbooks | 531 | **455** | **86%** |
| **total** | **5.151** | **461** | 8,9% |

Más **41 rótulos** con número fabricado donde Word no imprime ninguno (40 en `E-HB-40A`, 1 en
`E-HB-40-01A`). En total, **502 identidades de cláusula incorrectas** en el corpus.

**457 de las 461 son silenciosas** — ningún invariante las detecta. Solo las 4 de
`ECSS-Q-ST-30-02C` disparan `INV4`, y lo hacen porque producen un `0` intermedio que es
sintácticamente imposible. El resto son números plausibles y equivocados.

Los dos estándares CORE (`E-ST-40C Rev.1`, `Q-ST-80C Rev.2`) están **limpios**: 0 divergencias
sobre 727 cláusulas.

---

## 3. El mecanismo de resolución es UNO SOLO (dato clave para el Brief 5)

La parte 1 clasifica 18 documentos como `mixto` y 8 como `uniforme-ligado-a-estilo`, lo que a
primera vista sugiere familias que necesitan tratamiento distinto. **La parte 2 lo desmiente.**

`mixto` solo significa que conviven `numPr` directo en el párrafo y `numPr` heredado del estilo.
El resolver los trata igual —busca el propio, y si no lo hay sigue la cadena `w:basedOn`— y
acertó en los 26 documentos, en las dos familias, con y sin `startOverride`. No hubo que
tratar ninguna familia aparte.

**Conclusión para el Brief 5: resolver la numeración desde `numbering.xml` sirve para todo el
corpus.** No hace falta fallback por familias. Sí hacen falta los detalles que este diagnóstico
tuvo que implementar y que no son opcionales:

- seguir `w:basedOn` para la numeración ligada a estilo (sin esto, documentos enteros parecen
  no numerados);
- quedarse solo con el tramo entre marcadores `%N` del `lvlText` (sin esto, todo anexo ECSS
  sale como `Annex B` en vez de `B`);
- contadores por `abstractNumId`, no por `numId`;
- avanzar la lista también en párrafos vacíos, que consumen elemento;
- `startOverride` (lo usan 24 de los 26 documentos).

Queda un cabo suelto: `lvlOverride` con redefinición de nivel aparece en `E-HB-40-01A` y el
resolver lo detecta pero **no lo aplica**. Ese documento acertó igualmente en las cuatro
comprobaciones, pero es el único con esa característica y conviene cubrirlo en producción.

---

## 4. Tres causas, todas identificadas

**Causa 1 — un rótulo contado como sección desplaza todo lo que va detrás.**
`E-HB-40-01A` tiene *un* `Heading 1` (`Introduction`) enganchado a una lista que no imprime
número. El parser le asigna el `1`, y a partir de ahí sus **163 cláusulas** van desplazadas
exactamente una posición. Una sola cabecera mal contada corrompe el documento entero.

**Causa 2 — los niveles intermedios sin usar valen 1, no 0.**
En `Q-ST-30-02C`, un `Annex3` aparece sin un `Annex2` previo. El parser deja el contador
intermedio a 0 y produce `E.0.1`; Word renderiza el nivel no usado con su valor `start`, o sea
`E.1.1`. Es una diferencia semántica de una línea, y es la que dispara los únicos `INV4` del
corpus.

**Causa 3 — el parser asume una jerarquía única; los handbooks usan varias listas
independientes.** En `Q-HB-80-01A` y sus tres hermanos, un `Heading 1` no reinicia el contador
de los `Heading 2` porque pertenecen a definiciones de lista distintas — por eso Word imprime
`3.5` para un heading que el parser sitúa bajo la sección 4. *(Mecanismo inferido del XML; el
número impreso está verificado, la explicación no.)* Es la causa de 292 de las 455 divergencias
de handbook.

---

## 5. El daño está latente, no realizado — y eso caduca

El cruce con los datos del Brief 2 es lo que cambia la lectura de la urgencia:

| documento | divergencias | requisitos del documento |
|---|---|---|
| `E-HB-40-01A` | 163 | **0** |
| `Q-HB-80-02 Part 1A` | 162 | **0** |
| `Q-HB-80-02 Part 2A` | 65 | **0** |
| `Q-HB-80-01A` | 38 | **0** |
| `Q-HB-80-04A` | 27 | **0** |
| `Q-ST-30-02C` | 4 | 181 |
| `M-ST-40C Rev.1` | 2 | 237 |

**455 de las 461 divergencias están en documentos con cero requisitos.** Con el modelo actual
—donde la cita cuelga del requisito— esas 455 no pueden producir ninguna cita fantasma hoy:
no hay nada que citar bajo ellas.

Pero el Brief 2 estableció que esos mismos handbooks aportan 591 secciones de contenido y que
**un chunker anclado al requisito los deja fuera del corpus**. Es decir: el día que el chunking
use la sección como unidad —que es la única forma de que los handbooks aporten algo— esas 455
secciones entran al índice **con su identidad de cita equivocada**. El daño no está reparado por
la arquitectura, está aplazado por ella.

Las 6 divergencias en estándares sí son munición real hoy, pero **está sin medir cuántos
requisitos cuelgan exactamente de `L.4`/`L.5` de `M-ST-40C` y de `E.1.x`/`F.1.x` de
`Q-ST-30-02C`**. Es el único número que falta para dimensionar el daño realizado, y es barato de
obtener.

---

## 6. Qué queda abierto

- **Cuántos requisitos cuelgan de las 6 secciones divergentes de los estándares.** Único dato
  pendiente para cerrar la magnitud del daño actual.
- **`lvlOverride` con redefinición de nivel** (§3): detectado en `E-HB-40-01A`, no implementado.
- **La causa 3 está inferida, no probada** (§4). El número impreso está verificado; el mecanismo
  que lo explica, no. Si el Brief 5 resuelve desde `numbering.xml`, deja de importar —el
  resolver acierta sin necesidad de entender por qué el documento está montado así—, pero
  conviene no darla por cerrada.
- **La numeración correcta no resuelve la identidad de cita por sí sola.** Sigue pendiente la
  reversibilidad nombre-de-fichero ↔ `doc_id` (C-4 de la auditoría), que es la otra mitad del
  Brief 5.
