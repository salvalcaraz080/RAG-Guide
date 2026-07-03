=================== INICIO PLANTILLA SNN-brief.md ===================
# SNN — Brief de implementación

> Rama: `sesion-NN`. Al terminar, esta rama se fusiona a `main`.

## Objetivo de la sesión
(2–3 líneas: qué capacidad añade esta sesión al producto ECSS.)

## Decisiones ya cerradas (no re-decidir)
- DECISIÓN: … → ENFOQUE ELEGIDO: … (alternativa descartada: … / porqué: …)
- …

## Restricciones vigentes (heredadas del CLAUDE.md del repo)
- `uv` siempre, nunca `pip`. Toda dependencia vía `uv add`.
- Proveedor vía LiteLLM; nunca llamar a OpenAI/Anthropic directamente.
- (otras restricciones relevantes para esta sesión)

## Unidades de implementación (en orden)
Cada unidad cierra su ciclo completo antes de pasar a la siguiente.

### Unidad 1 — <nombre>
- Implementación: …
- Tests automáticos: …
- Tests de usuario (qué debe verificar el humano a mano): …

### Unidad 2 — <nombre>
- …

## Entregables de cierre (obligatorios, ver CLAUDE.md § Feedback loop)
- [ ] `sesiones/SNN-decisions.md` escrito (log crudo + etiqueta agnóstico/dominio).
- [ ] Entrada de `CHANGELOG.md` derivada de decisions.
- [ ] `CLAUDE.md` y `ARCHITECTURE.md` actualizados a estado actual (sin narrativa histórica).

## Qué NO hacer
- No pre-formatear `SNN-decisions.md` al esquema de la Guide (madurez/condición/señal).
- No añadir features que el máster introduce en sesiones posteriores.
- No meter narrativa histórica en CLAUDE.md/ARCHITECTURE.md (va al CHANGELOG).
==================== FIN PLANTILLA SNN-brief.md ====================