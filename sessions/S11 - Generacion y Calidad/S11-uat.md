# S11 · UAT — Brief 1

> **La comprobación del instrumento la hizo CC** (no es caso de UAT: es mecánica y ejecutable).
> Recuento independiente —sin importar nada del arnés— de las **42 ejecuciones** de
> `generation_20261018-164057`: los marcadores se leen del texto de cada respuesta, la relevancia sale
> de `golden_set.json` (no de la columna `relevant` del artefacto), y recall, precisión, inyectadas
> relevantes y citadas relevantes se comparan con la tabla **Per run** del informe.
>
> - **42/42 coinciden, 0 discrepancias**, 0 cláusulas inyectadas sin juicio. Ejemplos: q01 ej. 3
>   (`names_standard`) recall 0,67 · precisión 0,67 · 3/5 · 2/3; q09 ej. 3 (`cross_standard`) recall
>   0,67 · precisión 1,00 · 6/10 · 4/4.
> - Los `[n]` del texto coinciden con `citations` en las 42, y cada `standard` citado nombra el
>   documento de su grupo.
> - **Los 2 `misattributed` (q13 ej. 3) se confirman sobre el texto**: el marcador que sigue a la cita
>   en la respuesta es `[1]` (`ECSS-E-HB-40A §5.10.2.2.1`), la cita no está en ningún chunk de `[1]` y
>   sí en `[5]` (`§5.10.2.2.2`).

---

## Lo único que queda para ti (caso 2: requiere tu juicio)

**La pregunta:** cuando el modelo pone una cita literal exacta detrás del marcador de una cláusula
que no la contiene, pero la frase está en la cláusula **hermana, del mismo handbook y del mismo
tema**, ¿es un error que el producto debe tratar como tal (avisar, reintentar, suprimir), o es
tolerable?

No bloquea el cierre de este brief: es el insumo para la política de `S11-brief-2`.

### El caso

**Pregunta (q13, paráfrasis):** *«Once the satellite is in orbit, what do we need to have in place
to fix bugs in the onboard code and send up a corrected version without breaking anything?»*

**Lo que dice la respuesta** (ejecución 3, apartado *Validation before flight*):

> "The patch mechanism design and development is part of the initial software development process
> and can be validated as such during the qualification process" **[1]**, and importantly, "It is
> highly recommended during validation of the modification to exercise the patch procedure itself
> to verify its robustness and efficiency" **[1]**.

El resto de la respuesta cita `[1]` cinco veces más, con citas que **sí** están en `[1]`
(definiciones de *software reload* y *patch*, criterios de disponibilidad y ancho de banda, gestión
de dos binarios). `[5]` no se cita en ningún momento.

**`[1]` = `ECSS-E-HB-40A §5.10.2.2.1`** — informativa (handbook), juzgada **relevante**. Es la
sección general de **modificación en vuelo**: define *in-flight modification*, *software reload*,
*patch*, *patch cluster*, carga de OBCP; da los criterios de diseño (disponibilidad, ancho de banda
TM/TC, memoria) y recomienda dos binarios a bordo en sistemas de alta disponibilidad. **No dice nada
sobre cómo validar el mecanismo de parcheo.**

**`[5]` = `ECSS-E-HB-40A §5.10.2.2.2`** — informativa (handbook), juzgada **relevante**. Es la
subsección específica de **parches**. Texto íntegro de la parte relevante:

> It is highly recommended during validation of the modification to exercise the patch procedure
> itself to verify its robustness and efficiency.
> The patch mechanism design and development is part of the initial software development process
> and can be validated as such during the qualification process.
> The problematic part in the patch itself is to ensure the correct mapping in the patch application
> i.e. not patched zone needs to have an ISO-mapping demonstrated. The patch needs to be correctly
> linked in the hosting application.

Es decir: **las palabras son exactas y vienen del corpus; lo único falso es la dirección.** Un lector
que pinche en `[1]` para comprobar la recomendación de validar el procedimiento de parcheo no la
encontrará en esa cláusula.

### Qué cambia según lo que decidas

| Si lo consideras… | Consecuencia para el brief 2 |
|---|---|
| **Error que el usuario debe ver** | `misattributed` entra en la política como fallo de cita (al nivel de `not_found` o por encima: la cita *parece* verificable y no lo es). Opciones: reintento, aviso en la cita, o reasignar el marcador al grupo donde está (`found_in` lo da gratis). |
| **Tolerable dentro del mismo tema** | Haría falta definir «mismo tema» (¿mismo documento y cláusula hermana? ¿mismo padre `5.10.2.2`?) para que la tolerancia sea una regla y no un caso. Riesgo: el mismo patrón entre cláusulas no hermanas es la atribución falsa de D-1. |

Frecuencia medida, para dimensionarlo: **2 de 280 pares** en la referencia (9 en las 6 ejecuciones
de las dos pasadas, todas este mismo caso). Con la misma entrada, el modelo usa esas frases en las
6 ejecuciones de q13: en **4** las pone detrás de `[1]` y en **2** las atribuye bien, a `[5]`. No es
un error sistemático del contexto, sino una elección que el modelo hace distinto entre ejecuciones
idénticas.

### Respuesta del usuario

**Reasignar.** Las dos cláusulas son necesarias para la pregunta (§5.10.2.2.1 enmarca la
modificación de código en vuelo; §5.10.2.2.2 da lo específico de parches), y el usuario debe ver la
cláusula correcta de donde salió cada cita. Registrado en `S11-decisions.md` §8 como insumo para
`S11-brief-2`, con los casos que quedan por concretar.

### Si quieres leerlo tú mismo (opcional)

```powershell
$a = Get-Content -Raw -Encoding UTF8 data\groundtruth\generation\runs\generation_20261018-164057.json | ConvertFrom-Json
$m = ($a.queries | Where-Object query_id -eq 'q13').runs | Where-Object run -eq 3
$m.answer
($m.injected | Where-Object ref -eq 1).chunks | ForEach-Object { "---"; $_.cite_text }
($m.injected | Where-Object ref -eq 5).chunks | ForEach-Object { "---"; $_.cite_text }
```
