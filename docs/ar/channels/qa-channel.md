---
read_when:
    - أنت تقوم بتوصيل النقل الاصطناعي لـ QA ضمن تشغيل اختبار محلي أو في CI
    - أنت بحاجة إلى سطح إعدادات `qa-channel` المضمّن
    - أنت تعمل بشكل تكراري على أتمتة QA الشاملة من البداية إلى النهاية
summary: إضافة قناة اصطناعية من فئة Slack لسيناريوهات QA الحتمية في OpenClaw
title: قناة QA
x-i18n:
    generated_at: "2026-04-06T03:05:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b88cd73df2f61b34ad1eb83c3450f8fe15a51ac69fbb5a9eca0097564d67a06
    source_path: channels/qa-channel.md
    workflow: 15
---

# قناة QA

`qa-channel` هو ناقل رسائل اصطناعي مضمّن لاختبارات QA المؤتمتة في OpenClaw.

إنها ليست قناة إنتاجية. وهي موجودة لاختبار نفس حدود إضافة القناة
المستخدمة بواسطة وسائل النقل الحقيقية، مع إبقاء الحالة حتمية وقابلة
للفحص بالكامل.

## ما الذي تفعله اليوم

- صياغة أهداف من فئة Slack:
  - `dm:<user>`
  - `channel:<room>`
  - `thread:<room>/<thread>`
- ناقل اصطناعي قائم على HTTP من أجل:
  - حقن الرسائل الواردة
  - التقاط النصوص الصادرة
  - إنشاء سلاسل الرسائل
  - التفاعلات
  - التعديلات
  - عمليات الحذف
  - إجراءات البحث والقراءة
- مشغّل فحص ذاتي مضمّن على جهة المضيف يكتب تقريرًا بصيغة Markdown

## الإعدادات

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

مفاتيح الحساب المدعومة:

- `baseUrl`
- `botUserId`
- `botDisplayName`
- `pollTimeoutMs`
- `allowFrom`
- `defaultTo`
- `actions.messages`
- `actions.reactions`
- `actions.search`
- `actions.threads`

## المشغّل

المقطع العمودي الحالي:

```bash
pnpm qa:e2e
```

يتم الآن التوجيه عبر امتداد `qa-lab` المضمّن. فهو يبدأ
ناقل QA داخل المستودع، ويشغّل شريحة وقت التشغيل المضمّنة `qa-channel`، وينفّذ
فحصًا ذاتيًا حتميًا، ويكتب تقريرًا بصيغة Markdown ضمن `.artifacts/qa-e2e/`.

واجهة المستخدم الخاصة بمصحح الأخطاء:

```bash
pnpm qa:lab:build
pnpm openclaw qa ui
```

حزمة QA الكاملة المدعومة بالمستودع:

```bash
pnpm openclaw qa suite
```

يؤدي ذلك إلى تشغيل مصحح QA الخاص على عنوان URL محلي، بشكل منفصل عن
حزمة Control UI المشحونة.

## النطاق

النطاق الحالي ضيق عمدًا:

- الناقل + نقل الإضافة
- صياغة التوجيه المعتمد على سلاسل الرسائل
- إجراءات الرسائل المملوكة للقناة
- تقارير Markdown

ستضيف الأعمال اللاحقة:

- تنسيق OpenClaw عبر Docker
- تنفيذ مصفوفة المزوّدين/النماذج
- اكتشافًا أكثر ثراءً للسيناريوهات
- تنسيقًا أصليًا لـ OpenClaw لاحقًا
