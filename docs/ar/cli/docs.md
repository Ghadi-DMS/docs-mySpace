---
read_when:
    - تريد البحث في مستندات OpenClaw الحية من الطرفية
summary: مرجع CLI للأمر `openclaw docs` (البحث في فهرس المستندات الحي)
title: docs
x-i18n:
    generated_at: "2026-04-05T12:38:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: cfcceed872d7509b9843af3fae733a136bc5e26ded55c2ac47a16489a1636989
    source_path: cli/docs.md
    workflow: 15
---

# `openclaw docs`

ابحث في فهرس المستندات الحي.

الوسائط:

- `[query...]`: مصطلحات البحث التي سيتم إرسالها إلى فهرس المستندات الحي

أمثلة:

```bash
openclaw docs
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

ملاحظات:

- عند عدم تمرير أي استعلام، يفتح `openclaw docs` نقطة دخول البحث في المستندات الحية.
- يتم تمرير الاستعلامات متعددة الكلمات كطلب بحث واحد.
