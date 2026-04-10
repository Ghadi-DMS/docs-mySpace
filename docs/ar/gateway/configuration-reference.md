---
read_when:
    - أنت بحاجة إلى دلالات الإعدادات الدقيقة على مستوى الحقول أو إلى القيم الافتراضية
    - أنت تتحقق من كتل إعدادات القناة أو النموذج أو Gateway أو الأداة
summary: مرجع إعدادات Gateway لمفاتيح OpenClaw الأساسية والقيم الافتراضية والروابط إلى مراجع الأنظمة الفرعية المخصصة
title: مرجع الإعدادات
x-i18n:
    generated_at: "2026-04-10T07:17:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: c6aa34e3a9096c4dabef26b9cdfda4bd7693e16da43a88c1641cfad8ba1ec237
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# مرجع الإعدادات

مرجع الإعدادات الأساسية لملف `~/.openclaw/openclaw.json`. للحصول على نظرة عامة موجّهة حسب المهام، راجع [الإعدادات](/ar/gateway/configuration).

تغطي هذه الصفحة أسطح إعدادات OpenClaw الرئيسية، وتوفّر روابط خارجية عندما يكون لنظام فرعي مرجع أعمق خاص به. وهي **لا** تحاول تضمين كل كتالوج أوامر مملوك للقنوات/الإضافات أو كل إعدادات الذاكرة/QMD المتعمقة في صفحة واحدة.

مصدر الحقيقة في الكود:

- يعرض `openclaw config schema` مخطط JSON Schema الفعلي المستخدم للتحقق وواجهة Control UI، مع دمج بيانات الإضافات/القنوات المضمنة عند توفّرها
- يعيد `config.schema.lookup` عقدة مخطط واحدة ضمن نطاق مسار محدد لأدوات الاستكشاف التفصيلي
- تتحقق أوامر `pnpm config:docs:check` / `pnpm config:docs:gen` من تجزئة خط الأساس لوثائق الإعدادات مقارنة بسطح المخطط الحالي

المراجع المتعمقة المخصصة:

- [مرجع إعدادات الذاكرة](/ar/reference/memory-config) لـ `agents.defaults.memorySearch.*` و`memory.qmd.*` و`memory.citations` وإعدادات dreaming ضمن `plugins.entries.memory-core.config.dreaming`
- [أوامر الشرطة المائلة](/ar/tools/slash-commands) لكتالوج الأوامر الحالي المضمن + المجمّع
- صفحات القنوات/الإضافات المالكة لأسطح الأوامر الخاصة بكل قناة

صيغة الإعدادات هي **JSON5** (يُسمح بالتعليقات والفواصل اللاحقة). جميع الحقول اختيارية — يستخدم OpenClaw قيمًا افتراضية آمنة عند عدم تحديدها.

---

## القنوات

تبدأ كل قناة تلقائيًا عندما يوجد قسم إعداداتها (إلا إذا كان `enabled: false`).

### الوصول إلى الرسائل الخاصة والمجموعات

تدعم جميع القنوات سياسات الرسائل الخاصة وسياسات المجموعات:

| سياسة الرسائل الخاصة | السلوك                                                          |
| -------------------- | --------------------------------------------------------------- |
| `pairing` (افتراضي)  | يحصل المرسلون غير المعروفين على رمز اقتران لمرة واحدة؛ يجب أن يوافق المالك |
| `allowlist`          | يُسمح فقط للمرسلين الموجودين في `allowFrom` (أو مخزن السماح المقترن) |
| `open`               | السماح بكل الرسائل الخاصة الواردة (يتطلب `allowFrom: ["*"]`)   |
| `disabled`           | تجاهل كل الرسائل الخاصة الواردة                                 |

| سياسة المجموعة         | السلوك                                                 |
| ---------------------- | ------------------------------------------------------ |
| `allowlist` (افتراضي)  | يُسمح فقط بالمجموعات المطابقة لقائمة السماح المضبوطة   |
| `open`                 | تجاوز قوائم السماح الخاصة بالمجموعات (مع استمرار تطبيق اشتراط الإشارة) |
| `disabled`             | حظر جميع رسائل المجموعات/الغرف                         |

<Note>
يضبط `channels.defaults.groupPolicy` السياسة الافتراضية عندما لا يتم تعيين `groupPolicy` لموفّر معين.
تنتهي صلاحية رموز الاقتران بعد ساعة واحدة. ويقتصر عدد طلبات اقتران الرسائل الخاصة المعلّقة على **3 لكل قناة**.
إذا كانت كتلة الموفّر مفقودة بالكامل (`channels.<provider>` غير موجودة)، تعود سياسة المجموعات وقت التشغيل إلى `allowlist` (إغلاق آمن افتراضيًا) مع تحذير عند بدء التشغيل.
</Note>

### تجاوزات نموذج القناة

استخدم `channels.modelByChannel` لتثبيت معرّفات قنوات محددة على نموذج معيّن. تقبل القيم `provider/model` أو أسماء النماذج المستعارة المضبوطة. يُطبّق تعيين القناة عندما لا تكون للجلسة بالفعل قيمة تجاوز للنموذج (على سبيل المثال، عند ضبطها عبر `/model`).

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

### القيم الافتراضية للقنوات وإشارة النبض

استخدم `channels.defaults` لسلوك سياسة المجموعات وإشارة النبض المشتركة بين الموفّرين:

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

- `channels.defaults.groupPolicy`: سياسة المجموعات الاحتياطية عندما لا يتم تعيين `groupPolicy` على مستوى الموفّر.
- `channels.defaults.contextVisibility`: وضع الرؤية الافتراضي للسياق الإضافي لجميع القنوات. القيم: `all` (الافتراضي، تضمين كل سياق الاقتباس/المحادثة/السجل)، و`allowlist` (تضمين السياق من المرسلين الموجودين في قائمة السماح فقط)، و`allowlist_quote` (مثل allowlist لكن مع الاحتفاظ بسياق الاقتباس/الرد الصريح). التجاوز لكل قناة: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: تضمين حالات القنوات السليمة في مخرجات إشارة النبض.
- `channels.defaults.heartbeat.showAlerts`: تضمين الحالات المتدهورة/حالات الخطأ في مخرجات إشارة النبض.
- `channels.defaults.heartbeat.useIndicator`: عرض مخرجات إشارة النبض بأسلوب مؤشّر مضغوط.

### WhatsApp

يعمل WhatsApp عبر قناة الويب في Gateway (Baileys Web). ويبدأ تلقائيًا عند وجود جلسة مرتبطة.

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

- تستخدم الأوامر الصادرة الحساب `default` افتراضيًا إذا كان موجودًا؛ وإلا فسيُستخدم أول معرّف حساب مضبوط (بعد الفرز).
- يجاوز `channels.whatsapp.defaultAccount` الاختياري هذا الاختيار الافتراضي الاحتياطي للحساب عندما يطابق معرّف حساب مضبوطًا.
- ينقل `openclaw doctor` دليل مصادقة Baileys القديم أحادي الحساب إلى `whatsapp/default`.
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

- رمز البوت: `channels.telegram.botToken` أو `channels.telegram.tokenFile` (ملف عادي فقط؛ تُرفض الروابط الرمزية)، مع `TELEGRAM_BOT_TOKEN` كقيمة احتياطية للحساب الافتراضي.
- يجاوز `channels.telegram.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- في إعدادات تعدد الحسابات (معرّفا حساب أو أكثر)، اضبط قيمة افتراضية صريحة (`channels.telegram.defaultAccount` أو `channels.telegram.accounts.default`) لتجنّب التوجيه الاحتياطي؛ ويحذّر `openclaw doctor` عند غياب هذا الإعداد أو عدم صحته.
- يمنع `configWrites: false` عمليات كتابة الإعدادات التي يطلقها Telegram (ترحيل معرّفات supergroup وأوامر `/config set|unset`).
- تضبط إدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ارتباطات ACP دائمة لموضوعات المنتدى (استخدم الصيغة القياسية `chatId:topic:topicId` في `match.peer.id`). دلالات الحقول مشتركة في [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- تستخدم معاينات البث في Telegram الأمرين `sendMessage` و`editMessageText` (ويعمل ذلك في المحادثات المباشرة والجماعية).
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

- الرمز المميّز: `channels.discord.token`، مع `DISCORD_BOT_TOKEN` كقيمة احتياطية للحساب الافتراضي.
- تستخدم الاستدعاءات الصادرة المباشرة التي توفّر `token` صريحًا لـ Discord ذلك الرمز المميّز في الاستدعاء؛ بينما تظل إعدادات إعادة المحاولة/السياسات الخاصة بالحساب مأخوذة من الحساب المحدد في اللقطة النشطة لوقت التشغيل.
- يجاوز `channels.discord.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- استخدم `user:<id>` (رسالة خاصة) أو `channel:<id>` (قناة guild) كأهداف للتسليم؛ وتُرفض المعرّفات الرقمية المجردة.
- تكون الأسماء المختصرة لـ guilds بأحرف صغيرة مع استبدال المسافات بـ `-`؛ وتستخدم مفاتيح القنوات الاسم المختصر (من دون `#`). ويُفضَّل استخدام معرّفات guild.
- تُتجاهل الرسائل التي ينشئها البوت افتراضيًا. يفعّل `allowBots: true` قبولها؛ واستخدم `allowBots: "mentions"` لقبول رسائل البوت التي تذكر البوت فقط (مع استمرار تصفية رسائل البوت نفسه).
- يؤدي `channels.discord.guilds.<id>.ignoreOtherMentions` (وتجاوزاته على مستوى القناة) إلى إسقاط الرسائل التي تذكر مستخدمًا آخر أو دورًا آخر ولكن لا تذكر البوت (باستثناء @everyone/@here).
- يقوم `maxLinesPerMessage` (الافتراضي 17) بتقسيم الرسائل الطويلة عموديًا حتى عندما تكون أقل من 2000 حرف.
- يتحكم `channels.discord.threadBindings` في التوجيه المرتبط بسلاسل Discord:
  - `enabled`: تجاوز Discord لميزات الجلسات المرتبطة بالسلاسل (`/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age` والتسليم/التوجيه المرتبط)
  - `idleHours`: تجاوز Discord لإلغاء التركيز التلقائي بعد عدم النشاط بالساعات (`0` للتعطيل)
  - `maxAgeHours`: تجاوز Discord للحد الأقصى الصارم للعمر بالساعات (`0` للتعطيل)
  - `spawnSubagentSessions`: مفتاح تفعيل اختياري لإنشاء/ربط السلاسل تلقائيًا عبر `sessions_spawn({ thread: true })`
- تضبط إدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ارتباطات ACP دائمة للقنوات والسلاسل (استخدم معرّف القناة/السلسلة في `match.peer.id`). دلالات الحقول مشتركة في [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- يضبط `channels.discord.ui.components.accentColor` لون التمييز لحاويات مكوّنات Discord v2.
- يفعّل `channels.discord.voice` محادثات قنوات Discord الصوتية، بالإضافة إلى تجاوزات الانضمام التلقائي وTTS الاختيارية.
- يمرر `channels.discord.voice.daveEncryption` و`channels.discord.voice.decryptionFailureTolerance` مباشرةً إلى خيارات DAVE في `@discordjs/voice` (القيم الافتراضية `true` و`24`).
- يحاول OpenClaw أيضًا استعادة استقبال الصوت عبر مغادرة جلسة صوتية ثم الانضمام إليها مجددًا بعد تكرار إخفاقات فك التشفير.
- يُعد `channels.discord.streaming` مفتاح وضع البث القياسي. وتُرحَّل تلقائيًا القيم القديمة `streamMode` والقيم المنطقية `streaming`.
- يربط `channels.discord.autoPresence` التوفّر في وقت التشغيل بحالة حضور البوت (سليم => online، متدهور => idle، مستنفد => dnd) ويسمح بتجاوزات اختيارية لنص الحالة.
- يعيد `channels.discord.dangerouslyAllowNameMatching` تفعيل المطابقة بالأسماء/الوسوم القابلة للتغيير (وضع توافق طارئ).
- `channels.discord.execApprovals`: تسليم موافقات exec الأصلية في Discord وتفويض المعتمدين.
  - `enabled`: يمكن أن تكون `true` أو `false` أو `"auto"` (افتراضي). في الوضع التلقائي، تُفعَّل موافقات exec عندما يمكن حل المعتمدين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Discord المسموح لهم بالموافقة على طلبات exec. تعود إلى `commands.ownerAllowFrom` عند عدم تحديدها.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتمرير الموافقات لجميع الوكلاء.
  - `sessionFilter`: أنماط مفاتيح جلسات اختيارية (سلسلة جزئية أو regex).
  - `target`: المكان الذي تُرسل إليه مطالبات الموافقة. `"dm"` (الافتراضي) يرسلها إلى الرسائل الخاصة للمعتمدين، و`"channel"` يرسلها إلى القناة الأصلية، و`"both"` يرسلها إلى كليهما. عندما يتضمن الهدف `"channel"`، لا يمكن استخدام الأزرار إلا من قبل المعتمدين الذين تم حلهم.
  - `cleanupAfterResolve`: عند ضبطه على `true`، يحذف رسائل الموافقة الخاصة بعد الموافقة أو الرفض أو انتهاء المهلة.

**أوضاع إشعارات التفاعلات:** `off` (بدون)، و`own` (رسائل البوت، افتراضي)، و`all` (كل الرسائل)، و`allowlist` (من `guilds.<id>.users` على جميع الرسائل).

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

- JSON الخاص بحساب الخدمة: مضمن (`serviceAccount`) أو قائم على ملف (`serviceAccountFile`).
- يُدعم أيضًا SecretRef لحساب الخدمة (`serviceAccountRef`).
- القيم الاحتياطية من البيئة: `GOOGLE_CHAT_SERVICE_ACCOUNT` أو `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- استخدم `spaces/<spaceId>` أو `users/<userId>` كأهداف للتسليم.
- يعيد `channels.googlechat.dangerouslyAllowNameMatching` تفعيل مطابقة مبدأ البريد الإلكتروني القابل للتغيير (وضع توافق طارئ).

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
      streaming: {
        mode: "partial", // off | partial | block | progress
        nativeTransport: true, // use Slack native streaming API when mode=partial
      },
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

- يتطلب **وضع Socket** كلاً من `botToken` و`appToken` (مع `SLACK_BOT_TOKEN` و`SLACK_APP_TOKEN` كقيم احتياطية من البيئة للحساب الافتراضي).
- يتطلب **وضع HTTP** وجود `botToken` بالإضافة إلى `signingSecret` (في الجذر أو لكل حساب).
- تقبل `botToken` و`appToken` و`signingSecret` و`userToken` سلاسل نصية صريحة أو كائنات SecretRef.
- تكشف لقطات حساب Slack عن حقول المصدر/الحالة لكل بيانات الاعتماد مثل `botTokenSource` و`botTokenStatus` و`appTokenStatus`، وفي وضع HTTP، `signingSecretStatus`. تعني `configured_unavailable` أن الحساب مضبوط عبر SecretRef، لكن مسار الأمر/وقت التشغيل الحالي لم يتمكن من حل قيمة السر.
- يمنع `configWrites: false` عمليات كتابة الإعدادات التي يطلقها Slack.
- يجاوز `channels.slack.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- يُعد `channels.slack.streaming.mode` مفتاح وضع البث القياسي في Slack. ويتحكم `channels.slack.streaming.nativeTransport` في ناقل البث الأصلي لـ Slack. وتُرحَّل تلقائيًا القيم القديمة `streamMode` والقيم المنطقية `streaming` و`nativeStreaming`.
- استخدم `user:<id>` (رسالة خاصة) أو `channel:<id>` كأهداف للتسليم.

**أوضاع إشعارات التفاعلات:** `off` و`own` (افتراضي) و`all` و`allowlist` (من `reactionAllowlist`).

**عزل جلسات السلاسل:** يكون `thread.historyScope` لكل سلسلة على حدة (افتراضيًا) أو مشتركًا عبر القناة. ويؤدي `thread.inheritParent` إلى نسخ سجل القناة الأصلية إلى السلاسل الجديدة.

- يتطلب البث الأصلي في Slack بالإضافة إلى حالة السلسلة بأسلوب “is typing...” الخاصة بمساعد Slack هدف رد ضمن سلسلة. تبقى الرسائل الخاصة ذات المستوى الأعلى خارج السلاسل افتراضيًا، لذا تستخدم `typingReaction` أو التسليم العادي بدلًا من معاينة السلسلة.
- تضيف `typingReaction` تفاعلًا مؤقتًا إلى رسالة Slack الواردة أثناء تنفيذ الرد، ثم تزيله عند الاكتمال. استخدم shortcode لرمز تعبيري في Slack مثل `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: تسليم موافقات exec الأصلية في Slack وتفويض المعتمدين. نفس مخطط Discord: `enabled` (`true`/`false`/`"auto"`)، و`approvers` (معرّفات مستخدمي Slack)، و`agentFilter`، و`sessionFilter`، و`target` (`"dm"` أو `"channel"` أو `"both"`).

| مجموعة الإجراءات | الافتراضي | ملاحظات                     |
| ----------------- | --------- | --------------------------- |
| reactions         | مفعّل     | التفاعل + سرد التفاعلات     |
| messages          | مفعّل     | قراءة/إرسال/تعديل/حذف       |
| pins              | مفعّل     | تثبيت/إلغاء تثبيت/سرد       |
| memberInfo        | مفعّل     | معلومات العضو               |
| emojiList         | مفعّل     | قائمة الرموز التعبيرية المخصصة |

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

أوضاع الدردشة: `oncall` (الرد عند @-mention، افتراضي)، و`onmessage` (كل رسالة)، و`onchar` (الرسائل التي تبدأ ببادئة تشغيل).

عند تفعيل الأوامر الأصلية في Mattermost:

- يجب أن يكون `commands.callbackPath` مسارًا (مثل `/api/channels/mattermost/command`)، وليس URL كاملاً.
- يجب أن يشير `commands.callbackUrl` إلى نقطة نهاية OpenClaw Gateway وأن يكون قابلاً للوصول من خادم Mattermost.
- تتم مصادقة ردود slash الأصلية باستخدام الرموز المميّزة الخاصة بكل أمر التي يعيدها Mattermost أثناء تسجيل أوامر slash. إذا فشل التسجيل أو لم يتم تفعيل أي أوامر، فسيرفض OpenClaw عمليات الرد بالقيمة:
  `Unauthorized: invalid command token.`
- بالنسبة إلى مضيفات الرد الخاصة/الداخلية/ضمن tailnet، قد يتطلب Mattermost أن يتضمن `ServiceSettings.AllowedUntrustedInternalConnections` مضيف/نطاق مضيف الرد.
  استخدم قيم المضيف/النطاق، وليس URL كاملة.
- يتيح أو يمنع `channels.mattermost.configWrites` عمليات كتابة الإعدادات التي يطلقها Mattermost.
- يفرض `channels.mattermost.requireMention` وجود `@mention` قبل الرد في القنوات.
- يوفّر `channels.mattermost.groups.<channelId>.requireMention` تجاوزًا لاشتراط الإشارة لكل قناة (`"*"` للقيمة الافتراضية).
- يجاوز `channels.mattermost.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.

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

**أوضاع إشعارات التفاعلات:** `off` و`own` (افتراضي) و`all` و`allowlist` (من `reactionAllowlist`).

- `channels.signal.account`: يثبّت بدء تشغيل القناة على هوية حساب Signal محددة.
- `channels.signal.configWrites`: يتيح أو يمنع عمليات كتابة الإعدادات التي يطلقها Signal.
- يجاوز `channels.signal.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.

### BlueBubbles

يُعد BlueBubbles المسار الموصى به لـ iMessage (مدعومًا بإضافة، ومضبوطًا ضمن `channels.bluebubbles`).

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

- مسارات المفاتيح الأساسية المغطاة هنا: `channels.bluebubbles` و`channels.bluebubbles.dmPolicy`.
- يجاوز `channels.bluebubbles.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مضبوطًا.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات BlueBubbles بجلسات ACP دائمة. استخدم معرّف BlueBubbles أو سلسلة الهدف (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- إعداد قناة BlueBubbles الكامل موثّق في [BlueBubbles](/ar/channels/bluebubbles).

### iMessage

يشغّل OpenClaw الأمر `imsg rpc` ‏(JSON-RPC عبر stdio). لا حاجة إلى daemon أو منفذ.

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

- يتطلب Full Disk Access إلى قاعدة بيانات الرسائل.
- يُفضَّل استخدام أهداف `chat_id:<id>`. استخدم `imsg chats --limit 20` لسرد المحادثات.
- يمكن أن يشير `cliPath` إلى غلاف SSH؛ واضبط `remoteHost` (`host` أو `user@host`) لجلب المرفقات عبر SCP.
- يقيّد `attachmentRoots` و`remoteAttachmentRoots` مسارات المرفقات الواردة (الافتراضي: `/Users/*/Library/Messages/Attachments`).
- يستخدم SCP التحقق الصارم من مفتاح المضيف، لذا تأكد من أن مفتاح مضيف relay موجود مسبقًا في `~/.ssh/known_hosts`.
- يتيح `channels.imessage.configWrites` أو يمنع عمليات كتابة الإعدادات التي يطلقها iMessage.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات iMessage بجلسات ACP دائمة. استخدم معرّفًا موحّدًا أو هدف محادثة صريحًا (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).

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

- تستخدم مصادقة الرمز `accessToken`؛ وتستخدم مصادقة كلمة المرور `userId` + `password`.
- يوجّه `channels.matrix.proxy` حركة HTTP الخاصة بـ Matrix عبر وكيل HTTP(S) صريح. ويمكن للحسابات المسماة تجاوزه عبر `channels.matrix.accounts.<id>.proxy`.
- يسمح `channels.matrix.network.dangerouslyAllowPrivateNetwork` بخوادم homeserver الخاصة/الداخلية. ويُعد `proxy` وخيار الشبكة هذا عنصرَي تحكم مستقلين.
- يحدد `channels.matrix.defaultAccount` الحساب المفضل في إعدادات تعدد الحسابات.
- تكون القيمة الافتراضية لـ `channels.matrix.autoJoin` هي `off`، لذا تُتجاهل الغرف المدعو إليها ودعوات الرسائل الخاصة الجديدة حتى تضبط `autoJoin: "allowlist"` مع `autoJoinAllowlist` أو `autoJoin: "always"`.
- `channels.matrix.execApprovals`: تسليم موافقات exec الأصلية في Matrix وتفويض المعتمدين.
  - `enabled`: يمكن أن تكون `true` أو `false` أو `"auto"` (افتراضي). في الوضع التلقائي، تُفعَّل موافقات exec عندما يمكن حل المعتمدين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Matrix (مثل `@owner:example.org`) المسموح لهم بالموافقة على طلبات exec.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتمرير الموافقات لجميع الوكلاء.
  - `sessionFilter`: أنماط مفاتيح جلسات اختيارية (سلسلة جزئية أو regex).
  - `target`: المكان الذي تُرسل إليه مطالبات الموافقة. `"dm"` (افتراضي) أو `"channel"` (الغرفة الأصلية) أو `"both"`.
  - تجاوزات لكل حساب: `channels.matrix.accounts.<id>.execApprovals`.
- يتحكم `channels.matrix.dm.sessionScope` في كيفية تجميع الرسائل الخاصة في Matrix ضمن جلسات: `per-user` (افتراضي) يشارك حسب النظير الموجّه، بينما يعزل `per-room` كل غرفة رسائل خاصة.
- تستخدم مجسات الحالة في Matrix وعمليات البحث المباشر في الدليل سياسة الوكيل نفسها المستخدمة في حركة وقت التشغيل.
- إعداد Matrix الكامل وقواعد الاستهداف وأمثلة الإعداد موثقة في [Matrix](/ar/channels/matrix).

### Microsoft Teams

Microsoft Teams مدعوم بإضافة ويُضبط ضمن `channels.msteams`.

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

- مسارات المفاتيح الأساسية المغطاة هنا: `channels.msteams` و`channels.msteams.configWrites`.
- إعداد Teams الكامل (بيانات الاعتماد وwebhook وسياسة الرسائل الخاصة/المجموعات والتجاوزات لكل فريق/قناة) موثّق في [Microsoft Teams](/ar/channels/msteams).

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
- إعداد قناة IRC الكامل (المضيف/المنفذ/TLS/القنوات/قوائم السماح/اشتراط الإشارة) موثّق في [IRC](/ar/channels/irc).

### تعدد الحسابات (جميع القنوات)

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

- يُستخدم `default` عند حذف `accountId` (في CLI والتوجيه).
- لا تنطبق الرموز المميزة من البيئة إلا على الحساب **الافتراضي**.
- تنطبق إعدادات القناة الأساسية على جميع الحسابات ما لم تُتجاوز لكل حساب.
- استخدم `bindings[].match.accountId` لتوجيه كل حساب إلى وكيل مختلف.
- إذا أضفت حسابًا غير افتراضي عبر `openclaw channels add` (أو عبر إعداد القناة) بينما لا تزال تستخدم إعداد قناة أحادي الحساب على المستوى الأعلى، فسيقوم OpenClaw أولاً بترقية القيم الأحادية على المستوى الأعلى ذات النطاق المرتبط بالحساب إلى خريطة حسابات القناة حتى يستمر الحساب الأصلي في العمل. تنقل معظم القنوات هذه القيم إلى `channels.<channel>.accounts.default`؛ ويمكن لـ Matrix بدلًا من ذلك الحفاظ على هدف named/default موجود ومطابق.
- تظل الارتباطات الحالية الخاصة بالقناة فقط (من دون `accountId`) مطابقة للحساب الافتراضي؛ وتبقى الارتباطات ذات النطاق المرتبط بالحساب اختيارية.
- يقوم `openclaw doctor --fix` أيضًا بإصلاح الأشكال المختلطة عبر نقل القيم الأحادية على المستوى الأعلى ذات النطاق المرتبط بالحساب إلى الحساب المُرقّى المختار لتلك القناة. تستخدم معظم القنوات `accounts.default`؛ ويمكن لـ Matrix الحفاظ على هدف named/default موجود ومطابق بدلًا من ذلك.

### قنوات الإضافات الأخرى

تُضبط كثير من قنوات الإضافات على شكل `channels.<id>` وتكون موثقة في صفحات القنوات المخصصة لها (مثل Feishu وMatrix وLINE وNostr وZalo وNextcloud Talk وSynology Chat وTwitch).
راجع فهرس القنوات الكامل: [القنوات](/ar/channels).

### اشتراط الإشارة في محادثات المجموعات

تستخدم رسائل المجموعات افتراضيًا **اشتراط الإشارة** (إشارة عبر البيانات الوصفية أو أنماط regex الآمنة). وينطبق ذلك على محادثات المجموعات في WhatsApp وTelegram وDiscord وGoogle Chat وiMessage.

**أنواع الإشارات:**

- **الإشارات عبر البيانات الوصفية**: إشارات @-mention الأصلية على المنصة. يتم تجاهلها في وضع الدردشة الذاتية في WhatsApp.
- **أنماط النص**: أنماط regex آمنة في `agents.list[].groupChat.mentionPatterns`. يتم تجاهل الأنماط غير الصالحة والتكرار المتداخل غير الآمن.
- يُفرض اشتراط الإشارة فقط عندما يكون الاكتشاف ممكنًا (إشارات أصلية أو وجود نمط واحد على الأقل).

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

يضبط `messages.groupChat.historyLimit` القيمة الافتراضية العامة. ويمكن للقنوات التجاوز عبر `channels.<channel>.historyLimit` (أو لكل حساب). اضبط القيمة على `0` للتعطيل.

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

آلية الحل: تجاوز لكل رسالة خاصة → افتراضي الموفّر → بلا حد (الاحتفاظ بالجميع).

مدعوم في: `telegram` و`whatsapp` و`discord` و`slack` و`signal` و`imessage` و`msteams`.

#### وضع الدردشة الذاتية

أدرج رقمك الخاص في `allowFrom` لتفعيل وضع الدردشة الذاتية (يتجاهل إشارات @ الأصلية، ويرد فقط على أنماط النص):

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
    native: "auto", // register native commands when supported
    nativeSkills: "auto", // register native skill commands when supported
    text: true, // parse /commands in chat messages
    bash: false, // allow ! (alias: /bash)
    bashForegroundMs: 2000,
    config: false, // allow /config
    mcp: false, // allow /mcp
    plugins: false, // allow /plugins
    debug: false, // allow /debug
    restart: true, // allow /restart + gateway restart tool
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="تفاصيل الأوامر">

- تضبط هذه الكتلة أسطح الأوامر. لكتالوج الأوامر الحالي المضمن + المجمّع، راجع [أوامر الشرطة المائلة](/ar/tools/slash-commands).
- هذه الصفحة هي **مرجع لمفاتيح الإعدادات**، وليست كتالوج الأوامر الكامل. الأوامر المملوكة للقنوات/الإضافات مثل `/bot-ping` و`/bot-help` و`/bot-logs` الخاصة بـ QQ Bot، و`/card` الخاصة بـ LINE، و`/pair` الخاصة بإقران الأجهزة، و`/dreaming` الخاصة بالذاكرة، و`/phone` الخاصة بالتحكم بالهاتف، و`/voice` الخاصة بـ Talk، موثقة في صفحات القنوات/الإضافات الخاصة بها بالإضافة إلى [أوامر الشرطة المائلة](/ar/tools/slash-commands).
- يجب أن تكون الأوامر النصية رسائل **مستقلة** مع `/` في البداية.
- يؤدي `native: "auto"` إلى تشغيل الأوامر الأصلية في Discord وTelegram، مع إبقاء Slack معطلاً.
- يؤدي `nativeSkills: "auto"` إلى تشغيل أوامر Skills الأصلية في Discord وTelegram، مع إبقاء Slack معطلاً.
- تجاوز لكل قناة: `channels.discord.commands.native` (منطقي أو `"auto"`). تؤدي القيمة `false` إلى مسح الأوامر المسجلة سابقًا.
- تجاوز تسجيل Skills الأصلية لكل قناة باستخدام `channels.<provider>.commands.nativeSkills`.
- يضيف `channels.telegram.customCommands` إدخالات إضافية إلى قائمة بوت Telegram.
- يفعّل `bash: true` الأمر `! <cmd>` لصدفة المضيف. ويتطلب `tools.elevated.enabled` وأن يكون المرسل ضمن `tools.elevated.allowFrom.<channel>`.
- يفعّل `config: true` الأمر `/config` (لقراءة/كتابة `openclaw.json`). بالنسبة إلى عملاء Gateway `chat.send`، تتطلب عمليات الكتابة الدائمة عبر `/config set|unset` أيضًا `operator.admin`؛ بينما يبقى `/config show` للقراءة فقط متاحًا لعملاء المشغل العاديين ذوي نطاق الكتابة.
- يفعّل `mcp: true` الأمر `/mcp` لإعداد خادم MCP الذي يديره OpenClaw ضمن `mcp.servers`.
- يفعّل `plugins: true` الأمر `/plugins` لاكتشاف الإضافات وتثبيتها وعناصر التحكم في تفعيلها/تعطيلها.
- يتحكم `channels.<provider>.configWrites` في عمليات تعديل الإعدادات لكل قناة (الافتراضي: true).
- بالنسبة إلى القنوات متعددة الحسابات، يتحكم `channels.<provider>.accounts.<id>.configWrites` أيضًا في عمليات الكتابة التي تستهدف ذلك الحساب (مثل `/allowlist --config --account <id>` أو `/config set channels.<provider>.accounts.<id>...`).
- يؤدي `restart: false` إلى تعطيل `/restart` وإجراءات أداة إعادة تشغيل Gateway. القيمة الافتراضية: `true`.
- `ownerAllowFrom` هو قائمة السماح الصريحة للمالك الخاصة بالأوامر/الأدوات الحصرية للمالك. وهو منفصل عن `allowFrom`.
- يؤدي `ownerDisplay: "hash"` إلى تجزئة معرّفات المالك في موجّه النظام. اضبط `ownerDisplaySecret` للتحكم في التجزئة.
- `allowFrom` يكون لكل موفّر. عند ضبطه، يصبح هو **مصدر التفويض الوحيد** (ويتم تجاهل قوائم سماح القنوات/الاقتران و`useAccessGroups`).
- يتيح `useAccessGroups: false` للأوامر تجاوز سياسات مجموعات الوصول عندما لا يكون `allowFrom` مضبوطًا.
- خريطة وثائق الأوامر:
  - الكتالوج المضمن + المجمّع: [أوامر الشرطة المائلة](/ar/tools/slash-commands)
  - أسطح الأوامر الخاصة بكل قناة: [القنوات](/ar/channels)
  - أوامر QQ Bot: [QQ Bot](/ar/channels/qqbot)
  - أوامر الاقتران: [الاقتران](/ar/channels/pairing)
  - أمر البطاقة في LINE: [LINE](/ar/channels/line)
  - dreaming في الذاكرة: [Dreaming](/ar/concepts/dreaming)

</Accordion>

---

## القيم الافتراضية للوكلاء

### `agents.defaults.workspace`

الافتراضي: `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

جذر المستودع الاختياري المعروض في سطر Runtime في موجّه النظام. إذا لم يتم ضبطه، يكتشفه OpenClaw تلقائيًا عبر الصعود من مساحة العمل إلى الأعلى.

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

- احذف `agents.defaults.skills` للسماح غير المقيّد بـ Skills افتراضيًا.
- احذف `agents.list[].skills` لوراثة القيم الافتراضية.
- اضبط `agents.list[].skills: []` لعدم استخدام أي Skills.
- تكون القائمة غير الفارغة `agents.list[].skills` هي المجموعة النهائية لذلك الوكيل؛
  وهي لا تُدمج مع القيم الافتراضية.

### `agents.defaults.skipBootstrap`

يعطّل الإنشاء التلقائي لملفات bootstrap الخاصة بمساحة العمل (`AGENTS.md` و`SOUL.md` و`TOOLS.md` و`IDENTITY.md` و`USER.md` و`HEARTBEAT.md` و`BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

يتحكم في وقت حقن ملفات bootstrap الخاصة بمساحة العمل في موجّه النظام. القيمة الافتراضية: `"always"`.

- `"continuation-skip"`: تتجاوز أدوار المتابعة الآمنة (بعد اكتمال رد المساعد) إعادة حقن bootstrap لمساحة العمل، مما يقلل حجم الموجّه. وتظل عمليات heartbeat ومحاولات إعادة البناء بعد الضغط تعيد بناء السياق.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

الحد الأقصى للأحرف لكل ملف bootstrap في مساحة العمل قبل الاقتطاع. القيمة الافتراضية: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

الحد الأقصى الإجمالي للأحرف التي يتم حقنها عبر جميع ملفات bootstrap في مساحة العمل. القيمة الافتراضية: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

يتحكم في نص التحذير المرئي للوكيل عند اقتطاع سياق bootstrap.
القيمة الافتراضية: `"once"`.

- `"off"`: لا يحقن نص تحذير في موجّه النظام مطلقًا.
- `"once"`: يحقن التحذير مرة واحدة لكل توقيع اقتطاع فريد (موصى به).
- `"always"`: يحقن التحذير في كل تشغيل عندما يوجد اقتطاع.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

أقصى حجم بالبكسل لأطول ضلع في الصورة في كتل الصور ضمن السجل/الأدوات قبل استدعاءات الموفّر.
القيمة الافتراضية: `1200`.

تؤدي القيم الأقل عادةً إلى تقليل استخدام رموز الرؤية وحجم حمولة الطلب في التشغيلات الكثيفة بلقطات الشاشة.
وتحافظ القيم الأعلى على تفاصيل بصرية أكثر.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

المنطقة الزمنية لسياق موجّه النظام (وليس الطوابع الزمنية للرسائل). وتعود إلى المنطقة الزمنية للمضيف كقيمة احتياطية.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

تنسيق الوقت في موجّه النظام. القيمة الافتراضية: `auto` (تفضيل نظام التشغيل).

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

- يقبل `model` إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يضبط شكل السلسلة النموذج الأساسي فقط.
  - يضبط شكل الكائن النموذج الأساسي بالإضافة إلى نماذج التحويل عند الفشل المرتبة.
- يقبل `imageModel` إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم من قبل مسار أداة `image` بوصفه إعداد نموذج الرؤية الخاص بها.
  - ويُستخدم أيضًا كتوجيه احتياطي عندما لا يستطيع النموذج المحدد/الافتراضي قبول إدخال الصور.
- يقبل `imageGenerationModel` إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة إمكانية توليد الصور المشتركة وأي سطح أداة/إضافة مستقبلي يولّد صورًا.
  - القيم المعتادة: `google/gemini-3.1-flash-image-preview` لتوليد الصور الأصلي في Gemini، أو `fal/fal-ai/flux/dev` لـ fal، أو `openai/gpt-image-1` لـ OpenAI Images.
  - إذا اخترت موفّرًا/نموذجًا مباشرةً، فاضبط أيضًا مصادقة الموفّر/مفتاح API المطابق (على سبيل المثال `GEMINI_API_KEY` أو `GOOGLE_API_KEY` لـ `google/*`، و`OPENAI_API_KEY` لـ `openai/*`، و`FAL_KEY` لـ `fal/*`).
  - إذا لم يتم ضبطه، يظل بإمكان `image_generate` استنتاج قيمة افتراضية لموفّر مدعوم بالمصادقة. يحاول أولاً الموفّر الافتراضي الحالي، ثم بقية موفّري توليد الصور المسجلين بترتيب معرّفات الموفّرين.
- يقبل `musicGenerationModel` إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة إمكانية توليد الموسيقى المشتركة وأداة `music_generate` المضمنة.
  - القيم المعتادة: `google/lyria-3-clip-preview` أو `google/lyria-3-pro-preview` أو `minimax/music-2.5+`.
  - إذا لم يتم ضبطه، يظل بإمكان `music_generate` استنتاج قيمة افتراضية لموفّر مدعوم بالمصادقة. يحاول أولاً الموفّر الافتراضي الحالي، ثم بقية موفّري توليد الموسيقى المسجلين بترتيب معرّفات الموفّرين.
  - إذا اخترت موفّرًا/نموذجًا مباشرةً، فاضبط أيضًا مصادقة الموفّر/مفتاح API المطابق.
- يقبل `videoGenerationModel` إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة إمكانية توليد الفيديو المشتركة وأداة `video_generate` المضمنة.
  - القيم المعتادة: `qwen/wan2.6-t2v` أو `qwen/wan2.6-i2v` أو `qwen/wan2.6-r2v` أو `qwen/wan2.6-r2v-flash` أو `qwen/wan2.7-r2v`.
  - إذا لم يتم ضبطه، يظل بإمكان `video_generate` استنتاج قيمة افتراضية لموفّر مدعوم بالمصادقة. يحاول أولاً الموفّر الافتراضي الحالي، ثم بقية موفّري توليد الفيديو المسجلين بترتيب معرّفات الموفّرين.
  - إذا اخترت موفّرًا/نموذجًا مباشرةً، فاضبط أيضًا مصادقة الموفّر/مفتاح API المطابق.
  - يدعم موفّر توليد الفيديو Qwen المضمن ما يصل إلى فيديو خرج واحد، وصورة إدخال واحدة، و4 فيديوهات إدخال، ومدة قدرها 10 ثوانٍ، وخيارات على مستوى الموفّر هي `size` و`aspectRatio` و`resolution` و`audio` و`watermark`.
- يقبل `pdfModel` إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة أداة `pdf` لتوجيه النموذج.
  - إذا لم يتم ضبطه، تعود أداة PDF إلى `imageModel`، ثم إلى نموذج الجلسة/النموذج الافتراضي الذي تم حله.
- `pdfMaxBytesMb`: حد حجم PDF الافتراضي لأداة `pdf` عندما لا يتم تمرير `maxBytesMb` وقت الاستدعاء.
- `pdfMaxPages`: الحد الأقصى الافتراضي للصفحات التي تُؤخذ في الاعتبار بواسطة وضع الاستخراج الاحتياطي في أداة `pdf`.
- `verboseDefault`: مستوى verbose الافتراضي للوكلاء. القيم: `"off"` و`"on"` و`"full"`. الافتراضي: `"off"`.
- `elevatedDefault`: مستوى الإخراج المرتفع الافتراضي للوكلاء. القيم: `"off"` و`"on"` و`"ask"` و`"full"`. الافتراضي: `"on"`.
- `model.primary`: الصيغة `provider/model` (مثل `openai/gpt-5.4`). إذا حذفت الموفّر، يحاول OpenClaw أولاً اسمًا مستعارًا، ثم موفّرًا مضبوطًا فريدًا يطابق معرّف النموذج هذا تمامًا، وبعد ذلك فقط يعود إلى الموفّر الافتراضي المضبوط (سلوك توافق قديم ومُهمل، لذا يُفضّل استخدام `provider/model` الصريح). إذا لم يعد ذلك الموفّر يوفّر النموذج الافتراضي المضبوط، يعود OpenClaw إلى أول موفّر/نموذج مضبوط بدلًا من إظهار قيمة افتراضية قديمة لموفّر تمت إزالته.
- `models`: كتالوج النماذج المضبوط وقائمة السماح الخاصة بـ `/model`. يمكن أن يتضمن كل إدخال `alias` (اختصار) و`params` (خاصة بالموفّر، مثل `temperature` و`maxTokens` و`cacheRetention` و`context1m`).
- `params`: معلمات الموفّر الافتراضية العامة التي تُطبَّق على جميع النماذج. تُضبط في `agents.defaults.params` (مثل `{ cacheRetention: "long" }`).
- أسبقية دمج `params` (في الإعدادات): يتم تجاوز `agents.defaults.params` (الأساس العام) بواسطة `agents.defaults.models["provider/model"].params` (لكل نموذج)، ثم تتجاوز `agents.list[].params` (للوكيل المطابق بالمعرّف) حسب المفتاح. راجع [Prompt Caching](/ar/reference/prompt-caching) للتفاصيل.
- أدوات كتابة الإعدادات التي تعدّل هذه الحقول (مثل `/models set` و`/models set-image` وأوامر إضافة/إزالة القيم الاحتياطية) تحفظ الصيغة القياسية للكائن وتحافظ على قوائم القيم الاحتياطية الحالية متى أمكن.
- `maxConcurrent`: الحد الأقصى للتشغيلات المتوازية للوكلاء عبر الجلسات (مع بقاء كل جلسة متسلسلة بحد ذاتها). الافتراضي: 4.

**اختصارات الأسماء المستعارة المضمنة** (لا تنطبق إلا عندما يكون النموذج موجودًا في `agents.defaults.models`):

| الاسم المستعار      | النموذج                                |
| ------------------- | -------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`            |
| `sonnet`            | `anthropic/claude-sonnet-4-6`          |
| `gpt`               | `openai/gpt-5.4`                       |
| `gpt-mini`          | `openai/gpt-5.4-mini`                  |
| `gpt-nano`          | `openai/gpt-5.4-nano`                  |
| `gemini`            | `google/gemini-3.1-pro-preview`        |
| `gemini-flash`      | `google/gemini-3-flash-preview`        |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

تفوز أسماؤك المستعارة المضبوطة دائمًا على القيم الافتراضية.

تفعّل نماذج Z.AI GLM-4.x وضع التفكير تلقائيًا ما لم تضبط `--thinking off` أو تحدد بنفسك `agents.defaults.models["zai/<model>"].params.thinking`.
تفعّل نماذج Z.AI القيمة `tool_stream` افتراضيًا لبث استدعاء الأدوات. اضبط `agents.defaults.models["zai/<model>"].params.tool_stream` على `false` لتعطيلها.
تستخدم نماذج Anthropic Claude 4.6 القيمة الافتراضية `adaptive` للتفكير عندما لا يتم ضبط مستوى تفكير صريح.

### `agents.defaults.cliBackends`

واجهات CLI خلفية اختيارية لتشغيلات النص فقط الاحتياطية (من دون استدعاءات أدوات). وهي مفيدة كنسخة احتياطية عند فشل موفّري API.

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
        "my-cli": {
          command: "my-cli",
          args: ["--json"],
          output: "json",
          modelArg: "--model",
          sessionArg: "--session",
          sessionMode: "existing",
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
        },
      },
    },
  },
}
```

- واجهات CLI الخلفية مخصّصة للنص أولاً؛ وتكون الأدوات معطلة دائمًا.
- الجلسات مدعومة عندما يتم ضبط `sessionArg`.
- تمرير الصور مدعوم عندما يقبل `imageArg` مسارات الملفات.

### `agents.defaults.systemPromptOverride`

استبدل موجّه النظام الكامل الذي يجمعه OpenClaw بسلسلة ثابتة. يتم ضبطه على المستوى الافتراضي (`agents.defaults.systemPromptOverride`) أو لكل وكيل (`agents.list[].systemPromptOverride`). تكون قيم كل وكيل ذات أسبقية أعلى؛ ويتم تجاهل القيمة الفارغة أو التي تحتوي على مسافات بيضاء فقط. وهو مفيد لتجارب الموجّهات المضبوطة.

```json5
{
  agents: {
    defaults: {
      systemPromptOverride: "You are a helpful assistant.",
    },
  },
}
```

### `agents.defaults.heartbeat`

تشغيلات heartbeat الدورية.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m disables
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // default: true; false omits the Heartbeat section from the system prompt
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

- `every`: سلسلة مدة (ms/s/m/h). الافتراضي: `30m` (مصادقة مفتاح API) أو `1h` (مصادقة OAuth). اضبطها على `0m` للتعطيل.
- `includeSystemPromptSection`: عند ضبطها على false، تُحذف فقرة Heartbeat من موجّه النظام ويُتجاوز حقن `HEARTBEAT.md` ضمن سياق bootstrap. الافتراضي: `true`.
- `suppressToolErrorWarnings`: عند ضبطه على true، تُخفى حمولات تحذير أخطاء الأدوات أثناء تشغيلات heartbeat.
- `directPolicy`: سياسة التسليم المباشر/الرسائل الخاصة. تسمح `allow` (افتراضي) بالتسليم إلى الأهداف المباشرة. وتمنع `block` هذا التسليم وتُصدر `reason=dm-blocked`.
- `lightContext`: عند ضبطه على true، تستخدم تشغيلات heartbeat سياق bootstrap خفيفًا وتحتفظ فقط بالملف `HEARTBEAT.md` من ملفات bootstrap الخاصة بمساحة العمل.
- `isolatedSession`: عند ضبطه على true، يتم تشغيل كل heartbeat في جلسة جديدة من دون سجل محادثة سابق. وهو نفس نمط العزل الخاص بـ cron `sessionTarget: "isolated"`. ويقلل تكلفة الرموز لكل heartbeat من نحو 100 ألف إلى نحو 2-5 آلاف رمز.
- لكل وكيل: اضبط `agents.list[].heartbeat`. عندما يعرّف أي وكيل `heartbeat`، فإن **هؤلاء الوكلاء فقط** هم الذين يشغّلون heartbeat.
- تنفّذ تشغيلات heartbeat أدوار وكيل كاملة — الفواصل الأقصر تستهلك مزيدًا من الرموز.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // id of a registered compaction provider plugin (optional)
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

- `mode`: إما `default` أو `safeguard` (تلخيص مجزأ للسجلات الطويلة). راجع [Compaction](/ar/concepts/compaction).
- `provider`: معرّف إضافة موفّر compaction مسجلة. عند ضبطه، يتم استدعاء `summarize()` الخاصة بالموفّر بدلًا من التلخيص المضمن عبر LLM. ويعود إلى التلخيص المضمن عند الفشل. يؤدي ضبط موفّر إلى فرض `mode: "safeguard"`. راجع [Compaction](/ar/concepts/compaction).
- `timeoutSeconds`: الحد الأقصى بالثواني المسموح لعملية compaction واحدة قبل أن يُلغيها OpenClaw. الافتراضي: `900`.
- `identifierPolicy`: `strict` (افتراضي) أو `off` أو `custom`. تضيف القيمة `strict` إرشادات مضمنة للاحتفاظ بالمعرّفات المعتمة قبل تلخيص compaction.
- `identifierInstructions`: نص مخصص اختياري للحفاظ على المعرّفات، يُستخدم عندما تكون `identifierPolicy=custom`.
- `postCompactionSections`: أسماء أقسام H2/H3 اختيارية من `AGENTS.md` لإعادة حقنها بعد compaction. القيم الافتراضية هي `["Session Startup", "Red Lines"]`؛ واضبط `[]` لتعطيل إعادة الحقن. عند عدم ضبطها أو عند ضبطها صراحةً على هذا الزوج الافتراضي، يتم أيضًا قبول العناوين القديمة `Every Session`/`Safety` كقيمة احتياطية قديمة.
- `model`: تجاوز اختياري بصيغة `provider/model-id` لتلخيص compaction فقط. استخدمه عندما ينبغي أن تحتفظ الجلسة الرئيسية بنموذج معين بينما تعمل ملخصات compaction على نموذج آخر؛ وعند عدم ضبطه، يستخدم compaction النموذج الأساسي للجلسة.
- `notifyUser`: عند ضبطه على `true`، يرسل إشعارًا موجزًا إلى المستخدم عند بدء compaction (مثل: "Compacting context..."). وهو معطّل افتراضيًا للحفاظ على صمت compaction.
- `memoryFlush`: دور وكيل صامت قبل compaction التلقائي لتخزين الذكريات الدائمة. يتم تجاوزه عندما تكون مساحة العمل للقراءة فقط.

### `agents.defaults.contextPruning`

يقوم بتقليم **نتائج الأدوات القديمة** من السياق الموجود في الذاكرة قبل إرسالها إلى LLM. ولا **يعدّل** سجل الجلسة على القرص.

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

- يفعّل `mode: "cache-ttl"` تمريرات التقليم.
- يتحكم `ttl` في عدد المرات التي يمكن فيها تشغيل التقليم مرة أخرى (بعد آخر لمس لذاكرة التخزين المؤقت).
- يقوم التقليم أولًا باقتطاع نتائج الأدوات كبيرة الحجم اقتطاعًا خفيفًا، ثم يمسح نتائج الأدوات الأقدم مسحًا كاملًا عند الحاجة.

**الاقتطاع الخفيف** يحتفظ بالبداية + النهاية ويُدرج `...` في المنتصف.

**المسح الكامل** يستبدل نتيجة الأداة بالكامل بالنص النائب.

ملاحظات:

- لا يتم اقتطاع/مسح كتل الصور مطلقًا.
- تعتمد النِّسب على عدد الأحرف (تقريبية)، وليست أعداد رموز دقيقة.
- إذا كان عدد رسائل المساعد أقل من `keepLastAssistants`، يتم تجاوز التقليم.

</Accordion>

راجع [تقليم الجلسة](/ar/concepts/session-pruning) لتفاصيل السلوك.

### البث على شكل كتل

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

- تتطلب القنوات غير Telegram ضبط `*.blockStreaming: true` صراحةً لتفعيل الردود على شكل كتل.
- تجاوزات القنوات: `channels.<channel>.blockStreamingCoalesce` (ومتغيرات كل حساب). تستخدم Signal وSlack وDiscord وGoogle Chat افتراضيًا `minChars: 1500`.
- `humanDelay`: توقف عشوائي بين الردود على شكل كتل. تعني `natural` = من 800 إلى 2500 مللي ثانية. التجاوز لكل وكيل: `agents.list[].humanDelay`.

راجع [Streaming](/ar/concepts/streaming) لتفاصيل السلوك + التجزئة.

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

- القيم الافتراضية: `instant` للمحادثات المباشرة/الإشارات، و`message` لمحادثات المجموعات غير المشار فيها.
- تجاوزات لكل جلسة: `session.typingMode` و`session.typingIntervalSeconds`.

راجع [مؤشرات الكتابة](/ar/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

إعدادات sandbox اختيارية للوكيل المضمن. راجع [Sandboxing](/ar/gateway/sandboxing) للدليل الكامل.

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

<Accordion title="تفاصيل Sandbox">

**الخلفية:**

- `docker`: وقت تشغيل Docker محلي (افتراضي)
- `ssh`: وقت تشغيل بعيد عام مدعوم عبر SSH
- `openshell`: وقت تشغيل OpenShell

عند اختيار `backend: "openshell"`، تنتقل الإعدادات الخاصة بوقت التشغيل إلى
`plugins.entries.openshell.config`.

**إعدادات خلفية SSH:**

- `target`: هدف SSH بصيغة `user@host[:port]`
- `command`: أمر عميل SSH (الافتراضي: `ssh`)
- `workspaceRoot`: جذر بعيد مطلق يُستخدم لمساحات العمل لكل نطاق
- `identityFile` / `certificateFile` / `knownHostsFile`: ملفات محلية موجودة يمررها OpenClaw إلى OpenSSH
- `identityData` / `certificateData` / `knownHostsData`: محتويات مضمنة أو SecretRef يقوم OpenClaw بتحويلها إلى ملفات مؤقتة في وقت التشغيل
- `strictHostKeyChecking` / `updateHostKeys`: عناصر تحكم سياسة مفاتيح المضيف في OpenSSH

**أسبقية مصادقة SSH:**

- تكون `identityData` ذات أسبقية على `identityFile`
- تكون `certificateData` ذات أسبقية على `certificateFile`
- تكون `knownHostsData` ذات أسبقية على `knownHostsFile`
- يتم حل قيم `*Data` المدعومة بـ SecretRef من اللقطة النشطة لوقت تشغيل الأسرار قبل بدء جلسة sandbox

**سلوك خلفية SSH:**

- يزرع مساحة العمل البعيدة مرة واحدة بعد الإنشاء أو إعادة الإنشاء
- ثم يحافظ على مساحة عمل SSH البعيدة بوصفها المصدر القياسي
- يوجّه `exec` وأدوات الملفات ومسارات الوسائط عبر SSH
- لا يزامن التغييرات البعيدة مرة أخرى إلى المضيف تلقائيًا
- لا يدعم حاويات المتصفح ضمن sandbox

**وصول مساحة العمل:**

- `none`: مساحة عمل sandbox لكل نطاق ضمن `~/.openclaw/sandboxes`
- `ro`: مساحة عمل sandbox عند `/workspace`، مع تركيب مساحة عمل الوكيل للقراءة فقط عند `/agent`
- `rw`: تركيب مساحة عمل الوكيل قراءة/كتابة عند `/workspace`

**النطاق:**

- `session`: حاوية + مساحة عمل لكل جلسة
- `agent`: حاوية + مساحة عمل واحدة لكل وكيل (افتراضي)
- `shared`: حاوية ومساحة عمل مشتركتان (من دون عزل بين الجلسات)

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

- `mirror`: زرع البعيد من المحلي قبل التنفيذ، ثم المزامنة العكسية بعد التنفيذ؛ وتبقى مساحة العمل المحلية هي المصدر القياسي
- `remote`: زرع البعيد مرة واحدة عند إنشاء sandbox، ثم الإبقاء على مساحة العمل البعيدة بوصفها المصدر القياسي

في وضع `remote`، لا تتم مزامنة التعديلات المحلية على المضيف التي تتم خارج OpenClaw إلى sandbox تلقائيًا بعد خطوة الزرع.
يكون النقل عبر SSH إلى OpenShell sandbox، لكن الإضافة هي التي تملك دورة حياة sandbox والمزامنة المرآتية الاختيارية.

يُنفَّذ `setupCommand` مرة واحدة بعد إنشاء الحاوية (عبر `sh -lc`). ويتطلب اتصال شبكة خارجيًا، وجذرًا قابلاً للكتابة، ومستخدم root.

**تستخدم الحاويات افتراضيًا `network: "none"`** — اضبطها على `"bridge"` (أو شبكة bridge مخصصة) إذا كان الوكيل يحتاج إلى وصول صادر.
القيمة `"host"` محظورة. كما أن `"container:<id>"` محظورة افتراضيًا ما لم تضبط صراحةً
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` (وضع طارئ).

تُجهَّز **المرفقات الواردة** ضمن `media/inbound/*` في مساحة العمل النشطة.

يقوم `docker.binds` بتركيب أدلة مضيف إضافية؛ ويتم دمج التركيبات العامة وتلك الخاصة بكل وكيل.

**متصفح sandbox** (`sandbox.browser.enabled`): Chromium + CDP داخل حاوية. يتم حقن URL الخاص بـ noVNC في موجّه النظام. ولا يتطلب `browser.enabled` في `openclaw.json`.
يستخدم وصول المراقبة عبر noVNC مصادقة VNC افتراضيًا، ويُصدر OpenClaw URL برمز مميز قصير العمر (بدلًا من كشف كلمة المرور في الرابط المشترك).

- يؤدي `allowHostControl: false` (الافتراضي) إلى حظر الجلسات المعزولة ضمن sandbox من استهداف متصفح المضيف.
- تكون القيمة الافتراضية لـ `network` هي `openclaw-sandbox-browser` (شبكة bridge مخصصة). اضبطها على `bridge` فقط عندما تريد صراحةً اتصال bridge عامًا.
- يقيّد `cdpSourceRange` اختياريًا دخول CDP عند حافة الحاوية إلى نطاق CIDR (مثل `172.21.0.1/32`).
- يقوم `sandbox.browser.binds` بتركيب أدلة مضيف إضافية داخل حاوية متصفح sandbox فقط. وعند ضبطه (بما في ذلك `[]`) فإنه يستبدل `docker.binds` لحاوية المتصفح.
- تُعرَّف القيم الافتراضية للتشغيل في `scripts/sandbox-browser-entrypoint.sh` وتُضبط لمضيفي الحاويات:
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
  - `--disable-extensions` (مفعّلة افتراضيًا)
  - تكون `--disable-3d-apis` و`--disable-software-rasterizer` و`--disable-gpu`
    مفعلة افتراضيًا، ويمكن تعطيلها عبر
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` إذا كان استخدام WebGL/الرسوم ثلاثية الأبعاد يتطلب ذلك.
  - يعيد `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` تفعيل الإضافات إذا كان سير عملك
    يعتمد عليها.
  - يمكن تغيير `--renderer-process-limit=2` عبر
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`؛ اضبطها على `0` لاستخدام
    الحد الافتراضي للعمليات في Chromium.
  - بالإضافة إلى `--no-sandbox` و`--disable-setuid-sandbox` عند تفعيل `noSandbox`.
  - القيم الافتراضية هي خط الأساس لصورة الحاوية؛ استخدم صورة متصفح مخصصة مع
    entrypoint مخصص لتغيير القيم الافتراضية للحاوية.

</Accordion>

تعمل ميزة sandbox الخاصة بالمتصفح و`sandbox.docker.binds` مع Docker فقط.

أنشئ الصور:

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
- `default`: عند ضبط أكثر من وكيل على هذه القيمة، يفوز الأول (مع تسجيل تحذير). وإذا لم يتم ضبط أي منها، يكون أول إدخال في القائمة هو الافتراضي.
- `model`: يتجاوز شكل السلسلة `primary` فقط؛ بينما يتجاوز شكل الكائن `{ primary, fallbacks }` كليهما (`[]` يعطّل القيم الاحتياطية العامة). تظل مهام Cron التي تتجاوز `primary` فقط ترث القيم الاحتياطية الافتراضية ما لم تضبط `fallbacks: []`.
- `params`: معلمات بث لكل وكيل تُدمج فوق إدخال النموذج المحدد في `agents.defaults.models`. استخدمها لتجاوزات خاصة بالوكيل مثل `cacheRetention` أو `temperature` أو `maxTokens` من دون تكرار كتالوج النماذج بالكامل.
- `skills`: قائمة سماح اختيارية لـ Skills لكل وكيل. إذا حُذفت، يرث الوكيل `agents.defaults.skills` عند ضبطها؛ وتستبدل القائمة الصريحة القيم الافتراضية بدلًا من دمجها، وتعني `[]` عدم وجود Skills.
- `thinkingDefault`: مستوى التفكير الافتراضي الاختياري لكل وكيل (`off | minimal | low | medium | high | xhigh | adaptive`). يتجاوز `agents.defaults.thinkingDefault` لهذا الوكيل عندما لا يكون هناك تجاوز لكل رسالة أو لكل جلسة.
- `reasoningDefault`: تجاوز اختياري لكل وكيل لرؤية reasoning (`on | off | stream`). ويُطبّق عندما لا يكون هناك تجاوز لكل رسالة أو لكل جلسة.
- `fastModeDefault`: القيمة الافتراضية الاختيارية لوضع fast لكل وكيل (`true | false`). وتُطبّق عندما لا يكون هناك تجاوز لكل رسالة أو لكل جلسة.
- `runtime`: واصف runtime اختياري لكل وكيل. استخدم `type: "acp"` مع الإعدادات الافتراضية لـ `runtime.acp` (`agent` و`backend` و`mode` و`cwd`) عندما ينبغي أن يستخدم الوكيل جلسات harness من ACP افتراضيًا.
- `identity.avatar`: مسار نسبي إلى مساحة العمل، أو URL من نوع `http(s)`، أو URI من نوع `data:`.
- تستمد `identity` القيم الافتراضية: `ackReaction` من `emoji`، و`mentionPatterns` من `name`/`emoji`.
- `subagents.allowAgents`: قائمة سماح لمعرّفات الوكلاء لـ `sessions_spawn` (`["*"]` = أي وكيل؛ الافتراضي: الوكيل نفسه فقط).
- حاجز وراثة sandbox: إذا كانت جلسة الطالب داخل sandbox، فإن `sessions_spawn` يرفض الأهداف التي ستعمل خارج sandbox.
- `subagents.requireAgentId`: عند ضبطه على true، يمنع استدعاءات `sessions_spawn` التي تحذف `agentId` (ويفرض اختيارًا صريحًا للملف الشخصي؛ الافتراضي: false).

---

## التوجيه متعدد الوكلاء

شغّل عدة وكلاء معزولين داخل Gateway واحدة. راجع [الوكلاء المتعددون](/ar/concepts/multi-agent).

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

### حقول مطابقة الارتباط

- `type` (اختياري): `route` للتوجيه العادي (عند غياب النوع تكون القيمة الافتراضية route)، و`acp` لارتباطات محادثات ACP الدائمة.
- `match.channel` (مطلوب)
- `match.accountId` (اختياري؛ `*` = أي حساب؛ حذفه = الحساب الافتراضي)
- `match.peer` (اختياري؛ `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (اختياري؛ خاص بالقناة)
- `acp` (اختياري؛ فقط عند `type: "acp"`): `{ mode, label, cwd, backend }`

**ترتيب المطابقة الحتمي:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (مطابقة تامة، من دون peer/guild/team)
5. `match.accountId: "*"` (على مستوى القناة)
6. الوكيل الافتراضي

ضمن كل مستوى، يفوز أول إدخال مطابق في `bindings`.

بالنسبة إلى إدخالات `type: "acp"`، يحل OpenClaw المطابقة بحسب هوية المحادثة الدقيقة (`match.channel` + الحساب + `match.peer.id`) ولا يستخدم ترتيب طبقات ارتباطات route أعلاه.

### ملفات وصول لكل وكيل

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

<Accordion title="بدون وصول إلى نظام الملفات (مراسلة فقط)">

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

راجع [Sandbox والأدوات للوكلاء المتعددين](/ar/tools/multi-agent-sandbox-tools) لتفاصيل الأسبقية.

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

- **`scope`**: استراتيجية تجميع الجلسات الأساسية لسياقات محادثات المجموعات.
  - `per-sender` (افتراضي): يحصل كل مرسل على جلسة معزولة داخل سياق القناة.
  - `global`: يشترك جميع المشاركين في سياق القناة في جلسة واحدة (استخدمه فقط عندما يكون السياق المشترك مقصودًا).
- **`dmScope`**: كيفية تجميع الرسائل الخاصة.
  - `main`: تشترك جميع الرسائل الخاصة في الجلسة الرئيسية.
  - `per-peer`: عزل حسب معرّف المرسل عبر القنوات.
  - `per-channel-peer`: عزل لكل قناة + مرسل (موصى به لصناديق الوارد متعددة المستخدمين).
  - `per-account-channel-peer`: عزل لكل حساب + قناة + مرسل (موصى به لتعدد الحسابات).
- **`identityLinks`**: يربط المعرّفات القياسية بنظراء مسبوقين بموفّر لمشاركة الجلسات عبر القنوات.
- **`reset`**: سياسة إعادة التعيين الأساسية. يعيد `daily` التعيين عند `atHour` بالتوقيت المحلي؛ ويعيد `idle` التعيين بعد `idleMinutes`. وعند ضبط الاثنين معًا، تسري أول قيمة تنقضي.
- **`resetByType`**: تجاوزات حسب النوع (`direct` و`group` و`thread`). وتُقبل القيمة القديمة `dm` كاسم بديل لـ `direct`.
- **`parentForkMaxTokens`**: الحد الأقصى المسموح لـ `totalTokens` في الجلسة الأصلية عند إنشاء جلسة سلسلة متفرعة (الافتراضي `100000`).
  - إذا كانت قيمة `totalTokens` في الجلسة الأصلية أعلى من هذه القيمة، يبدأ OpenClaw جلسة سلسلة جديدة بدلًا من وراثة سجل المحادثة من الجلسة الأصلية.
  - اضبطها على `0` لتعطيل هذا الحاجز والسماح دائمًا بالتفرع من الأصل.
- **`mainKey`**: حقل قديم. يستخدم وقت التشغيل دائمًا `"main"` لحاوية الدردشة المباشرة الرئيسية.
- **`agentToAgent.maxPingPongTurns`**: الحد الأقصى لعدد أدوار الرد المتبادل بين الوكلاء أثناء تبادلات وكيل-إلى-وكيل (عدد صحيح، المجال: `0`–`5`). وتعطّل القيمة `0` تسلسل ping-pong.
- **`sendPolicy`**: يطابق حسب `channel` أو `chatType` (`direct|group|channel`، مع الاسم البديل القديم `dm`) أو `keyPrefix` أو `rawKeyPrefix`. يفوز أول رفض.
- **`maintenance`**: عناصر تحكم تنظيف مخزن الجلسات + الاحتفاظ.
  - `mode`: يؤدي `warn` إلى إصدار تحذيرات فقط؛ بينما يطبّق `enforce` التنظيف.
  - `pruneAfter`: حد العمر لإزالة الإدخالات القديمة (الافتراضي `30d`).
  - `maxEntries`: الحد الأقصى لعدد الإدخالات في `sessions.json` (الافتراضي `500`).
  - `rotateBytes`: تدوير `sessions.json` عندما يتجاوز هذا الحجم (الافتراضي `10mb`).
  - `resetArchiveRetention`: مدة الاحتفاظ بأرشيفات السجل `*.reset.<timestamp>`. وتعود افتراضيًا إلى `pruneAfter`؛ اضبطها على `false` للتعطيل.
  - `maxDiskBytes`: ميزانية قرص اختيارية لدليل الجلسات. في وضع `warn` يسجل تحذيرات؛ وفي وضع `enforce` يزيل أقدم العناصر/الجلسات أولًا.
  - `highWaterBytes`: الهدف الاختياري بعد تنظيف الميزانية. ويكون افتراضيًا `80%` من `maxDiskBytes`.
- **`threadBindings`**: القيم الافتراضية العامة لميزات الجلسات المرتبطة بالسلاسل.
  - `enabled`: مفتاح رئيسي افتراضي (يمكن للموفّرين تجاوزه؛ ويستخدم Discord القيمة `channels.discord.threadBindings.enabled`)
  - `idleHours`: القيمة الافتراضية لإلغاء التركيز التلقائي بعد عدم النشاط بالساعات (`0` للتعطيل؛ ويمكن للموفّرين التجاوز)
  - `maxAgeHours`: القيمة الافتراضية للحد الأقصى الصارم للعمر بالساعات (`0` للتعطيل؛ ويمكن للموفّرين التجاوز)

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

آلية الحل (الأكثر تحديدًا يفوز): الحساب → القناة → العام. تؤدي القيمة `""` إلى التعطيل وإيقاف التسلسل. وتشتق `"auto"` القيمة `[{identity.name}]`.

**متغيرات القالب:**

| المتغير          | الوصف                 | المثال                      |
| ---------------- | --------------------- | --------------------------- |
| `{model}`        | اسم النموذج المختصر   | `claude-opus-4-6`           |
| `{modelFull}`    | معرّف النموذج الكامل  | `anthropic/claude-opus-4-6` |
| `{provider}`     | اسم الموفّر           | `anthropic`                 |
| `{thinkingLevel}` | مستوى التفكير الحالي | `high` و`low` و`off`        |
| `{identity.name}` | اسم هوية الوكيل      | (مثل `"auto"`)              |

المتغيرات غير حساسة لحالة الأحرف. ويُعد `{think}` اسمًا بديلًا لـ `{thinkingLevel}`.

### تفاعل الإقرار

- تكون القيمة الافتراضية هي `identity.emoji` الخاصة بالوكيل النشط، وإلا `"👀"`. اضبط `""` للتعطيل.
- تجاوزات لكل قناة: `channels.<channel>.ackReaction` و`channels.<channel>.accounts.<id>.ackReaction`.
- ترتيب الحل: الحساب → القناة → `messages.ackReaction` → الاحتياطي من الهوية.
- النطاق: `group-mentions` (افتراضي) و`group-all` و`direct` و`all`.
- يؤدي `removeAckAfterReply` إلى إزالة الإقرار بعد الرد على Slack وDiscord وTelegram.
- يفعّل `messages.statusReactions.enabled` تفاعلات الحالة الدورية على Slack وDiscord وTelegram.
  في Slack وDiscord، يؤدي عدم الضبط إلى إبقاء تفاعلات الحالة مفعّلة عندما تكون تفاعلات الإقرار نشطة.
  في Telegram، اضبطه صراحةً على `true` لتفعيل تفاعلات الحالة الدورية.

### تأخير الإدخال الوارد

يجمع الرسائل النصية السريعة من المرسل نفسه في دور وكيل واحد. ويتم تفريغ الوسائط/المرفقات فورًا. وتتجاوز أوامر التحكم هذا التأخير.

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

- يتحكم `auto` في وضع TTS التلقائي الافتراضي: `off` أو `always` أو `inbound` أو `tagged`. ويمكن للأمر `/tts on|off` تجاوز التفضيلات المحلية، ويعرض `/tts status` الحالة الفعلية.
- يتجاوز `summaryModel` القيمة `agents.defaults.model.primary` للملخص التلقائي.
- يكون `modelOverrides` مفعّلًا افتراضيًا؛ وتكون القيمة الافتراضية لـ `modelOverrides.allowProvider` هي `false` (تفعيل اختياري).
- تعود مفاتيح API إلى `ELEVENLABS_API_KEY`/`XI_API_KEY` و`OPENAI_API_KEY` كقيم احتياطية.
- يتجاوز `openai.baseUrl` نقطة نهاية TTS الخاصة بـ OpenAI. ترتيب الحل هو: الإعدادات، ثم `OPENAI_TTS_BASE_URL`، ثم `https://api.openai.com/v1`.
- عندما يشير `openai.baseUrl` إلى نقطة نهاية غير تابعة لـ OpenAI، يتعامل OpenClaw معها على أنها خادم TTS متوافق مع OpenAI ويخفف التحقق من النموذج/الصوت.

---

## Talk

القيم الافتراضية لوضع Talk ‏(macOS/iOS/Android).

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

- يجب أن تطابق `talk.provider` مفتاحًا في `talk.providers` عند ضبط عدة موفّرين لـ Talk.
- مفاتيح Talk القديمة المسطحة (`talk.voiceId` و`talk.voiceAliases` و`talk.modelId` و`talk.outputFormat` و`talk.apiKey`) مخصّصة للتوافق فقط، ويتم ترحيلها تلقائيًا إلى `talk.providers.<provider>`.
- تعود معرّفات الأصوات إلى `ELEVENLABS_VOICE_ID` أو `SAG_VOICE_ID` كقيم احتياطية.
- تقبل `providers.*.apiKey` سلاسل نصية صريحة أو كائنات SecretRef.
- لا يُستخدم الاحتياطي `ELEVENLABS_API_KEY` إلا عندما لا يكون هناك مفتاح API مضبوط لـ Talk.
- يتيح `providers.*.voiceAliases` لتوجيهات Talk استخدام أسماء مألوفة.
- يتحكم `silenceTimeoutMs` في المدة التي ينتظرها وضع Talk بعد صمت المستخدم قبل إرسال النص المفرغ. ويؤدي عدم ضبطه إلى الاحتفاظ بنافذة التوقف الافتراضية للمنصة (`700 ms` على macOS وAndroid، و`900 ms` على iOS).

---

## الأدوات

### ملفات تعريف الأدوات

يضبط `tools.profile` قائمة سماح أساسية قبل `tools.allow`/`tools.deny`:

تضبط عملية الإعداد المحلية افتراضيًا التهيئات المحلية الجديدة على `tools.profile: "coding"` عندما لا تكون مضبوطة (مع الحفاظ على ملفات التعريف الصريحة الموجودة).

| ملف التعريف | يتضمن                                                                                                                        |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | `session_status` فقط                                                                                                          |
| `coding`    | `group:fs` و`group:runtime` و`group:web` و`group:sessions` و`group:memory` و`cron` و`image` و`image_generate` و`video_generate` |
| `messaging` | `group:messaging` و`sessions_list` و`sessions_history` و`sessions_send` و`session_status`                                      |
| `full`      | بلا تقييد (مثل عدم الضبط)                                                                                                     |

### مجموعات الأدوات

| المجموعة          | الأدوات                                                                                                                 |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`   | `exec` و`process` و`code_execution` (تُقبل `bash` كاسم بديل لـ `exec`)                                                |
| `group:fs`        | `read` و`write` و`edit` و`apply_patch`                                                                                 |
| `group:sessions`  | `sessions_list` و`sessions_history` و`sessions_send` و`sessions_spawn` و`sessions_yield` و`subagents` و`session_status` |
| `group:memory`    | `memory_search` و`memory_get`                                                                                          |
| `group:web`       | `web_search` و`x_search` و`web_fetch`                                                                                  |
| `group:ui`        | `browser` و`canvas`                                                                                                    |
| `group:automation` | `cron` و`gateway`                                                                                                     |
| `group:messaging` | `message`                                                                                                              |
| `group:nodes`     | `nodes`                                                                                                                |
| `group:agents`    | `agents_list`                                                                                                          |
| `group:media`     | `image` و`image_generate` و`video_generate` و`tts`                                                                     |
| `group:openclaw`  | جميع الأدوات المضمنة (باستثناء إضافات الموفّرين)                                                                      |

### `tools.allow` / `tools.deny`

سياسة السماح/المنع العامة للأدوات (يفوز المنع). غير حساسة لحالة الأحرف، وتدعم أحرف البدل `*`. وتُطبّق حتى عندما يكون Docker sandbox معطلاً.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

تقييد إضافي للأدوات لموفّرين أو نماذج محددة. الترتيب: ملف التعريف الأساسي → ملف تعريف الموفّر → السماح/المنع.

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

- لا يمكن للتجاوز لكل وكيل (`agents.list[].tools.elevated`) إلا أن يضيف مزيدًا من التقييد.
- يخزّن `/elevated on|off|ask|full` الحالة لكل جلسة؛ بينما تنطبق التوجيهات المضمنة على رسالة واحدة.
- يتجاوز `exec` المرتفع sandboxing ويستخدم مسار الإفلات المضبوط (`gateway` افتراضيًا، أو `node` عندما يكون هدف exec هو `node`).

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

تكون فحوصات أمان حلقات الأدوات **معطّلة افتراضيًا**. اضبط `enabled: true` لتفعيل الاكتشاف.
يمكن تعريف الإعدادات عمومًا في `tools.loopDetection` وتجاوزها لكل وكيل في `agents.list[].tools.loopDetection`.

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
- `warningThreshold`: عتبة النمط المتكرر من دون تقدم لإصدار التحذيرات.
- `criticalThreshold`: عتبة تكرار أعلى لحظر الحلقات الحرجة.
- `globalCircuitBreakerThreshold`: عتبة إيقاف صارم لأي تشغيل بلا تقدم.
- `detectors.genericRepeat`: التحذير عند تكرار استدعاءات الأداة نفسها/الوسائط نفسها.
- `detectors.knownPollNoProgress`: التحذير/الحظر عند أدوات الاستطلاع المعروفة (`process.poll` و`command_status` وما إلى ذلك).
- `detectors.pingPong`: التحذير/الحظر عند أنماط الأزواج المتناوبة من دون تقدم.
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

يضبط فهم الوسائط الواردة (الصورة/الصوت/الفيديو):

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

<Accordion title="حقول إدخال نموذج الوسائط">

**إدخال الموفّر** (`type: "provider"` أو محذوف):

- `provider`: معرّف موفّر API (`openai` أو `anthropic` أو `google`/`gemini` أو `groq`، إلخ.)
- `model`: تجاوز معرّف النموذج
- `profile` / `preferredProfile`: اختيار ملف `auth-profiles.json`

**إدخال CLI** (`type: "cli"`):

- `command`: الملف التنفيذي المراد تشغيله
- `args`: وسائط بقوالب (تدعم `{{MediaPath}}` و`{{Prompt}}` و`{{MaxChars}}` وما إلى ذلك)

**حقول مشتركة:**

- `capabilities`: قائمة اختيارية (`image` و`audio` و`video`). القيم الافتراضية: `openai`/`anthropic`/`minimax` ← صورة، و`google` ← صورة+صوت+فيديو، و`groq` ← صوت.
- `prompt` و`maxChars` و`maxBytes` و`timeoutSeconds` و`language`: تجاوزات لكل إدخال.
- عند الفشل يتم الرجوع إلى الإدخال التالي.

تتبع مصادقة الموفّر الترتيب القياسي: `auth-profiles.json` ← متغيرات البيئة ← `models.providers.*.apiKey`.

**حقول الإكمال غير المتزامن:**

- `asyncCompletion.directSend`: عند ضبطه على `true`، تحاول المهام المكتملة غير المتزامنة
  لـ `music_generate` و`video_generate` التسليم المباشر إلى القناة أولًا. القيمة الافتراضية: `false`
  (المسار القديم لإيقاظ جلسة الطالب/التسليم عبر النموذج).

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
- `agent`: أي جلسة تنتمي إلى معرّف الوكيل الحالي (وقد تشمل مستخدمين آخرين إذا كنت تشغّل جلسات لكل مرسل ضمن معرّف الوكيل نفسه).
- `all`: أي جلسة. ومع ذلك، يتطلب الاستهداف عبر الوكلاء `tools.agentToAgent`.
- تقييد sandbox: عندما تكون الجلسة الحالية داخل sandbox وتكون `agents.defaults.sandbox.sessionToolsVisibility="spawned"`، تُفرض القيمة `tree` على الرؤية حتى لو كانت `tools.sessions.visibility="all"`.

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

- المرفقات مدعومة فقط عند `runtime: "subagent"`. ويرفض runtime الخاص بـ ACP هذه المرفقات.
- تُحوَّل الملفات إلى مساحة عمل الابن ضمن `.openclaw/attachments/<uuid>/` مع ملف `.manifest.json`.
- يتم حجب محتوى المرفقات تلقائيًا من حفظ السجل.
- يتم التحقق من مدخلات Base64 بفحوصات صارمة للأبجدية/الحشو وحاجز لحجم ما قبل فك الترميز.
- تكون أذونات الملفات `0700` للأدلة و`0600` للملفات.
- يتبع التنظيف سياسة `cleanup`: يؤدي `delete` دائمًا إلى إزالة المرفقات؛ بينما يحتفظ بها `keep` فقط عندما تكون `retainOnSessionKeep: true`.

### `tools.experimental`

أعلام الأدوات المضمنة التجريبية. تكون معطّلة افتراضيًا ما لم تنطبق قاعدة تفعيل تلقائي خاصة بوقت تشغيل معيّن.

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

- `planTool`: يفعّل الأداة البنيوية التجريبية `update_plan` لتتبع الأعمال غير البسيطة متعددة الخطوات.
- الافتراضي: `false` للموفّرين غير OpenAI. وتُفعّلها تشغيلات OpenAI وOpenAI Codex تلقائيًا عند عدم ضبطها؛ اضبطها على `false` لتعطيل هذا التفعيل التلقائي.
- عند تفعيلها، يضيف موجّه النظام أيضًا إرشادات استخدام حتى لا يستخدمها النموذج إلا للأعمال الكبيرة ويحافظ على خطوة واحدة فقط في حالة `in_progress`.

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

- `model`: النموذج الافتراضي للوكلاء الفرعيين الذين يتم إنشاؤهم. وإذا لم يتم ضبطه، يرث الوكلاء الفرعيون نموذج المتصل.
- `allowAgents`: قائمة السماح الافتراضية لمعرّفات الوكلاء المستهدفة بواسطة `sessions_spawn` عندما لا يضبط الوكيل الطالب القيمة `subagents.allowAgents` الخاصة به (`["*"]` = أي وكيل؛ الافتراضي: الوكيل نفسه فقط).
- `runTimeoutSeconds`: المهلة الافتراضية (بالثواني) لـ `sessions_spawn` عندما يحذف استدعاء الأداة `runTimeoutSeconds`. وتعني القيمة `0` عدم وجود مهلة.
- سياسة الأدوات لكل وكيل فرعي: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## الموفّرون المخصصون وعناوين URL الأساسية

يستخدم OpenClaw كتالوج النماذج المضمن. أضف موفّرين مخصصين عبر `models.providers` في الإعدادات أو `~/.openclaw/agents/<agentId>/agent/models.json`.

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
- تجاوز جذر إعدادات الوكيل عبر `OPENCLAW_AGENT_DIR` (أو `PI_CODING_AGENT_DIR`، وهو اسم بديل قديم لمتغير البيئة).
- أسبقية الدمج لمعرّفات الموفّرين المتطابقة:
  - تفوز قيم `baseUrl` غير الفارغة في `models.json` الخاصة بالوكيل.
  - تفوز قيم `apiKey` غير الفارغة في الوكيل فقط عندما لا يكون `apiKey` لذلك الموفّر مُدارًا عبر SecretRef في سياق الإعدادات/ملف المصادقة الحالي.
  - يتم تحديث قيم `apiKey` للموفّرين المُدارين عبر SecretRef من مؤشرات المصدر (`ENV_VAR_NAME` لإشارات env، و`secretref-managed` لإشارات file/exec) بدلًا من حفظ الأسرار المحلولة.
  - يتم تحديث قيم ترويسات الموفّر المُدارة عبر SecretRef من مؤشرات المصدر (`secretref-env:ENV_VAR_NAME` لإشارات env، و`secretref-managed` لإشارات file/exec).
  - تعود قيم `apiKey`/`baseUrl` الفارغة أو المفقودة في الوكيل إلى `models.providers` في الإعدادات.
  - تستخدم القيم المتطابقة `contextWindow`/`maxTokens` للنموذج القيمة الأعلى بين الإعدادات الصريحة وقيم الكتالوج الضمنية.
  - تحافظ القيمة `contextTokens` المتطابقة للنموذج على حد runtime صريح عند وجوده؛ استخدمها لتقييد السياق الفعلي من دون تغيير بيانات النموذج الأصلية.
  - استخدم `models.mode: "replace"` عندما تريد من الإعدادات إعادة كتابة `models.json` بالكامل.
  - يكون حفظ المؤشرات معتمدًا على المصدر: تُكتب المؤشرات من اللقطة النشطة لإعدادات المصدر (قبل الحل)، وليس من قيم الأسرار المحلولة في وقت التشغيل.

### تفاصيل حقول الموفّر

- `models.mode`: سلوك كتالوج الموفّر (`merge` أو `replace`).
- `models.providers`: خريطة الموفّرين المخصصين، مفهرسة بحسب معرّف الموفّر.
- `models.providers.*.api`: مهيئ الطلبات (`openai-completions` أو `openai-responses` أو `anthropic-messages` أو `google-generative-ai`، إلخ).
- `models.providers.*.apiKey`: بيانات اعتماد الموفّر (يُفضّل SecretRef/الاستبدال عبر البيئة).
- `models.providers.*.auth`: استراتيجية المصادقة (`api-key` أو `token` أو `oauth` أو `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: بالنسبة إلى Ollama + `openai-completions`، يحقن `options.num_ctx` في الطلبات (الافتراضي: `true`).
- `models.providers.*.authHeader`: يفرض نقل بيانات الاعتماد في ترويسة `Authorization` عند الحاجة.
- `models.providers.*.baseUrl`: عنوان URL الأساسي لواجهة API في المنبع.
- `models.providers.*.headers`: ترويسات ثابتة إضافية لتوجيه الوكيل/المستأجر.
- `models.providers.*.request`: تجاوزات النقل لطلبات HTTP الخاصة بموفّر النموذج.
  - `request.headers`: ترويسات إضافية (تُدمج مع القيم الافتراضية للموفّر). تقبل القيم SecretRef.
  - `request.auth`: تجاوز استراتيجية المصادقة. الأوضاع: `"provider-default"` (استخدام المصادقة المضمنة للموفّر)، و`"authorization-bearer"` (مع `token`)، و`"header"` (مع `headerName` و`value` و`prefix` الاختياري).
  - `request.proxy`: تجاوز وكيل HTTP. الأوضاع: `"env-proxy"` (استخدام متغيرات البيئة `HTTP_PROXY`/`HTTPS_PROXY`) و`"explicit-proxy"` (مع `url`). ويقبل كلا الوضعين كائنًا فرعيًا اختياريًا `tls`.
  - `request.tls`: تجاوز TLS للاتصالات المباشرة. الحقول: `ca` و`cert` و`key` و`passphrase` (كلها تقبل SecretRef)، و`serverName` و`insecureSkipVerify`.
  - `request.allowPrivateNetwork`: عند ضبطه على `true`، يسمح باستخدام HTTPS إلى `baseUrl` عندما يحل DNS إلى نطاقات خاصة أو CGNAT أو نطاقات مشابهة، عبر حاجز جلب HTTP الخاص بالموفّر (تفعيل اختياري للمشغّل من أجل نقاط نهاية موثوقة مستضافة ذاتيًا ومتوافقة مع OpenAI). ويستخدم WebSocket الكائن `request` نفسه للترويسات/TLS، لكن ليس لحاجز SSRF هذا الخاص بالجلب. الافتراضي `false`.
- `models.providers.*.models`: إدخالات كتالوج نماذج صريحة للموفّر.
- `models.providers.*.models.*.contextWindow`: بيانات وصفية لنافذة سياق النموذج الأصلية.
- `models.providers.*.models.*.contextTokens`: حد سياق اختياري لوقت التشغيل. استخدمه عندما تريد ميزانية سياق فعلية أصغر من `contextWindow` الأصلية للنموذج.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: تلميح توافق اختياري. بالنسبة إلى `api: "openai-completions"` مع `baseUrl` غير فارغ وغير أصلي (المضيف ليس `api.openai.com`)، يفرض OpenClaw هذه القيمة على `false` في وقت التشغيل. أما `baseUrl` الفارغ/المحذوف فيُبقي سلوك OpenAI الافتراضي.
- `models.providers.*.models.*.compat.requiresStringContent`: تلميح توافق اختياري لنقاط نهاية الدردشة المتوافقة مع OpenAI والتي تدعم السلاسل النصية فقط. عندما تكون `true`، يقوم OpenClaw بتسطيح مصفوفات `messages[].content` النصية البحتة إلى سلاسل نصية عادية قبل إرسال الطلب.
- `plugins.entries.amazon-bedrock.config.discovery`: جذر إعدادات الاكتشاف التلقائي لـ Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: تشغيل/إيقاف الاكتشاف الضمني.
- `plugins.entries.amazon-bedrock.config.discovery.region`: منطقة AWS الخاصة بالاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: مرشح اختياري لمعرّف الموفّر من أجل اكتشاف موجّه.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: فترة الاستطلاع لتحديث الاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: نافذة السياق الاحتياطية للنماذج المكتشفة.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: الحد الاحتياطي الأقصى لرموز الإخراج للنماذج المكتشفة.

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

استخدم `cerebras/zai-glm-4.7` مع Cerebras؛ واستخدم `zai/glm-4.7` مع Z.AI المباشر.

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

اضبط `OPENCODE_API_KEY` (أو `OPENCODE_ZEN_API_KEY`). استخدم المراجع `opencode/...` لكتالوج Zen أو `opencode-go/...` لكتالوج Go. اختصار: `openclaw onboard --auth-choice opencode-zen` أو `openclaw onboard --auth-choice opencode-go`.

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

اضبط `ZAI_API_KEY`. تُقبل `z.ai/*` و`z-ai/*` كأسماء بديلة. اختصار: `openclaw onboard --auth-choice zai-api-key`.

- نقطة النهاية العامة: `https://api.z.ai/api/paas/v4`
- نقطة نهاية البرمجة (الافتراضية): `https://api.z.ai/api/coding/paas/v4`
- لنقطة النهاية العامة، عرّف موفّرًا مخصصًا مع تجاوز `baseUrl`.

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

بالنسبة إلى نقطة النهاية في الصين: `baseUrl: "https://api.moonshot.cn/v1"` أو `openclaw onboard --auth-choice moonshot-api-key-cn`.

تعلن نقاط نهاية Moonshot الأصلية عن توافق استخدام البث على ناقل
`openai-completions` المشترك، ويعتمد OpenClaw في ذلك على إمكانات نقطة النهاية
بدلًا من الاعتماد على معرّف الموفّر المضمن وحده.

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

متوافق مع Anthropic، وموفّر مضمن. اختصار: `openclaw onboard --auth-choice kimi-code-api-key`.

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

يجب أن يحذف عنوان URL الأساسي `/v1` (إذ إن عميل Anthropic يضيفه). اختصار: `openclaw onboard --auth-choice synthetic-api-key`.

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
تكون القيمة الافتراضية لكتالوج النماذج هي M2.7 فقط.
على مسار البث المتوافق مع Anthropic، يعطّل OpenClaw تفكير MiniMax
افتراضيًا ما لم تضبط `thinking` بنفسك صراحةً. ويعيد `/fast on` أو
`params.fastMode: true` كتابة `MiniMax-M2.7` إلى
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="النماذج المحلية (LM Studio)">

راجع [النماذج المحلية](/ar/gateway/local-models). باختصار: شغّل نموذجًا محليًا كبيرًا عبر LM Studio Responses API على عتاد قوي؛ واحتفظ بالنماذج المستضافة مدمجةً للاحتياط.

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

- `allowBundled`: قائمة سماح اختيارية لـ Skills المضمنة فقط (ولا تتأثر Skills المُدارة/الموجودة في مساحة العمل).
- `load.extraDirs`: جذور Skills مشتركة إضافية (أدنى أسبقية).
- `install.preferBrew`: عند ضبطه على true، تُفضَّل أدوات التثبيت عبر Homebrew عندما يكون `brew`
  متاحًا قبل الرجوع إلى أنواع مُثبّتات أخرى.
- `install.nodeManager`: تفضيل مُثبّت Node لمواصفات `metadata.openclaw.install`
  (`npm` | `pnpm` | `yarn` | `bun`).
- يؤدي `entries.<skillKey>.enabled: false` إلى تعطيل Skill حتى لو كانت مضمنة/مثبتة.
- `entries.<skillKey>.apiKey`: وسيلة مريحة للـ Skills التي تعلن متغير env أساسيًا (سلسلة نصية صريحة أو كائن SecretRef).

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

- يتم التحميل من `~/.openclaw/extensions` و`<workspace>/.openclaw/extensions` بالإضافة إلى `plugins.load.paths`.
- يقبل الاكتشاف إضافات OpenClaw الأصلية بالإضافة إلى حزم Codex المتوافقة وحزم Claude، بما في ذلك حزم Claude ذات التخطيط الافتراضي من دون manifest.
- **تتطلب تغييرات الإعدادات إعادة تشغيل Gateway.**
- `allow`: قائمة سماح اختيارية (لا يتم تحميل إلا الإضافات المدرجة). ويفوز `deny`.
- `plugins.entries.<id>.apiKey`: حقل مريح لمفتاح API على مستوى الإضافة (عند دعمه من الإضافة).
- `plugins.entries.<id>.env`: خريطة متغيرات env ضمن نطاق الإضافة.
- `plugins.entries.<id>.hooks.allowPromptInjection`: عندما تكون `false`، يحظر core القيمة `before_prompt_build` ويتجاهل الحقول المعدِّلة للموجّه من `before_agent_start` القديم، مع الحفاظ على `modelOverride` و`providerOverride` القديمين. وينطبق ذلك على hooks الخاصة بالإضافات الأصلية وأدلة hooks التي توفرها الحزم المدعومة.
- `plugins.entries.<id>.subagent.allowModelOverride`: يثق صراحةً بهذه الإضافة لطلب تجاوزات `provider` و`model` لكل تشغيل بالنسبة إلى تشغيلات الوكلاء الفرعيين في الخلفية.
- `plugins.entries.<id>.subagent.allowedModels`: قائمة سماح اختيارية لأهداف `provider/model` القياسية الخاصة بتجاوزات الوكلاء الفرعيين الموثوقة. استخدم `"*"` فقط عندما تريد عمدًا السماح بأي نموذج.
- `plugins.entries.<id>.config`: كائن إعدادات تعرّفه الإضافة (ويُتحقق منه عبر مخطط إضافة OpenClaw الأصلية عند توفّره).
- `plugins.entries.firecrawl.config.webFetch`: إعدادات موفّر جلب الويب Firecrawl.
  - `apiKey`: مفتاح API لـ Firecrawl (يقبل SecretRef). ويعود إلى `plugins.entries.firecrawl.config.webSearch.apiKey`، أو القيمة القديمة `tools.web.fetch.firecrawl.apiKey`، أو متغير البيئة `FIRECRAWL_API_KEY`.
  - `baseUrl`: عنوان URL الأساسي لواجهة API الخاصة بـ Firecrawl (الافتراضي: `https://api.firecrawl.dev`).
  - `onlyMainContent`: استخراج المحتوى الرئيسي فقط من الصفحات (الافتراضي: `true`).
  - `maxAgeMs`: أقصى عمر لذاكرة التخزين المؤقت بالمللي ثانية (الافتراضي: `172800000` / يومان).
  - `timeoutSeconds`: مهلة طلب الاستخلاص بالثواني (الافتراضي: `60`).
- `plugins.entries.xai.config.xSearch`: إعدادات xAI X Search ‏(بحث الويب Grok).
  - `enabled`: تفعيل موفّر X Search.
  - `model`: نموذج Grok المستخدم في البحث (مثل `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming`: إعدادات dreaming الخاصة بالذاكرة (تجريبية). راجع [Dreaming](/ar/concepts/dreaming) للمراحل والعتبات.
  - `enabled`: المفتاح الرئيسي لـ dreaming (الافتراضي `false`).
  - `frequency`: وتيرة cron لكل دورة dreaming كاملة (`"0 3 * * *"` افتراضيًا).
  - سياسة المراحل والعتبات هي تفاصيل تنفيذية (وليست مفاتيح إعدادات موجّهة للمستخدم).
- توجد إعدادات الذاكرة الكاملة في [مرجع إعدادات الذاكرة](/ar/reference/memory-config):
  - `agents.defaults.memorySearch.*`
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- يمكن لإضافات حِزم Claude المفعّلة أيضًا أن تساهم بقيم Pi افتراضية مضمنة من `settings.json`؛ ويطبّقها OpenClaw كإعدادات وكيل مُنظّفة، وليس كتصحيحات خام لإعدادات OpenClaw.
- `plugins.slots.memory`: اختر معرّف إضافة الذاكرة النشطة، أو `"none"` لتعطيل إضافات الذاكرة.
- `plugins.slots.contextEngine`: اختر معرّف إضافة محرك السياق النشط؛ وتكون القيمة الافتراضية `"legacy"` ما لم تثبّت محركًا آخر وتحدده.
- `plugins.installs`: بيانات تعريف التثبيت المُدارة من CLI والتي يستخدمها `openclaw plugins update`.
  - تتضمن `source` و`spec` و`sourcePath` و`installPath` و`version` و`resolvedName` و`resolvedVersion` و`resolvedSpec` و`integrity` و`shasum` و`resolvedAt` و`installedAt`.
  - تعامل مع `plugins.installs.*` على أنها حالة مُدارة؛ ويفضّل استخدام أوامر CLI بدلًا من التعديل اليدوي.

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

- يؤدي `evaluateEnabled: false` إلى تعطيل `act:evaluate` و`wait --fn`.
- تكون القيمة الافتراضية لـ `ssrfPolicy.dangerouslyAllowPrivateNetwork` هي `true` عند عدم ضبطها (نموذج شبكة موثوق).
- اضبط `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` لتصفح صارم يقتصر على الشبكات العامة.
- في الوضع الصارم، تخضع نقاط نهاية ملفات تعريف CDP البعيدة (`profiles.*.cdpUrl`) للحظر نفسه الخاص بالشبكات الخاصة أثناء فحوصات الوصول/الاكتشاف.
- لا يزال `ssrfPolicy.allowPrivateNetwork` مدعومًا كاسم بديل قديم.
- في الوضع الصارم، استخدم `ssrfPolicy.hostnameAllowlist` و`ssrfPolicy.allowedHostnames` للاستثناءات الصريحة.
- تكون الملفات التعريفية البعيدة في وضع الإرفاق فقط (بدء/إيقاف/إعادة تعيين معطلة).
- تقبل `profiles.*.cdpUrl` القيم `http://` و`https://` و`ws://` و`wss://`.
  استخدم HTTP(S) عندما تريد أن يكتشف OpenClaw المسار `/json/version`؛ واستخدم WS(S)
  عندما يزوّدك الموفّر بعنوان URL مباشر لـ DevTools WebSocket.
- تكون ملفات التعريف `existing-session` خاصة بالمضيف فقط وتستخدم Chrome MCP بدلًا من CDP.
- يمكن لملفات التعريف `existing-session` ضبط `userDataDir` لاستهداف ملف تعريف
  محدد لمتصفح مبني على Chromium مثل Brave أو Edge.
- تحافظ ملفات التعريف `existing-session` على حدود المسار الحالية في Chrome MCP:
  إجراءات تعتمد على اللقطات/المراجع بدل الاستهداف عبر محددات CSS، وخطافات
  تحميل ملف واحد، ومن دون تجاوزات لمهلات مربعات الحوار، ومن دون
  `wait --load networkidle`، أو `responsebody`، أو تصدير PDF، أو اعتراض التنزيلات،
  أو الإجراءات الدفعية.
- تعيّن ملفات التعريف المحلية المُدارة `openclaw` القيم `cdpPort` و`cdpUrl`
  تلقائيًا؛ ولا تضبط `cdpUrl` صراحةً إلا لـ CDP البعيد.
- ترتيب الاكتشاف التلقائي: المتصفح الافتراضي إذا كان مبنيًا على Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- خدمة التحكم: loopback فقط (ويُشتق المنفذ من `gateway.port`، والافتراضي `18791`).
- تضيف `extraArgs` أعلام تشغيل إضافية إلى بدء تشغيل Chromium المحلي (مثل
  `--disable-gpu` أو تحديد حجم النافذة أو أعلام التصحيح).

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

- `seamColor`: لون التمييز لواجهة التطبيق الأصلية (مثل تلوين فقاعة Talk Mode، إلخ).
- `assistant`: تجاوز هوية Control UI. وتعود القيمة الاحتياطية إلى هوية الوكيل النشط.

---

## Gateway

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
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // allowedOrigins: ["https://control.example.com"], // required for non-loopback Control UI
      // dangerouslyAllowHostHeaderOriginFallback: false, // dangerous Host-header origin fallback mode
      // allowInsecureAuth: false,
      // dangerouslyDisableDeviceAuth: false,
    },
    remote: {
      url: "ws://gateway.tailnet:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // Optional. Default false.
    allowRealIpFallback: false,
    tools: {
      // Additional /tools/invoke HTTP denies
      deny: ["browser"],
      // Remove tools from the default HTTP deny list
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="تفاصيل حقول Gateway">

- `mode`: `local` (تشغيل gateway) أو `remote` (الاتصال بـ gateway بعيد). ويرفض Gateway البدء ما لم يكن `local`.
- `port`: منفذ متعدد الإرسال واحد لـ WS + HTTP. الأسبقية: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: `auto` أو `loopback` (افتراضي) أو `lan` (`0.0.0.0`) أو `tailnet` (عنوان Tailscale IP فقط) أو `custom`.
- **الأسماء البديلة القديمة لـ bind**: استخدم قيم وضع bind في `gateway.bind` (`auto` أو `loopback` أو `lan` أو `tailnet` أو `custom`)، وليس الأسماء البديلة للمضيف (`0.0.0.0` أو `127.0.0.1` أو `localhost` أو `::` أو `::1`).
- **ملاحظة Docker**: يعمل ربط `loopback` الافتراضي على الاستماع على `127.0.0.1` داخل الحاوية. ومع استخدام Docker bridge networking ‏(`-p 18789:18789`) تصل الحركة على `eth0`، لذا يتعذر الوصول إلى gateway. استخدم `--network host`، أو اضبط `bind: "lan"` (أو `bind: "custom"` مع `customBindHost: "0.0.0.0"`) للاستماع على جميع الواجهات.
- **المصادقة**: مطلوبة افتراضيًا. تتطلب روابط non-loopback مصادقة gateway. وعمليًا يعني ذلك رمزًا/كلمة مرور مشتركة أو reverse proxy واعيًا بالهوية مع `gateway.auth.mode: "trusted-proxy"`. ويولّد معالج الإعداد رمزًا افتراضيًا.
- إذا كان كل من `gateway.auth.token` و`gateway.auth.password` مضبوطين (بما في ذلك SecretRefs)، فاضبط `gateway.auth.mode` صراحةً على `token` أو `password`. وتفشل تدفقات بدء التشغيل وتثبيت/إصلاح الخدمة عندما يكون الاثنان مضبوطين ولا يكون الوضع محددًا.
- `gateway.auth.mode: "none"`: وضع صريح بلا مصادقة. استخدمه فقط مع إعدادات local loopback موثوقة؛ ولا يُعرض هذا عمدًا في مطالبات الإعداد.
- `gateway.auth.mode: "trusted-proxy"`: فوّض المصادقة إلى reverse proxy واعٍ بالهوية ووثق بترويسات الهوية من `gateway.trustedProxies` (راجع [Trusted Proxy Auth](/ar/gateway/trusted-proxy-auth)). يتوقع هذا الوضع **مصدر proxy غير loopback**؛ ولا تفي reverse proxies الموجودة على loopback وفي المضيف نفسه بمتطلبات trusted-proxy auth.
- `gateway.auth.allowTailscale`: عند ضبطه على `true`، يمكن لترويسات هوية Tailscale Serve أن تستوفي مصادقة Control UI/WebSocket (بعد التحقق عبر `tailscale whois`). أما نقاط نهاية HTTP API فلا تستخدم مصادقة ترويسات Tailscale هذه؛ بل تتبع وضع مصادقة HTTP العادي للبوابة. ويفترض هذا التدفق من دون رمز مميز أن مضيف gateway موثوق. وتكون القيمة الافتراضية `true` عندما يكون `tailscale.mode = "serve"`.
- `gateway.auth.rateLimit`: محدِّد اختياري لمحاولات المصادقة الفاشلة. ويُطبّق لكل عنوان IP عميل ولكل نطاق مصادقة (يتم تتبّع السر المشترك ورمز الجهاز كلٌ على حدة). وتعطي المحاولات المحظورة `429` + `Retry-After`.
  - في مسار Control UI غير المتزامن الخاص بـ Tailscale Serve، تتم تسلسل المحاولات الفاشلة لـ `{scope, clientIp}` نفسه قبل كتابة الفشل. ولذلك قد يؤدي تزامن المحاولات السيئة من العميل نفسه إلى تشغيل المحدِّد في الطلب الثاني بدلًا من مرور الطلبين كمجرّد حالات عدم تطابق.
  - تكون القيمة الافتراضية لـ `gateway.auth.rateLimit.exemptLoopback` هي `true`؛ اضبطها على `false` عندما تريد عمدًا تطبيق تحديد المعدل أيضًا على حركة localhost (في إعدادات الاختبار أو deployments الصارمة عبر proxy).
- يتم دائمًا خنق محاولات مصادقة WS ذات الأصل المتصفح مع تعطيل استثناء loopback (دفاعًا إضافيًا ضد محاولات القوة الغاشمة المحلية من المتصفح).
- على loopback، تكون حالات القفل الناتجة عن هذه الأصول المعتمدة على المتصفح معزولة لكل قيمة `Origin`
  مُطبّعة، بحيث لا تؤدي الإخفاقات المتكررة من أصل localhost واحد تلقائيًا
  إلى حظر أصل مختلف.
- `tailscale.mode`: ‏`serve` ‏(tailnet فقط، مع bind على loopback) أو `funnel` ‏(عام، ويتطلب مصادقة).
- `controlUi.allowedOrigins`: قائمة سماح صريحة لأصول المتصفح لاتصالات Gateway WebSocket. مطلوبة عندما يُتوقع وجود عملاء متصفح من أصول non-loopback.
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: وضع خطير يفعّل الاحتياط إلى أصل ترويسة Host للـ deployments التي تعتمد عمدًا على سياسة أصل ترويسة Host.
- `remote.transport`: ‏`ssh` (افتراضي) أو `direct` ‏(ws/wss). بالنسبة إلى `direct`، يجب أن تكون `remote.url` بصيغة `ws://` أو `wss://`.
- `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`: تجاوز طارئ على جهة العميل يسمح باستخدام `ws://` النصي إلى عناوين IP موثوقة على الشبكات الخاصة؛ ويظل الافتراضي مقتصرًا على loopback فقط بالنسبة إلى الاتصال النصي.
- `gateway.remote.token` / `.password` هما حقلا بيانات اعتماد للعميل البعيد. ولا يضبطان مصادقة gateway بحد ذاتهما.
- `gateway.push.apns.relay.baseUrl`: عنوان HTTPS الأساسي للـ APNs relay الخارجي الذي تستخدمه نسخ iOS الرسمية/TestFlight بعد أن تنشر تسجيلات مدعومة بالـ relay إلى gateway. يجب أن يطابق هذا العنوان عنوان relay المضمّن في نسخة iOS.
- `gateway.push.apns.relay.timeoutMs`: مهلة الإرسال من gateway إلى relay بالمللي ثانية. الافتراضي: `10000`.
- يتم تفويض التسجيلات المدعومة بالـ relay إلى هوية gateway محددة. ويجلب تطبيق iOS المقترن القيمة `gateway.identity.get`، ويضمّن تلك الهوية في تسجيل relay، ويمرر grant للإرسال ضمن نطاق التسجيل إلى gateway. ولا يمكن لـ gateway آخر إعادة استخدام ذلك التسجيل المخزّن.
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: تجاوزات env مؤقتة لإعدادات relay المذكورة أعلاه.
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: منفذ هروب خاص بالتطوير فقط لعناوين relay على loopback باستخدام HTTP. وينبغي أن تبقى عناوين relay الإنتاجية على HTTPS.
- `gateway.channelHealthCheckMinutes`: فاصل مراقبة صحة القنوات بالدقائق. اضبطه على `0` لتعطيل إعادة التشغيل الخاصة بمراقب الصحة عالميًا. الافتراضي: `5`.
- `gateway.channelStaleEventThresholdMinutes`: عتبة socket القديم بالدقائق. اجعلها أكبر من أو مساوية لـ `gateway.channelHealthCheckMinutes`. الافتراضي: `30`.
- `gateway.channelMaxRestartsPerHour`: الحد الأقصى لإعادات التشغيل لكل قناة/حساب خلال ساعة متحركة. الافتراضي: `10`.
- `channels.<provider>.healthMonitor.enabled`: تعطيل اختياري لكل قناة لإعادات التشغيل بواسطة مراقب الصحة مع إبقاء المراقب العام مفعّلًا.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: تجاوز لكل حساب في القنوات متعددة الحسابات. وعند ضبطه، تكون له الأسبقية على التجاوز على مستوى القناة.
- يمكن لمسارات استدعاء gateway المحلية استخدام `gateway.remote.*` كقيمة احتياطية فقط عندما لا تكون `gateway.auth.*` مضبوطة.
- إذا كان `gateway.auth.token` / `gateway.auth.password` مضبوطًا صراحةً عبر SecretRef ولم يمكن حله، يفشل الحل مع الإغلاق الآمن (من دون إخفاء الاحتياط البعيد).
- `trustedProxies`: عناوين IP الخاصة بالـ reverse proxies التي تنهي TLS أو تحقن ترويسات العميل المُمرَّر. أدرج فقط الـ proxies التي تتحكم بها. وتظل إدخالات loopback صالحة لإعدادات proxy على المضيف نفسه/اكتشاف محلي (مثل Tailscale Serve أو reverse proxy محلي)، لكنها **لا** تجعل طلبات loopback مؤهلة لـ `gateway.auth.mode: "trusted-proxy"`.
- `allowRealIpFallback`: عند ضبطه على `true`، تقبل gateway الترويسة `X-Real-IP` إذا كانت `X-Forwarded-For` مفقودة. الافتراضي `false` لسلوك إغلاق آمن.
- `gateway.tools.deny`: أسماء أدوات إضافية محظورة لطلب HTTP `POST /tools/invoke` (توسّع قائمة المنع الافتراضية).
- `gateway.tools.allow`: إزالة أسماء أدوات من قائمة المنع الافتراضية الخاصة بـ HTTP.

</Accordion>

### نقاط النهاية المتوافقة مع OpenAI

- Chat Completions: معطّلة افتراضيًا. فعّلها عبر `gateway.http.endpoints.chatCompletions.enabled: true`.
- Responses API: ‏`gateway.http.endpoints.responses.enabled`.
- تقوية إدخالات URL في Responses:
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    تُعامل قوائم السماح الفارغة على أنها غير مضبوطة؛ استخدم `gateway.http.endpoints.responses.files.allowUrl=false`
    و/أو `gateway.http.endpoints.responses.images.allowUrl=false` لتعطيل جلب URL.
- ترويسة اختيارية لتقوية الاستجابة:
  - `gateway.http.securityHeaders.strictTransportSecurity` (اضبطها فقط لأصول HTTPS التي تتحكم بها؛ راجع [Trusted Proxy Auth](/ar/gateway/trusted-proxy-auth#tls-termination-and-hsts))

### عزل المثيلات المتعددة

شغّل عدة Gateways على مضيف واحد مع منافذ وأدلة حالة فريدة:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

أعلام مريحة: `--dev` (يستخدم `~/.openclaw-dev` + المنفذ `19001`)، و`--profile <name>` (يستخدم `~/.openclaw-<name>`).

راجع [Gateways متعددة](/ar/gateway/multiple-gateways).

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`: يفعّل إنهاء TLS عند مستمع gateway ‏(HTTPS/WSS) (الافتراضي: `false`).
- `autoGenerate`: ينشئ تلقائيًا زوج شهادة/مفتاح محلي موقّع ذاتيًا عندما لا تكون الملفات الصريحة مضبوطة؛ للاستخدام المحلي/التطويري فقط.
- `certPath`: مسار نظام الملفات إلى ملف شهادة TLS.
- `keyPath`: مسار نظام الملفات إلى ملف المفتاح الخاص لـ TLS؛ ويجب إبقاؤه مقيّد الأذونات.
- `caPath`: مسار اختياري لحزمة CA من أجل التحقق من العميل أو سلاسل الثقة المخصصة.

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`: يتحكم في كيفية تطبيق تعديلات الإعدادات في وقت التشغيل.
  - `"off"`: تجاهل التعديلات المباشرة؛ وتتطلب التغييرات إعادة تشغيل صريحة.
  - `"restart"`: إعادة تشغيل عملية gateway دائمًا عند تغيير الإعدادات.
  - `"hot"`: تطبيق التغييرات داخل العملية من دون إعادة تشغيل.
  - `"hybrid"` (افتراضي): جرّب hot reload أولًا؛ ثم ارجع إلى إعادة التشغيل عند الحاجة.
- `debounceMs`: نافذة debounce بالمللي ثانية قبل تطبيق تغييرات الإعدادات (عدد صحيح غير سالب).
- `deferralTimeoutMs`: الحد الأقصى للوقت بالمللي ثانية لانتظار العمليات الجارية قبل فرض إعادة التشغيل (الافتراضي: `300000` = 5 دقائق).

---

## Hooks

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    maxBodyBytes: 262144,
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: false,
    allowedSessionKeyPrefixes: ["hook:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "From: {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.4-mini",
      },
    ],
  },
}
```

المصادقة: `Authorization: Bearer <token>` أو `x-openclaw-token: <token>`.
تُرفض رموز hook المميزة في سلسلة الاستعلام.

ملاحظات التحقق والأمان:

- يتطلب `hooks.enabled=true` وجود `hooks.token` غير فارغ.
- يجب أن يكون `hooks.token` **مختلفًا** عن `gateway.auth.token`؛ ويُرفض إعادة استخدام رمز Gateway.
- لا يمكن أن يكون `hooks.path` مساويًا لـ `/`؛ استخدم مسارًا فرعيًا مخصصًا مثل `/hooks`.
- إذا كان `hooks.allowRequestSessionKey=true`، فقيّد `hooks.allowedSessionKeyPrefixes` (مثل `["hook:"]`).

**نقاط النهاية:**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - لا يُقبل `sessionKey` من حمولة الطلب إلا عندما يكون `hooks.allowRequestSessionKey=true` (الافتراضي: `false`).
- `POST /hooks/<name>` → يُحل عبر `hooks.mappings`

<Accordion title="تفاصيل التعيين">

- يطابق `match.path` المسار الفرعي بعد `/hooks` (على سبيل المثال `/hooks/gmail` → `gmail`).
- يطابق `match.source` حقلًا في الحمولة للمسارات العامة.
- تقرأ القوالب مثل `{{messages[0].subject}}` من الحمولة.
- يمكن أن يشير `transform` إلى وحدة JS/TS تُرجع إجراء hook.
  - يجب أن يكون `transform.module` مسارًا نسبيًا وأن يبقى ضمن `hooks.transformsDir` (تُرفض المسارات المطلقة وعمليات العبور).
- يوجّه `agentId` إلى وكيل محدد؛ وتعود المعرّفات غير المعروفة إلى الوكيل الافتراضي.
- `allowedAgentIds`: يقيّد التوجيه الصريح (`*` أو الحذف = السماح للجميع، و`[]` = منع الجميع).
- `defaultSessionKey`: مفتاح جلسة ثابت اختياري لتشغيلات وكيل hook عندما لا يكون هناك `sessionKey` صريح.
- `allowRequestSessionKey`: يسمح لمستدعي `/hooks/agent` بتعيين `sessionKey` (الافتراضي: `false`).
- `allowedSessionKeyPrefixes`: قائمة سماح اختيارية للبادئات الخاصة بقيم `sessionKey` الصريحة (في الطلب + التعيين)، مثل `["hook:"]`.
- يؤدي `deliver: true` إلى إرسال الرد النهائي إلى قناة؛ وتكون القيمة الافتراضية لـ `channel` هي `last`.
- يتجاوز `model` قيمة LLM لهذا التشغيل الخاص بالـ hook (ويجب أن يكون مسموحًا به إذا كان كتالوج النماذج مضبوطًا).

</Accordion>

### تكامل Gmail

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openrouter/meta-llama/llama-3.3-70b-instruct:free",
      thinking: "off",
    },
  },
}
```

- يبدأ Gateway تلقائيًا تشغيل `gog gmail watch serve` عند الإقلاع عندما يكون مضبوطًا. اضبط `OPENCLAW_SKIP_GMAIL_WATCHER=1` للتعطيل.
- لا تشغّل `gog gmail watch serve` منفصلًا إلى جانب Gateway.

---

## مضيف Canvas

```json5
{
  canvasHost: {
    root: "~/.openclaw/workspace/canvas",
    liveReload: true,
    // enabled: false, // or OPENCLAW_SKIP_CANVAS_HOST=1
  },
}
```

- يقدّم HTML/CSS/JS وA2UI القابلة للتعديل من الوكيل عبر HTTP ضمن منفذ Gateway:
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- محلي فقط: أبقِ `gateway.bind: "loopback"` (افتراضي).
- في روابط non-loopback: تتطلب مسارات canvas مصادقة Gateway (token/password/trusted-proxy)، مثل بقية أسطح HTTP الخاصة بـ Gateway.
- لا ترسل Node WebViews عادةً ترويسات المصادقة؛ وبعد اقتران العقدة واتصالها، يعلن Gateway عناوين URL للإمكانات ضمن نطاق العقدة للوصول إلى canvas/A2UI.
- ترتبط عناوين URL الخاصة بالإمكانات بجلسة WS النشطة للعقدة وتنتهي صلاحيتها بسرعة. ولا يُستخدم احتياط قائم على IP.
- يحقن عميل live-reload في HTML المقدَّم.
- ينشئ تلقائيًا ملف `index.html` ابتدائيًا عندما يكون الدليل فارغًا.
- يقدّم أيضًا A2UI على `/__openclaw__/a2ui/`.
- تتطلب التغييرات إعادة تشغيل gateway.
- عطّل live reload للأدلة الكبيرة أو عند أخطاء `EMFILE`.

---

## الاكتشاف

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal` (افتراضي): يحذف `cliPath` + `sshPort` من سجلات TXT.
- `full`: يتضمن `cliPath` + `sshPort`.
- يكون اسم المضيف افتراضيًا `openclaw`. وتجاوزه يتم عبر `OPENCLAW_MDNS_HOSTNAME`.

### النطاق الواسع (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

يكتب منطقة unicast DNS-SD ضمن `~/.openclaw/dns/`. ومن أجل الاكتشاف عبر الشبكات، قرنه بخادم DNS (يوصى بـ CoreDNS) + Tailscale split DNS.

الإعداد: `openclaw dns setup --apply`.

---

## البيئة

### `env` (متغيرات env مضمنة)

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- لا تُطبّق متغيرات env المضمنة إلا إذا كانت بيئة العملية تفتقد المفتاح.
- ملفات `.env`: ملف `.env` في دليل العمل الحالي + `~/.openclaw/.env` (ولا يجاوز أي منهما المتغيرات الموجودة).
- `shellEnv`: يستورد المفاتيح المفقودة والمتوقعة من ملف تعريف صدفة تسجيل الدخول لديك.
- راجع [البيئة](/ar/help/environment) للحصول على الأسبقية الكاملة.

### استبدال متغيرات env

أشر إلى متغيرات env في أي سلسلة إعدادات باستخدام `${VAR_NAME}`:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- لا تُطابق إلا الأسماء ذات الأحرف الكبيرة: `[A-Z_][A-Z0-9_]*`.
- تؤدي المتغيرات المفقودة/الفارغة إلى خطأ عند تحميل الإعدادات.
- استخدم `$${VAR}` للهروب إلى `${VAR}` الحرفية.
- يعمل مع `$include`.

---

## الأسرار

مراجع الأسرار إضافية: ما تزال القيم النصية الصريحة تعمل.

### `SecretRef`

استخدم شكل كائن واحد:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

التحقق:

- نمط `provider`: ‏`^[a-z][a-z0-9_-]{0,63}$`
- نمط `id` عند `source: "env"`: ‏`^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"` ‏`id`: مؤشر JSON مطلق (مثل `"/providers/openai/apiKey"`)
- نمط `id` عند `source: "exec"`: ‏`^[A-Za-z0-9][A-Za-z0-9._:/-]{0,255}$`
- يجب ألا تحتوي معرّفات `source: "exec"` على مقاطع مسار `.` أو `..` مفصولة بشرطة مائلة (على سبيل المثال `a/../b` مرفوض)

### سطح بيانات الاعتماد المدعوم

- المصفوفة القياسية: [سطح بيانات اعتماد SecretRef](/ar/reference/secretref-credential-surface)
- تستهدف `secrets apply` مسارات بيانات الاعتماد المدعومة في `openclaw.json`.
- تُضمَّن مراجع `auth-profiles.json` في الحل أثناء وقت التشغيل وفي تغطية التدقيق.

### إعدادات موفّري الأسرار

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // optional explicit env provider
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

ملاحظات:

- يدعم الموفّر `file` الوضعين `mode: "json"` و`mode: "singleValue"` (ويجب أن تكون `id` مساوية لـ `"value"` في وضع singleValue).
- يتطلب الموفّر `exec` مسار `command` مطلقًا ويستخدم حمولات البروتوكول على stdin/stdout.
- تُرفض مسارات الأوامر الرمزية افتراضيًا. اضبط `allowSymlinkCommand: true` للسماح بمسارات الروابط الرمزية مع التحقق من مسار الهدف الذي تم حله.
- إذا كان `trustedDirs` مضبوطًا، فسيُطبَّق فحص الأدلة الموثوقة على مسار الهدف الذي تم حله.
- تكون بيئة العملية الفرعية لـ `exec` دنيا افتراضيًا؛ ومرّر المتغيرات المطلوبة صراحةً باستخدام `passEnv`.
- تُحل مراجع الأسرار وقت التفعيل إلى لقطة موجودة في الذاكرة، ثم لا تقرأ مسارات الطلبات إلا من هذه اللقطة.
- يُطبّق ترشيح السطح النشط أثناء التفعيل: تؤدي المراجع غير المحلولة على الأسطح المفعّلة إلى فشل بدء التشغيل/إعادة التحميل، بينما تُتجاوز الأسطح غير النشطة مع تشخيصات.

---

## تخزين المصادقة

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai-codex:personal": { provider: "openai-codex", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      "openai-codex": ["openai-codex:personal"],
    },
  },
}
```

- تُخزَّن الملفات الشخصية لكل وكيل في `<agentDir>/auth-profiles.json`.
- يدعم `auth-profiles.json` مراجع على مستوى القيمة (`keyRef` لـ `api_key`، و`tokenRef` لـ `token`) لأوضاع بيانات الاعتماد الثابتة.
- لا تدعم الملفات الشخصية بوضع OAuth ‏(`auth.profiles.<id>.mode = "oauth"`) بيانات اعتماد ملفات المصادقة المدعومة بـ SecretRef.
- تأتي بيانات اعتماد runtime الثابتة من لقطات محلولة داخل الذاكرة؛ ويتم تنظيف إدخالات `auth.json` الثابتة القديمة عند اكتشافها.
- تتم واردات OAuth القديمة من `~/.openclaw/credentials/oauth.json`.
- راجع [OAuth](/ar/concepts/oauth).
- سلوك runtime الخاص بالأسرار وأدوات `audit/configure/apply`: ‏[إدارة الأسرار](/ar/gateway/secrets).

### `auth.cooldowns`

```json5
{
  auth: {
    cooldowns: {
      billingBackoffHours: 5,
      billingBackoffHoursByProvider: { anthropic: 3, openai: 8 },
      billingMaxHours: 24,
      authPermanentBackoffMinutes: 10,
      authPermanentMaxMinutes: 60,
      failureWindowHours: 24,
      overloadedProfileRotations: 1,
      overloadedBackoffMs: 0,
      rateLimitedProfileRotations: 1,
    },
  },
}
```

- `billingBackoffHours`: فترة التراجع الأساسية بالساعات عندما يفشل ملف شخصي بسبب
  أخطاء فوترة/رصيد غير كافٍ حقيقية (الافتراضي: `5`). وما يزال من الممكن أن
  يقع نص الفوترة الصريح هنا حتى في استجابات `401`/`403`، لكن
  تظل مطابقات النص الخاصة بالموفّر محصورة في الموفّر الذي يملكها (مثل
  OpenRouter ‏`Key limit exceeded`). أما رسائل حد الإنفاق الخاصة بنافذة الاستخدام
  أو المنظمة/مساحة العمل والقابلة لإعادة المحاولة في HTTP `402` فتبقى ضمن مسار
  `rate_limit` بدلًا من ذلك.
- `billingBackoffHoursByProvider`: تجاوزات اختيارية لكل موفّر لفترة تراجع الفوترة بالساعات.
- `billingMaxHours`: الحد الأعلى بالساعات للنمو الأسي لفترة تراجع الفوترة (الافتراضي: `24`).
- `authPermanentBackoffMinutes`: فترة التراجع الأساسية بالدقائق لإخفاقات `auth_permanent` عالية الثقة (الافتراضي: `10`).
- `authPermanentMaxMinutes`: الحد الأعلى بالدقائق لنمو فترة تراجع `auth_permanent` (الافتراضي: `60`).
- `failureWindowHours`: النافذة المتحركة بالساعات المستخدمة لعدادات التراجع (الافتراضي: `24`).
- `overloadedProfileRotations`: الحد الأقصى لعمليات تدوير ملف المصادقة للموفّر نفسه بسبب أخطاء التحميل الزائد قبل الانتقال إلى النموذج الاحتياطي (الافتراضي: `1`). وتقع أشكال انشغال الموفّر مثل `ModelNotReadyException` هنا.
- `overloadedBackoffMs`: تأخير ثابت قبل إعادة المحاولة مع تدوير ملف/موفّر محمّل زائدًا (الافتراضي: `0`).
- `rateLimitedProfileRotations`: الحد الأقصى لعمليات تدوير ملف المصادقة للموفّر نفسه بسبب أخطاء تحديد المعدل قبل الانتقال إلى النموذج الاحتياطي (الافتراضي: `1`). وتتضمن فئة تحديد المعدل تلك نصوصًا بصياغة الموفّر مثل `Too many concurrent requests` و`ThrottlingException` و`concurrency limit reached` و`workers_ai ... quota limit exceeded` و`resource exhausted`.

---

## التسجيل

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- ملف السجل الافتراضي: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`.
- اضبط `logging.file` لمسار ثابت.
- ترتفع قيمة `consoleLevel` إلى `debug` عند استخدام `--verbose`.
- `maxFileBytes`: الحد الأقصى لحجم ملف السجل بالبايت قبل إيقاف الكتابات (عدد صحيح موجب؛ الافتراضي: `524288000` = 500 MB). استخدم تدوير سجلات خارجيًا في deployments الإنتاجية.

---

## التشخيصات

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],
    stuckSessionWarnMs: 30000,

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      sampleRate: 1.0,
      flushIntervalMs: 5000,
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`: المفتاح الرئيسي لمخرجات instrumentation ‏(الافتراضي: `true`).
- `flags`: مصفوفة من سلاسل الأعلام التي تفعّل مخرجات سجل موجّهة (وتدعم أحرف البدل مثل `"telegram.*"` أو `"*"`).
- `stuckSessionWarnMs`: عتبة العمر بالمللي ثانية لإصدار تحذيرات الجلسات العالقة بينما تظل الجلسة في حالة المعالجة.
- `otel.enabled`: يفعّل مسار التصدير الخاص بـ OpenTelemetry ‏(الافتراضي: `false`).
- `otel.endpoint`: عنوان URL الخاص بالمجمّع لتصدير OTel.
- `otel.protocol`: ‏`"http/protobuf"` (افتراضي) أو `"grpc"`.
- `otel.headers`: ترويسات بيانات وصفية إضافية لـ HTTP/gRPC تُرسل مع طلبات تصدير OTel.
- `otel.serviceName`: اسم الخدمة لسمات المورد.
- `otel.traces` / `otel.metrics` / `otel.logs`: تفعيل تصدير التتبعات أو المقاييس أو السجلات.
- `otel.sampleRate`: معدل أخذ عينات التتبعات من `0` إلى `1`.
- `otel.flushIntervalMs`: فترة التفريغ الدوري للقياسات بالمللي ثانية.
- `cacheTrace.enabled`: تسجيل لقطات تتبع cache للتشغيلات المضمنة (الافتراضي: `false`).
- `cacheTrace.filePath`: مسار الإخراج لـ JSONL الخاص بتتبع cache (الافتراضي: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`).
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`: تتحكم في ما يتم تضمينه في مخرجات تتبع cache (وجميعها افتراضيًا: `true`).

---

## التحديث

```json5
{
  update: {
    channel: "stable", // stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
      stableDelayHours: 6,
      stableJitterHours: 12,
      betaCheckIntervalHours: 1,
    },
  },
}
```

- `channel`: قناة الإصدار لعمليات تثبيت npm/git — ‏`"stable"` أو `"beta"` أو `"dev"`.
- `checkOnStart`: التحقق من تحديثات npm عند بدء gateway ‏(الافتراضي: `true`).
- `auto.enabled`: تفعيل التحديث التلقائي في الخلفية لعمليات تثبيت الحزم (الافتراضي: `false`).
- `auto.stableDelayHours`: الحد الأدنى للتأخير بالساعات قبل التطبيق التلقائي لقناة stable ‏(الافتراضي: `6`؛ الحد الأقصى: `168`).
- `auto.stableJitterHours`: نافذة توزيع إضافية بالساعات لطرح قناة stable ‏(الافتراضي: `12`؛ الحد الأقصى: `168`).
- `auto.betaCheckIntervalHours`: عدد الساعات بين مرات التحقق في قناة beta ‏(الافتراضي: `1`؛ الحد الأقصى: `24`).

---

## ACP

```json5
{
  acp: {
    enabled: false,
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    maxConcurrentSessions: 10,

    stream: {
      coalesceIdleMs: 50,
      maxChunkChars: 1000,
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
      hiddenBoundarySeparator: "paragraph", // none | space | newline | paragraph
      maxOutputChars: 50000,
      maxSessionUpdateChars: 500,
    },

    runtime: {
      ttlMinutes: 30,
    },
  },
}
```

- `enabled`: بوابة الميزات العامة لـ ACP ‏(الافتراضي: `false`).
- `dispatch.enabled`: بوابة مستقلة لإرسال أدوار جلسات ACP ‏(الافتراضي: `true`). اضبطها على `false` للإبقاء على أوامر ACP متاحة مع حظر التنفيذ.
- `backend`: معرّف backend الافتراضي لـ runtime في ACP (ويجب أن يطابق إضافة runtime ACP مسجّلة).
- `defaultAgent`: معرّف وكيل ACP الاحتياطي المستهدف عندما لا تحدد عمليات الإنشاء هدفًا صريحًا.
- `allowedAgents`: قائمة سماح لمعرّفات الوكلاء المسموح بها لجلسات runtime في ACP؛ وتعني القائمة الفارغة عدم وجود تقييد إضافي.
- `maxConcurrentSessions`: الحد الأقصى لعدد جلسات ACP النشطة بالتوازي.
- `stream.coalesceIdleMs`: نافذة التفريغ عند الخمول بالمللي ثانية للنص المتدفق.
- `stream.maxChunkChars`: الحد الأقصى لحجم الكتلة قبل تقسيم إسقاط الكتلة المتدفقة.
- `stream.repeatSuppression`: حجب سطور الحالة/الأدوات المتكررة في كل دور (الافتراضي: `true`).
- `stream.deliveryMode`: تعني `"live"` البث التدريجي؛ بينما تعني `"final_only"` التخزين المؤقت حتى أحداث نهاية الدور.
- `stream.hiddenBoundarySeparator`: الفاصل قبل النص المرئي بعد أحداث الأدوات المخفية (الافتراضي: `"paragraph"`).
- `stream.maxOutputChars`: الحد الأقصى لأحرف خرج المساعد التي تُسقَط لكل دور ACP.
- `stream.maxSessionUpdateChars`: الحد الأقصى للأحرف الخاصة بسطور حالة/تحديث ACP المسقطة.
- `stream.tagVisibility`: سجل يربط أسماء الوسوم بتجاوزات رؤية منطقية لأحداث البث.
- `runtime.ttlMinutes`: مدة TTL للخمول بالدقائق لعمال جلسات ACP قبل أن يصبحوا مؤهلين للتنظيف.
- `runtime.installCommand`: أمر تثبيت اختياري يُشغَّل عند تمهيد بيئة runtime الخاصة بـ ACP.

---

## CLI

```json5
{
  cli: {
    banner: {
      taglineMode: "off", // random | default | off
    },
  },
}
```

- يتحكم `cli.banner.taglineMode` في نمط الشعار النصي:
  - `"random"` (افتراضي): شعارات نصية دوّارة مضحكة/موسمية.
  - `"default"`: شعار نصي محايد ثابت (`All your chats, one OpenClaw.`).
  - `"off"`: بدون نص شعار (مع الاستمرار في عرض عنوان الشريط/الإصدار).
- لإخفاء الشريط بالكامل (وليس الشعارات النصية فقط)، اضبط متغير البيئة `OPENCLAW_HIDE_BANNER=1`.

---

## Wizard

بيانات وصفية تكتبها تدفقات الإعداد الموجّهة في CLI ‏(`onboard` و`configure` و`doctor`):

```json5
{
  wizard: {
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
  },
}
```

---

## الهوية

راجع حقول الهوية في `agents.list` ضمن [القيم الافتراضية للوكلاء](#agent-defaults).

---

## Bridge (قديم، تمت إزالته)

لم تعد الإصدارات الحالية تتضمن TCP bridge. وتتصل العقد عبر Gateway WebSocket. ولم تعد مفاتيح `bridge.*` جزءًا من مخطط الإعدادات (ويفشل التحقق إلى أن تتم إزالتها؛ ويمكن لـ `openclaw doctor --fix` إزالة المفاتيح غير المعروفة).

<Accordion title="إعدادات bridge القديمة (مرجع تاريخي)">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    maxConcurrentRuns: 2,
    webhook: "https://example.invalid/legacy", // deprecated fallback for stored notify:true jobs
    webhookToken: "replace-with-dedicated-token", // optional bearer token for outbound webhook auth
    sessionRetention: "24h", // duration string or false
    runLog: {
      maxBytes: "2mb", // default 2_000_000 bytes
      keepLines: 2000, // default 2000
    },
  },
}
```

- `sessionRetention`: مدة الاحتفاظ بجلسات تشغيل cron المعزولة المكتملة قبل تقليمها من `sessions.json`. ويتحكم أيضًا في تنظيف نُسخ السجلات المؤرشفة الخاصة بعمليات cron المحذوفة. الافتراضي: `24h`؛ اضبطه على `false` للتعطيل.
- `runLog.maxBytes`: الحد الأقصى للحجم لكل ملف سجل تشغيل (`cron/runs/<jobId>.jsonl`) قبل التقليم. الافتراضي: `2_000_000` بايت.
- `runLog.keepLines`: أحدث السطور التي يُحتفظ بها عند تشغيل تقليم سجل التشغيل. الافتراضي: `2000`.
- `webhookToken`: رمز bearer يُستخدم في تسليم POST الخاص بـ webhook في cron ‏(`delivery.mode = "webhook"`)، وإذا حُذف فلن تُرسل ترويسة مصادقة.
- `webhook`: عنوان URL احتياطي قديم ومهجور لـ webhook ‏(http/https) ويُستخدم فقط للمهام المخزنة التي ما زال لديها `notify: true`.

### `cron.retry`

```json5
{
  cron: {
    retry: {
      maxAttempts: 3,
      backoffMs: [30000, 60000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "timeout", "server_error"],
    },
  },
}
```

- `maxAttempts`: الحد الأقصى لإعادات المحاولة للمهام ذات التشغيل الواحد عند الأخطاء العابرة (الافتراضي: `3`؛ المجال: `0`–`10`).
- `backoffMs`: مصفوفة تأخيرات backoff بالمللي ثانية لكل محاولة إعادة (الافتراضي: `[30000, 60000, 300000]`؛ من 1 إلى 10 إدخالات).
- `retryOn`: أنواع الأخطاء التي تُشغّل إعادة المحاولة — ‏`"rate_limit"` و`"overloaded"` و`"network"` و`"timeout"` و`"server_error"`. احذفها لإعادة المحاولة عند جميع الأنواع العابرة.

يُطبّق ذلك على مهام cron ذات التشغيل الواحد فقط. أما المهام المتكررة فلها معالجة فشل منفصلة.

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`: تفعيل تنبيهات الفشل لمهام cron ‏(الافتراضي: `false`).
- `after`: عدد الإخفاقات المتتالية قبل إطلاق التنبيه (عدد صحيح موجب، الحد الأدنى: `1`).
- `cooldownMs`: الحد الأدنى بالمللي ثانية بين التنبيهات المتكررة للوظيفة نفسها (عدد صحيح غير سالب).
- `mode`: وضع التسليم — يرسل `"announce"` عبر رسالة قناة؛ بينما يرسل `"webhook"` إلى webhook المضبوط.
- `accountId`: معرّف حساب أو قناة اختياري لتحديد نطاق تسليم التنبيه.

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- وجهة افتراضية لإشعارات فشل cron عبر جميع الوظائف.
- `mode`: ‏`"announce"` أو `"webhook"`؛ وتكون القيمة الافتراضية `"announce"` عندما تتوفر بيانات هدف كافية.
- `channel`: تجاوز القناة لتسليم announce. تعيد `"last"` استخدام آخر قناة تسليم معروفة.
- `to`: هدف announce صريح أو عنوان URL لـ webhook. وهو مطلوب في وضع webhook.
- `accountId`: تجاوز اختياري للحساب عند التسليم.
- تتجاوز `delivery.failureDestination` لكل وظيفة هذا الافتراضي العام.
- عندما لا تكون هناك وجهة فشل عامة ولا لكل وظيفة، تعود الوظائف التي تسلّم أصلًا عبر `announce` إلى ذلك الهدف الأساسي لـ announce عند الفشل.
- لا تكون `delivery.failureDestination` مدعومة إلا في الوظائف ذات `sessionTarget="isolated"` ما لم يكن `delivery.mode` الأساسي للوظيفة هو `"webhook"`.

راجع [وظائف Cron](/ar/automation/cron-jobs). ويتم تتبع تنفيذات cron المعزولة بوصفها [مهام في الخلفية](/ar/automation/tasks).

---

## متغيرات قوالب نماذج الوسائط

العناصر النائبة للقوالب التي يتم توسيعها في `tools.media.models[].args`:

| المتغير           | الوصف                                       |
| ----------------- | ------------------------------------------- |
| `{{Body}}`        | نص الرسالة الواردة الكامل                   |
| `{{RawBody}}`     | النص الخام (من دون أغلفة السجل/المرسل)      |
| `{{BodyStripped}}` | النص بعد إزالة إشارات المجموعات منه        |
| `{{From}}`        | معرّف المرسل                                |
| `{{To}}`          | معرّف الوجهة                                |
| `{{MessageSid}}`  | معرّف رسالة القناة                          |
| `{{SessionId}}`   | UUID الخاص بالجلسة الحالية                  |
| `{{IsNewSession}}` | `"true"` عند إنشاء جلسة جديدة              |
| `{{MediaUrl}}`    | عنوان pseudo-URL للوسائط الواردة            |
| `{{MediaPath}}`   | المسار المحلي للوسائط                       |
| `{{MediaType}}`   | نوع الوسائط (image/audio/document/…)        |
| `{{Transcript}}`  | النص المفرغ للصوت                           |
| `{{Prompt}}`      | موجّه الوسائط المحلول لإدخالات CLI          |
| `{{MaxChars}}`    | الحد الأقصى المحلول لأحرف الإخراج لإدخالات CLI |
| `{{ChatType}}`    | `"direct"` أو `"group"`                     |
| `{{GroupSubject}}` | موضوع المجموعة (بأفضل جهد)                 |
| `{{GroupMembers}}` | معاينة أعضاء المجموعة (بأفضل جهد)          |
| `{{SenderName}}`  | اسم العرض للمرسل (بأفضل جهد)               |
| `{{SenderE164}}`  | رقم هاتف المرسل (بأفضل جهد)                |
| `{{Provider}}`    | تلميح الموفّر (whatsapp أو telegram أو discord، إلخ.) |

---

## تضمينات الإعدادات (`$include`)

قسّم الإعدادات إلى عدة ملفات:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**سلوك الدمج:**

- ملف واحد: يستبدل الكائن الحاوي.
- مصفوفة من الملفات: تُدمج دمجًا عميقًا بالترتيب (وتتجاوز الملفات اللاحقة السابقة).
- المفاتيح الشقيقة: تُدمج بعد التضمينات (وتتجاوز القيم المضمّنة).
- التضمينات المتداخلة: حتى 10 مستويات عمق.
- المسارات: تُحل نسبةً إلى الملف المُضمِّن، ولكن يجب أن تبقى داخل دليل الإعدادات ذي المستوى الأعلى (`dirname` الخاص بـ `openclaw.json`). ويُسمح بالأشكال المطلقة/`../` فقط عندما تُحل في النهاية داخل هذا الحد.
- الأخطاء: رسائل واضحة للملفات المفقودة، وأخطاء التحليل، والتضمينات الدائرية.

---

_مرتبط: [الإعدادات](/ar/gateway/configuration) · [أمثلة الإعدادات](/ar/gateway/configuration-examples) · [Doctor](/ar/gateway/doctor)_
