---
read_when:
    - 你想在 OpenClaw 中使用 Qwen
    - 你之前使用过 Qwen OAuth
summary: 通过 OpenClaw 内置的 qwen 提供商使用 Qwen Cloud
title: Qwen
x-i18n:
    generated_at: "2026-04-05T22:30:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: f175793693ab6a4c3f1f4d42040e673c15faf7603a500757423e9e06977c989d
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**Qwen OAuth 已被移除。** 使用 `portal.qwen.ai` 端点的免费层 OAuth 集成
（`qwen-portal`）现已不可用。背景信息请参见
[Issue #49557](https://github.com/openclaw/openclaw/issues/49557)。

</Warning>

## 推荐：Qwen Cloud

OpenClaw 现在将 Qwen 视为一个一等内置提供商，其规范 id 为
`qwen`。该内置提供商面向 Qwen Cloud / Alibaba DashScope 和
Coding Plan 端点，并保留旧版 `modelstudio` id 作为兼容性别名。

- 提供商：`qwen`
- 首选环境变量：`QWEN_API_KEY`
- 为兼容性也接受：`MODELSTUDIO_API_KEY`、`DASHSCOPE_API_KEY`
- API 风格：兼容 OpenAI

如果你想使用 `qwen3.6-plus`，优先选择 **Standard（按量付费）** 端点。
Coding Plan 支持可能会落后于公开目录。

```bash
# 全局 Coding Plan 端点
openclaw onboard --auth-choice qwen-api-key

# 中国区 Coding Plan 端点
openclaw onboard --auth-choice qwen-api-key-cn

# 全局 Standard（按量付费）端点
openclaw onboard --auth-choice qwen-standard-api-key

# 中国区 Standard（按量付费）端点
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

旧版 `modelstudio-*` auth-choice id 和 `modelstudio/...` 模型引用仍然可作为兼容性别名使用，但新的设置流程应优先使用规范的
`qwen-*` auth-choice id 和 `qwen/...` 模型引用。

完成新手引导后，设置默认模型：

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## 计划类型和端点

| 计划 | 区域 | Auth choice | 端点 |
| -------------------------- | ------ | -------------------------- | ------------------------------------------------ |
| Standard（按量付费） | 中国区 | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1` |
| Standard（按量付费） | 全球 | `qwen-standard-api-key` | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan（订阅制） | 中国区 | `qwen-api-key-cn` | `coding.dashscope.aliyuncs.com/v1` |
| Coding Plan（订阅制） | 全球 | `qwen-api-key` | `coding-intl.dashscope.aliyuncs.com/v1` |

该提供商会根据你的 auth choice 自动选择端点。规范选项使用
`qwen-*` 系列；`modelstudio-*` 仅保留用于兼容。你也可以在配置中使用自定义
`baseUrl` 进行覆盖。

原生 Model Studio 端点会在共享的 `openai-completions` 传输协议上声明流式使用兼容性。OpenClaw 现在基于端点能力来判断这一点，因此，指向相同原生主机的兼容 DashScope 的自定义提供商 id 也会继承相同的流式使用行为，而不再要求必须使用内置的 `qwen` 提供商 id。

## 获取你的 API 密钥

- **管理密钥**：[home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **文档**：[docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## 内置目录

OpenClaw 当前内置了以下 Qwen 目录：

| 模型引用 | 输入 | 上下文 | 说明 |
| --------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus` | text, image | 1,000,000 | 默认模型 |
| `qwen/qwen3.6-plus` | text, image | 1,000,000 | 需要此模型时优先选择 Standard 端点 |
| `qwen/qwen3-max-2026-01-23` | text | 262,144 | Qwen Max 产品线 |
| `qwen/qwen3-coder-next` | text | 262,144 | 编码 |
| `qwen/qwen3-coder-plus` | text | 1,000,000 | 编码 |
| `qwen/MiniMax-M2.5` | text | 1,000,000 | 已启用推理 |
| `qwen/glm-5` | text | 202,752 | GLM |
| `qwen/glm-4.7` | text | 202,752 | GLM |
| `qwen/kimi-k2.5` | text, image | 262,144 | Alibaba 上的 Moonshot AI |

即使某个模型已存在于内置目录中，具体可用性仍可能因端点和计费计划而异。

原生流式使用兼容性同时适用于 Coding Plan 主机和 Standard 的
DashScope 兼容主机：

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Qwen 3.6 Plus 可用性

`qwen3.6-plus` 可用于 Standard（按量付费）Model Studio 端点：

- 中国区：`dashscope.aliyuncs.com/compatible-mode/v1`
- 全球：`dashscope-intl.aliyuncs.com/compatible-mode/v1`

如果 Coding Plan 端点对 `qwen3.6-plus` 返回“unsupported model”错误，请切换到 Standard（按量付费），而不是继续使用 Coding Plan 端点/密钥组合。

## 能力计划

`qwen` 扩展正被定位为完整 Qwen Cloud 能力面的供应商归属地，而不仅仅是编码/文本模型。

- 文本/聊天模型：现已内置
- 工具调用、结构化输出、思维链：继承自兼容 OpenAI 的传输协议
- 图像生成：计划在提供商插件层实现
- 图像/视频理解：现已在 Standard 端点内置
- 语音/音频：计划在提供商插件层实现
- 记忆嵌入/重排序：计划通过嵌入适配器能力面提供
- 视频生成：现已通过共享的视频生成能力内置

## 多模态附加功能

`qwen` 扩展现在还提供：

- 通过 `qwen-vl-max-latest` 提供视频理解
- 通过以下模型提供 Wan 视频生成：
  - `wan2.6-t2v`（默认）
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

这些多模态能力面使用的是 **Standard** DashScope 端点，而不是
Coding Plan 端点。

- 全球/国际版 Standard base URL：`https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- 中国区 Standard base URL：`https://dashscope.aliyuncs.com/compatible-mode/v1`

对于视频生成，OpenClaw 会先将已配置的 Qwen 区域映射到对应的
DashScope AIGC 主机，然后再提交任务：

- 全球/国际版：`https://dashscope-intl.aliyuncs.com`
- 中国区：`https://dashscope.aliyuncs.com`

这意味着，普通的 `models.providers.qwen.baseUrl` 即使指向
Coding Plan 或 Standard Qwen 主机之一，视频生成仍会使用正确区域的
DashScope 视频端点。

对于视频生成，请显式设置默认模型：

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

当前内置的 Qwen 视频生成限制：

- 每个请求最多 **1** 个输出视频
- 最多 **1** 张输入图片
- 最多 **4** 个输入视频
- 最长 **10 秒** 时长
- 支持 `size`、`aspectRatio`、`resolution`、`audio` 和 `watermark`
- 参考图片/视频模式当前要求使用 **远程 http(s) URL**。本地文件路径会在前置校验时被拒绝，因为 DashScope 视频端点不接受为这些引用上传本地缓冲区。

有关共享工具参数、提供商选择和故障切换行为，请参见 [视频生成](/zh-CN/tools/video-generation)。

## 环境说明

如果 Gateway 网关 以守护进程（launchd/systemd）方式运行，请确保
`QWEN_API_KEY` 对该进程可用（例如放在 `~/.openclaw/.env` 中，或通过
`env.shellEnv` 提供）。
