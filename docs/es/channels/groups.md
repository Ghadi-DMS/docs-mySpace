---
read_when:
    - Cambio del comportamiento de chats grupales o del control por menciones
summary: Comportamiento de chats grupales en todas las superficies (Discord/iMessage/Matrix/Microsoft Teams/Signal/Slack/Telegram/WhatsApp/Zalo)
title: Grupos
x-i18n:
    generated_at: "2026-04-05T12:35:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 39d066e0542b468c6f8b384b463e2316590ea09a00ecb2065053e1e2ce55bd5f
    source_path: channels/groups.md
    workflow: 15
---

# Grupos

OpenClaw trata los chats grupales de forma coherente en todas las superficies: Discord, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo.

## Introducción para principiantes (2 minutos)

OpenClaw “vive” en tus propias cuentas de mensajería. No hay un usuario de bot de WhatsApp independiente.
Si **tú** estás en un grupo, OpenClaw puede ver ese grupo y responder allí.

Comportamiento predeterminado:

- Los grupos están restringidos (`groupPolicy: "allowlist"`).
- Las respuestas requieren una mención, a menos que desactives explícitamente el control por menciones.

Traducción: los remitentes incluidos en la lista de permitidos pueden activar OpenClaw mencionándolo.

> Resumen rápido
>
> - El **acceso por mensaje directo** se controla con `*.allowFrom`.
> - El **acceso a grupos** se controla con `*.groupPolicy` + listas de permitidos (`*.groups`, `*.groupAllowFrom`).
> - La **activación de respuestas** se controla con el control por menciones (`requireMention`, `/activation`).

Flujo rápido (qué ocurre con un mensaje de grupo):

```
groupPolicy? disabled -> drop
groupPolicy? allowlist -> group allowed? no -> drop
requireMention? yes -> mentioned? no -> store for context only
otherwise -> reply
```

## Visibilidad del contexto y listas de permitidos

En la seguridad de grupos intervienen dos controles diferentes:

- **Autorización de activación**: quién puede activar el agente (`groupPolicy`, `groups`, `groupAllowFrom`, listas de permitidos específicas del canal).
- **Visibilidad del contexto**: qué contexto complementario se inyecta en el modelo (texto de respuesta, citas, historial del hilo, metadatos reenviados).

De forma predeterminada, OpenClaw prioriza el comportamiento normal del chat y mantiene el contexto principalmente tal como se recibió. Esto significa que las listas de permitidos deciden principalmente quién puede activar acciones, no un límite universal de ocultación para cada fragmento citado o histórico.

El comportamiento actual es específico de cada canal:

- Algunos canales ya aplican filtrado basado en remitente para contexto complementario en rutas específicas (por ejemplo, inicialización de hilos en Slack, búsquedas de respuestas/hilos en Matrix).
- Otros canales todavía pasan el contexto de cita/respuesta/reenvío tal como se recibe.

Dirección de endurecimiento (planificada):

- `contextVisibility: "all"` (predeterminado) mantiene el comportamiento actual tal como se recibe.
- `contextVisibility: "allowlist"` filtra el contexto complementario a remitentes en la lista de permitidos.
- `contextVisibility: "allowlist_quote"` es `allowlist` más una excepción explícita de cita/respuesta.

Hasta que este modelo de endurecimiento se implemente de forma coherente en todos los canales, espera diferencias según la superficie.

![Flujo de mensajes de grupo](/images/groups-flow.svg)

Si quieres...

| Objetivo                                     | Qué configurar                                           |
| -------------------------------------------- | -------------------------------------------------------- |
| Permitir todos los grupos, pero responder solo a @mentions | `groups: { "*": { requireMention: true } }`              |
| Desactivar todas las respuestas en grupos    | `groupPolicy: "disabled"`                                |
| Solo grupos específicos                      | `groups: { "<group-id>": { ... } }` (sin clave `"*"`)    |
| Solo tú puedes activar en grupos             | `groupPolicy: "allowlist"`, `groupAllowFrom: ["+1555..."]` |

## Claves de sesión

- Las sesiones de grupo usan claves de sesión `agent:<agentId>:<channel>:group:<id>` (las salas/canales usan `agent:<agentId>:<channel>:channel:<id>`).
- Los temas de foros de Telegram añaden `:topic:<threadId>` al id del grupo para que cada tema tenga su propia sesión.
- Los chats directos usan la sesión principal (o una por remitente si está configurado).
- Los heartbeat se omiten para las sesiones de grupo.

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## Patrón: mensajes directos personales + grupos públicos (agente único)

Sí, esto funciona bien si tu tráfico “personal” son **mensajes directos** y tu tráfico “público” son **grupos**.

Por qué: en modo de agente único, los mensajes directos normalmente llegan a la clave de sesión **principal** (`agent:main:main`), mientras que los grupos siempre usan claves de sesión **no principales** (`agent:main:<channel>:group:<id>`). Si habilitas el aislamiento con `mode: "non-main"`, esas sesiones de grupo se ejecutan en Docker mientras tu sesión principal de mensajes directos permanece en el host.

Esto te da un único “cerebro” de agente (espacio de trabajo + memoria compartidos), pero dos posturas de ejecución:

- **Mensajes directos**: herramientas completas (host)
- **Grupos**: sandbox + herramientas restringidas de mensajería (Docker)

> Si necesitas espacios de trabajo/personas realmente separados (“personal” y “público” nunca deben mezclarse), usa un segundo agente + enlaces. Consulta [Enrutamiento multiagente](/concepts/multi-agent).

Ejemplo (mensajes directos en el host, grupos aislados + solo herramientas de mensajería):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // groups/channels are non-main -> sandboxed
        scope: "session", // strongest isolation (one container per group/channel)
        workspaceAccess: "none",
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        // If allow is non-empty, everything else is blocked (deny still wins).
        allow: ["group:messaging", "group:sessions"],
        deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
      },
    },
  },
}
```

¿Quieres “los grupos solo pueden ver la carpeta X” en lugar de “sin acceso al host”? Mantén `workspaceAccess: "none"` y monta solo rutas incluidas en la lista de permitidos dentro del sandbox:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
        docker: {
          binds: [
            // hostPath:containerPath:mode
            "/home/user/FriendsShared:/data:ro",
          ],
        },
      },
    },
  },
}
```

Relacionado:

- Claves de configuración y valores predeterminados: [Configuración del gateway](/gateway/configuration-reference#agentsdefaultssandbox)
- Depurar por qué una herramienta está bloqueada: [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated)
- Detalles de bind mounts: [Aislamiento](/gateway/sandboxing#custom-bind-mounts)

## Etiquetas para mostrar

- Las etiquetas de la interfaz usan `displayName` cuando está disponible, con el formato `<channel>:<token>`.
- `#room` está reservado para salas/canales; los chats grupales usan `g-<slug>` (minúsculas, espacios -> `-`, conserva `#@+._-`).

## Política de grupos

Controla cómo se manejan los mensajes de grupos/salas por canal:

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // numeric Telegram user id (wizard can resolve @username)
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { allow: true } } },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { allow: true } },
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { allow: true },
        "#alias:example.org": { allow: true },
      },
    },
  },
}
```

| Política      | Comportamiento                                                   |
| ------------- | ---------------------------------------------------------------- |
| `"open"`      | Los grupos omiten las listas de permitidos; el control por menciones sigue aplicándose. |
| `"disabled"`  | Bloquea por completo todos los mensajes de grupo.                |
| `"allowlist"` | Solo permite grupos/salas que coincidan con la lista de permitidos configurada. |

Notas:

- `groupPolicy` es independiente del control por menciones (que requiere @mentions).
- WhatsApp/Telegram/Signal/iMessage/Microsoft Teams/Zalo: usa `groupAllowFrom` (respaldo: `allowFrom` explícito).
- Las aprobaciones de emparejamiento de mensajes directos (entradas almacenadas en `*-allowFrom`) se aplican solo al acceso por mensaje directo; la autorización de remitentes en grupos sigue siendo explícita mediante listas de permitidos de grupo.
- Discord: la lista de permitidos usa `channels.discord.guilds.<id>.channels`.
- Slack: la lista de permitidos usa `channels.slack.channels`.
- Matrix: la lista de permitidos usa `channels.matrix.groups`. Prefiere ids de sala o alias; la búsqueda de nombres de salas unidas es de mejor esfuerzo, y los nombres no resueltos se ignoran en tiempo de ejecución. Usa `channels.matrix.groupAllowFrom` para restringir remitentes; también se admiten listas de permitidos `users` por sala.
- Los mensajes directos de grupo se controlan por separado (`channels.discord.dm.*`, `channels.slack.dm.*`).
- La lista de permitidos de Telegram puede coincidir con ids de usuario (`"123456789"`, `"telegram:123456789"`, `"tg:123456789"`) o nombres de usuario (`"@alice"` o `"alice"`); los prefijos no distinguen mayúsculas de minúsculas.
- El valor predeterminado es `groupPolicy: "allowlist"`; si tu lista de permitidos de grupos está vacía, los mensajes de grupo se bloquean.
- Seguridad en tiempo de ejecución: cuando falta por completo un bloque de proveedor (`channels.<provider>` ausente), la política de grupos recurre a un modo de fallo cerrado (normalmente `allowlist`) en lugar de heredar `channels.defaults.groupPolicy`.

Modelo mental rápido (orden de evaluación para mensajes de grupo):

1. `groupPolicy` (`open`/`disabled`/`allowlist`)
2. listas de permitidos de grupo (`*.groups`, `*.groupAllowFrom`, lista de permitidos específica del canal)
3. control por menciones (`requireMention`, `/activation`)

## Control por menciones (predeterminado)

Los mensajes de grupo requieren una mención a menos que se anule por grupo. Los valores predeterminados viven por subsistema en `*.groups."*"`.

Responder a un mensaje del bot cuenta como una mención implícita (cuando el canal admite metadatos de respuesta). Esto se aplica a Telegram, WhatsApp, Slack, Discord y Microsoft Teams.

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false },
      },
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false },
      },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"],
          historyLimit: 50,
        },
      },
    ],
  },
}
```

Notas:

- `mentionPatterns` son patrones regex seguros que no distinguen mayúsculas de minúsculas; los patrones no válidos y las formas inseguras de repetición anidada se ignoran.
- Las superficies que proporcionan menciones explícitas siguen funcionando; los patrones son un respaldo.
- Anulación por agente: `agents.list[].groupChat.mentionPatterns` (útil cuando varios agentes comparten un grupo).
- El control por menciones solo se aplica cuando la detección de menciones es posible (menciones nativas o `mentionPatterns` configurados).
- Los valores predeterminados de Discord viven en `channels.discord.guilds."*"` (se pueden anular por servidor/canal).
- El contexto del historial de grupo se encapsula de manera uniforme entre canales y es **solo pendiente** (mensajes omitidos debido al control por menciones); usa `messages.groupChat.historyLimit` para el valor predeterminado global y `channels.<channel>.historyLimit` (o `channels.<channel>.accounts.*.historyLimit`) para las anulaciones. Establece `0` para desactivarlo.

## Restricciones de herramientas para grupos/canales (opcional)

Algunas configuraciones de canal permiten restringir qué herramientas están disponibles **dentro de un grupo/sala/canal específico**.

- `tools`: permite o deniega herramientas para todo el grupo.
- `toolsBySender`: anulaciones por remitente dentro del grupo.
  Usa prefijos explícitos de clave:
  `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` y comodín `"*"`.
  Las claves heredadas sin prefijo siguen aceptándose y coinciden solo como `id:`.

Orden de resolución (gana el más específico):

1. coincidencia de `toolsBySender` del grupo/canal
2. `tools` del grupo/canal
3. coincidencia de `toolsBySender` predeterminada (`"*"`)
4. `tools` predeterminada (`"*"`)

Ejemplo (Telegram):

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { tools: { deny: ["exec"] } },
        "-1001234567890": {
          tools: { deny: ["exec", "read", "write"] },
          toolsBySender: {
            "id:123456789": { alsoAllow: ["exec"] },
          },
        },
      },
    },
  },
}
```

Notas:

- Las restricciones de herramientas para grupos/canales se aplican además de la política global/de agente de herramientas (la denegación sigue prevaleciendo).
- Algunos canales usan una anidación diferente para salas/canales (por ejemplo, Discord `guilds.*.channels.*`, Slack `channels.*`, Microsoft Teams `teams.*.channels.*`).

## Listas de permitidos de grupos

Cuando se configura `channels.whatsapp.groups`, `channels.telegram.groups` o `channels.imessage.groups`, las claves actúan como una lista de permitidos de grupos. Usa `"*"` para permitir todos los grupos mientras sigues estableciendo el comportamiento predeterminado de mención.

Confusión habitual: la aprobación de emparejamiento de mensajes directos no es lo mismo que la autorización de grupos.
En los canales que admiten emparejamiento de mensajes directos, el almacén de emparejamiento desbloquea solo los mensajes directos. Los comandos de grupo siguen requiriendo autorización explícita del remitente del grupo mediante listas de permitidos de configuración como `groupAllowFrom` o el respaldo de configuración documentado para ese canal.

Intenciones habituales (copiar/pegar):

1. Desactivar todas las respuestas en grupos

```json5
{
  channels: { whatsapp: { groupPolicy: "disabled" } },
}
```

2. Permitir solo grupos específicos (WhatsApp)

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "123@g.us": { requireMention: true },
        "456@g.us": { requireMention: false },
      },
    },
  },
}
```

3. Permitir todos los grupos pero requerir mención (explícito)

```json5
{
  channels: {
    whatsapp: {
      groups: { "*": { requireMention: true } },
    },
  },
}
```

4. Solo el propietario puede activar en grupos (WhatsApp)

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
      groups: { "*": { requireMention: true } },
    },
  },
}
```

## Activación (solo propietario)

Los propietarios de grupos pueden alternar la activación por grupo:

- `/activation mention`
- `/activation always`

El propietario se determina mediante `channels.whatsapp.allowFrom` (o la propia E.164 del bot cuando no está definida). Envía el comando como un mensaje independiente. Otras superficies actualmente ignoran `/activation`.

## Campos de contexto

Las cargas útiles entrantes de grupos establecen:

- `ChatType=group`
- `GroupSubject` (si se conoce)
- `GroupMembers` (si se conoce)
- `WasMentioned` (resultado del control por menciones)
- Los temas de foros de Telegram también incluyen `MessageThreadId` e `IsForum`.

Notas específicas del canal:

- BlueBubbles puede enriquecer opcionalmente participantes sin nombre de grupos de macOS desde la base de datos local de Contactos antes de rellenar `GroupMembers`. Esto está desactivado de forma predeterminada y solo se ejecuta después de que se supere el control normal de grupos.

El prompt del sistema del agente incluye una introducción de grupo en el primer turno de una nueva sesión de grupo. Le recuerda al modelo que responda como un humano, evite tablas Markdown y evite escribir secuencias literales `\n`.

## Detalles específicos de iMessage

- Prefiere `chat_id:<id>` al enrutar o incluir en listas de permitidos.
- Lista chats: `imsg chats --limit 20`.
- Las respuestas de grupo siempre vuelven al mismo `chat_id`.

## Detalles específicos de WhatsApp

Consulta [Mensajes de grupo](/channels/group-messages) para el comportamiento específico de WhatsApp (inyección de historial, detalles del manejo de menciones).
