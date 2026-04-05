---
read_when:
    - Quieres conectar OpenClaw a QQ
    - Necesitas configurar las credenciales de QQ Bot
    - Quieres compatibilidad de QQ Bot para grupos o chats privados
summary: Configuración, ajustes y uso de QQ Bot
title: QQ Bot
x-i18n:
    generated_at: "2026-04-05T12:35:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0e58fb7b07c59ecbf80a1276368c4a007b45d84e296ed40cffe9845e0953696c
    source_path: channels/qqbot.md
    workflow: 15
---

# QQ Bot

QQ Bot se conecta a OpenClaw mediante la API oficial de QQ Bot (gateway WebSocket). El
plugin admite chat privado C2C, mensajes con @ en grupos y mensajes de canales de guild con
contenido multimedia enriquecido (imágenes, voz, video, archivos).

Estado: plugin incluido. Se admiten mensajes directos, chats de grupo, canales de guild y
contenido multimedia. No se admiten reacciones ni hilos.

## Plugin incluido

Las versiones actuales de OpenClaw incluyen QQ Bot, por lo que las compilaciones empaquetadas normales no necesitan
un paso independiente de `openclaw plugins install`.

## Configuración

1. Ve a la [QQ Open Platform](https://q.qq.com/) y escanea el código QR con tu
   QQ del teléfono para registrarte o iniciar sesión.
2. Haz clic en **Create Bot** para crear un nuevo bot de QQ.
3. Busca **AppID** y **AppSecret** en la página de configuración del bot y cópialos.

> AppSecret no se almacena en texto sin formato; si abandonas la página sin guardarlo,
> tendrás que generar uno nuevo.

4. Agrega el canal:

```bash
openclaw channels add --channel qqbot --token "AppID:AppSecret"
```

5. Reinicia el Gateway.

Rutas de configuración interactiva:

```bash
openclaw channels add
openclaw configure --section channels
```

## Configurar

Configuración mínima:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: "YOUR_APP_SECRET",
    },
  },
}
```

Variables de entorno de la cuenta predeterminada:

- `QQBOT_APP_ID`
- `QQBOT_CLIENT_SECRET`

AppSecret respaldado por archivo:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecretFile: "/path/to/qqbot-secret.txt",
    },
  },
}
```

Notas:

- El respaldo por variables de entorno se aplica solo a la cuenta predeterminada de QQ Bot.
- `openclaw channels add --channel qqbot --token-file ...` proporciona solo el
  AppSecret; el AppID ya debe estar configurado en la configuración o en `QQBOT_APP_ID`.
- `clientSecret` también acepta entrada SecretRef, no solo una cadena en texto sin formato.

### Configuración de varias cuentas

Ejecuta varios bots de QQ en una sola instancia de OpenClaw:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "111111111",
      clientSecret: "secret-of-bot-1",
      accounts: {
        bot2: {
          enabled: true,
          appId: "222222222",
          clientSecret: "secret-of-bot-2",
        },
      },
    },
  },
}
```

Cada cuenta inicia su propia conexión WebSocket y mantiene una caché de tokens independiente
(aislada por `appId`).

Agrega un segundo bot mediante la CLI:

```bash
openclaw channels add --channel qqbot --account bot2 --token "222222222:secret-of-bot-2"
```

### Voz (STT / TTS)

La compatibilidad con STT y TTS usa una configuración de dos niveles con prioridad por respaldo:

| Configuración | Específica del plugin | Respaldo del framework         |
| ------------- | --------------------- | ------------------------------ |
| STT           | `channels.qqbot.stt`  | `tools.media.audio.models[0]`  |
| TTS           | `channels.qqbot.tts`  | `messages.tts`                 |

```json5
{
  channels: {
    qqbot: {
      stt: {
        provider: "your-provider",
        model: "your-stt-model",
      },
      tts: {
        provider: "your-provider",
        model: "your-tts-model",
        voice: "your-voice",
      },
    },
  },
}
```

Establece `enabled: false` en cualquiera de ellos para deshabilitarlo.

El comportamiento de carga y transcodificación de audio saliente también puede ajustarse con
`channels.qqbot.audioFormatPolicy`:

- `sttDirectFormats`
- `uploadDirectFormats`
- `transcodeEnabled`

## Formatos de destino

| Formato                    | Descripción         |
| -------------------------- | ------------------- |
| `qqbot:c2c:OPENID`         | Chat privado (C2C)  |
| `qqbot:group:GROUP_OPENID` | Chat de grupo       |
| `qqbot:channel:CHANNEL_ID` | Canal de guild      |

> Cada bot tiene su propio conjunto de OpenID de usuario. Un OpenID recibido por el bot A **no puede**
> usarse para enviar mensajes mediante el bot B.

## Comandos slash

Comandos integrados interceptados antes de la cola de la IA:

| Comando        | Descripción                               |
| -------------- | ----------------------------------------- |
| `/bot-ping`    | Prueba de latencia                        |
| `/bot-version` | Muestra la versión del framework OpenClaw |
| `/bot-help`    | Lista todos los comandos                  |
| `/bot-upgrade` | Muestra el enlace de la guía de actualización de QQBot |
| `/bot-logs`    | Exporta los registros recientes del gateway como archivo |

Agrega `?` a cualquier comando para ver ayuda de uso (por ejemplo `/bot-upgrade ?`).

## Solución de problemas

- **El bot responde "gone to Mars":** las credenciales no están configuradas o Gateway no se ha iniciado.
- **No hay mensajes entrantes:** verifica que `appId` y `clientSecret` sean correctos, y que el
  bot esté habilitado en la QQ Open Platform.
- **La configuración con `--token-file` sigue mostrando que no está configurado:** `--token-file` solo establece
  el AppSecret. Aún necesitas `appId` en la configuración o `QQBOT_APP_ID`.
- **Los mensajes proactivos no llegan:** QQ puede interceptar los mensajes iniciados por el bot si
  el usuario no ha interactuado recientemente.
- **La voz no se transcribe:** asegúrate de que STT esté configurado y de que el proveedor sea accesible.
