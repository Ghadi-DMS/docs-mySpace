---
read_when:
    - لديك مشكلات في الاتصال/المصادقة وتريد إصلاحات موجهة
    - قمت بالتحديث وتريد فحصًا سريعًا للتأكد
summary: مرجع CLI للأمر `openclaw doctor` ‏(فحوصات السلامة + إصلاحات موجهة)
title: doctor
x-i18n:
    generated_at: "2026-04-05T12:38:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: d257a9e2797b4b0b50c1020165c8a1cd6a2342381bf9c351645ca37494c881e1
    source_path: cli/doctor.md
    workflow: 15
---

# `openclaw doctor`

فحوصات السلامة + إصلاحات سريعة لـ gateway والقنوات.

ذو صلة:

- استكشاف الأخطاء وإصلاحها: [استكشاف الأخطاء وإصلاحها](/gateway/troubleshooting)
- التدقيق الأمني: [الأمان](/gateway/security)

## أمثلة

```bash
openclaw doctor
openclaw doctor --repair
openclaw doctor --deep
openclaw doctor --repair --non-interactive
openclaw doctor --generate-gateway-token
```

## الخيارات

- `--no-workspace-suggestions`: تعطيل اقتراحات ذاكرة/بحث مساحة العمل
- `--yes`: قبول القيم الافتراضية من دون مطالبة
- `--repair`: تطبيق الإصلاحات الموصى بها من دون مطالبة
- `--fix`: اسم مستعار لـ `--repair`
- `--force`: تطبيق إصلاحات صارمة، بما في ذلك الكتابة فوق تكوين الخدمة المخصص عند الحاجة
- `--non-interactive`: التشغيل من دون مطالبات؛ عمليات ترحيل آمنة فقط
- `--generate-gateway-token`: إنشاء رمز gateway وتكوينه
- `--deep`: فحص خدمات النظام بحثًا عن تثبيتات gateway إضافية

ملاحظات:

- لا تعمل المطالبات التفاعلية (مثل إصلاحات keychain/OAuth) إلا عندما يكون stdin عبارة عن TTY ولا يكون `--non-interactive` مضبوطًا. تتخطى عمليات التشغيل بدون واجهة (cron أو Telegram أو بدون طرفية) المطالبات.
- يقوم `--fix` (الاسم المستعار لـ `--repair`) بكتابة نسخة احتياطية إلى `~/.openclaw/openclaw.json.bak` ويحذف مفاتيح التكوين غير المعروفة مع إدراج كل عملية إزالة.
- تكتشف فحوصات سلامة الحالة الآن ملفات النصوص اليتيمة في دليل الجلسات ويمكنها أرشفتها بصيغة `.deleted.<timestamp>` لاستعادة المساحة بأمان.
- يفحص Doctor أيضًا `~/.openclaw/cron/jobs.json` (أو `cron.store`) بحثًا عن أشكال مهام cron القديمة ويمكنه إعادة كتابتها في مكانها قبل أن يضطر المجدول إلى تطبيعها تلقائيًا في وقت التشغيل.
- يقوم Doctor بترحيل تكوين Talk المسطح القديم تلقائيًا (`talk.voiceId` و`talk.modelId` وما شابه) إلى `talk.provider` + `talk.providers.<provider>`.
- لم تعد عمليات التشغيل المتكررة لـ `doctor --fix` تُبلغ عن/تطبق تطبيع Talk عندما يكون الاختلاف الوحيد هو ترتيب مفاتيح الكائن.
- يتضمن Doctor فحصًا لجهوزية البحث في الذاكرة ويمكنه التوصية باستخدام `openclaw configure --section model` عندما تكون بيانات اعتماد embedding مفقودة.
- إذا كان وضع sandbox ممكّنًا ولكن Docker غير متاح، فإن Doctor يبلغ عن تحذير عالي الإشارة مع المعالجة (`install Docker` أو `openclaw config set agents.defaults.sandbox.mode off`).
- إذا كانت `gateway.auth.token`/`gateway.auth.password` مُدارة عبر SecretRef وغير متاحة في مسار الأمر الحالي، فإن Doctor يبلغ عن تحذير للقراءة فقط ولا يكتب بيانات اعتماد احتياطية بنص واضح.
- إذا فشل فحص SecretRef الخاص بالقناة في مسار الإصلاح، يواصل Doctor العمل ويبلغ عن تحذير بدلًا من الخروج المبكر.
- يتطلب الحل التلقائي لأسماء المستخدمين في Telegram `allowFrom` ‏(`doctor --fix`) رمز Telegram قابلًا للحل في مسار الأمر الحالي. إذا لم يكن فحص الرمز متاحًا، يبلغ Doctor عن تحذير ويتخطى الحل التلقائي في هذه الجولة.

## macOS: تجاوزات بيئة `launchctl`

إذا كنت قد شغّلت سابقًا `launchctl setenv OPENCLAW_GATEWAY_TOKEN ...` (أو `...PASSWORD`)، فإن هذه القيمة تتجاوز ملف التكوين الخاص بك ويمكن أن تسبب أخطاء "unauthorized" مستمرة.

```bash
launchctl getenv OPENCLAW_GATEWAY_TOKEN
launchctl getenv OPENCLAW_GATEWAY_PASSWORD

launchctl unsetenv OPENCLAW_GATEWAY_TOKEN
launchctl unsetenv OPENCLAW_GATEWAY_PASSWORD
```
