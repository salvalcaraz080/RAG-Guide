# S06 — Brief 2: Conversión `.doc`→`.docx` + cobertura del corpus (C-15)

> **Tipo:** trabajo de datos en tres fases dependientes — **reconocimiento → conversión →
> cobertura visible**. La fase 1 condiciona las otras dos: no se convierte nada hasta saber qué hay.
>
> **Contexto (el hallazgo dominante de la auditoría):** el corpus procesado son **14 de 26**
> documentos activos (cobertura 53%). Los 12 ausentes son `.doc` legacy que el parser no lee y que
> `verify_parser.py` descarta **en silencio** (glob `*.docx`, ignora `.doc`, reporta OK sobre lo que
> mira sin mencionar lo que no). Entre lo perdido está `ECSS-E-HB-40A`, el handbook CORE de
> ingeniería de software. Este brief cierra esa brecha: convierte los `.doc` a `.docx` y hace que la
> cobertura sea **visible y contada**, nunca implícita.
>
> **Frontera CC:** CC escribe el código de conversión y sus tests (mockeados, sin invocar LibreOffice
> real, sin tocar `data/`). **Ejecutar la conversión sobre los `.doc` reales es UAT del usuario** —
> toca disco y LibreOffice headless. CC entrega el script + guion; el usuario lo corre.

---

## Arquitectura de directorios (contrato nuevo, gobierna todo el brief)

```
data/
├── raw/          # SOLO lo que da el servidor. Inmutable. Propiedad exclusiva del downloader.
│                 #   .docx nativos + .doc legacy, tal cual se descargaron. NADIE escribe aquí salvo el downloader.
├── parse_ready/  # Entrada ÚNICA del parser. TODO .docx. Se construye desde raw/:
│                 #   - los .docx nativos de raw/ se COPIAN aquí
│                 #   - los .doc legacy de raw/ se CONVIERTEN aquí
└── processed/    # Salida del parser (JSON por documento). Sin cambios de ubicación.
```

**Reglas del contrato:**
- El parser (y `verify_parser.py`, y todo consumidor del corpus) lee de **`parse_ready/`**, nunca de
  `raw/`. Esto es un cambio: hoy leen de `raw/`.
- Sí se duplican los `.docx` nativos en disco (copia raw→parse_ready). Es el precio aceptado de
  «un solo directorio de entrada, homogéneo, sin colisiones de stem». A cambio: `raw/` queda auditable
  contra lo descargado, y un glob `parse_ready/*.docx` no puede mezclar original con derivado (cierra
  C-16 y las 12 colisiones de stem futuras **por construcción**, no por parche).
- `parse_ready/` es **derivable y desechable**: se puede borrar y reconstruir entero desde `raw/`. No
  es fuente de verdad de nada; es un artefacto de build. (Consecuencia: no se commitea, igual que
  `raw/` y `processed/`.)

---

## Fase 1 — Reconocimiento (CC investiga ANTES de escribir conversión)

No asumir nada sobre el estado actual. CC debe **averiguar y reportar** (en `corpus/RECON-conversion.md`
o similar) antes de implementar:

1. **¿Existe ya integración de LibreOffice en el repo?** Buscar invocaciones a `libreoffice`,
   `soffice`, `unoconv`, o conversión `.doc`→`.docx` en cualquier script de `corpus/`, `scripts/`, o
   utilidades. CLAUDE.md la lista como *pendiente*, pero la duda del usuario es legítima: el grafo y
   las cláusulas ya se extrajeron, y eso pudo requerir conversión. **Si existe una ruta de conversión
   previa, documentarla** (dónde, qué invoca, con qué flags/convenciones de nombre) — reutilizarla en
   vez de crear una segunda ruta paralela (lección de C-4: no dejar dos rutas sin declarar cuál es la
   buena).
2. **¿Sobre cuántos documentos se construyó el grafo — 14 o 26?** CLAUDE.md dice «14 documentos
   parseados, 273 referencias». Cruzar: ¿los nodos/aristas del grafo salen solo de los 14 `.docx`, o
   algún `.doc` entró por una conversión previa? Esto decide si el grafo **también** sufre C-15 (se
   construyó incompleto) y hay que regenerarlo tras la conversión. **Hipótesis a verificar, no asumir:
   probablemente el grafo es de 14 y habrá que regenerarlo — pero confirmarlo con evidencia del repo.**
3. **¿Qué herramienta de conversión está disponible?** Verificar qué hay instalable/invocable en el
   entorno objetivo (el del usuario, Windows según los guiones UAT previos con PowerShell): LibreOffice
   headless (`soffice --headless --convert-to docx`) es la vía por defecto del proyecto (ya en el
   stack de CLAUDE.md), pero CC confirma la invocación correcta y sus modos de fallo antes de
   depender de ella.

**Entregable de la fase 1:** un informe corto de reconocimiento + una recomendación sobre si el grafo
necesita regeneración. Si el reconocimiento revela algo que cambie el plan (p.ej. ya hay un script de
conversión con convenciones propias), **parar y reportar** antes de la fase 2 — no seguir con
suposiciones.

---

## Fase 2 — Conversión (`corpus/build_parse_ready.py` o similar)

Un script que construye `parse_ready/` desde `raw/`. Responsabilidad: **preparar el directorio de
entrada del parser**, no parsear.

**Qué hace:**
1. Copia cada `.docx` nativo de `raw/` a `parse_ready/` (verificación de que es DOCX válido — magic
   bytes, reutilizando la lógica del Brief 1b si es factible — antes de copiar; un `raw/` sano ya lo
   garantiza, pero la copia es barato sitio para reconfirmar).
2. Convierte cada `.doc` legacy de `raw/` a `.docx` en `parse_ready/` vía LibreOffice headless.
3. **Nombres:** el `.docx` resultante conserva el stem del `.doc` origen (misma identidad de
   documento). Ojo con la convención de nombres del downloader (C-4: `sanitize_filename` es lossy) —
   **no** re-derivar identidad desde el nombre aquí; la identidad canónica es problema del Brief 5, y
   este script solo preserva el stem existente, sea el que sea. Anotar con `# TODO` el enganche a
   Brief 5 donde el stem toque la identidad.

**Política de fallo de conversión — decisión tomada: (a) ruidoso, cuenta y sigue.**
- Un `.doc` que no convierte (corrupto, timeout de LibreOffice, formato raro) **no entra a
  `parse_ready/`**, pero la conversión **lo cuenta y lo nombra**: `convertidos: 10/12 · fallidos: 2
  [ECSS-X, ECSS-Y] motivo: …`. Nunca abortar el batch por un documento (b descartada), nunca perderlo
  sin traza (c es el pecado).
- Exit code ≠ 0 si algún documento esperado no llegó a `parse_ready/` — coherente con el Brief 1b.
  Con un matiz de dominio (ver abajo): la severidad de un fallo depende de la relevancia del documento,
  pero **la clasificación por relevancia es del manifiesto (Brief 4)**; aquí basta con contar y nombrar
  para que Brief 4 pueda ponderar.
- **Idempotencia:** re-ejecutar no re-convierte lo ya presente y válido en `parse_ready/` salvo
  `--force` (mismo patrón que el downloader). `parse_ready/` es reconstruible desde cero borrándolo.

**Nota de dominio (no cambia el mecanismo, informa la lectura del resultado):** si el que falla es
`ECSS-E-HB-40A` (handbook CORE) el impacto es mayor que si falla un ADJACENT. El script no decide eso
—solo cuenta y nombra—; la ponderación por relevancia la hace el manifiesto. Pero el guion UAT debe
mirar **explícitamente** si algún CORE/RELATED quedó sin convertir.

---

## Fase 3 — Cobertura visible (`verify_parser.py` deja de callar la brecha)

El corazón de C-15: la herramienta de verificación **omitía media entrada sin decirlo**. Se corrige.

1. **`verify_parser.py` lee de `parse_ready/`** (todo `.docx` homogéneo), no de `raw/` con glob
   `*.docx`. Al leer de `parse_ready/` ya no hay `.doc` que ignorar — pero eso **no basta**, porque el
   silencio real es no comparar contra lo esperado.
2. **Reconciliación explícita activos-vs-parseados.** La verificación debe cruzar tres cuentas y
   **reportar la brecha**, no solo validar lo presente:
   - documentos activos con contenido descargable en el inventario (lo que el corpus *declara* tener),
   - documentos en `parse_ready/` (lo que llegó a la entrada del parser),
   - documentos en `processed/` (lo que el parser produjo).
   Cualquier documento activo que no llegue a `processed/` es un **`NO PARSEADO` nominal con motivo**
   (no convertido / conversión fallida / parseo fallido), no una ausencia silenciosa. La cobertura
   (`parseados / activos`) es una salida de primera clase de la herramienta.
3. **Una herramienta de verificación que ignora entradas es peor que no tener verificación.** El fix
   central: `verify_parser.py` nunca debe reportar «OK» sobre un subconjunto sin nombrar el
   complemento. Si mira N de M, dice «miré N, faltan M−N y son estos».

**Frontera con Brief 4:** esta reconciliación es *cálculo en tiempo de verificación*, no un artefacto
persistido. El **manifiesto** (Brief 4) es donde esa cobertura se **persiste y se clasifica por
relevancia**. Aquí: que la herramienta la calcule y la muestre. Allí: que quede registrada y
ponderada. No adelantar el esquema del manifiesto.

**Regeneración del grafo (condicional al reconocimiento):** si la fase 1 confirma que el grafo se
construyó sobre 14, este brief **no** lo regenera (es trabajo con su propia validación — el grafo sin
validar contra ground truth es deuda conocida, Brief 6). Pero **deja anotado y medido** cuántos
documentos nuevos entran ahora al corpus, porque eso cambia el universo del grafo y de las tablas
(Brief 3). Solo medir y anotar; no regenerar a ciegas.

---

## Qué entrega CC

1. **`corpus/RECON-conversion.md`** — informe de la fase 1 (LibreOffice previo sí/no, grafo 14-vs-26,
   herramienta de conversión disponible) + recomendación sobre regeneración del grafo.
2. **Script de construcción de `parse_ready/`** (`corpus/build_parse_ready.py` o nombre que CC juzgue)
   — copia `.docx` + convierte `.doc`, con política (a), idempotencia, `--force`, exit code, y resumen
   estructurado (structlog, coherente con Brief 1b).
3. **`verify_parser.py` corregido** — lee de `parse_ready/`, reconcilia activos/parse_ready/processed,
   reporta cobertura y nombra lo ausente con motivo. Nunca «OK» parcial silencioso.
4. **Tests** (mockeados, sin LibreOffice real, sin `data/`): la lógica de decisión copiar-vs-convertir
   desde un listado de ficheros sintético; la reconciliación de cobertura desde tres listados
   sintéticos (incluir el caso «un activo no llegó a processed» → aparece como NO PARSEADO con motivo);
   el conteo de fallos de conversión (mock del invocador de LibreOffice que devuelve fallo para un
   fichero). `uv run pytest` en verde.
5. **Guion UAT** — pasos que el usuario ejecuta: construir `parse_ready/` desde `raw/` real, confirmar
   26 (o los que haya) `.docx` en `parse_ready/`, correr `verify_parser.py` y confirmar cobertura
   objetivo (idealmente 26/26; si <26, qué falló y si alguno es CORE/RELATED), y confirmar que
   `ECSS-E-HB-40A` (handbook CORE) está entre los convertidos. Incluir la comparación de qué documentos
   nuevos entran respecto a los 14 previos.
6. **`S06-decisions.md`** (append) con las decisiones de implementación, etiquetadas agnóstico/dominio.

## Qué NO hace CC

- No ejecuta LibreOffice ni toca `data/` (UAT del usuario).
- No resuelve la identidad canónica de cita (Brief 5) — solo preserva el stem existente, con `# TODO`
  donde enganche.
- No diseña el manifiesto (Brief 4) — solo calcula/muestra cobertura; no la persiste ni la clasifica.
- No regenera el grafo (Brief 6 / validación aparte) — solo mide y anota el cambio de universo.
- No parsea tablas (Brief 3).
- No borra el huérfano C-16 (usuario).

## Criterio de "hecho"

- `RECON-conversion.md` responde las tres preguntas de la fase 1 con evidencia del repo, no
  suposiciones.
- El script construye `parse_ready/` con política (a): fallos de conversión contados y nombrados,
  nunca silenciados; idempotente; exit code correcto.
- `verify_parser.py` lee de `parse_ready/` y **reporta cobertura reconciliada con motivo por cada
  ausencia** — se acabó el «OK» parcial silencioso.
- Los tests pasan sin red, sin LibreOffice, sin `data/`; incluyen el caso de ausencia (un activo que
  no llega a processed sale nombrado, no callado).
- Guion UAT entregado, con verificación explícita de que ningún CORE/RELATED quedó fuera y de que el
  universo nuevo (≈26) está medido contra el viejo (14).
