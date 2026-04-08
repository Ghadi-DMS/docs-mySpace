---
read_when:
    - أنت تبني إضافة قناة مراسلة جديدة
    - تريد ربط OpenClaw بمنصة مراسلة
    - تحتاج إلى فهم سطح محوّل ChannelPlugin
sidebarTitle: Channel Plugins
summary: دليل خطوة بخطوة لبناء إضافة قناة مراسلة لـ OpenClaw
title: بناء إضافات القنوات
x-i18n:
    generated_at: "2026-04-08T02:18:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: d23365b6d92006b30e671f9f0afdba40a2b88c845c5d2299d71c52a52985672f
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# بناء إضافات القنوات

يرشدك هذا الدليل خلال بناء إضافة قناة تربط OpenClaw بمنصة
مراسلة. وبنهاية هذا الدليل سيكون لديك قناة عاملة تتضمن أمان الرسائل المباشرة،
والاقتران، وتسلسل الردود، والمراسلة الصادرة.

<Info>
  إذا لم تكن قد بنيت أي إضافة OpenClaw من قبل، فاقرأ
  [البدء](/ar/plugins/building-plugins) أولًا للاطلاع على البنية الأساسية
  للحزمة وإعداد manifest.
</Info>

## كيف تعمل إضافات القنوات

لا تحتاج إضافات القنوات إلى أدوات send/edit/react خاصة بها. يحتفظ OpenClaw
بأداة `message` مشتركة واحدة في النواة. وتملك إضافتك ما يلي:

- **الإعدادات** — حلّ الحسابات ومعالج الإعداد
- **الأمان** — سياسة الرسائل المباشرة وقوائم السماح
- **الاقتران** — تدفق الموافقة على الرسائل المباشرة
- **بنية الجلسة** — كيفية ربط معرّفات المحادثات الخاصة بالمزوّد بالدردشات الأساسية ومعرّفات السلاسل والبدائل الأصلية
- **الصادر** — إرسال النصوص والوسائط والاستطلاعات إلى المنصة
- **التسلسل** — كيفية ربط الردود ضمن السلاسل

تملك النواة أداة الرسائل المشتركة، وربط الموجّهات، وشكل مفتاح الجلسة الخارجي،
وإدارة `:thread:` العامة، والتوزيع.

إذا كانت منصتك تخزن نطاقًا إضافيًا داخل معرّفات المحادثات، فاحتفظ بهذا التحليل
داخل الإضافة باستخدام `messaging.resolveSessionConversation(...)`. هذا هو
الخطاف القياسي لتحويل `rawId` إلى معرّف المحادثة الأساسي، ومعرّف السلسلة الاختياري،
و`baseConversationId` الصريح، وأي `parentConversationCandidates`.
عندما تُرجع `parentConversationCandidates`، احرص على ترتيبها من
الأصل الأضيق إلى المحادثة الأوسع/الأساسية.

يمكن أيضًا للإضافات المضمّنة التي تحتاج إلى التحليل نفسه قبل إقلاع
سجل القنوات أن تعرض ملف `session-key-api.ts` على المستوى الأعلى مع
تصدير مطابق `resolveSessionConversation(...)`. تستخدم النواة هذا السطح
الآمن للإقلاع فقط عندما لا يكون سجل الإضافات وقت التشغيل متاحًا بعد.

يبقى `messaging.resolveParentConversationCandidates(...)` متاحًا
كبديل توافق قديم عندما تحتاج الإضافة فقط إلى بدائل أصلية فوق
المعرّف العام/الخام. إذا وُجد الخطافان معًا، تستخدم النواة
`resolveSessionConversation(...).parentConversationCandidates` أولًا ولا
تعود إلى `resolveParentConversationCandidates(...)` إلا عندما يتجاهلها
الخطاف القياسي.

## الموافقات وقدرات القناة

لا تحتاج معظم إضافات القنوات إلى تعليمات برمجية خاصة بالموافقات.

- تملك النواة الأمر `/approve` داخل الدردشة نفسها، وحمولات أزرار الموافقة المشتركة، وآلية التسليم العامة البديلة.
- فضّل استخدام كائن `approvalCapability` واحد في إضافة القناة عندما تحتاج القناة إلى سلوك خاص بالموافقة.
- تمت إزالة `ChannelPlugin.approvals`. ضع حقائق تسليم/أصلية/عرض/مصادقة الموافقة في `approvalCapability`.
- يقتصر `plugin.auth` على login/logout فقط؛ ولم تعد النواة تقرأ خطافات مصادقة الموافقة من ذلك الكائن.
- يشكّل كل من `approvalCapability.authorizeActorAction` و`approvalCapability.getActionAvailabilityState` سطح مصادقة الموافقة القياسي.
- استخدم `approvalCapability.getActionAvailabilityState` لتوافر مصادقة الموافقة داخل الدردشة نفسها.
- إذا كانت قناتك تعرض موافقات exec أصلية، فاستخدم `approvalCapability.getExecInitiatingSurfaceState` لحالة السطح المُطلِق/العميل الأصلي عندما تختلف عن مصادقة الموافقة داخل الدردشة نفسها. تستخدم النواة هذا الخطاف الخاص بـ exec للتمييز بين `enabled` و`disabled`، ولتحديد ما إذا كانت القناة المُطلِقة تدعم موافقات exec الأصلية، ولتضمين القناة في إرشادات بدائل العميل الأصلي. يملأ `createApproverRestrictedNativeApprovalCapability(...)` هذا للحالة الشائعة.
- استخدم `outbound.shouldSuppressLocalPayloadPrompt` أو `outbound.beforeDeliverPayload` لسلوك دورة حياة الحمولة الخاص بالقناة مثل إخفاء مطالبات الموافقة المحلية المكررة أو إرسال مؤشرات الكتابة قبل التسليم.
- استخدم `approvalCapability.delivery` فقط لتوجيه الموافقة الأصلية أو منع التسليم البديل.
- استخدم `approvalCapability.nativeRuntime` للحقائق الأصلية للموافقة المملوكة للقناة. احرص على إبقائه كسولًا في نقاط دخول القناة الساخنة باستخدام `createLazyChannelApprovalNativeRuntimeAdapter(...)`، الذي يمكنه استيراد وحدة وقت التشغيل عند الطلب مع استمرار قدرة النواة على تجميع دورة حياة الموافقة.
- استخدم `approvalCapability.render` فقط عندما تحتاج القناة فعلًا إلى حمولات موافقة مخصصة بدلًا من العارض المشترك.
- استخدم `approvalCapability.describeExecApprovalSetup` عندما تريد القناة أن يشرح رد مسار التعطيل مفاتيح الإعداد الدقيقة اللازمة لتمكين موافقات exec الأصلية. يتلقى الخطاف `{ channel, channelLabel, accountId }`؛ ويجب على القنوات ذات الحسابات المسمّاة عرض مسارات على مستوى الحساب مثل `channels.<channel>.accounts.<id>.execApprovals.*` بدلًا من الإعدادات الافتراضية العليا.
- إذا كانت القناة تستطيع استنتاج هويات DM مستقرة شبيهة بالمالك من الإعدادات الحالية، فاستخدم `createResolvedApproverActionAuthAdapter` من `openclaw/plugin-sdk/approval-runtime` لتقييد `/approve` داخل الدردشة نفسها دون إضافة منطق أساسي خاص بالموافقة.
- إذا احتاجت القناة إلى تسليم موافقة أصلية، فأبقِ تعليمات القناة البرمجية مركّزة على تطبيع الهدف وحقائق النقل/العرض. استخدم `createChannelExecApprovalProfile` و`createChannelNativeOriginTargetResolver` و`createChannelApproverDmTargetResolver` و`createApproverRestrictedNativeApprovalCapability` من `openclaw/plugin-sdk/approval-runtime`. ضع الحقائق الخاصة بالقناة خلف `approvalCapability.nativeRuntime`، ويفضل عبر `createChannelApprovalNativeRuntimeAdapter(...)` أو `createLazyChannelApprovalNativeRuntimeAdapter(...)`، حتى تتمكن النواة من تجميع المعالج وامتلاك تصفية الطلبات والتوجيه وإزالة التكرار والانتهاء والاشتراك في البوابة وإشعارات التوجيه إلى مكان آخر. ينقسم `nativeRuntime` إلى عدة أسطح أصغر:
- `availability` — ما إذا كان الحساب مضبوطًا وما إذا كان ينبغي التعامل مع الطلب
- `presentation` — تحويل نموذج عرض الموافقة المشترك إلى حمولات أصلية معلقة/محسومة/منتهية أو إجراءات نهائية
- `transport` — تجهيز الأهداف وإرسال/تحديث/حذف رسائل الموافقة الأصلية
- `interactions` — خطافات bind/unbind/clear-action اختيارية للأزرار أو التفاعلات الأصلية
- `observe` — خطافات اختيارية لتشخيصات التسليم
- إذا احتاجت القناة إلى كائنات مملوكة لوقت التشغيل مثل عميل أو رمز أو تطبيق Bolt أو مستقبل webhook، فسجّلها عبر `openclaw/plugin-sdk/channel-runtime-context`. يتيح سجل runtime-context العام للنواة تهيئة معالجات مدفوعة بالقدرات من حالة بدء تشغيل القناة دون إضافة غراء تغليف خاص بالموافقة.
- لا تلجأ إلى `createChannelApprovalHandler` أو `createChannelNativeApprovalRuntime` ذوي المستوى الأدنى إلا عندما لا يكون السطح المدفوع بالقدرات معبّرًا بما يكفي بعد.
- يجب على قنوات الموافقة الأصلية تمرير كل من `accountId` و`approvalKind` عبر هذه المساعدات. يحافظ `accountId` على حصر سياسة الموافقة متعددة الحسابات ضمن حساب البوت الصحيح، ويحافظ `approvalKind` على إتاحة سلوك موافقات exec مقابل الإضافة للقناة دون فروع مضمنة صراحة في النواة.
- أصبحت النواة الآن تملك إشعارات إعادة توجيه الموافقات أيضًا. يجب ألا ترسل إضافات القنوات رسائل متابعة خاصة بها من نوع "تم إرسال الموافقة إلى الرسائل المباشرة / إلى قناة أخرى" من `createChannelNativeApprovalRuntime`؛ بل اعرض بدلًا من ذلك توجيه الأصل + الرسائل المباشرة للموافق عبر مساعدات قدرة الموافقة المشتركة واترك للنواة تجميع عمليات التسليم الفعلية قبل نشر أي إشعار إلى الدردشة المُطلِقة.
- حافظ على نوع معرّف الموافقة المسلَّم من البداية إلى النهاية. يجب ألا
  يخمّن العملاء الأصليون أو يعيدوا كتابة توجيه موافقة exec مقابل الإضافة من حالة محلية بالقناة.
- يمكن لأنواع الموافقة المختلفة أن تعرض عمدًا أسطحًا أصلية مختلفة.
  الأمثلة المضمّنة الحالية:
  - يحتفظ Slack بإتاحة توجيه الموافقة الأصلية لكل من معرّفات exec والإضافة.
  - يحتفظ Matrix بالتوجيه نفسه داخل الرسائل المباشرة/القناة وواجهة تفاعل reactions لموافقات exec
    والإضافة، مع الاستمرار في السماح لاختلاف المصادقة حسب نوع الموافقة.
- ما يزال `createApproverRestrictedNativeApprovalAdapter` موجودًا كغلاف توافق، لكن التعليمات البرمجية الجديدة يجب أن تفضّل مُنشئ القدرات وأن تعرض `approvalCapability` على الإضافة.

لنقاط دخول القنوات الساخنة، فضّل المسارات الفرعية الأضيق لوقت التشغيل عندما
تحتاج جزءًا واحدًا فقط من هذه العائلة:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

وبالمثل، فضّل `openclaw/plugin-sdk/setup-runtime`،
و`openclaw/plugin-sdk/setup-adapter-runtime`،
و`openclaw/plugin-sdk/reply-runtime`،
و`openclaw/plugin-sdk/reply-dispatch-runtime`،
و`openclaw/plugin-sdk/reply-reference`، و
`openclaw/plugin-sdk/reply-chunking` عندما لا تحتاج إلى السطح
الأوسع المظلّي.

وبالنسبة إلى الإعداد تحديدًا:

- يغطي `openclaw/plugin-sdk/setup-runtime` مساعدات الإعداد الآمنة لوقت التشغيل:
  محوّلات تصحيح الإعداد الآمنة للاستيراد (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`)، ومخرجات ملاحظات lookup،
  و`promptResolvedAllowFrom`، و`splitSetupEntries`، ومنشئات
  setup-proxy المفوّضة
- يمثّل `openclaw/plugin-sdk/setup-adapter-runtime` سطح المحوّل
  الضيق الواعي بالبيئة لـ `createEnvPatchedAccountSetupAdapter`
- يغطي `openclaw/plugin-sdk/channel-setup` منشئات الإعداد ذات التثبيت الاختياري
  بالإضافة إلى بعض العناصر الأولية الآمنة للإعداد:
  `createOptionalChannelSetupSurface` و`createOptionalChannelSetupAdapter`،

إذا كانت قناتك تدعم الإعداد أو المصادقة المعتمدين على env وكان ينبغي لتدفقات
البدء/الإعدادات العامة معرفة أسماء env هذه قبل تحميل وقت التشغيل، فصرّح بها في
manifest الإضافة باستخدام `channelEnvVars`. واحتفظ بـ `envVars` الخاصة بوقت تشغيل القناة
أو الثوابت المحلية للنصوص الموجهة للمشغّل فقط.
`createOptionalChannelSetupWizard` و`DEFAULT_ACCOUNT_ID`،
و`createTopLevelChannelDmPolicy` و`setSetupChannelEnabled` و
`splitSetupEntries`

- استخدم السطح الأوسع `openclaw/plugin-sdk/setup` فقط عندما تحتاج أيضًا إلى
  مساعدات الإعداد/الضبط المشتركة الأثقل مثل
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

إذا كانت قناتك تريد فقط الإعلان "ثبّت هذه الإضافة أولًا" في أسطح الإعداد،
ففضّل `createOptionalChannelSetupSurface(...)`. يفشل
المحوّل/المعالج المولّد بطريقة مغلقة عند كتابة الإعدادات والإنهاء، ويعيدان استخدام
رسالة "التثبيت مطلوب" نفسها عبر التحقق والإنهاء ونصوص روابط الوثائق.

لمسارات القنوات الساخنة الأخرى، فضّل المساعدات الضيقة على الأسطح
القديمة الأوسع:

- `openclaw/plugin-sdk/account-core`،
  و`openclaw/plugin-sdk/account-id`،
  و`openclaw/plugin-sdk/account-resolution`، و
  `openclaw/plugin-sdk/account-helpers` من أجل إعدادات الحسابات المتعددة
  والرجوع إلى الحساب الافتراضي
- `openclaw/plugin-sdk/inbound-envelope` و
  `openclaw/plugin-sdk/inbound-reply-dispatch` لتوجيه/غلاف الوارد
  وربط التسجيل والتوزيع
- `openclaw/plugin-sdk/messaging-targets` لتحليل/مطابقة الأهداف
- `openclaw/plugin-sdk/outbound-media` و
  `openclaw/plugin-sdk/outbound-runtime` لتحميل الوسائط بالإضافة إلى
  مفوّضات الهوية/الإرسال الصادرة
- `openclaw/plugin-sdk/thread-bindings-runtime` لدورة حياة ربط السلاسل
  وتسجيل المحوّلات
- `openclaw/plugin-sdk/agent-media-payload` فقط عندما يكون تخطيط حقل
  agent/media القديم ما يزال مطلوبًا
- `openclaw/plugin-sdk/telegram-command-config` من أجل التطبيع والتحقق من التكرارات/التعارضات
  الخاصة بأوامر Telegram المخصصة، وعقد إعداد الأوامر المستقر كبديل

يمكن للقنوات الخاصة بالمصادقة فقط عادةً أن تكتفي بالمسار الافتراضي: تتولى النواة الموافقات وتعرض الإضافة فقط قدرات الصادر/المصادقة. أما قنوات الموافقة الأصلية مثل Matrix وSlack وTelegram ووسائط الدردشة المخصصة فينبغي أن تستخدم المساعدات الأصلية المشتركة بدلًا من بناء دورة حياة الموافقة بنفسها.

## سياسة الإشارة الواردة

احرص على فصل التعامل مع الإشارات الواردة إلى طبقتين:

- جمع الأدلة المملوك للإضافة
- تقييم السياسة المشتركة

استخدم `openclaw/plugin-sdk/channel-inbound` للطبقة المشتركة.

ملائم للمنطق المحلي في الإضافة:

- اكتشاف الرد على البوت
- اكتشاف الاقتباس من البوت
- فحوص مشاركة السلسلة
- استبعاد رسائل الخدمة/النظام
- الذاكرات المؤقتة الأصلية للمنصة اللازمة لإثبات مشاركة البوت

ملائم للمساعد المشترك:

- `requireMention`
- نتيجة الإشارة الصريحة
- قائمة السماح الخاصة بالإشارة الضمنية
- تجاوز الأوامر
- قرار التخطي النهائي

التدفق المفضل:

1. احسب حقائق الإشارة المحلية.
2. مرّر هذه الحقائق إلى `resolveInboundMentionDecision({ facts, policy })`.
3. استخدم `decision.effectiveWasMentioned` و`decision.shouldBypassMention` و`decision.shouldSkip` في بوابة الوارد لديك.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";

const mentionMatch = matchesMentionWithExplicit(text, {
  mentionRegexes,
  mentionPatterns,
});

const facts = {
  canDetectMention: true,
  wasMentioned: mentionMatch.matched,
  hasAnyMention: mentionMatch.hasExplicitMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    allowedImplicitMentionKinds: requireExplicitMention ? [] : ["reply_to_bot", "quoted_bot"],
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

يكشف `api.runtime.channel.mentions` عن مساعدات الإشارة المشتركة نفسها
لإضافات القنوات المضمّنة التي تعتمد بالفعل على حقن وقت التشغيل:

- `buildMentionRegexes`
- `matchesMentionPatterns`
- `matchesMentionWithExplicit`
- `implicitMentionKindWhen`
- `resolveInboundMentionDecision`

تبقى مساعدات `resolveMentionGating*` الأقدم على
`openclaw/plugin-sdk/channel-inbound` كتصديرات توافق فقط. ويجب على
التعليمات البرمجية الجديدة استخدام `resolveInboundMentionDecision({ facts, policy })`.

## شرح عملي

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="الحزمة وmanifest">
    أنشئ ملفات الإضافة القياسية. إن حقل `channel` في `package.json` هو
    ما يجعل هذه إضافة قناة. وللاطلاع على سطح بيانات تعريف الحزمة الكامل،
    راجع [إعداد الإضافة والضبط](/ar/plugins/sdk-setup#openclawchannel):

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "Connect OpenClaw to Acme Chat."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "kind": "channel",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat channel plugin",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "acme-chat": {
            "type": "object",
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

  </Step>

  <Step title="ابنِ كائن إضافة القناة">
    تحتوي الواجهة `ChannelPlugin` على العديد من أسطح المحوّلات الاختيارية. ابدأ
    بالحد الأدنى — `id` و`setup` — وأضف المحوّلات حسب الحاجة.

    أنشئ `src/channel.ts`:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        setup: {
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    <Accordion title="ما الذي يفعله createChatChannelPlugin من أجلك">
      بدلًا من تنفيذ واجهات المحوّلات منخفضة المستوى يدويًا، تمرّر
      خيارات تصريحية ويقوم المنشئ بتركيبها:

      | الخيار | ما الذي يربطه |
      | --- | --- |
      | `security.dm` | محلّل أمان DM مقيّد من حقول الإعداد |
      | `pairing.text` | تدفق اقتران DM قائم على النص مع تبادل الرموز |
      | `threading` | محلّل وضع reply-to (ثابت، أو مقيد بالحساب، أو مخصص) |
      | `outbound.attachedResults` | دوال الإرسال التي ترجع بيانات تعريف النتيجة (معرّفات الرسائل) |

      يمكنك أيضًا تمرير كائنات محوّلات خام بدلًا من الخيارات التصريحية
      إذا كنت تحتاج إلى تحكم كامل.
    </Accordion>

  </Step>

  <Step title="صِل نقطة الدخول">
    أنشئ `index.ts`:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    ضع واصفات CLI المملوكة للقناة في `registerCliMetadata(...)` حتى يتمكن OpenClaw
    من عرضها في المساعدة الجذرية دون تفعيل وقت تشغيل القناة الكامل،
    بينما تلتقط التحميلات الكاملة العادية الواصفات نفسها لتسجيل الأوامر الفعلي.
    واحتفظ بـ `registerFull(...)` للعمل الخاص بوقت التشغيل فقط.
    إذا كان `registerFull(...)` يسجّل طرائق Gateway RPC، فاستخدم
    بادئة خاصة بالإضافة. تظل مساحات أسماء إدارة النواة (`config.*`،
    و`exec.approvals.*`، و`wizard.*`، و`update.*`) محجوزة وتُحل دائمًا إلى
    `operator.admin`.
    يعالج `defineChannelPluginEntry` تقسيم وضع التسجيل تلقائيًا. راجع
    [نقاط الدخول](/ar/plugins/sdk-entrypoints#definechannelpluginentry) للاطلاع على جميع
    الخيارات.

  </Step>

  <Step title="أضف نقطة دخول للإعداد">
    أنشئ `setup-entry.ts` للتحميل الخفيف أثناء الإعداد الأولي:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    يحمّل OpenClaw هذا بدلًا من نقطة الدخول الكاملة عندما تكون القناة معطلة
    أو غير مضبوطة. وهذا يتجنب سحب تعليمات وقت تشغيل ثقيلة أثناء تدفقات الإعداد.
    راجع [الإعداد والضبط](/ar/plugins/sdk-setup#setup-entry) للتفاصيل.

  </Step>

  <Step title="تعامل مع الرسائل الواردة">
    تحتاج إضافتك إلى استقبال الرسائل من المنصة وتمريرها إلى
    OpenClaw. النمط المعتاد هو webhook يتحقق من الطلب
    ويوزعه عبر معالج الوارد الخاص بقناتك:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // plugin-managed auth (verify signatures yourself)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Your inbound handler dispatches the message to OpenClaw.
          // The exact wiring depends on your platform SDK —
          // see a real example in the bundled Microsoft Teams or Google Chat plugin package.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      التعامل مع الرسائل الواردة خاص بالقناة. كل إضافة قناة تملك
      مسار الوارد الخاص بها. انظر إلى إضافات القنوات المضمّنة
      (مثل حزمة إضافة Microsoft Teams أو Google Chat) للاطلاع على أنماط حقيقية.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="الاختبار">
اكتب اختبارات متجاورة في `src/channel.test.ts`:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("resolves account from config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.setup!.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspects account without materializing secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("reports missing config", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.setup!.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test -- <bundled-plugin-root>/acme-chat/
    ```

    للاطلاع على مساعدات الاختبار المشتركة، راجع [الاختبار](/ar/plugins/sdk-testing).

  </Step>
</Steps>

## بنية الملفات

```
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel metadata
├── openclaw.plugin.json      # Manifest with config schema
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Public exports (optional)
├── runtime-api.ts            # Internal runtime exports (optional)
└── src/
    ├── channel.ts            # ChannelPlugin via createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # Platform API client
    └── runtime.ts            # Runtime store (if needed)
```

## موضوعات متقدمة

<CardGroup cols={2}>
  <Card title="خيارات التسلسل" icon="git-branch" href="/ar/plugins/sdk-entrypoints#registration-mode">
    أوضاع reply ثابتة أو مقيّدة بالحساب أو مخصصة
  </Card>
  <Card title="تكامل أداة الرسائل" icon="puzzle" href="/ar/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool واكتشاف الإجراءات
  </Card>
  <Card title="حلّ الهدف" icon="crosshair" href="/ar/plugins/architecture#channel-target-resolution">
    inferTargetChatType وlooksLikeId وresolveTarget
  </Card>
  <Card title="مساعدات وقت التشغيل" icon="settings" href="/ar/plugins/sdk-runtime">
    TTS وSTT والوسائط والوكيل الفرعي عبر api.runtime
  </Card>
</CardGroup>

<Note>
لا تزال بعض أسطح المساعدات المضمّنة موجودة لصيانة الإضافات المضمّنة
وللتوافق. وهي ليست النمط الموصى به لإضافات القنوات الجديدة؛
فضّل المسارات الفرعية العامة للقناة/الإعداد/الرد/وقت التشغيل من سطح SDK
المشترك ما لم تكن تصون عائلة تلك الإضافات المضمّنة مباشرة.
</Note>

## الخطوات التالية

- [إضافات المزوّد](/ar/plugins/sdk-provider-plugins) — إذا كانت إضافتك توفّر أيضًا نماذج
- [نظرة عامة على SDK](/ar/plugins/sdk-overview) — مرجع كامل لواردات المسارات الفرعية
- [اختبار SDK](/ar/plugins/sdk-testing) — أدوات الاختبار واختبارات العقود
- [Manifest الإضافة](/ar/plugins/manifest) — مخطط manifest الكامل
