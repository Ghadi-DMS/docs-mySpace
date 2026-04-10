---
read_when:
    - تشغيل الاختبارات محليًا أو في CI
    - إضافة اختبارات انحدار لأخطاء النموذج/المزوّد
    - تصحيح سلوك البوابة + الوكيل
summary: 'مجموعة الاختبار: مجموعات unit/e2e/live، مشغّلات Docker، وما الذي يغطيه كل اختبار'
title: الاختبار
x-i18n:
    generated_at: "2026-04-10T07:17:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: 21b78e59a5189f4e8e6e1b490d350f4735c0395da31d21fc5d10b825313026b4
    source_path: help/testing.md
    workflow: 15
---

# الاختبار

يحتوي OpenClaw على ثلاث مجموعات Vitest (unit/integration وe2e وlive) ومجموعة صغيرة من مشغّلات Docker.

هذه الوثيقة هي دليل "كيف نختبر":

- ما الذي تغطيه كل مجموعة (وما الذي لا تغطيه عمدًا)
- الأوامر التي يجب تشغيلها لسير العمل الشائع (محليًا، قبل الدفع، التصحيح)
- كيف تكتشف الاختبارات الحية بيانات الاعتماد وتحدد النماذج/المزوّدين
- كيفية إضافة اختبارات انحدار لمشكلات النماذج/المزوّدين في العالم الحقيقي

## البداية السريعة

في معظم الأيام:

- البوابة الكاملة (متوقعة قبل الدفع): `pnpm build && pnpm check && pnpm test`
- تشغيل أسرع للمجموعة الكاملة محليًا على جهاز ذي موارد كافية: `pnpm test:max`
- حلقة مراقبة Vitest مباشرة: `pnpm test:watch`
- استهداف الملفات مباشرة يوجّه الآن أيضًا مسارات الإضافات/القنوات: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- فضّل أولًا التشغيلات المستهدفة عندما تكون تعمل على تكرار فشل واحد.
- موقع QA مدعوم بـ Docker: `pnpm qa:lab:up`
- مسار QA مدعوم بآلة Linux افتراضية: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

عندما تلمس الاختبارات أو تريد ثقة إضافية:

- بوابة التغطية: `pnpm test:coverage`
- مجموعة E2E: `pnpm test:e2e`

عند تصحيح مزوّدين/نماذج حقيقية (يتطلب بيانات اعتماد حقيقية):

- المجموعة الحية (فحوصات النماذج + أدوات/صور البوابة): `pnpm test:live`
- استهداف ملف حي واحد بهدوء: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

نصيحة: عندما تحتاج فقط إلى حالة فاشلة واحدة، ففضّل تضييق الاختبارات الحية عبر متغيرات البيئة الخاصة بقائمة السماح الموضحة أدناه.

## مشغّلات خاصة بـ QA

توجد هذه الأوامر بجانب مجموعات الاختبار الرئيسية عندما تحتاج إلى واقعية QA-lab:

- `pnpm openclaw qa suite`
  - يشغّل سيناريوهات QA المعتمدة على المستودع مباشرة على المضيف.
- `pnpm openclaw qa suite --runner multipass`
  - يشغّل مجموعة QA نفسها داخل آلة Linux افتراضية مؤقتة عبر Multipass.
  - يحافظ على سلوك اختيار السيناريو نفسه كما في `qa suite` على المضيف.
  - يعيد استخدام أعلام اختيار المزوّد/النموذج نفسها كما في `qa suite`.
  - التشغيلات الحية تمرّر مدخلات مصادقة QA المدعومة والعملية للضيف:
    مفاتيح المزوّد المعتمدة على البيئة، ومسار إعدادات مزوّد QA الحي، و`CODEX_HOME`
    عند وجوده.
  - يجب أن تبقى مجلدات المخرجات تحت جذر المستودع حتى يتمكن الضيف من الكتابة مرة أخرى عبر
    مساحة العمل المركبة.
  - يكتب تقرير QA العادي + الملخص بالإضافة إلى سجلات Multipass ضمن
    `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - يبدأ موقع QA المدعوم بـ Docker لأعمال QA بأسلوب المشغّل.

## مجموعات الاختبار (ما الذي يعمل وأين)

فكّر في المجموعات باعتبارها "واقعية متزايدة" (ومعها تزداد القابلية للتذبذب/التكلفة):

### Unit / integration (الافتراضي)

- الأمر: `pnpm test`
- الإعداد: عشر تشغيلات شظايا متسلسلة (`vitest.full-*.config.ts`) عبر مشاريع Vitest المحددة الموجودة
- الملفات: قوائم core/unit ضمن `src/**/*.test.ts` و`packages/**/*.test.ts` و`test/**/*.test.ts` واختبارات `ui` الخاصة بـ node المدرجة في قائمة السماح والمشمولة بواسطة `vitest.unit.config.ts`
- النطاق:
  - اختبارات unit خالصة
  - اختبارات integration داخل العملية (مصادقة البوابة، التوجيه، الأدوات، التحليل، الإعدادات)
  - اختبارات انحدار حتمية للأخطاء المعروفة
- التوقعات:
  - تعمل في CI
  - لا تتطلب مفاتيح حقيقية
  - يجب أن تكون سريعة ومستقرة
- ملاحظة المشاريع:
  - تشغيل `pnpm test` غير المستهدف يشغّل الآن أحد عشر إعداد شظايا أصغر (`core-unit-src` و`core-unit-security` و`core-unit-ui` و`core-unit-support` و`core-support-boundary` و`core-contracts` و`core-bundled` و`core-runtime` و`agentic` و`auto-reply` و`extensions`) بدلًا من عملية مشروع جذرية أصلية ضخمة واحدة. هذا يقلل ذروة RSS على الأجهزة المحمّلة ويمنع أعمال auto-reply/الإضافات من تجويع المجموعات غير المرتبطة.
  - `pnpm test --watch` لا يزال يستخدم رسم مشروع الجذر الأصلي `vitest.config.ts`، لأن حلقة مراقبة متعددة الشظايا غير عملية.
  - `pnpm test` و`pnpm test:watch` و`pnpm test:perf:imports` توجّه أهداف الملفات/المجلدات الصريحة عبر المسارات المحددة أولًا، لذا فإن `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` يتجنب تكلفة بدء تشغيل مشروع الجذر الكامل.
  - `pnpm test:changed` يوسّع مسارات git المتغيرة إلى المسارات المحددة نفسها عندما يلمس الفرق فقط ملفات المصدر/الاختبار القابلة للتوجيه؛ أما تعديلات الإعداد/التهيئة فتعود إلى إعادة التشغيل الأوسع لمشروع الجذر.
  - بعض اختبارات `plugin-sdk` و`commands` تُوجَّه أيضًا عبر مسارات خفيفة مخصصة تتخطى `test/setup-openclaw-runtime.ts`؛ أما الملفات ذات الحالة/الأثقل وقت التشغيل فتبقى على المسارات الموجودة.
  - بعض ملفات مصدر المساعدة في `plugin-sdk` و`commands` تُطابق أيضًا التشغيلات في وضع changed مع اختبارات شقيقة صريحة في تلك المسارات الخفيفة، بحيث تتجنب تعديلات المساعدين إعادة تشغيل المجموعة الثقيلة الكاملة لذلك الدليل.
  - لدى `auto-reply` الآن ثلاث حاويات مخصصة: مساعدات core من المستوى الأعلى، واختبارات integration العليا `reply.*`، والشجرة الفرعية `src/auto-reply/reply/**`. هذا يُبقي أثقل أعمال حزمة الرد خارج اختبارات الحالة/الأجزاء/الرموز الأرخص.
- ملاحظة المشغّل المضمّن:
  - عندما تغيّر مدخلات اكتشاف أدوات الرسائل أو سياق وقت تشغيل الضغط،
    حافظ على مستويي التغطية معًا.
  - أضف اختبارات انحدار مركزة للمساعدين عند حدود التوجيه/التطبيع الخالصة.
  - وحافظ أيضًا على صحة مجموعات integration الخاصة بالمشغّل المضمّن:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`, و
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - تتحقق هذه المجموعات من أن المعرّفات المحددة وسلوك الضغط ما زالا يتدفقان
    عبر المسارات الحقيقية `run.ts` / `compact.ts`؛ اختبارات المساعدين وحدها ليست
    بديلًا كافيًا عن مسارات integration هذه.
- ملاحظة pool:
  - إعداد Vitest الأساسي يستخدم الآن `threads` افتراضيًا.
  - إعداد Vitest المشترك يثبت أيضًا `isolate: false` ويستخدم المشغّل غير المعزول عبر مشاريع الجذر وتهيئات e2e وlive.
  - يحتفظ مسار UI الجذري بإعداد `jsdom` والمُحسّن الخاص به، لكنه يعمل الآن أيضًا على المشغّل غير المعزول المشترك.
  - كل شظية `pnpm test` ترث القيم الافتراضية نفسها `threads` + `isolate: false` من إعداد Vitest المشترك.
  - يضيف مشغّل `scripts/run-vitest.mjs` المشترك الآن أيضًا `--no-maglev` لعمليات Node الفرعية الخاصة بـ Vitest افتراضيًا لتقليل اضطراب ترجمة V8 أثناء التشغيلات المحلية الكبيرة. اضبط `OPENCLAW_VITEST_ENABLE_MAGLEV=1` إذا كنت تحتاج إلى المقارنة مع سلوك V8 القياسي.
- ملاحظة التكرار المحلي السريع:
  - `pnpm test:changed` يوجّه عبر المسارات المحددة عندما تُطابق المسارات المتغيرة مجموعة أصغر بشكل واضح.
  - `pnpm test:max` و`pnpm test:changed:max` يحافظان على سلوك التوجيه نفسه، ولكن مع حد أعلى للعمال.
  - التوسّع التلقائي لعدد العمال محليًا أصبح محافظًا عمدًا الآن ويتراجع أيضًا عندما يكون متوسط حمل المضيف مرتفعًا أصلًا، بحيث تُحدث تشغيلات Vitest المحلية المتعددة ضررًا أقل افتراضيًا.
  - يضع إعداد Vitest الأساسي المشاريع/ملفات الإعداد ضمن `forceRerunTriggers` حتى تبقى إعادة التشغيل في وضع changed صحيحة عندما تتغير آلية توصيل الاختبارات.
  - يحتفظ الإعداد بتمكين `OPENCLAW_VITEST_FS_MODULE_CACHE` على المضيفين المدعومين؛ اضبط `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` إذا كنت تريد موقع cache صريحًا واحدًا من أجل profiling مباشر.
- ملاحظة تصحيح الأداء:
  - `pnpm test:perf:imports` يفعّل تقارير مدة الاستيراد في Vitest بالإضافة إلى مخرجات تفصيل الاستيراد.
  - `pnpm test:perf:imports:changed` يقيّد عرض profiling نفسه للملفات المتغيرة منذ `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` يقارن `test:changed` الموجّه بمسار مشروع الجذر الأصلي لذلك الفرق الملتزم ويطبع زمن التنفيذ بالإضافة إلى أقصى RSS على macOS.
- `pnpm test:perf:changed:bench -- --worktree` يختبر أداء الشجرة المتسخة الحالية عبر توجيه قائمة الملفات المتغيرة خلال `scripts/test-projects.mjs` وتهيئة Vitest لمشروع الجذر.
  - `pnpm test:perf:profile:main` يكتب ملف CPU profile للخيط الرئيسي لوقت بدء Vitest/Vite ونفقات التحويل.
  - `pnpm test:perf:profile:runner` يكتب ملفات CPU+heap profile للمشغّل لمجموعة unit مع تعطيل توازي الملفات.

### E2E (فحوصات البوابة الأساسية)

- الأمر: `pnpm test:e2e`
- الإعداد: `vitest.e2e.config.ts`
- الملفات: `src/**/*.e2e.test.ts` و`test/**/*.e2e.test.ts`
- القيم الافتراضية لوقت التشغيل:
  - يستخدم Vitest `threads` مع `isolate: false`، بما يتطابق مع بقية المستودع.
  - يستخدم عمالًا تكيفيين (CI: حتى 2، محليًا: 1 افتراضيًا).
  - يعمل في الوضع الصامت افتراضيًا لتقليل تكلفة إدخال/إخراج وحدة التحكم.
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_WORKERS=<n>` لفرض عدد العمال (بحد أقصى 16).
  - `OPENCLAW_E2E_VERBOSE=1` لإعادة تفعيل مخرجات وحدة التحكم المفصلة.
- النطاق:
  - سلوك البوابة من طرف إلى طرف عبر عدة مثيلات
  - أسطح WebSocket/HTTP، واقتران العقد، والشبكات الأثقل
- التوقعات:
  - يعمل في CI (عند تفعيله في خط الأنابيب)
  - لا يتطلب مفاتيح حقيقية
  - يحتوي على أجزاء متحركة أكثر من اختبارات unit (وقد يكون أبطأ)

### E2E: فحوصات OpenShell الخلفية الأساسية

- الأمر: `pnpm test:e2e:openshell`
- الملف: `test/openshell-sandbox.e2e.test.ts`
- النطاق:
  - يبدأ بوابة OpenShell معزولة على المضيف عبر Docker
  - ينشئ صندوقًا رمليًا من Dockerfile محلي مؤقت
  - يختبر الخلفية OpenShell في OpenClaw عبر `sandbox ssh-config` الحقيقي + تنفيذ SSH
  - يتحقق من سلوك نظام الملفات canonical البعيد عبر جسر fs الخاص بالصندوق الرملي
- التوقعات:
  - اختيارية فقط؛ ليست جزءًا من تشغيل `pnpm test:e2e` الافتراضي
  - تتطلب CLI محليًا لـ `openshell` بالإضافة إلى daemon Docker عامل
  - تستخدم `HOME` / `XDG_CONFIG_HOME` معزولين، ثم تدمّر بوابة الاختبار والصندوق الرملي
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_OPENSHELL=1` لتفعيل الاختبار عند تشغيل مجموعة e2e الأوسع يدويًا
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` للإشارة إلى CLI غير افتراضي أو برنامج wrapper نصي

### Live (مزوّدون حقيقيون + نماذج حقيقية)

- الأمر: `pnpm test:live`
- الإعداد: `vitest.live.config.ts`
- الملفات: `src/**/*.live.test.ts`
- الافتراضي: **مفعّل** بواسطة `pnpm test:live` (يضبط `OPENCLAW_LIVE_TEST=1`)
- النطاق:
  - "هل يعمل هذا المزوّد/النموذج فعليًا _اليوم_ باستخدام بيانات اعتماد حقيقية؟"
  - التقاط تغيّرات تنسيق المزوّد، وخصائص استدعاء الأدوات، ومشكلات المصادقة، وسلوك حدود المعدل
- التوقعات:
  - غير مستقرة في CI بطبيعتها (شبكات حقيقية، سياسات مزوّدين حقيقية، حصص، انقطاعات)
  - تكلف المال / تستخدم حدود المعدل
  - يُفضّل تشغيل مجموعات فرعية مضيقة بدلًا من "كل شيء"
- تقوم التشغيلات الحية بتحميل `~/.profile` لالتقاط مفاتيح API الناقصة.
- افتراضيًا، تعزل التشغيلات الحية `HOME` أيضًا وتنسخ مواد الإعداد/المصادقة إلى home اختبار مؤقت حتى لا تتمكن تجهيزات unit من تعديل `~/.openclaw` الحقيقي لديك.
- اضبط `OPENCLAW_LIVE_USE_REAL_HOME=1` فقط عندما تحتاج عمدًا إلى أن تستخدم الاختبارات الحية دليل home الحقيقي لديك.
- أصبح `pnpm test:live` الآن افتراضيًا في وضع أكثر هدوءًا: فهو يحتفظ بمخرجات التقدم `[live] ...`، لكنه يخفي إشعار `~/.profile` الإضافي ويكتم سجلات إقلاع البوابة/ضجيج Bonjour. اضبط `OPENCLAW_LIVE_TEST_QUIET=0` إذا كنت تريد سجلات بدء التشغيل الكاملة مرة أخرى.
- تدوير مفاتيح API (خاص بالمزوّد): اضبط `*_API_KEYS` بصيغة فاصلة/فاصلة منقوطة أو `*_API_KEY_1` و`*_API_KEY_2` (على سبيل المثال `OPENAI_API_KEYS` و`ANTHROPIC_API_KEYS` و`GEMINI_API_KEYS`) أو تجاوزًا لكل live عبر `OPENCLAW_LIVE_*_KEY`؛ تعيد الاختبارات المحاولة عند استجابات حدود المعدل.
- مخرجات التقدم/نبض الحياة:
  - تصدر المجموعات الحية الآن أسطر التقدم إلى stderr بحيث تظهر استدعاءات المزوّد الطويلة على أنها نشطة حتى عندما يكون التقاط وحدة التحكم في Vitest هادئًا.
  - يعطّل `vitest.live.config.ts` اعتراض وحدة التحكم في Vitest بحيث تتدفق أسطر تقدم المزوّد/البوابة فورًا أثناء التشغيلات الحية.
  - اضبط نبضات النماذج المباشرة عبر `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - اضبط نبضات البوابة/الفحص عبر `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## ما المجموعة التي ينبغي أن أشغّلها؟

استخدم جدول القرار هذا:

- تعديل المنطق/الاختبارات: شغّل `pnpm test` (و`pnpm test:coverage` إذا غيّرت الكثير)
- لمس شبكات البوابة / بروتوكول WS / الاقتران: أضف `pnpm test:e2e`
- تصحيح "الروبوت الخاص بي متوقف" / الأعطال الخاصة بالمزوّد / استدعاء الأدوات: شغّل `pnpm test:live` مضيقًا

## Live: مسح قدرات عقدة Android

- الاختبار: `src/gateway/android-node.capabilities.live.test.ts`
- السكربت: `pnpm android:test:integration`
- الهدف: استدعاء **كل أمر مُعلن عنه حاليًا** من عقدة Android متصلة والتأكد من سلوك عقد الأمر.
- النطاق:
  - إعداد مسبق/يدوي (المجموعة لا تثبّت التطبيق ولا تشغّله ولا تقترنه).
  - التحقق من `node.invoke` في البوابة لكل أمر على حدة لعقدة Android المحددة.
- الإعداد المسبق المطلوب:
  - تطبيق Android متصل ومقترن بالفعل بالبوابة.
  - إبقاء التطبيق في الواجهة الأمامية.
  - منح الأذونات/موافقة الالتقاط للقدرات التي تتوقع نجاحها.
- تجاوزات الهدف الاختيارية:
  - `OPENCLAW_ANDROID_NODE_ID` أو `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- تفاصيل إعداد Android الكاملة: [تطبيق Android](/ar/platforms/android)

## Live: فحص أساسي للنماذج (مفاتيح profile)

تنقسم الاختبارات الحية إلى طبقتين حتى نتمكن من عزل الأعطال:

- يوضح "النموذج المباشر" ما إذا كان المزوّد/النموذج قادرًا على الرد أصلًا باستخدام المفتاح المعطى.
- يوضح "فحص البوابة الأساسي" ما إذا كان مسار البوابة+الوكيل الكامل يعمل لهذا النموذج (الجلسات، السجل، الأدوات، سياسة sandbox، إلخ).

### الطبقة 1: إكمال مباشر للنموذج (من دون بوابة)

- الاختبار: `src/agents/models.profiles.live.test.ts`
- الهدف:
  - تعداد النماذج المكتشفة
  - استخدام `getApiKeyForModel` لاختيار النماذج التي لديك بيانات اعتماد لها
  - تشغيل إكمال صغير لكل نموذج (واختبارات انحدار مستهدفة عند الحاجة)
- كيفية التفعيل:
  - `pnpm test:live` (أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- اضبط `OPENCLAW_LIVE_MODELS=modern` (أو `all`، وهو اسم بديل لـ modern) لتشغيل هذه المجموعة فعليًا؛ وإلا فسيتم تخطيها للحفاظ على تركيز `pnpm test:live` على فحص البوابة الأساسي
- كيفية اختيار النماذج:
  - `OPENCLAW_LIVE_MODELS=modern` لتشغيل قائمة السماح الحديثة (Opus/Sonnet 4.6+، وGPT-5.x + Codex، وGemini 3، وGLM 4.7، وMiniMax M2.7، وGrok 4)
  - `OPENCLAW_LIVE_MODELS=all` هو اسم بديل لقائمة السماح الحديثة
  - أو `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (قائمة سماح مفصولة بفواصل)
  - تستخدم عمليات الفحص modern/all حدًا انتقائيًا عالي الإشارة افتراضيًا؛ اضبط `OPENCLAW_LIVE_MAX_MODELS=0` لإجراء فحص حديث شامل أو رقمًا موجبًا لحد أصغر.
- كيفية اختيار المزوّدين:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (قائمة سماح مفصولة بفواصل)
- من أين تأتي المفاتيح:
  - افتراضيًا: مخزن profile وبدائل env
  - اضبط `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض **مخزن profile** فقط
- سبب وجود هذا:
  - يفصل بين "واجهة API الخاصة بالمزوّد معطلة / المفتاح غير صالح" و"مسار وكيل البوابة معطل"
  - يحتوي اختبارات انحدار صغيرة ومعزولة (مثال: إعادة تشغيل الاستدلال + تدفقات استدعاء الأدوات في OpenAI Responses/Codex Responses)

### الطبقة 2: فحص البوابة + وكيل dev الأساسي (ما الذي يفعله "@openclaw" فعليًا)

- الاختبار: `src/gateway/gateway-models.profiles.live.test.ts`
- الهدف:
  - تشغيل بوابة داخل العملية
  - إنشاء/ترقيع جلسة `agent:dev:*` (مع تجاوز النموذج لكل تشغيل)
  - التكرار عبر النماذج التي تملك مفاتيح والتأكد من:
    - استجابة "ذات معنى" (من دون أدوات)
    - نجاح استدعاء أداة حقيقي (`read` probe)
    - نجاح فحوصات أدوات إضافية اختيارية (`exec+read` probe)
    - استمرار عمل مسارات الانحدار في OpenAI (استدعاء أدوات فقط ← متابعة)
- تفاصيل الفحوصات (حتى تتمكن من شرح الأعطال بسرعة):
  - فحص `read`: يكتب الاختبار ملف nonce في مساحة العمل ويطلب من الوكيل أن يقوم بـ `read` له ثم يعيد nonce.
  - فحص `exec+read`: يطلب الاختبار من الوكيل أن يكتب nonce إلى ملف مؤقت عبر `exec`، ثم يقوم بـ `read` له مرة أخرى.
  - فحص الصورة: يرفق الاختبار ملف PNG مُولّدًا (قط + رمز عشوائي) ويتوقع من النموذج أن يعيد `cat <CODE>`.
  - مرجع التنفيذ: `src/gateway/gateway-models.profiles.live.test.ts` و`src/gateway/live-image-probe.ts`.
- كيفية التفعيل:
  - `pnpm test:live` (أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- كيفية اختيار النماذج:
  - الافتراضي: قائمة السماح الحديثة (Opus/Sonnet 4.6+، وGPT-5.x + Codex، وGemini 3، وGLM 4.7، وMiniMax M2.7، وGrok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` هو اسم بديل لقائمة السماح الحديثة
  - أو اضبط `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (أو قائمة مفصولة بفواصل) للتضييق
  - تستخدم عمليات فحص البوابة modern/all حدًا انتقائيًا عالي الإشارة افتراضيًا؛ اضبط `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` لإجراء فحص حديث شامل أو رقمًا موجبًا لحد أصغر.
- كيفية اختيار المزوّدين (لتجنب "كل شيء في OpenRouter"):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (قائمة سماح مفصولة بفواصل)
- فحوصات الأدوات + الصور مفعّلة دائمًا في هذا الاختبار الحي:
  - فحص `read` + فحص `exec+read` (ضغط على الأدوات)
  - يعمل فحص الصورة عندما يعلن النموذج دعم إدخال الصور
  - التدفق (على مستوى عالٍ):
    - يولّد الاختبار ملف PNG صغيرًا يحتوي على “CAT” + رمز عشوائي (`src/gateway/live-image-probe.ts`)
    - يرسله عبر `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - تقوم البوابة بتحليل attachments إلى `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - يمرّر الوكيل المضمّن رسالة مستخدم متعددة الوسائط إلى النموذج
    - التأكيد: يحتوي الرد على `cat` + الرمز (مع سماحية OCR: الأخطاء البسيطة مقبولة)

نصيحة: لمعرفة ما يمكنك اختباره على جهازك (ومعرّفات `provider/model` الدقيقة)، شغّل:

```bash
openclaw models list
openclaw models list --json
```

## Live: فحص أساسي لخلفية CLI (Claude أو Codex أو Gemini أو CLIs محلية أخرى)

- الاختبار: `src/gateway/gateway-cli-backend.live.test.ts`
- الهدف: التحقق من مسار البوابة + الوكيل باستخدام خلفية CLI محلية، من دون لمس إعداداتك الافتراضية.
- القيم الافتراضية لفحص الخلفية الخاص بـ CLI موجودة ضمن تعريف `cli-backend.ts` التابع للإضافة المالكة.
- التفعيل:
  - `pnpm test:live` (أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- القيم الافتراضية:
  - المزوّد/النموذج الافتراضي: `claude-cli/claude-sonnet-4-6`
  - سلوك الأمر/المعاملات/الصور يأتي من بيانات التعريف التابعة للإضافة المالكة لخلفية CLI.
- التجاوزات (اختيارية):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` لإرسال مرفق صورة حقيقي (يتم حقن المسارات في prompt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` لتمرير مسارات ملفات الصور كمعاملات CLI بدلًا من الحقن في prompt.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (أو `"list"`) للتحكم في كيفية تمرير معاملات الصور عند تعيين `IMAGE_ARG`.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` لإرسال دور ثانٍ والتحقق من مسار الاستئناف.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` لتعطيل فحص الاستمرارية الافتراضي للجلسة نفسها عند الانتقال من Claude Sonnet إلى Opus (اضبطه على `1` لفرض التفعيل عندما يدعم النموذج المحدد هدف تبديل).

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

وصفات Docker لمزوّد واحد:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

ملاحظات:

- يوجد مشغّل Docker في `scripts/test-live-cli-backend-docker.sh`.
- يشغّل فحص خلفية CLI الحي داخل صورة Docker الخاصة بالمستودع بصفته المستخدم غير الجذر `node`.
- يحل بيانات تعريف فحص CLI الأساسية من الإضافة المالكة، ثم يثبّت حزمة CLI المطابقة على Linux (`@anthropic-ai/claude-code` أو `@openai/codex` أو `@google/gemini-cli`) داخل بادئة قابلة للكتابة ومخزنة مؤقتًا في `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (الافتراضي: `~/.cache/openclaw/docker-cli-tools`).
- أصبح فحص خلفية CLI الحي الآن يختبر المسار الكامل نفسه من طرف إلى طرف لكل من Claude وCodex وGemini: دور نصي، ثم دور تصنيف صورة، ثم استدعاء أداة MCP `cron` مع التحقق عبر CLI الخاص بالبوابة.
- يقوم الفحص الافتراضي لـ Claude أيضًا بترقيع الجلسة من Sonnet إلى Opus ويتحقق من أن الجلسة المستأنفة ما زالت تتذكر ملاحظة سابقة.

## Live: فحص ACP bind الأساسي (`/acp spawn ... --bind here`)

- الاختبار: `src/gateway/gateway-acp-bind.live.test.ts`
- الهدف: التحقق من مسار ربط المحادثة ACP الحقيقي باستخدام وكيل ACP حي:
  - إرسال `/acp spawn <agent> --bind here`
  - ربط محادثة اصطناعية لقناة الرسائل في مكانها
  - إرسال متابعة عادية على المحادثة نفسها
  - التحقق من أن المتابعة تصل إلى transcript جلسة ACP المرتبطة
- التفعيل:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- القيم الافتراضية:
  - وكلاء ACP في Docker: `claude,codex,gemini`
  - وكيل ACP عند التشغيل المباشر عبر `pnpm test:live ...`: `claude`
  - القناة الاصطناعية: سياق محادثة بأسلوب الرسائل المباشرة في Slack
  - خلفية ACP: `acpx`
- التجاوزات:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- ملاحظات:
  - يستخدم هذا المسار سطح البوابة `chat.send` مع حقول originating-route اصطناعية خاصة بالمشرف فقط، حتى تتمكن الاختبارات من إرفاق سياق قناة الرسائل من دون التظاهر بالتسليم الخارجي.
  - عندما لا يكون `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` مضبوطًا، يستخدم الاختبار سجل الوكلاء المضمّن في إضافة `acpx` المحددة لوكيل حزمة ACP.

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

- يوجد مشغّل Docker في `scripts/test-live-acp-bind-docker.sh`.
- يشغّل فحص ACP bind الأساسي افتراضيًا على جميع وكلاء CLI الحية المدعومة بالتسلسل: `claude` ثم `codex` ثم `gemini`.
- استخدم `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude` أو `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` أو `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` لتضييق المصفوفة.
- يقوم بتحميل `~/.profile`، ويجهّز مواد المصادقة المطابقة لـ CLI داخل الحاوية، ويثبت `acpx` في بادئة npm قابلة للكتابة، ثم يثبت CLI الحي المطلوب (`@anthropic-ai/claude-code` أو `@openai/codex` أو `@google/gemini-cli`) إذا كان مفقودًا.
- داخل Docker، يضبط المشغّل `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` حتى يحتفظ acpx بمتغيرات بيئة المزوّد القادمة من profile المحمّل والمتاحة لـ CLI الفرعي الخاص بالحزمة.

### وصفات live الموصى بها

قوائم السماح الضيقة والصريحة هي الأسرع والأقل تذبذبًا:

- نموذج واحد، مباشر (من دون بوابة):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- نموذج واحد، فحص بوابة أساسي:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- استدعاء الأدوات عبر عدة مزوّدين:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- التركيز على Google (مفتاح Gemini API + Antigravity):
  - Gemini (مفتاح API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

ملاحظات:

- `google/...` يستخدم Gemini API (مفتاح API).
- `google-antigravity/...` يستخدم جسر Antigravity OAuth (نقطة نهاية وكيل بأسلوب Cloud Code Assist).
- `google-gemini-cli/...` يستخدم Gemini CLI المحلي على جهازك (مصادقة منفصلة وخصائص أدوات مختلفة).
- Gemini API مقابل Gemini CLI:
  - API: يقوم OpenClaw باستدعاء Gemini API المستضاف من Google عبر HTTP (مفتاح API / مصادقة profile)؛ وهذا ما يقصده معظم المستخدمين عند قولهم "Gemini".
  - CLI: يقوم OpenClaw باستدعاء ملف `gemini` الثنائي المحلي عبر shell؛ وله مصادقة خاصة به وقد يتصرف بشكل مختلف (الدفق/دعم الأدوات/اختلاف الإصدارات).

## Live: مصفوفة النماذج (ما الذي نغطيه)

لا توجد "قائمة نماذج CI" ثابتة (لأن live اختيارية)، لكن هذه هي النماذج **الموصى بها** للتغطية بانتظام على جهاز تطوير يملك مفاتيح.

### مجموعة الفحص الأساسية الحديثة (استدعاء الأدوات + الصور)

هذا هو تشغيل "النماذج الشائعة" الذي نتوقع الحفاظ على عمله:

- OpenAI (غير Codex): `openai/gpt-5.4` (اختياري: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (أو `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` و`google/gemini-3-flash-preview` (تجنب نماذج Gemini 2.x الأقدم)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` و`google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

شغّل فحص البوابة الأساسي مع الأدوات + الصور:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### الأساس: استدعاء الأدوات (Read + Exec اختياري)

اختر على الأقل نموذجًا واحدًا لكل عائلة مزوّد:

- OpenAI: `openai/gpt-5.4` (أو `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (أو `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (أو `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

تغطية إضافية اختيارية (جيدة إن توفرت):

- xAI: `xai/grok-4` (أو أحدث إصدار متاح)
- Mistral: `mistral/`… (اختر نموذجًا واحدًا يدعم "tools" ومفعّلًا لديك)
- Cerebras: `cerebras/`… (إذا كان لديك وصول)
- LM Studio: `lmstudio/`… (محلي؛ استدعاء الأدوات يعتمد على وضع API)

### Vision: إرسال الصور (مرفق ← رسالة متعددة الوسائط)

ضمّن نموذجًا واحدًا على الأقل يدعم الصور في `OPENCLAW_LIVE_GATEWAY_MODELS` (مثل Claude أو Gemini أو متغيرات OpenAI الداعمة للرؤية، إلخ) لاختبار فحص الصور.

### المجمعات / البوابات البديلة

إذا كانت لديك مفاتيح مفعلة، فنحن ندعم أيضًا الاختبار عبر:

- OpenRouter: `openrouter/...` (مئات النماذج؛ استخدم `openclaw models scan` للعثور على مرشحين يدعمون الأدوات+الصور)
- OpenCode: `opencode/...` لـ Zen و`opencode-go/...` لـ Go (المصادقة عبر `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

مزيد من المزوّدين الذين يمكنك تضمينهم في مصفوفة live (إذا كانت لديك بيانات اعتماد/إعدادات):

- مدمجة: `openai` و`openai-codex` و`anthropic` و`google` و`google-vertex` و`google-antigravity` و`google-gemini-cli` و`zai` و`openrouter` و`opencode` و`opencode-go` و`xai` و`groq` و`cerebras` و`mistral` و`github-copilot`
- عبر `models.providers` (نقاط نهاية مخصصة): `minimax` (سحابي/API)، بالإضافة إلى أي proxy متوافق مع OpenAI/Anthropic (مثل LM Studio أو vLLM أو LiteLLM، إلخ)

نصيحة: لا تحاول تضمين "كل النماذج" بشكل ثابت في الوثائق. القائمة المرجعية هي كل ما تعيده `discoverModels(...)` على جهازك + أي مفاتيح متاحة.

## بيانات الاعتماد (لا تُجرِ commit لها أبدًا)

تكتشف الاختبارات الحية بيانات الاعتماد بالطريقة نفسها التي يفعلها CLI. والنتائج العملية لذلك:

- إذا كان CLI يعمل، فينبغي أن تجد الاختبارات الحية المفاتيح نفسها.
- إذا أبلغك اختبار حي بأنه "لا توجد بيانات اعتماد"، فقم بالتصحيح بالطريقة نفسها التي ستصحح بها `openclaw models list` / اختيار النموذج.

- ملفات مصادقة profile لكل وكيل: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (وهذا هو المقصود بـ "profile keys" في الاختبارات الحية)
- الإعدادات: `~/.openclaw/openclaw.json` (أو `OPENCLAW_CONFIG_PATH`)
- دليل الحالة القديم: `~/.openclaw/credentials/` (يُنسخ إلى home الحي المرحلي عند وجوده، لكنه ليس مخزن مفاتيح profile الرئيسي)
- تقوم التشغيلات الحية المحلية افتراضيًا بنسخ الإعدادات النشطة وملفات `auth-profiles.json` لكل وكيل و`credentials/` القديمة وأدلة مصادقة CLI الخارجية المدعومة إلى home اختبار مؤقت؛ وتتخطى homes الحية المرحلية `workspace/` و`sandboxes/`، كما تُزال تجاوزات المسار `agents.*.workspace` و`agentDir` حتى تبقى الفحوصات بعيدة عن مساحة العمل الحقيقية على المضيف.

إذا كنت تريد الاعتماد على مفاتيح env (مثل المصدّرة في `~/.profile`)، فشغّل الاختبارات المحلية بعد `source ~/.profile`، أو استخدم مشغّلات Docker أدناه (يمكنها تركيب `~/.profile` داخل الحاوية).

## Deepgram live (نسخ الصوت)

- الاختبار: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- التفعيل: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- الاختبار: `src/agents/byteplus.live.test.ts`
- التفعيل: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- تجاوز اختياري للنموذج: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- الاختبار: `extensions/comfy/comfy.live.test.ts`
- التفعيل: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- النطاق:
  - يختبر مسارات الصور والفيديو و`music_generate` المجمّعة في comfy
  - يتخطى كل قدرة ما لم يتم إعداد `models.providers.comfy.<capability>`
  - مفيد بعد تغيير إرسال سير عمل comfy أو polling أو التنزيلات أو تسجيل plugin

## Image generation live

- الاختبار: `src/image-generation/runtime.live.test.ts`
- الأمر: `pnpm test:live src/image-generation/runtime.live.test.ts`
- الحزمة: `pnpm test:live:media image`
- النطاق:
  - يعدد كل plugin مزوّد لتوليد الصور مسجّل
  - يحمّل متغيرات بيئة المزوّد الناقصة من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/الخاصة بـ env قبل ملفات auth profiles المخزنة افتراضيًا، حتى لا تحجب مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات اعتماد shell الحقيقية
  - يتخطى المزوّدين الذين لا يملكون مصادقة/‏profile/‏نموذجًا صالحًا
  - يشغّل متغيرات توليد الصور القياسية عبر قدرة runtime المشتركة:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- المزوّدون المجمّعون المشمولون حاليًا:
  - `openai`
  - `google`
- التضييق الاختياري:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- سلوك المصادقة الاختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن profile وتجاهل تجاوزات env فقط

## Music generation live

- الاختبار: `extensions/music-generation-providers.live.test.ts`
- التفعيل: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- الحزمة: `pnpm test:live:media music`
- النطاق:
  - يختبر المسار المشترك المجمّع لمزوّد توليد الموسيقى
  - يغطي حاليًا Google وMiniMax
  - يحمّل متغيرات بيئة المزوّد من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/الخاصة بـ env قبل ملفات auth profiles المخزنة افتراضيًا، حتى لا تحجب مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات اعتماد shell الحقيقية
  - يتخطى المزوّدين الذين لا يملكون مصادقة/‏profile/‏نموذجًا صالحًا
  - يشغّل وضعي runtime المعلنين كليهما عند توفرهما:
    - `generate` مع إدخال يعتمد على prompt فقط
    - `edit` عندما يعلن المزوّد `capabilities.edit.enabled`
  - التغطية الحالية للمسار المشترك:
    - `google`: `generate` و`edit`
    - `minimax`: `generate`
    - `comfy`: ملف Comfy حي منفصل، وليس هذا المسح المشترك
- التضييق الاختياري:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- سلوك المصادقة الاختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن profile وتجاهل تجاوزات env فقط

## Video generation live

- الاختبار: `extensions/video-generation-providers.live.test.ts`
- التفعيل: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- الحزمة: `pnpm test:live:media video`
- النطاق:
  - يختبر المسار المشترك المجمّع لمزوّد توليد الفيديو
  - يحمّل متغيرات بيئة المزوّد من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/الخاصة بـ env قبل ملفات auth profiles المخزنة افتراضيًا، حتى لا تحجب مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات اعتماد shell الحقيقية
  - يتخطى المزوّدين الذين لا يملكون مصادقة/‏profile/‏نموذجًا صالحًا
  - يشغّل وضعي runtime المعلنين كليهما عند توفرهما:
    - `generate` مع إدخال يعتمد على prompt فقط
    - `imageToVideo` عندما يعلن المزوّد `capabilities.imageToVideo.enabled` ويقبل المزوّد/النموذج المحدد إدخال الصورة المحلية المعتمد على buffer في المسح المشترك
    - `videoToVideo` عندما يعلن المزوّد `capabilities.videoToVideo.enabled` ويقبل المزوّد/النموذج المحدد إدخال الفيديو المحلي المعتمد على buffer في المسح المشترك
  - المزوّدون الحاليون المعلن عنهم لكن المتخطَّون في المسح المشترك لـ `imageToVideo`:
    - `vydra` لأن `veo3` المجمّع نصي فقط و`kling` المجمّع يتطلب URL صورة بعيدًا
  - تغطية Vydra الخاصة بالمزوّد:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - يشغّل ذلك الملف `veo3` لتحويل النص إلى فيديو بالإضافة إلى مسار `kling` يستخدم fixture بعنوان URL صورة بعيد افتراضيًا
  - تغطية `videoToVideo` الحية الحالية:
    - `runway` فقط عندما يكون النموذج المحدد هو `runway/gen4_aleph`
  - المزوّدون الحاليون المعلن عنهم لكن المتخطَّون في المسح المشترك لـ `videoToVideo`:
    - `alibaba` و`qwen` و`xai` لأن هذه المسارات تتطلب حاليًا عناوين URL مرجعية بعيدة من نوع `http(s)` / MP4
    - `google` لأن مسار Gemini/Veo المشترك الحالي يستخدم إدخالًا محليًا مدعومًا بـ buffer وهذا المسار غير مقبول في المسح المشترك
    - `openai` لأن المسار المشترك الحالي لا يضمن وصولًا خاصًا بالمؤسسة إلى inpaint/remix للفيديو
- التضييق الاختياري:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- سلوك المصادقة الاختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن profile وتجاهل تجاوزات env فقط

## حزمة media الحية

- الأمر: `pnpm test:live:media`
- الغرض:
  - يشغّل مجموعات الصور والموسيقى والفيديو الحية المشتركة عبر نقطة دخول أصلية واحدة للمستودع
  - يحمّل متغيرات بيئة المزوّد الناقصة تلقائيًا من `~/.profile`
  - يضيّق كل مجموعة تلقائيًا إلى المزوّدين الذين يملكون حاليًا مصادقة صالحة افتراضيًا
  - يعيد استخدام `scripts/test-live.mjs`، حتى يبقى سلوك heartbeat والوضع الهادئ متسقًا
- أمثلة:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## مشغّلات Docker (اختبارات اختيارية من نوع "يعمل على Linux")

تنقسم مشغّلات Docker هذه إلى مجموعتين:

- مشغّلات النماذج الحية: `test:docker:live-models` و`test:docker:live-gateway` يشغّلان فقط ملف live المطابق المعتمد على مفاتيح profile داخل صورة Docker الخاصة بالمستودع (`src/agents/models.profiles.live.test.ts` و`src/gateway/gateway-models.profiles.live.test.ts`)، مع تركيب دليل الإعدادات المحلي ومساحة العمل لديك (ومع تحميل `~/.profile` إذا تم تركيبه). نقاط الدخول المحلية المطابقة هي `test:live:models-profiles` و`test:live:gateway-profiles`.
- تستخدم مشغّلات Docker الحية افتراضيًا حدًا أصغر للفحص الأساسي حتى يبقى المسح الكامل عبر Docker عمليًا:
  يضبط `test:docker:live-models` افتراضيًا `OPENCLAW_LIVE_MAX_MODELS=12`،
  ويضبط `test:docker:live-gateway` افتراضيًا `OPENCLAW_LIVE_GATEWAY_SMOKE=1`،
  و`OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`،
  و`OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`، و
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. تجاوز متغيرات البيئة هذه عندما
  تريد عمدًا المسح الأكبر والأشمل.
- يقوم `test:docker:all` ببناء صورة Docker الحية مرة واحدة عبر `test:docker:live-build`، ثم يعيد استخدامها لمساري Docker الحيين.
- مشغّلات الفحص الأساسية للحاويات: `test:docker:openwebui` و`test:docker:onboard` و`test:docker:gateway-network` و`test:docker:mcp-channels` و`test:docker:plugins` تشغّل حاوية حقيقية واحدة أو أكثر وتتحقق من مسارات integration الأعلى مستوى.

تقوم مشغّلات Docker الخاصة بالنماذج الحية أيضًا بتركيب homes مصادقة CLI المطلوبة فقط (أو جميع homes المدعومة عندما لا يكون التشغيل مضيقًا)، ثم تنسخها إلى home الحاوية قبل التشغيل حتى تتمكن مصادقة OAuth الخاصة بـ CLI الخارجي من تحديث الرموز من دون تعديل مخزن المصادقة على المضيف:

- النماذج المباشرة: `pnpm test:docker:live-models` (السكربت: `scripts/test-live-models-docker.sh`)
- فحص ACP bind الأساسي: `pnpm test:docker:live-acp-bind` (السكربت: `scripts/test-live-acp-bind-docker.sh`)
- فحص خلفية CLI الأساسي: `pnpm test:docker:live-cli-backend` (السكربت: `scripts/test-live-cli-backend-docker.sh`)
- البوابة + وكيل dev: `pnpm test:docker:live-gateway` (السكربت: `scripts/test-live-gateway-models-docker.sh`)
- فحص Open WebUI الحي الأساسي: `pnpm test:docker:openwebui` (السكربت: `scripts/e2e/openwebui-docker.sh`)
- معالج onboarding (TTY، مع كامل scaffolding): `pnpm test:docker:onboard` (السكربت: `scripts/e2e/onboard-docker.sh`)
- شبكات البوابة (حاويتان، مصادقة WS + health): `pnpm test:docker:gateway-network` (السكربت: `scripts/e2e/gateway-network-docker.sh`)
- جسر قناة MCP (بوابة مزروعة مسبقًا + جسر stdio + فحص إطار إشعارات Claude الخام): `pnpm test:docker:mcp-channels` (السكربت: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (فحص التثبيت الأساسي + الاسم البديل `/plugin` + دلالات إعادة تشغيل Claude-bundle): `pnpm test:docker:plugins` (السكربت: `scripts/e2e/plugins-docker.sh`)

تقوم مشغّلات Docker الخاصة بالنماذج الحية أيضًا بتركيب النسخة الحالية من
المستودع للقراءة فقط ثم تجهيزها في دليل عمل مؤقت داخل الحاوية. وهذا يُبقي صورة runtime
خفيفة مع الاستمرار في تشغيل Vitest على المصدر/الإعدادات المحلية الدقيقة لديك.
تتخطى خطوة التجهيز الـ cache المحلية الكبيرة فقط ومخرجات بناء التطبيق مثل
`.pnpm-store` و`.worktrees` و`__openclaw_vitest__` وأدلة
`.build` المحلية للتطبيق أو مخرجات Gradle، حتى لا تقضي تشغيلات Docker الحية دقائق في نسخ
الملفات الخاصة بالجهاز.
كما تضبط أيضًا `OPENCLAW_SKIP_CHANNELS=1` حتى لا تبدأ فحوصات البوابة الحية
عمّال القنوات الحقيقية مثل Telegram وDiscord وغيرها داخل الحاوية.
لا يزال `test:docker:live-models` يشغّل `pnpm test:live`، لذا مرّر
`OPENCLAW_LIVE_GATEWAY_*` أيضًا عندما تحتاج إلى تضييق أو استبعاد تغطية البوابة
الحية من مسار Docker هذا.
يُعد `test:docker:openwebui` فحص توافق أعلى مستوى: فهو يبدأ
حاوية بوابة OpenClaw مع تفعيل نقاط نهاية HTTP المتوافقة مع OpenAI،
ويبدأ حاوية Open WebUI مثبّتة الإصدار مقابل تلك البوابة، ويسجّل الدخول عبر
Open WebUI، ويتحقق من أن `/api/models` يعرض `openclaw/default`، ثم يرسل
طلب دردشة حقيقيًا عبر وكيل `/api/chat/completions` في Open WebUI.
قد يكون التشغيل الأول أبطأ بشكل ملحوظ لأن Docker قد يحتاج إلى سحب
صورة Open WebUI وقد تحتاج Open WebUI إلى إكمال إعداد البدء البارد الخاص بها.
يتوقع هذا المسار وجود مفتاح نموذج حي صالح، ويُعد `OPENCLAW_PROFILE_FILE`
(الافتراضي `~/.profile`) الوسيلة الأساسية لتوفيره في التشغيلات داخل Docker.
تطبع التشغيلات الناجحة حمولة JSON صغيرة مثل `{ "ok": true, "model":
"openclaw/default", ... }`.
تم تصميم `test:docker:mcp-channels` ليكون حتميًا عمدًا ولا يحتاج إلى
حساب Telegram أو Discord أو iMessage حقيقي. فهو يشغّل
حاوية بوابة مزروعة مسبقًا، ثم يبدأ حاوية ثانية تشغّل `openclaw mcp serve`، ثم
يتحقق من اكتشاف المحادثات الموجّهة، وقراءة transcript، وبيانات تعريف المرفقات،
وسلوك قائمة انتظار الأحداث الحية، وتوجيه الإرسال الصادر، وإشعارات القناة +
الأذونات بأسلوب Claude عبر جسر MCP الحقيقي المبني على stdio. ويتحقق فحص الإشعارات
من إطارات stdio الخام لـ MCP مباشرة، بحيث يثبت الفحص ما الذي يصدره الجسر فعليًا،
وليس فقط ما قد تعرضه مجموعة SDK معينة للعميل.

فحص يدوي لخيط ACP بلغة طبيعية بسيطة (ليس ضمن CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- احتفظ بهذا السكربت لسير عمل الانحدار/التصحيح. قد تكون هناك حاجة إليه مرة أخرى للتحقق من توجيه خيوط ACP، لذا لا تحذفه.

متغيرات بيئة مفيدة:

- `OPENCLAW_CONFIG_DIR=...` (الافتراضي: `~/.openclaw`) يُركَّب إلى `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (الافتراضي: `~/.openclaw/workspace`) يُركَّب إلى `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (الافتراضي: `~/.profile`) يُركَّب إلى `/home/node/.profile` ويُحمَّل قبل تشغيل الاختبارات
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (الافتراضي: `~/.cache/openclaw/docker-cli-tools`) يُركَّب إلى `/home/node/.npm-global` من أجل تثبيتات CLI المخزنة مؤقتًا داخل Docker
- تُركَّب أدلة/ملفات مصادقة CLI الخارجية تحت `$HOME` للقراءة فقط ضمن `/host-auth...`، ثم تُنسخ إلى `/home/node/...` قبل بدء الاختبارات
  - الأدلة الافتراضية: `.minimax`
  - الملفات الافتراضية: `~/.codex/auth.json` و`~/.codex/config.toml` و`.claude.json` و`~/.claude/.credentials.json` و`~/.claude/settings.json` و`~/.claude/settings.local.json`
  - التشغيلات المضيقة للمزوّدين تركّب فقط الأدلة/الملفات المطلوبة المستنتجة من `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - التجاوز اليدوي عبر `OPENCLAW_DOCKER_AUTH_DIRS=all` أو `OPENCLAW_DOCKER_AUTH_DIRS=none` أو قائمة مفصولة بفواصل مثل `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` لتضييق التشغيل
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` لتصفية المزوّدين داخل الحاوية
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لضمان أن تأتي بيانات الاعتماد من مخزن profile (وليس من env)
- `OPENCLAW_OPENWEBUI_MODEL=...` لاختيار النموذج الذي تعرضه البوابة لفحص Open WebUI الأساسي
- `OPENCLAW_OPENWEBUI_PROMPT=...` لتجاوز prompt التحقق من nonce المستخدم في فحص Open WebUI الأساسي
- `OPENWEBUI_IMAGE=...` لتجاوز وسم صورة Open WebUI المثبّت

## التحقق السريع من الوثائق

شغّل فحوصات الوثائق بعد تعديلها: `pnpm check:docs`.
وشغّل التحقق الكامل من روابط وعناوين Mintlify عندما تحتاج إلى التحقق من عناوين الصفحة الداخلية أيضًا: `pnpm docs:check-links:anchors`.

## اختبارات الانحدار دون اتصال (آمنة لـ CI)

هذه اختبارات انحدار لـ "مسار حقيقي" من دون مزوّدين حقيقيين:

- استدعاء أدوات البوابة (OpenAI وهمي، مع بوابة حقيقية + حلقة وكيل): `src/gateway/gateway.test.ts` (الحالة: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- معالج البوابة (WS `wizard.start`/`wizard.next`، يكتب config + auth enforced): `src/gateway/gateway.test.ts` (الحالة: "runs wizard over ws and writes auth token config")

## تقييمات موثوقية الوكيل (Skills)

لدينا بالفعل بعض الاختبارات الآمنة لـ CI التي تتصرف مثل "تقييمات موثوقية الوكيل":

- استدعاء أدوات وهمية عبر البوابة الحقيقية + حلقة الوكيل (`src/gateway/gateway.test.ts`).
- تدفقات معالج كاملة من طرف إلى طرف تتحقق من ربط الجلسة وتأثيرات الإعدادات (`src/gateway/gateway.test.ts`).

ما يزال مفقودًا بالنسبة إلى Skills (راجع [Skills](/ar/tools/skills)):

- **اتخاذ القرار:** عندما تُدرج Skills في prompt، هل يختار الوكيل المهارة الصحيحة (أو يتجنب غير ذات الصلة)؟
- **الامتثال:** هل يقرأ الوكيل `SKILL.md` قبل الاستخدام ويتبع الخطوات/المعاملات المطلوبة؟
- **عقود سير العمل:** سيناريوهات متعددة الأدوار تؤكد ترتيب الأدوات، واستمرار سجل الجلسة، وحدود sandbox.

ينبغي أن تبقى التقييمات المستقبلية حتمية أولًا:

- مشغّل سيناريوهات يستخدم مزوّدين وهميين لتأكيد استدعاءات الأدوات + ترتيبها، وقراءات ملفات المهارة، وربط الجلسات.
- مجموعة صغيرة من السيناريوهات التي تركّز على المهارات (الاستخدام مقابل التجنب، البوابات، حقن prompt).
- تقييمات حية اختيارية (opt-in ومحكومة بـ env) فقط بعد توفر المجموعة الآمنة لـ CI.

## اختبارات العقود (شكل plugin والقناة)

تتحقق اختبارات العقود من أن كل plugin وقناة مسجلين يلتزمان بعقد
الواجهة الخاص بهما. فهي تتكرر عبر كل plugins المكتشفة وتشغّل مجموعة من
تأكيدات الشكل والسلوك. يتخطى مسار unit الافتراضي `pnpm test` عمدًا
هذه الملفات المشتركة الخاصة بالحدود وملفات الفحص الأساسي؛ شغّل أوامر العقود صراحة
عندما تلمس أسطح القنوات أو المزوّدين المشتركة.

### الأوامر

- كل العقود: `pnpm test:contracts`
- عقود القنوات فقط: `pnpm test:contracts:channels`
- عقود المزوّدين فقط: `pnpm test:contracts:plugins`

### عقود القنوات

توجد في `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - شكل plugin الأساسي (المعرّف، الاسم، القدرات)
- **setup** - عقد معالج الإعداد
- **session-binding** - سلوك ربط الجلسة
- **outbound-payload** - بنية حمولة الرسائل
- **inbound** - معالجة الرسائل الواردة
- **actions** - معالجات إجراءات القناة
- **threading** - معالجة معرّف الخيط
- **directory** - API الدليل/القائمة
- **group-policy** - فرض سياسة المجموعة

### عقود حالة المزوّد

توجد في `src/plugins/contracts/*.contract.test.ts`.

- **status** - فحوصات حالة القناة
- **registry** - شكل سجل plugin

### عقود المزوّدين

توجد في `src/plugins/contracts/*.contract.test.ts`:

- **auth** - عقد تدفق المصادقة
- **auth-choice** - اختيار/تحديد المصادقة
- **catalog** - API فهرس النماذج
- **discovery** - اكتشاف plugin
- **loader** - تحميل plugin
- **runtime** - runtime المزوّد
- **shape** - شكل/واجهة plugin
- **wizard** - معالج الإعداد

### متى يتم تشغيلها

- بعد تغيير صادرات plugin-sdk أو المسارات الفرعية
- بعد إضافة plugin قناة أو مزوّد أو تعديلها
- بعد إعادة هيكلة تسجيل plugin أو اكتشافها

تعمل اختبارات العقود في CI ولا تتطلب مفاتيح API حقيقية.

## إضافة اختبارات الانحدار (إرشادات)

عندما تصلح مشكلة مزوّد/نموذج اكتُشفت في live:

- أضف اختبار انحدار آمنًا لـ CI إن أمكن (مزوّد وهمي/بديل، أو التقط تحويل شكل الطلب الدقيق)
- إذا كانت المشكلة حية فقط بطبيعتها (حدود المعدل، سياسات المصادقة)، فأبقِ الاختبار الحي ضيقًا واختياريًا عبر متغيرات env
- فضّل استهداف أصغر طبقة تلتقط الخطأ:
  - خطأ في تحويل/إعادة تشغيل طلب المزوّد → اختبار النماذج المباشرة
  - خطأ في مسار جلسة/سجل/أدوات البوابة → فحص البوابة الحي الأساسي أو اختبار بوابة وهمي آمن لـ CI
- حاجز traversal الخاص بـ SecretRef:
  - يستخرج `src/secrets/exec-secret-ref-id-parity.test.ts` هدفًا نموذجيًا واحدًا لكل فئة SecretRef من بيانات تعريف السجل (`listSecretTargetRegistryEntries()`)، ثم يؤكد رفض معرّفات exec الخاصة بمقاطع traversal.
  - إذا أضفت عائلة أهداف SecretRef جديدة من `includeInPlan` في `src/secrets/target-registry-data.ts`، فحدّث `classifyTargetClass` في ذلك الاختبار. يفشل الاختبار عمدًا عند وجود معرّفات أهداف غير مصنفة حتى لا يتم تخطي الفئات الجديدة بصمت.
