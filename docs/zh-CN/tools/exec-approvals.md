---
read_when:
    - 配置 exec 审批或允许列表
    - 在 macOS 应用中实现 exec 审批 UX
    - 审查沙箱逃逸提示及其影响
summary: exec 审批、允许列表和沙箱逃逸提示
title: Exec 审批
x-i18n:
    generated_at: "2026-04-05T17:50:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: 39e91cd5c7615bdb9a6b201a85bde7514327910f6f12da5a4b0532bceb229c22
    source_path: tools/exec-approvals.md
    workflow: 15
---

# Exec 审批

Exec 审批是**配套应用 / 节点宿主机的防护栏**，用于让处于沙箱中的智能体在真实宿主机（`gateway` 或 `node`）上运行命令。你可以把它看作一种安全联锁：只有当策略 + 允许列表 +（可选）用户审批都一致同意时，命令才会被允许。
Exec 审批是**附加在**工具策略和提升执行门槛之上的（除非提升执行设置为 `full`，此时会跳过审批）。
生效策略取 `tools.exec.*` 与审批默认值中**更严格**的一方；如果某个审批字段被省略，则使用 `tools.exec` 中的对应值。
宿主机 exec 还会使用该机器上的本地审批状态。宿主机本地
`~/.openclaw/exec-approvals.json` 中设置的 `ask: "always"` 会持续提示，即使会话或配置默认值请求的是 `ask: "on-miss"`。
使用 `openclaw approvals get`、`openclaw approvals get --gateway` 或
`openclaw approvals get --node <id|name|ip>` 可检查请求的策略、宿主机策略来源以及最终生效结果。

如果配套应用 UI **不可用**，任何需要提示的请求都会由 **ask 回退**来处理（默认：拒绝）。

原生聊天审批客户端还可以在待处理审批消息上提供渠道特定的交互方式。例如，Matrix 可以在审批提示上预置反应快捷方式（`✅` 允许一次、`❌` 拒绝，以及在可用时的 `♾️` 永久允许），同时仍将消息中的 `/approve ...` 命令保留为回退方式。

## 适用位置

Exec 审批会在执行宿主机本地强制执行：

- **gateway 宿主机** → Gateway 网关机器上的 `openclaw` 进程
- **node 宿主机** → 节点运行器（macOS 配套应用或无头节点宿主机）

信任模型说明：

- 通过 Gateway 网关认证的调用方，是该 Gateway 网关的受信任操作员。
- 已配对节点会将这种受信任操作员能力扩展到节点宿主机上。
- Exec 审批可降低意外执行的风险，但它不是按用户划分的认证边界。
- 已批准的节点宿主机运行会绑定规范化执行上下文：规范化 `cwd`、精确 `argv`、存在时的环境变量绑定，以及在适用时固定的可执行文件路径。
- 对于 shell 脚本和直接解释器 / 运行时文件调用，OpenClaw 还会尝试绑定一个具体的本地文件操作数。如果该绑定文件在审批后、执行前发生变化，则运行会被拒绝，而不会执行发生漂移的内容。
- 这种文件绑定是有意设计的尽力而为机制，而不是对所有解释器 / 运行时加载器路径的完整语义模型。如果审批模式无法识别出恰好一个具体本地文件进行绑定，它会拒绝生成基于审批的运行，而不是假装提供了完整覆盖。

macOS 拆分：

- **node 宿主机服务** 通过本地 IPC 将 `system.run` 转发给 **macOS 应用**。
- **macOS 应用** 在 UI 上下文中强制执行审批并执行命令。

## 设置与存储

审批存储在执行宿主机上的本地 JSON 文件中：

`~/.openclaw/exec-approvals.json`

示例 schema：

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

## 无审批的 “YOLO” 模式

如果你希望宿主机 exec 在没有审批提示的情况下运行，你必须同时放开**两层**策略：

- OpenClaw 配置中的请求 exec 策略（`tools.exec.*`）
- `~/.openclaw/exec-approvals.json` 中的宿主机本地审批策略

除非你显式收紧，否则这现在是默认的宿主机行为：

- `tools.exec.security`：在 `gateway` / `node` 上为 `full`
- `tools.exec.ask`：`off`
- 宿主机 `askFallback`：`full`

重要区别：

- `tools.exec.host=auto` 选择 exec 在哪里运行：有沙箱时在沙箱中，否则在 gateway 上。
- YOLO 决定宿主机 exec 如何被审批：`security=full` 加 `ask=off`。
- 在 YOLO 模式下，OpenClaw 不会在已配置的宿主机 exec 策略之上，再额外添加独立的启发式命令混淆审批门槛。
- `auto` 不会让来自沙箱会话的 gateway 路由成为免费的覆盖项。按次调用的 `host=node` 请求在 `auto` 下是允许的；只有当没有激活沙箱运行时时，`host=gateway` 才能从 `auto` 使用。如果你想要稳定的非 auto 默认值，请设置 `tools.exec.host` 或显式使用 `/exec host=...`。

如果你希望采用更保守的设置，请将任一层收紧回 `allowlist` / `on-miss` 或 `deny`。

持久化的 gateway 宿主机“永不提示”设置：

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
openclaw gateway restart
```

然后将宿主机审批文件设置为匹配值：

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

对于 node 宿主机，请在该节点上应用相同的审批文件：

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

仅限当前会话的快捷方式：

- `/exec security=full ask=off` 只会更改当前会话。
- `/elevated full` 是一个紧急逃生快捷方式，也会跳过该会话的 exec 审批。

如果宿主机审批文件仍比配置更严格，则更严格的宿主机策略仍然优先。

## 策略旋钮

### 安全级别（`exec.security`）

- **deny**：阻止所有宿主机 exec 请求。
- **allowlist**：仅允许允许列表中的命令。
- **full**：允许所有内容（等同于提升执行）。

### Ask（`exec.ask`）

- **off**：从不提示。
- **on-miss**：仅当允许列表不匹配时提示。
- **always**：每条命令都提示。
- 当生效的 ask 模式为 `always` 时，`allow-always` 的持久信任不会抑制提示

### Ask fallback（`askFallback`）

如果需要提示但无法访问 UI，则由回退决定：

- **deny**：阻止。
- **allowlist**：仅在允许列表匹配时允许。
- **full**：允许。

### 内联解释器 eval 加固（`tools.exec.strictInlineEval`）

当 `tools.exec.strictInlineEval=true` 时，即使解释器二进制本身已被加入允许列表，OpenClaw 也会将内联代码 eval 形式视为仅能通过审批执行。

示例：

- `python -c`
- `node -e`, `node --eval`, `node -p`
- `ruby -e`
- `perl -e`, `perl -E`
- `php -r`
- `lua -e`
- `osascript -e`

这是针对无法干净映射到单一稳定文件操作数的解释器加载器的一种纵深防御。在严格模式下：

- 这些命令仍然需要显式审批；
- `allow-always` 不会自动为它们持久化新的允许列表条目。

## 允许列表（按智能体）

允许列表是**按智能体**划分的。如果存在多个智能体，请在 macOS 应用中切换你正在编辑的智能体。模式采用**不区分大小写的 glob 匹配**。
模式应解析为**二进制路径**（仅 basename 的条目会被忽略）。
旧版 `agents.default` 条目会在加载时迁移到 `agents.main`。
像 `echo ok && pwd` 这样的 shell 链式命令，仍需要每个顶层片段都满足允许列表规则。

示例：

- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

每个允许列表条目会跟踪：

- **id**：用于 UI 身份的稳定 UUID（可选）
- **最后使用**时间戳
- **最后使用的命令**
- **最后解析路径**

## 自动允许 Skills CLI

启用 **Auto-allow Skills CLIs** 后，已知 Skills 引用的可执行文件会在节点上（macOS 节点或无头节点宿主机）被视为已加入允许列表。此功能通过 Gateway 网关 RPC 使用 `skills.bins` 获取 Skill 二进制列表。如果你希望严格使用手动允许列表，请禁用它。

重要信任说明：

- 这是一个**隐式的便捷允许列表**，与手动路径允许列表条目分离。
- 它面向 Gateway 网关与节点处于同一信任边界内的受信任操作员环境。
- 如果你需要严格的显式信任，请保持 `autoAllowSkills: false`，并仅使用手动路径允许列表条目。

## Safe bins（仅 stdin）

`tools.exec.safeBins` 定义了一小组**仅 stdin** 的二进制文件（例如 `cut`），它们可以在允许列表模式下**无需**显式允许列表条目运行。Safe bins 会拒绝位置文件参数和类似路径的 token，因此它们只能处理输入流。
请将其视为流过滤器的狭义快速路径，而不是通用信任列表。
**不要**将解释器或运行时二进制文件（例如 `python3`、`node`、`ruby`、`bash`、`sh`、`zsh`）添加到 `safeBins`。
如果某个命令天然可以执行代码、执行子命令或读取文件，请优先使用显式允许列表条目，并保持审批提示启用。
自定义 safe bin 必须在 `tools.exec.safeBinProfiles.<bin>` 中定义显式配置文件。
校验仅根据 `argv` 形状进行确定性判断（不检查宿主机文件系统中的存在性），这可以避免 allow / deny 差异造成的文件存在性预言机行为。
默认 safe bins 会拒绝面向文件的选项（例如 `sort -o`、`sort --output`、
`sort --files0-from`、`sort --compress-program`、`sort --random-source`、
`sort --temporary-directory` / `-T`、`wc --files0-from`、`jq -f/--from-file`、
`grep -f/--file`）。
Safe bins 还会对破坏仅 stdin 行为的选项强制执行显式的按二进制 flag 策略（例如 `sort -o/--output/--compress-program` 和 grep 的递归 flag）。
在 safe-bin 模式下，长选项会按失败即关闭的方式校验：未知 flag 和含糊缩写都会被拒绝。
被 safe-bin 配置文件拒绝的 flag：

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`：`--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`：`--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`：`--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `wc`：`--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Safe bins 还会在执行时强制将 `argv` token 视为**字面文本**（不进行 glob 展开，也不展开 `$VARS`），适用于仅 stdin 的片段，因此像 `*` 或 `$HOME/...` 这样的模式不能被用来夹带文件读取。
Safe bins 还必须从受信任的二进制目录中解析（系统默认目录，加上可选的 `tools.exec.safeBinTrustedDirs`）。
`PATH` 条目绝不会自动被视为可信。
默认受信任的 safe-bin 目录被有意保持最小：`/bin`、`/usr/bin`。
如果你的 safe-bin 可执行文件位于包管理器 / 用户路径（例如
`/opt/homebrew/bin`、`/usr/local/bin`、`/opt/local/bin`、`/snap/bin`），请将这些路径显式添加到 `tools.exec.safeBinTrustedDirs`。
在允许列表模式下，shell 链接和重定向不会被自动允许。

当每个顶层片段都满足允许列表条件时（包括 safe bins 或 Skill 自动允许），允许使用 shell 链接（`&&`、`||`、`;`）。在允许列表模式下，重定向仍不受支持。
在允许列表解析期间，命令替换（`$()` / 反引号）会被拒绝，包括在双引号内部；如果你需要字面量 `$()` 文本，请使用单引号。
在 macOS 配套应用审批中，包含 shell 控制或展开语法（`&&`、`||`、`;`、`|`、`` ` ``、`$`、`<`、`>`、`(`、`)`）的原始 shell 文本会被视为允许列表未命中，除非 shell 二进制本身已加入允许列表。
对于 shell 包装器（`bash|sh|zsh ... -c/-lc`），按请求作用域的环境变量覆盖会被缩减为一个较小的显式允许列表（`TERM`、`LANG`、`LC_*`、`COLORTERM`、`NO_COLOR`、`FORCE_COLOR`）。
对于允许列表模式下的 allow-always 决策，已知的分发包装器
（`env`、`nice`、`nohup`、`stdbuf`、`timeout`）会持久化内部可执行文件路径，而不是包装器路径。
Shell 多路复用器（`busybox`、`toybox`）也会针对 shell applet（`sh`、`ash` 等）进行解包，因此会持久化内部可执行文件，而不是多路复用器二进制本身。如果某个包装器或多路复用器无法被安全解包，则不会自动持久化任何允许列表条目。
如果你将像 `python3` 或 `node` 这样的解释器加入允许列表，建议同时启用 `tools.exec.strictInlineEval=true`，这样内联 eval 仍然需要显式审批。在严格模式下，`allow-always` 仍可以持久化无害的解释器 / 脚本调用，但内联 eval 载体不会被自动持久化。

默认 safe bins：

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` 和 `sort` 不在默认列表中。如果你选择启用它们，请为其非 stdin 工作流保留显式允许列表条目。
对于 safe-bin 模式下的 `grep`，请使用 `-e` / `--regexp` 提供模式；位置参数形式的模式会被拒绝，以防文件操作数被伪装成有歧义的位置参数。

### Safe bins 与允许列表的区别

| 主题 | `tools.exec.safeBins` | 允许列表（`exec-approvals.json`） |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| 目标 | 自动允许狭义的 stdin 过滤器 | 显式信任特定可执行文件 |
| 匹配类型 | 可执行文件名 + safe-bin `argv` 策略 | 解析后的可执行文件路径 glob 模式 |
| 参数范围 | 受 safe-bin 配置文件和字面 token 规则限制 | 仅匹配路径；其余参数由你自行负责 |
| 典型示例 | `head`、`tail`、`tr`、`wc` | `jq`、`python3`、`node`、`ffmpeg`、自定义 CLI |
| 最佳用途 | 管道中的低风险文本转换 | 任何行为更广泛或具有副作用的工具 |

配置位置：

- `safeBins` 来自配置（`tools.exec.safeBins` 或按智能体的 `agents.list[].tools.exec.safeBins`）。
- `safeBinTrustedDirs` 来自配置（`tools.exec.safeBinTrustedDirs` 或按智能体的 `agents.list[].tools.exec.safeBinTrustedDirs`）。
- `safeBinProfiles` 来自配置（`tools.exec.safeBinProfiles` 或按智能体的 `agents.list[].tools.exec.safeBinProfiles`）。按智能体的配置文件键会覆盖全局键。
- 允许列表条目存储在宿主机本地 `~/.openclaw/exec-approvals.json` 的 `agents.<id>.allowlist` 下（或通过 Control UI / `openclaw approvals allowlist ...` 管理）。
- 当解释器 / 运行时二进制出现在 `safeBins` 中但没有显式配置文件时，`openclaw security audit` 会发出 `tools.exec.safe_bins_interpreter_unprofiled` 警告。
- `openclaw doctor --fix` 可以为缺失的自定义 `safeBinProfiles.<bin>` 条目生成 `{}` 脚手架（之后请审查并收紧）。解释器 / 运行时二进制不会被自动生成脚手架。

自定义配置文件示例：
__OC_I18N_900004__
如果你显式将 `jq` 加入 `safeBins`，OpenClaw 在 safe-bin 模式下仍会拒绝 `env` 内建，因此 `jq -n env` 无法在没有显式允许列表路径或审批提示的情况下导出宿主机进程环境。

## Control UI 编辑

使用 **Control UI → Nodes → Exec approvals** 卡片编辑默认值、按智能体覆盖和允许列表。选择一个范围（默认值或某个智能体），调整策略，添加 / 删除允许列表模式，然后点击**保存**。UI 会按模式显示**最后使用**元数据，方便你保持列表整洁。

目标选择器可选择 **Gateway**（本地审批）或 **Node**。节点必须声明 `system.execApprovals.get/set`（macOS 应用或无头节点宿主机）。
如果某个节点尚未声明 exec 审批，请直接编辑其本地
`~/.openclaw/exec-approvals.json`。

CLI：`openclaw approvals` 支持编辑 gateway 或 node（请参阅 [Approvals CLI](/cli/approvals)）。

## 审批流程

当需要提示时，Gateway 网关会向操作员客户端广播 `exec.approval.requested`。
Control UI 和 macOS 应用通过 `exec.approval.resolve` 进行处理，然后 Gateway 网关将已批准的请求转发到节点宿主机。

对于 `host=node`，审批请求会包含一个规范化的 `systemRunPlan` 负载。Gateway 网关在转发已批准的 `system.run` 请求时，会将该计划作为权威的命令 / `cwd` / 会话上下文。

这对于异步审批延迟很重要：

- 节点 exec 路径会预先准备一个规范计划
- 审批记录会存储该计划及其绑定元数据
- 一旦获批，最终转发的 `system.run` 调用会复用已存储计划，
  而不是信任调用方之后的编辑
- 如果调用方在审批请求创建后更改了 `command`、`rawCommand`、`cwd`、`agentId` 或
  `sessionKey`，Gateway 网关会将转发运行拒绝为审批不匹配

## 解释器 / 运行时命令

基于审批的解释器 / 运行时执行会刻意保持保守：

- 始终绑定精确的 `argv` / `cwd` / `env` 上下文。
- 对于直接 shell 脚本和直接运行时文件形式，会尽力绑定到一个具体的本地文件快照。
- 仍可解析为一个直接本地文件的常见包管理器包装形式（例如
  `pnpm exec`、`pnpm node`、`npm exec`、`npx`）会在绑定前先被解包。
- 如果 OpenClaw 无法为某个解释器 / 运行时命令识别出恰好一个具体本地文件（例如包脚本、eval 形式、特定运行时的加载链，或有歧义的多文件形式），则会拒绝基于审批的执行，而不是声称提供了并不存在的语义覆盖。
- 对于这些工作流，优先考虑使用沙箱隔离、单独的宿主机边界，或显式的受信任 allowlist / full 工作流，由操作员接受更宽泛的运行时语义。

当需要审批时，exec 工具会立即返回一个审批 ID。使用该 ID 可以关联后续系统事件（`Exec finished` / `Exec denied`）。如果在超时前没有收到决定，请求会被视为审批超时，并作为拒绝原因暴露出来。

### 后续投递行为

在已批准的异步 exec 完成后，OpenClaw 会向同一会话发送一个后续 `agent` turn。

- 如果存在有效的外部投递目标（可投递渠道加目标 `to`），后续投递会使用该渠道。
- 在仅 webchat 或仅内部会话且没有外部目标的流程中，后续投递会保持为仅会话（`deliver: false`）。
- 如果调用方显式请求严格的外部投递，但无法解析出外部渠道，请求会以 `INVALID_REQUEST` 失败。
- 如果启用了 `bestEffortDeliver`，且无法解析外部渠道，则投递会降级为仅会话，而不是失败。

确认对话框包含：

- 命令 + 参数
- `cwd`
- 智能体 ID
- 已解析的可执行文件路径
- 宿主机 + 策略元数据

操作：

- **Allow once** → 立即运行
- **Always allow** → 添加到允许列表并运行
- **Deny** → 阻止

## 将审批转发到聊天渠道

你可以将 exec 审批提示转发到任意聊天渠道（包括插件渠道），并通过 `/approve` 进行批准。它使用正常的出站投递流水线。

配置：
__OC_I18N_900005__
在聊天中回复：
__OC_I18N_900006__
`/approve` 命令同时处理 exec 审批和插件审批。如果该 ID 不匹配待处理的 exec 审批，它会自动转而检查插件审批。

### 插件审批转发

插件审批转发使用与 exec 审批相同的投递流水线，但在 `approvals.plugin` 下拥有独立配置。启用或禁用其中之一不会影响另一个。
__OC_I18N_900007__
配置结构与 `approvals.exec` 完全一致：`enabled`、`mode`、`agentFilter`、
`sessionFilter` 和 `targets` 的工作方式相同。

支持共享交互式回复的渠道会为 exec 审批和插件审批渲染相同的审批按钮。不支持共享交互式 UI 的渠道则会回退为包含 `/approve` 说明的纯文本。

### 任意渠道中的同聊天审批

当 exec 或插件审批请求来自可投递的聊天表面时，默认情况下该聊天本身现在也可以通过 `/approve` 进行审批。这适用于 Slack、Matrix 和 Microsoft Teams 等渠道，也适用于现有的 Web UI 和终端 UI 流程。

这条共享文本命令路径使用该会话的正常渠道认证模型。如果发起聊天本身已经可以发送命令并接收回复，那么审批请求就不再需要单独的原生投递适配器来保持待处理状态。

Discord 和 Telegram 也支持同聊天 `/approve`，但即使原生审批投递被禁用，这些渠道在授权时仍会使用其已解析的审批者列表。

对于 Telegram 和其他直接调用 Gateway 网关的原生审批客户端，这种回退会被有意限制在“未找到审批”失败的场景中。真实的 exec 审批拒绝 / 错误不会被静默重试为插件审批。

### 原生审批投递

有些渠道还可以充当原生审批客户端。原生客户端会在共享的同聊天 `/approve` 流程之上，增加审批者私信、来源聊天扇出以及渠道特定的交互式审批 UX。

当原生审批卡片 / 按钮可用时，该原生 UI 就是智能体面对的主要路径。除非工具结果表明聊天审批不可用，或手动审批是唯一剩余路径，否则智能体不应再额外回显重复的纯聊天 `/approve` 命令。

通用模型：

- 宿主机 exec 策略仍决定是否需要 exec 审批
- `approvals.exec` 控制是否将审批提示转发到其他聊天目标
- `channels.<channel>.execApprovals` 控制该渠道是否充当原生审批客户端

当满足以下全部条件时，原生审批客户端会自动启用优先私信投递：

- 该渠道支持原生审批投递
- 可以通过显式 `execApprovals.approvers` 或该渠道文档记录的回退来源解析出审批者
- `channels.<channel>.execApprovals.enabled` 未设置或为 `"auto"`

设置 `enabled: false` 可显式禁用原生审批客户端。设置 `enabled: true` 可在审批者可解析时强制启用。公开的来源聊天投递仍通过
`channels.<channel>.execApprovals.target` 显式控制。

常见问题：[为什么聊天审批有两个 exec 审批配置？](/help/faq#why-are-there-two-exec-approval-configs-for-chat-approvals)

- Discord：`channels.discord.execApprovals.*`
- Slack：`channels.slack.execApprovals.*`
- Telegram：`channels.telegram.execApprovals.*`

这些原生审批客户端会在共享的同聊天 `/approve` 流程和共享审批按钮之上，增加私信路由和可选的渠道扇出。

共享行为：

- Slack、Matrix、Microsoft Teams 以及类似的可投递聊天，会对同聊天 `/approve` 使用正常的渠道认证模型
- 当原生审批客户端自动启用时，默认的原生投递目标是审批者私信
- 对于 Discord 和 Telegram，只有已解析的审批者可以批准或拒绝
- Discord 审批者可以是显式的（`execApprovals.approvers`），也可以从 `commands.ownerAllowFrom` 推断
- Telegram 审批者可以是显式的（`execApprovals.approvers`），也可以从现有 owner 配置（`allowFrom`，以及在支持时的私信 `defaultTo`）推断
- Slack 审批者可以是显式的（`execApprovals.approvers`），也可以从 `commands.ownerAllowFrom` 推断
- Slack 原生按钮会保留审批 ID 类型，因此 `plugin:` ID 可以解析为插件审批，而无需第二层 Slack 本地回退
- Matrix 原生私信 / 渠道路由仅适用于 exec；Matrix 插件审批仍停留在共享的同聊天 `/approve` 和可选的 `approvals.plugin` 转发路径上
- 请求者不需要是审批者
- 当来源聊天本身已支持命令和回复时，它可以直接通过 `/approve` 进行审批
- 原生 Discord 审批按钮会按审批 ID 类型路由：`plugin:` ID 会直接进入插件审批，其他所有内容都会进入 exec 审批
- 原生 Telegram 审批按钮遵循与 `/approve` 相同的有界 exec 到插件回退逻辑
- 当原生 `target` 启用来源聊天投递时，审批提示会包含命令文本
- 待处理的 exec 审批默认会在 30 分钟后过期
- 如果没有任何操作员 UI 或已配置的审批客户端能够接受请求，则提示会回退到 `askFallback`

Telegram 默认投递到审批者私信（`target: "dm"`）。当你希望审批提示也出现在发起的 Telegram 聊天 / 话题中时，可以切换为 `channel` 或 `both`。对于 Telegram forum topics，OpenClaw 会保留该话题，用于审批提示和审批后的后续消息。

请参阅：

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)

### macOS IPC 流程
__OC_I18N_900008__
安全说明：

- Unix socket 模式为 `0600`，token 存储在 `exec-approvals.json` 中。
- 同 UID 对端检查。
- 质询 / 响应（nonce + HMAC token + 请求哈希）+ 短 TTL。

## 系统事件

Exec 生命周期会作为系统消息呈现：

- `Exec running`（仅当命令超过运行通知阈值时）
- `Exec finished`
- `Exec denied`

这些消息会在节点上报事件后发送到智能体的会话中。
gateway 宿主机 exec 审批在命令完成时（以及可选地在运行超过阈值时）也会发出相同的生命周期事件。
受审批门控的 exec 会复用审批 ID 作为这些消息中的 `runId`，便于关联。

## 审批被拒绝时的行为

当异步 exec 审批被拒绝时，OpenClaw 会阻止智能体在该会话中复用此前相同命令的任何早期运行输出。拒绝原因会带有明确说明，指出没有可用的命令输出，从而防止智能体声称存在新的输出，或使用先前成功运行留下的陈旧结果重复被拒绝的命令。

## 影响

- **full** 权限很强；尽可能优先使用允许列表。
- **ask** 让你保持在决策回路中，同时仍允许快速审批。
- 按智能体划分的允许列表可防止一个智能体的审批泄露到其他智能体。
- 审批仅适用于来自**已授权发送者**的宿主机 exec 请求。未授权发送者不能发出 `/exec`。
- `/exec security=full` 是为已授权操作员提供的会话级便捷方式，并且按设计会跳过审批。
  如果要硬性阻止宿主机 exec，请将审批安全级别设为 `deny`，或通过工具策略拒绝 `exec` 工具。

相关内容：

- [Exec 工具](/zh-CN/tools/exec)
- [提升执行模式](/zh-CN/tools/elevated)
- [Skills](/zh-CN/tools/skills)

## 相关

- [Exec](/zh-CN/tools/exec) — shell 命令执行工具
- [沙箱隔离](/zh-CN/gateway/sandboxing) — 沙箱模式和工作区访问
- [安全](/zh-CN/gateway/security) — 安全模型与加固
- [沙箱 vs 工具策略 vs 提升执行](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated) — 何时使用各项机制
