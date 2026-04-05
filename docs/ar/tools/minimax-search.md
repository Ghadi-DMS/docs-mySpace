---
read_when:
    - تريد استخدام MiniMax من أجل `web_search`
    - تحتاج إلى مفتاح MiniMax Coding Plan
    - تريد إرشادات حول مضيف البحث MiniMax CN/global
summary: بحث MiniMax عبر Coding Plan search API
title: بحث MiniMax
x-i18n:
    generated_at: "2026-04-05T12:59:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: b8c3767790f428fc7e239590a97e9dbee0d3bd6550ca3299ae22da0f5a57231a
    source_path: tools/minimax-search.md
    workflow: 15
---

# بحث MiniMax

يدعم OpenClaw MiniMax بصفته موفّر `web_search` عبر MiniMax
Coding Plan search API. وهو يعيد نتائج بحث منظَّمة تتضمن العناوين، وعناوين URL،
والمقتطفات، وعمليات البحث ذات الصلة.

## الحصول على مفتاح Coding Plan

<Steps>
  <Step title="أنشئ مفتاحًا">
    أنشئ أو انسخ مفتاح MiniMax Coding Plan من
    [MiniMax Platform](https://platform.minimax.io/user-center/basic-information/interface-key).
  </Step>
  <Step title="خزّن المفتاح">
    عيّن `MINIMAX_CODE_PLAN_KEY` في بيئة Gateway، أو قم بالتهيئة عبر:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

يقبل OpenClaw أيضًا `MINIMAX_CODING_API_KEY` كاسم env بديل. ولا يزال `MINIMAX_API_KEY`
يُقرأ كخيار احتياطي للتوافق عندما يكون يشير بالفعل إلى رمز coding-plan.

## التهيئة

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...", // اختياري إذا كان MINIMAX_CODE_PLAN_KEY معيّنًا
            region: "global", // أو "cn"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "minimax",
      },
    },
  },
}
```

**بديل البيئة:** عيّن `MINIMAX_CODE_PLAN_KEY` في بيئة Gateway.
بالنسبة إلى تثبيت Gateway، ضعه في `~/.openclaw/.env`.

## اختيار المنطقة

يستخدم MiniMax Search نقاط النهاية التالية:

- عالمي: `https://api.minimax.io/v1/coding_plan/search`
- الصين: `https://api.minimaxi.com/v1/coding_plan/search`

إذا لم يتم تعيين `plugins.entries.minimax.config.webSearch.region`، فسيقوم OpenClaw
بتحديد المنطقة بهذا الترتيب:

1. `tools.web.search.minimax.region` / `webSearch.region` المملوك لـ plugin
2. `MINIMAX_API_HOST`
3. `models.providers.minimax.baseUrl`
4. `models.providers.minimax-portal.baseUrl`

وهذا يعني أن الإعداد الأولي لـ CN أو `MINIMAX_API_HOST=https://api.minimaxi.com/...`
يحافظان تلقائيًا على MiniMax Search على مضيف CN أيضًا.

حتى عندما تكون قد أجريت مصادقة MiniMax عبر مسار OAuth `minimax-portal`،
فإن البحث على الويب لا يزال يُسجَّل بمعرّف المزوّد `minimax`؛ ويُستخدم عنوان URL الأساسي لمزوّد OAuth
فقط كتلميح للمنطقة من أجل اختيار مضيف CN/global.

## المعلمات المدعومة

يدعم MiniMax Search ما يلي:

- `query`
- `count` (يقوم OpenClaw بقص قائمة النتائج المُعادة إلى العدد المطلوب)

لا تتوفر حاليًا عوامل تصفية خاصة بالمزوّد.

## ذو صلة

- [نظرة عامة على Web Search](/tools/web) -- جميع الموفّرين والكشف التلقائي
- [MiniMax](/ar/providers/minimax) -- إعداد النموذج، والصور، والصوت، والمصادقة
