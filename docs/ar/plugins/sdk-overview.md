---
read_when:
    - تحتاج إلى معرفة أي مسار فرعي من SDK يجب الاستيراد منه
    - تريد مرجعًا لجميع أساليب التسجيل على OpenClawPluginApi
    - أنت تبحث عن تصدير محدد من SDK
sidebarTitle: SDK Overview
summary: خريطة الاستيراد، ومرجع API التسجيل، وبنية SDK
title: نظرة عامة على Plugin SDK
x-i18n:
    generated_at: "2026-04-06T03:11:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: d801641f26f39dc21490d2a69a337ff1affb147141360916b8b58a267e9f822a
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# نظرة عامة على Plugin SDK

Plugin SDK هو العقد typed بين الإضافات والنواة. هذه الصفحة هي
المرجع الخاص بـ **ما يجب استيراده** و**ما الذي يمكنك تسجيله**.

<Tip>
  **هل تبحث عن دليل عملي؟**
  - أول إضافة؟ ابدأ بـ [Getting Started](/ar/plugins/building-plugins)
  - إضافة قناة؟ راجع [Channel Plugins](/ar/plugins/sdk-channel-plugins)
  - إضافة موفر؟ راجع [Provider Plugins](/ar/plugins/sdk-provider-plugins)
</Tip>

## اصطلاح الاستيراد

استورد دائمًا من مسار فرعي محدد:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

كل مسار فرعي هو وحدة صغيرة ومستقلة بذاتها. يحافظ هذا على سرعة بدء التشغيل
ويمنع مشكلات التبعيات الدائرية. بالنسبة إلى مساعدات إدخال/بناء القنوات الخاصة،
فضّل `openclaw/plugin-sdk/channel-core`؛ واحتفظ بـ `openclaw/plugin-sdk/core` من أجل
السطح الأوسع والمساعدات المشتركة مثل
`buildChannelConfigSchema`.

لا تضف ولا تعتمد على موصلات تسهيلية مسماة باسم الموفر مثل
`openclaw/plugin-sdk/slack` أو `openclaw/plugin-sdk/discord` أو
`openclaw/plugin-sdk/signal` أو `openclaw/plugin-sdk/whatsapp` أو
موصلات المساعدة الموسومة بالقنوات. يجب أن تركب الإضافات المجمعة مسارات
SDK الفرعية العامة داخل ملفات `api.ts` أو `runtime-api.ts` الخاصة بها، ويجب على النواة
إما استخدام هذه الملفات المحلية الخاصة بالإضافة أو إضافة عقد SDK عام ودقيق
عندما تكون الحاجة مشتركة فعلًا بين القنوات.

ما تزال خريطة التصدير المولدة تحتوي على مجموعة صغيرة من موصلات المساعدة
للإضافات المجمعة مثل `plugin-sdk/feishu` و`plugin-sdk/feishu-setup` و
`plugin-sdk/zalo` و`plugin-sdk/zalo-setup` و`plugin-sdk/matrix*`. هذه
المسارات الفرعية موجودة فقط لصيانة الإضافات المجمعة والتوافق، وهي
مستبعدة عمدًا من الجدول الشائع أدناه وليست مسار الاستيراد الموصى به
للإضافات الخارجية الجديدة.

## مرجع المسارات الفرعية

أكثر المسارات الفرعية استخدامًا، مجمعة حسب الغرض. توجد القائمة الكاملة المولدة التي تضم
أكثر من 200 مسار فرعي في `scripts/lib/plugin-sdk-entrypoints.json`.

ما تزال المسارات الفرعية المحجوزة لمساعدات الإضافات المجمعة تظهر في تلك القائمة المولدة.
تعامل معها كأسطح تفاصيل تنفيذ/توافق ما لم تروّج صفحة توثيق
أحدها صراحة باعتباره عامًا.

### إدخال الإضافة

| المسار الفرعي              | أهم التصديرات                                                                                                                        |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`  | `definePluginEntry`                                                                                                                   |
| `plugin-sdk/core`          | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema` | `OpenClawSchema`                                                                                                                      |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                     |

<AccordionGroup>
  <Accordion title="المسارات الفرعية للقنوات">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | تصدير مخطط Zod الجذر `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`، بالإضافة إلى `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | مساعدات معالج الإعداد المشتركة، ومطالبات allowlist، وبناة حالة الإعداد |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | مساعدات التكوين متعدد الحسابات/بوابة الإجراءات، ومساعدات الرجوع إلى الحساب الافتراضي |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`، ومساعدات تطبيع معرّف الحساب |
    | `plugin-sdk/account-resolution` | مساعدات البحث عن الحساب + الرجوع إلى الافتراضي |
    | `plugin-sdk/account-helpers` | مساعدات دقيقة لقائمة الحسابات/إجراءات الحساب |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | أنواع مخطط تكوين القناة |
    | `plugin-sdk/telegram-command-config` | مساعدات تطبيع/تحقق الأوامر المخصصة في Telegram مع الرجوع إلى العقدة المجمعة |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | مساعدات المسار الوارد + بناء المغلف المشتركة |
    | `plugin-sdk/inbound-reply-dispatch` | مساعدات التسجيل والإرسال للرسائل الواردة المشتركة |
    | `plugin-sdk/messaging-targets` | مساعدات تحليل/مطابقة الأهداف |
    | `plugin-sdk/outbound-media` | مساعدات تحميل الوسائط الصادرة المشتركة |
    | `plugin-sdk/outbound-runtime` | مساعدات هوية الإرسال/التفويض الصادر |
    | `plugin-sdk/thread-bindings-runtime` | مساعدات دورة حياة ربط الخيوط والمهايئات |
    | `plugin-sdk/agent-media-payload` | باني حمولة وسائط الوكيل القديم |
    | `plugin-sdk/conversation-runtime` | مساعدات ربط المحادثة/الخيط والاقتران والربط المكوّن |
    | `plugin-sdk/runtime-config-snapshot` | مساعد أخذ لقطة تكوين وقت التشغيل |
    | `plugin-sdk/runtime-group-policy` | مساعدات حل سياسة المجموعات وقت التشغيل |
    | `plugin-sdk/channel-status` | مساعدات لقطة/ملخص حالة القناة المشتركة |
    | `plugin-sdk/channel-config-primitives` | بدائيات دقيقة لمخطط تكوين القناة |
    | `plugin-sdk/channel-config-writes` | مساعدات تفويض كتابة تكوين القناة |
    | `plugin-sdk/channel-plugin-common` | تصديرات تمهيدية مشتركة لإضافات القنوات |
    | `plugin-sdk/allowlist-config-edit` | مساعدات قراءة/تحرير تكوين allowlist |
    | `plugin-sdk/group-access` | مساعدات قرار الوصول إلى المجموعات المشتركة |
    | `plugin-sdk/direct-dm` | مساعدات الحماية/المصادقة للرسائل الخاصة المباشرة المشتركة |
    | `plugin-sdk/interactive-runtime` | مساعدات تطبيع/تقليص حمولة الرد التفاعلي |
    | `plugin-sdk/channel-inbound` | مساعدات إزالة الارتداد debounce ومطابقة الذكر والمغلفات |
    | `plugin-sdk/channel-send-result` | أنواع نتيجة الرد |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | مساعدات تحليل/مطابقة الأهداف |
    | `plugin-sdk/channel-contract` | أنواع عقد القناة |
    | `plugin-sdk/channel-feedback` | توصيلات الملاحظات/التفاعلات |
  </Accordion>

  <Accordion title="المسارات الفرعية للموفرين">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | مساعدات منسقة لإعداد الموفرات المحلية/المستضافة ذاتيًا |
    | `plugin-sdk/self-hosted-provider-setup` | مساعدات دقيقة لإعداد موفرات مستضافة ذاتيًا ومتوافقة مع OpenAI |
    | `plugin-sdk/provider-auth-runtime` | مساعدات حل API key وقت التشغيل لإضافات الموفرين |
    | `plugin-sdk/provider-auth-api-key` | مساعدات API key الخاصة بالإعداد الأولي/كتابة الملف الشخصي |
    | `plugin-sdk/provider-auth-result` | باني نتائج مصادقة OAuth القياسي |
    | `plugin-sdk/provider-auth-login` | مساعدات تسجيل الدخول التفاعلي المشتركة لإضافات الموفرين |
    | `plugin-sdk/provider-env-vars` | مساعدات البحث عن متغيرات بيئة مصادقة الموفر |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`، وبناة سياسة إعادة التشغيل المشتركة، ومساعدات نقطة نهاية الموفّر، ومساعدات تطبيع معرّف النموذج مثل `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | مساعدات قدرات HTTP/نقطة النهاية العامة للموفر |
    | `plugin-sdk/provider-web-fetch` | مساعدات تسجيل/تخزين مؤقت لموفّر جلب الويب |
    | `plugin-sdk/provider-web-search` | مساعدات تسجيل/تخزين مؤقت/تكوين لموفّر البحث في الويب |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`، وتنظيف مخطط Gemini + التشخيصات، ومساعدات توافق xAI مثل `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` وما شابهه |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`، وأنواع مغلفات التدفق، ومساعدات المغلفات المشتركة لـ Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | مساعدات تصحيح تكوين الإعداد الأولي |
    | `plugin-sdk/global-singleton` | مساعدات singleton/map/cache محلية على مستوى العملية |
  </Accordion>

  <Accordion title="المسارات الفرعية للمصادقة والأمان">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`، ومساعدات سجل الأوامر، ومساعدات تفويض المرسل |
    | `plugin-sdk/approval-auth-runtime` | مساعدات حل الموافقين ومصادقة الإجراءات داخل الدردشة نفسها |
    | `plugin-sdk/approval-client-runtime` | مساعدات الملف الشخصي/عامل التصفية لموافقات exec الأصلية |
    | `plugin-sdk/approval-delivery-runtime` | مهايئات قدرات/تسليم الموافقات الأصلية |
    | `plugin-sdk/approval-native-runtime` | مساعدات الهدف الأصلي للموافقة + ربط الحساب |
    | `plugin-sdk/approval-reply-runtime` | مساعدات حمولة الرد لموافقات exec/الإضافات |
    | `plugin-sdk/command-auth-native` | المصادقة الأصلية للأوامر + مساعدات الهدف الأصلي للجلسة |
    | `plugin-sdk/command-detection` | مساعدات اكتشاف الأوامر المشتركة |
    | `plugin-sdk/command-surface` | مساعدات تطبيع جسم الأمر وسطح الأمر |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | مساعدات الثقة المشتركة، وتقييد الرسائل الخاصة، والمحتوى الخارجي، وجمع الأسرار |
    | `plugin-sdk/ssrf-policy` | مساعدات سياسة SSRF لقائمة السماح بالمضيفين والشبكات الخاصة |
    | `plugin-sdk/ssrf-runtime` | مساعدات pinned-dispatcher وfetch المحمي من SSRF وسياسة SSRF |
    | `plugin-sdk/secret-input` | مساعدات تحليل مدخلات الأسرار |
    | `plugin-sdk/webhook-ingress` | مساعدات الطلب/الهدف لـ webhook |
    | `plugin-sdk/webhook-request-guards` | مساعدات حجم جسم الطلب/المهلة الزمنية |
  </Accordion>

  <Accordion title="المسارات الفرعية لوقت التشغيل والتخزين">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/runtime` | مساعدات عامة لوقت التشغيل/التسجيل/النسخ الاحتياطي/تثبيت الإضافات |
    | `plugin-sdk/runtime-env` | مساعدات دقيقة لبيئة وقت التشغيل، والمسجل، والمهلة، وإعادة المحاولة، والتراجع |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | مساعدات مشتركة لأوامر الإضافة/الـ hook/الـ http/التفاعل |
    | `plugin-sdk/hook-runtime` | مساعدات مشتركة لمسار webhook/الـ hook الداخلي |
    | `plugin-sdk/lazy-runtime` | مساعدات الاستيراد/الربط الكسول لوقت التشغيل مثل `createLazyRuntimeModule` و`createLazyRuntimeMethod` و`createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | مساعدات تنفيذ العمليات |
    | `plugin-sdk/cli-runtime` | مساعدات تنسيق CLI والانتظار والإصدار |
    | `plugin-sdk/gateway-runtime` | مساعدات عميل البوابة وتصحيح حالة القناة |
    | `plugin-sdk/config-runtime` | مساعدات تحميل/كتابة التكوين |
    | `plugin-sdk/telegram-command-config` | تطبيع اسم/وصف أوامر Telegram والتحقق من التكرار/التعارض، حتى عندما لا يتوفر سطح عقد Telegram المجمّع |
    | `plugin-sdk/approval-runtime` | مساعدات موافقات exec/الإضافات، وبناة قدرات الموافقة، ومساعدات المصادقة/الملف الشخصي، ومساعدات التوجيه/وقت التشغيل الأصلية |
    | `plugin-sdk/reply-runtime` | مساعدات مشتركة لوقت تشغيل الوارد/الرد، والتقسيم، والإرسال، والنبض، ومخطط الرد |
    | `plugin-sdk/reply-dispatch-runtime` | مساعدات دقيقة لإرسال/إنهاء الرد |
    | `plugin-sdk/reply-history` | مساعدات مشتركة لسجل الرد ضمن نافذة قصيرة مثل `buildHistoryContext` و`recordPendingHistoryEntry` و`clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | مساعدات دقيقة لتقسيم النص/Markdown |
    | `plugin-sdk/session-store-runtime` | مساعدات مسار مخزن الجلسة + updated-at |
    | `plugin-sdk/state-paths` | مساعدات مسارات دليل الحالة/OAuth |
    | `plugin-sdk/routing` | مساعدات ربط المسار/مفتاح الجلسة/الحساب مثل `resolveAgentRoute` و`buildAgentSessionKey` و`resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | مساعدات مشتركة لملخص حالة القناة/الحساب، والقيم الافتراضية لحالة وقت التشغيل، ومساعدات بيانات المشكلات |
    | `plugin-sdk/target-resolver-runtime` | مساعدات مشتركة لحل الأهداف |
    | `plugin-sdk/string-normalization-runtime` | مساعدات تطبيع slug/السلاسل |
    | `plugin-sdk/request-url` | استخراج عناوين URL النصية من مدخلات تشبه fetch/request |
    | `plugin-sdk/run-command` | مشغل أوامر موقّت بنتائج stdout/stderr مطبعة |
    | `plugin-sdk/param-readers` | قارئات معاملات شائعة للأدوات/CLI |
    | `plugin-sdk/tool-send` | استخراج حقول هدف الإرسال المعيارية من معاملات الأداة |
    | `plugin-sdk/temp-path` | مساعدات مشتركة لمسارات التنزيل المؤقت |
    | `plugin-sdk/logging-core` | مساعدات مسجل النظام الفرعي وإخفاء البيانات الحساسة |
    | `plugin-sdk/markdown-table-runtime` | مساعدات أوضاع جداول Markdown |
    | `plugin-sdk/json-store` | مساعدات صغيرة لقراءة/كتابة حالة JSON |
    | `plugin-sdk/file-lock` | مساعدات قفل الملفات القابلة لإعادة الدخول |
    | `plugin-sdk/persistent-dedupe` | مساعدات ذاكرة تخزين مؤقت لإزالة التكرار مدعومة بالقرص |
    | `plugin-sdk/acp-runtime` | مساعدات وقت التشغيل/الجلسة وإرسال الرد لـ ACP |
    | `plugin-sdk/agent-config-primitives` | بدائيات دقيقة لمخطط تكوين وقت تشغيل الوكيل |
    | `plugin-sdk/boolean-param` | قارئ مرن لمعاملات boolean |
    | `plugin-sdk/dangerous-name-runtime` | مساعدات حل مطابقة الأسماء الخطرة |
    | `plugin-sdk/device-bootstrap` | مساعدات bootstrap الخاصة بالجهاز وtoken الاقتران |
    | `plugin-sdk/extension-shared` | بدائيات مشتركة للقنوات السلبية ومساعدات الحالة |
    | `plugin-sdk/models-provider-runtime` | مساعدات أوامر `/models` وردود الموفر |
    | `plugin-sdk/skill-commands-runtime` | مساعدات عرض أوامر Skills |
    | `plugin-sdk/native-command-registry` | مساعدات تسجيل/بناء/تسلسل الأوامر الأصلية |
    | `plugin-sdk/provider-zai-endpoint` | مساعدات اكتشاف نقطة نهاية Z.AI |
    | `plugin-sdk/infra-runtime` | مساعدات أحداث النظام/النبض |
    | `plugin-sdk/collection-runtime` | مساعدات صغيرة لذاكرة تخزين مؤقت محدودة |
    | `plugin-sdk/diagnostic-runtime` | مساعدات أعلام وأحداث التشخيص |
    | `plugin-sdk/error-runtime` | رسم بياني للأخطاء، وتنسيق، ومساعدات التصنيف المشترك للأخطاء، و`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | مساعدات fetch المغلف والـ proxy والبحث المثبت |
    | `plugin-sdk/host-runtime` | مساعدات تطبيع اسم المضيف وSCP host |
    | `plugin-sdk/retry-runtime` | مساعدات تكوين إعادة المحاولة ومشغل إعادة المحاولة |
    | `plugin-sdk/agent-runtime` | مساعدات دليل/هوية/مساحة عمل الوكيل |
    | `plugin-sdk/directory-runtime` | استعلام/إزالة تكرار الدليل المدعوم بالتكوين |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="المسارات الفرعية للقدرات والاختبار">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/media-runtime` | مساعدات مشتركة لجلب/تحويل/تخزين الوسائط بالإضافة إلى بناة حمولة الوسائط |
    | `plugin-sdk/media-understanding` | أنواع موفر فهم الوسائط بالإضافة إلى تصديرات المساعدة المواجهة للموفر للصور/الصوت |
    | `plugin-sdk/text-runtime` | مساعدات مشتركة للنص/Markdown/التسجيل مثل إزالة النص الظاهر للمساعد، ومساعدات تصيير/تقسيم/جداول Markdown، ومساعدات إخفاء البيانات الحساسة، ومساعدات directive-tag، ومرافق النص الآمن |
    | `plugin-sdk/text-chunking` | مساعد تقسيم النص الصادر |
    | `plugin-sdk/speech` | أنواع موفر الكلام بالإضافة إلى مساعدات directive والسجل والتحقق المواجهة للموفر |
    | `plugin-sdk/speech-core` | أنواع موفر الكلام المشتركة، والسجل، وdirective، ومساعدات التطبيع |
    | `plugin-sdk/realtime-transcription` | أنواع موفر النسخ الفوري ومساعدات السجل |
    | `plugin-sdk/realtime-voice` | أنواع موفر الصوت الفوري ومساعدات السجل |
    | `plugin-sdk/image-generation` | أنواع موفر توليد الصور |
    | `plugin-sdk/image-generation-core` | أنواع توليد الصور المشتركة، والفشل الاحتياطي، والمصادقة، ومساعدات السجل |
    | `plugin-sdk/music-generation` | أنواع موفر/طلب/نتيجة توليد الموسيقى |
    | `plugin-sdk/music-generation-core` | أنواع توليد الموسيقى المشتركة، ومساعدات الفشل الاحتياطي، والبحث عن الموفر، وتحليل model-ref |
    | `plugin-sdk/video-generation` | أنواع موفر/طلب/نتيجة توليد الفيديو |
    | `plugin-sdk/video-generation-core` | أنواع توليد الفيديو المشتركة، ومساعدات الفشل الاحتياطي، والبحث عن الموفر، وتحليل model-ref |
    | `plugin-sdk/webhook-targets` | سجل أهداف webhook ومساعدات تثبيت المسارات |
    | `plugin-sdk/webhook-path` | مساعدات تطبيع مسار webhook |
    | `plugin-sdk/web-media` | مساعدات مشتركة لتحميل الوسائط البعيدة/المحلية |
    | `plugin-sdk/zod` | إعادة تصدير `zod` لمستهلكي Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="المسارات الفرعية للذاكرة">
    | المسار الفرعي | أهم التصديرات |
    | --- | --- |
    | `plugin-sdk/memory-core` | سطح مساعد memory-core المجمّع لمساعدات المدير/التكوين/الملف/CLI |
    | `plugin-sdk/memory-core-engine-runtime` | واجهة وقت التشغيل لمحرك فهرسة/بحث الذاكرة |
    | `plugin-sdk/memory-core-host-engine-foundation` | تصديرات محرك الأساس لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-embeddings` | تصديرات محرك embeddings لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-qmd` | تصديرات محرك QMD لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-storage` | تصديرات محرك التخزين لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-multimodal` | مساعدات الوسائط المتعددة لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-query` | مساعدات الاستعلام لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-secret` | مساعدات الأسرار لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-status` | مساعدات الحالة لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-cli` | مساعدات وقت تشغيل CLI لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-core` | مساعدات وقت التشغيل الأساسية لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-files` | مساعدات الملفات/وقت التشغيل لمضيف الذاكرة |
    | `plugin-sdk/memory-lancedb` | سطح مساعد memory-lancedb المجمّع |
  </Accordion>

  <Accordion title="المسارات الفرعية المحجوزة للمساعدات المجمعة">
    | العائلة | المسارات الفرعية الحالية | الاستخدام المقصود |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | مساعدات دعم إضافة Browser المجمعة (`browser-support` يبقى الملف الجامع الخاص بالتوافق) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | سطح المساعدة/وقت التشغيل لإضافة Matrix المجمعة |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | سطح المساعدة/وقت التشغيل لإضافة LINE المجمعة |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | سطح المساعدة لإضافة IRC المجمعة |
    | مساعدات خاصة بالقنوات | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | موصلات توافق/مساعدة للقنوات المجمعة |
    | مساعدات خاصة بالمصادقة/الإضافة | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | موصلات مساعدة للميزات/الإضافات المجمعة؛ ويصدر `plugin-sdk/github-copilot-token` حاليًا `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken`, و`resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## API التسجيل

يتلقى رد النداء `register(api)` كائن `OpenClawPluginApi` بهذه
الأساليب:

### تسجيل القدرات

| الأسلوب                                          | ما الذي يسجله                 |
| ------------------------------------------------ | ----------------------------- |
| `api.registerProvider(...)`                      | استدلال النص (LLM)            |
| `api.registerChannel(...)`                       | قناة مراسلة                   |
| `api.registerSpeechProvider(...)`                | تحويل النص إلى كلام / STT synthesis |
| `api.registerRealtimeTranscriptionProvider(...)` | نسخ فوري متدفق                |
| `api.registerRealtimeVoiceProvider(...)`         | جلسات صوت فوري ثنائية الاتجاه |
| `api.registerMediaUnderstandingProvider(...)`    | تحليل الصور/الصوت/الفيديو     |
| `api.registerImageGenerationProvider(...)`       | توليد الصور                   |
| `api.registerMusicGenerationProvider(...)`       | توليد الموسيقى                |
| `api.registerVideoGenerationProvider(...)`       | توليد الفيديو                 |
| `api.registerWebFetchProvider(...)`              | موفر جلب / كشط الويب          |
| `api.registerWebSearchProvider(...)`             | البحث في الويب                |

### الأدوات والأوامر

| الأسلوب                        | ما الذي يسجله                                |
| ----------------------------- | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | أداة وكيل (مطلوبة أو `{ optional: true }`) |
| `api.registerCommand(def)`      | أمر مخصص (يتجاوز LLM)                       |

### البنية التحتية

| الأسلوب                                        | ما الذي يسجله      |
| --------------------------------------------- | ------------------ |
| `api.registerHook(events, handler, opts?)`     | event hook         |
| `api.registerHttpRoute(params)`                | نقطة نهاية HTTP للبوابة |
| `api.registerGatewayMethod(name, handler)`     | أسلوب RPC للبوابة  |
| `api.registerCli(registrar, opts?)`            | أمر CLI فرعي       |
| `api.registerService(service)`                 | خدمة تعمل في الخلفية |
| `api.registerInteractiveHandler(registration)` | معالج تفاعلي       |

تظل مساحات أسماء إدارة النواة المحجوزة (`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`) دائمًا ضمن `operator.admin`، حتى إذا حاولت إضافة تعيين
نطاق أضيق لأسلوب البوابة. فضّل البادئات الخاصة بالإضافة من أجل
الأساليب المملوكة للإضافة.

### بيانات تسجيل CLI الوصفية

يقبل `api.registerCli(registrar, opts?)` نوعين من البيانات الوصفية على المستوى الأعلى:

- `commands`: جذور أوامر صريحة يملكها المسجل
- `descriptors`: واصفات أوامر على مستوى التحليل تُستخدم لمساعدة CLI الجذرية،
  والتوجيه، والتسجيل الكسول لـ CLI الخاص بالإضافة

إذا كنت تريد أن يظل أمر الإضافة محمّلًا بكسل في مسار CLI الجذري العادي،
فقدّم `descriptors` تغطي كل جذر أمر على المستوى الأعلى يعرّضه
ذلك المسجل.

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
        description: "Manage Matrix accounts, verification, devices, and profile state",
        hasSubcommands: true,
      },
    ],
  },
);
```

استخدم `commands` وحده فقط عندما لا تحتاج إلى تسجيل CLI كسول على مستوى الجذر.
ما يزال مسار التوافق eager هذا مدعومًا، لكنه لا يثبت
عناصر نائبة مدعومة بـ descriptor من أجل التحميل الكسول وقت التحليل.

### الفتحات الحصرية

| الأسلوب                                    | ما الذي يسجله                        |
| ----------------------------------------- | ------------------------------------ |
| `api.registerContextEngine(id, factory)`   | محرك سياق (واحد فقط نشط في كل مرة)   |
| `api.registerMemoryPromptSection(builder)` | باني قسم prompt للذاكرة             |
| `api.registerMemoryFlushPlan(resolver)`    | محلل خطة تفريغ الذاكرة              |
| `api.registerMemoryRuntime(runtime)`       | مهايئ وقت تشغيل الذاكرة             |

### مهايئات تضمين الذاكرة

| الأسلوب                                        | ما الذي يسجله                                 |
| --------------------------------------------- | --------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | مهايئ تضمين الذاكرة للإضافة النشطة            |

- `registerMemoryPromptSection` و`registerMemoryFlushPlan` و
  `registerMemoryRuntime` حصرية لإضافات الذاكرة.
- يتيح `registerMemoryEmbeddingProvider` لإضافة الذاكرة النشطة تسجيل
  معرّف واحد أو أكثر لمهايئات التضمين (مثل `openai` أو `gemini` أو معرّف
  مخصص معرّف بواسطة الإضافة).
- يُحل تكوين المستخدم مثل `agents.defaults.memorySearch.provider` و
  `agents.defaults.memorySearch.fallback` مقابل معرّفات المهايئات المسجلة.

### الأحداث ودورة الحياة

| الأسلوب                                      | ما الذي يفعله               |
| -------------------------------------------- | --------------------------- |
| `api.on(hookName, handler, opts?)`           | hook دورة حياة typed        |
| `api.onConversationBindingResolved(handler)` | رد نداء ربط المحادثة       |

### دلالات قرار hook

- `before_tool_call`: تكون إعادة `{ block: true }` نهائية. بمجرد أن يضبطها أي معالج، يتم تخطي المعالجات ذات الأولوية الأدنى.
- `before_tool_call`: تُعامل إعادة `{ block: false }` على أنها لا قرار (مثل حذف `block`)، وليس كتجاوز.
- `before_install`: تكون إعادة `{ block: true }` نهائية. بمجرد أن يضبطها أي معالج، يتم تخطي المعالجات ذات الأولوية الأدنى.
- `before_install`: تُعامل إعادة `{ block: false }` على أنها لا قرار (مثل حذف `block`)، وليس كتجاوز.
- `reply_dispatch`: تكون إعادة `{ handled: true, ... }` نهائية. بمجرد أن يدّعي أي معالج الإرسال، يتم تخطي المعالجات ذات الأولوية الأدنى ومسار إرسال النموذج الافتراضي.
- `message_sending`: تكون إعادة `{ cancel: true }` نهائية. بمجرد أن يضبطها أي معالج، يتم تخطي المعالجات ذات الأولوية الأدنى.
- `message_sending`: تُعامل إعادة `{ cancel: false }` على أنها لا قرار (مثل حذف `cancel`)، وليس كتجاوز.

### حقول كائن API

| الحقل                   | النوع                     | الوصف                                                                                         |
| ----------------------- | ------------------------- | --------------------------------------------------------------------------------------------- |
| `api.id`                | `string`                  | معرّف الإضافة                                                                                 |
| `api.name`              | `string`                  | اسم العرض                                                                                     |
| `api.version`           | `string?`                 | إصدار الإضافة (اختياري)                                                                       |
| `api.description`       | `string?`                 | وصف الإضافة (اختياري)                                                                         |
| `api.source`            | `string`                  | مسار مصدر الإضافة                                                                             |
| `api.rootDir`           | `string?`                 | دليل جذر الإضافة (اختياري)                                                                    |
| `api.config`            | `OpenClawConfig`          | لقطة التكوين الحالية (لقطة وقت تشغيل داخل الذاكرة النشطة عند توفرها)                         |
| `api.pluginConfig`      | `Record<string, unknown>` | تكوين خاص بالإضافة من `plugins.entries.<id>.config`                                          |
| `api.runtime`           | `PluginRuntime`           | [Runtime helpers](/ar/plugins/sdk-runtime)                                                       |
| `api.logger`            | `PluginLogger`            | مسجل ضمن النطاق (`debug`, `info`, `warn`, `error`)                                            |
| `api.registrationMode`  | `PluginRegistrationMode`  | وضع التحميل الحالي؛ تكون `"setup-runtime"` نافذة بدء/إعداد خفيفة قبل تحميل الإدخال الكامل |
| `api.resolvePath(input)` | `(string) => string`     | حل المسار نسبة إلى جذر الإضافة                                                                |

## اصطلاح الوحدات الداخلية

داخل إضافتك، استخدم ملفات barrel محلية من أجل عمليات الاستيراد الداخلية:

```
my-plugin/
  api.ts            # تصديرات عامة للمستهلكين الخارجيين
  runtime-api.ts    # تصديرات وقت تشغيل داخلية فقط
  index.ts          # نقطة إدخال الإضافة
  setup-entry.ts    # إدخال خفيف خاص بالإعداد فقط (اختياري)
```

<Warning>
  لا تستورد إضافتك أبدًا من خلال `openclaw/plugin-sdk/<your-plugin>`
  من كود الإنتاج. مرّر الاستيرادات الداخلية عبر `./api.ts` أو
  `./runtime-api.ts`. مسار SDK هو العقد الخارجي فقط.
</Warning>

تفضّل الآن الأسطح العامة للإضافات المجمعة المحمّلة عبر الواجهات (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts`، وملفات الإدخال العامة المشابهة) لقطة تكوين وقت التشغيل النشطة
عندما يكون OpenClaw قيد التشغيل بالفعل. وإذا لم تكن لقطة وقت التشغيل
موجودة بعد، فإنها تعود إلى ملف التكوين المحلول على القرص.

يمكن لإضافات الموفرين أيضًا كشف ملف عقد محلي ودقيق خاص بالإضافة عندما يكون
أحد المساعدات خاصًا عمدًا بالموفر ولا ينتمي بعد إلى مسار فرعي عام من SDK.
مثال مجمع حالي: يحتفظ موفر Anthropic بمساعدات تدفق Claude
داخل الموصل العام الخاص به `api.ts` / `contract-api.ts` بدلًا من
ترقية منطق رؤوس Anthropic beta و`service_tier` إلى عقد
عام ضمن `plugin-sdk/*`.

أمثلة مجمعة حالية أخرى:

- `@openclaw/openai-provider`: يصدّر `api.ts` أدوات بناء الموفر،
  ومساعدات النماذج الافتراضية، وأدوات بناء موفرات الوقت الحقيقي
- `@openclaw/openrouter-provider`: يصدّر `api.ts` أداة بناء الموفر بالإضافة إلى
  مساعدات الإعداد/التكوين

<Warning>
  يجب على كود الإنتاج الخاص بالامتدادات أيضًا تجنب استيرادات
  `openclaw/plugin-sdk/<other-plugin>`. إذا كان أحد المساعدات مشتركًا فعلًا، فقم
  بترقيته إلى مسار فرعي محايد من SDK مثل `openclaw/plugin-sdk/speech` أو
  `.../provider-model-shared` أو سطح آخر موجه بالقدرات بدلًا من
  ربط إضافتين معًا.
</Warning>

## ذو صلة

- [Entry Points](/ar/plugins/sdk-entrypoints) — خيارات `definePluginEntry` و`defineChannelPluginEntry`
- [Runtime Helpers](/ar/plugins/sdk-runtime) — المرجع الكامل لنطاق `api.runtime`
- [Setup and Config](/ar/plugins/sdk-setup) — التعبئة، وmanifest، ومخططات التكوين
- [Testing](/ar/plugins/sdk-testing) — أدوات الاختبار وقواعد lint
- [SDK Migration](/ar/plugins/sdk-migration) — الترحيل من الأسطح المهملة
- [Plugin Internals](/ar/plugins/architecture) — البنية العميقة ونموذج القدرات
