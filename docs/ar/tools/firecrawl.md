---
read_when:
    - تريد استخراج ويب مدعومًا بواسطة Firecrawl
    - تحتاج إلى مفتاح API لـ Firecrawl
    - تريد Firecrawl كموفّر `web_search`
    - تريد استخراجًا مقاومًا لآليات مكافحة الروبوتات من أجل `web_fetch`
summary: بحث Firecrawl، واستخراج المحتوى، وبديل `web_fetch`
title: Firecrawl
x-i18n:
    generated_at: "2026-04-05T12:58:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 45f17fc4b8e81e1bfe25f510b0a64ab0d50c4cc95bcf88d6ba7c62cece26162e
    source_path: tools/firecrawl.md
    workflow: 15
---

# Firecrawl

يمكن لـ OpenClaw استخدام **Firecrawl** بثلاث طرق:

- كموفّر `web_search`
- كأدوات plugin صريحة: `firecrawl_search` و `firecrawl_scrape`
- كأداة استخراج بديلة لـ `web_fetch`

إنها خدمة استضافة للاستخراج/البحث تدعم تجاوز آليات مكافحة الروبوتات والتخزين المؤقت،
مما يساعد مع المواقع المعتمدة بكثافة على JavaScript أو الصفحات التي تحظر طلبات HTTP العادية.

## الحصول على مفتاح API

1. أنشئ حساب Firecrawl وأنشئ مفتاح API.
2. خزّنه في التهيئة أو عيّن `FIRECRAWL_API_KEY` في بيئة gateway.

## تهيئة بحث Firecrawl

```json5
{
  tools: {
    web: {
      search: {
        provider: "firecrawl",
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webSearch: {
            apiKey: "FIRECRAWL_API_KEY_HERE",
            baseUrl: "https://api.firecrawl.dev",
          },
        },
      },
    },
  },
}
```

ملاحظات:

- يؤدي اختيار Firecrawl أثناء الإعداد الأولي أو عبر `openclaw configure --section web` إلى تمكين plugin Firecrawl المضمّن تلقائيًا.
- يدعم `web_search` مع Firecrawl كلًا من `query` و `count`.
- لاستخدام عناصر تحكم خاصة بـ Firecrawl مثل `sources` أو `categories` أو استخراج النتائج، استخدم `firecrawl_search`.
- يجب أن تبقى أي تجاوزات لـ `baseUrl` على `https://api.firecrawl.dev`.
- يُعد `FIRECRAWL_BASE_URL` خيار env الاحتياطي المشترك لعناوين Firecrawl الأساسية للبحث والاستخراج.

## تهيئة Firecrawl scrape + بديل web_fetch

```json5
{
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            apiKey: "FIRECRAWL_API_KEY_HERE",
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000,
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

ملاحظات:

- لا تُجرى محاولات البديل Firecrawl إلا عند توفر مفتاح API (`plugins.entries.firecrawl.config.webFetch.apiKey` أو `FIRECRAWL_API_KEY`).
- يتحكم `maxAgeMs` في عمر النتائج المخزنة مؤقتًا المسموح به (بالملي ثانية). الافتراضي هو يومان.
- تُرحَّل تلقائيًا تهيئة `tools.web.fetch.firecrawl.*` القديمة بواسطة `openclaw doctor --fix`.
- تقتصر تجاوزات Firecrawl scrape/base URL على `https://api.firecrawl.dev`.

يعيد `firecrawl_scrape` استخدام إعدادات ومتغيرات env نفسها الخاصة بـ `plugins.entries.firecrawl.config.webFetch.*`.

## أدوات plugin Firecrawl

### `firecrawl_search`

استخدم هذا عندما تريد عناصر تحكم في البحث خاصة بـ Firecrawl بدلًا من `web_search` العام.

المعلمات الأساسية:

- `query`
- `count`
- `sources`
- `categories`
- `scrapeResults`
- `timeoutSeconds`

### `firecrawl_scrape`

استخدم هذا للصفحات الثقيلة بـ JavaScript أو المحمية ضد الروبوتات عندما يكون `web_fetch` العادي ضعيفًا.

المعلمات الأساسية:

- `url`
- `extractMode`
- `maxChars`
- `onlyMainContent`
- `maxAgeMs`
- `proxy`
- `storeInCache`
- `timeoutSeconds`

## Stealth / تجاوز آليات مكافحة الروبوتات

يوفر Firecrawl معلمة **proxy mode** لتجاوز آليات مكافحة الروبوتات (`basic` أو `stealth` أو `auto`).
يستخدم OpenClaw دائمًا `proxy: "auto"` بالإضافة إلى `storeInCache: true` لطلبات Firecrawl.
إذا لم يتم تحديد proxy، فسيستخدم Firecrawl الوضع الافتراضي `auto`. يقوم `auto` بإعادة المحاولة باستخدام وسطاء stealth إذا فشلت محاولة أساسية، وقد يستهلك ذلك أرصدة أكثر
من الاستخراج الأساسي فقط.

## كيف يستخدم `web_fetch` Firecrawl

ترتيب استخراج `web_fetch`:

1. Readability (محلي)
2. Firecrawl (إذا تم اختياره أو اكتشافه تلقائيًا باعتباره بديل web-fetch النشط)
3. تنظيف HTML أساسي (البديل الأخير)

مفتاح الاختيار هو `tools.web.fetch.provider`. إذا لم تحدده، فسيقوم OpenClaw
باكتشاف أول موفّر web-fetch جاهز تلقائيًا من بيانات الاعتماد المتاحة.
اليوم، الموفّر المضمّن هو Firecrawl.

## ذو صلة

- [نظرة عامة على Web Search](/tools/web) -- جميع الموفّرين والكشف التلقائي
- [Web Fetch](/tools/web-fetch) -- أداة `web_fetch` مع بديل Firecrawl
- [Tavily](/tools/tavily) -- أدوات البحث + الاستخراج
