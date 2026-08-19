# Reconocimiento previo a la conversión `.doc`→`.docx` — S06 brief 2, fase 1

> Las tres preguntas de la fase 1, respondidas con **evidencia del repo y del entorno**, no con
> suposiciones. Conclusión: **el plan del brief no cambia**, pero el reconocimiento aporta dos
> detalles operativos que habrían roto la implementación ingenua (§3).

---

## 1. ¿Existe ya integración de LibreOffice en el repo?

**No. Nunca la hubo en código.** La duda era legítima —el corpus se exploró con documentos ya
convertidos— pero la conversión se hizo **a mano**, y lo que quedó fue el resultado, no la ruta.

Evidencia:

- `grep -i "libreoffice|soffice|unoconv|convert.to"` sobre todo el repo (`.py`, `.md`, `.toml`,
  `.yml`, Dockerfiles) devuelve **cero invocaciones**. Todas las apariciones son prosa:
  `CLAUDE.md` (decisión y pendiente), `ARCHITECTURE.md` (tabla de stack), `CHANGELOG.md`
  (histórico), y el docstring de `corpus/parser.py:10-11` («*.doc files must be converted to
  .docx with LibreOffice headless before parsing*») — que documenta un requisito **externo** al
  código, no una capacidad de éste.
- `CHANGELOG.md` 0.1.0: *«LibreOffice headless consolidado como solución de conversión (se
  integrará en Docker)»* — decisión, sin implementación. Sigue sin integrarse: ni el `Dockerfile`
  raíz ni el de `frontend/` lo instalan.
- `CHANGELOG.md` 0.2.0 registra `scripts/inspect_docx.py`, *«script temporal de PoC para
  inspeccionar la estructura jerárquica de documentos ECSS **convertidos con LibreOffice**»*.
  Ese script **ya no existe**: se borró en el commit `20f5899`.

**Y el PoC explica el huérfano C-16.** Recuperando el fichero borrado
(`git show 20f5899^:scripts/inspect_docx.py`), su primera línea de trabajo es:

```python
doc = Document("data/raw/ECSS-E-ST-70-01C16April2010.docx")
```

Es exactamente el fichero que la auditoría del Brief 1 marcó como huérfano y cuya procedencia
infirió como «descarga manual + conversión LibreOffice hecha a mano». **Queda confirmado con
evidencia directa**: es el residuo del PoC de 0.2.0. El script solo *consumía* un `.docx` ya
convertido; la conversión ocurrió fuera del repo y no dejó rastro reutilizable.

**Consecuencia para este brief:** no hay ruta previa que reutilizar ni convención de nombres que
respetar. Se crea la primera —y única— ruta de conversión, que es justo lo que la lección de C-4
pide (no dejar dos rutas conviviendo sin declarar cuál es la buena).

---

## 2. ¿Sobre cuántos documentos se construyó el grafo — 14 o 26?

**Sobre 14.** Confirmado, no inferido, por tres vías independientes:

1. **El código.** `corpus/graph.py:330` construye el universo con
   `processed_dir.glob("*.json")` sobre `data/processed/`, que contiene 14 JSON (+
   `references.json`). El grafo no puede ver nada que el parser no haya producido.
2. **El histórico.** `CHANGELOG.md` 0.5.0: *«14 DOCX parseados (12 `.doc` legacy pendientes de
   conversión LibreOffice)»* y *«**KG inicial (14 docs)**: 251 referencias, 300 nodos, 218
   aristas únicas»*. El número de documentos está escrito al lado del de aristas.
3. **El estado actual.** `CLAUDE.md` registra 273 referencias / 326 nodos / 247 aristas — una
   regeneración posterior (tras las mejoras del parser en anexos), pero **sobre los mismos 14
   documentos**.

**Sí: el grafo también sufre C-15.** Se construyó sobre el 54% del corpus, y ni el grafo ni sus
estadísticas lo dicen.

### Recomendación sobre regeneración

**Regenerar sí, pero no en este brief** — es lo que el brief ya anticipa, y el reconocimiento lo
confirma. Razones para no hacerlo aquí:

- El extractor de referencias tiene **deuda conocida sin validar** (ventana de asociación
  cláusula↔doc de ±80 chars, `break` que empareja la primera cláusula en orden de texto y no la
  más cercana). Regenerar sobre casi el doble de documentos amplificaría esos falsos positivos
  sin que nadie mida cuántos. Regenerar a ciegas convierte un problema medido en uno mayor y no
  medido.
- La validación contra ground truth etiquetado (~30–50 referencias) es prerequisito declarado en
  `CLAUDE.md` antes de construir retrieval encima. Ya existen
  `scripts/sample_for_groundtruth.py` y `scripts/score_groundtruth.py`, lo que sugiere que ese
  trabajo está empezado.

**Lo que sí hay que anotar y medir (y este brief entrega):** el universo pasa de 14 a 26
documentos — aunque para el grafo el número que cuenta es **20**, no 26; ver «Medición del
cambio de universo» abajo. Eso afecta a dos decisiones abiertas:

- La política de expansión por grafo. `CLAUDE.md` mide que 135 de 273 referencias (49%) tienen
  `target_clause_id` vacío. **Hipótesis que ahora es comprobable:** parte de esas referencias
  colgando apuntan a documentos que no estaban en el corpus por ser `.doc`. Entre los 12 que
  entran hay tres de los más referenciados del histórico —`ECSS-M-ST-40` (16 menciones),
  `ECSS-E-ST-10-06`, `ECSS-Q-ST-30-02`—, así que el porcentaje debería mejorar solo. La decisión
  sobre qué inyectar cuando el target es un documento entero merece tomarse **con los números
  nuevos**, no con los de 14 documentos.
- El chunking, por la misma razón: el universo sobre el que se calibra cambia.

---

## 3. ¿Qué herramienta de conversión está disponible?

**LibreOffice 26.2.3.2**, instalado en el entorno del usuario. Pero el reconocimiento destapa
**dos detalles que habrían roto la implementación ingenua** del brief
(`soffice --headless --convert-to docx`):

### 3.1 No está en el PATH

```
FOUND: C:\Program Files\LibreOffice\program\soffice.exe
soffice not on PATH
libreoffice not on PATH
```

Invocar `soffice` a secas falla con `FileNotFoundError`. El script debe **localizar** el
ejecutable: variable de entorno explícita → PATH → rutas conocidas por plataforma.

### 3.2 En Windows hay que usar `soffice.com`, no `soffice.exe`

Este es el detalle que importa. Los dos binarios existen y pesan lo mismo:

```
soffice.com  523688
soffice.exe  523688
```

Pero no se comportan igual:

| invocación | salida | exit code |
|---|---|---|
| `soffice.exe --version` | *(nada)* | — |
| `soffice.com --version` | `LibreOffice 26.2.3.2 70e089b…` | 0 |

`soffice.exe` está marcado como aplicación GUI: **se desacopla de la consola y retorna
inmediatamente**, sin esperar a terminar y sin escribir en stdout/stderr. Un `subprocess.run()`
contra él devolvería exit 0 al instante, con la conversión aún sin hacer o directamente fallida,
y el script contaría 12/12 convertidos mientras `parse_ready/` se llena a destiempo o no se
llena. **Sería un fallo silencioso de manual** — exactamente lo que S06 lleva dos briefs
eliminando, reintroducido por una diferencia de extensión.

`soffice.com` es el envoltorio de consola: espera, propaga exit code y escribe a stdout.

**Decisión:** en Windows se prefiere `soffice.com`; en Linux/macOS, `soffice`/`libreoffice`. Y
—porque ni con `soffice.com` es de fiar el exit code de LibreOffice— la verificación real no es
el código de salida sino **que el `.docx` de salida exista y pase los magic bytes** del Brief 1b.
LibreOffice tiene el hábito documentado de devolver 0 sin producir fichero.

### 3.3 Riesgo operativo: una instancia abierta bloquea la conversión

LibreOffice comparte perfil de usuario entre procesos. Si el usuario tiene LibreOffice abierto,
la invocación headless puede quedarse colgada o abortar en silencio. Se mitiga pasando un
**perfil de usuario temporal y aislado** (`-env:UserInstallation=file:///…`), de modo que la
conversión no dependa de si hay un Writer abierto en otra ventana. Sin esto, el guion UAT
tendría que incluir «cierra LibreOffice antes de ejecutar», que es una instrucción que alguien
se saltará.

---

## Medición del cambio de universo (post-conversión, 2026-07-29)

Los 12 `.doc` convertidos parsean. Lo que sigue es la medida que el brief pide dejar anotada,
porque cambia el universo del grafo (Brief 6) y del chunking, y no se puede tomar ninguna de
esas decisiones con los números de 14 documentos.

**Fidelidad de la conversión: buena.** Los documentos convertidos traen secciones, requisitos y
—dato revelador— marcas de revisión recuperadas (`tracked-change paras recovered`: 47 en
`E-HB-40A`, 128 en `E-ST-70-31C`, 42 en `M-ST-40C`). Que sobreviva el control de cambios
significa que LibreOffice preservó la estructura XML fina, no solo el texto plano.

| Documento nuevo | Requisitos | Secciones |
|---|---|---|
| `ECSS-E-ST-70-31C` | 187 | 242 |
| `ECSS-M-ST-40C Rev.1` | 237 | 185 |
| `ECSS-Q-ST-30-02C` | 181 | 87 |
| `ECSS-E-ST-70-01C` | 140 | 67 |
| `ECSS-M-ST-10C Rev.1` | 99 | 130 |
| `ECSS-E-ST-10-06C` | 54 | 63 |
| `ECSS-E-HB-40A` | 39 | 425 |
| `ECSS-Q-HB-80-01A` | **0** | 51 |
| `ECSS-Q-HB-80-02 Part 1A` | **0** | 184 |
| `ECSS-Q-HB-80-02 Part 2A` | **0** | 76 |
| `ECSS-Q-HB-80-04A` | **0** | 57 |
| `ECSS-M-ST-80C` | 66 | 68 |

**Totales:** +1.003 requisitos sobre los 6.919 de los 14 originales (**+14,5%**) y +1.635
secciones sobre 3.557 (**+46%**). El corpus queda en **7.922 requisitos y 5.192 secciones**
sobre 26 documentos.

La asimetría entre ambos porcentajes es el dato: los documentos que entran son **pesados en
secciones y ligeros en requisitos**. En requisitos el crecimiento es modesto porque
`ECSS-E-ST-70-41C`, ya presente, aporta él solo 3.185; en secciones, casi la mitad del corpus
es nueva.

### El hallazgo que importa para el grafo: seis documentos aportan cero referencias

Cuatro de los handbooks nuevos dan **0 requisitos**, y con los dos que ya estaban
(`E-HB-40-01A`, `Q-HB-80-03A Rev.1`) son **6 de 26 documentos sin ningún requisito**.

No es un fallo del parser: los handbooks ECSS son informativos por definición y no llevan
párrafos `require:level1`. Pero tiene una consecuencia directa que conviene ver antes de
regenerar nada: **`extract_references` deriva las referencias del texto de los requisitos**
(`corpus/graph.py:150-157`). Un documento sin requisitos no puede emitir ninguna
arista saliente.

Es decir: el universo del grafo **no pasa de 14 a 26, pasa de 14 a 20**. Esos seis documentos
entran al corpus como contenido recuperable —sus **591 secciones** son chunkables y sí pueden
ser *destino* de una referencia— pero son mudos como fuente.

Dos consecuencias para decisiones abiertas:

- **Para la expansión por grafo (Brief 6):** la hipótesis de que la conversión mejoraría el 49%
  de referencias con `target_clause_id` vacío sigue en pie —entran `M-ST-40`, `E-ST-10-06` y
  `Q-ST-30-02`, de los más referenciados—, pero el efecto vendrá de los 6 estándares nuevos, no
  de los handbooks.
- **Para el chunking:** `E-HB-40A` tiene 425 secciones y 39 requisitos, y
  `Q-HB-80-02 Part 1A` tiene 184 secciones y ninguno. Una estrategia de chunking anclada al
  requisito deja fuera casi todo el contenido de los handbooks. Si el producto va a responder
  con guía además de con norma, el chunker necesita tratar la sección como unidad, no solo el
  requisito.

### Hallazgos estructurales abiertos (material del Brief 3)

`verify_parser.py` reporta **23 de 26** documentos sin hallazgos. Los tres restantes:

- **`ECSS-Q-ST-30-02C`** — `INV4` ×4: `E.0.1` y `E.0.2` sin padre `E.0` (ídem en `F`). Un
  heading de nivel 3 aparece en el anexo sin nivel 2 previo, así que el contador intermedio
  queda a cero. **La numeración de esas cláusulas no es de fiar, y numeración es identidad de
  cita.**
- **`ECSS-E-HB-40A`** — `INV4`: `0.0.0.0.0.1` sin padre. Mismo defecto en su forma extrema: un
  `Heading 6` antes de cualquier heading superior.
- **`ECSS-E-ST-10C Rev.1`** — `INV5(soft)`, 1 requisito en minúscula. El propio check lo declara
  señal blanda.

Los dos `INV4` son la misma clase de defecto y **solo aparecen en documentos convertidos o en
handbooks**, no en los estándares nativos. Qué debe producir una cabecera profunda huérfana
—ignorarla, colgarla de la última sección abierta, o numerarla con el prefijo real— es decisión
de diseño del parser, no de este brief.

---

## Resumen

| Pregunta | Respuesta | Efecto en el plan |
|---|---|---|
| ¿Integración LibreOffice previa? | No, nunca en código. El PoC de 0.2.0 convirtió a mano y su script se borró | Ninguno: se crea la primera ruta, sin convenciones heredadas que respetar |
| ¿Grafo sobre 14 o 26? | **14**, confirmado por código, CHANGELOG y estado actual | El grafo sufre C-15. Se mide el cambio de universo; **no** se regenera aquí |
| ¿Herramienta disponible? | LibreOffice 26.2.3.2, **fuera del PATH**, y en Windows hay que usar `soffice.com` | Localización explícita del binario + verificación por artefacto, no por exit code |

**No hay razón para parar.** El plan de las fases 2 y 3 se ejecuta tal cual, con los tres
detalles operativos de §3 incorporados.

**Regalo del reconocimiento:** la procedencia del huérfano C-16 queda cerrada con evidencia
directa. Es el residuo del PoC de 0.2.0, no un fallo del downloader — sigue tocando borrarlo al
usuario, pero ya se sabe exactamente qué es y por qué está ahí.
