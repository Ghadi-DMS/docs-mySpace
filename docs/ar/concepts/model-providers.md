---
read_when:
    - تحتاج إلى مرجع لإعداد النماذج لكل موفر على حدة
    - تريد إعدادات تكوين نموذجية أو أوامر تهيئة CLI لموفري النماذج
summary: نظرة عامة على موفري النماذج مع إعدادات تكوين نموذجية + تدفقات CLI
title: موفرو النماذج
x-i18n:
    generated_at: "2026-04-08T02:16:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 26b36a2bc19a28a7ef39aa8e81a0050fea1d452ac4969122e5cdf8755e690258
    source_path: concepts/model-providers.md
    workflow: 15
---

# موفرو النماذج

تغطي هذه الصفحة **موفري LLM/النماذج** (وليس قنوات الدردشة مثل WhatsApp/Telegram).
للاطلاع على قواعد اختيار النماذج، راجع [/concepts/models](/ar/concepts/models).

## قواعد سريعة

- تستخدم مراجع النماذج الصيغة `provider/model` (مثال: `opencode/claude-opus-4-6`).
- إذا قمت بتعيين `agents.defaults.models`، فستصبح هي قائمة السماح.
- مساعدات CLI: ‏`openclaw onboard`، ‏`openclaw models list`، ‏`openclaw models set <provider/model>`.
- قواعد وقت التشغيل الاحتياطية، ومجسات التهدئة، واستمرارية تجاوز الجلسة
  موثقة في [/concepts/model-failover](/ar/concepts/model-failover).
- `models.providers.*.models[].contextWindow` هي بيانات تعريف أصلية للنموذج؛
  و`models.providers.*.models[].contextTokens` هو الحد الفعّال في وقت التشغيل.
- يمكن لإضافات الموفر حقن فهارس النماذج عبر `registerProvider({ catalog })`؛
  ويقوم OpenClaw بدمج هذا الناتج في `models.providers` قبل كتابة
  `models.json`.
- يمكن لبيانات تعريف الموفر أن تصرح بـ `providerAuthEnvVars` بحيث لا تحتاج
  مجسات المصادقة العامة المعتمدة على متغيرات البيئة إلى تحميل وقت تشغيل الإضافة. أما خريطة
  متغيرات البيئة الأساسية المتبقية فهي الآن فقط لموفري النواة/غير الإضافات
  وبعض حالات الأسبقية العامة مثل تهيئة Anthropic مع تفضيل مفتاح API أولًا.
- يمكن لإضافات الموفر أيضًا امتلاك سلوك وقت تشغيل الموفر عبر
  `normalizeModelId`، ‏`normalizeTransport`، ‏`normalizeConfig`،
  ‏`applyNativeStreamingUsageCompat`، ‏`resolveConfigApiKey`،
  ‏`resolveSyntheticAuth`، ‏`shouldDeferSyntheticProfileAuth`،
  ‏`resolveDynamicModel`، ‏`prepareDynamicModel`،
  ‏`normalizeResolvedModel`، ‏`contributeResolvedModelCompat`،
  ‏`capabilities`، ‏`normalizeToolSchemas`،
  ‏`inspectToolSchemas`، ‏`resolveReasoningOutputMode`،
  ‏`prepareExtraParams`، ‏`createStreamFn`، ‏`wrapStreamFn`،
  ‏`resolveTransportTurnState`، ‏`resolveWebSocketSessionPolicy`،
  ‏`createEmbeddingProvider`، ‏`formatApiKey`، ‏`refreshOAuth`،
  ‏`buildAuthDoctorHint`،
  ‏`matchesContextOverflowError`، ‏`classifyFailoverReason`،
  ‏`isCacheTtlEligible`، ‏`buildMissingAuthMessage`، ‏`suppressBuiltInModel`،
  ‏`augmentModelCatalog`، ‏`isBinaryThinking`، ‏`supportsXHighThinking`،
  ‏`resolveDefaultThinkingLevel`، ‏`applyConfigDefaults`، ‏`isModernModelRef`،
  ‏`prepareRuntimeAuth`، ‏`resolveUsageAuth`، ‏`fetchUsageSnapshot`، و
  ‏`onModelSelected`.
- ملاحظة: ‏`capabilities` في وقت تشغيل الموفر هي بيانات تعريف مشتركة للمشغّل (عائلة الموفر، وخصائص النصوص والأدوات، وتلميحات النقل/التخزين المؤقت). وهي ليست
  نفسها [نموذج القدرات العام](/ar/plugins/architecture#public-capability-model)
  الذي يصف ما الذي تسجله الإضافة (الاستدلال النصي، والكلام، وما إلى ذلك).

## السلوك المملوك لإضافة الموفر

يمكن لإضافات الموفر الآن امتلاك معظم المنطق الخاص بالموفر بينما يحتفظ OpenClaw
بحلقة الاستدلال العامة.

التقسيم المعتاد:

- `auth[].run` / `auth[].runNonInteractive`: يمتلك الموفر تدفقات التهيئة/تسجيل الدخول
  الخاصة بـ `openclaw onboard` و`openclaw models auth` والإعداد بدون واجهة
- `wizard.setup` / `wizard.modelPicker`: يمتلك الموفر تسميات خيارات المصادقة،
  والأسماء المستعارة القديمة، وتلميحات قائمة السماح الخاصة بالتهيئة، وإدخالات الإعداد في أدوات اختيار التهيئة/النموذج
- `catalog`: يظهر الموفر في `models.providers`
- `normalizeModelId`: يقوم الموفر بتطبيع معرّفات النماذج القديمة/المعاينة قبل
  البحث أو تحويلها إلى الشكل القياسي
- `normalizeTransport`: يقوم الموفر بتطبيع `api` / `baseUrl` الخاصة بعائلة النقل
  قبل التجميع العام للنموذج؛ ويتحقق OpenClaw من الموفر المطابق أولًا،
  ثم من إضافات الموفر الأخرى القادرة على الخطافات حتى تغيّر إحداها
  النقل فعلًا
- `normalizeConfig`: يقوم الموفر بتطبيع إعداد `models.providers.<id>` قبل
  أن يستخدمه وقت التشغيل؛ ويتحقق OpenClaw من الموفر المطابق أولًا، ثم من
  إضافات الموفر الأخرى القادرة على الخطافات حتى تغيّر إحداها الإعداد فعلًا. إذا لم
  تعِد أي خطافة موفر كتابة الإعداد، فإن مساعدات Google المجمعة للعائلة ما تزال
  تطبع إدخالات موفر Google المدعومة.
- `applyNativeStreamingUsageCompat`: يطبق الموفر إعادة كتابة توافق استخدام البث الأصلية المدفوعة بنقطة النهاية لموفري الإعداد
- `resolveConfigApiKey`: يقوم الموفر بحل مصادقة علامة البيئة لموفري الإعداد
  بدون فرض تحميل كامل لمصادقة وقت التشغيل. ويحتوي `amazon-bedrock` أيضًا على
  محلل مدمج لعلامات بيئة AWS هنا، رغم أن مصادقة وقت تشغيل Bedrock تستخدم
  سلسلة AWS SDK الافتراضية.
- `resolveSyntheticAuth`: يمكن للموفر إتاحة توفر المصادقة المحلية/المستضافة ذاتيًا أو غيرها من
  المصادقات المعتمدة على الإعداد بدون حفظ أسرار نصية صريحة
- `shouldDeferSyntheticProfileAuth`: يمكن للموفر وسم عناصر نائبة للمِلَفات التعريفية الاصطناعية المخزنة
  على أنها أقل أسبقية من المصادقة المستندة إلى البيئة/الإعداد
- `resolveDynamicModel`: يقبل الموفر معرّفات نماذج غير موجودة بعد في
  الفهرس الثابت المحلي
- `prepareDynamicModel`: يحتاج الموفر إلى تحديث البيانات التعريفية قبل إعادة
  محاولة الحل الديناميكي
- `normalizeResolvedModel`: يحتاج الموفر إلى إعادة كتابة النقل أو عنوان URL الأساسي
- `contributeResolvedModelCompat`: يضيف الموفر علامات توافق لنماذجه
  الخاصة بالمورّد حتى عندما تصل عبر نقل متوافق آخر
- `capabilities`: ينشر الموفر خصائص quirks للنصوص/الأدوات/عائلة الموفر
- `normalizeToolSchemas`: ينظف الموفر مخططات الأدوات قبل أن يراها
  المشغّل المضمن
- `inspectToolSchemas`: يعرض الموفر تحذيرات المخططات الخاصة بالنقل
  بعد التطبيع
- `resolveReasoningOutputMode`: يختار الموفر بين عقود إخراج الاستدلال
  الأصلية أو الموسومة
- `prepareExtraParams`: يضبط الموفر افتراضيًا أو يطبع معاملات الطلب لكل نموذج
- `createStreamFn`: يستبدل الموفر مسار البث العادي بنقل
  مخصص بالكامل
- `wrapStreamFn`: يطبّق الموفر أغلفة توافق الطلب/الجسم/النموذج
- `resolveTransportTurnState`: يوفّر الموفر رؤوسًا أو بيانات تعريف
  خاصة بكل دورة للنقل الأصلي
- `resolveWebSocketSessionPolicy`: يوفّر الموفر رؤوس جلسة WebSocket أصلية
  أو سياسة تبريد الجلسة
- `createEmbeddingProvider`: يمتلك الموفر سلوك تضمين الذاكرة عندما
  يكون من الأنسب أن يكون ضمن إضافة الموفر بدلًا من لوحة توزيع التضمين الأساسية
- `formatApiKey`: يقوم الموفر بتنسيق ملفات تعريف المصادقة المخزنة إلى سلسلة
  `apiKey` الخاصة بوقت التشغيل كما يتوقعها النقل
- `refreshOAuth`: يمتلك الموفر تحديث OAuth عندما لا تكون مجددات
  `pi-ai` المشتركة كافية
- `buildAuthDoctorHint`: يضيف الموفر إرشادات إصلاح عندما يفشل تحديث OAuth
- `matchesContextOverflowError`: يتعرف الموفر على أخطاء تجاوز نافذة السياق الخاصة بالموفر
  التي قد تفوتها الاستدلالات العامة
- `classifyFailoverReason`: يربط الموفر أخطاء النقل/API الخام الخاصة بالموفر
  بأسباب التحويل الاحتياطي مثل حد المعدل أو التحميل الزائد
- `isCacheTtlEligible`: يحدد الموفر أي معرّفات النماذج لدى المصدر تدعم مدة صلاحية التخزين المؤقت للمطالبة
- `buildMissingAuthMessage`: يستبدل الموفر خطأ مخزن المصادقة العام
  بتلميح استعادة خاص بالموفر
- `suppressBuiltInModel`: يخفي الموفر الصفوف القديمة لدى المصدر ويمكنه إرجاع
  خطأ مملوكًا للمورّد عند فشل الحل المباشر
- `augmentModelCatalog`: يضيف الموفر صفوف فهرس اصطناعية/نهائية بعد
  الاكتشاف ودمج الإعداد
- `isBinaryThinking`: يمتلك الموفر تجربة استخدام التفكير الثنائي تشغيل/إيقاف
- `supportsXHighThinking`: يفعّل الموفر `xhigh` لنماذج محددة
- `resolveDefaultThinkingLevel`: يمتلك الموفر سياسة `/think` الافتراضية لعائلة
  النماذج
- `applyConfigDefaults`: يطبق الموفر إعدادات افتراضية عامة خاصة بالموفر
  أثناء إنشاء الإعداد استنادًا إلى وضع المصادقة أو البيئة أو عائلة النموذج
- `isModernModelRef`: يمتلك الموفر مطابقة النموذج المفضل للبث الحي/الاختبار الدخاني
- `prepareRuntimeAuth`: يحول الموفر بيانات الاعتماد المكوّنة إلى رمز وقت تشغيل
  قصير العمر
- `resolveUsageAuth`: يحل الموفر بيانات اعتماد الاستخدام/الحصة لأجل `/usage`
  والواجهات ذات الصلة بالحالة/التقارير
- `fetchUsageSnapshot`: يمتلك الموفر جلب/تحليل نقطة نهاية الاستخدام بينما
  لا تزال النواة تمتلك الغلاف والتهيئة الملخصة
- `onModelSelected`: يشغّل الموفر آثارًا جانبية بعد الاختيار مثل
  telemetry أو حفظ الجلسة المملوك للموفر

أمثلة الحِزم الحالية:

- `anthropic`: بديل توافق أمامي لـ Claude 4.6، وتلميحات إصلاح المصادقة، وجلب
  نقطة نهاية الاستخدام، وبيانات تعريف TTL التخزين المؤقت/عائلة الموفر، وإعدادات
  افتراضية عامة تراعي المصادقة
- `amazon-bedrock`: مطابقة تجاوز السياق وتصنيف أسباب
  التحويل الاحتياطي المملوكان للموفر لأخطاء Bedrock الخاصة بالخنق/عدم الجاهزية، إضافة إلى
  عائلة الإعادة المشتركة `anthropic-by-model` لحمايات سياسة الإعادة الخاصة بـ Claude فقط
  على حركة Anthropic
- `anthropic-vertex`: حمايات سياسة الإعادة الخاصة بـ Claude فقط على
  حركة رسائل Anthropic
- `openrouter`: معرّفات نماذج تمريرية، وأغلفة طلبات، وتلميحات قدرات الموفر،
  وتنقية توقيع أفكار Gemini على حركة Gemini الوكيلة، وحقن الاستدلال
  الوكيلي عبر عائلة البث `openrouter-thinking`، وتمرير بيانات
  التوجيه، وسياسة TTL للتخزين المؤقت
- `github-copilot`: تهيئة/تسجيل دخول الجهاز، وبديل نموذج بتوافق أمامي،
  وتلميحات نصية لتفكير Claude، وتبادل رمز وقت التشغيل، وجلب نقطة
  نهاية الاستخدام
- `openai`: بديل توافق أمامي لـ GPT-5.4، وتطبيع مباشر لنقل OpenAI،
  وتلميحات فقدان مصادقة تراعي Codex، وكبت Spark، وصفوف فهرس
  اصطناعية لـ OpenAI/Codex، وسياسة التفكير/النموذج الحي، وتطبيع أسماء رموز الاستخدام المستعارة
  (`input` / `output` و`prompt` / `completion`)، وعائلة البث المشتركة
  `openai-responses-defaults` لأغلفة OpenAI/Codex الأصلية،
  وبيانات تعريف عائلة الموفر، وتسجيل موفر توليد الصور المجمّع
  لـ `gpt-image-1`، وتسجيل موفر توليد الفيديو المجمّع
  لـ `sora-2`
- `google` و`google-gemini-cli`: بديل توافق أمامي لـ Gemini 3.1،
  والتحقق الأصلي من إعادة Gemini، وتنقية إعادة التشغيل عند الإقلاع، ونمط
  إخراج استدلال موسوم، ومطابقة النماذج الحديثة، وتسجيل موفر
  توليد الصور المجمّع لنماذج Gemini image-preview، وتسجيل
  موفر توليد الفيديو المجمّع لنماذج Veo؛ كما أن Gemini CLI OAuth
  يمتلك أيضًا تنسيق رموز ملفات تعريف المصادقة، وتحليل رموز الاستخدام، وجلب
  نقطة نهاية الحصة لواجهات الاستخدام
- `moonshot`: نقل مشترك، وتطبيع حمولة التفكير المملوك للإضافة
- `kilocode`: نقل مشترك، ورؤوس طلبات مملوكة للإضافة، وتطبيع حمولة الاستدلال،
  وتنقية توقيع أفكار Gemini الوكيل، وسياسة TTL للتخزين المؤقت
- `zai`: بديل توافق أمامي لـ GLM-5، وافتراضات `tool_stream`، وسياسة TTL للتخزين المؤقت،
  وسياسة التفكير الثنائي/النموذج الحي، ومصادقة الاستخدام + جلب الحصة؛
  وتُنشأ معرّفات `glm-5*` غير المعروفة اصطناعيًا من قالب `glm-4.7` المجمّع
- `xai`: تطبيع نقل Responses الأصلي، وإعادة كتابة الأسماء المستعارة `/fast` لنسخ
  Grok السريعة، و`tool_stream` افتراضيًا، وتنظيف
  مخطط الأدوات/حمولة الاستدلال الخاص بـ xAI، وتسجيل موفر
  توليد الفيديو المجمّع لـ `grok-imagine-video`
- `mistral`: بيانات تعريف قدرات مملوكة للإضافة
- `opencode` و`opencode-go`: بيانات تعريف قدرات مملوكة للإضافة بالإضافة إلى
  تنقية توقيع أفكار Gemini الوكيل
- `alibaba`: فهرس توليد فيديو مملوك للإضافة لمراجع نماذج Wan المباشرة
  مثل `alibaba/wan2.6-t2v`
- `byteplus`: فهارس مملوكة للإضافة بالإضافة إلى تسجيل موفر توليد الفيديو المجمّع
  لنماذج Seedance لتحويل النص إلى فيديو/الصورة إلى فيديو
- `fal`: تسجيل موفر توليد الفيديو المجمّع لاستضافة خارجية
  وتسجيل موفر توليد الصور لاستضافة خارجية لنماذج صور FLUX بالإضافة إلى
  تسجيل موفر توليد الفيديو المجمّع لنماذج الفيديو المستضافة من جهات خارجية
- `cloudflare-ai-gateway` و`huggingface` و`kimi` و`nvidia` و`qianfan`،
  و`stepfun` و`synthetic` و`venice` و`vercel-ai-gateway` و`volcengine`:
  فهارس مملوكة للإضافة فقط
- `qwen`: فهارس مملوكة للإضافة للنماذج النصية بالإضافة إلى تسجيلات
  مشتركة لموفري فهم الوسائط وتوليد الفيديو لواجهاته متعددة الوسائط؛ ويستخدم
  توليد الفيديو في Qwen نقاط نهاية الفيديو القياسية لـ DashScope مع
  نماذج Wan المجمّعة مثل `wan2.6-t2v` و`wan2.7-r2v`
- `runway`: تسجيل موفر توليد فيديو مملوك للإضافة لنماذج Runway
  الأصلية المعتمدة على المهام مثل `gen4.5`
- `minimax`: فهارس مملوكة للإضافة، وتسجيل موفر توليد فيديو مجمّع
  لنماذج فيديو Hailuo، وتسجيل موفر توليد صور مجمّع
  لـ `image-01`، واختيار هجين لسياسة إعادة Anthropic/OpenAI،
  ومنطق مصادقة/لقطة الاستخدام
- `together`: فهارس مملوكة للإضافة بالإضافة إلى تسجيل موفر توليد الفيديو المجمّع
  لنماذج فيديو Wan
- `xiaomi`: فهارس مملوكة للإضافة بالإضافة إلى منطق مصادقة/لقطة الاستخدام

تمتلك الإضافة المجمّعة `openai` الآن كلا معرّفي الموفر: `openai` و
`openai-codex`.

يغطي هذا الموفرين الذين ما زالوا يلائمون وسائل النقل العادية في OpenClaw. أما الموفر
الذي يحتاج إلى منفذ طلبات مخصص بالكامل فهو سطح توسعة منفصل وأعمق.

## تدوير مفاتيح API

- يدعم تدويرًا عامًا للموفر لموفري محددين.
- قم بتكوين عدة مفاتيح عبر:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (تجاوز حي مفرد، بأعلى أولوية)
  - `<PROVIDER>_API_KEYS` (قائمة مفصولة بفواصل أو فاصلة منقوطة)
  - `<PROVIDER>_API_KEY` (المفتاح الأساسي)
  - `<PROVIDER>_API_KEY_*` (قائمة مرقمة، مثل `<PROVIDER>_API_KEY_1`)
- بالنسبة إلى موفري Google، يتم تضمين `GOOGLE_API_KEY` أيضًا كخيار احتياطي.
- يحافظ ترتيب اختيار المفاتيح على الأولوية ويزيل القيم المكررة.
- تتم إعادة محاولة الطلبات باستخدام المفتاح التالي فقط عند استجابات حد المعدل (على
  سبيل المثال `429` أو `rate_limit` أو `quota` أو `resource exhausted` أو `Too many
concurrent requests` أو `ThrottlingException` أو `concurrency limit reached`،
  أو `workers_ai ... quota limit exceeded`، أو رسائل حد الاستخدام الدورية).
- تفشل الحالات غير المرتبطة بحد المعدل فورًا؛ ولا تتم محاولة تدوير المفاتيح.
- عند فشل جميع المفاتيح المرشحة، يتم إرجاع الخطأ النهائي من آخر محاولة.

## الموفّرون المدمجون (فهرس pi-ai)

يأتي OpenClaw مع فهرس pi‑ai. لا تتطلب هذه الموفّرات أي إعداد
`models.providers`؛ فقط قم بتعيين المصادقة + اختر نموذجًا.

### OpenAI

- الموفّر: `openai`
- المصادقة: `OPENAI_API_KEY`
- التدوير الاختياري: `OPENAI_API_KEYS` و`OPENAI_API_KEY_1` و`OPENAI_API_KEY_2`، بالإضافة إلى `OPENCLAW_LIVE_OPENAI_KEY` (تجاوز مفرد)
- نماذج مثال: `openai/gpt-5.4`، `openai/gpt-5.4-pro`
- CLI: ‏`openclaw onboard --auth-choice openai-api-key`
- النقل الافتراضي هو `auto` (WebSocket أولًا، ثم SSE كخيار احتياطي)
- تجاوز لكل نموذج عبر `agents.defaults.models["openai/<model>"].params.transport` (`"sse"` أو `"websocket"` أو `"auto"`)
- يتم تفعيل الإحماء المسبق لـ OpenAI Responses WebSocket افتراضيًا عبر `params.openaiWsWarmup` (`true`/`false`)
- يمكن تمكين المعالجة ذات الأولوية في OpenAI عبر `agents.defaults.models["openai/<model>"].params.serviceTier`
- يقوم `/fast` و`params.fastMode` بربط طلبات Responses المباشرة `openai/*` إلى `service_tier=priority` على `api.openai.com`
- استخدم `params.serviceTier` عندما تريد طبقة صريحة بدلًا من مفتاح `/fast` المشترك
- تُطبَّق رؤوس الإسناد المخفية الخاصة بـ OpenClaw (`originator`، و`version`،
  و`User-Agent`) فقط على حركة OpenAI الأصلية إلى `api.openai.com`، وليس على
  الوكلاء العامة المتوافقة مع OpenAI
- تحتفظ مسارات OpenAI الأصلية أيضًا بخيار Responses `store`، وتلميحات التخزين المؤقت للمطالبة،
  وتهيئة حمولة توافق الاستدلال في OpenAI؛ أما مسارات الوكلاء فلا تحتفظ بذلك
- تم كبت `openai/gpt-5.3-codex-spark` عمدًا في OpenClaw لأن واجهة OpenAI الحية ترفضه؛ ويتم التعامل مع Spark على أنه خاص بـ Codex فقط

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- الموفّر: `anthropic`
- المصادقة: `ANTHROPIC_API_KEY`
- التدوير الاختياري: `ANTHROPIC_API_KEYS` و`ANTHROPIC_API_KEY_1` و`ANTHROPIC_API_KEY_2`، بالإضافة إلى `OPENCLAW_LIVE_ANTHROPIC_KEY` (تجاوز مفرد)
- نموذج مثال: `anthropic/claude-opus-4-6`
- CLI: ‏`openclaw onboard --auth-choice apiKey`
- تدعم طلبات Anthropic العامة المباشرة مفتاح `/fast` المشترك و`params.fastMode`، بما في ذلك حركة المرور المصادق عليها بمفتاح API وOAuth المرسلة إلى `api.anthropic.com`؛ ويقوم OpenClaw بربط ذلك إلى `service_tier` الخاص بـ Anthropic (`auto` مقابل `standard_only`)
- ملاحظة Anthropic: أخبرنا موظفو Anthropic أن استخدام Claude CLI بأسلوب OpenClaw مسموح به مرة أخرى، لذلك يتعامل OpenClaw مع إعادة استخدام Claude CLI واستخدام `claude -p` على أنهما معتمدان لهذا التكامل ما لم تنشر Anthropic سياسة جديدة.
- يظل رمز إعداد Anthropic متاحًا كمسار رمز مدعوم في OpenClaw، لكن OpenClaw يفضّل الآن إعادة استخدام Claude CLI و`claude -p` عند توفرهما.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- الموفّر: `openai-codex`
- المصادقة: OAuth (ChatGPT)
- نموذج مثال: `openai-codex/gpt-5.4`
- CLI: ‏`openclaw onboard --auth-choice openai-codex` أو `openclaw models auth login --provider openai-codex`
- النقل الافتراضي هو `auto` (WebSocket أولًا، ثم SSE كخيار احتياطي)
- تجاوز لكل نموذج عبر `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"` أو `"websocket"` أو `"auto"`)
- يتم أيضًا تمرير `params.serviceTier` على طلبات Codex Responses الأصلية (`chatgpt.com/backend-api`)
- تُرفق رؤوس الإسناد المخفية الخاصة بـ OpenClaw (`originator` و`version`،
  و`User-Agent`) فقط على حركة Codex الأصلية إلى
  `chatgpt.com/backend-api`، وليس على الوكلاء العامة المتوافقة مع OpenAI
- يشترك في مفتاح `/fast` نفسه وإعداد `params.fastMode` نفسه كما في `openai/*` المباشر؛ ويقوم OpenClaw بربط ذلك إلى `service_tier=priority`
- يظل `openai-codex/gpt-5.3-codex-spark` متاحًا عندما يعرِضه فهرس Codex OAuth؛ ويعتمد على الاستحقاق
- يحتفظ `openai-codex/gpt-5.4` بالقيم الأصلية `contextWindow = 1050000` وحد وقت تشغيل افتراضي `contextTokens = 272000`؛ ويمكنك تجاوز حد وقت التشغيل عبر `models.providers.openai-codex.models[].contextTokens`
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

### خيارات مستضافة أخرى على نمط الاشتراك

- [Qwen Cloud](/ar/providers/qwen): واجهة موفر Qwen Cloud بالإضافة إلى ربط نقاط نهاية Alibaba DashScope وCoding Plan
- [MiniMax](/ar/providers/minimax): وصول MiniMax Coding Plan عبر OAuth أو مفتاح API
- [GLM Models](/ar/providers/glm): نقاط نهاية Z.AI Coding Plan أو نقاط نهاية API العامة

### OpenCode

- المصادقة: `OPENCODE_API_KEY` (أو `OPENCODE_ZEN_API_KEY`)
- موفر وقت تشغيل Zen: ‏`opencode`
- موفر وقت تشغيل Go: ‏`opencode-go`
- نماذج مثال: `opencode/claude-opus-4-6`، `opencode-go/kimi-k2.5`
- CLI: ‏`openclaw onboard --auth-choice opencode-zen` أو `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (مفتاح API)

- الموفّر: `google`
- المصادقة: `GEMINI_API_KEY`
- التدوير الاختياري: `GEMINI_API_KEYS` و`GEMINI_API_KEY_1` و`GEMINI_API_KEY_2`، وخيار احتياطي `GOOGLE_API_KEY`، و`OPENCLAW_LIVE_GEMINI_KEY` (تجاوز مفرد)
- نماذج مثال: `google/gemini-3.1-pro-preview`، `google/gemini-3-flash-preview`
- التوافق: يتم تطبيع إعداد OpenClaw القديم الذي يستخدم `google/gemini-3.1-flash-preview` إلى `google/gemini-3-flash-preview`
- CLI: ‏`openclaw onboard --auth-choice gemini-api-key`
- تقبل عمليات Gemini المباشرة أيضًا `agents.defaults.models["google/<model>"].params.cachedContent`
  (أو الصيغة القديمة `cached_content`) لتمرير
  مقبض `cachedContents/...` أصلي خاص بالموفر؛ وتظهر إصابات ذاكرة التخزين المؤقت في Gemini على هيئة `cacheRead` في OpenClaw

### Google Vertex وGemini CLI

- الموفّرون: `google-vertex`، ‏`google-gemini-cli`
- المصادقة: يستخدم Vertex آلية gcloud ADC؛ ويستخدم Gemini CLI تدفق OAuth الخاص به
- تنبيه: تكامل Gemini CLI OAuth في OpenClaw غير رسمي. أبلغ بعض المستخدمين عن فرض قيود على حسابات Google بعد استخدام عملاء من جهات خارجية. راجع شروط Google واستخدم حسابًا غير حرج إذا اخترت المتابعة.
- يتم شحن Gemini CLI OAuth كجزء من إضافة `google` المجمّعة.
  - قم بتثبيت Gemini CLI أولًا:
    - `brew install gemini-cli`
    - أو `npm install -g @google/gemini-cli`
  - التمكين: `openclaw plugins enable google`
  - تسجيل الدخول: `openclaw models auth login --provider google-gemini-cli --set-default`
  - النموذج الافتراضي: `google-gemini-cli/gemini-3-flash-preview`
  - ملاحظة: **لا** تقوم بلصق client id أو secret في `openclaw.json`. يقوم تدفق تسجيل دخول CLI بتخزين
    الرموز في ملفات تعريف المصادقة على مضيف البوابة.
  - إذا فشلت الطلبات بعد تسجيل الدخول، فاضبط `GOOGLE_CLOUD_PROJECT` أو `GOOGLE_CLOUD_PROJECT_ID` على مضيف البوابة.
  - يتم تحليل ردود JSON الخاصة بـ Gemini CLI من `response`؛ ويعود الاستخدام احتياطيًا إلى
    `stats`، مع تطبيع `stats.cached` إلى `cacheRead` في OpenClaw.

### Z.AI (GLM)

- الموفّر: `zai`
- المصادقة: `ZAI_API_KEY`
- نموذج مثال: `zai/glm-5`
- CLI: ‏`openclaw onboard --auth-choice zai-api-key`
  - الأسماء المستعارة: يتم تطبيع `z.ai/*` و`z-ai/*` إلى `zai/*`
  - يقوم `zai-api-key` بالكشف التلقائي عن نقطة نهاية Z.AI المطابقة؛ بينما يفرض `zai-coding-global` و`zai-coding-cn` و`zai-global` و`zai-cn` واجهة محددة

### Vercel AI Gateway

- الموفّر: `vercel-ai-gateway`
- المصادقة: `AI_GATEWAY_API_KEY`
- نموذج مثال: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: ‏`openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- الموفّر: `kilocode`
- المصادقة: `KILOCODE_API_KEY`
- نموذج مثال: `kilocode/kilo/auto`
- CLI: ‏`openclaw onboard --auth-choice kilocode-api-key`
- عنوان URL الأساسي: `https://api.kilo.ai/api/gateway/`
- يشحن فهرس احتياطي ثابت `kilocode/kilo/auto`؛ ويمكن أن يوسّع
  الاكتشاف الحي لـ `https://api.kilo.ai/api/gateway/models` فهرس
  وقت التشغيل أكثر.
- التوجيه الدقيق لدى المصدر خلف `kilocode/kilo/auto` مملوك لـ Kilo Gateway،
  وليس مُرمَّزًا بشكل ثابت في OpenClaw.

راجع [/providers/kilocode](/ar/providers/kilocode) للحصول على تفاصيل الإعداد.

### إضافات موفّرين مدمجة أخرى

- OpenRouter: ‏`openrouter` ‏(`OPENROUTER_API_KEY`)
- نموذج مثال: `openrouter/auto`
- يطبق OpenClaw رؤوس إسناد التطبيق الموثقة في OpenRouter فقط عندما
  يستهدف الطلب فعليًا `openrouter.ai`
- يتم كذلك تقييد علامات `cache_control` الخاصة بـ Anthropic في OpenRouter إلى
  مسارات OpenRouter المتحقق منها، وليس عناوين URL وكيلة عشوائية
- يظل OpenRouter على المسار الوكيلي المتوافق مع OpenAI، لذلك لا يتم تمرير
  تهيئة الطلب الأصلية الخاصة بـ OpenAI فقط (`serviceTier`، وخيار Responses `store`،
  وتلميحات التخزين المؤقت للمطالبة، وحمولات توافق الاستدلال في OpenAI)
- تحتفظ مراجع OpenRouter المستندة إلى Gemini فقط بمسار
  تنقية توقيع أفكار Gemini الوكيل؛ وتظل إعادة التحقق الأصلية في Gemini وإعادة كتابة الإقلاع معطلتين
- Kilo Gateway: ‏`kilocode` ‏(`KILOCODE_API_KEY`)
- نموذج مثال: `kilocode/kilo/auto`
- تحتفظ مراجع Kilo المعتمدة على Gemini بمسار تنقية
  توقيع أفكار Gemini الوكيل نفسه؛ وتتجاوز التلميحات مثل `kilocode/kilo/auto` وغيرها من التلميحات غير المدعومة للاستدلال الوكيلي حقن الاستدلال الوكيلي
- MiniMax: ‏`minimax` (مفتاح API) و`minimax-portal` (OAuth)
- المصادقة: `MINIMAX_API_KEY` لـ `minimax`؛ و`MINIMAX_OAUTH_TOKEN` أو `MINIMAX_API_KEY` لـ `minimax-portal`
- نموذج مثال: `minimax/MiniMax-M2.7` أو `minimax-portal/MiniMax-M2.7`
- تكتب تهيئة MiniMax/إعداد مفتاح API تعريفات صريحة لنموذج M2.7 مع
  `input: ["text", "image"]`؛ بينما يحتفظ فهرس الموفر المجمّع بمراجع الدردشة
  على أنها نصية فقط حتى يتم إنشاء إعداد هذا الموفر
- Moonshot: ‏`moonshot` ‏(`MOONSHOT_API_KEY`)
- نموذج مثال: `moonshot/kimi-k2.5`
- Kimi Coding: ‏`kimi` ‏(`KIMI_API_KEY` أو `KIMICODE_API_KEY`)
- نموذج مثال: `kimi/kimi-code`
- Qianfan: ‏`qianfan` ‏(`QIANFAN_API_KEY`)
- نموذج مثال: `qianfan/deepseek-v3.2`
- Qwen Cloud: ‏`qwen` ‏(`QWEN_API_KEY` أو `MODELSTUDIO_API_KEY` أو `DASHSCOPE_API_KEY`)
- نموذج مثال: `qwen/qwen3.5-plus`
- NVIDIA: ‏`nvidia` ‏(`NVIDIA_API_KEY`)
- نموذج مثال: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: ‏`stepfun` / `stepfun-plan` ‏(`STEPFUN_API_KEY`)
- نماذج مثال: `stepfun/step-3.5-flash`، `stepfun-plan/step-3.5-flash-2603`
- Together: ‏`together` ‏(`TOGETHER_API_KEY`)
- نموذج مثال: `together/moonshotai/Kimi-K2.5`
- Venice: ‏`venice` ‏(`VENICE_API_KEY`)
- Xiaomi: ‏`xiaomi` ‏(`XIAOMI_API_KEY`)
- نموذج مثال: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: ‏`vercel-ai-gateway` ‏(`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: ‏`huggingface` ‏(`HUGGINGFACE_HUB_TOKEN` أو `HF_TOKEN`)
- Cloudflare AI Gateway: ‏`cloudflare-ai-gateway` ‏(`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: ‏`volcengine` ‏(`VOLCANO_ENGINE_API_KEY`)
- نموذج مثال: `volcengine-plan/ark-code-latest`
- BytePlus: ‏`byteplus` ‏(`BYTEPLUS_API_KEY`)
- نموذج مثال: `byteplus-plan/ark-code-latest`
- xAI: ‏`xai` ‏(`XAI_API_KEY`)
  - تستخدم طلبات xAI الأصلية المجمعة مسار xAI Responses
  - يعيد `/fast` أو `params.fastMode: true` كتابة `grok-3` و`grok-3-mini`،
    و`grok-4` و`grok-4-0709` إلى متغيراتها `*-fast`
  - يتم تفعيل `tool_stream` افتراضيًا؛ اضبط
    `agents.defaults.models["xai/<model>"].params.tool_stream` على `false` من
    أجل تعطيله
- Mistral: ‏`mistral` ‏(`MISTRAL_API_KEY`)
- نموذج مثال: `mistral/mistral-large-latest`
- CLI: ‏`openclaw onboard --auth-choice mistral-api-key`
- Groq: ‏`groq` ‏(`GROQ_API_KEY`)
- Cerebras: ‏`cerebras` ‏(`CEREBRAS_API_KEY`)
  - تستخدم نماذج GLM على Cerebras المعرّفين `zai-glm-4.7` و`zai-glm-4.6`.
  - عنوان URL الأساسي المتوافق مع OpenAI: ‏`https://api.cerebras.ai/v1`.
- GitHub Copilot: ‏`github-copilot` ‏(`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- نموذج مثال لـ Hugging Face Inference: ‏`huggingface/deepseek-ai/DeepSeek-R1`؛ CLI: ‏`openclaw onboard --auth-choice huggingface-api-key`. راجع [Hugging Face (Inference)](/ar/providers/huggingface).

## الموفّرون عبر `models.providers` (مخصص/عنوان URL أساسي)

استخدم `models.providers` (أو `models.json`) لإضافة موفّرين **مخصصين** أو
وكلاء متوافقين مع OpenAI/Anthropic.

تنشر العديد من إضافات الموفّرين المجمّعة أدناه بالفعل فهرسًا افتراضيًا.
استخدم إدخالات `models.providers.<id>` الصريحة فقط عندما تريد تجاوز
عنوان URL الأساسي الافتراضي أو الرؤوس أو قائمة النماذج.

### Moonshot AI (Kimi)

يأتي Moonshot كإضافة موفر مجمّعة. استخدم الموفّر المدمج افتراضيًا،
وأضف إدخال `models.providers.moonshot` صريحًا فقط عندما
تحتاج إلى تجاوز عنوان URL الأساسي أو بيانات تعريف النموذج:

- الموفّر: `moonshot`
- المصادقة: `MOONSHOT_API_KEY`
- نموذج مثال: `moonshot/kimi-k2.5`
- CLI: ‏`openclaw onboard --auth-choice moonshot-api-key` أو `openclaw onboard --auth-choice moonshot-api-key-cn`

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

- الموفّر: `kimi`
- المصادقة: `KIMI_API_KEY`
- نموذج مثال: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

ما يزال `kimi/k2p5` القديم مقبولًا كمعرّف نموذج للتوافق.

### Volcano Engine (Doubao)

يوفر Volcano Engine (火山引擎) الوصول إلى Doubao ونماذج أخرى داخل الصين.

- الموفّر: `volcengine` (الترميز: `volcengine-plan`)
- المصادقة: `VOLCANO_ENGINE_API_KEY`
- نموذج مثال: `volcengine-plan/ark-code-latest`
- CLI: ‏`openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

تستخدم التهيئة سطح الترميز افتراضيًا، لكن فهرس `volcengine/*`
العام يتم تسجيله في الوقت نفسه.

في أدوات اختيار نموذج التهيئة/الضبط، يفضّل خيار مصادقة Volcengine كِلَا
صفَّي `volcengine/*` و`volcengine-plan/*`. وإذا لم تكن هذه النماذج محمّلة بعد،
فإن OpenClaw يعود إلى الفهرس غير المصفّى بدلًا من عرض أداة اختيار فارغة
مقيدة بالموفّر.

النماذج المتاحة:

- `volcengine/doubao-seed-1-8-251228` ‏(Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` ‏(Kimi K2.5)
- `volcengine/glm-4-7-251222` ‏(GLM 4.7)
- `volcengine/deepseek-v3-2-251201` ‏(DeepSeek V3.2 128K)

نماذج الترميز (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (دولي)

يوفر BytePlus ARK الوصول إلى النماذج نفسها الموجودة في Volcano Engine للمستخدمين الدوليين.

- الموفّر: `byteplus` (الترميز: `byteplus-plan`)
- المصادقة: `BYTEPLUS_API_KEY`
- نموذج مثال: `byteplus-plan/ark-code-latest`
- CLI: ‏`openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

تستخدم التهيئة سطح الترميز افتراضيًا، لكن فهرس `byteplus/*`
العام يتم تسجيله في الوقت نفسه.

في أدوات اختيار نموذج التهيئة/الضبط، يفضّل خيار مصادقة BytePlus كِلَا
صفَّي `byteplus/*` و`byteplus-plan/*`. وإذا لم تكن هذه النماذج محمّلة بعد،
فإن OpenClaw يعود إلى الفهرس غير المصفّى بدلًا من عرض أداة اختيار فارغة
مقيدة بالموفّر.

النماذج المتاحة:

- `byteplus/seed-1-8-251228` ‏(Seed 1.8)
- `byteplus/kimi-k2-5-260127` ‏(Kimi K2.5)
- `byteplus/glm-4-7-251222` ‏(GLM 4.7)

نماذج الترميز (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

يوفر Synthetic نماذج متوافقة مع Anthropic خلف الموفّر `synthetic`:

- الموفّر: `synthetic`
- المصادقة: `SYNTHETIC_API_KEY`
- نموذج مثال: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
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

يتم تكوين MiniMax عبر `models.providers` لأنه يستخدم نقاط نهاية مخصصة:

- MiniMax OAuth (عالمي): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (الصين): `--auth-choice minimax-cn-oauth`
- مفتاح API لـ MiniMax (عالمي): `--auth-choice minimax-global-api`
- مفتاح API لـ MiniMax (الصين): `--auth-choice minimax-cn-api`
- المصادقة: `MINIMAX_API_KEY` لـ `minimax`؛ و`MINIMAX_OAUTH_TOKEN` أو
  `MINIMAX_API_KEY` لـ `minimax-portal`

راجع [/providers/minimax](/ar/providers/minimax) للحصول على تفاصيل الإعداد وخيارات النماذج ومقاطع التكوين.

على مسار البث المتوافق مع Anthropic في MiniMax، يعطّل OpenClaw التفكير
افتراضيًا ما لم تقم بتعيينه صراحةً، ويعيد `/fast on` كتابة
`MiniMax-M2.7` إلى `MiniMax-M2.7-highspeed`.

تقسيم القدرات المملوك للإضافة:

- تظل الإعدادات الافتراضية للنص/الدردشة على `minimax/MiniMax-M2.7`
- توليد الصور هو `minimax/image-01` أو `minimax-portal/image-01`
- فهم الصور هو `MiniMax-VL-01` مملوك للإضافة على كلا مساري مصادقة MiniMax
- يبقى البحث على الويب على معرّف الموفّر `minimax`

### Ollama

يأتي Ollama كإضافة موفر مجمّعة ويستخدم API الأصلي الخاص بـ Ollama:

- الموفّر: `ollama`
- المصادقة: غير مطلوبة (خادم محلي)
- نموذج مثال: `ollama/llama3.3`
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

يتم اكتشاف Ollama محليًا على `http://127.0.0.1:11434` عند الاشتراك
عبر `OLLAMA_API_KEY`، وتضيف إضافة الموفر المجمّعة Ollama مباشرةً إلى
`openclaw onboard` وأداة اختيار النموذج. راجع [/providers/ollama](/ar/providers/ollama)
للاطلاع على التهيئة ووضع السحابة/المحلي والإعداد المخصص.

### vLLM

يأتي vLLM كإضافة موفر مجمّعة للخوادم المحلية/المستضافة ذاتيًا
المتوافقة مع OpenAI:

- الموفّر: `vllm`
- المصادقة: اختيارية (بحسب خادمك)
- عنوان URL الأساسي الافتراضي: `http://127.0.0.1:8000/v1`

للاشتراك في الاكتشاف التلقائي محليًا (أي قيمة تعمل إذا كان خادمك لا يفرض المصادقة):

```bash
export VLLM_API_KEY="vllm-local"
```

ثم قم بتعيين نموذج (استبدله بأحد المعرّفات المعادة من `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

راجع [/providers/vllm](/ar/providers/vllm) للتفاصيل.

### SGLang

يأتي SGLang كإضافة موفر مجمّعة للخوادم السريعة المستضافة ذاتيًا
المتوافقة مع OpenAI:

- الموفّر: `sglang`
- المصادقة: اختيارية (بحسب خادمك)
- عنوان URL الأساسي الافتراضي: `http://127.0.0.1:30000/v1`

للاشتراك في الاكتشاف التلقائي محليًا (أي قيمة تعمل إذا كان خادمك لا
يفرض المصادقة):

```bash
export SGLANG_API_KEY="sglang-local"
```

ثم قم بتعيين نموذج (استبدله بأحد المعرّفات المعادة من `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

راجع [/providers/sglang](/ar/providers/sglang) للتفاصيل.

### الوكلاء المحليون (LM Studio وvLLM وLiteLLM وما إلى ذلك)

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

- بالنسبة إلى الموفّرين المخصصين، تكون `reasoning` و`input` و`cost` و`contextWindow` و`maxTokens` اختيارية.
  وعند حذفها، يستخدم OpenClaw الإعدادات الافتراضية التالية:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- الموصى به: تعيين قيم صريحة تطابق حدود الوكيل/النموذج لديك.
- بالنسبة إلى `api: "openai-completions"` على نقاط نهاية غير أصلية (أي `baseUrl` غير فارغ يكون مضيفه ليس `api.openai.com`)، يفرض OpenClaw القيمة `compat.supportsDeveloperRole: false` لتجنب أخطاء الموفّر 400 الخاصة بأدوار `developer` غير المدعومة.
- تتجاوز المسارات الوكيلة المتوافقة مع OpenAI أيضًا تهيئة الطلب الأصلية الخاصة بـ OpenAI فقط:
  لا يوجد `service_tier`، ولا `store` في Responses، ولا تلميحات للتخزين المؤقت للمطالبة، ولا
  تهيئة لحمولات توافق الاستدلال في OpenAI، ولا رؤوس إسناد مخفية خاصة بـ OpenClaw.
- إذا كان `baseUrl` فارغًا/غير محدد، يحتفظ OpenClaw بسلوك OpenAI الافتراضي (الذي يُحل إلى `api.openai.com`).
- من أجل الأمان، ما تزال القيمة الصريحة `compat.supportsDeveloperRole: true` تُتجاهل على نقاط النهاية غير الأصلية `openai-completions`.

## أمثلة CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

راجع أيضًا: [/gateway/configuration](/ar/gateway/configuration) للحصول على أمثلة تكوين كاملة.

## ذو صلة

- [Models](/ar/concepts/models) — تكوين النموذج والأسماء المستعارة
- [Model Failover](/ar/concepts/model-failover) — سلاسل الاحتياط وسلوك إعادة المحاولة
- [Configuration Reference](/ar/gateway/configuration-reference#agent-defaults) — مفاتيح تكوين النموذج
- [Providers](/ar/providers) — أدلة الإعداد الخاصة بكل موفّر
