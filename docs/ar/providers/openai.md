---
read_when:
    - أنت تريد استخدام نماذج OpenAI في OpenClaw
    - أنت تريد مصادقة اشتراك Codex بدلًا من مفاتيح API
summary: استخدم OpenAI عبر مفاتيح API أو اشتراك Codex في OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-05T12:54:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: 537119853503d398f9136170ac12ecfdbd9af8aef3c4c011f8ada4c664bdaf6d
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

توفر OpenAI واجهات API للمطورين لنماذج GPT. ويدعم Codex **تسجيل الدخول إلى ChatGPT** للوصول
القائم على الاشتراك أو **تسجيل الدخول بمفتاح API** للوصول القائم على الاستخدام. ويتطلب Codex cloud تسجيل الدخول إلى ChatGPT.
وتدعم OpenAI صراحةً استخدام OAuth القائم على الاشتراك في الأدوات/مسارات العمل الخارجية مثل OpenClaw.

## أسلوب التفاعل الافتراضي

يضيف OpenClaw افتراضيًا طبقة موجّه صغيرة خاصة بـ OpenAI لكل من
تشغيلات `openai/*` و`openai-codex/*`. وتحافظ هذه الطبقة على المساعد
دافئًا، وتعاونيًا، ومختصرًا، ومباشرًا من دون استبدال موجّه النظام الأساسي
لـ OpenClaw.

مفتاح الإعداد:

`plugins.entries.openai.config.personalityOverlay`

القيم المسموح بها:

- `"friendly"`: الافتراضي؛ يفعّل الطبقة الخاصة بـ OpenAI.
- `"off"`: يعطل الطبقة ويستخدم موجّه OpenClaw الأساسي فقط.

النطاق:

- ينطبق على نماذج `openai/*`.
- ينطبق على نماذج `openai-codex/*`.
- لا يؤثر في الموفّرين الآخرين.

هذا السلوك مفعّل افتراضيًا:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personalityOverlay: "friendly",
        },
      },
    },
  },
}
```

### تعطيل طبقة موجّه OpenAI

إذا كنت تفضّل موجّه OpenClaw الأساسي غير المعدل، فعطّل الطبقة:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personalityOverlay: "off",
        },
      },
    },
  },
}
```

يمكنك أيضًا ضبطها مباشرةً عبر CLI الخاص بالإعداد:

```bash
openclaw config set plugins.entries.openai.config.personalityOverlay off
```

## الخيار A: مفتاح OpenAI API ‏(منصة OpenAI)

**الأفضل لـ:** الوصول المباشر إلى API والفوترة حسب الاستخدام.
احصل على مفتاح API من لوحة تحكم OpenAI.

### الإعداد عبر CLI

```bash
openclaw onboard --auth-choice openai-api-key
# or non-interactive
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### مقتطف الإعداد

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

تسرد وثائق نماذج API الحالية الخاصة بـ OpenAI الطرازين `gpt-5.4` و`gpt-5.4-pro` للاستخدام المباشر
مع OpenAI API. ويعيد OpenClaw توجيه كليهما عبر مسار `openai/*` في Responses.
كما يقوم OpenClaw عمدًا بإخفاء السطر القديم `openai/gpt-5.3-codex-spark`,
لأن استدعاءات OpenAI API المباشرة ترفضه في الحركة الحية.

لا يكشف OpenClaw عن `openai/gpt-5.3-codex-spark` على مسار OpenAI المباشر
لـ API. ولا يزال `pi-ai` يشحن سطرًا مضمّنًا لذلك النموذج، لكن طلبات OpenAI API الحية
ترفضه حاليًا. ويُعامل Spark على أنه خاص بـ Codex فقط في OpenClaw.

## الخيار B: اشتراك OpenAI Code ‏(Codex)

**الأفضل لـ:** استخدام وصول اشتراك ChatGPT/Codex بدلًا من مفتاح API.
يتطلب Codex cloud تسجيل الدخول إلى ChatGPT، بينما يدعم Codex CLI تسجيل الدخول عبر ChatGPT أو عبر مفتاح API.

### الإعداد عبر CLI ‏(Codex OAuth)

```bash
# Run Codex OAuth in the wizard
openclaw onboard --auth-choice openai-codex

# Or run OAuth directly
openclaw models auth login --provider openai-codex
```

### مقتطف الإعداد (اشتراك Codex)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

تسرد مستندات Codex الحالية الخاصة بـ OpenAI النموذج `gpt-5.4` بوصفه نموذج Codex الحالي. ويقوم OpenClaw
بربط ذلك إلى `openai-codex/gpt-5.4` من أجل استخدام ChatGPT/Codex OAuth.

إذا أعاد الإعداد الأولي استخدام تسجيل دخول Codex CLI موجود، فستبقى بيانات الاعتماد هذه
مدارة بواسطة Codex CLI. وعند انتهاء صلاحيتها، يعيد OpenClaw قراءة مصدر Codex الخارجي
أولًا، وعندما يستطيع الموفّر تحديثها، فإنه يكتب بيانات الاعتماد المحدثة
مرة أخرى إلى تخزين Codex بدلًا من امتلاك نسخة منفصلة خاصة بـ OpenClaw
فقط.

إذا كان حساب Codex الخاص بك مؤهلًا لـ Codex Spark، فإن OpenClaw يدعم أيضًا:

- `openai-codex/gpt-5.3-codex-spark`

يعامل OpenClaw ‏Codex Spark على أنه خاص بـ Codex فقط. وهو لا يكشف عن
مسار مباشر بمفتاح API بصيغة `openai/gpt-5.3-codex-spark`.

كما يحتفظ OpenClaw أيضًا بالقيمة `openai-codex/gpt-5.3-codex-spark` عندما
يكتشفها `pi-ai`. تعامل معها على أنها معتمدة على الاستحقاق وتجريبية: فـ Codex Spark
منفصل عن GPT-5.4 ‏`/fast`، ويتوقف توفره على حساب Codex / ChatGPT
المسجّل الدخول.

### حد نافذة السياق في Codex

يعامل OpenClaw البيانات الوصفية لنموذج Codex والحد الفعلي للسياق في وقت التشغيل على أنهما
قيمتان منفصلتان.

بالنسبة إلى `openai-codex/gpt-5.4`:

- `contextWindow` الأصلي: `1050000`
- الحد الافتراضي لـ `contextTokens` في وقت التشغيل: `272000`

يحافظ هذا على صدق البيانات الوصفية للنموذج مع إبقاء
النافذة الافتراضية الأصغر في وقت التشغيل التي تتميز عمليًا بزمن انتقال وجودة أفضل.

إذا كنت تريد حدًا فعليًا مختلفًا، فاضبط `models.providers.<provider>.models[].contextTokens`:

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

استخدم `contextWindow` فقط عندما تقوم بتعريف أو تجاوز البيانات الوصفية
الأصلية للنموذج. واستخدم `contextTokens` عندما تريد تقييد ميزانية السياق في وقت التشغيل.

### النقل الافتراضي

يستخدم OpenClaw ‏`pi-ai` لبث النموذج. وبالنسبة إلى كل من `openai/*` و
`openai-codex/*`، يكون النقل الافتراضي هو `"auto"` ‏(WebSocket أولًا، ثم
الرجوع إلى SSE).

في وضع `"auto"`، يعيد OpenClaw أيضًا محاولة فشل WebSocket واحد مبكر وقابل لإعادة المحاولة
قبل أن يعود إلى SSE. أما وضع `"websocket"` الإجباري فلا يزال يُظهر أخطاء
النقل مباشرةً بدلًا من إخفائها عبر الرجوع.

بعد فشل WebSocket عند الاتصال أو في دور مبكر في وضع `"auto"`، يضع OpenClaw
علامة على مسار WebSocket الخاص بتلك الجلسة على أنه degraded لمدة تقارب 60 ثانية ويرسل
الأدوار اللاحقة عبر SSE أثناء فترة التهدئة بدلًا من التبديل المستمر بين
طرق النقل.

وبالنسبة إلى نقاط النهاية الأصلية لعائلة OpenAI ‏(`openai/*`, `openai-codex/*`, وAzure
OpenAI Responses)، يرفق OpenClaw أيضًا حالة هوية ثابتة للجلسة والدور
بالطلبات بحيث تبقى عمليات إعادة المحاولة وإعادة الاتصال والرجوع إلى SSE متسقة مع هوية
المحادثة نفسها. وعلى المسارات الأصلية لعائلة OpenAI يتضمن هذا
رؤوس هوية ثابتة للطلب تخص الجلسة/الدور بالإضافة إلى بيانات نقل وصفية مطابقة.

كما يقوم OpenClaw أيضًا بتطبيع عدّادات استخدام OpenAI عبر متغيرات النقل قبل
أن تصل إلى أسطح الجلسة/الحالة. وقد تبلّغ حركة Responses الأصلية الخاصة بـ OpenAI/Codex
عن الاستخدام بصيغة `input_tokens` / `output_tokens` أو
`prompt_tokens` / `completion_tokens`؛ ويتعامل OpenClaw معها على أنها
عدادات الإدخال والإخراج نفسها بالنسبة إلى `/status` و`/usage` وسجلات الجلسات.
وعندما تحذف حركة WebSocket الأصلية القيمة `total_tokens` (أو تبلغ عنها بالقيمة `0`)، يعود OpenClaw إلى
إجمالي الإدخال + الإخراج بعد التطبيع حتى تبقى شاشات الجلسة/الحالة مليئة بالبيانات.

يمكنك ضبط `agents.defaults.models.<provider/model>.params.transport`:

- `"sse"`: فرض SSE
- `"websocket"`: فرض WebSocket
- `"auto"`: تجربة WebSocket، ثم الرجوع إلى SSE

وبالنسبة إلى `openai/*` ‏(Responses API)، يفعّل OpenClaw أيضًا تهيئة WebSocket
افتراضيًا (`openaiWsWarmup: true`) عند استخدام WebSocket كوسيلة نقل.

مستندات OpenAI ذات الصلة:

- [Realtime API مع WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [بث استجابات API ‏(SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### تهيئة OpenAI WebSocket

تصف مستندات OpenAI التهيئة على أنها اختيارية. ويفعّلها OpenClaw افتراضيًا لـ
`openai/*` لتقليل زمن الانتقال في الدور الأول عند استخدام نقل WebSocket.

### تعطيل التهيئة

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### تفعيل التهيئة صراحةً

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### المعالجة ذات الأولوية في OpenAI وCodex

تكشف واجهة API الخاصة بـ OpenAI المعالجة ذات الأولوية عبر `service_tier=priority`. وفي
OpenClaw، اضبط `agents.defaults.models["<provider>/<model>"].params.serviceTier`
لتمرير هذا الحقل على نقاط النهاية الأصلية لـ OpenAI/Codex Responses.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

القيم المدعومة هي `auto` و`default` و`flex` و`priority`.

يمرر OpenClaw القيمة `params.serviceTier` إلى كل من طلبات Responses المباشرة الخاصة بـ `openai/*`
وطلبات Codex Responses الخاصة بـ `openai-codex/*` عندما تشير تلك النماذج
إلى نقاط النهاية الأصلية لـ OpenAI/Codex.

السلوك المهم:

- يجب أن يستهدف `openai/*` المباشر العنوان `api.openai.com`
- يجب أن يستهدف `openai-codex/*` العنوان `chatgpt.com/backend-api`
- إذا قمت بتوجيه أي من الموفّرين عبر base URL أو proxy آخر، فسيترك OpenClaw قيمة `service_tier` كما هي

### الوضع السريع في OpenAI

يكشف OpenClaw عن مفتاح تبديل مشترك للوضع السريع لكل من جلسات `openai/*` و
`openai-codex/*`:

- الدردشة/واجهة المستخدم: `/fast status|on|off`
- الإعداد: `agents.defaults.models["<provider>/<model>"].params.fastMode`

عند تفعيل الوضع السريع، يربطه OpenClaw بالمعالجة ذات الأولوية في OpenAI:

- استدعاءات Responses المباشرة الخاصة بـ `openai/*` إلى `api.openai.com` ترسل `service_tier = "priority"`
- كما ترسل استدعاءات Responses الخاصة بـ `openai-codex/*` إلى `chatgpt.com/backend-api` القيمة `service_tier = "priority"`
- يتم الحفاظ على قيم `service_tier` الموجودة في الحمولة
- لا يعيد الوضع السريع كتابة `reasoning` أو `text.verbosity`

مثال:

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

تتغلب التجاوزات الخاصة بالجلسة على الإعداد. ويؤدي مسح تجاوز الجلسة في واجهة Sessions
إلى إعادة الجلسة إلى الافتراضي المعيّن.

### المسارات الأصلية لـ OpenAI مقابل المسارات المتوافقة مع OpenAI

يتعامل OpenClaw مع نقاط نهاية OpenAI وCodex وAzure OpenAI المباشرة بشكل مختلف
عن بروكسيات `/v1` العامة المتوافقة مع OpenAI:

- تحتفظ المسارات الأصلية `openai/*` و`openai-codex/*` وAzure OpenAI
  بالقيمة `reasoning: { effort: "none" }` كما هي عندما تقوم صراحةً بتعطيل الاستدلال
- تضبط المسارات الأصلية لعائلة OpenAI مخططات الأدوات على الوضع الصارم افتراضيًا
- لا تُرفق رؤوس الإسناد المخفية الخاصة بـ OpenClaw ‏(`originator`, `version`, و
  `User-Agent`) إلا على مضيفات OpenAI الأصلية التي تم التحقق منها
  (`api.openai.com`) ومضيفات Codex الأصلية (`chatgpt.com/backend-api`)
- تحتفظ المسارات الأصلية لـ OpenAI/Codex بتشكيل الطلبات الخاص بـ OpenAI فقط مثل
  `service_tier`، و`store` في Responses، والحمولات المتوافقة مع OpenAI للاستدلال، و
  تلميحات التخزين المؤقت للمطالبات
- تحتفظ المسارات المتوافقة مع OpenAI بأسلوب proxy بالسلوك التوافقي الأكثر مرونة ولا
  تفرض مخططات أدوات صارمة أو تشكيل طلبات خاصًا بالمسارات الأصلية أو رؤوس
  إسناد OpenAI/Codex المخفية

يبقى Azure OpenAI ضمن سلة التوجيه الأصلية من أجل النقل وسلوك التوافق،
لكنه لا يتلقى رؤوس الإسناد المخفية الخاصة بـ OpenAI/Codex.

يحافظ هذا على سلوك OpenAI Responses الأصلي الحالي من دون فرض
طبقات OpenAI-compatible الأقدم على الخلفيات الخارجية `/v1`.

### الضغط على جانب الخادم في OpenAI Responses

بالنسبة إلى نماذج OpenAI Responses المباشرة (`openai/*` التي تستخدم `api: "openai-responses"` مع
`baseUrl` على `api.openai.com`)، يفعّل OpenClaw الآن تلقائيًا تلميحات الحمولة الخاصة
بالضغط على جانب خادم OpenAI:

- يفرض `store: true` ‏(ما لم يضبط توافق النموذج `supportsStore: false`)
- يحقن `context_management: [{ type: "compaction", compact_threshold: ... }]`

افتراضيًا، تكون قيمة `compact_threshold` هي `70%` من `contextWindow` الخاص بالنموذج (أو `80000`
عند عدم توفرها).

### تفعيل الضغط على جانب الخادم صراحةً

استخدم هذا عندما تريد فرض حقن `context_management` على
نماذج Responses المتوافقة (مثل Azure OpenAI Responses):

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### التفعيل بحد مخصص

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### تعطيل الضغط على جانب الخادم

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

يتحكم `responsesServerCompaction` فقط في حقن `context_management`.
ولا تزال نماذج OpenAI Responses المباشرة تفرض `store: true` ما لم يضبط التوافق
`supportsStore: false`.

## ملاحظات

- تستخدم مراجع النماذج دائمًا الصيغة `provider/model` (راجع [/concepts/models](/concepts/models)).
- توجد تفاصيل المصادقة + قواعد إعادة الاستخدام في [/concepts/oauth](/concepts/oauth).
