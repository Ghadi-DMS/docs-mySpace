---
read_when:
    - Necesitas la semántica exacta o los valores predeterminados de la configuración a nivel de campo
    - Estás validando bloques de configuración de canales, modelos, gateway o herramientas
summary: Referencia de configuración del Gateway para las claves principales de OpenClaw, los valores predeterminados y enlaces a referencias dedicadas de subsistemas
title: Referencia de configuración
x-i18n:
    generated_at: "2026-04-09T01:35:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: a9d6d0c542b9874809491978fdcf8e1a7bb35a4873db56aa797963d03af4453c
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# Referencia de configuración

Referencia de configuración principal para `~/.openclaw/openclaw.json`. Para una visión general orientada a tareas, consulta [Configuration](/es/gateway/configuration).

Esta página cubre las principales superficies de configuración de OpenClaw y enlaza a otras páginas cuando un subsistema tiene su propia referencia más profunda. **No** intenta incluir en una sola página cada catálogo de comandos propio de canales/plugins ni cada ajuste profundo de memoria/QMD.

Fuente de verdad del código:

- `openclaw config schema` imprime el JSON Schema activo usado para validación y Control UI, con los metadatos fusionados de bundles/plugins/canales cuando están disponibles
- `config.schema.lookup` devuelve un nodo del esquema limitado a una ruta para herramientas de exploración detallada
- `pnpm config:docs:check` / `pnpm config:docs:gen` validan el hash base de la documentación de configuración frente a la superficie actual del esquema

Referencias profundas dedicadas:

- [Memory configuration reference](/es/reference/memory-config) para `agents.defaults.memorySearch.*`, `memory.qmd.*`, `memory.citations` y la configuración de dreaming en `plugins.entries.memory-core.config.dreaming`
- [Slash Commands](/es/tools/slash-commands) para el catálogo actual de comandos integrados + agrupados
- páginas del canal/plugin propietario para superficies de comandos específicas del canal

El formato de configuración es **JSON5** (se permiten comentarios + comas finales). Todos los campos son opcionales: OpenClaw usa valores predeterminados seguros cuando se omiten.

---

## Canales

Cada canal se inicia automáticamente cuando existe su sección de configuración (a menos que `enabled: false`).

### Acceso a mensajes directos y grupos

Todos los canales admiten políticas de MD y políticas de grupo:

| Política de MD     | Comportamiento                                                  |
| ------------------ | --------------------------------------------------------------- |
| `pairing` (predeterminada) | Los remitentes desconocidos reciben un código de emparejamiento de un solo uso; el propietario debe aprobarlo |
| `allowlist`        | Solo remitentes en `allowFrom` (o en el almacén de permisos emparejado) |
| `open`             | Permitir todos los MD entrantes (requiere `allowFrom: ["*"]`)   |
| `disabled`         | Ignorar todos los MD entrantes                                  |

| Política de grupo    | Comportamiento                                         |
| -------------------- | ------------------------------------------------------ |
| `allowlist` (predeterminada) | Solo grupos que coinciden con la lista de permisos configurada |
| `open`               | Omite las listas de permisos de grupo (la exigencia de mención sigue aplicándose) |
| `disabled`           | Bloquea todos los mensajes de grupo/sala               |

<Note>
`channels.defaults.groupPolicy` establece el valor predeterminado cuando `groupPolicy` de un proveedor no está configurado.
Los códigos de emparejamiento caducan después de 1 hora. Las solicitudes pendientes de emparejamiento de MD están limitadas a **3 por canal**.
Si falta por completo un bloque de proveedor (`channels.<provider>` ausente), la política de grupo en tiempo de ejecución vuelve a `allowlist` (cierre por defecto) con una advertencia al inicio.
</Note>

### Sustituciones de modelo por canal

Usa `channels.modelByChannel` para fijar IDs de canal específicos a un modelo. Los valores aceptan `provider/model` o alias de modelo configurados. La asignación del canal se aplica cuando una sesión no tiene ya una sustitución de modelo (por ejemplo, establecida mediante `/model`).

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

- `channels.defaults.groupPolicy`: política de grupo de respaldo cuando no se configura un `groupPolicy` a nivel de proveedor.
- `channels.defaults.contextVisibility`: modo predeterminado de visibilidad de contexto suplementario para todos los canales. Valores: `all` (predeterminado, incluir todo el contexto citado/de hilo/historial), `allowlist` (incluir solo contexto de remitentes permitidos), `allowlist_quote` (igual que allowlist pero conservando el contexto explícito de cita/respuesta). Sustitución por canal: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: incluir estados de canal saludables en la salida de heartbeat.
- `channels.defaults.heartbeat.showAlerts`: incluir estados degradados/con error en la salida de heartbeat.
- `channels.defaults.heartbeat.useIndicator`: mostrar una salida de heartbeat compacta con estilo de indicador.

### WhatsApp

WhatsApp se ejecuta a través del canal web del gateway (Baileys Web). Se inicia automáticamente cuando existe una sesión vinculada.

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

<Accordion title="WhatsApp de múltiples cuentas">

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

- Los comandos salientes usan la cuenta `default` por defecto si está presente; de lo contrario, usan el primer ID de cuenta configurado (ordenado).
- `channels.whatsapp.defaultAccount` opcional sustituye esa selección de cuenta predeterminada de respaldo cuando coincide con un ID de cuenta configurado.
- El directorio heredado de autenticación Baileys de una sola cuenta es migrado por `openclaw doctor` a `whatsapp/default`.
- Sustituciones por cuenta: `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

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
      replyToMode: "first", // off | first | all | batched
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

- Token del bot: `channels.telegram.botToken` o `channels.telegram.tokenFile` (solo archivo regular; se rechazan symlinks), con `TELEGRAM_BOT_TOKEN` como respaldo para la cuenta predeterminada.
- `channels.telegram.defaultAccount` opcional sustituye la selección de cuenta predeterminada cuando coincide con un ID de cuenta configurado.
- En configuraciones de múltiples cuentas (2+ IDs de cuenta), establece una cuenta predeterminada explícita (`channels.telegram.defaultAccount` o `channels.telegram.accounts.default`) para evitar el enrutamiento por respaldo; `openclaw doctor` advierte cuando esto falta o es inválido.
- `configWrites: false` bloquea las escrituras de configuración iniciadas por Telegram (migraciones de ID de supergrupo, `/config set|unset`).
- Las entradas `bindings[]` de nivel superior con `type: "acp"` configuran enlaces ACP persistentes para temas de foro (usa el formato canónico `chatId:topic:topicId` en `match.peer.id`). La semántica de los campos se comparte en [ACP Agents](/es/tools/acp-agents#channel-specific-settings).
- Las vistas previas de stream de Telegram usan `sendMessage` + `editMessageText` (funciona en chats directos y de grupo).
- Política de reintento: consulta [Retry policy](/es/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
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
      replyToMode: "off", // off | first | all | batched
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
- Las llamadas salientes directas que proporcionan un `token` explícito de Discord usan ese token para la llamada; la configuración de reintento/política de cuenta sigue viniendo de la cuenta seleccionada en la instantánea activa en tiempo de ejecución.
- `channels.discord.defaultAccount` opcional sustituye la selección de cuenta predeterminada cuando coincide con un ID de cuenta configurado.
- Usa `user:<id>` (MD) o `channel:<id>` (canal de guild) para los destinos de entrega; los IDs numéricos sin prefijo se rechazan.
- Los slugs de guild van en minúsculas y con espacios reemplazados por `-`; las claves de canal usan el nombre convertido en slug (sin `#`). Prefiere los IDs de guild.
- Los mensajes creados por bots se ignoran de forma predeterminada. `allowBots: true` los habilita; usa `allowBots: "mentions"` para aceptar solo mensajes de bots que mencionen al bot (los mensajes propios siguen filtrados).
- `channels.discord.guilds.<id>.ignoreOtherMentions` (y las sustituciones por canal) descarta mensajes que mencionan a otro usuario o rol, pero no al bot (excepto @everyone/@here).
- `maxLinesPerMessage` (predeterminado 17) divide mensajes altos incluso cuando están por debajo de 2000 caracteres.
- `channels.discord.threadBindings` controla el enrutamiento ligado a hilos de Discord:
  - `enabled`: sustitución de Discord para las funciones de sesión ligadas a hilos (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` y entrega/enrutamiento ligados)
  - `idleHours`: sustitución de Discord para el auto-unfocus por inactividad en horas (`0` desactiva)
  - `maxAgeHours`: sustitución de Discord para la antigüedad máxima estricta en horas (`0` desactiva)
  - `spawnSubagentSessions`: interruptor opt-in para creación/enlace automático de hilos con `sessions_spawn({ thread: true })`
- Las entradas `bindings[]` de nivel superior con `type: "acp"` configuran enlaces ACP persistentes para canales e hilos (usa el ID de canal/hilo en `match.peer.id`). La semántica de los campos se comparte en [ACP Agents](/es/tools/acp-agents#channel-specific-settings).
- `channels.discord.ui.components.accentColor` establece el color de acento para los contenedores de componentes v2 de Discord.
- `channels.discord.voice` habilita conversaciones en canales de voz de Discord y sustituciones opcionales de auto-join + TTS.
- `channels.discord.voice.daveEncryption` y `channels.discord.voice.decryptionFailureTolerance` se pasan a las opciones DAVE de `@discordjs/voice` (`true` y `24` por defecto).
- OpenClaw además intenta recuperar la recepción de voz saliendo y volviendo a unirse a una sesión de voz después de fallos repetidos de descifrado.
- `channels.discord.streaming` es la clave canónica del modo de stream. Los valores heredados `streamMode` y booleanos `streaming` se migran automáticamente.
- `channels.discord.autoPresence` asigna la disponibilidad en tiempo de ejecución a la presencia del bot (saludable => online, degradado => idle, agotado => dnd) y permite sustituciones opcionales del texto de estado.
- `channels.discord.dangerouslyAllowNameMatching` vuelve a habilitar la coincidencia con nombres/tags mutables (modo de compatibilidad de emergencia).
- `channels.discord.execApprovals`: entrega de aprobaciones exec nativa de Discord y autorización de aprobadores.
  - `enabled`: `true`, `false` o `"auto"` (predeterminado). En modo auto, las aprobaciones exec se activan cuando los aprobadores pueden resolverse desde `approvers` o `commands.ownerAllowFrom`.
  - `approvers`: IDs de usuario de Discord autorizados para aprobar solicitudes exec. Usa `commands.ownerAllowFrom` como respaldo si se omite.
  - `agentFilter`: lista opcional de IDs de agente permitidos. Omítela para reenviar aprobaciones para todos los agentes.
  - `sessionFilter`: patrones opcionales de claves de sesión (subcadena o regex).
  - `target`: dónde enviar las solicitudes de aprobación. `"dm"` (predeterminado) las envía a los MD del aprobador, `"channel"` las envía al canal de origen, `"both"` las envía a ambos. Cuando el destino incluye `"channel"`, los botones solo pueden ser usados por aprobadores resueltos.
  - `cleanupAfterResolve`: cuando es `true`, elimina los MD de aprobación después de aprobar, denegar o agotar el tiempo.

**Modos de notificación de reacciones:** `off` (ninguno), `own` (mensajes del bot, predeterminado), `all` (todos los mensajes), `allowlist` (de `guilds.<id>.users` en todos los mensajes).

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
- Respaldos por entorno: `GOOGLE_CHAT_SERVICE_ACCOUNT` o `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- Usa `spaces/<spaceId>` o `users/<userId>` para los destinos de entrega.
- `channels.googlechat.dangerouslyAllowNameMatching` vuelve a habilitar la coincidencia con principal de correo mutable (modo de compatibilidad de emergencia).

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
      replyToMode: "off", // off | first | all | batched
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
      streaming: {
        mode: "partial", // off | partial | block | progress
        nativeTransport: true, // use Slack native streaming API when mode=partial
      },
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

- **Modo socket** requiere tanto `botToken` como `appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` como respaldo de entorno para la cuenta predeterminada).
- **Modo HTTP** requiere `botToken` más `signingSecret` (en la raíz o por cuenta).
- `botToken`, `appToken`, `signingSecret` y `userToken` aceptan
  cadenas en texto plano u objetos SecretRef.
- Las instantáneas de cuentas de Slack exponen campos por credencial de origen/estado como
  `botTokenSource`, `botTokenStatus`, `appTokenStatus` y, en modo HTTP,
  `signingSecretStatus`. `configured_unavailable` significa que la cuenta está
  configurada mediante SecretRef pero la ruta actual de comando/tiempo de ejecución no pudo
  resolver el valor del secreto.
- `configWrites: false` bloquea las escrituras de configuración iniciadas por Slack.
- `channels.slack.defaultAccount` opcional sustituye la selección de cuenta predeterminada cuando coincide con un ID de cuenta configurado.
- `channels.slack.streaming.mode` es la clave canónica del modo de stream de Slack. `channels.slack.streaming.nativeTransport` controla el transporte de streaming nativo de Slack. Los valores heredados `streamMode`, booleanos `streaming` y `nativeStreaming` se migran automáticamente.
- Usa `user:<id>` (MD) o `channel:<id>` para los destinos de entrega.

**Modos de notificación de reacciones:** `off`, `own` (predeterminado), `all`, `allowlist` (de `reactionAllowlist`).

**Aislamiento de sesión de hilos:** `thread.historyScope` es por hilo (predeterminado) o compartido por canal. `thread.inheritParent` copia la transcripción del canal padre a los nuevos hilos.

- El streaming nativo de Slack más el estado de hilo estilo “is typing...” del asistente de Slack requieren un destino de respuesta en hilo. Los MD de nivel superior permanecen fuera de hilo por defecto, por lo que usan `typingReaction` o entrega normal en lugar de la vista previa de estilo hilo.
- `typingReaction` agrega una reacción temporal al mensaje entrante de Slack mientras se está ejecutando una respuesta, y luego la elimina al finalizar. Usa un shortcode de emoji de Slack como `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: entrega de aprobaciones exec nativa de Slack y autorización de aprobadores. Mismo esquema que Discord: `enabled` (`true`/`false`/`"auto"`), `approvers` (IDs de usuario de Slack), `agentFilter`, `sessionFilter` y `target` (`"dm"`, `"channel"` o `"both"`).

| Grupo de acciones | Predeterminado | Notas                    |
| ----------------- | -------------- | ------------------------ |
| reactions         | habilitado     | Reaccionar + listar reacciones |
| messages          | habilitado     | Leer/enviar/editar/eliminar |
| pins              | habilitado     | Fijar/desfijar/listar    |
| memberInfo        | habilitado     | Información de miembros  |
| emojiList         | habilitado     | Lista de emoji personalizados |

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
        // URL explícita opcional para despliegues públicos/con proxy inverso
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

Modos de chat: `oncall` (responder con mención @, predeterminado), `onmessage` (cada mensaje), `onchar` (mensajes que empiezan con el prefijo disparador).

Cuando están habilitados los comandos nativos de Mattermost:

- `commands.callbackPath` debe ser una ruta (por ejemplo `/api/channels/mattermost/command`), no una URL completa.
- `commands.callbackUrl` debe resolver al endpoint del gateway de OpenClaw y ser accesible desde el servidor de Mattermost.
- Los callbacks nativos de slash se autentican con los tokens por comando devueltos
  por Mattermost durante el registro del slash command. Si el registro falla o no
  se activan comandos, OpenClaw rechaza los callbacks con
  `Unauthorized: invalid command token.`
- Para hosts de callback privados/tailnet/internos, Mattermost puede requerir
  que `ServiceSettings.AllowedUntrustedInternalConnections` incluya el host/dominio del callback.
  Usa valores de host/dominio, no URLs completas.
- `channels.mattermost.configWrites`: permitir o denegar escrituras de configuración iniciadas por Mattermost.
- `channels.mattermost.requireMention`: requerir `@mention` antes de responder en canales.
- `channels.mattermost.groups.<channelId>.requireMention`: sustitución por canal para la exigencia de mención (`"*"` para predeterminado).
- `channels.mattermost.defaultAccount` opcional sustituye la selección de cuenta predeterminada cuando coincide con un ID de cuenta configurado.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // vinculación de cuenta opcional
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

- `channels.signal.account`: fija el inicio del canal a una identidad específica de cuenta de Signal.
- `channels.signal.configWrites`: permitir o denegar escrituras de configuración iniciadas por Signal.
- `channels.signal.defaultAccount` opcional sustituye la selección de cuenta predeterminada cuando coincide con un ID de cuenta configurado.

### BlueBubbles

BlueBubbles es la ruta recomendada para iMessage (respaldada por plugin, configurada en `channels.bluebubbles`).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, controles de grupo y acciones avanzadas:
      // consulta /channels/bluebubbles
    },
  },
}
```

- Rutas clave principales cubiertas aquí: `channels.bluebubbles`, `channels.bluebubbles.dmPolicy`.
- `channels.bluebubbles.defaultAccount` opcional sustituye la selección de cuenta predeterminada cuando coincide con un ID de cuenta configurado.
- Las entradas `bindings[]` de nivel superior con `type: "acp"` pueden vincular conversaciones de BlueBubbles a sesiones ACP persistentes. Usa un identificador o cadena de destino de BlueBubbles (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) en `match.peer.id`. Semántica compartida de campos: [ACP Agents](/es/tools/acp-agents#channel-specific-settings).
- La configuración completa del canal BlueBubbles está documentada en [BlueBubbles](/es/channels/bluebubbles).

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

- `channels.imessage.defaultAccount` opcional sustituye la selección de cuenta predeterminada cuando coincide con un ID de cuenta configurado.

- Requiere Full Disk Access a la base de datos de Messages.
- Prefiere destinos `chat_id:<id>`. Usa `imsg chats --limit 20` para listar chats.
- `cliPath` puede apuntar a un wrapper SSH; configura `remoteHost` (`host` o `user@host`) para obtener adjuntos por SCP.
- `attachmentRoots` y `remoteAttachmentRoots` restringen las rutas de adjuntos entrantes (predeterminado: `/Users/*/Library/Messages/Attachments`).
- SCP usa verificación estricta de claves de host, así que asegúrate de que la clave del host de retransmisión ya exista en `~/.ssh/known_hosts`.
- `channels.imessage.configWrites`: permitir o denegar escrituras de configuración iniciadas por iMessage.
- Las entradas `bindings[]` de nivel superior con `type: "acp"` pueden vincular conversaciones de iMessage a sesiones ACP persistentes. Usa un identificador normalizado o un destino explícito de chat (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) en `match.peer.id`. Semántica compartida de campos: [ACP Agents](/es/tools/acp-agents#channel-specific-settings).

<Accordion title="Ejemplo de wrapper SSH de iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix está respaldado por extensión y se configura en `channels.matrix`.

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

- La autenticación con token usa `accessToken`; la autenticación con contraseña usa `userId` + `password`.
- `channels.matrix.proxy` enruta el tráfico HTTP de Matrix a través de un proxy HTTP(S) explícito. Las cuentas nombradas pueden sustituirlo con `channels.matrix.accounts.<id>.proxy`.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` permite homeservers privados/internos. `proxy` y esta habilitación de red son controles independientes.
- `channels.matrix.defaultAccount` selecciona la cuenta preferida en configuraciones de múltiples cuentas.
- `channels.matrix.autoJoin` usa por defecto `off`, así que las salas invitadas y las nuevas invitaciones tipo MD se ignoran hasta que configures `autoJoin: "allowlist"` con `autoJoinAllowlist` o `autoJoin: "always"`.
- `channels.matrix.execApprovals`: entrega de aprobaciones exec nativa de Matrix y autorización de aprobadores.
  - `enabled`: `true`, `false` o `"auto"` (predeterminado). En modo auto, las aprobaciones exec se activan cuando los aprobadores pueden resolverse desde `approvers` o `commands.ownerAllowFrom`.
  - `approvers`: IDs de usuario de Matrix (p. ej. `@owner:example.org`) autorizados para aprobar solicitudes exec.
  - `agentFilter`: lista opcional de IDs de agente permitidos. Omítela para reenviar aprobaciones para todos los agentes.
  - `sessionFilter`: patrones opcionales de claves de sesión (subcadena o regex).
  - `target`: dónde enviar las solicitudes de aprobación. `"dm"` (predeterminado), `"channel"` (sala de origen) o `"both"`.
  - Sustituciones por cuenta: `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope` controla cómo los MD de Matrix se agrupan en sesiones: `per-user` (predeterminado) comparte por par enrutado, mientras que `per-room` aísla cada sala de MD.
- Las sondas de estado de Matrix y las búsquedas en directorio en vivo usan la misma política de proxy que el tráfico en tiempo de ejecución.
- La configuración completa de Matrix, reglas de destino y ejemplos de configuración están documentados en [Matrix](/es/channels/matrix).

### Microsoft Teams

Microsoft Teams está respaldado por extensión y se configura en `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, políticas de equipo/canal:
      // consulta /channels/msteams
    },
  },
}
```

- Rutas clave principales cubiertas aquí: `channels.msteams`, `channels.msteams.configWrites`.
- La configuración completa de Teams (credenciales, webhook, política de MD/grupo, sustituciones por equipo/canal) está documentada en [Microsoft Teams](/es/channels/msteams).

### IRC

IRC está respaldado por extensión y se configura en `channels.irc`.

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
- `channels.irc.defaultAccount` opcional sustituye la selección de cuenta predeterminada cuando coincide con un ID de cuenta configurado.
- La configuración completa del canal IRC (host/puerto/TLS/canales/listas de permisos/exigencia de mención) está documentada en [IRC](/es/channels/irc).

### Múltiples cuentas (todos los canales)

Ejecuta varias cuentas por canal (cada una con su propio `accountId`):

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
- Los ajustes base del canal se aplican a todas las cuentas salvo que se sustituyan por cuenta.
- Usa `bindings[].match.accountId` para enrutar cada cuenta a un agente distinto.
- Si agregas una cuenta no predeterminada mediante `openclaw channels add` (o el onboarding del canal) mientras aún estás en una configuración de canal de una sola cuenta a nivel superior, OpenClaw promueve primero los valores de una sola cuenta de nivel superior con alcance de cuenta al mapa de cuentas del canal para que la cuenta original siga funcionando. La mayoría de los canales los mueve a `channels.<channel>.accounts.default`; Matrix puede conservar en su lugar un destino nombrado/default coincidente existente.
- Los enlaces existentes solo de canal (sin `accountId`) siguen coincidiendo con la cuenta predeterminada; los enlaces con alcance de cuenta siguen siendo opcionales.
- `openclaw doctor --fix` también repara formas mixtas moviendo los valores de una sola cuenta de nivel superior con alcance de cuenta a la cuenta promovida elegida para ese canal. La mayoría de los canales usa `accounts.default`; Matrix puede conservar un destino nombrado/default coincidente existente.

### Otros canales de extensión

Muchos canales de extensión se configuran como `channels.<id>` y se documentan en sus páginas dedicadas de canal (por ejemplo Feishu, Matrix, LINE, Nostr, Zalo, Nextcloud Talk, Synology Chat y Twitch).
Consulta el índice completo de canales: [Channels](/es/channels).

### Exigencia de mención en chats grupales

Los mensajes de grupo usan por defecto **require mention** (mención en metadatos o patrones regex seguros). Se aplica a chats grupales de WhatsApp, Telegram, Discord, Google Chat e iMessage.

**Tipos de mención:**

- **Menciones en metadatos**: menciones @ nativas de la plataforma. Se ignoran en el modo self-chat de WhatsApp.
- **Patrones de texto**: patrones regex seguros en `agents.list[].groupChat.mentionPatterns`. Los patrones inválidos y la repetición anidada insegura se ignoran.
- La exigencia de mención solo se aplica cuando la detección es posible (menciones nativas o al menos un patrón).

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

`messages.groupChat.historyLimit` establece el valor predeterminado global. Los canales pueden sustituirlo con `channels.<channel>.historyLimit` (o por cuenta). Establece `0` para desactivarlo.

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

Resolución: sustitución por MD → predeterminado del proveedor → sin límite (se conserva todo).

Compatibles: `telegram`, `whatsapp`, `discord`, `slack`, `signal`, `imessage`, `msteams`.

#### Modo self-chat

Incluye tu propio número en `allowFrom` para habilitar el modo self-chat (ignora las menciones @ nativas, responde solo a patrones de texto):

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
    native: "auto", // registrar comandos nativos cuando se admitan
    nativeSkills: "auto", // registrar comandos nativos de Skills cuando se admitan
    text: true, // analizar /commands en mensajes de chat
    bash: false, // permitir ! (alias: /bash)
    bashForegroundMs: 2000,
    config: false, // permitir /config
    mcp: false, // permitir /mcp
    plugins: false, // permitir /plugins
    debug: false, // permitir /debug
    restart: true, // permitir /restart + herramienta de reinicio del gateway
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="Detalles de comandos">

- Este bloque configura las superficies de comandos. Para el catálogo actual de comandos integrados + agrupados, consulta [Slash Commands](/es/tools/slash-commands).
- Esta página es una **referencia de claves de configuración**, no el catálogo completo de comandos. Los comandos propios de canal/plugin como QQ Bot `/bot-ping` `/bot-help` `/bot-logs`, LINE `/card`, device-pair `/pair`, memory `/dreaming`, phone-control `/phone` y Talk `/voice` están documentados en sus páginas de canal/plugin más [Slash Commands](/es/tools/slash-commands).
- Los comandos de texto deben ser mensajes **independientes** con `/` inicial.
- `native: "auto"` activa comandos nativos para Discord/Telegram y deja Slack desactivado.
- `nativeSkills: "auto"` activa comandos nativos de Skills para Discord/Telegram y deja Slack desactivado.
- Sustitución por canal: `channels.discord.commands.native` (bool o `"auto"`). `false` borra comandos registrados previamente.
- Sustituye el registro nativo de Skills por canal con `channels.<provider>.commands.nativeSkills`.
- `channels.telegram.customCommands` agrega entradas extra al menú del bot de Telegram.
- `bash: true` habilita `! <cmd>` para la shell del host. Requiere `tools.elevated.enabled` y remitente en `tools.elevated.allowFrom.<channel>`.
- `config: true` habilita `/config` (lee/escribe `openclaw.json`). Para clientes `chat.send` del gateway, las escrituras persistentes de `/config set|unset` también requieren `operator.admin`; `/config show` de solo lectura sigue disponible para clientes operadores normales con permiso de escritura.
- `mcp: true` habilita `/mcp` para la configuración de servidores MCP administrados por OpenClaw bajo `mcp.servers`.
- `plugins: true` habilita `/plugins` para descubrimiento, instalación y controles de habilitación/deshabilitación de plugins.
- `channels.<provider>.configWrites` controla mutaciones de configuración por canal (predeterminado: true).
- Para canales de múltiples cuentas, `channels.<provider>.accounts.<id>.configWrites` también controla las escrituras que apuntan a esa cuenta (por ejemplo `/allowlist --config --account <id>` o `/config set channels.<provider>.accounts.<id>...`).
- `restart: false` desactiva `/restart` y las acciones de herramienta de reinicio del gateway. Predeterminado: `true`.
- `ownerAllowFrom` es la lista explícita de propietarios permitidos para comandos/herramientas solo de propietario. Está separada de `allowFrom`.
- `ownerDisplay: "hash"` aplica hash a los IDs de propietario en el system prompt. Establece `ownerDisplaySecret` para controlar el hash.
- `allowFrom` es por proveedor. Cuando está configurado, es la **única** fuente de autorización (se ignoran las listas de permisos/emparejamiento del canal y `useAccessGroups`).
- `useAccessGroups: false` permite que los comandos omitan las políticas de grupos de acceso cuando `allowFrom` no está configurado.
- Mapa de documentación de comandos:
  - catálogo integrado + agrupado: [Slash Commands](/es/tools/slash-commands)
  - superficies de comandos específicas de canal: [Channels](/es/channels)
  - comandos de QQ Bot: [QQ Bot](/es/channels/qqbot)
  - comandos de emparejamiento: [Pairing](/es/channels/pairing)
  - comando de tarjeta de LINE: [LINE](/es/channels/line)
  - memory dreaming: [Dreaming](/es/concepts/dreaming)

</Accordion>

---

## Valores predeterminados del agente

### `agents.defaults.workspace`

Predeterminado: `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

Raíz opcional del repositorio mostrada en la línea Runtime del system prompt. Si no está configurada, OpenClaw la detecta automáticamente subiendo desde el workspace.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

Lista de permisos predeterminada opcional de Skills para agentes que no configuran
`agents.list[].skills`.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // hereda github, weather
      { id: "docs", skills: ["docs-search"] }, // reemplaza los predeterminados
      { id: "locked-down", skills: [] }, // sin Skills
    ],
  },
}
```

- Omite `agents.defaults.skills` para Skills sin restricción de forma predeterminada.
- Omite `agents.list[].skills` para heredar los predeterminados.
- Establece `agents.list[].skills: []` para no tener Skills.
- Una lista no vacía en `agents.list[].skills` es el conjunto final para ese agente;
  no se fusiona con los predeterminados.

### `agents.defaults.skipBootstrap`

Desactiva la creación automática de archivos bootstrap del workspace (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

Controla cuándo se inyectan los archivos bootstrap del workspace en el system prompt. Predeterminado: `"always"`.

- `"continuation-skip"`: los turnos seguros de continuación (después de una respuesta completada del asistente) omiten la reinyección del bootstrap del workspace, reduciendo el tamaño del prompt. Las ejecuciones de heartbeat y los reintentos posteriores a la compactación siguen reconstruyendo el contexto.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

Máximo de caracteres por archivo bootstrap del workspace antes de truncar. Predeterminado: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

Máximo total de caracteres inyectados entre todos los archivos bootstrap del workspace. Predeterminado: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Controla el texto de advertencia visible para el agente cuando el contexto bootstrap se trunca.
Predeterminado: `"once"`.

- `"off"`: nunca inyecta texto de advertencia en el system prompt.
- `"once"`: inyecta la advertencia una vez por firma única de truncamiento (recomendado).
- `"always"`: inyecta la advertencia en cada ejecución cuando existe truncamiento.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

Tamaño máximo en píxeles del lado más largo de la imagen en bloques de imagen de transcripción/herramienta antes de las llamadas al proveedor.
Predeterminado: `1200`.

Valores más bajos suelen reducir el uso de vision tokens y el tamaño de la carga útil de la solicitud en ejecuciones con muchas capturas de pantalla.
Valores más altos conservan más detalle visual.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

Zona horaria para el contexto del system prompt (no para las marcas de tiempo de los mensajes). Usa la zona horaria del host como respaldo.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Formato de hora en el system prompt. Predeterminado: `auto` (preferencia del SO).

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
      params: { cacheRetention: "long" }, // parámetros globales predeterminados del proveedor
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
  - También se usa como enrutamiento de respaldo cuando el modelo seleccionado/predeterminado no puede aceptar entrada de imagen.
- `imageGenerationModel`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - Lo usa la capacidad compartida de generación de imágenes y cualquier futura superficie de herramienta/plugin que genere imágenes.
  - Valores típicos: `google/gemini-3.1-flash-image-preview` para generación nativa de imágenes de Gemini, `fal/fal-ai/flux/dev` para fal, o `openai/gpt-image-1` para OpenAI Images.
  - Si seleccionas directamente un provider/model, configura también la autenticación/API key del proveedor correspondiente (por ejemplo `GEMINI_API_KEY` o `GOOGLE_API_KEY` para `google/*`, `OPENAI_API_KEY` para `openai/*`, `FAL_KEY` para `fal/*`).
  - Si se omite, `image_generate` todavía puede inferir un valor predeterminado de proveedor con autenticación. Primero prueba el proveedor predeterminado actual, luego los demás proveedores registrados de generación de imágenes en orden de ID de proveedor.
- `musicGenerationModel`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - Lo usa la capacidad compartida de generación de música y la herramienta integrada `music_generate`.
  - Valores típicos: `google/lyria-3-clip-preview`, `google/lyria-3-pro-preview` o `minimax/music-2.5+`.
  - Si se omite, `music_generate` todavía puede inferir un valor predeterminado de proveedor con autenticación. Primero prueba el proveedor predeterminado actual, luego los demás proveedores registrados de generación de música en orden de ID de proveedor.
  - Si seleccionas directamente un provider/model, configura también la autenticación/API key correspondiente del proveedor.
- `videoGenerationModel`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - Lo usa la capacidad compartida de generación de video y la herramienta integrada `video_generate`.
  - Valores típicos: `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` o `qwen/wan2.7-r2v`.
  - Si se omite, `video_generate` todavía puede inferir un valor predeterminado de proveedor con autenticación. Primero prueba el proveedor predeterminado actual, luego los demás proveedores registrados de generación de video en orden de ID de proveedor.
  - Si seleccionas directamente un provider/model, configura también la autenticación/API key correspondiente del proveedor.
  - El proveedor agrupado de generación de video Qwen admite hasta 1 video de salida, 1 imagen de entrada, 4 videos de entrada, 10 segundos de duración y opciones a nivel de proveedor `size`, `aspectRatio`, `resolution`, `audio` y `watermark`.
- `pdfModel`: acepta una cadena (`"provider/model"`) o un objeto (`{ primary, fallbacks }`).
  - Lo usa la herramienta `pdf` para el enrutamiento de modelos.
  - Si se omite, la herramienta PDF usa como respaldo `imageModel` y luego el modelo resuelto de la sesión/predeterminado.
- `pdfMaxBytesMb`: límite predeterminado de tamaño del PDF para la herramienta `pdf` cuando no se pasa `maxBytesMb` en el momento de la llamada.
- `pdfMaxPages`: páginas máximas predeterminadas consideradas por el modo de respaldo de extracción en la herramienta `pdf`.
- `verboseDefault`: nivel verbose predeterminado para agentes. Valores: `"off"`, `"on"`, `"full"`. Predeterminado: `"off"`.
- `elevatedDefault`: nivel predeterminado de salida elevada para agentes. Valores: `"off"`, `"on"`, `"ask"`, `"full"`. Predeterminado: `"on"`.
- `model.primary`: formato `provider/model` (p. ej. `openai/gpt-5.4`). Si omites el proveedor, OpenClaw intenta primero un alias, luego una coincidencia única de proveedor configurado para ese ID de modelo exacto, y solo después vuelve al proveedor predeterminado configurado (comportamiento heredado de compatibilidad, así que prefiere `provider/model` explícito). Si ese proveedor ya no expone el modelo predeterminado configurado, OpenClaw vuelve al primer provider/model configurado en lugar de mostrar un valor predeterminado obsoleto de un proveedor eliminado.
- `models`: el catálogo y la lista de modelos permitidos configurados para `/model`. Cada entrada puede incluir `alias` (atajo) y `params` (específicos del proveedor, por ejemplo `temperature`, `maxTokens`, `cacheRetention`, `context1m`).
- `params`: parámetros globales predeterminados del proveedor aplicados a todos los modelos. Se definen en `agents.defaults.params` (p. ej. `{ cacheRetention: "long" }`).
- Precedencia de fusión de `params` (configuración): `agents.defaults.params` (base global) es sustituido por `agents.defaults.models["provider/model"].params` (por modelo), luego `agents.list[].params` (ID de agente coincidente) sustituye por clave. Consulta [Prompt Caching](/es/reference/prompt-caching) para más detalles.
- Los escritores de configuración que mutan estos campos (por ejemplo `/models set`, `/models set-image` y los comandos de agregar/eliminar fallback) guardan la forma canónica de objeto y conservan las listas fallback existentes cuando es posible.
- `maxConcurrent`: máximo de ejecuciones de agentes en paralelo entre sesiones (cada sesión sigue serializada). Predeterminado: 4.

**Alias abreviados integrados** (solo se aplican cuando el modelo está en `agents.defaults.models`):

| Alias               | Modelo                                 |
| ------------------- | -------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`            |
| `sonnet`            | `anthropic/claude-sonnet-4-6`          |
| `gpt`               | `openai/gpt-5.4`                       |
| `gpt-mini`          | `openai/gpt-5.4-mini`                  |
| `gpt-nano`          | `openai/gpt-5.4-nano`                  |
| `gemini`            | `google/gemini-3.1-pro-preview`        |
| `gemini-flash`      | `google/gemini-3-flash-preview`        |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

Tus alias configurados siempre prevalecen sobre los predeterminados.

Los modelos Z.AI GLM-4.x habilitan automáticamente el modo thinking a menos que establezcas `--thinking off` o definas tú mismo `agents.defaults.models["zai/<model>"].params.thinking`.
Los modelos Z.AI habilitan `tool_stream` por defecto para el streaming de llamadas a herramientas. Establece `agents.defaults.models["zai/<model>"].params.tool_stream` en `false` para desactivarlo.
Los modelos Anthropic Claude 4.6 usan `adaptive` thinking por defecto cuando no se establece un nivel explícito de thinking.

### `agents.defaults.cliBackends`

Backends CLI opcionales para ejecuciones de respaldo solo de texto (sin llamadas a herramientas). Útiles como respaldo cuando fallan los proveedores de API.

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
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

- Los backends CLI son principalmente de texto; las herramientas siempre están desactivadas.
- Sesiones compatibles cuando se establece `sessionArg`.
- Paso de imágenes compatible cuando `imageArg` acepta rutas de archivo.

### `agents.defaults.systemPromptOverride`

Reemplaza todo el system prompt ensamblado por OpenClaw con una cadena fija. Defínelo en el nivel predeterminado (`agents.defaults.systemPromptOverride`) o por agente (`agents.list[].systemPromptOverride`). Los valores por agente tienen prioridad; un valor vacío o solo con espacios se ignora. Útil para experimentos controlados de prompts.

```json5
{
  agents: {
    defaults: {
      systemPromptOverride: "You are a helpful assistant.",
    },
  },
}
```

### `agents.defaults.heartbeat`

Ejecuciones periódicas de heartbeat.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m desactiva
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // predeterminado: true; false omite la sección Heartbeat del system prompt
        lightContext: false, // predeterminado: false; true conserva solo HEARTBEAT.md de los archivos bootstrap del workspace
        isolatedSession: false, // predeterminado: false; true ejecuta cada heartbeat en una sesión nueva (sin historial de conversación)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (predeterminado) | block
        target: "none", // predeterminado: none | opciones: last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
      },
    },
  },
}
```

- `every`: cadena de duración (ms/s/m/h). Predeterminado: `30m` (autenticación con API key) o `1h` (autenticación OAuth). Establece `0m` para desactivar.
- `includeSystemPromptSection`: cuando es false, omite la sección Heartbeat del system prompt y omite la inyección de `HEARTBEAT.md` en el contexto bootstrap. Predeterminado: `true`.
- `suppressToolErrorWarnings`: cuando es true, suprime las cargas de advertencia por errores de herramientas durante las ejecuciones de heartbeat.
- `directPolicy`: política de entrega directa/MD. `allow` (predeterminado) permite la entrega a destinos directos. `block` la suprime y emite `reason=dm-blocked`.
- `lightContext`: cuando es true, las ejecuciones de heartbeat usan un contexto bootstrap ligero y conservan solo `HEARTBEAT.md` de los archivos bootstrap del workspace.
- `isolatedSession`: cuando es true, cada heartbeat se ejecuta en una sesión nueva sin historial previo de conversación. Mismo patrón de aislamiento que cron `sessionTarget: "isolated"`. Reduce el coste por heartbeat de ~100K a ~2-5K tokens.
- Por agente: configura `agents.list[].heartbeat`. Cuando cualquier agente define `heartbeat`, **solo esos agentes** ejecutan heartbeats.
- Los heartbeats ejecutan turnos completos del agente: intervalos más cortos consumen más tokens.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // id de un plugin proveedor de compactación registrado (opcional)
        timeoutSeconds: 900,
        reserveTokensFloor: 24000,
        identifierPolicy: "strict", // strict | off | custom
        identifierInstructions: "Preserve deployment IDs, ticket IDs, and host:port pairs exactly.", // usado cuando identifierPolicy=custom
        postCompactionSections: ["Session Startup", "Red Lines"], // [] desactiva la reinyección
        model: "openrouter/anthropic/claude-sonnet-4-6", // sustitución opcional de modelo solo para compactación
        notifyUser: true, // envía un aviso breve cuando comienza la compactación (predeterminado: false)
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

- `mode`: `default` o `safeguard` (resumen por fragmentos para historiales largos). Consulta [Compaction](/es/concepts/compaction).
- `provider`: id de un plugin proveedor de compactación registrado. Cuando está configurado, se llama a `summarize()` del proveedor en lugar del resumen LLM integrado. Si falla, vuelve al integrado. Configurar un proveedor fuerza `mode: "safeguard"`. Consulta [Compaction](/es/concepts/compaction).
- `timeoutSeconds`: segundos máximos permitidos para una sola operación de compactación antes de que OpenClaw la aborte. Predeterminado: `900`.
- `identifierPolicy`: `strict` (predeterminado), `off` o `custom`. `strict` antepone la guía integrada de conservación de identificadores opacos durante el resumen de compactación.
- `identifierInstructions`: texto opcional personalizado de conservación de identificadores usado cuando `identifierPolicy=custom`.
- `postCompactionSections`: nombres opcionales de secciones H2/H3 de AGENTS.md que se reinyectan después de la compactación. Predeterminado: `["Session Startup", "Red Lines"]`; establece `[]` para desactivar la reinyección. Cuando no está configurado o se establece explícitamente en ese par predeterminado, también se aceptan los encabezados heredados `Every Session`/`Safety` como respaldo.
- `model`: sustitución opcional `provider/model-id` solo para el resumen de compactación. Úsalo cuando la sesión principal deba mantener un modelo pero los resúmenes de compactación deban ejecutarse con otro; cuando no está configurado, la compactación usa el modelo principal de la sesión.
- `notifyUser`: cuando es `true`, envía un aviso breve al usuario al comenzar la compactación (por ejemplo, "Compacting context..."). Está desactivado por defecto para que la compactación sea silenciosa.
- `memoryFlush`: turno agentic silencioso antes de la auto-compactación para almacenar memorias duraderas. Se omite cuando el workspace es de solo lectura.

### `agents.defaults.contextPruning`

Recorta **resultados antiguos de herramientas** del contexto en memoria antes de enviarlo al LLM. **No** modifica el historial de la sesión en disco.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // duración (ms/s/m/h), unidad predeterminada: minutos
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

<Accordion title="Comportamiento del modo cache-ttl">

- `mode: "cache-ttl"` habilita las pasadas de recorte.
- `ttl` controla con qué frecuencia puede volver a ejecutarse el recorte (después del último acceso a la caché).
- El recorte primero aplica soft-trim a los resultados sobredimensionados de herramientas, y luego hard-clear a resultados antiguos de herramientas si es necesario.

**Soft-trim** conserva el principio + el final e inserta `...` en el medio.

**Hard-clear** reemplaza todo el resultado de la herramienta con el placeholder.

Notas:

- Los bloques de imagen nunca se recortan ni limpian.
- Las proporciones se basan en caracteres (aproximadas), no en recuentos exactos de tokens.
- Si existen menos de `keepLastAssistants` mensajes del asistente, se omite el recorte.

</Accordion>

Consulta [Session Pruning](/es/concepts/session-pruning) para detalles del comportamiento.

### Block streaming

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom (usa minMs/maxMs)
    },
  },
}
```

- Los canales no Telegram requieren `*.blockStreaming: true` explícito para habilitar respuestas por bloques.
- Sustituciones por canal: `channels.<channel>.blockStreamingCoalesce` (y variantes por cuenta). Signal/Slack/Discord/Google Chat usan por defecto `minChars: 1500`.
- `humanDelay`: pausa aleatoria entre respuestas por bloques. `natural` = 800–2500ms. Sustitución por agente: `agents.list[].humanDelay`.

Consulta [Streaming](/es/concepts/streaming) para comportamiento + detalles de fragmentación.

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

- Predeterminados: `instant` para chats directos/menciones, `message` para chats grupales sin mención.
- Sustituciones por sesión: `session.typingMode`, `session.typingIntervalSeconds`.

Consulta [Typing Indicators](/es/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Sandboxing opcional para el agente integrado. Consulta [Sandboxing](/es/gateway/sandboxing) para la guía completa.

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
          // También se admiten SecretRefs / contenidos en línea:
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

<Accordion title="Detalles de sandbox">

**Backend:**

- `docker`: runtime local de Docker (predeterminado)
- `ssh`: runtime remoto genérico respaldado por SSH
- `openshell`: runtime de OpenShell

Cuando se selecciona `backend: "openshell"`, los ajustes específicos del runtime se mueven a
`plugins.entries.openshell.config`.

**Configuración del backend SSH:**

- `target`: destino SSH con formato `user@host[:port]`
- `command`: comando del cliente SSH (predeterminado: `ssh`)
- `workspaceRoot`: raíz remota absoluta usada para workspaces por alcance
- `identityFile` / `certificateFile` / `knownHostsFile`: archivos locales existentes pasados a OpenSSH
- `identityData` / `certificateData` / `knownHostsData`: contenidos en línea o SecretRefs que OpenClaw materializa en archivos temporales en tiempo de ejecución
- `strictHostKeyChecking` / `updateHostKeys`: ajustes de política de claves de host de OpenSSH

**Precedencia de autenticación SSH:**

- `identityData` prevalece sobre `identityFile`
- `certificateData` prevalece sobre `certificateFile`
- `knownHostsData` prevalece sobre `knownHostsFile`
- Los valores `*Data` respaldados por SecretRef se resuelven a partir de la instantánea activa del runtime de secretos antes de que comience la sesión sandbox

**Comportamiento del backend SSH:**

- siembra el workspace remoto una vez después de crear o recrear
- luego conserva como canónico el workspace SSH remoto
- enruta `exec`, herramientas de archivos y rutas de medios por SSH
- no sincroniza automáticamente al host los cambios remotos
- no admite contenedores de navegador sandbox

**Acceso al workspace:**

- `none`: workspace sandbox por alcance bajo `~/.openclaw/sandboxes`
- `ro`: workspace sandbox en `/workspace`, workspace del agente montado como solo lectura en `/agent`
- `rw`: workspace del agente montado en lectura/escritura en `/workspace`

**Alcance:**

- `session`: contenedor + workspace por sesión
- `agent`: un contenedor + workspace por agente (predeterminado)
- `shared`: contenedor y workspace compartidos (sin aislamiento entre sesiones)

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
          gateway: "lab", // opcional
          gatewayEndpoint: "https://lab.example", // opcional
          policy: "strict", // id de política de OpenShell opcional
          providers: ["openai"], // opcional
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**Modo OpenShell:**

- `mirror`: siembra el remoto desde local antes de exec, sincroniza de vuelta después de exec; el workspace local sigue siendo el canónico
- `remote`: siembra el remoto una vez cuando se crea el sandbox, luego conserva como canónico el workspace remoto

En el modo `remote`, las ediciones locales del host hechas fuera de OpenClaw no se sincronizan automáticamente al sandbox después del paso inicial de siembra.
El transporte es SSH al sandbox de OpenShell, pero el plugin es dueño del ciclo de vida del sandbox y de la sincronización espejo opcional.

**`setupCommand`** se ejecuta una vez después de crear el contenedor (mediante `sh -lc`). Necesita salida de red, raíz escribible y usuario root.

**Los contenedores usan por defecto `network: "none"`**; configúralo en `"bridge"` (o una red bridge personalizada) si el agente necesita acceso saliente.
`"host"` está bloqueado. `"container:<id>"` está bloqueado por defecto a menos que establezcas explícitamente
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` (emergencia).

**Los adjuntos entrantes** se almacenan temporalmente en `media/inbound/*` en el workspace activo.

**`docker.binds`** monta directorios adicionales del host; los binds globales y por agente se fusionan.

**Navegador en sandbox** (`sandbox.browser.enabled`): Chromium + CDP en un contenedor. La URL de noVNC se inyecta en el system prompt. No requiere `browser.enabled` en `openclaw.json`.
El acceso de observador noVNC usa autenticación VNC por defecto y OpenClaw emite una URL con token de corta duración (en lugar de exponer la contraseña en la URL compartida).

- `allowHostControl: false` (predeterminado) bloquea que las sesiones sandbox apunten al navegador del host.
- `network` usa por defecto `openclaw-sandbox-browser` (red bridge dedicada). Configúralo en `bridge` solo cuando quieras explícitamente conectividad global del bridge.
- `cdpSourceRange` restringe opcionalmente la entrada CDP en el borde del contenedor a un rango CIDR (por ejemplo `172.21.0.1/32`).
- `sandbox.browser.binds` monta directorios adicionales del host solo en el contenedor del navegador sandbox. Cuando está configurado (incluido `[]`), reemplaza `docker.binds` para el contenedor del navegador.
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
    habilitados por defecto y pueden desactivarse con
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` si el uso de WebGL/3D lo requiere.
  - `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` vuelve a habilitar extensiones si tu flujo de trabajo
    depende de ellas.
  - `--renderer-process-limit=2` puede cambiarse con
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`; establece `0` para usar
    el límite predeterminado de procesos de Chromium.
  - además de `--no-sandbox` y `--disable-setuid-sandbox` cuando `noSandbox` está habilitado.
  - Los valores predeterminados son la línea base de la imagen del contenedor; usa una imagen de navegador personalizada con un
    entrypoint personalizado para cambiar los valores predeterminados del contenedor.

</Accordion>

El sandboxing del navegador y `sandbox.docker.binds` solo son compatibles con Docker.

Compilar imágenes:

```bash
scripts/sandbox-setup.sh           # imagen principal de sandbox
scripts/sandbox-browser-setup.sh   # imagen opcional del navegador
```

### `agents.list` (sustituciones por agente)

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
        model: "anthropic/claude-opus-4-6", // o { primary, fallbacks }
        thinkingDefault: "high", // sustitución del nivel thinking por agente
        reasoningDefault: "on", // sustitución de visibilidad de reasoning por agente
        fastModeDefault: false, // sustitución de modo fast por agente
        params: { cacheRetention: "none" }, // sustituye por clave los params coincidentes de defaults.models
        skills: ["docs-search"], // reemplaza agents.defaults.skills cuando se establece
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
- `default`: cuando se establecen varios, gana el primero (se registra una advertencia). Si ninguno está establecido, la primera entrada de la lista es la predeterminada.
- `model`: la forma de cadena sustituye solo `primary`; la forma de objeto `{ primary, fallbacks }` sustituye ambos (`[]` desactiva los fallbacks globales). Los cron jobs que solo sustituyen `primary` siguen heredando los fallbacks predeterminados salvo que establezcas `fallbacks: []`.
- `params`: parámetros de stream por agente fusionados sobre la entrada del modelo seleccionado en `agents.defaults.models`. Úsalo para sustituciones específicas del agente como `cacheRetention`, `temperature` o `maxTokens` sin duplicar todo el catálogo de modelos.
- `skills`: lista opcional de Skills permitidas por agente. Si se omite, el agente hereda `agents.defaults.skills` cuando está configurado; una lista explícita reemplaza los predeterminados en lugar de fusionarse, y `[]` significa sin Skills.
- `thinkingDefault`: nivel predeterminado opcional de thinking por agente (`off | minimal | low | medium | high | xhigh | adaptive`). Sustituye `agents.defaults.thinkingDefault` para este agente cuando no hay sustitución por mensaje o sesión.
- `reasoningDefault`: visibilidad predeterminada opcional de reasoning por agente (`on | off | stream`). Se aplica cuando no hay sustitución de reasoning por mensaje o sesión.
- `fastModeDefault`: valor predeterminado opcional por agente para modo fast (`true | false`). Se aplica cuando no hay sustitución de modo fast por mensaje o sesión.
- `runtime`: descriptor opcional de runtime por agente. Usa `type: "acp"` con los valores predeterminados de `runtime.acp` (`agent`, `backend`, `mode`, `cwd`) cuando el agente deba usar sesiones del arnés ACP por defecto.
- `identity.avatar`: ruta relativa al workspace, URL `http(s)` o URI `data:`.
- `identity` deriva predeterminados: `ackReaction` desde `emoji`, `mentionPatterns` desde `name`/`emoji`.
- `subagents.allowAgents`: lista permitida de IDs de agente para `sessions_spawn` (`["*"]` = cualquiera; predeterminado: solo el mismo agente).
- Protección de herencia de sandbox: si la sesión solicitante está en sandbox, `sessions_spawn` rechaza objetivos que se ejecutarían sin sandbox.
- `subagents.requireAgentId`: cuando es true, bloquea llamadas `sessions_spawn` que omiten `agentId` (fuerza selección explícita de perfil; predeterminado: false).

---

## Enrutamiento multiagente

Ejecuta varios agentes aislados dentro de un solo Gateway. Consulta [Multi-Agent](/es/concepts/multi-agent).

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

### Campos de coincidencia del enlace

- `type` (opcional): `route` para enrutamiento normal (si falta, route es el predeterminado), `acp` para enlaces persistentes de conversación ACP.
- `match.channel` (obligatorio)
- `match.accountId` (opcional; `*` = cualquier cuenta; omitido = cuenta predeterminada)
- `match.peer` (opcional; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (opcional; específico del canal)
- `acp` (opcional; solo para entradas `type: "acp"`): `{ mode, label, cwd, backend }`

**Orden de coincidencia determinista:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (exacto, sin peer/guild/team)
5. `match.accountId: "*"` (en todo el canal)
6. Agente predeterminado

Dentro de cada nivel, gana la primera entrada coincidente de `bindings`.

Para entradas `type: "acp"`, OpenClaw resuelve por identidad exacta de conversación (`match.channel` + cuenta + `match.peer.id`) y no usa el orden por niveles de route binding anterior.

### Perfiles de acceso por agente

<Accordion title="Acceso completo (sin sandbox)">

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

<Accordion title="Herramientas + workspace de solo lectura">

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

<Accordion title="Sin acceso al sistema de archivos (solo mensajería)">

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

Consulta [Multi-Agent Sandbox & Tools](/es/tools/multi-agent-sandbox-tools) para detalles de precedencia.

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
    parentForkMaxTokens: 100000, // omitir fork del hilo padre por encima de este recuento de tokens (0 desactiva)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // duración o false
      maxDiskBytes: "500mb", // opcional presupuesto estricto
      highWaterBytes: "400mb", // objetivo opcional de limpieza
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // auto-unfocus predeterminado por inactividad en horas (`0` desactiva)
      maxAgeHours: 0, // antigüedad máxima estricta predeterminada en horas (`0` desactiva)
    },
    mainKey: "main", // heredado (el runtime siempre usa "main")
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Detalles de campos de sesión">

- **`scope`**: estrategia base de agrupación de sesiones para contextos de chats grupales.
  - `per-sender` (predeterminado): cada remitente obtiene una sesión aislada dentro de un contexto de canal.
  - `global`: todos los participantes en un contexto de canal comparten una sola sesión (úsalo solo cuando se quiera contexto compartido).
- **`dmScope`**: cómo se agrupan los MD.
  - `main`: todos los MD comparten la sesión principal.
  - `per-peer`: aísla por ID del remitente entre canales.
  - `per-channel-peer`: aísla por canal + remitente (recomendado para bandejas de entrada multiusuario).
  - `per-account-channel-peer`: aísla por cuenta + canal + remitente (recomendado para múltiples cuentas).
- **`identityLinks`**: asigna IDs canónicos a peers con prefijo de proveedor para compartir sesiones entre canales.
- **`reset`**: política principal de reinicio. `daily` reinicia a `atHour` en hora local; `idle` reinicia tras `idleMinutes`. Cuando ambos están configurados, gana el que venza antes.
- **`resetByType`**: sustituciones por tipo (`direct`, `group`, `thread`). Se acepta `dm` heredado como alias de `direct`.
- **`parentForkMaxTokens`**: `totalTokens` máximo permitido en la sesión padre al crear una sesión de hilo derivada (predeterminado `100000`).
  - Si `totalTokens` del padre supera este valor, OpenClaw inicia una nueva sesión de hilo en lugar de heredar el historial de transcripción del padre.
  - Establece `0` para desactivar esta protección y permitir siempre el fork del padre.
- **`mainKey`**: campo heredado. El runtime siempre usa `"main"` para el bucket principal de chat directo.
- **`agentToAgent.maxPingPongTurns`**: número máximo de turnos de respuesta de ida y vuelta entre agentes durante intercambios agente a agente (entero, rango: `0`–`5`). `0` desactiva la cadena ping-pong.
- **`sendPolicy`**: coincide por `channel`, `chatType` (`direct|group|channel`, con alias heredado `dm`), `keyPrefix` o `rawKeyPrefix`. Gana el primer deny.
- **`maintenance`**: limpieza + controles de retención del almacén de sesiones.
  - `mode`: `warn` solo emite advertencias; `enforce` aplica la limpieza.
  - `pruneAfter`: límite de antigüedad para entradas obsoletas (predeterminado `30d`).
  - `maxEntries`: número máximo de entradas en `sessions.json` (predeterminado `500`).
  - `rotateBytes`: rota `sessions.json` cuando supera este tamaño (predeterminado `10mb`).
  - `resetArchiveRetention`: retención para archivos de transcripción `*.reset.<timestamp>`. Usa `pruneAfter` como predeterminado; establece `false` para desactivar.
  - `maxDiskBytes`: presupuesto opcional de disco para el directorio de sesiones. En modo `warn` registra advertencias; en modo `enforce` elimina primero los artefactos/sesiones más antiguos.
  - `highWaterBytes`: objetivo opcional después de la limpieza por presupuesto. Predeterminado: `80%` de `maxDiskBytes`.
- **`threadBindings`**: valores predeterminados globales para funciones de sesión ligadas a hilos.
  - `enabled`: interruptor maestro predeterminado (los proveedores pueden sustituirlo; Discord usa `channels.discord.threadBindings.enabled`)
  - `idleHours`: auto-unfocus predeterminado por inactividad en horas (`0` desactiva; los proveedores pueden sustituir)
  - `maxAgeHours`: antigüedad máxima estricta predeterminada en horas (`0` desactiva; los proveedores pueden sustituir)

</Accordion>

---

## Mensajes

```json5
{
  messages: {
    responsePrefix: "🦞", // o "auto"
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
      debounceMs: 2000, // 0 desactiva
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Prefijo de respuesta

Sustituciones por canal/cuenta: `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Resolución (gana la más específica): cuenta → canal → global. `""` desactiva y detiene la cascada. `"auto"` deriva `[{identity.name}]`.

**Variables de plantilla:**

| Variable          | Descripción            | Ejemplo                     |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | Nombre corto del modelo | `claude-opus-4-6`           |
| `{modelFull}`     | Identificador completo del modelo | `anthropic/claude-opus-4-6` |
| `{provider}`      | Nombre del proveedor   | `anthropic`                 |
| `{thinkingLevel}` | Nivel actual de thinking | `high`, `low`, `off`        |
| `{identity.name}` | Nombre de identidad del agente | (igual que `"auto"`)          |

Las variables no distinguen mayúsculas de minúsculas. `{think}` es un alias de `{thinkingLevel}`.

### Reacción de acuse

- Usa por defecto `identity.emoji` del agente activo; en otro caso `"👀"`. Establece `""` para desactivar.
- Sustituciones por canal: `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Orden de resolución: cuenta → canal → `messages.ackReaction` → respaldo de identidad.
- Alcance: `group-mentions` (predeterminado), `group-all`, `direct`, `all`.
- `removeAckAfterReply`: elimina el acuse después de responder en Slack, Discord y Telegram.
- `messages.statusReactions.enabled`: habilita reacciones de estado del ciclo de vida en Slack, Discord y Telegram.
  En Slack y Discord, si no está configurado, mantiene activadas las reacciones de estado cuando las reacciones de acuse están activas.
  En Telegram, configúralo explícitamente en `true` para habilitar reacciones de estado del ciclo de vida.

### Debounce de entrada

Agrupa mensajes rápidos solo de texto del mismo remitente en un único turno del agente. Los medios/adjuntos se vacían inmediatamente. Los comandos de control omiten el debounce.

### TTS (text-to-speech)

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

- `auto` controla el modo predeterminado de auto-TTS: `off`, `always`, `inbound` o `tagged`. `/tts on|off` puede sustituir las preferencias locales, y `/tts status` muestra el estado efectivo.
- `summaryModel` sustituye `agents.defaults.model.primary` para el auto-resumen.
- `modelOverrides` está habilitado por defecto; `modelOverrides.allowProvider` usa `false` por defecto (opt-in).
- Las API keys usan como respaldo `ELEVENLABS_API_KEY`/`XI_API_KEY` y `OPENAI_API_KEY`.
- `openai.baseUrl` sustituye el endpoint TTS de OpenAI. Orden de resolución: configuración, luego `OPENAI_TTS_BASE_URL`, luego `https://api.openai.com/v1`.
- Cuando `openai.baseUrl` apunta a un endpoint que no es de OpenAI, OpenClaw lo trata como un servidor TTS compatible con OpenAI y relaja la validación de modelo/voz.

---

## Talk

Predeterminados para el modo Talk (macOS/iOS/Android).

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

- `talk.provider` debe coincidir con una clave en `talk.providers` cuando se configuran varios proveedores de Talk.
- Las claves heredadas planas de Talk (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) son solo de compatibilidad y se migran automáticamente a `talk.providers.<provider>`.
- Los IDs de voz usan como respaldo `ELEVENLABS_VOICE_ID` o `SAG_VOICE_ID`.
- `providers.*.apiKey` acepta cadenas en texto plano u objetos SecretRef.
- El respaldo `ELEVENLABS_API_KEY` solo se aplica cuando no se configura una API key de Talk.
- `providers.*.voiceAliases` permite que las directivas de Talk usen nombres amigables.
- `silenceTimeoutMs` controla cuánto espera el modo Talk después del silencio del usuario antes de enviar la transcripción. Si no está configurado, se conserva la ventana de pausa predeterminada de la plataforma (`700 ms en macOS y Android, 900 ms en iOS`).

---

## Herramientas

### Perfiles de herramientas

`tools.profile` establece una lista base de permisos antes de `tools.allow`/`tools.deny`:

El onboarding local establece por defecto en nuevas configuraciones locales `tools.profile: "coding"` cuando no está configurado (los perfiles explícitos existentes se conservan).

| Perfil      | Incluye                                                                                                                        |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | solo `session_status`                                                                                                           |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate`, `video_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                                       |
| `full`      | Sin restricción (igual que no configurado)                                                                                      |

### Grupos de herramientas

| Grupo              | Herramientas                                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` se acepta como alias de `exec`)                                                   |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                         |
| `group:sessions`   | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`       |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                  |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                          |
| `group:ui`         | `browser`, `canvas`                                                                                                            |
| `group:automation` | `cron`, `gateway`                                                                                                              |
| `group:messaging`  | `message`                                                                                                                      |
| `group:nodes`      | `nodes`                                                                                                                        |
| `group:agents`     | `agents_list`                                                                                                                  |
| `group:media`      | `image`, `image_generate`, `video_generate`, `tts`                                                                             |
| `group:openclaw`   | Todas las herramientas integradas (excluye plugins de proveedor)                                                               |

### `tools.allow` / `tools.deny`

Política global de permitir/denegar herramientas (deny prevalece). No distingue mayúsculas de minúsculas y admite comodines `*`. Se aplica incluso cuando el sandbox de Docker está desactivado.

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

Controla el acceso exec elevado fuera del sandbox:

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

- La sustitución por agente (`agents.list[].tools.elevated`) solo puede restringir aún más.
- `/elevated on|off|ask|full` guarda el estado por sesión; las directivas en línea se aplican a un solo mensaje.
- `exec` elevado omite el sandboxing y usa la ruta de escape configurada (`gateway` por defecto, o `node` cuando el destino exec es `node`).

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

Las verificaciones de seguridad de bucles de herramientas están **desactivadas por defecto**. Establece `enabled: true` para activar la detección.
Los ajustes pueden definirse globalmente en `tools.loopDetection` y sustituirse por agente en `agents.list[].tools.loopDetection`.

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

- `historySize`: historial máximo de llamadas a herramientas conservado para análisis de bucles.
- `warningThreshold`: umbral de patrones repetidos sin progreso para advertencias.
- `criticalThreshold`: umbral repetido más alto para bloquear bucles críticos.
- `globalCircuitBreakerThreshold`: umbral de parada estricta para cualquier ejecución sin progreso.
- `detectors.genericRepeat`: advierte sobre llamadas repetidas a la misma herramienta/con los mismos args.
- `detectors.knownPollNoProgress`: advierte/bloquea herramientas conocidas de sondeo (`process.poll`, `command_status`, etc.).
- `detectors.pingPong`: advierte/bloquea patrones alternos de pares sin progreso.
- Si `warningThreshold >= criticalThreshold` o `criticalThreshold >= globalCircuitBreakerThreshold`, la validación falla.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // o entorno BRAVE_API_KEY
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // opcional; omitir para autodetección
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

Configura la comprensión de medios entrantes (imagen/audio/video):

```json5
{
  tools: {
    media: {
      concurrency: 2,
      asyncCompletion: {
        directSend: false, // opt-in: enviar música/video asíncronos finalizados directamente al canal
      },
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

<Accordion title="Campos de entrada del modelo de medios">

**Entrada de proveedor** (`type: "provider"` u omitido):

- `provider`: id del proveedor de API (`openai`, `anthropic`, `google`/`gemini`, `groq`, etc.)
- `model`: sustitución del ID de modelo
- `profile` / `preferredProfile`: selección del perfil de `auth-profiles.json`

**Entrada CLI** (`type: "cli"`):

- `command`: ejecutable que se va a ejecutar
- `args`: args con plantilla (admite `{{MediaPath}}`, `{{Prompt}}`, `{{MaxChars}}`, etc.)

**Campos comunes:**

- `capabilities`: lista opcional (`image`, `audio`, `video`). Predeterminados: `openai`/`anthropic`/`minimax` → image, `google` → image+audio+video, `groq` → audio.
- `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: sustituciones por entrada.
- Los fallos pasan a la siguiente entrada.

La autenticación del proveedor sigue el orden estándar: `auth-profiles.json` → variables de entorno → `models.providers.*.apiKey`.

**Campos de finalización asíncrona:**

- `asyncCompletion.directSend`: cuando es `true`, las tareas completadas
  `music_generate` y `video_generate` intentan primero la entrega directa al canal. Predeterminado: `false`
  (ruta heredada de activación de sesión solicitante/entrega por modelo).

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

Controla a qué sesiones pueden dirigirse las herramientas de sesión (`sessions_list`, `sessions_history`, `sessions_send`).

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

- `self`: solo la clave de la sesión actual.
- `tree`: sesión actual + sesiones generadas por la sesión actual (subagentes).
- `agent`: cualquier sesión perteneciente al ID del agente actual (puede incluir otros usuarios si ejecutas sesiones per-sender bajo el mismo ID de agente).
- `all`: cualquier sesión. El direccionamiento entre agentes sigue requiriendo `tools.agentToAgent`.
- Restricción por sandbox: cuando la sesión actual está en sandbox y `agents.defaults.sandbox.sessionToolsVisibility="spawned"`, la visibilidad se fuerza a `tree` incluso si `tools.sessions.visibility="all"`.

### `tools.sessions_spawn`

Controla la compatibilidad con adjuntos en línea para `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: establece true para permitir adjuntos de archivo en línea
        maxTotalBytes: 5242880, // 5 MB totales entre todos los archivos
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB por archivo
        retainOnSessionKeep: false, // conservar adjuntos cuando cleanup="keep"
      },
    },
  },
}
```

Notas:

- Los adjuntos solo son compatibles con `runtime: "subagent"`. El runtime ACP los rechaza.
- Los archivos se materializan en el workspace hijo en `.openclaw/attachments/<uuid>/` con un `.manifest.json`.
- El contenido de los adjuntos se redacta automáticamente de la persistencia de la transcripción.
- Las entradas base64 se validan con comprobaciones estrictas de alfabeto/relleno y una protección de tamaño antes de decodificar.
- Los permisos de archivo son `0700` para directorios y `0600` para archivos.
- La limpieza sigue la política `cleanup`: `delete` siempre elimina los adjuntos; `keep` solo los conserva cuando `retainOnSessionKeep: true`.

### `tools.experimental`

Flags experimentales de herramientas integradas. Desactivado por defecto salvo que se aplique una regla de auto-enable específica del runtime.

```json5
{
  tools: {
    experimental: {
      planTool: true, // habilitar update_plan experimental
    },
  },
}
```

Notas:

- `planTool`: habilita la herramienta estructurada `update_plan` para seguimiento de trabajo no trivial de varios pasos.
- Predeterminado: `false` para proveedores no OpenAI. Las ejecuciones de OpenAI y OpenAI Codex la habilitan automáticamente cuando no está configurada; establece `false` para desactivar ese auto-enable.
- Cuando está habilitada, el system prompt también agrega una guía de uso para que el modelo solo la use en trabajo sustancial y conserve como máximo un paso `in_progress`.

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
- `allowAgents`: lista predeterminada de IDs de agente objetivo para `sessions_spawn` cuando el agente solicitante no establece su propio `subagents.allowAgents` (`["*"]` = cualquiera; predeterminado: solo el mismo agente).
- `runTimeoutSeconds`: timeout predeterminado (segundos) para `sessions_spawn` cuando la llamada a la herramienta omite `runTimeoutSeconds`. `0` significa sin timeout.
- Política de herramientas por subagente: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Proveedores personalizados y base URLs

OpenClaw usa el catálogo de modelos integrado. Agrega proveedores personalizados mediante `models.providers` en la configuración o `~/.openclaw/agents/<agentId>/agent/models.json`.

```json5
{
  models: {
    mode: "merge", // merge (predeterminado) | replace
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

- Usa `authHeader: true` + `headers` para necesidades personalizadas de autenticación.
- Sustituye la raíz de configuración del agente con `OPENCLAW_AGENT_DIR` (o `PI_CODING_AGENT_DIR`, alias heredado de variable de entorno).
- Precedencia de fusión para IDs de proveedor coincidentes:
  - Los valores `baseUrl` no vacíos de `models.json` del agente prevalecen.
  - Los valores `apiKey` no vacíos del agente prevalecen solo cuando ese proveedor no está administrado por SecretRef en el contexto actual de configuración/perfil de autenticación.
  - Los valores `apiKey` de proveedor gestionados por SecretRef se actualizan desde marcadores de origen (`ENV_VAR_NAME` para refs de entorno, `secretref-managed` para refs de archivo/exec) en lugar de persistir secretos resueltos.
  - Los valores de encabezado de proveedor gestionados por SecretRef se actualizan desde marcadores de origen (`secretref-env:ENV_VAR_NAME` para refs de entorno, `secretref-managed` para refs de archivo/exec).
  - `apiKey`/`baseUrl` vacíos o ausentes en el agente usan como respaldo `models.providers` en la configuración.
  - Los `contextWindow`/`maxTokens` del modelo coincidente usan el valor más alto entre la configuración explícita y los valores implícitos del catálogo.
  - `contextTokens` del modelo coincidente conserva un límite explícito de runtime cuando está presente; úsalo para limitar el contexto efectivo sin cambiar los metadatos nativos del modelo.
  - Usa `models.mode: "replace"` cuando quieras que la configuración reescriba completamente `models.json`.
  - La persistencia de marcadores es autoritativa respecto al origen: los marcadores se escriben desde la instantánea activa de configuración del origen (antes de la resolución), no desde los valores secretos resueltos en runtime.

### Detalles de campos del proveedor

- `models.mode`: comportamiento del catálogo de proveedores (`merge` o `replace`).
- `models.providers`: mapa de proveedores personalizados indexado por ID de proveedor.
- `models.providers.*.api`: adaptador de solicitudes (`openai-completions`, `openai-responses`, `anthropic-messages`, `google-generative-ai`, etc).
- `models.providers.*.apiKey`: credencial del proveedor (prefiere SecretRef/sustitución de entorno).
- `models.providers.*.auth`: estrategia de autenticación (`api-key`, `token`, `oauth`, `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: para Ollama + `openai-completions`, inyecta `options.num_ctx` en las solicitudes (predeterminado: `true`).
- `models.providers.*.authHeader`: fuerza el transporte de credenciales en el encabezado `Authorization` cuando sea necesario.
- `models.providers.*.baseUrl`: base URL de la API upstream.
- `models.providers.*.headers`: encabezados estáticos extra para enrutamiento proxy/inquilino.
- `models.providers.*.request`: sustituciones de transporte para solicitudes HTTP del proveedor de modelos.
  - `request.headers`: encabezados extra (fusionados con los predeterminados del proveedor). Los valores aceptan SecretRef.
  - `request.auth`: sustitución de estrategia de autenticación. Modos: `"provider-default"` (usa la autenticación integrada del proveedor), `"authorization-bearer"` (con `token`), `"header"` (con `headerName`, `value`, `prefix` opcional).
  - `request.proxy`: sustitución de proxy HTTP. Modos: `"env-proxy"` (usa variables de entorno `HTTP_PROXY`/`HTTPS_PROXY`), `"explicit-proxy"` (con `url`). Ambos modos aceptan un subobjeto `tls` opcional.
  - `request.tls`: sustitución TLS para conexiones directas. Campos: `ca`, `cert`, `key`, `passphrase` (todos aceptan SecretRef), `serverName`, `insecureSkipVerify`.
- `models.providers.*.models`: entradas explícitas del catálogo de modelos del proveedor.
- `models.providers.*.models.*.contextWindow`: metadatos nativos de la ventana de contexto del modelo.
- `models.providers.*.models.*.contextTokens`: límite opcional de contexto en runtime. Úsalo cuando quieras un presupuesto de contexto efectivo menor que el `contextWindow` nativo del modelo.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: pista opcional de compatibilidad. Para `api: "openai-completions"` con un `baseUrl` no nativo no vacío (host distinto de `api.openai.com`), OpenClaw lo fuerza a `false` en runtime. Un `baseUrl` vacío/omitido conserva el comportamiento predeterminado de OpenAI.
- `models.providers.*.models.*.compat.requiresStringContent`: pista opcional de compatibilidad para endpoints de chat compatibles con OpenAI que solo aceptan cadenas. Cuando es `true`, OpenClaw aplana arreglos `messages[].content` de solo texto en cadenas simples antes de enviar la solicitud.
- `plugins.entries.amazon-bedrock.config.discovery`: raíz de ajustes de auto-descubrimiento de Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: activar/desactivar el descubrimiento implícito.
- `plugins.entries.amazon-bedrock.config.discovery.region`: región de AWS para el descubrimiento.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: filtro opcional de ID de proveedor para descubrimiento dirigido.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: intervalo de sondeo para la actualización del descubrimiento.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: ventana de contexto de respaldo para modelos descubiertos.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: tokens máximos de salida de respaldo para modelos descubiertos.

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

Configura `OPENCODE_API_KEY` (o `OPENCODE_ZEN_API_KEY`). Usa referencias `opencode/...` para el catálogo Zen o referencias `opencode-go/...` para el catálogo Go. Atajo: `openclaw onboard --auth-choice opencode-zen` o `openclaw onboard --auth-choice opencode-go`.

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
- Endpoint de programación (predeterminado): `https://api.z.ai/api/coding/paas/v4`
- Para el endpoint general, define un proveedor personalizado con la sustitución de base URL.

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
`openai-completions`, y OpenClaw basa ese comportamiento en las capacidades del endpoint
en lugar de hacerlo solo en el ID del proveedor integrado.

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

<Accordion title="Synthetic (compatible con Anthropic)">

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

La base URL debe omitir `/v1` (el cliente de Anthropic lo agrega). Atajo: `openclaw onboard --auth-choice synthetic-api-key`.

</Accordion>

<Accordion title="MiniMax M2.7 (directo)">

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
El catálogo de modelos usa por defecto solo M2.7.
En la ruta de streaming compatible con Anthropic, OpenClaw desactiva el thinking de MiniMax
por defecto salvo que configures `thinking` explícitamente. `/fast on` o
`params.fastMode: true` reescribe `MiniMax-M2.7` como
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="Modelos locales (LM Studio)">

Consulta [Local Models](/es/gateway/local-models). En resumen: ejecuta un modelo local grande a través de la API Responses de LM Studio en hardware serio; mantén los modelos alojados fusionados como respaldo.

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
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // o cadena en texto plano
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: lista opcional de Skills agrupadas únicamente (no afecta a Skills administradas/del workspace).
- `load.extraDirs`: raíces extra de Skills compartidas (menor precedencia).
- `install.preferBrew`: cuando es true, prefiere instaladores Homebrew cuando `brew` está
  disponible antes de volver a otros tipos de instalador.
- `install.nodeManager`: preferencia del instalador Node para especificaciones
  `metadata.openclaw.install` (`npm` | `pnpm` | `yarn` | `bun`).
- `entries.<skillKey>.enabled: false` desactiva una Skill incluso si está agrupada/instalada.
- `entries.<skillKey>.apiKey`: campo práctico de API key para Skills que declaran una variable de entorno principal (cadena en texto plano u objeto SecretRef).

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

- Se cargan desde `~/.openclaw/extensions`, `<workspace>/.openclaw/extensions` y `plugins.load.paths`.
- El descubrimiento acepta plugins nativos de OpenClaw y bundles compatibles de Codex y Claude, incluidos bundles de Claude sin manifiesto y con diseño predeterminado.
- **Los cambios de configuración requieren reiniciar el gateway.**
- `allow`: lista opcional de permisos (solo se cargan los plugins listados). `deny` prevalece.
- `plugins.entries.<id>.apiKey`: campo práctico de API key a nivel de plugin (cuando el plugin lo admite).
- `plugins.entries.<id>.env`: mapa de variables de entorno con alcance de plugin.
- `plugins.entries.<id>.hooks.allowPromptInjection`: cuando es `false`, el núcleo bloquea `before_prompt_build` e ignora campos heredados que mutan el prompt desde `before_agent_start`, mientras conserva `modelOverride` y `providerOverride` heredados. Se aplica a hooks de plugins nativos y a directorios de hooks proporcionados por bundles compatibles.
- `plugins.entries.<id>.subagent.allowModelOverride`: confía explícitamente en este plugin para solicitar sustituciones por ejecución de `provider` y `model` para ejecuciones en segundo plano de subagentes.
- `plugins.entries.<id>.subagent.allowedModels`: lista opcional de objetivos canónicos `provider/model` permitidos para sustituciones de subagente confiables. Usa `"*"` solo cuando quieras permitir intencionalmente cualquier modelo.
- `plugins.entries.<id>.config`: objeto de configuración definido por el plugin (validado por el esquema del plugin nativo de OpenClaw cuando está disponible).
- `plugins.entries.firecrawl.config.webFetch`: ajustes del proveedor web-fetch de Firecrawl.
  - `apiKey`: API key de Firecrawl (acepta SecretRef). Usa como respaldo `plugins.entries.firecrawl.config.webSearch.apiKey`, el valor heredado `tools.web.fetch.firecrawl.apiKey` o la variable de entorno `FIRECRAWL_API_KEY`.
  - `baseUrl`: base URL de la API de Firecrawl (predeterminado: `https://api.firecrawl.dev`).
  - `onlyMainContent`: extraer solo el contenido principal de las páginas (predeterminado: `true`).
  - `maxAgeMs`: antigüedad máxima de caché en milisegundos (predeterminado: `172800000` / 2 días).
  - `timeoutSeconds`: timeout de la solicitud de scrape en segundos (predeterminado: `60`).
- `plugins.entries.xai.config.xSearch`: ajustes de xAI X Search (búsqueda web de Grok).
  - `enabled`: habilita el proveedor X Search.
  - `model`: modelo Grok que se usará para la búsqueda (p. ej. `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming`: ajustes de memory dreaming (experimental). Consulta [Dreaming](/es/concepts/dreaming) para fases y umbrales.
  - `enabled`: interruptor maestro de dreaming (predeterminado `false`).
  - `frequency`: cadencia cron para cada barrido completo de dreaming (`"0 3 * * *"` por defecto).
  - La política de fases y los umbrales son detalles de implementación (no claves de configuración orientadas al usuario).
- La configuración completa de memoria se encuentra en [Memory configuration reference](/es/reference/memory-config):
  - `agents.defaults.memorySearch.*`
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- Los plugins de bundle Claude habilitados también pueden aportar valores predeterminados Pi integrados desde `settings.json`; OpenClaw los aplica como ajustes saneados del agente, no como parches de configuración sin procesar de OpenClaw.
- `plugins.slots.memory`: elige el ID del plugin de memoria activo, o `"none"` para desactivar los plugins de memoria.
- `plugins.slots.contextEngine`: elige el ID del plugin de motor de contexto activo; usa por defecto `"legacy"` salvo que instales y selecciones otro motor.
- `plugins.installs`: metadatos de instalación administrados por CLI usados por `openclaw plugins update`.
  - Incluye `source`, `spec`, `sourcePath`, `installPath`, `version`, `resolvedName`, `resolvedVersion`, `resolvedSpec`, `integrity`, `shasum`, `resolvedAt`, `installedAt`.
  - Trata `plugins.installs.*` como estado administrado; prefiere comandos CLI sobre ediciones manuales.

Consulta [Plugins](/es/tools/plugin).

---

## Navegador

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // modo predeterminado de red de confianza
      // allowPrivateNetwork: true, // alias heredado
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

- `evaluateEnabled: false` desactiva `act:evaluate` y `wait --fn`.
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` usa `true` por defecto cuando no está configurado (modelo de red de confianza).
- Establece `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` para una navegación estricta del navegador solo en redes públicas.
- En modo estricto, los endpoints remotos de perfil CDP (`profiles.*.cdpUrl`) están sujetos al mismo bloqueo de red privada durante las comprobaciones de alcance/detección.
- `ssrfPolicy.allowPrivateNetwork` sigue siendo compatible como alias heredado.
- En modo estricto, usa `ssrfPolicy.hostnameAllowlist` y `ssrfPolicy.allowedHostnames` para excepciones explícitas.
- Los perfiles remotos son solo de adjunción (start/stop/reset desactivados).
- `profiles.*.cdpUrl` acepta `http://`, `https://`, `ws://` y `wss://`.
  Usa HTTP(S) cuando quieras que OpenClaw descubra `/json/version`; usa WS(S)
  cuando tu proveedor te dé una URL WebSocket directa de DevTools.
- Los perfiles `existing-session` son solo del host y usan Chrome MCP en lugar de CDP.
- Los perfiles `existing-session` pueden establecer `userDataDir` para apuntar a un perfil específico
  de navegador basado en Chromium como Brave o Edge.
- Los perfiles `existing-session` conservan los límites actuales de ruta de Chrome MCP:
  acciones basadas en snapshot/ref en lugar de direccionamiento por selector CSS, hooks de carga
  de un solo archivo, sin sustituciones de timeout de diálogos, sin `wait --load networkidle`,
  y sin `responsebody`, exportación PDF, interceptación de descargas o acciones por lotes.
- Los perfiles locales administrados `openclaw` asignan automáticamente `cdpPort` y `cdpUrl`; solo
  configura `cdpUrl` explícitamente para CDP remoto.
- Orden de autodetección: navegador predeterminado si es Chromium-based → Chrome → Brave → Edge → Chromium → Chrome Canary.
- Servicio de control: solo loopback (puerto derivado de `gateway.port`, predeterminado `18791`).
- `extraArgs` agrega flags extra de inicio al Chromium local (por ejemplo
  `--disable-gpu`, tamaño de ventana o flags de depuración).

---

## UI

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // emoji, texto corto, URL de imagen o URI de datos
    },
  },
}
```

- `seamColor`: color de acento para la UI nativa de la app (tono de la burbuja del modo Talk, etc.).
- `assistant`: sustitución de identidad para Control UI. Usa como respaldo la identidad del agente activo.

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
      // password: "your-password", // o OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // para mode=trusted-proxy; consulta /gateway/trusted-proxy-auth
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
      // allowedOrigins: ["https://control.example.com"], // obligatorio para Control UI no loopback
      // dangerouslyAllowHostHeaderOriginFallback: false, // modo peligroso de respaldo de origen por encabezado Host
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
    // Opcional. Predeterminado false.
    allowRealIpFallback: false,
    tools: {
      // Denegaciones HTTP adicionales de /tools/invoke
      deny: ["browser"],
      // Eliminar herramientas de la lista predeterminada de denegación HTTP
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

<Accordion title="Detalles de campos del gateway">

- `mode`: `local` (ejecuta gateway) o `remote` (conectarse a un gateway remoto). El gateway se niega a iniciarse salvo que sea `local`.
- `port`: puerto multiplexado único para WS + HTTP. Precedencia: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: `auto`, `loopback` (predeterminado), `lan` (`0.0.0.0`), `tailnet` (solo IP de Tailscale) o `custom`.
- **Alias heredados de bind**: usa valores de modo bind en `gateway.bind` (`auto`, `loopback`, `lan`, `tailnet`, `custom`), no alias de host (`