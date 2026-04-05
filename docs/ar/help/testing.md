---
read_when:
    - تشغيل الاختبارات محليًا أو في CI
    - إضافة حالات تراجع لأخطاء النماذج/المزوّدين
    - تصحيح سلوك Gateway + الوكيل
summary: 'مجموعة الاختبار: أجنحة unit/e2e/live، ومشغلات Docker، وما الذي يغطيه كل اختبار'
title: الاختبار
x-i18n:
    generated_at: "2026-04-05T12:47:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 854a39ae261d8749b8d8d82097b97a7c52cf2216d1fe622e302d830a888866ab
    source_path: help/testing.md
    workflow: 15
---

# الاختبار

يحتوي OpenClaw على ثلاثة أجنحة Vitest ‏(unit/integration وe2e وlive) ومجموعة صغيرة من مشغلات Docker.

هذه الوثيقة هي دليل “كيف نختبر”:

- ما الذي يغطيه كل جناح (وما الذي لا يغطيه عمدًا)
- ما الأوامر التي يجب تشغيلها لسيناريوهات العمل الشائعة (محليًا، قبل push، التصحيح)
- كيف تكتشف الاختبارات الحية بيانات الاعتماد وتختار النماذج/المزوّدين
- كيفية إضافة حالات تراجع لمشكلات النماذج/المزوّدين في العالم الحقيقي

## بدء سريع

في معظم الأيام:

- البوابة الكاملة (المتوقعة قبل push): `pnpm build && pnpm check && pnpm test`
- تشغيل محلي أسرع للمجموعة الكاملة على جهاز واسع الموارد: `pnpm test:max`
- حلقة watch مباشرة لـ Vitest ‏(إعداد modern projects): `pnpm test:watch`
- استهداف ملف مباشر يوجّه الآن أيضًا مسارات extension/channel: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`

عندما تلمس الاختبارات أو تريد ثقة إضافية:

- بوابة التغطية: `pnpm test:coverage`
- جناح E2E: ‏`pnpm test:e2e`

عند تصحيح مزوّدين/نماذج حقيقية (يتطلب بيانات اعتماد حقيقية):

- الجناح live ‏(النماذج + فحوصات أدوات/صور gateway): `pnpm test:live`
- استهداف ملف live واحد بهدوء: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

نصيحة: عندما تحتاج فقط إلى حالة فشل واحدة، فافضّل تضييق اختبارات live عبر متغيرات البيئة الخاصة بقائمة السماح الموضحة أدناه.

## أجنحة الاختبار (ما الذي يعمل وأين)

فكّر في الأجنحة على أنها “زيادة في الواقعية” (وزيادة في عدم الاستقرار/التكلفة):

### Unit / integration ‏(الافتراضي)

- الأمر: `pnpm test`
- الإعداد: `projects` أصلية في Vitest عبر `vitest.config.ts`
- الملفات: قوائم الجرد الأساسية لـ core/unit تحت `src/**/*.test.ts` و`packages/**/*.test.ts` و`test/**/*.test.ts` واختبارات `ui` لعقدة node المدرجة في القائمة البيضاء والمشمولة بواسطة `vitest.unit.config.ts`
- النطاق:
  - اختبارات unit خالصة
  - اختبارات integration داخل العملية (مصادقة gateway، والتوجيه، والأدوات، والتحليل، والإعدادات)
  - حالات تراجع حتمية للأخطاء المعروفة
- التوقعات:
  - تعمل في CI
  - لا تتطلب مفاتيح حقيقية
  - يجب أن تكون سريعة ومستقرة
- ملاحظة projects:
  - تستخدم الآن `pnpm test` و`pnpm test:watch` و`pnpm test:changed` إعداد `projects` نفسه من جذر Vitest.
  - يوجّه ترشيح الملفات المباشر أصلًا عبر رسم المشاريع في الجذر، لذا يعمل `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` من دون wrapper مخصص.
- ملاحظة المشغّل المضمّن:
  - عندما تغيّر مدخلات اكتشاف أداة الرسائل أو سياق وقت تشغيل compaction،
    حافظ على مستويي التغطية معًا.
  - أضف حالات تراجع مركّزة للمساعد لحدود التوجيه/التطبيع الخالصة.
  - وحافظ أيضًا على صحة أجنحة integration الخاصة بالمشغّل المضمّن:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    و`src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`، و
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - تتحقق هذه الأجنحة من أن المعرّفات المحددة وسلوك compaction لا يزالان يتدفقان
    عبر المسارات الحقيقية `run.ts` / `compact.ts`؛ واختبارات المساعد فقط
    ليست بديلًا كافيًا عن مسارات integration هذه.
- ملاحظة pool:
  - أصبح إعداد Vitest الأساسي الآن يستخدم `threads` افتراضيًا.
  - كما يثبت إعداد Vitest المشترك الآن `isolate: false` ويستخدم المشغّل غير المعزول عبر مشاريع الجذر وتهيئات e2e وlive.
  - يحتفظ مسار UI في الجذر بإعداد `jsdom` والمُحسّن الخاص به، لكنه يعمل الآن أيضًا على المشغّل غير المعزول المشترك.
  - يرث `pnpm test` القيم الافتراضية نفسها `threads` + `isolate: false` من إعداد المشاريع في `vitest.config.ts` في الجذر.
  - يضيف المشغّل المشترك `scripts/run-vitest.mjs` الآن أيضًا الخيار `--no-maglev` افتراضيًا لعمليات Node الفرعية الخاصة بـ Vitest لتقليل تبدل تجميع V8 أثناء التشغيلات المحلية الكبيرة. اضبط `OPENCLAW_VITEST_ENABLE_MAGLEV=1` إذا كنت تحتاج إلى المقارنة مع سلوك V8 الافتراضي.
- ملاحظة التكرار المحلي السريع:
  - يشغّل `pnpm test:changed` إعداد المشاريع الأصلي مع `--changed origin/main`.
  - يحتفظ `pnpm test:max` و`pnpm test:changed:max` بإعداد المشاريع الأصلي نفسه، لكن مع حد أعلى للعاملين.
  - أصبح التحجيم التلقائي للعاملين محليًا محافظًا عمدًا الآن ويتراجع أيضًا عندما يكون متوسط حمل المضيف مرتفعًا بالفعل، بحيث تتسبب تشغيلات Vitest المتزامنة المتعددة بضرر أقل افتراضيًا.
  - يعلّم إعداد Vitest الأساسي ملفات المشاريع/الإعدادات على أنها `forceRerunTriggers` حتى تبقى إعادة التشغيل في وضع changed صحيحة عندما تتغير أسلاك الاختبار.
  - يُبقي الإعداد `OPENCLAW_VITEST_FS_MODULE_CACHE` مفعّلًا على المضيفين المدعومين؛ اضبط `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` إذا كنت تريد موقع cache صريحًا واحدًا لأغراض التحليل المباشر.
- ملاحظة تصحيح الأداء:
  - يفعّل `pnpm test:perf:imports` تقارير مدة الاستيراد في Vitest بالإضافة إلى خرج تفصيلي للاستيراد.
  - يحصر `pnpm test:perf:imports:changed` عرض التحليل نفسه في الملفات التي تغيرت منذ `origin/main`.
  - يكتب `pnpm test:perf:profile:main` ملف CPU profile للخيط الرئيسي لتكاليف بدء التشغيل والتحويل في Vitest/Vite.
  - يكتب `pnpm test:perf:profile:runner` ملفات CPU+heap profile للمشغّل لجناح unit مع تعطيل توازي الملفات.

### E2E ‏(اختبار smoke للـ gateway)

- الأمر: `pnpm test:e2e`
- الإعداد: `vitest.e2e.config.ts`
- الملفات: `src/**/*.e2e.test.ts`، و`test/**/*.e2e.test.ts`
- القيم الافتراضية لوقت التشغيل:
  - يستخدم Vitest ‏`threads` مع `isolate: false`، بما يطابق بقية المستودع.
  - يستخدم عاملين تكيفيين (في CI: حتى 2، ومحليًا: 1 افتراضيًا).
  - يعمل في الوضع الصامت افتراضيًا لتقليل تكلفة الإدخال/الإخراج إلى console.
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_WORKERS=<n>` لفرض عدد العاملين (بحد أقصى 16).
  - `OPENCLAW_E2E_VERBOSE=1` لإعادة تمكين خرج console المفصل.
- النطاق:
  - سلوك gateway من طرف إلى طرف متعدد النسخ
  - أسطح WebSocket/HTTP، واقتران العقد، والشبكات الأثقل
- التوقعات:
  - يعمل في CI ‏(عند تمكينه في المسار)
  - لا يتطلب مفاتيح حقيقية
  - يحتوي على أجزاء متحركة أكثر من اختبارات unit ‏(وقد يكون أبطأ)

### E2E: اختبار smoke لخلفية OpenShell

- الأمر: `pnpm test:e2e:openshell`
- الملف: `test/openshell-sandbox.e2e.test.ts`
- النطاق:
  - يبدأ OpenShell gateway معزولًا على المضيف عبر Docker
  - ينشئ sandbox من Dockerfile محلي مؤقت
  - يختبر خلفية OpenShell في OpenClaw عبر `sandbox ssh-config` + تنفيذ SSH الحقيقي
  - يتحقق من سلوك نظام الملفات المرجعي البعيد عبر جسر نظام ملفات sandbox
- التوقعات:
  - اشتراك اختياري فقط؛ ليس جزءًا من التشغيل الافتراضي لـ `pnpm test:e2e`
  - يتطلب CLI محليًا لـ `openshell` بالإضافة إلى daemon Docker عامل
  - يستخدم `HOME` / `XDG_CONFIG_HOME` معزولين، ثم يدمّر gateway وsandbox الخاصين بالاختبار
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_OPENSHELL=1` لتمكين الاختبار عند تشغيل جناح e2e الأوسع يدويًا
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` للإشارة إلى ملف CLI تنفيذي غير افتراضي أو wrapper script

### Live ‏(مزوّدون حقيقيون + نماذج حقيقية)

- الأمر: `pnpm test:live`
- الإعداد: `vitest.live.config.ts`
- الملفات: `src/**/*.live.test.ts`
- الافتراضي: **مفعّل** بواسطة `pnpm test:live` ‏(يضبط `OPENCLAW_LIVE_TEST=1`)
- النطاق:
  - “هل يعمل هذا المزوّد/النموذج فعلًا _اليوم_ ببيانات اعتماد حقيقية؟”
  - التقاط تغييرات تنسيق المزوّد، وخصائص استدعاء الأدوات، ومشكلات المصادقة، وسلوك حدود المعدل
- التوقعات:
  - غير مستقرة بطبيعتها في CI ‏(شبكات حقيقية، وسياسات مزوّدين حقيقية، وحصص، وانقطاعات)
  - تكلف مالًا / تستخدم حدود المعدل
  - يُفضّل تشغيل مجموعات فرعية ضيقة بدلًا من “كل شيء”
- تستورد تشغيلات live الملف `~/.profile` لالتقاط مفاتيح API المفقودة.
- افتراضيًا، لا تزال تشغيلات live تعزل `HOME` وتنسخ مواد الإعدادات/المصادقة إلى home اختبار مؤقت بحيث لا تتمكن تجهيزات unit من تعديل `~/.openclaw` الحقيقي لديك.
- اضبط `OPENCLAW_LIVE_USE_REAL_HOME=1` فقط عندما تحتاج عمدًا إلى أن تستخدم اختبارات live دليل home الحقيقي لديك.
- يستخدم `pnpm test:live` الآن افتراضيًا وضعًا أكثر هدوءًا: فهو يبقي خرج التقدم `[live] ...`، لكنه يخفي إشعار `~/.profile` الإضافي ويكتم سجلات تمهيد gateway وضجيج Bonjour. اضبط `OPENCLAW_LIVE_TEST_QUIET=0` إذا كنت تريد سجلات بدء التشغيل الكاملة مجددًا.
- تدوير مفاتيح API ‏(خاص بكل مزوّد): اضبط `*_API_KEYS` بتنسيق الفواصل/الفواصل المنقوطة أو `*_API_KEY_1` و`*_API_KEY_2` ‏(مثل `OPENAI_API_KEYS` و`ANTHROPIC_API_KEYS` و`GEMINI_API_KEYS`) أو تجاوزًا خاصًا بـ live عبر `OPENCLAW_LIVE_*_KEY`؛ وتحاول الاختبارات مجددًا عند استجابات حدود المعدل.
- خرج التقدم/heartbeat:
  - تصدر أجنحة live الآن أسطر التقدم إلى stderr بحيث تظهر استدعاءات المزوّد الطويلة على أنها نشطة بصريًا حتى عندما يكون التقاط console في Vitest هادئًا.
  - يعطل `vitest.live.config.ts` اعتراض console في Vitest بحيث تتدفق أسطر تقدم المزوّد/gateway فورًا أثناء تشغيلات live.
  - اضبط heartbeat للنموذج المباشر باستخدام `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - اضبط heartbeat للـ gateway/probe باستخدام `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## أي جناح يجب أن أشغّله؟

استخدم جدول القرار هذا:

- تعديل المنطق/الاختبارات: شغّل `pnpm test` ‏(و`pnpm test:coverage` إذا كنت قد غيّرت كثيرًا)
- لمس شبكات gateway / بروتوكول WS / الاقتران: أضف `pnpm test:e2e`
- تصحيح “البوت متوقف” / أعطال خاصة بالمزوّد / استدعاء الأدوات: شغّل `pnpm test:live` مضيقًا

## Live: مسح قدرات عقدة Android

- الاختبار: `src/gateway/android-node.capabilities.live.test.ts`
- البرنامج النصي: `pnpm android:test:integration`
- الهدف: استدعاء **كل أمر معلن حاليًا** بواسطة عقدة Android متصلة والتحقق من سلوك عقد الأمر.
- النطاق:
  - إعداد مسبق/يدوي (لا يقوم الجناح بتثبيت التطبيق أو تشغيله أو إقرانه).
  - تحقق `node.invoke` في gateway لكل أمر بالنسبة إلى عقدة Android المحددة.
- الإعداد المسبق المطلوب:
  - تطبيق Android متصل بالفعل ومقترن بـ gateway.
  - إبقاء التطبيق في الواجهة الأمامية.
  - منح الأذونات/موافقة الالتقاط للقدرات التي تتوقع نجاحها.
- تجاوزات الهدف الاختيارية:
  - `OPENCLAW_ANDROID_NODE_ID` أو `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- تفاصيل إعداد Android الكاملة: [تطبيق Android](/platforms/android)

## Live: اختبار smoke للنموذج (مفاتيح profile)

تنقسم اختبارات live إلى طبقتين حتى نتمكن من عزل الإخفاقات:

- “النموذج المباشر” يخبرنا هل يستطيع المزوّد/النموذج الرد أصلًا باستخدام المفتاح المعطى.
- “اختبار smoke للـ Gateway” يخبرنا هل يعمل خط أنابيب gateway+agent الكامل مع هذا النموذج (الجلسات، والسجل، والأدوات، وسياسة sandbox، إلخ).

### الطبقة 1: إكمال النموذج المباشر (بدون gateway)

- الاختبار: `src/agents/models.profiles.live.test.ts`
- الهدف:
  - تعداد النماذج المكتشفة
  - استخدام `getApiKeyForModel` لتحديد النماذج التي لديك بيانات اعتماد لها
  - تشغيل إكمال صغير لكل نموذج (وحالات تراجع مستهدفة عند الحاجة)
- كيفية التمكين:
  - `pnpm test:live` ‏(أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- اضبط `OPENCLAW_LIVE_MODELS=modern` ‏(أو `all`، وهو اسم بديل لـ modern) لتشغيل هذا الجناح فعلًا؛ وإلا يتم تخطيه للحفاظ على تركيز `pnpm test:live` على اختبار smoke للـ gateway
- كيفية اختيار النماذج:
  - `OPENCLAW_LIVE_MODELS=modern` لتشغيل قائمة السماح الحديثة (Opus/Sonnet 4.6+، وGPT-5.x + Codex، وGemini 3، وGLM 4.7، وMiniMax M2.7، وGrok 4)
  - `OPENCLAW_LIVE_MODELS=all` هو اسم بديل لقائمة السماح الحديثة
  - أو `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` ‏(قائمة سماح مفصولة بفواصل)
- كيفية اختيار المزوّدين:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` ‏(قائمة سماح مفصولة بفواصل)
- من أين تأتي المفاتيح:
  - افتراضيًا: من مخزن profile والتراجعات البيئية
  - اضبط `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض **مخزن profile** فقط
- لماذا يوجد هذا:
  - يفصل بين “واجهة API للمزوّد معطلة / المفتاح غير صالح” و“خط أنابيب وكيل gateway معطل”
  - يحتوي على حالات تراجع صغيرة ومعزولة (مثال: إعادة تشغيل reasoning replay + تدفقات استدعاء الأدوات في OpenAI Responses/Codex Responses)

### الطبقة 2: Gateway + اختبار smoke لوكيل dev ‏(ما الذي يفعله "@openclaw" فعليًا)

- الاختبار: `src/gateway/gateway-models.profiles.live.test.ts`
- الهدف:
  - تشغيل gateway داخل العملية
  - إنشاء/ترقيع جلسة `agent:dev:*` ‏(مع تجاوز النموذج لكل تشغيل)
  - التكرار عبر النماذج التي تملك مفاتيح والتحقق من:
    - استجابة “ذات معنى” (بدون أدوات)
    - أن استدعاء أداة حقيقية يعمل (`read` probe)
    - فحوصات أدوات إضافية اختيارية (`exec+read` probe)
    - أن مسارات التراجع في OpenAI ‏(استدعاء أداة فقط → متابعة) تظل تعمل
- تفاصيل probe ‏(حتى تتمكن من شرح الأعطال بسرعة):
  - probe ‏`read`: يكتب الاختبار ملف nonce في مساحة العمل ويطلب من الوكيل `read` له وإرجاع الـ nonce.
  - probe ‏`exec+read`: يطلب الاختبار من الوكيل كتابة nonce إلى ملف مؤقت عبر `exec`، ثم `read` له مجددًا.
  - probe الصورة: يرفق الاختبار PNG مُنشأً (قط + رمز عشوائي) ويتوقع من النموذج إرجاع `cat <CODE>`.
  - مرجع التنفيذ: `src/gateway/gateway-models.profiles.live.test.ts` و`src/gateway/live-image-probe.ts`.
- كيفية التمكين:
  - `pnpm test:live` ‏(أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- كيفية اختيار النماذج:
  - الافتراضي: قائمة السماح الحديثة (Opus/Sonnet 4.6+، وGPT-5.x + Codex، وGemini 3، وGLM 4.7، وMiniMax M2.7، وGrok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` هو اسم بديل لقائمة السماح الحديثة
  - أو اضبط `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` ‏(أو قائمة مفصولة بفواصل) للتضييق
- كيفية اختيار المزوّدين (لتجنب “كل شيء عبر OpenRouter”):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` ‏(قائمة سماح مفصولة بفواصل)
- تكون فحوصات الأدوات + الصور مفعّلة دائمًا في اختبار live هذا:
  - probe ‏`read` + probe ‏`exec+read` ‏(ضغط أدوات)
  - يعمل probe الصورة عندما يعلن النموذج دعم إدخال الصور
  - التدفق (على مستوى عالٍ):
    - يولّد الاختبار PNG صغيرًا مع “CAT” + رمز عشوائي (`src/gateway/live-image-probe.ts`)
    - يرسله عبر `agent` مع `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - يحلل Gateway المرفقات إلى `images[]` ‏(`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - يمرر الوكيل المضمّن رسالة مستخدم متعددة الوسائط إلى النموذج
    - التحقق: يحتوي الرد على `cat` + الرمز (سماحية OCR: يسمح بأخطاء طفيفة)

نصيحة: لمعرفة ما يمكنك اختباره على جهازك (ومعرّفات `provider/model` الدقيقة)، شغّل:

```bash
openclaw models list
openclaw models list --json
```

## Live: اختبار smoke للخلفية CLI ‏(Claude CLI أو CLIs محلية أخرى)

- الاختبار: `src/gateway/gateway-cli-backend.live.test.ts`
- الهدف: التحقق من خط أنابيب Gateway + agent باستخدام خلفية CLI محلية، من دون لمس إعداداتك الافتراضية.
- التمكين:
  - `pnpm test:live` ‏(أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- الافتراضيات:
  - النموذج: `claude-cli/claude-sonnet-4-6`
  - الأمر: `claude`
  - الوسائط: `["-p","--output-format","stream-json","--include-partial-messages","--verbose","--permission-mode","bypassPermissions"]`
- التجاوزات (اختيارية):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-opus-4-6"`
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","stream-json","--include-partial-messages","--verbose","--permission-mode","bypassPermissions"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_CLEAR_ENV='["ANTHROPIC_API_KEY","ANTHROPIC_API_KEY_OLD"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` لإرسال مرفق صورة حقيقي (يتم حقن المسارات في prompt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` لتمرير مسارات ملفات الصور كوسائط CLI بدلًا من حقنها في prompt.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` ‏(أو `"list"`) للتحكم في كيفية تمرير وسائط الصور عند ضبط `IMAGE_ARG`.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` لإرسال دورة ثانية والتحقق من تدفق الاستئناف.
- `OPENCLAW_LIVE_CLI_BACKEND_DISABLE_MCP_CONFIG=0` للإبقاء على إعداد MCP في Claude CLI مفعّلًا (يقوم الافتراضي بحقن `--mcp-config` مؤقت وصارم وفارغ حتى تبقى خوادم MCP المحيطة/العالمية معطلة أثناء اختبار smoke).

مثال:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

وصفة Docker:

```bash
pnpm test:docker:live-cli-backend
```

ملاحظات:

- يوجد مشغّل Docker في `scripts/test-live-cli-backend-docker.sh`.
- يشغّل اختبار smoke للخلفية CLI الحي داخل صورة Docker الخاصة بالمستودع كمستخدم `node` غير الجذر، لأن Claude CLI يرفض `bypassPermissions` عندما يتم استدعاؤه كجذر.
- بالنسبة إلى `claude-cli`، يثبّت الحزمة Linux ‏`@anthropic-ai/claude-code` في بادئة قابلة للكتابة ومخزنة مؤقتًا في `OPENCLAW_DOCKER_CLI_TOOLS_DIR` ‏(الافتراضي: `~/.cache/openclaw/docker-cli-tools`).
- بالنسبة إلى `claude-cli`، يحقن اختبار smoke الحي إعداد MCP صارمًا وفارغًا ما لم تضبط `OPENCLAW_LIVE_CLI_BACKEND_DISABLE_MCP_CONFIG=0`.
- ينسخ `~/.claude` إلى الحاوية عند توفره، ولكن على الأجهزة التي تكون فيها مصادقة Claude مدعومة بـ `ANTHROPIC_API_KEY`، فإنه يحتفظ أيضًا بالقيمتين `ANTHROPIC_API_KEY` / `ANTHROPIC_API_KEY_OLD` لـ Claude CLI الفرعي عبر `OPENCLAW_LIVE_CLI_BACKEND_PRESERVE_ENV`.

## Live: اختبار smoke لارتباط ACP ‏(`/acp spawn ... --bind here`)

- الاختبار: `src/gateway/gateway-acp-bind.live.test.ts`
- الهدف: التحقق من تدفق ارتباط المحادثة الحقيقي في ACP مع وكيل ACP حي:
  - إرسال `/acp spawn <agent> --bind here`
  - ربط محادثة قناة رسائل اصطناعية في مكانها
  - إرسال متابعة عادية على المحادثة نفسها
  - التحقق من وصول المتابعة إلى السجل النصي لجلسة ACP المرتبطة
- التمكين:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- الافتراضيات:
  - وكيل ACP: ‏`claude`
  - قناة اصطناعية: سياق محادثة على نمط Slack DM
  - خلفية ACP: ‏`acpx`
- التجاوزات:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=/full/path/to/acpx`
- ملاحظات:
  - يستخدم هذا المسار سطح `chat.send` في gateway مع حقول مسار أصل اصطناعي خاصة بالمشرف فقط بحيث يمكن للاختبارات إرفاق سياق قناة الرسالة من دون التظاهر بالتسليم خارجيًا.
  - عندما لا يتم ضبط `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND`، يستخدم الاختبار أمر acpx المهيأ/المضمّن. وإذا كانت مصادقة الحزمة لديك تعتمد على متغيرات env من `~/.profile`، فافضّل أمر `acpx` مخصصًا يحافظ على env الخاصة بالمزوّد.

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

- يوجد مشغّل Docker في `scripts/test-live-acp-bind-docker.sh`.
- يقوم بتحميل `~/.profile`، وينسخ home المطابقة لمصادقة CLI ‏(`~/.claude` أو `~/.codex`) إلى الحاوية، ويثبت `acpx` في بادئة npm قابلة للكتابة، ثم يثبت CLI الحية المطلوبة (`@anthropic-ai/claude-code` أو `@openai/codex`) إذا كانت مفقودة.
- داخل Docker، يضبط المشغّل `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` حتى يبقي acpx متغيرات env الخاصة بالمزوّد والمتاحة من profile المحمّل متاحة لـ CLI الحزمة الفرعية.

### وصفات live الموصى بها

تكون قوائم السماح الضيقة والصريحة الأسرع والأقل عرضة لعدم الاستقرار:

- نموذج واحد، مباشر (بدون gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- نموذج واحد، اختبار smoke للـ gateway:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- استدعاء الأدوات عبر عدة مزوّدين:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- التركيز على Google ‏(مفتاح Gemini API + Antigravity):
  - Gemini ‏(مفتاح API): ‏`OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity ‏(OAuth): ‏`OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

ملاحظات:

- يستخدم `google/...` واجهة Gemini API ‏(مفتاح API).
- يستخدم `google-antigravity/...` جسر Antigravity OAuth ‏(نقطة نهاية وكيل على نمط Cloud Code Assist).
- يستخدم `google-gemini-cli/...` Gemini CLI المحلي على جهازك (مع مصادقة مستقلة وخصائص أدوات مختلفة).
- Gemini API مقابل Gemini CLI:
  - API: يستدعي OpenClaw واجهة Gemini API المستضافة من Google عبر HTTP ‏(مفتاح API / مصادقة profile)؛ وهذا ما يقصده معظم المستخدمين عندما يقولون “Gemini”.
  - CLI: ينفذ OpenClaw الملف الثنائي المحلي `gemini`؛ وله مصادقة خاصة به وقد يتصرف بشكل مختلف (دعم البث/الأدوات/اختلاف الإصدار).

## Live: مصفوفة النماذج (ما الذي نغطيه)

لا توجد “قائمة نماذج CI” ثابتة (live اختياري)، ولكن هذه هي النماذج **الموصى بها** للتغطية بانتظام على جهاز مطور يملك مفاتيح.

### مجموعة smoke الحديثة (استدعاء الأدوات + الصورة)

هذا هو تشغيل “النماذج الشائعة” الذي نتوقع أن يبقى عاملًا:

- OpenAI ‏(غير Codex): ‏`openai/gpt-5.4` ‏(اختياري: `openai/gpt-5.4-mini`)
- OpenAI Codex: ‏`openai-codex/gpt-5.4`
- Anthropic: ‏`anthropic/claude-opus-4-6` ‏(أو `anthropic/claude-sonnet-4-6`)
- Google ‏(Gemini API): ‏`google/gemini-3.1-pro-preview` و`google/gemini-3-flash-preview` ‏(تجنب نماذج Gemini 2.x الأقدم)
- Google ‏(Antigravity): ‏`google-antigravity/claude-opus-4-6-thinking` و`google-antigravity/gemini-3-flash`
- Z.AI ‏(GLM): ‏`zai/glm-4.7`
- MiniMax: ‏`minimax/MiniMax-M2.7`

شغّل اختبار smoke للـ gateway مع الأدوات + الصورة:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### الأساس: استدعاء الأدوات (Read + Exec اختياري)

اختر واحدًا على الأقل من كل عائلة مزوّدين:

- OpenAI: ‏`openai/gpt-5.4` ‏(أو `openai/gpt-5.4-mini`)
- Anthropic: ‏`anthropic/claude-opus-4-6` ‏(أو `anthropic/claude-sonnet-4-6`)
- Google: ‏`google/gemini-3-flash-preview` ‏(أو `google/gemini-3.1-pro-preview`)
- Z.AI ‏(GLM): ‏`zai/glm-4.7`
- MiniMax: ‏`minimax/MiniMax-M2.7`

تغطية إضافية اختيارية (من الجيد توفرها):

- xAI: ‏`xai/grok-4` ‏(أو أحدث إصدار متاح)
- Mistral: ‏`mistral/`… ‏(اختر نموذجًا واحدًا قادرًا على الأدوات وممكّنًا لديك)
- Cerebras: ‏`cerebras/`… ‏(إذا كان لديك وصول)
- LM Studio: ‏`lmstudio/`… ‏(محلي؛ يعتمد استدعاء الأدوات على وضع API)

### الرؤية: إرسال الصور (مرفق → رسالة متعددة الوسائط)

ضمّن نموذجًا واحدًا على الأقل قادرًا على الصور في `OPENCLAW_LIVE_GATEWAY_MODELS` ‏(متغيرات Claude/Gemini/OpenAI القادرة على الرؤية، إلخ) لاختبار probe الصورة.

### المجمّعات / البوابات البديلة

إذا كانت لديك مفاتيح مفعلة، فنحن ندعم أيضًا الاختبار عبر:

- OpenRouter: ‏`openrouter/...` ‏(مئات النماذج؛ استخدم `openclaw models scan` للعثور على مرشحين يدعمون الأدوات+الصور)
- OpenCode: ‏`opencode/...` لـ Zen و`opencode-go/...` لـ Go ‏(المصادقة عبر `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

مزيد من المزوّدين الذين يمكنك تضمينهم في مصفوفة live ‏(إذا كانت لديك بيانات الاعتماد/الإعدادات):

- مضمنة: `openai`، `openai-codex`، `anthropic`، `google`، `google-vertex`، `google-antigravity`، `google-gemini-cli`، `zai`، `openrouter`، `opencode`، `opencode-go`، `xai`، `groq`، `cerebras`، `mistral`، `github-copilot`
- عبر `models.providers` ‏(نقاط نهاية مخصصة): `minimax` ‏(سحابي/API)، بالإضافة إلى أي proxy متوافق مع OpenAI/Anthropic ‏(مثل LM Studio، وvLLM، وLiteLLM، إلخ)

نصيحة: لا تحاول ترميز “جميع النماذج” بشكل ثابت في الوثائق. فالقائمة المعتمدة هي ما تعيده `discoverModels(...)` على جهازك + ما هي المفاتيح المتاحة.

## بيانات الاعتماد (لا تلتزم بها أبدًا)

تكتشف اختبارات live بيانات الاعتماد بالطريقة نفسها التي يعمل بها CLI. والآثار العملية:

- إذا كان CLI يعمل، فيجب أن تعثر اختبارات live على المفاتيح نفسها.
- إذا قال اختبار live ‏“لا توجد بيانات اعتماد”، فقم بالتصحيح بالطريقة نفسها التي كنت ستصحح بها `openclaw models list` / اختيار النموذج.

- ملفات مصادقة per-agent: ‏`~/.openclaw/agents/<agentId>/agent/auth-profiles.json` ‏(وهذا هو المقصود بـ “profile keys” في اختبارات live)
- الإعدادات: ‏`~/.openclaw/openclaw.json` ‏(أو `OPENCLAW_CONFIG_PATH`)
- دليل الحالة القديم: ‏`~/.openclaw/credentials/` ‏(يُنسخ إلى home staged للاختبارات الحية عند وجوده، لكنه ليس مخزن profile keys الرئيسي)
- تقوم تشغيلات live المحلية افتراضيًا بنسخ الإعدادات النشطة وملفات `auth-profiles.json` لكل وكيل و`credentials/` القديمة وأدلة مصادقة CLI الخارجية المدعومة إلى home اختبار مؤقت؛ ويتم نزع تجاوزات المسارات `agents.*.workspace` / `agentDir` من تلك الإعدادات staged حتى تبقى probes بعيدة عن مساحة العمل الحقيقية على المضيف.

إذا كنت تريد الاعتماد على مفاتيح env ‏(مثل التي تم تصديرها في `~/.profile`)، فشغّل الاختبارات المحلية بعد `source ~/.profile`، أو استخدم مشغلات Docker أدناه (يمكنها تحميل `~/.profile` إلى الحاوية).

## Deepgram live ‏(نسخ الصوت)

- الاختبار: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- التمكين: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## خطة برمجة BytePlus live

- الاختبار: `src/agents/byteplus.live.test.ts`
- التمكين: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- تجاوز النموذج الاختياري: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## توليد الصور live

- الاختبار: `src/image-generation/runtime.live.test.ts`
- الأمر: `pnpm test:live src/image-generation/runtime.live.test.ts`
- النطاق:
  - يعدّد كل plugin مزوّد لتوليد الصور مسجل
  - يحمّل متغيرات env المفقودة الخاصة بالمزوّد من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/البيئية قبل ملفات المصادقة المخزنة افتراضيًا، حتى لا تحجب مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات اعتماد shell الحقيقية
  - يتخطى المزوّدين الذين ليس لديهم مصادقة/ملف/نموذج صالح
  - يشغّل متغيرات توليد الصور القياسية عبر قدرة وقت التشغيل المشتركة:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- المزوّدون المضمنون المغطّون حاليًا:
  - `openai`
  - `google`
- التضييق الاختياري:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- سلوك المصادقة الاختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن profile وتجاهل تجاوزات env فقط

## مشغلات Docker ‏(اختبارات اختيارية من نوع "يعمل على Linux")

تنقسم مشغلات Docker هذه إلى مجموعتين:

- مشغلات live-models: يشغّل كل من `test:docker:live-models` و`test:docker:live-gateway` فقط ملف live المطابق لمفاتيح profile داخل صورة Docker الخاصة بالمستودع (`src/agents/models.profiles.live.test.ts` و`src/gateway/gateway-models.profiles.live.test.ts`)، مع تحميل دليل الإعدادات المحلي ومساحة العمل (ومصدر `~/.profile` إذا تم تحميله). ونقاط الإدخال المحلية المطابقة هي `test:live:models-profiles` و`test:live:gateway-profiles`.
- تستخدم مشغلات Docker الحية افتراضيًا حد smoke أصغر بحيث يظل المسح الكامل في Docker عمليًا:
  يضبط `test:docker:live-models` افتراضيًا `OPENCLAW_LIVE_MAX_MODELS=12`،
  ويضبط `test:docker:live-gateway` افتراضيًا
  `OPENCLAW_LIVE_GATEWAY_SMOKE=1`،
  و`OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`،
  و`OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`، و
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. تجاوز متغيرات البيئة هذه عندما
  تريد عمدًا المسح الأكبر والأشمل.
- يقوم `test:docker:all` ببناء صورة Docker الحية مرة واحدة عبر `test:docker:live-build`، ثم يعيد استخدامها لمساري Docker الحيين.
- مشغلات smoke للحاويات: تشغّل `test:docker:openwebui` و`test:docker:onboard` و`test:docker:gateway-network` و`test:docker:mcp-channels` و`test:docker:plugins` حاوية أو أكثر حقيقية وتتحقق من مسارات integration أعلى مستوى.

تقوم مشغلات Docker الخاصة بـ live-models أيضًا بتحميل homes مصادقة CLI المطلوبة فقط (أو جميع المدعومة عندما لا يكون التشغيل مضيقًا)، ثم تنسخها إلى home الحاوية قبل التشغيل بحيث تتمكن OAuth الخاصة بـ CLI الخارجية من تحديث الرموز من دون تعديل مخزن المصادقة على المضيف:

- النماذج المباشرة: `pnpm test:docker:live-models` ‏(البرنامج النصي: `scripts/test-live-models-docker.sh`)
- اختبار smoke لارتباط ACP: ‏`pnpm test:docker:live-acp-bind` ‏(البرنامج النصي: `scripts/test-live-acp-bind-docker.sh`)
- اختبار smoke للخلفية CLI: ‏`pnpm test:docker:live-cli-backend` ‏(البرنامج النصي: `scripts/test-live-cli-backend-docker.sh`)
- Gateway + وكيل dev: ‏`pnpm test:docker:live-gateway` ‏(البرنامج النصي: `scripts/test-live-gateway-models-docker.sh`)
- اختبار smoke حي لـ Open WebUI: ‏`pnpm test:docker:openwebui` ‏(البرنامج النصي: `scripts/e2e/openwebui-docker.sh`)
- معالج onboarding ‏(TTY، تهيئة كاملة): ‏`pnpm test:docker:onboard` ‏(البرنامج النصي: `scripts/e2e/onboard-docker.sh`)
- شبكات Gateway ‏(حاويتان، مصادقة WS + الصحة): ‏`pnpm test:docker:gateway-network` ‏(البرنامج النصي: `scripts/e2e/gateway-network-docker.sh`)
- جسر قناة MCP ‏(Gateway مُهيّأ + stdio bridge + اختبار smoke خام لإطار إشعارات Claude): ‏`pnpm test:docker:mcp-channels` ‏(البرنامج النصي: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins ‏(اختبار smoke للتثبيت + الاسم البديل `/plugin` + دلالات إعادة التشغيل لحزمة Claude): ‏`pnpm test:docker:plugins` ‏(البرنامج النصي: `scripts/e2e/plugins-docker.sh`)

تقوم مشغلات Docker الخاصة بـ live-models أيضًا بتحميل نسخة checkout الحالية للقراءة فقط
ثم تهيئتها إلى دليل عمل مؤقت داخل الحاوية. وهذا يبقي صورة
وقت التشغيل نحيفة مع الاستمرار في تشغيل Vitest على المصدر/الإعدادات المحلية الدقيقة لديك.
كما تضبط أيضًا `OPENCLAW_SKIP_CHANNELS=1` حتى لا تبدأ probes الحية في gateway
عمّال قنوات Telegram/Discord/إلخ الحقيقيين داخل الحاوية.
ولا يزال `test:docker:live-models` يشغّل `pnpm test:live`، لذا مرّر
`OPENCLAW_LIVE_GATEWAY_*` أيضًا عندما تحتاج إلى تضييق أو استبعاد تغطية gateway
الحية من هذا المسار في Docker.
ويُعد `test:docker:openwebui` اختبار توافق smoke أعلى مستوى: فهو يشغّل
حاوية OpenClaw gateway مع تمكين نقاط نهاية HTTP المتوافقة مع OpenAI،
ثم يشغّل حاوية Open WebUI مثبتة مقابل ذلك gateway، ويسجّل الدخول عبر
Open WebUI، ويتحقق من أن `/api/models` يكشف `openclaw/default`، ثم يرسل
طلب دردشة حقيقيًا عبر وكيل `/api/chat/completions` في Open WebUI.
قد يكون التشغيل الأول أبطأ بشكل ملحوظ لأن Docker قد يحتاج إلى سحب
صورة Open WebUI ولأن Open WebUI قد يحتاج إلى إكمال إعداد البدء البارد الخاص به.
ويتوقع هذا المسار وجود مفتاح نموذج حي صالح، ويُعد `OPENCLAW_PROFILE_FILE`
‏(`~/.profile` افتراضيًا) الطريقة الأساسية لتوفيره في تشغيلات Docker.
وتطبع التشغيلات الناجحة حمولة JSON صغيرة مثل `{ "ok": true, "model":
"openclaw/default", ... }`.
أما `test:docker:mcp-channels` فهو حتمي عمدًا ولا يحتاج إلى
حساب Telegram أو Discord أو iMessage حقيقي. فهو يشغّل حاوية Gateway
مهيأة، ثم يبدأ حاوية ثانية تشغّل `openclaw mcp serve`، ثم
يتحقق من اكتشاف المحادثات الموجّهة، وقراءات السجل النصي، وبيانات تعريف المرفقات،
وسلوك طابور الأحداث الحية، وتوجيه الإرسال الصادر، وإشعارات القنوات + الأذونات
على نمط Claude عبر جسر stdio MCP الحقيقي. ويتحقق فحص الإشعارات
من إطارات stdio MCP الخام مباشرة، بحيث يختبر smoke ما يصدره الجسر
فعليًا، وليس فقط ما قد تُظهره مجموعة أدوات عميل محددة.

اختبار smoke يدوي لسلسلة ACP بلغة طبيعية (ليس في CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- احتفظ بهذا البرنامج النصي لسير عمل التراجع/التصحيح. فقد تكون هناك حاجة إليه مجددًا للتحقق من توجيه سلاسل ACP، لذلك لا تحذفه.

متغيرات env مفيدة:

- `OPENCLAW_CONFIG_DIR=...` ‏(الافتراضي: `~/.openclaw`) ويُحمّل إلى `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` ‏(الافتراضي: `~/.openclaw/workspace`) ويُحمّل إلى `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` ‏(الافتراضي: `~/.profile`) ويُحمّل إلى `/home/node/.profile` ويتم مصدرته قبل تشغيل الاختبارات
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` ‏(الافتراضي: `~/.cache/openclaw/docker-cli-tools`) ويُحمّل إلى `/home/node/.npm-global` لتخزين تثبيتات CLI داخل Docker
- تُحمّل أدلة مصادقة CLI الخارجية تحت `$HOME` للقراءة فقط ضمن `/host-auth/...`، ثم تُنسخ إلى `/home/node/...` قبل بدء الاختبارات
  - الافتراضي: تحميل جميع الأدلة المدعومة (`.codex` و`.claude` و`.minimax`)
  - التشغيلات المضيقة حسب المزوّد تحمل فقط الأدلة المطلوبة المستنتجة من `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - التجاوز اليدوي عبر `OPENCLAW_DOCKER_AUTH_DIRS=all` أو `OPENCLAW_DOCKER_AUTH_DIRS=none` أو قائمة مفصولة بفواصل مثل `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` لتضييق التشغيل
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` لتصفية المزوّدين داخل الحاوية
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لضمان أن تأتي بيانات الاعتماد من مخزن profile ‏(وليس env)
- `OPENCLAW_OPENWEBUI_MODEL=...` لاختيار النموذج الذي يكشفه gateway لاختبار smoke الخاص بـ Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` لتجاوز prompt فحص nonce المستخدمة بواسطة اختبار smoke لـ Open WebUI
- `OPENWEBUI_IMAGE=...` لتجاوز وسم صورة Open WebUI المثبتة

## التحقق من الوثائق

شغّل فحوصات الوثائق بعد تعديلها: `pnpm check:docs`.
وشغّل التحقق الكامل من روابط/مراسي Mintlify عندما تحتاج أيضًا إلى فحص العناوين داخل الصفحة: `pnpm docs:check-links:anchors`.

## التراجع دون اتصال (آمن لـ CI)

هذه حالات تراجع “لخط الأنابيب الحقيقي” من دون مزوّدين حقيقيين:

- استدعاء أدوات Gateway ‏(محاكاة OpenAI، مع gateway حقيقي + حلقة agent): ‏`src/gateway/gateway.test.ts` ‏(الحالة: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- معالج Gateway ‏(‏WS ‏`wizard.start`/`wizard.next`، يكتب الإعدادات + يفرض المصادقة): ‏`src/gateway/gateway.test.ts` ‏(الحالة: "runs wizard over ws and writes auth token config")

## تقييمات موثوقية الوكيل (Skills)

لدينا بالفعل بعض الاختبارات الآمنة لـ CI التي تتصرف مثل “تقييمات موثوقية الوكيل”:

- استدعاء أدوات وهمي عبر الـ gateway الحقيقي + حلقة agent ‏(`src/gateway/gateway.test.ts`).
- تدفقات معالج كاملة من طرف إلى طرف تتحقق من ربط الجلسة وتأثيرات الإعدادات (`src/gateway/gateway.test.ts`).

ما الذي لا يزال مفقودًا بالنسبة إلى Skills ‏(راجع [Skills](/tools/skills)):

- **اتخاذ القرار:** عندما تُدرج Skills في prompt، هل يختار الوكيل Skill الصحيحة (أو يتجنب غير ذات الصلة)؟
- **الامتثال:** هل يقرأ الوكيل `SKILL.md` قبل الاستخدام ويتبع الخطوات/الوسائط المطلوبة؟
- **عقود سير العمل:** سيناريوهات متعددة الأدوار تؤكد ترتيب الأدوات، واستمرار سجل الجلسة، وحدود sandbox.

يجب أن تظل التقييمات المستقبلية حتمية أولًا:

- مشغّل سيناريوهات يستخدم مزوّدين وهميين لتأكيد استدعاءات الأدوات + ترتيبها، وقراءات ملفات Skill، وربط الجلسات.
- مجموعة صغيرة من السيناريوهات المركزة على Skills ‏(استخدم مقابل تجنب، والبوابات، وحقن prompt).
- تقييمات live اختيارية (opt-in، ومحمية بـ env) فقط بعد اكتمال الجناح الآمن لـ CI.

## اختبارات العقود (شكل plugin والقناة)

تتحقق اختبارات العقود من أن كل plugin وقناة مسجلين يطابقان
عقد الواجهة الخاصة بهما. وهي تكرر عبر جميع plugins المكتشفة وتشغّل مجموعة من
تأكيدات الشكل والسلوك. ويتعمد مسار unit الافتراضي `pnpm test`
تخطي هذه الملفات المشتركة الخاصة بالوصلات وملفات smoke؛ شغّل أوامر العقود صراحة
عندما تلمس أسطح القنوات أو المزوّدين المشتركة.

### الأوامر

- جميع العقود: `pnpm test:contracts`
- عقود القنوات فقط: `pnpm test:contracts:channels`
- عقود المزوّدين فقط: `pnpm test:contracts:plugins`

### عقود القنوات

تقع في `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - الشكل الأساسي لـ plugin ‏(id، الاسم، القدرات)
- **setup** - عقد معالج الإعداد
- **session-binding** - سلوك ربط الجلسة
- **outbound-payload** - بنية حمولة الرسالة
- **inbound** - التعامل مع الرسائل الواردة
- **actions** - معالجات إجراءات القناة
- **threading** - التعامل مع معرّف السلسلة
- **directory** - API الدليل/قائمة الأعضاء
- **group-policy** - فرض سياسة المجموعة

### عقود حالة المزوّدين

تقع في `src/plugins/contracts/*.contract.test.ts`.

- **status** - فحوصات حالة القنوات
- **registry** - شكل سجل plugin

### عقود المزوّدين

تقع في `src/plugins/contracts/*.contract.test.ts`:

- **auth** - عقد تدفق المصادقة
- **auth-choice** - اختيار/تحديد المصادقة
- **catalog** - API فهرس النماذج
- **discovery** - اكتشاف plugin
- **loader** - تحميل plugin
- **runtime** - وقت تشغيل المزوّد
- **shape** - شكل/واجهة plugin
- **wizard** - معالج الإعداد

### متى يجب تشغيلها

- بعد تغيير صادرات plugin-sdk أو المسارات الفرعية
- بعد إضافة أو تعديل plugin لقناة أو مزوّد
- بعد إعادة هيكلة تسجيل plugin أو الاكتشاف

تعمل اختبارات العقود في CI ولا تتطلب مفاتيح API حقيقية.

## إضافة حالات تراجع (إرشادات)

عندما تصلح مشكلة في مزوّد/نموذج تم اكتشافها في live:

- أضف حالة تراجع آمنة لـ CI إن أمكن (مزود وهمي/Stub، أو التقاط التحويل الدقيق لشكل الطلب)
- إذا كانت المشكلة بطبيعتها live فقط (حدود المعدل، سياسات المصادقة)، فأبقِ اختبار live ضيقًا ومفعلًا اختياريًا عبر متغيرات env
- افضّل استهداف أصغر طبقة تلتقط الخطأ:
  - خطأ في تحويل/إعادة تشغيل طلب المزوّد → اختبار نماذج مباشر
  - خطأ في خط أنابيب جلسة/سجل/أداة gateway → اختبار smoke حي للـ gateway أو اختبار محاكاة gateway آمن لـ CI
- حاجز حماية لاجتياز SecretRef:
  - يستنتج `src/secrets/exec-secret-ref-id-parity.test.ts` هدفًا نموذجيًا واحدًا لكل فئة SecretRef من بيانات وصفية السجل (`listSecretTargetRegistryEntries()`)، ثم يؤكد رفض معرّفات exec الخاصة بمقاطع الاجتياز.
  - إذا أضفت عائلة هدف SecretRef جديدة من نوع `includeInPlan` في `src/secrets/target-registry-data.ts`، فحدّث `classifyTargetClass` في ذلك الاختبار. ويفشل الاختبار عمدًا مع المعرّفات غير المصنفة حتى لا يمكن تخطي الفئات الجديدة بصمت.
