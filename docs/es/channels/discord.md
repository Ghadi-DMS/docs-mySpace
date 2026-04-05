---
read_when:
    - Trabajar en funciones del canal de Discord
summary: Estado de compatibilidad, capacidades y configuración del bot de Discord
title: Discord
x-i18n:
    generated_at: "2026-04-05T12:36:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: e757d321d80d05642cd9e24b51fb47897bacaf8db19df83bd61a49a8ce51ed3a
    source_path: channels/discord.md
    workflow: 15
---

# Discord (API de bot)

Estado: listo para mensajes directos y canales de servidores mediante la gateway oficial de Discord.

<CardGroup cols={3}>
  <Card title="Emparejamiento" icon="link" href="/channels/pairing">
    Los mensajes directos de Discord usan el modo de emparejamiento de forma predeterminada.
  </Card>
  <Card title="Comandos slash" icon="terminal" href="/tools/slash-commands">
    Comportamiento nativo de comandos y catálogo de comandos.
  </Card>
  <Card title="Solución de problemas del canal" icon="wrench" href="/channels/troubleshooting">
    Diagnósticos y flujo de reparación entre canales.
  </Card>
</CardGroup>

## Configuración rápida

Necesitarás crear una nueva aplicación con un bot, añadir el bot a tu servidor y emparejarlo con OpenClaw. Recomendamos añadir tu bot a tu propio servidor privado. Si todavía no tienes uno, [crea uno primero](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (elige **Create My Own > For me and my friends**).

<Steps>
  <Step title="Crear una aplicación y un bot de Discord">
    Ve al [Portal para desarrolladores de Discord](https://discord.com/developers/applications) y haz clic en **New Application**. Asígnale un nombre como "OpenClaw".

    Haz clic en **Bot** en la barra lateral. Establece el **Username** con el nombre que uses para tu agente de OpenClaw.

  </Step>

  <Step title="Habilitar intents privilegiados">
    Sigue en la página **Bot**, desplázate hacia abajo hasta **Privileged Gateway Intents** y habilita:

    - **Message Content Intent** (obligatorio)
    - **Server Members Intent** (recomendado; obligatorio para listas de permitidos por rol y coincidencia de nombre a ID)
    - **Presence Intent** (opcional; solo se necesita para actualizaciones de presencia)

  </Step>

  <Step title="Copiar el token de tu bot">
    Vuelve a la parte superior de la página **Bot** y haz clic en **Reset Token**.

    <Note>
    A pesar del nombre, esto genera tu primer token; no se está “restableciendo” nada.
    </Note>

    Copia el token y guárdalo en algún lugar. Este es tu **Bot Token** y lo necesitarás en breve.

  </Step>

  <Step title="Generar una URL de invitación y añadir el bot a tu servidor">
    Haz clic en **OAuth2** en la barra lateral. Generarás una URL de invitación con los permisos correctos para añadir el bot a tu servidor.

    Desplázate hacia abajo hasta **OAuth2 URL Generator** y habilita:

    - `bot`
    - `applications.commands`

    Debajo aparecerá una sección **Bot Permissions**. Habilita:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (opcional)

    Copia la URL generada al final, pégala en tu navegador, selecciona tu servidor y haz clic en **Continue** para conectarlo. Ahora deberías ver tu bot en el servidor de Discord.

  </Step>

  <Step title="Habilitar el Developer Mode y recopilar tus IDs">
    De vuelta en la aplicación de Discord, necesitas habilitar el Developer Mode para poder copiar IDs internas.

    1. Haz clic en **User Settings** (icono de engranaje junto a tu avatar) → **Advanced** → activa **Developer Mode**
    2. Haz clic derecho en el **icono de tu servidor** en la barra lateral → **Copy Server ID**
    3. Haz clic derecho en **tu propio avatar** → **Copy User ID**

    Guarda tu **Server ID** y tu **User ID** junto a tu Bot Token; enviarás los tres a OpenClaw en el siguiente paso.

  </Step>

  <Step title="Permitir mensajes directos de miembros del servidor">
    Para que el emparejamiento funcione, Discord debe permitir que tu bot te envíe mensajes directos. Haz clic derecho en el **icono de tu servidor** → **Privacy Settings** → activa **Direct Messages**.

    Esto permite que los miembros del servidor (incluidos los bots) te envíen mensajes directos. Mantén esta opción habilitada si quieres usar mensajes directos de Discord con OpenClaw. Si solo planeas usar canales del servidor, puedes desactivar los mensajes directos después del emparejamiento.

  </Step>

  <Step title="Configura tu token del bot de forma segura (no lo envíes por chat)">
    El token de tu bot de Discord es un secreto (como una contraseña). Configúralo en la máquina que ejecuta OpenClaw antes de enviar mensajes a tu agente.

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set channels.discord.enabled true --strict-json
openclaw gateway
```

    Si OpenClaw ya se está ejecutando como servicio en segundo plano, reinícialo mediante la app de OpenClaw para Mac o deteniendo y reiniciando el proceso `openclaw gateway run`.

  </Step>

  <Step title="Configurar OpenClaw y emparejar">

    <Tabs>
      <Tab title="Pregúntale a tu agente">
        Chatea con tu agente de OpenClaw en cualquier canal existente (por ejemplo, Telegram) y díselo. Si Discord es tu primer canal, usa en su lugar la pestaña de CLI / configuración.

        > "Ya configuré mi token de bot de Discord en la configuración. Termina la configuración de Discord con User ID `<user_id>` y Server ID `<server_id>`."
      </Tab>
      <Tab title="CLI / configuración">
        Si prefieres la configuración basada en archivos, establece:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        Fallback de entorno para la cuenta predeterminada:

```bash
DISCORD_BOT_TOKEN=...
```

        Se admiten valores `token` en texto sin formato. También se admiten valores SecretRef para `channels.discord.token` en proveedores env/file/exec. Consulta [Gestión de secretos](/gateway/secrets).

      </Tab>
    </Tabs>

  </Step>

  <Step title="Aprobar el primer emparejamiento por mensaje directo">
    Espera hasta que la gateway esté en ejecución y luego envía un mensaje directo a tu bot en Discord. Responderá con un código de emparejamiento.

    <Tabs>
      <Tab title="Pregúntale a tu agente">
        Envía el código de emparejamiento a tu agente en tu canal existente:

        > "Aprueba este código de emparejamiento de Discord: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    Los códigos de emparejamiento caducan después de 1 hora.

    Ahora deberías poder chatear con tu agente en Discord por mensaje directo.

  </Step>
</Steps>

<Note>
La resolución de tokens distingue entre cuentas. Los valores de token en la configuración tienen prioridad sobre el fallback de entorno. `DISCORD_BOT_TOKEN` solo se usa para la cuenta predeterminada.
Para llamadas salientes avanzadas (herramienta de mensajes/acciones de canal), se usa un `token` explícito por llamada para esa llamada. Esto se aplica a acciones de envío y de lectura/sondeo (por ejemplo, read/search/fetch/thread/pins/permissions). La política de cuenta y la configuración de reintento siguen viniendo de la cuenta seleccionada en la instantánea activa del runtime.
</Note>

## Recomendado: configurar un espacio de trabajo de servidor

Una vez que los mensajes directos funcionen, puedes configurar tu servidor de Discord como un espacio de trabajo completo donde cada canal obtiene su propia sesión de agente con su propio contexto. Esto se recomienda para servidores privados donde solo estáis tú y tu bot.

<Steps>
  <Step title="Añadir tu servidor a la lista de permitidos de servidores">
    Esto permite que tu agente responda en cualquier canal de tu servidor, no solo en mensajes directos.

    <Tabs>
      <Tab title="Pregúntale a tu agente">
        > "Añade mi Server ID de Discord `<server_id>` a la lista de permitidos de servidores"
      </Tab>
      <Tab title="Configuración">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Permitir respuestas sin @mention">
    De forma predeterminada, tu agente solo responde en canales del servidor cuando se le menciona con @. En un servidor privado, probablemente querrás que responda a todos los mensajes.

    <Tabs>
      <Tab title="Pregúntale a tu agente">
        > "Permite que mi agente responda en este servidor sin necesidad de que se le mencione con @"
      </Tab>
      <Tab title="Configuración">
        Establece `requireMention: false` en la configuración de tu servidor:

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Planificar la memoria en canales del servidor">
    De forma predeterminada, la memoria a largo plazo (`MEMORY.md`) solo se carga en sesiones de mensajes directos. Los canales del servidor no cargan `MEMORY.md` automáticamente.

    <Tabs>
      <Tab title="Pregúntale a tu agente">
        > "Cuando haga preguntas en canales de Discord, usa memory_search o memory_get si necesitas contexto de largo plazo de MEMORY.md."
      </Tab>
      <Tab title="Manual">
        Si necesitas contexto compartido en cada canal, coloca las instrucciones estables en `AGENTS.md` o `USER.md` (se inyectan en cada sesión). Mantén las notas a largo plazo en `MEMORY.md` y accede a ellas cuando sea necesario con herramientas de memoria.
      </Tab>
    </Tabs>

  </Step>
</Steps>

Ahora crea algunos canales en tu servidor de Discord y empieza a chatear. Tu agente puede ver el nombre del canal, y cada canal obtiene su propia sesión aislada, así que puedes configurar `#coding`, `#home`, `#research` o lo que mejor se adapte a tu flujo de trabajo.

## Modelo de runtime

- La gateway es propietaria de la conexión de Discord.
- El enrutamiento de respuestas es determinista: las respuestas entrantes de Discord vuelven a Discord.
- De forma predeterminada (`session.dmScope=main`), los chats directos comparten la sesión principal del agente (`agent:main:main`).
- Los canales del servidor usan claves de sesión aisladas (`agent:<agentId>:discord:channel:<channelId>`).
- Los mensajes directos de grupo se ignoran de forma predeterminada (`channels.discord.dm.groupEnabled=false`).
- Los comandos slash nativos se ejecutan en sesiones de comandos aisladas (`agent:<agentId>:discord:slash:<userId>`), mientras siguen transportando `CommandTargetSessionKey` a la sesión de conversación enrutada.

## Canales de foro

Los canales de foro y multimedia de Discord solo aceptan publicaciones en hilos. OpenClaw admite dos formas de crearlos:

- Envía un mensaje al padre del foro (`channel:<forumId>`) para crear automáticamente un hilo. El título del hilo usa la primera línea no vacía de tu mensaje.
- Usa `openclaw message thread create` para crear un hilo directamente. No pases `--message-id` para canales de foro.

Ejemplo: enviar al padre del foro para crear un hilo

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

Ejemplo: crear un hilo de foro explícitamente

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

Los padres de foro no aceptan componentes de Discord. Si necesitas componentes, envía al propio hilo (`channel:<threadId>`).

## Componentes interactivos

OpenClaw admite contenedores de componentes v2 de Discord para mensajes del agente. Usa la herramienta de mensajes con una carga `components`. Los resultados de interacción se enrutan de vuelta al agente como mensajes entrantes normales y siguen la configuración existente de Discord `replyToMode`.

Bloques compatibles:

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- Las filas de acciones permiten hasta 5 botones o un único menú de selección
- Tipos de selección: `string`, `user`, `role`, `mentionable`, `channel`

De forma predeterminada, los componentes son de un solo uso. Establece `components.reusable=true` para permitir que botones, selectores y formularios se usen varias veces hasta que caduquen.

Para restringir quién puede hacer clic en un botón, establece `allowedUsers` en ese botón (IDs de usuario de Discord, tags o `*`). Cuando se configura, los usuarios no coincidentes reciben una denegación efímera.

Los comandos slash `/model` y `/models` abren un selector interactivo de modelos con listas desplegables de proveedor y modelo, además de un paso de envío. La respuesta del selector es efímera y solo el usuario que lo invocó puede usarla.

Archivos adjuntos:

- Los bloques `file` deben apuntar a una referencia de adjunto (`attachment://<filename>`)
- Proporciona el adjunto mediante `media`/`path`/`filePath` (archivo único); usa `media-gallery` para varios archivos
- Usa `filename` para sobrescribir el nombre de carga cuando deba coincidir con la referencia de adjunto

Formularios modales:

- Añade `components.modal` con hasta 5 campos
- Tipos de campo: `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- OpenClaw añade automáticamente un botón desencadenante

Ejemplo:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Optional fallback text",
  components: {
    reusable: true,
    text: "Choose a path",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Approve",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Decline", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Pick an option",
          options: [
            { label: "Option A", value: "a" },
            { label: "Option B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Details",
      triggerLabel: "Open form",
      fields: [
        { type: "text", label: "Requester" },
        {
          type: "select",
          label: "Priority",
          options: [
            { label: "Low", value: "low" },
            { label: "High", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## Control de acceso y enrutamiento

<Tabs>
  <Tab title="Política de mensajes directos">
    `channels.discord.dmPolicy` controla el acceso a mensajes directos (anterior: `channels.discord.dm.policy`):

    - `pairing` (predeterminado)
    - `allowlist`
    - `open` (requiere que `channels.discord.allowFrom` incluya `"*"`; anterior: `channels.discord.dm.allowFrom`)
    - `disabled`

    Si la política de mensajes directos no es abierta, los usuarios desconocidos se bloquean (o se les pide emparejamiento en modo `pairing`).

    Precedencia en varias cuentas:

    - `channels.discord.accounts.default.allowFrom` se aplica solo a la cuenta `default`.
    - Las cuentas con nombre heredan `channels.discord.allowFrom` cuando su propio `allowFrom` no está establecido.
    - Las cuentas con nombre no heredan `channels.discord.accounts.default.allowFrom`.

    Formato de destino de mensajes directos para entrega:

    - `user:<id>`
    - mención `<@id>`

    Los IDs numéricos sin prefijo son ambiguos y se rechazan a menos que se proporcione un tipo de destino explícito de usuario/canal.

  </Tab>

  <Tab title="Política de servidores">
    El manejo de servidores se controla mediante `channels.discord.groupPolicy`:

    - `open`
    - `allowlist`
    - `disabled`

    La línea base segura cuando existe `channels.discord` es `allowlist`.

    Comportamiento de `allowlist`:

    - el servidor debe coincidir con `channels.discord.guilds` (`id` preferido, slug aceptado)
    - listas de remitentes permitidos opcionales: `users` (se recomiendan IDs estables) y `roles` (solo IDs de rol); si se configura cualquiera de las dos, los remitentes están permitidos cuando coinciden con `users` O `roles`
    - la coincidencia directa de nombre/tag está desactivada de forma predeterminada; habilita `channels.discord.dangerouslyAllowNameMatching: true` solo como modo de compatibilidad de emergencia
    - se admiten nombres/tags para `users`, pero los IDs son más seguros; `openclaw security audit` avisa cuando se usan entradas de nombre/tag
    - si un servidor tiene `channels` configurado, se deniegan los canales no listados
    - si un servidor no tiene un bloque `channels`, se permiten todos los canales de ese servidor incluido en la lista de permitidos

    Ejemplo:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    Si solo estableces `DISCORD_BOT_TOKEN` y no creas un bloque `channels.discord`, el fallback del runtime es `groupPolicy="allowlist"` (con una advertencia en los registros), incluso si `channels.defaults.groupPolicy` es `open`.

  </Tab>

  <Tab title="Menciones y mensajes directos de grupo">
    Los mensajes de servidor están limitados por mención de forma predeterminada.

    La detección de menciones incluye:

    - mención explícita al bot
    - patrones de mención configurados (`agents.list[].groupChat.mentionPatterns`, fallback `messages.groupChat.mentionPatterns`)
    - comportamiento implícito de respuesta al bot en los casos compatibles

    `requireMention` se configura por servidor/canal (`channels.discord.guilds...`).
    `ignoreOtherMentions` opcionalmente descarta mensajes que mencionan a otro usuario/rol pero no al bot (excluyendo @everyone/@here).

    Mensajes directos de grupo:

    - predeterminado: ignorados (`dm.groupEnabled=false`)
    - lista de permitidos opcional mediante `dm.groupChannels` (IDs de canal o slugs)

  </Tab>
</Tabs>

### Enrutamiento de agentes basado en roles

Usa `bindings[].match.roles` para enrutar miembros de servidores de Discord a distintos agentes por ID de rol. Los bindings basados en roles aceptan solo IDs de rol y se evalúan después de los bindings peer o parent-peer y antes de los bindings solo de servidor. Si un binding también establece otros campos de coincidencia (por ejemplo `peer` + `guildId` + `roles`), todos los campos configurados deben coincidir.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Configuración del Developer Portal

<AccordionGroup>
  <Accordion title="Crear aplicación y bot">

    1. Discord Developer Portal -> **Applications** -> **New Application**
    2. **Bot** -> **Add Bot**
    3. Copia el token del bot

  </Accordion>

  <Accordion title="Intents privilegiados">
    En **Bot -> Privileged Gateway Intents**, habilita:

    - Message Content Intent
    - Server Members Intent (recomendado)

    Presence intent es opcional y solo es necesario si quieres recibir actualizaciones de presencia. Establecer la presencia del bot (`setPresence`) no requiere habilitar actualizaciones de presencia para miembros.

  </Accordion>

  <Accordion title="Scopes de OAuth y permisos base">
    Generador de URL de OAuth:

    - scopes: `bot`, `applications.commands`

    Permisos base típicos:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (opcional)

    Evita `Administrator` salvo que sea explícitamente necesario.

  </Accordion>

  <Accordion title="Copiar IDs">
    Habilita Discord Developer Mode y luego copia:

    - server ID
    - channel ID
    - user ID

    Prefiere IDs numéricos en la configuración de OpenClaw para auditorías y sondeos fiables.

  </Accordion>
</AccordionGroup>

## Comandos nativos y autenticación de comandos

- `commands.native` usa `"auto"` de forma predeterminada y está habilitado para Discord.
- Sobrescritura por canal: `channels.discord.commands.native`.
- `commands.native=false` borra explícitamente los comandos nativos de Discord registrados previamente.
- La autenticación de comandos nativos usa las mismas listas de permitidos/políticas de Discord que el manejo normal de mensajes.
- Los comandos pueden seguir siendo visibles en la interfaz de Discord para usuarios no autorizados; la ejecución sigue aplicando la autenticación de OpenClaw y devuelve "not authorized".

Consulta [Comandos slash](/tools/slash-commands) para ver el catálogo y el comportamiento de los comandos.

Configuración predeterminada de comandos slash:

- `ephemeral: true`

## Detalles de funciones

<AccordionGroup>
  <Accordion title="Etiquetas de respuesta y respuestas nativas">
    Discord admite etiquetas de respuesta en la salida del agente:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    Controlado por `channels.discord.replyToMode`:

    - `off` (predeterminado)
    - `first`
    - `all`

    Nota: `off` desactiva el encadenamiento implícito de respuestas. Las etiquetas explícitas `[[reply_to_*]]` siguen respetándose.

    Los IDs de mensaje se muestran en el contexto/historial para que los agentes puedan apuntar a mensajes específicos.

  </Accordion>

  <Accordion title="Vista previa de transmisión en vivo">
    OpenClaw puede transmitir borradores de respuesta enviando un mensaje temporal y editándolo a medida que llega el texto.

    - `channels.discord.streaming` controla la transmisión de vista previa (`off` | `partial` | `block` | `progress`, predeterminado: `off`).
    - El valor predeterminado sigue siendo `off` porque las ediciones de vista previa de Discord pueden alcanzar rápidamente los límites de tasa, especialmente cuando varios bots o gateways comparten la misma cuenta o tráfico del servidor.
    - `progress` se acepta para coherencia entre canales y se asigna a `partial` en Discord.
    - `channels.discord.streamMode` es un alias heredado y se migra automáticamente.
    - `partial` edita un único mensaje de vista previa a medida que llegan los tokens.
    - `block` emite fragmentos del tamaño de un borrador (usa `draftChunk` para ajustar tamaño y puntos de corte).

    Ejemplo:

```json5
{
  channels: {
    discord: {
      streaming: "partial",
    },
  },
}
```

    Valores predeterminados del troceado en modo `block` (limitados por `channels.discord.textChunkLimit`):

```json5
{
  channels: {
    discord: {
      streaming: "block",
      draftChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph",
      },
    },
  },
}
```

    La transmisión de vista previa es solo de texto; las respuestas multimedia vuelven a la entrega normal.

    Nota: la transmisión de vista previa es independiente de la transmisión por bloques. Cuando la transmisión por bloques está habilitada explícitamente para Discord, OpenClaw omite la vista previa para evitar la doble transmisión.

  </Accordion>

  <Accordion title="Historial, contexto y comportamiento de hilos">
    Contexto de historial del servidor:

    - `channels.discord.historyLimit` predeterminado `20`
    - fallback: `messages.groupChat.historyLimit`
    - `0` desactiva

    Controles de historial de mensajes directos:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    Comportamiento de hilos:

    - los hilos de Discord se enrutan como sesiones de canal
    - los metadatos del hilo padre pueden usarse para vinculación con la sesión padre
    - la configuración del hilo hereda la del canal padre salvo que exista una entrada específica del hilo

    Los temas de canal se inyectan como contexto **no confiable** (no como prompt del sistema).
    El contexto de respuesta y de mensaje citado actualmente permanece tal como se recibe.
    Las listas de permitidos de Discord controlan principalmente quién puede activar al agente, no un límite completo de redacción de contexto suplementario.

  </Accordion>

  <Accordion title="Sesiones vinculadas a hilos para subagentes">
    Discord puede vincular un hilo a un destino de sesión para que los mensajes posteriores de ese hilo sigan enrutándose a la misma sesión (incluidas sesiones de subagentes).

    Comandos:

    - `/focus <target>` vincula el hilo actual/nuevo a un objetivo de subagente/sesión
    - `/unfocus` elimina el vínculo del hilo actual
    - `/agents` muestra ejecuciones activas y estado del vínculo
    - `/session idle <duration|off>` inspecciona/actualiza la desvinculación automática por inactividad para vínculos enfocados
    - `/session max-age <duration|off>` inspecciona/actualiza la antigüedad máxima estricta para vínculos enfocados

    Configuración:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // opt-in
      },
    },
  },
}
```

    Notas:

    - `session.threadBindings.*` establece valores predeterminados globales.
    - `channels.discord.threadBindings.*` sobrescribe el comportamiento de Discord.
    - `spawnSubagentSessions` debe ser `true` para crear/vincular hilos automáticamente para `sessions_spawn({ thread: true })`.
    - `spawnAcpSessions` debe ser `true` para crear/vincular hilos automáticamente para ACP (`/acp spawn ... --thread ...` o `sessions_spawn({ runtime: "acp", thread: true })`).
    - Si los vínculos de hilo están deshabilitados para una cuenta, `/focus` y operaciones relacionadas no están disponibles.

    Consulta [Sub-agents](/tools/subagents), [ACP Agents](/tools/acp-agents) y [Referencia de configuración](/gateway/configuration-reference).

  </Accordion>

  <Accordion title="Vínculos persistentes de canal ACP">
    Para espacios de trabajo ACP estables y “siempre activos”, configura bindings ACP tipados de nivel superior dirigidos a conversaciones de Discord.

    Ruta de configuración:

    - `bindings[]` con `type: "acp"` y `match.channel: "discord"`

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
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    Notas:

    - `/acp spawn codex --bind here` vincula el canal o hilo actual de Discord en su lugar y mantiene los mensajes futuros enrutados a la misma sesión ACP.
    - Eso aún puede significar “iniciar una nueva sesión ACP de Codex”, pero no crea por sí mismo un nuevo hilo de Discord. El canal existente sigue siendo la superficie de chat.
    - Codex puede seguir ejecutándose en su propio `cwd` o espacio de trabajo backend en disco. Ese espacio de trabajo es estado de runtime, no un hilo de Discord.
    - Los mensajes de hilos pueden heredar el binding ACP del canal padre.
    - En un canal o hilo vinculado, `/new` y `/reset` restablecen la misma sesión ACP en el lugar.
    - Los vínculos temporales de hilo siguen funcionando y pueden sobrescribir la resolución de destino mientras estén activos.
    - `spawnAcpSessions` solo es obligatorio cuando OpenClaw necesita crear/vincular un hilo hijo mediante `--thread auto|here`. No es obligatorio para `/acp spawn ... --bind here` en el canal actual.

    Consulta [ACP Agents](/tools/acp-agents) para ver detalles del comportamiento de los bindings.

  </Accordion>

  <Accordion title="Notificaciones de reacciones">
    Modo de notificación de reacciones por servidor:

    - `off`
    - `own` (predeterminado)
    - `all`
    - `allowlist` (usa `guilds.<id>.users`)

    Los eventos de reacción se convierten en eventos del sistema y se adjuntan a la sesión de Discord enrutada.

  </Accordion>

  <Accordion title="Reacciones de confirmación">
    `ackReaction` envía un emoji de confirmación mientras OpenClaw procesa un mensaje entrante.

    Orden de resolución:

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - fallback del emoji de identidad del agente (`agents.list[].identity.emoji`, o "👀" en su defecto)

    Notas:

    - Discord acepta emoji unicode o nombres de emoji personalizados.
    - Usa `""` para desactivar la reacción en un canal o cuenta.

  </Accordion>

  <Accordion title="Escrituras de configuración">
    Las escrituras de configuración iniciadas por canal están habilitadas de forma predeterminada.

    Esto afecta a los flujos `/config set|unset` (cuando las funciones de comandos están habilitadas).

    Desactivar:

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Proxy de gateway">
    Enruta el tráfico WebSocket de la gateway de Discord y las búsquedas REST de arranque (ID de aplicación + resolución de lista de permitidos) a través de un proxy HTTP(S) con `channels.discord.proxy`.

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    Sobrescritura por cuenta:

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="Compatibilidad con PluralKit">
    Habilita la resolución de PluralKit para asignar mensajes proxificados a la identidad de miembro del sistema:

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // optional; needed for private systems
      },
    },
  },
}
```

    Notas:

    - las listas de permitidos pueden usar `pk:<memberId>`
    - los nombres para mostrar de miembros se comparan por nombre/slug solo cuando `channels.discord.dangerouslyAllowNameMatching: true`
    - las búsquedas usan el ID original del mensaje y están limitadas por ventana de tiempo
    - si la búsqueda falla, los mensajes proxificados se tratan como mensajes de bot y se descartan salvo que `allowBots=true`

  </Accordion>

  <Accordion title="Configuración de presencia">
    Las actualizaciones de presencia se aplican cuando estableces un campo de estado o actividad, o cuando habilitas la presencia automática.

    Ejemplo solo de estado:

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    Ejemplo de actividad (el tipo de actividad predeterminado es estado personalizado):

```json5
{
  channels: {
    discord: {
      activity: "Focus time",
      activityType: 4,
    },
  },
}
```

    Ejemplo de streaming:

```json5
{
  channels: {
    discord: {
      activity: "Live coding",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    Mapa de tipos de actividad:

    - 0: Playing
    - 1: Streaming (requiere `activityUrl`)
    - 2: Listening
    - 3: Watching
    - 4: Custom (usa el texto de la actividad como estado; el emoji es opcional)
    - 5: Competing

    Ejemplo de presencia automática (señal de salud del runtime):

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token exhausted",
      },
    },
  },
}
```

    La presencia automática asigna la disponibilidad del runtime al estado de Discord: healthy => online, degraded o unknown => idle, exhausted o unavailable => dnd. Sobrescrituras de texto opcionales:

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (admite el placeholder `{reason}`)

  </Accordion>

  <Accordion title="Aprobaciones en Discord">
    Discord admite el manejo de aprobaciones mediante botones en mensajes directos y, opcionalmente, puede publicar solicitudes de aprobación en el canal de origen.

    Ruta de configuración:

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (opcional; usa `commands.ownerAllowFrom` como fallback cuando es posible)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`, predeterminado: `dm`)
    - `agentFilter`, `sessionFilter`, `cleanupAfterResolve`

    Discord habilita automáticamente las aprobaciones nativas de ejecución cuando `enabled` no está establecido o vale `"auto"` y se puede resolver al menos un aprobador, ya sea desde `execApprovals.approvers` o desde `commands.ownerAllowFrom`. Discord no infiere aprobadores de ejecución desde `allowFrom` del canal, `dm.allowFrom` heredado ni `defaultTo` de mensaje directo. Establece `enabled: false` para desactivar explícitamente Discord como cliente nativo de aprobaciones.

    Cuando `target` es `channel` o `both`, la solicitud de aprobación es visible en el canal. Solo los aprobadores resueltos pueden usar los botones; otros usuarios reciben una denegación efímera. Las solicitudes de aprobación incluyen el texto del comando, así que habilita la entrega al canal solo en canales de confianza. Si no se puede derivar el ID del canal a partir de la clave de sesión, OpenClaw vuelve a la entrega por mensaje directo.

    Discord también representa los botones de aprobación compartidos usados por otros canales de chat. El adaptador nativo de Discord añade principalmente enrutamiento de mensajes directos a aprobadores y fanout a canales.
    Cuando esos botones están presentes, son la UX principal de aprobación; OpenClaw
    solo debe incluir un comando manual `/approve` cuando el resultado de la herramienta indica
    que las aprobaciones por chat no están disponibles o la aprobación manual es la única vía.

    La autenticación de gateway para este controlador usa el mismo contrato compartido de resolución de credenciales que otros clientes de Gateway:

    - autenticación local con prioridad de entorno (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` y luego `gateway.auth.*`)
    - en modo local, `gateway.remote.*` puede usarse como fallback solo cuando `gateway.auth.*` no está establecido; SecretRefs locales configurados pero no resueltos fallan de forma cerrada
    - compatibilidad con modo remoto mediante `gateway.remote.*` cuando corresponda
    - las sobrescrituras de URL son seguras para sobrescrituras: las sobrescrituras de CLI no reutilizan credenciales implícitas, y las sobrescrituras de entorno usan solo credenciales de entorno

    Comportamiento de resolución de aprobaciones:

    - los IDs con prefijo `plugin:` se resuelven mediante `plugin.approval.resolve`.
    - otros IDs se resuelven mediante `exec.approval.resolve`.
    - Discord no realiza aquí un salto extra de fallback de exec a plugin; el prefijo
      del ID decide qué método de gateway se llama.

    Las aprobaciones de ejecución caducan después de 30 minutos de forma predeterminada. Si las aprobaciones fallan con IDs de aprobación desconocidos, verifica la resolución de aprobadores, la habilitación de funciones y que el tipo de ID de aprobación entregado coincida con la solicitud pendiente.

    Documentación relacionada: [Exec approvals](/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## Herramientas y puertas de acciones

Las acciones de mensajes de Discord incluyen mensajería, administración de canales, moderación, presencia y acciones de metadatos.

Ejemplos principales:

- mensajería: `sendMessage`, `readMessages`, `editMessage`, `deleteMessage`, `threadReply`
- reacciones: `react`, `reactions`, `emojiList`
- moderación: `timeout`, `kick`, `ban`
- presencia: `setPresence`

Las puertas de acciones viven bajo `channels.discord.actions.*`.

Comportamiento predeterminado de puertas:

| Grupo de acciones                                                                                                                                                         | Predeterminado |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | habilitado     |
| roles                                                                                                                                                                     | deshabilitado  |
| moderation                                                                                                                                                                | deshabilitado  |
| presence                                                                                                                                                                  | deshabilitado  |

## Interfaz Components v2

OpenClaw usa Discord components v2 para aprobaciones de ejecución y marcadores entre contextos. Las acciones de mensajes de Discord también pueden aceptar `components` para una interfaz personalizada (avanzado; requiere construir una carga de componente mediante la herramienta de Discord), mientras que `embeds` heredados siguen disponibles pero no se recomiendan.

- `channels.discord.ui.components.accentColor` establece el color de acento usado por los contenedores de componentes de Discord (hexadecimal).
- Establécelo por cuenta con `channels.discord.accounts.<id>.ui.components.accentColor`.
- `embeds` se ignora cuando hay components v2 presentes.

Ejemplo:

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## Canales de voz

OpenClaw puede unirse a canales de voz de Discord para conversaciones continuas en tiempo real. Esto es independiente de los adjuntos de mensajes de voz.

Requisitos:

- Habilita comandos nativos (`commands.native` o `channels.discord.commands.native`).
- Configura `channels.discord.voice`.
- El bot necesita permisos Connect + Speak en el canal de voz de destino.

Usa el comando nativo exclusivo de Discord `/vc join|leave|status` para controlar sesiones. El comando usa el agente predeterminado de la cuenta y sigue las mismas reglas de lista de permitidos y política de grupo que otros comandos de Discord.

Ejemplo de unión automática:

```json5
{
  channels: {
    discord: {
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
    },
  },
}
```

Notas:

- `voice.tts` sobrescribe `messages.tts` solo para la reproducción de voz.
- Los turnos de transcripción de voz derivan el estado de propietario desde `allowFrom` de Discord (o `dm.allowFrom`); los hablantes que no son propietarios no pueden acceder a herramientas exclusivas del propietario (por ejemplo `gateway` y `cron`).
- La voz está habilitada de forma predeterminada; establece `channels.discord.voice.enabled=false` para desactivarla.
- `voice.daveEncryption` y `voice.decryptionFailureTolerance` se pasan a las opciones de unión de `@discordjs/voice`.
- Los valores predeterminados de `@discordjs/voice` son `daveEncryption=true` y `decryptionFailureTolerance=24` si no se establecen.
- OpenClaw también vigila los fallos de descifrado en recepción y se recupera automáticamente saliendo y volviendo a unirse al canal de voz tras fallos repetidos en una ventana corta.
- Si los registros de recepción muestran repetidamente `DecryptionFailed(UnencryptedWhenPassthroughDisabled)`, esto puede ser el bug de recepción ascendente de `@discordjs/voice` rastreado en [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419).

## Mensajes de voz

Los mensajes de voz de Discord muestran una vista previa de forma de onda y requieren audio OGG/Opus más metadatos. OpenClaw genera automáticamente la forma de onda, pero necesita que `ffmpeg` y `ffprobe` estén disponibles en el host de la gateway para inspeccionar y convertir archivos de audio.

Requisitos y restricciones:

- Proporciona una **ruta de archivo local** (las URL se rechazan).
- Omite contenido de texto (Discord no permite texto + mensaje de voz en la misma carga).
- Se acepta cualquier formato de audio; OpenClaw lo convierte a OGG/Opus cuando sea necesario.

Ejemplo:

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## Solución de problemas

<AccordionGroup>
  <Accordion title="Se usaron intents no permitidos o el bot no ve mensajes del servidor">

    - habilita Message Content Intent
    - habilita Server Members Intent cuando dependas de la resolución de usuario/miembro
    - reinicia la gateway después de cambiar intents

  </Accordion>

  <Accordion title="Los mensajes del servidor se bloquean inesperadamente">

    - verifica `groupPolicy`
    - verifica la lista de permitidos del servidor en `channels.discord.guilds`
    - si existe el mapa `channels` del servidor, solo se permiten los canales listados
    - verifica el comportamiento de `requireMention` y los patrones de mención

    Comprobaciones útiles:

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="Require mention es false pero sigue bloqueado">
    Causas comunes:

    - `groupPolicy="allowlist"` sin una lista de permitidos coincidente de servidor/canal
    - `requireMention` configurado en el lugar incorrecto (debe ir en `channels.discord.guilds` o en la entrada del canal)
    - remitente bloqueado por la lista de permitidos `users` del servidor/canal

  </Accordion>

  <Accordion title="Los controladores de larga duración exceden el tiempo o duplican respuestas">

    Registros típicos:

    - `Listener DiscordMessageListener timed out after 30000ms for event MESSAGE_CREATE`
    - `Slow listener detected ...`
    - `discord inbound worker timed out after ...`

    Ajuste del presupuesto del listener:

    - cuenta única: `channels.discord.eventQueue.listenerTimeout`
    - varias cuentas: `channels.discord.accounts.<accountId>.eventQueue.listenerTimeout`

    Ajuste del tiempo máximo de ejecución del worker:

    - cuenta única: `channels.discord.inboundWorker.runTimeoutMs`
    - varias cuentas: `channels.discord.accounts.<accountId>.inboundWorker.runTimeoutMs`
    - predeterminado: `1800000` (30 minutos); establece `0` para desactivar

    Línea base recomendada:

```json5
{
  channels: {
    discord: {
      accounts: {
        default: {
          eventQueue: {
            listenerTimeout: 120000,
          },
          inboundWorker: {
            runTimeoutMs: 1800000,
          },
        },
      },
    },
  },
}
```

    Usa `eventQueue.listenerTimeout` para configuración lenta del listener e `inboundWorker.runTimeoutMs`
    solo si quieres una válvula de seguridad independiente para turnos de agente en cola.

  </Accordion>

  <Accordion title="Desajustes en la auditoría de permisos">
    Las comprobaciones de permisos de `channels status --probe` solo funcionan para IDs numéricos de canal.

    Si usas claves slug, la coincidencia en runtime puede seguir funcionando, pero el sondeo no puede verificar completamente los permisos.

  </Accordion>

  <Accordion title="Problemas de mensajes directos y emparejamiento">

    - mensajes directos desactivados: `channels.discord.dm.enabled=false`
    - política de mensajes directos desactivada: `channels.discord.dmPolicy="disabled"` (anterior: `channels.discord.dm.policy`)
    - esperando aprobación de emparejamiento en modo `pairing`

  </Accordion>

  <Accordion title="Bucles de bot a bot">
    De forma predeterminada, los mensajes creados por bots se ignoran.

    Si estableces `channels.discord.allowBots=true`, usa reglas estrictas de mención y listas de permitidos para evitar comportamientos en bucle.
    Prefiere `channels.discord.allowBots="mentions"` para aceptar solo mensajes de bots que mencionan al bot.

  </Accordion>

  <Accordion title="La STT de voz pierde datos con DecryptionFailed(...)">

    - mantén OpenClaw actualizado (`openclaw update`) para que esté presente la lógica de recuperación de recepción de voz de Discord
    - confirma `channels.discord.voice.daveEncryption=true` (predeterminado)
    - empieza con `channels.discord.voice.decryptionFailureTolerance=24` (predeterminado ascendente) y ajusta solo si es necesario
    - vigila los registros para:
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - si los fallos continúan después del reingreso automático, recopila registros y compáralos con [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419)

  </Accordion>
</AccordionGroup>

## Indicadores de referencia de configuración

Referencia principal:

- [Referencia de configuración - Discord](/gateway/configuration-reference#discord)

Campos de Discord de alta señal:

- arranque/autenticación: `enabled`, `token`, `accounts.*`, `allowBots`
- política: `groupPolicy`, `dm.*`, `guilds.*`, `guilds.*.channels.*`
- comando: `commands.native`, `commands.useAccessGroups`, `configWrites`, `slashCommand.*`
- cola de eventos: `eventQueue.listenerTimeout` (presupuesto del listener), `eventQueue.maxQueueSize`, `eventQueue.maxConcurrency`
- worker entrante: `inboundWorker.runTimeoutMs`
- respuesta/historial: `replyToMode`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- entrega: `textChunkLimit`, `chunkMode`, `maxLinesPerMessage`
- transmisión: `streaming` (alias heredado: `streamMode`), `draftChunk`, `blockStreaming`, `blockStreamingCoalesce`
- medios/reintento: `mediaMaxMb`, `retry`
  - `mediaMaxMb` limita las cargas salientes de Discord (predeterminado: `8MB`)
- acciones: `actions.*`
- presencia: `activity`, `status`, `activityType`, `activityUrl`
- UI: `ui.components.accentColor`
- funciones: `threadBindings`, `bindings[]` de nivel superior (`type: "acp"`), `pluralkit`, `execApprovals`, `intents`, `agentComponents`, `heartbeat`, `responsePrefix`

## Seguridad y operaciones

- Trata los tokens de bot como secretos (`DISCORD_BOT_TOKEN` es preferible en entornos supervisados).
- Otorga permisos de Discord con privilegios mínimos.
- Si el despliegue/estado de comandos está desactualizado, reinicia la gateway y vuelve a comprobar con `openclaw channels status --probe`.

## Relacionado

- [Emparejamiento](/channels/pairing)
- [Grupos](/channels/groups)
- [Enrutamiento de canales](/channels/channel-routing)
- [Seguridad](/gateway/security)
- [Enrutamiento multiagente](/concepts/multi-agent)
- [Solución de problemas](/channels/troubleshooting)
- [Comandos slash](/tools/slash-commands)
