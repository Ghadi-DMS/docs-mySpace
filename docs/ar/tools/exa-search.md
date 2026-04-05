---
read_when:
    - تريد استخدام Exa من أجل `web_search`
    - تحتاج إلى `EXA_API_KEY`
    - تريد البحث العصبي أو استخراج المحتوى
summary: بحث Exa AI -- بحث عصبي وبحث بالكلمات المفتاحية مع استخراج المحتوى
title: بحث Exa
x-i18n:
    generated_at: "2026-04-05T12:58:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 307b727b4fb88756cac51c17ffd73468ca695c4481692e03d0b4a9969982a2a8
    source_path: tools/exa-search.md
    workflow: 15
---

# بحث Exa

يدعم OpenClaw [Exa AI](https://exa.ai/) بصفته موفّر `web_search`. يوفّر Exa
أوضاع بحث عصبي، وبالكلمات المفتاحية، وهجين، مع استخراج محتوى مدمج
(إبرازات، ونص، وملخصات).

## الحصول على مفتاح API

<Steps>
  <Step title="أنشئ حسابًا">
    سجّل في [exa.ai](https://exa.ai/) وأنشئ مفتاح API من
    لوحة التحكم الخاصة بك.
  </Step>
  <Step title="خزّن المفتاح">
    عيّن `EXA_API_KEY` في بيئة Gateway، أو قم بالتهيئة عبر:

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
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // اختياري إذا كان EXA_API_KEY معيّنًا
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**بديل البيئة:** عيّن `EXA_API_KEY` في بيئة Gateway.
بالنسبة إلى تثبيت Gateway، ضعه في `~/.openclaw/.env`.

## معلمات الأداة

| المعلمة | الوصف |
| ------------- | ----------------------------------------------------------------------------- |
| `query` | استعلام البحث (مطلوب) |
| `count` | النتائج المراد إرجاعها (1-100) |
| `type` | وضع البحث: `auto` أو `neural` أو `fast` أو `deep` أو `deep-reasoning` أو `instant` |
| `freshness` | عامل تصفية الوقت: `day` أو `week` أو `month` أو `year` |
| `date_after` | النتائج بعد هذا التاريخ (YYYY-MM-DD) |
| `date_before` | النتائج قبل هذا التاريخ (YYYY-MM-DD) |
| `contents` | خيارات استخراج المحتوى (انظر أدناه) |

### استخراج المحتوى

يمكن لـ Exa إرجاع محتوى مستخرج إلى جانب نتائج البحث. مرّر كائن `contents`
للتمكين:

```javascript
await web_search({
  query: "transformer architecture explained",
  type: "neural",
  contents: {
    text: true, // النص الكامل للصفحة
    highlights: { numSentences: 3 }, // الجمل الأساسية
    summary: true, // ملخص من AI
  },
});
```

| خيار contents | النوع | الوصف |
| --------------- | --------------------------------------------------------------------- | ---------------------- |
| `text` | `boolean \| { maxCharacters }` | استخراج النص الكامل للصفحة |
| `highlights` | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | استخراج الجمل الأساسية |
| `summary` | `boolean \| { query }` | ملخص منشأ بواسطة AI |

### أوضاع البحث

| الوضع | الوصف |
| ---------------- | --------------------------------- |
| `auto` | يختار Exa أفضل وضع (الافتراضي) |
| `neural` | بحث دلالي/قائم على المعنى |
| `fast` | بحث سريع بالكلمات المفتاحية |
| `deep` | بحث عميق وشامل |
| `deep-reasoning` | بحث عميق مع استدلال |
| `instant` | أسرع النتائج |

## ملاحظات

- إذا لم يتم توفير خيار `contents`، يستخدم Exa افتراضيًا `{ highlights: true }`
  بحيث تتضمن النتائج مقتطفات من الجمل الأساسية
- تحتفظ النتائج بحقلَي `highlightScores` و`summary` من استجابة API الخاصة بـ Exa
  عند توفرهما
- يتم تحليل أوصاف النتائج من الإبرازات أولًا، ثم الملخص، ثم
  النص الكامل — أيّها كان متاحًا
- لا يمكن الجمع بين `freshness` و`date_after`/`date_before` — استخدم
  وضعًا واحدًا لتصفية الوقت
- يمكن إرجاع ما يصل إلى 100 نتيجة لكل استعلام (وفقًا لقيود
  نوع البحث في Exa)
- تُخزَّن النتائج مؤقتًا لمدة 15 دقيقة افتراضيًا (قابلة للتهيئة عبر
  `cacheTtlMinutes`)
- Exa تكامل رسمي مع API مع استجابات JSON منظَّمة

## ذو صلة

- [نظرة عامة على Web Search](/tools/web) -- جميع الموفّرين والكشف التلقائي
- [بحث Brave](/tools/brave-search) -- نتائج منظَّمة مع عوامل تصفية البلد/اللغة
- [بحث Perplexity](/tools/perplexity-search) -- نتائج منظَّمة مع تصفية النطاقات
