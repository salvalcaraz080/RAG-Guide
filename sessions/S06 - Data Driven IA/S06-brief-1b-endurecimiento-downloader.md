# S06 — Brief 1b: Endurecimiento del downloader y el scraper

> **Tipo:** reparación dirigida. La especificación es `corpus/AUDIT-downloader.md` — este brief
> **aplica** las recomendaciones que allí quedaron *propuestas y sin aplicar*. **No re-audita, no
> explora, no rediseña.** Cada fix ya tiene ubicación y recomendación en el informe; aquí se
> ejecutan.
>
> **Contexto:** la auditoría (Brief 1) encontró que el downloader funciona pero **confía en el
> servidor sin verificar** y **falla en silencio** en varios puntos. Ninguno se ha cobrado una pieza
> todavía (los 28 ficheros en disco están sanos), pero son silencios que se pagan tres capas más
> abajo. Los corregimos **todos ahora** — diferirlos es olvidarlos.
>
> **Frontera CC (recordatorio):** CC escribe código + tests mockeados (`uv run pytest`, sin red, sin
> `data/raw/`). No ejecuta el downloader ni el scraper sobre datos reales — eso es UAT del usuario.

## Principio rector de este brief

Cada fix persigue una sola cosa: **convertir un fallo silencioso en uno ruidoso.** No se trata de
impedir que las cosas fallen, sino de que **cuando fallen, quede traza y el pipeline lo diga**. La
regla de clasificación del informe (abortar / saltar+log / silenciar) es el criterio: todo lo que
hoy es «silenciar» debe pasar a «saltar+log» o «abortar», nunca quedarse en silencio.

Un fix que cambia comportamiento (un documento que hoy «pasa» empezaría a «fallar») **debe llevar un
test de caracterización que afirme el comportamiento NUEVO deseado**, no el viejo. Donde el informe
ya dejó un test fijando el comportamiento incorrecto de hoy (`test_two_active_rows_...`,
`test_first_word_link_wins_...`), ese test se **actualiza** para afirmar el comportamiento correcto —
su diff en el PR *es* la evidencia del cambio de contrato.

---

## Grupo A — Verificación de integridad de la descarga (C-2, C-5, O-4)

El núcleo del brief. Hoy: si el servidor responde 200, lo que venga se renombra a `.docx` y se cuenta
como éxito. Un redirect a la página de login de ecss.nl (que el propio scraper documenta como
requisito de registro) se guardaría como documento válido.

- **C-2 — verificar el `.tmp` antes del `rename`** ([downloader.py:121-147], `download_file`):
  comprobar **magic bytes** (`PK\x03\x04` para DOCX/zip, `\xD0\xCF\x11\xE0` para DOC/OLE2) **y tamaño
  mínimo plausible** (~20 KB — ningún estándar ECSS real baja de ahí; usar una constante nombrada,
  p.ej. `MIN_PLAUSIBLE_DOC_BYTES = 20_000`, con comentario del porqué). Si falla: borrar el `.tmp`,
  contar el documento como **fallido** (no descargado), log con el motivo. Un HTML de login tiene
  firma `<` / `PK`-ausente y suele pesar poco → cae por ambas condiciones.
- **C-5 — aplicar la MISMA verificación a la rama «ya existe»** ([downloader.py:105-108]): hoy
  `if dest_path.exists()` equipara existir con estar-bien. Un fichero envenenado en una ejecución
  previa se salta para siempre. Verificar magic bytes + tamaño también aquí; si un fichero existente
  no pasa, tratarlo como si no estuviera (re-descargar) o marcarlo fallido — **decisión de detalle de
  CC**, documentada en el código, coherente con `--force`.
- **O-4 — flag `--force`** para re-descargar sin borrar a mano. Cinco líneas; va con este grupo
  porque integridad + `--force` son una sola historia (hoy la única recuperación ante un fichero
  corrupto es borrarlo manualmente, y requiere *saber* que está corrupto).

**Política:** los tres pasan de «silenciar» a «saltar+log». Cambio de comportamiento (documentos que
hoy pasan podrían empezar a fallar) → tests de caracterización con el resultado nuevo: un `.tmp` con
bytes de HTML → documento fallido; un `.tmp` de 2 KB → fallido; un DOCX válido → OK. Mockeados, sin
red (fixtures de bytes en memoria).

---

## Grupo B — El resumen como dato, no como banner (C-6, C-7, T-3, T-4)

Hoy el resumen es arte ASCII por `log.info`, imprime contadores sin decir *qué* falló, y el proceso
sale con código 0 aunque fallen los 14.

- **C-6 — el resumen lleva las listas**, no solo contadores: `failed` y `skipped_no_docx` con
  `ecss_number`, URL y motivo por entrada. Y **`main()` devuelve exit code ≠ 0 si hubo algún fallo**
  (un pipeline que reporta éxito habiendo producido nada es una trampa para cualquier automatización).
- **C-7 — capturar `OSError` (o `Exception`) por documento**, no solo `requests.RequestException`.
  Hoy un fallo de disco (lleno, permisos, `rename` que falla en Windows) mata el proceso y **el
  resumen nunca se imprime** — se pierde la traza de los que sí se descargaron. La política debe ser
  uniformemente «saltar+log» para que el resumen **siempre** se emita.
- **T-4 — migrar `downloader.py` y `scraper.py` de `logging` stdlib a `structlog`** (convención del
  proyecto). El pipeline de corpus es offline y no comparte el `request_id` de `app/observability.py`
  — no se le pide eso; solo usar el mismo logger estructurado, de modo que el día que el resumen sea
  dato (Brief 4) ya salga en JSON y no como banner ASCII embebido en registros.

**Sobre T-3 (registro estructurado por documento):** el informe lo señala como insumo del manifiesto
(Brief 4). Aquí **no** se diseña la forma final del registro de procedencia — eso es Brief 4. Lo que
sí entra: que el resumen sea *estructurado y consultable* (listas + exit code), no un banner. La
frontera: **1b hace el resumen legible-como-dato; Brief 4 decide el esquema del manifiesto que lo
persiste.** No adelantar el esquema del manifiesto aquí.

---

## Grupo C — Detección de anomalías silenciosas (C-1, C-3, C-8)

Tres invariantes «load-bearing» que hoy viven en propiedades accidentales del código, sin comprobación.
La consigna del informe es **detectar, no seleccionar**: un warning barato que no puede descartar nada
por error, convirtiendo un fallo silencioso en uno ruidoso. **No** implementar selección/resolución
automática — eso sería decisión de diseño, no corrección.

- **C-1 — vigencia en competencia** ([downloader.py:93-96], `build_download_plan`): warning cuando
  **dos filas activas comparten `domain_code` y ninguna lleva `Part`** (el caso `Q-HB-80-02 Part 1A`/
  `Part 2A` es legítimo y NO debe avisar — `domain_code` no es clave de documento). No descartar
  ninguna; solo avisar. El `# TODO: pendiente de decisión en Claude Chat` sobre *selección* automática
  de vigencia se mantiene.
- **C-3 — identidad del enlace de descarga** ([scraper.py], `select_download_links`, ya extraída en
  Brief 1): comparar el basename de la URL elegida contra el `ecss_number` normalizado y **avisar si no
  se parecen** (los ficheros de ecss.nl contienen el número del estándar en el nombre). No es
  validación fuerte, pero convierte «silenciar» en «saltar+log» el caso de bajar un anexo con el
  nombre del documento principal — el peor fallo posible, hoy sin traza. Actualizar
  `test_first_word_link_wins_even_if_it_is_an_annex` para reflejar que ahora **avisa**.
- **C-8 — prioridad de dedup explícita** ([scraper.py:364-380], `scrape_all`): hoy el que gane un
  documento activo sobre su gemelo superseded depende del **orden de inserción del literal `SOURCES`**.
  Hacer la precedencia explícita (ordenar las fuentes por prioridad declarada, no confiar en el orden
  del dict) y **loguear en debug cada descarte** con qué fuente ganó. Reordenar `SOURCES`
  alfabéticamente no debe poder hacer desaparecer un documento activo.

**Política:** los tres pasan a «saltar+log» (warning). Ninguno cambia qué se descarga hoy; añaden
señal. Tests: dos filas activas mismo `domain_code` sin `Part` → warning; con `Part` → sin warning;
URL cuyo basename no contiene el `ecss_number` → warning.

---

## Grupo D — Robustez y correcciones menores (C-9, C-11, C-13, C-14, C-10)

Baratas, sin controversia. Aplicar todas:

- **C-9 — lectura robusta del inventario** ([downloader.py:95-99]): validar las columnas requeridas
  una vez al abrir el CSV y fallar con **mensaje explícito** (qué columna falta) en vez de un
  `KeyError` crudo; leer con `encoding="utf-8-sig"` (inocuo sin BOM, salva el caso «alguien abrió el
  CSV en Excel y lo guardó»). Fail-fast es correcto aquí (inventario ilegible **debe** parar); solo
  mejorar el mensaje.
- **C-11 — cerrar la respuesta HTTP** ([downloader.py:132]): `with session.get(...) as resp:` para
  que un `raise_for_status()` no deje la conexión sin devolver al pool.
- **C-13 — `scraped_at` con timezone** ([scraper.py:275]): `datetime.now(timezone.utc).isoformat()`
  en vez de `datetime.utcnow()` (deprecado, y naive para un campo cuyo único fin es procedencia).
- **C-14 — acotar el selector** ([scraper.py:278]): `soup.select("ol li a")` captura cualquier lista
  ordenada (menús, navegación). Acotarlo al contenedor real del listado de documentos (el informe no
  fija el selector exacto — CC elige el más específico que capture el listado y excluya navegación).
  Autolimitado hoy, pero barato de cerrar.
- **C-10 — limpieza de `.tmp` huérfanos**: al arrancar el downloader, barrer `*.tmp` de una ejecución
  anterior interrumpida por kill duro (el `try/except` no cubre `SIGINT`/corte de luz). Log de lo
  barrido.

---

## Fuera de alcance de 1b (se difiere a su brief, deliberadamente)

Estas recomendaciones del informe **NO** se tocan aquí porque pertenecen a un brief posterior cuyo
diseño las gobierna:

- **C-4 (identidad de cita reversible)** → Brief 5. No tocar `sanitize_filename` ni la reconstrucción
  de `doc_id` desde nombre de fichero aquí; la solución es que la identidad viaje desde el manifiesto,
  y eso se decide en Brief 4→5.
- **C-15 (cobertura `.doc`/`.docx`, conversión)** → Brief 2. Es el trabajo grande siguiente; 1b no
  convierte nada ni toca `verify_parser.py`.
- **C-16 (huérfano `ECSS-E-ST-70-01C16April2010`)** → **el usuario lo borra a mano** (son sus datos).
  1b no lo toca. La arquitectura `raw/` inmutable + `parse_ready/` que lo previene en el futuro es
  Brief 2.
- **C-12 (`production_status` sin usar)** → señal para el manifiesto, Brief 4.
- **O-1 (poda de revisiones viejas)** → reconciliación contra manifiesto, Brief 4.
- **O-2 (scraper visita todas las páginas de detalle)** → optimización con coste/beneficio modesto
  atado al alcance del corpus; se decide junto con ese alcance, no ahora.
- **O-5 (cache del índice del scraper)** → no paga su complejidad (fuente que cambia un puñado de
  veces al año); no se hace.
- **Deuda de ruff preexistente** (imports sin usar en `verify_parser.py`/`test_parser.py`, f-string
  vacío en `diagnose_doc.py`) → **sí** aplicar `uv run ruff check --fix` sobre esos ficheros si es
  autocorregible y no cambia comportamiento; es limpieza barata que ya estaba señalada. Si algún fix
  no es mecánico, dejarlo y anotarlo.

---

## Qué entrega CC

1. Los fixes de los Grupos A-D aplicados en `corpus/downloader.py` y `corpus/scraper.py`.
2. Tests de caracterización nuevos/actualizados (mockeados, sin red) — con foco en los cambios de
   comportamiento del Grupo A y los warnings del Grupo C. Los dos tests que hoy fijan comportamiento
   incorrecto (`test_two_active_rows_...` queda como está — C-1 solo avisa, no cambia qué se descarga;
   `test_first_word_link_wins_...` se **actualiza** para afirmar el warning de C-3).
3. `uv run pytest` en verde.
4. Actualización de `corpus/AUDIT-downloader.md`: marcar cada hallazgo aplicado como resuelto (o una
   nota breve al pie «aplicado en Brief 1b»), para que el informe siga siendo el registro vivo de qué
   queda abierto. No reescribir el informe; solo anotar estado.
5. `SNN-decisions.md` (S06) con el log de decisiones de implementación etiquetadas agnóstico/dominio,
   como marca el flujo.

## Qué NO hace CC

- No ejecuta downloader/scraper sobre datos reales (UAT del usuario).
- No toca `verify_parser.py`, el parser, el grafo, ni `Citation` (Briefs 2/3/5).
- No convierte `.doc` (Brief 2), no borra el huérfano C-16 (usuario), no diseña el manifiesto (Brief 4).
- No implementa selección automática de vigencia ni resolución de enlaces — solo **detección** (warnings).

## Criterio de "hecho"

- Los Grupos A-D están aplicados; ningún fallo del informe clasificado «silenciar» queda en silencio
  (o pasa a saltar+log/abortar, o está explícitamente diferido a su brief arriba).
- `uv run pytest` pasa; los cambios de comportamiento tienen test que afirma el resultado **nuevo**.
- El downloader devuelve exit code ≠ 0 ante cualquier fallo, y su resumen lista qué falló con motivo.
- `AUDIT-downloader.md` refleja qué se aplicó y qué sigue abierto.
- Se ha entregado un guion de UAT propuesto para que el usuario ejecute el downloader endurecido sobre
  datos reales (incluye: forzar un fallo de integridad — p.ej. apuntar a una URL que devuelve HTML — y
  confirmar que ahora sale como `Failed`, no como `Downloaded`).
