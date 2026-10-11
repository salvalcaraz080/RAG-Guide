# S09 — Guion de UAT (Streamlit) — ejecutada

> **Ejecutada por el usuario.** Resultados en cada caso; lectura conjunta y revisión de soporte al
> final. Conclusiones trasladadas a `S09-decisions.md` §5.
>
> Para el usuario. Lo que CC podía ejecutar ya está ejecutado (`S09-decisions.md` §1–§2: suite,
> diagnóstico §3.8, smoke de cuatro preguntas, fail-closed con la base parada). Esta UAT es la
> que no tiene oráculo: **un ingeniero leyendo si la respuesta y sus citas se sostienen.** Gasta
> llamadas reales (Haiku, del orden de céntimos en total).

Ejecutar desde `C:\Users\salca\proyectos\RAG-ECSS`, en PowerShell.

---

## 0. Arranque

```powershell
docker compose up -d
docker compose ps --format "{{.Name}} {{.Status}}"
```

Esperado: `rag-ecss-db` y `rag-ecss-api` `healthy`, `rag-ecss-frontend` `Up`. Abrir
<http://localhost:8501>. El sidebar de diagnóstico sale porque `FRONTEND_ENV` es `development`.

Para ver los dos eventos de log tras cada pregunta (en otra consola):

```powershell
docker logs rag-ecss-api --since 15m | Select-String "retrieval_completed|citations_validated"
```

## Qué mirar en CADA caso

1. **Marcadores ↔ lista de citas.** Cada `[n]` del texto tiene su entrada `[n]` en «Sources», y
   no hay en «Sources» ninguna cláusula que el texto no marque.
2. **Citas literales visibles.** Al enunciar un requisito aparece `"…" [n]`. Abrir la fuente `[n]` y
   comprobar que esas palabras están en el verbatim.
3. **Autoridad en la traza.** Cada átomo lleva `⚖️ normative` o `ℹ️ informative`. Una NOTE no se
   presenta en el texto como requisito.
4. **Citado vs recuperado.** El desplegable «Retrieved but not cited» contiene el resto de lo que
   el modelo recibió.
5. **Los dos eventos de log**, con el mismo `request_id`: `retrieval_completed` (ids, rangos,
   distancias, cláusulas, `context_tokens`, `embedding_ms`, `db_ms`) y `citations_validated`
   (`verified`/`not_found`/`invalid_ref`, `markers_without_quote`). Ninguno lleva el texto de la
   pregunta ni de la respuesta.

---

## 1. Identificador explícito

> What does ECSS-E-ST-40C require for dynamic memory management in flight software?

**Ya visto por CC:** el retrieval vectorial **no trae ninguna cláusula de ECSS-E-ST-40C**, y el
modelo rechaza con `Not covered by the provided material:` diciendo de dónde son las fuentes.
Mirar que el rechazo hable de **las fuentes recibidas** y no afirme que ECSS no lo regula. Sin
traza de usuario (no hay citas); en el sidebar, la evidencia cruda sí está.

**Resultado:**
> Not covered by the provided material:
>
> The reference material provided does not contain substantive requirements from ECSS-E-ST-40C on dynamic memory management in flight software. The only ECSS-E-ST-40 source included is an informative handbook section (ECSS-E-HB-40A) that discusses memory budget verification approaches, not binding requirements.
> 
> The normative requirements in the material address memory management through ECSS-E-ST-70-01C and ECSS-E-ST-70-41C, but they do not specifically prescribe whether or how dynamic memory allocation may or must be used in flight software. ECSS-E-ST-70-01C only requires that "Dynamic and static memory requirements shall be predictable at procedure preparation time" [3], and the handbook notes that "for flight software there is no dynamic memory mechanism implementation (state of the art)" [4], but this is informative rather than a normative prohibition or requirement.
> 
> To answer your question properly, I would need access to the actual normative clauses of ECSS-E-ST-40C or ECSS-E-ST-70-01C that directly address dynamic memory management policy.

**Intepretación**: esta respuesta se ve extraña ya que pide al usuario acceso a las clausulas de ECSS-E-ST-40C o ECSS-E-ST-70-01C, algo que es trabajo del mecanismo de retrieval, no del usuario que pregunta. El sidebar muestra atomos recuperados, pero la respuesta se construye como si no hubiera habido retrieval.
En cuanto al ranking, los chunks con menos distancia son 2 notas informativas que únicamente dicen "Refer also to clause 5.4.3.3.", lo que resulta menos relevante que otros que sí llevan requisitos definidos.

## 2. Par de criticidad B/C

> What is the difference between software criticality category B and C in ECSS-Q-ST-80C?

Mirar: que cite la tabla D-1 de `ECSS-Q-ST-80C` (o `ECSS-Q-ST-40C §6.5.6.3`); que la diferencia
B/C que cuenta sea la de la tabla (categoría de función + provisiones compensatorias); en el log,
`verified` igual a las citas literales que ves.

**Resultado:** parece una respuesta adecuada en este caso. La respuesta incluye [4], la respuesta tiene bien en cuenta el verbatim de la cida.

## 3. Pregunta enumerativa

> Which ECSS standards apply to the development of critical flight software?

Mirar: que diga que la lista **se limita a las fuentes proporcionadas** (regla 4 del prompt); que
no meta estándares que no estén en «Sources» ni en «Retrieved but not cited»; si lo que lista
viene de un handbook (`-HB-`), que no lo presente como obligación.

**Resultado:**
> Based on the provided material, the following ECSS standards apply to the development of critical flight software:
>
> ECSS-E-ST-40C is the primary standard for software development. It "covers all aspects of space software engineering including requirements definition, design, production, verification and validation, transfer, operations and maintenance" [1] and applies to "the development of any kind of software in the space domain" [3], including flight software.
> 
> ECSS-Q-ST-80 is the complementary product assurance standard. "The Space Engineering Software Standard is always complemented by the Space Product Assurance Standard (ECSS-Q-ST-80C)" [1]. For safety-critical software specifically, "A safety-critical software product shall be designed, implemented, verified and operated according to the engineering and product assurance requirements defined in ECSS-E-ST-40 and ECSS-Q-ST-80" [8].
>
> For secure software development, "ECSS-Q-ST-80 and ECSS-E-ST-40 shall be followed" [4].
>
> Critical software is defined as "software of criticality category A, B or C" [6], and the tailoring of these standards is driven by the software's criticality level [5].
>
> The provided material confirms that both standards work together as a coherent framework for critical flight software development, though it does not exhaustively list all requirements that these standards contain.

**Intepretación**: No me gusta que las respuestas siempre incluyan la muletilla del tipo "based on the provided material", da lugar a entender que no todo el material relevante puede estar incluido. Y esto es algo que nosotros como developers debemos conocer si ocurre, pero no el usuario, ya que merma su confianza en el producto.
En la respuesta, no hay remenciones ferencias a los hanbbooks, cuando sin embargo si se han recuperado chunks referentes a ellos.

Todas las [n] citas en esta respuesta parafrasean de forma muy precisa el texto original.

## 4. Seguimiento anafórico

Justo después del caso 2, **en la misma conversación**:

> And what does ECSS-Q-ST-80C say about it?

**Ya visto por CC:** falla. El retrieval recibe solo esta frase, el «it» no recupera nada útil, y
el modelo rechaza. Lo que hay que juzgar es **cuánto molesta** en uso real (es el insumo de la
decisión diferida «reescritura de seguimientos conversacionales»). Prueba también una variante
explícita, «What does ECSS-Q-ST-80C say about software criticality category B?», para comparar.

**Resultado:**
> Not covered by the provided material:
>
> The reference material provided does not contain the substantive requirements from ECSS-Q-ST-80C itself. The sources include only normative sections stating that "The following normative documents contain provisions which, through reference in this text, constitute provisions of this ECSS Standard" [1][2], along with mapping tables and cross-references, but they do not contain the actual text of ECSS-Q-ST-80C's requirements regarding critical flight software development.
>
> To answer what ECSS-Q-ST-80C specifically says about critical flight software, I would need sources that include the requirement clauses from that standard.

**Interpretación:** personalmente creo que molesta bastante que el chat no sea capaz de seguir la conversación con un pronombre que hace referencia a una respuesta anterior. Si no conseguimos que esto ocurra, entonces el producto no deberia ser un chat, si no un asistente de preguntas aisladas donde cada consulta es independiente.

Al preguntar "What does ECSS-Q-ST-80C say about software criticality category B", sí que devuelve una respuesta adecuada.

## 5. Fuera de dominio

> What's a good recipe for a chocolate cake?

Esperado: empieza por `Out of scope:`, sin «Sources». En el log, `refusal=out_of_domain` y
`retrieval_completed` igualmente (el retrieval corre antes de que el modelo decida).

**Resultado:** El esperado, un mensaje Out of scope.

## 6. Fuera de material (en dominio, no en el corpus)

> What are the ECSS requirements for the thermal vacuum testing of a satellite battery?

Esperado: `Not covered by the provided material:` o una respuesta muy acotada a lo recuperado. Si
responde, todo lo que afirme tiene que estar marcado y verificable en «Sources».

**Resultado:** El esperado, un mensaje Not covered by the provided material.

---

## Lectura de la UAT (CC, con los logs de la sesión)

Datos añadidos por CC sin coste: los `citations_validated` de la UAT y el `embed_text` de las notas
que encabezan el ranking del caso 1.

1. **El seguimiento anafórico es un fallo de producto.** Veredicto del usuario: si un pronombre no
   funciona, el producto no es un chat. La causa está aislada (el retrieval solo ve la pregunta
   actual; la variante explícita responde bien). La reescritura de seguimientos pasa de diferida a
   candidata prioritaria; la técnica la decide Claude Chat.
2. **Rechazos con contenido.** En los logs, **4 de los 6 rechazos `out_of_material` llevaban citas
   literales `verified`**. Son respuestas parciales con la frase de rechazo delante (caso 1). Como
   un rechazo vacía `citations`, el usuario ve `[3]` y `[4]` en el texto **sin fuentes debajo**.
   Pregunta para Chat: ¿un tercer estado («respuesta parcial»), o un rechazo que cita conserva sus
   citas?
3. **La voz deja ver la tubería.** La muletilla «based on the provided material» y el «I would need
   access to the clauses» del caso 1 tienen el mismo origen: el modelo habla del material que
   recibió, no del corpus. Lo empujan las reglas 4 y 5 del prompt y la propia frase marcador
   `Not covered by the provided material:`, que llega tal cual al usuario. El aviso de cobertura
   es honesto si se formula sobre el corpus indexado; que el retrieval se quedó corto es señal para
   el developer (está en los logs), no para el usuario. Cambiar la frase marcador toca
   `_refusal_kind` y su contrato.
4. **Notas huérfanas arriba del ranking: causa confirmada.** El `embed_text` de
   `ECSS-E-ST-70-41C#6.6.4.2::n0` es casi todo cabecera («…ST[06] memory management > … Memory
   accessibility») más 29 caracteres de texto. El vector representa el título, no la nota, y el
   título casa con «memory management». Afecta a todo chunk diminuto. Vías: fusionar la nota con
   su requisito en el chunking (re-chunkea y re-embebe: agrupable con el tokenizador) o que la
   nota huérfana arrastre su requisito en la expansión.
5. **Menores.**
   - Caso 3, handbooks omitidos: coherente con la regla 3 (la pregunta es qué *aplica*). Mostrarlos
     como guía sería decisión de producto.
   - `not_found=0` en toda la UAT: ningún falso negativo de la normalización, pero la muestra no da
     una tasa útil para S11.
   - Anomalía sin identificar: una respuesta no rechazada **sin ningún marcador**
     (`refusal=None`, `valid_markers=0`, 4 cláusulas inyectadas), presentada como anclada y sin
     «Sources».

### Soporte de cada `[n]`: revisado a mano en los casos 2 y 3

Hay tres niveles de comprobación sobre un `[n]`:

| Nivel | Qué comprueba | Quién | Resultado |
|---|---|---|---|
| Pertenencia | `[n]` apunta a una fuente inyectada | `resolve_markers` | 0 marcadores inválidos |
| Literalidad | Las comillas están en la fuente | `validate_quotes` | 0 `not_found` |
| Soporte | La fuente `[n]` respalda lo que afirma la frase | Usuario, leyendo | **Satisfactorio** en los casos 2 y 3 |

El usuario abrió cada fuente `[n]` de los casos 2 y 3 y comprobó que el texto que la acompaña
**reproduce o parafrasea con precisión** el verbatim original. No encontró ningún `[n]` que
apuntara a una cláusula que no dijera lo que el texto le atribuye.

Alcance, para no sobreleerlo:

- **Es un veredicto cualitativo**, no una clasificación frase a frase *respalda / en parte / no
  respalda*. No da todavía una tasa de soporte numérica.
- **Muestra pequeña:** dos respuestas. La variante explícita del caso 4 no se revisó.
- **La síntesis final** de una respuesta (p. ej. «both standards work together as a coherent
  framework…» en el caso 3) es una conclusión del modelo sin marcador: este método no la cubre.

---

## Qué devolver a Claude Chat — respondido

- **¿Las citas se sostienen leyendo la fuente?** Sí, en los casos 2 y 3 (ver arriba).
- **¿Algún `[n]` apunta a una cláusula que no dice lo que el texto le atribuye?** No se encontró
  ninguno en la muestra revisada.
- **¿Algún `not_found` con una cita que, leída, sí está en la fuente?** No hubo ningún `not_found`
  en toda la UAT, así que no hay falsos negativos de la normalización que revisar.
