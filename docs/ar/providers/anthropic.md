---
read_when:
    - أنت تريد استخدام نماذج Anthropic في OpenClaw
    - أنت تريد إعادة استخدام مصادقة اشتراك Claude CLI على مضيف البوابة
summary: استخدم Anthropic Claude عبر مفاتيح API أو Claude CLI في OpenClaw
title: Anthropic
x-i18n:
    generated_at: "2026-04-05T12:53:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 80f2b614eba4563093522e5157848fc54a16770a2fae69f17c54f1b9bfff624f
    source_path: providers/anthropic.md
    workflow: 15
---

# Anthropic (Claude)

تقوم Anthropic ببناء عائلة نماذج **Claude** وتوفر الوصول إليها عبر API.
في OpenClaw، يجب أن يستخدم إعداد Anthropic الجديد مفتاح API أو الخلفية المحلية Claude CLI.
ولا تزال ملفات تعريف رموز Anthropic القديمة الحالية محترمة في وقت التشغيل
إذا كانت مُعدّة بالفعل.

<Warning>
تُوثّق مستندات Claude Code العامة من Anthropic صراحةً استخدام CLI غير التفاعلي
مثل `claude -p`. واستنادًا إلى تلك المستندات، نعتقد أن مسار الرجوع المحلي
الذي يديره المستخدم عبر Claude Code CLI مسموح به على الأرجح.

وبشكل منفصل، أبلغت Anthropic مستخدمي OpenClaw في **4 أبريل 2026 الساعة 12:00 ظهرًا
بتوقيت PT / 8:00 مساءً بتوقيت BST** أن **OpenClaw يُحتسب على أنه harness تابع لجهة خارجية**. ووفقًا
لسياستهم المعلنة، فإن حركة Claude-login التي يقودها OpenClaw لم تعد تستخدم
مجموعة اشتراك Claude المضمنة، بل تتطلب بدلًا من ذلك **Extra Usage**
(الدفع حسب الاستخدام، مع فوترة منفصلة عن الاشتراك).

ويخص هذا التمييز في السياسة **إعادة استخدام Claude CLI بقيادة OpenClaw**،
وليس تشغيل `claude` مباشرةً في الطرفية الخاصة بك. ومع ذلك، لا تزال سياسة Anthropic
الخاصة بالـ harness التابع لجهة خارجية تترك قدرًا كافيًا من الغموض حول
الاستخدام المدعوم بالاشتراك في المنتجات الخارجية بحيث لا نوصي بهذا
المسار للإنتاج.

مستندات Anthropic العامة الحالية:

- [مرجع Claude Code CLI](https://code.claude.com/docs/en/cli-reference)
- [نظرة عامة على Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview)

- [استخدام Claude Code مع خطة Pro أو Max الخاصة بك](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [استخدام Claude Code مع خطة Team أو Enterprise الخاصة بك](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)

إذا كنت تريد أوضح مسار للفوترة، فاستخدم مفتاح Anthropic API بدلًا من ذلك.
يدعم OpenClaw أيضًا خيارات أخرى بنمط الاشتراك، بما في ذلك [OpenAI
Codex](/providers/openai)، و[Qwen Cloud Coding Plan](/providers/qwen)،
و[MiniMax Coding Plan](/providers/minimax)، و[Z.AI / GLM Coding
Plan](/providers/glm).
</Warning>

## الخيار A: مفتاح Anthropic API

**الأفضل لـ:** الوصول القياسي عبر API والفوترة حسب الاستخدام.
أنشئ مفتاح API الخاص بك من Anthropic Console.

### الإعداد عبر CLI

```bash
openclaw onboard
# choose: Anthropic API key

# or non-interactive
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### مقتطف إعداد Claude CLI

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## القيم الافتراضية للتفكير (Claude 4.6)

- تستخدم نماذج Anthropic Claude 4.6 التفكير `adaptive` افتراضيًا في OpenClaw عندما لا يتم ضبط مستوى تفكير صريح.
- يمكنك التجاوز لكل رسالة (`/think:<level>`) أو في معلمات النموذج:
  `agents.defaults.models["anthropic/<model>"].params.thinking`.
- مستندات Anthropic ذات الصلة:
  - [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
  - [Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## الوضع السريع (Anthropic API)

يدعم مفتاح التبديل المشترك `/fast` في OpenClaw أيضًا حركة Anthropic العامة المباشرة، بما في ذلك الطلبات الموثقة عبر مفتاح API وOAuth المرسلة إلى `api.anthropic.com`.

- يربط `/fast on` إلى `service_tier: "auto"`
- يربط `/fast off` إلى `service_tier: "standard_only"`
- الإعداد الافتراضي:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-sonnet-4-6": {
          params: { fastMode: true },
        },
      },
    },
  },
}
```

القيود المهمة:

- لا يحقن OpenClaw مستويات خدمة Anthropic إلا للطلبات المباشرة إلى `api.anthropic.com`. إذا قمت بتوجيه `anthropic/*` عبر proxy أو بوابة، فإن `/fast` يترك `service_tier` كما هو.
- تتجاوز معلمات Anthropic الصريحة `serviceTier` أو `service_tier` الافتراضي الخاص بـ `/fast` عندما يتم ضبطهما معًا.
- تبلّغ Anthropic عن المستوى الفعلي في الاستجابة ضمن `usage.service_tier`. وفي الحسابات التي لا تملك سعة Priority Tier، قد يظل `service_tier: "auto"` يتحلل إلى `standard`.

## التخزين المؤقت للمطالبات (Anthropic API)

يدعم OpenClaw ميزة التخزين المؤقت للمطالبات الخاصة بـ Anthropic. وهذه الميزة **خاصة بـ API فقط**؛ إذ لا تحترم مصادقة رموز Anthropic القديمة إعدادات التخزين المؤقت.

### الإعداد

استخدم المعلمة `cacheRetention` في إعداد النموذج:

| القيمة | مدة التخزين المؤقت | الوصف |
| ------- | -------------- | ------------------------ |
| `none`  | لا يوجد تخزين مؤقت     | تعطيل التخزين المؤقت للمطالبات   |
| `short` | 5 دقائق      | الافتراضي لمصادقة مفتاح API |
| `long`  | ساعة واحدة         | تخزين مؤقت ممتد           |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

### القيم الافتراضية

عند استخدام مصادقة مفتاح Anthropic API، يطبق OpenClaw تلقائيًا `cacheRetention: "short"` (تخزين مؤقت لمدة 5 دقائق) على جميع نماذج Anthropic. ويمكنك تجاوز ذلك عبر ضبط `cacheRetention` صراحةً في إعداداتك.

### تجاوزات `cacheRetention` لكل وكيل

استخدم معلمات مستوى النموذج كخط أساس لديك، ثم تجاوز وكلاء محددين عبر `agents.list[].params`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" }, // baseline for most agents
        },
      },
    },
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } }, // override for this agent only
    ],
  },
}
```

ترتيب دمج الإعداد لمعلمات التخزين المؤقت:

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params` ‏(مطابق لـ `id`، ويتجاوز حسب المفتاح)

يتيح هذا لوكيل واحد الاحتفاظ بتخزين مؤقت طويل العمر، بينما يعطّل وكيل آخر على النموذج نفسه التخزين المؤقت لتجنب تكاليف الكتابة على الحركة المتدفقة أو ذات إعادة الاستخدام المنخفضة.

### ملاحظات Bedrock Claude

- تقبل نماذج Anthropic Claude على Bedrock ‏(`amazon-bedrock/*anthropic.claude*`) تمرير `cacheRetention` عند إعداده.
- تُجبر نماذج Bedrock غير التابعة لـ Anthropic على `cacheRetention: "none"` في وقت التشغيل.
- كما تُهيّئ القيم الذكية الافتراضية لمفتاح Anthropic API القيمة `cacheRetention: "short"` لمراجع نماذج Claude-on-Bedrock عندما لا تكون هناك قيمة صريحة مضبوطة.

## نافذة سياق 1M ‏(Anthropic beta)

تخضع نافذة السياق 1M من Anthropic لتقييد beta. وفي OpenClaw، فعّلها لكل نموذج
باستخدام `params.context1m: true` لنماذج Opus/Sonnet المدعومة.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { context1m: true },
        },
      },
    },
  },
}
```

يربط OpenClaw هذا إلى `anthropic-beta: context-1m-2025-08-07` على طلبات Anthropic.

لا يتم تفعيل هذا إلا عندما تكون `params.context1m` مضبوطة صراحةً على `true` لذلك
النموذج.

المتطلب: يجب أن تسمح Anthropic باستخدام السياق الطويل على بيانات الاعتماد تلك
(عادةً فوترة مفتاح API، أو مسار Claude-login في OpenClaw / مصادقة الرموز القديمة
مع تفعيل Extra Usage). وإلا فستعيد Anthropic:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

ملاحظة: ترفض Anthropic حاليًا طلبات beta الخاصة بـ `context-1m-*` عند استخدام
مصادقة رموز Anthropic القديمة (`sk-ant-oat-*`). وإذا قمت بإعداد
`context1m: true` مع وضع المصادقة القديم هذا، فسيسجل OpenClaw تحذيرًا ويعود
إلى نافذة السياق القياسية عبر تخطي ترويسة beta الخاصة بـ context1m
مع الإبقاء على إصدارات OAuth beta المطلوبة.

## الخيار B: Claude CLI كموفّر الرسائل

**الأفضل لـ:** مضيف بوابة أحادي المستخدم لديه Claude CLI مثبت ومسجل الدخول
بالفعل، كحل رجوع محلي بدلًا من المسار الموصى به للإنتاج.

ملاحظة الفوترة: نعتقد أن الرجوع إلى Claude Code CLI مسموح به على الأرجح من أجل الأتمتة المحلية
التي يديرها المستخدم استنادًا إلى مستندات CLI العامة الخاصة بـ Anthropic. ومع ذلك،
تخلق سياسة Anthropic الخاصة بالـ harness التابع لجهة خارجية قدرًا كافيًا من الغموض حول
الاستخدام المدعوم بالاشتراك في المنتجات الخارجية بحيث لا نوصي به في
الإنتاج. كما أخبرت Anthropic مستخدمي OpenClaw أن استخدام Claude
CLI **بقيادة OpenClaw** يُعامل على أنه حركة harness تابعة لجهة خارجية، وابتداءً من **4 أبريل 2026
الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST**، يتطلب **Extra Usage** بدلًا من
حدود اشتراك Claude المضمنة.

يستخدم هذا المسار الملف التنفيذي المحلي `claude` لاستدلال النموذج بدلًا من استدعاء
Anthropic API مباشرة. ويتعامل معه OpenClaw على أنه **CLI backend provider**
بمراجع نماذج مثل:

- `claude-cli/claude-sonnet-4-6`
- `claude-cli/claude-opus-4-6`

كيف يعمل:

1. يشغّل OpenClaw الأمر `claude -p --output-format stream-json --include-partial-messages ...`
   على **مضيف البوابة** ويرسل المطالبة عبر stdin.
2. يرسل الدور الأول `--session-id <uuid>`.
3. تعيد الأدوار اللاحقة استخدام جلسة Claude المخزنة عبر `--resume <sessionId>`.
4. لا تزال رسائل الدردشة الخاصة بك تمر عبر مسار الرسائل العادي في OpenClaw، لكن
   الرد الفعلي للنموذج يتم إنتاجه بواسطة Claude CLI.

### المتطلبات

- Claude CLI مثبت على مضيف البوابة ومتوفر على PATH، أو مُعدّ
  بمسار أمر مطلق.
- Claude CLI موثّق بالفعل على ذلك المضيف نفسه:

```bash
claude auth status
```

- يقوم OpenClaw بتحميل plugin Anthropic المضمّن تلقائيًا عند بدء تشغيل البوابة عندما
  تشير إعداداتك صراحةً إلى `claude-cli/...` أو إعداد خلفية `claude-cli`.

### مقتطف الإعداد

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "claude-cli/claude-sonnet-4-6",
      },
      models: {
        "claude-cli/claude-sonnet-4-6": {},
      },
      sandbox: { mode: "off" },
    },
  },
}
```

إذا لم يكن الملف التنفيذي `claude` موجودًا على PATH الخاص بمضيف البوابة:

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "claude-cli": {
          command: "/opt/homebrew/bin/claude",
        },
      },
    },
  },
}
```

### ما الذي تحصل عليه

- إعادة استخدام مصادقة اشتراك Claude من CLI المحلي (تُقرأ في وقت التشغيل، ولا تُخزَّن)
- توجيه الرسائل/الجلسات العادي في OpenClaw
- استمرارية جلسة Claude CLI عبر الأدوار (تُبطل عند تغيّر المصادقة)
- أدوات البوابة مكشوفة إلى Claude CLI عبر جسر MCP على loopback
- بث JSONL مع تقدم حي للرسائل الجزئية

### الترحيل من مصادقة Anthropic إلى Claude CLI

إذا كنت تستخدم حاليًا `anthropic/...` مع ملف تعريف رمز قديم أو مفتاح API وتريد
تحويل مضيف البوابة نفسه إلى Claude CLI، فإن OpenClaw يدعم ذلك كمسار
ترحيل عادي لمصادقة الموفّر.

المتطلبات الأساسية:

- Claude CLI مثبت على **مضيف البوابة نفسه** الذي يشغّل OpenClaw
- Claude CLI مسجل الدخول هناك بالفعل: `claude auth login`

ثم شغّل:

```bash
openclaw models auth login --provider anthropic --method cli --set-default
```

أو أثناء الإعداد الأولي:

```bash
openclaw onboard --auth-choice anthropic-cli
```

يفضّل كل من `openclaw onboard` و`openclaw configure` التفاعليين الآن **Anthropic
Claude CLI** أولًا و**Anthropic API key** ثانيًا.

ما الذي يفعله هذا:

- يتحقق من أن Claude CLI مسجّل الدخول بالفعل على مضيف البوابة
- يبدّل النموذج الافتراضي إلى `claude-cli/...`
- يعيد كتابة بدائل النموذج الافتراضي لـ Anthropic مثل `anthropic/claude-opus-4-6`
  إلى `claude-cli/claude-opus-4-6`
- يضيف إدخالات `claude-cli/...` المطابقة إلى `agents.defaults.models`

تحقق سريع:

```bash
openclaw models status
```

يجب أن ترى النموذج الأساسي المحلول تحت `claude-cli/...`.

ما الذي **لا** يفعله:

- حذف ملفات تعريف المصادقة الحالية الخاصة بـ Anthropic
- إزالة كل مراجع الإعداد القديمة `anthropic/...` خارج مسار
  النموذج الافتراضي/قائمة السماح الرئيسية

وهذا يجعل التراجع بسيطًا: فقط غيّر النموذج الافتراضي مرة أخرى إلى `anthropic/...` إذا
احتجت إلى ذلك.

### القيود المهمة

- هذا **ليس** موفّر Anthropic API. بل هو وقت تشغيل CLI المحلي.
- لا يحقن OpenClaw استدعاءات الأدوات مباشرة. يتلقى Claude CLI أدوات
  البوابة عبر جسر MCP على loopback ‏(`bundleMcp: true`، وهو الافتراضي).
- يبث Claude CLI الردود عبر JSONL ‏(`stream-json` مع
  `--include-partial-messages`). وتُرسل المطالبات عبر stdin، وليس argv.
- تُقرأ المصادقة في وقت التشغيل من بيانات اعتماد Claude CLI الحية ولا تُخزَّن
  في ملفات تعريف OpenClaw. كما تُخفى مطالبات Keychain في السياقات
  غير التفاعلية.
- يُتتبَّع إعادة استخدام الجلسات عبر بيانات وصفية `cliSessionBinding`. وعندما تتغير
  حالة تسجيل دخول Claude CLI ‏(إعادة تسجيل الدخول، تدوير الرمز)، تُبطَل الجلسات
  المخزنة وتبدأ جلسة جديدة.
- هذا الأنسب لمضيف بوابة شخصي، وليس لإعدادات فوترة مشتركة متعددة المستخدمين.

مزيد من التفاصيل: [/gateway/cli-backends](/gateway/cli-backends)

## ملاحظات

- لا تزال مستندات Claude Code العامة من Anthropic توثق الاستخدام المباشر لـ CLI مثل
  `claude -p`. ونعتقد أن الرجوع المحلي الذي يديره المستخدم مسموح به على الأرجح، لكن
  الإشعار المنفصل من Anthropic إلى مستخدمي OpenClaw يقول إن مسار
  Claude-login الخاص بـ **OpenClaw** هو استخدام harness تابع لجهة خارجية ويتطلب **Extra Usage**
  (الدفع حسب الاستخدام مع فوترة منفصلة عن الاشتراك). وبالنسبة إلى الإنتاج، نحن
  نوصي باستخدام مفاتيح Anthropic API بدلًا من ذلك.
- أصبح Anthropic setup-token متاحًا مرة أخرى في OpenClaw كمسار قديم/يدوي. ولا يزال إشعار Anthropic الخاص بـ OpenClaw حول الفوترة ينطبق، لذا استخدمه مع توقع أن Anthropic تتطلب **Extra Usage** لهذا المسار.
- توجد تفاصيل المصادقة + قواعد إعادة الاستخدام في [/concepts/oauth](/concepts/oauth).

## استكشاف الأخطاء وإصلاحها

**أخطاء 401 / أصبح الرمز غير صالح فجأة**

- قد تنتهي صلاحية مصادقة رموز Anthropic القديمة أو يتم إبطالها.
- بالنسبة إلى الإعداد الجديد، انتقل إلى مفتاح Anthropic API أو إلى مسار Claude CLI المحلي على مضيف البوابة.

**لم يتم العثور على مفتاح API للموفّر "anthropic"**

- المصادقة تكون **لكل وكيل**. ولا ترث الوكلاء الجدد مفاتيح الوكيل الرئيسي.
- أعد تشغيل الإعداد الأولي لذلك الوكيل، أو اضبط مفتاح API على مضيف البوابة،
  ثم تحقق عبر `openclaw models status`.

**لم يتم العثور على بيانات اعتماد لملف التعريف `anthropic:default`**

- شغّل `openclaw models status` لمعرفة ملف تعريف المصادقة النشط.
- أعد تشغيل الإعداد الأولي، أو اضبط مفتاح API أو Claude CLI لذلك المسار الخاص بملف التعريف.

**لا يوجد ملف تعريف مصادقة متاح (كلها في وضع cooldown/unavailable)**

- تحقق من `openclaw models status --json` لرؤية `auth.unusableProfiles`.
- يمكن أن تكون فترات التهدئة الخاصة بحدود المعدل في Anthropic مرتبطة بالنموذج، لذلك قد يظل
  نموذج Anthropic شقيق قابلًا للاستخدام حتى عندما يكون النموذج الحالي في فترة تهدئة.
- أضف ملف تعريف Anthropic آخر أو انتظر انتهاء فترة التهدئة.

المزيد: [/gateway/troubleshooting](/gateway/troubleshooting) و[/help/faq](/help/faq).
