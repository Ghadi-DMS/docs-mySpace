---
read_when:
    - 编辑系统提示文本、工具列表或时间/心跳部分
    - 更改工作区引导或 Skills 注入行为
summary: OpenClaw 系统提示包含什么内容，以及它是如何组装的
title: 系统提示
x-i18n:
    generated_at: "2026-04-12T02:53:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 057f01aac51f7737b5223f61f5d55e552d9011232aebb130426e269d8f6c257f
    source_path: concepts/system-prompt.md
    workflow: 15
---

# 系统提示

OpenClaw 会为每次智能体运行构建一个自定义系统提示。该提示由 **OpenClaw 自有**，不使用 pi-coding-agent 的默认提示。

该提示由 OpenClaw 组装，并注入到每次智能体运行中。

提供商插件可以贡献具备缓存感知能力的提示指导，而无需替换整个
OpenClaw 自有提示。提供商运行时可以：

- 替换一小组具名核心部分（`interaction_style`、
  `tool_call_style`、`execution_bias`）
- 在提示缓存边界上方注入一个**稳定前缀**
- 在提示缓存边界下方注入一个**动态后缀**

对于特定模型家族的调优，请使用由提供商拥有的贡献。保留旧版
`before_prompt_build` 提示变更机制，以用于兼容性或真正的全局提示更改，而不是常规的提供商行为。

## 结构

该提示被有意设计得简洁，并使用固定部分：

- **工具**：结构化工具的事实来源提醒，加上运行时工具使用指导。
- **安全**：简短的护栏提醒，用于避免权力寻求行为或绕过监督。
- **Skills**（可用时）：告诉模型如何按需加载技能说明。
- **OpenClaw 自更新**：如何使用
  `config.schema.lookup` 安全检查配置，使用 `config.patch` 修补配置，使用 `config.apply` 替换完整配置，以及仅在用户明确请求时运行 `update.run`。仅限所有者使用的 `gateway` 工具也会拒绝重写
  `tools.exec.ask` / `tools.exec.security`，包括会规范化到这些受保护 exec 路径的旧版 `tools.bash.*`
  别名。
- **工作区**：工作目录（`agents.defaults.workspace`）。
- **文档**：OpenClaw 文档的本地路径（仓库或 npm 包）以及何时阅读它们。
- **工作区文件（已注入）**：表明引导文件已包含在下方。
- **沙箱**（启用时）：表明运行时处于沙箱隔离中、沙箱路径，以及是否可用提升权限的 exec。
- **当前日期和时间**：用户本地时间、时区和时间格式。
- **回复标签**：适用于受支持提供商的可选回复标签语法。
- **心跳**：心跳提示和确认行为，当默认智能体启用心跳时显示。
- **运行时**：主机、操作系统、node、模型、仓库根目录（检测到时）、思考级别（一行）。
- **推理**：当前可见性级别 + `/reasoning` 切换提示。

“工具”部分还包含针对长时间运行工作的运行时指导：

- 对于未来的后续跟进（`check back later`、提醒、周期性工作），使用 cron，
  而不是 `exec` 睡眠循环、`yieldMs` 延迟技巧或重复的 `process`
  轮询
- `exec` / `process` 仅用于那些现在启动并会继续在后台运行的命令
- 当启用自动完成唤醒时，只启动命令一次，并依赖
  在其输出内容或失败时触发的基于推送的唤醒路径
- 当你需要检查正在运行的命令时，使用 `process` 查看日志、状态、输入或进行干预
- 如果任务更大，优先使用 `sessions_spawn`；子智能体完成是基于推送的，并会自动向请求者回报
- 不要仅为了等待完成而循环轮询 `subagents list` / `sessions_list`

当启用实验性的 `update_plan` 工具时，“工具”部分还会告诉
模型仅将其用于非琐碎的多步骤工作，始终只保留一个
`in_progress` 步骤，并避免在每次更新后重复整个计划。

系统提示中的安全护栏是建议性的。它们引导模型行为，但不强制执行策略。对于硬性约束，请使用工具策略、exec 审批、沙箱隔离和渠道允许列表；运营者可以按设计禁用这些机制。

在具有原生审批卡片/按钮的渠道上，运行时提示现在会告诉
智能体优先依赖该原生审批 UI。只有当工具结果表明聊天审批不可用，或手动审批是唯一可行路径时，它才应包含手动
`/approve` 命令。

## 提示模式

OpenClaw 可以为子智能体渲染更小的系统提示。运行时会为每次运行设置一个
`promptMode`（不是面向用户的配置）：

- `full`（默认）：包含上述所有部分。
- `minimal`：用于子智能体；省略 **Skills**、**Memory Recall**、**OpenClaw
  自更新**、**模型别名**、**用户身份**、**回复标签**、
  **消息传递**、**静默回复** 和 **心跳**。工具、**安全**、
  工作区、沙箱、当前日期和时间（已知时）、运行时以及注入的
  上下文仍然可用。
- `none`：仅返回基础身份行。

当 `promptMode=minimal` 时，额外注入的提示会被标记为 **子智能体
上下文**，而不是 **群聊上下文**。

## 工作区引导注入

引导文件会被裁剪并附加在 **项目上下文** 下，以便模型无需显式读取就能看到身份和配置上下文：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（仅在全新工作区中）
- 存在时使用 `MEMORY.md`，否则回退为小写形式的 `memory.md`

除非有特定文件级别的门控，否则所有这些文件都会在每一轮中**注入到上下文窗口**中。
当默认智能体禁用心跳，或
`agents.defaults.heartbeat.includeSystemPromptSection` 为 false 时，正常运行中会省略 `HEARTBEAT.md`。请保持注入文件简洁——尤其是 `MEMORY.md`，它会随着时间增长，并导致上下文使用量意外升高以及更频繁的压缩。

> **注意：** `memory/*.md` 每日文件**不是**常规引导
> 项目上下文的一部分。在普通轮次中，它们通过
> `memory_search` 和 `memory_get` 工具按需访问，因此除非模型明确读取它们，否则不会计入
> 上下文窗口。裸 `/new` 和 `/reset` 轮次是例外：运行时可以在首次轮次前
> 预置最近的每日记忆，作为一次性的启动上下文块。

大型文件会被截断并加上标记。每个文件的最大大小由
`agents.defaults.bootstrapMaxChars` 控制（默认：20000）。跨文件注入的引导总内容
受 `agents.defaults.bootstrapTotalMaxChars`
限制（默认：150000）。缺失文件会注入简短的缺失文件标记。发生截断时，OpenClaw 可以在项目上下文中注入警告块；可通过
`agents.defaults.bootstrapPromptTruncationWarning` 控制（`off`、`once`、`always`；
默认：`once`）。

子智能体会话只注入 `AGENTS.md` 和 `TOOLS.md`（其他引导文件
会被过滤掉，以保持子智能体上下文较小）。

内部 hooks 可以通过 `agent:bootstrap` 拦截此步骤，以变更或替换
注入的引导文件（例如将 `SOUL.md` 替换为其他 persona）。

如果你想让智能体听起来不那么通用，可以从
[SOUL.md Personality Guide](/zh-CN/concepts/soul) 开始。

要检查每个注入文件分别贡献了多少内容（原始值 vs 注入值、截断情况，以及工具 schema 开销），请使用 `/context list` 或 `/context detail`。参见 [Context](/zh-CN/concepts/context)。

## 时间处理

当已知用户时区时，系统提示会包含一个专门的 **当前日期和时间** 部分。为了保持提示缓存稳定，它现在只包含**时区**（不包含动态时钟或时间格式）。

当智能体需要当前时间时，请使用 `session_status`；状态卡包含时间戳行。该工具还可以选择设置每个会话的模型覆盖
（`model=default` 会清除它）。

可通过以下项配置：

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat`（`auto` | `12` | `24`）

完整行为细节请参见 [Date & Time](/zh-CN/date-time)。

## Skills

当存在符合条件的技能时，OpenClaw 会注入一个简洁的 **可用 Skills 列表**
（`formatSkillsForPrompt`），其中包含每个技能的**文件路径**。该
提示会指示模型使用 `read` 加载列出位置中的 SKILL.md（工作区、托管或内置）。如果没有符合条件的技能，则省略 Skills 部分。

资格条件包括技能元数据门控、运行时环境/配置检查，
以及当配置了 `agents.defaults.skills` 或
`agents.list[].skills` 时的有效智能体技能允许列表。

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

这样可以让基础提示保持简洁，同时仍然支持有针对性的技能使用。

## 文档

可用时，系统提示会包含一个 **文档** 部分，指向
本地 OpenClaw 文档目录（仓库工作区中的 `docs/` 或内置 npm
包文档），并且还会注明公共镜像、源码仓库、社区 Discord，以及用于
Skills 发现的 ClawHub（[https://clawhub.ai](https://clawhub.ai)）。该提示会指示模型在处理
OpenClaw 行为、命令、配置或架构时优先查阅本地文档，并在可行时自行运行
`openclaw status`（只有在无法访问时才询问用户）。
