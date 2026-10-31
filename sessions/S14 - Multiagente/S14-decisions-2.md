# S14 · Decisiones — Brief 2: adopción de los huecos tipados y filtro de degradaciones

Sin llamadas al proveedor. Juicio de producto en `S14-uat.md` §6.

## Qué cambió

- **D1:** `GENERATION_GAP_KINDS` default `on` (config y `.env.example`). `off` sigue existiendo. El
  `.env` local ya estaba en `on`.
- **D2:** referencia vigente, `gaps` de `generation_20261024-140241` (recall 0,84, precisión 0,86,
  `verified` 1,00, citadas / inyectadas 0,46, mediana 18,3 s, 0,0030 USD por turno). Ni `compare.py`
  ni el arnés la tenían cableada: el cambio es de documentación.
- **D3–D5:** `_DEGRADATION_NOTES` (`frontend/presentation.py`) es a la vez la lista de lo que se
  muestra y su texto: cuatro tipos, un solo sitio. Lo que no está en ella no se pinta. El contrato
  no cambia.

## Consecuencias que el brief no nombraba

- **`xref+bounded` ya no repite los defaults:** lleva los huecos sin tipar. Los defaults son `gaps`.
- **Aislamiento nuevo en la suite:** `tests/app/conftest.py` fija `off` con un fixture autouse,
  porque los dobles escriben `not_found` como cadenas y ni `.env` ni un clone sin él pueden decidir
  el veredicto. Los huecos tipados tienen sus tests explícitos, incluido uno con la opción puesta
  por configuración.
- **El prefijo cacheado** pasa a 3.401 tokens (mediana leída de caché en la pasada).
- **El TODO de `verification.py`** sobre rechazos con hueco tipado sigue en el código: el chat decidió
  dejar el comportamiento, y este brief no toca validación. Se puede retirar cuando se toque.

## Tests

**1537 verde** (1533 + 4). El mismo resultado con `GENERATION_GAP_KINDS=off` en el entorno, y T8 sigue
valiendo.

**Fallo anterior a este brief, no corregido:** sin `.env`, con solo `ANTHROPIC_API_KEY`, fallan 3
tests, y fallan igual con el código commiteado (`test_llm_service_imports_neither_db_nor_models_nor_retrieval`,
`test_app_never_imports_sentence_transformers_nor_torch`,
`test_the_real_router_cannot_reach_a_provider`). Incumple la regla de que la suite de contrato dé el
mismo veredicto en un clone con solo esa clave.

## UAT

Automatizada, sin llamadas: la app Streamlit real con `AppTest` sobre respuestas guardadas
(`S14-uat-2.md`). Los 5 casos, OK.

## Etiquetado

- **Agnóstico:** un aviso casi universal se ignora, así que el filtro es de presentación y el
  contrato lo sigue llevando todo; adoptar por error asimétrico (pesa la precisión de lo servido, no
  el recall, cuando fallar devuelve el comportamiento anterior); una opción con default nuevo pide
  fijar el valor en los dobles de test.
- **Dominio:** las truncaciones son la política `bounded` + `clause`, no un fallo.
