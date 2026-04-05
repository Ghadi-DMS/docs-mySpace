---
read_when:
    - أنت تريد تعديل بيانات الاعتماد أو الأجهزة أو الإعدادات الافتراضية للوكيل بشكل تفاعلي
summary: مرجع CLI للأمر `openclaw configure` ‏(مطالبات الإعداد التفاعلية)
title: configure
x-i18n:
    generated_at: "2026-04-05T12:37:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 989569fdb8e1b31ce3438756b3ed9bf18e0c8baf611c5981643ba5925459c98f
    source_path: cli/configure.md
    workflow: 15
---

# `openclaw configure`

مطالبة تفاعلية لإعداد بيانات الاعتماد والأجهزة والإعدادات الافتراضية للوكيل.

ملاحظة: يتضمن قسم **Model** الآن اختيارًا متعددًا لقائمة السماح
`agents.defaults.models` ‏(ما يظهر في `/model` وفي منتقي النموذج).

عندما يبدأ configure من اختيار مصادقة مزوّد، فإن منتقيات النموذج الافتراضي
وقائمة السماح تفضّل ذلك المزوّد تلقائيًا. وبالنسبة إلى المزوّدين المقترنين مثل
Volcengine/BytePlus، فإن التفضيل نفسه يطابق أيضًا متغيرات خطة البرمجة الخاصة بهم
(`volcengine-plan/*` و`byteplus-plan/*`). وإذا كان عامل تصفية
المزوّد المفضّل سينتج قائمة فارغة، فإن configure يعود إلى الفهرس غير المصفّى
بدلًا من إظهار منتقٍ فارغ.

نصيحة: يؤدي تشغيل `openclaw config` من دون أمر فرعي إلى فتح المعالج نفسه. استخدم
`openclaw config get|set|unset` للتعديلات غير التفاعلية.

بالنسبة إلى البحث على الويب، يتيح لك `openclaw configure --section web` اختيار مزوّد
وتهيئة بيانات اعتماده. وتعرض بعض المزوّدات أيضًا مطالبات متابعة خاصة بالمزوّد:

- يمكن لـ **Grok** أن يقدّم إعداد `x_search` اختياريًا باستخدام `XAI_API_KEY` نفسه
  ويتيح لك اختيار نموذج `x_search`.
- يمكن لـ **Kimi** أن يطلب منطقة Moonshot API ‏(`api.moonshot.ai` مقابل
  `api.moonshot.cn`) ونموذج البحث على الويب الافتراضي لـ Kimi.

ذو صلة:

- مرجع إعدادات Gateway: [Configuration](/gateway/configuration)
- Config CLI: [Config](/cli/config)

## الخيارات

- `--section <section>`: عامل تصفية للأقسام قابل للتكرار

الأقسام المتاحة:

- `workspace`
- `model`
- `web`
- `gateway`
- `daemon`
- `channels`
- `plugins`
- `skills`
- `health`

ملاحظات:

- يؤدي اختيار مكان تشغيل Gateway دائمًا إلى تحديث `gateway.mode`. ويمكنك اختيار "Continue" من دون أقسام أخرى إذا كان هذا كل ما تحتاج إليه.
- تطلب الخدمات الموجهة للقنوات (Slack/Discord/Matrix/Microsoft Teams) قوائم سماح للقنوات/الغرف أثناء الإعداد. ويمكنك إدخال الأسماء أو المعرّفات؛ ويحاول المعالج تحويل الأسماء إلى معرّفات عندما يكون ذلك ممكنًا.
- إذا شغّلت خطوة تثبيت daemon، فإن مصادقة الرمز تتطلب رمزًا، وإذا كانت `gateway.auth.token` مُدارة عبر SecretRef، فإن configure يتحقق من SecretRef لكنه لا يحفظ قيم الرمز النصية الصريحة التي تم حلها ضمن بيانات بيئة خدمة المشرف.
- إذا كانت مصادقة الرمز تتطلب رمزًا وكان SecretRef المهيأ للرمز غير محلول، فإن configure يحظر تثبيت daemon مع إرشادات معالجة واضحة.
- إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مهيأين وكانت `gateway.auth.mode` غير مضبوطة، فإن configure يحظر تثبيت daemon حتى يتم ضبط الوضع صراحةً.

## أمثلة

```bash
openclaw configure
openclaw configure --section web
openclaw configure --section model --section channels
openclaw configure --section gateway --section daemon
```
