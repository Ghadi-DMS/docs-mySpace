---
read_when:
    - أنت تريد استخدام نماذج Anthropic في OpenClaw
summary: استخدم Claude من Anthropic عبر مفاتيح API في OpenClaw
title: Anthropic
x-i18n:
    generated_at: "2026-04-06T03:11:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: bbc6c4938674aedf20ff944bc04e742c9a7e77a5ff10ae4f95b5718504c57c2d
    source_path: providers/anthropic.md
    workflow: 15
---

# Anthropic (Claude)

تطوّر Anthropic عائلة نماذج **Claude** وتوفّر الوصول إليها عبر API.
في OpenClaw، يجب أن يستخدم إعداد Anthropic الجديد مفتاح API. ولا تزال
ملفات تعريف رموز Anthropic القديمة مُعترفًا بها في وقت التشغيل إذا كانت
مهيأة بالفعل.

<Warning>
بالنسبة إلى Anthropic في OpenClaw، يكون تقسيم الفوترة كما يلي:

- **مفتاح API من Anthropic**: فوترة Anthropic API العادية.
- **مصادقة اشتراك Claude داخل OpenClaw**: أخبرت Anthropic مستخدمي OpenClaw في
  **4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن هذا يُحتسب
  على أنه استخدام لحزمة طرف ثالث ويتطلب **Extra Usage** (الدفع حسب الاستخدام،
  وتتم فوترته بشكل منفصل عن الاشتراك).

تتطابق إعادة الإنتاج المحلية لدينا مع هذا التقسيم:

- قد يظل `claude -p` المباشر يعمل
- قد يؤدي `claude -p --append-system-prompt ...` إلى تفعيل حماية Extra Usage عندما
  يعرّف الطلب OpenClaw
- لا تؤدي صيغة طلب النظام المشابهة لـ OpenClaw نفسها إلى إعادة إنتاج الحظر على
  مسار Anthropic SDK + `ANTHROPIC_API_KEY`

لذا فإن القاعدة العملية هي: **مفتاح API من Anthropic، أو اشتراك Claude مع
Extra Usage**. إذا كنت تريد أوضح مسار إنتاجي، فاستخدم مفتاح API من Anthropic.

مستندات Anthropic العامة الحالية:

- [مرجع Claude Code CLI](https://code.claude.com/docs/en/cli-reference)
- [نظرة عامة على Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview)

- [استخدام Claude Code مع خطة Pro أو Max الخاصة بك](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [استخدام Claude Code مع خطة Team أو Enterprise الخاصة بك](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)

إذا كنت تريد أوضح مسار للفوترة، فاستخدم مفتاح API من Anthropic بدلًا من ذلك.
يدعم OpenClaw أيضًا خيارات أخرى مستضافة على نمط الاشتراك، بما في ذلك [OpenAI
Codex](/ar/providers/openai)، و[Qwen Cloud Coding Plan](/ar/providers/qwen)، و
[MiniMax Coding Plan](/ar/providers/minimax)، و[Z.AI / GLM Coding
Plan](/ar/providers/glm).
</Warning>

## الخيار A: مفتاح API من Anthropic

**الأفضل من أجل:** الوصول القياسي إلى API والفوترة حسب الاستخدام.
أنشئ مفتاح API الخاص بك في Anthropic Console.

### إعداد CLI

```bash
openclaw onboard
# choose: Anthropic API key

# or non-interactive
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### مقتطف إعداد Anthropic

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## الإعدادات الافتراضية للتفكير (Claude 4.6)

- تستخدم نماذج Anthropic Claude 4.6 مستوى التفكير `adaptive` افتراضيًا في OpenClaw عندما لا يتم تعيين مستوى تفكير صريح.
- يمكنك التجاوز لكل رسالة (`/think:<level>`) أو ضمن معاملات النموذج:
  `agents.defaults.models["anthropic/<model>"].params.thinking`.
- مستندات Anthropic ذات الصلة:
  - [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
  - [Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## الوضع السريع (Anthropic API)

يدعم مفتاح `/fast` المشترك في OpenClaw أيضًا حركة Anthropic العامة المباشرة، بما في ذلك الطلبات الموثقة بمفتاح API وOAuth المرسلة إلى `api.anthropic.com`.

- يتم ربط `/fast on` إلى `service_tier: "auto"`
- يتم ربط `/fast off` إلى `service_tier: "standard_only"`
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

قيود مهمة:

- لا يحقن OpenClaw مستويات خدمة Anthropic إلا لطلبات `api.anthropic.com` المباشرة. إذا قمت بتوجيه `anthropic/*` عبر وكيل أو بوابة، فسيترك `/fast` قيمة `service_tier` كما هي.
- تتجاوز معاملات نموذج Anthropic الصريحة `serviceTier` أو `service_tier` الإعداد الافتراضي لـ `/fast` عند تعيين الاثنين معًا.
- تعرض Anthropic المستوى الفعلي على الاستجابة ضمن `usage.service_tier`. وفي الحسابات التي لا تحتوي على سعة Priority Tier، قد تظل `service_tier: "auto"` تُحل إلى `standard`.

## التخزين المؤقت للطلبات (Anthropic API)

يدعم OpenClaw ميزة التخزين المؤقت للطلبات الخاصة بـ Anthropic. هذه الميزة **خاصة بـ API فقط**؛ إذ لا تحترم مصادقة رموز Anthropic القديمة إعدادات التخزين المؤقت.

### الإعدادات

استخدم المعامل `cacheRetention` في إعداد النموذج لديك:

| القيمة   | مدة التخزين المؤقت | الوصف                    |
| ------- | -------------- | ------------------------ |
| `none`  | بدون تخزين مؤقت     | تعطيل التخزين المؤقت للطلبات   |
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

### الإعدادات الافتراضية

عند استخدام مصادقة مفتاح Anthropic API، يطبّق OpenClaw تلقائيًا `cacheRetention: "short"` (تخزين مؤقت لمدة 5 دقائق) على جميع نماذج Anthropic. ويمكنك تجاوز ذلك بتعيين `cacheRetention` صراحةً في إعداداتك.

### تجاوزات `cacheRetention` لكل وكيل

استخدم معاملات مستوى النموذج كخط أساس، ثم تجاوز وكلاء محددين عبر `agents.list[].params`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" }, // خط الأساس لمعظم الوكلاء
        },
      },
    },
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } }, // تجاوز لهذا الوكيل فقط
    ],
  },
}
```

ترتيب دمج الإعدادات للمعاملات المرتبطة بالتخزين المؤقت:

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params` (مطابقة `id`، مع التجاوز حسب المفتاح)

يسمح هذا لوكيل واحد بالاحتفاظ بذاكرة مؤقتة طويلة العمر بينما يعطّل وكيل آخر على النموذج نفسه التخزين المؤقت لتجنب تكاليف الكتابة على الحركة المتقطعة أو منخفضة إعادة الاستخدام.

### ملاحظات Bedrock Claude

- تقبل نماذج Anthropic Claude على Bedrock (`amazon-bedrock/*anthropic.claude*`) تمرير `cacheRetention` عند تهيئتها.
- تُفرض القيمة `cacheRetention: "none"` في وقت التشغيل على نماذج Bedrock غير التابعة لـ Anthropic.
- تقوم الإعدادات الذكية الافتراضية لمفتاح Anthropic API أيضًا بتهيئة `cacheRetention: "short"` لمراجع نماذج Claude-on-Bedrock عندما لا يتم تعيين قيمة صريحة.

## نافذة سياق 1M‏ (نسخة Anthropic التجريبية)

إن نافذة السياق 1M الخاصة بـ Anthropic مقيّدة بمرحلة تجريبية. في OpenClaw، فعّلها لكل نموذج
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

يقوم OpenClaw بربط هذا إلى `anthropic-beta: context-1m-2025-08-07` على طلبات Anthropic.

لا يتم تفعيل هذا إلا عندما تكون `params.context1m` مضبوطة صراحةً على `true` لذلك
النموذج.

المتطلب: يجب أن تسمح Anthropic باستخدام السياق الطويل مع بيانات الاعتماد تلك
(عادةً فوترة مفتاح API، أو مسار تسجيل دخول Claude في OpenClaw / مصادقة الرمز القديمة
مع تفعيل Extra Usage). وإلا تُرجع Anthropic:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

ملاحظة: ترفض Anthropic حاليًا طلبات النسخة التجريبية `context-1m-*` عند استخدام
مصادقة رمز Anthropic القديمة (`sk-ant-oat-*`). إذا قمت بتهيئة
`context1m: true` مع وضع المصادقة القديم هذا، فسيسجل OpenClaw تحذيرًا ويعود
إلى نافذة السياق القياسية عن طريق تخطي ترويسة النسخة التجريبية context1m
مع الإبقاء على إصدارات OAuth التجريبية المطلوبة.

## تمت الإزالة: الواجهة الخلفية لـ Claude CLI

تمت إزالة الواجهة الخلفية المضمّنة `claude-cli` الخاصة بـ Anthropic.

- يشير إشعار Anthropic بتاريخ 4 أبريل 2026 إلى أن حركة Claude-login المدفوعة بواسطة OpenClaw
  تُعد استخدامًا لحزمة طرف ثالث وتتطلب **Extra Usage**.
- كما تُظهر إعادة الإنتاج المحلية لدينا أن
  `claude -p --append-system-prompt ...` المباشر قد يصطدم بالحماية نفسها عندما
  يعرّف الطلب المضاف OpenClaw.
- لا يصطدم طلب النظام المشابه لـ OpenClaw نفسه بهذه الحماية على
  مسار Anthropic SDK + `ANTHROPIC_API_KEY`.
- استخدم مفاتيح Anthropic API لحركة Anthropic في OpenClaw.

## ملاحظات

- لا تزال مستندات Claude Code العامة من Anthropic توثق الاستخدام المباشر لـ CLI مثل
  `claude -p`، لكن الإشعار المنفصل من Anthropic إلى مستخدمي OpenClaw ينص على أن
  مسار Claude-login الخاص بـ **OpenClaw** يُعد استخدامًا لحزمة طرف ثالث ويتطلب
  **Extra Usage** (الدفع حسب الاستخدام، وتتم فوترته بشكل منفصل عن الاشتراك).
  كما تُظهر إعادة الإنتاج المحلية لدينا أن
  `claude -p --append-system-prompt ...` المباشر قد يصطدم بالحماية نفسها عندما
  يعرّف الطلب المضاف OpenClaw، بينما لا يُعاد إنتاج ذلك مع صيغة الطلب نفسها على
  مسار Anthropic SDK + `ANTHROPIC_API_KEY`. للإنتاج، نحن
  نوصي باستخدام مفاتيح Anthropic API بدلًا من ذلك.
- أصبح رمز إعداد Anthropic متاحًا مرة أخرى في OpenClaw كمسار قديم/يدوي. ولا يزال إشعار الفوترة الخاص بـ OpenClaw من Anthropic ساريًا، لذا استخدمه مع توقع أن Anthropic تتطلب **Extra Usage** لهذا المسار.
- توجد تفاصيل المصادقة + قواعد إعادة الاستخدام في [/concepts/oauth](/ar/concepts/oauth).

## استكشاف الأخطاء وإصلاحها

**أخطاء 401 / أصبح الرمز غير صالح فجأة**

- قد تنتهي صلاحية مصادقة رمز Anthropic القديمة أو تُلغى.
- بالنسبة إلى الإعداد الجديد، انتقل إلى مفتاح Anthropic API.

**لم يتم العثور على مفتاح API للموفر "anthropic"**

- المصادقة **لكل وكيل**. لا ترث الوكلاء الجدد مفاتيح الوكيل الرئيسي.
- أعد تشغيل الإعداد الأولي لهذا الوكيل، أو قم بتهيئة مفتاح API على مضيف
  البوابة، ثم تحقّق باستخدام `openclaw models status`.

**لم يتم العثور على بيانات اعتماد لملف التعريف `anthropic:default`**

- شغّل `openclaw models status` لمعرفة ملف تعريف المصادقة النشط.
- أعد تشغيل الإعداد الأولي، أو قم بتهيئة مفتاح API لذلك المسار الخاص بملف التعريف.

**لا يوجد ملف تعريف مصادقة متاح (جميعها في التهدئة/غير متاحة)**

- تحقّق من `openclaw models status --json` لمعرفة `auth.unusableProfiles`.
- يمكن أن تكون فترات تهدئة حد المعدل في Anthropic مرتبطة بالنموذج، لذا قد يظل نموذج Anthropic
  شقيق قابلًا للاستخدام حتى عندما يكون النموذج الحالي في فترة تهدئة.
- أضف ملف تعريف Anthropic آخر أو انتظر انتهاء فترة التهدئة.

المزيد: [/gateway/troubleshooting](/ar/gateway/troubleshooting) و [/help/faq](/ar/help/faq).
