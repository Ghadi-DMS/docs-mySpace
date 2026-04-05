---
read_when:
    - أنت تريد مزوّد بحث ويب ذاتي الاستضافة
    - أنت تريد استخدام SearXNG من أجل `web_search`
    - أنت تحتاج إلى خيار بحث يركّز على الخصوصية أو يعمل في بيئة معزولة شبكيًا
summary: بحث ويب SearXNG -- مزوّد بحث وصفي ذاتي الاستضافة ولا يتطلب مفتاحًا
title: بحث SearXNG
x-i18n:
    generated_at: "2026-04-05T12:59:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0a8fc7f890b7595d17c5ef8aede9b84bb2459f30a53d5d87c4e7423e1ac83ca5
    source_path: tools/searxng-search.md
    workflow: 15
---

# بحث SearXNG

يدعم OpenClaw [SearXNG](https://docs.searxng.org/) باعتباره مزوّد `web_search` **ذاتي الاستضافة ولا يتطلب مفتاحًا**. SearXNG هو محرك بحث وصفي مفتوح المصدر
يجمع النتائج من Google وBing وDuckDuckGo ومصادر أخرى.

المزايا:

- **مجاني وغير محدود** -- لا حاجة إلى مفتاح API أو اشتراك تجاري
- **الخصوصية / العزل الشبكي** -- لا تغادر الاستعلامات شبكتك
- **يعمل في أي مكان** -- لا توجد قيود إقليمية على واجهات API التجارية للبحث

## الإعداد

<Steps>
  <Step title="شغّل مثيل SearXNG">
    ```bash
    docker run -d -p 8888:8080 searxng/searxng
    ```

    أو استخدم أي نشر SearXNG حالي لديك إمكانية الوصول إليه. راجع
    [وثائق SearXNG](https://docs.searxng.org/) لإعداد الإنتاج.

  </Step>
  <Step title="الإعداد">
    ```bash
    openclaw configure --section web
    # Select "searxng" as the provider
    ```

    أو اضبط متغير البيئة ودع الاكتشاف التلقائي يعثر عليه:

    ```bash
    export SEARXNG_BASE_URL="http://localhost:8888"
    ```

  </Step>
</Steps>

## الإعدادات

```json5
{
  tools: {
    web: {
      search: {
        provider: "searxng",
      },
    },
  },
}
```

إعدادات على مستوى الإضافة لمثيل SearXNG:

```json5
{
  plugins: {
    entries: {
      searxng: {
        config: {
          webSearch: {
            baseUrl: "http://localhost:8888",
            categories: "general,news", // optional
            language: "en", // optional
          },
        },
      },
    },
  },
}
```

يقبل الحقل `baseUrl` أيضًا كائنات SecretRef.

قواعد النقل:

- يعمل `https://` مع مضيفات SearXNG العامة أو الخاصة
- لا يُقبل `http://` إلا لمضيفات الشبكات الخاصة الموثوقة أو loopback
- يجب أن تستخدم مضيفات SearXNG العامة `https://`

## متغير البيئة

اضبط `SEARXNG_BASE_URL` كبديل عن الإعدادات:

```bash
export SEARXNG_BASE_URL="http://localhost:8888"
```

عند ضبط `SEARXNG_BASE_URL` وعدم تهيئة مزوّد صريح، يختار الاكتشاف التلقائي
SearXNG تلقائيًا (بأدنى أولوية -- أي مزوّد مدعوم بواجهة API ومهيأ
بمفتاح يفوز أولًا).

## مرجع إعدادات الإضافة

| الحقل        | الوصف                                                          |
| ------------ | -------------------------------------------------------------- |
| `baseUrl`    | عنوان URL الأساسي لمثيل SearXNG الخاص بك (مطلوب)              |
| `categories` | فئات مفصولة بفواصل مثل `general` أو `news` أو `science`       |
| `language`   | رمز اللغة للنتائج مثل `en` أو `de` أو `fr`                    |

## ملاحظات

- **JSON API** -- يستخدم نقطة النهاية الأصلية `format=json` الخاصة بـ SearXNG، وليس كشط HTML
- **لا حاجة إلى مفتاح API** -- يعمل مع أي مثيل SearXNG مباشرةً
- **التحقق من عنوان URL الأساسي** -- يجب أن يكون `baseUrl` عنوان URL صالحًا من نوع `http://` أو `https://`؛ ويجب أن تستخدم المضيفات العامة `https://`
- **ترتيب الاكتشاف التلقائي** -- يُفحص SearXNG أخيرًا (الترتيب 200) في
  الاكتشاف التلقائي. تعمل أولًا المزوّدات المدعومة بواجهة API والمهيأة بمفاتيح، ثم
  DuckDuckGo (الترتيب 100)، ثم Ollama Web Search (الترتيب 110)
- **ذاتي الاستضافة** -- أنت تتحكم في المثيل والاستعلامات ومحركات البحث العلوية
- تكون `categories` افتراضيًا `general` عندما لا تكون مهيأة

<Tip>
  لكي تعمل JSON API الخاصة بـ SearXNG، تأكد من أن مثيل SearXNG لديك مفعّل فيه تنسيق `json`
  في ملف `settings.yml` ضمن `search.formats`.
</Tip>

## ذو صلة

- [نظرة عامة على Web Search](/tools/web) -- جميع المزوّدين والاكتشاف التلقائي
- [بحث DuckDuckGo](/tools/duckduckgo-search) -- خيار رجوع آخر لا يتطلب مفتاحًا
- [بحث Brave](/tools/brave-search) -- نتائج منظَّمة مع فئة مجانية
