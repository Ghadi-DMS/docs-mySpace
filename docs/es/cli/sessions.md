---
read_when:
    - Quieres listar las sesiones almacenadas y ver la actividad reciente
summary: Referencia de la CLI para `openclaw sessions` (listar sesiones almacenadas + uso)
title: sessions
x-i18n:
    generated_at: "2026-04-05T12:38:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 47eb55d90bd0681676283310cfa50dcacc95dff7d9a39bf2bb188788c6e5e5ba
    source_path: cli/sessions.md
    workflow: 15
---

# `openclaw sessions`

Lista las sesiones de conversación almacenadas.

```bash
openclaw sessions
openclaw sessions --agent work
openclaw sessions --all-agents
openclaw sessions --active 120
openclaw sessions --verbose
openclaw sessions --json
```

Selección de alcance:

- predeterminado: almacén del agente predeterminado configurado
- `--verbose`: registro detallado
- `--agent <id>`: un almacén de agente configurado
- `--all-agents`: agrega todos los almacenes de agentes configurados
- `--store <path>`: ruta explícita del almacén (no se puede combinar con `--agent` ni con `--all-agents`)

`openclaw sessions --all-agents` lee los almacenes de agentes configurados. El gateway y ACP
tienen un descubrimiento de sesiones más amplio: también incluyen almacenes solo en disco encontrados bajo
la raíz predeterminada `agents/` o una raíz de `session.store` con plantilla. Esos
almacenes descubiertos deben resolverse a archivos regulares `sessions.json` dentro de la raíz
del agente; se omiten enlaces simbólicos y rutas fuera de la raíz.

Ejemplos de JSON:

`openclaw sessions --all-agents --json`:

```json
{
  "path": null,
  "stores": [
    { "agentId": "main", "path": "/home/user/.openclaw/agents/main/sessions/sessions.json" },
    { "agentId": "work", "path": "/home/user/.openclaw/agents/work/sessions/sessions.json" }
  ],
  "allAgents": true,
  "count": 2,
  "activeMinutes": null,
  "sessions": [
    { "agentId": "main", "key": "agent:main:main", "model": "gpt-5" },
    { "agentId": "work", "key": "agent:work:main", "model": "claude-opus-4-6" }
  ]
}
```

## Mantenimiento de limpieza

Ejecuta el mantenimiento ahora (en lugar de esperar al siguiente ciclo de escritura):

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --agent work --dry-run
openclaw sessions cleanup --all-agents --dry-run
openclaw sessions cleanup --enforce
openclaw sessions cleanup --enforce --active-key "agent:main:telegram:direct:123"
openclaw sessions cleanup --json
```

`openclaw sessions cleanup` usa la configuración de `session.maintenance` desde la configuración:

- Nota sobre el alcance: `openclaw sessions cleanup` mantiene solo los almacenes/transcripciones de sesiones. No poda los logs de ejecución de cron (`cron/runs/<jobId>.jsonl`), que se gestionan con `cron.runLog.maxBytes` y `cron.runLog.keepLines` en [Cron configuration](/automation/cron-jobs#configuration) y se explican en [Cron maintenance](/automation/cron-jobs#maintenance).

- `--dry-run`: muestra cuántas entradas se podarían/limitarían sin escribir.
  - En modo texto, la simulación imprime una tabla de acciones por sesión (`Action`, `Key`, `Age`, `Model`, `Flags`) para que puedas ver qué se conservaría y qué se eliminaría.
- `--enforce`: aplica el mantenimiento incluso cuando `session.maintenance.mode` es `warn`.
- `--fix-missing`: elimina entradas cuyos archivos de transcripción faltan, aunque normalmente aún no superen el límite por antigüedad o cantidad.
- `--active-key <key>`: protege una clave activa específica frente a la expulsión por presupuesto de disco.
- `--agent <id>`: ejecuta la limpieza para un almacén de agente configurado.
- `--all-agents`: ejecuta la limpieza para todos los almacenes de agentes configurados.
- `--store <path>`: ejecuta la limpieza sobre un archivo `sessions.json` específico.
- `--json`: imprime un resumen JSON. Con `--all-agents`, la salida incluye un resumen por almacén.

`openclaw sessions cleanup --all-agents --dry-run --json`:

```json
{
  "allAgents": true,
  "mode": "warn",
  "dryRun": true,
  "stores": [
    {
      "agentId": "main",
      "storePath": "/home/user/.openclaw/agents/main/sessions/sessions.json",
      "beforeCount": 120,
      "afterCount": 80,
      "pruned": 40,
      "capped": 0
    },
    {
      "agentId": "work",
      "storePath": "/home/user/.openclaw/agents/work/sessions/sessions.json",
      "beforeCount": 18,
      "afterCount": 18,
      "pruned": 0,
      "capped": 0
    }
  ]
}
```

Relacionado:

- Configuración de sesiones: [Configuration reference](/gateway/configuration-reference#session)
