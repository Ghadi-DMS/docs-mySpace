---
read_when:
    - Configurar la compatibilidad con iMessage
    - Depurar el envío y la recepción de iMessage
summary: Compatibilidad heredada con iMessage mediante imsg (JSON-RPC sobre stdio). Las configuraciones nuevas deben usar BlueBubbles.
title: iMessage
x-i18n:
    generated_at: "2026-04-05T12:35:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 086d85bead49f75d12ae6b14ac917af52375b6afd28f6af1a0dcbbc7fcb628a0
    source_path: channels/imessage.md
    workflow: 15
---

# iMessage (heredado: imsg)

<Warning>
Para nuevas implementaciones de iMessage, usa <a href="/channels/bluebubbles">BlueBubbles</a>.

La integración `imsg` es heredada y puede eliminarse en una versión futura.
</Warning>

Estado: integración heredada de CLI externa. Gateway inicia `imsg rpc` y se comunica por JSON-RPC sobre stdio (sin daemon/puerto independiente).

<CardGroup cols={3}>
  <Card title="BlueBubbles (recomendado)" icon="message-circle" href="/channels/bluebubbles">
    Ruta preferida de iMessage para configuraciones nuevas.
  </Card>
  <Card title="Emparejamiento" icon="link" href="/channels/pairing">
    Los mensajes directos de iMessage usan el modo de emparejamiento de forma predeterminada.
  </Card>
  <Card title="Referencia de configuración" icon="settings" href="/gateway/configuration-reference#imessage">
    Referencia completa de campos de iMessage.
  </Card>
</CardGroup>

## Configuración rápida

<Tabs>
  <Tab title="Mac local (ruta rápida)">
    <Steps>
      <Step title="Instala y verifica imsg">

```bash
brew install steipete/tap/imsg
imsg rpc --help
```

      </Step>

      <Step title="Configura OpenClaw">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/<you>/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="Inicia gateway">

```bash
openclaw gateway
```

      </Step>

      <Step title="Aprueba el primer emparejamiento de mensaje directo (dmPolicy predeterminada)">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        Las solicitudes de emparejamiento vencen después de 1 hora.
      </Step>
    </Steps>

  </Tab>

  <Tab title="Mac remota por SSH">
    OpenClaw solo requiere un `cliPath` compatible con stdio, por lo que puedes hacer que `cliPath` apunte a un script contenedor que use SSH a una Mac remota y ejecute `imsg`.

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

    Configuración recomendada cuando los archivos adjuntos están habilitados:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // usado para recuperar archivos adjuntos por SCP
      includeAttachments: true,
      // Opcional: reemplaza las raíces de archivos adjuntos permitidas.
      // Los valores predeterminados incluyen /Users/*/Library/Messages/Attachments
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    Si `remoteHost` no está configurado, OpenClaw intenta detectarlo automáticamente analizando el script contenedor SSH.
    `remoteHost` debe ser `host` o `user@host` (sin espacios ni opciones de SSH).
    OpenClaw usa verificación estricta de clave de host para SCP, por lo que la clave del host de retransmisión ya debe existir en `~/.ssh/known_hosts`.
    Las rutas de archivos adjuntos se validan con respecto a las raíces permitidas (`attachmentRoots` / `remoteAttachmentRoots`).

  </Tab>
</Tabs>

## Requisitos y permisos (macOS)

- Messages debe tener sesión iniciada en la Mac que ejecuta `imsg`.
- Se requiere Full Disk Access para el contexto de proceso que ejecuta OpenClaw/`imsg` (acceso a la base de datos de Messages).
- Se requiere permiso de Automation para enviar mensajes a través de Messages.app.

<Tip>
Los permisos se conceden por contexto de proceso. Si gateway se ejecuta sin interfaz (LaunchAgent/SSH), ejecuta un comando interactivo una vez en ese mismo contexto para activar las solicitudes:

```bash
imsg chats --limit 1
# o
imsg send <handle> "test"
```

</Tip>

## Control de acceso y enrutamiento

<Tabs>
  <Tab title="Política de mensajes directos">
    `channels.imessage.dmPolicy` controla los mensajes directos:

    - `pairing` (predeterminado)
    - `allowlist`
    - `open` (requiere que `allowFrom` incluya `"*"`)
    - `disabled`

    Campo de allowlist: `channels.imessage.allowFrom`.

    Las entradas de la allowlist pueden ser identificadores o destinos de chat (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`).

  </Tab>

  <Tab title="Política de grupos y menciones">
    `channels.imessage.groupPolicy` controla el manejo de grupos:

    - `allowlist` (predeterminado cuando está configurado)
    - `open`
    - `disabled`

    Allowlist de remitentes de grupo: `channels.imessage.groupAllowFrom`.

    Respaldo en tiempo de ejecución: si `groupAllowFrom` no está configurado, las comprobaciones de remitentes de grupos de iMessage recurren a `allowFrom` cuando está disponible.
    Nota de tiempo de ejecución: si `channels.imessage` falta por completo, el tiempo de ejecución recurre a `groupPolicy="allowlist"` y registra una advertencia (incluso si `channels.defaults.groupPolicy` está configurado).

    Filtrado por menciones para grupos:

    - iMessage no tiene metadatos nativos de mención
    - la detección de menciones usa patrones regex (`agents.list[].groupChat.mentionPatterns`, con respaldo en `messages.groupChat.mentionPatterns`)
    - sin patrones configurados, el filtrado por menciones no puede aplicarse

    Los comandos de control de remitentes autorizados pueden omitir el filtrado por menciones en grupos.

  </Tab>

  <Tab title="Sesiones y respuestas deterministas">
    - Los mensajes directos usan enrutamiento directo; los grupos usan enrutamiento de grupo.
    - Con el valor predeterminado `session.dmScope=main`, los mensajes directos de iMessage se agrupan en la sesión principal del agente.
    - Las sesiones de grupo están aisladas (`agent:<agentId>:imessage:group:<chat_id>`).
    - Las respuestas se enrutan de vuelta a iMessage usando los metadatos del canal/destino de origen.

    Comportamiento de hilo similar a grupo:

    Algunos hilos de iMessage con varios participantes pueden llegar con `is_group=false`.
    Si ese `chat_id` está configurado explícitamente en `channels.imessage.groups`, OpenClaw lo trata como tráfico de grupo (filtrado de grupo + aislamiento de sesión de grupo).

  </Tab>
</Tabs>

## Bindings de conversaciones ACP

Los chats heredados de iMessage también pueden vincularse a sesiones ACP.

Flujo rápido para operadores:

- Ejecuta `/acp spawn codex --bind here` dentro del mensaje directo o del chat de grupo permitido.
- Los mensajes futuros en esa misma conversación de iMessage se enrutan a la sesión ACP iniciada.
- `/new` y `/reset` restablecen esa misma sesión ACP vinculada en su lugar.
- `/acp close` cierra la sesión ACP y elimina el binding.

Se admiten bindings persistentes configurados mediante entradas `bindings[]` de nivel superior con `type: "acp"` y `match.channel: "imessage"`.

`match.peer.id` puede usar:

- identificador normalizado de mensaje directo, como `+15555550123` o `user@example.com`
- `chat_id:<id>` (recomendado para bindings de grupo estables)
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

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
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

Consulta [Agentes ACP](/tools/acp-agents) para el comportamiento compartido de los bindings ACP.

## Patrones de implementación

<AccordionGroup>
  <Accordion title="Usuario bot dedicado de macOS (identidad de iMessage independiente)">
    Usa un Apple ID y un usuario de macOS dedicados para que el tráfico del bot quede aislado de tu perfil personal de Messages.

    Flujo típico:

    1. Crea o inicia sesión en un usuario dedicado de macOS.
    2. Inicia sesión en Messages con el Apple ID del bot en ese usuario.
    3. Instala `imsg` en ese usuario.
    4. Crea un script contenedor SSH para que OpenClaw pueda ejecutar `imsg` en el contexto de ese usuario.
    5. Haz que `channels.imessage.accounts.<id>.cliPath` y `.dbPath` apunten a ese perfil de usuario.

    La primera ejecución puede requerir aprobaciones en la GUI (Automation + Full Disk Access) en la sesión de ese usuario bot.

  </Accordion>

  <Accordion title="Mac remota por Tailscale (ejemplo)">
    Topología habitual:

    - gateway se ejecuta en Linux/VM
    - iMessage + `imsg` se ejecuta en una Mac de tu tailnet
    - el contenedor `cliPath` usa SSH para ejecutar `imsg`
    - `remoteHost` habilita la recuperación de archivos adjuntos por SCP

    Ejemplo:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
      includeAttachments: true,
      dbPath: "/Users/bot/Library/Messages/chat.db",
    },
  },
}
```

```bash
#!/usr/bin/env bash
exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
```

    Usa claves SSH para que tanto SSH como SCP no sean interactivos.
    Asegúrate de que la clave del host sea de confianza primero (por ejemplo `ssh bot@mac-mini.tailnet-1234.ts.net`) para que se complete `known_hosts`.

  </Accordion>

  <Accordion title="Patrón de varias cuentas">
    iMessage admite configuración por cuenta en `channels.imessage.accounts`.

    Cada cuenta puede reemplazar campos como `cliPath`, `dbPath`, `allowFrom`, `groupPolicy`, `mediaMaxMb`, configuración del historial y allowlists de raíces de archivos adjuntos.

  </Accordion>
</AccordionGroup>

## Contenido multimedia, fragmentación y destinos de entrega

<AccordionGroup>
  <Accordion title="Archivos adjuntos y multimedia">
    - la ingesta de archivos adjuntos entrantes es opcional: `channels.imessage.includeAttachments`
    - las rutas remotas de archivos adjuntos pueden recuperarse por SCP cuando `remoteHost` está configurado
    - las rutas de archivos adjuntos deben coincidir con las raíces permitidas:
      - `channels.imessage.attachmentRoots` (local)
      - `channels.imessage.remoteAttachmentRoots` (modo SCP remoto)
      - patrón de raíz predeterminado: `/Users/*/Library/Messages/Attachments`
    - SCP usa verificación estricta de clave de host (`StrictHostKeyChecking=yes`)
    - el tamaño del contenido multimedia saliente usa `channels.imessage.mediaMaxMb` (16 MB de forma predeterminada)
  </Accordion>

  <Accordion title="Fragmentación saliente">
    - límite de fragmento de texto: `channels.imessage.textChunkLimit` (4000 de forma predeterminada)
    - modo de fragmentación: `channels.imessage.chunkMode`
      - `length` (predeterminado)
      - `newline` (división por párrafos primero)
  </Accordion>

  <Accordion title="Formatos de direccionamiento">
    Destinos explícitos preferidos:

    - `chat_id:123` (recomendado para enrutamiento estable)
    - `chat_guid:...`
    - `chat_identifier:...`

    También se admiten destinos por identificador:

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

```bash
imsg chats --limit 20
```

  </Accordion>
</AccordionGroup>

## Escrituras de configuración

iMessage permite por defecto escrituras de configuración iniciadas por el canal (para `/config set|unset` cuando `commands.config: true`).

Deshabilitar:

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

## Solución de problemas

<AccordionGroup>
  <Accordion title="imsg no encontrado o RPC no compatible">
    Valida el binario y la compatibilidad con RPC:

```bash
imsg rpc --help
openclaw channels status --probe
```

    Si la sonda informa que RPC no es compatible, actualiza `imsg`.

  </Accordion>

  <Accordion title="Los mensajes directos se ignoran">
    Comprueba:

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - aprobaciones de emparejamiento (`openclaw pairing list imessage`)

  </Accordion>

  <Accordion title="Los mensajes de grupo se ignoran">
    Comprueba:

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - comportamiento de allowlist de `channels.imessage.groups`
    - configuración del patrón de menciones (`agents.list[].groupChat.mentionPatterns`)

  </Accordion>

  <Accordion title="Fallan los archivos adjuntos remotos">
    Comprueba:

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - autenticación por clave SSH/SCP desde el host gateway
    - la clave del host existe en `~/.ssh/known_hosts` en el host gateway
    - legibilidad de la ruta remota en la Mac que ejecuta Messages

  </Accordion>

  <Accordion title="Se omitieron las solicitudes de permisos de macOS">
    Vuelve a ejecutar en una terminal GUI interactiva en el mismo contexto de usuario/sesión y aprueba las solicitudes:

```bash
imsg chats --limit 1
imsg send <handle> "test"
```

    Confirma que Full Disk Access y Automation estén concedidos para el contexto de proceso que ejecuta OpenClaw/`imsg`.

  </Accordion>
</AccordionGroup>

## Punteros a la referencia de configuración

- [Referencia de configuración - iMessage](/gateway/configuration-reference#imessage)
- [Configuración de gateway](/gateway/configuration)
- [Emparejamiento](/channels/pairing)
- [BlueBubbles](/channels/bluebubbles)

## Relacionado

- [Descripción general de canales](/channels) — todos los canales compatibles
- [Emparejamiento](/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/channels/groups) — comportamiento del chat de grupo y filtrado por menciones
- [Enrutamiento de canales](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/gateway/security) — modelo de acceso y refuerzo
