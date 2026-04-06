---
read_when:
    - تريد استخدام Qwen مع OpenClaw
    - سبق لك استخدام Qwen OAuth
summary: استخدم Qwen Cloud عبر مزوّد `qwen` المجمّع في OpenClaw
title: Qwen
x-i18n:
    generated_at: "2026-04-06T03:11:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: f175793693ab6a4c3f1f4d42040e673c15faf7603a500757423e9e06977c989d
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**تمت إزالة Qwen OAuth.** لم يعد تكامل OAuth الخاص بالفئة المجانية
(`qwen-portal`) الذي كان يستخدم نقاط نهاية `portal.qwen.ai` متاحًا.
راجع [Issue #49557](https://github.com/openclaw/openclaw/issues/49557) للاطلاع
على الخلفية.

</Warning>

## الموصى به: Qwen Cloud

يعامل OpenClaw الآن Qwen على أنه مزوّد مجمّع أساسي ذو معرّف قانوني
`qwen`. ويستهدف المزوّد المجمّع نقاط نهاية Qwen Cloud / Alibaba DashScope و
Coding Plan، ويحافظ على استمرار عمل معرّفات `modelstudio` القديمة
كاسم بديل للتوافق.

- المزوّد: `qwen`
- متغير env المفضل: `QWEN_API_KEY`
- المقبول أيضًا للتوافق: `MODELSTUDIO_API_KEY`، `DASHSCOPE_API_KEY`
- نمط API: متوافق مع OpenAI

إذا كنت تريد `qwen3.6-plus`، ففضّل نقطة النهاية **Standard (الدفع حسب الاستخدام)**.
وقد يتأخر دعم Coding Plan عن الفهرس العام.

```bash
# نقطة نهاية Coding Plan العالمية
openclaw onboard --auth-choice qwen-api-key

# نقطة نهاية Coding Plan في الصين
openclaw onboard --auth-choice qwen-api-key-cn

# نقطة نهاية Standard (الدفع حسب الاستخدام) العالمية
openclaw onboard --auth-choice qwen-standard-api-key

# نقطة نهاية Standard (الدفع حسب الاستخدام) في الصين
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

لا تزال معرّفات `modelstudio-*` القديمة لـ auth-choice ومراجع النماذج `modelstudio/...`
تعمل كأسماء بديلة للتوافق، لكن ينبغي أن تفضّل تدفقات الإعداد الجديدة
معرّفات `qwen-*` القانونية لـ auth-choice ومراجع `qwen/...` للنماذج.

بعد onboarding، عيّن نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## أنواع الخطط ونقاط النهاية

| الخطة                      | المنطقة | auth-choice                | نقطة النهاية                                     |
| -------------------------- | ------- | -------------------------- | ------------------------------------------------ |
| Standard (الدفع حسب الاستخدام) | الصين   | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (الدفع حسب الاستخدام) | عالمي   | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (اشتراك)       | الصين   | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (اشتراك)       | عالمي   | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

يختار المزوّد نقطة النهاية تلقائيًا بناءً على auth-choice الخاص بك. وتستخدم
الخيارات القانونية عائلة `qwen-*`؛ أما `modelstudio-*` فتبقى للتوافق فقط.
ويمكنك التجاوز باستخدام `baseUrl` مخصص في config.

تعلن نقاط نهاية Model Studio الأصلية عن توافق استخدام البث على
وسيط `openai-completions` المشترك. ويربط OpenClaw ذلك الآن بإمكانات نقطة النهاية،
لذلك ترث معرّفات المزوّدات المخصصة المتوافقة مع DashScope التي تستهدف
المضيفين الأصليين أنفسهم سلوك استخدام البث نفسه بدلًا من
اشتراط معرّف المزوّد المضمّن `qwen` تحديدًا.

## احصل على مفتاح API الخاص بك

- **إدارة المفاتيح**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **المستندات**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## الفهرس المضمّن

يشحن OpenClaw حاليًا فهرس Qwen المجمّع التالي:

| مرجع النموذج                | الإدخال      | السياق    | ملاحظات                                              |
| -------------------------- | ------------ | --------- | ---------------------------------------------------- |
| `qwen/qwen3.5-plus`        | نص، صورة     | 1,000,000 | النموذج الافتراضي                                    |
| `qwen/qwen3.6-plus`        | نص، صورة     | 1,000,000 | فضّل نقاط نهاية Standard عندما تحتاج هذا النموذج      |
| `qwen/qwen3-max-2026-01-23`| نص           | 262,144   | سلسلة Qwen Max                                       |
| `qwen/qwen3-coder-next`    | نص           | 262,144   | للبرمجة                                              |
| `qwen/qwen3-coder-plus`    | نص           | 1,000,000 | للبرمجة                                              |
| `qwen/MiniMax-M2.5`        | نص           | 1,000,000 | الاستدلال مفعّل                                      |
| `qwen/glm-5`               | نص           | 202,752   | GLM                                                  |
| `qwen/glm-4.7`             | نص           | 202,752   | GLM                                                  |
| `qwen/kimi-k2.5`           | نص، صورة     | 262,144   | Moonshot AI عبر Alibaba                              |

قد يظل التوفر مختلفًا حسب نقطة النهاية وخطة الفوترة حتى عندما يكون النموذج
موجودًا في الفهرس المجمّع.

ينطبق توافق استخدام البث الأصلي على كل من مضيفي Coding Plan
ومضيفي Standard المتوافقين مع DashScope:

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## توفر Qwen 3.6 Plus

يتوفر `qwen3.6-plus` على نقاط نهاية Model Studio من نوع
Standard (الدفع حسب الاستخدام):

- الصين: `dashscope.aliyuncs.com/compatible-mode/v1`
- عالمي: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

إذا أعادت نقاط نهاية Coding Plan خطأ "unsupported model" للنموذج
`qwen3.6-plus`، فانتقل إلى Standard (الدفع حسب الاستخدام) بدلًا من زوج
نقطة النهاية/المفتاح الخاص بـ Coding Plan.

## خطة الإمكانات

يجري وضع إضافة `qwen` على أنها موطن المورّد للسطح الكامل الخاص بـ Qwen
Cloud، وليس فقط لنماذج البرمجة/النص.

- نماذج النص/الدردشة: مضمّنة الآن
- استدعاء الأدوات، والمخرجات المهيكلة، والتفكير: موروثة من الوسيط المتوافق مع OpenAI
- توليد الصور: مخطط له على مستوى إضافة المزوّد
- فهم الصور/الفيديو: مضمّن الآن على نقطة النهاية Standard
- الكلام/الصوت: مخطط له على مستوى إضافة المزوّد
- تضمينات الذاكرة/إعادة الترتيب: مخطط لها عبر سطح محول التضمين
- توليد الفيديو: مضمّن الآن عبر إمكانية توليد الفيديو المشتركة

## إضافات متعددة الوسائط

تكشف إضافة `qwen` الآن أيضًا عن:

- فهم الفيديو عبر `qwen-vl-max-latest`
- توليد فيديو Wan عبر:
  - `wan2.6-t2v` (الافتراضي)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

تستخدم هذه الأسطح متعددة الوسائط نقاط نهاية DashScope من نوع **Standard**، وليس
نقاط نهاية Coding Plan.

- `base URL` القياسي العالمي/الدولي: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- `base URL` القياسي في الصين: `https://dashscope.aliyuncs.com/compatible-mode/v1`

وبالنسبة إلى توليد الفيديو، يربط OpenClaw منطقة Qwen المهيأة بمضيف
DashScope AIGC الإقليمي المطابق قبل إرسال المهمة:

- عالمي/دولي: `https://dashscope-intl.aliyuncs.com`
- الصين: `https://dashscope.aliyuncs.com`

وهذا يعني أن القيمة العادية `models.providers.qwen.baseUrl` التي تشير إلى أي من
مضيفي Qwen من نوع Coding Plan أو Standard ستبقي توليد الفيديو على
نقطة نهاية فيديو DashScope الإقليمية الصحيحة.

وبالنسبة إلى توليد الفيديو، عيّن نموذجًا افتراضيًا صراحةً:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

حدود توليد الفيديو الحالية المجمعة في Qwen:

- حتى **1** فيديو خرج لكل طلب
- حتى **1** صورة إدخال
- حتى **4** فيديوهات إدخال
- مدة تصل إلى **10 ثوانٍ**
- يدعم `size` و`aspectRatio` و`resolution` و`audio` و`watermark`
- يتطلب وضع الصورة/الفيديو المرجعي حاليًا **عناوين URL بعيدة من نوع http(s)**. تُرفض
  مسارات الملفات المحلية مسبقًا لأن نقطة نهاية فيديو DashScope لا
  تقبل رفع مخازن محلية لتلك المراجع.

راجع [Video Generation](/tools/video-generation) للاطلاع على
المعلمات المشتركة للأداة، واختيار المزوّد، وسلوك failover.

## ملاحظة حول البيئة

إذا كانت Gateway تعمل كخدمة daemon ‏(launchd/systemd)، فتأكد من أن `QWEN_API_KEY`
متاح لتلك العملية (مثلًا في `~/.openclaw/.env` أو عبر
`env.shellEnv`).
