---
read_when:
    - تريد موفر بحث ويب لا يتطلب مفتاح API
    - تريد استخدام DuckDuckGo مع `web_search`
    - تحتاج إلى بديل بحث احتياطي بلا إعدادات
summary: بحث الويب DuckDuckGo -- موفر احتياطي لا يتطلب مفتاحًا (تجريبي، قائم على HTML)
title: بحث DuckDuckGo
x-i18n:
    generated_at: "2026-04-05T12:57:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 31f8e3883584534396c247c3d8069ea4c5b6399e0ff13a9dd0c8ee0c3da02096
    source_path: tools/duckduckgo-search.md
    workflow: 15
---

# بحث DuckDuckGo

يدعم OpenClaw استخدام DuckDuckGo كموفر `web_search` **لا يتطلب مفتاحًا**. لا حاجة إلى
مفتاح API أو حساب.

<Warning>
  DuckDuckGo تكامل **تجريبي وغير رسمي** يستخرج النتائج
  من صفحات البحث غير المعتمدة على JavaScript في DuckDuckGo — وليس من API رسمي. توقّع
  حدوث أعطال أحيانًا بسبب صفحات تحديات البوتات أو تغييرات HTML.
</Warning>

## الإعداد

لا حاجة إلى مفتاح API — فقط اضبط DuckDuckGo كموفر لك:

<Steps>
  <Step title="التهيئة">
    ```bash
    openclaw configure --section web
    # Select "duckduckgo" as the provider
    ```
  </Step>
</Steps>

## الإعدادات

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

إعدادات اختيارية على مستوى plugin للمنطقة وSafeSearch:

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // DuckDuckGo region code
            safeSearch: "moderate", // "strict", "moderate", or "off"
          },
        },
      },
    },
  },
}
```

## معاملات الأداة

| المعامل | الوصف |
| ------------ | ---------------------------------------------------------- |
| `query`      | استعلام البحث (مطلوب)                                    |
| `count`      | النتائج المراد إرجاعها (1-10، الافتراضي: 5)                       |
| `region`     | رمز منطقة DuckDuckGo (مثل `us-en` أو `uk-en` أو `de-de`)    |
| `safeSearch` | مستوى SafeSearch: ‏`strict` أو `moderate` (الافتراضي) أو `off` |

يمكن أيضًا ضبط المنطقة وSafeSearch في إعدادات plugin (انظر أعلاه) — معاملات
الأداة تتجاوز قيم الإعدادات لكل استعلام.

## ملاحظات

- **لا يوجد مفتاح API** — يعمل مباشرة، من دون أي إعداد
- **تجريبي** — يجمع النتائج من صفحات بحث HTML غير المعتمدة على JavaScript
  الخاصة بـ DuckDuckGo، وليس من API أو SDK رسمي
- **خطر تحديات البوتات** — قد يعرض DuckDuckGo اختبارات CAPTCHA أو يحظر الطلبات
  عند الاستخدام الكثيف أو المؤتمت
- **تحليل HTML** — تعتمد النتائج على بنية الصفحة، والتي قد تتغير من دون
  إشعار
- **ترتيب الاكتشاف التلقائي** — DuckDuckGo هو أول بديل احتياطي
  لا يتطلب مفتاحًا (الترتيب 100) في الاكتشاف التلقائي. موفرو الخدمة المعتمدون على API مع مفاتيح
  مُعدّة يعملون أولًا، ثم Ollama Web Search (الترتيب 110)، ثم SearXNG (الترتيب 200)
- **تكون قيمة SafeSearch الافتراضية moderate** عندما لا يكون مضبوطًا

<Tip>
  للاستخدام الإنتاجي، فكّر في [Brave Search](/tools/brave-search) (تتوفر
  فئة مجانية) أو موفر آخر يعتمد على API.
</Tip>

## ذو صلة

- [نظرة عامة على Web Search](/tools/web) -- جميع موفري الخدمة والاكتشاف التلقائي
- [Brave Search](/tools/brave-search) -- نتائج منظمة مع فئة مجانية
- [Exa Search](/tools/exa-search) -- بحث عصبي مع استخراج للمحتوى
