---
read_when:
    - 说明流式传输或分块在各渠道中的工作方式
    - 更改分块流式传输或渠道分块行为
    - 调试重复/过早的分块回复或渠道预览流式传输
summary: 流式传输 + 分块行为（分块回复、渠道预览流式传输、模式映射）
title: 流式传输和分块
x-i18n:
    generated_at: "2026-04-08T04:59:54Z"
    model: gpt-5.4
    provider: openai
    source_hash: a8e847bb7da890818cd79dec7777f6ae488e6d6c0468e948e56b6b6c598e0000
    source_path: concepts/streaming.md
    workflow: 15
---

# 流式传输 + 分块

OpenClaw 有两层彼此独立的流式传输：

- **分块流式传输（渠道）：** 在助手写作时发送已完成的**块**。这些是正常的渠道消息（不是 token 增量）。
- **预览流式传输（Telegram/Discord/Slack）：** 在生成过程中更新一个临时的**预览消息**。

目前还**没有真正的 token 增量流式传输**发送到渠道消息。预览流式传输是基于消息的（发送 + 编辑/追加）。

## 分块流式传输（渠道消息）

分块流式传输会在助手输出可用时，以较粗粒度的块发送。

```
Model output
  └─ text_delta/events
       ├─ (blockStreamingBreak=text_end)
       │    └─ chunker emits blocks as buffer grows
       └─ (blockStreamingBreak=message_end)
            └─ chunker flushes at message_end
                   └─ channel send (block replies)
```

图例：

- `text_delta/events`：模型流事件（对于非流式模型，这些事件可能很稀疏）。
- `chunker`：应用最小/最大边界和断点偏好的 `EmbeddedBlockChunker`。
- `channel send`：实际发出的出站消息（分块回复）。

**控制项：**

- `agents.defaults.blockStreamingDefault`：`"on"`/`"off"`（默认关闭）。
- 渠道覆盖：`*.blockStreaming`（以及每账号变体）可为每个渠道强制设为 `"on"`/`"off"`。
- `agents.defaults.blockStreamingBreak`：`"text_end"` 或 `"message_end"`。
- `agents.defaults.blockStreamingChunk`：`{ minChars, maxChars, breakPreference? }`。
- `agents.defaults.blockStreamingCoalesce`：`{ minChars?, maxChars?, idleMs? }`（在发送前合并流式块）。
- 渠道硬上限：`*.textChunkLimit`（例如 `channels.whatsapp.textChunkLimit`）。
- 渠道分块模式：`*.chunkMode`（默认 `length`，`newline` 会先按空行即段落边界拆分，再按长度分块）。
- Discord 软上限：`channels.discord.maxLinesPerMessage`（默认 17）会拆分过高的回复，以避免 UI 裁剪。

**边界语义：**

- `text_end`：一旦 chunker 产生块就立即流式发送；在每个 `text_end` 时刷新。
- `message_end`：等待助手消息完成后，再刷新缓冲输出。

`message_end` 仍然会在缓冲文本超过 `maxChars` 时使用 chunker，因此它可以在结尾发出多个块。

## 分块算法（低/高边界）

分块由 `EmbeddedBlockChunker` 实现：

- **低边界：** 在缓冲区达到 `minChars` 之前不发送（除非被强制发送）。
- **高边界：** 优先在 `maxChars` 之前拆分；如果必须强制拆分，就在 `maxChars` 处拆分。
- **断点偏好：** `paragraph` → `newline` → `sentence` → `whitespace` → 硬拆分。
- **代码围栏：** 绝不在围栏内部拆分；如果被迫在 `maxChars` 处拆分，会先关闭再重新打开围栏，以保持 Markdown 有效。

`maxChars` 会被限制到渠道的 `textChunkLimit`，因此你无法超过各渠道的上限。

## 合并（合并流式块）

启用分块流式传输时，OpenClaw 可以在发送前**合并连续的分块**。
这样既能提供渐进式输出，又能减少“单行刷屏”。

- 合并会等待**空闲间隔**（`idleMs`）后再刷新。
- 缓冲区受 `maxChars` 限制，超过时会刷新。
- `minChars` 会阻止过小的片段被提前发送，直到积累足够文本
  （最终刷新始终会发送剩余文本）。
- 连接符由 `blockStreamingChunk.breakPreference`
  决定（`paragraph` → `\n\n`，`newline` → `\n`，`sentence` → 空格）。
- 可通过 `*.blockStreamingCoalesce` 进行渠道级覆盖（包括每账号配置）。
- 对于 Signal/Slack/Discord，默认合并 `minChars` 会提升到 1500，除非被覆盖。

## 块之间的类人节奏

启用分块流式传输时，你可以在分块回复之间加入**随机暂停**
（第一个块之后）。这会让多气泡回复显得更自然。

- 配置：`agents.defaults.humanDelay`（可通过 `agents.list[].humanDelay` 按智能体覆盖）。
- 模式：`off`（默认）、`natural`（800–2500ms）、`custom`（`minMs`/`maxMs`）。
- 仅适用于**分块回复**，不适用于最终回复或工具摘要。

## “流式发送分块，还是一次性全部发送”

它对应为：

- **流式发送分块：** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"`（边生成边发送）。非 Telegram 渠道还需要将 `*.blockStreaming` 设为 `true`。
- **在结尾一次性发送全部内容：** `blockStreamingBreak: "message_end"`（刷新一次；如果内容很长，可能仍会拆成多个块）。
- **不使用分块流式传输：** `blockStreamingDefault: "off"`（只发送最终回复）。

**渠道说明：** 除非显式将
`*.blockStreaming` 设为 `true`，否则分块流式传输**默认关闭**。渠道仍然可以启用实时预览
（`channels.<channel>.streaming`），而不发送分块回复。

配置位置提醒：`blockStreaming*` 默认值位于
`agents.defaults` 下，而不是根配置下。

## 预览流式传输模式

规范键名：`channels.<channel>.streaming`

模式：

- `off`：禁用预览流式传输。
- `partial`：使用单个预览，并用最新文本替换。
- `block`：以分块/追加的方式更新预览。
- `progress`：在生成期间显示进度/状态预览，完成后给出最终答案。

### 渠道映射

| 渠道 | `off` | `partial` | `block` | `progress` |
| ---- | ----- | --------- | ------- | ---------- |
| Telegram | ✅ | ✅ | ✅ | 映射为 `partial` |
| Discord | ✅ | ✅ | ✅ | 映射为 `partial` |
| Slack | ✅ | ✅ | ✅ | ✅ |

仅限 Slack：

- `channels.slack.streaming.nativeTransport` 在 `channels.slack.streaming.mode="partial"` 时切换 Slack 原生流式 API 调用（默认：`true`）。
- Slack 原生流式传输和 Slack 助手线程状态都需要回复线程目标；顶层私信不会显示这种线程式预览。

旧键迁移：

- Telegram：`streamMode` 和布尔值 `streaming` 会自动迁移到枚举 `streaming`。
- Discord：`streamMode` 和布尔值 `streaming` 会自动迁移到枚举 `streaming`。
- Slack：`streamMode` 会自动迁移到 `streaming.mode`；布尔值 `streaming` 会自动迁移到 `streaming.mode` 加 `streaming.nativeTransport`；旧版 `nativeStreaming` 会自动迁移到 `streaming.nativeTransport`。

### 运行时行为

Telegram：

- 在私信和群组/话题中使用 `sendMessage` + `editMessageText` 更新预览。
- 当 Telegram 分块流式传输被显式启用时，会跳过预览流式传输（以避免双重流式传输）。
- `/reasoning stream` 可以将 reasoning 写入预览。

Discord：

- 使用发送 + 编辑预览消息。
- `block` 模式使用草稿分块（`draftChunk`）。
- 当 Discord 分块流式传输被显式启用时，会跳过预览流式传输。

Slack：

- `partial` 在可用时可以使用 Slack 原生流式传输（`chat.startStream`/`append`/`stop`）。
- `block` 使用追加式草稿预览。
- `progress` 使用状态预览文本，然后发送最终答案。

## 相关内容

- [消息](/zh-CN/concepts/messages) — 消息生命周期和投递
- [重试](/zh-CN/concepts/retry) — 投递失败时的重试行为
- [渠道](/zh-CN/channels) — 各渠道的流式传输支持
