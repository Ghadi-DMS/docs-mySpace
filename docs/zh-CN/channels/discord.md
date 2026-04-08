---
read_when:
    - 处理 Discord 渠道功能时
summary: Discord 机器人支持状态、功能与配置
title: Discord
x-i18n:
    generated_at: "2026-04-08T06:30:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3cd2886fad941ae2129e681911309539e9a65a2352b777b538d7f4686a68f73f
    source_path: channels/discord.md
    workflow: 15
---

# Discord（Bot API）

状态：已就绪，可通过官方 Discord Gateway 网关用于私信和服务器渠道。

<CardGroup cols={3}>
  <Card title="配对" icon="link" href="/zh-CN/channels/pairing">
    Discord 私信默认使用配对模式。
  </Card>
  <Card title="斜杠命令" icon="terminal" href="/zh-CN/tools/slash-commands">
    原生命令行为与命令目录。
  </Card>
  <Card title="渠道故障排除" icon="wrench" href="/zh-CN/channels/troubleshooting">
    跨渠道诊断与修复流程。
  </Card>
</CardGroup>

## 快速设置

你需要创建一个带机器人的新应用，将机器人添加到你的服务器，并将其与 OpenClaw 配对。我们建议你将机器人添加到你自己的私有服务器。如果你还没有服务器，请先[创建一个](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server)（选择 **Create My Own > For me and my friends**）。

<Steps>
  <Step title="创建 Discord 应用和机器人">
    前往 [Discord Developer Portal](https://discord.com/developers/applications)，然后点击 **New Application**。给它起一个类似 “OpenClaw” 的名字。

    点击侧边栏中的 **Bot**。将 **Username** 设置为你对 OpenClaw 智能体的称呼。

  </Step>

  <Step title="启用特权 intents">
    仍然停留在 **Bot** 页面，向下滚动到 **Privileged Gateway Intents** 并启用：

    - **Message Content Intent**（必需）
    - **Server Members Intent**（推荐；角色 allowlist 和名称到 ID 匹配所必需）
    - **Presence Intent**（可选；仅在你需要 presence 更新时才需要）

  </Step>

  <Step title="复制你的机器人令牌">
    滚回 **Bot** 页面顶部并点击 **Reset Token**。

    <Note>
    尽管名称如此，这会生成你的第一个令牌——并没有任何内容被“重置”。
    </Note>

    复制该令牌并保存到某处。这就是你的 **Bot Token**，你很快就会用到它。

  </Step>

  <Step title="生成邀请 URL 并将机器人添加到你的服务器">
    点击侧边栏中的 **OAuth2**。你将生成一个带有正确权限的邀请 URL，用于将机器人添加到你的服务器。

    向下滚动到 **OAuth2 URL Generator** 并启用：

    - `bot`
    - `applications.commands`

    下方会出现一个 **Bot Permissions** 区域。启用：

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions（可选）

    复制底部生成的 URL，将其粘贴到浏览器中，选择你的服务器，然后点击 **Continue** 完成连接。现在你应该能在 Discord 服务器中看到你的机器人。

  </Step>

  <Step title="启用 Developer Mode 并收集你的 ID">
    回到 Discord 应用中，你需要启用 Developer Mode，这样才能复制内部 ID。

    1. 点击 **User Settings**（头像旁边的齿轮图标）→ **Advanced** → 打开 **Developer Mode**
    2. 在侧边栏中右键点击你的**服务器图标** → **Copy Server ID**
    3. 右键点击你自己的**头像** → **Copy User ID**

    将你的 **Server ID** 和 **User ID** 与 Bot Token 一起保存——下一步你需要把这三项都发送给 OpenClaw。

  </Step>

  <Step title="允许来自服务器成员的私信">
    要让配对正常工作，Discord 需要允许你的机器人向你发送私信。右键点击你的**服务器图标** → **Privacy Settings** → 打开 **Direct Messages**。

    这样服务器成员（包括机器人）就可以向你发送私信。如果你想在 OpenClaw 中使用 Discord 私信，请保持启用。如果你只打算使用服务器渠道，则可以在配对完成后关闭私信。

  </Step>

  <Step title="安全地设置你的机器人令牌（不要在聊天中发送）">
    你的 Discord 机器人令牌是机密信息（就像密码一样）。在给智能体发消息之前，先在运行 OpenClaw 的机器上设置它。

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set channels.discord.enabled true --strict-json
openclaw gateway
```

    如果 OpenClaw 已经作为后台服务运行，请通过 OpenClaw Mac 应用重启它，或停止并重新启动 `openclaw gateway run` 进程。

  </Step>

  <Step title="配置 OpenClaw 并配对">

    <Tabs>
      <Tab title="询问你的智能体">
        在任何现有渠道（例如 Telegram）上与你的 OpenClaw 智能体聊天并告诉它。如果 Discord 是你的第一个渠道，请改用 CLI / config 标签页。

        > “我已经在配置中设置好了 Discord 机器人令牌。请使用 User ID `<user_id>` 和 Server ID `<server_id>` 完成 Discord 设置。”
      </Tab>
      <Tab title="CLI / config">
        如果你更喜欢基于文件的配置，请设置：

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        默认账户的环境变量回退：

```bash
DISCORD_BOT_TOKEN=...
```

        支持明文 `token` 值。`channels.discord.token` 也支持跨 env/file/exec 提供商的 SecretRef 值。参见[密钥管理](/zh-CN/gateway/secrets)。

      </Tab>
    </Tabs>

  </Step>

  <Step title="批准首次私信配对">
    等待 Gateway 网关运行，然后在 Discord 中向你的机器人发送私信。它会回复一个配对码。

    <Tabs>
      <Tab title="询问你的智能体">
        将配对码发送给你现有渠道中的智能体：

        > “批准这个 Discord 配对码：`<CODE>`”
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    配对码会在 1 小时后过期。

    现在你应该可以通过 Discord 私信与你的智能体聊天了。

  </Step>
</Steps>

<Note>
令牌解析具有账户感知能力。配置中的令牌值优先于环境变量回退。`DISCORD_BOT_TOKEN` 仅用于默认账户。
对于高级出站调用（消息工具/渠道操作），显式的每次调用 `token` 会用于该次调用。这适用于发送和读取/探测类操作（例如 read/search/fetch/thread/pins/permissions）。账户策略/重试设置仍然来自活动运行时快照中选定的账户。
</Note>

## 推荐：设置服务器工作区

一旦私信可用，你就可以将 Discord 服务器设置为完整工作区，其中每个渠道都有自己的智能体会话和独立上下文。对于只有你和机器人参与的私有服务器，我们推荐这样做。

<Steps>
  <Step title="将你的服务器添加到服务器 allowlist">
    这样可以让你的智能体在服务器中的任意渠道响应，而不只是私信。

    <Tabs>
      <Tab title="询问你的智能体">
        > “将我的 Discord Server ID `<server_id>` 添加到服务器 allowlist”
      </Tab>
      <Tab title="配置">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="允许无需 @mention 的响应">
    默认情况下，你的智能体只会在被 @mentioned 时才在服务器渠道中响应。对于私有服务器，你大概率会希望它对每条消息都响应。

    <Tabs>
      <Tab title="询问你的智能体">
        > “允许我的智能体在这个服务器中无需被 @mentioned 也能响应”
      </Tab>
      <Tab title="配置">
        在你的服务器配置中设置 `requireMention: false`：

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="为服务器渠道中的记忆做好规划">
    默认情况下，长期记忆（MEMORY.md）只会在私信会话中加载。服务器渠道不会自动加载 MEMORY.md。

    <Tabs>
      <Tab title="询问你的智能体">
        > “当我在 Discord 渠道中提问时，如果你需要来自 MEMORY.md 的长期上下文，请使用 memory_search 或 memory_get。”
      </Tab>
      <Tab title="手动">
        如果你需要在每个渠道中共享上下文，请将稳定指令放在 `AGENTS.md` 或 `USER.md` 中（它们会注入到每个会话）。将长期笔记保存在 `MEMORY.md` 中，并按需通过记忆工具访问。
      </Tab>
    </Tabs>

  </Step>
</Steps>

现在，在你的 Discord 服务器中创建一些渠道并开始聊天吧。你的智能体可以看到渠道名称，而且每个渠道都会获得自己隔离的会话——所以你可以设置 `#coding`、`#home`、`#research`，或任何适合你工作流的内容。

## 运行时模型

- Gateway 网关拥有 Discord 连接。
- 回复路由是确定性的：来自 Discord 的入站回复会返回到 Discord。
- 默认情况下（`session.dmScope=main`），私聊会共享智能体主会话（`agent:main:main`）。
- 服务器渠道使用隔离的会话键（`agent:<agentId>:discord:channel:<channelId>`）。
- 默认忽略群组私信（`channels.discord.dm.groupEnabled=false`）。
- 原生斜杠命令在隔离的命令会话中运行（`agent:<agentId>:discord:slash:<userId>`），同时仍会将 `CommandTargetSessionKey` 传递到被路由的会话会话。

## 论坛渠道

Discord 论坛和媒体渠道只接受线程帖子。OpenClaw 支持两种创建方式：

- 向论坛父级发送消息（`channel:<forumId>`）以自动创建线程。线程标题会使用你消息中第一行非空内容。
- 使用 `openclaw message thread create` 直接创建线程。不要为论坛渠道传递 `--message-id`。

示例：发送到论坛父级以创建线程

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

示例：显式创建论坛线程

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

论坛父级不接受 Discord 组件。如果你需要组件，请发送到线程本身（`channel:<threadId>`）。

## 交互式组件

OpenClaw 为智能体消息支持 Discord components v2 容器。使用带有 `components` 载荷的消息工具。交互结果会像普通入站消息一样被路由回智能体，并遵循现有的 Discord `replyToMode` 设置。

支持的块：

- `text`、`section`、`separator`、`actions`、`media-gallery`、`file`
- 操作行最多允许 5 个按钮或单个选择菜单
- 选择类型：`string`、`user`、`role`、`mentionable`、`channel`

默认情况下，组件只能使用一次。设置 `components.reusable=true` 以允许按钮、选择器和表单在过期前被多次使用。

要限制谁可以点击按钮，请在该按钮上设置 `allowedUsers`（Discord 用户 ID、标签或 `*`）。配置后，不匹配的用户会收到一条临时拒绝消息。

`/model` 和 `/models` 斜杠命令会打开一个交互式模型选择器，其中包含提供商和模型下拉框，以及一个提交步骤。选择器回复是临时的，且只有调用该命令的用户可以使用它。

文件附件：

- `file` 块必须指向一个附件引用（`attachment://<filename>`）
- 通过 `media`/`path`/`filePath` 提供附件（单文件）；多个文件请使用 `media-gallery`
- 当上传名称应与附件引用匹配时，使用 `filename` 覆盖上传名称

模态表单：

- 添加 `components.modal`，最多支持 5 个字段
- 字段类型：`text`、`checkbox`、`radio`、`select`、`role-select`、`user-select`
- OpenClaw 会自动添加一个触发按钮

示例：

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Optional fallback text",
  components: {
    reusable: true,
    text: "Choose a path",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Approve",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Decline", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Pick an option",
          options: [
            { label: "Option A", value: "a" },
            { label: "Option B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Details",
      triggerLabel: "Open form",
      fields: [
        { type: "text", label: "Requester" },
        {
          type: "select",
          label: "Priority",
          options: [
            { label: "Low", value: "low" },
            { label: "High", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## 访问控制与路由

<Tabs>
  <Tab title="私信策略">
    `channels.discord.dmPolicy` 控制私信访问（旧版：`channels.discord.dm.policy`）：

    - `pairing`（默认）
    - `allowlist`
    - `open`（要求 `channels.discord.allowFrom` 包含 `"*"`；旧版：`channels.discord.dm.allowFrom`）
    - `disabled`

    如果私信策略不是 open，未知用户会被阻止（或在 `pairing` 模式下提示进行配对）。

    多账户优先级：

    - `channels.discord.accounts.default.allowFrom` 仅适用于 `default` 账户。
    - 已命名账户在其自身 `allowFrom` 未设置时，会继承 `channels.discord.allowFrom`。
    - 已命名账户不会继承 `channels.discord.accounts.default.allowFrom`。

    用于投递的私信目标格式：

    - `user:<id>`
    - `<@id>` mention

    纯数字 ID 存在歧义，会被拒绝，除非显式提供 user/channel 目标类型。

  </Tab>

  <Tab title="服务器策略">
    服务器处理由 `channels.discord.groupPolicy` 控制：

    - `open`
    - `allowlist`
    - `disabled`

    当存在 `channels.discord` 时，安全基线为 `allowlist`。

    `allowlist` 行为：

    - 服务器必须匹配 `channels.discord.guilds`（优先使用 `id`，也接受 slug）
    - 可选的发送者 allowlist：`users`（推荐使用稳定 ID）和 `roles`（仅角色 ID）；如果配置了任一项，则发送者在匹配 `users` 或 `roles` 时被允许
    - 默认禁用直接名称/标签匹配；仅在紧急兼容模式下启用 `channels.discord.dangerouslyAllowNameMatching: true`
    - `users` 支持名称/标签，但 ID 更安全；当使用名称/标签条目时，`openclaw security audit` 会发出警告
    - 如果某个服务器配置了 `channels`，则未列出的渠道会被拒绝
    - 如果某个服务器没有 `channels` 块，则该 allowlist 服务器中的所有渠道都被允许

    示例：

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    如果你只设置了 `DISCORD_BOT_TOKEN` 而没有创建 `channels.discord` 块，则运行时回退为 `groupPolicy="allowlist"`（日志中会有警告），即使 `channels.defaults.groupPolicy` 是 `open` 也是如此。

  </Tab>

  <Tab title="Mention 和群组私信">
    默认情况下，服务器消息受 mention 门控。

    mention 检测包括：

    - 显式提及机器人
    - 已配置的 mention 模式（`agents.list[].groupChat.mentionPatterns`，回退为 `messages.groupChat.mentionPatterns`）
    - 在支持场景中的隐式回复机器人行为

    `requireMention` 按服务器/渠道配置（`channels.discord.guilds...`）。
    `ignoreOtherMentions` 可选择丢弃提及了其他用户/角色但未提及机器人的消息（不包括 @everyone/@here）。

    群组私信：

    - 默认：忽略（`dm.groupEnabled=false`）
    - 可选 allowlist：通过 `dm.groupChannels`（渠道 ID 或 slug）

  </Tab>
</Tabs>

### 基于角色的智能体路由

使用 `bindings[].match.roles` 可按角色 ID 将 Discord 服务器成员路由到不同智能体。基于角色的绑定仅接受角色 ID，并且会在 peer 或 parent-peer 绑定之后、guild-only 绑定之前进行评估。如果某个绑定还设置了其他匹配字段（例如 `peer` + `guildId` + `roles`），则所有已配置字段都必须匹配。

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Developer Portal 设置

<AccordionGroup>
  <Accordion title="创建应用和机器人">

    1. Discord Developer Portal -> **Applications** -> **New Application**
    2. **Bot** -> **Add Bot**
    3. 复制机器人令牌

  </Accordion>

  <Accordion title="特权 intents">
    在 **Bot -> Privileged Gateway Intents** 中，启用：

    - Message Content Intent
    - Server Members Intent（推荐）

    Presence intent 是可选的，只有当你希望接收 presence 更新时才需要。如果你要设置机器人 presence（`setPresence`），则不需要为成员启用 presence 更新。

  </Accordion>

  <Accordion title="OAuth scopes 和基线权限">
    OAuth URL 生成器：

    - scopes：`bot`、`applications.commands`

    典型基线权限：

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions（可选）

    除非明确需要，否则避免使用 `Administrator`。

  </Accordion>

  <Accordion title="复制 ID">
    启用 Discord Developer Mode，然后复制：

    - server ID
    - channel ID
    - user ID

    在 OpenClaw 配置中优先使用数字 ID，以获得可靠的审计和探测结果。

  </Accordion>
</AccordionGroup>

## 原生命令和命令鉴权

- `commands.native` 默认为 `"auto"`，并为 Discord 启用。
- 每渠道覆盖：`channels.discord.commands.native`。
- `commands.native=false` 会显式清除先前注册的 Discord 原生命令。
- 原生命令鉴权与普通消息处理使用相同的 Discord allowlist/策略。
- 即使对未获授权的用户，命令仍可能显示在 Discord UI 中；执行时仍会强制执行 OpenClaw 鉴权并返回 “not authorized”。

命令目录和行为请参见[斜杠命令](/zh-CN/tools/slash-commands)。

默认斜杠命令设置：

- `ephemeral: true`

## 功能细节

<AccordionGroup>
  <Accordion title="回复标签和原生回复">
    Discord 支持智能体输出中的回复标签：

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    由 `channels.discord.replyToMode` 控制：

    - `off`（默认）
    - `first`
    - `all`
    - `batched`

    注意：`off` 会禁用隐式回复线程。显式的 `[[reply_to_*]]` 标签仍会生效。
    `first` 总是会将隐式原生回复引用附加到该轮的第一条出站 Discord 消息。
    `batched` 仅在入站轮次是由多条消息去抖合并后的批次时，才附加 Discord 的隐式原生回复引用。这在你主要希望针对含糊的突发聊天启用原生回复，而不是每个单消息轮次都启用时很有用。

    消息 ID 会显示在上下文/历史中，因此智能体可以针对特定消息进行回复。

  </Accordion>

  <Accordion title="实时流预览">
    OpenClaw 可以通过发送一条临时消息并在文本到达时编辑它来流式显示草稿回复。

    - `channels.discord.streaming` 控制预览流式传输（`off` | `partial` | `block` | `progress`，默认：`off`）。
    - 默认保持为 `off`，因为 Discord 预览编辑很容易触发速率限制，尤其是在多个机器人或 Gateway 网关共享同一账户或服务器流量时。
    - `progress` 为了跨渠道一致性而被接受，并在 Discord 上映射为 `partial`。
    - `channels.discord.streamMode` 是旧版别名，会被自动迁移。
    - `partial` 会在 token 到达时编辑一条预览消息。
    - `block` 会输出草稿大小的分块（使用 `draftChunk` 调整大小和断点）。

    示例：

```json5
{
  channels: {
    discord: {
      streaming: "partial",
    },
  },
}
```

    `block` 模式的分块默认值（受 `channels.discord.textChunkLimit` 限制）：

```json5
{
  channels: {
    discord: {
      streaming: "block",
      draftChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph",
      },
    },
  },
}
```

    预览流式传输仅支持文本；媒体回复会回退到正常投递。

    注意：预览流式传输与分块流式传输是分开的。当 Discord 明确启用分块流式传输时，OpenClaw 会跳过预览流，以避免双重流式传输。

  </Accordion>

  <Accordion title="历史、上下文和线程行为">
    服务器历史上下文：

    - `channels.discord.historyLimit` 默认值为 `20`
    - 回退：`messages.groupChat.historyLimit`
    - `0` 表示禁用

    私信历史控制：

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    线程行为：

    - Discord 线程会作为渠道会话进行路由
    - 父线程元数据可用于父会话链接
    - 线程配置会继承父渠道配置，除非存在线程专用条目

    渠道主题会作为**不受信任**的上下文注入（而不是作为系统提示词）。
    回复和引用消息上下文当前会按接收时原样保留。
    Discord allowlist 主要用于限制谁可以触发智能体，而不是完整的补充上下文脱敏边界。

  </Accordion>

  <Accordion title="用于子智能体的线程绑定会话">
    Discord 可以将线程绑定到某个会话目标，这样该线程中的后续消息会持续路由到同一个会话（包括子智能体会话）。

    命令：

    - `/focus <target>` 将当前/新线程绑定到子智能体/会话目标
    - `/unfocus` 移除当前线程绑定
    - `/agents` 显示活动运行和绑定状态
    - `/session idle <duration|off>` 检查/更新聚焦绑定的非活动自动取消聚焦设置
    - `/session max-age <duration|off>` 检查/更新聚焦绑定的硬性最大存活时长

    配置：

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // opt-in
      },
    },
  },
}
```

    说明：

    - `session.threadBindings.*` 设置全局默认值。
    - `channels.discord.threadBindings.*` 覆盖 Discord 行为。
    - 必须将 `spawnSubagentSessions` 设为 true，才能为 `sessions_spawn({ thread: true })` 自动创建/绑定线程。
    - 必须将 `spawnAcpSessions` 设为 true，才能为 ACP（`/acp spawn ... --thread ...` 或 `sessions_spawn({ runtime: "acp", thread: true })`）自动创建/绑定线程。
    - 如果某个账户的线程绑定被禁用，则 `/focus` 和相关线程绑定操作不可用。

    参见[子智能体](/zh-CN/tools/subagents)、[ACP Agents](/zh-CN/tools/acp-agents) 和[配置参考](/zh-CN/gateway/configuration-reference)。

  </Accordion>

  <Accordion title="持久 ACP 渠道绑定">
    对于稳定的“始终在线” ACP 工作区，可配置顶层类型化 ACP 绑定，将其定向到 Discord 会话。

    配置路径：

    - `bindings[]`，其中 `type: "acp"` 且 `match.channel: "discord"`

    示例：

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    说明：

    - `/acp spawn codex --bind here` 会就地绑定当前 Discord 渠道或线程，并让未来消息持续路由到同一个 ACP 会话。
    - 这仍然可能意味着“启动一个新的 Codex ACP 会话”，但它本身不会创建新的 Discord 线程。现有渠道仍是聊天界面。
    - Codex 仍可能在其自己的 `cwd` 或磁盘上的后端工作区中运行。该工作区属于运行时状态，而不是 Discord 线程。
    - 线程消息可以继承父渠道的 ACP 绑定。
    - 在已绑定的渠道或线程中，`/new` 和 `/reset` 会就地重置同一个 ACP 会话。
    - 临时线程绑定仍然有效，并可在生效期间覆盖目标解析。
    - 只有当 OpenClaw 需要通过 `--thread auto|here` 创建/绑定子线程时，才需要 `spawnAcpSessions`。对于当前渠道中的 `/acp spawn ... --bind here`，则不需要。

    有关绑定行为的详细信息，请参见 [ACP Agents](/zh-CN/tools/acp-agents)。

  </Accordion>

  <Accordion title="反应通知">
    每服务器的反应通知模式：

    - `off`
    - `own`（默认）
    - `all`
    - `allowlist`（使用 `guilds.<id>.users`）

    反应事件会转换为系统事件并附加到已路由的 Discord 会话。

  </Accordion>

  <Accordion title="确认反应">
    `ackReaction` 会在 OpenClaw 处理入站消息时发送一个确认 emoji。

    解析顺序：

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - 智能体身份 emoji 回退（`agents.list[].identity.emoji`，否则为 `"👀"`）

    说明：

    - Discord 接受 unicode emoji 或自定义 emoji 名称。
    - 使用 `""` 可为某个渠道或账户禁用该反应。

  </Accordion>

  <Accordion title="配置写入">
    默认启用由渠道发起的配置写入。

    这会影响 `/config set|unset` 流程（当启用命令功能时）。

    禁用：

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Gateway 网关代理">
    通过 `channels.discord.proxy` 将 Discord Gateway 网关 WebSocket 流量和启动 REST 查找（应用 ID + allowlist 解析）路由到 HTTP(S) 代理。

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    每账户覆盖：

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="PluralKit 支持">
    启用 PluralKit 解析，以将代理消息映射到系统成员身份：

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // optional; needed for private systems
      },
    },
  },
}
```

    说明：

    - allowlist 可使用 `pk:<memberId>`
    - 成员显示名称仅会在 `channels.discord.dangerouslyAllowNameMatching: true` 时按名称/slug 匹配
    - 查找会使用原始消息 ID，并受时间窗口限制
    - 如果查找失败，代理消息会被视为机器人消息并丢弃，除非设置 `allowBots=true`

  </Accordion>

  <Accordion title="Presence 配置">
    当你设置状态或活动字段，或启用自动 presence 时，会应用 presence 更新。

    仅状态示例：

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    活动示例（自定义状态是默认活动类型）：

```json5
{
  channels: {
    discord: {
      activity: "Focus time",
      activityType: 4,
    },
  },
}
```

    Streaming 示例：

```json5
{
  channels: {
    discord: {
      activity: "Live coding",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    活动类型映射：

    - 0：Playing
    - 1：Streaming（需要 `activityUrl`）
    - 2：Listening
    - 3：Watching
    - 4：Custom（使用活动文本作为状态内容；emoji 可选）
    - 5：Competing

    自动 presence 示例（运行时健康信号）：

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token exhausted",
      },
    },
  },
}
```

    自动 presence 将运行时可用性映射为 Discord 状态：healthy => online，degraded 或 unknown => idle，exhausted 或 unavailable => dnd。可选文本覆盖：

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText`（支持 `{reason}` 占位符）

  </Accordion>

  <Accordion title="Discord 中的审批">
    Discord 支持在私信中使用基于按钮的审批处理，并且可以选择在来源渠道中发布审批提示。

    配置路径：

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers`（可选；在可能时回退到 `commands.ownerAllowFrom`）
    - `channels.discord.execApprovals.target`（`dm` | `channel` | `both`，默认：`dm`）
    - `agentFilter`、`sessionFilter`、`cleanupAfterResolve`

    当 `enabled` 未设置或为 `"auto"`，且至少可以从 `execApprovals.approvers` 或 `commands.ownerAllowFrom` 解析出一位审批人时，Discord 会自动启用原生 exec 审批。Discord 不会从渠道 `allowFrom`、旧版 `dm.allowFrom` 或私信 `defaultTo` 推断 exec 审批人。将 `enabled: false` 设置为显式禁用 Discord 作为原生审批客户端。

    当 `target` 为 `channel` 或 `both` 时，审批提示会显示在渠道中。只有已解析的审批人可以使用按钮；其他用户会收到临时拒绝消息。审批提示包含命令文本，因此只应在受信任渠道中启用渠道投递。如果无法从会话键推导出渠道 ID，OpenClaw 会回退到私信投递。

    Discord 也会渲染其他聊天渠道使用的共享审批按钮。原生 Discord 适配器主要补充审批人私信路由和渠道扇出。
    当这些按钮存在时，它们是主要的审批 UX；只有当工具结果表明聊天审批不可用，或手动审批是唯一途径时，OpenClaw 才应包含手动 `/approve` 命令。

    此处理器的 Gateway 网关鉴权使用与其他 Gateway 网关客户端相同的共享凭证解析契约：

    - env 优先的本地鉴权（`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`，然后是 `gateway.auth.*`）
    - 在本地模式下，仅当 `gateway.auth.*` 未设置时，`gateway.remote.*` 才可作为回退使用；已配置但无法解析的本地 SecretRef 会以安全关闭方式失败
    - 在适用时通过 `gateway.remote.*` 支持远程模式
    - URL 覆盖是安全覆盖的：CLI 覆盖不会复用隐式凭证，而 env 覆盖仅使用 env 凭证

    审批解析行为：

    - 以 `plugin:` 为前缀的 ID 通过 `plugin.approval.resolve` 解析。
    - 其他 ID 通过 `exec.approval.resolve` 解析。
    - Discord 不会在这里额外执行 exec 到 plugin 的回退跳转；ID 前缀决定它调用哪个 Gateway 网关方法。

    Exec 审批默认在 30 分钟后过期。如果审批因未知审批 ID 而失败，请检查审批人解析、功能启用状态，以及已投递的审批 ID 类型是否与待处理请求匹配。

    相关文档：[Exec approvals](/zh-CN/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## 工具和操作门控

Discord 消息操作包括消息发送、渠道管理、审核、presence 和元数据操作。

核心示例：

- 消息：`sendMessage`、`readMessages`、`editMessage`、`deleteMessage`、`threadReply`
- 反应：`react`、`reactions`、`emojiList`
- 审核：`timeout`、`kick`、`ban`
- presence：`setPresence`

`event-create` 操作接受一个可选的 `image` 参数（URL 或本地文件路径），用于设置计划事件封面图。

操作门控位于 `channels.discord.actions.*` 下。

默认门控行为：

| 操作组 | 默认值 |
| --- | --- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | enabled |
| roles | disabled |
| moderation | disabled |
| presence | disabled |

## Components v2 UI

OpenClaw 为 exec 审批和跨上下文标记使用 Discord components v2。Discord 消息操作也可以接受 `components` 用于自定义 UI（高级功能；需要通过 discord 工具构造组件载荷），而旧版 `embeds` 仍可用，但不推荐使用。

- `channels.discord.ui.components.accentColor` 设置 Discord 组件容器使用的强调色（hex）。
- 每账户可通过 `channels.discord.accounts.<id>.ui.components.accentColor` 设置。
- 当存在 components v2 时，会忽略 `embeds`。

示例：

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## 语音渠道

OpenClaw 可以加入 Discord 语音渠道，进行实时、连续的对话。这与语音消息附件是分开的。

要求：

- 启用原生命令（`commands.native` 或 `channels.discord.commands.native`）。
- 配置 `channels.discord.voice`。
- 机器人需要在目标语音渠道中具有 Connect + Speak 权限。

使用仅限 Discord 的原生命令 `/vc join|leave|status` 来控制会话。该命令使用账户默认智能体，并遵循与其他 Discord 命令相同的 allowlist 和 group policy 规则。

自动加入示例：

```json5
{
  channels: {
    discord: {
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
    },
  },
}
```

说明：

- `voice.tts` 仅覆盖语音播放的 `messages.tts`。
- 语音转写轮次从 Discord `allowFrom`（或 `dm.allowFrom`）派生 owner 状态；非 owner 发言者不能访问仅限 owner 的工具（例如 `gateway` 和 `cron`）。
- 默认启用语音；设置 `channels.discord.voice.enabled=false` 可禁用。
- `voice.daveEncryption` 和 `voice.decryptionFailureTolerance` 会直通传递给 `@discordjs/voice` 的加入选项。
- 如果未设置，`@discordjs/voice` 默认值为 `daveEncryption=true` 和 `decryptionFailureTolerance=24`。
- OpenClaw 还会监视接收端解密失败，并在短时间窗口内发生重复失败时，通过离开/重新加入语音渠道来自动恢复。
- 如果接收日志反复显示 `DecryptionFailed(UnencryptedWhenPassthroughDisabled)`，则这可能是上游 `@discordjs/voice` 接收问题，详见 [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419)。

## 语音消息

Discord 语音消息会显示波形预览，并要求使用 OGG/Opus 音频及元数据。OpenClaw 会自动生成波形，但它需要在 Gateway 网关主机上可用的 `ffmpeg` 和 `ffprobe` 来检查和转换音频文件。

要求与限制：

- 提供**本地文件路径**（不接受 URL）。
- 省略文本内容（Discord 不允许在同一载荷中同时包含文本和语音消息）。
- 接受任意音频格式；必要时 OpenClaw 会将其转换为 OGG/Opus。

示例：

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## 故障排除

<AccordionGroup>
  <Accordion title="使用了不允许的 intents，或机器人看不到服务器消息">

    - 启用 Message Content Intent
    - 当你依赖 user/member 解析时，启用 Server Members Intent
    - 更改 intents 后重启 Gateway 网关

  </Accordion>

  <Accordion title="服务器消息被意外阻止">

    - 检查 `groupPolicy`
    - 检查 `channels.discord.guilds` 下的服务器 allowlist
    - 如果存在服务器 `channels` 映射，则仅允许列出的渠道
    - 检查 `requireMention` 行为和 mention 模式

    实用检查：

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="Require mention 为 false 但仍然被阻止">
    常见原因：

    - `groupPolicy="allowlist"` 但没有匹配的服务器/渠道 allowlist
    - `requireMention` 配置在了错误位置（必须位于 `channels.discord.guilds` 或渠道条目下）
    - 发送者被服务器/渠道 `users` allowlist 阻止

  </Accordion>

  <Accordion title="长时间运行的处理器超时或重复回复">

    典型日志：

    - `Listener DiscordMessageListener timed out after 30000ms for event MESSAGE_CREATE`
    - `Slow listener detected ...`
    - `discord inbound worker timed out after ...`

    监听器预算调节项：

    - 单账户：`channels.discord.eventQueue.listenerTimeout`
    - 多账户：`channels.discord.accounts.<accountId>.eventQueue.listenerTimeout`

    Worker 运行超时调节项：

    - 单账户：`channels.discord.inboundWorker.runTimeoutMs`
    - 多账户：`channels.discord.accounts.<accountId>.inboundWorker.runTimeoutMs`
    - 默认值：`1800000`（30 分钟）；设为 `0` 可禁用

    推荐基线：

```json5
{
  channels: {
    discord: {
      accounts: {
        default: {
          eventQueue: {
            listenerTimeout: 120000,
          },
          inboundWorker: {
            runTimeoutMs: 1800000,
          },
        },
      },
    },
  },
}
```

    对于较慢的监听器设置，使用 `eventQueue.listenerTimeout`；只有当你希望为排队的智能体轮次单独设置安全阀时，才使用 `inboundWorker.runTimeoutMs`。

  </Accordion>

  <Accordion title="权限审计不匹配">
    `channels status --probe` 权限检查仅对数字渠道 ID 有效。

    如果你使用 slug 键，运行时匹配仍然可能有效，但探测无法完整验证权限。

  </Accordion>

  <Accordion title="私信和配对问题">

    - 私信已禁用：`channels.discord.dm.enabled=false`
    - 私信策略已禁用：`channels.discord.dmPolicy="disabled"`（旧版：`channels.discord.dm.policy`）
    - 在 `pairing` 模式下等待配对批准

  </Accordion>

  <Accordion title="机器人到机器人的循环">
    默认情况下，会忽略由机器人撰写的消息。

    如果你设置了 `channels.discord.allowBots=true`，请使用严格的 mention 和 allowlist 规则来避免循环行为。
    优先使用 `channels.discord.allowBots="mentions"`，仅接受提及该机器人的机器人消息。

  </Accordion>

  <Accordion title="语音 STT 因 DecryptionFailed(...) 丢失">

    - 保持 OpenClaw 为最新版本（`openclaw update`），以确保具备 Discord 语音接收恢复逻辑
    - 确认 `channels.discord.voice.daveEncryption=true`（默认）
    - 从 `channels.discord.voice.decryptionFailureTolerance=24`（上游默认值）开始，仅在需要时调整
    - 观察日志：
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - 如果自动重新加入后仍持续失败，请收集日志并对照 [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419)

  </Accordion>
</AccordionGroup>

## 配置参考指针

主要参考：

- [配置参考 - Discord](/zh-CN/gateway/configuration-reference#discord)

高信号 Discord 字段：

- 启动/鉴权：`enabled`、`token`、`accounts.*`、`allowBots`
- 策略：`groupPolicy`、`dm.*`、`guilds.*`、`guilds.*.channels.*`
- 命令：`commands.native`、`commands.useAccessGroups`、`configWrites`、`slashCommand.*`
- 事件队列：`eventQueue.listenerTimeout`（监听器预算）、`eventQueue.maxQueueSize`、`eventQueue.maxConcurrency`
- 入站 worker：`inboundWorker.runTimeoutMs`
- 回复/历史：`replyToMode`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`
- 投递：`textChunkLimit`、`chunkMode`、`maxLinesPerMessage`
- 流式传输：`streaming`（旧版别名：`streamMode`）、`draftChunk`、`blockStreaming`、`blockStreamingCoalesce`
- 媒体/重试：`mediaMaxMb`、`retry`
  - `mediaMaxMb` 限制 Discord 出站上传大小（默认：`100MB`）
- 操作：`actions.*`
- presence：`activity`、`status`、`activityType`、`activityUrl`
- UI：`ui.components.accentColor`
- 功能：`threadBindings`、顶层 `bindings[]`（`type: "acp"`）、`pluralkit`、`execApprovals`、`intents`、`agentComponents`、`heartbeat`、`responsePrefix`

## 安全与运维

- 将机器人令牌视为机密信息（在受监管环境中优先使用 `DISCORD_BOT_TOKEN`）。
- 授予最小权限的 Discord 权限。
- 如果命令部署/状态陈旧，请重启 Gateway 网关，并使用 `openclaw channels status --probe` 重新检查。

## 相关内容

- [配对](/zh-CN/channels/pairing)
- [群组](/zh-CN/channels/groups)
- [渠道路由](/zh-CN/channels/channel-routing)
- [安全](/zh-CN/gateway/security)
- [多智能体路由](/zh-CN/concepts/multi-agent)
- [故障排除](/zh-CN/channels/troubleshooting)
- [斜杠命令](/zh-CN/tools/slash-commands)
