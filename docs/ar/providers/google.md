---
read_when:
    - تريد استخدام نماذج Google Gemini مع OpenClaw
    - تحتاج إلى تدفق المصادقة بمفتاح API
summary: إعداد Google Gemini (مفتاح API، وإنشاء الصور، وفهم الوسائط، والبحث على الويب)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-06T03:11:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 358d33a68275b01ebd916a3621dd651619cb9a1d062e2fb6196a7f3c501c015a
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

يوفّر plugin الخاص بـ Google الوصول إلى نماذج Gemini عبر Google AI Studio، بالإضافة
إلى إنشاء الصور، وفهم الوسائط (الصور/الصوت/الفيديو)، والبحث على الويب عبر
Gemini Grounding.

- الموفّر: `google`
- المصادقة: `GEMINI_API_KEY` أو `GOOGLE_API_KEY`
- API: ‏Google Gemini API

## البدء السريع

1. عيّن مفتاح API:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. عيّن نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## مثال غير تفاعلي

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## القدرات

| القدرة | مدعومة |
| ---------------------- | ----------------- |
| إكمالات الدردشة | نعم |
| إنشاء الصور | نعم |
| إنشاء الموسيقى | نعم |
| فهم الصور | نعم |
| تفريغ الصوت | نعم |
| فهم الفيديو | نعم |
| البحث على الويب (Grounding) | نعم |
| التفكير/الاستدلال | نعم (`Gemini 3.1+`) |

## إعادة استخدام ذاكرة Gemini المؤقتة المباشرة

في عمليات تشغيل Gemini API المباشرة (`api: "google-generative-ai"`)، يمرر OpenClaw الآن
مقبض `cachedContent` المهيأ إلى طلبات Gemini.

- هيئ المعلمات لكل نموذج أو المعلمات العامة باستخدام
  `cachedContent` أو الاسم القديم `cached_content`
- إذا وُجدا معًا، فإن `cachedContent` له الأولوية
- مثال على القيمة: `cachedContents/prebuilt-context`
- يُطبَّع استخدام Gemini cache-hit إلى `cacheRead` في OpenClaw من
  `cachedContentTokenCount` القادم من upstream

مثال:

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## إنشاء الصور

يستخدم موفّر إنشاء الصور المضمن `google` افتراضيًا
`google/gemini-3.1-flash-image-preview`.

- يدعم أيضًا `google/gemini-3-pro-image-preview`
- الإنشاء: حتى 4 صور لكل طلب
- وضع التحرير: مفعّل، وحتى 5 صور إدخال
- عناصر التحكم الهندسية: `size` و`aspectRatio` و`resolution`

يبقى إنشاء الصور، وفهم الوسائط، وGemini Grounding كلها تحت
معرّف الموفّر `google`.

لاستخدام Google بوصفه موفّر الصور الافتراضي:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

راجع [إنشاء الصور](/ar/tools/image-generation) للاطلاع على معلمات
الأداة المشتركة، واختيار الموفّر، وسلوك التحويل الاحتياطي.

## إنشاء الفيديو

يسجل plugin المضمن `google` أيضًا إنشاء الفيديو عبر الأداة المشتركة
`video_generate`.

- نموذج الفيديو الافتراضي: `google/veo-3.1-fast-generate-preview`
- الأوضاع: نص إلى فيديو، وصورة إلى فيديو، وتدفقات مرجعية لفيديو واحد
- يدعم `aspectRatio` و`resolution` و`audio`
- حد المدة الحالي: **من 4 إلى 8 ثوانٍ**

لاستخدام Google بوصفه موفّر الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

راجع [إنشاء الفيديو](/tools/video-generation) للاطلاع على معلمات
الأداة المشتركة، واختيار الموفّر، وسلوك التحويل الاحتياطي.

## إنشاء الموسيقى

يسجل plugin المضمن `google` أيضًا إنشاء الموسيقى عبر الأداة المشتركة
`music_generate`.

- نموذج الموسيقى الافتراضي: `google/lyria-3-clip-preview`
- يدعم أيضًا `google/lyria-3-pro-preview`
- عناصر التحكم في prompt: ‏`lyrics` و`instrumental`
- تنسيق الإخراج: `mp3` افتراضيًا، بالإضافة إلى `wav` على `google/lyria-3-pro-preview`
- المدخلات المرجعية: حتى 10 صور
- تفصل التشغيلات المدعومة بالجلسات عبر تدفق المهمة/الحالة المشترك، بما في ذلك `action: "status"`

لاستخدام Google بوصفه موفّر الموسيقى الافتراضي:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

راجع [إنشاء الموسيقى](/tools/music-generation) للاطلاع على معلمات
الأداة المشتركة، واختيار الموفّر، وسلوك التحويل الاحتياطي.

## ملاحظة البيئة

إذا كان Gateway يعمل كخدمة daemon ‏(`launchd/systemd`)، فتأكد من أن `GEMINI_API_KEY`
متاح لتلك العملية (على سبيل المثال، في `~/.openclaw/.env` أو عبر
`env.shellEnv`).
