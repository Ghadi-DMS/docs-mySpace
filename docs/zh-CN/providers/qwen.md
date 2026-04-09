---
read_when:
    - 你想在 OpenClaw 中使用 Qwen
    - 你之前使用过 Qwen OAuth
summary: 通过 OpenClaw 内置的 qwen 提供商使用 Qwen Cloud
title: Qwen
x-i18n:
    generated_at: "2026-04-09T00:06:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4786df2cb6ec1ab29d191d012c61dcb0e5468bf0f8561fbbb50eed741efad325
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**Qwen OAuth 已被移除。** 使用 `portal.qwen.ai` 端点的免费层 OAuth 集成
（`qwen-portal`）已不再可用。
背景信息请参见 [Issue #49557](https://github.com/openclaw/openclaw/issues/49557)。

</Warning>

## 推荐：Qwen Cloud

OpenClaw 现在将 Qwen 视为一等内置提供商，其规范 id 为
`qwen`。该内置提供商面向 Qwen Cloud / Alibaba DashScope 和
Coding Plan 端点，并保留旧版 `modelstudio` id 作为兼容别名。

- 提供商：`qwen`
- 首选环境变量：`QWEN_API_KEY`
- 出于兼容性也接受：`MODELSTUDIO_API_KEY`、`DASHSCOPE_API_KEY`
- API 风格：兼容 OpenAI

如果你想使用 `qwen3.6-plus`，建议优先选择**标准版（按量付费）**端点。
Coding Plan 支持可能会落后于公开目录。

```bash
# 全局 Coding Plan 端点
openclaw onboard --auth-choice qwen-api-key

# 中国区 Coding Plan 端点
openclaw onboard --auth-choice qwen-api-key-cn

# 全局标准版（按量付费）端点
openclaw onboard --auth-choice qwen-standard-api-key

# 中国区标准版（按量付费）端点
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

旧版 `modelstudio-*` auth-choice id 和 `modelstudio/...` 模型引用仍然
可作为兼容别名继续使用，但新的设置流程应优先使用规范的
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

## 套餐类型和端点

| 套餐 | 区域 | Auth choice | 端点 |
| -------------------------- | ------ | -------------------------- | ------------------------------------------------ |
| 标准版（按量付费） | 中国区 | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1` |
| 标准版（按量付费） | 全球 | `qwen-standard-api-key` | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan（订阅制） | 中国区 | `qwen-api-key-cn` | `coding.dashscope.aliyuncs.com/v1` |
| Coding Plan（订阅制） | 全球 | `qwen-api-key` | `coding-intl.dashscope.aliyuncs.com/v1` |

该提供商会根据你的 auth choice 自动选择端点。规范选择使用
`qwen-*` 系列；`modelstudio-*` 仅保留为兼容用途。
你也可以在配置中使用自定义 `baseUrl` 进行覆盖。

原生 Model Studio 端点会在共享的 `openai-completions` 传输协议上声明
流式使用兼容性。OpenClaw 现在会根据端点能力来判断这一点，因此，指向相同原生主机的
兼容 DashScope 的自定义提供商 id 也会继承相同的流式使用行为，而不再要求必须使用
内置的 `qwen` 提供商 id。

## 获取你的 API 密钥

- **管理密钥**：[home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **文档**：[docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## 内置目录

OpenClaw 当前内置了以下 Qwen 目录。配置后的目录具备端点感知能力：
Coding Plan 配置会省略那些目前只确认可在标准版端点上正常工作的模型。

| 模型引用 | 输入 | 上下文 | 说明 |
| --------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus` | text, image | 1,000,000 | 默认模型 |
| `qwen/qwen3.6-plus` | text, image | 1,000,000 | 需要该模型时建议使用标准版端点 |
| `qwen/qwen3-max-2026-01-23` | text | 262,144 | Qwen Max 系列 |
| `qwen/qwen3-coder-next` | text | 262,144 | 编程 |
| `qwen/qwen3-coder-plus` | text | 1,000,000 | 编程 |
| `qwen/MiniMax-M2.5` | text | 1,000,000 | 已启用推理 |
| `qwen/glm-5` | text | 202,752 | GLM |
| `qwen/glm-4.7` | text | 202,752 | GLM |
| `qwen/kimi-k2.5` | text, image | 262,144 | Moonshot AI，经由 Alibaba |

即使某个模型出现在内置目录中，其可用性仍可能因端点和计费套餐而异。

原生流式传输使用兼容性同时适用于 Coding Plan 主机和
标准版兼容 DashScope 的主机：

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Qwen 3.6 Plus 可用性

`qwen3.6-plus` 可在标准版（按量付费）Model Studio
端点上使用：

- 中国区：`dashscope.aliyuncs.com/compatible-mode/v1`
- 全球：`dashscope-intl.aliyuncs.com/compatible-mode/v1`

如果 Coding Plan 端点对 `qwen3.6-plus` 返回“unsupported model”错误，
请切换到标准版（按量付费），而不是继续使用 Coding Plan
端点 / 密钥组合。

## 能力规划

`qwen` 扩展正被定位为完整 Qwen
Cloud 能力面的厂商归属地，而不仅仅是编程 / 文本模型。

- 文本 / 聊天模型：现已内置
- 工具调用、结构化输出、思考：继承自兼容 OpenAI 的传输协议
- 图像生成：计划在提供商插件层实现
- 图像 / 视频理解：现已在标准版端点内置
- 语音 / 音频：计划在提供商插件层实现
- 记忆嵌入 / 重排序：计划通过嵌入适配器能力面提供
- 视频生成：现已通过共享视频生成能力内置

## 多模态附加能力

`qwen` 扩展现在还提供：

- 通过 `qwen-vl-max-latest` 实现视频理解
- 通过以下模型实现 Wan 视频生成：
  - `wan2.6-t2v`（默认）
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

这些多模态能力面使用的是**标准版** DashScope 端点，而不是
Coding Plan 端点。

- 全球 / 国际版标准 base URL：`https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- 中国区标准 base URL：`https://dashscope.aliyuncs.com/compatible-mode/v1`

对于视频生成，OpenClaw 会先将已配置的 Qwen 区域映射到对应的
DashScope AIGC 主机，然后再提交任务：

- 全球 / 国际版：`https://dashscope-intl.aliyuncs.com`
- 中国区：`https://dashscope.aliyuncs.com`

这意味着，正常的 `models.providers.qwen.baseUrl` 即使指向
Coding Plan 或标准版 Qwen 主机之一，视频生成仍会使用正确的
区域 DashScope 视频端点。

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

当前内置 Qwen 视频生成限制：

- 每次请求最多 **1** 个输出视频
- 最多 **1** 张输入图像
- 最多 **4** 个输入视频
- 最长 **10 秒** 时长
- 支持 `size`、`aspectRatio`、`resolution`、`audio` 和 `watermark`
- 参考图像 / 视频模式当前要求使用**远程 http(s) URL**。本地
  文件路径会被预先拒绝，因为 DashScope 视频端点不接受这些参考内容的
  本地缓冲区上传。

有关共享工具参数、提供商选择和故障转移行为，请参见
[Video Generation](/zh-CN/tools/video-generation)。

## 环境说明

如果 Gateway 网关 以守护进程（launchd/systemd）方式运行，请确保 `QWEN_API_KEY`
可被该进程访问（例如放在 `~/.openclaw/.env` 中，或通过
`env.shellEnv` 提供）。
