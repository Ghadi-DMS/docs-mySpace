---
read_when:
    - تريد تشغيل OpenClaw مقابل خادم SGLang محلي
    - تريد نقاط نهاية `/v1` متوافقة مع OpenAI مع نماذجك الخاصة
summary: شغّل OpenClaw مع SGLang ‏(خادم مستضاف ذاتيًا ومتوافق مع OpenAI)
title: SGLang
x-i18n:
    generated_at: "2026-04-05T12:53:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9850277c6c5e318e60237688b4d8a5b1387d4e9586534ae2eb6ad953abba8948
    source_path: providers/sglang.md
    workflow: 15
---

# SGLang

يمكن لـ SGLang خدمة النماذج مفتوحة المصدر عبر HTTP API **متوافقة مع OpenAI**.
ويمكن لـ OpenClaw الاتصال بـ SGLang باستخدام API ‏`openai-completions`.

كما يمكن لـ OpenClaw أيضًا **اكتشاف** النماذج المتاحة من SGLang تلقائيًا عندما
تشترك باستخدام `SGLANG_API_KEY` ‏(أي قيمة تعمل إذا كان خادمك لا يفرض المصادقة)
ولا تعرّف إدخالًا صريحًا `models.providers.sglang`.

## بدء سريع

1. ابدأ SGLang باستخدام خادم متوافق مع OpenAI.

يجب أن يعرض base URL لديك نقاط النهاية `/v1` ‏(مثل `/v1/models`,
و`/v1/chat/completions`). وغالبًا ما يعمل SGLang على:

- `http://127.0.0.1:30000/v1`

2. اشترك (أي قيمة تعمل إذا لم تُضبط مصادقة):

```bash
export SGLANG_API_KEY="sglang-local"
```

3. شغّل الإعداد الأولي واختر `SGLang`، أو اضبط نموذجًا مباشرة:

```bash
openclaw onboard
```

```json5
{
  agents: {
    defaults: {
      model: { primary: "sglang/your-model-id" },
    },
  },
}
```

## اكتشاف النماذج (مزوّد ضمني)

عندما يكون `SGLANG_API_KEY` مضبوطًا (أو توجد auth profile) وأنت **لا**
تعرّف `models.providers.sglang`، سيستعلم OpenClaw عن:

- `GET http://127.0.0.1:30000/v1/models`

وسيحوّل المعرّفات المُعادة إلى إدخالات نماذج.

إذا ضبطت `models.providers.sglang` صراحةً، فسيُتخطى الاكتشاف التلقائي
ويجب عليك تعريف النماذج يدويًا.

## إعداد صريح (نماذج يدوية)

استخدم الإعداد الصريح عندما:

- تعمل SGLang على مضيف/منفذ مختلفين.
- تريد تثبيت قيم `contextWindow`/`maxTokens`.
- يتطلب خادمك API key حقيقية (أو تريد التحكم في الترويسات).

```json5
{
  models: {
    providers: {
      sglang: {
        baseUrl: "http://127.0.0.1:30000/v1",
        apiKey: "${SGLANG_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "your-model-id",
            name: "Local SGLang Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

## استكشاف الأخطاء وإصلاحها

- تحقق من أن الخادم قابل للوصول:

```bash
curl http://127.0.0.1:30000/v1/models
```

- إذا فشلت الطلبات بأخطاء مصادقة، فاضبط `SGLANG_API_KEY` حقيقية تطابق
  إعداد خادمك، أو هيّئ المزوّد صراحةً تحت
  `models.providers.sglang`.

## سلوك بنمط الوكيل

تُعامَل SGLang على أنها واجهة `/v1` متوافقة مع OpenAI بنمط الوكيل، وليست
نقطة نهاية OpenAI أصلية.

- لا يُطبَّق هنا تشكيل الطلبات الأصلية الخاصة بـ OpenAI فقط
- لا يوجد `service_tier`، ولا Responses `store`، ولا تلميحات تخزين مؤقت للموجّه، ولا
  تشكيل لحمولة توافق الاستدلال الخاصة بـ OpenAI
- لا تُحقن ترويسات الإسناد المخفية الخاصة بـ OpenClaw (`originator`, `version`, `User-Agent`)
  على base URLs المخصصة الخاصة بـ SGLang
