---
read_when:
    - تريد مفتاح API واحدًا للعديد من نماذج LLM
    - تحتاج إلى إرشادات إعداد Baidu Qianfan
summary: استخدم واجهة Qianfan الموحدة للوصول إلى العديد من النماذج في OpenClaw
title: Qianfan
x-i18n:
    generated_at: "2026-04-05T12:53:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 965d83dd968563447ce3571a73bd71c6876275caff8664311a852b2f9827e55b
    source_path: providers/qianfan.md
    workflow: 15
---

# دليل مزوّد Qianfan

Qianfan هي منصة MaaS من Baidu، وتوفر **واجهة API موحدة** توجّه الطلبات إلى العديد من النماذج خلف
نقطة نهاية واحدة ومفتاح API واحد. وهي متوافقة مع OpenAI، لذلك تعمل معظم حزم OpenAI SDK
بمجرد تبديل base URL.

## المتطلبات المسبقة

1. حساب Baidu Cloud مع وصول إلى Qianfan API
2. مفتاح API من لوحة Qianfan
3. تثبيت OpenClaw على نظامك

## الحصول على مفتاح API

1. زر [لوحة Qianfan](https://console.bce.baidu.com/qianfan/ais/console/apiKey)
2. أنشئ تطبيقًا جديدًا أو اختر تطبيقًا موجودًا
3. أنشئ مفتاح API (بالتنسيق: `bce-v3/ALTAK-...`)
4. انسخ مفتاح API لاستخدامه مع OpenClaw

## إعداد CLI

```bash
openclaw onboard --auth-choice qianfan-api-key
```

## مقتطف التكوين

```json5
{
  env: { QIANFAN_API_KEY: "bce-v3/ALTAK-..." },
  agents: {
    defaults: {
      model: { primary: "qianfan/deepseek-v3.2" },
      models: {
        "qianfan/deepseek-v3.2": { alias: "QIANFAN" },
      },
    },
  },
  models: {
    providers: {
      qianfan: {
        baseUrl: "https://qianfan.baidubce.com/v2",
        api: "openai-completions",
        models: [
          {
            id: "deepseek-v3.2",
            name: "DEEPSEEK V3.2",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 98304,
            maxTokens: 32768,
          },
          {
            id: "ernie-5.0-thinking-preview",
            name: "ERNIE-5.0-Thinking-Preview",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 119000,
            maxTokens: 64000,
          },
        ],
      },
    },
  },
}
```

## ملاحظات

- مرجع النموذج المضمن الافتراضي: `qianfan/deepseek-v3.2`
- base URL الافتراضي: `https://qianfan.baidubce.com/v2`
- يتضمن الفهرس المضمن حاليًا `deepseek-v3.2` و`ernie-5.0-thinking-preview`
- لا تُضف أو تتجاوز `models.providers.qianfan` إلا عندما تحتاج إلى base URL مخصص أو بيانات وصفية مخصصة للنموذج
- يعمل Qianfan عبر مسار النقل المتوافق مع OpenAI، وليس عبر تشكيل الطلبات الأصلي لـ OpenAI

## مستندات ذات صلة

- [تكوين OpenClaw](/gateway/configuration)
- [موفرو النماذج](/concepts/model-providers)
- [إعداد الوكيل](/concepts/agent)
- [مستندات Qianfan API](https://cloud.baidu.com/doc/qianfan-api/s/3m7of64lb)
