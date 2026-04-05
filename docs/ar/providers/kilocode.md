---
read_when:
    - أنت تريد مفتاح API واحدًا للعديد من LLMs
    - أنت تريد تشغيل النماذج عبر Kilo Gateway في OpenClaw
summary: استخدم Kilo Gateway API الموحدة للوصول إلى العديد من النماذج في OpenClaw
title: Kilo Gateway
x-i18n:
    generated_at: "2026-04-05T12:53:19Z"
    model: gpt-5.4
    provider: openai
    source_hash: 857266967b4a7553d501990631df2bae0f849d061521dc9f34e29687ecb94884
    source_path: providers/kilocode.md
    workflow: 15
---

# Kilo Gateway

يوفر Kilo Gateway **API موحدة** توجّه الطلبات إلى العديد من النماذج خلف
نقطة نهاية واحدة ومفتاح API واحد. وهو متوافق مع OpenAI، لذلك تعمل معظم حزم SDK الخاصة بـ OpenAI بمجرد تبديل base URL.

## الحصول على مفتاح API

1. انتقل إلى [app.kilo.ai](https://app.kilo.ai)
2. سجّل الدخول أو أنشئ حسابًا
3. انتقل إلى API Keys وأنشئ مفتاحًا جديدًا

## إعداد CLI

```bash
openclaw onboard --auth-choice kilocode-api-key
```

أو اضبط متغير البيئة:

```bash
export KILOCODE_API_KEY="<your-kilocode-api-key>" # pragma: allowlist secret
```

## مقتطف إعدادات

```json5
{
  env: { KILOCODE_API_KEY: "<your-kilocode-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "kilocode/kilo/auto" },
    },
  },
}
```

## النموذج الافتراضي

النموذج الافتراضي هو `kilocode/kilo/auto`، وهو نموذج توجيه ذكي مملوك للمزوّد
تديره Kilo Gateway.

يتعامل OpenClaw مع `kilocode/kilo/auto` على أنه مرجع افتراضي ثابت، لكنه لا
ينشر تعيينًا مدعومًا بالمصدر من المهمة إلى النموذج الأصلي لهذا المسار.

## النماذج المتاحة

يكتشف OpenClaw النماذج المتاحة ديناميكيًا من Kilo Gateway عند بدء التشغيل. استخدم
`/models kilocode` لرؤية القائمة الكاملة للنماذج المتاحة مع حسابك.

يمكن استخدام أي نموذج متاح على gateway مع البادئة `kilocode/`:

```
kilocode/kilo/auto              (الافتراضي - توجيه ذكي)
kilocode/anthropic/claude-sonnet-4
kilocode/openai/gpt-5.4
kilocode/google/gemini-3-pro-preview
...والعديد غيرها
```

## ملاحظات

- تكون مراجع النماذج بصيغة `kilocode/<model-id>` ‏(مثل `kilocode/anthropic/claude-sonnet-4`).
- النموذج الافتراضي: `kilocode/kilo/auto`
- Base URL: ‏`https://api.kilo.ai/api/gateway/`
- يتضمن فهرس التراجع المضمّن دائمًا `kilocode/kilo/auto` ‏(`Kilo Auto`) مع
  `input: ["text", "image"]`، و`reasoning: true`، و`contextWindow: 1000000`،
  و`maxTokens: 128000`
- عند بدء التشغيل، يحاول OpenClaw استخدام `GET https://api.kilo.ai/api/gateway/models` ويقوم
  بدمج النماذج المكتشفة قبل فهرس التراجع الثابت
- التوجيه الأصلي الدقيق خلف `kilocode/kilo/auto` مملوك لـ Kilo Gateway،
  وليس مبرمجًا بشكل ثابت داخل OpenClaw
- يتم توثيق Kilo Gateway في المصدر على أنه متوافق مع OpenRouter، لذلك يبقى على
  مسار proxy-style المتوافق مع OpenAI بدلًا من تشكيل الطلبات الأصلية لـ OpenAI
- تبقى مراجع Kilo المدعومة بواسطة Gemini على مسار proxy-Gemini، لذلك يحافظ OpenClaw على
  تنظيف توقيع التفكير الخاص بـ Gemini هناك من دون تمكين
  التحقق الأصلي من إعادة تشغيل Gemini أو إعادة كتابة bootstrap.
- يضيف غلاف البث المشترك الخاص بـ Kilo ترويسة تطبيق المزوّد ويطبع
  حمولات reasoning الخاصة بالـ proxy لمراجع النماذج الملموسة المدعومة. أما `kilocode/kilo/auto`
  والتلميحات الأخرى غير المدعومة لـ proxy reasoning فتتخطى حقن reasoning هذا.
- لمزيد من خيارات النماذج/المزوّدين، راجع [/concepts/model-providers](/concepts/model-providers).
- يستخدم Kilo Gateway رمز Bearer مع مفتاح API الخاص بك داخليًا.
