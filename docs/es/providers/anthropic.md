---
read_when:
    - Quieres usar modelos de Anthropic en OpenClaw
    - Quieres reutilizar la autenticación por suscripción de Claude CLI en el host del gateway
summary: Usa Anthropic Claude mediante API keys o Claude CLI en OpenClaw
title: Anthropic
x-i18n:
    generated_at: "2026-04-05T12:51:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 80f2b614eba4563093522e5157848fc54a16770a2fae69f17c54f1b9bfff624f
    source_path: providers/anthropic.md
    workflow: 15
---

# Anthropic (Claude)

Anthropic desarrolla la familia de modelos **Claude** y proporciona acceso mediante una API.
En OpenClaw, la nueva configuración de Anthropic debe usar una API key o el backend
local de Claude CLI. Los perfiles heredados de token de Anthropic ya configurados
siguen respetándose en runtime.

<Warning>
La documentación pública de Claude Code de Anthropic documenta explícitamente el
uso no interactivo de la CLI, como `claude -p`. Basándonos en esa documentación,
creemos que el respaldo local de Claude Code CLI gestionado por el usuario
probablemente esté permitido.

Por separado, Anthropic notificó a los usuarios de OpenClaw el **4 de abril de 2026 a las 12:00 PM
PT / 8:00 PM BST** que **OpenClaw cuenta como un arnés de terceros**. Según
la política que indicaron, el tráfico de inicio de sesión de Claude impulsado por OpenClaw ya no usa el
grupo de suscripción de Claude incluido y, en su lugar, requiere **Extra Usage**
(pago por uso, facturado por separado de la suscripción).

Esa distinción de política trata sobre la **reutilización de Claude CLI impulsada por OpenClaw**, no
sobre ejecutar `claude` directamente en tu propia terminal. Dicho esto, la política de Anthropic
para arneses de terceros sigue dejando suficiente ambigüedad en torno al
uso respaldado por suscripción en productos externos como para que no
recomendemos esta ruta para producción.

Documentación pública actual de Anthropic:

- [Referencia de Claude Code CLI](https://code.claude.com/docs/en/cli-reference)
- [Resumen del SDK de Claude Agent](https://platform.claude.com/docs/en/agent-sdk/overview)

- [Uso de Claude Code con tu plan Pro o Max](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [Uso de Claude Code con tu plan Team o Enterprise](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)

Si quieres la ruta de facturación más clara, usa una API key de Anthropic en su lugar.
OpenClaw también admite otras opciones de estilo suscripción, incluidas [OpenAI
Codex](/providers/openai), [Qwen Cloud Coding Plan](/providers/qwen),
[MiniMax Coding Plan](/providers/minimax) y [Z.AI / GLM Coding
Plan](/providers/glm).
</Warning>

## Opción A: API key de Anthropic

**Ideal para:** acceso estándar a la API y facturación por uso.
Crea tu API key en la consola de Anthropic.

### Configuración por CLI

```bash
openclaw onboard
# choose: Anthropic API key

# or non-interactive
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### Fragmento de configuración para Claude CLI

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Valores predeterminados de thinking (Claude 4.6)

- Los modelos Anthropic Claude 4.6 usan `adaptive` thinking de forma predeterminada en OpenClaw cuando no se establece un nivel de thinking explícito.
- Puedes sobrescribirlo por mensaje (`/think:<level>`) o en los parámetros del modelo:
  `agents.defaults.models["anthropic/<model>"].params.thinking`.
- Documentación relacionada de Anthropic:
  - [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
  - [Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## Modo rápido (API de Anthropic)

El interruptor compartido `/fast` de OpenClaw también admite tráfico directo a la API pública de Anthropic, incluidas solicitudes autenticadas con API key y OAuth enviadas a `api.anthropic.com`.

- `/fast on` se asigna a `service_tier: "auto"`
- `/fast off` se asigna a `service_tier: "standard_only"`
- Valor predeterminado en la configuración:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-sonnet-4-6": {
          params: { fastMode: true },
        },
      },
    },
  },
}
```

Límites importantes:

- OpenClaw solo inyecta service tiers de Anthropic para solicitudes directas a `api.anthropic.com`. Si enrutas `anthropic/*` a través de un proxy o gateway, `/fast` deja `service_tier` intacto.
- Los parámetros explícitos del modelo `serviceTier` o `service_tier` de Anthropic tienen prioridad sobre el valor predeterminado de `/fast` cuando ambos están establecidos.
- Anthropic informa el nivel efectivo en la respuesta bajo `usage.service_tier`. En cuentas sin capacidad de Priority Tier, `service_tier: "auto"` puede seguir resolviéndose como `standard`.

## Caché de prompts (API de Anthropic)

OpenClaw admite la función de caché de prompts de Anthropic. Esto es **solo para API**; la autenticación heredada por token de Anthropic no respeta la configuración de caché.

### Configuración

Usa el parámetro `cacheRetention` en la configuración de tu modelo:

| Valor   | Duración de caché | Descripción                  |
| ------- | ----------------- | ---------------------------- |
| `none`  | Sin caché         | Desactiva la caché de prompts |
| `short` | 5 minutos         | Predeterminado para autenticación con API Key |
| `long`  | 1 hora            | Caché extendida              |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

### Valores predeterminados

Cuando usas autenticación con API Key de Anthropic, OpenClaw aplica automáticamente `cacheRetention: "short"` (caché de 5 minutos) para todos los modelos de Anthropic. Puedes sobrescribir esto configurando explícitamente `cacheRetention` en tu configuración.

### Sobrescrituras de `cacheRetention` por agente

Usa parámetros a nivel de modelo como línea base y luego sobrescribe agentes concretos mediante `agents.list[].params`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" }, // baseline for most agents
        },
      },
    },
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } }, // override for this agent only
    ],
  },
}
```

Orden de fusión de configuración para parámetros relacionados con caché:

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params` (coincide por `id`, sobrescribe por clave)

Esto permite que un agente mantenga una caché de larga duración mientras otro agente, usando el mismo modelo, desactiva la caché para evitar costos de escritura en tráfico ráfaga o con poco reuso.

### Notas sobre Claude en Bedrock

- Los modelos Anthropic Claude en Bedrock (`amazon-bedrock/*anthropic.claude*`) aceptan el paso de `cacheRetention` cuando está configurado.
- Los modelos Bedrock que no son Anthropic se fuerzan a `cacheRetention: "none"` en runtime.
- Los valores inteligentes predeterminados de API key de Anthropic también establecen `cacheRetention: "short"` para referencias de modelos Claude-on-Bedrock cuando no hay un valor explícito.

## Ventana de contexto de 1M (beta de Anthropic)

La ventana de contexto de 1M de Anthropic está protegida por beta. En OpenClaw, habilítala por modelo
con `params.context1m: true` para los modelos Opus/Sonnet compatibles.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { context1m: true },
        },
      },
    },
  },
}
```

OpenClaw asigna esto a `anthropic-beta: context-1m-2025-08-07` en las solicitudes a Anthropic.

Esto solo se activa cuando `params.context1m` está establecido explícitamente en `true` para
ese modelo.

Requisito: Anthropic debe permitir el uso de contexto largo con esa credencial
(normalmente facturación con API key, o la ruta de inicio de sesión de Claude de OpenClaw / autenticación heredada por token
con Extra Usage habilitado). De lo contrario, Anthropic devuelve:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

Nota: Anthropic rechaza actualmente las solicitudes beta `context-1m-*` cuando se usa
autenticación heredada por token de Anthropic (`sk-ant-oat-*`). Si configuras
`context1m: true` con ese modo de autenticación heredado, OpenClaw registra una advertencia y
recurre a la ventana de contexto estándar omitiendo el encabezado beta de context1m
mientras mantiene las betas OAuth requeridas.

## Opción B: Claude CLI como proveedor de mensajes

**Ideal para:** un host de gateway de un solo usuario que ya tenga Claude CLI instalado
e iniciado, como respaldo local en lugar de la ruta recomendada para producción.

Nota de facturación: creemos que el respaldo de Claude Code CLI probablemente está permitido para automatización local gestionada por el usuario, basándonos en la documentación pública de la CLI de Anthropic. Aun así,
la política de Anthropic sobre arneses de terceros crea suficiente ambigüedad respecto al
uso respaldado por suscripción en productos externos como para que no lo recomendemos en
producción. Anthropic también comunicó a los usuarios de OpenClaw que el uso de Claude
CLI **impulsado por OpenClaw** se trata como tráfico de arnés de terceros y, desde el **4 de abril de 2026
a las 12:00 PM PT / 8:00 PM BST**, requiere **Extra Usage** en lugar de los
límites incluidos de la suscripción a Claude.

Esta ruta usa el binario local `claude` para la inferencia del modelo en lugar de llamar
directamente a la API de Anthropic. OpenClaw lo trata como un **proveedor de backend CLI**
con referencias de modelo como:

- `claude-cli/claude-sonnet-4-6`
- `claude-cli/claude-opus-4-6`

Cómo funciona:

1. OpenClaw inicia `claude -p --output-format stream-json --include-partial-messages ...`
   en el **host del gateway** y envía el prompt por stdin.
2. El primer turno envía `--session-id <uuid>`.
3. Los turnos de seguimiento reutilizan la sesión almacenada de Claude mediante `--resume <sessionId>`.
4. Tus mensajes de chat siguen pasando por la canalización normal de mensajes de OpenClaw, pero
   la respuesta real del modelo la produce Claude CLI.

### Requisitos

- Claude CLI instalado en el host del gateway y disponible en PATH, o configurado
  con una ruta absoluta al comando.
- Claude CLI ya autenticado en ese mismo host:

```bash
claude auth status
```

- OpenClaw carga automáticamente el plugin empaquetado de Anthropic al iniciar el gateway cuando tu
  configuración hace referencia explícita a `claude-cli/...` o a la configuración del backend `claude-cli`.

### Fragmento de configuración

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "claude-cli/claude-sonnet-4-6",
      },
      models: {
        "claude-cli/claude-sonnet-4-6": {},
      },
      sandbox: { mode: "off" },
    },
  },
}
```

Si el binario `claude` no está en PATH del host del gateway:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "claude-cli": {
          command: "/opt/homebrew/bin/claude",
        },
      },
    },
  },
}
```

### Qué obtienes

- Reutilización de autenticación por suscripción de Claude desde la CLI local (se lee en runtime, no se persiste)
- Enrutamiento normal de mensajes/sesiones de OpenClaw
- Continuidad de sesión de Claude CLI entre turnos (se invalida con cambios de autenticación)
- Herramientas del gateway expuestas a Claude CLI mediante puente MCP loopback
- Streaming JSONL con progreso en vivo de mensajes parciales

### Migrar de autenticación de Anthropic a Claude CLI

Si actualmente usas `anthropic/...` con un perfil heredado de token o API key y quieres
cambiar el mismo host del gateway a Claude CLI, OpenClaw admite esto como una ruta normal
de migración de autenticación de proveedor.

Requisitos previos:

- Claude CLI instalado en el **mismo host del gateway** que ejecuta OpenClaw
- Claude CLI ya autenticado allí: `claude auth login`

Luego ejecuta:

```bash
openclaw models auth login --provider anthropic --method cli --set-default
```

O durante onboarding:

```bash
openclaw onboard --auth-choice anthropic-cli
```

`openclaw onboard` interactivo y `openclaw configure` ahora priorizan primero **Anthropic
Claude CLI** y en segundo lugar **Anthropic API key**.

Esto hace lo siguiente:

- verifica que Claude CLI ya haya iniciado sesión en el host del gateway
- cambia el modelo predeterminado a `claude-cli/...`
- reescribe respaldos de modelo predeterminados de Anthropic como `anthropic/claude-opus-4-6`
  a `claude-cli/claude-opus-4-6`
- añade entradas `claude-cli/...` correspondientes a `agents.defaults.models`

Verificación rápida:

```bash
openclaw models status
```

Deberías ver el modelo principal resuelto bajo `claude-cli/...`.

Lo que **no** hace:

- eliminar tus perfiles de autenticación existentes de Anthropic
- eliminar todas las referencias antiguas `anthropic/...` de la configuración fuera de la ruta principal del
  modelo/lista de permitidos predeterminados

Eso hace que revertir sea sencillo: vuelve a cambiar el modelo predeterminado a `anthropic/...` si
lo necesitas.

### Límites importantes

- Esto **no** es el proveedor API de Anthropic. Es el runtime de la CLI local.
- OpenClaw no inyecta llamadas a herramientas directamente. Claude CLI recibe herramientas del gateway
  mediante un puente MCP loopback (`bundleMcp: true`, el valor predeterminado).
- Claude CLI transmite respuestas mediante JSONL (`stream-json` con
  `--include-partial-messages`). Los prompts se envían por stdin, no por argv.
- La autenticación se lee en runtime desde credenciales activas de Claude CLI y no se persiste
  en perfiles de OpenClaw. Los prompts de keychain se suprimen en contextos no interactivos.
- La reutilización de sesiones se sigue mediante metadatos `cliSessionBinding`. Cuando cambia el estado
  de inicio de sesión de Claude CLI (nuevo login, rotación de token), las sesiones almacenadas se
  invalidan y comienza una sesión nueva.
- Encaja mejor en un host de gateway personal, no en configuraciones compartidas de facturación multiusuario.

Más detalles: [/gateway/cli-backends](/gateway/cli-backends)

## Notas

- La documentación pública de Claude Code de Anthropic sigue documentando el uso directo de la CLI como
  `claude -p`. Creemos que el respaldo local gestionado por el usuario probablemente está permitido, pero
  la notificación independiente de Anthropic a los usuarios de OpenClaw dice que la ruta de inicio de sesión de Claude de **OpenClaw**
  es uso de arnés de terceros y requiere **Extra Usage**
  (pago por uso facturado por separado de la suscripción). Para producción,
  recomendamos usar API keys de Anthropic.
- El setup-token de Anthropic vuelve a estar disponible en OpenClaw como ruta heredada/manual. La notificación de facturación específica de Anthropic para OpenClaw sigue aplicándose, así que úsalo esperando que Anthropic requiera **Extra Usage** para esta ruta.
- Los detalles de autenticación y las reglas de reutilización están en [/concepts/oauth](/concepts/oauth).

## Resolución de problemas

**Errores 401 / token de repente no válido**

- La autenticación heredada por token de Anthropic puede expirar o revocarse.
- Para una nueva configuración, migra a una API key de Anthropic o a la ruta local de Claude CLI en el host del gateway.

**No API key found for provider "anthropic"**

- La autenticación es **por agente**. Los agentes nuevos no heredan las claves del agente principal.
- Vuelve a ejecutar onboarding para ese agente, o configura una API key en el host del gateway
  y luego verifica con `openclaw models status`.

**No credentials found for profile `anthropic:default`**

- Ejecuta `openclaw models status` para ver qué perfil de autenticación está activo.
- Vuelve a ejecutar onboarding o configura una API key o Claude CLI para esa ruta de perfil.

**No available auth profile (all in cooldown/unavailable)**

- Comprueba `openclaw models status --json` para ver `auth.unusableProfiles`.
- Los cooldowns por rate limit de Anthropic pueden depender del modelo, por lo que un modelo Anthropic
  hermano puede seguir siendo utilizable aunque el actual esté en cooldown.
- Añade otro perfil de Anthropic o espera a que termine el cooldown.

Más información: [/gateway/troubleshooting](/gateway/troubleshooting) y [/help/faq](/help/faq).
