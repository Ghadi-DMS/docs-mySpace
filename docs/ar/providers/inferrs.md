---
read_when:
    - تريد تشغيل OpenClaw مقابل خادم inferrs محلي
    - أنت تقدّم Gemma أو نموذجًا آخر عبر inferrs
    - تحتاج إلى علامات التوافق الدقيقة في OpenClaw لـ inferrs
summary: تشغيل OpenClaw عبر inferrs ‏(خادم محلي متوافق مع OpenAI)
title: inferrs
x-i18n:
    generated_at: "2026-04-08T02:18:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: d84f660d49a682d0c0878707eebe1bc1e83dd115850687076ea3938b9f9c86c6
    source_path: providers/inferrs.md
    workflow: 15
---

# inferrs

يمكن لـ [inferrs](https://github.com/ericcurtin/inferrs) تقديم نماذج محلية خلف
واجهة `/v1` API متوافقة مع OpenAI. يعمل OpenClaw مع `inferrs` عبر
مسار `openai-completions` العام.

من الأفضل حاليًا التعامل مع `inferrs` على أنه واجهة خلفية مخصصة
ذاتية الاستضافة متوافقة مع OpenAI، وليس plugin موفر مخصصًا في OpenClaw.

## بداية سريعة

1. ابدأ `inferrs` مع نموذج.

مثال:

```bash
inferrs serve gg-hf-gg/gemma-4-E2B-it \
  --host 127.0.0.1 \
  --port 8080 \
  --device metal
```

2. تحقّق من أن الخادم قابل للوصول.

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/v1/models
```

3. أضف إدخال موفر صريحًا في OpenClaw ووجّه النموذج الافتراضي إليه.

## مثال إعداد كامل

يستخدم هذا المثال Gemma 4 على خادم `inferrs` محلي.

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/gg-hf-gg/gemma-4-E2B-it" },
      models: {
        "inferrs/gg-hf-gg/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "gg-hf-gg/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## لماذا يهم `requiresStringContent`

تقبل بعض مسارات Chat Completions في `inferrs` فقط
`messages[].content` كسلسلة نصية، وليس كمصفوفات منظَّمة لأجزاء المحتوى.

إذا فشلت تشغيلات OpenClaw مع خطأ مثل:

```text
messages[1].content: invalid type: sequence, expected a string
```

فضبط:

```json5
compat: {
  requiresStringContent: true
}
```

سيقوم OpenClaw بتسطيح أجزاء المحتوى النصية الخالصة إلى سلاسل نصية عادية قبل إرسال
الطلب.

## ملاحظة Gemma ومخطط الأدوات

تقبل بعض التركيبات الحالية من `inferrs` + Gemma طلبات
`/v1/chat/completions` المباشرة الصغيرة، لكنها لا تزال تفشل في أدوار
runtime الكاملة للوكيل في OpenClaw.

إذا حدث ذلك، فجرّب هذا أولًا:

```json5
compat: {
  requiresStringContent: true,
  supportsTools: false
}
```

يعطّل هذا سطح مخطط الأدوات في OpenClaw لهذا النموذج ويمكن أن يقلل ضغط
الموجّه على الواجهات الخلفية المحلية الأكثر صرامة.

إذا ظلت الطلبات المباشرة الصغيرة تعمل لكن أدوار الوكيل العادية في OpenClaw لا تزال
تتعطل داخل `inferrs`، فعادةً ما تكون المشكلة المتبقية
سلوكًا من النموذج/الخادم upstream وليس من طبقة النقل في OpenClaw.

## اختبار smoke يدوي

بعد الإعداد، اختبر الطبقتين كلتيهما:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"gg-hf-gg/gemma-4-E2B-it","messages":[{"role":"user","content":"What is 2 + 2?"}],"stream":false}'

openclaw infer model run \
  --model inferrs/gg-hf-gg/gemma-4-E2B-it \
  --prompt "What is 2 + 2? Reply with one short sentence." \
  --json
```

إذا نجح الأمر الأول لكن فشل الثاني، فاستخدم ملاحظات استكشاف الأخطاء وإصلاحها
أدناه.

## استكشاف الأخطاء وإصلاحها

- فشل `curl /v1/models`: `inferrs` لا يعمل، أو يتعذر الوصول إليه، أو أنه
  غير مربوط بالمضيف/المنفذ المتوقع.
- `messages[].content ... expected a string`: اضبط
  `compat.requiresStringContent: true`.
- تنجح استدعاءات `/v1/chat/completions` المباشرة الصغيرة، لكن يفشل `openclaw infer model run`:
  جرّب `compat.supportsTools: false`.
- لم يعد OpenClaw يتلقى أخطاء مخطط، لكن `inferrs` لا يزال يتعطل في أدوار
  الوكيل الأكبر: تعامل مع ذلك على أنه قيد في `inferrs` أو في النموذج من upstream وخفف
  ضغط الموجّه أو بدّل الواجهة الخلفية/النموذج المحلي.

## سلوك على نمط الوكيل

يُعامَل `inferrs` على أنه واجهة خلفية `/v1` متوافقة مع OpenAI على نمط الوكيل، وليس
كنقطة نهاية OpenAI أصلية.

- لا ينطبق هنا تشكيل الطلبات الخاص بـ OpenAI الأصلي فقط
- لا يوجد `service_tier`، ولا Responses `store`، ولا تلميحات ذاكرة التخزين المؤقت للموجّه، ولا
  تشكيل حمولة توافق الاستدلال الخاصة بـ OpenAI
- لا يتم حقن ترويسات الإسناد المخفية في OpenClaw (`originator` و`version` و`User-Agent`)
  على عناوين `inferrs` الأساسية المخصصة

## راجع أيضًا

- [النماذج المحلية](/ar/gateway/local-models)
- [استكشاف أخطاء البوابة وإصلاحها](/ar/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)
- [موفرو النماذج](/ar/concepts/model-providers)
