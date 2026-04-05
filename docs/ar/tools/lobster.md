---
read_when:
    - تريد سير عمل حتميًا متعدد الخطوات مع موافقات صريحة
    - تحتاج إلى استئناف سير عمل دون إعادة تشغيل الخطوات السابقة
summary: وقت تشغيل سير عمل مضبوط الأنواع لـ OpenClaw مع بوابات موافقة قابلة للاستئناف.
title: Lobster
x-i18n:
    generated_at: "2026-04-05T12:59:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 82718c15d571406ad6f1507de22a528fdab873edfc6aafae10742e500f6a5eda
    source_path: tools/lobster.md
    workflow: 15
---

# Lobster

Lobster هو غلاف سير عمل يتيح لـ OpenClaw تشغيل تسلسلات أدوات متعددة الخطوات كعملية واحدة حتمية مع نقاط تحقق موافقة صريحة.

يمثل Lobster طبقة تأليف أعلى بدرجة واحدة من أعمال الخلفية المنفصلة. ولتنسيق التدفق فوق المهام الفردية، راجع [Task Flow](/ar/automation/taskflow) ‏(`openclaw tasks flow`). أما لسجل نشاط المهام، فراجع [`openclaw tasks`](/ar/automation/tasks).

## الفكرة الأساسية

يمكن لمساعدك بناء الأدوات التي تدير نفسه. اطلب سير عمل، وبعد 30 دقيقة يصبح لديك CLI بالإضافة إلى مسارات تنفيذ تعمل في استدعاء واحد. Lobster هو القطعة الناقصة: مسارات تنفيذ حتمية، وموافقات صريحة، وحالة قابلة للاستئناف.

## لماذا

اليوم، تتطلب تدفقات العمل المعقدة العديد من استدعاءات الأدوات ذهابًا وإيابًا. كل استدعاء يكلّف رموزًا، ويجب على LLM تنسيق كل خطوة. ينقل Lobster هذا التنسيق إلى وقت تشغيل مضبوط الأنواع:

- **استدعاء واحد بدلًا من عدة استدعاءات**: يشغّل OpenClaw استدعاء أداة Lobster واحدًا ويحصل على نتيجة منظَّمة.
- **الموافقات مضمّنة**: توقف التأثيرات الجانبية (إرسال بريد إلكتروني، أو نشر تعليق) سير العمل حتى تتم الموافقة عليها صراحةً.
- **قابل للاستئناف**: تعيد تدفقات العمل المتوقفة رمزًا مميزًا؛ وافق واستأنف دون إعادة تشغيل كل شيء.

## لماذا DSL بدلًا من برامج عادية؟

Lobster صغير عمدًا. فالهدف ليس "لغة جديدة"، بل مواصفة مسار تنفيذ متوقعة وملائمة لـ AI مع موافقات من الدرجة الأولى ورموز استئناف.

- **الموافقة/الاستئناف مضمّنان**: يمكن لبرنامج عادي أن يطلب من إنسان الموافقة، لكنه لا يستطيع _التوقف والاستئناف_ باستخدام رمز دائم من دون أن تخترع أنت وقت التشغيل هذا بنفسك.
- **الحتمية + القابلية للتدقيق**: مسارات التنفيذ عبارة عن بيانات، لذا يسهل تسجيلها، ومقارنتها، وإعادة تشغيلها، ومراجعتها.
- **سطح مقيّد لـ AI**: يقلل النحو الصغير + تمرير JSON من مسارات الشيفرة "الإبداعية" ويجعل التحقق واقعيًا.
- **سياسة الأمان مدمجة**: تُفرض المهلات، وحدود المخرجات، وفحوصات sandbox، وقوائم السماح بواسطة وقت التشغيل، وليس بواسطة كل سكربت.
- **لا يزال قابلًا للبرمجة**: يمكن لكل خطوة استدعاء أي CLI أو سكربت. وإذا كنت تريد JS/TS، فأنشئ ملفات `.lobster` من الشيفرة.

## كيف يعمل

يشغّل OpenClaw ‏CLI المحلي `lobster` في **وضع الأداة** ويحلل غلاف JSON من stdout.
وإذا توقّف مسار التنفيذ لطلب موافقة، تعيد الأداة `resumeToken` حتى تتمكن من المتابعة لاحقًا.

## النمط: CLI صغير + تمرير JSON + موافقات

ابنِ أوامر صغيرة تتحدث JSON، ثم اربطها في استدعاء Lobster واحد. (أسماء الأوامر أدناه مجرد أمثلة — استبدلها بأوامرك الخاصة.)

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt 'Apply changes?'",
  "timeoutMs": 30000
}
```

إذا طلب مسار التنفيذ موافقة، فاستأنف باستخدام الرمز:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

يقوم AI بإطلاق سير العمل؛ وينفذ Lobster الخطوات. وتُبقي بوابات الموافقة التأثيرات الجانبية صريحة وقابلة للتدقيق.

مثال: حوّل عناصر الإدخال إلى استدعاءات أدوات:

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## خطوات LLM بتنسيق JSON فقط (`llm-task`)

بالنسبة إلى تدفقات العمل التي تحتاج إلى **خطوة LLM منظَّمة**، فعّل أداة plugin الاختيارية
`llm-task` واستدعها من Lobster. وهذا يُبقي سير العمل
حتميًا مع السماح في الوقت نفسه بالتصنيف/التلخيص/الصياغة باستخدام نموذج.

فعّل الأداة:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "allow": ["llm-task"] }
      }
    ]
  }
}
```

استخدمها داخل مسار تنفيذ:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Given the input email, return intent and draft.",
  "thinking": "low",
  "input": { "subject": "Hello", "body": "Can you help?" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

راجع [LLM Task](/tools/llm-task) للحصول على التفاصيل وخيارات الإعداد.

## ملفات سير العمل (.lobster)

يمكن لـ Lobster تشغيل ملفات سير عمل YAML/JSON تحتوي على الحقول `name` و`args` و`steps` و`env` و`condition` و`approval`. وفي استدعاءات أدوات OpenClaw، اضبط `pipeline` على مسار الملف.

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

ملاحظات:

- يمرر `stdin: $step.stdout` و`stdin: $step.json` مخرجات خطوة سابقة.
- يمكن لـ `condition` ‏(أو `when`) تقييد الخطوات بناءً على `$step.approved`.

## تثبيت Lobster

ثبّت Lobster CLI على **المضيف نفسه** الذي يشغّل OpenClaw Gateway ‏(راجع [مستودع Lobster](https://github.com/openclaw/lobster))، وتأكد من أن `lobster` موجود على `PATH`.

## تفعيل الأداة

Lobster هي **أداة plugin اختيارية** (وليست مفعّلة افتراضيًا).

الموصى به (إضافي وآمن):

```json
{
  "tools": {
    "alsoAllow": ["lobster"]
  }
}
```

أو لكل وكيل:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "alsoAllow": ["lobster"]
        }
      }
    ]
  }
}
```

تجنب استخدام `tools.allow: ["lobster"]` ما لم تكن تنوي العمل في وضع قائمة السماح المقيّد.

ملاحظة: قوائم السماح اختيارية بالنسبة إلى plugins الاختيارية. وإذا كانت قائمة السماح لديك تسمّي فقط
أدوات plugin ‏(مثل `lobster`)، فإن OpenClaw يبقي الأدوات الأساسية مفعّلة. ولتقييد الأدوات الأساسية،
أدرج الأدوات الأساسية أو المجموعات التي تريدها في قائمة السماح أيضًا.

## مثال: فرز البريد الإلكتروني

من دون Lobster:

```
User: "Check my email and draft replies"
→ openclaw calls gmail.list
→ LLM summarizes
→ User: "draft replies to #2 and #5"
→ LLM drafts
→ User: "send #2"
→ openclaw calls gmail.send
(repeat daily, no memory of what was triaged)
```

مع Lobster:

```json
{
  "action": "run",
  "pipeline": "email.triage --limit 20",
  "timeoutMs": 30000
}
```

يعيد غلاف JSON ‏(مبتورًا):

```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "5 need replies, 2 need action" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "Send 2 draft replies?",
    "items": [],
    "resumeToken": "..."
  }
}
```

يوافق المستخدم ← استئناف:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

سير عمل واحد. حتمي. آمن.

## معلمات الأداة

### `run`

شغّل مسار تنفيذ في وضع الأداة.

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

شغّل ملف سير عمل مع وسائط:

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

### `resume`

تابع سير عمل متوقفًا بعد الموافقة.

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

### المدخلات الاختيارية

- `cwd`: دليل عمل نسبي لمسار التنفيذ (ويجب أن يبقى ضمن دليل العمل الحالي للعملية).
- `timeoutMs`: اقضِ على العملية الفرعية إذا تجاوزت هذه المدة (الافتراضي: 20000).
- `maxStdoutBytes`: اقضِ على العملية الفرعية إذا تجاوز stdout هذا الحجم (الافتراضي: 512000).
- `argsJson`: سلسلة JSON تُمرَّر إلى `lobster run --args-json` ‏(لملفات سير العمل فقط).

## غلاف المخرجات

يعيد Lobster غلاف JSON بإحدى الحالات الثلاث:

- `ok` → انتهى بنجاح
- `needs_approval` → متوقف مؤقتًا؛ ويتطلب `requiresApproval.resumeToken` للاستئناف
- `cancelled` → مرفوض أو ملغى صراحةً

تعرض الأداة الغلاف في كل من `content` ‏(JSON منسّق) و`details` ‏(الكائن الخام).

## الموافقات

إذا كان `requiresApproval` موجودًا، فافحص الموجّه وقرّر:

- `approve: true` → استأنف وتابع التأثيرات الجانبية
- `approve: false` → ألغِ سير العمل وأنهه

استخدم `approve --preview-from-stdin --limit N` لإرفاق معاينة JSON بطلبات الموافقة من دون glue مخصص من jq/heredoc. وأصبحت رموز الاستئناف الآن مدمجة: يخزّن Lobster حالة استئناف سير العمل تحت دليل حالته ويعيد مفتاح رمز صغيرًا.

## OpenProse

يتكامل OpenProse جيدًا مع Lobster: استخدم `/prose` لتنسيق التحضير متعدد الوكلاء، ثم شغّل مسار تنفيذ Lobster للحصول على موافقات حتمية. وإذا كان برنامج Prose يحتاج إلى Lobster، فاسمح بأداة `lobster` للوكلاء الفرعيين عبر `tools.subagents.tools`. راجع [OpenProse](/ar/prose).

## الأمان

- **عمليات فرعية محلية فقط** — لا توجد استدعاءات شبكة من plugin نفسه.
- **لا أسرار** — لا يدير Lobster ‏OAuth؛ بل يستدعي أدوات OpenClaw التي تقوم بذلك.
- **مدرك لـ sandbox** — يتم تعطيله عندما يكون سياق الأداة داخل sandbox.
- **مقوّى** — اسم تنفيذي ثابت (`lobster`) على `PATH`؛ مع فرض المهلات وحدود المخرجات.

## استكشاف الأخطاء وإصلاحها

- **`lobster subprocess timed out`** → زد `timeoutMs`، أو قسّم مسار تنفيذ طويلًا.
- **`lobster output exceeded maxStdoutBytes`** → ارفع `maxStdoutBytes` أو قلّل حجم المخرجات.
- **`lobster returned invalid JSON`** → تأكد من أن مسار التنفيذ يعمل في وضع الأداة ويطبع JSON فقط.
- **`lobster failed (code …)`** → شغّل مسار التنفيذ نفسه في طرفية لفحص stderr.

## تعلّم المزيد

- [Plugins](/tools/plugin)
- [تأليف أدوات plugin](/ar/plugins/building-plugins#registering-agent-tools)

## دراسة حالة: تدفقات عمل المجتمع

أحد الأمثلة العامة: CLI ‏"الدماغ الثاني" + مسارات تنفيذ Lobster التي تدير ثلاثة مخازن Markdown ‏(شخصي، وشريك، ومشترك). يصدر CLI بيانات JSON للإحصاءات، وقوائم الوارد، وعمليات فحص العناصر القديمة؛ ويربط Lobster هذه الأوامر في تدفقات عمل مثل `weekly-review` و`inbox-triage` و`memory-consolidation` و`shared-task-sync`، وكلها مع بوابات موافقة. ويتعامل AI مع الأحكام (التصنيف) عند توفره ويعود إلى قواعد حتمية عند عدم توفره.

- سلسلة: [https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- المستودع: [https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## ذو صلة

- [الأتمتة والمهام](/ar/automation) — جدولة تدفقات عمل Lobster
- [نظرة عامة على الأتمتة](/ar/automation) — جميع آليات الأتمتة
- [نظرة عامة على الأدوات](/tools) — جميع أدوات الوكيل المتاحة
