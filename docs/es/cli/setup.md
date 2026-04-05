---
read_when:
    - Estás realizando la configuración inicial sin la incorporación completa de la CLI
    - Quieres establecer la ruta predeterminada del espacio de trabajo
summary: Referencia de CLI para `openclaw setup` (inicializar configuración + espacio de trabajo)
title: setup
x-i18n:
    generated_at: "2026-04-05T12:38:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: f538aac341c749043ad959e35f2ed99c844ab8c3500ff59aa159d940bd301792
    source_path: cli/setup.md
    workflow: 15
---

# `openclaw setup`

Inicializa `~/.openclaw/openclaw.json` y el espacio de trabajo del agente.

Relacionado:

- Primeros pasos: [Getting started](/es/start/getting-started)
- Incorporación de CLI: [Onboarding (CLI)](/es/start/wizard)

## Ejemplos

```bash
openclaw setup
openclaw setup --workspace ~/.openclaw/workspace
openclaw setup --wizard
openclaw setup --non-interactive --mode remote --remote-url wss://gateway-host:18789 --remote-token <token>
```

## Opciones

- `--workspace <dir>`: directorio del espacio de trabajo del agente (almacenado como `agents.defaults.workspace`)
- `--wizard`: ejecutar la incorporación
- `--non-interactive`: ejecutar la incorporación sin solicitudes
- `--mode <local|remote>`: modo de incorporación
- `--remote-url <url>`: URL de WebSocket del Gateway remoto
- `--remote-token <token>`: token del Gateway remoto

Para ejecutar la incorporación mediante setup:

```bash
openclaw setup --wizard
```

Notas:

- `openclaw setup` simple inicializa la configuración y el espacio de trabajo sin el flujo completo de incorporación.
- La incorporación se ejecuta automáticamente cuando hay presentes indicadores de incorporación (`--wizard`, `--non-interactive`, `--mode`, `--remote-url`, `--remote-token`).
