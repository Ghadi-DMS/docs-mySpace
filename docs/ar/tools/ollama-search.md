---
read_when:
    - تريد استخدام Ollama مع `web_search`
    - تريد موفر `web_search` لا يتطلب مفتاحًا
    - تحتاج إلى إرشادات إعداد بحث الويب عبر Ollama
summary: بحث الويب عبر Ollama باستخدام مضيف Ollama المُهيأ لديك
title: بحث الويب عبر Ollama
x-i18n:
    generated_at: "2026-04-05T12:59:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3c1d0765594e0eb368c25cca21a712c054e71cf43e7bfb385d10feddd990f4fd
    source_path: tools/ollama-search.md
    workflow: 15
---

# بحث الويب عبر Ollama

يدعم OpenClaw **بحث الويب عبر Ollama** بوصفه موفر `web_search` مضمّنًا.
ويستخدم API التجريبي لبحث الويب في Ollama ويعيد نتائج منظَّمة
تتضمن العناوين وعناوين URL والمقتطفات.

وعلى خلاف موفر نماذج Ollama، لا يتطلب هذا الإعداد مفتاح API
افتراضيًا. لكنه يتطلب:

- مضيف Ollama يمكن لـ OpenClaw الوصول إليه
- `ollama signin`

## الإعداد

<Steps>
  <Step title="ابدأ تشغيل Ollama">
    تأكد من أن Ollama مثبّت ويعمل.
  </Step>
  <Step title="سجّل الدخول">
    شغّل:

    ```bash
    ollama signin
    ```

  </Step>
  <Step title="اختر بحث الويب عبر Ollama">
    شغّل:

    ```bash
    openclaw configure --section web
    ```

    ثم اختر **بحث الويب عبر Ollama** بوصفه الموفّر.

  </Step>
</Steps>

إذا كنت تستخدم Ollama بالفعل للنماذج، فإن بحث الويب عبر Ollama يعيد استخدام
المضيف المُهيأ نفسه.

## التكوين

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

تجاوز اختياري لمضيف Ollama:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
      },
    },
  },
}
```

إذا لم يُضبط عنوان URL أساسي صريح لـ Ollama، يستخدم OpenClaw القيمة `http://127.0.0.1:11434`.

إذا كان مضيف Ollama لديك يتوقع bearer auth، فإن OpenClaw يعيد استخدام
`models.providers.ollama.apiKey` (أو مصادقة الموفّر المطابقة المعتمدة على env)
لطلبات بحث الويب أيضًا.

## ملاحظات

- لا يتطلب هذا الموفّر حقل مفتاح API خاصًا ببحث الويب.
- إذا كان مضيف Ollama محميًا بالمصادقة، فإن OpenClaw يعيد استخدام مفتاح API
  العادي لموفر Ollama عند توفره.
- يحذّر OpenClaw أثناء الإعداد إذا كان Ollama غير قابل للوصول أو لم يُسجَّل الدخول إليه، لكنه
  لا يمنع الاختيار.
- يمكن للكشف التلقائي أثناء runtime الرجوع إلى بحث الويب عبر Ollama عندما لا يكون هناك
  موفر أعلى أولوية ومُهيأ ببيانات اعتماد.
- يستخدم الموفّر نقطة النهاية التجريبية `/api/experimental/web_search`
  في Ollama.

## ذو صلة

- [نظرة عامة على البحث في الويب](/tools/web) -- جميع الموفّرين والكشف التلقائي
- [Ollama](/ar/providers/ollama) -- إعداد نماذج Ollama وأوضاع السحابة/الوضع المحلي
