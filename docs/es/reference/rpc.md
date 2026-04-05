---
read_when:
    - Añades o cambias integraciones con CLI externos
    - Depuras adaptadores RPC (signal-cli, imsg)
summary: Adaptadores RPC para CLI externos (signal-cli, imsg heredado) y patrones del gateway
title: Adaptadores RPC
x-i18n:
    generated_at: "2026-04-05T12:52:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 06dc6b97184cc704ba4ec4a9af90502f4316bcf717c3f4925676806d8b184c57
    source_path: reference/rpc.md
    workflow: 15
---

# Adaptadores RPC

OpenClaw integra CLI externos mediante JSON-RPC. Hoy se usan dos patrones.

## Patrón A: daemon HTTP (signal-cli)

- `signal-cli` se ejecuta como daemon con JSON-RPC sobre HTTP.
- El stream de eventos es SSE (`/api/v1/events`).
- Sonda de estado: `/api/v1/check`.
- OpenClaw controla el ciclo de vida cuando `channels.signal.autoStart=true`.

Consulta [Signal](/channels/signal) para ver configuración y endpoints.

## Patrón B: proceso hijo stdio (heredado: imsg)

> **Nota:** Para configuraciones nuevas de iMessage, usa [BlueBubbles](/channels/bluebubbles) en su lugar.

- OpenClaw genera `imsg rpc` como proceso hijo (integración heredada de iMessage).
- JSON-RPC está delimitado por líneas sobre stdin/stdout (un objeto JSON por línea).
- No requiere puerto TCP ni daemon.

Métodos centrales usados:

- `watch.subscribe` → notificaciones (`method: "message"`)
- `watch.unsubscribe`
- `send`
- `chats.list` (sonda/diagnósticos)

Consulta [iMessage](/channels/imessage) para ver la configuración heredada y el direccionamiento (se prefiere `chat_id`).

## Directrices para adaptadores

- El Gateway controla el proceso (inicio/parada vinculados al ciclo de vida del proveedor).
- Mantén clientes RPC resilientes: tiempos de espera, reinicio al salir.
- Prefiere IDs estables (por ejemplo `chat_id`) frente a cadenas de visualización.
