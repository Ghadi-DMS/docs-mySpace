---
read_when:
    - تحتاج إلى دلالات إعداد دقيقة على مستوى الحقول أو إلى القيم الافتراضية
    - أنت تتحقق من كتل إعداد القناة أو النموذج أو gateway أو الأداة
summary: المرجع الكامل لكل مفتاح إعداد في OpenClaw، والقيم الافتراضية، وإعدادات القنوات
title: مرجع التهيئة
x-i18n:
    generated_at: "2026-04-05T12:48:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: bb4c6de7955aa0c6afa2d20f12a0e3782b16ab2c1b6bf3ed0a8910be2f0a47d1
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# مرجع التهيئة

كل حقل متاح في `~/.openclaw/openclaw.json`. للحصول على نظرة عامة موجهة بالمهام، راجع [التهيئة](/gateway/configuration).

تنسيق الإعداد هو **JSON5** (تُسمح التعليقات والفواصل الختامية). جميع الحقول اختيارية — يستخدم OpenClaw قيمًا افتراضية آمنة عند حذفها.

---

## القنوات

تبدأ كل قناة تلقائيًا عندما يكون قسم إعدادها موجودًا (ما لم يكن `enabled: false`).

### الوصول إلى الرسائل المباشرة والمجموعات

تدعم جميع القنوات سياسات الرسائل المباشرة وسياسات المجموعات:

| سياسة الرسائل المباشرة | السلوك |
| ------------------- | --------------------------------------------------------------- |
| `pairing` (الافتراضي) | يحصل المرسلون غير المعروفين على رمز اقتران لمرة واحدة؛ ويجب أن يوافق المالك |
| `allowlist` | فقط المرسلون الموجودون في `allowFrom` (أو مخزن السماح المقترن) |
| `open` | السماح بكل الرسائل المباشرة الواردة (يتطلب `allowFrom: ["*"]`) |
| `disabled` | تجاهل كل الرسائل المباشرة الواردة |

| سياسة المجموعة | السلوك |
| --------------------- | ------------------------------------------------------ |
| `allowlist` (الافتراضي) | فقط المجموعات المطابقة لقائمة السماح المهيأة |
| `open` | تجاوز قوائم السماح الخاصة بالمجموعات (مع بقاء بوابة الإشارة مطبقة) |
| `disabled` | حظر كل رسائل المجموعة/الغرفة |

<Note>
تضبط `channels.defaults.groupPolicy` القيمة الافتراضية عندما تكون `groupPolicy` الخاصة بأحد الموفّرين غير مضبوطة.
تنتهي صلاحية رموز الاقتران بعد ساعة واحدة. ويُحدَّد عدد طلبات اقتران الرسائل المباشرة المعلقة بـ **3 لكل قناة**.
إذا كانت كتلة الموفّر مفقودة بالكامل (`channels.<provider>` غير موجودة)، فإن سياسة المجموعة في وقت التشغيل تعود احتياطيًا إلى `allowlist` (إغلاق عند الفشل) مع تحذير عند بدء التشغيل.
</Note>

### تجاوزات نموذج القناة

استخدم `channels.modelByChannel` لتثبيت معرّفات قنوات محددة على نموذج ما. تقبل القيم `provider/model` أو الأسماء المستعارة للنماذج المهيأة. ويُطبَّق تعيين القناة عندما لا تكون لدى الجلسة بالفعل قيمة تجاوز للنموذج (على سبيل المثال، مضبوطة عبر `/model`).

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

### القيم الافتراضية للقنوات وheartbeat

استخدم `channels.defaults` لسلوك سياسة المجموعات وheartbeat المشترك بين الموفّرين:

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

- `channels.defaults.groupPolicy`: سياسة المجموعة الاحتياطية عندما تكون `groupPolicy` على مستوى الموفّر غير مضبوطة.
- `channels.defaults.contextVisibility`: وضع ظهور السياق الإضافي الافتراضي لجميع القنوات. القيم: `all` (الافتراضي، تضمين كل سياق الاقتباس/السلسلة/السجل)، و`allowlist` (تضمين السياق فقط من المرسلين المسموح لهم)، و`allowlist_quote` (مثل allowlist لكن مع الاحتفاظ بسياق الاقتباس/الرد الصريح). تجاوز لكل قناة: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: تضمين حالات القنوات السليمة في مخرجات heartbeat.
- `channels.defaults.heartbeat.showAlerts`: تضمين الحالات المتدهورة/الخطأ في مخرجات heartbeat.
- `channels.defaults.heartbeat.useIndicator`: عرض مخرجات heartbeat مختصرة على نمط المؤشر.

### WhatsApp

يعمل WhatsApp عبر قناة الويب في gateway ‏(Baileys Web). ويبدأ تلقائيًا عندما توجد جلسة مرتبطة.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // العلامات الزرقاء (false في وضع self-chat)
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

- تستخدم الأوامر الصادرة افتراضيًا الحساب `default` إذا كان موجودًا؛ وإلا أول معرّف حساب مهيأ (بعد الترتيب).
- تتجاوز `channels.whatsapp.defaultAccount` الاختيار الاحتياطي الافتراضي لذلك الحساب عندما تطابق معرّف حساب مهيأ.
- يُرحَّل دليل مصادقة Baileys القديم للحساب الواحد بواسطة `openclaw doctor` إلى `whatsapp/default`.
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
      replyToMode: "first", // off | first | all
      linkPreview: true,
      streaming: "partial", // off | partial | block | progress (الافتراضي: off؛ فعّل صراحة لتجنب حدود معدل تعديل المعاينات)
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

- رمز البوت: `channels.telegram.botToken` أو `channels.telegram.tokenFile` (ملف عادي فقط؛ تُرفض الروابط الرمزية)، مع `TELEGRAM_BOT_TOKEN` كاحتياطي للحساب الافتراضي.
- تتجاوز `channels.telegram.defaultAccount` اختيار الحساب الافتراضي عندما تطابق معرّف حساب مهيأ.
- في الإعدادات متعددة الحسابات (معرّفا حساب أو أكثر)، عيّن قيمة افتراضية صريحة (`channels.telegram.defaultAccount` أو `channels.telegram.accounts.default`) لتجنب التوجيه الاحتياطي؛ ويحذر `openclaw doctor` عند غيابها أو عدم صلاحيتها.
- يؤدي `configWrites: false` إلى حظر كتابات الإعداد التي يبدأها Telegram ‏(عمليات ترحيل معرّفات supergroup، و`/config set|unset`).
- تضبط إدخالات `bindings[]` ذات المستوى الأعلى التي تحتوي على `type: "acp"` روابط ACP دائمة لموضوعات المنتدى (استخدم الصيغة القانونية `chatId:topic:topicId` في `match.peer.id`). وتُشارك دلالات الحقول في [ACP Agents](/tools/acp-agents#channel-specific-settings).
- تستخدم معاينات بث Telegram الأمرين `sendMessage` + `editMessageText` ‏(وتعمل في الدردشات المباشرة والجماعية).
- سياسة إعادة المحاولة: راجع [سياسة إعادة المحاولة](/concepts/retry).

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 8,
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
      replyToMode: "off", // off | first | all
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
      streaming: "off", // off | partial | block | progress (يتم تعيين progress إلى partial على Discord)
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
        spawnSubagentSessions: false, // opt-in لـ sessions_spawn({ thread: true })
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
- تستخدم الاستدعاءات الصادرة المباشرة التي توفر `token` صريحًا لـ Discord ذلك الرمز في الاستدعاء؛ أما إعدادات إعادة المحاولة/السياسة الخاصة بالحساب فتظل مأخوذة من الحساب المحدد في لقطة وقت التشغيل النشطة.
- تتجاوز `channels.discord.defaultAccount` اختيار الحساب الافتراضي عندما تطابق معرّف حساب مهيأ.
- استخدم `user:<id>` ‏(رسالة مباشرة) أو `channel:<id>` ‏(قناة guild) لأهداف التسليم؛ وتُرفض المعرّفات الرقمية المجردة.
- تكون slugs الخاصة بـ guilds بأحرف صغيرة مع استبدال المسافات بـ `-`؛ وتستخدم مفاتيح القنوات الاسم المطبّع (من دون `#`). ويفضّل استخدام معرّفات guild.
- يتم تجاهل الرسائل التي أنشأها bot افتراضيًا. يؤدي `allowBots: true` إلى تمكينها؛ واستخدم `allowBots: "mentions"` لقبول رسائل bot التي تذكر bot فقط (مع استمرار تصفية الرسائل الذاتية).
- تؤدي `channels.discord.guilds.<id>.ignoreOtherMentions` ‏(وتجاوزات القنوات) إلى إسقاط الرسائل التي تذكر مستخدمًا أو دورًا آخر لكن لا تذكر bot ‏(باستثناء @everyone/@here).
- يقوم `maxLinesPerMessage` ‏(الافتراضي 17) بتقسيم الرسائل الطويلة عموديًا حتى عندما تكون أقل من 2000 حرف.
- يتحكم `channels.discord.threadBindings` في التوجيه المرتبط بسلاسل Discord:
  - `enabled`: تجاوز Discord لميزات الجلسات المرتبطة بالسلاسل (`/focus` و`/unfocus` و`/agents` و`/session idle` و`/session max-age` والتسليم/التوجيه المرتبط)
  - `idleHours`: تجاوز Discord لإلغاء التركيز التلقائي بسبب عدم النشاط بالساعات (`0` للتعطيل)
  - `maxAgeHours`: تجاوز Discord للعمر الأقصى الصارم بالساعات (`0` للتعطيل)
  - `spawnSubagentSessions`: مفتاح opt-in للإنشاء/الربط التلقائي للسلاسل عبر `sessions_spawn({ thread: true })`
- تضبط إدخالات `bindings[]` ذات المستوى الأعلى التي تحتوي على `type: "acp"` روابط ACP دائمة للقنوات والسلاسل (استخدم معرّف القناة/السلسلة في `match.peer.id`). وتُشارك دلالات الحقول في [ACP Agents](/tools/acp-agents#channel-specific-settings).
- تضبط `channels.discord.ui.components.accentColor` لون التمييز لحاويات Discord components v2.
- تمكّن `channels.discord.voice` محادثات قنوات Discord الصوتية وتجاوزات auto-join + TTS الاختيارية.
- تمرر `channels.discord.voice.daveEncryption` و`channels.discord.voice.decryptionFailureTolerance` مباشرة إلى خيارات DAVE في `@discordjs/voice` ‏(`true` و`24` افتراضيًا).
- يحاول OpenClaw أيضًا استعادة استقبال الصوت عبر مغادرة/إعادة الانضمام إلى جلسة صوتية بعد تكرار إخفاقات فك التشفير.
- تمثل `channels.discord.streaming` مفتاح وضع البث القانوني. ويتم ترحيل `streamMode` القديم وقيم `streaming` المنطقية تلقائيًا.
- تعيّن `channels.discord.autoPresence` التوافر في وقت التشغيل إلى حالة حضور bot ‏(سليم => online، متدهور => idle، منهك => dnd) وتسمح بتجاوزات نص الحالة الاختيارية.
- تعيد `channels.discord.dangerouslyAllowNameMatching` تمكين المطابقة حسب الاسم/الوسم القابل للتغيير (وضع توافق طارئ).
- `channels.discord.execApprovals`: تسليم موافقات exec الأصلية في Discord وتفويض الموافقين.
  - `enabled`: ‏`true` أو `false` أو `"auto"` ‏(الافتراضي). في وضع auto، تُفعَّل موافقات exec عندما يمكن حل الموافقين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Discord المسموح لهم بالموافقة على طلبات exec. وتعود احتياطيًا إلى `commands.ownerAllowFrom` عند الحذف.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتمرير الموافقات لكل الوكلاء.
  - `sessionFilter`: أنماط مفاتيح جلسات اختيارية (substring أو regex).
  - `target`: مكان إرسال مطالبات الموافقة. يرسل `"dm"` ‏(الافتراضي) إلى الرسائل المباشرة للموافقين، ويرسل `"channel"` إلى القناة الأصلية، ويرسل `"both"` إلى كليهما. وعندما يتضمن الهدف `"channel"`، لا تكون الأزرار قابلة للاستخدام إلا من قبل الموافقين المحلولين.
  - `cleanupAfterResolve`: عندما تكون `true`، تحذف رسائل الموافقة المباشرة بعد الموافقة أو الرفض أو انتهاء المهلة.

**أوضاع إشعارات التفاعل:** ‏`off` ‏(لا شيء)، و`own` ‏(رسائل bot، الافتراضي)، و`all` ‏(كل الرسائل)، و`allowlist` ‏(من `guilds.<id>.users` على كل الرسائل).

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

- JSON لحساب الخدمة: مضمن (`serviceAccount`) أو قائم على ملف (`serviceAccountFile`).
- يُدعم أيضًا Service account SecretRef ‏(`serviceAccountRef`).
- القيم الاحتياطية من env: ‏`GOOGLE_CHAT_SERVICE_ACCOUNT` أو `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- استخدم `spaces/<spaceId>` أو `users/<userId>` لأهداف التسليم.
- تعيد `channels.googlechat.dangerouslyAllowNameMatching` تمكين مطابقة البريد الإلكتروني الرئيسي القابل للتغيير (وضع توافق طارئ).

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
      replyToMode: "off", // off | first | all
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
      nativeStreaming: true, // استخدام Slack native streaming API عندما تكون streaming=partial
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

- **Socket mode** يتطلب كلاً من `botToken` و`appToken` ‏(`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` كاحتياطي env للحساب الافتراضي).
- **HTTP mode** يتطلب `botToken` بالإضافة إلى `signingSecret` ‏(على الجذر أو لكل حساب).
- تقبل `botToken` و`appToken` و`signingSecret` و`userToken` سلاسل نصية صريحة
  أو كائنات SecretRef.
- تعرض لقطات حساب Slack حقول مصدر/حالة لكل بيانات الاعتماد مثل
  `botTokenSource` و`botTokenStatus` و`appTokenStatus`، وفي وضع HTTP،
  `signingSecretStatus`. وتعني `configured_unavailable` أن الحساب
  مهيأ عبر SecretRef لكن مسار الأمر/وقت التشغيل الحالي لم يتمكن من
  حل قيمة السر.
- يؤدي `configWrites: false` إلى حظر كتابات الإعداد التي يبدأها Slack.
- تتجاوز `channels.slack.defaultAccount` اختيار الحساب الافتراضي عندما تطابق معرّف حساب مهيأ.
- تمثل `channels.slack.streaming` مفتاح وضع البث القانوني. ويتم ترحيل `streamMode` القديم وقيم `streaming` المنطقية تلقائيًا.
- استخدم `user:<id>` ‏(رسالة مباشرة) أو `channel:<id>` لأهداف التسليم.

**أوضاع إشعارات التفاعل:** ‏`off` و`own` ‏(الافتراضي) و`all` و`allowlist` ‏(من `reactionAllowlist`).

**عزل جلسات السلاسل:** يكون `thread.historyScope` لكل سلسلة ‏(الافتراضي) أو مشتركًا عبر القناة. وتنسخ `thread.inheritParent` نص القناة الأصلية إلى السلاسل الجديدة.

- تضيف `typingReaction` تفاعلًا مؤقتًا إلى رسالة Slack الواردة أثناء تشغيل الرد، ثم تزيله عند الاكتمال. استخدم shortcode لـ Slack emoji مثل `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: تسليم موافقات exec الأصلية في Slack وتفويض الموافقين. المخطط نفسه الخاص بـ Discord: ‏`enabled` ‏(`true`/`false`/`"auto"`)، و`approvers` ‏(معرّفات مستخدمي Slack)، و`agentFilter`، و`sessionFilter`، و`target` ‏(`"dm"` أو `"channel"` أو `"both"`).

| مجموعة الإجراء | الافتراضي | ملاحظات |
| ------------ | ------- | ---------------------- |
| reactions | مفعّل | التفاعل + سرد التفاعلات |
| messages | مفعّل | القراءة/الإرسال/التعديل/الحذف |
| pins | مفعّل | التثبيت/إزالة التثبيت/السرد |
| memberInfo | مفعّل | معلومات العضو |
| emojiList | مفعّل | قائمة emoji المخصصة |

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
        native: true, // opt-in
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // عنوان URL صريح اختياري للنشرات ذات reverse-proxy/public
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

أوضاع الدردشة: ‏`oncall` ‏(الرد عند @-mention، الافتراضي)، و`onmessage` ‏(كل رسالة)، و`onchar` ‏(الرسائل التي تبدأ ببادئة المشغّل).

عندما تكون أوامر Mattermost الأصلية مفعّلة:

- يجب أن يكون `commands.callbackPath` مسارًا (مثل `/api/channels/mattermost/command`) وليس عنوان URL كاملًا.
- يجب أن يُحل `commands.callbackUrl` إلى نقطة نهاية OpenClaw gateway وأن يكون قابلاً للوصول من خادم Mattermost.
- تتم مصادقة native slash callbacks باستخدام الرموز المميزة لكل أمر التي يعيدها
  Mattermost أثناء تسجيل slash command. وإذا فشل التسجيل أو لم يتم تنشيط
  أي أوامر، يرفض OpenClaw callbacks بالرسالة
  `Unauthorized: invalid command token.`
- بالنسبة إلى مضيفي callback الخاصين/ضمن tailnet/الداخليين، قد يتطلب Mattermost
  أن تتضمن `ServiceSettings.AllowedUntrustedInternalConnections` المضيف/النطاق الخاص بالـ callback.
  استخدم قيم المضيف/النطاق، وليس عناوين URL كاملة.
- `channels.mattermost.configWrites`: السماح أو المنع لكتابات الإعداد التي يبدأها Mattermost.
- `channels.mattermost.requireMention`: اشتراط `@mention` قبل الرد في القنوات.
- `channels.mattermost.groups.<channelId>.requireMention`: تجاوز بوابة الإشارة لكل قناة (`"*"` للافتراضي).
- تتجاوز `channels.mattermost.defaultAccount` اختيار الحساب الافتراضي عندما تطابق معرّف حساب مهيأ.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // ربط حساب اختياري
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

**أوضاع إشعارات التفاعل:** ‏`off` و`own` ‏(الافتراضي) و`all` و`allowlist` ‏(من `reactionAllowlist`).

- `channels.signal.account`: تثبيت بدء تشغيل القناة على هوية حساب Signal محددة.
- `channels.signal.configWrites`: السماح أو المنع لكتابات الإعداد التي يبدأها Signal.
- تتجاوز `channels.signal.defaultAccount` اختيار الحساب الافتراضي عندما تطابق معرّف حساب مهيأ.

### BlueBubbles

تُعد BlueBubbles المسار الموصى به لـ iMessage ‏(مدعومة بإضافة، ومهيأة تحت `channels.bluebubbles`).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl وpassword وwebhookPath وعناصر تحكم المجموعات والإجراءات المتقدمة:
      // راجع /channels/bluebubbles
    },
  },
}
```

- تشمل مسارات المفاتيح الأساسية هنا: `channels.bluebubbles` و`channels.bluebubbles.dmPolicy`.
- تتجاوز `channels.bluebubbles.defaultAccount` اختيار الحساب الافتراضي عندما تطابق معرّف حساب مهيأ.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات BlueBubbles بجلسات ACP دائمة. استخدم مقبض BlueBubbles أو سلسلة هدف (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [ACP Agents](/tools/acp-agents#channel-specific-settings).
- تُوثق تهيئة قناة BlueBubbles الكاملة في [BlueBubbles](/channels/bluebubbles).

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

- تتجاوز `channels.imessage.defaultAccount` اختيار الحساب الافتراضي عندما تطابق معرّف حساب مهيأ.

- يتطلب Full Disk Access إلى قاعدة بيانات Messages.
- يُفضّل استخدام أهداف `chat_id:<id>`. استخدم `imsg chats --limit 20` لسرد الدردشات.
- يمكن لـ `cliPath` أن يشير إلى SSH wrapper؛ واضبط `remoteHost` ‏(`host` أو `user@host`) من أجل جلب المرفقات عبر SCP.
- تقيّد `attachmentRoots` و`remoteAttachmentRoots` مسارات المرفقات الواردة (الافتراضي: `/Users/*/Library/Messages/Attachments`).
- يستخدم SCP تحققًا صارمًا من مفتاح المضيف، لذا تأكد من أن مفتاح مضيف relay موجود بالفعل في `~/.ssh/known_hosts`.
- `channels.imessage.configWrites`: السماح أو المنع لكتابات الإعداد التي يبدأها iMessage.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات iMessage بجلسات ACP دائمة. استخدم مقبضًا مطبّعًا أو هدف دردشة صريحًا (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [ACP Agents](/tools/acp-agents#channel-specific-settings).

<Accordion title="مثال SSH wrapper لـ iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix مدعومة بإضافة ومهيأة تحت `channels.matrix`.

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

- تستخدم المصادقة بالرمز المميز `accessToken`؛ وتستخدم المصادقة بكلمة المرور `userId` + `password`.
- يوجّه `channels.matrix.proxy` حركة Matrix HTTP عبر HTTP(S) proxy صريح. ويمكن للحسابات المسماة تجاوزه عبر `channels.matrix.accounts.<id>.proxy`.
- تسمح `channels.matrix.allowPrivateNetwork` بالخوادم المنزلية الخاصة/الداخلية. ويُعد `proxy` و`allowPrivateNetwork` عنصرين مستقلين.
- تختار `channels.matrix.defaultAccount` الحساب المفضل في الإعدادات متعددة الحسابات.
- `channels.matrix.execApprovals`: تسليم موافقات exec الأصلية في Matrix وتفويض الموافقين.
  - `enabled`: ‏`true` أو `false` أو `"auto"` ‏(الافتراضي). في وضع auto، تُفعَّل موافقات exec عندما يمكن حل الموافقين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Matrix ‏(مثل `@owner:example.org`) المسموح لهم بالموافقة على طلبات exec.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتمرير الموافقات لكل الوكلاء.
  - `sessionFilter`: أنماط مفاتيح جلسات اختيارية (substring أو regex).
  - `target`: مكان إرسال مطالبات الموافقة. `"dm"` ‏(الافتراضي)، أو `"channel"` ‏(الغرفة الأصلية)، أو `"both"`.
  - تجاوزات لكل حساب: `channels.matrix.accounts.<id>.execApprovals`.
- تستخدم فحوصات حالة Matrix وعمليات البحث الحية في الدليل سياسة proxy نفسها الخاصة بحركة وقت التشغيل.
- تُوثق تهيئة Matrix الكاملة وقواعد الاستهداف وأمثلة الإعداد في [Matrix](/channels/matrix).

### Microsoft Teams

Microsoft Teams مدعومة بإضافة ومهيأة تحت `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId وappPassword وtenantId وwebhook وسياسات الفريق/القناة:
      // راجع /channels/msteams
    },
  },
}
```

- تشمل مسارات المفاتيح الأساسية هنا: `channels.msteams` و`channels.msteams.configWrites`.
- تُوثق تهيئة Teams الكاملة (بيانات الاعتماد، وwebhook، وسياسة الرسائل المباشرة/المجموعة، والتجاوزات لكل فريق/قناة) في [Microsoft Teams](/channels/msteams).

### IRC

IRC مدعومة بإضافة ومهيأة تحت `channels.irc`.

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

- تشمل مسارات المفاتيح الأساسية هنا: `channels.irc` و`channels.irc.dmPolicy` و`channels.irc.configWrites` و`channels.irc.nickserv.*`.
- تتجاوز `channels.irc.defaultAccount` اختيار الحساب الافتراضي عندما تطابق معرّف حساب مهيأ.
- تُوثق تهيئة قناة IRC الكاملة (المضيف/المنفذ/TLS/القنوات/قوائم السماح/بوابة الإشارة) في [IRC](/channels/irc).

### تعدد الحسابات (جميع القنوات)

شغّل عدة حسابات لكل قناة (كل منها له `accountId` خاص به):

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

- يُستخدم `default` عندما يتم حذف `accountId` ‏(CLI + التوجيه).
- لا تنطبق رموز env إلا على الحساب **الافتراضي**.
- تُطبَّق إعدادات القناة الأساسية على جميع الحسابات ما لم يتم تجاوزها لكل حساب.
- استخدم `bindings[].match.accountId` لتوجيه كل حساب إلى وكيل مختلف.
- إذا أضفت حسابًا غير افتراضي عبر `openclaw channels add` ‏(أو onboarding الخاص بالقناة) بينما ما زلت تستخدم إعداد قناة ذي مستوى أعلى لحساب واحد، فإن OpenClaw يقوم أولًا بترقية القيم العليا الخاصة بالحساب الواحد ذات النطاق الحسابي إلى خريطة حسابات القناة حتى يستمر الحساب الأصلي في العمل. تنقلها معظم القنوات إلى `channels.<channel>.accounts.default`؛ ويمكن لـ Matrix الاحتفاظ بهدف مسمى/افتراضي مطابق موجود بدلًا من ذلك.
- تظل الروابط الموجودة الخاصة بالقناة فقط (من دون `accountId`) تطابق الحساب الافتراضي؛ وتبقى الروابط ضمن نطاق الحساب اختيارية.
- يصلح `openclaw doctor --fix` أيضًا الأشكال المختلطة عبر نقل القيم العليا الخاصة بالحساب الواحد ذات النطاق الحسابي إلى الحساب المُرقّى المختار لتلك القناة. تستخدم معظم القنوات `accounts.default`؛ ويمكن لـ Matrix الحفاظ على هدف مسمى/افتراضي مطابق موجود بدلًا من ذلك.

### قنوات الإضافات الأخرى

تُهيأ العديد من قنوات الإضافات على شكل `channels.<id>` وتُوثق في صفحات القنوات المخصصة لها (على سبيل المثال Feishu وMatrix وLINE وNostr وZalo وNextcloud Talk وSynology Chat وTwitch).
راجع فهرس القنوات الكامل: [القنوات](/channels).

### بوابة الإشارة في الدردشة الجماعية

تفترض رسائل المجموعات افتراضيًا **ضرورة الإشارة** (إشارة في البيانات الوصفية أو أنماط regex آمنة). وينطبق ذلك على دردشات المجموعات في WhatsApp وTelegram وDiscord وGoogle Chat وiMessage.

**أنواع الإشارات:**

- **إشارات البيانات الوصفية**: إشارات @ الأصلية للمنصة. ويتم تجاهلها في وضع self-chat في WhatsApp.
- **أنماط النص**: أنماط regex آمنة في `agents.list[].groupChat.mentionPatterns`. ويتم تجاهل الأنماط غير الصالحة والتكرار المتداخل غير الآمن.
- تُفرض بوابة الإشارة فقط عندما يكون الكشف ممكنًا (إشارات أصلية أو نمط واحد على الأقل).

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

يضبط `messages.groupChat.historyLimit` القيمة الافتراضية العامة. ويمكن للقنوات التجاوز عبر `channels.<channel>.historyLimit` ‏(أو لكل حساب). اضبطه على `0` للتعطيل.

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

الحل: تجاوز لكل رسالة مباشرة ← افتراضي الموفّر ← لا حد (الاحتفاظ بكل شيء).

المدعوم: `telegram` و`whatsapp` و`discord` و`slack` و`signal` و`imessage` و`msteams`.

#### وضع self-chat

ضمّن رقمك الخاص في `allowFrom` لتمكين وضع self-chat ‏(يتجاهل إشارات @ الأصلية، ويرد فقط على الأنماط النصية):

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
    native: "auto", // تسجيل الأوامر الأصلية عند الدعم
    text: true, // تحليل /commands في رسائل الدردشة
    bash: false, // السماح بـ ! (اسم مستعار: /bash)
    bashForegroundMs: 2000,
    config: false, // السماح بـ /config
    debug: false, // السماح بـ /debug
    restart: false, // السماح بـ /restart + أداة gateway restart
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
- يقوم `native: "auto"` بتشغيل الأوامر الأصلية لـ Discord/Telegram ويُبقي Slack معطلًا.
- تجاوز لكل قناة: `channels.discord.commands.native` ‏(قيمة منطقية أو `"auto"`). تؤدي `false` إلى مسح الأوامر المسجلة سابقًا.
- تضيف `channels.telegram.customCommands` إدخالات إضافية إلى قائمة بوت Telegram.
- يؤدي `bash: true` إلى تمكين `! <cmd>` لـ shell الخاص بالمضيف. ويتطلب `tools.elevated.enabled` وأن يكون المرسل ضمن `tools.elevated.allowFrom.<channel>`.
- يؤدي `config: true` إلى تمكين `/config` ‏(قراءة/كتابة `openclaw.json`). بالنسبة إلى عملاء gateway ‏`chat.send`، تتطلب كتابات `/config set|unset` الدائمة أيضًا `operator.admin`؛ بينما يظل `/config show` للقراءة فقط متاحًا لعملاء المشغّل العاديين ذوي صلاحية الكتابة.
- تتحكم `channels.<provider>.configWrites` في طفرات الإعداد لكل قناة (الافتراضي: true).
- بالنسبة إلى القنوات متعددة الحسابات، تتحكم `channels.<provider>.accounts.<id>.configWrites` أيضًا في الكتابات التي تستهدف ذلك الحساب (على سبيل المثال `/allowlist --config --account <id>` أو `/config set channels.<provider>.accounts.<id>...`).
- تكون `allowFrom` لكل موفّر. وعند ضبطها، تصبح **مصدر التفويض الوحيد** (وتُتجاهل قوائم السماح/الاقتران الخاصة بالقناة و`useAccessGroups`).
- يسمح `useAccessGroups: false` للأوامر بتجاوز سياسات مجموعات الوصول عندما لا تكون `allowFrom` مضبوطة.

</Accordion>

---

## القيم الافتراضية للوكيل

### `agents.defaults.workspace`

الافتراضي: `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

جذر مستودع اختياري يظهر في سطر Runtime داخل system prompt. وإذا لم يُضبط، يكتشفه OpenClaw تلقائيًا بالصعود من workspace.

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
      { id: "docs", skills: ["docs-search"] }, // يستبدل القيم الافتراضية
      { id: "locked-down", skills: [] }, // بلا Skills
    ],
  },
}
```

- احذف `agents.defaults.skills` للحصول على Skills غير مقيّدة افتراضيًا.
- احذف `agents.list[].skills` لوراثة القيم الافتراضية.
- اضبط `agents.list[].skills: []` لعدم وجود Skills.
- تمثل قائمة `agents.list[].skills` غير الفارغة المجموعة النهائية لذلك الوكيل؛
  وهي لا تُدمج مع القيم الافتراضية.

### `agents.defaults.skipBootstrap`

يعطل الإنشاء التلقائي لملفات bootstrap في workspace ‏(`AGENTS.md` و`SOUL.md` و`TOOLS.md` و`IDENTITY.md` و`USER.md` و`HEARTBEAT.md` و`BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.bootstrapMaxChars`

الحد الأقصى للأحرف لكل ملف bootstrap في workspace قبل الاقتطاع. الافتراضي: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

الحد الأقصى الإجمالي للأحرف المحقونة عبر جميع ملفات bootstrap في workspace. الافتراضي: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

يتحكم في نص التحذير المرئي للوكيل عندما يُقتطع سياق bootstrap.
الافتراضي: `"once"`.

- `"off"`: لا يحقن نص تحذير في system prompt مطلقًا.
- `"once"`: يحقن التحذير مرة واحدة لكل توقيع اقتطاع فريد (موصى به).
- `"always"`: يحقن التحذير في كل تشغيل عند وجود اقتطاع.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

الحد الأقصى لحجم البكسلات لأطول ضلع في الصورة في كتل الصور الخاصة بالنص/الأداة قبل استدعاءات الموفّر.
الافتراضي: `1200`.

غالبًا ما تقلل القيم الأقل من استخدام vision-token وحجم حمولة الطلب في التشغيلات الثقيلة على لقطات الشاشة.
أما القيم الأعلى فتحافظ على تفاصيل بصرية أكثر.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

المنطقة الزمنية لسياق system prompt ‏(وليس لطوابع الرسائل الزمنية). وتعود احتياطيًا إلى المنطقة الزمنية للمضيف.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

تنسيق الوقت في system prompt. الافتراضي: `auto` ‏(تفضيل نظام التشغيل).

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
      params: { cacheRetention: "long" }, // معلمات موفّر افتراضية عامة
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

- `model`: تقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يضبط شكل السلسلة النموذج الأساسي فقط.
  - يضبط شكل الكائن النموذج الأساسي بالإضافة إلى نماذج التراجع الاحتياطي المرتبة.
- `imageModel`: تقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - تُستخدم في مسار أداة `image` بوصفها إعداد نموذج الرؤية.
  - وتُستخدم أيضًا كتوجيه تراجع احتياطي عندما لا يستطيع النموذج المحدد/الافتراضي قبول إدخال الصور.
- `imageGenerationModel`: تقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - تُستخدم بواسطة قدرة إنشاء الصور المشتركة وأي سطح أداة/إضافة مستقبلي ينشئ صورًا.
  - القيم النموذجية: `google/gemini-3.1-flash-image-preview` لإنشاء الصور الأصلي في Gemini، أو `fal/fal-ai/flux/dev` لـ fal، أو `openai/gpt-image-1` لـ OpenAI Images.
  - إذا اخترت موفّرًا/نموذجًا مباشرة، فقم بتهيئة المصادقة/مفتاح API المطابق للموفّر أيضًا (على سبيل المثال `GEMINI_API_KEY` أو `GOOGLE_API_KEY` لـ `google/*`، و`OPENAI_API_KEY` لـ `openai/*`، و`FAL_KEY` لـ `fal/*`).
  - إذا حُذفت، فلا يزال بإمكان `image_generate` استنتاج افتراضي موفّر مدعوم بالمصادقة. فهو يحاول أولًا موفّر الجلسة الافتراضي الحالي، ثم باقي موفّري إنشاء الصور المسجلين بترتيب معرّفات الموفّرين.
- `videoGenerationModel`: تقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - تُستخدم بواسطة قدرة إنشاء الفيديو المشتركة.
  - القيم النموذجية: `qwen/wan2.6-t2v` و`qwen/wan2.6-i2v` و`qwen/wan2.6-r2v` و`qwen/wan2.6-r2v-flash` أو `qwen/wan2.7-r2v`.
  - اضبط هذا صراحة قبل استخدام إنشاء الفيديو المشترك. وعلى خلاف `imageGenerationModel`، لا تستنتج بيئة تشغيل إنشاء الفيديو افتراضيًا لموفّر حتى الآن.
  - إذا اخترت موفّرًا/نموذجًا مباشرة، فقم بتهيئة المصادقة/مفتاح API المطابق للموفّر أيضًا.
  - يدعم موفّر Qwen المدمج لإنشاء الفيديو حاليًا حتى 1 فيديو خرج، و1 صورة دخل، و4 فيديوهات دخل، و10 ثوانٍ مدة، وخيارات على مستوى الموفّر مثل `size` و`aspectRatio` و`resolution` و`audio` و`watermark`.
- `pdfModel`: تقبل إما سلسلة (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - تُستخدم بواسطة أداة `pdf` لتوجيه النموذج.
  - إذا حُذفت، تعود أداة PDF احتياطيًا إلى `imageModel`، ثم إلى نموذج الجلسة/الافتراضي المحلول.
- `pdfMaxBytesMb`: حد حجم PDF الافتراضي لأداة `pdf` عندما لا يتم تمرير `maxBytesMb` وقت الاستدعاء.
- `pdfMaxPages`: الحد الأقصى الافتراضي للصفحات التي تؤخذ في الاعتبار بواسطة وضع الاستخراج الاحتياطي في أداة `pdf`.
- `verboseDefault`: مستوى verbose الافتراضي للوكلاء. القيم: `"off"` و`"on"` و`"full"`. الافتراضي: `"off"`.
- `elevatedDefault`: مستوى المخرجات elevated الافتراضي للوكلاء. القيم: `"off"` و`"on"` و`"ask"` و`"full"`. الافتراضي: `"on"`.
- `model.primary`: التنسيق `provider/model` ‏(مثل `openai/gpt-5.4`). إذا حذفت الموفّر، يحاول OpenClaw أولًا الاسم المستعار، ثم مطابقة موفّر مهيأ فريد لذلك المعرّف الدقيق للنموذج، وبعد ذلك فقط يعود إلى الموفّر الافتراضي المهيأ (سلوك توافق قديم مهمل، لذا يُفضّل استخدام `provider/model` الصريح). وإذا لم يعد ذلك الموفّر يعرض النموذج الافتراضي المهيأ، فإن OpenClaw يعود احتياطيًا إلى أول موفّر/نموذج مهيأ بدلًا من إظهار افتراضي قديم لموفّر تمت إزالته.
- `models`: فهرس النماذج المهيأ وقائمة السماح الخاصة بـ `/model`. ويمكن أن يتضمن كل إدخال `alias` ‏(اختصار) و`params` ‏(خاصة بالموفّر، مثل `temperature` و`maxTokens` و`cacheRetention` و`context1m`).
- `params`: معلمات موفّر افتراضية عامة تُطبَّق على جميع النماذج. تُضبط عند `agents.defaults.params` ‏(مثل `{ cacheRetention: "long" }`).
- أسبقية دمج `params` ‏(في الإعداد): يُتجاوز `agents.defaults.params` ‏(القاعدة العامة) بواسطة `agents.defaults.models["provider/model"].params` ‏(لكل نموذج)، ثم تتجاوز `agents.list[].params` ‏(المطابقة لمعرّف الوكيل) حسب المفتاح. راجع [Prompt Caching](/reference/prompt-caching) للتفاصيل.
- تحفظ أدوات كتابة الإعداد التي تطفّر هذه الحقول (مثل `/models set` و`/models set-image` وأوامر إضافة/إزالة التراجعات الاحتياطية) شكل الكائن القانوني وتحافظ على قوائم التراجع الاحتياطي الموجودة عند الإمكان.
- `maxConcurrent`: الحد الأقصى للتشغيلات المتوازية للوكلاء عبر الجلسات (مع بقاء كل جلسة متسلسلة). الافتراضي: 4.

**اختصارات alias المدمجة** (تُطبّق فقط عندما يكون النموذج في `agents.defaults.models`):

| Alias | النموذج |
| ------------------- | -------------------------------------- |
| `opus` | `anthropic/claude-opus-4-6` |
| `sonnet` | `anthropic/claude-sonnet-4-6` |
| `gpt` | `openai/gpt-5.4` |
| `gpt-mini` | `openai/gpt-5.4-mini` |
| `gpt-nano` | `openai/gpt-5.4-nano` |
| `gemini` | `google/gemini-3.1-pro-preview` |
| `gemini-flash` | `google/gemini-3-flash-preview` |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

تفوز الأسماء المستعارة التي تهيئها أنت دائمًا على القيم الافتراضية.

تفعّل نماذج Z.AI ‏GLM-4.x وضع التفكير تلقائيًا ما لم تضبط `--thinking off` أو تعرّف `agents.defaults.models["zai/<model>"].params.thinking` بنفسك.
وتفعّل نماذج Z.AI القيمة `tool_stream` افتراضيًا لبث استدعاء الأداة. اضبط `agents.defaults.models["zai/<model>"].params.tool_stream` على `false` لتعطيله.
وتفترض نماذج Anthropic Claude 4.6 القيمة `adaptive` للتفكير عندما لا يكون هناك مستوى تفكير صريح مضبوط.

### `agents.defaults.cliBackends`

واجهات خلفية CLI اختيارية للتشغيلات الاحتياطية النصية فقط (من دون استدعاءات أدوات). وهي مفيدة كنسخة احتياطية عند فشل موفّري API.

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "claude-cli": {
          command: "/opt/homebrew/bin/claude",
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

- تكون CLI backends موجهة للنص أولًا؛ وتكون الأدوات معطلة دائمًا.
- تُدعم الجلسات عند ضبط `sessionArg`.
- يُدعم تمرير الصور عند قبول `imageArg` لمسارات الملفات.

### `agents.defaults.heartbeat`

تشغيلات heartbeat الدورية.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m للتعطيل
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        lightContext: false, // الافتراضي: false؛ تجعل true الاحتفاظ بـ HEARTBEAT.md فقط من ملفات bootstrap في workspace
        isolatedSession: false, // الافتراضي: false؛ تجعل true كل heartbeat يعمل في جلسة جديدة (بلا سجل محادثة)
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

- `every`: سلسلة مدة (ms/s/m/h). الافتراضي: `30m` ‏(مصادقة API-key) أو `1h` ‏(مصادقة OAuth). اضبطها على `0m` للتعطيل.
- `suppressToolErrorWarnings`: عندما تكون true، تُخفي حمولات تحذيرات أخطاء الأدوات أثناء تشغيل heartbeat.
- `directPolicy`: سياسة التسليم المباشر/الرسائل المباشرة. يسمح `allow` ‏(الافتراضي) بالتسليم المباشر؛ ويمنع `block` التسليم المباشر ويُصدر `reason=dm-blocked`.
- `lightContext`: عندما تكون true، تستخدم تشغيلات heartbeat سياق bootstrap خفيفًا وتحتفظ فقط بـ `HEARTBEAT.md` من ملفات bootstrap في workspace.
- `isolatedSession`: عندما تكون true، يعمل كل heartbeat في جلسة جديدة بلا سجل محادثة سابق. ونمط العزل نفسه الخاص بـ cron ‏`sessionTarget: "isolated"`. ويقلل تكلفة الرموز لكل heartbeat من ~100K إلى ~2-5K tokens.
- لكل وكيل: اضبط `agents.list[].heartbeat`. وعندما يعرّف أي وكيل `heartbeat`، **تعمل فقط تلك الوكلاء**.
- تنفذ heartbeat دورات وكيل كاملة — والفواصل الأقصر تستهلك رموزًا أكثر.

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
        identifierInstructions: "Preserve deployment IDs, ticket IDs, and host:port pairs exactly.", // تُستخدم عندما يكون identifierPolicy=custom
        postCompactionSections: ["Session Startup", "Red Lines"], // [] لتعطيل إعادة الحقن
        model: "openrouter/anthropic/claude-sonnet-4-6", // تجاوز اختياري لنموذج خاص بالضغط فقط
        notifyUser: true, // إرسال إشعار موجز عند بدء الضغط (الافتراضي: false)
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

- `mode`: ‏`default` أو `safeguard` ‏(تلخيص مقسّم إلى أجزاء للتواريخ الطويلة). راجع [Compaction](/concepts/compaction).
- `timeoutSeconds`: الحد الأقصى بالثواني المسموح به لعملية ضغط واحدة قبل أن يُجهضها OpenClaw. الافتراضي: `900`.
- `identifierPolicy`: ‏`strict` ‏(الافتراضي)، أو `off`، أو `custom`. تقوم `strict` بإضافة توجيهات مدمجة للاحتفاظ بالمعرّفات المعتمة أثناء تلخيص الضغط.
- `identifierInstructions`: نص مخصص اختياري للحفاظ على المعرّفات يُستخدم عندما يكون `identifierPolicy=custom`.
- `postCompactionSections`: أسماء أقسام H2/H3 اختيارية من `AGENTS.md` لإعادة حقنها بعد الضغط. الافتراضي `["Session Startup", "Red Lines"]`؛ اضبطها على `[]` لتعطيل إعادة الحقن. وعندما تكون غير مضبوطة أو مضبوطة صراحةً على هذا الزوج الافتراضي، تُقبل أيضًا العناوين القديمة `Every Session`/`Safety` كحل احتياطي قديم.
- `model`: تجاوز اختياري `provider/model-id` لتلخيص الضغط فقط. استخدمه عندما يجب أن تبقي الجلسة الرئيسية نموذجًا ما لكن تُنفذ ملخصات الضغط على نموذج آخر؛ وعند حذفه، يستخدم الضغط النموذج الأساسي للجلسة.
- `notifyUser`: عند `true`، يرسل إشعارًا موجزًا للمستخدم عند بدء الضغط (مثل "Compacting context..."). وهو معطّل افتراضيًا حتى يبقى الضغط صامتًا.
- `memoryFlush`: دورة agentic صامتة قبل الضغط التلقائي لتخزين الذكريات الدائمة. وتُتخطى عندما تكون workspace للقراءة فقط.

### `agents.defaults.contextPruning`

يقتطع **نتائج الأدوات القديمة** من السياق داخل الذاكرة قبل إرسالها إلى LLM. ولا **يعدّل** سجل الجلسة على القرص.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // مدة (ms/s/m/h)، الوحدة الافتراضية: دقائق
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

- يؤدي `mode: "cache-ttl"` إلى تمكين تمريرات الاقتطاع.
- يتحكم `ttl` في عدد المرات التي يمكن أن يعمل فيها الاقتطاع مجددًا (بعد آخر لمسة للذاكرة المؤقتة).
- يقوم الاقتطاع أولًا باقتطاع ناعم soft-trim لنتائج الأدوات كبيرة الحجم، ثم يمسح النتائج الأقدم hard-clear إذا لزم الأمر.

يحافظ **Soft-trim** على البداية + النهاية ويُدرج `...` في الوسط.

ويستبدل **Hard-clear** نتيجة الأداة كاملةً بالعُنصر النائب.

ملاحظات:

- لا يتم أبدًا اقتطاع/مسح كتل الصور.
- تعتمد النسب على الأحرف (تقريبية)، وليس على عدد الرموز بدقة.
- إذا كان عدد رسائل المساعد أقل من `keepLastAssistants`، يتم تخطي الاقتطاع.

</Accordion>

راجع [Session Pruning](/concepts/session-pruning) لتفاصيل السلوك.

### Block streaming

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
- تجاوزات لكل قناة: `channels.<channel>.blockStreamingCoalesce` ‏(ومتغيراتها لكل حساب). تفترض Signal/Slack/Discord/Google Chat القيم `minChars: 1500` افتراضيًا.
- `humanDelay`: إيقاف عشوائي بين الردود الكتلية. `natural` = ‏800–2500ms. تجاوز لكل وكيل: `agents.list[].humanDelay`.

راجع [Streaming](/concepts/streaming) للسلوك + تفاصيل التجزئة.

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

- القيم الافتراضية: `instant` للدردشات المباشرة/الإشارات، و`message` للدردشات الجماعية غير المشار فيها.
- تجاوزات لكل جلسة: `session.typingMode` و`session.typingIntervalSeconds`.

راجع [Typing Indicators](/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

وضع sandbox اختياري للوكيل المدمج. راجع [Sandboxing](/gateway/sandboxing) للحصول على الدليل الكامل.

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
          // تُدعم أيضًا SecretRefs / المحتويات المضمنة:
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

**Backend:**

- `docker`: بيئة تشغيل Docker محلية (الافتراضي)
- `ssh`: بيئة تشغيل بعيدة عامة مدعومة بـ SSH
- `openshell`: بيئة تشغيل OpenShell

عند اختيار `backend: "openshell"`، تنتقل الإعدادات الخاصة ببيئة التشغيل إلى
`plugins.entries.openshell.config`.

**إعداد SSH backend:**

- `target`: هدف SSH بصيغة `user@host[:port]`
- `command`: أمر عميل SSH ‏(الافتراضي: `ssh`)
- `workspaceRoot`: جذر بعيد مطلق يُستخدم لمساحات العمل لكل نطاق
- `identityFile` / `certificateFile` / `knownHostsFile`: ملفات محلية موجودة تُمرَّر إلى OpenSSH
- `identityData` / `certificateData` / `knownHostsData`: محتويات مضمنة أو SecretRefs يحولها OpenClaw إلى ملفات مؤقتة وقت التشغيل
- `strictHostKeyChecking` / `updateHostKeys`: عناصر تحكم سياسة مفاتيح المضيف في OpenSSH

**أسبقية مصادقة SSH:**

- يفوز `identityData` على `identityFile`
- يفوز `certificateData` على `certificateFile`
- يفوز `knownHostsData` على `knownHostsFile`
- تُحل قيم `*Data` المدعومة بـ SecretRef من لقطة بيئة الأسرار النشطة قبل بدء جلسة sandbox

**سلوك SSH backend:**

- يزرع workspace البعيدة مرة واحدة بعد الإنشاء أو إعادة الإنشاء
- ثم يُبقي SSH workspace البعيدة بوصفها الأصل القانوني
- يوجه `exec` وأدوات الملفات ومسارات الوسائط عبر SSH
- لا يزامن التغييرات البعيدة تلقائيًا إلى المضيف
- لا يدعم حاويات متصفح sandbox

**الوصول إلى Workspace:**

- `none`: ‏workspace خاصة بـ sandbox لكل نطاق تحت `~/.openclaw/sandboxes`
- `ro`: ‏workspace داخل sandbox عند `/workspace`، وتُربط agent workspace للقراءة فقط عند `/agent`
- `rw`: تُربط agent workspace للقراءة/الكتابة عند `/workspace`

**Scope:**

- `session`: حاوية + workspace لكل جلسة
- `agent`: حاوية + workspace واحدة لكل وكيل (الافتراضي)
- `shared`: حاوية وworkspace مشتركتان (من دون عزل بين الجلسات)

**تهيئة إضافة OpenShell:**

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

- `mirror`: زرع البعيد من المحلي قبل exec، ثم المزامنة إلى الخلف بعد exec؛ وتبقى workspace المحلية هي الأصل القانوني
- `remote`: زرع البعيد مرة واحدة عند إنشاء sandbox، ثم إبقاء workspace البعيدة هي الأصل القانوني

في وضع `remote`، لا تتم مزامنة التعديلات المحلية على المضيف التي تُجرى خارج OpenClaw تلقائيًا إلى sandbox بعد خطوة الزرع.
تكون وسيلة النقل هي SSH إلى OpenShell sandbox، لكن الإضافة تمتلك دورة حياة sandbox والمزامنة المرآتية الاختيارية.

يعمل `setupCommand` مرة واحدة بعد إنشاء الحاوية (عبر `sh -lc`). ويحتاج إلى خروج شبكة، وجذر قابل للكتابة، ومستخدم root.

**تستخدم الحاويات افتراضيًا `network: "none"`** — اضبطها على `"bridge"` (أو شبكة bridge مخصصة) إذا كان الوكيل يحتاج إلى وصول صادر.
يُحظر `"host"`. ويُحظر `"container:<id>"` افتراضيًا ما لم تضبط صراحةً
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` ‏(وضع طارئ).

**تُجهَّز المرفقات الواردة** في `media/inbound/*` ضمن workspace النشطة.

**يقوم `docker.binds`** بربط أدلة مضيف إضافية؛ وتُدمج الربوط العامة وربوط كل وكيل.

**المتصفح داخل Sandbox** ‏(`sandbox.browser.enabled`): Chromium + CDP في حاوية. ويُحقن عنوان noVNC في system prompt. ولا يتطلب `browser.enabled` في `openclaw.json`.
ويستخدم وصول المراقبة عبر noVNC مصادقة VNC افتراضيًا ويصدر OpenClaw عنوان URL مميزًا قصير العمر (بدلًا من كشف كلمة المرور في الرابط المشترك).

- يؤدي `allowHostControl: false` ‏(الافتراضي) إلى حظر استهداف جلسات sandbox للمتصفح الموجود على المضيف.
- تفترض `network` القيمة `openclaw-sandbox-browser` ‏(شبكة bridge مخصصة). اضبطها على `bridge` فقط عندما تريد صراحةً اتصالًا عامًا عبر bridge.
- تقيّد `cdpSourceRange` اختياريًا إدخال CDP عند حافة الحاوية إلى نطاق CIDR ‏(مثل `172.21.0.1/32`).
- تقوم `sandbox.browser.binds` بربط أدلة مضيف إضافية داخل حاوية متصفح sandbox فقط. وعند ضبطها (بما في ذلك `[]`)، فإنها تستبدل `docker.binds` لحاوية المتصفح.
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
  - `--disable-extensions` ‏(مفعّل افتراضيًا)
  - تكون `--disable-3d-apis` و`--disable-software-rasterizer` و`--disable-gpu`
    مفعلة افتراضيًا ويمكن تعطيلها بواسطة
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` إذا كان استخدام WebGL/3D يتطلب ذلك.
  - تعيد `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` تمكين الإضافات إذا كان سير العمل لديك
    يعتمد عليها.
  - يمكن تغيير `--renderer-process-limit=2` بواسطة
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`؛ اضبطه على `0` لاستخدام
    حد العمليات الافتراضي لـ Chromium.
  - بالإضافة إلى `--no-sandbox` و`--disable-setuid-sandbox` عندما يكون `noSandbox` مفعّلًا.
  - تمثل القيم الافتراضية خط الأساس لصورة الحاوية؛ استخدم صورة متصفح مخصصة ذات
    entrypoint مخصص لتغيير القيم الافتراضية للحاوية.

</Accordion>

تتوفر Browser sandboxing و`sandbox.docker.binds` حاليًا فقط مع Docker.

بناء الصور:

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
        reasoningDefault: "on", // تجاوز ظهور reasoning الافتراضي لكل وكيل
        fastModeDefault: false, // تجاوز fast mode الافتراضي لكل وكيل
        params: { cacheRetention: "none" }, // تتجاوز matching defaults.models params حسب المفتاح
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
- `default`: عند ضبط عدة قيَم، يفوز الأول (مع تسجيل تحذير). وإذا لم يُضبط أي منها، يكون أول إدخال في القائمة هو الافتراضي.
- `model`: يتجاوز شكل السلسلة `primary` فقط؛ أما شكل الكائن `{ primary, fallbacks }` فيتجاوز كليهما (`[]` لتعطيل التراجعات الافتراضية العامة). وما زالت مهام Cron التي تتجاوز `primary` فقط ترث التراجعات الافتراضية ما لم تضبط `fallbacks: []`.
- `params`: معلمات stream لكل وكيل تُدمج فوق إدخال النموذج المحدد في `agents.defaults.models`. استخدم ذلك لتجاوزات خاصة بالوكيل مثل `cacheRetention` أو `temperature` أو `maxTokens` من دون تكرار فهرس النماذج كاملًا.
- `skills`: قائمة سماح اختيارية لـ Skills لكل وكيل. وإذا حُذفت، يرث الوكيل `agents.defaults.skills` عند ضبطها؛ وتستبدل القائمة الصريحة القيم الافتراضية بدلًا من دمجها، وتعني `[]` عدم وجود Skills.
- `thinkingDefault`: مستوى التفكير الافتراضي الاختياري لكل وكيل (`off | minimal | low | medium | high | xhigh | adaptive`). ويتجاوز `agents.defaults.thinkingDefault` لهذا الوكيل عندما لا يكون هناك تجاوز لكل رسالة أو جلسة.
- `reasoningDefault`: ظهور reasoning الافتراضي الاختياري لكل وكيل (`on | off | stream`). ويُطبَّق عندما لا يكون هناك تجاوز reasoning لكل رسالة أو جلسة.
- `fastModeDefault`: القيمة الافتراضية الاختيارية لوضع fast لكل وكيل (`true | false`). وتُطبَّق عندما لا يكون هناك تجاوز لوضع fast لكل رسالة أو جلسة.
- `runtime`: واصف runtime اختياري لكل وكيل. استخدم `type: "acp"` مع القيم الافتراضية لـ `runtime.acp` ‏(`agent` و`backend` و`mode` و`cwd`) عندما يجب أن يستخدم الوكيل جلسات ACP harness افتراضيًا.
- `identity.avatar`: مسار نسبي إلى workspace، أو عنوان URL من نوع `http(s)`، أو `data:` URI.
- تستنتج `identity` القيم الافتراضية: `ackReaction` من `emoji`، و`mentionPatterns` من `name`/`emoji`.
- `subagents.allowAgents`: قائمة سماح لمعرّفات الوكلاء الخاصة بـ `sessions_spawn` ‏(`["*"]` = أي وكيل؛ الافتراضي: الوكيل نفسه فقط).
- حاجز وراثة sandbox: إذا كانت جلسة الطالب داخل sandbox، فإن `sessions_spawn` يرفض الأهداف التي ستعمل من دون sandbox.
- `subagents.requireAgentId`: عند true، يحظر استدعاءات `sessions_spawn` التي تحذف `agentId` ‏(يفرض اختيار profile صريح؛ الافتراضي: false).

---

## التوجيه متعدد الوكلاء

شغّل عدة وكلاء معزولين داخل Gateway واحد. راجع [Multi-Agent](/concepts/multi-agent).

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

### حقول مطابقة Binding

- `type` ‏(اختياري): `route` للتوجيه العادي (وغياب النوع يعني route)، و`acp` لروابط محادثات ACP الدائمة.
- `match.channel` ‏(مطلوب)
- `match.accountId` ‏(اختياري؛ `*` = أي حساب؛ والحذف = الحساب الافتراضي)
- `match.peer` ‏(اختياري؛ `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` ‏(اختياريان؛ خاصان بالقناة)
- `acp` ‏(اختياري؛ فقط لـ `type: "acp"`): ‏`{ mode, label, cwd, backend }`

**ترتيب المطابقة الحتمي:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` ‏(مطابقة دقيقة، من دون peer/guild/team)
5. `match.accountId: "*"` ‏(على مستوى القناة)
6. الوكيل الافتراضي

داخل كل طبقة، يفوز أول إدخال مطابق في `bindings`.

بالنسبة إلى إدخالات `type: "acp"`، يحل OpenClaw وفق هوية المحادثة الدقيقة (`match.channel` + الحساب + `match.peer.id`) ولا يستخدم ترتيب طبقات route أعلاه.

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

<Accordion title="أدوات + Workspace للقراءة فقط">

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

راجع [Multi-Agent Sandbox & Tools](/tools/multi-agent-sandbox-tools) لتفاصيل الأسبقية.

---

## Session

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
    parentForkMaxTokens: 100000, // تخطي fork الخاص بالخيط الأب فوق هذا العدد من الرموز (0 للتعطيل)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // مدة أو false
      maxDiskBytes: "500mb", // ميزانية صارمة اختيارية
      highWaterBytes: "400mb", // هدف تنظيف اختياري
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // الافتراضي لإلغاء التركيز التلقائي بسبب عدم النشاط بالساعات (`0` للتعطيل)
      maxAgeHours: 0, // الافتراضي للعمر الأقصى الصارم بالساعات (`0` للتعطيل)
    },
    mainKey: "main", // قديم (يستخدم وقت التشغيل دائمًا "main")
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="تفاصيل حقول Session">

- **`scope`**: استراتيجية التجميع الأساسية للجلسات في سياقات الدردشة الجماعية.
  - `per-sender` ‏(الافتراضي): يحصل كل مرسل على جلسة معزولة داخل سياق القناة.
  - `global`: يشترك جميع المشاركين في سياق القناة في جلسة واحدة (استخدمه فقط عندما يكون السياق المشترك مقصودًا).
- **`dmScope`**: كيفية تجميع الرسائل المباشرة.
  - `main`: تشترك جميع الرسائل المباشرة في الجلسة الرئيسية.
  - `per-peer`: العزل حسب معرّف المرسل عبر القنوات.
  - `per-channel-peer`: العزل لكل قناة + مرسل (موصى به لصناديق الوارد متعددة المستخدمين).
  - `per-account-channel-peer`: العزل لكل حساب + قناة + مرسل (موصى به لتعدد الحسابات).
- **`identityLinks`**: تعيين المعرّفات القانونية إلى أقران مسبوقين بالموفّر لمشاركة الجلسة عبر القنوات.
- **`reset`**: سياسة إعادة التعيين الأساسية. يعيد `daily` التعيين عند `atHour` حسب التوقيت المحلي؛ ويعيد `idle` التعيين بعد `idleMinutes`. وعندما يُهيأ كلاهما، يفوز ما تنتهي صلاحيته أولًا.
- **`resetByType`**: تجاوزات لكل نوع (`direct` و`group` و`thread`). ويُقبل `dm` القديم كاسم مستعار لـ `direct`.
- **`parentForkMaxTokens`**: الحد الأقصى لقيمة `totalTokens` لجلسة الأب المسموح بها عند إنشاء جلسة سلسلة fork ‏(الافتراضي `100000`).
  - إذا كانت قيمة `totalTokens` في الجلسة الأب أعلى من هذا الحد، يبدأ OpenClaw جلسة سلسلة جديدة بدلًا من وراثة سجل نص الجلسة الأب.
  - اضبطه على `0` لتعطيل هذا الحاجز والسماح دائمًا بعمل fork من الأب.
- **`mainKey`**: حقل قديم. يستخدم وقت التشغيل الآن دائمًا `"main"` لحاوية الدردشة المباشرة الرئيسية.
- **`agentToAgent.maxPingPongTurns`**: الحد الأقصى لعدد أدوار الرد المتبادل بين الوكلاء أثناء تبادلات وكيل-إلى-وكيل (عدد صحيح، المجال: `0`–`5`). وتعني `0` تعطيل تسلسل ping-pong.
- **`sendPolicy`**: المطابقة حسب `channel` أو `chatType` ‏(`direct|group|channel`، مع الاسم القديم `dm` كاسم مستعار)، أو `keyPrefix`، أو `rawKeyPrefix`. ويفوز أول deny.
- **`maintenance`**: عناصر تحكم تنظيف + احتفاظ لمخزن الجلسات.
  - `mode`: يقوم `warn` بإصدار تحذيرات فقط؛ بينما يطبق `enforce` التنظيف.
  - `pruneAfter`: حد عمر للجلسات القديمة (الافتراضي `30d`).
  - `maxEntries`: الحد الأقصى لعدد الإدخالات في `sessions.json` ‏(الافتراضي `500`).
  - `rotateBytes`: تدوير `sessions.json` عندما يتجاوز هذا الحجم (الافتراضي `10mb`).
  - `resetArchiveRetention`: مدة الاحتفاظ بأرشيفات النص `*.reset.<timestamp>`. وتفترض افتراضيًا `pruneAfter`؛ واضبطها على `false` للتعطيل.
  - `maxDiskBytes`: ميزانية اختيارية لدليل الجلسات على القرص. في وضع `warn` يسجل تحذيرات؛ وفي وضع `enforce` يزيل أقدم العناصر/الجلسات أولًا.
  - `highWaterBytes`: هدف اختياري بعد تنظيف الميزانية. ويفترض افتراضيًا `80%` من `maxDiskBytes`.
- **`threadBindings`**: القيم الافتراضية العامة لميزات الجلسات المرتبطة بالسلاسل.
  - `enabled`: مفتاح افتراضي رئيسي (يمكن للموفّرين تجاوزه؛ ويستخدم Discord القيمة `channels.discord.threadBindings.enabled`)
  - `idleHours`: افتراضي إلغاء التركيز التلقائي بسبب عدم النشاط بالساعات (`0` للتعطيل؛ ويمكن للموفّرين التجاوز)
  - `maxAgeHours`: افتراضي العمر الأقصى الصارم بالساعات (`0` للتعطيل؛ ويمكن للموفّرين التجاوز)

</Accordion>

---

## Messages

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

الحل (الأكثر تحديدًا يفوز): account → channel → global. تؤدي `""` إلى التعطيل وإيقاف السلسلة. ويستنتج `"auto"` القيمة `[{identity.name}]`.

**متغيرات القالب:**

| المتغير | الوصف | المثال |
| ----------------- | ---------------------- | --------------------------- |
| `{model}` | اسم النموذج المختصر | `claude-opus-4-6` |
| `{modelFull}` | معرّف النموذج الكامل | `anthropic/claude-opus-4-6` |
| `{provider}` | اسم الموفّر | `anthropic` |
| `{thinkingLevel}` | مستوى التفكير الحالي | `high` أو `low` أو `off` |
| `{identity.name}` | اسم هوية الوكيل | (مثل `"auto"`) |

المتغيرات غير حساسة لحالة الأحرف. ويمثل `{think}` اسمًا مستعارًا لـ `{thinkingLevel}`.

### Ack reaction

- تُشتق افتراضيًا من `identity.emoji` للوكيل النشط، وإلا فالقيمة `"👀"`. اضبط `""` للتعطيل.
- تجاوزات لكل قناة: `channels.<channel>.ackReaction` و`channels.<channel>.accounts.<id>.ackReaction`.
- ترتيب الحل: account → channel → `messages.ackReaction` → الرجوع الاحتياطي إلى identity.
- النطاق: `group-mentions` ‏(الافتراضي)، و`group-all`، و`direct`، و`all`.
- تؤدي `removeAckAfterReply` إلى إزالة ack بعد الرد على Slack وDiscord وTelegram.
- تمكّن `messages.statusReactions.enabled` تفاعلات الحالة المرتبطة بدورة الحياة على Slack وDiscord وTelegram.
  على Slack وDiscord، تعني القيمة غير المضبوطة إبقاء تفاعلات الحالة مفعلة عندما تكون ack reactions نشطة.
  وعلى Telegram، اضبطها صراحةً على `true` لتمكين تفاعلات الحالة المرتبطة بدورة الحياة.

### Inbound debounce

تجمع الرسائل النصية السريعة من المرسل نفسه في دورة وكيل واحدة. وتؤدي الوسائط/المرفقات إلى التفريغ الفوري. وتتجاوز أوامر التحكم عملية debounce.

### TTS ‏(تحويل النص إلى كلام)

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

- تتحكم `auto` في TTS التلقائي. ويؤدي `/tts off|always|inbound|tagged` إلى تجاوز لكل جلسة.
- تتجاوز `summaryModel` القيمة `agents.defaults.model.primary` للتلخيص التلقائي.
- تكون `modelOverrides` مفعلة افتراضيًا؛ وتفترض `modelOverrides.allowProvider` القيمة `false` افتراضيًا (opt-in).
- تعود مفاتيح API احتياطيًا إلى `ELEVENLABS_API_KEY`/`XI_API_KEY` و`OPENAI_API_KEY`.
- تتجاوز `openai.baseUrl` نقطة نهاية OpenAI TTS. وترتيب الحل هو: config، ثم `OPENAI_TTS_BASE_URL`، ثم `https://api.openai.com/v1`.
- عندما تشير `openai.baseUrl` إلى نقطة نهاية غير OpenAI، يتعامل OpenClaw معها كخادم TTS متوافق مع OpenAI ويخفف التحقق من النموذج/الصوت.

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

- يجب أن تطابق `talk.provider` مفتاحًا في `talk.providers` عندما يتم تهيئة عدة موفّري Talk.
- تكون مفاتيح Talk المسطحة القديمة (`talk.voiceId` و`talk.voiceAliases` و`talk.modelId` و`talk.outputFormat` و`talk.apiKey`) للتوافق فقط، ويتم ترحيلها تلقائيًا إلى `talk.providers.<provider>`.
- تعود معرّفات الأصوات احتياطيًا إلى `ELEVENLABS_VOICE_ID` أو `SAG_VOICE_ID`.
- تقبل `providers.*.apiKey` سلاسل نصية صريحة أو كائنات SecretRef.
- يُطبّق الرجوع الاحتياطي إلى `ELEVENLABS_API_KEY` فقط عندما لا يكون مفتاح API لـ Talk مهيأ.
- تسمح `providers.*.voiceAliases` لتوجيهات Talk باستخدام أسماء مألوفة.
- تتحكم `silenceTimeoutMs` في مدة انتظار Talk mode بعد صمت المستخدم قبل إرسال النص المفرغ. ويؤدي حذفها إلى الإبقاء على نافذة التوقف الافتراضية الخاصة بالمنصة (`700 ms على macOS وAndroid، و900 ms على iOS`).

---

## Tools

### ملفات تعريف الأدوات

يضبط `tools.profile` قائمة سماح أساسية قبل `tools.allow`/`tools.deny`:

تضبط عملية onboarding المحلية افتراضيًا الإعدادات المحلية الجديدة على `tools.profile: "coding"` عندما تكون غير مضبوطة (مع الحفاظ على الملفات الشخصية الصريحة الموجودة).

| الملف الشخصي | يتضمن |
| ----------- | ------------------------------------------------------------------------------------------------------------- |
| `minimal` | `session_status` فقط |
| `coding` | `group:fs` و`group:runtime` و`group:web` و`group:sessions` و`group:memory` و`cron` و`image` و`image_generate` |
| `messaging` | `group:messaging` و`sessions_list` و`sessions_history` و`sessions_send` و`session_status` |
| `full` | بلا تقييد (مثل unset) |

### مجموعات الأدوات

| المجموعة | الأدوات |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `group:runtime` | `exec` و`process` و`code_execution` ‏(`bash` يُقبل كاسم مستعار لـ `exec`) |
| `group:fs` | `read` و`write` و`edit` و`apply_patch` |
| `group:sessions` | `sessions_list` و`sessions_history` و`sessions_send` و`sessions_spawn` و`sessions_yield` و`subagents` و`session_status` |
| `group:memory` | `memory_search` و`memory_get` |
| `group:web` | `web_search` و`x_search` و`web_fetch` |
| `group:ui` | `browser` و`canvas` |
| `group:automation` | `cron` و`gateway` |
| `group:messaging` | `message` |
| `group:nodes` | `nodes` |
| `group:agents` | `agents_list` |
| `group:media` | `image` و`image_generate` و`tts` |
| `group:openclaw` | جميع الأدوات المدمجة (باستثناء إضافات الموفّر) |

### `tools.allow` / `tools.deny`

سياسة سماح/منع الأدوات العامة (يفوز deny). غير حساسة لحالة الأحرف، وتدعم wildcards من نوع `*`. وتُطبَّق حتى عند تعطيل Docker sandbox.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

تقييد إضافي للأدوات لموفّرين أو نماذج محددة. الترتيب: base profile → provider profile → allow/deny.

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

يتحكم في وصول exec elevated خارج sandbox:

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

- لا يمكن للتجاوز لكل وكيل (`agents.list[].tools.elevated`) إلا أن يزيد التقييد.
- يخزن `/elevated on|off|ask|full` الحالة لكل جلسة؛ بينما تنطبق التوجيهات المضمنة على رسالة واحدة.
- يتجاوز `exec` elevated وضع sandboxing ويستخدم مسار escape المهيأ (`gateway` افتراضيًا، أو `node` عندما يكون هدف exec هو `node`).

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

تكون فحوصات أمان حلقات الأدوات **معطلة افتراضيًا**. اضبط `enabled: true` لتفعيل الكشف.
يمكن تعريف الإعدادات بشكل عام في `tools.loopDetection` وتجاوزها لكل وكيل عند `agents.list[].tools.loopDetection`.

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

- `historySize`: الحد الأقصى لسجل استدعاءات الأدوات المحتفظ به لتحليل الحلقات.
- `warningThreshold`: عتبة الأنماط المتكررة عديمة التقدم الخاصة بالتحذيرات.
- `criticalThreshold`: عتبة أعلى لحظر الحلقات الحرجة.
- `globalCircuitBreakerThreshold`: عتبة توقف صارمة لأي تشغيل عديم التقدم.
- `detectors.genericRepeat`: التحذير من تكرار الاستدعاء نفسه/المعاملات نفسها.
- `detectors.knownPollNoProgress`: التحذير/الحظر على أدوات polling المعروفة (`process.poll` و`command_status` وما شابه).
- `detectors.pingPong`: التحذير/الحظر على أنماط الأزواج المتناوبة عديمة التقدم.
- إذا كانت `warningThreshold >= criticalThreshold` أو `criticalThreshold >= globalCircuitBreakerThreshold`، يفشل التحقق.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // أو BRAVE_API_KEY من env
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // اختياري؛ احذفه للاكتشاف التلقائي
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

تهيئة فهم الوسائط الواردة (الصورة/الصوت/الفيديو):

```json5
{
  tools: {
    media: {
      concurrency: 2,
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

- `provider`: معرّف موفّر API ‏(`openai` أو `anthropic` أو `google`/`gemini` أو `groq`، وما إلى ذلك)
- `model`: تجاوز معرّف النموذج
- `profile` / `preferredProfile`: اختيار profile من `auth-profiles.json`

**إدخال CLI** ‏(`type: "cli"`):

- `command`: الملف التنفيذي المطلوب تشغيله
- `args`: وسائط بقوالب (تدعم `{{MediaPath}}` و`{{Prompt}}` و`{{MaxChars}}`، وما إلى ذلك)

**الحقول المشتركة:**

- `capabilities`: قائمة اختيارية (`image` أو `audio` أو `video`). القيم الافتراضية: `openai`/`anthropic`/`minimax` → image، و`google` → image+audio+video، و`groq` → audio.
- `prompt` و`maxChars` و`maxBytes` و`timeoutSeconds` و`language`: تجاوزات لكل إدخال.
- عند الفشل، يعود إلى الإدخال التالي.

تتبع مصادقة الموفّر الترتيب القياسي: `auth-profiles.json` → متغيرات البيئة → `models.providers.*.apiKey`.

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

يتحكم في الجلسات التي يمكن استهدافها بواسطة أدوات الجلسة (`sessions_list` و`sessions_history` و`sessions_send`).

الافتراضي: `tree` ‏(الجلسة الحالية + الجلسات التي أنشأتها، مثل subagents).

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
- `agent`: أي جلسة تنتمي إلى معرّف الوكيل الحالي (وقد يتضمن مستخدمين آخرين إذا كنت تشغّل جلسات per-sender تحت معرّف الوكيل نفسه).
- `all`: أي جلسة. وما زال الاستهداف عبر الوكلاء يتطلب `tools.agentToAgent`.
- تقييد sandbox: عندما تكون الجلسة الحالية داخل sandbox وتكون `agents.defaults.sandbox.sessionToolsVisibility="spawned"`، تُفرض visibility إلى `tree` حتى لو كانت `tools.sessions.visibility="all"`.

### `tools.sessions_spawn`

يتحكم في دعم المرفقات المضمنة لـ `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: اضبط true للسماح بمرفقات ملفات مضمنة
        maxTotalBytes: 5242880, // 5 MB إجماليًا عبر كل الملفات
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB لكل ملف
        retainOnSessionKeep: false, // الاحتفاظ بالمرفقات عندما تكون cleanup="keep"
      },
    },
  },
}
```

ملاحظات:

- لا تُدعم المرفقات إلا مع `runtime: "subagent"`. ويرفض ACP runtime ذلك.
- تُجسَّد الملفات داخل workspace الابن في `.openclaw/attachments/<uuid>/` مع `.manifest.json`.
- تُحجب محتويات المرفقات تلقائيًا من حفظ النص.
- يتم التحقق من إدخالات Base64 باستخدام فحوص صارمة للأبجدية/الحشو وحاجز حجم قبل فك الترميز.
- تكون أذونات الملفات `0700` للأدلة و`0600` للملفات.
- يتبع التنظيف سياسة `cleanup`: تقوم `delete` دائمًا بإزالة المرفقات؛ بينما تحتفظ `keep` بها فقط عندما تكون `retainOnSessionKeep: true`.

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

- `model`: النموذج الافتراضي للوكلاء الفرعيين المنشئين. وإذا حُذف، يرث الوكلاء الفرعيون نموذج المستدعي.
- `allowAgents`: قائمة السماح الافتراضية لمعرّفات الوكلاء المستهدفين الخاصة بـ `sessions_spawn` عندما لا يضبط وكيل الطالب `subagents.allowAgents` خاصته (`["*"]` = أي وكيل؛ الافتراضي: الوكيل نفسه فقط).
- `runTimeoutSeconds`: المهلة الافتراضية (بالثواني) لـ `sessions_spawn` عندما يحذف استدعاء الأداة `runTimeoutSeconds`. وتعني `0` عدم وجود مهلة.
- سياسة أدوات لكل subagent: ‏`tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## الموفّرون المخصصون وbase URLs

يستخدم OpenClaw فهرس النماذج المدمج. أضف موفّرين مخصصين عبر `models.providers` في الإعداد أو `~/.openclaw/agents/<agentId>/agent/models.json`.

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
- تجاوز جذر إعداد الوكيل باستخدام `OPENCLAW_AGENT_DIR` ‏(أو `PI_CODING_AGENT_DIR`، وهو اسم مستعار قديم لمتغير بيئة).
- أسبقية الدمج لمعرّفات الموفّرين المتطابقة:
  - تفوز قيم `baseUrl` غير الفارغة في `models.json` الخاصة بالوكيل.
  - تفوز قيم `apiKey` غير الفارغة للوكيل فقط عندما لا يكون ذلك الموفّر مُدارًا عبر SecretRef في سياق config/auth-profile الحالي.
  - تُحدّث قيم `apiKey` المُدارة عبر SecretRef من علامات المصدر (`ENV_VAR_NAME` لمراجع env، و`secretref-managed` لمراجع file/exec) بدلًا من حفظ الأسرار المحلولة.
  - تُحدّث قيم رؤوس الموفّر المُدارة عبر SecretRef من علامات المصدر (`secretref-env:ENV_VAR_NAME` لمراجع env، و`secretref-managed` لمراجع file/exec).
  - تعود قيم `apiKey`/`baseUrl` الفارغة أو المحذوفة للوكيل إلى `models.providers` في الإعداد.
  - تستخدم قيم `contextWindow`/`maxTokens` للنماذج المتطابقة القيمة الأعلى بين الإعداد الصريح وقيم الفهرس الضمنية.
  - يحافظ `contextTokens` للنماذج المتطابقة على حد وقت تشغيل صريح عندما يكون موجودًا؛ استخدمه لتقييد السياق الفعّال من دون تغيير بيانات النموذج الأصلية.
  - استخدم `models.mode: "replace"` عندما تريد أن يعيد الإعداد كتابة `models.json` بالكامل.
  - يكون حفظ العلامات محكومًا بالمصدر: تُكتب العلامات من لقطة إعداد المصدر النشطة (قبل الحل)، وليس من قيم الأسرار المحلولة وقت التشغيل.

### تفاصيل حقول الموفّر

- `models.mode`: سلوك فهرس الموفّر (`merge` أو `replace`).
- `models.providers`: خريطة الموفّرين المخصصين مفهرسة حسب معرّف الموفّر.
- `models.providers.*.api`: مهايئ الطلب (`openai-completions` أو `openai-responses` أو `anthropic-messages` أو `google-generative-ai`، وغير ذلك).
- `models.providers.*.apiKey`: بيانات اعتماد الموفّر (يُفضّل SecretRef/env substitution).
- `models.providers.*.auth`: استراتيجية المصادقة (`api-key` أو `token` أو `oauth` أو `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: بالنسبة إلى Ollama + `openai-completions`، يحقن `options.num_ctx` في الطلبات (الافتراضي: `true`).
- `models.providers.*.authHeader`: فرض نقل بيانات الاعتماد في ترويسة `Authorization` عند الحاجة.
- `models.providers.*.baseUrl`: عنوان URL الأساسي لواجهة API upstream.
- `models.providers.*.headers`: ترويسات ثابتة إضافية لتوجيه proxy/tenant.
- `models.providers.*.request`: تجاوزات النقل لطلبات HTTP الخاصة بموفّر النموذج.
  - `request.headers`: ترويسات إضافية (تُدمج مع القيم الافتراضية للموفّر). وتقبل القيم SecretRef.
  - `request.auth`: تجاوز استراتيجية المصادقة. الأوضاع: `"provider-default"` ‏(استخدام مصادقة الموفّر المدمجة)، و`"authorization-bearer"` ‏(مع `token`)، و`"header"` ‏(مع `headerName` و`value` و`prefix` الاختياري).
  - `request.proxy`: تجاوز HTTP proxy. الأوضاع: `"env-proxy"` ‏(استخدام متغيرات البيئة `HTTP_PROXY`/`HTTPS_PROXY`) و`"explicit-proxy"` ‏(مع `url`). ويقبل كلا الوضعين كائن `tls` اختياريًا.
  - `request.tls`: تجاوز TLS للاتصالات المباشرة. الحقول: `ca` و`cert` و`key` و`passphrase` ‏(تقبل جميعها SecretRef)، و`serverName`، و`insecureSkipVerify`.
- `models.providers.*.models`: إدخالات فهرس نماذج الموفّر الصريحة.
- `models.providers.*.models.*.contextWindow`: بيانات نافذة السياق الأصلية للنموذج.
- `models.providers.*.models.*.contextTokens`: حد سياق اختياري لوقت التشغيل. استخدمه عندما تريد ميزانية سياق فعالة أصغر من `contextWindow` الأصلية للنموذج.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: تلميح توافق اختياري. بالنسبة إلى `api: "openai-completions"` مع `baseUrl` غير أصلي وغير فارغ (مضيفه ليس `api.openai.com`)، يفرض OpenClaw هذه القيمة إلى `false` وقت التشغيل. أما `baseUrl` الفارغ/المحذوف فيُبقي السلوك الافتراضي الخاص بـ OpenAI.
- `plugins.entries.amazon-bedrock.config.discovery`: جذر إعدادات الاكتشاف التلقائي لـ Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: تشغيل/إيقاف الاكتشاف الضمني.
- `plugins.entries.amazon-bedrock.config.discovery.region`: منطقة AWS الخاصة بالاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: مرشح اختياري لمعرّف الموفّر للاكتشاف الموجّه.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: فاصل polling لتحديث الاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: نافذة سياق احتياطية للنماذج المكتشفة.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: الحد الافتراضي الأقصى لرموز الخرج للنماذج المكتشفة.

### أمثلة للموفّرين

<Accordion title="Cerebras ‏(GLM 4.6 / 4.7)">

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

استخدم `cerebras/zai-glm-4.7` مع Cerebras؛ واستخدم `zai/glm-4.7` مع Z.AI direct.

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

اضبط `OPENCODE_API_KEY` ‏(أو `OPENCODE_ZEN_API_KEY`). استخدم مراجع `opencode/...` لفهرس Zen أو `opencode-go/...` لفهرس Go. اختصار: `openclaw onboard --auth-choice opencode-zen` أو `openclaw onboard --auth-choice opencode-go`.

</Accordion>

<Accordion title="Z.AI ‏(GLM-4.7)">

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
- نقطة النهاية الخاصة بالبرمجة (الافتراضي): `https://api.z.ai/api/coding/paas/v4`
- بالنسبة إلى نقطة النهاية العامة، عرّف موفّرًا مخصصًا مع تجاوز base URL.

</Accordion>

<Accordion title="Moonshot AI ‏(Kimi)">

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

تعلن نقاط النهاية الأصلية لـ Moonshot عن توافق استخدام البث على واجهة
`openai-completions` المشتركة، ويقوم OpenClaw الآن بربط ذلك
بقدرات نقطة النهاية بدلًا من الاعتماد على معرّف الموفّر المدمج وحده.

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

متوافق مع Anthropic، موفّر مدمج. اختصار: `openclaw onboard --auth-choice kimi-code-api-key`.

</Accordion>

<Accordion title="Synthetic ‏(متوافق مع Anthropic)">

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

يجب أن يحذف Base URL اللاحقة `/v1` ‏(عميل Anthropic يضيفها). اختصار: `openclaw onboard --auth-choice synthetic-api-key`.

</Accordion>

<Accordion title="MiniMax M2.7 ‏(مباشر)">

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

اضبط `MINIMAX_API_KEY`. اختصارات:
`openclaw onboard --auth-choice minimax-global-api` أو
`openclaw onboard --auth-choice minimax-cn-api`.
يفترض فهرس النماذج الآن M2.7 فقط.
وفي مسار البث المتوافق مع Anthropic، يعطّل OpenClaw التفكير في MiniMax
افتراضيًا ما لم تضبط `thinking` بنفسك صراحة. ويؤدي `/fast on` أو
`params.fastMode: true` إلى إعادة كتابة `MiniMax-M2.7` إلى
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="النماذج المحلية ‏(LM Studio)">

راجع [النماذج المحلية](/gateway/local-models). باختصار: شغّل نموذجًا محليًا كبيرًا عبر LM Studio Responses API على عتاد قوي؛ واحتفظ بالنماذج المستضافة مدمجة للتراجع الاحتياطي.

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
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // أو سلسلة نصية صريحة
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: قائمة سماح اختيارية لـ Skills المدمجة فقط (ولا تتأثر Skills المُدارة/الخاصة بمساحة العمل).
- `load.extraDirs`: جذور Skills مشتركة إضافية (أدنى أسبقية).
- `install.preferBrew`: عند true، يفضّل أدوات التثبيت الخاصة بـ Homebrew عندما يكون `brew` متاحًا
  قبل الرجوع إلى أنواع تثبيت أخرى.
- `install.nodeManager`: تفضيل مدير node لمواصفات `metadata.openclaw.install`
  ‏(`npm` | `pnpm` | `yarn` | `bun`).
- يؤدي `entries.<skillKey>.enabled: false` إلى تعطيل Skill حتى لو كانت مدمجة/مثبتة.
- `entries.<skillKey>.apiKey`: حقل API key مريح على مستوى Skill ‏(عندما تدعمه Skill).

---

## Plugins

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

- تُحمَّل من `~/.openclaw/extensions` و`<workspace>/.openclaw/extensions` بالإضافة إلى `plugins.load.paths`.
- يقبل الاكتشاف إضافات OpenClaw الأصلية بالإضافة إلى الحزم المتوافقة مع Codex وحزم Claude، بما في ذلك حزم Claude ذات التخطيط الافتراضي من دون manifest.
- **تتطلب تغييرات الإعداد إعادة تشغيل gateway.**
- `allow`: قائمة سماح اختيارية (لا تُحمَّل إلا الإضافات المُدرجة). ويفوز `deny`.
- `plugins.entries.<id>.apiKey`: حقل API key مريح على مستوى الإضافة (عندما تدعمه الإضافة).
- `plugins.entries.<id>.env`: خريطة متغيرات env ضمن نطاق الإضافة.
- `plugins.entries.<id>.hooks.allowPromptInjection`: عندما تكون `false`، تمنع core ‏`before_prompt_build` وتتجاهل الحقول المعدِّلة للموجّه من `before_agent_start` القديم، مع الحفاظ على `modelOverride` و`providerOverride` القديمين. وينطبق ذلك على hooks الإضافات الأصلية ودلائل hooks المقدمة من الحزم المدعومة.
- `plugins.entries.<id>.subagent.allowModelOverride`: الثقة صراحةً في هذه الإضافة لطلب تجاوزات `provider` و`model` لكل تشغيل بالنسبة إلى التشغيلات الخلفية لـ subagent.
- `plugins.entries.<id>.subagent.allowedModels`: قائمة سماح اختيارية لأهداف `provider/model` القانونية الخاصة بتجاوزات subagent الموثوق بها. استخدم `"*"` فقط عندما تريد عمدًا السماح بأي نموذج.
- `plugins.entries.<id>.config`: كائن إعداد يعرّفه plugin ‏(يخضع للتحقق بواسطة مخطط الإضافة الأصلي في OpenClaw عندما يتوفر).
- `plugins.entries.firecrawl.config.webFetch`: إعدادات موفّر Firecrawl web-fetch.
  - `apiKey`: مفتاح API لـ Firecrawl ‏(يقبل SecretRef). ويعود احتياطيًا إلى `plugins.entries.firecrawl.config.webSearch.apiKey`، أو `tools.web.fetch.firecrawl.apiKey` القديم، أو متغير البيئة `FIRECRAWL_API_KEY`.
  - `baseUrl`: عنوان API الأساسي لـ Firecrawl ‏(الافتراضي: `https://api.firecrawl.dev`).
  - `onlyMainContent`: استخراج المحتوى الرئيسي فقط من الصفحات (الافتراضي: `true`).
  - `maxAgeMs`: الحد الأقصى لعمر الذاكرة المؤقتة بالميلي ثانية (الافتراضي: `172800000` / يومان).
  - `timeoutSeconds`: مهلة طلب الكشط بالثواني (الافتراضي: `60`).
- `plugins.entries.xai.config.xSearch`: إعدادات xAI X Search ‏(بحث Grok على الويب).
  - `enabled`: تمكين موفّر X Search.
  - `model`: نموذج Grok المطلوب استخدامه للبحث (مثل `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming`: إعدادات memory dreaming ‏(تجريبية). راجع [Dreaming](/concepts/memory-dreaming) للأوضاع والعتبات.
  - `mode`: إعداد مسبق لوتيرة dreaming ‏(`"off"` أو `"core"` أو `"rem"` أو `"deep"`). الافتراضي: `"off"`.
  - `cron`: تجاوز اختياري لتعريف cron الخاص بجدول dreaming.
  - `timezone`: المنطقة الزمنية لتقييم الجدول (وتعود احتياطيًا إلى `agents.defaults.userTimezone`).
  - `limit`: الحد الأقصى للمرشحين المراد ترقيتهم في كل دورة.
  - `minScore`: العتبة الدنيا للدرجة المرجّحة للترقية.
  - `minRecallCount`: العتبة الدنيا لعدد مرات الاستدعاء.
  - `minUniqueQueries`: العتبة الدنيا لعدد الاستعلامات المميزة.
- يمكن أيضًا لحزم Claude المفعلة أن تساهم بقيم Pi افتراضية مدمجة من `settings.json`؛ ويطبّقها OpenClaw بوصفها إعدادات وكيل منظفة، وليس كتصحيحات إعداد خامة لـ OpenClaw.
- `plugins.slots.memory`: اختر معرّف plugin الذاكرة النشطة، أو `"none"` لتعطيل إضافات الذاكرة.
- `plugins.slots.contextEngine`: اختر معرّف plugin محرك السياق النشط؛ ويفترض `"legacy"` افتراضيًا ما لم تثبت وتحدد محركًا آخر.
- `plugins.installs`: بيانات تعريف تثبيت يديرها CLI وتُستخدم بواسطة `openclaw plugins update`.
  - تتضمن `source` و`spec` و`sourcePath` و`installPath` و`version` و`resolvedName` و`resolvedVersion` و`resolvedSpec` و`integrity` و`shasum` و`resolvedAt` و`installedAt`.
  - تعامل مع `plugins.installs.*` على أنها حالة مُدارة؛ ويفضّل استخدام أوامر CLI بدل التعديلات اليدوية.

راجع [Plugins](/tools/plugin).

---

## Browser

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // وضع الشبكة الموثوقة الافتراضي
      // allowPrivateNetwork: true, // اسم مستعار قديم
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
- تفترض `ssrfPolicy.dangerouslyAllowPrivateNetwork` القيمة `true` عندما تكون غير مضبوطة (نموذج الشبكة الموثوقة).
- اضبط `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` للحصول على تنقل متصفح صارم إلى الشبكات العامة فقط.
- في الوضع الصارم، تخضع نقاط نهاية CDP البعيدة (`profiles.*.cdpUrl`) لحظر الشبكات الخاصة نفسه أثناء فحوصات الوصول/الاكتشاف.
- ما زالت `ssrfPolicy.allowPrivateNetwork` مدعومة كاسم مستعار قديم.
- في الوضع الصارم، استخدم `ssrfPolicy.hostnameAllowlist` و`ssrfPolicy.allowedHostnames` من أجل الاستثناءات الصريحة.
- تكون الملفات الشخصية البعيدة attach-only ‏(بدء/إيقاف/إعادة تعيين معطلة).
- تقبل `profiles.*.cdpUrl` القيم `http://` و`https://` و`ws://` و`wss://`.
  استخدم HTTP(S) عندما تريد أن يكتشف OpenClaw المسار `/json/version`؛ واستخدم WS(S)
  عندما يزوّدك الموفّر بعنوان WebSocket مباشر لـ DevTools.
- تكون ملفات `existing-session` خاصة بالمضيف فقط وتستخدم Chrome MCP بدل CDP.
- يمكن لملفات `existing-session` أن تضبط `userDataDir` لاستهداف ملف تعريف
  محدد في متصفح قائم على Chromium مثل Brave أو Edge.
- تحتفظ ملفات `existing-session` بحدود مسار Chrome MCP الحالية:
  إجراءات قائمة على snapshot/ref بدل الاستهداف عبر CSS selectors، وخطافات رفع ملف واحد،
  وعدم وجود تجاوزات لمهلة dialog، وعدم وجود `wait --load networkidle`، ولا
  `responsebody`، أو تصدير PDF، أو اعتراض تنزيلات، أو إجراءات مجمعة.
- تقوم ملفات `openclaw` المحلية المدارة بتعيين `cdpPort` و`cdpUrl` تلقائيًا؛ لا تضبط
  `cdpUrl` صراحة إلا في حالة CDP البعيد.
- ترتيب الاكتشاف التلقائي: المتصفح الافتراضي إذا كان قائمًا على Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- خدمة التحكم: loopback فقط (منفذ مشتق من `gateway.port`، الافتراضي `18791`).
- تضيف `extraArgs` علامات تشغيل إضافية إلى بدء Chromium المحلي (مثل
  `--disable-gpu`، أو ضبط حجم النافذة، أو علامات التصحيح).

---

## UI

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // emoji أو نص قصير أو URL صورة أو data URI
    },
  },
}
```

- `seamColor`: لون التمييز لواجهة التطبيق الأصلية (مثل لون فقاعة Talk Mode، وما إلى ذلك).
- `assistant`: تجاوز هوية Control UI. وتعود احتياطيًا إلى هوية الوكيل النشط.

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
      // password: "your-password", // أو OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // لـ mode=trusted-proxy؛ راجع /gateway/trusted-proxy-auth
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
      // allowedOrigins: ["https://control.example.com"], // مطلوبة لـ Control UI غير loopback
      // dangerouslyAllowHostHeaderOriginFallback: false, // وضع Host-header origin fallback الخطير
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
    // اختياري. الافتراضي false.
    allowRealIpFallback: false,
    tools: {
      // عناصر deny إضافية لـ /tools/invoke HTTP
      deny: ["browser"],
      // إزالة أدوات من قائمة deny الافتراضية الخاصة بـ HTTP
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

- `mode`: ‏`local` ‏(تشغيل gateway) أو `remote` ‏(الاتصال بـ remote gateway). وترفض Gateway البدء إلا في حالة `local`.
- `port`: منفذ multiplexed واحد لـ WS + HTTP. الأسبقية: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: ‏`auto` أو `loopback` ‏(الافتراضي) أو `lan` ‏(`0.0.0.0`) أو `tailnet` ‏(عنوان Tailscale IP فقط) أو `custom`.
- **أسماء bind المستعارة القديمة**: استخدم قيم وضع bind في `gateway.bind` ‏(`auto` أو `loopback` أو `lan` أو `tailnet` أو `custom`) وليس أسماء المضيف المستعارة (`0.0.0.