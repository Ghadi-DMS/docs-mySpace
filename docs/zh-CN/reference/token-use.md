---
read_when:
    - 解释 Token 使用量、成本或上下文窗口
    - 调试上下文增长或压缩行为
summary: OpenClaw 如何构建提示词上下文并报告 Token 使用量 + 成本
title: Token 使用量和成本
x-i18n:
    generated_at: "2026-04-12T02:53:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: f8c856549cd28b8364a640e6fa9ec26aa736895c7a993e96cbe85838e7df2dfb
    source_path: reference/token-use.md
    workflow: 15
---

# Token 使用量和成本

OpenClaw 跟踪的是 **tokens**，而不是字符。Token 因模型而异，但大多数 OpenAI 风格模型对英文文本的平均值约为每个 token 对应 4 个字符。

## 系统提示词是如何构建的

OpenClaw 会在每次运行时组装自己的系统提示词。它包括：

- 工具列表 + 简短说明
- Skills 列表（仅元数据；说明会按需通过 `read` 加载）
- 自更新说明
- 工作区 + 启动文件（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`、`BOOTSTRAP.md`（当其为新文件时），以及在存在时使用 `MEMORY.md`，否则使用小写回退文件 `memory.md`）。大文件会被 `agents.defaults.bootstrapMaxChars`（默认值：20000）截断，而启动注入的总量受 `agents.defaults.bootstrapTotalMaxChars`（默认值：150000）限制。`memory/*.md` 每日文件不属于常规启动提示词的一部分；在普通轮次中，它们仍通过内存工具按需访问，但纯 `/new` 和 `/reset` 可以在第一轮前置一个一次性的启动上下文块，其中包含最近的每日记忆。该启动前导由 `agents.defaults.startupContext` 控制。
- 时间（UTC + 用户时区）
- 回复标签 + heartbeat 行为
- 运行时元数据（主机 / OS / 模型 / thinking）

完整拆解请参见 [System Prompt](/zh-CN/concepts/system-prompt)。

## 上下文窗口中包含哪些内容

模型接收到的所有内容都会计入上下文限制：

- 系统提示词（上面列出的所有部分）
- 对话历史（用户 + 助手消息）
- 工具调用和工具结果
- 附件 / 转录内容（图像、音频、文件）
- 压缩摘要和裁剪产物
- 提供商包装层或安全头（不可见，但仍会计数）

对于图像，OpenClaw 会在调用提供商之前，对转录 / 工具图像负载进行缩放。使用 `agents.defaults.imageMaxDimensionPx`（默认值：`1200`）来调节这一点：

- 更低的值通常会减少视觉 token 使用量和负载大小。
- 更高的值会保留更多视觉细节，适合 OCR / UI 密集型截图。

如果你想查看实用拆解信息（按注入文件、工具、Skills 和系统提示词大小划分），请使用 `/context list` 或 `/context detail`。参见 [Context](/zh-CN/concepts/context)。

## 如何查看当前 Token 使用量

在聊天中使用以下命令：

- `/status` → **富含表情符号的状态卡片**，显示会话模型、上下文使用量、上一次回复的输入 / 输出 tokens，以及**预估成本**（仅限 API key）。
- `/usage off|tokens|full` → 在每条回复后附加一个**按回复统计的使用量页脚**。
  - 按会话持久化（存储为 `responseUsage`）。
  - OAuth 鉴权**隐藏成本**（仅显示 tokens）。
- `/usage cost` → 从 OpenClaw 会话日志中显示本地成本汇总。

其他界面：

- **TUI / Web TUI：**支持 `/status` + `/usage`。
- **CLI：**`openclaw status --usage` 和 `openclaw channels list` 会显示标准化后的提供商配额窗口（`X% left`，而不是按回复的成本）。
  当前支持使用量窗口的提供商有：Anthropic、GitHub Copilot、Gemini CLI、OpenAI Codex、MiniMax、Xiaomi 和 z.ai。

使用量界面会在显示前标准化常见的提供商原生字段别名。对于 OpenAI 家族的 Responses 流量，这包括 `input_tokens` / `output_tokens` 和 `prompt_tokens` / `completion_tokens`，因此传输层特定字段名不会影响 `/status`、`/usage` 或会话摘要。
Gemini CLI JSON 使用量也会被标准化：回复文本来自 `response`，而 `stats.cached` 会映射为 `cacheRead`；当 CLI 省略显式的 `stats.input` 字段时，会使用 `stats.input_tokens - stats.cached`。
对于原生 OpenAI 家族 Responses 流量，WebSocket / SSE 使用量别名也会以相同方式标准化；当 `total_tokens` 缺失或为 `0` 时，总量会回退为标准化后的输入 + 输出。
当当前会话快照信息较少时，`/status` 和 `session_status` 也可以从最近的转录使用量日志中恢复 token / 缓存计数器以及当前运行时模型标签。现有的非零实时值仍优先于转录回退值；当存储总量缺失或更小时，较大的、面向提示词的转录总量可以胜出。
提供商配额窗口的使用量鉴权会在可用时来自提供商特定钩子；否则 OpenClaw 会回退为匹配认证配置、环境变量或配置中的 OAuth / API-key 凭证。

## 成本估算（显示时）

成本会根据你的模型定价配置进行估算：

```
models.providers.<provider>.models[].cost
```

这些值表示 `input`、`output`、`cacheRead` 和 `cacheWrite` 的**每 100 万 tokens 的美元价格**。如果缺少定价信息，OpenClaw 只会显示 tokens。OAuth tokens 永远不会显示美元成本。

## Cache TTL 和裁剪的影响

提供商提示词缓存仅在缓存 TTL 窗口内生效。OpenClaw 可以选择运行**cache-ttl pruning**：当缓存 TTL 过期后，它会裁剪会话，然后重置缓存窗口，这样后续请求就可以复用新鲜缓存的上下文，而不是重新缓存完整历史。这样在会话闲置超过 TTL 时，可以降低缓存写入成本。

你可以在 [Gateway configuration](/zh-CN/gateway/configuration) 中配置它，并在 [Session pruning](/zh-CN/concepts/session-pruning) 中查看行为细节。

Heartbeat 可以在空闲间隔之间保持缓存**温热**。如果你的模型缓存 TTL 是 `1h`，将 heartbeat 间隔设置为略短于该值（例如 `55m`）可以避免重新缓存完整提示词，从而减少缓存写入成本。

在多智能体设置中，你可以保留一份共享模型配置，并通过 `agents.list[].params.cacheRetention` 为每个智能体调整缓存行为。

如需查看逐项参数指南，请参见 [Prompt Caching](/zh-CN/reference/prompt-caching)。

对于 Anthropic API 定价，缓存读取比输入 tokens 便宜得多，而缓存写入会按更高倍数计费。最新费率和 TTL 倍率请参见 Anthropic 的提示词缓存定价：
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### 示例：使用 heartbeat 保持 1h 缓存温热

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### 示例：按智能体缓存策略混合流量

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # 大多数智能体的默认基线
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # 为深度会话保持长缓存温热
    - id: "alerts"
      params:
        cacheRetention: "none" # 对突发通知避免缓存写入
```

`agents.list[].params` 会合并到所选模型的 `params` 之上，因此你可以只覆盖 `cacheRetention`，而保持其他模型默认值不变。

### 示例：启用 Anthropic 1M 上下文 beta header

Anthropic 的 1M 上下文窗口目前仍受 beta 限制。启用受支持的 Opus 或 Sonnet 模型上的 `context1m` 后，OpenClaw 可以注入所需的 `anthropic-beta` 值。

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        params:
          context1m: true
```

这会映射到 Anthropic 的 `context-1m-2025-08-07` beta header。

这仅在该模型条目上设置 `context1m: true` 时生效。

要求：该凭证必须具备使用长上下文的资格。否则，Anthropic 会对此请求返回提供商侧的速率限制错误。

如果你使用 OAuth / 订阅 tokens（`sk-ant-oat-*`）来认证 Anthropic，OpenClaw 会跳过 `context-1m-*` beta header，因为 Anthropic 当前会以 HTTP 401 拒绝该组合。

## 减少 Token 压力的技巧

- 使用 `/compact` 来总结长会话。
- 在你的工作流中裁剪大型工具输出。
- 对截图较多的会话，降低 `agents.defaults.imageMaxDimensionPx`。
- 保持 Skill 描述简短（Skill 列表会被注入到提示词中）。
- 对冗长、探索性的工作优先使用较小的模型。

关于确切的 Skill 列表开销计算公式，请参见 [Skills](/zh-CN/tools/skills)。
