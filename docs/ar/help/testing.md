---
read_when:
    - تشغيل الاختبارات محليًا أو في CI
    - إضافة اختبارات تراجع لأخطاء النموذج/الموفر
    - تصحيح سلوك Gateway + الوكيل
summary: 'عدة الاختبار: مجموعات unit/e2e/live، ومشغلات Docker، وما الذي يغطيه كل اختبار'
title: الاختبار
x-i18n:
    generated_at: "2026-04-06T03:09:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: cfa174e565df5fdf957234b7909beaf1304aa026e731cc2c433ca7d931681b56
    source_path: help/testing.md
    workflow: 15
---

# الاختبار

يحتوي OpenClaw على ثلاث مجموعات Vitest (unit/integration وe2e وlive) ومجموعة
صغيرة من مشغلات Docker.

هذا المستند هو دليل "كيف نختبر":

- ما الذي تغطيه كل مجموعة (وما الذي لا تغطيه _عن قصد_)
- الأوامر التي يجب تشغيلها في سير العمل الشائع (محلي، قبل الدفع، تصحيح الأخطاء)
- كيف تكتشف الاختبارات الحية بيانات الاعتماد وتختار النماذج/الموفرين
- كيف تضيف اختبارات تراجع لمشكلات النماذج/الموفرين في العالم الحقيقي

## البدء السريع

في معظم الأيام:

- البوابة الكاملة (متوقعة قبل الدفع): `pnpm build && pnpm check && pnpm test`
- تشغيل أسرع للمجموعة الكاملة محليًا على جهاز واسع الموارد: `pnpm test:max`
- حلقة مراقبة Vitest مباشرة (إعداد modern projects): `pnpm test:watch`
- يستوعب الاستهداف المباشر للملفات الآن أيضًا مسارات extension/channel: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`

عندما تلمس الاختبارات أو تريد ثقة إضافية:

- بوابة التغطية: `pnpm test:coverage`
- مجموعة E2E: `pnpm test:e2e`

عند تصحيح موفرين/نماذج حقيقية (يتطلب بيانات اعتماد حقيقية):

- مجموعة live (مجسات النماذج + أدوات/صور Gateway): `pnpm test:live`
- استهداف ملف live واحد بهدوء: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

نصيحة: عندما تحتاج فقط إلى حالة فشل واحدة، فاختر تضييق اختبارات live عبر متغيرات البيئة allowlist الموضحة أدناه.

## مجموعات الاختبار (ما الذي يعمل وأين)

فكر في المجموعات على أنها "تزيد في الواقعية" (وتزيد في عدم الاستقرار/التكلفة):

### Unit / integration (الافتراضي)

- الأمر: `pnpm test`
- الإعداد: Vitest `projects` أصلي عبر `vitest.config.ts`
- الملفات: قوائم core/unit ضمن `src/**/*.test.ts` و`packages/**/*.test.ts` و`test/**/*.test.ts` واختبارات `ui` الخاصة بـ node والمدرجة في allowlist والمغطاة بواسطة `vitest.unit.config.ts`
- النطاق:
  - اختبارات unit خالصة
  - اختبارات integration داخل العملية (مصادقة gateway، والتوجيه، والأدوات، والتحليل، والإعدادات)
  - اختبارات تراجع حتمية لأخطاء معروفة
- التوقعات:
  - تعمل في CI
  - لا تتطلب مفاتيح حقيقية
  - يجب أن تكون سريعة ومستقرة
- ملاحظة المشاريع:
  - تستخدم الآن `pnpm test` و`pnpm test:watch` و`pnpm test:changed` جميعها إعداد `projects` الجذري الأصلي نفسه في Vitest.
  - تمر مرشحات الملفات المباشرة أصلًا عبر رسم المشروع الجذري، لذا يعمل `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` من دون غلاف مخصص.
- ملاحظة embedded runner:
  - عندما تغيّر مدخلات اكتشاف message-tool أو سياق وقت تشغيل compaction،
    فحافظ على مستويي التغطية كليهما.
  - أضف اختبارات تراجع مركزة للمساعدات عند حدود التوجيه/التطبيع الخالصة.
  - وحافظ أيضًا على سلامة مجموعات integration الخاصة بـ embedded runner:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`, و
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - تتحقق هذه المجموعات من أن المعرّفات المحددة وسلوك compaction لا يزالان
    يتدفقان عبر المسارات الحقيقية `run.ts` / `compact.ts`؛ ولا تُعد
    اختبارات المساعدات فقط بديلًا كافيًا عن مسارات integration هذه.
- ملاحظة الـ pool:
  - أصبح إعداد Vitest الأساسي يستخدم `threads` افتراضيًا.
  - كما يثبت إعداد Vitest المشترك `isolate: false` ويستخدم المشغل غير المعزول عبر المشاريع الجذرية وإعدادات e2e وlive.
  - يحتفظ مسار UI الجذري بإعداد `jsdom` والمُحسِّن الخاص به، لكنه يعمل الآن أيضًا على المشغل المشترك غير المعزول.
  - يرث `pnpm test` افتراضيات `threads` و`isolate: false` نفسها من إعداد `projects` في `vitest.config.ts` الجذري.
  - يضيف المشغل المشترك `scripts/run-vitest.mjs` الآن أيضًا `--no-maglev` افتراضيًا لعمليات Node التابعة لـ Vitest لتقليل تقلبات ترجمة V8 أثناء التشغيلات المحلية الكبيرة. اضبط `OPENCLAW_VITEST_ENABLE_MAGLEV=1` إذا كنت تحتاج المقارنة بسلوك V8 القياسي.
- ملاحظة التكرار المحلي السريع:
  - يشغّل `pnpm test:changed` إعداد projects الأصلي مع `--changed origin/main`.
  - يحتفظ `pnpm test:max` و`pnpm test:changed:max` بإعداد projects الأصلي نفسه، ولكن مع حد أعلى للعمال.
  - أصبح التوسع التلقائي لعدد العمال محليًا محافظًا عن عمد الآن، كما يتراجع عندما يكون متوسط تحميل المضيف مرتفعًا بالفعل، بحيث تُحدث تشغيلات Vitest المتزامنة المتعددة ضررًا أقل افتراضيًا.
  - يحدد إعداد Vitest الأساسي ملفات projects/config كعناصر `forceRerunTriggers` بحيث تبقى إعادة التشغيل في وضع changed صحيحة عندما يتغير wiring الخاص بالاختبارات.
  - يبقي الإعداد `OPENCLAW_VITEST_FS_MODULE_CACHE` مفعّلًا على المضيفين المدعومين؛ اضبط `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` إذا أردت موقع cache صريحًا واحدًا لأغراض profiling المباشر.
- ملاحظة تصحيح الأداء:
  - يفعّل `pnpm test:perf:imports` تقارير مدة الاستيراد في Vitest مع مخرجات تفصيلية للاستيراد.
  - يقيّد `pnpm test:perf:imports:changed` العرض التحليلي نفسه للملفات المتغيرة منذ `origin/main`.
  - يكتب `pnpm test:perf:profile:main` ملف تعريف CPU للخيط الرئيسي لتكاليف بدء Vitest/Vite والتحويل.
  - يكتب `pnpm test:perf:profile:runner` ملفات تعريف CPU+heap للمشغّل في مجموعة unit مع تعطيل التوازي على مستوى الملف.

### E2E (فحص Gateway شامل)

- الأمر: `pnpm test:e2e`
- الإعداد: `vitest.e2e.config.ts`
- الملفات: `src/**/*.e2e.test.ts` و`test/**/*.e2e.test.ts`
- افتراضيات وقت التشغيل:
  - يستخدم Vitest `threads` مع `isolate: false`، بما يطابق بقية المستودع.
  - يستخدم عمالًا تكيفيين (CI: حتى 2، محليًا: 1 افتراضيًا).
  - يعمل في الوضع الصامت افتراضيًا لتقليل حمل إدخال/إخراج وحدة التحكم.
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_WORKERS=<n>` لفرض عدد العمال (بحد أقصى 16).
  - `OPENCLAW_E2E_VERBOSE=1` لإعادة تمكين مخرجات وحدة التحكم التفصيلية.
- النطاق:
  - سلوك Gateway متعدد المثيلات من طرف إلى طرف
  - أسطح WebSocket/HTTP، واقتران العقد، وشبكات أثقل
- التوقعات:
  - تعمل في CI (عند تفعيلها في خط الأنابيب)
  - لا تتطلب مفاتيح حقيقية
  - تحتوي على أجزاء متحركة أكثر من اختبارات unit (وقد تكون أبطأ)

### E2E: فحص OpenShell backend

- الأمر: `pnpm test:e2e:openshell`
- الملف: `test/openshell-sandbox.e2e.test.ts`
- النطاق:
  - يبدأ Gateway OpenShell معزولًا على المضيف عبر Docker
  - ينشئ sandbox من Dockerfile محلي مؤقت
  - يختبر OpenClaw backend الخاص بـ OpenShell عبر `sandbox ssh-config` وSSH exec حقيقيين
  - يتحقق من سلوك نظام الملفات القياسي البعيد عبر جسر sandbox fs
- التوقعات:
  - اختياري فقط؛ ليس جزءًا من التشغيل الافتراضي `pnpm test:e2e`
  - يتطلب CLI محليًا لـ `openshell` وDocker daemon عاملة
  - يستخدم `HOME` / `XDG_CONFIG_HOME` معزولين، ثم يدمر Gateway وsandbox الخاصين بالاختبار
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_OPENSHELL=1` لتمكين الاختبار عند تشغيل مجموعة e2e الأوسع يدويًا
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` للإشارة إلى CLI binary أو wrapper script غير افتراضي

### Live (موفرون حقيقيون + نماذج حقيقية)

- الأمر: `pnpm test:live`
- الإعداد: `vitest.live.config.ts`
- الملفات: `src/**/*.live.test.ts`
- الافتراضي: **مفعّل** بواسطة `pnpm test:live` (يضبط `OPENCLAW_LIVE_TEST=1`)
- النطاق:
  - "هل يعمل هذا الموفر/النموذج فعلًا _اليوم_ باستخدام بيانات اعتماد حقيقية؟"
  - اكتشاف تغييرات تنسيق الموفر، وخصائص استدعاء الأدوات، ومشكلات المصادقة، وسلوك تحديد المعدل
- التوقعات:
  - غير مستقرة في CI بطبيعتها (شبكات حقيقية، وسياسات موفرين حقيقية، وحصص، وانقطاعات)
  - تكلّف مالًا / تستخدم حدود المعدل
  - يُفضَّل تشغيل مجموعات فرعية مضيقة بدلًا من "كل شيء"
- تستورد التشغيلات الحية `~/.profile` لاكتشاف مفاتيح API المفقودة.
- افتراضيًا، تعزل التشغيلات الحية أيضًا `HOME` وتنسخ مواد config/auth إلى home اختبار مؤقت حتى لا تتمكن تجهيزات unit من تغيير `~/.openclaw` الحقيقي لديك.
- اضبط `OPENCLAW_LIVE_USE_REAL_HOME=1` فقط عندما تحتاج عن قصد إلى أن تستخدم اختبارات live دليل home الحقيقي لديك.
- يستخدم `pnpm test:live` الآن وضعًا أكثر هدوءًا افتراضيًا: فهو يحتفظ بمخرجات التقدم `[live] ...`، لكنه يخفي إشعار `~/.profile` الإضافي ويكتم سجلات إقلاع gateway وضجيج Bonjour. اضبط `OPENCLAW_LIVE_TEST_QUIET=0` إذا أردت سجلات بدء التشغيل الكاملة مجددًا.
- تدوير مفاتيح API (خاص بكل موفر): اضبط `*_API_KEYS` بصيغة فاصلة/فاصلة منقوطة أو `*_API_KEY_1` و`*_API_KEY_2` (مثل `OPENAI_API_KEYS` و`ANTHROPIC_API_KEYS` و`GEMINI_API_KEYS`) أو تجاوزًا لكل live عبر `OPENCLAW_LIVE_*_KEY`؛ تعيد الاختبارات المحاولة عند استجابات تحديد المعدل.
- مخرجات التقدم/نبضات الحياة:
  - تبعث مجموعات live الآن أسطر التقدم إلى stderr حتى تبدو استدعاءات الموفر الطويلة نشطة بوضوح حتى عندما يكون التقاط وحدة تحكم Vitest هادئًا.
  - يعطّل `vitest.live.config.ts` اعتراض وحدة التحكم في Vitest بحيث تتدفق أسطر تقدم الموفر/gateway فورًا أثناء التشغيلات الحية.
  - اضبط نبضات الحياة للنموذج المباشر عبر `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - واضبط نبضات الحياة لـ gateway/probe عبر `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## أي مجموعة يجب أن أشغّل؟

استخدم جدول القرار هذا:

- تعديل المنطق/الاختبارات: شغّل `pnpm test` (و`pnpm test:coverage` إذا غيّرت الكثير)
- لمس شبكات gateway / بروتوكول WS / الاقتران: أضف `pnpm test:e2e`
- تصحيح "الروبوت الخاص بي متوقف" / الإخفاقات الخاصة بموفر معين / استدعاء الأدوات: شغّل `pnpm test:live` مضيقًا

## Live: مسح قدرات عقدة Android

- الاختبار: `src/gateway/android-node.capabilities.live.test.ts`
- السكربت: `pnpm android:test:integration`
- الهدف: استدعاء **كل أمر مُعلن عنه حاليًا** من عقدة Android متصلة والتحقق من سلوك عقدة الأمر.
- النطاق:
  - إعداد مسبق/يدوي (لا تقوم المجموعة بتثبيت التطبيق أو تشغيله أو اقترانه).
  - تحقق `node.invoke` في gateway أمرًا بأمر لعقدة Android المحددة.
- الإعداد المسبق المطلوب:
  - تطبيق Android متصل ومقترن بالفعل مع gateway.
  - إبقاء التطبيق في الواجهة الأمامية.
  - منح الأذونات/موافقة الالتقاط للقدرات التي تتوقع نجاحها.
- تجاوزات هدف اختيارية:
  - `OPENCLAW_ANDROID_NODE_ID` أو `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- تفاصيل إعداد Android الكاملة: [تطبيق Android](/ar/platforms/android)

## Live: فحص النموذج (مفاتيح ملفات التعريف)

تنقسم اختبارات live إلى طبقتين حتى نتمكن من عزل الإخفاقات:

- "النموذج المباشر" يخبرنا ما إذا كان الموفر/النموذج يمكنه الرد أصلًا باستخدام المفتاح المحدد.
- "فحص Gateway" يخبرنا ما إذا كان خط أنابيب gateway+agent الكامل يعمل لهذا النموذج (الجلسات، والسجل، والأدوات، وسياسة sandbox، وما إلى ذلك).

### الطبقة 1: إكمال النموذج المباشر (من دون gateway)

- الاختبار: `src/agents/models.profiles.live.test.ts`
- الهدف:
  - تعداد النماذج المكتشفة
  - استخدام `getApiKeyForModel` لتحديد النماذج التي لديك بيانات اعتماد لها
  - تشغيل إكمال صغير لكل نموذج (واختبارات تراجع مستهدفة عند الحاجة)
- كيفية التمكين:
  - `pnpm test:live` (أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- اضبط `OPENCLAW_LIVE_MODELS=modern` (أو `all`، وهو اسم مستعار لـ modern) لتشغيل هذه المجموعة فعلًا؛ وإلا فسيتم تخطيها للحفاظ على تركيز `pnpm test:live` على فحص gateway
- كيفية اختيار النماذج:
  - `OPENCLAW_LIVE_MODELS=modern` لتشغيل allowlist الحديثة (Opus/Sonnet 4.6+، وGPT-5.x + Codex، وGemini 3، وGLM 4.7، وMiniMax M2.7، وGrok 4)
  - `OPENCLAW_LIVE_MODELS=all` هو اسم مستعار لـ allowlist الحديثة
  - أو `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (allowlist مفصولة بفواصل)
- كيفية اختيار الموفّرين:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity"` (allowlist مفصولة بفواصل)
- من أين تأتي المفاتيح:
  - افتراضيًا: مخزن ملفات التعريف وبدائل env
  - اضبط `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض **مخزن ملفات التعريف** فقط
- سبب وجود هذا:
  - يفصل بين "واجهة API الخاصة بالموفر معطلة / المفتاح غير صالح" و"خط أنابيب gateway agent معطّل"
  - يحتوي على اختبارات تراجع صغيرة ومعزولة (مثال: إعادة تشغيل الاستدلال + تدفقات استدعاء الأدوات في OpenAI Responses/Codex Responses)

### الطبقة 2: فحص Gateway + وكيل dev (ما الذي يفعله "@openclaw" فعليًا)

- الاختبار: `src/gateway/gateway-models.profiles.live.test.ts`
- الهدف:
  - تشغيل gateway داخل العملية
  - إنشاء/تصحيح جلسة `agent:dev:*` (تجاوز النموذج لكل تشغيل)
  - التكرار على النماذج التي لها مفاتيح والتحقق من:
    - استجابة "ذات معنى" (من دون أدوات)
    - نجاح استدعاء أداة حقيقي (مجس read)
    - مجسات أدوات إضافية اختيارية (مجس exec+read)
    - استمرار عمل مسارات التراجع في OpenAI (استدعاء أداة فقط → متابعة)
- تفاصيل المجس (حتى يمكنك شرح الإخفاقات بسرعة):
  - مجس `read`: يكتب الاختبار ملف nonce في مساحة العمل ويطلب من الوكيل أن يقرأه عبر `read` ويعيد nonce.
  - مجس `exec+read`: يطلب الاختبار من الوكيل أن يكتب nonce إلى ملف مؤقت عبر `exec`، ثم يقرأه مجددًا.
  - مجس الصورة: يرفق الاختبار PNG مُنشأً (قطة + رمز عشوائي) ويتوقع أن يعيد النموذج `cat <CODE>`.
  - مرجع التنفيذ: `src/gateway/gateway-models.profiles.live.test.ts` و`src/gateway/live-image-probe.ts`.
- كيفية التمكين:
  - `pnpm test:live` (أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- كيفية اختيار النماذج:
  - الافتراضي: allowlist الحديثة (Opus/Sonnet 4.6+، وGPT-5.x + Codex، وGemini 3، وGLM 4.7، وMiniMax M2.7، وGrok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` هو اسم مستعار لـ allowlist الحديثة
  - أو اضبط `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (أو قائمة مفصولة بفواصل) للتضييق
- كيفية اختيار الموفّرين (تجنب "كل شيء في OpenRouter"):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,openai,anthropic,zai,minimax"` (allowlist مفصولة بفواصل)
- مجسات الأدوات + الصور مفعلة دائمًا في اختبار live هذا:
  - مجس `read` + مجس `exec+read` (ضغط الأدوات)
  - يعمل مجس الصورة عندما يعلن النموذج دعم إدخال الصور
  - التدفق (مستوى عالٍ):
    - يولّد الاختبار PNG صغيرًا جدًا يحمل "CAT" + رمزًا عشوائيًا (`src/gateway/live-image-probe.ts`)
    - يرسله عبر `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - يحلل Gateway المرفقات إلى `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - يمرر الوكيل المضمّن رسالة مستخدم متعددة الوسائط إلى النموذج
    - التحقق: يحتوي الرد على `cat` + الرمز (تسامح OCR: الأخطاء الطفيفة مسموحة)

نصيحة: لمعرفة ما يمكنك اختباره على جهازك (ومعرّفات `provider/model` الدقيقة)، شغّل:

```bash
openclaw models list
openclaw models list --json
```

## Live: فحص ACP bind (`/acp spawn ... --bind here`)

- الاختبار: `src/gateway/gateway-acp-bind.live.test.ts`
- الهدف: التحقق من تدفق ربط المحادثة ACP الحقيقي مع وكيل ACP حي:
  - إرسال `/acp spawn <agent> --bind here`
  - ربط محادثة synthetic message-channel في مكانها
  - إرسال متابعة عادية على المحادثة نفسها
  - التحقق من وصول المتابعة إلى transcript الجلسة المرتبطة بـ ACP
- التمكين:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- الافتراضيات:
  - وكيل ACP: `claude`
  - القناة الاصطناعية: سياق محادثة بأسلوب Slack DM
  - ACP backend: `acpx`
- التجاوزات:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- ملاحظات:
  - يستخدم هذا المسار سطح gateway `chat.send` مع حقول originating-route اصطناعية للمسؤولين فقط حتى تتمكن الاختبارات من إرفاق سياق message-channel من دون التظاهر بالتسليم الخارجي.
  - عندما لا يكون `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` مضبوطًا، يستخدم الاختبار سجل الوكلاء المضمّن داخل إضافة `acpx` لوكيل ACP التجريبي المحدد.

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

ملاحظات Docker:

- يوجد مشغل Docker في `scripts/test-live-acp-bind-docker.sh`.
- يستورد `~/.profile`، ويجهز مواد مصادقة CLI المطابقة داخل الحاوية، ويثبت `acpx` في npm prefix قابل للكتابة، ثم يثبت CLI الحي المطلوب (`@anthropic-ai/claude-code` أو `@openai/codex`) إذا كان مفقودًا.
- داخل Docker، يضبط المشغل `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` حتى يحتفظ acpx بمتغيرات بيئة الموفر من ملف التعريف المستورد المتاحة لـ CLI الفرعي.

### وصفات live الموصى بها

تكون allowlist الضيقة والصريحة أسرع وأقل عرضة لعدم الاستقرار:

- نموذج واحد، مباشر (من دون gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- نموذج واحد، فحص gateway:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- استدعاء الأدوات عبر عدة موفرين:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- تركيز Google (مفتاح Gemini API + Antigravity):
  - Gemini (مفتاح API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

ملاحظات:

- يستخدم `google/...` Gemini API (مفتاح API).
- يستخدم `google-antigravity/...` جسر OAuth الخاص بـ Antigravity (نقطة نهاية وكيل على نمط Cloud Code Assist).

## Live: مصفوفة النماذج (ما الذي نغطيه)

لا توجد "قائمة نماذج CI" ثابتة (live اختيارية)، لكن هذه هي النماذج **الموصى بها** للتغطية بانتظام على جهاز تطوير يملك مفاتيح.

### مجموعة الفحص الحديثة (استدعاء الأدوات + الصور)

هذا هو تشغيل "النماذج الشائعة" الذي نتوقع استمراره في العمل:

- OpenAI (غير Codex): `openai/gpt-5.4` (اختياري: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (أو `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` و`google/gemini-3-flash-preview` (تجنب نماذج Gemini 2.x الأقدم)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` و`google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

شغّل فحص gateway مع الأدوات + الصور:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### الأساس: استدعاء الأدوات (Read + Exec اختياري)

اختر واحدًا على الأقل من كل عائلة موفر:

- OpenAI: `openai/gpt-5.4` (أو `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (أو `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (أو `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

تغطية إضافية اختيارية (من الجيد توفرها):

- xAI: `xai/grok-4` (أو أحدث إصدار متاح)
- Mistral: `mistral/`… (اختر نموذجًا واحدًا قادرًا على "الأدوات" لديك)
- Cerebras: `cerebras/`… (إذا كان لديك وصول)
- LM Studio: `lmstudio/`… (محلي؛ استدعاء الأدوات يعتمد على وضع API)

### Vision: إرسال صورة (مرفق → رسالة متعددة الوسائط)

ضمّن نموذجًا واحدًا على الأقل قادرًا على الصور في `OPENCLAW_LIVE_GATEWAY_MODELS` (Claude/Gemini/إصدارات OpenAI القادرة على الرؤية، إلخ) لاختبار مجس الصورة.

### Aggregators / البوابات البديلة

إذا كانت لديك مفاتيح مفعلة، فنحن ندعم أيضًا الاختبار عبر:

- OpenRouter: `openrouter/...` (مئات النماذج؛ استخدم `openclaw models scan` للعثور على مرشحين قادرين على الأدوات+الصور)
- OpenCode: `opencode/...` لـ Zen و`opencode-go/...` لـ Go (المصادقة عبر `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

موفرون آخرون يمكنك تضمينهم في مصفوفة live (إذا كانت لديك بيانات اعتماد/إعدادات):

- مدمجون: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- عبر `models.providers` (نقاط نهاية مخصصة): `minimax` (سحابي/API)، بالإضافة إلى أي proxy متوافق مع OpenAI/Anthropic (مثل LM Studio وvLLM وLiteLLM وغيرها)

نصيحة: لا تحاول ترميز "كل النماذج" بشكل ثابت في المستندات. القائمة المرجعية هي ما يعيده `discoverModels(...)` على جهازك + أي مفاتيح متاحة.

## بيانات الاعتماد (لا تُرسل أبدًا إلى المستودع)

تكتشف اختبارات live بيانات الاعتماد بالطريقة نفسها التي تعمل بها CLI. والآثار العملية لذلك:

- إذا كانت CLI تعمل، فيجب أن تجد اختبارات live المفاتيح نفسها.
- إذا قالت لك اختبارات live "لا توجد بيانات اعتماد"، فقم بالتصحيح بالطريقة نفسها التي ستصحح بها `openclaw models list` / اختيار النموذج.

- ملفات تعريف المصادقة لكل وكيل: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (هذا هو المقصود بـ "مفاتيح ملفات التعريف" في اختبارات live)
- الإعدادات: `~/.openclaw/openclaw.json` (أو `OPENCLAW_CONFIG_PATH`)
- دليل الحالة القديم: `~/.openclaw/credentials/` (يُنسخ إلى home الحي المجهز عند وجوده، لكنه ليس مخزن مفاتيح ملفات التعريف الرئيسي)
- تنسخ التشغيلات المحلية الحية الإعدادات النشطة وملفات `auth-profiles.json` لكل وكيل و`credentials/` القديمة وأدلة مصادقة CLI الخارجية المدعومة إلى home اختبار مؤقت افتراضيًا؛ كما تُزال تجاوزات المسار `agents.*.workspace` / `agentDir` من تلك الإعدادات المجهزة حتى تبقى المجسات بعيدة عن مساحة العمل الحقيقية على المضيف.

إذا أردت الاعتماد على مفاتيح env (مثل المفاتيح المصدّرة في `~/.profile`)، فشغّل الاختبارات المحلية بعد `source ~/.profile`، أو استخدم مشغلات Docker أدناه (يمكنها ربط `~/.profile` داخل الحاوية).

## Deepgram live (تفريغ الصوت)

- الاختبار: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- التمكين: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- الاختبار: `src/agents/byteplus.live.test.ts`
- التمكين: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- تجاوز نموذج اختياري: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- الاختبار: `extensions/comfy/comfy.live.test.ts`
- التمكين: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- النطاق:
  - يختبر المسارات المضمنة للصور والفيديو و`music_generate` في comfy
  - يتخطى كل قدرة ما لم يكن `models.providers.comfy.<capability>` مهيأً
  - مفيد بعد تغيير إرسال سير عمل comfy أو polling أو التنزيلات أو تسجيل الإضافة

## Image generation live

- الاختبار: `src/image-generation/runtime.live.test.ts`
- الأمر: `pnpm test:live src/image-generation/runtime.live.test.ts`
- النطاق:
  - يعدّد كل إضافة موفر مسجلة لإنشاء الصور
  - يحمّل متغيرات env الخاصة بالموفر والمفقودة من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/الموجودة في env قبل ملفات تعريف المصادقة المخزنة افتراضيًا، حتى لا تخفي مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات اعتماد shell الحقيقية
  - يتخطى الموفّرين الذين لا يملكون مصادقة/ملف تعريف/نموذجًا صالحًا
  - يشغّل متغيرات إنشاء الصور القياسية عبر قدرة وقت التشغيل المشتركة:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- الموفّرون المضمنون المغطّون حاليًا:
  - `openai`
  - `google`
- تضييق اختياري:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- سلوك مصادقة اختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن ملفات التعريف وتجاهل تجاوزات env فقط

## Music generation live

- الاختبار: `extensions/music-generation-providers.live.test.ts`
- التمكين: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- النطاق:
  - يختبر مسار موفر إنشاء الموسيقى المضمن المشترك
  - يغطي حاليًا Google وMiniMax
  - يحمّل متغيرات env الخاصة بالموفر من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يتخطى الموفّرين الذين لا يملكون مصادقة/ملف تعريف/نموذجًا صالحًا
- تضييق اختياري:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`

## مشغلات Docker (فحوصات اختيارية من نوع "يعمل على Linux")

تنقسم مشغلات Docker هذه إلى فئتين:

- مشغلات live-model: يشغّل `test:docker:live-models` و`test:docker:live-gateway` فقط ملف live المطابق المعتمد على مفاتيح ملفات التعريف داخل صورة Docker الخاصة بالمستودع (`src/agents/models.profiles.live.test.ts` و`src/gateway/gateway-models.profiles.live.test.ts`)، مع ربط دليل الإعدادات المحلي ومساحة العمل لديك (واستيراد `~/.profile` إذا تم ربطه). وتُعد نقاط الدخول المحلية المطابقة هما `test:live:models-profiles` و`test:live:gateway-profiles`.
- تستخدم مشغلات Docker الحية افتراضيًا حدًا أصغر للفحص حتى يبقى اجتياح Docker الكامل عمليًا:
  يضبط `test:docker:live-models` افتراضيًا `OPENCLAW_LIVE_MAX_MODELS=12`، ويضبط
  `test:docker:live-gateway` افتراضيًا `OPENCLAW_LIVE_GATEWAY_SMOKE=1`،
  و`OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`،
  و`OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`، و
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. يمكنك تجاوز هذه المتغيرات عندما
  تريد صراحةً الفحص الأكبر والأشمل.
- يبني `test:docker:all` صورة live Docker مرة واحدة عبر `test:docker:live-build`، ثم يعيد استخدامها لمساري Docker الحيين.
- مشغلات فحص الحاويات: تشغّل `test:docker:openwebui` و`test:docker:onboard` و`test:docker:gateway-network` و`test:docker:mcp-channels` و`test:docker:plugins` حاوية واحدة أو أكثر حقيقية وتتحقق من مسارات integration عالية المستوى.

تقوم مشغلات Docker الخاصة بـ live-model أيضًا بربط فقط أدلة مصادقة CLI المطلوبة (أو كل الأدلة المدعومة عندما لا يكون التشغيل مضيقًا)، ثم تنسخها إلى home الحاوية قبل التشغيل حتى يتمكن OAuth الخاص بـ CLI الخارجي من تحديث الرموز من دون تغيير مخزن المصادقة على المضيف:

- النماذج المباشرة: `pnpm test:docker:live-models` (السكربت: `scripts/test-live-models-docker.sh`)
- ACP bind smoke: `pnpm test:docker:live-acp-bind` (السكربت: `scripts/test-live-acp-bind-docker.sh`)
- Gateway + وكيل dev: `pnpm test:docker:live-gateway` (السكربت: `scripts/test-live-gateway-models-docker.sh`)
- فحص Open WebUI live: `pnpm test:docker:openwebui` (السكربت: `scripts/e2e/openwebui-docker.sh`)
- معالج onboarding (TTY، وتجهيز كامل): `pnpm test:docker:onboard` (السكربت: `scripts/e2e/onboard-docker.sh`)
- شبكات Gateway (حاويتان، ومصادقة WS + health): `pnpm test:docker:gateway-network` (السكربت: `scripts/e2e/gateway-network-docker.sh`)
- جسر قناة MCP (Gateway مهيأ مسبقًا + جسر stdio + فحص خام لإطارات إشعارات Claude): `pnpm test:docker:mcp-channels` (السكربت: `scripts/e2e/mcp-channels-docker.sh`)
- الإضافات (فحص التثبيت + الاسم المستعار `/plugin` + دلالات إعادة تشغيل Claude-bundle): `pnpm test:docker:plugins` (السكربت: `scripts/e2e/plugins-docker.sh`)

تقوم مشغلات Docker الخاصة بـ live-model أيضًا بربط نسخة السحب الحالية للقراءة فقط
وتهيئتها في workdir مؤقت داخل الحاوية. وهذا يُبقي صورة وقت التشغيل نحيفة
مع استمرار تشغيل Vitest على المصدر/الإعدادات المحلية الدقيقة الخاصة بك.
تتخطى خطوة التهيئة مخازن cache المحلية الكبيرة فقط ومخرجات بناء التطبيقات مثل
`.pnpm-store` و`.worktrees` و`__openclaw_vitest__` ومجلدات `.build` المحلية للتطبيق أو
مجلدات مخرجات Gradle، حتى لا تقضي تشغيلات Docker الحية دقائق في نسخ
مصنوعات خاصة بالجهاز.
كما تضبط `OPENCLAW_SKIP_CHANNELS=1` حتى لا تبدأ مجسات gateway الحية
عوامل القنوات الحقيقية مثل Telegram/Discord/etc. داخل الحاوية.
لا يزال `test:docker:live-models` يشغّل `pnpm test:live`، لذا مرّر
`OPENCLAW_LIVE_GATEWAY_*` أيضًا عندما تحتاج إلى تضييق أو استبعاد تغطية gateway
الحية من مسار Docker هذا.
يمثل `test:docker:openwebui` فحص توافق عالي المستوى: فهو يبدأ
حاوية OpenClaw gateway مع تمكين نقاط نهاية HTTP المتوافقة مع OpenAI،
ويبدأ حاوية Open WebUI مثبتة الإصدار ضد ذلك gateway، ويسجل الدخول عبر
Open WebUI، ويتحقق من أن `/api/models` يعرض `openclaw/default`، ثم يرسل
طلب دردشة حقيقيًا عبر proxy الخاص بـ `/api/chat/completions` في Open WebUI.
قد يكون التشغيل الأول أبطأ بشكل ملحوظ لأن Docker قد يحتاج إلى سحب
صورة Open WebUI وقد يحتاج Open WebUI إلى إكمال إعداد البدء البارد الخاص به.
يتوقع هذا المسار مفتاح نموذج حي صالحًا، ويُعد `OPENCLAW_PROFILE_FILE`
(`~/.profile` افتراضيًا) الطريقة الأساسية لتوفيره في التشغيلات ضمن Docker.
تطبع التشغيلات الناجحة حمولة JSON صغيرة مثل `{ "ok": true, "model":
"openclaw/default", ... }`.
تم تصميم `test:docker:mcp-channels` ليكون حتميًا عن قصد ولا يحتاج إلى
حساب Telegram أو Discord أو iMessage حقيقي. فهو يشغّل حاوية Gateway
مهيأة مسبقًا، ثم يبدأ حاوية ثانية تشغّل `openclaw mcp serve`، ثم
يتحقق من اكتشاف المحادثات الموجّهة، وقراءات transcript، وبيانات المرفقات الوصفية،
وسلوك قائمة الأحداث الحية، وتوجيه الإرسال الخارجي، وإشعارات القنوات +
الأذونات على نمط Claude عبر جسر stdio MCP الحقيقي. ويفحص تحقق الإشعارات
إطارات stdio MCP الخام مباشرة، بحيث يثبت الفحص ما الذي يبثه الجسر فعلًا،
وليس فقط ما يعرضه SDK عميل معين.

فحص يدوي لمسار ACP باللغة الطبيعية (ليس في CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- احتفظ بهذا السكربت لسير عمل اختبارات التراجع/تصحيح الأخطاء. فقد تكون هناك حاجة إليه مجددًا للتحقق من توجيه سلاسل ACP، لذا لا تحذفه.

متغيرات env مفيدة:

- `OPENCLAW_CONFIG_DIR=...` (الافتراضي: `~/.openclaw`) يُربط إلى `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (الافتراضي: `~/.openclaw/workspace`) يُربط إلى `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (الافتراضي: `~/.profile`) يُربط إلى `/home/node/.profile` ويُستورد قبل تشغيل الاختبارات
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (الافتراضي: `~/.cache/openclaw/docker-cli-tools`) يُربط إلى `/home/node/.npm-global` لتخزين تثبيتات CLI مؤقتًا داخل Docker
- تُربط أدلة/ملفات مصادقة CLI الخارجية تحت `$HOME` للقراءة فقط تحت `/host-auth...`، ثم تُنسخ إلى `/home/node/...` قبل بدء الاختبارات
  - الأدلة الافتراضية: `.minimax`
  - الملفات الافتراضية: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - في التشغيلات المضيقة حسب الموفر، تُربط فقط الأدلة/الملفات اللازمة المستنتجة من `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - تجاوز يدوي عبر `OPENCLAW_DOCKER_AUTH_DIRS=all` أو `OPENCLAW_DOCKER_AUTH_DIRS=none` أو قائمة مفصولة بفواصل مثل `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` لتضييق التشغيل
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` لتصفية الموفّرين داخل الحاوية
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لضمان أن تأتي بيانات الاعتماد من مخزن ملفات التعريف (وليس env)
- `OPENCLAW_OPENWEBUI_MODEL=...` لاختيار النموذج الذي يعرضه gateway لفحص Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` لتجاوز prompt التحقق من nonce المستخدمة في فحص Open WebUI
- `OPENWEBUI_IMAGE=...` لتجاوز وسم صورة Open WebUI المثبتة

## التحقق السريع من المستندات

شغّل فحوصات المستندات بعد تعديلها: `pnpm check:docs`.
وشغّل التحقق الكامل من Mintlify anchors عندما تحتاج أيضًا إلى فحوصات عناوين داخل الصفحة: `pnpm docs:check-links:anchors`.

## اختبار التراجع دون اتصال (آمن لـ CI)

هذه اختبارات تراجع "لخط الأنابيب الحقيقي" من دون موفرين حقيقيين:

- استدعاء أدوات Gateway (OpenAI وهمي، وgateway حقيقي + حلقة وكيل): `src/gateway/gateway.test.ts` (الحالة: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- معالج Gateway (WS `wizard.start`/`wizard.next`، يكتب config + auth المفروض): `src/gateway/gateway.test.ts` (الحالة: "runs wizard over ws and writes auth token config")

## تقييمات موثوقية الوكيل (Skills)

لدينا بالفعل بعض الاختبارات الآمنة لـ CI التي تتصرف مثل "تقييمات موثوقية الوكيل":

- استدعاء أدوات وهمي عبر gateway الحقيقي + حلقة الوكيل (`src/gateway/gateway.test.ts`).
- تدفقات المعالج من طرف إلى طرف التي تتحقق من wiring الجلسة وتأثيرات config (`src/gateway/gateway.test.ts`).

ما لا يزال مفقودًا لـ Skills (راجع [Skills](/ar/tools/skills)):

- **اتخاذ القرار:** عندما تُدرج Skills في prompt، هل يختار الوكيل Skill الصحيحة (أو يتجنب غير ذات الصلة)؟
- **الامتثال:** هل يقرأ الوكيل `SKILL.md` قبل الاستخدام ويتبع الخطوات/الوسائط المطلوبة؟
- **عقود سير العمل:** سيناريوهات متعددة الأدوار تتحقق من ترتيب الأدوات، واستمرار session history، وحدود sandbox.

يجب أن تبقى التقييمات المستقبلية حتمية أولًا:

- مشغّل سيناريوهات يستخدم موفرين وهميين للتحقق من استدعاءات الأدوات + ترتيبها، وقراءات ملفات Skill، وwiring الجلسة.
- مجموعة صغيرة من السيناريوهات المركزة على Skills (الاستخدام مقابل التجنب، والبوابات، وحقن prompt).
- تقييمات live اختيارية (opt-in ومقيدة بـ env) فقط بعد وجود المجموعة الآمنة لـ CI.

## اختبارات العقد (شكل plugin وchannel)

تتحقق اختبارات العقد من أن كل plugin وchannel مسجل يطابق
عقد الواجهة الخاص به. فهي تتكرر على جميع plugins المكتشفة وتشغّل مجموعة من
اختبارات الشكل والسلوك. ويتخطى مسار unit الافتراضي `pnpm test` عمدًا
هذه الملفات الخاصة بالواجهات المشتركة وملفات الفحص؛ شغّل أوامر العقد صراحةً
عندما تلمس الأسطح المشتركة للقنوات أو الموفّرين.

### الأوامر

- جميع العقود: `pnpm test:contracts`
- عقود القنوات فقط: `pnpm test:contracts:channels`
- عقود الموفّرين فقط: `pnpm test:contracts:plugins`

### عقود القنوات

تقع في `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - الشكل الأساسي للإضافة (المعرّف، والاسم، والقدرات)
- **setup** - عقد معالج الإعداد
- **session-binding** - سلوك ربط الجلسة
- **outbound-payload** - بنية حمولة الرسالة
- **inbound** - معالجة الرسائل الواردة
- **actions** - معالجات إجراءات القناة
- **threading** - معالجة معرّف السلسلة
- **directory** - واجهة directory/roster API
- **group-policy** - فرض سياسة المجموعات

### عقود حالة الموفّر

تقع في `src/plugins/contracts/*.contract.test.ts`.

- **status** - مجسات حالة القناة
- **registry** - شكل سجل plugin

### عقود الموفّر

تقع في `src/plugins/contracts/*.contract.test.ts`:

- **auth** - عقد تدفق المصادقة
- **auth-choice** - اختيار/تحديد المصادقة
- **catalog** - API فهرس النماذج
- **discovery** - اكتشاف plugin
- **loader** - تحميل plugin
- **runtime** - وقت تشغيل الموفّر
- **shape** - شكل/واجهة plugin
- **wizard** - معالج الإعداد

### متى تُشغَّل

- بعد تغيير صادرات plugin-sdk أو المسارات الفرعية
- بعد إضافة أو تعديل channel أو provider plugin
- بعد إعادة هيكلة تسجيل plugin أو اكتشافه

تعمل اختبارات العقد في CI ولا تتطلب مفاتيح API حقيقية.

## إضافة اختبارات تراجع (إرشادات)

عندما تصلح مشكلة في موفر/نموذج اكتُشفت في live:

- أضف اختبار تراجع آمنًا لـ CI إن أمكن (موفر وهمي/بديل، أو التقط تحويل شكل الطلب الدقيق)
- إذا كانت المشكلة بطبيعتها مرتبطة بـ live فقط (حدود المعدل، أو سياسات المصادقة)، فأبقِ اختبار live ضيقًا واختياريًا عبر متغيرات env
- افضّل استهداف أصغر طبقة تلتقط الخطأ:
  - خطأ في تحويل/إعادة تشغيل طلب الموفّر → اختبار النماذج المباشرة
  - خطأ في مسار جلسة/سجل/أداة gateway → فحص gateway live أو اختبار gateway وهمي آمن لـ CI
- حاجز حماية اجتياز SecretRef:
  - يستمد `src/secrets/exec-secret-ref-id-parity.test.ts` هدفًا نموذجيًا واحدًا لكل فئة SecretRef من بيانات السجل الوصفية (`listSecretTargetRegistryEntries()`)، ثم يتحقق من رفض معرّفات exec الخاصة بمقاطع الاجتياز.
  - إذا أضفت عائلة أهداف SecretRef جديدة من نوع `includeInPlan` في `src/secrets/target-registry-data.ts`، فحدّث `classifyTargetClass` في ذلك الاختبار. يفشل الاختبار عمدًا عند المعرّفات غير المصنفة حتى لا يمكن تخطي الفئات الجديدة بصمت.
