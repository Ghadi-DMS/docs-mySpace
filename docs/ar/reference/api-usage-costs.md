---
read_when:
    - تريد فهم الميزات التي قد تستدعي واجهات API مدفوعة
    - تحتاج إلى مراجعة المفاتيح والتكاليف وإمكانية رؤية الاستخدام
    - أنت تشرح تقارير التكلفة في /status أو /usage
summary: راجع ما الذي يمكن أن ينفق المال، وما المفاتيح المستخدمة، وكيفية عرض الاستخدام
title: استخدام API والتكاليف
x-i18n:
    generated_at: "2026-04-05T12:55:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: 71789950fe54dcdcd3e34c8ad6e3143f749cdfff5bbc2f14be4b85aaa467b14c
    source_path: reference/api-usage-costs.md
    workflow: 15
---

# استخدام API والتكاليف

تسرد هذه الوثيقة **الميزات التي يمكن أن تستدعي مفاتيح API** وأين تظهر تكاليفها. وهي تركّز على
ميزات OpenClaw التي يمكن أن تولّد استخدامًا لمزوّد الخدمة أو استدعاءات API مدفوعة.

## أين تظهر التكاليف (الدردشة + CLI)

**لقطة تكلفة لكل جلسة**

- يعرض `/status` نموذج الجلسة الحالي، واستخدام السياق، وعدد رموز الاستجابة الأخيرة.
- إذا كان النموذج يستخدم **مصادقة بمفتاح API**، فإن `/status` يعرض أيضًا **التكلفة التقديرية** لآخر رد.
- إذا كانت بيانات تعريف الجلسة المباشرة محدودة، يمكن لـ `/status` استعادة
  عدّادات الرموز/التخزين المؤقت وعلامة نموذج وقت التشغيل النشط من أحدث إدخال
  استخدام في النص التفريغي. وتظل القيم المباشرة غير الصفرية الموجودة لها الأولوية، ويمكن
  لإجماليات النص التفريغي بحجم الموجّه أن تتغلب عندما تكون الإجماليات المخزنة مفقودة أو أصغر.

**تذييل التكلفة لكل رسالة**

- يضيف `/usage full` تذييل استخدام إلى كل رد، بما في ذلك **التكلفة التقديرية** (لمفاتيح API فقط).
- يعرض `/usage tokens` الرموز فقط؛ وتخفي تدفقات OAuth/الرموز ونمط الاشتراك وتدفقات CLI تكلفة الدولار.
- ملاحظة Gemini CLI: عندما يعيد CLI مخرجات JSON، يقرأ OpenClaw بيانات الاستخدام من
  `stats`، ويطبّع `stats.cached` إلى `cacheRead`، ويستنتج رموز الإدخال
  من `stats.input_tokens - stats.cached` عند الحاجة.

ملاحظة Anthropic: لا تزال مستندات Claude Code العامة من Anthropic تتضمن
استخدام Claude Code المباشر في الطرفية ضمن حدود خطة Claude. وبشكل منفصل،
أبلغت Anthropic مستخدمي OpenClaw بأنه ابتداءً من **4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST**، فإن
مسار تسجيل الدخول إلى Claude في **OpenClaw** سيُحتسب على أنه استخدام حزمة طرف ثالث
ويتطلب **Extra Usage** تتم فوترته بشكل منفصل عن الاشتراك. ولا توفّر Anthropic
تقديرًا بالدولار لكل رسالة يمكن لـ OpenClaw عرضه في
`/usage full`.

**نوافذ استخدام CLI (حصص المزوّد)**

- يعرض `openclaw status --usage` و `openclaw channels list` **نوافذ الاستخدام** الخاصة بالمزوّد
  (لقطات للحصة، وليست تكاليف لكل رسالة).
- يتم توحيد المخرجات المقروءة بشريًا إلى `X% left` عبر مختلف المزوّدين.
- مزوّدو نوافذ الاستخدام الحاليون: Anthropic وGitHub Copilot وGemini CLI،
  وOpenAI Codex وMiniMax وXiaomi وz.ai.
- ملاحظة MiniMax: تعني الحقول الخام `usage_percent` / `usagePercent` الحصة
  المتبقية، لذلك يقوم OpenClaw بعكسها قبل العرض. وتظل الحقول المستندة إلى العدد
  لها الأولوية عند وجودها. وإذا أعاد المزوّد `model_remains`، يفضّل OpenClaw
  إدخال نموذج الدردشة، ويشتق تسمية النافذة من الطوابع الزمنية عند الحاجة،
  ويتضمن اسم النموذج في تسمية الخطة.
- تأتي مصادقة الاستخدام لنوافذ الحصة هذه من خطافات خاصة بالمزوّد عند
  توفرها؛ وإلا يعود OpenClaw إلى مطابقة بيانات اعتماد OAuth/مفتاح API
  من ملفات تعريف المصادقة أو البيئة أو الإعدادات.

راجع [استخدام الرموز والتكاليف](/reference/token-use) للحصول على التفاصيل والأمثلة.

## كيفية اكتشاف المفاتيح

يمكن لـ OpenClaw التقاط بيانات الاعتماد من:

- **ملفات تعريف المصادقة** (لكل وكيل، ومخزنة في `auth-profiles.json`).
- **متغيرات البيئة** (مثل `OPENAI_API_KEY` و `BRAVE_API_KEY` و `FIRECRAWL_API_KEY`).
- **الإعدادات** (`models.providers.*.apiKey` و `plugins.entries.*.config.webSearch.apiKey`،
  و `plugins.entries.firecrawl.config.webFetch.apiKey` و `memorySearch.*`،
  و `talk.providers.*.apiKey`).
- **Skills** (`skills.entries.<name>.apiKey`) والتي قد تصدّر المفاتيح إلى بيئة عملية Skill.

## الميزات التي يمكن أن تنفق المفاتيح

### 1) استجابات النموذج الأساسية (الدردشة + الأدوات)

يستخدم كل رد أو استدعاء أداة **مزوّد النموذج الحالي** (OpenAI أو Anthropic أو غيرهما). وهذا هو
المصدر الأساسي للاستخدام والتكلفة.

يشمل ذلك أيضًا المزوّدين المستضافين بنمط الاشتراك الذين ما زالوا يفرضون رسومًا خارج
واجهة OpenClaw المحلية، مثل **OpenAI Codex** و **Alibaba Cloud Model Studio
Coding Plan** و **MiniMax Coding Plan** و **Z.AI / GLM Coding Plan**،
ومسار تسجيل الدخول إلى Claude في OpenClaw من Anthropic مع تفعيل **Extra Usage**.

راجع [النماذج](/ar/providers/models) لإعدادات التسعير و[استخدام الرموز والتكاليف](/reference/token-use) للعرض.

### 2) فهم الوسائط (الصوت/الصورة/الفيديو)

يمكن تلخيص الوسائط الواردة أو نسخها قبل تشغيل الرد. وهذا يستخدم واجهات API للنماذج/المزوّدين.

- الصوت: OpenAI / Groq / Deepgram / Google / Mistral.
- الصورة: OpenAI / OpenRouter / Anthropic / Google / MiniMax / Moonshot / Qwen / Z.AI.
- الفيديو: Google / Qwen / Moonshot.

راجع [فهم الوسائط](/ar/nodes/media-understanding).

### 3) إنشاء الصور والفيديو

يمكن لقدرات الإنشاء المشتركة أيضًا أن تنفق مفاتيح المزوّد:

- إنشاء الصور: OpenAI / Google / fal / MiniMax
- إنشاء الفيديو: Qwen

يمكن لإنشاء الصور استنتاج مزوّد افتراضي مدعوم بالمصادقة عندما
يكون `agents.defaults.imageGenerationModel` غير مضبوط. أما إنشاء الفيديو فيتطلب حاليًا
ضبطًا صريحًا لـ `agents.defaults.videoGenerationModel` مثل
`qwen/wan2.6-t2v`.

راجع [إنشاء الصور](/tools/image-generation) و[Qwen Cloud](/ar/providers/qwen)
و[النماذج](/ar/concepts/models).

### 4) تضمينات الذاكرة + البحث الدلالي

يستخدم البحث الدلالي في الذاكرة **واجهات API للتضمين** عند ضبطه على مزوّدين بعيدين:

- `memorySearch.provider = "openai"` → تضمينات OpenAI
- `memorySearch.provider = "gemini"` → تضمينات Gemini
- `memorySearch.provider = "voyage"` → تضمينات Voyage
- `memorySearch.provider = "mistral"` → تضمينات Mistral
- `memorySearch.provider = "ollama"` → تضمينات Ollama (محلي/مستضاف ذاتيًا؛ عادةً بلا فوترة API مستضافة)
- رجوع اختياري إلى مزوّد بعيد إذا فشلت التضمينات المحلية

يمكنك إبقاءه محليًا باستخدام `memorySearch.provider = "local"` (من دون استخدام API).

راجع [الذاكرة](/ar/concepts/memory).

### 5) أداة البحث على الويب

قد يترتب على `web_search` رسوم استخدام اعتمادًا على مزوّدك:

- **Brave Search API**: `BRAVE_API_KEY` أو `plugins.entries.brave.config.webSearch.apiKey`
- **Exa**: `EXA_API_KEY` أو `plugins.entries.exa.config.webSearch.apiKey`
- **Firecrawl**: `FIRECRAWL_API_KEY` أو `plugins.entries.firecrawl.config.webSearch.apiKey`
- **Gemini (Google Search)**: `GEMINI_API_KEY` أو `plugins.entries.google.config.webSearch.apiKey`
- **Grok (xAI)**: `XAI_API_KEY` أو `plugins.entries.xai.config.webSearch.apiKey`
- **Kimi (Moonshot)**: `KIMI_API_KEY` أو `MOONSHOT_API_KEY` أو `plugins.entries.moonshot.config.webSearch.apiKey`
- **MiniMax Search**: `MINIMAX_CODE_PLAN_KEY` أو `MINIMAX_CODING_API_KEY` أو `MINIMAX_API_KEY` أو `plugins.entries.minimax.config.webSearch.apiKey`
- **Ollama Web Search**: من دون مفتاح افتراضيًا، لكنه يتطلب مضيف Ollama يمكن الوصول إليه بالإضافة إلى `ollama signin`؛ ويمكنه أيضًا إعادة استخدام مصادقة الحامل العادية لمزوّد Ollama عندما يتطلبها المضيف
- **Perplexity Search API**: `PERPLEXITY_API_KEY` أو `OPENROUTER_API_KEY` أو `plugins.entries.perplexity.config.webSearch.apiKey`
- **Tavily**: `TAVILY_API_KEY` أو `plugins.entries.tavily.config.webSearch.apiKey`
- **DuckDuckGo**: رجوع احتياطي بلا مفتاح (من دون فوترة API، لكنه غير رسمي ويعتمد على HTML)
- **SearXNG**: `SEARXNG_BASE_URL` أو `plugins.entries.searxng.config.webSearch.baseUrl` (بلا مفتاح/مستضاف ذاتيًا؛ من دون فوترة API مستضافة)

لا تزال مسارات المزوّد القديمة `tools.web.search.*` تُحمَّل عبر طبقة التوافق المؤقتة، لكنها لم تعد سطح الإعدادات الموصى به.

**رصيد Brave Search المجاني:** تتضمن كل خطة من Brave رصيدًا مجانيًا متجددًا
بقيمة \$5 شهريًا. وتبلغ تكلفة خطة Search ‏\$5 لكل 1,000 طلب، لذا يغطي الرصيد
1,000 طلب شهريًا من دون رسوم. اضبط حد الاستخدام الخاص بك في لوحة تحكم Brave
لتجنب الرسوم غير المتوقعة.

راجع [أدوات الويب](/tools/web).

### 5) أداة جلب الويب (Firecrawl)

يمكن لـ `web_fetch` استدعاء **Firecrawl** عند وجود مفتاح API:

- `FIRECRAWL_API_KEY` أو `plugins.entries.firecrawl.config.webFetch.apiKey`

إذا لم يتم إعداد Firecrawl، تعود الأداة إلى الجلب المباشر + readability (من دون API مدفوع).

راجع [أدوات الويب](/tools/web).

### 6) لقطات استخدام المزوّد (الحالة/الصحة)

تستدعي بعض أوامر الحالة **نقاط نهاية استخدام المزوّد** لعرض نوافذ الحصة أو صحة المصادقة.
وعادةً ما تكون هذه الاستدعاءات منخفضة الحجم، لكنها ما زالت تضرب واجهات API الخاصة بالمزوّد:

- `openclaw status --usage`
- `openclaw models status --json`

راجع [CLI النماذج](/cli/models).

### 7) تلخيص وسيلة الحماية للضغط

يمكن لوسيلة الحماية الخاصة بالضغط أن تلخّص محفوظات الجلسة باستخدام **النموذج الحالي**،
مما يستدعي واجهات API الخاصة بالمزوّد عند تشغيلها.

راجع [إدارة الجلسة + الضغط](/reference/session-management-compaction).

### 8) فحص / استكشاف النموذج

يمكن لـ `openclaw models scan` اختبار نماذج OpenRouter ويستخدم `OPENROUTER_API_KEY` عندما
يكون الاختبار مفعّلًا.

راجع [CLI النماذج](/cli/models).

### 9) Talk (الكلام)

يمكن لوضع Talk استدعاء **ElevenLabs** عند إعداده:

- `ELEVENLABS_API_KEY` أو `talk.providers.elevenlabs.apiKey`

راجع [وضع Talk](/ar/nodes/talk).

### 10) Skills (واجهات API لجهات خارجية)

يمكن لـ Skills تخزين `apiKey` في `skills.entries.<name>.apiKey`. وإذا استخدمت Skill هذا المفتاح لواجهات API خارجية،
فقد تترتب عليها تكاليف وفقًا لمزوّد Skill.

راجع [Skills](/tools/skills).
