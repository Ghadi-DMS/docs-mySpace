---
read_when:
    - تحتاج إلى دلالات إعدادات دقيقة على مستوى الحقول أو إلى القيم الافتراضية
    - أنت تتحقق من كتل إعدادات القنوات أو النماذج أو البوابة أو الأدوات
summary: مرجع كامل لكل مفتاح إعدادات في OpenClaw والقيم الافتراضية وإعدادات القنوات
title: مرجع الإعدادات
x-i18n:
    generated_at: "2026-04-06T03:12:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6aa6b24b593f6f07118817afabea4cc7842aca6b7c5602b45f479b40c1685230
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# مرجع الإعدادات

كل حقل متاح في `~/.openclaw/openclaw.json`. للحصول على نظرة عامة موجهة بالمهام، راجع [الإعدادات](/ar/gateway/configuration).

تنسيق الإعدادات هو **JSON5** (التعليقات + الفواصل اللاحقة مسموح بها). جميع الحقول اختيارية — يستخدم OpenClaw قيمًا افتراضية آمنة عند حذفها.

---

## القنوات

تبدأ كل قناة تلقائيًا عندما يكون قسم إعداداتها موجودًا (إلا إذا كانت `enabled: false`).

### الوصول إلى الرسائل الخاصة والمجموعات

تدعم جميع القنوات سياسات الرسائل الخاصة وسياسات المجموعات:

| سياسة الرسائل الخاصة | السلوك |
| ------------------- | ------ |
| `pairing` (الافتراضي) | يحصل المرسلون غير المعروفين على رمز اقتران لمرة واحدة؛ ويجب على المالك الموافقة |
| `allowlist` | فقط المرسلون الموجودون في `allowFrom` (أو في مخزن السماح المقترن) |
| `open` | السماح بكل الرسائل الخاصة الواردة (يتطلب `allowFrom: ["*"]`) |
| `disabled` | تجاهل كل الرسائل الخاصة الواردة |

| سياسة المجموعات | السلوك |
| --------------------- | ------------------------------------------------------ |
| `allowlist` (الافتراضي) | فقط المجموعات المطابقة لقائمة السماح المضبوطة |
| `open` | تجاوز قوائم سماح المجموعات (مع استمرار تطبيق تقييد الإشارات) |
| `disabled` | حظر كل رسائل المجموعات/الغرف |

<Note>
يضبط `channels.defaults.groupPolicy` السياسة الافتراضية عندما تكون `groupPolicy` الخاصة بموفر ما غير مضبوطة.
تنتهي صلاحية رموز الاقتران بعد ساعة واحدة. ويُحدد الحد الأقصى لطلبات اقتران الرسائل الخاصة المعلقة عند **3 لكل قناة**.
إذا كانت كتلة الموفر مفقودة بالكامل (`channels.<provider>` غير موجودة)، فإن سياسة المجموعات في وقت التشغيل تعود إلى `allowlist` (إغلاق آمن) مع تحذير عند بدء التشغيل.
</Note>

### تجاوزات النموذج بحسب القناة

استخدم `channels.modelByChannel` لتثبيت معرّفات قنوات معينة على نموذج محدد. تقبل القيم `provider/model` أو الأسماء المستعارة للنماذج المضبوطة. ينطبق تعيين القناة عندما لا تكون للجلسة بالفعل قيمة تجاوز للنموذج (على سبيل المثال، تم تعيينها عبر `/model`).

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-4.1",
      },
      telegram: {
        "-1001234567890": "openai/gpt-4.1-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
      },
    },
  },
}
```

### الإعدادات الافتراضية للقنوات وheartbeat

استخدم `channels.defaults` لسلوك سياسة المجموعات وheartbeat المشترك عبر الموفرين:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`: سياسة المجموعات الاحتياطية عندما لا تكون `groupPolicy` على مستوى الموفر مضبوطة.
- `channels.defaults.contextVisibility`: وضع رؤية السياق التكميلي الافتراضي لكل القنوات. القيم: `all` (الافتراضي، تضمين كل سياق الاقتباس/السلسلة/السجل)، و`allowlist` (تضمين السياق من المرسلين الموجودين في قائمة السماح فقط)، و`allowlist_quote` (مثل allowlist لكن مع الإبقاء على سياق الاقتباس/الرد الصريح). تجاوز لكل قناة: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: تضمين حالات القنوات السليمة في مخرجات heartbeat.
- `channels.defaults.heartbeat.showAlerts`: تضمين الحالات المتدهورة/حالات الخطأ في مخرجات heartbeat.
- `channels.defaults.heartbeat.useIndicator`: عرض مخرجات heartbeat مدمجة بأسلوب المؤشر.

### WhatsApp

يعمل WhatsApp من خلال قناة الويب في البوابة (Baileys Web). ويبدأ تلقائيًا عندما تكون هناك جلسة مرتبطة.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // علامتا الصح الزرقاوان (false في وضع الدردشة الذاتية)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
  web: {
    enabled: true,
    heartbeatSeconds: 60,
    reconnect: {
      initialMs: 2000,
      maxMs: 120000,
      factor: 1.4,
      jitter: 0.2,
      maxAttempts: 0,
    },
  },
}
```

<Accordion title="WhatsApp متعدد الحسابات">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- تستخدم الأوامر الصادرة الحساب `default` افتراضيًا إذا كان موجودًا؛ وإلا يُستخدم أول معرّف حساب مضبوط (بعد الفرز).
- يجاوز `channels.whatsapp.defaultAccount` الاختياري اختيار الحساب الافتراضي الاحتياطي عندما يطابق معرّف حساب مضبوطًا.
- ينقل `openclaw doctor` دليل مصادقة Baileys القديم أحادي الحساب إلى `whatsapp/default`.
- تجاوزات لكل حساب: `channels.whatsapp.accounts.<id>.sendReadReceipts` و`channels.whatsapp.accounts.<id>.dmPolicy` و`channels.whatsapp.accounts.<id>.allowFrom`.

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: "partial", // off | partial | block | progress (الافتراضي: off؛ فعّله صراحة لتجنب حدود معدل تعديل المعاينة)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- Bot token: ‏`channels.telegram.botToken` أو `channels.telegram.tokenFile` (ملف عادي فقط؛ تُرفض الروابط الرمزية)، مع `TELEGRAM_BOT_TOKEN` كقيمة احتياطية للحساب الافتراضي.
- يجاوز `channels.telegram.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- في إعدادات تعدد الحسابات (معرّفا حسابين أو أكثر)، اضبط قيمة افتراضية صريحة (`channels.telegram.defaultAccount` أو `channels.telegram.accounts.default`) لتجنب التوجيه الاحتياطي؛ ويحذّر `openclaw doctor` عندما تكون هذه القيمة مفقودة أو غير صالحة.
- تمنع `configWrites: false` عمليات كتابة الإعدادات التي يبدأها Telegram (ترحيلات معرّف supergroup، وأوامر `/config set|unset`).
- تضبط إدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` روابط ACP دائمة لموضوعات المنتدى (استخدم `chatId:topic:topicId` القياسي في `match.peer.id`). تتم مشاركة دلالات الحقول في [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- تستخدم معاينات البث في Telegram ‏`sendMessage` + `editMessageText` (وتعمل في الدردشات المباشرة والجماعية).
- سياسة إعادة المحاولة: راجع [سياسة إعادة المحاولة](/ar/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Short answers only.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      chunkMode: "length", // length | newline
      streaming: "off", // off | partial | block | progress (progress تتحول إلى partial في Discord)
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // تفعيل اختياري لـ sessions_spawn({ thread: true })
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- الرمز المميز: `channels.discord.token`، مع `DISCORD_BOT_TOKEN` كقيمة احتياطية للحساب الافتراضي.
- تستخدم الاستدعاءات الصادرة المباشرة التي تقدم `token` صريحًا لـ Discord هذا الرمز في الاستدعاء؛ بينما تظل إعدادات إعادة المحاولة/السياسة الخاصة بالحساب تأتي من الحساب المحدد في لقطة وقت التشغيل النشطة.
- يجاوز `channels.discord.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- استخدم `user:<id>` ‏(DM) أو `channel:<id>` ‏(قناة guild) لأهداف التسليم؛ وتُرفض المعرّفات الرقمية العارية.
- تكون slugs الخاصة بـ Guild بأحرف صغيرة مع استبدال المسافات بـ `-`؛ وتستخدم مفاتيح القنوات الاسم المحوّل إلى slug (من دون `#`). ويفضل استخدام معرّفات guild.
- تُتجاهل الرسائل التي أنشأها البوت افتراضيًا. يفعّل `allowBots: true` قبولها؛ واستخدم `allowBots: "mentions"` لقبول رسائل البوت التي تذكر البوت فقط (مع استمرار تصفية الرسائل الذاتية).
- يقوم `channels.discord.guilds.<id>.ignoreOtherMentions` (وتجاوزات القنوات) بإسقاط الرسائل التي تذكر مستخدمًا أو دورًا آخر لكن لا تذكر البوت (باستثناء @everyone/@here).
- يقوم `maxLinesPerMessage` (الافتراضي 17) بتقسيم الرسائل الطويلة رأسيًا حتى لو كانت أقل من 2000 حرف.
- يتحكم `channels.discord.threadBindings` في التوجيه المرتبط بسلاسل Discord:
  - `enabled`: تجاوز Discord لميزات الجلسات المرتبطة بالسلسلة (`/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age` والتسليم/التوجيه المرتبط)
  - `idleHours`: تجاوز Discord لإلغاء التركيز التلقائي بسبب عدم النشاط بالساعات (`0` للتعطيل)
  - `maxAgeHours`: تجاوز Discord لأقصى عمر ثابت بالساعات (`0` للتعطيل)
  - `spawnSubagentSessions`: مفتاح تفعيل اختياري لإنشاء/ربط سلاسل تلقائيًا بواسطة `sessions_spawn({ thread: true })`
- تضبط إدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` روابط ACP دائمة للقنوات والسلاسل (استخدم معرّف القناة/السلسلة في `match.peer.id`). تتم مشاركة دلالات الحقول في [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- يضبط `channels.discord.ui.components.accentColor` لون التمييز لحاويات Discord components v2.
- يفعّل `channels.discord.voice` المحادثات في قنوات Discord الصوتية وتجاوزات الانضمام التلقائي + TTS الاختيارية.
- تمرر `channels.discord.voice.daveEncryption` و`channels.discord.voice.decryptionFailureTolerance` إلى خيارات DAVE في `@discordjs/voice` (القيم الافتراضية `true` و`24`).
- يحاول OpenClaw أيضًا استعادة استقبال الصوت عبر مغادرة/إعادة الانضمام إلى جلسة صوتية بعد فشل متكرر في فك التشفير.
- `channels.discord.streaming` هو المفتاح القياسي لوضع البث. وتتم ترحيل القيم القديمة `streamMode` والقيم المنطقية `streaming` تلقائيًا.
- يربط `channels.discord.autoPresence` توافر وقت التشغيل بحالة حضور البوت (سليم => online، متدهور => idle، مستنفد => dnd) ويسمح بتجاوزات نص الحالة الاختيارية.
- يعيد `channels.discord.dangerouslyAllowNameMatching` تفعيل المطابقة بالاسم/الوسم القابل للتغيير (وضع توافق للطوارئ).
- `channels.discord.execApprovals`: تسليم موافقات exec الأصلية في Discord وتفويض الموافقين.
  - `enabled`: ‏`true` أو `false` أو `"auto"` (الافتراضي). في وضع auto، تتفعّل موافقات exec عندما يمكن حل الموافقين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Discord المسموح لهم بالموافقة على طلبات exec. يعود إلى `commands.ownerAllowFrom` عند الحذف.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتوجيه الموافقات لكل الوكلاء.
  - `sessionFilter`: أنماط مفاتيح جلسة اختيارية (سلسلة جزئية أو regex).
  - `target`: مكان إرسال مطالبات الموافقة. ترسل `"dm"` (الافتراضي) إلى الرسائل الخاصة للموافقين، وترسل `"channel"` إلى القناة الأصلية، وترسل `"both"` إلى كليهما. عندما يتضمن الهدف `"channel"`، لا يمكن استخدام الأزرار إلا من قبل الموافقين الذين تم حلهم.
  - `cleanupAfterResolve`: عندما تكون `true`، تحذف الرسائل الخاصة الخاصة بالموافقة بعد الموافقة أو الرفض أو انتهاء المهلة.

**أوضاع إشعارات التفاعلات:** ‏`off` (لا شيء)، و`own` (رسائل البوت، الافتراضي)، و`all` (كل الرسائل)، و`allowlist` (من `guilds.<id>.users` على كل الرسائل).

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dm: {
        enabled: true,
        policy: "pairing",
        allowFrom: ["users/1234567890"],
      },
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- JSON لحساب الخدمة: مضمن (`serviceAccount`) أو من ملف (`serviceAccountFile`).
- كما أن SecretRef لحساب الخدمة مدعوم أيضًا (`serviceAccountRef`).
- القيم الاحتياطية من البيئة: `GOOGLE_CHAT_SERVICE_ACCOUNT` أو `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- استخدم `spaces/<spaceId>` أو `users/<userId>` لأهداف التسليم.
- يعيد `channels.googlechat.dangerouslyAllowNameMatching` تفعيل المطابقة مع email principal القابل للتغيير (وضع توافق للطوارئ).

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { allow: true, requireMention: true, allowBots: false },
        "#general": {
          allow: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "Short answers only.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all | batched
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      textChunkLimit: 4000,
      chunkMode: "length",
      streaming: "partial", // off | partial | block | progress (وضع المعاينة)
      nativeStreaming: true, // استخدام واجهة البث الأصلية في Slack عندما streaming=partial
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- **Socket mode** يتطلب كلًا من `botToken` و`appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` كقيم احتياطية من البيئة للحساب الافتراضي).
- **HTTP mode** يتطلب `botToken` بالإضافة إلى `signingSecret` (على الجذر أو لكل حساب).
- تقبل `botToken` و`appToken` و`signingSecret` و`userToken` سلاسل نصية
  عادية أو كائنات SecretRef.
- تكشف لقطات حساب Slack عن حقول مصدر/حالة لكل بيانات اعتماد مثل
  `botTokenSource` و`botTokenStatus` و`appTokenStatus` و، في وضع HTTP،
  `signingSecretStatus`. وتعني `configured_unavailable` أن الحساب
  مضبوط عبر SecretRef لكن مسار الأمر/وقت التشغيل الحالي لم يتمكن
  من حل قيمة السر.
- تمنع `configWrites: false` عمليات كتابة الإعدادات التي يبدأها Slack.
- يجاوز `channels.slack.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- `channels.slack.streaming` هو المفتاح القياسي لوضع البث. وتتم ترحيل القيم القديمة `streamMode` والقيم المنطقية `streaming` تلقائيًا.
- استخدم `user:<id>` ‏(DM) أو `channel:<id>` لأهداف التسليم.

**أوضاع إشعارات التفاعلات:** ‏`off`، و`own` (الافتراضي)، و`all`، و`allowlist` (من `reactionAllowlist`).

**عزل جلسات السلاسل:** يكون `thread.historyScope` لكل سلسلة (الافتراضي) أو مشتركًا عبر القناة. ويقوم `thread.inheritParent` بنسخ سجل القناة الأصلية إلى السلاسل الجديدة.

- يضيف `typingReaction` تفاعلًا مؤقتًا إلى رسالة Slack الواردة أثناء تشغيل الرد، ثم يزيله عند الاكتمال. استخدم shortcode لرمز Slack التعبيري مثل `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: تسليم موافقات exec الأصلية في Slack وتفويض الموافقين. المخطط نفسه كما في Discord: ‏`enabled` ‏(`true`/`false`/`"auto"`)، و`approvers` (معرّفات مستخدمي Slack)، و`agentFilter`، و`sessionFilter`، و`target` ‏(`"dm"` أو `"channel"` أو `"both"`).

| مجموعة الإجراءات | الافتراضي | ملاحظات |
| ------------ | ------- | ---------------------- |
| reactions    | مفعّل | التفاعل + سرد التفاعلات |
| messages     | مفعّل | قراءة/إرسال/تعديل/حذف |
| pins         | مفعّل | تثبيت/إلغاء تثبيت/سرد |
| memberInfo   | مفعّل | معلومات العضو |
| emojiList    | مفعّل | قائمة الرموز التعبيرية المخصصة |

### Mattermost

يأتي Mattermost كإضافة: `openclaw plugins install @openclaw/mattermost`.

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // تفعيل اختياري
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // عنوان URL صريح اختياري لعمليات النشر عبر reverse-proxy/العامة
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

أوضاع الدردشة: `oncall` (الرد عند @-mention، وهو الافتراضي)، و`onmessage` (كل رسالة)، و`onchar` (الرسائل التي تبدأ ببادئة تشغيل).

عندما تكون الأوامر الأصلية في Mattermost مفعّلة:

- يجب أن يكون `commands.callbackPath` مسارًا (مثل `/api/channels/mattermost/command`) وليس عنوان URL كاملًا.
- يجب أن يشير `commands.callbackUrl` إلى نقطة نهاية بوابة OpenClaw وأن يكون قابلاً للوصول من خادم Mattermost.
- تتم مصادقة استدعاءات الشرطة المائلة الأصلية باستخدام الرموز المميزة لكل أمر التي يعيدها
  Mattermost أثناء تسجيل أوامر الشرطة المائلة. إذا فشل التسجيل أو لم
  يتم تفعيل أي أوامر، يرفض OpenClaw الاستدعاءات برسالة
  `Unauthorized: invalid command token.`
- بالنسبة إلى مستضيفات الاستدعاء الخاصة/داخل tailnet/الداخلية، قد يتطلب Mattermost
  أن تتضمن `ServiceSettings.AllowedUntrustedInternalConnections` مستضيف/نطاق الاستدعاء.
  استخدم قيم المستضيف/النطاق، وليس عناوين URL كاملة.
- `channels.mattermost.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي يبدأها Mattermost.
- `channels.mattermost.requireMention`: طلب `@mention` قبل الرد في القنوات.
- `channels.mattermost.groups.<channelId>.requireMention`: تجاوز تقييد الإشارة لكل قناة (`"*"` للقيمة الافتراضية).
- يجاوز `channels.mattermost.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // ربط اختياري للحساب
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**أوضاع إشعارات التفاعلات:** ‏`off`، و`own` (الافتراضي)، و`all`، و`allowlist` (من `reactionAllowlist`).

- `channels.signal.account`: تثبيت بدء القناة على هوية حساب Signal محددة.
- `channels.signal.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي يبدأها Signal.
- يجاوز `channels.signal.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.

### BlueBubbles

يُعد BlueBubbles المسار الموصى به لـ iMessage (مدعومًا بإضافة، ويُضبط ضمن `channels.bluebubbles`).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, group controls, and advanced actions:
      // راجع /channels/bluebubbles
    },
  },
}
```

- مسارات المفاتيح الأساسية المغطاة هنا: `channels.bluebubbles` و`channels.bluebubbles.dmPolicy`.
- يجاوز `channels.bluebubbles.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات BlueBubbles بجلسات ACP دائمة. استخدم BlueBubbles handle أو سلسلة الهدف (`chat_id:*`، `chat_guid:*`، `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- تم توثيق إعدادات قناة BlueBubbles الكاملة في [BlueBubbles](/ar/channels/bluebubbles).

### iMessage

يقوم OpenClaw بتشغيل `imsg rpc` ‏(JSON-RPC عبر stdio). لا حاجة إلى daemon أو منفذ.

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      region: "US",
    },
  },
}
```

- يجاوز `channels.imessage.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.

- يتطلب Full Disk Access لقاعدة بيانات Messages.
- يفضل استخدام أهداف `chat_id:<id>`. استخدم `imsg chats --limit 20` لسرد الدردشات.
- يمكن أن يشير `cliPath` إلى غلاف SSH؛ واضبط `remoteHost` ‏(`host` أو `user@host`) لجلب المرفقات عبر SCP.
- تقيد `attachmentRoots` و`remoteAttachmentRoots` مسارات المرفقات الواردة (الافتراضي: `/Users/*/Library/Messages/Attachments`).
- يستخدم SCP التحقق الصارم من مفتاح المستضيف، لذا تأكد من أن مفتاح مستضيف الترحيل موجود بالفعل في `~/.ssh/known_hosts`.
- `channels.imessage.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي يبدأها iMessage.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات iMessage بجلسات ACP دائمة. استخدم handle مُطبَّعًا أو هدف دردشة صريحًا (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).

<Accordion title="مثال على غلاف SSH لـ iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix مدعوم بإضافة ويُضبط ضمن `channels.matrix`.

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- تستخدم مصادقة الرمز المميز `accessToken`؛ بينما تستخدم مصادقة كلمة المرور `userId` + `password`.
- يوجه `channels.matrix.proxy` حركة HTTP الخاصة بـ Matrix عبر وكيل HTTP(S) صريح. ويمكن للحسابات المسماة تجاوزه عبر `channels.matrix.accounts.<id>.proxy`.
- يسمح `channels.matrix.allowPrivateNetwork` بخوادم homeserver الخاصة/الداخلية. ويُعد `proxy` و`allowPrivateNetwork` عنصرَي تحكم مستقلين.
- يحدد `channels.matrix.defaultAccount` الحساب المفضل في إعدادات تعدد الحسابات.
- `channels.matrix.execApprovals`: تسليم موافقات exec الأصلية في Matrix وتفويض الموافقين.
  - `enabled`: ‏`true` أو `false` أو `"auto"` (الافتراضي). في وضع auto، تتفعّل موافقات exec عندما يمكن حل الموافقين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Matrix (مثل `@owner:example.org`) المسموح لهم بالموافقة على طلبات exec.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتوجيه الموافقات لكل الوكلاء.
  - `sessionFilter`: أنماط مفاتيح جلسة اختيارية (سلسلة جزئية أو regex).
  - `target`: مكان إرسال مطالبات الموافقة. `"dm"` (الافتراضي)، أو `"channel"` (الغرفة الأصلية)، أو `"both"`.
  - تجاوزات لكل حساب: `channels.matrix.accounts.<id>.execApprovals`.
- يتحكم `channels.matrix.dm.sessionScope` في كيفية تجميع الرسائل الخاصة في Matrix ضمن جلسات: تشارك `per-user` (الافتراضي) بحسب النظير الموجَّه، بينما تعزل `per-room` كل غرفة DM على حدة.
- تستخدم فحوصات حالة Matrix وعمليات البحث المباشر في الدليل سياسة الوكيل نفسها المستخدمة في حركة وقت التشغيل.
- يتم توثيق إعدادات Matrix الكاملة وقواعد الاستهداف وأمثلة الإعداد في [Matrix](/ar/channels/matrix).

### Microsoft Teams

Microsoft Teams مدعوم بإضافة ويُضبط ضمن `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, team/channel policies:
      // راجع /channels/msteams
    },
  },
}
```

- مسارات المفاتيح الأساسية المغطاة هنا: `channels.msteams` و`channels.msteams.configWrites`.
- تم توثيق إعدادات Teams الكاملة (بيانات الاعتماد وwebhook وسياسة الرسائل الخاصة/المجموعات والتجاوزات لكل فريق/قناة) في [Microsoft Teams](/ar/channels/msteams).

### IRC

IRC مدعوم بإضافة ويُضبط ضمن `channels.irc`.

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- مسارات المفاتيح الأساسية المغطاة هنا: `channels.irc` و`channels.irc.dmPolicy` و`channels.irc.configWrites` و`channels.irc.nickserv.*`.
- يجاوز `channels.irc.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- يتم توثيق إعدادات قناة IRC الكاملة (المستضيف/المنفذ/TLS/القنوات/قوائم السماح/تقييد الإشارات) في [IRC](/ar/channels/irc).

### تعدد الحسابات (كل القنوات)

شغّل عدة حسابات لكل قناة (لكل منها `accountId` خاص به):

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- يُستخدم `default` عند حذف `accountId` (CLI + التوجيه).
- لا تنطبق رموز البيئة إلا على الحساب **default**.
- تنطبق إعدادات القناة الأساسية على كل الحسابات ما لم يتم تجاوزها لكل حساب.
- استخدم `bindings[].match.accountId` لتوجيه كل حساب إلى وكيل مختلف.
- إذا أضفت حسابًا غير افتراضي عبر `openclaw channels add` (أو إعداد القناة) بينما لا تزال تستخدم إعداد قناة أحادي الحساب في المستوى الأعلى، يقوم OpenClaw أولًا بترقية القيم أحادية الحساب في المستوى الأعلى ذات النطاق الخاص بالحساب إلى خريطة حسابات القناة حتى يستمر الحساب الأصلي في العمل. تنقل معظم القنوات هذه القيم إلى `channels.<channel>.accounts.default`؛ ويمكن لـ Matrix بدلاً من ذلك الحفاظ على هدف مسمى/افتراضي موجود ومطابق.
- تظل روابط القناة فقط الموجودة مسبقًا (من دون `accountId`) مطابقة للحساب الافتراضي؛ بينما تظل الروابط ذات النطاق الخاص بالحساب اختيارية.
- يقوم `openclaw doctor --fix` أيضًا بإصلاح الأشكال المختلطة عن طريق نقل القيم أحادية الحساب ذات النطاق الخاص بالحساب في المستوى الأعلى إلى الحساب المُرقّى المختار لتلك القناة. تستخدم معظم القنوات `accounts.default`؛ ويمكن لـ Matrix بدلاً من ذلك الحفاظ على هدف مسمى/افتراضي موجود ومطابق.

### قنوات الإضافات الأخرى

تُضبط العديد من قنوات الإضافات على شكل `channels.<id>` ويتم توثيقها في صفحات القنوات المخصصة لها (مثل Feishu وMatrix وLINE وNostr وZalo وNextcloud Talk وSynology Chat وTwitch).
راجع فهرس القنوات الكامل: [القنوات](/ar/channels).

### تقييد الإشارة في دردشات المجموعات

تتطلب رسائل المجموعات الإشارة **افتراضيًا** (إشارة عبر metadata أو أنماط regex آمنة). ينطبق ذلك على دردشات مجموعات WhatsApp وTelegram وDiscord وGoogle Chat وiMessage.

**أنواع الإشارات:**

- **إشارات metadata**: إشارات @-mentions الأصلية في المنصة. ويتم تجاهلها في وضع الدردشة الذاتية في WhatsApp.
- **أنماط النص**: أنماط regex آمنة في `agents.list[].groupChat.mentionPatterns`. ويتم تجاهل الأنماط غير الصالحة والتكرار المتداخل غير الآمن.
- لا يُفرض تقييد الإشارة إلا عندما يكون الكشف ممكنًا (إشارات أصلية أو نمط واحد على الأقل).

```json5
{
  messages: {
    groupChat: { historyLimit: 50 },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

يضبط `messages.groupChat.historyLimit` القيمة الافتراضية العامة. ويمكن للقنوات التجاوز عبر `channels.<channel>.historyLimit` (أو لكل حساب). اضبطه على `0` للتعطيل.

#### حدود سجل الرسائل الخاصة

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

الحل: تجاوز لكل DM ← افتراضي الموفر ← بلا حد (الاحتفاظ بكل شيء).

المدعوم: `telegram` و`whatsapp` و`discord` و`slack` و`signal` و`imessage` و`msteams`.

#### وضع الدردشة الذاتية

ضمّن رقمك الخاص في `allowFrom` لتفعيل وضع الدردشة الذاتية (يتجاهل @-mentions الأصلية، ويرد فقط على أنماط النص):

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### الأوامر (معالجة أوامر الدردشة)

```json5
{
  commands: {
    native: "auto", // تسجيل الأوامر الأصلية عندما تكون مدعومة
    text: true, // تحليل /commands في رسائل الدردشة
    bash: false, // السماح بـ ! (اسم بديل: /bash)
    bashForegroundMs: 2000,
    config: false, // السماح بـ /config
    debug: false, // السماح بـ /debug
    restart: false, // السماح بـ /restart + أداة إعادة تشغيل البوابة
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="تفاصيل الأوامر">

- يجب أن تكون الأوامر النصية رسائل **مستقلة** تبدأ بـ `/`.
- يقوم `native: "auto"` بتفعيل الأوامر الأصلية لـ Discord/Telegram، ويترك Slack معطّلًا.
- التجاوز لكل قناة: `channels.discord.commands.native` ‏(bool أو `"auto"`). تؤدي القيمة `false` إلى مسح الأوامر المسجلة مسبقًا.
- يضيف `channels.telegram.customCommands` إدخالات إضافية إلى قائمة bot في Telegram.
- يفعّل `bash: true` الأمر `! <cmd>` لصدفة المستضيف. ويتطلب `tools.elevated.enabled` وأن يكون المرسل ضمن `tools.elevated.allowFrom.<channel>`.
- يفعّل `config: true` الأمر `/config` (قراءة/كتابة `openclaw.json`). بالنسبة إلى عملاء `chat.send` في البوابة، تتطلب عمليات الكتابة الدائمة عبر `/config set|unset` أيضًا `operator.admin`؛ بينما يظل `/config show` للقراءة فقط متاحًا لعملاء المشغل ذوي نطاق الكتابة العادي.
- يتحكم `channels.<provider>.configWrites` في طفرات الإعدادات لكل قناة (الافتراضي: true).
- بالنسبة إلى القنوات متعددة الحسابات، يتحكم أيضًا `channels.<provider>.accounts.<id>.configWrites` في عمليات الكتابة التي تستهدف ذلك الحساب (مثل `/allowlist --config --account <id>` أو `/config set channels.<provider>.accounts.<id>...`).
- يكون `allowFrom` لكل موفر. عند ضبطه، يكون هو **المصدر الوحيد** للتفويض (ويتم تجاهل قوائم سماح القنوات/الاقتران و`useAccessGroups`).
- يسمح `useAccessGroups: false` للأوامر بتجاوز سياسات مجموعات الوصول عندما لا تكون `allowFrom` مضبوطة.

</Accordion>

---

## الإعدادات الافتراضية للوكلاء

### `agents.defaults.workspace`

الافتراضي: `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

جذر مستودع اختياري يظهر في سطر Runtime داخل system prompt. إذا لم يتم ضبطه، يكتشفه OpenClaw تلقائيًا بالانتقال صعودًا من مساحة العمل.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

قائمة سماح افتراضية اختيارية لـ Skills للوكلاء الذين لا يضبطون
`agents.list[].skills`.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // يرث github, weather
      { id: "docs", skills: ["docs-search"] }, // يستبدل الإعدادات الافتراضية
      { id: "locked-down", skills: [] }, // بلا Skills
    ],
  },
}
```

- احذف `agents.defaults.skills` للسماح بكل Skills افتراضيًا بلا تقييد.
- احذف `agents.list[].skills` ليرث القيم الافتراضية.
- اضبط `agents.list[].skills: []` لعدم السماح بأي Skills.
- تكون القائمة غير الفارغة في `agents.list[].skills` هي المجموعة النهائية لذلك الوكيل؛
  ولا تندمج مع القيم الافتراضية.

### `agents.defaults.skipBootstrap`

يعطّل الإنشاء التلقائي لملفات bootstrap لمساحة العمل (`AGENTS.md` و`SOUL.md` و`TOOLS.md` و`IDENTITY.md` و`USER.md` و`HEARTBEAT.md` و`BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.bootstrapMaxChars`

أقصى عدد من الأحرف لكل ملف bootstrap لمساحة العمل قبل الاقتطاع. الافتراضي: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

أقصى إجمالي عدد للأحرف المحقونة عبر كل ملفات bootstrap لمساحة العمل. الافتراضي: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

يتحكم في نص التحذير المرئي للوكيل عندما يتم اقتطاع سياق bootstrap.
الافتراضي: `"once"`.

- `"off"`: لا تحقن نص تحذير في system prompt مطلقًا.
- `"once"`: احقن التحذير مرة واحدة لكل توقيع اقتطاع فريد (موصى به).
- `"always"`: احقن التحذير في كل تشغيل عندما يوجد اقتطاع.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

أقصى حجم بكسل لأطول ضلع في الصورة داخل كتل الصور في transcript/tool قبل استدعاءات الموفر.
الافتراضي: `1200`.

تقلل القيم المنخفضة عادةً من استخدام رموز الرؤية ومن حجم حمولة الطلب في التشغيلات الغنية بلقطات الشاشة.
بينما تحافظ القيم الأعلى على مزيد من التفاصيل البصرية.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

المنطقة الزمنية لسياق system prompt (وليس للطوابع الزمنية للرسائل). تعود إلى المنطقة الزمنية للمستضيف.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

تنسيق الوقت في system prompt. الافتراضي: `auto` (تفضيل نظام التشغيل).

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
        fallbacks: ["google/gemini-3.1-flash-image-preview"],
      },
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-i2v"],
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // params عامة افتراضية للموفر
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 3,
    },
  },
}
```

- `model`: يقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يضبط شكل السلسلة النموذج الأساسي فقط.
  - ويضبط شكل الكائن النموذج الأساسي مع نماذج failover مرتبة.
- `imageModel`: يقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة مسار أداة `image` باعتباره إعداد نموذج الرؤية الخاص بها.
  - ويُستخدم أيضًا كتوجيه احتياطي عندما لا يستطيع النموذج المحدد/الافتراضي قبول إدخال الصور.
- `imageGenerationModel`: يقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة قدرة توليد الصور المشتركة وأي سطح أداة/إضافة مستقبلي يولد الصور.
  - القيم الشائعة: `google/gemini-3.1-flash-image-preview` لتوليد الصور الأصلي في Gemini، أو `fal/fal-ai/flux/dev` لـ fal، أو `openai/gpt-image-1` لـ OpenAI Images.
  - إذا اخترت موفرًا/نموذجًا مباشرًا، فاضبط أيضًا مصادقة/مفتاح API الخاص بالموفر المطابق (مثل `GEMINI_API_KEY` أو `GOOGLE_API_KEY` لـ `google/*`، أو `OPENAI_API_KEY` لـ `openai/*`، أو `FAL_KEY` لـ `fal/*`).
  - إذا تم حذفه، فما زال بإمكان `image_generate` استنتاج موفر افتراضي مدعوم بالمصادقة. إذ يحاول أولًا موفر الإعدادات الافتراضي الحالي، ثم بقية موفري توليد الصور المسجلين بترتيب معرّف الموفر.
- `musicGenerationModel`: يقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة قدرة توليد الموسيقى المشتركة وبواسطة الأداة المدمجة `music_generate`.
  - القيم الشائعة: `google/lyria-3-clip-preview` أو `google/lyria-3-pro-preview` أو `minimax/music-2.5+`.
  - إذا تم حذفه، فما زال بإمكان `music_generate` استنتاج موفر افتراضي مدعوم بالمصادقة. إذ يحاول أولًا موفر الإعدادات الافتراضي الحالي، ثم بقية موفري توليد الموسيقى المسجلين بترتيب معرّف الموفر.
  - إذا اخترت موفرًا/نموذجًا مباشرًا، فاضبط أيضًا مصادقة/مفتاح API الخاص بالموفر المطابق.
- `videoGenerationModel`: يقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة قدرة توليد الفيديو المشتركة وبواسطة الأداة المدمجة `video_generate`.
  - القيم الشائعة: `qwen/wan2.6-t2v` أو `qwen/wan2.6-i2v` أو `qwen/wan2.6-r2v` أو `qwen/wan2.6-r2v-flash` أو `qwen/wan2.7-r2v`.
  - إذا تم حذفه، فما زال بإمكان `video_generate` استنتاج موفر افتراضي مدعوم بالمصادقة. إذ يحاول أولًا موفر الإعدادات الافتراضي الحالي، ثم بقية موفري توليد الفيديو المسجلين بترتيب معرّف الموفر.
  - إذا اخترت موفرًا/نموذجًا مباشرًا، فاضبط أيضًا مصادقة/مفتاح API الخاص بالموفر المطابق.
  - يدعم موفر توليد الفيديو المدمج لـ Qwen حاليًا حتى 1 فيديو خرج، و1 صورة إدخال، و4 مقاطع فيديو إدخال، ومدة 10 ثوانٍ، وخيارات على مستوى الموفر مثل `size` و`aspectRatio` و`resolution` و`audio` و`watermark`.
- `pdfModel`: يقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة أداة `pdf` لتوجيه النموذج.
  - إذا تم حذفه، تعود أداة PDF إلى `imageModel`، ثم إلى النموذج المحلول الخاص بالجلسة/الافتراضي.
- `pdfMaxBytesMb`: حد حجم PDF الافتراضي لأداة `pdf` عندما لا يتم تمرير `maxBytesMb` وقت الاستدعاء.
- `pdfMaxPages`: الحد الأقصى الافتراضي للصفحات التي يأخذها وضع الاستخراج الاحتياطي في أداة `pdf` في الاعتبار.
- `verboseDefault`: مستوى verbose الافتراضي للوكلاء. القيم: `"off"`، و`"on"`، و`"full"`. الافتراضي: `"off"`.
- `elevatedDefault`: مستوى المخرجات المرتفعة الافتراضي للوكلاء. القيم: `"off"`، و`"on"`، و`"ask"`، و`"full"`. الافتراضي: `"on"`.
- `model.primary`: بالتنسيق `provider/model` (مثل `openai/gpt-5.4`). إذا حذفت الموفر، يحاول OpenClaw أولًا الاسم المستعار، ثم مطابقة موفر مضبوط وفريد لذلك معرّف النموذج تمامًا، وبعد ذلك فقط يعود إلى الموفر الافتراضي المضبوط (سلوك توافق قديم، لذا يفضل استخدام `provider/model` الصريح). إذا لم يعد ذلك الموفر يوفّر النموذج الافتراضي المضبوط، يعود OpenClaw إلى أول موفر/نموذج مضبوط بدلًا من إظهار افتراضي قديم لموفر تمت إزالته.
- `models`: فهرس النماذج المضبوط وقائمة السماح لـ `/model`. ويمكن أن يتضمن كل إدخال `alias` (اختصار) و`params` (خاصة بالموفر، مثل `temperature` و`maxTokens` و`cacheRetention` و`context1m`).
- `params`: معلمات موفر افتراضية عامة تطبق على كل النماذج. تُضبط في `agents.defaults.params` (مثل `{ cacheRetention: "long" }`).
- أسبقية دمج `params` (في الإعدادات): يتم تجاوز `agents.defaults.params` (الأساس العام) بواسطة `agents.defaults.models["provider/model"].params` (لكل نموذج)، ثم تتجاوز `agents.list[].params` (لمعرّف الوكيل المطابق) حسب المفتاح. راجع [تخزين prompt مؤقتًا](/ar/reference/prompt-caching) للتفاصيل.
- تحفظ أدوات كتابة الإعدادات التي تطفر هذه الحقول (مثل `/models set` و`/models set-image` وأوامر إضافة/إزالة fallback) شكل الكائن القياسي وتحافظ على قوائم fallback الحالية عندما يكون ذلك ممكنًا.
- `maxConcurrent`: الحد الأقصى لعمليات تشغيل الوكلاء المتوازية عبر الجلسات (مع بقاء كل جلسة متسلسلة). الافتراضي: 4.

**اختصارات alias المدمجة** (لا تنطبق إلا عندما يكون النموذج ضمن `agents.defaults.models`):

| الاسم المستعار | النموذج |
| ------------------- | -------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`            |
| `sonnet`            | `anthropic/claude-sonnet-4-6`          |
| `gpt`               | `openai/gpt-5.4`                       |
| `gpt-mini`          | `openai/gpt-5.4-mini`                  |
| `gpt-nano`          | `openai/gpt-5.4-nano`                  |
| `gemini`            | `google/gemini-3.1-pro-preview`        |
| `gemini-flash`      | `google/gemini-3-flash-preview`        |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

تتغلب الأسماء المستعارة التي تضبطها دائمًا على القيم الافتراضية.

تفعّل نماذج Z.AI GLM-4.x وضع التفكير تلقائيًا ما لم تضبط `--thinking off` أو تعرّف `agents.defaults.models["zai/<model>"].params.thinking` بنفسك.
وتفعّل نماذج Z.AI ‏`tool_stream` افتراضيًا لبث استدعاءات الأدوات. اضبط `agents.defaults.models["zai/<model>"].params.tool_stream` على `false` لتعطيله.
وتستخدم نماذج Anthropic Claude 4.6 وضع التفكير `adaptive` افتراضيًا عندما لا يتم تعيين مستوى تفكير صريح.

- الجلسات مدعومة عندما يتم تعيين `sessionArg`.
- تمرير الصور مدعوم عندما تقبل `imageArg` مسارات الملفات.

### `agents.defaults.heartbeat`

تشغيلات heartbeat دورية.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m للتعطيل
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        lightContext: false, // الافتراضي: false؛ true يبقي فقط HEARTBEAT.md من ملفات bootstrap الخاصة بمساحة العمل
        isolatedSession: false, // الافتراضي: false؛ true يشغّل كل heartbeat في جلسة جديدة (من دون سجل محادثة)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (الافتراضي) | block
        target: "none", // الافتراضي: none | الخيارات: last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
      },
    },
  },
}
```

- `every`: سلسلة مدة (ms/s/m/h). الافتراضي: `30m` (مصادقة API-key) أو `1h` (مصادقة OAuth). اضبطها على `0m` للتعطيل.
- `suppressToolErrorWarnings`: عندما تكون true، يتم كبت حمولات تحذيرات أخطاء الأدوات أثناء تشغيلات heartbeat.
- `directPolicy`: سياسة التسليم المباشر/DM. تسمح `allow` (الافتراضي) بالتسليم المباشر إلى الأهداف. وتمنع `block` التسليم المباشر وتصدر `reason=dm-blocked`.
- `lightContext`: عندما تكون true، تستخدم تشغيلات heartbeat سياق bootstrap خفيفًا وتبقي فقط `HEARTBEAT.md` من ملفات bootstrap لمساحة العمل.
- `isolatedSession`: عندما تكون true، تعمل كل heartbeat في جلسة جديدة بلا سجل محادثة سابق. نمط العزل نفسه مثل cron `sessionTarget: "isolated"`. يقلل تكلفة الرموز لكل heartbeat من نحو 100K إلى نحو 2-5K رمز.
- لكل وكيل: اضبط `agents.list[].heartbeat`. عندما يعرّف أي وكيل `heartbeat`، **تعمل heartbeat لهؤلاء الوكلاء فقط**.
- تشغّل heartbeat أدوار وكيل كاملة — والفواصل الأقصر تستهلك مزيدًا من الرموز.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        timeoutSeconds: 900,
        reserveTokensFloor: 24000,
        identifierPolicy: "strict", // strict | off | custom
        identifierInstructions: "Preserve deployment IDs, ticket IDs, and host:port pairs exactly.", // يُستخدم عندما identifierPolicy=custom
        postCompactionSections: ["Session Startup", "Red Lines"], // [] لتعطيل إعادة الحقن
        model: "openrouter/anthropic/claude-sonnet-4-6", // تجاوز اختياري لنموذج مخصص للـ compaction فقط
        notifyUser: true, // إرسال إشعار موجز للمستخدم عند بدء compaction (الافتراضي: false)
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 6000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with the exact silent token NO_REPLY if nothing to store.",
        },
      },
    },
  },
}
```

- `mode`: ‏`default` أو `safeguard` (تلخيص مجزأ للتواريخ الطويلة). راجع [Compaction](/ar/concepts/compaction).
- `timeoutSeconds`: الحد الأقصى للثواني المسموح بها لعملية compaction واحدة قبل أن يجهضها OpenClaw. الافتراضي: `900`.
- `identifierPolicy`: ‏`strict` (الافتراضي) أو `off` أو `custom`. وتقوم `strict` بإضافة إرشادات مدمجة للاحتفاظ بالمعرّفات غير الشفافة أثناء تلخيص compaction.
- `identifierInstructions`: نص اختياري مخصص للحفاظ على المعرّفات يُستخدم عندما يكون `identifierPolicy=custom`.
- `postCompactionSections`: أسماء أقسام H2/H3 اختيارية من `AGENTS.md` لإعادة حقنها بعد compaction. القيمة الافتراضية `["Session Startup", "Red Lines"]`؛ واضبط `[]` للتعطيل. عندما تكون غير مضبوطة أو مضبوطة صراحة على هذا الزوج الافتراضي، يتم قبول العناوين الأقدم `Every Session`/`Safety` أيضًا كبديل قديم.
- `model`: تجاوز اختياري لـ `provider/model-id` لتلخيص compaction فقط. استخدمه عندما يجب أن تحافظ الجلسة الرئيسية على نموذج معيّن بينما يجب أن تعمل تلخيصات compaction على نموذج آخر؛ وعندما لا يتم ضبطه، يستخدم compaction النموذج الأساسي للجلسة.
- `notifyUser`: عندما تكون `true`، يرسل إشعارًا موجزًا إلى المستخدم عند بدء compaction (مثل "Compacting context..."). وهو معطّل افتراضيًا لإبقاء compaction صامتًا.
- `memoryFlush`: دور وكيلي صامت قبل auto-compaction لتخزين الذكريات الدائمة. ويتم تخطيه عندما تكون مساحة العمل للقراءة فقط.

### `agents.defaults.contextPruning`

يقتطع **نتائج الأدوات القديمة** من السياق الموجود في الذاكرة قبل إرسالها إلى LLM. ولا يعدّل سجل الجلسة على القرص.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // مدة (ms/s/m/h)، الوحدة الافتراضية: الدقائق
        keepLastAssistants: 3,
        softTrimRatio: 0.3,
        hardClearRatio: 0.5,
        minPrunableToolChars: 50000,
        softTrim: { maxChars: 4000, headChars: 1500, tailChars: 1500 },
        hardClear: { enabled: true, placeholder: "[Old tool result content cleared]" },
        tools: { deny: ["browser", "canvas"] },
      },
    },
  },
}
```

<Accordion title="سلوك وضع cache-ttl">

- يفعّل `mode: "cache-ttl"` تمريرات الاقتطاع.
- يتحكم `ttl` في عدد المرات التي يمكن أن يعمل فيها الاقتطاع مرة أخرى (بعد آخر لمس لذاكرة التخزين المؤقت).
- يقوم الاقتطاع أولًا باقتطاع نتائج الأدوات الكبيرة جزئيًا، ثم يمسح نتائج الأدوات الأقدم كليًا إذا لزم الأمر.

يحافظ **soft-trim** على البداية + النهاية ويدرج `...` في المنتصف.

ويستبدل **hard-clear** نتيجة الأداة بالكامل بالعنصر النائب.

ملاحظات:

- لا يتم اقتطاع/مسح كتل الصور مطلقًا.
- النِّسب تعتمد على الأحرف (تقريبية)، وليست على عدد الرموز بدقة.
- إذا كان هناك أقل من `keepLastAssistants` من رسائل المساعد، يتم تخطي الاقتطاع.

</Accordion>

راجع [اقتطاع الجلسة](/ar/concepts/session-pruning) لتفاصيل السلوك.

### بث الكتل

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom (استخدم minMs/maxMs)
    },
  },
}
```

- تتطلب القنوات غير Telegram تفعيل `*.blockStreaming: true` صراحة لتمكين الردود الكتلية.
- تجاوزات القنوات: `channels.<channel>.blockStreamingCoalesce` (ومتغيرات كل حساب). وتستخدم Signal/Slack/Discord/Google Chat افتراضيًا `minChars: 1500`.
- `humanDelay`: توقف عشوائي بين الردود الكتلية. وتعني `natural` = ‏800–2500ms. تجاوز لكل وكيل: `agents.list[].humanDelay`.

راجع [البث](/ar/concepts/streaming) لسلوك + تفاصيل التجزئة.

### مؤشرات الكتابة

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- القيم الافتراضية: `instant` للدردشات المباشرة/الإشارات، و`message` لدردشات المجموعات غير المشار إليها.
- تجاوزات لكل جلسة: `session.typingMode` و`session.typingIntervalSeconds`.

راجع [مؤشرات الكتابة](/ar/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

وضع sandbox اختياري للوكيل المضمن. راجع [Sandboxing](/ar/gateway/sandboxing) للدليل الكامل.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        backend: "docker", // docker | ssh | openshell
        scope: "agent", // session | agent | shared
        workspaceAccess: "none", // none | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefs / inline contents also supported:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

<Accordion title="تفاصيل sandbox">

**الواجهة الخلفية:**

- `docker`: بيئة Docker محلية (الافتراضي)
- `ssh`: بيئة بعيدة عامة مدعومة بـ SSH
- `openshell`: بيئة OpenShell

عندما يتم اختيار `backend: "openshell"`، تنتقل الإعدادات الخاصة بوقت التشغيل إلى
`plugins.entries.openshell.config`.

**إعدادات الواجهة الخلفية لـ SSH:**

- `target`: هدف SSH بالصيغة `user@host[:port]`
- `command`: أمر عميل SSH (الافتراضي: `ssh`)
- `workspaceRoot`: جذر بعيد مطلق يستخدم لمساحات العمل بحسب النطاق
- `identityFile` / `certificateFile` / `knownHostsFile`: ملفات محلية موجودة تمرَّر إلى OpenSSH
- `identityData` / `certificateData` / `knownHostsData`: محتوى مضمن أو SecretRefs يقوم OpenClaw بتحويله إلى ملفات مؤقتة وقت التشغيل
- `strictHostKeyChecking` / `updateHostKeys`: أدوات سياسة مفاتيح المستضيف في OpenSSH

**أسبقية مصادقة SSH:**

- تتغلب `identityData` على `identityFile`
- وتتغلب `certificateData` على `certificateFile`
- وتتغلب `knownHostsData` على `knownHostsFile`
- يتم حل القيم المدعومة بـ SecretRef من نوع `*Data` من لقطة وقت التشغيل النشطة للأسرار قبل بدء جلسة sandbox

**سلوك الواجهة الخلفية لـ SSH:**

- يبذر مساحة العمل البعيدة مرة واحدة بعد الإنشاء أو إعادة الإنشاء
- ثم يحافظ على مساحة العمل البعيدة عبر SSH بصفتها المرجع الأساسي
- يوجّه `exec` وأدوات الملفات ومسارات الوسائط عبر SSH
- لا يزامن التغييرات البعيدة تلقائيًا إلى المستضيف
- لا يدعم حاويات متصفح sandbox

**الوصول إلى مساحة العمل:**

- `none`: مساحة عمل sandbox بحسب النطاق ضمن `~/.openclaw/sandboxes`
- `ro`: مساحة عمل sandbox في `/workspace`، مع تحميل مساحة عمل الوكيل في `/agent` للقراءة فقط
- `rw`: تحميل مساحة عمل الوكيل للقراءة/الكتابة في `/workspace`

**النطاق:**

- `session`: حاوية + مساحة عمل لكل جلسة
- `agent`: حاوية + مساحة عمل واحدة لكل وكيل (الافتراضي)
- `shared`: حاوية ومساحة عمل مشتركتان (بلا عزل بين الجلسات)

**إعدادات إضافة OpenShell:**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // mirror | remote
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // اختياري
          gatewayEndpoint: "https://lab.example", // اختياري
          policy: "strict", // معرّف سياسة OpenShell اختياري
          providers: ["openai"], // اختياري
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**وضع OpenShell:**

- `mirror`: يبذر البعيد من المحلي قبل exec، ثم يزامن عودةً بعد exec؛ وتظل مساحة العمل المحلية هي المرجع الأساسي
- `remote`: يبذر البعيد مرة واحدة عند إنشاء sandbox، ثم يحافظ على مساحة العمل البعيدة بصفتها المرجع الأساسي

في وضع `remote`، لا تتم مزامنة التعديلات المحلية على المستضيف التي تُجرى خارج OpenClaw تلقائيًا إلى sandbox بعد خطوة البذر.
تُستخدم SSH للنقل إلى sandbox الخاص بـ OpenShell، لكن الإضافة تملك دورة حياة sandbox والمزامنة المرآتية الاختيارية.

يعمل **`setupCommand`** مرة واحدة بعد إنشاء الحاوية (عبر `sh -lc`). ويتطلب خروج شبكة، وجذرًا قابلًا للكتابة، ومستخدم root.

**تستخدم الحاويات افتراضيًا `network: "none"`** — اضبطها على `"bridge"` (أو شبكة bridge مخصصة) إذا كان الوكيل يحتاج إلى وصول صادر.
يتم حظر `"host"`. ويتم حظر `"container:<id>"` افتراضيًا ما لم تضبط
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` صراحة (للطوارئ).

**تُوضع المرفقات الواردة** في `media/inbound/*` داخل مساحة العمل النشطة.

**يقوم `docker.binds`** بتحميل أدلة إضافية من المستضيف؛ ويتم دمج التحميلات العامة وتلك الخاصة بكل وكيل.

**متصفح sandbox** ‏(`sandbox.browser.enabled`): Chromium + CDP داخل حاوية. يتم حقن عنوان noVNC URL في system prompt. ولا يتطلب `browser.enabled` في `openclaw.json`.
يستخدم الوصول للمراقبة عبر noVNC مصادقة VNC افتراضيًا، ويصدر OpenClaw عنوان URL برمز مميز قصير العمر (بدلًا من كشف كلمة المرور في الرابط المشترك).

- يمنع `allowHostControl: false` (الافتراضي) الجلسات المحمية بـ sandbox من استهداف متصفح المستضيف.
- تكون `network` افتراضيًا `openclaw-sandbox-browser` (شبكة bridge مخصصة). واضبطها على `bridge` فقط عندما تريد صراحة اتصال bridge عامًا.
- يقيّد `cdpSourceRange` اختياريًا دخول CDP على حافة الحاوية إلى مدى CIDR (مثل `172.21.0.1/32`).
- يقوم `sandbox.browser.binds` بتحميل أدلة إضافية من المستضيف في حاوية متصفح sandbox فقط. وعند ضبطه (بما في ذلك `[]`)، فإنه يستبدل `docker.binds` لحاوية المتصفح.
- تُعرّف القيم الافتراضية للإطلاق في `scripts/sandbox-browser-entrypoint.sh` وقد تم ضبطها لمستضيفات الحاويات:
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-3d-apis`
  - `--disable-gpu`
  - `--disable-software-rasterizer`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-features=TranslateUI`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--renderer-process-limit=2`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--disable-extensions` (مفعّل افتراضيًا)
  - يتم تفعيل `--disable-3d-apis` و`--disable-software-rasterizer` و`--disable-gpu`
    افتراضيًا، ويمكن تعطيلها باستخدام
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` إذا كان استخدام WebGL/3D يتطلب ذلك.
  - يعيد `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` تفعيل الإضافات إذا كان سير عملك
    يعتمد عليها.
  - يمكن تغيير `--renderer-process-limit=2` باستخدام
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`؛ اضبطه على `0` لاستخدام
    حد العمليات الافتراضي في Chromium.
  - بالإضافة إلى `--no-sandbox` و`--disable-setuid-sandbox` عندما يكون `noSandbox` مفعّلًا.
  - تمثل القيم الافتراضية خط الأساس لصورة الحاوية؛ استخدم صورة متصفح مخصصة مع
    entrypoint مخصص لتغيير القيم الافتراضية للحاوية.

</Accordion>

يقتصر Browser sandboxing و`sandbox.docker.binds` حاليًا على Docker فقط.

أنشئ الصور:

```bash
scripts/sandbox-setup.sh           # صورة sandbox الرئيسية
scripts/sandbox-browser-setup.sh   # صورة المتصفح الاختيارية
```

### `agents.list` (تجاوزات لكل وكيل)

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // أو { primary, fallbacks }
        thinkingDefault: "high", // تجاوز مستوى التفكير الافتراضي لكل وكيل
        reasoningDefault: "on", // تجاوز رؤية reasoning الافتراضية لكل وكيل
        fastModeDefault: false, // تجاوز fast mode الافتراضي لكل وكيل
        params: { cacheRetention: "none" }, // تتجاوز params الخاصة بالنموذج المطابق في defaults.models حسب المفتاح
        skills: ["docs-search"], // تستبدل agents.defaults.skills عند ضبطها
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`: معرّف وكيل ثابت (مطلوب).
- `default`: عند ضبط أكثر من واحد، يفوز الأول (ويتم تسجيل تحذير). وإذا لم يُضبط أي واحد، يكون أول إدخال في القائمة هو الافتراضي.
- `model`: يتجاوز شكل السلسلة `primary` فقط؛ بينما يتجاوز شكل الكائن `{ primary, fallbacks }` كليهما (`[]` تعطّل القيم الافتراضية العامة لـ fallbacks). تظل وظائف cron التي تتجاوز `primary` فقط ترث fallbacks الافتراضية ما لم تضبط `fallbacks: []`.
- `params`: معلمات stream لكل وكيل يتم دمجها فوق إدخال النموذج المحدد في `agents.defaults.models`. استخدم هذا لتجاوزات خاصة بالوكيل مثل `cacheRetention` أو `temperature` أو `maxTokens` من دون تكرار فهرس النماذج بأكمله.
- `skills`: قائمة سماح Skills اختيارية لكل وكيل. إذا حُذفت، يرث الوكيل `agents.defaults.skills` عندما تكون مضبوطة؛ والقائمة الصريحة تستبدل القيم الافتراضية بدل دمجها، و`[]` تعني عدم وجود Skills.
- `thinkingDefault`: مستوى التفكير الافتراضي الاختياري لكل وكيل (`off | minimal | low | medium | high | xhigh | adaptive`). ويتجاوز `agents.defaults.thinkingDefault` لهذا الوكيل عندما لا يكون هناك تجاوز لكل رسالة أو جلسة.
- `reasoningDefault`: تجاوز اختياري لرؤية reasoning الافتراضية لكل وكيل (`on | off | stream`). ويطبق عندما لا يكون هناك تجاوز لـ reasoning لكل رسالة أو جلسة.
- `fastModeDefault`: افتراضي اختياري لوضع fast mode (`true | false`) لكل وكيل. ويطبق عندما لا يكون هناك تجاوز لكل رسالة أو جلسة.
- `runtime`: واصف runtime اختياري لكل وكيل. استخدم `type: "acp"` مع القيم الافتراضية لـ `runtime.acp` (`agent` و`backend` و`mode` و`cwd`) عندما يجب أن يستخدم الوكيل جلسات ACP harness افتراضيًا.
- `identity.avatar`: مسار نسبي لمساحة العمل، أو عنوان URL من نوع `http(s)`، أو URI من نوع `data:`.
- تستنتج `identity` القيم الافتراضية: `ackReaction` من `emoji`، و`mentionPatterns` من `name`/`emoji`.
- `subagents.allowAgents`: قائمة سماح لمعرّفات الوكلاء لـ `sessions_spawn` (`["*"]` = أي وكيل؛ الافتراضي: الوكيل نفسه فقط).
- حاجز وراثة sandbox: إذا كانت جلسة الطالب ضمن sandbox، يرفض `sessions_spawn` الأهداف التي ستعمل خارج sandbox.
- `subagents.requireAgentId`: عندما تكون true، تحظر استدعاءات `sessions_spawn` التي تحذف `agentId` (ويفرض ذلك اختيار ملف تعريف صريح؛ الافتراضي: false).

---

## توجيه الوكلاء المتعددين

شغّل عدة وكلاء معزولين داخل بوابة واحدة. راجع [الوكلاء المتعددون](/ar/concepts/multi-agent).

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### حقول مطابقة الربط

- `type` (اختياري): `route` للتوجيه العادي (ويُعد النوع المفقود route افتراضيًا)، و`acp` لروابط المحادثات الدائمة لـ ACP.
- `match.channel` (مطلوب)
- `match.accountId` (اختياري؛ `*` = أي حساب؛ والمحذوف = الحساب الافتراضي)
- `match.peer` (اختياري؛ `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (اختياري؛ خاصة بالقناة)
- `acp` (اختياري؛ فقط لإدخالات `type: "acp"`): ‏`{ mode, label, cwd, backend }`

**ترتيب المطابقة الحتمي:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (مطابقة تامة، بلا peer/guild/team)
5. `match.accountId: "*"` (على مستوى القناة)
6. الوكيل الافتراضي

ضمن كل مستوى، يفوز أول إدخال `bindings` مطابق.

بالنسبة إلى إدخالات `type: "acp"`، يحل OpenClaw وفق هوية المحادثة الدقيقة (`match.channel` + الحساب + `match.peer.id`) ولا يستخدم ترتيب مستويات route أعلاه.

### ملفات تعريف الوصول لكل وكيل

<Accordion title="وصول كامل (بلا sandbox)">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="أدوات + مساحة عمل للقراءة فقط">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="بلا وصول إلى نظام الملفات (مراسلة فقط)">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

راجع [Sandbox وأدوات الوكلاء المتعددين](/ar/tools/multi-agent-sandbox-tools) لتفاصيل الأسبقية.

---

## الجلسة

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    parentForkMaxTokens: 100000, // تخطّي fork للسلسلة الأصلية فوق هذا العدد من الرموز (0 للتعطيل)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // مدة أو false
      maxDiskBytes: "500mb", // ميزانية قصوى اختيارية
      highWaterBytes: "400mb", // هدف تنظيف اختياري
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // إلغاء التركيز التلقائي الافتراضي بسبب عدم النشاط بالساعات (`0` للتعطيل)
      maxAgeHours: 0, // الحد الأقصى الثابت الافتراضي للعمر بالساعات (`0` للتعطيل)
    },
    mainKey: "main", // قديم (يستخدم runtime دائمًا "main")
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="تفاصيل حقول الجلسة">

- **`scope`**: إستراتيجية التجميع الأساسية للجلسات في سياقات دردشات المجموعات.
  - `per-sender` (الافتراضي): يحصل كل مرسل على جلسة معزولة داخل سياق القناة.
  - `global`: يشترك كل المشاركين في سياق القناة في جلسة واحدة (استخدمه فقط عندما يكون السياق المشترك مقصودًا).
- **`dmScope`**: كيفية تجميع الرسائل الخاصة.
  - `main`: تشترك كل الرسائل الخاصة في الجلسة الرئيسية.
  - `per-peer`: عزل بحسب معرّف المرسل عبر القنوات.
  - `per-channel-peer`: عزل بحسب القناة + المرسل (موصى به لصناديق الوارد متعددة المستخدمين).
  - `per-account-channel-peer`: عزل بحسب الحساب + القناة + المرسل (موصى به لتعدد الحسابات).
- **`identityLinks`**: يربط المعرّفات القياسية بالنظراء المسبوقين بالموفر من أجل مشاركة الجلسات عبر القنوات.
- **`reset`**: سياسة reset الأساسية. يقوم `daily` بإعادة التعيين عند `atHour` حسب التوقيت المحلي؛ ويعيد `idle` التعيين بعد `idleMinutes`. وعندما يتم ضبط كليهما، يفوز الذي تنتهي صلاحيته أولًا.
- **`resetByType`**: تجاوزات بحسب النوع (`direct` و`group` و`thread`). ويُقبل `dm` القديم كاسم بديل لـ `direct`.
- **`parentForkMaxTokens`**: الحد الأقصى المسموح به من `totalTokens` لجلسة الأصل عند إنشاء جلسة سلسلة مفروعة (الافتراضي `100000`).
  - إذا تجاوزت قيمة `totalTokens` للأصل هذا الحد، يبدأ OpenClaw جلسة سلسلة جديدة بدلًا من وراثة سجل transcript الخاص بالأصل.
  - اضبطه على `0` لتعطيل هذا الحاجز والسماح دائمًا بالتفريع من الأصل.
- **`mainKey`**: حقل قديم. يستخدم runtime الآن دائمًا `"main"` للدلو الرئيسي للدردشة المباشرة.
- **`agentToAgent.maxPingPongTurns`**: الحد الأقصى لعدد أدوار الرد المتبادل بين الوكلاء أثناء تبادلات وكيل-إلى-وكيل (عدد صحيح، المدى: `0`–`5`). وتعطّل القيمة `0` تسلسل ping-pong.
- **`sendPolicy`**: المطابقة بحسب `channel` أو `chatType` ‏(`direct|group|channel`، مع `dm` القديم كاسم بديل)، أو `keyPrefix`، أو `rawKeyPrefix`. ويفوز أول deny.
- **`maintenance`**: عناصر التحكم في تنظيف مخزن الجلسات والاحتفاظ بها.
  - `mode`: يطلق `warn` تحذيرات فقط؛ بينما يطبق `enforce` التنظيف.
  - `pruneAfter`: حد العمر للإدخالات القديمة (الافتراضي `30d`).
  - `maxEntries`: الحد الأقصى لعدد الإدخالات في `sessions.json` (الافتراضي `500`).
  - `rotateBytes`: تدوير `sessions.json` عندما يتجاوز هذا الحجم (الافتراضي `10mb`).
  - `resetArchiveRetention`: مدة الاحتفاظ بأرشيفات transcript من نوع `*.reset.<timestamp>`. وتعود افتراضيًا إلى `pruneAfter`؛ واضبطها على `false` للتعطيل.
  - `maxDiskBytes`: ميزانية قرص اختيارية لدليل الجلسات. في وضع `warn` يسجل تحذيرات؛ وفي وضع `enforce` يزيل أقدم القطع الأثرية/الجلسات أولًا.
  - `highWaterBytes`: هدف اختياري بعد تنظيف الميزانية. والقيمة الافتراضية هي `80%` من `maxDiskBytes`.
- **`threadBindings`**: القيم الافتراضية العامة لميزات الجلسات المرتبطة بالسلاسل.
  - `enabled`: مفتاح افتراضي رئيسي (يمكن للموفرين تجاوزه؛ ويستخدم Discord ‏`channels.discord.threadBindings.enabled`)
  - `idleHours`: القيمة الافتراضية لإلغاء التركيز التلقائي بسبب عدم النشاط بالساعات (`0` للتعطيل؛ ويمكن للموفرين التجاوز)
  - `maxAgeHours`: القيمة الافتراضية للحد الأقصى الثابت للعمر بالساعات (`0` للتعطيل؛ ويمكن للموفرين التجاوز)

</Accordion>

---

## الرسائل

```json5
{
  messages: {
    responsePrefix: "🦞", // أو "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all
    removeAckAfterReply: false,
    queue: {
      mode: "collect", // steer | followup | collect | steer-backlog | steer+backlog | queue | interrupt
      debounceMs: 1000,
      cap: 20,
      drop: "summarize", // old | new | summarize
      byChannel: {
        whatsapp: "collect",
        telegram: "collect",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 للتعطيل
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### بادئة الرد

تجاوزات لكل قناة/حساب: `channels.<channel>.responsePrefix` و`channels.<channel>.accounts.<id>.responsePrefix`.

الحل (الأكثر تحديدًا يفوز): الحساب ← القناة ← العام. تعطل `""` البادئة وتوقف السلسلة التصاعدية. وتستنتج `"auto"` القيمة `[{identity.name}]`.

**متغيرات القالب:**

| المتغير | الوصف | المثال |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | الاسم المختصر للنموذج | `claude-opus-4-6`           |
| `{modelFull}`     | معرّف النموذج الكامل | `anthropic/claude-opus-4-6` |
| `{provider}`      | اسم الموفر | `anthropic`                 |
| `{thinkingLevel}` | مستوى التفكير الحالي | `high`, `low`, `off`        |
| `{identity.name}` | اسم هوية الوكيل | (مثل `"auto"`)          |

تُعامل المتغيرات من دون حساسية لحالة الأحرف. ويعد `{think}` اسمًا بديلًا لـ `{thinkingLevel}`.

### تفاعل التأكيد

- يعود افتراضيًا إلى `identity.emoji` للوكيل النشط، أو `"👀"` إذا لم توجد. واضبط `""` للتعطيل.
- تجاوزات لكل قناة: `channels.<channel>.ackReaction` و`channels.<channel>.accounts.<id>.ackReaction`.
- ترتيب الحل: الحساب ← القناة ← `messages.ackReaction` ← بديل الهوية.
- النطاق: `group-mentions` (الافتراضي)، و`group-all`، و`direct`، و`all`.
- يزيل `removeAckAfterReply` تفاعل التأكيد بعد الرد في Slack وDiscord وTelegram.
- يفعّل `messages.statusReactions.enabled` تفاعلات الحالة الدورية في Slack وDiscord وTelegram.
  في Slack وDiscord، يبقي تركه غير مضبوط تفاعلات الحالة مفعّلة عندما تكون تفاعلات التأكيد نشطة.
  وفي Telegram، اضبطه صراحة على `true` لتفعيل تفاعلات الحالة الدورية.

### إزالة الارتجاج للرسائل الواردة

يجمع الرسائل النصية السريعة من المرسل نفسه في دور وكيل واحد. وتؤدي الوسائط/المرفقات إلى تفريغ فوري. وتتجاوز أوامر التحكم إزالة الارتجاج.

### TTS (تحويل النص إلى كلام)

```json5
{
  messages: {
    tts: {
      auto: "always", // off | always | inbound | tagged
      mode: "final", // final | all
      provider: "elevenlabs",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: { enabled: true },
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.openclaw/settings/tts.json",
      elevenlabs: {
        apiKey: "elevenlabs_api_key",
        baseUrl: "https://api.elevenlabs.io",
        voiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      openai: {
        apiKey: "openai_api_key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        voice: "alloy",
      },
    },
  },
}
```

- يتحكم `auto` في TTS التلقائي. ويتجاوز `/tts off|always|inbound|tagged` ذلك لكل جلسة.
- يتجاوز `summaryModel` القيمة `agents.defaults.model.primary` من أجل التلخيص التلقائي.
- تكون `modelOverrides` مفعّلة افتراضيًا؛ وتكون القيمة الافتراضية لـ `modelOverrides.allowProvider` هي `false` (تفعيل اختياري).
- تعود مفاتيح API إلى `ELEVENLABS_API_KEY`/`XI_API_KEY` و`OPENAI_API_KEY`.
- يتجاوز `openai.baseUrl` نقطة نهاية OpenAI TTS. وترتيب الحل هو: الإعدادات، ثم `OPENAI_TTS_BASE_URL`، ثم `https://api.openai.com/v1`.
- عندما يشير `openai.baseUrl` إلى نقطة نهاية ليست لـ OpenAI، يعاملها OpenClaw على أنها خادم TTS متوافق مع OpenAI ويخفف التحقق من صحة النموذج/الصوت.

---

## Talk

الإعدادات الافتراضية لوضع Talk ‏(macOS/iOS/Android).

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
    },
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
  },
}
```

- يجب أن يطابق `talk.provider` مفتاحًا في `talk.providers` عند ضبط عدة موفري Talk.
- مفاتيح Talk القديمة المسطحة (`talk.voiceId` و`talk.voiceAliases` و`talk.modelId` و`talk.outputFormat` و`talk.apiKey`) هي للتوافق فقط ويتم ترحيلها تلقائيًا إلى `talk.providers.<provider>`.
- تعود معرّفات الأصوات إلى `ELEVENLABS_VOICE_ID` أو `SAG_VOICE_ID`.
- تقبل `providers.*.apiKey` سلاسل نصية عادية أو كائنات SecretRef.
- لا تنطبق القيمة الاحتياطية `ELEVENLABS_API_KEY` إلا عندما لا يكون هناك مفتاح Talk API مضبوط.
- يسمح `providers.*.voiceAliases` لتوجيهات Talk باستخدام أسماء سهلة.
- يتحكم `silenceTimeoutMs` في المدة التي ينتظرها وضع Talk بعد صمت المستخدم قبل إرسال transcript. وعندما لا يتم ضبطه، يحتفظ بنافذة التوقف الافتراضية للمنصة (`700 ms على macOS وAndroid، و900 ms على iOS`).

---

## الأدوات

### ملفات تعريف الأدوات

يضبط `tools.profile` قائمة سماح أساسية قبل `tools.allow`/`tools.deny`:

تضبط عملية الإعداد المحلي الإعدادات الجديدة محليًا افتراضيًا على `tools.profile: "coding"` عندما لا تكون مضبوطة (وتُحافظ على ملفات التعريف الصريحة الموجودة).

| ملف التعريف | يتضمن |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | `session_status` فقط |
| `coding`    | `group:fs`، و`group:runtime`، و`group:web`، و`group:sessions`، و`group:memory`، و`cron`، و`image`، و`image_generate`، و`video_generate` |
| `messaging` | `group:messaging`، و`sessions_list`، و`sessions_history`، و`sessions_send`، و`session_status` |
| `full`      | بلا تقييد (مثل عدم الضبط) |

### مجموعات الأدوات

| المجموعة | الأدوات |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`، و`process`، و`code_execution` (`bash` مقبول كاسم بديل لـ `exec`) |
| `group:fs`         | `read`، و`write`، و`edit`، و`apply_patch` |
| `group:sessions`   | `sessions_list`، و`sessions_history`، و`sessions_send`، و`sessions_spawn`، و`sessions_yield`، و`subagents`، و`session_status` |
| `group:memory`     | `memory_search`، و`memory_get` |
| `group:web`        | `web_search`، و`x_search`، و`web_fetch` |
| `group:ui`         | `browser`، و`canvas` |
| `group:automation` | `cron`، و`gateway` |
| `group:messaging`  | `message` |
| `group:nodes`      | `nodes` |
| `group:agents`     | `agents_list` |
| `group:media`      | `image`، و`image_generate`، و`video_generate`، و`tts` |
| `group:openclaw`   | كل الأدوات المدمجة (باستثناء إضافات الموفر) |

### `tools.allow` / `tools.deny`

سياسة السماح/المنع العامة للأدوات (والمنع يفوز). غير حساسة لحالة الأحرف، وتدعم أحرف البدل `*`. وتُطبق حتى عندما يكون Docker sandbox معطّلًا.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

تقييد إضافي للأدوات لنماذج أو موفرين محددين. الترتيب: ملف التعريف الأساسي ← ملف تعريف الموفر ← السماح/المنع.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.elevated`

يتحكم في وصول exec المرتفع خارج sandbox:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- يمكن لتجاوز كل وكيل (`agents.list[].tools.elevated`) فقط زيادة التقييد.
- يقوم `/elevated on|off|ask|full` بتخزين الحالة لكل جلسة؛ بينما تنطبق التوجيهات المضمنة على رسالة واحدة.
- يتجاوز `exec` المرتفع sandboxing ويستخدم مسار الهروب المضبوط (`gateway` افتراضيًا، أو `node` عندما يكون هدف exec هو `node`).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      applyPatch: {
        enabled: false,
        allowModels: ["gpt-5.4"],
      },
    },
  },
}
```

### `tools.loopDetection`

فحوصات أمان حلقات الأدوات **معطّلة افتراضيًا**. اضبط `enabled: true` لتفعيل الكشف.
يمكن تعريف الإعدادات عالميًا في `tools.loopDetection` وتجاوزها لكل وكيل في `agents.list[].tools.loopDetection`.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
      historySize: 30,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },
  },
}
```

- `historySize`: الحد الأقصى لسجل استدعاءات الأدوات المحتفَظ به لتحليل الحلقات.
- `warningThreshold`: عتبة أنماط عدم التقدم المتكررة للتحذيرات.
- `criticalThreshold`: عتبة أعلى لحظر الحلقات الحرجة.
- `globalCircuitBreakerThreshold`: عتبة إيقاف صارم لأي تشغيل بلا تقدم.
- `detectors.genericRepeat`: التحذير من استدعاءات الأداة نفسها/الوسائط نفسها بشكل متكرر.
- `detectors.knownPollNoProgress`: التحذير/الحظر للأدوات المعروفة للاستطلاع (`process.poll` و`command_status` وغيرها).
- `detectors.pingPong`: التحذير/الحظر لأنماط الأزواج المتبادلة بلا تقدم.
- إذا كانت `warningThreshold >= criticalThreshold` أو `criticalThreshold >= globalCircuitBreakerThreshold`، يفشل التحقق.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // أو BRAVE_API_KEY من البيئة
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // اختياري؛ احذفه للكشف التلقائي
        maxChars: 50000,
        maxCharsCap: 50000,
        maxResponseBytes: 2000000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

### `tools.media`

يضبط فهم الوسائط الواردة (صورة/صوت/فيديو):

```json5
{
  tools: {
    media: {
      concurrency: 2,
      asyncCompletion: {
        directSend: false, // تفعيل اختياري: إرسال المهام الموسيقية/الفيديو غير المتزامنة المكتملة مباشرة إلى القناة
      },
      audio: {
        enabled: true,
        maxBytes: 20971520,
        scope: {
          default: "deny",
          rules: [{ action: "allow", match: { chatType: "direct" } }],
        },
        models: [
          { provider: "openai", model: "gpt-4o-mini-transcribe" },
          { type: "cli", command: "whisper", args: ["--model", "base", "{{MediaPath}}"] },
        ],
      },
      video: {
        enabled: true,
        maxBytes: 52428800,
        models: [{ provider: "google", model: "gemini-3-flash-preview" }],
      },
    },
  },
}
```

<Accordion title="حقول إدخال نموذج الوسائط">

**إدخال الموفر** (`type: "provider"` أو محذوف):

- `provider`: معرّف موفر API (`openai` أو `anthropic` أو `google`/`gemini` أو `groq` وما إلى ذلك)
- `model`: تجاوز معرّف النموذج
- `profile` / `preferredProfile`: اختيار ملف تعريف `auth-profiles.json`

**إدخال CLI** (`type: "cli"`):

- `command`: الملف التنفيذي الذي سيُشغَّل
- `args`: وسائط قالبية (تدعم `{{MediaPath}}` و`{{Prompt}}` و`{{MaxChars}}` وغيرها)

**الحقول المشتركة:**

- `capabilities`: قائمة اختيارية (`image` أو `audio` أو `video`). القيم الافتراضية: `openai`/`anthropic`/`minimax` ← صورة، و`google` ← صورة+صوت+فيديو، و`groq` ← صوت.
- `prompt` و`maxChars` و`maxBytes` و`timeoutSeconds` و`language`: تجاوزات لكل إدخال.
- تؤدي حالات الفشل إلى الرجوع إلى الإدخال التالي.

تتبع مصادقة الموفر الترتيب القياسي: `auth-profiles.json` ← متغيرات البيئة ← `models.providers.*.apiKey`.

**حقول الإكمال غير المتزامن:**

- `asyncCompletion.directSend`: عندما تكون `true`، تحاول مهام
  `music_generate` و`video_generate` غير المتزامنة المكتملة التسليم المباشر إلى القناة أولًا. الافتراضي: `false`
  (مسار wake/model-delivery القديم لجلسة الطالب).

</Accordion>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

يتحكم في الجلسات التي يمكن استهدافها بواسطة أدوات الجلسات (`sessions_list` و`sessions_history` و`sessions_send`).

الافتراضي: `tree` (الجلسة الحالية + الجلسات التي أنشأتها، مثل subagents).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

ملاحظات:

- `self`: مفتاح الجلسة الحالية فقط.
- `tree`: الجلسة الحالية + الجلسات التي أنشأتها الجلسة الحالية (subagents).
- `agent`: أي جلسة تخص معرّف الوكيل الحالي (وقد يشمل ذلك مستخدمين آخرين إذا كنت تشغل جلسات per-sender تحت معرّف الوكيل نفسه).
- `all`: أي جلسة. ولا يزال الاستهداف عبر الوكلاء يتطلب `tools.agentToAgent`.
- تثبيت sandbox: عندما تكون الجلسة الحالية داخل sandbox وتكون `agents.defaults.sandbox.sessionToolsVisibility="spawned"`، تُفرض الرؤية على `tree` حتى لو كانت `tools.sessions.visibility="all"`.

### `tools.sessions_spawn`

يتحكم في دعم المرفقات المضمنة لـ `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // تفعيل اختياري: اضبط true للسماح بمرفقات الملفات المضمنة
        maxTotalBytes: 5242880, // 5 MB إجماليًا عبر كل الملفات
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB لكل ملف
        retainOnSessionKeep: false, // الاحتفاظ بالمرفقات عندما cleanup="keep"
      },
    },
  },
}
```

ملاحظات:

- المرفقات مدعومة فقط لـ `runtime: "subagent"`. ويرفض ACP runtime هذه المرفقات.
- تُنشأ الملفات داخل مساحة عمل الطفل في `.openclaw/attachments/<uuid>/` مع `.manifest.json`.
- يتم إخفاء محتوى المرفقات تلقائيًا من تخزين transcript.
- يتم التحقق من إدخالات Base64 باستخدام فحوص صارمة للأبجدية/الحشو مع حاجز حجم قبل فك الترميز.
- تكون أذونات الملفات `0700` للأدلة و`0600` للملفات.
- يتبع التنظيف سياسة `cleanup`: يزيل `delete` المرفقات دائمًا؛ بينما يحتفظ `keep` بها فقط عندما تكون `retainOnSessionKeep: true`.

### `tools.experimental`

أعلام أدوات مدمجة تجريبية. تكون معطّلة افتراضيًا ما لم تنطبق قاعدة تفعيل تلقائي خاصة بوقت التشغيل.

```json5
{
  tools: {
    experimental: {
      planTool: true, // تفعيل update_plan التجريبية
    },
  },
}
```

ملاحظات:

- `planTool`: يفعّل أداة `update_plan` المنظمة لتتبع العمل متعدد الخطوات غير التافه.
- الافتراضي: `false` للموفرين غير OpenAI. وتقوم تشغيلات OpenAI وOpenAI Codex بتفعيلها تلقائيًا.
- عند التفعيل، تضيف system prompt أيضًا إرشادات استخدام حتى يستخدمها النموذج فقط في الأعمال الجوهرية ويحافظ على خطوة واحدة فقط في حالة `in_progress`.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: النموذج الافتراضي للوكلاء الفرعيين الذين يتم إنشاؤهم. وإذا حُذف، يرث الوكلاء الفرعيون نموذج المستدعي.
- `allowAgents`: قائمة السماح الافتراضية لمعرّفات الوكلاء الهدف لـ `sessions_spawn` عندما لا يضبط الوكيل الطالب `subagents.allowAgents` الخاص به (`["*"]` = أي وكيل؛ الافتراضي: الوكيل نفسه فقط).
- `runTimeoutSeconds`: المهلة الافتراضية (بالثواني) لـ `sessions_spawn` عندما تحذف استدعاء الأداة قيمة `runTimeoutSeconds`. وتعني `0` عدم وجود مهلة.
- سياسة الأدوات لكل وكيل فرعي: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## الموفرون المخصصون وعناوين URL الأساسية

يستخدم OpenClaw فهرس النماذج المدمج. أضف موفرين مخصصين عبر `models.providers` في الإعدادات أو `~/.openclaw/agents/<agentId>/agent/models.json`.

```json5
{
  models: {
    mode: "merge", // merge (الافتراضي) | replace
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

- استخدم `authHeader: true` + `headers` لاحتياجات المصادقة المخصصة.
- تجاوز جذر إعدادات الوكيل عبر `OPENCLAW_AGENT_DIR` (أو `PI_CODING_AGENT_DIR`، وهو اسم بيئي قديم بديل).
- أسبقية الدمج لمعرّفات الموفر المطابقة:
  - تفوز قيم `baseUrl` غير الفارغة في `models.json` الخاص بالوكيل.
  - تفوز قيم `apiKey` غير الفارغة للوكيل فقط عندما لا يكون ذلك الموفر مُدارًا بواسطة SecretRef في سياق الإعدادات/ملف التعريف الحالي.
  - يتم تحديث قيم `apiKey` الخاصة بالموفر المُدار بواسطة SecretRef من علامات المصدر (`ENV_VAR_NAME` لمراجع البيئة، و`secretref-managed` لمراجع الملف/exec) بدلًا من حفظ الأسرار المحلولة.
  - يتم تحديث قيم header للموفر المُدار بواسطة SecretRef من علامات المصدر (`secretref-env:ENV_VAR_NAME` لمراجع البيئة، و`secretref-managed` لمراجع الملف/exec).
  - تعود قيم `apiKey`/`baseUrl` الفارغة أو المحذوفة في الوكيل إلى `models.providers` في الإعدادات.
  - تستخدم قيم `contextWindow`/`maxTokens` للنموذج المطابق القيمة الأعلى بين الإعداد الصريح وقيم الفهرس الضمنية.
  - تحافظ `contextTokens` للنموذج المطابق على حد وقت تشغيل صريح عندما يكون موجودًا؛ استخدمها لتقييد السياق الفعلي من دون تغيير بيانات النموذج الأصلية.
  - استخدم `models.mode: "replace"` عندما تريد أن تعيد الإعدادات كتابة `models.json` بالكامل.
  - يعتمد حفظ العلامات على المصدر باعتباره المرجع: تُكتب العلامات من لقطة إعدادات المصدر النشطة (قبل الحل)، وليس من قيم الأسرار المحلولة في وقت التشغيل.

### تفاصيل حقول الموفر

- `models.mode`: سلوك فهرس الموفر (`merge` أو `replace`).
- `models.providers`: خريطة موفرين مخصصة مفاتيحها هي معرّفات الموفر.
- `models.providers.*.api`: محول الطلب (`openai-completions` أو `openai-responses` أو `anthropic-messages` أو `google-generative-ai` وما إلى ذلك).
- `models.providers.*.apiKey`: بيانات اعتماد الموفر (يفضل SecretRef/الاستبدال من البيئة).
- `models.providers.*.auth`: إستراتيجية المصادقة (`api-key` أو `token` أو `oauth` أو `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: بالنسبة إلى Ollama + `openai-completions`، يحقن `options.num_ctx` في الطلبات (الافتراضي: `true`).
- `models.providers.*.authHeader`: فرض نقل بيانات الاعتماد في ترويسة `Authorization` عند الحاجة.
- `models.providers.*.baseUrl`: عنوان URL الأساسي لـ API العلوية.
- `models.providers.*.headers`: ترويسات ثابتة إضافية للتوجيه عبر proxy/tenant.
- `models.providers.*.request`: تجاوزات النقل لطلبات HTTP الخاصة بموفر النموذج.
  - `request.headers`: ترويسات إضافية (تُدمج مع القيم الافتراضية للموفر). وتقبل القيم SecretRef.
  - `request.auth`: تجاوز لإستراتيجية المصادقة. الأوضاع: `"provider-default"` (استخدام المصادقة المدمجة للموفر)، و`"authorization-bearer"` (مع `token`)، و`"header"` (مع `headerName` و`value` و`prefix` الاختياري).
  - `request.proxy`: تجاوز لوكيل HTTP. الأوضاع: `"env-proxy"` (استخدام متغيرات البيئة `HTTP_PROXY`/`HTTPS_PROXY`)، و`"explicit-proxy"` (مع `url`). ويقبل كلا الوضعين كائن `tls` اختياريًا.
  - `request.tls`: تجاوز TLS للاتصالات المباشرة. الحقول: `ca` و`cert` و`key` و`passphrase` (كلها تقبل SecretRef)، و`serverName`، و`insecureSkipVerify`.
- `models.providers.*.models`: إدخالات فهرس نماذج صريحة للموفر.
- `models.providers.*.models.*.contextWindow`: بيانات وصفية لنافذة السياق الأصلية للنموذج.
- `models.providers.*.models.*.contextTokens`: حد سياق اختياري لوقت التشغيل. استخدمه عندما تريد ميزانية سياق فعلية أصغر من `contextWindow` الأصلية للنموذج.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: تلميح توافق اختياري. بالنسبة إلى `api: "openai-completions"` مع `baseUrl` غير فارغ وغير أصلي (مستضيف ليس `api.openai.com`)، يفرض OpenClaw هذه القيمة إلى `false` وقت التشغيل. وتحافظ `baseUrl` الفارغة/المحذوفة على سلوك OpenAI الافتراضي.
- `plugins.entries.amazon-bedrock.config.discovery`: جذر إعدادات الاكتشاف التلقائي لـ Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: تشغيل/إيقاف الاكتشاف الضمني.
- `plugins.entries.amazon-bedrock.config.discovery.region`: منطقة AWS الخاصة بالاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: مرشح اختياري لمعرّف الموفر من أجل اكتشاف موجَّه.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: الفاصل الزمني للاستطلاع من أجل تحديث الاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: نافذة السياق الاحتياطية للنماذج المكتشفة.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: الحد الأقصى الاحتياطي لرموز الإخراج للنماذج المكتشفة.

### أمثلة على الموفرين

<Accordion title="Cerebras (GLM 4.6 / 4.7)">

```json5
{
  env: { CEREBRAS_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: {
        primary: "cerebras/zai-glm-4.7",
        fallbacks: ["cerebras/zai-glm-4.6"],
      },
      models: {
        "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
        "cerebras/zai-glm-4.6": { alias: "GLM 4.6 (Cerebras)" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
          { id: "zai-glm-4.6", name: "GLM 4.6 (Cerebras)" },
        ],
      },
    },
  },
}
```

استخدم `cerebras/zai-glm-4.7` لـ Cerebras؛ واستخدم `zai/glm-4.7` لـ Z.AI المباشر.

</Accordion>

<Accordion title="OpenCode">

```json5
{
  agents: {
    defaults: {
      model: { primary: "opencode/claude-opus-4-6" },
      models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
    },
  },
}
```

اضبط `OPENCODE_API_KEY` (أو `OPENCODE_ZEN_API_KEY`). استخدم المراجع `opencode/...` لفهرس Zen أو المراجع `opencode-go/...` لفهرس Go. الاختصار: `openclaw onboard --auth-choice opencode-zen` أو `openclaw onboard --auth-choice opencode-go`.

</Accordion>

<Accordion title="Z.AI (GLM-4.7)">

```json5
{
  agents: {
    defaults: {
      model: { primary: "zai/glm-4.7" },
      models: { "zai/glm-4.7": {} },
    },
  },
}
```

اضبط `ZAI_API_KEY`. وتُقبل `z.ai/*` و`z-ai/*` كأسماء بديلة. الاختصار: `openclaw onboard --auth-choice zai-api-key`.

- نقطة النهاية العامة: `https://api.z.ai/api/paas/v4`
- نقطة نهاية البرمجة (الافتراضي): `https://api.z.ai/api/coding/paas/v4`
- بالنسبة إلى نقطة النهاية العامة، عرّف موفرًا مخصصًا مع تجاوز عنوان URL الأساسي.

</Accordion>

<Accordion title="Moonshot AI (Kimi)">

```json5
{
  env: { MOONSHOT_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "moonshot/kimi-k2.5" },
      models: { "moonshot/kimi-k2.5": { alias: "Kimi K2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "kimi-k2.5",
            name: "Kimi K2.5",
            reasoning: false,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 262144,
          },
        ],
      },
    },
  },
}
```

بالنسبة إلى نقطة نهاية الصين: `baseUrl: "https://api.moonshot.cn/v1"` أو `openclaw onboard --auth-choice moonshot-api-key-cn`.

تعلن نقاط نهاية Moonshot الأصلية عن توافق استخدام البث على النقل المشترك
`openai-completions`، ويعتمد OpenClaw الآن ذلك على قدرات نقطة النهاية
بدلًا من الاعتماد على معرّف الموفر المدمج وحده.

</Accordion>

<Accordion title="Kimi Coding">

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "kimi/kimi-code" },
      models: { "kimi/kimi-code": { alias: "Kimi Code" } },
    },
  },
}
```

متوافق مع Anthropic، وموفر مدمج. الاختصار: `openclaw onboard --auth-choice kimi-code-api-key`.

</Accordion>

<Accordion title="Synthetic (متوافق مع Anthropic)">

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M2.5": { alias: "MiniMax M2.5" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M2.5",
            name: "MiniMax M2.5",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 192000,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

يجب أن يحذف العنوان الأساسي `/v1` (إذ يقوم عميل Anthropic بإضافتها). الاختصار: `openclaw onboard --auth-choice synthetic-api-key`.

</Accordion>

<Accordion title="MiniMax M2.7 (مباشر)">

```json5
{
  agents: {
    defaults: {
      model: { primary: "minimax/MiniMax-M2.7" },
      models: {
        "minimax/MiniMax-M2.7": { alias: "Minimax" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

اضبط `MINIMAX_API_KEY`. الاختصارات:
`openclaw onboard --auth-choice minimax-global-api` أو
`openclaw onboard --auth-choice minimax-cn-api`.
أصبح فهرس النماذج الآن يستخدم M2.7 فقط افتراضيًا.
وعلى مسار البث المتوافق مع Anthropic، يعطّل OpenClaw التفكير في MiniMax
افتراضيًا ما لم تضبط `thinking` صراحة بنفسك. يقوم `/fast on` أو
`params.fastMode: true` بإعادة كتابة `MiniMax-M2.7` إلى
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="النماذج المحلية (LM Studio)">

راجع [النماذج المحلية](/ar/gateway/local-models). الخلاصة: شغّل نموذجًا محليًا كبيرًا عبر LM Studio Responses API على عتاد قوي؛ واحتفظ بالنماذج المستضافة مدمجة لاستخدامها كمسار احتياطي.

</Accordion>

---

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // أو سلسلة نصية عادية
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: قائمة سماح اختيارية لـ Skills المجمعة فقط (ولا تتأثر Skills المُدارة/الخاصة بمساحة العمل).
- `load.extraDirs`: جذور Skills مشتركة إضافية (أدنى أسبقية).
- `install.preferBrew`: عندما تكون true، يفضل مثبّتات Homebrew عند توفر `brew`
  قبل الرجوع إلى أنواع المثبّتات الأخرى.
- `install.nodeManager`: تفضيل مثبّت node لمواصفات `metadata.openclaw.install`
  (`npm` | `pnpm` | `yarn` | `bun`).
- يؤدي `entries.<skillKey>.enabled: false` إلى تعطيل Skill حتى لو كانت مجمعة/مثبّتة.
- `entries.<skillKey>.apiKey`: حقل تسهيلي لمفتاح API على مستوى Skill عندما يعرّف الـ Skill متغير بيئة أساسيًا (سلسلة نصية عادية أو كائن SecretRef).

---

## الإضافات

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-extension"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- يتم التحميل من `~/.openclaw/extensions`، و`<workspace>/.openclaw/extensions`، بالإضافة إلى `plugins.load.paths`.
- يقبل الاكتشاف إضافات OpenClaw الأصلية بالإضافة إلى حِزم Codex المتوافقة وحِزم Claude، بما في ذلك حِزم Claude ذات التخطيط الافتراضي من دون manifest.
- **تتطلب تغييرات الإعدادات إعادة تشغيل البوابة.**
- `allow`: قائمة سماح اختيارية (لا تُحمَّل إلا الإضافات المدرجة). ويفوز `deny`.
- `plugins.entries.<id>.apiKey`: حقل تسهيلي لمفتاح API على مستوى الإضافة (عندما تدعمه الإضافة).
- `plugins.entries.<id>.env`: خريطة متغيرات بيئة ضمن نطاق الإضافة.
- `plugins.entries.<id>.hooks.allowPromptInjection`: عندما تكون `false`، تمنع core الحدث `before_prompt_build` وتتجاهل الحقول التي تطفر prompt من `before_agent_start` القديم، مع الحفاظ على `modelOverride` و`providerOverride` القديمين. وينطبق ذلك على hooks الخاصة بالإضافات الأصلية وعلى أدلة hooks المقدمة من الحِزم المدعومة.
- `plugins.entries.<id>.subagent.allowModelOverride`: الوثوق صراحة بهذه الإضافة لطلب تجاوزات `provider` و`model` لكل تشغيل لعمليات الوكيل الفرعي في الخلفية.
- `plugins.entries.<id>.subagent.allowedModels`: قائمة سماح اختيارية للأهداف القياسية `provider/model` الخاصة بتجاوزات الوكيل الفرعي الموثوقة. استخدم `"*"` فقط عندما تريد عمدًا السماح بأي نموذج.
- `plugins.entries.<id>.config`: كائن إعدادات معرّف من الإضافة (ويتم التحقق منه بواسطة مخطط الإضافة الأصلي لـ OpenClaw عند توفره).
- `plugins.entries.firecrawl.config.webFetch`: إعدادات موفر الجلب عبر الويب Firecrawl.
  - `apiKey`: مفتاح API الخاص بـ Firecrawl (يقبل SecretRef). ويعود إلى `plugins.entries.firecrawl.config.webSearch.apiKey`، أو `tools.web.fetch.firecrawl.apiKey` القديم، أو متغير البيئة `FIRECRAWL_API_KEY`.
  - `baseUrl`: عنوان URL الأساسي لـ API في Firecrawl (الافتراضي: `https://api.firecrawl.dev`).
  - `onlyMainContent`: استخراج المحتوى الرئيسي فقط من الصفحات (الافتراضي: `true`).
  - `maxAgeMs`: أقصى عمر للذاكرة المؤقتة بالمللي ثانية (الافتراضي: `172800000` / يومان).
  - `timeoutSeconds`: مهلة طلب الكشط بالثواني (الافتراضي: `60`).
- `plugins.entries.xai.config.xSearch`: إعدادات xAI X Search ‏(بحث Grok على الويب).
  - `enabled`: تفعيل موفر X Search.
  - `model`: نموذج Grok المستخدم للبحث (مثل `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming`: إعدادات dreaming للذاكرة (تجريبية). راجع [Dreaming](/concepts/dreaming) للمراحل والعتبات.
  - `enabled`: مفتاح dreaming الرئيسي (الافتراضي `false`).
  - `frequency`: وتيرة cron لكل اجتياح كامل لـ dreaming (الافتراضي `"0 3 * * *"`).
  - سياسة المراحل والعتبات هي تفاصيل تنفيذية (وليست مفاتيح إعدادات مرئية للمستخدم).
- يمكن لإضافات حزم Claude المفعّلة أيضًا أن تساهم بقيم Pi افتراضية مضمنة من `settings.json`؛ ويطبقها OpenClaw بوصفها إعدادات وكيل منقّاة، وليس كترقيعات خام لإعدادات OpenClaw.
- `plugins.slots.memory`: اختر معرّف إضافة الذاكرة النشطة، أو `"none"` لتعطيل إضافات الذاكرة.
- `plugins.slots.contextEngine`: اختر معرّف إضافة محرك السياق النشط؛ والقيمة الافتراضية `"legacy"` ما لم تثبّت وتحدد محركًا آخر.
- `plugins.installs`: بيانات تعريف تثبيتات مُدارة من CLI يستخدمها `openclaw plugins update`.
  - تتضمن `source` و`spec` و`sourcePath` و`installPath` و`version` و`resolvedName` و`resolvedVersion` و`resolvedSpec` و`integrity` و`shasum` و`resolvedAt` و`installedAt`.
  - تعامل مع `plugins.installs.*` بوصفها حالة مُدارة؛ وفضّل أوامر CLI على التعديلات اليدوية.

راجع [الإضافات](/ar/tools/plugin).

---

## المتصفح

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // وضع الشبكة الموثوقة الافتراضي
      // allowPrivateNetwork: true, // اسم بديل قديم
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- يؤدي `evaluateEnabled: false` إلى تعطيل `act:evaluate` و`wait --fn`.
- تكون `ssrfPolicy.dangerouslyAllowPrivateNetwork` افتراضيًا `true` عندما لا تُضبط (نموذج الشبكة الموثوق).
- اضبط `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` للملاحة الصارمة في المتصفح على الشبكات العامة فقط.
- في الوضع الصارم، تخضع نقاط نهاية ملفات تعريف CDP البعيدة (`profiles.*.cdpUrl`) للحظر نفسه الخاص بالشبكات الخاصة أثناء فحوصات الوصول/الاكتشاف.
- لا يزال `ssrfPolicy.allowPrivateNetwork` مدعومًا كاسم بديل قديم.
- في الوضع الصارم، استخدم `ssrfPolicy.hostnameAllowlist` و`ssrfPolicy.allowedHostnames` للاستثناءات الصريحة.
- تكون ملفات التعريف البعيدة attach-only (وتُعطل start/stop/reset).
- تقبل `profiles.*.cdpUrl` القيم `http://` و`https://` و`ws://` و`wss://`.
  استخدم HTTP(S) عندما تريد أن يكتشف OpenClaw ‏`/json/version`؛ واستخدم WS(S)
  عندما يزوّدك موفرك بعنوان DevTools WebSocket مباشر.
- تكون ملفات تعريف `existing-session` خاصة بالمستضيف فقط وتستخدم Chrome MCP بدلًا من CDP.
- يمكن لملفات تعريف `existing-session` ضبط `userDataDir` لاستهداف
  ملف تعريف محدد لمتصفح قائم على Chromium مثل Brave أو Edge.
- تحتفظ ملفات تعريف `existing-session` بالحدود الحالية لمسارات Chrome MCP:
  إجراءات قائمة على اللقطات/المراجع بدلًا من استهداف CSS selector، وخطافات رفع ملف واحد،
  وعدم وجود تجاوزات لمهلات مربعات الحوار، وعدم وجود `wait --load networkidle`،
  وعدم وجود `responsebody` أو تصدير PDF أو اعتراض التنزيلات أو إجراءات دفعية.
- تقوم ملفات تعريف `openclaw` المُدارة محليًا بتعيين `cdpPort` و`cdpUrl` تلقائيًا؛ ولا
  تضبط `cdpUrl` صراحة إلا من أجل CDP البعيد.
- ترتيب الاكتشاف التلقائي: المتصفح الافتراضي إذا كان قائمًا على Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- خدمة التحكم: loopback فقط (المنفذ مشتق من `gateway.port`، والافتراضي `18791`).
- يضيف `extraArgs` أعلام تشغيل إضافية إلى بدء Chromium المحلي (مثل
  `--disable-gpu` أو حجم النافذة أو أعلام التصحيح).

---

## واجهة المستخدم

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // رمز تعبيري، نص قصير، عنوان URL لصورة، أو data URI
    },
  },
}
```

- `seamColor`: لون التمييز لواجهات التطبيقات الأصلية (مثل لون فقاعة Talk Mode، إلخ).
- `assistant`: تجاوز هوية Control UI. ويعود إلى هوية الوكيل النشط.

---

## البوابة

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // أو OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // لـ mode=trusted-proxy؛ راجع /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback