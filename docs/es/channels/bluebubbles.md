---
read_when:
    - Configurar el canal BlueBubbles
    - Solucionar problemas de emparejamiento de webhooks
    - Configurar iMessage en macOS
summary: iMessage mediante el servidor macOS de BlueBubbles (envío/recepción REST, escritura, reacciones, emparejamiento, acciones avanzadas).
title: BlueBubbles
x-i18n:
    generated_at: "2026-04-05T12:35:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: ed8e59a165bdfb8fd794ee2ad6e4dacd44aa02d512312c5f2fd7d15f863380bb
    source_path: channels/bluebubbles.md
    workflow: 15
---

# BlueBubbles (REST de macOS)

Estado: plugin integrado que se comunica con el servidor macOS de BlueBubbles a través de HTTP. **Recomendado para la integración con iMessage** por su API más completa y su configuración más sencilla en comparación con el canal imsg heredado.

## Plugin integrado

Las versiones actuales de OpenClaw incluyen BlueBubbles, por lo que las compilaciones empaquetadas normales no
necesitan un paso separado de `openclaw plugins install`.

## Descripción general

- Se ejecuta en macOS mediante la aplicación auxiliar de BlueBubbles ([bluebubbles.app](https://bluebubbles.app)).
- Recomendado/probado: macOS Sequoia (15). macOS Tahoe (26) funciona; la edición actualmente está rota en Tahoe, y las actualizaciones del icono de grupo pueden informar éxito pero no sincronizarse.
- OpenClaw se comunica con él a través de su API REST (`GET /api/v1/ping`, `POST /message/text`, `POST /chat/:id/*`).
- Los mensajes entrantes llegan mediante webhooks; las respuestas salientes, los indicadores de escritura, las confirmaciones de lectura y los tapbacks son llamadas REST.
- Los archivos adjuntos y los stickers se incorporan como medios entrantes (y se exponen al agente cuando es posible).
- El emparejamiento/la lista de permitidos funciona igual que en otros canales (`/channels/pairing`, etc.) con `channels.bluebubbles.allowFrom` + códigos de emparejamiento.
- Las reacciones se muestran como eventos del sistema igual que en Slack/Telegram, para que los agentes puedan "mencionarlas" antes de responder.
- Funciones avanzadas: editar, deshacer envío, respuestas en hilo, efectos de mensajes, gestión de grupos.

## Inicio rápido

1. Instala el servidor BlueBubbles en tu Mac (sigue las instrucciones en [bluebubbles.app/install](https://bluebubbles.app/install)).
2. En la configuración de BlueBubbles, habilita la API web y establece una contraseña.
3. Ejecuta `openclaw onboard` y selecciona BlueBubbles, o configúralo manualmente:

   ```json5
   {
     channels: {
       bluebubbles: {
         enabled: true,
         serverUrl: "http://192.168.1.100:1234",
         password: "example-password",
         webhookPath: "/bluebubbles-webhook",
       },
     },
   }
   ```

4. Apunta los webhooks de BlueBubbles a tu gateway (ejemplo: `https://your-gateway-host:3000/bluebubbles-webhook?password=<password>`).
5. Inicia el gateway; registrará el controlador del webhook y comenzará el emparejamiento.

Nota de seguridad:

- Establece siempre una contraseña para el webhook.
- La autenticación del webhook siempre es obligatoria. OpenClaw rechaza las solicitudes de webhook de BlueBubbles a menos que incluyan una contraseña/guid que coincida con `channels.bluebubbles.password` (por ejemplo `?password=<password>` o `x-password`), independientemente de la topología de loopback/proxy.
- La autenticación por contraseña se comprueba antes de leer/analizar los cuerpos completos de los webhooks.

## Mantener Messages.app activa (VM / configuraciones sin interfaz)

Algunas configuraciones de VM de macOS / siempre activas pueden hacer que Messages.app quede “inactiva” (los eventos entrantes se detienen hasta que la aplicación se abre o pasa al primer plano). Una solución sencilla es **tocar Messages cada 5 minutos** usando un AppleScript + LaunchAgent.

### 1) Guardar el AppleScript

Guárdalo como:

- `~/Scripts/poke-messages.scpt`

Script de ejemplo (no interactivo; no roba el foco):

```applescript
try
  tell application "Messages"
    if not running then
      launch
    end if

    -- Touch the scripting interface to keep the process responsive.
    set _chatCount to (count of chats)
  end tell
on error
  -- Ignore transient failures (first-run prompts, locked session, etc).
end try
```

### 2) Instalar un LaunchAgent

Guárdalo como:

- `~/Library/LaunchAgents/com.user.poke-messages.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.user.poke-messages</string>

    <key>ProgramArguments</key>
    <array>
      <string>/bin/bash</string>
      <string>-lc</string>
      <string>/usr/bin/osascript &quot;$HOME/Scripts/poke-messages.scpt&quot;</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>StartInterval</key>
    <integer>300</integer>

    <key>StandardOutPath</key>
    <string>/tmp/poke-messages.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/poke-messages.err</string>
  </dict>
</plist>
```

Notas:

- Esto se ejecuta **cada 300 segundos** y **al iniciar sesión**.
- La primera ejecución puede activar avisos de **Automatización** de macOS (`osascript` → Messages). Apruébalos en la misma sesión de usuario que ejecuta el LaunchAgent.

Cárgalo:

```bash
launchctl unload ~/Library/LaunchAgents/com.user.poke-messages.plist 2>/dev/null || true
launchctl load ~/Library/LaunchAgents/com.user.poke-messages.plist
```

## Onboarding

BlueBubbles está disponible en el onboarding interactivo:

```
openclaw onboard
```

El asistente solicita:

- **URL del servidor** (obligatorio): dirección del servidor BlueBubbles (por ejemplo, `http://192.168.1.100:1234`)
- **Contraseña** (obligatorio): contraseña de la API de la configuración de BlueBubbles Server
- **Ruta del webhook** (opcional): por defecto `/bluebubbles-webhook`
- **Política de MD**: emparejamiento, lista de permitidos, abierto o deshabilitado
- **Lista de permitidos**: números de teléfono, correos electrónicos o destinos de chat

También puedes añadir BlueBubbles mediante la CLI:

```
openclaw channels add bluebubbles --http-url http://192.168.1.100:1234 --password <password>
```

## Control de acceso (MD + grupos)

MD:

- Predeterminado: `channels.bluebubbles.dmPolicy = "pairing"`.
- Los remitentes desconocidos reciben un código de emparejamiento; los mensajes se ignoran hasta ser aprobados (los códigos caducan después de 1 hora).
- Apruébalos mediante:
  - `openclaw pairing list bluebubbles`
  - `openclaw pairing approve bluebubbles <CODE>`
- El emparejamiento es el intercambio de tokens predeterminado. Detalles: [Emparejamiento](/channels/pairing)

Grupos:

- `channels.bluebubbles.groupPolicy = open | allowlist | disabled` (predeterminado: `allowlist`).
- `channels.bluebubbles.groupAllowFrom` controla quién puede activar en grupos cuando se establece `allowlist`.

### Enriquecimiento de nombres de contacto (macOS, opcional)

Los webhooks de grupo de BlueBubbles a menudo solo incluyen direcciones sin procesar de los participantes. Si quieres que el contexto `GroupMembers` muestre en su lugar nombres de contactos locales, puedes habilitar opcionalmente el enriquecimiento desde Contactos locales en macOS:

- `channels.bluebubbles.enrichGroupParticipantsFromContacts = true` habilita la búsqueda. Predeterminado: `false`.
- Las búsquedas solo se ejecutan después de que el acceso al grupo, la autorización de comandos y el control por mención hayan permitido el paso del mensaje.
- Solo se enriquecen los participantes telefónicos sin nombre.
- Los números de teléfono sin procesar siguen siendo la alternativa cuando no se encuentra ninguna coincidencia local.

```json5
{
  channels: {
    bluebubbles: {
      enrichGroupParticipantsFromContacts: true,
    },
  },
}
```

### Control por mención (grupos)

BlueBubbles admite control por mención para chats grupales, igualando el comportamiento de iMessage/WhatsApp:

- Usa `agents.list[].groupChat.mentionPatterns` (o `messages.groupChat.mentionPatterns`) para detectar menciones.
- Cuando `requireMention` está habilitado para un grupo, el agente solo responde cuando se le menciona.
- Los comandos de control de remitentes autorizados omiten el control por mención.

Configuración por grupo:

```json5
{
  channels: {
    bluebubbles: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123"],
      groups: {
        "*": { requireMention: true }, // predeterminado para todos los grupos
        "iMessage;-;chat123": { requireMention: false }, // anulación para un grupo específico
      },
    },
  },
}
```

### Control de comandos

- Los comandos de control (por ejemplo, `/config`, `/model`) requieren autorización.
- Usa `allowFrom` y `groupAllowFrom` para determinar la autorización de comandos.
- Los remitentes autorizados pueden ejecutar comandos de control incluso sin mencionar en grupos.

## Vinculaciones de conversaciones ACP

Los chats de BlueBubbles pueden convertirse en espacios de trabajo ACP duraderos sin cambiar la capa de transporte.

Flujo rápido del operador:

- Ejecuta `/acp spawn codex --bind here` dentro del MD o del chat grupal permitido.
- Los mensajes futuros en esa misma conversación de BlueBubbles se enrutan a la sesión ACP generada.
- `/new` y `/reset` restablecen la misma sesión ACP vinculada en su lugar.
- `/acp close` cierra la sesión ACP y elimina la vinculación.

También se admiten vinculaciones persistentes configuradas mediante entradas `bindings[]` de nivel superior con `type: "acp"` y `match.channel: "bluebubbles"`.

`match.peer.id` puede usar cualquier forma de destino BlueBubbles compatible:

- identificador de MD normalizado como `+15555550123` o `user@example.com`
- `chat_id:<id>`
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

Para vinculaciones de grupo estables, prefiere `chat_id:*` o `chat_identifier:*`.

Ejemplo:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "bluebubbles",
        accountId: "default",
        peer: { kind: "dm", id: "+15555550123" },
      },
      acp: { label: "codex-imessage" },
    },
  ],
}
```

Consulta [Agentes ACP](/tools/acp-agents) para conocer el comportamiento compartido de las vinculaciones ACP.

## Escritura + confirmaciones de lectura

- **Indicadores de escritura**: se envían automáticamente antes y durante la generación de la respuesta.
- **Confirmaciones de lectura**: controladas por `channels.bluebubbles.sendReadReceipts` (predeterminado: `true`).
- **Indicadores de escritura**: OpenClaw envía eventos de inicio de escritura; BlueBubbles borra la escritura automáticamente al enviar o por tiempo de espera (la detención manual mediante DELETE no es fiable).

```json5
{
  channels: {
    bluebubbles: {
      sendReadReceipts: false, // deshabilitar confirmaciones de lectura
    },
  },
}
```

## Acciones avanzadas

BlueBubbles admite acciones avanzadas de mensajes cuando están habilitadas en la configuración:

```json5
{
  channels: {
    bluebubbles: {
      actions: {
        reactions: true, // tapbacks (predeterminado: true)
        edit: true, // editar mensajes enviados (macOS 13+, roto en macOS 26 Tahoe)
        unsend: true, // deshacer envío de mensajes (macOS 13+)
        reply: true, // respuestas en hilo por GUID del mensaje
        sendWithEffect: true, // efectos de mensaje (slam, loud, etc.)
        renameGroup: true, // renombrar chats grupales
        setGroupIcon: true, // establecer icono/foto del chat grupal (inestable en macOS 26 Tahoe)
        addParticipant: true, // añadir participantes a grupos
        removeParticipant: true, // eliminar participantes de grupos
        leaveGroup: true, // salir de chats grupales
        sendAttachment: true, // enviar archivos adjuntos/medios
      },
    },
  },
}
```

Acciones disponibles:

- **react**: añadir/eliminar reacciones tapback (`messageId`, `emoji`, `remove`)
- **edit**: editar un mensaje enviado (`messageId`, `text`)
- **unsend**: deshacer el envío de un mensaje (`messageId`)
- **reply**: responder a un mensaje específico (`messageId`, `text`, `to`)
- **sendWithEffect**: enviar con efecto de iMessage (`text`, `to`, `effectId`)
- **renameGroup**: renombrar un chat grupal (`chatGuid`, `displayName`)
- **setGroupIcon**: establecer el icono/foto de un chat grupal (`chatGuid`, `media`) — inestable en macOS 26 Tahoe (la API puede devolver éxito, pero el icono no se sincroniza).
- **addParticipant**: añadir alguien a un grupo (`chatGuid`, `address`)
- **removeParticipant**: eliminar alguien de un grupo (`chatGuid`, `address`)
- **leaveGroup**: salir de un chat grupal (`chatGuid`)
- **upload-file**: enviar medios/archivos (`to`, `buffer`, `filename`, `asVoice`)
  - Notas de voz: establece `asVoice: true` con audio **MP3** o **CAF** para enviar como mensaje de voz de iMessage. BlueBubbles convierte MP3 → CAF al enviar notas de voz.
- Alias heredado: `sendAttachment` sigue funcionando, pero `upload-file` es el nombre canónico de la acción.

### IDs de mensajes (cortos frente a completos)

OpenClaw puede mostrar IDs de mensaje _cortos_ (por ejemplo, `1`, `2`) para ahorrar tokens.

- `MessageSid` / `ReplyToId` pueden ser IDs cortos.
- `MessageSidFull` / `ReplyToIdFull` contienen los IDs completos del proveedor.
- Los IDs cortos están en memoria; pueden caducar al reiniciar o al desalojar la caché.
- Las acciones aceptan `messageId` corto o completo, pero los IDs cortos producirán error si ya no están disponibles.

Usa IDs completos para automatizaciones y almacenamiento duraderos:

- Plantillas: `{{MessageSidFull}}`, `{{ReplyToIdFull}}`
- Contexto: `MessageSidFull` / `ReplyToIdFull` en las cargas útiles entrantes

Consulta [Configuración](/gateway/configuration) para ver las variables de plantilla.

## Streaming por bloques

Controla si las respuestas se envían como un único mensaje o se transmiten en bloques:

```json5
{
  channels: {
    bluebubbles: {
      blockStreaming: true, // habilitar streaming por bloques (desactivado de forma predeterminada)
    },
  },
}
```

## Medios + límites

- Los archivos adjuntos entrantes se descargan y almacenan en la caché de medios.
- Límite de medios mediante `channels.bluebubbles.mediaMaxMb` para medios entrantes y salientes (predeterminado: 8 MB).
- El texto saliente se fragmenta según `channels.bluebubbles.textChunkLimit` (predeterminado: 4000 caracteres).

## Referencia de configuración

Configuración completa: [Configuración](/gateway/configuration)

Opciones del proveedor:

- `channels.bluebubbles.enabled`: habilitar/deshabilitar el canal.
- `channels.bluebubbles.serverUrl`: URL base de la API REST de BlueBubbles.
- `channels.bluebubbles.password`: contraseña de la API.
- `channels.bluebubbles.webhookPath`: ruta del endpoint del webhook (predeterminado: `/bluebubbles-webhook`).
- `channels.bluebubbles.dmPolicy`: `pairing | allowlist | open | disabled` (predeterminado: `pairing`).
- `channels.bluebubbles.allowFrom`: lista de permitidos de MD (identificadores, correos electrónicos, números E.164, `chat_id:*`, `chat_guid:*`).
- `channels.bluebubbles.groupPolicy`: `open | allowlist | disabled` (predeterminado: `allowlist`).
- `channels.bluebubbles.groupAllowFrom`: lista de permitidos de remitentes de grupo.
- `channels.bluebubbles.enrichGroupParticipantsFromContacts`: en macOS, enriquecer opcionalmente a los participantes de grupo sin nombre desde Contactos locales después de que pasen los filtros. Predeterminado: `false`.
- `channels.bluebubbles.groups`: configuración por grupo (`requireMention`, etc.).
- `channels.bluebubbles.sendReadReceipts`: enviar confirmaciones de lectura (predeterminado: `true`).
- `channels.bluebubbles.blockStreaming`: habilitar streaming por bloques (predeterminado: `false`; necesario para respuestas en streaming).
- `channels.bluebubbles.textChunkLimit`: tamaño del fragmento saliente en caracteres (predeterminado: 4000).
- `channels.bluebubbles.chunkMode`: `length` (predeterminado) divide solo al superar `textChunkLimit`; `newline` divide en líneas en blanco (límites de párrafo) antes de fragmentar por longitud.
- `channels.bluebubbles.mediaMaxMb`: límite de medios entrantes/salientes en MB (predeterminado: 8).
- `channels.bluebubbles.mediaLocalRoots`: lista de permitidos explícita de directorios locales absolutos permitidos para rutas de medios locales salientes. El envío mediante rutas locales se deniega de forma predeterminada a menos que esto esté configurado. Anulación por cuenta: `channels.bluebubbles.accounts.<accountId>.mediaLocalRoots`.
- `channels.bluebubbles.historyLimit`: máximo de mensajes de grupo para el contexto (0 lo deshabilita).
- `channels.bluebubbles.dmHistoryLimit`: límite del historial de MD.
- `channels.bluebubbles.actions`: habilitar/deshabilitar acciones específicas.
- `channels.bluebubbles.accounts`: configuración de múltiples cuentas.

Opciones globales relacionadas:

- `agents.list[].groupChat.mentionPatterns` (o `messages.groupChat.mentionPatterns`).
- `messages.responsePrefix`.

## Direccionamiento / destinos de entrega

Prefiere `chat_guid` para un enrutamiento estable:

- `chat_guid:iMessage;-;+15555550123` (preferido para grupos)
- `chat_id:123`
- `chat_identifier:...`
- Identificadores directos: `+15555550123`, `user@example.com`
  - Si un identificador directo no tiene un chat de MD existente, OpenClaw creará uno mediante `POST /api/v1/chat/new`. Esto requiere que la API privada de BlueBubbles esté habilitada.

## Seguridad

- Las solicitudes de webhook se autentican comparando los parámetros de consulta o encabezados `guid`/`password` con `channels.bluebubbles.password`.
- Mantén en secreto la contraseña de la API y el endpoint del webhook (trátalos como credenciales).
- No existe omisión por localhost para la autenticación del webhook de BlueBubbles. Si haces proxy del tráfico del webhook, mantén la contraseña de BlueBubbles en la solicitud de extremo a extremo. `gateway.trustedProxies` no sustituye a `channels.bluebubbles.password` aquí. Consulta [Seguridad del gateway](/gateway/security#reverse-proxy-configuration).
- Habilita HTTPS + reglas de firewall en el servidor BlueBubbles si lo expones fuera de tu LAN.

## Solución de problemas

- Si los eventos de escritura/lectura dejan de funcionar, revisa los registros del webhook de BlueBubbles y verifica que la ruta del gateway coincida con `channels.bluebubbles.webhookPath`.
- Los códigos de emparejamiento caducan después de una hora; usa `openclaw pairing list bluebubbles` y `openclaw pairing approve bluebubbles <code>`.
- Las reacciones requieren la API privada de BlueBubbles (`POST /api/v1/message/react`); asegúrate de que la versión del servidor la exponga.
- Editar/deshacer envío requiere macOS 13+ y una versión compatible del servidor BlueBubbles. En macOS 26 (Tahoe), la edición actualmente está rota debido a cambios en la API privada.
- Las actualizaciones del icono de grupo pueden ser inestables en macOS 26 (Tahoe): la API puede devolver éxito, pero el nuevo icono no se sincroniza.
- OpenClaw oculta automáticamente las acciones conocidas como rotas según la versión de macOS del servidor BlueBubbles. Si la edición sigue apareciendo en macOS 26 (Tahoe), desactívala manualmente con `channels.bluebubbles.actions.edit=false`.
- Para información de estado/salud: `openclaw status --all` o `openclaw status --deep`.

Para una referencia general del flujo de trabajo de canales, consulta [Channels](/channels) y la guía de [Plugins](/tools/plugin).

## Relacionado

- [Resumen de canales](/channels) — todos los canales compatibles
- [Emparejamiento](/channels/pairing) — autenticación de MD y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento del chat grupal y control por mención
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y endurecimiento
