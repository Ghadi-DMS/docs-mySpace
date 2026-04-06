---
read_when:
    - 你想在 OpenClaw 中使用 Vydra 媒体生成功能
    - 你需要 Vydra API 密钥设置指南
summary: 在 OpenClaw 中使用 Vydra 图像、视频和语音
title: Vydra
x-i18n:
    generated_at: "2026-04-06T18:49:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 24006a687ed6f9792e7b2b10927cc7ad71c735462a92ce03d5fa7c2b2ee2fcc2
    source_path: providers/vydra.md
    workflow: 15
---

# Vydra

内置的 Vydra 插件新增了以下功能：

- 通过 `vydra/grok-imagine` 进行图像生成
- 通过 `vydra/veo3` 和 `vydra/kling` 进行视频生成
- 通过 Vydra 基于 ElevenLabs 的 TTS 路由进行语音合成

OpenClaw 对这三种能力都使用同一个 `VYDRA_API_KEY`。

## 重要基础 URL

使用 `https://www.vydra.ai/api/v1`。

Vydra 的顶级主机（`https://vydra.ai/api/v1`）当前会重定向到 `www`。某些 HTTP 客户端会在这种跨主机重定向时丢弃 `Authorization`，从而让有效的 API 密钥表现成具有误导性的认证失败。内置插件会直接使用 `www` 基础 URL 以避免这个问题。

## 设置

交互式新手引导：

```bash
openclaw onboard --auth-choice vydra-api-key
```

或者直接设置环境变量：

```bash
export VYDRA_API_KEY="vydra_live_..."
```

## 图像生成

默认图像模型：

- `vydra/grok-imagine`

将其设为默认图像提供商：

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "vydra/grok-imagine",
      },
    },
  },
}
```

当前内置支持仅限文生图。Vydra 托管的编辑路由需要远程图像 URL，而 OpenClaw 在内置插件中尚未添加 Vydra 专用的上传桥接。

共享工具行为请参见 [图像生成](/zh-CN/tools/image-generation)。

## 视频生成

已注册的视频模型：

- `vydra/veo3` 用于文生视频
- `vydra/kling` 用于图生视频

将 Vydra 设为默认视频提供商：

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "vydra/veo3",
      },
    },
  },
}
```

注意：

- `vydra/veo3` 在内置中仅作为文生视频提供。
- `vydra/kling` 当前需要远程图像 URL 引用。本地文件上传会被直接拒绝。
- Vydra 当前的 `kling` HTTP 路由在是否要求 `image_url` 或 `video_url` 方面表现并不一致；内置提供商会将同一个远程图像 URL 同时映射到这两个字段。
- 内置插件保持保守处理，不会转发诸如宽高比、分辨率、水印或生成音频等未文档化的样式控制参数。

提供商专属的实时覆盖：

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_LIVE_VYDRA_VIDEO=1 \
pnpm test:live -- extensions/vydra/vydra.live.test.ts
```

内置的 Vydra 实时测试文件现在覆盖：

- `vydra/veo3` 文生视频
- 使用远程图像 URL 的 `vydra/kling` 图生视频

需要时可覆盖远程图像测试夹具：

```bash
export OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL="https://example.com/reference.png"
```

共享工具行为请参见 [视频生成](/zh-CN/tools/video-generation)。

## 语音合成

将 Vydra 设为语音提供商：

```json5
{
  messages: {
    tts: {
      provider: "vydra",
      providers: {
        vydra: {
          apiKey: "${VYDRA_API_KEY}",
          voiceId: "21m00Tcm4TlvDq8ikWAM",
        },
      },
    },
  },
}
```

默认值：

- 模型：`elevenlabs/tts`
- 语音 ID：`21m00Tcm4TlvDq8ikWAM`

内置插件当前会暴露一个已知稳定可用的默认语音，并返回 MP3 音频文件。

## 相关内容

- [提供商目录](/zh-CN/providers/index)
- [图像生成](/zh-CN/tools/image-generation)
- [视频生成](/zh-CN/tools/video-generation)
