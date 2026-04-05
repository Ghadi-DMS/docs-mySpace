---
read_when:
    - 你希望在 API 提供商失败时有一个可靠的回退方案
    - 你正在运行 Codex CLI 或其他本地 AI CLI，并希望复用它们
    - 你想了解用于 CLI 后端工具访问的 MCP loopback 桥接
summary: CLI 后端：带可选 MCP 工具桥接的本地 AI CLI 回退方案
title: CLI 后端
x-i18n:
    generated_at: "2026-04-05T17:47:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: fe30bb4d5f51adcda53bf69a4f88e27832e78630ed5bfd8a33a66b6412b66f2d
    source_path: gateway/cli-backends.md
    workflow: 15
---

# CLI 后端（回退运行时）

当 API 提供商宕机、受到速率限制或暂时行为异常时，OpenClaw 可以运行**本地 AI CLI** 作为**纯文本回退**。这是一个有意保守的设计：

- **OpenClaw 工具不会被直接注入**，但设置了 `bundleMcp: true` 的后端
  可以通过 loopback MCP 桥接接收 Gateway 网关工具。
- 对支持该能力的 CLI 提供 **JSONL 分块流式传输**。
- **支持会话**（因此后续轮次能保持连贯）。
- 如果 CLI 接受图像路径，**图像可以透传**。

这被设计为一个**安全网**，而不是主要路径。当你希望在不依赖外部 API 的情况下获得“始终可用”的文本回复时，可以使用它。

如果你想要完整的封装运行时，包括 ACP 会话控制、后台任务、
线程 / 对话绑定以及持久化的外部编码会话，请改用
[ACP 智能体](/zh-CN/tools/acp-agents)。CLI 后端不是 ACP。

## 适合初学者的快速开始

你可以**无需任何配置**使用 Codex CLI（内置 OpenAI 插件
会注册一个默认后端）：

```bash
openclaw agent --message "hi" --model codex-cli/gpt-5.4
```

如果你的 Gateway 网关运行在 launchd / systemd 下，并且 PATH 很精简，只需添加
命令路径：

```json5
{
  agents: {
    defaults: {
      cliBackends: {
        "codex-cli": {
          command: "/opt/homebrew/bin/codex",
        },
      },
    },
  },
}
```

就是这样。除了 CLI 本身之外，不需要密钥，也不需要额外的认证配置。

如果你在 Gateway 网关主机上将某个内置 CLI 后端作为**主要消息提供商**使用，OpenClaw 现在会在你的配置
明确在模型引用或
`agents.defaults.cliBackends` 下引用该后端时，自动加载其所属的内置插件。

## 作为回退使用

将 CLI 后端添加到你的回退列表中，这样它只会在主模型失败时运行：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["codex-cli/gpt-5.4"],
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "codex-cli/gpt-5.4": {},
      },
    },
  },
}
```

说明：

- 如果你使用 `agents.defaults.models`（允许列表），你也必须将 CLI 后端模型包含在其中。
- 如果主提供商失败（认证、速率限制、超时），OpenClaw 会
  接着尝试 CLI 后端。

## 配置概览

所有 CLI 后端都位于：

```
agents.defaults.cliBackends
```

每个条目以**提供商 ID** 为键（例如 `codex-cli`、`my-cli`）。
该提供商 ID 会成为你的模型引用左侧部分：

```
<provider>/<model>
```

### 配置示例

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
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            "claude-opus-4-6": "opus",
            "claude-sonnet-4-6": "sonnet",
          },
          sessionArg: "--session",
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptArg: "--system",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          serialize: true,
        },
      },
    },
  },
}
```

## 工作原理

1. 根据提供商前缀（`codex-cli/...`）**选择后端**。
2. 使用相同的 OpenClaw 提示词和工作区上下文**构建系统提示词**。
3. 如果支持，会使用会话 ID **执行 CLI**，以保持历史一致。
4. **解析输出**（JSON 或纯文本）并返回最终文本。
5. 按后端**持久化会话 ID**，以便后续轮次复用相同的 CLI 会话。

<Warning>
内置 Anthropic `claude-cli` 后端已被移除，因为 Anthropic 的
OpenClaw 计费边界发生了变化。OpenClaw 仍然支持通用 CLI
后端，但 Anthropic API 流量应直接使用 Anthropic 提供商，
而不是已移除的本地 Claude CLI 路径。
</Warning>

## 会话

- 如果 CLI 支持会话，请设置 `sessionArg`（例如 `--session-id`）或
  `sessionArgs`（占位符为 `{sessionId}`），用于需要将该 ID 插入
  多个标志的情况。
- 如果 CLI 使用带有不同标志的**恢复子命令**，请设置
  `resumeArgs`（恢复时替换 `args`），并可选设置 `resumeOutput`
  （用于非 JSON 恢复）。
- `sessionMode`：
  - `always`：始终发送会话 ID（若未存储则生成新的 UUID）。
  - `existing`：仅当之前已存储会话 ID 时才发送。
  - `none`：从不发送会话 ID。

串行化说明：

- `serialize: true` 会保持同一通道中的运行按顺序执行。
- 大多数 CLI 会在一个提供商通道上串行执行。
- 当后端认证状态发生变化时，包括重新登录、令牌轮换或认证配置文件凭证变化，OpenClaw 会丢弃已存储的 CLI 会话复用状态。

## 图像（透传）

如果你的 CLI 接受图像路径，请设置 `imageArg`：

```json5
imageArg: "--image",
imageMode: "repeat"
```

OpenClaw 会将 base64 图像写入临时文件。如果设置了 `imageArg`，这些
路径会作为 CLI 参数传递。如果缺少 `imageArg`，OpenClaw 会将
文件路径附加到提示词中（路径注入）；这对于那些能从普通路径自动加载本地文件的 CLI 来说已经足够。

## 输入 / 输出

- `output: "json"`（默认）会尝试解析 JSON 并提取文本 + 会话 ID。
- 对于 Gemini CLI JSON 输出，当 `usage` 缺失或为空时，OpenClaw 会从 `response` 读取回复文本，并从
  `stats` 读取用量。
- `output: "jsonl"` 会解析 JSONL 流（例如 Codex CLI `--json`），并提取最终智能体消息以及存在时的会话
  标识符。
- `output: "text"` 会将 stdout 视为最终响应。

输入模式：

- `input: "arg"`（默认）会将提示词作为最后一个 CLI 参数传递。
- `input: "stdin"` 会通过 stdin 发送提示词。
- 如果提示词很长且设置了 `maxPromptArgChars`，则会改用 stdin。

## 默认值（插件所有）

内置 OpenAI 插件也会为 `codex-cli` 注册一个默认值：

- `command: "codex"`
- `args: ["exec","--json","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `resumeArgs: ["exec","resume","{sessionId}","--color","never","--sandbox","workspace-write","--skip-git-repo-check"]`
- `output: "jsonl"`
- `resumeOutput: "text"`
- `modelArg: "--model"`
- `imageArg: "--image"`
- `sessionMode: "existing"`

内置 Google 插件也会为 `google-gemini-cli` 注册一个默认值：

- `command: "gemini"`
- `args: ["--prompt", "--output-format", "json"]`
- `resumeArgs: ["--resume", "{sessionId}", "--prompt", "--output-format", "json"]`
- `modelArg: "--model"`
- `sessionMode: "existing"`
- `sessionIdFields: ["session_id", "sessionId"]`

前提条件：本地 Gemini CLI 必须已安装，并可在
`PATH` 中通过 `gemini` 使用（`brew install gemini-cli` 或
`npm install -g @google/gemini-cli`）。

Gemini CLI JSON 说明：

- 回复文本从 JSON `response` 字段读取。
- 当 `usage` 缺失或为空时，用量会回退到 `stats`。
- `stats.cached` 会被标准化为 OpenClaw `cacheRead`。
- 如果 `stats.input` 缺失，OpenClaw 会通过
  `stats.input_tokens - stats.cached` 推导输入令牌数。

仅在需要时覆盖（常见情况：使用绝对 `command` 路径）。

## 插件所有的默认值

CLI 后端默认值现在是插件表面的一部分：

- 插件通过 `api.registerCliBackend(...)` 注册它们。
- 后端 `id` 会成为模型引用中的提供商前缀。
- `agents.defaults.cliBackends.<id>` 中的用户配置仍会覆盖插件默认值。
- 后端特定的配置清理仍通过可选的
  `normalizeConfig` hook 由插件自身负责。

## 内置 MCP 叠加层

CLI 后端**不会**直接接收 OpenClaw 工具调用，但后端可以通过设置
`bundleMcp: true` 选择接收自动生成的 MCP 配置叠加层。

当前内置行为：

- `codex-cli`：无内置 MCP 叠加层
- `google-gemini-cli`：无内置 MCP 叠加层

启用内置 MCP 时，OpenClaw 会：

- 启动一个 loopback HTTP MCP 服务器，将 Gateway 网关工具暴露给 CLI 进程
- 使用每会话令牌（`OPENCLAW_MCP_TOKEN`）对桥接进行认证
- 将工具访问范围限定在当前会话、账户和渠道上下文内
- 为当前工作区加载已启用的内置 MCP 服务器
- 将它们与任何现有的后端 `--mcp-config` 合并
- 重写 CLI 参数以传递 `--strict-mcp-config --mcp-config <generated-file>`

如果没有启用任何 MCP 服务器，只要后端选择了内置 MCP，OpenClaw 仍会注入严格配置，以便后台运行保持隔离。

## 限制

- **没有直接的 OpenClaw 工具调用。** OpenClaw 不会将工具调用直接注入
  CLI 后端协议。只有当后端选择
  `bundleMcp: true` 时，后端才能看到 Gateway 网关工具。
- **流式传输取决于后端。** 某些后端会流式输出 JSONL；另一些则会
  缓冲到退出时再输出。
- **结构化输出**取决于 CLI 的 JSON 格式。
- **Codex CLI 会话**通过文本输出恢复（不是 JSONL），其结构性
  不如初始的 `--json` 运行。OpenClaw 会话仍然可以
  正常工作。

## 故障排除

- **找不到 CLI**：将 `command` 设置为完整路径。
- **模型名称错误**：使用 `modelAliases` 将 `provider/model` 映射到 CLI 模型。
- **没有会话连续性**：确保已设置 `sessionArg` 且 `sessionMode` 不是
  `none`（Codex CLI 当前无法使用 JSON 输出进行恢复）。
- **图像被忽略**：设置 `imageArg`（并确认 CLI 支持文件路径）。
