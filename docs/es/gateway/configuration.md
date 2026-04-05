---
read_when:
    - Configurar OpenClaw por primera vez
    - Buscar patrones comunes de configuración
    - Navegar a secciones específicas de configuración
summary: 'Resumen de configuración: tareas comunes, configuración rápida y enlaces a la referencia completa'
title: Configuración
x-i18n:
    generated_at: "2026-04-05T12:42:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: a39a7de09c5f9540785ec67f37d435a7a86201f0f5f640dae663054f35976712
    source_path: gateway/configuration.md
    workflow: 15
---

# Configuración

OpenClaw lee una configuración opcional en <Tooltip tip="JSON5 admite comentarios y comas finales">**JSON5**</Tooltip> desde `~/.openclaw/openclaw.json`.

Si falta el archivo, OpenClaw usa valores predeterminados seguros. Motivos comunes para añadir una configuración:

- Conectar canales y controlar quién puede enviar mensajes al bot
- Establecer modelos, herramientas, sandboxing o automatización (cron, hooks)
- Ajustar sesiones, medios, red o interfaz de usuario

Consulta la [referencia completa](/gateway/configuration-reference) para ver todos los campos disponibles.

<Tip>
**¿Eres nuevo en la configuración?** Empieza con `openclaw onboard` para una configuración interactiva, o consulta la guía [Configuration Examples](/gateway/configuration-examples) para ver configuraciones completas listas para copiar y pegar.
</Tip>

## Configuración mínima

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## Editar la configuración

<Tabs>
  <Tab title="Asistente interactivo">
    ```bash
    openclaw onboard       # flujo completo de onboarding
    openclaw configure     # asistente de configuración
    ```
  </Tab>
  <Tab title="CLI (one-liners)">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```
  </Tab>
  <Tab title="Interfaz de usuario de Control">
    Abre [http://127.0.0.1:18789](http://127.0.0.1:18789) y usa la pestaña **Config**.
    La interfaz de usuario de Control renderiza un formulario a partir del esquema de configuración activo, incluidos los metadatos de documentación de campo
    `title` / `description`, además de esquemas de plugins y canales cuando
    están disponibles, con un editor de **Raw JSON** como vía de escape. Para interfaces
    con mayor nivel de detalle y otras herramientas, la gateway también expone `config.schema.lookup` para
    obtener un nodo del esquema delimitado a una ruta junto con resúmenes inmediatos de sus hijos.
  </Tab>
  <Tab title="Edición directa">
    Edita `~/.openclaw/openclaw.json` directamente. La Gateway observa el archivo y aplica los cambios automáticamente (consulta [recarga en caliente](#config-hot-reload)).
  </Tab>
</Tabs>

## Validación estricta

<Warning>
OpenClaw solo acepta configuraciones que coincidan por completo con el esquema. Las claves desconocidas, los tipos mal formados o los valores no válidos hacen que la Gateway **se niegue a iniciarse**. La única excepción en el nivel raíz es `$schema` (cadena), para que los editores puedan adjuntar metadatos de JSON Schema.
</Warning>

Notas sobre herramientas del esquema:

- `openclaw config schema` imprime la misma familia de JSON Schema usada por la interfaz de usuario de Control
  y la validación de configuración.
- Los valores `title` y `description` de los campos se trasladan a la salida del esquema para
  herramientas de edición y formularios.
- Las entradas de objetos anidados, comodines (`*`) y elementos de arreglo (`[]`) heredan los mismos
  metadatos de documentación cuando existe documentación de campo coincidente.
- Las ramas de composición `anyOf` / `oneOf` / `allOf` también heredan los mismos metadatos de documentación,
  por lo que las variantes de unión/intersección conservan la misma ayuda de campo.
- `config.schema.lookup` devuelve una ruta de configuración normalizada con un nodo superficial
  del esquema (`title`, `description`, `type`, `enum`, `const`, límites comunes
  y campos de validación similares), metadatos coincidentes de sugerencias de interfaz y resúmenes
  inmediatos de sus hijos para herramientas con navegación detallada.
- Los esquemas dinámicos de plugins/canales se fusionan cuando la gateway puede cargar el
  registro actual de manifiestos.

Cuando falla la validación:

- La Gateway no inicia
- Solo funcionan los comandos de diagnóstico (`openclaw doctor`, `openclaw logs`, `openclaw health`, `openclaw status`)
- Ejecuta `openclaw doctor` para ver los problemas exactos
- Ejecuta `openclaw doctor --fix` (o `--yes`) para aplicar reparaciones

## Tareas comunes

<AccordionGroup>
  <Accordion title="Configurar un canal (WhatsApp, Telegram, Discord, etc.)">
    Cada canal tiene su propia sección de configuración en `channels.<provider>`. Consulta la página dedicada de cada canal para ver los pasos de configuración:

    - [WhatsApp](/channels/whatsapp) — `channels.whatsapp`
    - [Telegram](/channels/telegram) — `channels.telegram`
    - [Discord](/channels/discord) — `channels.discord`
    - [Feishu](/channels/feishu) — `channels.feishu`
    - [Google Chat](/channels/googlechat) — `channels.googlechat`
    - [Microsoft Teams](/channels/msteams) — `channels.msteams`
    - [Slack](/channels/slack) — `channels.slack`
    - [Signal](/channels/signal) — `channels.signal`
    - [iMessage](/channels/imessage) — `channels.imessage`
    - [Mattermost](/channels/mattermost) — `channels.mattermost`

    Todos los canales comparten el mismo patrón de política de mensajes directos:

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // pairing | allowlist | open | disabled
          allowFrom: ["tg:123"], // solo para allowlist/open
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Elegir y configurar modelos">
    Establece el modelo principal y los fallbacks opcionales:

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-6",
            fallbacks: ["openai/gpt-5.4"],
          },
          models: {
            "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
            "openai/gpt-5.4": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - `agents.defaults.models` define el catálogo de modelos y actúa como la lista de permitidos para `/model`.
    - Las referencias de modelo usan el formato `provider/model` (por ejemplo `anthropic/claude-opus-4-6`).
    - `agents.defaults.imageMaxDimensionPx` controla la reducción de escala de imágenes en transcripciones/herramientas (predeterminado `1200`); los valores más bajos suelen reducir el uso de vision tokens en ejecuciones con muchas capturas de pantalla.
    - Consulta [Models CLI](/concepts/models) para cambiar de modelo en el chat y [Model Failover](/concepts/model-failover) para la rotación de autenticación y el comportamiento de fallback.
    - Para proveedores personalizados/autohospedados, consulta [Custom providers](/gateway/configuration-reference#custom-providers-and-base-urls) en la referencia.

  </Accordion>

  <Accordion title="Controlar quién puede enviar mensajes al bot">
    El acceso por mensajes directos se controla por canal mediante `dmPolicy`:

    - `"pairing"` (predeterminado): los remitentes desconocidos reciben un código de emparejamiento de un solo uso para aprobar
    - `"allowlist"`: solo remitentes en `allowFrom` (o en el almacén de permitidos emparejado)
    - `"open"`: permite todos los mensajes directos entrantes (requiere `allowFrom: ["*"]`)
    - `"disabled"`: ignora todos los mensajes directos

    Para grupos, usa `groupPolicy` + `groupAllowFrom` o listas de permitidos específicas del canal.

    Consulta la [referencia completa](/gateway/configuration-reference#dm-and-group-access) para ver detalles por canal.

  </Accordion>

  <Accordion title="Configurar el filtro por mención en chats de grupo">
    Los mensajes de grupo requieren **mención** de forma predeterminada. Configura patrones por agente:

    ```json5
    {
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **Menciones de metadatos**: menciones nativas con @ (mencionar con toque en WhatsApp, @bot en Telegram, etc.)
    - **Patrones de texto**: patrones regex seguros en `mentionPatterns`
    - Consulta la [referencia completa](/gateway/configuration-reference#group-chat-mention-gating) para ver sobrescrituras por canal y el modo de chat propio.

  </Accordion>

  <Accordion title="Restringir Skills por agente">
    Usa `agents.defaults.skills` para una línea base compartida y luego sobrescribe agentes específicos
    con `agents.list[].skills`:

    ```json5
    {
      agents: {
        defaults: {
          skills: ["github", "weather"],
        },
        list: [
          { id: "writer" }, // hereda github, weather
          { id: "docs", skills: ["docs-search"] }, // reemplaza los valores predeterminados
          { id: "locked-down", skills: [] }, // sin Skills
        ],
      },
    }
    ```

    - Omite `agents.defaults.skills` para Skills sin restricciones de forma predeterminada.
    - Omite `agents.list[].skills` para heredar los valores predeterminados.
    - Establece `agents.list[].skills: []` para no tener Skills.
    - Consulta [Skills](/tools/skills), [Skills config](/tools/skills-config) y
      la [Configuration Reference](/gateway/configuration-reference#agentsdefaultsskills).

  </Accordion>

  <Accordion title="Ajustar la supervisión de salud de canales de la gateway">
    Controla con qué agresividad la gateway reinicia canales que parecen obsoletos:

    ```json5
    {
      gateway: {
        channelHealthCheckMinutes: 5,
        channelStaleEventThresholdMinutes: 30,
        channelMaxRestartsPerHour: 10,
      },
      channels: {
        telegram: {
          healthMonitor: { enabled: false },
          accounts: {
            alerts: {
              healthMonitor: { enabled: true },
            },
          },
        },
      },
    }
    ```

    - Establece `gateway.channelHealthCheckMinutes: 0` para desactivar globalmente los reinicios del monitor de salud.
    - `channelStaleEventThresholdMinutes` debe ser mayor o igual que el intervalo de comprobación.
    - Usa `channels.<provider>.healthMonitor.enabled` o `channels.<provider>.accounts.<id>.healthMonitor.enabled` para desactivar los reinicios automáticos de un canal o cuenta sin desactivar el monitor global.
    - Consulta [Health Checks](/gateway/health) para depuración operativa y la [referencia completa](/gateway/configuration-reference#gateway) para todos los campos.

  </Accordion>

  <Accordion title="Configurar sesiones y reinicios">
    Las sesiones controlan la continuidad y el aislamiento de la conversación:

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // recomendado para varios usuarios
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: `main` (compartido) | `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: valores predeterminados globales para el enrutamiento de sesiones vinculadas a hilos (Discord admite `/focus`, `/unfocus`, `/agents`, `/session idle` y `/session max-age`).
    - Consulta [Session Management](/concepts/session) para alcance, enlaces de identidad y política de envío.
    - Consulta la [referencia completa](/gateway/configuration-reference#session) para todos los campos.

  </Accordion>

  <Accordion title="Habilitar sandboxing">
    Ejecuta sesiones de agente en contenedores Docker aislados:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    Construye primero la imagen: `scripts/sandbox-setup.sh`

    Consulta [Sandboxing](/gateway/sandboxing) para la guía completa y la [referencia completa](/gateway/configuration-reference#agentsdefaultssandbox) para todas las opciones.

  </Accordion>

  <Accordion title="Habilitar push respaldado por relay para compilaciones oficiales de iOS">
    El push respaldado por relay se configura en `openclaw.json`.

    Establece esto en la configuración de gateway:

    ```json5
    {
      gateway: {
        push: {
          apns: {
            relay: {
              baseUrl: "https://relay.example.com",
              // Opcional. Predeterminado: 10000
              timeoutMs: 10000,
            },
          },
        },
      },
    }
    ```

    Equivalente en CLI:

    ```bash
    openclaw config set gateway.push.apns.relay.baseUrl https://relay.example.com
    ```

    Qué hace esto:

    - Permite que la gateway envíe `push.test`, avisos de activación y activaciones de reconexión mediante el relay externo.
    - Usa un permiso de envío delimitado al registro, reenviado por la app iOS emparejada. La gateway no necesita un token de relay para todo el despliegue.
    - Vincula cada registro respaldado por relay a la identidad de gateway con la que se emparejó la app iOS, de modo que otra gateway no pueda reutilizar el registro almacenado.
    - Mantiene las compilaciones locales/manuales de iOS en APNs directo. Los envíos respaldados por relay se aplican solo a compilaciones oficiales distribuidas que se registraron mediante el relay.
    - Debe coincidir con la URL base del relay incorporada en la compilación oficial/TestFlight de iOS, para que el registro y el tráfico de envío lleguen al mismo despliegue de relay.

    Flujo de extremo a extremo:

    1. Instala una compilación oficial/TestFlight de iOS compilada con la misma URL base de relay.
    2. Configura `gateway.push.apns.relay.baseUrl` en la gateway.
    3. Empareja la app iOS con la gateway y deja que se conecten tanto las sesiones de nodo como las de operador.
    4. La app iOS obtiene la identidad de la gateway, se registra con el relay usando App Attest más el recibo de la app, y luego publica la carga `push.apns.register` respaldada por relay en la gateway emparejada.
    5. La gateway almacena el identificador del relay y el permiso de envío, y luego los usa para `push.test`, avisos de activación y activaciones de reconexión.

    Notas operativas:

    - Si cambias la app iOS a otra gateway, vuelve a conectar la app para que pueda publicar un nuevo registro de relay vinculado a esa gateway.
    - Si distribuyes una nueva compilación iOS que apunta a un despliegue diferente de relay, la app actualiza su registro de relay almacenado en caché en lugar de reutilizar el origen de relay anterior.

    Nota de compatibilidad:

    - `OPENCLAW_APNS_RELAY_BASE_URL` y `OPENCLAW_APNS_RELAY_TIMEOUT_MS` siguen funcionando como sobrescrituras temporales de entorno.
    - `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true` sigue siendo una vía de escape de desarrollo solo para loopback; no conserves URL de relay HTTP en la configuración.

    Consulta [iOS App](/platforms/ios#relay-backed-push-for-official-builds) para el flujo completo e [Authentication and trust flow](/platforms/ios#authentication-and-trust-flow) para el modelo de seguridad del relay.

  </Accordion>

  <Accordion title="Configurar heartbeat (comprobaciones periódicas)">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: cadena de duración (`30m`, `2h`). Establece `0m` para desactivar.
    - `target`: `last` | `none` | `<channel-id>` (por ejemplo `discord`, `matrix`, `telegram` o `whatsapp`)
    - `directPolicy`: `allow` (predeterminado) o `block` para objetivos de heartbeat de estilo DM
    - Consulta [Heartbeat](/gateway/heartbeat) para la guía completa.

  </Accordion>

  <Accordion title="Configurar cron jobs">
    ```json5
    {
      cron: {
        enabled: true,
        maxConcurrentRuns: 2,
        sessionRetention: "24h",
        runLog: {
          maxBytes: "2mb",
          keepLines: 2000,
        },
      },
    }
    ```

    - `sessionRetention`: depura sesiones aisladas de ejecuciones completadas de `sessions.json` (predeterminado `24h`; establece `false` para desactivar).
    - `runLog`: depura `cron/runs/<jobId>.jsonl` por tamaño y líneas retenidas.
    - Consulta [Cron Jobs](/automation/cron-jobs) para ver la visión general de la función y ejemplos de CLI.

  </Accordion>

  <Accordion title="Configurar webhooks (hooks)">
    Habilita endpoints HTTP webhook en la Gateway:

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    Nota de seguridad:
    - Trata todo el contenido de las cargas hook/webhook como entrada no confiable.
    - Usa un `hooks.token` dedicado; no reutilices el token compartido de Gateway.
    - La autenticación de hooks es solo por encabezado (`Authorization: Bearer ...` o `x-openclaw-token`); se rechazan tokens en query string.
    - `hooks.path` no puede ser `/`; mantén la entrada de webhooks en una subruta dedicada como `/hooks`.
    - Mantén desactivadas las banderas de omisión de contenido inseguro (`hooks.gmail.allowUnsafeExternalContent`, `hooks.mappings[].allowUnsafeExternalContent`) salvo para depuración muy delimitada.
    - Si habilitas `hooks.allowRequestSessionKey`, establece también `hooks.allowedSessionKeyPrefixes` para acotar las claves de sesión seleccionadas por quien llama.
    - Para agentes impulsados por hooks, prefiere niveles modernos de modelo fuertes y una política estricta de herramientas (por ejemplo, solo mensajería más sandboxing cuando sea posible).

    Consulta la [referencia completa](/gateway/configuration-reference#hooks) para ver todas las opciones de mapping y la integración con Gmail.

  </Accordion>

  <Accordion title="Configurar enrutamiento multiagente">
    Ejecuta varios agentes aislados con espacios de trabajo y sesiones separados:

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    Consulta [Multi-Agent](/concepts/multi-agent) y la [referencia completa](/gateway/configuration-reference#multi-agent-routing) para ver reglas de binding y perfiles de acceso por agente.

  </Accordion>

  <Accordion title="Dividir la configuración en varios archivos ($include)">
    Usa `$include` para organizar configuraciones grandes:

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **Un solo archivo**: reemplaza el objeto contenedor
    - **Arreglo de archivos**: se fusionan en profundidad en orden (el último gana)
    - **Claves vecinas**: se fusionan después de los includes (sobrescriben los valores incluidos)
    - **Includes anidados**: compatibles hasta 10 niveles de profundidad
    - **Rutas relativas**: se resuelven en relación con el archivo que incluye
    - **Manejo de errores**: errores claros para archivos faltantes, errores de análisis e includes circulares

  </Accordion>
</AccordionGroup>

## Recarga en caliente de la configuración

La Gateway observa `~/.openclaw/openclaw.json` y aplica los cambios automáticamente: no hace falta reiniciar manualmente para la mayoría de los ajustes.

### Modos de recarga

| Modo                   | Comportamiento                                                                        |
| ---------------------- | ------------------------------------------------------------------------------------- |
| **`hybrid`** (predeterminado) | Aplica en caliente al instante los cambios seguros. Reinicia automáticamente para los críticos. |
| **`hot`**              | Solo aplica en caliente los cambios seguros. Registra una advertencia cuando se necesita reiniciar; tú te encargas. |
| **`restart`**          | Reinicia la Gateway ante cualquier cambio de configuración, seguro o no.              |
| **`off`**              | Desactiva la observación de archivos. Los cambios se aplican en el siguiente reinicio manual. |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### Qué se aplica en caliente y qué requiere reinicio

La mayoría de los campos se aplican en caliente sin tiempo de inactividad. En modo `hybrid`, los cambios que requieren reinicio se gestionan automáticamente.

| Categoría            | Campos                                                               | ¿Requiere reinicio? |
| ------------------- | -------------------------------------------------------------------- | ------------------- |
| Canales            | `channels.*`, `web` (WhatsApp): todos los canales integrados y de extensión | No              |
| Agente y modelos      | `agent`, `agents`, `models`, `routing`                               | No                  |
| Automatización          | `hooks`, `cron`, `agent.heartbeat`                                   | No                  |
| Sesiones y mensajes | `session`, `messages`                                                | No                  |
| Herramientas y medios       | `tools`, `browser`, `skills`, `audio`, `talk`                        | No                  |
| UI y varios           | `ui`, `logging`, `identity`, `bindings`                              | No                  |
| Servidor Gateway      | `gateway.*` (port, bind, auth, tailscale, TLS, HTTP)                 | **Sí**              |
| Infraestructura      | `discovery`, `canvasHost`, `plugins`                                 | **Sí**              |

<Note>
`gateway.reload` y `gateway.remote` son excepciones: cambiarlos **no** provoca reinicio.
</Note>

## RPC de configuración (actualizaciones programáticas)

<Note>
Las RPC de escritura del plano de control (`config.apply`, `config.patch`, `update.run`) tienen un límite de frecuencia de **3 solicitudes por 60 segundos** por `deviceId+clientIp`. Cuando se alcanza el límite, la RPC devuelve `UNAVAILABLE` con `retryAfterMs`.
</Note>

Flujo seguro/predeterminado:

- `config.schema.lookup`: inspecciona un subárbol de configuración delimitado a una ruta con un nodo superficial
  del esquema, metadatos coincidentes de sugerencias y resúmenes inmediatos de sus hijos
- `config.get`: obtiene la instantánea actual + hash
- `config.patch`: ruta preferida para actualizaciones parciales
- `config.apply`: solo para reemplazo completo de configuración
- `update.run`: autoactualización explícita + reinicio

Cuando no vayas a reemplazar toda la configuración, prefiere `config.schema.lookup`
y luego `config.patch`.

<AccordionGroup>
  <Accordion title="config.apply (reemplazo completo)">
    Valida + escribe la configuración completa y reinicia la Gateway en un solo paso.

    <Warning>
    `config.apply` reemplaza la **configuración completa**. Usa `config.patch` para actualizaciones parciales, o `openclaw config set` para claves individuales.
    </Warning>

    Parámetros:

    - `raw` (string): carga JSON5 para toda la configuración
    - `baseHash` (opcional): hash de configuración de `config.get` (obligatorio cuando la configuración existe)
    - `sessionKey` (opcional): clave de sesión para el ping de reactivación posterior al reinicio
    - `note` (opcional): nota para el sentinel de reinicio
    - `restartDelayMs` (opcional): retraso antes del reinicio (predeterminado 2000)

    Las solicitudes de reinicio se consolidan mientras ya hay una pendiente/en curso, y se aplica un tiempo de espera de 30 segundos entre ciclos de reinicio.

    ```bash
    openclaw gateway call config.get --params '{}'  # capturar payload.hash
    openclaw gateway call config.apply --params '{
      "raw": "{ agents: { defaults: { workspace: \"~/.openclaw/workspace\" } } }",
      "baseHash": "<hash>",
      "sessionKey": "agent:main:whatsapp:direct:+15555550123"
    }'
    ```

  </Accordion>

  <Accordion title="config.patch (actualización parcial)">
    Fusiona una actualización parcial en la configuración existente (semántica de JSON merge patch):

    - Los objetos se fusionan recursivamente
    - `null` elimina una clave
    - Los arreglos se reemplazan

    Parámetros:

    - `raw` (string): JSON5 con solo las claves que se deben cambiar
    - `baseHash` (obligatorio): hash de configuración de `config.get`
    - `sessionKey`, `note`, `restartDelayMs`: iguales que en `config.apply`

    El comportamiento de reinicio coincide con `config.apply`: reinicios pendientes consolidados más un tiempo de espera de 30 segundos entre ciclos de reinicio.

    ```bash
    openclaw gateway call config.patch --params '{
      "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
      "baseHash": "<hash>"
    }'
    ```

  </Accordion>
</AccordionGroup>

## Variables de entorno

OpenClaw lee variables de entorno del proceso padre, además de:

- `.env` desde el directorio de trabajo actual (si existe)
- `~/.openclaw/.env` (fallback global)

Ninguno de los dos archivos sobrescribe variables de entorno existentes. También puedes establecer variables de entorno inline en la configuración:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="Importación de entorno del shell (opcional)">
  Si está habilitado y faltan las claves esperadas, OpenClaw ejecuta tu shell de inicio de sesión e importa solo las claves que faltan:

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

Equivalente en variable de entorno: `OPENCLAW_LOAD_SHELL_ENV=1`
</Accordion>

<Accordion title="Sustitución de variables de entorno en valores de configuración">
  Haz referencia a variables de entorno en cualquier valor de cadena de la configuración con `${VAR_NAME}`:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

Reglas:

- Solo coinciden nombres en mayúsculas: `[A-Z_][A-Z0-9_]*`
- Variables faltantes/vacías provocan un error al cargar
- Escapa con `$${VAR}` para salida literal
- Funciona dentro de archivos `$include`
- Sustitución inline: `"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="Refs de secretos (env, file, exec)">
  Para los campos que admiten objetos SecretRef, puedes usar:

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "image-lab": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/image-lab/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccountRef: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

Los detalles de SecretRef (incluido `secrets.providers` para `env`/`file`/`exec`) están en [Secrets Management](/gateway/secrets).
Las rutas de credenciales compatibles se enumeran en [SecretRef Credential Surface](/reference/secretref-credential-surface).
</Accordion>

Consulta [Environment](/help/environment) para ver la precedencia completa y las fuentes.

## Referencia completa

Para la referencia completa campo por campo, consulta **[Configuration Reference](/gateway/configuration-reference)**.

---

_Relacionado: [Configuration Examples](/gateway/configuration-examples) · [Configuration Reference](/gateway/configuration-reference) · [Doctor](/gateway/doctor)_
