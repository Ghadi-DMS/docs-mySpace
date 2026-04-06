---
read_when:
    - أنت تريد استخدام توليد الفيديو عبر Runway في OpenClaw
    - أنت بحاجة إلى إعداد مفتاح API / متغيرات البيئة لـ Runway
    - أنت تريد جعل Runway موفّر الفيديو الافتراضي
summary: إعداد توليد الفيديو عبر Runway في OpenClaw
title: Runway
x-i18n:
    generated_at: "2026-04-06T03:11:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: bc615d1a26f7a4b890d29461e756690c858ecb05024cf3c4d508218022da6e76
    source_path: providers/runway.md
    workflow: 15
---

# Runway

يشحن OpenClaw موفّرًا مضمّنًا باسم `runway` لتوليد الفيديو المستضاف.

- معرّف الموفّر: `runway`
- المصادقة: `RUNWAYML_API_SECRET` (القياسي) أو `RUNWAY_API_KEY`
- API: توليد الفيديو المعتمد على المهام في Runway ‏(`GET /v1/tasks/{id}` polling)

## البدء السريع

1. عيّن مفتاح API:

```bash
openclaw onboard --auth-choice runway-api-key
```

2. عيّن Runway كموفّر الفيديو الافتراضي:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "runway/gen4.5"
```

3. اطلب من الوكيل إنشاء فيديو. سيتم استخدام Runway تلقائيًا.

## الأوضاع المدعومة

| الوضع           | النموذج              | الإدخال المرجعي         |
| -------------- | ------------------ | ----------------------- |
| نص إلى فيديو  | `gen4.5` (افتراضي) | لا شيء                    |
| صورة إلى فيديو | `gen4.5`           | صورة محلية أو بعيدة واحدة |
| فيديو إلى فيديو | `gen4_aleph`       | فيديو محلي أو بعيد واحد |

- يتم دعم مراجع الصور والفيديو المحلية عبر معرّفات URI للبيانات.
- يتطلب وضع الفيديو إلى فيديو حاليًا `runway/gen4_aleph` تحديدًا.
- تعرض التشغيلات النصية فقط حاليًا نسب الأبعاد `16:9` و `9:16`.

## الإعدادات

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## ذو صلة

- [توليد الفيديو](/tools/video-generation) -- معاملات الأداة المشتركة، واختيار الموفّر، والسلوك غير المتزامن
- [مرجع الإعدادات](/ar/gateway/configuration-reference#agent-defaults)
