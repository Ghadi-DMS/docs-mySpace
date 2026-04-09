---
read_when:
    - Quieres un respaldo confiable cuando fallen los proveedores de API
    - Estás ejecutando Codex CLI u otras CLI de IA locales y quieres reutilizarlas
    - Quieres comprender el puente loopback de MCP para el acceso a herramientas del backend de CLI
summary: 'Backends de CLI: respaldo con CLI de IA local con puente de herramientas MCP opcional'
title: Backends de CLI
x-i18n:
    generated_at: "2026-04-09T01:27:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9b458f9fe6fa64c47864c8c180f3dedfd35c5647de470a2a4d31c26165663c20
    source_path: gateway/cli-backends.md
    workflow: 15
---

# Backends de CLI (entorno de ejecución de respaldo)

OpenClaw puede ejecutar **CLI de IA locales** como **respaldo solo de texto** cuando los proveedores de API no están disponibles,
tienen límite de tasa o presentan un comportamiento incorrecto temporal. Esto es intencionalmente conservador:

- **Las herramientas de OpenClaw no se inyectan directamente**, pero los backends con `bundleMcp: true`
  pueden recibir herramientas del gateway mediante un puente loopback de MCP.
- **Streaming JSONL** para las CLI que lo admiten.
- **Se admiten sesiones** (para que los turnos de seguimiento mantengan la coherencia).
- **Las imágenes pueden pasarse** si la CLI acepta rutas de imágenes.

Esto está diseñado como una **red de seguridad** en lugar de una ruta principal. Úsalo cuando
quieras respuestas de texto que “siempre funcionen” sin depender de APIs externas.

Si quieres un entorno de ejecución completo con controles de sesión ACP, tareas en segundo plano,
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

Eso es todo. No se necesitan claves ni configuración de autenticación extra más allá de la propia CLI.

Si usas un backend de CLI incluido como **proveedor principal de mensajes** en un
host de gateway, OpenClaw ahora carga automáticamente el plugin incluido propietario cuando tu configuración
hace referencia explícita a ese backend en una referencia de modelo o en
`agents.defaults.cliBackends`.

## Uso como respaldo

Agrega un backend de CLI a tu lista de respaldo para que solo se ejecute cuando fallen los modelos principales:

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

- Si usas `agents.defaults.models` (lista de permitidos), también debes incluir allí los modelos de tu backend de CLI.
- Si falla el proveedor principal (autenticación, límites de tasa, tiempos de espera), OpenClaw
  probará luego el backend de CLI.

## Resumen de configuración

Todos los backends de CLI viven en:

```
agents.defaults.cliBackends
```

Cada entrada está indexada por un **id de proveedor** (por ejemplo, `codex-cli`, `my-cli`).
El id de proveedor pasa a ser el lado izquierdo de tu referencia de modelo:

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
          // Las CLI de estilo Codex pueden apuntar a un archivo de prompt en su lugar:
          // systemPromptFileConfigArg: "-c",
          // systemPromptFileConfigKey: "model_instructions_file",
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
2. **Construye un prompt de sistema** usando el mismo prompt de OpenClaw + contexto del espacio de trabajo.
3. **Ejecuta la CLI** con un id de sesión (si se admite) para que el historial se mantenga consistente.
4. **Analiza la salida** (JSON o texto plano) y devuelve el texto final.
5. **Persiste los id de sesión** por backend, para que los seguimientos reutilicen la misma sesión de la CLI.

<Note>
El backend `claude-cli` incluido de Anthropic vuelve a ser compatible. El personal de Anthropic
nos dijo que el uso de Claude CLI al estilo OpenClaw vuelve a estar permitido, por lo que OpenClaw trata
el uso de `claude -p` como autorizado para esta integración, a menos que Anthropic publique
una nueva política.
</Note>

El backend `codex-cli` incluido de OpenAI pasa el prompt de sistema de OpenClaw a través de
la anulación de configuración `model_instructions_file` de Codex (`-c
model_instructions_file="..."`). Codex no expone una bandera de estilo Claude
`--append-system-prompt`, por lo que OpenClaw escribe el prompt ensamblado en un
archivo temporal para cada sesión nueva de Codex CLI.

## Sesiones

- Si la CLI admite sesiones, establece `sessionArg` (por ejemplo, `--session-id`) o
  `sessionArgs` (marcador `{sessionId}`) cuando el ID deba insertarse
  en varias banderas.
- Si la CLI usa un **subcomando de reanudación** con banderas diferentes, establece
  `resumeArgs` (reemplaza `args` al reanudar) y opcionalmente `resumeOutput`
  (para reanudaciones que no son JSON).
- `sessionMode`:
  - `always`: envía siempre un id de sesión (UUID nuevo si no hay ninguno almacenado).
  - `existing`: envía un id de sesión solo si ya había uno almacenado.
  - `none`: no envía nunca un id de sesión.

Notas sobre serialización:

- `serialize: true` mantiene ordenadas las ejecuciones en la misma vía.
- La mayoría de las CLI serializan en una vía de proveedor.
- OpenClaw descarta la reutilización de sesión almacenada de la CLI cuando cambia el estado de autenticación del backend, lo que incluye volver a iniciar sesión, rotación de tokens o un cambio en la credencial del perfil de autenticación.

## Imágenes (paso directo)

Si tu CLI acepta rutas de imágenes, establece `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw escribirá las imágenes base64 en archivos temporales. Si `imageArg` está establecido, esas
rutas se pasan como argumentos de CLI. Si falta `imageArg`, OpenClaw agrega las
rutas de archivo al prompt (inyección de ruta), lo cual es suficiente para las CLI que cargan
automáticamente archivos locales desde rutas simples.

## Entradas / salidas

- `output: "json"` (predeterminado) intenta analizar JSON y extraer texto + id de sesión.
- Para la salida JSON de Gemini CLI, OpenClaw lee el texto de respuesta desde `response` y
  el uso desde `stats` cuando falta `usage` o está vacío.
- `output: "jsonl"` analiza flujos JSONL (por ejemplo Codex CLI `--json`) y extrae el mensaje final del agente más los identificadores
  de sesión cuando están presentes.
- `output: "text"` trata stdout como la respuesta final.

Modos de entrada:

- `input: "arg"` (predeterminado) pasa el prompt como el último argumento de CLI.
- `input: "stdin"` envía el prompt por stdin.
- Si el prompt es muy largo y se establece `maxPromptArgChars`, se usa stdin.

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

Requisito previo: la Gemini CLI local debe estar instalada y disponible como
`gemini` en `PATH` (`brew install gemini-cli` o
`npm install -g @google/gemini-cli`).

Notas sobre JSON de Gemini CLI:

- El texto de respuesta se lee del campo JSON `response`.
- El uso recurre a `stats` cuando `usage` no está presente o está vacío.
- `stats.cached` se normaliza como `cacheRead` en OpenClaw.
- Si falta `stats.input`, OpenClaw deriva los tokens de entrada a partir de
  `stats.input_tokens - stats.cached`.

Anúlalo solo si es necesario (algo común: ruta `command` absoluta).

## Valores predeterminados propiedad del plugin

Los valores predeterminados del backend de CLI ahora forman parte de la superficie del plugin:

- Los plugins los registran con `api.registerCliBackend(...)`.
- El `id` del backend pasa a ser el prefijo de proveedor en las referencias de modelo.
- La configuración del usuario en `agents.defaults.cliBackends.<id>` sigue anulando el valor predeterminado del plugin.
- La limpieza de configuración específica del backend sigue siendo propiedad del plugin mediante el hook opcional
  `normalizeConfig`.

## Superposiciones de MCP incluidas

Los backends de CLI **no** reciben llamadas de herramientas de OpenClaw directamente, pero un backend puede
optar por una superposición de configuración MCP generada con `bundleMcp: true`.

Comportamiento incluido actual:

- `claude-cli`: archivo de configuración MCP estricto generado
- `codex-cli`: anulaciones de configuración en línea para `mcp_servers`
- `google-gemini-cli`: archivo de configuración del sistema Gemini generado

Cuando bundle MCP está habilitado, OpenClaw:

- inicia un servidor HTTP MCP loopback que expone herramientas del gateway al proceso de CLI
- autentica el puente con un token por sesión (`OPENCLAW_MCP_TOKEN`)
- limita el acceso a herramientas a la sesión, cuenta y contexto del canal actuales
- carga los servidores bundle-MCP habilitados para el espacio de trabajo actual
- los fusiona con cualquier forma existente de configuración/ajustes MCP del backend
- reescribe la configuración de inicio usando el modo de integración propiedad del backend desde la extensión propietaria

Si no hay servidores MCP habilitados, OpenClaw igualmente inyecta una configuración estricta cuando un
backend opta por bundle MCP para que las ejecuciones en segundo plano sigan aisladas.

## Limitaciones

- **No hay llamadas directas a herramientas de OpenClaw.** OpenClaw no inyecta llamadas de herramientas en
  el protocolo del backend de CLI. Los backends solo ven herramientas del gateway cuando optan por
  `bundleMcp: true`.
- **El streaming es específico del backend.** Algunos backends transmiten JSONL; otros almacenan en búfer
  hasta salir.
- **Las salidas estructuradas** dependen del formato JSON de la CLI.
- **Las sesiones de Codex CLI** se reanudan mediante salida de texto (sin JSONL), lo que es menos
  estructurado que la ejecución inicial con `--json`. Las sesiones de OpenClaw siguen funcionando
  normalmente.

## Solución de problemas

- **CLI no encontrada**: establece `command` en una ruta completa.
- **Nombre de modelo incorrecto**: usa `modelAliases` para asignar `provider/model` → modelo de CLI.
- **Sin continuidad de sesión**: asegúrate de que `sessionArg` esté establecido y de que `sessionMode` no sea
  `none` (actualmente Codex CLI no puede reanudarse con salida JSON).
- **Imágenes ignoradas**: establece `imageArg` (y verifica que la CLI admita rutas de archivo).
