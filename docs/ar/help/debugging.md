---
read_when:
    - تحتاج إلى فحص مخرجات النموذج الخام لتسرّب الاستدلال
    - تريد تشغيل Gateway في وضع المراقبة أثناء التكرار
    - تحتاج إلى سير عمل قابل للتكرار لتصحيح الأخطاء
summary: 'أدوات تصحيح الأخطاء: وضع المراقبة، وتدفقات النموذج الخام، وتتبع تسرّب الاستدلال'
title: تصحيح الأخطاء
x-i18n:
    generated_at: "2026-04-06T03:07:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4bc72e8d6cad3a1acaad066f381c82309583fabf304c589e63885f2685dc704e
    source_path: help/debugging.md
    workflow: 15
---

# تصحيح الأخطاء

تغطي هذه الصفحة أدوات مساعدة لتصحيح الأخطاء لمخرجات البث، خاصة عندما
يمزج مزوّد ما الاستدلال داخل النص العادي.

## تجاوزات تصحيح الأخطاء في وقت التشغيل

استخدم `/debug` في الدردشة لتعيين تجاوزات config **خاصة بوقت التشغيل فقط** (في الذاكرة، لا على القرص).
يكون `/debug` معطّلًا افتراضيًا؛ فعّله باستخدام `commands.debug: true`.
وهذا مفيد عندما تحتاج إلى تبديل إعدادات غير شائعة من دون تعديل `openclaw.json`.

أمثلة:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug unset messages.responsePrefix
/debug reset
```

يقوم `/debug reset` بمسح جميع التجاوزات والعودة إلى config الموجود على القرص.

## وضع المراقبة لـ Gateway

للتكرار السريع، شغّل البوابة تحت مراقب الملفات:

```bash
pnpm gateway:watch
```

ويُطابِق هذا:

```bash
node scripts/watch-node.mjs gateway --force
```

يعيد المراقب التشغيل عند الملفات ذات الصلة بالبناء ضمن `src/`، وملفات مصدر الإضافات،
و`package.json` الخاص بالإضافات وبيانات `openclaw.plugin.json` الوصفية، و`tsconfig.json`،
و`package.json`، و`tsdown.config.ts`. تؤدي تغييرات البيانات الوصفية للإضافات إلى إعادة تشغيل
البوابة من دون فرض إعادة بناء `tsdown`؛ أما تغييرات المصدر وconfig فلا تزال
تعيد بناء `dist` أولًا.

أضف أي رايات CLI للبوابة بعد `gateway:watch` وسيتم تمريرها
في كل إعادة تشغيل. كما أن إعادة تشغيل أمر المراقبة نفسه للمستودع/مجموعة الرايات نفسها
تستبدل الآن المراقب الأقدم بدلًا من ترك عمليات أصل مكررة للمراقب.

## ملف التطوير + بوابة التطوير (`--dev`)

استخدم ملف التطوير لعزل الحالة وتشغيل إعداد آمن وقابل للتخلص منه من أجل
تصحيح الأخطاء. هناك رايتا `--dev` **اثنتان**:

- **`--dev` العمومي (الملف):** يعزل الحالة تحت `~/.openclaw-dev` ويضبط
  منفذ البوابة افتراضيًا على `19001` (وتتحول المنافذ المشتقة معه).
- **`gateway --dev`:** يوجّه Gateway إلى إنشاء config افتراضي +
  مساحة عمل تلقائيًا عند غيابهما (وتخطي `BOOTSTRAP.md`).

التدفق الموصى به (ملف تطوير + تهيئة تطوير):

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

إذا لم يكن لديك تثبيت عمومي بعد، فشغّل CLI عبر `pnpm openclaw ...`.

ما الذي يفعله هذا:

1. **عزل الملف** (`--dev` العمومي)
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001` (وتتحول منافذ المتصفح/canvas وفقًا لذلك)

2. **تهيئة التطوير** (`gateway --dev`)
   - يكتب config أدنى عند غيابه (`gateway.mode=local`، وربط loopback).
   - يضبط `agent.workspace` إلى مساحة عمل التطوير.
   - يضبط `agent.skipBootstrap=true` (من دون `BOOTSTRAP.md`).
   - يملأ ملفات مساحة العمل عند غيابها:
     `AGENTS.md` و`SOUL.md` و`TOOLS.md` و`IDENTITY.md` و`USER.md` و`HEARTBEAT.md`.
   - الهوية الافتراضية: **C3‑PO** (droid بروتوكولي).
   - يتخطى مزوّدي القنوات في وضع التطوير (`OPENCLAW_SKIP_CHANNELS=1`).

تدفق إعادة الضبط (بداية جديدة):

```bash
pnpm gateway:dev:reset
```

ملاحظة: `--dev` هي راية ملف **عمومية** وتلتهمها بعض المشغلات.
إذا احتجت إلى كتابتها صراحة، فاستخدم صيغة متغير البيئة:

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

يقوم `--reset` بمسح config وبيانات الاعتماد والجلسات ومساحة عمل التطوير (باستخدام
`trash` لا `rm`) ثم يعيد إنشاء إعداد التطوير الافتراضي.

نصيحة: إذا كانت بوابة غير مخصّصة للتطوير تعمل بالفعل (عبر launchd/systemd)، فأوقفها أولًا:

```bash
openclaw gateway stop
```

## تسجيل التدفق الخام (OpenClaw)

يمكن لـ OpenClaw تسجيل **تدفق المساعد الخام** قبل أي ترشيح/تنسيق.
وهذه أفضل طريقة لمعرفة ما إذا كان الاستدلال يصل على شكل دلتا نصية عادية
(أو ككتل تفكير منفصلة).

فعّله عبر CLI:

```bash
pnpm gateway:watch --raw-stream
```

تجاوز اختياري للمسار:

```bash
pnpm gateway:watch --raw-stream --raw-stream-path ~/.openclaw/logs/raw-stream.jsonl
```

متغيرات البيئة المكافئة:

```bash
OPENCLAW_RAW_STREAM=1
OPENCLAW_RAW_STREAM_PATH=~/.openclaw/logs/raw-stream.jsonl
```

الملف الافتراضي:

`~/.openclaw/logs/raw-stream.jsonl`

## تسجيل الأجزاء الخام (pi-mono)

لالتقاط **أجزاء OpenAI-compat الخام** قبل تحليلها إلى كتل،
يكشف pi-mono مسجلًا منفصلًا:

```bash
PI_RAW_STREAM=1
```

مسار اختياري:

```bash
PI_RAW_STREAM_PATH=~/.pi-mono/logs/raw-openai-completions.jsonl
```

الملف الافتراضي:

`~/.pi-mono/logs/raw-openai-completions.jsonl`

> ملاحظة: لا يُصدَر هذا إلا من العمليات التي تستخدم
> مزوّد `openai-completions` الخاص بـ pi-mono.

## ملاحظات السلامة

- قد تتضمن سجلات التدفق الخام prompts كاملة، ومخرجات الأدوات، وبيانات المستخدم.
- احتفظ بالسجلات محليًا واحذفها بعد تصحيح الأخطاء.
- إذا شاركت السجلات، فنقِّح الأسرار ومعلومات PII أولًا.
