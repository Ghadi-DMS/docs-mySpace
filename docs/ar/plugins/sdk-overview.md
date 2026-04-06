---
read_when:
    - تحتاج إلى معرفة أي مسار فرعي من SDK يجب الاستيراد منه
    - تريد مرجعًا لجميع أساليب التسجيل على OpenClawPluginApi
    - أنت تبحث عن تصدير محدد من SDK
sidebarTitle: SDK Overview
summary: خريطة الاستيراد، ومرجع واجهة برمجة تطبيقات التسجيل، وبنية SDK
title: نظرة عامة على Plugin SDK
x-i18n:
    generated_at: "2026-04-06T07:19:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: acd2887ef52c66b2f234858d812bb04197ecd0bfb3e4f7bf3622f8fdc765acad
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# نظرة عامة على Plugin SDK

يُعد Plugin SDK العقد المطبّع بين الإضافات والنواة. تمثل هذه الصفحة
مرجعًا لـ **ما يجب استيراده** و**ما الذي يمكنك تسجيله**.

<Tip>
  **هل تبحث عن دليل إرشادي؟**
  - أول إضافة؟ ابدأ من [البدء](/ar/plugins/building-plugins)
  - إضافة قناة؟ راجع [إضافات القنوات](/ar/plugins/sdk-channel-plugins)
  - إضافة مزود؟ راجع [إضافات المزودين](/ar/plugins/sdk-provider-plugins)
</Tip>

## اصطلاح الاستيراد

استورد دائمًا من مسار فرعي محدد:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

كل مسار فرعي هو وحدة صغيرة مستقلة بذاتها. يساعد ذلك في إبقاء بدء التشغيل
سريعًا ويمنع مشكلات التبعيات الدائرية. بالنسبة إلى مساعدات الإدخال/البناء
الخاصة بالقنوات، فضّل `openclaw/plugin-sdk/channel-core`؛ واحتفظ بـ
`openclaw/plugin-sdk/core` للسطح الأوسع والمساعدات المشتركة مثل
`buildChannelConfigSchema`.

لا تضف أو تعتمد على واجهات تسهيلية مسماة باسم المزود مثل
`openclaw/plugin-sdk/slack` أو `openclaw/plugin-sdk/discord` أو
`openclaw/plugin-sdk/signal` أو `openclaw/plugin-sdk/whatsapp` أو
واجهات مساعدة موسومة باسم القناة. ينبغي أن تُركّب الإضافات المجمعة
المسارات الفرعية العامة لـ SDK داخل ملفات `api.ts` أو `runtime-api.ts`
الخاصة بها، ويجب أن تستخدم النواة إما هذه الملفات المحلية للإضافة أو
أن تضيف عقدًا عامًا ضيقًا في SDK عندما تكون الحاجة فعلًا عابرة للقنوات.

لا تزال خريطة التصدير المُولدة تتضمن مجموعة صغيرة من واجهات المساعدة الخاصة
بالإضافات المجمعة مثل `plugin-sdk/feishu` و`plugin-sdk/feishu-setup` و
`plugin-sdk/zalo` و`plugin-sdk/zalo-setup` و`plugin-sdk/matrix*`. توجد
هذه المسارات الفرعية لصيانة الإضافات المجمعة والتوافق فقط؛ وهي مستبعدة
عن قصد من الجدول الشائع أدناه وليست مسار الاستيراد الموصى به للإضافات
الطرفية الجديدة.

## مرجع المسارات الفرعية

أكثر المسارات الفرعية استخدامًا، مجمعة حسب الغرض. توجد القائمة الكاملة
المُولدة التي تضم أكثر من 200 مسار فرعي في `scripts/lib/plugin-sdk-entrypoints.json`.

لا تزال المسارات الفرعية المحجوزة لمساعدات الإضافات المجمعة تظهر في تلك
القائمة المُولدة. تعامل معها على أنها أسطح تنفيذ/توافق ما لم تروّج صفحة
توثيقية صراحةً لأحدها على أنه عام.

### إدخال الإضافة

| المسار الفرعي                | أهم الصادرات                                                                                                                          |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                    |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                       |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                      |

<AccordionGroup>
  <Accordion title="المسارات الفرعية للقنوات">
    | المسار الفرعي | أهم الصادرات |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | تصدير مخطط Zod الجذر لـ `openclaw.json` (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`، بالإضافة إلى `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | مساعدات معالج الإعداد المشتركة، ومطالبات قائمة السماح، وبناة حالة الإعداد |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | مساعدات إعدادات/بوابة إجراءات متعددة الحسابات، ومساعدات الرجوع إلى الحساب الافتراضي |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`، ومساعدات تطبيع معرف الحساب |
    | `plugin-sdk/account-resolution` | مساعدات البحث عن الحساب + الرجوع إلى الافتراضي |
    | `plugin-sdk/account-helpers` | مساعدات ضيقة لقائمة الحسابات/إجراءات الحساب |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | أنواع مخطط إعدادات القناة |
    | `plugin-sdk/telegram-command-config` | مساعدات تطبيع/تحقق الأوامر المخصصة في Telegram مع الرجوع إلى العقدة المجمعة |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | مساعدات مشتركة لبناء المسار الداخلي + المغلف |
    | `plugin-sdk/inbound-reply-dispatch` | مساعدات مشتركة للتسجيل الداخلي والتوزيع |
    | `plugin-sdk/messaging-targets` | مساعدات تحليل/مطابقة الأهداف |
    | `plugin-sdk/outbound-media` | مساعدات مشتركة لتحميل الوسائط الصادرة |
    | `plugin-sdk/outbound-runtime` | مساعدات الهوية الصادرة/مندوب الإرسال |
    | `plugin-sdk/thread-bindings-runtime` | دورة حياة ربط الخيوط ومساعدات المهايئ |
    | `plugin-sdk/agent-media-payload` | باني حمولة وسائط الوكيل القديم |
    | `plugin-sdk/conversation-runtime` | ربط المحادثة/الخيط، والاقتران، ومساعدات الربط المُعد |
    | `plugin-sdk/runtime-config-snapshot` | مساعد اللقطة الحالية لإعدادات وقت التشغيل |
    | `plugin-sdk/runtime-group-policy` | مساعدات حل سياسة المجموعة في وقت التشغيل |
    | `plugin-sdk/channel-status` | مساعدات مشتركة للقطة/ملخص حالة القناة |
    | `plugin-sdk/channel-config-primitives` | أوليات ضيقة لمخطط إعدادات القناة |
    | `plugin-sdk/channel-config-writes` | مساعدات تفويض كتابة إعدادات القناة |
    | `plugin-sdk/channel-plugin-common` | صادرات تمهيدية مشتركة لإضافات القنوات |
    | `plugin-sdk/allowlist-config-edit` | مساعدات قراءة/تعديل إعدادات قائمة السماح |
    | `plugin-sdk/group-access` | مساعدات مشتركة لاتخاذ قرارات وصول المجموعات |
    | `plugin-sdk/direct-dm` | مساعدات مشتركة لمصادقة/حراسة الرسائل المباشرة |
    | `plugin-sdk/interactive-runtime` | مساعدات تطبيع/اختزال حمولة الردود التفاعلية |
    | `plugin-sdk/channel-inbound` | مساعدات إزالة التكرار المؤقت، ومطابقة الإشارات، والمغلفات |
    | `plugin-sdk/channel-send-result` | أنواع نتائج الرد |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | مساعدات تحليل/مطابقة الأهداف |
    | `plugin-sdk/channel-contract` | أنواع عقد القناة |
    | `plugin-sdk/channel-feedback` | توصيل الملاحظات/التفاعلات |
  </Accordion>

  <Accordion title="المسارات الفرعية للمزودين">
    | المسار الفرعي | أهم الصادرات |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | مساعدات منسقة لإعداد المزودات المحلية/المستضافة ذاتيًا |
    | `plugin-sdk/self-hosted-provider-setup` | مساعدات مركزة لإعداد مزودات مستضافة ذاتيًا متوافقة مع OpenAI |
    | `plugin-sdk/provider-auth-runtime` | مساعدات حل مفاتيح API وقت التشغيل لإضافات المزودين |
    | `plugin-sdk/provider-auth-api-key` | مساعدات إعداد/كتابة ملفات تعريف مفتاح API |
    | `plugin-sdk/provider-auth-result` | باني نتائج المصادقة OAuth القياسي |
    | `plugin-sdk/provider-auth-login` | مساعدات تسجيل دخول تفاعلية مشتركة لإضافات المزودين |
    | `plugin-sdk/provider-env-vars` | مساعدات البحث عن متغيرات البيئة لمصادقة المزود |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`، وبناة سياسة الإعادة المشتركة، ومساعدات نقاط نهاية المزود، ومساعدات تطبيع معرف النموذج مثل `normalizeNativeXaiModelId` |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | مساعدات عامة لقدرات HTTP/نقاط النهاية للمزود |
    | `plugin-sdk/provider-web-fetch` | مساعدات تسجيل/تخزين مؤقت لمزود الجلب عبر الويب |
    | `plugin-sdk/provider-web-search` | مساعدات تسجيل/تخزين مؤقت/إعدادات لمزود البحث عبر الويب |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`، وتنظيف مخطط Gemini + أدوات التشخيص، ومساعدات التوافق لـ xAI مثل `resolveXaiModelCompatPatch` / `applyXaiModelCompat` |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` وما شابه |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`، وأنواع مغلفات التدفق، ومساعدات المغلفات المشتركة لـ Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot |
    | `plugin-sdk/provider-onboard` | مساعدات تصحيح إعدادات التهيئة |
    | `plugin-sdk/global-singleton` | مساعدات singleton/map/cache محلية للعملية |
  </Accordion>

  <Accordion title="المسارات الفرعية للمصادقة والأمان">
    | المسار الفرعي | أهم الصادرات |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`، ومساعدات سجل الأوامر، ومساعدات تفويض المُرسِل |
    | `plugin-sdk/approval-auth-runtime` | حل الموافقين ومساعدات مصادقة الإجراءات داخل نفس المحادثة |
    | `plugin-sdk/approval-client-runtime` | مساعدات ملف تعريف/تصفية الموافقة على التنفيذ الأصلي |
    | `plugin-sdk/approval-delivery-runtime` | مهايئات قدرات/تسليم الموافقات الأصلية |
    | `plugin-sdk/approval-native-runtime` | مساعدات الهدف الأصلي للموافقة + ربط الحساب |
    | `plugin-sdk/approval-reply-runtime` | مساعدات حمولة رد الموافقة على التنفيذ/الإضافة |
    | `plugin-sdk/command-auth-native` | مصادقة الأوامر الأصلية + مساعدات الهدف الأصلي للجلسة |
    | `plugin-sdk/command-detection` | مساعدات مشتركة لاكتشاف الأوامر |
    | `plugin-sdk/command-surface` | مساعدات تطبيع جسم الأمر وسطح الأمر |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | مساعدات مشتركة للثقة، وتقييد الرسائل المباشرة، والمحتوى الخارجي، وجمع الأسرار |
    | `plugin-sdk/ssrf-policy` | مساعدات قائمة السماح بالمضيفين وسياسة SSRF للشبكة الخاصة |
    | `plugin-sdk/ssrf-runtime` | مساعدات pinned-dispatcher وfetch المحمي من SSRF وسياسة SSRF |
    | `plugin-sdk/secret-input` | مساعدات تحليل إدخال الأسرار |
    | `plugin-sdk/webhook-ingress` | مساعدات طلب/هدف webhook |
    | `plugin-sdk/webhook-request-guards` | مساعدات حجم جسم الطلب/المهلة الزمنية |
  </Accordion>

  <Accordion title="المسارات الفرعية لوقت التشغيل والتخزين">
    | المسار الفرعي | أهم الصادرات |
    | --- | --- |
    | `plugin-sdk/runtime` | مساعدات واسعة لوقت التشغيل/التسجيل/النسخ الاحتياطي/تثبيت الإضافات |
    | `plugin-sdk/runtime-env` | مساعدات ضيقة لبيئة وقت التشغيل، والمسجل، والمهلة، وإعادة المحاولة، والتراجع |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | مساعدات مشتركة للأوامر/الخطافات/http/التفاعل الخاصة بالإضافة |
    | `plugin-sdk/hook-runtime` | مساعدات مشتركة لخط أنابيب webhook/الخطافات الداخلية |
    | `plugin-sdk/lazy-runtime` | مساعدات الاستيراد/الربط الكسول لوقت التشغيل مثل `createLazyRuntimeModule` و`createLazyRuntimeMethod` و`createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | مساعدات تنفيذ العمليات |
    | `plugin-sdk/cli-runtime` | مساعدات تنسيق CLI والانتظار والإصدار |
    | `plugin-sdk/gateway-runtime` | مساعدات عميل Gateway وتصحيح حالة القناة |
    | `plugin-sdk/config-runtime` | مساعدات تحميل/كتابة الإعدادات |
    | `plugin-sdk/telegram-command-config` | مساعدات تطبيع أسماء/أوصاف أوامر Telegram والتحقق من التكرارات/التعارضات، حتى عندما يكون سطح عقد Telegram المجمّع غير متاح |
    | `plugin-sdk/approval-runtime` | مساعدات الموافقة على التنفيذ/الإضافة، وبناة قدرات الموافقة، ومساعدات المصادقة/الملف الشخصي، ومساعدات التوجيه/وقت التشغيل الأصلية |
    | `plugin-sdk/reply-runtime` | مساعدات مشتركة لوقت التشغيل الداخلي/الرد، والتقسيم، والتوزيع، والنبض، ومخطط الرد |
    | `plugin-sdk/reply-dispatch-runtime` | مساعدات ضيقة لتوزيع/إنهاء الرد |
    | `plugin-sdk/reply-history` | مساعدات مشتركة لسجل الردود ضمن نافذة قصيرة مثل `buildHistoryContext` و`recordPendingHistoryEntry` و`clearHistoryEntriesIfEnabled` |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | مساعدات ضيقة لتقسيم النص/Markdown |
    | `plugin-sdk/session-store-runtime` | مساعدات مسار مخزن الجلسة + `updated-at` |
    | `plugin-sdk/state-paths` | مساعدات مسارات دليل الحالة/OAuth |
    | `plugin-sdk/routing` | مساعدات المسار/مفتاح الجلسة/ربط الحساب مثل `resolveAgentRoute` و`buildAgentSessionKey` و`resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | مساعدات مشتركة لملخص حالة القناة/الحساب، والإعدادات الافتراضية لحالة وقت التشغيل، ومساعدات بيانات المشكلات |
    | `plugin-sdk/target-resolver-runtime` | مساعدات مشتركة لحل الأهداف |
    | `plugin-sdk/string-normalization-runtime` | مساعدات تطبيع slug/السلاسل النصية |
    | `plugin-sdk/request-url` | استخراج عناوين URL النصية من مدخلات شبيهة بـ fetch/request |
    | `plugin-sdk/run-command` | مشغّل أوامر موقّت مع نتائج stdout/stderr مطبّعة |
    | `plugin-sdk/param-readers` | قارئات معاملات شائعة للأدوات/CLI |
    | `plugin-sdk/tool-send` | استخراج حقول هدف الإرسال القياسية من معاملات الأداة |
    | `plugin-sdk/temp-path` | مساعدات مشتركة لمسارات تنزيل الملفات المؤقتة |
    | `plugin-sdk/logging-core` | مساعدات مسجل النظام الفرعي وإخفاء البيانات الحساسة |
    | `plugin-sdk/markdown-table-runtime` | مساعدات أوضاع جداول Markdown |
    | `plugin-sdk/json-store` | مساعدات صغيرة لقراءة/كتابة حالة JSON |
    | `plugin-sdk/file-lock` | مساعدات قفل ملفات قابلة لإعادة الدخول |
    | `plugin-sdk/persistent-dedupe` | مساعدات ذاكرة تخزين مؤقت لإزالة التكرار مدعومة بالقرص |
    | `plugin-sdk/acp-runtime` | مساعدات ACP لوقت التشغيل/الجلسة وتوزيع الرد |
    | `plugin-sdk/agent-config-primitives` | أوليات ضيقة لمخطط إعدادات وقت تشغيل الوكيل |
    | `plugin-sdk/boolean-param` | قارئ معاملات منطقي مرن |
    | `plugin-sdk/dangerous-name-runtime` | مساعدات حل مطابقة الأسماء الخطرة |
    | `plugin-sdk/device-bootstrap` | مساعدات تمهيد الجهاز ورموز الاقتران |
    | `plugin-sdk/extension-shared` | أوليات مشتركة للقنوات السلبية ومساعدات الحالة |
    | `plugin-sdk/models-provider-runtime` | مساعدات رد `/models` والأوامر الخاصة بالمزود |
    | `plugin-sdk/skill-commands-runtime` | مساعدات سرد أوامر Skills |
    | `plugin-sdk/native-command-registry` | مساعدات سجل/بناء/تسلسل الأوامر الأصلية |
    | `plugin-sdk/provider-zai-endpoint` | مساعدات اكتشاف نقطة نهاية Z.AI |
    | `plugin-sdk/infra-runtime` | مساعدات أحداث النظام/النبض |
    | `plugin-sdk/collection-runtime` | مساعدات صغيرة لذاكرة تخزين مؤقت محدودة |
    | `plugin-sdk/diagnostic-runtime` | مساعدات أعلام وأحداث التشخيص |
    | `plugin-sdk/error-runtime` | مساعدات مخطط الأخطاء، والتنسيق، وتصنيف الأخطاء المشترك، و`isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | مساعدات fetch المغلف، والوكيل، والبحث المثبّت |
    | `plugin-sdk/host-runtime` | مساعدات تطبيع اسم المضيف ومضيف SCP |
    | `plugin-sdk/retry-runtime` | مساعدات إعداد إعادة المحاولة ومشغّل إعادة المحاولة |
    | `plugin-sdk/agent-runtime` | مساعدات دليل/هوية/مساحة عمل الوكيل |
    | `plugin-sdk/directory-runtime` | استعلام/إزالة تكرار الأدلة المعتمد على الإعدادات |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="المسارات الفرعية للقدرات والاختبار">
    | المسار الفرعي | أهم الصادرات |
    | --- | --- |
    | `plugin-sdk/media-runtime` | مساعدات مشتركة لجلب/تحويل/تخزين الوسائط بالإضافة إلى بناة حمولات الوسائط |
    | `plugin-sdk/media-generation-runtime` | مساعدات مشتركة للتعامل مع فشل توليد الوسائط، واختيار المرشحين، ورسائل غياب النموذج |
    | `plugin-sdk/media-understanding` | أنواع مزودي فهم الوسائط بالإضافة إلى صادرات مساعدات الصور/الصوت الموجهة للمزود |
    | `plugin-sdk/text-runtime` | مساعدات مشتركة للنص/Markdown/التسجيل مثل إزالة النص المرئي للمساعد، ومساعدات عرض/تقسيم/جداول Markdown، ومساعدات إخفاء البيانات، ومساعدات وسم التوجيه، وأدوات النص الآمن |
    | `plugin-sdk/text-chunking` | مساعد تقسيم النص الصادر |
    | `plugin-sdk/speech` | أنواع مزودي الكلام بالإضافة إلى صادرات مساعدات التوجيه والسجل والتحقق الموجهة للمزود |
    | `plugin-sdk/speech-core` | أنواع مزودي الكلام المشتركة، والسجل، والتوجيه، ومساعدات التطبيع |
    | `plugin-sdk/realtime-transcription` | أنواع مزودي النسخ الفوري ومساعدات السجل |
    | `plugin-sdk/realtime-voice` | أنواع مزودي الصوت الفوري ومساعدات السجل |
    | `plugin-sdk/image-generation` | أنواع مزودي توليد الصور |
    | `plugin-sdk/image-generation-core` | أنواع مشتركة لتوليد الصور، ومساعدات التعامل مع الفشل، والمصادقة، والسجل |
    | `plugin-sdk/music-generation` | أنواع مزود/طلب/نتيجة توليد الموسيقى |
    | `plugin-sdk/music-generation-core` | أنواع مشتركة لتوليد الموسيقى، ومساعدات التعامل مع الفشل، والبحث عن المزود، وتحليل مرجع النموذج |
    | `plugin-sdk/video-generation` | أنواع مزود/طلب/نتيجة توليد الفيديو |
    | `plugin-sdk/video-generation-core` | أنواع مشتركة لتوليد الفيديو، ومساعدات التعامل مع الفشل، والبحث عن المزود، وتحليل مرجع النموذج |
    | `plugin-sdk/webhook-targets` | مساعدات سجل أهداف webhook وتثبيت المسارات |
    | `plugin-sdk/webhook-path` | مساعدات تطبيع مسار webhook |
    | `plugin-sdk/web-media` | مساعدات مشتركة لتحميل الوسائط البعيدة/المحلية |
    | `plugin-sdk/zod` | إعادة تصدير `zod` لمستهلكي Plugin SDK |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="المسارات الفرعية للذاكرة">
    | المسار الفرعي | أهم الصادرات |
    | --- | --- |
    | `plugin-sdk/memory-core` | سطح مساعدات memory-core المجمّع لمدير/إعدادات/ملفات/مساعدات CLI |
    | `plugin-sdk/memory-core-engine-runtime` | واجهة وقت تشغيل لفهرسة/بحث الذاكرة |
    | `plugin-sdk/memory-core-host-engine-foundation` | صادرات محرك الأساس لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-embeddings` | صادرات محرك التضمينات لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-qmd` | صادرات محرك QMD لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-engine-storage` | صادرات محرك التخزين لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-multimodal` | مساعدات متعددة الوسائط لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-query` | مساعدات الاستعلام لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-secret` | مساعدات الأسرار لمضيف الذاكرة |
    | `plugin-sdk/memory-core-host-events` | مساعدات سجل أحداث مضيف الذاكرة |
    | `plugin-sdk/memory-core-host-status` | مساعدات حالة مضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-cli` | مساعدات CLI لوقت تشغيل مضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-core` | مساعدات النواة لوقت تشغيل مضيف الذاكرة |
    | `plugin-sdk/memory-core-host-runtime-files` | مساعدات الملفات/وقت التشغيل لمضيف الذاكرة |
    | `plugin-sdk/memory-host-core` | اسم بديل محايد للمورّد لمساعدات نواة وقت تشغيل مضيف الذاكرة |
    | `plugin-sdk/memory-host-events` | اسم بديل محايد للمورّد لمساعدات سجل أحداث مضيف الذاكرة |
    | `plugin-sdk/memory-host-files` | اسم بديل محايد للمورّد لمساعدات ملفات/وقت تشغيل مضيف الذاكرة |
    | `plugin-sdk/memory-host-markdown` | مساعدات مشتركة لـ Markdown المُدار للإضافات القريبة من الذاكرة |
    | `plugin-sdk/memory-host-search` | واجهة وقت تشغيل الذاكرة النشطة للوصول إلى مدير البحث |
    | `plugin-sdk/memory-host-status` | اسم بديل محايد للمورّد لمساعدات حالة مضيف الذاكرة |
    | `plugin-sdk/memory-lancedb` | سطح مساعدات memory-lancedb المجمّع |
  </Accordion>

  <Accordion title="المسارات الفرعية المحجوزة للمساعدات المجمعة">
    | العائلة | المسارات الفرعية الحالية | الاستخدام المقصود |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | مساعدات دعم إضافة Browser المجمعة (`browser-support` يبقى الواجهة المتوافقة) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | سطح مساعدات/وقت تشغيل Matrix المجمّع |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | سطح مساعدات/وقت تشغيل LINE المجمّع |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | سطح مساعدات IRC المجمّع |
    | مساعدات خاصة بالقنوات | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | واجهات توافق/مساعدة لقنوات مجمعة |
    | مساعدات خاصة بالمصادقة/الإضافة | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | واجهات مساعدة للميزات/الإضافات المجمعة؛ يصدّر `plugin-sdk/github-copilot-token` حاليًا `DEFAULT_COPILOT_API_BASE_URL` و`deriveCopilotApiBaseUrlFromToken` و`resolveCopilotApiToken` |
  </Accordion>
</AccordionGroup>

## واجهة برمجة تطبيقات التسجيل

يتلقى رد النداء `register(api)` كائن `OpenClawPluginApi` بهذه
الأساليب:

### تسجيل القدرات

| الأسلوب                                         | ما الذي يسجله                  |
| ------------------------------------------------ | ------------------------------ |
| `api.registerProvider(...)`                      | استدلال النصوص (LLM)           |
| `api.registerChannel(...)`                       | قناة مراسلة                    |
| `api.registerSpeechProvider(...)`                | تحويل النص إلى كلام / تركيب STT |
| `api.registerRealtimeTranscriptionProvider(...)` | نسخ فوري متدفق                 |
| `api.registerRealtimeVoiceProvider(...)`         | جلسات صوتية فورية ثنائية الاتجاه |
| `api.registerMediaUnderstandingProvider(...)`    | تحليل الصور/الصوت/الفيديو      |
| `api.registerImageGenerationProvider(...)`       | توليد الصور                    |
| `api.registerMusicGenerationProvider(...)`       | توليد الموسيقى                 |
| `api.registerVideoGenerationProvider(...)`       | توليد الفيديو                  |
| `api.registerWebFetchProvider(...)`              | مزود جلب / كشط ويب             |
| `api.registerWebSearchProvider(...)`             | بحث ويب                        |

### الأدوات والأوامر

| الأسلوب                          | ما الذي يسجله                                 |
| ------------------------------- | --------------------------------------------- |
| `api.registerTool(tool, opts?)` | أداة وكيل (مطلوبة أو `{ optional: true }`)    |
| `api.registerCommand(def)`      | أمر مخصص (يتجاوز LLM)                         |

### البنية التحتية

| الأسلوب                                         | ما الذي يسجله                         |
| ---------------------------------------------- | ------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | خطاف أحداث                            |
| `api.registerHttpRoute(params)`                | نقطة نهاية HTTP في Gateway            |
| `api.registerGatewayMethod(name, handler)`     | أسلوب Gateway RPC                     |
| `api.registerCli(registrar, opts?)`            | أمر فرعي في CLI                       |
| `api.registerService(service)`                 | خدمة تعمل في الخلفية                  |
| `api.registerInteractiveHandler(registration)` | معالج تفاعلي                          |
| `api.registerMemoryPromptSupplement(builder)`  | قسم مطالبات إضافي مجاور للذاكرة       |
| `api.registerMemoryCorpusSupplement(adapter)`  | متن إضافي للبحث/القراءة في الذاكرة    |

تظل مساحات أسماء إدارة النواة المحجوزة (`config.*` و`exec.approvals.*` و`wizard.*` و
`update.*`) دائمًا ضمن `operator.admin`، حتى إذا حاولت إضافة تعيين
نطاق أضيق لأسلوب Gateway. فضّل بادئات خاصة بالإضافة
للأساليب المملوكة لها.

### بيانات تعريف تسجيل CLI

يقبل `api.registerCli(registrar, opts?)` نوعين من البيانات التعريفية
على المستوى الأعلى:

- `commands`: جذور أوامر صريحة يملكها المُسجِّل
- `descriptors`: واصفات أوامر وقت التحليل المستخدمة لمساعدة CLI الجذرية،
  والتوجيه، وتسجيل CLI الكسول للإضافات

إذا أردت أن يظل أمر الإضافة محمّلًا كسولًا في مسار CLI الجذري العادي،
فوفّر `descriptors` تغطي كل جذر أمر من المستوى الأعلى يكشفه ذلك
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
        description: "إدارة حسابات Matrix والتحقق والأجهزة وحالة الملف الشخصي",
        hasSubcommands: true,
      },
    ],
  },
);
```

استخدم `commands` وحده فقط عندما لا تحتاج إلى تسجيل CLI كسول للجذر.
لا يزال مسار التوافق المتعجل هذا مدعومًا، لكنه لا يثبت عناصر نائبة
مدعومة بالواصفات للتحميل الكسول في وقت التحليل.

### الفتحات الحصرية

| الأسلوب                                     | ما الذي يسجله                       |
| ------------------------------------------ | ----------------------------------- |
| `api.registerContextEngine(id, factory)`   | محرك سياق (واحد نشط في كل مرة)      |
| `api.registerMemoryPromptSection(builder)` | باني قسم مطالبات الذاكرة            |
| `api.registerMemoryFlushPlan(resolver)`    | محلل خطة تفريغ الذاكرة              |
| `api.registerMemoryRuntime(runtime)`       | مهايئ وقت تشغيل الذاكرة             |

### مهايئات تضمين الذاكرة

| الأسلوب                                         | ما الذي يسجله                                |
| ---------------------------------------------- | -------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | مهايئ تضمين الذاكرة للإضافة النشطة           |

- `registerMemoryPromptSection` و`registerMemoryFlushPlan` و
  `registerMemoryRuntime` حصرية لإضافات الذاكرة.
- يتيح `registerMemoryEmbeddingProvider` لإضافة الذاكرة النشطة تسجيل
  معرف مهايئ تضمين واحد أو أكثر (مثل `openai` أو `gemini` أو معرف
  مخصص تعرّفه الإضافة).
- تُحل إعدادات المستخدم مثل `agents.defaults.memorySearch.provider` و
  `agents.defaults.memorySearch.fallback` وفقًا لمعرفات المهايئات
  المسجلة تلك.

### الأحداث ودورة الحياة

| الأسلوب                                       | ما الذي يفعله              |
| -------------------------------------------- | -------------------------- |
| `api.on(hookName, handler, opts?)`           | خطاف دورة حياة مطبّع       |
| `api.onConversationBindingResolved(handler)` | رد نداء ربط المحادثة       |

### دلالات قرارات الخطافات

- `before_tool_call`: تكون إعادة `{ block: true }` نهائية. بمجرد أن يعيّنها أي معالج، يتم تخطي المعالجات الأقل أولوية.
- `before_tool_call`: تُعامل إعادة `{ block: false }` على أنها بلا قرار (مثل حذف `block`)، وليس على أنها تجاوز.
- `before_install`: تكون إعادة `{ block: true }` نهائية. بمجرد أن يعيّنها أي معالج، يتم تخطي المعالجات الأقل أولوية.
- `before_install`: تُعامل إعادة `{ block: false }` على أنها بلا قرار (مثل حذف `block`)، وليس على أنها تجاوز.
- `reply_dispatch`: تكون إعادة `{ handled: true, ... }` نهائية. بمجرد أن يطالب أي معالج بالتوزيع، يتم تخطي المعالجات الأقل أولوية ومسار توزيع النموذج الافتراضي.
- `message_sending`: تكون إعادة `{ cancel: true }` نهائية. بمجرد أن يعيّنها أي معالج، يتم تخطي المعالجات الأقل أولوية.
- `message_sending`: تُعامل إعادة `{ cancel: false }` على أنها بلا قرار (مثل حذف `cancel`)، وليس على أنها تجاوز.

### حقول كائن API

| الحقل                    | النوع                      | الوصف                                                                                         |
| ------------------------ | ------------------------- | --------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | معرف الإضافة                                                                                  |
| `api.name`               | `string`                  | اسم العرض                                                                                     |
| `api.version`            | `string?`                 | إصدار الإضافة (اختياري)                                                                       |
| `api.description`        | `string?`                 | وصف الإضافة (اختياري)                                                                         |
| `api.source`             | `string`                  | مسار مصدر الإضافة                                                                             |
| `api.rootDir`            | `string?`                 | الدليل الجذري للإضافة (اختياري)                                                               |
| `api.config`             | `OpenClawConfig`          | لقطة الإعدادات الحالية (اللقطة الحالية داخل الذاكرة لوقت التشغيل عند توفرها)                  |
| `api.pluginConfig`       | `Record<string, unknown>` | إعدادات خاصة بالإضافة من `plugins.entries.<id>.config`                                        |
| `api.runtime`            | `PluginRuntime`           | [مساعدات وقت التشغيل](/ar/plugins/sdk-runtime)                                                   |
| `api.logger`             | `PluginLogger`            | مسجل ذو نطاق محدد (`debug`, `info`, `warn`, `error`)                                          |
| `api.registrationMode`   | `PluginRegistrationMode`  | وضع التحميل الحالي؛ تمثل `"setup-runtime"` نافذة بدء التشغيل/الإعداد الخفيفة قبل الإدخال الكامل |
| `api.resolvePath(input)` | `(string) => string`      | حل المسار نسبةً إلى جذر الإضافة                                                                |

## اصطلاح الوحدات الداخلية

داخل إضافتك، استخدم ملفات barrel محلية للاستيرادات الداخلية:

```
my-plugin/
  api.ts            # صادرات عامة للمستهلكين الخارجيين
  runtime-api.ts    # صادرات داخلية فقط لوقت التشغيل
  index.ts          # نقطة إدخال الإضافة
  setup-entry.ts    # إدخال خفيف للإعداد فقط (اختياري)
```

<Warning>
  لا تستورد إضافتك نفسها عبر `openclaw/plugin-sdk/<your-plugin>`
  من كود الإنتاج. وجّه الاستيرادات الداخلية عبر `./api.ts` أو
  `./runtime-api.ts`. مسار SDK هو العقد الخارجي فقط.
</Warning>

تفضّل الآن الأسطح العامة للإضافات المجمعة المحمّلة عبر الواجهة (`api.ts` و`runtime-api.ts` و
`index.ts` و`setup-entry.ts` وملفات الإدخال العامة المماثلة)
اللقطة النشطة لإعدادات وقت التشغيل عندما يكون OpenClaw قيد التشغيل بالفعل. وإذا لم
توجد لقطة وقت تشغيل بعد، فسيتم الرجوع إلى ملف الإعدادات المحلول على القرص.

يمكن لإضافات المزودين أيضًا كشف ملف contract محلي ضيق خاص بالإضافة عندما
يكون أحد المساعدات خاصًا بالمزود عمدًا ولا ينتمي بعد إلى مسار فرعي عام
في SDK. المثال المجمّع الحالي: يحتفظ مزود Anthropic بمساعدات تدفق Claude
في واجهته العامة `api.ts` / `contract-api.ts` الخاصة به بدلًا من ترقية
منطق رأس Anthropic beta و`service_tier` إلى عقد عام
`plugin-sdk/*`.

أمثلة مجمعة حالية أخرى:

- `@openclaw/openai-provider`: يصدّر `api.ts` بناة المزودات،
  ومساعدات النماذج الافتراضية، وبناة المزودات الفورية
- `@openclaw/openrouter-provider`: يصدّر `api.ts` باني المزود بالإضافة إلى
  مساعدات التهيئة/الإعدادات

<Warning>
  يجب أن يتجنب كود الإنتاج الخاص بالامتدادات أيضًا استيراد
  `openclaw/plugin-sdk/<other-plugin>`. إذا كانت أداة ما مشتركة فعلًا،
  فقم بترقيتها إلى مسار فرعي محايد في SDK مثل
  `openclaw/plugin-sdk/speech` أو `.../provider-model-shared` أو سطح آخر
  موجّه بالقدرات بدلًا من ربط إضافتين معًا.
</Warning>

## ذو صلة

- [نقاط الإدخال](/ar/plugins/sdk-entrypoints) — خيارات `definePluginEntry` و`defineChannelPluginEntry`
- [مساعدات وقت التشغيل](/ar/plugins/sdk-runtime) — المرجع الكامل لمساحة الأسماء `api.runtime`
- [الإعداد والتهيئة](/ar/plugins/sdk-setup) — الحزم، وملفات البيان، ومخططات الإعدادات
- [الاختبار](/ar/plugins/sdk-testing) — أدوات الاختبار وقواعد lint
- [ترحيل SDK](/ar/plugins/sdk-migration) — الترحيل من الأسطح المهجورة
- [البنية الداخلية للإضافات](/ar/plugins/architecture) — البنية العميقة ونموذج القدرات
