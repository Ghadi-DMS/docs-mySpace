---
read_when:
    - تريد نماذج GLM في OpenClaw
    - تحتاج إلى اصطلاح تسمية النماذج والإعداد
summary: نظرة عامة على عائلة نماذج GLM + كيفية استخدامها في OpenClaw
title: نماذج GLM
x-i18n:
    generated_at: "2026-04-05T12:53:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 59622edab5094d991987f9788fbf08b33325e737e7ff88632b0c3ac89412d4c7
    source_path: providers/glm.md
    workflow: 15
---

# نماذج GLM

GLM هي **عائلة نماذج** (وليست شركة) متاحة عبر منصة Z.AI. وفي OpenClaw، يتم الوصول إلى
نماذج GLM عبر المزوّد `zai` ومعرّفات نماذج مثل `zai/glm-5`.

## إعداد CLI

```bash
# إعداد عام باستخدام مفتاح API مع اكتشاف تلقائي لنقطة النهاية
openclaw onboard --auth-choice zai-api-key

# Coding Plan Global، موصى به لمستخدمي Coding Plan
openclaw onboard --auth-choice zai-coding-global

# Coding Plan CN (منطقة الصين)، موصى به لمستخدمي Coding Plan
openclaw onboard --auth-choice zai-coding-cn

# General API
openclaw onboard --auth-choice zai-global

# General API CN (منطقة الصين)
openclaw onboard --auth-choice zai-cn
```

## مقتطف التكوين

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "zai/glm-5" } } },
}
```

يسمح `zai-api-key` لـ OpenClaw باكتشاف نقطة النهاية المطابقة في Z.AI من المفتاح
وتطبيق `base URL` الصحيح تلقائيًا. استخدم الخيارات الإقليمية الصريحة عندما
تريد فرض سطح Coding Plan أو General API محدد.

## نماذج GLM المضمنة الحالية

يقوم OpenClaw حاليًا بتهيئة المزوّد المضمن `zai` بمراجع GLM التالية:

- `glm-5.1`
- `glm-5`
- `glm-5-turbo`
- `glm-5v-turbo`
- `glm-4.7`
- `glm-4.7-flash`
- `glm-4.7-flashx`
- `glm-4.6`
- `glm-4.6v`
- `glm-4.5`
- `glm-4.5-air`
- `glm-4.5-flash`
- `glm-4.5v`

## ملاحظات

- قد تتغير إصدارات GLM ومدى توفرها؛ راجع مستندات Z.AI لمعرفة الأحدث.
- مرجع النموذج المضمن الافتراضي هو `zai/glm-5`.
- للحصول على تفاصيل المزوّد، راجع [/providers/zai](/providers/zai).
