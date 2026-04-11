---
read_when:
    - 你想在 OpenClaw 中使用 fal 图像生成
    - 你需要 `FAL_KEY` 认证流程
    - 你想为 `image_generate` 或 `video_generate` 使用 fal 默认值
summary: 在 OpenClaw 中设置 fal 图像和视频生成
title: fal
x-i18n:
    generated_at: "2026-04-11T01:39:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6865c3110cbb9d63f4ecec4467f89a801478759cd7a2f3efa83e8f7ce1dc3e67
    source_path: providers/fal.md
    workflow: 15
---

# fal

OpenClaw 内置了 `fal` 提供商，用于托管式图像和视频生成。

- 提供商：`fal`
- 认证：`FAL_KEY`（规范名称；`FAL_API_KEY` 也可作为后备方案）
- API：fal 模型端点

## 快速开始

1. 设置 API 密钥：

```bash
openclaw onboard --auth-choice fal-api-key
```

2. 设置默认图像模型：

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## 图像生成

内置的 `fal` 图像生成提供商默认使用
`fal/fal-ai/flux/dev`。

- 生成：每次请求最多 4 张图像
- 编辑模式：已启用，支持 1 张参考图像
- 支持 `size`、`aspectRatio` 和 `resolution`
- 当前编辑限制：fal 图像编辑端点**不**支持
  `aspectRatio` 覆盖

要将 fal 用作默认图像提供商：

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## 视频生成

内置的 `fal` 视频生成提供商默认使用
`fal/fal-ai/minimax/video-01-live`。

- 模式：文生视频和单图参考流程
- 运行时：基于队列的提交 / 状态 / 结果流程，用于长时间运行的任务
- Seedance 2.0 模型引用：
  - `fal/bytedance/seedance-2.0/fast/text-to-video`
  - `fal/bytedance/seedance-2.0/fast/image-to-video`
  - `fal/bytedance/seedance-2.0/text-to-video`
  - `fal/bytedance/seedance-2.0/image-to-video`

要将 Seedance 2.0 用作默认视频模型：

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
      },
    },
  },
}
```

## 相关内容

- [图像生成](/zh-CN/tools/image-generation)
- [视频生成](/zh-CN/tools/video-generation)
- [配置参考](/zh-CN/gateway/configuration-reference#agent-defaults)
