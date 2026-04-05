---
read_when:
    - عندما تريد استخدام Perplexity Search للبحث على الويب
    - عندما تحتاج إلى إعداد PERPLEXITY_API_KEY أو OPENROUTER_API_KEY
summary: واجهة Perplexity Search API والتوافق مع Sonar/OpenRouter من أجل web_search
title: بحث Perplexity
x-i18n:
    generated_at: "2026-04-05T12:59:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 06d97498e26e5570364e1486cb75584ed53b40a0091bf0210e1ea62f62d562ea
    source_path: tools/perplexity-search.md
    workflow: 15
---

# واجهة Perplexity Search API

يدعم OpenClaw واجهة Perplexity Search API بوصفها موفّرًا لـ `web_search`.
وهي تعيد نتائج منظّمة تحتوي على الحقول `title` و`url` و`snippet`.

ولأجل التوافق، يدعم OpenClaw أيضًا إعدادات Perplexity Sonar/OpenRouter القديمة.
إذا كنت تستخدم `OPENROUTER_API_KEY`، أو مفتاحًا بصيغة `sk-or-...` في `plugins.entries.perplexity.config.webSearch.apiKey`، أو تضبط `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`، فسيتحول الموفّر إلى مسار chat-completions ويعيد إجابات مركبة بالذكاء الاصطناعي مع استشهادات بدلًا من نتائج Search API المنظّمة.

## الحصول على مفتاح Perplexity API

1. أنشئ حساب Perplexity على [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)
2. أنشئ مفتاح API من لوحة التحكم
3. خزّن المفتاح في التكوين أو اضبط `PERPLEXITY_API_KEY` في بيئة Gateway.

## التوافق مع OpenRouter

إذا كنت تستخدم OpenRouter بالفعل مع Perplexity Sonar، فأبقِ `provider: "perplexity"` واضبط `OPENROUTER_API_KEY` في بيئة Gateway، أو خزّن مفتاحًا بصيغة `sk-or-...` في `plugins.entries.perplexity.config.webSearch.apiKey`.

عناصر التحكم الاختيارية للتوافق:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## أمثلة التكوين

### واجهة Perplexity Search API الأصلية

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

### التوافق مع OpenRouter / Sonar

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "<openrouter-api-key>",
            baseUrl: "https://openrouter.ai/api/v1",
            model: "perplexity/sonar-pro",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

## أين تضبط المفتاح

**عبر التكوين:** شغّل `openclaw configure --section web`. فهو يخزّن المفتاح في
`~/.openclaw/openclaw.json` ضمن `plugins.entries.perplexity.config.webSearch.apiKey`.
ويقبل هذا الحقل أيضًا كائنات SecretRef.

**عبر البيئة:** اضبط `PERPLEXITY_API_KEY` أو `OPENROUTER_API_KEY`
في بيئة عملية Gateway. وبالنسبة إلى تثبيت gateway، ضعه في
`~/.openclaw/.env` (أو في بيئة الخدمة الخاصة بك). راجع [متغيرات البيئة](/ar/help/faq#env-vars-and-env-loading).

إذا كان `provider: "perplexity"` مهيأً وكان SecretRef الخاص بمفتاح Perplexity غير محلول من دون بديل env، فسيفشل بدء التشغيل/إعادة التحميل سريعًا.

## معلمات الأداة

تنطبق هذه المعلمات على مسار واجهة Perplexity Search API الأصلية.

| المعلمة              | الوصف                                                |
| -------------------- | ---------------------------------------------------- |
| `query`              | استعلام البحث (مطلوب)                                |
| `count`              | عدد النتائج المطلوب إرجاعها (1-10، الافتراضي: 5)     |
| `country`            | رمز الدولة ISO مكوّن من حرفين (مثل "US" أو "DE")    |
| `language`           | رمز اللغة ISO 639-1 (مثل "en" أو "de" أو "fr")      |
| `freshness`          | عامل تصفية زمني: `day` ‏(24 ساعة) أو `week` أو `month` أو `year` |
| `date_after`         | النتائج المنشورة بعد هذا التاريخ فقط (YYYY-MM-DD)    |
| `date_before`        | النتائج المنشورة قبل هذا التاريخ فقط (YYYY-MM-DD)    |
| `domain_filter`      | مصفوفة قائمة سماح/قائمة حظر للنطاقات (الحد الأقصى 20) |
| `max_tokens`         | إجمالي ميزانية المحتوى (الافتراضي: 25000، الحد الأقصى: 1000000) |
| `max_tokens_per_page`| حد الرموز لكل صفحة (الافتراضي: 2048)                 |

بالنسبة إلى مسار التوافق القديم Sonar/OpenRouter:

- تُقبل `query` و`count` و`freshness`
- تكون `count` للتوافق فقط هناك؛ إذ يظل الرد عبارة عن
  إجابة مركبة واحدة مع استشهادات بدلًا من قائمة من N نتائج
- تعيد عوامل التصفية الخاصة بـ Search API فقط، مثل `country` و`language` و`date_after`،
  و`date_before` و`domain_filter` و`max_tokens` و`max_tokens_per_page`
  أخطاء صريحة

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

// Domain filtering (allowlist)
await web_search({
  query: "climate research",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// Domain filtering (denylist - prefix with -)
await web_search({
  query: "product reviews",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// More content extraction
await web_search({
  query: "detailed AI research",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### قواعد تصفية النطاقات

- الحد الأقصى 20 نطاقًا لكل عامل تصفية
- لا يمكن خلط قائمة السماح وقائمة الحظر في الطلب نفسه
- استخدم السابقة `-` لعناصر قائمة الحظر (مثل `["-reddit.com"]`)

## ملاحظات

- تعيد واجهة Perplexity Search API نتائج بحث ويب منظّمة (`title` و`url` و`snippet`)
- يؤدي OpenRouter أو الضبط الصريح لـ `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` إلى إعادة Perplexity إلى Sonar chat completions من أجل التوافق
- يعيد توافق Sonar/OpenRouter إجابة مركبة واحدة مع استشهادات، وليس صفوف نتائج منظّمة
- تُخزَّن النتائج مؤقتًا لمدة 15 دقيقة افتراضيًا (قابلة للتكوين عبر `cacheTtlMinutes`)

## ذو صلة

- [نظرة عامة على البحث على الويب](/tools/web) -- جميع الموفّرين والكشف التلقائي
- [مستندات Perplexity Search API](https://docs.perplexity.ai/docs/search/quickstart) -- وثائق Perplexity الرسمية
- [Brave Search](/tools/brave-search) -- نتائج منظّمة مع عوامل تصفية الدولة/اللغة
- [Exa Search](/tools/exa-search) -- بحث عصبي مع استخراج المحتوى
