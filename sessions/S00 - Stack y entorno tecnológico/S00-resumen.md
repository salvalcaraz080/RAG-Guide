# S00 — Panorama (pre-curso)

## Stack técnico del programa
- Backend en Python 3.11+ con FastAPI (async nativo, validación Pydantic, docs OpenAPI automática) + Uvicorn.
- Gestión de dependencias con `uv` (venv + lockfile reproducible).
- Todo el entorno corre en Docker / Docker Compose (backend, DB, frontend con un único `docker-compose up`).
- El máster usa Streamlit como frontend de prototipado para los ejercicios; separación explícita backend (FastAPI, API REST) / frontend, consumible por cualquier cliente.
- El `docker-compose` del máster trae PostgreSQL con soporte vectorial (pgvector) como default.

## Panorama de modelos (a fecha del pre-curso)
- Dos familias principales: LLMs (generación/razonamiento) y modelos de embeddings (búsqueda semántica).
- Cada proveedor organiza su catálogo en tiers: modelo de referencia (máxima capacidad), modelo de equilibrio (coste/calidad), modelo ligero (latencia/coste mínimo).
- Los nombres concretos de modelo rotan cada pocas semanas — no fijar en código sin capa de indirección (config/wrapper).

## Decisiones para RAG-ECSS
Ninguna tomada en esta sesión — es contenido de instalación y panorama, sin fricción de implementación todavía.

## Abierto / pendiente de decidir
- Frontend de consulta para ECSS: no se hereda Streamlit por defecto, se decide cuando corresponda.
- Vector store: no se hereda pgvector por defecto, se decide en S08 (comparativa de BBDD vectoriales).
- Configuración técnica del entorno (Docker/uv/FastAPI) se documentará en `README.md` de `RAG-ECSS`, derivada de lo que efectivamente se instale en S01, no de esta guía genérica.

## Próximo paso
S01 — LLMs + setup.
