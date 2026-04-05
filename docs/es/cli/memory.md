---
read_when:
    - Quieres indexar o buscar memoria semántica
    - Estás depurando la disponibilidad o la indexación de memoria
    - Quieres promover memoria recuperada de corto plazo a `MEMORY.md`
summary: Referencia de CLI para `openclaw memory` (estado/indexación/búsqueda/promoción)
title: memory
x-i18n:
    generated_at: "2026-04-05T12:38:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: a89e3a819737bb63521128ae63d9e25b5cd9db35c3ea4606d087a8ad48b41eab
    source_path: cli/memory.md
    workflow: 15
---

# `openclaw memory`

Gestiona la indexación y la búsqueda de memoria semántica.
La proporciona el plugin de memoria activo (predeterminado: `memory-core`; configura `plugins.slots.memory = "none"` para deshabilitarlo).

Relacionado:

- Concepto de memoria: [Memory](/concepts/memory)
- Plugins: [Plugins](/tools/plugin)

## Ejemplos

```bash
openclaw memory status
openclaw memory status --deep
openclaw memory status --fix
openclaw memory index --force
openclaw memory search "meeting notes"
openclaw memory search --query "deployment" --max-results 20
openclaw memory promote --limit 10 --min-score 0.75
openclaw memory promote --apply
openclaw memory promote --json --min-recall-count 0 --min-unique-queries 0
openclaw memory status --json
openclaw memory status --deep --index
openclaw memory status --deep --index --verbose
openclaw memory status --agent main
openclaw memory index --agent main --verbose
```

## Opciones

`memory status` y `memory index`:

- `--agent <id>`: limita el alcance a un solo agente. Sin él, estos comandos se ejecutan para cada agente configurado; si no hay ninguna lista de agentes configurada, recurren al agente predeterminado.
- `--verbose`: emite logs detallados durante los sondeos y la indexación.

`memory status`:

- `--deep`: sondea la disponibilidad de vectores e incrustaciones.
- `--index`: ejecuta una nueva indexación si el almacén está sucio (implica `--deep`).
- `--fix`: repara bloqueos obsoletos de recuperación y normaliza metadatos de promoción.
- `--json`: imprime salida JSON.

`memory index`:

- `--force`: fuerza una reindexación completa.

`memory search`:

- Entrada de consulta: pasa `[query]` posicional o `--query <text>`.
- Si se proporcionan ambos, `--query` tiene prioridad.
- Si no se proporciona ninguno, el comando termina con un error.
- `--agent <id>`: limita el alcance a un solo agente (predeterminado: el agente predeterminado).
- `--max-results <n>`: limita el número de resultados devueltos.
- `--min-score <n>`: filtra coincidencias con puntuación baja.
- `--json`: imprime resultados en JSON.

`memory promote`:

Previsualiza y aplica promociones de memoria de corto plazo.

```bash
openclaw memory promote [--apply] [--limit <n>] [--include-promoted]
```

- `--apply` -- escribe promociones en `MEMORY.md` (predeterminado: solo vista previa).
- `--limit <n>` -- limita el número de candidatos mostrados.
- `--include-promoted` -- incluye entradas ya promovidas en ciclos anteriores.

Opciones completas:

- Clasifica candidatos de corto plazo de `memory/YYYY-MM-DD.md` usando señales ponderadas de recuperación (`frequency`, `relevance`, `query diversity`, `recency`).
- Usa eventos de recuperación capturados cuando `memory_search` devuelve aciertos de memoria diaria.
- Modo opcional de dream automático: cuando `plugins.entries.memory-core.config.dreaming.mode` es `core`, `deep` o `rem`, `memory-core` gestiona automáticamente un trabajo cron que activa la promoción en segundo plano (no se requiere `openclaw cron add` manual).
- `--agent <id>`: limita el alcance a un solo agente (predeterminado: el agente predeterminado).
- `--limit <n>`: número máximo de candidatos para devolver o aplicar.
- `--min-score <n>`: puntuación mínima ponderada de promoción.
- `--min-recall-count <n>`: recuento mínimo de recuperaciones requerido para un candidato.
- `--min-unique-queries <n>`: número mínimo de consultas distintas requerido para un candidato.
- `--apply`: agrega los candidatos seleccionados a `MEMORY.md` y los marca como promovidos.
- `--include-promoted`: incluye en la salida candidatos ya promovidos.
- `--json`: imprime salida JSON.

## Dreaming (experimental)

Dreaming es la pasada nocturna de reflexión para la memoria. Se llama "dreaming" porque el sistema vuelve a examinar lo que se recuperó durante el día y decide qué merece conservarse a largo plazo.

- Es opcional y está deshabilitado de forma predeterminada.
- Habilítalo con `plugins.entries.memory-core.config.dreaming.mode`.
- Puedes alternar modos desde el chat con `/dreaming off|core|rem|deep`. Ejecuta `/dreaming` (o `/dreaming options`) para ver qué hace cada modo.
- Cuando está habilitado, `memory-core` crea y mantiene automáticamente un trabajo cron gestionado.
- Configura `dreaming.limit` en `0` si quieres tener dreaming habilitado pero con la promoción automática efectivamente en pausa.
- La clasificación usa señales ponderadas: frecuencia de recuperación, relevancia de recuperación, diversidad de consultas y recencia temporal (las recuperaciones recientes decaen con el tiempo).
- La promoción a `MEMORY.md` solo ocurre cuando se cumplen umbrales de calidad, por lo que la memoria a largo plazo mantiene una señal alta en lugar de acumular detalles aislados.

Preajustes de modo predeterminados:

- `core`: diariamente a las `0 3 * * *`, `minScore=0.75`, `minRecallCount=3`, `minUniqueQueries=2`
- `deep`: cada 12 horas (`0 */12 * * *`), `minScore=0.8`, `minRecallCount=3`, `minUniqueQueries=3`
- `rem`: cada 6 horas (`0 */6 * * *`), `minScore=0.85`, `minRecallCount=4`, `minUniqueQueries=3`

Ejemplo:

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "mode": "core"
          }
        }
      }
    }
  }
}
```

Notas:

- `memory index --verbose` imprime detalles por fase (proveedor, modelo, fuentes, actividad de lotes).
- `memory status` incluye cualquier ruta adicional configurada mediante `memorySearch.extraPaths`.
- Si los campos de clave de API remota de memoria efectivamente activos están configurados como SecretRefs, el comando resuelve esos valores desde la instantánea activa del gateway. Si el gateway no está disponible, el comando falla de inmediato.
- Nota sobre discrepancia de versión del gateway: esta ruta de comando requiere un gateway compatible con `secrets.resolve`; los gateways antiguos devuelven un error de método desconocido.
- La cadencia de dreaming usa de forma predeterminada la programación preestablecida de cada modo. Sustituye la cadencia con `plugins.entries.memory-core.config.dreaming.frequency` como expresión cron (por ejemplo `0 3 * * *`) y ajústala con `timezone`, `limit`, `minScore`, `minRecallCount` y `minUniqueQueries`.
