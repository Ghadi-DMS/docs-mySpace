---
read_when:
    - تحتاج إلى مرجع لإعداد النماذج حسب كل مزوّد
    - تريد أمثلة للإعداد أو أوامر الإعداد الأولي عبر CLI لموفري النماذج
summary: نظرة عامة على مزوّدي النماذج مع أمثلة للإعداد + تدفقات CLI
title: موفرو النماذج
x-i18n:
    generated_at: "2026-04-05T12:42:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5d8f56a2a5319de03f7b86e7b19b9a89e7023f757930b5b5949568f680352a3a
    source_path: concepts/model-providers.md
    workflow: 15
---

# موفرو النماذج

تغطي هذه الصفحة **موفري LLM/النماذج** (وليس قنوات الدردشة مثل WhatsApp/Telegram).
لقواعد اختيار النماذج، راجع [/concepts/models](/concepts/models).

## قواعد سريعة

- تستخدم مراجع النماذج الصيغة `provider/model` (مثال: `opencode/claude-opus-4-6`).
- إذا ضبطت `agents.defaults.models`، فستصبح قائمة السماح.
- مساعدات CLI: ‏`openclaw onboard`، و`openclaw models list`، و`openclaw models set <provider/model>`.
- قواعد التشغيل الاحتياطية وقت التشغيل، ومجسات التهدئة، واستمرارية تجاوزات الجلسات
  موثقة في [/concepts/model-failover](/concepts/model-failover).
- `models.providers.*.models[].contextWindow` هي بيانات وصفية أصلية للنموذج؛
  أما `models.providers.*.models[].contextTokens` فهو الحد الفعلي وقت التشغيل.
- يمكن للمكونات الإضافية الخاصة بالمزوّدات حقن فهارس النماذج عبر `registerProvider({ catalog })`؛
  ويقوم OpenClaw بدمج هذا الناتج في `models.providers` قبل كتابة
  `models.json`.
- يمكن لملفات manifest الخاصة بالمزوّدات إعلان `providerAuthEnvVars` حتى لا تحتاج
  مجسات المصادقة العامة المعتمدة على env إلى تحميل وقت تشغيل المكون الإضافي. أما خريطة
  متغيرات env الأساسية المتبقية فهي الآن فقط للمزوّدات غير المعتمدة على المكونات الإضافية/الأساسية
  وبعض حالات الأسبقية العامة مثل الإعداد الأولي لـ Anthropic مع أولوية API key.
- يمكن للمكونات الإضافية الخاصة بالمزوّدات أيضًا امتلاك سلوك وقت تشغيل المزوّد عبر
  `normalizeModelId`, `normalizeTransport`, `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`,
  `normalizeResolvedModel`, `contributeResolvedModelCompat`,
  `capabilities`, `normalizeToolSchemas`,
  `inspectToolSchemas`, `resolveReasoningOutputMode`,
  `prepareExtraParams`, `createStreamFn`, `wrapStreamFn`,
  `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`,
  `createEmbeddingProvider`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`,
  `matchesContextOverflowError`, `classifyFailoverReason`,
  `isCacheTtlEligible`, `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot`, و
  `onModelSelected`.
- ملاحظة: إن `capabilities` الخاصة بوقت تشغيل المزوّد هي بيانات وصفية مشتركة للمشغّل
  (عائلة المزوّد، وخصائص النصوص/الأدوات، وتلميحات النقل/التخزين المؤقت). وهي ليست
  نفسها [نموذج الإمكانات العامة](/plugins/architecture#public-capability-model)
  الذي يصف ما الذي يسجله المكون الإضافي (استدلال نصي، وكلام، وغير ذلك).

## السلوك المملوك للمزوّد عبر المكونات الإضافية

يمكن للمكونات الإضافية الخاصة بالمزوّدات الآن امتلاك معظم المنطق الخاص بالمزوّد، بينما يحتفظ OpenClaw
بحلقة الاستدلال العامة.

التقسيم المعتاد:

- `auth[].run` / `auth[].runNonInteractive`: يمتلك المزوّد تدفقات الإعداد الأولي/تسجيل الدخول
  لـ `openclaw onboard` و`openclaw models auth` والإعداد غير التفاعلي
- `wizard.setup` / `wizard.modelPicker`: يمتلك المزوّد تسميات خيارات المصادقة،
  والأسماء البديلة القديمة، وتلميحات قائمة السماح في الإعداد الأولي، وإدخالات
  الإعداد في منتقيات الإعداد الأولي/النموذج
- `catalog`: يظهر المزوّد في `models.providers`
- `normalizeModelId`: يطبّع المزوّد معرّفات النماذج القديمة/المعاينة قبل
  البحث أو التحويل إلى الصيغة القانونية
- `normalizeTransport`: يطبّع المزوّد `api` / `baseUrl` الخاصة بعائلة النقل
  قبل تجميع النموذج العام؛ ويتحقق OpenClaw من المزوّد المطابق أولًا،
  ثم من المكونات الإضافية الأخرى القادرة على استخدام الخطافات حتى يُجري أحدها
  تغييرًا فعليًا على النقل
- `normalizeConfig`: يطبّع المزوّد إعداد `models.providers.<id>` قبل
  أن يستخدمه وقت التشغيل؛ ويتحقق OpenClaw من المزوّد المطابق أولًا، ثم من
  المكونات الإضافية الأخرى القادرة على استخدام الخطافات حتى يُجري أحدها
  تغييرًا فعليًا على الإعداد. وإذا لم تُعد كتابة الإعداد عبر خطاف مزوّد،
  تستمر المساعدات المضمّنة الخاصة بعائلة Google في
  تطبيع إدخالات مزوّدي Google المدعومة.
- `applyNativeStreamingUsageCompat`: يطبّق المزوّد عمليات إعادة كتابة توافق الاستخدام للبث الأصلي المدفوعة بنقطة النهاية لمزوّدي الإعداد
- `resolveConfigApiKey`: يحل المزوّد مصادقة env-marker لمزوّدي الإعداد
  من دون فرض تحميل مصادقة وقت التشغيل الكاملة. ويحتوي `amazon-bedrock` أيضًا على
  محلل AWS env-marker مضمّن هنا، رغم أن مصادقة وقت تشغيل Bedrock تستخدم
  سلسلة AWS SDK الافتراضية.
- `resolveSyntheticAuth`: يمكن للمزوّد عرض توفّر المصادقة المحلية/المستضافة ذاتيًا
  أو غيرها من أنواع المصادقة المدعومة من الإعداد من دون حفظ أسرار نصية صريحة
- `shouldDeferSyntheticProfileAuth`: يمكن للمزوّد وضع العناصر النائبة المخزنة لملفات التعريف الاصطناعية
  بأولوية أقل من المصادقة المدعومة من env/config
- `resolveDynamicModel`: يقبل المزوّد معرّفات نماذج غير موجودة بعد في
  الفهرس الثابت المحلي
- `prepareDynamicModel`: يحتاج المزوّد إلى تحديث بيانات وصفية قبل إعادة محاولة
  الحل الديناميكي
- `normalizeResolvedModel`: يحتاج المزوّد إلى إعادة كتابة النقل أو base URL
- `contributeResolvedModelCompat`: يقدّم المزوّد علامات توافق خاصة
  بنماذج البائع لديه حتى عندما تصل عبر نقل متوافق آخر
- `capabilities`: ينشر المزوّد خصائص النصوص/الأدوات/عائلة المزوّد
- `normalizeToolSchemas`: ينظف المزوّد schema الأدوات قبل أن يراها
  المشغّل المضمّن
- `inspectToolSchemas`: يعرض المزوّد تحذيرات schema خاصة بالنقل
  بعد التطبيع
- `resolveReasoningOutputMode`: يختار المزوّد عقود مخرجات الاستدلال
  الأصلية مقابل الموسومة
- `prepareExtraParams`: يضبط المزوّد القيم الافتراضية أو يطبّع معلمات الطلب لكل نموذج
- `createStreamFn`: يستبدل المزوّد مسار البث العادي بنقل مخصص بالكامل
- `wrapStreamFn`: يطبّق المزوّد أغلفة توافق للطلب/الترويسات/الجسم/النموذج
- `resolveTransportTurnState`: يزوّد المزوّد ترويسات أو بيانات وصفية
  أصلية لكل دور
- `resolveWebSocketSessionPolicy`: يزوّد المزوّد ترويسات جلسة WebSocket أصلية
  أو سياسة تهدئة الجلسة
- `createEmbeddingProvider`: يمتلك المزوّد سلوك embeddings الذاكرة عندما
  يكون من الأنسب أن يعيش داخل المكون الإضافي للمزوّد بدل لوحة التحويل الأساسية للـ embeddings
- `formatApiKey`: ينسق المزوّد ملفات تعريف المصادقة المخزنة إلى سلسلة
  `apiKey` الخاصة بوقت التشغيل والمتوقعة من النقل
- `refreshOAuth`: يمتلك المزوّد تحديث OAuth عندما لا تكفي أدوات التحديث
  المشتركة `pi-ai`
- `buildAuthDoctorHint`: يضيف المزوّد إرشادات إصلاح عند فشل
  تحديث OAuth
- `matchesContextOverflowError`: يتعرف المزوّد على
  أخطاء تجاوز نافذة السياق الخاصة به والتي قد تفوتها
  الاستدلالات العامة
- `classifyFailoverReason`: يربط المزوّد الأخطاء الخام الخاصة بالنقل/API
  بأسباب الانتقال الاحتياطي مثل rate limit أو overload
- `isCacheTtlEligible`: يحدد المزوّد أي معرّفات النماذج العليا تدعم TTL للتخزين المؤقت للموجّه
- `buildMissingAuthMessage`: يستبدل المزوّد خطأ مخزن المصادقة العام
  بتلميح استعادة خاص بالمزوّد
- `suppressBuiltInModel`: يخفي المزوّد الصفوف العليا القديمة ويمكن أن يعيد
  خطأً مملوكًا للبائع عند فشل الحل المباشر
- `augmentModelCatalog`: يضيف المزوّد صفوف فهرس اصطناعية/نهائية بعد
  الاكتشاف ودمج الإعداد
- `isBinaryThinking`: يمتلك المزوّد تجربة الاستخدام الثنائية للتفكير تشغيل/إيقاف
- `supportsXHighThinking`: يفعّل المزوّد `xhigh`
  لنماذج محددة
- `resolveDefaultThinkingLevel`: يمتلك المزوّد سياسة `/think` الافتراضية لعائلة
  من النماذج
- `applyConfigDefaults`: يطبّق المزوّد القيم الافتراضية العامة الخاصة به
  أثناء تحويل الإعداد اعتمادًا على وضع المصادقة أو env أو عائلة النموذج
- `isModernModelRef`: يمتلك المزوّد مطابقة النموذج المفضّل live/smoke
- `prepareRuntimeAuth`: يحوّل المزوّد بيانات اعتماد مُعدّة إلى token قصير العمر لوقت التشغيل
- `resolveUsageAuth`: يحل المزوّد بيانات اعتماد الاستخدام/الحصة لـ `/usage`
  والأسطح ذات الصلة بالحالة/التقارير
- `fetchUsageSnapshot`: يمتلك المزوّد جلب/تحليل نقطة نهاية الاستخدام بينما
  يحتفظ الأساس بقالب الملخص والتنسيق
- `onModelSelected`: يشغّل المزوّد تأثيرات لاحقة بعد الاختيار مثل
  telemetry أو حفظ جلسة مملوك للمزوّد

الأمثلة المضمّنة الحالية:

- `anthropic`: بديل توافق مستقبلي لـ Claude 4.6، وتلميحات إصلاح المصادقة، وجلب
  نقطة نهاية الاستخدام، وبيانات cache-TTL/عائلة المزوّد، وقيم افتراضية
  عامة للإعداد مدركة للمصادقة
- `amazon-bedrock`: مطابقة تجاوز السياق وتصنيف
  أسباب الانتقال الاحتياطي المملوكة للمزوّد لأخطاء Bedrock الخاصة مثل throttle/not-ready، بالإضافة إلى
  عائلة إعادة التشغيل المشتركة `anthropic-by-model` لحمايات
  سياسات إعادة التشغيل الخاصة بـ Claude فقط على حركة Anthropic
- `anthropic-vertex`: حمايات سياسات إعادة التشغيل الخاصة بـ Claude فقط على
  حركة Anthropic-message
- `openrouter`: معرّفات نماذج تمريرية، وأغلفة طلبات، وتلميحات إمكانات المزوّد،
  وتنقية Gemini thought-signature على حركة Gemini المارة عبر proxy،
  وحقن الاستدلال عبر proxy من خلال عائلة البث `openrouter-thinking`،
  وتمرير بيانات التوجيه الوصفية، وسياسة cache-TTL
- `github-copilot`: إعداد أولي/تسجيل دخول الجهاز، وبديل نموذج لتوافق مستقبلي،
  وتلميحات نص Claude-thinking، وتبادل token وقت التشغيل، وجلب
  نقطة نهاية الاستخدام
- `openai`: بديل توافق مستقبلي لـ GPT-5.4، وتطبيع نقل OpenAI المباشر،
  وتلميحات غياب المصادقة المدركة لـ Codex، وكبت Spark، وصفوف فهرس
  اصطناعية لـ OpenAI/Codex، وسياسة التفكير/النماذج الحية، وتطبيع أسماء usage-token البديلة
  (`input` / `output` وعائلتا `prompt` / `completion`)، وعائلة البث المشتركة
  `openai-responses-defaults` لأغلفة OpenAI/Codex الأصلية، وبيانات
  عائلة المزوّد
- `google` و`google-gemini-cli`: بديل توافق مستقبلي لـ Gemini 3.1،
  والتحقق من إعادة التشغيل الأصلي لـ Gemini، وتنقية إعادة التشغيل في bootstrap،
  ووضع مخرجات الاستدلال الموسوم، ومطابقة النماذج الحديثة؛ كما تمتلك Gemini CLI OAuth
  أيضًا تنسيق token لملفات تعريف المصادقة، وتحليل usage-token، وجلب
  نقطة نهاية الحصة لأسطح الاستخدام
- `moonshot`: نقل مشترك، وتطبيع حمولة التفكير المملوك للمكون الإضافي
- `kilocode`: نقل مشترك، وترويسات طلبات مملوكة للمكون الإضافي، وتطبيع
  حمولة الاستدلال، وتنقية proxy-Gemini thought-signature، وسياسة
  cache-TTL
- `zai`: بديل توافق مستقبلي لـ GLM-5، وقيم `tool_stream` الافتراضية، وسياسة cache-TTL،
  وسياسة التفكير الثنائي/النماذج الحية، ومصادقة الاستخدام + جلب الحصة؛
  أما المعرّفات غير المعروفة `glm-5*` فتُنشأ اصطناعيًا من القالب المضمّن `glm-4.7`
- `xai`: تطبيع نقل Responses الأصلي، وإعادة كتابة الأسماء البديلة `/fast` لمتغيرات
  Grok السريعة، والقيمة الافتراضية `tool_stream`، وتنظيف
  schema الأدوات / حمولة الاستدلال الخاصة بـ xAI
- `mistral`: بيانات إمكانات مملوكة للمكون الإضافي
- `opencode` و`opencode-go`: بيانات إمكانات مملوكة للمكون الإضافي بالإضافة إلى
  تنقية proxy-Gemini thought-signature
- `byteplus`, `cloudflare-ai-gateway`, `huggingface`, `kimi`,
  `nvidia`, `qianfan`, `stepfun`, `synthetic`, `together`, `venice`,
  `vercel-ai-gateway`, و`volcengine`: فهارس مملوكة للمكونات الإضافية فقط
- `qwen`: فهارس مملوكة للمكونات الإضافية للنماذج النصية بالإضافة إلى
  تسجيلات موفر media-understanding وتوليد الفيديو المشتركة
  للأسطح متعددة الوسائط الخاصة به؛ ويستخدم توليد الفيديو في Qwen نقاط النهاية القياسية
  للفيديو في DashScope مع نماذج Wan المضمّنة مثل `wan2.6-t2v` و`wan2.7-r2v`
- `minimax`: فهارس مملوكة للمكونات الإضافية، واختيار مختلط لسياسات إعادة التشغيل
  بين Anthropic/OpenAI، ومنطق مصادقة/لقطة الاستخدام
- `xiaomi`: فهارس مملوكة للمكونات الإضافية بالإضافة إلى منطق مصادقة/لقطة الاستخدام

يمتلك المكون الإضافي المضمّن `openai` الآن كلا معرفي المزوّد:
`openai` و`openai-codex`.

ويغطي هذا المزوّدات التي ما زالت تناسب وسائل النقل العادية في OpenClaw. أما المزوّد
الذي يحتاج إلى منفّذ طلبات مخصص بالكامل فهو سطح امتداد منفصل وأعمق.

## تدوير API key

- يدعم تدويرًا عامًا للمزوّدات لمزوّدات محددة.
- اضبط عدة مفاتيح عبر:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (تجاوز حي واحد، أعلى أولوية)
  - `<PROVIDER>_API_KEYS` (قائمة مفصولة بفواصل أو فاصلة منقوطة)
  - `<PROVIDER>_API_KEY` (المفتاح الأساسي)
  - `<PROVIDER>_API_KEY_*` (قائمة مرقمة، مثل `<PROVIDER>_API_KEY_1`)
- بالنسبة إلى مزوّدي Google، يُدرج `GOOGLE_API_KEY` أيضًا كبديل احتياطي.
- يحافظ ترتيب اختيار المفاتيح على الأولوية ويزيل التكرارات.
- لا تُعاد محاولة الطلبات بالمفتاح التالي إلا عند استجابات rate-limit (مثل
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded`، أو رسائل حدود استخدام دورية).
- تفشل الأخطاء غير المرتبطة بـ rate-limit مباشرة؛ ولا يُجرى أي تدوير للمفاتيح.
- عندما تفشل جميع المفاتيح المرشحة، يُعاد الخطأ النهائي من آخر محاولة.

## المزوّدون المضمّنون (فهرس pi-ai)

يأتي OpenClaw مع فهرس pi‑ai. لا تتطلب هذه المزوّدات أي
إعداد `models.providers`؛ فقط اضبط المصادقة + اختر نموذجًا.

### OpenAI

- المزوّد: `openai`
- المصادقة: `OPENAI_API_KEY`
- التدوير الاختياري: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`، بالإضافة إلى `OPENCLAW_LIVE_OPENAI_KEY` (تجاوز فردي)
- أمثلة النماذج: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: ‏`openclaw onboard --auth-choice openai-api-key`
- النقل الافتراضي هو `auto` ‏(WebSocket أولًا، وSSE احتياطيًا)
- تجاوز لكل نموذج عبر `agents.defaults.models["openai/<model>"].params.transport` ‏(`"sse"`, `"websocket"`, أو `"auto"`)
- تكون تهيئة OpenAI Responses WebSocket الافتراضية مفعلة عبر `params.openaiWsWarmup` ‏(`true`/`false`)
- يمكن تفعيل المعالجة ذات الأولوية في OpenAI عبر `agents.defaults.models["openai/<model>"].params.serviceTier`
- يقوم `/fast` و`params.fastMode` بربط طلبات Responses المباشرة `openai/*` إلى `service_tier=priority` على `api.openai.com`
- استخدم `params.serviceTier` عندما تريد مستوى صريحًا بدل مفتاح `/fast` المشترك
- لا تنطبق ترويسات الإسناد المخفية الخاصة بـ OpenClaw (`originator`, `version`,
  `User-Agent`) إلا على حركة OpenAI الأصلية إلى `api.openai.com`، وليس
  على الوكلاء المتوافقين عمومًا مع OpenAI
- كما تحتفظ مسارات OpenAI الأصلية بـ Responses `store`، وتلميحات التخزين المؤقت للموجّه، و
  تشكيل حمولة توافق الاستدلال الخاصة بـ OpenAI؛ أما مسارات proxy فلا تفعل ذلك
- تم كبت `openai/gpt-5.3-codex-spark` عمدًا في OpenClaw لأن OpenAI API الحية ترفضه؛ ويُعامل Spark على أنه Codex-only

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- المزوّد: `anthropic`
- المصادقة: `ANTHROPIC_API_KEY`
- التدوير الاختياري: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`، بالإضافة إلى `OPENCLAW_LIVE_ANTHROPIC_KEY` (تجاوز فردي)
- مثال النموذج: `anthropic/claude-opus-4-6`
- CLI: ‏`openclaw onboard --auth-choice apiKey` أو `openclaw onboard --auth-choice anthropic-cli`
- تدعم طلبات Anthropic العامة المباشرة مفتاح `/fast` المشترك و`params.fastMode`، بما في ذلك حركة المرور المصادَق عليها عبر API key وOAuth المرسلة إلى `api.anthropic.com`؛ ويربط OpenClaw ذلك إلى `service_tier` في Anthropic ‏(`auto` مقابل `standard_only`)
- ملاحظة فوترة: ما تزال وثائق Claude Code العامة من Anthropic تتضمن استخدام Claude Code المباشر من الطرفية ضمن حدود خطة Claude. وبشكل منفصل، أبلغت Anthropic مستخدمي OpenClaw في **4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** أن مسار تسجيل الدخول إلى Claude ضمن **OpenClaw** يُعد استخدامًا لحزمة طرف ثالث ويتطلب **Extra Usage** تُحتسب بشكل منفصل عن الاشتراك.
- أصبح Anthropic setup-token متاحًا مرة أخرى كمسار OpenClaw قديم/يدوي. استخدمه مع توقع أن Anthropic أبلغت مستخدمي OpenClaw أن هذا المسار يتطلب **Extra Usage**.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- المزوّد: `openai-codex`
- المصادقة: OAuth ‏(ChatGPT)
- مثال النموذج: `openai-codex/gpt-5.4`
- CLI: ‏`openclaw onboard --auth-choice openai-codex` أو `openclaw models auth login --provider openai-codex`
- النقل الافتراضي هو `auto` ‏(WebSocket أولًا، وSSE احتياطيًا)
- تجاوز لكل نموذج عبر `agents.defaults.models["openai-codex/<model>"].params.transport` ‏(`"sse"`, `"websocket"`, أو `"auto"`)
- يُمرَّر `params.serviceTier` أيضًا في طلبات Codex Responses الأصلية (`chatgpt.com/backend-api`)
- لا تُرفق ترويسات الإسناد المخفية الخاصة بـ OpenClaw (`originator`, `version`,
  `User-Agent`) إلا على حركة Codex الأصلية إلى
  `chatgpt.com/backend-api`، وليس على الوكلاء المتوافقين عمومًا مع OpenAI
- يشترك في مفتاح `/fast` نفسه وإعداد `params.fastMode` كما في `openai/*` المباشر؛ ويربط OpenClaw ذلك إلى `service_tier=priority`
- يبقى `openai-codex/gpt-5.3-codex-spark` متاحًا عندما يوفّره فهرس Codex OAuth؛ ويتوقف على الاستحقاق
- يحتفظ `openai-codex/gpt-5.4` بالقيمة الأصلية `contextWindow = 1050000` والحد الافتراضي وقت التشغيل `contextTokens = 272000`؛ ويمكنك تجاوز الحد وقت التشغيل عبر `models.providers.openai-codex.models[].contextTokens`
- ملاحظة سياسة: إن OpenAI Codex OAuth مدعوم صراحةً للأدوات/سير العمل الخارجية مثل OpenClaw.

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.4", contextTokens: 160000 }],
      },
    },
  },
}
```

### خيارات مستضافة أخرى بنمط الاشتراك

- [Qwen Cloud](/providers/qwen): سطح موفر Qwen Cloud بالإضافة إلى ربط Alibaba DashScope وCoding Plan
- [MiniMax](/providers/minimax): وصول MiniMax Coding Plan عبر OAuth أو API key
- [GLM Models](/providers/glm): نقاط نهاية Z.AI Coding Plan أو واجهات API العامة

### OpenCode

- المصادقة: `OPENCODE_API_KEY` ‏(أو `OPENCODE_ZEN_API_KEY`)
- مزوّد وقت تشغيل Zen: ‏`opencode`
- مزوّد وقت تشغيل Go: ‏`opencode-go`
- أمثلة النماذج: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: ‏`openclaw onboard --auth-choice opencode-zen` أو `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini ‏(API key)

- المزوّد: `google`
- المصادقة: `GEMINI_API_KEY`
- التدوير الاختياري: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, والبديل `GOOGLE_API_KEY`، بالإضافة إلى `OPENCLAW_LIVE_GEMINI_KEY` (تجاوز فردي)
- أمثلة النماذج: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- التوافق: يُطبَّع إعداد OpenClaw القديم الذي يستخدم `google/gemini-3.1-flash-preview` إلى `google/gemini-3-flash-preview`
- CLI: ‏`openclaw onboard --auth-choice gemini-api-key`
- تقبل تشغيلات Gemini المباشرة أيضًا `agents.defaults.models["google/<model>"].params.cachedContent`
  (أو الاسم القديم `cached_content`) لتمرير
  مقبض `cachedContents/...` أصلي خاص بالمزوّد؛ وتظهر إصابات التخزين المؤقت في Gemini على أنها `cacheRead` في OpenClaw

### Google Vertex وGemini CLI

- المزوّدون: `google-vertex`, `google-gemini-cli`
- المصادقة: يستخدم Vertex ‏gcloud ADC؛ ويستخدم Gemini CLI تدفق OAuth الخاص به
- تحذير: إن Gemini CLI OAuth في OpenClaw تكامل غير رسمي. وقد أبلغ بعض المستخدمين عن قيود على حسابات Google بعد استخدام عملاء من أطراف ثالثة. راجع شروط Google واستخدم حسابًا غير حرج إذا اخترت المتابعة.
- يُشحن Gemini CLI OAuth كجزء من المكون الإضافي المضمّن `google`.
  - ثبّت Gemini CLI أولًا:
    - `brew install gemini-cli`
    - أو `npm install -g @google/gemini-cli`
  - فعّل: `openclaw plugins enable google`
  - سجل الدخول: `openclaw models auth login --provider google-gemini-cli --set-default`
  - النموذج الافتراضي: `google-gemini-cli/gemini-3.1-pro-preview`
  - ملاحظة: أنت **لا** تلصق client id أو secret في `openclaw.json`. إذ يخزن تدفق تسجيل الدخول في CLI
    الرموز في ملفات تعريف المصادقة على مضيف gateway.
  - إذا فشلت الطلبات بعد تسجيل الدخول، فاضبط `GOOGLE_CLOUD_PROJECT` أو `GOOGLE_CLOUD_PROJECT_ID` على مضيف gateway.
  - تُحلّل ردود Gemini CLI JSON من `response`؛ ويعود الاستخدام إلى
    `stats`، مع تطبيع `stats.cached` إلى `cacheRead` في OpenClaw.

### Z.AI ‏(GLM)

- المزوّد: `zai`
- المصادقة: `ZAI_API_KEY`
- مثال النموذج: `zai/glm-5`
- CLI: ‏`openclaw onboard --auth-choice zai-api-key`
  - الأسماء البديلة: تُطبّع `z.ai/*` و`z-ai/*` إلى `zai/*`
  - يقوم `zai-api-key` بالكشف التلقائي عن نقطة نهاية Z.AI المطابقة؛ بينما تفرض `zai-coding-global`, `zai-coding-cn`, `zai-global`, و`zai-cn` سطحًا محددًا

### Vercel AI Gateway

- المزوّد: `vercel-ai-gateway`
- المصادقة: `AI_GATEWAY_API_KEY`
- مثال النموذج: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: ‏`openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- المزوّد: `kilocode`
- المصادقة: `KILOCODE_API_KEY`
- مثال النموذج: `kilocode/kilo/auto`
- CLI: ‏`openclaw onboard --auth-choice kilocode-api-key`
- Base URL: ‏`https://api.kilo.ai/api/gateway/`
- يأتي الفهرس الاحتياطي الثابت مع `kilocode/kilo/auto`؛ بينما يمكن لاكتشاف
  `https://api.kilo.ai/api/gateway/models` الحي توسيع فهرس وقت التشغيل أكثر.
- إن التوجيه الفعلي الأعلى وراء `kilocode/kilo/auto` تملكه Kilo Gateway،
  وليس مشفرًا بشكل ثابت داخل OpenClaw.

راجع [/providers/kilocode](/providers/kilocode) لتفاصيل الإعداد.

### مكونات إضافية أخرى مضمّنة للمزوّدات

- OpenRouter: ‏`openrouter` ‏(`OPENROUTER_API_KEY`)
- مثال النموذج: `openrouter/auto`
- يطبّق OpenClaw ترويسات إسناد التطبيق الموثقة في OpenRouter فقط عندما
  يكون الطلب موجّهًا فعليًا إلى `openrouter.ai`
- كما أن علامات `cache_control` الخاصة بـ Anthropic في OpenRouter
  محصورة أيضًا في مسارات OpenRouter المتحقق منها، وليست لعناوين proxy العشوائية
- يبقى OpenRouter على مسار proxy المتوافق بنمط OpenAI، لذلك لا يُمرَّر
  تشكيل الطلبات الأصلي الخاص بـ OpenAI فقط (`serviceTier`, Responses `store`,
  تلميحات التخزين المؤقت للموجّه، حمولات توافق الاستدلال في OpenAI)
- تحتفظ مراجع OpenRouter المدعومة من Gemini
  فقط بمسار تنقية proxy-Gemini thought-signature؛ أما التحقق الأصلي من إعادة تشغيل Gemini
  وإعادات كتابة bootstrap فتبقى معطلة
- Kilo Gateway: ‏`kilocode` ‏(`KILOCODE_API_KEY`)
- مثال النموذج: `kilocode/kilo/auto`
- تحتفظ مراجع Kilo المدعومة من Gemini بمسار تنقية
  proxy-Gemini thought-signature نفسه؛ أما `kilocode/kilo/auto` وتلميحات proxy-reasoning-unsupported الأخرى
  فتتجاوز حقن الاستدلال عبر proxy
- MiniMax: ‏`minimax` ‏(API key) و`minimax-portal` ‏(OAuth)
- المصادقة: `MINIMAX_API_KEY` لـ `minimax`؛ و`MINIMAX_OAUTH_TOKEN` أو `MINIMAX_API_KEY` لـ `minimax-portal`
- مثال النموذج: `minimax/MiniMax-M2.7` أو `minimax-portal/MiniMax-M2.7`
- يكتب إعداد MiniMax والإعداد عبر API key تعريفات صريحة
  لنموذج M2.7 مع `input: ["text", "image"]`؛ بينما يبقي فهرس المزوّد المضمّن
  مراجع الدردشة نصية فقط حتى يتم تحويل إعداد هذا المزوّد إلى هيئة فعلية
- Moonshot: ‏`moonshot` ‏(`MOONSHOT_API_KEY`)
- مثال النموذج: `moonshot/kimi-k2.5`
- Kimi Coding: ‏`kimi` ‏(`KIMI_API_KEY` أو `KIMICODE_API_KEY`)
- مثال النموذج: `kimi/kimi-code`
- Qianfan: ‏`qianfan` ‏(`QIANFAN_API_KEY`)
- مثال النموذج: `qianfan/deepseek-v3.2`
- Qwen Cloud: ‏`qwen` ‏(`QWEN_API_KEY`, `MODELSTUDIO_API_KEY`, أو `DASHSCOPE_API_KEY`)
- مثال النموذج: `qwen/qwen3.5-plus`
- NVIDIA: ‏`nvidia` ‏(`NVIDIA_API_KEY`)
- مثال النموذج: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: ‏`stepfun` / `stepfun-plan` ‏(`STEPFUN_API_KEY`)
- أمثلة النماذج: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: ‏`together` ‏(`TOGETHER_API_KEY`)
- مثال النموذج: `together/moonshotai/Kimi-K2.5`
- Venice: ‏`venice` ‏(`VENICE_API_KEY`)
- Xiaomi: ‏`xiaomi` ‏(`XIAOMI_API_KEY`)
- مثال النموذج: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: ‏`vercel-ai-gateway` ‏(`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: ‏`huggingface` ‏(`HUGGINGFACE_HUB_TOKEN` أو `HF_TOKEN`)
- Cloudflare AI Gateway: ‏`cloudflare-ai-gateway` ‏(`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: ‏`volcengine` ‏(`VOLCANO_ENGINE_API_KEY`)
- مثال النموذج: `volcengine-plan/ark-code-latest`
- BytePlus: ‏`byteplus` ‏(`BYTEPLUS_API_KEY`)
- مثال النموذج: `byteplus-plan/ark-code-latest`
- xAI: ‏`xai` ‏(`XAI_API_KEY`)
  - تستخدم طلبات xAI الأصلية المضمّنة في OpenClaw مسار xAI Responses
  - يقوم `/fast` أو `params.fastMode: true` بإعادة كتابة `grok-3`, `grok-3-mini`,
    `grok-4`, و`grok-4-0709` إلى متغيراتها `*-fast`
  - تكون `tool_stream` مفعّلة افتراضيًا؛ اضبط
    `agents.defaults.models["xai/<model>"].params.tool_stream` إلى `false`
    لتعطيلها
- Mistral: ‏`mistral` ‏(`MISTRAL_API_KEY`)
- مثال النموذج: `mistral/mistral-large-latest`
- CLI: ‏`openclaw onboard --auth-choice mistral-api-key`
- Groq: ‏`groq` ‏(`GROQ_API_KEY`)
- Cerebras: ‏`cerebras` ‏(`CEREBRAS_API_KEY`)
  - تستخدم نماذج GLM على Cerebras المعرفين `zai-glm-4.7` و`zai-glm-4.6`.
  - Base URL المتوافق مع OpenAI: ‏`https://api.cerebras.ai/v1`.
- GitHub Copilot: ‏`github-copilot` ‏(`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- مثال نموذج Hugging Face Inference: ‏`huggingface/deepseek-ai/DeepSeek-R1`; وCLI: ‏`openclaw onboard --auth-choice huggingface-api-key`. راجع [Hugging Face (Inference)](/providers/huggingface).

## المزوّدون عبر `models.providers` ‏(مخصص/base URL)

استخدم `models.providers` ‏(أو `models.json`) لإضافة مزوّدين **مخصصين** أو
وكلاء متوافقين مع OpenAI/Anthropic.

تنشر كثير من المكونات الإضافية المضمّنة للمزوّدات أدناه فهرسًا افتراضيًا بالفعل.
استخدم إدخالات `models.providers.<id>` الصريحة فقط عندما تريد تجاوز
base URL أو الترويسات أو قائمة النماذج الافتراضية.

### Moonshot AI ‏(Kimi)

يأتي Moonshot كمكوّن إضافي مضمّن للمزوّد. استخدم المزوّد المضمّن
افتراضيًا، وأضف إدخال `models.providers.moonshot` صريحًا فقط عندما
تحتاج إلى تجاوز base URL أو البيانات الوصفية للنموذج:

- المزوّد: `moonshot`
- المصادقة: `MOONSHOT_API_KEY`
- مثال النموذج: `moonshot/kimi-k2.5`
- CLI: ‏`openclaw onboard --auth-choice moonshot-api-key` أو `openclaw onboard --auth-choice moonshot-api-key-cn`

معرّفات نماذج Kimi K2:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.5`
- `moonshot/kimi-k2-thinking`
- `moonshot/kimi-k2-thinking-turbo`
- `moonshot/kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }],
      },
    },
  },
}
```

### Kimi Coding

يستخدم Kimi Coding نقطة النهاية المتوافقة مع Anthropic الخاصة بـ Moonshot AI:

- المزوّد: `kimi`
- المصادقة: `KIMI_API_KEY`
- مثال النموذج: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

يظل `kimi/k2p5` القديم مقبولًا كمعرّف نموذج متوافق.

### Volcano Engine ‏(Doubao)

توفر Volcano Engine ‏(火山引擎) الوصول إلى Doubao ونماذج أخرى داخل الصين.

- المزوّد: `volcengine` ‏(للبرمجة: `volcengine-plan`)
- المصادقة: `VOLCANO_ENGINE_API_KEY`
- مثال النموذج: `volcengine-plan/ark-code-latest`
- CLI: ‏`openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

يعتمد الإعداد الأولي افتراضيًا على سطح البرمجة، لكن فهرس `volcengine/*`
العام يُسجَّل في الوقت نفسه.

في منتقيات النماذج ضمن الإعداد الأولي/configure، يفضّل خيار مصادقة Volcengine
كلًا من الصفوف `volcengine/*` و`volcengine-plan/*`. وإذا لم تكن هذه النماذج
محملة بعد، يعود OpenClaw إلى الفهرس غير المصفّى بدل عرض
منتقٍ فارغ محدد بالمزوّد.

النماذج المتاحة:

- `volcengine/doubao-seed-1-8-251228` ‏(Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` ‏(Kimi K2.5)
- `volcengine/glm-4-7-251222` ‏(GLM 4.7)
- `volcengine/deepseek-v3-2-251201` ‏(DeepSeek V3.2 128K)

نماذج البرمجة (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus ‏(دولي)

يوفر BytePlus ARK الوصول إلى النماذج نفسها التي توفرها Volcano Engine للمستخدمين الدوليين.

- المزوّد: `byteplus` ‏(للبرمجة: `byteplus-plan`)
- المصادقة: `BYTEPLUS_API_KEY`
- مثال النموذج: `byteplus-plan/ark-code-latest`
- CLI: ‏`openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

يعتمد الإعداد الأولي افتراضيًا على سطح البرمجة، لكن فهرس `byteplus/*`
العام يُسجَّل في الوقت نفسه.

في منتقيات النماذج ضمن الإعداد الأولي/configure، يفضّل خيار مصادقة BytePlus
كلًا من الصفوف `byteplus/*` و`byteplus-plan/*`. وإذا لم تكن هذه النماذج
محملة بعد، يعود OpenClaw إلى الفهرس غير المصفّى بدل عرض
منتقٍ فارغ محدد بالمزوّد.

النماذج المتاحة:

- `byteplus/seed-1-8-251228` ‏(Seed 1.8)
- `byteplus/kimi-k2-5-260127` ‏(Kimi K2.5)
- `byteplus/glm-4-7-251222` ‏(GLM 4.7)

نماذج البرمجة (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

يوفر Synthetic نماذج متوافقة مع Anthropic خلف المزوّد `synthetic`:

- المزوّد: `synthetic`
- المصادقة: `SYNTHETIC_API_KEY`
- مثال النموذج: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI: ‏`openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M2.5", name: "MiniMax M2.5" }],
      },
    },
  },
}
```

### MiniMax

يُضبط MiniMax عبر `models.providers` لأنه يستخدم نقاط نهاية مخصصة:

- MiniMax OAuth ‏(عالمي): `--auth-choice minimax-global-oauth`
- MiniMax OAuth ‏(CN): `--auth-choice minimax-cn-oauth`
- MiniMax API key ‏(عالمي): `--auth-choice minimax-global-api`
- MiniMax API key ‏(CN): `--auth-choice minimax-cn-api`
- المصادقة: `MINIMAX_API_KEY` لـ `minimax`؛ و`MINIMAX_OAUTH_TOKEN` أو
  `MINIMAX_API_KEY` لـ `minimax-portal`

راجع [/providers/minimax](/providers/minimax) لتفاصيل الإعداد وخيارات النماذج ومقتطفات الإعداد.

في مسار البث المتوافق مع Anthropic في MiniMax، يعطل OpenClaw التفكير
افتراضيًا ما لم تضبطه صراحةً، ويعيد `/fast on` كتابة
`MiniMax-M2.7` إلى `MiniMax-M2.7-highspeed`.

تقسيم الإمكانات المملوك للمكون الإضافي:

- تبقى القيم الافتراضية للنص/الدردشة على `minimax/MiniMax-M2.7`
- يكون توليد الصور عبر `minimax/image-01` أو `minimax-portal/image-01`
- يكون فهم الصور عبر `MiniMax-VL-01` المملوك للمكون الإضافي على مساري مصادقة MiniMax
- يبقى البحث على الويب على معرّف المزوّد `minimax`

### Ollama

يأتي Ollama كمكوّن إضافي مضمّن للمزوّد ويستخدم API الأصلية لـ Ollama:

- المزوّد: `ollama`
- المصادقة: غير مطلوبة (خادم محلي)
- مثال النموذج: `ollama/llama3.3`
- التثبيت: [https://ollama.com/download](https://ollama.com/download)

```bash
# ثبّت Ollama، ثم اسحب نموذجًا:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

يُكتشف Ollama محليًا عند `http://127.0.0.1:11434` عندما تشترك فيه عبر
`OLLAMA_API_KEY`، ويضيف المكون الإضافي المضمّن للمزوّد Ollama مباشرة إلى
`openclaw onboard` ومنتقي النماذج. راجع [/providers/ollama](/providers/ollama)
للحصول على تفاصيل الإعداد الأولي، ووضع السحابة/المحلي، والإعداد المخصص.

### vLLM

يأتي vLLM كمكوّن إضافي مضمّن للمزوّد لخوادم
OpenAI-compatible المحلية/المستضافة ذاتيًا:

- المزوّد: `vllm`
- المصادقة: اختيارية (بحسب خادمك)
- Base URL الافتراضي: `http://127.0.0.1:8000/v1`

للاشتراك في الاكتشاف التلقائي محليًا (أي قيمة تعمل إذا كان خادمك لا يفرض المصادقة):

```bash
export VLLM_API_KEY="vllm-local"
```

ثم اضبط نموذجًا (استبدله بأحد المعرّفات التي تعيدها `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

راجع [/providers/vllm](/providers/vllm) للتفاصيل.

### SGLang

يأتي SGLang كمكوّن إضافي مضمّن للمزوّد لخوادم
OpenAI-compatible السريعة المستضافة ذاتيًا:

- المزوّد: `sglang`
- المصادقة: اختيارية (بحسب خادمك)
- Base URL الافتراضي: `http://127.0.0.1:30000/v1`

للاشتراك في الاكتشاف التلقائي محليًا (أي قيمة تعمل إذا كان خادمك لا
يفرض المصادقة):

```bash
export SGLANG_API_KEY="sglang-local"
```

ثم اضبط نموذجًا (استبدله بأحد المعرّفات التي تعيدها `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

راجع [/providers/sglang](/providers/sglang) للتفاصيل.

### الوكلاء المحليون ‏(LM Studio، وvLLM، وLiteLLM، وغير ذلك)

مثال (متوافق مع OpenAI):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "LMSTUDIO_KEY",
        api: "openai-completions",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

ملاحظات:

- بالنسبة إلى المزوّدين المخصصين، تكون `reasoning` و`input` و`cost` و`contextWindow` و`maxTokens` اختيارية.
  وعند حذفها، يستخدم OpenClaw القيم الافتراضية التالية:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- الموصى به: اضبط قيمًا صريحة تطابق حدود وكيلك/نموذجك.
- بالنسبة إلى `api: "openai-completions"` على نقاط نهاية غير أصلية (أي `baseUrl` غير فارغة يكون مضيفها ليس `api.openai.com`)، يفرض OpenClaw القيمة `compat.supportsDeveloperRole: false` لتجنب أخطاء 400 الخاصة بالمزوّد عند عدم دعم أدوار `developer`.
- كما تتجاوز المسارات المتوافقة مع OpenAI بنمط proxy أيضًا تشكيل الطلبات الأصلية الخاصة بـ OpenAI فقط:
  لا `service_tier`، ولا Responses `store`، ولا تلميحات تخزين مؤقت للموجّه، ولا
  تشكيل حمولة توافق الاستدلال الخاصة بـ OpenAI، ولا ترويسات إسناد
  OpenClaw المخفية.
- إذا كانت `baseUrl` فارغة/غير موجودة، يحتفظ OpenClaw بسلوك OpenAI الافتراضي (الذي يُحل إلى `api.openai.com`).
- ولأسباب تتعلق بالسلامة، يُتجاوز حتى الضبط الصريح `compat.supportsDeveloperRole: true` على نقاط نهاية `openai-completions` غير الأصلية.

## أمثلة CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

راجع أيضًا: [/gateway/configuration](/gateway/configuration) للحصول على أمثلة إعداد كاملة.

## ذو صلة

- [النماذج](/concepts/models) — إعداد النماذج والأسماء البديلة
- [Model Failover](/concepts/model-failover) — سلاسل البدائل الاحتياطية وسلوك إعادة المحاولة
- [مرجع الإعداد](/gateway/configuration-reference#agent-defaults) — مفاتيح إعداد النماذج
- [المزوّدون](/providers) — أدلة الإعداد لكل مزوّد
