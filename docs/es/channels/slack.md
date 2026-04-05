---
read_when:
    - Configurar Slack o depurar el modo socket/HTTP de Slack
summary: Configuración de Slack y comportamiento en tiempo de ejecución (Socket Mode + HTTP Events API)
title: Slack
x-i18n:
    generated_at: "2026-04-05T12:37:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: efb37e1f04e1ac8ac3786c36ffc20013dacdc654bfa61e7f6e8df89c4902d2ab
    source_path: channels/slack.md
    workflow: 15
---

# Slack

Estado: listo para producción para mensajes directos y canales mediante integraciones de aplicaciones de Slack. El modo predeterminado es Socket Mode; también se admite el modo HTTP Events API.

<CardGroup cols={3}>
  <Card title="Emparejamiento" icon="link" href="/channels/pairing">
    Los mensajes directos de Slack usan el modo de emparejamiento de forma predeterminada.
  </Card>
  <Card title="Comandos slash" icon="terminal" href="/tools/slash-commands">
    Comportamiento nativo de comandos y catálogo de comandos.
  </Card>
  <Card title="Solución de problemas de canales" icon="wrench" href="/channels/troubleshooting">
    Diagnósticos multicanal y guías de reparación.
  </Card>
</CardGroup>

## Configuración rápida

<Tabs>
  <Tab title="Socket Mode (predeterminado)">
    <Steps>
      <Step title="Crea la app de Slack y los tokens">
        En la configuración de la app de Slack:

        - habilita **Socket Mode**
        - crea un **App Token** (`xapp-...`) con `connections:write`
        - instala la app y copia el **Bot Token** (`xoxb-...`)
      </Step>

      <Step title="Configura OpenClaw">

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: "xapp-...",
      botToken: "xoxb-...",
    },
  },
}
```

        Respaldo por variables de entorno (solo cuenta predeterminada):

```bash
SLACK_APP_TOKEN=xapp-...
SLACK_BOT_TOKEN=xoxb-...
```

      </Step>

      <Step title="Suscribe eventos de la app">
        Suscribe eventos del bot para:

        - `app_mention`
        - `message.channels`, `message.groups`, `message.im`, `message.mpim`
        - `reaction_added`, `reaction_removed`
        - `member_joined_channel`, `member_left_channel`
        - `channel_rename`
        - `pin_added`, `pin_removed`

        Habilita también la **Messages Tab** de App Home para mensajes directos.
      </Step>

      <Step title="Inicia gateway">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="Modo HTTP Events API">
    <Steps>
      <Step title="Configura la app de Slack para HTTP">

        - establece el modo en HTTP (`channels.slack.mode="http"`)
        - copia el **Signing Secret** de Slack
        - establece la URL de solicitud de Event Subscriptions, Interactivity y comandos slash en la misma ruta de webhook (predeterminada `/slack/events`)

      </Step>

      <Step title="Configura el modo HTTP de OpenClaw">

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: "xoxb-...",
      signingSecret: "your-signing-secret",
      webhookPath: "/slack/events",
    },
  },
}
```

      </Step>

      <Step title="Usa rutas de webhook únicas para HTTP con varias cuentas">
        Se admite el modo HTTP por cuenta.

        Da a cada cuenta un `webhookPath` distinto para que los registros no entren en conflicto.
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Lista de verificación de manifiesto y scopes

<AccordionGroup>
  <Accordion title="Ejemplo de manifiesto de app de Slack" defaultOpen>

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw",
      "always_online": true
    },
    "app_home": {
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Send a message to OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

  </Accordion>

  <Accordion title="Scopes opcionales de user token (operaciones de lectura)">
    Si configuras `channels.slack.userToken`, los scopes de lectura habituales son:

    - `channels:history`, `groups:history`, `im:history`, `mpim:history`
    - `channels:read`, `groups:read`, `im:read`, `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (si dependes de lecturas de búsqueda de Slack)

  </Accordion>
</AccordionGroup>

## Modelo de tokens

- `botToken` + `appToken` son obligatorios para Socket Mode.
- El modo HTTP requiere `botToken` + `signingSecret`.
- `botToken`, `appToken`, `signingSecret` y `userToken` aceptan cadenas
  en texto sin formato u objetos SecretRef.
- Los tokens de configuración reemplazan el respaldo por variables de entorno.
- El respaldo por variables de entorno `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` se aplica solo a la cuenta predeterminada.
- `userToken` (`xoxp-...`) es solo de configuración (sin respaldo por variables de entorno) y usa de forma predeterminada comportamiento de solo lectura (`userTokenReadOnly: true`).
- Opcional: agrega `chat:write.customize` si quieres que los mensajes salientes usen la identidad del agente activo (`username` e icono personalizados). `icon_emoji` usa la sintaxis `:emoji_name:`.

Comportamiento de instantánea de estado:

- La inspección de cuentas de Slack rastrea campos `*Source` y `*Status`
  por credencial (`botToken`, `appToken`, `signingSecret`, `userToken`).
- El estado es `available`, `configured_unavailable` o `missing`.
- `configured_unavailable` significa que la cuenta está configurada mediante SecretRef
  u otra fuente secreta no inline, pero la ruta actual del comando/tiempo de ejecución
  no pudo resolver el valor real.
- En modo HTTP, se incluye `signingSecretStatus`; en Socket Mode, el
  par requerido es `botTokenStatus` + `appTokenStatus`.

<Tip>
Para acciones/lecturas de directorio, el user token puede tener prioridad cuando está configurado. Para escrituras, el bot token sigue siendo el preferido; las escrituras con user token solo se permiten cuando `userTokenReadOnly: false` y el bot token no está disponible.
</Tip>

## Acciones y controles

Las acciones de Slack se controlan con `channels.slack.actions.*`.

Grupos de acciones disponibles en las herramientas actuales de Slack:

| Grupo      | Predeterminado |
| ---------- | -------------- |
| messages   | enabled        |
| reactions  | enabled        |
| pins       | enabled        |
| memberInfo | enabled        |
| emojiList  | enabled        |

Las acciones actuales de mensajes de Slack incluyen `send`, `upload-file`, `download-file`, `read`, `edit`, `delete`, `pin`, `unpin`, `list-pins`, `member-info` y `emoji-list`.

## Control de acceso y enrutamiento

<Tabs>
  <Tab title="Política de mensajes directos">
    `channels.slack.dmPolicy` controla el acceso a mensajes directos (heredado: `channels.slack.dm.policy`):

    - `pairing` (predeterminado)
    - `allowlist`
    - `open` (requiere que `channels.slack.allowFrom` incluya `"*"`; heredado: `channels.slack.dm.allowFrom`)
    - `disabled`

    Indicadores de mensajes directos:

    - `dm.enabled` (predeterminado true)
    - `channels.slack.allowFrom` (preferido)
    - `dm.allowFrom` (heredado)
    - `dm.groupEnabled` (los mensajes directos de grupo usan false de forma predeterminada)
    - `dm.groupChannels` (allowlist opcional de MPIM)

    Prioridad con varias cuentas:

    - `channels.slack.accounts.default.allowFrom` se aplica solo a la cuenta `default`.
    - Las cuentas con nombre heredan `channels.slack.allowFrom` cuando su propio `allowFrom` no está configurado.
    - Las cuentas con nombre no heredan `channels.slack.accounts.default.allowFrom`.

    El emparejamiento en mensajes directos usa `openclaw pairing approve slack <code>`.

  </Tab>

  <Tab title="Política de canales">
    `channels.slack.groupPolicy` controla el manejo de canales:

    - `open`
    - `allowlist`
    - `disabled`

    La allowlist de canales está en `channels.slack.channels` y debe usar IDs de canal estables.

    Nota de tiempo de ejecución: si `channels.slack` falta por completo (configuración solo con variables de entorno), el tiempo de ejecución recurre a `groupPolicy="allowlist"` y registra una advertencia (incluso si `channels.defaults.groupPolicy` está configurado).

    Resolución de nombre/ID:

    - las entradas de allowlist de canales y de mensajes directos se resuelven al iniciar cuando el acceso al token lo permite
    - las entradas no resueltas de nombre de canal se conservan tal como se configuraron, pero se ignoran para el enrutamiento de forma predeterminada
    - la autorización entrante y el enrutamiento de canales usan primero IDs de forma predeterminada; la coincidencia directa de nombre de usuario/slug requiere `channels.slack.dangerouslyAllowNameMatching: true`

  </Tab>

  <Tab title="Menciones y usuarios del canal">
    Los mensajes en canales usan filtrado por menciones de forma predeterminada.

    Fuentes de mención:

    - mención explícita de la app (`<@botId>`)
    - patrones regex de mención (`agents.list[].groupChat.mentionPatterns`, con respaldo en `messages.groupChat.mentionPatterns`)
    - comportamiento implícito de respuesta en hilo al bot

    Controles por canal (`channels.slack.channels.<id>`; nombres solo mediante resolución al inicio o `dangerouslyAllowNameMatching`):

    - `requireMention`
    - `users` (allowlist)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`, `toolsBySender`
    - formato de clave de `toolsBySender`: `id:`, `e164:`, `username:`, `name:` o comodín `"*"`
      (las claves heredadas sin prefijo siguen asignándose solo a `id:`)

  </Tab>
</Tabs>

## Hilos, sesiones y etiquetas de respuesta

- Los mensajes directos se enrutan como `direct`; los canales como `channel`; los MPIM como `group`.
- Con el valor predeterminado `session.dmScope=main`, los mensajes directos de Slack se agrupan en la sesión principal del agente.
- Sesiones de canal: `agent:<agentId>:slack:channel:<channelId>`.
- Las respuestas en hilo pueden crear sufijos de sesión de hilo (`:thread:<threadTs>`) cuando corresponde.
- `channels.slack.thread.historyScope` usa `thread` de forma predeterminada; `thread.inheritParent` usa `false` de forma predeterminada.
- `channels.slack.thread.initialHistoryLimit` controla cuántos mensajes existentes del hilo se recuperan cuando comienza una nueva sesión de hilo (predeterminado `20`; establece `0` para deshabilitarlo).

Controles de respuesta en hilo:

- `channels.slack.replyToMode`: `off|first|all` (predeterminado `off`)
- `channels.slack.replyToModeByChatType`: por `direct|group|channel`
- respaldo heredado para chats directos: `channels.slack.dm.replyToMode`

Se admiten etiquetas manuales de respuesta:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

Nota: `replyToMode="off"` deshabilita **todo** el encadenamiento de respuestas en Slack, incluidas las etiquetas explícitas `[[reply_to_*]]`. Esto difiere de Telegram, donde las etiquetas explícitas siguen respetándose en modo `"off"`. La diferencia refleja los modelos de hilo de cada plataforma: los hilos de Slack ocultan los mensajes del canal, mientras que las respuestas de Telegram siguen siendo visibles en el flujo principal del chat.

## Reacciones de acuse

`ackReaction` envía un emoji de confirmación mientras OpenClaw procesa un mensaje entrante.

Orden de resolución:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- respaldo con el emoji de identidad del agente (`agents.list[].identity.emoji`, si no `"👀"`)

Notas:

- Slack espera shortcodes (por ejemplo `"eyes"`).
- Usa `""` para deshabilitar la reacción para la cuenta de Slack o globalmente.

## Streaming de texto

`channels.slack.streaming` controla el comportamiento de vista previa en vivo:

- `off`: deshabilita el streaming de vista previa en vivo.
- `partial` (predeterminado): reemplaza el texto de vista previa con la salida parcial más reciente.
- `block`: agrega actualizaciones de vista previa fragmentadas.
- `progress`: muestra texto de estado de progreso mientras se genera y luego envía el texto final.

`channels.slack.nativeStreaming` controla el streaming nativo de texto de Slack cuando `streaming` es `partial` (predeterminado: `true`).

- Debe haber un hilo de respuesta disponible para que aparezca el streaming nativo de texto. La selección del hilo sigue usando `replyToMode`. Sin uno, se usa la vista previa normal del borrador.
- Los medios y las cargas no textuales recurren a la entrega normal.
- Si el streaming falla a mitad de la respuesta, OpenClaw recurre a la entrega normal para las cargas restantes.

Usa la vista previa del borrador en lugar del streaming nativo de texto de Slack:

```json5
{
  channels: {
    slack: {
      streaming: "partial",
      nativeStreaming: false,
    },
  },
}
```

Claves heredadas:

- `channels.slack.streamMode` (`replace | status_final | append`) se migra automáticamente a `channels.slack.streaming`.
- el booleano `channels.slack.streaming` se migra automáticamente a `channels.slack.nativeStreaming`.

## Respaldo de reacción de escritura

`typingReaction` agrega una reacción temporal al mensaje entrante de Slack mientras OpenClaw procesa una respuesta y luego la elimina cuando la ejecución termina. Esto es especialmente útil fuera de respuestas en hilo, que usan de forma predeterminada un indicador de estado "is typing...".

Orden de resolución:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

Notas:

- Slack espera shortcodes (por ejemplo `"hourglass_flowing_sand"`).
- La reacción es best-effort y se intenta limpiar automáticamente después de la respuesta o cuando se completa la ruta de error.

## Multimedia, fragmentación y entrega

<AccordionGroup>
  <Accordion title="Archivos adjuntos entrantes">
    Los archivos adjuntos de Slack se descargan desde URLs privadas alojadas por Slack (flujo de solicitud autenticada con token) y se escriben en el almacén de medios cuando la recuperación se realiza correctamente y los límites de tamaño lo permiten.

    El límite de tamaño entrante en tiempo de ejecución usa `20MB` de forma predeterminada, a menos que `channels.slack.mediaMaxMb` lo reemplace.

  </Accordion>

  <Accordion title="Texto y archivos salientes">
    - los fragmentos de texto usan `channels.slack.textChunkLimit` (predeterminado 4000)
    - `channels.slack.chunkMode="newline"` habilita la división por párrafos primero
    - los envíos de archivos usan las APIs de carga de Slack y pueden incluir respuestas en hilo (`thread_ts`)
    - el límite de contenido multimedia saliente sigue `channels.slack.mediaMaxMb` cuando está configurado; de lo contrario, los envíos por canal usan los valores predeterminados por tipo MIME del pipeline de medios
  </Accordion>

  <Accordion title="Destinos de entrega">
    Destinos explícitos preferidos:

    - `user:<id>` para mensajes directos
    - `channel:<id>` para canales

    Los mensajes directos de Slack se abren mediante las APIs de conversaciones de Slack al enviar a destinos de usuario.

  </Accordion>
</AccordionGroup>

## Comandos y comportamiento slash

- El modo automático de comandos nativos está **desactivado** para Slack (`commands.native: "auto"` no habilita los comandos nativos de Slack).
- Habilita los manejadores de comandos nativos de Slack con `channels.slack.commands.native: true` (o global `commands.native: true`).
- Cuando los comandos nativos están habilitados, registra comandos slash coincidentes en Slack (nombres `/<command>`), con una excepción:
  - registra `/agentstatus` para el comando de estado (Slack reserva `/status`)
- Si los comandos nativos no están habilitados, puedes ejecutar un único comando slash configurado mediante `channels.slack.slashCommand`.
- Los menús nativos de argumentos ahora adaptan su estrategia de renderizado:
  - hasta 5 opciones: bloques de botones
  - 6-100 opciones: menú de selección estática
  - más de 100 opciones: selección externa con filtrado asíncrono de opciones cuando están disponibles los manejadores de opciones de interactividad
  - si los valores de opción codificados superan los límites de Slack, el flujo recurre a botones
- Para cargas largas de opciones, los menús de argumentos de comandos slash usan un diálogo de confirmación antes de despachar un valor seleccionado.

Configuración predeterminada de comandos slash:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

Las sesiones slash usan claves aisladas:

- `agent:<agentId>:slack:slash:<userId>`

y aun así enrutan la ejecución del comando contra la sesión de conversación de destino (`CommandTargetSessionKey`).

## Respuestas interactivas

Slack puede renderizar controles de respuesta interactiva escritos por el agente, pero esta función está deshabilitada de forma predeterminada.

Habilítala globalmente:

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

O habilítala solo para una cuenta de Slack:

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

Cuando está habilitada, los agentes pueden emitir directivas de respuesta exclusivas de Slack:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

Estas directivas se compilan en Slack Block Kit y enrutan clics o selecciones a través de la ruta existente de eventos de interacción de Slack.

Notas:

- Esta es una UI específica de Slack. Los demás canales no traducen las directivas de Slack Block Kit a sus propios sistemas de botones.
- Los valores de callback interactivo son tokens opacos generados por OpenClaw, no valores sin procesar escritos por el agente.
- Si los bloques interactivos generados superan los límites de Slack Block Kit, OpenClaw recurre a la respuesta de texto original en lugar de enviar una carga de bloques no válida.

## Aprobaciones de exec en Slack

Slack puede actuar como cliente nativo de aprobación con botones interactivos e interacciones, en lugar de recurrir a la UI web o al terminal.

- Las aprobaciones de exec usan `channels.slack.execApprovals.*` para el enrutamiento nativo de mensajes directos/canales.
- Las aprobaciones de plugins aún pueden resolverse a través de la misma superficie de botones nativa de Slack cuando la solicitud ya llega a Slack y el tipo de ID de aprobación es `plugin:`.
- La autorización del aprobador sigue aplicándose: solo los usuarios identificados como aprobadores pueden aprobar o denegar solicitudes a través de Slack.

Esto usa la misma superficie compartida de botones de aprobación que otros canales. Cuando `interactivity` está habilitado en la configuración de tu app de Slack, las solicitudes de aprobación se renderizan como botones de Block Kit directamente en la conversación.
Cuando esos botones están presentes, son la UX principal de aprobación; OpenClaw
solo debe incluir un comando manual `/approve` cuando el resultado de la herramienta indique que las
aprobaciones por chat no están disponibles o que la aprobación manual es la única vía.

Ruta de configuración:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (opcional; usa como respaldo `commands.ownerAllowFrom` cuando es posible)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`, predeterminado: `dm`)
- `agentFilter`, `sessionFilter`

Slack habilita automáticamente las aprobaciones nativas de exec cuando `enabled` no está configurado o es `"auto"` y al menos un
aprobador se resuelve. Establece `enabled: false` para deshabilitar explícitamente Slack como cliente nativo de aprobación.
Establece `enabled: true` para forzar la activación de aprobaciones nativas cuando se resuelven aprobadores.

Comportamiento predeterminado sin configuración explícita de aprobación de exec para Slack:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

La configuración nativa explícita de Slack solo es necesaria cuando quieres reemplazar aprobadores, agregar filtros u
optar por entrega en el chat de origen:

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

El reenvío compartido de `approvals.exec` es independiente. Úsalo solo cuando las solicitudes de aprobación de exec también deban
enrutarse a otros chats o a destinos explícitos fuera de banda. El reenvío compartido de `approvals.plugin` también es
independiente; los botones nativos de Slack aún pueden resolver aprobaciones de plugins cuando esas solicitudes ya llegan
a Slack.

`/approve` en el mismo chat también funciona en canales y mensajes directos de Slack que ya admiten comandos. Consulta [Aprobaciones de exec](/tools/exec-approvals) para el modelo completo de reenvío de aprobaciones.

## Eventos y comportamiento operativo

- Las ediciones/eliminaciones de mensajes y las difusiones de hilos se asignan a eventos del sistema.
- Los eventos de agregar/eliminar reacción se asignan a eventos del sistema.
- Los eventos de unión/salida de miembros, creación/cambio de nombre de canales y agregar/eliminar fijados se asignan a eventos del sistema.
- `channel_id_changed` puede migrar claves de configuración de canales cuando `configWrites` está habilitado.
- Los metadatos de tema/propósito del canal se tratan como contexto no confiable y pueden inyectarse en el contexto de enrutamiento.
- El iniciador del hilo y la siembra inicial del contexto del historial del hilo se filtran mediante las allowlists configuradas de remitentes cuando corresponde.
- Las acciones de bloques e interacciones modales emiten eventos estructurados del sistema `Slack interaction: ...` con campos de carga enriquecidos:
  - acciones de bloques: valores seleccionados, etiquetas, valores de selector y metadatos `workflow_*`
  - eventos modales `view_submission` y `view_closed` con metadatos de canal enrutados e inputs de formularios

## Punteros a la referencia de configuración

Referencia principal:

- [Referencia de configuración - Slack](/gateway/configuration-reference#slack)

  Campos de Slack de alta relevancia:
  - modo/autenticación: `mode`, `botToken`, `appToken`, `signingSecret`, `webhookPath`, `accounts.*`
  - acceso a mensajes directos: `dm.enabled`, `dmPolicy`, `allowFrom` (heredado: `dm.policy`, `dm.allowFrom`), `dm.groupEnabled`, `dm.groupChannels`
  - interruptor de compatibilidad: `dangerouslyAllowNameMatching` (último recurso; mantenlo desactivado salvo que sea necesario)
  - acceso a canales: `groupPolicy`, `channels.*`, `channels.*.users`, `channels.*.requireMention`
  - hilos/historial: `replyToMode`, `replyToModeByChatType`, `thread.*`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
  - entrega: `textChunkLimit`, `chunkMode`, `mediaMaxMb`, `streaming`, `nativeStreaming`
  - operaciones/funciones: `configWrites`, `commands.native`, `slashCommand.*`, `actions.*`, `userToken`, `userTokenReadOnly`

## Solución de problemas

<AccordionGroup>
  <Accordion title="No hay respuestas en canales">
    Comprueba, en este orden:

    - `groupPolicy`
    - allowlist de canales (`channels.slack.channels`)
    - `requireMention`
    - allowlist `users` por canal

    Comandos útiles:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="Se ignoran los mensajes directos">
    Comprueba:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (o heredado `channels.slack.dm.policy`)
    - aprobaciones de emparejamiento / entradas de allowlist

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="Socket mode no conecta">
    Valida los tokens de bot y app, y que Socket Mode esté habilitado en la configuración de la app de Slack.

    Si `openclaw channels status --probe --json` muestra `botTokenStatus` o
    `appTokenStatus: "configured_unavailable"`, la cuenta de Slack está
    configurada, pero el tiempo de ejecución actual no pudo resolver el valor
    respaldado por SecretRef.

  </Accordion>

  <Accordion title="El modo HTTP no recibe eventos">
    Valida:

    - signing secret
    - ruta de webhook
    - URLs de solicitud de Slack (Events + Interactivity + Slash Commands)
    - `webhookPath` único por cuenta HTTP

    Si `signingSecretStatus: "configured_unavailable"` aparece en las
    instantáneas de cuenta, la cuenta HTTP está configurada, pero el tiempo de ejecución actual no pudo
    resolver el signing secret respaldado por SecretRef.

  </Accordion>

  <Accordion title="Los comandos nativos/slash no se activan">
    Verifica qué pretendías:

    - modo de comandos nativos (`channels.slack.commands.native: true`) con comandos slash coincidentes registrados en Slack
    - o modo de comando slash único (`channels.slack.slashCommand.enabled: true`)

    Comprueba también `commands.useAccessGroups` y las allowlists de canal/usuario.

  </Accordion>
</AccordionGroup>

## Relacionado

- [Emparejamiento](/channels/pairing)
- [Grupos](/channels/groups)
- [Seguridad](/gateway/security)
- [Enrutamiento de canales](/channels/channel-routing)
- [Solución de problemas](/channels/troubleshooting)
- [Configuración](/gateway/configuration)
- [Comandos slash](/tools/slash-commands)
