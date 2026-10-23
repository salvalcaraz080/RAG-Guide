# S12 · UAT — Brief 4: `bounded` por defecto y verificación 1.2

> **Por qué es tuyo:** el caso 1 tiene coste real (una pregunta al modelo), y el caso 2 requiere las
> sesiones de verificación, que abres tú. CC ha comprobado el contrato (1.417 tests), que el default
> es `bounded` y que `check` rechaza 1.1 sin tocar sus informes.
>
> **Coste:** una pregunta a Haiku 5.5 (< 0,005 USD) y seis sesiones de verificación.

## 1 · `bounded` por defecto

`.env` ya dice `RETRIEVAL_EXPANSION=bounded` (y `.env.example`, igual). El contenedor lee `.env` al
crearse, así que hay que recrearlo:

```powershell
docker compose up -d --force-recreate api
```

**🔄 New conversation**:

```
What is the Software Development Plan in ECSS-E-ST-40C?
```

**Esperado:** lo mismo que en la UAT del brief 2 con `bounded`: respuesta desarrollada, unos 11
requisitos del DRD de O.2.1 citados y sin bloque «I could not find…». En la evidencia, los chunks
añadidos con ➕.

**Resultado:** Esperado, citados 9 requisitos. Clausula O.2.1 expandida con 22 clausulas. Aquí tengo una pregunta, si se ha expandido 22 veces, y todas esas clausulas contienen requisitos, por qué no se citan todos?

## 2 · Verificación 1.2: V8–V13

V8–V13 ya están creados (solo `question.md`). Para cada uno, como en la UAT del brief 3:

1. Sesión en `C:\Users\salca\proyectos\ecss-verification` (`claude`): «Haz el caso V8». Escribe la
   fase 1 y para.
2. En otro PowerShell, en la raíz del repo:
   ```powershell
   uv run python -m corpus.groundtruth.verification.cases reveal V8 --workspace C:\Users\salca\proyectos\ecss-verification
   ```
3. En la sesión: «Contexto revelado». Completa la fase 2.
4. ```powershell
   uv run python -m corpus.groundtruth.verification.cases check V8 --workspace C:\Users\salca\proyectos\ecss-verification
   ```

Una sesión nueva por caso. En la salida del `check`, mira la línea `mechanical reading` y, si
salen, las de punteros `inferred` y las de punteros literales que no llevan a una respuesta
principal.

**Tabla final** (la misma de `S12-decisions-4.md` §4):

| Caso | Aspecto | Diagnóstico (corregido) | Lectura 1.2 | ¿Coincide? |
|---|---|---|---|---|
| V8 | q12-a3 | B1-standard | B1 | yes |
| V9 | q05-a1 | B4 | B2 | no |
| V10 | q09-a2 | C | C | yes |
| V11 | q13-a2 | B4 | VALIDATION ERROR | |
| V12 | q04-a3 | B4 | B2 | no |
| V13 | q05-a2 | B4 | B2 | no |

**Cómo leer «¿Coincide?»:** `check` compara con el artefacto del diagnóstico (inmutable), así que
su «same category» usa la categoría **original**. Usa la columna de la tabla, que es la corregida.
`B2?` y `B3/B4?` frente a un B4 son «sin decidir»: no llegan a B1, y entre B2 y B4 decide un rango
que mide CC.

**Esperado:** las discrepancias de V3, V4, V6 y V7 desaparecen o quedan explicadas (puntero
`inferred`, respuesta solo relacionada, o `answered_as_phrased=no`), y V9 y V12 ya no salen B1 salvo
con puntero literal. **Revisa a mano solo las discrepancias que queden, más 2–3 al azar.**
