---
read_when:
    - Depurar indicadores de estado de la app de Mac
summary: Cómo informa la app de macOS los estados de salud del gateway/Baileys
title: Comprobaciones de estado (macOS)
x-i18n:
    generated_at: "2026-04-05T12:48:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: f9223b2bbe272b32526f79cf878510ac5104e788402d94a1b1627e72c5fbebf5
    source_path: platforms/mac/health.md
    workflow: 15
---

# Comprobaciones de estado en macOS

Cómo ver desde la app de la barra de menús si el canal vinculado está en buen estado.

## Barra de menús

- El punto de estado ahora refleja el estado de Baileys:
  - Verde: vinculado + socket abierto recientemente.
  - Naranja: conectando/reintentando.
  - Rojo: sesión cerrada o sondeo fallido.
- La línea secundaria dice "linked · auth 12m" o muestra el motivo del fallo.
- El elemento de menú "Run Health Check" activa un sondeo bajo demanda.

## Configuración

- La pestaña General gana una tarjeta de estado que muestra: antigüedad de autenticación vinculada, ruta/recuento del almacén de sesiones, hora de la última comprobación, último error/código de estado y botones para Run Health Check / Reveal Logs.
- Usa una instantánea en caché para que la IU cargue al instante y recurra a una degradación elegante cuando esté sin conexión.
- La pestaña **Channels** muestra el estado del canal + controles para WhatsApp/Telegram (QR de inicio de sesión, cierre de sesión, sondeo, última desconexión/error).

## Cómo funciona el sondeo

- La app ejecuta `openclaw health --json` mediante `ShellExecutor` cada ~60 s y bajo demanda. El sondeo carga credenciales e informa el estado sin enviar mensajes.
- Almacena por separado en caché la última instantánea correcta y el último error para evitar parpadeos; muestra la marca de tiempo de cada uno.

## En caso de duda

- Aún puedes usar el flujo de la CLI de [Estado del gateway](/gateway/health) (`openclaw status`, `openclaw status --deep`, `openclaw health --json`) y seguir `/tmp/openclaw/openclaw-*.log` para `web-heartbeat` / `web-reconnect`.
