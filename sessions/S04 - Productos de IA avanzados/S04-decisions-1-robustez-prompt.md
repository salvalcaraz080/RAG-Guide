# S04 — Decisions log, brief 1/2: robustez de prompt

> Log crudo de decisiones + porqué. Etiquetado `[agnóstico]` (aplica a cualquier proyecto
> LLM) / `[dominio]` (específico de RAG-ECSS).

---

## 1. Anatomía instrucciones vs contexto `[agnóstico]`

El prompt final tiene dos partes de naturaleza distinta:

- **Instrucciones estáticas** — el *contrato* con el modelo (rol, grounding, citación,
  formato, scope). No cambian por pregunta ni por corpus. Candidatas a versionarse como
  artefacto propio, revisable en una PR sin leer Python.
- **Contexto** — el *dato* inyectado (aquí, el fragmento ECSS de `context/reference.py`).
  Cambia con el corpus (en M2, con cada retrieval).

Esta distinción ya existía en el código (`_STATIC_INSTRUCTIONS` vs `_build_context_block`,
separados desde S03 brief 2 por la clave de caché), pero solo el primero se movió a
archivo. `context/reference.py` no se toca en este brief.

## 2. Sin motor de plantillas `[agnóstico]`

`app/prompts/loader.py` es una función que hace `Path.read_text(encoding="utf-8")`. Nada
más. El prompt de M1 no tiene condicionales ni variables textuales dentro del bloque
estático (la única parte variable, el material de referencia, se sigue componiendo en
Python vía `_build_context_block`, fuera del archivo). Adoptar Jinja2 aquí sería una
dependencia sin beneficio — se revisará si M2/M3 introduce ramas reales dentro del texto
estático.

## 3. Path relativo al módulo, no al cwd `[agnóstico]`

`Path(__file__).parent / "query" / "system.md"` en vez de una ruta relativa al directorio
de trabajo. El proceso arranca desde sitios distintos (Docker, `uv run pytest`, `uv run
uvicorn` local) — una ruta relativa al cwd se habría roto según quién lo lance. Verificado
también dentro del contenedor Docker (`docker compose exec api python -c
"from app.prompts.loader import load_query_system_instructions; ..."` devolvió True).

## 4. `_STATIC_INSTRUCTIONS` se mantiene como nombre/constante de módulo `[dominio]`

En vez de llamar al loader inline dentro de `build_system_prompt`, se asigna el resultado
a `_STATIC_INSTRUCTIONS = load_query_system_instructions()` a nivel de módulo en
`llm_service.py`. Motivo puramente de compatibilidad de tests: los tests de caché
(`test_llm_service_cache.py::test_key_changes_with_system_prompt`) hacen
`monkeypatch.setattr(llm_service, "_STATIC_INSTRUCTIONS", "...")` para probar la
invalidación de clave sin depender del contenido real del archivo. Cambiar el nombre o
inlinear la llamada habría roto ese test sin necesidad — el refactor de este brief es de
*origen* del valor, no de su forma de consumo.

## 5. Dos tipos de rechazo, no uno `[dominio]`

- **Out of domain**: la pregunta no es de ECSS/aeroespacial en absoluto (comida, deporte,
  etc.).
- **Out of material**: la pregunta SÍ es de ECSS, pero el fragmento concreto inyectado
  (`ECSS-E-ST-40C 5.2.4.8`, el único que existe en M1) no la cubre.

El estimador del máster (proyecto hermano) solo necesitaba el primero. Aquí el segundo es
el que importa de verdad para la trazabilidad: es el que impide la cita fantasma cuando el
corpus no tiene la respuesta. Con un solo fragmento en M1, "out of material" se dispara
para casi cualquier pregunta que no toque ese fragmento — comportamiento esperado y
correcto, no un bug a "arreglar"; el refuerzo es preparación para M2 (corpus real), donde
el caso se reducirá a lo que ni el corpus completo cubre.

## 6. Política *filter*, instruida en el prompt, no validación Python `[agnóstico]`

El guardrail de grounding/scope en M1 es responsabilidad del propio modelo (instrucción de
degradar con un mensaje claro), no un validador post-respuesta en código. La verificación
real de que el modelo no alucinó una cláusula (comparar la respuesta contra el material
inyectado, palabra por palabra o semánticamente) es trabajo de S11 — no se fabrica esa
pieza en este brief solo porque el guardrail existe conceptualmente. Las otras dos
políticas del marco (`exception`, `retry`) siguen sin implementarse: son para guardrails de
input (M2) y structured output (S11) que aún no existen.

## 7. Consecuencia de caché: entradas huérfanas tras el refuerzo `[dominio]`

Verificado en dos pasos:

1. **Tras el movimiento puro** (sin el refuerzo de scope): `build_system_prompt` produjo un
   string **byte-idéntico** al de antes del refactor (comparado programáticamente contra
   una captura hecha ANTES de tocar nada) → el slot `system_prompt` de la clave de caché no
   cambia → cero invalidación por el refactor.
2. **Tras el refuerzo** (con la sección de scope añadida): el prompt final cambia →
   `build_cache_key` hashea un `system_prompt` distinto → las entradas cacheadas con el
   prompt viejo quedan huérfanas (nunca más se les acierta) y expiran solas por
   `CACHE_TTL`. Es la invalidación por versión de prompt funcionando correctamente — mismo
   mecanismo que ya existía para `corpus_version` (S03 brief 2). No hace falta borrado
   explícito ni un nuevo slot de versión de prompt.

## 8. Re-verificación provider-agnostic `[agnóstico]`

El prompt reforzado se re-validó contra Anthropic (primario) y OpenAI (fallback, forzado
vía `mock_testing_fallbacks=True`, mismo mecanismo que S03-brief-1-fix) en los 3 escenarios
del brief: respondible por el fragmento, out of domain, out of material. Los 6 tests
(`test_system_prompt_provider_agnostic.py`, marcados `real_llm`) pasan contra las dos APIs
reales.

Nota sobre el test de "out of material": la primera versión de la aserción fallaba porque
ambos modelos, correctamente, MENCIONAN el nombre del estándar preguntado
(`ECSS-E-ST-70-41`) al decir explícitamente que no lo cubren — eso es el comportamiento
deseado, no una alucinación. La aserción se corrigió para prohibir solo una **cláusula
inventada** pegada a ese estándar (`ECSS-E-ST-70-41` seguido de un número), no la mera
mención del nombre.

Los tests `real_llm` de este brief llaman a `call_llm`/`build_system_prompt` directamente
(no al pipeline completo `answer_query`), porque `_citations()` en `llm_service.py`
siempre devuelve la cita candidata del fragmento inyectado **independientemente de si el
modelo realmente respondió o rechazó** (no se extraen del texto generado, ver
`ARCHITECTURE.md` §3 "Contrato de citación"). Aserciones tipo "citations vacío" en un
rechazo solo tienen sentido si se mide contra el TEXTO del modelo, no contra el campo
`citations` de `QueryResponse` — cambiar eso es una decisión de contrato fuera de alcance
de este brief (ver anclaje del brief 1, §7: campo nuevo para señalar rechazo
programáticamente, `# TODO: pendiente de decisión en Claude Chat`).

## 9. Qué NO se tocó (por diseño, no por olvido)

- `context/reference.py` y `_build_context_block`: sin cambios.
- `QueryResponse`/`StreamDoneEvent`: sin campos nuevos. Un rechazo hoy es indistinguible de
  una respuesta *programáticamente* (solo se sabe leyendo el texto) — anclaje explícito
  para brief 2 / S11.
- Sin Jinja2, sin `v1/` de versionado por directorios.
