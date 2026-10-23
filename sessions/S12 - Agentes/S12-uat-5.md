# S12 · UAT — Brief 5: `xref` encendido por defecto

> **Por qué es tuyo:** el caso 2 tiene coste real (una pregunta al modelo) y se juzga leyendo la
> respuesta. CC ha comprobado el contrato (1.419 tests), la configuración efectiva del contenedor
> (`bounded clause`) y `check V11`.
>
> **Coste:** una pregunta a Haiku 5.5 (< 0,005 USD), unos 17 s.

## 1 · Configuración efectiva

`.env` y `.env.example` dicen ya `RETRIEVAL_EXPANSION=bounded` y `RETRIEVAL_XREF=clause`, los mismos
valores que el default del código. Recrea `api` para que lea el `.env` actual y reinicia `frontend`
(Streamlit no recarga los módulos que importa, así que la UI nueva pide el proceso nuevo):

```powershell
docker compose up -d --force-recreate api
docker compose restart frontend
docker compose exec api python -c "from app.config import get_settings; s = get_settings(); print(s.RETRIEVAL_EXPANSION, s.RETRIEVAL_XREF)"
```

**Esperado:** `bounded clause`.

## 2 · q09

**🔄 New conversation**:

```
What do ECSS documents require regarding the reuse of existing or off-the-shelf software in a flight project?
```

**Esperado:** la respuesta cita normativa de **ECSS-Q-ST-80C 6.2.7.x** (p. ej. 6.2.7.3, «The existing
software shall be assessed…») además de los handbooks. En la evidencia, esas cláusulas salen con
**«➕ by cross-reference»** (antes decían «added to complete the clause»). Las que completa `bounded`
siguen con «➕ N added».

**Resultado:** OK

## 3 · `check V11`

```powershell
uv run python -m corpus.groundtruth.verification.cases check V11 --workspace C:\Users\salca\proyectos\ecss-verification
```

**Esperado:** `VALID`, con la lectura `B2?` y el puntero SAVOIR como `untitled (…)` entre los
inferidos. Frente al B4 del diagnóstico es «sin decidir» y queda en B4 por el rango medido
(`S12-decisions-4.md` §4).

**Resultado:** OK
