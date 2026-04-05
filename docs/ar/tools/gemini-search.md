---
read_when:
    - عندما تريد استخدام Gemini مع web_search
    - عندما تحتاج إلى GEMINI_API_KEY
    - عندما تريد الاستناد إلى Google Search
summary: بحث Gemini على الويب مع الاستناد إلى Google Search
title: بحث Gemini
x-i18n:
    generated_at: "2026-04-05T12:58:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 42644176baca6b4b041142541618f6f68361d410d6f425cc4104cd88d9f7c480
    source_path: tools/gemini-search.md
    workflow: 15
---

# بحث Gemini

يدعم OpenClaw نماذج Gemini مع
[الاستناد المدمج إلى Google Search](https://ai.google.dev/gemini-api/docs/grounding)،
الذي يعيد إجابات مركبة بالذكاء الاصطناعي ومدعومة بنتائج Google Search الحية مع
استشهادات.

## احصل على مفتاح API

<Steps>
  <Step title="إنشاء مفتاح">
    انتقل إلى [Google AI Studio](https://aistudio.google.com/apikey) وأنشئ
    مفتاح API.
  </Step>
  <Step title="تخزين المفتاح">
    اضبط `GEMINI_API_KEY` في بيئة Gateway، أو قم بالتهيئة عبر:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## التكوين

```json5
{
  plugins: {
    entries: {
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // optional if GEMINI_API_KEY is set
            model: "gemini-2.5-flash", // default
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "gemini",
      },
    },
  },
}
```

**بديل البيئة:** اضبط `GEMINI_API_KEY` في بيئة Gateway.
بالنسبة إلى تثبيت gateway، ضعه في `~/.openclaw/.env`.

## كيف يعمل

على عكس موفري البحث التقليديين الذين يعيدون قائمة من الروابط والمقتطفات،
يستخدم Gemini الاستناد إلى Google Search لإنتاج إجابات مركبة بالذكاء الاصطناعي مع
استشهادات مضمّنة. وتتضمن النتائج كلاً من الإجابة المركبة وعناوين URL
الخاصة بالمصادر.

- تُحل عناوين URL الخاصة بالاستشهادات من استناد Gemini تلقائيًا من عناوين
  إعادة التوجيه الخاصة بـ Google إلى عناوين URL مباشرة.
- يستخدم حل إعادة التوجيه مسار الحماية من SSRF ‏(فحوصات HEAD + إعادة التوجيه +
  التحقق من `http/https`) قبل إرجاع عنوان URL النهائي للاستشهاد.
- يستخدم حل إعادة التوجيه إعدادات SSRF صارمة افتراضيًا، لذا يتم حظر عمليات إعادة التوجيه إلى
  أهداف خاصة/داخلية.

## المعلمات المدعومة

يدعم بحث Gemini المعلمة `query`.

يُقبل `count` من أجل التوافق مع `web_search` المشترك، لكن استناد Gemini
لا يزال يعيد إجابة مركبة واحدة مع استشهادات بدلًا من
قائمة من N نتائج.

لا يتم دعم عوامل التصفية الخاصة بالموفر مثل `country` و`language` و`freshness` و
`domain_filter`.

## اختيار النموذج

النموذج الافتراضي هو `gemini-2.5-flash` (سريع وفعّال من حيث التكلفة). يمكن استخدام أي نموذج Gemini
يدعم الاستناد عبر
`plugins.entries.google.config.webSearch.model`.

## ذو صلة

- [نظرة عامة على البحث على الويب](/tools/web) -- جميع الموفّرين والكشف التلقائي
- [Brave Search](/tools/brave-search) -- نتائج منظمة مع مقتطفات
- [Perplexity Search](/tools/perplexity-search) -- نتائج منظمة + استخراج المحتوى
