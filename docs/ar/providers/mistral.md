---
read_when:
    - تريد استخدام نماذج Mistral في OpenClaw
    - تحتاج إلى إعداد Mistral بمفتاح API ومراجع النماذج
summary: استخدم نماذج Mistral ونسخ Voxtral الصوتي مع OpenClaw
title: Mistral
x-i18n:
    generated_at: "2026-04-08T02:18:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e32a0eb2a37dba6383ba338b06a8d0be600e7443aa916225794ccb0fdf46aee
    source_path: providers/mistral.md
    workflow: 15
---

# Mistral

يدعم OpenClaw موفر Mistral لكلٍّ من توجيه النماذج النصية/المرئية (`mistral/...`) ونسخ
الصوت عبر Voxtral ضمن فهم الوسائط.
كما يمكن استخدام Mistral لتضمينات الذاكرة (`memorySearch.provider = "mistral"`).

## إعداد CLI

```bash
openclaw onboard --auth-choice mistral-api-key
# أو بشكل غير تفاعلي
openclaw onboard --mistral-api-key "$MISTRAL_API_KEY"
```

## مقتطف إعدادات (موفر LLM)

```json5
{
  env: { MISTRAL_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
}
```

## فهرس LLM المضمن

يشحن OpenClaw حاليًا فهرس Mistral المضمن التالي:

| مرجع النموذج                     | الإدخال      | السياق  | الحد الأقصى للإخراج | ملاحظات                                                           |
| -------------------------------- | ------------ | ------- | ------------------- | ----------------------------------------------------------------- |
| `mistral/mistral-large-latest`   | نص، صورة     | 262,144 | 16,384              | النموذج الافتراضي                                                 |
| `mistral/mistral-medium-2508`    | نص، صورة     | 262,144 | 8,192               | Mistral Medium 3.1                                                |
| `mistral/mistral-small-latest`   | نص، صورة     | 128,000 | 16,384              | Mistral Small 4؛ استدلال قابل للضبط عبر `reasoning_effort` في API |
| `mistral/pixtral-large-latest`   | نص، صورة     | 128,000 | 32,768              | Pixtral                                                           |
| `mistral/codestral-latest`       | نص           | 256,000 | 4,096               | للبرمجة                                                           |
| `mistral/devstral-medium-latest` | نص           | 262,144 | 32,768              | Devstral 2                                                        |
| `mistral/magistral-small`        | نص           | 128,000 | 40,000              | مع تمكين الاستدلال                                                |

## مقتطف إعدادات (نسخ الصوت باستخدام Voxtral)

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

## الاستدلال القابل للضبط (`mistral-small-latest`)

يشير `mistral/mistral-small-latest` إلى Mistral Small 4 ويدعم [الاستدلال القابل للضبط](https://docs.mistral.ai/capabilities/reasoning/adjustable) على Chat Completions API عبر `reasoning_effort` (تقلل `none` التفكير الإضافي في المخرجات إلى الحد الأدنى؛ بينما يعرض `high` آثار التفكير الكاملة قبل الإجابة النهائية).

يربط OpenClaw مستوى **thinking** في الجلسة بـ API الخاص بـ Mistral:

- **off** / **minimal** → `none`
- **low** / **medium** / **high** / **xhigh** / **adaptive** → `high`

لا تستخدم النماذج الأخرى في فهرس Mistral المضمن هذا المعامل؛ واصل استخدام نماذج `magistral-*` عندما تريد سلوك Mistral الأصلي المعتمد على الاستدلال أولًا.

## ملاحظات

- تستخدم مصادقة Mistral المتغير `MISTRAL_API_KEY`.
- يكون عنوان URL الأساسي للموفر افتراضيًا `https://api.mistral.ai/v1`.
- النموذج الافتراضي في الإعداد الأولي هو `mistral/mistral-large-latest`.
- نموذج الصوت الافتراضي لفهم الوسائط في Mistral هو `voxtral-mini-latest`.
- يستخدم مسار نسخ الوسائط `/v1/audio/transcriptions`.
- يستخدم مسار تضمينات الذاكرة `/v1/embeddings` (النموذج الافتراضي: `mistral-embed`).
