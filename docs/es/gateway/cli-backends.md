---
read_when:
    - Quieres un respaldo confiable cuando fallan los proveedores de API
    - Estás ejecutando Claude CLI u otras CLI locales de IA y quieres reutilizarlas
    - Quieres entender el puente MCP loopback para el acceso a herramientas del backend de CLI
summary: 'Backends de CLI: respaldo con CLI local de IA y puente opcional de herramientas MCP'
title: Backends de CLI
x-i18n:
    generated_at: "2026-04-05T12:41:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 823f3aeea6be50e5aa15b587e0944e79e862cecb7045f9dd44c93c544024bce1
    source_path: gateway/cli-backends.md
    workflow: 15
---

# Backends de CLI (runtime de respaldo)

OpenClaw puede ejecutar **CLI locales de IA** como **respaldo solo de texto** cuando los proveedores de API no funcionan,
están limitados por tasa o se comportan mal temporalmente. Esto es intencionalmente conservador:

- **Las herramientas de OpenClaw no se inyectan directamente**, pero los backends con `bundleMcp: true`
  (el valor predeterminado de Claude CLI) pueden recibir herramientas del gateway mediante un puente MCP loopback.
- **Streaming JSONL** (Claude CLI usa `--output-format stream-json` con
  `--include-partial-messages`; los prompts se envían por stdin).
- **Se admiten sesiones** (para que los turnos de seguimiento mantengan la coherencia).
- **Las imágenes pueden pasarse** si la CLI acepta rutas de imágenes.

Esto está diseñado como una **red de seguridad** más que como una ruta principal. Úsalo cuando
quieras respuestas de texto del tipo “siempre funciona” sin depender de APIs externas.

Si quieres un runtime completo tipo harness con controles de sesión ACP, tareas en segundo plano,
vinculación de hilos/conversaciones y sesiones externas persistentes de programación, usa
[Agentes ACP](/tools/acp-agents) en su lugar. Los backends de CLI no son ACP.

## Inicio rápido para principiantes

Puedes usar Claude CLI **sin ninguna configuración** (el plugin incluido de Anthropic
registra un backend predeterminado):

```bash
openclaw agent --message "hi" --model claude-cli/claude-sonnet-4-6
```

Codex CLI también funciona de inmediato (a través del plugin incluido de OpenAI):

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
        "claude-cli": {
          command: "/opt/homebrew/bin/claude",
        },
      },
    },
  },
}
```

Eso es todo. No se necesitan claves ni configuración adicional de autenticación más allá de la propia CLI.

Si usas un backend de CLI incluido como **proveedor principal de mensajes** en un
host gateway, OpenClaw ahora carga automáticamente el plugin incluido propietario cuando tu configuración
hace referencia explícita a ese backend en una referencia de modelo o en
`agents.defaults.cliBackends`.

## Usarlo como respaldo

Agrega un backend de CLI a tu lista de respaldo para que solo se ejecute cuando fallen los modelos principales:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["claude-cli/claude-sonnet-4-6", "claude-cli/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "claude-cli/claude-sonnet-4-6": {},
        "claude-cli/claude-opus-4-6": {},
      },
    },
  },
}
```

Notas:

- Si usas `agents.defaults.models` (allowlist), debes incluir `claude-cli/...`.
- Si el proveedor principal falla (autenticación, límites de tasa, tiempos de espera), OpenClaw
  intentará a continuación con el backend de CLI.
- El backend incluido de Claude CLI sigue aceptando alias más cortos como
  `claude-cli/opus`, `claude-cli/opus-4.6` o `claude-cli/sonnet`, pero la documentación
  y los ejemplos de configuración usan las referencias canónicas `claude-cli/claude-*`.

## Resumen de configuración

Todos los backends de CLI se encuentran en:

```
agents.defaults.cliBackends
```

Cada entrada usa como clave un **id de proveedor** (por ejemplo, `claude-cli`, `my-cli`).
El id del proveedor se convierte en el lado izquierdo de tu referencia de modelo:

```
<provider>/<model>
```

### Ejemplo de configuración

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "claude-cli": {
          command: "/opt/homebrew/bin/claude",
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

1. **Selecciona un backend** según el prefijo del proveedor (`claude-cli/...`).
2. **Construye un system prompt** usando el mismo prompt + contexto de espacio de trabajo de OpenClaw.
3. **Ejecuta la CLI** con un id de sesión (si se admite) para que el historial se mantenga consistente.
4. **Analiza la salida** (JSON o texto sin formato) y devuelve el texto final.
5. **Conserva los ids de sesión** por backend, para que los seguimientos reutilicen la misma sesión de CLI.

## Sesiones

- Si la CLI admite sesiones, define `sessionArg` (por ejemplo `--session-id`) o
  `sessionArgs` (marcador `{sessionId}`) cuando el ID deba insertarse
  en varios indicadores.
- Si la CLI usa un **subcomando de reanudación** con indicadores distintos, define
  `resumeArgs` (reemplaza `args` al reanudar) y opcionalmente `resumeOutput`
  (para reanudaciones no JSON).
- `sessionMode`:
  - `always`: siempre envía un id de sesión (un UUID nuevo si no hay ninguno almacenado).
  - `existing`: solo envía un id de sesión si ya se almacenó antes.
  - `none`: nunca envía un id de sesión.

Notas sobre serialización:

- `serialize: true` mantiene ordenadas las ejecuciones del mismo carril.
- La mayoría de las CLI serializan en un carril por proveedor.
- `claude-cli` es más estrecho: las ejecuciones reanudadas se serializan por id de sesión de Claude, y las ejecuciones nuevas se serializan por ruta del espacio de trabajo. Los espacios de trabajo independientes pueden ejecutarse en paralelo.
- OpenClaw descarta la reutilización de sesiones almacenadas de CLI cuando cambia el estado de autenticación del backend, incluidos nuevo inicio de sesión, rotación de token o un cambio en la credencial del perfil de autenticación.

## Imágenes (paso directo)

Si tu CLI acepta rutas de imágenes, define `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw escribirá imágenes base64 en archivos temporales. Si `imageArg` está definido, esas
rutas se pasan como argumentos de CLI. Si falta `imageArg`, OpenClaw agrega las
rutas de archivo al prompt (inyección de ruta), lo cual es suficiente para CLI que cargan
automáticamente archivos locales a partir de rutas simples (comportamiento de Claude CLI).

## Entradas / salidas

- `output: "json"` (predeterminado) intenta analizar JSON y extraer texto + id de sesión.
- Para la salida JSON de Gemini CLI, OpenClaw lee el texto de respuesta desde `response` y
  el uso desde `stats` cuando `usage` falta o está vacío.
- `output: "jsonl"` analiza flujos JSONL (por ejemplo Claude CLI `stream-json`
  y Codex CLI `--json`) y extrae el mensaje final del agente junto con identificadores
  de sesión cuando están presentes.
- `output: "text"` trata stdout como la respuesta final.

Modos de entrada:

- `input: "arg"` (predeterminado) pasa el prompt como último argumento de la CLI.
- `input: "stdin"` envía el prompt por stdin.
- Si el prompt es muy largo y `maxPromptArgChars` está definido, se usa stdin.

## Valores predeterminados (propiedad del plugin)

El plugin incluido de Anthropic registra un valor predeterminado para `claude-cli`:

- `command: "claude"`
- `args: ["-p", "--output-format", "stream-json", "--include-partial-messages", "--verbose", "--permission-mode", "bypassPermissions"]`
- `resumeArgs: ["-p", "--output-format", "stream-json", "--include-partial-messages", "--verbose", "--permission-mode", "bypassPermissions", "--resume", "{sessionId}"]`
- `output: "jsonl"`
- `input: "stdin"`
- `modelArg: "--model"`
- `systemPromptArg: "--append-system-prompt"`
- `sessionArg: "--session-id"`
- `systemPromptWhen: "first"`
- `sessionMode: "always"`

El plugin incluido de OpenAI también registra un valor predeterminado para `codex-cli`:

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

El plugin incluido de Google también registra un valor predeterminado para `google-gemini-cli`:

- `command: "gemini"`
- `args: ["--prompt", "--output-format", "json"]`
- `resumeArgs: ["--resume", "{sessionId}", "--prompt", "--output-format", "json"]`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

Requisito previo: la Gemini CLI local debe estar instalada y disponible como
`gemini` en `PATH` (`brew install gemini-cli` o
`npm install -g @google/gemini-cli`).

Notas sobre JSON de Gemini CLI:

- El texto de respuesta se lee desde el campo JSON `response`.
- El uso recurre a `stats` cuando `usage` no existe o está vacío.
- `stats.cached` se normaliza como `cacheRead` en OpenClaw.
- Si falta `stats.input`, OpenClaw deriva los tokens de entrada a partir de
  `stats.input_tokens - stats.cached`.

Reemplaza solo si es necesario (caso común: ruta `command` absoluta).

## Valores predeterminados propiedad del plugin

Los valores predeterminados de backend de CLI ahora forman parte de la superficie del plugin:

- Los plugins los registran con `api.registerCliBackend(...)`.
- El backend `id` se convierte en el prefijo de proveedor en las referencias de modelo.
- La configuración del usuario en `agents.defaults.cliBackends.<id>` sigue reemplazando el valor predeterminado del plugin.
- La limpieza de configuración específica del backend sigue siendo propiedad del plugin mediante el hook opcional `normalizeConfig`.

## Superposiciones MCP incluidas

Los backends de CLI **no** reciben llamadas a herramientas de OpenClaw directamente, pero un backend puede
optar por una superposición de configuración MCP generada con `bundleMcp: true`.

Comportamiento incluido actual:

- `claude-cli`: `bundleMcp: true` (predeterminado)
- `codex-cli`: sin superposición bundle MCP
- `google-gemini-cli`: sin superposición bundle MCP

Cuando bundle MCP está habilitado, OpenClaw:

- inicia un servidor MCP HTTP loopback que expone herramientas del gateway al proceso de la CLI
- autentica el puente con un token por sesión (`OPENCLAW_MCP_TOKEN`)
- limita el acceso a herramientas a la sesión, cuenta y contexto de canal actuales
- carga los servidores bundle-MCP habilitados para el espacio de trabajo actual
- los fusiona con cualquier `--mcp-config` existente del backend
- reescribe los argumentos de la CLI para pasar `--strict-mcp-config --mcp-config <generated-file>`

El indicador `--strict-mcp-config` impide que Claude CLI herede servidores MCP
ambientales a nivel de usuario o global. Si no hay servidores MCP habilitados, OpenClaw aun así
inyecta una configuración vacía estricta para que las ejecuciones en segundo plano permanezcan aisladas.

## Limitaciones

- **Sin llamadas directas a herramientas de OpenClaw.** OpenClaw no inyecta llamadas a herramientas en
  el protocolo del backend de CLI. Sin embargo, los backends con `bundleMcp: true` (el
  valor predeterminado de Claude CLI) reciben herramientas del gateway mediante un puente MCP loopback,
  por lo que Claude CLI puede invocar herramientas de OpenClaw a través de su compatibilidad nativa con MCP.
- **El streaming es específico del backend.** Claude CLI usa streaming JSONL
  (`stream-json` con `--include-partial-messages`); otros backends de CLI pueden
  seguir usando búfer hasta la salida.
- **Las salidas estructuradas** dependen del formato JSON de la CLI.
- **Las sesiones de Codex CLI** se reanudan mediante salida de texto (sin JSONL), que está menos
  estructurada que la ejecución inicial con `--json`. Las sesiones de OpenClaw siguen funcionando
  normalmente.

## Solución de problemas

- **CLI no encontrada**: define `command` con una ruta completa.
- **Nombre de modelo incorrecto**: usa `modelAliases` para asignar `provider/model` → modelo de CLI.
- **No hay continuidad de sesión**: asegúrate de que `sessionArg` esté definido y que `sessionMode` no sea
  `none` (actualmente Codex CLI no puede reanudar con salida JSON).
- **Las imágenes se ignoran**: define `imageArg` (y verifica que la CLI admita rutas de archivo).
