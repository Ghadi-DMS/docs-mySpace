---
read_when:
    - تريد رجوعًا احتياطيًا موثوقًا عندما تفشل مزودات API
    - أنت تشغّل Claude CLI أو AI CLIs محلية أخرى وتريد إعادة استخدامها
    - تريد فهم جسر MCP loopback للوصول إلى أدوات CLI backend
summary: 'CLI backends: رجوع احتياطي إلى AI CLI محلي مع جسر أدوات MCP اختياري'
title: CLI Backends
x-i18n:
    generated_at: "2026-04-05T12:42:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 823f3aeea6be50e5aa15b587e0944e79e862cecb7045f9dd44c93c544024bce1
    source_path: gateway/cli-backends.md
    workflow: 15
---

# CLI backends (وقت تشغيل الرجوع الاحتياطي)

يمكن لـ OpenClaw تشغيل **AI CLIs محلية** كـ **رجوع احتياطي نصي فقط** عندما تكون مزودات API متوقفة،
أو خاضعة لحدود المعدل، أو تتصرف بشكل غير صحيح مؤقتًا. هذا السلوك محافظ عمدًا:

- **لا يتم حقن أدوات OpenClaw مباشرة**، لكن يمكن للـ backends التي تحتوي على `bundleMcp: true`
  (وهو الإعداد الافتراضي لـ Claude CLI) أن تتلقى أدوات gateway عبر جسر MCP loopback.
- **بث JSONL** (يستخدم Claude CLI القيمة `--output-format stream-json` مع
  `--include-partial-messages`؛ وتُرسل المطالبات عبر stdin).
- **الجلسات مدعومة** (بحيث تبقى الأدوار اللاحقة مترابطة).
- **يمكن تمرير الصور** إذا كان CLI يقبل مسارات الصور.

هذا مصمم ليكون **شبكة أمان** أكثر من كونه مسارًا أساسيًا. استخدمه عندما
تريد استجابات نصية من نوع “يعمل دائمًا” من دون الاعتماد على APIs خارجية.

إذا كنت تريد وقت تشغيل harness كاملًا مع عناصر تحكم جلسات ACP، والمهام الخلفية،
وربط الخيوط/المحادثات، وجلسات ترميز خارجية دائمة، فاستخدم
[ACP Agents](/tools/acp-agents) بدلًا من ذلك. إن CLI backends ليست ACP.

## بدء سريع مناسب للمبتدئين

يمكنك استخدام Claude CLI **من دون أي تكوين** (فـ plugin المضمّن الخاص بـ Anthropic
يسجّل backend افتراضيًا):

```bash
openclaw agent --message "hi" --model claude-cli/claude-sonnet-4-6
```

يعمل Codex CLI أيضًا مباشرةً (عبر plugin المضمّن الخاص بـ OpenAI):

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

إذا كان gateway يعمل تحت launchd/systemd وكانت قيمة PATH محدودة، فأضف فقط
مسار الأمر:

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

هذا كل شيء. لا حاجة إلى مفاتيح أو إعداد مصادقة إضافي بخلاف CLI نفسها.

إذا كنت تستخدم CLI backend مضمّنة بصفتها **مزوّد الرسائل الأساسي** على
مضيف gateway، فإن OpenClaw يحمّل الآن plugin المضمّن المالكة تلقائيًا عندما يشير تكوينك
صراحةً إلى ذلك backend في مرجع نموذج أو تحت
`agents.defaults.cliBackends`.

## استخدامها كرجوع احتياطي

أضف CLI backend إلى قائمة fallback بحيث تعمل فقط عند فشل النماذج الأساسية:

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

ملاحظات:

- إذا كنت تستخدم `agents.defaults.models` (قائمة السماح)، فيجب أن تتضمن `claude-cli/...`.
- إذا فشل المزوّد الأساسي (المصادقة، حدود المعدل، المهلات)، فسيحاول OpenClaw
  CLI backend بعده.
- لا يزال Claude CLI backend المضمّن يقبل الأسماء المستعارة الأقصر مثل
  `claude-cli/opus` أو `claude-cli/opus-4.6` أو `claude-cli/sonnet`، لكن
  تستخدم الوثائق وأمثلة التكوين المراجع الأساسية `claude-cli/claude-*`.

## نظرة عامة على التكوين

توجد جميع CLI backends تحت:

```
agents.defaults.cliBackends
```

يكون كل إدخال مفهرسًا بواسطة **معرّف مزوّد** (مثل `claude-cli` أو `my-cli`).
ويصبح معرّف المزوّد هو الطرف الأيسر من مرجع النموذج لديك:

```
<provider>/<model>
```

### مثال على التكوين

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

## كيف يعمل

1. **يحدد backend** بناءً على بادئة المزوّد (`claude-cli/...`).
2. **يبني مطالبة نظام** باستخدام مطالبة OpenClaw نفسها + سياق مساحة العمل.
3. **ينفّذ CLI** باستخدام معرّف جلسة (إذا كان مدعومًا) حتى يبقى السجل متسقًا.
4. **يحلل الإخراج** (JSON أو نص عادي) ويعيد النص النهائي.
5. **يحفظ معرّفات الجلسات** لكل backend، بحيث تعيد الأدوار اللاحقة استخدام جلسة CLI نفسها.

## الجلسات

- إذا كانت CLI تدعم الجلسات، فاضبط `sessionArg` (مثل `--session-id`) أو
  `sessionArgs` (العنصر النائب `{sessionId}`) عندما يلزم إدراج المعرّف
  في عدة علامات.
- إذا كانت CLI تستخدم **أمرًا فرعيًا للاستئناف** مع علامات مختلفة، فاضبط
  `resumeArgs` (ويحل محل `args` عند الاستئناف) واختياريًا `resumeOutput`
  (لعمليات الاستئناف غير المعتمدة على JSON).
- `sessionMode`:
  - `always`: أرسل دائمًا معرّف جلسة (UUID جديد إذا لم يكن هناك معرّف مخزّن).
  - `existing`: أرسل معرّف جلسة فقط إذا كان مخزنًا من قبل.
  - `none`: لا ترسل معرّف جلسة أبدًا.

ملاحظات حول التسلسل:

- يحافظ `serialize: true` على ترتيب التشغيلات في المسار نفسه.
- تسلسل معظم CLIs يكون على مسار مزوّد واحد.
- `claude-cli` أضيق من ذلك: تُسلسل التشغيلات المستأنفة لكل معرّف جلسة Claude، وتُسلسل التشغيلات الجديدة لكل مسار مساحة عمل. ويمكن لمساحات العمل المستقلة أن تعمل بالتوازي.
- يتخلى OpenClaw عن إعادة استخدام جلسة CLI المخزنة عندما تتغير حالة مصادقة backend، بما في ذلك إعادة تسجيل الدخول، أو تدوير الرموز، أو تغيير بيانات اعتماد ملف تعريف المصادقة.

## الصور (تمرير مباشر)

إذا كانت CLI لديك تقبل مسارات الصور، فاضبط `imageArg`:

```json5
imageArg: "--image",
imageMode: "repeat"
```

سيكتب OpenClaw صور base64 إلى ملفات مؤقتة. وإذا تم ضبط `imageArg`، فسيتم
تمرير هذه المسارات كوسائط CLI. وإذا كان `imageArg` غير موجود، فإن OpenClaw
يُلحق مسارات الملفات بالمطالبة (حقن مسار)، وهو ما يكفي للـ CLIs التي
تحمّل الملفات المحلية تلقائيًا من المسارات النصية العادية (سلوك Claude CLI).

## الإدخالات / المخرجات

- `output: "json"` (الافتراضي) يحاول تحليل JSON واستخراج النص + معرّف الجلسة.
- بالنسبة إلى مخرجات JSON الخاصة بـ Gemini CLI، يقرأ OpenClaw نص الرد من `response`
  والاستخدام من `stats` عندما تكون `usage` مفقودة أو فارغة.
- يحلل `output: "jsonl"` تدفقات JSONL (مثل Claude CLI `stream-json`
  وCodex CLI `--json`) ويستخرج رسالة الوكيل النهائية بالإضافة إلى معرّفات
  الجلسة عند وجودها.
- `output: "text"` يعامل stdout على أنه الاستجابة النهائية.

أوضاع الإدخال:

- `input: "arg"` (الافتراضي) يمرر المطالبة باعتبارها آخر وسيطة CLI.
- `input: "stdin"` يرسل المطالبة عبر stdin.
- إذا كانت المطالبة طويلة جدًا وكان `maxPromptArgChars` مضبوطًا، فسيُستخدم stdin.

## الإعدادات الافتراضية (مملوكة لـ plugin)

يسجل plugin المضمّن الخاص بـ Anthropic إعدادًا افتراضيًا لـ `claude-cli`:

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

يسجل plugin المضمّن الخاص بـ OpenAI أيضًا إعدادًا افتراضيًا لـ `codex-cli`:

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

يسجل plugin المضمّن الخاص بـ Google أيضًا إعدادًا افتراضيًا لـ `google-gemini-cli`:

- `command: "gemini"`
- `args: ["--prompt", "--output-format", "json"]`
- `resumeArgs: ["--resume", "{sessionId}", "--prompt", "--output-format", "json"]`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

المتطلب الأساسي: يجب أن تكون Gemini CLI المحلية مثبتة ومتاحة باسم
`gemini` على `PATH` (`brew install gemini-cli` أو
`npm install -g @google/gemini-cli`).

ملاحظات JSON الخاصة بـ Gemini CLI:

- يُقرأ نص الرد من الحقل `response` في JSON.
- يعود الاستخدام إلى `stats` عندما تكون `usage` غائبة أو فارغة.
- تُطبّع `stats.cached` إلى OpenClaw `cacheRead`.
- إذا كانت `stats.input` مفقودة، يشتق OpenClaw رموز الإدخال من
  `stats.input_tokens - stats.cached`.

لا تتجاوز هذه الإعدادات إلا عند الحاجة (والأكثر شيوعًا: مسار `command` مطلق).

## الإعدادات الافتراضية المملوكة لـ plugin

أصبحت الإعدادات الافتراضية لـ CLI backend الآن جزءًا من سطح plugin:

- تسجلها plugins باستخدام `api.registerCliBackend(...)`.
- يصبح `id` الخاص بـ backend هو بادئة المزوّد في مراجع النموذج.
- يظل تكوين المستخدم في `agents.defaults.cliBackends.<id>` متقدمًا على الإعداد الافتراضي للـ plugin.
- يظل تنظيف التكوين الخاص بـ backend مملوكًا للـ plugin عبر الخطاف الاختياري
  `normalizeConfig`.

## تراكبات Bundle MCP

لا تتلقى CLI backends **استدعاءات أدوات OpenClaw مباشرة**، لكن يمكن لـ backend
الاشتراك في تراكب MCP مكوّن مولّد باستخدام `bundleMcp: true`.

السلوك المضمّن الحالي:

- `claude-cli`: ‏`bundleMcp: true` (الافتراضي)
- `codex-cli`: لا يوجد تراكب bundle MCP
- `google-gemini-cli`: لا يوجد تراكب bundle MCP

عند تمكين bundle MCP، فإن OpenClaw:

- يشغّل خادم MCP محليًا عبر HTTP loopback يكشف أدوات gateway لعملية CLI
- يصادق الجسر باستخدام رمز لكل جلسة (`OPENCLAW_MCP_TOKEN`)
- يقيّد الوصول إلى الأدوات بسياق الجلسة والحساب والقناة الحالية
- يحمّل خوادم bundle-MCP الممكّنة لمساحة العمل الحالية
- يدمجها مع أي `--mcp-config` موجودة مسبقًا في backend
- يعيد كتابة وسائط CLI لتمرير `--strict-mcp-config --mcp-config <generated-file>`

يمنع العلم `--strict-mcp-config` Claude CLI من وراثة خوادم MCP
الموجودة على مستوى المستخدم أو المستوى العام. وإذا لم تكن هناك خوادم MCP ممكّنة، فإن OpenClaw
يحقن مع ذلك تكوينًا صارمًا فارغًا حتى تبقى التشغيلات الخلفية معزولة.

## القيود

- **لا توجد استدعاءات مباشرة لأدوات OpenClaw.** لا يحقن OpenClaw استدعاءات الأدوات ضمن
  بروتوكول CLI backend. ومع ذلك، فإن backends التي تحتوي على `bundleMcp: true` (وهو
  الإعداد الافتراضي لـ Claude CLI) تتلقى أدوات gateway عبر جسر MCP loopback،
  لذلك يستطيع Claude CLI استدعاء أدوات OpenClaw عبر دعم MCP الأصلي لديه.
- **يعتمد البث على backend.** يستخدم Claude CLI بث JSONL
  (`stream-json` مع `--include-partial-messages`)؛ وقد
  تظل CLI backends الأخرى مخزنة مؤقتًا حتى الخروج.
- **المخرجات المهيكلة** تعتمد على تنسيق JSON الخاص بـ CLI.
- **جلسات Codex CLI** تُستأنف عبر مخرجات نصية (وليس JSONL)، وهو ما يجعلها
  أقل هيكلةً من التشغيل الأولي باستخدام `--json`. ومع ذلك، تظل جلسات OpenClaw
  تعمل بشكل طبيعي.

## استكشاف الأخطاء وإصلاحها

- **تعذر العثور على CLI**: اضبط `command` على مسار كامل.
- **اسم نموذج خاطئ**: استخدم `modelAliases` لربط `provider/model` → نموذج CLI.
- **لا يوجد استمرارية للجلسة**: تأكد من ضبط `sessionArg` وأن `sessionMode` ليست
  `none` (لا يستطيع Codex CLI حاليًا الاستئناف مع مخرجات JSON).
- **يتم تجاهل الصور**: اضبط `imageArg` (وتأكد من أن CLI تدعم مسارات الملفات).
