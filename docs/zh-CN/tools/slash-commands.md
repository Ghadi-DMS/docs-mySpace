---
read_when:
    - 使用或配置聊天命令时
    - 调试命令路由或权限时
summary: Slash commands：文本与原生命令、配置和支持的命令
title: Slash Commands
x-i18n:
    generated_at: "2026-04-05T17:50:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 13c6a4648d48e47469e7b2c16f135b3e58e3c39f859960df882c7af370c083bd
    source_path: tools/slash-commands.md
    workflow: 15
---

# Slash Commands

命令由 Gateway 网关处理。大多数命令必须作为一条**独立**消息发送，并且以 `/` 开头。
仅主机可用的 bash 聊天命令使用 `! <cmd>`（`/bash <cmd>` 是别名）。

这里有两个相关系统：

- **命令**：独立的 `/...` 消息。
- **指令**：`/think`、`/fast`、`/verbose`、`/reasoning`、`/elevated`、`/exec`、`/model`、`/queue`。
  - 指令会在模型看到消息前从消息中剥离。
  - 在普通聊天消息中（不是纯指令消息），它们会被视为“内联提示”，且**不会**持久化会话设置。
  - 在纯指令消息中（消息只包含指令），它们会持久化到会话，并回复确认信息。
  - 指令仅对**已授权发送者**生效。如果设置了 `commands.allowFrom`，它就是唯一使用的允许列表；否则授权来自渠道允许列表/配对以及 `commands.useAccessGroups`。
    未授权发送者会看到这些指令被当作纯文本处理。

还有少量**内联快捷命令**（仅允许列表/已授权发送者可用）：`/help`、`/commands`、`/status`、`/whoami`（`/id`）。
它们会立即运行，在模型看到消息之前被剥离，剩余文本则继续走正常流程。

## 配置

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: false,
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text`（默认 `true`）启用在聊天消息中解析 `/...`。
  - 在没有原生命令的界面上（WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams），即使你将其设为 `false`，文本命令仍然有效。
- `commands.native`（默认 `"auto"`）注册原生命令。
  - 自动：Discord/Telegram 开启；Slack 关闭（直到你添加 slash commands）；对不支持原生命令的提供商则忽略。
  - 设置 `channels.discord.commands.native`、`channels.telegram.commands.native` 或 `channels.slack.commands.native` 可按提供商覆盖（布尔值或 `"auto"`）。
  - `false` 会在启动时清除之前已在 Discord/Telegram 上注册的命令。Slack 命令在 Slack 应用中管理，不会被自动移除。
- `commands.nativeSkills`（默认 `"auto"`）在支持时以原生方式注册 **skill** 命令。
  - 自动：Discord/Telegram 开启；Slack 关闭（Slack 要求每个 skill 单独创建一个 slash command）。
  - 设置 `channels.discord.commands.nativeSkills`、`channels.telegram.commands.nativeSkills` 或 `channels.slack.commands.nativeSkills` 可按提供商覆盖（布尔值或 `"auto"`）。
- `commands.bash`（默认 `false`）启用 `! <cmd>` 来运行主机 shell 命令（`/bash <cmd>` 是别名；需要 `tools.elevated` 允许列表）。
- `commands.bashForegroundMs`（默认 `2000`）控制 bash 在切换到后台模式前等待多久（`0` 表示立即转后台）。
- `commands.config`（默认 `false`）启用 `/config`（读取/写入 `openclaw.json`）。
- `commands.mcp`（默认 `false`）启用 `/mcp`（读取/写入位于 `mcp.servers` 下、由 OpenClaw 管理的 MCP 配置）。
- `commands.plugins`（默认 `false`）启用 `/plugins`（插件发现/状态，以及安装 + 启用/禁用控制）。
- `commands.debug`（默认 `false`）启用 `/debug`（仅运行时覆盖）。
- `commands.allowFrom`（可选）为命令授权设置按提供商划分的允许列表。配置后，它将成为命令和指令的唯一授权来源（渠道允许列表/配对以及 `commands.useAccessGroups` 会被忽略）。使用 `"*"` 作为全局默认值；按提供商的键会覆盖它。
- `commands.useAccessGroups`（默认 `true`）在未设置 `commands.allowFrom` 时，对命令强制执行允许列表/策略。

## 命令列表

文本 + 原生（启用时）：

- `/help`
- `/commands`
- `/tools [compact|verbose]`（显示当前智能体此刻可用的内容；`verbose` 会增加说明）
- `/skill <name> [input]`（按名称运行一个 skill）
- `/status`（显示当前状态；在可用时包含当前模型提供商的 provider 使用量/配额）
- `/tasks`（列出当前会话的后台任务；显示活跃和最近任务详情，以及智能体本地回退计数）
- `/allowlist`（列出/添加/移除允许列表条目）
- `/approve <id> <decision>`（处理 exec 审批提示；可用决策请使用待处理审批消息）
- `/context [list|detail|json]`（解释“上下文”；`detail` 显示每个文件 + 每个工具 + 每个 skill + system prompt 大小）
- `/btw <question>`（针对当前会话提出一个临时旁路问题，而不改变未来的会话上下文；见 [/tools/btw](/zh-CN/tools/btw)）
- `/export-session [path]`（别名：`/export`）（将当前会话导出为包含完整 system prompt 的 HTML）
- `/whoami`（显示你的发送者 ID；别名：`/id`）
- `/session idle <duration|off>`（管理聚焦线程绑定的空闲自动取消聚焦）
- `/session max-age <duration|off>`（管理聚焦线程绑定的硬性最大时长自动取消聚焦）
- `/subagents list|kill|log|info|send|steer|spawn`（检查、控制或为当前会话生成子智能体运行）
- `/acp spawn|cancel|steer|close|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|sessions`（检查并控制 ACP 运行时会话）
- `/agents`（列出此会话绑定到线程的智能体）
- `/focus <target>`（Discord：将此线程或一个新线程绑定到某个会话/子智能体目标）
- `/unfocus`（Discord：移除当前线程绑定）
- `/kill <id|#|all>`（立即中止此会话中一个或全部正在运行的子智能体；无确认消息）
- `/steer <id|#> <message>`（立即引导一个正在运行的子智能体：若可能则在运行中调整，否则中止当前工作并基于引导消息重启）
- `/tell <id|#> <message>`（`/steer` 的别名）
- `/config show|get|set|unset`（将配置持久化到磁盘，仅 owner；需要 `commands.config: true`）
- `/mcp show|get|set|unset`（管理 OpenClaw MCP 服务器配置，仅 owner；需要 `commands.mcp: true`）
- `/plugins list|show|get|install|enable|disable`（检查已发现插件、安装新插件并切换启用状态；写操作仅 owner 可用；需要 `commands.plugins: true`）
  - `/plugin` 是 `/plugins` 的别名。
  - `/plugin install <spec>` 接受与 `openclaw plugins install` 相同的插件规格：本地路径/归档、npm 包或 `clawhub:<pkg>`。
  - 启用/禁用写操作仍会回复重启提示。在受监视的前台 Gateway 网关上，OpenClaw 可能会在写入后立即自动执行该重启。
- `/debug show|set|unset|reset`（运行时覆盖，仅 owner；需要 `commands.debug: true`）
- `/usage off|tokens|full|cost`（每次回复的用量页脚或本地成本摘要）
- `/tts off|always|inbound|tagged|status|provider|limit|summary|audio`（控制 TTS；见 [/tts](/zh-CN/tools/tts)）
  - Discord：原生命令是 `/voice`（Discord 保留 `/tts`）；文本 `/tts` 仍可使用。
- `/stop`
- `/restart`
- `/dock-telegram`（别名：`/dock_telegram`）（将回复切换到 Telegram）
- `/dock-discord`（别名：`/dock_discord`）（将回复切换到 Discord）
- `/dock-slack`（别名：`/dock_slack`）（将回复切换到 Slack）
- `/activation mention|always`（仅群组）
- `/send on|off|inherit`（仅 owner）
- `/reset` 或 `/new [model]`（可选模型提示；剩余内容会继续传递）
- `/think <off|minimal|low|medium|high|xhigh>`（按模型/提供商动态提供选项；别名：`/thinking`、`/t`）
- `/fast status|on|off`（省略参数时显示当前实际 fast-mode 状态）
- `/verbose on|full|off`（别名：`/v`）
- `/reasoning on|off|stream`（别名：`/reason`；开启时会发送一条以 `Reasoning:` 为前缀的独立消息；`stream` = 仅 Telegram 草稿）
- `/elevated on|off|ask|full`（别名：`/elev`；`full` 会跳过 exec 审批）
- `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>`（发送 `/exec` 可查看当前状态）
- `/model <name>`（别名：`/models`；或 `agents.defaults.models.*.alias` 中定义的 `/<alias>`）
- `/queue <mode>`（以及类似 `debounce:2s cap:25 drop:summarize` 的选项；发送 `/queue` 可查看当前设置）
- `/bash <command>`（仅主机；`! <command>` 的别名；需要 `commands.bash: true` + `tools.elevated` 允许列表）
- `/dreaming [on|off|status|help]` 或 `/dreaming [enable|disable] [light|deep|rem]`（切换 dreaming 阶段或显示状态；见 [Dreaming](/concepts/dreaming)）

仅文本：

- `/compact [instructions]`（见 [/concepts/compaction](/zh-CN/concepts/compaction)）
- `! <command>`（仅主机；一次只能一个；长任务请使用 `!poll` + `!stop`）
- `!poll`（检查输出/状态；接受可选 `sessionId`；`/bash poll` 也可用）
- `!stop`（停止正在运行的 bash 任务；接受可选 `sessionId`；`/bash stop` 也可用）

说明：

- 命令在命令与参数之间可接受可选的 `:`（例如 `/think: high`、`/send: on`、`/help:`）。
- `/new <model>` 接受模型别名、`provider/model` 或提供商名称（模糊匹配）；如果没有匹配，文本会被当作消息正文处理。
- 要查看完整的提供商使用量明细，请使用 `openclaw status --usage`。
- `/allowlist add|remove` 需要 `commands.config=true`，并遵循渠道 `configWrites`。
- 在多账户渠道中，面向配置的 `/allowlist --account <id>` 和 `/config set channels.<provider>.accounts.<id>...` 也会遵循目标账户的 `configWrites`。
- `/usage` 控制每次回复的用量页脚；`/usage cost` 会根据 OpenClaw 会话日志输出本地成本摘要。
- `/restart` 默认启用；设置 `commands.restart: false` 可禁用。
- 仅 Discord 原生命令：`/vc join|leave|status` 控制语音频道（需要 `channels.discord.voice` 和原生命令；无文本形式）。
- Discord 线程绑定命令（`/focus`、`/unfocus`、`/agents`、`/session idle`、`/session max-age`）需要有效启用线程绑定（`session.threadBindings.enabled` 和/或 `channels.discord.threadBindings.enabled`）。
- ACP 命令参考和运行时行为： [ACP Agents](/zh-CN/tools/acp-agents)。
- `/verbose` 旨在用于调试和额外可见性；正常使用时请保持**关闭**。
- `/fast on|off` 会持久化一个会话覆盖。使用 Sessions UI 的 `inherit` 选项可清除它并回退到配置默认值。
- `/fast` 是提供商特定的：OpenAI/OpenAI Codex 会在原生 Responses 端点上将其映射为 `service_tier=priority`，而直接的公共 Anthropic 请求（包括发送到 `api.anthropic.com` 的 OAuth 认证流量）会将其映射为 `service_tier=auto` 或 `standard_only`。参见 [OpenAI](/zh-CN/providers/openai) 和 [Anthropic](/zh-CN/providers/anthropic)。
- 与工具失败相关的摘要仍会在适用时显示，但详细失败文本仅在 `/verbose` 为 `on` 或 `full` 时包含。
- `/reasoning`（以及 `/verbose`）在群组场景中风险较高：它们可能会暴露你原本无意公开的内部推理或工具输出。建议保持关闭，尤其是在群聊中。
- `/model` 会立即持久化新的会话模型。
- 如果智能体处于空闲状态，下一次运行会立即使用它。
- 如果某次运行已经活跃，OpenClaw 会将实时切换标记为待处理，并且只会在一个干净的重试点重启到新模型。
- 如果工具活动或回复输出已经开始，待处理切换可能会继续排队，直到稍后的重试机会或下一次用户轮次。
- **快速路径：** 来自允许列表发送者的纯命令消息会被立即处理（绕过队列 + 模型）。
- **群组提及门控：** 来自允许列表发送者的纯命令消息会绕过提及要求。
- **内联快捷命令（仅允许列表发送者）：** 某些命令即使嵌入在普通消息中也能工作，并会在模型看到剩余文本前被剥离。
  - 示例：`hey /status` 会触发状态回复，而剩余文本会继续走正常流程。
- 当前支持：`/help`、`/commands`、`/status`、`/whoami`（`/id`）。
- 未授权的纯命令消息会被静默忽略，而内联 `/...` 标记会被当作普通文本。
- **Skill 命令：** `user-invocable` Skills 会以 slash commands 暴露。名称会被规范化为 `a-z0-9_`（最多 32 个字符）；冲突时会加数字后缀（例如 `_2`）。
  - `/skill <name> [input]` 按名称运行一个 skill（当原生命令限制阻止按 skill 暴露命令时很有用）。
  - 默认情况下，skill 命令会作为普通请求转发给模型。
  - Skills 可选择声明 `command-dispatch: tool`，以便将命令直接路由给工具（确定性、无模型）。
  - 示例：`/prose`（OpenProse 插件）——参见 [OpenProse](/zh-CN/prose)。
- **原生命令参数：** Discord 对动态选项使用自动补全（当你省略必需参数时也会使用按钮菜单）。Telegram 和 Slack 会在某个命令支持选项且你省略参数时显示按钮菜单。

## `/tools`

`/tools` 回答的是运行时问题，而不是配置问题：**此智能体在当前对话中此刻能使用什么**。

- 默认 `/tools` 是紧凑视图，便于快速浏览。
- `/tools verbose` 会增加简短说明。
- 支持参数的原生命令界面会暴露相同的 `compact|verbose` 模式切换。
- 结果是会话级的，因此更改智能体、渠道、线程、发送者授权或模型，都可能改变输出。
- `/tools` 包含运行时实际可达的工具，包括核心工具、已连接的插件工具以及归属于渠道的工具。

对于配置档案和覆盖编辑，请使用 Control UI Tools 面板或配置/目录界面，而不要把 `/tools` 当作静态目录。

## 使用量界面（各处显示内容）

- **提供商使用量/配额**（例如：“Claude 剩余 80%”）会在启用使用量跟踪时显示在当前模型提供商的 `/status` 中。OpenClaw 会将提供商窗口规范化为“剩余 %”；对于 MiniMax，显示前会将仅剩余百分比字段反转，而 `model_remains` 响应会优先使用聊天模型条目加带模型标签的计划标签。
- `/status` 中的 **token/cache 行** 可以在实时会话快照较稀疏时回退到最新 transcript 使用量条目。现有的非零实时值仍然优先，而 transcript 回退还可以在已存储总量缺失或更小时恢复活跃运行时模型标签以及更偏向提示词的总量。
- **每次回复的 token/成本** 由 `/usage off|tokens|full` 控制（附加到普通回复后）。
- `/model status` 关注的是**模型/凭证/端点**，不是使用量。

## 模型选择（`/model`）

`/model` 是作为一条指令实现的。

示例：

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

说明：

- `/model` 和 `/model list` 会显示一个紧凑的编号选择器（模型家族 + 可用提供商）。
- 在 Discord 上，`/model` 和 `/models` 会打开一个交互式选择器，其中包含提供商和模型下拉菜单，以及一个提交步骤。
- `/model <#>` 会从该选择器中选择（并在可能时优先当前提供商）。
- `/model status` 会显示详细视图，包括已配置的提供商端点（`baseUrl`）和 API 模式（`api`），如果可用。

## 调试覆盖

`/debug` 允许你设置**仅运行时**的配置覆盖（在内存中，而不是磁盘上）。仅 owner 可用。默认禁用；通过 `commands.debug: true` 启用。

示例：

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

说明：

- 覆盖会立即应用于新的配置读取，但**不会**写入 `openclaw.json`。
- 使用 `/debug reset` 清除所有覆盖，并恢复到磁盘上的配置。

## 配置更新

`/config` 会写入你磁盘上的配置（`openclaw.json`）。仅 owner 可用。默认禁用；通过 `commands.config: true` 启用。

示例：

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

说明：

- 写入前会验证配置；无效变更会被拒绝。
- `/config` 更新会在重启后保留。

## MCP 更新

`/mcp` 会将由 OpenClaw 管理的 MCP 服务器定义写入 `mcp.servers` 下。仅 owner 可用。默认禁用；通过 `commands.mcp: true` 启用。

示例：

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

说明：

- `/mcp` 将配置存储在 OpenClaw 配置中，而不是 Pi 拥有的项目设置中。
- 运行时适配器决定哪些传输实际上可执行。

## 插件更新

`/plugins` 允许操作员检查已发现插件并在配置中切换启用状态。只读流程可使用 `/plugin` 作为别名。默认禁用；通过 `commands.plugins: true` 启用。

示例：

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

说明：

- `/plugins list` 和 `/plugins show` 会针对当前工作区加磁盘上的配置执行真实的插件发现。
- `/plugins enable|disable` 仅更新插件配置；不会安装或卸载插件。
- 启用/禁用变更后，请重启 Gateway 网关以应用变更。

## 界面说明

- **文本命令** 在普通聊天会话中运行（私信共享 `main`，群组有各自的会话）。
- **原生命令** 使用隔离会话：
  - Discord：`agent:<agentId>:discord:slash:<userId>`
  - Slack：`agent:<agentId>:slack:slash:<userId>`（前缀可通过 `channels.slack.slashCommand.sessionPrefix` 配置）
  - Telegram：`telegram:slash:<userId>`（通过 `CommandTargetSessionKey` 定位聊天会话）
- **`/stop`** 会针对活跃聊天会话，从而中止当前运行。
- **Slack：** `channels.slack.slashCommand` 仍然支持单个 `/openclaw` 风格命令。如果你启用了 `commands.native`，则必须为每个内置命令在 Slack 中分别创建一个 slash command（名称与 `/help` 相同）。Slack 的命令参数菜单以临时 Block Kit 按钮形式传递。
  - Slack 原生命令例外：注册 `/agentstatus`（而不是 `/status`），因为 Slack 保留 `/status`。文本 `/status` 在 Slack 消息中仍可使用。

## BTW 旁路问题

`/btw` 是一个关于当前会话的快速**旁路问题**。

与普通聊天不同：

- 它使用当前会话作为背景上下文，
- 它以单独的**无工具**一次性调用运行，
- 它不会改变未来的会话上下文，
- 它不会写入 transcript 历史，
- 它会作为实时旁路结果交付，而不是普通助手消息。

这使得 `/btw` 在你希望主任务继续进行的同时，临时获得某个澄清时非常有用。

示例：

```text
/btw what are we doing right now?
```

完整行为和客户端 UX 细节请参见 [BTW Side Questions](/zh-CN/tools/btw)。
