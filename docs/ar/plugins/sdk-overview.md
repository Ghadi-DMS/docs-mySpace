---
read_when:
    - تحتاج إلى معرفة مسار SDK الفرعي الذي يجب الاستيراد منه
    - تريد مرجعًا لجميع طرق التسجيل في OpenClawPluginApi
    - أنت تبحث عن تصدير SDK محدد
sidebarTitle: SDK Overview
summary: مرجع خريطة الاستيراد وAPI التسجيل وبنية SDK
title: نظرة عامة على Plugin SDK
x-i18n:
    generated_at: "2026-04-05T12:52:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0d7d8b6add0623766d36e81588ae783b525357b2f5245c38c8e2b07c5fc1d2b5
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# نظرة عامة على Plugin SDK

تُعد Plugin SDK العقد المطبوع بين plugins والنواة. وهذه الصفحة هي
المرجع الخاص بـ **ما الذي يجب استيراده** و**ما الذي يمكنك تسجيله**.

<Tip>
  **هل تبحث عن دليل إرشادي؟**
  - أول plugin؟ ابدأ من [البدء](/plugins/building-plugins)
  - plugin قناة؟ راجع [Plugins القنوات](/plugins/sdk-channel-plugins)
  - plugin مزوّد؟ راجع [Plugins المزوّد](/plugins/sdk-provider-plugins)
</Tip>

## أسلوب الاستيراد

استورد دائمًا من مسار فرعي محدد:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

كل مسار فرعي هو وحدة صغيرة مستقلة بذاتها. وهذا يحافظ على سرعة بدء التشغيل
ويمنع مشكلات الاعتماد الدائري. وبالنسبة إلى مساعدات entry/build الخاصة بالقنوات،
ففضّل `openclaw/plugin-sdk/channel-core`؛ واحتفظ بـ `openclaw/plugin-sdk/core` من أجل
السطح الأوسع والمساعدات المشتركة مثل
`buildChannelConfigSchema`.

لا تضف أو تعتمد على مسارات ملائمة مسماة باسم المزوّد مثل
`openclaw/plugin-sdk/slack` أو `openclaw/plugin-sdk/discord` أو
`openclaw/plugin-sdk/signal` أو `openclaw/plugin-sdk/whatsapp` أو
مسارات مساعدة تحمل علامة القناة التجارية. يجب أن تقوم Plugins المضمّنة بتركيب
المسارات الفرعية العامة لـ SDK داخل ملفات `api.ts` أو `runtime-api.ts` الخاصة بها، ويجب على النواة
إما استخدام هذه الملفات المحلية الخاصة بـ plugin أو إضافة عقد SDK عام ضيق
عندما تكون الحاجة عابرة للقنوات فعلًا.

ما تزال خريطة التصدير المُولدة تحتوي على مجموعة صغيرة من
مسارات المساعدة الخاصة بـ plugins المضمّنة مثل `plugin-sdk/feishu` و`plugin-sdk/feishu-setup` و`plugin-sdk/zalo` و`plugin-sdk/zalo-setup` و`plugin-sdk/matrix*`. هذه
المسارات الفرعية موجودة لصيانة plugins المضمّنة والتوافق فقط؛ وتم حذفها عمدًا من الجدول الشائع أدناه وليست مسار الاستيراد الموصى به لـ plugins الخارجية الجديدة.

## مرجع المسارات الفرعية

أكثر المسارات الفرعية استخدامًا، مجمعة حسب الغرض. توجد القائمة الكاملة
المولدة التي تضم أكثر من 200 مسار فرعي في `scripts/lib/plugin-sdk-entrypoints.json`.

ما تزال المسارات الفرعية المحجوزة لمساعدات plugins المضمّنة تظهر في تلك القائمة المولدة.
تعامل معها على أنها تفاصيل تنفيذ/أسطح توافق ما لم
تروّج صفحة توثيق لها صراحةً على أنها عامة.

### إدخال plugin

| المسار الفرعي                | أهم التصديرات                                                                                                                       |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`    | `definePluginEntry`                                                                                                                 |
| `plugin-sdk/core`            | `defineChannelPluginEntry` و`createChatChannelPlugin` و`createChannelPluginBase` و`defineSetupPluginEntry` و`buildChannelConfigSchema` |
| `plugin-sdk/config-schema`   | `OpenClawSchema`                                                                                                                    |
| `plugin-sdk/provider-entry`  | `defineSingleProviderPluginEntry`                                                                                                   |

<AccordionGroup>
  <Accordion title="المسارات الفرعية للقنوات">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry` و`defineSetupPluginEntry` و`createChatChannelPlugin` و`createChannelPluginBase` |
    | `plugin-sdk/config-schema` | تصدير مخطط Zod الجذري لـ `openclaw.json` ‏(`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface` و`createOptionalChannelSetupAdapter` و`createOptionalChannelSetupWizard`، بالإضافة إلى `DEFAULT_ACCOUNT_ID` و`createTopLevelChannelDmPolicy` و`setSetupChannelEnabled` و`splitSetupEntries` |
    | `plugin-sdk/setup` | مساعدات مشتركة لمعالج الإعداد، ومطالبات allowlist، وبناة حالة الإعداد |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter` و`createEnvPatchedAccountSetupAdapter` و`createSetupInputPresenceValidator` و`noteChannelLookupFailure` و`noteChannelLookupSummary` و`promptResolvedAllowFrom` و`splitSetupEntries` و`createAllowlistSetupWizardProxy` و`createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand` و`detectBinary` و`extractArchive` و`resolveBrewExecutable` و`formatDocsLink` و`CONFIG_DIR` |
    | `plugin-sdk/account-core` | مساعدات تكوين/بوابات إجراءات متعددة الحسابات ومساعدات الرجوع إلى الحساب الافتراضي |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID` ومساعدات تطبيع معرّف الحساب |
    | `plugin-sdk/account-resolution` | مساعدات البحث عن الحساب + الرجوع إلى الافتراضي |
    | `plugin-sdk/account-helpers` | مساعدات ضيقة لقائمة الحسابات/إجراءات الحساب |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | أنواع مخطط تكوين القناة |
    | `plugin-sdk/telegram-command-config` | مساعدات تطبيع/تحقق لأوامر Telegram المخصصة مع بديل العقد المضمّن |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | مساعدات مشتركة للمسار الوارد وبناء الغلاف |
    | `plugin-sdk/inbound-reply-dispatch` | مساعدات مشتركة لتسجيل الوارد وإرساله |
    | `plugin-sdk/messaging-targets` | مساعدات تحليل/مطابقة الأهداف |
    | `plugin-sdk/outbound-media` | مساعدات مشتركة لتحميل الوسائط الصادرة |
    | `plugin-sdk/outbound-runtime` | مساعدات هوية/تفويض الإرسال الصادر |
    | `plugin-sdk/thread-bindings-runtime` | مساعدات دورة حياة ارتباطات السلاسل والمهايئ |
    | `plugin-sdk/agent-media-payload` | باني حمولة وسائط الوكيل القديم |
    | `plugin-sdk/conversation-runtime` | مساعدات المحادثة/ربط السلاسل/الاقتران/الارتباطات المكوّنة |
    | `plugin-sdk/runtime-config-snapshot` | مساعد لقطة تكوين وقت التشغيل |
    | `plugin-sdk/runtime-group-policy` | مساعدات حل سياسة المجموعة في وقت التشغيل |
    | `plugin-sdk/channel-status` | مساعدات مشتركة للّقطة/الملخص الخاص بحالة القناة |
    | `plugin-sdk/channel-config-primitives` | أوليات ضيقة لمخطط تكوين القناة |
    | `plugin-sdk/channel-config-writes` | مساعدات تفويض كتابة تكوين القناة |
    | `plugin-sdk/channel-plugin-common` | تصديرات تمهيدية مشتركة لـ plugin القناة |
    | `plugin-sdk/allowlist-config-edit` | مساعدات قراءة/تحرير تكوين allowlist |
    | `plugin-sdk/group-access` | مساعدات مشتركة لقرارات الوصول إلى المجموعات |
    | `plugin-sdk/direct-dm` | مساعدات مشتركة لمصادقة/حراسة الرسائل الخاصة المباشرة |
    | `plugin-sdk/interactive-runtime` | مساعدات تطبيع/تقليص حمولات الردود التفاعلية |
    | `plugin-sdk/channel-inbound` | إزالة الارتجاج، ومطابقة الإشارات، ومساعدات الغلاف |
    | `plugin-sdk/channel-send-result` | أنواع نتائج الرد |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema` و`createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | مساعدات تحليل/مطابقة الأهداف |
    | `plugin-sdk/channel-contract` | أنواع عقد القناة |
    | `plugin-sdk/channel-feedback` | توصيل الملاحظات/التفاعلات |
  </Accordion>

  <Accordion title="المسارات الفرعية للمزوّدين">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | مساعدات إعداد منسقة لمزوّدات محلية/مستضافة ذاتيًا |
    | `plugin-sdk/self-hosted-provider-setup` | مساعدات إعداد مركزة لمزوّدات متوافقة ذاتيًا مع OpenAI |
    | `plugin-sdk/cli-backend` | القيم الافتراضية لـ CLI backend + ثوابت watchdog |
    | `plugin-sdk/provider-auth-runtime` | مساعدات حل مفتاح API في وقت التشغيل لـ plugins المزوّد |
    | `plugin-sdk/provider-auth-api-key` | مساعدات onboarding/الكتابة لملف التعريف لمفتاح API |
    | `plugin-sdk/provider-auth-result` | باني قياسي لنتيجة مصادقة OAuth |
    | `plugin-sdk/provider-auth-login` | مساعدات تسجيل الدخول التفاعلية المشتركة لـ plugins المزوّد |
    | `plugin-sdk/provider-env-vars` | مساعدات البحث عن متغيرات البيئة الخاصة بمصادقة المزوّد |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod` و`ensureApiKeyFromOptionEnvOrPrompt` و`upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily` و`buildProviderReplayFamilyHooks` و`normalizeModelCompat` وبناة سياسات replay المشتركة، ومساعدات نقاط نهاية المزوّد، ومساعدات تطبيع معرّف النموذج مثل `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate` و`buildSingleProviderApiKeyCatalog` و`supportsNativeStreamingUsageCompat` و`applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | مساعدات عامة لإمكانات HTTP/نقاط نهاية المزوّد |
    | `plugin-sdk/provider-web-fetch` | مساعدات تسجيل/تخزين مؤقت لمزوّد web-fetch |
    | `plugin-sdk/provider-web-search` | مساعدات تسجيل/تخزين مؤقت/تكوين لمزوّد web-search |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily` و`buildProviderToolCompatFamilyHooks` وتنظيف مخطط Gemini + التشخيصات، ومساعدات توافق xAI مثل `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` وما شابه |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily` و`buildProviderStreamFamilyHooks` و`composeProviderStreamWrappers` وأنواع مغلفات stream، ومساعدات مغلفات Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot المشتركة |
    | `plugin-sdk/provider-onboard` | مساعدات ترقيع تكوين onboarding |
    | `plugin-sdk/global-singleton` | مساعدات singleton/map/cache محلية للعملية |
  </Accordion>

  <Accordion title="المسارات الفرعية للمصادقة والأمان">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`، ومساعدات سجل الأوامر، ومساعدات تفويض المرسل |
    | `plugin-sdk/approval-auth-runtime` | حل الموافق ومساعدات مصادقة الإجراءات داخل الدردشة نفسها |
    | `plugin-sdk/approval-client-runtime` | مساعدات profile/filter الخاصة بموافقات exec الأصلية |
    | `plugin-sdk/approval-delivery-runtime` | مهايئات القدرة/التسليم للموافقات الأصلية |
    | `plugin-sdk/approval-native-runtime` | مساعدات الهدف الأصلي للموافقة + ربط الحساب |
    | `plugin-sdk/approval-reply-runtime` | مساعدات حمولة الرد الخاصة بموافقات exec/plugin |
    | `plugin-sdk/command-auth-native` | مصادقة الأوامر الأصلية + مساعدات الأهداف الأصلية للجلسات |
    | `plugin-sdk/command-detection` | مساعدات مشتركة لاكتشاف الأوامر |
    | `plugin-sdk/command-surface` | تطبيع نص الأمر ومساعدات سطح الأوامر |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | مساعدات مشتركة للثقة، وتقييد DM، والمحتوى الخارجي، وجمع الأسرار |
    | `plugin-sdk/ssrf-policy` | مساعدات سياسة SSRF لقائمة سماح المضيف والشبكة الخاصة |
    | `plugin-sdk/ssrf-runtime` | مساعدات pinned-dispatcher وfetch المحمية بـ SSRF وسياسة SSRF |
    | `plugin-sdk/secret-input` | مساعدات تحليل مدخلات الأسرار |
    | `plugin-sdk/webhook-ingress` | مساعدات طلبات/أهداف webhook |
    | `plugin-sdk/webhook-request-guards` | مساعدات حجم جسم الطلب/المهلة |
  </Accordion>

  <Accordion title="المسارات الفرعية لوقت التشغيل والتخزين">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/runtime` | مساعدات واسعة لوقت التشغيل/التسجيل/النسخ الاحتياطي/تثبيت plugin |
    | `plugin-sdk/runtime-env` | مساعدات ضيقة لبيئة وقت التشغيل، وlogger، والمهلة، وإعادة المحاولة، والتراجع |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | مساعدات مشتركة لأوامر/خطافات/http/تفاعلات plugin |
    | `plugin-sdk/hook-runtime` | مساعدات مشتركة لخط أنابيب webhooks/الخطافات الداخلية |
    | `plugin-sdk/lazy-runtime` | مساعدات الاستيراد/الربط الكسول لوقت التشغيل مثل `createLazyRuntimeModule` و`createLazyRuntimeMethod` و`createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | مساعدات تنفيذ العمليات |
    | `plugin-sdk/cli-runtime` | مساعدات تنسيق CLI والانتظار والإصدار |
    | `plugin-sdk/gateway-runtime` | مساعدات عميل Gateway وترقيع حالة القناة |
    | `plugin-sdk/config-runtime` | مساعدات تحميل/كتابة التكوين |
    | `plugin-sdk/telegram-command-config` | تطبيع اسم/وصف أوامر Telegram والتحقق من التكرار/التعارض، حتى عندما لا يكون سطح عقد Telegram المضمّن متاحًا |
    | `plugin-sdk/approval-runtime` | مساعدات موافقة exec/plugin، وبناة قدرات الموافقة، ومساعدات auth/profile، ومساعدات التوجيه/وقت التشغيل الأصلية |
    | `plugin-sdk/reply-runtime` | مساعدات مشتركة لوقت التشغيل الخاص بالوارد/الرد، والتجزئة، والإرسال، وheartbeat، ومخطط الرد |
    | `plugin-sdk/reply-dispatch-runtime` | مساعدات ضيقة لإرسال/إنهاء الرد |
    | `plugin-sdk/reply-history` | مساعدات مشتركة لنافذة سجل الرد القصيرة مثل `buildHistoryContext` و`recordPendingHistoryEntry` و`clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | مساعدات ضيقة لتجزئة النص/Markdown |
    | `plugin-sdk/session-store-runtime` | مساعدات مسار مخزن الجلسة + updated-at |
    | `plugin-sdk/state-paths` | مساعدات مسارات دليل الحالة/OAuth |
    | `plugin-sdk/routing` | مساعدات المسار/مفتاح الجلسة/ربط الحساب مثل `resolveAgentRoute` و`buildAgentSessionKey` و`resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | مساعدات مشتركة لملخص حالة القناة/الحساب، وافتراضيات حالة وقت التشغيل، ومساعدات بيانات تعريف المشكلات |
    | `plugin-sdk/target-resolver-runtime` | مساعدات مشتركة لحل الأهداف |
    | `plugin-sdk/string-normalization-runtime` | مساعدات تطبيع slug/string |
    | `plugin-sdk/request-url` | استخراج عناوين URL النصية من مدخلات شبيهة بـ fetch/request |
    | `plugin-sdk/run-command` | مشغّل أوامر مؤقت بنتائج stdout/stderr مُطبَّعة |
    | `plugin-sdk/param-readers` | قارئات معلمات شائعة للأدوات/CLI |
    | `plugin-sdk/tool-send` | استخراج حقول هدف الإرسال المعتمدة من وسائط الأداة |
    | `plugin-sdk/temp-path` | مساعدات مشتركة لمسارات التنزيل المؤقتة |
    | `plugin-sdk/logging-core` | logger للنظام الفرعي ومساعدات التنقيح |
    | `plugin-sdk/markdown-table-runtime` | مساعدات أوضاع جداول Markdown |
    | `plugin-sdk/json-store` | مساعدات صغيرة لقراءة/كتابة حالة JSON |
    | `plugin-sdk/file-lock` | مساعدات file-lock معاد الدخول |
    | `plugin-sdk/persistent-dedupe` | مساعدات ذاكرة تخزين مؤقت dedupe مدعومة بالقرص |
    | `plugin-sdk/acp-runtime` | مساعدات وقت تشغيل/جلسات ACP |
    | `plugin-sdk/agent-config-primitives` | أوليات ضيقة لمخطط تكوين وقت تشغيل الوكيل |
    | `plugin-sdk/boolean-param` | قارئ معلمات منطقي مرن |
    | `plugin-sdk/dangerous-name-runtime` | مساعدات حل مطابقة الأسماء الخطرة |
    | `plugin-sdk/device-bootstrap` | مساعدات bootstrap الجهاز ورمز الاقتران |
    | `plugin-sdk/extension-shared` | أوليات مشتركة للقناة السلبية ومساعدات الحالة |
    | `plugin-sdk/models-provider-runtime` | مساعدات الرد لأوامر `/models`/المزوّد |
    | `plugin-sdk/skill-commands-runtime` | مساعدات سرد أوامر Skills |
    | `plugin-sdk/native-command-registry` | مساعدات سجل/بناء/تسلسل الأوامر الأصلية |
    | `plugin-sdk/provider-zai-endpoint` | مساعدات اكتشاف نقطة نهاية Z.AI |
    | `plugin-sdk/infra-runtime` | مساعدات system event/heartbeat |
    | `plugin-sdk/collection-runtime` | مساعدات صغيرة لذاكرة تخزين مؤقت محدودة |
    | `plugin-sdk/diagnostic-runtime` | مساعدات أعلام وأحداث التشخيص |
    | `plugin-sdk/error-runtime` | رسم الأخطاء، والتنسيق، ومساعدات تصنيف الأخطاء المشتركة، و`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | مساعدات fetch المغلفة، وproxy، والبحث المثبت |
    | `plugin-sdk/host-runtime` | مساعدات تطبيع اسم المضيف وSCP host |
    | `plugin-sdk/retry-runtime` | مساعدات تكوين وتشغيل إعادة المحاولة |
    | `plugin-sdk/agent-runtime` | مساعدات agent dir/الهوية/مساحة العمل |
    | `plugin-sdk/directory-runtime` | استعلام/إزالة التكرار للدليل المعتمد على التكوين |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="المسارات الفرعية للقدرات والاختبار">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/media-runtime` | مساعدات مشتركة لجلب/تحويل/تخزين الوسائط بالإضافة إلى بُناة حمولات الوسائط |
    | `plugin-sdk/media-understanding` | أنواع مزوّد فهم الوسائط بالإضافة إلى تصديرات مساعدات الصور/الصوت الموجهة للمزوّد |
    | `plugin-sdk/text-runtime` | مساعدات مشتركة للنص/Markdown/التسجيل مثل إزالة النص المرئي للمساعد، ومساعدات render/chunking/table في Markdown، ومساعدات التنقيح، ومساعدات directive-tag، وأدوات النص الآمن |
    | `plugin-sdk/text-chunking` | مساعد تجزئة النص الصادر |
    | `plugin-sdk/speech` | أنواع مزوّد speech بالإضافة إلى تصديرات المساعدات الموجهة للمزوّد الخاصة بالـ directive والسجل والتحقق |
    | `plugin-sdk/speech-core` | أنواع مزوّد speech المشتركة، والسجل، والـ directive، ومساعدات التطبيع |
    | `plugin-sdk/realtime-transcription` | أنواع مزوّد transcription الفورية ومساعدات السجل |
    | `plugin-sdk/realtime-voice` | أنواع مزوّد الصوت الفوري ومساعدات السجل |
    | `plugin-sdk/image-generation` | أنواع مزوّد توليد الصور |
    | `plugin-sdk/image-generation-core` | أنواع توليد الصور المشتركة، وfailover، والمصادقة، ومساعدات السجل |
    | `plugin-sdk/video-generation` | أنواع مزوّد/طلب/نتيجة توليد الفيديو |
    | `plugin-sdk/video-generation-core` | أنواع توليد الفيديو المشتركة، ومساعدات failover، والبحث عن المزوّد، وتحليل model-ref |
    | `plugin-sdk/webhook-targets` | سجل أهداف webhook ومساعدات تثبيت المسارات |
    | `plugin-sdk/webhook-path` | مساعدات تطبيع مسار webhook |
    | `plugin-sdk/web-media` | مساعدات مشتركة لتحميل الوسائط البعيدة/المحلية |
    | `plugin-sdk/zod` | إعادة تصدير `zod` لمستهلكي Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases` و`shouldAckReaction` |
  </Accordion>

  <Accordion title="المسارات الفرعية للذاكرة">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/memory-core` | سطح مساعد memory-core المضمّن من أجل مساعدات المدير/التكوين/الملف/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | واجهة وقت التشغيل لفهرسة/بحث الذاكرة |
    | `plugin-sdk/memory-core-host-engine-foundation` | تصديرات محرك الأساس لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-embeddings` | تصديرات محرك embeddings لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-qmd` | تصديرات محرك QMD لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-storage` | تصديرات محرك التخزين لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-multimodal` | مساعدات متعددة الوسائط لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-query` | مساعدات الاستعلام لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-secret` | مساعدات الأسرار لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-status` | مساعدات الحالة لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-cli` | مساعدات وقت تشغيل CLI لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-core` | مساعدات وقت التشغيل الأساسية لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-files` | مساعدات الملفات/وقت التشغيل لمضيف الذاكرة |
    | `plugin-sdk/memory-lancedb` | سطح مساعد memory-lancedb المضمّن |
  </Accordion>

  <Accordion title="المسارات الفرعية المحجوزة للمساعدات المضمّنة">
    | العائلة | المسارات الفرعية الحالية | الاستخدام المقصود |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-config-support` و`plugin-sdk/browser-support` | مساعدات دعم plugin ‏browser المضمّنة |
    | Matrix | `plugin-sdk/matrix` و`plugin-sdk/matrix-helper` و`plugin-sdk/matrix-runtime-heavy` و`plugin-sdk/matrix-runtime-shared` و`plugin-sdk/matrix-runtime-surface` و`plugin-sdk/matrix-surface` و`plugin-sdk/matrix-thread-bindings` | سطح المساعدة/وقت التشغيل المضمّن لـ Matrix |
    | Line | `plugin-sdk/line` و`plugin-sdk/line-core` و`plugin-sdk/line-runtime` و`plugin-sdk/line-surface` | سطح المساعدة/وقت التشغيل المضمّن لـ LINE |
    | IRC | `plugin-sdk/irc` و`plugin-sdk/irc-surface` | سطح المساعدة المضمّن لـ IRC |
    | مساعدات خاصة بالقنوات | `plugin-sdk/googlechat` و`plugin-sdk/zalouser` و`plugin-sdk/bluebubbles` و`plugin-sdk/bluebubbles-policy` و`plugin-sdk/mattermost` و`plugin-sdk/mattermost-policy` و`plugin-sdk/feishu-conversation` و`plugin-sdk/msteams` و`plugin-sdk/nextcloud-talk` و`plugin-sdk/nostr` و`plugin-sdk/tlon` و`plugin-sdk/twitch` | أسطح توافق/مساعدة مضمّنة للقنوات |
    | مساعدات خاصة بالمصادقة/plugin | `plugin-sdk/github-copilot-login` و`plugin-sdk/github-copilot-token` و`plugin-sdk/diagnostics-otel` و`plugin-sdk/diffs` و`plugin-sdk/llm-task` و`plugin-sdk/thread-ownership` و`plugin-sdk/voice-call` | أسطح مساعدة مضمّنة للميزات/plugins؛ ويصدّر `plugin-sdk/github-copilot-token` حاليًا `DEFAULT_COPILOT_API_BASE_URL` و`deriveCopilotApiBaseUrlFromToken` و`resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API التسجيل

يتلقى رد النداء `register(api)` كائن `OpenClawPluginApi` بهذه
الطرق:

### تسجيل القدرات

| الطريقة                                           | ما الذي تسجله                |
| ------------------------------------------------ | ---------------------------- |
| `api.registerProvider(...)`                      | الاستدلال النصي (LLM)        |
| `api.registerCliBackend(...)`                    | واجهة CLI خلفية للاستدلال المحلي |
| `api.registerChannel(...)`                       | قناة مراسلة                  |
| `api.registerSpeechProvider(...)`                | تحويل النص إلى كلام / تركيب STT |
| `api.registerRealtimeTranscriptionProvider(...)` | transcription فورية متدفقة   |
| `api.registerRealtimeVoiceProvider(...)`         | جلسات صوت فورية ثنائية الاتجاه |
| `api.registerMediaUnderstandingProvider(...)`    | تحليل الصور/الصوت/الفيديو    |
| `api.registerImageGenerationProvider(...)`       | توليد الصور                  |
| `api.registerVideoGenerationProvider(...)`       | توليد الفيديو                |
| `api.registerWebFetchProvider(...)`              | مزوّد web fetch / scrape     |
| `api.registerWebSearchProvider(...)`             | web search                   |

### الأدوات والأوامر

| الطريقة                         | ما الذي تسجله                                  |
| ------------------------------- | ---------------------------------------------- |
| `api.registerTool(tool, opts?)` | أداة وكيل (مطلوبة أو `{ optional: true }`)     |
| `api.registerCommand(def)`      | أمر مخصص (يتجاوز LLM)                          |

### البنية التحتية

| الطريقة                                        | ما الذي تسجله      |
| --------------------------------------------- | ------------------ |
| `api.registerHook(events, handler, opts?)`    | خطاف حدث           |
| `api.registerHttpRoute(params)`               | نقطة نهاية HTTP لـ Gateway |
| `api.registerGatewayMethod(name, handler)`    | طريقة Gateway RPC  |
| `api.registerCli(registrar, opts?)`           | أمر فرعي CLI       |
| `api.registerService(service)`                | خدمة خلفية         |
| `api.registerInteractiveHandler(registration)`| معالج تفاعلي       |

تظل مساحات أسماء الإدارة الأساسية المحجوزة (`config.*` و`exec.approvals.*` و`wizard.*` و`update.*`)
دائمًا ضمن `operator.admin`، حتى لو حاولت plugin تعيين
نطاق أضيق لطريقة gateway. فضّل البوادئ الخاصة بـ plugin
للطرق المملوكة لـ plugin.

### بيانات تعريف تسجيل CLI

يقبل `api.registerCli(registrar, opts?)` نوعين من بيانات التعريف العليا:

- `commands`: جذور أوامر صريحة يملكها الـ registrar
- `descriptors`: واصفات أوامر وقت التحليل المستخدمة لمساعدة CLI الجذرية،
  والتوجيه، والتسجيل الكسول لـ CLI الخاصة بـ plugin

إذا كنت تريد أن يبقى أمر plugin محمّلًا بكسل في مسار CLI الجذري العادي،
فوفّر `descriptors` تغطي كل جذر أمر من المستوى الأعلى يعرضه ذلك الـ registrar.

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

استخدم `commands` بمفردها فقط عندما لا تحتاج إلى تسجيل CLI جذري كسول.
يظل مسار التوافق eager هذا مدعومًا، لكنه لا يثبت
عناصر نائبة مدعومة بـ descriptor من أجل التحميل الكسول وقت التحليل.

### تسجيل CLI backend

تسمح `api.registerCliBackend(...)` لأي plugin بامتلاك التكوين الافتراضي لـ
CLI backend محلية للذكاء الاصطناعي مثل `claude-cli` أو `codex-cli`.

- يصبح `id` الخاص بالواجهة الخلفية بادئة المزوّد في model refs مثل `claude-cli/opus`.
- يستخدم `config` للواجهة الخلفية الشكل نفسه الموجود في `agents.defaults.cliBackends.<id>`.
- يظل تكوين المستخدم هو الفائز. يدمج OpenClaw قيمة `agents.defaults.cliBackends.<id>` فوق
  القيمة الافتراضية لـ plugin قبل تشغيل CLI.
- استخدم `normalizeConfig` عندما تحتاج الواجهة الخلفية إلى إعادة كتابة توافقية بعد الدمج
  (مثل تطبيع أشكال العلامات القديمة).

### الـ slots الحصرية

| الطريقة                                    | ما الذي تسجله                          |
| ----------------------------------------- | -------------------------------------- |
| `api.registerContextEngine(id, factory)`  | محرك سياق (واحد نشط في كل مرة)         |
| `api.registerMemoryPromptSection(builder)`| باني قسم مطالبة الذاكرة               |
| `api.registerMemoryFlushPlan(resolver)`   | محلل خطة تفريغ الذاكرة                |
| `api.registerMemoryRuntime(runtime)`      | مهايئ وقت تشغيل الذاكرة               |

### مهايئات embeddings الخاصة بالذاكرة

| الطريقة                                        | ما الذي تسجله                                  |
| --------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | مهايئ embedding للذاكرة لـ plugin النشطة        |

- تُعد `registerMemoryPromptSection` و`registerMemoryFlushPlan` و
  `registerMemoryRuntime` حصرية لـ plugins الذاكرة.
- تسمح `registerMemoryEmbeddingProvider` لـ plugin الذاكرة النشطة بتسجيل
  معرّف embedding واحد أو أكثر (مثل `openai` أو `gemini` أو معرّف مخصص تعرفه plugin).
- يُحل تكوين المستخدم مثل `agents.defaults.memorySearch.provider` و
  `agents.defaults.memorySearch.fallback` مقابل
  معرّفات المهايئات المسجلة تلك.

### الأحداث ودورة الحياة

| الطريقة                                      | ما الذي تفعله             |
| ------------------------------------------- | ------------------------- |
| `api.on(hookName, handler, opts?)`          | خطاف دورة حياة مطبوع      |
| `api.onConversationBindingResolved(handler)`| callback لحل ارتباط المحادثة |

### دلالات قرارات الخطافات

- `before_tool_call`: إعادة `{ block: true }` تكون نهائية. فبمجرد أن يضبط أي معالج هذه القيمة، يتم تجاوز المعالجات ذات الأولوية الأدنى.
- `before_tool_call`: تُعامل إعادة `{ block: false }` على أنها عدم اتخاذ قرار (مثل حذف `block`)، وليست تجاوزًا.
- `before_install`: إعادة `{ block: true }` تكون نهائية. فبمجرد أن يضبط أي معالج هذه القيمة، يتم تجاوز المعالجات ذات الأولوية الأدنى.
- `before_install`: تُعامل إعادة `{ block: false }` على أنها عدم اتخاذ قرار (مثل حذف `block`)، وليست تجاوزًا.
- `message_sending`: إعادة `{ cancel: true }` تكون نهائية. فبمجرد أن يضبط أي معالج هذه القيمة، يتم تجاوز المعالجات ذات الأولوية الأدنى.
- `message_sending`: تُعامل إعادة `{ cancel: false }` على أنها عدم اتخاذ قرار (مثل حذف `cancel`)، وليست تجاوزًا.

### حقول كائن API

| الحقل                   | النوع                     | الوصف                                                                                     |
| ----------------------- | ------------------------- | ----------------------------------------------------------------------------------------- |
| `api.id`                | `string`                  | معرّف plugin                                                                              |
| `api.name`              | `string`                  | اسم العرض                                                                                 |
| `api.version`           | `string?`                 | إصدار plugin ‏(اختياري)                                                                   |
| `api.description`       | `string?`                 | وصف plugin ‏(اختياري)                                                                     |
| `api.source`            | `string`                  | مسار مصدر plugin                                                                          |
| `api.rootDir`           | `string?`                 | الدليل الجذري لـ plugin ‏(اختياري)                                                        |
| `api.config`            | `OpenClawConfig`          | لقطة التكوين الحالية (لقطة وقت التشغيل النشطة داخل الذاكرة عندما تكون متاحة)             |
| `api.pluginConfig`      | `Record<string, unknown>` | تكوين plugin الخاص من `plugins.entries.<id>.config`                                       |
| `api.runtime`           | `PluginRuntime`           | [مساعدات وقت التشغيل](/plugins/sdk-runtime)                                               |
| `api.logger`            | `PluginLogger`            | logger ذو نطاق (`debug` و`info` و`warn` و`error`)                                         |
| `api.registrationMode`  | `PluginRegistrationMode`  | وضع التحميل الحالي؛ وتُعد `"setup-runtime"` نافذة بدء تشغيل/إعداد خفيفة قبل الإدخال الكامل |
| `api.resolvePath(input)`| `(string) => string`      | حل المسار نسبةً إلى جذر plugin                                                           |

## أسلوب الوحدات الداخلية

داخل plugin الخاصة بك، استخدم ملفات barrel محلية للاستيرادات الداخلية:

```
my-plugin/
  api.ts            # تصديرات عامة للمستهلكين الخارجيين
  runtime-api.ts    # تصديرات وقت تشغيل داخلية فقط
  index.ts          # نقطة دخول plugin
  setup-entry.ts    # إدخال خفيف خاص بالإعداد فقط (اختياري)
```

<Warning>
  لا تستورد plugin الخاصة بك عبر `openclaw/plugin-sdk/<your-plugin>`
  من شيفرة الإنتاج مطلقًا. وجّه الاستيرادات الداخلية عبر `./api.ts` أو
  `./runtime-api.ts`. إن مسار SDK هو العقد الخارجي فقط.
</Warning>

تفضّل الأسطح العامة لـ plugins المضمّنة المحمّلة عبر الواجهة (`api.ts` و`runtime-api.ts` و
`index.ts` و`setup-entry.ts` وملفات الإدخال العامة المشابهة) الآن استخدام
لقطة تكوين وقت التشغيل النشطة عندما يكون OpenClaw يعمل بالفعل. وإذا لم توجد لقطة وقت تشغيل بعد،
فإنها تعود إلى ملف التكوين المحلول على القرص.

يمكن لـ Plugins المزوّد أيضًا عرض barrel ضيق للعقد المحلي داخل plugin عندما تكون
مساعدة ما خاصة بالمزوّد عمدًا ولا تنتمي بعد إلى مسار فرعي عام في SDK. المثال المضمّن الحالي:
يحتفظ مزود Anthropic بمساعدات Claude stream الخاصة به في
المسار العام المحلي `api.ts` / `contract-api.ts` بدلًا من
ترقية منطق رؤوس Anthropic التجريبية و`service_tier` إلى عقد عام من نوع
`plugin-sdk/*`.

أمثلة مضمّنة أخرى حالية:

- `@openclaw/openai-provider`: يصدر `api.ts` بُناة المزوّد،
  ومساعدات النماذج الافتراضية، وبُناة مزوّدات realtime
- `@openclaw/openrouter-provider`: يصدر `api.ts` باني المزوّد بالإضافة إلى
  مساعدات onboarding/config

<Warning>
  يجب على شيفرة الإنتاج الخاصة بالامتدادات أيضًا تجنب استيرادات
  `openclaw/plugin-sdk/<other-plugin>`. وإذا كانت مساعدة ما مشتركة فعلًا، فقم بترقيتها إلى مسار SDK محايد
  مثل `openclaw/plugin-sdk/speech` أو `.../provider-model-shared` أو سطح آخر
  موجه نحو القدرات بدلًا من ربط pluginين معًا.
</Warning>

## ذو صلة

- [نقاط الدخول](/plugins/sdk-entrypoints) — خيارات `definePluginEntry` و`defineChannelPluginEntry`
- [مساعدات وقت التشغيل](/plugins/sdk-runtime) — المرجع الكامل لمساحة الأسماء `api.runtime`
- [الإعداد والتكوين](/plugins/sdk-setup) — الحزم، والبيانات الوصفية، ومخططات التكوين
- [الاختبار](/plugins/sdk-testing) — أدوات الاختبار وقواعد lint
- [ترحيل SDK](/plugins/sdk-migration) — الترحيل من الأسطح المهجورة
- [البنية الداخلية لـ Plugin](/plugins/architecture) — البنية العميقة ونموذج القدرات
