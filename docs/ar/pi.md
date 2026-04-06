---
read_when:
    - فهم تصميم تكامل Pi SDK في OpenClaw
    - تعديل دورة حياة جلسة الوكيل أو الأدوات أو توصيل المزود لـ Pi
summary: بنية تكامل وكيل Pi المضمن في OpenClaw ودورة حياة الجلسة
title: بنية تكامل Pi
x-i18n:
    generated_at: "2026-04-06T03:09:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28594290b018b7cc2963d33dbb7cec6a0bd817ac486dafad59dd2ccabd482582
    source_path: pi.md
    workflow: 15
---

# بنية تكامل Pi

يصف هذا المستند كيفية تكامل OpenClaw مع [pi-coding-agent](https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent) والحزم الشقيقة له (`pi-ai` و`pi-agent-core` و`pi-tui`) لتشغيل قدرات وكيل الذكاء الاصطناعي الخاصة به.

## نظرة عامة

يستخدم OpenClaw حزمة Pi SDK لتضمين وكيل برمجة بالذكاء الاصطناعي داخل بنية بوابة المراسلة الخاصة به. وبدلًا من تشغيل pi كعملية فرعية أو استخدام وضع RPC، يستورد OpenClaw مباشرة `AgentSession` من pi وينشئه عبر `createAgentSession()`. يوفّر هذا النهج المضمن ما يلي:

- تحكم كامل في دورة حياة الجلسة ومعالجة الأحداث
- حقن أدوات مخصصة (المراسلة، وsandbox، والإجراءات الخاصة بالقنوات)
- تخصيص موجّه النظام لكل قناة/سياق
- استمرارية الجلسات مع دعم التفريع/الضغط
- تدوير ملفات تعريف المصادقة متعددة الحسابات مع آلية تجاوز الفشل
- تبديل النماذج بطريقة غير مرتبطة بمزود معين

## تبعيات الحزم

```json
{
  "@mariozechner/pi-agent-core": "0.64.0",
  "@mariozechner/pi-ai": "0.64.0",
  "@mariozechner/pi-coding-agent": "0.64.0",
  "@mariozechner/pi-tui": "0.64.0"
}
```

| الحزمة | الغرض |
| ----------------- | ------------------------------------------------------------------------------------------------------ |
| `pi-ai` | تجريدات LLM الأساسية: `Model` و`streamSimple` وأنواع الرسائل وواجهات برمجة المزود |
| `pi-agent-core` | حلقة الوكيل، وتنفيذ الأدوات، وأنواع `AgentMessage` |
| `pi-coding-agent` | SDK عالي المستوى: `createAgentSession` و`SessionManager` و`AuthStorage` و`ModelRegistry` والأدوات المدمجة |
| `pi-tui` | مكونات واجهة الطرفية (تُستخدم في وضع TUI المحلي في OpenClaw) |

## بنية الملفات

```
src/agents/
├── pi-embedded-runner.ts          # إعادة تصدير من pi-embedded-runner/
├── pi-embedded-runner/
│   ├── run.ts                     # نقطة الدخول الرئيسية: runEmbeddedPiAgent()
│   ├── run/
│   │   ├── attempt.ts             # منطق المحاولة الواحدة مع إعداد الجلسة
│   │   ├── params.ts              # النوع RunEmbeddedPiAgentParams
│   │   ├── payloads.ts            # بناء حمولات الاستجابة من نتائج التشغيل
│   │   ├── images.ts              # حقن صور نموذج الرؤية
│   │   └── types.ts               # EmbeddedRunAttemptResult
│   ├── abort.ts                   # اكتشاف خطأ الإلغاء
│   ├── cache-ttl.ts               # تتبع Cache TTL لتقليم السياق
│   ├── compact.ts                 # منطق الضغط اليدوي/التلقائي
│   ├── extensions.ts              # تحميل امتدادات pi للتشغيلات المضمنة
│   ├── extra-params.ts            # معلمات البث الخاصة بالمزود
│   ├── google.ts                  # إصلاحات ترتيب الأدوار لـ Google/Gemini
│   ├── history.ts                 # تحديد السجل (DM مقابل المجموعة)
│   ├── lanes.ts                   # مسارات أوامر الجلسة/العالمية
│   ├── logger.ts                  # مسجل النظام الفرعي
│   ├── model.ts                   # حل النموذج عبر ModelRegistry
│   ├── runs.ts                    # تتبع التشغيلات النشطة، الإلغاء، قائمة الانتظار
│   ├── sandbox-info.ts            # معلومات sandbox لموجّه النظام
│   ├── session-manager-cache.ts   # تخزين مؤقت لنسخ SessionManager
│   ├── session-manager-init.ts    # تهيئة ملف الجلسة
│   ├── system-prompt.ts           # منشئ موجّه النظام
│   ├── tool-split.ts              # تقسيم الأدوات إلى builtIn وcustom
│   ├── types.ts                   # EmbeddedPiAgentMeta وEmbeddedPiRunResult
│   └── utils.ts                   # ربط ThinkLevel ووصف الخطأ
├── pi-embedded-subscribe.ts       # الاشتراك في أحداث الجلسة/إرسالها
├── pi-embedded-subscribe.types.ts # SubscribeEmbeddedPiSessionParams
├── pi-embedded-subscribe.handlers.ts # مصنع معالجات الأحداث
├── pi-embedded-subscribe.handlers.lifecycle.ts
├── pi-embedded-subscribe.handlers.types.ts
├── pi-embedded-block-chunker.ts   # تقسيم الردود المتدفقة إلى كتل
├── pi-embedded-messaging.ts       # تتبع الإرسال عبر أداة المراسلة
├── pi-embedded-helpers.ts         # تصنيف الأخطاء والتحقق من الدور
├── pi-embedded-helpers/           # وحدات المساعدة
├── pi-embedded-utils.ts           # أدوات التنسيق
├── pi-tools.ts                    # createOpenClawCodingTools()
├── pi-tools.abort.ts              # تغليف AbortSignal للأدوات
├── pi-tools.policy.ts             # سياسة قائمة السماح/المنع للأدوات
├── pi-tools.read.ts               # تخصيصات أداة القراءة
├── pi-tools.schema.ts             # تطبيع مخطط الأدوات
├── pi-tools.types.ts              # الاسم المستعار للنوع AnyAgentTool
├── pi-tool-definition-adapter.ts  # المهايئ AgentTool -> ToolDefinition
├── pi-settings.ts                 # تجاوزات الإعدادات
├── pi-hooks/                      # خطافات pi مخصصة
│   ├── compaction-safeguard.ts    # امتداد الحماية
│   ├── compaction-safeguard-runtime.ts
│   ├── context-pruning.ts         # امتداد تقليم السياق المعتمد على Cache-TTL
│   └── context-pruning/
├── model-auth.ts                  # حل ملف تعريف المصادقة
├── auth-profiles.ts               # مخزن الملفات، التهدئة، تجاوز الفشل
├── model-selection.ts             # حل النموذج الافتراضي
├── models-config.ts               # إنشاء models.json
├── model-catalog.ts               # تخزين مؤقت لفهرس النماذج
├── context-window-guard.ts        # التحقق من نافذة السياق
├── failover-error.ts              # الصنف FailoverError
├── defaults.ts                    # DEFAULT_PROVIDER وDEFAULT_MODEL
├── system-prompt.ts               # buildAgentSystemPrompt()
├── system-prompt-params.ts        # حل معلمات موجّه النظام
├── system-prompt-report.ts        # إنشاء تقرير التصحيح
├── tool-summaries.ts              # ملخصات وصف الأدوات
├── tool-policy.ts                 # حل سياسة الأدوات
├── transcript-policy.ts           # سياسة التحقق من النص التفريغي
├── skills.ts                      # بناء لقطات Skills والموجّهات
├── skills/                        # النظام الفرعي لـ Skills
├── sandbox.ts                     # حل سياق sandbox
├── sandbox/                       # النظام الفرعي لـ sandbox
├── channel-tools.ts               # حقن الأدوات الخاصة بالقنوات
├── openclaw-tools.ts              # أدوات OpenClaw الخاصة
├── bash-tools.ts                  # أدوات exec/process
├── apply-patch.ts                 # أداة apply_patch (OpenAI)
├── tools/                         # تنفيذات الأدوات الفردية
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

توجد الآن أوقات تشغيل إجراءات الرسائل الخاصة بالقنوات في
أدلة الامتدادات المملوكة للإضافات بدلًا من `src/agents/tools`، على سبيل المثال:

- ملفات وقت تشغيل إجراءات إضافة Discord
- ملف وقت تشغيل إجراء إضافة Slack
- ملف وقت تشغيل إجراء إضافة Telegram
- ملف وقت تشغيل إجراء إضافة WhatsApp

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

داخل `runEmbeddedAttempt()` (الذي يستدعيه `runEmbeddedPiAgent()`)، تُستخدم Pi SDK:

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

يشترك `subscribeEmbeddedPiSession()` في أحداث `AgentSession` الخاصة بـ pi:

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

- `message_start` / `message_end` / `message_update` (بث النص/التفكير)
- `tool_execution_start` / `tool_execution_update` / `tool_execution_end`
- `turn_start` / `turn_end`
- `agent_start` / `agent_end`
- `auto_compaction_start` / `auto_compaction_end`

### 4. التوجيه

بعد الإعداد، يتم توجيه الموجّه إلى الجلسة:

```typescript
await session.prompt(effectivePrompt, { images: imageResult.images });
```

تتعامل SDK مع حلقة الوكيل الكاملة: الإرسال إلى LLM، وتنفيذ استدعاءات الأدوات، وبث الردود.

يكون حقن الصور محليًا للموجّه: يحمّل OpenClaw مراجع الصور من الموجّه الحالي و
يمررها عبر `images` لذلك الدور فقط. وهو لا يعيد فحص الأدوار الأقدم في السجل
لإعادة حقن حمولة الصور.

## بنية الأدوات

### مسار الأدوات

1. **الأدوات الأساسية**: `codingTools` الخاصة بـ pi (read وbash وedit وwrite)
2. **بدائل مخصصة**: يستبدل OpenClaw bash بـ `exec`/`process`، ويخصص read/edit/write لـ sandbox
3. **أدوات OpenClaw**: المراسلة، المتصفح، canvas، الجلسات، cron، gateway، وغيرها
4. **أدوات القنوات**: أدوات الإجراءات الخاصة بـ Discord/Telegram/Slack/WhatsApp
5. **تصفية السياسة**: تتم تصفية الأدوات حسب الملف الشخصي، والمزود، والوكيل، والمجموعة، وسياسات sandbox
6. **تطبيع المخطط**: تنظيف المخططات لمعالجة خصائص Gemini/OpenAI
7. **تغليف AbortSignal**: تغليف الأدوات لاحترام إشارات الإلغاء

### مهايئ تعريف الأدوات

يحتوي `AgentTool` من pi-agent-core على توقيع `execute` مختلف عن `ToolDefinition` في pi-coding-agent. يقوم المهايئ في `pi-tool-definition-adapter.ts` بردم هذه الفجوة:

```typescript
export function toToolDefinitions(tools: AnyAgentTool[]): ToolDefinition[] {
  return tools.map((tool) => ({
    name: tool.name,
    label: tool.label ?? name,
    description: tool.description ?? "",
    parameters: tool.parameters,
    execute: async (toolCallId, params, onUpdate, _ctx, signal) => {
      // يختلف توقيع pi-coding-agent عن pi-agent-core
      return await tool.execute(toolCallId, params, signal, onUpdate);
    },
  }));
}
```

### استراتيجية تقسيم الأدوات

يقوم `splitSdkTools()` بتمرير جميع الأدوات عبر `customTools`:

```typescript
export function splitSdkTools(options: { tools: AnyAgentTool[]; sandboxEnabled: boolean }) {
  return {
    builtInTools: [], // فارغ. نحن نستبدل كل شيء
    customTools: toToolDefinitions(options.tools),
  };
}
```

وهذا يضمن بقاء تصفية السياسات في OpenClaw، وتكامل sandbox، ومجموعة الأدوات الموسعة متسقة عبر جميع المزودين.

## بناء موجّه النظام

يُبنى موجّه النظام في `buildAgentSystemPrompt()` (`system-prompt.ts`). وهو يجمع موجّهًا كاملًا يتضمن أقسامًا مثل Tooling وTool Call Style وSafety guardrails ومرجع OpenClaw CLI وSkills وDocs وWorkspace وSandbox وMessaging وReply Tags وVoice وSilent Replies وHeartbeats وبيانات وقت التشغيل الوصفية، بالإضافة إلى Memory وReactions عند تفعيلهما، وكذلك ملفات السياق الاختيارية ومحتوى موجّه نظام إضافي. يتم تقليم الأقسام لاستخدام وضع الموجّه الأدنى المستخدم مع الوكلاء الفرعيين.

يُطبَّق الموجّه بعد إنشاء الجلسة عبر `applySystemPromptOverrideToSession()`:

```typescript
const systemPromptOverride = createSystemPromptOverride(appendPrompt);
applySystemPromptOverrideToSession(session, systemPromptOverride);
```

## إدارة الجلسات

### ملفات الجلسات

الجلسات هي ملفات JSONL ذات بنية شجرية (روابط id/parentId). يدير `SessionManager` في Pi عملية الاستمرارية:

```typescript
const sessionManager = SessionManager.open(params.sessionFile);
```

يلف OpenClaw هذا بواسطة `guardSessionManager()` من أجل أمان نتائج الأدوات.

### التخزين المؤقت للجلسات

يخزن `session-manager-cache.ts` نسخ SessionManager مؤقتًا لتجنب تحليل الملفات بشكل متكرر:

```typescript
await prewarmSessionFile(params.sessionFile);
sessionManager = SessionManager.open(params.sessionFile);
trackSessionManagerAccess(params.sessionFile);
```

### تحديد السجل

يقوم `limitHistoryTurns()` بتقليم سجل المحادثة بناءً على نوع القناة (DM مقابل المجموعة).

### الضغط

يتم تشغيل الضغط التلقائي عند تجاوز السياق. تشمل تواقيع التجاوز الشائعة
`request_too_large` و`context length exceeded` و`input exceeds the
maximum number of tokens` و`input token count exceeds the maximum number of
input tokens` و`input is too long for the model` و`ollama error: context
length exceeded`. ويتولى `compactEmbeddedPiSessionDirect()` عملية
الضغط اليدوي:

```typescript
const compactResult = await compactEmbeddedPiSessionDirect({
  sessionId, sessionFile, provider, model, ...
});
```

## المصادقة وحل النموذج

### ملفات تعريف المصادقة

يحتفظ OpenClaw بمخزن لملفات تعريف المصادقة يتضمن عدة مفاتيح API لكل مزود:

```typescript
const authStore = ensureAuthProfileStore(agentDir, { allowKeychainPrompt: false });
const profileOrder = resolveAuthProfileOrder({ cfg, store: authStore, provider, preferredProfile });
```

تتم مداورة الملفات عند الإخفاقات مع تتبع فترات التهدئة:

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

// يستخدم ModelRegistry وAuthStorage الخاصين بـ pi
authStorage.setRuntimeApiKey(model.provider, apiKeyInfo.apiKey);
```

### تجاوز الفشل

يؤدي `FailoverError` إلى تفعيل الرجوع إلى نموذج بديل عند الإعداد:

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

## امتدادات Pi

يحمّل OpenClaw امتدادات Pi مخصصة لسلوكيات متخصصة:

### حماية الضغط

يضيف `src/agents/pi-hooks/compaction-safeguard.ts` حواجز حماية إلى الضغط، بما في ذلك ميزانية رموز تكيفية بالإضافة إلى ملخصات إخفاق الأدوات وعمليات الملفات:

```typescript
if (resolveCompactionMode(params.cfg) === "safeguard") {
  setCompactionSafeguardRuntime(params.sessionManager, { maxHistoryShare });
  paths.push(resolvePiExtensionPath("compaction-safeguard"));
}
```

### تقليم السياق

ينفذ `src/agents/pi-hooks/context-pruning.ts` تقليم السياق المعتمد على Cache-TTL:

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

## البث والردود المقطعة

### تقسيم الكتل

يدير `EmbeddedBlockChunker` النص المتدفق وتحويله إلى كتل رد منفصلة:

```typescript
const blockChunker = blockChunking ? new EmbeddedBlockChunker(blockChunking) : null;
```

### إزالة وسوم التفكير/الوسوم النهائية

تتم معالجة المخرجات المتدفقة لإزالة كتل `<think>`/`<thinking>` واستخراج محتوى `<final>`:

```typescript
const stripBlockTags = (text: string, state: { thinking: boolean; final: boolean }) => {
  // إزالة محتوى <think>...</think>
  // إذا كان enforceFinalTag مفعّلًا، فأعد فقط محتوى <final>...</final>
};
```

### توجيهات الرد

يتم تحليل توجيهات الرد مثل `[[media:url]]` و`[[voice]]` و`[[reply:id]]` واستخراجها:

```typescript
const { text: cleanedText, mediaUrls, audioAsVoice, replyToId } = consumeReplyDirectives(chunk);
```

## معالجة الأخطاء

### تصنيف الأخطاء

يقوم `pi-embedded-helpers.ts` بتصنيف الأخطاء من أجل معالجتها بالشكل المناسب:

```typescript
isContextOverflowError(errorText)     // السياق كبير جدًا
isCompactionFailureError(errorText)   // فشل الضغط
isAuthAssistantError(lastAssistant)   // فشل المصادقة
isRateLimitAssistantError(...)        // تم بلوغ حد المعدل
isFailoverAssistantError(...)         // يجب تنفيذ تجاوز الفشل
classifyFailoverReason(errorText)     // "auth" | "rate_limit" | "quota" | "timeout" | ...
```

### الرجوع في مستوى التفكير

إذا كان مستوى التفكير غير مدعوم، يتم الرجوع إلى مستوى بديل:

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

## تكامل sandbox

عند تفعيل وضع sandbox، يتم تقييد الأدوات والمسارات:

```typescript
const sandbox = await resolveSandboxContext({
  config: params.config,
  sessionKey: sandboxSessionKey,
  workspaceDir: resolvedWorkspace,
});

if (sandboxRoot) {
  // استخدام أدوات read/edit/write داخل sandbox
  // يعمل Exec داخل الحاوية
  // يستخدم Browser عنوان URL للجسر
}
```

## المعالجة الخاصة بالمزود

### Anthropic

- إزالة سلسلة الرفض السحرية
- التحقق من الدور للأدوار المتتالية
- تحقق صارم من معلمات أدوات Pi في upstream

### Google/Gemini

- تنظيف مخطط الأدوات المملوك للإضافة

### OpenAI

- أداة `apply_patch` لنماذج Codex
- التعامل مع خفض مستوى التفكير

## تكامل TUI

يحتوي OpenClaw أيضًا على وضع TUI محلي يستخدم مكونات pi-tui مباشرة:

```typescript
// src/tui/tui.ts
import { ... } from "@mariozechner/pi-tui";
```

ويوفر هذا تجربة طرفية تفاعلية مشابهة لوضع pi الأصلي.

## الفروق الأساسية عن Pi CLI

| الجانب | Pi CLI | OpenClaw المضمن |
| --------------- | ----------------------- | ---------------------------------------------------------------------------------------------- |
| الاستدعاء | أمر `pi` / RPC | SDK عبر `createAgentSession()` |
| الأدوات | أدوات البرمجة الافتراضية | مجموعة أدوات OpenClaw مخصصة |
| موجّه النظام | `AGENTS.md` + موجّهات | ديناميكي لكل قناة/سياق |
| تخزين الجلسات | `~/.pi/agent/sessions/` | `~/.openclaw/agents/<agentId>/sessions/` (أو `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`) |
| المصادقة | بيانات اعتماد واحدة | ملفات تعريف متعددة مع تدوير |
| الامتدادات | تُحمّل من القرص | برمجيًا + مسارات قرص |
| معالجة الأحداث | عرض TUI | قائم على callbacks (مثل onBlockReply، وغيرها) |

## اعتبارات مستقبلية

مجالات قد تحتاج إلى إعادة عمل محتملة:

1. **محاذاة توقيع الأداة**: يجري حاليًا التكييف بين تواقيع pi-agent-core وpi-coding-agent
2. **تغليف مدير الجلسة**: يضيف `guardSessionManager` أمانًا لكنه يزيد التعقيد
3. **تحميل الامتدادات**: يمكن استخدام `ResourceLoader` الخاص بـ Pi بصورة أكثر مباشرة
4. **تعقيد معالج البث**: ازداد حجم `subscribeEmbeddedPiSession`
5. **خصائص المزود**: توجد مسارات شيفرة خاصة بكثير من المزودين وقد تتمكن pi من التعامل معها مستقبلًا

## الاختبارات

يغطي تكامل Pi مجموعات الاختبار التالية:

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

مباشر/اختياري:

- `src/agents/pi-embedded-runner-extraparams.live.test.ts` (فعّل `OPENCLAW_LIVE_TEST=1`)

للاطلاع على أوامر التشغيل الحالية، راجع [Pi Development Workflow](/ar/pi-dev).
