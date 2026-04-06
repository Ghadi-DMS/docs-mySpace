---
read_when:
    - تريد استخدام توليد الصور عبر fal في OpenClaw
    - تحتاج إلى تدفق المصادقة FAL_KEY
    - تريد الإعدادات الافتراضية لـ fal من أجل image_generate أو video_generate
summary: إعداد توليد الصور والفيديو عبر fal في OpenClaw
title: fal
x-i18n:
    generated_at: "2026-04-06T03:11:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1922907d2c8360c5877a56495323d54bd846d47c27a801155e3d11e3f5706fbd
    source_path: providers/fal.md
    workflow: 15
---

# fal

يوفر OpenClaw موفر `fal` مجمعًا لتوليد الصور والفيديو المستضاف.

- الموفّر: `fal`
- المصادقة: `FAL_KEY` (الأساسي؛ ويعمل `FAL_API_KEY` أيضًا كخيار احتياطي)
- API: نقاط نهاية نماذج fal

## بداية سريعة

1. اضبط API key:

```bash
openclaw onboard --auth-choice fal-api-key
```

2. اضبط نموذج الصور الافتراضي:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## توليد الصور

يستخدم موفّر توليد الصور `fal` المجمّع افتراضيًا
`fal/fal-ai/flux/dev`.

- التوليد: حتى 4 صور لكل طلب
- وضع التحرير: مفعّل، مع صورة مرجعية واحدة
- يدعم `size` و`aspectRatio` و`resolution`
- ملاحظة حالية على التحرير: لا تدعم نقطة نهاية تحرير الصور في fal
  تجاوزات `aspectRatio`

لاستخدام fal كموفّر الصور الافتراضي:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## توليد الفيديو

يستخدم موفّر توليد الفيديو `fal` المجمّع افتراضيًا
`fal/fal-ai/minimax/video-01-live`.

- الأوضاع: تحويل النص إلى فيديو وتدفقات مرجعية بصورة واحدة
- وقت التشغيل: تدفق إرسال/حالة/نتيجة مدعوم بالطابور للمهام طويلة التشغيل

لاستخدام fal كموفّر الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/fal-ai/minimax/video-01-live",
      },
    },
  },
}
```

## ذو صلة

- [Image Generation](/ar/tools/image-generation)
- [Video Generation](/tools/video-generation)
- [Configuration Reference](/ar/gateway/configuration-reference#agent-defaults)
