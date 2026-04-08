---
read_when:
    - أنت ترى التحذير OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - أنت ترى التحذير OPENCLAW_EXTENSION_API_DEPRECATED
    - أنت تقوم بتحديث إضافة إلى معمارية الإضافات الحديثة
    - أنت تصون إضافة OpenClaw خارجية
sidebarTitle: Migrate to SDK
summary: الانتقال من طبقة التوافق العكسي القديمة إلى Plugin SDK الحديثة
title: ترحيل Plugin SDK
x-i18n:
    generated_at: "2026-04-08T02:18:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 155a8b14bc345319c8516ebdb8a0ccdea2c5f7fa07dad343442996daee21ecad
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# ترحيل Plugin SDK

انتقل OpenClaw من طبقة توافق عكسي واسعة إلى معمارية إضافات حديثة
ذات عمليات استيراد مركزة وموثقة. إذا كانت إضافتك قد بُنيت قبل
المعمارية الجديدة، فسيساعدك هذا الدليل على الترحيل.

## ما الذي يتغير

كان نظام الإضافات القديم يوفر سطحين واسعين ومفتوحين يتيحان للإضافات استيراد
أي شيء تحتاج إليه من نقطة دخول واحدة:

- **`openclaw/plugin-sdk/compat`** — استيراد واحد يعيد تصدير عشرات
  الأدوات المساعدة. وقد تم تقديمه لإبقاء الإضافات الأقدم المعتمدة على الخطافات تعمل بينما
  كانت معمارية الإضافات الجديدة قيد الإنشاء.
- **`openclaw/extension-api`** — جسر يمنح الإضافات وصولًا مباشرًا إلى
  أدوات جانب المضيف مثل مشغّل الوكيل المضمن.

كلا السطحين أصبح الآن **مهملًا**. ما زالا يعملان في وقت التشغيل، لكن
يجب ألا تستخدمهما الإضافات الجديدة، وينبغي أن تهاجر الإضافات الحالية قبل أن تزيلهما
الإصدار الرئيسي التالي.

<Warning>
  ستُزال طبقة التوافق العكسي في إصدار رئيسي مستقبلي.
  وستتعطل الإضافات التي ما تزال تستورد من هذه الأسطح عند حدوث ذلك.
</Warning>

## لماذا تغيّر هذا

تسبب النهج القديم في مشكلات:

- **بدء تشغيل بطيء** — كان استيراد أداة مساعدة واحدة يحمّل عشرات الوحدات غير المرتبطة
- **تبعيات دائرية** — جعلت إعادة التصدير الواسعة من السهل إنشاء دورات استيراد
- **سطح API غير واضح** — لم تكن هناك طريقة لمعرفة أي التصديرات مستقرة وأيها داخلية

تصلح Plugin SDK الحديثة هذا: كل مسار استيراد (`openclaw/plugin-sdk/\<subpath\>`)
هو وحدة صغيرة مستقلة بذاتها ذات غرض واضح وعقد موثق.

كما أُزيلت أيضًا طبقات الراحة القديمة لموفري القنوات المجمعة. فالاستيرادات
مثل `openclaw/plugin-sdk/slack` و`openclaw/plugin-sdk/discord`،
و`openclaw/plugin-sdk/signal` و`openclaw/plugin-sdk/whatsapp`،
وطبقات الأدوات المساعدة الموسومة بالقناة، و
`openclaw/plugin-sdk/telegram-core` كانت اختصارات داخلية خاصة بالمستودع الأحادي، وليست
عقود إضافات مستقرة. استخدم بدلًا منها المسارات الفرعية العامة الضيقة في SDK. وداخل
مساحة عمل الإضافات المجمعة، احتفظ بالأدوات المساعدة المملوكة للموفر في
`api.ts` أو `runtime-api.ts` الخاصين بتلك الإضافة.

أمثلة موفّرين مجمّعين حالية:

- يحتفظ Anthropic بأدوات مساعدة البث الخاصة بـ Claude في طبقة
  `api.ts` / `contract-api.ts` الخاصة به
- يحتفظ OpenAI ببناة الموفر، وأدوات مساعدة النموذج الافتراضي، وبناة
  الموفّر الآني في `api.ts` الخاص به
- يحتفظ OpenRouter بباني الموفر وأدوات مساعدة التهيئة/الإعداد في
  `api.ts` الخاص به

## كيفية الترحيل

<Steps>
  <Step title="ترحيل المعالجات الأصلية للموافقة إلى حقائق القدرات">
    تعرض إضافات القنوات القادرة على الموافقة الآن سلوك الموافقة الأصلي من خلال
    `approvalCapability.nativeRuntime` بالإضافة إلى سجل سياق وقت التشغيل المشترك.

    التغييرات الأساسية:

    - استبدل `approvalCapability.handler.loadRuntime(...)` بـ
      `approvalCapability.nativeRuntime`
    - انقل المصادقة/التسليم الخاصة بالموافقة من البنية القديمة `plugin.auth` /
      `plugin.approvals` إلى `approvalCapability`
    - تمت إزالة `ChannelPlugin.approvals` من عقد
      الإضافات العامة للقنوات؛ انقل حقول التسليم/الأصلية/العرض إلى `approvalCapability`
    - يظل `plugin.auth` مخصصًا فقط لتدفقات تسجيل الدخول/الخروج الخاصة بالقناة؛ أما خطافات
      مصادقة الموافقة فيه فلم تعد تُقرأ من النواة
    - سجّل كائنات وقت التشغيل المملوكة للقناة، مثل العملاء أو الرموز أو تطبيقات
      Bolt، عبر `openclaw/plugin-sdk/channel-runtime-context`
    - لا ترسل إشعارات إعادة توجيه مملوكة للإضافة من معالجات الموافقة الأصلية؛
      فالنواة تمتلك الآن إشعارات "تم التوجيه إلى مكان آخر" المستندة إلى نتائج التسليم الفعلية
    - عند تمرير `channelRuntime` إلى `createChannelManager(...)`، وفّر
      سطح `createPluginRuntime().channel` حقيقيًا. يتم رفض القوالب الجزئية.

    راجع `/plugins/sdk-channel-plugins` للاطلاع على التخطيط الحالي
    لقدرة الموافقة.

  </Step>

  <Step title="مراجعة سلوك الرجوع الاحتياطي لملتفات Windows">
    إذا كانت إضافتك تستخدم `openclaw/plugin-sdk/windows-spawn`،
    فإن ملفات `.cmd`/`.bat` غير المحلولة في Windows تفشل الآن بشكل مغلق ما لم تمرر
    صراحةً `allowShellFallback: true`.

    ```typescript
    // قبل
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // بعد
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // اضبط هذا فقط للمتصلين المتوافقين الموثوقين الذين يقبلون عمدًا
      // الرجوع الاحتياطي عبر الصدفة.
      allowShellFallback: true,
    });
    ```

    إذا لم يكن المتصل لديك يعتمد عمدًا على الرجوع الاحتياطي عبر الصدفة، فلا تضبط
    `allowShellFallback` وتعامل مع الخطأ المطروح بدلًا من ذلك.

  </Step>

  <Step title="العثور على الاستيرادات المهملة">
    ابحث في إضافتك عن الاستيرادات من أي من السطحين المهملين:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="الاستبدال باستيرادات مركزة">
    يرتبط كل تصدير من السطح القديم بمسار استيراد حديث محدد:

    ```typescript
    // قبل (طبقة توافق عكسي مهملة)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // بعد (استيرادات حديثة ومركزة)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    بالنسبة إلى أدوات جانب المضيف، استخدم وقت تشغيل الإضافة المحقون بدلًا من الاستيراد
    المباشر:

    ```typescript
    // قبل (جسر extension-api مهمل)
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
    | أدوات مساعدة مخزن الجلسات | `api.runtime.agent.session.*` |

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
  | `plugin-sdk/plugin-entry` | أداة مساعد إدخال الإضافة القياسية | `definePluginEntry` |
  | `plugin-sdk/core` | إعادة تصدير جامعة قديمة لتعريفات/بناة إدخال القنوات | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | تصدير مخطط التكوين الجذري | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | أداة مساعد إدخال موفر واحد | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | تعريفات وبناة إدخال القنوات المركزة | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | أدوات مساعدة مشتركة لمعالج الإعداد | مطالبات قائمة السماح، وبناة حالة الإعداد |
  | `plugin-sdk/setup-runtime` | أدوات مساعدة وقت تشغيل الإعداد | مهايئات تصحيح الإعداد الآمنة للاستيراد، وأدوات مساعدة ملاحظات البحث، و`promptResolvedAllowFrom` و`splitSetupEntries` ووكلاء الإعداد المفوضون |
  | `plugin-sdk/setup-adapter-runtime` | أدوات مساعدة لمهايئات الإعداد | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | أدوات مساعدة لأدوات الإعداد | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | أدوات مساعدة متعددة الحسابات | أدوات مساعدة قائمة الحسابات/التكوين/بوابة الإجراءات |
  | `plugin-sdk/account-id` | أدوات مساعدة لمعرّف الحساب | `DEFAULT_ACCOUNT_ID`، وتطبيع معرّف الحساب |
  | `plugin-sdk/account-resolution` | أدوات مساعدة للبحث عن الحساب | أدوات مساعدة للبحث عن الحساب + الرجوع الافتراضي |
  | `plugin-sdk/account-helpers` | أدوات مساعدة ضيقة للحساب | أدوات مساعدة قائمة الحسابات/إجراءات الحساب |
  | `plugin-sdk/channel-setup` | مهايئات معالج الإعداد | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`، بالإضافة إلى `DEFAULT_ACCOUNT_ID` و`createTopLevelChannelDmPolicy` و`setSetupChannelEnabled` و`splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | بدائيات إقران الرسائل الخاصة | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | أسلاك بادئة الرد وحالة الكتابة | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | مصانع مهايئات التكوين | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | بناة مخطط التكوين | أنواع مخطط تكوين القناة |
  | `plugin-sdk/telegram-command-config` | أدوات مساعدة لتكوين أوامر Telegram | تطبيع أسماء الأوامر، وتشذيب الوصف، والتحقق من التكرار/التعارض |
  | `plugin-sdk/channel-policy` | حل سياسات المجموعات/الرسائل الخاصة | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | تتبع حالة الحساب | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | أدوات مساعدة لأغلفة الوارد | أدوات مساعدة مشتركة للمسارات وبناء الأظرف |
  | `plugin-sdk/inbound-reply-dispatch` | أدوات مساعدة لردود الوارد | أدوات مساعدة مشتركة للتسجيل والإرسال |
  | `plugin-sdk/messaging-targets` | تحليل أهداف المراسلة | أدوات مساعدة لتحليل/مطابقة الأهداف |
  | `plugin-sdk/outbound-media` | أدوات مساعدة للوسائط الصادرة | تحميل الوسائط الصادرة المشتركة |
  | `plugin-sdk/outbound-runtime` | أدوات مساعدة لوقت التشغيل الصادر | أدوات مساعدة لهوية الإرسال/وكيل الإرسال |
  | `plugin-sdk/thread-bindings-runtime` | أدوات مساعدة لربط السلاسل | دورة حياة ربط السلاسل وأدوات المهايئات |
  | `plugin-sdk/agent-media-payload` | أدوات مساعدة قديمة لحمولات الوسائط | باني حمولة وسائط الوكيل لتخطيطات الحقول القديمة |
  | `plugin-sdk/channel-runtime` | طبقة توافق مهملة | أدوات وقت تشغيل القناة القديمة فقط |
  | `plugin-sdk/channel-send-result` | أنواع نتائج الإرسال | أنواع نتائج الرد |
  | `plugin-sdk/runtime-store` | تخزين دائم للإضافة | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | أدوات مساعدة واسعة لوقت التشغيل | أدوات وقت التشغيل/التسجيل/النسخ الاحتياطي/تثبيت الإضافات |
  | `plugin-sdk/runtime-env` | أدوات ضيقة لبيئة وقت التشغيل | أدوات التسجيل/بيئة وقت التشغيل، والمهلة، وإعادة المحاولة، والتراجع التدريجي |
  | `plugin-sdk/plugin-runtime` | أدوات مساعدة مشتركة لوقت تشغيل الإضافة | أدوات مساعدة لأوامر/خطافات/HTTP/تفاعل الإضافات |
  | `plugin-sdk/hook-runtime` | أدوات مساعدة لمسار الخطافات | أدوات مساعدة مشتركة لمسار خطافات الويب/الداخلية |
  | `plugin-sdk/lazy-runtime` | أدوات مساعدة لوقت التشغيل الكسول | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | أدوات مساعدة للعمليات | أدوات مساعدة مشتركة للتنفيذ |
  | `plugin-sdk/cli-runtime` | أدوات مساعدة لوقت تشغيل CLI | تنسيق الأوامر، والانتظارات، وأدوات مساعدة الإصدارات |
  | `plugin-sdk/gateway-runtime` | أدوات مساعدة للبوابة | عميل البوابة وأدوات مساعدة تصحيح حالة القنوات |
  | `plugin-sdk/config-runtime` | أدوات مساعدة للتكوين | أدوات مساعدة لتحميل/كتابة التكوين |
  | `plugin-sdk/telegram-command-config` | أدوات مساعدة لأوامر Telegram | أدوات مساعدة مستقرة احتياطيًا للتحقق من أوامر Telegram عندما لا يكون سطح عقد Telegram المجمّع متاحًا |
  | `plugin-sdk/approval-runtime` | أدوات مساعدة لمطالبات الموافقة | حمولة موافقة التنفيذ/الإضافة، وأدوات مساعدة قدرة/ملف تعريف الموافقة، وأدوات مساعدة التوجيه/وقت التشغيل للموافقة الأصلية |
  | `plugin-sdk/approval-auth-runtime` | أدوات مساعدة لمصادقة الموافقة | حل الموافق، ومصادقة الإجراء في الدردشة نفسها |
  | `plugin-sdk/approval-client-runtime` | أدوات مساعدة لعميل الموافقة | أدوات مساعدة ملف التعريف/المرشحات لموافقة التنفيذ الأصلية |
  | `plugin-sdk/approval-delivery-runtime` | أدوات مساعدة لتسليم الموافقة | مهايئات قدرة/تسليم الموافقة الأصلية |
  | `plugin-sdk/approval-gateway-runtime` | أدوات مساعدة لبوابة الموافقة | أداة مساعدة مشتركة لحل بوابة الموافقة |
  | `plugin-sdk/approval-handler-adapter-runtime` | أدوات مساعدة لمهايئات الموافقة | أدوات مساعدة خفيفة لتحميل مهايئات الموافقة الأصلية لنقاط إدخال القنوات الساخنة |
  | `plugin-sdk/approval-handler-runtime` | أدوات مساعدة لمعالجات الموافقة | أدوات أوسع لوقت تشغيل معالجات الموافقة؛ فَضّل الطبقات الأضيق للمهايئات/البوابة عندما تكفي |
  | `plugin-sdk/approval-native-runtime` | أدوات مساعدة لهدف الموافقة | أدوات مساعدة ربط الهدف/الحساب للموافقة الأصلية |
  | `plugin-sdk/approval-reply-runtime` | أدوات مساعدة لردود الموافقة | أدوات مساعدة لحمولات رد الموافقة على التنفيذ/الإضافة |
  | `plugin-sdk/channel-runtime-context` | أدوات مساعدة لسياق وقت تشغيل القناة | أدوات مساعدة عامة لتسجيل/جلب/مراقبة سياق وقت تشغيل القناة |
  | `plugin-sdk/security-runtime` | أدوات مساعدة للأمان | أدوات مساعدة مشتركة للثقة، وبوابات الرسائل الخاصة، والمحتوى الخارجي، وتجميع الأسرار |
  | `plugin-sdk/ssrf-policy` | أدوات مساعدة لسياسة SSRF | أدوات مساعدة لقائمة سماح المضيف وسياسة الشبكة الخاصة |
  | `plugin-sdk/ssrf-runtime` | أدوات مساعدة لوقت تشغيل SSRF | أدوات مساعدة للمُرسِل المثبت، والجلب المحروس، وسياسات SSRF |
  | `plugin-sdk/collection-runtime` | أدوات مساعدة لذاكرة التخزين المحدودة | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | أدوات مساعدة لبوابات التشخيص | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | أدوات مساعدة لتنسيق الأخطاء | `formatUncaughtError`, `isApprovalNotFoundError`، وأدوات رسم الأخطاء |
  | `plugin-sdk/fetch-runtime` | أدوات مساعدة للجلب/الوكيل المغلف | `resolveFetch`، وأدوات مساعدة الوكيل |
  | `plugin-sdk/host-runtime` | أدوات مساعدة لتطبيع المضيف | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | أدوات مساعدة لإعادة المحاولة | `RetryConfig`, `retryAsync`، ومشغلات السياسات |
  | `plugin-sdk/allow-from` | تنسيق قائمة السماح | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | ربط مدخلات قائمة السماح | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | بوابات الأوامر وأدوات سطح الأوامر | `resolveControlCommandGate`، وأدوات مساعدة تفويض المرسل، وأدوات سجل الأوامر |
  | `plugin-sdk/secret-input` | تحليل المدخلات السرية | أدوات مساعدة للمدخلات السرية |
  | `plugin-sdk/webhook-ingress` | أدوات مساعدة لطلبات Webhook | أدوات مساعدة لهدف Webhook |
  | `plugin-sdk/webhook-request-guards` | أدوات مساعدة لحراسة طلبات Webhook | أدوات مساعدة لقراءة/تقييد جسم الطلب |
  | `plugin-sdk/reply-runtime` | وقت تشغيل الرد المشترك | الإرسال الوارد، والنبض، ومخطط الرد، والتجزئة |
  | `plugin-sdk/reply-dispatch-runtime` | أدوات ضيقة لإرسال الرد | الإنهاء + أدوات مساعدة إرسال الموفر |
  | `plugin-sdk/reply-history` | أدوات مساعدة لسجل الرد | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | تخطيط مراجع الرد | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | أدوات مساعدة لتجزئة الرد | أدوات مساعدة لتجزئة النص/Markdown |
  | `plugin-sdk/session-store-runtime` | أدوات مساعدة لمخزن الجلسات | أدوات مساعدة لمسار المخزن ووقت آخر تحديث |
  | `plugin-sdk/state-paths` | أدوات مساعدة لمسارات الحالة | أدوات مساعدة لأدلة الحالة وOAuth |
  | `plugin-sdk/routing` | أدوات مساعدة للتوجيه/مفاتيح الجلسة | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`، وأدوات تطبيع مفاتيح الجلسة |
  | `plugin-sdk/status-helpers` | أدوات مساعدة لحالة القناة | بناة ملخصات حالة القناة/الحساب، وافتراضات حالة وقت التشغيل، وأدوات بيانات تعريف المشكلات |
  | `plugin-sdk/target-resolver-runtime` | أدوات مساعدة لحل الأهداف | أدوات مساعدة مشتركة لحل الأهداف |
  | `plugin-sdk/string-normalization-runtime` | أدوات مساعدة لتطبيع السلاسل | أدوات مساعدة لتطبيع الـ slug/السلاسل |
  | `plugin-sdk/request-url` | أدوات مساعدة لعناوين URL للطلبات | استخراج عناوين URL النصية من المدخلات الشبيهة بالطلبات |
  | `plugin-sdk/run-command` | أدوات مساعدة للأوامر المؤقتة | مشغّل أوامر مؤقت مع stdout/stderr مطبعين |
  | `plugin-sdk/param-readers` | قارئات المعاملات | قارئات شائعة لمعاملات الأدوات/CLI |
  | `plugin-sdk/tool-send` | استخراج إرسال الأدوات | استخراج حقول هدف الإرسال القياسية من وسائط الأدوات |
  | `plugin-sdk/temp-path` | أدوات مساعدة للمسارات المؤقتة | أدوات مساعدة مشتركة لمسارات تنزيل الملفات المؤقتة |
  | `plugin-sdk/logging-core` | أدوات مساعدة للتسجيل | مسجل النظام الفرعي وأدوات الإخفاء |
  | `plugin-sdk/markdown-table-runtime` | أدوات مساعدة لجداول Markdown | أدوات مساعدة لأنماط جداول Markdown |
  | `plugin-sdk/reply-payload` | أنواع ردود الرسائل | أنواع حمولة الرد |
  | `plugin-sdk/provider-setup` | أدوات مساعدة منسقة لإعداد الموفّرين المحليين/المستضافين ذاتيًا | أدوات مساعدة لاكتشاف/تكوين الموفّرين المستضافين ذاتيًا |
  | `plugin-sdk/self-hosted-provider-setup` | أدوات مركزة لإعداد الموفّرين المستضافين ذاتيًا المتوافقين مع OpenAI | أدوات مساعدة الاكتشاف/التكوين نفسها للموفّرين المستضافين ذاتيًا |
  | `plugin-sdk/provider-auth-runtime` | أدوات مساعدة لمصادقة موفر وقت التشغيل | أدوات مساعدة لحل مفاتيح API في وقت التشغيل |
  | `plugin-sdk/provider-auth-api-key` | أدوات مساعدة لإعداد مفتاح API للموفر | أدوات مساعدة لتهيئة/كتابة ملفات تعريف مفتاح API |
  | `plugin-sdk/provider-auth-result` | أدوات مساعدة لنتائج مصادقة الموفر | باني قياسي لنتائج مصادقة OAuth |
  | `plugin-sdk/provider-auth-login` | أدوات مساعدة لتسجيل الدخول التفاعلي للموفر | أدوات مساعدة مشتركة لتسجيل الدخول التفاعلي |
  | `plugin-sdk/provider-env-vars` | أدوات مساعدة لمتغيرات بيئة الموفر | أدوات مساعدة للبحث عن متغيرات بيئة مصادقة الموفر |
  | `plugin-sdk/provider-model-shared` | أدوات مساعدة مشتركة لنماذج/إعادة تشغيل الموفّر | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`، وبناة سياسات الإعادة المشتركة، وأدوات مساعدة لنقاط نهاية الموفّر، وأدوات تطبيع معرّفات النماذج |
  | `plugin-sdk/provider-catalog-shared` | أدوات مساعدة مشتركة لفهرس الموفّر | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | تصحيحات تهيئة الموفّر | أدوات مساعدة لتكوين التهيئة |
  | `plugin-sdk/provider-http` | أدوات مساعدة HTTP للموفّر | أدوات مساعدة عامة لقدرات HTTP/نقاط النهاية الخاصة بالموفّر |
  | `plugin-sdk/provider-web-fetch` | أدوات مساعدة جلب الويب للموفّر | أدوات مساعدة لتسجيل/تخزين موفّر web-fetch |
  | `plugin-sdk/provider-web-search-contract` | أدوات مساعدة لعقد بحث الويب للموفّر | أدوات ضيقة لعقد تكوين/اعتماد بحث الويب مثل `enablePluginInConfig` و`resolveProviderWebSearchPluginConfig` وأدوات ضبط/جلب بيانات الاعتماد المقيّدة |
  | `plugin-sdk/provider-web-search` | أدوات مساعدة لبحث الويب للموفّر | أدوات مساعدة لتسجيل/تخزين/وقت تشغيل موفّر بحث الويب |
  | `plugin-sdk/provider-tools` | أدوات مساعدة لتوافق الأدوات/المخططات للموفّر | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`، وتنظيف مخططات Gemini + التشخيصات، وأدوات توافق xAI مثل `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
  | `plugin-sdk/provider-usage` | أدوات مساعدة لاستخدام الموفّر | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage`، وأدوات مساعدة أخرى لاستخدام الموفّر |
  | `plugin-sdk/provider-stream` | أدوات مساعدة لأغلفة بث الموفّر | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`، وأنواع أغلفة البث، وأدوات مساعدة مشتركة لأغلفة Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
  | `plugin-sdk/keyed-async-queue` | طابور async مرتب | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | أدوات مساعدة مشتركة للوسائط | أدوات مساعدة لجلب/تحويل/تخزين الوسائط بالإضافة إلى بناة حمولات الوسائط |
  | `plugin-sdk/media-generation-runtime` | أدوات مساعدة مشتركة لتوليد الوسائط | أدوات مساعدة مشتركة للتحويل الاحتياطي، واختيار المرشحين، ورسائل النماذج المفقودة لتوليد الصور/الفيديو/الموسيقى |
  | `plugin-sdk/media-understanding` | أدوات مساعدة لفهم الوسائط | أنواع موفري فهم الوسائط بالإضافة إلى تصديرات أدوات مساعدة الصور/الصوت المواجهة للموفّر |
  | `plugin-sdk/text-runtime` | أدوات مساعدة مشتركة للنص | إزالة النص المرئي للمساعد، وأدوات عرض/تجزئة/جداول Markdown، وأدوات الإخفاء، وأدوات وسم التوجيهات، وأدوات النص الآمن، وأدوات ذات صلة بالنص/التسجيل |
  | `plugin-sdk/text-chunking` | أدوات مساعدة لتجزئة النص | أداة مساعدة لتجزئة النص الصادر |
  | `plugin-sdk/speech` | أدوات مساعدة للكلام | أنواع موفري الكلام بالإضافة إلى تصديرات أدوات مساعدة التوجيهات والسجل والتحقق المواجهة للموفّر |
  | `plugin-sdk/speech-core` | نواة كلام مشتركة | أنواع موفري الكلام، والسجل، والتوجيهات، والتطبيع |
  | `plugin-sdk/realtime-transcription` | أدوات مساعدة للنسخ الفوري | أنواع الموفّرين وأدوات السجل |
  | `plugin-sdk/realtime-voice` | أدوات مساعدة للصوت الفوري | أنواع الموفّرين وأدوات السجل |
  | `plugin-sdk/image-generation-core` | نواة مشتركة لتوليد الصور | أنواع توليد الصور، والتحويل الاحتياطي، والمصادقة، وأدوات السجل |
  | `plugin-sdk/music-generation` | أدوات مساعدة لتوليد الموسيقى | أنواع موفّر/طلب/نتيجة توليد الموسيقى |
  | `plugin-sdk/music-generation-core` | نواة مشتركة لتوليد الموسيقى | أنواع توليد الموسيقى، وأدوات التحويل الاحتياطي، والبحث عن الموفّر، وتحليل مراجع النماذج |
  | `plugin-sdk/video-generation` | أدوات مساعدة لتوليد الفيديو | أنواع موفّر/طلب/نتيجة توليد الفيديو |
  | `plugin-sdk/video-generation-core` | نواة مشتركة لتوليد الفيديو | أنواع توليد الفيديو، وأدوات التحويل الاحتياطي، والبحث عن الموفّر، وتحليل مراجع النماذج |
  | `plugin-sdk/interactive-runtime` | أدوات مساعدة للردود التفاعلية | تطبيع/اختزال حمولات الردود التفاعلية |
  | `plugin-sdk/channel-config-primitives` | بدائيات تكوين القنوات | بدائيات ضيقة لمخطط تكوين القناة |
  | `plugin-sdk/channel-config-writes` | أدوات مساعدة لكتابة تكوين القنوات | أدوات مساعدة لتفويض كتابة تكوين القنوات |
  | `plugin-sdk/channel-plugin-common` | تمهيد مشترك للقنوات | تصديرات تمهيد مشتركة لإضافات القنوات |
  | `plugin-sdk/channel-status` | أدوات مساعدة لحالة القنوات | أدوات مساعدة مشتركة لالتقاط/تلخيص حالة القناة |
  | `plugin-sdk/allowlist-config-edit` | أدوات مساعدة لتكوين قائمة السماح | أدوات مساعدة لتحرير/قراءة تكوين قائمة السماح |
  | `plugin-sdk/group-access` | أدوات مساعدة للوصول إلى المجموعات | أدوات مساعدة مشتركة لاتخاذ قرارات وصول المجموعات |
  | `plugin-sdk/direct-dm` | أدوات مساعدة للرسائل الخاصة المباشرة | أدوات مساعدة مشتركة لمصادقة/حراسة الرسائل الخاصة المباشرة |
  | `plugin-sdk/extension-shared` | أدوات مساعدة مشتركة للإضافات | بدائيات القنوات/الحالة السلبية والمساعد الوكيل المحيط |
  | `plugin-sdk/webhook-targets` | أدوات مساعدة لأهداف Webhook | سجل أهداف Webhook وأدوات تثبيت المسارات |
  | `plugin-sdk/webhook-path` | أدوات مساعدة لمسارات Webhook | أدوات مساعدة لتطبيع مسارات Webhook |
  | `plugin-sdk/web-media` | أدوات مساعدة مشتركة لوسائط الويب | أدوات مساعدة لتحميل الوسائط البعيدة/المحلية |
  | `plugin-sdk/zod` | إعادة تصدير Zod | `zod` معاد تصديره لمستهلكي Plugin SDK |
  | `plugin-sdk/memory-core` | أدوات memory-core المجمعة | سطح أدوات مساعدة مدير/تكوين/ملفات/CLI الخاصة بالذاكرة |
  | `plugin-sdk/memory-core-engine-runtime` | واجهة وقت تشغيل لمحرك الذاكرة | واجهة وقت تشغيل فهرسة/بحث الذاكرة |
  | `plugin-sdk/memory-core-host-engine-foundation` | محرك الأساس لمضيف الذاكرة | تصديرات محرك الأساس لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-embeddings` | محرك التضمين لمضيف الذاكرة | تصديرات محرك التضمين لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-qmd` | محرك QMD لمضيف الذاكرة | تصديرات محرك QMD لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-engine-storage` | محرك التخزين لمضيف الذاكرة | تصديرات محرك التخزين لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-multimodal` | أدوات مساعدة متعددة الوسائط لمضيف الذاكرة | أدوات مساعدة متعددة الوسائط لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-query` | أدوات مساعدة للاستعلام في مضيف الذاكرة | أدوات مساعدة للاستعلام في مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-secret` | أدوات مساعدة للأسرار في مضيف الذاكرة | أدوات مساعدة للأسرار في مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-events` | أدوات مساعدة لسجل أحداث مضيف الذاكرة | أدوات مساعدة لسجل أحداث مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-status` | أدوات مساعدة لحالة مضيف الذاكرة | أدوات مساعدة لحالة مضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-cli` | وقت تشغيل CLI لمضيف الذاكرة | أدوات مساعدة وقت تشغيل CLI لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-core` | وقت تشغيل النواة لمضيف الذاكرة | أدوات مساعدة وقت تشغيل النواة لمضيف الذاكرة |
  | `plugin-sdk/memory-core-host-runtime-files` | أدوات مساعدة للملفات/وقت التشغيل لمضيف الذاكرة | أدوات مساعدة للملفات/وقت التشغيل لمضيف الذاكرة |
  | `plugin-sdk/memory-host-core` | اسم مستعار لوقت تشغيل نواة مضيف الذاكرة | اسم مستعار محايد للمورّد لأدوات وقت تشغيل نواة مضيف الذاكرة |
  | `plugin-sdk/memory-host-events` | اسم مستعار لسجل أحداث مضيف الذاكرة | اسم مستعار محايد للمورّد لأدوات سجل أحداث مضيف الذاكرة |
  | `plugin-sdk/memory-host-files` | اسم مستعار لملفات/وقت تشغيل مضيف الذاكرة | اسم مستعار محايد للمورّد لأدوات ملفات/وقت تشغيل مضيف الذاكرة |
  | `plugin-sdk/memory-host-markdown` | أدوات مساعدة لـ markdown المُدار | أدوات مساعدة مشتركة لـ markdown المُدار للإضافات المجاورة للذاكرة |
  | `plugin-sdk/memory-host-search` | واجهة بحث الذاكرة النشطة | واجهة وقت تشغيل كسولة لمدير بحث الذاكرة النشطة |
  | `plugin-sdk/memory-host-status` | اسم مستعار لحالة مضيف الذاكرة | اسم مستعار محايد للمورّد لأدوات حالة مضيف الذاكرة |
  | `plugin-sdk/memory-lancedb` | أدوات memory-lancedb المجمعة | سطح أدوات مساعدة memory-lancedb |
  | `plugin-sdk/testing` | أدوات الاختبار | أدوات الاختبار والمحاكاة |
</Accordion>

هذا الجدول هو عمدًا مجموعة الترحيل الشائعة، وليس سطح SDK الكامل.
وتوجد القائمة الكاملة التي تضم أكثر من 200 نقطة إدخال في
`scripts/lib/plugin-sdk-entrypoints.json`.

ما تزال تلك القائمة تتضمن بعض طبقات أدوات المساعدة الخاصة بالإضافات المجمعة مثل
`plugin-sdk/feishu` و`plugin-sdk/feishu-setup` و`plugin-sdk/zalo`،
و`plugin-sdk/zalo-setup` و`plugin-sdk/matrix*`. وما تزال هذه المداخل مُصدَّرة
لصيانة الإضافات المجمعة والتوافق، لكنها أُزيلت عمدًا من
جدول الترحيل الشائع وليست الهدف الموصى به
لكود الإضافات الجديد.

تنطبق القاعدة نفسها على عائلات الأدوات المجمعة الأخرى مثل:

- أدوات دعم المتصفح: `plugin-sdk/browser-cdp` و`plugin-sdk/browser-config-runtime` و`plugin-sdk/browser-config-support` و`plugin-sdk/browser-control-auth` و`plugin-sdk/browser-node-runtime` و`plugin-sdk/browser-profiles` و`plugin-sdk/browser-security-runtime` و`plugin-sdk/browser-setup-tools` و`plugin-sdk/browser-support`
- Matrix: ‏`plugin-sdk/matrix*`
- LINE: ‏`plugin-sdk/line*`
- IRC: ‏`plugin-sdk/irc*`
- أسطح الأدوات/الإضافات المجمعة مثل `plugin-sdk/googlechat`،
  و`plugin-sdk/zalouser` و`plugin-sdk/bluebubbles*`,
  و`plugin-sdk/mattermost*` و`plugin-sdk/msteams`,
  و`plugin-sdk/nextcloud-talk` و`plugin-sdk/nostr` و`plugin-sdk/tlon`,
  و`plugin-sdk/twitch`,
  و`plugin-sdk/github-copilot-login` و`plugin-sdk/github-copilot-token`,
  و`plugin-sdk/diagnostics-otel` و`plugin-sdk/diffs` و`plugin-sdk/llm-task`,
  و`plugin-sdk/thread-ownership` و`plugin-sdk/voice-call`

يكشف `plugin-sdk/github-copilot-token` حاليًا عن
سطح أدوات الرموز الضيق `DEFAULT_COPILOT_API_BASE_URL`،
و`deriveCopilotApiBaseUrlFromToken`، و`resolveCopilotApiToken`.

استخدم أضيق استيراد يطابق المهمة. وإذا لم تتمكن من العثور على تصدير،
فتحقق من المصدر في `src/plugin-sdk/` أو اسأل في Discord.

## الجدول الزمني للإزالة

| متى | ما الذي يحدث |
| ---------------------- | ----------------------------------------------------------------------- |
| **الآن** | تُصدر الأسطح المهملة تحذيرات في وقت التشغيل |
| **الإصدار الرئيسي التالي** | ستُزال الأسطح المهملة؛ وستفشل الإضافات التي ما تزال تستخدمها |

لقد تم بالفعل ترحيل جميع الإضافات الأساسية. ويجب أن تهاجر الإضافات الخارجية
قبل الإصدار الرئيسي التالي.

## كتم التحذيرات مؤقتًا

اضبط متغيرات البيئة هذه أثناء عملك على الترحيل:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

هذا منفذ هروب مؤقت، وليس حلًا دائمًا.

## ذو صلة

- [Getting Started](/ar/plugins/building-plugins) — أنشئ إضافتك الأولى
- [SDK Overview](/ar/plugins/sdk-overview) — مرجع كامل لاستيراد المسارات الفرعية
- [Channel Plugins](/ar/plugins/sdk-channel-plugins) — بناء إضافات القنوات
- [Provider Plugins](/ar/plugins/sdk-provider-plugins) — بناء إضافات الموفّرين
- [Plugin Internals](/ar/plugins/architecture) — تعمق في المعمارية
- [Plugin Manifest](/ar/plugins/manifest) — مرجع مخطط البيان
