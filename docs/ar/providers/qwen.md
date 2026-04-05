---
x-i18n:
    generated_at: "2026-04-05T12:53:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 895b701d3a3950ea7482e5e870663ed93e0355e679199ed4622718d588ef18fa
    source_path: providers/qwen.md
    workflow: 15
---

summary: "استخدم Qwen Cloud عبر مزوّد qwen المضمّن في OpenClaw"
read_when:

- أنت تريد استخدام Qwen مع OpenClaw
- كنت تستخدم Qwen OAuth سابقًا
  title: "Qwen"

---

# Qwen

<Warning>

**تمت إزالة Qwen OAuth.** لم يعد تكامل OAuth الخاص بالمستوى المجاني
(`qwen-portal`) الذي كان يستخدم نقاط نهاية `portal.qwen.ai` متاحًا.
راجع [Issue #49557](https://github.com/openclaw/openclaw/issues/49557) للحصول على
الخلفية.

</Warning>

## الموصى به: Qwen Cloud

يتعامل OpenClaw الآن مع Qwen كمزوّد مضمّن من الدرجة الأولى بمعرّف معياري
هو `qwen`. ويستهدف المزوّد المضمّن نقاط نهاية Qwen Cloud / Alibaba DashScope و
Coding Plan مع الإبقاء على معرّفات `modelstudio` القديمة
كاسم بديل للتوافق.

- المزوّد: `qwen`
- متغير البيئة المفضل: `QWEN_API_KEY`
- المقبول أيضًا للتوافق: `MODELSTUDIO_API_KEY`، `DASHSCOPE_API_KEY`
- نمط API: متوافق مع OpenAI

إذا كنت تريد `qwen3.6-plus`، ففضّل نقطة نهاية **Standard (pay-as-you-go)**.
قد يتأخر دعم Coding Plan عن الفهرس العام.

```bash
# Global Coding Plan endpoint
openclaw onboard --auth-choice qwen-api-key

# China Coding Plan endpoint
openclaw onboard --auth-choice qwen-api-key-cn

# Global Standard (pay-as-you-go) endpoint
openclaw onboard --auth-choice qwen-standard-api-key

# China Standard (pay-as-you-go) endpoint
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

لا تزال معرّفات `modelstudio-*` القديمة الخاصة بـ auth-choice ومراجع النماذج `modelstudio/...`
تعمل كأسماء بديلة للتوافق، لكن يجب أن تفضّل تدفقات الإعداد الجديدة
معرّفات `qwen-*` المعيارية الخاصة بـ auth-choice ومراجع النماذج `qwen/...`.

بعد onboarding، اضبط نموذجًا افتراضيًا:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## خطة القدرات

يجري وضع extension ‏`qwen` كموطن المزوّد لسطح Qwen
Cloud الكامل، وليس فقط لنماذج البرمجة/النص.

- نماذج النص/الدردشة: مضمّنة الآن
- استدعاء الأدوات، والخرج المنظم، والتفكير: موروثة من النقل المتوافق مع OpenAI
- توليد الصور: مخطط له على مستوى provider-plugin
- فهم الصور/الفيديو: مضمّن الآن على نقطة نهاية Standard
- الكلام/الصوت: مخطط له على مستوى provider-plugin
- التضمينات/إعادة الترتيب للذاكرة: مخطط لها عبر سطح محول التضمين
- توليد الفيديو: مضمّن الآن عبر قدرة توليد الفيديو المشتركة

## الإضافات متعددة الوسائط

يكشف extension ‏`qwen` الآن أيضًا عن:

- فهم الفيديو عبر `qwen-vl-max-latest`
- توليد فيديو Wan عبر:
  - `wan2.6-t2v` ‏(الافتراضي)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

تستخدم هذه الأسطح متعددة الوسائط نقاط نهاية DashScope **Standard**، وليس
نقاط نهاية Coding Plan.

- عنوان URL الأساسي العالمي/الدولي لـ Standard: ‏`https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- عنوان URL الأساسي لـ Standard في الصين: ‏`https://dashscope.aliyuncs.com/compatible-mode/v1`

بالنسبة إلى توليد الفيديو، يربط OpenClaw منطقة Qwen المهيأة بمضيف
DashScope AIGC المطابق قبل إرسال المهمة:

- عالمي/دولي: `https://dashscope-intl.aliyuncs.com`
- الصين: `https://dashscope.aliyuncs.com`

وهذا يعني أن `models.providers.qwen.baseUrl` العادي الذي يشير إلى أي من
مضيفي Qwen الخاصين بـ Coding Plan أو Standard سيُبقي توليد الفيديو على
نقطة نهاية فيديو DashScope الإقليمية الصحيحة.

بالنسبة إلى توليد الفيديو، اضبط نموذجًا افتراضيًا صراحةً:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

الحدود الحالية المضمنة لتوليد الفيديو في Qwen:

- حتى **1** فيديو خرج لكل طلب
- حتى **1** صورة إدخال
- حتى **4** مقاطع فيديو إدخال
- حتى **10 ثوانٍ** مدة
- يدعم `size` و`aspectRatio` و`resolution` و`audio` و`watermark`

راجع [Qwen / Model Studio](/providers/qwen_modelstudio) للحصول على تفاصيل
على مستوى نقطة النهاية وملاحظات التوافق.
