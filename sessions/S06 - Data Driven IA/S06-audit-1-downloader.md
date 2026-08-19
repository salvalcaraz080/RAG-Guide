# Auditoría del downloader y el scraper — S06 brief 1

**Alcance:** `corpus/downloader.py` y `corpus/scraper.py`, auditados **por lectura de código**.
No se ha ejecutado ninguno de los dos (red + `data/raw/` + `data/inventory/` son territorio UAT).
Los hallazgos que dependen de los bytes reales en disco se resuelven con
`scripts/audit_raw_corpus.py`, que entrega esta auditoría para que lo ejecute el usuario
(ver «Guion de verificación manual» al final).

**Cómo leer las severidades.** El criterio no es «qué tan roto está el código» sino **qué tan
invisible es el daño**. Un fallo ruidoso en un pipeline de datos es barato; un fallo que produce
un corpus plausible pero equivocado se paga tres capas más abajo, en una cita que un ingeniero
aeroespacial se cree.

**Política de fallo** (cada hallazgo del Eje 1 la lleva clasificada):
- **abortar** — el pipeline para, el usuario se entera.
- **saltar+log** — el documento se pierde, queda traza.
- **silenciar** — el documento se pierde o se corrompe y **nada** lo registra. Es el pecado.

---

## Resumen ejecutivo

> **Actualizado tras ejecutar `scripts/audit_raw_corpus.py` sobre el corpus real (2026-07-29).**
> Los datos reordenan el informe: el fallo que la lectura de código señalaba como más peligroso
> **no se ha materializado**, y ha aparecido otro, mayor, que auditar el downloader por sí solo
> no podía ver. Ver «Verificación sobre datos reales» abajo para la evidencia.

**El hallazgo dominante es C-15: `data/processed/` contiene 14 documentos de los 26 activos que
el downloader descargó.** Los 12 ausentes son `.doc` legacy que el parser no lee y que
`verify_parser.py` descarta **en silencio**. El downloader hizo su trabajo bien; el corpus que
M2 va a consumir son, sin que nada lo diga, 14 de los 26 documentos que el inventario declara
(cobertura 53%). Entre lo perdido está `ECSS-E-HB-40A`, el handbook de ingeniería de software.

Detrás va la familia que sí detectó la lectura de código, y que sigue siendo válida como
endurecimiento aunque hoy no haya daño realizado: **el downloader confía en que lo que le
devuelve el servidor es lo que pidió, y nunca lo comprueba.** No verifica que el payload sea un
Word (C-2), ni que el enlace elegido sea el documento principal y no un anexo (C-3), ni que un
fichero ya presente en `data/raw/` sea íntegro (C-5). Los 28 ficheros en disco han pasado la
verificación de magic bytes: **el riesgo es real y está sin cubrir, pero no se ha cobrado
ninguna pieza todavía**.

C-4 (identidad de cita) hay que **matizarlo a la baja respecto a la primera versión de este
informe**: el round-trip `ECSS-E-ST-40C Rev.1` → `_Rev_1` → `Rev 1` pierde el punto, y 14 de 27
ficheros lo sufren, pero **los JSON de `data/processed/` conservan el `Rev.1` correcto**. Quien
generó el corpus procesado no pasó por la ruta lossy. Es una trampa armada, no un daño hecho —
lo que no la hace menos urgente para Brief 3, porque M2 escribirá su ingesta copiando alguno de
los dos patrones que hoy conviven.

Lo demás está razonablemente bien: el staging `.tmp` está bien pensado, el filtrado de
superseded funciona, los fallos de red degradan en vez de abortar, la separación red/lógica ya
permitía testear sin red, y la descarga secuencial con 2 s de espera es la decisión correcta y
**no debe paralelizarse**.

| # | Hallazgo | Severidad | Política hoy | Sobre datos reales | Estado |
|---|---|---|---|---|---|
| C-15 | El corpus procesado son 14 de 26 activos; los `.doc` se pierden sin traza | **Crítica** | **silenciar** | **confirmado** | **abierto → Brief 2** |
| C-2 | No se verifica que lo descargado sea un Word | Alta | **silenciar** | no materializado (28/28 OK) | **aplicado (1b)** |
| C-1 | Vigencia delegada por completo a `source_type`, sin verificación | Alta | saltar+log (parcial) | sin conflictos hoy | **aplicado (1b, detección)** |
| C-3 | El enlace de descarga se elige sin validar identidad | Alta | **silenciar** | no observado | **aplicado (1b, detección)** |
| C-4 | Nombre de fichero ↔ doc_id no es reversible (pierde `Rev.1`) | Alta | **silenciar** | latente: 14/27 ficheros, 0 JSON | abierto → Brief 5 |
| C-5 | «Ya existe» se equipara a «está bien» | Media | **silenciar** | sin ficheros corruptos | **aplicado (1b)** |
| C-6 | El resumen no dice *qué* falló; el proceso sale con código 0 | Media | **silenciar** | — | **aplicado (1b)** |
| C-7 | Un `OSError` aborta el batch entero y se pierde el resumen | Media | abortar | — | **aplicado (1b)** |
| C-8 | La dedup del scraper depende del orden de `SOURCES` y no deja traza | Media | **silenciar** | — | **aplicado (1b)** |
| C-16 | `data/raw/` mezcla descargas con residuo manual, sin poda | Media | **silenciar** | **confirmado** (1 huérfano, 2 ficheros) | abierto (usuario + Brief 2) |
| C-9 | Acceso crudo a columnas del CSV → `KeyError` opaco | Baja | abortar | — | **aplicado (1b)** |
| C-10 | `.tmp` huérfano; y colisión de stem entre `X.doc` y `X.docx` | Baja | **silenciar** | colisión **presente** en disco | **aplicado (1b)** |
| C-11 | La respuesta HTTP no se cierra | Baja | — | — | **aplicado (1b)** |
| C-12 | `production_status` se scrapea y no se usa | Info | — | — | abierto → Brief 4 |
| C-13 | `scraped_at` es naive y usa `datetime.utcnow()` (deprecado) | Baja | — | — | **aplicado (1b)** |
| C-14 | El selector `ol li a` captura cualquier lista ordenada de la página | Baja | saltar+log | sin filas basura | **aplicado (1b)** |

---

## Verificación sobre datos reales (ejecutada por el usuario, 2026-07-29)

Salida de `scripts/audit_raw_corpus.py` sobre `data/raw/` (28 ficheros) y
`data/inventory/ecss_inventory_software.csv`.

### Lo que salió bien

- **Integridad (C-2, C-5): 28 de 28 ficheros son Word de verdad.** 15 con firma `PK\x03\x04`
  (DOCX/zip) y 13 con `\xD0\xCF\x11\xE0` (DOC/OLE2). Ninguno es HTML, ninguno está por debajo
  del umbral de tamaño plausible. **La redirección al muro de registro no ha ocurrido.** El
  agujero sigue abierto, pero conviene decirlo con precisión: es una defensa que falta, no un
  incendio.
- **`.tmp` huérfanos: cero.** El staging funciona.
- **Vigencia (C-1): sin revisiones en competencia.** El único `domain_code` duplicado es
  `Q-HB-80-02` (`Part 1A` / `Part 2A`), que es el caso legítimo previsto. `ECSS-E-ST-40C Rev.1`
  es el único activo de `E-ST-40` y su fichero está en disco; el de 2009 no está. **La
  afirmación verificable que pedía el brief se cumple.**
- **Cobertura de descarga: sin `FALTA`.** Los 26 documentos activos con `docx_url` tienen su
  fichero. El downloader no ha perdido nada.

### C-15 — El corpus procesado son 14 de 26 documentos · **CRÍTICA** · silenciar

Cruzando los tres artefactos:

| Etapa | Cuenta | Detalle |
|---|---|---|
| Inventario, activos con `docx_url` | 26 | lo que el corpus declara tener |
| `data/raw/`, ficheros legítimos | 26 | 14 `.docx` + 12 `.doc` |
| `data/processed/`, JSON | **14** | exactamente los 14 `.docx` |

Medido por el check 6 del script: `activos con DOCX: 26 | parseados: 14 | cobertura: 53%`, con
los 12 ausentes identificados uno a uno y todos `.doc` legacy.

Los 14 JSON de `data/processed/` coinciden **uno a uno** con los 14 `.docx` legítimos. Los 12
`.doc` legacy no han sido parseados nunca, y **nada en el pipeline lo dice**:

- El downloader los descarga y reporta éxito — correctamente, su trabajo es traer bytes.
- `verify_parser.py` los descarta en silencio: `_expand`
  (`corpus/verify_parser.py:139`) termina en
  `if str(p).lower().endswith(".docx")`, y con el argumento por defecto (`data/raw`) glob-ea
  `*.docx`. Corriéndolo sobre el corpus completo informaría «OK» sobre 15 ficheros **sin
  mencionar los 13 que ni miró**. Una herramienta de verificación que omite media entrada sin
  decirlo es el peor sitio posible para este fallo.
- Nadie compara 26 contra 14.

**Qué se pierde exactamente** (los 12 `.doc`, con su relevancia según `SOFTWARE_RULES`):

| Documento | Relevancia |
|---|---|
| `ECSS-E-HB-40A` — Software engineering handbook | CORE (keyword «software») |
| `ECSS-Q-HB-80-01A`, `Q-HB-80-02 Part 1A`, `Part 2A`, `Q-HB-80-04A` — handbooks de SW PA | CORE |
| `ECSS-E-ST-10-06C` — Technical requirements specification | RELATED |
| `ECSS-E-ST-70-01C` — On-board control procedures | RELATED |
| `ECSS-E-ST-70-31C` — M&C data definition | RELATED |
| `ECSS-Q-ST-30-02C` — FMEA/FMECA | RELATED |
| `ECSS-M-ST-10C Rev.1`, `M-ST-40C Rev.1`, `M-ST-80C` — gestión de proyecto/configuración/riesgo | ADJACENT |

Los dos estándares CORE normativos (`E-ST-40C Rev.1`, `Q-ST-80C Rev.2`) **sí están**. Lo que
falta es toda la capa de handbooks —incluido el de ingeniería de software— y buena parte del
tejido RELATED. Para un RAG cuyo argumento de venta son las citas verificables, eso significa
que hoy hay preguntas cuya respuesta correcta vive en un documento que el sistema descargó,
tiene en disco, y no puede ver.

**Conexión con el grafo:** CLAUDE.md registra que 135 de 273 referencias (49%) tienen
`target_clause_id` vacío. **Hipótesis comprobable, no verificada aquí:** parte de esas
referencias colgando apuntan a documentos que no están en `data/processed/` porque son `.doc`.
Si se confirma, el porcentaje de expansión por grafo mejorará solo al cerrar C-15, y la
decisión pendiente sobre «qué inyectar cuando el target es un documento entero» se toma sobre
otros números. Merece medirse antes de decidir esa política.

**Recomendación:** la conversión `.doc` → `.docx` con LibreOffice headless ya está en la lista
de decisiones pendientes de CLAUDE.md. Este hallazgo la reclasifica: **no es una mejora futura,
es la mitad del corpus**. Y, con independencia de cuándo se haga, el pipeline debe **contar y
reportar la brecha** (activos declarados vs parseados) en vez de dejarla implícita — es
exactamente la medida que el manifiesto del Brief 4 tiene que llevar.

### C-16 — `data/raw/` mezcla descargas con residuo manual · **MEDIA** · silenciar

El check 3 marca un huérfano, `ECSS-E-ST-70-01C16April2010`, presente **dos veces** (`.doc` de
0.69 MB y `.docx` de 0.29 MB) y sin fila activa en el inventario.

**Procedencia (inferida, no probada):** ese nombre **no lo pudo generar el downloader**. El
fichero en ecss.nl se llama `ECSS-E-ST-70-01C(16April2010).doc`, y `sanitize_filename` convierte
los paréntesis en `_`, lo que habría dado `ECSS-E-ST-70-01C_16April2010`. El nombre en disco no
tiene ese separador: los paréntesis fueron **eliminados**, no sustituidos — huella de una
descarga por navegador o un renombrado a mano. El `.docx` hermano, un 58% más pequeño que el
`.doc`, encaja con una conversión LibreOffice hecha manualmente. Y el `.doc` pesa exactamente lo
mismo (0.69 MB) que `ECSS-E-ST-70-01C.doc`, el que sí bajó el downloader: **es el mismo
documento con dos nombres**.

**Por qué importa más de lo que parece:** no es que sobre un fichero, es que `data/raw/` no
tiene una regla de qué puede vivir ahí. Mezcla lo que trajo el downloader con lo que puso una
persona a mano, y nada distingue lo uno de lo otro. Si alguien parsea `data/raw/*.docx` sin
filtrar por inventario, mete en el corpus un duplicado de `ECSS-E-ST-70-01C` bajo el `doc_id`
inventado `ECSS-E-ST-70-01C16April2010` — un documento que no existe. Que hoy no haya pasado es
suerte: los 14 JSON salieron de una lista basada en el inventario, no del glob.

**Recomendación:** (a) confirmar con el usuario y borrar los dos ficheros — **no los toco, son
datos suyos**; (b) tratar `data/raw/` como inmutable y **propiedad exclusiva del downloader**, y
escribir los `.docx` convertidos por LibreOffice en un directorio aparte (`data/converted/`),
de modo que «lo que dio el servidor» y «lo que derivamos» no compartan carpeta; (c) hacer que
todo consumidor del corpus itere el **manifiesto**, nunca un glob de directorio.

El punto (b) no es teórico: hoy `ECSS-E-ST-70-01C` ya existe como `.doc` **y** `.docx` en la
misma carpeta. Cuando se convierta el resto, esa colisión de stem se repetirá **12 veces más**,
y a partir de ahí un glob de `*.docx` sobre `data/raw/` dejará de poder distinguir original de
derivado. Decidir el destino de la conversión **antes** de ejecutarla cuesta cero; después,
cuesta desenredar 26 ficheros.

### C-4 — Corrección: la trampa está armada, pero no ha disparado

14 de los 27 stems en disco no sobreviven el round-trip (todos los `Rev.N`, más los dos
huérfanos). **Pero los 14 JSON de `data/processed/` se llaman `ECSS-E-ST-40C_Rev.1.json` — con
el punto intacto.** Como `save_parsed` deriva el nombre del `doc_id`
(`corpus/parser.py:393`) preservando el `.`, eso demuestra que el `doc_id` con el que
se parseó era el correcto: **la ejecución que produjo el corpus no pasó por
`verify_parser.py`** (que además habría incluido el huérfano, y no está).

Conclusión honesta: la primera versión de este informe afirmaba que el string mal escrito «acaba
en `ParsedDocument.doc_id` y, de ahí, en la cita». Sobre los datos reales **eso no ha ocurrido**.
Lo que hay son **dos rutas de identidad conviviendo** —una correcta, basada en el inventario, y
otra lossy, basada en el nombre de fichero— sin que ninguna esté declarada como la buena. El
riesgo real es que la ingesta de M2 herede la equivocada. Sigue siendo material de Brief 3, con
la severidad intacta y el diagnóstico corregido.

---

## Eje 1 — Correctitud

### C-2 — No se verifica que lo descargado sea realmente un Word · **ALTA** · silenciar

> **Sobre datos reales: no materializado.** Los 28 ficheros de `data/raw/` pasan la
> verificación de magic bytes. El agujero existe; no se ha cobrado ninguna pieza.
> La recomendación se mantiene como endurecimiento, no como incendio.

**Dónde:** `corpus/downloader.py:121-147`, `download_file`.

**Qué hace hoy:** hace `GET` con `stream=True`, comprueba `raise_for_status()` y vuelca cada chunk
a `.tmp`. Si el servidor responde 200, **lo que venga** se renombra a `.docx` y se cuenta como
descarga correcta. No se mira el `Content-Type`, ni los magic bytes (`PK\x03\x04` para DOCX,
`\xD0\xCF\x11\xE0` para DOC legacy), ni el tamaño.

**Por qué es problema:** no es un riesgo hipotético en este sitio concreto. El propio docstring de
`scraper.py` lo dice: *«The site requires registration to download documents»*. Una sesión sin
cookie válida, un cambio de política del portal o un enlace caducado se manifiestan típicamente
como **redirección 302 a una página de login que responde 200 con HTML** — y `requests` sigue
redirecciones por defecto. Resultado: `ECSS-E-ST-40C_Rev_1.docx` que en realidad es la portada de
registro de ecss.nl, contabilizado como `Downloaded: 1`. El fallo aflora mucho después, en el
parser, con un mensaje que no apunta aquí; o peor, no aflora y produce un documento vacío.

Además, `resp.iter_content()` no compara lo recibido contra `Content-Length`: una respuesta
truncada normalmente levanta `ChunkedEncodingError`, pero no hay red de seguridad si no.

**Recomendación:** verificar el `.tmp` **antes** del `rename` — magic bytes + tamaño mínimo
plausible (~20 KB; ningún estándar ECSS real baja de ahí). Si falla, borrar el `.tmp` y contar el
documento como fallido. Es la mitad de la solución al problema entero: convierte tres modos de
fallo silencioso en un `Failed: 1` visible. Marcado con `# TODO` en el código; es cambio de
comportamiento (documentos que hoy «pasan» empezarían a fallar), así que se decide en Claude Chat.

---

### C-1 — La vigencia se delega por completo a `source_type` · **ALTA** · saltar+log (parcial)

**Dónde:** `corpus/downloader.py:93-96`, `build_download_plan`.

**Qué hace hoy:** una sola línea — `if row["source_type"] == "superseded_standards": continue`.
Todo lo demás se descarga. Es decir: **qué revisión es la vigente lo decide ecss.nl**, según en qué
índice apareció el documento cuando se scrapeó. El downloader no compara fechas, ni números de
revisión, ni `domain_code`.

**Por qué es (a medias) correcto:** para el caso que motiva el brief, funciona. `ECSS-E-ST-40C Rev.1`
vive en `active_standards` con `docx_url`; `ECSS-E-ST-40C` vive en `superseded_standards` con las
columnas de URL vacías. La primera se descarga, la segunda se descarta antes siquiera de mirar la
URL. Hay test que lo fija: `test_active_revision_wins_over_its_superseded_predecessor`.

**Por qué sigue siendo un problema:** el invariante «entre los activos hay como mucho un documento
por `domain_code`» es **load-bearing y no está declarado ni comprobado en ningún sitio**. Si
ecss.nl lista transitoriamente dos revisiones como activas (pasa durante una publicación), el
downloader se lleva las dos, el parser las ingiere las dos, y el corpus contiene dos versiones del
mismo estándar respondiendo a la misma pregunta. Nadie se entera:
`test_two_active_rows_of_the_same_domain_are_both_downloaded` fija ese comportamiento de hoy.

Y hay una trampa para quien intente arreglarlo a lo bruto: **`domain_code` no es una clave de
documento**. `ECSS-Q-HB-80-02 Part 1A` y `Part 2A` comparten `domain_code` `Q-HB-80-02`
legítimamente — son partes distintas, no revisiones competidoras. Una regla de «un activo por
domain_code» descartaría una parte real del corpus.

**Recomendación:** **detectar, no seleccionar**. Un warning al construir el plan cuando dos filas
activas comparten `domain_code` y ninguna lleva `Part` es barato, no puede descartar nada por
error, y convierte un fallo silencioso en uno ruidoso. Implementar selección automática de
vigencia (por fecha o por número de revisión) es una decisión de diseño, no una corrección:
marcado con `# TODO: pendiente de decisión en Claude Chat` en `build_download_plan`.

El script `scripts/audit_raw_corpus.py` (check 4) comprueba este invariante contra el inventario
real y distingue partes de revisiones.

---

### C-3 — El enlace de descarga se elige sin validar identidad · **ALTA** · silenciar

**Dónde:** `corpus/scraper.py:208-232`, `select_download_links` (extraída en esta
auditoría de `scrape_download_links`).

**Qué hace hoy:** recorre los `<a href>` de la página de detalle, se queda con el **primer**
enlace bajo `wp-content/uploads` cuya extensión sea `.doc`/`.docx`, y con el primer `.pdf`. El
docstring original afirmaba: *«The first PDF link is always the main document; supporting files
appear after it»*. Esa afirmación **no está verificada** y se aplicaba tácitamente también a la
rama DOCX, donde nunca se declaró.

**Por qué es problema:** las páginas de ECSS listan anexos, DRDs y ficheros de apoyo junto al
documento principal. Si en alguna página el orden no es el asumido, el inventario guarda la URL de
un anexo, el downloader lo descarga, y lo guarda **con el nombre del documento principal** — porque
el stem sale del `ecss_number`, no de la URL (ver `derive_dest_path`). El resultado es un fichero
llamado `ECSS-E-ST-40C_Rev_1.docx` cuyo contenido es un anexo. Nada en el pipeline lo detecta: el
parser encontrará secciones y requisitos perfectamente válidos, y el sistema citará cláusulas
inexistentes en el documento que dice citar. Es el peor fallo posible de este brief y no deja
traza.

Que el stem venga de la identidad y no de la URL es **la decisión correcta** (evita nombres
arbitrarios del servidor), pero tiene este precio: rompe la única señal que quedaba para detectar
una discrepancia.

**Recomendación:** comparar el basename de la URL elegida contra el `ecss_number` normalizado y
avisar cuando no se parezcan (los ficheros de ecss.nl sí contienen el número del estándar en el
nombre). No basta como validación fuerte, pero convierte «silenciar» en «saltar+log» por el coste
de una función pura. Se propone, no se aplica: cambia lo que el scraper escribe en el inventario.
`test_first_word_link_wins_even_if_it_is_an_annex` fija el comportamiento actual.

---

### C-4 — El nombre de fichero no es un identificador reversible · **ALTA** · silenciar

> **Sobre datos reales: latente.** 14 de 27 stems sufren el round-trip, pero los 14 JSON de
> `data/processed/` conservan `Rev.1` correctamente. Ver «C-4 — Corrección» arriba: el
> diagnóstico de abajo describe la ruta lossy, que existe pero no es la que produjo el corpus.

**Dónde:** `corpus/downloader.py:33-41`, `sanitize_filename`; consumido por
`corpus/verify_parser.py:150` y `corpus/diagnose_doc.py:76`.

**Qué hace hoy:** `sanitize_filename` mapea ` `, `.`, `/`, `(`, `)` **todos al mismo carácter**
`_` y colapsa repeticiones. Es una función no inyectiva y no invertible.
`ECSS-E-ST-40C Rev.1` → `ECSS-E-ST-40C_Rev_1`.

Las dos herramientas que reconstruyen el `doc_id` desde el disco hacen
`path.stem.replace("_", " ")`, lo que devuelve **`ECSS-E-ST-40C Rev 1`** — sin el punto. Ese string
no existe en el inventario, no es el nombre oficial del documento, y es el que entra en
`parse_document(path, doc_id)` → `ParsedDocument.doc_id` → JSON de `data/processed/`.

**Alcance real del daño (medido, no supuesto):** el grafo **no** se ve afectado.
`canonical_doc_id` usa `\s*Rev\.?\s*\d+$` (`corpus/graph.py:69`), que absorbe tanto `Rev.1`
como `Rev 1`, así que los nodos salen igual. El daño está en **la cita**: CLAUDE.md establece que
el contrato de cita conserva el nombre de documento completo *con* revisión. Hoy ese nombre nace
mal escrito en el primer eslabón del pipeline.

Colisiones: además del punto, `ECSS-E-ST-40C (Rev 1)` y `ECSS-E-ST-40C Rev.1` colapsan al mismo
stem. En el corpus actual no ocurre, pero la función no lo impide.

**Recomendación:** dejar de reconstruir la identidad desde el nombre de fichero. El `ecss_number`
canónico está en el inventario y debería viajar explícitamente hasta el parser (es justo lo que un
manifiesto de corpus resuelve — Brief 4). **No se toca aquí**: `verify_parser.py` y
`diagnose_doc.py` son tooling del parser, fuera del alcance de este brief, y la identidad de cita
es Brief 3. El check 5 de `scripts/audit_raw_corpus.py` mide cuántos ficheros reales sufren el
mismatch.

---

### C-5 — «Ya existe» se equipara a «está bien» · **MEDIA** · silenciar

**Dónde:** `corpus/downloader.py:105-108`.

**Qué hace hoy:** `if dest_path.exists(): skipped_exists`. La única condición es que el inodo
exista. Un fichero de 0 bytes, un HTML de login guardado en una ejecución anterior (C-2), o un
DOCX truncado, cuentan todos como «ya descargado».

**Por qué es problema:** el fichero de staging `.tmp` protege bien contra dejar un fichero parcial
*en esta* ejecución — está bien resuelto y merece decirse. Pero **no protege contra el contenido
equivocado**, y una vez que un fichero envenenado aterriza en `data/raw/`, es permanente: todas
las ejecuciones futuras lo saltan, y sin `--force` la única forma de recuperarse es borrarlo a
mano. El daño de C-2 pasa de ser un incidente a ser un estado.

**Recomendación:** la misma verificación de C-2, aplicada también a la rama «ya existe» (magic
bytes + tamaño), y una flag `--force`. Ambas cosas se proponen, no se aplican.

---

### C-6 — El resumen no dice *qué* falló, y el proceso sale con 0 · **MEDIA** · silenciar

**Dónde:** `corpus/downloader.py:154-174`, `print_summary`;
`corpus/downloader.py:181-245`, `run` / `main`.

**Qué hace hoy:** el resumen imprime **contadores** — `Downloaded: 11`, `Failed: 3` — sin la lista
de qué documentos son los tres. Esa información existe (`log.error` la escribió al pasar), pero
está dispersa por el log y no sobrevive a la sesión. Y `run()` devuelve `None`: el proceso termina
con código 0 **aunque fallen los 14**.

**Por qué es problema:** el producto de este pipeline no es «un número de descargas», es «el
corpus». La pregunta operativa siempre es *qué falta*, y hoy hay que reconstruirla leyendo el
scrollback. El exit code 0 con 14 fallos es, literalmente, un pipeline que reporta éxito habiendo
producido nada; cualquier automatización que lo llame se lo cree.

**Recomendación:** el resumen debe llevar las listas (`failed`, `skipped_no_docx`) con su URL y
motivo, y `main()` debe devolver un código distinto de cero si hubo fallos. Esto es **el insumo
directo del manifiesto de corpus (Brief 4)**: un registro estructurado por documento —
`ecss_number`, URL de origen, timestamp, tamaño, resultado — cubre a la vez C-6, la procedencia
(C-13) y la reconciliación de O-1. No se aplica aquí porque la forma de ese registro es
precisamente lo que decide el Brief 4.

---

### C-7 — Un `OSError` aborta el batch entero · **MEDIA** · abortar

**Dónde:** `corpus/downloader.py:216-223`.

**Qué hace hoy:** el bucle captura `except requests.RequestException`. Los fallos de escritura
(disco lleno, permisos, un `rename` que falla en Windows porque el destino apareció entre medias)
levantan `OSError`, que **no** se captura: la excepción sale de `run()`, mata el proceso y
**el resumen nunca se imprime**. Se pierde la traza de los documentos que sí se habían descargado
en esa ejecución.

**Por qué es problema:** la inconsistencia. Los fallos de red degradan educadamente y los de disco
son fatales, sin que nada justifique la asimetría. Un disco lleno en el documento 12 de 14 tira
por tierra el informe de los 11 anteriores.

**Recomendación:** capturar `Exception` por tarea (registrando el tipo) o al menos añadir `OSError`,
para que la política sea uniformemente «saltar+log» y el resumen siempre se emita. Cambio pequeño
pero es cambio de comportamiento: se propone.

---

### C-8 — La dedup del scraper depende del orden de `SOURCES` y no deja traza · **MEDIA** · silenciar

**Dónde:** `corpus/scraper.py:364-380`, `scrape_all`.

**Qué hace hoy:** `seen_numbers` descarta cualquier documento cuyo `ecss_number` ya se haya visto.
Como `SOURCES` es un dict y Python preserva el orden de inserción, hoy se recorre
`active_standards` → `active_handbooks` → `superseded_standards`, y por tanto la versión activa
gana. **El orden de un literal de diccionario es lo único que garantiza que un documento activo no
sea descartado en favor de su gemelo superseded.**

**Por qué es problema:** es un invariante crítico que vive en una propiedad accidental del código,
sin comentario que lo señale y sin test que lo proteja. Reordenar `SOURCES` alfabéticamente —el
tipo de cambio inocente que cualquiera haría— haría que `superseded_standards` se procesara primero
y un documento activo desapareciera del inventario. Y como la dedup **no registra nada al
descartar**, la desaparición sería invisible: simplemente habría un documento menos.

**Recomendación:** hacer la prioridad explícita (ordenar las fuentes por precedencia declarada en
vez de confiar en el literal) y loguear en `debug` cada descarte con qué fuente ganó. Se propone.

---

### C-9 — Acceso crudo a columnas del CSV · **BAJA** · abortar (con mal mensaje)

**Dónde:** `corpus/downloader.py:95-99`.

`row["source_type"]`, `row["ecss_number"]`, `row["docx_url"]` sin `.get()`. Si el esquema del
inventario cambia, o si alguien abre el CSV en Excel y lo guarda (Excel añade un BOM UTF-8, que
convierte la primera cabecera en `﻿ecss_number`), el fallo es un `KeyError` crudo con
traceback. Fail-fast es la política correcta aquí — un inventario ilegible **debe** parar el
pipeline — pero el mensaje no dice qué se esperaba.

**Recomendación:** validar las columnas requeridas una vez al abrir el fichero y fallar con un
mensaje explícito; leer con `encoding="utf-8-sig"` (inocuo para ficheros sin BOM). No aplicado:
es robustez ante un escenario no observado, y la regla de sencillez del proyecto pide no añadir
manejo de errores especulativo.

---

### C-10 — `.tmp` huérfano, y colisión de stem `.doc`/`.docx` · **BAJA** · silenciar

> **Sobre datos reales:** cero `.tmp` huérfanos. Pero la colisión de stem que abajo se daba
> por irrelevante **está presente**: `ECSS-E-ST-70-01C16April2010` existe como `.doc` y como
> `.docx`. Ver C-16 — se multiplicará por 12 al convertir el resto con LibreOffice.

**Dónde:** `corpus/downloader.py:138-146`.

El `try/except` limpia el `.tmp` ante cualquier excepción, pero no ante un `SIGINT`/kill duro ni un
corte de luz. El resto queda en `data/raw/`, no molesta a nadie (no coincide con ningún
`dest_path`) y nadie lo limpia ni lo menciona. El check 2 de `scripts/audit_raw_corpus.py` los
detecta. Nota menor adicional: `dest_path.with_suffix(".tmp")` colapsa `X.doc` y `X.docx` en el
mismo `X.tmp` — irrelevante hoy porque la descarga es secuencial y hay una fila por número.

---

### C-11 — La respuesta HTTP no se cierra · **BAJA**

`corpus/downloader.py:132`: con `stream=True`, si `raise_for_status()` levanta, la
conexión no se devuelve al pool. Sobre 14 documentos no tiene consecuencia práctica.
`with session.get(...) as resp:` lo resuelve en una línea.

---

### C-12 — `production_status` se scrapea y no se usa · **INFO**

`scrape_production_statuses` (`corpus/scraper.py:320`) recorre la tabla de documentos en
producción y guarda el estado en el inventario. El downloader **nunca lo lee**. No es un defecto —
es una señal disponible y sin explotar: sabe qué documentos activos tienen una revisión en curso, o
sea, cuáles van a quedar obsoletos pronto. Entra bien en el manifiesto del Brief 4.

Detalle menor: el emparejamiento es por `startswith` sobre un dict, así que si varias claves
encajan gana la primera en orden de inserción de la tabla. Sin efecto hoy.

---

### C-13 — `scraped_at` es naive y usa una API deprecada · **BAJA**

`corpus/scraper.py:275`: `datetime.utcnow().isoformat()` produce
`2026-07-28T10:31:00.123456` — sin marca de zona, indistinguible de una hora local, y `utcnow()`
está deprecado desde Python 3.12. Para un campo cuyo único propósito es la **procedencia** eso es
un defecto pequeño pero real. `datetime.now(timezone.utc).isoformat()` lo arregla. No aplicado:
cambia el valor escrito en el CSV, y el formato de los artefactos de procedencia lo fija el Brief 4.

---

### C-14 — El selector `ol li a` es demasiado ancho · **BAJA** · saltar+log

`corpus/scraper.py:278`: `soup.select("ol li a")` captura **cualquier** enlace dentro de
cualquier lista ordenada de la página, incluidos menús o navegación. Un enlace espurio produce una
fila con `ecss_number` basura (el fallback de `parse_number_from_text` devuelve el texto entero) y
casi siempre `software_relevance` vacío, con lo que se cae del inventario software. El fallo es
autolimitado; si colase, aparecería como `skipped_no_docx` con un warning. Aceptable.

---

## Eje 2 — Optimización

### O-1 — No hay poda: las revisiones viejas se quedan para siempre · **MEDIA**

> **Sobre datos reales: confirmado**, aunque con residuo manual en vez de una revisión
> derogada — un huérfano (2 ficheros), ver C-16. El mecanismo que falla es el mismo: nadie
> pregunta «¿debería existir esto?».

Cuando ECSS publica `Rev.2` de un estándar, el inventario cambia y el downloader se trae
`..._Rev_2.docx`. **El `..._Rev_1.docx` anterior sigue en `data/raw/`**, sin fila activa que lo
respalde, y nada lo señala. `verify_parser.py` con su argumento por defecto (`data/raw`) parsearía
**los dos**, y el corpus contendría la revisión derogada junto a la vigente, ambas con pinta de
buenas.

Es la contrapartida exacta de la idempotencia: como «existe ⇒ correcto», nunca se pregunta
«¿debería existir?». La respuesta no es borrar automáticamente (destruir descargas es
irreversible), sino **tener una lista declarada de qué debe estar** y reconciliar contra ella. Eso
es el manifiesto del Brief 4. El check 3 de `scripts/audit_raw_corpus.py` hace hoy esa
reconciliación y marca los huérfanos.

### O-2 — El scraper visita la página de detalle de todos los documentos activos · **MEDIA**

`corpus/scraper.py:296-298`: `scrape_index_page` hace una petición extra **por cada
documento del índice**, con 1 s de espera, para extraer los enlaces de descarga. El filtro por
relevancia software se aplica mucho después, en `main()`
(`corpus/scraper.py:460`). Sobre un índice de ~150 estándares activos, eso son ~150
peticiones y ~3 minutos de espera para acabar quedándose con ~15 documentos: **~10× el trabajo de
red necesario**, y ~10× la carga sobre un servidor público al que el propio código intenta tratar
con educación.

**Coste/beneficio:** el arreglo es mover el filtro antes del fetch de detalle — barato. Pero cambia
el contenido de `ecss_inventory_full.csv`, que dejaría de tener `pdf_url`/`docx_url` para los
documentos no-software. Si ese CSV es solo un censo, la pérdida es nula; si se usa para explorar
ampliaciones futuras del corpus, no. Como el scraper se ejecuta un puñado de veces al año, el
beneficio real es modesto: **se propone, no se aplica**, y se decide junto con el alcance del
corpus.

### O-3 — Descarga secuencial con 2 s de espera · **correcta, no tocar**

14 documentos secuenciales ≈ 30 s de espera más el tiempo de transferencia. Paralelizar ahorraría
segundos sobre una tarea que se ejecuta un par de veces al año, a cambio de golpear un servidor
público gratuito y de complicar el manejo de errores. **La recomendación es explícitamente no
hacerlo.**

### O-4 — Idempotencia · **correcta, con un hueco**

Saltar lo ya descargado es el comportamiento adecuado. Falta `--force` para re-descargar sin tener
que borrar ficheros a mano (hoy es la única vía de recuperación ante un fichero corrupto, y
requiere saber que está corrupto — ver C-5). Cuesta cinco líneas; se propone junto con la
verificación de integridad, porque juntas forman una sola historia.

### O-5 — El scraper no cachea el índice · **BAJA, aceptable**

Cada ejecución re-consulta la página de estados, los tres índices y todas las páginas de detalle.
No hay cache ni condicional HTTP (`If-Modified-Since`/ETag). Para una fuente que cambia varias
veces al año y un comando que se lanza a mano, cachear no paga su complejidad. La frecuencia
razonable de re-scrapeo es **cuando se sospeche una publicación nueva**, no periódica. La
comparación útil no es «¿ha cambiado el HTML?» sino «¿ha cambiado el inventario?», y eso se ve
mejor versionando el CSV — otra vez, Brief 4.

---

## Eje 3 — Testeabilidad y medibilidad

### T-1 — Separación red/lógica: **ya era buena**

En el downloader, toda la red vive en `download_file`; `build_download_plan` y `sanitize_filename`
son lógica pura (o casi: leen CSV y consultan `exists()`, ambas cosas testeables con `tmp_path`).
Por eso `tests/corpus/test_downloader.py` ya cubría el plan sin mocks de red. El scraper estaba
peor: la selección de enlaces estaba enredada con el `fetch_html`.

### T-2 — Extracciones aplicadas en esta auditoría

Dos extracciones mecánicas, sin cambio de comportamiento (el brief las autoriza explícitamente):

- **`derive_dest_path(ecss_number, docx_url, raw_dir)`** (`corpus/downloader.py:62-77`) —
  sacada del cuerpo de `build_download_plan`. Ahora la regla «el stem viene de la identidad, la
  extensión viene de la URL» es una función con nombre, con docstring que dice por qué, y con
  tests directos.
- **`select_download_links(hrefs)`** (`corpus/scraper.py:208`) — sacada de
  `scrape_download_links`, que se queda con la parte de red. La regla de selección (C-3) es ahora
  testeable sin tocar la red.

Tests añadidos (`uv run pytest` en verde, 259 pasados):
`TestDeriveDestPath` (5 casos), `TestSelectDownloadLinks` (7 casos), y dos casos de vigencia en
`TestBuildDownloadPlan` — incluida la fixture que pide el brief: `ECSS-E-ST-40C Rev.1` activo con
URL + `ECSS-E-ST-40C` superseded sin URL.

Dos de esos tests fijan comportamiento **que la auditoría considera incorrecto**
(`test_two_active_rows_of_the_same_domain_are_both_downloaded`,
`test_first_word_link_wins_even_if_it_is_an_annex`). Están puestos a propósito y así documentados:
sirven de ancla para que, cuando se decida el arreglo, el cambio de contrato sea visible en el
diff en vez de silencioso.

### T-3 — No hay traza estructurada, solo un banner · **MEDIA**

`print_summary` (`corpus/downloader.py:154`) compone un banner ASCII a base de
`log.info` con `\n` y `=` embebidos. Se lee bien en una terminal y **es inservible como dato**: no
se puede consultar, ni diferenciar entre ejecuciones, ni serializar. Si el logging del proyecto
pasara a `JSONRenderer`, produciría una decena de registros JSON con arte ASCII dentro.

Lo que hace falta es lo contrario: **un registro por documento** (ecss_number, URL, timestamp,
bytes, resultado) más un agregado. Eso es a la vez la métrica de «¿funcionó?», la procedencia que
falta en C-13 y la base del manifiesto del Brief 4 — por eso no se diseña aquí.

### T-4 — `logging` estándar en vez de `structlog` · **BAJA**

Ambos módulos usan `logging` de la stdlib con `basicConfig` en su `main()`. La convención del
proyecto («`logging` / `structlog`, nunca `print()`») se cumple al pie de la letra, y el pipeline
de corpus es offline: no comparte el `request_id` ni el contexto de `app/observability.py`. No es
un defecto hoy. Se vuelve uno el día que el resumen tenga que ser dato (T-3): ahí conviene alinear
`corpus/` con `structlog` de una vez.

### T-5 — `print()` en producción: **no hay**

Ni `downloader.py` ni `scraper.py` usan `print()`. Sí lo hacen `verify_parser.py` y
`diagnose_doc.py`, que son herramientas CLI de diagnóstico cuyo producto *es* la salida por
pantalla — uso legítimo, y además fuera del alcance de este brief.

**Deuda preexistente detectada de paso, no tocada** (regla de cambios quirúrgicos): `ruff` reporta
6 avisos en ficheros ajenos a esta auditoría — imports sin usar en `corpus/verify_parser.py` y
`tests/corpus/test_parser.py`, y un f-string vacío en `corpus/diagnose_doc.py`. Todos
autocorregibles con `uv run ruff check --fix`.

---

## Estado tras el Brief 1b (endurecimiento)

Brief 1b aplicó los Grupos A–D contra este informe. **Ningún hallazgo clasificado «silenciar»
sigue en silencio**: o pasó a saltar+log/abortar, o está explícitamente diferido con su brief.

**Cerrados en 1b** — C-2 y C-5 (verificación de magic bytes + tamaño mínimo antes de promover
el `.tmp`, y también sobre los ficheros ya presentes, que ahora se re-descargan si no pasan),
C-6 (resumen con listas nominales + exit code ≠ 0), C-7 (`except Exception` por documento, el
resumen se emite siempre), C-8 (`SOURCE_PRIORITY` explícita + traza de los descartes), C-9
(validación de columnas con mensaje explícito + `utf-8-sig`), C-10 (barrido de `.tmp` al
arrancar), C-11 (`with session.get(...)`), C-13 (`datetime.now(timezone.utc)`), C-14 (selector
acotado al contenedor de contenido), T-3/T-4 (`structlog` y resumen estructurado en ambos
módulos), O-4 (`--force`).

**Cerrados como detección, no resolución** — C-1 (warning cuando dos activos comparten
`domain_code` sin ser `Part`; el `# TODO` sobre selección automática de vigencia sigue vivo) y
C-3 (`check_link_identity` avisa si el basename no contiene el número o parece fichero de
apoyo; la elección del enlace no se altera).

**Siguen abiertos, cada uno con su brief** — C-15 (Brief 2), C-4 (Brief 5), C-16 (borrado por
el usuario + arquitectura `raw/` inmutable en Brief 2), C-12 y O-1 (Brief 4), O-2 (atado al
alcance del corpus). O-5 se descarta: no paga su complejidad.

Detalle de las decisiones de implementación:
`RAG-Guide/sessions/S06 - Data Driven IA/S06-decisions-1b-endurecimiento-downloader.md`.

---

## Cambios aplicados en el Brief 1 (auditoría)

| Fichero | Cambio | Tipo |
|---|---|---|
| `corpus/downloader.py` | Extraída `derive_dest_path` de `build_download_plan` | mecánico, sin cambio de comportamiento |
| `corpus/downloader.py` | `# TODO` de decisión en `build_download_plan` (C-1) y `download_file` (C-2) | comentario |
| `corpus/scraper.py` | Extraída `select_download_links` de `scrape_download_links` | mecánico, sin cambio de comportamiento |
| `tests/corpus/test_downloader.py` | `TestDeriveDestPath` + 2 casos de vigencia | tests |
| `tests/corpus/test_scraper_parsing.py` | `TestSelectDownloadLinks` | tests |
| `scripts/audit_raw_corpus.py` | Diagnóstico read-only sobre `data/raw/` + `data/processed/` (6 checks) | nuevo |

**Nada más se ha tocado.** Toda recomendación de las anteriores que cambie comportamiento
(verificación de integridad, `--force`, exit code, resumen estructurado, detección de vigencia,
validación de identidad del enlace, filtro previo del scraper, `timezone.utc`) queda **propuesta y
sin aplicar**, a decidir en Claude Chat con este informe delante.

---

## Guion de verificación manual (UAT)

Lo ejecuta el usuario; CC no lo ha corrido. Requiere red y toca `data/`.

### Paso 0 — Partir de un estado conocido

```powershell
# Fotografía del estado actual, por si hay que volver
Get-ChildItem data\raw | Select-Object Name, Length, LastWriteTime
```

### Paso 1 — Diagnóstico del corpus que ya está en disco · **ejecutado 2026-07-29**

```powershell
uv run python scripts/audit_raw_corpus.py
```

Qué mirar, check por check (entre corchetes, el resultado de la ejecución del 29/07):
1. **Integridad** — todo fichero debe salir como `DOCX(zip)` o `DOC(ole2)`. Cualquier
   `DESCONOCIDO` o `parece HTML` confirmaría C-2 sobre datos reales. *[28/28 OK]*
2. **`.tmp` huérfanos** — deberían ser cero. *[cero]*
3. **Reconciliación** — `FALTA` = documento activo que nunca se descargó (gap invisible).
   `HUÉRFANO` = fichero sin fila activa (O-1, C-16). *[0 FALTA, 1 huérfano]*
4. **Unicidad de vigencia** — sin `REVISIONES EN COMPETENCIA`; las líneas `Part 1A / Part 2A`
   marcadas como OK son correctas. *[sin conflictos]*
5. **Round-trip de identidad** — cada `MISMATCH` es un fichero cuyo `doc_id` se reconstruiría mal
   desde el nombre (C-4). *[14 de 27 — pero `data/processed/` tiene los doc_id correctos]*
6. **Cobertura descargado → parseado** — `NO PARSEADO` = documento que está en `data/raw/` y no
   llegó a `data/processed/`. Es la medida directa de C-15. *[12 `NO PARSEADO`, todos `.doc`
   legacy; `activos con DOCX: 26 | parseados: 14 | cobertura: 53%`]*

### Paso 2 — Confirmar la vigencia de ECSS-E-ST-40C (afirmación verificable del brief)

```powershell
# El activo debe ser Rev.1 de 2025; el de 2009 debe estar como superseded y sin URL
Import-Csv data\inventory\ecss_inventory_software.csv |
  Where-Object { $_.domain_code -eq 'E-ST-40' } |
  Format-Table ecss_number, pub_date, source_type, @{n='has_docx';e={[bool]$_.docx_url}}
```

**Criterio de aceptación:** exactamente una fila con `source_type = active_standards` para
`E-ST-40`, con `ecss_number = ECSS-E-ST-40C Rev.1`, `pub_date` de **abril de 2025** y
`has_docx = True`. Si aparece `ECSS-E-ST-40C` (2009) como activo, o si el activo no tiene DOCX,
parar: el corpus está construido sobre la revisión equivocada.

```powershell
# Y que el fichero en disco es el Rev.1, no el de 2009
Get-ChildItem data\raw\ECSS-E-ST-40C*
```

Debe existir `ECSS-E-ST-40C_Rev_1.docx` y **no** debe existir `ECSS-E-ST-40C.docx`.

### Paso 3 — Dry-run del downloader (sin red de descarga)

```powershell
uv run python -m corpus.downloader --dry-run
```

**Criterio de aceptación:** sobre un `data/raw/` ya poblado, `Would download 0 document(s)` y
`Skipped (exists) : 14`. Si propone descargar algo, es que el inventario contiene un documento que
falta en disco — cruzar con el `FALTA` del paso 1. Revisar también `Skipped (no DOCX)`: cada uno es
un documento activo que el corpus **no tiene** y del que solo avisa esta línea.

### Paso 4 — Descarga en limpio (opcional, ~1 min de red; destructivo)

Solo si se quiere ejercitar el camino completo. Mueve lo actual, no lo borres:

```powershell
Move-Item data\raw data\raw_backup_20260728
uv run python -m corpus.downloader
```

**Criterios de aceptación:**
- El resumen final reporta `Downloaded : 14`, `Failed : 0`.
- `uv run python scripts/audit_raw_corpus.py` pasa el check 1 sobre los 14 ficheros nuevos
  (**este es el paso que realmente confirma o descarta C-2**: si alguno sale como HTML, el
  downloader lleva descargando basura y contándola como éxito).
- Comparar tamaños contra el backup:
  ```powershell
  Compare-Object (Get-ChildItem data\raw_backup_20260728 | Select-Object Name, Length) `
                 (Get-ChildItem data\raw | Select-Object Name, Length) -Property Name, Length
  ```
  Diferencias de tamaño en un mismo nombre significan que ECSS ha republicado el fichero sin
  cambiar el número — anotarlo, es información para el manifiesto del Brief 4.
- Si todo cuadra, borrar el backup; si no, restaurarlo.

### Paso 5 — Interrumpir a mano (opcional, valida el staging `.tmp`)

Lanzar el paso 4 y cortar con `Ctrl+C` a mitad. Después:

```powershell
Get-ChildItem data\raw\*.tmp
```

**Criterio de aceptación:** puede quedar **como mucho un** `.tmp` (el documento en curso, C-10), y
**ningún** `.docx` de tamaño anómalo. Relanzar el downloader debe completar los que faltaban sin
tocar los ya buenos.

### Qué reportar de vuelta a Claude Chat

1. Salida completa de `scripts/audit_raw_corpus.py` (los 5 checks).
2. La tabla del paso 2 (vigencia de E-ST-40).
3. Si se hizo el paso 4: el resumen del downloader y si algún fichero falló la verificación de
   magic bytes.

Con eso se decide qué recomendaciones de este informe se implementan y en qué brief.
