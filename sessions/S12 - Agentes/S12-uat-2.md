# S12 · UAT — Brief 2: expansión por referencia en la interfaz

> **Por qué es tuyo:** caso 2 de la frontera (coste real, y la lectura de la respuesta es tuya). CC
> ha verificado el contrato (1.380 tests, con la equivalencia byte a byte con `off`), la medición de
> 4 configuraciones (`S12-decisions-2.md` §5) y a mano las citas de los aspectos resueltos.
>
> **Coste:** 4–5 preguntas a Haiku 5.5, menos de 0,02 USD. Cada respuesta tarda 10–20 s.

## 0 · Preparación

La base local ya tiene la migración 0005 y las etiquetas de tabla. En otra máquina, antes de nada:

```powershell
uv run alembic upgrade head
uv run python -m app.services.table_labels
```

Activa la expansión por referencia: en `.env`, `RETRIEVAL_XREF=clause` (a mano en el editor), y:

```powershell
docker compose up -d --force-recreate api
```

Abre `http://localhost:8501`. Para ver qué añadió la expansión en cada respuesta:

```powershell
docker compose logs api --since 10m 2>&1 | Select-String "references_expanded"
```

---

## 1 · Una consulta del conjunto `xref`

**🔄 New conversation**:

```
What do ECSS documents require regarding the reuse of existing or off-the-shelf software in a flight project?
```

**Esperado:**
- Entre las citas aparece **ECSS-Q-ST-80C 6.2.7.x** (p. ej. 6.2.7.3, «The existing software shall be
  assessed with regards to the applicable functional, performance and quality requirements»). Antes
  era `not_found` («binding requirement wording on reuse»).
- En la evidencia, esas cláusulas salen marcadas con **➕**, como lo añadido por expansión.
- En el log, `references_expanded` con `ECSS-Q-ST-80C` `6.2.7` entre lo expandido.

**Anota:** si el texto de la respuesta usa la cláusula traída (con cita literal o parafraseada), y si
la etiqueta «➕ added to complete the clause» te parece adecuada para algo traído **por
referencia**. Es la misma marca que `orphans`/`bounded`, por X6, pero el texto dice «complete the
clause».

**Resultado:** OK
- Se incluyen varias citas de **ECSS-Q-ST-80C 6.2.7.x**.
- Salen marcadas con **➕**
- detected= 41. `ECSS-Q-ST-80C` `6.2.7` está entre lo expandido

## 2 · Una consulta sin referencias

**🔄 New conversation**:

```
What does Q-ST-80 require for handling software problem reports and nonconformances?
```

**Esperado:** en el log, `references_expanded` con `expanded=[]` (en la medición, 2 referencias
detectadas y ninguna expandida). La respuesta, las citas y la latencia son del mismo orden que con la
expansión apagada (mediana de referencia ~13 s). Si quieres compararlo en directo, repite la pregunta
con `RETRIEVAL_XREF=off` y otro `--force-recreate`.

**Resultado:** OK, 8 citas en total, ninguna expandida. En retrieved but not cited, hay una clausula con expansión. Latencia OK.

## 3 · Tu caso del SDP, con `bounded` (indicativo)

Pon `RETRIEVAL_XREF=off` y `RETRIEVAL_EXPANSION=bounded` en `.env`, y `docker compose up -d
--force-recreate api`. **🔄 New conversation**:

```
What is the Software Development Plan in ECSS-E-ST-40C?
```

**Anota** qué cambia frente a lo que viste en la UAT de pre-S12 (con `orphans`): si la respuesta
desarrolla el contenido del SDP (E-ST-40C O.2.1 tiene 41 requisitos de contenido) o sigue limitándose
a O.1.1, y si las viñetas «I could not find…» siguen siendo de cosas no preguntadas. No está en el
golden set: es una señal, no una medición.

**Resultado:** OK a medias. Ahora la respuesta está más desarrollada, pero no cita 41 requisitos de contenido como mencionas sino 11. Hay dos clausulas "retrieved", una de ellas con 22 expansiones. Ya no hay sección "I could not find...".

**Deshaz los cambios:** `RETRIEVAL_EXPANSION=orphans` y `RETRIEVAL_XREF=off` en `.env`, y
`docker compose up -d --force-recreate api`.

**Resultado:** KO. Con orphans, la respuesta es mucho más escueta y cita solo 4 requisitos de las clausulas O.1.1, O.2.1 y O.1.2.

---

**Si algo falla:** copia la pregunta, la respuesta y la línea de `references_expanded`.
