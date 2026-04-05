---
read_when:
    - Quieres conectar eventos de Gmail Pub/Sub a OpenClaw
    - Quieres comandos auxiliares de webhook
summary: Referencia de la CLI para `openclaw webhooks` (utilidades de webhook + Gmail Pub/Sub)
title: webhooks
x-i18n:
    generated_at: "2026-04-05T12:39:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2b22ce879c3a94557be57919b4d2b3e92ff4d41fbae7bc88d2ab07cd4bbeac83
    source_path: cli/webhooks.md
    workflow: 15
---

# `openclaw webhooks`

Utilidades e integraciones de webhooks (Gmail Pub/Sub, utilidades de webhook).

Relacionado:

- Webhooks: [Webhooks](/automation/cron-jobs#webhooks)
- Gmail Pub/Sub: [Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration)

## Gmail

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail run
```

### `webhooks gmail setup`

Configura Gmail watch, Pub/Sub y la entrega de webhooks de OpenClaw.

Obligatorio:

- `--account <email>`

Opciones:

- `--project <id>`
- `--topic <name>`
- `--subscription <name>`
- `--label <label>`
- `--hook-url <url>`
- `--hook-token <token>`
- `--push-token <token>`
- `--bind <host>`
- `--port <port>`
- `--path <path>`
- `--include-body`
- `--max-bytes <n>`
- `--renew-minutes <n>`
- `--tailscale <funnel|serve|off>`
- `--tailscale-path <path>`
- `--tailscale-target <target>`
- `--push-endpoint <url>`
- `--json`

Ejemplos:

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

### `webhooks gmail run`

Ejecuta `gog watch serve` junto con el bucle de renovación automática de watch.

Opciones:

- `--account <email>`
- `--topic <topic>`
- `--subscription <name>`
- `--label <label>`
- `--hook-url <url>`
- `--hook-token <token>`
- `--push-token <token>`
- `--bind <host>`
- `--port <port>`
- `--path <path>`
- `--include-body`
- `--max-bytes <n>`
- `--renew-minutes <n>`
- `--tailscale <funnel|serve|off>`
- `--tailscale-path <path>`
- `--tailscale-target <target>`

Ejemplo:

```bash
openclaw webhooks gmail run --account you@example.com
```

Consulta la [documentación de Gmail Pub/Sub](/automation/cron-jobs#gmail-pubsub-integration) para ver el flujo de configuración integral y los detalles operativos.
