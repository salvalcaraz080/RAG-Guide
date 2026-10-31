# S14 · UAT — Brief 2: el desplegable con el filtro nuevo

> **Ejecutada por CC** (2026-10-30). La app real (`frontend/streamlit_app.py`) con `AppTest`, dentro
> del contenedor `frontend`, con `backend_client` sustituido por las respuestas `gaps` de
> `generation_20261024-140241`. Sin llamadas al proveedor, como pide el brief. El guion del brief
> preguntaba en la UI; las respuestas guardadas dan los mismos casos sin coste.
>
> **Para Salva, opcional:** esto comprueba qué se pinta, no cómo se ve. Si quieres juzgar el aspecto,
> pregunta q14 turno 3 en http://localhost:8501 (un turno, ≈ 0,003 USD).
>
> **Aspecto, juzgado por Salva (2026-10-31): parcial.** El desplegable sale bien, pero la lista larga
> de cláusulas no le sirve al usuario. Se revisa en la sesión de UI/UX, con el producto terminado.

| Caso | `degradations` del contrato | Desplegable | Líneas | Resultado |
|---|---|---|---|---|
| q14 turno 3 | `expansion_truncated`, `xref_over_limit`, `xref_truncated` | «⚙️ Technical limits on this answer (1)», cerrado | solo «Cross-references not followed, the limit was reached: ECSS-E-ST-70-41C §6.2.6, …» | **OK** |
| q02 | ninguna | no | — | **OK** |
| q04 | `xref_truncated` | no | — | **OK** |
| q08 | `xref_truncated` | no | — | **OK** |
| q13 | `expansion_truncated` | no | — | **OK** |

El contrato conserva las truncaciones en los cinco (D4).

**Default nuevo:** la API, recreada, sirve `on` y el prompt activo lleva la regla de los tres tipos.
