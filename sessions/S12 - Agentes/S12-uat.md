# S12 · UAT — Brief 1: verificar la clasificación de los `not_found`

> **Por qué es tuyo** (caso 1 de la frontera): la clasificación es un juicio sobre qué dice un
> estándar, y no hay oráculo. CC ha comprobado mecánicamente lo comprobable: que cada cita es literal
> en su chunk, que lo marcado G estaba inyectado y lo marcado B no, que cada referencia B1 está en un
> chunk inyectado y que seguirla lleva a la respuesta, que cada rango de B2/B3 sale igual al
> re-ejecutar la variante, y que las 65 líneas originales están asignadas una sola vez. Lo que no
> puede comprobar es si la cláusula **responde de verdad** al aspecto. Eso es lo que te toca.
>
> **Coste:** ninguno. Sin llamadas de generación; las variantes de retrieval leen sus embeddings de la
> caché, así que tampoco hay llamadas de embeddings.

## 0 · Preparación

Abre el informe legible, que tiene la evidencia completa (citas, puntero, variante, líneas
originales) de cada aspecto:

```
data\groundtruth\generation\diagnostics\notfound_20261019-141324.md
```

Para ver una cláusula entera del corpus (en la raíz del repo, PowerShell):

```powershell
uv run python -m scripts.notfound_diagnosis clause ECSS-M-ST-80C 7.2
```

Para repetir una variante de retrieval (necesita la base en marcha: `docker compose ps` → `db`):

```powershell
uv run python -m scripts.notfound_diagnosis rank "applicability of ISVV requirements per software criticality category" --doc ECSS-Q-ST-80C --k 10
uv run python -m scripts.notfound_diagnosis original q11 --k 20
```

`original` muestra la consulta de producción con su filtro; los chunks marcados con `*` son los
que se inyectaron.

## 1 · Qué mirar en cada tipo

- **B1** (puntero desde lo inyectado): (a) que la cita *reference* esté en un chunk que se inyectó
  (lo comprobó el script) y que **de verdad remita** a donde está la respuesta; (b) que la cláusula
  *answer* responda al aspecto. Mira la columna «Ruta»: un puntero a un **estándar entero** es más
  débil que uno a una cláusula, y el rango dice si seguirlo con la consulta original lo trae al top 10.
- **B2** (variante): que la variante sea algo que un plan razonable habría pedido **leyendo solo la
  pregunta** (no algo que ya conoce la respuesta), y que la cláusula responda.
- **A / C / G** (muestra): A, que la respuesta esté de verdad fuera del corpus; C, que las cláusulas
  citadas cubran el tema y no lo especifiquen; G, que lo inyectado respondiera ya el aspecto.
- **Espurio** (⚑): si el aspecto excede lo que pedía la pregunta. Si discrepas, anótalo.

## 2 · La tabla

**Obligatorias:** todas las B1 y B2, y las de confianza `baja`. **Muestra:** A, C y G (son 5;
están todas). **Opcional:** B3 y B4. En la última columna: `ok`, o la categoría que pondrías y por
qué en pocas palabras. El chat aplica la regla de §4 del brief con tu clasificación corregida.

| # | Aspecto | Cat. | Conf. | Revisión | Ruta | Ver la respuesta | ¿De acuerdo? / categoría corregida |
|---|---|---|---|---|---|---|---|
| q01-a1 | la lista completa de revisiones de proyecto de software | B1 | media | **obligatoria** | puntero (clause) a `ECSS-E-ST-40C` §5.3.3.2 | `clause ECSS-E-ST-40C 5.3.3.2` | CC: ok con reserva: el puntero es una fila de matriz de trazabilidad, no un «see»; aceptable como B3 (rango 18). No cambia el recuento |
| q01-a2 | la documentación que se entrega en cada revisión | B1 | alta | **obligatoria** | puntero (same_clause) a `ECSS-E-ST-40C` §A | `clause ECSS-E-ST-40C A` | CC: ok |
| q03-a1 | la asignación de categoría para funciones de criticidad III y IV | B1 | alta | **obligatoria** | puntero (same_clause) a `ECSS-Q-ST-80C` §D.1 | `clause ECSS-Q-ST-80C D.1` | CC: ok |
| q04-a1 | los criterios que asignan cada elemento a un nivel de ISVV · parcial | A | alta | muestra | fuera: ESA ISVV Guide | `clause ECSS-E-HB-40A 5.6.2.2.2` | CC: ok |
| q04-a2 | las actividades de ISVV de cada nivel | A | alta | muestra | fuera: ESA ISVV Guide | `clause ECSS-E-HB-40A 5.6.2.2.2` | CC: ok |
| q04-a3 | cuándo es obligatoria la ISVV en un proyecto · parcial | B2 | baja | **obligatoria** | variante filtro ['ECSS-Q-ST-80C'] → rango 8 | `clause ECSS-Q-ST-80C D.2` | CC: **→ B4**. La variante que llega usa «applicability», vocabulario de la respuesta; la faceta natural, con la criticidad ya metida, queda en el 27 |
| q05-a1 | el registro de la versión de cada fichero y documento en cada entrega · parcial | B4 | media | opcional | — | `clause ECSS-M-ST-40C E.2.1` | CC: ok |
| q05-a2 | quién puede cambiar los elementos tras la entrega · parcial | B4 | media | opcional | — | `clause ECSS-M-ST-40C 5.3.2.1` | CC: ok |
| q06-a1 | la independencia exigida a la categoría C | B1 | alta | **obligatoria** | puntero (clause) a `ECSS-Q-ST-80C` §D.2 | `clause ECSS-Q-ST-80C D.2` | CC: ok |
| q06-a2 | los requisitos de verificación y pruebas que distinguen B de C, además de la cobertura | B1 | alta | **obligatoria** | puntero (clause) a `ECSS-E-ST-40C` §R.2 | `clause ECSS-E-ST-40C R.2` | CC: ok |
| q07-a1 | cómo se compara una sprint review con el alcance de una PDR o CDR | G | media | muestra | — | `clause ECSS-E-HB-40-01A 6.6.2.1` | CC: ok |
| q08-a1 | la relación detallada entre las cláusulas de Q-ST-80C y el procedimiento de SFMEA | G | media | muestra | — | `clause ECSS-Q-ST-80C 6.2.2.3` | CC: ok |
| q08-a2 | los pasos del procedimiento de SFMEA | B1 | media | **obligatoria** | puntero a estándar `ECSS-Q-ST-30-02C` → rango 15 | `clause ECSS-Q-ST-30-02C 4.1` | CC: ok (la ruta más fuerte es «The following sections», que lleva a las cláusulas hermanas 6.2.2.3–7) |
| q08-a3 | los requisitos de Q-ST-30-02C aplicables a la SFMEA · parcial | B1 | media | **obligatoria** | puntero (clause) a `ECSS-Q-ST-30-02C` §4.6 | `clause ECSS-Q-ST-30-02C 4.6` | CC: **→ G**. El handbook inyectado ya dice lo que exige 4.6.f, y Q-30-02 no define la SFMEA |
| q09-a1 | la redacción vinculante de los requisitos de reutilización | B1 | alta | **obligatoria** | puntero (clause) a `ECSS-Q-ST-80C` §6.2.7 | `clause ECSS-Q-ST-80C 6.2.7.3` | CC: ok |
| q09-a2 | requisitos específicos de proyectos de vuelo | C | media | muestra | — | `clause ECSS-Q-ST-80C 6.2.7.4` | CC: ok |
| q09-a3 | los requisitos de cualificación del software reutilizado | B1 | alta | **obligatoria** | puntero (clause) a `ECSS-Q-ST-80C` §6.2.7 | `clause ECSS-Q-ST-80C 6.2.7.8` | CC: ok |
| q10-a1 | los valores objetivo numéricos de cada métrica ⚑ espurio · parcial | B1 | media | **obligatoria** | puntero a estándar `ECSS-Q-HB-80-04A` → rango 15 | `clause ECSS-Q-HB-80-04A 3.8` | CC: ok como B1, pero espurio: si el chat no cuenta lo espurio, q10 sale de B1 |
| q11-a1 | las reglas de redacción de cada requisito individual · parcial | B2 | alta | **obligatoria** | variante filtro ['ECSS-E-ST-10-06C'] → rango 1 | `clause ECSS-E-ST-10-06C 7.2.3` | **Verificación independiente V1: → B1 media.** El contexto ya apuntaba a Q-ST-80 cl. 7.2 (5.4.2.1) y a 7.2.1.3 (B.2.1); la regla de software está en Q-ST-80C 7.2.1.2. La ruta B2 a E-ST-10-06C también es real: la fase 1 lo elige con solo leer la pregunta |
| q12-a1 | el proceso de identificación de riesgos de software | B1 | alta | **obligatoria** | puntero a estándar `ECSS-M-ST-80C` → rango 4 | `clause ECSS-M-ST-80C 5.2.2.1` | CC: ok, confianza → media (M-ST-80 es genérico, no da métodos propios del software) |
| q12-a2 | cómo se consolidan y registran los riesgos de software en el proceso del proyecto | B1 | alta | **obligatoria** | puntero a estándar `ECSS-M-ST-80C` → rango 3 | `clause ECSS-M-ST-80C 6.1` | CC: ok |
| q12-a3 | los criterios de aceptación de riesgos de software | B1 | media | **obligatoria** | puntero a estándar `ECSS-M-ST-80C` → rango >50 | `clause ECSS-M-ST-80C A.2.1` | CC: **→ C**. M-ST-80 no da criterios: delega en la política del proyecto. Mismo criterio que la parte de riesgo de q04-a3 |
| q12-a4 | el contenido de la política de riesgos del proyecto | B1 | alta | **obligatoria** | puntero a estándar `ECSS-M-ST-80C` → rango 9 | `clause ECSS-M-ST-80C A.2.1` | CC: ok |
| q13-a1 | requisitos normativos para subir en órbita una versión corregida | B3 | alta | opcional | consulta original → rango 17 | `clause ECSS-E-ST-40C 5.10.2.2` | CC: ok |
| q13-a2 | requisitos normativos de regresión: que la corrección no afecte al resto · parcial | B4 | media | opcional | — | `clause ECSS-E-ST-40C 5.10.4.3` | CC: ok |
| q14-a1 | lo que el software de vuelo debe proveer para ejecutar OBCP | B1 | alta | **obligatoria** | puntero a estándar `ECSS-E-ST-70-01C` → rango 2 | `clause ECSS-E-ST-70-01C 5.4.4.2` | CC: ok (5.4.2.3::r0, las operaciones del motor, responde mejor que 5.4.4.2) |
| q14-a2 | los requisitos de los subservicios de gestión de OBCP y del motor | B1 | baja | **obligatoria** | puntero (name) a `ECSS-E-ST-70-41C` §6.18.4 | `clause ECSS-E-ST-70-41C 6.18.4.4.5` | CC: ok como B1 baja; si no se acepta el puntero por título, B3. No cambia el recuento |

## 3 · Las cuatro que más pesan si cambian

Si vas justo de tiempo, estas son las que mueven el recuento por consultas de B1 o de B2:

1. **q01-a1** (B1, media): la «referencia» es una fila de una tabla de trazabilidad
   (`5.3.3.2b`), no un «see». Si no la aceptas como puntero, pasa a **B3** (rango 18), pero q01
   sigue en B1 por q01-a2.
2. **q04-a3** (B2, baja): la variante que llega nombra la «matriz de aplicabilidad». Si te parece
   que eso ya sabe la respuesta, es **B4** y q04 sale de B2.
3. **q11-a1** (B2, alta): ¿un plan que lee «rules … how each individual software requirement is
   written» pediría E-ST-10-06C? Si no, es **B4** y B2 se queda con q04 sola, o con ninguna.
4. **q14-a2** (B1, baja): puntero por título. Si no lo aceptas, es **B3** (rango 31); q14 sigue en B1
   por q14-a1.

## 4 · Revisión de CC (autorrevisión adversarial)

> **No es independiente.** La clasificación es de CC, y esta pasada también: relee cada evidencia
> buscando motivos para corregirla. Reduce lo que Salva tiene que mirar, pero no sustituye D1. Las
> correcciones van solo en esta tabla; el artefacto es inmutable y no se toca. El chat aplica la
> regla con la clasificación corregida.

**Correcciones (4; la de q11 sale de la verificación independiente de §5):**

| Aspecto | Antes | Después | Motivo |
|---|---|---|---|
| q04-a3 | B2 baja | **B4** media | la única variante que llega usa el vocabulario de la respuesta («applicability») |
| q08-a3 | B1 media | **G** media | el contenido de 4.6.f ya estaba en el handbook inyectado |
| q12-a3 | B1 media | **C** media | ECSS no fija criterios de aceptación: los delega en la política del proyecto |
| q11-a1 | B2 alta | **B1** media | verificación independiente V1 (§5): el contexto apuntaba a Q-ST-80 7.2 |

**Recuento por consultas tras la revisión:**

| Categoría | Antes | Después | Consultas (después) |
|---|---|---|---|
| B1 | 8 | **9** | q01, q03, q06, q08, q09, q10, q11, q12, q14 (8 si no cuenta el espurio de q10) |
| B2 | 2 | **0** | — (q04, si no se acepta su corrección a B4) |
| B3 | 1 | 1 | q13 |
| B4 | 2 | 3 | q04, q05, q13 |
| G | 2 | 2 | q07, q08 |
| A | 1 | 1 | q04 |
| C | 1 | 2 | q09, q12 |

**Lo que sigue siendo tuyo:** q01-a1, q04-a3 y q14-a2 de §3 (q11-a1 ya está resuelta por la
verificación independiente de §5), y si se cuenta el aspecto espurio de q10. Lo demás lo he leído
entero y no he encontrado motivo para cambiarlo, aunque es mi propio juicio: §5 muestra que ese
juicio tiene un sesgo en los aspectos «sin puntero».

## 5 · Verificación independiente — caso V1 (q11-a1)

Una sesión de Claude Code abierta fuera del repo (`C:/Users/salca/proyectos/ecss-verification`),
con solo el export de cláusulas, el índice y `VERIFY.md`, y sin ver la clasificación. Su respuesta
está en `answer_V1.json`. CC comprobó sus **22 citas: todas literales**, y que sus dos punteros de
contexto (E-ST-40C 5.4.2.1 y B.2.1) estaban de verdad entre lo que tuvo el modelo.

- **Fase 1, antes de ver el contexto:** consultaría E-ST-40C, **E-ST-10-06C**, Q-ST-80C y E-HB-40A, en
  ese orden. La ruta B2 a 10-06C se deduce de la pregunta.
- **Contexto:** respondía en parte (identificación única, método de validación).
- **Puntero que CC no vio:** 5.4.2.1 dice «(see ECSS-Q-ST-80 clause 7.2)» y B.2.1 «as per …
  ECSS-Q-ST-80, 7.2.1.3». En Q-ST-80C 7.2.1.2 está la regla propia del software (correct,
  unambiguous, complete, consistent, verifiable, traceable), que no se inyectó porque el filtro de
  routing dejó fuera Q-ST-80C. Por la regla de desempate, **B1 gana a B2**. Confianza media: el
  puntero cuelga de «product quality requirements», no de la redacción de requisitos.
- **Además:** D.2.1, inyectada solo en parte, trae más reglas por requisito (estructura,
  trazabilidad). Es el mismo patrón `same_clause` de q01-a2 y q03.

**Sesgo de método que esto destapa:** CC buscó punteros hacia donde ya creía que estaba la
respuesta, no leyó todos los punteros del contexto. Afecta sobre todo a los aspectos clasificados
como «sin puntero»: B4 (q05-a1, q05-a2, q13-a2), q04-a3 y las C (q09-a2, q12-a3). Pasarlos por el
mismo montaje (casos V2–V7) es lo que haría robusto el recuento antes de que el chat aplique la regla.
