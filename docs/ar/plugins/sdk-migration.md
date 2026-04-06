---
read_when:
    - تظهر لك رسالة التحذير OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - تظهر لك رسالة التحذير OPENCLAW_EXTENSION_API_DEPRECATED
    - أنت تحدّث إضافة إلى بنية الإضافات الحديثة
    - أنت مسؤول عن إضافة OpenClaw خارجية
sidebarTitle: Migrate to SDK
summary: الانتقال من طبقة التوافق مع الإصدارات السابقة القديمة إلى Plugin SDK الحديثة
title: ترحيل Plugin SDK
x-i18n:
    generated_at: "2026-04-06T03:10:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: b71ce69b30c3bb02da1b263b1d11dc3214deae5f6fc708515e23b5a1c7bb7c8f
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# ترحيل Plugin SDK

انتقل OpenClaw من طبقة توافق واسعة مع الإصدارات السابقة إلى بنية إضافات
حديثة تعتمد على عمليات استيراد مركزة وموثقة. إذا كانت إضافتك قد بُنيت قبل
البنية الجديدة، فسيساعدك هذا الدليل على ترحيلها.

## ما الذي يتغير

كان نظام الإضافات القديم يوفّر سطحين واسعين يتيحان للإضافات استيراد
أي شيء تحتاج إليه من نقطة دخول واحدة:

- **`openclaw/plugin-sdk/compat`** — استيراد واحد يعيد تصدير عشرات
  الأدوات المساعدة. وقد أُضيف هذا للحفاظ على عمل الإضافات القديمة المعتمدة على الخطافات
  أثناء بناء بنية الإضافات الجديدة.
- **`openclaw/extension-api`** — جسر أتاح للإضافات الوصول المباشر إلى
  الأدوات المساعدة على جانب المضيف مثل مشغّل الوكيل المضمن.

هاتان الواجهتان الآن **مهجورتان**. وما زالتا تعملان في وقت التشغيل، لكن يجب ألا تستخدمهما
الإضافات الجديدة، كما ينبغي للإضافات الحالية الترحيل قبل أن يزيلهما
الإصدار الرئيسي التالي.

<Warning>
  ستتم إزالة طبقة التوافق مع الإصدارات السابقة في إصدار رئيسي مستقبلي.
  الإضافات التي لا تزال تستورد من هذه الواجهات ستتعطل عند حدوث ذلك.
</Warning>

## لماذا تغير هذا

أدى النهج القديم إلى مشكلات:

- **بدء تشغيل بطيء** — كان استيراد أداة مساعدة واحدة يحمّل عشرات الوحدات غير المرتبطة
- **تبعيات دائرية** — جعلت إعادة التصدير الواسعة من السهل إنشاء دورات استيراد
- **سطح API غير واضح** — لم تكن هناك طريقة لمعرفة أي التصديرات مستقرة وأيها داخلية

تعالج Plugin SDK الحديثة ذلك: فكل مسار استيراد (`openclaw/plugin-sdk/\<subpath\>`)
هو وحدة صغيرة مستقلة ذات غرض واضح وعقد موثق.

كما أزيلت أيضًا طبقات الراحة القديمة الخاصة بالمزودين للقنوات المضمنة. إن عمليات الاستيراد
مثل `openclaw/plugin-sdk/slack` و`openclaw/plugin-sdk/discord`
و`openclaw/plugin-sdk/signal` و`openclaw/plugin-sdk/whatsapp`
ومسارات الأدوات المساعدة ذات العلامة التجارية للقنوات، و
`openclaw/plugin-sdk/telegram-core` كانت اختصارات خاصة بالمستودع الأحادي، وليست
عقود إضافات مستقرة. استخدم بدلًا من ذلك المسارات الفرعية العامة والضيقة في SDK. داخل
مساحة عمل الإضافات المضمنة، احتفظ بالأدوات المساعدة المملوكة للمزود في
`api.ts` أو `runtime-api.ts` الخاصين بتلك الإضافة.

أمثلة المزودين المضمنين الحالية:

- يحتفظ Anthropic بأدوات Claude المساعدة الخاصة بالبث في `api.ts` /
  `contract-api.ts` الخاص به
- يحتفظ OpenAI ببناة المزودين، وأدوات النموذج الافتراضي، وبناة مزود
  realtime في `api.ts` الخاص به
- يحتفظ OpenRouter بباني المزود وأدوات الإعداد الأولي/الإعدادات المساعدة في
  `api.ts` الخاص به

## كيفية الترحيل

<Steps>
  <Step title="راجع سلوك الرجوع الاحتياطي لغلاف Windows">
    إذا كانت إضافتك تستخدم `openclaw/plugin-sdk/windows-spawn`، فإن أغلفة Windows
    `.cmd`/`.bat` غير المحلولة ستفشل الآن بشكل مغلق ما لم تمرر صراحةً
    `allowShellFallback: true`.

    ```typescript
    // قبل
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // بعد
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // اضبط هذا فقط للجهات المتوافقة الموثوقة التي تقبل عمدًا
      // الرجوع الاحتياطي عبر shell.
      allowShellFallback: true,
    });
    ```

    إذا لم يكن المستدعي يعتمد عمدًا على الرجوع الاحتياطي عبر shell، فلا تضبط
    `allowShellFallback` وتعامل مع الخطأ المُلقى بدلًا من ذلك.

  </Step>

  <Step title="اعثر على عمليات الاستيراد المهجورة">
    ابحث في إضافتك عن عمليات الاستيراد من أي من السطحين المهجورين:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="استبدلها بعمليات استيراد مركزة">
    كل تصدير من السطح القديم يقابله مسار استيراد حديث محدد:

    ```typescript
    // قبل (طبقة التوافق مع الإصدارات السابقة المهجورة)
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

    بالنسبة إلى الأدوات المساعدة على جانب المضيف، استخدم وقت تشغيل الإضافة المحقون بدلًا من الاستيراد
    المباشر:

    ```typescript
    // قبل (جسر extension-api المهجور)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // بعد (وقت تشغيل محقون)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    ينطبق النمط نفسه على أدوات الجسر القديمة الأخرى:

    | الاستيراد القديم | البديل الحديث |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | أدوات مساعدات مخزن الجلسة | `api.runtime.agent.session.*` |

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
  | مسار الاستيراد | الغرض | التصديرات الأساسية |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | أداة مساعد إدخال الإضافة القياسية | `definePluginEntry` |
  | `plugin-sdk/core` | إعادة تصدير شاملة قديمة لتعريفات/بناة إدخال القنوات | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | تصدير مخطط إعدادات الجذر | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | أداة مساعد إدخال مزود واحد | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | تعريفات وبناة إدخال القنوات المركزة | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | أدوات مساعدة مشتركة لمعالج الإعداد | مطالبات قائمة السماح، وبناة حالة الإعداد |
  | `plugin-sdk/setup-runtime` | أدوات مساعدة لوقت تشغيل الإعداد | مهايئات تصحيح الإعدادات الآمنة للاستيراد، وأدوات lookup-note المساعدة، و`promptResolvedAllowFrom`، و`splitSetupEntries`، ووكلاء الإعداد المفوض |
  | `plugin-sdk/setup-adapter-runtime` | أدوات مساعدة لمهايئ الإعداد | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | أدوات مساعدة لأدوات الإعداد | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | أدوات مساعدة للحسابات المتعددة | أدوات مساعدة لقائمة الحسابات/الإعدادات/بوابات الإجراءات |
  | `plugin-sdk/account-id` | أدوات مساعدة لمعرّف الحساب | `DEFAULT_ACCOUNT_ID`، وتطبيع معرّف الحساب |
  | `plugin-sdk/account-resolution` | أدوات مساعدة للبحث عن الحساب | أدوات مساعدة للبحث عن الحساب + الرجوع الافتراضي |
  | `plugin-sdk/account-helpers` | أدوات مساعدة ضيقة للحساب | أدوات مساعدة لقائمة الحسابات/إجراءات الحساب |
  | `plugin-sdk/channel-setup` | مهايئات معالج الإعداد | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`، بالإضافة إلى `DEFAULT_ACCOUNT_ID` و`createTopLevelChannelDmPolicy` و`setSetupChannelEnabled` و`splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | بدائيات إقران DM | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | ربط بادئة الرد + الكتابة | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | مصانع مهايئات الإعدادات | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | بناة مخطط الإعدادات | أنواع مخطط إعدادات القناة |
  | `plugin-sdk/telegram-command-config` | أدوات مساعدة لإعداد أوامر Telegram | تطبيع أسماء الأوامر، تقليم الوصف، التحقق من التكرار/التعارض |
  | `plugin-sdk/channel-policy` | حل سياسة المجموعة/DM | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | تتبع حالة الحساب | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | أدوات مساعدة للمغلف الوارد | أدوات مساعدة مشتركة لبناء route + envelope |
  | `plugin-sdk/inbound-reply-dispatch` | أدوات مساعدة للردود الواردة | أدوات مساعدة مشتركة للتسجيل والإرسال |
  | `plugin-sdk/messaging-targets` | تحليل أهداف المراسلة | أدوات مساعدة لتحليل/مطابقة الأهداف |
  | `plugin-sdk/outbound-media` | أدوات مساعدة للوسائط الصادرة | تحميل وسائط صادرة مشتركة |
  | `plugin-sdk/outbound-runtime` | أدوات مساعدة لوقت التشغيل الصادر | أدوات مساعدة لهوية الإرسال/تفويض الإرسال |
  | `plugin-sdk/thread-bindings-runtime` | أدوات مساعدة لربط الخيوط | دورة حياة ربط الخيوط وأدوات المهايئ المساعدة |
  | `plugin-sdk/agent-media-payload` | أدوات مساعدة قديمة لحمولات الوسائط | باني حمولة وسائط الوكيل لتخطيطات الحقول القديمة |
  | `plugin-sdk/channel-runtime` | Shim توافق مهجور | أدوات وقت تشغيل القناة القديمة فقط |
  | `plugin-sdk/channel-send-result` | أنواع نتائج الإرسال | أنواع نتائج الرد |
  | `plugin-sdk/runtime-store` | تخزين الإضافات الدائم | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | أدوات مساعدة واسعة لوقت التشغيل | أدوات مساعدة لوقت التشغيل/التسجيل/النسخ الاحتياطي/تثبيت الإضافات |
  | `plugin-sdk/runtime-env` | أدوات مساعدة ضيقة لبيئة وقت التشغيل | أدوات مساعدة للمسجل/بيئة وقت التشغيل، والمهلة، وإعادة المحاولة، والتراجع التدريجي |
  | `plugin-sdk/plugin-runtime` | أدوات مساعدة مشتركة لوقت تشغيل الإضافة | أدوات مساعدة للأوامر/الخطافات/http/التفاعل الخاصة بالإضافة |
  | `plugin-sdk/hook-runtime` | أدوات مساعدة لمسار الخطافات | أدوات مساعدة مشتركة لمسار webhook/الخطافات الداخلية |
  | `plugin-sdk/lazy-runtime` | أدوات مساعدة لوقت التشغيل الكسول | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | أدوات مساعدة للعمليات | أدوات مساعدة مشتركة لـ exec |
  | `plugin-sdk/cli-runtime` | أدوات مساعدة لوقت تشغيل CLI | تنسيق الأوامر، والانتظار، وأدوات إصدار المساعدة |
  | `plugin-sdk/gateway-runtime` | أدوات مساعدة للبوابة | عميل البوابة وأدوات تصحيح حالة القناة |
  | `plugin-sdk/config-runtime` | أدوات مساعدة للإعدادات | أدوات مساعدة لتحميل/كتابة الإعدادات |
  | `plugin-sdk/telegram-command-config` | أدوات مساعدة لأوامر Telegram | أدوات مساعدة للتحقق من أوامر Telegram الثابتة عند الرجوع عندما لا يكون سطح عقد Telegram المضمن متاحًا |
  | `plugin-sdk/approval-runtime` | أدوات مساعدة لمطالبة الموافقة | حمولة موافقة exec/plugin، وأدوات مساعدة لقدرات/ملفات الموافقة، وأدوات مساعدة للتوجيه/وقت التشغيل للموافقة الأصلية |
  | `plugin-sdk/approval-auth-runtime` | أدوات مساعدة لمصادقة الموافقة | حل المعتمدين، ومصادقة الإجراء داخل المحادثة نفسها |
  | `plugin-sdk/approval-client-runtime` | أدوات مساعدة لعميل الموافقة | أدوات مساعدة لملف/مرشح الموافقة الأصلية على exec |
  | `plugin-sdk/approval-delivery-runtime` | أدوات مساعدة لتسليم الموافقة | مهايئات قدرة/تسليم الموافقة الأصلية |
  | `plugin-sdk/approval-native-runtime` | أدوات مساعدة لهدف الموافقة | أدوات مساعدة لربط هدف/حساب الموافقة الأصلية |
  | `plugin-sdk/approval-reply-runtime` | أدوات مساعدة لرد الموافقة | أدوات مساعدة لحمولة رد موافقة exec/plugin |
  | `plugin-sdk/security-runtime` | أدوات مساعدة للأمان | أدوات مساعدة مشتركة للثقة، وبوابة DM، والمحتوى الخارجي، وجمع الأسرار |
  | `plugin-sdk/ssrf-policy` | أدوات مساعدة لسياسة SSRF | أدوات مساعدة لقائمة سماح المضيف وسياسة الشبكة الخاصة |
  | `plugin-sdk/ssrf-runtime` | أدوات مساعدة لوقت تشغيل SSRF | pinned-dispatcher، وguarded fetch، وأدوات مساعدة لسياسة SSRF |
  | `plugin-sdk/collection-runtime` | أدوات مساعدة لذاكرة التخزين المحدودة | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | أدوات مساعدة لبوابة التشخيص | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | أدوات مساعدة لتنسيق الأخطاء | `formatUncaughtError`, `isApprovalNotFoundError`، وأدوات مساعدة لرسم الأخطاء |
  | `plugin-sdk/fetch-runtime` | أدوات مساعدة لـ fetch/proxy المغلفة | `resolveFetch`، وأدوات proxy المساعدة |
  | `plugin-sdk/host-runtime` | أدوات مساعدة لتطبيع المضيف | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | أدوات مساعدة لإعادة المحاولة | `RetryConfig`, `retryAsync`، ومنفذو السياسات |
  | `plugin-sdk/allow-from` | تنسيق قائمة السماح | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | ربط مدخلات قائمة السماح | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | بوابة الأوامر وأدوات مساعدة لسطح الأوامر | `resolveControlCommandGate`، وأدوات مساعدة لتخويل المرسل، وأدوات مساعدة لسجل الأوامر |
  | `plugin-sdk/secret-input` | تحليل مدخلات الأسرار | أدوات مساعدة لمدخلات الأسرار |
  | `plugin-sdk/webhook-ingress` | أدوات مساعدة لطلبات webhook | أدوات مساعدة لهدف webhook |
  | `plugin-sdk/webhook-request-guards` | أدوات مساعدة لحراسة طلبات webhook | أدوات مساعدة لقراءة/تقييد جسم الطلب |
  | `plugin-sdk/reply-runtime` | وقت تشغيل الرد المشترك | الإرسال الوارد، heartbeat، مخطط الرد، التقسيم |
  | `plugin-sdk/reply-dispatch-runtime` | أدوات مساعدة ضيقة لإرسال الرد | أدوات مساعدة للإكمال + إرسال المزود |
  | `plugin-sdk/reply-history` | أدوات مساعدة لسجل الرد | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | تخطيط مرجع الرد | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | أدوات مساعدة لتقسيم الرد | أدوات مساعدة لتقسيم النص/Markdown |
  | `plugin-sdk/session-store-runtime` | أدوات مساعدة لمخزن الجلسة | أدوات مساعدة لمسار المخزن + updated-at |
  | `plugin-sdk/state-paths` | أدوات مساعدة لمسارات الحالة | أدوات مساعدة لمسارات الحالة وOAuth |
  | `plugin-sdk/routing` | أدوات مساعدة للتوجيه/مفتاح الجلسة | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`، وأدوات مساعدة لتطبيع مفتاح الجلسة |
  | `plugin-sdk/status-helpers` | أدوات مساعدة لحالة القناة | بناة ملخص حالة القناة/الحساب، والقيم الافتراضية لحالة وقت التشغيل، وأدوات بيانات المشكلة المساعدة |
  | `plugin-sdk/target-resolver-runtime` | أدوات مساعدة لحل الهدف | أدوات مساعدة مشتركة لحل الهدف |
  | `plugin-sdk/string-normalization-runtime` | أدوات مساعدة لتطبيع السلاسل | أدوات مساعدة لتطبيع slug/السلاسل |
  | `plugin-sdk/request-url` | أدوات مساعدة لعناوين URL للطلب | استخراج عناوين URL النصية من المدخلات الشبيهة بالطلبات |
  | `plugin-sdk/run-command` | أدوات مساعدة للأوامر المؤقتة | مشغّل أوامر مؤقت مع stdout/stderr مطبّعين |
  | `plugin-sdk/param-readers` | قارئات المعلمات | قارئات معلمات شائعة للأدوات/CLI |
  | `plugin-sdk/tool-send` | استخراج إرسال الأداة | استخراج حقول هدف الإرسال القياسية من وسائط الأداة |
  | `plugin-sdk/temp-path` | أدوات مساعدة للمسارات المؤقتة | أدوات مساعدة مشتركة لمسارات التنزيل المؤقت |
  | `plugin-sdk/logging-core` | أدوات مساعدة للتسجيل | المسجل الفرعي وأدوات إخفاء البيانات المساعدة |
  | `plugin-sdk/markdown-table-runtime` | أدوات مساعدة لجداول Markdown | أدوات مساعدة لوضع جدول Markdown |
  | `plugin-sdk/reply-payload` | أنواع رد الرسائل | أنواع حمولة الرد |
  | `plugin-sdk/provider-setup` | أدوات مساعدة منسقة لإعداد المزود المحلي/المستضاف ذاتيًا | أدوات مساعدة لاكتشاف/إعداد المزود المستضاف ذاتيًا |
  | `plugin-sdk/self-hosted-provider-setup` | أدوات مساعدة مركزة لإعداد مزود OpenAI-compatible المستضاف ذاتيًا | الأدوات نفسها لاكتشاف/إعداد المزود المستضاف ذاتيًا |
  | `plugin-sdk/provider-auth-runtime` | أدوات مساعدة لمصادقة المزود في وقت التشغيل | أدوات مساعدة لحل مفتاح API في وقت التشغيل |
  | `plugin-sdk/provider-auth-api-key` | أدوات مساعدة لإعداد مفتاح API للمزود | أدوات مساعدة للإعداد الأولي/كتابة الملف الشخصي لمفتاح API |
  | `plugin-sdk/provider-auth-result` | أدوات مساعدة لنتيجة مصادقة المزود | باني نتيجة مصادقة OAuth القياسي |
  | `plugin-sdk/provider-auth-login` | أدوات مساعدة لتسجيل الدخول التفاعلي للمزود | أدوات مساعدة مشتركة لتسجيل الدخول التفاعلي |
  | `plugin-sdk/provider-env-vars` | أدوات مساعدة لمتغيرات بيئة المزود | أدوات مساعدة للبحث عن متغيرات بيئة مصادقة المزود |
  | `plugin-sdk/provider-model-shared` | أدوات مساعدة مشتركة لنموذج/إعادة تشغيل المزود | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`، وبناة سياسات إعادة التشغيل المشتركة، وأدوات مساعدة لنقاط نهاية المزود، وأدوات تطبيع معرّف النموذج |
  | `plugin-sdk/provider-catalog-shared` | أدوات مساعدة مشتركة لفهرس المزود | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | تصحيحات الإعداد الأولي للمزود | أدوات مساعدة لإعدادات الإعداد الأولي |
  | `plugin-sdk/provider-http` | أدوات مساعدة HTTP للمزود | أدوات مساعدة عامة لقدرات HTTP/نقطة النهاية للمزود |
  | `plugin-sdk/provider-web-fetch` | أدوات مساعدة web-fetch للمزود | أدوات مساعدة لتسجيل/تخزين مزود web-fetch |
  | `plugin-sdk/provider-web-search` | أدوات مساعدة web-search للمزود | أدوات مساعدة لتسجيل/تخزين/إعداد مزود web-search |
  | `plugin-sdk/provider-tools` | أدوات مساعدة لتوافق أداة/مخطط المزود | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`، وتنظيف مخطط Gemini + التشخيص، وأدوات توافق xAI المساعدة مثل `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | أدوات مساعدة لاستخدام المزود | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage`، وغيرها من أدوات استخدام المزود المساعدة |
  | `plugin-sdk/provider-stream` | أدوات مساعدة لالتفاف بث المزود | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`، وأنواع التفاف البث، وأدوات مساعدة مشتركة لالتفاف Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | قائمة انتظار async مرتبة | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | أدوات مساعدة مشتركة للوسائط | أدوات مساعدة لجلب/تحويل/تخزين الوسائط بالإضافة إلى بناة حمولة الوسائط |
  | `plugin-sdk/media-understanding` | أدوات مساعدة لفهم الوسائط | أنواع مزود فهم الوسائط بالإضافة إلى تصديرات الأدوات المساعدة الموجهة للمزود للصور/الصوت |
  | `plugin-sdk/text-runtime` | أدوات مساعدة مشتركة للنص | إزالة النص المرئي للمساعد، وعرض/تقسيم/جداول markdown، وأدوات إخفاء البيانات، ووسوم التوجيه، وأدوات النص الآمن، وغيرها من أدوات النص/التسجيل ذات الصلة |
  | `plugin-sdk/text-chunking` | أدوات مساعدة لتقسيم النص | أداة مساعدة لتقسيم النص الصادر |
  | `plugin-sdk/speech` | أدوات مساعدة للكلام | أنواع مزود الكلام بالإضافة إلى أدوات مساعدة للتوجيه، والسجل، والتحقق موجهة للمزود |
  | `plugin-sdk/speech-core` | نواة الكلام المشتركة | أنواع مزود الكلام، السجل، التوجيهات، التطبيع |
  | `plugin-sdk/realtime-transcription` | أدوات مساعدة للتفريغ الفوري | أنواع المزود وأدوات السجل المساعدة |
  | `plugin-sdk/realtime-voice` | أدوات مساعدة للصوت الفوري | أنواع المزود وأدوات السجل المساعدة |
  | `plugin-sdk/image-generation-core` | نواة توليد الصور المشتركة | أنواع توليد الصور، وتجاوز الفشل، والمصادقة، وأدوات السجل المساعدة |
  | `plugin-sdk/music-generation` | أدوات مساعدة لتوليد الموسيقى | أنواع المزود/الطلب/النتيجة لتوليد الموسيقى |
  | `plugin-sdk/music-generation-core` | نواة توليد الموسيقى المشتركة | أنواع توليد الموسيقى، وأدوات تجاوز الفشل، والبحث عن المزود، وتحليل مرجع النموذج |
  | `plugin-sdk/video-generation` | أدوات مساعدة لتوليد الفيديو | أنواع المزود/الطلب/النتيجة لتوليد الفيديو |
  | `plugin-sdk/video-generation-core` | نواة توليد الفيديو المشتركة | أنواع توليد الفيديو، وأدوات تجاوز الفشل، والبحث عن المزود، وتحليل مرجع النموذج |
  | `plugin-sdk/interactive-runtime` | أدوات مساعدة للرد التفاعلي | تطبيع/اختزال حمولة الرد التفاعلي |
  | `plugin-sdk/channel-config-primitives` | بدائيات إعدادات القناة | بدائيات ضيقة لمخطط إعدادات القناة |
  | `plugin-sdk/channel-config-writes` | أدوات مساعدة لكتابة إعدادات القناة | أدوات مساعدة لتخويل كتابة إعدادات القناة |
  | `plugin-sdk/channel-plugin-common` | تمهيد القناة المشترك | تصديرات تمهيد القناة المشتركة |
  | `plugin-sdk/channel-status` | أدوات مساعدة لحالة القناة | أدوات مساعدة مشتركة للقطات/ملخصات حالة القناة |
  | `plugin-sdk/allowlist-config-edit` | أدوات مساعدة لإعدادات قائمة السماح | أدوات مساعدة لتحرير/قراءة إعدادات قائمة السماح |
  | `plugin-sdk/group-access` | أدوات مساعدة للوصول الجماعي | أدوات مساعدة مشتركة لقرارات الوصول الجماعي |
  | `plugin-sdk/direct-dm` | أدوات مساعدة للرسائل المباشرة المباشرة | أدوات مساعدة مشتركة للمصادقة/الحراسة في الرسائل المباشرة المباشرة |
  | `plugin-sdk/extension-shared` | أدوات مساعدة مشتركة للإضافات | بدائيات أدوات المساعدة للقنوات السلبية/الحالة |
  | `plugin-sdk/webhook-targets` | أدوات مساعدة لأهداف webhook | سجل أهداف webhook وأدوات تثبيت route |
  | `plugin-sdk/webhook-path` | أدوات مساعدة لمسار webhook | أدوات مساعدة لتطبيع مسار webhook |
  | `plugin-sdk/web-media` | أدوات مساعدة مشتركة لوسائط الويب | أدوات مساعدة لتحميل الوسائط البعيدة/المحلية |
  | `plugin-sdk/zod` | إعادة تصدير Zod | إعادة تصدير `zod` لمستهلكي Plugin SDK |
  | `plugin-sdk/memory-core` | أدوات مساعدة مضمّنة لـ memory-core | سطح أدوات المساعدة لمدير الذاكرة/الإعدادات/الملفات/CLI |
  | `plugin-sdk/memory-core-engine-runtime` | واجهة وقت تشغيل لمحرك الذاكرة | واجهة وقت تشغيل لفهرسة/بحث الذاكرة |
  | `plugin-sdk/memory-core-host-engine-foundation` | محرك الأساس لمضيف الذاكرة | تصديرات محرك الأساس لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-embeddings` | محرك embeddings لمضيف الذاكرة | تصديرات محرك embeddings لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-qmd` | محرك QMD لمضيف الذاكرة | تصديرات محرك QMD لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-storage` | محرك التخزين لمضيف الذاكرة | تصديرات محرك التخزين لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-multimodal` | أدوات مساعدة متعددة الوسائط لمضيف الذاكرة | أدوات مساعدة متعددة الوسائط لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-query` | أدوات مساعدة للاستعلام في مضيف الذاكرة | أدوات مساعدة للاستعلام في مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-secret` | أدوات مساعدة للأسرار في مضيف الذاكرة | أدوات مساعدة للأسرار في مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-status` | أدوات مساعدة للحالة في مضيف الذاكرة | أدوات مساعدة للحالة في مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-cli` | وقت تشغيل CLI لمضيف الذاكرة | أدوات مساعدة لوقت تشغيل CLI لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-core` | وقت التشغيل الأساسي لمضيف الذاكرة | أدوات مساعدة لوقت التشغيل الأساسي لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-files` | أدوات مساعدة لملفات/وقت تشغيل مضيف الذاكرة | أدوات مساعدة لملفات/وقت تشغيل مضيف الذاكرة |
  | `plugin-sdk/memory-lancedb` | أدوات مساعدة مضمّنة لـ memory-lancedb | سطح أدوات المساعدة لـ memory-lancedb |
  | `plugin-sdk/testing` | أدوات الاختبار | أدوات مساعدة ومحاكيات للاختبار |
</Accordion>

هذا الجدول هو عمدًا مجموعة الترحيل الشائعة، وليس سطح SDK الكامل.
توجد القائمة الكاملة التي تضم أكثر من 200 نقطة دخول في
`scripts/lib/plugin-sdk-entrypoints.json`.

ولا تزال تلك القائمة تتضمن بعض طبقات أدوات الإضافات المضمنة المساعدة مثل
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, و`plugin-sdk/matrix*`. وتظل هذه
مصدّرة لأغراض صيانة الإضافات المضمنة والتوافق، لكنها حُذفت عمدًا من
جدول الترحيل الشائع وليست الهدف الموصى به
لشيفرة الإضافات الجديدة.

تنطبق القاعدة نفسها على عائلات الأدوات المساعدة المضمنة الأخرى مثل:

- أدوات مساعدة دعم المتصفح: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- الأسطح المضمنة للمساعدات/الإضافات مثل `plugin-sdk/googlechat`,
  و`plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  و`plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  و`plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  و`plugin-sdk/twitch`,
  و`plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  و`plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  و`plugin-sdk/thread-ownership`, و`plugin-sdk/voice-call`

يكشف `plugin-sdk/github-copilot-token` حاليًا سطح أدوات الرمز المساعد
الضيق `DEFAULT_COPILOT_API_BASE_URL`,
و`deriveCopilotApiBaseUrlFromToken`، و`resolveCopilotApiToken`.

استخدم أضيق عملية استيراد تطابق المهمة. إذا لم تتمكن من العثور على تصدير،
فتحقق من المصدر في `src/plugin-sdk/` أو اسأل في Discord.

## الجدول الزمني للإزالة

| متى | ما الذي يحدث |
| ---------------------- | ----------------------------------------------------------------------- |
| **الآن** | تصدر الواجهات المهجورة تحذيرات في وقت التشغيل |
| **الإصدار الرئيسي التالي** | ستُزال الواجهات المهجورة؛ وستفشل الإضافات التي لا تزال تستخدمها |

تم ترحيل جميع الإضافات الأساسية بالفعل. ويجب على الإضافات الخارجية الترحيل
قبل الإصدار الرئيسي التالي.

## كتم التحذيرات مؤقتًا

اضبط متغيرات البيئة التالية أثناء العمل على الترحيل:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

هذا مخرج مؤقت، وليس حلًا دائمًا.

## ذو صلة

- [البدء](/ar/plugins/building-plugins) — أنشئ أول إضافة لك
- [نظرة عامة على SDK](/ar/plugins/sdk-overview) — المرجع الكامل للاستيراد عبر المسارات الفرعية
- [إضافات القنوات](/ar/plugins/sdk-channel-plugins) — بناء إضافات القنوات
- [إضافات المزودين](/ar/plugins/sdk-provider-plugins) — بناء إضافات المزودين
- [الداخلية الخاصة بالإضافات](/ar/plugins/architecture) — نظرة معمقة على البنية
- [بيان الإضافة](/ar/plugins/manifest) — مرجع مخطط البيان
