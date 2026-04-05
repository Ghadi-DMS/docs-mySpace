---
read_when: You want multiple isolated agents (workspaces + auth) in one gateway process.
status: active
summary: 'Enrutamiento multiagente: agentes aislados, cuentas de canal y bindings'
title: Enrutamiento multiagente
x-i18n:
    generated_at: "2026-04-05T12:40:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7e8bc48f229d01aa793ca4137e5a59f2a5ceb0ba65841710aaf69f53a672be60
    source_path: concepts/multi-agent.md
    workflow: 15
---

# Enrutamiento multiagente

Objetivo: varios agentes _aislados_ (espacio de trabajo + `agentDir` + sesiones separados), además de varias cuentas de canal (por ejemplo, dos WhatsApps) en una sola Gateway en ejecución. La entrada se enruta a un agente mediante bindings.

## ¿Qué es "un agente"?

Un **agente** es un cerebro completamente delimitado con su propio:

- **Espacio de trabajo** (archivos, AGENTS.md/SOUL.md/USER.md, notas locales, reglas de personalidad).
- **Directorio de estado** (`agentDir`) para perfiles de autenticación, registro de modelos y configuración por agente.
- **Almacén de sesiones** (historial de chat + estado de enrutamiento) en `~/.openclaw/agents/<agentId>/sessions`.

Los perfiles de autenticación son **por agente**. Cada agente lee desde su propio:

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

`sessions_history` también es aquí la vía más segura para recuperar información entre sesiones: devuelve
una vista acotada y saneada, no un volcado sin procesar de la transcripción. La recuperación del asistente elimina
etiquetas de pensamiento, andamiaje de `<relevant-memories>`, cargas XML de llamadas de herramientas en texto plano
(incluido `<tool_call>...</tool_call>`,
`<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`,
`<function_calls>...</function_calls>`, y bloques truncados de llamadas de herramientas),
andamiaje degradado de llamadas de herramientas, tokens de control del modelo filtrados en ASCII/ancho completo,
y XML malformado de llamadas de herramientas de MiniMax antes de la redacción/truncado.

Las credenciales del agente principal **no** se comparten automáticamente. Nunca reutilices `agentDir`
entre agentes (provoca colisiones de autenticación/sesión). Si quieres compartir credenciales,
copia `auth-profiles.json` al `agentDir` del otro agente.

Las Skills se cargan desde el espacio de trabajo de cada agente más raíces compartidas como
`~/.openclaw/skills`, y luego se filtran mediante la lista de permitidos efectiva de Skills del agente cuando
está configurada. Usa `agents.defaults.skills` para una línea base compartida y
`agents.list[].skills` para sustitución por agente. Consulta
[Skills: per-agent vs shared](/tools/skills#per-agent-vs-shared-skills) y
[Skills: agent skill allowlists](/tools/skills#agent-skill-allowlists).

La Gateway puede alojar **un agente** (predeterminado) o **muchos agentes** en paralelo.

**Nota sobre el espacio de trabajo:** el espacio de trabajo de cada agente es el **cwd predeterminado**, no un sandbox estricto.
Las rutas relativas se resuelven dentro del espacio de trabajo, pero las rutas absolutas pueden
alcanzar otras ubicaciones del host a menos que el sandboxing esté habilitado. Consulta
[Sandboxing](/gateway/sandboxing).

## Rutas (mapa rápido)

- Configuración: `~/.openclaw/openclaw.json` (o `OPENCLAW_CONFIG_PATH`)
- Directorio de estado: `~/.openclaw` (o `OPENCLAW_STATE_DIR`)
- Espacio de trabajo: `~/.openclaw/workspace` (o `~/.openclaw/workspace-<agentId>`)
- Directorio del agente: `~/.openclaw/agents/<agentId>/agent` (o `agents.list[].agentDir`)
- Sesiones: `~/.openclaw/agents/<agentId>/sessions`

### Modo de un solo agente (predeterminado)

Si no haces nada, OpenClaw ejecuta un único agente:

- `agentId` usa de forma predeterminada **`main`**.
- Las sesiones se identifican como `agent:main:<mainKey>`.
- El espacio de trabajo usa de forma predeterminada `~/.openclaw/workspace` (o `~/.openclaw/workspace-<profile>` cuando `OPENCLAW_PROFILE` está establecido).
- El estado usa de forma predeterminada `~/.openclaw/agents/main/agent`.

## Asistente de agentes

Usa el asistente de agentes para añadir un nuevo agente aislado:

```bash
openclaw agents add work
```

Luego añade `bindings` (o deja que el asistente lo haga) para enrutar los mensajes entrantes.

Verifica con:

```bash
openclaw agents list --bindings
```

## Inicio rápido

<Steps>
  <Step title="Crear el espacio de trabajo de cada agente">

Usa el asistente o crea los espacios de trabajo manualmente:

```bash
openclaw agents add coding
openclaw agents add social
```

Cada agente obtiene su propio espacio de trabajo con `SOUL.md`, `AGENTS.md` y `USER.md` opcional, además de un `agentDir` dedicado y un almacén de sesiones bajo `~/.openclaw/agents/<agentId>`.

  </Step>

  <Step title="Crear cuentas de canal">

Crea una cuenta por agente en tus canales preferidos:

- Discord: un bot por agente, habilita Message Content Intent y copia cada token.
- Telegram: un bot por agente mediante BotFather, copia cada token.
- WhatsApp: vincula cada número de teléfono por cuenta.

```bash
openclaw channels login --channel whatsapp --account work
```

Consulta las guías de canal: [Discord](/channels/discord), [Telegram](/channels/telegram), [WhatsApp](/channels/whatsapp).

  </Step>

  <Step title="Añadir agentes, cuentas y bindings">

Añade agentes en `agents.list`, cuentas de canal en `channels.<channel>.accounts` y conéctalos con `bindings` (ejemplos abajo).

  </Step>

  <Step title="Reiniciar y verificar">

```bash
openclaw gateway restart
openclaw agents list --bindings
openclaw channels status --probe
```

  </Step>
</Steps>

## Múltiples agentes = múltiples personas, múltiples personalidades

Con **múltiples agentes**, cada `agentId` se convierte en una **personalidad completamente aislada**:

- **Diferentes números de teléfono/cuentas** (por `accountId` de cada canal).
- **Diferentes personalidades** (mediante archivos del espacio de trabajo por agente como `AGENTS.md` y `SOUL.md`).
- **Autenticación + sesiones separadas** (sin interferencia entre ellas salvo que se habilite explícitamente).

Esto permite que **múltiples personas** compartan un servidor Gateway manteniendo sus “cerebros” de IA y datos aislados.

## Búsqueda de memoria QMD entre agentes

Si un agente debe buscar en las transcripciones de sesiones QMD de otro agente, añade
colecciones adicionales en `agents.list[].memorySearch.qmd.extraCollections`.
Usa `agents.defaults.memorySearch.qmd.extraCollections` solo cuando todos los agentes
deban heredar las mismas colecciones compartidas de transcripciones.

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspaces/main",
      memorySearch: {
        qmd: {
          extraCollections: [{ path: "~/agents/family/sessions", name: "family-sessions" }],
        },
      },
    },
    list: [
      {
        id: "main",
        workspace: "~/workspaces/main",
        memorySearch: {
          qmd: {
            extraCollections: [{ path: "notes" }], // se resuelve dentro del espacio de trabajo -> colección llamada "notes-main"
          },
        },
      },
      { id: "family", workspace: "~/workspaces/family" },
    ],
  },
  memory: {
    backend: "qmd",
    qmd: { includeDefaultMemory: false },
  },
}
```

La ruta de la colección adicional puede compartirse entre agentes, pero el nombre de la colección
permanece explícito cuando la ruta está fuera del espacio de trabajo del agente. Las rutas dentro del
espacio de trabajo siguen estando delimitadas por agente para que cada agente conserve su propio conjunto de búsqueda de transcripciones.

## Un número de WhatsApp, varias personas (división de mensajes directos)

Puedes enrutar **distintos mensajes directos de WhatsApp** a distintos agentes manteniéndote en **una sola cuenta de WhatsApp**. Haz coincidencia por E.164 del remitente (como `+15551234567`) con `peer.kind: "direct"`. Las respuestas siguen saliendo desde el mismo número de WhatsApp (sin identidad de remitente por agente).

Detalle importante: los chats directos se contraen a la **clave de sesión principal** del agente, así que el aislamiento real requiere **un agente por persona**.

Ejemplo:

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

Notas:

- El control de acceso de mensajes directos es **global por cuenta de WhatsApp** (pairing/allowlist), no por agente.
- Para grupos compartidos, vincula el grupo a un agente o usa [Broadcast groups](/channels/broadcast-groups).

## Reglas de enrutamiento (cómo los mensajes eligen un agente)

Los bindings son **deterministas** y **gana el más específico**:

1. coincidencia `peer` (id exacto de mensaje directo/grupo/canal)
2. coincidencia `parentPeer` (herencia de hilos)
3. `guildId + roles` (enrutamiento por rol de Discord)
4. `guildId` (Discord)
5. `teamId` (Slack)
6. coincidencia `accountId` para un canal
7. coincidencia a nivel de canal (`accountId: "*"`)
8. fallback al agente predeterminado (`agents.list[].default`, o en su defecto la primera entrada de la lista, predeterminado: `main`)

Si varios bindings coinciden en el mismo nivel, gana el primero según el orden de la configuración.
Si un binding establece varios campos de coincidencia (por ejemplo `peer` + `guildId`), todos los campos especificados son obligatorios (semántica `AND`).

Detalle importante sobre el ámbito de la cuenta:

- Un binding que omite `accountId` coincide solo con la cuenta predeterminada.
- Usa `accountId: "*"` para un fallback a nivel de canal en todas las cuentas.
- Si más adelante añades el mismo binding para el mismo agente con un id de cuenta explícito, OpenClaw convierte el binding existente solo de canal en uno delimitado por cuenta en lugar de duplicarlo.

## Varias cuentas / números de teléfono

Los canales que admiten **varias cuentas** (por ejemplo WhatsApp) usan `accountId` para identificar
cada inicio de sesión. Cada `accountId` puede enrutarse a un agente distinto, de modo que un servidor puede alojar
varios números de teléfono sin mezclar sesiones.

Si quieres una cuenta predeterminada a nivel de canal cuando se omite `accountId`, establece
`channels.<channel>.defaultAccount` (opcional). Si no está establecido, OpenClaw usa como fallback
`default` si existe; en caso contrario, el primer id de cuenta configurado (ordenado).

Los canales comunes que admiten este patrón incluyen:

- `whatsapp`, `telegram`, `discord`, `slack`, `signal`, `imessage`
- `irc`, `line`, `googlechat`, `mattermost`, `matrix`, `nextcloud-talk`
- `bluebubbles`, `zalo`, `zalouser`, `nostr`, `feishu`

## Conceptos

- `agentId`: un “cerebro” (espacio de trabajo, autenticación por agente, almacén de sesiones por agente).
- `accountId`: una instancia de cuenta de canal (por ejemplo, cuenta de WhatsApp `"personal"` frente a `"biz"`).
- `binding`: enruta mensajes entrantes a un `agentId` por `(channel, accountId, peer)` y opcionalmente ids de servidor/equipo.
- Los chats directos se contraen a `agent:<agentId>:<mainKey>` (principal “main” por agente; `session.mainKey`).

## Ejemplos por plataforma

### Bots de Discord por agente

Cada cuenta de bot de Discord se asigna a un `accountId` único. Vincula cada cuenta a un agente y mantén listas de permitidos por bot.

```json5
{
  agents: {
    list: [
      { id: "main", workspace: "~/.openclaw/workspace-main" },
      { id: "coding", workspace: "~/.openclaw/workspace-coding" },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "discord", accountId: "default" } },
    { agentId: "coding", match: { channel: "discord", accountId: "coding" } },
  ],
  channels: {
    discord: {
      groupPolicy: "allowlist",
      accounts: {
        default: {
          token: "DISCORD_BOT_TOKEN_MAIN",
          guilds: {
            "123456789012345678": {
              channels: {
                "222222222222222222": { allow: true, requireMention: false },
              },
            },
          },
        },
        coding: {
          token: "DISCORD_BOT_TOKEN_CODING",
          guilds: {
            "123456789012345678": {
              channels: {
                "333333333333333333": { allow: true, requireMention: false },
              },
            },
          },
        },
      },
    },
  },
}
```

Notas:

- Invita cada bot al servidor y habilita Message Content Intent.
- Los tokens viven en `channels.discord.accounts.<id>.token` (la cuenta predeterminada puede usar `DISCORD_BOT_TOKEN`).

### Bots de Telegram por agente

```json5
{
  agents: {
    list: [
      { id: "main", workspace: "~/.openclaw/workspace-main" },
      { id: "alerts", workspace: "~/.openclaw/workspace-alerts" },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "telegram", accountId: "default" } },
    { agentId: "alerts", match: { channel: "telegram", accountId: "alerts" } },
  ],
  channels: {
    telegram: {
      accounts: {
        default: {
          botToken: "123456:ABC...",
          dmPolicy: "pairing",
        },
        alerts: {
          botToken: "987654:XYZ...",
          dmPolicy: "allowlist",
          allowFrom: ["tg:123456789"],
        },
      },
    },
  },
}
```

Notas:

- Crea un bot por agente con BotFather y copia cada token.
- Los tokens viven en `channels.telegram.accounts.<id>.botToken` (la cuenta predeterminada puede usar `TELEGRAM_BOT_TOKEN`).

### Números de WhatsApp por agente

Vincula cada cuenta antes de iniciar la gateway:

```bash
openclaw channels login --channel whatsapp --account personal
openclaw channels login --channel whatsapp --account biz
```

`~/.openclaw/openclaw.json` (JSON5):

```js
{
  agents: {
    list: [
      {
        id: "home",
        default: true,
        name: "Home",
        workspace: "~/.openclaw/workspace-home",
        agentDir: "~/.openclaw/agents/home/agent",
      },
      {
        id: "work",
        name: "Work",
        workspace: "~/.openclaw/workspace-work",
        agentDir: "~/.openclaw/agents/work/agent",
      },
    ],
  },

  // Enrutamiento determinista: gana la primera coincidencia (primero el más específico).
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

    // Sobrescritura opcional por peer (ejemplo: enviar un grupo específico al agente work).
    {
      agentId: "work",
      match: {
        channel: "whatsapp",
        accountId: "personal",
        peer: { kind: "group", id: "1203630...@g.us" },
      },
    },
  ],

  // Desactivado de forma predeterminada: la mensajería entre agentes debe habilitarse explícitamente + incluirse en la lista de permitidos.
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },

  channels: {
    whatsapp: {
      accounts: {
        personal: {
          // Sobrescritura opcional. Predeterminado: ~/.openclaw/credentials/whatsapp/personal
          // authDir: "~/.openclaw/credentials/whatsapp/personal",
        },
        biz: {
          // Sobrescritura opcional. Predeterminado: ~/.openclaw/credentials/whatsapp/biz
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

## Ejemplo: chat diario en WhatsApp + trabajo profundo en Telegram

División por canal: enruta WhatsApp a un agente rápido del día a día y Telegram a un agente Opus.

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-6",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-6",
      },
    ],
  },
  bindings: [
    { agentId: "chat", match: { channel: "whatsapp" } },
    { agentId: "opus", match: { channel: "telegram" } },
  ],
}
```

Notas:

- Si tienes varias cuentas para un canal, añade `accountId` al binding (por ejemplo `{ channel: "whatsapp", accountId: "personal" }`).
- Para enrutar un único mensaje directo/grupo a Opus manteniendo el resto en chat, añade un binding `match.peer` para ese peer; las coincidencias de peer siempre ganan frente a las reglas de todo el canal.

## Ejemplo: mismo canal, un peer a Opus

Mantén WhatsApp en el agente rápido, pero enruta un mensaje directo a Opus:

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-6",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-6",
      },
    ],
  },
  bindings: [
    {
      agentId: "opus",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551234567" } },
    },
    { agentId: "chat", match: { channel: "whatsapp" } },
  ],
}
```

Los bindings de peer siempre ganan, así que mantenlos por encima de la regla de todo el canal.

## Agente familiar vinculado a un grupo de WhatsApp

Vincula un agente familiar dedicado a un único grupo de WhatsApp, con filtrado por mención
y una política de herramientas más estricta:

```json5
{
  agents: {
    list: [
      {
        id: "family",
        name: "Family",
        workspace: "~/.openclaw/workspace-family",
        identity: { name: "Family Bot" },
        groupChat: {
          mentionPatterns: ["@family", "@familybot", "@Family Bot"],
        },
        sandbox: {
          mode: "all",
          scope: "agent",
        },
        tools: {
          allow: [
            "exec",
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
        },
      },
    ],
  },
  bindings: [
    {
      agentId: "family",
      match: {
        channel: "whatsapp",
        peer: { kind: "group", id: "120363999999999999@g.us" },
      },
    },
  ],
}
```

Notas:

- Las listas allow/deny de herramientas son **herramientas**, no Skills. Si una Skill necesita ejecutar un
  binario, asegúrate de que `exec` esté permitido y de que el binario exista en el sandbox.
- Para un filtrado más estricto, establece `agents.list[].groupChat.mentionPatterns` y mantén
  habilitadas las listas de permitidos del grupo para el canal.

## Configuración de sandbox y herramientas por agente

Cada agente puede tener sus propias restricciones de sandbox y herramientas:

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // Sin sandbox para el agente personal
        },
        // Sin restricciones de herramientas: todas las herramientas disponibles
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // Siempre en sandbox
          scope: "agent",  // Un contenedor por agente
          docker: {
            // Configuración opcional de una sola vez después de crear el contenedor
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // Solo la herramienta read
          deny: ["exec", "write", "edit", "apply_patch"],    // Denegar otras
        },
      },
    ],
  },
}
```

Nota: `setupCommand` vive en `sandbox.docker` y se ejecuta una vez al crear el contenedor.
Las sobrescrituras por agente de `sandbox.docker.*` se ignoran cuando el ámbito resuelto es `"shared"`.

**Ventajas:**

- **Aislamiento de seguridad**: restringe herramientas para agentes no confiables
- **Control de recursos**: usa sandbox para agentes específicos mientras otros permanecen en el host
- **Políticas flexibles**: distintos permisos por agente

Nota: `tools.elevated` es **global** y se basa en el remitente; no se configura por agente.
Si necesitas límites por agente, usa `agents.list[].tools` para denegar `exec`.
Para orientar grupos, usa `agents.list[].groupChat.mentionPatterns` para que las @mentions se asignen claramente al agente previsto.

Consulta [Multi-Agent Sandbox & Tools](/tools/multi-agent-sandbox-tools) para ver ejemplos detallados.

## Relacionado

- [Channel Routing](/channels/channel-routing) — cómo se enrutan los mensajes a los agentes
- [Sub-Agents](/tools/subagents) — generar ejecuciones de agentes en segundo plano
- [ACP Agents](/tools/acp-agents) — ejecutar arneses externos de programación
- [Presence](/concepts/presence) — presencia y disponibilidad del agente
- [Session](/concepts/session) — aislamiento y enrutamiento de sesiones
