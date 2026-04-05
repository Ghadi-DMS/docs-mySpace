---
read_when:
    - تريد تشغيل OpenClaw مع نماذج سحابية أو محلية عبر Ollama
    - تحتاج إلى إرشادات إعداد وتهيئة Ollama
summary: تشغيل OpenClaw مع Ollama ‏(النماذج السحابية والمحلية)
title: Ollama
x-i18n:
    generated_at: "2026-04-05T12:54:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 337b8ec3a7756e591e6d6f82e8ad13417f0f20c394ec540e8fc5756e0fc13c29
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama هي بيئة تشغيل LLM محلية تجعل من السهل تشغيل النماذج مفتوحة المصدر على جهازك. يتكامل OpenClaw مع API الأصلية لـ Ollama ‏(`/api/chat`)، ويدعم البث واستدعاء الأدوات، ويمكنه الاكتشاف التلقائي لنماذج Ollama المحلية عندما تفعّل ذلك باستخدام `OLLAMA_API_KEY` ‏(أو ملف تعريف مصادقة) ولا تعرّف إدخالًا صريحًا لـ `models.providers.ollama`.

<Warning>
**مستخدمو Ollama البعيدة**: لا تستخدم عنوان URL المتوافق مع OpenAI لمسار `/v1` ‏(`http://host:11434/v1`) مع OpenClaw. فهذا يكسر استدعاء الأدوات، وقد تخرج النماذج JSON الأدوات الخام كنص عادي. استخدم بدلًا من ذلك عنوان API الأصلي لـ Ollama: ‏`baseUrl: "http://host:11434"` ‏(من دون `/v1`).
</Warning>

## بدء سريع

### Onboarding ‏(موصى به)

أسرع طريقة لإعداد Ollama هي عبر onboarding:

```bash
openclaw onboard
```

اختر **Ollama** من قائمة الموفّرين. ستقوم onboarding بما يلي:

1. ستسأل عن base URL الخاص بـ Ollama الذي يمكن الوصول إلى مثيلك من خلاله (الافتراضي `http://127.0.0.1:11434`).
2. ستتيح لك الاختيار بين **Cloud + Local** ‏(النماذج السحابية والنماذج المحلية) أو **Local** ‏(النماذج المحلية فقط).
3. ستفتح تدفق تسجيل دخول في المتصفح إذا اخترت **Cloud + Local** ولم تكن قد سجلت الدخول إلى ollama.com.
4. ستكتشف النماذج المتاحة وتقترح القيم الافتراضية.
5. ستنفذ `pull` للنموذج المحدد تلقائيًا إذا لم يكن متاحًا محليًا.

كما يُدعم الوضع غير التفاعلي:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

يمكنك اختياريًا تحديد base URL أو نموذج مخصص:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

### الإعداد اليدوي

1. ثبّت Ollama: ‏[https://ollama.com/download](https://ollama.com/download)

2. نفّذ `pull` لنموذج محلي إذا كنت تريد استدلالًا محليًا:

```bash
ollama pull glm-4.7-flash
# أو
ollama pull gpt-oss:20b
# أو
ollama pull llama3.3
```

3. إذا كنت تريد النماذج السحابية أيضًا، فسجّل الدخول:

```bash
ollama signin
```

4. شغّل onboarding واختر `Ollama`:

```bash
openclaw onboard
```

- `Local`: النماذج المحلية فقط
- `Cloud + Local`: النماذج المحلية بالإضافة إلى النماذج السحابية
- النماذج السحابية مثل `kimi-k2.5:cloud` و`minimax-m2.5:cloud` و`glm-5:cloud` **لا** تتطلب تنفيذ `ollama pull` محليًا

يقترح OpenClaw حاليًا:

- الافتراضي المحلي: `glm-4.7-flash`
- القيم الافتراضية السحابية: `kimi-k2.5:cloud` و`minimax-m2.5:cloud` و`glm-5:cloud`

5. إذا كنت تفضل الإعداد اليدوي، ففعّل Ollama لـ OpenClaw مباشرة (أي قيمة تعمل؛ لا تتطلب Ollama مفتاحًا حقيقيًا):

```bash
# تعيين متغير البيئة
export OLLAMA_API_KEY="ollama-local"

# أو التهيئة في ملف الإعداد
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. افحص النماذج أو بدّل بينها:

```bash
openclaw models list
openclaw models set ollama/glm-4.7-flash
```

7. أو اضبط القيمة الافتراضية في الإعداد:

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/glm-4.7-flash" },
    },
  },
}
```

## اكتشاف النماذج (موفّر ضمني)

عندما تضبط `OLLAMA_API_KEY` ‏(أو ملف تعريف مصادقة) و**لا** تعرّف `models.providers.ollama`، يكتشف OpenClaw النماذج من مثيل Ollama المحلي على `http://127.0.0.1:11434`:

- يستعلم `/api/tags`
- يستخدم عمليات lookup من نوع `/api/show` بأفضل جهد لقراءة `contextWindow` عند التوفر
- يوسم `reasoning` باستخدام heuristics لأسماء النماذج (`r1` أو `reasoning` أو `think`)
- يضبط `maxTokens` على الحد الأقصى الافتراضي للرموز في Ollama الذي يستخدمه OpenClaw
- يضبط جميع التكاليف على `0`

وهذا يتجنب إدخالات النماذج اليدوية مع إبقاء الفهرس متوافقًا مع مثيل Ollama المحلي.

لرؤية النماذج المتاحة:

```bash
ollama list
openclaw models list
```

لإضافة نموذج جديد، ما عليك سوى تنفيذ `pull` له باستخدام Ollama:

```bash
ollama pull mistral
```

سيتم اكتشاف النموذج الجديد تلقائيًا وسيصبح متاحًا للاستخدام.

إذا ضبطت `models.providers.ollama` صراحة، فسيتم تخطي الاكتشاف التلقائي وسيتعين عليك تعريف النماذج يدويًا (راجع أدناه).

## الإعداد

### إعداد أساسي (اكتشاف ضمني)

أبسط طريقة لتمكين Ollama هي عبر متغير البيئة:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### إعداد صريح (نماذج يدوية)

استخدم الإعداد الصريح عندما:

- تعمل Ollama على مضيف/منفذ آخر.
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

إذا كان `OLLAMA_API_KEY` مضبوطًا، يمكنك حذف `apiKey` من إدخال الموفّر وسيملؤه OpenClaw من أجل فحوصات التوافر.

### Base URL مخصص (إعداد صريح)

إذا كانت Ollama تعمل على مضيف أو منفذ مختلفين (يؤدي الإعداد الصريح إلى تعطيل الاكتشاف التلقائي، لذا عرّف النماذج يدويًا):

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // بدون /v1 - استخدم عنوان API الأصلي لـ Ollama
        api: "ollama", // اضبطه صراحة لضمان سلوك استدعاء الأدوات الأصلي
      },
    },
  },
}
```

<Warning>
لا تضف `/v1` إلى عنوان URL. فالمسار `/v1` يستخدم الوضع المتوافق مع OpenAI، حيث لا يكون استدعاء الأدوات موثوقًا. استخدم عنوان Ollama الأساسي من دون لاحقة مسار.
</Warning>

### اختيار النموذج

بمجرد التهيئة، تصبح كل نماذج Ollama لديك متاحة:

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

تتيح لك النماذج السحابية تشغيل نماذج مستضافة سحابيًا (مثل `kimi-k2.5:cloud` و`minimax-m2.5:cloud` و`glm-5:cloud`) إلى جانب نماذجك المحلية.

لاستخدام النماذج السحابية، اختر وضع **Cloud + Local** أثناء الإعداد. ويتحقق المعالج مما إذا كنت مسجل الدخول ويفتح تدفق تسجيل دخول في المتصفح عند الحاجة. وإذا تعذّر التحقق من المصادقة، يعود المعالج إلى القيم الافتراضية للنماذج المحلية.

يمكنك أيضًا تسجيل الدخول مباشرة على [ollama.com/signin](https://ollama.com/signin).

## Ollama Web Search

يدعم OpenClaw أيضًا **Ollama Web Search** بوصفه موفّر `web_search`
مدمجًا.

- يستخدم مضيف Ollama المهيأ لديك (`models.providers.ollama.baseUrl` عندما
  يكون مضبوطًا، وإلا `http://127.0.0.1:11434`).
- لا يحتاج إلى مفتاح.
- يتطلب أن تكون Ollama قيد التشغيل وأن تكون قد سجلت الدخول عبر `ollama signin`.

اختر **Ollama Web Search** أثناء `openclaw onboard` أو
`openclaw configure --section web`، أو اضبط:

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

للحصول على التفاصيل الكاملة الخاصة بالإعداد والسلوك، راجع [Ollama Web Search](/tools/ollama-search).

## متقدم

### نماذج الاستدلال

يتعامل OpenClaw افتراضيًا مع النماذج التي تحمل أسماء مثل `deepseek-r1` أو `reasoning` أو `think` على أنها قادرة على الاستدلال:

```bash
ollama pull deepseek-r1:32b
```

### تكاليف النماذج

Ollama مجانية وتعمل محليًا، لذلك تُضبط جميع تكاليف النماذج على $0.

### تهيئة البث

يستخدم تكامل OpenClaw مع Ollama **API الأصلية لـ Ollama** ‏(`/api/chat`) افتراضيًا، والتي تدعم بالكامل البث واستدعاء الأدوات في الوقت نفسه. ولا حاجة إلى أي إعداد خاص.

#### الوضع القديم المتوافق مع OpenAI

<Warning>
**استدعاء الأدوات ليس موثوقًا في الوضع المتوافق مع OpenAI.** استخدم هذا الوضع فقط إذا كنت تحتاج إلى تنسيق OpenAI من أجل proxy ولا تعتمد على سلوك استدعاء الأدوات الأصلي.
</Warning>

إذا كنت تحتاج إلى استخدام نقطة النهاية المتوافقة مع OpenAI بدلًا من ذلك (مثلًا خلف proxy لا يدعم إلا تنسيق OpenAI)، فاضبط `api: "openai-completions"` صراحة:

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

قد لا يدعم هذا الوضع البث + استدعاء الأدوات في وقت واحد. وقد تحتاج إلى تعطيل البث باستخدام `params: { streaming: false }` في إعداد النموذج.

عندما يُستخدم `api: "openai-completions"` مع Ollama، يحقن OpenClaw القيمة `options.num_ctx` افتراضيًا حتى لا تعود Ollama بصمت إلى نافذة سياق مقدارها 4096. وإذا كان proxy/upstream لديك يرفض حقول `options` غير المعروفة، فقم بتعطيل هذا السلوك:

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

بالنسبة إلى النماذج المكتشفة تلقائيًا، يستخدم OpenClaw نافذة السياق التي يبلّغ عنها Ollama عندما تكون متاحة، وإلا فإنه يعود إلى نافذة سياق Ollama الافتراضية التي يستخدمها OpenClaw. ويمكنك تجاوز `contextWindow` و`maxTokens` في إعداد الموفّر الصريح.

## استكشاف الأخطاء وإصلاحها

### لم يتم اكتشاف Ollama

تأكد من أن Ollama قيد التشغيل وأنك قد ضبطت `OLLAMA_API_KEY` ‏(أو ملف تعريف مصادقة)، وأنك **لم** تعرّف إدخالًا صريحًا لـ `models.providers.ollama`:

```bash
ollama serve
```

وتأكد من أن API قابلة للوصول:

```bash
curl http://localhost:11434/api/tags
```

### لا توجد نماذج متاحة

إذا لم يكن نموذجك مدرجًا، فإما:

- أن تنفّذ `pull` للنموذج محليًا، أو
- أن تعرّف النموذج صراحة في `models.providers.ollama`.

لإضافة نماذج:

```bash
ollama list  # عرض ما هو مثبت
ollama pull glm-4.7-flash
ollama pull gpt-oss:20b
ollama pull llama3.3     # أو نموذج آخر
```

### تم رفض الاتصال

تحقق من أن Ollama تعمل على المنفذ الصحيح:

```bash
# تحقق مما إذا كانت Ollama تعمل
ps aux | grep ollama

# أو أعد تشغيل Ollama
ollama serve
```

## راجع أيضًا

- [موفرو النماذج](/concepts/model-providers) - نظرة عامة على جميع الموفّرين
- [اختيار النموذج](/concepts/models) - كيفية اختيار النماذج
- [التهيئة](/gateway/configuration) - المرجع الكامل للإعداد
