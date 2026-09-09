# S08 · Decisiones — Brief 5 (búsqueda híbrida)

> Decisiones crudas tomadas al implementar, etiquetadas **[AGNÓSTICO]** (transferible a
> cualquier sistema RAG) o **[DOMINIO]** (específico de ECSS / de este proyecto).
>
> **Verificación completa:** `uv run pytest` **837 verde** (28 tests nuevos), lint limpio, y la UAT
> ejecutada entera contra la pila real (`S08-brief-5-uat.md`). Los números de §1 están **medidos**.

---

## 1. El veredicto B/C: **la híbrida sí reordena, y la hipótesis del brief queda refutada**

El brief anticipaba que probablemente **no** subiría, con un argumento razonable: los chunks que
adelantan al discriminante «también contienen los tokens b, c, criticality, category», así que un
`ts_rank` básico no tendría con qué separarlos.

**Medido, sube — y de forma limpia:**

```
>> discriminante: vector=3 hybrid=1  ->  sube
```

| # | modo vector | modo híbrida |
|---|---|---|
| 1 | `ECSS-E-HB-40A §4.2.4.2` — «…es categoría A, B o C» | **`ECSS-Q-ST-40C §6.5.6.3` [table]** (v#3 l#7) |
| 2 | `ECSS-Q-ST-40C §6.5.6.3` — «se asignará (A,B,C)» | **`ECSS-Q-ST-80C §D.1` [table]** (v#6 l#5) |
| 3 | **`ECSS-Q-ST-40C §6.5.6.3` [table]** ← el bueno | `ECSS-E-HB-40A §4.2.4.2` (v#1 l#16) |

Dos cosas que no estaban previstas:

1. **La tabla D-1 real de `ECSS-Q-ST-80C` entra en el puesto 2.** Es *la* tabla del par trampa de
   S07, y el vector puro la dejaba en el rango 6 — **fuera del top-5**. Así que la fusión no solo
   reordenó: **recuperó** algo que el vector no devolvía. El encuadre «es de ranking, no de
   recall» era cierto para el discriminante de 40C pero no para el de 80C.
2. **Por qué funcionó**, contra el pronóstico: los chunks que estorbaban son cortos y
   definicionales («critical software is… category A, B or C»), mientras que las **tablas** son
   largas y repiten `category`, `criticality`, `B` y `C` muchas veces. `ts_rank` pondera por
   frecuencia de término, así que premió justamente al contenido que *enumera fila por fila* — que
   es lo mismo que lo hace discriminante. La señal «es una tabla que desglosa por categoría», que
   el brief daba por no capturable, resultó tener un correlato léxico: **densidad de repetición**.

Lección transferible: *un canal léxico no distingue «explica la diferencia» de «menciona los
términos», pero sí distingue densidad — y en un corpus tabular la densidad de repetición es un
proxy sorprendentemente bueno de «este fragmento desglosa el tema».*

### 1.1 …y a cambio **degrada la query genérica**

Esto no lo pedía el brief y es la mitad del resultado:

| arquetipo | solape top-5 | top-1 vector cae a | d media vector → híbrida |
|---|---|---|---|
| 1 match directo | 3/5 | 2 | 0,3822 → 0,3958 |
| 2 reformulación | 3/5 | 1 | 0,3985 → 0,3997 |
| 3 fuera de dominio | 3/5 | 1 | 0,4939 → 0,4858 |
| **4 ambigua** | **1/5** | **4** | 0,4488 → **0,4932** |
| 5 identificadores | 2/5 | 3 | 0,2678 → 0,2926 |

En «verification requirements» la híbrida **solo conserva 1 de los 5** resultados del vector, y su
nuevo puesto 1 es un chunk que el canal vectorial rankeaba **el 47**. La causa es estructural: con
términos OR-eados, una query genérica casa 3.199 chunks y `ts_rank` los ordena por densidad, que en
una query sin especificidad no correlaciona con relevancia. **RRF sin pesos le da al canal léxico
el mismo voto siempre.**

**Conclusión operativa, que es el enunciado exacto de lo que S10 tiene delante:** el canal léxico
ayuda donde el vector falla (identificadores) y estorba donde el vector acertaba (genéricas). Lo
que falta no es otro mecanismo sino **modular su peso** — y eso es afinado de fusión, exactamente
lo que este brief difería.

### 1.2 Una trampa de lectura: la distancia media empeora incluso cuando la híbrida acierta

En el arquetipo 5 la fusión es claramente mejor y la distancia media **sube** (0,2678 → 0,2926),
porque los chunks semánticamente más cercanos eran precisamente los equivocados. Refuerza lo que
midió el brief 4 §8.5: **la distancia no es una métrica de calidad**, ni siquiera agregada. Quien
la usara como criterio de aceptación de la híbrida habría concluido que empeora en 4 de 5 casos.

---

## 2. Se indexa `cite_text`, jamás `embed_text` — **[AGNÓSTICO]**

El invariante `embed ≠ cite` de S07, aplicado por tercera vez y en un sitio nuevo. Si el índice
full-text se construyera sobre el texto embebido, el **header contextual** (estándar + ruta de
secciones) entraría en el índice, y una query que mencionara «5.4.3» casaría por ese header con
**todos** los chunks bajo esa sección. Serían matches léxicos fabricados por nosotros, no presentes
en el documento. El match por identificadores se sirve con las columnas tipadas y sus filtros.

La columna es **generada y STORED**, no escrita por la aplicación ni por un trigger: la deriva
entre `cite_text` y su tsvector es imposible por construcción, y **la ingesta no ha necesitado
ningún cambio** — los 13.876 chunks ya cargados tienen su tsvector sin re-ingerir nada.

---

## 3. Los términos van con OR, no con AND — **[AGNÓSTICO]** (decisión no trivial)

`plainto_tsquery` y `websearch_to_tsquery` ANDean los términos por defecto. Con eso, «requirements
for dynamic memory management in flight software» exige que un chunk contenga los seis lexemas:
casi ninguno lo hace.

El fallo sería **silencioso y perfecto**: el canal léxico devolvería 0 filas, la fusión RRF
degeneraría en el ranking vectorial, y la híbrida se comportaría exactamente como el modo vector
**pareciendo que había corrido**. Un test de humo diría «funciona». Con OR, quien discrimina es
`ts_rank`, que es su trabajo.

Las palabras `or`/`and`/`not` que vengan dentro de la pregunta se **descartan** en vez de
escaparse: dentro de la frase de un usuario son palabras que buscar, no sintaxis, y dejarlas
reescribiría la query en silencio.

---

## 4. Fusión por rango y no por score — **[AGNÓSTICO]**

La distancia coseno (menor es mejor, rango observado 0,26–0,51) y `ts_rank` (mayor es mejor, escala
abierta) no comparten escala. Combinarlas por valor exige normalizar y elegir pesos, que es
justamente el afinado diferido.

Hay además una razón medida, no solo de principio: el brief 4 §8.5 demostró que **la escala de
distancia no está calibrada** (fuera-de-dominio y genérico-en-dominio se solapan). Fundir por
score construiría sobre un número que ya sabemos que no soporta esa lectura. RRF solo usa el
**orden**, que es lo único de lo que ambos canales sí aportan información fiable.

Detalles que resultaron ser decisiones:

- **Desempate determinista** (score, luego rango vectorial, luego id). Sin criterio explícito el
  orden dependería de la iteración del dict — no-determinismo invisible en un resultado que se
  compara entre corridas, que es exactamente lo que hace este arnés.
- **Cada hit reporta su rango en cada canal y su score RRF.** Sin eso, «¿la fusión movió este
  chunk, y gracias a qué canal?» habría que re-derivarlo; con eso, el informe se lee. Es lo que
  permitió ver que la tabla D-1 entró por rango léxico 5 desde un rango vectorial 6.
- **`distance = None` para un hit que solo trajo el léxico.** Nunca se midió contra el vector de
  la query. Rellenarlo con 0 lo haría parecer un match perfecto; con 1, el peor. Se admite el
  hueco, y el arnés lo imprime como `d= --`.

---

## 5. Dos sentencias y fusión en Python, no un SQL con full outer join — **[AGNÓSTICO]**

Cada canal es su propia query, con sus filtros en su `WHERE`, así que **no se renuncia a filtrar en
SQL** (que era el argumento de pgvector). Lo que se gana es que la fusión queda como **lógica pura**
y 10 de los 28 tests nuevos la ejercitan sin base de datos.

Los filtros se aplican **igual en los dos canales**, con la misma validación: un hit excluido por
filtro en el vectorial no puede reaparecer por el léxico. Se factorizó `_apply_filters` y
`_selection` para que eso sea cierto por construcción y no por copiar y pegar bien.

---

## 6. Sorpresas

### 6.1 103 chunks sin contenido indexable — **[DOMINIO]**, hallazgo incidental

La migración dejó 103 tsvector vacíos. No es un fallo: su `cite_text` es literalmente `-` (51) o
`<<deleted>>` y variantes (52) — **marcadores de cláusula retirada en revisión** que el parser
conservó como contenido citable.

Lo relevante es que **tienen embedding y el canal vectorial puede devolverlos** como resultado. El
léxico los excluye por accidente (no producen lexemas), no por diseño. Es una decisión de chunking
—¿debe existir un chunk cuyo texto dice que no hay texto?— y está anotada como pendiente.

Lección transferible: *añadir un segundo canal de recuperación audita el primero. El léxico no
buscaba encontrar basura en el índice vectorial; la encontró porque no pudo indexarla.*

### 6.2 El test de deriva modelo↔migración saltó por tercera vez, y por un motivo nuevo

La primera versión de la migración añadía la columna con SQL crudo (`op.execute("ALTER TABLE …")`),
que es lo natural para una columna generada. El test exige que el nombre de columna aparezca
**entrecomillado** en el fuente de alguna migración, y en SQL crudo no lo está.

En vez de relajar el test, se reescribió la migración con `op.add_column(sa.Column("cite_tsv",
TSVECTOR(), sa.Computed(...)))`, que emite el mismo DDL y es consistente con las otras dos. El test
tenía razón por el motivo equivocado, y forzó la versión mejor.

### 6.3 `literal_binds` no sabe renderizar `REGCONFIG`

Compilar la query léxica con `literal_binds=True` —lo que hacía el script de `EXPLAIN` del brief
4— revienta con `No literal value renderer is available for literal value "'english'" with datatype
REGCONFIG`. No afecta a producción (ahí va parametrizada); solo al arnés de diagnóstico, para el
que se escribió el `EXPLAIN` a mano.

---

## 7. Lo que se dejó sin hacer, según el brief

- **Sin tuning de fusión**: ni pesos, ni `c` afinado, ni score normalizado. El resultado de §1.1
  dice exactamente por qué hace falta, y es el encargo de S10.
- **Sin config de text-search a medida** ni criticidad como columna tipada. Medido con `english`
  básico, que resultó suficiente para el caso B/C.
- **Sin reranking ni expansión por grafo.** Nota: el brief 4 apuntaba al reranker como el fix más
  probable del ranking B/C. La medida de este brief dice que **el canal léxico ya basta para ese
  caso**, así que el reranker queda para lo que el léxico no resuelve — que ahora sabemos que es
  la query genérica.
- **Sin cablear el seam**, sin índice vectorial, sin `halfvec`, sin re-embeber, sin abrir deuda
  diferida.
