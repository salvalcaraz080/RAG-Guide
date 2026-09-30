# pre-M4 · ronda 2 · Mapa de reubicación — Brief de documentación (C1 de la ronda 2)

> Para cada bloque que sale de `CLAUDE.md` o `ARCHITECTURE.md`: **a dónde fue**, o «duplicado,
> eliminado». Sintetizar ≠ borrar: nada se pierde sin quedar aquí anotado.
>
> **Tamaños:** `CLAUDE.md` 1.133 → **263** líneas. `ARCHITECTURE.md` 1.441 → **1.511** (crece: es
> el hogar que *recibe*). Citas a brief/sesión/auditoría en `.py`: 227 → **0**. Docstrings de
> producción fuera del estándar: 265 → **0**. Suite: **913 verde**.

---

## 1. Qué salió de `CLAUDE.md`

| Bloque (líneas aprox.) | Destino | Nota |
|---|---|---|
| Estado actual en prosa larga (~55) | **CLAUDE**, condensado a 20 | Las cifras medidas de la híbrida (0,449 → 0,493; puesto 1 → 4) bajan a **ARCHITECTURE §7**, donde vive la decisión abierta que justifican |
| Estructura de carpetas completa, `app/` + `corpus/` + `migrations/` + `tests/` (~110) | **duplicado, eliminado** | `ARCHITECTURE.md` §3 y §5 ya tenían los dos árboles. CLAUDE conserva un mapa de 15 líneas con solo lo que no es obvio (`paths`, `identity`, `docx_text`, `diagnostics`, `groundtruth`) |
| Frontera ejecución / diagnóstico (~6) | **CLAUDE** | Es regla de organización: se queda, reescrita en 3 líneas y con `groundtruth/` como tercer caso |
| Orden del pipeline y comandos (~8) | **duplicado, eliminado** | `corpus/RUNBOOK.md` es su hogar. CLAUDE deja la secuencia en una línea y el enlace |
| Cadena de identidad del corpus (~6) | **duplicado, eliminado** | `ARCHITECTURE.md` §1 ya la dibuja |
| Tabla de progresión de módulos M1/M2/M3 (~8) | **duplicado, eliminado** | `ARCHITECTURE.md` §2. CLAUDE deja el estado en 3 líneas |
| Stack detallado con notas por dependencia (~50) | **duplicado, eliminado** | `ARCHITECTURE.md` §6. CLAUDE deja 8 bullets sin notas |
| **Decisiones consolidadas** (~370) | **ARCHITECTURE §3 y §5**, mayormente **duplicado** | Ver el desglose de §2 abajo: 5 eran únicas y se reubicaron; el resto ya estaba |
| **Decisiones pendientes** (~160) | **ARCHITECTURE §7** | §7 pasa a ser el hogar del detalle; CLAUDE deja 12 titulares y el enlace. §7 invertía la referencia («ver CLAUDE.md para la lista viva»), ya corregido |
| Convenciones de código (~30) | **CLAUDE** | Son reglas. Reescritas con el estándar de tres campos y la prohibición de citar briefs |
| Tests (~90) | **CLAUDE**, condensado a ~35 | Las dos suites, la frontera CC/usuario y el invariante de proveedor son reglas. El detalle de *por qué* aísla cada fixture ya vive en `tests/conftest.py` y `tests/app/conftest.py` |
| Gestión de dependencias (~10) | **CLAUDE** | Regla operativa |
| Variables de entorno, lista completa (~45) | **ARCHITECTURE §6** (subsección nueva) | CLAUDE deja solo las dos con guardarraíl de arranque (`WEB_CONCURRENCY`, `LLM_EVAL_MODE`) y la regla `.env` + `.env.example` |
| Dominio ECSS (~20) | **CLAUDE**, condensado a 6 | Nomenclatura y tiers: contexto mínimo que CC necesita sin abrir otro fichero |
| Grafo de referencias, sección entera (~85) | **duplicado, eliminado** salvo un bloque | `ARCHITECTURE.md` §5 ya tenía diseño, shape de `references.json` y deuda del extractor. **Único no duplicado:** el contrato de runtime («`nx.DiGraph` no es dependencia del camino de petición; la expansión se hará contra un índice de adyacencia») → **reubicado a ARCHITECTURE §5** |
| Checklist de fin de sesión (~30) | **CLAUDE**, condensado a 15 | Regla de cierre |

### Decisiones consolidadas: qué era único

Se comprobó una a una contra `ARCHITECTURE.md` antes de borrar. Ya estaban allí (**duplicado,
eliminado**): régimen exacto sin índice vectorial, dos identidades de documento, `embed ≠ cite`,
`authority` por chunk, `embedding` nullable, migraciones a mano, las cinco guardas de la ingesta,
predicado + oráculo de marcadores, `ORACLE_ALLOW_LIST`, filtros dentro del top-k, `<=>` siempre,
RRF por rango, `rrf_sort_key`, códigos HTTP por culpa, seam de embeddings async, fingerprints por
stage, accesor único del manifiesto, `DERIVED_ARTEFACTS`, anclas de chunking, header sin
identificadores, tablas por mecanismo, numeración resuelta, `accepted_text`, rótulo ≠ cláusula.

**Reubicadas porque no estaban:**

1. `identity.key_to_stem` — comparar nombres entre directorios traduce hacia el extremo con
   pérdida → **ARCHITECTURE §5** (antes de «Numeración de sección»).
2. `count_tokens` como función y la deuda del tokenizador real → **ARCHITECTURE §5** (sustituye a
   la línea obsoleta de `TokenCounter` inyectable) y **§7**.
3. `--check` de staleness en `chunk_all`/`embed_all` → **ARCHITECTURE §5**, antes de la ingesta.
4. Contrato de runtime del grafo (índice de adyacencia, no NetworkX) → **ARCHITECTURE §5**.
5. Nota C5: `ingestion` no corre dentro del contenedor, se decide en M6 → **ARCHITECTURE §5**
   (§7 lo referencia). Es el texto que el brief de limpieza dejó explícitamente para aquí.

---

## 2. Qué salió de `ARCHITECTURE.md`

| Bloque | Destino | Nota |
|---|---|---|
| 44 etiquetas `(S0X brief N)` inline, y las referencias a sesiones como calendario («es S09», «trabajo de S11») | **CHANGELOG** | El histórico ya está allí sesión a sesión. Donde la frase necesitaba el dato, se reescribió como estado: «es el cable que falta», «sigue sin decidir» |
| Narrativa de la retirada de la cache (~14 líneas: qué se eliminó, en qué orden, qué se perdió) | **CHANGELOG** + **Guide** | El estado («no hay cache, no vuelve sin hit rate ni clave construible») se queda en §3, en 8 líneas. La lección agnóstica ya es nodo cerrado de `RAG-GUIDE.md` fase 4 |
| Incidente del gate de conteo de marcadores (~5 líneas: el `dropped != 121`, los cinco marcadores servidos) | **CHANGELOG** | Se queda el invariante vivo, en una frase: «un instrumento que se mide consigo mismo no puede detectar su propia omisión» |
| Anécdota de verificación manual del `sse_client` (~2) | **CHANGELOG** | La propiedad («testeable como Python plano») se queda |
| `scripts/*_groundtruth.py` en §7 | **corregido** | Ruta movida por el brief de limpieza: `corpus/groundtruth/` |

---

## 3. Qué salió del código

| Bloque | Destino |
|---|---|
| 227 líneas con citas a brief/sesión/auditoría (`app/`, `corpus/`, `frontend/`, `scripts/`, `tests/`, `migrations/`) | **CHANGELOG**: el «qué brief lo decidió» ya estaba allí. En el código se conservó el **porqué** cuando lo había, reescrito sin fecha ni número |
| Docstrings que mezclaban qué-hace con por-qué e historia (157 reescritos) | **docstring**, partidos en `Def:` / `Why:` / `Args:` / `Return:` |
| 108 funciones sin docstring | **docstring**, escritos nuevos |

---

## 4. Candidatos agnósticos para la Guide — **anotados, no escritos**

Graduar es paso de juicio del chat; aquí solo se listan los que aparecieron al enrutar:

1. **Un hogar por decisión.** Contar algo en dos sitios no es redundancia inofensiva: es dos
   sitios que pueden divergir, y el que se lee primero gana sin que nadie lo decida.
2. **Un instrumento que se mide consigo mismo no puede detectar su propia omisión.** Vale para el
   gate de marcadores, para la conservación de tablas y para cualquier chequeo derivado de la
   misma fuente que verifica. (Puede que ya esté graduado; si no, es el candidato más fuerte.)
3. **La conservación se mide contra la fuente, no contra la salida.**
4. **Un chequeo que sale rojo siempre deja de leerse**: un aviso blando no puede decidir el
   código de salida.
5. **Cuando dos grafías no se pueden reconciliar, se traduce hacia la lossy** — es la única
   dirección bien definida.
6. **Colapsar una abstracción retira deuda, pero no cierra la pregunta que la motivó**: hay que
   anotar la pregunta donde vive la deuda, no en el código que se borra.
7. **Validar antes de gastar**: el orden entre validación y llamada cara es parte del diseño.

---

## 5. Lo que queda pendiente de este brief

- **La pasada de estilo sobre `ARCHITECTURE.md` es parcial.** Se aplicó el enrutado (des-etiquetado
  completo, narrativa histórica fuera, huecos rellenados) y se recortaron los tres pasajes
  claramente históricos, pero **no** una reescritura frase a frase de las 1.511 líneas. Las
  secciones §3 y §5 siguen siendo densas y admiten acortado sin perder contenido.
- **`CHANGELOG.md` no se reordenó.** Recibe lo que sale de los otros dos, pero su propia
  estructura (entradas por versión) queda como está.
