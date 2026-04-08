---
read_when:
    - 你需要精确到字段级别的配置语义或默认值
    - 你正在校验渠道、模型、Gateway 网关或工具配置块
summary: Gateway 网关配置参考，涵盖 OpenClaw 核心键、默认值以及指向专用子系统参考文档的链接
title: 配置参考
x-i18n:
    generated_at: "2026-04-08T06:34:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: fd5f69050455e610fc7c825d95f546a16aa867b8a9c0d692a48de36419b0afc2
    source_path: gateway/configuration-reference.md
    workflow: 15
---

# 配置参考

`~/.openclaw/openclaw.json` 的核心配置参考。有关面向任务的概览，请参阅 [Configuration](/zh-CN/gateway/configuration)。

本页涵盖 OpenClaw 的主要配置面，并在某个子系统拥有自己更深入的参考文档时提供外链。它**不会**尝试在一页中内联每个由渠道/插件拥有的命令目录，或每个深层内存/QMD 调整项。

代码真相来源：

- `openclaw config schema` 会打印用于校验和 Control UI 的实时 JSON Schema，并在可用时合并内置/插件/渠道元数据
- `config.schema.lookup` 返回一个按路径限定的 schema 节点，供深入查看工具使用
- `pnpm config:docs:check` / `pnpm config:docs:gen` 会根据当前 schema 表面校验配置文档基线哈希

专门的深度参考文档：

- [Memory configuration reference](/zh-CN/reference/memory-config) 适用于 `agents.defaults.memorySearch.*`、`memory.qmd.*`、`memory.citations`，以及位于 `plugins.entries.memory-core.config.dreaming` 下的 dreaming 配置
- [Slash Commands](/zh-CN/tools/slash-commands) 适用于当前内置 + 内置插件命令目录
- 各自的渠道/插件页面适用于特定渠道命令表面

配置格式为 **JSON5**（允许注释 + 尾随逗号）。所有字段都是可选的——省略时，OpenClaw 会使用安全默认值。

---

## 渠道

只要某个渠道的配置段存在，它就会自动启动（除非 `enabled: false`）。

### 私信和群组访问

所有渠道都支持私信策略和群组策略：

| 私信策略              | 行为                                                             |
| --------------------- | ---------------------------------------------------------------- |
| `pairing`（默认）     | 未知发送者会收到一次性配对码；所有者必须批准                     |
| `allowlist`           | 仅允许 `allowFrom` 中的发送者（或已配对允许存储中的发送者）      |
| `open`                | 允许所有传入私信（要求 `allowFrom: ["*"]`）                      |
| `disabled`            | 忽略所有传入私信                                                 |

| 群组策略                | 行为                                                   |
| ----------------------- | ------------------------------------------------------ |
| `allowlist`（默认）     | 仅允许与所配置允许列表匹配的群组                       |
| `open`                  | 绕过群组允许列表（但仍应用提及门控）                   |
| `disabled`              | 屏蔽所有群组/房间消息                                  |

<Note>
`channels.defaults.groupPolicy` 会在提供商的 `groupPolicy` 未设置时作为默认值。
配对码会在 1 小时后过期。待处理的私信配对请求每个渠道最多 **3 个**。
如果某个提供商配置块完全缺失（即不存在 `channels.<provider>`），运行时群组策略会回退为 `allowlist`（默认拒绝）并在启动时给出警告。
</Note>

### 渠道模型覆盖

使用 `channels.modelByChannel` 将特定渠道 ID 固定到某个模型。值接受 `provider/model` 或已配置的模型别名。当某个会话尚未有模型覆盖时（例如通过 `/model` 设置），会应用此渠道映射。

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

### 渠道默认值与心跳

使用 `channels.defaults` 为各提供商共享群组策略和心跳行为：

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

- `channels.defaults.groupPolicy`：当提供商级 `groupPolicy` 未设置时的后备群组策略。
- `channels.defaults.contextVisibility`：所有渠道的默认补充上下文可见性模式。取值：`all`（默认，包含所有引用/线程/历史上下文）、`allowlist`（仅包含允许列表发送者的上下文）、`allowlist_quote`（与 allowlist 相同，但保留显式引用/回复上下文）。每渠道覆盖：`channels.<channel>.contextVisibility`。
- `channels.defaults.heartbeat.showOk`：在心跳输出中包含健康渠道状态。
- `channels.defaults.heartbeat.showAlerts`：在心跳输出中包含降级/错误状态。
- `channels.defaults.heartbeat.useIndicator`：渲染紧凑的指示器样式心跳输出。

### WhatsApp

WhatsApp 通过 Gateway 网关的 web 渠道（Baileys Web）运行。当存在已链接会话时会自动启动。

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

<Accordion title="多账号 WhatsApp">

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

- 出站命令默认使用 `default` 账号；如果不存在，则使用按排序后的第一个已配置账号 ID。
- 可选的 `channels.whatsapp.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖该后备默认账号选择。
- 旧版单账号 Baileys 认证目录会由 `openclaw doctor` 迁移到 `whatsapp/default`。
- 每账号覆盖：`channels.whatsapp.accounts.<id>.sendReadReceipts`、`channels.whatsapp.accounts.<id>.dmPolicy`、`channels.whatsapp.accounts.<id>.allowFrom`。

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

- Bot token：`channels.telegram.botToken` 或 `channels.telegram.tokenFile`（仅接受常规文件；拒绝符号链接），默认账号还可回退到 `TELEGRAM_BOT_TOKEN`。
- 可选的 `channels.telegram.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖默认账号选择。
- 在多账号设置（2 个或以上账号 ID）中，请设置显式默认值（`channels.telegram.defaultAccount` 或 `channels.telegram.accounts.default`）以避免后备路由；若缺失或无效，`openclaw doctor` 会发出警告。
- `configWrites: false` 会阻止由 Telegram 发起的配置写入（超级群组 ID 迁移、`/config set|unset`）。
- 顶层 `bindings[]` 中 `type: "acp"` 的条目可为论坛主题配置持久 ACP 绑定（在 `match.peer.id` 中使用规范 `chatId:topic:topicId`）。字段语义共享于 [ACP Agents](/zh-CN/tools/acp-agents#channel-specific-settings)。
- Telegram 流式预览使用 `sendMessage` + `editMessageText`（在私聊和群聊中都可用）。
- 重试策略：参见 [Retry policy](/zh-CN/concepts/retry)。

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

- Token：`channels.discord.token`，默认账号还可回退到 `DISCORD_BOT_TOKEN`。
- 直接出站调用如果显式提供 Discord `token`，该次调用会使用该 token；账号重试/策略设置仍来自活动运行时快照中所选账号。
- 可选的 `channels.discord.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖默认账号选择。
- 交付目标请使用 `user:<id>`（私信）或 `channel:<id>`（服务器频道）；裸数字 ID 会被拒绝。
- 服务器 slug 为小写，空格会替换为 `-`；频道键使用 slug 化名称（不带 `#`）。优先使用服务器 ID。
- 默认会忽略机器人自己写的消息。`allowBots: true` 会启用它们；使用 `allowBots: "mentions"` 则只接受提及机器人的机器人消息（仍会过滤自己的消息）。
- `channels.discord.guilds.<id>.ignoreOtherMentions`（以及频道级覆盖）会丢弃提及了其他用户或角色但未提及机器人的消息（不包括 @everyone/@here）。
- `maxLinesPerMessage`（默认 17）会在消息行数过高时拆分，即使未超过 2000 字符。
- `channels.discord.threadBindings` 控制 Discord 线程绑定路由：
  - `enabled`：线程绑定会话功能的 Discord 覆盖（`/focus`、`/unfocus`、`/agents`、`/session idle`、`/session max-age`，以及绑定交付/路由）
  - `idleHours`：Discord 侧按不活动自动取消聚焦的小时数覆盖（`0` 表示禁用）
  - `maxAgeHours`：Discord 侧硬性最大年龄小时数覆盖（`0` 表示禁用）
  - `spawnSubagentSessions`：为 `sessions_spawn({ thread: true })` 自动创建/绑定线程的选择加入开关
- 顶层 `bindings[]` 中 `type: "acp"` 的条目可为频道和线程配置持久 ACP 绑定（在 `match.peer.id` 中使用频道/线程 ID）。字段语义共享于 [ACP Agents](/zh-CN/tools/acp-agents#channel-specific-settings)。
- `channels.discord.ui.components.accentColor` 设置 Discord components v2 容器的强调色。
- `channels.discord.voice` 启用 Discord 语音频道对话，以及可选的自动加入 + TTS 覆盖。
- `channels.discord.voice.daveEncryption` 和 `channels.discord.voice.decryptionFailureTolerance` 会透传给 `@discordjs/voice` 的 DAVE 选项（默认分别为 `true` 和 `24`）。
- OpenClaw 还会在重复解密失败后通过离开/重新加入语音会话来尝试恢复语音接收。
- `channels.discord.streaming` 是规范的流式模式键。旧版 `streamMode` 和布尔型 `streaming` 值会被自动迁移。
- `channels.discord.autoPresence` 会将运行时可用性映射到机器人在线状态（healthy => online，degraded => idle，exhausted => dnd），并允许可选状态文本覆盖。
- `channels.discord.dangerouslyAllowNameMatching` 会重新启用可变名称/tag 匹配（紧急兼容模式）。
- `channels.discord.execApprovals`：Discord 原生 exec 审批交付与审批人授权。
  - `enabled`：`true`、`false` 或 `"auto"`（默认）。在 auto 模式下，当可从 `approvers` 或 `commands.ownerAllowFrom` 解析到审批人时，exec 审批会激活。
  - `approvers`：允许批准 exec 请求的 Discord 用户 ID。省略时回退到 `commands.ownerAllowFrom`。
  - `agentFilter`：可选智能体 ID 允许列表。省略则转发所有智能体的审批请求。
  - `sessionFilter`：可选会话键模式（子串或正则）。
  - `target`：将审批提示发送到何处。`"dm"`（默认）发送到审批人私信，`"channel"` 发送到原始频道，`"both"` 发送到两处。当目标包含 `"channel"` 时，按钮仅对已解析出的审批人可用。
  - `cleanupAfterResolve`：为 `true` 时，在批准、拒绝或超时后删除审批私信。

**反应通知模式：** `off`（无）、`own`（机器人自己的消息，默认）、`all`（所有消息）、`allowlist`（来自 `guilds.<id>.users` 且适用于所有消息）。

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

- 服务账号 JSON：内联（`serviceAccount`）或基于文件（`serviceAccountFile`）。
- 也支持服务账号 SecretRef（`serviceAccountRef`）。
- 环境变量回退：`GOOGLE_CHAT_SERVICE_ACCOUNT` 或 `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE`。
- 交付目标请使用 `spaces/<spaceId>` 或 `users/<userId>`。
- `channels.googlechat.dangerouslyAllowNameMatching` 会重新启用可变电子邮件主体匹配（紧急兼容模式）。

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

- **Socket mode** 需要同时提供 `botToken` 和 `appToken`（默认账号的环境变量回退为 `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN`）。
- **HTTP mode** 需要 `botToken` 加 `signingSecret`（可位于根级或每账号级）。
- `botToken`、`appToken`、`signingSecret` 和 `userToken` 接受明文字符串或 SecretRef 对象。
- Slack 账号快照会暴露按凭证划分的来源/状态字段，例如 `botTokenSource`、`botTokenStatus`、`appTokenStatus`，以及在 HTTP mode 下的 `signingSecretStatus`。`configured_unavailable` 表示账号通过 SecretRef 配置，但当前命令/运行时路径无法解析该密钥值。
- `configWrites: false` 会阻止由 Slack 发起的配置写入。
- 可选的 `channels.slack.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖默认账号选择。
- `channels.slack.streaming.mode` 是规范的 Slack 流式模式键。`channels.slack.streaming.nativeTransport` 控制 Slack 的原生流式传输。旧版 `streamMode`、布尔型 `streaming` 和 `nativeStreaming` 值会自动迁移。
- 交付目标请使用 `user:<id>`（私信）或 `channel:<id>`。

**反应通知模式：** `off`、`own`（默认）、`all`、`allowlist`（来自 `reactionAllowlist`）。

**线程会话隔离：** `thread.historyScope` 可为每线程（默认）或共享到频道。`thread.inheritParent` 会把父频道转录复制到新线程。

- Slack 原生流式传输以及 Slack 助手样式的 “is typing...” 线程状态需要回复线程目标。顶级私信默认保持非线程，因此它们会使用 `typingReaction` 或普通交付，而不是线程样式预览。
- `typingReaction` 会在回复运行期间给传入 Slack 消息添加一个临时 reaction，并在完成后移除。请使用 Slack emoji 简码，如 `"hourglass_flowing_sand"`。
- `channels.slack.execApprovals`：Slack 原生 exec 审批交付与审批人授权。Schema 与 Discord 相同：`enabled`（`true`/`false`/`"auto"`）、`approvers`（Slack 用户 ID）、`agentFilter`、`sessionFilter` 和 `target`（`"dm"`、`"channel"` 或 `"both"`）。

| 操作组       | 默认值   | 说明                 |
| ------------ | -------- | -------------------- |
| reactions    | 启用     | 添加 reaction + 列出 reactions |
| messages     | 启用     | 读取/发送/编辑/删除  |
| pins         | 启用     | 置顶/取消置顶/列出   |
| memberInfo   | 启用     | 成员信息             |
| emojiList    | 启用     | 自定义 emoji 列表    |

### Mattermost

Mattermost 作为插件提供：`openclaw plugins install @openclaw/mattermost`。

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

聊天模式：`oncall`（在 @ 提及时回复，默认）、`onmessage`（每条消息都回复）、`onchar`（以触发前缀开头的消息）。

启用 Mattermost 原生命令时：

- `commands.callbackPath` 必须是路径（例如 `/api/channels/mattermost/command`），不能是完整 URL。
- `commands.callbackUrl` 必须解析到 OpenClaw Gateway 网关端点，并且 Mattermost 服务器能够访问。
- 原生斜杠命令回调使用 Mattermost 在斜杠命令注册期间返回的每命令 token 进行认证。如果注册失败，或没有激活任何命令，OpenClaw 会使用 `Unauthorized: invalid command token.` 拒绝回调。
- 对于私有/tailnet/内部回调主机，Mattermost 可能需要将回调主机/域名加入 `ServiceSettings.AllowedUntrustedInternalConnections`。
  请使用主机/域名值，而不是完整 URL。
- `channels.mattermost.configWrites`：允许或拒绝由 Mattermost 发起的配置写入。
- `channels.mattermost.requireMention`：要求在频道中先 `@mention` 才回复。
- `channels.mattermost.groups.<channelId>.requireMention`：按频道设置的提及门控覆盖（`"*"` 表示默认）。
- 可选的 `channels.mattermost.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖默认账号选择。

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

**反应通知模式：** `off`、`own`（默认）、`all`、`allowlist`（来自 `reactionAllowlist`）。

- `channels.signal.account`：将渠道启动固定到特定的 Signal 账号身份。
- `channels.signal.configWrites`：允许或拒绝由 Signal 发起的配置写入。
- 可选的 `channels.signal.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖默认账号选择。

### BlueBubbles

BlueBubbles 是推荐的 iMessage 路径（由插件支持，配置位于 `channels.bluebubbles` 下）。

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

- 此处涵盖的核心键路径：`channels.bluebubbles`、`channels.bluebubbles.dmPolicy`。
- 可选的 `channels.bluebubbles.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖默认账号选择。
- 顶层 `bindings[]` 中 `type: "acp"` 的条目可将 BlueBubbles 对话绑定到持久 ACP 会话。在 `match.peer.id` 中使用 BlueBubbles handle 或目标字符串（`chat_id:*`、`chat_guid:*`、`chat_identifier:*`）。共享字段语义： [ACP Agents](/zh-CN/tools/acp-agents#channel-specific-settings)。
- 完整 BlueBubbles 渠道配置见 [BlueBubbles](/zh-CN/channels/bluebubbles)。

### iMessage

OpenClaw 会生成 `imsg rpc`（基于 stdio 的 JSON-RPC）。不需要守护进程或端口。

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

- 可选的 `channels.imessage.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖默认账号选择。

- 需要对 Messages DB 授予 Full Disk Access。
- 优先使用 `chat_id:<id>` 目标。使用 `imsg chats --limit 20` 列出聊天。
- `cliPath` 可以指向 SSH 包装器；设置 `remoteHost`（`host` 或 `user@host`）用于通过 SCP 获取附件。
- `attachmentRoots` 和 `remoteAttachmentRoots` 会限制传入附件路径（默认：`/Users/*/Library/Messages/Attachments`）。
- SCP 使用严格主机密钥检查，因此请确保中继主机密钥已存在于 `~/.ssh/known_hosts` 中。
- `channels.imessage.configWrites`：允许或拒绝由 iMessage 发起的配置写入。
- 顶层 `bindings[]` 中 `type: "acp"` 的条目可将 iMessage 对话绑定到持久 ACP 会话。在 `match.peer.id` 中使用标准化 handle 或显式聊天目标（`chat_id:*`、`chat_guid:*`、`chat_identifier:*`）。共享字段语义： [ACP Agents](/zh-CN/tools/acp-agents#channel-specific-settings)。

<Accordion title="iMessage SSH 包装器示例">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix 由扩展支持，配置位于 `channels.matrix` 下。

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

- Token 认证使用 `accessToken`；密码认证使用 `userId` + `password`。
- `channels.matrix.proxy` 会将 Matrix HTTP 流量路由到显式 HTTP(S) 代理。命名账号可通过 `channels.matrix.accounts.<id>.proxy` 覆盖它。
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` 允许私有/内部 homeserver。`proxy` 和这个网络选择加入是彼此独立的控制项。
- `channels.matrix.defaultAccount` 会在多账号设置中选择首选账号。
- `channels.matrix.autoJoin` 默认为 `off`，因此受邀房间和新的私信式邀请会被忽略，直到你设置 `autoJoin: "allowlist"` 配合 `autoJoinAllowlist`，或设置 `autoJoin: "always"`。
- `channels.matrix.execApprovals`：Matrix 原生 exec 审批交付与审批人授权。
  - `enabled`：`true`、`false` 或 `"auto"`（默认）。在 auto 模式下，当可从 `approvers` 或 `commands.ownerAllowFrom` 解析到审批人时，exec 审批会激活。
  - `approvers`：允许批准 exec 请求的 Matrix 用户 ID（例如 `@owner:example.org`）。
  - `agentFilter`：可选智能体 ID 允许列表。省略则转发所有智能体的审批请求。
  - `sessionFilter`：可选会话键模式（子串或正则）。
  - `target`：将审批提示发送到何处。`"dm"`（默认）、`"channel"`（源房间）或 `"both"`。
  - 每账号覆盖：`channels.matrix.accounts.<id>.execApprovals`。
- `channels.matrix.dm.sessionScope` 控制 Matrix 私信如何分组成会话：`per-user`（默认）按路由后的对端共享，`per-room` 则将每个私信房间隔离。
- Matrix 状态探测和实时目录查找使用与运行时流量相同的代理策略。
- 完整的 Matrix 配置、目标规则和设置示例见 [Matrix](/zh-CN/channels/matrix)。

### Microsoft Teams

Microsoft Teams 由扩展支持，配置位于 `channels.msteams` 下。

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

- 此处涵盖的核心键路径：`channels.msteams`、`channels.msteams.configWrites`。
- 完整 Teams 配置（凭证、webhook、私信/群组策略、按团队/频道覆盖）见 [Microsoft Teams](/zh-CN/channels/msteams)。

### IRC

IRC 由扩展支持，配置位于 `channels.irc` 下。

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

- 此处涵盖的核心键路径：`channels.irc`、`channels.irc.dmPolicy`、`channels.irc.configWrites`、`channels.irc.nickserv.*`。
- 可选的 `channels.irc.defaultAccount` 会在其匹配某个已配置账号 ID 时覆盖默认账号选择。
- 完整 IRC 渠道配置（主机/端口/TLS/频道/允许列表/提及门控）见 [IRC](/zh-CN/channels/irc)。

### 多账号（所有渠道）

为每个渠道运行多个账号（每个账号都有自己的 `accountId`）：

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

- 当省略 `accountId` 时使用 `default`（CLI + 路由）。
- 环境变量 token 仅适用于 **default** 账号。
- 基础渠道设置会应用于所有账号，除非按账号覆盖。
- 使用 `bindings[].match.accountId` 将每个账号路由到不同智能体。
- 如果你通过 `openclaw channels add`（或渠道新手引导）添加非默认账号，而当前仍是单账号顶级渠道配置，OpenClaw 会先把按账号限定的顶级单账号值提升到渠道账号映射中，这样原始账号还能继续工作。大多数渠道会把它们移到 `channels.<channel>.accounts.default`；Matrix 则可以保留现有匹配的命名/默认目标。
- 现有仅渠道级绑定（无 `accountId`）会继续匹配默认账号；按账号绑定仍然是可选的。
- `openclaw doctor --fix` 也会通过把按账号限定的顶级单账号值移动到该渠道所选的提升账号中来修复混合形状。大多数渠道使用 `accounts.default`；Matrix 则可以保留现有匹配的命名/默认目标。

### 其他扩展渠道

许多扩展渠道都配置为 `channels.<id>`，并记录在各自专属的渠道页面中（例如 Feishu、Matrix、LINE、Nostr、Zalo、Nextcloud Talk、Synology Chat 和 Twitch）。
查看完整渠道索引：[Channels](/zh-CN/channels)。

### 群聊提及门控

群组消息默认**要求提及**（元数据提及或安全正则模式）。适用于 WhatsApp、Telegram、Discord、Google Chat 和 iMessage 群聊。

**提及类型：**

- **元数据提及**：平台原生 @ 提及。在 WhatsApp 自聊模式下会被忽略。
- **文本模式**：位于 `agents.list[].groupChat.mentionPatterns` 中的安全正则模式。无效模式和不安全的嵌套重复会被忽略。
- 仅在能够检测提及时（原生提及或至少有一个模式）才会强制执行提及门控。

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

`messages.groupChat.historyLimit` 设置全局默认值。各渠道可通过 `channels.<channel>.historyLimit`（或按账号）覆盖。设置为 `0` 可禁用。

#### 私信历史限制

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

解析顺序：按私信覆盖 → 提供商默认值 → 无限制（全部保留）。

支持：`telegram`、`whatsapp`、`discord`、`slack`、`signal`、`imessage`、`msteams`。

#### 自聊模式

将你自己的号码包含在 `allowFrom` 中即可启用自聊模式（忽略原生 @ 提及，只响应文本模式）：

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

### 命令（聊天命令处理）

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

<Accordion title="命令详情">

- 此配置块用于配置命令表面。关于当前内置 + 内置插件命令目录，请参见 [Slash Commands](/zh-CN/tools/slash-commands)。
- 本页是**配置键参考**，不是完整命令目录。像 QQ Bot `/bot-ping` `/bot-help` `/bot-logs`、LINE `/card`、device-pair `/pair`、memory `/dreaming`、phone-control `/phone` 以及 Talk `/voice` 这样的渠道/插件专属命令，记录在它们各自的渠道/插件页面以及 [Slash Commands](/zh-CN/tools/slash-commands) 中。
- 文本命令必须是带前导 `/` 的**独立**消息。
- `native: "auto"` 会为 Discord/Telegram 打开原生命令，而 Slack 保持关闭。
- `nativeSkills: "auto"` 会为 Discord/Telegram 打开原生 skill 命令，而 Slack 保持关闭。
- 可按渠道覆盖：`channels.discord.commands.native`（布尔值或 `"auto"`）。`false` 会清除先前注册的命令。
- 使用 `channels.<provider>.commands.nativeSkills` 覆盖按渠道的原生 skill 注册。
- `channels.telegram.customCommands` 会添加额外的 Telegram 机器人菜单项。
- `bash: true` 启用用于主机 shell 的 `! <cmd>`。要求 `tools.elevated.enabled` 为真，且发送者位于 `tools.elevated.allowFrom.<channel>` 中。
- `config: true` 启用 `/config`（读取/写入 `openclaw.json`）。对于 Gateway 网关 `chat.send` 客户端，持久化 `/config set|unset` 写入还需要 `operator.admin`；只读的 `/config show` 对普通具写权限的 operator 客户端仍然可用。
- `mcp: true` 启用 `/mcp`，用于管理位于 `mcp.servers` 下的 OpenClaw 托管 MCP 服务器配置。
- `plugins: true` 启用 `/plugins`，用于插件发现、安装和启用/禁用控制。
- `channels.<provider>.configWrites` 按渠道控制配置变更（默认：true）。
- 对于多账号渠道，`channels.<provider>.accounts.<id>.configWrites` 也会控制以该账号为目标的写入（例如 `/allowlist --config --account <id>` 或 `/config set channels.<provider>.accounts.<id>...`）。
- `restart: false` 会禁用 `/restart` 和 Gateway 网关重启工具操作。默认：`true`。
- `ownerAllowFrom` 是显式的 owner 允许列表，用于仅 owner 可用的命令/工具。它独立于 `allowFrom`。
- `ownerDisplay: "hash"` 会在系统提示中对 owner ID 做哈希。设置 `ownerDisplaySecret` 以控制哈希。
- `allowFrom` 是按提供商的。设置后，它会成为**唯一**授权来源（渠道允许列表/配对 和 `useAccessGroups` 会被忽略）。
- `useAccessGroups: false` 允许在未设置 `allowFrom` 时，命令绕过访问组策略。
- 命令文档映射：
  - 内置 + 内置插件目录：[Slash Commands](/zh-CN/tools/slash-commands)
  - 特定渠道命令表面：[Channels](/zh-CN/channels)
  - QQ Bot 命令：[QQ Bot](/zh-CN/channels/qqbot)
  - 配对命令：[Pairing](/zh-CN/channels/pairing)
  - LINE card 命令：[LINE](/zh-CN/channels/line)
  - memory dreaming：[Dreaming](/zh-CN/concepts/dreaming)

</Accordion>

---

## 智能体默认值

### `agents.defaults.workspace`

默认值：`~/.openclaw/workspace`。

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### `agents.defaults.repoRoot`

可选的仓库根目录，会显示在系统提示的 Runtime 行中。如果未设置，OpenClaw 会从工作区向上遍历自动检测。

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

对于未设置 `agents.list[].skills` 的智能体，可选的默认 Skills 允许列表。

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

- 省略 `agents.defaults.skills` 可使默认 Skills 不受限制。
- 省略 `agents.list[].skills` 以继承默认值。
- 设置 `agents.list[].skills: []` 表示没有 Skills。
- 非空的 `agents.list[].skills` 列表就是该智能体的最终集合；它不会与默认值合并。

### `agents.defaults.skipBootstrap`

禁用自动创建工作区 bootstrap 文件（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、`BOOTSTRAP.md`）。

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.contextInjection`

控制何时将工作区 bootstrap 文件注入系统提示。默认值：`"always"`。

- `"continuation-skip"`：安全续接轮次（在 assistant 完成回复之后）会跳过工作区 bootstrap 的再次注入，从而减小提示大小。心跳运行和压缩后的重试仍会重建上下文。

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

### `agents.defaults.bootstrapMaxChars`

每个工作区 bootstrap 文件在截断前允许的最大字符数。默认值：`20000`。

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

### `agents.defaults.bootstrapTotalMaxChars`

跨所有工作区 bootstrap 文件注入的总最大字符数。默认值：`150000`。

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 150000 } },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

控制在 bootstrap 上下文被截断时，对智能体可见的警告文本。默认值：`"once"`。

- `"off"`：永不将警告文本注入系统提示。
- `"once"`：对每个唯一的截断签名只注入一次警告（推荐）。
- `"always"`：只要存在截断，每次运行都注入警告。

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "once" } }, // off | once | always
}
```

### `agents.defaults.imageMaxDimensionPx`

在发起提供商调用前，转录/工具图片块中图片最长边的最大像素尺寸。
默认值：`1200`。

较低的值通常会减少视觉 token 用量和请求负载大小，适合截图很多的运行。
较高的值会保留更多视觉细节。

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.userTimezone`

系统提示上下文使用的时区（不是消息时间戳）。会回退到主机时区。

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

系统提示中的时间格式。默认值：`auto`（操作系统偏好）。

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

- `model`：可接受字符串（`"provider/model"`）或对象（`{ primary, fallbacks }`）。
  - 字符串形式只设置主模型。
  - 对象形式设置主模型以及有序故障切换模型。
- `imageModel`：可接受字符串（`"provider/model"`）或对象（`{ primary, fallbacks }`）。
  - 用于 `image` 工具路径，作为其视觉模型配置。
  - 当所选/默认模型无法接受图片输入时，也会用于后备路由。
- `imageGenerationModel`：可接受字符串（`"provider/model"`）或对象（`{ primary, fallbacks }`）。
  - 用于共享图像生成能力，以及未来任何生成图像的工具/插件表面。
  - 典型值：原生 Gemini 图像生成使用 `google/gemini-3.1-flash-image-preview`，fal 使用 `fal/fal-ai/flux/dev`，OpenAI Images 使用 `openai/gpt-image-1`。
  - 如果你直接选择某个提供商/模型，也要配置匹配的提供商 auth/API key（例如 `google/*` 需要 `GEMINI_API_KEY` 或 `GOOGLE_API_KEY`，`openai/*` 需要 `OPENAI_API_KEY`，`fal/*` 需要 `FAL_KEY`）。
  - 如果省略，`image_generate` 仍可推断出一个带 auth 的提供商默认值。它会先尝试当前默认提供商，再按提供商 ID 顺序尝试剩余已注册的图像生成提供商。
- `musicGenerationModel`：可接受字符串（`"provider/model"`）或对象（`{ primary, fallbacks }`）。
  - 用于共享音乐生成能力和内置 `music_generate` 工具。
  - 典型值：`google/lyria-3-clip-preview`、`google/lyria-3-pro-preview` 或 `minimax/music-2.5+`。
  - 如果省略，`music_generate` 仍可推断出一个带 auth 的提供商默认值。它会先尝试当前默认提供商，再按提供商 ID 顺序尝试剩余已注册的音乐生成提供商。
  - 如果你直接选择某个提供商/模型，也要配置匹配的提供商 auth/API key。
- `videoGenerationModel`：可接受字符串（`"provider/model"`）或对象（`{ primary, fallbacks }`）。
  - 用于共享视频生成能力和内置 `video_generate` 工具。
  - 典型值：`qwen/wan2.6-t2v`、`qwen/wan2.6-i2v`、`qwen/wan2.6-r2v`、`qwen/wan2.6-r2v-flash` 或 `qwen/wan2.7-r2v`。
  - 如果省略，`video_generate` 仍可推断出一个带 auth 的提供商默认值。它会先尝试当前默认提供商，再按提供商 ID 顺序尝试剩余已注册的视频生成提供商。
  - 如果你直接选择某个提供商/模型，也要配置匹配的提供商 auth/API key。
  - 内置的 Qwen 视频生成提供商目前最多支持 1 个输出视频、1 张输入图像、4 个输入视频、10 秒时长，以及提供商级 `size`、`aspectRatio`、`resolution`、`audio` 和 `watermark` 选项。
- `pdfModel`：可接受字符串（`"provider/model"`）或对象（`{ primary, fallbacks }`）。
  - 用于 `pdf` 工具的模型路由。
  - 如果省略，PDF 工具会回退到 `imageModel`，再回退到已解析的会话/默认模型。
- `pdfMaxBytesMb`：当调用时未传入 `maxBytesMb` 时，`pdf` 工具使用的默认 PDF 大小限制。
- `pdfMaxPages`：`pdf` 工具在提取后备模式中默认考虑的最大页数。
- `verboseDefault`：智能体默认详细级别。取值：`"off"`、`"on"`、`"full"`。默认：`"off"`。
- `elevatedDefault`：智能体默认 elevated 输出级别。取值：`"off"`、`"on"`、`"ask"`、`"full"`。默认：`"on"`。
- `model.primary`：格式为 `provider/model`（例如 `openai/gpt-5.4`）。如果省略提供商，OpenClaw 会先尝试别名，再尝试某个唯一已配置提供商中精确匹配该模型 ID 的项，最后才会回退到已配置默认提供商（这是已弃用的兼容行为，因此推荐显式使用 `provider/model`）。如果该提供商不再暴露所配置的默认模型，OpenClaw 会回退到第一个已配置的提供商/模型，而不是暴露过时的已移除提供商默认值。
- `models`：为 `/model` 配置的模型目录和允许列表。每个条目可包含 `alias`（快捷方式）和 `params`（提供商特定参数，例如 `temperature`、`maxTokens`、`cacheRetention`、`context1m`）。
- `params`：应用于所有模型的全局默认提供商参数。设置在 `agents.defaults.params`（例如 `{ cacheRetention: "long" }`）。
- `params` 合并优先级（配置）：`agents.defaults.params`（全局基础）会被 `agents.defaults.models["provider/model"].params`（按模型）覆盖，再由 `agents.list[].params`（匹配智能体 ID）按键覆盖。详见 [Prompt Caching](/zh-CN/reference/prompt-caching)。
- 变更这些字段的配置写入器（例如 `/models set`、`/models set-image` 以及后备 add/remove 命令）会保存规范对象形式，并在可能时保留现有后备列表。
- `maxConcurrent`：跨会话的最大并行智能体运行数（每个会话内部仍然串行）。默认：4。

**内置别名简写**（仅在模型存在于 `agents.defaults.models` 中时适用）：

| 别名                | 模型                                   |
| ------------------- | -------------------------------------- |
| `opus`              | `anthropic/claude-opus-4-6`            |
| `sonnet`            | `anthropic/claude-sonnet-4-6`          |
| `gpt`               | `openai/gpt-5.4`                       |
| `gpt-mini`          | `openai/gpt-5.4-mini`                  |
| `gpt-nano`          | `openai/gpt-5.4-nano`                  |
| `gemini`            | `google/gemini-3.1-pro-preview`        |
| `gemini-flash`      | `google/gemini-3-flash-preview`        |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite-preview` |

你自己配置的别名始终优先于默认值。

Z.AI GLM-4.x 模型会自动启用 thinking 模式，除非你设置 `--thinking off` 或自行定义 `agents.defaults.models["zai/<model>"].params.thinking`。
Z.AI 模型默认启用 `tool_stream` 进行工具调用流式传输。将 `agents.defaults.models["zai/<model>"].params.tool_stream` 设为 `false` 可禁用。
Anthropic Claude 4.6 模型在未设置显式 thinking 级别时，默认使用 `adaptive` thinking。

### `agents.defaults.cliBackends`

文本专用后备运行（无工具调用）的可选 CLI 后端。当 API 提供商失败时可作为备份。

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

- CLI 后端以文本为主；工具始终禁用。
- 设置了 `sessionArg` 时支持会话。
- 当 `imageArg` 接受文件路径时支持图片透传。

### `agents.defaults.systemPromptOverride`

用固定字符串替换整个由 OpenClaw 组装的系统提示。可设置在默认级（`agents.defaults.systemPromptOverride`）或按智能体设置（`agents.list[].systemPromptOverride`）。按智能体的值优先；空字符串或仅空白字符的值会被忽略。适合受控的提示实验。

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

周期性心跳运行。

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

- `every`：时长字符串（ms/s/m/h）。默认：`30m`（API key auth）或 `1h`（OAuth auth）。设为 `0m` 可禁用。
- `includeSystemPromptSection`：设为 false 时，会从系统提示中省略 Heartbeat 部分，并跳过将 `HEARTBEAT.md` 注入 bootstrap 上下文。默认：`true`。
- `suppressToolErrorWarnings`：为 true 时，会在心跳运行中抑制工具错误警告负载。
- `directPolicy`：直接/私信交付策略。`allow`（默认）允许直接目标交付。`block` 会抑制直接目标交付，并发出 `reason=dm-blocked`。
- `lightContext`：为 true 时，心跳运行使用轻量 bootstrap 上下文，并只保留工作区 bootstrap 文件中的 `HEARTBEAT.md`。
- `isolatedSession`：为 true 时，每次心跳运行都在一个全新会话中执行，不带任何先前对话历史。与 cron `sessionTarget: "isolated"` 相同的隔离模式。可将每次心跳的 token 成本从约 100K 降到约 2-5K。
- 按智能体设置：使用 `agents.list[].heartbeat`。当任意智能体定义了 `heartbeat` 时，**只有这些智能体**会运行心跳。
- 心跳会运行完整智能体轮次——更短的间隔会消耗更多 token。

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

- `mode`：`default` 或 `safeguard`（针对长历史的分块摘要）。参见 [Compaction](/zh-CN/concepts/compaction)。
- `provider`：已注册 compaction 提供商插件的 ID。设置后，会调用该提供商的 `summarize()` 而不是内置 LLM 摘要；失败时回退到内置实现。设置该提供商会强制使用 `mode: "safeguard"`。参见 [Compaction](/zh-CN/concepts/compaction)。
- `timeoutSeconds`：OpenClaw 放弃单次 compaction 操作前允许的最大秒数。默认：`900`。
- `identifierPolicy`：`strict`（默认）、`off` 或 `custom`。`strict` 会在 compaction 摘要时预置内置的不透明标识符保留指导。
- `identifierInstructions`：当 `identifierPolicy=custom` 时使用的可选自定义标识符保留文本。
- `postCompactionSections`：可选的 AGENTS.md H2/H3 章节名，在 compaction 后重新注入。默认值为 `["Session Startup", "Red Lines"]`；设为 `[]` 可禁用重新注入。当未设置或显式设置为该默认对时，旧版 `Every Session`/`Safety` 标题也会作为兼容后备被接受。
- `model`：仅用于 compaction 摘要的可选 `provider/model-id` 覆盖。当你希望主会话继续使用一个模型，但 compaction 摘要改用另一个模型时使用；未设置时，compaction 使用会话的主模型。
- `notifyUser`：为 `true` 时，在 compaction 开始时向用户发送简短提示（例如 “Compacting context...”）。默认禁用，以保持 compaction 静默。
- `memoryFlush`：在自动 compaction 之前进行静默的智能体轮次以存储持久记忆。工作区只读时会跳过。

### `agents.defaults.contextPruning`

在将上下文发送给 LLM 之前，从内存上下文中裁剪**旧工具结果**。**不会**修改磁盘上的会话历史。

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

<Accordion title="cache-ttl 模式行为">

- `mode: "cache-ttl"` 启用裁剪轮次。
- `ttl` 控制裁剪距离上次 cache 触达之后，多久才能再次运行。
- 如有需要，裁剪会先对过大的工具结果进行软裁剪，然后再对更旧的工具结果进行硬清空。

**软裁剪** 会保留开头和结尾，并在中间插入 `...`。

**硬清空** 会用占位符替换整个工具结果。

说明：

- 图片块永远不会被裁剪/清空。
- 比例基于字符数（近似），而不是精确 token 数。
- 如果 assistant 消息少于 `keepLastAssistants` 条，则跳过裁剪。

</Accordion>

行为详情参见 [Session Pruning](/zh-CN/concepts/session-pruning)。

### 分块流式传输

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

- 非 Telegram 渠道需要显式 `*.blockStreaming: true` 才会启用分块回复。
- 渠道覆盖：`channels.<channel>.blockStreamingCoalesce`（以及按账号变体）。Signal/Slack/Discord/Google Chat 默认 `minChars: 1500`。
- `humanDelay`：分块回复之间的随机停顿。`natural` = 800–2500ms。按智能体覆盖：`agents.list[].humanDelay`。

行为和分块细节见 [Streaming](/zh-CN/concepts/streaming)。

### 输入中指示器

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

- 默认值：私聊/被提及时使用 `instant`，未被提及的群聊使用 `message`。
- 按会话覆盖：`session.typingMode`、`session.typingIntervalSeconds`。

参见 [Typing Indicators](/zh-CN/concepts/typing-indicators)。

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

内嵌智能体的可选沙箱隔离。完整指南请参见 [Sandboxing](/zh-CN/gateway/sandboxing)。

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

<Accordion title="沙箱详情">

**后端：**

- `docker`：本地 Docker 运行时（默认）
- `ssh`：通用 SSH 支持的远程运行时
- `openshell`：OpenShell 运行时

选择 `backend: "openshell"` 时，运行时特定设置移到
`plugins.entries.openshell.config`。

**SSH 后端配置：**

- `target`：格式为 `user@host[:port]` 的 SSH 目标
- `command`：SSH 客户端命令（默认：`ssh`）
- `workspaceRoot`：用于按作用域工作区的绝对远程根目录
- `identityFile` / `certificateFile` / `knownHostsFile`：传递给 OpenSSH 的现有本地文件
- `identityData` / `certificateData` / `knownHostsData`：内联内容或 SecretRef，OpenClaw 会在运行时将其实体化为临时文件
- `strictHostKeyChecking` / `updateHostKeys`：OpenSSH 主机密钥策略控制项

**SSH auth 优先级：**

- `identityData` 优先于 `identityFile`
- `certificateData` 优先于 `certificateFile`
- `knownHostsData` 优先于 `knownHostsFile`
- SecretRef 支持的 `*Data` 值会在沙箱会话启动前从活动 secrets 运行时快照中解析

**SSH 后端行为：**

- 在创建或重建后只为远程工作区植入一次初始内容
- 之后保持远程 SSH 工作区为规范副本
- 通过 SSH 路由 `exec`、文件工具和媒体路径
- 不会自动将远程更改同步回主机
- 不支持沙箱浏览器容器

**工作区访问：**

- `none`：位于 `~/.openclaw/sandboxes` 下的按作用域沙箱工作区
- `ro`：沙箱工作区位于 `/workspace`，智能体工作区以只读方式挂载到 `/agent`
- `rw`：智能体工作区以读写方式挂载到 `/workspace`

**作用域：**

- `session`：每会话一个容器 + 工作区
- `agent`：每智能体一个容器 + 工作区（默认）
- `shared`：共享容器和工作区（无跨会话隔离）

**OpenShell 插件配置：**

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

**OpenShell 模式：**

- `mirror`：在 exec 前从本地植入远程内容，在 exec 后同步回来；本地工作区保持为规范副本
- `remote`：在创建沙箱时只向远程植入一次，然后保持远程工作区为规范副本

在 `remote` 模式下，在 OpenClaw 之外对主机本地所做的编辑，在初始植入步骤之后不会自动同步到沙箱中。
传输使用 SSH 连接到 OpenShell 沙箱，但插件负责沙箱生命周期和可选的 mirror 同步。

**`setupCommand`** 会在容器创建后运行一次（通过 `sh -lc`）。需要网络出口、可写根文件系统和 root 用户。

**容器默认使用 `network: "none"`**——如果智能体需要出站访问，请设置为 `"bridge"`（或自定义 bridge 网络）。
`"host"` 会被阻止。默认也会阻止 `"container:<id>"`，除非你显式设置
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`（紧急模式）。

**传入附件** 会被暂存到活动工作区中的 `media/inbound/*`。

**`docker.binds`** 会挂载额外的主机目录；全局和按智能体的 binds 会合并。

**沙箱隔离浏览器**（`sandbox.browser.enabled`）：容器中的 Chromium + CDP。noVNC URL 会注入系统提示。不要求在 `openclaw.json` 中启用 `browser.enabled`。
默认通过 VNC auth 提供 noVNC 观察者访问，OpenClaw 会发出短时有效的 token URL（而不是在共享 URL 中暴露密码）。

- `allowHostControl: false`（默认）会阻止沙箱隔离会话瞄准主机浏览器。
- `network` 默认为 `openclaw-sandbox-browser`（专用 bridge 网络）。仅当你明确需要全局 bridge 连通性时，才设置为 `bridge`。
- `cdpSourceRange` 可选择通过 CIDR 范围（例如 `172.21.0.1/32`）在容器边界限制 CDP 入站流量。
- `sandbox.browser.binds` 仅将额外主机目录挂载到沙箱浏览器容器中。设置后（包括 `[]`），它会替代浏览器容器中的 `docker.binds`。
- 启动默认值定义在 `scripts/sandbox-browser-entrypoint.sh` 中，并针对容器主机做了调优：
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
  - `--disable-extensions`（默认启用）
  - `--disable-3d-apis`、`--disable-software-rasterizer` 和 `--disable-gpu` 默认启用；如果你的 WebGL/3D 使用场景需要它们，可通过 `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` 禁用这些标志。
  - `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` 会重新启用扩展，如果你的工作流依赖它们。
  - `--renderer-process-limit=2` 可通过 `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` 更改；设为 `0` 可使用 Chromium 的默认进程限制。
  - 以及在启用 `noSandbox` 时附加 `--no-sandbox` 和 `--disable-setuid-sandbox`。
  - 这些默认值是容器镜像基线；如果要更改容器默认值，请使用带有自定义入口点的自定义浏览器镜像。

</Accordion>

浏览器沙箱隔离和 `sandbox.docker.binds` 目前仅支持 Docker。

构建镜像：

```bash
scripts/sandbox-setup.sh           # main sandbox image
scripts/sandbox-browser-setup.sh   # optional browser image
```

### `agents.list`（按智能体覆盖）

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

- `id`：稳定的智能体 ID（必填）。
- `default`：如果设置了多个，以第一个为准（会记录警告）。如果都未设置，则列表中的第一个为默认值。
- `model`：字符串形式只覆盖 `primary`；对象形式 `{ primary, fallbacks }` 同时覆盖两者（`[]` 会禁用全局后备）。仅覆盖 `primary` 的 cron 作业仍会继承默认后备，除非你设置 `fallbacks: []`。
- `params`：按智能体的流参数，会合并到 `agents.defaults.models` 中所选模型条目之上。用于智能体特定覆盖，如 `cacheRetention`、`temperature` 或 `maxTokens`，而无需复制整个模型目录。
- `skills`：可选的按智能体 Skills 允许列表。如果省略，则该智能体会继承 `agents.defaults.skills`（如果已设置）；显式列表会替换默认值而不是合并，`[]` 表示无 Skills。
- `thinkingDefault`：可选的按智能体默认 thinking 级别（`off | minimal | low | medium | high | xhigh | adaptive`）。当没有按消息或按会话覆盖时，会覆盖该智能体的 `agents.defaults.thinkingDefault`。
- `reasoningDefault`：可选的按智能体默认 reasoning 可见性（`on | off | stream`）。当没有按消息或按会话 reasoning 覆盖时适用。
- `fastModeDefault`：可选的按智能体 fast mode 默认值（`true | false`）。当没有按消息或按会话 fast-mode 覆盖时适用。
- `runtime`：可选的按智能体运行时描述符。当智能体应默认使用 ACP harness 会话时，可使用 `type: "acp"` 搭配 `runtime.acp` 默认值（`agent`、`backend`、`mode`、`cwd`）。
- `identity.avatar`：相对于工作区的路径、`http(s)` URL，或 `data:` URI。
- `identity` 会派生默认值：`ackReaction` 源自 `emoji`，`mentionPatterns` 源自 `name`/`emoji`。
- `subagents.allowAgents`：`sessions_spawn` 的智能体 ID 允许列表（`["*"]` = 任意；默认：仅同一智能体）。
- 沙箱继承保护：如果请求方会话处于沙箱中，`sessions_spawn` 会拒绝会导致以非沙箱方式运行的目标。
- `subagents.requireAgentId`：为 true 时，阻止省略 `agentId` 的 `sessions_spawn` 调用（强制显式 profile 选择；默认：false）。

---

## 多智能体路由

在一个 Gateway 网关中运行多个隔离智能体。参见 [Multi-Agent](/zh-CN/concepts/multi-agent)。

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

### 绑定匹配字段

- `type`（可选）：普通路由用 `route`（缺失时默认 route），持久 ACP 对话绑定用 `acp`
- `match.channel`（必填）
- `match.accountId`（可选；`*` = 任意账号；省略 = 默认账号）
- `match.peer`（可选；`{ kind: direct|group|channel, id }`）
- `match.guildId` / `match.teamId`（可选；特定渠道）
- `acp`（可选；仅用于 `type: "acp"`）：`{ mode, label, cwd, backend }`

**确定性匹配顺序：**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId`（精确，无 peer/guild/team）
5. `match.accountId: "*"`（渠道级）
6. 默认智能体

在每一层内，第一个匹配的 `bindings` 条目胜出。

对于 `type: "acp"` 条目，OpenClaw 会按精确对话身份（`match.channel` + account + `match.peer.id`）解析，而不会使用上面的 route 绑定层级顺序。

### 按智能体访问配置

<Accordion title="完全访问（无沙箱）">

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

<Accordion title="只读工具 + 工作区">

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

<Accordion title="无文件系统访问（仅消息）">

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

优先级细节见 [Multi-Agent Sandbox & Tools](/zh-CN/tools/multi-agent-sandbox-tools)。

---

## 会话

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

<Accordion title="会话字段详情">

- **`scope`**：群聊上下文的基础会话分组策略。
  - `per-sender`（默认）：在某个渠道上下文中，每个发送者都有隔离会话。
  - `global`：某个渠道上下文中的所有参与者共享一个会话（仅在确实需要共享上下文时使用）。
- **`dmScope`**：私信如何分组。
  - `main`：所有私信共享主会话。
  - `per-peer`：按发送者 ID 跨渠道隔离。
  - `per-channel-peer`：按渠道 + 发送者隔离（推荐用于多用户收件箱）。
  - `per-account-channel-peer`：按账号 + 渠道 + 发送者隔离（推荐用于多账号）。
- **`identityLinks`**：将规范 ID 映射到带提供商前缀的对端，用于跨渠道会话共享。
- **`reset`**：主重置策略。`daily` 在本地时间 `atHour` 重置；`idle` 在空闲 `idleMinutes` 后重置。当两者都配置时，先到期者生效。
- **`resetByType`**：按类型覆盖（`direct`、`group`、`thread`）。旧版 `dm` 可作为 `direct` 的别名。
- **`parentForkMaxTokens`**：创建分叉线程会话时，父会话允许的最大 `totalTokens`（默认 `100000`）。
  - 如果父会话 `totalTokens` 高于此值，OpenClaw 会启动一个新的线程会话，而不是继承父会话的转录历史。
  - 设为 `0` 可禁用此保护，并始终允许父会话分叉。
- **`mainKey`**：旧版字段。运行时现在始终对主私聊桶使用 `"main"`。
- **`agentToAgent.maxPingPongTurns`**：智能体之间进行 agent-to-agent 交换时允许的最大来回回复轮数（整数，范围：`0`–`5`）。`0` 会禁用 ping-pong 链。
- **`sendPolicy`**：按 `channel`、`chatType`（`direct|group|channel`，旧版 `dm` 为别名）、`keyPrefix` 或 `rawKeyPrefix` 匹配。第一个 deny 优先。
- **`maintenance`**：会话存储清理 + 保留控制。
  - `mode`：`warn` 仅发出警告；`enforce` 执行清理。
  - `pruneAfter`：过期条目的年龄截止（默认 `30d`）。
  - `maxEntries`：`sessions.json` 中的最大条目数（默认 `500`）。
  - `rotateBytes`：当 `sessions.json` 超过此大小时轮转（默认 `10mb`）。
  - `resetArchiveRetention`：`*.reset.<timestamp>` 转录归档的保留期。默认与 `pruneAfter` 相同；设为 `false` 可禁用。
  - `maxDiskBytes`：可选的会话目录磁盘预算。在 `warn` 模式下会记录警告；在 `enforce` 模式下会优先移除最旧的产物/会话。
  - `highWaterBytes`：预算清理后的可选目标值。默认是 `maxDiskBytes` 的 `80%`。
- **`threadBindings`**：线程绑定会话功能的全局默认值。
  - `enabled`：主默认开关（提供商可覆盖；Discord 使用 `channels.discord.threadBindings.enabled`）
  - `idleHours`：按不活动自动取消聚焦的默认小时数（`0` 表示禁用；提供商可覆盖）
  - `maxAgeHours`：硬性最大年龄的默认小时数（`0` 表示禁用；提供商可覆盖）

</Accordion>

---

## 消息

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

### 回复前缀

按渠道/账号覆盖：`channels.<channel>.responsePrefix`、`channels.<channel>.accounts.<id>.responsePrefix`。

解析顺序（最具体者胜出）：账号 → 渠道 → 全局。`""` 会禁用并停止级联。`"auto"` 会派生为 `[{identity.name}]`。

**模板变量：**

| 变量               | 说明               | 示例                        |
| ------------------ | ------------------ | --------------------------- |
| `{model}`          | 短模型名           | `claude-opus-4-6`           |
| `{modelFull}`      | 完整模型标识符     | `anthropic/claude-opus-4-6` |
| `{provider}`       | 提供商名称         | `anthropic`                 |
| `{thinkingLevel}`  | 当前 thinking 级别 | `high`、`low`、`off`        |
| `{identity.name}`  | 智能体身份名称     | （与 `"auto"` 相同）        |

变量不区分大小写。`{think}` 是 `{thinkingLevel}` 的别名。

### 确认 reaction

- 默认为活动智能体的 `identity.emoji`，否则为 `"👀"`。设为 `""` 可禁用。
- 按渠道覆盖：`channels.<channel>.ackReaction`、`channels.<channel>.accounts.<id>.ackReaction`。
- 解析顺序：账号 → 渠道 → `messages.ackReaction` → identity 后备。
- 作用域：`group-mentions`（默认）、`group-all`、`direct`、`all`。
- `removeAckAfterReply`：在 Slack、Discord 和 Telegram 上回复后移除 ack。
- `messages.statusReactions.enabled`：在 Slack、Discord 和 Telegram 上启用生命周期状态 reactions。
  在 Slack 和 Discord 上，未设置时，如果 ack reactions 处于启用状态，则状态 reactions 也保持启用。
  在 Telegram 上，需要显式设置为 `true` 才会启用生命周期状态 reactions。

### 传入防抖

将来自同一发送者的快速文本消息批量合并为单个智能体轮次。媒体/附件会立即冲刷。控制命令会绕过防抖。

### TTS（文本转语音）

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

- `auto` 控制默认自动 TTS 模式：`off`、`always`、`inbound` 或 `tagged`。`/tts on|off` 可覆盖本地偏好，`/tts status` 会显示生效状态。
- `summaryModel` 会覆盖自动摘要使用的 `agents.defaults.model.primary`。
- `modelOverrides` 默认启用；`modelOverrides.allowProvider` 默认为 `false`（选择加入）。
- API key 会回退到 `ELEVENLABS_API_KEY`/`XI_API_KEY` 和 `OPENAI_API_KEY`。
- `openai.baseUrl` 会覆盖 OpenAI TTS 端点。解析顺序是配置、`OPENAI_TTS_BASE_URL`，然后 `https://api.openai.com/v1`。
- 当 `openai.baseUrl` 指向非 OpenAI 端点时，OpenClaw 会将其视为 OpenAI 兼容 TTS 服务器，并放宽模型/语音校验。

---

## Talk

Talk 模式（macOS/iOS/Android）的默认值。

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

- 当配置了多个 Talk 提供商时，`talk.provider` 必须匹配 `talk.providers` 中的某个键。
- 旧版扁平 Talk 键（`talk.voiceId`、`talk.voiceAliases`、`talk.modelId`、`talk.outputFormat`、`talk.apiKey`）仅为兼容保留，并会自动迁移到 `talk.providers.<provider>`。
- Voice ID 会回退到 `ELEVENLABS_VOICE_ID` 或 `SAG_VOICE_ID`。
- `providers.*.apiKey` 接受明文字符串或 SecretRef 对象。
- `ELEVENLABS_API_KEY` 回退仅在未配置任何 Talk API key 时适用。
- `providers.*.voiceAliases` 允许 Talk 指令使用友好名称。
- `silenceTimeoutMs` 控制 Talk 模式在用户静默后等待多久再发送转录。未设置时保持平台默认停顿窗口（`macOS 和 Android 为 700 ms，iOS 为 900 ms`）。

---

## 工具

### 工具配置档案

`tools.profile` 会在 `tools.allow`/`tools.deny` 之前设置一个基础允许列表：

本地新手引导会在未设置时将新的本地配置默认设为 `tools.profile: "coding"`（已有显式 profile 会保留）。

| 配置档案     | 包含内容                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`    | 仅 `session_status`                                                                                                             |
| `coding`     | `group:fs`、`group:runtime`、`group:web`、`group:sessions`、`group:memory`、`cron`、`image`、`image_generate`、`video_generate` |
| `messaging`  | `group:messaging`、`sessions_list`、`sessions_history`、`sessions_send`、`session_status`                                       |
| `full`       | 不限制（与未设置相同）                                                                                                          |

### 工具组

| 组                 | 工具                                                                                                                   |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | `exec`、`process`、`code_execution`（`bash` 可作为 `exec` 的别名）                                                     |
| `group:fs`         | `read`、`write`、`edit`、`apply_patch`                                                                                 |
| `group:sessions`   | `sessions_list`、`sessions_history`、`sessions_send`、`sessions_spawn`、`sessions_yield`、`subagents`、`session_status` |
| `group:memory`     | `memory_search`、`memory_get`                                                                                          |
| `group:web`        | `web_search`、`x_search`、`web_fetch`                                                                                  |
| `group:ui`         | `browser`、`canvas`                                                                                                    |
| `group:automation` | `cron`、`gateway`                                                                                                      |
| `group:messaging`  | `message`                                                                                                              |
| `group:nodes`      | `nodes`                                                                                                                |
| `group:agents`     | `agents_list`                                                                                                          |
| `group:media`      | `image`、`image_generate`、`video_generate`、`tts`                                                                     |
| `group:openclaw`   | 所有内置工具（不含 provider 插件）                                                                                     |

### `tools.allow` / `tools.deny`

全局工具允许/拒绝策略（deny 优先）。不区分大小写，支持 `*` 通配符。即使 Docker 沙箱关闭也会应用。

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

### `tools.byProvider`

进一步限制特定提供商或模型的工具。顺序：基础 profile → 提供商 profile → allow/deny。

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

控制沙箱之外的 elevated exec 访问：

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

- 按智能体覆盖（`agents.list[].tools.elevated`）只能进一步收紧。
- `/elevated on|off|ask|full` 会按会话存储状态；内联指令仅应用于单条消息。
- elevated `exec` 会绕过沙箱隔离，并使用已配置的逃逸路径（默认 `gateway`，若 exec 目标为 `node` 则使用 `node`）。

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

工具循环安全检查**默认禁用**。设置 `enabled: true` 可激活检测。
设置可在 `tools.loopDetection` 中全局定义，也可按智能体在 `agents.list[].tools.loopDetection` 中覆盖。

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

- `historySize`：为循环分析保留的最大工具调用历史。
- `warningThreshold`：触发警告的重复无进展模式阈值。
- `criticalThreshold`：用于拦截严重循环的更高重复阈值。
- `globalCircuitBreakerThreshold`：任意无进展运行的硬停止阈值。
- `detectors.genericRepeat`：对重复的同工具/同参数调用发出警告。
- `detectors.knownPollNoProgress`：对已知轮询工具（`process.poll`、`command_status` 等）的无进展情况发出警告/阻止。
- `detectors.pingPong`：对交替出现的无进展配对模式发出警告/阻止。
- 如果 `warningThreshold >= criticalThreshold` 或 `criticalThreshold >= globalCircuitBreakerThreshold`，校验会失败。

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

配置传入媒体理解（图像/音频/视频）：

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

<Accordion title="媒体模型条目字段">

**Provider 条目**（`type: "provider"` 或省略）：

- `provider`：API 提供商 ID（`openai`、`anthropic`、`google`/`gemini`、`groq` 等）
- `model`：模型 ID 覆盖
- `profile` / `preferredProfile`：`auth-profiles.json` 配置档案选择

**CLI 条目**（`type: "cli"`）：

- `command`：要运行的可执行文件
- `args`：模板化参数（支持 `{{MediaPath}}`、`{{Prompt}}`、`{{MaxChars}}` 等）

**通用字段：**

- `capabilities`：可选列表（`image`、`audio`、`video`）。默认值：`openai`/`anthropic`/`minimax` → image，`google` → image+audio+video，`groq` → audio。
- `prompt`、`maxChars`、`maxBytes`、`timeoutSeconds`、`language`：按条目覆盖。
- 失败时会回退到下一个条目。

提供商 auth 遵循标准顺序：`auth-profiles.json` → 环境变量 → `models.providers.*.apiKey`。

**异步完成字段：**

- `asyncCompletion.directSend`：为 `true` 时，已完成的异步 `music_generate`
  和 `video_generate` 任务会优先尝试直接交付到渠道。默认：`false`
  （旧版请求者会话唤醒/模型交付路径）。

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

控制会话工具（`sessions_list`、`sessions_history`、`sessions_send`）可以定位哪些会话。

默认值：`tree`（当前会话 + 由其生成的会话，如 subagents）。

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

说明：

- `self`：仅当前会话键。
- `tree`：当前会话 + 由当前会话生成的会话（subagents）。
- `agent`：属于当前智能体 ID 的任意会话（如果你在同一智能体 ID 下运行按发送者会话，可能包括其他用户）。
- `all`：任意会话。跨智能体定位仍需要 `tools.agentToAgent`。
- 沙箱钳制：当当前会话处于沙箱中，且 `agents.defaults.sandbox.sessionToolsVisibility="spawned"` 时，即使 `tools.sessions.visibility="all"`，可见性也会被强制为 `tree`。

### `tools.sessions_spawn`

控制 `sessions_spawn` 的内联附件支持。

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

说明：

- 附件仅支持 `runtime: "subagent"`。ACP runtime 会拒绝它们。
- 文件会实体化到子工作区中的 `.openclaw/attachments/<uuid>/`，并附带一个 `.manifest.json`。
- 附件内容会自动从转录持久化中脱敏。
- Base64 输入会经过严格字母表/填充校验，以及解码前大小保护。
- 文件权限为目录 `0700`、文件 `0600`。
- 清理由 `cleanup` 策略控制：`delete` 总会移除附件；`keep` 仅在 `retainOnSessionKeep: true` 时保留。

### `tools.experimental`

实验性内置工具开关。默认关闭，除非某条运行时特定的自动启用规则适用。

```json5
{
  tools: {
    experimental: {
      planTool: true, // enable experimental update_plan
    },
  },
}
```

说明：

- `planTool`：启用结构化 `update_plan` 工具，用于非平凡多步骤工作跟踪。
- 默认：对于非 OpenAI 提供商为 `false`。OpenAI 和 OpenAI Codex 运行在未设置时会自动启用；设为 `false` 可禁用该自动启用。
- 启用后，系统提示也会加入使用指导，使模型仅在实质性工作中使用它，并且最多只保留一个 `in_progress` 步骤。

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

- `model`：生成的子智能体默认模型。如果省略，子智能体会继承调用者模型。
- `allowAgents`：当请求方智能体没有设置自己的 `subagents.allowAgents` 时，`sessions_spawn` 的目标智能体 ID 默认允许列表（`["*"]` = 任意；默认：仅同一智能体）。
- `runTimeoutSeconds`：当工具调用省略 `runTimeoutSeconds` 时，`sessions_spawn` 使用的默认超时（秒）。`0` 表示无超时。
- 每子智能体工具策略：`tools.subagents.tools.allow` / `tools.subagents.tools.deny`。

---

## 自定义提供商和 base URL

OpenClaw 使用内置模型目录。可通过配置中的 `models.providers` 或 `~/.openclaw/agents/<agentId>/agent/models.json` 添加自定义提供商。

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

- 对于自定义 auth 需求，使用 `authHeader: true` + `headers`。
- 使用 `OPENCLAW_AGENT_DIR`（或旧版环境变量别名 `PI_CODING_AGENT_DIR`）覆盖智能体配置根目录。
- 匹配提供商 ID 的合并优先级：
  - 非空的智能体 `models.json` `baseUrl` 值优先。
  - 仅当该提供商在当前配置/auth-profile 上下文中不是 SecretRef 托管时，非空的智能体 `apiKey` 值才优先。
  - SecretRef 托管的提供商 `apiKey` 值会从源标记刷新（环境变量引用为 `ENV_VAR_NAME`，文件/exec 引用为 `secretref-managed`），而不会持久化已解析的密钥。
  - SecretRef 托管的提供商 header 值会从源标记刷新（环境变量引用为 `secretref-env:ENV_VAR_NAME`，文件/exec 引用为 `secretref-managed`）。
  - 空或缺失的智能体 `apiKey`/`baseUrl` 会回退到配置中的 `models.providers`。
  - 匹配模型的 `contextWindow`/`maxTokens` 会取显式配置与隐式目录值中的较高者。
  - 匹配模型的 `contextTokens` 在存在时会保留显式运行时上限；用它可限制生效上下文，而不更改模型原生元数据。
  - 当你希望配置完全重写 `models.json` 时，使用 `models.mode: "replace"`。
  - 标记持久化遵循源权威：标记是根据活动源配置快照（解析前）写入的，而不是根据已解析运行时密钥值写入。

### 提供商字段详情

- `models.mode`：提供商目录行为（`merge` 或 `replace`）。
- `models.providers`：按提供商 ID 作为键的自定义提供商映射。
- `models.providers.*.api`：请求适配器（`openai-completions`、`openai-responses`、`anthropic-messages`、`google-generative-ai` 等）。
- `models.providers.*.apiKey`：提供商凭证（优先使用 SecretRef/环境变量替换）。
- `models.providers.*.auth`：auth 策略（`api-key`、`token`、`oauth`、`aws-sdk`）。
- `models.providers.*.injectNumCtxForOpenAICompat`：对于 Ollama + `openai-completions`，向请求中注入 `options.num_ctx`（默认：`true`）。
- `models.providers.*.authHeader`：在需要时强制通过 `Authorization` header 传输凭证。
- `models.providers.*.baseUrl`：上游 API base URL。
- `models.providers.*.headers`：用于代理/租户路由的额外静态 headers。
- `models.providers.*.request`：模型提供商 HTTP 请求的传输覆盖。
  - `request.headers`：额外 headers（与提供商默认值合并）。值接受 SecretRef。
  - `request.auth`：auth 策略覆盖。模式：`"provider-default"`（使用提供商内置 auth）、`"authorization-bearer"`（配合 `token`）、`"header"`（配合 `headerName`、`value`，可选 `prefix`）。
  - `request.proxy`：HTTP 代理覆盖。模式：`"env-proxy"`（使用 `HTTP_PROXY`/`HTTPS_PROXY` 环境变量）、`"explicit-proxy"`（配合 `url`）。两种模式都接受可选的 `tls` 子对象。
  - `request.tls`：直连的 TLS 覆盖。字段：`ca`、`cert`、`key`、`passphrase`（都接受 SecretRef）、`serverName`、`insecureSkipVerify`。
- `models.providers.*.models`：显式提供商模型目录条目。
- `models.providers.*.models.*.contextWindow`：原生模型上下文窗口元数据。
- `models.providers.*.models.*.contextTokens`：可选运行时上下文上限。当你希望生效上下文预算小于模型原生 `contextWindow` 时，请使用它。
- `models.providers.*.models.*.compat.supportsDeveloperRole`：可选兼容性提示。对于 `api: "openai-completions"` 且 `baseUrl` 非空且非原生（host 不是 `api.openai.com`）时，OpenClaw 会在运行时强制将其设为 `false`。空/省略的 `baseUrl` 会保留默认 OpenAI 行为。
- `models.providers.*.models.*.compat.requiresStringContent`：用于仅支持字符串的 OpenAI 兼容聊天端点的可选兼容性提示。为 `true` 时，OpenClaw 会在发送请求前把纯文本 `messages[].content` 数组压平成普通字符串。
- `plugins.entries.amazon-bedrock.config.discovery`：Bedrock 自动发现设置根。
- `plugins.entries.amazon-bedrock.config.discovery.enabled`：开启/关闭隐式发现。
- `plugins.entries.amazon-bedrock.config.discovery.region`：用于发现的 AWS 区域。
- `plugins.entries.amazon-bedrock.config.discovery.providerFilter`：用于定向发现的可选 provider-id 过滤器。
- `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`：发现刷新的轮询间隔。
- `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`：已发现模型的后备上下文窗口。
- `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`：已发现模型的后备最大输出 token。

### 提供商示例

<Accordion title="Cerebras（GLM 4.6 / 4.7）">

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

Cerebras 请使用 `cerebras/zai-glm-4.7`；Z.AI 直连请使用 `zai/glm-4.7`。

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

设置 `OPENCODE_API_KEY`（或 `OPENCODE_ZEN_API_KEY`）。Zen 目录使用 `opencode/...` 引用，Go 目录使用 `opencode-go/...` 引用。快捷方式：`openclaw onboard --auth-choice opencode-zen` 或 `openclaw onboard --auth-choice opencode-go`。

</Accordion>

<Accordion title="Z.AI（GLM-4.7）">

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

设置 `ZAI_API_KEY`。`z.ai/*` 和 `z-ai/*` 都是可接受的别名。快捷方式：`openclaw onboard --auth-choice zai-api-key`。

- 通用端点：`https://api.z.ai/api/paas/v4`
- Coding 端点（默认）：`https://api.z.ai/api/coding/paas/v4`
- 若使用通用端点，请定义一个带 base URL 覆盖的自定义提供商。

</Accordion>

<Accordion title="Moonshot AI（Kimi）">

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

中国端点：`baseUrl: "https://api.moonshot.cn/v1"` 或 `openclaw onboard --auth-choice moonshot-api-key-cn`。

原生 Moonshot 端点会在共享的
`openai-completions` 传输上声明流式使用兼容性，而 OpenClaw 现在根据端点能力而不只是内置 provider id 来判定这一点。

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

Anthropic 兼容，内置提供商。快捷方式：`openclaw onboard --auth-choice kimi-code-api-key`。

</Accordion>

<Accordion title="Synthetic（Anthropic 兼容）">

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

Base URL 不应包含 `/v1`（Anthropic 客户端会自行追加）。快捷方式：`openclaw onboard --auth-choice synthetic-api-key`。

</Accordion>

<Accordion title="MiniMax M2.7（直连）">

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

设置 `MINIMAX_API_KEY`。快捷方式：
`openclaw onboard --auth-choice minimax-global-api` 或
`openclaw onboard --auth-choice minimax-cn-api`。
模型目录现在默认仅包含 M2.7。
在 Anthropic 兼容的流式路径上，除非你显式设置 `thinking`，否则 OpenClaw 默认会关闭 MiniMax thinking。
`/fast on` 或 `params.fastMode: true` 会将 `MiniMax-M2.7` 重写为
`MiniMax-M2.7-highspeed`。

</Accordion>

<Accordion title="本地模型（LM Studio）">

请参见 [Local Models](/zh-CN/gateway/local-models)。简要来说：在配置较强硬件上通过 LM Studio Responses API 运行大型本地模型；同时保留已合并的托管模型作为后备。

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

- `allowBundled`：仅针对内置 Skills 的可选允许列表（不影响托管/工作区 Skills）。
- `load.extraDirs`：额外的共享 Skills 根目录（优先级最低）。
- `install.preferBrew`：为 true 时，如果 `brew` 可用，则优先使用 Homebrew 安装器，再回退到其他安装器类型。
- `install.nodeManager`：用于 `metadata.openclaw.install` 规范的 node 安装器偏好（`npm` | `pnpm` | `yarn` | `bun`）。
- `entries.<skillKey>.enabled: false`：即使某个 skill 已内置/安装，也会禁用它。
- `entries.<skillKey>.apiKey`：为声明了主要环境变量的 skill 提供便捷 API key 字段（明文字符串或 SecretRef 对象）。

---

## 插件

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

- 从 `~/.openclaw/extensions`、`<workspace>/.openclaw/extensions` 和 `plugins.load.paths` 加载。
- 设备发现接受原生 OpenClaw 插件，以及兼容的 Codex bundles 和 Claude bundles，包括无 manifest 的 Claude 默认布局 bundles。
- **配置更改需要重启 Gateway 网关。**
- `allow`：可选允许列表（仅加载列出的插件）。`deny` 优先。
- `plugins.entries.<id>.apiKey`：插件级 API key 便捷字段（若插件支持）。
- `plugins.entries.<id>.env`：插件作用域环境变量映射。
- `plugins.entries.<id>.hooks.allowPromptInjection`：为 `false` 时，核心会阻止 `before_prompt_build`，并忽略旧版 `before_agent_start` 中会修改提示的字段，同时保留旧版 `modelOverride` 和 `providerOverride`。适用于原生插件 hooks 和支持的 bundle 提供 hook 目录。
- `plugins.entries.<id>.subagent.allowModelOverride`：显式信任此插件，使其可为后台子智能体运行请求按次 `provider` 和 `model` 覆盖。
- `plugins.entries.<id>.subagent.allowedModels`：受信任子智能体覆盖的可选规范 `provider/model` 目标允许列表。仅当你确实想允许任意模型时，才使用 `"*"`。
- `plugins.entries.<id>.config`：插件定义的配置对象（若可用，则由原生 OpenClaw 插件 schema 校验）。
- `plugins.entries.firecrawl.config.webFetch`：Firecrawl web 获取提供商设置。
  - `apiKey`：Firecrawl API key（接受 SecretRef）。会回退到 `plugins.entries.firecrawl.config.webSearch.apiKey`、旧版 `tools.web.fetch.firecrawl.apiKey`，或 `FIRECRAWL_API_KEY` 环境变量。
  - `baseUrl`：Firecrawl API base URL（默认：`https://api.firecrawl.dev`）。
  - `onlyMainContent`：仅提取页面主体内容（默认：`true`）。
  - `maxAgeMs`：最大缓存年龄（毫秒，默认：`172800000` / 2 天）。
  - `timeoutSeconds`：抓取请求超时（秒，默认：`60`）。
- `plugins.entries.xai.config.xSearch`：xAI X Search（Grok web 搜索）设置。
  - `enabled`：启用 X Search 提供商。
  - `model`：用于搜索的 Grok 模型（例如 `"grok-4-1-fast"`）。
- `plugins.entries.memory-core.config.dreaming`：memory dreaming（实验性）设置。有关阶段和阈值，请参见 [Dreaming](/zh-CN/concepts/dreaming)。
  - `enabled`：dreaming 主开关（默认 `false`）。
  - `frequency`：每次完整 dreaming 扫描的 cron 频率（默认 `"0 3 * * *"`）。
  - 阶段策略和阈值是实现细节（不是面向用户的配置键）。
- 完整 memory 配置见 [Memory configuration reference](/zh-CN/reference/memory-config)：
  - `agents.defaults.memorySearch.*`
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- 已启用的 Claude bundle 插件也可以从 `settings.json` 提供嵌入式 Pi 默认值；OpenClaw 会将它们作为清理后的智能体设置应用，而不是原始 OpenClaw 配置补丁。
- `plugins.slots.memory`：选择活动 memory 插件 ID，或用 `"none"` 禁用 memory 插件。
- `plugins.slots.contextEngine`：选择活动 context engine 插件 ID；默认是 `"legacy"`，除非你安装并选择了其他 engine。
- `plugins.installs`：CLI 管理的安装元数据，供 `openclaw plugins update` 使用。
  - 包括 `source`、`spec`、`sourcePath`、`installPath`、`version`、`resolvedName`、`resolvedVersion`、`resolvedSpec`、`integrity`、`shasum`、`resolvedAt`、`installedAt`。
  - 请将 `plugins.installs.*` 视为托管状态；优先使用 CLI 命令，而不是手动编辑。

参见 [Plugins](/zh-CN/tools/plugin)。

---

## 浏览器

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

- `evaluateEnabled: false` 会禁用 `act:evaluate` 和 `wait --fn`。
- 当未设置时，`ssrfPolicy.dangerouslyAllowPrivateNetwork` 默认为 `true`（受信任网络模型）。
- 将 `ssrfPolicy.dangerouslyAllowPrivateNetwork: false` 设为严格的仅公网浏览导航。
- 在严格模式下，远程 CDP 配置档案端点（`profiles.*.cdpUrl`）在可达性/发现检查期间也会受到同样的私有网络阻止。
- `ssrfPolicy.allowPrivateNetwork` 仍作为旧版别名受支持。
- 在严格模式下，使用 `ssrfPolicy.hostnameAllowlist` 和 `ssrfPolicy.allowedHostnames` 来配置显式例外。
- 远程配置档案为仅附加模式（禁用 start/stop/reset）。
- `profiles.*.cdpUrl` 接受 `http://`、`https://`、`ws://` 和 `wss://`。
  当你希望 OpenClaw 自动发现 `/json/version` 时，请使用 HTTP(S)；
  当你的提供商直接给出 DevTools WebSocket URL 时，请使用 WS(S)。
- `existing-session` 配置档案仅限主机，并使用 Chrome MCP 而非 CDP。
- `existing-session` 配置档案可设置 `userDataDir` 以定位特定的
  Chromium 系浏览器配置，例如 Brave 或 Edge。
- `existing-session` 配置档案保留当前 Chrome MCP 路由限制：
  基于 snapshot/ref 的操作而不是 CSS 选择器定位、单文件上传 hooks、
  不支持对话框超时覆盖、不支持 `wait --load networkidle`，以及不支持
  `responsebody`、PDF 导出、下载拦截或批量操作。
- 本地托管的 `openclaw` 配置档案会自动分配 `cdpPort` 和 `cdpUrl`；仅在远程 CDP 时显式设置 `cdpUrl`。
- 自动检测顺序：默认浏览器（如果是 Chromium 系）→ Chrome → Brave → Edge → Chromium → Chrome Canary。
- 控制服务：仅 loopback（端口派生自 `gateway.port`，默认 `18791`）。
- `extraArgs` 会将额外启动标志附加到本地 Chromium 启动中（例如
  `--disable-gpu`、窗口大小，或调试标志）。

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

- `seamColor`：原生应用 UI 外观的强调色（Talk 模式气泡色调等）。
- `assistant`：Control UI 身份覆盖。会回退到活动智能体身份。

---

## Gateway 网关

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

<Accordion title="Gateway 网关字段详情">

- `mode`：`local`（运行 gateway）或 `remote`（连接远程 gateway）。除非为 `local`，否则 Gateway 网关会拒绝启动。
- `port`：用于 WS + HTTP 的单一复用端口。优先级：`--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`。
- `bind`：`auto`、`loopback`（默认）、`lan`（`0.0.0.0`）、`tailnet`（仅 Tailscale IP）或 `custom`。
- **旧版 bind 别名**：在 `gateway.bind` 中使用 bind mode 值（`auto`、`loopback`、`lan`、`tailnet`、`custom`），不要使用主机别名（`0.0.0.0`、`127.0.0.1