---
read_when:
    - عندما ترى التحذير OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - عندما ترى التحذير OPENCLAW_EXTENSION_API_DEPRECATED
    - عندما تكون بصدد تحديث إضافة إلى بنية الإضافات الحديثة
    - عندما تكون مسؤولًا عن إضافة OpenClaw خارجية
sidebarTitle: Migrate to SDK
summary: الانتقال من طبقة التوافق مع الإصدارات السابقة القديمة إلى Plugin SDK الحديث
title: ترحيل Plugin SDK
x-i18n:
    generated_at: "2026-04-06T07:19:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 94f12d1376edd8184714cc4dbea4a88fa8ed652f65e9365ede6176f3bf441b33
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# ترحيل Plugin SDK

انتقل OpenClaw من طبقة توافق واسعة مع الإصدارات السابقة إلى بنية إضافات
حديثة تتضمن عمليات استيراد محددة وموثقة. إذا كانت إضافتك قد بُنيت قبل
البنية الجديدة، فسيساعدك هذا الدليل على الترحيل.

## ما الذي يتغير

كان نظام الإضافات القديم يوفّر سطحين مفتوحين على اتساعهما يتيحان للإضافات
استيراد أي شيء تحتاجه من نقطة دخول واحدة:

- **`openclaw/plugin-sdk/compat`** — استيراد واحد يعيد تصدير عشرات
  الأدوات المساعدة. تم تقديمه لإبقاء الإضافات الأقدم المعتمدة على الخطافات
  تعمل أثناء بناء بنية الإضافات الجديدة.
- **`openclaw/extension-api`** — جسر يتيح للإضافات وصولًا مباشرًا إلى
  الأدوات المساعدة من جهة المضيف مثل مشغّل الوكيل المضمّن.

كلا السطحين الآن **مهجوران**. ما زالا يعملان وقت التشغيل، لكن يجب ألا
تستخدمهما الإضافات الجديدة، ويجب على الإضافات الحالية الترحيل قبل أن
تزيلهما الإصدارة الرئيسية التالية.

<Warning>
  ستُزال طبقة التوافق مع الإصدارات السابقة في إصدار رئيسي مستقبلي.
  الإضافات التي ما زالت تستورد من هذه الأسطح ستتعطل عند حدوث ذلك.
</Warning>

## لماذا تغيّر هذا

تسبب النهج القديم في مشكلات:

- **بدء تشغيل بطيء** — كان استيراد أداة مساعدة واحدة يحمّل عشرات الوحدات غير المرتبطة
- **تبعيات دائرية** — كانت إعادة التصدير الواسعة تجعل إنشاء دورات استيراد أمرًا سهلًا
- **سطح API غير واضح** — لم تكن هناك طريقة لمعرفة أي الصادرات مستقرة وأيها داخلية

يصلح Plugin SDK الحديث هذا: فكل مسار استيراد (`openclaw/plugin-sdk/\<subpath\>`)
هو وحدة صغيرة مستقلة ذات غرض واضح وعقد موثق.

كما أزيلت أيضًا طبقات الراحة القديمة الخاصة بالموفّرات للقنوات المضمّنة. إن
عمليات الاستيراد مثل `openclaw/plugin-sdk/slack` و`openclaw/plugin-sdk/discord`
و`openclaw/plugin-sdk/signal` و`openclaw/plugin-sdk/whatsapp`،
وطبقات الأدوات المساعدة ذات العلامة الخاصة بالقنوات، و
`openclaw/plugin-sdk/telegram-core` لم تكن عقود إضافات مستقرة،
بل كانت اختصارات خاصة بمستودع mono داخلي. استخدم بدلًا منها مسارات فرعية
عامة وضيقة ضمن SDK. وداخل مساحة عمل الإضافات المضمّنة، احتفظ بالأدوات
المساعدة المملوكة للموفّر داخل `api.ts` أو `runtime-api.ts` الخاصين بتلك الإضافة.

أمثلة الموفّرات المضمّنة الحالية:

- يحتفظ Anthropic بأدوات البث الخاصة بـ Claude في طبقة `api.ts` /
  `contract-api.ts` الخاصة به
- يحتفظ OpenAI ببُناة الموفّر، وأدوات النموذج الافتراضي، وبُناة موفّر
  الوقت الحقيقي في `api.ts` الخاص به
- يحتفظ OpenRouter بباني الموفّر وأدوات الإعداد/التهيئة في
  `api.ts` الخاص به

## كيفية الترحيل

<Steps>
  <Step title="راجِع سلوك الرجوع إلى غلاف Windows">
    إذا كانت إضافتك تستخدم `openclaw/plugin-sdk/windows-spawn`، فإن أغلفة
    Windows من نوع `.cmd`/`.bat` غير المحلولة ستفشل الآن بإغلاق آمن ما لم
    تمرّر صراحةً `allowShellFallback: true`.

    ```typescript
    // Before
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // After
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Only set this for trusted compatibility callers that intentionally
      // accept shell-mediated fallback.
      allowShellFallback: true,
    });
    ```

    إذا لم يكن المستدعي لديك يعتمد عمدًا على الرجوع إلى shell، فلا تضبط
    `allowShellFallback` وتعامل مع الخطأ المُلقى بدلًا من ذلك.

  </Step>

  <Step title="اعثر على عمليات الاستيراد المهجورة">
    ابحث في إضافتك عن عمليات الاستيراد من أيٍّ من السطحين المهجورين:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="استبدلها بعمليات استيراد محددة">
    كل عنصر مُصدَّر من السطح القديم يقابله مسار استيراد حديث ومحدد:

    ```typescript
    // Before (deprecated backwards-compatibility layer)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // After (modern focused imports)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    بالنسبة إلى الأدوات المساعدة من جهة المضيف، استخدم وقت تشغيل الإضافة
    المحقون بدلًا من الاستيراد المباشر:

    ```typescript
    // Before (deprecated extension-api bridge)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // After (injected runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    ينطبق النمط نفسه على أدوات الجسر القديمة الأخرى:

    | الاستيراد القديم | المكافئ الحديث |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | أدوات مساعد مخزن الجلسات | `api.runtime.agent.session.*` |

  </Step>

  <Step title="ابنِ واختبر">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## مرجع مسارات الاستيراد

<Accordion title="جدول مسارات الاستيراد الشائعة">
  | مسار الاستيراد | الغرض | أهم الصادرات |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | أداة دخول الإضافة القياسية | `definePluginEntry` |
  | `plugin-sdk/core` | إعادة تصدير جامعة قديمة لتعريفات/بُنّاة دخول القنوات | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | تصدير مخطط التهيئة الجذري | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | أداة دخول موفّر واحد | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | تعريفات وبُنّاة دخول القنوات المركزة | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | أدوات مساعد معالج الإعداد المشتركة | مطالبات قائمة السماح، وبُنّاة حالة الإعداد |
  | `plugin-sdk/setup-runtime` | أدوات مساعد وقت التشغيل الخاصة بالإعداد | محوّلات رقع الإعداد الآمنة للاستيراد، وأدوات ملاحظات البحث، و`promptResolvedAllowFrom` و`splitSetupEntries` ووكلاء الإعداد المفوّضون |
  | `plugin-sdk/setup-adapter-runtime` | أدوات مساعد محوّل الإعداد | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | أدوات أدوات الإعداد | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | أدوات مساعد الحسابات المتعددة | أدوات مساعد قائمة/تهيئة/بوابة إجراءات الحساب |
  | `plugin-sdk/account-id` | أدوات مساعد معرّف الحساب | `DEFAULT_ACCOUNT_ID`، وتطبيع معرّف الحساب |
  | `plugin-sdk/account-resolution` | أدوات مساعد البحث عن الحساب | أدوات مساعد البحث عن الحساب + الرجوع إلى الافتراضي |
  | `plugin-sdk/account-helpers` | أدوات مساعد حساب ضيقة | أدوات مساعد قائمة الحساب/إجراءات الحساب |
  | `plugin-sdk/channel-setup` | محوّلات معالج الإعداد | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, بالإضافة إلى `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | بدائيات الاقتران عبر DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | توصيل بادئة الرد + الكتابة | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | مصانع محوّلات التهيئة | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | بُنّاة مخطط التهيئة | أنواع مخطط تهيئة القناة |
  | `plugin-sdk/telegram-command-config` | أدوات مساعد تهيئة أوامر Telegram | تطبيع اسم الأمر، وتشذيب الوصف، والتحقق من التكرار/التعارض |
  | `plugin-sdk/channel-policy` | تحليل سياسات المجموعات/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | تتبع حالة الحساب | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | أدوات مساعد الغلاف الوارد | أدوات مساعد المسار المشترك + باني الغلاف |
  | `plugin-sdk/inbound-reply-dispatch` | أدوات مساعد الرد الوارد | أدوات مساعد التسجيل والإرسال المشترك |
  | `plugin-sdk/messaging-targets` | تحليل أهداف المراسلة | أدوات مساعد تحليل/مطابقة الأهداف |
  | `plugin-sdk/outbound-media` | أدوات مساعد الوسائط الصادرة | تحميل الوسائط الصادرة المشترك |
  | `plugin-sdk/outbound-runtime` | أدوات مساعد وقت التشغيل الصادر | أدوات مساعد هوية/مفوّض الإرسال الصادر |
  | `plugin-sdk/thread-bindings-runtime` | أدوات مساعد ربط سلاسل الرسائل | أدوات مساعد دورة حياة ربط السلاسل والمحوّل |
  | `plugin-sdk/agent-media-payload` | أدوات مساعد حمولة الوسائط القديمة | باني حمولة وسائط الوكيل لتخطيطات الحقول القديمة |
  | `plugin-sdk/channel-runtime` | طبقة توافق وسيطة مهجورة | أدوات وقت تشغيل القنوات القديمة فقط |
  | `plugin-sdk/channel-send-result` | أنواع نتيجة الإرسال | أنواع نتيجة الرد |
  | `plugin-sdk/runtime-store` | تخزين الإضافة الدائم | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | أدوات مساعد وقت تشغيل واسعة | أدوات مساعد وقت التشغيل/التسجيل/النسخ الاحتياطي/تثبيت الإضافات |
  | `plugin-sdk/runtime-env` | أدوات مساعد بيئة وقت تشغيل ضيقة | أدوات مساعد المسجل/بيئة وقت التشغيل، والمهلة، وإعادة المحاولة، والتراجع التدريجي |
  | `plugin-sdk/plugin-runtime` | أدوات مساعد وقت تشغيل الإضافة المشتركة | أدوات مساعد أوامر/خطافات/HTTP/تفاعلية للإضافة |
  | `plugin-sdk/hook-runtime` | أدوات مساعد مسار الخطافات | أدوات مساعد مسار خطافات webhook/الخطافات الداخلية المشتركة |
  | `plugin-sdk/lazy-runtime` | أدوات مساعد وقت التشغيل الكسول | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | أدوات مساعد العمليات | أدوات مساعد exec المشتركة |
  | `plugin-sdk/cli-runtime` | أدوات مساعد وقت تشغيل CLI | أدوات تنسيق الأوامر، والانتظار، وإصدار النسخة |
  | `plugin-sdk/gateway-runtime` | أدوات مساعد البوابة | عميل البوابة وأدوات تصحيح حالة القناة |
  | `plugin-sdk/config-runtime` | أدوات مساعد التهيئة | أدوات مساعد تحميل/كتابة التهيئة |
  | `plugin-sdk/telegram-command-config` | أدوات مساعد أوامر Telegram | أدوات مساعد التحقق من أوامر Telegram الثابتة كخيار بديل عندما لا يكون سطح عقد Telegram المضمّن متاحًا |
  | `plugin-sdk/approval-runtime` | أدوات مساعد مطالبة الموافقة | حمولة موافقة exec/الإضافة، وأدوات مساعد إمكانات/ملف الموافقة، وأدوات مساعد توجيه/وقت تشغيل الموافقة الأصلية |
  | `plugin-sdk/approval-auth-runtime` | أدوات مساعد مصادقة الموافقة | تحليل الموافق، ومصادقة الإجراء داخل المحادثة نفسها |
  | `plugin-sdk/approval-client-runtime` | أدوات مساعد عميل الموافقة | أدوات مساعد ملف/مرشح موافقة exec الأصلية |
  | `plugin-sdk/approval-delivery-runtime` | أدوات مساعد تسليم الموافقة | محوّلات إمكانات/تسليم الموافقة الأصلية |
  | `plugin-sdk/approval-native-runtime` | أدوات مساعد هدف الموافقة | أدوات مساعد ربط هدف/حساب الموافقة الأصلية |
  | `plugin-sdk/approval-reply-runtime` | أدوات مساعد رد الموافقة | أدوات مساعد حمولة رد موافقة exec/الإضافة |
  | `plugin-sdk/security-runtime` | أدوات مساعد الأمان | أدوات مساعد الثقة المشتركة، وبوابات DM، والمحتوى الخارجي، وتجميع الأسرار |
  | `plugin-sdk/ssrf-policy` | أدوات مساعد سياسة SSRF | أدوات مساعد قائمة السماح للمضيف وسياسة الشبكة الخاصة |
  | `plugin-sdk/ssrf-runtime` | أدوات مساعد وقت تشغيل SSRF | `Pinned-dispatcher`، و`guarded fetch`، وأدوات مساعد سياسة SSRF |
  | `plugin-sdk/collection-runtime` | أدوات مساعد التخزين المؤقت المحدود | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | أدوات مساعد بوابة التشخيص | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | أدوات مساعد تنسيق الأخطاء | `formatUncaughtError`, `isApprovalNotFoundError`، وأدوات مساعد رسم الأخطاء |
  | `plugin-sdk/fetch-runtime` | أدوات مساعد fetch/proxy المغلفة | `resolveFetch`، وأدوات مساعد proxy |
  | `plugin-sdk/host-runtime` | أدوات مساعد تطبيع المضيف | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | أدوات مساعد إعادة المحاولة | `RetryConfig`, `retryAsync`، ومشغلات السياسة |
  | `plugin-sdk/allow-from` | تنسيق قائمة السماح | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | ربط مدخلات قائمة السماح | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | بوابات الأوامر وأدوات سطح الأوامر | `resolveControlCommandGate`، وأدوات مساعد تفويض المرسِل، وأدوات مساعد سجل الأوامر |
  | `plugin-sdk/secret-input` | تحليل مدخلات الأسرار | أدوات مساعد مدخلات الأسرار |
  | `plugin-sdk/webhook-ingress` | أدوات مساعد طلبات webhook | أدوات webhook المساعدة الخاصة بالهدف |
  | `plugin-sdk/webhook-request-guards` | أدوات مساعد حماية جسم webhook | أدوات مساعد قراءة/تقييد جسم الطلب |
  | `plugin-sdk/reply-runtime` | وقت تشغيل الرد المشترك | الإرسال الوارد، والنبض، ومخطط الرد، والتقسيم |
  | `plugin-sdk/reply-dispatch-runtime` | أدوات مساعد إرسال الرد الضيقة | أدوات مساعد الإنهاء + الإرسال الخاصة بالموفّر |
  | `plugin-sdk/reply-history` | أدوات مساعد سجل الردود | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | تخطيط مرجع الرد | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | أدوات مساعد تقسيم الرد | أدوات مساعد تقسيم النص/Markdown |
  | `plugin-sdk/session-store-runtime` | أدوات مساعد مخزن الجلسات | أدوات مساعد مسار المخزن + وقت آخر تحديث |
  | `plugin-sdk/state-paths` | أدوات مساعد مسارات الحالة | أدوات مساعد مجلدات الحالة وOAuth |
  | `plugin-sdk/routing` | أدوات مساعد التوجيه/مفتاح الجلسة | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`، وأدوات مساعد تطبيع مفتاح الجلسة |
  | `plugin-sdk/status-helpers` | أدوات مساعد حالة القناة | بُنّاة ملخص حالة القناة/الحساب، والافتراضيات الخاصة بحالة وقت التشغيل، وأدوات بيانات المشكلات |
  | `plugin-sdk/target-resolver-runtime` | أدوات مساعد محلل الهدف | أدوات مساعد محلل الهدف المشتركة |
  | `plugin-sdk/string-normalization-runtime` | أدوات مساعد تطبيع السلاسل | أدوات مساعد تطبيع slug/السلاسل |
  | `plugin-sdk/request-url` | أدوات مساعد URL للطلب | استخراج URL نصي من مدخلات شبيهة بالطلب |
  | `plugin-sdk/run-command` | أدوات مساعد الأوامر الموقّتة | مشغّل أوامر موقّت مع `stdout`/`stderr` مطبّعين |
  | `plugin-sdk/param-readers` | قارئات المعاملات | قارئات معاملات الأدوات/CLI الشائعة |
  | `plugin-sdk/tool-send` | استخراج إرسال الأداة | استخراج حقول هدف الإرسال القياسية من وسيطات الأداة |
  | `plugin-sdk/temp-path` | أدوات مساعد المسار المؤقت | أدوات مساعد مسار التحميل المؤقت المشترك |
  | `plugin-sdk/logging-core` | أدوات مساعد التسجيل | مسجل النظام الفرعي وأدوات إخفاء البيانات |
  | `plugin-sdk/markdown-table-runtime` | أدوات مساعد جداول Markdown | أدوات مساعد أنماط جداول Markdown |
  | `plugin-sdk/reply-payload` | أنواع رد الرسائل | أنواع حمولة الرد |
  | `plugin-sdk/provider-setup` | أدوات إعداد منتقاة لموفّرات محلية/مستضافة ذاتيًا | أدوات مساعد اكتشاف/تهيئة الموفّر المستضاف ذاتيًا |
  | `plugin-sdk/self-hosted-provider-setup` | أدوات إعداد مركزة لموفّرات OpenAI-compatible المستضافة ذاتيًا | الأدوات نفسها لاكتشاف/تهيئة الموفّر المستضاف ذاتيًا |
  | `plugin-sdk/provider-auth-runtime` | أدوات مساعد مصادقة وقت تشغيل الموفّر | أدوات مساعد تحليل مفتاح API في وقت التشغيل |
  | `plugin-sdk/provider-auth-api-key` | أدوات مساعد إعداد مفتاح API للموفّر | أدوات مساعد التهيئة الأولية/كتابة الملف التعريفي لمفتاح API |
  | `plugin-sdk/provider-auth-result` | أدوات مساعد نتيجة مصادقة الموفّر | باني نتيجة مصادقة OAuth القياسي |
  | `plugin-sdk/provider-auth-login` | أدوات مساعد تسجيل الدخول التفاعلي للموفّر | أدوات مساعد تسجيل الدخول التفاعلي المشتركة |
  | `plugin-sdk/provider-env-vars` | أدوات مساعد متغيرات بيئة الموفّر | أدوات مساعد البحث عن متغيرات بيئة مصادقة الموفّر |
  | `plugin-sdk/provider-model-shared` | أدوات مساعد نموذج/إعادة تشغيل الموفّر المشتركة | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`، وبُنّاة سياسة إعادة التشغيل المشتركة، وأدوات نقاط نهاية الموفّر المساعدة، وأدوات تطبيع معرّف النموذج |
  | `plugin-sdk/provider-catalog-shared` | أدوات مساعد فهرس الموفّر المشتركة | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | ترقيعات التهيئة الأولية للموفّر | أدوات مساعد تهيئة onboarding |
  | `plugin-sdk/provider-http` | أدوات مساعد HTTP للموفّر | أدوات مساعد HTTP العامة للموفّر/إمكانات نقاط النهاية |
  | `plugin-sdk/provider-web-fetch` | أدوات مساعد web-fetch للموفّر | أدوات مساعد تسجيل/تخزين مؤقت لموفّر web-fetch |
  | `plugin-sdk/provider-web-search` | أدوات مساعد web-search للموفّر | أدوات مساعد تسجيل/تخزين مؤقت/تهيئة لموفّر web-search |
  | `plugin-sdk/provider-tools` | أدوات مساعد التوافق بين أدوات/مخططات الموفّر | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`، وتنظيف مخطط Gemini + التشخيص، وأدوات التوافق الخاصة بـ xAI مثل `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | أدوات مساعد استخدام الموفّر | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage`، وأدوات مساعد استخدام الموفّر الأخرى |
  | `plugin-sdk/provider-stream` | أدوات مساعد غلاف بث الموفّر | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`، وأنواع أغلفة البث، وأدوات أغلفة Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot المشتركة |
  | `plugin-sdk/keyed-async-queue` | طابور async مرتب | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | أدوات مساعد الوسائط المشتركة | أدوات مساعد جلب/تحويل/تخزين الوسائط بالإضافة إلى بُنّاة حمولة الوسائط |
  | `plugin-sdk/media-generation-runtime` | أدوات مساعد توليد الوسائط المشتركة | أدوات مساعد failover المشتركة، واختيار المرشح، ورسائل النماذج المفقودة لتوليد الصور/الفيديو/الموسيقى |
  | `plugin-sdk/media-understanding` | أدوات مساعد فهم الوسائط | أنواع موفّر فهم الوسائط بالإضافة إلى صادرات أدوات صورة/صوت موجهة للموفّر |
  | `plugin-sdk/text-runtime` | أدوات مساعد النص المشتركة | إزالة النص الظاهر للمساعد، وأدوات عرض/تقسيم/جداول markdown، وأدوات إخفاء البيانات، وأدوات وسم التوجيه، وأدوات النص الآمن، وأدوات النص/التسجيل ذات الصلة |
  | `plugin-sdk/text-chunking` | أدوات مساعد تقسيم النص | أداة تقسيم النص الصادر |
  | `plugin-sdk/speech` | أدوات مساعد الكلام | أنواع موفّر الكلام بالإضافة إلى صادرات الأدوات الموجهة للموفّر الخاصة بالتوجيه والسجل والتحقق |
  | `plugin-sdk/speech-core` | النواة المشتركة للكلام | أنواع موفّر الكلام، والسجل، والتوجيهات، والتطبيع |
  | `plugin-sdk/realtime-transcription` | أدوات مساعد النسخ الفوري | أنواع الموفّر وأدوات السجل المساعدة |
  | `plugin-sdk/realtime-voice` | أدوات مساعد الصوت الفوري | أنواع الموفّر وأدوات السجل المساعدة |
  | `plugin-sdk/image-generation-core` | النواة المشتركة لتوليد الصور | أنواع توليد الصور، وfailover، والمصادقة، وأدوات السجل المساعدة |
  | `plugin-sdk/music-generation` | أدوات مساعد توليد الموسيقى | أنواع موفّر/طلب/نتيجة توليد الموسيقى |
  | `plugin-sdk/music-generation-core` | النواة المشتركة لتوليد الموسيقى | أنواع توليد الموسيقى، وأدوات failover، والبحث عن الموفّر، وتحليل مرجع النموذج |
  | `plugin-sdk/video-generation` | أدوات مساعد توليد الفيديو | أنواع موفّر/طلب/نتيجة توليد الفيديو |
  | `plugin-sdk/video-generation-core` | النواة المشتركة لتوليد الفيديو | أنواع توليد الفيديو، وأدوات failover، والبحث عن الموفّر، وتحليل مرجع النموذج |
  | `plugin-sdk/interactive-runtime` | أدوات مساعد الرد التفاعلي | تطبيع/اختزال حمولة الرد التفاعلي |
  | `plugin-sdk/channel-config-primitives` | بدائيات تهيئة القناة | بدائيات ضيقة لمخطط تهيئة القناة |
  | `plugin-sdk/channel-config-writes` | أدوات مساعد كتابة تهيئة القناة | أدوات مساعد تفويض كتابة تهيئة القناة |
  | `plugin-sdk/channel-plugin-common` | تمهيد القناة المشترك | صادرات تمهيد إضافات القنوات المشتركة |
  | `plugin-sdk/channel-status` | أدوات مساعد حالة القناة | أدوات مساعد اللقطة/الملخص المشتركة لحالة القناة |
  | `plugin-sdk/allowlist-config-edit` | أدوات مساعد تهيئة قائمة السماح | أدوات مساعد تعديل/قراءة تهيئة قائمة السماح |
  | `plugin-sdk/group-access` | أدوات مساعد وصول المجموعات | أدوات مساعد قرارات وصول المجموعات المشتركة |
  | `plugin-sdk/direct-dm` | أدوات مساعد Direct-DM | أدوات مساعد مصادقة/حماية Direct-DM المشتركة |
  | `plugin-sdk/extension-shared` | أدوات مساعد الإضافات المشتركة | بدائيات أدوات الحالة/القنوات السلبية المساعدة |
  | `plugin-sdk/webhook-targets` | أدوات مساعد أهداف webhook | سجل أهداف webhook وأدوات تثبيت المسارات |
  | `plugin-sdk/webhook-path` | أدوات مساعد مسار webhook | أدوات مساعد تطبيع مسار webhook |
  | `plugin-sdk/web-media` | أدوات مساعد وسائط الويب المشتركة | أدوات مساعد تحميل الوسائط البعيدة/المحلية |
  | `plugin-sdk/zod` | إعادة تصدير Zod | إعادة تصدير `zod` لمستهلكي Plugin SDK |
  | `plugin-sdk/memory-core` | أدوات memory-core المضمّنة | سطح أدوات مدير/تهيئة/ملف/CLI الخاصة بالذاكرة |
  | `plugin-sdk/memory-core-engine-runtime` | واجهة وقت تشغيل محرك الذاكرة | واجهة وقت تشغيل فهرسة/بحث الذاكرة |
  | `plugin-sdk/memory-core-host-engine-foundation` | محرك أساس مضيف الذاكرة | صادرات محرك أساس مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-embeddings` | محرك التضمينات لمضيف الذاكرة | صادرات محرك التضمينات لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-qmd` | محرك QMD لمضيف الذاكرة | صادرات محرك QMD لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-storage` | محرك تخزين مضيف الذاكرة | صادرات محرك تخزين مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-multimodal` | أدوات مساعد الوسائط المتعددة لمضيف الذاكرة | أدوات مساعد الوسائط المتعددة لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-query` | أدوات مساعد استعلام مضيف الذاكرة | أدوات مساعد استعلام مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-secret` | أدوات مساعد أسرار مضيف الذاكرة | أدوات مساعد أسرار مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-events` | أدوات مساعد سجل أحداث مضيف الذاكرة | أدوات مساعد سجل أحداث مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-status` | أدوات مساعد حالة مضيف الذاكرة | أدوات مساعد حالة مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-cli` | وقت تشغيل CLI لمضيف الذاكرة | أدوات مساعد وقت تشغيل CLI لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-core` | وقت تشغيل النواة لمضيف الذاكرة | أدوات مساعد وقت تشغيل النواة لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-files` | أدوات مساعد ملفات/وقت تشغيل مضيف الذاكرة | أدوات مساعد ملفات/وقت تشغيل مضيف الذاكرة |
  | `plugin-sdk/memory-host-core` | اسم مستعار لوقت تشغيل نواة مضيف الذاكرة | اسم مستعار محايد للمورّد لأدوات مساعد وقت تشغيل نواة مضيف الذاكرة |
  | `plugin-sdk/memory-host-events` | اسم مستعار لسجل أحداث مضيف الذاكرة | اسم مستعار محايد للمورّد لأدوات مساعد سجل أحداث مضيف الذاكرة |
  | `plugin-sdk/memory-host-files` | اسم مستعار لملفات/وقت تشغيل مضيف الذاكرة | اسم مستعار محايد للمورّد لأدوات مساعد ملفات/وقت تشغيل مضيف الذاكرة |
  | `plugin-sdk/memory-host-markdown` | أدوات مساعد Markdown المُدار | أدوات مساعد Markdown المُدار المشتركة للإضافات المجاورة للذاكرة |
  | `plugin-sdk/memory-host-search` | واجهة بحث الذاكرة النشطة | واجهة وقت تشغيل كسولة لمدير بحث الذاكرة النشطة |
  | `plugin-sdk/memory-host-status` | اسم مستعار لحالة مضيف الذاكرة | اسم مستعار محايد للمورّد لأدوات مساعد حالة مضيف الذاكرة |
  | `plugin-sdk/memory-lancedb` | أدوات memory-lancedb المضمّنة | سطح أدوات memory-lancedb |
  | `plugin-sdk/testing` | أدوات الاختبار | أدوات الاختبار والمحاكيات |
</Accordion>

هذا الجدول هو عمدًا المجموعة الشائعة الخاصة بالترحيل، وليس كامل سطح
SDK. القائمة الكاملة التي تتجاوز 200 نقطة دخول موجودة في
`scripts/lib/plugin-sdk-entrypoints.json`.

وما تزال تلك القائمة تتضمن بعض طبقات أدوات المساعدة الخاصة بالإضافات المضمّنة مثل
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, و`plugin-sdk/matrix*`. تظل هذه المسارات مُصدَّرة من أجل
صيانة الإضافات المضمّنة والتوافق، لكنها مستبعَدة عمدًا من جدول الترحيل
الشائع وليست الهدف الموصى به لكود الإضافات الجديدة.

تنطبق القاعدة نفسها على عائلات الأدوات المساعدة المضمّنة الأخرى مثل:

- أدوات دعم المتصفح: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- أسطح أدوات/إضافات مضمّنة مثل `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership`, و`plugin-sdk/voice-call`

يكشف `plugin-sdk/github-copilot-token` حاليًا سطح أدوات الرموز الضيق
`DEFAULT_COPILOT_API_BASE_URL`,
و`deriveCopilotApiBaseUrlFromToken`، و`resolveCopilotApiToken`.

استخدم أضيق استيراد يطابق المهمة. إذا لم تتمكن من العثور على عنصر مُصدَّر،
فتحقق من المصدر في `src/plugin-sdk/` أو اسأل في Discord.

## الجدول الزمني للإزالة

| متى | ما الذي يحدث |
| ---------------------- | ----------------------------------------------------------------------- |
| **الآن**                | تُصدر الأسطح المهجورة تحذيرات وقت التشغيل                               |
| **الإصدار الرئيسي التالي** | ستُزال الأسطح المهجورة؛ وستفشل الإضافات التي ما زالت تستخدمها |

لقد تم بالفعل ترحيل جميع الإضافات الأساسية. يجب على الإضافات الخارجية
الترحيل قبل الإصدار الرئيسي التالي.

## كتم التحذيرات مؤقتًا

اضبط متغيرات البيئة هذه أثناء عملك على الترحيل:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

هذا مخرج مؤقت، وليس حلًا دائمًا.

## ذو صلة

- [البدء](/ar/plugins/building-plugins) — أنشئ إضافتك الأولى
- [نظرة عامة على SDK](/ar/plugins/sdk-overview) — المرجع الكامل لعمليات الاستيراد عبر المسارات الفرعية
- [إضافات القنوات](/ar/plugins/sdk-channel-plugins) — إنشاء إضافات القنوات
- [إضافات الموفّرات](/ar/plugins/sdk-provider-plugins) — إنشاء إضافات الموفّرات
- [الآليات الداخلية للإضافة](/ar/plugins/architecture) — شرح معماري متعمق
- [بيان الإضافة](/ar/plugins/manifest) — مرجع مخطط البيان
