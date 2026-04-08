---
read_when:
    - عند تشغيل الاختبارات محليًا أو في CI
    - عند إضافة اختبارات تراجعية لأخطاء النموذج/الموفّر
    - عند تصحيح سلوك gateway + الوكيل
summary: 'عدة الاختبار: أجنحة unit/e2e/live، ومشغلات Docker، وما الذي يغطيه كل اختبار'
title: الاختبار
x-i18n:
    generated_at: "2026-04-08T02:18:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: ace2c19bfc350780475f4348264a4b55be2b4ccbb26f0d33b4a6af34510943b5
    source_path: help/testing.md
    workflow: 15
---

# الاختبار

يحتوي OpenClaw على ثلاثة أجنحة Vitest ‏(unit/integration، وe2e، وlive) ومجموعة صغيرة من مشغلات Docker.

هذه الوثيقة هي دليل "كيف نختبر":

- ما الذي يغطيه كل جناح (وما الذي لا يغطيه عمدًا)
- ما الأوامر التي يجب تشغيلها لمسارات العمل الشائعة (محليًا، قبل الدفع، التصحيح)
- كيف تكتشف الاختبارات الحية بيانات الاعتماد وتختار النماذج/الموفّرين
- كيف تضيف اختبارات تراجعية لمشكلات النماذج/الموفّرين الواقعية

## البدء السريع

في معظم الأيام:

- البوابة الكاملة (متوقعة قبل الدفع): `pnpm build && pnpm check && pnpm test`
- تشغيل أسرع للجناح الكامل محليًا على جهاز واسع الموارد: `pnpm test:max`
- حلقة مراقبة Vitest مباشرة: `pnpm test:watch`
- استهداف مباشر للملفات يوجّه الآن مسارات extension/channel أيضًا: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- موقع QA مدعوم بـ Docker: `pnpm qa:lab:up`

عندما تلمس الاختبارات أو تريد ثقة إضافية:

- بوابة التغطية: `pnpm test:coverage`
- جناح E2E: `pnpm test:e2e`

عند تصحيح موفّرين/نماذج حقيقيين (يتطلب بيانات اعتماد حقيقية):

- الجناح الحي (فحوصات النماذج + أدوات/صور gateway): `pnpm test:live`
- استهداف ملف حي واحد بهدوء: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

نصيحة: عندما تحتاج إلى حالة فاشلة واحدة فقط، فالأفضل تضييق الاختبارات الحية عبر متغيرات البيئة الخاصة بقائمة السماح الموضحة أدناه.

## أجنحة الاختبار (ما الذي يعمل وأين)

فكّر في الأجنحة على أنها "تزداد واقعية" (وتزداد معها القابلية للتعطل/التكلفة):

### Unit / integration ‏(الافتراضي)

- الأمر: `pnpm test`
- الإعداد: عشر عمليات shard متتابعة (`vitest.full-*.config.ts`) على مشاريع Vitest المحددة الحالية
- الملفات: قوائم core/unit ضمن `src/**/*.test.ts` و`packages/**/*.test.ts` و`test/**/*.test.ts`، واختبارات `ui` الخاصة بـ node المدرجة في القائمة البيضاء والتي يغطيها `vitest.unit.config.ts`
- النطاق:
  - اختبارات unit خالصة
  - اختبارات integration داخل العملية (مصادقة gateway، والتوجيه، والأدوات، والتحليل، والإعدادات)
  - اختبارات تراجعية حتمية للأخطاء المعروفة
- التوقعات:
  - تعمل في CI
  - لا تتطلب مفاتيح حقيقية
  - يجب أن تكون سريعة ومستقرة
- ملاحظة المشاريع:
  - أصبح `pnpm test` غير المستهدف يشغّل الآن إحدى عشرة إعدادات shard أصغر (`core-unit-src` و`core-unit-security` و`core-unit-ui` و`core-unit-support` و`core-support-boundary` و`core-contracts` و`core-bundled` و`core-runtime` و`agentic` و`auto-reply` و`extensions`) بدلًا من عملية root-project أصلية واحدة ضخمة. وهذا يقلل ذروة RSS على الأجهزة المحمّلة ويمنع أعمال auto-reply/extension من تجويع الأجنحة غير المرتبطة.
  - لا يزال `pnpm test --watch` يستخدم مخطط المشروع الأصلي في `vitest.config.ts`، لأن حلقة مراقبة متعددة الـ shard غير عملية.
  - يقوم `pnpm test` و`pnpm test:watch` و`pnpm test:perf:imports` بتوجيه أهداف الملفات/الأدلة الصريحة عبر المسارات المحددة أولًا، لذا فإن `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` يتجنب تكلفة بدء root project الكامل.
  - يوسّع `pnpm test:changed` مسارات git المتغيرة إلى هذه المسارات المحددة نفسها عندما يمس الفرق فقط ملفات المصدر/الاختبار القابلة للتوجيه؛ أما تعديلات الإعداد/التهيئة فتعود إلى إعادة التشغيل الأوسع لمشروع الجذر.
  - يتم أيضًا توجيه اختبارات `plugin-sdk` و`commands` المحددة عبر مسارات خفيفة مخصصة تتخطى `test/setup-openclaw-runtime.ts`؛ أما الملفات الثقيلة ذات الحالة/وقت التشغيل فتبقى على المسارات الحالية.
  - كما أن ملفات المصدر المساعدة المحددة في `plugin-sdk` و`commands` تربط تشغيل وضع التغييرات باختبارات شقيقة صريحة ضمن تلك المسارات الخفيفة، بحيث تتجنب تعديلات المساعدات إعادة تشغيل الجناح الثقيل الكامل لذلك الدليل.
  - أصبح لدى `auto-reply` الآن ثلاثة buckets مخصصة: مساعدات core من المستوى الأعلى، واختبارات integration العليا `reply.*`، وشجرة `src/auto-reply/reply/**`. وهذا يبقي أثقل أعمال reply harness بعيدًا عن اختبارات الحالة/الأجزاء/الرموز الأرخص.
- ملاحظة المشغّل المضمن:
  - عندما تغيّر مدخلات اكتشاف message-tool أو سياق وقت تشغيل compaction،
    حافظ على مستويي التغطية كليهما.
  - أضف اختبارات تراجعية مركزة للمساعدات عند حدود التوجيه/التطبيع الخالصة.
  - واحرص أيضًا على بقاء أجنحة integration الخاصة بالمشغّل المضمن سليمة:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`،
    و`src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`، و
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - تتحقق هذه الأجنحة من أن المعرّفات ذات النطاق وسلوك compaction لا يزالان يمران
    عبر المسارات الحقيقية `run.ts` / `compact.ts`؛ ولا تُعد اختبارات المساعدات فقط
    بديلًا كافيًا عن مسارات integration هذه.
- ملاحظة الـ pool:
  - أصبح إعداد Vitest الأساسي يستخدم `threads` افتراضيًا.
  - كما أن إعداد Vitest المشترك يثبت `isolate: false` ويستخدم المشغّل غير المعزول عبر مشاريع الجذر وe2e وlive.
  - يحتفظ مسار UI الجذري بإعداد `jsdom` والمُحسّن الخاص به، لكنه يعمل الآن أيضًا على المشغّل المشترك غير المعزول.
  - يرث كل shard في `pnpm test` الافتراضيات نفسها `threads` + `isolate: false` من إعداد Vitest المشترك.
  - يضيف المشغّل المشترك `scripts/run-vitest.mjs` الآن أيضًا `--no-maglev` لعمليات Node الفرعية الخاصة بـ Vitest افتراضيًا لتقليل اهتزازات ترجمة V8 أثناء التشغيلات المحلية الكبيرة. اضبط `OPENCLAW_VITEST_ENABLE_MAGLEV=1` إذا كنت بحاجة إلى المقارنة مع سلوك V8 القياسي.
- ملاحظة التكرار المحلي السريع:
  - يوجّه `pnpm test:changed` عبر المسارات المحددة عندما ترتبط المسارات المتغيرة بوضوح بجناح أصغر.
  - يحتفظ كل من `pnpm test:max` و`pnpm test:changed:max` بسلوك التوجيه نفسه، فقط مع حد أعلى أكبر للعمال.
  - أصبح الضبط التلقائي لعدد العمال محليًا متحفظًا عمدًا الآن ويتراجع أيضًا عندما يكون متوسط حمل المضيف مرتفعًا بالفعل، بحيث تسبب تشغيلات Vitest المتعددة المتزامنة ضررًا أقل افتراضيًا.
  - يضع إعداد Vitest الأساسي المشاريع/ملفات الإعداد كـ `forceRerunTriggers` بحيث تبقى إعادة التشغيل في وضع التغييرات صحيحة عند تغير أسلاك الاختبار.
  - يُبقي الإعداد `OPENCLAW_VITEST_FS_MODULE_CACHE` مفعّلًا على المضيفين المدعومين؛ اضبط `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` إذا أردت موقع تخزين مؤقت صريحًا واحدًا لأغراض profiling المباشر.
- ملاحظة تصحيح الأداء:
  - يفعّل `pnpm test:perf:imports` تقارير مدة الاستيراد في Vitest بالإضافة إلى مخرجات تفصيل الاستيراد.
  - ويقصر `pnpm test:perf:imports:changed` عرض profiling نفسه على الملفات المتغيرة منذ `origin/main`.
- يقارن `pnpm test:perf:changed:bench -- --ref <git-ref>` بين `test:changed` الموجّه ومسار root-project الأصلي لذلك الفرق الملتزم ويطبع زمن التنفيذ إضافة إلى macOS max RSS.
- يقوم `pnpm test:perf:changed:bench -- --worktree` بقياس الشجرة المتسخة الحالية عبر توجيه قائمة الملفات المتغيرة من خلال `scripts/test-projects.mjs` وإعداد root Vitest.
  - يكتب `pnpm test:perf:profile:main` ملف تعريف CPU للخيط الرئيسي لوقت بدء Vitest/Vite وأعباء التحويل.
  - ويكتب `pnpm test:perf:profile:runner` ملفات تعريف CPU+heap للمشغّل لجناح unit مع تعطيل التوازي على مستوى الملفات.

### E2E ‏(فحص gateway الأساسي)

- الأمر: `pnpm test:e2e`
- الإعداد: `vitest.e2e.config.ts`
- الملفات: `src/**/*.e2e.test.ts` و`test/**/*.e2e.test.ts`
- افتراضيات وقت التشغيل:
  - يستخدم Vitest ‏`threads` مع `isolate: false`، بما يطابق بقية المستودع.
  - يستخدم عمالًا تكيفيين (CI: حتى 2، محليًا: 1 افتراضيًا).
  - يعمل في الوضع الصامت افتراضيًا لتقليل كلفة I/O الخاصة بالطرفية.
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_WORKERS=<n>` لفرض عدد العمال (بحد أقصى 16).
  - `OPENCLAW_E2E_VERBOSE=1` لإعادة تفعيل مخرجات الطرفية التفصيلية.
- النطاق:
  - سلوك end-to-end لـ gateway متعدد المثيلات
  - أسطح WebSocket/HTTP، واقتران العقد، والشبكات الأثقل
- التوقعات:
  - يعمل في CI ‏(عند تفعيله في المسار)
  - لا يتطلب مفاتيح حقيقية
  - يحتوي على أجزاء متحركة أكثر من اختبارات unit ‏(وقد يكون أبطأ)

### E2E: فحص OpenShell backend الأساسي

- الأمر: `pnpm test:e2e:openshell`
- الملف: `test/openshell-sandbox.e2e.test.ts`
- النطاق:
  - يبدأ OpenShell gateway معزولًا على المضيف عبر Docker
  - ينشئ sandbox من Dockerfile محلي مؤقت
  - يختبر OpenClaw OpenShell backend عبر `sandbox ssh-config` وSSH exec حقيقيين
  - يتحقق من سلوك نظام الملفات canonical البعيد عبر جسر sandbox fs
- التوقعات:
  - اختيارية فقط؛ ليست جزءًا من التشغيل الافتراضي `pnpm test:e2e`
  - تتطلب CLI محليًا لـ `openshell` بالإضافة إلى Docker daemon عامل
  - تستخدم `HOME` / `XDG_CONFIG_HOME` معزولين، ثم تدمر test gateway وsandbox
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_OPENSHELL=1` لتفعيل الاختبار عند تشغيل جناح e2e الأوسع يدويًا
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` للإشارة إلى ملف CLI تنفيذي غير افتراضي أو wrapper script

### Live ‏(موفّرون حقيقيون + نماذج حقيقية)

- الأمر: `pnpm test:live`
- الإعداد: `vitest.live.config.ts`
- الملفات: `src/**/*.live.test.ts`
- الافتراضي: **مفعّل** بواسطة `pnpm test:live` ‏(يضبط `OPENCLAW_LIVE_TEST=1`)
- النطاق:
  - "هل يعمل هذا الموفّر/النموذج فعلًا _اليوم_ باستخدام بيانات اعتماد حقيقية؟"
  - التقاط تغييرات تنسيق الموفّر، وخصائص استدعاء الأدوات، ومشكلات المصادقة، وسلوك حدود المعدل
- التوقعات:
  - ليست مستقرة في CI بطبيعتها (شبكات حقيقية، وسياسات موفّرين حقيقية، وحصص، وانقطاعات)
  - تكلف مالًا / تستخدم حدود المعدل
  - يُفضَّل تشغيل مجموعات فرعية ضيقة بدلًا من "كل شيء"
- تقوم التشغيلات الحية بتحميل `~/.profile` لالتقاط مفاتيح API المفقودة.
- افتراضيًا، لا تزال التشغيلات الحية تعزل `HOME` وتنسخ مواد config/auth إلى home اختبار مؤقت حتى لا تتمكن fixtures الخاصة بـ unit من تعديل `~/.openclaw` الحقيقي.
- اضبط `OPENCLAW_LIVE_USE_REAL_HOME=1` فقط عندما تحتاج عمدًا إلى أن تستخدم الاختبارات الحية دليل home الحقيقي.
- أصبح `pnpm test:live` افتراضيًا الآن أكثر هدوءًا: فهو يبقي على مخرجات التقدم `[live] ...`، لكنه يخفي إشعار `~/.profile` الإضافي ويكتم سجلات تمهيد gateway وضجيج Bonjour. اضبط `OPENCLAW_LIVE_TEST_QUIET=0` إذا أردت استعادة سجلات البدء الكاملة.
- تدوير مفاتيح API ‏(خاص بكل موفّر): اضبط `*_API_KEYS` بتنسيق comma/semicolon أو `*_API_KEY_1` و`*_API_KEY_2` (مثل `OPENAI_API_KEYS` و`ANTHROPIC_API_KEYS` و`GEMINI_API_KEYS`) أو تجاوزًا لكل تشغيل حي عبر `OPENCLAW_LIVE_*_KEY`؛ وتعاد محاولة الاختبارات عند استجابات حدود المعدل.
- مخرجات التقدم/heartbeat:
  - تُصدر الأجنحة الحية الآن أسطر التقدم إلى stderr بحيث تبقى استدعاءات الموفّر الطويلة مرئية النشاط حتى عندما يكون التقاط طرفية Vitest هادئًا.
  - يعطل `vitest.live.config.ts` اعتراض الطرفية في Vitest بحيث تُبث أسطر تقدم الموفّر/gateway مباشرة أثناء التشغيلات الحية.
  - اضبط heartbeat للنماذج المباشرة عبر `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - واضبط heartbeat الخاص بـ gateway/probe عبر `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## أي جناح يجب أن أشغّل؟

استخدم جدول القرار هذا:

- عند تعديل المنطق/الاختبارات: شغّل `pnpm test` ‏(و`pnpm test:coverage` إذا غيّرت الكثير)
- عند لمس شبكات gateway / بروتوكول WS / الاقتران: أضف `pnpm test:e2e`
- عند تصحيح "البوت الخاص بي متوقف" / أعطال خاصة بالموفّر / استدعاء الأدوات: شغّل `pnpm test:live` مضيقًا

## Live: مسح قدرات عقدة Android

- الاختبار: `src/gateway/android-node.capabilities.live.test.ts`
- السكربت: `pnpm android:test:integration`
- الهدف: استدعاء **كل أمر مُعلن عنه حاليًا** بواسطة عقدة Android متصلة والتحقق من سلوك عقدة الأمر.
- النطاق:
  - إعداد مسبق/يدوي (لا يقوم الجناح بتثبيت التطبيق أو تشغيله أو إقرانه).
  - تحقق gateway `node.invoke` أمرًا بأمر لعقدة Android المحددة.
- الإعداد المسبق المطلوب:
  - أن يكون تطبيق Android متصلًا ومقترنًا بالفعل مع gateway.
  - إبقاء التطبيق في الواجهة الأمامية.
  - منح الأذونات/الموافقة على الالتقاط للقدرات التي تتوقع نجاحها.
- تجاوزات الهدف الاختيارية:
  - `OPENCLAW_ANDROID_NODE_ID` أو `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- تفاصيل الإعداد الكاملة لـ Android: [Android App](/ar/platforms/android)

## Live: فحص النموذج الأساسي (مفاتيح الملفات الشخصية)

تنقسم الاختبارات الحية إلى طبقتين حتى نتمكن من عزل الأعطال:

- "النموذج المباشر" يخبرنا ما إذا كان الموفّر/النموذج قادرًا على الإجابة أصلًا باستخدام المفتاح المعطى.
- "فحص Gateway الأساسي" يخبرنا ما إذا كانت منظومة gateway+agent الكاملة تعمل لذلك النموذج (الجلسات، والسجل، والأدوات، وسياسة sandbox، وما إلى ذلك).

### الطبقة 1: إكمال مباشر للنموذج (بدون gateway)

- الاختبار: `src/agents/models.profiles.live.test.ts`
- الهدف:
  - تعداد النماذج المكتشفة
  - استخدام `getApiKeyForModel` لاختيار النماذج التي لديك بيانات اعتماد لها
  - تشغيل إكمال صغير لكل نموذج (واختبارات تراجعية مستهدفة عند الحاجة)
- كيفية التفعيل:
  - `pnpm test:live` ‏(أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- اضبط `OPENCLAW_LIVE_MODELS=modern` ‏(أو `all`، وهو اسم بديل لـ modern) لتشغيل هذا الجناح فعليًا؛ وإلا فسيُتخطى ليبقي `pnpm test:live` مركزًا على فحص gateway الأساسي
- كيفية اختيار النماذج:
  - `OPENCLAW_LIVE_MODELS=modern` لتشغيل قائمة السماح الحديثة (Opus/Sonnet 4.6+، وGPT-5.x + Codex، وGemini 3، وGLM 4.7، وMiniMax M2.7، وGrok 4)
  - `OPENCLAW_LIVE_MODELS=all` هو اسم بديل لقائمة السماح الحديثة
  - أو `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` ‏(قائمة سماح مفصولة بفواصل)
- كيفية اختيار الموفّرين:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` ‏(قائمة سماح مفصولة بفواصل)
- من أين تأتي المفاتيح:
  - افتراضيًا: من مخزن الملفات الشخصية وبدائل البيئة
  - اضبط `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض **مخزن الملفات الشخصية** فقط
- لماذا هذا موجود:
  - يفصل بين "API الموفّر معطوب / المفتاح غير صالح" و"منظومة gateway agent معطوبة"
  - يحتوي اختبارات تراجعية صغيرة ومعزولة (مثال: تدفقات replay + tool-call الخاصة بـ OpenAI Responses/Codex Responses reasoning)

### الطبقة 2: فحص Gateway + وكيل dev الأساسي (ما الذي يفعله "@openclaw" فعليًا)

- الاختبار: `src/gateway/gateway-models.profiles.live.test.ts`
- الهدف:
  - تشغيل gateway داخل العملية
  - إنشاء/ترقيع جلسة `agent:dev:*` ‏(مع تجاوز النموذج لكل تشغيل)
  - التكرار عبر النماذج ذات المفاتيح والتحقق من:
    - استجابة "ذات معنى" ‏(من دون أدوات)
    - أن استدعاء أداة حقيقي يعمل (`read` probe)
    - probes إضافية اختيارية للأدوات (`exec+read` probe)
    - استمرار عمل مسارات OpenAI التراجعية (استدعاء-أداة-فقط → متابعة)
- تفاصيل الـ probe ‏(حتى تتمكن من شرح الأعطال بسرعة):
  - probe ‏`read`: يكتب الاختبار ملف nonce في workspace ويطلب من الوكيل `read` له وإرجاع nonce.
  - probe ‏`exec+read`: يطلب الاختبار من الوكيل كتابة nonce عبر `exec` في ملف مؤقت، ثم `read` له.
  - probe الصورة: يرفق الاختبار PNG مولدًا (`cat` + رمز عشوائي) ويتوقع من النموذج إرجاع `cat <CODE>`.
  - مرجع التنفيذ: `src/gateway/gateway-models.profiles.live.test.ts` و`src/gateway/live-image-probe.ts`.
- كيفية التفعيل:
  - `pnpm test:live` ‏(أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- كيفية اختيار النماذج:
  - الافتراضي: قائمة السماح الحديثة (Opus/Sonnet 4.6+، وGPT-5.x + Codex، وGemini 3، وGLM 4.7، وMiniMax M2.7، وGrok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` هو اسم بديل لقائمة السماح الحديثة
  - أو اضبط `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` ‏(أو قائمة مفصولة بفواصل) للتضييق
- كيفية اختيار الموفّرين (لتجنب "كل شيء عبر OpenRouter"):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` ‏(قائمة سماح مفصولة بفواصل)
- تكون أدوات وصور probe مفعلة دائمًا في هذا الاختبار الحي:
  - probe ‏`read` + probe ‏`exec+read` ‏(ضغط الأدوات)
  - يعمل probe الصورة عندما يعلن النموذج دعمه لمدخلات الصور
  - التدفق (على مستوى عالٍ):
    - يولد الاختبار PNG صغيرًا يحمل "CAT" + رمزًا عشوائيًا (`src/gateway/live-image-probe.ts`)
    - يرسله عبر `agent` ‏`attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - يحلل Gateway المرفقات إلى `images[]` ‏(`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - يمرر الوكيل المضمن رسالة مستخدم متعددة الوسائط إلى النموذج
    - التحقق: تحتوي الاستجابة على `cat` + الرمز (سماحية OCR: يُسمح بأخطاء طفيفة)

نصيحة: لمعرفة ما الذي يمكنك اختباره على جهازك (ومعرّفات `provider/model` الدقيقة)، شغّل:

```bash
openclaw models list
openclaw models list --json
```

## Live: فحص CLI backend الأساسي (Claude أو Codex أو Gemini أو غيرها من CLI المحلية)

- الاختبار: `src/gateway/gateway-cli-backend.live.test.ts`
- الهدف: التحقق من منظومة Gateway + agent باستخدام CLI backend محلي، من دون لمس الإعدادات الافتراضية.
- تعيش افتراضيات الفحص الخاصة بكل backend مع تعريف `cli-backend.ts` العائد إلى extension المالكة.
- التفعيل:
  - `pnpm test:live` ‏(أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- الافتراضيات:
  - الموفّر/النموذج الافتراضي: `claude-cli/claude-sonnet-4-6`
  - تأتي سلوكيات command/args/image من بيانات plugin metadata الخاصة بـ CLI backend المالكة.
- التجاوزات (اختيارية):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` لإرسال مرفق صورة حقيقي (يتم حقن المسارات في المطالبة).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` لتمرير مسارات ملفات الصور كوسائط CLI بدلًا من حقنها في المطالبة.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` ‏(أو `"list"`) للتحكم في كيفية تمرير وسائط الصورة عندما يكون `IMAGE_ARG` مضبوطًا.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` لإرسال دورة ثانية والتحقق من تدفق الاستئناف.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` لتعطيل probe الاستمرارية الافتراضي داخل الجلسة نفسها من Claude Sonnet إلى Opus ‏(اضبطه إلى `1` لفرضه عندما يدعم النموذج المحدد هدف تبديل).

مثال:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

وصفة Docker:

```bash
pnpm test:docker:live-cli-backend
```

وصفات Docker لموفّر واحد:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

ملاحظات:

- يعيش مشغّل Docker في `scripts/test-live-cli-backend-docker.sh`.
- يشغّل فحص CLI-backend الحي داخل صورة Docker الخاصة بالمستودع بوصف المستخدم غير الجذر `node`.
- يحل بيانات smoke metadata الخاصة بـ CLI من extension المالكة، ثم يثبت حزمة Linux CLI المطابقة (`@anthropic-ai/claude-code` أو `@openai/codex` أو `@google/gemini-cli`) في prefix قابل للكتابة ومخبأ مؤقتًا في `OPENCLAW_DOCKER_CLI_TOOLS_DIR` ‏(الافتراضي: `~/.cache/openclaw/docker-cli-tools`).
- يمارس فحص CLI-backend الحي الآن التدفق نفسه من طرف إلى طرف لـ Claude وCodex وGemini: دورة نصية، ودورة تصنيف صور، ثم استدعاء أداة MCP ‏`cron` يتم التحقق منه عبر gateway CLI.
- يقوم الفحص الافتراضي لـ Claude أيضًا بترقيع الجلسة من Sonnet إلى Opus ويتحقق من أن الجلسة المستأنفة لا تزال تتذكر ملاحظة سابقة.

## Live: فحص ACP bind الأساسي (`/acp spawn ... --bind here`)

- الاختبار: `src/gateway/gateway-acp-bind.live.test.ts`
- الهدف: التحقق من تدفق ربط المحادثة ACP الحقيقي مع وكيل ACP حي:
  - إرسال `/acp spawn <agent> --bind here`
  - ربط محادثة channel اصطناعية في مكانها
  - إرسال متابعة عادية على تلك المحادثة نفسها
  - التحقق من أن المتابعة تهبط في transcript الخاص بجلسة ACP المرتبطة
- التفعيل:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- الافتراضيات:
  - وكلاء ACP في Docker: ‏`claude,codex,gemini`
  - وكيل ACP لتشغيل `pnpm test:live ...` المباشر: `claude`
  - قناة اصطناعية: سياق محادثة بأسلوب Slack DM
  - ACP backend: `acpx`
- التجاوزات:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- ملاحظات:
  - يستخدم هذا المسار سطح gateway ‏`chat.send` مع حقول originating-route اصطناعية خاصة بالمسؤول فقط بحيث تتمكن الاختبارات من إرفاق سياق message-channel من دون التظاهر بتوصيل خارجي.
  - عندما لا يكون `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` مضبوطًا، يستخدم الاختبار سجل الوكلاء المضمن المدمج في plugin ‏`acpx` لوكيل ACP التجريبي المحدد.

مثال:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

وصفة Docker:

```bash
pnpm test:docker:live-acp-bind
```

وصفات Docker لوكيل واحد:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

ملاحظات Docker:

- يعيش مشغّل Docker في `scripts/test-live-acp-bind-docker.sh`.
- يشغّل فحص ACP bind الأساسي افتراضيًا على كل وكلاء CLI الحية المدعومة بالتتابع: `claude` ثم `codex` ثم `gemini`.
- استخدم `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude` أو `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` أو `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` لتضييق المصفوفة.
- يقوم بتحميل `~/.profile`، وتهيئة مواد مصادقة CLI المطابقة داخل الحاوية، وتثبيت `acpx` في npm prefix قابل للكتابة، ثم يثبت CLI الحي المطلوب (`@anthropic-ai/claude-code` أو `@openai/codex` أو `@google/gemini-cli`) إذا كان مفقودًا.
- داخل Docker، يضبط المشغّل `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` بحيث يحتفظ acpx بمتغيرات بيئة الموفّر القادمة من profile المحمّل ومتاحة لـ harness CLI الابن.

### وصفات live الموصى بها

قوائم السماح الضيقة والصريحة هي الأسرع والأقل تعرضًا للتعطل:

- نموذج واحد، مباشر (بدون gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- نموذج واحد، فحص gateway الأساسي:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- استدعاء الأدوات عبر عدة موفّرين:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- التركيز على Google ‏(مفتاح Gemini API + Antigravity):
  - Gemini ‏(مفتاح API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity ‏(OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

ملاحظات:

- يستخدم `google/...` واجهة Gemini API ‏(مفتاح API).
- يستخدم `google-antigravity/...` جسر Antigravity OAuth ‏(نقطة نهاية وكيل بأسلوب Cloud Code Assist).
- يستخدم `google-gemini-cli/...` Gemini CLI المحلي على جهازك (مصادقة منفصلة وخصائص أدوات مختلفة).
- Gemini API مقابل Gemini CLI:
  - API: يستدعي OpenClaw واجهة Gemini API المستضافة من Google عبر HTTP ‏(مصادقة مفتاح API / profile)؛ وهذا ما يقصده معظم المستخدمين عندما يقولون "Gemini".
  - CLI: يستدعي OpenClaw ملف `gemini` التنفيذي المحلي؛ وله مصادقته الخاصة وقد يتصرف بشكل مختلف (البث/دعم الأدوات/اختلاف الإصدارات).

## Live: مصفوفة النماذج (ما الذي نغطيه)

لا توجد "قائمة نماذج CI" ثابتة (إذ إن live اختيارية)، لكن هذه هي النماذج **الموصى بها** للتغطية بانتظام على جهاز تطوير يملك مفاتيح.

### مجموعة الفحص الحديثة (استدعاء الأدوات + الصور)

هذا هو تشغيل "النماذج الشائعة" الذي نتوقع أن يستمر في العمل:

- OpenAI ‏(غير Codex): ‏`openai/gpt-5.4` ‏(اختياري: `openai/gpt-5.4-mini`)
- OpenAI Codex: ‏`openai-codex/gpt-5.4`
- Anthropic: ‏`anthropic/claude-opus-4-6` ‏(أو `anthropic/claude-sonnet-4-6`)
- Google ‏(Gemini API): ‏`google/gemini-3.1-pro-preview` و`google/gemini-3-flash-preview` ‏(تجنب نماذج Gemini 2.x الأقدم)
- Google ‏(Antigravity): ‏`google-antigravity/claude-opus-4-6-thinking` و`google-antigravity/gemini-3-flash`
- Z.AI ‏(GLM): ‏`zai/glm-4.7`
- MiniMax: ‏`minimax/MiniMax-M2.7`

شغّل فحص gateway الأساسي مع الأدوات + الصورة:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### الأساس: استدعاء الأدوات (Read + Exec اختياري)

اختر واحدة على الأقل من كل عائلة موفّرين:

- OpenAI: ‏`openai/gpt-5.4` ‏(أو `openai/gpt-5.4-mini`)
- Anthropic: ‏`anthropic/claude-opus-4-6` ‏(أو `anthropic/claude-sonnet-4-6`)
- Google: ‏`google/gemini-3-flash-preview` ‏(أو `google/gemini-3.1-pro-preview`)
- Z.AI ‏(GLM): ‏`zai/glm-4.7`
- MiniMax: ‏`minimax/MiniMax-M2.7`

تغطية إضافية اختيارية (من الجيد توفرها):

- xAI: ‏`xai/grok-4` ‏(أو أحدث إصدار متاح)
- Mistral: ‏`mistral/`… ‏(اختر نموذجًا واحدًا قادرًا على "tools" لديك مفعّلًا)
- Cerebras: ‏`cerebras/`… ‏(إذا كان لديك وصول)
- LM Studio: ‏`lmstudio/`… ‏(محلي؛ يعتمد استدعاء الأدوات على وضع API)

### الرؤية: إرسال صورة (مرفق ← رسالة متعددة الوسائط)

ضمّن نموذجًا واحدًا على الأقل قادرًا على الصور في `OPENCLAW_LIVE_GATEWAY_MODELS` ‏(Claude/Gemini/OpenAI ذات القدرات البصرية، وغيرها) لتفعيل probe الصورة.

### المجمّعات / البوابات البديلة

إذا كانت لديك مفاتيح مفعلة، فنحن ندعم أيضًا الاختبار عبر:

- OpenRouter: ‏`openrouter/...` ‏(مئات النماذج؛ استخدم `openclaw models scan` للعثور على مرشحين قادرين على الأدوات+الصور)
- OpenCode: ‏`opencode/...` لـ Zen و`opencode-go/...` لـ Go ‏(المصادقة عبر `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

موفّرون آخرون يمكنك تضمينهم في مصفوفة live ‏(إذا كانت لديك بيانات الاعتماد/الإعدادات):

- المضمنة: `openai` و`openai-codex` و`anthropic` و`google` و`google-vertex` و`google-antigravity` و`google-gemini-cli` و`zai` و`openrouter` و`opencode` و`opencode-go` و`xai` و`groq` و`cerebras` و`mistral` و`github-copilot`
- عبر `models.providers` ‏(نقاط نهاية مخصصة): `minimax` ‏(سحابي/API)، بالإضافة إلى أي proxy متوافق مع OpenAI/Anthropic ‏(LM Studio، وvLLM، وLiteLLM، وما إلى ذلك)

نصيحة: لا تحاول ترميز "كل النماذج" في الوثائق بشكل صلب. القائمة المرجعية الموثوقة هي ما تُرجعه `discoverModels(...)` على جهازك + أي مفاتيح متاحة.

## بيانات الاعتماد (لا تُلتزم أبدًا)

تكتشف الاختبارات الحية بيانات الاعتماد بالطريقة نفسها التي يفعلها CLI. والنتائج العملية لذلك:

- إذا كان CLI يعمل، فيجب أن تجد الاختبارات الحية المفاتيح نفسها.
- إذا قال اختبار حي "لا توجد بيانات اعتماد"، فقم بالتصحيح بالطريقة نفسها التي تُصحّح بها `openclaw models list` / اختيار النموذج.

- ملفات المصادقة الشخصية لكل وكيل: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` ‏(هذا ما تعنيه "profile keys" في الاختبارات الحية)
- الإعدادات: `~/.openclaw/openclaw.json` ‏(أو `OPENCLAW_CONFIG_PATH`)
- دليل الحالة القديم: `~/.openclaw/credentials/` ‏(يُنسخ إلى home الحي المرحلي عند وجوده، لكن ليس مخزن مفاتيح الملفات الشخصية الرئيسي)
- تقوم التشغيلات الحية المحلية بنسخ الإعدادات النشطة وملفات `auth-profiles.json` لكل وكيل و`credentials/` القديمة وأدلة مصادقة CLI الخارجية المدعومة إلى home اختبار مؤقت افتراضيًا؛ وتتخطى homes الحية المرحلية `workspace/` و`sandboxes/`، كما تُزال تجاوزات المسارات `agents.*.workspace` / `agentDir` بحيث تبقى probes بعيدة عن workspace الحقيقي للمضيف.

إذا كنت تريد الاعتماد على مفاتيح البيئة (مثلًا المصدّرة في `~/.profile`)، فشغّل الاختبارات المحلية بعد `source ~/.profile`، أو استخدم مشغلات Docker أدناه (يمكنها تركيب `~/.profile` داخل الحاوية).

## Deepgram live ‏(نسخ صوتي)

- الاختبار: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- التفعيل: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- الاختبار: `src/agents/byteplus.live.test.ts`
- التفعيل: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- تجاوز النموذج الاختياري: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- الاختبار: `extensions/comfy/comfy.live.test.ts`
- التفعيل: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- النطاق:
  - يختبر مسارات comfy المضمنة للصور والفيديو و`music_generate`
  - يتخطى كل قدرة ما لم يكن `models.providers.comfy.<capability>` مهيأ
  - مفيد بعد تعديل إرسال workflow في comfy، أو polling، أو التنزيلات، أو تسجيل plugin

## Image generation live

- الاختبار: `src/image-generation/runtime.live.test.ts`
- الأمر: `pnpm test:live src/image-generation/runtime.live.test.ts`
- الحاضنة: `pnpm test:live:media image`
- النطاق:
  - يعدد كل provider plugin مسجل لتوليد الصور
  - يحمّل متغيرات بيئة الموفّر المفقودة من login shell الخاص بك (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/البيئية قبل ملفات المصادقة الشخصية المخزنة افتراضيًا، حتى لا تحجب مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات الاعتماد الحقيقية في shell
  - يتخطى الموفّرين الذين لا يملكون مصادقة/ملفًا شخصيًا/نموذجًا صالحًا
  - يشغّل متغيرات توليد الصور القياسية عبر قدرة وقت التشغيل المشتركة:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- الموفّرون المضمنون الحاليون المشمولون:
  - `openai`
  - `google`
- تضييق اختياري:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- سلوك مصادقة اختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن الملفات الشخصية وتجاهل التجاوزات البيئية فقط

## Music generation live

- الاختبار: `extensions/music-generation-providers.live.test.ts`
- التفعيل: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- الحاضنة: `pnpm test:live:media music`
- النطاق:
  - يختبر مسار music-generation provider المضمن المشترك
  - يغطي حاليًا Google وMiniMax
  - يحمّل متغيرات بيئة الموفّر من login shell الخاص بك (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/البيئية قبل ملفات المصادقة الشخصية المخزنة افتراضيًا، حتى لا تحجب مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات الاعتماد الحقيقية في shell
  - يتخطى الموفّرين الذين لا يملكون مصادقة/ملفًا شخصيًا/نموذجًا صالحًا
  - يشغّل وضعي وقت التشغيل المعلنين كليهما عند توفرهما:
    - `generate` مع مدخلات تعتمد على prompt فقط
    - `edit` عندما يعلن الموفّر `capabilities.edit.enabled`
  - التغطية الحالية في المسار المشترك:
    - `google`: ‏`generate`، ‏`edit`
    - `minimax`: ‏`generate`
    - `comfy`: ملف Comfy حي منفصل، وليس هذا المسح المشترك
- تضييق اختياري:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- سلوك مصادقة اختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن الملفات الشخصية وتجاهل التجاوزات البيئية فقط

## Video generation live

- الاختبار: `extensions/video-generation-providers.live.test.ts`
- التفعيل: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- الحاضنة: `pnpm test:live:media video`
- النطاق:
  - يختبر مسار video-generation provider المضمن المشترك
  - يحمّل متغيرات بيئة الموفّر من login shell الخاص بك (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/البيئية قبل ملفات المصادقة الشخصية المخزنة افتراضيًا، حتى لا تحجب مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات الاعتماد الحقيقية في shell
  - يتخطى الموفّرين الذين لا يملكون مصادقة/ملفًا شخصيًا/نموذجًا صالحًا
  - يشغّل وضعي وقت التشغيل المعلنين كليهما عند توفرهما:
    - `generate` مع مدخلات تعتمد على prompt فقط
    - `imageToVideo` عندما يعلن الموفّر `capabilities.imageToVideo.enabled` ويقبل الموفّر/النموذج المحدد إدخال الصور المحلية المعتمد على buffer في المسح المشترك
    - `videoToVideo` عندما يعلن الموفّر `capabilities.videoToVideo.enabled` ويقبل الموفّر/النموذج المحدد إدخال الفيديو المحلي المعتمد على buffer في المسح المشترك
  - موفّرو `imageToVideo` المعلن عنهم حاليًا لكن المتخطَّين في المسح المشترك:
    - `vydra` لأن `veo3` المضمن نصي فقط و`kling` المضمن يتطلب URL صورة بعيدًا
  - تغطية Vydra الخاصة بكل موفّر:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - يشغّل هذا الملف `veo3` من نص إلى فيديو بالإضافة إلى مسار `kling` يستخدم fixture لعنوان URL صورة بعيد افتراضيًا
  - التغطية الحية الحالية لـ `videoToVideo`:
    - `runway` فقط عندما يكون النموذج المحدد هو `runway/gen4_aleph`
  - موفّرو `videoToVideo` المعلن عنهم حاليًا لكن المتخطَّين في المسح المشترك:
    - `alibaba` و`qwen` و`xai` لأن هذه المسارات تتطلب حاليًا عناوين URL مرجعية بعيدة `http(s)` / MP4
    - `google` لأن مسار Gemini/Veo المشترك الحالي يستخدم مدخلات محلية معتمدة على buffer وهذا المسار غير مقبول في المسح المشترك
    - `openai` لأن المسار المشترك الحالي يفتقر إلى ضمانات الوصول الخاصة بالمؤسسة لـ video inpaint/remix
- تضييق اختياري:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- سلوك مصادقة اختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن الملفات الشخصية وتجاهل التجاوزات البيئية فقط

## Media live harness

- الأمر: `pnpm test:live:media`
- الغرض:
  - يشغّل أجنحة image وmusic وvideo الحية المشتركة عبر نقطة دخول أصلية للمستودع
  - يحمّل تلقائيًا متغيرات بيئة الموفّر المفقودة من `~/.profile`
  - يضيّق تلقائيًا كل جناح إلى الموفّرين الذين لديهم حاليًا مصادقة قابلة للاستخدام افتراضيًا
  - يعيد استخدام `scripts/test-live.mjs`، بحيث يبقى سلوك heartbeat والوضع الهادئ متسقًا
- أمثلة:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## مشغلات Docker ‏(اختيارية لفحوصات "يعمل على Linux")

تنقسم مشغلات Docker هذه إلى فئتين:

- مشغلات النماذج الحية: يشغّل `test:docker:live-models` و`test:docker:live-gateway` فقط ملف live الخاص بمفاتيح الملفات الشخصية المطابق لهما داخل صورة Docker الخاصة بالمستودع (`src/agents/models.profiles.live.test.ts` و`src/gateway/gateway-models.profiles.live.test.ts`) مع تركيب دليل الإعدادات المحلي وworkspace ‏(وتحميل `~/.profile` إذا تم تركيبه). ونقطتا الدخول المحليتان المطابقتان هما `test:live:models-profiles` و`test:live:gateway-profiles`.
- تستخدم مشغلات Docker الحية افتراضيًا حدًا أصغر للفحص حتى يظل المسح الكامل في Docker عمليًا:
  يستخدم `test:docker:live-models` افتراضيًا `OPENCLAW_LIVE_MAX_MODELS=12`، و
  يستخدم `test:docker:live-gateway` افتراضيًا `OPENCLAW_LIVE_GATEWAY_SMOKE=1`،
  و`OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`،
  و`OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`، و
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. تجاوز متغيرات البيئة هذه عندما
  تريد عمدًا المسح الأكبر والأشمل.
- يقوم `test:docker:all` ببناء صورة Docker الحية مرة واحدة عبر `test:docker:live-build`، ثم يعيد استخدامها للمسارين الحيين في Docker.
- مشغلات الفحص داخل الحاويات: تقلع `test:docker:openwebui` و`test:docker:onboard` و`test:docker:gateway-network` و`test:docker:mcp-channels` و`test:docker:plugins` حاوية واحدة أو أكثر حقيقية وتتحقق من مسارات integration أعلى مستوى.

كما تقوم مشغلات Docker الخاصة بالنماذج الحية بتركيب homes مصادقة CLI المطلوبة فقط (أو جميعها عندما لا يكون التشغيل مضيقًا)، ثم تنسخها إلى home الحاوية قبل التشغيل حتى تتمكن OAuth الخاصة بـ external-CLI من تحديث الرموز دون تعديل مخزن مصادقة المضيف:

- النماذج المباشرة: `pnpm test:docker:live-models` ‏(السكربت: `scripts/test-live-models-docker.sh`)
- فحص ACP bind الأساسي: `pnpm test:docker:live-acp-bind` ‏(السكربت: `scripts/test-live-acp-bind-docker.sh`)
- فحص CLI backend الأساسي: `pnpm test:docker:live-cli-backend` ‏(السكربت: `scripts/test-live-cli-backend-docker.sh`)
- Gateway + وكيل dev: ‏`pnpm test:docker:live-gateway` ‏(السكربت: `scripts/test-live-gateway-models-docker.sh`)
- Open WebUI live smoke: ‏`pnpm test:docker:openwebui` ‏(السكربت: `scripts/e2e/openwebui-docker.sh`)
- معالج onboarding ‏(TTY، والتهيئة الكاملة): `pnpm test:docker:onboard` ‏(السكربت: `scripts/e2e/onboard-docker.sh`)
- شبكات Gateway ‏(حاويتان، ومصادقة WS + السلامة): `pnpm test:docker:gateway-network` ‏(السكربت: `scripts/e2e/gateway-network-docker.sh`)
- جسر قناة MCP ‏(Gateway مهيأ مسبقًا + stdio bridge + فحص Claude raw notification-frame): ‏`pnpm test:docker:mcp-channels` ‏(السكربت: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins ‏(فحص التثبيت + الاسم البديل `/plugin` + دلالات إعادة تشغيل Claude-bundle): ‏`pnpm test:docker:plugins` ‏(السكربت: `scripts/e2e/plugins-docker.sh`)

كما تقوم مشغلات Docker الخاصة بالنماذج الحية بتركيب النسخة الحالية من المستودع للقراءة فقط
وتهيئتها داخل دليل عمل مؤقت داخل الحاوية. وهذا يبقي صورة وقت التشغيل
نحيفة مع الاستمرار في تشغيل Vitest على المصدر/الإعدادات المحلية الدقيقة لديك.
وتتخطى خطوة التهيئة مخازن التخزين المؤقت المحلية الكبيرة ومخرجات بناء التطبيق مثل
`.pnpm-store` و`.worktrees` و`__openclaw_vitest__` وأدلة `.build` المحلية للتطبيق أو
مجلدات مخرجات Gradle، حتى لا تقضي تشغيلات Docker الحية دقائق في نسخ
أصول خاصة بالجهاز.
كما تضبط `OPENCLAW_SKIP_CHANNELS=1` حتى لا تبدأ probes الحية الخاصة بـ gateway
عمّال قنوات Telegram/Discord/... حقيقيين داخل الحاوية.
ولا يزال `test:docker:live-models` يشغّل `pnpm test:live`، لذا مرّر
`OPENCLAW_LIVE_GATEWAY_*` أيضًا عندما تحتاج إلى تضييق أو استبعاد تغطية gateway
الحية من مسار Docker ذاك.
أما `test:docker:openwebui` فهو فحص توافق أعلى مستوى: إذ يبدأ
حاوية OpenClaw gateway مع تفعيل نقاط نهاية HTTP المتوافقة مع OpenAI،
ثم يبدأ حاوية Open WebUI مثبتة ضد تلك gateway، ويسجل الدخول عبر
Open WebUI، ويتحقق من أن `/api/models` يعرض `openclaw/default`، ثم يرسل
طلب chat حقيقيًا عبر proxy ‏`/api/chat/completions` في Open WebUI.
قد يكون التشغيل الأول أبطأ بشكل ملحوظ لأن Docker قد يحتاج إلى سحب
صورة Open WebUI وقد يحتاج Open WebUI إلى إنهاء إعداد البدء البارد الخاص به.
يتوقع هذا المسار وجود مفتاح نموذج حي صالح، ويُعد `OPENCLAW_PROFILE_FILE`
‏(`~/.profile` افتراضيًا) الوسيلة الأساسية لتوفيره في التشغيلات المعتمدة على Docker.
تطبع التشغيلات الناجحة حمولة JSON صغيرة مثل `{ "ok": true, "model":
"openclaw/default", ... }`.
أما `test:docker:mcp-channels` فهو حتمي عمدًا ولا يحتاج إلى
حساب Telegram أو Discord أو iMessage حقيقي. فهو يقلع حاوية Gateway
مهيأة مسبقًا، ويبدأ حاوية ثانية تشغّل `openclaw mcp serve`، ثم
يتحقق من اكتشاف المحادثات الموجّهة، وقراءة transcripts، وmetadata الخاصة بالمرفقات،
وسلوك قائمة الأحداث الحية، وتوجيه الإرسال الصادر، وإشعارات القنوات +
الأذونات بأسلوب Claude عبر جسر stdio MCP الحقيقي. ويفحص تحقق الإشعارات
إطارات stdio MCP الخام مباشرة بحيث يتحقق الفحص مما يصدره الجسر فعليًا،
وليس فقط مما قد تكشفه مجموعة أدوات عميل معينة.

فحص ACP اليدوي للخيوط بلغة طبيعية (ليس ضمن CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- احتفظ بهذا السكربت لمسارات العمل الخاصة بالاختبارات التراجعية/التصحيح. فقد تكون هناك حاجة إليه مرة أخرى للتحقق من توجيه خيوط ACP، لذلك لا تحذفه.

متغيرات بيئة مفيدة:

- `OPENCLAW_CONFIG_DIR=...` ‏(الافتراضي: `~/.openclaw`) يُركّب إلى `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` ‏(الافتراضي: `~/.openclaw/workspace`) يُركّب إلى `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` ‏(الافتراضي: `~/.profile`) يُركّب إلى `/home/node/.profile` ويُحمّل قبل تشغيل الاختبارات
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` ‏(الافتراضي: `~/.cache/openclaw/docker-cli-tools`) يُركّب إلى `/home/node/.npm-global` لتثبيتات CLI المخبأة داخل Docker
- تُركّب أدلة/ملفات مصادقة CLI الخارجية تحت `$HOME` للقراءة فقط ضمن `/host-auth...`، ثم تُنسخ إلى `/home/node/...` قبل بدء الاختبارات
  - الأدلة الافتراضية: `.minimax`
  - الملفات الافتراضية: `~/.codex/auth.json` و`~/.codex/config.toml` و`.claude.json` و`~/.claude/.credentials.json` و`~/.claude/settings.json` و`~/.claude/settings.local.json`
  - التشغيلات المضيقة بحسب الموفّر تركّب فقط الأدلة/الملفات المطلوبة المستنتجة من `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - تجاوز يدويًا عبر `OPENCLAW_DOCKER_AUTH_DIRS=all` أو `OPENCLAW_DOCKER_AUTH_DIRS=none` أو قائمة مفصولة بفواصل مثل `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` لتضييق التشغيل
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` لتصفية الموفّرين داخل الحاوية
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لضمان أن بيانات الاعتماد تأتي من مخزن الملفات الشخصية (وليس من البيئة)
- `OPENCLAW_OPENWEBUI_MODEL=...` لاختيار النموذج الذي تعرضه gateway لفحص Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` لتجاوز مطالبة التحقق بالـ nonce المستخدمة في فحص Open WebUI
- `OPENWEBUI_IMAGE=...` لتجاوز وسم صورة Open WebUI المثبتة

## سلامة الوثائق

شغّل فحوصات الوثائق بعد تعديلها: `pnpm check:docs`.
وشغّل التحقق الكامل من anchors الخاصة بـ Mintlify عندما تحتاج إلى فحوصات عناوين داخل الصفحة أيضًا: `pnpm docs:check-links:anchors`.

## اختبار تراجعي دون اتصال (آمن على CI)

هذه اختبارات تراجعية "لمنظومة حقيقية" من دون موفّرين حقيقيين:

- استدعاء أدوات Gateway ‏(OpenAI وهمي، وgateway حقيقية + حلقة agent): ‏`src/gateway/gateway.test.ts` ‏(الحالة: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Gateway wizard ‏(‏`wizard.start`/`wizard.next` عبر WS، ويكتب config + auth المفروضة): ‏`src/gateway/gateway.test.ts` ‏(الحالة: "runs wizard over ws and writes auth token config")

## تقييمات موثوقية الوكيل (Skills)

لدينا بالفعل بعض الاختبارات الآمنة على CI التي تتصرف مثل "تقييمات موثوقية الوكيل":

- استدعاء أدوات وهمي عبر gateway الحقيقية + حلقة agent ‏(`src/gateway/gateway.test.ts`).
- تدفقات wizard من طرف إلى طرف تتحقق من توصيل الجلسة وآثار الإعدادات (`src/gateway/gateway.test.ts`).

ما الذي لا يزال مفقودًا لـ Skills ‏(انظر [Skills](/ar/tools/skills)):

- **اتخاذ القرار:** عندما تُسرد Skills في المطالبة، هل يختار الوكيل Skill الصحيحة (أو يتجنب غير ذات الصلة)؟
- **الامتثال:** هل يقرأ الوكيل `SKILL.md` قبل الاستخدام ويلتزم بالخطوات/الوسائط المطلوبة؟
- **عقود مسار العمل:** سيناريوهات متعددة الأدوار تتحقق من ترتيب الأدوات، واستمرار سجل الجلسة، وحدود sandbox.

يجب أن تبقى التقييمات المستقبلية حتمية أولًا:

- مشغّل سيناريوهات يستخدم موفّرين وهميين للتحقق من استدعاءات الأدوات + ترتيبها، وقراءات ملف skill، وتوصيل الجلسة.
- مجموعة صغيرة من السيناريوهات المركزة على skill ‏(استخدم مقابل تجنب، والبوابات، وحقن المطالبات).
- تقييمات حية اختيارية ‏(مفعلة اختياريًا، ومحكومة بمتغيرات البيئة) فقط بعد توفر الجناح الآمن على CI.

## اختبارات العقود (شكل plugin وchannel)

تتحقق اختبارات العقود من أن كل plugin وchannel مسجلين يطابقان
عقد الواجهة الخاص بهما. فهي تكرّر على جميع plugins المكتشفة وتشغّل مجموعة من
التحققات الخاصة بالشكل والسلوك. ويتخطى مسار unit الافتراضي `pnpm test`
هذه الملفات المشتركة الخاصة بالحدود والفحوصات الأساسية عمدًا؛ شغّل أوامر العقود صراحة
عندما تلمس أسطح channel أو provider المشتركة.

### الأوامر

- كل العقود: `pnpm test:contracts`
- عقود القنوات فقط: `pnpm test:contracts:channels`
- عقود الموفّرين فقط: `pnpm test:contracts:plugins`

### عقود القنوات

تقع في `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - الشكل الأساسي لـ plugin ‏(id، والاسم، والقدرات)
- **setup** - عقد معالج الإعداد
- **session-binding** - سلوك ربط الجلسة
- **outbound-payload** - بنية حمولة الرسالة
- **inbound** - معالجة الرسائل الواردة
- **actions** - معالجات إجراءات القناة
- **threading** - معالجة معرّف الخيط
- **directory** - API الدليل/القائمة
- **group-policy** - فرض سياسة المجموعات

### عقود حالة الموفّرين

تقع في `src/plugins/contracts/*.contract.test.ts`.

- **status** - probes حالة القناة
- **registry** - شكل سجل plugin

### عقود الموفّرين

تقع في `src/plugins/contracts/*.contract.test.ts`:

- **auth** - عقد تدفق المصادقة
- **auth-choice** - اختيار/انتقاء المصادقة
- **catalog** - API كتالوج النماذج
- **discovery** - اكتشاف plugin
- **loader** - تحميل plugin
- **runtime** - وقت تشغيل الموفّر
- **shape** - شكل/واجهة plugin
- **wizard** - معالج الإعداد

### متى تُشغَّل

- بعد تغيير صادرات `plugin-sdk` أو المسارات الفرعية
- بعد إضافة أو تعديل channel أو provider plugin
- بعد إعادة هيكلة تسجيل plugin أو اكتشافها

تعمل اختبارات العقود في CI ولا تتطلب مفاتيح API حقيقية.

## إضافة اختبارات تراجعية (إرشادات)

عندما تصلح مشكلة في موفّر/نموذج اكتُشفت في live:

- أضف اختبارًا تراجعيًا آمنًا على CI إن أمكن (موفّر وهمي/مُحاكى، أو التقط التحويل الدقيق لشكل الطلب)
- إذا كانت بطبيعتها حية فقط (حدود المعدل، وسياسات المصادقة)، فأبقِ الاختبار الحي ضيقًا ومفعّلًا اختياريًا عبر متغيرات البيئة
- افضّل استهداف أصغر طبقة تلتقط الخطأ:
  - خطأ في تحويل/إعادة تشغيل طلب الموفّر → اختبار النماذج المباشرة
  - خطأ في منظومة جلسة/سجل/أداة gateway → فحص gateway الحي الأساسي أو اختبار gateway mock آمن على CI
- حاجز اجتياز SecretRef:
  - يقوم `src/secrets/exec-secret-ref-id-parity.test.ts` باشتقاق هدف مُعين واحد لكل فئة SecretRef من metadata الخاصة بالسجل (`listSecretTargetRegistryEntries()`)، ثم يتحقق من رفض معرّفات exec لمقاطع الاجتياز.
  - إذا أضفت عائلة هدف SecretRef جديدة ضمن `includeInPlan` في `src/secrets/target-registry-data.ts`، فحدّث `classifyTargetClass` في ذلك الاختبار. ويفشل الاختبار عمدًا عند وجود معرّفات أهداف غير مصنفة حتى لا يمكن تخطي الفئات الجديدة بصمت.
