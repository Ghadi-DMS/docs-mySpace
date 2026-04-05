---
read_when:
    - تريد نماذج Z.AI / GLM في OpenClaw
    - تحتاج إلى إعداد بسيط باستخدام ZAI_API_KEY
summary: استخدم Z.AI (نماذج GLM) مع OpenClaw
title: Z.AI
x-i18n:
    generated_at: "2026-04-05T12:54:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 48006cdd580484f0c62e2877b27a6a68d7bc44795b3e97a28213d95182d9acf9
    source_path: providers/zai.md
    workflow: 15
---

# Z.AI

Z.AI هي منصة API الخاصة بنماذج **GLM**. وهي توفر واجهات REST API لنماذج GLM وتستخدم مفاتيح API
للمصادقة. أنشئ مفتاح API الخاص بك من لوحة Z.AI. ويستخدم OpenClaw المزوّد `zai`
مع مفتاح API من Z.AI.

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

يسمح `zai-api-key` لـ OpenClaw باكتشاف نقطة النهاية المطابقة في Z.AI من المفتاح و
تطبيق `base URL` الصحيح تلقائيًا. استخدم الخيارات الإقليمية الصريحة عندما
تريد فرض سطح Coding Plan أو General API محدد.

## فهرس GLM المضمن

يقوم OpenClaw حاليًا بتهيئة المزوّد المضمن `zai` بما يلي:

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

- تتوفر نماذج GLM بصيغة `zai/<model>` (مثال: `zai/glm-5`).
- مرجع النموذج المضمن الافتراضي: `zai/glm-5`
- ما زالت معرّفات `glm-5*` غير المعروفة تُحل مستقبلًا على مسار المزوّد المضمن
  عبر توليد بيانات وصفية يملكها المزوّد من القالب `glm-4.7` عندما يطابق المعرّف
  شكل عائلة GLM-5 الحالية.
- يتم تمكين `tool_stream` افتراضيًا لبث استدعاءات الأدوات في Z.AI. اضبط
  `agents.defaults.models["zai/<model>"].params.tool_stream` على `false` لتعطيله.
- راجع [/providers/glm](/providers/glm) للحصول على نظرة عامة على عائلة النماذج.
- تستخدم Z.AI مصادقة Bearer باستخدام مفتاح API الخاص بك.
