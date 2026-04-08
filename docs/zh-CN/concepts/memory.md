---
read_when:
    - 你想了解内存是如何工作的
    - 你想知道应该写入哪些内存文件
summary: OpenClaw 如何在跨会话中记住内容
title: 内存概览
x-i18n:
    generated_at: "2026-04-08T03:01:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3bb8552341b0b651609edaaae826a22fdc535d240aed4fad4af4b069454004af
    source_path: concepts/memory.md
    workflow: 15
---

# 内存概览

OpenClaw 会通过在你的智能体工作区中写入**纯 Markdown 文件**来记住内容。模型只会“记住”那些被保存到磁盘上的内容——不存在隐藏状态。

## 工作原理

你的智能体有三个与内存相关的文件：

- **`MEMORY.md`** —— 长期记忆。持久化的事实、偏好和决策。会在每次私信会话开始时加载。
- **`memory/YYYY-MM-DD.md`** —— 每日笔记。持续记录的上下文和观察。今天和昨天的笔记会被自动加载。
- **`DREAMS.md`**（实验性，可选）—— 供人工审阅的梦境日记和 dreaming 扫描摘要。

这些文件位于智能体工作区中（默认是 `~/.openclaw/workspace`）。

<Tip>
如果你想让智能体记住某件事，只要直接告诉它：“记住我更喜欢 TypeScript。” 它会把内容写入合适的文件。
</Tip>

## 内存工具

智能体有两个用于处理内存的工具：

- **`memory_search`** —— 使用语义搜索查找相关笔记，即使措辞与原始内容不同也可以找到。
- **`memory_get`** —— 读取特定的内存文件或行范围。

这两个工具都由当前激活的内存插件提供（默认：`memory-core`）。

## Memory Wiki 配套插件

如果你希望持久内存的行为更像一个被维护的知识库，而不只是原始笔记，请使用内置的 `memory-wiki` 插件。

`memory-wiki` 会将持久知识编译为一个 wiki 保险库，包含：

- 确定性的页面结构
- 结构化的主张和证据
- 矛盾与新鲜度跟踪
- 生成的仪表板
- 面向智能体/运行时消费者的编译摘要
- wiki 原生工具，例如 `wiki_search`、`wiki_get`、`wiki_apply` 和 `wiki_lint`

它不会替代当前激活的内存插件。当前激活的内存插件仍然负责回忆、提升和 dreaming。`memory-wiki` 则在其旁边增加了一层带有丰富溯源信息的知识层。

参见 [Memory Wiki](/zh-CN/plugins/memory-wiki)。

## 内存搜索

当配置了嵌入提供商后，`memory_search` 会使用**混合搜索**——将向量相似度（语义含义）与关键词匹配（如 ID 和代码符号等精确术语）结合起来。一旦你为任意受支持的提供商配置了 API 密钥，这项功能即可开箱即用。

<Info>
OpenClaw 会根据可用的 API 密钥自动检测你的嵌入提供商。如果你已配置 OpenAI、Gemini、Voyage 或 Mistral 的密钥，内存搜索会自动启用。
</Info>

有关搜索工作方式、调优选项和提供商设置的详细信息，请参见
[内存搜索](/zh-CN/concepts/memory-search)。

## 内存后端

<CardGroup cols={3}>
<Card title="内置（默认）" icon="database" href="/zh-CN/concepts/memory-builtin">
基于 SQLite。开箱即用，支持关键词搜索、向量相似度和混合搜索。无需额外依赖。
</Card>
<Card title="QMD" icon="search" href="/zh-CN/concepts/memory-qmd">
本地优先的 sidecar，支持重排序、查询扩展，并且能够索引工作区之外的目录。
</Card>
<Card title="Honcho" icon="brain" href="/zh-CN/concepts/memory-honcho">
AI 原生的跨会话内存，支持用户建模、语义搜索和多智能体感知。需要安装插件。
</Card>
</CardGroup>

## 知识 wiki 层

<CardGroup cols={1}>
<Card title="Memory Wiki" icon="book" href="/zh-CN/plugins/memory-wiki">
将持久内存编译为一个具有丰富溯源信息的 wiki 保险库，包含主张、仪表板、桥接模式，以及对 Obsidian 友好的工作流。
</Card>
</CardGroup>

## 自动内存刷新

在 [压缩](/zh-CN/concepts/compaction) 总结你的对话之前，OpenClaw 会运行一个静默轮次，提醒智能体将重要上下文保存到内存文件中。此功能默认开启——你无需进行任何配置。

<Tip>
内存刷新可以防止在压缩期间丢失上下文。如果你的智能体在对话中有尚未写入文件的重要事实，这些内容会在摘要生成之前自动保存。
</Tip>

## Dreaming（实验性）

Dreaming 是一种可选的内存后台整合流程。它会收集短期信号、为候选项打分，并仅将符合条件的项目提升到长期记忆中（`MEMORY.md`）。

它旨在让长期记忆保持高信噪比：

- **选择加入**：默认禁用。
- **定时执行**：启用后，`memory-core` 会自动管理一个周期性 cron 任务，用于执行完整的 dreaming 扫描。
- **阈值控制**：提升必须通过分数、回忆频率和查询多样性门槛。
- **可审阅**：阶段摘要和日记条目会写入 `DREAMS.md`，供人工审阅。

有关阶段行为、评分信号和梦境日记详情，请参见
[Dreaming（实验性）](/zh-CN/concepts/dreaming)。

## CLI

```bash
openclaw memory status          # 检查索引状态和提供商
openclaw memory search "query"  # 从命令行搜索
openclaw memory index --force   # 重建索引
```

## 延伸阅读

- [内置内存引擎](/zh-CN/concepts/memory-builtin) —— 默认的 SQLite 后端
- [QMD 内存引擎](/zh-CN/concepts/memory-qmd) —— 高级本地优先 sidecar
- [Honcho 内存](/zh-CN/concepts/memory-honcho) —— AI 原生的跨会话内存
- [Memory Wiki](/zh-CN/plugins/memory-wiki) —— 编译后的知识保险库和 wiki 原生工具
- [内存搜索](/zh-CN/concepts/memory-search) —— 搜索流水线、提供商和调优
- [Dreaming（实验性）](/zh-CN/concepts/dreaming) —— 将短期回忆后台提升到长期记忆
- [内存配置参考](/zh-CN/reference/memory-config) —— 所有配置项
- [压缩](/zh-CN/concepts/compaction) —— 压缩如何与内存交互
