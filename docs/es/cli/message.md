---
read_when:
    - Agregando o modificando acciones de la CLI de mensajes
    - Cambiando el comportamiento saliente de los canales
summary: Referencia de la CLI para `openclaw message` (send + acciones de canal)
title: message
x-i18n:
    generated_at: "2026-04-05T12:38:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: b70f36189d028d59db25cd8b39d7c67883eaea71bea2358ee6314eec6cd2fa51
    source_path: cli/message.md
    workflow: 15
---

# `openclaw message`

Comando saliente único para enviar mensajes y acciones de canal
(Discord/Google Chat/iMessage/Matrix/Mattermost (plugin)/Microsoft Teams/Signal/Slack/Telegram/WhatsApp).

## Uso

```
openclaw message <subcommand> [flags]
```

Selección de canal:

- `--channel` es obligatorio si hay más de un canal configurado.
- Si exactamente un canal está configurado, se convierte en el valor predeterminado.
- Valores: `discord|googlechat|imessage|matrix|mattermost|msteams|signal|slack|telegram|whatsapp` (Mattermost requiere plugin)

Formatos de destino (`--target`):

- WhatsApp: E.164 o JID de grupo
- Telegram: chat id o `@username`
- Discord: `channel:<id>` o `user:<id>` (o mención `<@id>`; los id numéricos sin formato se tratan como canales)
- Google Chat: `spaces/<spaceId>` o `users/<userId>`
- Slack: `channel:<id>` o `user:<id>` (se acepta el id de canal sin formato)
- Mattermost (plugin): `channel:<id>`, `user:<id>` o `@username` (los id sin formato se tratan como canales)
- Signal: `+E.164`, `group:<id>`, `signal:+E.164`, `signal:group:<id>` o `username:<name>`/`u:<name>`
- iMessage: handle, `chat_id:<id>`, `chat_guid:<guid>` o `chat_identifier:<id>`
- Matrix: `@user:server`, `!room:server` o `#alias:server`
- Microsoft Teams: id de conversación (`19:...@thread.tacv2`) o `conversation:<id>` o `user:<aad-object-id>`

Búsqueda por nombre:

- Para proveedores compatibles (Discord/Slack/etc.), los nombres de canal como `Help` o `#help` se resuelven mediante la caché de directorio.
- Si hay un fallo de caché, OpenClaw intentará una búsqueda de directorio en vivo cuando el proveedor lo admita.

## Flags comunes

- `--channel <name>`
- `--account <id>`
- `--target <dest>` (canal o usuario de destino para send/poll/read/etc.)
- `--targets <name>` (repetible; solo broadcast)
- `--json`
- `--dry-run`
- `--verbose`

## Comportamiento de SecretRef

- `openclaw message` resuelve los SecretRef de canal compatibles antes de ejecutar la acción seleccionada.
- La resolución se limita al destino de la acción activa cuando es posible:
  - limitada al canal cuando se establece `--channel` (o se deduce a partir de destinos con prefijo como `discord:...`)
  - limitada a la cuenta cuando se establece `--account` (superficies globales del canal + superficies de la cuenta seleccionada)
  - cuando se omite `--account`, OpenClaw no fuerza un alcance de SecretRef de cuenta `default`
- Los SecretRef no resueltos en canales no relacionados no bloquean una acción de mensaje dirigida.
- Si el SecretRef del canal/cuenta seleccionado no está resuelto, el comando falla de forma cerrada para esa acción.

## Acciones

### Núcleo

- `send`
  - Canales: WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost (plugin)/Signal/iMessage/Matrix/Microsoft Teams
  - Obligatorio: `--target`, además de `--message` o `--media`
  - Opcional: `--media`, `--interactive`, `--buttons`, `--components`, `--card`, `--reply-to`, `--thread-id`, `--gif-playback`, `--force-document`, `--silent`
  - Cargas útiles interactivas compartidas: `--interactive` envía una carga útil JSON interactiva nativa del canal cuando es compatible
  - Solo Telegram: `--buttons` (requiere `channels.telegram.capabilities.inlineButtons` para permitirlo)
  - Solo Telegram: `--force-document` (envía imágenes y GIF como documentos para evitar la compresión de Telegram)
  - Solo Telegram: `--thread-id` (id de tema del foro)
  - Solo Slack: `--thread-id` (marca de tiempo del hilo; `--reply-to` usa el mismo campo)
  - Solo Discord: carga útil JSON `--components`
  - Canales con Adaptive Card: carga útil JSON `--card` cuando es compatible
  - Telegram + Discord: `--silent`
  - Solo WhatsApp: `--gif-playback`

- `poll`
  - Canales: WhatsApp/Telegram/Discord/Matrix/Microsoft Teams
  - Obligatorio: `--target`, `--poll-question`, `--poll-option` (repetible)
  - Opcional: `--poll-multi`
  - Solo Discord: `--poll-duration-hours`, `--silent`, `--message`
  - Solo Telegram: `--poll-duration-seconds` (5-600), `--silent`, `--poll-anonymous` / `--poll-public`, `--thread-id`

- `react`
  - Canales: Discord/Google Chat/Slack/Telegram/WhatsApp/Signal/Matrix
  - Obligatorio: `--message-id`, `--target`
  - Opcional: `--emoji`, `--remove`, `--participant`, `--from-me`, `--target-author`, `--target-author-uuid`
  - Nota: `--remove` requiere `--emoji` (omite `--emoji` para borrar las propias reacciones cuando sea compatible; consulta /tools/reactions)
  - Solo WhatsApp: `--participant`, `--from-me`
  - Reacciones de grupos de Signal: se requiere `--target-author` o `--target-author-uuid`

- `reactions`
  - Canales: Discord/Google Chat/Slack/Matrix
  - Obligatorio: `--message-id`, `--target`
  - Opcional: `--limit`

- `read`
  - Canales: Discord/Slack/Matrix
  - Obligatorio: `--target`
  - Opcional: `--limit`, `--before`, `--after`
  - Solo Discord: `--around`

- `edit`
  - Canales: Discord/Slack/Matrix
  - Obligatorio: `--message-id`, `--message`, `--target`

- `delete`
  - Canales: Discord/Slack/Telegram/Matrix
  - Obligatorio: `--message-id`, `--target`

- `pin` / `unpin`
  - Canales: Discord/Slack/Matrix
  - Obligatorio: `--message-id`, `--target`

- `pins` (list)
  - Canales: Discord/Slack/Matrix
  - Obligatorio: `--target`

- `permissions`
  - Canales: Discord/Matrix
  - Obligatorio: `--target`
  - Solo Matrix: disponible cuando el cifrado de Matrix está habilitado y se permiten acciones de verificación

- `search`
  - Canales: Discord
  - Obligatorio: `--guild-id`, `--query`
  - Opcional: `--channel-id`, `--channel-ids` (repetible), `--author-id`, `--author-ids` (repetible), `--limit`

### Hilos

- `thread create`
  - Canales: Discord
  - Obligatorio: `--thread-name`, `--target` (id de canal)
  - Opcional: `--message-id`, `--message`, `--auto-archive-min`

- `thread list`
  - Canales: Discord
  - Obligatorio: `--guild-id`
  - Opcional: `--channel-id`, `--include-archived`, `--before`, `--limit`

- `thread reply`
  - Canales: Discord
  - Obligatorio: `--target` (id de hilo), `--message`
  - Opcional: `--media`, `--reply-to`

### Emojis

- `emoji list`
  - Discord: `--guild-id`
  - Slack: sin flags adicionales

- `emoji upload`
  - Canales: Discord
  - Obligatorio: `--guild-id`, `--emoji-name`, `--media`
  - Opcional: `--role-ids` (repetible)

### Stickers

- `sticker send`
  - Canales: Discord
  - Obligatorio: `--target`, `--sticker-id` (repetible)
  - Opcional: `--message`

- `sticker upload`
  - Canales: Discord
  - Obligatorio: `--guild-id`, `--sticker-name`, `--sticker-desc`, `--sticker-tags`, `--media`

### Roles / Canales / Miembros / Voz

- `role info` (Discord): `--guild-id`
- `role add` / `role remove` (Discord): `--guild-id`, `--user-id`, `--role-id`
- `channel info` (Discord): `--target`
- `channel list` (Discord): `--guild-id`
- `member info` (Discord/Slack): `--user-id` (+ `--guild-id` para Discord)
- `voice status` (Discord): `--guild-id`, `--user-id`

### Eventos

- `event list` (Discord): `--guild-id`
- `event create` (Discord): `--guild-id`, `--event-name`, `--start-time`
  - Opcional: `--end-time`, `--desc`, `--channel-id`, `--location`, `--event-type`

### Moderación (Discord)

- `timeout`: `--guild-id`, `--user-id` (opcional `--duration-min` o `--until`; omite ambos para borrar el timeout)
- `kick`: `--guild-id`, `--user-id` (+ `--reason`)
- `ban`: `--guild-id`, `--user-id` (+ `--delete-days`, `--reason`)
  - `timeout` también admite `--reason`

### Broadcast

- `broadcast`
  - Canales: cualquier canal configurado; usa `--channel all` para apuntar a todos los proveedores
  - Obligatorio: `--targets <target...>`
  - Opcional: `--message`, `--media`, `--dry-run`

## Ejemplos

Enviar una respuesta en Discord:

```
openclaw message send --channel discord \
  --target channel:123 --message "hi" --reply-to 456
```

Enviar un mensaje de Discord con components:

```
openclaw message send --channel discord \
  --target channel:123 --message "Choose:" \
  --components '{"text":"Choose a path","blocks":[{"type":"actions","buttons":[{"label":"Approve","style":"success"},{"label":"Decline","style":"danger"}]}]}'
```

Consulta [Discord components](/channels/discord#interactive-components) para ver el esquema completo.

Enviar una carga útil interactiva compartida:

```bash
openclaw message send --channel googlechat --target spaces/AAA... \
  --message "Choose:" \
  --interactive '{"text":"Choose a path","blocks":[{"type":"actions","buttons":[{"label":"Approve"},{"label":"Decline"}]}]}'
```

Crear una encuesta en Discord:

```
openclaw message poll --channel discord \
  --target channel:123 \
  --poll-question "Snack?" \
  --poll-option Pizza --poll-option Sushi \
  --poll-multi --poll-duration-hours 48
```

Crear una encuesta en Telegram (cierre automático en 2 minutos):

```
openclaw message poll --channel telegram \
  --target @mychat \
  --poll-question "Lunch?" \
  --poll-option Pizza --poll-option Sushi \
  --poll-duration-seconds 120 --silent
```

Enviar un mensaje proactivo en Teams:

```
openclaw message send --channel msteams \
  --target conversation:19:abc@thread.tacv2 --message "hi"
```

Crear una encuesta en Teams:

```
openclaw message poll --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --poll-question "Lunch?" \
  --poll-option Pizza --poll-option Sushi
```

Reaccionar en Slack:

```
openclaw message react --channel slack \
  --target C123 --message-id 456 --emoji "✅"
```

Reaccionar en un grupo de Signal:

```
openclaw message react --channel signal \
  --target signal:group:abc123 --message-id 1737630212345 \
  --emoji "✅" --target-author-uuid 123e4567-e89b-12d3-a456-426614174000
```

Enviar botones inline de Telegram:

```
openclaw message send --channel telegram --target @mychat --message "Choose:" \
  --buttons '[ [{"text":"Yes","callback_data":"cmd:yes"}], [{"text":"No","callback_data":"cmd:no"}] ]'
```

Enviar una Adaptive Card de Teams:

```bash
openclaw message send --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --card '{"type":"AdaptiveCard","version":"1.5","body":[{"type":"TextBlock","text":"Status update"}]}'
```

Enviar una imagen de Telegram como documento para evitar la compresión:

```bash
openclaw message send --channel telegram --target @mychat \
  --media ./diagram.png --force-document
```
