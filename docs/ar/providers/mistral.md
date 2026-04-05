---
read_when:
    - تريد استخدام نماذج Mistral في OpenClaw
    - تحتاج إلى onboarding لمفتاح Mistral API وmodel refs
summary: استخدم نماذج Mistral ونسخ Voxtral مع OpenClaw
title: Mistral
x-i18n:
    generated_at: "2026-04-05T12:53:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8f61b9e0656dd7e0243861ddf14b1b41a07c38bff27cef9ad0815d14c8e34408
    source_path: providers/mistral.md
    workflow: 15
---

# Mistral

يدعم OpenClaw مزود Mistral لكل من توجيه النماذج النصية/الصور (`mistral/...`) ونسخ
الصوت عبر Voxtral في فهم الوسائط.
كما يمكن استخدام Mistral في embeddings الخاصة بالذاكرة (`memorySearch.provider = "mistral"`).

## إعداد CLI

```bash
openclaw onboard --auth-choice mistral-api-key
# أو بشكل غير تفاعلي
openclaw onboard --mistral-api-key "$MISTRAL_API_KEY"
```

## مقتطف تكوين (مزوّد LLM)

```json5
{
  env: { MISTRAL_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
}
```

## كتالوج LLM المضمّن

يشحن OpenClaw حاليًا كتالوج Mistral المضمّن التالي:

| مرجع النموذج                       | الإدخال      | السياق  | الحد الأقصى للخرج | ملاحظات                   |
| ---------------------------------- | ------------ | ------- | ----------------- | ------------------------- |
| `mistral/mistral-large-latest`     | نص، صورة     | 262,144 | 16,384            | النموذج الافتراضي         |
| `mistral/mistral-medium-2508`      | نص، صورة     | 262,144 | 8,192             | Mistral Medium 3.1        |
| `mistral/mistral-small-latest`     | نص، صورة     | 128,000 | 16,384            | نموذج أصغر متعدد الوسائط  |
| `mistral/pixtral-large-latest`     | نص، صورة     | 128,000 | 32,768            | Pixtral                   |
| `mistral/codestral-latest`         | نص           | 256,000 | 4,096             | للبرمجة                   |
| `mistral/devstral-medium-latest`   | نص           | 262,144 | 32,768            | Devstral 2                |
| `mistral/magistral-small`          | نص           | 128,000 | 40,000            | يدعم الاستدلال            |

## مقتطف تكوين (نسخ الصوت باستخدام Voxtral)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", model: "voxtral-mini-latest" }],
      },
    },
  },
}
```

## ملاحظات

- تستخدم مصادقة Mistral القيمة `MISTRAL_API_KEY`.
- تكون القيمة الافتراضية لعنوان URL الأساسي للمزوّد هي `https://api.mistral.ai/v1`.
- النموذج الافتراضي في onboarding هو `mistral/mistral-large-latest`.
- نموذج الصوت الافتراضي لفهم الوسائط في Mistral هو `voxtral-mini-latest`.
- يستخدم مسار نسخ الوسائط المسار `/v1/audio/transcriptions`.
- يستخدم مسار embeddings الخاصة بالذاكرة المسار `/v1/embeddings` ‏(النموذج الافتراضي: `mistral-embed`).
