---
read_when:
    - 你想从你自己的 GPU 主机提供模型服务
    - 你正在配置 LM Studio 或兼容 OpenAI 的代理
    - 你需要最安全的本地模型指导
summary: 在本地 LLM（LM Studio、vLLM、LiteLLM、自定义 OpenAI 端点）上运行 OpenClaw
title: 本地模型
x-i18n:
    generated_at: "2026-04-15T10:43:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: d9b4939bb67f4fea23de45e88cdf29b887b2992389bc581058361d27bd227dd3
    source_path: gateway/local-models.md
    workflow: 15
---

# 本地模型

本地部署是可行的，但 OpenClaw 需要大上下文窗口，并且要对提示注入有很强的防护。小显存卡会截断上下文，并削弱安全性。尽量把配置拉高：**至少 2 台满配 Mac Studio，或同等级 GPU 设备（约 3 万美元以上）**。单张 **24 GB** GPU 只适合更轻量的提示，并且延迟会更高。使用你能运行的**最大 / 完整尺寸模型变体**；激进量化或“small”检查点会提高提示注入风险（参见[安全](/zh-CN/gateway/security)）。

如果你想要摩擦最低的本地部署方案，从 [LM Studio](/zh-CN/providers/lmstudio) 或 [Ollama](/zh-CN/providers/ollama) 加 `openclaw onboard` 开始。这一页是面向更高端本地堆栈和自定义兼容 OpenAI 的本地服务器的偏好型指南。

## 推荐：LM Studio + 大型本地模型（Responses API）

这是目前最好的本地方案。先在 LM Studio 中加载一个大型模型（例如完整尺寸的 Qwen、DeepSeek 或 Llama 构建），启用本地服务器（默认是 `http://127.0.0.1:1234`），然后使用 Responses API，将推理过程与最终文本分离。

```json5
{
  agents: {
    defaults: {
      model: { primary: “lmstudio/my-local-model” },
      models: {
        “anthropic/claude-opus-4-6”: { alias: “Opus” },
        “lmstudio/my-local-model”: { alias: “Local” },
      },
    },
  },
  models: {
    mode: “merge”,
    providers: {
      lmstudio: {
        baseUrl: “http://127.0.0.1:1234/v1”,
        apiKey: “lmstudio”,
        api: “openai-responses”,
        models: [
          {
            id: “my-local-model”,
            name: “Local Model”,
            reasoning: false,
            input: [“text”],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

**设置检查清单**

- 安装 LM Studio：[https://lmstudio.ai](https://lmstudio.ai)
- 在 LM Studio 中，下载**可用的最大模型构建**（避免使用 “small” / 重度量化变体），启动服务器，并确认 `http://127.0.0.1:1234/v1/models` 能列出该模型。
- 用 LM Studio 中显示的实际模型 ID 替换 `my-local-model`。
- 保持模型处于已加载状态；冷加载会增加启动延迟。
- 如果你的 LM Studio 构建不同，调整 `contextWindow` / `maxTokens`。
- 对于 WhatsApp，坚持使用 Responses API，这样只会发送最终文本。

即使在本地运行，也建议保留托管模型配置；使用 `models.mode: "merge"`，这样回退模型仍然可用。

### 混合配置：托管主模型，本地回退

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### 本地优先，托管兜底

交换主模型和回退模型的顺序；保留相同的 provider 配置块和 `models.mode: "merge"`，这样当本地主机不可用时，你仍然可以回退到 Sonnet 或 Opus。

### 区域托管 / 数据路由

- 托管版的 MiniMax / Kimi / GLM 变体也可通过 OpenRouter 获取，并支持区域固定端点（例如托管在美国）。你可以在那里选择区域变体，以便将流量限制在你指定的司法辖区内，同时继续使用 `models.mode: "merge"`，为 Anthropic / OpenAI 保留回退能力。
- 纯本地仍然是隐私性最强的方案；当你需要提供商功能、但又希望控制数据流向时，区域托管路由是折中方案。

## 其他兼容 OpenAI 的本地代理

只要 vLLM、LiteLLM、OAI-proxy 或自定义 Gateway 网关暴露的是兼容 OpenAI 风格的 `/v1` 端点，就可以使用。把上面的 provider 配置块替换成你的端点和模型 ID 即可：

```json5
{
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

保留 `models.mode: "merge"`，这样托管模型仍然可作为回退使用。

关于本地 / 代理 `/v1` 后端的行为说明：

- OpenClaw 会将这些后端视为代理风格的兼容 OpenAI 路由，而不是原生 OpenAI 端点
- 仅适用于原生 OpenAI 的请求整形不会在这里生效：没有 `service_tier`、没有 Responses 的 `store`、没有 OpenAI 推理兼容负载整形，也没有提示缓存提示
- 隐藏的 OpenClaw 归因请求头（`originator`、`version`、`User-Agent`）不会注入到这些自定义代理 URL 中

对于更严格的兼容 OpenAI 后端，兼容性说明如下：

- 有些服务器在 Chat Completions 中只接受字符串类型的 `messages[].content`，不接受结构化的 content-part 数组。对于这些端点，请设置 `models.providers.<provider>.models[].compat.requiresStringContent: true`。
- 一些更小或更严格的本地后端，在面对 OpenClaw 完整智能体运行时的提示结构时会不稳定，尤其是在包含工具 schema 时。如果后端能处理很小的直接 `/v1/chat/completions` 调用，但在正常的 OpenClaw 智能体轮次中失败，请先尝试设置 `agents.defaults.experimental.localModelLean: true`，以移除 `browser`、`cron` 和 `message` 这类重量级默认工具；这是实验性标志，不是稳定的默认模式设置。如果这样仍然失败，再尝试 `models.providers.<provider>.models[].compat.supportsTools: false`。
- 如果后端只会在更大的 OpenClaw 运行中失败，剩余问题通常是上游模型 / 服务器容量不足，或后端本身的 bug，而不是 OpenClaw 的传输层问题。

## 故障排除

- Gateway 网关能访问代理吗？`curl http://127.0.0.1:1234/v1/models`
- LM Studio 模型被卸载了？重新加载；冷启动是常见的“卡住”原因。
- 当检测到上下文窗口低于 **32k** 时，OpenClaw 会发出警告；低于 **16k** 时会直接阻止运行。如果你触发了这个预检查，请提高服务器 / 模型的上下文限制，或选择更大的模型。
- 上下文报错？降低 `contextWindow`，或提高服务器限制。
- 兼容 OpenAI 的服务器返回 `messages[].content ... expected a string`？
  在该模型条目上添加 `compat.requiresStringContent: true`。
- 很小的直接 `/v1/chat/completions` 调用可以工作，但 `openclaw infer model run` 在 Gemma 或其他本地模型上失败？先用 `compat.supportsTools: false` 禁用工具 schema，然后重新测试。如果服务器仍然只会在更大的 OpenClaw 提示下崩溃，就应将其视为上游服务器 / 模型的限制。
- 安全性：本地模型会跳过提供商侧过滤；保持智能体职责范围尽量收窄，并开启压缩，以限制提示注入的影响半径。
