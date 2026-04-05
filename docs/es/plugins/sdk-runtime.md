---
read_when:
    - Necesitas llamar a auxiliares del núcleo desde un plugin (TTS, STT, generación de imágenes, búsqueda web, subagente)
    - Quieres entender qué expone api.runtime
    - Estás accediendo a auxiliares de configuración, agente o medios desde código de plugin
sidebarTitle: Runtime Helpers
summary: 'api.runtime: los auxiliares de runtime inyectados disponibles para los plugins'
title: Auxiliares de runtime de plugins
x-i18n:
    generated_at: "2026-04-05T12:50:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 667edff734fd30f9b05d55eae6360830a45ae8f3012159f88a37b5e05404e666
    source_path: plugins/sdk-runtime.md
    workflow: 15
---

# Auxiliares de runtime de plugins

Referencia para el objeto `api.runtime` inyectado en cada plugin durante el
registro. Usa estos auxiliares en lugar de importar directamente internos del host.

<Tip>
  **¿Buscas un recorrido guiado?** Consulta [Plugins de canal](/plugins/sdk-channel-plugins)
  o [Plugins de proveedor](/plugins/sdk-provider-plugins) para ver guías paso a paso
  que muestran estos auxiliares en contexto.
</Tip>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

## Espacios de nombres de runtime

### `api.runtime.agent`

Identidad del agente, directorios y gestión de sesiones.

```typescript
// Resolver el directorio de trabajo del agente
const agentDir = api.runtime.agent.resolveAgentDir(cfg);

// Resolver el espacio de trabajo del agente
const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg);

// Obtener la identidad del agente
const identity = api.runtime.agent.resolveAgentIdentity(cfg);

// Obtener el nivel de pensamiento predeterminado
const thinking = api.runtime.agent.resolveThinkingDefault(cfg, provider, model);

// Obtener el tiempo de espera del agente
const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

// Asegurar que existe el espacio de trabajo
await api.runtime.agent.ensureAgentWorkspace(cfg);

// Ejecutar un agente Pi integrado
const agentDir = api.runtime.agent.resolveAgentDir(cfg);
const result = await api.runtime.agent.runEmbeddedPiAgent({
  sessionId: "my-plugin:task-1",
  runId: crypto.randomUUID(),
  sessionFile: path.join(agentDir, "sessions", "my-plugin-task-1.jsonl"),
  workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg),
  prompt: "Summarize the latest changes",
  timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
});
```

Los **auxiliares del almacén de sesiones** están en `api.runtime.agent.session`:

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
  message: "Expand this query into focused follow-up searches.",
  provider: "openai", // anulación opcional
  model: "gpt-4.1-mini", // anulación opcional
  deliver: false,
});

// Esperar a que termine
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
  Las anulaciones de modelo (`provider`/`model`) requieren adhesión del operador mediante
  `plugins.entries.<id>.subagent.allowModelOverride: true` en la configuración.
  Los plugins no confiables aún pueden ejecutar subagentes, pero las solicitudes de anulación se rechazan.
</Warning>

### `api.runtime.taskFlow`

Vincula un runtime de Task Flow a una clave de sesión existente de OpenClaw o a un contexto
de herramienta de confianza, y luego crea y gestiona Task Flows sin pasar un propietario en cada llamada.

```typescript
const taskFlow = api.runtime.taskFlow.fromToolContext(ctx);

const created = taskFlow.createManaged({
  controllerId: "my-plugin/review-batch",
  goal: "Review new pull requests",
});

const child = taskFlow.runTask({
  flowId: created.flowId,
  runtime: "acp",
  childSessionKey: "agent:main:subagent:reviewer",
  task: "Review PR #123",
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
clave de sesión de OpenClaw de confianza de tu propia capa de vinculación. No vincules a partir de entrada bruta
del usuario.

### `api.runtime.tts`

Síntesis de texto a voz.

```typescript
// TTS estándar
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

// TTS optimizado para telefonía
const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

// Enumerar voces disponibles
const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Usa la selección de proveedor y la configuración `messages.tts` del núcleo. Devuelve un
búfer de audio PCM + frecuencia de muestreo.

### `api.runtime.mediaUnderstanding`

Análisis de imagen, audio y video.

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
  mime: "audio/ogg", // opcional, cuando no se puede inferir el MIME
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

Devuelve `{ text: undefined }` cuando no se produce salida (p. ej. entrada omitida).

<Info>
  `api.runtime.stt.transcribeAudioFile(...)` sigue existiendo como alias de compatibilidad
  para `api.runtime.mediaUnderstanding.transcribeAudioFile(...)`.
</Info>

### `api.runtime.imageGeneration`

Generación de imágenes.

```typescript
const result = await api.runtime.imageGeneration.generate({
  prompt: "A robot painting a sunset",
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
  args: { query: "OpenClaw plugin SDK", count: 5 },
});
```

### `api.runtime.media`

Utilidades de medios de bajo nivel.

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

Utilidades de nivel de sistema.

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

Auxiliares de runtime específicos del canal (disponibles cuando se carga un plugin de canal).

## Almacenar referencias de runtime

Usa `createPluginRuntimeStore` para almacenar la referencia de runtime y usarla fuera
del callback `register`:

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

## Otros campos `api` de nivel superior

Además de `api.runtime`, el objeto API también proporciona:

| Campo                    | Tipo                      | Descripción                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | ID del plugin                                                                                   |
| `api.name`               | `string`                  | Nombre para mostrar del plugin                                                                         |
| `api.config`             | `OpenClawConfig`          | Instantánea de configuración actual (instantánea activa en memoria del runtime cuando está disponible)                  |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuración específica del plugin de `plugins.entries.<id>.config`                                   |
| `api.logger`             | `PluginLogger`            | Registrador con alcance (`debug`, `info`, `warn`, `error`)                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modo de carga actual; `"setup-runtime"` es la ventana ligera de arranque/configuración previa a la entrada completa |
| `api.resolvePath(input)` | `(string) => string`      | Resuelve una ruta relativa a la raíz del plugin                                                  |

## Relacionado

- [Resumen del SDK](/plugins/sdk-overview) -- referencia de subrutas
- [Puntos de entrada del SDK](/plugins/sdk-entrypoints) -- opciones de `definePluginEntry`
- [Internos de plugins](/plugins/architecture) -- modelo de capacidades y registro
