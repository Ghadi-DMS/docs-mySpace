---
read_when:
    - تحتاج إلى الدلالات الدقيقة للإعدادات على مستوى الحقول أو إلى القيم الافتراضية
    - أنت تتحقق من كتل إعدادات القناة أو النموذج أو البوابة أو الأدوات
summary: مرجع كامل لكل مفتاح إعدادات في OpenClaw، والقيم الافتراضية، وإعدادات القنوات
title: مرجع الإعدادات
x-i18n:
    generated_at: "2026-04-06T07:23:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6ae6c19666f65433361e1c8b100ae710448c8aa055a60c140241a8aea09b98a5
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# مرجع الإعدادات

كل حقل متاح في `~/.openclaw/openclaw.json`. للحصول على نظرة عامة موجّهة بالمهام، راجع [الإعدادات](/ar/gateway/configuration).

تنسيق الإعدادات هو **JSON5** (يُسمح بالتعليقات والفواصل اللاحقة). جميع الحقول اختيارية — يستخدم OpenClaw قيماً افتراضية آمنة عند حذفها.

---

## القنوات

تبدأ كل قناة تلقائياً عندما يكون قسم إعداداتها موجوداً (ما لم يكن `enabled: false`).

### الوصول إلى الرسائل المباشرة والمجموعات

تدعم جميع القنوات سياسات الرسائل المباشرة وسياسات المجموعات:

| سياسة الرسائل المباشرة | السلوك |
| ---------------------- | ------ |
| `pairing` (الافتراضي) | يحصل المرسلون غير المعروفين على رمز اقتران لمرة واحدة؛ ويجب على المالك الموافقة |
| `allowlist` | فقط المرسلون الموجودون في `allowFrom` (أو مخزن السماح المقترن) |
| `open` | السماح بكل الرسائل المباشرة الواردة (يتطلب `allowFrom: ["*"]`) |
| `disabled` | تجاهل كل الرسائل المباشرة الواردة |

| سياسة المجموعات | السلوك |
| --------------- | ------ |
| `allowlist` (الافتراضي) | فقط المجموعات المطابقة لقائمة السماح المكوّنة |
| `open` | تجاوز قوائم السماح الخاصة بالمجموعات (مع استمرار تطبيق تقييد الإشارات) |
| `disabled` | حظر كل رسائل المجموعات/الغرف |

<Note>
يضبط `channels.defaults.groupPolicy` القيمة الافتراضية عندما لا تكون `groupPolicy` الخاصة بالموفّر معيّنة.
تنتهي صلاحية رموز الاقتران بعد ساعة واحدة. ويُحدَّد الحد الأقصى لطلبات اقتران الرسائل المباشرة المعلّقة بـ **3 لكل قناة**.
إذا كانت كتلة الموفّر مفقودة بالكامل (`channels.<provider>` غير موجودة)، فإن سياسة المجموعات وقت التشغيل تعود إلى `allowlist` (إغلاق افتراضي) مع تحذير عند بدء التشغيل.
</Note>

### تجاوزات نموذج القناة

استخدم `channels.modelByChannel` لتثبيت معرّفات قنوات محددة على نموذج معيّن. تقبل القيم `provider/model` أو الأسماء المستعارة للنماذج المكوّنة. يُطبَّق تعيين القناة عندما لا تكون الجلسة لديها بالفعل تجاوز لنموذج ما (على سبيل المثال، تم ضبطه عبر `/model`).

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

### الإعدادات الافتراضية للقنوات وHeartbeat

استخدم `channels.defaults` لسلوك سياسة المجموعات وHeartbeat المشترك عبر الموفّرين:

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

- `channels.defaults.groupPolicy`: سياسة المجموعات الاحتياطية عندما لا تكون `groupPolicy` على مستوى الموفّر معيّنة.
- `channels.defaults.contextVisibility`: وضع الرؤية الافتراضي للسياق التكميلي لجميع القنوات. القيم: `all` (الافتراضي، تضمين كل سياق الاقتباس/الخيط/السجل)، و`allowlist` (تضمين السياق من المرسلين الموجودين في قائمة السماح فقط)، و`allowlist_quote` (مثل allowlist لكن مع الاحتفاظ بسياق الاقتباس/الرد الصريح). التجاوز لكل قناة: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: تضمين حالات القنوات السليمة في مخرجات Heartbeat.
- `channels.defaults.heartbeat.showAlerts`: تضمين الحالات المتدهورة/حالات الخطأ في مخرجات Heartbeat.
- `channels.defaults.heartbeat.useIndicator`: عرض مخرجات Heartbeat مدمجة بنمط المؤشر.

### WhatsApp

يعمل WhatsApp عبر قناة الويب الخاصة بالبوابة (Baileys Web). ويبدأ تلقائياً عند وجود جلسة مرتبطة.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // blue ticks (false in self-chat mode)
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

- تستخدم الأوامر الصادرة الحساب `default` افتراضياً إذا كان موجوداً؛ وإلا فسيُستخدم أول معرّف حساب مكوَّن (بعد الترتيب).
- يجاوز `channels.whatsapp.defaultAccount` الاختياري هذا الاختيار الافتراضي الاحتياطي للحساب عندما يطابق معرّف حساب مكوَّناً.
- يتم ترحيل دليل مصادقة Baileys القديم أحادي الحساب بواسطة `openclaw doctor` إلى `whatsapp/default`.
- التجاوزات لكل حساب: `channels.whatsapp.accounts.<id>.sendReadReceipts` و`channels.whatsapp.accounts.<id>.dmPolicy` و`channels.whatsapp.accounts.<id>.allowFrom`.

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
      streaming: "partial", // off | partial | block | progress (default: off; opt in explicitly to avoid preview-edit rate limits)
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

- رمز البوت: `channels.telegram.botToken` أو `channels.telegram.tokenFile` (ملف عادي فقط؛ تُرفَض الروابط الرمزية)، مع `TELEGRAM_BOT_TOKEN` كخيار احتياطي للحساب الافتراضي.
- يجاوز `channels.telegram.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّناً.
- في إعدادات الحسابات المتعددة (معرّفا حسابين أو أكثر)، اضبط قيمة افتراضية صريحة (`channels.telegram.defaultAccount` أو `channels.telegram.accounts.default`) لتجنّب التوجيه الاحتياطي؛ ويحذّر `openclaw doctor` عندما تكون هذه القيمة مفقودة أو غير صالحة.
- يمنع `configWrites: false` عمليات الكتابة إلى الإعدادات التي تبدأ من Telegram (ترحيلات معرّفات المجموعات الفائقة، و`/config set|unset`).
- تقوم إدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` بإعداد ارتباطات ACP دائمة لموضوعات المنتدى (استخدم الصيغة القياسية `chatId:topic:topicId` في `match.peer.id`). وتكون دلالات الحقول مشتركة في [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- تستخدم معاينات البث في Telegram `sendMessage` + `editMessageText` (وتعمل في المحادثات المباشرة والجماعية).
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
      streaming: "off", // off | partial | block | progress (progress maps to partial on Discord)
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
        spawnSubagentSessions: false, // opt-in for sessions_spawn({ thread: true })
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

- الرمز: `channels.discord.token`، مع `DISCORD_BOT_TOKEN` كخيار احتياطي للحساب الافتراضي.
- تستخدم الاستدعاءات الصادرة المباشرة التي توفّر `token` صريحاً لـ Discord هذا الرمز للاستدعاء؛ بينما تظل إعدادات إعادة المحاولة/السياسة الخاصة بالحساب مأخوذة من الحساب المحدد في لقطة وقت التشغيل النشطة.
- يجاوز `channels.discord.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّناً.
- استخدم `user:<id>` (رسالة مباشرة) أو `channel:<id>` (قناة guild) كأهداف للتسليم؛ تُرفَض المعرّفات الرقمية العارية.
- تكون slugs الخاصة بـ guild بأحرف صغيرة مع استبدال المسافات بـ `-`؛ وتستخدم مفاتيح القنوات الاسم المحوَّل إلى slug (من دون `#`). ويفضَّل استخدام معرّفات guild.
- يتم تجاهل الرسائل التي أنشأها البوت افتراضياً. يفعّل `allowBots: true` هذه الرسائل؛ واستخدم `allowBots: "mentions"` لقبول رسائل البوت التي تذكر البوت فقط (مع استمرار تصفية الرسائل الخاصة بالبوت نفسه).
- يسقط `channels.discord.guilds.<id>.ignoreOtherMentions` (وتجاوزات القنوات) الرسائل التي تذكر مستخدماً آخر أو دوراً آخر لكن لا تذكر البوت (باستثناء @everyone/@here).
- يقوم `maxLinesPerMessage` (الافتراضي 17) بتقسيم الرسائل الطويلة عمودياً حتى لو كانت أقل من 2000 حرف.
- يتحكم `channels.discord.threadBindings` في التوجيه المرتبط بخيوط Discord:
  - `enabled`: تجاوز Discord لميزات الجلسات المرتبطة بالخيط (`/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age` والتسليم/التوجيه المرتبط)
  - `idleHours`: تجاوز Discord لإلغاء التركيز التلقائي بعد عدم النشاط بالساعات (`0` يعطّله)
  - `maxAgeHours`: تجاوز Discord للحد الأقصى الصلب للعمر بالساعات (`0` يعطّله)
  - `spawnSubagentSessions`: مفتاح اشتراك اختياري لإنشاء/ربط خيوط تلقائياً مع `sessions_spawn({ thread: true })`
- تقوم إدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` بإعداد ارتباطات ACP دائمة للقنوات والخيوط (استخدم معرّف القناة/الخيط في `match.peer.id`). ودلالات الحقول مشتركة في [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- يضبط `channels.discord.ui.components.accentColor` لون التمييز لحاويات Discord components v2.
- يفعّل `channels.discord.voice` محادثات قنوات Discord الصوتية مع إمكانات الانضمام التلقائي وتجاوزات TTS الاختيارية.
- يمرّر `channels.discord.voice.daveEncryption` و`channels.discord.voice.decryptionFailureTolerance` إلى خيارات DAVE في `@discordjs/voice` (القيم الافتراضية `true` و`24`).
- يحاول OpenClaw أيضاً استعادة استقبال الصوت من خلال مغادرة جلسة صوتية ثم إعادة الانضمام إليها بعد تكرار حالات فشل فك التشفير.
- `channels.discord.streaming` هو مفتاح وضع البث القياسي. ويتم ترحيل `streamMode` القديم وقيم `streaming` المنطقية تلقائياً.
- يربط `channels.discord.autoPresence` حالة التوفّر أثناء التشغيل بحضور البوت (سليم => online، متدهور => idle، منهك => dnd) ويسمح بتجاوزات اختيارية لنص الحالة.
- يعيد `channels.discord.dangerouslyAllowNameMatching` تمكين المطابقة بالأسماء/الوسوم القابلة للتغيير (وضع توافق استثنائي).
- `channels.discord.execApprovals`: تسليم موافقات exec الأصلية في Discord وتفويض الموافقين.
  - `enabled`: `true` أو `false` أو `"auto"` (الافتراضي). في الوضع التلقائي، تُفعَّل موافقات exec عندما يمكن حل الموافقين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Discord المسموح لهم بالموافقة على طلبات exec. ويُستخدم `commands.ownerAllowFrom` كاحتياطي عند الحذف.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتمرير الموافقات لجميع الوكلاء.
  - `sessionFilter`: أنماط اختيارية لمفاتيح الجلسات (سلسلة فرعية أو regex).
  - `target`: مكان إرسال مطالبات الموافقة. يرسل `"dm"` (الافتراضي) إلى الرسائل المباشرة للموافقين، ويرسل `"channel"` إلى القناة الأصلية، ويرسل `"both"` إلى كليهما. وعندما يتضمن الهدف `"channel"`، لا يمكن استخدام الأزرار إلا من قبل الموافقين الذين جرى حلّهم.
  - `cleanupAfterResolve`: عند `true`، يحذف رسائل الموافقة المباشرة بعد الموافقة أو الرفض أو انتهاء المهلة.

**أوضاع إشعارات التفاعلات:** `off` (بدون)، و`own` (رسائل البوت، الافتراضي)، و`all` (كل الرسائل)، و`allowlist` (من `guilds.<id>.users` على جميع الرسائل).

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

- JSON حساب الخدمة: مضمّن (`serviceAccount`) أو معتمد على ملف (`serviceAccountFile`).
- كما أن SecretRef لحساب الخدمة مدعوم أيضاً (`serviceAccountRef`).
- الخيارات الاحتياطية من البيئة: `GOOGLE_CHAT_SERVICE_ACCOUNT` أو `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- استخدم `spaces/<spaceId>` أو `users/<userId>` كأهداف للتسليم.
- يعيد `channels.googlechat.dangerouslyAllowNameMatching` تمكين مطابقة عناوين البريد الإلكتروني القابلة للتغيير (وضع توافق استثنائي).

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
      streaming: "partial", // off | partial | block | progress (preview mode)
      nativeStreaming: true, // use Slack native streaming API when streaming=partial
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

- يتطلب **وضع Socket** كلاً من `botToken` و`appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` كاحتياطي من البيئة للحساب الافتراضي).
- يتطلب **وضع HTTP** وجود `botToken` مع `signingSecret` (في الجذر أو لكل حساب).
- تقبل `botToken` و`appToken` و`signingSecret` و`userToken`
  سلاسل نصية صريحة أو كائنات SecretRef.
- تعرض لقطات حساب Slack حقول مصدر/حالة لكل اعتماد مثل
  `botTokenSource` و`botTokenStatus` و`appTokenStatus`، وفي وضع HTTP،
  `signingSecretStatus`. وتعني `configured_unavailable` أن الحساب
  مكوَّن عبر SecretRef لكن مسار الأمر/وقت التشغيل الحالي لم يتمكن
  من حل قيمة السر.
- يمنع `configWrites: false` عمليات الكتابة إلى الإعدادات التي تبدأ من Slack.
- يجاوز `channels.slack.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّناً.
- `channels.slack.streaming` هو مفتاح وضع البث القياسي. ويتم ترحيل `streamMode` القديم وقيم `streaming` المنطقية تلقائياً.
- استخدم `user:<id>` (رسالة مباشرة) أو `channel:<id>` كأهداف للتسليم.

**أوضاع إشعارات التفاعلات:** `off` و`own` (الافتراضي) و`all` و`allowlist` (من `reactionAllowlist`).

**عزل جلسات الخيوط:** يكون `thread.historyScope` لكل خيط على حدة (الافتراضي) أو مشتركاً على مستوى القناة. ويقوم `thread.inheritParent` بنسخ سجل القناة الأصلية إلى الخيوط الجديدة.

- يضيف `typingReaction` تفاعلاً مؤقتاً إلى رسالة Slack الواردة أثناء تشغيل الرد، ثم يزيله عند الاكتمال. استخدم shortcode لرمز تعبيري في Slack مثل `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: تسليم موافقات exec الأصلية في Slack وتفويض الموافقين. نفس مخطط Discord: `enabled` (`true`/`false`/`"auto"`)، و`approvers` (معرّفات مستخدمي Slack)، و`agentFilter`، و`sessionFilter`، و`target` (`"dm"` أو `"channel"` أو `"both"`).

| مجموعة الإجراءات | الافتراضي | ملاحظات |
| ---------------- | --------- | ------- |
| reactions | مفعّل | التفاعل + سرد التفاعلات |
| messages | مفعّل | قراءة/إرسال/تعديل/حذف |
| pins | مفعّل | تثبيت/إلغاء تثبيت/سرد |
| memberInfo | مفعّل | معلومات العضو |
| emojiList | مفعّل | قائمة الرموز التعبيرية المخصصة |

### Mattermost

يُشحن Mattermost كإضافة: `openclaw plugins install @openclaw/mattermost`.

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
        native: true, // opt-in
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Optional explicit URL for reverse-proxy/public deployments
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

أوضاع الدردشة: `oncall` (الرد عند @-mention، وهو الافتراضي)، و`onmessage` (كل رسالة)، و`onchar` (الرسائل التي تبدأ ببادئة تشغيل).

عند تمكين أوامر Mattermost الأصلية:

- يجب أن تكون `commands.callbackPath` مساراً (مثل `/api/channels/mattermost/command`) وليس URL كاملاً.
- يجب أن تحل `commands.callbackUrl` إلى نقطة نهاية بوابة OpenClaw وأن تكون قابلة للوصول من خادم Mattermost.
- تتم مصادقة استدعاءات slash الأصلية باستخدام الرموز المميزة لكل أمر
  التي يعيدها Mattermost أثناء تسجيل أوامر slash. إذا فشل التسجيل أو لم يتم
  تفعيل أي أوامر، فسيرفض OpenClaw الاستدعاءات برسالة
  `Unauthorized: invalid command token.`
- بالنسبة لمضيفي الاستدعاء الخاصين/الداخليين/ضمن tailnet، قد يتطلب Mattermost
  أن يتضمن `ServiceSettings.AllowedUntrustedInternalConnections` المضيف/النطاق الخاص بالاستدعاء.
  استخدم قيم المضيف/النطاق، وليس URL كاملة.
- `channels.mattermost.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي تبدأ من Mattermost.
- `channels.mattermost.requireMention`: طلب `@mention` قبل الرد في القنوات.
- `channels.mattermost.groups.<channelId>.requireMention`: تجاوز تقييد الإشارات لكل قناة (`"*"` للافتراضي).
- يجاوز `channels.mattermost.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّناً.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // optional account binding
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

**أوضاع إشعارات التفاعلات:** `off` و`own` (الافتراضي) و`all` و`allowlist` (من `reactionAllowlist`).

- `channels.signal.account`: يثبت بدء تشغيل القناة على هوية حساب Signal محددة.
- `channels.signal.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي تبدأ من Signal.
- يجاوز `channels.signal.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّناً.

### BlueBubbles

BlueBubbles هو المسار الموصى به لـ iMessage (مدعوم بإضافة، ويُضبط تحت `channels.bluebubbles`).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, group controls, and advanced actions:
      // see /channels/bluebubbles
    },
  },
}
```

- المسارات الأساسية للمفاتيح المغطاة هنا: `channels.bluebubbles` و`channels.bluebubbles.dmPolicy`.
- يجاوز `channels.bluebubbles.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّناً.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات BlueBubbles بجلسات ACP دائمة. استخدم BlueBubbles handle أو سلسلة target (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- إعداد قناة BlueBubbles الكامل موثق في [BlueBubbles](/ar/channels/bluebubbles).

### iMessage

يقوم OpenClaw بتشغيل `imsg rpc` (JSON-RPC عبر stdio). لا حاجة إلى daemon أو منفذ.

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

- يجاوز `channels.imessage.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّناً.

- يتطلب Full Disk Access لقاعدة بيانات Messages.
- يُفضَّل استخدام الأهداف `chat_id:<id>`. استخدم `imsg chats --limit 20` لسرد المحادثات.
- يمكن أن يشير `cliPath` إلى غلاف SSH؛ واضبط `remoteHost` (`host` أو `user@host`) لجلب المرفقات عبر SCP.
- يقيّد `attachmentRoots` و`remoteAttachmentRoots` مسارات المرفقات الواردة (الافتراضي: `/Users/*/Library/Messages/Attachments`).
- يستخدم SCP تحققاً صارماً من مفتاح المضيف، لذا تأكد من أن مفتاح مضيف relay موجود بالفعل في `~/.ssh/known_hosts`.
- `channels.imessage.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي تبدأ من iMessage.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات iMessage بجلسات ACP دائمة. استخدم handle مطبّعاً أو هدف دردشة صريحاً (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).

<Accordion title="مثال على غلاف SSH لـ iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix مدعوم بإضافة ويُضبط تحت `channels.matrix`.

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

- تستخدم مصادقة الرمز `accessToken`؛ وتستخدم مصادقة كلمة المرور `userId` + `password`.
- يوجّه `channels.matrix.proxy` حركة Matrix HTTP عبر وكيل HTTP(S) صريح. ويمكن للحسابات المسمّاة تجاوزه عبر `channels.matrix.accounts.<id>.proxy`.
- يسمح `channels.matrix.allowPrivateNetwork` بخوادم homeserver الخاصة/الداخلية. ويُعد `proxy` و`allowPrivateNetwork` عنصرين مستقلين.
- يحدد `channels.matrix.defaultAccount` الحساب المفضّل في إعدادات الحسابات المتعددة.
- `channels.matrix.execApprovals`: تسليم موافقات exec الأصلية في Matrix وتفويض الموافقين.
  - `enabled`: `true` أو `false` أو `"auto"` (الافتراضي). في الوضع التلقائي، تُفعَّل موافقات exec عندما يمكن حل الموافقين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Matrix (مثل `@owner:example.org`) المسموح لهم بالموافقة على طلبات exec.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتمرير الموافقات لجميع الوكلاء.
  - `sessionFilter`: أنماط اختيارية لمفاتيح الجلسات (سلسلة فرعية أو regex).
  - `target`: مكان إرسال مطالبات الموافقة. `"dm"` (الافتراضي)، أو `"channel"` (الغرفة الأصلية)، أو `"both"`.
  - التجاوزات لكل حساب: `channels.matrix.accounts.<id>.execApprovals`.
- يتحكم `channels.matrix.dm.sessionScope` في كيفية تجميع الرسائل المباشرة في Matrix إلى جلسات: يشارك `per-user` (الافتراضي) بحسب peer الموجَّه، بينما يعزل `per-room` كل غرفة DM.
- تستخدم فحوص الحالة وعمليات البحث المباشر في الدليل نفس سياسة الوكيل المستخدمة لحركة وقت التشغيل.
- تم توثيق إعداد Matrix الكامل، وقواعد الاستهداف، وأمثلة الإعداد في [Matrix](/ar/channels/matrix).

### Microsoft Teams

Microsoft Teams مدعوم بإضافة ويُضبط تحت `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, team/channel policies:
      // see /channels/msteams
    },
  },
}
```

- المسارات الأساسية للمفاتيح المغطاة هنا: `channels.msteams` و`channels.msteams.configWrites`.
- إعداد Teams الكامل (بيانات الاعتماد، وwebhook، وسياسة الرسائل المباشرة/المجموعات، والتجاوزات لكل فريق/قناة) موثّق في [Microsoft Teams](/ar/channels/msteams).

### IRC

IRC مدعوم بإضافة ويُضبط تحت `channels.irc`.

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

- المسارات الأساسية للمفاتيح المغطاة هنا: `channels.irc` و`channels.irc.dmPolicy` و`channels.irc.configWrites` و`channels.irc.nickserv.*`.
- يجاوز `channels.irc.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّناً.
- إعداد قناة IRC الكامل (host/port/TLS/channels/allowlists/mention gating) موثق في [IRC](/ar/channels/irc).

### متعدد الحسابات (كل القنوات)

شغّل عدة حسابات لكل قناة (لكل منها `accountId` خاص بها):

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

- يُستخدم `default` عندما يتم حذف `accountId` (CLI + التوجيه).
- لا تُطبَّق رموز البيئة إلا على الحساب **الافتراضي**.
- تُطبَّق إعدادات القناة الأساسية على جميع الحسابات ما لم يتم تجاوزها لكل حساب.
- استخدم `bindings[].match.accountId` لتوجيه كل حساب إلى وكيل مختلف.
- إذا أضفت حساباً غير افتراضي عبر `openclaw channels add` (أو إعداد القناة) بينما لا تزال على إعداد قناة علوي أحادي الحساب، فإن OpenClaw يرقّي قيم المستوى الأعلى أحادية الحساب ذات النطاق الخاص بالحساب إلى خريطة حسابات القناة أولاً حتى يستمر الحساب الأصلي في العمل. تنقل معظم القنوات هذه القيم إلى `channels.<channel>.accounts.default`؛ ويمكن لـ Matrix بدلاً من ذلك الحفاظ على هدف مسمّى/افتراضي موجود ومطابق.
- تظل الارتباطات الحالية الخاصة بالقناة فقط (من دون `accountId`) مطابقة للحساب الافتراضي؛ وتبقى الارتباطات ذات النطاق الخاص بالحساب اختيارية.
- يقوم `openclaw doctor --fix` أيضاً بإصلاح الأشكال المختلطة من خلال نقل قيم المستوى الأعلى أحادية الحساب ذات النطاق الخاص بالحساب إلى الحساب المُرقّى المختار لتلك القناة. تستخدم معظم القنوات `accounts.default`؛ ويمكن لـ Matrix الحفاظ على هدف مسمّى/افتراضي موجود ومطابق بدلاً من ذلك.

### قنوات الإضافات الأخرى

يتم ضبط العديد من قنوات الإضافات على هيئة `channels.<id>` وتوثيقها في صفحات القنوات المخصصة لها (مثل Feishu وMatrix وLINE وNostr وZalo وNextcloud Talk وSynology Chat وTwitch).
راجع فهرس القنوات الكامل: [القنوات](/ar/channels).

### تقييد الإشارات في دردشات المجموعات

تستخدم رسائل المجموعات افتراضياً **اشتراط الإشارة** (إشارة metadata أو أنماط regex آمنة). وينطبق ذلك على دردشات مجموعات WhatsApp وTelegram وDiscord وGoogle Chat وiMessage.

**أنواع الإشارات:**

- **إشارات metadata**: إشارات @ الأصلية للمنصة. يتم تجاهلها في وضع الدردشة الذاتية في WhatsApp.
- **أنماط النص**: أنماط regex آمنة في `agents.list[].groupChat.mentionPatterns`. يتم تجاهل الأنماط غير الصالحة والتكرارات المتداخلة غير الآمنة.
- يُفرض تقييد الإشارات فقط عندما يكون الاكتشاف ممكناً (إشارات أصلية أو نمط واحد على الأقل).

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

يضبط `messages.groupChat.historyLimit` القيمة الافتراضية العامة. ويمكن للقنوات تجاوزها عبر `channels.<channel>.historyLimit` (أو لكل حساب). اضبطه على `0` للتعطيل.

#### حدود سجل الرسائل المباشرة

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

آلية الحل: تجاوز لكل رسالة مباشرة → القيمة الافتراضية للموفّر → بدون حد (الاحتفاظ بكل شيء).

المدعوم: `telegram` و`whatsapp` و`discord` و`slack` و`signal` و`imessage` و`msteams`.

#### وضع الدردشة الذاتية

أدرج رقمك الخاص في `allowFrom` لتمكين وضع الدردشة الذاتية (يتجاهل إشارات @ الأصلية، ويرد فقط على أنماط النص):

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

### الأوامر (التعامل مع أوامر الدردشة)

```json5
{
  commands: {
    native: "auto", // register native commands when supported
    text: true, // parse /commands in chat messages
    bash: false, // allow ! (alias: /bash)
    bashForegroundMs: 2000,
    config: false, // allow /config
    debug: false, // allow /debug
    restart: false, // allow /restart + gateway restart tool
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
- يفعّل `native: "auto"` الأوامر الأصلية لـ Discord/Telegram، ويترك Slack معطّلاً.
- تجاوز لكل قناة: `channels.discord.commands.native` (قيمة منطقية أو `"auto"`). تؤدي `false` إلى مسح الأوامر المسجّلة سابقاً.
- تضيف `channels.telegram.customCommands` إدخالات إضافية إلى قائمة بوت Telegram.
- يفعّل `bash: true` الأمر `! <cmd>` لصدفة المضيف. ويتطلب `tools.elevated.enabled` وأن يكون المرسل ضمن `tools.elevated.allowFrom.<channel>`.
- يفعّل `config: true` الأمر `/config` (للقراءة/الكتابة في `openclaw.json`). بالنسبة لعملاء `chat.send` في البوابة، تتطلب عمليات الكتابة الدائمة بواسطة `/config set|unset` أيضاً `operator.admin`؛ بينما يبقى `/config show` للقراءة فقط متاحاً للعملاء العاديين ذوي نطاق الكتابة.
- تتحكم `channels.<provider>.configWrites` في عمليات تغيير الإعدادات لكل قناة (الافتراضي: true).
- بالنسبة للقنوات متعددة الحسابات، تتحكم `channels.<provider>.accounts.<id>.configWrites` أيضاً في عمليات الكتابة التي تستهدف ذلك الحساب (مثل `/allowlist --config --account <id>` أو `/config set channels.<provider>.accounts.<id>...`).
- `allowFrom` يكون لكل موفّر. عند تعيينه، يصبح **مصدر التفويض الوحيد** (ويتم تجاهل قوائم السماح/الاقتران الخاصة بالقناة و`useAccessGroups`).
- يسمح `useAccessGroups: false` للأوامر بتجاوز سياسات مجموعات الوصول عندما لا يكون `allowFrom` معيّناً.

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

جذر مستودع اختياري يظهر في سطر Runtime ضمن system prompt. إذا لم يُعيَّن، يكتشفه OpenClaw تلقائياً عبر الصعود من مساحة العمل.

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
      { id: "writer" }, // inherits github, weather
      { id: "docs", skills: ["docs-search"] }, // replaces defaults
      { id: "locked-down", skills: [] }, // no skills
    ],
  },
}
```

- احذف `agents.defaults.skills` للحصول على Skills غير مقيّدة افتراضياً.
- احذف `agents.list[].skills` لكي يرث القيم الافتراضية.
- اضبط `agents.list[].skills: []` لعدم السماح بأي Skills.
- تمثل القائمة غير الفارغة في `agents.list[].skills` المجموعة النهائية لذلك الوكيل؛
  ولا تُدمج مع القيم الافتراضية.

### `agents.defaults.skipBootstrap`

يعطّل الإنشاء التلقائي لملفات bootstrap في مساحة العمل (`AGENTS.md` و`SOUL.md` و`TOOLS.md` و`IDENTITY.md` و`USER.md` و`HEARTBEAT.md` و`BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

يتحكم في توقيت حقن ملفات bootstrap الخاصة بمساحة العمل داخل system prompt. القيمة الافتراضية: `"always"`.

- `"continuation-skip"`: تتخطى أدوار المتابعة الآمنة (بعد رد مساعد مكتمل) إعادة حقن bootstrap لمساحة العمل، مما يقلل حجم prompt. وتستمر تشغيلات Heartbeat ومحاولات إعادة البناء بعد الضغط في إعادة بناء السياق.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

الحد الأقصى لعدد الأحرف لكل ملف bootstrap في مساحة العمل قبل الاقتطاع. الافتراضي: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

الحد الأقصى لإجمالي الأحرف المحقونة عبر جميع ملفات bootstrap في مساحة العمل. الافتراضي: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

يتحكم في نص التحذير المرئي للوكيل عند اقتطاع سياق bootstrap.
الافتراضي: `"once"`.

- `"off"`: لا يحقن نص تحذير في system prompt أبداً.
- `"once"`: يحقن التحذير مرة واحدة لكل توقيع اقتطاع فريد (موصى به).
- `"always"`: يحقن التحذير في كل تشغيل عند وجود اقتطاع.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

الحد الأقصى لحجم البكسل لأطول ضلع للصورة في كتل الصور داخل السجل/الأدوات قبل استدعاءات الموفّر.
الافتراضي: `1200`.

عادةً ما تقلل القيم الأقل من استخدام vision-token وحجم حمولة الطلب في التشغيلات الكثيفة باللقطات.
أما القيم الأعلى فتحافظ على مزيد من التفاصيل المرئية.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

المنطقة الزمنية لسياق system prompt (وليست للطوابع الزمنية للرسائل). وتعود إلى المنطقة الزمنية للمضيف كخيار احتياطي.

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
      params: { cacheRetention: "long" }, // global default provider params
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

- `model`: يقبل إما سلسلة (`"provider/model"`) أو كائناً (`{ primary, fallbacks }`).
  - تضبط صيغة السلسلة النموذج الأساسي فقط.
  - تضبط صيغة الكائن النموذج الأساسي بالإضافة إلى نماذج failover مرتبة.
- `imageModel`: يقبل إما سلسلة (`"provider/model"`) أو كائناً (`{ primary, fallbacks }`).
  - يُستخدم بواسطة مسار أداة `image` باعتباره إعداد نموذج الرؤية.
  - كما يُستخدم كتوجيه احتياطي عندما لا يستطيع النموذج المحدد/الافتراضي قبول مدخلات الصور.
- `imageGenerationModel`: يقبل إما سلسلة (`"provider/model"`) أو كائناً (`{ primary, fallbacks }`).
  - يُستخدم بواسطة إمكانية توليد الصور المشتركة وأي سطح أداة/إضافة مستقبلي يولد الصور.
  - القيم النموذجية: `google/gemini-3.1-flash-image-preview` لتوليد صور Gemini الأصلي، أو `fal/fal-ai/flux/dev` لـ fal، أو `openai/gpt-image-1` لـ OpenAI Images.
  - إذا حددت موفّراً/نموذجاً مباشرة، فاضبط أيضاً مصادقة/مفتاح API المطابق لذلك الموفّر (مثل `GEMINI_API_KEY` أو `GOOGLE_API_KEY` لـ `google/*`، و`OPENAI_API_KEY` لـ `openai/*`، و`FAL_KEY` لـ `fal/*`).
  - إذا حُذف، فلا يزال `image_generate` قادراً على استنتاج موفّر افتراضي مدعوم بالمصادقة. ويحاول أولاً موفّر الإعداد الافتراضي الحالي، ثم بقية موفّري توليد الصور المسجّلين وفق ترتيب معرّف الموفّر.
- `musicGenerationModel`: يقبل إما سلسلة (`"provider/model"`) أو كائناً (`{ primary, fallbacks }`).
  - يُستخدم بواسطة إمكانية توليد الموسيقى المشتركة والأداة المضمنة `music_generate`.
  - القيم النموذجية: `google/lyria-3-clip-preview` أو `google/lyria-3-pro-preview` أو `minimax/music-2.5+`.
  - إذا حُذف، فلا يزال `music_generate` قادراً على استنتاج موفّر افتراضي مدعوم بالمصادقة. ويحاول أولاً موفّر الإعداد الافتراضي الحالي، ثم بقية موفّري توليد الموسيقى المسجّلين وفق ترتيب معرّف الموفّر.
  - إذا حددت موفّراً/نموذجاً مباشرة، فاضبط أيضاً مصادقة/مفتاح API المطابق لذلك الموفّر.
- `videoGenerationModel`: يقبل إما سلسلة (`"provider/model"`) أو كائناً (`{ primary, fallbacks }`).
  - يُستخدم بواسطة إمكانية توليد الفيديو المشتركة والأداة المضمنة `video_generate`.
  - القيم النموذجية: `qwen/wan2.6-t2v` أو `qwen/wan2.6-i2v` أو `qwen/wan2.6-r2v` أو `qwen/wan2.6-r2v-flash` أو `qwen/wan2.7-r2v`.
  - إذا حُذف، فلا يزال `video_generate` قادراً على استنتاج موفّر افتراضي مدعوم بالمصادقة. ويحاول أولاً موفّر الإعداد الافتراضي الحالي، ثم بقية موفّري توليد الفيديو المسجّلين وفق ترتيب معرّف الموفّر.
  - إذا حددت موفّراً/نموذجاً مباشرة، فاضبط أيضاً مصادقة/مفتاح API المطابق لذلك الموفّر.
  - يدعم موفّر توليد الفيديو المضمن لـ Qwen حالياً حتى 1 فيديو خرج، و1 صورة دخل، و4 فيديوهات دخل، ومدة 10 ثوانٍ، وخيارات `size` و`aspectRatio` و`resolution` و`audio` و`watermark` على مستوى الموفّر.
- `pdfModel`: يقبل إما سلسلة (`"provider/model"`) أو كائناً (`{ primary, fallbacks }`).
  - يُستخدم بواسطة أداة `pdf` لتوجيه النماذج.
  - إذا حُذف، تعود أداة PDF إلى `imageModel`، ثم إلى النموذج المحلول للجلسة/الافتراضي.
- `pdfMaxBytesMb`: حد حجم PDF الافتراضي لأداة `pdf` عندما لا يتم تمرير `maxBytesMb` وقت الاستدعاء.
- `pdfMaxPages`: الحد الأقصى الافتراضي للصفحات التي تؤخذ في الاعتبار بواسطة وضع الاستخراج الاحتياطي في أداة `pdf`.
- `verboseDefault`: مستوى verbose الافتراضي للوكلاء. القيم: `"off"` و`"on"` و`"full"`. الافتراضي: `"off"`.
- `elevatedDefault`: مستوى المخرجات المرتفعة الافتراضي للوكلاء. القيم: `"off"` و`"on"` و`"ask"` و`"full"`. الافتراضي: `"on"`.
- `model.primary`: الصيغة `provider/model` (مثل `openai/gpt-5.4`). إذا حذفت الموفّر، يحاول OpenClaw أولاً الاسم المستعار، ثم مطابقة فريدة لموفّر مكوَّن لذلك المعرّف الدقيق للنموذج، وبعدها فقط يعود إلى الموفّر الافتراضي المكوَّن (سلوك توافق قديم؛ لذا يُفضَّل استخدام `provider/model` الصريح). وإذا لم يعد ذلك الموفّر يوفّر النموذج الافتراضي المكوَّن، يعود OpenClaw إلى أول موفّر/نموذج مكوَّن بدلاً من إظهار إعداد افتراضي قديم لموفّر محذوف.
- `models`: فهرس النماذج المكوَّن وقائمة السماح لـ `/model`. يمكن أن يتضمن كل إدخال `alias` (اختصار) و`params` (خاصة بالموفّر، مثل `temperature` و`maxTokens` و`cacheRetention` و`context1m`).
- `params`: معاملات الموفّر الافتراضية العامة المطبّقة على كل النماذج. تُضبط عند `agents.defaults.params` (مثلاً `{ cacheRetention: "long" }`).
- أولوية دمج `params` (في الإعدادات): يتم تجاوز `agents.defaults.params` (الأساس العام) بواسطة `agents.defaults.models["provider/model"].params` (لكل نموذج)، ثم تتجاوز `agents.list[].params` (المطابقة لمعرّف الوكيل) المفاتيح على حدة. راجع [Prompt Caching](/ar/reference/prompt-caching) للتفاصيل.
- تقوم أدوات كتابة الإعدادات التي تغيّر هذه الحقول (مثل `/models set` و`/models set-image` وأوامر إضافة/إزالة fallback) بحفظ الصيغة القياسية للكائن والحفاظ على قوائم fallback الموجودة عندما يكون ذلك ممكناً.
- `maxConcurrent`: الحد الأقصى لتشغيلات الوكلاء المتوازية عبر الجلسات (مع بقاء كل جلسة متسلسلة). الافتراضي: 4.

**اختصارات الأسماء المستعارة المضمنة** (تُطبَّق فقط عندما يكون النموذج موجوداً في `agents.defaults.models`):

| الاسم المستعار | النموذج |
| -------------- | ------- |
| `opus` | `anthropic/claude-opus-4-6` |
| `sonnet` | `anthropic/claude-sonnet-4-6` |
| `gpt` | `openai/gpt-5.4` |
| `gpt-mini` | `openai/gpt-5.4-mini` |
| `gpt-nano` | `openai/gpt-5.4-nano` |
| `gemini` | `google/gemini-3.1-pro-preview` |
| `gemini-flash` | `google/gemini-3-flash-preview` |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

تتغلب الأسماء المستعارة التي تكوّنها أنت دائماً على الافتراضيات.

تفعّل نماذج Z.AI GLM-4.x وضع التفكير تلقائياً ما لم تضبط `--thinking off` أو تعرّف `agents.defaults.models["zai/<model>"].params.thinking` بنفسك.
تفعّل نماذج Z.AI `tool_stream` افتراضياً لبث استدعاءات الأدوات. اضبط `agents.defaults.models["zai/<model>"].params.tool_stream` على `false` لتعطيله.
تستخدم نماذج Anthropic Claude 4.6 وضع التفكير `adaptive` افتراضياً عندما لا يتم تعيين مستوى تفكير صريح.

- تكون الجلسات مدعومة عندما يتم تعيين `sessionArg`.
- يكون تمرير الصور مدعوماً عندما يقبل `imageArg` مسارات الملفات.

### `agents.defaults.heartbeat`

تشغيلات Heartbeat دورية.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m disables
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        lightContext: false, // default: false; true keeps only HEARTBEAT.md from workspace bootstrap files
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (default) | block
        target: "none", // default: none | options: last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
      },
    },
  },
}
```

- `every`: سلسلة مدة (ms/s/m/h). الافتراضي: `30m` (مصادقة بمفتاح API) أو `1h` (مصادقة OAuth). اضبطه على `0m` للتعطيل.
- `suppressToolErrorWarnings`: عند true، يمنع حمولات تحذير أخطاء الأدوات أثناء تشغيلات Heartbeat.
- `directPolicy`: سياسة التسليم المباشر/DM. تسمح `allow` (الافتراضي) بالتسليم المباشر إلى الهدف. وتمنع `block` التسليم المباشر إلى الهدف وتنتج `reason=dm-blocked`.
- `lightContext`: عند true، تستخدم تشغيلات Heartbeat سياق bootstrap خفيف الوزن وتحتفظ فقط بـ `HEARTBEAT.md` من ملفات bootstrap في مساحة العمل.
- `isolatedSession`: عند true، يعمل كل Heartbeat في جلسة جديدة من دون سجل محادثات سابق. وهو نفس نمط العزل الخاص بـ cron `sessionTarget: "isolated"`. ويقلل تكلفة الرموز لكل Heartbeat من نحو 100K إلى نحو 2-5K رمز.
- لكل وكيل: اضبط `agents.list[].heartbeat`. عندما يعرّف أي وكيل `heartbeat`، **لا تعمل إلا هؤلاء الوكلاء** عبر Heartbeat.
- تشغّل Heartbeats أدوار وكيل كاملة — الفواصل الأقصر تستهلك مزيداً من الرموز.

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
        identifierInstructions: "Preserve deployment IDs, ticket IDs, and host:port pairs exactly.", // used when identifierPolicy=custom
        postCompactionSections: ["Session Startup", "Red Lines"], // [] disables reinjection
        model: "openrouter/anthropic/claude-sonnet-4-6", // optional compaction-only model override
        notifyUser: true, // send a brief notice when compaction starts (default: false)
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

- `mode`: `default` أو `safeguard` (تلخيص مجزّأ للتواريخ الطويلة). راجع [Compaction](/ar/concepts/compaction).
- `timeoutSeconds`: الحد الأقصى بالثواني المسموح لعملية compaction واحدة قبل أن يلغيها OpenClaw. الافتراضي: `900`.
- `identifierPolicy`: `strict` (الافتراضي)، أو `off`، أو `custom`. تضيف `strict` إرشاداً مضمّناً للاحتفاظ بالمعرّفات المعتمة أثناء تلخيص compaction.
- `identifierInstructions`: نص اختياري مخصص للحفاظ على المعرّفات يُستخدم عندما `identifierPolicy=custom`.
- `postCompactionSections`: أسماء أقسام H2/H3 اختيارية من `AGENTS.md` لإعادة حقنها بعد compaction. الافتراضي `["Session Startup", "Red Lines"]`؛ اضبط `[]` لتعطيل إعادة الحقن. وعندما يكون غير معيّن أو معيّناً صراحةً على هذا الزوج الافتراضي، تُقبل أيضاً عناوين `Every Session`/`Safety` القديمة كخيار احتياطي متوافق.
- `model`: تجاوز اختياري `provider/model-id` خاص بتلخيص compaction فقط. استخدمه عندما يجب أن تبقى الجلسة الرئيسية على نموذج واحد بينما تعمل ملخصات compaction على نموذج آخر؛ وعند حذفه، تستخدم compaction النموذج الأساسي للجلسة.
- `notifyUser`: عند `true`، يرسل إشعاراً موجزاً إلى المستخدم عند بدء compaction (مثل "Compacting context..."). وهو معطّل افتراضياً لإبقاء compaction صامتاً.
- `memoryFlush`: دور وكيل صامت قبل الضغط التلقائي لتخزين الذكريات الدائمة. ويتم تخطيه عندما تكون مساحة العمل للقراءة فقط.

### `agents.defaults.contextPruning`

يقتطع **نتائج الأدوات القديمة** من السياق داخل الذاكرة قبل إرسالها إلى LLM. ولا **يعدّل** سجل الجلسة على القرص.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // duration (ms/s/m/h), default unit: minutes
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
- يتحكم `ttl` في عدد المرات التي يمكن بعدها أن يعمل الاقتطاع مجدداً (بعد آخر لمس للذاكرة المؤقتة).
- يقوم الاقتطاع أولاً باقتصاص نتائج الأدوات الضخمة جزئياً، ثم يمسح نتائج الأدوات الأقدم بالكامل إذا لزم الأمر.

**الاقتطاع الجزئي** يحتفظ بالبداية والنهاية ويُدرج `...` في الوسط.

**المسح الكامل** يستبدل نتيجة الأداة كاملةً بالنص النائب.

ملاحظات:

- لا يتم اقتطاع/مسح كتل الصور أبداً.
- النِّسب قائمة على عدد الأحرف (تقريبية)، وليست على عدد الرموز بدقة.
- إذا وُجد عدد أقل من `keepLastAssistants` من رسائل المساعد، يتم تخطي الاقتطاع.

</Accordion>

راجع [اقتطاع الجلسات](/ar/concepts/session-pruning) للحصول على تفاصيل السلوك.

### بث الكتل

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom (use minMs/maxMs)
    },
  },
}
```

- تتطلب القنوات غير Telegram تعيين `*.blockStreaming: true` صراحةً لتمكين الردود على شكل كتل.
- تجاوزات القنوات: `channels.<channel>.blockStreamingCoalesce` (ومتغيرات كل حساب). تستخدم Signal/Slack/Discord/Google Chat افتراضياً `minChars: 1500`.
- `humanDelay`: توقف عشوائي بين ردود الكتل. تعني `natural` = 800–2500ms. التجاوز لكل وكيل: `agents.list[].humanDelay`.

راجع [البث](/ar/concepts/streaming) لمعرفة السلوك وتفاصيل التقسيم.

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
- التجاوزات لكل جلسة: `session.typingMode` و`session.typingIntervalSeconds`.

راجع [مؤشرات الكتابة](/ar/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

وضع sandbox اختياري للوكيل المضمن. راجع [Sandboxing](/ar/gateway/sandboxing) للحصول على الدليل الكامل.

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

- `docker`: بيئة Docker محلية (الافتراضية)
- `ssh`: بيئة عامة بعيدة عبر SSH
- `openshell`: بيئة OpenShell

عند اختيار `backend: "openshell"`، تنتقل الإعدادات الخاصة بوقت التشغيل إلى
`plugins.entries.openshell.config`.

**إعدادات واجهة SSH الخلفية:**

- `target`: هدف SSH بصيغة `user@host[:port]`
- `command`: أمر عميل SSH (الافتراضي: `ssh`)
- `workspaceRoot`: جذر بعيد مطلق يُستخدم لمساحات العمل حسب النطاق
- `identityFile` / `certificateFile` / `knownHostsFile`: ملفات محلية موجودة تُمرَّر إلى OpenSSH
- `identityData` / `certificateData` / `knownHostsData`: محتويات مضمّنة أو SecretRefs يقوم OpenClaw بتحويلها إلى ملفات مؤقتة وقت التشغيل
- `strictHostKeyChecking` / `updateHostKeys`: مقابض سياسة مفاتيح المضيف في OpenSSH

**أولوية مصادقة SSH:**

- تتغلب `identityData` على `identityFile`
- تتغلب `certificateData` على `certificateFile`
- تتغلب `knownHostsData` على `knownHostsFile`
- يتم حل قيم `*Data` المدعومة بـ SecretRef من لقطة secrets لوقت التشغيل النشطة قبل بدء جلسة sandbox

**سلوك واجهة SSH الخلفية:**

- يزرع مساحة العمل البعيدة مرة واحدة بعد الإنشاء أو إعادة الإنشاء
- ثم يحتفظ بمساحة العمل البعيدة عبر SSH كمرجع أساسي
- يوجّه `exec` وأدوات الملفات ومسارات الوسائط عبر SSH
- لا يزامن التغييرات البعيدة مرة أخرى إلى المضيف تلقائياً
- لا يدعم حاويات متصفح sandbox

**وصول مساحة العمل:**

- `none`: مساحة عمل sandbox بحسب النطاق تحت `~/.openclaw/sandboxes`
- `ro`: مساحة عمل sandbox عند `/workspace`، مع تركيب مساحة عمل الوكيل للقراءة فقط عند `/agent`
- `rw`: تركيب مساحة عمل الوكيل للقراءة/الكتابة عند `/workspace`

**النطاق:**

- `session`: حاوية + مساحة عمل لكل جلسة
- `agent`: حاوية + مساحة عمل واحدة لكل وكيل (الافتراضي)
- `shared`: حاوية ومساحة عمل مشتركتان (من دون عزل بين الجلسات)

**إعداد إضافة OpenShell:**

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
          gateway: "lab", // optional
          gatewayEndpoint: "https://lab.example", // optional
          policy: "strict", // optional OpenShell policy id
          providers: ["openai"], // optional
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**وضع OpenShell:**

- `mirror`: يزرع البعيد من المحلي قبل exec، ويزامن العودة بعد exec؛ وتبقى مساحة العمل المحلية هي المرجع الأساسي
- `remote`: يزرع البعيد مرة واحدة عند إنشاء sandbox، ثم يحتفظ بمساحة العمل البعيدة كمرجع أساسي

في وضع `remote`، لا تُزامَن تعديلات المضيف المحلية التي تُجرى خارج OpenClaw تلقائياً إلى sandbox بعد خطوة الزرع.
يتم النقل عبر SSH إلى OpenShell sandbox، لكن الإضافة تملك دورة حياة sandbox ومزامنة mirror الاختيارية.

**`setupCommand`** يعمل مرة واحدة بعد إنشاء الحاوية (عبر `sh -lc`). ويتطلب خروجاً للشبكة، وجذراً قابلاً للكتابة، ومستخدم root.

**تستخدم الحاويات افتراضياً `network: "none"`** — اضبطها على `"bridge"` (أو شبكة bridge مخصصة) إذا احتاج الوكيل إلى وصول خارجي.
يتم حظر `"host"`. كما يتم حظر `"container:<id>"` افتراضياً ما لم تضبط صراحةً
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` (وضع استثنائي).

**تُجهَّز المرفقات الواردة** داخل `media/inbound/*` في مساحة العمل النشطة.

**`docker.binds`** يركب أدلة مضيف إضافية؛ ويتم دمج binds العامة ومع تلك الخاصة بكل وكيل.

**متصفح sandbox** (`sandbox.browser.enabled`): Chromium + CDP داخل حاوية. يتم حقن URL الخاص بـ noVNC داخل system prompt. ولا يتطلب `browser.enabled` في `openclaw.json`.
يستخدم وصول المراقبة عبر noVNC مصادقة VNC افتراضياً ويُصدر OpenClaw URL برمز قصير العمر (بدلاً من كشف كلمة المرور في الرابط المشترك).

- `allowHostControl: false` (الافتراضي) يمنع الجلسات المعزولة من استهداف متصفح المضيف.
- يكون `network` افتراضياً `openclaw-sandbox-browser` (شبكة bridge مخصصة). اضبطه إلى `bridge` فقط عندما تريد صراحةً اتصال bridge عاماً.
- يقيّد `cdpSourceRange` اختيارياً الدخول إلى CDP على حافة الحاوية إلى نطاق CIDR (مثل `172.21.0.1/32`).
- تقوم `sandbox.browser.binds` بتركيب أدلة مضيف إضافية في حاوية متصفح sandbox فقط. وعند تعيينها (بما في ذلك `[]`)، فإنها تستبدل `docker.binds` لحاوية المتصفح.
- تُعرَّف القيم الافتراضية للتشغيل في `scripts/sandbox-browser-entrypoint.sh` ومضبوطة لمضيفي الحاويات:
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
  - `--disable-extensions` (مفعّل افتراضياً)
  - تكون `--disable-3d-apis` و`--disable-software-rasterizer` و`--disable-gpu`
    مفعّلة افتراضياً ويمكن تعطيلها عبر
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` إذا كان استخدام WebGL/3D يتطلب ذلك.
  - يعيد `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` تمكين الإضافات إذا كان سير عملك
    يعتمد عليها.
  - يمكن تغيير `--renderer-process-limit=2` عبر
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`؛ اضبطه إلى `0` لاستخدام
    حد العمليات الافتراضي في Chromium.
  - بالإضافة إلى `--no-sandbox` و`--disable-setuid-sandbox` عندما يكون `noSandbox` مفعّلاً.
  - تمثل القيم الافتراضية خط الأساس لصورة الحاوية؛ استخدم صورة متصفح مخصصة مع
    entrypoint مخصص لتغيير افتراضيات الحاوية.

</Accordion>

يدعم browser sandboxing و`sandbox.docker.binds` حالياً Docker فقط.

ابنِ الصور:

```bash
scripts/sandbox-setup.sh           # main sandbox image
scripts/sandbox-browser-setup.sh   # optional browser image
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
        model: "anthropic/claude-opus-4-6", // or { primary, fallbacks }
        thinkingDefault: "high", // per-agent thinking level override
        reasoningDefault: "on", // per-agent reasoning visibility override
        fastModeDefault: false, // per-agent fast mode override
        params: { cacheRetention: "none" }, // overrides matching defaults.models params by key
        skills: ["docs-search"], // replaces agents.defaults.skills when set
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
- `default`: عند تعيين عدة قيم، يفوز الأول (مع تسجيل تحذير). وإذا لم تُعيَّن أي قيمة، يكون أول إدخال في القائمة هو الافتراضي.
- `model`: تتجاوز صيغة السلسلة قيمة `primary` فقط؛ بينما تتجاوز صيغة الكائن `{ primary, fallbacks }` كليهما (`[]` يعطّل fallback العامة). تظل وظائف cron التي تتجاوز `primary` فقط ترث fallback الافتراضية ما لم تضبط `fallbacks: []`.
- `params`: معاملات بث لكل وكيل تُدمج فوق إدخال النموذج المحدد في `agents.defaults.models`. استخدم ذلك لتجاوزات خاصة بالوكيل مثل `cacheRetention` أو `temperature` أو `maxTokens` من دون تكرار فهرس النماذج بالكامل.
- `skills`: قائمة سماح اختيارية لـ Skills لكل وكيل. إذا حُذفت، يرث الوكيل `agents.defaults.skills` عند تعيينها؛ وتستبدل القائمة الصريحة القيم الافتراضية بدلاً من الدمج، و`[]` تعني عدم وجود Skills.
- `thinkingDefault`: مستوى التفكير الافتراضي الاختياري لكل وكيل (`off | minimal | low | medium | high | xhigh | adaptive`). ويتجاوز `agents.defaults.thinkingDefault` لهذا الوكيل عندما لا يكون هناك تجاوز لكل رسالة أو جلسة.
- `reasoningDefault`: التجاوز الاختياري لكل وكيل لرؤية reasoning (`on | off | stream`). ويُطبَّق عندما لا يكون هناك تجاوز للـ reasoning لكل رسالة أو جلسة.
- `fastModeDefault`: القيمة الافتراضية الاختيارية لكل وكيل لوضع fast (`true | false`). وتُطبَّق عندما لا يكون هناك تجاوز لوضع fast لكل رسالة أو جلسة.
- `runtime`: واصف runtime اختياري لكل وكيل. استخدم `type: "acp"` مع القيم الافتراضية لـ `runtime.acp` (`agent` و`backend` و`mode` و`cwd`) عندما يجب أن يستخدم الوكيل جلسات ACP harness افتراضياً.
- `identity.avatar`: مسار نسبي إلى مساحة العمل، أو URL من نوع `http(s)`، أو URI من نوع `data:`.
- تستنتج `identity` القيم الافتراضية: `ackReaction` من `emoji`، و`mentionPatterns` من `name`/`emoji`.
- `subagents.allowAgents`: قائمة سماح لمعرّفات الوكلاء لـ `sessions_spawn` (`["*"]` = أي وكيل؛ الافتراضي: نفس الوكيل فقط).
- حاجز وراثة sandbox: إذا كانت جلسة الطالب معزولة، فإن `sessions_spawn` يرفض الأهداف التي ستعمل من دون sandbox.
- `subagents.requireAgentId`: عندما تكون true، يحظر استدعاءات `sessions_spawn` التي تحذف `agentId` (يفرض اختيار ملف تعريف صريح؛ الافتراضي: false).

---

## التوجيه متعدد الوكلاء

شغّل عدة وكلاء معزولين داخل بوابة واحدة. راجع [متعدد الوكلاء](/ar/concepts/multi-agent).

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

- `type` (اختياري): تكون `route` للتوجيه العادي (والنوع المفقود يعود افتراضياً إلى route)، و`acp` لارتباطات محادثات ACP الدائمة.
- `match.channel` (مطلوب)
- `match.accountId` (اختياري؛ `*` = أي حساب؛ والمحذوف = الحساب الافتراضي)
- `match.peer` (اختياري؛ `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (اختياري؛ خاص بالقناة)
- `acp` (اختياري؛ فقط لـ `type: "acp"`): `{ mode, label, cwd, backend }`

**ترتيب المطابقة الحتمي:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (مطابقة تامة، من دون peer/guild/team)
5. `match.accountId: "*"` (على مستوى القناة)
6. الوكيل الافتراضي

ضمن كل طبقة، يفوز أول إدخال مطابق في `bindings`.

بالنسبة لإدخالات `type: "acp"`، يحل OpenClaw حسب هوية المحادثة الدقيقة (`match.channel` + الحساب + `match.peer.id`) ولا يستخدم ترتيب طبقات ربط route أعلاه.

### ملفات الوصول لكل وكيل

<Accordion title="وصول كامل (من دون sandbox)">

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

<Accordion title="من دون وصول إلى نظام الملفات (مراسلة فقط)">

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

راجع [Sandbox والأدوات في متعدد الوكلاء](/ar/tools/multi-agent-sandbox-tools) لمعرفة تفاصيل الأولوية.

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
    parentForkMaxTokens: 100000, // skip parent-thread fork above this token count (0 disables)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // duration or false
      maxDiskBytes: "500mb", // optional hard budget
      highWaterBytes: "400mb", // optional cleanup target
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // default inactivity auto-unfocus in hours (`0` disables)
      maxAgeHours: 0, // default hard max age in hours (`0` disables)
    },
    mainKey: "main", // legacy (runtime always uses "main")
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="تفاصيل حقول الجلسة">

- **`scope`**: استراتيجية التجميع الأساسية للجلسات في سياقات دردشات المجموعات.
  - `per-sender` (الافتراضي): يحصل كل مرسل على جلسة معزولة ضمن سياق القناة.
  - `global`: يشترك جميع المشاركين في سياق القناة في جلسة واحدة (استخدمه فقط عندما يكون السياق المشترك مقصوداً).
- **`dmScope`**: كيفية تجميع الرسائل المباشرة.
  - `main`: تشترك كل الرسائل المباشرة في الجلسة الرئيسية.
  - `per-peer`: عزل حسب معرّف المرسل عبر القنوات.
  - `per-channel-peer`: عزل لكل قناة + مرسل (موصى به لصناديق الوارد متعددة المستخدمين).
  - `per-account-channel-peer`: عزل لكل حساب + قناة + مرسل (موصى به لتعدد الحسابات).
- **`identityLinks`**: يربط المعرفات القياسية بأقران مسبوقين باسم الموفّر للمشاركة بين القنوات.
- **`reset`**: سياسة إعادة التعيين الأساسية. تعيد `daily` التعيين عند `atHour` بالتوقيت المحلي؛ وتعيد `idle` التعيين بعد `idleMinutes`. وعند تكوين كلاهما، يفوز ما تنتهي مدته أولاً.
- **`resetByType`**: تجاوزات حسب النوع (`direct` و`group` و`thread`). يُقبل `dm` القديم كاسم مستعار لـ `direct`.
- **`parentForkMaxTokens`**: الحد الأقصى لـ `totalTokens` في الجلسة الأصلية المسموح به عند إنشاء جلسة خيط متفرعة (الافتراضي `100000`).
  - إذا تجاوزت `totalTokens` في الأصل هذه القيمة، يبدأ OpenClaw جلسة خيط جديدة بدلاً من وراثة سجل الجلسة الأصلية.
  - اضبطه على `0` لتعطيل هذا الحاجز والسماح دائماً بتفريع الأصل.
- **`mainKey`**: حقل قديم. يستخدم وقت التشغيل الآن دائماً `"main"` لدلو الدردشة المباشرة الرئيسي.
- **`agentToAgent.maxPingPongTurns`**: الحد الأقصى لأدوار الرد المتبادل بين الوكلاء أثناء تبادلات agent-to-agent (عدد صحيح، المجال: `0`–`5`). تعني `0` تعطيل تسلسل ping-pong.
- **`sendPolicy`**: يطابق حسب `channel` أو `chatType` (`direct|group|channel`، مع الاسم القديم `dm`) أو `keyPrefix` أو `rawKeyPrefix`. وأول deny يفوز.
- **`maintenance`**: عناصر تحكم التنظيف والاحتفاظ لمخزن الجلسات.
  - `mode`: يقوم `warn` بإصدار تحذيرات فقط؛ بينما يطبّق `enforce` التنظيف.
  - `pruneAfter`: حد زمني لعمر الإدخالات القديمة (الافتراضي `30d`).
  - `maxEntries`: الحد الأقصى لعدد الإدخالات في `sessions.json` (الافتراضي `500`).
  - `rotateBytes`: يدوّر `sessions.json` عندما يتجاوز هذا الحجم (الافتراضي `10mb`).
  - `resetArchiveRetention`: فترة الاحتفاظ بأرشيفات السجل `*.reset.<timestamp>`. وتعود افتراضياً إلى `pruneAfter`؛ واضبطها على `false` للتعطيل.
  - `maxDiskBytes`: ميزانية اختيارية لقرص دليل الجلسات. في وضع `warn` تسجل تحذيرات؛ وفي وضع `enforce` تزيل أقدم العناصر/الجلسات أولاً.
  - `highWaterBytes`: هدف اختياري بعد تنظيف الميزانية. يعود افتراضياً إلى `80%` من `maxDiskBytes`.
- **`threadBindings`**: الافتراضيات العامة لميزات الجلسات المرتبطة بالخيوط.
  - `enabled`: مفتاح افتراضي رئيسي (يمكن للموفّرين تجاوزه؛ يستخدم Discord `channels.discord.threadBindings.enabled`)
  - `idleHours`: إلغاء التركيز التلقائي الافتراضي بعد عدم النشاط بالساعات (`0` يعطّله؛ ويمكن للموفّرين تجاوزه)
  - `maxAgeHours`: الحد الأقصى الصلب الافتراضي للعمر بالساعات (`0` يعطّله؛ ويمكن للموفّرين تجاوزه)

</Accordion>

---

## الرسائل

```json5
{
  messages: {
    responsePrefix: "🦞", // or "auto"
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
      debounceMs: 2000, // 0 disables
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

آلية الحل (الأكثر تحديداً يفوز): الحساب → القناة → العام. تعني `""` التعطيل وإيقاف التسلسل. أما `"auto"` فتُشتق على هيئة `[{identity.name}]`.

**متغيرات القالب:**

| المتغير | الوصف | المثال |
| ------- | ------ | ------- |
| `{model}` | الاسم المختصر للنموذج | `claude-opus-4-6` |
| `{modelFull}` | معرّف النموذج الكامل | `anthropic/claude-opus-4-6` |
| `{provider}` | اسم الموفّر | `anthropic` |
| `{thinkingLevel}` | مستوى التفكير الحالي | `high`، `low`، `off` |
| `{identity.name}` | اسم هوية الوكيل | (مثل `"auto"`) |

المتغيرات غير حساسة لحالة الأحرف. ويُعد `{think}` اسماً مستعاراً لـ `{thinkingLevel}`.

### تفاعل التأكيد

- يعود افتراضياً إلى `identity.emoji` للوكيل النشط، وإلا `"👀"`. اضبطه على `""` للتعطيل.
- تجاوزات لكل قناة: `channels.<channel>.ackReaction` و`channels.<channel>.accounts.<id>.ackReaction`.
- ترتيب الحل: الحساب → القناة → `messages.ackReaction` → الاحتياطي من الهوية.
- النطاق: `group-mentions` (الافتراضي)، و`group-all`، و`direct`، و`all`.
- يزيل `removeAckAfterReply` تفاعل التأكيد بعد الرد في Slack وDiscord وTelegram.
- يفعّل `messages.statusReactions.enabled` تفاعلات الحالة المرحلية في Slack وDiscord وTelegram.
  في Slack وDiscord، يحافظ عدم التعيين على بقاء تفاعلات الحالة مفعّلة عندما تكون تفاعلات التأكيد نشطة.
  في Telegram، اضبطه صراحةً على `true` لتمكين تفاعلات الحالة المرحلية.

### إزالة ارتداد الرسائل الواردة

يجمع الرسائل النصية السريعة من المرسل نفسه في دور وكيل واحد. ويتم تفريغ الوسائط/المرفقات فوراً. وتتجاوز أوامر التحكم إزالة الارتداد.

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

- يتحكم `auto` في TTS التلقائي. وتتجاوزه `/tts off|always|inbound|tagged` لكل جلسة.
- يتجاوز `summaryModel` قيمة `agents.defaults.model.primary` للملخص التلقائي.
- يكون `modelOverrides` مفعّلاً افتراضياً؛ ويعود `modelOverrides.allowProvider` افتراضياً إلى `false` (اشتراك اختياري).
- تعود مفاتيح API إلى `ELEVENLABS_API_KEY`/`XI_API_KEY` و`OPENAI_API_KEY`.
- يتجاوز `openai.baseUrl` نقطة نهاية OpenAI TTS. ترتيب الحل هو الإعدادات، ثم `OPENAI_TTS_BASE_URL`، ثم `https://api.openai.com/v1`.
- عندما يشير `openai.baseUrl` إلى نقطة نهاية غير OpenAI، يتعامل OpenClaw معها كخادم TTS متوافق مع OpenAI ويخفف التحقق من النموذج/الصوت.

---

## Talk

القيم الافتراضية لوضع Talk (macOS/iOS/Android).

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

- يجب أن يطابق `talk.provider` مفتاحاً في `talk.providers` عند تكوين عدة موفّري Talk.
- مفاتيح Talk المسطحة القديمة (`talk.voiceId` و`talk.voiceAliases` و`talk.modelId` و`talk.outputFormat` و`talk.apiKey`) مخصصة للتوافق فقط ويتم ترحيلها تلقائياً إلى `talk.providers.<provider>`.
- تعود معرّفات الصوت إلى `ELEVENLABS_VOICE_ID` أو `SAG_VOICE_ID`.
- تقبل `providers.*.apiKey` سلاسل نصية صريحة أو كائنات SecretRef.
- لا يُطبّق الاحتياطي `ELEVENLABS_API_KEY` إلا عندما لا يكون مفتاح API لـ Talk مكوّناً.
- تتيح `providers.*.voiceAliases` لتوجيهات Talk استخدام أسماء ودية.
- يتحكم `silenceTimeoutMs` في المدة التي ينتظرها وضع Talk بعد صمت المستخدم قبل إرسال النص المفرغ. وعند عدم تعيينه، يحتفظ بنافذة التوقف الافتراضية للمنصة (`700 ms على macOS وAndroid، و900 ms على iOS`).

---

## الأدوات

### ملفات الأدوات

يضبط `tools.profile` قائمة سماح أساسية قبل `tools.allow`/`tools.deny`:

تضبط عملية الإعداد المحلية الافتراضية للإعدادات المحلية الجديدة `tools.profile: "coding"` عندما تكون غير معيّنة (مع الحفاظ على الملفات الصريحة الموجودة).

| الملف | يتضمن |
| ----- | ------ |
| `minimal` | `session_status` فقط |
| `coding` | `group:fs` و`group:runtime` و`group:web` و`group:sessions` و`group:memory` و`cron` و`image` و`image_generate` و`video_generate` |
| `messaging` | `group:messaging` و`sessions_list` و`sessions_history` و`sessions_send` و`session_status` |
| `full` | بدون تقييد (مماثل لعدم التعيين) |

### مجموعات الأدوات

| المجموعة | الأدوات |
| -------- | ------- |
| `group:runtime` | `exec` و`process` و`code_execution` (`bash` مقبول كاسم مستعار لـ `exec`) |
| `group:fs` | `read` و`write` و`edit` و`apply_patch` |
| `group:sessions` | `sessions_list` و`sessions_history` و`sessions_send` و`sessions_spawn` و`sessions_yield` و`subagents` و`session_status` |
| `group:memory` | `memory_search` و`memory_get` |
| `group:web` | `web_search` و`x_search` و`web_fetch` |
| `group:ui` | `browser` و`canvas` |
| `group:automation` | `cron` و`gateway` |
| `group:messaging` | `message` |
| `group:nodes` | `nodes` |
| `group:agents` | `agents_list` |
| `group:media` | `image` و`image_generate` و`video_generate` و`tts` |
| `group:openclaw` | جميع الأدوات المضمنة (باستثناء إضافات الموفّرين) |

### `tools.allow` / `tools.deny`

سياسة السماح/المنع العامة للأدوات (المنع يفوز). غير حساسة لحالة الأحرف، وتدعم البدل `*`. تُطبَّق حتى عندما يكون Docker sandbox معطّلاً.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

تقيّد الأدوات أكثر بالنسبة إلى موفّرين أو نماذج محددة. الترتيب: الملف الأساسي → ملف الموفّر → السماح/المنع.

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

- يمكن للتجاوز لكل وكيل (`agents.list[].tools.elevated`) فقط أن يزيد التقييد.
- يخزن `/elevated on|off|ask|full` الحالة لكل جلسة؛ وتُطبّق التوجيهات المضمنة على رسالة واحدة.
- يتجاوز `exec` المرتفع sandboxing ويستخدم مسار الهروب المكوَّن (`gateway` افتراضياً، أو `node` عندما يكون هدف exec هو `node`).

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

فحوص أمان حلقات الأدوات **معطّلة افتراضياً**. اضبط `enabled: true` لتفعيل الاكتشاف.
يمكن تعريف الإعدادات عالمياً في `tools.loopDetection` وتجاوزها لكل وكيل عند `agents.list[].tools.loopDetection`.

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
- `warningThreshold`: عتبة التحذير لأنماط التكرار بلا تقدم.
- `criticalThreshold`: عتبة أعلى لحظر الحلقات الحرجة المتكررة.
- `globalCircuitBreakerThreshold`: عتبة إيقاف صلبة لأي تشغيل بلا تقدم.
- `detectors.genericRepeat`: يحذر من تكرار استدعاءات نفس الأداة/نفس الوسائط.
- `detectors.knownPollNoProgress`: يحذر/يحظر أدوات الاستطلاع المعروفة (`process.poll` و`command_status` وغيرها).
- `detectors.pingPong`: يحذر/يحظر الأنماط الزوجية المتناوبة بلا تقدم.
- إذا كانت `warningThreshold >= criticalThreshold` أو `criticalThreshold >= globalCircuitBreakerThreshold`، يفشل التحقق.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // or BRAVE_API_KEY env
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // optional; omit for auto-detect
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
        directSend: false, // opt-in: send finished async music/video directly to the channel
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

<Accordion title="حقول إدخالات نماذج الوسائط">

**إدخال الموفّر** (`type: "provider"` أو محذوف):

- `provider`: معرّف موفّر API (`openai` أو `anthropic` أو `google`/`gemini` أو `groq`، إلخ)
- `model`: تجاوز معرّف النموذج
- `profile` / `preferredProfile`: اختيار ملف `auth-profiles.json`

**إدخال CLI** (`type: "cli"`):

- `command`: الملف التنفيذي الذي يجب تشغيله
- `args`: وسائط مقلّبة بالقوالب (تدعم `{{MediaPath}}` و`{{Prompt}}` و`{{MaxChars}}` وغيرها)

**الحقول المشتركة:**

- `capabilities`: قائمة اختيارية (`image` و`audio` و`video`). القيم الافتراضية: `openai`/`anthropic`/`minimax` → صورة، `google` → صورة+صوت+فيديو، `groq` → صوت.
- `prompt` و`maxChars` و`maxBytes` و`timeoutSeconds` و`language`: تجاوزات لكل إدخال.
- تنتقل الإخفاقات إلى الإدخال التالي.

تتبع مصادقة الموفّر الترتيب القياسي: `auth-profiles.json` → متغيرات البيئة → `models.providers.*.apiKey`.

**حقول الإكمال غير المتزامن:**

- `asyncCompletion.directSend`: عند `true`، تحاول المهام المكتملة غير المتزامنة لـ `music_generate`
  و`video_generate` أولاً التسليم المباشر إلى القناة. الافتراضي: `false`
  (المسار القديم لإيقاظ جلسة الطالب/تسليم النموذج).

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

الافتراضي: `tree` (الجلسة الحالية + الجلسات التي أنشأتها، مثل الوكلاء الفرعيين).

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
- `tree`: الجلسة الحالية + الجلسات التي أنشأتها الجلسة الحالية (الوكلاء الفرعيون).
- `agent`: أي جلسة تنتمي إلى معرّف الوكيل الحالي (وقد يشمل ذلك مستخدمين آخرين إذا كنت تشغّل جلسات per-sender تحت معرّف الوكيل نفسه).
- `all`: أي جلسة. ولا يزال الاستهداف عبر وكلاء مختلفين يتطلب `tools.agentToAgent`.
- ضبط sandbox: عندما تكون الجلسة الحالية معزولة و`agents.defaults.sandbox.sessionToolsVisibility="spawned"`، تُفرض الرؤية إلى `tree` حتى لو كانت `tools.sessions.visibility="all"`.

### `tools.sessions_spawn`

يتحكم في دعم المرفقات المضمنة لـ `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: set true to allow inline file attachments
        maxTotalBytes: 5242880, // 5 MB total across all files
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB per file
        retainOnSessionKeep: false, // keep attachments when cleanup="keep"
      },
    },
  },
}
```

ملاحظات:

- لا تُدعم المرفقات إلا لـ `runtime: "subagent"`. ويرفض ACP runtime هذه المرفقات.
- تُحوَّل الملفات إلى مساحة عمل الابن تحت `.openclaw/attachments/<uuid>/` مع ملف `.manifest.json`.
- يتم تلقائياً إخفاء محتوى المرفقات من حفظ السجل.
- يتم التحقق من مدخلات Base64 بفحوص صارمة للأبجدية/الحشو وحاجز حجم قبل فك الترميز.
- تكون أذونات الملفات `0700` للأدلة و`0600` للملفات.
- يتبع التنظيف سياسة `cleanup`: يزيل `delete` المرفقات دائماً؛ بينما يحتفظ `keep` بها فقط عندما `retainOnSessionKeep: true`.

### `tools.experimental`

أعلام أدوات مضمّنة تجريبية. وهي معطّلة افتراضياً ما لم تُطبَّق قاعدة تفعيل تلقائي خاصة بوقت تشغيل معيّن.

```json5
{
  tools: {
    experimental: {
      planTool: true, // enable experimental update_plan
    },
  },
}
```

ملاحظات:

- `planTool`: يفعّل الأداة البنيوية التجريبية `update_plan` لتتبع الأعمال متعددة الخطوات غير البسيطة.
- الافتراضي: `false` للموفّرين غير OpenAI. وتقوم تشغيلات OpenAI وOpenAI Codex بتفعيلها تلقائياً.
- عند تفعيلها، تضيف system prompt أيضاً إرشادات استخدام حتى لا يستخدمها النموذج إلا للعمل الجوهري ويحافظ على خطوة واحدة فقط كحد أقصى في الحالة `in_progress`.

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

- `model`: النموذج الافتراضي للوكلاء الفرعيين المنشئين. وإذا حُذف، يرث الوكلاء الفرعيون نموذج الطالب.
- `allowAgents`: قائمة السماح الافتراضية لمعرّفات الوكلاء المستهدفين في `sessions_spawn` عندما لا يضبط الوكيل الطالب `subagents.allowAgents` الخاص به (`["*"]` = أي وكيل؛ الافتراضي: نفس الوكيل فقط).
- `runTimeoutSeconds`: المهلة الافتراضية (بالثواني) لـ `sessions_spawn` عندما يحذف استدعاء الأداة `runTimeoutSeconds`. وتعني `0` عدم وجود مهلة.
- سياسة الأدوات لكل وكيل فرعي: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## الموفّرون المخصصون وbase URLs

يستخدم OpenClaw فهرس النماذج المضمن. أضف موفّرين مخصصين عبر `models.providers` في الإعدادات أو `~/.openclaw/agents/<agentId>/agent/models.json`.

```json5
{
  models: {
    mode: "merge", // merge (default) | replace
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
- تجاوز جذر إعداد الوكيل عبر `OPENCLAW_AGENT_DIR` (أو `PI_CODING_AGENT_DIR`، وهو اسم قديم بديل لمتغير البيئة).
- أولوية الدمج لمعرّفات الموفّرين المطابقة:
  - تفوز قيم `baseUrl` غير الفارغة في `models.json` الخاص بالوكيل.
  - تفوز قيم `apiKey` غير الفارغة الخاصة بالوكيل فقط عندما لا يكون ذلك الموفّر مُداراً عبر SecretRef في سياق الإعداد/ملف المصادقة الحالي.
  - تُحدَّث قيم `apiKey` للموفّرين المُدارين عبر SecretRef من علامات المصدر (`ENV_VAR_NAME` لمراجع البيئة، و`secretref-managed` لمراجع الملفات/الأوامر التنفيذية) بدلاً من حفظ الأسرار المحلولة.
  - تُحدَّث قيم headers الخاصة بالموفّر المُدار عبر SecretRef من علامات المصدر (`secretref-env:ENV_VAR_NAME` لمراجع البيئة، و`secretref-managed` لمراجع الملفات/الأوامر التنفيذية).
  - تعود `apiKey`/`baseUrl` الفارغة أو المحذوفة على مستوى الوكيل إلى `models.providers` في الإعدادات.
  - تستخدم قيم `contextWindow`/`maxTokens` الأعلى بين الإعداد الصريح وقيم الفهرس الضمنية للنموذج المطابق.
  - تحافظ `contextTokens` المطابقة على الحد الصريح الفعّال في وقت التشغيل عندما يكون موجوداً؛ استخدمها لتقييد السياق الفعلي من دون تغيير البيانات الوصفية الأصلية للنموذج.
  - استخدم `models.mode: "replace"` عندما تريد من الإعدادات إعادة كتابة `models.json` بالكامل.
  - حفظ العلامات يعتمد على المصدر كمرجع: تُكتب العلامات من لقطة إعداد المصدر النشطة (قبل الحل)، وليس من قيم الأسرار المحلولة في وقت التشغيل.

### تفاصيل حقول الموفّر

- `models.mode`: سلوك فهرس الموفّر (`merge` أو `replace`).
- `models.providers`: خريطة الموفّرين المخصصين مفهرسة حسب معرّف الموفّر.
- `models.providers.*.api`: محوّل الطلب (`openai-completions` أو `openai-responses` أو `anthropic-messages` أو `google-generative-ai`، إلخ).
- `models.providers.*.apiKey`: اعتماد الموفّر (ويُفضّل SecretRef/استبدال البيئة).
- `models.providers.*.auth`: استراتيجية المصادقة (`api-key` أو `token` أو `oauth` أو `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: بالنسبة إلى Ollama + `openai-completions`، يحقن `options.num_ctx` في الطلبات (الافتراضي: `true`).
- `models.providers.*.authHeader`: يفرض نقل الاعتماد في ترويسة `Authorization` عند الحاجة.
- `models.providers.*.baseUrl`: base URL لواجهة API العليا.
- `models.providers.*.headers`: ترويسات ثابتة إضافية للتوجيه عبر الوكيل/المستأجر.
- `models.providers.*.request`: تجاوزات النقل لطلبات HTTP الخاصة بموفّر النموذج.
  - `request.headers`: ترويسات إضافية (تُدمج مع افتراضيات الموفّر). تقبل القيم SecretRef.
  - `request.auth`: تجاوز استراتيجية المصادقة. الأوضاع: `"provider-default"` (استخدام المصادقة المضمنة للموفّر)، و`"authorization-bearer"` (مع `token`)، و`"header"` (مع `headerName` و`value` و`prefix` الاختياري).
  - `request.proxy`: تجاوز وكيل HTTP. الأوضاع: `"env-proxy"` (استخدام متغيرات البيئة `HTTP_PROXY`/`HTTPS_PROXY`)، و`"explicit-proxy"` (مع `url`). يدعم كلا الوضعين كائناً فرعياً اختيارياً `tls`.
  - `request.tls`: تجاوز TLS للاتصالات المباشرة. الحقول: `ca` و`cert` و`key` و`passphrase` (كلها تقبل SecretRef)، و`serverName`، و`insecureSkipVerify`.
- `models.providers.*.models`: إدخالات فهرس نماذج صريحة للموفّر.
- `models.providers.*.models.*.contextWindow`: بيانات وصفية لنافذة السياق الأصلية للنموذج.
- `models.providers.*.models.*.contextTokens`: حد سياق اختياري لوقت التشغيل. استخدمه عندما تريد ميزانية سياق فعّالة أصغر من `contextWindow` الأصلية للنموذج.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: تلميح توافق اختياري. بالنسبة إلى `api: "openai-completions"` مع `baseUrl` غير فارغ وغير أصلي (مضيف ليس `api.openai.com`)، يفرض OpenClaw هذه القيمة إلى `false` وقت التشغيل. أما `baseUrl` الفارغ/المحذوف فيبقي سلوك OpenAI الافتراضي.
- `plugins.entries.amazon-bedrock.config.discovery`: جذر إعدادات الاكتشاف التلقائي لـ Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: تشغيل/إيقاف الاكتشاف الضمني.
- `plugins.entries.amazon-bedrock.config.discovery.region`: منطقة AWS للاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: مرشح اختياري لمعرّف الموفّر لاكتشاف موجّه.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: فترة الاستطلاع لتحديث الاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: نافذة السياق الاحتياطية للنماذج المكتشفة.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: الحد الأقصى الاحتياطي لرموز الخرج للنماذج المكتشفة.

### أمثلة على الموفّرين

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

استخدم `cerebras/zai-glm-4.7` لـ Cerebras؛ و`zai/glm-4.7` لـ Z.AI المباشر.

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

اضبط `OPENCODE_API_KEY` (أو `OPENCODE_ZEN_API_KEY`). استخدم مراجع `opencode/...` لفهرس Zen أو مراجع `opencode-go/...` لفهرس Go. اختصار: `openclaw onboard --auth-choice opencode-zen` أو `openclaw onboard --auth-choice opencode-go`.

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

اضبط `ZAI_API_KEY`. وتُقبل `z.ai/*` و`z-ai/*` كأسماء مستعارة. اختصار: `openclaw onboard --auth-choice zai-api-key`.

- نقطة النهاية العامة: `https://api.z.ai/api/paas/v4`
- نقطة النهاية البرمجية (الافتراضية): `https://api.z.ai/api/coding/paas/v4`
- بالنسبة إلى نقطة النهاية العامة، عرّف موفّراً مخصصاً مع تجاوز base URL.

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

تعلن نقاط النهاية الأصلية لـ Moonshot عن توافق استخدام البث على مسار
`openai-completions` المشترك، ويعتمد OpenClaw الآن ذلك على قدرات نقطة النهاية
بدلاً من معرّف الموفّر المضمن وحده.

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

متوافق مع Anthropic ومُضمَّن كموفّر. اختصار: `openclaw onboard --auth-choice kimi-code-api-key`.

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

يجب أن يحذف base URL اللاحقة `/v1` (لأن عميل Anthropic يضيفها). اختصار: `openclaw onboard --auth-choice synthetic-api-key`.

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
يستخدم فهرس النماذج الآن M2.7 فقط افتراضياً.
في مسار البث المتوافق مع Anthropic، يعطّل OpenClaw تفكير MiniMax
افتراضياً ما لم تضبط `thinking` بنفسك صراحةً. يعيد `/fast on` أو
`params.fastMode: true` كتابة `MiniMax-M2.7` إلى
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="النماذج المحلية (LM Studio)">

راجع [النماذج المحلية](/ar/gateway/local-models). باختصار: شغّل نموذجاً محلياً كبيراً عبر LM Studio Responses API على عتاد قوي؛ واحتفظ بالنماذج المستضافة مدموجة كخيار احتياطي.

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
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // or plaintext string
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: قائمة سماح اختيارية لـ Skills المضمنة فقط (ولا تتأثر Skills المُدارة/Skills مساحات العمل).
- `load.extraDirs`: جذور Skills مشتركة إضافية (أدنى أولوية).
- `install.preferBrew`: عند true، يفضّل مثبّتات Homebrew عندما يكون `brew` متاحاً
  قبل الرجوع إلى أنواع المثبّتات الأخرى.
- `install.nodeManager`: تفضيل مثبّت node لمواصفات `metadata.openclaw.install`
  (`npm` | `pnpm` | `yarn` | `bun`).
- `entries.<skillKey>.enabled: false` يعطّل Skill حتى لو كانت مضمّنة/مثبّتة.
- `entries.<skillKey>.apiKey`: حقل تسهيلي لمفتاح API على مستوى Skill (عند دعم Skill لذلك) للـ Skills التي تعلن متغير بيئة أساسيّاً (سلسلة نصية صريحة أو كائن SecretRef).

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

- يتم تحميلها من `~/.openclaw/extensions` و`<workspace>/.openclaw/extensions` بالإضافة إلى `plugins.load.paths`.
- يَقبل الاكتشاف إضافات OpenClaw الأصلية بالإضافة إلى الحزم المتوافقة مع Codex وحزم Claude، بما في ذلك حزم Claude ذات التخطيط الافتراضي من دون manifest.
- **تتطلب تغييرات الإعدادات إعادة تشغيل البوابة.**
- `allow`: قائمة سماح اختيارية (يتم تحميل الإضافات المذكورة فقط). ويفوز `deny`.
- `plugins.entries.<id>.apiKey`: حقل تسهيلي لمفتاح API على مستوى الإضافة (عندما تدعمه الإضافة).
- `plugins.entries.<id>.env`: خريطة متغيرات بيئة على مستوى الإضافة.
- `plugins.entries.<id>.hooks.allowPromptInjection`: عندما تكون `false`، يمنع core الخطاف `before_prompt_build` ويتجاهل الحقول المعدِّلة لـ prompt من `before_agent_start` القديم، مع الحفاظ على `modelOverride` و`providerOverride` القديمين. وينطبق ذلك على خطافات الإضافات الأصلية وأدلة الخطافات المقدمة من الحزم المدعومة.
- `plugins.entries.<id>.subagent.allowModelOverride`: يثق صراحةً بهذه الإضافة لطلب تجاوزات `provider` و`model` لكل تشغيل لوظائف الوكلاء الفرعيين الخلفية.
- `plugins.entries.<id>.subagent.allowedModels`: قائمة سماح اختيارية لأهداف `provider/model` القياسية لتجاوزات الوكلاء الفرعيين الموثوق بها. استخدم `"*"` فقط عندما تقصد السماح بأي نموذج.
- `plugins.entries.<id>.config`: كائن إعدادات معرّف من الإضافة (ويتم التحقق منه بواسطة مخطط إضافة OpenClaw الأصلية عند توفره).
- `plugins.entries.firecrawl.config.webFetch`: إعدادات موفّر جلب الويب Firecrawl.
  - `apiKey`: مفتاح API لـ Firecrawl (يقبل SecretRef). ويعود احتياطياً إلى `plugins.entries.firecrawl.config.webSearch.apiKey`، أو `tools.web.fetch.firecrawl.apiKey` القديم، أو متغير البيئة `FIRECRAWL_API_KEY`.
  - `baseUrl`: base URL لواجهة Firecrawl API (الافتراضي: `https://api.firecrawl.dev`).
  - `onlyMainContent`: استخراج المحتوى الرئيسي فقط من الصفحات (الافتراضي: `true`).
  - `maxAgeMs`: أقصى عمر للذاكرة المؤقتة بالميلي ثانية (الافتراضي: `172800000` / يومان).
  - `timeoutSeconds`: مهلة طلب الجلب بالثواني (الافتراضي: `60`).
- `plugins.entries.xai.config.xSearch`: إعدادات xAI X Search (بحث Grok على الويب).
  - `enabled`: تفعيل موفّر X Search.
  - `model`: نموذج Grok المستخدم للبحث (مثل `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming`: إعدادات dreaming الخاصة بالذاكرة (تجريبية). راجع [Dreaming](/ar/concepts/dreaming) لمعرفة المراحل والعتبات.
  - `enabled`: مفتاح dreaming الرئيسي (الافتراضي `false`).
  - `frequency`: وتيرة cron لكل دورة dreaming كاملة (`"0 3 * * *"` افتراضياً).
  - سياسة المراحل والعتبات هي تفاصيل تنفيذية (وليست مفاتيح إعداد موجهة للمستخدم).
- يمكن لحزم Claude bundle المفعّلة أيضاً أن تساهم بإعدادات Pi الافتراضية المضمّنة من `settings.json`؛ ويطبّق OpenClaw هذه القيم كإعدادات وكيل مُنقّاة، وليس كرقع إعداد OpenClaw خام.
- `plugins.slots.memory`: اختر معرّف إضافة الذاكرة النشطة، أو `"none"` لتعطيل إضافات الذاكرة.
- `plugins.slots.contextEngine`: اختر معرّف إضافة محرك السياق النشط؛ ويعود افتراضياً إلى `"legacy"` ما لم تثبّت محركاً آخر وتحدده.
- `plugins.installs`: بيانات تعريف التثبيت المُدارة من CLI والمستخدمة بواسطة `openclaw plugins update`.
  - تتضمن `source` و`spec` و`sourcePath` و`installPath` و`version` و`resolvedName` و`resolvedVersion` و`resolvedSpec` و`integrity` و`shasum` و`resolvedAt` و`installedAt`.
  - تعامل مع `plugins.installs.*` كحالة مُدارة؛ ويفضَّل استخدام أوامر CLI بدلاً من التعديلات اليدوية.

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
      dangerouslyAllowPrivateNetwork: true, // default trusted-network mode
      // allowPrivateNetwork: true, // legacy alias
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

- يعطّل `evaluateEnabled: false` كلاً من `act:evaluate` و`wait --fn`.
- يعود `ssrfPolicy.dangerouslyAllowPrivateNetwork` افتراضياً إلى `true` عندما لا يكون معيّناً (نموذج شبكة موثوقة).
- اضبط `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` من أجل تنقل متصفح صارم يقتصر على الإنترنت العام.
- في الوضع الصارم، تخضع نقاط نهاية CDP الخاصة بالملفات البعيدة (`profiles.*.cdpUrl`) لنفس حظر الشبكات الخاصة أثناء فحوص الوصول/الاكتشاف.
- يظل `ssrfPolicy.allowPrivateNetwork` مدعوماً كاسم مستعار قديم.
- في الوضع الصارم، استخدم `ssrfPolicy.hostnameAllowlist` و`ssrfPolicy.allowedHostnames` للاستثناءات الصريحة.
- الملفات البعيدة تكون attach-only (بدون start/stop/reset).
- تقبل `profiles.*.cdpUrl` البروتوكولات `http://` و`https://` و`ws://` و`wss://`.
  استخدم HTTP(S) عندما تريد من OpenClaw اكتشاف `/json/version`؛ واستخدم WS(S)
  عندما يوفّر لك المزوّد URL مباشر لـ DevTools WebSocket.
- ملفات `existing-session` مخصصة للمضيف فقط وتستخدم Chrome MCP بدلاً من CDP.
- يمكن لملفات `existing-session` تعيين `userDataDir` لاستهداف ملف تعريف
  متصفح Chromium-based محدد مثل Brave أو Edge.
- تحتفظ ملفات `existing-session` بحدود مسار Chrome MCP الحالية:
  إجراءات قائمة على snapshot/ref بدلاً من استهداف محددات CSS، وخطافات رفع ملف واحد،
  ومن دون تجاوزات مهلة الحوارات، ومن دون `wait --load networkidle`، ومن دون
  `responsebody` أو تصدير PDF أو اعتراض التنزيل أو الإجراءات الدفعية.
- تقوم ملفات `openclaw` المحلية المُدارة بتعيين `cdpPort` و`cdpUrl` تلقائياً؛ ولا
  تضبط `cdpUrl` صراحةً إلا لـ CDP البعيد.
- ترتيب الاكتشاف التلقائي: المتصفح الافتراضي إذا كان Chromium-based → Chrome → Brave → Edge → Chromium → Chrome Canary.
- خدمة التحكم: loopback فقط (منفذ مشتق من `gateway.port`، الافتراضي `18791`).
- تضيف `extraArgs` أعلام تشغيل إضافية إلى بدء Chromium المحلي (مثل
  `--disable-gpu`، أو تحجيم النافذة، أو أعلام التصحيح).

---

## UI

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // emoji, short text, image URL, or data URI
    },
  },
}
```

- `seamColor`: لون التمييز لعناصر UI الأصلية (مثل لون فقاعة Talk Mode وغيرها).
- `assistant`: تجاوز هوية Control UI. ويعود احتياطياً إلى هوية الوكيل النشط.

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
      // password: "your-password", // or OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // for mode=trusted-proxy; see /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },