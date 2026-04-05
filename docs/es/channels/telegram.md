---
read_when:
    - Trabajando en funciones de Telegram o webhooks
summary: Estado de compatibilidad, capacidades y configuración del bot de Telegram
title: Telegram
x-i18n:
    generated_at: "2026-04-05T12:38:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 39fbf328375fbc5d08ec2e3eed58b19ee0afa102010ecbc02e074a310ced157e
    source_path: channels/telegram.md
    workflow: 15
---

# Telegram (Bot API)

Estado: listo para producción para MD de bots + grupos mediante grammY. El sondeo largo es el modo predeterminado; el modo webhook es opcional.

<CardGroup cols={3}>
  <Card title="Pairing" icon="link" href="/channels/pairing">
    La política predeterminada de MD para Telegram es pairing.
  </Card>
  <Card title="Channel troubleshooting" icon="wrench" href="/channels/troubleshooting">
    Diagnóstico entre canales y guías de reparación.
  </Card>
  <Card title="Gateway configuration" icon="settings" href="/gateway/configuration">
    Patrones y ejemplos completos de configuración de canales.
  </Card>
</CardGroup>

## Configuración rápida

<Steps>
  <Step title="Create the bot token in BotFather">
    Abre Telegram y chatea con **@BotFather** (confirma que el identificador sea exactamente `@BotFather`).

    Ejecuta `/newbot`, sigue las indicaciones y guarda el token.

  </Step>

  <Step title="Configure token and DM policy">

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

    Variable de entorno de respaldo: `TELEGRAM_BOT_TOKEN=...` (solo cuenta predeterminada).
    Telegram **no** usa `openclaw channels login telegram`; configura el token en config/env y luego inicia el gateway.

  </Step>

  <Step title="Start gateway and approve first DM">

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    Los códigos de pairing caducan después de 1 hora.

  </Step>

  <Step title="Add the bot to a group">
    Añade el bot a tu grupo y luego configura `channels.telegram.groups` y `groupPolicy` para que coincidan con tu modelo de acceso.
  </Step>
</Steps>

<Note>
El orden de resolución del token tiene en cuenta la cuenta. En la práctica, los valores de configuración tienen prioridad sobre la variable de entorno de respaldo, y `TELEGRAM_BOT_TOKEN` solo se aplica a la cuenta predeterminada.
</Note>

## Configuración en el lado de Telegram

<AccordionGroup>
  <Accordion title="Privacy mode and group visibility">
    Los bots de Telegram usan **Privacy Mode** de forma predeterminada, lo que limita qué mensajes de grupo reciben.

    Si el bot debe ver todos los mensajes del grupo, haz una de estas dos cosas:

    - desactiva el modo de privacidad mediante `/setprivacy`, o
    - convierte al bot en administrador del grupo.

    Al cambiar el modo de privacidad, elimina y vuelve a añadir el bot en cada grupo para que Telegram aplique el cambio.

  </Accordion>

  <Accordion title="Group permissions">
    El estado de administrador se controla en la configuración del grupo de Telegram.

    Los bots administradores reciben todos los mensajes del grupo, lo que resulta útil para un comportamiento siempre activo en grupos.

  </Accordion>

  <Accordion title="Helpful BotFather toggles">

    - `/setjoingroups` para permitir/denegar añadidos a grupos
    - `/setprivacy` para el comportamiento de visibilidad en grupos

  </Accordion>
</AccordionGroup>

## Control de acceso y activación

<Tabs>
  <Tab title="DM policy">
    `channels.telegram.dmPolicy` controla el acceso a mensajes directos:

    - `pairing` (predeterminado)
    - `allowlist` (requiere al menos un ID de remitente en `allowFrom`)
    - `open` (requiere que `allowFrom` incluya `"*"`)
    - `disabled`

    `channels.telegram.allowFrom` acepta ID numéricos de usuario de Telegram. Los prefijos `telegram:` / `tg:` se aceptan y se normalizan.
    `dmPolicy: "allowlist"` con `allowFrom` vacío bloquea todos los MD y la validación de configuración lo rechaza.
    El onboarding acepta entrada `@username` y la resuelve a ID numéricos.
    Si actualizaste y tu configuración contiene entradas de lista de permitidos `@username`, ejecuta `openclaw doctor --fix` para resolverlas (mejor esfuerzo; requiere un token de bot de Telegram).
    Si antes dependías de archivos de lista de permitidos del almacén de pairing, `openclaw doctor --fix` puede recuperar entradas en `channels.telegram.allowFrom` en flujos de lista de permitidos (por ejemplo, cuando `dmPolicy: "allowlist"` aún no tiene ID explícitos).

    Para bots de un solo propietario, prefiere `dmPolicy: "allowlist"` con ID numéricos explícitos en `allowFrom` para mantener la política de acceso de forma duradera en la configuración (en lugar de depender de aprobaciones de pairing previas).

    Confusión habitual: aprobar el pairing de MD no significa "este remitente está autorizado en todas partes".
    El pairing concede acceso solo a MD. La autorización del remitente en grupos sigue viniendo de las listas de permitidos explícitas de la configuración.
    Si quieres "estoy autorizado una vez y funcionan tanto los MD como los comandos de grupo", coloca tu ID numérico de usuario de Telegram en `channels.telegram.allowFrom`.

    ### Encontrar tu ID de usuario de Telegram

    Más seguro (sin bot de terceros):

    1. Envía un MD a tu bot.
    2. Ejecuta `openclaw logs --follow`.
    3. Lee `from.id`.

    Método oficial de Bot API:

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    Método de terceros (menos privado): `@userinfobot` o `@getidsbot`.

  </Tab>

  <Tab title="Group policy and allowlists">
    Se aplican juntos dos controles:

    1. **Qué grupos están permitidos** (`channels.telegram.groups`)
       - sin configuración `groups`:
         - con `groupPolicy: "open"`: cualquier grupo puede pasar las comprobaciones de ID de grupo
         - con `groupPolicy: "allowlist"` (predeterminado): los grupos se bloquean hasta que añadas entradas en `groups` (o `"*"`)
       - `groups` configurado: actúa como lista de permitidos (ID explícitos o `"*"`)

    2. **Qué remitentes están permitidos en grupos** (`channels.telegram.groupPolicy`)
       - `open`
       - `allowlist` (predeterminado)
       - `disabled`

    `groupAllowFrom` se usa para filtrar remitentes en grupos. Si no está configurado, Telegram recurre a `allowFrom`.
    Las entradas de `groupAllowFrom` deben ser ID numéricos de usuario de Telegram (los prefijos `telegram:` / `tg:` se normalizan).
    No pongas ID de chat de grupo o supergrupo de Telegram en `groupAllowFrom`. Los ID de chat negativos pertenecen a `channels.telegram.groups`.
    Las entradas no numéricas se ignoran para la autorización del remitente.
    Límite de seguridad (`2026.2.25+`): la autenticación de remitentes en grupos **no** hereda aprobaciones del almacén de pairing de MD.
    El pairing sigue siendo solo para MD. Para grupos, configura `groupAllowFrom` o `allowFrom` por grupo/por tema.
    Si `groupAllowFrom` no está configurado, Telegram recurre a `allowFrom` de la configuración, no al almacén de pairing.
    Patrón práctico para bots de un solo propietario: configura tu ID de usuario en `channels.telegram.allowFrom`, deja `groupAllowFrom` sin definir y permite los grupos objetivo en `channels.telegram.groups`.
    Nota de ejecución: si falta por completo `channels.telegram`, la ejecución usa por defecto el modo cerrado `groupPolicy="allowlist"` a menos que `channels.defaults.groupPolicy` esté configurado explícitamente.

    Ejemplo: permitir a cualquier miembro en un grupo específico:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

    Ejemplo: permitir solo a usuarios específicos dentro de un grupo específico:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

    <Warning>
      Error habitual: `groupAllowFrom` no es una lista de permitidos de grupos de Telegram.

      - Coloca los ID negativos de grupo o supergrupo de Telegram como `-1001234567890` en `channels.telegram.groups`.
      - Coloca los ID de usuario de Telegram como `8734062810` en `groupAllowFrom` cuando quieras limitar qué personas dentro de un grupo permitido pueden activar el bot.
      - Usa `groupAllowFrom: ["*"]` solo cuando quieras que cualquier miembro de un grupo permitido pueda hablar con el bot.
    </Warning>

  </Tab>

  <Tab title="Mention behavior">
    Las respuestas en grupo requieren mención de forma predeterminada.

    La mención puede venir de:

    - una mención nativa `@botusername`, o
    - patrones de mención en:
      - `agents.list[].groupChat.mentionPatterns`
      - `messages.groupChat.mentionPatterns`

    Alternadores de comandos a nivel de sesión:

    - `/activation always`
    - `/activation mention`

    Estos actualizan solo el estado de la sesión. Usa la configuración para persistencia.

    Ejemplo de configuración persistente:

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false },
      },
    },
  },
}
```

    Obtener el ID del chat de grupo:

    - reenvía un mensaje del grupo a `@userinfobot` / `@getidsbot`
    - o lee `chat.id` desde `openclaw logs --follow`
    - o inspecciona `getUpdates` de Bot API

  </Tab>
</Tabs>

## Comportamiento en tiempo de ejecución

- Telegram pertenece al proceso del gateway.
- El enrutamiento es determinista: las respuestas entrantes de Telegram vuelven a Telegram (el modelo no elige canales).
- Los mensajes entrantes se normalizan en la envoltura de canal compartida con metadatos de respuesta y marcadores de posición de medios.
- Las sesiones de grupo están aisladas por ID de grupo. Los temas de foros añaden `:topic:<threadId>` para mantener el aislamiento entre temas.
- Los mensajes de MD pueden llevar `message_thread_id`; OpenClaw los enruta con claves de sesión conscientes del hilo y conserva el ID del hilo para las respuestas.
- El sondeo largo usa grammY runner con secuenciación por chat/por hilo. La concurrencia general del receptor de runner usa `agents.defaults.maxConcurrent`.
- Telegram Bot API no admite confirmaciones de lectura (`sendReadReceipts` no se aplica).

## Referencia de funciones

<AccordionGroup>
  <Accordion title="Live stream preview (message edits)">
    OpenClaw puede transmitir respuestas parciales en tiempo real:

    - chats directos: mensaje de vista previa + `editMessageText`
    - grupos/temas: mensaje de vista previa + `editMessageText`

    Requisito:

    - `channels.telegram.streaming` es `off | partial | block | progress` (predeterminado: `partial`)
    - `progress` se asigna a `partial` en Telegram (compatibilidad con nomenclatura entre canales)
    - los valores heredados `channels.telegram.streamMode` y booleanos `streaming` se asignan automáticamente

    Para respuestas solo de texto:

    - MD: OpenClaw conserva el mismo mensaje de vista previa y realiza una edición final en el mismo lugar (sin segundo mensaje)
    - grupo/tema: OpenClaw conserva el mismo mensaje de vista previa y realiza una edición final en el mismo lugar (sin segundo mensaje)

    Para respuestas complejas (por ejemplo, cargas de medios), OpenClaw recurre a la entrega final normal y después limpia el mensaje de vista previa.

    La transmisión de vista previa es independiente de la transmisión por bloques. Cuando la transmisión por bloques está habilitada explícitamente para Telegram, OpenClaw omite la transmisión de vista previa para evitar una doble transmisión.

    Si el transporte nativo de borrador no está disponible o es rechazado, OpenClaw recurre automáticamente a `sendMessage` + `editMessageText`.

    Transmisión de razonamiento solo para Telegram:

    - `/reasoning stream` envía el razonamiento a la vista previa en vivo mientras se genera
    - la respuesta final se envía sin el texto del razonamiento

  </Accordion>

  <Accordion title="Formatting and HTML fallback">
    El texto saliente usa `parse_mode: "HTML"` de Telegram.

    - El texto tipo Markdown se renderiza como HTML seguro para Telegram.
    - El HTML sin procesar del modelo se escapa para reducir los fallos de análisis de Telegram.
    - Si Telegram rechaza el HTML analizado, OpenClaw reintenta como texto sin formato.

    Las vistas previas de enlaces están habilitadas de forma predeterminada y pueden deshabilitarse con `channels.telegram.linkPreview: false`.

  </Accordion>

  <Accordion title="Native commands and custom commands">
    El registro del menú de comandos de Telegram se gestiona al inicio con `setMyCommands`.

    Valores predeterminados de comandos nativos:

    - `commands.native: "auto"` habilita comandos nativos para Telegram

    Añadir entradas de menú de comandos personalizadas:

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
    },
  },
}
```

    Reglas:

    - los nombres se normalizan (eliminan el `/` inicial, minúsculas)
    - patrón válido: `a-z`, `0-9`, `_`, longitud `1..32`
    - los comandos personalizados no pueden sobrescribir comandos nativos
    - los conflictos/duplicados se omiten y se registran

    Notas:

    - los comandos personalizados son solo entradas de menú; no implementan el comportamiento automáticamente
    - los comandos de plugins/Skills pueden seguir funcionando cuando se escriben incluso si no se muestran en el menú de Telegram

    Si los comandos nativos están deshabilitados, los integrados se eliminan. Los comandos personalizados/de plugins aún pueden registrarse si están configurados.

    Fallos habituales de configuración:

    - `setMyCommands failed` con `BOT_COMMANDS_TOO_MUCH` significa que el menú de Telegram siguió desbordado después de recortarlo; reduce comandos de plugins/Skills/personalizados o deshabilita `channels.telegram.commands.native`.
    - `setMyCommands failed` con errores de red/fetch normalmente significa que el DNS/HTTPS saliente hacia `api.telegram.org` está bloqueado.

    ### Comandos de pairing de dispositivos (plugin `device-pair`)

    Cuando el plugin `device-pair` está instalado:

    1. `/pair` genera el código de configuración
    2. pega el código en la app de iOS
    3. `/pair pending` lista las solicitudes pendientes (incluidos rol/alcances)
    4. aprueba la solicitud:
       - `/pair approve <requestId>` para aprobación explícita
       - `/pair approve` cuando solo hay una solicitud pendiente
       - `/pair approve latest` para la más reciente

    El código de configuración lleva un token de bootstrap de corta duración. La transferencia integrada de bootstrap mantiene el token de nodo principal en `scopes: []`; cualquier token de operador transferido permanece limitado a `operator.approvals`, `operator.read`, `operator.talk.secrets` y `operator.write`. Las comprobaciones de alcance de bootstrap usan prefijos por rol, por lo que esa lista de permitidos de operador solo satisface solicitudes de operador; los roles que no son operador siguen necesitando alcances bajo su propio prefijo de rol.

    Si un dispositivo vuelve a intentarlo con detalles de autenticación modificados (por ejemplo, rol/alcances/clave pública), la solicitud pendiente anterior se reemplaza y la nueva solicitud usa un `requestId` diferente. Vuelve a ejecutar `/pair pending` antes de aprobar.

    Más detalles: [Pairing](/channels/pairing#pair-via-telegram-recommended-for-ios).

  </Accordion>

  <Accordion title="Inline buttons">
    Configura el alcance del teclado en línea:

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

    Anulación por cuenta:

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

    Alcances:

    - `off`
    - `dm`
    - `group`
    - `all`
    - `allowlist` (predeterminado)

    El valor heredado `capabilities: ["inlineButtons"]` se asigna a `inlineButtons: "all"`.

    Ejemplo de acción de mensaje:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "Choose an option:",
  buttons: [
    [
      { text: "Yes", callback_data: "yes" },
      { text: "No", callback_data: "no" },
    ],
    [{ text: "Cancel", callback_data: "cancel" }],
  ],
}
```

    Los clics de callback se pasan al agente como texto:
    `callback_data: <value>`

  </Accordion>

  <Accordion title="Telegram message actions for agents and automation">
    Las acciones de herramientas de Telegram incluyen:

    - `sendMessage` (`to`, `content`, `mediaUrl`, `replyToMessageId`, `messageThreadId` opcionales)
    - `react` (`chatId`, `messageId`, `emoji`)
    - `deleteMessage` (`chatId`, `messageId`)
    - `editMessage` (`chatId`, `messageId`, `content`)
    - `createForumTopic` (`chatId`, `name`, `iconColor`, `iconCustomEmojiId` opcionales)

    Las acciones de mensajes del canal exponen alias ergonómicos (`send`, `react`, `delete`, `edit`, `sticker`, `sticker-search`, `topic-create`).

    Controles de activación:

    - `channels.telegram.actions.sendMessage`
    - `channels.telegram.actions.deleteMessage`
    - `channels.telegram.actions.reactions`
    - `channels.telegram.actions.sticker` (predeterminado: deshabilitado)

    Nota: `edit` y `topic-create` están actualmente habilitados de forma predeterminada y no tienen alternadores separados `channels.telegram.actions.*`.
    Los envíos en tiempo de ejecución usan la instantánea activa de configuración/secretos (inicio/recarga), por lo que las rutas de acción no realizan una nueva resolución ad hoc de SecretRef en cada envío.

    Semántica de eliminación de reacciones: [/tools/reactions](/tools/reactions)

  </Accordion>

  <Accordion title="Reply threading tags">
    Telegram admite etiquetas explícitas de encadenamiento de respuestas en la salida generada:

    - `[[reply_to_current]]` responde al mensaje que activó la acción
    - `[[reply_to:<id>]]` responde a un ID de mensaje específico de Telegram

    `channels.telegram.replyToMode` controla el manejo:

    - `off` (predeterminado)
    - `first`
    - `all`

    Nota: `off` deshabilita el encadenamiento implícito de respuestas. Las etiquetas explícitas `[[reply_to_*]]` se siguen respetando.

  </Accordion>

  <Accordion title="Forum topics and thread behavior">
    Supergrupos de foro:

    - las claves de sesión de temas añaden `:topic:<threadId>`
    - las respuestas y las acciones de escritura se dirigen al hilo del tema
    - ruta de configuración del tema:
      `channels.telegram.groups.<chatId>.topics.<threadId>`

    Caso especial de tema general (`threadId=1`):

    - los envíos de mensajes omiten `message_thread_id` (Telegram rechaza `sendMessage(...thread_id=1)`)
    - las acciones de escritura siguen incluyendo `message_thread_id`

    Herencia de temas: las entradas de temas heredan la configuración del grupo salvo que se sobrescriba (`requireMention`, `allowFrom`, `skills`, `systemPrompt`, `enabled`, `groupPolicy`).
    `agentId` es exclusivo del tema y no se hereda de los valores predeterminados del grupo.

    **Enrutamiento de agentes por tema**: cada tema puede enrutar a un agente distinto configurando `agentId` en la configuración del tema. Esto da a cada tema su propio espacio de trabajo, memoria y sesión aislados. Ejemplo:

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // Tema general → agente principal
                "3": { agentId: "zu" },        // Tema de desarrollo → agente zu
                "5": { agentId: "coder" }      // Revisión de código → agente coder
              }
            }
          }
        }
      }
    }
    ```

    Cada tema tiene entonces su propia clave de sesión: `agent:zu:telegram:group:-1001234567890:topic:3`

    **Enlace persistente de temas ACP**: los temas de foros pueden fijar sesiones de arnés ACP mediante enlaces ACP tipados de nivel superior:

    - `bindings[]` con `type: "acp"` y `match.channel: "telegram"`

    Ejemplo:

    ```json5
    {
      agents: {
        list: [
          {
            id: "codex",
            runtime: {
              type: "acp",
              acp: {
                agent: "codex",
                backend: "acpx",
                mode: "persistent",
                cwd: "/workspace/openclaw",
              },
            },
          },
        ],
      },
      bindings: [
        {
          type: "acp",
          agentId: "codex",
          match: {
            channel: "telegram",
            accountId: "default",
            peer: { kind: "group", id: "-1001234567890:topic:42" },
          },
        },
      ],
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "42": {
                  requireMention: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    Actualmente esto se limita a temas de foros en grupos y supergrupos.

    **Creación de ACP vinculado a hilos desde el chat**:

    - `/acp spawn <agent> --thread here|auto` puede enlazar el tema actual de Telegram a una nueva sesión ACP.
    - Los mensajes posteriores del tema se enrutan directamente a la sesión ACP enlazada (no se requiere `/acp steer`).
    - OpenClaw fija el mensaje de confirmación de creación dentro del tema después de un enlace correcto.
    - Requiere `channels.telegram.threadBindings.spawnAcpSessions=true`.

    El contexto de la plantilla incluye:

    - `MessageThreadId`
    - `IsForum`

    Comportamiento de hilos en MD:

    - los chats privados con `message_thread_id` mantienen el enrutamiento de MD pero usan claves de sesión y destinos de respuesta conscientes del hilo.

  </Accordion>

  <Accordion title="Audio, video, and stickers">
    ### Mensajes de audio

    Telegram distingue entre notas de voz y archivos de audio.

    - predeterminado: comportamiento de archivo de audio
    - etiqueta `[[audio_as_voice]]` en la respuesta del agente para forzar el envío como nota de voz

    Ejemplo de acción de mensaje:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

    ### Mensajes de vídeo

    Telegram distingue entre archivos de vídeo y notas de vídeo.

    Ejemplo de acción de mensaje:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    Las notas de vídeo no admiten subtítulos; el texto del mensaje proporcionado se envía por separado.

    ### Stickers

    Manejo de stickers entrantes:

    - WEBP estático: se descarga y procesa (marcador `<media:sticker>`)
    - TGS animado: se omite
    - WEBM de vídeo: se omite

    Campos de contexto de stickers:

    - `Sticker.emoji`
    - `Sticker.setName`
    - `Sticker.fileId`
    - `Sticker.fileUniqueId`
    - `Sticker.cachedDescription`

    Archivo de caché de stickers:

    - `~/.openclaw/telegram/sticker-cache.json`

    Los stickers se describen una vez (cuando es posible) y se almacenan en caché para reducir llamadas repetidas de visión.

    Habilitar acciones de stickers:

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

    Acción para enviar sticker:

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    Buscar stickers en caché:

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "cat waving",
  limit: 5,
}
```

  </Accordion>

  <Accordion title="Reaction notifications">
    Las reacciones de Telegram llegan como actualizaciones `message_reaction` (separadas de las cargas de mensajes).

    Cuando están habilitadas, OpenClaw pone en cola eventos del sistema como:

    - `Telegram reaction added: 👍 by Alice (@alice) on msg 42`

    Configuración:

    - `channels.telegram.reactionNotifications`: `off | own | all` (predeterminado: `own`)
    - `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` (predeterminado: `minimal`)

    Notas:

    - `own` significa solo reacciones de usuarios a mensajes enviados por el bot (mejor esfuerzo mediante caché de mensajes enviados).
    - Los eventos de reacción siguen respetando los controles de acceso de Telegram (`dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`); los remitentes no autorizados se descartan.
    - Telegram no proporciona ID de hilo en las actualizaciones de reacción.
      - los grupos que no son foros se enrutan a la sesión de chat grupal
      - los grupos de foro se enrutan a la sesión del tema general del grupo (`:topic:1`), no al tema exacto de origen

    `allowed_updates` para sondeo/webhook incluyen `message_reaction` automáticamente.

  </Accordion>

  <Accordion title="Ack reactions">
    `ackReaction` envía un emoji de confirmación mientras OpenClaw procesa un mensaje entrante.

    Orden de resolución:

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - emoji de respaldo de identidad del agente (`agents.list[].identity.emoji`, o bien "👀")

    Notas:

    - Telegram espera emoji Unicode (por ejemplo, "👀").
    - Usa `""` para deshabilitar la reacción para un canal o cuenta.

  </Accordion>

  <Accordion title="Config writes from Telegram events and commands">
    Las escrituras de configuración del canal están habilitadas de forma predeterminada (`configWrites !== false`).

    Las escrituras activadas por Telegram incluyen:

    - eventos de migración de grupos (`migrate_to_chat_id`) para actualizar `channels.telegram.groups`
    - `/config set` y `/config unset` (requiere habilitación de comandos)

    Deshabilitar:

```json5
{
  channels: {
    telegram: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Long polling vs webhook">
    Predeterminado: sondeo largo.

    Modo webhook:

    - configura `channels.telegram.webhookUrl`
    - configura `channels.telegram.webhookSecret` (obligatorio cuando `webhookUrl` está configurado)
    - `channels.telegram.webhookPath` opcional (predeterminado `/telegram-webhook`)
    - `channels.telegram.webhookHost` opcional (predeterminado `127.0.0.1`)
    - `channels.telegram.webhookPort` opcional (predeterminado `8787`)

    El escuchador local predeterminado para el modo webhook se enlaza a `127.0.0.1:8787`.

    Si tu endpoint público es distinto, coloca un proxy inverso delante y apunta `webhookUrl` a la URL pública.
    Configura `webhookHost` (por ejemplo `0.0.0.0`) cuando necesites intencionalmente ingreso externo.

  </Accordion>

  <Accordion title="Limits, retry, and CLI targets">
    - `channels.telegram.textChunkLimit` tiene como valor predeterminado 4000.
    - `channels.telegram.chunkMode="newline"` prefiere límites de párrafo (líneas en blanco) antes de dividir por longitud.
    - `channels.telegram.mediaMaxMb` (predeterminado 100) limita el tamaño de los medios entrantes y salientes de Telegram.
    - `channels.telegram.timeoutSeconds` sobrescribe el tiempo de espera del cliente de la API de Telegram (si no se configura, se aplica el valor predeterminado de grammY).
    - el historial de contexto de grupo usa `channels.telegram.historyLimit` o `messages.groupChat.historyLimit` (predeterminado 50); `0` lo deshabilita.
    - el contexto suplementario de respuesta/cita/reenvío se pasa actualmente tal como se recibe.
    - las listas de permitidos de Telegram controlan principalmente quién puede activar el agente, no son un límite completo de redacción del contexto suplementario.
    - controles del historial de MD:
      - `channels.telegram.dmHistoryLimit`
      - `channels.telegram.dms["<user_id>"].historyLimit`
    - la configuración `channels.telegram.retry` se aplica a los ayudantes de envío de Telegram (CLI/herramientas/acciones) para errores recuperables de API saliente.

    El destino de envío de CLI puede ser un ID numérico de chat o un nombre de usuario:

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
```

    Los sondeos de Telegram usan `openclaw message poll` y admiten temas de foros:

```bash
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300 --poll-public
```

    Indicadores de sondeo solo de Telegram:

    - `--poll-duration-seconds` (5-600)
    - `--poll-anonymous`
    - `--poll-public`
    - `--thread-id` para temas de foros (o usa un destino `:topic:`)

    El envío de Telegram también admite:

    - `--buttons` para teclados en línea cuando `channels.telegram.capabilities.inlineButtons` lo permite
    - `--force-document` para enviar imágenes y GIF salientes como documentos en lugar de cargas comprimidas de foto o medios animados

    Control de acciones:

    - `channels.telegram.actions.sendMessage=false` deshabilita los mensajes salientes de Telegram, incluidos los sondeos
    - `channels.telegram.actions.poll=false` deshabilita la creación de sondeos de Telegram y mantiene habilitados los envíos normales

  </Accordion>

  <Accordion title="Exec approvals in Telegram">
    Telegram admite aprobaciones de ejecución en MD de aprobadores y opcionalmente puede publicar solicitudes de aprobación en el chat o tema de origen.

    Ruta de configuración:

    - `channels.telegram.execApprovals.enabled`
    - `channels.telegram.execApprovals.approvers` (opcional; recurre a los ID numéricos de propietario inferidos de `allowFrom` y `defaultTo` directo cuando es posible)
    - `channels.telegram.execApprovals.target` (`dm` | `channel` | `both`, predeterminado: `dm`)
    - `agentFilter`, `sessionFilter`

    Los aprobadores deben ser ID numéricos de usuario de Telegram. Telegram habilita automáticamente aprobaciones nativas de ejecución cuando `enabled` no está configurado o es `"auto"` y se puede resolver al menos un aprobador, ya sea desde `execApprovals.approvers` o desde la configuración numérica del propietario de la cuenta (`allowFrom` y `defaultTo` de mensaje directo). Configura `enabled: false` para deshabilitar explícitamente Telegram como cliente nativo de aprobación. De lo contrario, las solicitudes de aprobación recurren a otras rutas de aprobación configuradas o a la política de respaldo de aprobación de ejecución.

    Telegram también renderiza los botones de aprobación compartidos usados por otros canales de chat. El adaptador nativo de Telegram añade principalmente el enrutamiento de MD para aprobadores, la difusión a canal/tema y las pistas de escritura antes de la entrega.
    Cuando esos botones están presentes, son la UX principal de aprobación; OpenClaw
    solo debería incluir un comando manual `/approve` cuando el resultado de la herramienta diga
    que las aprobaciones por chat no están disponibles o que la aprobación manual es la única vía.

    Reglas de entrega:

    - `target: "dm"` envía solicitudes de aprobación solo a los MD de aprobadores resueltos
    - `target: "channel"` envía la solicitud de vuelta al chat/tema de Telegram de origen
    - `target: "both"` envía a los MD de aprobadores y al chat/tema de origen

    Solo los aprobadores resueltos pueden aprobar o denegar. Los no aprobadores no pueden usar `/approve` ni los botones de aprobación de Telegram.

    Comportamiento de resolución de aprobación:

    - Los ID con prefijo `plugin:` siempre se resuelven mediante aprobaciones del plugin.
    - Otros ID prueban primero `exec.approval.resolve`.
    - Si Telegram también está autorizado para aprobaciones del plugin y el gateway dice
      que la aprobación de ejecución es desconocida o caducó, Telegram reintenta una vez mediante
      `plugin.approval.resolve`.
    - Las denegaciones/errores reales de aprobación de ejecución no pasan silenciosamente a la
      resolución de aprobación del plugin.

    La entrega por canal muestra el texto del comando en el chat, así que habilita `channel` o `both` solo en grupos/temas de confianza. Cuando la solicitud llega a un tema de foro, OpenClaw conserva el tema tanto para la solicitud de aprobación como para el seguimiento posterior a la aprobación. Las aprobaciones de ejecución caducan después de 30 minutos de forma predeterminada.

    Los botones de aprobación en línea también dependen de que `channels.telegram.capabilities.inlineButtons` permita la superficie objetivo (`dm`, `group` o `all`).

    Documentación relacionada: [Aprobaciones de ejecución](/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## Controles de respuesta de error

Cuando el agente encuentra un error de entrega o de proveedor, Telegram puede responder con el texto del error o suprimirlo. Dos claves de configuración controlan este comportamiento:

| Key                                 | Values            | Default | Description                                                                                     |
| ----------------------------------- | ----------------- | ------- | ----------------------------------------------------------------------------------------------- |
| `channels.telegram.errorPolicy`     | `reply`, `silent` | `reply` | `reply` envía un mensaje de error amigable al chat. `silent` suprime por completo las respuestas de error. |
| `channels.telegram.errorCooldownMs` | number (ms)       | `60000` | Tiempo mínimo entre respuestas de error al mismo chat. Evita spam de errores durante caídas.        |

Se admiten anulaciones por cuenta, por grupo y por tema (la misma herencia que otras claves de configuración de Telegram).

```json5
{
  channels: {
    telegram: {
      errorPolicy: "reply",
      errorCooldownMs: 120000,
      groups: {
        "-1001234567890": {
          errorPolicy: "silent", // suprime errores en este grupo
        },
      },
    },
  },
}
```

## Solución de problemas

<AccordionGroup>
  <Accordion title="Bot does not respond to non mention group messages">

    - Si `requireMention=false`, el modo de privacidad de Telegram debe permitir visibilidad completa.
      - BotFather: `/setprivacy` -> Disable
      - luego elimina y vuelve a añadir el bot al grupo
    - `openclaw channels status` muestra una advertencia cuando la configuración espera mensajes de grupo sin mención.
    - `openclaw channels status --probe` puede comprobar ID numéricos explícitos de grupo; el comodín `"*"` no puede sondearse para membresía.
    - prueba rápida de sesión: `/activation always`.

  </Accordion>

  <Accordion title="Bot not seeing group messages at all">

    - cuando existe `channels.telegram.groups`, el grupo debe estar listado (o incluir `"*"`)
    - verifica la membresía del bot en el grupo
    - revisa los registros: `openclaw logs --follow` para ver motivos de omisión

  </Accordion>

  <Accordion title="Commands work partially or not at all">

    - autoriza la identidad de tu remitente (pairing y/o `allowFrom` numérico)
    - la autorización de comandos sigue aplicándose incluso cuando la política de grupo es `open`
    - `setMyCommands failed` con `BOT_COMMANDS_TOO_MUCH` significa que el menú nativo tiene demasiadas entradas; reduce comandos de plugins/Skills/personalizados o deshabilita los menús nativos
    - `setMyCommands failed` con errores de red/fetch normalmente indica problemas de alcance DNS/HTTPS hacia `api.telegram.org`

  </Accordion>

  <Accordion title="Polling or network instability">

    - Node 22+ + fetch/proxy personalizado pueden activar un comportamiento de cancelación inmediata si los tipos de AbortSignal no coinciden.
    - Algunos hosts resuelven `api.telegram.org` primero a IPv6; una salida IPv6 defectuosa puede causar fallos intermitentes de la API de Telegram.
    - Si los registros incluyen `TypeError: fetch failed` o `Network request for 'getUpdates' failed!`, OpenClaw ahora los reintenta como errores de red recuperables.
    - En hosts VPS con salida/TLS directa inestable, enruta las llamadas a la API de Telegram mediante `channels.telegram.proxy`:

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - Node 22+ usa de forma predeterminada `autoSelectFamily=true` (excepto WSL2) y `dnsResultOrder=ipv4first`.
    - Si tu host es WSL2 o explícitamente funciona mejor con comportamiento solo IPv4, fuerza la selección de familia:

```yaml
channels:
  telegram:
    network:
      autoSelectFamily: false
```

    - Las respuestas del rango de prueba RFC 2544 (`198.18.0.0/15`) ya están permitidas
      para descargas de medios de Telegram de forma predeterminada. Si una IP falsa de confianza o un
      proxy transparente reescribe `api.telegram.org` a alguna otra
      dirección privada/interna/de uso especial durante las descargas de medios, puedes
      habilitar la omisión solo para Telegram:

```yaml
channels:
  telegram:
    network:
      dangerouslyAllowPrivateNetwork: true
```

    - La misma opción también está disponible por cuenta en
      `channels.telegram.accounts.<accountId>.network.dangerouslyAllowPrivateNetwork`.
    - Si tu proxy resuelve hosts de medios de Telegram en `198.18.x.x`, deja primero desactivada la
      marca peligrosa. Los medios de Telegram ya permiten por defecto el rango de referencia RFC 2544.

    <Warning>
      `channels.telegram.network.dangerouslyAllowPrivateNetwork` debilita las protecciones
      SSRF de medios de Telegram. Úsalo solo para entornos de proxy de confianza controlados por el operador
      como el enrutamiento con IP falsa de Clash, Mihomo o Surge cuando sintetizan
      respuestas privadas o de uso especial fuera del rango de referencia RFC 2544.
      Déjalo desactivado para acceso normal de Telegram a través de internet pública.
    </Warning>

    - Anulaciones por entorno (temporales):
      - `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`
      - `OPENCLAW_TELEGRAM_ENABLE_AUTO_SELECT_FAMILY=1`
      - `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first`
    - Validar respuestas DNS:

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

Más ayuda: [Solución de problemas de canales](/channels/troubleshooting).

## Punteros de referencia de configuración de Telegram

Referencia principal:

- `channels.telegram.enabled`: habilitar/deshabilitar el inicio del canal.
- `channels.telegram.botToken`: token del bot (BotFather).
- `channels.telegram.tokenFile`: leer el token desde una ruta de archivo normal. Los enlaces simbólicos se rechazan.
- `channels.telegram.dmPolicy`: `pairing | allowlist | open | disabled` (predeterminado: pairing).
- `channels.telegram.allowFrom`: lista de permitidos de MD (ID numéricos de usuario de Telegram). `allowlist` requiere al menos un ID de remitente. `open` requiere `"*"`. `openclaw doctor --fix` puede resolver entradas heredadas `@username` a ID y recuperar entradas de lista de permitidos desde archivos del almacén de pairing en flujos de migración de lista de permitidos.
- `channels.telegram.actions.poll`: habilitar o deshabilitar la creación de sondeos de Telegram (predeterminado: habilitado; sigue requiriendo `sendMessage`).
- `channels.telegram.defaultTo`: destino predeterminado de Telegram usado por CLI `--deliver` cuando no se proporciona `--reply-to` explícito.
- `channels.telegram.groupPolicy`: `open | allowlist | disabled` (predeterminado: allowlist).
- `channels.telegram.groupAllowFrom`: lista de permitidos de remitentes de grupo (ID numéricos de usuario de Telegram). `openclaw doctor --fix` puede resolver entradas heredadas `@username` a ID. Las entradas no numéricas se ignoran en el momento de autenticación. La autenticación de grupos no usa el respaldo del almacén de pairing de MD (`2026.2.25+`).
- Precedencia de múltiples cuentas:
  - Cuando se configuran dos o más ID de cuenta, configura `channels.telegram.defaultAccount` (o incluye `channels.telegram.accounts.default`) para hacer explícito el enrutamiento predeterminado.
  - Si no se configura ninguno, OpenClaw recurre al primer ID de cuenta normalizado y `openclaw doctor` muestra una advertencia.
  - `channels.telegram.accounts.default.allowFrom` y `channels.telegram.accounts.default.groupAllowFrom` se aplican solo a la cuenta `default`.
  - Las cuentas con nombre heredan `channels.telegram.allowFrom` y `channels.telegram.groupAllowFrom` cuando los valores a nivel de cuenta no están configurados.
  - Las cuentas con nombre no heredan `channels.telegram.accounts.default.allowFrom` / `groupAllowFrom`.
- `channels.telegram.groups`: valores predeterminados por grupo + lista de permitidos (usa `"*"` para valores predeterminados globales).
  - `channels.telegram.groups.<id>.groupPolicy`: anulación por grupo para `groupPolicy` (`open | allowlist | disabled`).
  - `channels.telegram.groups.<id>.requireMention`: control predeterminado por mención.
  - `channels.telegram.groups.<id>.skills`: filtro de Skills (omitir = todas las Skills, vacío = ninguna).
  - `channels.telegram.groups.<id>.allowFrom`: anulación por grupo de la lista de permitidos de remitentes.
  - `channels.telegram.groups.<id>.systemPrompt`: prompt del sistema adicional para el grupo.
  - `channels.telegram.groups.<id>.enabled`: deshabilita el grupo cuando es `false`.
  - `channels.telegram.groups.<id>.topics.<threadId>.*`: anulaciones por tema (campos de grupo + `agentId` exclusivo del tema).
  - `channels.telegram.groups.<id>.topics.<threadId>.agentId`: enrutar este tema a un agente específico (sobrescribe el enrutamiento a nivel de grupo y de binding).
- `channels.telegram.groups.<id>.topics.<threadId>.groupPolicy`: anulación por tema para `groupPolicy` (`open | allowlist | disabled`).
- `channels.telegram.groups.<id>.topics.<threadId>.requireMention`: anulación por tema del control por mención.
- `bindings[]` de nivel superior con `type: "acp"` e ID canónico de tema `chatId:topic:topicId` en `match.peer.id`: campos de enlace persistente de temas ACP (consulta [Agentes ACP](/tools/acp-agents#channel-specific-settings)).
- `channels.telegram.direct.<id>.topics.<threadId>.agentId`: enrutar temas de MD a un agente específico (el mismo comportamiento que los temas de foros).
- `channels.telegram.execApprovals.enabled`: habilitar Telegram como cliente de aprobación de ejecución basado en chat para esta cuenta.
- `channels.telegram.execApprovals.approvers`: ID de usuario de Telegram autorizados para aprobar o denegar solicitudes de ejecución. Opcional cuando `channels.telegram.allowFrom` o un `channels.telegram.defaultTo` directo ya identifica al propietario.
- `channels.telegram.execApprovals.target`: `dm | channel | both` (predeterminado: `dm`). `channel` y `both` conservan el tema de Telegram de origen cuando está presente.
- `channels.telegram.execApprovals.agentFilter`: filtro opcional por ID de agente para solicitudes de aprobación reenviadas.
- `channels.telegram.execApprovals.sessionFilter`: filtro opcional por clave de sesión (subcadena o regex) para solicitudes de aprobación reenviadas.
- `channels.telegram.accounts.<account>.execApprovals`: anulación por cuenta para el enrutamiento de aprobaciones de ejecución de Telegram y la autorización de aprobadores.
- `channels.telegram.capabilities.inlineButtons`: `off | dm | group | all | allowlist` (predeterminado: allowlist).
- `channels.telegram.accounts.<account>.capabilities.inlineButtons`: anulación por cuenta.
- `channels.telegram.commands.nativeSkills`: habilitar/deshabilitar comandos nativos de Skills de Telegram.
- `channels.telegram.replyToMode`: `off | first | all` (predeterminado: `off`).
- `channels.telegram.textChunkLimit`: tamaño de fragmento saliente (caracteres).
- `channels.telegram.chunkMode`: `length` (predeterminado) o `newline` para dividir por líneas en blanco (límites de párrafo) antes de fragmentar por longitud.
- `channels.telegram.linkPreview`: alternar vistas previas de enlaces para mensajes salientes (predeterminado: true).
- `channels.telegram.streaming`: `off | partial | block | progress` (vista previa de transmisión en vivo; predeterminado: `partial`; `progress` se asigna a `partial`; `block` es compatibilidad heredada del modo de vista previa). La transmisión de vista previa de Telegram usa un único mensaje de vista previa que se edita en el mismo lugar.
- `channels.telegram.mediaMaxMb`: límite de medios entrantes/salientes de Telegram (MB, predeterminado: 100).
- `channels.telegram.retry`: política de reintento para los ayudantes de envío de Telegram (CLI/herramientas/acciones) en errores recuperables de API saliente (intentos, `minDelayMs`, `maxDelayMs`, `jitter`).
- `channels.telegram.network.autoSelectFamily`: sobrescribir `autoSelectFamily` de Node (true=habilitar, false=deshabilitar). Predeterminado habilitado en Node 22+, con WSL2 deshabilitado de forma predeterminada.
- `channels.telegram.network.dnsResultOrder`: sobrescribir el orden de resultados DNS (`ipv4first` o `verbatim`). Predeterminado `ipv4first` en Node 22+.
- `channels.telegram.network.dangerouslyAllowPrivateNetwork`: opción peligrosa para entornos de IP falsa o proxy transparente de confianza donde las descargas de medios de Telegram resuelven `api.telegram.org` a direcciones privadas/internas/de uso especial fuera de la concesión predeterminada del rango de referencia RFC 2544.
- `channels.telegram.proxy`: URL de proxy para llamadas a Bot API (SOCKS/HTTP).
- `channels.telegram.webhookUrl`: habilitar modo webhook (requiere `channels.telegram.webhookSecret`).
- `channels.telegram.webhookSecret`: secreto de webhook (obligatorio cuando `webhookUrl` está configurado).
- `channels.telegram.webhookPath`: ruta local del webhook (predeterminado `/telegram-webhook`).
- `channels.telegram.webhookHost`: host local de enlace del webhook (predeterminado `127.0.0.1`).
- `channels.telegram.webhookPort`: puerto local de enlace del webhook (predeterminado `8787`).
- `channels.telegram.actions.reactions`: controla las reacciones de herramientas de Telegram.
- `channels.telegram.actions.sendMessage`: controla los envíos de mensajes de herramientas de Telegram.
- `channels.telegram.actions.deleteMessage`: controla la eliminación de mensajes de herramientas de Telegram.
- `channels.telegram.actions.sticker`: controla acciones de stickers de Telegram — envío y búsqueda (predeterminado: false).
- `channels.telegram.reactionNotifications`: `off | own | all` — controla qué reacciones activan eventos del sistema (predeterminado: `own` cuando no está configurado).
- `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` — controla la capacidad de reacción del agente (predeterminado: `minimal` cuando no está configurado).
- `channels.telegram.errorPolicy`: `reply | silent` — controla el comportamiento de respuesta de error (predeterminado: `reply`). Se admiten anulaciones por cuenta/grupo/tema.
- `channels.telegram.errorCooldownMs`: ms mínimos entre respuestas de error al mismo chat (predeterminado: `60000`). Evita spam de errores durante caídas.

- [Referencia de configuración - Telegram](/gateway/configuration-reference#telegram)

Campos específicos de Telegram de alta señal:

- inicio/autenticación: `enabled`, `botToken`, `tokenFile`, `accounts.*` (`tokenFile` debe apuntar a un archivo normal; los enlaces simbólicos se rechazan)
- control de acceso: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`, `groups.*.topics.*`, `bindings[]` de nivel superior (`type: "acp"`)
- aprobaciones de ejecución: `execApprovals`, `accounts.*.execApprovals`
- comando/menú: `commands.native`, `commands.nativeSkills`, `customCommands`
- encadenamiento/respuestas: `replyToMode`
- streaming: `streaming` (vista previa), `blockStreaming`
- formato/entrega: `textChunkLimit`, `chunkMode`, `linkPreview`, `responsePrefix`
- medios/red: `mediaMaxMb`, `timeoutSeconds`, `retry`, `network.autoSelectFamily`, `network.dangerouslyAllowPrivateNetwork`, `proxy`
- webhook: `webhookUrl`, `webhookSecret`, `webhookPath`, `webhookHost`
- acciones/capacidades: `capabilities.inlineButtons`, `actions.sendMessage|editMessage|deleteMessage|reactions|sticker`
- reacciones: `reactionNotifications`, `reactionLevel`
- errores: `errorPolicy`, `errorCooldownMs`
- escrituras/historial: `configWrites`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`

## Relacionado

- [Pairing](/channels/pairing)
- [Grupos](/channels/groups)
- [Seguridad](/gateway/security)
- [Enrutamiento de canales](/channels/channel-routing)
- [Enrutamiento multiagente](/concepts/multi-agent)
- [Solución de problemas](/channels/troubleshooting)
