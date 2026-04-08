---
read_when:
    - تحتاج إلى دلالات إعدادات دقيقة على مستوى الحقول أو القيم الافتراضية
    - أنت تتحقق من كتل إعدادات القناة أو النموذج أو البوابة أو الأدوات
summary: مرجع كامل لكل مفتاح إعدادات في OpenClaw، والقيم الافتراضية، وإعدادات القنوات
title: مرجع الإعدادات
x-i18n:
    generated_at: "2026-04-08T02:19:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2c7991b948cbbb7954a3e26280089ab00088e7f4878ec0b0540c3c9acf222ebb
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# مرجع الإعدادات

كل الحقول المتاحة في `~/.openclaw/openclaw.json`. للحصول على نظرة عامة موجهة حسب المهام، راجع [الإعدادات](/ar/gateway/configuration).

تنسيق الإعدادات هو **JSON5** (يُسمح بالتعليقات + الفواصل اللاحقة). جميع الحقول اختيارية — يستخدم OpenClaw قيمًا افتراضية آمنة عند حذفها.

---

## القنوات

تبدأ كل قناة تلقائيًا عندما يكون قسم إعداداتها موجودًا (ما لم يكن `enabled: false`).

### الوصول إلى الرسائل الخاصة والمجموعات

تدعم جميع القنوات سياسات الرسائل الخاصة وسياسات المجموعات:

| سياسة الرسائل الخاصة | السلوك                                                          |
| -------------------- | --------------------------------------------------------------- |
| `pairing` (افتراضي)  | يحصل المرسلون غير المعروفين على رمز إقران لمرة واحدة؛ يجب أن يوافق المالك |
| `allowlist`          | فقط المرسلون الموجودون في `allowFrom` (أو مخزن السماح المقترن)    |
| `open`               | السماح بجميع الرسائل الخاصة الواردة (يتطلب `allowFrom: ["*"]`)   |
| `disabled`           | تجاهل جميع الرسائل الخاصة الواردة                                |

| سياسة المجموعة         | السلوك                                                 |
| ---------------------- | ------------------------------------------------------ |
| `allowlist` (افتراضي)  | فقط المجموعات المطابقة لقائمة السماح المكوّنة           |
| `open`                 | تجاوز قوائم السماح للمجموعات (مع استمرار تطبيق تقييد الإشارات) |
| `disabled`             | حظر جميع رسائل المجموعات/الغرف                         |

<Note>
يضبط `channels.defaults.groupPolicy` القيمة الافتراضية عندما تكون `groupPolicy` الخاصة بالمزوّد غير معيّنة.
تنتهي صلاحية رموز الإقران بعد ساعة واحدة. يتم تحديد الحد الأقصى لطلبات إقران الرسائل الخاصة المعلقة إلى **3 لكل قناة**.
إذا كانت كتلة المزوّد مفقودة بالكامل (`channels.<provider>` غير موجودة)، فإن سياسة المجموعة وقت التشغيل تعود إلى `allowlist` (إغلاق افتراضي آمن) مع تحذير عند بدء التشغيل.
</Note>

### تجاوزات نموذج القناة

استخدم `channels.modelByChannel` لتثبيت معرّفات قنوات محددة على نموذج معيّن. تقبل القيم `provider/model` أو الأسماء المستعارة للنماذج المكوّنة. يتم تطبيق تعيين القناة عندما لا تكون الجلسة تملك بالفعل تجاوزًا للنموذج (على سبيل المثال، تم تعيينه عبر `/model`).

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

### القيم الافتراضية للقنوات والنبض الصحي

استخدم `channels.defaults` لسلوك سياسة المجموعة والنبض الصحي المشترك بين المزوّدين:

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

- `channels.defaults.groupPolicy`: سياسة المجموعة الاحتياطية عندما لا تكون `groupPolicy` على مستوى المزوّد معيّنة.
- `channels.defaults.contextVisibility`: وضع رؤية السياق الإضافي الافتراضي لجميع القنوات. القيم: `all` (افتراضي، تضمين كل سياق الاقتباس/الخيط/السجل)، `allowlist` (تضمين السياق فقط من المرسلين المسموح لهم)، `allowlist_quote` (مثل allowlist لكن مع الاحتفاظ بسياق الاقتباس/الرد الصريح). تجاوز لكل قناة: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: تضمين حالات القنوات السليمة في مخرجات النبض الصحي.
- `channels.defaults.heartbeat.showAlerts`: تضمين حالات التدهور/الأخطاء في مخرجات النبض الصحي.
- `channels.defaults.heartbeat.useIndicator`: عرض مخرجات النبض الصحي بأسلوب مؤشر مدمج.

### WhatsApp

يعمل WhatsApp عبر قناة الويب الخاصة بالبوابة (Baileys Web). يبدأ تلقائيًا عندما توجد جلسة مرتبطة.

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

- تستخدم الأوامر الصادرة الحساب `default` افتراضيًا إن كان موجودًا؛ وإلا فمعرّف الحساب الأول المكوَّن (بعد الفرز).
- يجاوز `channels.whatsapp.defaultAccount` الاختياري هذا الحساب الافتراضي الاحتياطي عندما يطابق معرّف حساب مكوَّن.
- يتم ترحيل دليل مصادقة Baileys القديم أحادي الحساب بواسطة `openclaw doctor` إلى `whatsapp/default`.
- تجاوزات لكل حساب: `channels.whatsapp.accounts.<id>.sendReadReceipts`، و`channels.whatsapp.accounts.<id>.dmPolicy`، و`channels.whatsapp.accounts.<id>.allowFrom`.

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

- رمز البوت: `channels.telegram.botToken` أو `channels.telegram.tokenFile` (ملف عادي فقط؛ تُرفض الروابط الرمزية)، مع `TELEGRAM_BOT_TOKEN` كخيار احتياطي للحساب الافتراضي.
- يجاوز `channels.telegram.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّن.
- في إعدادات تعدد الحسابات (معرّفان أو أكثر)، عيّن افتراضيًا صريحًا (`channels.telegram.defaultAccount` أو `channels.telegram.accounts.default`) لتجنب التوجيه الاحتياطي؛ ويُظهر `openclaw doctor` تحذيرًا عندما يكون هذا مفقودًا أو غير صالح.
- `configWrites: false` يمنع عمليات كتابة الإعدادات التي يبدؤها Telegram (ترحيلات معرّف supergroup و`/config set|unset`).
- تُكوّن إدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` روابط ACP دائمة لموضوعات المنتدى (استخدم الصيغة القياسية `chatId:topic:topicId` في `match.peer.id`). تُشارك دلالات الحقول في [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- تستخدم معاينات البث في Telegram `sendMessage` + `editMessageText` (وتعمل في الدردشات المباشرة والجماعية).
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
- تستخدم الاستدعاءات الصادرة المباشرة التي توفر `token` صريحًا لـ Discord ذلك الرمز لهذا الاستدعاء؛ بينما تظل إعدادات إعادة المحاولة/السياسة الخاصة بالحساب آتية من الحساب المحدد في اللقطة النشطة وقت التشغيل.
- يجاوز `channels.discord.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّن.
- استخدم `user:<id>` (رسالة خاصة) أو `channel:<id>` (قناة guild) كأهداف للتسليم؛ ويتم رفض المعرّفات الرقمية المجردة.
- تكون Slugات الـ Guild بحروف صغيرة مع استبدال المسافات بـ `-`؛ وتستخدم مفاتيح القنوات الاسم المُحوّل إلى slug (من دون `#`). يفضَّل استخدام معرّفات الـ Guild.
- يتم تجاهل الرسائل التي يكتبها البوت افتراضيًا. يفعّل `allowBots: true` استقبالها؛ واستخدم `allowBots: "mentions"` لقبول رسائل البوت التي تذكر البوت فقط (مع استمرار تصفية رسائل البوت نفسه).
- `channels.discord.guilds.<id>.ignoreOtherMentions` (وتجاوزات القنوات) يُسقط الرسائل التي تذكر مستخدمًا أو دورًا آخر لكن لا تذكر البوت (باستثناء @everyone/@here).
- `maxLinesPerMessage` (الافتراضي 17) يقسم الرسائل الطويلة رأسيًا حتى لو كانت أقل من 2000 حرف.
- يتحكم `channels.discord.threadBindings` في التوجيه المرتبط بخيوط Discord:
  - `enabled`: تجاوز Discord لميزات الجلسات المرتبطة بالخيوط (`/focus`، `/unfocus`، `/agents`، `/session idle`، `/session max-age`، والتسليم/التوجيه المرتبط)
  - `idleHours`: تجاوز Discord لإلغاء التركيز التلقائي بعد عدم النشاط بالساعات (`0` للتعطيل)
  - `maxAgeHours`: تجاوز Discord للعمر الأقصى الصارم بالساعات (`0` للتعطيل)
  - `spawnSubagentSessions`: مفتاح اشتراك اختياري لإنشاء/ربط الخيط تلقائيًا عبر `sessions_spawn({ thread: true })`
- تُكوّن إدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` روابط ACP دائمة للقنوات والخيوط (استخدم معرّف القناة/الخيط في `match.peer.id`). تُشارك دلالات الحقول في [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- يضبط `channels.discord.ui.components.accentColor` لون التمييز لحاويات Discord components v2.
- يفعّل `channels.discord.voice` محادثات قنوات الصوت في Discord مع الانضمام التلقائي الاختياري + تجاوزات TTS.
- يمرّر `channels.discord.voice.daveEncryption` و`channels.discord.voice.decryptionFailureTolerance` إلى خيارات DAVE في `@discordjs/voice` (افتراضيًا `true` و`24`).
- يحاول OpenClaw أيضًا استعادة استقبال الصوت بمغادرة/إعادة الانضمام إلى جلسة صوتية بعد تكرار فشل فك التشفير.
- `channels.discord.streaming` هو مفتاح وضع البث القياسي. يتم ترحيل `streamMode` القديم وقيم `streaming` المنطقية تلقائيًا.
- يربط `channels.discord.autoPresence` حالة التوفّر وقت التشغيل بحالة حضور البوت (سليم => online، متدهور => idle، منهك => dnd) ويسمح بتجاوزات نص الحالة الاختيارية.
- يعيد `channels.discord.dangerouslyAllowNameMatching` تمكين مطابقة الاسم/الوسم القابلة للتغيير (وضع توافق لكسر الزجاج).
- `channels.discord.execApprovals`: تسليم موافقات exec الأصلية في Discord وتفويض الموافقين.
  - `enabled`: `true` أو `false` أو `"auto"` (افتراضي). في الوضع التلقائي، تُفعَّل موافقات exec عندما يمكن حل الموافقين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Discord المسموح لهم بالموافقة على طلبات exec. تعود إلى `commands.ownerAllowFrom` عند حذفها.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتمرير الموافقات لجميع الوكلاء.
  - `sessionFilter`: أنماط اختيارية لمفاتيح الجلسات (substring أو regex).
  - `target`: مكان إرسال مطالبات الموافقة. `"dm"` (افتراضي) يرسل إلى الرسائل الخاصة للموافقين، `"channel"` يرسل إلى القناة الأصلية، `"both"` يرسل إلى الاثنين. عندما يتضمن الهدف `"channel"`، لا يمكن استخدام الأزرار إلا من قبل الموافقين الذين تم حلهم.
  - `cleanupAfterResolve`: عندما يكون `true`، يحذف رسائل الموافقة الخاصة بعد الموافقة أو الرفض أو انتهاء المهلة.

**أوضاع إشعارات التفاعلات:** `off` (بلا شيء)، `own` (رسائل البوت، افتراضي)، `all` (كل الرسائل)، `allowlist` (من `guilds.<id>.users` على كل الرسائل).

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

- JSON لحساب الخدمة: مضمّن (`serviceAccount`) أو قائم على ملف (`serviceAccountFile`).
- كما أن SecretRef لحساب الخدمة مدعوم أيضًا (`serviceAccountRef`).
- بدائل البيئة: `GOOGLE_CHAT_SERVICE_ACCOUNT` أو `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- استخدم `spaces/<spaceId>` أو `users/<userId>` كأهداف للتسليم.
- يعيد `channels.googlechat.dangerouslyAllowNameMatching` تمكين مطابقة principal البريد الإلكتروني القابلة للتغيير (وضع توافق لكسر الزجاج).

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

- **وضع Socket** يتطلب كلاً من `botToken` و`appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` كبديل بيئي للحساب الافتراضي).
- **وضع HTTP** يتطلب `botToken` بالإضافة إلى `signingSecret` (على الجذر أو لكل حساب).
- تقبل `botToken` و`appToken` و`signingSecret` و`userToken` سلاسل نصية صريحة أو كائنات SecretRef.
- تعرض لقطات حساب Slack حقول مصدر/حالة لكل اعتماد مثل `botTokenSource` و`botTokenStatus` و`appTokenStatus`، وفي وضع HTTP أيضًا `signingSecretStatus`. تعني `configured_unavailable` أن الحساب مكوَّن عبر SecretRef لكن مسار الأمر/وقت التشغيل الحالي لم يتمكن من حل قيمة السر.
- `configWrites: false` يمنع عمليات كتابة الإعدادات التي يبدؤها Slack.
- يجاوز `channels.slack.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّن.
- `channels.slack.streaming` هو مفتاح وضع البث القياسي. يتم ترحيل `streamMode` القديم وقيم `streaming` المنطقية تلقائيًا.
- استخدم `user:<id>` (رسالة خاصة) أو `channel:<id>` كأهداف للتسليم.

**أوضاع إشعارات التفاعلات:** `off`، `own` (افتراضي)، `all`، `allowlist` (من `reactionAllowlist`).

**عزل جلسة الخيط:** يكون `thread.historyScope` لكل خيط على حدة (افتراضي) أو مشتركًا عبر القناة. ينسخ `thread.inheritParent` سجل القناة الأصلية إلى الخيوط الجديدة.

- يضيف `typingReaction` تفاعلًا مؤقتًا إلى رسالة Slack الواردة أثناء تشغيل الرد، ثم يزيله عند الاكتمال. استخدم shortcode لرمز Slack التعبيري مثل `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: تسليم موافقات exec الأصلية في Slack وتفويض الموافقين. نفس مخطط Discord: `enabled` (`true`/`false`/`"auto"`)، و`approvers` (معرّفات مستخدمي Slack)، و`agentFilter`، و`sessionFilter`، و`target` (`"dm"` أو `"channel"` أو `"both"`).

| مجموعة الإجراءات | الافتراضي | الملاحظات              |
| ---------------- | --------- | ---------------------- |
| reactions        | مفعّل     | التفاعل + عرض التفاعلات |
| messages         | مفعّل     | قراءة/إرسال/تحرير/حذف  |
| pins             | مفعّل     | تثبيت/إلغاء تثبيت/عرض  |
| memberInfo       | مفعّل     | معلومات العضو          |
| emojiList        | مفعّل     | قائمة الإيموجي المخصصة  |

### Mattermost

يأتي Mattermost كإضافة plugin: `openclaw plugins install @openclaw/mattermost`.

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

أوضاع الدردشة: `oncall` (الرد عند @-mention، افتراضي)، `onmessage` (كل رسالة)، `onchar` (الرسائل التي تبدأ ببادئة التشغيل).

عندما تكون الأوامر الأصلية لـ Mattermost مفعلة:

- يجب أن يكون `commands.callbackPath` مسارًا (مثل `/api/channels/mattermost/command`)، وليس URL كاملًا.
- يجب أن يحل `commands.callbackUrl` إلى نقطة نهاية بوابة OpenClaw وأن يكون قابلًا للوصول من خادم Mattermost.
- تتم مصادقة callbacks الخاصة بأوامر slash الأصلية باستخدام الرموز الخاصة بكل أمر التي يعيدها Mattermost أثناء تسجيل أوامر slash. إذا فشل التسجيل أو لم يتم تفعيل أي أوامر، يرفض OpenClaw الاستدعاءات الراجعة بالرسالة
  `Unauthorized: invalid command token.`
- بالنسبة لمضيفي callbacks الخاصين/الداخليين/على tailnet، قد يتطلب Mattermost
  `ServiceSettings.AllowedUntrustedInternalConnections` أن يتضمن مضيف/نطاق callback.
  استخدم قيم المضيف/النطاق، وليس عناوين URL الكاملة.
- `channels.mattermost.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي يبدؤها Mattermost.
- `channels.mattermost.requireMention`: يشترط `@mention` قبل الرد في القنوات.
- `channels.mattermost.groups.<channelId>.requireMention`: تجاوز تقييد الإشارات لكل قناة (`"*"` للافتراضي).
- يجاوز `channels.mattermost.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّن.

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

**أوضاع إشعارات التفاعلات:** `off`، `own` (افتراضي)، `all`، `allowlist` (من `reactionAllowlist`).

- `channels.signal.account`: يثبت بدء تشغيل القناة على هوية حساب Signal محددة.
- `channels.signal.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي يبدؤها Signal.
- يجاوز `channels.signal.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّن.

### BlueBubbles

BlueBubbles هو مسار iMessage الموصى به (مدعوم بإضافة plugin، ويُضبط تحت `channels.bluebubbles`).

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

- مسارات المفاتيح الأساسية المشمولة هنا: `channels.bluebubbles` و`channels.bluebubbles.dmPolicy`.
- يجاوز `channels.bluebubbles.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّن.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات BlueBubbles بجلسات ACP دائمة. استخدم مقبض BlueBubbles أو سلسلة الهدف (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).
- تم توثيق الإعداد الكامل لقناة BlueBubbles في [BlueBubbles](/ar/channels/bluebubbles).

### iMessage

يشغّل OpenClaw الأمر `imsg rpc` (JSON-RPC عبر stdio). لا يلزم daemon أو منفذ.

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

- يجاوز `channels.imessage.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّن.

- يتطلب Full Disk Access لقاعدة بيانات Messages.
- يفضَّل استخدام أهداف `chat_id:<id>`. استخدم `imsg chats --limit 20` لعرض الدردشات.
- يمكن أن يشير `cliPath` إلى غلاف SSH؛ اضبط `remoteHost` (`host` أو `user@host`) لجلب المرفقات عبر SCP.
- يقيّد `attachmentRoots` و`remoteAttachmentRoots` مسارات المرفقات الواردة (الافتراضي: `/Users/*/Library/Messages/Attachments`).
- يستخدم SCP تحققًا صارمًا من مفتاح المضيف، لذا تأكد من أن مفتاح مضيف relay موجود بالفعل في `~/.ssh/known_hosts`.
- `channels.imessage.configWrites`: السماح أو المنع لعمليات كتابة الإعدادات التي يبدؤها iMessage.
- يمكن لإدخالات `bindings[]` ذات المستوى الأعلى مع `type: "acp"` ربط محادثات iMessage بجلسات ACP دائمة. استخدم مقبضًا مُطبّعًا أو هدف دردشة صريحًا (`chat_id:*` أو `chat_guid:*` أو `chat_identifier:*`) في `match.peer.id`. دلالات الحقول المشتركة: [وكلاء ACP](/ar/tools/acp-agents#channel-specific-settings).

<Accordion title="مثال غلاف SSH لـ iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix مدعوم بإضافة extension ويتم ضبطه تحت `channels.matrix`.

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
- يسمح `channels.matrix.network.dangerouslyAllowPrivateNetwork` بخوادم homeserver الخاصة/الداخلية. إن `proxy` وهذا الاشتراك في الشبكة هما عنصران مستقلان.
- يحدد `channels.matrix.defaultAccount` الحساب المفضل في إعدادات تعدد الحسابات.
- يكون `channels.matrix.autoJoin` افتراضيًا `off`، لذا يتم تجاهل الغرف المدعو إليها ودعوات الرسائل الخاصة الجديدة حتى تضبط `autoJoin: "allowlist"` مع `autoJoinAllowlist` أو `autoJoin: "always"`.
- `channels.matrix.execApprovals`: تسليم موافقات exec الأصلية في Matrix وتفويض الموافقين.
  - `enabled`: `true` أو `false` أو `"auto"` (افتراضي). في الوضع التلقائي، تُفعَّل موافقات exec عندما يمكن حل الموافقين من `approvers` أو `commands.ownerAllowFrom`.
  - `approvers`: معرّفات مستخدمي Matrix (مثل `@owner:example.org`) المسموح لهم بالموافقة على طلبات exec.
  - `agentFilter`: قائمة سماح اختيارية لمعرّفات الوكلاء. احذفها لتمرير الموافقات لجميع الوكلاء.
  - `sessionFilter`: أنماط اختيارية لمفاتيح الجلسات (substring أو regex).
  - `target`: مكان إرسال مطالبات الموافقة. `"dm"` (افتراضي)، أو `"channel"` (الغرفة الأصلية)، أو `"both"`.
  - تجاوزات لكل حساب: `channels.matrix.accounts.<id>.execApprovals`.
- يتحكم `channels.matrix.dm.sessionScope` في كيفية تجميع رسائل Matrix الخاصة داخل الجلسات: `per-user` (افتراضي) يشارك حسب النظير الموجَّه، بينما `per-room` يعزل كل غرفة DM.
- تستخدم فحوصات حالة Matrix وعمليات البحث المباشر في الدليل نفس سياسة الوكيل المستخدمة لحركة وقت التشغيل.
- تم توثيق إعدادات Matrix الكاملة وقواعد الاستهداف وأمثلة الإعداد في [Matrix](/ar/channels/matrix).

### Microsoft Teams

Microsoft Teams مدعوم بإضافة extension ويتم ضبطه تحت `channels.msteams`.

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

- مسارات المفاتيح الأساسية المشمولة هنا: `channels.msteams` و`channels.msteams.configWrites`.
- تم توثيق إعدادات Teams الكاملة (الاعتمادات، webhook، سياسة الرسائل الخاصة/المجموعات، وتجاوزات كل فريق/قناة) في [Microsoft Teams](/ar/channels/msteams).

### IRC

IRC مدعوم بإضافة extension ويتم ضبطه تحت `channels.irc`.

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

- مسارات المفاتيح الأساسية المشمولة هنا: `channels.irc` و`channels.irc.dmPolicy` و`channels.irc.configWrites` و`channels.irc.nickserv.*`.
- يجاوز `channels.irc.defaultAccount` الاختياري اختيار الحساب الافتراضي عندما يطابق معرّف حساب مكوَّن.
- تم توثيق إعداد قناة IRC الكامل (المضيف/المنفذ/TLS/القنوات/قوائم السماح/تقييد الإشارات) في [IRC](/ar/channels/irc).

### تعدد الحسابات (كل القنوات)

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

- يُستخدم `default` عند حذف `accountId` (في CLI + التوجيه).
- لا تنطبق رموز البيئة إلا على الحساب **الافتراضي**.
- تنطبق إعدادات القناة الأساسية على جميع الحسابات ما لم يتم تجاوزها لكل حساب.
- استخدم `bindings[].match.accountId` لتوجيه كل حساب إلى وكيل مختلف.
- إذا أضفت حسابًا غير افتراضي عبر `openclaw channels add` (أو إعداد القناة) بينما ما زلت على إعداد قناة أحادي الحساب في المستوى الأعلى، يقوم OpenClaw أولاً بترقية القيم أحادية الحساب على المستوى الأعلى ذات النطاق الحسابي إلى خريطة حسابات القناة حتى يستمر الحساب الأصلي في العمل. تنقل معظم القنوات هذه القيم إلى `channels.<channel>.accounts.default`؛ ويمكن لـ Matrix بدلًا من ذلك الحفاظ على هدف موجود مطابق باسم/default.
- تستمر الروابط الحالية الخاصة بالقناة فقط (من دون `accountId`) في مطابقة الحساب الافتراضي؛ وتظل الروابط محددة النطاق بالحساب اختيارية.
- يقوم `openclaw doctor --fix` أيضًا بإصلاح الأشكال المختلطة بنقل القيم أحادية الحساب على المستوى الأعلى ذات النطاق الحسابي إلى الحساب المُرقّى المختار لتلك القناة. تستخدم معظم القنوات `accounts.default`؛ ويمكن لـ Matrix الحفاظ على هدف موجود مطابق باسم/default بدلًا من ذلك.

### قنوات extension أخرى

تُضبط العديد من قنوات extension كـ `channels.<id>` ويتم توثيقها في صفحات القنوات المخصصة لها (مثل Feishu وMatrix وLINE وNostr وZalo وNextcloud Talk وSynology Chat وTwitch).
راجع فهرس القنوات الكامل: [القنوات](/ar/channels).

### تقييد الإشارات في الدردشة الجماعية

تفترض رسائل المجموعات افتراضيًا **اشتراط الإشارة** (إشارة ضمنية في البيانات الوصفية أو أنماط regex آمنة). ينطبق هذا على مجموعات WhatsApp وTelegram وDiscord وGoogle Chat وiMessage.

**أنواع الإشارات:**

- **إشارات البيانات الوصفية**: إشارات @ الأصلية للمنصة. يتم تجاهلها في وضع الدردشة الذاتية لـ WhatsApp.
- **أنماط النص**: أنماط regex آمنة في `agents.list[].groupChat.mentionPatterns`. يتم تجاهل الأنماط غير الصالحة والتكرار المتداخل غير الآمن.
- يتم فرض تقييد الإشارات فقط عندما يكون الاكتشاف ممكنًا (إشارات أصلية أو نمط واحد على الأقل).

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

يضبط `messages.groupChat.historyLimit` القيمة الافتراضية العامة. ويمكن للقنوات تجاوزها عبر `channels.<channel>.historyLimit` (أو لكل حساب). اضبطها إلى `0` للتعطيل.

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

آلية الحل: تجاوز لكل DM → افتراضي المزوّد → لا حد (الاحتفاظ بكل شيء).

المدعوم: `telegram`، `whatsapp`، `discord`، `slack`، `signal`، `imessage`، `msteams`.

#### وضع الدردشة الذاتية

أدرج رقمك الخاص في `allowFrom` لتفعيل وضع الدردشة الذاتية (يتجاهل إشارات @ الأصلية ويرد فقط على الأنماط النصية):

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
- `native: "auto"` يفعّل الأوامر الأصلية لـ Discord/Telegram ويُبقي Slack معطلاً.
- تجاوز لكل قناة: `channels.discord.commands.native` (قيمة منطقية أو `"auto"`). تؤدي `false` إلى مسح الأوامر المسجلة سابقًا.
- يضيف `channels.telegram.customCommands` إدخالات إضافية إلى قائمة بوت Telegram.
- يفعّل `bash: true` الأمر `! <cmd>` لصدفة المضيف. يتطلب `tools.elevated.enabled` وأن يكون المرسل ضمن `tools.elevated.allowFrom.<channel>`.
- يفعّل `config: true` الأمر `/config` (قراءة/كتابة `openclaw.json`). بالنسبة لعملاء `chat.send` في البوابة، تتطلب الكتابات الدائمة عبر `/config set|unset` أيضًا `operator.admin`؛ بينما يبقى `/config show` للقراءة فقط متاحًا للعملاء العاديين ذوي نطاق الكتابة.
- يقيّد `channels.<provider>.configWrites` عمليات تعديل الإعدادات لكل قناة (الافتراضي: true).
- بالنسبة للقنوات متعددة الحسابات، يقيّد `channels.<provider>.accounts.<id>.configWrites` أيضًا عمليات الكتابة التي تستهدف ذلك الحساب (مثل `/allowlist --config --account <id>` أو `/config set channels.<provider>.accounts.<id>...`).
- يكون `allowFrom` لكل مزوّد. عند تعيينه، يصبح هو **مصدر التفويض الوحيد** (ويتم تجاهل قوائم السماح/الإقران الخاصة بالقناة و`useAccessGroups`).
- يسمح `useAccessGroups: false` للأوامر بتجاوز سياسات مجموعات الوصول عندما لا يكون `allowFrom` معيّنًا.

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

جذر مستودع اختياري يُعرض في سطر Runtime ضمن system prompt. إذا لم يُعيَّن، يكتشفه OpenClaw تلقائيًا عبر الصعود من مساحة العمل.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

قائمة سماح افتراضية اختيارية لـ Skills للوكلاء الذين لا يعيّنون
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

- احذف `agents.defaults.skills` للحصول على Skills غير مقيّدة افتراضيًا.
- احذف `agents.list[].skills` لوراثة القيم الافتراضية.
- عيّن `agents.list[].skills: []` لعدم وجود أي Skills.
- قائمة `agents.list[].skills` غير الفارغة هي المجموعة النهائية لذلك الوكيل؛
  ولا يتم دمجها مع القيم الافتراضية.

### `agents.defaults.skipBootstrap`

يعطّل الإنشاء التلقائي لملفات bootstrap الخاصة بمساحة العمل (`AGENTS.md`، `SOUL.md`، `TOOLS.md`، `IDENTITY.md`، `USER.md`، `HEARTBEAT.md`، `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

يتحكم في وقت حقن ملفات bootstrap الخاصة بمساحة العمل في system prompt. الافتراضي: `"always"`.

- `"continuation-skip"`: في أدوار المتابعة الآمنة (بعد اكتمال رد المساعد) يتم تخطي إعادة حقن bootstrap الخاص بمساحة العمل، مما يقلل حجم prompt. تظل تشغيلات heartbeat وإعادات المحاولة بعد الضغط تعيد بناء السياق.

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

الحد الأقصى لإجمالي الأحرف المحقونة عبر جميع ملفات bootstrap الخاصة بمساحة العمل. الافتراضي: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

يتحكم في نص التحذير الظاهر للوكيل عندما يتم اقتطاع سياق bootstrap.
الافتراضي: `"once"`.

- `"off"`: لا يحقن نص تحذير في system prompt أبدًا.
- `"once"`: يحقن التحذير مرة واحدة لكل توقيع اقتطاع فريد (موصى به).
- `"always"`: يحقن التحذير في كل تشغيل عندما يوجد اقتطاع.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

أقصى حجم بكسلي لأطول ضلع في الصورة داخل كتل صور transcript/tool قبل استدعاءات المزوّد.
الافتراضي: `1200`.

القيم الأقل تقلل عادةً من استخدام vision-token وحجم حمولة الطلب في التشغيلات الثقيلة باللقطات.
والقيم الأعلى تحافظ على مزيد من التفاصيل البصرية.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

المنطقة الزمنية لسياق system prompt (وليس الطوابع الزمنية للرسائل). تعود إلى المنطقة الزمنية للمضيف.

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

- `model`: يقبل إما سلسلة نصية (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - صيغة السلسلة تضبط النموذج الأساسي فقط.
  - صيغة الكائن تضبط النموذج الأساسي بالإضافة إلى نماذج الفشل الاحتياطي بالترتيب.
- `imageModel`: يقبل إما سلسلة نصية (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة مسار أداة `image` بوصفه إعداد نموذج الرؤية.
  - ويُستخدم أيضًا كتوجيه احتياطي عندما لا يستطيع النموذج المحدد/الافتراضي قبول إدخال الصور.
- `imageGenerationModel`: يقبل إما سلسلة نصية (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة قدرة توليد الصور المشتركة وأي سطح أداة/إضافة plugin مستقبلي يولد صورًا.
  - القيم المعتادة: `google/gemini-3.1-flash-image-preview` لتوليد صور Gemini الأصلي، أو `fal/fal-ai/flux/dev` لـ fal، أو `openai/gpt-image-1` لصور OpenAI.
  - إذا اخترت مزودًا/نموذجًا مباشرة، فقم أيضًا بضبط مصادقة/مفتاح API للمزوّد المطابق (مثل `GEMINI_API_KEY` أو `GOOGLE_API_KEY` لـ `google/*`، أو `OPENAI_API_KEY` لـ `openai/*`، أو `FAL_KEY` لـ `fal/*`).
  - إذا تم حذفه، فلا يزال بإمكان `image_generate` استنتاج مزود افتراضي مدعوم بالمصادقة. يحاول أولاً المزوّد الافتراضي الحالي، ثم مزوّدي توليد الصور الآخرين المسجلين حسب ترتيب معرّف المزوّد.
- `musicGenerationModel`: يقبل إما سلسلة نصية (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة قدرة توليد الموسيقى المشتركة وأداة `music_generate` المضمّنة.
  - القيم المعتادة: `google/lyria-3-clip-preview` أو `google/lyria-3-pro-preview` أو `minimax/music-2.5+`.
  - إذا تم حذفه، فلا يزال بإمكان `music_generate` استنتاج مزود افتراضي مدعوم بالمصادقة. يحاول أولاً المزوّد الافتراضي الحالي، ثم مزوّدي توليد الموسيقى الآخرين المسجلين حسب ترتيب معرّف المزوّد.
  - إذا اخترت مزودًا/نموذجًا مباشرة، فقم أيضًا بضبط مصادقة/مفتاح API للمزوّد المطابق.
- `videoGenerationModel`: يقبل إما سلسلة نصية (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة قدرة توليد الفيديو المشتركة وأداة `video_generate` المضمّنة.
  - القيم المعتادة: `qwen/wan2.6-t2v` أو `qwen/wan2.6-i2v` أو `qwen/wan2.6-r2v` أو `qwen/wan2.6-r2v-flash` أو `qwen/wan2.7-r2v`.
  - إذا تم حذفه، فلا يزال بإمكان `video_generate` استنتاج مزود افتراضي مدعوم بالمصادقة. يحاول أولاً المزوّد الافتراضي الحالي، ثم مزوّدي توليد الفيديو الآخرين المسجلين حسب ترتيب معرّف المزوّد.
  - إذا اخترت مزودًا/نموذجًا مباشرة، فقم أيضًا بضبط مصادقة/مفتاح API للمزوّد المطابق.
  - يدعم مزوّد Qwen المضمّن لتوليد الفيديو حاليًا حتى 1 فيديو خرج، و1 صورة إدخال، و4 فيديوهات إدخال، ومدة 10 ثوانٍ، وخيارات على مستوى المزوّد مثل `size` و`aspectRatio` و`resolution` و`audio` و`watermark`.
- `pdfModel`: يقبل إما سلسلة نصية (`"provider/model"`) أو كائنًا (`{ primary, fallbacks }`).
  - يُستخدم بواسطة أداة `pdf` لتوجيه النموذج.
  - إذا تم حذفه، تعود أداة PDF إلى `imageModel` ثم إلى النموذج المحلول للجلسة/الافتراضي.
- `pdfMaxBytesMb`: حد حجم PDF الافتراضي لأداة `pdf` عندما لا يتم تمرير `maxBytesMb` وقت الاستدعاء.
- `pdfMaxPages`: الحد الأقصى الافتراضي للصفحات التي تؤخذ بعين الاعتبار في وضع الاسترجاع الاحتياطي داخل أداة `pdf`.
- `verboseDefault`: مستوى verbose الافتراضي للوكلاء. القيم: `"off"`، `"on"`، `"full"`. الافتراضي: `"off"`.
- `elevatedDefault`: مستوى المخرجات elevated الافتراضي للوكلاء. القيم: `"off"`، `"on"`، `"ask"`، `"full"`. الافتراضي: `"on"`.
- `model.primary`: التنسيق `provider/model` (مثلاً `openai/gpt-5.4`). إذا حذفت المزوّد، يحاول OpenClaw أولاً اسمًا مستعارًا، ثم مطابقة مزوّد مكوَّن فريدة لذلك معرّف النموذج الدقيق، وبعدها فقط يعود إلى المزوّد الافتراضي المكوَّن (سلوك توافق قديم، لذا يُفضّل استخدام `provider/model` صريحًا). إذا لم يعد ذلك المزوّد يوفّر النموذج الافتراضي المكوَّن، يعود OpenClaw إلى أول مزوّد/نموذج مكوَّن بدلًا من إظهار افتراضي قديم لمزوّد تمت إزالته.
- `models`: فهرس النماذج المكوَّن وقائمة السماح الخاصة بـ `/model`. يمكن أن يتضمن كل إدخال `alias` (اختصار) و`params` (خاصة بالمزوّد، مثل `temperature` أو `maxTokens` أو `cacheRetention` أو `context1m`).
- `params`: معلمات المزوّد الافتراضية العامة المطبقة على جميع النماذج. يتم ضبطها في `agents.defaults.params` (مثل `{ cacheRetention: "long" }`).
- أولوية دمج `params` (في الإعدادات): يتم تجاوز `agents.defaults.params` (الأساس العام) بواسطة `agents.defaults.models["provider/model"].params` (لكل نموذج)، ثم تتجاوز `agents.list[].params` (لـ agent id المطابق) حسب المفتاح. راجع [Prompt Caching](/ar/reference/prompt-caching) للتفاصيل.
- تحفظ كاتبات الإعدادات التي تعدل هذه الحقول (مثل `/models set` و`/models set-image` وأوامر إضافة/إزالة fallback) صيغة الكائن القياسية وتحافظ على قوائم fallback الحالية متى أمكن.
- `maxConcurrent`: الحد الأقصى للتشغيلات المتوازية للوكلاء عبر الجلسات (مع بقاء كل جلسة متسلسلة). الافتراضي: 4.

**اختصارات الأسماء المستعارة المضمّنة** (تنطبق فقط عندما يكون النموذج موجودًا في `agents.defaults.models`):

| الاسم المستعار      | النموذج                                 |
| ------------------- | --------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`             |
| `sonnet`            | `anthropic/claude-sonnet-4-6`           |
| `gpt`               | `openai/gpt-5.4`                        |
| `gpt-mini`          | `openai/gpt-5.4-mini`                   |
| `gpt-nano`          | `openai/gpt-5.4-nano`                   |
| `gemini`            | `google/gemini-3.1-pro-preview`         |
| `gemini-flash`      | `google/gemini-3-flash-preview`         |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview`  |

تتغلب الأسماء المستعارة التي تضبطها أنت دائمًا على القيم الافتراضية.

تفعّل نماذج Z.AI GLM-4.x وضع التفكير تلقائيًا ما لم تضبط `--thinking off` أو تعرّف بنفسك `agents.defaults.models["zai/<model>"].params.thinking`.
وتفعّل نماذج Z.AI `tool_stream` افتراضيًا لبث استدعاءات الأدوات. اضبط `agents.defaults.models["zai/<model>"].params.tool_stream` إلى `false` لتعطيله.
وتستخدم نماذج Anthropic Claude 4.6 افتراضيًا التفكير `adaptive` عندما لا يكون مستوى التفكير معيّنًا صراحة.

### `agents.defaults.cliBackends`

خلفيات CLI اختيارية لتشغيلات الاحتياط النصية فقط (من دون استدعاءات أدوات). مفيدة كنسخة احتياطية عندما تفشل مزوّدو API.

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

- خلفيات CLI موجهة للنص أولاً؛ يتم دائمًا تعطيل الأدوات.
- الجلسات مدعومة عندما يكون `sessionArg` معيّنًا.
- تمرير الصور مدعوم عندما يقبل `imageArg` مسارات الملفات.

### `agents.defaults.heartbeat`

تشغيلات heartbeat دورية.

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

- `every`: سلسلة مدة (ms/s/m/h). الافتراضي: `30m` (مصادقة API-key) أو `1h` (مصادقة OAuth). اضبطها إلى `0m` للتعطيل.
- `suppressToolErrorWarnings`: عندما تكون true، تُخفي حمولات تحذير أخطاء الأدوات أثناء تشغيلات heartbeat.
- `directPolicy`: سياسة التسليم المباشر/الخاص. `allow` (افتراضي) يسمح بالتسليم المباشر المستهدف. `block` يمنع التسليم المباشر المستهدف ويصدر `reason=dm-blocked`.
- `lightContext`: عندما تكون true، تستخدم تشغيلات heartbeat سياق bootstrap خفيف الوزن وتحتفظ فقط بـ `HEARTBEAT.md` من ملفات bootstrap الخاصة بمساحة العمل.
- `isolatedSession`: عندما تكون true، يعمل كل heartbeat في جلسة جديدة من دون سجل محادثة سابق. نفس نمط العزل المستخدم في cron مع `sessionTarget: "isolated"`. يقلل كلفة الرموز لكل heartbeat من حوالي ~100K إلى ~2-5K token.
- لكل وكيل: عيّن `agents.list[].heartbeat`. عندما يعرّف أي وكيل `heartbeat`، **تعمل heartbeat فقط لهؤلاء الوكلاء**.
- تشغّل heartbeat أدوار الوكيل كاملة — الفترات الأقصر تستهلك مزيدًا من الرموز.

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

- `mode`: `default` أو `safeguard` (تلخيص مقسم لسجلات طويلة). راجع [الضغط](/ar/concepts/compaction).
- `provider`: معرّف إضافة plugin مسجلة لمزوّد compaction. عند تعيينه، يتم استدعاء `summarize()` الخاص بالمزوّد بدلًا من التلخيص المضمّن المعتمد على LLM. ويعود إلى المضمن عند الفشل. يؤدي تعيين مزوّد إلى فرض `mode: "safeguard"`. راجع [الضغط](/ar/concepts/compaction).
- `timeoutSeconds`: الحد الأقصى بالثواني المسموح به لعملية compaction واحدة قبل أن يوقفها OpenClaw. الافتراضي: `900`.
- `identifierPolicy`: `strict` (افتراضي) أو `off` أو `custom`. تضيف `strict` إرشادات مضمّنة للحفاظ على المعرفات المعتمة قبل تلخيص compaction.
- `identifierInstructions`: نص مخصص اختياري للحفاظ على المعرفات يُستخدم عندما يكون `identifierPolicy=custom`.
- `postCompactionSections`: أسماء أقسام H2/H3 اختيارية من AGENTS.md لإعادة حقنها بعد compaction. افتراضيًا `["Session Startup", "Red Lines"]`؛ اضبطها إلى `[]` لتعطيل إعادة الحقن. عندما تكون غير معيّنة أو معيّنة صراحةً إلى هذا الزوج الافتراضي، تُقبل أيضًا العناوين القديمة `Every Session`/`Safety` كحل احتياطي قديم.
- `model`: تجاوز اختياري `provider/model-id` لعمليات تلخيص compaction فقط. استخدمه عندما ينبغي أن تبقى الجلسة الرئيسية على نموذج معيّن لكن تعمل ملخصات compaction على نموذج آخر؛ وعند حذفه، يستخدم compaction النموذج الأساسي للجلسة.
- `notifyUser`: عندما تكون `true`، يرسل إشعارًا مختصرًا للمستخدم عند بدء compaction (مثل "Compacting context..."). يكون معطلًا افتراضيًا لإبقاء compaction صامتًا.
- `memoryFlush`: دور وكيل صامت قبل compaction التلقائي لتخزين الذكريات الدائمة. يتم تجاوزه عندما تكون مساحة العمل للقراءة فقط.

### `agents.defaults.contextPruning`

يقتطع **نتائج الأدوات القديمة** من السياق داخل الذاكرة قبل إرسالها إلى LLM. ولا يعدّل سجل الجلسة المخزّن على القرص.

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
- يتحكم `ttl` في عدد المرات التي يمكن أن يعمل فيها الاقتطاع مجددًا (بعد آخر لمسة للذاكرة المؤقتة).
- يقوم الاقتطاع أولًا باقتطاعٍ ناعم لنتائج الأدوات كبيرة الحجم، ثم يزيل النتائج الأقدم إزالةً كاملة إذا لزم الأمر.

**الاقتطاع الناعم** يحتفظ بالبداية + النهاية ويُدرج `...` في الوسط.

**الإزالة الكاملة** تستبدل نتيجة الأداة بأكملها بالعنصر النائب.

ملاحظات:

- لا يتم اقتطاع/إزالة كتل الصور أبدًا.
- النسب مبنية على الأحرف (تقريبية)، وليست على عدد الرموز بدقة.
- إذا وُجد عدد أقل من `keepLastAssistants` من رسائل المساعد، يتم تجاوز الاقتطاع.

</Accordion>

راجع [اقتطاع الجلسة](/ar/concepts/session-pruning) لتفاصيل السلوك.

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

- تتطلب القنوات غير Telegram تفعيل `*.blockStreaming: true` صراحةً لتمكين الردود الكتلية.
- تجاوزات القنوات: `channels.<channel>.blockStreamingCoalesce` (ومتغيراتها لكل حساب). تفترض Signal/Slack/Discord/Google Chat افتراضيًا `minChars: 1500`.
- `humanDelay`: توقف عشوائي بين الردود الكتلية. `natural` = 800–2500ms. تجاوز لكل وكيل: `agents.list[].humanDelay`.

راجع [البث](/ar/concepts/streaming) للحصول على السلوك + تفاصيل تقسيم الكتل.

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

- القيم الافتراضية: `instant` للدردشات المباشرة/الإشارات، و`message` لمجموعات الدردشة غير المشار فيها.
- تجاوزات لكل جلسة: `session.typingMode`، `session.typingIntervalSeconds`.

راجع [مؤشرات الكتابة](/ar/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

الحماية الاختيارية داخل sandbox للوكيل المضمّن. راجع [الحماية داخل sandbox](/ar/gateway/sandboxing) للحصول على الدليل الكامل.

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

**الخلفية:**

- `docker`: وقت تشغيل Docker محلي (افتراضي)
- `ssh`: وقت تشغيل بعيد عام معتمد على SSH
- `openshell`: وقت تشغيل OpenShell

عند اختيار `backend: "openshell"`، تنتقل الإعدادات الخاصة بوقت التشغيل إلى
`plugins.entries.openshell.config`.

**إعدادات خلفية SSH:**

- `target`: هدف SSH بصيغة `user@host[:port]`
- `command`: أمر عميل SSH (الافتراضي: `ssh`)
- `workspaceRoot`: الجذر البعيد المطلق المستخدم لمساحات العمل حسب النطاق
- `identityFile` / `certificateFile` / `knownHostsFile`: ملفات محلية موجودة يتم تمريرها إلى OpenSSH
- `identityData` / `certificateData` / `knownHostsData`: محتويات مضمّنة أو SecretRefs يقوم OpenClaw بتحويلها إلى ملفات مؤقتة وقت التشغيل
- `strictHostKeyChecking` / `updateHostKeys`: مفاتيح سياسة مفاتيح المضيف في OpenSSH

**أولوية مصادقة SSH:**

- `identityData` يتغلب على `identityFile`
- `certificateData` يتغلب على `certificateFile`
- `knownHostsData` يتغلب على `knownHostsFile`
- يتم حل قيم `*Data` المدعومة بـ SecretRef من اللقطة النشطة لوقت تشغيل الأسرار قبل بدء جلسة sandbox

**سلوك خلفية SSH:**

- يزرع مساحة العمل البعيدة مرة واحدة بعد الإنشاء أو إعادة الإنشاء
- ثم يُبقي مساحة عمل SSH البعيدة هي المرجع الأساسي
- يمرّر `exec` وأدوات الملفات ومسارات الوسائط عبر SSH
- لا يزامن التغييرات البعيدة تلقائيًا إلى المضيف
- لا يدعم حاويات المتصفح داخل sandbox

**وصول مساحة العمل:**

- `none`: مساحة عمل sandbox لكل نطاق تحت `~/.openclaw/sandboxes`
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

- `mirror`: زرع البعيد من المحلي قبل exec، ثم المزامنة العكسية بعد exec؛ تبقى مساحة العمل المحلية هي المرجع الأساسي
- `remote`: زرع البعيد مرة واحدة عند إنشاء sandbox، ثم إبقاء مساحة العمل البعيدة هي المرجع الأساسي

في وضع `remote`، لا تتم مزامنة التعديلات المحلية على المضيف التي تتم خارج OpenClaw إلى sandbox تلقائيًا بعد خطوة الزرع.
يتم النقل عبر SSH إلى sandbox الخاص بـ OpenShell، لكن الإضافة تملك دورة حياة sandbox والمزامنة المرآتية الاختيارية.

**`setupCommand`** يعمل مرة واحدة بعد إنشاء الحاوية (عبر `sh -lc`). ويتطلب خروجًا للشبكة، وجذرًا قابلاً للكتابة، ومستخدم root.

**تكون الحاويات افتراضيًا على `network: "none"`** — اضبطها على `"bridge"` (أو شبكة bridge مخصصة) إذا كان الوكيل يحتاج إلى وصول صادر.
يتم حظر `"host"`. ويتم حظر `"container:<id>"` افتراضيًا ما لم تعيّن صراحة
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` (كسر زجاج).

**المرفقات الواردة** يتم تجهيزها داخل `media/inbound/*` في مساحة العمل النشطة.

**`docker.binds`** يركّب أدلة مضيف إضافية؛ ويتم دمج التركيبات العامة والتركيبات لكل وكيل.

**المتصفح داخل sandbox** (`sandbox.browser.enabled`): Chromium + CDP في حاوية. يتم حقن عنوان noVNC في system prompt. ولا يتطلب `browser.enabled` في `openclaw.json`.
يستخدم وصول المراقب إلى noVNC مصادقة VNC افتراضيًا ويصدر OpenClaw رابط token قصير العمر (بدلاً من كشف كلمة المرور في الرابط المشترك).

- `allowHostControl: false` (افتراضي) يمنع الجلسات الموجودة داخل sandbox من استهداف متصفح المضيف.
- تكون `network` افتراضيًا `openclaw-sandbox-browser` (شبكة bridge مخصصة). اضبطها على `bridge` فقط عندما تريد صراحةً اتصال bridge عالمي.
- يقيّد `cdpSourceRange` اختياريًا دخول CDP على حافة الحاوية إلى نطاق CIDR (مثل `172.21.0.1/32`).
- يركّب `sandbox.browser.binds` أدلة مضيف إضافية داخل حاوية متصفح sandbox فقط. عند تعيينه (بما في ذلك `[]`)، فإنه يستبدل `docker.binds` لحاوية المتصفح.
- تُعرّف الإعدادات الافتراضية للتشغيل في `scripts/sandbox-browser-entrypoint.sh` وتم ضبطها لمضيفي الحاويات:
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
  - تكون `--disable-3d-apis` و`--disable-software-rasterizer` و`--disable-gpu`
    مفعّلة افتراضيًا ويمكن تعطيلها عبر
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` إذا كان استخدام WebGL/3D يتطلب ذلك.
  - يعيد `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` تفعيل الإضافات إذا كان سير عملك يعتمد عليها.
  - يمكن تغيير `--renderer-process-limit=2` عبر
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`؛ واضبطه إلى `0` لاستخدام
    حد العمليات الافتراضي لـ Chromium.
  - بالإضافة إلى `--no-sandbox` و`--disable-setuid-sandbox` عندما يكون `noSandbox` مفعلاً.
  - هذه الإعدادات الافتراضية هي خط الأساس لصورة الحاوية؛ استخدم صورة متصفح مخصصة مع نقطة
    دخول مخصصة لتغيير افتراضيات الحاوية.

</Accordion>

حماية المتصفح داخل sandbox و`sandbox.docker.binds` مدعومتان حاليًا فقط مع Docker.

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
- `default`: عند تعيين عدة وكلاء، يفوز الأول (مع تسجيل تحذير). وإذا لم يُعيَّن أي منها، يكون أول إدخال في القائمة هو الافتراضي.
- `model`: صيغة السلسلة تتجاوز `primary` فقط؛ وصيغة الكائن `{ primary, fallbacks }` تتجاوز كليهما (`[]` تعطل fallback العام). تبقى مهام cron التي تتجاوز `primary` فقط ترث fallback الافتراضي ما لم تعيّن `fallbacks: []`.
- `params`: معلمات بث لكل وكيل تُدمج فوق إدخال النموذج المحدد في `agents.defaults.models`. استخدمها لتجاوزات خاصة بالوكيل مثل `cacheRetention` أو `temperature` أو `maxTokens` دون تكرار فهرس النماذج بالكامل.
- `skills`: قائمة سماح اختيارية لـ Skills لكل وكيل. إذا حُذفت، يرث الوكيل `agents.defaults.skills` عند تعيينها؛ وتستبدل القائمة الصريحة القيم الافتراضية بدلًا من دمجها، وتعني `[]` عدم وجود Skills.
- `thinkingDefault`: مستوى التفكير الافتراضي الاختياري لكل وكيل (`off | minimal | low | medium | high | xhigh | adaptive`). يتجاوز `agents.defaults.thinkingDefault` لهذا الوكيل عندما لا يكون هناك تجاوز لكل رسالة أو جلسة.
- `reasoningDefault`: تجاوز اختياري لرؤية الاستدلال الافتراضية لكل وكيل (`on | off | stream`). يُطبّق عندما لا يكون هناك تجاوز للاستدلال لكل رسالة أو جلسة.
- `fastModeDefault`: افتراضي اختياري لكل وكيل لوضع fast (`true | false`). يُطبّق عندما لا يكون هناك تجاوز لوضع fast لكل رسالة أو جلسة.
- `runtime`: واصف وقت تشغيل اختياري لكل وكيل. استخدم `type: "acp"` مع الإعدادات الافتراضية في `runtime.acp` (`agent`، `backend`، `mode`، `cwd`) عندما ينبغي أن يستخدم الوكيل افتراضيًا جلسات ACP harness.
- `identity.avatar`: مسار نسبي لمساحة العمل، أو URL من نوع `http(s)`، أو `data:` URI.
- يستنتج `identity` القيم الافتراضية: `ackReaction` من `emoji`، و`mentionPatterns` من `name`/`emoji`.
- `subagents.allowAgents`: قائمة سماح لمعرّفات الوكلاء في `sessions_spawn` (`["*"]` = أي وكيل؛ الافتراضي: نفس الوكيل فقط).
- حارس وراثة sandbox: إذا كانت جلسة الطالب داخل sandbox، يرفض `sessions_spawn` الأهداف التي ستعمل خارج sandbox.
- `subagents.requireAgentId`: عندما تكون true، يحظر استدعاءات `sessions_spawn` التي تحذف `agentId` (يفرض اختيار ملف تعريف صريح؛ الافتراضي: false).

---

## التوجيه متعدد الوكلاء

شغّل عدة وكلاء معزولين داخل بوابة واحدة. راجع [Multi-Agent](/ar/concepts/multi-agent).

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

- `type` (اختياري): `route` للتوجيه العادي (والنوع المفقود يفترض route)، و`acp` لروابط محادثة ACP الدائمة.
- `match.channel` (مطلوب)
- `match.accountId` (اختياري؛ `*` = أي حساب؛ المحذوف = الحساب الافتراضي)
- `match.peer` (اختياري؛ `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (اختياري؛ خاصة بقناة معينة)
- `acp` (اختياري؛ فقط لـ `type: "acp"`): `{ mode, label, cwd, backend }`

**ترتيب المطابقة الحتمي:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (مطابقة تامة، بلا peer/guild/team)
5. `match.accountId: "*"` (على مستوى القناة)
6. الوكيل الافتراضي

ضمن كل مستوى، يفوز أول إدخال مطابق في `bindings`.

بالنسبة لإدخالات `type: "acp"`، يحل OpenClaw حسب هوية المحادثة الدقيقة (`match.channel` + الحساب + `match.peer.id`) ولا يستخدم ترتيب مستويات route أعلاه.

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

راجع [Multi-Agent Sandbox & Tools](/ar/tools/multi-agent-sandbox-tools) للحصول على تفاصيل الأولوية.

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

- **`scope`**: استراتيجية تجميع الجلسات الأساسية لسياقات الدردشة الجماعية.
  - `per-sender` (افتراضي): يحصل كل مرسل على جلسة معزولة داخل سياق القناة.
  - `global`: يشارك جميع المشاركين في سياق القناة جلسة واحدة (استخدمه فقط عندما يكون السياق المشترك مقصودًا).
- **`dmScope`**: كيفية تجميع الرسائل الخاصة.
  - `main`: تشترك جميع الرسائل الخاصة في الجلسة الرئيسية.
  - `per-peer`: يعزل حسب معرّف المرسل عبر القنوات.
  - `per-channel-peer`: يعزل لكل قناة + مرسل (موصى به لصناديق الوارد متعددة المستخدمين).
  - `per-account-channel-peer`: يعزل لكل حساب + قناة + مرسل (موصى به لتعدد الحسابات).
- **`identityLinks`**: يربط المعرّفات القياسية بالنظراء مسبوقي المزوّد لمشاركة الجلسات عبر القنوات.
- **`reset`**: سياسة إعادة التعيين الأساسية. يعيد `daily` التعيين عند `atHour` بالتوقيت المحلي؛ ويعيد `idle` التعيين بعد `idleMinutes`. عند ضبط الاثنين، يفوز أول ما تنتهي صلاحيته.
- **`resetByType`**: تجاوزات حسب النوع (`direct` و`group` و`thread`). ويُقبل `dm` القديم كاسم مستعار لـ `direct`.
- **`parentForkMaxTokens`**: الحد الأقصى لـ `totalTokens` في الجلسة الأصلية المسموح به عند إنشاء جلسة خيط متفرعة (الافتراضي `100000`).
  - إذا كانت قيمة `totalTokens` في الجلسة الأصلية أعلى من هذه القيمة، يبدأ OpenClaw جلسة خيط جديدة بدلًا من وراثة سجل الجلسة الأصلية.
  - اضبطه إلى `0` لتعطيل هذا الحارس والسماح دائمًا بالتفرع من الأصل.
- **`mainKey`**: حقل قديم. يستخدم وقت التشغيل الآن دائمًا `"main"` لحاوية الدردشة المباشرة الرئيسية.
- **`agentToAgent.maxPingPongTurns`**: الحد الأقصى لأدوار الرد المتبادل بين الوكلاء أثناء تبادلات وكيل-إلى-وكيل (عدد صحيح، النطاق: `0`–`5`). تؤدي `0` إلى تعطيل سلسلة ping-pong.
- **`sendPolicy`**: المطابقة حسب `channel` أو `chatType` (`direct|group|channel`، مع الاسم المستعار القديم `dm`) أو `keyPrefix` أو `rawKeyPrefix`. يفوز أول منع.
- **`maintenance`**: تحكمات تنظيف مخزن الجلسات + الاحتفاظ.
  - `mode`: `warn` يصدر تحذيرات فقط؛ و`enforce` يطبق التنظيف.
  - `pruneAfter`: حد عمر لإزالة الإدخالات القديمة (الافتراضي `30d`).
  - `maxEntries`: الحد الأقصى لعدد الإدخالات في `sessions.json` (الافتراضي `500`).
  - `rotateBytes`: تدوير `sessions.json` عندما يتجاوز هذا الحجم (الافتراضي `10mb`).
  - `resetArchiveRetention`: مدة الاحتفاظ بأرشيفات transcript من نوع `*.reset.<timestamp>`. افتراضيًا تساوي `pruneAfter`؛ واضبطها إلى `false` لتعطيلها.
  - `maxDiskBytes`: ميزانية قرص اختيارية لدليل الجلسات. في وضع `warn` تسجل تحذيرات؛ وفي وضع `enforce` تزيل أقدم العناصر/الجلسات أولًا.
  - `highWaterBytes`: الهدف الاختياري بعد تنظيف الميزانية. افتراضيًا `80%` من `maxDiskBytes`.
- **`threadBindings`**: القيم الافتراضية العامة لميزات الجلسات المرتبطة بالخيوط.
  - `enabled`: مفتاح افتراضي رئيسي (يمكن للمزوّدين تجاوزه؛ ويستخدم Discord `channels.discord.threadBindings.enabled`)
  - `idleHours`: الافتراضي العام لإلغاء التركيز التلقائي بعد عدم النشاط بالساعات (`0` للتعطيل؛ ويمكن للمزوّدين تجاوزه)
  - `maxAgeHours`: الافتراضي العام للعمر الأقصى الصارم بالساعات (`0` للتعطيل؛ ويمكن للمزوّدين تجاوزه)

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

تجاوزات لكل قناة/حساب: `channels.<channel>.responsePrefix`، و`channels.<channel>.accounts.<id>.responsePrefix`.

آلية الحل (الأكثر تحديدًا يفوز): الحساب → القناة → العام. يؤدي `""` إلى التعطيل وإيقاف التسلسل. ويستنتج `"auto"` القيمة `[{identity.name}]`.

**متغيرات القالب:**

| المتغير           | الوصف                 | المثال                      |
| ----------------- | --------------------- | --------------------------- |
| `{model}`         | اسم النموذج المختصر   | `claude-opus-4-6`           |
| `{modelFull}`     | معرّف النموذج الكامل  | `anthropic/claude-opus-4-6` |
| `{provider}`      | اسم المزوّد           | `anthropic`                 |
| `{thinkingLevel}` | مستوى التفكير الحالي  | `high`، `low`، `off`        |
| `{identity.name}` | اسم هوية الوكيل       | (مثل `"auto"`)              |

المتغيرات غير حساسة لحالة الأحرف. ويُعد `{think}` اسمًا مستعارًا لـ `{thinkingLevel}`.

### تفاعل الإقرار

- يكون افتراضيًا `identity.emoji` للوكيل النشط، وإلا `"👀"`. اضبطه إلى `""` للتعطيل.
- تجاوزات لكل قناة: `channels.<channel>.ackReaction`، و`channels.<channel>.accounts.<id>.ackReaction`.
- ترتيب الحل: الحساب → القناة → `messages.ackReaction` → هوية احتياطية.
- النطاق: `group-mentions` (افتراضي)، أو `group-all`، أو `direct`، أو `all`.
- يزيل `removeAckAfterReply` تفاعل الإقرار بعد الرد في Slack وDiscord وTelegram.
- يفعّل `messages.statusReactions.enabled` تفاعلات الحالة عبر دورة الحياة في Slack وDiscord وTelegram.
  في Slack وDiscord، يؤدي تركه غير معيّن إلى بقاء تفاعلات الحالة مفعلة عندما تكون تفاعلات الإقرار نشطة.
  وفي Telegram، اضبطه صراحةً إلى `true` لتمكين تفاعلات الحالة عبر دورة الحياة.

### إزالة الارتداد للرسائل الواردة

يجمع الرسائل النصية السريعة المتتابعة من المرسل نفسه في دور وكيل واحد. تؤدي الوسائط/المرفقات إلى التفريغ فورًا. تتجاوز أوامر التحكم إزالة الارتداد.

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

- يتحكم `auto` في TTS التلقائي. ويجاوز `/tts off|always|inbound|tagged` ذلك لكل جلسة.
- يجاوز `summaryModel` القيمة `agents.defaults.model.primary` للملخص التلقائي.
- يكون `modelOverrides` مفعّلًا افتراضيًا؛ وتكون `modelOverrides.allowProvider` افتراضيًا `false` (اشتراك اختياري).
- تعود مفاتيح API إلى `ELEVENLABS_API_KEY`/`XI_API_KEY` و`OPENAI_API_KEY`.
- يجاوز `openai.baseUrl` نقطة نهاية TTS الخاصة بـ OpenAI. ترتيب الحل هو: الإعدادات، ثم `OPENAI_TTS_BASE_URL`، ثم `https://api.openai.com/v1`.
- عندما يشير `openai.baseUrl` إلى نقطة نهاية غير OpenAI، يتعامل OpenClaw معها على أنها خادم TTS متوافق مع OpenAI ويخفف التحقق من النموذج/الصوت.

---

## Talk

الإعدادات الافتراضية لوضع Talk (macOS/iOS/Android).

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

- يجب أن يطابق `talk.provider` مفتاحًا في `talk.providers` عندما تكون هناك عدة مزوّدي Talk مكوّنين.
- مفاتيح Talk المسطحة القديمة (`talk.voiceId` و`talk.voiceAliases` و`talk.modelId` و`talk.outputFormat` و`talk.apiKey`) هي للتوافق فقط ويتم ترحيلها تلقائيًا إلى `talk.providers.<provider>`.
- تعود معرّفات الأصوات إلى `ELEVENLABS_VOICE_ID` أو `SAG_VOICE_ID`.
- تقبل `providers.*.apiKey` سلاسل نصية صريحة أو كائنات SecretRef.
- ينطبق البديل `ELEVENLABS_API_KEY` فقط عندما لا يكون مفتاح Talk API مكوّنًا.
- يتيح `providers.*.voiceAliases` لتوجيهات Talk استخدام أسماء ودية.
- يتحكم `silenceTimeoutMs` في مدة انتظار وضع Talk بعد صمت المستخدم قبل إرسال transcript. يؤدي حذفه إلى الإبقاء على نافذة التوقف الافتراضية للمنصة (`700 ms على macOS وAndroid، و900 ms على iOS`).

---

## الأدوات

### ملفات تعريف الأدوات

يضبط `tools.profile` قائمة سماح أساسية قبل `tools.allow`/`tools.deny`:

تفترض عملية الإعداد المحلي افتراضيًا الإعدادات المحلية الجديدة إلى `tools.profile: "coding"` عندما يكون غير معيّن (مع الحفاظ على الملفات الصريحة الموجودة).

| الملف التعريفي | يتضمن                                                                                                                         |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`      | `session_status` فقط                                                                                                            |
| `coding`       | `group:fs` و`group:runtime` و`group:web` و`group:sessions` و`group:memory` و`cron` و`image` و`image_generate` و`video_generate` |
| `messaging`    | `group:messaging` و`sessions_list` و`sessions_history` و`sessions_send` و`session_status`                                      |
| `full`         | بلا قيود (مثل غير المعيّن)                                                                                                     |

### مجموعات الأدوات

| المجموعة           | الأدوات                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec` و`process` و`code_execution` (ويُقبل `bash` كاسم مستعار لـ `exec`)                                               |
| `group:fs`         | `read` و`write` و`edit` و`apply_patch`                                                                                   |
| `group:sessions`   | `sessions_list` و`sessions_history` و`sessions_send` و`sessions_spawn` و`sessions_yield` و`subagents` و`session_status` |
| `group:memory`     | `memory_search` و`memory_get`                                                                                            |
| `group:web`        | `web_search` و`x_search` و`web_fetch`                                                                                    |
| `group:ui`         | `browser` و`canvas`                                                                                                      |
| `group:automation` | `cron` و`gateway`                                                                                                        |
| `group:messaging`  | `message`                                                                                                                |
| `group:nodes`      | `nodes`                                                                                                                  |
| `group:agents`     | `agents_list`                                                                                                            |
| `group:media`      | `image` و`image_generate` و`video_generate` و`tts`                                                                       |
| `group:openclaw`   | جميع الأدوات المضمّنة (باستثناء إضافات مزوّدي plugin)                                                                   |

### `tools.allow` / `tools.deny`

سياسة السماح/المنع العامة للأدوات (المنع يفوز). غير حساسة لحالة الأحرف، وتدعم أحرفًا عامة `*`. تُطبق حتى عند إيقاف Docker sandbox.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

تقييد إضافي للأدوات لمزوّدين أو نماذج محددة. الترتيب: الملف التعريفي الأساسي → ملف المزود → السماح/المنع.

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

- لا يمكن لتجاوز كل وكيل (`agents.list[].tools.elevated`) إلا أن يضيف مزيدًا من التقييد.
- يخزن `/elevated on|off|ask|full` الحالة لكل جلسة؛ وتُطبق التوجيهات المضمنة على رسالة واحدة.
- يتجاوز `exec` المرتفع الحماية داخل sandbox ويستخدم مسار الخروج المكوّن (`gateway` افتراضيًا، أو `node` عندما يكون هدف exec هو `node`).

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

تكون فحوصات أمان حلقات الأدوات **معطلة افتراضيًا**. اضبط `enabled: true` لتفعيل الاكتشاف.
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

- `historySize`: أقصى سجل لاستدعاءات الأدوات محتفَظ به لتحليل الحلقات.
- `warningThreshold`: حد نمط التكرار بلا تقدم لإصدار التحذيرات.
- `criticalThreshold`: حد أعلى للتكرار لحظر الحلقات الحرجة.
- `globalCircuitBreakerThreshold`: حد إيقاف صارم لأي تشغيل بلا تقدم.
- `detectors.genericRepeat`: التحذير عند تكرار نفس الأداة/نفس الوسائط.
- `detectors.knownPollNoProgress`: التحذير/الحظر لأدوات الاستطلاع المعروفة (`process.poll` و`command_status` وغيرهما).
- `detectors.pingPong`: التحذير/الحظر للأنماط الزوجية المتناوبة بلا تقدم.
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

<Accordion title="حقول إدخال نموذج الوسائط">

**إدخال المزوّد** (`type: "provider"` أو محذوف):

- `provider`: معرّف مزود API (`openai` أو `anthropic` أو `google`/`gemini` أو `groq`، إلخ)
- `model`: تجاوز معرّف النموذج
- `profile` / `preferredProfile`: اختيار ملف `auth-profiles.json`

**إدخال CLI** (`type: "cli"`):

- `command`: البرنامج التنفيذي المراد تشغيله
- `args`: وسائط بقوالب (تدعم `{{MediaPath}}` و`{{Prompt}}` و`{{MaxChars}}`، إلخ)

**الحقول المشتركة:**

- `capabilities`: قائمة اختيارية (`image`، `audio`، `video`). القيم الافتراضية: `openai`/`anthropic`/`minimax` → image، و`google` → image+audio+video، و`groq` → audio.
- `prompt` و`maxChars` و`maxBytes` و`timeoutSeconds` و`language`: تجاوزات لكل إدخال.
- تعود الإخفاقات إلى الإدخال التالي.

تتبع مصادقة المزوّد الترتيب القياسي: `auth-profiles.json` → متغيرات البيئة → `models.providers.*.apiKey`.

**حقول الإكمال غير المتزامن:**

- `asyncCompletion.directSend`: عندما تكون `true`، تحاول مهام `music_generate`
  و`video_generate` غير المتزامنة المكتملة التسليم المباشر إلى القناة أولًا. الافتراضي: `false`
  (مسار الإيقاظ/التسليم بالنموذج الخاص بجلسة الطالب القديمة).

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

يتحكم في الجلسات التي يمكن استهدافها بواسطة أدوات الجلسات (`sessions_list`، `sessions_history`، `sessions_send`).

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
- `agent`: أي جلسة تخص معرّف الوكيل الحالي (وقد يشمل مستخدمين آخرين إذا كنت تشغّل جلسات لكل مرسل تحت نفس معرّف الوكيل).
- `all`: أي جلسة. وما يزال الاستهداف عبر الوكلاء يتطلب `tools.agentToAgent`.
- تقييد sandbox: عندما تكون الجلسة الحالية داخل sandbox وكان `agents.defaults.sandbox.sessionToolsVisibility="spawned"`، تُفرض الرؤية إلى `tree` حتى لو كانت `tools.sessions.visibility="all"`.

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

- المرفقات مدعومة فقط مع `runtime: "subagent"`. ويرفض ACP runtime هذه المرفقات.
- يتم إنشاء الملفات داخل مساحة عمل الابن في `.openclaw/attachments/<uuid>/` مع ملف `.manifest.json`.
- يتم إخفاء محتوى المرفقات تلقائيًا من حفظ transcript.
- يتم التحقق من مدخلات Base64 بفحص صارم للأبجدية/الحشو وحارس حجم قبل فك الترميز.
- أذونات الملفات هي `0700` للأدلة و`0600` للملفات.
- يتبع التنظيف سياسة `cleanup`: يقوم `delete` دائمًا بإزالة المرفقات؛ بينما يحتفظ `keep` بها فقط عندما تكون `retainOnSessionKeep: true`.

### `tools.experimental`

أعلام الأدوات المضمّنة التجريبية. تكون معطلة افتراضيًا ما لم تنطبق قاعدة تمكين تلقائي خاصة بوقت التشغيل.

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

- `planTool`: يفعّل أداة `update_plan` التجريبية المنظمة لتتبع الأعمال غير البسيطة متعددة الخطوات.
- الافتراضي: `false` لمزوّدي غير OpenAI. وتقوم تشغيلات OpenAI وOpenAI Codex بتفعيلها تلقائيًا.
- عندما تكون مفعّلة، يضيف system prompt أيضًا إرشادات استخدام بحيث يستخدمها النموذج فقط للأعمال الكبيرة ويحافظ على خطوة واحدة كحد أقصى في حالة `in_progress`.

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
- `allowAgents`: قائمة السماح الافتراضية لمعرّفات الوكلاء الهدف لـ `sessions_spawn` عندما لا يعيّن وكيل الطالب `subagents.allowAgents` الخاص به (`["*"]` = أي وكيل؛ الافتراضي: نفس الوكيل فقط).
- `runTimeoutSeconds`: المهلة الافتراضية (بالثواني) لـ `sessions_spawn` عندما يحذف استدعاء الأداة `runTimeoutSeconds`. وتعني `0` عدم وجود مهلة.
- سياسة الأدوات لكل وكيل فرعي: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## المزوّدون المخصصون وعناوين URL الأساسية

يستخدم OpenClaw فهرس النماذج المضمّن. أضف مزوّدين مخصصين عبر `models.providers` في الإعدادات أو `~/.openclaw/agents/<agentId>/agent/models.json`.

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
- تجاوز جذر إعدادات الوكيل عبر `OPENCLAW_AGENT_DIR` (أو `PI_CODING_AGENT_DIR`، وهو اسم مستعار قديم لمتغير البيئة).
- أولوية الدمج لمعرّفات المزوّد المطابقة:
  - تفوز قيم `baseUrl` غير الفارغة في `models.json` الخاص بالوكيل.
  - تفوز قيم `apiKey` غير الفارغة في الوكيل فقط عندما لا يكون ذلك المزوّد مُدارًا عبر SecretRef في سياق الإعدادات/ملف المصادقة الحالي.
  - يتم تحديث قيم `apiKey` للمزوّد المُدار عبر SecretRef من علامات المصدر (`ENV_VAR_NAME` لمراجع env، و`secretref-managed` لمراجع file/exec) بدلًا من حفظ الأسرار المحلولة.
  - يتم تحديث قيم رؤوس المزوّد المُدارة عبر SecretRef من علامات المصدر (`secretref-env:ENV_VAR_NAME` لمراجع env، و`secretref-managed` لمراجع file/exec).
  - تعود قيم `apiKey`/`baseUrl` الفارغة أو المفقودة في الوكيل إلى `models.providers` في الإعدادات.
  - تستخدم قيم `contextWindow`/`maxTokens` الأعلى بين الإعداد الصريح وقيم الفهرس الضمنية للنموذج المطابق.
  - تحفظ `contextTokens` المطابقة حد وقت التشغيل الصريح عندما يكون موجودًا؛ استخدمها لتقييد السياق الفعّال من دون تغيير بيانات النموذج الأصلية.
  - استخدم `models.mode: "replace"` عندما تريد أن تعيد الإعدادات كتابة `models.json` بالكامل.
  - يكون حفظ العلامات معتمدًا على المصدر: تُكتب العلامات من لقطة الإعدادات النشطة للمصدر (قبل الحل)، وليس من قيم الأسرار المحلولة وقت التشغيل.

### تفاصيل حقول المزوّد

- `models.mode`: سلوك فهرس المزوّد (`merge` أو `replace`).
- `models.providers`: خريطة المزوّدين المخصصين مفهرسة حسب معرّف المزوّد.
- `models.providers.*.api`: محول الطلب (`openai-completions` أو `openai-responses` أو `anthropic-messages` أو `google-generative-ai`، إلخ).
- `models.providers.*.apiKey`: اعتماد المزوّد (يفضَّل SecretRef/استبدال env).
- `models.providers.*.auth`: استراتيجية المصادقة (`api-key` أو `token` أو `oauth` أو `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: بالنسبة إلى Ollama مع `openai-completions`، يحقن `options.num_ctx` في الطلبات (الافتراضي: `true`).
- `models.providers.*.authHeader`: فرض نقل الاعتماد في رأس `Authorization` عند الحاجة.
- `models.providers.*.baseUrl`: عنوان API الأساسي الأعلى.
- `models.providers.*.headers`: رؤوس ثابتة إضافية لتوجيه proxy/المستأجر.
- `models.providers.*.request`: تجاوزات النقل لطلبات HTTP الخاصة بمزوّد النموذج.
  - `request.headers`: رؤوس إضافية (تُدمج مع افتراضيات المزوّد). القيم تقبل SecretRef.
  - `request.auth`: تجاوز لاستراتيجية المصادقة. الأوضاع: `"provider-default"` (استخدام المصادقة المضمنة للمزوّد)، أو `"authorization-bearer"` (مع `token`)، أو `"header"` (مع `headerName` و`value` و`prefix` الاختياري).
  - `request.proxy`: تجاوز لوكيل HTTP. الأوضاع: `"env-proxy"` (استخدام متغيري البيئة `HTTP_PROXY`/`HTTPS_PROXY`)، و`"explicit-proxy"` (مع `url`). ويقبل كلا الوضعين كائن `tls` اختياري.
  - `request.tls`: تجاوز TLS للاتصالات المباشرة. الحقول: `ca` و`cert` و`key` و`passphrase` (كلها تقبل SecretRef)، و`serverName`، و`insecureSkipVerify`.
- `models.providers.*.models`: إدخالات فهرس نماذج صريحة للمزوّد.
- `models.providers.*.models.*.contextWindow`: بيانات تعريف نافذة السياق الأصلية للنموذج.
- `models.providers.*.models.*.contextTokens`: حد سياق اختياري لوقت التشغيل. استخدمه عندما تريد ميزانية سياق فعّالة أصغر من `contextWindow` الأصلية للنموذج.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: تلميح توافق اختياري. بالنسبة إلى `api: "openai-completions"` مع `baseUrl` غير أصلي وغير فارغ (مضيف ليس `api.openai.com`)، يفرض OpenClaw هذا إلى `false` وقت التشغيل. ويُبقي `baseUrl` الفارغ/المحذوف سلوك OpenAI الافتراضي.
- `models.providers.*.models.*.compat.requiresStringContent`: تلميح توافق اختياري لنقاط نهاية الدردشة المتوافقة مع OpenAI التي تقبل السلاسل فقط. عندما يكون `true`، يسطّح OpenClaw مصفوفات `messages[].content` النصية البحتة إلى سلاسل عادية قبل إرسال الطلب.
- `plugins.entries.amazon-bedrock.config.discovery`: جذر إعدادات الاكتشاف التلقائي لـ Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: تشغيل/إيقاف الاكتشاف الضمني.
- `plugins.entries.amazon-bedrock.config.discovery.region`: منطقة AWS للاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: مرشح معرّف مزوّد اختياري للاكتشاف الموجّه.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: فترة الاستطلاع لتحديث الاكتشاف.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: نافذة سياق احتياطية للنماذج المكتشفة.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: الحد الاحتياطي الأقصى لرموز الخرج للنماذج المكتشفة.

### أمثلة على المزوّدين

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

اضبط `ZAI_API_KEY`. يُقبل `z.ai/*` و`z-ai/*` كأسماء مستعارة. اختصار: `openclaw onboard --auth-choice zai-api-key`.

- نقطة النهاية العامة: `https://api.z.ai/api/paas/v4`
- نقطة نهاية البرمجة (افتراضية): `https://api.z.ai/api/coding/paas/v4`
- بالنسبة لنقطة النهاية العامة، عرّف مزودًا مخصصًا مع تجاوز `baseUrl`.

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

تعلن نقاط نهاية Moonshot الأصلية توافق استخدام البث على مسار النقل المشترك
`openai-completions`، ويعتمد OpenClaw الآن ذلك على قدرات نقطة النهاية
بدلًا من الاكتفاء بمعرّف المزوّد المضمّن وحده.

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

متوافق مع Anthropic ومزوّد مضمّن. اختصار: `openclaw onboard --auth-choice kimi-code-api-key`.

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

يجب أن يحذف `baseUrl` اللاحقة `/v1` (عميل Anthropic يضيفها). اختصار: `openclaw onboard --auth-choice synthetic-api-key`.

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
يفترض فهرس النماذج الآن M2.7 فقط.
على مسار البث المتوافق مع Anthropic، يعطّل OpenClaw التفكير في MiniMax
افتراضيًا ما لم تضبط `thinking` بنفسك صراحةً. يعيد `/fast on` أو
`params.fastMode: true` كتابة `MiniMax-M2.7` إلى
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="النماذج المحلية (LM Studio)">

راجع [النماذج المحلية](/ar/gateway/local-models). باختصار: شغّل نموذجًا محليًا كبيرًا عبر LM Studio Responses API على عتاد قوي؛ واحتفظ بالنماذج المستضافة مدمجة من أجل fallback.

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

- `allowBundled`: قائمة سماح اختيارية لـ Skills المضمنة فقط (ولا تتأثر Skills المُدارة/Skills مساحة العمل).
- `load.extraDirs`: جذور Skills مشتركة إضافية (بأدنى أولوية).
- `install.preferBrew`: عندما تكون true، يفضّل مُثبّتات Homebrew عندما يكون `brew`
  متاحًا قبل الرجوع إلى أنواع المثبتات الأخرى.
- `install.nodeManager`: تفضيل مُثبّت Node لمواصفات `metadata.openclaw.install`
  (`npm` | `pnpm` | `yarn` | `bun`).
- `entries.<skillKey>.enabled: false` يعطّل Skill حتى لو كان مضمّنًا/مثبتًا.
- `entries.<skillKey>.apiKey`: حقل ملاءمة لمفتاح API للـ Skills التي تعلن متغير env أساسيًا (سلسلة صريحة أو كائن SecretRef).

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

- يتم تحميلها من `~/.openclaw/extensions` و`<workspace>/.openclaw/extensions` و`plugins.load.paths`.
- يقبل الاكتشاف إضافات OpenClaw الأصلية بالإضافة إلى حزم Codex المتوافقة وحزم Claude، بما في ذلك حزم Claude بلا manifest ذات التخطيط الافتراضي.
- **تتطلب تغييرات الإعدادات إعادة تشغيل للبوابة.**
- `allow`: قائمة سماح اختيارية (يتم تحميل الإضافات المدرجة فقط). يفوز `deny`.
- `plugins.entries.<id>.apiKey`: حقل ملاءمة لمفتاح API على مستوى الإضافة (عندما تدعمه الإضافة).
- `plugins.entries.<id>.env`: خريطة متغيرات بيئة ضمن نطاق الإضافة.
- `plugins.entries.<id>.hooks.allowPromptInjection`: عندما تكون `false`، يمنع core الخطاف `before_prompt_build` ويتجاهل الحقول المعدِّلة للـ prompt من `before_agent_start` القديم، مع الحفاظ على `modelOverride` و`providerOverride` القديمين. ينطبق ذلك على خطافات الإضافات الأصلية وأدلة الخطافات المدعومة التي توفرها الحزم.
- `plugins.entries.<id>.subagent.allowModelOverride`: يثق صراحةً في هذه الإضافة لطلب تجاوزات `provider` و`model` لكل تشغيل أثناء تشغيلات الوكلاء الفرعيين في الخلفية.
- `plugins.entries.<id>.subagent.allowedModels`: قائمة سماح اختيارية للأهداف القياسية `provider/model` لتجاوزات الوكلاء الفرعيين الموثوقة. استخدم `"*"` فقط عندما تريد عمدًا السماح بأي نموذج.
- `plugins.entries.<id>.config`: كائن إعدادات معرَّف بواسطة الإضافة (يتم التحقق منه وفق مخطط إضافة OpenClaw الأصلية عندما يكون متاحًا).
- `plugins.entries.firecrawl.config.webFetch`: إعدادات مزود جلب الويب Firecrawl.
  - `apiKey`: مفتاح API لـ Firecrawl (يقبل SecretRef). ويعود إلى `plugins.entries.firecrawl.config.webSearch.apiKey`، أو `tools.web.fetch.firecrawl.apiKey` القديم، أو متغير البيئة `FIRECRAWL_API_KEY`.
  - `baseUrl`: عنوان API الأساسي لـ Firecrawl (الافتراضي: `https://api.firecrawl.dev`).
  - `onlyMainContent`: استخراج المحتوى الرئيسي فقط من الصفحات (الافتراضي: `true`).
  - `maxAgeMs`: الحد الأقصى لعمر الذاكرة المؤقتة بالمللي ثانية (الافتراضي: `172800000` / يومان).
  - `timeoutSeconds`: مهلة طلب الاستخراج scrape بالثواني (الافتراضي: `60`).
- `plugins.entries.xai.config.xSearch`: إعدادات xAI X Search (بحث الويب Grok).
  - `enabled`: تفعيل مزود X Search.
  - `model`: نموذج Grok المستخدم للبحث (مثل `"grok-4-1-fast"`).
- `plugins.entries.memory-core.config.dreaming`: إعدادات dreaming الخاصة بالذاكرة (تجريبية). راجع [Dreaming](/ar/concepts/dreaming) للمراحل والحدود.
  - `enabled`: المفتاح الرئيسي لـ dreaming (الافتراضي `false`).
  - `frequency`: إيقاع cron لكل دورة dreaming كاملة (`"0 3 * * *"` افتراضيًا).
  - سياسة المراحل والحدود هي تفاصيل تنفيذية (وليست مفاتيح إعدادات موجهة للمستخدم).
- يمكن لإضافات حزم Claude المفعّلة أيضًا المساهمة بافتراضيات Pi مضمّنة من `settings.json`؛ ويطبقها OpenClaw كإعدادات وكيل منقّحة، لا كتصحيحات إعدادات OpenClaw خام.
- `plugins.slots.memory`: اختر معرّف إضافة الذاكرة النشطة، أو `"none"` لتعطيل إضافات الذاكرة.
- `plugins.slots.contextEngine`: اختر معرّف محرك السياق النشط؛ ويكون افتراضيًا `"legacy"` ما لم تثبّت وتحدد محركًا آخر.
- `plugins.installs`: بيانات تعريف التثبيت المُدارة من CLI والمستخدمة بواسطة `openclaw plugins update`.
  - تتضمن `source` و`spec` و`sourcePath` و`installPath` و`version` و`resolvedName` و`resolvedVersion` و`resolvedSpec` و`integrity` و`shasum` و`resolvedAt` و`installedAt`.
  - تعامل مع `plugins.installs.*` كحالة مُدارة؛ وفضّل أوامر CLI بدل التحرير اليدوي.

راجع [Plugins](/ar/tools/plugin).

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

- `evaluateEnabled: false` يعطّل `act:evaluate` و`wait --fn`.
- يكون `ssrfPolicy.dangerouslyAllowPrivateNetwork` افتراضيًا `true` عندما لا يُعيّن (نموذج شبكة موثوقة).
- اضبط `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` لتصفح متشدد عام فقط.
- في الوضع المتشدد، تخضع نقاط نهاية ملف CDP البعيد (`profiles.*.cdpUrl`) لنفس حظر الشبكة الخاصة أثناء فحوصات الوصول/الاكتشاف.
- يظل `ssrfPolicy.allowPrivateNetwork` مدعومًا كاسم مستعار قديم.
- في الوضع المتشدد، استخدم `ssrfPolicy.hostnameAllowlist` و`ssrfPolicy.allowedHostnames` للاستثناءات الصريحة.
- تكون الملفات الشخصية البعيدة attach-only (بدء/إيقاف/إعادة تعيين معطلة).
- يقبل `profiles.*.cdpUrl` القيم `http://` و`https://` و`ws://` و`wss://`.
  استخدم HTTP(S) عندما تريد من OpenClaw اكتشاف `/json/version`؛ واستخدم WS(S)
  عندما يوفّر لك المزوّد عنوان DevTools WebSocket مباشرًا.
- تكون ملفات `existing-session` محصورة بالمضيف فقط وتستخدم Chrome MCP بدل CDP.
- يمكن لملفات `existing-session` تعيين `userDataDir` لاستهداف ملف
  Chromium-based browser محدد مثل Brave أو Edge.
- تحافظ ملفات `existing-session` على قيود مسارات Chrome MCP الحالية:
  إجراءات قائمة على snapshot/ref بدل الاستهداف عبر CSS-selector، وخطافات رفع
  ملف واحد، ومن دون تجاوزات مهلة مربعات الحوار، ومن دون `wait --load networkidle`،
  ومن دون `responsebody` أو تصدير PDF أو اعتراض التنزيلات أو الإجراءات الدفعية.
- تقوم ملفات `openclaw` المحلية المُدارة بتعيين `cdpPort` و`cdpUrl` تلقائيًا؛ لا
  تعيّن `cdpUrl` صراحة إلا لـ CDP البعيد.
- ترتيب الاكتشاف التلقائي: المتصفح الافتراضي إذا كان Chromium-based → Chrome → Brave → Edge → Chromium → Chrome Canary.
- خدمة التحكم: loopback فقط (منفذ مشتق من `gateway.port`، الافتراضي `18791`).
- يضيف `extraArgs` رايات تشغيل إضافية إلى بدء Chromium المحلي (على سبيل المثال
  `--disable-gpu`، أو تحديد حجم النافذة، أو رايات التصحيح).

---

## واجهة المستخدم

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: