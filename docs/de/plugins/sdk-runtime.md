---
read_when:
    - Sie müssen Core-Hilfen aus einem Plugin aufrufen (TTS, STT, Bildgenerierung, Websuche, Subagent)
    - Sie möchten verstehen, was api.runtime verfügbar macht
    - Sie greifen aus Plugin-Code auf Konfigurations-, Agent- oder Medienhilfen zu
sidebarTitle: Runtime Helpers
summary: api.runtime -- die injizierten Laufzeithilfen, die Plugins zur Verfügung stehen
title: Plugin-Laufzeithilfen
x-i18n:
    generated_at: "2026-04-08T02:17:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: acb9e56678e9ed08d0998dfafd7cd1982b592be5bc34d9e2d2c1f70274f8f248
    source_path: plugins/sdk-runtime.md
    workflow: 15
---

# Plugin-Laufzeithilfen

Referenz für das `api.runtime`-Objekt, das während der Registrierung in jedes Plugin
injiziert wird. Verwenden Sie diese Hilfen, anstatt Host-Interna direkt zu importieren.

<Tip>
  **Suchen Sie nach einer Schritt-für-Schritt-Anleitung?** Siehe [Channel Plugins](/de/plugins/sdk-channel-plugins)
  oder [Provider Plugins](/de/plugins/sdk-provider-plugins) für Schritt-für-Schritt-Anleitungen,
  die diese Hilfen im Kontext zeigen.
</Tip>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

## Laufzeit-Namespaces

### `api.runtime.agent`

Agent-Identität, Verzeichnisse und Sitzungsverwaltung.

```typescript
// Arbeitsverzeichnis des Agents auflösen
const agentDir = api.runtime.agent.resolveAgentDir(cfg);

// Agent-Workspace auflösen
const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg);

// Agent-Identität abrufen
const identity = api.runtime.agent.resolveAgentIdentity(cfg);

// Standardmäßige Thinking-Stufe abrufen
const thinking = api.runtime.agent.resolveThinkingDefault(cfg, provider, model);

// Agent-Timeout abrufen
const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

// Sicherstellen, dass der Workspace existiert
await api.runtime.agent.ensureAgentWorkspace(cfg);

// Einen eingebetteten Pi-Agent ausführen
const agentDir = api.runtime.agent.resolveAgentDir(cfg);
const result = await api.runtime.agent.runEmbeddedPiAgent({
  sessionId: "my-plugin:task-1",
  runId: crypto.randomUUID(),
  sessionFile: path.join(agentDir, "sessions", "my-plugin-task-1.jsonl"),
  workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg),
  prompt: "Fassen Sie die neuesten Änderungen zusammen",
  timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
});
```

**Hilfen für den Sitzungsspeicher** befinden sich unter `api.runtime.agent.session`:

```typescript
const storePath = api.runtime.agent.session.resolveStorePath(cfg);
const store = api.runtime.agent.session.loadSessionStore(cfg);
await api.runtime.agent.session.saveSessionStore(cfg, store);
const filePath = api.runtime.agent.session.resolveSessionFilePath(cfg, sessionId);
```

### `api.runtime.agent.defaults`

Standardkonstanten für Modell und Anbieter:

```typescript
const model = api.runtime.agent.defaults.model; // z. B. "anthropic/claude-sonnet-4-6"
const provider = api.runtime.agent.defaults.provider; // z. B. "anthropic"
```

### `api.runtime.subagent`

Subagent-Ausführungen im Hintergrund starten und verwalten.

```typescript
// Eine Subagent-Ausführung starten
const { runId } = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Erweitern Sie diese Abfrage zu fokussierten Folge-Suchanfragen.",
  provider: "openai", // optionale Überschreibung
  model: "gpt-4.1-mini", // optionale Überschreibung
  deliver: false,
});

// Auf Abschluss warten
const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

// Sitzungsnachrichten lesen
const { messages } = await api.runtime.subagent.getSessionMessages({
  sessionKey: "agent:main:subagent:search-helper",
  limit: 10,
});

// Eine Sitzung löschen
await api.runtime.subagent.deleteSession({
  sessionKey: "agent:main:subagent:search-helper",
});
```

<Warning>
  Modellüberschreibungen (`provider`/`model`) erfordern eine Opt-in-Freigabe
  des Operators über `plugins.entries.<id>.subagent.allowModelOverride: true` in der Konfiguration.
  Nicht vertrauenswürdige Plugins können weiterhin Subagents ausführen, aber Überschreibungsanfragen werden abgelehnt.
</Warning>

### `api.runtime.taskFlow`

Eine Task-Flow-Laufzeit an einen vorhandenen OpenClaw-Sitzungsschlüssel oder einen vertrauenswürdigen Tool-
Kontext binden und dann Task Flows erstellen und verwalten, ohne bei jedem Aufruf einen Eigentümer zu übergeben.

```typescript
const taskFlow = api.runtime.taskFlow.fromToolContext(ctx);

const created = taskFlow.createManaged({
  controllerId: "my-plugin/review-batch",
  goal: "Neue Pull Requests prüfen",
});

const child = taskFlow.runTask({
  flowId: created.flowId,
  runtime: "acp",
  childSessionKey: "agent:main:subagent:reviewer",
  task: "PR #123 prüfen",
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

Verwenden Sie `bindSession({ sessionKey, requesterOrigin })`, wenn Sie bereits einen
vertrauenswürdigen OpenClaw-Sitzungsschlüssel aus Ihrer eigenen Bindungsschicht haben. Binden Sie nicht anhand roher
Benutzereingaben.

### `api.runtime.tts`

Text-zu-Sprache-Synthese.

```typescript
// Standard-TTS
const clip = await api.runtime.tts.textToSpeech({
  text: "Hallo von OpenClaw",
  cfg: api.config,
});

// Für Telefonie optimiertes TTS
const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
  text: "Hallo von OpenClaw",
  cfg: api.config,
});

// Verfügbare Stimmen auflisten
const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Verwendet die Core-Konfiguration `messages.tts` und die Anbieterauswahl. Gibt einen PCM-Audio-
Puffer plus Abtastrate zurück.

### `api.runtime.mediaUnderstanding`

Bild-, Audio- und Videoanalyse.

```typescript
// Ein Bild beschreiben
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

// Audio transkribieren
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  mime: "audio/ogg", // optional, wenn sich der MIME-Typ nicht ableiten lässt
});

// Ein Video beschreiben
const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

// Generische Dateianalyse
const result = await api.runtime.mediaUnderstanding.runFile({
  filePath: "/tmp/inbound-file.pdf",
  cfg: api.config,
});
```

Gibt `{ text: undefined }` zurück, wenn keine Ausgabe erzeugt wird (z. B. übersprungene Eingabe).

<Info>
  `api.runtime.stt.transcribeAudioFile(...)` bleibt als Kompatibilitätsalias
  für `api.runtime.mediaUnderstanding.transcribeAudioFile(...)` bestehen.
</Info>

### `api.runtime.imageGeneration`

Bildgenerierung.

```typescript
const result = await api.runtime.imageGeneration.generate({
  prompt: "Ein Roboter, der einen Sonnenuntergang malt",
  cfg: api.config,
});

const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
```

### `api.runtime.webSearch`

Websuche.

```typescript
const providers = api.runtime.webSearch.listProviders({ config: api.config });

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: { query: "OpenClaw plugin SDK", count: 5 },
});
```

### `api.runtime.media`

Low-Level-Medienhilfen.

```typescript
const webMedia = await api.runtime.media.loadWebMedia(url);
const mime = await api.runtime.media.detectMime(buffer);
const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
const metadata = await api.runtime.media.getImageMetadata(filePath);
const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
```

### `api.runtime.config`

Konfiguration laden und schreiben.

```typescript
const cfg = await api.runtime.config.loadConfig();
await api.runtime.config.writeConfigFile(cfg);
```

### `api.runtime.system`

Hilfsfunktionen auf Systemebene.

```typescript
await api.runtime.system.enqueueSystemEvent(event);
api.runtime.system.requestHeartbeatNow();
const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
const hint = api.runtime.system.formatNativeDependencyHint(pkg);
```

### `api.runtime.events`

Ereignisabonnements.

```typescript
api.runtime.events.onAgentEvent((event) => {
  /* ... */
});
api.runtime.events.onSessionTranscriptUpdate((update) => {
  /* ... */
});
```

### `api.runtime.logging`

Protokollierung.

```typescript
const verbose = api.runtime.logging.shouldLogVerbose();
const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
```

### `api.runtime.modelAuth`

Auflösung von Modell- und Anbieter-Authentifizierung.

```typescript
const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });
const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
  provider: "openai",
  cfg,
});
```

### `api.runtime.state`

Auflösung des Zustandsverzeichnisses.

```typescript
const stateDir = api.runtime.state.resolveStateDir();
```

### `api.runtime.tools`

Memory-Tool-Fabriken und CLI.

```typescript
const getTool = api.runtime.tools.createMemoryGetTool(/* ... */);
const searchTool = api.runtime.tools.createMemorySearchTool(/* ... */);
api.runtime.tools.registerMemoryCli(/* ... */);
```

### `api.runtime.channel`

Kanalspezifische Laufzeithilfen (verfügbar, wenn ein Kanal-Plugin geladen ist).

`api.runtime.channel.mentions` ist die gemeinsame Oberfläche für Richtlinien zu eingehenden Erwähnungen für
gebündelte Kanal-Plugins, die Laufzeitinjektion verwenden:

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

Verfügbare Erwähnungshilfen:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

`api.runtime.channel.mentions` stellt absichtlich nicht die älteren
Kompatibilitätshilfen `resolveMentionGating*` bereit. Bevorzugen Sie den normalisierten
Pfad `{ facts, policy }`.

## Laufzeitreferenzen speichern

Verwenden Sie `createPluginRuntimeStore`, um die Laufzeitreferenz zur Verwendung außerhalb
des Callbacks `register` zu speichern:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>("my-plugin runtime not initialized");

// In Ihrem Einstiegspunkt
export default defineChannelPluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Example",
  plugin: myPlugin,
  setRuntime: store.setRuntime,
});

// In anderen Dateien
export function getRuntime() {
  return store.getRuntime(); // löst eine Ausnahme aus, wenn nicht initialisiert
}

export function tryGetRuntime() {
  return store.tryGetRuntime(); // gibt null zurück, wenn nicht initialisiert
}
```

## Andere `api`-Felder auf oberster Ebene

Neben `api.runtime` stellt das API-Objekt außerdem Folgendes bereit:

| Feld                    | Typ                       | Beschreibung                                                                              |
| ----------------------- | ------------------------- | ----------------------------------------------------------------------------------------- |
| `api.id`                | `string`                  | Plugin-ID                                                                                 |
| `api.name`              | `string`                  | Anzeigename des Plugins                                                                   |
| `api.config`            | `OpenClawConfig`          | Aktueller Konfigurations-Snapshot (aktiver In-Memory-Laufzeit-Snapshot, falls verfügbar) |
| `api.pluginConfig`      | `Record<string, unknown>` | Plugin-spezifische Konfiguration aus `plugins.entries.<id>.config`                        |
| `api.logger`            | `PluginLogger`            | Bereichsbezogener Logger (`debug`, `info`, `warn`, `error`)                               |
| `api.registrationMode`  | `PluginRegistrationMode`  | Aktueller Lademodus; `"setup-runtime"` ist das schlanke Start-/Einrichtungsfenster vor dem vollständigen Entry |
| `api.resolvePath(input)`| `(string) => string`      | Einen Pfad relativ zur Plugin-Wurzel auflösen                                             |

## Verwandt

- [SDK Overview](/de/plugins/sdk-overview) -- Referenz für Subpaths
- [SDK Entry Points](/de/plugins/sdk-entrypoints) -- Optionen für `definePluginEntry`
- [Plugin Internals](/de/plugins/architecture) -- Fähigkeitsmodell und Registry
