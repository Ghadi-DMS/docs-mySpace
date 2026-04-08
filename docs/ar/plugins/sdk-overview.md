---
read_when:
    - تحتاج إلى معرفة المسار الفرعي في SDK الذي يجب الاستيراد منه
    - تريد مرجعًا لجميع أساليب التسجيل على OpenClawPluginApi
    - تبحث عن تصدير محدد في SDK
sidebarTitle: SDK Overview
summary: خريطة الاستيراد، ومرجع API للتسجيل، وبنية SDK
title: نظرة عامة على Plugin SDK
x-i18n:
    generated_at: "2026-04-08T02:18:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: c5a41bd82d165dfbb7fbd6e4528cf322e9133a51efe55fa8518a7a0a626d9d30
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# نظرة عامة على Plugin SDK

Plugin SDK هو العقد المطبّع بين الإضافات والنواة. هذه الصفحة هي
المرجع الخاص بـ **ما الذي يجب استيراده** و**ما الذي يمكنك تسجيله**.

<Tip>
  **هل تبحث عن دليل عملي؟**
  - أول إضافة؟ ابدأ من [البدء](/ar/plugins/building-plugins)
  - إضافة قناة؟ راجع [إضافات القنوات](/ar/plugins/sdk-channel-plugins)
  - إضافة موفّر؟ راجع [إضافات الموفّرات](/ar/plugins/sdk-provider-plugins)
</Tip>

## اصطلاح الاستيراد

استورد دائمًا من مسار فرعي محدد:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

كل مسار فرعي هو وحدة صغيرة ومستقلة بذاتها. يحافظ هذا على سرعة بدء التشغيل
ويمنع مشكلات التبعيات الدائرية. بالنسبة إلى أدوات الإدخال/البناء الخاصة بالقنوات،
فضّل `openclaw/plugin-sdk/channel-core`؛ واحتفظ بـ `openclaw/plugin-sdk/core` من أجل
السطح الأشمل والمظلّي والمساعدات المشتركة مثل
`buildChannelConfigSchema`.

لا تضف ولا تعتمد على مسارات تسهيلية مسماة على اسم الموفّر مثل
`openclaw/plugin-sdk/slack` أو `openclaw/plugin-sdk/discord`،
أو `openclaw/plugin-sdk/signal` أو `openclaw/plugin-sdk/whatsapp`، أو
مسارات مساعدات تحمل علامة قناة محددة. يجب على الإضافات المضمّنة أن تركّب
مسارات SDK الفرعية العامة داخل ملفات `api.ts` أو `runtime-api.ts` الخاصة بها، ويجب على النواة
إما استخدام هذه الملفات المحلية للإضافة أو إضافة عقد SDK عام ضيّق
عندما تكون الحاجة عابرة للقنوات فعلًا.

لا تزال خريطة التصدير المولدة تحتوي على مجموعة صغيرة من
مسارات المساعدات الخاصة بالإضافات المضمّنة مثل `plugin-sdk/feishu` و`plugin-sdk/feishu-setup`،
و`plugin-sdk/zalo` و`plugin-sdk/zalo-setup` و`plugin-sdk/matrix*`. هذه
المسارات الفرعية موجودة فقط لصيانة الإضافات المضمّنة والتوافق؛ وهي
مستبعدة عمدًا من الجدول الشائع أدناه وليست مسار الاستيراد
الموصى به للإضافات الخارجية الجديدة.

## مرجع المسارات الفرعية

أكثر المسارات الفرعية استخدامًا، مجمّعة حسب الغرض. القائمة الكاملة المولدة والتي تضم
أكثر من 200 مسار فرعي موجودة في `scripts/lib/plugin-sdk-entrypoints.json`.

لا تزال المسارات الفرعية المحجوزة الخاصة بمساعدات الإضافات المضمّنة تظهر في تلك القائمة المولدة.
تعامل مع هذه المسارات على أنها تفاصيل تنفيذ/أسطح توافق إلا إذا
روّجت لها صفحة توثيق صراحة على أنها عامة.

### إدخال الإضافة

| المسار الفرعي | الصادرات الأساسية |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="المسارات الفرعية للقنوات">
    | المسار الفرعي | الصادرات الأساسية |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | تصدير مخطط Zod الجذر لـ `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`، بالإضافة إلى `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | مساعدات معالج الإعداد المشتركة، ومطالبات قائمة السماح، وبناة حالة الإعداد |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | مساعدات تكوين/بوابة إجراء الحسابات المتعددة ومساعدات الرجوع إلى الحساب الافتراضي |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`، ومساعدات تطبيع معرّف الحساب |
    | `plugin-sdk/account-resolution` | مساعدات البحث عن الحساب + الرجوع إلى الافتراضي |
    | `plugin-sdk/account-helpers` | مساعدات ضيقة لقائمة الحسابات/إجراءات الحساب |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | أنواع مخطط تكوين القناة |
    | `plugin-sdk/telegram-command-config` | مساعدات تطبيع/تحقق للأوامر المخصصة في Telegram مع رجوع إلى العقدة المضمّنة |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | مساعدات بناء المسار والمغلف المشترك للرسائل الواردة |
    | `plugin-sdk/inbound-reply-dispatch` | مساعدات التسجيل والإرسال المشتركة للرسائل الواردة |
    | `plugin-sdk/messaging-targets` | مساعدات تحليل/مطابقة الأهداف |
    | `plugin-sdk/outbound-media` | مساعدات تحميل الوسائط الصادرة المشتركة |
    | `plugin-sdk/outbound-runtime` | مساعدات هوية/تفويض الإرسال الصادر |
    | `plugin-sdk/thread-bindings-runtime` | مساعدات دورة حياة وموائمات ربط الخيوط |
    | `plugin-sdk/agent-media-payload` | باني حمولة وسائط الوكيل القديم |
    | `plugin-sdk/conversation-runtime` | مساعدات المحادثة/ربط الخيوط/الإقران/الربط المهيأ |
    | `plugin-sdk/runtime-config-snapshot` | مساعد أخذ لقطة من تكوين وقت التشغيل |
    | `plugin-sdk/runtime-group-policy` | مساعدات حل سياسة المجموعة في وقت التشغيل |
    | `plugin-sdk/channel-status` | مساعدات اللقطة/الملخص المشتركة لحالة القناة |
    | `plugin-sdk/channel-config-primitives` | بدائيات ضيقة لمخطط تكوين القناة |
    | `plugin-sdk/channel-config-writes` | مساعدات تفويض كتابة تكوين القناة |
    | `plugin-sdk/channel-plugin-common` | صادرات تمهيدية مشتركة لإضافات القنوات |
    | `plugin-sdk/allowlist-config-edit` | مساعدات قراءة/تحرير تكوين قائمة السماح |
    | `plugin-sdk/group-access` | مساعدات مشتركة لاتخاذ قرار وصول المجموعة |
    | `plugin-sdk/direct-dm` | مساعدات مشتركة لمصادقة/حماية الرسائل المباشرة المباشرة |
    | `plugin-sdk/interactive-runtime` | مساعدات تطبيع/اختزال حمولة الرد التفاعلي |
    | `plugin-sdk/channel-inbound` | مساعدات إزالة الارتداد، ومطابقة الذكر، وسياسة الذكر للرسائل الواردة، ومساعدات المغلف |
    | `plugin-sdk/channel-send-result` | أنواع نتيجة الرد |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | مساعدات تحليل/مطابقة الأهداف |
    | `plugin-sdk/channel-contract` | أنواع عقد القناة |
    | `plugin-sdk/channel-feedback` | أسلاك التغذية الراجعة/التفاعلات |
    | `plugin-sdk/channel-secret-runtime` | مساعدات ضيقة لعقد الأسرار مثل `collectSimpleChannelFieldAssignments` و`getChannelSurface` و`pushAssignment` وأنواع أهداف الأسرار |
  </Accordion>

  <Accordion title="المسارات الفرعية للموفّرات">
    | المسار الفرعي | الصادرات الأساسية |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | مساعدات منسقة لإعداد الموفّرات المحلية/المستضافة ذاتيًا |
    | `plugin-sdk/self-hosted-provider-setup` | مساعدات إعداد مركزة لموفّرات مستضافة ذاتيًا متوافقة مع OpenAI |
    | `plugin-sdk/cli-backend` | القيم الافتراضية للواجهة الخلفية لـ CLI + ثوابت المراقبة |
    | `plugin-sdk/provider-auth-runtime` | مساعدات حل مفاتيح API وقت التشغيل لإضافات الموفّرات |
    | `plugin-sdk/provider-auth-api-key` | مساعدات الإلحاق/الكتابة في الملف الشخصي لمفتاح API مثل `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | باني نتيجة مصادقة OAuth القياسي |
    | `plugin-sdk/provider-auth-login` | مساعدات تسجيل الدخول التفاعلية المشتركة لإضافات الموفّرات |
    | `plugin-sdk/provider-env-vars` | مساعدات البحث عن متغيرات بيئة مصادقة الموفّر |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`، وبناة سياسة الإعادة المشتركة، ومساعدات نقطة نهاية الموفّر، ومساعدات تطبيع معرّف النموذج مثل `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | مساعدات عامة لقدرات HTTP/نقطة النهاية الخاصة بالموفّر |
    | `plugin-sdk/provider-web-fetch-contract` | مساعدات ضيقة لعقود تكوين/اختيار web-fetch مثل `enablePluginInConfig` و`WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | مساعدات تسجيل/ذاكرة التخزين المؤقت لموفّر web-fetch |
    | `plugin-sdk/provider-web-search-contract` | مساعدات ضيقة لعقود تكوين/بيانات اعتماد web-search مثل `enablePluginInConfig` و`resolveProviderWebSearchPluginConfig` وضبط/جلب بيانات الاعتماد ذات النطاق |
    | `plugin-sdk/provider-web-search` | مساعدات التسجيل/الذاكرة المؤقتة/وقت التشغيل لموفّر web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`، وتنظيف مخطط Gemini + التشخيصات، ومساعدات التوافق مع xAI مثل `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` وما شابهها |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`، وأنواع مغطيات البث، ومساعدات المغطيات المشتركة لـ Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | مساعدات ترقيع تكوين الإعداد الأولي |
    | `plugin-sdk/global-singleton` | مساعدات القيم المفردة/الخرائط/الذاكرات المؤقتة المحلية للعملية |
  </Accordion>

  <Accordion title="المسارات الفرعية للمصادقة والأمان">
    | المسار الفرعي | الصادرات الأساسية |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`، ومساعدات سجل الأوامر، ومساعدات تفويض المرسِل |
    | `plugin-sdk/approval-auth-runtime` | مساعدات حل الموافقين ومصادقة الإجراء في نفس الدردشة |
    | `plugin-sdk/approval-client-runtime` | مساعدات الملف الشخصي/المرشح للموافقات الأصلية لـ exec |
    | `plugin-sdk/approval-delivery-runtime` | موائمات قدرات/تسليم الموافقات الأصلية |
    | `plugin-sdk/approval-gateway-runtime` | مساعد مشترك لحل بوابة الموافقات |
    | `plugin-sdk/approval-handler-adapter-runtime` | مساعدات خفيفة لتحميل موائم الموافقات الأصلية لنقاط إدخال القنوات الساخنة |
    | `plugin-sdk/approval-handler-runtime` | مساعدات أوسع لوقت تشغيل معالج الموافقات؛ فضّل المسارات الأضيق الخاصة بالموائم/البوابة عندما تكون كافية |
    | `plugin-sdk/approval-native-runtime` | مساعدات الهدف الأصلي للموافقات + ربط الحساب |
    | `plugin-sdk/approval-reply-runtime` | مساعدات حمولة رد موافقات exec/الإضافة |
    | `plugin-sdk/command-auth-native` | مساعدات مصادقة الأوامر الأصلية + أهداف الجلسات الأصلية |
    | `plugin-sdk/command-detection` | مساعدات مشتركة لاكتشاف الأوامر |
    | `plugin-sdk/command-surface` | تطبيع جسم الأمر ومساعدات سطح الأوامر |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | مساعدات ضيقة لتجميع عقود الأسرار لأسطح أسرار القناة/الإضافة |
    | `plugin-sdk/secret-ref-runtime` | مساعدات ضيقة لـ `coerceSecretRef` وكتابة SecretRef لتحليل عقود الأسرار/التكوين |
    | `plugin-sdk/security-runtime` | مساعدات مشتركة للثقة، وبوابات الرسائل المباشرة، والمحتوى الخارجي، وتجميع الأسرار |
    | `plugin-sdk/ssrf-policy` | مساعدات قائمة السماح للمضيف وسياسة SSRF للشبكات الخاصة |
    | `plugin-sdk/ssrf-runtime` | مساعدات المرسِل المثبّت، والجلب المحمي بـ SSRF، وسياسة SSRF |
    | `plugin-sdk/secret-input` | مساعدات تحليل إدخال الأسرار |
    | `plugin-sdk/webhook-ingress` | مساعدات طلب/هدف Webhook |
    | `plugin-sdk/webhook-request-guards` | مساعدات حجم جسم الطلب/المهلة الزمنية |
  </Accordion>

  <Accordion title="المسارات الفرعية لوقت التشغيل والتخزين">
    | المسار الفرعي | الصادرات الأساسية |
    | --- | --- |
    | `plugin-sdk/runtime` | مساعدات عامة لوقت التشغيل/السجلات/النسخ الاحتياطي/تثبيت الإضافات |
    | `plugin-sdk/runtime-env` | مساعدات ضيقة لبيئة وقت التشغيل، والمسجل، والمهلة، وإعادة المحاولة، وbackoff |
    | `plugin-sdk/channel-runtime-context` | مساعدات عامة لتسجيل والبحث عن سياق وقت تشغيل القناة |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | مساعدات مشتركة لأوامر/خطافات/http/التفاعل الخاصة بالإضافةات |
    | `plugin-sdk/hook-runtime` | مساعدات مشتركة لمسار Webhook/الخطافات الداخلية |
    | `plugin-sdk/lazy-runtime` | مساعدات الاستيراد/الربط الكسول لوقت التشغيل مثل `createLazyRuntimeModule` و`createLazyRuntimeMethod` و`createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | مساعدات تنفيذ العملية |
    | `plugin-sdk/cli-runtime` | مساعدات التنسيق والانتظار والإصدار لـ CLI |
    | `plugin-sdk/gateway-runtime` | مساعدات عميل البوابة وترقيعات حالة القناة |
    | `plugin-sdk/config-runtime` | مساعدات تحميل/كتابة التكوين |
    | `plugin-sdk/telegram-command-config` | تطبيع اسم/وصف أمر Telegram وفحوصات التكرار/التعارض، حتى عند غياب سطح عقد Telegram المضمّن |
    | `plugin-sdk/approval-runtime` | مساعدات موافقات exec/الإضافة، وبناة قدرات الموافقة، ومساعدات المصادقة/الملف الشخصي، ومساعدات التوجيه/وقت التشغيل الأصلية |
    | `plugin-sdk/reply-runtime` | مساعدات مشتركة لوقت تشغيل الوارد/الرد، والتجزئة، والإرسال، والنبض، ومخطط الرد |
    | `plugin-sdk/reply-dispatch-runtime` | مساعدات ضيقة لإرسال/إنهاء الرد |
    | `plugin-sdk/reply-history` | مساعدات مشتركة لسجل الردود ذي النافذة القصيرة مثل `buildHistoryContext` و`recordPendingHistoryEntry` و`clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | مساعدات ضيقة لتجزئة النص/Markdown |
    | `plugin-sdk/session-store-runtime` | مساعدات مسار مخزن الجلسة و`updated-at` |
    | `plugin-sdk/state-paths` | مساعدات مسارات الحالة/دليل OAuth |
    | `plugin-sdk/routing` | مساعدات المسار/مفتاح الجلسة/ربط الحساب مثل `resolveAgentRoute` و`buildAgentSessionKey` و`resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | مساعدات مشتركة لملخص حالة القناة/الحساب، والقيم الافتراضية لحالة وقت التشغيل، ومساعدات بيانات تعريف المشكلات |
    | `plugin-sdk/target-resolver-runtime` | مساعدات مشتركة لحل الأهداف |
    | `plugin-sdk/string-normalization-runtime` | مساعدات تطبيع السلاسل/slug |
    | `plugin-sdk/request-url` | استخراج عناوين URL النصية من مدخلات شبيهة بـ fetch/request |
    | `plugin-sdk/run-command` | مشغل أوامر مؤقت بنتائج stdout/stderr مطبعة |
    | `plugin-sdk/param-readers` | قارئات معلمات شائعة للأدوات/CLI |
    | `plugin-sdk/tool-send` | استخراج حقول هدف الإرسال القياسية من وسائط الأداة |
    | `plugin-sdk/temp-path` | مساعدات مشتركة لمسار التنزيل المؤقت |
    | `plugin-sdk/logging-core` | مساعدات المسجل الفرعي والتنقيح |
    | `plugin-sdk/markdown-table-runtime` | مساعدات أوضاع جداول Markdown |
    | `plugin-sdk/json-store` | مساعدات صغيرة لقراءة/كتابة حالة JSON |
    | `plugin-sdk/file-lock` | مساعدات إعادة الدخول لقفل الملفات |
    | `plugin-sdk/persistent-dedupe` | مساعدات ذاكرة التخزين المؤقت لإزالة التكرار المدعومة بالقرص |
    | `plugin-sdk/acp-runtime` | مساعدات ACP وقت التشغيل/الجلسة وإرسال الرد |
    | `plugin-sdk/agent-config-primitives` | بدائيات ضيقة لمخطط تكوين وقت تشغيل الوكيل |
    | `plugin-sdk/boolean-param` | قارئ معلمات منطقي مرن |
    | `plugin-sdk/dangerous-name-runtime` | مساعدات حل مطابقة الأسماء الخطرة |
    | `plugin-sdk/device-bootstrap` | مساعدات bootstrap للجهاز ورمز الإقران |
    | `plugin-sdk/extension-shared` | بدائيات مشتركة لمساعدات القنوات السلبية، والحالة، والوكيل المحيط |
    | `plugin-sdk/models-provider-runtime` | مساعدات أوامر `/models` / ردود الموفّر |
    | `plugin-sdk/skill-commands-runtime` | مساعدات سرد أوامر Skills |
    | `plugin-sdk/native-command-registry` | مساعدات سجل/بناء/تسلسل الأوامر الأصلية |
    | `plugin-sdk/provider-zai-endpoint` | مساعدات اكتشاف نقطة نهاية Z.A.I |
    | `plugin-sdk/infra-runtime` | مساعدات أحداث النظام/النبض |
    | `plugin-sdk/collection-runtime` | مساعدات صغيرة لذاكرة تخزين مؤقتة محدودة |
    | `plugin-sdk/diagnostic-runtime` | مساعدات أعلام التشخيص والأحداث |
    | `plugin-sdk/error-runtime` | مساعدات رسم الأخطاء، والتنسيق، وتصنيف الأخطاء المشتركة، و`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | مساعدات fetch المغلّفة، والوكيل، والبحث المثبّت |
    | `plugin-sdk/host-runtime` | مساعدات تطبيع اسم المضيف ومضيف SCP |
    | `plugin-sdk/retry-runtime` | مساعدات تكوين إعادة المحاولة ومشغّل إعادة المحاولة |
    | `plugin-sdk/agent-runtime` | مساعدات دليل/هوية/مساحة عمل الوكيل |
    | `plugin-sdk/directory-runtime` | استعلام/إزالة تكرار الدليل المدعوم بالتكوين |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="المسارات الفرعية للقدرات والاختبار">
    | المسار الفرعي | الصادرات الأساسية |
    | --- | --- |
    | `plugin-sdk/media-runtime` | مساعدات مشتركة لجلب/تحويل/تخزين الوسائط بالإضافة إلى بناة حمولات الوسائط |
    | `plugin-sdk/media-generation-runtime` | مساعدات مشتركة للإخفاق المرحلي في توليد الوسائط، واختيار المرشحين، ورسائل النماذج المفقودة |
    | `plugin-sdk/media-understanding` | أنواع موفّر فهم الوسائط بالإضافة إلى صادرات مساعدات الصور/الصوت المواجهة للموفّر |
    | `plugin-sdk/text-runtime` | مساعدات مشتركة للنص/Markdown/السجلات مثل إزالة النص المرئي للمساعد، ومساعدات عرض/تجزئة/جداول Markdown، ومساعدات التنقيح، ووسوم التوجيه، وأدوات النص الآمن |
    | `plugin-sdk/text-chunking` | مساعد تجزئة النص الصادر |
    | `plugin-sdk/speech` | أنواع موفّر الكلام بالإضافة إلى مساعدات التوجيه، والسجل، والتحقق المواجهة للموفّر |
    | `plugin-sdk/speech-core` | أنواع موفّر الكلام المشتركة، والسجل، والتوجيه، ومساعدات التطبيع |
    | `plugin-sdk/realtime-transcription` | أنواع موفّر النسخ الفوري ومساعدات السجل |
    | `plugin-sdk/realtime-voice` | أنواع موفّر الصوت الفوري ومساعدات السجل |
    | `plugin-sdk/image-generation` | أنواع موفّر توليد الصور |
    | `plugin-sdk/image-generation-core` | أنواع توليد الصور المشتركة، والإخفاق المرحلي، والمصادقة، ومساعدات السجل |
    | `plugin-sdk/music-generation` | أنواع موفّر/طلب/نتيجة توليد الموسيقى |
    | `plugin-sdk/music-generation-core` | أنواع توليد الموسيقى المشتركة، ومساعدات الإخفاق المرحلي، والبحث عن الموفّر، وتحليل مرجع النموذج |
    | `plugin-sdk/video-generation` | أنواع موفّر/طلب/نتيجة توليد الفيديو |
    | `plugin-sdk/video-generation-core` | أنواع توليد الفيديو المشتركة، ومساعدات الإخفاق المرحلي، والبحث عن الموفّر، وتحليل مرجع النموذج |
    | `plugin-sdk/webhook-targets` | سجل أهداف Webhook ومساعدات تثبيت المسارات |
    | `plugin-sdk/webhook-path` | مساعدات تطبيع مسار Webhook |
    | `plugin-sdk/web-media` | مساعدات مشتركة لتحميل الوسائط البعيدة/المحلية |
    | `plugin-sdk/zod` | إعادة تصدير `zod` لمستهلكي Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="المسارات الفرعية للذاكرة">
    | المسار الفرعي | الصادرات الأساسية |
    | --- | --- |
    | `plugin-sdk/memory-core` | سطح مساعد memory-core المضمّن لمساعدات المدير/التكوين/الملف/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | واجهة وقت التشغيل لفهرسة/بحث الذاكرة |
    | `plugin-sdk/memory-core-host-engine-foundation` | صادرات محرك الأساس لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-embeddings` | صادرات محرك embeddings لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-qmd` | صادرات محرك QMD لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-storage` | صادرات محرك التخزين لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-multimodal` | مساعدات الوسائط المتعددة لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-query` | مساعدات استعلام مضيف الذاكرة |
    | `plugin-sdk/memory-core-host-secret` | مساعدات الأسرار لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-events` | مساعدات دفتر أحداث مضيف الذاكرة |
    | `plugin-sdk/memory-core-host-status` | مساعدات حالة مضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-cli` | مساعدات CLI وقت التشغيل لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-core` | مساعدات وقت التشغيل الأساسية لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-files` | مساعدات الملفات/وقت التشغيل لمضيف الذاكرة |
    | `plugin-sdk/memory-host-core` | اسم مستعار محايد للمورّد لمساعدات وقت التشغيل الأساسية لمضيف الذاكرة |
    | `plugin-sdk/memory-host-events` | اسم مستعار محايد للمورّد لمساعدات دفتر أحداث مضيف الذاكرة |
    | `plugin-sdk/memory-host-files` | اسم مستعار محايد للمورّد لمساعدات الملفات/وقت التشغيل لمضيف الذاكرة |
    | `plugin-sdk/memory-host-markdown` | مساعدات Markdown المُدار المشتركة للإضافات المجاورة للذاكرة |
    | `plugin-sdk/memory-host-search` | واجهة وقت تشغيل الذاكرة النشطة للوصول إلى مدير البحث |
    | `plugin-sdk/memory-host-status` | اسم مستعار محايد للمورّد لمساعدات حالة مضيف الذاكرة |
    | `plugin-sdk/memory-lancedb` | سطح مساعد memory-lancedb المضمّن |
  </Accordion>

  <Accordion title="المسارات الفرعية المحجوزة للمساعدات المضمّنة">
    | العائلة | المسارات الفرعية الحالية | الاستخدام المقصود |
    | --- | --- | --- |
    | المتصفح | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | مساعدات دعم إضافة المتصفح المضمّنة (`browser-support` يبقى الشريط المتوافق) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | سطح مساعد/وقت تشغيل Matrix المضمّن |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | سطح مساعد/وقت تشغيل LINE المضمّن |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | سطح مساعد IRC المضمّن |
    | مساعدات خاصة بالقنوات | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | مسارات توافق/مساعدات القنوات المضمّنة |
    | مساعدات خاصة بالمصادقة/الإضافة | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | مسارات مساعدات الميزات/الإضافات المضمّنة؛ يصدّر `plugin-sdk/github-copilot-token` حاليًا `DEFAULT_COPILOT_API_BASE_URL` و`deriveCopilotApiBaseUrlFromToken` و`resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API التسجيل

يتلقى استدعاء `register(api)` كائن `OpenClawPluginApi` بهذه
الأساليب:

### تسجيل القدرات

| الأسلوب | ما الذي يسجله |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | استدلال النص (LLM)             |
| `api.registerCliBackend(...)`                    | واجهة خلفية محلية للاستدلال عبر CLI      |
| `api.registerChannel(...)`                       | قناة مراسلة                |
| `api.registerSpeechProvider(...)`                | تحويل النص إلى كلام / توليف STT   |
| `api.registerRealtimeTranscriptionProvider(...)` | نسخ فوري متدفق |
| `api.registerRealtimeVoiceProvider(...)`         | جلسات صوت فورية ثنائية الاتجاه   |
| `api.registerMediaUnderstandingProvider(...)`    | تحليل الصور/الصوت/الفيديو       |
| `api.registerImageGenerationProvider(...)`       | توليد الصور                 |
| `api.registerMusicGenerationProvider(...)`       | توليد الموسيقى                 |
| `api.registerVideoGenerationProvider(...)`       | توليد الفيديو                 |
| `api.registerWebFetchProvider(...)`              | موفّر Web fetch / scrape      |
| `api.registerWebSearchProvider(...)`             | البحث على الويب                       |

### الأدوات والأوامر

| الأسلوب | ما الذي يسجله |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | أداة وكيل (مطلوبة أو `{ optional: true }`) |
| `api.registerCommand(def)`      | أمر مخصص (يتجاوز LLM)             |

### البنية التحتية

| الأسلوب | ما الذي يسجله |
| ---------------------------------------------- | --------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | خطاف حدث                              |
| `api.registerHttpRoute(params)`                | نقطة نهاية HTTP للبوابة                   |
| `api.registerGatewayMethod(name, handler)`     | أسلوب RPC للبوابة                      |
| `api.registerCli(registrar, opts?)`            | أمر فرعي في CLI                          |
| `api.registerService(service)`                 | خدمة خلفية                      |
| `api.registerInteractiveHandler(registration)` | معالج تفاعلي                     |
| `api.registerMemoryPromptSupplement(builder)`  | قسم إضافي للمطالبات المجاورة للذاكرة |
| `api.registerMemoryCorpusSupplement(adapter)`  | corpus إضافي لبحث/قراءة الذاكرة      |

تظل مساحات أسماء الإدارة الأساسية المحجوزة (`config.*` و`exec.approvals.*` و`wizard.*`،
و`update.*`) دائمًا `operator.admin`، حتى إذا حاولت إضافة تعيين
نطاق أضيق لأسلوب البوابة. فضّل البادئات الخاصة بالإضافة
للأساليب التي تملكها الإضافة.

### بيانات تعريف تسجيل CLI

يقبل `api.registerCli(registrar, opts?)` نوعين من بيانات التعريف في المستوى الأعلى:

- `commands`: جذور أوامر صريحة يملكها المُسجِّل
- `descriptors`: واصفات أوامر وقت التحليل المستخدمة لمساعدة CLI الجذرية،
  والتوجيه، وتسجيل CLI الكسول للإضافة

إذا كنت تريد أن يظل أمر الإضافة محمّلًا بكسل في المسار الجذري العادي لـ CLI،
فقدّم `descriptors` تغطي كل جذر أمر من المستوى الأعلى يكشفه ذلك
المُسجِّل.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "إدارة حسابات Matrix، والتحقق، والأجهزة، وحالة الملف الشخصي",
        hasSubcommands: true,
      },
    ],
  },
);
```

استخدم `commands` وحده فقط عندما لا تحتاج إلى تسجيل CLI جذري كسول.
لا يزال هذا المسار التوافقي المتعجل مدعومًا، لكنه لا يثبت
عناصر نائبة مدعومة بالواصفات من أجل التحميل الكسول وقت التحليل.

### تسجيل الواجهة الخلفية لـ CLI

يتيح `api.registerCliBackend(...)` لإضافة أن تملك التكوين الافتراضي لواجهة خلفية
محلية لـ CLI للذكاء الاصطناعي مثل `codex-cli`.

- يصبح `id` الخاص بالواجهة الخلفية بادئة الموفّر في مراجع النماذج مثل `codex-cli/gpt-5`.
- يستخدم `config` الخاص بالواجهة الخلفية الشكل نفسه المستخدم في `agents.defaults.cliBackends.<id>`.
- يظل تكوين المستخدم هو الغالب. يدمج OpenClaw `agents.defaults.cliBackends.<id>` فوق
  القيمة الافتراضية للإضافة قبل تشغيل CLI.
- استخدم `normalizeConfig` عندما تحتاج الواجهة الخلفية إلى إعادة كتابة توافقية بعد الدمج
  (على سبيل المثال، تطبيع أشكال الأعلام القديمة).

### الفتحات الحصرية

| الأسلوب | ما الذي يسجله |
| ------------------------------------------ | ------------------------------------- |
| `api.registerContextEngine(id, factory)`   | محرك السياق (واحد نشط في كل مرة) |
| `api.registerMemoryCapability(capability)` | قدرة الذاكرة الموحدة             |
| `api.registerMemoryPromptSection(builder)` | باني قسم مطالبة الذاكرة         |
| `api.registerMemoryFlushPlan(resolver)`    | محلّل خطة تفريغ الذاكرة            |
| `api.registerMemoryRuntime(runtime)`       | موائم وقت تشغيل الذاكرة                |

### موائمات تضمين الذاكرة

| الأسلوب | ما الذي يسجله |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | موائم تضمين ذاكرة للإضافة النشطة |

- `registerMemoryCapability` هو API الإضافة الحصرية المفضلة الخاصة بذاكرة الذاكرة.
- قد يكشف `registerMemoryCapability` أيضًا `publicArtifacts.listArtifacts(...)`
  بحيث تتمكن الإضافات المرافقة من استهلاك مصنوعات الذاكرة المصدّرة عبر
  `openclaw/plugin-sdk/memory-host-core` بدلًا من الوصول إلى التخطيط الخاص
  لإضافة ذاكرة محددة.
- `registerMemoryPromptSection` و`registerMemoryFlushPlan` و
  `registerMemoryRuntime` هي APIs حصرية متوافقة مع الإصدارات القديمة لإضافات الذاكرة.
- يتيح `registerMemoryEmbeddingProvider` لإضافة الذاكرة النشطة تسجيل
  معرّف واحد أو أكثر لموائمات التضمين (مثل `openai` أو `gemini` أو معرّف
  مخصص تعرّفه الإضافة).
- يُحل تكوين المستخدم مثل `agents.defaults.memorySearch.provider` و
  `agents.defaults.memorySearch.fallback` مقابل معرّفات الموائمات المسجلة هذه.

### الأحداث ودورة الحياة

| الأسلوب | ما الذي يفعله |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | خطاف دورة حياة مطبّع          |
| `api.onConversationBindingResolved(handler)` | استدعاء رد رابط المحادثة |

### دلالات قرارات الخطافات

- `before_tool_call`: إرجاع `{ block: true }` نهائي. بمجرد أن يضبطه أي معالج، يتم تخطي المعالجات ذات الأولوية الأقل.
- `before_tool_call`: إرجاع `{ block: false }` يُعامل على أنه لا قرار (مثل حذف `block`)، وليس كتجاوز.
- `before_install`: إرجاع `{ block: true }` نهائي. بمجرد أن يضبطه أي معالج، يتم تخطي المعالجات ذات الأولوية الأقل.
- `before_install`: إرجاع `{ block: false }` يُعامل على أنه لا قرار (مثل حذف `block`)، وليس كتجاوز.
- `reply_dispatch`: إرجاع `{ handled: true, ... }` نهائي. بمجرد أن يطالب أي معالج بالإرسال، يتم تخطي المعالجات ذات الأولوية الأقل ومسار إرسال النموذج الافتراضي.
- `message_sending`: إرجاع `{ cancel: true }` نهائي. بمجرد أن يضبطه أي معالج، يتم تخطي المعالجات ذات الأولوية الأقل.
- `message_sending`: إرجاع `{ cancel: false }` يُعامل على أنه لا قرار (مثل حذف `cancel`)، وليس كتجاوز.

### حقول كائن API

| الحقل | النوع | الوصف |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | معرّف الإضافة                                                                                   |
| `api.name`               | `string`                  | اسم العرض                                                                                |
| `api.version`            | `string?`                 | إصدار الإضافة (اختياري)                                                                   |
| `api.description`        | `string?`                 | وصف الإضافة (اختياري)                                                               |
| `api.source`             | `string`                  | مسار مصدر الإضافة                                                                          |
| `api.rootDir`            | `string?`                 | الدليل الجذري للإضافة (اختياري)                                                            |
| `api.config`             | `OpenClawConfig`          | لقطة التكوين الحالية (لقطة وقت التشغيل النشطة داخل الذاكرة عند توفرها)                  |
| `api.pluginConfig`       | `Record<string, unknown>` | تكوين خاص بالإضافة من `plugins.entries.<id>.config`                                   |
| `api.runtime`            | `PluginRuntime`           | [مساعدات وقت التشغيل](/ar/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | مسجّل ذو نطاق (`debug`, `info`, `warn`, `error`)                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | وضع التحميل الحالي؛ `"setup-runtime"` هي نافذة بدء التشغيل/الإعداد الخفيفة قبل الإدخال الكامل |
| `api.resolvePath(input)` | `(string) => string`      | حل المسار نسبةً إلى جذر الإضافة                                                        |

## اصطلاح الوحدات الداخلية

داخل إضافتك، استخدم ملفات barrel محلية للاستيرادات الداخلية:

```
my-plugin/
  api.ts            # الصادرات العامة للمستهلكين الخارجيين
  runtime-api.ts    # صادرات وقت تشغيل داخلية فقط
  index.ts          # نقطة إدخال الإضافة
  setup-entry.ts    # إدخال خفيف للإعداد فقط (اختياري)
```

<Warning>
  لا تستورد إضافتك الخاصة أبدًا من خلال `openclaw/plugin-sdk/<your-plugin>`
  من كود الإنتاج. وجّه الاستيرادات الداخلية عبر `./api.ts` أو
  `./runtime-api.ts`. مسار SDK هو العقد الخارجي فقط.
</Warning>

تفضّل الآن الأسطح العامة للإضافات المضمّنة المحمّلة عبر الواجهة (`api.ts` و`runtime-api.ts`،
و`index.ts` و`setup-entry.ts`، وملفات الإدخال العامة المشابهة)
لقطة تكوين وقت التشغيل النشطة عندما يكون OpenClaw قيد التشغيل بالفعل. وإذا لم تكن
لقطة وقت التشغيل موجودة بعد، فإنها ترجع إلى ملف التكوين المحلول على القرص.

يمكن لإضافات الموفّرات أيضًا كشف ملف barrel محلي ضيق خاص بالإضافة عندما يكون
المساعد خاصًا بالموفّر عمدًا ولا ينتمي بعد إلى مسار فرعي عام في SDK.
المثال المضمّن الحالي: يحتفظ موفّر Anthropic بمساعدات بث Claude
الخاصة به في سطحه العام `api.ts` / `contract-api.ts` بدلًا من
ترقية منطق رؤوس Anthropic التجريبية و`service_tier` إلى عقد
عام من `plugin-sdk/*`.

أمثلة مضمّنة حالية أخرى:

- `@openclaw/openai-provider`: يصدّر `api.ts` بُناة الموفّر،
  ومساعدات النموذج الافتراضي، وبناة الموفّر الفوري
- `@openclaw/openrouter-provider`: يصدّر `api.ts` باني الموفّر بالإضافة إلى
  مساعدات الإعداد الأولي/التكوين

<Warning>
  يجب أيضًا على كود الإنتاج في الامتدادات تجنّب الاستيراد من `openclaw/plugin-sdk/<other-plugin>`.
  إذا كان المساعد مشتركًا فعلًا، فقم بترقيته إلى مسار فرعي محايد في SDK
  مثل `openclaw/plugin-sdk/speech` أو `.../provider-model-shared` أو سطح آخر
  موجّه للقدرات بدلًا من ربط إضافتين معًا.
</Warning>

## ذو صلة

- [نقاط الإدخال](/ar/plugins/sdk-entrypoints) — خيارات `definePluginEntry` و`defineChannelPluginEntry`
- [مساعدات وقت التشغيل](/ar/plugins/sdk-runtime) — المرجع الكامل لمساحة الاسم `api.runtime`
- [الإعداد والتكوين](/ar/plugins/sdk-setup) — الحزم، وmanifestات، ومخططات التكوين
- [الاختبار](/ar/plugins/sdk-testing) — أدوات الاختبار وقواعد lint
- [ترحيل SDK](/ar/plugins/sdk-migration) — الترحيل من الأسطح القديمة المهملة
- [الداخلية الخاصة بالإضافات](/ar/plugins/architecture) — بنية عميقة ونموذج القدرات
