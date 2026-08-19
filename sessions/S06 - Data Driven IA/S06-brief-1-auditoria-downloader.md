# S06 — Brief 1: Auditoría del downloader (y scraper)

> **Tipo:** auditoría por lectura de código + entrega de guion de verificación. **No** es una
> reescritura. **No** ejecutas el downloader ni el scraper (tocan red y `data/raw/`/`data/inventory/`
> — territorio UAT del usuario, ver frontera en `CLAUDE.md`).
>
> **Contexto:** `corpus/` deja de estar congelado en S06. El downloader es el **primer eslabón** del
> pipeline de datos: si descarga el documento equivocado (una revisión superseded en vez de la
> vigente), todo lo que viene detrás — parser, grafo, citas — hereda el error con cara de correcto.
> Por eso se audita antes que nada. Esta auditoría es de **diagnóstico**, no de cambio: primero
> sabemos qué hay, luego (en briefs posteriores o al final de este) se corrige lo que salga.

## Objetivo

Auditar `corpus/downloader.py` y `corpus/scraper.py` contra tres ejes, **en este orden de
prioridad** (si el eje 1 falla, los otros dos son secundarios):

1. **Correctitud** — ¿la lógica hace lo que debe? ¿es robusta ante lo que puede salir mal?
2. **Optimización** — ¿hay trabajo redundante, descargas repetidas, ineficiencias evitables?
3. **Testeabilidad y medibilidad** — ¿se puede testear sin red? ¿deja traza/métricas de lo que hizo?

El entregable es un **informe de auditoría** (`corpus/AUDIT-downloader.md`) + los tests que se puedan
añadir sin red + un **guion de verificación manual** para que el usuario lo ejecute sobre los datos
reales. **No** modificar la lógica del downloader/scraper en este brief salvo lo que se indique
explícitamente abajo (fixes triviales y seguros); los cambios de fondo que la auditoría revele se
proponen en el informe para decidirse en Claude Chat, no se aplican unilateralmente.

---

## Alcance de cada eje

### Eje 1 — Correctitud (prioridad máxima)

Auditar **por lectura** y reportar hallazgos. Preguntas concretas que el informe debe responder, con
referencia a línea/función:

**Vigencia y selección de documentos:**
- ¿El scraper distingue `active_standards` de `superseded_standards`? (el inventario tiene una
  columna `source_type` con esos valores). ¿Cómo decide cuál es la versión vigente cuando un mismo
  `domain_code` aparece varias veces? (ej.: `E-ST-40` aparece como `ECSS-E-ST-40C Rev.1` activo-2025
  y `ECSS-E-ST-40C` superseded-2009).
- ¿El downloader descarga **solo** los `active_standards`, o también arrastra superseded? (los
  superseded del inventario **no tienen `docx_url`/`pdf_url`** — columnas vacías; si el downloader
  no lo maneja, ¿qué hace ante una URL vacía?).
- ¿Qué campo usa el downloader como fuente de URL — `docx_url`, `pdf_url`, o deriva la URL? ¿Prefiere
  DOCX sobre DOC/PDF? (el proyecto parsea DOCX; DOC legacy se convierte con LibreOffice).

**Robustez ante fallos** (cada uno: ¿qué hace el código hoy?, ¿debería?):
- URL que devuelve 404 / 403 / redirect / timeout.
- URL vacía o malformada en el inventario.
- Archivo ya descargado en `data/raw/` (¿re-descarga?, ¿salta?, ¿verifica integridad?).
- Descarga parcial / interrumpida (¿deja un archivo corrupto a medias que el parser luego leerá?).
- Fallo de red a mitad de un batch de 14 (¿aborta todo?, ¿continúa con los demás?, ¿reporta cuáles
  fallaron?).
- Nombre de archivo: ¿cómo se deriva el nombre local desde la URL/metadata? ¿colisiones posibles
  entre documentos? ¿el nombre preserva la revisión (`Rev.1`) o la pierde? *(esto conecta con
  Brief 3 — identidad de cita: si el nombre de archivo ya pierde la revisión, es un síntoma
  temprano del mismo problema).*

**Integridad de lo descargado:**
- ¿Se verifica que lo descargado es realmente un DOCX (no una página HTML de error servida con 200)?
- ¿Se registra qué se descargó, desde qué URL, cuándo? (procedencia — enlaza con el manifiesto de
  corpus, Brief 4).

**Política de fallo (declararla explícitamente por cada modo de fallo):** para cada fallo detectado
arriba, clasificar la respuesta actual del código como una de: **abortar** (fail-fast), **saltar y
continuar** (degradar, con log), o **silenciar** (el pecado — fallo que no deja traza). Marcar todo
lo que hoy sea "silenciar" como hallazgo prioritario: en un pipeline de datos, un documento que no
se descargó y de cuya ausencia nadie se entera es exactamente el "gap invisible" que produce un
corpus que miente por omisión.

### Eje 2 — Optimización

- **Descargas redundantes:** ¿re-descarga documentos ya presentes e íntegros? Debería poder saltar
  lo ya descargado (idempotencia) salvo `--force` explícito.
- **Concurrencia:** ¿descarga secuencial o en paralelo? Para 14 documentos secuencial es aceptable
  (no sobre-optimizar); reportar si hay una razón para paralelizar, no implementarla sin decisión.
- **Llamadas de red repetidas al scraper:** ¿el scraper re-consulta ecss.nl cada vez o cachea el
  índice? ¿Cada cuánto se re-scrapea razonablemente?
- **Regla de sencillez del proyecto:** no proponer optimizaciones que no paguen su complejidad. Para
  un corpus de 14 documentos públicos que se revisan cada años, la idempotencia importa; el paralelismo
  probablemente no. Reportar coste/beneficio, no optimizar por defecto.

### Eje 3 — Testeabilidad y medibilidad

- **Separación red / lógica:** ¿la lógica de *qué descargar y cómo nombrarlo* está separada del
  *acto de descargar* (I/O de red)? Si están entrelazadas, no se puede testear sin red. Proponer (y,
  si es trivial y seguro, aplicar) la extracción de la lógica pura a funciones testeables: selección
  de versión vigente, derivación de nombre de archivo, parseo de una fila de inventario, decisión
  descargar-vs-saltar. **Estas funciones puras SÍ se testean en este brief** (mockeadas, sin red).
- **Traza / métricas:** ¿el downloader deja un registro de qué hizo — descargados, saltados, fallidos,
  con URL y timestamp? Si no, proponer un resumen estructurado al final del run (cuántos intentados,
  cuántos OK, cuántos fallidos y por qué). Esto es el insumo del manifiesto de corpus (Brief 4) y la
  medida de "¿funcionó?".
- **`structlog`, no `print`:** verificar que usa el logger estructurado del proyecto (convención de
  código), no `print`.

---

## Qué entrega CC

1. **`corpus/AUDIT-downloader.md`** — informe de auditoría estructurado por los tres ejes. Cada
   hallazgo: ubicación (archivo/función/línea), qué hace hoy, por qué es problema (o por qué está
   bien), y recomendación. Los hallazgos del Eje 1 con su política de fallo clasificada
   (abortar/saltar/silenciar). Ordenar por severidad, no por orden de aparición en el código.

2. **Tests de la lógica pura** (`tests/corpus/test_downloader.py`, o extender el existente) — solo de
   las funciones que sean testeables sin red: selección de versión vigente desde una fila de
   inventario mockeada, derivación de nombre de archivo, decisión descargar-vs-saltar. **Sin red, sin
   `data/raw/`, sin `data/inventory/` real** — fixtures con filas de inventario sintéticas (incluir
   como fixture al menos un caso con `Rev.1` activo + su superseded sin URL, para ejercitar la
   selección de vigencia). `uv run pytest` debe pasar.

3. **Fixes triviales y seguros aplicados** (solo estos; lo demás se propone, no se aplica):
   `print`→`structlog` si lo hubiera; extracción de una función pura si es mecánica y no cambia
   comportamiento; un `# TODO: pendiente de decisión en Claude Chat` donde la auditoría encuentre algo
   que requiera decisión de diseño.

4. **Guion de verificación manual (UAT)** — pasos que el *usuario* ejecuta sobre datos reales, que CC
   no puede correr: p. ej. "ejecutar el downloader en limpio y confirmar que `data/raw/` contiene los
   14 DOCX vigentes, que `ECSS-E-ST-40C` es el `Rev.1` de abril 2025 y no el de 2009, que ningún
   archivo es un HTML de error, y que el resumen final reporta 14/14 OK". CC entrega el guion; **no lo
   ejecuta**.

## Qué NO hace CC en este brief

- No ejecuta el downloader ni el scraper (red + `data/raw/`).
- No reescribe la arquitectura del downloader — audita y propone; los cambios de fondo se deciden en
  Claude Chat con el informe delante.
- No toca el parser, el grafo, ni `Citation` (son Briefs 2/3).
- No decide la forma del manifiesto de corpus (Brief 4), aunque su informe alimente esa decisión.

## Criterio de "hecho"

- `corpus/AUDIT-downloader.md` existe, cubre los tres ejes, con hallazgos ubicados y priorizados por
  severidad, y cada hallazgo del Eje 1 con su política de fallo clasificada.
- Los tests de lógica pura pasan (`uv run pytest`), sin red ni datos reales.
- El guion de verificación manual está entregado, con al menos una afirmación verificable sobre
  vigencia de revisión (40C = Rev.1 2025, no 2009).
- Ningún cambio de fondo aplicado sin decisión; lo que requiere decisión está marcado con `# TODO` y
  listado en el informe.
