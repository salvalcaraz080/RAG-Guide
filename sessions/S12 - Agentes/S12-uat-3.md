# S12 · UAT — Brief 3: el flujo de verificación 1.1

> **Por qué es tuyo:** caso 1 de la frontera. El flujo lo recorre una sesión de Claude Code que
> abres tú; CC no puede abrirla. CC ha comprobado el contrato (1.403 tests) y, en PowerShell, que
> `reveal` se niega sin fase 1 y que `check` rechaza V1 (1.0) sin tocar su informe.
>
> **Coste:** el de una sesión de Claude Code sobre el workspace; ninguna llamada del producto.

Los casos V2–V7 ya están creados en 1.1 (solo `question.md`). Usa **V2** para esta prueba; los
demás quedan para la verificación de los aspectos «sin puntero».

## 1 · El caso empieza solo con la pregunta

```powershell
Get-ChildItem C:\Users\salca\proyectos\ecss-verification\cases\V2
```

**Esperado:** solo `question.md`.

## 2 · La sesión escribe la fase 1 y para

En un PowerShell normal (no el workspace de VS Code, que incluye el repo):

```powershell
cd C:\Users\salca\proyectos\ecss-verification
claude
```

Primer mensaje: `Haz el caso V2`.

**Esperado:** escribe `cases\V2\answer.json` con `case`, `protocol_version: "1.1"` y `phase1`, y
**se para** diciendo que espera a que reveles el contexto. No intenta abrir `context.md`, que no
existe.

## 3 · Revelar y completar

En **otro** PowerShell, en la raíz del repo:

```powershell
uv run python -m corpus.groundtruth.verification.cases reveal V2 --workspace C:\Users\salca\proyectos\ecss-verification
```

**Esperado:** `case V2: context revealed; phase 2 can start`, y aparece `cases\V2\context.md`. En la
sesión: `Contexto revelado`. Completa la fase 2 sin tocar `phase1`, y cada puntero lleva
`points_to_doc` y `points_to_section`.

## 4 · Check, y la trampa

```powershell
uv run python -m corpus.groundtruth.verification.cases check V2 --workspace C:\Users\salca\proyectos\ecss-verification
```

**Esperado:** `VALID`, la lectura mecánica con su tipo de B1 si lo hay (`B1-clause` o
`B1-standard`), lo que dijo el diagnóstico y si coinciden. V2 es q12-a3: `check` compara con el
artefacto del diagnóstico, que es inmutable y dice **B1**, pero la UAT del brief 1 lo corrigió a
**C**. Si el verificador ve que M-ST-80C no fija criterios (no encontrado, o C), la discrepancia con
B1 es la esperada y confirma la corrección.

**La trampa:** `answer.json` tiene dos clases de `why`: la de cada ítem de `phase1` y la de
`gap_asked_for`, que no es de la fase 1. Cambiar la segunda debe dar `VALID`. Para tocar
exactamente la primera, sin guardar el fichero con BOM, y con copia:

```powershell
$f = "C:\Users\salca\proyectos\ecss-verification\cases\V2\answer.json"
Copy-Item $f "$f.bak"
$j = Get-Content $f -Raw -Encoding UTF8 | ConvertFrom-Json
$j.phase1[0].why = $j.phase1[0].why + " (editado)"
[IO.File]::WriteAllText($f, ($j | ConvertTo-Json -Depth 20))
uv run python -m corpus.groundtruth.verification.cases check V2 --workspace C:\Users\salca\proyectos\ecss-verification
```

**Esperado:** `INVALID` con `phase 1 changed after the context was revealed`. Restaura y repite:

```powershell
Move-Item "$f.bak" $f -Force
uv run python -m corpus.groundtruth.verification.cases check V2 --workspace C:\Users\salca\proyectos\ecss-verification
```

**Esperado:** `VALID` otra vez.

---

**A partir de aquí** (UAT del brief 2 y siguientes): corre V3–V7 igual. Revisa a mano solo los casos
en que la lectura y el diagnóstico **no coinciden**, más 2–3 al azar.

---

## Resultados

### V2
1 a 3 --> OK
4 (la trampa) --> KO. Al cambiar una "why" del answer.json sigue devolviendo un VALID al ejecutar el comando CHECK

**Revisión de CC (2026-10-20):** el `check` sí detecta un cambio en `phase1`. Reproducido sobre una
copia del caso: cambiar la `why` de `phase1[0]` da `INVALID`. Causa más probable: la instrucción era
ambigua, porque hay otra `why` en `gap_asked_for` que no pertenece a la fase 1, y cambiarla da
`VALID` con razón. El paso 4 ahora da el comando exacto. De paso apareció un fallo real: un
`answer.json` guardado con BOM hacía caer `check` con un traceback. Corregido y con test. **Repite
el paso 4 con los comandos de arriba.**

**Lectura de V2:** `B1-standard` («diagnosis said: B1 · same category: yes»). Ojo: el artefacto del
diagnóstico dice B1, pero la UAT del brief 1 lo corrigió a **C**. El verificador encuentra lo mismo
que CC (los criterios son del proyecto; M-ST-80C A.2.1 obliga a la política de riesgos a
definirlos, y el puntero inyectado de Q-ST-80C 5.3.1 lleva a M-ST-80C), pero lo clasifica como
B1-standard. **Es una discrepancia con la clasificación corregida, y te toca decidirla:** C («ECSS
no fija criterios») o B1-standard («el corpus dice dónde se definen y el contexto apunta allí»).

### V3 a V7
Todas VALID.