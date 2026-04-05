---
read_when:
    - أنت تريد مفتاح API واحدًا للعديد من LLMs
    - أنت تريد تشغيل النماذج عبر OpenRouter في OpenClaw
summary: استخدم API الموحدة من OpenRouter للوصول إلى العديد من النماذج في OpenClaw
title: OpenRouter
x-i18n:
    generated_at: "2026-04-05T12:53:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8dd354ba060bcb47724c89ae17c8e2af8caecac4bd996fcddb584716c1840b87
    source_path: providers/openrouter.md
    workflow: 15
---

# OpenRouter

يوفر OpenRouter **API موحدة** توجّه الطلبات إلى العديد من النماذج خلف
نقطة نهاية واحدة ومفتاح API واحد. وهو متوافق مع OpenAI، لذلك تعمل معظم حزم SDK الخاصة بـ OpenAI بمجرد تبديل base URL.

## إعداد CLI

```bash
openclaw onboard --auth-choice openrouter-api-key
```

## مقتطف إعدادات

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## ملاحظات

- تكون مراجع النماذج بصيغة `openrouter/<provider>/<model>`.
- يستخدم onboarding النموذج الافتراضي `openrouter/auto`. ويمكنك التبديل إلى نموذج ملموس لاحقًا باستخدام
  `openclaw models set openrouter/<provider>/<model>`.
- لمزيد من خيارات النماذج/المزوّدين، راجع [/concepts/model-providers](/concepts/model-providers).
- يستخدم OpenRouter رمز Bearer مع مفتاح API الخاص بك داخليًا.
- في طلبات OpenRouter الحقيقية (`https://openrouter.ai/api/v1`)، يضيف OpenClaw أيضًا
  ترويسات إسناد التطبيق الموثقة من OpenRouter:
  ‏`HTTP-Referer: https://openclaw.ai`، و`X-OpenRouter-Title: OpenClaw`، و
  `X-OpenRouter-Categories: cli-agent`.
- على مسارات OpenRouter المتحقق منها، تحتفظ مراجع نماذج Anthropic أيضًا
  بعلامات `cache_control` الخاصة بـ Anthropic في OpenRouter والتي يستخدمها OpenClaw من أجل
  إعادة استخدام أفضل لـ prompt-cache على كتل system/developer prompt.
- إذا أعدت توجيه مزوّد OpenRouter إلى proxy/base URL آخر، فلن يقوم OpenClaw
  بحقن هذه الترويسات الخاصة بـ OpenRouter أو علامات cache الخاصة بـ Anthropic.
- لا يزال OpenRouter يعمل عبر المسار المتوافق مع OpenAI بنمط proxy، لذا فإن
  تشكيل الطلبات الأصلي الخاص بـ OpenAI فقط مثل `serviceTier`، و`store` في Responses،
  وحمولات التوافق الخاصة بـ OpenAI reasoning، وتلميحات prompt-cache لا يتم تمريرها.
- تبقى مراجع OpenRouter المدعومة بواسطة Gemini على مسار proxy-Gemini: إذ يحافظ OpenClaw على
  تنظيف توقيع التفكير الخاص بـ Gemini هناك، لكنه لا يمكّن التحقق الأصلي
  من إعادة تشغيل Gemini أو إعادة كتابة bootstrap.
- على المسارات المدعومة غير `auto`، يربط OpenClaw مستوى thinking المحدد
  بحمولات reasoning الخاصة بـ OpenRouter proxy. أما تلميحات النماذج غير المدعومة
  و`openrouter/auto` فتتخطى حقن reasoning هذا.
- إذا مررت توجيه مزوّد OpenRouter ضمن معلمات النموذج، فسيقوم OpenClaw بتمريره
  كبيانات وصفية لتوجيه OpenRouter قبل تشغيل أغلفة البث المشتركة.
