---
read_when:
    - Trabajar en el comportamiento del canal de WhatsApp/web o en el enrutamiento de bandeja de entrada
summary: Compatibilidad del canal de WhatsApp, controles de acceso, comportamiento de entrega y operaciones
title: WhatsApp
x-i18n:
    generated_at: "2026-04-05T12:37:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: c16a468b3f47fdf7e4fc3fd745b5c49c7ccebb7af0e8c87c632b78b04c583e49
    source_path: channels/whatsapp.md
    workflow: 15
---

# WhatsApp (canal web)

Estado: listo para producción mediante WhatsApp Web (Baileys). La gateway es propietaria de las sesiones vinculadas.

## Instalación (bajo demanda)

- Onboarding (`openclaw onboard`) y `openclaw channels add --channel whatsapp`
  solicitan instalar el plugin de WhatsApp la primera vez que lo seleccionas.
- `openclaw channels login --channel whatsapp` también ofrece el flujo de instalación cuando
  el plugin todavía no está presente.
- Canal de desarrollo + checkout de git: usa de forma predeterminada la ruta local del plugin.
- Stable/Beta: usa de forma predeterminada el paquete npm `@openclaw/whatsapp`.

La instalación manual sigue estando disponible:

```bash
openclaw plugins install @openclaw/whatsapp
```

<CardGroup cols={3}>
  <Card title="Emparejamiento" icon="link" href="/channels/pairing">
    La política predeterminada de mensajes directos es emparejamiento para remitentes desconocidos.
  </Card>
  <Card title="Solución de problemas del canal" icon="wrench" href="/channels/troubleshooting">
    Diagnósticos y guías de reparación entre canales.
  </Card>
  <Card title="Configuración de gateway" icon="settings" href="/gateway/configuration">
    Patrones y ejemplos completos de configuración del canal.
  </Card>
</CardGroup>

## Configuración rápida

<Steps>
  <Step title="Configurar la política de acceso de WhatsApp">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="Vincular WhatsApp (QR)">

```bash
openclaw channels login --channel whatsapp
```

    Para una cuenta específica:

```bash
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="Iniciar la gateway">

```bash
openclaw gateway
```

  </Step>

  <Step title="Aprobar la primera solicitud de emparejamiento (si usas el modo de emparejamiento)">

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    Las solicitudes de emparejamiento caducan después de 1 hora. Las solicitudes pendientes tienen un límite de 3 por canal.

  </Step>
</Steps>

<Note>
OpenClaw recomienda ejecutar WhatsApp en un número independiente cuando sea posible. (Los metadatos del canal y el flujo de configuración están optimizados para esa configuración, pero también se admiten configuraciones con número personal).
</Note>

## Patrones de despliegue

<AccordionGroup>
  <Accordion title="Número dedicado (recomendado)">
    Este es el modo operativo más limpio:

    - identidad de WhatsApp separada para OpenClaw
    - límites de listas de permitidos y enrutamiento de mensajes directos más claros
    - menor probabilidad de confusión por chatear contigo mismo

    Patrón de política mínima:

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Alternativa con número personal">
    Onboarding admite el modo de número personal y escribe una línea base compatible con chat contigo mismo:

    - `dmPolicy: "allowlist"`
    - `allowFrom` incluye tu número personal
    - `selfChatMode: true`

    En runtime, las protecciones de chat contigo mismo se basan en el número propio vinculado y `allowFrom`.

  </Accordion>

  <Accordion title="Ámbito del canal solo WhatsApp Web">
    El canal de la plataforma de mensajería está basado en WhatsApp Web (`Baileys`) en la arquitectura actual de canales de OpenClaw.

    No existe un canal de mensajería de WhatsApp de Twilio separado en el registro integrado de canales de chat.

  </Accordion>
</AccordionGroup>

## Modelo de runtime

- La gateway es propietaria del socket de WhatsApp y del bucle de reconexión.
- Los envíos salientes requieren un listener de WhatsApp activo para la cuenta de destino.
- Los chats de estado y difusión se ignoran (`@status`, `@broadcast`).
- Los chats directos usan reglas de sesión de mensajes directos (`session.dmScope`; `main` de forma predeterminada colapsa los mensajes directos en la sesión principal del agente).
- Las sesiones de grupo están aisladas (`agent:<agentId>:whatsapp:group:<jid>`).

## Control de acceso y activación

<Tabs>
  <Tab title="Política de mensajes directos">
    `channels.whatsapp.dmPolicy` controla el acceso al chat directo:

    - `pairing` (predeterminado)
    - `allowlist`
    - `open` (requiere que `allowFrom` incluya `"*"`)
    - `disabled`

    `allowFrom` acepta números con formato E.164 (normalizados internamente).

    Sobrescritura para varias cuentas: `channels.whatsapp.accounts.<id>.dmPolicy` (y `allowFrom`) tienen prioridad sobre los valores predeterminados a nivel de canal para esa cuenta.

    Detalles del comportamiento en runtime:

    - los emparejamientos se conservan en el almacén de permitidos del canal y se fusionan con `allowFrom` configurado
    - si no se configura ninguna lista de permitidos, el número propio vinculado está permitido de forma predeterminada
    - los mensajes directos salientes `fromMe` nunca se emparejan automáticamente

  </Tab>

  <Tab title="Política de grupo + listas de permitidos">
    El acceso a grupos tiene dos capas:

    1. **Lista de permitidos de pertenencia al grupo** (`channels.whatsapp.groups`)
       - si se omite `groups`, todos los grupos son elegibles
       - si `groups` está presente, actúa como lista de permitidos de grupos (se permite `"*"`)

    2. **Política de remitentes de grupo** (`channels.whatsapp.groupPolicy` + `groupAllowFrom`)
       - `open`: se omite la lista de remitentes permitidos
       - `allowlist`: el remitente debe coincidir con `groupAllowFrom` (o `*`)
       - `disabled`: bloquea toda entrada de grupos

    Fallback de lista de remitentes permitidos:

    - si `groupAllowFrom` no está establecido, el runtime usa `allowFrom` como fallback cuando está disponible
    - las listas de remitentes permitidos se evalúan antes de la activación por mención/respuesta

    Nota: si no existe ningún bloque `channels.whatsapp`, el fallback de política de grupo en runtime es `allowlist` (con una advertencia en el registro), incluso si `channels.defaults.groupPolicy` está establecido.

  </Tab>

  <Tab title="Menciones + /activation">
    Las respuestas en grupos requieren mención de forma predeterminada.

    La detección de menciones incluye:

    - menciones explícitas de WhatsApp a la identidad del bot
    - patrones regex de mención configurados (`agents.list[].groupChat.mentionPatterns`, fallback `messages.groupChat.mentionPatterns`)
    - detección implícita de respuesta al bot (el remitente de la respuesta coincide con la identidad del bot)

    Nota de seguridad:

    - citar/responder solo satisface el filtro de mención; **no** concede autorización al remitente
    - con `groupPolicy: "allowlist"`, los remitentes no incluidos en la lista de permitidos siguen bloqueados aunque respondan al mensaje de un usuario permitido

    Comando de activación a nivel de sesión:

    - `/activation mention`
    - `/activation always`

    `activation` actualiza el estado de la sesión (no la configuración global). Está protegido para el propietario.

  </Tab>
</Tabs>

## Comportamiento de número personal y chat contigo mismo

Cuando el número propio vinculado también está presente en `allowFrom`, se activan las protecciones de chat contigo mismo de WhatsApp:

- omitir confirmaciones de lectura en turnos de chat contigo mismo
- ignorar el comportamiento de activación automática por mention-JID que de otro modo te haría ping a ti mismo
- si `messages.responsePrefix` no está establecido, las respuestas en chat contigo mismo usan de forma predeterminada `[{identity.name}]` o `[openclaw]`

## Normalización de mensajes y contexto

<AccordionGroup>
  <Accordion title="Sobre envolvente de entrada + contexto de respuesta">
    Los mensajes entrantes de WhatsApp se encapsulan en el sobre compartido de entrada.

    Si existe una respuesta citada, el contexto se añade de esta forma:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    Los campos de metadatos de respuesta también se rellenan cuando están disponibles (`ReplyToId`, `ReplyToBody`, `ReplyToSender`, remitente JID/E.164).

  </Accordion>

  <Accordion title="Placeholders de medios y extracción de ubicación/contacto">
    Los mensajes entrantes solo de medios se normalizan con placeholders como:

    - `<media:image>`
    - `<media:video>`
    - `<media:audio>`
    - `<media:document>`
    - `<media:sticker>`

    Las cargas de ubicación y contacto se normalizan a contexto textual antes del enrutamiento.

  </Accordion>

  <Accordion title="Inyección pendiente del historial de grupos">
    Para grupos, los mensajes no procesados pueden almacenarse en búfer e inyectarse como contexto cuando finalmente se activa el bot.

    - límite predeterminado: `50`
    - configuración: `channels.whatsapp.historyLimit`
    - fallback: `messages.groupChat.historyLimit`
    - `0` desactiva

    Marcadores de inyección:

    - `[Chat messages since your last reply - for context]`
    - `[Current message - respond to this]`

  </Accordion>

  <Accordion title="Confirmaciones de lectura">
    Las confirmaciones de lectura están habilitadas de forma predeterminada para los mensajes entrantes de WhatsApp aceptados.

    Desactivar globalmente:

    ```json5
    {
      channels: {
        whatsapp: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    Sobrescritura por cuenta:

    ```json5
    {
      channels: {
        whatsapp: {
          accounts: {
            work: {
              sendReadReceipts: false,
            },
          },
        },
      },
    }
    ```

    Los turnos de chat contigo mismo omiten las confirmaciones de lectura incluso cuando están habilitadas globalmente.

  </Accordion>
</AccordionGroup>

## Entrega, fragmentación y medios

<AccordionGroup>
  <Accordion title="Fragmentación de texto">
    - límite de fragmentación predeterminado: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.chunkMode = "length" | "newline"`
    - el modo `newline` prefiere límites de párrafo (líneas en blanco) y luego usa como fallback una fragmentación segura por longitud
  </Accordion>

  <Accordion title="Comportamiento de medios salientes">
    - admite cargas de imagen, video, audio (nota de voz PTT) y documento
    - `audio/ogg` se reescribe a `audio/ogg; codecs=opus` para compatibilidad con notas de voz
    - la reproducción de GIF animados se admite mediante `gifPlayback: true` en envíos de video
    - los subtítulos se aplican al primer elemento multimedia al enviar cargas de respuesta multimeda
    - la fuente multimedia puede ser HTTP(S), `file://` o rutas locales
  </Accordion>

  <Accordion title="Límites de tamaño de medios y comportamiento de fallback">
    - límite de guardado de medios entrantes: `channels.whatsapp.mediaMaxMb` (predeterminado `50`)
    - límite de envío de medios salientes: `channels.whatsapp.mediaMaxMb` (predeterminado `50`)
    - las sobrescrituras por cuenta usan `channels.whatsapp.accounts.<accountId>.mediaMaxMb`
    - las imágenes se optimizan automáticamente (redimensionado/barrido de calidad) para ajustarse a los límites
    - ante fallo al enviar medios, el fallback del primer elemento envía una advertencia de texto en lugar de descartar la respuesta silenciosamente
  </Accordion>
</AccordionGroup>

## Nivel de reacciones

`channels.whatsapp.reactionLevel` controla qué tan ampliamente usa el agente reacciones con emoji en WhatsApp:

| Nivel         | Reacciones de confirmación | Reacciones iniciadas por el agente | Descripción                                          |
| ------------- | -------------------------- | ---------------------------------- | ---------------------------------------------------- |
| `"off"`       | No                         | No                                 | Sin reacciones en absoluto                           |
| `"ack"`       | Sí                         | No                                 | Solo reacciones de confirmación (acuse previo a respuesta) |
| `"minimal"`   | Sí                         | Sí (conservadoras)                 | Confirmación + reacciones del agente con guía conservadora |
| `"extensive"` | Sí                         | Sí (fomentadas)                    | Confirmación + reacciones del agente con guía fomentada    |

Predeterminado: `"minimal"`.

Las sobrescrituras por cuenta usan `channels.whatsapp.accounts.<id>.reactionLevel`.

```json5
{
  channels: {
    whatsapp: {
      reactionLevel: "ack",
    },
  },
}
```

## Reacciones de confirmación

WhatsApp admite reacciones de confirmación inmediatas al recibir entradas mediante `channels.whatsapp.ackReaction`.
Las reacciones de confirmación están controladas por `reactionLevel`: se suprimen cuando `reactionLevel` es `"off"`.

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // always | mentions | never
      },
    },
  },
}
```

Notas de comportamiento:

- se envían inmediatamente después de aceptar la entrada (antes de responder)
- los fallos se registran, pero no bloquean la entrega normal de la respuesta
- el modo de grupo `mentions` reacciona en turnos activados por mención; la activación de grupo `always` actúa como bypass para esta comprobación
- WhatsApp usa `channels.whatsapp.ackReaction` (el heredado `messages.ackReaction` no se usa aquí)

## Varias cuentas y credenciales

<AccordionGroup>
  <Accordion title="Selección de cuenta y valores predeterminados">
    - los IDs de cuenta provienen de `channels.whatsapp.accounts`
    - selección de cuenta predeterminada: `default` si está presente; en caso contrario, el primer ID de cuenta configurado (ordenado)
    - los IDs de cuenta se normalizan internamente para la búsqueda
  </Accordion>

  <Accordion title="Rutas de credenciales y compatibilidad heredada">
    - ruta actual de autenticación: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
    - archivo de respaldo: `creds.json.bak`
    - la autenticación heredada predeterminada en `~/.openclaw/credentials/` sigue reconociéndose/migrándose para flujos de cuenta predeterminada
  </Accordion>

  <Accordion title="Comportamiento de cierre de sesión">
    `openclaw channels logout --channel whatsapp [--account <id>]` borra el estado de autenticación de WhatsApp para esa cuenta.

    En directorios heredados de autenticación, `oauth.json` se conserva mientras se eliminan los archivos de autenticación de Baileys.

  </Accordion>
</AccordionGroup>

## Herramientas, acciones y escrituras de configuración

- La compatibilidad con herramientas del agente incluye la acción de reacción de WhatsApp (`react`).
- Puertas de acciones:
  - `channels.whatsapp.actions.reactions`
  - `channels.whatsapp.actions.polls`
- Las escrituras de configuración iniciadas por canal están habilitadas de forma predeterminada (desactívalas con `channels.whatsapp.configWrites=false`).

## Solución de problemas

<AccordionGroup>
  <Accordion title="No vinculado (se requiere QR)">
    Síntoma: el estado del canal informa que no está vinculado.

    Solución:

    ```bash
    openclaw channels login --channel whatsapp
    openclaw channels status
    ```

  </Accordion>

  <Accordion title="Vinculado pero desconectado / bucle de reconexión">
    Síntoma: cuenta vinculada con desconexiones repetidas o intentos de reconexión.

    Solución:

    ```bash
    openclaw doctor
    openclaw logs --follow
    ```

    Si es necesario, vuelve a vincular con `channels login`.

  </Accordion>

  <Accordion title="Sin listener activo al enviar">
    Los envíos salientes fallan rápidamente cuando no existe un listener activo de gateway para la cuenta de destino.

    Asegúrate de que la gateway esté en ejecución y de que la cuenta esté vinculada.

  </Accordion>

  <Accordion title="Los mensajes de grupo se ignoran inesperadamente">
    Comprueba en este orden:

    - `groupPolicy`
    - `groupAllowFrom` / `allowFrom`
    - entradas de la lista de permitidos `groups`
    - filtro de mención (`requireMention` + patrones de mención)
    - claves duplicadas en `openclaw.json` (JSON5): las entradas posteriores sobrescriben las anteriores, así que mantén un solo `groupPolicy` por ámbito

  </Accordion>

  <Accordion title="Advertencia de runtime de Bun">
    El runtime de gateway de WhatsApp debe usar Node. Bun está marcado como incompatible para el funcionamiento estable de la gateway de WhatsApp/Telegram.
  </Accordion>
</AccordionGroup>

## Indicadores de referencia de configuración

Referencia principal:

- [Referencia de configuración - WhatsApp](/gateway/configuration-reference#whatsapp)

Campos de WhatsApp de alta señal:

- acceso: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`
- entrega: `textChunkLimit`, `chunkMode`, `mediaMaxMb`, `sendReadReceipts`, `ackReaction`, `reactionLevel`
- varias cuentas: `accounts.<id>.enabled`, `accounts.<id>.authDir`, sobrescrituras a nivel de cuenta
- operaciones: `configWrites`, `debounceMs`, `web.enabled`, `web.heartbeatSeconds`, `web.reconnect.*`
- comportamiento de sesión: `session.dmScope`, `historyLimit`, `dmHistoryLimit`, `dms.<id>.historyLimit`

## Relacionado

- [Emparejamiento](/channels/pairing)
- [Grupos](/channels/groups)
- [Seguridad](/gateway/security)
- [Enrutamiento de canales](/channels/channel-routing)
- [Enrutamiento multiagente](/concepts/multi-agent)
- [Solución de problemas](/channels/troubleshooting)
