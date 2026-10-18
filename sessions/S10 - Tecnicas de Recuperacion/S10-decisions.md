# S10 · Decisiones — Brief 1 (golden set de retrieval y arnés de medición)

> Decisiones tomadas al implementar `S10-brief.md`, etiquetadas **[AGNÓSTICO]** o **[DOMINIO]**.
> D1–D8 del brief se implementan tal cual y no se repiten; aquí va lo que el brief delegó (D9), lo
> que hubo que decidir al implementarlas y los resultados de las verificaciones de §3.6.
>
> **Verificación:** `uv run pytest` **1069 verde** (1017 + 52 netos). Contra la pila real: export
> generado, arnés ejecutado de punta a punta con un golden set de humo (10 consultas, 2
> configuraciones, salida «inválida» y pares exportados como se espera) y las cuatro
> verificaciones baratas. La anotación la ejecutó el usuario (caso 1 de la frontera: no hay
> oráculo); con ella, la tabla vector vs híbrida de §7.

---

## 1. Ubicación (D9)

| Qué | Dónde | Versionado |
|---|---|---|
| Código: export, importador, métricas, arnés | `corpus/groundtruth/retrieval/` | sí |
| Protocolo de anotación | `corpus/groundtruth/retrieval/protocol.md` | sí |
| Golden set | `corpus/groundtruth/retrieval/golden_set.json` (lo escribe el importador) | sí |
| Export de cláusulas + pares sin juzgar | `data/groundtruth/retrieval/annotation/` o un workspace externo | no |
| Informes de medición | `data/groundtruth/retrieval/measurements/` | no |
| Verificaciones baratas | `scripts/lexical_checks.py` + `scripts/output_lexical_checks.txt` | sí |

- **[AGNÓSTICO] `corpus/groundtruth/`, no `scripts/` ni un paquete nuevo.** La frontera de
  `CLAUDE.md` pregunta «¿escribe, y se vuelve a ejecutar?»: el arnés escribe plantillas para
  etiquetar (export, pares) y se re-ejecuta con cada técnica — exactamente la definición de
  `corpus/groundtruth/`. Un paquete de nivel superior (`evaluation/`) habría adelantado la decisión
  pendiente del «stack de evaluación». Que `corpus/` importe de `app/` tiene precedente
  (`embed_all`, `embedding_sanity`).
- **[AGNÓSTICO] Protocolo y golden set junto al código que los valida**, fuera de `data/`: una sola
  carpeta contiene la vara (protocolo), la medida (golden set) y quien la aplica.
- **[AGNÓSTICO] Lo que lee el anotador y lo que mide el arnés, en carpetas distintas**
  (`annotation/` vs `measurements/`), para que el protocolo pueda prohibir abrir resultados sin
  depender de que el anotador distinga ficheros.
- **[AGNÓSTICO] `.gitignore` gana `data/groundtruth/retrieval/`.** Solo cubría `*.csv`, así que el
  export (6,2 MB de texto ECSS) y los informes habrían entrado en un commit.
- **[DOMINIO] Las verificaciones van a `scripts/`**: investigación de esta sesión, no herramienta
  que se re-ejecuta.

## 2. El anotador, aislado del repo

- **[AGNÓSTICO] La sesión de anotación corre en un workspace fuera del repo**, con solo
  `protocol.md`, `INDEX.md` y `clauses/`. El brief la abría *sobre el repo*, pero una sesión de
  Claude Code ahí carga `.claude/CLAUDE.md`, que describe el retriever (modo, top-k, fusión,
  fallos conocidos): eso rompe D4 antes de empezar. Fuera del repo no hay CLAUDE.md de proyecto en
  la cadena de directorios (comprobado: ni en `proyectos\` ni en `salca\`), y la memoria de
  proyecto es otra. El export ya admite `--out`.
- **[AGNÓSTICO] El protocolo pide dos campos de cabecera nuevos: `criterion` e
  `input_fingerprint`.** El brief exige rechazar si faltan, pero el formato del protocolo no los
  tenía; el anotador los copia (el criterio, de su propio texto; el fingerprint, de `INDEX.md`) y
  el importador los compara. Se mantiene la **versión 1.0**: nunca se ejecutó, y el criterio no
  cambia. Otros ajustes de redacción: rutas del workspace, nombre de los ficheros de salida, y que
  toda consulta lleve su objeto `judgments` aunque esté vacío (distingue «ninguna relevante» de
  «no hecha»). **Si Claude Chat prefiere un 1.1, es un cambio de una línea.**

## 3. Export de cláusulas

- **[DOMINIO] Mismo universo que la base:** se descartan los 181 marcadores no citables con el
  mismo predicado que la ingesta (`is_non_citable_marker`). El anotador juzga lo que el retrieval
  puede devolver. **4.433 cláusulas, 26 documentos, 6,2 MB** en 26 ficheros (uno por documento) +
  `INDEX.md`. `input_fingerprint` `ab3fce8b5cb1a0d4`.
- **[DOMINIO] Orden de chunks dentro de la cláusula = orden del artefacto** (el chunker emite
  requisitos antes que el contenido de sección). El orden de documento no es derivable — el mismo
  límite que `group_by_clause` ya declara. Las cláusulas sí van por orden de componentes
  (`5.2 < 5.10 < A.1`). Lo dice la cabecera de cada fichero, junto con que las secciones partidas
  con solapamiento repiten alguna frase.
- **[DOMINIO] Números de sección repetidos: 1 de 4 llega al export** (`ECSS-Q-HB-80-02 Part 2A
  §4.1.1`). Los otros tres (`Q-HB-80-01A §4` y `§5`, `Q-HB-80-02 Part 1A §4`) son encabezados sin
  chunks propios. La cláusula marcada lleva sus dos títulos y el aviso.
- **[AGNÓSTICO] El texto contra el que se valida una cita es el texto exportado** (verbatims unidos
  por línea en blanco), así que una cita que cruce de un chunk al siguiente casa: tras normalizar,
  el separador es un espacio.

## 4. Importador

- **[AGNÓSTICO] Rechazo con todas las razones a la vez**, no la primera: un fichero de 10 consultas
  con tres errores se arregla en una vuelta.
- **[AGNÓSTICO] Dos niveles de casado de cita.** Literal = la normalización del validador de citas
  (`citations.normalize`, la misma que usa `validate_quotes`): pasa sin más. Solo tras quitar toda
  puntuación: **revisión**. Ni así: **rechazo**. Mínimo de 40 caracteres normalizados; una cláusula
  más corta solo se cita entera.
- **[AGNÓSTICO] Los ítems prohíben claves desconocidas** (`extra="forbid"`): una errata como
  `evidence_qoute` se leería como cita vacía. Los ficheros las ignoran, para que el anotador pueda
  devolver el de pares con su cabecera.
- **[AGNÓSTICO] Nunca sobrescribe:** `import` no reemplaza un `golden_set.json` existente, y un par
  ya juzgado se rechaza en modo B. Cambiar un juicio es editar el fichero a mano y revisar el diff.
- **[AGNÓSTICO] La revisión vive en el golden set** (`review` por juicio, `golden review` la lista,
  el informe la cuenta). Resolverla es poner `review` a `null` o retirar el juicio.

## 5. Métricas y arnés

- **[AGNÓSTICO] Una configuración es un `ContextProvider` más su pool de candidatos.** El arnés
  mide el seam de producción, así que una técnica futura (filtro, pesos, reranker, expansión) es una
  entrada en `CONFIGURATIONS`. Las de partida: `vector` e `hybrid` con `top_k` y
  `RETRIEVAL_HISTORY_QUESTIONS` de settings, candidatos = 50 chunks (`CANDIDATE_POOL`).
- **[AGNÓSTICO] La query de retrieval es la de producción:** el arnés llama a
  `llm_service._compose_retrieval_query` en vez de reimplementarla. Importar una función privada es
  el precio de que D8 no pueda divergir.
- **[AGNÓSTICO] `k` = cláusulas entregadas, por consulta** (varía: 6–10 con `top_k=10` en el humo).
  Un ratio sin denominador es `n/a`, nunca 0: sin relevantes conocidas no hay recall.
- **[AGNÓSTICO] Cláusula normativa = la que tiene algún chunk normativo.** Una cláusula mezcla
  requisito y NOTE; cuenta como normativa si algo de ella obliga.
- **[AGNÓSTICO] Calidad de la primera ejecución, latencia de las 5 siguientes** (mediana, embedding
  incluido). Además el arnés marca si las 6 ejecuciones entregaron las mismas cláusulas (`stable`):
  el embedding de la query no es bit a bit determinista. En el humo, 20/20 estables.
- **[AGNÓSTICO] Los pares sin juzgar se exportan ordenados por cláusula, sin configuración ni
  rango**: el anotador no puede saber qué puso primero cada sistema.
- **[AGNÓSTICO] Guardas de corpus:** el arnés se niega si el export o la base tienen otro
  `input_fingerprint` que el golden set. Salida 0 válida, 3 inválida, 1 negada (el 2 es de
  argparse).

**Humo contra la base (golden set falso, 1 juicio):** las 10 consultas × 2 configuraciones corren;
**109 pares únicos sin juzgar** entre las dos (75 en `vector`, 80 en `hybrid`). Es el orden de
magnitud de la primera ronda de modo B, menos lo que la fase 2 ya cubra. Latencia de retrieval,
mediana: **~300 ms `vector`, ~370 ms `hybrid`**, dominada por el embedding de la query.

---

## 6. Verificaciones baratas (§3.6)

Salida completa en `scripts/output_lexical_checks.txt`. Coste: 2 embeddings.

### 6.1 Identificador como contenedor — **confirmado para E-ST-40C**

| Query | Modo | Del nombrado | Lo citan | Otros | Cláusulas |
|---|---|---|---|---|---|
| «What does ECSS-E-ST-40C require for dynamic memory…» | vector | 0 | 0 | 9 | 9 |
| | hybrid | **0** | **3** (las 3 por el canal léxico) | 4 | 7 |
| «…category B and C in ECSS-Q-ST-80C?» | vector | 2 | 1 | 3 | 6 |
| | hybrid | 2 | 2 | 4 | 8 |

La híbrida no trae ni una cláusula de `ECSS-E-ST-40C`: lo que el canal léxico añade son cláusulas
de handbooks (`E-HB-40A`, `Q-HB-80-02 Part 2A`) que **mencionan** el identificador. El
identificador funciona como contenedor de documentos que lo citan, no como filtro. En Q-ST-80C las
dos del nombrado ya venían por el vector.

### 6.2 Tokenización de IDs — **el canal léxico rompe los compuestos antes de Postgres**

`ts_debug('english', 'ECSS-E-ST-40C')` da el compuesto `ecss-e-st-40c` (**0,7 %** de los chunks)
**y** sus partes `ecss`, `e`, `st`, `40c`. Pero `retrieval.lexical_terms` parte la pregunta por
caracteres no alfanuméricos *antes* de llegar a Postgres, así que la tsquery real es
`'ecss' | 'e' | 'st' | '40c'`: el compuesto discriminante nunca se busca.

| Lexema | df (chunks) |
|---|---|
| `ecss` | 9,6 % |
| `st` | 7,5 % |
| `e` | 6,1 % |
| `q` | 3,5 % |
| `40c` / `80c` | 1,1 % / 0,6 % |
| compuesto `ecss-e-st-40c` / `e-st-40c` / `q-st-80c` | 0,7 % / 0,4 % / 0,4 % |

`e`/`st` no casan medio corpus, pero sí ~6–8 % cada uno, y en OR. `5.3.5.1` llega como
`'5' | '3' | '1'` (Postgres lo guardaría como token `version`, 7 chunks); `Rev.2` como `'rev' | '2'`
(`2`: 6,4 %). **Hallazgo para el brief 2:** el pre-tokenizado de `lexical_terms` destruye la señal
que Postgres sí conserva.

### 6.3 Hipótesis IDF — **confirmada**

`ts_rank` (normalización 0) no tiene componente IDF. Arquetipo 1, «requirements for dynamic memory
management in flight software»: el término raro, `dynam` (df **0,8 %**), aparece en **1 de los 10**
primeros léxicos; los comunes, `requir` (20,6 %) y `softwar` (15,1 %), en todos, con frecuencias de
hasta 17 y 52 en chunks largos. 8 de los 10 son prosa de `E-HB-40A`. Arquetipo 4, «verification
requirements»: los 10 primeros son tablas largas de anexos DRD (`E-ST-40C §Q`, `§R`) con `requir`×20.
**Gana quien repite términos comunes en chunks largos**, que es lo que degrada a la genérica en RRF.

### 6.4 Volumen y versiones

**Ninguna revisión duplicada**: un documento por estándar. Pero **`ECSS-E-ST-70-41C` es el 35,2 %
de los chunks** (4.821 de 13.695), frente al 7,9 % de `E-ST-40C`; en el humo, 6 de las 9 cláusulas
vectoriales de la pregunta sobre memoria dinámica en E-ST-40C son de `E-ST-70-41C`.

---

## 7. Golden set y primera tabla: vector vs híbrida

**Golden set** (`claude-opus-5-5`, 2026-10-12): 10 consultas, fase 2 con **301** cláusulas
relevantes (16–55 por consulta) y **lista de revisión vacía** — todas las citas casaron con la
normalización del validador. Una ronda de modo B: **52 pares** sin juzgar entre las dos
configuraciones, **12 relevantes** (23 %) que la fase 2 no había encontrado. Total: **353 juicios,
313 relevantes**. Sin drift check.

**Medición válida** (0 sin juzgar en las dos), `top_k=10`, `history_questions=2`:

| Estrato | n | p@k vec / hyb | R@k vec / hyb | R@k norm. vec / hyb | R@cand vec / hyb |
|---|---|---|---|---|---|
| names_standard | 2 | **0,41** / 0,29 | **0,17** / 0,07 | **0,17** / 0,06 | **0,48** / 0,42 |
| generic | 2 | **1,00** / 0,88 | 0,34 / 0,34 | 0,20 / 0,20 | **0,66** / 0,53 |
| paraphrase | 1 | **0,70** / 0,44 | **0,15** / 0,09 | **0,09** / 0,00 | **0,28** / 0,23 |
| comparative | 2 | 0,75 / 0,75 | 0,17 / **0,23** | 0,04 / 0,04 | **0,49** / 0,40 |
| cross_standard | 2 | 0,74 / **0,89** | 0,15 / 0,15 | 0,03 / 0,03 | **0,48** / 0,44 |
| multi_turn | 1 | 0,25 / **0,33** | 0,09 / **0,13** | 0,05 / 0,05 | **0,26** / 0,17 |
| **todas** | 10 | **0,68** / 0,64 | **0,19** / 0,18 | **0,10** / 0,07 | **0,48** / 0,40 |

Latencia de retrieval (mediana): **297 ms `vector`, 414 ms `hybrid`**.

Lectura (indicativa: 1–2 consultas por estrato):

- **La híbrida actual no gana en ningún estrato de forma clara**, y pierde justo donde se esperaba
  que ayudara: `names_standard` (p@k 0,41 → 0,29). Coincide con §6.1: lo que el canal léxico añade
  son cláusulas que citan el estándar, no las suyas. Gana en `cross_standard` (precisión) y en la
  única `multi_turn`, con una consulta por lado.
- **El problema dominante es de recall, no de orden.** R@candidatos es **0,48** en `vector`: la
  mitad de las relevantes conocidas no aparece ni en los 50 primeros chunks, así que un reranker
  sobre ese pool tiene un techo bajo. En `hybrid` el pool es peor (0,40).
- **R@k está acotado por construcción**: con k = 5–10 cláusulas y 16–55 relevantes por consulta, el
  máximo alcanzable promedia ~0,27 en `vector`. p@k y R@candidatos son las columnas informativas.
- **Lo normativo llega poco** (R@k normativo 0,10 / 0,07, frente a 0,19 / 0,18 global).
- **q01** («Which project reviews does ECSS-E-ST-40C define…», 44 relevantes, 43 de E-ST-40C):
  `vector` entrega 5 cláusulas, 1 relevante; R@cand 0,20. Es el caso del estándar nombrado.
- **Estabilidad:** 19/20 ejecuciones estables; `q03` en `vector` cambió de cláusulas entre
  repeticiones (el embedding de la query no es bit a bit determinista).

Informe completo: `data/groundtruth/retrieval/measurements/measurement_20261012-201354.md`.
