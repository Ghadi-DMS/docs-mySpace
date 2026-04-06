---
read_when:
    - أنت تريد مسارات عمل حتمية متعددة الخطوات مع موافقات صريحة
    - أنت بحاجة إلى استئناف مسار عمل من دون إعادة تشغيل الخطوات السابقة
summary: بيئة تشغيل Typed لمسارات العمل في OpenClaw مع بوابات موافقة قابلة للاستئناف.
title: Lobster
x-i18n:
    generated_at: "2026-04-06T03:14:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: c1014945d104ef8fdca0d30be89e35136def1b274c6403b06de29e8502b8124b
    source_path: tools/lobster.md
    workflow: 15
---

# Lobster

Lobster هو غلاف لمسارات العمل يتيح لـ OpenClaw تشغيل تسلسلات أدوات متعددة الخطوات كعملية واحدة حتمية مع نقاط تحقق صريحة للموافقة.

يقع Lobster على مستوى تأليف واحد فوق العمل المنفصل في الخلفية. ولتنسيق التدفقات فوق المهام الفردية، راجع [Task Flow](/ar/automation/taskflow) ‏(`openclaw tasks flow`). أما بالنسبة إلى سجل نشاط المهام، فراجع [`openclaw tasks`](/ar/automation/tasks).

## الخطاف

يمكن لمساعدك بناء الأدوات التي تدير نفسه. اطلب مسار عمل، وبعد 30 دقيقة يصبح لديك CLI بالإضافة إلى خطوط معالجة تعمل كمكالمة واحدة. Lobster هو القطعة الناقصة: خطوط معالجة حتمية، وموافقات صريحة، وحالة قابلة للاستئناف.

## لماذا

اليوم، تتطلب مسارات العمل المعقدة العديد من استدعاءات الأدوات ذهابًا وإيابًا. كل استدعاء يستهلك رموزًا، ويجب على LLM تنسيق كل خطوة. ينقل Lobster هذا التنسيق إلى بيئة تشغيل typed:

- **مكالمة واحدة بدلًا من العديد**: يشغّل OpenClaw استدعاء أداة Lobster واحدًا ويحصل على نتيجة منظَّمة.
- **الموافقات مدمجة**: تؤدي التأثيرات الجانبية (إرسال بريد إلكتروني، ونشر تعليق) إلى إيقاف مسار العمل حتى تتم الموافقة عليه صراحةً.
- **قابل للاستئناف**: تعيد مسارات العمل المتوقفة رمزًا مميزًا؛ وافق ثم استأنف من دون إعادة تشغيل كل شيء.

## لماذا DSL بدلًا من البرامج العادية؟

Lobster صغير عمدًا. والهدف ليس "لغة جديدة"، بل مواصفة خطوط معالجة قابلة للتنبؤ وصديقة للذكاء الاصطناعي مع موافقات من الدرجة الأولى ورموز استئناف.

- **الموافقة/الاستئناف مدمجان**: يمكن للبرنامج العادي مطالبة إنسان، لكنه لا يستطيع _التوقف والاستئناف_ باستخدام رمز دائم من دون أن تبتكر أنت بنفسك بيئة التشغيل تلك.
- **الحتمية + القابلية للتدقيق**: خطوط المعالجة هي بيانات، لذا يسهل تسجيلها، وإظهار الفروقات بينها، وإعادة تشغيلها، ومراجعتها.
- **سطح مقيّد للذكاء الاصطناعي**: إن وجود قواعد صغيرة + تمرير JSON يقلل مسارات الشيفرة "الإبداعية" ويجعل التحقق واقعيًا.
- **سياسة الأمان مدمجة**: تُفرَض المهلات، وحدود المخرجات، وفحوصات sandbox، وقوائم السماح بواسطة بيئة التشغيل، لا بواسطة كل برنامج نصي.
- **لا يزال قابلًا للبرمجة**: يمكن لكل خطوة استدعاء أي CLI أو برنامج نصي. وإذا كنت تريد JS/TS، فأنشئ ملفات `.lobster` من الشيفرة.

## كيف يعمل

يشغّل OpenClaw مسارات عمل Lobster **داخل العملية** باستخدام مشغّل مضمّن. لا يتم تشغيل أي عملية فرعية خارجية لـ CLI؛ إذ يعمل محرك مسار العمل داخل عملية البوابة ويعيد غلاف JSON مباشرة.
إذا توقف خط المعالجة من أجل الموافقة، تعيد الأداة `resumeToken` حتى تتمكن من المتابعة لاحقًا.

## النمط: CLI صغير + تمرير JSON + موافقات

ابنِ أوامر صغيرة تتحدث JSON، ثم اربطها في استدعاء Lobster واحد. (أسماء الأوامر أدناه أمثلة فقط — استبدلها بأوامرك الخاصة.)

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

إذا طلب خط المعالجة موافقة، فاستأنف باستخدام الرمز:

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

يقوم الذكاء الاصطناعي بتشغيل مسار العمل؛ وينفذ Lobster الخطوات. وتحافظ بوابات الموافقة على كون التأثيرات الجانبية صريحة وقابلة للتدقيق.

مثال: ربط عناصر الإدخال باستدعاءات الأدوات:

```bash
gog.gmail.search --query 'newer_than:1d' \
  | openclaw.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## خطوات LLM المعتمدة على JSON فقط (`llm-task`)

بالنسبة إلى مسارات العمل التي تحتاج إلى **خطوة LLM منظَّمة**، فعّل أداة الإضافة الاختيارية
`llm-task` واستدعها من Lobster. يحافظ هذا على حتمية مسار العمل
مع الاستمرار في السماح لك بالتصنيف/التلخيص/الصياغة باستخدام نموذج.

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

استخدمها في خط معالجة:

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

راجع [LLM Task](/ar/tools/llm-task) للحصول على التفاصيل وخيارات الإعداد.

## ملفات مسارات العمل (.lobster)

يمكن لـ Lobster تشغيل ملفات YAML/JSON الخاصة بمسارات العمل التي تحتوي على الحقول `name` و`args` و`steps` و`env` و`condition` و`approval`. وفي استدعاءات أدوات OpenClaw، اضبط `pipeline` على مسار الملف.

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
- يمكن لـ `condition` (أو `when`) أن يقيّد الخطوات اعتمادًا على `$step.approved`.

## تثبيت Lobster

تعمل مسارات عمل Lobster المضمّنة داخل العملية؛ ولا يلزم وجود ملف ثنائي منفصل باسم `lobster`. ويأتي المشغّل المضمّن مع إضافة Lobster.

إذا كنت بحاجة إلى CLI المستقل الخاص بـ Lobster لأغراض التطوير أو خطوط المعالجة الخارجية، فثبّته من [مستودع Lobster](https://github.com/openclaw/lobster) وتأكد من أن `lobster` موجود على `PATH`.

## تفعيل الأداة

Lobster هو **أداة إضافة اختيارية** (غير مفعلة افتراضيًا).

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

تجنب استخدام `tools.allow: ["lobster"]` ما لم تكن تنوي التشغيل في وضع قائمة السماح المقيدة.

ملاحظة: قوائم السماح اختيارية للإضافات الاختيارية. إذا كانت قائمة السماح لديك تسمّي فقط
أدوات الإضافات (مثل `lobster`)، فإن OpenClaw يبقي الأدوات الأساسية مفعلة. ولتقييد الأدوات الأساسية،
ضمّن الأدوات الأساسية أو المجموعات التي تريدها في قائمة السماح أيضًا.

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

يعيد غلاف JSON (مبتورًا):

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

مسار عمل واحد. حتمي. آمن.

## معاملات الأداة

### `run`

شغّل خط معالجة في وضع الأداة.

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

شغّل ملف مسار عمل مع المعاملات:

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

### `resume`

تابع مسار عمل متوقفًا بعد الموافقة.

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

### المدخلات الاختيارية

- `cwd`: دليل العمل النسبي لخط المعالجة (ويجب أن يبقى ضمن دليل عمل البوابة).
- `timeoutMs`: إيقاف مسار العمل إذا تجاوز هذه المدة (الافتراضي: 20000).
- `maxStdoutBytes`: إيقاف مسار العمل إذا تجاوزت المخرجات هذا الحجم (الافتراضي: 512000).
- `argsJson`: سلسلة JSON تُمرَّر إلى `lobster run --args-json` (لملفات مسارات العمل فقط).

## غلاف المخرجات

يعيد Lobster غلاف JSON بإحدى ثلاث حالات:

- `ok` → انتهى بنجاح
- `needs_approval` → تم الإيقاف مؤقتًا؛ يلزم `requiresApproval.resumeToken` للاستئناف
- `cancelled` → تم الرفض أو الإلغاء صراحةً

تعرض الأداة الغلاف في كلٍّ من `content` ‏(JSON منسّق) و`details` ‏(الكائن الخام).

## الموافقات

إذا كانت `requiresApproval` موجودة، فافحص المطالبة وقرّر:

- `approve: true` → استأنف وتابع التأثيرات الجانبية
- `approve: false` → ألغِ وأنهِ مسار العمل

استخدم `approve --preview-from-stdin --limit N` لإرفاق معاينة JSON بطلبات الموافقة من دون لصق مخصص باستخدام jq/heredoc. أصبحت رموز الاستئناف الآن مضغوطة: يخزّن Lobster حالة استئناف مسار العمل ضمن دليل حالته ويعيد مفتاح رمز صغيرًا.

## OpenProse

يقترن OpenProse جيدًا مع Lobster: استخدم `/prose` لتنسيق التحضير متعدد الوكلاء، ثم شغّل خط معالجة Lobster للحصول على موافقات حتمية. وإذا احتاج برنامج Prose إلى Lobster، فاسمح بأداة `lobster` للوكلاء الفرعيين عبر `tools.subagents.tools`. راجع [OpenProse](/ar/prose).

## الأمان

- **محلي داخل العملية فقط** — تُنفَّذ مسارات العمل داخل عملية البوابة؛ ولا توجد استدعاءات شبكة من الإضافة نفسها.
- **من دون أسرار** — لا يدير Lobster ‏OAuth؛ بل يستدعي أدوات OpenClaw التي تدير ذلك.
- **مدرك لـ sandbox** — يتم تعطيله عندما يكون سياق الأداة داخل sandbox.
- **محصّن** — تُفرَض المهلات وحدود المخرجات بواسطة المشغّل المضمّن.

## استكشاف الأخطاء وإصلاحها

- **`lobster timed out`** → زد `timeoutMs` أو قسّم خط معالجة طويلًا.
- **`lobster output exceeded maxStdoutBytes`** → ارفع `maxStdoutBytes` أو قلّل حجم المخرجات.
- **`lobster returned invalid JSON`** → تأكد من أن خط المعالجة يعمل في وضع الأداة ويطبع JSON فقط.
- **`lobster failed`** → تحقق من سجلات البوابة للحصول على تفاصيل خطأ المشغّل المضمّن.

## تعلّم المزيد

- [الإضافات](/ar/tools/plugin)
- [تأليف أدوات الإضافات](/ar/plugins/building-plugins#registering-agent-tools)

## دراسة حالة: مسارات عمل المجتمع

أحد الأمثلة العامة: CLI لـ “second brain” + خطوط معالجة Lobster تدير ثلاثة مخازن Markdown (شخصي، وشريك، ومشترك). يُخرج CLI بيانات JSON للإحصاءات، وقوائم inbox، وعمليات فحص التقادم؛ ويربط Lobster هذه الأوامر في مسارات عمل مثل `weekly-review` و`inbox-triage` و`memory-consolidation` و`shared-task-sync`، ولكل منها بوابات موافقة. ويتولى الذكاء الاصطناعي إصدار الأحكام (التصنيف) عند توفره ويعود إلى قواعد حتمية عند عدم توفره.

- النقاش: [https://x.com/plattenschieber/status/2014508656335770033](https://x.com/plattenschieber/status/2014508656335770033)
- المستودع: [https://github.com/bloomedai/brain-cli](https://github.com/bloomedai/brain-cli)

## ذو صلة

- [Automation & Tasks](/ar/automation) — جدولة مسارات عمل Lobster
- [نظرة عامة على الأتمتة](/ar/automation) — جميع آليات الأتمتة
- [نظرة عامة على الأدوات](/ar/tools) — جميع أدوات الوكيل المتاحة
