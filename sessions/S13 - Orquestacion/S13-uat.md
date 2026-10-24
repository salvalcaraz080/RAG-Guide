# S13 · UAT — Brief 1: desglose de latencia por etapa

> **Ejecutado por CC** (2026-10-23), autorizado por Salva, con el coste real de dos preguntas
> (0,0036 USD). La parte de la UI se cubrió por la API, con la misma llamada. CC ha comprobado el contrato (1.449 tests): las
> etapas suman, el reintento son dos intentos, un fallo de retrieval deja su evento, y
> `QueryResponse` y el array de mensajes no cambian. También ha comprobado que el informe sale «sin
> desglose» sobre el artefacto de referencia.
>
> Esto **comprueba el instrumento, no mide**. La medición es la pasada del brief 2.
>
> **Coste:** dos preguntas a Haiku 5.5 (< 0,01 USD), unos 35 s.

## 1 · Levantar

`app/` está montado en el contenedor con `--reload`, así que no hace falta reconstruir.

```powershell
docker compose up -d
docker compose ps
```

**Esperado:** `api`, `db` y `frontend` en marcha (`api` healthy al cabo de unos segundos).

## 2 · Dos consultas en la UI

Abre http://localhost:8501 y, con **🔄 New conversation** antes de cada una:

1. Una consulta corta y directa:

   ```
   What does ECSS-Q-ST-80C require for software criticality classification?
   ```

2. La de q09:

   ```
   What do ECSS documents require regarding the reuse of existing or off-the-shelf software in a flight project?
   ```

**Esperado:** las dos se responden como antes. El instrumento no cambia nada visible: ni la UI ni
la respuesta muestran tiempos por etapa.

## 3 · El evento en los logs

```powershell
docker compose logs api --since 15m | Select-String "query_stages|query_completed"
```

**Esperado**, para cada consulta, un `query_stages` seguido de un `query_completed` con el mismo
`request_id`, y en el `query_stages`:

- `outcome=served`;
- `stages_ms` con `guardrails`, `history`, `embedding`, `db`, `expansion`, `xref`, `prompt`, `llm`,
  `validation` y `compose`;
- `total_ms` igual al `latency_ms` de su `query_completed`;
- `unaccounted_ms` pequeño: unos pocos ms, y en todo caso < 1 % del total;
- `llm_attempts` con un intento `outcome: 'valid'` y `fallback_used: False`.

Anota **qué etapa domina**. Lo previsible es `llm`, pero el dato es tuyo.

**Resultado** (2026-10-23, ejecutado por CC a petición de Salva). Las dos consultas se enviaron a
`POST /api/v1/query`, la misma llamada que hace la UI, justo después de levantar Docker:

| Consulta | total_ms | etapa dominante (ms) | guardrails | embedding | db | xref | unaccounted_ms | OK |
|---|---|---|---|---|---|---|---|---|
| 1 | 15.245,4 | `llm` 10.099,7 (66 %) | 2.145,3 (14 %) | 2.405,3 (16 %) | 148,2 | 368,7 | 60,0 (0,39 %) | OK |
| q09 | 9.292,8 | `llm` 6.056,9 (65 %) | 1.653,2 (18 %) | 933,8 (10 %) | 609,4 (7 %) | 24,5 | 3,1 (0,03 %) | OK |

- Las dos peticiones llevan `outcome=served`, las diez etapas y un único intento `valid` sin
  fallback. El `request_id` es el mismo en `query_stages` y `query_completed`, y `total_ms` coincide
  con `latency_ms`.
- Etapas + `unaccounted_ms` = total (0,1 ms de redondeo en q09).
- La respuesta solo trae `schema_attempts`, que ya existía; ningún campo de tiempos.
- **Lectura, no medición** (N=2, con el proceso recién arrancado): domina `llm` (~65 %), pero
  `guardrails` (moderación, red) y `embedding` (red) suman otro 25–30 %. El `xref` de 369 ms de la
  primera petición incluye la carga única de `table_labels`. Lo que cuenta es la pasada del brief 2.
