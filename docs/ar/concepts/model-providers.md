---
read_when:
    - أنت بحاجة إلى مرجع إعداد النماذج لكل موفر على حدة
    - أنت تريد إعدادات نموذجية أو أوامر CLI للإعداد الأولي لموفري النماذج
summary: نظرة عامة على موفري النماذج مع إعدادات نموذجية وتدفّقات CLI
title: موفرو النماذج
x-i18n:
    generated_at: "2026-04-06T03:08:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 15e4b82e07221018a723279d309e245bb4023bc06e64b3c910ef2cae3dfa2599
    source_path: concepts/model-providers.md
    workflow: 15
---

# موفرو النماذج

تغطي هذه الصفحة **موفري LLM/النماذج** (وليس قنوات الدردشة مثل WhatsApp/Telegram).
للاطلاع على قواعد اختيار النماذج، راجع [/concepts/models](/ar/concepts/models).

## قواعد سريعة

- تستخدم مراجع النماذج الصيغة `provider/model` (مثال: `opencode/claude-opus-4-6`).
- إذا قمت بتعيين `agents.defaults.models`، فستصبح قائمة السماح.
- مساعدات CLI: `openclaw onboard` و `openclaw models list` و `openclaw models set <provider/model>`.
- تم توثيق قواعد وقت التشغيل الاحتياطية، وفحوصات التهدئة، واستمرارية تجاوز الجلسة في
  [/concepts/model-failover](/ar/concepts/model-failover).
- `models.providers.*.models[].contextWindow` هي بيانات تعريف النموذج الأصلية؛
  أما `models.providers.*.models[].contextTokens` فهي الحد الفعّال في وقت التشغيل.
- يمكن لإضافات الموفر إدخال كتالوجات النماذج عبر `registerProvider({ catalog })`؛
  ويقوم OpenClaw بدمج هذا الناتج في `models.providers` قبل كتابة
  `models.json`.
- يمكن لبيانات تعريف الموفر التصريحية أن تعلن عن `providerAuthEnvVars` بحيث لا تحتاج
  فحوصات المصادقة العامة المعتمدة على المتغيرات البيئية إلى تحميل وقت تشغيل الإضافة. أصبحت خريطة
  متغيرات البيئة الأساسية المتبقية الآن مخصصة فقط للموفرين غير الإضافيين/الأساسيين وبعض حالات
  الأولوية العامة مثل الإعداد الأولي القائم على مفتاح API أولًا لـ Anthropic.
- يمكن لإضافات الموفر أيضًا امتلاك سلوك وقت تشغيل الموفر عبر
  `normalizeModelId` و `normalizeTransport` و `normalizeConfig` و
  `applyNativeStreamingUsageCompat` و `resolveConfigApiKey` و
  `resolveSyntheticAuth` و `shouldDeferSyntheticProfileAuth` و
  `resolveDynamicModel` و `prepareDynamicModel` و
  `normalizeResolvedModel` و `contributeResolvedModelCompat` و
  `capabilities` و `normalizeToolSchemas` و
  `inspectToolSchemas` و `resolveReasoningOutputMode` و
  `prepareExtraParams` و `createStreamFn` و `wrapStreamFn` و
  `resolveTransportTurnState` و `resolveWebSocketSessionPolicy` و
  `createEmbeddingProvider` و `formatApiKey` و `refreshOAuth` و
  `buildAuthDoctorHint` و
  `matchesContextOverflowError` و `classifyFailoverReason` و
  `isCacheTtlEligible` و `buildMissingAuthMessage` و `suppressBuiltInModel` و
  `augmentModelCatalog` و `isBinaryThinking` و `supportsXHighThinking` و
  `resolveDefaultThinkingLevel` و `applyConfigDefaults` و `isModernModelRef` و
  `prepareRuntimeAuth` و `resolveUsageAuth` و `fetchUsageSnapshot` و
  `onModelSelected`.
- ملاحظة: إن `capabilities` الخاصة بوقت تشغيل الموفر هي بيانات تعريف مشتركة للمشغّل
  (عائلة الموفر، وخصائص النصوص/الأدوات، وتلميحات النقل/الذاكرة المؤقتة). وهي ليست
  مماثلة لـ [نموذج القدرات العام](/ar/plugins/architecture#public-capability-model)
  الذي يصف ما الذي تسجله الإضافة (استدلال نصي، وكلام، وما إلى ذلك).

## السلوك المملوك لإضافة الموفر

يمكن لإضافات الموفر الآن امتلاك معظم المنطق الخاص بكل موفر بينما يحتفظ OpenClaw
بحلقة الاستدلال العامة.

التقسيم المعتاد:

- `auth[].run` / `auth[].runNonInteractive`: يمتلك الموفر تدفقات الإعداد الأولي/تسجيل الدخول
  لـ `openclaw onboard` و `openclaw models auth` والإعداد غير التفاعلي
- `wizard.setup` / `wizard.modelPicker`: يمتلك الموفر تسميات خيارات المصادقة،
  والأسماء المستعارة القديمة، وتلميحات قائمة السماح في الإعداد الأولي، وإدخالات الإعداد في منتقيات الإعداد الأولي/النموذج
- `catalog`: يظهر الموفر في `models.providers`
- `normalizeModelId`: يطبّع الموفر معرّفات النماذج القديمة/المعاينة قبل
  البحث أو التوحيد القياسي
- `normalizeTransport`: يطبّع الموفر `api` / `baseUrl` لعائلة النقل
  قبل التجميع العام للنموذج؛ يتحقق OpenClaw أولًا من الموفر المطابق،
  ثم من إضافات الموفر الأخرى القادرة على الخطافات إلى أن يغيّر أحدها
  النقل فعليًا
- `normalizeConfig`: يطبّع الموفر إعداد `models.providers.<id>` قبل أن
  يستخدمه وقت التشغيل؛ يتحقق OpenClaw أولًا من الموفر المطابق، ثم من إضافات
  الموفر الأخرى القادرة على الخطافات إلى أن يغيّر أحدها الإعداد فعليًا. إذا لم تقم
  أي خطافة موفر بإعادة كتابة الإعداد، فإن مساعدات Google-family المضمّنة لا تزال
  تطبّع إدخالات موفر Google المدعومة.
- `applyNativeStreamingUsageCompat`: يطبق الموفر إعادة كتابة توافق استخدام البث الأصلي الموجّهة بنقطة النهاية لموفري الإعدادات
- `resolveConfigApiKey`: يحل الموفر مصادقة علامات متغيرات البيئة لموفري الإعدادات
  من دون فرض تحميل مصادقة وقت التشغيل كاملة. كما يمتلك `amazon-bedrock`
  محلّلًا مدمجًا لعلامات متغيرات بيئة AWS هنا، رغم أن مصادقة وقت تشغيل Bedrock
  تستخدم سلسلة AWS SDK الافتراضية.
- `resolveSyntheticAuth`: يمكن للموفر إظهار توافر المصادقة المحلية/المستضافة ذاتيًا أو غيرها من
  المصادقات المعتمدة على الإعدادات من دون تخزين أسرار نصية صريحة
- `shouldDeferSyntheticProfileAuth`: يمكن للموفر وضع علامة على العناصر النائبة للمصادقة الاصطناعية المخزّنة في الملف الشخصي
  باعتبارها أقل أولوية من المصادقة المعتمدة على البيئة/الإعدادات
- `resolveDynamicModel`: يقبل الموفر معرّفات نماذج غير موجودة بعد في
  الكتالوج الثابت المحلي
- `prepareDynamicModel`: يحتاج الموفر إلى تحديث بيانات التعريف قبل إعادة المحاولة
  في الحل الديناميكي
- `normalizeResolvedModel`: يحتاج الموفر إلى إعادة كتابة النقل أو عنوان URL الأساسي
- `contributeResolvedModelCompat`: يساهم الموفر بعلامات التوافق لنماذج
  مورده حتى عندما تصل عبر نقل متوافق آخر
- `capabilities`: ينشر الموفر خصائص النصوص/الأدوات/عائلة الموفر
- `normalizeToolSchemas`: ينظّف الموفر مخططات الأدوات قبل أن يراها المشغّل
  المضمّن
- `inspectToolSchemas`: يعرض الموفر تحذيرات المخطط الخاصة بالنقل
  بعد التطبيع
- `resolveReasoningOutputMode`: يختار الموفر عقود مخرجات الاستدلال
  الأصلية أو المعلّمة
- `prepareExtraParams`: يضبط الموفر افتراضيًا أو يطبّع معاملات الطلب لكل نموذج
- `createStreamFn`: يستبدل الموفر مسار البث العادي بنقل
  مخصص بالكامل
- `wrapStreamFn`: يطبّق الموفر أغلفة توافق الرؤوس/النص/النموذج على الطلب
- `resolveTransportTurnState`: يوفّر الموفر رؤوس أو بيانات تعريف
  أصلية لكل دور
- `resolveWebSocketSessionPolicy`: يوفّر الموفر رؤوس جلسة WebSocket أصلية
  أو سياسة تهدئة الجلسة
- `createEmbeddingProvider`: يمتلك الموفر سلوك تضمين الذاكرة عندما
  يكون من الأنسب أن يتبع إضافة الموفر بدلًا من لوحة التبديل الأساسية للتضمين
- `formatApiKey`: ينسّق الموفر ملفات تعريف المصادقة المخزنة إلى
  سلسلة `apiKey` الخاصة بوقت التشغيل التي يتوقعها النقل
- `refreshOAuth`: يمتلك الموفر عملية تحديث OAuth عندما لا تكون محدِّثات
  `pi-ai` المشتركة كافية
- `buildAuthDoctorHint`: يضيف الموفر إرشادات إصلاح عندما يفشل تحديث OAuth
- `matchesContextOverflowError`: يتعرف الموفر على أخطاء تجاوز نافذة السياق
  الخاصة به التي قد تفوتها الاستدلالات العامة
- `classifyFailoverReason`: يربط الموفر الأخطاء الخام الخاصة به على مستوى النقل/API
  بأسباب التبديل الاحتياطي مثل حد المعدل أو الحمل الزائد
- `isCacheTtlEligible`: يحدد الموفر معرّفات النماذج العليا التي تدعم مدة TTL لذاكرة الطلب المؤقتة
- `buildMissingAuthMessage`: يستبدل الموفر خطأ مخزن المصادقة العام
  بتلميح استرداد خاص بالموفر
- `suppressBuiltInModel`: يخفي الموفر الصفوف العليا القديمة ويمكنه إرجاع
  خطأ مملوك للمورّد عند فشل الحل المباشر
- `augmentModelCatalog`: يضيف الموفر صفوف كتالوج اصطناعية/نهائية بعد
  الاكتشاف ودمج الإعدادات
- `isBinaryThinking`: يمتلك الموفر تجربة المستخدم الخاصة بالتفكير الثنائي تشغيل/إيقاف
- `supportsXHighThinking`: يدرج الموفر النماذج المحددة ضمن `xhigh`
- `resolveDefaultThinkingLevel`: يمتلك الموفر سياسة `/think` الافتراضية لعائلة
  نموذجية
- `applyConfigDefaults`: يطبق الموفر الإعدادات العامة الافتراضية الخاصة به
  أثناء إنشاء الإعدادات وفقًا لوضع المصادقة أو البيئة أو عائلة النموذج
- `isModernModelRef`: يمتلك الموفر مطابقة النموذج المفضل في اختبارات live/smoke
- `prepareRuntimeAuth`: يحوّل الموفر بيانات اعتماد مهيّأة إلى رمز
  وقت تشغيل قصير العمر
- `resolveUsageAuth`: يحل الموفر بيانات اعتماد الاستخدام/الحصة الخاصة بـ `/usage`
  والأسطح ذات الصلة بالحالة/التقارير
- `fetchUsageSnapshot`: يمتلك الموفر جلب/تحليل نقطة نهاية الاستخدام بينما
  لا يزال اللب يمتلك الغلاف الملخّص والتنسيق
- `onModelSelected`: ينفذ الموفر تأثيرات جانبية بعد الاختيار مثل
  القياس عن بُعد أو حفظ الجلسات المملوك للموفر

أمثلة مضمّنة حاليًا:

- `anthropic`: احتياط توافق أمامي لـ Claude 4.6، وتلميحات إصلاح المصادقة، وجلب
  نقطة نهاية الاستخدام، وبيانات تعريف TTL للذاكرة المؤقتة/عائلة الموفر، وإعدادات عامة
  افتراضية تراعي المصادقة
- `amazon-bedrock`: مطابقة تجاوز السياق المملوكة للموفر وتصنيف
  سبب التبديل الاحتياطي لأخطاء Bedrock الخاصة بالحدّ/عدم الجاهزية، بالإضافة إلى
  عائلة إعادة التشغيل المشتركة `anthropic-by-model` لحواجز سياسة إعادة التشغيل الخاصة بـ Claude فقط
  على حركة Anthropic
- `anthropic-vertex`: حواجز سياسة إعادة التشغيل الخاصة بـ Claude فقط على حركة
  رسائل Anthropic
- `openrouter`: معرّفات نماذج تمريرية، وأغلفة الطلبات، وتلميحات قدرات الموفر،
  وتنقية توقيع التفكير لـ Gemini على حركة Gemini الوكيلة، وحقن الاستدلال الوكيلي
  عبر عائلة البث `openrouter-thinking`، وتمرير بيانات تعريف التوجيه،
  وسياسة TTL للذاكرة المؤقتة
- `github-copilot`: الإعداد الأولي/تسجيل دخول الجهاز، واحتياط نموذج توافق أمامي،
  وتلميحات نص Claude-thinking، وتبادل رمز وقت التشغيل، وجلب نقطة
  نهاية الاستخدام
- `openai`: احتياط توافق أمامي لـ GPT-5.4، وتطبيع نقل OpenAI المباشر،
  وتلميحات فقدان المصادقة المدركة لـ Codex، وإخفاء Spark، وصفوف كتالوج
  OpenAI/Codex اصطناعية، وسياسة التفكير/النموذج الحي، وتطبيع الأسماء المستعارة
  لرموز الاستخدام (`input` / `output` و `prompt` / `completion`)، وعائلة البث
  المشتركة `openai-responses-defaults` لأغلفة OpenAI/Codex الأصلية،
  وبيانات تعريف عائلة الموفر، وتسجيل موفر توليد الصور المضمّن لـ
  `gpt-image-1`، وتسجيل موفر توليد الفيديو المضمّن لـ
  `sora-2`
- `google`: احتياط توافق أمامي لـ Gemini 3.1، والتحقق الأصلي من إعادة تشغيل Gemini،
  وتنقية إعادة التشغيل في bootstrap، ووضع مخرجات الاستدلال المعلّم،
  ومطابقة النماذج الحديثة، وتسجيل موفر توليد الصور المضمّن لنماذج
  معاينة صور Gemini، وتسجيل موفر توليد الفيديو المضمّن لنماذج Veo
- `moonshot`: نقل مشترك، وتطبيع حمولة التفكير المملوك للإضافة
- `kilocode`: نقل مشترك، ورؤوس طلبات مملوكة للإضافة، وتطبيع حمولة الاستدلال،
  وتنقية توقيع التفكير لـ Gemini الوكيل، وسياسة TTL للذاكرة المؤقتة
- `zai`: احتياط توافق أمامي لـ GLM-5، وافتراضات `tool_stream`، وسياسة TTL للذاكرة المؤقتة،
  وسياسة التفكير الثنائي/النموذج الحي، ومصادقة الاستخدام + جلب الحصة؛
  يتم توليف المعرفات غير المعروفة `glm-5*` من القالب المضمّن `glm-4.7`
- `xai`: تطبيع نقل Responses الأصلي، وإعادة كتابة الاسم المستعار `/fast` لـ
  المتغيرات السريعة من Grok، و `tool_stream` الافتراضي، وتنظيف مخطط الأدوات /
  حمولة الاستدلال الخاص بـ xAI، وتسجيل موفر توليد الفيديو المضمّن لـ
  `grok-imagine-video`
- `mistral`: بيانات تعريف القدرات المملوكة للإضافة
- `opencode` و `opencode-go`: بيانات تعريف القدرات المملوكة للإضافة بالإضافة
  إلى تنقية توقيع التفكير لـ Gemini الوكيل
- `alibaba`: كتالوج توليد فيديو مملوك للإضافة لمراجع نماذج Wan المباشرة
  مثل `alibaba/wan2.6-t2v`
- `byteplus`: كتالوجات مملوكة للإضافة بالإضافة إلى تسجيل موفر توليد الفيديو المضمّن
  لنماذج Seedance من نص إلى فيديو/من صورة إلى فيديو
- `fal`: تسجيل موفر توليد فيديو مضمّن لموفري
  توليد الصور الخارجيين المستضافين لنماذج صور FLUX بالإضافة إلى تسجيل موفر
  توليد الفيديو المضمّن لنماذج الفيديو الخارجية المستضافة
- `cloudflare-ai-gateway` و `huggingface` و `kimi` و `nvidia` و `qianfan` و
  `stepfun` و `synthetic` و `venice` و `vercel-ai-gateway` و `volcengine`:
  كتالوجات مملوكة للإضافة فقط
- `qwen`: كتالوجات مملوكة للإضافة للنماذج النصية بالإضافة إلى تسجيلات
  مشتركة لموفري فهم الوسائط وتوليد الفيديو لأسطحه متعددة الوسائط؛ يستخدم
  توليد الفيديو في Qwen نقاط نهاية الفيديو Standard DashScope مع نماذج Wan
  المضمّنة مثل `wan2.6-t2v` و `wan2.7-r2v`
- `runway`: تسجيل موفر توليد فيديو مملوك للإضافة لنماذج Runway الأصلية
  القائمة على المهام مثل `gen4.5`
- `minimax`: كتالوجات مملوكة للإضافة، وتسجيل موفر توليد الفيديو المضمّن
  لنماذج فيديو Hailuo، وتسجيل موفر توليد الصور المضمّن لـ
  `image-01`، واختيار سياسة إعادة تشغيل هجينة بين Anthropic/OpenAI،
  ومنطق مصادقة/لقطة الاستخدام
- `together`: كتالوجات مملوكة للإضافة بالإضافة إلى تسجيل موفر توليد الفيديو المضمّن
  لنماذج فيديو Wan
- `xiaomi`: كتالوجات مملوكة للإضافة بالإضافة إلى منطق مصادقة/لقطة الاستخدام

تمتلك الإضافة المضمّنة `openai` الآن معرّفي الموفر كليهما: `openai` و
`openai-codex`.

يغطي ذلك الموفّرين الذين ما زالوا يناسبون وسائل النقل العادية في OpenClaw. أما الموفر
الذي يحتاج إلى منفّذ طلبات مخصص بالكامل فهو سطح امتداد آخر أعمق.

## تدوير مفاتيح API

- يدعم التدوير العام للمفاتيح لموفرين محددين.
- قم بتهيئة مفاتيح متعددة عبر:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (تجاوز حي واحد، أعلى أولوية)
  - `<PROVIDER>_API_KEYS` (قائمة مفصولة بفواصل أو فاصلات منقوطة)
  - `<PROVIDER>_API_KEY` (المفتاح الأساسي)
  - `<PROVIDER>_API_KEY_*` (قائمة مرقمة، مثل `<PROVIDER>_API_KEY_1`)
- بالنسبة إلى موفري Google، يتم أيضًا تضمين `GOOGLE_API_KEY` كحل احتياطي.
- يحافظ ترتيب اختيار المفتاح على الأولوية ويزيل القيم المكررة.
- تتم إعادة محاولة الطلبات باستخدام المفتاح التالي فقط عند استجابات حد المعدل (على
  سبيل المثال `429` أو `rate_limit` أو `quota` أو `resource exhausted` أو `Too many
concurrent requests` أو `ThrottlingException` أو `concurrency limit reached` أو
  `workers_ai ... quota limit exceeded` أو رسائل حد الاستخدام الدورية).
- تفشل حالات الفشل غير المرتبطة بحد المعدل مباشرة؛ ولا تتم محاولة تدوير المفاتيح.
- عندما تفشل جميع المفاتيح المرشحة، يتم إرجاع الخطأ النهائي من آخر محاولة.

## الموفّرون المدمجون (كتالوج pi-ai)

يشحن OpenClaw مع كتالوج pi‑ai. لا تتطلب هذه الموفّرات أي إعداد
`models.providers`؛ فقط اضبط المصادقة واختر نموذجًا.

### OpenAI

- الموفر: `openai`
- المصادقة: `OPENAI_API_KEY`
- التدوير الاختياري: `OPENAI_API_KEYS` و `OPENAI_API_KEY_1` و `OPENAI_API_KEY_2`، بالإضافة إلى `OPENCLAW_LIVE_OPENAI_KEY` (تجاوز واحد)
- أمثلة على النماذج: `openai/gpt-5.4` و `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- النقل الافتراضي هو `auto` (WebSocket أولًا، مع SSE احتياطيًا)
- قم بالتجاوز لكل نموذج عبر `agents.defaults.models["openai/<model>"].params.transport` (`"sse"` أو `"websocket"` أو `"auto"`)
- يتم تفعيل الإحماء المسبق لـ OpenAI Responses WebSocket افتراضيًا عبر `params.openaiWsWarmup` (`true`/`false`)
- يمكن تفعيل المعالجة ذات الأولوية في OpenAI عبر `agents.defaults.models["openai/<model>"].params.serviceTier`
- يقوم `/fast` و `params.fastMode` بربط طلبات Responses المباشرة `openai/*` إلى `service_tier=priority` على `api.openai.com`
- استخدم `params.serviceTier` عندما تريد طبقة صريحة بدلًا من مفتاح `/fast` المشترك
- تُطبَّق رؤوس الإسناد المخفية الخاصة بـ OpenClaw (`originator` و `version` و
  `User-Agent`) فقط على حركة OpenAI الأصلية إلى `api.openai.com`، وليس على
  الوكلاء العامة المتوافقة مع OpenAI
- تحتفظ مسارات OpenAI الأصلية أيضًا بـ Responses `store` وتلميحات الذاكرة المؤقتة للطلبات
  وتشكيل الحمولة المتوافق مع استدلال OpenAI؛ أما مسارات الوكيل فلا تفعل ذلك
- تم إخفاء `openai/gpt-5.3-codex-spark` عمدًا في OpenClaw لأن واجهة OpenAI الحية ترفضه؛ ويتم التعامل مع Spark على أنه خاص بـ Codex فقط

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- الموفر: `anthropic`
- المصادقة: `ANTHROPIC_API_KEY`
- التدوير الاختياري: `ANTHROPIC_API_KEYS` و `ANTHROPIC_API_KEY_1` و `ANTHROPIC_API_KEY_2`، بالإضافة إلى `OPENCLAW_LIVE_ANTHROPIC_KEY` (تجاوز واحد)
- مثال على النموذج: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- تدعم طلبات Anthropic العامة المباشرة مفتاح `/fast` المشترك و `params.fastMode`، بما في ذلك الحركة الموثقة بمفتاح API وOAuth المرسلة إلى `api.anthropic.com`؛ ويقوم OpenClaw بربط ذلك إلى `service_tier` الخاصة بـ Anthropic (`auto` مقابل `standard_only`)
- ملاحظة الفوترة: بالنسبة إلى Anthropic في OpenClaw، فإن الانقسام العملي هو **مفتاح API** أو **اشتراك Claude مع Extra Usage**. وقد أخبرت Anthropic مستخدمي OpenClaw في **4 أبريل 2026 الساعة 12:00 ظهرًا بتوقيت PT / 8:00 مساءً بتوقيت BST** بأن مسار تسجيل دخول Claude في **OpenClaw** يُحتسب على أنه استخدام لحزمة طرف ثالث ويتطلب **Extra Usage** تتم فوترته بشكل منفصل عن الاشتراك. كما تُظهر إعادة الإنتاج المحلية لدينا أن سلسلة الطلب التعريفية الخاصة بـ OpenClaw لا تظهر على مسار Anthropic SDK + مفتاح API.
- أصبح رمز إعداد Anthropic متاحًا مرة أخرى كمسار OpenClaw قديم/يدوي. استخدمه مع توقع أن Anthropic أخبرت مستخدمي OpenClaw بأن هذا المسار يتطلب **Extra Usage**.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- الموفر: `openai-codex`
- المصادقة: OAuth (ChatGPT)
- مثال على النموذج: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex` أو `openclaw models auth login --provider openai-codex`
- النقل الافتراضي هو `auto` (WebSocket أولًا، مع SSE احتياطيًا)
- قم بالتجاوز لكل نموذج عبر `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"` أو `"websocket"` أو `"auto"`)
- يتم أيضًا تمرير `params.serviceTier` على طلبات Codex Responses الأصلية (`chatgpt.com/backend-api`)
- لا يتم إرفاق رؤوس الإسناد المخفية الخاصة بـ OpenClaw (`originator` و `version` و
  `User-Agent`) إلا على حركة Codex الأصلية إلى
  `chatgpt.com/backend-api`، وليس على الوكلاء العامة المتوافقة مع OpenAI
- يشارك نفس مفتاح `/fast` وإعداد `params.fastMode` كما في `openai/*` المباشر؛ ويقوم OpenClaw بربط ذلك إلى `service_tier=priority`
- يظل `openai-codex/gpt-5.3-codex-spark` متاحًا عندما يعرضه كتالوج Codex OAuth؛ ويتوقف ذلك على الاستحقاق
- يحتفظ `openai-codex/gpt-5.4` بالقيمة الأصلية `contextWindow = 1050000` وبحد وقت تشغيل افتراضي `contextTokens = 272000`؛ ويمكنك تجاوز حد وقت التشغيل عبر `models.providers.openai-codex.models[].contextTokens`
- ملاحظة السياسة: يتم دعم OpenAI Codex OAuth صراحةً للأدوات/سير العمل الخارجية مثل OpenClaw.

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

### خيارات أخرى مستضافة على نمط الاشتراك

- [Qwen Cloud](/ar/providers/qwen): سطح موفر Qwen Cloud بالإضافة إلى ربط نقاط نهاية Alibaba DashScope وCoding Plan
- [MiniMax](/ar/providers/minimax): وصول MiniMax Coding Plan عبر OAuth أو مفتاح API
- [GLM Models](/ar/providers/glm): Z.AI Coding Plan أو نقاط نهاية API العامة

### OpenCode

- المصادقة: `OPENCODE_API_KEY` (أو `OPENCODE_ZEN_API_KEY`)
- موفر وقت تشغيل Zen: `opencode`
- موفر وقت تشغيل Go: `opencode-go`
- أمثلة على النماذج: `opencode/claude-opus-4-6` و `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen` أو `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (مفتاح API)

- الموفر: `google`
- المصادقة: `GEMINI_API_KEY`
- التدوير الاختياري: `GEMINI_API_KEYS` و `GEMINI_API_KEY_1` و `GEMINI_API_KEY_2` واحتياطي `GOOGLE_API_KEY` و `OPENCLAW_LIVE_GEMINI_KEY` (تجاوز واحد)
- أمثلة على النماذج: `google/gemini-3.1-pro-preview` و `google/gemini-3-flash-preview`
- التوافق: يتم تطبيع إعداد OpenClaw القديم الذي يستخدم `google/gemini-3.1-flash-preview` إلى `google/gemini-3-flash-preview`
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- تقبل تشغيلات Gemini المباشرة أيضًا `agents.defaults.models["google/<model>"].params.cachedContent`
  (أو `cached_content` القديم) لتمرير معرّف
  `cachedContents/...` أصلي خاص بالموفر؛ وتظهر إصابات ذاكرة Gemini المؤقتة على هيئة OpenClaw `cacheRead`

### Google Vertex

- الموفر: `google-vertex`
- المصادقة: gcloud ADC
  - يتم تحليل ردود JSON الخاصة بـ Gemini CLI من `response`؛ ويعود الاستخدام احتياطيًا إلى
    `stats`، مع تطبيع `stats.cached` إلى OpenClaw `cacheRead`.

### Z.AI (GLM)

- الموفر: `zai`
- المصادقة: `ZAI_API_KEY`
- مثال على النموذج: `zai/glm-5`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - الأسماء المستعارة: يتم تطبيع `z.ai/*` و `z-ai/*` إلى `zai/*`
  - يكتشف `zai-api-key` نقطة نهاية Z.AI المطابقة تلقائيًا؛ بينما تفرض `zai-coding-global` و `zai-coding-cn` و `zai-global` و `zai-cn` سطحًا محددًا

### Vercel AI Gateway

- الموفر: `vercel-ai-gateway`
- المصادقة: `AI_GATEWAY_API_KEY`
- مثال على النموذج: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- الموفر: `kilocode`
- المصادقة: `KILOCODE_API_KEY`
- مثال على النموذج: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- عنوان URL الأساسي: `https://api.kilo.ai/api/gateway/`
- يشحن كتالوج احتياطي ثابت بـ `kilocode/kilo/auto`؛ ويمكن لاكتشاف
  `https://api.kilo.ai/api/gateway/models` الحي توسيع كتالوج وقت التشغيل أكثر.
- إن التوجيه الأعلى الدقيق خلف `kilocode/kilo/auto` مملوك لـ Kilo Gateway،
  وليس مضمّنًا بشكل ثابت في OpenClaw.

راجع [/providers/kilocode](/ar/providers/kilocode) للحصول على تفاصيل الإعداد.

### إضافات الموفّر المضمّنة الأخرى

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- مثال على النموذج: `openrouter/auto`
- يطبق OpenClaw رؤوس إسناد التطبيق الموثقة الخاصة بـ OpenRouter فقط عندما
  يستهدف الطلب فعليًا `openrouter.ai`
- كما يتم تقييد علامات `cache_control` الخاصة بـ Anthropic والمميزة لـ OpenRouter
  على المسارات المتحقق منها الخاصة بـ OpenRouter، وليس على عناوين URL الوكيلة العشوائية
- يظل OpenRouter على مسار الوكيل المتوافق مع OpenAI، لذا لا يتم تمرير
  تشكيل الطلب الأصلي الخاص بـ OpenAI فقط (`serviceTier` و Responses `store` و
  تلميحات ذاكرة الطلب المؤقتة وحمولات التوافق مع استدلال OpenAI)
- تحتفظ مراجع OpenRouter المبنية على Gemini فقط بتنقية توقيع التفكير لـ Gemini الوكيل؛
  بينما يظل التحقق الأصلي من إعادة تشغيل Gemini وإعادة كتابة bootstrap معطلين
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- مثال على النموذج: `kilocode/kilo/auto`
- تحتفظ مراجع Kilo المبنية على Gemini بمسار تنقية توقيع التفكير لـ Gemini الوكيل نفسه؛
  أما `kilocode/kilo/auto` والتلميحات الأخرى التي لا تدعم الاستدلال الوكيلي
  فتتجاوز حقن الاستدلال الوكيلي
- MiniMax: `minimax` (مفتاح API) و `minimax-portal` (OAuth)
- المصادقة: `MINIMAX_API_KEY` لـ `minimax`؛ و `MINIMAX_OAUTH_TOKEN` أو `MINIMAX_API_KEY` لـ `minimax-portal`
- مثال على النموذج: `minimax/MiniMax-M2.7` أو `minimax-portal/MiniMax-M2.7`
- يكتب إعداد MiniMax/مفتاح API تعريفات صريحة لنموذج M2.7 مع
  `input: ["text", "image"]`؛ بينما يبقي كتالوج الموفر المضمّن مراجع الدردشة
  نصية فقط إلى أن يتم إنشاء إعداد ذلك الموفر
- Moonshot: `moonshot` (`MOONSHOT_API_KEY`)
- مثال على النموذج: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi` (`KIMI_API_KEY` أو `KIMICODE_API_KEY`)
- مثال على النموذج: `kimi/kimi-code`
- Qianfan: `qianfan` (`QIANFAN_API_KEY`)
- مثال على النموذج: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen` (`QWEN_API_KEY` أو `MODELSTUDIO_API_KEY` أو `DASHSCOPE_API_KEY`)
- مثال على النموذج: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia` (`NVIDIA_API_KEY`)
- مثال على النموذج: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- أمثلة على النماذج: `stepfun/step-3.5-flash` و `stepfun-plan/step-3.5-flash-2603`
- Together: `together` (`TOGETHER_API_KEY`)
- مثال على النموذج: `together/moonshotai/Kimi-K2.5`
- Venice: `venice` (`VENICE_API_KEY`)
- Xiaomi: `xiaomi` (`XIAOMI_API_KEY`)
- مثال على النموذج: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: `huggingface` (`HUGGINGFACE_HUB_TOKEN` أو `HF_TOKEN`)
- Cloudflare AI Gateway: `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- مثال على النموذج: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus` (`BYTEPLUS_API_KEY`)
- مثال على النموذج: `byteplus-plan/ark-code-latest`
- xAI: `xai` (`XAI_API_KEY`)
  - تستخدم طلبات xAI الأصلية المضمّنة مسار xAI Responses
  - يعيد `/fast` أو `params.fastMode: true` كتابة `grok-3` و `grok-3-mini` و
    `grok-4` و `grok-4-0709` إلى متغيراتها `*-fast`
  - يتم تفعيل `tool_stream` افتراضيًا؛ اضبط
    `agents.defaults.models["xai/<model>"].params.tool_stream` على `false` من أجل
    تعطيله
- Mistral: `mistral` (`MISTRAL_API_KEY`)
- مثال على النموذج: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq` (`GROQ_API_KEY`)
- Cerebras: `cerebras` (`CEREBRAS_API_KEY`)
  - تستخدم نماذج GLM على Cerebras المعرّفات `zai-glm-4.7` و `zai-glm-4.6`.
  - عنوان URL الأساسي المتوافق مع OpenAI: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- مثال على نموذج Hugging Face Inference: `huggingface/deepseek-ai/DeepSeek-R1`؛ CLI: `openclaw onboard --auth-choice huggingface-api-key`. راجع [Hugging Face (Inference)](/ar/providers/huggingface).

## الموفّرون عبر `models.providers` (مخصص/عنوان URL أساسي)

استخدم `models.providers` (أو `models.json`) لإضافة **موفّرين مخصصين** أو
وكلاء متوافقين مع OpenAI/Anthropic.

تنشر العديد من إضافات الموفّر المضمّنة أدناه كتالوجًا افتراضيًا بالفعل.
استخدم إدخالات `models.providers.<id>` الصريحة فقط عندما تريد تجاوز
عنوان URL الأساسي أو الرؤوس أو قائمة النماذج الافتراضية.

### Moonshot AI (Kimi)

يشحن Moonshot كإضافة موفر مضمّنة. استخدم الموفر المدمج افتراضيًا،
وأضف إدخال `models.providers.moonshot` صريحًا فقط عندما
تحتاج إلى تجاوز عنوان URL الأساسي أو بيانات تعريف النموذج:

- الموفر: `moonshot`
- المصادقة: `MOONSHOT_API_KEY`
- مثال على النموذج: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` أو `openclaw onboard --auth-choice moonshot-api-key-cn`

معرّفات نموذج Kimi K2:

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

- الموفر: `kimi`
- المصادقة: `KIMI_API_KEY`
- مثال على النموذج: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

لا يزال `kimi/k2p5` القديم مقبولًا كمعرّف نموذج للتوافق.

### Volcano Engine (Doubao)

يوفر Volcano Engine (火山引擎) الوصول إلى Doubao ونماذج أخرى داخل الصين.

- الموفر: `volcengine` (البرمجة: `volcengine-plan`)
- المصادقة: `VOLCANO_ENGINE_API_KEY`
- مثال على النموذج: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

يعتمد الإعداد الأولي افتراضيًا على سطح البرمجة، لكن يتم تسجيل الكتالوج العام `volcengine/*`
في الوقت نفسه.

في منتقيات الإعداد الأولي/تكوين النموذج، يفضّل خيار مصادقة Volcengine كلا من
الصفوف `volcengine/*` و `volcengine-plan/*`. وإذا لم تكن هذه النماذج قد حُمّلت بعد،
فإن OpenClaw يعود إلى الكتالوج غير المفلتر بدلًا من إظهار منتقٍ فارغ
مقيّد بالموفر.

النماذج المتاحة:

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

نماذج البرمجة (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (دولي)

يوفر BytePlus ARK الوصول إلى النماذج نفسها التي يوفرها Volcano Engine للمستخدمين الدوليين.

- الموفر: `byteplus` (البرمجة: `byteplus-plan`)
- المصادقة: `BYTEPLUS_API_KEY`
- مثال على النموذج: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

يعتمد الإعداد الأولي افتراضيًا على سطح البرمجة، لكن يتم تسجيل الكتالوج العام `byteplus/*`
في الوقت نفسه.

في منتقيات الإعداد الأولي/تكوين النموذج، يفضّل خيار مصادقة BytePlus كلا من
الصفوف `byteplus/*` و `byteplus-plan/*`. وإذا لم تكن هذه النماذج قد حُمّلت بعد،
فإن OpenClaw يعود إلى الكتالوج غير المفلتر بدلًا من إظهار منتقٍ فارغ
مقيّد بالموفر.

النماذج المتاحة:

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

نماذج البرمجة (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

يوفر Synthetic نماذج متوافقة مع Anthropic خلف الموفر `synthetic`:

- الموفر: `synthetic`
- المصادقة: `SYNTHETIC_API_KEY`
- مثال على النموذج: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

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

يتم تكوين MiniMax عبر `models.providers` لأنه يستخدم نقاط نهاية مخصصة:

- MiniMax OAuth (عالمي): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (الصين): `--auth-choice minimax-cn-oauth`
- MiniMax مفتاح API (عالمي): `--auth-choice minimax-global-api`
- MiniMax مفتاح API (الصين): `--auth-choice minimax-cn-api`
- المصادقة: `MINIMAX_API_KEY` لـ `minimax`؛ و `MINIMAX_OAUTH_TOKEN` أو
  `MINIMAX_API_KEY` لـ `minimax-portal`

راجع [/providers/minimax](/ar/providers/minimax) للحصول على تفاصيل الإعداد، وخيارات النماذج، ومقتطفات الإعداد.

على مسار البث المتوافق مع Anthropic في MiniMax، يعطّل OpenClaw التفكير
افتراضيًا ما لم تقم بتعيينه صراحة، ويقوم `/fast on` بإعادة كتابة
`MiniMax-M2.7` إلى `MiniMax-M2.7-highspeed`.

تقسيم القدرات المملوك للإضافة:

- تبقى إعدادات النص/الدردشة الافتراضية على `minimax/MiniMax-M2.7`
- يكون توليد الصور هو `minimax/image-01` أو `minimax-portal/image-01`
- يكون فهم الصور هو `MiniMax-VL-01` المملوك للإضافة على مساري مصادقة MiniMax
- يبقى البحث على الويب على معرّف الموفر `minimax`

### Ollama

يشحن Ollama كإضافة موفر مضمّنة ويستخدم API الأصلية لـ Ollama:

- الموفر: `ollama`
- المصادقة: لا يلزم (خادم محلي)
- مثال على النموذج: `ollama/llama3.3`
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

يتم اكتشاف Ollama محليًا على `http://127.0.0.1:11434` عندما تقوم بتفعيله عبر
`OLLAMA_API_KEY`، وتضيف إضافة الموفر المضمّنة Ollama مباشرة إلى
`openclaw onboard` ومنتقي النماذج. راجع [/providers/ollama](/ar/providers/ollama)
للحصول على الإعداد الأولي، والوضع السحابي/المحلي، والإعدادات المخصصة.

### vLLM

يشحن vLLM كإضافة موفر مضمّنة لخوادم OpenAI-compatible المحلية/المستضافة ذاتيًا:

- الموفر: `vllm`
- المصادقة: اختيارية (حسب خادمك)
- عنوان URL الأساسي الافتراضي: `http://127.0.0.1:8000/v1`

للتفعيل المحلي للاكتشاف التلقائي (أي قيمة تعمل إذا كان خادمك لا يفرض المصادقة):

```bash
export VLLM_API_KEY="vllm-local"
```

ثم عيّن نموذجًا (استبدله بأحد المعرّفات التي يعيدها `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

راجع [/providers/vllm](/ar/providers/vllm) للتفاصيل.

### SGLang

يشحن SGLang كإضافة موفر مضمّنة لخوادم OpenAI-compatible المستضافة ذاتيًا
السريعة:

- الموفر: `sglang`
- المصادقة: اختيارية (حسب خادمك)
- عنوان URL الأساسي الافتراضي: `http://127.0.0.1:30000/v1`

للتفعيل المحلي للاكتشاف التلقائي (أي قيمة تعمل إذا كان خادمك لا
يفرض المصادقة):

```bash
export SGLANG_API_KEY="sglang-local"
```

ثم عيّن نموذجًا (استبدله بأحد المعرّفات التي يعيدها `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

راجع [/providers/sglang](/ar/providers/sglang) للتفاصيل.

### الوكلاء المحليون (LM Studio و vLLM و LiteLLM وما إلى ذلك)

مثال (متوافق مع OpenAI):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "محلي" } },
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
            name: "نموذج محلي",
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

- بالنسبة إلى الموفّرين المخصصين، تكون الحقول `reasoning` و `input` و `cost` و `contextWindow` و `maxTokens` اختيارية.
  وعند حذفها، يستخدم OpenClaw القيم الافتراضية التالية:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- الموصى به: تعيين قيم صريحة تطابق حدود الوكيل/النموذج لديك.
- بالنسبة إلى `api: "openai-completions"` على نقاط النهاية غير الأصلية (أي `baseUrl` غير فارغ لا يكون مضيفه `api.openai.com`)، يفرض OpenClaw `compat.supportsDeveloperRole: false` لتجنب أخطاء الموفّر 400 الخاصة بأدوار `developer` غير المدعومة.
- تتجاوز أيضًا المسارات الوكيلة المتوافقة مع OpenAI تشكيل الطلب الأصلي الخاص بـ OpenAI فقط:
  لا يوجد `service_tier`، ولا `Responses store`، ولا تلميحات لذاكرة الطلب المؤقتة، ولا
  تشكيل حمولة متوافق مع استدلال OpenAI، ولا رؤوس إسناد OpenClaw المخفية.
- إذا كان `baseUrl` فارغًا/محذوفًا، يحتفظ OpenClaw بسلوك OpenAI الافتراضي (الذي يُحل إلى `api.openai.com`).
- ولأسباب تتعلق بالسلامة، يتم أيضًا تجاوز `compat.supportsDeveloperRole: true` الصريح على نقاط نهاية `openai-completions` غير الأصلية.

## أمثلة CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

راجع أيضًا: [/gateway/configuration](/ar/gateway/configuration) للحصول على أمثلة إعداد كاملة.

## ذو صلة

- [النماذج](/ar/concepts/models) — إعداد النماذج والأسماء المستعارة
- [التبديل الاحتياطي للنماذج](/ar/concepts/model-failover) — سلاسل الاحتياط وسلوك إعادة المحاولة
- [مرجع الإعدادات](/ar/gateway/configuration-reference#agent-defaults) — مفاتيح إعداد النماذج
- [الموفّرون](/ar/providers) — أدلة الإعداد لكل موفر
