---
read_when:
    - 通过智能体生成视频时
    - 配置视频生成提供商和模型时
    - 了解 `video_generate` 工具参数时
summary: 使用已配置的提供商（例如 Qwen）生成视频
title: 视频生成
x-i18n:
    generated_at: "2026-04-05T17:50:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: c31872d82318072717e8789cfedbc5b199cdd0adf10272af2d4d77b62d062b5e
    source_path: tools/video-generation.md
    workflow: 15
---

# 视频生成

`video_generate` 工具允许智能体使用你已配置的提供商来创建视频。生成的视频会作为媒体附件自动随智能体的回复一起发送。

<Note>
只有在至少有一个视频生成提供商可用时，该工具才会显示。如果你在智能体工具中看不到 `video_generate`，请配置 `agents.defaults.videoGenerationModel` 或设置提供商 API 密钥。
</Note>

## 快速开始

1. 为至少一个提供商设置 API 密钥（例如 `QWEN_API_KEY`）。
2. 可选：设置你偏好的模型：

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: "qwen/wan2.6-t2v",
    },
  },
}
```

3. 向智能体提问：_“生成一段 5 秒钟的电影感视频，内容是一只友好的龙虾在日落时冲浪。”_

智能体会自动调用 `video_generate`。无需配置工具允许名单——只要有可用提供商，它默认就是启用的。

## 支持的提供商

| 提供商 | 默认模型 | 参考输入 | API 密钥 |
| -------- | ------------- | ---------------- | ---------------------------------------------------------- |
| Qwen | `wan2.6-t2v` | 是，远程 URL | `QWEN_API_KEY`, `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY` |

使用 `action: "list"` 可在运行时查看可用的提供商和模型：

```
/tool video_generate action=list
```

## 工具参数

| 参数 | 类型 | 描述 |
| ----------------- | -------- | ------------------------------------------------------------------------------------- |
| `prompt` | string | 视频生成提示词（`action: "generate"` 时必填） |
| `action` | string | `"generate"`（默认）或 `"list"`，用于查看提供商 |
| `model` | string | 提供商/模型覆盖，例如 `qwen/wan2.6-t2v` |
| `image` | string | 单个参考图像路径或 URL |
| `images` | string[] | 多个参考图像（最多 5 个） |
| `video` | string | 单个参考视频路径或 URL |
| `videos` | string[] | 多个参考视频（最多 4 个） |
| `size` | string | 当提供商支持时的尺寸提示 |
| `aspectRatio` | string | 宽高比：`1:1`、`2:3`、`3:2`、`3:4`、`4:3`、`4:5`、`5:4`、`9:16`、`16:9`、`21:9` |
| `resolution` | string | 分辨率提示：`480P`、`720P` 或 `1080P` |
| `durationSeconds` | number | 目标时长（秒） |
| `audio` | boolean | 当提供商支持时启用生成音频 |
| `watermark` | boolean | 当支持时切换提供商水印 |
| `filename` | string | 输出文件名提示 |

并非所有提供商都支持所有参数。该工具会在提交请求前验证提供商的能力限制。

## 配置

### 模型选择

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

### 提供商选择顺序

生成视频时，OpenClaw 会按以下顺序尝试提供商：

1. 工具调用中的 **`model` 参数**（如果智能体指定了）
2. 配置中的 **`videoGenerationModel.primary`**
3. 按顺序使用 **`videoGenerationModel.fallbacks`**
4. **自动检测** —— 仅使用有认证支持的提供商默认值：
   - 先使用当前默认提供商
   - 再按 provider-id 顺序使用其余已注册的视频生成提供商

如果某个提供商失败，会自动尝试下一个候选项。如果全部失败，错误信息会包含每次尝试的详细信息。

## Qwen 参考输入

内置的 Qwen 提供商支持文生视频以及图像/视频参考模式，但上游 DashScope 视频端点当前对参考输入**要求必须是远程 http(s) URL**。本地文件路径和上传的缓冲区会在前端直接被拒绝，而不是被静默忽略。

## 相关内容

- [工具概览](/zh-CN/tools) —— 所有可用的智能体工具
- [Qwen](/zh-CN/providers/qwen) —— Qwen 专属设置与限制
- [配置参考](/zh-CN/gateway/configuration-reference#agent-defaults) —— `videoGenerationModel` 配置
- [模型](/zh-CN/concepts/models) —— 模型配置与故障转移
