# S06 — Decisions log, brief 1b: endurecimiento del downloader y el scraper

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier pipeline de
> datos) / `[dominio]` (específico de RAG-ECSS y del corpus ECSS).
>
> El brief venía con la especificación cerrada (`corpus/AUDIT-downloader.md`), así que aquí solo
> están las decisiones que el brief **dejaba abiertas** o que la implementación obligó a tomar.

---

## 1. Un fichero solo llega a `data/raw/` cuando ya se sabe que es válido `[agnóstico]`

La verificación de integridad podía colocarse en dos sitios: después del `rename`, comprobando
el fichero final, o antes, comprobando el `.tmp`. Se eligió **antes**, y el orden importa más de
lo que parece.

Verificar después obliga a decidir qué hacer con un fichero malo que **ya está** en el destino:
borrarlo (y perder la copia buena anterior si la había), dejarlo (y volver al problema de
partida), o moverlo a un limbo. Verificando el `.tmp`, el destino es un sitio donde por
construcción solo entra lo que pasó el control: no hay estado intermedio malo que gestionar.
El test `test_a_failed_download_never_overwrites_a_good_local_file` es exactamente esa
propiedad: una re-descarga que falla deja intacta la copia buena que ya había.

Esto convierte el `.tmp` —que existía solo para evitar ficheros parciales— en la **compuerta de
calidad** del pipeline. Un buen seam paga dos veces.

## 2. Un fichero existente que no pasa la verificación se re-descarga, no se marca fallido `[agnóstico]`

El brief dejaba la elección a CC («re-descargar o marcarlo fallido — decisión de detalle»).

Se eligió **re-descargar**. Marcarlo fallido sería técnicamente más conservador, pero deja al
usuario con un error que solo puede resolver borrando el fichero a mano, y para eso primero
tiene que entender que el mensaje habla de un fichero envenenado en una ejecución anterior. La
re-descarga es autorreparadora: el caso que motiva C-5 (un HTML de login guardado hace meses) se
arregla solo la próxima vez que alguien corre el downloader.

El riesgo de re-descargar —destruir algo bueno— está cubierto por la decisión §1: si la
re-descarga falla, el fichero anterior sigue ahí. Y no colisiona con `--force`: son la misma
puerta, una automática por integridad y otra manual por voluntad. Lo único que se cuidó fue no
mezclarlas en el reporte: `--force` no marca los ficheros como inválidos
(`test_force_redownloads_a_valid_file`), porque no lo son.

## 3. `rename` → `replace`: el bug que la reparación automática destapó `[agnóstico]`

`Path.rename` **falla en Windows si el destino existe**. Mientras el destino nunca existía (el
downloader solo bajaba lo ausente) no se notaba. En cuanto §2 y `--force` introducen la
sobrescritura, `rename` habría lanzado `FileExistsError` justo en el camino de recuperación —
es decir, el downloader habría sido incapaz de reparar un fichero corrupto, en Windows, y solo
en Windows.

SÍNTOMA → una feature nueva (re-descarga) rompiendo en una plataforma. CAUSA → API de
`pathlib` con semántica distinta por SO. CAMBIO → `Path.replace`, que es atómico y sobrescribe
en ambos. ETIQUETA → `[agnóstico]`. Cubierto por
`test_redownload_overwrites_an_existing_file`.

## 4. El umbral de tamaño mínimo: 20 KB, y por qué un suelo y no una horquilla `[dominio]`

`MIN_PLAUSIBLE_DOC_BYTES = 20_000`. El documento real más pequeño del corpus ronda los 130 KB
(`ECSS-Q-ST-10C Rev.1`), así que 20 KB deja un factor 6 de holgura: no puede haber falsos
positivos con documentos legítimos, y sí atrapa páginas de error y HTML de login, que pesan
unos pocos KB.

Se descartó poner también un techo. Un documento anormalmente **grande** no es sospechoso en
este corpus: `ECSS-E-ST-70-31C` pesa 10,7 MB, veinte veces la mediana, y es legítimo. Un techo
sería un falso positivo esperando a ocurrir contra un beneficio nulo.

## 5. La verificación mira magic bytes, no `Content-Type` `[agnóstico]`

Se descartó comprobar la cabecera `Content-Type` de la respuesta. Es más barata, pero es **lo
que el servidor dice**, no lo que mandó; un portal mal configurado sirve un HTML de error con
`Content-Type: application/msword` sin inmutarse. Los magic bytes son el fichero mismo. Dado
que la verificación existe precisamente porque no nos fiamos del servidor, preguntarle a él
habría sido circular.

## 6. `skipped_no_docx` NO hace fallar la ejecución `[dominio]`

El brief pide exit code ≠ 0 «si hubo algún fallo». La pregunta abierta era si un documento
activo sin `docx_url` cuenta como fallo. **Se decidió que no.**

Es una propiedad permanente del inventario: hay estándares ECSS publicados solo como PDF, y eso
no cambia entre ejecuciones. Si contara como fallo, el comando estaría rojo de forma permanente
y el exit code dejaría de significar nada — el clásico mecanismo de alarma que todo el mundo
aprende a ignorar. Se resuelve con un `log.warning` nominal y su lista en el resumen: visible,
pero sin quemar la señal binaria. `test_missing_docx_alone_does_not_fail_the_run` lo fija.

Tensión que queda: si algún día un documento CORE desaparece del inventario, esto lo degradaría
a warning cuando debería ser un fallo. La distinción correcta sería por relevancia
(`software_relevance`), no por causa — pero eso necesita el manifiesto del Brief 4 para saber
qué se esperaba tener.

## 7. C-3 detecta con dos señales distintas, y una de ellas es deliberadamente débil `[dominio]`

`check_link_identity` compara el basename de la URL con el `ecss_number`. La implementación
obvia —igualdad tras normalizar— es inservible aquí: los ficheros reales de ecss.nl llevan
tokens extra (`ECSS-E-ST-70-01C(16April2010).doc`), así que la igualdad estricta marcaría casi
todo el corpus. Se implementó **contención** (¿está el número dentro del nombre?), que atrapa el
caso grave —bajar otro documento distinto— sin falsos positivos.

Pero la contención sola **no** atrapa el caso que el brief nombra como el peor: un anexo del
mismo documento contiene el número igualmente. De ahí la segunda señal, una lista corta de
marcadores (`annex`, `drd`, `corr`…). Es heurística y lo dice el docstring: *«no puede probar
que el enlace es correcto; sí puede decir cuándo es obviamente incorrecto»*.

Se descartó lo que habría sido más eficaz —elegir el enlace cuyo nombre mejor encaje— porque el
brief acota a detección: reordenar la selección es resolución automática, y esa es decisión de
diseño. `test_first_word_link_still_wins_but_is_now_flagged` fija ambas mitades: la elección no
cambia, la traza sí existe.

## 8. C-1 se detecta desde el `ecss_number`, no desde la columna `domain_code` `[dominio]`

Para avisar de revisiones en competencia hace falta el `domain_code`. Está en el inventario como
columna, pero usarla habría convertido `domain_code` en columna **requerida**, ampliando el
contrato del CSV solo para un warning. Se reutiliza `parse_domain_code` de `scraper.py`, que ya
lo deriva del número. Las columnas requeridas siguen siendo tres.

Sub-decisión: la regla de exclusión de partes es «**ninguna** del grupo lleva `Part`». El caso
real (`Q-HB-80-02 Part 1A` / `Part 2A`) es legítimo, pero un grupo mixto —una parte y una
revisión suelta— ya no es ese patrón y sí merece aviso.

## 9. `structlog` propio para `corpus/`, sin importar `app/observability.py` `[agnóstico]`

`configure_logging` de la app construye su config desde `app.config.Settings`, que **exige
`ANTHROPIC_API_KEY`**. Importarla desde `corpus/` habría hecho que el downloader necesitase una
credencial de LLM para bajar ficheros — acoplamiento absurdo entre el pipeline de datos offline
y el runtime del producto. `corpus/log_config.py` replica la configuración mínima (13 líneas),
sin `request_id` porque aquí no hay requests.

Elección de renderer: **TTY → consola, no-TTY → JSON**. La app decide por `APP_ENV`; aquí no hay
entorno que consultar, y el criterio útil es distinto: un humano mirando la terminal quiere
texto, y `> run.log` quiere JSON. Es el mismo objetivo (que el resumen sea dato cuando toque)
por el medio que aplica a un CLI.

## 10. Los dos resúmenes pasan a ser registros, incluido el del scraper `[agnóstico]`

El brief encarga T-3 solo al downloader y deja el esquema del manifiesto al Brief 4. Se convirtió
también el `print_summary` del scraper: bajo `JSONRenderer` habría emitido una docena de objetos
JSON con arte ASCII dentro, que es peor que el banner original. Migrar el logger sin migrar lo
que emite habría dejado el trabajo a medias.

Se respetó la frontera: los resúmenes son **estructurados y consultables** (campos, listas
nominales), pero **no se persisten** ni se define esquema de procedencia. Eso sigue siendo
Brief 4.

Efecto colateral buscado: el scraper ahora emite `active_without_docx` con los nombres. Es la
misma clase de gap que C-15 destapó en el parser —un documento que se pierde y nadie cuenta—
y ahora se nombra en origen.

## 11. C-14: acotar el selector con fallback, no elegir un selector nuevo a ciegas `[dominio]`

El brief pide acotar `soup.select("ol li a")` y deja el selector a CC. Sin poder mirar el HTML
real de ecss.nl (es red, es UAT), fijar un selector concreto habría sido adivinar: si acierto,
bien; si no, el scraper deja de encontrar documentos y el fallo es total.

Se implementó una **cascada con degradación**: `main` → `article` → `.entry-content` →
`#content` → el documento entero. Si alguno existe, el listado queda acotado; si ninguno, el
comportamiento es idéntico al de antes. La mejora es estrictamente no destructiva, que es lo
único responsable cuando no puedes verificar contra la fuente.

## 12. Qué NO se hizo `[dominio]`

- **Selección automática de vigencia** (C-1): solo detección. `# TODO` vivo en
  `build_download_plan`.
- **Reordenar la elección de enlace** (C-3): solo detección. `# TODO` implícito en el docstring.
- **Paralelizar descargas** (O-3): el informe recomienda explícitamente no hacerlo.
- **Cache del índice del scraper** (O-5): descartado, no paga su complejidad.
- **Filtrar por relevancia antes del fetch de detalle** (O-2): cambia el contenido de
  `ecss_inventory_full.csv`; atado al alcance del corpus.
- **Tocar `sanitize_filename`, `verify_parser.py`, el parser o el grafo**: C-4 y C-15 son
  Briefs 5 y 2.

## Tensiones abiertas

- **El exit code no distingue relevancia** (§6). Un handbook ADJACENT sin DOCX y un estándar
  CORE ausente pesan lo mismo hoy: warning. La distinción necesita saber qué se esperaba tener,
  o sea el manifiesto del Brief 4.
- **`check_link_identity` no puede validar de verdad** (§7). Detecta lo obvio. La validación
  fuerte sería abrir el DOCX y leer su portada, que es trabajo del parser y llegaría demasiado
  tarde para arreglar la descarga. Puede que la respuesta correcta no sea validar mejor el
  enlace, sino reconciliar después contra el manifiesto.
- **La cascada de selectores de §11 no está verificada contra el HTML real.** Es segura por
  construcción (degrada al comportamiento anterior), pero si ninguna de las cuatro clases existe
  en ecss.nl, C-14 sigue abierto de facto y nadie se entera. Lo cubre el paso 5 del UAT.
