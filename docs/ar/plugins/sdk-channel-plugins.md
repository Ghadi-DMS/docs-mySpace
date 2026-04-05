---
read_when:
    - أنت تبني plugin قناة مراسلة جديدة
    - أنت تريد توصيل OpenClaw بمنصة مراسلة
    - أنت بحاجة إلى فهم سطح محول ChannelPlugin
sidebarTitle: Channel Plugins
summary: دليل خطوة بخطوة لبناء plugin قناة مراسلة لـ OpenClaw
title: بناء Plugins القنوات
x-i18n:
    generated_at: "2026-04-05T12:51:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 68a6ad2c75549db8ce54f7e22ca9850d7ed68c5cd651c9bb41c9f73769f48aba
    source_path: plugins/sdk-channel-plugins.md
    workflow: 15
---

# بناء Plugins القنوات

يشرح هذا الدليل كيفية بناء plugin قناة يربط OpenClaw بمنصة
مراسلة. وفي النهاية سيكون لديك قناة عاملة تتضمن أمان الرسائل المباشرة،
والاقتران، وسلاسل الردود، والمراسلة الصادرة.

<Info>
  إذا لم تكن قد بنيت أي plugin لـ OpenClaw من قبل، فاقرأ
  [البدء](/plugins/building-plugins) أولًا للتعرف على بنية الحزمة
  الأساسية وإعداد manifest.
</Info>

## كيف تعمل Plugins القنوات

لا تحتاج Plugins القنوات إلى أدوات send/edit/react خاصة بها. يحتفظ OpenClaw
بأداة `message` مشتركة واحدة في النواة. أما plugin الخاص بك فيمتلك:

- **الإعداد** — تحليل الحسابات ومعالج الإعداد
- **الأمان** — سياسة الرسائل المباشرة وقوائم السماح
- **الاقتران** — تدفق الموافقة على الرسائل المباشرة
- **قواعد الجلسة** — كيفية ربط معرّفات المحادثة الخاصة بالموفّر بالدردشات الأساسية ومعرّفات السلاسل وبدائل الأصل
- **الصادر** — إرسال النص والوسائط والاستطلاعات إلى المنصة
- **السلاسل** — كيفية تنظيم الردود ضمن سلاسل

تمتلك النواة أداة الرسائل المشتركة، وربط prompt، والشكل الخارجي لمفتاح الجلسة،
والإدارة العامة لـ `:thread:`، والتوزيع.

إذا كانت منصتك تخزن نطاقًا إضافيًا داخل معرّفات المحادثات، فاحتفظ بهذا التحليل
داخل plugin باستخدام `messaging.resolveSessionConversation(...)`. فهذه هي
الوصلة القياسية لربط `rawId` بمعرّف المحادثة الأساسي، ومعرّف سلسلة اختياري،
وقيمة `baseConversationId` صريحة، وأي `parentConversationCandidates`.
وعندما تعيد `parentConversationCandidates`، فاحرص على ترتيبها من
الأصل الأضيق إلى الأصل الأوسع/المحادثة الأساسية.

يمكن أيضًا لـ plugins المضمّنة التي تحتاج إلى التحليل نفسه قبل تشغيل سجل القنوات
أن تكشف ملفًا علويًا `session-key-api.ts` مع دالة
`resolveSessionConversation(...)` مطابقة في التصدير. وتستخدم النواة هذا السطح
الآمن في bootstrap فقط عندما لا يكون سجل plugins في وقت التشغيل متاحًا بعد.

لا تزال `messaging.resolveParentConversationCandidates(...)` متاحة كحل رجوع
قديم من أجل التوافق عندما يحتاج plugin فقط إلى بدائل الأصل فوق
المعرّف العام/الخام. وإذا وُجدت الوصلتان معًا، تستخدم النواة
`resolveSessionConversation(...).parentConversationCandidates` أولًا ولا تعود
إلى `resolveParentConversationCandidates(...)` إلا عندما تهملها الوصلة القياسية.

## الموافقات وإمكانات القنوات

لا تحتاج معظم Plugins القنوات إلى شيفرة خاصة بالموافقات.

- تمتلك النواة الأمر `/approve` داخل الدردشة نفسها، وحمولات أزرار الموافقة المشتركة، وآليات التسليم العامة الاحتياطية.
- فضّل كائن `approvalCapability` واحدًا على plugin القناة عندما تحتاج القناة إلى سلوك خاص بالموافقة.
- تُعد `approvalCapability.authorizeActorAction` و`approvalCapability.getActionAvailabilityState` الوصلة القياسية لمصادقة الموافقة.
- استخدم `outbound.shouldSuppressLocalPayloadPrompt` أو `outbound.beforeDeliverPayload` لسلوك دورة حياة الحمولة الخاص بالقناة، مثل إخفاء مطالبات الموافقة المحلية المكررة أو إرسال مؤشرات الكتابة قبل التسليم.
- استخدم `approvalCapability.delivery` فقط للتوجيه الأصلي للموافقات أو منع التسليم الاحتياطي.
- استخدم `approvalCapability.render` فقط عندما تحتاج القناة فعلًا إلى حمولات موافقة مخصصة بدلًا من العارض المشترك.
- إذا كانت القناة تستطيع استنتاج هويات شبيهة بالمالك في الرسائل المباشرة من الإعداد الحالي، فاستخدم `createResolvedApproverActionAuthAdapter` من `openclaw/plugin-sdk/approval-runtime` لتقييد `/approve` داخل الدردشة نفسها من دون إضافة منطق خاص بالموافقة إلى النواة.
- إذا احتاجت القناة إلى تسليم موافقة أصلي، فأبقِ شيفرة القناة مركزة على تسوية الهدف ووصلات النقل. واستخدم `createChannelExecApprovalProfile` و`createChannelNativeOriginTargetResolver` و`createChannelApproverDmTargetResolver` و`createApproverRestrictedNativeApprovalCapability` و`createChannelNativeApprovalRuntime` من `openclaw/plugin-sdk/approval-runtime` حتى تمتلك النواة التصفية والتوجيه وإزالة التكرار وانتهاء الصلاحية والاشتراك في البوابة.
- يجب على القنوات ذات الموافقات الأصلية تمرير كل من `accountId` و`approvalKind` عبر هذه المساعدات. فـ `accountId` يحافظ على نطاق سياسة الموافقات متعددة الحسابات على حساب البوت الصحيح، و`approvalKind` يبقي سلوك موافقات exec مقابل plugin متاحًا للقناة من دون فروع ثابتة داخل النواة.
- حافظ على نوع معرّف الموافقة المُسلَّم من البداية إلى النهاية. يجب ألا
  يقوم العملاء الأصليون بتخمين أو إعادة كتابة توجيه الموافقات exec مقابل plugin انطلاقًا من حالة محلية خاصة بالقناة.
- يمكن عمدًا لأنواع الموافقات المختلفة أن تكشف أسطحًا أصلية مختلفة.
  ومن الأمثلة المضمّنة الحالية:
  - يحتفظ Slack بالتوجيه الأصلي للموافقات متاحًا لكل من معرّفات exec وplugin.
  - يحتفظ Matrix بالتوجيه الأصلي في الرسائل المباشرة/القنوات لموافقات exec فقط، ويترك
    موافقات plugin على مسار `/approve` المشترك داخل الدردشة نفسها.
- لا يزال `createApproverRestrictedNativeApprovalAdapter` موجودًا كغلاف للتوافق، لكن الشيفرة الجديدة يجب أن تفضّل باني الإمكانات وتكشف `approvalCapability` على plugin.

بالنسبة إلى نقاط دخول القنوات الساخنة، فضّل المسارات الفرعية الأضيق في وقت التشغيل عندما
تحتاج فقط إلى جزء واحد من هذه العائلة:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`

وبالمثل، فضّل `openclaw/plugin-sdk/setup-runtime`,
و`openclaw/plugin-sdk/setup-adapter-runtime`,
و`openclaw/plugin-sdk/reply-runtime`,
و`openclaw/plugin-sdk/reply-dispatch-runtime`,
و`openclaw/plugin-sdk/reply-reference`، و
`openclaw/plugin-sdk/reply-chunking` عندما لا تحتاج إلى
السطح الأشمل.

وبالنسبة إلى الإعداد تحديدًا:

- يغطي `openclaw/plugin-sdk/setup-runtime` مساعدات الإعداد الآمنة وقت التشغيل:
  محولات patch الآمنة للاستيراد (`createPatchedAccountSetupAdapter`,
  `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`)، ومخرجات ملاحظات البحث،
  و`promptResolvedAllowFrom`، و`splitSetupEntries`، وبناة
  setup-proxy المفوضة
- يمثل `openclaw/plugin-sdk/setup-adapter-runtime` الوصلة الضيقة
  الواعية بالبيئة من أجل `createEnvPatchedAccountSetupAdapter`
- يغطي `openclaw/plugin-sdk/channel-setup` بناة الإعداد الخاصة بالتثبيت الاختياري
  بالإضافة إلى عدد قليل من العناصر الأولية الآمنة للإعداد:
  `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`,
  `createOptionalChannelSetupWizard`, `DEFAULT_ACCOUNT_ID`,
  `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, و
  `splitSetupEntries`
- استخدم الوصلة الأشمل `openclaw/plugin-sdk/setup` فقط عندما تحتاج أيضًا إلى
  مساعدات إعداد/تهيئة مشتركة أثقل مثل
  `moveSingleAccountChannelSectionToDefaultAccount(...)`

إذا كانت قناتك تريد فقط الإعلان عن "ثبّت هذا plugin أولًا" في أسطح الإعداد،
ففضّل `createOptionalChannelSetupSurface(...)`. يفشل المحول/المعالج المولّد
بشكل مغلق على عمليات كتابة الإعداد والإنهاء النهائي، ويعيد استخدام رسالة
اشتراط التثبيت نفسها عبر التحقق والإنهاء ونسخة رابط الوثائق.

وبالنسبة إلى المسارات الأخرى الساخنة في القنوات، فضّل المساعدات الضيقة بدلًا من
الأسطح القديمة الأوسع:

- `openclaw/plugin-sdk/account-core`,
  `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution`, و
  `openclaw/plugin-sdk/account-helpers` من أجل إعدادات الحسابات المتعددة
  والرجوع إلى الحساب الافتراضي
- `openclaw/plugin-sdk/inbound-envelope` و
  `openclaw/plugin-sdk/inbound-reply-dispatch` من أجل ربط
  التوجيه/الغلاف الوارد وربط record-and-dispatch
- `openclaw/plugin-sdk/messaging-targets` لتحليل/مطابقة الأهداف
- `openclaw/plugin-sdk/outbound-media` و
  `openclaw/plugin-sdk/outbound-runtime` من أجل تحميل الوسائط بالإضافة إلى
  مفوضات الهوية/الإرسال الصادرة
- `openclaw/plugin-sdk/thread-bindings-runtime` من أجل دورة حياة
  thread-binding وتسجيل المحولات
- `openclaw/plugin-sdk/agent-media-payload` فقط عندما يكون تخطيط
  حقل agent/media payload قديمًا لا يزال مطلوبًا
- `openclaw/plugin-sdk/telegram-command-config` من أجل تسوية
  الأوامر المخصصة في Telegram، والتحقق من التكرار/التعارض، وعقد إعداد الأوامر
  الثابت في الرجوع

يمكن للقنوات الخاصة بالمصادقة فقط أن تتوقف عادةً عند المسار الافتراضي: فالنواة تتولى الموافقات ويكشف plugin فقط إمكانات الصادر/المصادقة. أما القنوات ذات الموافقات الأصلية مثل Matrix وSlack وTelegram ووسائط الدردشة المخصصة فيجب أن تستخدم المساعدات الأصلية المشتركة بدلًا من بناء دورة حياة الموافقات بنفسها.

## الشرح العملي

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="الحزمة وmanifest">
    أنشئ ملفات plugin القياسية. إن الحقل `channel` في `package.json` هو
    ما يجعل هذا plugin قناة. وللحصول على سطح بيانات الحزمة الكامل،
    راجع [إعداد Plugin وConfig](/plugins/sdk-setup#openclawchannel):

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

  <Step title="ابنِ كائن plugin الخاص بالقناة">
    يحتوي واجهة `ChannelPlugin` على الكثير من أسطح المحولات الاختيارية. ابدأ
    بالحد الأدنى — `id` و`setup` — ثم أضف المحولات حسب الحاجة.

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

    <Accordion title="ما الذي تفعله createChatChannelPlugin نيابةً عنك">
      بدلًا من تنفيذ واجهات المحولات منخفضة المستوى يدويًا، فإنك تمرر
      خيارات وصفية ويتولى الباني تركيبها:

      | الخيار | ما الذي يربطه |
      | --- | --- |
      | `security.dm` | محلل أمان رسائل مباشرة مقيّد النطاق من حقول الإعداد |
      | `pairing.text` | تدفق اقتران رسائل مباشرة قائم على النص مع تبادل الرموز |
      | `threading` | محلل وضع reply-to ‏(ثابت أو مقيّد بالحساب أو مخصص) |
      | `outbound.attachedResults` | دوال إرسال تعيد بيانات وصفية للنتيجة (معرّفات الرسائل) |

      يمكنك أيضًا تمرير كائنات محولات خام بدلًا من الخيارات الوصفية
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

    ضع واصفات CLI الخاصة بالقناة في `registerCliMetadata(...)` حتى يتمكن OpenClaw
    من عرضها في مساعدة الجذر من دون تفعيل وقت تشغيل القناة الكامل،
    بينما تظل التحميلات الكاملة العادية تلتقط الواصفات نفسها لتسجيل الأوامر الفعلي.
    وأبقِ `registerFull(...)` للأعمال الخاصة بوقت التشغيل فقط.
    وإذا كان `registerFull(...)` يسجل طرق Gateway RPC، فاستخدم
    بادئة خاصة بالـ plugin. تبقى مساحات أسماء الإدارة في النواة (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) محجوزة وتتحلل دائمًا إلى
    `operator.admin`.
    يتولى `defineChannelPluginEntry` فصل أوضاع التسجيل تلقائيًا. راجع
    [نقاط الدخول](/plugins/sdk-entrypoints#definechannelpluginentry) لجميع
    الخيارات.

  </Step>

  <Step title="أضف setup entry">
    أنشئ `setup-entry.ts` من أجل تحميل خفيف أثناء الإعداد الأولي:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    يحمّل OpenClaw هذا بدلًا من نقطة الدخول الكاملة عندما تكون القناة معطلة
    أو غير مهيأة. وهذا يتجنب سحب شيفرة وقت تشغيل ثقيلة أثناء تدفقات الإعداد.
    راجع [الإعداد وConfig](/plugins/sdk-setup#setup-entry) للتفاصيل.

  </Step>

  <Step title="تعامل مع الرسائل الواردة">
    يحتاج plugin الخاص بك إلى استلام الرسائل من المنصة وإعادة توجيهها إلى
    OpenClaw. النمط المعتاد هو webhook يتحقق من الطلب ثم
    يوزعه عبر معالج الوارد الخاص بقناتك:

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
      التعامل مع الرسائل الواردة خاص بكل قناة. فكل plugin قناة يمتلك
      خط معالجة الوارد الخاص به. انظر إلى Plugins القنوات المضمّنة
      (مثل حزمة plugin الخاصة بـ Microsoft Teams أو Google Chat) للحصول على أنماط حقيقية.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="اختبر">
اكتب اختبارات موضوعة بجانب الشيفرة في `src/channel.test.ts`:

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

    وبالنسبة إلى مساعدات الاختبار المشتركة، راجع [الاختبار](/plugins/sdk-testing).

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
  <Card title="خيارات السلاسل" icon="git-branch" href="/plugins/sdk-entrypoints#registration-mode">
    أوضاع reply ثابتة أو مقيّدة بالحساب أو مخصصة
  </Card>
  <Card title="تكامل أداة الرسائل" icon="puzzle" href="/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool واكتشاف الإجراءات
  </Card>
  <Card title="تحليل الهدف" icon="crosshair" href="/plugins/architecture#channel-target-resolution">
    inferTargetChatType, looksLikeId, resolveTarget
  </Card>
  <Card title="مساعدات وقت التشغيل" icon="settings" href="/plugins/sdk-runtime">
    TTS وSTT والوسائط وsubagent عبر api.runtime
  </Card>
</CardGroup>

<Note>
لا تزال بعض الوصلات المساعدة المضمّنة موجودة من أجل صيانة plugins المضمّنة
والتوافق. لكنها ليست النمط الموصى به لـ Plugins القنوات الجديدة؛
فضّل المسارات الفرعية العامة channel/setup/reply/runtime من
سطح SDK المشترك ما لم تكن تصون تلك العائلة المضمّنة من plugins مباشرةً.
</Note>

## الخطوات التالية

- [Plugins الموفّر](/plugins/sdk-provider-plugins) — إذا كان plugin الخاص بك يوفّر نماذج أيضًا
- [نظرة عامة على SDK](/plugins/sdk-overview) — مرجع استيراد المسارات الفرعية الكامل
- [اختبار SDK](/plugins/sdk-testing) — أدوات الاختبار واختبارات العقد
- [manifest الـ Plugin](/plugins/manifest) — مخطط manifest الكامل
