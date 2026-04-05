---
read_when:
    - تريد جلب عنوان URL واستخراج محتوى قابل للقراءة منه
    - تحتاج إلى إعداد `web_fetch` أو إعداد الرجوع الاحتياطي الخاص به عبر Firecrawl
    - تريد فهم حدود `web_fetch` وآلية التخزين المؤقت فيه
sidebarTitle: Web Fetch
summary: أداة `web_fetch` -- جلب HTTP مع استخراج محتوى قابل للقراءة
title: Web Fetch
x-i18n:
    generated_at: "2026-04-05T13:00:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60c933a25d0f4511dc1683985988e115b836244c5eac4c6667b67c8eb15401e0
    source_path: tools/web-fetch.md
    workflow: 15
---

# Web Fetch

تقوم أداة `web_fetch` بتنفيذ طلب HTTP GET عادي واستخراج محتوى قابل للقراءة
(من HTML إلى markdown أو نص). وهي **لا** تنفّذ JavaScript.

بالنسبة إلى المواقع المعتمدة بكثافة على JS أو الصفحات المحمية بتسجيل الدخول، استخدم
[Web Browser](/tools/browser) بدلًا من ذلك.

## بداية سريعة

`web_fetch` **مفعّلة افتراضيًا** -- ولا تحتاج إلى أي إعداد. يمكن للوكيل
استدعاؤها فورًا:

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## معلمات الأداة

| المعلمة | النوع | الوصف |
| ------------- | -------- | ---------------------------------------- |
| `url` | `string` | عنوان URL المطلوب جلبه (مطلوب، http/https فقط) |
| `extractMode` | `string` | ‏`"markdown"` ‏(الافتراضي) أو `"text"` |
| `maxChars` | `number` | اقتطاع المخرجات إلى هذا العدد من الأحرف |

## كيف تعمل

<Steps>
  <Step title="الجلب">
    ترسل طلب HTTP GET باستخدام User-Agent شبيه بـ Chrome وترويسة
    `Accept-Language`. وتحظر أسماء المضيفين الخاصة/الداخلية، وتعيد التحقق من عمليات إعادة التوجيه.
  </Step>
  <Step title="الاستخراج">
    تشغّل Readability ‏(استخراج المحتوى الرئيسي) على استجابة HTML.
  </Step>
  <Step title="الرجوع الاحتياطي (اختياري)">
    إذا فشل Readability وكان Firecrawl مُعدًا، تعيد المحاولة عبر
    API الخاص بـ Firecrawl مع وضع تجاوز حماية الروبوتات.
  </Step>
  <Step title="التخزين المؤقت">
    تُخزَّن النتائج مؤقتًا لمدة 15 دقيقة (قابلة للإعداد) لتقليل
    عمليات الجلب المتكررة لعنوان URL نفسه.
  </Step>
</Steps>

## الإعدادات

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // default: true
        provider: "firecrawl", // optional; omit for auto-detect
        maxChars: 50000, // max output chars
        maxCharsCap: 50000, // hard cap for maxChars param
        maxResponseBytes: 2000000, // max download size before truncation
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true, // use Readability extraction
        userAgent: "Mozilla/5.0 ...", // override User-Agent
      },
    },
  },
}
```

## الرجوع الاحتياطي عبر Firecrawl

إذا فشل استخراج Readability، يمكن لـ `web_fetch` الرجوع احتياطيًا إلى
[Firecrawl](/tools/firecrawl) لتجاوز حماية الروبوتات وتحسين الاستخراج:

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // optional; omit for auto-detect from available credentials
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            apiKey: "fc-...", // optional if FIRECRAWL_API_KEY is set
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 86400000, // cache duration (1 day)
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

يدعم `plugins.entries.firecrawl.config.webFetch.apiKey` كائنات SecretRef.
ويتم ترحيل إعدادات `tools.web.fetch.firecrawl.*` القديمة تلقائيًا بواسطة `openclaw doctor --fix`.

<Note>
  إذا كان Firecrawl مفعّلًا وكان SecretRef الخاص به غير محلول بدون
  رجوع احتياطي عبر متغير البيئة `FIRECRAWL_API_KEY`،
  فإن بدء تشغيل gateway يفشل بسرعة.
</Note>

<Note>
  يتم تقييد تجاوزات Firecrawl ‏`baseUrl`: إذ يجب أن تستخدم `https://` و
  المضيف الرسمي لـ Firecrawl ‏(`api.firecrawl.dev`).
</Note>

سلوك وقت التشغيل الحالي:

- يختار `tools.web.fetch.provider` مزوّد الرجوع الاحتياطي للجلب بشكل صريح.
- إذا تم حذف `provider`، يكتشف OpenClaw تلقائيًا أول مزوّد
  جاهز لجلب الويب من بيانات الاعتماد المتاحة. والمزوّد المضمّن اليوم هو Firecrawl.
- إذا تم تعطيل Readability، فإن `web_fetch` يتخطى مباشرةً إلى
  مزوّد الرجوع الاحتياطي المحدد. وإذا لم يكن أي مزوّد متاحًا، فإنه يفشل بشكل مغلق.

## الحدود والأمان

- يتم تقييد `maxChars` إلى `tools.web.fetch.maxCharsCap`
- يتم وضع حد أقصى لجسم الاستجابة عند `maxResponseBytes` قبل التحليل؛ وتُقتطع
  الاستجابات كبيرة الحجم مع تحذير
- يتم حظر أسماء المضيفين الخاصة/الداخلية
- يتم التحقق من عمليات إعادة التوجيه وتقييدها بواسطة `maxRedirects`
- تعمل `web_fetch` على أساس أفضل جهد -- إذ تحتاج بعض المواقع إلى [Web Browser](/tools/browser)

## ملفات تعريف الأدوات

إذا كنت تستخدم ملفات تعريف الأدوات أو قوائم السماح، فأضف `web_fetch` أو `group:web`:

```json5
{
  tools: {
    allow: ["web_fetch"],
    // or: allow: ["group:web"]  (includes web_fetch, web_search, and x_search)
  },
}
```

## ذو صلة

- [Web Search](/tools/web) -- ابحث في الويب باستخدام عدة مزوّدين
- [Web Browser](/tools/browser) -- أتمتة متصفح كاملة للمواقع المعتمدة بكثافة على JS
- [Firecrawl](/tools/firecrawl) -- أدوات البحث والكشط الخاصة بـ Firecrawl
