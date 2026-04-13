---
read_when:
    - تشغيل الاختبارات محليًا أو في CI
    - إضافة اختبارات انحدار لأخطاء النموذج/المزوّد
    - تصحيح سلوك Gateway + الوكيل
summary: 'مجموعة الاختبار: مجموعات unit/e2e/live، ومشغّلات Docker، وما الذي يغطيه كل اختبار'
title: الاختبار
x-i18n:
    generated_at: "2026-04-13T07:28:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3db91b4bc36f626cd014958ec66b08b9cecd9faaa20a5746cd3a49ad4b0b1c38
    source_path: help/testing.md
    workflow: 15
---

# الاختبار

يحتوي OpenClaw على ثلاث مجموعات Vitest (unit/integration و e2e و live) ومجموعة صغيرة من مشغّلات Docker.

هذا المستند هو دليل "كيف نختبر":

- ما الذي تغطيه كل مجموعة (وما الذي لا تغطيه عمدًا)
- الأوامر التي يجب تشغيلها لعمليات العمل الشائعة (محليًا، قبل الدفع، تصحيح الأخطاء)
- كيف تكتشف الاختبارات الحية بيانات الاعتماد وتختار النماذج/المزوّدين
- كيفية إضافة اختبارات انحدار لمشكلات النماذج/المزوّدين الواقعية

## البداية السريعة

في معظم الأيام:

- البوابة الكاملة (متوقعة قبل الدفع): `pnpm build && pnpm check && pnpm test`
- تشغيل أسرع للمجموعة الكاملة محليًا على جهاز ذي موارد كبيرة: `pnpm test:max`
- حلقة مراقبة Vitest مباشرة: `pnpm test:watch`
- الاستهداف المباشر للملفات يمر الآن أيضًا عبر مسارات الامتدادات/القنوات: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- فضّل التشغيلات المستهدفة أولًا عندما تعمل على تكرار إصلاح فشل واحد.
- موقع QA مدعوم بـ Docker: `pnpm qa:lab:up`
- مسار QA مدعوم بآلة Linux افتراضية: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

عندما تلمس الاختبارات أو تريد مزيدًا من الثقة:

- بوابة التغطية: `pnpm test:coverage`
- مجموعة E2E: `pnpm test:e2e`

عند تصحيح مزوّدين/نماذج حقيقية (يتطلب بيانات اعتماد حقيقية):

- المجموعة الحية (فحوصات النماذج + أدوات/صور Gateway): `pnpm test:live`
- استهدف ملفًا حيًا واحدًا بهدوء: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

نصيحة: عندما تحتاج فقط إلى حالة فشل واحدة، ففضّل تضييق الاختبارات الحية عبر متغيرات البيئة الخاصة بقائمة السماح الموصوفة أدناه.

## المشغّلات الخاصة بـ QA

توجد هذه الأوامر بجانب مجموعات الاختبار الرئيسية عندما تحتاج إلى واقعية QA-lab:

- `pnpm openclaw qa suite`
  - يشغّل سيناريوهات QA المدعومة من المستودع مباشرة على المضيف.
  - يشغّل عدة سيناريوهات محددة بالتوازي افتراضيًا مع عمّال Gateway معزولين، حتى 64 عاملًا أو بعدد السيناريوهات المحددة. استخدم `--concurrency <count>` لضبط عدد العمال، أو `--concurrency 1` للمسار التسلسلي الأقدم.
- `pnpm openclaw qa suite --runner multipass`
  - يشغّل مجموعة QA نفسها داخل آلة Multipass Linux افتراضية مؤقتة.
  - يحتفظ بسلوك اختيار السيناريو نفسه كما في `qa suite` على المضيف.
  - يعيد استخدام أعلام اختيار المزوّد/النموذج نفسها الخاصة بـ `qa suite`.
  - تعيد التشغيلات الحية تمرير مدخلات مصادقة QA المدعومة والعملية للضيف:
    مفاتيح المزوّد المعتمدة على env، ومسار إعداد مزوّد QA الحي، و `CODEX_HOME` عند وجوده.
  - يجب أن تبقى مجلدات الإخراج تحت جذر المستودع حتى يتمكن الضيف من الكتابة مرة أخرى عبر مساحة العمل المركّبة.
  - يكتب تقرير + ملخص QA العاديين بالإضافة إلى سجلات Multipass تحت
    `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - يبدأ موقع QA المدعوم بـ Docker لأعمال QA بأسلوب المشغّل.
- `pnpm openclaw qa matrix`
  - يشغّل مسار Matrix QA الحي مقابل خادم Tuwunel منزلي مؤقت ومدعوم بـ Docker.
  - يوفّر ثلاثة مستخدمين مؤقتين لـ Matrix (`driver` و `sut` و `observer`) بالإضافة إلى غرفة خاصة واحدة، ثم يبدأ عنصر Gateway فرعي لـ QA باستخدام Matrix Plugin الحقيقي كوسيلة نقل SUT.
  - يستخدم صورة Tuwunel المستقرة المثبتة `ghcr.io/matrix-construct/tuwunel:v1.5.1` افتراضيًا. تجاوزها باستخدام `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` عندما تحتاج إلى اختبار صورة مختلفة.
  - يدعم Matrix حاليًا فقط `--credential-source env` لأن المسار يوفّر مستخدمين مؤقتين محليًا.
  - يكتب تقرير Matrix QA وملخصًا وقطعة observed-events تحت `.artifacts/qa-e2e/...`.
- `pnpm openclaw qa telegram`
  - يشغّل مسار Telegram QA الحي مقابل مجموعة خاصة حقيقية باستخدام رمزي bot الخاصين بـ driver و SUT من env.
  - يتطلب `OPENCLAW_QA_TELEGRAM_GROUP_ID` و `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` و `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. يجب أن يكون معرّف المجموعة هو معرّف دردشة Telegram رقميًا.
  - يدعم `--credential-source convex` لبيانات الاعتماد المشتركة المجمّعة. استخدم وضع env افتراضيًا، أو اضبط `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` لاختيار عقود الإعارة المجمّعة.
  - يتطلب botين مختلفين في المجموعة الخاصة نفسها، على أن يعرّض bot الخاص بـ SUT اسم مستخدم Telegram.
  - من أجل ملاحظة مستقرة بين bot وآخر، فعّل وضع Bot-to-Bot Communication Mode في `@BotFather` لكلا botين وتأكد من أن bot الخاص بـ driver يمكنه ملاحظة حركة bot داخل المجموعة.
  - يكتب تقرير Telegram QA وملخصًا وقطعة observed-messages تحت `.artifacts/qa-e2e/...`.

تشارك مسارات النقل الحية عقدًا معياريًا واحدًا حتى لا تنحرف وسائل النقل الجديدة:

يبقى `qa-channel` مجموعة QA تركيبية واسعة وليس جزءًا من مصفوفة تغطية النقل الحي.

| المسار | Canary | بوابة الإشارة | حظر قائمة السماح | رد على المستوى الأعلى | استئناف بعد إعادة التشغيل | متابعة الخيط | عزل الخيط | ملاحظة التفاعلات | أمر المساعدة |
| ------ | ------ | ------------- | ---------------- | ---------------------- | ------------------------- | ------------ | ---------- | ---------------- | ------------ |
| Matrix   | x      | x              | x               | x               | x              | x                | x                | x                    |              |
| Telegram | x      |                |                 |                 |                |                  |                  |                      | x            |

### بيانات اعتماد Telegram المشتركة عبر Convex (v1)

عندما يكون `--credential-source convex` (أو `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`) مفعّلًا لـ
`openclaw qa telegram`، يحصل QA lab على إعارة حصرية من مجموعة مدعومة بـ Convex، ويرسل Heartbeat
لتلك الإعارة أثناء تشغيل المسار، ويحرّر الإعارة عند الإيقاف.

هيكل مشروع Convex المرجعي:

- `qa/convex-credential-broker/`

متغيرات env المطلوبة:

- `OPENCLAW_QA_CONVEX_SITE_URL` (على سبيل المثال `https://your-deployment.convex.site`)
- سر واحد للدور المحدد:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` للدور `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` للدور `ci`
- اختيار دور بيانات الاعتماد:
  - CLI: `--credential-role maintainer|ci`
  - القيمة الافتراضية في env: `OPENCLAW_QA_CREDENTIAL_ROLE` (القيمة الافتراضية `maintainer`)

متغيرات env الاختيارية:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (القيمة الافتراضية `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (القيمة الافتراضية `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (القيمة الافتراضية `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (القيمة الافتراضية `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (القيمة الافتراضية `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (معرّف تتبع اختياري)
- يسمح `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` بعناوين URL الخاصة بـ Convex من نوع `http://` على loopback لأغراض التطوير المحلي فقط.

يجب أن يستخدم `OPENCLAW_QA_CONVEX_SITE_URL` البروتوكول `https://` في التشغيل العادي.

تتطلب أوامر الإدارة الخاصة بالمشرفين (إضافة/إزالة/سرد المجموعة)
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` تحديدًا.

مساعدات CLI للمشرفين:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

استخدم `--json` للحصول على خرج قابل للقراءة آليًا في السكربتات وأدوات CI.

عقد نقاط النهاية الافتراضي (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`):

- `POST /acquire`
  - الطلب: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - النجاح: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - الاستنفاد/قابل لإعادة المحاولة: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /heartbeat`
  - الطلب: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - النجاح: `{ status: "ok" }` (أو `2xx` فارغ)
- `POST /release`
  - الطلب: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - النجاح: `{ status: "ok" }` (أو `2xx` فارغ)
- `POST /admin/add` (سر maintainer فقط)
  - الطلب: `{ kind, actorId, payload, note?, status? }`
  - النجاح: `{ status: "ok", credential }`
- `POST /admin/remove` (سر maintainer فقط)
  - الطلب: `{ credentialId, actorId }`
  - النجاح: `{ status: "ok", changed, credential }`
  - حماية الإعارة النشطة: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (سر maintainer فقط)
  - الطلب: `{ kind?, status?, includePayload?, limit? }`
  - النجاح: `{ status: "ok", credentials, count }`

شكل الحمولة لنوع Telegram:

- `{ groupId: string, driverToken: string, sutToken: string }`
- يجب أن يكون `groupId` سلسلة تمثل معرّف دردشة Telegram رقميًا.
- يتحقق `admin/add` من هذا الشكل لـ `kind: "telegram"` ويرفض الحمولة غير الصحيحة.

### إضافة قناة إلى QA

تتطلب إضافة قناة إلى نظام QA المكتوب بـ markdown شيئين فقط بالضبط:

1. محول نقل للقناة.
2. حزمة سيناريوهات تمارس عقد القناة.

لا تضف مشغّل QA خاصًا بالقناة عندما يمكن لمشغّل `qa-lab` المشترك
أن يمتلك التدفق.

يمتلك `qa-lab` الآليات المشتركة:

- بدء المجموعة وإنهاؤها
- تزامن العمال
- كتابة القطع
- توليد التقارير
- تنفيذ السيناريوهات
- الأسماء المستعارة التوافقية لسيناريوهات `qa-channel` الأقدم

يمتلك محول القناة عقد النقل:

- كيفية ضبط Gateway لذلك النقل
- كيفية التحقق من الجاهزية
- كيفية حقن الأحداث الواردة
- كيفية ملاحظة الرسائل الصادرة
- كيفية كشف النصوص المنسوخة وحالة النقل المطبّعة
- كيفية تنفيذ الإجراءات المدعومة بالنقل
- كيفية التعامل مع إعادة الضبط أو التنظيف الخاص بالنقل

الحد الأدنى لاعتماد قناة جديدة هو:

1. تنفيذ محول النقل على واجهة `qa-lab` المشتركة.
2. تسجيل المحول في سجل النقل.
3. إبقاء الآليات الخاصة بالنقل داخل المحول أو حزام القناة.
4. تأليف أو تكييف سيناريوهات markdown تحت `qa/scenarios/`.
5. استخدام مساعدات السيناريو العامة للسيناريوهات الجديدة.
6. الإبقاء على الأسماء المستعارة التوافقية الحالية عاملة ما لم يكن المستودع يقوم بترحيل مقصود.

قاعدة القرار صارمة:

- إذا أمكن التعبير عن السلوك مرة واحدة في `qa-lab`، فضعه في `qa-lab`.
- إذا كان السلوك يعتمد على نقل قناة واحدة، فأبقِه في ذلك المحول أو حزام Plugin.
- إذا احتاج السيناريو إلى قدرة جديدة يمكن لأكثر من قناة استخدامها، فأضف مساعدًا عامًا بدلًا من فرع خاص بقناة في `suite.ts`.
- إذا كان السلوك ذا معنى فقط لنقل واحد، فأبقِ السيناريو خاصًا بذلك النقل واجعل ذلك واضحًا في عقد السيناريو.

الأسماء العامة المفضلة للمساعدات في السيناريوهات الجديدة هي:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

تبقى الأسماء المستعارة التوافقية متاحة للسيناريوهات الحالية، بما في ذلك:

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

يجب أن تستخدم الأعمال الجديدة الخاصة بالقنوات أسماء المساعدات العامة.
توجد الأسماء المستعارة التوافقية لتجنب ترحيل شامل دفعة واحدة، وليست نموذجًا
لتأليف السيناريوهات الجديدة.

## مجموعات الاختبار (ما الذي يعمل وأين)

فكّر في المجموعات على أنها "زيادة في الواقعية" (وزيادة في عدم الاستقرار/التكلفة):

### Unit / integration (الافتراضي)

- الأمر: `pnpm test`
- الإعداد: عشر تشغيلات shards متسلسلة (`vitest.full-*.config.ts`) عبر مشاريع Vitest المحددة الموجودة
- الملفات: مخزونات core/unit تحت `src/**/*.test.ts` و `packages/**/*.test.ts` و `test/**/*.test.ts` واختبارات `ui` الخاصة بـ node والمسموح بها والمغطاة بواسطة `vitest.unit.config.ts`
- النطاق:
  - اختبارات unit خالصة
  - اختبارات integration داخل العملية (مصادقة gateway، والتوجيه، والأدوات، والتحليل، والإعداد)
  - اختبارات انحدار حتمية للأخطاء المعروفة
- التوقعات:
  - تعمل في CI
  - لا تتطلب مفاتيح حقيقية
  - يجب أن تكون سريعة ومستقرة
- ملاحظة المشاريع:
  - التشغيل غير المستهدف لـ `pnpm test` يشغّل الآن أحد عشر إعداد shard أصغر (`core-unit-src` و `core-unit-security` و `core-unit-ui` و `core-unit-support` و `core-support-boundary` و `core-contracts` و `core-bundled` و `core-runtime` و `agentic` و `auto-reply` و `extensions`) بدلًا من عملية root-project أصلية واحدة ضخمة. هذا يقلل ذروة RSS على الأجهزة المحمّلة ويتجنب أن تؤدي أعمال auto-reply/extension إلى تجويع المجموعات غير المرتبطة.
  - لا يزال `pnpm test --watch` يستخدم رسم المشاريع الأصلي لـ root `vitest.config.ts`، لأن حلقة watch متعددة الـ shards غير عملية.
  - `pnpm test` و `pnpm test:watch` و `pnpm test:perf:imports` تمرّر أهداف الملفات/المجلدات الصريحة عبر المسارات المحددة أولًا، لذلك فإن `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` يتجنب تكلفة بدء التشغيل الكاملة لـ root project.
  - يقوم `pnpm test:changed` بتوسيع مسارات git المتغيرة إلى المسارات المحددة نفسها عندما يقتصر الفرق على ملفات source/test القابلة للتوجيه؛ أما تعديلات config/setup فتعود إلى إعادة تشغيل root-project الواسعة.
  - اختبارات unit الخفيفة من حيث الاستيراد من agents و commands و plugins ومساعدات auto-reply و `plugin-sdk` والمناطق المشابهة من الأدوات الخالصة تمر عبر مسار `unit-fast`، الذي يتجاوز `test/setup-openclaw-runtime.ts`؛ بينما تبقى الملفات ذات الحالة/الكثيفة وقت التشغيل على المسارات الحالية.
  - تُربط أيضًا ملفات source المساعدة المحددة في `plugin-sdk` و `commands` في تشغيل changed-mode باختبارات sibling صريحة في تلك المسارات الخفيفة، بحيث تتجنب تعديلات المساعدات إعادة تشغيل المجموعة الثقيلة الكاملة لذلك الدليل.
  - يملك `auto-reply` الآن ثلاث مجموعات مخصصة: مساعدات core العلوية، واختبارات integration العلوية `reply.*`، والشجرة الفرعية `src/auto-reply/reply/**`. هذا يُبقي أثقل أعمال harness الخاصة بالردود بعيدًا عن اختبارات status/chunk/token الرخيصة.
- ملاحظة المشغّل المضمّن:
  - عندما تغيّر مدخلات اكتشاف أدوات الرسائل أو سياق وقت التشغيل الخاص بـ Compaction،
    حافظ على مستويي التغطية كليهما.
  - أضف اختبارات انحدار مركزة للمساعدات الخاصة بحدود التوجيه/التطبيع الخالصة.
  - وحافظ أيضًا على سلامة مجموعات integration الخاصة بالمشغّل المضمّن:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`،
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`، و
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - تتحقق هذه المجموعات من أن المعرّفات المحددة وسلوك Compaction لا يزالان يتدفقان
    عبر المسارات الحقيقية `run.ts` / `compact.ts`؛ ولا تُعد اختبارات المساعدات وحدها
    بديلًا كافيًا عن مسارات integration هذه.
- ملاحظة الـ pool:
  - أصبح إعداد Vitest الأساسي يستخدم `threads` افتراضيًا.
  - كما يثبّت إعداد Vitest المشترك `isolate: false` ويستخدم المشغّل غير المعزول عبر إعدادات root projects و e2e و live.
  - يحتفظ مسار root UI بإعداد `jsdom` والمُحسِّن الخاص به، لكنه يعمل الآن أيضًا على المشغّل المشترك غير المعزول.
  - يرث كل shard في `pnpm test` القيم الافتراضية نفسها `threads` + `isolate: false` من إعداد Vitest المشترك.
  - يضيف مشغّل `scripts/run-vitest.mjs` المشترك الآن أيضًا `--no-maglev` افتراضيًا لعمليات Node الفرعية الخاصة بـ Vitest لتقليل تذبذب الترجمة في V8 أثناء التشغيلات المحلية الكبيرة. اضبط `OPENCLAW_VITEST_ENABLE_MAGLEV=1` إذا كنت بحاجة إلى المقارنة مع سلوك V8 القياسي.
- ملاحظة التكرار المحلي السريع:
  - يمر `pnpm test:changed` عبر المسارات المحددة عندما تتطابق المسارات المتغيرة بوضوح مع مجموعة أصغر.
  - يحتفظ `pnpm test:max` و `pnpm test:changed:max` بسلوك التوجيه نفسه، فقط مع حد أعلى أكبر للعمال.
  - أصبح التحجيم التلقائي المحلي للعمال محافظًا عمدًا الآن، كما أنه يتراجع عندما يكون متوسط حمل المضيف مرتفعًا أصلًا، بحيث تتسبب تشغيلات Vitest المتزامنة المتعددة بضرر أقل افتراضيًا.
  - يضع إعداد Vitest الأساسي ملفات المشاريع/الإعداد كـ `forceRerunTriggers` حتى تبقى إعادة التشغيل في changed-mode صحيحة عند تغيّر توصيلات الاختبار.
  - يحتفظ الإعداد بتمكين `OPENCLAW_VITEST_FS_MODULE_CACHE` على المضيفين المدعومين؛ اضبط `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` إذا كنت تريد موقع cache صريحًا واحدًا لأغراض profiling المباشر.
- ملاحظة تصحيح الأداء:
  - يفعّل `pnpm test:perf:imports` تقارير مدة الاستيراد في Vitest بالإضافة إلى خرج تفصيلي للاستيراد.
  - يقيّد `pnpm test:perf:imports:changed` عرض profiling نفسه على الملفات المتغيرة منذ `origin/main`.
- يقارن `pnpm test:perf:changed:bench -- --ref <git-ref>` بين `test:changed` الموجّه ومسار root-project الأصلي لذلك الفرق الملتزم، ويطبع زمن التنفيذ بالإضافة إلى macOS max RSS.
- يقيّم `pnpm test:perf:changed:bench -- --worktree` الشجرة المتسخة الحالية بتمرير قائمة الملفات المتغيرة عبر `scripts/test-projects.mjs` وإعداد root Vitest.
  - يكتب `pnpm test:perf:profile:main` ملف تعريف CPU للخيط الرئيسي لزمن بدء Vitest/Vite وكلفة التحويل.
  - يكتب `pnpm test:perf:profile:runner` ملفات تعريف CPU+heap الخاصة بالمشغّل لمجموعة unit مع تعطيل التوازي على مستوى الملفات.

### E2E (اختبار smoke لـ gateway)

- الأمر: `pnpm test:e2e`
- الإعداد: `vitest.e2e.config.ts`
- الملفات: `src/**/*.e2e.test.ts` و `test/**/*.e2e.test.ts`
- القيم الافتراضية لوقت التشغيل:
  - يستخدم `threads` في Vitest مع `isolate: false`، بما يتوافق مع بقية المستودع.
  - يستخدم عمالًا متكيفين (CI: حتى 2، محليًا: 1 افتراضيًا).
  - يعمل في الوضع الصامت افتراضيًا لتقليل كلفة I/O في الطرفية.
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_WORKERS=<n>` لفرض عدد العمال (بحد أقصى 16).
  - `OPENCLAW_E2E_VERBOSE=1` لإعادة تفعيل خرج الطرفية التفصيلي.
- النطاق:
  - سلوك gateway من طرف إلى طرف متعدد النسخ
  - أسطح WebSocket/HTTP، واقتران Node، والشبكات الأثقل
- التوقعات:
  - تعمل في CI (عندما تكون مفعّلة في خط التنفيذ)
  - لا تتطلب مفاتيح حقيقية
  - تحتوي على أجزاء متحركة أكثر من اختبارات unit (وقد تكون أبطأ)

### E2E: اختبار smoke للواجهة الخلفية OpenShell

- الأمر: `pnpm test:e2e:openshell`
- الملف: `test/openshell-sandbox.e2e.test.ts`
- النطاق:
  - يبدأ OpenShell gateway معزولًا على المضيف عبر Docker
  - ينشئ sandbox من Dockerfile محلي مؤقت
  - يختبر الواجهة الخلفية OpenClaw الخاصة بـ OpenShell عبر `sandbox ssh-config` + تنفيذ SSH الحقيقي
  - يتحقق من سلوك نظام الملفات canonical البعيد عبر جسر sandbox fs
- التوقعات:
  - اختيارية فقط؛ وليست جزءًا من التشغيل الافتراضي `pnpm test:e2e`
  - تتطلب CLI محليًا لـ `openshell` بالإضافة إلى Docker daemon عامل
  - تستخدم `HOME` / `XDG_CONFIG_HOME` معزولين، ثم تدمّر gateway و sandbox الخاصين بالاختبار
- تجاوزات مفيدة:
  - `OPENCLAW_E2E_OPENSHELL=1` لتمكين الاختبار عند تشغيل مجموعة e2e الأوسع يدويًا
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` للإشارة إلى ملف CLI ثنائي أو wrapper script غير افتراضي

### Live (مزوّدون حقيقيون + نماذج حقيقية)

- الأمر: `pnpm test:live`
- الإعداد: `vitest.live.config.ts`
- الملفات: `src/**/*.live.test.ts`
- الافتراضي: **مفعّل** بواسطة `pnpm test:live` (يضبط `OPENCLAW_LIVE_TEST=1`)
- النطاق:
  - "هل يعمل هذا المزوّد/النموذج فعلًا _اليوم_ باستخدام بيانات اعتماد حقيقية؟"
  - اكتشاف تغييرات تنسيق المزوّد، ومراوغات استدعاء الأدوات، ومشكلات المصادقة، وسلوك تحديد المعدل
- التوقعات:
  - غير مستقرة بالنسبة إلى CI بطبيعتها (شبكات حقيقية، وسياسات مزوّد حقيقية، وحصص، وانقطاعات)
  - تكلف مالًا / تستخدم حدود المعدل
  - يُفضّل تشغيل مجموعات فرعية مضيقة بدلًا من "كل شيء"
- تعتمد التشغيلات الحية على `~/.profile` لالتقاط مفاتيح API الناقصة.
- افتراضيًا، لا تزال التشغيلات الحية تعزل `HOME` وتنسخ مواد config/auth إلى دليل home مؤقت خاص بالاختبار حتى لا تتمكن fixtures الخاصة بـ unit من تعديل `~/.openclaw` الحقيقي لديك.
- اضبط `OPENCLAW_LIVE_USE_REAL_HOME=1` فقط عندما تحتاج عمدًا إلى أن تستخدم الاختبارات الحية دليل home الحقيقي لديك.
- يستخدم `pnpm test:live` الآن وضعًا أكثر هدوءًا افتراضيًا: فهو يحتفظ بخرج التقدّم `[live] ...`، لكنه يخفي إشعار `~/.profile` الإضافي ويكتم سجلات bootstrap الخاصة بـ gateway وضجيج Bonjour. اضبط `OPENCLAW_LIVE_TEST_QUIET=0` إذا كنت تريد استعادة سجلات بدء التشغيل الكاملة.
- تدوير مفاتيح API (خاص بالمزوّد): اضبط `*_API_KEYS` بتنسيق comma/semicolon أو `*_API_KEY_1` و `*_API_KEY_2` (على سبيل المثال `OPENAI_API_KEYS` و `ANTHROPIC_API_KEYS` و `GEMINI_API_KEYS`) أو تجاوزًا لكل اختبار حي عبر `OPENCLAW_LIVE_*_KEY`؛ تعيد الاختبارات المحاولة عند استجابات تحديد المعدل.
- خرج التقدّم/Heartbeat:
  - تصدر المجموعات الحية الآن أسطر التقدّم إلى stderr حتى تظهر مكالمات المزوّد الطويلة على أنها نشطة بوضوح حتى عندما يكون التقاط الطرفية في Vitest هادئًا.
  - يعطّل `vitest.live.config.ts` اعتراض الطرفية في Vitest بحيث تُبث أسطر التقدّم الخاصة بالمزوّد/ gateway مباشرة أثناء التشغيلات الحية.
  - اضبط Heartbeat الخاص بالنموذج المباشر باستخدام `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - اضبط Heartbeat الخاص بـ gateway/probe باستخدام `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## أي مجموعة يجب أن أشغّل؟

استخدم جدول القرار هذا:

- تعديل المنطق/الاختبارات: شغّل `pnpm test` (و `pnpm test:coverage` إذا غيّرت الكثير)
- لمس شبكات gateway / بروتوكول WS / الاقتران: أضف `pnpm test:e2e`
- تصحيح "البوت الخاص بي متوقف" / الأعطال الخاصة بالمزوّد / استدعاء الأدوات: شغّل `pnpm test:live` مضيقًا

## Live: مسح قدرات Android Node

- الاختبار: `src/gateway/android-node.capabilities.live.test.ts`
- السكربت: `pnpm android:test:integration`
- الهدف: استدعاء **كل أمر يتم الإعلان عنه حاليًا** بواسطة Android Node متصل والتحقق من سلوك عقد الأمر.
- النطاق:
  - إعداد يدوي/مسبق الشروط (المجموعة لا تثبّت التطبيق ولا تشغّله ولا تقترنه).
  - تحقق `node.invoke` على مستوى gateway أمرًا بأمر من أجل Android Node المحدد.
- الإعداد المسبق المطلوب:
  - أن يكون تطبيق Android متصلًا ومقترنًا بالفعل بـ gateway.
  - إبقاء التطبيق في الواجهة الأمامية.
  - منح الأذونات/الموافقة على الالتقاط للقدرات التي تتوقع نجاحها.
- تجاوزات الهدف الاختيارية:
  - `OPENCLAW_ANDROID_NODE_ID` أو `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- تفاصيل إعداد Android الكاملة: [تطبيق Android](/ar/platforms/android)

## Live: اختبار smoke للنموذج (مفاتيح profile)

تنقسم الاختبارات الحية إلى طبقتين حتى نتمكن من عزل الإخفاقات:

- يخبرنا "النموذج المباشر" ما إذا كان المزوّد/النموذج قادرًا على الرد أساسًا بالمفتاح المعطى.
- يخبرنا "اختبار smoke لـ Gateway" ما إذا كان خط gateway+agent الكامل يعمل لهذا النموذج (sessions، والسجل، والأدوات، وسياسة sandbox، إلخ).

### الطبقة 1: إكمال مباشر للنموذج (من دون gateway)

- الاختبار: `src/agents/models.profiles.live.test.ts`
- الهدف:
  - تعداد النماذج المكتشفة
  - استخدام `getApiKeyForModel` لاختيار النماذج التي لديك بيانات اعتماد لها
  - تشغيل إكمال صغير لكل نموذج (واختبارات انحدار مستهدفة عند الحاجة)
- كيفية التمكين:
  - `pnpm test:live` (أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- اضبط `OPENCLAW_LIVE_MODELS=modern` (أو `all`، وهو اسم مستعار لـ modern) لتشغيل هذه المجموعة فعلًا؛ وإلا فسيتم تخطيها للحفاظ على تركيز `pnpm test:live` على اختبار smoke لـ gateway
- كيفية اختيار النماذج:
  - `OPENCLAW_LIVE_MODELS=modern` لتشغيل قائمة السماح الحديثة (Opus/Sonnet 4.6+ و GPT-5.x + Codex و Gemini 3 و GLM 4.7 و MiniMax M2.7 و Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` هو اسم مستعار لقائمة السماح الحديثة
  - أو `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (قائمة سماح مفصولة بفواصل)
  - تستخدم عمليات المسح modern/all حدًا منسقًا عالي الإشارة افتراضيًا؛ اضبط `OPENCLAW_LIVE_MAX_MODELS=0` لمسح حديث شامل أو رقمًا موجبًا لحد أصغر.
- كيفية اختيار المزوّدين:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (قائمة سماح مفصولة بفواصل)
- من أين تأتي المفاتيح:
  - افتراضيًا: من مخزن profile والبدائل الاحتياطية في env
  - اضبط `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض **مخزن profile** فقط
- لماذا يوجد هذا:
  - يفصل بين "واجهة API الخاصة بالمزوّد معطلة / المفتاح غير صالح" و "خط gateway agent معطل"
  - يحتوي على اختبارات انحدار صغيرة ومعزولة (مثال: OpenAI Responses/Codex Responses reasoning replay + تدفقات tool-call)

### الطبقة 2: اختبار smoke لـ Gateway + dev agent (ما الذي يفعله "@openclaw" فعليًا)

- الاختبار: `src/gateway/gateway-models.profiles.live.test.ts`
- الهدف:
  - تشغيل gateway داخل العملية
  - إنشاء/ترقيع جلسة `agent:dev:*` (مع تجاوز النموذج في كل تشغيل)
  - التكرار عبر النماذج التي تملك مفاتيح والتحقق من:
    - استجابة "ذات معنى" (من دون أدوات)
    - أن استدعاء أداة حقيقي يعمل (`read` probe)
    - probes إضافية اختيارية للأدوات (`exec+read` probe)
    - استمرار عمل مسارات انحدار OpenAI (`tool-call-only → follow-up`)
- تفاصيل probes (حتى تتمكن من شرح الإخفاقات بسرعة):
  - probe `read`: يكتب الاختبار ملف nonce في مساحة العمل ويطلب من الوكيل `read` قراءته ثم إعادة nonce.
  - probe `exec+read`: يطلب الاختبار من الوكيل كتابة nonce في ملف مؤقت عبر `exec`، ثم قراءته مجددًا عبر `read`.
  - probe الصورة: يرفق الاختبار ملف PNG مُولّدًا (قطة + رمز عشوائي) ويتوقع من النموذج إرجاع `cat <CODE>`.
  - مرجع التنفيذ: `src/gateway/gateway-models.profiles.live.test.ts` و `src/gateway/live-image-probe.ts`.
- كيفية التمكين:
  - `pnpm test:live` (أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
- كيفية اختيار النماذج:
  - الافتراضي: قائمة السماح الحديثة (Opus/Sonnet 4.6+ و GPT-5.x + Codex و Gemini 3 و GLM 4.7 و MiniMax M2.7 و Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` هو اسم مستعار لقائمة السماح الحديثة
  - أو اضبط `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (أو قائمة مفصولة بفواصل) للتضييق
  - تستخدم عمليات المسح gateway الحديثة/`all` حدًا منسقًا عالي الإشارة افتراضيًا؛ اضبط `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` لمسح حديث شامل أو رقمًا موجبًا لحد أصغر.
- كيفية اختيار المزوّدين (تجنّب "كل شيء عبر OpenRouter"):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (قائمة سماح مفصولة بفواصل)
- تكون probes الأدوات + الصور مفعّلة دائمًا في هذا الاختبار الحي:
  - probe `read` + probe `exec+read` (ضغط على الأدوات)
  - يعمل probe الصورة عندما يعلن النموذج دعم إدخال الصور
  - التدفق (على مستوى عالٍ):
    - يولّد الاختبار ملف PNG صغيرًا يحتوي على “CAT” + رمز عشوائي (`src/gateway/live-image-probe.ts`)
    - يرسله عبر `agent` مع `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - يحلل Gateway المرفقات إلى `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - يمرّر الوكيل المضمّن رسالة مستخدم متعددة الوسائط إلى النموذج
    - التحقق: يحتوي الرد على `cat` + الرمز (مع سماحية OCR: الأخطاء الطفيفة مسموح بها)

نصيحة: لمعرفة ما يمكنك اختباره على جهازك (ومعرّفات `provider/model` الدقيقة)، شغّل:

```bash
openclaw models list
openclaw models list --json
```

## Live: اختبار smoke للواجهة الخلفية CLI (Claude أو Codex أو Gemini أو CLI محلية أخرى)

- الاختبار: `src/gateway/gateway-cli-backend.live.test.ts`
- الهدف: التحقق من خط Gateway + الوكيل باستخدام واجهة خلفية CLI محلية، من دون لمس إعدادك الافتراضي.
- توجد القيم الافتراضية لاختبار smoke الخاصة بكل واجهة خلفية مع تعريف `cli-backend.ts` المملوك للامتداد المعني.
- التمكين:
  - `pnpm test:live` (أو `OPENCLAW_LIVE_TEST=1` إذا كنت تستدعي Vitest مباشرة)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- القيم الافتراضية:
  - المزوّد/النموذج الافتراضي: `claude-cli/claude-sonnet-4-6`
  - يأتي سلوك الأمر/الوسائط/الصورة من بيانات metadata الخاصة بـ CLI backend Plugin المالكة.
- التجاوزات (اختيارية):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` لإرسال مرفق صورة حقيقي (تُحقن المسارات في prompt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` لتمرير مسارات ملفات الصور كوسائط CLI بدلًا من حقنها في prompt.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (أو `"list"`) للتحكم في كيفية تمرير وسائط الصور عند ضبط `IMAGE_ARG`.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` لإرسال دور ثانٍ والتحقق من تدفق الاستئناف.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` لتعطيل probe الاستمرارية الافتراضي داخل الجلسة نفسها Claude Sonnet -> Opus (اضبطه على `1` لفرض تشغيله عندما يدعم النموذج المحدد هدف تبديل).

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
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

ملاحظات:

- يوجد مشغّل Docker في `scripts/test-live-cli-backend-docker.sh`.
- يشغّل اختبار smoke الحي لواجهة CLI الخلفية داخل صورة Docker الخاصة بالمستودع كمستخدم `node` غير الجذر.
- يحل metadata الخاصة باختبار smoke للواجهة CLI من الامتداد المالك، ثم يثبت حزمة CLI المطابقة الخاصة بـ Linux (`@anthropic-ai/claude-code` أو `@openai/codex` أو `@google/gemini-cli`) في بادئة قابلة للكتابة ومخزنة مؤقتًا عند `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (الافتراضي: `~/.cache/openclaw/docker-cli-tools`).
- يتطلب `pnpm test:docker:live-cli-backend:claude-subscription` مصادقة OAuth محمولة لاشتراك Claude Code عبر أحد الخيارين: `~/.claude/.credentials.json` مع `claudeAiOauth.subscriptionType` أو `CLAUDE_CODE_OAUTH_TOKEN` من `claude setup-token`. يثبت أولًا نجاح `claude -p` مباشرة داخل Docker، ثم يشغّل دورين لواجهة Gateway CLI الخلفية من دون الاحتفاظ بمتغيرات env الخاصة بمفتاح Anthropic API. يعطّل هذا المسار الخاص بالاشتراك probe أداة/صورة Claude MCP افتراضيًا لأن Claude يوجّه حاليًا استخدام التطبيقات الخارجية عبر فوترة استخدام إضافي بدلًا من حدود خطة الاشتراك العادية.
- يمارس اختبار smoke الحي لواجهة CLI الخلفية الآن التدفق الكامل نفسه من طرف إلى طرف لكل من Claude و Codex و Gemini: دور نصي، ثم دور تصنيف صورة، ثم استدعاء أداة MCP `cron` يتم التحقق منه عبر Gateway CLI.
- كما يقوم اختبار smoke الافتراضي لـ Claude بترقيع الجلسة من Sonnet إلى Opus ويتحقق من أن الجلسة المستأنفة لا تزال تتذكر ملاحظة سابقة.

## Live: اختبار smoke لربط ACP (`/acp spawn ... --bind here`)

- الاختبار: `src/gateway/gateway-acp-bind.live.test.ts`
- الهدف: التحقق من تدفق Conversation-bind الحقيقي لـ ACP باستخدام وكيل ACP حي:
  - إرسال `/acp spawn <agent> --bind here`
  - ربط محادثة channel للرسائل تركيبية في مكانها
  - إرسال متابعة عادية على نفس المحادثة
  - التحقق من أن المتابعة تصل إلى transcript الجلسة المرتبطة بـ ACP
- التمكين:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- القيم الافتراضية:
  - وكلاء ACP في Docker: `claude,codex,gemini`
  - وكيل ACP للتشغيل المباشر `pnpm test:live ...`: `claude`
  - القناة التركيبية: سياق محادثة على نمط Slack DM
  - الواجهة الخلفية لـ ACP: `acpx`
- التجاوزات:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- ملاحظات:
  - يستخدم هذا المسار سطح gateway `chat.send` مع حقول originating-route تركيبية مخصصة للمشرف فقط حتى تتمكن الاختبارات من إرفاق سياق قناة الرسائل من دون الادعاء بالتسليم الخارجي.
  - عندما لا يكون `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` مضبوطًا، يستخدم الاختبار سجل الوكلاء المضمّن الخاص بـ Plugin `acpx` للوكيل المحدد في harness الخاص بـ ACP.

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
- افتراضيًا، يشغّل اختبار smoke لربط ACP مقابل جميع وكلاء CLI الحيين المدعومين بالتسلسل: `claude` ثم `codex` ثم `gemini`.
- استخدم `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude` أو `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` أو `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` لتضييق المصفوفة.
- يعتمد على `~/.profile`، ويجهّز مادة مصادقة CLI المطابقة داخل الحاوية، ويثبت `acpx` في بادئة npm قابلة للكتابة، ثم يثبت CLI الحي المطلوب (`@anthropic-ai/claude-code` أو `@openai/codex` أو `@google/gemini-cli`) إذا لم يكن موجودًا.
- داخل Docker، يضبط المشغّل `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` حتى يحافظ acpx على متغيرات env الخاصة بالمزوّد من profile الذي جرى تحميله لتكون متاحة لـ harness CLI الفرعي.

## Live: اختبار smoke لـ Codex app-server harness

- الهدف: التحقق من Codex harness المملوك لـ Plugin عبر طريقة gateway
  العادية `agent`:
  - تحميل Plugin `codex` المضمّن
  - اختيار `OPENCLAW_AGENT_RUNTIME=codex`
  - إرسال أول دور للوكيل عبر gateway إلى `codex/gpt-5.4`
  - إرسال دور ثانٍ إلى جلسة OpenClaw نفسها والتحقق من أن خيط app-server
    يمكنه الاستئناف
  - تشغيل `/codex status` و `/codex models` عبر مسار أوامر gateway نفسه
- الاختبار: `src/gateway/gateway-codex-harness.live.test.ts`
- التمكين: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- النموذج الافتراضي: `codex/gpt-5.4`
- probe الصورة الاختياري: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- probe MCP/tool الاختياري: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- يضبط اختبار smoke القيمة `OPENCLAW_AGENT_HARNESS_FALLBACK=none` حتى لا يتمكن
  Codex harness المعطّل من النجاح عبر الرجوع الاحتياطي الصامت إلى PI.
- المصادقة: `OPENAI_API_KEY` من shell/profile، بالإضافة إلى نسخ اختيارية من
  `~/.codex/auth.json` و `~/.codex/config.toml`

وصفة محلية:

```bash
source ~/.profile
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=codex/gpt-5.4 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

وصفة Docker:

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

ملاحظات Docker:

- يوجد مشغّل Docker في `scripts/test-live-codex-harness-docker.sh`.
- يعتمد على `~/.profile` المركّب، ويمرّر `OPENAI_API_KEY`، وينسخ ملفات
  مصادقة Codex CLI عند وجودها، ويثبت `@openai/codex` في بادئة npm
  قابلة للكتابة ومركّبة، ويجهّز شجرة المصدر، ثم يشغّل فقط اختبار Codex-harness الحي.
- يفعّل Docker probes الصورة و MCP/tool افتراضيًا. اضبط
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` أو
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` عندما تحتاج إلى تشغيل تضييقي أكثر لأغراض التصحيح.
- يصدّر Docker أيضًا `OPENCLAW_AGENT_HARNESS_FALLBACK=none`، بما يتطابق مع
  إعداد الاختبار الحي حتى لا يتمكن `openai-codex/*` أو الرجوع الاحتياطي إلى PI من إخفاء
  انحدار في Codex harness.

### وصفات حية موصى بها

قوائم السماح الضيقة والصريحة هي الأسرع والأقل عرضة للتذبذب:

- نموذج واحد، مباشر (من دون gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- نموذج واحد، اختبار smoke لـ gateway:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- استدعاء الأدوات عبر عدة مزوّدين:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- التركيز على Google (مفتاح Gemini API + Antigravity):
  - Gemini (مفتاح API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

ملاحظات:

- يستخدم `google/...` واجهة Gemini API (مفتاح API).
- يستخدم `google-antigravity/...` جسر Antigravity OAuth (نقطة نهاية وكيل على نمط Cloud Code Assist).
- يستخدم `google-gemini-cli/...` Gemini CLI المحلي على جهازك (مصادقة منفصلة ومراوغات أدوات منفصلة).
- Gemini API مقابل Gemini CLI:
  - API: يستدعي OpenClaw واجهة Gemini API المستضافة من Google عبر HTTP (مفتاح API / مصادقة profile)؛ وهذا هو ما يقصده معظم المستخدمين عندما يقولون "Gemini".
  - CLI: يستدعي OpenClaw ملف `gemini` الثنائي المحلي؛ وله مصادقته الخاصة ويمكن أن يتصرف بشكل مختلف (البث، ودعم الأدوات، واختلاف الإصدارات).

## Live: مصفوفة النماذج (ما الذي نغطيه)

لا توجد "قائمة نماذج CI" ثابتة (لأن live اختيارية)، لكن هذه هي النماذج **الموصى بها** للتغطية المنتظمة على جهاز تطوير يملك مفاتيح.

### مجموعة smoke الحديثة (استدعاء الأدوات + الصور)

هذا هو تشغيل "النماذج الشائعة" الذي نتوقع الحفاظ على عمله:

- OpenAI (غير Codex): `openai/gpt-5.4` (اختياري: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (أو `anthropic/claude-sonnet-4-6`)
- Google (Gemini API): `google/gemini-3.1-pro-preview` و `google/gemini-3-flash-preview` (تجنب نماذج Gemini 2.x الأقدم)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` و `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

شغّل اختبار smoke لـ gateway مع الأدوات + الصور:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### خط الأساس: استدعاء الأدوات (Read + Exec اختياري)

اختر واحدًا على الأقل من كل عائلة مزوّدين:

- OpenAI: `openai/gpt-5.4` (أو `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (أو `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (أو `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

تغطية إضافية اختيارية (من الجيد توفرها):

- xAI: `xai/grok-4` (أو أحدث إصدار متاح)
- Mistral: `mistral/`… (اختر نموذجًا واحدًا مفعّلًا وقادرًا على `tools`)
- Cerebras: `cerebras/`… (إذا كانت لديك صلاحية وصول)
- LM Studio: `lmstudio/`… (محلي؛ يعتمد استدعاء الأدوات على وضع API)

### Vision: إرسال الصور (مرفق ← رسالة متعددة الوسائط)

ضمّن نموذجًا واحدًا على الأقل قادرًا على التعامل مع الصور في `OPENCLAW_LIVE_GATEWAY_MODELS` (مثل Claude/Gemini/OpenAI بالإصدارات الداعمة للرؤية، إلخ) لممارسة probe الصور.

### المجمعات / البوابات البديلة

إذا كانت لديك مفاتيح مفعّلة، فإننا ندعم أيضًا الاختبار عبر:

- OpenRouter: `openrouter/...` (مئات النماذج؛ استخدم `openclaw models scan` للعثور على المرشحين القادرين على الأدوات+الصور)
- OpenCode: `opencode/...` لـ Zen و `opencode-go/...` لـ Go (المصادقة عبر `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

مزيد من المزوّدين الذين يمكنك تضمينهم في المصفوفة الحية (إذا كانت لديك بيانات الاعتماد/الإعداد):

- المضمّنون: `openai` و `openai-codex` و `anthropic` و `google` و `google-vertex` و `google-antigravity` و `google-gemini-cli` و `zai` و `openrouter` و `opencode` و `opencode-go` و `xai` و `groq` و `cerebras` و `mistral` و `github-copilot`
- عبر `models.providers` (نقاط نهاية مخصصة): `minimax` (سحابي/API)، بالإضافة إلى أي proxy متوافق مع OpenAI/Anthropic (مثل LM Studio و vLLM و LiteLLM وغيرها)

نصيحة: لا تحاول تثبيت "كل النماذج" بشكل صلب في المستندات. القائمة المرجعية هي ما يعيده `discoverModels(...)` على جهازك + المفاتيح المتاحة.

## بيانات الاعتماد (لا تُلتزم أبدًا)

تكتشف الاختبارات الحية بيانات الاعتماد بالطريقة نفسها التي يعمل بها CLI. الآثار العملية:

- إذا كان CLI يعمل، فيجب أن تعثر الاختبارات الحية على المفاتيح نفسها.
- إذا قال اختبار حي "لا توجد بيانات اعتماد"، فقم بالتصحيح بالطريقة نفسها التي ستصحح بها `openclaw models list` / اختيار النموذج.

- ملفات profile للمصادقة لكل وكيل: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (وهذا ما تعنيه "profile keys" في الاختبارات الحية)
- الإعداد: `~/.openclaw/openclaw.json` (أو `OPENCLAW_CONFIG_PATH`)
- دليل الحالة القديم: `~/.openclaw/credentials/` (يُنسخ إلى home الحي المجهّز عند وجوده، لكنه ليس مخزن المفاتيح الرئيسي لـ profile)
- تقوم التشغيلات الحية المحلية بنسخ الإعداد النشط وملفات `auth-profiles.json` لكل وكيل و `credentials/` القديم وأدلة مصادقة CLI الخارجية المدعومة إلى home اختبار مؤقت افتراضيًا؛ وتتجاوز البيئات الحية المجهّزة `workspace/` و `sandboxes/`، كما تُزال تجاوزات المسار `agents.*.workspace` / `agentDir` حتى تبقى probes بعيدة عن مساحة العمل الحقيقية على المضيف.

إذا كنت تريد الاعتماد على مفاتيح env (مثل تلك المصدّرة في `~/.profile`)، فشغّل الاختبارات المحلية بعد `source ~/.profile`، أو استخدم مشغّلات Docker أدناه (يمكنها تركيب `~/.profile` داخل الحاوية).

## Deepgram live (نسخ الصوت)

- الاختبار: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- التمكين: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- الاختبار: `src/agents/byteplus.live.test.ts`
- التمكين: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- تجاوز النموذج الاختياري: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- الاختبار: `extensions/comfy/comfy.live.test.ts`
- التمكين: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- النطاق:
  - يمارس المسارات المضمّنة الخاصة بـ comfy للصور والفيديو و `music_generate`
  - يتجاوز كل قدرة ما لم يكن `models.providers.comfy.<capability>` مضبوطًا
  - مفيد بعد تغيير تقديم تدفق عمل comfy أو polling أو التنزيلات أو تسجيل Plugin

## Image generation live

- الاختبار: `src/image-generation/runtime.live.test.ts`
- الأمر: `pnpm test:live src/image-generation/runtime.live.test.ts`
- الحزام: `pnpm test:live:media image`
- النطاق:
  - يعدّد كل Plugin مزوّد مسجل لتوليد الصور
  - يحمّل متغيرات env المفقودة الخاصة بالمزوّد من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/من env قبل ملفات profile المخزنة افتراضيًا، حتى لا تخفي مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات اعتماد shell الحقيقية
  - يتجاوز المزوّدين الذين لا يملكون مصادقة/profile/نموذجًا صالحًا
  - يشغّل المتغيرات القياسية لتوليد الصور عبر القدرة المشتركة لوقت التشغيل:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- المزوّدون المضمّنون الحاليون الذين تتم تغطيتهم:
  - `openai`
  - `google`
- التضييق الاختياري:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- سلوك المصادقة الاختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن profile وتجاهل التجاوزات المعتمدة على env فقط

## Music generation live

- الاختبار: `extensions/music-generation-providers.live.test.ts`
- التمكين: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- الحزام: `pnpm test:live:media music`
- النطاق:
  - يمارس المسار المضمّن المشترك لمزوّدات توليد الموسيقى
  - يغطي حاليًا Google و MiniMax
  - يحمّل متغيرات env الخاصة بالمزوّد من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/من env قبل ملفات profile المخزنة افتراضيًا، حتى لا تخفي مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات اعتماد shell الحقيقية
  - يتجاوز المزوّدين الذين لا يملكون مصادقة/profile/نموذجًا صالحًا
  - يشغّل وضعي وقت التشغيل المعلنين كليهما عند توفرهما:
    - `generate` مع إدخال يعتمد على prompt فقط
    - `edit` عندما يعلن المزوّد `capabilities.edit.enabled`
  - التغطية الحالية في المسار المشترك:
    - `google`: `generate` و `edit`
    - `minimax`: `generate`
    - `comfy`: ملف Comfy حي منفصل، وليس هذا المسح المشترك
- التضييق الاختياري:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- سلوك المصادقة الاختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن profile وتجاهل التجاوزات المعتمدة على env فقط

## Video generation live

- الاختبار: `extensions/video-generation-providers.live.test.ts`
- التمكين: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- الحزام: `pnpm test:live:media video`
- النطاق:
  - يمارس المسار المضمّن المشترك لمزوّدات توليد الفيديو
  - يحمّل متغيرات env الخاصة بالمزوّد من shell تسجيل الدخول (`~/.profile`) قبل الفحص
  - يستخدم مفاتيح API الحية/من env قبل ملفات profile المخزنة افتراضيًا، حتى لا تخفي مفاتيح الاختبار القديمة في `auth-profiles.json` بيانات اعتماد shell الحقيقية
  - يتجاوز المزوّدين الذين لا يملكون مصادقة/profile/نموذجًا صالحًا
  - يشغّل أوضاع وقت التشغيل المعلنة كليهما عند توفرهما:
    - `generate` مع إدخال يعتمد على prompt فقط
    - `imageToVideo` عندما يعلن المزوّد `capabilities.imageToVideo.enabled` ويقبل المزوّد/النموذج المحدد إدخال الصور المحلية المعتمد على buffer في المسح المشترك
    - `videoToVideo` عندما يعلن المزوّد `capabilities.videoToVideo.enabled` ويقبل المزوّد/النموذج المحدد إدخال الفيديو المحلي المعتمد على buffer في المسح المشترك
  - المزوّدون المعلن عنهم حاليًا لكن المتجاوزون في المسح المشترك لـ `imageToVideo`:
    - `vydra` لأن `veo3` المضمّن نصي فقط و `kling` المضمّن يتطلب عنوان URL بعيدًا للصورة
  - تغطية Vydra الخاصة بالمزوّد:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - يشغّل هذا الملف `veo3` لتحويل النص إلى فيديو بالإضافة إلى مسار `kling` يستخدم fixture لعنوان URL بعيد للصورة افتراضيًا
  - التغطية الحية الحالية لـ `videoToVideo`:
    - `runway` فقط عندما يكون النموذج المحدد هو `runway/gen4_aleph`
  - المزوّدون المعلن عنهم حاليًا لكن المتجاوزون في المسح المشترك لـ `videoToVideo`:
    - `alibaba` و `qwen` و `xai` لأن هذه المسارات تتطلب حاليًا عناوين URL مرجعية بعيدة من نوع `http(s)` / MP4
    - `google` لأن مسار Gemini/Veo المشترك الحالي يستخدم إدخالًا محليًا معتمدًا على buffer وهذا المسار غير مقبول في المسح المشترك
    - `openai` لأن المسار المشترك الحالي يفتقر إلى ضمانات وصول خاصة بالمؤسسة لفيديو inpaint/remix
- التضييق الاختياري:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- سلوك المصادقة الاختياري:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لفرض مصادقة مخزن profile وتجاهل التجاوزات المعتمدة على env فقط

## حزام media live

- الأمر: `pnpm test:live:media`
- الغرض:
  - يشغّل مجموعات image و music و video الحية المشتركة عبر نقطة دخول أصلية واحدة للمستودع
  - يحمّل تلقائيًا متغيرات env المفقودة الخاصة بالمزوّد من `~/.profile`
  - يضيّق تلقائيًا كل مجموعة إلى المزوّدين الذين يملكون حاليًا مصادقة صالحة افتراضيًا
  - يعيد استخدام `scripts/test-live.mjs`، بحيث يبقى سلوك Heartbeat والوضع الهادئ متسقًا
- أمثلة:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## مشغّلات Docker (فحوصات اختيارية من نوع "تعمل على Linux")

تنقسم مشغّلات Docker هذه إلى فئتين:

- مشغّلات النماذج الحية: `test:docker:live-models` و `test:docker:live-gateway` يشغّلان فقط ملف الاختبار الحي المطابق لكل منهما والمخصص لمفاتيح profile داخل صورة Docker الخاصة بالمستودع (`src/agents/models.profiles.live.test.ts` و `src/gateway/gateway-models.profiles.live.test.ts`)، مع تركيب دليل الإعداد المحلي ومساحة العمل لديك (ومع تحميل `~/.profile` إذا جرى تركيبه). نقاط الدخول المحلية المطابقة هي `test:live:models-profiles` و `test:live:gateway-profiles`.
- تستخدم مشغّلات Docker الحية افتراضيًا حدًا أصغر لاختبار smoke حتى يبقى المسح الكامل داخل Docker عمليًا:
  يستخدم `test:docker:live-models` افتراضيًا `OPENCLAW_LIVE_MAX_MODELS=12`، ويستخدم
  `test:docker:live-gateway` افتراضيًا `OPENCLAW_LIVE_GATEWAY_SMOKE=1`،
  و `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`،
  و `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`، و
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. تجاوز متغيرات env هذه عندما
  تريد صراحةً المسح الأكبر والأشمل.
- يبني `test:docker:all` صورة Docker الحية مرة واحدة عبر `test:docker:live-build`، ثم يعيد استخدامها لمساري Docker الحيين.
- مشغّلات smoke الخاصة بالحاويات: `test:docker:openwebui` و `test:docker:onboard` و `test:docker:gateway-network` و `test:docker:mcp-channels` و `test:docker:plugins` تشغّل حاوية واحدة أو أكثر حقيقية وتتحقق من مسارات تكامل ذات مستوى أعلى.

تقوم مشغّلات Docker الخاصة بالنماذج الحية أيضًا بتركيب أدلة home الخاصة بمصادقة CLI المطلوبة فقط (أو جميع الأدلة المدعومة عندما لا يكون التشغيل مضيّقًا)، ثم تنسخها إلى home الحاوية قبل التشغيل حتى تتمكن مصادقة OAuth الخاصة بـ CLI الخارجية من تحديث الرموز من دون تعديل مخزن المصادقة على المضيف:

- النماذج المباشرة: `pnpm test:docker:live-models` (السكريبت: `scripts/test-live-models-docker.sh`)
- اختبار smoke لربط ACP: `pnpm test:docker:live-acp-bind` (السكريبت: `scripts/test-live-acp-bind-docker.sh`)
- اختبار smoke لواجهة CLI الخلفية: `pnpm test:docker:live-cli-backend` (السكريبت: `scripts/test-live-cli-backend-docker.sh`)
- اختبار smoke لـ Codex app-server harness: `pnpm test:docker:live-codex-harness` (السكريبت: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + dev agent: `pnpm test:docker:live-gateway` (السكريبت: `scripts/test-live-gateway-models-docker.sh`)
- اختبار smoke حي لـ Open WebUI: `pnpm test:docker:openwebui` (السكريبت: `scripts/e2e/openwebui-docker.sh`)
- معالج onboarding (TTY، تهيئة كاملة): `pnpm test:docker:onboard` (السكريبت: `scripts/e2e/onboard-docker.sh`)
- شبكات Gateway (حاويتان، مصادقة WS + الصحة): `pnpm test:docker:gateway-network` (السكريبت: `scripts/e2e/gateway-network-docker.sh`)
- جسر قناة MCP (Gateway مُهيأ مسبقًا + جسر stdio + اختبار smoke خام لإطارات الإشعارات على نمط Claude): `pnpm test:docker:mcp-channels` (السكريبت: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (اختبار smoke للتثبيت + الاسم المستعار `/plugin` + دلالات إعادة تشغيل Claude-bundle): `pnpm test:docker:plugins` (السكريبت: `scripts/e2e/plugins-docker.sh`)

تقوم مشغّلات Docker الخاصة بالنماذج الحية أيضًا بتركيب النسخة الحالية من المستودع
بوضع القراءة فقط وتجهيزها في workdir مؤقت داخل الحاوية. وهذا يُبقي صورة وقت التشغيل
نحيفة، مع الاستمرار في تشغيل Vitest مقابل source/config المحليين الدقيقين لديك.
تتجاوز خطوة التجهيز التخزينات المؤقتة المحلية الكبيرة فقط ومخرجات بناء التطبيقات مثل
`.pnpm-store` و `.worktrees` و `__openclaw_vitest__` وأدلة `.build` المحلية للتطبيق أو
مخرجات Gradle حتى لا تقضي التشغيلات الحية داخل Docker دقائق في نسخ
القطع الأثرية الخاصة بكل جهاز.
كما تضبط أيضًا `OPENCLAW_SKIP_CHANNELS=1` حتى لا تبدأ probes الحية الخاصة بـ gateway
عمّال القنوات الحقيقية مثل Telegram/Discord وغيرها داخل الحاوية.
لا يزال `test:docker:live-models` يشغّل `pnpm test:live`، لذا مرّر
`OPENCLAW_LIVE_GATEWAY_*` أيضًا عندما تحتاج إلى تضييق أو استبعاد تغطية gateway
الحية من مسار Docker هذا.
يُعد `test:docker:openwebui` اختبار smoke توافقًا أعلى مستوى: فهو يبدأ
حاوية OpenClaw gateway مع تمكين نقاط نهاية HTTP المتوافقة مع OpenAI،
ثم يبدأ حاوية Open WebUI مثبتة الإصدار مقابل ذلك gateway، ويسجل الدخول عبر
Open WebUI، ويتحقق من أن `/api/models` يعرّض `openclaw/default`، ثم يرسل
طلب دردشة حقيقيًا عبر proxy الخاص بـ `/api/chat/completions` في Open WebUI.
قد يكون التشغيل الأول أبطأ بشكل ملحوظ لأن Docker قد يحتاج إلى سحب
صورة Open WebUI وقد تحتاج Open WebUI إلى إكمال إعداد البدء البارد الخاص بها.
يتوقع هذا المسار مفتاح نموذج حي صالحًا، ويُعد `OPENCLAW_PROFILE_FILE`
(الافتراضي `~/.profile`) الطريقة الأساسية لتوفيره في التشغيلات داخل Docker.
تطبع التشغيلات الناجحة حمولة JSON صغيرة مثل `{ "ok": true, "model":
"openclaw/default", ... }`.
يُعد `test:docker:mcp-channels` حتميًا عمدًا ولا يحتاج إلى
حساب حقيقي على Telegram أو Discord أو iMessage. فهو يشغّل حاوية Gateway
مُهيأة مسبقًا، ثم يبدأ حاوية ثانية تشغّل `openclaw mcp serve`، ثم
يتحقق من اكتشاف المحادثات الموجّهة، وقراءة transcript، وmetadata الخاصة بالمرفقات،
وسلوك طابور الأحداث الحية، وتوجيه الإرسال الصادر، وإشعارات القناة +
الأذونات على نمط Claude عبر جسر stdio MCP الحقيقي. يفحص اختبار الإشعارات
إطارات stdio MCP الخام مباشرة حتى يتحقق اختبار smoke مما يصدره الجسر
فعليًا، وليس فقط مما تعرضه مجموعة SDK معينة للعميل.

اختبار smoke يدوي لخيط ACP بلغة طبيعية عادية (ليس في CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- احتفظ بهذا السكريبت من أجل تدفقات عمل الانحدار/تصحيح الأخطاء. فقد تكون هناك حاجة إليه مرة أخرى للتحقق من توجيه خيوط ACP، لذلك لا تحذفه.

متغيرات env المفيدة:

- `OPENCLAW_CONFIG_DIR=...` (الافتراضي: `~/.openclaw`) ويُركّب إلى `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (الافتراضي: `~/.openclaw/workspace`) ويُركّب إلى `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (الافتراضي: `~/.profile`) ويُركّب إلى `/home/node/.profile` ويُحمّل قبل تشغيل الاختبارات
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (الافتراضي: `~/.cache/openclaw/docker-cli-tools`) ويُركّب إلى `/home/node/.npm-global` من أجل تثبيتات CLI المخزنة مؤقتًا داخل Docker
- تُركّب أدلة/ملفات مصادقة CLI الخارجية تحت `$HOME` بوضع القراءة فقط تحت `/host-auth...`، ثم تُنسخ إلى `/home/node/...` قبل بدء الاختبارات
  - الأدلة الافتراضية: `.minimax`
  - الملفات الافتراضية: `~/.codex/auth.json` و `~/.codex/config.toml` و `.claude.json` و `~/.claude/.credentials.json` و `~/.claude/settings.json` و `~/.claude/settings.local.json`
  - تشغيلات المزوّد المضيّقة تركّب فقط الأدلة/الملفات المطلوبة المستنتجة من `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - تجاوز ذلك يدويًا باستخدام `OPENCLAW_DOCKER_AUTH_DIRS=all` أو `OPENCLAW_DOCKER_AUTH_DIRS=none` أو قائمة مفصولة بفواصل مثل `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` لتضييق التشغيل
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` لتصفية المزوّدين داخل الحاوية
- `OPENCLAW_SKIP_DOCKER_BUILD=1` لإعادة استخدام صورة `openclaw:local-live` الحالية في إعادة التشغيلات التي لا تحتاج إلى إعادة بناء
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` لضمان أن تأتي بيانات الاعتماد من مخزن profile (وليس من env)
- `OPENCLAW_OPENWEBUI_MODEL=...` لاختيار النموذج الذي يعرّضه gateway لاختبار smoke الخاص بـ Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` لتجاوز prompt فحص nonce المستخدم في اختبار smoke الخاص بـ Open WebUI
- `OPENWEBUI_IMAGE=...` لتجاوز وسم صورة Open WebUI المثبتة

## سلامة المستندات

شغّل فحوصات المستندات بعد تعديلها: `pnpm check:docs`.
وشغّل التحقق الكامل من روابط/anchors الخاصة بـ Mintlify عندما تحتاج إلى فحوصات عناوين داخل الصفحة أيضًا: `pnpm docs:check-links:anchors`.

## اختبار انحدار دون اتصال (آمن لـ CI)

هذه اختبارات انحدار "لخط حقيقي" من دون مزوّدين حقيقيين:

- استدعاء أدوات Gateway (OpenAI وهمي، مع gateway + حلقة agent حقيقية): `src/gateway/gateway.test.ts` (الحالة: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- معالج Gateway (‏WS `wizard.start`/`wizard.next`، مع فرض كتابة config + auth): `src/gateway/gateway.test.ts` (الحالة: "runs wizard over ws and writes auth token config")

## تقييمات موثوقية الوكيل (Skills)

لدينا بالفعل بعض الاختبارات الآمنة لـ CI التي تتصرف مثل "تقييمات موثوقية الوكيل":

- استدعاء أدوات وهمية عبر حلقة gateway + agent الحقيقية (`src/gateway/gateway.test.ts`).
- تدفقات wizard من طرف إلى طرف تتحقق من توصيل الجلسة وتأثيرات config (`src/gateway/gateway.test.ts`).

ما لا يزال مفقودًا بالنسبة إلى Skills (راجع [Skills](/ar/tools/skills)):

- **اتخاذ القرار:** عند إدراج Skills في prompt، هل يختار الوكيل Skill الصحيحة (أو يتجنب غير ذات الصلة)؟
- **الامتثال:** هل يقرأ الوكيل `SKILL.md` قبل الاستخدام ويلتزم بالخطوات/الوسائط المطلوبة؟
- **عقود سير العمل:** سيناريوهات متعددة الأدوار تؤكد ترتيب الأدوات، واستمرار سجل الجلسة، وحدود sandbox.

يجب أن تبقى التقييمات المستقبلية حتمية أولًا:

- مشغّل سيناريوهات يستخدم مزوّدين وهميين لتأكيد استدعاءات الأدوات + ترتيبها، وقراءات ملفات Skill، وتوصيل الجلسات.
- مجموعة صغيرة من السيناريوهات المركزة على Skills (استخدام مقابل تجنب، والبوابات، وحقن prompt).
- تقييمات حية اختيارية (مقيّدة بالبيئة ومفعّلة اختياريًا) فقط بعد وضع المجموعة الآمنة لـ CI.

## اختبارات العقود (شكل Plugin والقناة)

تتحقق اختبارات العقود من أن كل Plugin وقناة مسجلين يلتزمان
بعقد الواجهة الخاصة بهما. فهي تتكرر على جميع Plugins المكتشفة وتشغّل مجموعة من
التحققات الخاصة بالشكل والسلوك. يتجاوز مسار unit الافتراضي `pnpm test`
عمدًا هذه الملفات المشتركة الخاصة بالوصلات واختبار smoke؛ شغّل أوامر العقود صراحةً
عندما تلمس الأسطح المشتركة للقنوات أو المزوّدين.

### الأوامر

- جميع العقود: `pnpm test:contracts`
- عقود القنوات فقط: `pnpm test:contracts:channels`
- عقود المزوّدين فقط: `pnpm test:contracts:plugins`

### عقود القنوات

تقع في `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - شكل Plugin الأساسي (المعرّف، والاسم، والقدرات)
- **setup** - عقد معالج الإعداد
- **session-binding** - سلوك ربط الجلسة
- **outbound-payload** - بنية حمولة الرسالة
- **inbound** - معالجة الرسائل الواردة
- **actions** - معالجات إجراءات القناة
- **threading** - معالجة معرّف الخيط
- **directory** - واجهة API للدليل/القائمة
- **group-policy** - فرض سياسة المجموعة

### عقود حالة المزوّد

تقع في `src/plugins/contracts/*.contract.test.ts`.

- **status** - probes حالة القناة
- **registry** - شكل سجل Plugin

### عقود المزوّدين

تقع في `src/plugins/contracts/*.contract.test.ts`:

- **auth** - عقد تدفق المصادقة
- **auth-choice** - اختيار/تحديد المصادقة
- **catalog** - واجهة API لفهرس النماذج
- **discovery** - اكتشاف Plugin
- **loader** - تحميل Plugin
- **runtime** - وقت تشغيل المزوّد
- **shape** - شكل/واجهة Plugin
- **wizard** - معالج الإعداد

### متى يجب التشغيل

- بعد تغيير صادرات أو مسارات `plugin-sdk`
- بعد إضافة Plugin قناة أو مزوّد أو تعديله
- بعد إعادة هيكلة تسجيل Plugin أو اكتشافه

تعمل اختبارات العقود في CI ولا تتطلب مفاتيح API حقيقية.

## إضافة اختبارات انحدار (إرشادات)

عند إصلاح مشكلة مزوّد/نموذج اكتُشفت في live:

- أضف اختبار انحدار آمنًا لـ CI إن أمكن (مزوّد وهمي/بديل، أو التقط شكل تحويل الطلب الدقيق)
- إذا كانت المشكلة حية بطبيعتها فقط (حدود المعدل، وسياسات المصادقة)، فأبقِ الاختبار الحي ضيقًا ومفعّلًا اختياريًا عبر متغيرات env
- فضّل استهداف أصغر طبقة تلتقط الخطأ:
  - خطأ تحويل/إعادة تشغيل طلب المزوّد → اختبار النماذج المباشرة
  - خطأ في خط الجلسة/السجل/الأدوات في gateway → اختبار smoke حي لـ gateway أو اختبار gateway وهمي وآمن لـ CI
- حاجز حماية عبور SecretRef:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` يشتق هدفًا نموذجيًا واحدًا لكل فئة SecretRef من metadata الخاصة بالسجل (`listSecretTargetRegistryEntries()`)، ثم يؤكد رفض معرّفات exec الخاصة بمقاطع العبور.
  - إذا أضفت عائلة هدف SecretRef جديدة مع `includeInPlan` في `src/secrets/target-registry-data.ts`، فحدّث `classifyTargetClass` في ذلك الاختبار. يفشل الاختبار عمدًا عند وجود معرّفات هدف غير مصنفة حتى لا يمكن تجاوز الفئات الجديدة بصمت.
