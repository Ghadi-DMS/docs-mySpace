---
read_when:
    - تريد تهيئة Perplexity كمزوّد للبحث على الويب
    - تحتاج إلى Perplexity API key أو إعداد OpenRouter proxy
summary: إعداد مزود Perplexity للبحث على الويب (API key، وأوضاع البحث، والتصفية)
title: Perplexity ‏(المزوّد)
x-i18n:
    generated_at: "2026-04-05T12:53:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: df9082d15d6a36a096e21efe8cee78e4b8643252225520f5b96a0b99cf5a7a4b
    source_path: providers/perplexity-provider.md
    workflow: 15
---

# Perplexity ‏(مزود البحث على الويب)

يوفر المكوّن الإضافي Perplexity إمكانات البحث على الويب عبر Perplexity
Search API أو Perplexity Sonar عبر OpenRouter.

<Note>
تغطي هذه الصفحة إعداد **المزوّد** الخاص بـ Perplexity. أما **الأداة**
الخاصة بـ Perplexity (كيفية استخدام الوكيل لها)، فراجع [Perplexity tool](/tools/perplexity-search).
</Note>

- النوع: مزوّد بحث على الويب (وليس مزوّد نماذج)
- المصادقة: `PERPLEXITY_API_KEY` ‏(مباشر) أو `OPENROUTER_API_KEY` ‏(عبر OpenRouter)
- مسار الإعداد: `plugins.entries.perplexity.config.webSearch.apiKey`

## بدء سريع

1. اضبط API key:

```bash
openclaw configure --section web
```

أو اضبطها مباشرة:

```bash
openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
```

2. سيستخدم الوكيل Perplexity تلقائيًا في عمليات البحث على الويب عند إعدادها.

## أوضاع البحث

يختار المكوّن الإضافي وسيلة النقل تلقائيًا بناءً على بادئة API key:

| بادئة المفتاح | وسيلة النقل                     | الميزات                                           |
| ------------- | ------------------------------- | ------------------------------------------------- |
| `pplx-`       | Perplexity Search API الأصلية   | نتائج منظَّمة، ومرشحات النطاق/اللغة/التاريخ       |
| `sk-or-`      | OpenRouter ‏(Sonar)             | إجابات مولَّدة بالذكاء الاصطناعي مع استشهادات      |

## التصفية في API الأصلية

عند استخدام Perplexity API الأصلية ‏(مفتاح `pplx-`)، تدعم عمليات البحث:

- **Country**: رمز بلد مكوّن من حرفين
- **Language**: رمز لغة ISO 639-1
- **Date range**: يوم، أسبوع، شهر، سنة
- **Domain filters**: قائمة سماح/قائمة حظر (بحد أقصى 20 نطاقًا)
- **Content budget**: ‏`max_tokens`, `max_tokens_per_page`

## ملاحظة حول البيئة

إذا كانت Gateway تعمل كـ daemon ‏(launchd/systemd)، فتأكد من أن
`PERPLEXITY_API_KEY` متاحة لتلك العملية (على سبيل المثال، في
`~/.openclaw/.env` أو عبر `env.shellEnv`).
