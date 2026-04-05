---
read_when:
    - Necesitas la semántica exacta o los valores predeterminados de la configuración a nivel de campo
    - Estás validando bloques de configuración de canal, modelo, gateway o herramientas
summary: Referencia completa de todas las claves de configuración de OpenClaw, valores predeterminados y ajustes de canal
title: Referencia de configuración
x-i18n:
    generated_at: "2026-04-05T12:49:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: bb4c6de7955aa0c6afa2d20f12a0e3782b16ab2c1b6bf3ed0a8910be2f0a47d1
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# Referencia de configuración

Todos los campos disponibles en `~/.openclaw/openclaw.json`. Para una visión general orientada a tareas, consulta [Configuración](/gateway/configuration).

El formato de configuración es **JSON5** (se permiten comentarios y comas finales). Todos los campos son opcionales: OpenClaw usa valores predeterminados seguros cuando se omiten.

---

## Canales

Cada canal se inicia automáticamente cuando existe su sección de configuración (a menos que `enabled: false`).

### Acceso a MD y grupos

Todos los canales admiten políticas de MD y políticas de grupo:

| Política de MD      | Comportamiento                                                   |
| ------------------- | --------------------------------------------------------------- |
| `pairing` (predeterminada) | Los remitentes desconocidos reciben un código de pairing de un solo uso; el propietario debe aprobarlo |
| `allowlist`         | Solo remitentes en `allowFrom` (o en el almacén de permitidos de pairing) |
| `open`              | Permitir todos los MD entrantes (requiere `allowFrom: ["*"]`) |
| `disabled`          | Ignorar todos los MD entrantes                                          |

| Política de grupo     | Comportamiento                                               |
| --------------------- | ------------------------------------------------------------ |
| `allowlist` (predeterminada) | Solo grupos que coinciden con la lista de permitidos configurada |
| `open`                | Omite las listas de permitidos de grupos (el control por mención sigue aplicándose) |
| `disabled`            | Bloquea todos los mensajes de grupo/sala                          |

<Note>
`channels.defaults.groupPolicy` establece el valor predeterminado cuando `groupPolicy` de un proveedor no está configurado.
Los códigos de pairing caducan después de 1 hora. Las solicitudes pendientes de pairing de MD están limitadas a **3 por canal**.
Si falta por completo un bloque de proveedor (`channels.<provider>` ausente), la política de grupo en tiempo de ejecución usa como alternativa `allowlist` (cerrado por defecto) con una advertencia al inicio.
</Note>

### Anulaciones de modelo por canal

Usa `channels.modelByChannel` para fijar ID de canal específicos a un modelo. Los valores aceptan `provider/model` o alias de modelo configurados. La asignación de canal se aplica cuando una sesión todavía no tiene una anulación de modelo (por ejemplo, establecida mediante `/model`).

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-4.1",
      },
      telegram: {
        "-1001234567890": "openai/gpt-4.1-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

### Valores predeterminados de canal y heartbeat

Usa `channels.defaults` para compartir el comportamiento de política de grupo y heartbeat entre proveedores:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`: política de grupo de respaldo cuando `groupPolicy` a nivel de proveedor no está configurada.
- `channels.defaults.contextVisibility`: modo predeterminado de visibilidad de contexto suplementario para todos los canales. Valores: `all` (predeterminado, incluye todo el contexto citado/de hilo/historial), `allowlist` (solo incluye contexto de remitentes permitidos), `allowlist_quote` (igual que allowlist, pero conserva el contexto explícito de cita/respuesta). Anulación por canal: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: incluir estados correctos de canales en la salida de heartbeat.
- `channels.defaults.heartbeat.showAlerts`: incluir estados degradados/con errores en la salida de heartbeat.
- `channels.defaults.heartbeat.useIndicator`: renderizar la salida de heartbeat en formato compacto con indicadores.

### WhatsApp

WhatsApp se ejecuta mediante el canal web del gateway (Baileys Web). Se inicia automáticamente cuando existe una sesión enlazada.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // doble check azul (false en modo self-chat)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
  web: {
    enabled: true,
    heartbeatSeconds: 60,
    reconnect: {
      initialMs: 2000,
      maxMs: 120000,
      factor: 1.4,
      jitter: 0.2,
      maxAttempts: 0,
    },
  },
}
```

<Accordion title="Multi-account WhatsApp">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- Los comandos salientes usan de forma predeterminada la cuenta `default` si existe; en caso contrario, el primer ID de cuenta configurado (ordenado).
- `channels.whatsapp.defaultAccount` opcional sobrescribe esa selección predeterminada de cuenta de respaldo cuando coincide con un ID de cuenta configurado.
- El directorio heredado de autenticación Baileys de una sola cuenta se migra mediante `openclaw doctor` a `whatsapp/default`.
- Anulaciones por cuenta: `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all
      linkPreview: true,
      streaming: "partial", // off | partial | block | progress (default: off; opt in explicitly to avoid preview-edit rate limits)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Token del bot: `channels.telegram.botToken` o `channels.telegram.tokenFile` (solo archivo normal; se rechazan enlaces simbólicos), con `TELEGRAM_BOT_TOKEN` como respaldo para la cuenta predeterminada.
- `channels.telegram.defaultAccount` opcional sobrescribe la selección predeterminada de cuenta cuando coincide con un ID de cuenta configurado.
- En configuraciones de múltiples cuentas (2 o más ID de cuenta), configura un valor predeterminado explícito (`channels.telegram.defaultAccount` o `channels.telegram.accounts.default`) para evitar el enrutamiento por respaldo; `openclaw doctor` muestra una advertencia cuando falta o no es válido.
- `configWrites: false` bloquea las escrituras de configuración iniciadas por Telegram (migraciones de ID de supergrupo, `/config set|unset`).
- Las entradas `bindings[]` de nivel superior con `type: "acp"` configuran enlaces ACP persistentes para temas de foros (usa el ID canónico `chatId:topic:topicId` en `match.peer.id`). La semántica de los campos se comparte en [Agentes ACP](/tools/acp-agents#channel-specific-settings).
- Las vistas previas de streaming de Telegram usan `sendMessage` + `editMessageText` (funciona en chats directos y de grupo).
- Política de reintento: consulta [Política de reintento](/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 8,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Short answers only.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      chunkMode: "length", // length | newline
      streaming: "off", // off | partial | block | progress (progress maps to partial on Discord)
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // opt-in for sessions_spawn({ thread: true })
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- Token: `channels.discord.token`, con `DISCORD_BOT_TOKEN` como respaldo para la cuenta predeterminada.
- Las llamadas salientes directas que proporcionan un `token` de Discord explícito usan ese token para la llamada; la configuración de reintento/política de la cuenta sigue viniendo de la cuenta seleccionada en la instantánea activa del entorno de ejecución.
- `channels.discord.defaultAccount` opcional sobrescribe la selección predeterminada de cuenta cuando coincide con un ID de cuenta configurado.
- Usa `user:<id>` (MD) o `channel:<id>` (canal del servidor) para los destinos de entrega; los ID numéricos sin prefijo se rechazan.
- Los slugs de los servidores van en minúsculas con espacios reemplazados por `-`; las claves de canal usan el nombre convertido en slug (sin `#`). Prefiere ID de servidor.
- Los mensajes escritos por bots se ignoran de forma predeterminada. `allowBots: true` los habilita; usa `allowBots: "mentions"` para aceptar solo mensajes de bots que mencionen al bot (los mensajes propios siguen filtrados).
- `channels.discord.guilds.<id>.ignoreOtherMentions` (y las anulaciones por canal) descarta mensajes que mencionan a otro usuario o rol pero no al bot (excluyendo @everyone/@here).
- `maxLinesPerMessage` (predeterminado 17) divide mensajes altos incluso cuando están por debajo de 2000 caracteres.
- `channels.discord.threadBindings` controla el enrutamiento vinculado a hilos de Discord:
  - `enabled`: anulación de Discord para funciones de sesión vinculadas a hilos (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` y entrega/enrutamiento vinculados)
  - `idleHours`: anulación de Discord para desfocalización automática por inactividad en horas (`0` desactiva)
  - `maxAgeHours`: anulación de Discord para edad máxima estricta en horas (`0` desactiva)
  - `spawnSubagentSessions`: interruptor optativo para la creación/enlace automático de hilos de `sessions_spawn({ thread: true })`
- Las entradas `bindings[]` de nivel superior con `type: "acp"` configuran enlaces ACP persistentes para canales e hilos (usa el ID de canal/hilo en `match.peer.id`). La semántica de los campos se comparte en [Agentes ACP](/tools/acp-agents#channel-specific-settings).
- `channels.discord.ui.components.accentColor` establece el color de acento para los contenedores de componentes v2 de Discord.
- `channels.discord.voice` habilita conversaciones en canales de voz de Discord y anulaciones opcionales de auto-join + TTS.
- `channels.discord.voice.daveEncryption` y `channels.discord.voice.decryptionFailureTolerance` se transfieren a las opciones DAVE de `@discordjs/voice` (`true` y `24` de forma predeterminada).
- OpenClaw también intenta recuperar la recepción de voz saliendo y volviendo a unirse a una sesión de voz después de fallos repetidos de descifrado.
- `channels.discord.streaming` es la clave canónica del modo de streaming. Los valores heredados `streamMode` y booleanos `streaming` se migran automáticamente.
- `channels.discord.autoPresence` asigna la disponibilidad del entorno de ejecución a la presencia del bot (saludable => online, degradado => idle, agotado => dnd) y permite anulaciones opcionales del texto de estado.
- `channels.discord.dangerouslyAllowNameMatching` vuelve a habilitar la coincidencia mutable por nombre/tag (modo de compatibilidad de emergencia).
- `channels.discord.execApprovals`: entrega nativa de aprobaciones de ejecución de Discord y autorización de aprobadores.
  - `enabled`: `true`, `false` o `"auto"` (predeterminado). En modo auto, las aprobaciones de ejecución se activan cuando se pueden resolver aprobadores desde `approvers` o `commands.ownerAllowFrom`.
  - `approvers`: ID de usuario de Discord autorizados para aprobar solicitudes de ejecución. Recurre a `commands.ownerAllowFrom` cuando se omite.
  - `agentFilter`: lista de permitidos opcional de ID de agente. Omítela para reenviar aprobaciones para todos los agentes.
  - `sessionFilter`: patrones opcionales de clave de sesión (subcadena o regex).
  - `target`: dónde enviar solicitudes de aprobación. `"dm"` (predeterminado) las envía a los MD de aprobadores, `"channel"` al canal de origen, `"both"` a ambos. Cuando el destino incluye `"channel"`, los botones solo pueden ser usados por aprobadores resueltos.
  - `cleanupAfterResolve`: cuando es `true`, elimina los MD de aprobación tras la aprobación, denegación o expiración.

**Modos de notificación de reacciones:** `off` (ninguna), `own` (mensajes del bot, predeterminado), `all` (todos los mensajes), `allowlist` (de `guilds.<id>.users` en todos los mensajes).

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dm: {
        enabled: true,
        policy: "pairing",
        allowFrom: ["users/1234567890"],
      },
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- JSON de cuenta de servicio: en línea (`serviceAccount`) o basado en archivo (`serviceAccountFile`).
- También se admite SecretRef para la cuenta de servicio (`serviceAccountRef`).
- Variables de entorno de respaldo: `GOOGLE_CHAT_SERVICE_ACCOUNT` o `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- Usa `spaces/<spaceId>` o `users/<userId>` para destinos de entrega.
- `channels.googlechat.dangerouslyAllowNameMatching` vuelve a habilitar la coincidencia mutable por principal de correo electrónico (modo de compatibilidad de emergencia).

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { allow: true, requireMention: true, allowBots: false },
        "#general": {
          allow: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Short answers only.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      textChunkLimit: 4000,
      chunkMode: "length",
      streaming: "partial", // off | partial | block | progress (preview mode)
      nativeStreaming: true, // use Slack native streaming API when streaming=partial
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- **Socket mode** requiere tanto `botToken` como `appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` como respaldo de variables de entorno de la cuenta predeterminada).
- **HTTP mode** requiere `botToken` más `signingSecret` (en la raíz o por cuenta).
- `botToken`, `appToken`, `signingSecret` y `userToken` aceptan cadenas en texto plano
  u objetos SecretRef.
- Las instantáneas de cuentas de Slack exponen campos por credencial de fuente/estado como
  `botTokenSource`, `botTokenStatus`, `appTokenStatus` y, en
  HTTP mode, `signingSecretStatus`. `configured_unavailable` significa que la cuenta está
  configurada mediante SecretRef, pero la ruta actual de comando/ejecución no pudo
  resolver el valor del secreto.
- `configWrites: false` bloquea las escrituras de configuración iniciadas por Slack.
- `channels.slack.defaultAccount` opcional sobrescribe la selección predeterminada de cuenta cuando coincide con un ID de cuenta configurado.
- `channels.slack.streaming` es la clave canónica del modo de streaming. Los valores heredados `streamMode` y booleanos `streaming` se migran automáticamente.
- Usa `user:<id>` (MD) o `channel:<id>` para destinos de entrega.

**Modos de notificación de reacciones:** `off`, `own` (predeterminado), `all`, `allowlist` (de `reactionAllowlist`).

**Aislamiento de sesión por hilo:** `thread.historyScope` es por hilo (predeterminado) o compartido en el canal. `thread.inheritParent` copia la transcripción del canal principal a los nuevos hilos.

- `typingReaction` añade una reacción temporal al mensaje entrante de Slack mientras se ejecuta una respuesta, y luego la elimina al completarse. Usa un shortcode de emoji de Slack como `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: entrega nativa de aprobaciones de ejecución de Slack y autorización de aprobadores. Mismo esquema que Discord: `enabled` (`true`/`false`/`"auto"`), `approvers` (ID de usuario de Slack), `agentFilter`, `sessionFilter` y `target` (`"dm"`, `"channel"` o `"both"`).

| Grupo de acciones | Predeterminado | Notas                  |
| ------------ | ------- | ---------------------- |
| reactions    | habilitado | Reaccionar + listar reacciones |
| messages     | habilitado | Leer/enviar/editar/eliminar  |
| pins         | habilitado | Fijar/desfijar/listar         |
| memberInfo   | habilitado | Información de miembro            |
| emojiList    | habilitado | Lista de emoji personalizados      |

### Mattermost

Mattermost se distribuye como plugin: `openclaw plugins install @openclaw/mattermost`.

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // opt-in
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Optional explicit URL for reverse-proxy/public deployments
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

Modos de chat: `oncall` (responde al @-mention, predeterminado), `onmessage` (cada mensaje), `onchar` (mensajes que empiezan con un prefijo disparador).

Cuando los comandos nativos de Mattermost están habilitados:

- `commands.callbackPath` debe ser una ruta (por ejemplo `/api/channels/mattermost/command`), no una URL completa.
- `commands.callbackUrl` debe resolverse al endpoint del gateway de OpenClaw y debe ser accesible desde el servidor de Mattermost.
- Los callbacks nativos de slash están autenticados con los tokens por comando devueltos
  por Mattermost durante el registro del slash command. Si el registro falla o no
  se activa ningún comando, OpenClaw rechaza los callbacks con
  `Unauthorized: invalid command token.`
- Para hosts de callback privados/tailnet/internos, Mattermost puede requerir
  que `ServiceSettings.AllowedUntrustedInternalConnections` incluya el host/dominio del callback.
  Usa valores de host/dominio, no URL completas.
- `channels.mattermost.configWrites`: permitir o denegar escrituras de configuración iniciadas por Mattermost.
- `channels.mattermost.requireMention`: requerir `@mention` antes de responder en canales.
- `channels.mattermost.groups.<channelId>.requireMention`: anulación por canal del control por mención (`"*"` para el valor predeterminado).
- `channels.mattermost.defaultAccount` opcional sobrescribe la selección predeterminada de cuenta cuando coincide con un ID de cuenta configurado.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // optional account binding
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**Modos de notificación de reacciones:** `off`, `own` (predeterminado), `all`, `allowlist` (de `reactionAllowlist`).

- `channels.signal.account`: fija el inicio del canal a una identidad específica de cuenta Signal.
- `channels.signal.configWrites`: permitir o denegar escrituras de configuración iniciadas por Signal.
- `channels.signal.defaultAccount` opcional sobrescribe la selección predeterminada de cuenta cuando coincide con un ID de cuenta configurado.

### BlueBubbles

BlueBubbles es la ruta recomendada para iMessage (respaldada por plugin, configurada en `channels.bluebubbles`).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, group controls, and advanced actions:
      // see /channels/bluebubbles
    },
  },
}
```

- Rutas clave principales cubiertas aquí: `channels.bluebubbles`, `channels.bluebubbles.dmPolicy`.
- `channels.bluebubbles.defaultAccount` opcional sobrescribe la selección predeterminada de cuenta cuando coincide con un ID de cuenta configurado.
- Las entradas `bindings[]` de nivel superior con `type: "acp"` pueden enlazar conversaciones de BlueBubbles a sesiones ACP persistentes. Usa un handle o cadena de destino de BlueBubbles (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) en `match.peer.id`. Semántica compartida de campos: [Agentes ACP](/tools/acp-agents#channel-specific-settings).
- La configuración completa del canal BlueBubbles está documentada en [BlueBubbles](/channels/bluebubbles).

### iMessage

OpenClaw inicia `imsg rpc` (JSON-RPC sobre stdio). No requiere daemon ni puerto.

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      region: "US",
    },
  },
}
```

- `channels.imessage.defaultAccount` opcional sobrescribe la selección predeterminada de cuenta cuando coincide con un ID de cuenta configurado.

- Requiere Full Disk Access a la base de datos de Messages.
- Prefiere destinos `chat_id:<id>`. Usa `imsg chats --limit 20` para listar chats.
- `cliPath` puede apuntar a un wrapper SSH; configura `remoteHost` (`host` o `user@host`) para obtener adjuntos por SCP.
- `attachmentRoots` y `remoteAttachmentRoots` restringen las rutas de adjuntos entrantes (predeterminado: `/Users/*/Library/Messages/Attachments`).
- SCP usa verificación estricta de claves de host, así que asegúrate de que la clave del host relay ya exista en `~/.ssh/known_hosts`.
- `channels.imessage.configWrites`: permitir o denegar escrituras de configuración iniciadas por iMessage.
- Las entradas `bindings[]` de nivel superior con `type: "acp"` pueden enlazar conversaciones de iMessage a sesiones ACP persistentes. Usa un handle normalizado o un destino de chat explícito (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) en `match.peer.id`. Semántica compartida de campos: [Agentes ACP](/tools/acp-agents#channel-specific-settings).

<Accordion title="iMessage SSH wrapper example">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix está respaldado por extensión y configurado en `channels.matrix`.

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- La autenticación por token usa `accessToken`; la autenticación por contraseña usa `userId` + `password`.
- `channels.matrix.proxy` enruta el tráfico HTTP de Matrix mediante un proxy HTTP(S) explícito. Las cuentas con nombre pueden sobrescribirlo con `channels.matrix.accounts.<id>.proxy`.
- `channels.matrix.allowPrivateNetwork` permite homeservers privados/internos. `proxy` y `allowPrivateNetwork` son controles independientes.
- `channels.matrix.defaultAccount` selecciona la cuenta preferida en configuraciones de múltiples cuentas.
- `channels.matrix.execApprovals`: entrega nativa de aprobaciones de ejecución de Matrix y autorización de aprobadores.
  - `enabled`: `true`, `false` o `"auto"` (predeterminado). En modo auto, las aprobaciones de ejecución se activan cuando se pueden resolver aprobadores desde `approvers` o `commands.ownerAllowFrom`.
  - `approvers`: ID de usuario de Matrix (por ejemplo `@owner:example.org`) autorizados para aprobar solicitudes de ejecución.
  - `agentFilter`: lista de permitidos opcional de ID de agente. Omítela para reenviar aprobaciones para todos los agentes.
  - `sessionFilter`: patrones opcionales de clave de sesión (subcadena o regex).
  - `target`: dónde enviar solicitudes de aprobación. `"dm"` (predeterminado), `"channel"` (sala de origen) o `"both"`.
  - Anulaciones por cuenta: `channels.matrix.accounts.<id>.execApprovals`.
- Los sondeos de estado de Matrix y las búsquedas en directorio en vivo usan la misma política de proxy que el tráfico del entorno de ejecución.
- La configuración completa de Matrix, reglas de destino y ejemplos de configuración están documentados en [Matrix](/channels/matrix).

### Microsoft Teams

Microsoft Teams está respaldado por extensión y configurado en `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, team/channel policies:
      // see /channels/msteams
    },
  },
}
```

- Rutas clave principales cubiertas aquí: `channels.msteams`, `channels.msteams.configWrites`.
- La configuración completa de Teams (credenciales, webhook, política de MD/grupos, anulaciones por equipo/por canal) está documentada en [Microsoft Teams](/channels/msteams).

### IRC

IRC está respaldado por extensión y configurado en `channels.irc`.

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- Rutas clave principales cubiertas aquí: `channels.irc`, `channels.irc.dmPolicy`, `channels.irc.configWrites`, `channels.irc.nickserv.*`.
- `channels.irc.defaultAccount` opcional sobrescribe la selección predeterminada de cuenta cuando coincide con un ID de cuenta configurado.
- La configuración completa del canal IRC (host/puerto/TLS/canales/listas de permitidos/control por mención) está documentada en [IRC](/channels/irc).

### Multi-account (todos los canales)

Ejecuta múltiples cuentas por canal (cada una con su propio `accountId`):

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- `default` se usa cuando se omite `accountId` (CLI + enrutamiento).
- Los tokens de entorno solo se aplican a la cuenta **default**.
- La configuración base del canal se aplica a todas las cuentas salvo que se sobrescriba por cuenta.
- Usa `bindings[].match.accountId` para enrutar cada cuenta a un agente diferente.
- Si añades una cuenta que no sea predeterminada mediante `openclaw channels add` (o mediante onboarding del canal) mientras sigues con una configuración de canal de nivel superior de una sola cuenta, OpenClaw primero promueve los valores de nivel superior de esa cuenta individual al mapa de cuentas del canal para que la cuenta original siga funcionando. La mayoría de los canales los mueven a `channels.<channel>.accounts.default`; Matrix puede conservar un destino con nombre/default existente que coincida.
- Los enlaces existentes solo de canal (sin `accountId`) siguen coincidiendo con la cuenta predeterminada; los enlaces con alcance de cuenta siguen siendo opcionales.
- `openclaw doctor --fix` también repara formas mixtas moviendo los valores de nivel superior de una sola cuenta con alcance de cuenta a la cuenta promovida elegida para ese canal. La mayoría de los canales usan `accounts.default`; Matrix puede conservar un destino con nombre/default existente que coincida.

### Otros canales de extensión

Muchos canales de extensión se configuran como `channels.<id>` y están documentados en sus páginas dedicadas de canal (por ejemplo Feishu, Matrix, LINE, Nostr, Zalo, Nextcloud Talk, Synology Chat y Twitch).
Consulta el índice completo de canales: [Canales](/channels).

### Control por mención en chats de grupo

Los mensajes de grupo requieren **mención** de forma predeterminada (mención en metadatos o patrones regex seguros). Se aplica a chats de grupo de WhatsApp, Telegram, Discord, Google Chat e iMessage.

**Tipos de mención:**

- **Menciones de metadatos**: @-mentions nativos de la plataforma. Se ignoran en modo self-chat de WhatsApp.
- **Patrones de texto**: patrones regex seguros en `agents.list[].groupChat.mentionPatterns`. Los patrones no válidos y la repetición anidada insegura se ignoran.
- El control por mención solo se aplica cuando la detección es posible (menciones nativas o al menos un patrón).

```json5
{
  messages: {
    groupChat: { historyLimit: 50 },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` establece el valor predeterminado global. Los canales pueden sobrescribirlo con `channels.<channel>.historyLimit` (o por cuenta). Configura `0` para desactivar.

#### Límites de historial de MD

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

Resolución: anulación por MD → valor predeterminado del proveedor → sin límite (se conserva todo).

Compatible con: `telegram`, `whatsapp`, `discord`, `slack`, `signal`, `imessage`, `msteams`.

#### Modo self-chat

Incluye tu propio número en `allowFrom` para habilitar el modo self-chat (ignora @-mentions nativos, solo responde a patrones de texto):

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### Comandos (manejo de comandos de chat)

```json5
{
  commands: {
    native: "auto", // register native commands when supported
    text: true, // parse /commands in chat messages
    bash: false, // allow ! (alias: /bash)
    bashForegroundMs: 2000,
    config: false, // allow /config
    debug: false, // allow /debug
    restart: false, // allow /restart + gateway restart tool
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="Command details">

- Los comandos de texto deben ser mensajes **independientes** con `/` al inicio.
- `native: "auto"` activa comandos nativos para Discord/Telegram, deja Slack desactivado.
- Anulación por canal: `channels.discord.commands.native` (bool o `"auto"`). `false` borra comandos registrados previamente.
- `channels.telegram.customCommands` añade entradas extra al menú del bot de Telegram.
- `bash: true` habilita `! <cmd>` para el shell del host. Requiere `tools.elevated.enabled` y que el remitente esté en `tools.elevated.allowFrom.<channel>`.
- `config: true` habilita `/config` (lee/escribe `openclaw.json`). Para clientes `chat.send` del gateway, las escrituras persistentes de `/config set|unset` también requieren `operator.admin`; `/config show` de solo lectura sigue disponible para clientes operator normales con alcance de escritura.
- `channels.<provider>.configWrites` controla mutaciones de configuración por canal (predeterminado: true).
- Para canales de múltiples cuentas, `channels.<provider>.accounts.<id>.configWrites` también controla escrituras dirigidas a esa cuenta (por ejemplo `/allowlist --config --account <id>` o `/config set channels.<provider>.accounts.<id>...`).
- `allowFrom` es por proveedor. Cuando está configurado, es la **única** fuente de autorización (las listas de permitidos/pairing del canal y `useAccessGroups` se ignoran).
- `useAccessGroups: false` permite que los comandos omitan las políticas de grupos de acceso cuando `allowFrom` no está configurado.

</Accordion>

---

## Valores predeterminados de agentes

### `agents.defaults.workspace`

Predeterminado: `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

Raíz opcional del repositorio que se muestra en la línea Runtime del prompt del sistema. Si no está configurada, OpenClaw la detecta automáticamente recorriendo hacia arriba desde el espacio de trabajo.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

Lista de permitidos predeterminada opcional de Skills para agentes que no configuran
`agents.list[].skills`.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // inherits github, weather
      { id: "docs", skills: ["docs-search"] }, // replaces defaults
      { id: "locked-down", skills: [] }, // no skills
    ],
  },
}
```

- Omite `agents.defaults.skills` para que las Skills no tengan restricciones de forma predeterminada.
- Omite `agents.list[].skills` para heredar los valores predeterminados.
- Configura `agents.list[].skills: []` para no tener Skills.
- Una lista no vacía en `agents.list[].skills` es el conjunto final para ese agente; no
  se combina con los valores predeterminados.

### `agents.defaults.skipBootstrap`

Deshabilita la creación automática de archivos bootstrap del espacio de trabajo (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.bootstrapMaxChars`

Máximo de caracteres por archivo bootstrap del espacio de trabajo antes del truncamiento. Predeterminado: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

Máximo total de caracteres inyectados entre todos los archivos bootstrap del espacio de trabajo. Predeterminado: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Controla el texto de advertencia visible para el agente cuando el contexto bootstrap se trunca.
Predeterminado: `"once"`.

- `"off"`: nunca inyecta texto de advertencia en el prompt del sistema.
- `"once"`: inyecta la advertencia una vez por cada firma de truncamiento única (recomendado).
- `"always"`: inyecta la advertencia en cada ejecución cuando exista truncamiento.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

Tamaño máximo en píxeles para el lado más largo de la imagen en bloques de imagen de transcripción/herramientas antes de las llamadas al proveedor.
Predeterminado: `1200`.

Los valores más bajos suelen reducir el uso de tokens de visión y el tamaño de la carga de la solicitud en ejecuciones con muchas capturas de pantalla.
Los valores más altos conservan más detalle visual.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

Zona horaria para el contexto del prompt del sistema (no para marcas de tiempo de mensajes). Usa como respaldo la zona horaria del host.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Formato de hora en el prompt del sistema. Predeterminado: `auto` (preferencia del SO).

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
        fallbacks: ["google/gemini-3.1-flash-image-preview"],
      },
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-i2v"],
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // global default provider params
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 3,
    },
  },
}
```

- `model`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - La forma de cadena establece solo el modelo principal.
  - La forma de objeto establece el principal más los modelos de failover ordenados.
- `imageModel`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - Lo usa la ruta de la herramienta `image` como su configuración de modelo de visión.
  - También se usa como enrutamiento de respaldo cuando el modelo seleccionado/predeterminado no puede aceptar entrada de imágenes.
- `imageGenerationModel`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - Lo usa la capacidad compartida de generación de imágenes y cualquier futura superficie de herramienta/plugin que genere imágenes.
  - Valores típicos: `google/gemini-3.1-flash-image-preview` para generación nativa de imágenes con Gemini, `fal/fal-ai/flux/dev` para fal, o `openai/gpt-image-1` para OpenAI Images.
  - Si seleccionas directamente un proveedor/modelo, configura también la autenticación/clave de API correspondiente del proveedor (por ejemplo `GEMINI_API_KEY` o `GOOGLE_API_KEY` para `google/*`, `OPENAI_API_KEY` para `openai/*`, `FAL_KEY` para `fal/*`).
  - Si se omite, `image_generate` aún puede inferir un valor predeterminado del proveedor respaldado por autenticación. Primero prueba el proveedor predeterminado actual y luego los proveedores restantes registrados de generación de imágenes en orden de ID de proveedor.
- `videoGenerationModel`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - Lo usa la capacidad compartida de generación de vídeo.
  - Valores típicos: `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` o `qwen/wan2.7-r2v`.
  - Configúralo explícitamente antes de usar la generación de vídeo compartida. A diferencia de `imageGenerationModel`, el entorno de ejecución de generación de vídeo todavía no infiere un proveedor predeterminado.
  - Si seleccionas directamente un proveedor/modelo, configura también la autenticación/clave de API correspondiente del proveedor.
  - El proveedor integrado de generación de vídeo de Qwen actualmente admite hasta 1 vídeo de salida, 1 imagen de entrada, 4 vídeos de entrada, 10 segundos de duración y opciones a nivel de proveedor `size`, `aspectRatio`, `resolution`, `audio` y `watermark`.
- `pdfModel`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - Lo usa la herramienta `pdf` para el enrutamiento de modelos.
  - Si se omite, la herramienta PDF recurre a `imageModel` y luego al modelo resuelto de la sesión/predeterminado.
- `pdfMaxBytesMb`: límite de tamaño PDF predeterminado para la herramienta `pdf` cuando no se pasa `maxBytesMb` al llamar a la herramienta.
- `pdfMaxPages`: máximo predeterminado de páginas consideradas por el modo de respaldo de extracción en la herramienta `pdf`.
- `verboseDefault`: nivel verbose predeterminado para agentes. Valores: `"off"`, `"on"`, `"full"`. Predeterminado: `"off"`.
- `elevatedDefault`: nivel predeterminado de salida elevada para agentes. Valores: `"off"`, `"on"`, `"ask"`, `"full"`. Predeterminado: `"on"`.
- `model.primary`: formato `provider/model` (por ejemplo `openai/gpt-5.4`). Si omites el proveedor, OpenClaw intenta primero un alias, luego una coincidencia única de proveedor configurado para ese ID de modelo exacto, y solo entonces recurre al proveedor predeterminado configurado (comportamiento heredado en desuso, así que prefiere `provider/model` explícito). Si ese proveedor ya no expone el modelo predeterminado configurado, OpenClaw recurre al primer proveedor/modelo configurado en lugar de mostrar un valor predeterminado obsoleto de un proveedor eliminado.
- `models`: el catálogo de modelos configurado y la lista de permitidos para `/model`. Cada entrada puede incluir `alias` (atajo) y `params` (específicos del proveedor, por ejemplo `temperature`, `maxTokens`, `cacheRetention`, `context1m`).
- `params`: parámetros globales predeterminados del proveedor aplicados a todos los modelos. Configúralos en `agents.defaults.params` (por ejemplo `{ cacheRetention: "long" }`).
- Precedencia de combinación de `params` (configuración): `agents.defaults.params` (base global) es sobrescrito por `agents.defaults.models["provider/model"].params` (por modelo), luego `agents.list[].params` (ID de agente coincidente) sobrescribe por clave. Consulta [Prompt Caching](/reference/prompt-caching) para más detalles.
- Los escritores de configuración que mutan estos campos (por ejemplo `/models set`, `/models set-image` y comandos add/remove de fallback) guardan la forma canónica de objeto y conservan las listas de fallback existentes cuando es posible.
- `maxConcurrent`: máximo de ejecuciones paralelas de agentes entre sesiones (cada sesión sigue serializada). Predeterminado: 4.

**Atajos de alias integrados** (solo se aplican cuando el modelo está en `agents.defaults.models`):

| Alias               | Modelo                                  |
| ------------------- | -------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`            |
| `sonnet`            | `anthropic/claude-sonnet-4-6`          |
| `gpt`               | `openai/gpt-5.4`                       |
| `gpt-mini`          | `openai/gpt-5.4-mini`                  |
| `gpt-nano`          | `openai/gpt-5.4-nano`                  |
| `gemini`            | `google/gemini-3.1-pro-preview`        |
| `gemini-flash`      | `google/gemini-3-flash-preview`        |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

Tus alias configurados siempre tienen prioridad sobre los predeterminados.

Los modelos GLM-4.x de Z.AI habilitan automáticamente el modo thinking salvo que configures `--thinking off` o definas `agents.defaults.models["zai/<model>"].params.thinking` tú mismo.
Los modelos Z.AI habilitan `tool_stream` de forma predeterminada para el streaming de llamadas a herramientas. Configura `agents.defaults.models["zai/<model>"].params.tool_stream` en `false` para desactivarlo.
Los modelos Claude 4.6 de Anthropic usan `adaptive` thinking de forma predeterminada cuando no se establece un nivel explícito de thinking.

### `agents.defaults.cliBackends`

Backends CLI opcionales para ejecuciones de respaldo solo de texto (sin llamadas a herramientas). Son útiles como respaldo cuando fallan los proveedores de API.

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "claude-cli": {
          command: "/opt/homebrew/bin/claude",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          modelArg: "--model",
          sessionArg: "--session",
          sessionMode: "existing",
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
        },
      },
    },
  },
}
```

- Los backends CLI están orientados primero al texto; las herramientas siempre están deshabilitadas.
- Se admiten sesiones cuando `sessionArg` está configurado.
- Se admite paso directo de imágenes cuando `imageArg` acepta rutas de archivo.

### `agents.defaults.heartbeat`

Ejecuciones periódicas de heartbeat.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m disables
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        lightContext: false, // default: false; true keeps only HEARTBEAT.md from workspace bootstrap files
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (default) | block
        target: "none", // default: none | options: last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
      },
    },
  },
}
```

- `every`: cadena de duración (ms/s/m/h). Predeterminado: `30m` (autenticación con clave de API) o `1h` (autenticación OAuth). Configura `0m` para desactivar.
- `suppressToolErrorWarnings`: cuando es true, suprime las cargas útiles de advertencia de error de herramienta durante ejecuciones de heartbeat.
- `directPolicy`: política de entrega directa/MD. `allow` (predeterminado) permite entrega a destinos directos. `block` suprime la entrega a destinos directos y emite `reason=dm-blocked`.
- `lightContext`: cuando es true, las ejecuciones de heartbeat usan contexto bootstrap liviano y conservan solo `HEARTBEAT.md` de los archivos bootstrap del espacio de trabajo.
- `isolatedSession`: cuando es true, cada ejecución de heartbeat se realiza en una sesión nueva sin historial de conversación previo. Mismo patrón de aislamiento que cron `sessionTarget: "isolated"`. Reduce el costo de tokens por heartbeat de ~100K a ~2-5K tokens.
- Por agente: configura `agents.list[].heartbeat`. Cuando algún agente define `heartbeat`, **solo esos agentes** ejecutan heartbeats.
- Los heartbeats ejecutan turnos completos de agente: los intervalos más cortos consumen más tokens.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        timeoutSeconds: 900,
        reserveTokensFloor: 24000,
        identifierPolicy: "strict", // strict | off | custom
        identifierInstructions: "Preserve deployment IDs, ticket IDs, and host:port pairs exactly.", // used when identifierPolicy=custom
        postCompactionSections: ["Session Startup", "Red Lines"], // [] disables reinjection
        model: "openrouter/anthropic/claude-sonnet-4-6", // optional compaction-only model override
        notifyUser: true, // send a brief notice when compaction starts (default: false)
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 6000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with the exact silent token NO_REPLY if nothing to store.",
        },
      },
    },
  },
}
```

- `mode`: `default` o `safeguard` (resumen por bloques para historiales largos). Consulta [Compactación](/concepts/compaction).
- `timeoutSeconds`: máximo de segundos permitidos para una sola operación de compactación antes de que OpenClaw la aborte. Predeterminado: `900`.
- `identifierPolicy`: `strict` (predeterminado), `off` o `custom`. `strict` antepone orientación integrada de conservación de identificadores opacos durante el resumen de compactación.
- `identifierInstructions`: texto opcional personalizado de preservación de identificadores usado cuando `identifierPolicy=custom`.
- `postCompactionSections`: nombres opcionales de secciones H2/H3 de AGENTS.md para reinyectar después de la compactación. Predeterminado: `["Session Startup", "Red Lines"]`; configura `[]` para desactivar la reinyección. Cuando no está configurado o se establece explícitamente en ese par predeterminado, también se aceptan los encabezados antiguos `Every Session`/`Safety` como respaldo heredado.
- `model`: anulación opcional `provider/model-id` solo para el resumen de compactación. Úsalo cuando la sesión principal deba mantener un modelo pero los resúmenes de compactación deban ejecutarse en otro; si no se configura, la compactación usa el modelo principal de la sesión.
- `notifyUser`: cuando es `true`, envía un aviso breve al usuario cuando empieza la compactación (por ejemplo, "Compacting context..."). Está desactivado de forma predeterminada para mantener la compactación silenciosa.
- `memoryFlush`: turno agéntico silencioso antes de la compactación automática para almacenar memorias duraderas. Se omite cuando el espacio de trabajo es de solo lectura.

### `agents.defaults.contextPruning`

Poda **resultados antiguos de herramientas** del contexto en memoria antes de enviarlo al LLM. **No** modifica el historial de sesión en disco.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // duration (ms/s/m/h), default unit: minutes
        keepLastAssistants: 3,
        softTrimRatio: 0.3,
        hardClearRatio: 0.5,
        minPrunableToolChars: 50000,
        softTrim: { maxChars: 4000, headChars: 1500, tailChars: 1500 },
        hardClear: { enabled: true, placeholder: "[Old tool result content cleared]" },
        tools: { deny: ["browser", "canvas"] },
      },
    },
  },
}
```

<Accordion title="cache-ttl mode behavior">

- `mode: "cache-ttl"` habilita pasadas de poda.
- `ttl` controla con qué frecuencia puede volver a ejecutarse la poda (después del último toque de caché).
- La poda primero recorta suavemente resultados sobredimensionados de herramientas y luego vacía por completo los resultados de herramientas más antiguos si es necesario.

**Soft-trim** conserva el principio y el final e inserta `...` en el medio.

**Hard-clear** reemplaza todo el resultado de la herramienta por el marcador.

Notas:

- Los bloques de imagen nunca se recortan ni vacían.
- Las proporciones se basan en caracteres (aproximadas), no en conteos exactos de tokens.
- Si existen menos de `keepLastAssistants` mensajes del asistente, se omite la poda.

</Accordion>

Consulta [Poda de sesión](/concepts/session-pruning) para detalles del comportamiento.

### Block streaming

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom (use minMs/maxMs)
    },
  },
}
```

- Los canales que no son Telegram requieren `*.blockStreaming: true` explícito para habilitar respuestas por bloques.
- Anulaciones por canal: `channels.<channel>.blockStreamingCoalesce` (y variantes por cuenta). Signal/Slack/Discord/Google Chat usan `minChars: 1500` de forma predeterminada.
- `humanDelay`: pausa aleatoria entre respuestas por bloques. `natural` = 800–2500 ms. Anulación por agente: `agents.list[].humanDelay`.

Consulta [Streaming](/concepts/streaming) para el comportamiento y detalles de particionado.

### Indicadores de escritura

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- Valores predeterminados: `instant` para chats directos/menciones, `message` para chats de grupo sin mención.
- Anulaciones por sesión: `session.typingMode`, `session.typingIntervalSeconds`.

Consulta [Indicadores de escritura](/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Sandboxing opcional para el agente integrado. Consulta [Sandboxing](/gateway/sandboxing) para la guía completa.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        backend: "docker", // docker | ssh | openshell
        scope: "agent", // session | agent | shared
        workspaceAccess: "none", // none | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefs / inline contents also supported:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

<Accordion title="Sandbox details">

**Backend:**

- `docker`: entorno local de Docker (predeterminado)
- `ssh`: entorno remoto genérico respaldado por SSH
- `openshell`: entorno OpenShell

Cuando se selecciona `backend: "openshell"`, los ajustes específicos del entorno de ejecución pasan a
`plugins.entries.openshell.config`.

**Configuración del backend SSH:**

- `target`: destino SSH con formato `user@host[:port]`
- `command`: comando del cliente SSH (predeterminado: `ssh`)
- `workspaceRoot`: raíz remota absoluta usada para espacios de trabajo por alcance
- `identityFile` / `certificateFile` / `knownHostsFile`: archivos locales existentes pasados a OpenSSH
- `identityData` / `certificateData` / `knownHostsData`: contenidos en línea o SecretRefs que OpenClaw materializa en archivos temporales en tiempo de ejecución
- `strictHostKeyChecking` / `updateHostKeys`: opciones de política de claves de host de OpenSSH

**Precedencia de autenticación SSH:**

- `identityData` tiene prioridad sobre `identityFile`
- `certificateData` tiene prioridad sobre `certificateFile`
- `knownHostsData` tiene prioridad sobre `knownHostsFile`
- Los valores `*Data` respaldados por SecretRef se resuelven desde la instantánea activa del entorno de secretos antes de que comience la sesión sandbox

**Comportamiento del backend SSH:**

- siembra el espacio de trabajo remoto una vez después de crear o recrear
- luego mantiene canónico el espacio de trabajo SSH remoto
- enruta `exec`, las herramientas de archivos y rutas de medios por SSH
- no sincroniza automáticamente cambios remotos de vuelta al host
- no admite contenedores de navegador sandbox

**Acceso al espacio de trabajo:**

- `none`: espacio de trabajo sandbox por alcance en `~/.openclaw/sandboxes`
- `ro`: espacio de trabajo sandbox en `/workspace`, espacio de trabajo del agente montado como solo lectura en `/agent`
- `rw`: espacio de trabajo del agente montado con lectura/escritura en `/workspace`

**Scope:**

- `session`: contenedor + espacio de trabajo por sesión
- `agent`: un contenedor + espacio de trabajo por agente (predeterminado)
- `shared`: contenedor y espacio de trabajo compartidos (sin aislamiento entre sesiones)

**Configuración del plugin OpenShell:**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // mirror | remote
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // optional
          gatewayEndpoint: "https://lab.example", // optional
          policy: "strict", // optional OpenShell policy id
          providers: ["openai"], // optional
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**Modo OpenShell:**

- `mirror`: siembra el remoto desde el local antes de exec, sincroniza de vuelta tras exec; el espacio de trabajo local sigue siendo canónico
- `remote`: siembra el remoto una vez cuando se crea el sandbox, luego mantiene canónico el espacio de trabajo remoto

En modo `remote`, las ediciones locales del host hechas fuera de OpenClaw no se sincronizan automáticamente con el sandbox después del paso de siembra.
El transporte es SSH dentro del sandbox de OpenShell, pero el plugin posee el ciclo de vida del sandbox y la sincronización mirror opcional.

**`setupCommand`** se ejecuta una vez después de crear el contenedor (mediante `sh -lc`). Necesita salida de red, raíz con escritura y usuario root.

**Los contenedores usan `network: "none"` de forma predeterminada**: configúralo en `"bridge"` (o una red bridge personalizada) si el agente necesita acceso saliente.
`"host"` está bloqueado. `"container:<id>"` está bloqueado de forma predeterminada salvo que configures explícitamente
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` (emergencia).

**Los adjuntos entrantes** se preparan en `media/inbound/*` en el espacio de trabajo activo.

**`docker.binds`** monta directorios adicionales del host; los binds globales y por agente se combinan.

**Navegador sandbox** (`sandbox.browser.enabled`): Chromium + CDP en un contenedor. La URL de noVNC se inyecta en el prompt del sistema. No requiere `browser.enabled` en `openclaw.json`.
El acceso de observación noVNC usa autenticación VNC de forma predeterminada y OpenClaw emite una URL con token de corta duración (en lugar de exponer la contraseña en la URL compartida).

- `allowHostControl: false` (predeterminado) bloquea que las sesiones sandbox apunten al navegador del host.
- `network` usa `openclaw-sandbox-browser` de forma predeterminada (red bridge dedicada). Configúralo en `bridge` solo cuando quieras explícitamente conectividad global del bridge.
- `cdpSourceRange` opcionalmente restringe el ingreso a CDP en el borde del contenedor a un rango CIDR (por ejemplo `172.21.0.1/32`).
- `sandbox.browser.binds` monta directorios adicionales del host solo dentro del contenedor del navegador sandbox. Cuando se configura (incluido `[]`), reemplaza `docker.binds` para el contenedor del navegador.
- Los valores de inicio predeterminados están definidos en `scripts/sandbox-browser-entrypoint.sh` y ajustados para hosts con contenedores:
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-3d-apis`
  - `--disable-gpu`
  - `--disable-software-rasterizer`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-features=TranslateUI`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--renderer-process-limit=2`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--disable-extensions` (habilitado por defecto)
  - `--disable-3d-apis`, `--disable-software-rasterizer` y `--disable-gpu` están
    habilitados por defecto y pueden deshabilitarse con
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` si el uso de WebGL/3D lo requiere.
  - `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` vuelve a habilitar extensiones si tu flujo de trabajo
    depende de ellas.
  - `--renderer-process-limit=2` puede cambiarse con
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`; configura `0` para usar el
    límite predeterminado de procesos de Chromium.
  - además de `--no-sandbox` y `--disable-setuid-sandbox` cuando `noSandbox` está habilitado.
  - Los valores predeterminados son la línea base de la imagen del contenedor; usa una imagen de navegador personalizada con un entrypoint personalizado para cambiar los valores predeterminados del contenedor.

</Accordion>

El sandboxing del navegador y `sandbox.docker.binds` actualmente son solo para Docker.

Compila las imágenes:

```bash
scripts/sandbox-setup.sh           # main sandbox image
scripts/sandbox-browser-setup.sh   # optional browser image
```

### `agents.list` (anulaciones por agente)

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // or { primary, fallbacks }
        thinkingDefault: "high", // per-agent thinking level override
        reasoningDefault: "on", // per-agent reasoning visibility override
        fastModeDefault: false, // per-agent fast mode override
        params: { cacheRetention: "none" }, // overrides matching defaults.models params by key
        skills: ["docs-search"], // replaces agents.defaults.skills when set
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`: ID estable del agente (obligatorio).
- `default`: cuando se configuran varios, el primero gana (se registra advertencia). Si no se configura ninguno, la primera entrada de la lista es la predeterminada.
- `model`: la forma de cadena sobrescribe solo `primary`; la forma de objeto `{ primary, fallbacks }` sobrescribe ambos (`[]` desactiva los fallbacks globales). Los trabajos cron que solo sobrescriben `primary` siguen heredando los fallbacks predeterminados salvo que configures `fallbacks: []`.
- `params`: parámetros de flujo por agente combinados sobre la entrada de modelo seleccionada en `agents.defaults.models`. Úsalo para anulaciones específicas del agente como `cacheRetention`, `temperature` o `maxTokens` sin duplicar todo el catálogo de modelos.
- `skills`: lista de permitidos opcional de Skills por agente. Si se omite, el agente hereda `agents.defaults.skills` cuando está configurado; una lista explícita reemplaza los valores predeterminados en lugar de combinarlos, y `[]` significa sin Skills.
- `thinkingDefault`: valor predeterminado opcional de thinking por agente (`off | minimal | low | medium | high | xhigh | adaptive`). Sobrescribe `agents.defaults.thinkingDefault` para este agente cuando no se ha establecido una anulación por mensaje o sesión.
- `reasoningDefault`: visibilidad predeterminada opcional de reasoning por agente (`on | off | stream`). Se aplica cuando no se ha establecido una anulación de reasoning por mensaje o sesión.
- `fastModeDefault`: valor predeterminado opcional por agente para fast mode (`true | false`). Se aplica cuando no se ha establecido una anulación por mensaje o sesión.
- `runtime`: descriptor opcional del entorno de ejecución por agente. Usa `type: "acp"` con valores predeterminados `runtime.acp` (`agent`, `backend`, `mode`, `cwd`) cuando el agente deba usar por defecto sesiones de arnés ACP.
- `identity.avatar`: ruta relativa al espacio de trabajo, URL `http(s)` o URI `data:`.
- `identity` deriva valores predeterminados: `ackReaction` de `emoji`, `mentionPatterns` de `name`/`emoji`.
- `subagents.allowAgents`: lista de permitidos de ID de agente para `sessions_spawn` (`["*"]` = cualquiera; predeterminado: solo el mismo agente).
- Protección de herencia del sandbox: si la sesión solicitante está en sandbox, `sessions_spawn` rechaza destinos que se ejecutarían sin sandbox.
- `subagents.requireAgentId`: cuando es true, bloquea llamadas a `sessions_spawn` que omitan `agentId` (fuerza selección explícita de perfil; predeterminado: false).

---

## Enrutamiento multiagente

Ejecuta múltiples agentes aislados dentro de un mismo Gateway. Consulta [Multi-Agent](/concepts/multi-agent).

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### Campos `match` del binding

- `type` (opcional): `route` para enrutamiento normal (si falta `type`, usa route por defecto), `acp` para enlaces ACP persistentes de conversación.
- `match.channel` (obligatorio)
- `match.accountId` (opcional; `*` = cualquier cuenta; omitido = cuenta predeterminada)
- `match.peer` (opcional; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (opcional; específicos del canal)
- `acp` (opcional; solo para entradas `type: "acp"`): `{ mode, label, cwd, backend }`

**Orden de coincidencia determinista:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (exacto, sin peer/guild/team)
5. `match.accountId: "*"` (a nivel de canal)
6. Agente predeterminado

Dentro de cada nivel, gana la primera entrada de `bindings` que coincide.

Para las entradas `type: "acp"`, OpenClaw resuelve por identidad exacta de conversación (`match.channel` + cuenta + `match.peer.id`) y no usa el orden de niveles de route anterior.

### Perfiles de acceso por agente

<Accordion title="Full access (no sandbox)">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Read-only tools + workspace">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="No filesystem access (messaging only)">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

Consulta [Sandbox y herramientas multiagente](/tools/multi-agent-sandbox-tools) para detalles de precedencia.

---

## Sesión

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    parentForkMaxTokens: 100000, // skip parent-thread fork above this token count (0 disables)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // duration or false
      maxDiskBytes: "500mb", // optional hard budget
      highWaterBytes: "400mb", // optional cleanup target
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // default inactivity auto-unfocus in hours (`0` disables)
      maxAgeHours: 0, // default hard max age in hours (`0` disables)
    },
    mainKey: "main", // legacy (runtime always uses "main")
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Session field details">

- **`scope`**: estrategia base de agrupación de sesión para contextos de chat de grupo.
  - `per-sender` (predeterminado): cada remitente tiene una sesión aislada dentro de un contexto de canal.
  - `global`: todos los participantes de un contexto de canal comparten una sola sesión (úsalo solo cuando se pretenda contexto compartido).
- **`dmScope`**: cómo se agrupan los MD.
  - `main`: todos los MD comparten la sesión principal.
  - `per-peer`: aislar por ID de remitente entre canales.
  - `per-channel-peer`: aislar por canal + remitente (recomendado para bandejas de entrada multiusuario).
  - `per-account-channel-peer`: aislar por cuenta + canal + remitente (recomendado para múltiples cuentas).
- **`identityLinks`**: asigna ID canónicos a peers con prefijo de proveedor para compartir sesiones entre canales.
- **`reset`**: política principal de reinicio. `daily` reinicia a la `atHour` en hora local; `idle` reinicia tras `idleMinutes`. Cuando ambos están configurados, gana el que expire primero.
- **`resetByType`**: anulaciones por tipo (`direct`, `group`, `thread`). Se acepta el heredado `dm` como alias de `direct`.
- **`parentForkMaxTokens`**: máximo de `totalTokens` permitido en la sesión padre al crear una sesión de hilo bifurcada (predeterminado `100000`).
  - Si `totalTokens` del padre está por encima de este valor, OpenClaw inicia una sesión de hilo nueva en lugar de heredar el historial de la transcripción padre.
  - Configura `0` para desactivar esta protección y permitir siempre la bifurcación desde el padre.
- **`mainKey`**: campo heredado. El entorno de ejecución ahora siempre usa `"main"` para el bucket principal de chat directo.
- **`agentToAgent.maxPingPongTurns`**: máximo de turnos de respuesta entre agentes durante intercambios agente a agente (entero, rango: `0`–`5`). `0` desactiva el encadenamiento ping-pong.
- **`sendPolicy`**: coincide por `channel`, `chatType` (`direct|group|channel`, con alias heredado `dm`), `keyPrefix` o `rawKeyPrefix`. La primera denegación gana.
- **`maintenance`**: controles de limpieza y retención del almacén de sesiones.
  - `mode`: `warn` solo emite advertencias; `enforce` aplica la limpieza.
  - `pruneAfter`: límite de antigüedad para entradas obsoletas (predeterminado `30d`).
  - `maxEntries`: número máximo de entradas en `sessions.json` (predeterminado `500`).
  - `rotateBytes`: rota `sessions.json` cuando supera este tamaño (predeterminado `10mb`).
  - `resetArchiveRetention`: retención para archivos de transcripción `*.reset.<timestamp>`. De forma predeterminada usa `pruneAfter`; configura `false` para desactivar.
  - `maxDiskBytes`: presupuesto opcional de disco para el directorio de sesiones. En modo `warn` registra advertencias; en modo `enforce` elimina primero los artefactos/sesiones más antiguos.
  - `highWaterBytes`: objetivo opcional tras la limpieza por presupuesto. De forma predeterminada usa `80%` de `maxDiskBytes`.
- **`threadBindings`**: valores predeterminados globales para funciones de sesión vinculadas a hilos.
  - `enabled`: interruptor maestro predeterminado (los proveedores pueden sobrescribirlo; Discord usa `channels.discord.threadBindings.enabled`)
  - `idleHours`: desfocalización automática predeterminada por inactividad en horas (`0` desactiva; los proveedores pueden sobrescribir)
  - `maxAgeHours`: edad máxima estricta predeterminada en horas (`0` desactiva; los proveedores pueden sobrescribir)

</Accordion>

---

## Mensajes

```json5
{
  messages: {
    responsePrefix: "🦞", // or "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all
    removeAckAfterReply: false,
    queue: {
      mode: "collect", // steer | followup | collect | steer-backlog | steer+backlog | queue | interrupt
      debounceMs: 1000,
      cap: 20,
      drop: "summarize", // old | new | summarize
      byChannel: {
        whatsapp: "collect",
        telegram: "collect",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 disables
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Prefijo de respuesta

Anulaciones por canal/cuenta: `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Resolución (gana el más específico): cuenta → canal → global. `""` desactiva y detiene la cascada. `"auto"` deriva `[{identity.name}]`.

**Variables de plantilla:**

| Variable          | Descripción            | Ejemplo                     |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | Nombre corto del modelo       | `claude-opus-4-6`           |
| `{modelFull}`     | Identificador completo del modelo  | `anthropic/claude-opus-4-6` |
| `{provider}`      | Nombre del proveedor          | `anthropic`                 |
| `{thinkingLevel}` | Nivel actual de thinking | `high`, `low`, `off`        |
| `{identity.name}` | Nombre de la identidad del agente    | (igual que `"auto"`)          |

Las variables no distinguen mayúsculas/minúsculas. `{think}` es un alias de `{thinkingLevel}`.

### Reacción de confirmación

- De forma predeterminada usa `identity.emoji` del agente activo o, en su defecto, `"👀"`. Configura `""` para desactivar.
- Anulaciones por canal: `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Orden de resolución: cuenta → canal → `messages.ackReaction` → respaldo de identidad.
- Alcance: `group-mentions` (predeterminado), `group-all`, `direct`, `all`.
- `removeAckAfterReply`: elimina la confirmación tras responder en Slack, Discord y Telegram.
- `messages.statusReactions.enabled`: habilita reacciones de estado del ciclo de vida en Slack, Discord y Telegram.
  En Slack y Discord, si no está configurado, las reacciones de estado permanecen habilitadas cuando las reacciones de confirmación están activas.
  En Telegram, configúralo explícitamente en `true` para habilitar reacciones de estado del ciclo de vida.

### Debounce entrante

Agrupa mensajes rápidos de solo texto del mismo remitente en un solo turno de agente. Los medios/adjuntos se vacían inmediatamente. Los comandos de control omiten el debounce.

### TTS (texto a voz)

```json5
{
  messages: {
    tts: {
      auto: "always", // off | always | inbound | tagged
      mode: "final", // final | all
      provider: "elevenlabs",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: { enabled: true },
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
      elevenlabs: {
        apiKey: "elevenlabs_api_key",
        baseUrl: "https://api.elevenlabs.io",
        voiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      openai: {
        apiKey: "openai_api_key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        voice: "alloy",
      },
    },
  },
}
```

- `auto` controla el auto-TTS. `/tts off|always|inbound|tagged` lo sobrescribe por sesión.
- `summaryModel` sobrescribe `agents.defaults.model.primary` para el resumen automático.
- `modelOverrides` está habilitado por defecto; `modelOverrides.allowProvider` usa `false` como valor predeterminado (opt-in).
- Las claves de API usan como respaldo `ELEVENLABS_API_KEY`/`XI_API_KEY` y `OPENAI_API_KEY`.
- `openai.baseUrl` sobrescribe el endpoint de TTS de OpenAI. El orden de resolución es configuración, luego `OPENAI_TTS_BASE_URL` y después `https://api.openai.com/v1`.
- Cuando `openai.baseUrl` apunta a un endpoint que no es de OpenAI, OpenClaw lo trata como un servidor TTS compatible con OpenAI y relaja la validación de modelo/voz.

---

## Talk

Valores predeterminados para el modo Talk (macOS/iOS/Android).

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
    },
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

- `talk.provider` debe coincidir con una clave de `talk.providers` cuando se configuran varios proveedores Talk.
- Las claves heredadas planas de Talk (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) son solo de compatibilidad y se migran automáticamente a `talk.providers.<provider>`.
- Los ID de voz usan como respaldo `ELEVENLABS_VOICE_ID` o `SAG_VOICE_ID`.
- `providers.*.apiKey` acepta cadenas en texto plano u objetos SecretRef.
- El respaldo `ELEVENLABS_API_KEY` solo se aplica cuando no hay una clave de API de Talk configurada.
- `providers.*.voiceAliases` permite que las directivas de Talk usen nombres amigables.
- `silenceTimeoutMs` controla cuánto espera el modo Talk después del silencio del usuario antes de enviar la transcripción. Si no se configura, conserva la ventana de pausa predeterminada de la plataforma (`700 ms en macOS y Android, 900 ms en iOS`).

---

## Herramientas

### Perfiles de herramientas

`tools.profile` establece una lista de permitidos base antes de `tools.allow`/`tools.deny`:

El onboarding local usa por defecto `tools.profile: "coding"` en las nuevas configuraciones locales cuando no está configurado (los perfiles explícitos existentes se conservan).

| Perfil     | Incluye                                                                                                      |
| ----------- | ------------------------------------------------------------------------------------------------------------- |
| `minimal`   | solo `session_status`                                                                                         |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                     |
| `full`      | Sin restricciones (igual que sin configurar)                                                                                |

### Grupos de herramientas

| Grupo              | Herramientas                                                                                                                   |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` se acepta como alias de `exec`)                                         |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                  |
| `group:sessions`   | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                           |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                   |
| `group:ui`         | `browser`, `canvas`                                                                                                     |
| `group:automation` | `cron`, `gateway`                                                                                                       |
| `group:messaging`  | `message`                                                                                                               |
| `group:nodes`      | `nodes`                                                                                                                 |
| `group:agents`     | `agents_list`                                                                                                           |
| `group:media`      | `image`, `image_generate`, `tts`                                                                                        |
| `group:openclaw`   | Todas las herramientas integradas (excluye plugins de proveedor)                                                                          |

### `tools.allow` / `tools.deny`

Política global de permitir/denegar herramientas (deny gana). No distingue mayúsculas/minúsculas y admite comodines `*`. Se aplica incluso cuando el sandbox Docker está desactivado.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

Restringe aún más las herramientas para proveedores o modelos específicos. Orden: perfil base → perfil del proveedor → allow/deny.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.elevated`

Controla el acceso elevado a exec fuera del sandbox:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- La anulación por agente (`agents.list[].tools.elevated`) solo puede restringir aún más.
- `/elevated on|off|ask|full` almacena el estado por sesión; las directivas inline se aplican a un único mensaje.
- `exec` elevado omite el sandbox y usa la ruta de escape configurada (`gateway` de forma predeterminada, o `node` cuando el destino de exec es `node`).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      applyPatch: {
        enabled: false,
        allowModels: ["gpt-5.4"],
      },
    },
  },
}
```

### `tools.loopDetection`

Las comprobaciones de seguridad de bucles de herramientas están **deshabilitadas de forma predeterminada**. Configura `enabled: true` para activar la detección.
La configuración puede definirse globalmente en `tools.loopDetection` y sobrescribirse por agente en `agents.list[].tools.loopDetection`.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },
  },
}
```

- `historySize`: máximo historial de llamadas a herramientas conservado para análisis de bucles.
- `warningThreshold`: umbral de patrón repetitivo sin progreso para advertencias.
- `criticalThreshold`: umbral repetitivo superior para bloquear bucles críticos.
- `globalCircuitBreakerThreshold`: umbral de parada estricta para cualquier ejecución sin progreso.
- `detectors.genericRepeat`: advierte sobre llamadas repetidas a la misma herramienta/con los mismos argumentos.
- `detectors.knownPollNoProgress`: advierte/bloquea herramientas de sondeo conocidas (`process.poll`, `command_status`, etc.).
- `detectors.pingPong`: advierte/bloquea patrones alternos por pares sin progreso.
- Si `warningThreshold >= criticalThreshold` o `criticalThreshold >= globalCircuitBreakerThreshold`, la validación falla.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // or BRAVE_API_KEY env
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // optional; omit for auto-detect
        maxChars: 50000,
        maxCharsCap: 50000,
        maxResponseBytes: 2000000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

### `tools.media`

Configura el entendimiento de medios entrantes (imagen/audio/vídeo):

```json5
{
  tools: {
    media: {
      concurrency: 2,
      audio: {
        enabled: true,
        maxBytes: 20971520,
        scope: {
          default: "deny",
          rules: [{ action: "allow", match: { chatType: "direct" } }],
        },
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          { type: "cli", command: "whisper", args: ["--model", "base", "{{MediaPath}}"] },
        ],
      },
      video: {
        enabled: true,
        maxBytes: 52428800,
        models: [{ provider: "google", model: "gemini-3-flash-preview" }],
      },
    },
  },
}
```

<Accordion title="Media model entry fields">

**Entrada de proveedor** (`type: "provider"` u omitido):

- `provider`: ID del proveedor de API (`openai`, `anthropic`, `google`/`gemini`, `groq`, etc.)
- `model`: anulación del ID del modelo
- `profile` / `preferredProfile`: selección de perfil de `auth-profiles.json`

**Entrada CLI** (`type: "cli"`):

- `command`: ejecutable que se ejecutará
- `args`: argumentos con plantillas (admite `{{MediaPath}}`, `{{Prompt}}`, `{{MaxChars}}`, etc.)

**Campos comunes:**

- `capabilities`: lista opcional (`image`, `audio`, `video`). Predeterminados: `openai`/`anthropic`/`minimax` → imagen, `google` → imagen+audio+vídeo, `groq` → audio.
- `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: anulaciones por entrada.
- Los fallos recurren a la siguiente entrada.

La autenticación del proveedor sigue el orden estándar: `auth-profiles.json` → variables de entorno → `models.providers.*.apiKey`.

</Accordion>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

Controla qué sesiones pueden ser objetivo de las herramientas de sesión (`sessions_list`, `sessions_history`, `sessions_send`).

Predeterminado: `tree` (sesión actual + sesiones generadas por ella, como subagentes).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

Notas:

- `self`: solo la clave de sesión actual.
- `tree`: sesión actual + sesiones generadas por la sesión actual (subagentes).
- `agent`: cualquier sesión que pertenezca al ID del agente actual (puede incluir otros usuarios si ejecutas sesiones por remitente bajo el mismo ID de agente).
- `all`: cualquier sesión. El direccionamiento entre agentes sigue requiriendo `tools.agentToAgent`.
- Restricción por sandbox: cuando la sesión actual está en sandbox y `agents.defaults.sandbox.sessionToolsVisibility="spawned"`, la visibilidad se fuerza a `tree` incluso si `tools.sessions.visibility="all"`.

### `tools.sessions_spawn`

Controla la compatibilidad de adjuntos inline para `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: set true to allow inline file attachments
        maxTotalBytes: 5242880, // 5 MB total across all files
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB per file
        retainOnSessionKeep: false, // keep attachments when cleanup="keep"
      },
    },
  },
}
```

Notas:

- Los adjuntos solo son compatibles con `runtime: "subagent"`. ACP runtime los rechaza.
- Los archivos se materializan en el espacio de trabajo hijo en `.openclaw/attachments/<uuid>/` con un `.manifest.json`.
- El contenido de los adjuntos se redacciona automáticamente de la persistencia de la transcripción.
- Las entradas base64 se validan con comprobaciones estrictas de alfabeto/relleno y una protección de tamaño antes de la decodificación.
- Los permisos de archivos son `0700` para directorios y `0600` para archivos.
- La limpieza sigue la política `cleanup`: `delete` siempre elimina adjuntos; `keep` los conserva solo cuando `retainOnSessionKeep: true`.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: modelo predeterminado para subagentes generados. Si se omite, los subagentes heredan el modelo del llamador.
- `allowAgents`: lista de permitidos predeterminada de ID de agente de destino para `sessions_spawn` cuando el agente solicitante no configura su propio `subagents.allowAgents` (`["*"]` = cualquiera; predeterminado: solo el mismo agente).
- `runTimeoutSeconds`: tiempo de espera predeterminado (segundos) para `sessions_spawn` cuando la llamada a la herramienta omite `runTimeoutSeconds`. `0` significa sin tiempo de espera.
- Política de herramientas por subagente: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Proveedores personalizados y base URLs

OpenClaw usa el catálogo de modelos integrado. Añade proveedores personalizados mediante `models.providers` en la configuración o `~/.openclaw/agents/<agentId>/agent/models.json`.

```json5
{
  models: {
    mode: "merge", // merge (default) | replace
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

- Usa `authHeader: true` + `headers` para necesidades de autenticación personalizadas.
- Sobrescribe la raíz de configuración del agente con `OPENCLAW_AGENT_DIR` (o `PI_CODING_AGENT_DIR`, un alias heredado de variable de entorno).
- Precedencia de combinación para ID de proveedor coincidentes:
  - Los valores `baseUrl` no vacíos de `models.json` del agente tienen prioridad.
  - Los valores `apiKey` no vacíos del agente tienen prioridad solo cuando ese proveedor no está gestionado por SecretRef en el contexto actual de config/auth-profile.
  - Los valores `apiKey` de proveedores gestionados por SecretRef se actualizan desde marcadores de fuente (`ENV_VAR_NAME` para refs de entorno, `secretref-managed` para refs file/exec) en lugar de persistir secretos resueltos.
  - Los valores de cabeceras de proveedores gestionados por SecretRef se actualizan desde marcadores de fuente (`secretref-env:ENV_VAR_NAME` para refs de entorno, `secretref-managed` para refs file/exec).
  - Los valores `apiKey`/`baseUrl` vacíos o ausentes del agente usan como respaldo `models.providers` en la configuración.
  - `contextWindow`/`maxTokens` del modelo coincidente usan el valor mayor entre la configuración explícita y los valores implícitos del catálogo.
  - `contextTokens` del modelo coincidente conserva un límite explícito del entorno de ejecución cuando existe; úsalo para limitar el contexto efectivo sin cambiar los metadatos nativos del modelo.
  - Usa `models.mode: "replace"` cuando quieras que la configuración reescriba completamente `models.json`.
  - La persistencia de marcadores es autoritativa por fuente: los marcadores se escriben desde la instantánea activa de la configuración fuente (antes de la resolución), no desde los valores secretos resueltos del entorno de ejecución.

### Detalles de campos del proveedor

- `models.mode`: comportamiento del catálogo de proveedores (`merge` o `replace`).
- `models.providers`: mapa de proveedores personalizados con clave por ID de proveedor.
- `models.providers.*.api`: adaptador de solicitud (`openai-completions`, `openai-responses`, `anthropic-messages`, `google-generative-ai`, etc.).
- `models.providers.*.apiKey`: credencial del proveedor (prefiere SecretRef/sustitución por entorno).
- `models.providers.*.auth`: estrategia de autenticación (`api-key`, `token`, `oauth`, `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: para Ollama + `openai-completions`, inyecta `options.num_ctx` en las solicitudes (predeterminado: `true`).
- `models.providers.*.authHeader`: fuerza el transporte de credenciales en la cabecera `Authorization` cuando sea necesario.
- `models.providers.*.baseUrl`: URL base de la API upstream.
- `models.providers.*.headers`: cabeceras estáticas extra para enrutamiento de proxy/inquilino.
- `models.providers.*.request`: anulaciones de transporte para solicitudes HTTP de model-provider.
  - `request.headers`: cabeceras extra (combinadas con los valores predeterminados del proveedor). Los valores aceptan SecretRef.
  - `request.auth`: anulación de estrategia de autenticación. Modos: `"provider-default"` (usar la autenticación integrada del proveedor), `"authorization-bearer"` (con `token`), `"header"` (con `headerName`, `value`, `prefix` opcional).
  - `request.proxy`: anulación de proxy HTTP. Modos: `"env-proxy"` (usar variables de entorno `HTTP_PROXY`/`HTTPS_PROXY`), `"explicit-proxy"` (con `url`). Ambos modos aceptan un subobjeto `tls` opcional.
  - `request.tls`: anulación TLS para conexiones directas. Campos: `ca`, `cert`, `key`, `passphrase` (todos aceptan SecretRef), `serverName`, `insecureSkipVerify`.
- `models.providers.*.models`: entradas explícitas del catálogo de modelos del proveedor.
- `models.providers.*.models.*.contextWindow`: metadatos de ventana de contexto nativa del modelo.
- `models.providers.*.models.*.contextTokens`: límite opcional de contexto del entorno de ejecución. Úsalo cuando quieras un presupuesto de contexto efectivo menor que el `contextWindow` nativo del modelo.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: pista opcional de compatibilidad. Para `api: "openai-completions"` con un `baseUrl` no nativo y no vacío (host distinto de `api.openai.com`), OpenClaw fuerza esto a `false` en tiempo de ejecución. Un `baseUrl` vacío/omitido mantiene el comportamiento predeterminado de OpenAI.
- `plugins.entries.amazon-bedrock.config.discovery`: raíz de la configuración de autodetección de Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: activa/desactiva la detección implícita.
- `plugins.entries.amazon-bedrock.config.discovery.region`: región AWS para detección.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: filtro opcional por ID de proveedor para detección dirigida.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: intervalo de sondeo para actualizar la detección.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: ventana de contexto de respaldo para modelos detectados.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: máximo de tokens de salida de respaldo para modelos detectados.

### Ejemplos de proveedores

<Accordion title="Cerebras (GLM 4.6 / 4.7)">

```json5
{
  env: { CEREBRAS_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: {
        primary: "cerebras/zai-glm-4.7",
        fallbacks: ["cerebras/zai-glm-4.6"],
      },
      models: {
        "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
        "cerebras/zai-glm-4.6": { alias: "GLM 4.6 (Cerebras)" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
          { id: "zai-glm-4.6", name: "GLM 4.6 (Cerebras)" },
        ],
      },
    },
  },
}
```

Usa `cerebras/zai-glm-4.7` para Cerebras; `zai/glm-4.7` para Z.AI directo.

</Accordion>

<Accordion title="OpenCode">

```json5
{
  agents: {
    defaults: {
      model: { primary: "opencode/claude-opus-4-6" },
      models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
    },
  },
}
```

Configura `OPENCODE_API_KEY` (o `OPENCODE_ZEN_API_KEY`). Usa referencias `opencode/...` para el catálogo Zen o `opencode-go/...` para el catálogo Go. Atajo: `openclaw onboard --auth-choice opencode-zen` o `openclaw onboard --auth-choice opencode-go`.

</Accordion>

<Accordion title="Z.AI (GLM-4.7)">

```json5
{
  agents: {
    defaults: {
      model: { primary: "zai/glm-4.7" },
      models: { "zai/glm-4.7": {} },
    },
  },
}
```

Configura `ZAI_API_KEY`. `z.ai/*` y `z-ai/*` se aceptan como alias. Atajo: `openclaw onboard --auth-choice zai-api-key`.

- Endpoint general: `https://api.z.ai/api/paas/v4`
- Endpoint de coding (predeterminado): `https://api.z.ai/api/coding/paas/v4`
- Para el endpoint general, define un proveedor personalizado con la anulación de base URL.

</Accordion>

<Accordion title="Moonshot AI (Kimi)">

```json5
{
  env: { MOONSHOT_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "moonshot/kimi-k2.5" },
      models: { "moonshot/kimi-k2.5": { alias: "Kimi K2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "kimi-k2.5",
            name: "Kimi K2.5",
            reasoning: false,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
        ],
      },
    },
  },
}
```

Para el endpoint de China: `baseUrl: "https://api.moonshot.cn/v1"` o `openclaw onboard --auth-choice moonshot-api-key-cn`.

Los endpoints nativos de Moonshot anuncian compatibilidad de uso de streaming en el transporte compartido
`openai-completions`, y OpenClaw ahora basa eso en las capacidades del endpoint
en lugar de solo en el ID integrado del proveedor.

</Accordion>

<Accordion title="Kimi Coding">

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "kimi/kimi-code" },
      models: { "kimi/kimi-code": { alias: "Kimi Code" } },
    },
  },
}
```

Compatible con Anthropic, proveedor integrado. Atajo: `openclaw onboard --auth-choice kimi-code-api-key`.

</Accordion>

<Accordion title="Synthetic (Anthropic-compatible)">

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M2.5": { alias: "MiniMax M2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M2.5",
            name: "MiniMax M2.5",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 192000,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

La base URL debe omitir `/v1` (el cliente Anthropic la añade). Atajo: `openclaw onboard --auth-choice synthetic-api-key`.

</Accordion>

<Accordion title="MiniMax M2.7 (direct)">

```json5
{
  agents: {
    defaults: {
      model: { primary: "minimax/MiniMax-M2.7" },
      models: {
        "minimax/MiniMax-M2.7": { alias: "Minimax" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Configura `MINIMAX_API_KEY`. Atajos:
`openclaw onboard --auth-choice minimax-global-api` o
`openclaw onboard --auth-choice minimax-cn-api`.
El catálogo de modelos ahora usa por defecto solo M2.7.
En la ruta de streaming compatible con Anthropic, OpenClaw deshabilita thinking de MiniMax
de forma predeterminada salvo que configures `thinking` explícitamente tú mismo. `/fast on` o
`params.fastMode: true` reescribe `MiniMax-M2.7` a
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="Local models (LM Studio)">

Consulta [Modelos locales](/gateway/local-models). Resumen: ejecuta un gran modelo local mediante la API Responses de LM Studio en hardware serio; mantén modelos alojados combinados para respaldo.

</Accordion>

---

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // or plaintext string
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: lista de permitidos opcional solo para Skills integradas (las Skills gestionadas/del espacio de trabajo no se ven afectadas).
- `load.extraDirs`: raíces compartidas extra de Skills (precedencia más baja).
- `install.preferBrew`: cuando es true, prefiere instaladores Homebrew cuando `brew` está
  disponible antes de recurrir a otros tipos de instalador.
- `install.nodeManager`: preferencia de instalador Node para especificaciones
  `metadata.openclaw.install` (`npm` | `pnpm` | `yarn` | `bun`).
- `entries.<skillKey>.enabled: false` deshabilita una Skill incluso si está integrada/instalada.
- `entries.<skillKey>.apiKey`: campo de conveniencia para Skills que declaran una variable env principal (cadena en texto plano u objeto SecretRef).

---

## Plugins

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-extension"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- Se cargan desde `~/.openclaw/extensions`, `<workspace>/.openclaw/extensions`, además de `plugins.load.paths`.
- El descubrimiento acepta plugins nativos de OpenClaw además de bundles compatibles de Codex y Claude, incluidos bundles de Claude sin manifest con diseño predeterminado.
- **Los cambios de configuración requieren reiniciar el gateway.**
- `allow`: lista de permitidos opcional (solo cargan los plugins listados). `deny` tiene prioridad.
- `plugins.entries.<id>.apiKey`: campo de conveniencia de clave de API a nivel de plugin (cuando el plugin lo admite).
- `plugins.entries.<id>.env`: mapa de variables de entorno con alcance del plugin.
- `plugins.entries.<id>.hooks.allowPromptInjection`: cuando es `false`, el núcleo bloquea `before_prompt_build` e ignora campos mutadores de prompt de `before_agent_start` heredado, preservando `modelOverride` y `providerOverride` heredados. Se aplica a hooks de plugins nativos y a directorios de hooks aportados por bundles compatibles.
- `plugins.entries.<id>.subagent.allowModelOverride`: confía explícitamente en este plugin para solicitar anulaciones por ejecución de `provider` y `model` para ejecuciones de subagentes en segundo plano.
- `plugins.entries.<id>.subagent.allowedModels`: lista de permitidos opcional de destinos canónicos `provider/model` para anulaciones confiables de subagentes. Usa `"*"` solo cuando quieras intencionalmente permitir cualquier modelo.
- `plugins.entries.<id>.config`: objeto de configuración definido por el plugin (validado por el esquema nativo del plugin de OpenClaw cuando está disponible).
- `plugins.entries.firecrawl.config.webFetch`: ajustes del proveedor web-fetch Firecrawl.
  - `apiKey`: clave de API de Firecrawl (acepta SecretRef). Usa como respaldo `plugins.entries.firecrawl.config.webSearch.apiKey`, el heredado `tools.web.fetch.firecrawl.apiKey` o la variable de entorno `FIRECRAWL_API_KEY`.
  - `baseUrl`: URL base de la API de Firecrawl (predeterminado: `https://api.firecrawl.dev`).
  - `onlyMainContent`: extraer solo el contenido principal de las páginas (predeterminado: `true`).
  - `maxAgeMs`: antigüedad máxima de caché en milisegundos (predeterminado: `172800000` / 2 días).
  - `timeoutSeconds`: tiempo de espera de la solicitud scrape en segundos (predeterminado: `60`).
- `plugins.entries.xai.config.xSearch`: ajustes de xAI X Search (búsqueda web de Grok).
  - `enabled`: habilita el proveedor X Search.
  - `model`: modelo Grok que se usará para búsqueda (por ejemplo `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming`: ajustes de memory dreaming (experimental). Consulta [Dreaming](/concepts/memory-dreaming) para modos y umbrales.
  - `mode`: preajuste de cadencia de dreaming (`"off"`, `"core"`, `"rem"`, `"deep"`). Predeterminado: `"off"`.
  - `cron`: anulación opcional de la expresión cron para la programación de dreaming.
  - `timezone`: zona horaria para evaluar la programación (usa como respaldo `agents.defaults.userTimezone`).
  - `limit`: máximo de candidatos a promover por ciclo.
  - `minScore`: umbral mínimo de puntuación ponderada para la promoción.
  - `minRecallCount`: umbral mínimo de cantidad de recalls.
  - `minUniqueQueries`: umbral mínimo de cantidad de consultas distintas.
- Los plugins Claude bundle habilitados también pueden aportar valores predeterminados Pi integrados desde `settings.json`; OpenClaw los aplica como ajustes sanitizados del agente, no como parches sin procesar de configuración de OpenClaw.
- `plugins.slots.memory`: elige el ID del plugin de memoria activo o `"none"` para deshabilitar plugins de memoria.
- `plugins.slots.contextEngine`: elige el ID del motor de contexto activo; usa `"legacy"` de forma predeterminada salvo que instales y selecciones otro motor.
- `plugins.installs`: metadatos de instalación gestionados por CLI usados por `openclaw plugins update`.
  - Incluye `source`, `spec`, `sourcePath`, `installPath`, `version`, `resolvedName`, `resolvedVersion`, `resolvedSpec`, `integrity`, `shasum`, `resolvedAt`, `installedAt`.
  - Trata `plugins.installs.*` como estado gestionado; prefiere comandos CLI a ediciones manuales.

Consulta [Plugins](/tools/plugin).

---

## Browser

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // default trusted-network mode
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false` deshabilita `act:evaluate` y `wait --fn`.
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` usa `true` de forma predeterminada cuando no está configurado (modelo de red de confianza).
- Configura `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` para una navegación estricta del browser solo en redes públicas.
- En modo estricto, los endpoints de perfiles CDP remotos (`profiles.*.cdpUrl`) están sujetos al mismo bloqueo de red privada durante las comprobaciones de alcance/detección.
- `ssrfPolicy.allowPrivateNetwork` sigue siendo compatible como alias heredado.
- En modo estricto, usa `ssrfPolicy.hostnameAllowlist` y `ssrfPolicy.allowedHostnames` para excepciones explícitas.
- Los perfiles remotos son solo de conexión (start/stop/reset deshabilitados).
- `profiles.*.cdpUrl` acepta `http://`, `https://`, `ws://` y `wss://`.
  Usa HTTP(S) cuando quieras que OpenClaw descubra `/json/version`; usa WS(S)
  cuando tu proveedor te dé una URL directa de DevTools WebSocket.
- Los perfiles `existing-session` son solo del host y usan Chrome MCP en lugar de CDP.
- Los perfiles `existing-session` pueden configurar `userDataDir` para apuntar a un perfil específico
  de navegador basado en Chromium como Brave o Edge.
- Los perfiles `existing-session` mantienen los límites actuales de la ruta Chrome MCP:
  acciones basadas en snapshot/ref en lugar de selección por CSS, hooks de carga de un único archivo,
  sin anulaciones de tiempo de espera de diálogos, sin `wait --load networkidle`, y sin
  `responsebody`, exportación PDF, interceptación de descargas ni acciones por lotes.
- Los perfiles locales gestionados `openclaw` asignan automáticamente `cdpPort` y `cdpUrl`; solo
  configura `cdpUrl` explícitamente para CDP remoto.
- Orden de autodetección: navegador predeterminado si está basado en Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- Servicio de control: solo loopback (puerto derivado de `gateway.port`, predeterminado `18791`).
- `extraArgs` añade flags extra de lanzamiento al inicio local de Chromium (por ejemplo
  `--disable-gpu`, tamaño de ventana o flags de depuración).

---

## UI

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // emoji, short text, image URL, or data URI
    },
  },
}
```

- `seamColor`: color de acento para la interfaz nativa (tono de la burbuja en Talk Mode, etc.).
- `assistant`: anulación de identidad de la Control UI. Usa como respaldo la identidad del agente activo.

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // or OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // for mode=trusted-proxy; see /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // allowedOrigins: ["https://control.example.com"], // required for non-loopback Control UI
      // dangerouslyAllowHostHeaderOriginFallback: false, // dangerous Host-header origin fallback mode
      // allowInsecureAuth: false,
      // dangerouslyDisableDeviceAuth: false,
    },
    remote: {
      url: "ws://gateway.tailnet:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // Optional. Default false.
    allowRealIpFallback: false,
    tools: {
      // Additional /tools/invoke HTTP denies
      deny: ["browser"],
      // Remove tools from the default HTTP deny list
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Gateway field details">

- `mode`: `local` (ejecutar gateway) o `remote` (conectarse a un gateway remoto). El gateway se niega a iniciarse salvo que sea `local`.
- `port`: puerto multiplexado único para WS + HTTP. Precedencia: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: `auto`, `loopback` (predeterminado), `lan` (`0.0.0.0`), `tailnet` (solo IP Tailscale) o `custom`.
- **Alias heredados de bind**: usa valores de modo bind en `gateway.bind` (`auto`, `loopback`, `lan`, `tailnet`, `custom`), no alias de host (`0.0.0.0`, `127.0.0.1`, `localhost`, `::`, `::1`).
- **Nota sobre Docker**: el bind predeterminado `loopback` escucha en `127.0.0.1` dentro del contenedor. Con redes bridge de Docker (`-p 18789:18789`), el tráfico llega por `eth0`, así que el gateway no es accesible. Usa `--network host`, o configura `bind: "lan"` (o `bind: "custom"` con `customBindHost: "0.0.0.0"`) para escuchar en todas las interfaces.
- **Auth**: obligatoria por defecto. Los binds que no son loopback requieren autenticación de gateway. En la práctica eso significa un token/contraseña compartido o un proxy inverso con reconocimiento de identidad con `gateway.auth.mode: "trusted-proxy"`. El asistente de onboarding genera un token por defecto.
- Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados (incluidos SecretRefs), configura `gateway.auth.mode` explícitamente en `token` o `password`. El inicio y los flujos de instalación/reparación del servicio fallan cuando ambos están configurados y `mode` no está establecido.
- `gateway.auth.mode: "none"`: modo explícito sin autenticación. Úsalo solo para configuraciones de loopback local de confianza; intencionalmente no se ofrece en los prompts de onboarding.
- `gateway.auth.mode: "trusted-proxy"`: delega la autenticación a un proxy inverso con reconocimiento de identidad y confía en cabeceras de identidad de `gateway.trustedProxies` (consulta [Autenticación de proxy de confianza](/gateway/trusted-proxy-auth)). Este modo espera una fuente de proxy **no loopback**; los proxies inversos loopback en el mismo host no cumplen la autenticación trusted-proxy.
- `gateway.auth.allowTailscale`: cuando es `true`, las cabeceras de identidad de Tailscale Serve pueden satisfacer la autenticación de Control UI/WebSocket (verificadas mediante `tailscale whois`). Los endpoints de la API HTTP **no** usan esa autenticación de cabeceras Tailscale; siguen el modo normal de autenticación HTTP del gateway. Este flujo sin token asume que el host del gateway es de confianza. Usa `true` de forma predeterminada cuando `tailscale.mode = "serve"`.
- `gateway.auth.rateLimit`: limitador opcional de autenticaciones fallidas. Se aplica por IP cliente y por alcance de autenticación (secreto compartido y device-token se rastrean de forma independiente). Los intentos bloqueados devuelven `429` + `Retry-After`.
  - En la ruta asíncrona de Control UI de Tailscale Serve, los intentos fallidos para el mismo `{scope, clientIp}` se serializan antes de la escritura del fallo. Por tanto, intentos simultáneos incorrectos del mismo cliente pueden activar el limitador en la segunda solicitud en lugar de dejar pasar ambas como simples desajustes.
  - `gateway.auth.rateLimit.exemptLoopback` usa `true` de forma predeterminada; configura `false` cuando quieras intencionalmente aplicar rate limit también al tráfico localhost (para entornos de prueba o despliegues de proxy estrictos).
- Los intentos de autenticación WS originados en browser siempre se limitan con la exención de loopback deshabilitada (defensa adicional contra fuerza bruta localhost basada en browser).
- En loopback, esos bloqueos originados en localhost por browser se aíslan por valor `Origin`
  normalizado, de modo que fallos repetidos desde un origen localhost no bloquean
  automáticamente otro origen distinto.
- `tailscale.mode`: `serve` (solo tailnet, bind loopback) o `funnel` (público, requiere auth).
- `controlUi.allowedOrigins`: lista explícita de orígenes de browser permitidos para conexiones WebSocket del Gateway. Obligatoria cuando se esperan clientes browser desde orígenes no loopback.
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: modo peligroso que habilita el respaldo de origen por cabecera Host para despliegues que dependen intencionalmente de esa política.
- `remote.transport`: `ssh` (predeterminado) o `direct` (ws/wss). Para `direct`, `remote.url` debe ser `ws://` o `wss://`.
- `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`: anulación del lado cliente de emergencia que permite `ws://` en texto claro hacia IP privadas de confianza; el valor predeterminado sigue siendo permitir texto claro solo para loopback.
- `gateway.remote.token` / `.password` son campos de credenciales del cliente remoto. No configuran por sí solos la autenticación del gateway.
- `gateway.push.apns.relay.baseUrl`: URL base HTTPS del relay APNs externo usado por compilaciones oficiales/TestFlight de iOS después de que publiquen registros respaldados por relay en el gateway. Esta URL debe coincidir con la URL del relay compilada en la build de iOS.
- `gateway.push.apns.relay.timeoutMs`: tiempo de espera en milisegundos para envíos del gateway al relay. Predeterminado: `10000`.
- Los registros respaldados por relay se delegan a una identidad específica del gateway. La app iOS emparejada obtiene `gateway.identity.get`, incluye esa identidad en el registro del relay y reenvía una concesión de envío con alcance de registro al gateway. Otro gateway no puede reutilizar ese registro almacenado.
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: anulaciones temporales por entorno para la configuración del relay anterior.
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: vía de escape solo para desarrollo para URL de relay HTTP loopback. Las URL de relay de producción deben seguir usando HTTPS.
- `gateway.channelHealthCheckMinutes`: intervalo en minutos del monitor de salud de canales. Configura `0` para desactivar globalmente los reinicios del monitor de salud. Predeterminado: `5`.
- `gateway.channelStaleEventThresholdMinutes`: umbral en minutos para sockets obsoletos. Mantenlo mayor o igual que `gateway.channelHealthCheckMinutes`. Predeterminado: `30`.
- `gateway.channelMaxRestartsPerHour`: máximo de reinicios por canal/cuenta dentro de una hora móvil. Predeterminado: `10`.
- `channels.<provider>.healthMonitor.enabled`: exclusión por canal de los reinicios del monitor de salud mientras el monitor global sigue habilitado.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: anulación por cuenta para canales de múltiples cuentas. Cuando está configurado, tiene prioridad sobre la anulación a nivel de canal.
- Las rutas locales de llamada al gateway pueden usar `gateway.remote.*` como respaldo solo cuando `gateway.auth.*` no está configurado.
- Si `gateway.auth.token` / `gateway.auth.password` está configurado explícitamente mediante SecretRef y no se resuelve, la resolución falla en modo cerrado (sin respaldo remoto que enmascare el fallo).
- `trustedProxies`: IP de proxies inversos que terminan TLS o inyectan cabeceras de cliente reenviado. Enumera solo proxies que controlas. Las entradas loopback siguen siendo válidas para configuraciones de mismo host/detección local (por ejemplo Tailscale Serve o un proxy inverso local), pero **no** hacen que las solicitudes loopback sean elegibles para `gateway.auth.mode: "trusted-proxy"`.
- `allowRealIpFallback`: cuando es `true`, el gateway acepta `X-Real-IP` si falta `X-Forwarded-For`. Predeterminado `false` para comportamiento cerrado por defecto.
- `gateway.tools.deny`: nombres extra de herramientas bloqueadas para HTTP `POST /tools/invoke` (extiende la lista predeterminada de denegación).
- `gateway.tools.allow`: elimina nombres de herramientas de la lista predeterminada de denegación HTTP.

</Accordion>

### Endpoints compatibles con OpenAI

- Chat Completions: deshabilitado de forma predeterminada. Habilítalo con `gateway.http.endpoints.chatCompletions.enabled: true`.
- Responses API: `gateway.http.endpoints.responses.enabled`.
- Endurecimiento de entradas URL en Responses:
  - `