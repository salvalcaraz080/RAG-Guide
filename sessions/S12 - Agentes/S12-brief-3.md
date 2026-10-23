# S12 — Brief 3 (mini): verificación independiente, protocolo 1.1

> Mini-brief. Lee `.claude/CLAUDE.md` y `ARCHITECTURE.md`. Rama `s12-agentes`. Se ejecuta **después**
> del brief 2 y **antes** de usar la verificación en su UAT. Los commits los hace Salva.
>
> **Regla del brief:** cerrar dos huecos de la herramienta, no ampliarla. Ningún tipo de verificación
> nuevo (p. ej. juicios de relevancia del golden set): cada tipo tendrá su propio protocolo cuando
> toque.

---

## 0. Contexto

`corpus/groundtruth/verification/` (commit `9711d43`) se construyó fuera de un brief. El diseño es
bueno: el verificador no ve la taxonomía, devuelve hechos y la traducción a categorías la hace el
código; las citas se comprueban por código; la huella del corpus va en el workspace y en cada caso.
La revisión del chat encontró dos huecos que afectan a lo que significa una verificación.

---

## 1. Decisiones

| # | Decisión | Porqué |
|---|---|---|
| V1 | `Pointer` gana `points_to_doc` (obligatorio) y `points_to_section` (nulo si el puntero nombra un estándar entero o un título sin número). `points_to` libre se mantiene como explicación | Hoy `points_to` es texto libre y nadie comprueba que el destino sea la respuesta |
| V2 | `mechanical_reading` da **B1 solo si un puntero de `from_context` (verificado como inyectado) tiene como destino una cláusula de `in_corpus.answers`**, y distingue `B1-clause` (coincide `points_to_section`, o es antecesora o descendiente de la sección respuesta) de `B1-standard` (`points_to_section` nulo y coincide `points_to_doc`). Un puntero cuyo destino no coincide no cuenta | Es la definición de B1 del brief 1 ("seguir el puntero lleva a la respuesta"). La distinción cláusula/estándar es la que fijó el alcance del brief 2 |
| V3 | **Orden de fases garantizado por el flujo:** `new` crea solo `question.md`; un comando `reveal <id>` escribe `context.md` y **se niega** si `answer.json` no existe o no tiene `phase1` válido. Al revelar se guarda el hash de `phase1`, y `check` marca como problema que `phase1` haya cambiado después | La regla 5 del protocolo pasa de ser una petición a ser una garantía |
| V4 | `PROTOCOL_VERSION = "1.1"`. `VERIFY.md`/`protocol.md` actualizados (campos nuevos de puntero, el paso de revelar). `check` **rechaza** respuestas de otra versión de protocolo | Cambia lo que significa una verificación |
| V5 | El caso V1 (protocolo 1.0) **no se reescribe**: queda como histórico con su versión. Si se quiere su lectura con 1.1, se rehace como caso nuevo | Mismo principio de inmutabilidad que los artefactos del arnés |

---

## 2. Trabajo

1. Modelos y `mechanical_reading` (V1, V2). La comparación con la categoría de CC del diagnóstico
   trata `B1-clause` y `B1-standard` como B1 para la coincidencia de categoría, y reporta el tipo
   aparte.
2. Comandos `new` / `reveal` / `check` (V3). `reveal` es la única forma de que exista `context.md`.
   `workspace.md` (el `CLAUDE.md` del workspace) indica que, tras escribir la fase 1, el verificador
   para y espera a que el usuario revele el contexto.
3. Protocolo 1.1 (V4), re-inicialización del workspace si hace falta (la huella de corpus no cambia).
4. Docs: `ARCHITECTURE.md` / `.claude/CLAUDE.md` describen el flujo `new → fase 1 → reveal → fase 2
   → check`. `CHANGELOG.md` derivado.
5. **Registro de la excepción:** en `S12-decisions-3.md`, una sección que documente la construcción
   original de la herramienta (fuera de brief): qué hace, por qué (sustituir la parte pesada de la
   UAT de clasificación), protocolo 1.0, resultado del caso V1 y la lección de método que destapó
   (buscar punteros solo hacia la respuesta sospechada sesga hacia B2/B4).

---

## 3. Tests (sin red)

- `uv run pytest` en verde; recuento frente al de cierre del brief 2.
- `mechanical_reading`:
  - puntero inyectado con destino = cláusula respuesta → `B1-clause`;
  - destino = antecesora de la cláusula respuesta (p. ej. `7.2` frente a `7.2.1.2`) → `B1-clause`;
  - destino = estándar de la respuesta sin sección → `B1-standard`;
  - puntero inyectado con **destino distinto** de toda respuesta → **no** B1 (cae a `B2?` o
    `B3/B4?`);
  - puntero citado como contexto pero no inyectado → problema, no B1 (ya existía; se mantiene).
- `reveal`: se niega sin `answer.json`, se niega sin `phase1` válido, funciona con él; `check`
  detecta `phase1` modificado tras revelar.
- `check` rechaza `protocol_version` distinto de 1.1.

---

## 4. Llamadas reales

Ninguna. La herramienta no llama a modelos; los casos los corre una sesión aparte que abre Salva.

---

## 5. UAT (usuario)

1. `new` de un caso → la carpeta solo tiene `question.md`.
2. Abrir una sesión en el workspace, pedir el caso: escribe la fase 1 y para.
3. `reveal` → aparece `context.md`; la sesión completa la fase 2.
4. `check` → lectura con tipo de B1. Probar a editar `phase1` a mano tras revelar: `check` lo marca.

**Uso en la UAT del brief 2 (a partir de aquí):** la verificación sustituye la comprobación pesada.
Salva revisa solo **los casos en que verificador y diagnóstico discrepan, más 2–3 al azar**. La
independencia es de contexto, no de modelo: esa muestra es el ancla humana.

---

## 6. Cierre

`S12-decisions-3.md` (incluida la sección de la excepción, §2.5) y `S12-uat-3.md`.
