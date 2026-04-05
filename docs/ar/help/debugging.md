---
read_when:
    - تحتاج إلى فحص مخرجات النموذج الخام لتسرب reasoning
    - تريد تشغيل Gateway في وضع المراقبة أثناء التكرار
    - تحتاج إلى سير عمل قابل للتكرار لتصحيح الأخطاء
summary: 'أدوات تصحيح الأخطاء: وضع المراقبة، وتدفقات النموذج الخام، وتتبع تسرب reasoning'
title: تصحيح الأخطاء
x-i18n:
    generated_at: "2026-04-05T12:45:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: f90d944ecc2e846ca0b26a162126ceefb3a3c6cf065c99b731359ec79d4289e3
    source_path: help/debugging.md
    workflow: 15
---

# تصحيح الأخطاء

تغطي هذه الصفحة وسائل المساعدة الخاصة بتصحيح أخطاء المخرجات المتدفقة، خاصة عندما
يمزج موفّر ما reasoning داخل النص العادي.

## تجاوزات تصحيح الأخطاء في وقت التشغيل

استخدم `/debug` في الدردشة لتعيين تجاوزات إعدادات **خاصة بوقت التشغيل فقط** (في الذاكرة، وليس على القرص).
يكون `/debug` معطلًا افتراضيًا؛ فعّله عبر `commands.debug: true`.
وهذا مفيد عندما تحتاج إلى تبديل إعدادات غير شائعة دون تعديل `openclaw.json`.

أمثلة:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug unset messages.responsePrefix
/debug reset
```

يقوم `/debug reset` بمسح كل التجاوزات والعودة إلى الإعدادات الموجودة على القرص.

## وضع المراقبة في Gateway

من أجل التكرار السريع، شغّل البوابة تحت مراقب الملفات:

```bash
pnpm gateway:watch
```

ويطابق هذا:

```bash
node scripts/watch-node.mjs gateway --force
```

يعيد المراقب التشغيل عند الملفات ذات الصلة بالبنية ضمن `src/`، وملفات مصدر الإضافات،
و`package.json` الخاص بالإضافة وبيانات `openclaw.plugin.json` الوصفية، و`tsconfig.json`،
و`package.json`، و`tsdown.config.ts`. تؤدي تغييرات البيانات الوصفية للإضافة إلى إعادة تشغيل
البوابة من دون فرض إعادة بناء `tsdown`؛ بينما تعيد تغييرات المصدر والإعدادات
بناء `dist` أولًا.

أضف أي علامات CLI خاصة بالبوابة بعد `gateway:watch` وسيتم تمريرها
في كل إعادة تشغيل.

## ملف تعريف التطوير + بوابة التطوير (--dev)

استخدم ملف تعريف التطوير لعزل الحالة وتشغيل إعداد آمن ومؤقت من أجل
تصحيح الأخطاء. توجد علامتا `--dev` **اثنتان**:

- **`--dev` العامة (ملف التعريف):** تعزل الحالة تحت `~/.openclaw-dev` وتضبط
  منفذ البوابة افتراضيًا على `19001` (وتتغير المنافذ المشتقة تبعًا لذلك).
- **`gateway --dev`:** يطلب من Gateway إنشاء إعداد افتراضي + مساحة عمل تلقائيًا
  عند غيابهما (وتخطي `BOOTSTRAP.md`).

التدفق الموصى به (ملف تعريف التطوير + bootstrap التطوير):

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

إذا لم يكن لديك تثبيت عام بعد، فشغّل CLI عبر `pnpm openclaw ...`.

ما الذي يفعله هذا:

1. **عزل ملف التعريف** (`--dev` العامة)
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001` (ويتغير browser/canvas وفقًا لذلك)

2. **bootstrap التطوير** (`gateway --dev`)
   - يكتب إعدادًا أدنى إذا كان مفقودًا (`gateway.mode=local`، وbind على loopback).
   - يعيّن `agent.workspace` إلى مساحة عمل التطوير.
   - يعيّن `agent.skipBootstrap=true` (من دون `BOOTSTRAP.md`).
   - يملأ ملفات مساحة العمل إذا كانت مفقودة:
     `AGENTS.md` و`SOUL.md` و`TOOLS.md` و`IDENTITY.md` و`USER.md` و`HEARTBEAT.md`.
   - الهوية الافتراضية: **C3‑PO** (روبوت البروتوكول).
   - يتخطى موفّري القنوات في وضع التطوير (`OPENCLAW_SKIP_CHANNELS=1`).

تدفق إعادة التعيين (بداية جديدة):

```bash
pnpm gateway:dev:reset
```

ملاحظة: `--dev` هي علامة ملف تعريف **عامة** وقد تلتهمها بعض أدوات التشغيل.
إذا احتجت إلى كتابتها صراحة، فاستخدم صيغة متغير البيئة:

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

يقوم `--reset` بمسح الإعدادات، وبيانات الاعتماد، والجلسات، ومساحة عمل التطوير (باستخدام
`trash` وليس `rm`)، ثم يعيد إنشاء إعداد التطوير الافتراضي.

نصيحة: إذا كانت هناك بوابة غير مخصصة للتطوير تعمل بالفعل (launchd/systemd)، فأوقفها أولًا:

```bash
openclaw gateway stop
```

## تسجيل التدفق الخام (OpenClaw)

يمكن لـ OpenClaw تسجيل **تدفق المساعد الخام** قبل أي تصفية/تنسيق.
وهذه هي أفضل طريقة لمعرفة ما إذا كان reasoning يصل كفروق نصية عادية
(أو ككتل thinking منفصلة).

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

> ملاحظة: لا يتم إصدار هذا إلا من العمليات التي تستخدم
> الموفّر `openai-completions` الخاص بـ pi-mono.

## ملاحظات الأمان

- قد تتضمن سجلات التدفق الخام prompts كاملة، ومخرجات الأدوات، وبيانات المستخدم.
- احتفظ بالسجلات محليًا واحذفها بعد تصحيح الأخطاء.
- إذا شاركت السجلات، فاحذف الأسرار وPII أولًا.
