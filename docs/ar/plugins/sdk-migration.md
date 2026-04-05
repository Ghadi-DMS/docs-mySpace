---
read_when:
    - تظهر لك رسالة التحذير OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - تظهر لك رسالة التحذير OPENCLAW_EXTENSION_API_DEPRECATED
    - أنت تحدّث إضافة إلى معمارية الإضافات الحديثة
    - أنت مسؤول عن إضافة OpenClaw خارجية
sidebarTitle: Migrate to SDK
summary: الترحيل من طبقة التوافق العكسي القديمة إلى Plugin SDK الحديثة
title: ترحيل Plugin SDK
x-i18n:
    generated_at: "2026-04-05T12:52:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: c420b8d7de17aee16c5aa67e3a88da5750f0d84b07dd541f061081080e081196
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# ترحيل Plugin SDK

انتقلت OpenClaw من طبقة توافق عكسي واسعة إلى معمارية إضافات حديثة
ذات عمليات استيراد مركزة وموثقة. إذا كانت إضافتك قد بُنيت قبل
المعمارية الجديدة، فسيساعدك هذا الدليل على الترحيل.

## ما الذي يتغير

وفّر نظام الإضافات القديم سطحين مفتوحين على مصراعيهما كانا يسمحان للإضافات باستيراد
أي شيء تحتاج إليه من نقطة دخول واحدة:

- **`openclaw/plugin-sdk/compat`** — استيراد واحد يعيد تصدير عشرات
  المساعدات. وقد قُدّم لإبقاء الإضافات القديمة المعتمدة على hooks تعمل بينما كانت
  معمارية الإضافات الجديدة قيد البناء.
- **`openclaw/extension-api`** — جسر كان يمنح الإضافات وصولًا مباشرًا إلى
  مساعدات جهة المضيف مثل مشغّل الوكيل المدمج.

كلا السطحين أصبح الآن **مهملًا**. وما زالا يعملان في وقت التشغيل، لكن
يجب ألا تستخدمهما الإضافات الجديدة، وينبغي للإضافات الحالية الترحيل قبل أن يزيلهما
الإصدار الرئيسي التالي.

<Warning>
  ستتم إزالة طبقة التوافق العكسي في إصدار رئيسي مستقبلي.
  وستتعطل الإضافات التي ما زالت تستورد من هذه الواجهات عندما يحدث ذلك.
</Warning>

## لماذا تغير هذا

تسبب النهج القديم في مشكلات:

- **بدء تشغيل بطيء** — كان استيراد مساعد واحد يحمّل عشرات الوحدات غير المرتبطة
- **اعتماديات دائرية** — جعلت عمليات إعادة التصدير الواسعة من السهل إنشاء دورات استيراد
- **سطح API غير واضح** — لم تكن هناك طريقة لمعرفة أي التصديرات كانت مستقرة وأيها داخلية

تصلح Plugin SDK الحديثة ذلك: فكل مسار استيراد (`openclaw/plugin-sdk/\<subpath\>`)
هو وحدة صغيرة مستقلة ذات غرض واضح وعقد موثق.

كما أزيلت أيضًا طبقات الراحة القديمة الخاصة بالموفّرين للقنوات المدمجة. فعمليات الاستيراد
مثل `openclaw/plugin-sdk/slack` و`openclaw/plugin-sdk/discord`،
و`openclaw/plugin-sdk/signal` و`openclaw/plugin-sdk/whatsapp`،
وطبقات المساعدة الموسومة بالقناة، و
`openclaw/plugin-sdk/telegram-core` كانت اختصارات خاصة بمستودع mono وليست
عقودًا مستقرة للإضافات. استخدم بدلًا منها subpaths عامة ضيقة في SDK. وداخل
workspace الخاصة بالإضافة المدمجة، احتفظ بالمساعدات المملوكة للموفّر داخل
`api.ts` أو `runtime-api.ts` الخاصين بتلك الإضافة.

أمثلة الموفّرين المدمجين الحالية:

- يحتفظ Anthropic بمساعدات البث الخاصة بـ Claude داخل طبقة `api.ts` /
  `contract-api.ts` الخاصة به
- يحتفظ OpenAI ببناة الموفّرين، ومساعدات النماذج الافتراضية، وبناة الموفّرين
  الفورية داخل `api.ts` الخاص به
- يحتفظ OpenRouter بباني الموفّر ومساعدات onboarding/config داخل
  `api.ts` الخاص به

## كيفية الترحيل

<Steps>
  <Step title="مراجعة سلوك الرجوع الاحتياطي لـ Windows wrapper">
    إذا كانت إضافتك تستخدم `openclaw/plugin-sdk/windows-spawn`، فإن
    أغلفة Windows ‏`.cmd`/`.bat` غير المحلولة تفشل الآن بشكل مغلق ما لم تمرر
    `allowShellFallback: true` صراحة.

    ```typescript
    // قبل
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // بعد
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // اضبط هذا فقط للمستدعين الموثوقين المتوافقين الذين
      // يقبلون عمدًا الرجوع الاحتياطي عبر shell.
      allowShellFallback: true,
    });
    ```

    إذا لم يكن المستدعي لديك يعتمد عمدًا على الرجوع الاحتياطي عبر shell، فلا تضبط
    `allowShellFallback` وتعامل مع الخطأ المرفوع بدلًا من ذلك.

  </Step>

  <Step title="البحث عن عمليات الاستيراد المهملة">
    ابحث في إضافتك عن عمليات الاستيراد من أي من السطحين المهملين:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="الاستبدال بعمليات استيراد مركزة">
    يقابل كل تصدير من السطح القديم مسار استيراد حديثًا محددًا:

    ```typescript
    // قبل (طبقة التوافق العكسي المهملة)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // بعد (عمليات استيراد حديثة ومركزة)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    بالنسبة إلى المساعدات الموجودة جهة المضيف، استخدم plugin runtime المحقون بدلًا من الاستيراد
    المباشر:

    ```typescript
    // قبل (جسر extension-api المهمل)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // بعد (runtime محقونة)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    ينطبق النمط نفسه على مساعدات الجسر القديمة الأخرى:

    | الاستيراد القديم | المعادل الحديث |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | مساعدات مخزن الجلسة | `api.runtime.agent.session.*` |

  </Step>

  <Step title="البناء والاختبار">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## مرجع مسارات الاستيراد

<Accordion title="جدول مسارات الاستيراد الشائعة">
  | مسار الاستيراد | الغرض | التصديرات الأساسية |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | مساعد إدخال الإضافة القانوني | `definePluginEntry` |
  | `plugin-sdk/core` | إعادة تصدير جامعة قديمة لتعريفات/بناة إدخال القنوات | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | تصدير مخطط الإعداد الجذري | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | مساعد إدخال موفّر واحد | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | تعريفات وبناة إدخال القنوات المركزة | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | مساعدات setup المشتركة | مطالبات قائمة السماح، وبناة حالة setup |
  | `plugin-sdk/setup-runtime` | مساعدات runtime في وقت setup | مهايئات تصحيح setup الآمنة للاستيراد، ومساعدات ملاحظات lookup، و`promptResolvedAllowFrom`، و`splitSetupEntries`، ووكلاء setup المفوضون |
  | `plugin-sdk/setup-adapter-runtime` | مساعدات مهايئات setup | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | مساعدات أدوات setup | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | مساعدات تعدد الحسابات | مساعدات قوائم/إعدادات/بوابات أفعال الحساب |
  | `plugin-sdk/account-id` | مساعدات account-id | `DEFAULT_ACCOUNT_ID`، وتطبيع account-id |
  | `plugin-sdk/account-resolution` | مساعدات lookup للحساب | مساعدات lookup للحساب + fallback الافتراضي |
  | `plugin-sdk/account-helpers` | مساعدات حساب ضيقة | مساعدات قائمة الحساب/إجراء الحساب |
  | `plugin-sdk/channel-setup` | مهايئات معالج setup | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`، بالإضافة إلى `DEFAULT_ACCOUNT_ID` و`createTopLevelChannelDmPolicy` و`setSetupChannelEnabled` و`splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | بدائيات اقتران الرسائل المباشرة | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | ربط بادئة الرد + مؤشرات الكتابة | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | مصانع مهايئات الإعداد | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | بناة مخطط الإعداد | أنواع مخطط إعداد القناة |
  | `plugin-sdk/telegram-command-config` | مساعدات إعداد أوامر Telegram | تطبيع أسماء الأوامر، واقتطاع الوصف، والتحقق من التكرار/التعارض |
  | `plugin-sdk/channel-policy` | حل سياسات المجموعة/الرسائل المباشرة | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | تتبع حالة الحساب | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | مساعدات المغلف الوارد | مساعدات route + builder للمغلف المشتركة |
  | `plugin-sdk/inbound-reply-dispatch` | مساعدات الرد الوارد | مساعدات record-and-dispatch المشتركة |
  | `plugin-sdk/messaging-targets` | تحليل أهداف المراسلة | مساعدات تحليل/مطابقة الأهداف |
  | `plugin-sdk/outbound-media` | مساعدات الوسائط الصادرة | تحميل الوسائط الصادرة المشتركة |
  | `plugin-sdk/outbound-runtime` | مساعدات runtime الصادرة | مساعدات هوية/تفويض إرسال صادر |
  | `plugin-sdk/thread-bindings-runtime` | مساعدات ربط السلاسل | دورة حياة ربط السلاسل ومساعدات المهايئ |
  | `plugin-sdk/agent-media-payload` | مساعدات حمولة الوسائط القديمة | منشئ حمولة وسائط الوكيل لتخطيطات الحقول القديمة |
  | `plugin-sdk/channel-runtime` | طبقة توافق مهملة | أدوات runtime قديمة للقناة فقط |
  | `plugin-sdk/channel-send-result` | أنواع نتائج الإرسال | أنواع نتائج الرد |
  | `plugin-sdk/runtime-store` | تخزين الإضافات الدائم | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | مساعدات runtime واسعة | مساعدات runtime/logging/backup/plugin-install |
  | `plugin-sdk/runtime-env` | مساعدات بيئة runtime ضيقة | logger/runtime env، والمهلة، وإعادة المحاولة، ومساعدات backoff |
  | `plugin-sdk/plugin-runtime` | مساعدات runtime مشتركة للإضافة | مساعدات أوامر/hooks/http/تفاعلية للإضافة |
  | `plugin-sdk/hook-runtime` | مساعدات مسار hook | مساعدات pipeline مشتركة لـ webhook/internal hook |
  | `plugin-sdk/lazy-runtime` | مساعدات runtime الكسولة | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | مساعدات العمليات | مساعدات exec المشتركة |
  | `plugin-sdk/cli-runtime` | مساعدات runtime لـ CLI | تنسيق الأوامر، والانتظار، ومساعدات الإصدار |
  | `plugin-sdk/gateway-runtime` | مساعدات Gateway | عميل Gateway ومساعدات تصحيح حالة القناة |
  | `plugin-sdk/config-runtime` | مساعدات الإعداد | مساعدات تحميل/كتابة الإعداد |
  | `plugin-sdk/telegram-command-config` | مساعدات أوامر Telegram | مساعدات تحقق أوامر Telegram الثابتة في fallback عندما يكون سطح عقد Telegram المدمج غير متاح |
  | `plugin-sdk/approval-runtime` | مساعدات مطالبات الموافقة | حمولة موافقة exec/plugin، ومساعدات قدرة/ملف تعريف الموافقة، ومساعدات التوجيه/runtime الأصلية للموافقة |
  | `plugin-sdk/approval-auth-runtime` | مساعدات مصادقة الموافقة | حل الموافقين، ومصادقة الإجراء في الدردشة نفسها |
  | `plugin-sdk/approval-client-runtime` | مساعدات عميل الموافقة | مساعدات ملف تعريف/مرشح موافقة exec الأصلية |
  | `plugin-sdk/approval-delivery-runtime` | مساعدات تسليم الموافقة | مهايئات قدرة/تسليم الموافقة الأصلية |
  | `plugin-sdk/approval-native-runtime` | مساعدات هدف الموافقة | مساعدات ربط هدف/حساب الموافقة الأصلية |
  | `plugin-sdk/approval-reply-runtime` | مساعدات رد الموافقة | مساعدات حمولة رد موافقة exec/plugin |
  | `plugin-sdk/security-runtime` | مساعدات الأمان | مساعدات الثقة المشتركة، وبوابة الرسائل المباشرة، والمحتوى الخارجي، وجمع الأسرار |
  | `plugin-sdk/ssrf-policy` | مساعدات سياسة SSRF | قائمة سماح المضيف ومساعدات سياسة الشبكة الخاصة |
  | `plugin-sdk/ssrf-runtime` | مساعدات runtime لـ SSRF | مساعدات pinned-dispatcher وguarded fetch وسياسة SSRF |
  | `plugin-sdk/collection-runtime` | مساعدات bounded cache | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | مساعدات بوابة التشخيص | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | مساعدات تنسيق الأخطاء | `formatUncaughtError`, `isApprovalNotFoundError`، ومساعدات رسم الأخطاء |
  | `plugin-sdk/fetch-runtime` | مساعدات fetch/proxy المغلفة | `resolveFetch`، ومساعدات proxy |
  | `plugin-sdk/host-runtime` | مساعدات تطبيع المضيف | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | مساعدات إعادة المحاولة | `RetryConfig`, `retryAsync`, مشغلات السياسات |
  | `plugin-sdk/allow-from` | تنسيق قائمة السماح | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | تعيين إدخالات قائمة السماح | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | بوابة الأوامر ومساعدات سطح الأوامر | `resolveControlCommandGate`، ومساعدات تفويض المرسل، ومساعدات سجل الأوامر |
  | `plugin-sdk/secret-input` | تحليل إدخال السر | مساعدات إدخال الأسرار |
  | `plugin-sdk/webhook-ingress` | مساعدات طلب webhook | أدوات هدف webhook |
  | `plugin-sdk/webhook-request-guards` | مساعدات حراسة جسم webhook | مساعدات قراءة/تحديد حجم جسم الطلب |
  | `plugin-sdk/reply-runtime` | runtime رد مشتركة | dispatch وارد، heartbeat، مخطط الرد، التجزئة |
  | `plugin-sdk/reply-dispatch-runtime` | مساعدات dispatch للرد الضيقة | مساعدات finalize + dispatch الخاصة بالموفّر |
  | `plugin-sdk/reply-history` | مساعدات سجل الرد | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | تخطيط مرجع الرد | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | مساعدات تجزئة الرد | مساعدات تجزئة النص/Markdown |
  | `plugin-sdk/session-store-runtime` | مساعدات مخزن الجلسة | مساعدات مسار المخزن + updated-at |
  | `plugin-sdk/state-paths` | مساعدات مسارات الحالة | مساعدات دليل الحالة وOAuth |
  | `plugin-sdk/routing` | مساعدات routing/session-key | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`، ومساعدات تطبيع session-key |
  | `plugin-sdk/status-helpers` | مساعدات حالة القناة | بناة ملخص/لقطة حالة القناة/الحساب، والقيم الافتراضية لـ runtime-state، ومساعدات بيانات وصفية للمشكلات |
  | `plugin-sdk/target-resolver-runtime` | مساعدات حل الهدف | مساعدات حل الهدف المشتركة |
  | `plugin-sdk/string-normalization-runtime` | مساعدات تطبيع السلاسل | مساعدات تطبيع slug/string |
  | `plugin-sdk/request-url` | مساعدات عنوان URL للطلب | استخراج عناوين URL النصية من مدخلات تشبه الطلب |
  | `plugin-sdk/run-command` | مساعدات الأوامر ذات المهلة | مشغّل أوامر ذات مهلة مع stdout/stderr مُطبّعين |
  | `plugin-sdk/param-readers` | قارئات المعلمات | قارئات معلمات شائعة للأداة/CLI |
  | `plugin-sdk/tool-send` | استخراج إرسال الأداة | استخراج حقول هدف الإرسال القانونية من وسائط الأداة |
  | `plugin-sdk/temp-path` | مساعدات المسارات المؤقتة | مساعدات مسار التنزيل المؤقت المشتركة |
  | `plugin-sdk/logging-core` | مساعدات التسجيل | subsystem logger ومساعدات الحجب |
  | `plugin-sdk/markdown-table-runtime` | مساعدات جداول Markdown | مساعدات وضع جدول Markdown |
  | `plugin-sdk/reply-payload` | أنواع رد الرسائل | أنواع حمولة الرد |
  | `plugin-sdk/provider-setup` | مساعدات setup منسقة للموفّر المحلي/المستضاف ذاتيًا | مساعدات اكتشاف/تهيئة الموفّر المستضاف ذاتيًا |
  | `plugin-sdk/self-hosted-provider-setup` | مساعدات setup مركزة للموفّر المستضاف ذاتيًا والمتوافق مع OpenAI | مساعدات اكتشاف/تهيئة الموفّر المستضاف ذاتيًا نفسها |
  | `plugin-sdk/provider-auth-runtime` | مساعدات مصادقة runtime للموفّر | مساعدات حل API-key في وقت التشغيل |
  | `plugin-sdk/provider-auth-api-key` | مساعدات setup لمفتاح API للموفّر | مساعدات onboarding/كتابة profile لمفتاح API |
  | `plugin-sdk/provider-auth-result` | مساعدات auth-result للموفّر | منشئ OAuth auth-result القياسي |
  | `plugin-sdk/provider-auth-login` | مساعدات login التفاعلية للموفّر | مساعدات login التفاعلي المشتركة |
  | `plugin-sdk/provider-env-vars` | مساعدات env-vars للموفّر | مساعدات lookup لمتغيرات env الخاصة بمصادقة الموفّر |
  | `plugin-sdk/provider-model-shared` | مساعدات model/replay مشتركة للموفّر | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`، وبناة سياسة replay المشتركة، ومساعدات نقطة نهاية الموفّر، ومساعدات تطبيع معرّف النموذج |
  | `plugin-sdk/provider-catalog-shared` | مساعدات فهرس مشتركة للموفّر | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | تصحيحات onboarding للموفّر | مساعدات إعداد onboarding |
  | `plugin-sdk/provider-http` | مساعدات HTTP للموفّر | مساعدات HTTP/قدرات نقطة النهاية العامة للموفّر |
  | `plugin-sdk/provider-web-fetch` | مساعدات web-fetch للموفّر | مساعدات تسجيل/ذاكرة مؤقتة لموفّر web-fetch |
  | `plugin-sdk/provider-web-search` | مساعدات web-search للموفّر | مساعدات تسجيل/ذاكرة مؤقتة/إعداد لموفّر web-search |
  | `plugin-sdk/provider-tools` | مساعدات توافق أداة/مخطط الموفّر | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`، وتنظيف مخطط Gemini + التشخيصات، ومساعدات توافق xAI مثل `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | مساعدات استخدام الموفّر | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage`، ومساعدات استخدام موفّرين أخرى |
  | `plugin-sdk/provider-stream` | مساعدات غلاف stream للموفّر | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`، وأنواع stream wrapper، ومساعدات wrapper المشتركة لـ Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | صف async مرتب | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | مساعدات وسائط مشتركة | مساعدات fetch/transform/store للوسائط بالإضافة إلى بناة حمولة الوسائط |
  | `plugin-sdk/media-understanding` | مساعدات فهم الوسائط | أنواع موفّري فهم الوسائط بالإضافة إلى تصديرات مساعدات الصورة/الصوت المواجهة للموفّر |
  | `plugin-sdk/text-runtime` | مساعدات نص مشتركة | إزالة النص المرئي للمساعد، ومساعدات render/chunking/table الخاصة بـ Markdown، ومساعدات الحجب، ومساعدات directive-tag، وأدوات النص الآمن، ومساعدات النص/التسجيل ذات الصلة |
  | `plugin-sdk/text-chunking` | مساعدات تجزئة النص | مساعد تجزئة النص الصادر |
  | `plugin-sdk/speech` | مساعدات Speech | أنواع موفّري الكلام بالإضافة إلى التصديرات المواجهة للموفّر الخاصة بـ directive وregistry والتحقق |
  | `plugin-sdk/speech-core` | Speech core مشتركة | أنواع موفّري الكلام، والسجل، وdirectives، والتطبيع |
  | `plugin-sdk/realtime-transcription` | مساعدات النسخ الفوري | أنواع الموفّرين ومساعدات السجل |
  | `plugin-sdk/realtime-voice` | مساعدات الصوت الفوري | أنواع الموفّرين ومساعدات السجل |
  | `plugin-sdk/image-generation-core` | core مشتركة لإنشاء الصور | أنواع إنشاء الصور، والتراجع الاحتياطي، والمصادقة، ومساعدات السجل |
  | `plugin-sdk/video-generation` | مساعدات إنشاء الفيديو | أنواع موفّر/طلب/نتيجة إنشاء الفيديو |
  | `plugin-sdk/video-generation-core` | core مشتركة لإنشاء الفيديو | أنواع إنشاء الفيديو، ومساعدات التراجع الاحتياطي، وlookup للموفّر، وتحليل model-ref |
  | `plugin-sdk/interactive-runtime` | مساعدات الرد التفاعلي | تطبيع/تقليص حمولة الرد التفاعلي |
  | `plugin-sdk/channel-config-primitives` | بدائيات إعداد القناة | بدائيات ضيقة لمخطط إعداد القناة |
  | `plugin-sdk/channel-config-writes` | مساعدات كتابة إعداد القناة | مساعدات تفويض كتابة إعداد القناة |
  | `plugin-sdk/channel-plugin-common` | prelude مشتركة للقناة | تصديرات prelude مشتركة لإضافة القناة |
  | `plugin-sdk/channel-status` | مساعدات حالة القناة | مساعدات snapshot/summary المشتركة لحالة القناة |
  | `plugin-sdk/allowlist-config-edit` | مساعدات إعداد allowlist | مساعدات تعديل/قراءة إعداد allowlist |
  | `plugin-sdk/group-access` | مساعدات وصول المجموعة | مساعدات قرار وصول المجموعة المشتركة |
  | `plugin-sdk/direct-dm` | مساعدات الرسائل المباشرة المباشرة | مساعدات مصادقة/حراسة الرسائل المباشرة المشتركة |
  | `plugin-sdk/extension-shared` | مساعدات extension مشتركة | بدائيات مساعدة لحالة/قناة سلبية |
  | `plugin-sdk/webhook-targets` | مساعدات أهداف webhook | سجل أهداف webhook ومساعدات تثبيت المسارات |
  | `plugin-sdk/webhook-path` | مساعدات مسار webhook | مساعدات تطبيع مسار webhook |
  | `plugin-sdk/web-media` | مساعدات وسائط ويب مشتركة | مساعدات تحميل الوسائط البعيدة/المحلية |
  | `plugin-sdk/zod` | إعادة تصدير Zod | إعادة تصدير `zod` لمستهلكي Plugin SDK |
  | `plugin-sdk/memory-core` | مساعدات memory-core المدمجة | سطح مساعدات مدير/إعداد/ملف/CLI الخاص بالذاكرة |
  | `plugin-sdk/memory-core-engine-runtime` | واجهة runtime لمحرك الذاكرة | واجهة runtime للفهرسة/البحث في الذاكرة |
  | `plugin-sdk/memory-core-host-engine-foundation` | محرك الأساس للمضيف الخاص بالذاكرة | تصديرات محرك الأساس للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-engine-embeddings` | محرك embeddings للمضيف الخاص بالذاكرة | تصديرات محرك embeddings للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-engine-qmd` | محرك QMD للمضيف الخاص بالذاكرة | تصديرات محرك QMD للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-engine-storage` | محرك التخزين للمضيف الخاص بالذاكرة | تصديرات محرك التخزين للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-multimodal` | مساعدات الوسائط المتعددة للمضيف الخاص بالذاكرة | مساعدات الوسائط المتعددة للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-query` | مساعدات الاستعلام للمضيف الخاص بالذاكرة | مساعدات الاستعلام للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-secret` | مساعدات الأسرار للمضيف الخاص بالذاكرة | مساعدات الأسرار للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-status` | مساعدات الحالة للمضيف الخاص بالذاكرة | مساعدات الحالة للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-runtime-cli` | runtime CLI للمضيف الخاص بالذاكرة | مساعدات runtime CLI للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-runtime-core` | runtime core للمضيف الخاص بالذاكرة | مساعدات runtime core للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-core-host-runtime-files` | مساعدات الملفات/runtime للمضيف الخاص بالذاكرة | مساعدات الملفات/runtime للمضيف الخاص بالذاكرة |
  | `plugin-sdk/memory-lancedb` | مساعدات memory-lancedb المدمجة | سطح مساعدات Memory-lancedb |
  | `plugin-sdk/testing` | أدوات الاختبار | مساعدات الاختبار وmocks |
</Accordion>

يمثل هذا الجدول عمدًا مجموعة الترحيل الشائعة، وليس السطح الكامل لـ SDK.
وتوجد القائمة الكاملة التي تضم أكثر من 200 نقطة دخول في
`scripts/lib/plugin-sdk-entrypoints.json`.

ولا تزال هذه القائمة تتضمن بعض طبقات المساعدة الخاصة بالإضافات المدمجة مثل
`plugin-sdk/feishu` و`plugin-sdk/feishu-setup` و`plugin-sdk/zalo`،
و`plugin-sdk/zalo-setup` و`plugin-sdk/matrix*`. وتظل هذه الطبقات مُصدَّرة من أجل
صيانة الإضافات المدمجة والتوافق، لكنها أُزيلت عمدًا
من جدول الترحيل الشائع وليست الهدف الموصى به
لشيفرة الإضافات الجديدة.

وتنطبق القاعدة نفسها على عائلات المساعدات المدمجة الأخرى مثل:

- مساعدات دعم المتصفح: `plugin-sdk/browser-config-support`, `plugin-sdk/browser-support`
- Matrix: ‏`plugin-sdk/matrix*`
- LINE: ‏`plugin-sdk/line*`
- IRC: ‏`plugin-sdk/irc*`
- طبقات المساعدة/الإضافات المدمجة مثل `plugin-sdk/googlechat`,
  و`plugin-sdk/zalouser`، و`plugin-sdk/bluebubbles*`,
  و`plugin-sdk/mattermost*`، و`plugin-sdk/msteams`,
  و`plugin-sdk/nextcloud-talk`، و`plugin-sdk/nostr`، و`plugin-sdk/tlon`,
  و`plugin-sdk/twitch`,
  و`plugin-sdk/github-copilot-login`، و`plugin-sdk/github-copilot-token`,
  و`plugin-sdk/diagnostics-otel`، و`plugin-sdk/diffs`، و`plugin-sdk/llm-task`,
  و`plugin-sdk/thread-ownership`، و`plugin-sdk/voice-call`

يكشف `plugin-sdk/github-copilot-token` حاليًا سطح المساعدة الضيق الخاص بالرمز
`DEFAULT_COPILOT_API_BASE_URL`،
و`deriveCopilotApiBaseUrlFromToken`، و`resolveCopilotApiToken`.

استخدم أضيق عملية استيراد تطابق المهمة. وإذا لم تتمكن من العثور على تصدير،
فراجع المصدر في `src/plugin-sdk/` أو اسأل على Discord.

## الجدول الزمني للإزالة

| متى | ماذا يحدث |
| ---------------------- | ----------------------------------------------------------------------- |
| **الآن** | تصدر الواجهات المهملة تحذيرات في وقت التشغيل |
| **الإصدار الرئيسي التالي** | ستتم إزالة الواجهات المهملة؛ وستفشل الإضافات التي ما زالت تستخدمها |

تم بالفعل ترحيل كل الإضافات الأساسية. وينبغي أن تُرحّل الإضافات الخارجية
قبل الإصدار الرئيسي التالي.

## كتم التحذيرات مؤقتًا

اضبط متغيرات البيئة هذه أثناء العمل على الترحيل:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

هذا منفذ هروب مؤقت، وليس حلًا دائمًا.

## ذو صلة

- [البدء](/plugins/building-plugins) — ابنِ أول إضافة لك
- [نظرة عامة على SDK](/plugins/sdk-overview) — المرجع الكامل لاستيرادات subpath
- [Channel Plugins](/plugins/sdk-channel-plugins) — بناء إضافات القنوات
- [Provider Plugins](/plugins/sdk-provider-plugins) — بناء إضافات الموفّر
- [البنية الداخلية للإضافة](/plugins/architecture) — تعمق في المعمارية
- [Plugin Manifest](/plugins/manifest) — مرجع مخطط manifest
