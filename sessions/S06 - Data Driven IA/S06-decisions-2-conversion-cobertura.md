# S06 — Decisions log, brief 2: conversión `.doc`→`.docx` + cobertura del corpus (C-15)

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier pipeline de
> datos) / `[dominio]` (específico de RAG-ECSS y del corpus ECSS).
>
> El reconocimiento de la fase 1 está en `corpus/RECON-conversion.md`. Aquí solo las decisiones
> de implementación que el brief dejaba abiertas o que el entorno obligó a tomar.

---

## 1. `soffice.com` y no `soffice.exe`: el fallo silencioso que el brief habría reintroducido `[agnóstico]`

El brief propone `soffice --headless --convert-to docx`. En el entorno del usuario (Windows) esa
invocación tiene **dos problemas**, y el segundo es grave.

El primero es menor: LibreOffice 26.2.3.2 está instalado pero **no en el PATH**, así que
`soffice` a secas es `FileNotFoundError`. Se resuelve localizando el binario.

El segundo es de fondo. `soffice.exe` y `soffice.com` pesan lo mismo y viven en la misma carpeta,
pero no son lo mismo:

| invocación | stdout | exit code |
|---|---|---|
| `soffice.exe --version` | *(nada)* | — |
| `soffice.com --version` | `LibreOffice 26.2.3.2 …` | 0 |

`soffice.exe` está marcado como aplicación GUI: **se desacopla de la consola y retorna al
instante**, sin esperar y sin escribir nada. Un `subprocess.run()` contra él habría devuelto
exit 0 inmediatamente con la conversión sin hacer, y el script habría reportado *12/12
convertidos* sobre un `parse_ready/` vacío o a medio llenar.

Eso es exactamente el fallo silencioso que S06 lleva tres briefs eliminando, reintroducido por
una diferencia de extensión de fichero. Es el tipo de detalle que no se descubre razonando sobre
la herramienta, solo ejecutándola — y justifica por sí solo que la fase 1 fuese reconocimiento
antes que implementación.

## 2. El éxito de la conversión lo decide el artefacto, no el exit code `[agnóstico]`

Aunque se use `soffice.com`, **no se confía en su código de salida**. LibreOffice tiene el hábito
documentado de devolver 0 sin producir fichero (formato de entrada raro, perfil bloqueado,
filtro que falla a medias). `convert_with_libreoffice` comprueba tres cosas en orden:

1. que el `.docx` de salida **exista**,
2. que pase la **verificación de magic bytes + tamaño** del Brief 1b (`verify_word_payload`),
3. y solo entonces lo acepta; si no, lo borra y devuelve el motivo.

Es el mismo principio que la compuerta `.tmp` del downloader: **a un directorio de entrada solo
llega lo que ya se sabe válido**. Reutilizar `verify_word_payload` en vez de escribir una
segunda comprobación es deliberado — dos definiciones distintas de «esto es un Word» acabarían
divergiendo.

## 3. Perfil de usuario temporal para LibreOffice `[agnóstico]`

LibreOffice comparte el perfil de usuario entre procesos: si el usuario tiene un Writer abierto,
la conversión headless puede colgarse o abortar en silencio. Se pasa
`-env:UserInstallation=file:///<tempdir>` con un directorio temporal por conversión.

La alternativa era documentarlo en el guion UAT («cierra LibreOffice antes de ejecutar»). Se
descartó: una precondición que alguien se va a saltar no es una precondición, es un bug con
buenos modales. Más `--norestore` y un timeout de 180 s, porque colgarse es un modo de fallo
real y un proceso colgado no deja traza.

## 4. `parse_ready/` duplica los `.docx` nativos, y eso es la solución, no el precio `[agnóstico]`

El brief ya cierra esta decisión, pero conviene registrar por qué la duplicación es correcta y no
una concesión.

La alternativa —convertir los `.doc` *dentro* de `raw/`— parece ahorrar 20 MB, pero produce
**12 colisiones de stem**: `X.doc` y `X.docx` conviviendo, donde un glob `*.docx` ya no puede
distinguir original de derivado. Ese defecto exacto **ya existe** en el corpus con el residuo
del PoC (`ECSS-E-ST-70-01C16April2010`), y era la C-16 de la auditoría.

Con `parse_ready/` el problema no se mitiga: **no puede ocurrir**. `raw/` queda auditable contra
lo que dio el servidor, `parse_ready/` es homogéneo por construcción, y todo consumidor tiene un
único directorio de entrada. El coste es disco duplicado de un artefacto de build reconstruible.

## 5. Ante una colisión de stem en `raw/`, gana el `.docx` nativo — pero se reporta `[dominio]`

Caso real: `raw/` contiene `X.doc` y `X.docx` para la misma identidad. Ambos apuntan al mismo
`parse_ready/X.docx`.

Se prefiere el `.docx` nativo. Convertir un `.doc` teniendo ya el original sin pérdidas sería
degradar a propósito. Pero la colisión **se reporta** (`stem_collision_in_raw`): significa que
`raw/` tiene dos ficheros para un documento, que es una anomalía aunque la resolución sea obvia.

Se descartó fallar ante la colisión: bloquearía el brief entero por el residuo conocido de C-16,
cuyo borrado es del usuario.

## 6. Si falta LibreOffice y hay `.doc` que convertir: abortar antes de empezar `[agnóstico]`

Tensión con la política (a) del brief («nunca abortar el batch por un documento»). La resolución
es que **no es un fallo de documento, es un fallo de entorno**: no falla un `.doc` concreto,
fallan los doce por la misma causa.

Si se aplicase la política (a), el script copiaría los 14 `.docx`, fallaría los 12 `.doc` y
dejaría un `parse_ready/` con exactamente la misma brecha 14/26 que este brief existe para
cerrar — con cara de haber trabajado. Abortar antes de escribir nada (exit 2, mensaje que dice
cómo arreglarlo con `LIBREOFFICE_PATH`) es más honesto.

Matiz que sí respeta la política: la comprobación **solo ocurre si hay conversiones pendientes**.
Un corpus todo-`.docx` no necesita LibreOffice y no debe fallar por no tenerlo
(`test_missing_libreoffice_is_irrelevant_without_conversions`).

## 7. La cobertura se reconcilia normalizando identidades, y eso es un parche declarado `[dominio]`

Los tres artefactos deletrean el mismo documento de tres formas:

```
inventario    ECSS-E-ST-40C Rev.1
parse_ready   ECSS-E-ST-40C_Rev_1     (sanitize_filename, lossy)
processed     ECSS-E-ST-40C_Rev.1     (save_parsed, conserva el punto)
```

Para cruzarlos hay que normalizar a alfanuméricos. Funciona, pero **es un parche sobre C-4**: la
reconciliación solo necesita esta gimnasia porque no existe una clave canónica de documento. Va
marcado con `# TODO` apuntando a Brief 5 — cuando la identidad viaje por el manifiesto, la
comparación pasa a ser por clave y esta función se simplifica a una intersección de conjuntos.

Registrarlo importa porque la normalización es **silenciosamente frágil**: si dos documentos
distintos normalizasen igual, se contarían como uno y la cobertura mentiría en la dirección
optimista. Hoy no ocurre; nada lo garantiza.

## 8. La cobertura solo se calcula en la invocación por defecto `[agnóstico]`

`verify_parser.py` sin argumentos verifica `parse_ready/` entero **y** reconcilia cobertura. Con
ficheros concretos, solo verifica esos.

El motivo: una ausencia no tiene fichero que inspeccionar. Reconciliar cobertura cuando el
usuario pidió dos documentos concretos reportaría 24 «NO PARSEADO» que no son un hallazgo, son la
consecuencia de lo que pidió. Ruido que enseña a ignorar el informe.

Sub-decisión: `run_coverage_check` devuelve ≠ 0 si hay brecha. **La cobertura incompleta es un
fallo de la verificación, no una nota informativa** — que era precisamente el problema de C-15.

## 9. `unexpected` no compensa a `missing` `[dominio]`

La reconciliación reporta también los documentos en `processed/` sin fila activa en el inventario
(el huérfano C-16). Se cuentan aparte y **no entran en el cálculo de cobertura**: un documento de
más no puede compensar uno de menos. Fijado por
`test_unexpected_documents_do_not_inflate_coverage`.

## 10. Bug preexistente corregido de paso: el import roto de `verify_parser.py` `[agnóstico]`

El fichero tenía `from parser import parse_document, accepted_text` con el comentario
*«in repo: from corpus.parser import …»*, y un docstring que apuntaba a `scripts/verify_parser.py`
cuando el fichero vive en `corpus/`. **La herramienta no arrancaba** desde la raíz del repo.

Se corrigió a `from corpus.parser import …` porque la fase 3 exige que la herramienta funcione:
una verificación que no se puede ejecutar es una forma extrema del mismo problema que C-15.

## 11. Qué NO se hizo `[dominio]`

- **Regenerar el grafo.** El reconocimiento confirma que se construyó sobre 14 documentos, pero
  regenerarlo sobre ~26 con el extractor de referencias **sin validar** amplificaría falsos
  positivos ya conocidos sin medirlos. Se mide el cambio de universo; la regeneración va con su
  validación contra ground truth.
- **Resolver la identidad canónica** (Brief 5): el stem se preserva tal cual, con `# TODO` en los
  dos puntos donde toca identidad.
- **Persistir o ponderar la cobertura** (Brief 4): se calcula y se muestra; no se guarda ni se
  clasifica por relevancia.
- **Borrar el huérfano C-16**: son datos del usuario. Ahora tiene procedencia documentada (§ del
  RECON: es el residuo del PoC de `scripts/inspect_docx.py`, borrado en `20f5899`).
- **Instalar LibreOffice en el Dockerfile.** `CLAUDE.md` lo lista como pendiente desde 0.1.0 y
  sigue sin hacerse; el script funciona en local, que es donde se ejecuta el pipeline de corpus
  hoy. Anotado como tensión.

## Tensiones abiertas

- **La conversión no está en Docker.** El pipeline de corpus depende de que haya un LibreOffice
  en la máquina del desarrollador. Reproducible para el usuario, no para un tercero que clone el
  repo. Cuando `corpus/` tenga que correr en CI, esto hay que resolverlo.
- **No se mide la fidelidad de la conversión.** El script verifica que el `.docx` resultante es un
  Word válido, no que LibreOffice haya preservado estilos `require:level1` y `Heading N`, que es
  de lo que depende el parser entero. El CHANGELOG 0.2.0 dice que se verificó a mano en el PoC —
  sobre **un** documento. Los 12 convertidos son fe hasta que el parser los procese, y ahí es
  donde lo dirán los invariantes estructurales de `verify_parser.py`. **El guion UAT lo cubre
  como paso explícito**, pero conviene saber que el riesgo se traslada, no desaparece.
- **La normalización de identidades (§7) puede colisionar** y mentiría hacia arriba. Se cierra
  con el manifiesto.
