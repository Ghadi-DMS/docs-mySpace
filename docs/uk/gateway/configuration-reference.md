---
read_when:
    - Вам потрібні точні семантики або значення за замовчуванням для конфігурації на рівні полів
    - Ви перевіряєте блоки конфігурації каналу, моделі, gateway або інструментів
summary: Довідник конфігурації Gateway для основних ключів OpenClaw, значень за замовчуванням і посилань на окремі довідники підсистем
title: Довідник конфігурації
x-i18n:
    generated_at: "2026-04-08T19:05:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: a9d6d0c542b9874809491978fdcf8e1a7bb35a4873db56aa797963d03af4453c
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# Довідник конфігурації

Основний довідник конфігурації для `~/.openclaw/openclaw.json`. Для огляду, орієнтованого на завдання, див. [Configuration](/uk/gateway/configuration).

Ця сторінка охоплює основні поверхні конфігурації OpenClaw і містить посилання назовні, коли підсистема має власний глибший довідник. Вона **не** намагається вбудувати на одну сторінку кожен каталог команд, що належить каналу/плагіну, або кожен глибокий параметр пам’яті/QMD.

Джерело істини в коді:

- `openclaw config schema` виводить актуальну JSON Schema, яка використовується для валідації та Control UI, із злитими метаданими bundled/plugin/channel, коли вони доступні
- `config.schema.lookup` повертає один вузол схеми з областю шляху для інструментів деталізації
- `pnpm config:docs:check` / `pnpm config:docs:gen` перевіряють baseline hash документації конфігурації відносно поточної поверхні схеми

Окремі глибокі довідники:

- [Memory configuration reference](/uk/reference/memory-config) для `agents.defaults.memorySearch.*`, `memory.qmd.*`, `memory.citations` і конфігурації dreaming у `plugins.entries.memory-core.config.dreaming`
- [Slash Commands](/uk/tools/slash-commands) для поточного каталогу built-in + bundled команд
- сторінки власника каналу/плагіна для поверхонь команд, специфічних для каналу

Формат конфігурації — **JSON5** (дозволені коментарі + кінцеві коми). Усі поля необов’язкові — OpenClaw використовує безпечні значення за замовчуванням, якщо їх пропущено.

---

## Канали

Кожен канал запускається автоматично, коли існує його секція конфігурації (якщо не вказано `enabled: false`).

### Доступ до DM і груп

Усі канали підтримують політики DM і групові політики:

| Політика DM         | Поведінка                                                      |
| ------------------- | -------------------------------------------------------------- |
| `pairing` (типово)  | Невідомі відправники отримують одноразовий код pair; власник має схвалити |
| `allowlist`         | Лише відправники в `allowFrom` (або в paired allow store)      |
| `open`              | Дозволити всі вхідні DM (потрібно `allowFrom: ["*"]`)          |
| `disabled`          | Ігнорувати всі вхідні DM                                       |

| Групова політика      | Поведінка                                              |
| --------------------- | ------------------------------------------------------ |
| `allowlist` (типово)  | Лише групи, що відповідають налаштованому allowlist    |
| `open`                | Обходить group allowlist-и (gating за згадками все ще застосовується) |
| `disabled`            | Блокує всі повідомлення груп/кімнат                    |

<Note>
`channels.defaults.groupPolicy` задає значення за замовчуванням, коли `groupPolicy` провайдера не встановлено.
Коди pair-ing спливають через 1 годину. Непідтверджені запити на DM pairing обмежені **3 на канал**.
Якщо блок провайдера відсутній повністю (немає `channels.<provider>`), політика груп під час виконання повертається до `allowlist` (fail-closed) з попередженням під час запуску.
</Note>

### Перевизначення моделі для каналів

Використовуйте `channels.modelByChannel`, щоб закріпити конкретні ID каналів за моделлю. Значення приймають `provider/model` або налаштовані псевдоніми моделей. Прив’язка каналу застосовується, коли в сесії ще немає перевизначення моделі (наприклад, встановленого через `/model`).

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

### Значення каналів за замовчуванням і heartbeat

Використовуйте `channels.defaults` для спільної поведінки group-policy і heartbeat між провайдерами:

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

- `channels.defaults.groupPolicy`: запасна групова політика, коли `groupPolicy` на рівні провайдера не встановлено.
- `channels.defaults.contextVisibility`: типовий режим видимості додаткового контексту для всіх каналів. Значення: `all` (типово, включати весь quoted/thread/history контекст), `allowlist` (включати контекст лише від allowlisted відправників), `allowlist_quote` (те саме, що allowlist, але зберігати явний контекст quote/reply). Перевизначення на рівні каналу: `channels.<channel>.contextVisibility`.
- `channels.defaults.heartbeat.showOk`: включати стани справних каналів у вивід heartbeat.
- `channels.defaults.heartbeat.showAlerts`: включати стани з деградацією/помилками у вивід heartbeat.
- `channels.defaults.heartbeat.useIndicator`: відображати компактний вивід heartbeat у стилі індикатора.

### WhatsApp

WhatsApp працює через web channel gateway (`Baileys Web`). Він запускається автоматично, коли існує пов’язана сесія.

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      chunkMode: "length", // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // сині галочки (false у режимі self-chat)
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

<Accordion title="Багатообліковий WhatsApp">

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

- Вихідні команди типово використовують обліковий запис `default`, якщо він є; інакше — перший налаштований id облікового запису (відсортований).
- Необов’язковий `channels.whatsapp.defaultAccount` перевизначає цей вибір запасного облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.
- Застарілий однообліковий Baileys auth dir мігрується `openclaw doctor` у `whatsapp/default`.
- Перевизначення на рівні облікового запису: `channels.whatsapp.accounts.<id>.sendReadReceipts`, `channels.whatsapp.accounts.<id>.dmPolicy`, `channels.whatsapp.accounts.<id>.allowFrom`.

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
      streaming: "partial", // off | partial | block | progress (типово: off; явно вмикайте, щоб уникати rate limits preview-edit)
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

- Токен бота: `channels.telegram.botToken` або `channels.telegram.tokenFile` (лише звичайний файл; symlink відхиляються), з `TELEGRAM_BOT_TOKEN` як fallback для облікового запису за замовчуванням.
- Необов’язковий `channels.telegram.defaultAccount` перевизначає вибір облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.
- У багатооблікових конфігураціях (2+ id облікових записів) задайте явний обліковий запис за замовчуванням (`channels.telegram.defaultAccount` або `channels.telegram.accounts.default`), щоб уникнути запасної маршрутизації; `openclaw doctor` попереджає, якщо цього немає або це недійсно.
- `configWrites: false` блокує записи конфігурації, ініційовані через Telegram (міграції ID supergroup, `/config set|unset`).
- Елементи верхнього рівня `bindings[]` з `type: "acp"` налаштовують постійні ACP bindings для тем форуму (використовуйте канонічний `chatId:topic:topicId` у `match.peer.id`). Семантика полів спільна в [ACP Agents](/uk/tools/acp-agents#channel-specific-settings).
- Попередній перегляд потоків Telegram використовує `sendMessage` + `editMessageText` (працює в особистих і групових чатах).
- Політика retry: див. [Retry policy](/uk/concepts/retry).

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
      streaming: "off", // off | partial | block | progress (progress відображається як partial у Discord)
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
        spawnSubagentSessions: false, // opt-in для `sessions_spawn({ thread: true })`
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

- Токен: `channels.discord.token`, з `DISCORD_BOT_TOKEN` як fallback для облікового запису за замовчуванням.
- Прямі вихідні виклики, які надають явний Discord `token`, використовують цей токен для виклику; налаштування retry/policy облікового запису все одно беруться з вибраного облікового запису в активному runtime snapshot.
- Необов’язковий `channels.discord.defaultAccount` перевизначає вибір облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.
- Використовуйте `user:<id>` (DM) або `channel:<id>` (канал guild) для цілей доставки; прості числові ID відхиляються.
- Slug guild-ів мають нижній регістр із пробілами, заміненими на `-`; ключі каналів використовують slugged name (без `#`). Надавайте перевагу ID guild.
- Повідомлення, написані ботом, типово ігноруються. `allowBots: true` вмикає їх; використовуйте `allowBots: "mentions"`, щоб приймати лише повідомлення ботів, які згадують бота (власні повідомлення все одно фільтруються).
- `channels.discord.guilds.<id>.ignoreOtherMentions` (і перевизначення каналу) відкидає повідомлення, які згадують іншого користувача чи роль, але не бота (за винятком @everyone/@here).
- `maxLinesPerMessage` (типово 17) розбиває високі повідомлення, навіть якщо вони коротші за 2000 символів.
- `channels.discord.threadBindings` керує маршрутизацією, прив’язаною до Discord thread:
  - `enabled`: перевизначення Discord для можливостей сесій, прив’язаних до thread (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`, а також bound delivery/routing)
  - `idleHours`: перевизначення Discord для автоматичного unfocus через бездіяльність у годинах (`0` вимикає)
  - `maxAgeHours`: перевизначення Discord для жорсткого максимального віку в годинах (`0` вимикає)
  - `spawnSubagentSessions`: opt-in перемикач для автоматичного створення/прив’язки thread у `sessions_spawn({ thread: true })`
- Елементи верхнього рівня `bindings[]` з `type: "acp"` налаштовують постійні ACP bindings для каналів і thread-ів (використовуйте id каналу/thread у `match.peer.id`). Семантика полів спільна в [ACP Agents](/uk/tools/acp-agents#channel-specific-settings).
- `channels.discord.ui.components.accentColor` задає accent color для контейнерів Discord components v2.
- `channels.discord.voice` вмикає розмови в голосових каналах Discord і необов’язкові auto-join + TTS overrides.
- `channels.discord.voice.daveEncryption` і `channels.discord.voice.decryptionFailureTolerance` напряму передаються до параметрів DAVE у `@discordjs/voice` (типово `true` і `24`).
- OpenClaw додатково намагається відновити voice receive, залишаючи/повторно приєднуючись до voice session після повторних помилок дешифрування.
- `channels.discord.streaming` — канонічний ключ режиму stream. Застарілі `streamMode` і булеві значення `streaming` мігруються автоматично.
- `channels.discord.autoPresence` відображає доступність runtime у присутність бота (healthy => online, degraded => idle, exhausted => dnd) і дозволяє необов’язкові перевизначення status text.
- `channels.discord.dangerouslyAllowNameMatching` знову вмикає зіставлення mutable name/tag (режим сумісності break-glass).
- `channels.discord.execApprovals`: нативна доставка exec approval у Discord і авторизація approver.
  - `enabled`: `true`, `false` або `"auto"` (типово). В auto mode exec approvals активуються, коли approver-ів можна визначити з `approvers` або `commands.ownerAllowFrom`.
  - `approvers`: ID користувачів Discord, яким дозволено схвалювати запити exec. Якщо пропущено, fallback до `commands.ownerAllowFrom`.
  - `agentFilter`: необов’язковий allowlist ID агентів. Пропустіть, щоб пересилати approvals для всіх агентів.
  - `sessionFilter`: необов’язкові шаблони ключів сесій (substring або regex).
  - `target`: куди надсилати запити на схвалення. `"dm"` (типово) надсилає в DM approver-ам, `"channel"` надсилає в початковий канал, `"both"` надсилає в обидва місця. Коли target включає `"channel"`, кнопки можуть використовувати лише визначені approver-и.
  - `cleanupAfterResolve`: коли `true`, видаляє approval DM після схвалення, відхилення або тайм-ауту.

**Режими сповіщень про реакції:** `off` (жодних), `own` (повідомлення бота, типово), `all` (усі повідомлення), `allowlist` (від `guilds.<id>.users` на всіх повідомленнях).

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

- JSON service account: вбудований (`serviceAccount`) або на основі файла (`serviceAccountFile`).
- Також підтримується Service account SecretRef (`serviceAccountRef`).
- Fallback-и середовища: `GOOGLE_CHAT_SERVICE_ACCOUNT` або `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`.
- Використовуйте `spaces/<spaceId>` або `users/<userId>` для цілей доставки.
- `channels.googlechat.dangerouslyAllowNameMatching` знову вмикає зіставлення mutable email principal (режим сумісності break-glass).

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
        nativeTransport: true, // використовувати рідний streaming API Slack, коли mode=partial
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

- **Socket mode** потребує і `botToken`, і `appToken` (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` для env fallback облікового запису за замовчуванням).
- **HTTP mode** потребує `botToken` плюс `signingSecret` (у корені або для конкретного облікового запису).
- `botToken`, `appToken`, `signingSecret` і `userToken` приймають plaintext
  рядки або об’єкти SecretRef.
- Slack account snapshot-и надають поля джерела/стану для кожного credential, такі як
  `botTokenSource`, `botTokenStatus`, `appTokenStatus`, а в HTTP mode —
  `signingSecretStatus`. `configured_unavailable` означає, що обліковий запис
  налаштовано через SecretRef, але поточний шлях command/runtime не зміг
  визначити значення секрету.
- `configWrites: false` блокує записи конфігурації, ініційовані через Slack.
- Необов’язковий `channels.slack.defaultAccount` перевизначає вибір облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.
- `channels.slack.streaming.mode` — канонічний ключ режиму stream для Slack. `channels.slack.streaming.nativeTransport` керує рідним transport streaming у Slack. Застарілі `streamMode`, булеві `streaming` і `nativeStreaming` мігруються автоматично.
- Використовуйте `user:<id>` (DM) або `channel:<id>` для цілей доставки.

**Режими сповіщень про реакції:** `off`, `own` (типово), `all`, `allowlist` (із `reactionAllowlist`).

**Ізоляція thread session:** `thread.historyScope` — окремо для thread (типово) або спільно в межах каналу. `thread.inheritParent` копіює transcript батьківського каналу в нові thread-и.

- Рідний streaming у Slack плюс статус thread у стилі Slack assistant "is typing..." потребують ціль reply thread. DM верхнього рівня типово залишаються поза thread, тому вони використовують `typingReaction` або звичайну доставку замість preview у стилі thread.
- `typingReaction` додає тимчасову реакцію до вхідного повідомлення Slack, поки виконується відповідь, а потім прибирає її після завершення. Використовуйте shortcode emoji Slack, наприклад `"hourglass_flowing_sand"`.
- `channels.slack.execApprovals`: нативна доставка exec approval у Slack і авторизація approver. Та сама схема, що й у Discord: `enabled` (`true`/`false`/`"auto"`), `approvers` (ID користувачів Slack), `agentFilter`, `sessionFilter` і `target` (`"dm"`, `"channel"` або `"both"`).

| Група дій    | Типово   | Примітки                 |
| ------------ | -------- | ------------------------ |
| reactions    | enabled  | Реакції + список реакцій |
| messages     | enabled  | Читання/надсилання/редагування/видалення |
| pins         | enabled  | Закріпити/відкріпити/список |
| memberInfo   | enabled  | Інформація про учасника  |
| emojiList    | enabled  | Список користувацьких emoji |

### Mattermost

Mattermost постачається як плагін: `openclaw plugins install @openclaw/mattermost`.

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
        // Необов’язковий явний URL для reverse-proxy/public deployments
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      chunkMode: "length",
    },
  },
}
```

Режими чату: `oncall` (відповідати на @-згадки, типово), `onmessage` (кожне повідомлення), `onchar` (повідомлення, що починаються з trigger prefix).

Коли рідні команди Mattermost увімкнено:

- `commands.callbackPath` має бути шляхом (наприклад `/api/channels/mattermost/command`), а не повним URL.
- `commands.callbackUrl` має резолвитися до endpoint OpenClaw gateway і бути доступним із сервера Mattermost.
- Native slash callbacks автентифікуються за допомогою токенів для кожної команди, які
  Mattermost повертає під час реєстрації slash command. Якщо реєстрація не вдається або
  не активовано жодної команди, OpenClaw відхиляє callbacks із
  `Unauthorized: invalid command token.`
- Для private/tailnet/internal callback hosts Mattermost може вимагати,
  щоб `ServiceSettings.AllowedUntrustedInternalConnections` містив callback host/domain.
  Використовуйте значення host/domain, а не повні URL.
- `channels.mattermost.configWrites`: дозволити або заборонити записи конфігурації, ініційовані через Mattermost.
- `channels.mattermost.requireMention`: вимагати `@mention` перед відповіддю в каналах.
- `channels.mattermost.groups.<channelId>.requireMention`: перевизначення mention-gating для окремого каналу (`"*"` для типового значення).
- Необов’язковий `channels.mattermost.defaultAccount` перевизначає вибір облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // необов’язкова прив’язка облікового запису
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

**Режими сповіщень про реакції:** `off`, `own` (типово), `all`, `allowlist` (із `reactionAllowlist`).

- `channels.signal.account`: закріпити запуск каналу за конкретною ідентичністю облікового запису Signal.
- `channels.signal.configWrites`: дозволити або заборонити записи конфігурації, ініційовані через Signal.
- Необов’язковий `channels.signal.defaultAccount` перевизначає вибір облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.

### BlueBubbles

BlueBubbles — рекомендований шлях для iMessage (на базі плагіна, налаштовується в `channels.bluebubbles`).

```json5
{
  channels: {
    bluebubbles: {
      enabled: true,
      dmPolicy: "pairing",
      // serverUrl, password, webhookPath, керування групами та розширені дії:
      // див. /channels/bluebubbles
    },
  },
}
```

- Основні шляхи ключів, охоплені тут: `channels.bluebubbles`, `channels.bluebubbles.dmPolicy`.
- Необов’язковий `channels.bluebubbles.defaultAccount` перевизначає вибір облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.
- Елементи верхнього рівня `bindings[]` з `type: "acp"` можуть прив’язувати розмови BlueBubbles до постійних ACP sessions. Використовуйте handle або target string BlueBubbles (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) у `match.peer.id`. Спільна семантика полів: [ACP Agents](/uk/tools/acp-agents#channel-specific-settings).
- Повну конфігурацію каналу BlueBubbles задокументовано в [BlueBubbles](/uk/channels/bluebubbles).

### iMessage

OpenClaw запускає `imsg rpc` (JSON-RPC через stdio). Жоден daemon або port не потрібен.

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

- Необов’язковий `channels.imessage.defaultAccount` перевизначає вибір облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.

- Потрібен Full Disk Access до Messages DB.
- Надавайте перевагу цілям `chat_id:<id>`. Використовуйте `imsg chats --limit 20`, щоб переглянути список чатів.
- `cliPath` може вказувати на SSH wrapper; задайте `remoteHost` (`host` або `user@host`) для отримання вкладень через SCP.
- `attachmentRoots` і `remoteAttachmentRoots` обмежують шляхи вхідних вкладень (типово: `/Users/*/Library/Messages/Attachments`).
- SCP використовує сувору перевірку host-key, тому переконайтеся, що ключ relay host уже існує в `~/.ssh/known_hosts`.
- `channels.imessage.configWrites`: дозволити або заборонити записи конфігурації, ініційовані через iMessage.
- Елементи верхнього рівня `bindings[]` з `type: "acp"` можуть прив’язувати розмови iMessage до постійних ACP sessions. Використовуйте нормалізований handle або явну chat target (`chat_id:*`, `chat_guid:*`, `chat_identifier:*`) у `match.peer.id`. Спільна семантика полів: [ACP Agents](/uk/tools/acp-agents#channel-specific-settings).

<Accordion title="Приклад SSH wrapper для iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix працює через extension і налаштовується в `channels.matrix`.

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

- Автентифікація токеном використовує `accessToken`; автентифікація паролем використовує `userId` + `password`.
- `channels.matrix.proxy` маршрутизує HTTP-трафік Matrix через явний HTTP(S) proxy. Іменовані облікові записи можуть перевизначити його через `channels.matrix.accounts.<id>.proxy`.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` дозволяє приватні/внутрішні homeserver-и. `proxy` і цей network opt-in — незалежні механізми.
- `channels.matrix.defaultAccount` вибирає пріоритетний обліковий запис у багатооблікових конфігураціях.
- `channels.matrix.autoJoin` типово має значення `off`, тому запрошені кімнати й нові DM-подібні запрошення ігноруються, доки ви не встановите `autoJoin: "allowlist"` з `autoJoinAllowlist` або `autoJoin: "always"`.
- `channels.matrix.execApprovals`: нативна доставка exec approval у Matrix і авторизація approver.
  - `enabled`: `true`, `false` або `"auto"` (типово). В auto mode exec approvals активуються, коли approver-ів можна визначити з `approvers` або `commands.ownerAllowFrom`.
  - `approvers`: ID користувачів Matrix (наприклад `@owner:example.org`), яким дозволено схвалювати exec requests.
  - `agentFilter`: необов’язковий allowlist ID агентів. Пропустіть, щоб пересилати approvals для всіх агентів.
  - `sessionFilter`: необов’язкові шаблони ключів сесій (substring або regex).
  - `target`: куди надсилати approval prompts. `"dm"` (типово), `"channel"` (початкова кімната) або `"both"`.
  - Перевизначення на рівні облікового запису: `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope` керує тим, як DM Matrix групуються в sessions: `per-user` (типово) спільно використовує routed peer, тоді як `per-room` ізолює кожну DM room.
- Matrix status probes і live directory lookups використовують ту саму proxy policy, що й runtime traffic.
- Повну конфігурацію Matrix, правила targeting і приклади налаштування задокументовано в [Matrix](/uk/channels/matrix).

### Microsoft Teams

Microsoft Teams працює через extension і налаштовується в `channels.msteams`.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, політики team/channel:
      // див. /channels/msteams
    },
  },
}
```

- Основні шляхи ключів, охоплені тут: `channels.msteams`, `channels.msteams.configWrites`.
- Повну конфігурацію Teams (облікові дані, webhook, політика DM/group, перевизначення для окремих team/channel) задокументовано в [Microsoft Teams](/uk/channels/msteams).

### IRC

IRC працює через extension і налаштовується в `channels.irc`.

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

- Основні шляхи ключів, охоплені тут: `channels.irc`, `channels.irc.dmPolicy`, `channels.irc.configWrites`, `channels.irc.nickserv.*`.
- Необов’язковий `channels.irc.defaultAccount` перевизначає вибір облікового запису за замовчуванням, якщо він збігається з id налаштованого облікового запису.
- Повну конфігурацію каналу IRC (host/port/TLS/channels/allowlists/mention gating) задокументовано в [IRC](/uk/channels/irc).

### Багатообліковість (усі канали)

Запускайте кілька облікових записів на канал (кожен зі своїм `accountId`):

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

- `default` використовується, коли `accountId` пропущено (CLI + routing).
- Env tokens застосовуються лише до облікового запису **default**.
- Базові налаштування каналу застосовуються до всіх облікових записів, якщо їх не перевизначено для конкретного облікового запису.
- Використовуйте `bindings[].match.accountId`, щоб маршрутизувати кожен обліковий запис до іншого агента.
- Якщо ви додаєте не-default обліковий запис через `openclaw channels add` (або onboarding каналу), все ще перебуваючи в однообліковій top-level конфігурації каналу, OpenClaw спочатку переносить однооблікові top-level значення з областю облікового запису в map облікових записів каналу, щоб вихідний обліковий запис продовжив працювати. Для більшості каналів вони переходять у `channels.<channel>.accounts.default`; Matrix може натомість зберегти наявну відповідну іменовану/default ціль.
- Наявні bindings лише для каналу (без `accountId`) продовжують відповідати обліковому запису за замовчуванням; bindings з областю облікового запису залишаються необов’язковими.
- `openclaw doctor --fix` також виправляє змішані форми, переміщуючи однооблікові top-level значення з областю облікового запису у promoted account, вибраний для цього каналу. Більшість каналів використовують `accounts.default`; Matrix може зберегти наявну відповідну іменовану/default ціль.

### Інші канали extension

Багато каналів extension налаштовуються як `channels.<id>` і задокументовані на власних окремих сторінках каналів (наприклад, Feishu, Matrix, LINE, Nostr, Zalo, Nextcloud Talk, Synology Chat і Twitch).
Див. повний індекс каналів: [Channels](/uk/channels).

### Mention gating у групових чатах

Для групових повідомлень типово **потрібна згадка** (згадка в метаданих або безпечні regex patterns). Застосовується до групових чатів WhatsApp, Telegram, Discord, Google Chat та iMessage.

**Типи згадок:**

- **Metadata mentions**: рідні @-згадки платформи. Ігноруються в режимі self-chat WhatsApp.
- **Text patterns**: безпечні regex patterns у `agents.list[].groupChat.mentionPatterns`. Недійсні patterns і небезпечні вкладені повторення ігноруються.
- Mention gating застосовується лише тоді, коли виявлення можливе (рідні згадки або принаймні один pattern).

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

`messages.groupChat.historyLimit` задає глобальне типове значення. Канали можуть перевизначати його через `channels.<channel>.historyLimit` (або на рівні облікового запису). Встановіть `0`, щоб вимкнути.

#### Ліміти історії DM

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

Порядок визначення: перевизначення для окремого DM → значення провайдера за замовчуванням → без ліміту (зберігається все).

Підтримується для: `telegram`, `whatsapp`, `discord`, `slack`, `signal`, `imessage`, `msteams`.

#### Режим self-chat

Додайте власний номер у `allowFrom`, щоб увімкнути режим self-chat (ігнорує рідні @-згадки, відповідає лише на text patterns):

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

### Commands (обробка chat command)

```json5
{
  commands: {
    native: "auto", // реєструвати native commands, коли підтримується
    nativeSkills: "auto", // реєструвати native skill commands, коли підтримується
    text: true, // розбирати /commands у chat messages
    bash: false, // дозволити ! (alias: /bash)
    bashForegroundMs: 2000,
    config: false, // дозволити /config
    mcp: false, // дозволити /mcp
    plugins: false, // дозволити /plugins
    debug: false, // дозволити /debug
    restart: true, // дозволити /restart + інструмент restart gateway
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

<Accordion title="Деталі команд">

- Цей блок налаштовує поверхні команд. Для поточного каталогу built-in + bundled команд див. [Slash Commands](/uk/tools/slash-commands).
- Ця сторінка — це **довідник ключів конфігурації**, а не повний каталог команд. Команди, що належать каналу/плагіну, такі як QQ Bot `/bot-ping` `/bot-help` `/bot-logs`, LINE `/card`, device-pair `/pair`, memory `/dreaming`, phone-control `/phone` і Talk `/voice`, задокументовані на сторінках відповідних каналів/плагінів та в [Slash Commands](/uk/tools/slash-commands).
- Текстові команди мають бути **окремими** повідомленнями, що починаються з `/`.
- `native: "auto"` вмикає native commands для Discord/Telegram, залишає Slack вимкненим.
- `nativeSkills: "auto"` вмикає native skill commands для Discord/Telegram, залишає Slack вимкненим.
- Перевизначення на рівні каналу: `channels.discord.commands.native` (bool або `"auto"`). `false` очищає раніше зареєстровані commands.
- Перевизначайте native skill registration для каналу через `channels.<provider>.commands.nativeSkills`.
- `channels.telegram.customCommands` додає додаткові елементи меню Telegram bot.
- `bash: true` вмикає `! <cmd>` для host shell. Потрібно `tools.elevated.enabled`, а відправник має бути в `tools.elevated.allowFrom.<channel>`.
- `config: true` вмикає `/config` (читання/запис `openclaw.json`). Для клієнтів gateway `chat.send` постійні записи `/config set|unset` також потребують `operator.admin`; лише читання `/config show` залишається доступним для звичайних operator clients із правом запису.
- `mcp: true` вмикає `/mcp` для конфігурації MCP server під керуванням OpenClaw у `mcp.servers`.
- `plugins: true` вмикає `/plugins` для виявлення плагінів, встановлення та керування enable/disable.
- `channels.<provider>.configWrites` керує мутаціями конфігурації для кожного каналу (типово: true).
- Для багатооблікових каналів `channels.<provider>.accounts.<id>.configWrites` також керує записами, що націлені на цей обліковий запис (наприклад `/allowlist --config --account <id>` або `/config set channels.<provider>.accounts.<id>...`).
- `restart: false` вимикає `/restart` і дії інструмента restart gateway. Типове значення: `true`.
- `ownerAllowFrom` — це явний owner allowlist для owner-only commands/tools. Він окремий від `allowFrom`.
- `ownerDisplay: "hash"` хешує owner id у system prompt. Задайте `ownerDisplaySecret`, щоб керувати хешуванням.
- `allowFrom` задається для кожного провайдера. Якщо його встановлено, це **єдине** джерело авторизації (channel allowlists/pairing і `useAccessGroups` ігноруються).
- `useAccessGroups: false` дозволяє commands обходити політики access-group, коли `allowFrom` не встановлено.
- Мапа документації команд:
  - built-in + bundled каталог: [Slash Commands](/uk/tools/slash-commands)
  - поверхні команд, специфічні для каналу: [Channels](/uk/channels)
  - команди QQ Bot: [QQ Bot](/uk/channels/qqbot)
  - команди pairing: [Pairing](/uk/channels/pairing)
  - команда картки LINE: [LINE](/uk/channels/line)
  - dreaming пам’яті: [Dreaming](/uk/concepts/dreaming)

</Accordion>

---

## Типові значення агентів

### `agents.defaults.workspace`

Типове значення: `~/.openclaw/workspace`.

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

Необов’язковий корінь репозиторію, який показується в рядку Runtime system prompt. Якщо не встановлено, OpenClaw автоматично визначає його, піднімаючись вгору від workspace.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

Необов’язковий типовий allowlist Skills для агентів, які не задають
`agents.list[].skills`.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // успадковує github, weather
      { id: "docs", skills: ["docs-search"] }, // замінює типові
      { id: "locked-down", skills: [] }, // без skills
    ],
  },
}
```

- Пропустіть `agents.defaults.skills`, щоб типово не обмежувати Skills.
- Пропустіть `agents.list[].skills`, щоб успадкувати типові значення.
- Встановіть `agents.list[].skills: []`, щоб не дозволити жодних Skills.
- Непорожній список `agents.list[].skills` є фінальним набором для цього агента;
  він не зливається з типовими значеннями.

### `agents.defaults.skipBootstrap`

Вимикає автоматичне створення файлів bootstrap workspace (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

Керує тим, коли файли bootstrap workspace вбудовуються в system prompt. Типове значення: `"always"`.

- `"continuation-skip"`: безпечні continuation turns (після завершеної відповіді assistant) пропускають повторне вбудовування bootstrap workspace, зменшуючи розмір prompt. Запуски heartbeat і повторні спроби після compaction усе одно перебудовують контекст.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

Максимальна кількість символів у файлі bootstrap workspace перед обрізанням. Типове значення: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

Максимальна загальна кількість символів, що вбудовуються з усіх файлів bootstrap workspace. Типове значення: `150000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Керує текстом попередження, видимим агенту, коли bootstrap context обрізано.
Типове значення: `"once"`.

- `"off"`: ніколи не вбудовувати текст попередження в system prompt.
- `"once"`: вбудовувати попередження один раз для кожного унікального підпису обрізання (рекомендовано).
- `"always"`: вбудовувати попередження під час кожного запуску, якщо є обрізання.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

Максимальний розмір у пікселях для найдовшої сторони зображення в блоках image transcript/tool перед викликами провайдера.
Типове значення: `1200`.

Менші значення зазвичай зменшують використання vision-token і розмір request payload для запусків із великою кількістю скриншотів.
Більші значення зберігають більше візуальних деталей.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

Часовий пояс для контексту system prompt (не для timestamps повідомлень). Якщо не задано, використовується часовий пояс хоста.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Формат часу в system prompt. Типове значення: `auto` (налаштування ОС).

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
      params: { cacheRetention: "long" }, // глобальні типові provider params
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

- `model`: приймає або рядок (`"provider/model"`), або об’єкт (`{ primary, fallbacks }`).
  - Рядкова форма задає лише primary model.
  - Форма об’єкта задає primary плюс упорядковані failover models.
- `imageModel`: приймає або рядок (`"provider/model"`), або об’єкт (`{ primary, fallbacks }`).
  - Використовується шляхом інструмента `image` як конфігурація його vision-model.
  - Також використовується як fallback routing, коли вибрана/типова модель не може приймати image input.
- `imageGenerationModel`: приймає або рядок (`"provider/model"`), або об’єкт (`{ primary, fallbacks }`).
  - Використовується спільною можливістю генерування зображень і будь-якою майбутньою поверхнею tool/plugin, що генерує зображення.
  - Типові значення: `google/gemini-3.1-flash-image-preview` для нативного генерування зображень Gemini, `fal/fal-ai/flux/dev` для fal або `openai/gpt-image-1` для OpenAI Images.
  - Якщо ви вибираєте provider/model напряму, також налаштуйте відповідну auth/API key провайдера (наприклад `GEMINI_API_KEY` або `GOOGLE_API_KEY` для `google/*`, `OPENAI_API_KEY` для `openai/*`, `FAL_KEY` для `fal/*`).
  - Якщо пропущено, `image_generate` усе одно може вивести типове значення провайдера з auth. Спочатку він намагається використати поточного провайдера за замовчуванням, а потім інші зареєстровані провайдери image-generation у порядку provider-id.
- `musicGenerationModel`: приймає або рядок (`"provider/model"`), або об’єкт (`{ primary, fallbacks }`).
  - Використовується спільною можливістю генерації музики та built-in інструментом `music_generate`.
  - Типові значення: `google/lyria-3-clip-preview`, `google/lyria-3-pro-preview` або `minimax/music-2.5+`.
  - Якщо пропущено, `music_generate` усе одно може вивести типове значення провайдера з auth. Спочатку він намагається використати поточного провайдера за замовчуванням, а потім інші зареєстровані провайдери music-generation у порядку provider-id.
  - Якщо ви вибираєте provider/model напряму, також налаштуйте відповідну auth/API key провайдера.
- `videoGenerationModel`: приймає або рядок (`"provider/model"`), або об’єкт (`{ primary, fallbacks }`).
  - Використовується спільною можливістю генерування відео та built-in інструментом `video_generate`.
  - Типові значення: `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` або `qwen/wan2.7-r2v`.
  - Якщо пропущено, `video_generate` усе одно може вивести типове значення провайдера з auth. Спочатку він намагається використати поточного провайдера за замовчуванням, а потім інші зареєстровані провайдери video-generation у порядку provider-id.
  - Якщо ви вибираєте provider/model напряму, також налаштуйте відповідну auth/API key провайдера.
  - Bundled провайдер генерації відео Qwen підтримує до 1 вихідного відео, 1 вхідного зображення, 4 вхідних відео, тривалість 10 секунд і параметри рівня провайдера `size`, `aspectRatio`, `resolution`, `audio` і `watermark`.
- `pdfModel`: приймає або рядок (`"provider/model"`), або об’єкт (`{ primary, fallbacks }`).
  - Використовується інструментом `pdf` для маршрутизації моделей.
  - Якщо пропущено, інструмент PDF повертається до `imageModel`, а потім до визначеної session/default model.
- `pdfMaxBytesMb`: типовий ліміт розміру PDF для інструмента `pdf`, коли `maxBytesMb` не передається під час виклику.
- `pdfMaxPages`: типова максимальна кількість сторінок, що враховується режимом extraction fallback в інструменті `pdf`.
- `verboseDefault`: типовий рівень verbose для агентів. Значення: `"off"`, `"on"`, `"full"`. Типове значення: `"off"`.
- `elevatedDefault`: типовий рівень elevated-output для агентів. Значення: `"off"`, `"on"`, `"ask"`, `"full"`. Типове значення: `"on"`.
- `model.primary`: формат `provider/model` (наприклад `openai/gpt-5.4`). Якщо ви пропускаєте провайдера, OpenClaw спочатку намагається використати alias, потім унікальний configured-provider match для точного model id, і лише після цього повертається до налаштованого default provider (застаріла поведінка сумісності, тому надавайте перевагу явному `provider/model`). Якщо цей провайдер більше не надає налаштовану модель за замовчуванням, OpenClaw повертається до першої налаштованої provider/model, а не показує застаріле типове значення видаленого провайдера.
- `models`: налаштований каталог моделей і allowlist для `/model`. Кожен елемент може містити `alias` (скорочення) і `params` (специфічні для провайдера, наприклад `temperature`, `maxTokens`, `cacheRetention`, `context1m`).
- `params`: глобальні типові параметри провайдера, що застосовуються до всіх моделей. Встановлюються в `agents.defaults.params` (наприклад `{ cacheRetention: "long" }`).
- Пріоритет злиття `params` (config): `agents.defaults.params` (глобальна база) перевизначається `agents.defaults.models["provider/model"].params` (для конкретної моделі), потім `agents.list[].params` (для відповідного agent id) перевизначає за ключем. Див. [Prompt Caching](/uk/reference/prompt-caching) для подробиць.
- Конфігураційні записувачі, що змінюють ці поля (наприклад `/models set`, `/models set-image` і команди додавання/видалення fallback), зберігають канонічну об’єктну форму й по змозі зберігають наявні fallback lists.
- `maxConcurrent`: максимальна кількість паралельних запусків агентів між сесіями (кожна сесія все одно серіалізується). Типове значення: 4.

**Вбудовані shorthand-псевдоніми** (застосовуються лише тоді, коли модель є в `agents.defaults.models`):

| Alias               | Model                                  |
| ------------------- | -------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`            |
| `sonnet`            | `anthropic/claude-sonnet-4-6`          |
| `gpt`               | `openai/gpt-5.4`                       |
| `gpt-mini`          | `openai/gpt-5.4-mini`                  |
| `gpt-nano`          | `openai/gpt-5.4-nano`                  |
| `gemini`            | `google/gemini-3.1-pro-preview`        |
| `gemini-flash`      | `google/gemini-3-flash-preview`        |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

Ваші налаштовані alias-и завжди мають пріоритет над типовими.

Моделі Z.AI GLM-4.x автоматично вмикають thinking mode, якщо ви не задасте `--thinking off` або не визначите `agents.defaults.models["zai/<model>"].params.thinking` самостійно.
Моделі Z.AI типово вмикають `tool_stream` для stream-ing викликів інструментів. Встановіть `agents.defaults.models["zai/<model>"].params.tool_stream` у `false`, щоб вимкнути це.
Для моделей Anthropic Claude 4.6 типово використовується thinking `adaptive`, коли явний рівень thinking не задано.

### `agents.defaults.cliBackends`

Необов’язкові CLI backends для резервних text-only запусків (без викликів інструментів). Корисно як backup, коли API providers зазнають невдачі.

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

- CLI backends орієнтовані насамперед на текст; tools завжди вимкнені.
- Sessions підтримуються, коли задано `sessionArg`.
- Передавання зображень підтримується, коли `imageArg` приймає file paths.

### `agents.defaults.systemPromptOverride`

Замінює весь system prompt, зібраний OpenClaw, на фіксований рядок. Встановлюється на рівні default (`agents.defaults.systemPromptOverride`) або для окремого агента (`agents.list[].systemPromptOverride`). Значення для конкретного агента мають пріоритет; порожнє або таке, що складається лише з пробілів, значення ігнорується. Корисно для контрольованих експериментів із prompt.

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

Періодичні heartbeat runs.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m вимикає
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // типово: true; false прибирає секцію Heartbeat із system prompt
        lightContext: false, // типово: false; true залишає лише HEARTBEAT.md з bootstrap files workspace
        isolatedSession: false, // типово: false; true запускає кожен heartbeat у свіжій сесії (без історії розмови)
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (типово) | block
        target: "none", // типово: none | варіанти: last | whatsapp | telegram | discord | ...
        prompt: "Read HEARTBEAT.md if it exists...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
      },
    },
  },
}
```

- `every`: рядок тривалості (ms/s/m/h). Типове значення: `30m` (автентифікація API key) або `1h` (OAuth auth). Встановіть `0m`, щоб вимкнути.
- `includeSystemPromptSection`: коли false, прибирає секцію Heartbeat із system prompt і пропускає вбудовування `HEARTBEAT.md` у bootstrap context. Типове значення: `true`.
- `suppressToolErrorWarnings`: коли true, приглушує payload-и попереджень про помилки інструментів під час heartbeat runs.
- `directPolicy`: політика direct/DM delivery. `allow` (типово) дозволяє direct-target delivery. `block` приглушує direct-target delivery і виводить `reason=dm-blocked`.
- `lightContext`: коли true, heartbeat runs використовують полегшений bootstrap context і залишають лише `HEARTBEAT.md` із bootstrap files workspace.
- `isolatedSession`: коли true, кожен heartbeat запускається в новій сесії без попередньої історії розмов. Та сама схема ізоляції, що й у cron `sessionTarget: "isolated"`. Зменшує вартість токенів одного heartbeat приблизно зі ~100K до ~2-5K токенів.
- Для окремого агента: задайте `agents.list[].heartbeat`. Коли будь-який агент визначає `heartbeat`, heartbeat запускаються **лише для цих агентів**.
- Heartbeat-и виконують повні ходи агента — коротші інтервали витрачають більше токенів.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // id зареєстрованого compaction provider plugin (необов’язково)
        timeoutSeconds: 900,
        reserveTokensFloor: 24000,
        identifierPolicy: "strict", // strict | off | custom
        identifierInstructions: "Preserve deployment IDs, ticket IDs, and host:port pairs exactly.", // використовується, коли identifierPolicy=custom
        postCompactionSections: ["Session Startup", "Red Lines"], // [] вимикає повторне вбудовування
        model: "openrouter/anthropic/claude-sonnet-4-6", // необов’язкове перевизначення моделі лише для compaction
        notifyUser: true, // надсилати коротке повідомлення, коли починається compaction (типово: false)
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

- `mode`: `default` або `safeguard` (узагальнення частинами для довгих історій). Див. [Compaction](/uk/concepts/compaction).
- `provider`: id зареєстрованого compaction provider plugin. Якщо встановлено, викликається `summarize()` провайдера замість built-in LLM summarization. У разі помилки виконується fallback до built-in. Встановлення provider примушує `mode: "safeguard"`. Див. [Compaction](/uk/concepts/compaction).
- `timeoutSeconds`: максимальна кількість секунд, дозволена для однієї операції compaction перед тим, як OpenClaw перерве її. Типове значення: `900`.
- `identifierPolicy`: `strict` (типово), `off` або `custom`. `strict` додає built-in guidance для збереження opaque identifier перед compaction summarization.
- `identifierInstructions`: необов’язковий користувацький текст для збереження identifier, який використовується, коли `identifierPolicy=custom`.
- `postCompactionSections`: необов’язкові назви секцій H2/H3 з AGENTS.md, які треба повторно вбудувати після compaction. Типове значення — `["Session Startup", "Red Lines"]`; задайте `[]`, щоб вимкнути повторне вбудовування. Якщо значення не задано або явно встановлено саме цю типову пару, старі заголовки `Every Session`/`Safety` також приймаються як legacy fallback.
- `model`: необов’язкове перевизначення `provider/model-id` лише для compaction summarization. Використовуйте це, коли основна сесія має залишатися на одній моделі, а compaction summaries повинні виконуватися на іншій; якщо не задано, compaction використовує primary model сесії.
- `notifyUser`: коли `true`, надсилає користувачу коротке повідомлення, коли починається compaction (наприклад, "Compacting context..."). Типово вимкнено, щоб compaction був безшумним.
- `memoryFlush`: тихий agentic turn перед auto-compaction для збереження довготривалої пам’яті. Пропускається, коли workspace доступний лише для читання.

### `agents.defaults.contextPruning`

Обрізає **старі tool results** з in-memory context перед надсиланням до LLM. **Не** змінює історію сесії на диску.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // off | cache-ttl
        ttl: "1h", // тривалість (ms/s/m/h), одиниця за замовчуванням: хвилини
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

<Accordion title="Поведінка режиму cache-ttl">

- `mode: "cache-ttl"` вмикає проходи pruning.
- `ttl` керує тим, як часто pruning може запускатися повторно (після останнього cache touch).
- Pruning спочатку м’яко обрізає завеликі результати інструментів, а потім жорстко очищує старіші результати, якщо це потрібно.

**Soft-trim** зберігає початок + кінець і вставляє `...` посередині.

**Hard-clear** замінює весь результат інструмента placeholder-ом.

Примітки:

- Image blocks ніколи не обрізаються/очищуються.
- Співвідношення базуються на символах (приблизно), а не на точній кількості токенів.
- Якщо є менше ніж `keepLastAssistants` assistant messages, pruning пропускається.

</Accordion>

Див. [Session Pruning](/uk/concepts/session-pruning) для подробиць поведінки.

### Block streaming

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200 },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off | natural | custom (використовуйте minMs/maxMs)
    },
  },
}
```

- Канали, окрім Telegram, потребують явного `*.blockStreaming: true`, щоб увімкнути block replies.
- Перевизначення для каналу: `channels.<channel>.blockStreamingCoalesce` (і варіанти для окремих облікових записів). Для Signal/Slack/Discord/Google Chat типово `minChars: 1500`.
- `humanDelay`: випадкова пауза між block replies. `natural` = 800–2500ms. Перевизначення для агента: `agents.list[].humanDelay`.

Див. [Streaming](/uk/concepts/streaming) для поведінки та подробиць chunking.

### Typing indicators

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

- Типові значення: `instant` для direct chats/mentions, `message` для group chats без згадки.
- Перевизначення для сесії: `session.typingMode`, `session.typingIntervalSeconds`.

Див. [Typing Indicators](/uk/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Необов’язкове sandboxing для вбудованого агента. Повний посібник див. в [Sandboxing](/uk/gateway/sandboxing).

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
          // Також підтримуються SecretRef / inline contents:
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

<Accordion title="Деталі sandbox">

**Backend:**

- `docker`: локальний Docker runtime (типово)
- `ssh`: загальний віддалений runtime на базі SSH
- `openshell`: runtime OpenShell

Коли вибрано `backend: "openshell"`, налаштування, специфічні для runtime, переносяться до
`plugins.entries.openshell.config`.

**Конфігурація SSH backend:**

- `target`: SSH target у форматі `user@host[:port]`
- `command`: команда SSH client (типово: `ssh`)
- `workspaceRoot`: абсолютний віддалений root, що використовується для workspace-ів кожного scope
- `identityFile` / `certificateFile` / `knownHostsFile`: наявні локальні файли, передані в OpenSSH
- `identityData` / `certificateData` / `knownHostsData`: inline contents або SecretRef, які OpenClaw матеріалізує у temp files під час виконання
- `strictHostKeyChecking` / `updateHostKeys`: параметри політики host-key OpenSSH

**Пріоритет SSH auth:**

- `identityData` має пріоритет над `identityFile`
- `certificateData` має пріоритет над `certificateFile`
- `knownHostsData` має пріоритет над `knownHostsFile`
- Значення `*Data` на базі SecretRef визначаються з активного secrets runtime snapshot перед запуском sandbox session

**Поведінка SSH backend:**

- ініціалізує remote workspace один раз після create або recreate
- потім зберігає remote SSH workspace як канонічний
- маршрутизує `exec`, file tools і media paths через SSH
- не синхронізує remote changes назад на host автоматично
- не підтримує sandbox browser containers

**Доступ до workspace:**

- `none`: workspace sandbox для кожного scope у `~/.openclaw/sandboxes`
- `ro`: workspace sandbox у `/workspace`, workspace агента монтується лише для читання в `/agent`
- `rw`: workspace агента монтується для читання/запису в `/workspace`

**Scope:**

- `session`: container + workspace для кожної сесії
- `agent`: один container + workspace на агента (типово)
- `shared`: спільний container і workspace (без ізоляції між сесіями)

**Конфігурація plugin OpenShell:**

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
          gateway: "lab", // необов’язково
          gatewayEndpoint: "https://lab.example", // необов’язково
          policy: "strict", // необов’язковий id політики OpenShell
          providers: ["openai"], // необов’язково
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**Режим OpenShell:**

- `mirror`: ініціалізувати remote з local перед exec, синхронізувати назад після exec; local workspace лишається канонічним
- `remote`: ініціалізувати remote один раз під час створення sandbox, потім зберігати remote workspace канонічним

У режимі `remote` локальні зміни на host, зроблені поза OpenClaw, не синхронізуються в sandbox автоматично після етапу ініціалізації.
Transport здійснюється через SSH у sandbox OpenShell, але plugin володіє життєвим циклом sandbox та необов’язковою mirror sync.

**`setupCommand`** виконується один раз після створення container (через `sh -lc`). Потрібні network egress, writable root, root user.

**Типово containers використовують `network: "none"`** — встановіть `"bridge"` (або custom bridge network), якщо агенту потрібен вихід назовні.
`"host"` блокується. `"container:<id>"` типово блокується, якщо ви явно не встановите
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` (break-glass).

**Вхідні вкладення** staging-уються в `media/inbound/*` в активному workspace.

**`docker.binds`** монтує додаткові каталоги host; global і per-agent binds зливаються.

**Sandboxed browser** (`sandbox.browser.enabled`): Chromium + CDP у container. URL noVNC вбудовується в system prompt. Не потребує `browser.enabled` в `openclaw.json`.
Observer access через noVNC типово використовує VNC auth, а OpenClaw видає short-lived token URL (замість того, щоб показувати пароль у спільному URL).

- `allowHostControl: false` (типово) блокує націлення sandboxed sessions на browser host.
- `network` типово дорівнює `openclaw-sandbox-browser` (виділена bridge network). Встановлюйте `bridge` лише тоді, коли вам явно потрібне глобальне bridge connectivity.
- `cdpSourceRange` необов’язково обмежує CDP ingress на краю container до CIDR range (наприклад `172.21.0.1/32`).
- `sandbox.browser.binds` монтує додаткові каталоги host лише в container sandbox browser. Якщо задано (включно з `[]`), він замінює `docker.binds` для container browser.
- Типові параметри запуску визначені в `scripts/sandbox-browser-entrypoint.sh` і налаштовані для container hosts:
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
  - `--disable-extensions` (типово увімкнено)
  - `--disable-3d-apis`, `--disable-software-rasterizer` і `--disable-gpu`
    типово увімкнені й можуть бути вимкнені через
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0`, якщо для WebGL/3D це потрібно.
  - `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` знову вмикає extensions, якщо вони
    потрібні у вашому робочому процесі.
  - `--renderer-process-limit=2` можна змінити через
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`; установіть `0`, щоб використовувати
    типовий ліміт process Chromium.
  - плюс `--no-sandbox` і `--disable-setuid-sandbox`, коли увімкнено `noSandbox`.
  - Типові значення — це baseline container image; використовуйте custom browser image з custom
    entrypoint, щоб змінити типові значення container.

</Accordion>

Browser sandboxing і `sandbox.docker.binds` доступні лише для Docker.

Збірка образів:

```bash
scripts/sandbox-setup.sh           # основний образ sandbox
scripts/sandbox-browser-setup.sh   # необов’язковий образ browser
```

### `agents.list` (перевизначення для окремого агента)

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
        model: "anthropic/claude-opus-4-6", // або { primary, fallbacks }
        thinkingDefault: "high", // перевизначення рівня thinking для агента
        reasoningDefault: "on", // перевизначення видимості reasoning для агента
        fastModeDefault: false, // перевизначення fast mode для агента
        params: { cacheRetention: "none" }, // перевизначає matching defaults.models params за ключем
        skills: ["docs-search"], // замінює agents.defaults.skills, коли задано
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

- `id`: стабільний id агента (обов’язково).
- `default`: якщо встановлено для кількох, перший має пріоритет (реєструється попередження). Якщо не встановлено жодного, типовим є перший елемент списку.
- `model`: рядкова форма перевизначає лише `primary`; форма об’єкта `{ primary, fallbacks }` перевизначає обидва (`[]` вимикає global fallbacks). Cron jobs, що перевизначають лише `primary`, усе ще успадковують default fallbacks, якщо ви не задасте `fallbacks: []`.
- `params`: stream params для окремого агента, злиті поверх вибраного елемента моделі в `agents.defaults.models`. Використовуйте це для перевизначень на рівні агента, таких як `cacheRetention`, `temperature` або `maxTokens`, без дублювання всього каталогу моделей.
- `skills`: необов’язковий allowlist Skills для окремого агента. Якщо пропущено, агент успадковує `agents.defaults.skills`, коли воно встановлено; явний список замінює типові значення, а не зливається з ними, а `[]` означає відсутність Skills.
- `thinkingDefault`: необов’язковий типовий рівень thinking для агента (`off | minimal | low | medium | high | xhigh | adaptive`). Перевизначає `agents.defaults.thinkingDefault` для цього агента, коли не задано жодного перевизначення для повідомлення або сесії.
- `reasoningDefault`: необов’язковий типовий рівень видимості reasoning (`on | off | stream`). Застосовується, коли не задано жодного перевизначення reasoning для повідомлення або сесії.
- `fastModeDefault`: необов’язкове типове значення fast mode для агента (`true | false`). Застосовується, коли не задано жодного перевизначення fast-mode для повідомлення або сесії.
- `runtime`: необов’язковий descriptor runtime для окремого агента. Використовуйте `type: "acp"` з defaults у `runtime.acp` (`agent`, `backend`, `mode`, `cwd`), коли агент має типово використовувати ACP harness sessions.
- `identity.avatar`: шлях відносно workspace, `http(s)` URL або `data:` URI.
- `identity` виводить типові значення: `ackReaction` з `emoji`, `mentionPatterns` з `name`/`emoji`.
- `subagents.allowAgents`: allowlist id агентів для `sessions_spawn` (`["*"]` = будь-який; типово: лише той самий агент).
- Захист успадкування sandbox: якщо requester session sandboxed, `sessions_spawn` відхиляє цілі, які працювали б без sandbox.
- `subagents.requireAgentId`: коли true, блокує виклики `sessions_spawn`, у яких пропущено `agentId` (примушує явно вибирати profile; типово: false).

---

## Маршрутизація між кількома агентами

Запускайте кількох ізольованих агентів в одному Gateway. Див. [Multi-Agent](/uk/concepts/multi-agent).

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

### Поля відповідності binding

- `type` (необов’язково): `route` для звичайної маршрутизації (якщо type відсутній, типово використовується route), `acp` для постійних ACP conversation bindings.
- `match.channel` (обов’язково)
- `match.accountId` (необов’язково; `*` = будь-який обліковий запис; якщо пропущено = обліковий запис за замовчуванням)
- `match.peer` (необов’язково; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (необов’язково; залежить від каналу)
- `acp` (необов’язково; лише для `type: "acp"`): `{ mode, label, cwd, backend }`

**Детермінований порядок відповідності:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (точний, без peer/guild/team)
5. `match.accountId: "*"` (у межах усього каналу)
6. Агент за замовчуванням

У межах кожного рівня перемагає перший запис `bindings`, що збігається.

Для записів `type: "acp"` OpenClaw виконує зіставлення за точною ідентичністю conversation (`match.channel` + account + `match.peer.id`) і не використовує наведену вище tier order для route binding.

### Профілі доступу для окремих агентів

<Accordion title="Повний доступ (без sandbox)">

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

<Accordion title="Інструменти й workspace лише для читання">

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

<Accordion title="Без доступу до файлової системи (лише messaging)">

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

Див. [Multi-Agent Sandbox & Tools](/uk/tools/multi-agent-sandbox-tools) для подробиць щодо пріоритетів.

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
    parentForkMaxTokens: 100000, // пропустити parent-thread fork вище цього ліміту токенів (0 вимикає)
    maintenance: {
      mode: "warn", // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
      resetArchiveRetention: "30d", // тривалість або false
      maxDiskBytes: "500mb", // необов’язковий жорсткий бюджет
      highWaterBytes: "400mb", // необов’язкова ціль очищення
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // типове автоматичне unfocus через бездіяльність у годинах (`0` вимикає)
      maxAgeHours: 0, // типовий жорсткий максимальний вік у годинах (`0` вимикає)
    },
    mainKey: "main", // legacy (runtime завжди використовує "main")
    agentToAgent: { maxPingPongTurns: 5 },
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Деталі полів session">

- **`scope`**: базова стратегія групування sessions для контекстів групового чату.
  - `per-sender` (типово): кожен відправник отримує ізольовану session у межах контексту каналу.
  - `global`: усі учасники контексту каналу поділяють одну session (використовуйте лише тоді, коли задумано спільний контекст).
- **`dmScope`**: як групуються DM.
  - `main`: усі DM використовують спільну main session.
  - `per-peer`: ізоляція за id відправника між каналами.
  - `per-channel-peer`: ізоляція за каналом + відправником (рекомендовано для inbox-ів із багатьма користувачами).
  - `per-account-channel-peer`: ізоляція за обліковим записом + каналом + відправником (рекомендовано для багатообліковості).
- **`identityLinks`**: мапа канонічних id до peer-ів із префіксом провайдера для спільного використання сесій між каналами.
- **`reset`**: основна політика reset. `daily` скидає о `atHour` за місцевим часом; `idle` скидає після `idleMinutes`. Коли налаштовано обидва варіанти, перемагає той, чий термін настане раніше.
- **`resetByType`**: перевизначення за типом (`direct`, `group`, `thread`). Застарілий `dm` приймається як alias для `direct`.
- **`parentForkMaxTokens`**: максимальне значення `totalTokens` parent session, дозволене під час створення forked thread session (типово `100000`).
  - Якщо `totalTokens` батьківської сесії перевищує це значення, OpenClaw запускає свіжу thread session замість успадкування parent transcript history.
  - Встановіть `0`, щоб вимкнути цей захист і завжди дозволяти parent forking.
- **`mainKey`**: застаріле поле. Runtime завжди використовує `"main"` для основного кошика direct-chat.
- **`agentToAgent.maxPingPongTurns`**: максимальна кількість відповідей туди-сюди між агентами під час обміну agent-to-agent (ціле число, діапазон: `0`–`5`). `0` вимикає ping-pong chaining.
- **`sendPolicy`**: зіставлення за `channel`, `chatType` (`direct|group|channel`, із legacy alias `dm`), `keyPrefix` або `rawKeyPrefix`. Перший deny має пріоритет.
- **`maintenance`**: очищення + керування утриманням session-store.
  - `mode`: `warn` лише генерує попередження; `enforce` застосовує очищення.
  - `pruneAfter`: поріг віку для застарілих записів (типово `30d`).
  - `maxEntries`: максимальна кількість записів у `sessions.json` (типово `500`).
  - `rotateBytes`: ротація `sessions.json`, коли він перевищує цей розмір (типово `10mb`).
  - `resetArchiveRetention`: термін зберігання archive transcript `*.reset.<timestamp>`. Типово дорівнює `pruneAfter`; задайте `false`, щоб вимкнути.
  - `maxDiskBytes`: необов’язковий дисковий бюджет для каталогу sessions. У режимі `warn` просто журналює попередження; у режимі `enforce` спочатку видаляє найстаріші artifacts/sessions.
  - `highWaterBytes`: необов’язкова ціль після очищення за бюджетом. Типово `80%` від `maxDiskBytes`.
- **`threadBindings`**: глобальні типові значення для можливостей сесій, прив’язаних до thread.
  - `enabled`: основний типовий перемикач (провайдери можуть перевизначити; Discord використовує `channels.discord.threadBindings.enabled`)
  - `idleHours`: типове автоматичне unfocus через бездіяльність у годинах (`0` вимикає; провайдери можуть перевизначити)
  - `maxAgeHours`: типовий жорсткий максимальний вік у годинах (`0` вимикає; провайдери можуть перевизначити)

</Accordion>

---

## Messages

```json5
{
  messages: {
    responsePrefix: "🦞", // або "auto"
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
      debounceMs: 2000, // 0 вимикає
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Префікс відповіді

Перевизначення на рівні каналу/облікового запису: `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Порядок визначення (найспецифічніше перемагає): account → channel → global. `""` вимикає й зупиняє cascade. `"auto"` виводить `[{identity.name}]`.

**Шаблонні змінні:**

| Змінна           | Опис                    | Приклад                     |
| ---------------- | ----------------------- | --------------------------- |
| `{model}`        | Коротка назва моделі    | `claude-opus-4-6`           |
| `{modelFull}`    | Повний ідентифікатор моделі | `anthropic/claude-opus-4-6` |
| `{provider}`     | Назва провайдера        | `anthropic`                 |
| `{thinkingLevel}`| Поточний рівень thinking | `high`, `low`, `off`       |
| `{identity.name}`| Назва identity агента   | (те саме, що `"auto"`)      |

Змінні нечутливі до регістру. `{think}` — alias для `{thinkingLevel}`.

### Ack reaction

- Типово береться з `identity.emoji` активного агента, інакше `"👀"`. Встановіть `""`, щоб вимкнути.
- Перевизначення на рівні каналу: `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Порядок визначення: account → channel → `messages.ackReaction` → fallback identity.
- Scope: `group-mentions` (типово), `group-all`, `direct`, `all`.
- `removeAckAfterReply`: прибирає ack після reply у Slack, Discord і Telegram.
- `messages.statusReactions.enabled`: вмикає lifecycle status reactions у Slack, Discord і Telegram.
  У Slack і Discord, якщо не встановлено, status reactions залишаються увімкненими, коли активні ack reactions.
  У Telegram явно встановіть `true`, щоб увімкнути lifecycle status reactions.

### Inbound debounce

Об’єднує швидкі text-only messages від того самого відправника в один хід агента. Media/attachments скидаються негайно. Команди керування обходять debouncing.

### TTS (text-to-speech)

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

- `auto` керує типовим режимом auto-TTS: `off`, `always`, `inbound` або `tagged`. `/tts on|off` може перевизначити локальні prefs, а `/tts status` показує фактичний стан.
- `summaryModel` перевизначає `agents.defaults.model.primary` для auto-summary.
- `modelOverrides` типово увімкнено; `modelOverrides.allowProvider` типово має значення `false` (потрібне явне увімкнення).
- API keys повертаються до `ELEVENLABS_API_KEY`/`XI_API_KEY` і `OPENAI_API_KEY`.
- `openai.baseUrl` перевизначає endpoint TTS OpenAI. Порядок визначення: config, потім `OPENAI_TTS_BASE_URL`, потім `https://api.openai.com/v1`.
- Коли `openai.baseUrl` вказує на не-OpenAI endpoint, OpenClaw розглядає його як OpenAI-compatible TTS server і послаблює перевірку model/voice.

---

## Talk

Типові значення для режиму Talk (macOS/iOS/Android).

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

- `talk.provider` має збігатися з ключем у `talk.providers`, коли налаштовано кілька провайдерів Talk.
- Застарілі плоскі ключі Talk (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) використовуються лише для сумісності й автоматично мігрують у `talk.providers.<provider>`.
- Voice ID повертаються до `ELEVENLABS_VOICE_ID` або `SAG_VOICE_ID`.
- `providers.*.apiKey` приймає plaintext strings або об’єкти SecretRef.
- Fallback `ELEVENLABS_API_KEY` застосовується лише тоді, коли не налаштовано Talk API key.
- `providers.*.voiceAliases` дозволяє директивам Talk використовувати дружні назви.
- `silenceTimeoutMs` керує тим, скільки Talk mode чекає після тиші користувача перед надсиланням transcript. Якщо не встановлено, зберігається типове platform pause window (`700 ms на macOS і Android, 900 ms на iOS`).

---

## Tools

### Профілі інструментів

`tools.profile` задає базовий allowlist перед `tools.allow`/`tools.deny`:

Під час локального onboarding нові локальні конфігурації типово отримують `tools.profile: "coding"`, якщо значення не встановлено (наявні явні профілі зберігаються).

| Профіль     | Містить                                                                                                                        |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | лише `session_status`                                                                                                           |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate`, `video_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                                       |
| `full`      | Без обмежень (так само, як і коли не задано)                                                                                    |

### Групи інструментів

| Група             | Інструменти                                                                                                                |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`   | `exec`, `process`, `code_execution` (`bash` приймається як alias для `exec`)                                             |
| `group:fs`        | `read`, `write`, `edit`, `apply_patch`                                                                                     |
| `group:sessions`  | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status` |
| `group:memory`    | `memory_search`, `memory_get`                                                                                              |
| `group:web`       | `web_search`, `x_search`, `web_fetch`                                                                                      |
| `group:ui`        | `browser`, `canvas`                                                                                                        |
| `group:automation`| `cron`, `gateway`                                                                                                          |
| `group:messaging` | `message`                                                                                                                  |
| `group:nodes`     | `nodes`                                                                                                                    |
| `group:agents`    | `agents_list`                                                                                                              |
| `group:media`     | `image`, `image_generate`, `video_generate`, `tts`                                                                         |
| `group:openclaw`  | Усі built-in tools (без provider plugins)                                                                                  |

### `tools.allow` / `tools.deny`

Глобальна політика allow/deny для інструментів (deny має пріоритет). Нечутлива до регістру, підтримує wildcard-и `*`. Застосовується, навіть коли Docker sandbox вимкнено.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

Додатково обмежує інструменти для конкретних провайдерів або моделей. Порядок: base profile → provider profile → allow/deny.

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

Керує elevated exec access поза sandbox:

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

- Перевизначення на рівні агента (`agents.list[].tools.elevated`) може лише додатково обмежувати.
- `/elevated on|off|ask|full` зберігає стан для кожної сесії; inline directives застосовуються до одного повідомлення.
- Elevated `exec` обходить sandboxing і використовує налаштований escape path (`gateway` типово або `node`, коли exec target дорівнює `node`).

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

Перевірки безпеки loop для інструментів **типово вимкнені**. Встановіть `enabled: true`, щоб активувати виявлення.
Налаштування можна визначити глобально в `tools.loopDetection` і перевизначити для окремого агента в `agents.list[].tools.loopDetection`.

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

- `historySize`: максимальна історія викликів інструментів, яка зберігається для аналізу loop.
- `warningThreshold`: поріг повторюваних шаблонів без прогресу для попереджень.
- `criticalThreshold`: вищий поріг повторень для блокування критичних loop.
- `globalCircuitBreakerThreshold`: жорсткий поріг зупинки для будь-якого запуску без прогресу.
- `detectors.genericRepeat`: попереджати про повторні виклики одного й того самого інструмента з однаковими аргументами.
- `detectors.knownPollNoProgress`: попереджати/блокувати відомі poll tools (`process.poll`, `command_status` тощо).
- `detectors.pingPong`: попереджати/блокувати чергування парних шаблонів без прогресу.
- Якщо `warningThreshold >= criticalThreshold` або `criticalThreshold >= globalCircuitBreakerThreshold`, валідація завершується помилкою.

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // або env BRAVE_API_KEY
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // необов’язково; пропустіть для auto-detect
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

Налаштовує розуміння вхідних медіа (image/audio/video):

```json5
{
  tools: {
    media: {
      concurrency: 2,
      asyncCompletion: {
        directSend: false, // opt-in: надсилати завершені async music/video напряму в канал
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

<Accordion title="Поля запису media model">

**Запис провайдера** (`type: "provider"` або пропущено):

- `provider`: id API-провайдера (`openai`, `anthropic`, `google`/`gemini`, `groq` тощо)
- `model`: перевизначення model id
- `profile` / `preferredProfile`: вибір profile з `auth-profiles.json`

**CLI entry** (`type: "cli"`):

- `command`: executable для запуску
- `args`: шаблонізовані аргументи (підтримує `{{MediaPath}}`, `{{Prompt}}`, `{{MaxChars}}` тощо)

**Спільні поля:**

- `capabilities`: необов’язковий список (`image`, `audio`, `video`). Типові значення: `openai`/`anthropic`/`minimax` → image, `google` → image+audio+video, `groq` → audio.
- `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: перевизначення для конкретного entry.
- У разі помилки виконується fallback до наступного entry.

Provider auth дотримується стандартного порядку: `auth-profiles.json` → env vars → `models.providers.*.apiKey`.

**Поля async completion:**

- `asyncCompletion.directSend`: коли `true`, завершені async-завдання `music_generate`
  і `video_generate` спочатку намагаються доставити результат напряму в канал. Типове значення: `false`
  (legacy шлях requester-session wake/model-delivery).

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

Керує тим, на які sessions можуть націлюватися session tools (`sessions_list`, `sessions_history`, `sessions_send`).

Типове значення: `tree` (поточна session + sessions, породжені нею, наприклад subagents).

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

Примітки:

- `self`: лише поточний session key.
- `tree`: поточна session + sessions, породжені поточною session (subagents).
- `agent`: будь-яка session, що належить поточному agent id (може включати інших користувачів, якщо ви запускаєте per-sender sessions під тим самим agent id).
- `all`: будь-яка session. Націлення між агентами все одно потребує `tools.agentToAgent`.
- Sandbox clamp: коли поточна session sandboxed і `agents.defaults.sandbox.sessionToolsVisibility="spawned"`, visibility примусово встановлюється в `tree`, навіть якщо `tools.sessions.visibility="all"`.

### `tools.sessions_spawn`

Керує підтримкою inline attachments для `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // opt-in: встановіть true, щоб дозволити inline file attachments
        maxTotalBytes: 5242880, // 5 MB загалом для всіх файлів
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB на файл
        retainOnSessionKeep: false, // зберігати attachments, коли cleanup="keep"
      },
    },
  },
}
```

Примітки:

- Attachments підтримуються лише для `runtime: "subagent"`. Runtime ACP їх відхиляє.
- Файли матеріалізуються в child workspace у `.openclaw/attachments/<uuid>/` з `.manifest.json`.
- Вміст attachment автоматично редагується в transcript persistence.
- Base64 inputs перевіряються за допомогою суворих перевірок alphabet/padding і guard-а розміру до декодування.
- Дозволи на файли: `0700` для каталогів і `0600` для файлів.
- Очищення дотримується політики `cleanup`: `delete` завжди видаляє attachments; `keep` зберігає їх лише тоді, коли `retainOnSessionKeep: true`.

### `tools.experimental`

Експериментальні built-in flags для інструментів. Типово вимкнено, якщо не діє runtime-specific правило auto-enable.

```json5
{
  tools: {
    experimental: {
      planTool: true, // увімкнути експериментальний update_plan
    },
  },
}
```

Примітки:

- `planTool`: вмикає структурований інструмент `update_plan` для відстеження нетривіальної багатокрокової роботи.
- Типове значення: `false` для не-OpenAI провайдерів. Запуски OpenAI і OpenAI Codex автоматично вмикають його, якщо значення не задано; встановіть `false`, щоб вимкнути це автоувімкнення.
- Коли увімкнено, system prompt також додає guidance щодо використання, щоб модель використовувала його лише для суттєвої роботи й тримала не більше одного кроку в стані `in_progress`.

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

- `model`: типова модель для spawned sub-agents. Якщо пропущено, sub-agents успадковують модель викликача.
- `allowAgents`: типовий allowlist target agent ids для `sessions_spawn`, коли requester agent не задає власний `subagents.allowAgents` (`["*"]` = будь-який; типово: лише той самий агент).
- `runTimeoutSeconds`: типовий timeout (секунди) для `sessions_spawn`, коли виклик інструмента пропускає `runTimeoutSeconds`. `0` означає відсутність timeout.
- Політика інструментів для subagent: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Custom providers і base URLs

OpenClaw використовує built-in model catalog. Додавайте custom providers через `models.providers` у config або `~/.openclaw/agents/<agentId>/agent/models.json`.

```json5
{
  models: {
    mode: "merge", // merge (типово) | replace
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

- Використовуйте `authHeader: true` + `headers` для custom auth.
- Перевизначайте корінь agent config через `OPENCLAW_AGENT_DIR` (або `PI_CODING_AGENT_DIR`, застарілий alias environment variable).
- Пріоритет злиття для однакових provider IDs:
  - Непорожні значення `baseUrl` в agent `models.json` мають пріоритет.
  - Непорожні значення `apiKey` в agent мають пріоритет лише тоді, коли цим provider-ом не керує SecretRef у поточному контексті config/auth-profile.
  - Значення `apiKey` provider-а під керуванням SecretRef оновлюються з source markers (`ENV_VAR_NAME` для env refs, `secretref-managed` для file/exec refs) замість збереження визначених секретів.
  - Значення header provider-а під керуванням SecretRef оновлюються з source markers (`secretref-env:ENV_VAR_NAME` для env refs, `secretref-managed` для file/exec refs).
  - Порожні або відсутні значення `apiKey`/`baseUrl` в agent повертаються до `models.providers` у config.
  - Однакові значення `contextWindow`/`maxTokens` для model використовують вище значення між явним config і неявними значеннями catalog.
  - Для однакових model `contextTokens` зберігається явний runtime cap, якщо він присутній; використовуйте його, щоб обмежити фактичний context без зміни native model metadata.
  - Використовуйте `models.mode: "replace"`, коли хочете, щоб config повністю переписав `models.json`.
  - Marker persistence є source-authoritative: markers записуються з active source config snapshot (до resolution), а не з визначених runtime secret values.

### Деталі полів provider

- `models.mode`: поведінка provider catalog (`merge` або `replace`).
- `models.providers`: мапа custom provider-ів за ключем provider id.
- `models.providers.*.api`: request adapter (`openai-completions`, `openai-responses`, `anthropic-messages`, `google-generative-ai` тощо).
- `models.providers.*.apiKey`: credential провайдера (краще використовувати SecretRef/env substitution).
- `models.providers.*.auth`: стратегія auth (`api-key`, `token`, `oauth`, `aws-sdk`).
- `models.providers.*.injectNumCtxForOpenAICompat`: для Ollama + `openai-completions` додає `options.num_ctx` у requests (типово: `true`).
- `models.providers.*.authHeader`: примусово передавати credential у заголовку `Authorization`, коли це потрібно.
- `models.providers.*.baseUrl`: base URL upstream API.
- `models.providers.*.headers`: додаткові статичні заголовки для proxy/tenant routing.
- `models.providers.*.request`: transport overrides для HTTP-запитів провайдера моделей.
  - `request.headers`: додаткові заголовки (зливаються з типовими заголовками провайдера). Значення приймають SecretRef.
  - `request.auth`: перевизначення auth strategy. Режими: `"provider-default"` (використовувати built-in auth провайдера), `"authorization-bearer"` (із `token`), `"header"` (із `headerName`, `value`, необов’язковим `prefix`).
  - `request.proxy`: перевизначення HTTP proxy. Режими: `"env-proxy"` (використовувати env vars `HTTP_PROXY`/`HTTPS_PROXY`), `"explicit-proxy"` (із `url`). Обидва режими приймають необов’язковий підоб’єкт `tls`.
  - `request.tls`: перевизначення TLS для прямих з’єднань. Поля: `ca`, `cert`, `key`, `passphrase` (усі приймають SecretRef), `serverName`, `insecureSkipVerify`.
- `models.providers.*.models`: явні записи model catalog провайдера.
- `models.providers.*.models.*.contextWindow`: метадані native model context window.
- `models.providers.*.models.*.contextTokens`: необов’язковий runtime context cap. Використовуйте це, коли вам потрібен менший фактичний budget context, ніж native `contextWindow` моделі.
- `models.providers.*.models.*.compat.supportsDeveloperRole`: необов’язкова підказка сумісності. Для `api: "openai-completions"` з непорожнім ненативним `baseUrl` (host не `api.openai.com`) OpenClaw примусово встановлює це значення в `false` під час виконання. Порожній/пропущений `baseUrl` зберігає типову поведінку OpenAI.
- `models.providers.*.models.*.compat.requiresStringContent`: необов’язкова підказка сумісності для OpenAI-compatible chat endpoint-ів, що підтримують лише рядки. Коли `true`, OpenClaw сплющує чисто текстові масиви `messages[].content` у звичайні рядки перед надсиланням request.
- `plugins.entries.amazon-bedrock.config.discovery`: корінь налаштувань auto-discovery Bedrock.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`: увімкнути/вимкнути неявне discovery.
- `plugins.entries.amazon-bedrock.config.discovery.region`: регіон AWS для discovery.
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: необов’язковий фільтр provider-id для цільового discovery.
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: інтервал polling для оновлення discovery.
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: fallback context window для виявлених моделей.
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: fallback max output tokens для виявлених моделей.

### Приклади provider-ів

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

Використовуйте `cerebras/zai-glm-4.7` для Cerebras; `zai/glm-4.7` для прямого Z.AI.

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

Встановіть `OPENCODE_API_KEY` (або `OPENCODE_ZEN_API_KEY`). Використовуйте посилання `opencode/...` для каталогу Zen або `opencode-go/...` для каталогу Go. Скорочення: `openclaw onboard --auth-choice opencode-zen` або `openclaw onboard --auth-choice opencode-go`.

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

Встановіть `ZAI_API_KEY`. Приймаються alias-и `z.ai/*` і `z-ai/*`. Скорочення: `openclaw onboard --auth-choice zai-api-key`.

- Загальний endpoint: `https://api.z.ai/api/paas/v4`
- Coding endpoint (типовий): `https://api.z.ai/api/coding/paas/v4`
- Для загального endpoint визначте custom provider із перевизначенням base URL.

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

Для endpoint у Китаї: `baseUrl: "https://api.moonshot.cn/v1"` або `openclaw onboard --auth-choice moonshot-api-key-cn`.

Нативні endpoint-и Moonshot оголошують сумісність використання streaming на спільному
transport `openai-completions`, і OpenClaw визначає це за можливостями endpoint-а,
а не лише за built-in provider id.

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

Сумісний з Anthropic, built-in provider. Скорочення: `openclaw onboard --auth-choice kimi-code-api-key`.

</Accordion>

<Accordion title="Synthetic (сумісний з Anthropic)">

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

Base URL має не містити `/v1` (client Anthropic додає його сам). Скорочення: `openclaw onboard --auth-choice synthetic-api-key`.

</Accordion>

<Accordion title="MiniMax M2.7 (напряму)">

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

Встановіть `MINIMAX_API_KEY`. Скорочення:
`openclaw onboard --auth-choice minimax-global-api` або
`openclaw onboard --auth-choice minimax-cn-api`.
Каталог моделей типово містить лише M2.7.
На Anthropic-compatible streaming path OpenClaw типово вимикає thinking MiniMax,
якщо ви явно не задасте `thinking` самостійно. `/fast on` або
`params.fastMode: true` переписує `MiniMax-M2.7` на
`MiniMax-M2.7-highspeed`.

</Accordion>

<Accordion title="Локальні моделі (LM Studio)">

Див. [Local Models](/uk/gateway/local-models). Коротко: запускайте велику локальну модель через LM Studio Responses API на серйозному обладнанні; зберігайте hosted models у merged стані для fallback.

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
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // або plaintext string
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: необов’язковий allowlist лише для bundled Skills (managed/workspace skills не зачіпаються).
- `load.extraDirs`: додаткові сп