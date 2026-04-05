---
read_when:
    - Diagnosticas conectividad de canales o el estado del gateway
    - Quieres entender los comandos y opciones de CLI para comprobación de estado
summary: Comandos de comprobación de estado y supervisión del estado del gateway
title: Comprobaciones de estado
x-i18n:
    generated_at: "2026-04-05T12:41:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: b8824bca34c4d1139f043481c75f0a65d83e54008898c34cf69c6f98fd04e819
    source_path: gateway/health.md
    workflow: 15
---

# Comprobaciones de estado (CLI)

Guía breve para verificar la conectividad de canales sin tener que adivinar.

## Comprobaciones rápidas

- `openclaw status` — resumen local: accesibilidad/modo del gateway, sugerencia de actualización, antigüedad de autenticación de canales vinculados, sesiones y actividad reciente.
- `openclaw status --all` — diagnóstico local completo (solo lectura, con color, seguro para pegar al depurar).
- `openclaw status --deep` — pide al gateway en ejecución un sondeo de estado en vivo (`health` con `probe:true`), incluidas sondas por cuenta de canal cuando se admiten.
- `openclaw health` — pide al gateway en ejecución su instantánea de estado (solo WS; sin sockets directos de canal desde la CLI).
- `openclaw health --verbose` — fuerza una sonda de estado en vivo e imprime detalles de conexión del gateway.
- `openclaw health --json` — salida de instantánea de estado legible por máquina.
- Envía `/status` como mensaje independiente en WhatsApp/WebChat para obtener una respuesta de estado sin invocar al agente.
- Logs: sigue `/tmp/openclaw/openclaw-*.log` y filtra por `web-heartbeat`, `web-reconnect`, `web-auto-reply`, `web-inbound`.

## Diagnósticos profundos

- Credenciales en disco: `ls -l ~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (el `mtime` debería ser reciente).
- Almacén de sesiones: `ls -l ~/.openclaw/agents/<agentId>/sessions/sessions.json` (la ruta puede anularse en la configuración). El recuento y los destinatarios recientes aparecen en `status`.
- Flujo de revinculación: `openclaw channels logout && openclaw channels login --verbose` cuando aparezcan códigos de estado 409–515 o `loggedOut` en los logs. (Nota: el flujo de inicio de sesión por QR se reinicia automáticamente una vez para el estado 515 tras el emparejamiento).

## Configuración del monitor de estado

- `gateway.channelHealthCheckMinutes`: con qué frecuencia el gateway comprueba el estado de los canales. Predeterminado: `5`. Configura `0` para desactivar globalmente los reinicios del monitor de estado.
- `gateway.channelStaleEventThresholdMinutes`: cuánto tiempo puede permanecer inactivo un canal conectado antes de que el monitor de estado lo considere obsoleto y lo reinicie. Predeterminado: `30`. Mantén este valor mayor o igual que `gateway.channelHealthCheckMinutes`.
- `gateway.channelMaxRestartsPerHour`: límite móvil de una hora para reinicios del monitor de estado por canal/cuenta. Predeterminado: `10`.
- `channels.<provider>.healthMonitor.enabled`: desactiva los reinicios del monitor de estado para un canal específico mientras mantiene habilitada la supervisión global.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: anulación para varias cuentas que prevalece sobre la configuración a nivel de canal.
- Estas anulaciones por canal se aplican a los monitores de canal integrados que las exponen actualmente: Discord, Google Chat, iMessage, Microsoft Teams, Signal, Slack, Telegram y WhatsApp.

## Cuando algo falla

- `logged out` o estado 409–515 → vuelve a vincular con `openclaw channels logout` y luego `openclaw channels login`.
- Gateway inaccesible → inícialo: `openclaw gateway --port 18789` (usa `--force` si el puerto está ocupado).
- No hay mensajes entrantes → confirma que el teléfono vinculado esté en línea y que el remitente esté permitido (`channels.whatsapp.allowFrom`); para chats grupales, asegúrate de que las reglas de lista de permitidos y mención coincidan (`channels.whatsapp.groups`, `agents.list[].groupChat.mentionPatterns`).

## Comando dedicado "health"

`openclaw health` pide al gateway en ejecución su instantánea de estado (sin sockets directos de canal desde la CLI). De forma predeterminada, puede devolver una instantánea reciente en caché del gateway; el gateway luego actualiza esa caché en segundo plano. `openclaw health --verbose` fuerza en su lugar una sonda en vivo. El comando informa la antigüedad de credenciales o autenticación vinculadas cuando está disponible, resúmenes de sonda por canal, resumen del almacén de sesiones y duración de la sonda. Sale con código distinto de cero si no se puede acceder al gateway o si la sonda falla o agota el tiempo de espera.

Opciones:

- `--json`: salida JSON legible por máquina
- `--timeout <ms>`: anula el tiempo de espera predeterminado de la sonda de 10 s
- `--verbose`: fuerza una sonda en vivo e imprime detalles de conexión del gateway
- `--debug`: alias de `--verbose`

La instantánea de estado incluye: `ok` (booleano), `ts` (marca de tiempo), `durationMs` (tiempo de sonda), estado por canal, disponibilidad del agente y resumen del almacén de sesiones.
