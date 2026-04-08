---
read_when:
    - 你想在 OpenClaw 中使用 Google Gemini 模型
    - 你需要 API 密钥或 OAuth 认证流程
summary: Google Gemini 设置（API 密钥 + OAuth、图像生成、媒体理解、Web 搜索）
title: Google（Gemini）
x-i18n:
    generated_at: "2026-04-08T06:28:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: fad2ff68987301bd86145fa6e10de8c7b38d5bd5dbcd13db9c883f7f5b9a4e01
    source_path: providers/google.md
    workflow: 15
---

# Google（Gemini）

Google 插件通过 Google AI Studio 提供对 Gemini 模型的访问，以及通过 Gemini Grounding 提供图像生成、媒体理解（图像/音频/视频）和 Web 搜索。

- 提供商：`google`
- 认证：`GEMINI_API_KEY` 或 `GOOGLE_API_KEY`
- API：Google Gemini API
- 替代提供商：`google-gemini-cli`（OAuth）

## 快速开始

1. 设置 API 密钥：

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. 设置默认模型：

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## 非交互式示例

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## OAuth（Gemini CLI）

另一个提供商 `google-gemini-cli` 使用 PKCE OAuth，而不是 API 密钥。这个集成并非官方集成；一些用户反馈会遇到账户限制。使用风险由你自行承担。

- 默认模型：`google-gemini-cli/gemini-3-flash-preview`
- 别名：`gemini-cli`
- 安装前提：本地可用的 Gemini CLI，命令名为 `gemini`
  - Homebrew：`brew install gemini-cli`
  - npm：`npm install -g @google/gemini-cli`
- 登录：

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

环境变量：

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

（或使用 `GEMINI_CLI_*` 变体。）

如果在登录后 Gemini CLI OAuth 请求失败，请在 Gateway 网关主机上设置 `GOOGLE_CLOUD_PROJECT` 或 `GOOGLE_CLOUD_PROJECT_ID`，然后重试。

如果在浏览器流程开始前登录失败，请确认本地 `gemini` 命令已安装并且在 `PATH` 中。OpenClaw 同时支持 Homebrew 安装和全局 npm 安装，包括常见的 Windows/npm 布局。

Gemini CLI JSON 使用说明：

- 回复文本来自 CLI JSON 的 `response` 字段。
- 当 CLI 将 `usage` 留空时，用量会回退到 `stats`。
- `stats.cached` 会被规范化为 OpenClaw `cacheRead`。
- 如果 `stats.input` 缺失，OpenClaw 会根据 `stats.input_tokens - stats.cached` 推导输入 token 数。

## 功能

| 功能 | 支持情况 |
| ---------------------- | ----------------- |
| 聊天补全 | 是 |
| 图像生成 | 是 |
| 音乐生成 | 是 |
| 图像理解 | 是 |
| 音频转写 | 是 |
| 视频理解 | 是 |
| Web 搜索（Grounding） | 是 |
| 思考/推理 | 是（Gemini 3.1+） |
| Gemma 4 模型 | 是 |

Gemma 4 模型（例如 `gemma-4-26b-a4b-it`）支持思考模式。OpenClaw 会将 `thinkingBudget` 重写为 Gemma 4 支持的 Google `thinkingLevel`。将 thinking 设置为 `off` 时，会保持禁用 thinking，而不是映射为 `MINIMAL`。

## 直接复用 Gemini 缓存

对于直接 Gemini API 运行（`api: "google-generative-ai"`），OpenClaw 现在会将已配置的 `cachedContent` 句柄传递给 Gemini 请求。

- 可以使用 `cachedContent` 或旧版 `cached_content` 配置按模型或全局参数
- 如果两者同时存在，优先使用 `cachedContent`
- 示例值：`cachedContents/prebuilt-context`
- Gemini 缓存命中用量会根据上游 `cachedContentTokenCount` 规范化为 OpenClaw `cacheRead`

示例：

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## 图像生成

内置的 `google` 图像生成提供商默认使用 `google/gemini-3.1-flash-image-preview`。

- 也支持 `google/gemini-3-pro-image-preview`
- 生成：每次请求最多 4 张图像
- 编辑模式：已启用，最多支持 5 张输入图像
- 几何控制：`size`、`aspectRatio` 和 `resolution`

仅支持 OAuth 的 `google-gemini-cli` 提供商是一个独立的文本推理接口。图像生成、媒体理解和 Gemini Grounding 仍然使用 `google` 提供商 id。

要将 Google 设为默认图像提供商：

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

有关共享工具参数、提供商选择和故障切换行为，请参阅 [图像生成](/zh-CN/tools/image-generation)。

## 视频生成

内置的 `google` 插件还通过共享的 `video_generate` 工具注册了视频生成功能。

- 默认视频模型：`google/veo-3.1-fast-generate-preview`
- 模式：文生视频、图生视频，以及单视频参考流程
- 支持 `aspectRatio`、`resolution` 和 `audio`
- 当前时长限制：**4 到 8 秒**

要将 Google 设为默认视频提供商：

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

有关共享工具参数、提供商选择和故障切换行为，请参阅 [视频生成](/zh-CN/tools/video-generation)。

## 音乐生成

内置的 `google` 插件还通过共享的 `music_generate` 工具注册了音乐生成功能。

- 默认音乐模型：`google/lyria-3-clip-preview`
- 也支持 `google/lyria-3-pro-preview`
- 提示词控制：`lyrics` 和 `instrumental`
- 输出格式：默认 `mp3`，并且 `google/lyria-3-pro-preview` 还支持 `wav`
- 参考输入：最多 10 张图像
- 基于会话的运行会通过共享的任务/状态流程分离处理，包括 `action: "status"`

要将 Google 设为默认音乐提供商：

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

有关共享工具参数、提供商选择和故障切换行为，请参阅 [音乐生成](/zh-CN/tools/music-generation)。

## 环境说明

如果 Gateway 网关以守护进程方式运行（launchd/systemd），请确保该进程可以访问 `GEMINI_API_KEY`（例如在 `~/.openclaw/.env` 中，或通过 `env.shellEnv`）。
