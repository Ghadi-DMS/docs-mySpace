---
read_when:
    - تريد تشغيل OpenClaw مقابل خادم vLLM محلي
    - تريد نقاط نهاية `/v1` متوافقة مع OpenAI مع نماذجك الخاصة
summary: شغّل OpenClaw مع vLLM ‏(خادم محلي متوافق مع OpenAI)
title: vLLM
x-i18n:
    generated_at: "2026-04-05T12:54:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: ebde34d0453586d10340680b8d51465fdc98bd28e8a96acfaeb24606886b50f4
    source_path: providers/vllm.md
    workflow: 15
---

# vLLM

يمكن لـ vLLM تقديم نماذج مفتوحة المصدر (وبعض النماذج المخصصة) عبر واجهة HTTP API **متوافقة مع OpenAI**. ويمكن لـ OpenClaw الاتصال بـ vLLM باستخدام API من نوع `openai-completions`.

يمكن لـ OpenClaw أيضًا **اكتشاف النماذج المتاحة تلقائيًا** من vLLM عندما تشترك في ذلك عبر `VLLM_API_KEY` ‏(أي قيمة تعمل إذا كان خادمك لا يفرض المصادقة) ولا تعرّف إدخال `models.providers.vllm` صريحًا.

## البدء السريع

1. ابدأ تشغيل vLLM باستخدام خادم متوافق مع OpenAI.

يجب أن يعرض عنوان URL الأساسي لديك نقاط نهاية `/v1` ‏(مثل `/v1/models` و`/v1/chat/completions`). يعمل vLLM غالبًا على:

- `http://127.0.0.1:8000/v1`

2. اشترك في الميزة (أي قيمة تعمل إذا لم يتم تكوين مصادقة):

```bash
export VLLM_API_KEY="vllm-local"
```

3. اختر نموذجًا (استبدله بأحد معرّفات نماذج vLLM لديك):

```json5
{
  agents: {
    defaults: {
      model: { primary: "vllm/your-model-id" },
    },
  },
}
```

## اكتشاف النموذج (مزوّد ضمني)

عندما تكون `VLLM_API_KEY` معيّنة (أو يوجد ملف تعريف مصادقة) و**لا** تعرّف `models.providers.vllm`، سيستعلم OpenClaw من:

- `GET http://127.0.0.1:8000/v1/models`

...ويحوّل المعرّفات المعادة إلى إدخالات نماذج.

إذا قمت بتعيين `models.providers.vllm` صراحةً، فسيتم تخطي الاكتشاف التلقائي ويجب عليك تعريف النماذج يدويًا.

## التكوين الصريح (نماذج يدوية)

استخدم التكوين الصريح عندما:

- يعمل vLLM على مضيف/منفذ مختلف.
- تريد تثبيت قيم `contextWindow`/`maxTokens`.
- يتطلب خادمك مفتاح API حقيقيًا (أو تريد التحكم في الرؤوس).

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "your-model-id",
            name: "Local vLLM Model",
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

- تحقق من إمكانية الوصول إلى الخادم:

```bash
curl http://127.0.0.1:8000/v1/models
```

- إذا فشلت الطلبات بسبب أخطاء مصادقة، فعيّن `VLLM_API_KEY` حقيقيًا يطابق تكوين خادمك، أو قم بتكوين المزوّد صراحةً تحت `models.providers.vllm`.

## سلوك شبيه بالـ proxy

يُعامل vLLM على أنه واجهة خلفية من نوع proxy متوافقة مع OpenAI `/v1`، وليس
نقطة نهاية OpenAI أصلية.

- لا ينطبق هنا تشكيل الطلبات الخاص بـ OpenAI الأصلية فقط
- لا يوجد `service_tier`، ولا `store` الخاصة بـ Responses، ولا تلميحات ذاكرة التخزين المؤقت للمطالبات، ولا تشكيل حمولات reasoning-compat الخاصة بـ OpenAI
- لا يتم حقن رؤوس الإسناد المخفية الخاصة بـ OpenClaw ‏(`originator` و`version` و`User-Agent`)
  على عناوين URL الأساسية المخصصة لـ vLLM
