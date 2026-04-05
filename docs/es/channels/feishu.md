---
read_when:
    - Quieres conectar un bot de Feishu/Lark
    - Estás configurando el canal de Feishu
summary: Resumen, funciones y configuración del bot de Feishu
title: Feishu
x-i18n:
    generated_at: "2026-04-05T12:35:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e39b6dfe3a3aa4ebbdb992975e570e4f1b5e79f3b400a555fc373a0d1889952
    source_path: channels/feishu.md
    workflow: 15
---

# Bot de Feishu

Feishu (Lark) es una plataforma de chat para equipos que usan las empresas para mensajería y colaboración. Este plugin conecta OpenClaw con un bot de Feishu/Lark mediante la suscripción a eventos por WebSocket de la plataforma, para que los mensajes puedan recibirse sin exponer una URL pública de webhook.

---

## Plugin incluido

Feishu viene incluido con las versiones actuales de OpenClaw, por lo que no se requiere instalar un plugin por separado.

Si estás usando una compilación antigua o una instalación personalizada que no incluye Feishu integrado, instálalo manualmente:

```bash
openclaw plugins install @openclaw/feishu
```

---

## Inicio rápido

Hay dos formas de agregar el canal de Feishu:

### Método 1: onboarding (recomendado)

Si acabas de instalar OpenClaw, ejecuta el onboarding:

```bash
openclaw onboard
```

El asistente te guía en lo siguiente:

1. Crear una app de Feishu y recopilar las credenciales
2. Configurar las credenciales de la app en OpenClaw
3. Iniciar el gateway

✅ **Después de la configuración**, comprueba el estado del gateway:

- `openclaw gateway status`
- `openclaw logs --follow`

### Método 2: configuración con la CLI

Si ya completaste la instalación inicial, agrega el canal mediante la CLI:

```bash
openclaw channels add
```

Elige **Feishu** y luego introduce el App ID y el App Secret.

✅ **Después de la configuración**, administra el gateway:

- `openclaw gateway status`
- `openclaw gateway restart`
- `openclaw logs --follow`

---

## Paso 1: Crear una app de Feishu

### 1. Abre Feishu Open Platform

Visita [Feishu Open Platform](https://open.feishu.cn/app) e inicia sesión.

Los tenants de Lark (global) deben usar [https://open.larksuite.com/app](https://open.larksuite.com/app) y establecer `domain: "lark"` en la configuración de Feishu.

### 2. Crea una app

1. Haz clic en **Create enterprise app**
2. Completa el nombre y la descripción de la app
3. Elige un icono para la app

![Create enterprise app](/images/feishu-step2-create-app.png)

### 3. Copia las credenciales

En **Credentials & Basic Info**, copia:

- **App ID** (formato: `cli_xxx`)
- **App Secret**

❗ **Importante:** mantén el App Secret en privado.

![Get credentials](/images/feishu-step3-credentials.png)

### 4. Configura los permisos

En **Permissions**, haz clic en **Batch import** y pega:

```json
{
  "scopes": {
    "tenant": [
      "aily:file:read",
      "aily:file:write",
      "application:application.app_message_stats.overview:readonly",
      "application:application:self_manage",
      "application:bot.menu:write",
      "cardkit:card:read",
      "cardkit:card:write",
      "contact:user.employee_id:readonly",
      "corehr:file:download",
      "event:ip_list",
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.members:bot_access",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:resource"
    ],
    "user": ["aily:file:read", "aily:file:write", "im:chat.access_event.bot_p2p_chat:read"]
  }
}
```

![Configure permissions](/images/feishu-step4-permissions.png)

### 5. Habilita la capacidad de bot

En **App Capability** > **Bot**:

1. Habilita la capacidad de bot
2. Establece el nombre del bot

![Enable bot capability](/images/feishu-step5-bot-capability.png)

### 6. Configura la suscripción a eventos

⚠️ **Importante:** antes de configurar la suscripción a eventos, asegúrate de lo siguiente:

1. Ya ejecutaste `openclaw channels add` para Feishu
2. El gateway está en ejecución (`openclaw gateway status`)

En **Event Subscription**:

1. Elige **Use long connection to receive events** (WebSocket)
2. Agrega el evento: `im.message.receive_v1`
3. (Opcional) Para los flujos de trabajo de comentarios de Drive, agrega también: `drive.notice.comment_add_v1`

⚠️ Si el gateway no está en ejecución, es posible que la configuración de conexión larga no se guarde correctamente.

![Configure event subscription](/images/feishu-step6-event-subscription.png)

### 7. Publica la app

1. Crea una versión en **Version Management & Release**
2. Envíala para revisión y publícala
3. Espera la aprobación del administrador (las apps empresariales suelen aprobarse automáticamente)

---

## Paso 2: Configurar OpenClaw

### Configurar con el asistente (recomendado)

```bash
openclaw channels add
```

Elige **Feishu** y pega tu App ID y App Secret.

### Configurar mediante archivo de configuración

Edita `~/.openclaw/openclaw.json`:

```json5
{
  channels: {
    feishu: {
      enabled: true,
      dmPolicy: "pairing",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "My AI assistant",
        },
      },
    },
  },
}
```

Si usas `connectionMode: "webhook"`, configura tanto `verificationToken` como `encryptKey`. El servidor de webhooks de Feishu se enlaza a `127.0.0.1` de forma predeterminada; establece `webhookHost` solo si necesitas intencionalmente una dirección de enlace distinta.

#### Verification Token y Encrypt Key (modo webhook)

Al usar el modo webhook, configura tanto `channels.feishu.verificationToken` como `channels.feishu.encryptKey` en tu configuración. Para obtener esos valores:

1. En Feishu Open Platform, abre tu app
2. Ve a **Development** → **Events & Callbacks** (开发配置 → 事件与回调)
3. Abre la pestaña **Encryption** (加密策略)
4. Copia **Verification Token** y **Encrypt Key**

La captura de pantalla siguiente muestra dónde encontrar el **Verification Token**. El **Encrypt Key** aparece en la misma sección **Encryption**.

![Verification Token location](/images/feishu-verification-token.png)

### Configurar mediante variables de entorno

```bash
export FEISHU_APP_ID="cli_xxx"
export FEISHU_APP_SECRET="xxx"
```

### Dominio de Lark (global)

Si tu tenant está en Lark (internacional), establece el dominio en `lark` (o una cadena de dominio completa). Puedes configurarlo en `channels.feishu.domain` o por cuenta (`channels.feishu.accounts.<id>.domain`).

```json5
{
  channels: {
    feishu: {
      domain: "lark",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
        },
      },
    },
  },
}
```

### Indicadores de optimización de cuota

Puedes reducir el uso de la API de Feishu con dos indicadores opcionales:

- `typingIndicator` (predeterminado `true`): cuando es `false`, omite las llamadas de reacción de escritura.
- `resolveSenderNames` (predeterminado `true`): cuando es `false`, omite las llamadas de búsqueda del perfil del remitente.

Configúralos en el nivel superior o por cuenta:

```json5
{
  channels: {
    feishu: {
      typingIndicator: false,
      resolveSenderNames: false,
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          typingIndicator: true,
          resolveSenderNames: false,
        },
      },
    },
  },
}
```

---

## Paso 3: Iniciar + probar

### 1. Inicia el gateway

```bash
openclaw gateway
```

### 2. Envía un mensaje de prueba

En Feishu, encuentra tu bot y envíale un mensaje.

### 3. Aprueba el pairing

De forma predeterminada, el bot responde con un código de pairing. Apruébalo:

```bash
openclaw pairing approve feishu <CODE>
```

Después de la aprobación, ya puedes chatear normalmente.

---

## Resumen

- **Canal de bot de Feishu**: bot de Feishu gestionado por el gateway
- **Enrutamiento determinista**: las respuestas siempre regresan a Feishu
- **Aislamiento de sesiones**: los mensajes directos comparten una sesión principal; los grupos están aislados
- **Conexión WebSocket**: conexión larga mediante el SDK de Feishu, sin necesidad de una URL pública

---

## Control de acceso

### Mensajes directos

- **Predeterminado**: `dmPolicy: "pairing"` (los usuarios desconocidos reciben un código de pairing)
- **Aprobar pairing**:

  ```bash
  openclaw pairing list feishu
  openclaw pairing approve feishu <CODE>
  ```

- **Modo allowlist**: establece `channels.feishu.allowFrom` con los Open ID permitidos

### Chats grupales

**1. Política de grupo** (`channels.feishu.groupPolicy`):

- `"open"` = permitir a todos en grupos
- `"allowlist"` = permitir solo `groupAllowFrom`
- `"disabled"` = deshabilitar mensajes de grupo

Predeterminado: `allowlist`

**2. Requisito de mención** (`channels.feishu.requireMention`, sobrescribible mediante `channels.feishu.groups.<chat_id>.requireMention`):

- `true` explícito = requiere @mention
- `false` explícito = responde sin menciones
- cuando no está establecido y `groupPolicy: "open"` = predeterminado `false`
- cuando no está establecido y `groupPolicy` no es `"open"` = predeterminado `true`

---

## Ejemplos de configuración de grupos

### Permitir todos los grupos, sin requerir @mention (predeterminado para grupos abiertos)

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
    },
  },
}
```

### Permitir todos los grupos, pero seguir requiriendo @mention

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
      requireMention: true,
    },
  },
}
```

### Permitir solo grupos específicos

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      // Los ID de grupo de Feishu (chat_id) tienen un formato como: oc_xxx
      groupAllowFrom: ["oc_xxx", "oc_yyy"],
    },
  },
}
```

### Restringir qué remitentes pueden enviar mensajes en un grupo (allowlist de remitentes)

Además de permitir el grupo en sí, **todos los mensajes** de ese grupo se controlan mediante el open_id del remitente: solo los usuarios incluidos en `groups.<chat_id>.allowFrom` tienen sus mensajes procesados; los mensajes de otros miembros se ignoran (esto es un control completo a nivel de remitente, no solo para comandos de control como /reset o /new).

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          // Los ID de usuario de Feishu (open_id) tienen un formato como: ou_xxx
          allowFrom: ["ou_user1", "ou_user2"],
        },
      },
    },
  },
}
```

---

<a id="get-groupuser-ids"></a>

## Obtener ID de grupo/usuario

### ID de grupo (chat_id)

Los ID de grupo tienen un formato como `oc_xxx`.

**Método 1 (recomendado)**

1. Inicia el gateway y haz @mention al bot en el grupo
2. Ejecuta `openclaw logs --follow` y busca `chat_id`

**Método 2**

Usa el depurador de la API de Feishu para listar chats de grupo.

### ID de usuario (open_id)

Los ID de usuario tienen un formato como `ou_xxx`.

**Método 1 (recomendado)**

1. Inicia el gateway y envía un mensaje directo al bot
2. Ejecuta `openclaw logs --follow` y busca `open_id`

**Método 2**

Consulta las solicitudes de pairing para ver los Open ID de usuario:

```bash
openclaw pairing list feishu
```

---

## Comandos comunes

| Comando   | Descripción                 |
| --------- | --------------------------- |
| `/status` | Mostrar el estado del bot   |
| `/reset`  | Restablecer la sesión       |
| `/model`  | Mostrar/cambiar el modelo   |

> Nota: Feishu todavía no admite menús de comandos nativos, por lo que los comandos deben enviarse como texto.

## Comandos de administración del gateway

| Comando                    | Descripción                    |
| -------------------------- | ------------------------------ |
| `openclaw gateway status`  | Mostrar el estado del gateway  |
| `openclaw gateway install` | Instalar/iniciar el servicio del gateway |
| `openclaw gateway stop`    | Detener el servicio del gateway |
| `openclaw gateway restart` | Reiniciar el servicio del gateway |
| `openclaw logs --follow`   | Seguir los logs del gateway    |

---

## Solución de problemas

### El bot no responde en chats grupales

1. Asegúrate de que el bot esté agregado al grupo
2. Asegúrate de hacer @mention al bot (comportamiento predeterminado)
3. Comprueba que `groupPolicy` no esté configurado como `"disabled"`
4. Revisa los logs: `openclaw logs --follow`

### El bot no recibe mensajes

1. Asegúrate de que la app esté publicada y aprobada
2. Asegúrate de que la suscripción a eventos incluya `im.message.receive_v1`
3. Asegúrate de que **long connection** esté habilitada
4. Asegúrate de que los permisos de la app estén completos
5. Asegúrate de que el gateway esté en ejecución: `openclaw gateway status`
6. Revisa los logs: `openclaw logs --follow`

### Fuga de App Secret

1. Restablece el App Secret en Feishu Open Platform
2. Actualiza el App Secret en tu configuración
3. Reinicia el gateway

### Fallos al enviar mensajes

1. Asegúrate de que la app tenga el permiso `im:message:send_as_bot`
2. Asegúrate de que la app esté publicada
3. Revisa los logs para ver los errores detallados

---

## Configuración avanzada

### Varias cuentas

```json5
{
  channels: {
    feishu: {
      defaultAccount: "main",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "Primary bot",
        },
        backup: {
          appId: "cli_yyy",
          appSecret: "yyy",
          name: "Backup bot",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` controla qué cuenta de Feishu se usa cuando las API salientes no especifican un `accountId` explícitamente.

### Límites de mensajes

- `textChunkLimit`: tamaño de fragmento del texto saliente (predeterminado: 2000 caracteres)
- `mediaMaxMb`: límite de subida/descarga de medios (predeterminado: 30MB)

### Streaming

Feishu admite respuestas en streaming mediante tarjetas interactivas. Cuando está habilitado, el bot actualiza una tarjeta mientras genera texto.

```json5
{
  channels: {
    feishu: {
      streaming: true, // habilitar salida de tarjetas en streaming (predeterminado true)
      blockStreaming: true, // habilitar streaming a nivel de bloque (predeterminado true)
    },
  },
}
```

Establece `streaming: false` para esperar a la respuesta completa antes de enviarla.

### Sesiones ACP

Feishu admite ACP para:

- mensajes directos
- conversaciones de tema en grupo

El ACP de Feishu se basa en comandos de texto. No hay menús nativos de comandos slash, así que usa mensajes `/acp ...` directamente en la conversación.

#### Enlaces ACP persistentes

Usa enlaces ACP tipados de nivel superior para fijar un mensaje directo o una conversación de tema de Feishu a una sesión ACP persistente.

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
        channel: "feishu",
        accountId: "default",
        peer: { kind: "direct", id: "ou_1234567890" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "group", id: "oc_group_chat:topic:om_topic_root" },
      },
      acp: { label: "codex-feishu-topic" },
    },
  ],
}
```

#### Creación de ACP vinculada al hilo desde el chat

En un mensaje directo o una conversación de tema de Feishu, puedes crear y vincular una sesión ACP en el mismo lugar:

```text
/acp spawn codex --thread here
```

Notas:

- `--thread here` funciona para mensajes directos y temas de Feishu.
- Los mensajes posteriores en el mensaje directo/tema vinculado se enrutan directamente a esa sesión ACP.
- La v1 no se dirige a chats grupales genéricos sin tema.

### Enrutamiento multiagente

Usa `bindings` para enrutar mensajes directos o grupos de Feishu a distintos agentes.

```json5
{
  agents: {
    list: [
      { id: "main" },
      {
        id: "clawd-fan",
        workspace: "/home/user/clawd-fan",
        agentDir: "/home/user/.openclaw/agents/clawd-fan/agent",
      },
      {
        id: "clawd-xi",
        workspace: "/home/user/clawd-xi",
        agentDir: "/home/user/.openclaw/agents/clawd-xi/agent",
      },
    ],
  },
  bindings: [
    {
      agentId: "main",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_xxx" },
      },
    },
    {
      agentId: "clawd-fan",
      match: {
        channel: "feishu",
        peer: { kind: "direct", id: "ou_yyy" },
      },
    },
    {
      agentId: "clawd-xi",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzz" },
      },
    },
  ],
}
```

Campos de enrutamiento:

- `match.channel`: `"feishu"`
- `match.peer.kind`: `"direct"` o `"group"`
- `match.peer.id`: Open ID de usuario (`ou_xxx`) o ID de grupo (`oc_xxx`)

Consulta [Obtener ID de grupo/usuario](#get-groupuser-ids) para ver consejos de búsqueda.

---

## Referencia de configuración

Configuración completa: [Gateway configuration](/gateway/configuration)

Opciones clave:

| Setting                                           | Descripción                              | Predeterminado   |
| ------------------------------------------------- | ---------------------------------------- | ---------------- |
| `channels.feishu.enabled`                         | Habilitar/deshabilitar el canal          | `true`           |
| `channels.feishu.domain`                          | Dominio de la API (`feishu` o `lark`)    | `feishu`         |
| `channels.feishu.connectionMode`                  | Modo de transporte de eventos            | `websocket`      |
| `channels.feishu.defaultAccount`                  | ID de cuenta predeterminada para el enrutamiento saliente | `default`        |
| `channels.feishu.verificationToken`               | Obligatorio para el modo webhook         | -                |
| `channels.feishu.encryptKey`                      | Obligatorio para el modo webhook         | -                |
| `channels.feishu.webhookPath`                     | Ruta del webhook                         | `/feishu/events` |
| `channels.feishu.webhookHost`                     | Host de enlace del webhook               | `127.0.0.1`      |
| `channels.feishu.webhookPort`                     | Puerto de enlace del webhook             | `3000`           |
| `channels.feishu.accounts.<id>.appId`             | App ID                                   | -                |
| `channels.feishu.accounts.<id>.appSecret`         | App Secret                               | -                |
| `channels.feishu.accounts.<id>.domain`            | Sobrescritura del dominio de la API por cuenta | `feishu`         |
| `channels.feishu.dmPolicy`                        | Política de mensajes directos            | `pairing`        |
| `channels.feishu.allowFrom`                       | Allowlist de mensajes directos (lista de `open_id`) | -                |
| `channels.feishu.groupPolicy`                     | Política de grupo                        | `allowlist`      |
| `channels.feishu.groupAllowFrom`                  | Allowlist de grupos                      | -                |
| `channels.feishu.requireMention`                  | Requerir @mention de forma predeterminada | condicional      |
| `channels.feishu.groups.<chat_id>.requireMention` | Sobrescritura por grupo para requerir @mention | heredado         |
| `channels.feishu.groups.<chat_id>.enabled`        | Habilitar grupo                          | `true`           |
| `channels.feishu.textChunkLimit`                  | Tamaño del fragmento del mensaje         | `2000`           |
| `channels.feishu.mediaMaxMb`                      | Límite de tamaño de medios               | `30`             |
| `channels.feishu.streaming`                       | Habilitar salida de tarjetas en streaming | `true`           |
| `channels.feishu.blockStreaming`                  | Habilitar streaming por bloques          | `true`           |

---

## Referencia de dmPolicy

| Value         | Comportamiento                                                 |
| ------------- | -------------------------------------------------------------- |
| `"pairing"`   | **Predeterminado.** Los usuarios desconocidos reciben un código de pairing; deben ser aprobados |
| `"allowlist"` | Solo los usuarios en `allowFrom` pueden chatear                |
| `"open"`      | Permitir a todos los usuarios (requiere `"*"` en `allowFrom`)  |
| `"disabled"`  | Deshabilitar mensajes directos                                 |

---

## Tipos de mensajes compatibles

### Recepción

- ✅ Texto
- ✅ Texto enriquecido (post)
- ✅ Imágenes
- ✅ Archivos
- ✅ Audio
- ✅ Video/medios
- ✅ Stickers

### Envío

- ✅ Texto
- ✅ Imágenes
- ✅ Archivos
- ✅ Audio
- ✅ Video/medios
- ✅ Tarjetas interactivas
- ⚠️ Texto enriquecido (formato estilo post y tarjetas, no funciones arbitrarias de autoría de Feishu)

### Hilos y respuestas

- ✅ Respuestas en línea
- ✅ Respuestas en hilos de tema cuando Feishu expone `reply_in_thread`
- ✅ Las respuestas con medios mantienen compatibilidad con hilos al responder a un mensaje de hilo/tema

## Comentarios de Drive

Feishu puede activar el agente cuando alguien agrega un comentario en un documento de Feishu Drive (Docs, Sheets, etc.). El agente recibe el texto del comentario, el contexto del documento y el hilo del comentario para poder responder en el hilo o realizar ediciones en el documento.

Requisitos:

- Suscribirse a `drive.notice.comment_add_v1` en la configuración de suscripción a eventos de tu app de Feishu
  (junto con `im.message.receive_v1`, que ya existe)
- La herramienta Drive está habilitada de forma predeterminada; desactívala con `channels.feishu.tools.drive: false`

La herramienta `feishu_drive` expone estas acciones de comentarios:

| Acción                 | Descripción                            |
| ---------------------- | -------------------------------------- |
| `list_comments`        | Listar comentarios en un documento     |
| `list_comment_replies` | Listar respuestas en un hilo de comentarios |
| `add_comment`          | Agregar un nuevo comentario de nivel superior |
| `reply_comment`        | Responder a un hilo de comentarios existente |

Cuando el agente gestiona un evento de comentario de Drive, recibe:

- el texto del comentario y el remitente
- metadatos del documento (título, tipo, URL)
- el contexto del hilo del comentario para responder en el hilo

Después de realizar ediciones en el documento, se guía al agente para usar `feishu_drive.reply_comment` para notificar al comentarista y luego generar el token silencioso exacto `NO_REPLY` / `no_reply` para evitar envíos duplicados.

## Superficie de acciones en tiempo de ejecución

Feishu expone actualmente estas acciones en tiempo de ejecución:

- `send`
- `read`
- `edit`
- `thread-reply`
- `pin`
- `list-pins`
- `unpin`
- `member-info`
- `channel-info`
- `channel-list`
- `react` y `reactions` cuando las reacciones están habilitadas en la configuración
- acciones de comentarios `feishu_drive`: `list_comments`, `list_comment_replies`, `add_comment`, `reply_comment`

## Relacionado

- [Channels Overview](/channels) — todos los canales compatibles
- [Pairing](/channels/pairing) — autenticación de mensajes directos y flujo de pairing
- [Groups](/channels/groups) — comportamiento de chats grupales y control por menciones
- [Channel Routing](/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Security](/gateway/security) — modelo de acceso y endurecimiento
