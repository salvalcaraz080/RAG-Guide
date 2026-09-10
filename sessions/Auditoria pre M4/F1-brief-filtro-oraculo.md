# Fix-brief F1 — Filtro de marcadores no-citables + oráculo de completitud

**Primero de cuatro fix-briefs** del triaje del Paso D (F1→F2→F3→F4). Cierra el cluster de mayor
severidad: **T1-1** (el filtro de C2 deja pasar 5 marcadores, servidos hoy como `requirement/
normative`), **T1-3** (el gate de conteo no puede detectar que el predicado se quede corto),
**T1-8** (`derive_documents` supone que ningún documento pierde todos sus chunks) y **O-2** (55
chunks `None.`/`None`, mismo riesgo).

**Insumo:** `sessions/Auditoria pre M4/pasoD-diagnostic-report.md`, hallazgos T1-1/T1-3/T1-8 y O-2.

**Naturaleza:** fix-brief. CC implementa lo listado, corre la verificación, escribe decisiones +
CHANGELOG + estado en `CLAUDE.md`. **No** re-embed, **no** re-chunk (todo es política de carga; los
chunks descartados siguen en `chunks.json`/`embeddings.npy`, reversible). No commitea.

**Ficheros:** `app/services/ingestion.py`, `tests/app/test_ingestion.py`, `.claude/CLAUDE.md`,
`CHANGELOG.md`, y `sessions/Auditoria pre M4/F1-decisions.md`.

---

## El principio de fondo (por qué no es whack-a-mole)

El gate `dropped != 121` **se mide con el propio predicado**, así que es estructuralmente ciego a un
predicado que sub-empareja: por eso T1-1 se coló con la ingesta en verde. Añadir las formas que
faltan y subir el número deja el mismo agujero para el siguiente corpus. La corrección es separar
**dos instrumentos distintos**:

- **El predicado de descarte** — exacto, enumera las formas que *sabemos* descartar. Es lo que filtra.
- **El oráculo de completitud** — más ancho e **independiente**, corre sobre los **supervivientes** y
  exige que ninguno parezca un marcador. Es un instrumento distinto del predicado, así que caza lo
  que el predicado se deja. Sustituye al gate de conteo.

El oráculo no es desechable: cuando se arregle aguas arriba (parser, T2-4 diferido), el **mismo
oráculo** pasa a asegurar "no se produce ningún chunk-marcador" — es el test de aceptación de ese fix
futuro. Constrúyelo como función importable + test, no inline.

---

## Tarea 1 — Ampliar el predicado de descarte (T1-1 + O-2)

Hoy: `text == "-" or text.startswith("<<deleted")`. Se le escapan (todas en base y citables ahora):

- `<< deleted, …>>` — espacio tras `<<` (rompe el `startswith`).
- `>>deleted>>` — typo del documento fuente.
- Los 55 `None.` / `None` — marcadores de "sin contenido" (O-2), decidido: no se sirven como cita.

**Renombrar** `is_retracted`/`drop_retracted` → algo que no mienta sobre su alcance (ya no es solo
"retirada"): p. ej. `is_non_citable_marker` / `drop_non_citable`. *(La Guide: un nombre nombra lo que
distingue; "retracted" ahora sub-declara.)*

El predicado se aplica sobre el **cite_text completo normalizado** (no substring — el marcador *es*
todo el cite_text), en categorías nombradas para que el código documente el porqué:
- bare dash: cite_text normalizado `== "-"`.
- retirada/movido: encerrado en `<<…>>` o `>>…>>` con `deleted`/`moved` dentro, **tolerante al
  espaciado** (normalizar whitespace antes de comparar).
- sin-contenido: cite_text normalizado ∈ {`None`, `None.`}.

**Ojo con el sobre-emparejamiento:** compara el cite_text *entero*, nunca subcadena — una cláusula
real que mencione "deleted" o "none" dentro de texto sustantivo no debe caer.

---

## Tarea 2 — Sustituir el gate de conteo por el oráculo de completitud (T1-3)

**Quitar** `EXPECTED_RETRACTED = 121` y el gate `dropped != 121`. El conteo se **loguea** (para la
trayectoria: cuántos se descartaron) pero **no se asserta**. Esto elimina de paso el segundo hueco de
T1-3 (el comentario ataba 121 a `corpus_version` pero `drop_*` nunca lo leía — atadura documental, no
verificada): sin número mágico no hay atadura que mantener.

**El oráculo** (`assert_no_marker_survivors` o similar): tras filtrar, recorre los supervivientes con
una heurística **deliberadamente más ancha** que el predicado de descarte y exige **cero**
coincidencias. Cualquier superviviente que la dispare **detiene la ingesta y lo nombra** (misma
disciplina que el módulo: "se niega antes de escribir").

- La heurística es más ancha: tras normalizar agresivamente (strip de `<>-.` y whitespace, lower),
  marca si el residuo queda **vacío** o **sin palabras de contenido** (solo tokens de marcador:
  `deleted`, `moved`, `none`, `covered by`, `see`, `refer to`…). Que sea más ancha que el predicado
  es el punto — si fuera igual, solo reconfirmaría el predicado y no cazaría nada nuevo.
- **Falsos positivos del oráculo:** una cláusula real corta que la heurística marque va a una
  **allow-list explícita** (excepciones revisadas a mano), no se relaja la heurística. Para *este*
  corpus el objetivo es que la allow-list quede **vacía** — si el predicado caza todos los
  marcadores, el oráculo sobre supervivientes debe dar cero sin excepciones.

**Auto-test del oráculo** (que prueba que es un instrumento independiente, no un eco del predicado):
con el predicado de descarte **desactivado**, el oráculo debe marcar **todos** los ~181 marcadores;
con el predicado activo, debe marcar **cero** supervivientes. Si con el predicado desactivado el
oráculo no los ve todos, el oráculo no es más ancho que el predicado y no sirve.

---

## Tarea 3 — Endurecer `derive_documents` / `build_rows` (T1-8)

Tras filtrar, `documents` se deriva de los chunks supervivientes; un documento cuyos chunks fueran
**todos** marcadores desaparecería de `ids` y `build_rows` reventaría con `KeyError` — o peor, se
cargaría sin filas sin que nada lo nombre. Hoy no pasa (el más afectado, `ECSS-E-HB-40A`, 497→446),
pero al ampliar el filtro (Tarea 1) el margen baja.

**Fix por construcción** (nodo "all-or-nothing" de la Guide: *25 de 26 no es "casi", es un corpus
más pequeño ocupando el sitio de uno completo*): asertar que **todo documento declarado que entró al
filtro conserva ≥1 chunk**; si alguno cae a cero, **error ruidoso que lo nombra** antes de tocar la
base. Eso hace el `KeyError` imposible por construcción y convierte un fallo silencioso en uno que se
ve.

---

## Verificación

- Los 5 marcadores de T1-1 y los 55 `None.` **ya no están en la base** tras re-ingestar; `POST
  /api/v1/search` (`mode=hybrid`, query «requirement moved deleted») **no** devuelve ninguno.
- **El oráculo sobre supervivientes da cero**, allow-list vacía. Este es el gate de completitud —
  **no** un conteo.
- El auto-test del oráculo pasa (ve todos los marcadores sin predicado; cero con predicado).
- `derive_documents`: un caso sintético "documento con todos los chunks marcados" dispara **error
  ruidoso nombrando el doc**, no `KeyError`.
- **Reemplazar el test tautológico** `test_markers_are_recognised`: sus 6 casos venían todos de lo
  que el predicado ya cazaba. Los nuevos incluyen las formas antes fallidas (`<< deleted…>>` con
  espacio, `>>deleted>>`, `None.`) y un test del oráculo (inyectar un superviviente con pinta de
  marcador → la ingesta se detiene y lo nombra).
- **No reintroducir un conteo como gate.** El total resultante rondará **~13.695** (13.755 − 5 − ~55);
  **reportarlo** como observabilidad, pero el gate es el oráculo, no esa cifra. El número exacto lo
  fija el predicado+oráculo, no se prefija.
- Re-ingesta: `TRUNCATE` + recarga (la ingesta es idempotente por upsert, no borra — mismo motivo que
  en C2).

## Docs (las escribe CC al cerrar)

- `CLAUDE.md`: **cerrar el pendiente** de los "2 párrafos `None.`" — resuelto como marcadores
  sin-contenido (55, filtrados en carga). Reformular la entrada del filtro: predicado + oráculo de
  completitud (no conteo). Actualizar la cifra de estado a la resultante.
- `CHANGELOG.md`: entrada de F1.
- `F1-decisions.md`: log crudo, etiquetado agnóstico/dominio. Anotar explícitamente que **el oráculo
  es el test de aceptación del fix aguas arriba diferido (T2-4)** — para que quien lo retome no lo
  reescriba.

## Fuera de alcance

No re-embed, no re-chunk, no parser (el fix robusto aguas arriba sigue diferido — dispara re-chunk +
re-embed, y el oráculo de F1 es justo lo que lo hará verificable cuando llegue). No tocar retrieval
ni generación. No reintroducir gate de conteo bajo ninguna forma.
