---
read_when:
    - Quieres una alternativa confiable cuando fallen los proveedores de API
    - Estás ejecutando Codex CLI u otros CLI de IA locales y quieres reutilizarlos
    - Quieres entender el puente loopback de MCP para el acceso a herramientas del backend de CLI
summary: 'Backends de CLI: alternativa local de CLI de IA con puente de herramientas MCP opcional'
title: Backends de CLI
x-i18n:
    generated_at: "2026-04-08T02:14:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: b0e8c41f5f5a8e34466f6b765e5c08585ef1788fa9e9d953257324bcc6cbc414
    source_path: gateway/cli-backends.md
    workflow: 15
---

# Backends de CLI (runtime alternativo)

OpenClaw puede ejecutar **CLI de IA locales** como una **alternativa solo de texto** cuando los proveedores de API no están disponibles,
están limitados por tasa o se comportan mal temporalmente. Esto es intencionalmente conservador:

- **Las herramientas de OpenClaw no se inyectan directamente**, pero los backends con `bundleMcp: true`
  pueden recibir herramientas del gateway mediante un puente MCP loopback.
- **Streaming JSONL** para los CLI que lo admiten.
- **Se admiten sesiones** (para que los turnos de seguimiento sigan siendo coherentes).
- **Las imágenes pueden pasarse** si el CLI acepta rutas de imágenes.

Esto está diseñado como una **red de seguridad** más que como una ruta principal. Úsalo cuando
quieras respuestas de texto de “siempre funciona” sin depender de API externas.

Si quieres un runtime de arnés completo con controles de sesión ACP, tareas en segundo plano,
vinculación de hilo/conversación y sesiones externas de programación persistentes, usa
[ACP Agents](/es/tools/acp-agents) en su lugar. Los backends de CLI no son ACP.

## Inicio rápido para principiantes

Puedes usar Codex CLI **sin ninguna configuración** (el plugin OpenAI incluido
registra un backend predeterminado):

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

Si tu gateway se ejecuta bajo launchd/systemd y PATH es mínimo, agrega solo la
ruta del comando:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

Eso es todo. No se necesitan claves ni configuración de autenticación adicional más allá del propio CLI.

Si usas un backend de CLI incluido como **proveedor principal de mensajes** en un
host de gateway, OpenClaw ahora carga automáticamente el plugin incluido propietario cuando tu configuración
hace referencia explícita a ese backend en una referencia de modelo o en
`agents.defaults.cliBackends`.

## Uso como alternativa

Agrega un backend de CLI a tu lista de alternativas para que solo se ejecute cuando fallen los modelos principales:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.4"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/gpt-5.4": {},
      },
    },
  },
}
```

Notas:

- Si usas `agents.defaults.models` (lista permitida), también debes incluir ahí los modelos de tu backend de CLI.
- Si falla el proveedor principal (autenticación, límites de tasa, tiempos de espera), OpenClaw
  probará después el backend de CLI.

## Resumen de configuración

Todos los backends de CLI viven en:

```
agents.defaults.cliBackends
```

Cada entrada se identifica con un **id de proveedor** (por ejemplo, `codex-cli`, `my-cli`).
El id del proveedor se convierte en la parte izquierda de tu referencia de modelo:

```
<provider>/<model>
```

### Ejemplo de configuración

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          serialize: true,
        },
      },
    },
  },
}
```

## Cómo funciona

1. **Selecciona un backend** según el prefijo del proveedor (`codex-cli/...`).
2. **Construye un prompt del sistema** usando el mismo prompt de OpenClaw + contexto del espacio de trabajo.
3. **Ejecuta el CLI** con un id de sesión (si se admite) para que el historial siga siendo consistente.
4. **Analiza la salida** (JSON o texto sin formato) y devuelve el texto final.
5. **Persiste los id de sesión** por backend, para que los seguimientos reutilicen la misma sesión del CLI.

<Note>
El backend incluido `claude-cli` de Anthropic vuelve a ser compatible. El personal de Anthropic
nos dijo que el uso de Claude CLI al estilo de OpenClaw vuelve a estar permitido, por lo que OpenClaw trata
el uso de `claude -p` como autorizado para esta integración, a menos que Anthropic publique
una nueva política.
</Note>

## Sesiones

- Si el CLI admite sesiones, configura `sessionArg` (por ejemplo, `--session-id`) o
  `sessionArgs` (marcador `{sessionId}`) cuando el ID deba insertarse
  en varias banderas.
- Si el CLI usa un **subcomando de reanudación** con banderas diferentes, configura
  `resumeArgs` (reemplaza `args` al reanudar) y, opcionalmente, `resumeOutput`
  (para reanudaciones no JSON).
- `sessionMode`:
  - `always`: siempre envía un id de sesión (un UUID nuevo si no hay ninguno almacenado).
  - `existing`: solo envía un id de sesión si ya se había almacenado antes.
  - `none`: nunca envía un id de sesión.

Notas sobre serialización:

- `serialize: true` mantiene ordenadas las ejecuciones del mismo carril.
- La mayoría de los CLI serializan en un carril de proveedor.
- OpenClaw descarta la reutilización de sesiones de CLI almacenadas cuando cambia el estado de autenticación del backend, incluido relogin, rotación de tokens o un cambio en las credenciales del perfil de autenticación.

## Imágenes (paso directo)

Si tu CLI acepta rutas de imágenes, configura `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw escribirá las imágenes base64 en archivos temporales. Si `imageArg` está configurado, esas
rutas se pasan como argumentos del CLI. Si falta `imageArg`, OpenClaw agrega las
rutas de archivo al prompt (inyección de ruta), lo que es suficiente para los CLI que cargan automáticamente
archivos locales a partir de rutas simples.

## Entradas / salidas

- `output: "json"` (predeterminado) intenta analizar JSON y extraer texto + id de sesión.
- Para la salida JSON de Gemini CLI, OpenClaw lee el texto de respuesta de `response` y
  el uso de `stats` cuando falta `usage` o está vacío.
- `output: "jsonl"` analiza streams JSONL (por ejemplo Codex CLI `--json`) y extrae el mensaje final del agente, además de los identificadores de sesión
  cuando están presentes.
- `output: "text"` trata stdout como la respuesta final.

Modos de entrada:

- `input: "arg"` (predeterminado) pasa el prompt como el último argumento del CLI.
- `input: "stdin"` envía el prompt mediante stdin.
- Si el prompt es muy largo y `maxPromptArgChars` está configurado, se usa stdin.

## Valores predeterminados (propiedad del plugin)

El plugin OpenAI incluido también registra un valor predeterminado para `codex-cli`:

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

El plugin Google incluido también registra un valor predeterminado para `google-gemini-cli`:

- `command: "gemini"`
- `args: ["--output-format", "json", "--prompt", "{prompt}"]`
- `resumeArgs: ["--resume", "{sessionId}", "--output-format", "json", "--prompt", "{prompt}"]`
- `imageArg: "@"`
- `imagePathScope: "workspace"`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

Requisito previo: el Gemini CLI local debe estar instalado y disponible como
`gemini` en `PATH` (`brew install gemini-cli` o
`npm install -g @google/gemini-cli`).

Notas sobre JSON de Gemini CLI:

- El texto de respuesta se lee del campo JSON `response`.
- El uso recurre a `stats` cuando `usage` está ausente o vacío.
- `stats.cached` se normaliza en OpenClaw como `cacheRead`.
- Si falta `stats.input`, OpenClaw deriva los tokens de entrada de
  `stats.input_tokens - stats.cached`.

Sobrescribe esto solo si es necesario (lo más común: ruta absoluta de `command`).

## Valores predeterminados propiedad del plugin

Los valores predeterminados de los backends de CLI ahora forman parte de la superficie del plugin:

- Los plugins los registran con `api.registerCliBackend(...)`.
- El `id` del backend se convierte en el prefijo del proveedor en las referencias de modelo.
- La configuración de usuario en `agents.defaults.cliBackends.<id>` sigue sobrescribiendo el valor predeterminado del plugin.
- La limpieza de configuración específica del backend sigue siendo propiedad del plugin mediante el hook opcional
  `normalizeConfig`.

## Superposiciones Bundle MCP

Los backends de CLI **no** reciben llamadas de herramientas de OpenClaw directamente, pero un backend puede
optar por una superposición de configuración MCP generada con `bundleMcp: true`.

Comportamiento incluido actual:

- `claude-cli`: archivo de configuración MCP estricto generado
- `codex-cli`: sobrescrituras de configuración en línea para `mcp_servers`
- `google-gemini-cli`: archivo de configuración del sistema Gemini generado

Cuando Bundle MCP está habilitado, OpenClaw:

- genera un servidor HTTP MCP loopback que expone herramientas del gateway al proceso del CLI
- autentica el puente con un token por sesión (`OPENCLAW_MCP_TOKEN`)
- limita el acceso a herramientas a la sesión, cuenta y contexto de canal actuales
- carga los servidores bundle-MCP habilitados para el espacio de trabajo actual
- los combina con cualquier forma existente de configuración/ajustes MCP del backend
- reescribe la configuración de inicio usando el modo de integración propiedad del backend desde la extensión propietaria

Si no hay servidores MCP habilitados, OpenClaw sigue inyectando una configuración estricta cuando un
backend opta por Bundle MCP para que las ejecuciones en segundo plano permanezcan aisladas.

## Limitaciones

- **No hay llamadas directas a herramientas de OpenClaw.** OpenClaw no inyecta llamadas de herramientas en
  el protocolo del backend de CLI. Los backends solo ven herramientas del gateway cuando optan por
  `bundleMcp: true`.
- **El streaming es específico del backend.** Algunos backends transmiten JSONL; otros almacenan
  en búfer hasta la salida.
- **Las salidas estructuradas** dependen del formato JSON del CLI.
- **Las sesiones de Codex CLI** se reanudan mediante salida de texto (sin JSONL), lo que está menos
  estructurado que la ejecución inicial con `--json`. Las sesiones de OpenClaw siguen funcionando
  normalmente.

## Solución de problemas

- **CLI no encontrado**: configura `command` con una ruta completa.
- **Nombre de modelo incorrecto**: usa `modelAliases` para mapear `provider/model` → modelo del CLI.
- **Sin continuidad de sesión**: asegúrate de que `sessionArg` esté configurado y que `sessionMode` no sea
  `none` (actualmente Codex CLI no puede reanudarse con salida JSON).
- **Imágenes ignoradas**: configura `imageArg` (y verifica que el CLI admita rutas de archivo).
