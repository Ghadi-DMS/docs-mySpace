---
read_when:
    - تريد استخدام نماذج OpenAI في OpenClaw
    - تريد مصادقة اشتراك Codex بدلًا من مفاتيح API
summary: استخدم OpenAI عبر مفاتيح API أو اشتراك Codex في OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-06T03:12:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e04db5787f6ed7b1eda04d965c10febae10809fc82ae4d9769e7163234471f5
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

توفّر OpenAI واجهات API للمطورين لنماذج GPT. يدعم Codex **تسجيل الدخول عبر ChatGPT** للوصول
عبر الاشتراك أو **تسجيل الدخول عبر مفتاح API** للوصول المعتمد على الاستخدام. يتطلب Codex cloud تسجيل الدخول عبر ChatGPT.
وتدعم OpenAI صراحةً استخدام OAuth الخاص بالاشتراك في الأدوات/سير العمل الخارجية مثل OpenClaw.

## نمط التفاعل الافتراضي

يمكن لـ OpenClaw إضافة طبقة موجّه صغيرة خاصة بـ OpenAI لكل من تشغيلات `openai/*` و
`openai-codex/*`. افتراضيًا، تُبقي هذه الطبقة المساعد دافئًا،
وتعاونيًا، وموجزًا، ومباشرًا، وأكثر تعبيرًا عاطفيًا قليلًا
من دون استبدال موجّه نظام OpenClaw الأساسي. كما تسمح الطبقة الودية أيضًا
باستخدام رمز تعبيري من حين لآخر عندما يكون مناسبًا بشكل طبيعي، مع الحفاظ
على الإخراج العام موجزًا.

مفتاح الإعدادات:

`plugins.entries.openai.config.personality`

القيم المسموح بها:

- `"friendly"`: الافتراضي؛ يفعّل الطبقة الخاصة بـ OpenAI.
- `"off"`: يعطّل الطبقة ويستخدم موجّه OpenClaw الأساسي فقط.

النطاق:

- ينطبق على نماذج `openai/*`.
- ينطبق على نماذج `openai-codex/*`.
- لا يؤثر في المزودين الآخرين.

هذا السلوك مفعّل افتراضيًا. احتفظ بالقيمة `"friendly"` صراحةً إذا كنت تريد
أن يستمر ذلك رغم تغييرات الإعدادات المحلية مستقبلًا:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "friendly",
        },
      },
    },
  },
}
```

### تعطيل طبقة موجّه OpenAI

إذا كنت تريد موجّه OpenClaw الأساسي غير المعدل، فاضبط الطبقة على `"off"`:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "off",
        },
      },
    },
  },
}
```

يمكنك أيضًا ضبطها مباشرة عبر CLI للإعدادات:

```bash
openclaw config set plugins.entries.openai.config.personality off
```

## الخيار أ: مفتاح API لـ OpenAI ‏(OpenAI Platform)

**الأفضل لـ:** الوصول المباشر إلى API والفوترة حسب الاستخدام.
احصل على مفتاح API من لوحة معلومات OpenAI.

### إعداد CLI

```bash
openclaw onboard --auth-choice openai-api-key
# أو بدون تفاعل
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### مقتطف إعدادات

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

تسرد وثائق نماذج API الحالية من OpenAI النموذجين `gpt-5.4` و`gpt-5.4-pro` للاستخدام المباشر
مع OpenAI API. ويقوم OpenClaw بتمريرهما عبر مسار `openai/*` Responses.
ويخفي OpenClaw عمدًا الصف القديم `openai/gpt-5.3-codex-spark`،
لأن استدعاءات OpenAI API المباشرة ترفضه في حركة المرور الحية.

لا يعرّض OpenClaw `openai/gpt-5.3-codex-spark` على مسار OpenAI المباشر
لـ API. لا يزال `pi-ai` يشحن صفًا مدمجًا لذلك النموذج، لكن طلبات OpenAI API الحية
ترفضه حاليًا. ويتم التعامل مع Spark على أنه خاص بـ Codex فقط في OpenClaw.

## توليد الصور

تسجل إضافة `openai` المضمنة أيضًا توليد الصور عبر أداة
`image_generate` المشتركة.

- نموذج الصور الافتراضي: `openai/gpt-image-1`
- التوليد: حتى 4 صور لكل طلب
- وضع التعديل: مفعّل، حتى 5 صور مرجعية
- يدعم `size`
- ملاحظة خاصة حالية بـ OpenAI: لا يمرر OpenClaw حاليًا تجاوزات `aspectRatio` أو
  `resolution` إلى OpenAI Images API

لاستخدام OpenAI كمزود الصور الافتراضي:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

راجع [توليد الصور](/ar/tools/image-generation) للحصول على معلمات
الأداة المشتركة، واختيار المزود، وسلوك تجاوز الفشل.

## توليد الفيديو

تسجل إضافة `openai` المضمنة أيضًا توليد الفيديو عبر أداة
`video_generate` المشتركة.

- نموذج الفيديو الافتراضي: `openai/sora-2`
- الأوضاع: نص إلى فيديو، وصورة إلى فيديو، وتدفقات مرجعية/تعديل لفيديو واحد
- الحدود الحالية: صورة واحدة أو فيديو مرجعي واحد
- ملاحظة خاصة حالية بـ OpenAI: لا يمرر OpenClaw حاليًا سوى تجاوزات `size`
  لتوليد الفيديو الأصلي في OpenAI. أما التجاوزات الاختيارية غير المدعومة
  مثل `aspectRatio` و`resolution` و`audio` و`watermark` فيتم تجاهلها
  والإبلاغ عنها كتحذير من الأداة.

لاستخدام OpenAI كمزود الفيديو الافتراضي:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openai/sora-2",
      },
    },
  },
}
```

راجع [توليد الفيديو](/tools/video-generation) للحصول على معلمات
الأداة المشتركة، واختيار المزود، وسلوك تجاوز الفشل.

## الخيار ب: اشتراك OpenAI Code ‏(Codex)

**الأفضل لـ:** استخدام وصول اشتراك ChatGPT/Codex بدلًا من مفتاح API.
يتطلب Codex cloud تسجيل الدخول عبر ChatGPT، بينما يدعم Codex CLI تسجيل الدخول عبر ChatGPT أو مفتاح API.

### إعداد CLI ‏(Codex OAuth)

```bash
# شغّل Codex OAuth في المعالج
openclaw onboard --auth-choice openai-codex

# أو شغّل OAuth مباشرة
openclaw models auth login --provider openai-codex
```

### مقتطف إعدادات (اشتراك Codex)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

تسرد وثائق Codex الحالية من OpenAI النموذج `gpt-5.4` بوصفه نموذج Codex الحالي. ويقوم OpenClaw
بربط ذلك إلى `openai-codex/gpt-5.4` لاستخدام ChatGPT/Codex OAuth.

إذا أعاد الإعداد الأولي استخدام تسجيل دخول موجود في Codex CLI، فستظل بيانات الاعتماد تلك
مدارة من قبل Codex CLI. وعند انتهاء الصلاحية، يعيد OpenClaw قراءة مصدر Codex الخارجي
أولًا، وعندما يكون المزود قادرًا على تحديثه، يكتب بيانات الاعتماد المحدّثة
مرة أخرى إلى تخزين Codex بدلًا من الاستحواذ عليها في نسخة منفصلة
خاصة بـ OpenClaw فقط.

إذا كان حساب Codex الخاص بك مؤهلًا لـ Codex Spark، فإن OpenClaw يدعم أيضًا:

- `openai-codex/gpt-5.3-codex-spark`

يتعامل OpenClaw مع Codex Spark على أنه خاص بـ Codex فقط. ولا يعرّض
مسار مفتاح API مباشرًا `openai/gpt-5.3-codex-spark`.

كما يحافظ OpenClaw أيضًا على `openai-codex/gpt-5.3-codex-spark` عندما يكتشفه `pi-ai`.
تعامل معه على أنه معتمد على الأهلية وتجريبي: Codex Spark منفصل عن GPT-5.4 `/fast`،
ويعتمد توفره على حساب Codex / ChatGPT المسجل دخوله.

### حد نافذة سياق Codex

يتعامل OpenClaw مع بيانات تعريف نموذج Codex والحد التشغيلي للسياق
على أنهما قيمتان منفصلتان.

بالنسبة إلى `openai-codex/gpt-5.4`:

- `contextWindow` الأصلي: `1050000`
- الحد الافتراضي لـ `contextTokens` في وقت التشغيل: `272000`

وهذا يبقي بيانات تعريف النموذج صحيحة مع الحفاظ على نافذة وقت تشغيل افتراضية أصغر
تتميز عمليًا بزمن استجابة وجودة أفضل.

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

استخدم `contextWindow` فقط عندما تكون تعرّف أو تتجاوز بيانات تعريف النموذج
الأصلية. واستخدم `contextTokens` عندما تريد تقييد ميزانية السياق في وقت التشغيل.

### النقل الافتراضي

يستخدم OpenClaw `pi-ai` لبث النماذج. لكل من `openai/*` و
`openai-codex/*`، يكون النقل الافتراضي هو `"auto"` ‏(WebSocket أولًا، ثم fallback
إلى SSE).

في وضع `"auto"`، يعيد OpenClaw أيضًا محاولة فشل WebSocket مبكر قابل لإعادة المحاولة مرة واحدة
قبل أن يعود إلى SSE. أما وضع `"websocket"` الإجباري فما يزال يعرض أخطاء النقل
مباشرة بدلًا من إخفائها وراء fallback.

بعد فشل WebSocket عند الاتصال أو في دور مبكر في وضع `"auto"`، يضع OpenClaw
مسار WebSocket لتلك الجلسة في حالة degraded لمدة تقارب 60 ثانية، ويرسل
الأدوار اللاحقة عبر SSE أثناء فترة التهدئة بدلًا من التذبذب بين
وسائل النقل.

بالنسبة إلى نقاط نهاية عائلة OpenAI الأصلية (`openai/*` و`openai-codex/*` وAzure
OpenAI Responses)، يرفق OpenClaw أيضًا حالة هوية ثابتة للجلسة والدور
بالطلبات حتى تبقى عمليات إعادة المحاولة وإعادة الاتصال وfallback إلى SSE
متسقة مع هوية المحادثة نفسها. وعلى مسارات عائلة OpenAI الأصلية يتضمن ذلك
رؤوس هوية طلب ثابتة للجلسة/الدور بالإضافة إلى بيانات تعريف نقل مطابقة.

كما يطبع OpenClaw أيضًا عدادات استخدام OpenAI عبر اختلافات النقل قبل أن تصل
إلى أسطح الجلسة/الحالة. قد تُبلغ حركة المرور الأصلية لـ OpenAI/Codex Responses
عن الاستخدام على شكل `input_tokens` / `output_tokens` أو
`prompt_tokens` / `completion_tokens`؛ ويتعامل OpenClaw معها على أنها عدادات
الإدخال والإخراج نفسها في `/status` و`/usage` وسجلات الجلسات. وعندما تحذف
حركة WebSocket الأصلية `total_tokens` ‏(أو تُبلغ عنها بالقيمة `0`)، يعود OpenClaw إلى
الإجمالي المطبّع للإدخال + الإخراج حتى تظل شاشات الجلسة/الحالة مملوءة.

يمكنك ضبط `agents.defaults.models.<provider/model>.params.transport`:

- `"sse"`: فرض SSE
- `"websocket"`: فرض WebSocket
- `"auto"`: محاولة WebSocket، ثم fallback إلى SSE

بالنسبة إلى `openai/*` ‏(Responses API)، يفعّل OpenClaw أيضًا التهيئة المسبقة لـ WebSocket
افتراضيًا (`openaiWsWarmup: true`) عند استخدام نقل WebSocket.

وثائق OpenAI ذات الصلة:

- [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

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

### التهيئة المسبقة لـ OpenAI WebSocket

تصف وثائق OpenAI التهيئة المسبقة على أنها اختيارية. ويقوم OpenClaw بتفعيلها افتراضيًا لـ
`openai/*` لتقليل زمن الاستجابة في الدور الأول عند استخدام نقل WebSocket.

### تعطيل التهيئة المسبقة

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

### تفعيل التهيئة المسبقة صراحةً

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

تعرض API الخاصة بـ OpenAI المعالجة ذات الأولوية عبر `service_tier=priority`. في
OpenClaw، اضبط `agents.defaults.models["<provider>/<model>"].params.serviceTier`
لتمرير هذا الحقل على نقاط نهاية OpenAI/Codex Responses الأصلية.

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

يمرر OpenClaw `params.serviceTier` إلى كل من طلبات
`openai/*` Responses المباشرة وطلبات `openai-codex/*` Codex Responses عندما تشير تلك النماذج
إلى نقاط نهاية OpenAI/Codex الأصلية.

سلوك مهم:

- يجب أن يستهدف `openai/*` المباشر `api.openai.com`
- يجب أن يستهدف `openai-codex/*` العنوان `chatgpt.com/backend-api`
- إذا قمت بتوجيه أي من المزودين عبر base URL أو proxy آخر، يترك OpenClaw `service_tier` دون تعديل

### الوضع السريع في OpenAI

يعرّض OpenClaw مفتاح تبديل مشترك للوضع السريع لكل من جلسات `openai/*` و
`openai-codex/*`:

- الدردشة/واجهة المستخدم: `/fast status|on|off`
- الإعدادات: `agents.defaults.models["<provider>/<model>"].params.fastMode`

عند تفعيل الوضع السريع، يربطه OpenClaw بمعالجة OpenAI ذات الأولوية:

- ترسل استدعاءات `openai/*` Responses المباشرة إلى `api.openai.com` القيمة `service_tier = "priority"`
- كما ترسل استدعاءات `openai-codex/*` Responses إلى `chatgpt.com/backend-api` القيمة `service_tier = "priority"`
- يتم الحفاظ على قيم `service_tier` الموجودة في الحمولة
- لا يعيد الوضع السريع كتابة `reasoning` أو `text.verbosity`

بالنسبة إلى GPT 5.4 تحديدًا، فإن الإعداد الأكثر شيوعًا هو:

- إرسال `/fast on` في جلسة تستخدم `openai/gpt-5.4` أو `openai-codex/gpt-5.4`
- أو ضبط `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true`
- إذا كنت تستخدم أيضًا Codex OAuth، فاضبط `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true` أيضًا

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

تتغلب تجاوزات الجلسة على الإعدادات. ويؤدي مسح تجاوز الجلسة في واجهة Sessions
إلى إعادة الجلسة إلى الإعداد الافتراضي المضبوط.

### مسارات OpenAI الأصلية مقابل المسارات المتوافقة مع OpenAI

يتعامل OpenClaw مع نقاط نهاية OpenAI وCodex وAzure OpenAI المباشرة
بشكل مختلف عن وكلاء `/v1` العامة المتوافقة مع OpenAI:

- تحافظ المسارات الأصلية `openai/*` و`openai-codex/*` وAzure OpenAI على
  `reasoning: { effort: "none" }` دون تغيير عندما تعطل الاستدلال صراحةً
- تجعل مسارات عائلة OpenAI الأصلية مخططات الأدوات في الوضع الصارم افتراضيًا
- لا تُرفق رؤوس الإسناد المخفية الخاصة بـ OpenClaw (`originator` و`version` و
  `User-Agent`) إلا على مضيفات OpenAI الأصلية المتحقق منها
  (`api.openai.com`) ومضيفات Codex الأصلية (`chatgpt.com/backend-api`)
- تحافظ مسارات OpenAI/Codex الأصلية على تشكيل الطلبات الخاص بـ OpenAI مثل
  `service_tier` و`store` في Responses وحمولات توافق الاستدلال في OpenAI و
  تلميحات ذاكرة التخزين المؤقت للموجّه
- تحافظ المسارات المتوافقة مع OpenAI بأسلوب الوكلاء على سلوك التوافق الأكثر مرونة ولا
  تفرض مخططات أدوات صارمة، أو تشكيل طلبات أصلي فقط، أو رؤوس
  إسناد OpenAI/Codex المخفية

يبقى Azure OpenAI ضمن فئة التوجيه الأصلي من حيث النقل وسلوك التوافق،
لكنه لا يتلقى رؤوس الإسناد المخفية الخاصة بـ OpenAI/Codex.

يحافظ هذا على سلوك OpenAI Responses الأصلي الحالي من دون فرض
طبقات OpenAI المتوافقة الأقدم على واجهات `/v1` الخلفية التابعة لجهات خارجية.

### الضغط من جهة الخادم في OpenAI Responses

بالنسبة إلى نماذج OpenAI Responses المباشرة (`openai/*` التي تستخدم `api: "openai-responses"` مع
`baseUrl` على `api.openai.com`)، يقوم OpenClaw الآن بتفعيل تلميحات حمولة الضغط
من جهة خادم OpenAI تلقائيًا:

- يفرض `store: true` ‏(ما لم يضبط توافق النموذج `supportsStore: false`)
- يحقن `context_management: [{ type: "compaction", compact_threshold: ... }]`

افتراضيًا، تكون قيمة `compact_threshold` مساوية لـ `70%` من `contextWindow` الخاص بالنموذج (أو `80000`
عند عدم توفرها).

### تفعيل الضغط من جهة الخادم صراحةً

استخدم هذا عندما تريد فرض حقن `context_management` على نماذج
Responses المتوافقة (مثل Azure OpenAI Responses):

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

### التفعيل مع حد مخصص

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

### تعطيل الضغط من جهة الخادم

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

- تستخدم مراجع النماذج دائمًا الصيغة `provider/model` ‏(راجع [/concepts/models](/ar/concepts/models)).
- توجد تفاصيل المصادقة + قواعد إعادة الاستخدام في [/concepts/oauth](/ar/concepts/oauth).
