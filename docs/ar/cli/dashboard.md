---
read_when:
    - أنت تريد فتح واجهة Control UI باستخدام الرمز المميز الحالي الخاص بك
    - أنت تريد طباعة عنوان URL دون تشغيل المتصفح
summary: مرجع CLI للأمر `openclaw dashboard` (فتح واجهة Control)
title: dashboard
x-i18n:
    generated_at: "2026-04-05T12:38:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: a34cd109a3803e2910fcb4d32f2588aa205a4933819829ef5598f0780f586c94
    source_path: cli/dashboard.md
    workflow: 15
---

# `openclaw dashboard`

افتح واجهة Control UI باستخدام المصادقة الحالية الخاصة بك.

```bash
openclaw dashboard
openclaw dashboard --no-open
```

ملاحظات:

- يقوم `dashboard` بحل SecretRefs المُعدّة في `gateway.auth.token` عندما يكون ذلك ممكنًا.
- بالنسبة إلى الرموز المميزة المُدارة عبر SecretRef (المحلولة أو غير المحلولة)، يقوم `dashboard` بطباعة/نسخ/فتح عنوان URL غير مضمّن فيه الرمز المميز لتجنب كشف الأسرار الخارجية في مخرجات الطرفية أو سجل الحافظة أو معاملات تشغيل المتصفح.
- إذا كان `gateway.auth.token` مُدارًا عبر SecretRef لكنه غير محلول في مسار هذا الأمر، فسيطبع الأمر عنوان URL غير مضمّن فيه الرمز المميز مع إرشادات معالجة واضحة بدلًا من تضمين عنصر نائب غير صالح للرمز المميز.
