---
read_when:
    - 添加或修改 doctor 迁移时
    - 引入破坏性配置变更时
summary: Doctor 命令：健康检查、配置迁移和修复步骤
title: Doctor
x-i18n:
    generated_at: "2026-04-05T17:48:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: f20637d373c3b1be7c4a3d5424228908b5639cfded8ad5acb9c29f882d0f8d2b
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` 是 OpenClaw 的修复 + 迁移工具。它会修复过时的配置/状态、检查健康状况，并提供可执行的修复步骤。

## 快速开始

```bash
openclaw doctor
```

### 无头 / 自动化

```bash
openclaw doctor --yes
```

接受默认值而不提示（包括在适用时的重启/服务/沙箱修复步骤）。

```bash
openclaw doctor --repair
```

无需提示即可应用推荐修复（在安全情况下执行修复 + 重启）。

```bash
openclaw doctor --repair --force
```

也应用激进修复（会覆盖自定义 supervisor 配置）。

```bash
openclaw doctor --non-interactive
```

无提示运行，并且只应用安全迁移（配置规范化 + 磁盘状态移动）。会跳过需要人工确认的重启/服务/沙箱操作。
检测到旧版状态迁移时会自动运行。

```bash
openclaw doctor --deep
```

扫描系统服务以查找额外的 Gateway 网关安装（launchd/systemd/schtasks）。

如果你想在写入前先检查变更，请先打开配置文件：

```bash
cat ~/.openclaw/openclaw.json
```

## 它会做什么（摘要）

- 针对 git 安装的可选预检更新（仅交互模式）。
- UI 协议新鲜度检查（当协议 schema 更新时重建 Control UI）。
- 健康检查 + 重启提示。
- Skills 状态摘要（可用/缺失/被阻止）和插件状态。
- 旧版值的配置规范化。
- 将旧版扁平 `talk.*` 字段迁移为 `talk.provider` + `talk.providers.<provider>` 的 Talk 配置迁移。
- 针对旧版 Chrome 扩展配置和 Chrome MCP 就绪情况的浏览器迁移检查。
- OpenCode 提供商覆盖警告（`models.providers.opencode` / `models.providers.opencode-go`）。
- OpenAI Codex OAuth 配置档案的 OAuth TLS 前置条件检查。
- 旧版磁盘状态迁移（sessions/agent 目录/WhatsApp 凭证）。
- 旧版插件清单 contract 键迁移（`speechProviders`、`realtimeTranscriptionProviders`、`realtimeVoiceProviders`、`mediaUnderstandingProviders`、`imageGenerationProviders`、`videoGenerationProviders`、`webFetchProviders`、`webSearchProviders` → `contracts`）。
- 旧版 cron 存储迁移（`jobId`、`schedule.cron`、顶层 delivery/payload 字段、payload `provider`、简单的 `notify: true` webhook 回退任务）。
- 会话锁文件检查和过时锁清理。
- 状态完整性和权限检查（sessions、transcripts、state 目录）。
- 本地运行时的配置文件权限检查（chmod 600）。
- 模型凭证健康检查：检查 OAuth 过期情况，可刷新即将过期的 token，并报告 auth-profile 冷却/禁用状态。
- 检测额外的工作区目录（`~/openclaw`）。
- 启用沙箱隔离时的沙箱镜像修复。
- 旧版服务迁移和额外 Gateway 网关检测。
- Matrix 渠道旧版状态迁移（在 `--fix` / `--repair` 模式下）。
- Gateway 网关运行时检查（服务已安装但未运行；缓存的 launchd 标签）。
- 渠道状态警告（从正在运行的 Gateway 网关探测）。
- Supervisor 配置审计（launchd/systemd/schtasks）及可选修复。
- Gateway 网关运行时最佳实践检查（Node 与 Bun、版本管理器路径）。
- Gateway 网关端口冲突诊断（默认 `18789`）。
- 开放私信策略的安全警告。
- 本地 token 模式的 Gateway 网关认证检查（在没有 token 来源时提供生成 token；不会覆盖 token SecretRef 配置）。
- Linux 上的 systemd linger 检查。
- 工作区 bootstrap 文件大小检查（上下文文件的截断/接近限制警告）。
- Shell 补全状态检查和自动安装/升级。
- 记忆搜索 embedding 提供商就绪情况检查（本地模型、远程 API 密钥或 QMD 二进制）。
- 源码安装检查（pnpm workspace 不匹配、缺失 UI 资源、缺失 tsx 二进制）。
- 写入更新后的配置 + 向导元数据。

## 详细行为与原理

### 0）可选更新（git 安装）

如果这是一个 git checkout，且 doctor 在交互模式下运行，它会先提供在运行 doctor 之前进行更新（fetch/rebase/build）的选项。

### 1）配置规范化

如果配置中包含旧版值形态（例如没有渠道特定覆盖的 `messages.ackReaction`），doctor 会将其规范化为当前 schema。

这也包括旧版 Talk 扁平字段。当前公开的 Talk 配置是
`talk.provider` + `talk.providers.<provider>`。Doctor 会将旧的
`talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` 形态重写到提供商映射中。

### 2）旧版配置键迁移

当配置中包含已弃用键时，其他命令会拒绝运行，并要求你运行
`openclaw doctor`。

Doctor 将会：

- 解释发现了哪些旧版键。
- 显示它应用的迁移。
- 用更新后的 schema 重写 `~/.openclaw/openclaw.json`。

当 Gateway 网关在启动时检测到旧版配置格式，也会自动运行 doctor 迁移，因此无需手动干预即可修复过时配置。
Cron 任务存储迁移由 `openclaw doctor --fix` 处理。

当前迁移包括：

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → 顶层 `bindings`
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- 旧版 `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>`（`openai`/`elevenlabs`/`microsoft`/`edge`）→ `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>`（`openai`/`elevenlabs`/`microsoft`/`edge`）→ `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>`（`openai`/`elevenlabs`/`microsoft`/`edge`）→ `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>`（`openai`/`elevenlabs`/`microsoft`/`edge`）→ `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- 对于具有命名 `accounts` 但仍保留单账户顶层渠道值的渠道，将这些账户范围的值移动到为该渠道选定的提升账户中（大多数渠道使用 `accounts.default`；Matrix 可以保留现有的匹配命名/默认目标）
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*`（tools/elevated/exec/sandbox/subagents）
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- 移除 `browser.relayBindHost`（旧版扩展 relay 设置）

Doctor 警告还包括多账户渠道的默认账户指导：

- 如果配置了两个或更多 `channels.<channel>.accounts` 条目，但没有 `channels.<channel>.defaultAccount` 或 `accounts.default`，doctor 会警告回退路由可能选中意外的账户。
- 如果 `channels.<channel>.defaultAccount` 被设置为未知账户 ID，doctor 会发出警告并列出已配置的账户 ID。

### 2b）OpenCode 提供商覆盖

如果你手动添加了 `models.providers.opencode`、`opencode-zen` 或 `opencode-go`，
它会覆盖来自 `@mariozechner/pi-ai` 的内置 OpenCode 目录。
这可能会强制模型走错误的 API，或将成本归零。Doctor 会发出警告，以便你移除该覆盖并恢复按模型的 API 路由 + 成本。

### 2c）浏览器迁移和 Chrome MCP 就绪情况

如果你的浏览器配置仍指向已移除的 Chrome 扩展路径，doctor 会将其规范化为当前的主机本地 Chrome MCP 附加模型：

- `browser.profiles.*.driver: "extension"` 会变为 `"existing-session"`
- `browser.relayBindHost` 会被移除

当你使用 `defaultProfile: "user"` 或已配置的 `existing-session` 配置档案时，doctor 还会审计主机本地 Chrome MCP 路径：

- 检查 Google Chrome 是否已安装在同一主机上，用于默认自动连接配置档案
- 检查检测到的 Chrome 版本，并在其低于 Chrome 144 时发出警告
- 提醒你在浏览器 inspect 页面中启用远程调试（例如
  `chrome://inspect/#remote-debugging`、`brave://inspect/#remote-debugging`，
  或 `edge://inspect/#remote-debugging`）

Doctor 无法替你启用 Chrome 侧设置。主机本地 Chrome MCP 仍然需要：

- Gateway 网关/节点主机上安装 144+ 的 Chromium 内核浏览器
- 浏览器在本地运行
- 在该浏览器中启用远程调试
- 在浏览器中批准首次附加授权提示

这里的就绪情况仅涉及本地附加前置条件。Existing-session 会保留当前 Chrome MCP 的路由限制；像 `responsebody`、PDF 导出、下载拦截和批处理操作这样的高级路由，仍然需要受管浏览器或原始 CDP 配置档案。

此检查**不**适用于 Docker、沙箱、remote-browser 或其他无头流程。这些流程将继续使用原始 CDP。

### 2d）OAuth TLS 前置条件

当配置了 OpenAI Codex OAuth 配置档案时，doctor 会探测 OpenAI 授权端点，以验证本地 Node/OpenSSL TLS 栈是否能够验证证书链。如果探测因证书错误失败（例如 `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`、证书过期或自签名证书），doctor 会输出特定平台的修复指导。在 macOS 上，如果使用的是 Homebrew Node，修复方式通常是 `brew postinstall ca-certificates`。使用 `--deep` 时，即使 Gateway 网关健康，该探测也会运行。

### 3）旧版状态迁移（磁盘布局）

Doctor 可以将较旧的磁盘布局迁移到当前结构：

- Sessions 存储 + transcripts：
  - 从 `~/.openclaw/sessions/` 到 `~/.openclaw/agents/<agentId>/sessions/`
- Agent 目录：
  - 从 `~/.openclaw/agent/` 到 `~/.openclaw/agents/<agentId>/agent/`
- WhatsApp 凭证状态（Baileys）：
  - 从旧版 `~/.openclaw/credentials/*.json`（`oauth.json` 除外）
  - 到 `~/.openclaw/credentials/whatsapp/<accountId>/...`（默认账户 ID：`default`）

这些迁移尽力而为且是幂等的；如果它保留了任何旧版文件夹作为备份，doctor 会发出警告。Gateway 网关/CLI 在启动时也会自动迁移旧版 sessions + agent 目录，因此历史记录/凭证/模型会落到按智能体划分的路径中，而无需手动运行 doctor。WhatsApp 凭证有意仅通过 `openclaw doctor` 迁移。Talk 提供商/提供商映射规范化现在会按结构相等性比较，因此仅键顺序差异不会再触发重复的无操作 `doctor --fix` 更改。

### 3a）旧版插件清单迁移

Doctor 会扫描所有已安装插件清单中的已弃用顶层能力键
（`speechProviders`、`realtimeTranscriptionProviders`、
`realtimeVoiceProviders`、`mediaUnderstandingProviders`、
`imageGenerationProviders`、`videoGenerationProviders`、`webFetchProviders`、
`webSearchProviders`）。发现后，它会提供将其移入 `contracts`
对象并原地重写清单文件的选项。该迁移是幂等的；
如果 `contracts` 键中已经有相同值，就会移除旧版键而不重复数据。

### 3b）旧版 cron 存储迁移

Doctor 还会检查 cron 任务存储（默认是 `~/.openclaw/cron/jobs.json`，
若被覆盖则使用 `cron.store`）中调度器仍为兼容性而接受的旧任务形态。

当前的 cron 清理包括：

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- 顶层 payload 字段（`message`、`model`、`thinking`、...）→ `payload`
- 顶层 delivery 字段（`deliver`、`channel`、`to`、`provider`、...）→ `delivery`
- payload 中的 `provider` delivery 别名 → 显式 `delivery.channel`
- 简单的旧版 `notify: true` webhook 回退任务 → 显式 `delivery.mode="webhook"` 且 `delivery.to=cron.webhook`

Doctor 仅会在能够不改变行为的情况下自动迁移 `notify: true` 任务。
如果某个任务同时包含旧版 notify 回退和现有的非 webhook delivery 模式，doctor 会发出警告，并将该任务留待手动检查。

### 3c）会话锁清理

Doctor 会扫描每个智能体会话目录中的过时写锁文件——这些文件是在会话异常退出时遗留的。对于找到的每个锁文件，它会报告：
路径、PID、该 PID 是否仍存活、锁的存在时间，以及它是否被视为过时（PID 已死或超过 30 分钟）。在 `--fix` / `--repair`
模式下，它会自动删除过时锁文件；否则会输出说明，并指示你使用 `--fix` 重新运行。

### 4）状态完整性检查（会话持久化、路由和安全）

状态目录是运行系统的大脑中枢。如果它消失了，你将丢失
sessions、凭证、日志和配置（除非你在别处有备份）。

Doctor 会检查：

- **状态目录缺失**：警告灾难性的状态丢失，提示重新创建目录，并提醒你它无法恢复丢失的数据。
- **状态目录权限**：验证是否可写；提供修复权限的选项
  （并在检测到所有者/组不匹配时给出 `chown` 提示）。
- **macOS 云同步状态目录**：当状态解析到 iCloud Drive
  （`~/Library/Mobile Documents/com~apple~CloudDocs/...`）或
  `~/Library/CloudStorage/...` 下时发出警告，因为同步支持路径可能导致更慢的 I/O 以及锁/同步竞争。
- **Linux SD 或 eMMC 状态目录**：当状态解析到 `mmcblk*`
  挂载源时发出警告，因为基于 SD 或 eMMC 的随机 I/O 在会话和凭证写入场景中可能更慢且磨损更快。
- **会话目录缺失**：`sessions/` 和会话存储目录是持久化历史记录并避免 `ENOENT` 崩溃所必需的。
- **Transcript 不匹配**：当最近的会话条目缺少 transcript 文件时发出警告。
- **主会话“单行 JSONL”**：当主 transcript 只有一行时标记出来（历史记录没有持续累积）。
- **多个状态目录**：当多个主目录下存在多个 `~/.openclaw` 文件夹，或 `OPENCLAW_STATE_DIR` 指向其他位置时发出警告（历史记录可能在多个安装间分裂）。
- **远程模式提醒**：如果 `gateway.mode=remote`，doctor 会提醒你在远程主机上运行它（状态存储在那里）。
- **配置文件权限**：如果 `~/.openclaw/openclaw.json` 对组/全局可读，则发出警告，并提供将权限收紧到 `600` 的选项。

### 5）模型凭证健康检查（OAuth 过期）

Doctor 会检查凭证存储中的 OAuth 配置档案，在 token 即将过期/已过期时发出警告，并可在安全时刷新它们。如果 Anthropic
OAuth/token 配置档案已过时，它会建议使用 Anthropic API 密钥或旧版 Anthropic setup-token 路径。
刷新提示仅在交互模式（TTY）下出现；`--non-interactive`
会跳过刷新尝试。

Doctor 还会检测已移除的 Anthropic Claude CLI 旧状态。如果旧的
`anthropic:claude-cli` 凭证字节仍存在于 `auth-profiles.json` 中，
doctor 会将其转换回 Anthropic token/OAuth 配置档案，并重写过时的
`claude-cli/...` 模型引用以及 `agents.defaults.cliBackends.claude-cli`。
如果这些字节已不存在，doctor 会移除过时配置并输出恢复命令。

Doctor 还会报告由于以下原因而暂时不可用的 auth profiles：

- 短期冷却（速率限制/超时/凭证失败）
- 较长时间禁用（计费/额度失败）

### 6）Hooks 模型校验

如果设置了 `hooks.gmail.model`，doctor 会根据目录和允许列表验证模型引用，并在其无法解析或不被允许时发出警告。

### 7）沙箱镜像修复

启用沙箱隔离时，doctor 会检查 Docker 镜像，并在当前镜像缺失时提供构建或切换到旧版名称的选项。

### 7b）内置插件运行时依赖

Doctor 会验证内置插件运行时依赖（例如 Discord 插件运行时包）是否存在于 OpenClaw 安装根目录中。
如果缺失，doctor 会报告这些包，并在
`openclaw doctor --fix` / `openclaw doctor --repair` 模式下安装它们。

### 8）Gateway 网关服务迁移和清理提示

Doctor 会检测旧版 Gateway 网关服务（launchd/systemd/schtasks），并提供移除它们以及使用当前 Gateway 网关端口安装 OpenClaw 服务的选项。它还可以扫描额外的类似 Gateway 网关服务并输出清理提示。
按配置档案命名的 OpenClaw Gateway 网关服务被视为一等公民，不会被标记为“额外”。

### 8b）启动 Matrix 迁移

当 Matrix 渠道账户存在待处理或可执行的旧版状态迁移时，
doctor（在 `--fix` / `--repair` 模式下）会先创建迁移前快照，然后运行尽力而为的迁移步骤：旧版 Matrix 状态迁移和旧版加密状态准备。这两个步骤都不是致命的；错误会被记录，启动会继续。在只读模式下（不带 `--fix` 的 `openclaw doctor`），此检查会被完全跳过。

### 9）安全警告

当某个提供商对私信开放但没有允许列表，或某项策略以危险方式配置时，doctor 会发出警告。

### 10）systemd linger（Linux）

如果作为 systemd 用户服务运行，doctor 会确保已启用 lingering，以便 Gateway 网关在注销后仍保持运行。

### 11）工作区状态（Skills、插件和旧版目录）

Doctor 会输出默认智能体的工作区状态摘要：

- **Skills 状态**：统计可用、缺少要求、和被允许列表阻止的 Skills 数量。
- **旧版工作区目录**：当 `~/openclaw` 或其他旧版工作区目录与当前工作区并存时发出警告。
- **插件状态**：统计已加载/已禁用/出错的插件数量；列出所有出错插件的插件 ID；报告内置插件能力。
- **插件兼容性警告**：标记与当前运行时存在兼容性问题的插件。
- **插件诊断**：显示插件注册表在加载时发出的任何警告或错误。

### 11b）Bootstrap 文件大小

Doctor 会检查工作区 bootstrap 文件（例如 `AGENTS.md`、
`CLAUDE.md` 或其他注入的上下文文件）是否接近或超过配置的字符预算。它会报告每个文件的原始字符数与注入字符数、截断百分比、截断原因（`max/file` 或 `max/total`），以及总注入字符数占总预算的比例。当文件被截断或接近限制时，doctor 会给出调优 `agents.defaults.bootstrapMaxChars`
和 `agents.defaults.bootstrapTotalMaxChars` 的建议。

### 11c）Shell 补全

Doctor 会检查当前 shell 是否已安装 Tab 补全
（zsh、bash、fish 或 PowerShell）：

- 如果 shell 配置文件使用了较慢的动态补全模式
  （`source <(openclaw completion ...)`），doctor 会将其升级为更快的缓存文件变体。
- 如果补全已在配置文件中配置，但缓存文件缺失，
  doctor 会自动重新生成缓存。
- 如果完全未配置补全，它会提示你安装
  （仅交互模式；使用 `--non-interactive` 时跳过）。

运行 `openclaw completion --write-state` 可手动重新生成缓存。

### 12）Gateway 网关认证检查（本地 token）

Doctor 会检查本地 Gateway 网关 token 认证的就绪情况。

- 如果 token 模式需要 token 且没有 token 来源，doctor 会提供生成一个 token 的选项。
- 如果 `gateway.auth.token` 由 SecretRef 管理但不可用，doctor 会发出警告，且不会用明文覆盖它。
- 仅当未配置 token SecretRef 时，`openclaw doctor --generate-gateway-token` 才会强制生成。

### 12b）只读的 SecretRef 感知修复

某些修复流程需要检查已配置的凭证，而不能削弱运行时快速失败行为。

- `openclaw doctor --fix` 现在对目标配置修复使用与 status 系列命令相同的只读 SecretRef 摘要模型。
- 示例：Telegram `allowFrom` / `groupAllowFrom` 的 `@username` 修复会在可用时尝试使用已配置的机器人凭证。
- 如果 Telegram bot token 通过 SecretRef 配置，但在当前命令路径中不可用，doctor 会报告该凭证已配置但不可用，并跳过自动解析，而不是崩溃或错误地将 token 报告为缺失。

### 13）Gateway 网关健康检查 + 重启

Doctor 会运行健康检查，并在 Gateway 网关看起来不健康时提供重启选项。

### 13b）记忆搜索就绪情况

Doctor 会检查为默认智能体配置的记忆搜索 embedding 提供商是否已就绪。其行为取决于已配置的后端和提供商：

- **QMD 后端**：探测 `qmd` 二进制是否可用且可启动。
  如果不能，则输出修复指导，包括 npm 包和手动二进制路径选项。
- **显式本地提供商**：检查本地模型文件或已识别的远程/可下载模型 URL。
  如果缺失，则建议切换到远程提供商。
- **显式远程提供商**（`openai`、`voyage` 等）：验证环境中或凭证存储中是否存在 API 密钥。若缺失，则输出可执行的修复提示。
- **自动提供商**：先检查本地模型可用性，再按自动选择顺序尝试每个远程提供商。

当 Gateway 网关探测结果可用时（检查时 Gateway 网关是健康的），doctor 会将其结果与 CLI 可见配置交叉比对，并指出任何不一致之处。

使用 `openclaw memory status --deep` 可在运行时验证 embedding 就绪情况。

### 14）渠道状态警告

如果 Gateway 网关健康，doctor 会运行渠道状态探测，并报告带有建议修复方式的警告。

### 15）Supervisor 配置审计 + 修复

Doctor 会检查已安装的 supervisor 配置（launchd/systemd/schtasks）是否缺少或落后于当前默认值（例如 systemd network-online 依赖项和重启延迟）。当发现不匹配时，它会建议更新，并可将服务文件/任务重写为当前默认值。

说明：

- `openclaw doctor` 会在重写 supervisor 配置前进行提示。
- `openclaw doctor --yes` 会接受默认修复提示。
- `openclaw doctor --repair` 会无提示应用推荐修复。
- `openclaw doctor --repair --force` 会覆盖自定义 supervisor 配置。
- 如果 token 认证需要 token，且 `gateway.auth.token` 由 SecretRef 管理，doctor 的服务安装/修复会验证 SecretRef，但不会将解析后的明文 token 值持久化到 supervisor 服务环境元数据中。
- 如果 token 认证需要 token，而已配置的 token SecretRef 未解析，doctor 会阻止安装/修复路径，并提供可执行的指导。
- 如果同时配置了 `gateway.auth.token` 和 `gateway.auth.password`，而 `gateway.auth.mode` 未设置，doctor 会阻止安装/修复，直到显式设置 mode。
- 对于 Linux user-systemd 单元，doctor 的 token 漂移检查现在在比较服务认证元数据时会同时包含 `Environment=` 和 `EnvironmentFile=` 来源。
- 你始终可以通过 `openclaw gateway install --force` 强制完整重写。

### 16）Gateway 网关运行时 + 端口诊断

Doctor 会检查服务运行时（PID、上次退出状态），并在服务已安装但实际未运行时发出警告。它还会检查 Gateway 网关端口（默认 `18789`）上的端口冲突，并报告可能原因（Gateway 网关已在运行、SSH 隧道）。

### 17）Gateway 网关运行时最佳实践

当 Gateway 网关服务运行在 Bun 上或运行在版本管理的 Node 路径
（`nvm`、`fnm`、`volta`、`asdf` 等）时，doctor 会发出警告。WhatsApp + Telegram 渠道需要 Node，而版本管理器路径在升级后可能失效，因为服务不会加载你的 shell 初始化。Doctor 会在系统 Node 可用时提供迁移到系统 Node 安装的选项（Homebrew/apt/choco）。

### 18）配置写入 + 向导元数据

Doctor 会持久化任何配置变更，并写入向导元数据以记录本次 doctor 运行。

### 19）工作区提示（备份 + 记忆系统）

当工作区缺少记忆系统时，doctor 会提出建议；如果工作区尚未由 git 管理，它还会输出备份提示。

有关工作区结构和 git 备份（推荐使用私有 GitHub 或 GitLab）的完整指南，请参见 [/concepts/agent-workspace](/zh-CN/concepts/agent-workspace)。
