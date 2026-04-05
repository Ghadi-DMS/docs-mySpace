---
read_when:
    - تريد استخدام Brave Search مع `web_search`
    - تحتاج إلى `BRAVE_API_KEY` أو تفاصيل الخطة
summary: إعداد Brave Search API لـ `web_search`
title: Brave Search
x-i18n:
    generated_at: "2026-04-05T12:57:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: bc026a69addf74375a0e407805b875ff527c77eb7298b2f5bb0e165197f77c0c
    source_path: tools/brave-search.md
    workflow: 15
---

# Brave Search API

يدعم OpenClaw واجهة Brave Search API كمزوّد `web_search`.

## الحصول على مفتاح API

1. أنشئ حساب Brave Search API على [https://brave.com/search/api/](https://brave.com/search/api/)
2. في لوحة المعلومات، اختر خطة **Search** وأنشئ مفتاح API.
3. خزّن المفتاح في الإعدادات أو اضبط `BRAVE_API_KEY` في بيئة Gateway.

## مثال للإعدادات

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // or "llm-context"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

توجد الآن إعدادات Brave الخاصة بالمزوّد ضمن `plugins.entries.brave.config.webSearch.*`.
ولا يزال المسار القديم `tools.web.search.apiKey` يُحمَّل عبر طبقة التوافق، لكنه لم يعد مسار الإعدادات الأساسي.

يتحكم `webSearch.mode` في وسيلة نقل Brave:

- `web` (الافتراضي): بحث ويب عادي من Brave مع العناوين وعناوين URL والمقتطفات
- `llm-context`: واجهة Brave LLM Context API مع مقاطع نصية مستخرجة مسبقًا ومصادر للإسناد

## معلمات الأداة

| Parameter     | Description                                                         |
| ------------- | ------------------------------------------------------------------- |
| `query`       | استعلام البحث (مطلوب)                                               |
| `count`       | عدد النتائج المطلوب إرجاعها (1-10، الافتراضي: 5)                   |
| `country`     | رمز الدولة ISO مكوّن من حرفين (مثل `"US"` و`"DE"`)                  |
| `language`    | رمز اللغة ISO 639-1 لنتائج البحث (مثل `"en"` و`"de"` و`"fr"`)       |
| `search_lang` | رمز لغة البحث في Brave (مثل `en` و`en-gb` و`zh-hans`)               |
| `ui_lang`     | رمز لغة ISO لعناصر واجهة المستخدم                                   |
| `freshness`   | مرشح الوقت: `day` (24 ساعة) أو `week` أو `month` أو `year`          |
| `date_after`  | النتائج المنشورة بعد هذا التاريخ فقط (YYYY-MM-DD)                   |
| `date_before` | النتائج المنشورة قبل هذا التاريخ فقط (YYYY-MM-DD)                   |

**أمثلة:**

```javascript
// بحث مخصص حسب الدولة واللغة
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// نتائج حديثة (خلال الأسبوع الماضي)
await web_search({
  query: "AI news",
  freshness: "week",
});

// بحث ضمن نطاق تاريخ
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## ملاحظات

- يستخدم OpenClaw خطة Brave **Search**. إذا كانت لديك اشتراك قديم (مثل الخطة المجانية الأصلية مع 2,000 استعلام/شهر)، فإنه يظل صالحًا لكنه لا يتضمن الميزات الأحدث مثل LLM Context أو حدود المعدل الأعلى.
- تتضمن كل خطة Brave **رصيدًا مجانيًا بقيمة 5 دولارات شهريًا** (يتجدد). وتبلغ تكلفة خطة Search مقدار 5 دولارات لكل 1,000 طلب، لذا يغطي الرصيد 1,000 استعلام/شهر. اضبط حد الاستخدام في لوحة معلومات Brave لتجنب الرسوم غير المتوقعة. راجع [بوابة Brave API](https://brave.com/search/api/) للاطلاع على الخطط الحالية.
- تتضمن خطة Search نقطة نهاية LLM Context وحقوق الاستدلال الخاصة بالذكاء الاصطناعي. ويتطلب تخزين النتائج لتدريب النماذج أو ضبطها خطة ذات حقوق تخزين صريحة. راجع [شروط الخدمة](https://api-dashboard.search.brave.com/terms-of-service) الخاصة بـ Brave.
- يعيد وضع `llm-context` إدخالات مصادر مؤسَّسة بدلًا من شكل مقتطفات بحث الويب المعتاد.
- لا يدعم وضع `llm-context` القيم `ui_lang` أو `freshness` أو `date_after` أو `date_before`.
- يجب أن يتضمن `ui_lang` وسم منطقة فرعيًا مثل `en-US`.
- تُخزَّن النتائج مؤقتًا لمدة 15 دقيقة افتراضيًا (ويمكن ضبطها عبر `cacheTtlMinutes`).

## ذو صلة

- [نظرة عامة على Web Search](/tools/web) -- جميع المزوّدين والاكتشاف التلقائي
- [Perplexity Search](/tools/perplexity-search) -- نتائج منظَّمة مع تصفية حسب النطاق
- [Exa Search](/tools/exa-search) -- بحث عصبي مع استخراج المحتوى
