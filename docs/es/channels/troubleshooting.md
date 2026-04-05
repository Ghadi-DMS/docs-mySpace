---
read_when:
    - El transporte del canal indica que está conectado, pero las respuestas fallan
    - Necesitas comprobaciones específicas del canal antes de consultar documentación profunda del proveedor
summary: Resolución rápida de problemas a nivel de canal con firmas de errores y correcciones por canal
title: Resolución de problemas de canales
x-i18n:
    generated_at: "2026-04-05T12:36:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: d45d8220505ea420d970b20bc66e65216c2d7024b5736db1936421ffc0676e1f
    source_path: channels/troubleshooting.md
    workflow: 15
---

# Resolución de problemas de canales

Usa esta página cuando un canal se conecte pero el comportamiento sea incorrecto.

## Secuencia de comandos

Ejecuta estos comandos en este orden primero:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Línea base correcta:

- `Runtime: running`
- `RPC probe: ok`
- El sondeo del canal muestra el transporte conectado y, cuando es compatible, `works` o `audit ok`

## WhatsApp

### Firmas de error de WhatsApp

| Síntoma                         | Comprobación más rápida                              | Corrección                                              |
| ------------------------------- | ---------------------------------------------------- | ------------------------------------------------------- |
| Conectado pero sin respuestas a mensajes directos | `openclaw pairing list whatsapp`                     | Aprueba el remitente o cambia la política de mensajes directos/lista de permitidos. |
| Los mensajes de grupo se ignoran | Comprueba `requireMention` + patrones de mención en la configuración | Menciona al bot o flexibiliza la política de mención para ese grupo. |
| Desconexiones aleatorias/bucles de nuevo inicio de sesión | `openclaw channels status --probe` + registros       | Vuelve a iniciar sesión y verifica que el directorio de credenciales esté en buen estado. |

Resolución completa de problemas: [/channels/whatsapp#troubleshooting](/channels/whatsapp#troubleshooting)

## Telegram

### Firmas de error de Telegram

| Síntoma                             | Comprobación más rápida                        | Corrección                                                                  |
| ----------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------------- |
| `/start` pero sin un flujo de respuesta utilizable | `openclaw pairing list telegram`               | Aprueba el emparejamiento o cambia la política de mensajes directos.         |
| El bot está en línea pero el grupo permanece en silencio | Verifica el requisito de mención y el modo privacidad del bot | Desactiva el modo privacidad para la visibilidad del grupo o menciona al bot. |
| Fallos de envío con errores de red  | Inspecciona los registros para ver fallos de llamadas a la API de Telegram | Corrige el enrutamiento DNS/IPv6/proxy hacia `api.telegram.org`.             |
| `setMyCommands` rechazado al iniciar | Inspecciona los registros para ver `BOT_COMMANDS_TOO_MUCH` | Reduce los comandos personalizados/de plugins/Skills de Telegram o desactiva los menús nativos. |
| Actualizaste y la lista de permitidos te bloquea | `openclaw security audit` y listas de permitidos de la configuración | Ejecuta `openclaw doctor --fix` o sustituye `@username` por IDs numéricos de remitente. |

Resolución completa de problemas: [/channels/telegram#troubleshooting](/channels/telegram#troubleshooting)

## Discord

### Firmas de error de Discord

| Síntoma                         | Comprobación más rápida              | Corrección                                                |
| ------------------------------- | ------------------------------------ | --------------------------------------------------------- |
| El bot está en línea pero no responde en el servidor | `openclaw channels status --probe`   | Permite el servidor/canal y verifica la intención de contenido del mensaje. |
| Los mensajes de grupo se ignoran | Comprueba en los registros descartes por control de menciones | Menciona al bot o establece `requireMention: false` para el servidor/canal. |
| Faltan respuestas a mensajes directos | `openclaw pairing list discord`      | Aprueba el emparejamiento de mensajes directos o ajusta la política de mensajes directos. |

Resolución completa de problemas: [/channels/discord#troubleshooting](/channels/discord#troubleshooting)

## Slack

### Firmas de error de Slack

| Síntoma                                | Comprobación más rápida                      | Corrección                                                                                                                                              |
| -------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| El modo Socket está conectado pero no hay respuestas | `openclaw channels status --probe`           | Verifica el token de la app + el token del bot y los ámbitos requeridos; observa `botTokenStatus` / `appTokenStatus = configured_unavailable` en configuraciones respaldadas por SecretRef. |
| Mensajes directos bloqueados           | `openclaw pairing list slack`                | Aprueba el emparejamiento o flexibiliza la política de mensajes directos.                                                                              |
| Mensaje de canal ignorado              | Comprueba `groupPolicy` y la lista de permitidos del canal | Permite el canal o cambia la política a `open`.                                                                                                         |

Resolución completa de problemas: [/channels/slack#troubleshooting](/channels/slack#troubleshooting)

## iMessage y BlueBubbles

### Firmas de error de iMessage y BlueBubbles

| Síntoma                          | Comprobación más rápida                                                    | Corrección                                            |
| -------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------- |
| No hay eventos entrantes         | Verifica la accesibilidad del webhook/servidor y los permisos de la app    | Corrige la URL del webhook o el estado del servidor BlueBubbles. |
| Puede enviar pero no recibir en macOS | Comprueba los permisos de privacidad de macOS para la automatización de Messages | Vuelve a conceder permisos TCC y reinicia el proceso del canal. |
| Remitente de mensaje directo bloqueado | `openclaw pairing list imessage` o `openclaw pairing list bluebubbles`     | Aprueba el emparejamiento o actualiza la lista de permitidos. |

Resolución completa de problemas:

- [/channels/imessage#troubleshooting](/channels/imessage#troubleshooting)
- [/channels/bluebubbles#troubleshooting](/channels/bluebubbles#troubleshooting)

## Signal

### Firmas de error de Signal

| Síntoma                         | Comprobación más rápida               | Corrección                                               |
| ------------------------------- | ------------------------------------- | -------------------------------------------------------- |
| El daemon es accesible pero el bot está en silencio | `openclaw channels status --probe`    | Verifica la URL/cuenta del daemon `signal-cli` y el modo de recepción. |
| Mensaje directo bloqueado       | `openclaw pairing list signal`        | Aprueba el remitente o ajusta la política de mensajes directos. |
| Las respuestas de grupo no se activan | Comprueba la lista de permitidos del grupo y los patrones de mención | Añade el remitente/grupo o flexibiliza el control.       |

Resolución completa de problemas: [/channels/signal#troubleshooting](/channels/signal#troubleshooting)

## QQ Bot

### Firmas de error de QQ Bot

| Síntoma                         | Comprobación más rápida                         | Corrección                                                      |
| ------------------------------- | ----------------------------------------------- | --------------------------------------------------------------- |
| El bot responde "gone to Mars"  | Verifica `appId` y `clientSecret` en la configuración | Configura las credenciales o reinicia el gateway.               |
| No hay mensajes entrantes       | `openclaw channels status --probe`              | Verifica las credenciales en la plataforma abierta de QQ.       |
| La voz no se transcribe         | Comprueba la configuración del proveedor de STT | Configura `channels.qqbot.stt` o `tools.media.audio`.           |
| Los mensajes proactivos no llegan | Comprueba los requisitos de interacción de la plataforma QQ | QQ puede bloquear mensajes iniciados por el bot sin interacción reciente. |

Resolución completa de problemas: [/channels/qqbot#troubleshooting](/channels/qqbot#troubleshooting)

## Matrix

### Firmas de error de Matrix

| Síntoma                             | Comprobación más rápida                   | Corrección                                                                |
| ----------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------- |
| Has iniciado sesión pero ignora mensajes de sala | `openclaw channels status --probe`        | Comprueba `groupPolicy`, la lista de permitidos de salas y el control por menciones. |
| Los mensajes directos no se procesan | `openclaw pairing list matrix`            | Aprueba el remitente o ajusta la política de mensajes directos.            |
| Las salas cifradas fallan           | `openclaw matrix verify status`           | Vuelve a verificar el dispositivo y luego comprueba `openclaw matrix verify backup status`. |
| La restauración de copia de seguridad está pendiente/rota | `openclaw matrix verify backup status`    | Ejecuta `openclaw matrix verify backup restore` o vuelve a ejecutarlo con una clave de recuperación. |
| El inicio de cross-signing/bootstrap parece incorrecto | `openclaw matrix verify bootstrap`        | Repara el almacenamiento secreto, el cross-signing y el estado de la copia de seguridad en un solo paso. |

Configuración y puesta en marcha completas: [Matrix](/channels/matrix)
