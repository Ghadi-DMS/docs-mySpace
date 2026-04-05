---
read_when:
    - تريد استخدام Grok من أجل `web_search`
    - تحتاج إلى `XAI_API_KEY` من أجل البحث على الويب
summary: بحث Grok على الويب عبر استجابات xAI المرتكزة على الويب
title: بحث Grok
x-i18n:
    generated_at: "2026-04-05T12:58:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: ae2343012eebbe75d3ecdde3cb4470415c3275b694d0339bc26c46675a652054
    source_path: tools/grok-search.md
    workflow: 15
---

# بحث Grok

يدعم OpenClaw Grok بصفته موفّر `web_search`، باستخدام استجابات xAI
المرتكزة على الويب لإنتاج إجابات مركّبة بواسطة AI ومدعومة بنتائج بحث مباشرة
مع استشهادات.

يمكن لمفتاح `XAI_API_KEY` نفسه أيضًا تشغيل أداة `x_search` المضمّنة للبحث في
منشورات X ‏(Twitter سابقًا). إذا خزّنت المفتاح ضمن
`plugins.entries.xai.config.webSearch.apiKey`، فإن OpenClaw يعيد الآن استخدامه
كخيار احتياطي لموفّر نماذج xAI المضمّن أيضًا.

بالنسبة إلى مقاييس X على مستوى المنشور مثل إعادة النشر، أو الردود، أو العلامات المرجعية، أو المشاهدات، فاختر
`x_search` باستخدام عنوان URL الدقيق للمنشور أو معرّف الحالة بدلًا من استعلام
بحث عام.

## الإعداد الأولي والتهيئة

إذا اخترت **Grok** أثناء:

- `openclaw onboard`
- `openclaw configure --section web`

فيمكن لـ OpenClaw عرض خطوة متابعة منفصلة لتمكين `x_search` باستخدام
`XAI_API_KEY` نفسه. وهذه المتابعة:

- تظهر فقط بعد اختيار Grok من أجل `web_search`
- ليست خيارًا منفصلًا على المستوى الأعلى لموفّر البحث على الويب
- يمكنها اختياريًا تعيين نموذج `x_search` أثناء التدفق نفسه

إذا تخطيتها، يمكنك تمكين `x_search` أو تغييره لاحقًا في التهيئة.

## الحصول على مفتاح API

<Steps>
  <Step title="أنشئ مفتاحًا">
    احصل على مفتاح API من [xAI](https://console.x.ai/).
  </Step>
  <Step title="خزّن المفتاح">
    عيّن `XAI_API_KEY` في بيئة Gateway، أو قم بالتهيئة عبر:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## التهيئة

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // اختياري إذا كان XAI_API_KEY معيّنًا
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "grok",
      },
    },
  },
}
```

**بديل البيئة:** عيّن `XAI_API_KEY` في بيئة Gateway.
بالنسبة إلى تثبيت Gateway، ضعه في `~/.openclaw/.env`.

## كيف يعمل

يستخدم Grok استجابات xAI المرتكزة على الويب لتركيب إجابات مع استشهادات مضمّنة،
على نحو مشابه لنهج Gemini في الإسناد عبر Google Search.

## المعلمات المدعومة

يدعم بحث Grok المعلمة `query`.

تُقبل `count` للتوافق المشترك مع `web_search`، لكن Grok لا يزال
يعيد إجابة مركّبة واحدة مع استشهادات بدلًا من قائمة من N نتيجة.

لا تتوفر حاليًا عوامل تصفية خاصة بالمزوّد.

## ذو صلة

- [نظرة عامة على Web Search](/tools/web) -- جميع الموفّرين والكشف التلقائي
- [‏x_search في Web Search](/tools/web#x_search) -- بحث X من الدرجة الأولى عبر xAI
- [بحث Gemini](/tools/gemini-search) -- إجابات مركّبة بواسطة AI عبر الإسناد من Google
