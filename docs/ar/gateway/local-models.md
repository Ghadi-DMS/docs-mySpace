---
read_when:
    - تريد تقديم النماذج من جهاز GPU الخاص بك
    - أنت تقوم بربط LM Studio أو وكيل متوافق مع OpenAI
    - تحتاج إلى أكثر إرشادات النماذج المحلية أمانًا
summary: شغّل OpenClaw على نماذج LLM محلية (LM Studio وvLLM وLiteLLM ونقاط نهاية OpenAI مخصصة)
title: النماذج المحلية
x-i18n:
    generated_at: "2026-04-08T02:14:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: d619d72b0e06914ebacb7e9f38b746caf1b9ce8908c9c6638c3acdddbaa025e8
    source_path: gateway/local-models.md
    workflow: 15
---

# النماذج المحلية

التشغيل المحلي ممكن، لكن OpenClaw يتوقع سياقًا كبيرًا + وسائل دفاع قوية ضد حقن المطالبات. البطاقات الصغيرة تقتطع السياق وتسرّب الأمان. استهدف مستوى مرتفعًا: **استوديوهين Mac Studio ممتلئين بالمواصفات على الأقل أو جهاز GPU مكافئ (~$30k+)**. تعمل وحدة **24 GB** GPU واحدة فقط مع المطالبات الأخف وزمن استجابة أعلى. استخدم **أكبر / النسخة الكاملة من النموذج التي يمكنك تشغيلها**؛ إذ إن نقاط التحقق المكمّمة بشدة أو “الصغيرة” تزيد من خطر حقن المطالبات (راجع [الأمان](/ar/gateway/security)).

إذا كنت تريد أقل إعداد محلي احتكاكًا، فابدأ بـ [Ollama](/ar/providers/ollama) و`openclaw onboard`. هذه الصفحة هي الدليل العملي الموجّه للإعدادات المحلية الأعلى مستوى وخوادم OpenAI المحلية المخصصة المتوافقة.

## الموصى به: LM Studio + نموذج محلي كبير (Responses API)

أفضل بنية محلية حالية. حمّل نموذجًا كبيرًا في LM Studio (على سبيل المثال، إصدارًا كامل الحجم من Qwen أو DeepSeek أو Llama)، ثم فعّل الخادم المحلي (الافتراضي `http://127.0.0.1:1234`)، واستخدم Responses API لإبقاء الاستدلال منفصلًا عن النص النهائي.

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

**قائمة الإعداد**

- ثبّت LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- في LM Studio، نزّل **أكبر إصدار نموذج متاح** (تجنب المتغيرات “الصغيرة”/المكمّمة بشدة)، ثم شغّل الخادم، وتأكد من أن `http://127.0.0.1:1234/v1/models` يعرضه.
- استبدل `my-local-model` بمعرّف النموذج الفعلي الظاهر في LM Studio.
- أبقِ النموذج محمّلًا؛ فالتحميل البارد يضيف زمن تأخير عند البدء.
- عدّل `contextWindow`/`maxTokens` إذا كان إصدار LM Studio لديك مختلفًا.
- بالنسبة إلى WhatsApp، التزم بـ Responses API حتى يُرسَل النص النهائي فقط.

أبقِ النماذج المستضافة مضبوطة حتى عند التشغيل المحلي؛ استخدم `models.mode: "merge"` حتى تبقى خيارات الرجوع الاحتياطي متاحة.

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

بدّل ترتيب الأساسي والاحتياطي؛ وأبقِ كتلة providers نفسها و`models.mode: "merge"` حتى تتمكن من الرجوع إلى Sonnet أو Opus عندما يتعطل الجهاز المحلي.

### الاستضافة الإقليمية / توجيه البيانات

- تتوفر أيضًا متغيرات MiniMax/Kimi/GLM المستضافة على OpenRouter مع نقاط نهاية مثبّتة على مناطق محددة (مثل الاستضافة داخل الولايات المتحدة). اختر المتغير الإقليمي هناك للحفاظ على حركة المرور ضمن الولاية القضائية التي تختارها مع الاستمرار في استخدام `models.mode: "merge"` لخيارات الرجوع الاحتياطي من Anthropic/OpenAI.
- يظل التشغيل المحلي فقط هو المسار الأقوى للخصوصية؛ أما التوجيه الإقليمي المستضاف فهو الحل الوسط عندما تحتاج إلى ميزات المزوّد لكنك تريد التحكم في تدفق البيانات.

## وكلاء محليون آخرون متوافقون مع OpenAI

تعمل vLLM وLiteLLM وOAI-proxy أو البوابات المخصصة إذا كانت تعرض نقطة نهاية `/v1` بأسلوب OpenAI. استبدل كتلة provider أعلاه بنقطة النهاية ومعرّف النموذج الخاصين بك:

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

أبقِ `models.mode: "merge"` حتى تظل النماذج المستضافة متاحة كخيارات رجوع احتياطي.

ملاحظة سلوكية لواجهات `/v1` المحلية/الوسيطة:

- يتعامل OpenClaw مع هذه الواجهات على أنها مسارات وكيل متوافقة مع OpenAI، وليست
  نقاط نهاية OpenAI أصلية
- لا ينطبق هنا تشكيل الطلبات الخاص بـ OpenAI الأصلية فقط: لا
  يوجد `service_tier`، ولا `store` الخاص بـ Responses، ولا تشكيل حمولة
  التوافق مع الاستدلال الخاص بـ OpenAI، ولا تلميحات لذاكرة التخزين المؤقت للمطالبات
- لا يتم حقن ترويسات الإسناد المخفية الخاصة بـ OpenClaw (`originator` و`version` و`User-Agent`)
  على عناوين URL الخاصة بهذه الوكلاء المخصصة

ملاحظات التوافق لواجهات OpenAI المتوافقة الأكثر صرامة:

- تقبل بعض الخوادم فقط `messages[].content` كسلسلة نصية في Chat Completions، وليس
  مصفوفات أجزاء المحتوى المهيكلة. اضبط
  `models.providers.<provider>.models[].compat.requiresStringContent: true` لهذه
  النقاط.
- بعض الواجهات المحلية الأصغر أو الأكثر صرامة تكون غير مستقرة مع الشكل الكامل
  لمطالبة بيئة تشغيل الوكيل في OpenClaw، خاصة عند تضمين مخططات الأدوات. إذا كانت
  الواجهة تعمل مع استدعاءات `/v1/chat/completions` المباشرة الصغيرة ولكنها تفشل في
  الأدوار العادية لوكيل OpenClaw، فجرّب
  `models.providers.<provider>.models[].compat.supportsTools: false` أولًا.
- إذا كانت الواجهة لا تزال تفشل فقط في تشغيلات OpenClaw الأكبر، فعادةً ما تكون
  المشكلة المتبقية في سعة النموذج/الخادم upstream أو في خطأ في الواجهة الخلفية،
  وليس في طبقة النقل الخاصة بـ OpenClaw.

## استكشاف الأخطاء وإصلاحها

- هل تستطيع البوابة الوصول إلى الوكيل؟ `curl http://127.0.0.1:1234/v1/models`.
- هل تم إلغاء تحميل نموذج LM Studio؟ أعد تحميله؛ فالبدء البارد سبب شائع لـ “التعليق”.
- أخطاء في السياق؟ خفّض `contextWindow` أو ارفع حد الخادم.
- هل يعيد الخادم المتوافق مع OpenAI الخطأ `messages[].content ... expected a string`؟
  أضف `compat.requiresStringContent: true` إلى إدخال ذلك النموذج.
- تعمل استدعاءات `/v1/chat/completions` المباشرة الصغيرة، لكن `openclaw infer model run`
  يفشل على Gemma أو نموذج محلي آخر؟ عطّل مخططات الأدوات أولًا عبر
  `compat.supportsTools: false`، ثم أعد الاختبار. إذا استمر الخادم في التعطل فقط
  مع مطالبات OpenClaw الأكبر، فاعتبر ذلك قيدًا في الخادم/النموذج upstream.
- الأمان: تتجاوز النماذج المحلية عوامل التصفية الموجودة لدى المزوّد؛ أبقِ الوكلاء محدودي النطاق وفعّل الضغط لتقليل نطاق تأثير حقن المطالبات.
