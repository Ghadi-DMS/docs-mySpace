---
read_when:
    - Necesitas llamar asistentes del núcleo desde un plugin (TTS, STT, generación de imágenes, búsqueda web, subagente)
    - Quieres entender qué expone api.runtime
    - Estás accediendo a asistentes de configuración, agente o multimedia desde código de plugin
sidebarTitle: Runtime Helpers
summary: api.runtime -- los asistentes de tiempo de ejecución inyectados disponibles para los plugins
title: Asistentes de tiempo de ejecución de plugins
x-i18n:
    generated_at: "2026-04-08T02:17:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: acb9e56678e9ed08d0998dfafd7cd1982b592be5bc34d9e2d2c1f70274f8f248
    source_path: plugins/sdk-runtime.md
    workflow: 15
---

# Asistentes de tiempo de ejecución de plugins

Referencia para el objeto `api.runtime` inyectado en cada plugin durante el
registro. Usa estos asistentes en lugar de importar directamente elementos internos del host.

<Tip>
  **¿Buscas una guía paso a paso?** Consulta [Plugins de canal](/es/plugins/sdk-channel-plugins)
  o [Plugins de proveedor](/es/plugins/sdk-provider-plugins) para ver guías paso a paso
  que muestran estos asistentes en contexto.
</Tip>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

## Espacios de nombres del tiempo de ejecución

### `api.runtime.agent`

Identidad del agente, directorios y gestión de sesiones.

```typescript
// Resolver el directorio de trabajo del agente
const agentDir = api.runtime.agent.resolveAgentDir(cfg);

// Resolver el espacio de trabajo del agente
const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg);

// Obtener la identidad del agente
const identity = api.runtime.agent.resolveAgentIdentity(cfg);

// Obtener el nivel de razonamiento predeterminado
const thinking = api.runtime.agent.resolveThinkingDefault(cfg, provider, model);

// Obtener el tiempo de espera del agente
const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

// Asegurarse de que exista el espacio de trabajo
await api.runtime.agent.ensureAgentWorkspace(cfg);

// Ejecutar un agente Pi integrado
const agentDir = api.runtime.agent.resolveAgentDir(cfg);
const result = await api.runtime.agent.runEmbeddedPiAgent({
  sessionId: "my-plugin:task-1",
  runId: crypto.randomUUID(),
  sessionFile: path.join(agentDir, "sessions", "my-plugin-task-1.jsonl"),
  workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg),
  prompt: "Resume los cambios más recientes",
  timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
});
```

Los **asistentes del almacén de sesiones** están en `api.runtime.agent.session`:

```typescript
const storePath = api.runtime.agent.session.resolveStorePath(cfg);
const store = api.runtime.agent.session.loadSessionStore(cfg);
await api.runtime.agent.session.saveSessionStore(cfg, store);
const filePath = api.runtime.agent.session.resolveSessionFilePath(cfg, sessionId);
```

### `api.runtime.agent.defaults`

Constantes de proveedor y modelo predeterminados:

```typescript
const model = api.runtime.agent.defaults.model; // p. ej. "anthropic/claude-sonnet-4-6"
const provider = api.runtime.agent.defaults.provider; // p. ej. "anthropic"
```

### `api.runtime.subagent`

Inicia y gestiona ejecuciones de subagentes en segundo plano.

```typescript
// Iniciar una ejecución de subagente
const { runId } = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Amplía esta consulta en búsquedas de seguimiento enfocadas.",
  provider: "openai", // anulación opcional
  model: "gpt-4.1-mini", // anulación opcional
  deliver: false,
});

// Esperar a que se complete
const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

// Leer mensajes de sesión
const { messages } = await api.runtime.subagent.getSessionMessages({
  sessionKey: "agent:main:subagent:search-helper",
  limit: 10,
});

// Eliminar una sesión
await api.runtime.subagent.deleteSession({
  sessionKey: "agent:main:subagent:search-helper",
});
```

<Warning>
  Las anulaciones de modelo (`provider`/`model`) requieren aceptación explícita
  del operador mediante `plugins.entries.<id>.subagent.allowModelOverride: true` en la configuración.
  Los plugins no confiables aún pueden ejecutar subagentes, pero las solicitudes de anulación se rechazan.
</Warning>

### `api.runtime.taskFlow`

Vincula un tiempo de ejecución de Task Flow a una clave de sesión existente de OpenClaw o a un
contexto de herramienta de confianza, y luego crea y gestiona Task Flows sin pasar un propietario en cada llamada.

```typescript
const taskFlow = api.runtime.taskFlow.fromToolContext(ctx);

const created = taskFlow.createManaged({
  controllerId: "my-plugin/review-batch",
  goal: "Revisar nuevas pull requests",
});

const child = taskFlow.runTask({
  flowId: created.flowId,
  runtime: "acp",
  childSessionKey: "agent:main:subagent:reviewer",
  task: "Revisar PR #123",
  status: "running",
  startedAt: Date.now(),
});

const waiting = taskFlow.setWaiting({
  flowId: created.flowId,
  expectedRevision: created.revision,
  currentStep: "await-human-reply",
  waitJson: { kind: "reply", channel: "telegram" },
});
```

Usa `bindSession({ sessionKey, requesterOrigin })` cuando ya tengas una
clave de sesión de OpenClaw de confianza desde tu propia capa de vinculación. No vincules a partir de entrada sin procesar del usuario.

### `api.runtime.tts`

Síntesis de texto a voz.

```typescript
// TTS estándar
const clip = await api.runtime.tts.textToSpeech({
  text: "Hola desde OpenClaw",
  cfg: api.config,
});

// TTS optimizado para telefonía
const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
  text: "Hola desde OpenClaw",
  cfg: api.config,
});

// Enumerar voces disponibles
const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Usa la configuración principal `messages.tts` y la selección de proveedor. Devuelve un
búfer de audio PCM + frecuencia de muestreo.

### `api.runtime.mediaUnderstanding`

Análisis de imágenes, audio y video.

```typescript
// Describir una imagen
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

// Transcribir audio
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  mime: "audio/ogg", // opcional, para cuando no se puede inferir el MIME
});

// Describir un video
const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

// Análisis genérico de archivos
const result = await api.runtime.mediaUnderstanding.runFile({
  filePath: "/tmp/inbound-file.pdf",
  cfg: api.config,
});
```

Devuelve `{ text: undefined }` cuando no se produce ninguna salida (por ejemplo, entrada omitida).

<Info>
  `api.runtime.stt.transcribeAudioFile(...)` sigue estando disponible como alias de compatibilidad
  para `api.runtime.mediaUnderstanding.transcribeAudioFile(...)`.
</Info>

### `api.runtime.imageGeneration`

Generación de imágenes.

```typescript
const result = await api.runtime.imageGeneration.generate({
  prompt: "Un robot pintando una puesta de sol",
  cfg: api.config,
});

const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
```

### `api.runtime.webSearch`

Búsqueda web.

```typescript
const providers = api.runtime.webSearch.listProviders({ config: api.config });

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: { query: "SDK de plugins de OpenClaw", count: 5 },
});
```

### `api.runtime.media`

Utilidades multimedia de bajo nivel.

```typescript
const webMedia = await api.runtime.media.loadWebMedia(url);
const mime = await api.runtime.media.detectMime(buffer);
const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
const metadata = await api.runtime.media.getImageMetadata(filePath);
const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
```

### `api.runtime.config`

Carga y escritura de configuración.

```typescript
const cfg = await api.runtime.config.loadConfig();
await api.runtime.config.writeConfigFile(cfg);
```

### `api.runtime.system`

Utilidades a nivel de sistema.

```typescript
await api.runtime.system.enqueueSystemEvent(event);
api.runtime.system.requestHeartbeatNow();
const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
const hint = api.runtime.system.formatNativeDependencyHint(pkg);
```

### `api.runtime.events`

Suscripciones a eventos.

```typescript
api.runtime.events.onAgentEvent((event) => {
  /* ... */
});
api.runtime.events.onSessionTranscriptUpdate((update) => {
  /* ... */
});
```

### `api.runtime.logging`

Registro.

```typescript
const verbose = api.runtime.logging.shouldLogVerbose();
const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
```

### `api.runtime.modelAuth`

Resolución de autenticación de modelos y proveedores.

```typescript
const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });
const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
  provider: "openai",
  cfg,
});
```

### `api.runtime.state`

Resolución del directorio de estado.

```typescript
const stateDir = api.runtime.state.resolveStateDir();
```

### `api.runtime.tools`

Factorías de herramientas de memoria y CLI.

```typescript
const getTool = api.runtime.tools.createMemoryGetTool(/* ... */);
const searchTool = api.runtime.tools.createMemorySearchTool(/* ... */);
api.runtime.tools.registerMemoryCli(/* ... */);
```

### `api.runtime.channel`

Asistentes de tiempo de ejecución específicos del canal (disponibles cuando se carga un plugin de canal).

`api.runtime.channel.mentions` es la superficie compartida de la política de menciones entrantes para
plugins de canal empaquetados que usan inyección en tiempo de ejecución:

```typescript
const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
  mentionRegexes,
  mentionPatterns,
});

const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
  facts: {
    canDetectMention: true,
    wasMentioned: mentionMatch.matched,
    implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
      "reply_to_bot",
      isReplyToBot,
    ),
  },
  policy: {
    isGroup,
    requireMention,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});
```

Asistentes de menciones disponibles:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

`api.runtime.channel.mentions` intencionadamente no expone los asistentes heredados
de compatibilidad `resolveMentionGating*`. Prefiere la ruta normalizada
`{ facts, policy }`.

## Almacenar referencias al tiempo de ejecución

Usa `createPluginRuntimeStore` para almacenar la referencia del tiempo de ejecución y usarla fuera
de la devolución de llamada `register`:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>("my-plugin runtime not initialized");

// En tu punto de entrada
export default defineChannelPluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Example",
  plugin: myPlugin,
  setRuntime: store.setRuntime,
});

// En otros archivos
export function getRuntime() {
  return store.getRuntime(); // lanza si no está inicializado
}

export function tryGetRuntime() {
  return store.tryGetRuntime(); // devuelve null si no está inicializado
}
```

## Otros campos de nivel superior de `api`

Además de `api.runtime`, el objeto API también proporciona:

| Field                    | Type                      | Description                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Id del plugin                                                                               |
| `api.name`               | `string`                  | Nombre para mostrar del plugin                                                              |
| `api.config`             | `OpenClawConfig`          | Instantánea de la configuración actual (instantánea activa en memoria del tiempo de ejecución cuando está disponible) |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuración específica del plugin desde `plugins.entries.<id>.config`                     |
| `api.logger`             | `PluginLogger`            | Logger con alcance (`debug`, `info`, `warn`, `error`)                                       |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modo de carga actual; `"setup-runtime"` es la ventana ligera de inicio/configuración previa a la entrada completa |
| `api.resolvePath(input)` | `(string) => string`      | Resuelve una ruta relativa a la raíz del plugin                                             |

## Relacionado

- [Resumen del SDK](/es/plugins/sdk-overview) -- referencia de subrutas
- [Puntos de entrada del SDK](/es/plugins/sdk-entrypoints) -- opciones de `definePluginEntry`
- [Elementos internos de plugins](/es/plugins/architecture) -- modelo de capacidades y registro
