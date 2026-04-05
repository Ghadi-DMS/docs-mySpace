---
read_when:
    - تريد استخدام Brave Search لـ `web_search`
    - تحتاج إلى `BRAVE_API_KEY` أو تفاصيل الخطة
summary: إعداد Brave Search API لـ `web_search`
title: Brave Search (المسار القديم)
x-i18n:
    generated_at: "2026-04-05T12:34:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7788e4cee7dc460819e55095c87df8cea29ba3a8bd3cef4c0e98ac601b45b651
    source_path: brave-search.md
    workflow: 15
---

# Brave Search API

يدعم OpenClaw واجهة Brave Search API باعتبارها موفّر `web_search`.

## الحصول على مفتاح API

1. أنشئ حساب Brave Search API على [https://brave.com/search/api/](https://brave.com/search/api/)
2. في لوحة التحكم، اختر خطة **Search** وأنشئ مفتاح API.
3. خزّن المفتاح في الإعدادات أو اضبط `BRAVE_API_KEY` في بيئة Gateway.

## مثال على الإعداد

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

توجد الآن إعدادات بحث Brave الخاصة بالموفّر ضمن `plugins.entries.brave.config.webSearch.*`.
ولا يزال `tools.web.search.apiKey` القديم يُحمَّل عبر طبقة التوافق، لكنه لم يعد مسار الإعداد القانوني.

يتحكم `webSearch.mode` في آلية النقل الخاصة بـ Brave:

- `web` (الافتراضي): بحث ويب عادي من Brave مع العناوين وعناوين URL والمقتطفات
- `llm-context`: واجهة Brave LLM Context API مع مقاطع نصية مستخرجة مسبقًا ومصادر للاستناد

## معلمات الأداة

| المعلمة | الوصف |
| ------------- | ------------------------------------------------------------------- |
| `query` | استعلام البحث (مطلوب) |
| `count` | عدد النتائج المطلوب إرجاعها (1-10، الافتراضي: 5) |
| `country` | رمز بلد ISO مكوّن من حرفين (مثل: "US" و"DE") |
| `language` | رمز لغة ISO 639-1 لنتائج البحث (مثل: "en" و"de" و"fr") |
| `search_lang` | رمز لغة البحث في Brave (مثل: `en` و`en-gb` و`zh-hans`) |
| `ui_lang` | رمز لغة ISO لعناصر واجهة المستخدم |
| `freshness` | عامل تصفية زمني: `day` (24 ساعة) أو `week` أو `month` أو `year` |
| `date_after` | النتائج المنشورة بعد هذا التاريخ فقط (YYYY-MM-DD) |
| `date_before` | النتائج المنشورة قبل هذا التاريخ فقط (YYYY-MM-DD) |

**أمثلة:**

```javascript
// Country and language-specific search
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// Recent results (past week)
await web_search({
  query: "AI news",
  freshness: "week",
});

// Date range search
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## ملاحظات

- يستخدم OpenClaw خطة Brave **Search**. إذا كان لديك اشتراك قديم (مثل الخطة المجانية الأصلية مع 2,000 استعلام شهريًا)، فسيظل صالحًا لكنه لا يتضمن الميزات الأحدث مثل LLM Context أو حدود المعدل الأعلى.
- تتضمن كل خطة Brave **رصيدًا مجانيًا بقيمة \$5 شهريًا** (يتجدد). وتبلغ تكلفة خطة Search مقدار \$5 لكل 1,000 طلب، لذلك يغطي الرصيد 1,000 استعلام شهريًا. اضبط حد الاستخدام في لوحة تحكم Brave لتجنب الرسوم غير المتوقعة. راجع [بوابة Brave API](https://brave.com/search/api/) للاطلاع على الخطط الحالية.
- تتضمن خطة Search نقطة نهاية LLM Context وحقوق استدلال الذكاء الاصطناعي. ويتطلب تخزين النتائج لتدريب النماذج أو ضبطها خطة تتضمن حقوق تخزين صريحة. راجع [شروط الخدمة](https://api-dashboard.search.brave.com/terms-of-service) الخاصة بـ Brave.
- يعيد وضع `llm-context` إدخالات مصدر مستندة بدلًا من شكل مقتطفات بحث الويب العادي.
- لا يدعم وضع `llm-context` كلاً من `ui_lang` و`freshness` و`date_after` و`date_before`.
- يجب أن يتضمن `ui_lang` وسمًا فرعيًا للمنطقة مثل `en-US`.
- تُخزَّن النتائج مؤقتًا لمدة 15 دقيقة افتراضيًا (يمكن ضبطها عبر `cacheTtlMinutes`).

راجع [أدوات الويب](/tools/web) للحصول على إعداد `web_search` الكامل.
