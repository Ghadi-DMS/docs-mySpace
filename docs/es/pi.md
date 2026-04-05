---
read_when:
    - Comprender el diseño de integración del SDK de Pi en OpenClaw
    - Modificar el ciclo de vida de la sesión del agente, las herramientas o la conexión de proveedores para Pi
summary: Arquitectura de la integración del agente Pi integrado de OpenClaw y ciclo de vida de la sesión
title: Arquitectura de integración de Pi
x-i18n:
    generated_at: "2026-04-05T12:48:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 596de5fbb1430008698079f211db200e02ca8485547550fd81571a459c4c83c7
    source_path: pi.md
    workflow: 15
---

# Arquitectura de integración de Pi

Este documento describe cómo OpenClaw se integra con [pi-coding-agent](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent) y sus paquetes hermanos (`pi-ai`, `pi-agent-core`, `pi-tui`) para impulsar sus capacidades de agente de IA.

## Resumen

OpenClaw usa el SDK de pi para integrar un agente de programación con IA en su arquitectura de gateway de mensajería. En lugar de iniciar pi como un subproceso o usar modo RPC, OpenClaw importa e instancia directamente el `AgentSession` de pi mediante `createAgentSession()`. Este enfoque integrado proporciona:

- Control total sobre el ciclo de vida de la sesión y el manejo de eventos
- Inyección de herramientas personalizadas (mensajería, sandbox, acciones específicas de canal)
- Personalización del prompt del sistema por canal/contexto
- Persistencia de sesiones con compatibilidad para ramificación/compactación
- Rotación de perfiles de autenticación de varias cuentas con failover
- Cambio de modelo independiente del proveedor

## Dependencias de paquetes

```json
{
  "@mariozechner/pi-agent-core": "0.64.0",
  "@mariozechner/pi-ai": "0.64.0",
  "@mariozechner/pi-coding-agent": "0.64.0",
  "@mariozechner/pi-tui": "0.64.0"
}
```

| Paquete          | Propósito                                                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------------ |
| `pi-ai`          | Abstracciones principales de LLM: `Model`, `streamSimple`, tipos de mensajes, API de proveedores      |
| `pi-agent-core`  | Bucle del agente, ejecución de herramientas, tipos `AgentMessage`                                      |
| `pi-coding-agent` | SDK de alto nivel: `createAgentSession`, `SessionManager`, `AuthStorage`, `ModelRegistry`, herramientas integradas |
| `pi-tui`         | Componentes de interfaz de terminal (usados en el modo TUI local de OpenClaw)                         |

## Estructura de archivos

```
src/agents/
├── pi-embedded-runner.ts          # Re-exports from pi-embedded-runner/
├── pi-embedded-runner/
│   ├── run.ts                     # Main entry: runEmbeddedPiAgent()
│   ├── run/
│   │   ├── attempt.ts             # Single attempt logic with session setup
│   │   ├── params.ts              # RunEmbeddedPiAgentParams type
│   │   ├── payloads.ts            # Build response payloads from run results
│   │   ├── images.ts              # Vision model image injection
│   │   └── types.ts               # EmbeddedRunAttemptResult
│   ├── abort.ts                   # Abort error detection
│   ├── cache-ttl.ts               # Cache TTL tracking for context pruning
│   ├── compact.ts                 # Manual/auto compaction logic
│   ├── extensions.ts              # Load pi extensions for embedded runs
│   ├── extra-params.ts            # Provider-specific stream params
│   ├── google.ts                  # Google/Gemini turn ordering fixes
│   ├── history.ts                 # History limiting (DM vs group)
│   ├── lanes.ts                   # Session/global command lanes
│   ├── logger.ts                  # Subsystem logger
│   ├── model.ts                   # Model resolution via ModelRegistry
│   ├── runs.ts                    # Active run tracking, abort, queue
│   ├── sandbox-info.ts            # Sandbox info for system prompt
│   ├── session-manager-cache.ts   # SessionManager instance caching
│   ├── session-manager-init.ts    # Session file initialization
│   ├── system-prompt.ts           # System prompt builder
│   ├── tool-split.ts              # Split tools into builtIn vs custom
│   ├── types.ts                   # EmbeddedPiAgentMeta, EmbeddedPiRunResult
│   └── utils.ts                   # ThinkLevel mapping, error description
├── pi-embedded-subscribe.ts       # Session event subscription/dispatch
├── pi-embedded-subscribe.types.ts # SubscribeEmbeddedPiSessionParams
├── pi-embedded-subscribe.handlers.ts # Event handler factory
├── pi-embedded-subscribe.handlers.lifecycle.ts
├── pi-embedded-subscribe.handlers.types.ts
├── pi-embedded-block-chunker.ts   # Streaming block reply chunking
├── pi-embedded-messaging.ts       # Messaging tool sent tracking
├── pi-embedded-helpers.ts         # Error classification, turn validation
├── pi-embedded-helpers/           # Helper modules
├── pi-embedded-utils.ts           # Formatting utilities
├── pi-tools.ts                    # createOpenClawCodingTools()
├── pi-tools.abort.ts              # AbortSignal wrapping for tools
├── pi-tools.policy.ts             # Tool allowlist/denylist policy
├── pi-tools.read.ts               # Read tool customizations
├── pi-tools.schema.ts             # Tool schema normalization
├── pi-tools.types.ts              # AnyAgentTool type alias
├── pi-tool-definition-adapter.ts  # AgentTool -> ToolDefinition adapter
├── pi-settings.ts                 # Settings overrides
├── pi-hooks/                      # Custom pi hooks
│   ├── compaction-safeguard.ts    # Safeguard extension
│   ├── compaction-safeguard-runtime.ts
│   ├── context-pruning.ts         # Cache-TTL context pruning extension
│   └── context-pruning/
├── model-auth.ts                  # Auth profile resolution
├── auth-profiles.ts               # Profile store, cooldown, failover
├── model-selection.ts             # Default model resolution
├── models-config.ts               # models.json generation
├── model-catalog.ts               # Model catalog cache
├── context-window-guard.ts        # Context window validation
├── failover-error.ts              # FailoverError class
├── defaults.ts                    # DEFAULT_PROVIDER, DEFAULT_MODEL
├── system-prompt.ts               # buildAgentSystemPrompt()
├── system-prompt-params.ts        # System prompt parameter resolution
├── system-prompt-report.ts        # Debug report generation
├── tool-summaries.ts              # Tool description summaries
├── tool-policy.ts                 # Tool policy resolution
├── transcript-policy.ts           # Transcript validation policy
├── skills.ts                      # Skill snapshot/prompt building
├── skills/                        # Skill subsystem
├── sandbox.ts                     # Sandbox context resolution
├── sandbox/                       # Sandbox subsystem
├── channel-tools.ts               # Channel-specific tool injection
├── openclaw-tools.ts              # OpenClaw-specific tools
├── bash-tools.ts                  # exec/process tools
├── apply-patch.ts                 # apply_patch tool (OpenAI)
├── tools/                         # Individual tool implementations
│   ├── browser-tool.ts
│   ├── canvas-tool.ts
│   ├── cron-tool.ts
│   ├── gateway-tool.ts
│   ├── image-tool.ts
│   ├── message-tool.ts
│   ├── nodes-tool.ts
│   ├── session*.ts
│   ├── web-*.ts
│   └── ...
└── ...
```

Los runtimes de acciones de mensajes específicos de canal ahora viven en los
directorios de extensiones propiedad del plugin en lugar de bajo `src/agents/tools`, por ejemplo:

- los archivos runtime de acciones del plugin de Discord
- el archivo runtime de acciones del plugin de Slack
- el archivo runtime de acciones del plugin de Telegram
- el archivo runtime de acciones del plugin de WhatsApp

## Flujo principal de integración

### 1. Ejecutar un agente integrado

El punto de entrada principal es `runEmbeddedPiAgent()` en `pi-embedded-runner/run.ts`:

```typescript
import { runEmbeddedPiAgent } from "./agents/pi-embedded-runner.js";

const result = await runEmbeddedPiAgent({
  sessionId: "user-123",
  sessionKey: "main:whatsapp:+1234567890",
  sessionFile: "/path/to/session.jsonl",
  workspaceDir: "/path/to/workspace",
  config: openclawConfig,
  prompt: "Hello, how are you?",
  provider: "anthropic",
  model: "claude-sonnet-4-6",
  timeoutMs: 120_000,
  runId: "run-abc",
  onBlockReply: async (payload) => {
    await sendToChannel(payload.text, payload.mediaUrls);
  },
});
```

### 2. Creación de sesión

Dentro de `runEmbeddedAttempt()` (llamado por `runEmbeddedPiAgent()`), se usa el SDK de pi:

```typescript
import {
  createAgentSession,
  DefaultResourceLoader,
  SessionManager,
  SettingsManager,
} from "@mariozechner/pi-coding-agent";

const resourceLoader = new DefaultResourceLoader({
  cwd: resolvedWorkspace,
  agentDir,
  settingsManager,
  additionalExtensionPaths,
});
await resourceLoader.reload();

const { session } = await createAgentSession({
  cwd: resolvedWorkspace,
  agentDir,
  authStorage: params.authStorage,
  modelRegistry: params.modelRegistry,
  model: params.model,
  thinkingLevel: mapThinkingLevel(params.thinkLevel),
  tools: builtInTools,
  customTools: allCustomTools,
  sessionManager,
  settingsManager,
  resourceLoader,
});

applySystemPromptOverrideToSession(session, systemPromptOverride);
```

### 3. Suscripción a eventos

`subscribeEmbeddedPiSession()` se suscribe a los eventos de `AgentSession` de pi:

```typescript
const subscription = subscribeEmbeddedPiSession({
  session: activeSession,
  runId: params.runId,
  verboseLevel: params.verboseLevel,
  reasoningMode: params.reasoningLevel,
  toolResultFormat: params.toolResultFormat,
  onToolResult: params.onToolResult,
  onReasoningStream: params.onReasoningStream,
  onBlockReply: params.onBlockReply,
  onPartialReply: params.onPartialReply,
  onAgentEvent: params.onAgentEvent,
});
```

Los eventos manejados incluyen:

- `message_start` / `message_end` / `message_update` (texto/pensamiento en streaming)
- `tool_execution_start` / `tool_execution_update` / `tool_execution_end`
- `turn_start` / `turn_end`
- `agent_start` / `agent_end`
- `auto_compaction_start` / `auto_compaction_end`

### 4. Prompting

Tras la configuración, se envía el prompt a la sesión:

```typescript
await session.prompt(effectivePrompt, { images: imageResult.images });
```

El SDK gestiona el bucle completo del agente: envío al LLM, ejecución de llamadas a herramientas y streaming de respuestas.

La inyección de imágenes es local al prompt: OpenClaw carga referencias de imagen del prompt actual y
las pasa mediante `images` solo para ese turno. No vuelve a escanear turnos antiguos del historial
para volver a inyectar cargas útiles de imágenes.

## Arquitectura de herramientas

### Canalización de herramientas

1. **Herramientas base**: `codingTools` de pi (`read`, `bash`, `edit`, `write`)
2. **Reemplazos personalizados**: OpenClaw reemplaza bash por `exec`/`process`, personaliza read/edit/write para sandbox
3. **Herramientas de OpenClaw**: mensajería, browser, canvas, sesiones, cron, gateway, etc.
4. **Herramientas de canal**: herramientas de acción específicas de Discord/Telegram/Slack/WhatsApp
5. **Filtrado por políticas**: herramientas filtradas por perfil, proveedor, agente, grupo y políticas de sandbox
6. **Normalización de esquemas**: esquemas limpiados para peculiaridades de Gemini/OpenAI
7. **Envoltorio de AbortSignal**: herramientas envueltas para respetar señales de aborto

### Adaptador de definición de herramientas

El `AgentTool` de pi-agent-core tiene una firma `execute` diferente de la `ToolDefinition` de pi-coding-agent. El adaptador en `pi-tool-definition-adapter.ts` hace de puente:

```typescript
export function toToolDefinitions(tools: AnyAgentTool[]): ToolDefinition[] {
  return tools.map((tool) => ({
    name: tool.name,
    label: tool.label ?? name,
    description: tool.description ?? "",
    parameters: tool.parameters,
    execute: async (toolCallId, params, onUpdate, _ctx, signal) => {
      // pi-coding-agent signature differs from pi-agent-core
      return await tool.execute(toolCallId, params, signal, onUpdate);
    },
  }));
}
```

### Estrategia de división de herramientas

`splitSdkTools()` pasa todas las herramientas mediante `customTools`:

```typescript
export function splitSdkTools(options: { tools: AnyAgentTool[]; sandboxEnabled: boolean }) {
  return {
    builtInTools: [], // Empty. We override everything
    customTools: toToolDefinitions(options.tools),
  };
}
```

Esto garantiza que el filtrado por políticas, la integración con sandbox y el conjunto ampliado de herramientas de OpenClaw sigan siendo coherentes entre proveedores.

## Construcción del prompt del sistema

El prompt del sistema se construye en `buildAgentSystemPrompt()` (`system-prompt.ts`). Ensambla un prompt completo con secciones que incluyen Tooling, Tool Call Style, Safety guardrails, referencia de la CLI de OpenClaw, Skills, documentación, espacio de trabajo, Sandbox, mensajería, etiquetas de respuesta, voz, respuestas silenciosas, Heartbeats, metadatos de runtime, además de Memory y Reactions cuando están habilitados, y archivos de contexto opcionales y contenido adicional de prompt del sistema. Las secciones se recortan para el modo de prompt mínimo usado por subagentes.

El prompt se aplica después de crear la sesión mediante `applySystemPromptOverrideToSession()`:

```typescript
const systemPromptOverride = createSystemPromptOverride(appendPrompt);
applySystemPromptOverrideToSession(session, systemPromptOverride);
```

## Gestión de sesiones

### Archivos de sesión

Las sesiones son archivos JSONL con estructura en árbol (enlaces `id`/`parentId`). El `SessionManager` de pi gestiona la persistencia:

```typescript
const sessionManager = SessionManager.open(params.sessionFile);
```

OpenClaw lo envuelve con `guardSessionManager()` para seguridad de resultados de herramientas.

### Caché de sesiones

`session-manager-cache.ts` almacena en caché instancias de SessionManager para evitar análisis repetidos del archivo:

```typescript
await prewarmSessionFile(params.sessionFile);
sessionManager = SessionManager.open(params.sessionFile);
trackSessionManagerAccess(params.sessionFile);
```

### Límite de historial

`limitHistoryTurns()` recorta el historial de conversación según el tipo de canal (DM frente a grupo).

### Compactación

La compactación automática se activa al desbordar el contexto. Las firmas comunes de desbordamiento
incluyen `request_too_large`, `context length exceeded`, `input exceeds the
maximum number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model` y `ollama error: context
length exceeded`. `compactEmbeddedPiSessionDirect()` gestiona la compactación
manual:

```typescript
const compactResult = await compactEmbeddedPiSessionDirect({
  sessionId, sessionFile, provider, model, ...
});
```

## Autenticación y resolución de modelos

### Perfiles de autenticación

OpenClaw mantiene un almacén de perfiles de autenticación con varias claves API por proveedor:

```typescript
const authStore = ensureAuthProfileStore(agentDir, { allowKeychainPrompt: false });
const profileOrder = resolveAuthProfileOrder({ cfg, store: authStore, provider, preferredProfile });
```

Los perfiles rotan en caso de fallo con seguimiento de cooldown:

```typescript
await markAuthProfileFailure({ store, profileId, reason, cfg, agentDir });
const rotated = await advanceAuthProfile();
```

### Resolución de modelos

```typescript
import { resolveModel } from "./pi-embedded-runner/model.js";

const { model, error, authStorage, modelRegistry } = resolveModel(
  provider,
  modelId,
  agentDir,
  config,
);

// Uses pi's ModelRegistry and AuthStorage
authStorage.setRuntimeApiKey(model.provider, apiKeyInfo.apiKey);
```

### Failover

`FailoverError` activa el respaldo del modelo cuando está configurado:

```typescript
if (fallbackConfigured && isFailoverErrorMessage(errorText)) {
  throw new FailoverError(errorText, {
    reason: promptFailoverReason ?? "unknown",
    provider,
    model: modelId,
    profileId,
    status: resolveFailoverStatus(promptFailoverReason),
  });
}
```

## Extensiones de Pi

OpenClaw carga extensiones personalizadas de pi para comportamientos especializados:

### Salvaguarda de compactación

`src/agents/pi-hooks/compaction-safeguard.ts` añade protecciones a la compactación, incluidas presupuestación adaptativa de tokens y resúmenes de fallos de herramientas y operaciones de archivo:

```typescript
if (resolveCompactionMode(params.cfg) === "safeguard") {
  setCompactionSafeguardRuntime(params.sessionManager, { maxHistoryShare });
  paths.push(resolvePiExtensionPath("compaction-safeguard"));
}
```

### Depuración de contexto

`src/agents/pi-hooks/context-pruning.ts` implementa depuración de contexto basada en TTL de caché:

```typescript
if (cfg?.agents?.defaults?.contextPruning?.mode === "cache-ttl") {
  setContextPruningRuntime(params.sessionManager, {
    settings,
    contextWindowTokens,
    isToolPrunable,
    lastCacheTouchAt,
  });
  paths.push(resolvePiExtensionPath("context-pruning"));
}
```

## Streaming y respuestas por bloques

### Fragmentación por bloques

`EmbeddedBlockChunker` gestiona el streaming del texto en bloques discretos de respuesta:

```typescript
const blockChunker = blockChunking ? new EmbeddedBlockChunker(blockChunking) : null;
```

### Eliminación de etiquetas de pensamiento/final

La salida en streaming se procesa para eliminar bloques `<think>`/`<thinking>` y extraer el contenido `<final>`:

```typescript
const stripBlockTags = (text: string, state: { thinking: boolean; final: boolean }) => {
  // Strip <think>...</think> content
  // If enforceFinalTag, only return <final>...</final> content
};
```

### Directivas de respuesta

Las directivas de respuesta como `[[media:url]]`, `[[voice]]`, `[[reply:id]]` se analizan y extraen:

```typescript
const { text: cleanedText, mediaUrls, audioAsVoice, replyToId } = consumeReplyDirectives(chunk);
```

## Manejo de errores

### Clasificación de errores

`pi-embedded-helpers.ts` clasifica errores para el manejo adecuado:

```typescript
isContextOverflowError(errorText)     // Context too large
isCompactionFailureError(errorText)   // Compaction failed
isAuthAssistantError(lastAssistant)   // Auth failure
isRateLimitAssistantError(...)        // Rate limited
isFailoverAssistantError(...)         // Should failover
classifyFailoverReason(errorText)     // "auth" | "rate_limit" | "quota" | "timeout" | ...
```

### Respaldo del nivel de pensamiento

Si un nivel de pensamiento no es compatible, se usa un respaldo:

```typescript
const fallbackThinking = pickFallbackThinkingLevel({
  message: errorText,
  attempted: attemptedThinking,
});
if (fallbackThinking) {
  thinkLevel = fallbackThinking;
  continue;
}
```

## Integración con sandbox

Cuando el modo sandbox está habilitado, las herramientas y las rutas quedan restringidas:

```typescript
const sandbox = await resolveSandboxContext({
  config: params.config,
  sessionKey: sandboxSessionKey,
  workspaceDir: resolvedWorkspace,
});

if (sandboxRoot) {
  // Use sandboxed read/edit/write tools
  // Exec runs in container
  // Browser uses bridge URL
}
```

## Manejo específico por proveedor

### Anthropic

- Limpieza de cadenas mágicas de rechazo
- Validación de turnos para roles consecutivos
- Compatibilidad de parámetros de Claude Code

### Google/Gemini

- Saneamiento de esquemas de herramientas propiedad del plugin

### OpenAI

- Herramienta `apply_patch` para modelos Codex
- Manejo de reducción del nivel de pensamiento

## Integración con TUI

OpenClaw también tiene un modo TUI local que usa directamente componentes de pi-tui:

```typescript
// src/tui/tui.ts
import { ... } from "@mariozechner/pi-tui";
```

Esto proporciona la experiencia interactiva de terminal similar al modo nativo de pi.

## Diferencias clave frente a la CLI de Pi

| Aspecto         | CLI de Pi                | OpenClaw integrado                                                                              |
| --------------- | ------------------------ | ----------------------------------------------------------------------------------------------- |
| Invocación      | comando `pi` / RPC       | SDK mediante `createAgentSession()`                                                             |
| Herramientas    | herramientas de programación predeterminadas | conjunto personalizado de herramientas de OpenClaw                                  |
| Prompt del sistema | AGENTS.md + prompts   | dinámico por canal/contexto                                                                     |
| Almacenamiento de sesiones | `~/.pi/agent/sessions/` | `~/.openclaw/agents/<agentId>/sessions/` (o `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`) |
| Auth            | credencial única         | varios perfiles con rotación                                                                    |
| Extensiones     | cargadas desde disco     | programáticas + rutas en disco                                                                  |
| Manejo de eventos | renderizado TUI        | basado en callbacks (`onBlockReply`, etc.)                                                      |

## Consideraciones futuras

Áreas con potencial de rediseño:

1. **Alineación de firmas de herramientas**: actualmente se adaptan firmas entre pi-agent-core y pi-coding-agent
2. **Envoltorio del gestor de sesiones**: `guardSessionManager` añade seguridad, pero aumenta la complejidad
3. **Carga de extensiones**: podría usar `ResourceLoader` de pi de forma más directa
4. **Complejidad del manejador de streaming**: `subscribeEmbeddedPiSession` ha crecido mucho
5. **Peculiaridades de proveedores**: muchos recorridos de código específicos por proveedor que pi podría gestionar potencialmente

## Pruebas

La cobertura de la integración con Pi abarca estas suites:

- `src/agents/pi-*.test.ts`
- `src/agents/pi-auth-json.test.ts`
- `src/agents/pi-embedded-*.test.ts`
- `src/agents/pi-embedded-helpers*.test.ts`
- `src/agents/pi-embedded-runner*.test.ts`
- `src/agents/pi-embedded-runner/**/*.test.ts`
- `src/agents/pi-embedded-subscribe*.test.ts`
- `src/agents/pi-tools*.test.ts`
- `src/agents/pi-tool-definition-adapter*.test.ts`
- `src/agents/pi-settings.test.ts`
- `src/agents/pi-hooks/**/*.test.ts`

En vivo / adhesión voluntaria:

- `src/agents/pi-embedded-runner-extraparams.live.test.ts` (habilita `OPENCLAW_LIVE_TEST=1`)

Para los comandos actuales de ejecución, consulta [Flujo de desarrollo de Pi](/pi-dev).
