---
read_when:
    - تريد تقديم النماذج من جهاز GPU الخاص بك
    - أنت توصل LM Studio أو proxy متوافقًا مع OpenAI
    - تحتاج إلى أفضل إرشادات أمان للنماذج المحلية
summary: تشغيل OpenClaw على نماذج LLM محلية (LM Studio وvLLM وLiteLLM ونقاط نهاية OpenAI مخصصة)
title: النماذج المحلية
x-i18n:
    generated_at: "2026-04-05T12:43:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b99c8fb57f65c0b765fc75bd36933221b5aeb94c4a3f3428f92640ae064f8b6
    source_path: gateway/local-models.md
    workflow: 15
---

# النماذج المحلية

النماذج المحلية ممكنة، لكن OpenClaw يتوقع سياقًا كبيرًا + دفاعات قوية ضد حقن المطالبات. البطاقات الصغيرة تقطع السياق وتضعف الأمان. استهدف مستوى عالٍ: **≥2 من أجهزة Mac Studio بمواصفات قصوى أو جهاز GPU مكافئ (~30 ألف دولار فأكثر)**. تعمل وحدة **24 GB** GPU واحدة فقط مع المطالبات الأخف وبزمن استجابة أعلى. استخدم **أكبر / النسخة الكاملة من النموذج التي يمكنك تشغيلها**؛ إذ إن نقاط الفحص المكممة بشدة أو “الصغيرة” ترفع خطر حقن المطالبات (راجع [Security](/gateway/security)).

إذا كنت تريد أقل إعداد محلي احتكاكًا، فابدأ بـ [Ollama](/providers/ollama) و`openclaw onboard`. هذه الصفحة هي الدليل العملي الموجّه للحِزم المحلية الأعلى مستوى وخوادم OpenAI-compatible المحلية المخصصة.

## الموصى به: LM Studio + نموذج محلي كبير (Responses API)

أفضل حزمة محلية حاليًا. حمّل نموذجًا كبيرًا في LM Studio (مثل إصدار كامل من Qwen أو DeepSeek أو Llama)، وفعّل الخادم المحلي (الافتراضي `http://127.0.0.1:1234`)، واستخدم Responses API لإبقاء الاستدلال منفصلًا عن النص النهائي.

```json5
{
  agents: {
    defaults: {
      model: { primary: “lmstudio/my-local-model” },
      models: {
        “anthropic/claude-opus-4-6”: { alias: “Opus” },
        “lmstudio/my-local-model”: { alias: “Local” },
      },
    },
  },
  models: {
    mode: “merge”,
    providers: {
      lmstudio: {
        baseUrl: “http://127.0.0.1:1234/v1”,
        apiKey: “lmstudio”,
        api: “openai-responses”,
        models: [
          {
            id: “my-local-model”,
            name: “Local Model”,
            reasoning: false,
            input: [“text”],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

**قائمة التحقق للإعداد**

- ثبّت LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- في LM Studio، نزّل **أكبر إصدار نموذج متاح** (تجنب الإصدارات “الصغيرة”/المكممة بشدة)، وابدأ الخادم، وتأكد من أن `http://127.0.0.1:1234/v1/models` يسرده.
- استبدل `my-local-model` بمعرّف النموذج الفعلي الظاهر في LM Studio.
- أبقِ النموذج محمّلًا؛ إذ تضيف البداية الباردة زمن تأخير عند بدء التشغيل.
- اضبط `contextWindow`/`maxTokens` إذا كان إصدار LM Studio لديك يختلف.
- بالنسبة إلى WhatsApp، التزم بـ Responses API بحيث يُرسل النص النهائي فقط.

أبقِ النماذج المستضافة مكوّنة حتى عند التشغيل محليًا؛ استخدم `models.mode: "merge"` حتى تظل fallbacks متاحة.

### تكوين هجين: مستضاف كأساسي، ومحلي كرجوع احتياطي

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### محلي أولًا مع شبكة أمان مستضافة

بدّل ترتيب الأساسي والرجوع الاحتياطي؛ وأبقِ كتلة providers نفسها و`models.mode: "merge"` حتى تتمكن من الرجوع إلى Sonnet أو Opus عندما يتوقف الجهاز المحلي.

### الاستضافة الإقليمية / توجيه البيانات

- توجد أيضًا إصدارات MiniMax/Kimi/GLM المستضافة على OpenRouter مع نقاط نهاية مثبتة على مناطق محددة (مثل الاستضافة في الولايات المتحدة). اختر الإصدار الإقليمي هناك للحفاظ على حركة البيانات ضمن النطاق القضائي الذي تختاره مع الاستمرار في استخدام `models.mode: "merge"` لعمليات الرجوع الاحتياطي إلى Anthropic/OpenAI.
- يظل التشغيل المحلي فقط هو أقوى مسار للخصوصية؛ أما التوجيه الإقليمي المستضاف فهو حل وسط عندما تحتاج إلى ميزات المزوّد ولكنك تريد التحكم في تدفق البيانات.

## Proxies محلية أخرى متوافقة مع OpenAI

تعمل vLLM وLiteLLM وOAI-proxy أو gateways المخصصة إذا كانت تعرض نقطة نهاية `/v1` بأسلوب OpenAI. استبدل كتلة provider أعلاه بنقطة النهاية ومعرّف النموذج لديك:

```json5
{
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

أبقِ `models.mode: "merge"` حتى تظل النماذج المستضافة متاحة كخيارات fallback.

ملاحظة سلوكية لخلفيات `/v1` المحلية/الممررة عبر proxy:

- يتعامل OpenClaw مع هذه المسارات باعتبارها مسارات OpenAI-compatible بنمط proxy، وليس
  كنقاط نهاية OpenAI أصلية
- لا ينطبق هنا تشكيل الطلبات الخاص بـ OpenAI الأصلية فقط: لا
  `service_tier`، ولا `store` الخاصة بـ Responses، ولا تشكيل حمولة
  التوافق مع reasoning في OpenAI، ولا تلميحات prompt-cache
- لا يتم حقن رؤوس الإسناد المخفية الخاصة بـ OpenClaw (`originator`, `version`, `User-Agent`)
  في عناوين proxy المخصصة هذه

## استكشاف الأخطاء وإصلاحها

- هل يستطيع gateway الوصول إلى proxy؟ ‏`curl http://127.0.0.1:1234/v1/models`.
- هل تم إلغاء تحميل نموذج LM Studio؟ أعد تحميله؛ فالبداية الباردة سبب شائع لحالة “التعليق”.
- أخطاء السياق؟ خفّض `contextWindow` أو ارفع حد الخادم لديك.
- الأمان: تتجاوز النماذج المحلية عوامل التصفية من جهة المزوّد؛ لذا أبقِ الوكلاء محدودين وشغّل الضغط للحد من نطاق تأثير حقن المطالبات.
