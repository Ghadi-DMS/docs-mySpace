---
read_when:
    - تريد استخدام نماذج Google Gemini مع OpenClaw
    - تحتاج إلى تدفق المصادقة بمفتاح API أو OAuth
summary: إعداد Google Gemini ‏(مفتاح API وOAuth، وتوليد الصور، وفهم الوسائط، والبحث على الويب)
title: Google ‏(Gemini)
x-i18n:
    generated_at: "2026-04-05T12:53:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: fa3c4326e83fad277ae4c2cb9501b6e89457afcfa7e3e1d57ae01c9c0c6846e2
    source_path: providers/google.md
    workflow: 15
---

# Google ‏(Gemini)

يوفر plugin ‏Google الوصول إلى نماذج Gemini عبر Google AI Studio، بالإضافة إلى
توليد الصور، وفهم الوسائط (الصورة/الصوت/الفيديو)، والبحث على الويب عبر
Gemini Grounding.

- المزوّد: `google`
- المصادقة: `GEMINI_API_KEY` أو `GOOGLE_API_KEY`
- API: ‏Google Gemini API
- مزوّد بديل: `google-gemini-cli` ‏(OAuth)

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

## OAuth ‏(Gemini CLI)

يستخدم المزوّد البديل `google-gemini-cli` بروتوكول PKCE OAuth بدلًا من مفتاح
API. وهذا تكامل غير رسمي؛ وقد أبلغ بعض المستخدمين عن وجود
قيود على الحساب. استخدمه على مسؤوليتك الخاصة.

- النموذج الافتراضي: `google-gemini-cli/gemini-3.1-pro-preview`
- الاسم المستعار: `gemini-cli`
- متطلب التثبيت: توفر Gemini CLI محليًا باسم `gemini`
  - Homebrew: ‏`brew install gemini-cli`
  - npm: ‏`npm install -g @google/gemini-cli`
- تسجيل الدخول:

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

متغيرات البيئة:

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(أو متغيرات `GEMINI_CLI_*` المكافئة.)

إذا فشلت طلبات Gemini CLI OAuth بعد تسجيل الدخول، فعيّن
`GOOGLE_CLOUD_PROJECT` أو `GOOGLE_CLOUD_PROJECT_ID` على مضيف gateway ثم
أعد المحاولة.

إذا فشل تسجيل الدخول قبل بدء تدفق المتصفح، فتأكد من أن الأمر المحلي `gemini`
مثبّت وموجود في `PATH`. يدعم OpenClaw كلًا من تثبيتات Homebrew
وتثبيتات npm العامة، بما في ذلك تخطيطات Windows/npm الشائعة.

ملاحظات استخدام JSON في Gemini CLI:

- يأتي نص الرد من الحقل `response` في JSON الخاص بـ CLI.
- يعود الاستخدام إلى `stats` عندما تترك CLI قيمة `usage` فارغة.
- يتم تطبيع `stats.cached` إلى `cacheRead` في OpenClaw.
- إذا كانت `stats.input` مفقودة، يشتق OpenClaw رموز الإدخال من
  `stats.input_tokens - stats.cached`.

## القدرات

| القدرة                 | مدعومة            |
| ---------------------- | ----------------- |
| إكمالات الدردشة        | نعم               |
| توليد الصور            | نعم               |
| فهم الصور              | نعم               |
| نسخ الصوت              | نعم               |
| فهم الفيديو            | نعم               |
| البحث على الويب (Grounding) | نعم         |
| التفكير/الاستدلال      | نعم (Gemini 3.1+) |

## إعادة استخدام ذاكرة Gemini المؤقتة مباشرة

بالنسبة إلى تشغيلات Gemini API المباشرة (`api: "google-generative-ai"`)، يمرر OpenClaw الآن
المقبض `cachedContent` المكوَّن إلى طلبات Gemini.

- كوّن المعلمات لكل نموذج أو عالميًا باستخدام
  `cachedContent` أو الصيغة القديمة `cached_content`
- إذا وُجد الاثنان، فإن `cachedContent` هي الفائزة
- مثال على القيمة: `cachedContents/prebuilt-context`
- يتم تطبيع استخدام إصابة ذاكرة Gemini المؤقتة إلى `cacheRead` في OpenClaw من
  `cachedContentTokenCount` الصادر من المصدر

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

## توليد الصور

يعتمد مزود توليد الصور `google` المضمّن افتراضيًا على
`google/gemini-3.1-flash-image-preview`.

- يدعم أيضًا `google/gemini-3-pro-image-preview`
- التوليد: حتى 4 صور لكل طلب
- وضع التحرير: مفعّل، حتى 5 صور إدخال
- عناصر التحكم الهندسية: `size` و`aspectRatio` و`resolution`

يمثل المزوّد `google-gemini-cli` المعتمد على OAuth فقط سطحًا منفصلًا
للاستدلال النصي. أما توليد الصور، وفهم الوسائط، وGemini Grounding فتبقى على
معرّف المزوّد `google`.

## ملاحظة حول البيئة

إذا كانت Gateway تعمل كـ daemon ‏(launchd/systemd)، فتأكد من أن `GEMINI_API_KEY`
متاح لتلك العملية (مثلًا، في `~/.openclaw/.env` أو عبر
`env.shellEnv`).
