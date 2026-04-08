---
read_when:
    - تريد تشغيل OpenClaw مع نماذج سحابية أو محلية عبر Ollama
    - تحتاج إلى إرشادات إعداد وتكوين Ollama
summary: تشغيل OpenClaw مع Ollama (النماذج السحابية والمحلية)
title: Ollama
x-i18n:
    generated_at: "2026-04-08T02:19:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 222ec68f7d4bb29cc7796559ddef1d5059f5159e7a51e2baa3a271ddb3abb716
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama هو بيئة تشغيل محلية لـ LLM تجعل من السهل تشغيل النماذج مفتوحة المصدر على جهازك. يتكامل OpenClaw مع API الأصلي لـ Ollama (`/api/chat`)، ويدعم البث واستدعاء الأدوات، ويمكنه اكتشاف نماذج Ollama المحلية تلقائيًا عندما تختار ذلك عبر `OLLAMA_API_KEY` (أو ملف تعريف مصادقة) ولا تعرّف إدخال `models.providers.ollama` صريحًا.

<Warning>
**مستخدمو Ollama البعيد**: لا تستخدم عنوان URL المتوافق مع OpenAI الخاص بـ `/v1` (`http://host:11434/v1`) مع OpenClaw. فهذا يعطّل استدعاء الأدوات وقد تُخرج النماذج JSON الأدوات الخام كنص عادي. استخدم بدلًا من ذلك عنوان URL الخاص بـ API الأصلي لـ Ollama: ‏`baseUrl: "http://host:11434"` (من دون `/v1`).
</Warning>

## البدء السريع

### الإعداد الأولي (مستحسن)

أسرع طريقة لإعداد Ollama هي عبر الإعداد الأولي:

```bash
openclaw onboard
```

اختر **Ollama** من قائمة الموفّرين. سيقوم الإعداد الأولي بما يلي:

1. سيسألك عن عنوان URL الأساسي لـ Ollama الذي يمكن الوصول إلى المثيل من خلاله (الافتراضي `http://127.0.0.1:11434`).
2. يتيح لك اختيار **Cloud + Local** (نماذج سحابية ونماذج محلية) أو **Local** (نماذج محلية فقط).
3. يفتح تدفق تسجيل دخول عبر المتصفح إذا اخترت **Cloud + Local** ولم تكن قد سجلت الدخول إلى ollama.com.
4. يكتشف النماذج المتاحة ويقترح الإعدادات الافتراضية.
5. يسحب النموذج المحدد تلقائيًا إذا لم يكن متاحًا محليًا.

الوضع غير التفاعلي مدعوم أيضًا:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

يمكنك اختياريًا تحديد عنوان URL أساسي مخصص أو نموذج مخصص:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

### الإعداد اليدوي

1. ثبّت Ollama: [https://ollama.com/download](https://ollama.com/download)

2. اسحب نموذجًا محليًا إذا كنت تريد استدلالًا محليًا:

```bash
ollama pull gemma4
# أو
ollama pull gpt-oss:20b
# أو
ollama pull llama3.3
```

3. إذا كنت تريد النماذج السحابية أيضًا، فسجّل الدخول:

```bash
ollama signin
```

4. شغّل الإعداد الأولي واختر `Ollama`:

```bash
openclaw onboard
```

- `Local`: نماذج محلية فقط
- `Cloud + Local`: نماذج محلية بالإضافة إلى نماذج سحابية
- النماذج السحابية مثل `kimi-k2.5:cloud` و`minimax-m2.7:cloud` و`glm-5.1:cloud` لا **تتطلب** `ollama pull` محليًا

يقترح OpenClaw حاليًا:

- الافتراضي المحلي: `gemma4`
- الإعدادات الافتراضية السحابية: `kimi-k2.5:cloud` و`minimax-m2.7:cloud` و`glm-5.1:cloud`

5. إذا كنت تفضّل الإعداد اليدوي، فقم بتمكين Ollama مباشرة لـ OpenClaw (أي قيمة تعمل؛ لا يتطلب Ollama مفتاحًا حقيقيًا):

```bash
# تعيين متغير البيئة
export OLLAMA_API_KEY="ollama-local"

# أو التكوين في ملف التكوين
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. افحص النماذج أو بدّل بينها:

```bash
openclaw models list
openclaw models set ollama/gemma4
```

7. أو عيّن الإعداد الافتراضي في التكوين:

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/gemma4" },
    },
  },
}
```

## اكتشاف النماذج (الموفّر الضمني)

عندما تعيّن `OLLAMA_API_KEY` (أو ملف تعريف مصادقة) و**لا** تعرّف `models.providers.ollama`، يكتشف OpenClaw النماذج من مثيل Ollama المحلي على `http://127.0.0.1:11434`:

- يستعلم عن `/api/tags`
- يستخدم عمليات بحث `/api/show` بأفضل جهد لقراءة `contextWindow` عند توفره
- يضع علامة `reasoning` باستخدام استدلال يعتمد على اسم النموذج (`r1` أو `reasoning` أو `think`)
- يضبط `maxTokens` على الحد الأقصى الافتراضي للرموز في Ollama المستخدم من قبل OpenClaw
- يضبط جميع التكاليف على `0`

يؤدي ذلك إلى تجنب إدخالات النماذج اليدوية مع إبقاء الكتالوج متوافقًا مع مثيل Ollama المحلي.

لمعرفة النماذج المتاحة:

```bash
ollama list
openclaw models list
```

لإضافة نموذج جديد، ما عليك سوى سحبه عبر Ollama:

```bash
ollama pull mistral
```

سيتم اكتشاف النموذج الجديد تلقائيًا وسيصبح متاحًا للاستخدام.

إذا عيّنت `models.providers.ollama` صراحة، فسيتم تخطي الاكتشاف التلقائي، ويجب عليك تعريف النماذج يدويًا (انظر أدناه).

## التكوين

### إعداد أساسي (اكتشاف ضمني)

أبسط طريقة لتمكين Ollama هي عبر متغير البيئة:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### إعداد صريح (نماذج يدوية)

استخدم التكوين الصريح عندما:

- يعمل Ollama على مضيف/منفذ آخر.
- تريد فرض نوافذ سياق أو قوائم نماذج محددة.
- تريد تعريفات نماذج يدوية بالكامل.

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "gpt-oss:20b",
            name: "GPT-OSS 20B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192 * 10
          }
        ]
      }
    }
  }
}
```

إذا كان `OLLAMA_API_KEY` معيّنًا، فيمكنك حذف `apiKey` من إدخال الموفّر وسيقوم OpenClaw بملئه من أجل فحوصات التوفر.

### عنوان URL أساسي مخصص (تكوين صريح)

إذا كان Ollama يعمل على مضيف أو منفذ مختلفين (يعطّل التكوين الصريح الاكتشاف التلقائي، لذا عرّف النماذج يدويًا):

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // بدون /v1 - استخدم عنوان URL الخاص بـ API الأصلي لـ Ollama
        api: "ollama", // عيّنه صراحة لضمان سلوك استدعاء الأدوات الأصلي
      },
    },
  },
}
```

<Warning>
لا تضف `/v1` إلى عنوان URL. يستخدم المسار `/v1` الوضع المتوافق مع OpenAI، حيث لا يكون استدعاء الأدوات موثوقًا. استخدم عنوان URL الأساسي لـ Ollama من دون لاحقة مسار.
</Warning>

### اختيار النموذج

بمجرد التكوين، ستكون جميع نماذج Ollama الخاصة بك متاحة:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## النماذج السحابية

تتيح لك النماذج السحابية تشغيل نماذج مستضافة سحابيًا (على سبيل المثال `kimi-k2.5:cloud` و`minimax-m2.7:cloud` و`glm-5.1:cloud`) إلى جانب نماذجك المحلية.

لاستخدام النماذج السحابية، اختر وضع **Cloud + Local** أثناء الإعداد. يتحقق المعالج مما إذا كنت قد سجلت الدخول ويفتح تدفق تسجيل دخول عبر المتصفح عند الحاجة. إذا تعذر التحقق من المصادقة، فسيعود المعالج إلى الإعدادات الافتراضية للنماذج المحلية.

يمكنك أيضًا تسجيل الدخول مباشرة على [ollama.com/signin](https://ollama.com/signin).

## Ollama Web Search

يدعم OpenClaw أيضًا **Ollama Web Search** كموفّر
`web_search` مضمّن.

- يستخدم مضيف Ollama المهيأ لديك (`models.providers.ollama.baseUrl` عند
  تعيينه، وإلا `http://127.0.0.1:11434`).
- لا يحتاج إلى مفتاح.
- يتطلب أن يكون Ollama قيد التشغيل وأن تكون قد سجلت الدخول عبر `ollama signin`.

اختر **Ollama Web Search** أثناء `openclaw onboard` أو
`openclaw configure --section web`، أو عيّن:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

للحصول على تفاصيل الإعداد والسلوك الكاملة، راجع [Ollama Web Search](/ar/tools/ollama-search).

## متقدم

### نماذج الاستدلال

يعامل OpenClaw النماذج ذات الأسماء مثل `deepseek-r1` أو `reasoning` أو `think` على أنها قادرة على الاستدلال افتراضيًا:

```bash
ollama pull deepseek-r1:32b
```

### تكاليف النماذج

Ollama مجاني ويعمل محليًا، لذا يتم تعيين جميع تكاليف النماذج إلى $0.

### تكوين البث

يستخدم تكامل OpenClaw مع Ollama **API الأصلي لـ Ollama** (`/api/chat`) افتراضيًا، وهو يدعم بالكامل البث واستدعاء الأدوات في الوقت نفسه. لا يلزم أي تكوين خاص.

#### الوضع القديم المتوافق مع OpenAI

<Warning>
**استدعاء الأدوات ليس موثوقًا في الوضع المتوافق مع OpenAI.** استخدم هذا الوضع فقط إذا كنت بحاجة إلى تنسيق OpenAI من أجل وكيل ولا تعتمد على سلوك استدعاء الأدوات الأصلي.
</Warning>

إذا كنت بحاجة إلى استخدام نقطة النهاية المتوافقة مع OpenAI بدلًا من ذلك (مثلًا خلف وكيل لا يدعم إلا تنسيق OpenAI)، فعيّن `api: "openai-completions"` صراحة:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: true, // الافتراضي: true
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

قد لا يدعم هذا الوضع البث + استدعاء الأدوات في الوقت نفسه. قد تحتاج إلى تعطيل البث باستخدام `params: { streaming: false }` في تكوين النموذج.

عند استخدام `api: "openai-completions"` مع Ollama، يحقن OpenClaw القيمة `options.num_ctx` افتراضيًا حتى لا يعود Ollama بصمت إلى نافذة سياق قدرها 4096. إذا كان وكيلك/خادمك الصاعد يرفض حقول `options` غير المعروفة، فعطّل هذا السلوك:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: false,
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

### نوافذ السياق

بالنسبة إلى النماذج المكتشفة تلقائيًا، يستخدم OpenClaw نافذة السياق التي يبلغ عنها Ollama عند توفرها، وإلا فإنه يعود إلى نافذة سياق Ollama الافتراضية المستخدمة من قبل OpenClaw. يمكنك تجاوز `contextWindow` و`maxTokens` في تكوين الموفّر الصريح.

## استكشاف الأخطاء وإصلاحها

### لم يتم اكتشاف Ollama

تأكد من أن Ollama قيد التشغيل وأنك عيّنت `OLLAMA_API_KEY` (أو ملف تعريف مصادقة)، وأنك **لم** تعرّف إدخال `models.providers.ollama` صريحًا:

```bash
ollama serve
```

وتأكد من أن API يمكن الوصول إليه:

```bash
curl http://localhost:11434/api/tags
```

### لا توجد نماذج متاحة

إذا لم يكن نموذجك مدرجًا، فإما أن:

- تسحب النموذج محليًا، أو
- تعرّف النموذج صراحة في `models.providers.ollama`.

لإضافة نماذج:

```bash
ollama list  # اعرض ما هو مثبّت
ollama pull gemma4
ollama pull gpt-oss:20b
ollama pull llama3.3     # أو نموذج آخر
```

### تم رفض الاتصال

تحقق من أن Ollama يعمل على المنفذ الصحيح:

```bash
# تحقق مما إذا كان Ollama قيد التشغيل
ps aux | grep ollama

# أو أعد تشغيل Ollama
ollama serve
```

## راجع أيضًا

- [موفّرات النماذج](/ar/concepts/model-providers) - نظرة عامة على جميع الموفّرين
- [اختيار النموذج](/ar/concepts/models) - كيفية اختيار النماذج
- [التكوين](/ar/gateway/configuration) - المرجع الكامل للتكوين
