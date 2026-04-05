---
read_when:
    - أنت تحتاج إلى استدعاء مساعدات core من plugin ‏(TTS، STT، توليد الصور، البحث على الويب، subagent)
    - أنت تريد فهم ما الذي يكشفه `api.runtime`
    - أنت تصل إلى مساعدات الإعدادات أو الوكيل أو الوسائط من شيفرة plugin
sidebarTitle: Runtime Helpers
summary: '`api.runtime` -- مساعدات وقت التشغيل المحقونة والمتاحة لكل plugin'
title: مساعدات وقت تشغيل plugin
x-i18n:
    generated_at: "2026-04-05T12:51:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 667edff734fd30f9b05d55eae6360830a45ae8f3012159f88a37b5e05404e666
    source_path: plugins/sdk-runtime.md
    workflow: 15
---

# مساعدات وقت تشغيل plugin

مرجع للكائن `api.runtime` المحقون في كل plugin أثناء
التسجيل. استخدم هذه المساعدات بدلًا من استيراد العناصر الداخلية للمضيف مباشرة.

<Tip>
  **تبحث عن شرح تطبيقي؟** راجع [Channel Plugins](/plugins/sdk-channel-plugins)
  أو [Provider Plugins](/plugins/sdk-provider-plugins) للحصول على أدلة خطوة بخطوة
  تعرض هذه المساعدات ضمن السياق.
</Tip>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

## مساحات أسماء وقت التشغيل

### `api.runtime.agent`

هوية الوكيل، والأدلة، وإدارة الجلسات.

```typescript
// Resolve the agent's working directory
const agentDir = api.runtime.agent.resolveAgentDir(cfg);

// Resolve agent workspace
const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg);

// Get agent identity
const identity = api.runtime.agent.resolveAgentIdentity(cfg);

// Get default thinking level
const thinking = api.runtime.agent.resolveThinkingDefault(cfg, provider, model);

// Get agent timeout
const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

// Ensure workspace exists
await api.runtime.agent.ensureAgentWorkspace(cfg);

// Run an embedded Pi agent
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

توجد **مساعدات مخزن الجلسات** تحت `api.runtime.agent.session`:

```typescript
const storePath = api.runtime.agent.session.resolveStorePath(cfg);
const store = api.runtime.agent.session.loadSessionStore(cfg);
await api.runtime.agent.session.saveSessionStore(cfg, store);
const filePath = api.runtime.agent.session.resolveSessionFilePath(cfg, sessionId);
```

### `api.runtime.agent.defaults`

ثوابت النموذج والمزوّد الافتراضيين:

```typescript
const model = api.runtime.agent.defaults.model; // e.g. "anthropic/claude-sonnet-4-6"
const provider = api.runtime.agent.defaults.provider; // e.g. "anthropic"
```

### `api.runtime.subagent`

تشغيل وإدارة تشغيلات subagent في الخلفية.

```typescript
// Start a subagent run
const { runId } = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai", // optional override
  model: "gpt-4.1-mini", // optional override
  deliver: false,
});

// Wait for completion
const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

// Read session messages
const { messages } = await api.runtime.subagent.getSessionMessages({
  sessionKey: "agent:main:subagent:search-helper",
  limit: 10,
});

// Delete a session
await api.runtime.subagent.deleteSession({
  sessionKey: "agent:main:subagent:search-helper",
});
```

<Warning>
  تتطلب تجاوزات النموذج (`provider`/`model`) اشتراكًا صريحًا من المشغّل عبر
  `plugins.entries.<id>.subagent.allowModelOverride: true` في الإعدادات.
  لا تزال Plugins غير الموثوقة قادرة على تشغيل subagents، لكن طلبات التجاوز تُرفض.
</Warning>

### `api.runtime.taskFlow`

اربط وقت تشغيل Task Flow بمفتاح جلسة OpenClaw موجود أو بسياق أداة موثوق،
ثم أنشئ Task Flows وأدرها من دون تمرير مالك في كل استدعاء.

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

استخدم `bindSession({ sessionKey, requesterOrigin })` عندما يكون لديك بالفعل
مفتاح جلسة OpenClaw موثوق من طبقة الربط الخاصة بك. ولا تربط من إدخال
مستخدم خام.

### `api.runtime.tts`

تركيب النص إلى كلام.

```typescript
// Standard TTS
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

// Telephony-optimized TTS
const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

// List available voices
const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

يستخدم إعدادات `messages.tts` الأساسية واختيار المزوّد. ويعيد مخزن PCM صوتيًا
+ معدل العينة.

### `api.runtime.mediaUnderstanding`

تحليل الصور والصوت والفيديو.

```typescript
// Describe an image
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

// Transcribe audio
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  mime: "audio/ogg", // optional, for when MIME cannot be inferred
});

// Describe a video
const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

// Generic file analysis
const result = await api.runtime.mediaUnderstanding.runFile({
  filePath: "/tmp/inbound-file.pdf",
  cfg: api.config,
});
```

يعيد `{ text: undefined }` عندما لا يتم إنتاج أي خرج (مثلًا عند تخطي الإدخال).

<Info>
  لا يزال `api.runtime.stt.transcribeAudioFile(...)` موجودًا كاسم بديل
  للتوافق مع `api.runtime.mediaUnderstanding.transcribeAudioFile(...)`.
</Info>

### `api.runtime.imageGeneration`

توليد الصور.

```typescript
const result = await api.runtime.imageGeneration.generate({
  prompt: "A robot painting a sunset",
  cfg: api.config,
});

const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
```

### `api.runtime.webSearch`

البحث على الويب.

```typescript
const providers = api.runtime.webSearch.listProviders({ config: api.config });

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: { query: "OpenClaw plugin SDK", count: 5 },
});
```

### `api.runtime.media`

أدوات وسائط منخفضة المستوى.

```typescript
const webMedia = await api.runtime.media.loadWebMedia(url);
const mime = await api.runtime.media.detectMime(buffer);
const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
const metadata = await api.runtime.media.getImageMetadata(filePath);
const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
```

### `api.runtime.config`

تحميل الإعدادات وكتابتها.

```typescript
const cfg = await api.runtime.config.loadConfig();
await api.runtime.config.writeConfigFile(cfg);
```

### `api.runtime.system`

أدوات على مستوى النظام.

```typescript
await api.runtime.system.enqueueSystemEvent(event);
api.runtime.system.requestHeartbeatNow();
const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
const hint = api.runtime.system.formatNativeDependencyHint(pkg);
```

### `api.runtime.events`

اشتراكات الأحداث.

```typescript
api.runtime.events.onAgentEvent((event) => {
  /* ... */
});
api.runtime.events.onSessionTranscriptUpdate((update) => {
  /* ... */
});
```

### `api.runtime.logging`

التسجيل.

```typescript
const verbose = api.runtime.logging.shouldLogVerbose();
const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
```

### `api.runtime.modelAuth`

حل مصادقة النموذج والمزوّد.

```typescript
const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });
const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
  provider: "openai",
  cfg,
});
```

### `api.runtime.state`

حل دليل الحالة.

```typescript
const stateDir = api.runtime.state.resolveStateDir();
```

### `api.runtime.tools`

مصانع أداة الذاكرة وCLI.

```typescript
const getTool = api.runtime.tools.createMemoryGetTool(/* ... */);
const searchTool = api.runtime.tools.createMemorySearchTool(/* ... */);
api.runtime.tools.registerMemoryCli(/* ... */);
```

### `api.runtime.channel`

مساعدات وقت تشغيل خاصة بالقناة (تتوفر عند تحميل plugin قناة).

## تخزين مراجع وقت التشغيل

استخدم `createPluginRuntimeStore` لتخزين مرجع وقت التشغيل من أجل استخدامه خارج
استدعاء `register`:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>("my-plugin runtime not initialized");

// In your entry point
export default defineChannelPluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Example",
  plugin: myPlugin,
  setRuntime: store.setRuntime,
});

// In other files
export function getRuntime() {
  return store.getRuntime(); // throws if not initialized
}

export function tryGetRuntime() {
  return store.tryGetRuntime(); // returns null if not initialized
}
```

## حقول `api` الأخرى في المستوى الأعلى

إلى جانب `api.runtime`، يوفر كائن API أيضًا:

| الحقل                    | النوع                     | الوصف                                                                                       |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | معرّف plugin                                                                                |
| `api.name`               | `string`                  | اسم عرض plugin                                                                              |
| `api.config`             | `OpenClawConfig`          | لقطة الإعدادات الحالية (اللقطة النشطة داخل الذاكرة لوقت التشغيل عند توفرها)               |
| `api.pluginConfig`       | `Record<string, unknown>` | إعدادات plugin الخاصة من `plugins.entries.<id>.config`                                     |
| `api.logger`             | `PluginLogger`            | مسجل ضمن النطاق (`debug` و`info` و`warn` و`error`)                                         |
| `api.registrationMode`   | `PluginRegistrationMode`  | وضع التحميل الحالي؛ وتمثل `"setup-runtime"` نافذة بدء تشغيل/إعداد خفيفة قبل نقطة الدخول الكاملة |
| `api.resolvePath(input)` | `(string) => string`      | حل مسار نسبةً إلى جذر plugin                                                                |

## ذو صلة

- [نظرة عامة على SDK](/plugins/sdk-overview) -- مرجع المسارات الفرعية
- [نقاط دخول SDK](/plugins/sdk-entrypoints) -- خيارات `definePluginEntry`
- [الآليات الداخلية لـ plugin](/plugins/architecture) -- نموذج القدرات والسجل
