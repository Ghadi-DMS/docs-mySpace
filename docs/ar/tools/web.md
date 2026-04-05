---
read_when:
    - تريد تمكين `web_search` أو تكوينه
    - تريد تمكين `x_search` أو تكوينه
    - تحتاج إلى اختيار موفر بحث
    - تريد فهم الاكتشاف التلقائي والرجوع إلى موفر احتياطي
sidebarTitle: Web Search
summary: '`web_search` و`x_search` و`web_fetch` -- ابحث في الويب، أو ابحث في منشورات X، أو اجلب محتوى صفحة'
title: بحث الويب
x-i18n:
    generated_at: "2026-04-05T13:00:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: b8b9a5d641dcdcbe7c099c8862898f12646f43151b6c4152d69c26af9b17e0fa
    source_path: tools/web.md
    workflow: 15
---

# بحث الويب

تبحث أداة `web_search` في الويب باستخدام الموفر الذي أعددته
وتعيد النتائج. تُخزَّن النتائج مؤقتًا حسب الاستعلام لمدة 15 دقيقة (قابلة للتهيئة).

يتضمن OpenClaw أيضًا `x_search` لمنشورات X (المعروف سابقًا باسم Twitter) و
`web_fetch` لجلب عناوين URL بشكل خفيف. في هذه المرحلة، يبقى `web_fetch`
محليًا بينما يمكن لـ `web_search` و`x_search` استخدام xAI Responses داخليًا.

<Info>
  `web_search` أداة HTTP خفيفة، وليست أتمتة متصفح. بالنسبة إلى
  المواقع الثقيلة المعتمدة على JS أو التي تتطلب تسجيل دخول، استخدم [Web Browser](/tools/browser). أما
  لجلب عنوان URL محدد، فاستخدم [Web Fetch](/tools/web-fetch).
</Info>

## بداية سريعة

<Steps>
  <Step title="اختر موفرًا">
    اختر موفرًا وأكمل أي إعداد مطلوب. بعض الموفرين
    لا يحتاجون إلى مفتاح، بينما يستخدم آخرون مفاتيح API. راجع صفحات الموفرين أدناه
    للحصول على التفاصيل.
  </Step>
  <Step title="التهيئة">
    ```bash
    openclaw configure --section web
    ```
    يخزن هذا الموفر وأي بيانات اعتماد لازمة. ويمكنك أيضًا ضبط متغير بيئة
    (مثل `BRAVE_API_KEY`) وتخطي هذه الخطوة للموفرين
    المعتمدين على API.
  </Step>
  <Step title="استخدمه">
    يمكن للوكيل الآن استدعاء `web_search`:

    ```javascript
    await web_search({ query: "OpenClaw plugin SDK" });
    ```

    بالنسبة إلى منشورات X، استخدم:

    ```javascript
    await x_search({ query: "dinner recipes" });
    ```

  </Step>
</Steps>

## اختيار موفر

<CardGroup cols={2}>
  <Card title="Brave Search" icon="shield" href="/tools/brave-search">
    نتائج منظمة مع مقتطفات. تدعم وضع `llm-context` ومرشحات البلد/اللغة. تتوفر فئة مجانية.
  </Card>
  <Card title="DuckDuckGo" icon="bird" href="/tools/duckduckgo-search">
    بديل احتياطي لا يحتاج إلى مفتاح. لا حاجة إلى مفتاح API. تكامل غير رسمي قائم على HTML.
  </Card>
  <Card title="Exa" icon="brain" href="/tools/exa-search">
    بحث عصبي + بالكلمات المفتاحية مع استخراج المحتوى (إبرازات، نص، ملخصات).
  </Card>
  <Card title="Firecrawl" icon="flame" href="/tools/firecrawl">
    نتائج منظمة. يفضّل اقترانه مع `firecrawl_search` و`firecrawl_scrape` للاستخراج العميق.
  </Card>
  <Card title="Gemini" icon="sparkles" href="/tools/gemini-search">
    إجابات مركبة بالذكاء الاصطناعي مع استشهادات عبر إسناد Google Search.
  </Card>
  <Card title="Grok" icon="zap" href="/tools/grok-search">
    إجابات مركبة بالذكاء الاصطناعي مع استشهادات عبر الإسناد الويب من xAI.
  </Card>
  <Card title="Kimi" icon="moon" href="/tools/kimi-search">
    إجابات مركبة بالذكاء الاصطناعي مع استشهادات عبر بحث الويب من Moonshot.
  </Card>
  <Card title="MiniMax Search" icon="globe" href="/tools/minimax-search">
    نتائج منظمة عبر MiniMax Coding Plan search API.
  </Card>
  <Card title="Ollama Web Search" icon="globe" href="/tools/ollama-search">
    بحث لا يحتاج إلى مفتاح عبر مضيف Ollama المهيأ لديك. يتطلب `ollama signin`.
  </Card>
  <Card title="Perplexity" icon="search" href="/tools/perplexity-search">
    نتائج منظمة مع عناصر تحكم لاستخراج المحتوى وتصفية النطاقات.
  </Card>
  <Card title="SearXNG" icon="server" href="/tools/searxng-search">
    بحث تجميعي مستضاف ذاتيًا. لا حاجة إلى مفتاح API. يجمع Google وBing وDuckDuckGo وغير ذلك.
  </Card>
  <Card title="Tavily" icon="globe" href="/tools/tavily">
    نتائج منظمة مع عمق البحث وتصفية الموضوع و`tavily_extract` لاستخراج عناوين URL.
  </Card>
</CardGroup>

### مقارنة بين الموفرين

| الموفر                                  | نمط النتيجة               | المرشحات                                          | مفتاح API                                                                          |
| ----------------------------------------- | -------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------- |
| [Brave](/tools/brave-search)              | مقتطفات منظمة        | البلد، اللغة، الوقت، وضع `llm-context`      | `BRAVE_API_KEY`                                                                  |
| [DuckDuckGo](/tools/duckduckgo-search)    | مقتطفات منظمة        | --                                               | لا شيء (من دون مفتاح)                                                                  |
| [Exa](/tools/exa-search)                  | منظم + مستخرج     | الوضع العصبي/الكلمات المفتاحية، التاريخ، استخراج المحتوى    | `EXA_API_KEY`                                                                    |
| [Firecrawl](/tools/firecrawl)             | مقتطفات منظمة        | عبر أداة `firecrawl_search`                      | `FIRECRAWL_API_KEY`                                                              |
| [Gemini](/tools/gemini-search)            | مركب بالذكاء الاصطناعي + استشهادات | --                                               | `GEMINI_API_KEY`                                                                 |
| [Grok](/tools/grok-search)                | مركب بالذكاء الاصطناعي + استشهادات | --                                               | `XAI_API_KEY`                                                                    |
| [Kimi](/tools/kimi-search)                | مركب بالذكاء الاصطناعي + استشهادات | --                                               | `KIMI_API_KEY` / `MOONSHOT_API_KEY`                                              |
| [MiniMax Search](/tools/minimax-search)   | مقتطفات منظمة        | المنطقة (`global` / `cn`)                         | `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY`                               |
| [Ollama Web Search](/tools/ollama-search) | مقتطفات منظمة        | --                                               | لا شيء افتراضيًا؛ يلزم `ollama signin`، ويمكنه إعادة استخدام مصادقة bearer لموفر Ollama |
| [Perplexity](/tools/perplexity-search)    | مقتطفات منظمة        | البلد، اللغة، الوقت، النطاقات، حدود المحتوى | `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY`                                      |
| [SearXNG](/tools/searxng-search)          | مقتطفات منظمة        | الفئات، اللغة                             | لا شيء (مستضاف ذاتيًا)                                                               |
| [Tavily](/tools/tavily)                   | مقتطفات منظمة        | عبر أداة `tavily_search`                         | `TAVILY_API_KEY`                                                                 |

## الاكتشاف التلقائي

## بحث الويب الأصلي في Codex

يمكن للنماذج القادرة على Codex اختياريًا استخدام أداة Responses الأصلية `web_search` الخاصة بالموفر بدلًا من دالة `web_search` المُدارة في OpenClaw.

- قم بتهيئتها ضمن `tools.web.search.openaiCodex`
- لا تتفعّل إلا للنماذج القادرة على Codex (`openai-codex/*` أو الموفرين الذين يستخدمون `api: "openai-codex-responses"`)
- يظل `web_search` المُدار مطبقًا على النماذج غير القادرة على Codex
- `mode: "cached"` هو الإعداد الافتراضي والموصى به
- يؤدي `tools.web.search.enabled: false` إلى تعطيل كل من البحث المُدار والبحث الأصلي

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        openaiCodex: {
          enabled: true,
          mode: "cached",
          allowedDomains: ["example.com"],
          contextSize: "high",
          userLocation: {
            country: "US",
            city: "New York",
            timezone: "America/New_York",
          },
        },
      },
    },
  },
}
```

إذا كان بحث Codex الأصلي مفعّلًا لكن النموذج الحالي غير قادر على Codex، فسيُبقي OpenClaw على سلوك `web_search` المُدار العادي.

## إعداد بحث الويب

قوائم الموفرين في المستندات وتدفقات الإعداد مرتبة أبجديًا. أما الاكتشاف التلقائي فيحافظ على
ترتيب أولوية منفصل.

إذا لم يتم ضبط `provider`، يتحقق OpenClaw من الموفرين بهذا الترتيب ويستخدم
أول موفر جاهز:

الموفرون المعتمدون على API أولًا:

1. **Brave** -- ‏`BRAVE_API_KEY` أو `plugins.entries.brave.config.webSearch.apiKey` (الترتيب 10)
2. **MiniMax Search** -- ‏`MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` أو `plugins.entries.minimax.config.webSearch.apiKey` (الترتيب 15)
3. **Gemini** -- ‏`GEMINI_API_KEY` أو `plugins.entries.google.config.webSearch.apiKey` (الترتيب 20)
4. **Grok** -- ‏`XAI_API_KEY` أو `plugins.entries.xai.config.webSearch.apiKey` (الترتيب 30)
5. **Kimi** -- ‏`KIMI_API_KEY` / `MOONSHOT_API_KEY` أو `plugins.entries.moonshot.config.webSearch.apiKey` (الترتيب 40)
6. **Perplexity** -- ‏`PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY` أو `plugins.entries.perplexity.config.webSearch.apiKey` (الترتيب 50)
7. **Firecrawl** -- ‏`FIRECRAWL_API_KEY` أو `plugins.entries.firecrawl.config.webSearch.apiKey` (الترتيب 60)
8. **Exa** -- ‏`EXA_API_KEY` أو `plugins.entries.exa.config.webSearch.apiKey` (الترتيب 65)
9. **Tavily** -- ‏`TAVILY_API_KEY` أو `plugins.entries.tavily.config.webSearch.apiKey` (الترتيب 70)

ثم البدائل الاحتياطية التي لا تحتاج إلى مفتاح:

10. **DuckDuckGo** -- بديل HTML احتياطي لا يحتاج إلى مفتاح ولا حساب أو مفتاح API (الترتيب 100)
11. **Ollama Web Search** -- بديل احتياطي لا يحتاج إلى مفتاح عبر مضيف Ollama المهيأ لديك؛ يتطلب أن يكون Ollama قابلًا للوصول وأن يكون مسجل الدخول عبر `ollama signin` ويمكنه إعادة استخدام مصادقة bearer الخاصة بموفر Ollama إذا احتاجها المضيف (الترتيب 110)
12. **SearXNG** -- ‏`SEARXNG_BASE_URL` أو `plugins.entries.searxng.config.webSearch.baseUrl` (الترتيب 200)

إذا لم يُكتشف أي موفر، فسيعود إلى Brave (وستظهر لك
رسالة خطأ بخصوص مفتاح مفقود تطلب منك تهيئة واحد).

<Note>
  تدعم جميع حقول مفاتيح الموفرين كائنات SecretRef. في وضع الاكتشاف التلقائي،
  يحل OpenClaw مفتاح الموفر المحدد فقط -- أما SecretRef غير المحددة
  فتبقى غير نشطة.
</Note>

## الإعدادات

```json5
{
  tools: {
    web: {
      search: {
        enabled: true, // الافتراضي: true
        provider: "brave", // أو احذفه للاكتشاف التلقائي
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
    },
  },
}
```

توجد الإعدادات الخاصة بكل موفر (مفاتيح API، وعناوين URL الأساسية، والأوضاع) تحت
`plugins.entries.<plugin>.config.webSearch.*`. راجع صفحات الموفرين
للحصول على أمثلة.

اختيار موفر الاحتياط لـ `web_fetch` منفصل:

- اختره باستخدام `tools.web.fetch.provider`
- أو احذف هذا الحقل ودع OpenClaw يكتشف تلقائيًا أول موفر web-fetch
  جاهز من بيانات الاعتماد المتاحة
- اليوم، موفر web-fetch المضمّن هو Firecrawl، ويُهيّأ تحت
  `plugins.entries.firecrawl.config.webFetch.*`

عند اختيار **Kimi** أثناء `openclaw onboard` أو
`openclaw configure --section web`، يمكن لـ OpenClaw أيضًا أن يطلب:

- منطقة Moonshot API (`https://api.moonshot.ai/v1` أو `https://api.moonshot.cn/v1`)
- نموذج Kimi الافتراضي لبحث الويب (الافتراضي `kimi-k2.5`)

بالنسبة إلى `x_search`، قم بتهيئة `plugins.entries.xai.config.xSearch.*`. فهو يستخدم
الاحتياط نفسه `XAI_API_KEY` المستخدم في بحث الويب Grok.
تُنقل إعدادات `tools.web.x_search.*` القديمة تلقائيًا بواسطة `openclaw doctor --fix`.
عند اختيار Grok أثناء `openclaw onboard` أو `openclaw configure --section web`،
يمكن لـ OpenClaw أيضًا أن يعرض إعداد `x_search` الاختياري باستخدام المفتاح نفسه.
وهذه خطوة متابعة منفصلة داخل مسار Grok، وليست خيارًا منفصلًا على المستوى الأعلى
لموفر بحث الويب. وإذا اخترت موفرًا آخر، فلن يعرض OpenClaw
مطالبة `x_search`.

### تخزين مفاتيح API

<Tabs>
  <Tab title="ملف الإعدادات">
    شغّل `openclaw configure --section web` أو اضبط المفتاح مباشرة:

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "YOUR_KEY", // pragma: allowlist secret
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="متغير البيئة">
    اضبط متغير بيئة الموفر في بيئة عملية Gateway:

    ```bash
    export BRAVE_API_KEY="YOUR_KEY"
    ```

    بالنسبة إلى تثبيت gateway، ضعه في `~/.openclaw/.env`.
    راجع [Env vars](/ar/help/faq#env-vars-and-env-loading).

  </Tab>
</Tabs>

## معاملات الأداة

| المعامل             | الوصف                                           |
| --------------------- | ----------------------------------------------------- |
| `query`               | استعلام البحث (مطلوب)                               |
| `count`               | النتائج المراد إرجاعها (1-10، الافتراضي: 5)                  |
| `country`             | رمز بلد ISO من حرفين (مثل "US"، "DE")           |
| `language`            | رمز لغة ISO 639-1 (مثل "en"، "de")             |
| `search_lang`         | رمز لغة البحث (Brave فقط)                     |
| `freshness`           | مرشح الوقت: `day` أو `week` أو `month` أو `year`        |
| `date_after`          | النتائج بعد هذا التاريخ (YYYY-MM-DD)                  |
| `date_before`         | النتائج قبل هذا التاريخ (YYYY-MM-DD)                 |
| `ui_lang`             | رمز لغة الواجهة (Brave فقط)                         |
| `domain_filter`       | مصفوفة سماح/منع للنطاقات (Perplexity فقط)     |
| `max_tokens`          | ميزانية المحتوى الإجمالية، الافتراضي 25000 (Perplexity فقط) |
| `max_tokens_per_page` | حد الرموز لكل صفحة، الافتراضي 2048 (Perplexity فقط)  |

<Warning>
  لا تعمل كل المعاملات مع جميع الموفرين. وضع Brave `llm-context`
  يرفض `ui_lang` و`freshness` و`date_after` و`date_before`.
  تعيد Gemini وGrok وKimi إجابة مركبة واحدة مع استشهادات. وهي
  تقبل `count` من أجل توافق الأداة المشتركة، لكنه لا يغير
  شكل الإجابة المرتكزة.
  يتصرف Perplexity بالطريقة نفسها عند استخدام مسار التوافق Sonar/OpenRouter
  (`plugins.entries.perplexity.config.webSearch.baseUrl` /
  `model` أو `OPENROUTER_API_KEY`).
  يقبل SearXNG عناوين `http://` فقط للمضيفين الموثوقين على الشبكة الخاصة أو loopback؛
  أما نقاط نهاية SearXNG العامة فيجب أن تستخدم `https://`.
  لا يدعم Firecrawl وTavily سوى `query` و`count` عبر `web_search`
  -- استخدم أدواتهما المخصصة للخيارات المتقدمة.
</Warning>

## x_search

تستعلم `x_search` عن منشورات X (المعروف سابقًا باسم Twitter) باستخدام xAI وتعيد
إجابات مركبة بالذكاء الاصطناعي مع استشهادات. تقبل استعلامات بلغة طبيعية و
مرشحات منظمة اختيارية. لا يفعّل OpenClaw أداة xAI المضمنة `x_search`
إلا على الطلب الذي يخدم استدعاء هذه الأداة.

<Note>
  توثق xAI أن `x_search` تدعم البحث بالكلمات المفتاحية، والبحث الدلالي، وبحث
  المستخدم، وجلب سلاسل المحادثات. وبالنسبة إلى إحصاءات التفاعل لكل منشور مثل إعادة النشر،
  والردود، والإشارات المرجعية، أو المشاهدات، ففضّل بحثًا موجّهًا لعنوان URL الخاص بالمنشور
  أو معرّف الحالة مباشرة. قد تعثر عمليات البحث العامة بالكلمات المفتاحية على المنشور الصحيح، لكنها قد تعيد
  بيانات وصفية أقل اكتمالًا لكل منشور. نمط جيد هو: حدد المنشور أولًا، ثم
  نفذ استعلام `x_search` ثانيًا يركّز على ذلك المنشور بعينه.
</Note>

### إعدادات x_search

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          xSearch: {
            enabled: true,
            model: "grok-4-1-fast-non-reasoning",
            inlineCitations: false,
            maxTurns: 2,
            timeoutSeconds: 30,
            cacheTtlMinutes: 15,
          },
          webSearch: {
            apiKey: "xai-...", // اختياري إذا كان XAI_API_KEY مضبوطًا
          },
        },
      },
    },
  },
}
```

### معاملات x_search

| المعامل                    | الوصف                                            |
| ---------------------------- | ------------------------------------------------------ |
| `query`                      | استعلام البحث (مطلوب)                                |
| `allowed_x_handles`          | قصر النتائج على أسماء مستخدمين محددة على X                 |
| `excluded_x_handles`         | استبعاد أسماء مستخدمين محددة على X                             |
| `from_date`                  | تضمين المنشورات في هذا التاريخ أو بعده فقط (YYYY-MM-DD)  |
| `to_date`                    | تضمين المنشورات في هذا التاريخ أو قبله فقط (YYYY-MM-DD) |
| `enable_image_understanding` | السماح لـ xAI بفحص الصور المرفقة بالمنشورات المطابقة      |
| `enable_video_understanding` | السماح لـ xAI بفحص الفيديوهات المرفقة بالمنشورات المطابقة      |

### مثال على x_search

```javascript
await x_search({
  query: "dinner recipes",
  allowed_x_handles: ["nytfood"],
  from_date: "2026-03-01",
});
```

```javascript
// إحصاءات كل منشور: استخدم عنوان URL الدقيق للحالة أو معرّف الحالة كلما أمكن
await x_search({
  query: "https://x.com/huntharo/status/1905678901234567890",
});
```

## أمثلة

```javascript
// بحث أساسي
await web_search({ query: "OpenClaw plugin SDK" });

// بحث خاص بألمانيا
await web_search({ query: "TV online schauen", country: "DE", language: "de" });

// نتائج حديثة (الأسبوع الماضي)
await web_search({ query: "AI developments", freshness: "week" });

// نطاق تاريخ
await web_search({
  query: "climate research",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// تصفية النطاقات (Perplexity فقط)
await web_search({
  query: "product reviews",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});
```

## ملفات تعريف الأدوات

إذا كنت تستخدم ملفات تعريف الأدوات أو قوائم السماح، فأضف `web_search` أو `x_search` أو `group:web`:

```json5
{
  tools: {
    allow: ["web_search", "x_search"],
    // أو: allow: ["group:web"]  (يتضمن web_search وx_search وweb_fetch)
  },
}
```

## ذو صلة

- [Web Fetch](/tools/web-fetch) -- اجلب عنوان URL واستخرج المحتوى القابل للقراءة
- [Web Browser](/tools/browser) -- أتمتة متصفح كاملة للمواقع الثقيلة المعتمدة على JS
- [Grok Search](/tools/grok-search) -- Grok كموفر `web_search`
- [Ollama Web Search](/tools/ollama-search) -- بحث ويب من دون مفتاح عبر مضيف Ollama الخاص بك
