---
read_when:
    - أنت تبني إضافة قناة مراسلة جديدة
    - أنت تريد ربط OpenClaw بمنصة مراسلة
    - أنت بحاجة إلى فهم سطح محول ChannelPlugin
sidebarTitle: Channel Plugins
summary: دليل خطوة بخطوة لبناء إضافة قناة مراسلة لـ OpenClaw
title: بناء إضافات القنوات
x-i18n:
    generated_at: "2026-04-06T03:10:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 66b52c10945a8243d803af3bf7e1ea0051869ee92eda2af5718d9bb24fbb8552
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# بناء إضافات القنوات

يرشدك هذا الدليل خلال بناء إضافة قناة تربط OpenClaw بمنصة
مراسلة. وبنهاية الدليل سيكون لديك قناة عاملة تتضمن أمان الرسائل المباشرة،
والإقران، وخيطنة الردود، والمراسلة الصادرة.

<Info>
  إذا لم تكن قد أنشأت أي إضافة لـ OpenClaw من قبل، فاقرأ
  [البدء](/ar/plugins/building-plugins) أولًا للتعرف على بنية الحزمة
  الأساسية وإعداد البيان.
</Info>

## كيف تعمل إضافات القنوات

لا تحتاج إضافات القنوات إلى أدوات send/edit/react خاصة بها. يحتفظ OpenClaw
بأداة `message` واحدة مشتركة في اللب. وتمتلك إضافتك ما يلي:

- **الإعدادات** — حل الحسابات ومعالج الإعداد
- **الأمان** — سياسة الرسائل المباشرة وقوائم السماح
- **الإقران** — تدفق الموافقة على الرسائل المباشرة
- **صياغة الجلسة** — كيفية ربط معرّفات المحادثات الخاصة بالموفر بالمحادثات الأساسية ومعرّفات الخيوط والبدائل الأصلية
- **الصادر** — إرسال النص والوسائط والاستطلاعات إلى المنصة
- **الخيطنة** — كيفية تنظيم الردود ضمن خيوط

يمتلك اللب أداة الرسائل المشتركة، وربط الطلبات، وشكل مفتاح الجلسة الخارجي،
وآليات `:thread:` العامة، والتوجيه.

إذا كانت منصتك تخزن نطاقًا إضافيًا داخل معرّفات المحادثات، فاحتفظ بهذا التحليل
داخل الإضافة باستخدام `messaging.resolveSessionConversation(...)`. هذه هي الخطافة
القياسية لربط `rawId` بمعرّف المحادثة الأساسي، ومعرّف الخيط الاختياري،
و`baseConversationId` الصريح، وأي `parentConversationCandidates`.
عندما تعيد `parentConversationCandidates`، احرص على ترتيبها من
الأصل الأضيق إلى المحادثة الأساسية/الأوسع.

يمكن للإضافات المضمّنة التي تحتاج إلى التحليل نفسه قبل إقلاع سجل القنوات
أيضًا أن تعرض ملفًا علويًا باسم `session-key-api.ts` مع تصدير مطابق
`resolveSessionConversation(...)`. يستخدم اللب هذا السطح الآمن للإقلاع
فقط عندما لا يكون سجل إضافات وقت التشغيل متاحًا بعد.

لا تزال `messaging.resolveParentConversationCandidates(...)` متاحة كبديل
توافقي قديم عندما تحتاج الإضافة فقط إلى بدائل أصلية فوق المعرّف
العام/الخام. وإذا وُجدت كلتا الخطافتين، يستخدم اللب
`resolveSessionConversation(...).parentConversationCandidates` أولًا، ثم
يعود فقط إلى `resolveParentConversationCandidates(...)` عندما لا تتضمن الخطافة
القياسية هذه القيم.

## الموافقات وقدرات القناة

معظم إضافات القنوات لا تحتاج إلى أي شيفرة خاصة بالموافقة.

- يمتلك اللب أمر `/approve` داخل المحادثة نفسها، وحمولات أزرار الموافقة المشتركة، والتسليم الاحتياطي العام.
- فضّل استخدام كائن `approvalCapability` واحد على إضافة القناة عندما تحتاج القناة إلى سلوك خاص بالموافقة.
- يشكل `approvalCapability.authorizeActorAction` و `approvalCapability.getActionAvailabilityState` سطح المصادقة القياسي للموافقة.
- إذا كانت قناتك تعرض موافقات تنفيذ أصلية، فنفّذ `approvalCapability.getActionAvailabilityState` حتى عندما تكون وسيلة النقل الأصلية موجودة بالكامل تحت `approvalCapability.native`. يستخدم اللب خطافة التوفر هذه للتمييز بين `enabled` و `disabled`، وتحديد ما إذا كانت القناة البادئة تدعم الموافقات الأصلية، وإدراج القناة في إرشادات الرجوع إلى العميل الأصلي.
- استخدم `outbound.shouldSuppressLocalPayloadPrompt` أو `outbound.beforeDeliverPayload` لسلوك دورة حياة الحمولة الخاص بالقناة، مثل إخفاء مطالبات الموافقة المحلية المكررة أو إرسال مؤشرات الكتابة قبل التسليم.
- استخدم `approvalCapability.delivery` فقط لتوجيه الموافقة الأصلية أو منع التسليم الاحتياطي.
- استخدم `approvalCapability.render` فقط عندما تحتاج القناة فعلًا إلى حمولات موافقة مخصصة بدلًا من المصيّر المشترك.
- استخدم `approvalCapability.describeExecApprovalSetup` عندما تريد القناة أن يشرح الرد في مسار التعطيل مفاتيح الإعداد الدقيقة المطلوبة لتفعيل موافقات التنفيذ الأصلية. تستقبل الخطافة `{ channel, channelLabel, accountId }`؛ ويجب على القنوات ذات الحسابات المسماة عرض مسارات مرتبطة بالحساب مثل `channels.<channel>.accounts.<id>.execApprovals.*` بدلًا من الإعدادات الافتراضية ذات المستوى الأعلى.
- إذا كانت القناة تستطيع استنتاج هويات مستقرة شبيهة بالمالك في الرسائل المباشرة من الإعدادات الحالية، فاستخدم `createResolvedApproverActionAuthAdapter` من `openclaw/plugin-sdk/approval-runtime` لتقييد `/approve` داخل المحادثة نفسها من دون إضافة منطق خاص بالموافقة إلى اللب.
- إذا كانت القناة تحتاج إلى تسليم موافقة أصلية، فأبقِ شيفرة القناة مركزة على تطبيع الهدف وخطافات النقل. استخدم `createChannelExecApprovalProfile` و `createChannelNativeOriginTargetResolver` و `createChannelApproverDmTargetResolver` و `createApproverRestrictedNativeApprovalCapability` و `createChannelNativeApprovalRuntime` من `openclaw/plugin-sdk/approval-runtime` بحيث يمتلك اللب تصفية الطلبات والتوجيه وإزالة التكرار وانتهاء الصلاحية واشتراك البوابة.
- يجب على قنوات الموافقة الأصلية تمرير كلٍّ من `accountId` و `approvalKind` عبر هذه المساعدات. يحافظ `accountId` على حصر سياسة الموافقة متعددة الحسابات ضمن حساب البوت الصحيح، بينما يحافظ `approvalKind` على إتاحة سلوك موافقة التنفيذ مقابل موافقة الإضافة للقناة من دون فروع ثابتة في اللب.
- حافظ على نوع معرّف الموافقة المُسلَّم من البداية إلى النهاية. يجب ألا
  تخمّن العملاء الأصلية أو تعيد كتابة توجيه موافقة التنفيذ مقابل موافقة الإضافة من حالة محلية داخل القناة.
- يمكن لأنواع الموافقة المختلفة أن تعرض عمدًا أسطحًا أصلية مختلفة.
  الأمثلة المضمّنة الحالية:
  - يحتفظ Slack بإتاحة توجيه الموافقة الأصلية لكلٍّ من معرّفات التنفيذ والإضافة.
  - يحتفظ Matrix بتوجيه الرسائل المباشرة/القنوات الأصلي لموافقات التنفيذ فقط ويترك
    موافقات الإضافات على مسار `/approve` المشترك داخل المحادثة نفسها.
- لا يزال `createApproverRestrictedNativeApprovalAdapter` موجودًا كغلاف توافقي، لكن يجب أن تفضّل الشيفرة الجديدة منشئ القدرات وتعرض `approvalCapability` على الإضافة.

بالنسبة إلى نقاط دخول القنوات الساخنة، فضّل المسارات الفرعية الأضيق في وقت التشغيل عندما
تحتاج إلى جزء واحد فقط من هذه العائلة:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`

وبالمثل، فضّل `openclaw/plugin-sdk/setup-runtime`،
و`openclaw/plugin-sdk/setup-adapter-runtime`،
و`openclaw/plugin-sdk/reply-runtime`،
و`openclaw/plugin-sdk/reply-dispatch-runtime`،
و`openclaw/plugin-sdk/reply-reference`، و
`openclaw/plugin-sdk/reply-chunking` عندما لا تحتاج إلى
السطح الأوسع الشامل.

وبالنسبة إلى الإعداد تحديدًا:

- يغطي `openclaw/plugin-sdk/setup-runtime` مساعدات الإعداد الآمنة لوقت التشغيل:
  محولات تصحيح الإعداد الآمنة للاستيراد (`createPatchedAccountSetupAdapter`,
  و`createEnvPatchedAccountSetupAdapter`,
  و`createSetupInputPresenceValidator`)، ومخرجات ملاحظات البحث،
  و`promptResolvedAllowFrom`، و`splitSetupEntries`، ومنشئات
  الوكيل المُفوَّض للإعداد
- يشكل `openclaw/plugin-sdk/setup-adapter-runtime` سطح المحول الضيق المدرك للبيئة
  الخاص بـ `createEnvPatchedAccountSetupAdapter`
- يغطي `openclaw/plugin-sdk/channel-setup` منشئات الإعداد ذات التثبيت الاختياري
  بالإضافة إلى عدد قليل من البدائيات الآمنة للإعداد:
  `createOptionalChannelSetupSurface`، و`createOptionalChannelSetupAdapter`،
  و`createOptionalChannelSetupWizard`، و`DEFAULT_ACCOUNT_ID`،
  و`createTopLevelChannelDmPolicy`، و`setSetupChannelEnabled`، و
  `splitSetupEntries`
- استخدم سطح `openclaw/plugin-sdk/setup` الأوسع فقط عندما تحتاج أيضًا إلى
  مساعدات الإعداد/التهيئة المشتركة الأثقل مثل
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

إذا كانت قناتك تريد فقط الإعلان عن "ثبّت هذه الإضافة أولًا" في أسطح الإعداد،
ففضّل `createOptionalChannelSetupSurface(...)`. إذ تفشل
المحولات/المعالجات المولدة بشكل مغلق عند كتابة الإعدادات وإنهائها، كما تعيد استخدام
رسالة التثبيت المطلوبة نفسها في التحقق والإنهاء ونسخة رابط المستندات.

وبالنسبة إلى مسارات القنوات الساخنة الأخرى، فضّل المساعدات الضيقة على الأسطح
القديمة الأوسع:

- `openclaw/plugin-sdk/account-core`،
  و`openclaw/plugin-sdk/account-id`،
  و`openclaw/plugin-sdk/account-resolution`، و
  `openclaw/plugin-sdk/account-helpers` لإعدادات تعدد الحسابات
  والرجوع إلى الحساب الافتراضي
- `openclaw/plugin-sdk/inbound-envelope` و
  `openclaw/plugin-sdk/inbound-reply-dispatch` لربط
  المسار/الغلاف الوارد والتسجيل ثم التوجيه
- `openclaw/plugin-sdk/messaging-targets` لتحليل/مطابقة الأهداف
- `openclaw/plugin-sdk/outbound-media` و
  `openclaw/plugin-sdk/outbound-runtime` لتحميل الوسائط بالإضافة إلى
  مفوضات الهوية/الإرسال الصادرة
- `openclaw/plugin-sdk/thread-bindings-runtime` لدورة حياة ربط الخيوط
  وتسجيل المحولات
- `openclaw/plugin-sdk/agent-media-payload` فقط عندما تظل بنية
  حقول حمولة الوكيل/الوسائط القديمة مطلوبة
- `openclaw/plugin-sdk/telegram-command-config` لتطبيع الأوامر المخصصة في Telegram،
  والتحقق من التكرار/التعارض، وعقد إعداد الأوامر الثابت عند الرجوع

يمكن للقنوات المعتمدة على المصادقة فقط عادةً الاكتفاء بالمسار الافتراضي: يتولى اللب الموافقات بينما تعرض الإضافة فقط قدرات الصادر/المصادقة. أما قنوات الموافقة الأصلية مثل Matrix وSlack وTelegram ووسائل نقل الدردشة المخصصة، فيجب أن تستخدم المساعدات الأصلية المشتركة بدلًا من تنفيذ دورة حياة الموافقة بنفسها.

## الشرح العملي

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="الحزمة والبيان">
    أنشئ ملفات الإضافة القياسية. الحقل `channel` في `package.json` هو
    ما يجعل هذه الإضافة إضافة قناة. للحصول على السطح الكامل لبيانات تعريف الحزمة،
    راجع [إعداد الإضافة والتهيئة](/ar/plugins/sdk-setup#openclawchannel):

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
    تحتوي الواجهة `ChannelPlugin` على العديد من أسطح المحولات الاختيارية. ابدأ
    بالحد الأدنى — `id` و `setup` — ثم أضف المحولات حسب الحاجة.

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

    <Accordion title="ما الذي تفعله `createChatChannelPlugin` نيابة عنك">
      بدلًا من تنفيذ واجهات المحولات منخفضة المستوى يدويًا، فإنك تمرر
      خيارات تصريحية ويقوم المنشئ بتركيبها:

      | الخيار | ما الذي يربطه |
      | --- | --- |
      | `security.dm` | محلّل أمان DM محصور مشتق من حقول الإعداد |
      | `pairing.text` | تدفق إقران DM نصي مع تبادل الرمز |
      | `threading` | محلّل وضع الرد (`reply-to`) ثابتًا أو مقيّدًا بالحساب أو مخصصًا |
      | `outbound.attachedResults` | دوال إرسال تعيد بيانات تعريف النتيجة (معرّفات الرسائل) |

      يمكنك أيضًا تمرير كائنات محولات خام بدلًا من الخيارات التصريحية
      إذا كنت بحاجة إلى تحكم كامل.
    </Accordion>

  </Step>

  <Step title="اربط نقطة الدخول">
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
    من عرضها في المساعدة الجذرية من دون تفعيل وقت تشغيل القناة الكامل،
    بينما تلتقط التحميلات الكاملة العادية الواصفات نفسها من أجل تسجيل الأوامر
    الفعلي. واحتفظ بـ `registerFull(...)` للأعمال الخاصة بوقت التشغيل فقط.
    إذا كان `registerFull(...)` يسجل أساليب RPC للبوابة، فاستخدم
    بادئة خاصة بالإضافة. تظل مساحات أسماء الإدارة الأساسية (`config.*`،
    و`exec.approvals.*`، و`wizard.*`، و`update.*`) محجوزة وتُحل دائمًا
    إلى `operator.admin`.
    يتولى `defineChannelPluginEntry` تقسيم وضع التسجيل تلقائيًا. راجع
    [نقاط الدخول](/ar/plugins/sdk-entrypoints#definechannelpluginentry) للاطلاع على
    جميع الخيارات.

  </Step>

  <Step title="أضف إدخال إعداد">
    أنشئ `setup-entry.ts` للتحميل الخفيف أثناء الإعداد الأولي:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    يحمّل OpenClaw هذا بدلًا من الإدخال الكامل عندما تكون القناة معطلة
    أو غير مهيأة. وهذا يمنع سحب شيفرة وقت تشغيل ثقيلة أثناء تدفقات الإعداد.
    راجع [الإعداد والتهيئة](/ar/plugins/sdk-setup#setup-entry) للحصول على التفاصيل.

  </Step>

  <Step title="تعامل مع الرسائل الواردة">
    تحتاج إضافتك إلى تلقي الرسائل من المنصة وتمريرها إلى
    OpenClaw. النمط المعتاد هو webhook يتحقق من الطلب ثم
    يوجّهه عبر معالج الوارد الخاص بقناتك:

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
      إن معالجة الرسائل الواردة خاصة بكل قناة. تمتلك كل إضافة قناة
      خط معالجة الوارد الخاص بها. انظر إلى إضافات القنوات المضمّنة
      (مثل حزمة إضافة Microsoft Teams أو Google Chat) للاطلاع على أنماط واقعية.
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

    للحصول على مساعدات الاختبار المشتركة، راجع [الاختبار](/ar/plugins/sdk-testing).

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
  <Card title="خيارات الخيطنة" icon="git-branch" href="/ar/plugins/sdk-entrypoints#registration-mode">
    أوضاع رد ثابتة أو مقيّدة بالحساب أو مخصصة
  </Card>
  <Card title="تكامل أداة الرسائل" icon="puzzle" href="/ar/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool واكتشاف الإجراءات
  </Card>
  <Card title="حل الأهداف" icon="crosshair" href="/ar/plugins/architecture#channel-target-resolution">
    inferTargetChatType و looksLikeId و resolveTarget
  </Card>
  <Card title="مساعدات وقت التشغيل" icon="settings" href="/ar/plugins/sdk-runtime">
    TTS و STT والوسائط والوكيل الفرعي عبر api.runtime
  </Card>
</CardGroup>

<Note>
لا تزال بعض الأسطح المساعدة المضمّنة موجودة من أجل صيانة الإضافات المضمّنة
والتوافق. لكنها ليست النمط الموصى به لإضافات القنوات الجديدة؛
فضّل المسارات الفرعية العامة للقناة/الإعداد/الرد/وقت التشغيل من سطح SDK
المشترك، ما لم تكن تصون عائلة الإضافات المضمّنة تلك مباشرة.
</Note>

## الخطوات التالية

- [إضافات الموفر](/ar/plugins/sdk-provider-plugins) — إذا كانت إضافتك توفّر نماذج أيضًا
- [نظرة عامة على SDK](/ar/plugins/sdk-overview) — مرجع كامل لاستيراد المسارات الفرعية
- [اختبار SDK](/ar/plugins/sdk-testing) — أدوات الاختبار واختبارات العقود
- [بيان الإضافة](/ar/plugins/manifest) — المخطط الكامل للبيان
