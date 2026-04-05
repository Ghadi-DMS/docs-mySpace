---
read_when:
    - تشغيل مضيف العقدة headless
    - إقران عقدة غير macOS لأجل system.run
summary: مرجع CLI للأمر `openclaw node` (مضيف العقدة headless)
title: node
x-i18n:
    generated_at: "2026-04-05T12:38:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6123b33ec46f2b85f2c815947435ac91bbe84456165ff0e504453356da55b46d
    source_path: cli/node.md
    workflow: 15
---

# `openclaw node`

شغّل **مضيف عقدة headless** يتصل بـ Gateway WebSocket ويكشف
`system.run` / `system.which` على هذا الجهاز.

## لماذا تستخدم مضيف عقدة؟

استخدم مضيف عقدة عندما تريد أن تقوم الوكلاء **بتشغيل الأوامر على أجهزة أخرى** في
شبكتك من دون تثبيت تطبيق macOS مصاحب كامل هناك.

حالات الاستخدام الشائعة:

- تشغيل الأوامر على أجهزة Linux/Windows بعيدة (خوادم بناء، أجهزة مختبر، NAS).
- إبقاء exec **ضمن صندوق حماية** على البوابة، مع تفويض عمليات التشغيل المعتمدة إلى مضيفين آخرين.
- توفير هدف تنفيذ خفيف وheadless لعمليات الأتمتة أو عقد CI.

يظل التنفيذ محميًا بواسطة **موافقات exec** وقوائم السماح لكل وكيل على
مضيف العقدة، بحيث يمكنك إبقاء الوصول إلى الأوامر محدودًا وصريحًا.

## وكيل المتصفح (بدون إعداد)

تعلن مضيفات العقد تلقائيًا عن وكيل متصفح إذا لم يتم
تعطيل `browser.enabled` على العقدة. يتيح هذا للوكيل استخدام أتمتة المتصفح على تلك العقدة
من دون إعداد إضافي.

بشكل افتراضي، يكشف الوكيل عن سطح ملف تعريف المتصفح العادي للعقدة. وإذا
ضبطت `nodeHost.browserProxy.allowProfiles`، يصبح الوكيل مقيِّدًا:
يتم رفض استهداف ملفات التعريف غير الموجودة في قائمة السماح، كما تُحظر
مسارات إنشاء/حذف ملفات التعريف الدائمة عبر الوكيل.

عطّله على العقدة عند الحاجة:

```json5
{
  nodeHost: {
    browserProxy: {
      enabled: false,
    },
  },
}
```

## التشغيل (الواجهة الأمامية)

```bash
openclaw node run --host <gateway-host> --port 18789
```

الخيارات:

- `--host <host>`: مضيف Gateway WebSocket (الافتراضي: `127.0.0.1`)
- `--port <port>`: منفذ Gateway WebSocket (الافتراضي: `18789`)
- `--tls`: استخدام TLS لاتصال البوابة
- `--tls-fingerprint <sha256>`: بصمة شهادة TLS المتوقعة (`sha256`)
- `--node-id <id>`: تجاوز معرّف العقدة (يمسح رمز الاقتران)
- `--display-name <name>`: تجاوز الاسم المعروض للعقدة

## مصادقة البوابة لمضيف العقدة

يقوم `openclaw node run` و`openclaw node install` بتحليل مصادقة البوابة من الإعداد/البيئة (لا توجد علامات `--token`/`--password` على أوامر node):

- يتم التحقق أولًا من `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`.
- ثم الرجوع إلى الإعداد المحلي: `gateway.auth.token` / `gateway.auth.password`.
- في الوضع المحلي، لا يرث مضيف العقدة عمدًا `gateway.remote.token` / `gateway.remote.password`.
- إذا تم إعداد `gateway.auth.token` / `gateway.auth.password` صراحةً عبر SecretRef وكان غير محلول، يفشل تحليل مصادقة العقدة بشكل مغلق (من دون رجوع بعيد يخفي المشكلة).
- في `gateway.mode=remote`، تكون حقول العميل البعيد (`gateway.remote.token` / `gateway.remote.password`) مؤهلة أيضًا وفق قواعد أسبقية الوضع البعيد.
- لا يحترم تحليل مصادقة مضيف العقدة إلا متغيرات البيئة `OPENCLAW_GATEWAY_*`.

## الخدمة (الخلفية)

ثبّت مضيف عقدة headless كخدمة مستخدم.

```bash
openclaw node install --host <gateway-host> --port 18789
```

الخيارات:

- `--host <host>`: مضيف Gateway WebSocket (الافتراضي: `127.0.0.1`)
- `--port <port>`: منفذ Gateway WebSocket (الافتراضي: `18789`)
- `--tls`: استخدام TLS لاتصال البوابة
- `--tls-fingerprint <sha256>`: بصمة شهادة TLS المتوقعة (`sha256`)
- `--node-id <id>`: تجاوز معرّف العقدة (يمسح رمز الاقتران)
- `--display-name <name>`: تجاوز الاسم المعروض للعقدة
- `--runtime <runtime>`: بيئة تشغيل الخدمة (`node` أو `bun`)
- `--force`: إعادة التثبيت/الاستبدال إذا كان مثبتًا بالفعل

إدارة الخدمة:

```bash
openclaw node status
openclaw node stop
openclaw node restart
openclaw node uninstall
```

استخدم `openclaw node run` للحصول على مضيف عقدة في الواجهة الأمامية (من دون خدمة).

تقبل أوامر الخدمة الخيار `--json` لإخراج قابل للقراءة آليًا.

## الاقتران

ينشئ أول اتصال طلب اقتران جهاز معلقًا (`role: node`) على البوابة.
وافق عليه عبر:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

إذا أعادت العقدة محاولة الاقتران مع تفاصيل مصادقة متغيرة (دور/نطاقات/مفتاح عام)،
فسيتم استبدال الطلب المعلق السابق وإنشاء `requestId` جديد.
شغّل `openclaw devices list` مرة أخرى قبل الموافقة.

يخزّن مضيف العقدة معرّف العقدة، والرمز، والاسم المعروض، ومعلومات اتصال البوابة في
`~/.openclaw/node.json`.

## موافقات exec

يتم تقييد `system.run` بواسطة موافقات exec المحلية:

- `~/.openclaw/exec-approvals.json`
- [موافقات Exec](/tools/exec-approvals)
- `openclaw approvals --node <id|name|ip>` (للتعديل من البوابة)

بالنسبة إلى تنفيذ العقدة غير المتزامن المعتمد، يقوم OpenClaw بإعداد `systemRunPlan`
قياسي قبل المطالبة. ثم تعيد عملية إعادة توجيه `system.run` المعتمدة لاحقًا استخدام تلك
الخطة المخزنة، لذلك يتم رفض التعديلات على حقول command/cwd/session بعد إنشاء
طلب الموافقة بدلًا من تغيير ما ستنفذه العقدة.
