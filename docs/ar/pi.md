---
read_when:
    - فهم تصميم تكامل Pi SDK في OpenClaw
    - تعديل دورة حياة جلسة الوكيل، أو الأدوات، أو ربط الموفّرين لـ Pi
summary: بنية تكامل Pi agent المضمن في OpenClaw ودورة حياة الجلسة
title: بنية تكامل Pi
x-i18n:
    generated_at: "2026-04-05T12:50:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 596de5fbb1430008698079f211db200e02ca8485547550fd81571a459c4c83c7
    source_path: pi.md
    workflow: 15
---

# بنية تكامل Pi

تصف هذه الوثيقة كيف يندمج OpenClaw مع [pi-coding-agent](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent) والحزم الشقيقة له (`pi-ai` و`pi-agent-core` و`pi-tui`) لتشغيل قدرات وكيل الذكاء الاصطناعي لديه.

## نظرة عامة

يستخدم OpenClaw حزمة pi SDK لتضمين وكيل ترميز بالذكاء الاصطناعي داخل بنية بوابة المراسلة الخاصة به. وبدلًا من تشغيل pi كعملية فرعية أو استخدام وضع RPC، يستورد OpenClaw مباشرة ويُنشئ `AgentSession` الخاص بـ pi عبر `createAgentSession()`. ويوفر هذا النهج المضمن:

- تحكمًا كاملًا في دورة حياة الجلسة ومعالجة الأحداث
- حقن أدوات مخصصة (المراسلة، وsandbox، وإجراءات خاصة بكل قناة)
- تخصيص system prompt لكل قناة/سياق
- استمرارية الجلسة مع دعم التفرع/الضغط
- تدوير ملفات تعريف auth متعددة الحسابات مع الرجوع عند الفشل
- تبديل النماذج بشكل مستقل عن الموفّر

## تبعيات الحزمة

```json
{
  "@mariozechner/pi-agent-core": "0.64.0",
  "@mariozechner/pi-ai": "0.64.0",
  "@mariozechner/pi-coding-agent": "0.64.0",
  "@mariozechner/pi-tui": "0.64.0"
}
```

| الحزمة | الغرض |
| ------ | ----- |
| `pi-ai` | تجريدات LLM الأساسية: `Model` و`streamSimple` وأنواع الرسائل وواجهات API الخاصة بالموفّرين |
| `pi-agent-core` | حلقة الوكيل، وتنفيذ الأدوات، وأنواع `AgentMessage` |
| `pi-coding-agent` | SDK عالي المستوى: `createAgentSession` و`SessionManager` و`AuthStorage` و`ModelRegistry` والأدوات المضمنة |
| `pi-tui` | مكونات واجهة الطرفية (تُستخدم في وضع TUI المحلي في OpenClaw) |

## بنية الملفات

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

توجد بيئات تشغيل إجراءات الرسائل الخاصة بكل قناة الآن في أدلة
الإضافات المملوكة لها بدلًا من وجودها تحت `src/agents/tools`، على سبيل المثال:

- ملفات وقت تشغيل إجراءات إضافة Discord
- ملف وقت تشغيل إجراءات إضافة Slack
- ملف وقت تشغيل إجراءات إضافة Telegram
- ملف وقت تشغيل إجراءات إضافة WhatsApp

## مسار التكامل الأساسي

### 1. تشغيل وكيل مضمن

نقطة الدخول الرئيسية هي `runEmbeddedPiAgent()` في `pi-embedded-runner/run.ts`:

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

### 2. إنشاء الجلسة

داخل `runEmbeddedAttempt()` ‏(التي يستدعيها `runEmbeddedPiAgent()`)، تُستخدم حزمة pi SDK:

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

### 3. الاشتراك في الأحداث

تقوم `subscribeEmbeddedPiSession()` بالاشتراك في أحداث `AgentSession` الخاصة بـ pi:

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

تشمل الأحداث التي تتم معالجتها:

- `message_start` / `message_end` / `message_update` ‏(بث النص/التفكير)
- `tool_execution_start` / `tool_execution_update` / `tool_execution_end`
- `turn_start` / `turn_end`
- `agent_start` / `agent_end`
- `auto_compaction_start` / `auto_compaction_end`

### 4. إرسال prompt

بعد الإعداد، يتم إرسال prompt إلى الجلسة:

```typescript
await session.prompt(effectivePrompt, { images: imageResult.images });
```

تتعامل SDK مع حلقة الوكيل الكاملة: الإرسال إلى LLM، وتنفيذ استدعاءات الأدوات، وبث الاستجابات.

يكون حقن الصور محليًا للـ prompt: إذ يحمّل OpenClaw مراجع الصور من prompt الحالي
ويمررها عبر `images` لذلك الدور فقط. وهو لا يعيد فحص أدوار السجل الأقدم
لإعادة حقن حمولات الصور.

## بنية الأدوات

### مسار الأدوات

1. **الأدوات الأساسية**: ‏`codingTools` الخاصة بـ pi ‏(read، وbash، وedit، وwrite)
2. **بدائل مخصصة**: يستبدل OpenClaw أداة bash بـ `exec`/`process`، ويخصص read/edit/write لـ sandbox
3. **أدوات OpenClaw**: المراسلة، والمتصفح، وcanvas، والجلسات، وcron، وgateway، وغير ذلك
4. **أدوات القنوات**: أدوات إجراءات خاصة بـ Discord/Telegram/Slack/WhatsApp
5. **ترشيح السياسة**: تُرشَّح الأدوات حسب الملف التعريفي، والموفّر، والوكيل، والمجموعة، وسياسات sandbox
6. **تطبيع المخطط**: تنظيف المخططات لمعالجة حالات Gemini/OpenAI الخاصة
7. **تغليف AbortSignal**: تغليف الأدوات لاحترام إشارات الإيقاف

### مكيّف تعريف الأداة

لدى `AgentTool` في pi-agent-core توقيع `execute` مختلف عن `ToolDefinition` في pi-coding-agent. ويقوم المكيّف في `pi-tool-definition-adapter.ts` بربط ذلك:

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

### استراتيجية تقسيم الأدوات

تقوم `splitSdkTools()` بتمرير جميع الأدوات عبر `customTools`:

```typescript
export function splitSdkTools(options: { tools: AnyAgentTool[]; sandboxEnabled: boolean }) {
  return {
    builtInTools: [], // Empty. We override everything
    customTools: toToolDefinitions(options.tools),
  };
}
```

يضمن هذا أن يبقى ترشيح السياسة في OpenClaw، وتكامل sandbox، ومجموعة الأدوات الموسعة متسقة عبر الموفّرين.

## بناء system prompt

يتم بناء system prompt في `buildAgentSystemPrompt()` ‏(`system-prompt.ts`). وهو يجمع prompt كاملة مع أقسام تشمل Tooling، وTool Call Style، وحواجز الأمان، ومرجع OpenClaw CLI، وSkills، والوثائق، ومساحة العمل، وSandbox، والمراسلة، وReply Tags، والصوت، والردود الصامتة، وHeartbeats، وبيانات وقت التشغيل، بالإضافة إلى الذاكرة والتفاعلات عند تمكينها، وكذلك ملفات السياق الاختيارية ومحتوى system prompt الإضافي. وتُقص الأقسام في وضع prompt الأدنى المستخدم بواسطة الوكلاء الفرعيين.

تُطبَّق prompt على الجلسة بعد إنشائها عبر `applySystemPromptOverrideToSession()`:

```typescript
const systemPromptOverride = createSystemPromptOverride(appendPrompt);
applySystemPromptOverrideToSession(session, systemPromptOverride);
```

## إدارة الجلسات

### ملفات الجلسات

الجلسات عبارة عن ملفات JSONL ذات بنية شجرية (ربط عبر id/parentId). ويتولى `SessionManager` في Pi الاستمرارية:

```typescript
const sessionManager = SessionManager.open(params.sessionFile);
```

ويلف OpenClaw ذلك باستخدام `guardSessionManager()` من أجل أمان نتائج الأدوات.

### التخزين المؤقت للجلسات

يقوم `session-manager-cache.ts` بتخزين مثيلات SessionManager مؤقتًا لتجنب تكرار تحليل الملفات:

```typescript
await prewarmSessionFile(params.sessionFile);
sessionManager = SessionManager.open(params.sessionFile);
trackSessionManagerAccess(params.sessionFile);
```

### تقييد السجل

تقوم `limitHistoryTurns()` بقص سجل المحادثة وفقًا لنوع القناة (رسائل خاصة مقابل مجموعة).

### الضغط

يُفعَّل الضغط التلقائي عند تجاوز السياق. وتشمل تواقيع تجاوز السياق الشائعة
`request_too_large` و`context length exceeded` و`input exceeds the
maximum number of tokens` و`input token count exceeds the maximum number of
input tokens` و`input is too long for the model` و`ollama error: context
length exceeded`. وتتولى `compactEmbeddedPiSessionDirect()` معالجة
الضغط اليدوي:

```typescript
const compactResult = await compactEmbeddedPiSessionDirect({
  sessionId, sessionFile, provider, model, ...
});
```

## المصادقة وحل النموذج

### ملفات تعريف auth

يحافظ OpenClaw على مخزن لملفات تعريف auth مع عدة مفاتيح API لكل موفّر:

```typescript
const authStore = ensureAuthProfileStore(agentDir, { allowKeychainPrompt: false });
const profileOrder = resolveAuthProfileOrder({ cfg, store: authStore, provider, preferredProfile });
```

تُدوَّر الملفات التعريفية عند الفشل مع تتبع فترات التهدئة:

```typescript
await markAuthProfileFailure({ store, profileId, reason, cfg, agentDir });
const rotated = await advanceAuthProfile();
```

### حل النموذج

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

### الرجوع عند الفشل

تؤدي `FailoverError` إلى الرجوع إلى نموذج بديل عند الإعداد:

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

## إضافات Pi

يحمّل OpenClaw إضافات pi مخصصة لسلوكيات متخصصة:

### حماية الضغط

تضيف `src/agents/pi-hooks/compaction-safeguard.ts` وسائل حماية إلى الضغط، بما في ذلك ضبط ميزانية الرموز بشكل تكيفي بالإضافة إلى ملخصات فشل الأدوات وعمليات الملفات:

```typescript
if (resolveCompactionMode(params.cfg) === "safeguard") {
  setCompactionSafeguardRuntime(params.sessionManager, { maxHistoryShare });
  paths.push(resolvePiExtensionPath("compaction-safeguard"));
}
```

### تقليم السياق

تطبّق `src/agents/pi-hooks/context-pruning.ts` تقليم السياق المستند إلى Cache-TTL:

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

## البث وردود الكتل

### تقسيم الكتل

يدير `EmbeddedBlockChunker` بث النص إلى كتل رد منفصلة:

```typescript
const blockChunker = blockChunking ? new EmbeddedBlockChunker(blockChunking) : null;
```

### إزالة وسوم Thinking/Final

تُعالج مخرجات البث لإزالة كتل `<think>`/`<thinking>` واستخراج محتوى `<final>`:

```typescript
const stripBlockTags = (text: string, state: { thinking: boolean; final: boolean }) => {
  // Strip <think>...</think> content
  // If enforceFinalTag, only return <final>...</final> content
};
```

### توجيهات الرد

يتم تحليل توجيهات الرد مثل `[[media:url]]` و`[[voice]]` و`[[reply:id]]` واستخراجها:

```typescript
const { text: cleanedText, mediaUrls, audioAsVoice, replyToId } = consumeReplyDirectives(chunk);
```

## معالجة الأخطاء

### تصنيف الأخطاء

تقوم `pi-embedded-helpers.ts` بتصنيف الأخطاء من أجل التعامل المناسب معها:

```typescript
isContextOverflowError(errorText)     // Context too large
isCompactionFailureError(errorText)   // Compaction failed
isAuthAssistantError(lastAssistant)   // Auth failure
isRateLimitAssistantError(...)        // Rate limited
isFailoverAssistantError(...)         // Should failover
classifyFailoverReason(errorText)     // "auth" | "rate_limit" | "quota" | "timeout" | ...
```

### الرجوع في مستوى التفكير

إذا كان مستوى التفكير غير مدعوم، فيتم الرجوع:

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

## تكامل Sandbox

عند تمكين وضع sandbox، يتم تقييد الأدوات والمسارات:

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

## المعالجة الخاصة بكل موفّر

### Anthropic

- تنظيف سلسلة الرفض السحرية
- التحقق من الأدوار المتتالية
- توافق معلمات Claude Code

### Google/Gemini

- تنقية مخطط الأدوات المملوك للإضافات

### OpenAI

- أداة `apply_patch` لنماذج Codex
- معالجة خفض مستوى التفكير

## تكامل TUI

لدى OpenClaw أيضًا وضع TUI محلي يستخدم مكونات pi-tui مباشرة:

```typescript
// src/tui/tui.ts
import { ... } from "@mariozechner/pi-tui";
```

يوفر هذا تجربة طرفية تفاعلية مشابهة للوضع الأصلي في pi.

## الفروق الأساسية عن Pi CLI

| الجانب | Pi CLI | OpenClaw Embedded |
| ------ | ------ | ----------------- |
| الاستدعاء | أمر `pi` / ‏RPC | SDK عبر `createAgentSession()` |
| الأدوات | أدوات الترميز الافتراضية | مجموعة أدوات OpenClaw المخصصة |
| system prompt | ‏AGENTS.md + prompts | ديناميكية بحسب القناة/السياق |
| تخزين الجلسات | `~/.pi/agent/sessions/` | ‏`~/.openclaw/agents/<agentId>/sessions/` ‏(أو `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`) |
| المصادقة | بيانات اعتماد واحدة | ملفات تعريف متعددة مع تدوير |
| الإضافات | تُحمَّل من القرص | مسارات برمجية + مسارات على القرص |
| معالجة الأحداث | عرض TUI | قائمة على callbacks ‏(`onBlockReply`، إلخ) |

## اعتبارات مستقبلية

مجالات قد تحتاج إلى إعادة عمل:

1. **محاذاة توقيعات الأدوات**: يوجد حاليًا تكييف بين توقيعات pi-agent-core وpi-coding-agent
2. **تغليف session manager**: يضيف `guardSessionManager` أمانًا لكنه يزيد التعقيد
3. **تحميل الإضافات**: يمكن استخدام `ResourceLoader` الخاص بـ pi بشكل أكثر مباشرة
4. **تعقيد معالج البث**: أصبح `subscribeEmbeddedPiSession` كبيرًا
5. **الحالات الخاصة بالموفّرين**: توجد مسارات شيفرة كثيرة خاصة بكل موفّر قد تتمكن pi من معالجتها مستقبلًا

## الاختبارات

تغطي مجموعات الاختبارات التالية تكامل Pi:

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

اختبارات مباشرة/اختيارية:

- `src/agents/pi-embedded-runner-extraparams.live.test.ts` ‏(فعّل `OPENCLAW_LIVE_TEST=1`)

للاطلاع على أوامر التشغيل الحالية، راجع [سير عمل تطوير Pi](/pi-dev).
