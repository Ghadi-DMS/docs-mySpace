---
read_when:
    - 编辑系统提示词文本、工具列表或时间 / 心跳部分时
    - 更改工作区引导或 Skills 注入行为时
summary: OpenClaw 系统提示词包含哪些内容以及如何组装
title: 系统提示词
x-i18n:
    generated_at: "2026-04-05T17:47:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: f14ba7f16dda81ac973d72be05931fa246bdfa0e1068df1a84d040ebd551c236
    source_path: concepts/system-prompt.md
    workflow: 15
---

# 系统提示词

OpenClaw 会为每次智能体运行构建自定义系统提示词。该提示词由 **OpenClaw 自有**，不使用 pi-coding-agent 的默认提示词。

提示词由 OpenClaw 组装，并注入到每次智能体运行中。

提供商插件可以贡献具备缓存感知能力的提示词引导，而无需替换完整的 OpenClaw 自有提示词。提供商运行时可以：

- 替换一小组具名核心部分（`interaction_style`、`tool_call_style`、`execution_bias`）
- 在提示词缓存边界之上注入**稳定前缀**
- 在提示词缓存边界之下注入**动态后缀**

将提供商自有贡献用于针对特定模型家族的调优。保留旧版
`before_prompt_build` 提示词变更机制，用于兼容性或真正的全局提示词更改，而不是常规的提供商行为。

## 结构

该提示词有意保持紧凑，并使用固定部分：

- **工具**：结构化工具事实来源提醒，加上运行时工具使用引导。
- **安全**：简短的护栏提醒，用于避免追求权力的行为或绕过监督。
- **Skills**（可用时）：告诉模型如何按需加载 skill 说明。
- **OpenClaw 自更新**：如何使用
  `config.schema.lookup` 安全检查配置，使用 `config.patch` 修补配置，使用 `config.apply` 替换完整配置，以及仅在用户明确请求时运行 `update.run`。仅所有者可用的 `gateway` 工具也会拒绝重写
  `tools.exec.ask` / `tools.exec.security`，包括会规范化为这些受保护 exec 路径的旧版 `tools.bash.*`
  别名。
- **工作区**：工作目录（`agents.defaults.workspace`）。
- **文档**：OpenClaw 文档的本地路径（仓库或 npm 包）以及何时读取它们。
- **工作区文件（已注入）**：表示引导文件已包含在下方。
- **沙箱**（启用时）：表示沙箱隔离运行时、沙箱路径以及是否可使用提升权限的 exec。
- **当前日期和时间**：用户本地时间、时区和时间格式。
- **回复标签**：受支持提供商的可选回复标签语法。
- **心跳**：心跳提示词和确认行为。
- **运行时**：主机、操作系统、node、仓库根目录（检测到时）、思考级别（一行）。
- **推理**：当前可见性级别 + /reasoning 切换提示。

“工具”部分还包含针对长时间运行工作的运行时引导：

- 对未来的后续操作使用 cron（`check back later`、提醒、周期性工作）
  ，而不是使用 `exec` 睡眠循环、`yieldMs` 延迟技巧或重复的 `process`
  轮询
- 仅将 `exec` / `process` 用于立即启动并持续在后台运行的命令
- 当启用了自动完成唤醒时，只启动命令一次，并在其输出内容或失败时依赖基于推送的唤醒路径
- 当你需要检查正在运行的命令时，使用 `process` 查看日志、状态、输入或进行干预
- 如果任务更大，优先使用 `sessions_spawn`；子智能体完成是基于推送的，并会自动通知回请求方
- 不要为了等待完成而循环轮询 `subagents list` / `sessions_list`

当启用实验性的 `update_plan` 工具时，“工具”部分还会告诉模型：
仅将其用于非平凡的多步骤工作，始终保持恰好一个
`in_progress` 步骤，并避免在每次更新后重复整个计划。

系统提示词中的安全护栏属于建议性内容。它们会引导模型行为，但不强制执行策略。硬性约束应使用工具策略、exec 审批、沙箱隔离和渠道允许列表；操作员可按设计禁用这些约束。

在具有原生审批卡片 / 按钮的渠道上，运行时提示词现在会告诉
智能体优先依赖该原生审批 UI。只有当工具结果表明聊天审批不可用，或手动审批是唯一途径时，它才应包含手动
`/approve` 命令。

## 提示词模式

OpenClaw 可以为子智能体渲染更小的系统提示词。运行时会为每次运行设置一个
`promptMode`（不是面向用户的配置）：

- `full`（默认）：包含上述所有部分。
- `minimal`：用于子智能体；省略 **Skills**、**Memory Recall**、**OpenClaw
  Self-Update**、**Model Aliases**、**User Identity**、**Reply Tags**、
  **Messaging**、**Silent Replies** 和 **Heartbeats**。工具、**安全**、
  工作区、沙箱、当前日期和时间（已知时）、运行时和注入的上下文仍然可用。
- `none`：仅返回基础身份行。

当 `promptMode=minimal` 时，额外注入的提示词会标记为 **子智能体
上下文**，而不是 **群聊上下文**。

## 工作区引导注入

引导文件会被裁剪并附加在 **项目上下文** 下，以便模型在无需显式读取的情况下看到身份和配置文件上下文：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（仅在全新工作区中）
- 存在时使用 `MEMORY.md`，否则回退为小写的 `memory.md`

所有这些文件都会在每一轮**注入到上下文窗口中**，这意味着它们会消耗 token。请保持其简洁——尤其是 `MEMORY.md`，它会随着时间增长，并可能导致上下文用量意外升高以及更频繁的压缩。

> **注意：** `memory/*.md` 每日文件**不会**被自动注入。它们会按需通过 `memory_search` 和 `memory_get` 工具访问，因此除非模型显式读取它们，否则不会计入上下文窗口。

大文件会被截断，并附带标记。每个文件的最大大小由
`agents.defaults.bootstrapMaxChars` 控制（默认值：20000）。跨文件注入的引导内容总量上限由 `agents.defaults.bootstrapTotalMaxChars`
控制（默认值：150000）。缺失的文件会注入简短的缺失文件标记。发生截断时，OpenClaw 可以在项目上下文中注入警告块；可通过
`agents.defaults.bootstrapPromptTruncationWarning` 控制（`off`、`once`、`always`；
默认值：`once`）。

子智能体会话仅注入 `AGENTS.md` 和 `TOOLS.md`（其他引导文件会被过滤掉，以保持子智能体上下文较小）。

内部 hooks 可以通过 `agent:bootstrap` 拦截此步骤，以变更或替换
注入的引导文件（例如将 `SOUL.md` 替换为其他 persona）。

如果你希望让智能体听起来不那么通用，请从
[SOUL.md Personality Guide](/zh-CN/concepts/soul) 开始。

要检查每个注入文件贡献了多少内容（原始与注入、截断以及工具 schema 开销），请使用 `/context list` 或 `/context detail`。参见 [Context](/zh-CN/concepts/context)。

## 时间处理

当用户时区已知时，系统提示词会包含专门的**当前日期和时间**部分。为了保持提示词缓存稳定，它现在只包含**时区**（不包含动态时钟或时间格式）。

当智能体需要当前时间时，请使用 `session_status`；状态卡片包含时间戳行。相同工具还可以选择设置每个会话的模型覆盖（`model=default` 会清除它）。

配置方式：

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat`（`auto` | `12` | `24`）

完整行为细节请参见 [Date & Time](/zh-CN/date-time)。

## Skills

当存在符合条件的 Skills 时，OpenClaw 会注入一个紧凑的**可用 Skills 列表**
（`formatSkillsForPrompt`），其中包含每个 skill 的**文件路径**。提示词会指示模型使用 `read` 加载列出位置（工作区、托管或内置）的 SKILL.md。若无符合条件的 Skills，则省略 Skills 部分。

符合条件与否会考虑 skill 元数据门控、运行时环境 / 配置检查，以及在配置了 `agents.defaults.skills` 或
`agents.list[].skills` 时生效的智能体 skill 允许列表。

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

这样可以保持基础提示词较小，同时仍支持有针对性的 skill 使用。

## 文档

可用时，系统提示词会包含一个**文档**部分，指向本地 OpenClaw 文档目录（仓库工作区中的 `docs/` 或内置 npm
包文档），并说明公共镜像、源仓库、社区 Discord 以及用于发现 Skills 的 ClawHub（[https://clawhub.ai](https://clawhub.ai)）。提示词会指示模型在处理 OpenClaw 行为、命令、配置或架构时优先查阅本地文档，并在可能时自行运行
`openclaw status`（仅在缺少访问权限时才询问用户）。
