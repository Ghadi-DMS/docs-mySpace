---
read_when:
    - تريد استخدام Kimi من أجل `web_search`
    - تحتاج إلى `KIMI_API_KEY` أو `MOONSHOT_API_KEY`
summary: بحث Kimi على الويب عبر بحث Moonshot على الويب
title: بحث Kimi
x-i18n:
    generated_at: "2026-04-05T12:58:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 753757a5497a683c35b4509ed3709b9514dc14a45612675d0f729ae6668c82a5
    source_path: tools/kimi-search.md
    workflow: 15
---

# بحث Kimi

يدعم OpenClaw Kimi بصفته موفّر `web_search`، باستخدام بحث Moonshot على الويب
لإنتاج إجابات مركّبة بواسطة AI مع استشهادات.

## الحصول على مفتاح API

<Steps>
  <Step title="أنشئ مفتاحًا">
    احصل على مفتاح API من [Moonshot AI](https://platform.moonshot.cn/).
  </Step>
  <Step title="خزّن المفتاح">
    عيّن `KIMI_API_KEY` أو `MOONSHOT_API_KEY` في بيئة Gateway، أو
    قم بالتهيئة عبر:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

عندما تختار **Kimi** أثناء `openclaw onboard` أو
`openclaw configure --section web`، يمكن لـ OpenClaw أيضًا أن يسأل عن:

- منطقة Moonshot API:
  - `https://api.moonshot.ai/v1`
  - `https://api.moonshot.cn/v1`
- نموذج بحث الويب الافتراضي لـ Kimi (الافتراضي هو `kimi-k2.5`)

## التهيئة

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // اختياري إذا كان KIMI_API_KEY أو MOONSHOT_API_KEY معيّنًا
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.5",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

إذا كنت تستخدم مضيف China API للدردشة (`models.providers.moonshot.baseUrl`:
`https://api.moonshot.cn/v1`)، فإن OpenClaw يعيد استخدام المضيف نفسه من أجل
`web_search` في Kimi عند حذف `tools.web.search.kimi.baseUrl`، بحيث لا تصل المفاتيح من
[platform.moonshot.cn](https://platform.moonshot.cn/) إلى
نقطة النهاية الدولية عن طريق الخطأ (والتي كثيرًا ما تعيد HTTP 401). تجاوز ذلك
باستخدام `tools.web.search.kimi.baseUrl` عندما تحتاج إلى عنوان URL أساسي مختلف للبحث.

**بديل البيئة:** عيّن `KIMI_API_KEY` أو `MOONSHOT_API_KEY` في
بيئة Gateway. بالنسبة إلى تثبيت Gateway، ضعه في `~/.openclaw/.env`.

إذا حذفت `baseUrl`، فسيستخدم OpenClaw افتراضيًا `https://api.moonshot.ai/v1`.
وإذا حذفت `model`، فسيستخدم OpenClaw افتراضيًا `kimi-k2.5`.

## كيف يعمل

يستخدم Kimi بحث Moonshot على الويب لتركيب إجابات مع استشهادات مضمّنة،
على نحو مشابه لنهج الردود المرتكزة في Gemini وGrok.

## المعلمات المدعومة

يدعم بحث Kimi المعلمة `query`.

تُقبل `count` للتوافق المشترك مع `web_search`، لكن Kimi لا يزال
يعيد إجابة مركّبة واحدة مع استشهادات بدلًا من قائمة من N نتيجة.

لا تتوفر حاليًا عوامل تصفية خاصة بالمزوّد.

## ذو صلة

- [نظرة عامة على Web Search](/tools/web) -- جميع الموفّرين والكشف التلقائي
- [Moonshot AI](/ar/providers/moonshot) -- وثائق موفّر نموذج Moonshot + Kimi Coding
- [بحث Gemini](/tools/gemini-search) -- إجابات مركّبة بواسطة AI عبر الإسناد من Google
- [بحث Grok](/tools/grok-search) -- إجابات مركّبة بواسطة AI عبر الإسناد من xAI
