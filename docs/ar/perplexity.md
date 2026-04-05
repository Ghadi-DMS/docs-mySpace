---
read_when:
    - تريد استخدام Perplexity Search للبحث على الويب
    - تحتاج إلى إعداد `PERPLEXITY_API_KEY` أو `OPENROUTER_API_KEY`
summary: Perplexity Search API وتوافق Sonar/OpenRouter مع `web_search`
title: Perplexity Search (المسار القديم)
x-i18n:
    generated_at: "2026-04-05T12:49:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: ba91e63e7412f3b6f889ee11f4a66563014932a1dc7be8593fe2262a4877b89b
    source_path: perplexity.md
    workflow: 15
---

# Perplexity Search API

يدعم OpenClaw ‏Perplexity Search API كمزوّد `web_search`.
ويعيد نتائج مهيكلة تحتوي على الحقول `title` و`url` و`snippet`.

ولأجل التوافق، يدعم OpenClaw أيضًا إعدادات Perplexity Sonar/OpenRouter القديمة.
إذا كنت تستخدم `OPENROUTER_API_KEY`، أو مفتاح `sk-or-...` في `plugins.entries.perplexity.config.webSearch.apiKey`، أو تضبط `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`، فإن المزوّد ينتقل إلى مسار chat-completions ويعيد إجابات مركّبة بالذكاء الاصطناعي مع الاستشهادات بدلًا من نتائج Search API المهيكلة.

## الحصول على مفتاح Perplexity API

1. أنشئ حساب Perplexity على [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)
2. أنشئ مفتاح API في لوحة التحكم
3. خزّن المفتاح في التكوين أو اضبط `PERPLEXITY_API_KEY` في بيئة Gateway.

## توافق OpenRouter

إذا كنت تستخدم OpenRouter مسبقًا مع Perplexity Sonar، فأبقِ `provider: "perplexity"` واضبط `OPENROUTER_API_KEY` في بيئة Gateway، أو خزّن مفتاح `sk-or-...` في `plugins.entries.perplexity.config.webSearch.apiKey`.

عناصر تحكم توافقية اختيارية:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## أمثلة التكوين

### Perplexity Search API الأصلية

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

### توافق OpenRouter / Sonar

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

**عبر التكوين:** شغّل `openclaw configure --section web`. وسيخزن المفتاح في
`~/.openclaw/openclaw.json` تحت `plugins.entries.perplexity.config.webSearch.apiKey`.
ويقبل هذا الحقل أيضًا كائنات SecretRef.

**عبر البيئة:** اضبط `PERPLEXITY_API_KEY` أو `OPENROUTER_API_KEY`
في بيئة عملية Gateway. وبالنسبة إلى تثبيت gateway، ضعها في
`~/.openclaw/.env` (أو في بيئة الخدمة الخاصة بك). راجع [Env vars](/help/faq#env-vars-and-env-loading).

إذا كان `provider: "perplexity"` مكوّنًا وكان SecretRef الخاصة بمفتاح Perplexity غير محلولة من دون رجوع إلى env، فإن بدء التشغيل/إعادة التحميل يفشل سريعًا.

## معاملات الأداة

تنطبق هذه المعاملات على مسار Perplexity Search API الأصلي.

| المعامل              | الوصف                                                  |
| -------------------- | ------------------------------------------------------ |
| `query`              | استعلام البحث (مطلوب)                                  |
| `count`              | عدد النتائج المطلوب إرجاعها (1-10، الافتراضي: 5)       |
| `country`            | رمز بلد ISO مكوّن من حرفين (مثل "US"، "DE")            |
| `language`           | رمز لغة ISO 639-1 (مثل "en"، "de"، "fr")               |
| `freshness`          | مرشح الزمن: `day` (24h) أو `week` أو `month` أو `year` |
| `date_after`         | النتائج المنشورة بعد هذا التاريخ فقط (YYYY-MM-DD)      |
| `date_before`        | النتائج المنشورة قبل هذا التاريخ فقط (YYYY-MM-DD)      |
| `domain_filter`      | مصفوفة قائمة سماح/رفض للنطاقات (الحد الأقصى 20)        |
| `max_tokens`         | ميزانية المحتوى الإجمالية (الافتراضي: 25000، الحد الأقصى: 1000000) |
| `max_tokens_per_page`| حد الرموز لكل صفحة (الافتراضي: 2048)                   |

أما بالنسبة إلى مسار توافق Sonar/OpenRouter القديم:

- تُقبل `query` و`count` و`freshness`
- تكون `count` للتوافق فقط هناك؛ وتظل الاستجابة إجابة مركّبة واحدة
  مع استشهادات بدلًا من قائمة نتائج بعدد N
- تعيد المرشحات الخاصة بـ Search API فقط مثل `country` و`language` و`date_after`,
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

### قواعد مرشح النطاق

- الحد الأقصى 20 نطاقًا لكل مرشح
- لا يمكن مزج قائمة السماح وقائمة الرفض في الطلب نفسه
- استخدم البادئة `-` لعناصر قائمة الرفض (مثل `["-reddit.com"]`)

## ملاحظات

- تعيد Perplexity Search API نتائج بحث ويب مهيكلة (`title`, `url`, `snippet`)
- يؤدي OpenRouter أو `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` الصريحان إلى إعادة Perplexity إلى Sonar chat completions لأجل التوافق
- يعيد توافق Sonar/OpenRouter إجابة مركّبة واحدة مع استشهادات، وليس صفوف نتائج مهيكلة
- يتم تخزين النتائج مؤقتًا لمدة 15 دقيقة افتراضيًا (قابلة للتكوين عبر `cacheTtlMinutes`)

راجع [Web tools](/tools/web) للاطلاع على تكوين `web_search` الكامل.
وراجع [وثائق Perplexity Search API](https://docs.perplexity.ai/docs/search/quickstart) لمزيد من التفاصيل.
