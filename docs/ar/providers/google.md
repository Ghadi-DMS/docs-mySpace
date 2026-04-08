---
read_when:
    - تريد استخدام نماذج Google Gemini مع OpenClaw
    - تحتاج إلى تدفق المصادقة بمفتاح API أو OAuth
summary: إعداد Google Gemini ‏(مفتاح API وOAuth، وإنشاء الصور، وفهم الوسائط، والبحث على الويب)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-08T02:17:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: e9e558f5ce35c853e0240350be9a1890460c5f7f7fd30b05813a656497dee516
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

يوفر plugin Google إمكانية الوصول إلى نماذج Gemini عبر Google AI Studio، بالإضافة إلى
إنشاء الصور، وفهم الوسائط (الصور/الصوت/الفيديو)، والبحث على الويب عبر
Gemini Grounding.

- الموفر: `google`
- المصادقة: `GEMINI_API_KEY` أو `GOOGLE_API_KEY`
- API: Google Gemini API
- موفر بديل: `google-gemini-cli` ‏(OAuth)

## بداية سريعة

1. اضبط مفتاح API:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. اضبط نموذجًا افتراضيًا:

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

## OAuth (Gemini CLI)

يستخدم موفر بديل باسم `google-gemini-cli` مصادقة PKCE OAuth بدلًا من مفتاح API.
هذا تكامل غير رسمي؛ وقد أبلغ بعض المستخدمين عن
قيود على الحساب. استخدمه على مسؤوليتك الخاصة.

- النموذج الافتراضي: `google-gemini-cli/gemini-3-flash-preview`
- الاسم المستعار: `gemini-cli`
- متطلب التثبيت المسبق: أن يكون Gemini CLI المحلي متاحًا باسم `gemini`
  - Homebrew: `brew install gemini-cli`
  - npm: `npm install -g @google/gemini-cli`
- تسجيل الدخول:

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

متغيرات البيئة:

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(أو الصيغ `GEMINI_CLI_*`.)

إذا فشلت طلبات Gemini CLI OAuth بعد تسجيل الدخول، فاضبط
`GOOGLE_CLOUD_PROJECT` أو `GOOGLE_CLOUD_PROJECT_ID` على مضيف البوابة ثم
أعد المحاولة.

إذا فشل تسجيل الدخول قبل بدء تدفق المتصفح، فتأكد من أن الأمر المحلي `gemini`
مثبّت وموجود على `PATH`. يدعم OpenClaw كلاً من عمليات التثبيت عبر Homebrew
وعمليات التثبيت العامة عبر npm، بما في ذلك تخطيطات Windows/npm الشائعة.

ملاحظات استخدام Gemini CLI JSON:

- يأتي نص الرد من الحقل `response` في JSON الخاص بـ CLI.
- يعود الاستخدام إلى `stats` عندما يترك CLI الحقل `usage` فارغًا.
- تتم تسوية `stats.cached` إلى OpenClaw `cacheRead`.
- إذا كان `stats.input` مفقودًا، فإن OpenClaw يشتق رموز الإدخال من
  `stats.input_tokens - stats.cached`.

## الإمكانات

| الإمكانية             | مدعومة           |
| --------------------- | ---------------- |
| إكمالات الدردشة       | نعم              |
| إنشاء الصور           | نعم              |
| إنشاء الموسيقى        | نعم              |
| فهم الصور             | نعم              |
| نسخ الصوت             | نعم              |
| فهم الفيديو           | نعم              |
| البحث على الويب (Grounding) | نعم        |
| التفكير/الاستدلال     | نعم (Gemini 3.1+) |

## إعادة استخدام ذاكرة Gemini المؤقتة المباشرة

بالنسبة إلى التشغيلات المباشرة لـ Gemini API ‏(`api: "google-generative-ai"`)، يقوم OpenClaw الآن
بتمرير معرّف `cachedContent` المهيأ إلى طلبات Gemini.

- اضبط معلمات لكل نموذج أو معلمات عامة باستخدام
  `cachedContent` أو `cached_content` القديم
- إذا وُجد الاثنان، تكون الأولوية لـ `cachedContent`
- مثال على القيمة: `cachedContents/prebuilt-context`
- تتم تسوية استخدام إصابة الذاكرة المؤقتة في Gemini إلى OpenClaw `cacheRead` من
  `cachedContentTokenCount` الصادر من الخدمة upstream

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

يفترض موفر إنشاء الصور `google` المضمن افتراضيًا استخدام
`google/gemini-3.1-flash-image-preview`.

- يدعم أيضًا `google/gemini-3-pro-image-preview`
- الإنشاء: حتى 4 صور لكل طلب
- وضع التحرير: مفعّل، وحتى 5 صور إدخال
- عناصر التحكم الهندسية: `size` و`aspectRatio` و`resolution`

يُعد الموفر `google-gemini-cli` المعتمد على OAuth فقط واجهة منفصلة
للاستدلال النصي. أما إنشاء الصور، وفهم الوسائط، وGemini Grounding فتبقى على
معرّف الموفر `google`.

لاستخدام Google كموفر الصور الافتراضي:

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

راجع [إنشاء الصور](/ar/tools/image-generation) للتعرّف على
معلمات الأداة المشتركة، واختيار الموفر، وسلوك التبديل الاحتياطي.

## إنشاء الفيديو

يسجل plugin `google` المضمن أيضًا إنشاء الفيديو عبر الأداة المشتركة
`video_generate`.

- نموذج الفيديو الافتراضي: `google/veo-3.1-fast-generate-preview`
- الأوضاع: نص إلى فيديو، وصورة إلى فيديو، وتدفقات مرجعية لفيديو واحد
- يدعم `aspectRatio` و`resolution` و`audio`
- القيد الحالي للمدة: **من 4 إلى 8 ثوانٍ**

لاستخدام Google كموفر الفيديو الافتراضي:

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

راجع [إنشاء الفيديو](/ar/tools/video-generation) للتعرّف على
معلمات الأداة المشتركة، واختيار الموفر، وسلوك التبديل الاحتياطي.

## إنشاء الموسيقى

يسجل plugin `google` المضمن أيضًا إنشاء الموسيقى عبر الأداة المشتركة
`music_generate`.

- نموذج الموسيقى الافتراضي: `google/lyria-3-clip-preview`
- يدعم أيضًا `google/lyria-3-pro-preview`
- عناصر التحكم في الموجّه: `lyrics` و`instrumental`
- تنسيق الإخراج: `mp3` افتراضيًا، بالإضافة إلى `wav` على `google/lyria-3-pro-preview`
- المدخلات المرجعية: حتى 10 صور
- يتم فصل التشغيلات المدعومة بالجلسات عبر تدفق المهمة/الحالة المشترك، بما في ذلك `action: "status"`

لاستخدام Google كموفر الموسيقى الافتراضي:

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

راجع [إنشاء الموسيقى](/ar/tools/music-generation) للتعرّف على
معلمات الأداة المشتركة، واختيار الموفر، وسلوك التبديل الاحتياطي.

## ملاحظة البيئة

إذا كانت البوابة تعمل كخدمة خلفية (launchd/systemd)، فتأكد من أن `GEMINI_API_KEY`
متاح لتلك العملية (على سبيل المثال، في `~/.openclaw/.env` أو عبر
`env.shellEnv`).
