---
read_when:
    - 调整思考、快速模式或详细输出指令的解析方式或默认值
summary: '`/think`、`/fast`、`/verbose`、`/trace` 的指令语法，以及推理可见性'
title: 思考级别
x-i18n:
    generated_at: "2026-04-12T18:22:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4f3b1341281f07ba4e9061e3355845dca234be04cc0d358594312beeb7676e68
    source_path: tools/thinking.md
    workflow: 15
---

# 思考级别（`/think` 指令）

## 它的作用

- 可在任何入站消息正文中使用内联指令：`/t <level>`、`/think:<level>` 或 `/thinking <level>`。
- 级别（别名）：`off | minimal | low | medium | high | xhigh | adaptive`
  - minimal → “think”
  - low → “think hard”
  - medium → “think harder”
  - high → “ultrathink”（最大预算）
  - xhigh → “ultrathink+”（仅适用于 GPT-5.2 和 Codex 模型）
  - adaptive → 由提供商管理的自适应推理预算（支持 Anthropic Claude 4.6 模型系列）
  - `x-high`、`x_high`、`extra-high`、`extra high` 和 `extra_high` 都映射为 `xhigh`。
  - `highest`、`max` 映射为 `high`。
- 提供商说明：
  - 当未显式设置思考级别时，Anthropic Claude 4.6 模型默认使用 `adaptive`。
  - 在 Anthropic 兼容流式路径上，MiniMax（`minimax/*`）默认使用 `thinking: { type: "disabled" }`，除非你在模型参数或请求参数中显式设置 thinking。这样可避免 MiniMax 的非原生 Anthropic 流格式泄露 `reasoning_content` 增量。
  - Z.AI（`zai/*`）只支持二元 thinking（`on`/`off`）。任何非 `off` 级别都会被视为 `on`（映射为 `low`）。
  - Moonshot（`moonshot/*`）将 `/think off` 映射为 `thinking: { type: "disabled" }`，并将任何非 `off` 级别映射为 `thinking: { type: "enabled" }`。启用 thinking 时，Moonshot 只接受 `tool_choice` 为 `auto|none`；OpenClaw 会将不兼容的值规范化为 `auto`。

## 解析顺序

1. 消息上的内联指令（仅适用于该条消息）。
2. 会话覆盖（通过发送仅包含指令的消息来设置）。
3. 每个智能体的默认值（配置中的 `agents.list[].thinkingDefault`）。
4. 全局默认值（配置中的 `agents.defaults.thinkingDefault`）。
5. 回退：Anthropic Claude 4.6 模型为 `adaptive`，其他支持推理的模型为 `low`，否则为 `off`。

## 设置会话默认值

- 发送一条**仅包含该指令**的消息（允许空白），例如 `/think:medium` 或 `/t high`。
- 该设置会保留在当前会话中（默认按发送者划分）；可通过 `/think:off` 或会话空闲重置来清除。
- 会发送确认回复（`Thinking level set to high.` / `Thinking disabled.`）。如果级别无效（例如 `/thinking big`），该命令会被拒绝并给出提示，会话状态保持不变。
- 发送不带参数的 `/think`（或 `/think:`）可查看当前思考级别。

## 按智能体应用

- **嵌入式 Pi**：解析后的级别会传递给进程内的 Pi 智能体运行时。

## 快速模式（`/fast`）

- 级别：`on|off`。
- 仅包含指令的消息会切换会话快速模式覆盖，并回复 `Fast mode enabled.` / `Fast mode disabled.`。
- 发送不带模式的 `/fast`（或 `/fast status`）可查看当前生效的快速模式状态。
- OpenClaw 按以下顺序解析快速模式：
  1. 内联 / 仅指令 `/fast on|off`
  2. 会话覆盖
  3. 每个智能体的默认值（`agents.list[].fastModeDefault`）
  4. 每个模型的配置：`agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. 回退：`off`
- 对于 `openai/*`，快速模式会通过在受支持的 Responses 请求中发送 `service_tier=priority` 来映射为 OpenAI 优先级处理。
- 对于 `openai-codex/*`，快速模式会在 Codex Responses 上发送相同的 `service_tier=priority` 标志。OpenClaw 在这两种认证路径上共用一个 `/fast` 开关。
- 对于直接发送到 `anthropic/*` 的公共请求，包括发送到 `api.anthropic.com` 的 OAuth 认证流量，快速模式会映射为 Anthropic 服务层级：`/fast on` 设置 `service_tier=auto`，`/fast off` 设置 `service_tier=standard_only`。
- 对于 Anthropic 兼容路径上的 `minimax/*`，`/fast on`（或 `params.fastMode: true`）会将 `MiniMax-M2.7` 改写为 `MiniMax-M2.7-highspeed`。
- 当两者都设置时，显式的 Anthropic `serviceTier` / `service_tier` 模型参数会覆盖快速模式默认值。对于非 Anthropic 代理 base URL，OpenClaw 仍会跳过注入 Anthropic 服务层级。

## 详细输出指令（`/verbose` 或 `/v`）

- 级别：`on`（最少）| `full` | `off`（默认）。
- 仅包含指令的消息会切换会话详细输出，并回复 `Verbose logging enabled.` / `Verbose logging disabled.`；无效级别会返回提示，不会更改状态。
- `/verbose off` 会存储显式的会话覆盖；可在会话 UI 中选择 `inherit` 以清除它。
- 内联指令仅影响该条消息；否则应用会话 / 全局默认值。
- 发送不带参数的 `/verbose`（或 `/verbose:`）可查看当前详细输出级别。
- 启用详细输出时，会发出结构化工具结果的智能体（Pi、其他 JSON 智能体）会将每次工具调用作为单独的仅元数据消息发回；如果可用，会以 `<emoji> <tool-name>: <arg>` 为前缀（路径 / 命令）。这些工具摘要会在每个工具启动时立即发送（单独气泡），而不是作为流式增量发送。
- 工具失败摘要在普通模式下仍然可见，但原始错误详情后缀会被隐藏，除非详细输出为 `on` 或 `full`。
- 当详细输出为 `full` 时，工具输出也会在完成后转发（单独气泡，并截断到安全长度）。如果你在一次运行进行中切换 `/verbose on|full|off`，后续工具气泡会遵循新设置。

## 插件追踪指令（`/trace`）

- 级别：`on` | `off`（默认）。
- 仅包含指令的消息会切换会话插件追踪输出，并回复 `Plugin trace enabled.` / `Plugin trace disabled.`。
- 内联指令仅影响该条消息；否则应用会话 / 全局默认值。
- 发送不带参数的 `/trace`（或 `/trace:`）可查看当前追踪级别。
- `/trace` 比 `/verbose` 更窄：它只暴露插件拥有的追踪 / 调试行，例如 Active Memory 调试摘要。
- 追踪行可能出现在 `/status` 中，也可能作为普通助手回复后的附加诊断消息出现。

## 推理可见性（`/reasoning`）

- 级别：`on|off|stream`。
- 仅包含指令的消息会切换是否在回复中显示 thinking 块。
- 启用后，推理会作为**单独消息**发送，并以 `Reasoning:` 为前缀。
- `stream`（仅 Telegram）：在回复生成期间，将推理流式传输到 Telegram 草稿气泡中，然后发送不带推理内容的最终答案。
- 别名：`/reason`。
- 发送不带参数的 `/reasoning`（或 `/reasoning:`）可查看当前推理级别。
- 解析顺序：内联指令，然后是会话覆盖，然后是每个智能体的默认值（`agents.list[].reasoningDefault`），最后是回退值（`off`）。

## 相关内容

- 提权模式文档位于 [Elevated mode](/zh-CN/tools/elevated)。

## 心跳

- 心跳探测正文是已配置的心跳提示（默认：`Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`）。心跳消息中的内联指令会像平常一样生效（但应避免通过心跳更改会话默认值）。
- 心跳传递默认仅发送最终负载。若还要发送单独的 `Reasoning:` 消息（如果可用），请设置 `agents.defaults.heartbeat.includeReasoning: true` 或按智能体设置 `agents.list[].heartbeat.includeReasoning: true`。

## Web 聊天 UI

- 页面加载时，Web 聊天思考选择器会镜像来自入站会话存储 / 配置的会话已存储级别。
- 选择其他级别会立即通过 `sessions.patch` 写入会话覆盖；它不会等待下一次发送，也不是一次性的 `thinkingOnce` 覆盖。
- 第一个选项始终是 `Default (<resolved level>)`，其中解析出的默认值来自当前会话模型：Anthropic/Bedrock 上的 Claude 4.6 为 `adaptive`，其他支持推理的模型为 `low`，否则为 `off`。
- 选择器会保持提供商感知：
  - 大多数提供商显示 `off | minimal | low | medium | high | adaptive`
  - Z.AI 显示二元选项 `off | on`
- `/think:<level>` 仍然有效，并会更新同一个已存储的会话级别，因此聊天指令与选择器保持同步。
