---
read_when:
    - تريد تقديم النماذج من جهاز GPU الخاص بك
    - أنت تقوم بإعداد LM Studio أو وكيل متوافق مع OpenAI
    - تحتاج إلى أكثر الإرشادات أمانًا للنماذج المحلية
summary: شغّل OpenClaw على نماذج LLM المحلية (LM Studio، vLLM، LiteLLM، ونقاط نهاية OpenAI المخصصة)
title: النماذج المحلية
x-i18n:
    generated_at: "2026-04-13T07:28:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3ecb61b3e6e34d3666f9b688cd694d92c5fb211cf8c420fa876f7ccf5789154a
    source_path: gateway/local-models.md
    workflow: 15
---

# النماذج المحلية

الاستخدام المحلي ممكن، لكن OpenClaw يتوقع سياقًا كبيرًا + وسائل دفاع قوية ضد حقن الأوامر. البطاقات الصغيرة تقطع السياق وتُضعف الأمان. استهدف مواصفات عالية: **≥2 من أجهزة Mac Studio بأقصى تجهيز أو ما يعادلها من منصات GPU (~$30k+)**. تعمل بطاقة GPU واحدة بسعة **24 GB** فقط مع الطلبات الأخف وبزمن استجابة أعلى. استخدم **أكبر / النسخة كاملة الحجم من النموذج التي يمكنك تشغيلها**؛ فالإصدارات المضغوطة بشدة أو نقاط التحقق “الصغيرة” تزيد من خطر حقن الأوامر (راجع [الأمان](/ar/gateway/security)).

إذا كنت تريد أقل إعداد محلي تعقيدًا، فابدأ بـ [LM Studio](/ar/providers/lmstudio) أو [Ollama](/ar/providers/ollama) واستخدم `openclaw onboard`. هذه الصفحة هي الدليل التوجيهي الموصى به للبُنى المحلية الأعلى مستوى وخوادم OpenAI-compatible المحلية المخصصة.

## الموصى به: LM Studio + نموذج محلي كبير (Responses API)

أفضل بنية محلية حاليًا. حمّل نموذجًا كبيرًا في LM Studio (على سبيل المثال، إصدارًا كامل الحجم من Qwen أو DeepSeek أو Llama)، ثم فعّل الخادم المحلي (الافتراضي `http://127.0.0.1:1234`)، واستخدم Responses API لإبقاء الاستدلال منفصلًا عن النص النهائي.

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
- في LM Studio، نزّل **أكبر إصدار نموذج متاح** (تجنب الإصدارات “الصغيرة”/المضغوطة بشدة)، ثم شغّل الخادم وتأكد من أن `http://127.0.0.1:1234/v1/models` يعرضه.
- استبدل `my-local-model` بمعرّف النموذج الفعلي الظاهر في LM Studio.
- أبقِ النموذج محمّلًا؛ فالتحميل البارد يضيف زمن بدء.
- عدّل `contextWindow`/`maxTokens` إذا كان إصدار LM Studio لديك يختلف.
- بالنسبة إلى WhatsApp، التزم باستخدام Responses API بحيث يتم إرسال النص النهائي فقط.

أبقِ النماذج المستضافة مُعدّة حتى عند التشغيل المحلي؛ استخدم `models.mode: "merge"` حتى تظل البدائل الاحتياطية متاحة.

### إعداد هجين: أساسي مستضاف، واحتياطي محلي

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

بدّل ترتيب الأساسي والاحتياطي؛ وأبقِ كتلة providers نفسها و`models.mode: "merge"` حتى تتمكن من الرجوع إلى Sonnet أو Opus عندما يتوقف الجهاز المحلي.

### الاستضافة الإقليمية / توجيه البيانات

- تتوفر أيضًا إصدارات MiniMax/Kimi/GLM المستضافة على OpenRouter مع نقاط نهاية مقيّدة حسب المنطقة (مثل الاستضافة داخل الولايات المتحدة). اختر الإصدار الإقليمي هناك للحفاظ على حركة البيانات ضمن النطاق القضائي الذي تختاره، مع الاستمرار في استخدام `models.mode: "merge"` لبدائل Anthropic/OpenAI الاحتياطية.
- يظل التشغيل المحلي فقط هو أقوى خيار للخصوصية؛ أما التوجيه الإقليمي المستضاف فهو حل وسط عندما تحتاج إلى ميزات المزوّد لكنك تريد التحكم في تدفق البيانات.

## وكلاء محليون آخرون متوافقون مع OpenAI

يعمل vLLM وLiteLLM وOAI-proxy أو البوابات المخصصة إذا كانت توفّر نقطة نهاية `/v1` بأسلوب OpenAI. استبدل كتلة provider أعلاه بنقطة النهاية ومعرّف النموذج لديك:

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

أبقِ `models.mode: "merge"` حتى تظل النماذج المستضافة متاحة كبدائل احتياطية.

ملاحظة سلوكية لواجهات `/v1` المحلية/الوسيطة:

- يتعامل OpenClaw مع هذه المسارات على أنها مسارات وكيل متوافقة مع OpenAI، وليست
  نقاط نهاية OpenAI أصلية
- لا تنطبق هنا صياغة الطلب الخاصة بـ OpenAI الأصلية فقط: لا يوجد
  `service_tier`، ولا `store` الخاص بـ Responses، ولا صياغة حمولة
  التوافق مع reasoning في OpenAI، ولا تلميحات cache الخاصة بالطلبات
- لا يتم حقن ترويسات الإسناد المخفية الخاصة بـ OpenClaw (`originator`، `version`، `User-Agent`)
  على عناوين URL الخاصة بهذه الوكلاء المخصصة

ملاحظات التوافق للواجهات الأكثر تشددًا والمتوافقة مع OpenAI:

- تقبل بعض الخوادم فقط القيمة النصية `messages[].content` في Chat Completions، وليس
  مصفوفات أجزاء المحتوى المنظمة. اضبط
  `models.providers.<provider>.models[].compat.requiresStringContent: true` لتلك
  النقاط النهائية.
- بعض الواجهات المحلية الأصغر أو الأكثر تشددًا تكون غير مستقرة مع الشكل الكامل
  لطلب agent-runtime في OpenClaw، خاصة عند تضمين مخططات الأدوات. إذا كانت
  الواجهة تعمل مع استدعاءات `/v1/chat/completions` المباشرة الصغيرة لكنها تفشل مع
  الأدوار العادية للوكلاء في OpenClaw، فجرّب أولًا
  `models.providers.<provider>.models[].compat.supportsTools: false`.
- إذا استمرت الواجهة بالفشل فقط مع عمليات OpenClaw الأكبر، فغالبًا ما تكون المشكلة
  سعة النموذج/الخادم من الجهة العليا أو خطأ في الواجهة الخلفية، وليست طبقة النقل في OpenClaw.

## استكشاف الأخطاء وإصلاحها

- هل يستطيع Gateway الوصول إلى الوكيل؟ `curl http://127.0.0.1:1234/v1/models`.
- هل تم إلغاء تحميل نموذج LM Studio؟ أعد تحميله؛ فالبدء البارد سبب شائع لـ “التعليق”.
- أخطاء السياق؟ خفّض `contextWindow` أو ارفع حد الخادم لديك.
- هل يعيد خادم OpenAI-compatible الخطأ `messages[].content ... expected a string`؟
  أضف `compat.requiresStringContent: true` إلى إدخال ذلك النموذج.
- هل تعمل استدعاءات `/v1/chat/completions` المباشرة الصغيرة، لكن `openclaw infer model run`
  يفشل مع Gemma أو نموذج محلي آخر؟ عطّل مخططات الأدوات أولًا عبر
  `compat.supportsTools: false`، ثم اختبر مجددًا. إذا استمر الخادم في الانهيار فقط
  مع طلبات OpenClaw الأكبر، فاعتبر ذلك قيدًا من الخادم/النموذج في الجهة العليا.
- الأمان: تتجاوز النماذج المحلية عوامل التصفية من جهة المزوّد؛ أبقِ الوكلاء محدودي النطاق وفعّل Compaction لتقليل نطاق تأثير حقن الأوامر.
