---
read_when:
    - أنت تريد التحقق سريعًا من صحة البوابة قيد التشغيل
summary: مرجع CLI للأمر `openclaw health` (لقطة لصحة البوابة عبر RPC)
title: health
x-i18n:
    generated_at: "2026-04-05T12:38:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4ed2b9ceefee6159cabaae9172d2d88174626456e7503d5d2bcd142634188ff0
    source_path: cli/health.md
    workflow: 15
---

# `openclaw health`

اجلب معلومات الصحة من البوابة قيد التشغيل.

الخيارات:

- `--json`: مخرجات قابلة للقراءة آليًا
- `--timeout <ms>`: مهلة الاتصال بالمللي ثانية (الافتراضي `10000`)
- `--verbose`: تسجيل مفصل
- `--debug`: اسم بديل لـ `--verbose`

أمثلة:

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

ملاحظات:

- يطلب `openclaw health` الافتراضي لقطة الصحة من البوابة قيد التشغيل. عندما
  تكون لدى البوابة بالفعل لقطة مخزنة مؤقتًا وحديثة، يمكنها إرجاع تلك الحمولة المخزنة مؤقتًا
  والتحديث في الخلفية.
- يفرض `--verbose` إجراء فحص مباشر، ويطبع تفاصيل اتصال البوابة، ويوسّع
  المخرجات المقروءة للبشر لتشمل جميع الحسابات والوكلاء المهيئين.
- تتضمن المخرجات مخازن جلسات لكل وكيل عند تهيئة عدة وكلاء.
