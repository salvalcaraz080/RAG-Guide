# pre-M4 · ronda 2 · Decisiones — Brief de limpieza (C2–C7, D1, D2, menor, de la ronda 2)

> Log crudo de lo decidido al implementar, etiquetado **[AGNÓSTICO]** (transferible a cualquier
> pipeline RAG) o **[DOMINIO]** (específico de ECSS / de este proyecto).
>
> **Naturaleza del brief:** ejecución de pendientes ya ratificadas, no decisión nueva. Ninguna
> tarea re-chunkea ni re-embebe.
>
> **Verificación transversal:** `uv run pytest` **913 verde** (desde 906), ruff limpio salvo el
> `E402` deliberado de `app/main.py`, y el guardián de identidad intacto:
>
> | momento | `input_fingerprint` | chunks (hash de los 13.876) | re-parseo byte a byte |
> |---|---|---|---|
> | baseline (antes del brief) | `ab3fce8b5cb1a0d4` | `ce96a65165dabd7b` | — |
> | tras C2 (el único que toca el chunker) | `ab3fce8b5cb1a0d4` | `ce96a65165dabd7b` | — |
> | final (tras todos los bloques) | `ab3fce8b5cb1a0d4` | `ce96a65165dabd7b` | idéntico |
>
> El grafo (329 referencias), el dry-run de ingesta (26 documentos / 13.695 chunks) y el reporte
> salen también idénticos al baseline.

---

## 1. Colapsar una abstracción **retira** deuda; no la cierra — **[AGNÓSTICO]** (C2)

`TokenCounter` era una clase abstracta con **una** implementación, justificada por «el modelo de
embedding sigue abierto». Esa premisa murió cuando se eligió `text-embedding-3-small`, así que la
jerarquía pasó a ser ceremonia: dos clases para una fórmula de una línea.

Lo que se colapsó y lo que **no**:

- **Se colapsó la forma:** `count_tokens(text)`, más un alias `TokenCounter = Callable[[str], int]`
  para que los tests sigan fijando ventanas pequeñas con un contador propio. El punto de inyección
  no es especulativo: tiene usuarios hoy.
- **No se tocó la heurística.** Y ahí está el matiz que evita confundir limpieza con cambio: pasar
  al tokenizador real **mueve los cortes de sección**, y eso re-chunkea y re-embebe el corpus. Es
  una operación cara, no una mejora gratis.

La pregunta viva se anotó **donde vive la deuda** (pendiente de ventana y contador en
`CLAUDE.md`), agrupada con el fix de marcadores aguas arriba, que fuerza exactamente el mismo
re-embedding. Anotarla en el docstring de la función que se borraba la habría enterrado.

**El fingerprint es lo que separa las dos cosas:** si colapsar hubiera cambiado un corte, el
digest de `chunks.json` lo habría gritado. Salió idéntico, así que fue refactor y no cambio de
corpus.

---

## 2. Cuando dos grafías no se pueden reconciliar, se traduce hacia la lossy — **[AGNÓSTICO]** (C3)

El mismo documento se escribe de tres maneras: `ECSS-E-ST-40C Rev.1` (inventario),
`ECSS-E-ST-40C_Rev_1` (`raw/`, `parse_ready/`) y `ECSS-E-ST-40C_Rev.1` (`processed/`). Tres sitios
comparaban nombres entre directorios y cada uno se inventó su normalización: dos borraban toda la
puntuación (`re.sub(r"[^A-Za-z0-9]", "", name).upper()`) y el tercero hacía
`entry.key.replace('.', '_')`.

Borrar puntuación **funciona y es falso**: compara «algo parecido» y acierta por la forma de los
identificadores ECSS, no por una regla. La alternativa correcta es una **traducción**, y la
dirección no es libre: `sanitize_filename` pierde el punto y no es invertible, así que solo se
puede traducir hacia el extremo con pérdida. Eso es `identity.key_to_stem`, que acepta las tres
grafías y es idempotente.

Lo que **no** se hizo, a propósito: renombrar `data/raw/` para unificar de verdad. Ese directorio
es «solo lo que dio el servidor», inmutable por diseño; renombrarlo para que cuadre con una
convención interna es exactamente la clase de comodidad que rompe la trazabilidad de origen.

Queda intacta la pendiente ligada —el inventario y el manifiesto como dos fuentes de «qué debería
existir»—: este brief unificó la *traducción*, no la *fuente*.

---

## 3. Un instrumento que escribe no es un diagnóstico — **[DOMINIO]** (C4)

El arnés de ground truth vivía en `scripts/`, que la frontera del proyecto reserva para
investigación puntual, pero se re-ejecuta cada vez que alguien etiqueta (RUNBOOK §8). El destino
«obvio» era `corpus/diagnostics/`, y es el incorrecto: la regla de esa carpeta es **no escribir
nada en `data/`**, y estos scripts escriben las plantillas CSV que el humano rellena.

De ahí un tercer sitio, `corpus/groundtruth/`, con la frontera explicada en su `__init__`: mide
como un diagnóstico pero produce como un stage. Efecto lateral: al ser paquete desaparecen los
`sys.path.insert` + `# noqa: E402` de los tres módulos y de su test.

**Criterio transferible:** la pregunta que ordena el árbol no es «¿esto mide o transforma?» sino
«¿esto escribe, y se vuelve a ejecutar?».

---

## 4. La señal de staleness vale aguas arriba de la guarda — **[AGNÓSTICO]** (C7)

`is_stale()` existía en `chunk_all` y `embed_all` con tests y **ningún llamante**: código probado
e inalcanzable. La tentación era borrarlo por muerto, porque «la ingesta ya rechaza un par
incoherente».

No es lo mismo:

- La guarda de ingesta compara `chunks.json` con el sidecar de embeddings. Si `data/processed/` se
  movió bajo **los dos**, los dos siguen siendo coherentes entre sí y la guarda no ve nada.
- `chunk_all --check` compara los chunks guardados con el contenido actual de `processed/`, que es
  justo el eslabón que la guarda no cubre.
- Y llega **antes de pagar**: la guarda descubre el problema cuando los vectores ya están
  comprados; `--check` lo dice antes.

Expuesto con tres códigos de salida y un «no se sabe» que no se disfraza: **0** al día, **1**
regenerar, **2** no comprobable (sin artefacto o sin fingerprint guardado).

---

## 5. Quitar una dependencia se confirma ejecutando, no leyendo — **[AGNÓSTICO]** (C6)

Ningún `import anthropic` en el repo y litellm no la declara, pero eso solo descarta el uso
**estático**. Un cliente HTTP puede importar el SDK de forma perezosa dentro de un handler y solo
en runtime. Así que se quitó y se hizo **una llamada real**: respondió
`claude-haiku-4-5-20251001`, 23/15 tokens, y `sys.modules` no contenía `anthropic` durante la
llamada.

`python-dotenv` era redundante: llega igualmente por `pydantic-settings`, `fastapi[standard]` y
`litellm`. Se quita de las directas, no del entorno.

---

## 6. Un chequeo que sale rojo siempre deja de leerse — **[AGNÓSTICO]** (D2)

`verify_parser` salía con código 1 sobre el corpus real por un `INV5(soft)`: un requisito que
empieza en minúscula. El propio código lo llamaba «señal blanda, no fallo» y a la vez lo metía en
la lista que decide el veredicto.

El daño no es el 1: es que **enseña a ignorar el instrumento**. Un verificador permanentemente
rojo deja de distinguir entre «esto siempre está así» y «esto acaba de romperse», y en un pipeline
cuyo trabajo es hacer visibles las ausencias eso es perder el instrumento entero.

Separado en `is_soft`/`hard_failures`: los blandos se imprimen como `SOFT`, se listan aparte y no
tocan el código de salida. Hoy el comando sale **0** con el aviso todavía visible.

---

## 7. Quitar una rama muerta afirma algo; hay que anotar qué — **[AGNÓSTICO]** (D1)

La rama «chunk sin vector → `embedding` NULL» de `build_rows` era inalcanzable: la guarda de
conteo rechaza el par antes. Quitarla es correcto, pero **no es neutral**: afirma «no soportamos
fila sin vector».

Puede dejar de ser cierto —un chunk demasiado largo para embeber, o una unidad solo-léxica en
S09+—, así que la reversión queda escrita en el docstring de la función, junto a la razón de que
la columna siga siendo nullable. La información que se pierde al borrar código no es la rama: es
por qué estaba.

---

## 8. Validar antes de gastar — **[AGNÓSTICO]** (menor)

`retrieval.search` embebía la query y **después** validaba los filtros, así que un
`authority="mandatory"` costaba una llamada al proveedor para acabar en un 400. Se reordena.

Las sentencias siguen validando por su cuenta: esa validación protege a quien construya un
`build_query` directamente, y duplicarla es barato. Lo que se añade arriba no es la regla, es el
**momento**. El test lo fija con un cliente de embeddings que revienta si alguien lo llama.

---

## 9. Lo que NO se tocó, y por qué — **[DOMINIO]**

- **C5 — `ingestion` no corre dentro del contenedor.** Es un batch que importa `corpus/`, y la
  imagen solo copia `app/`. Dónde debe vivir (`jobs/` de nivel superior vs meter `corpus/` en la
  imagen) es diseño de imagen y microservicios: **se decide en M6**, que está en el temario. Aquí
  no hay cambio de código, solo constancia.
- **`raw/` no se renombra** (C3), la fuente inventario-vs-manifiesto sigue diferida, y el
  tokenizador real sigue agrupado con el fix de marcadores.

---

## 10. Traspaso al brief de documentación

Este brief dejó `ARCHITECTURE.md` **sin tocar a propósito** (un solo brief debe reescribirlo).
Queda desactualizado en:

- **línea ~1047** — «`TokenCounter` inyectable, heurístico por defecto: `tiktoken` es específico de
  OpenAI y el modelo de embedding sigue abierto». Hoy es `count_tokens`, el modelo está decidido y
  la pregunta viva es el coste del re-embedding.
- **línea ~1438** — «`scripts/*_groundtruth.py`» → `corpus/groundtruth/`.
- **C5** — falta el texto que documente que `ingestion` no corre en el contenedor y que la
  decisión es de M6.
- Conviene mencionar `key_to_stem` donde se explican las identidades, y `--check` donde se
  describen los stages de chunking y embedding.
