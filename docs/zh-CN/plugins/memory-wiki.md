---
read_when:
    - 你想要超出普通 `MEMORY.md` 笔记之外的持久化知识
    - 你正在配置内置的 memory-wiki 插件
    - 你想了解 `wiki_search`、`wiki_get` 或桥接模式
summary: memory-wiki：带有来源溯源、声明、仪表板和桥接模式的编译式知识库
title: Memory Wiki
x-i18n:
    generated_at: "2026-04-08T03:01:24Z"
    model: gpt-5.4
    provider: openai
    source_hash: b78dd6a4ef4451dae6b53197bf0c7c2a2ba846b08e4a3a93c1026366b1598d82
    source_path: plugins/memory-wiki.md
    workflow: 15
---

# Memory Wiki

`memory-wiki` 是一个内置插件，可将持久化记忆转变为一个编译式知识库。

它**不会**替代当前激活的记忆插件。当前激活的记忆插件仍然负责召回、提升、索引和梦境生成。`memory-wiki` 与其并行工作，将持久化知识编译为一个可导航的 wiki，包含确定性的页面、结构化声明、来源溯源、仪表板以及机器可读摘要。

当你希望记忆更像一个经过维护的知识层，而不是一堆 Markdown 文件时，就可以使用它。

## 它新增了什么

- 具有确定性页面布局的专用 wiki 资料库
- 结构化的声明和证据元数据，而不只是纯文本内容
- 页面级别的来源溯源、置信度、矛盾点和待解问题
- 面向智能体/运行时使用者的编译摘要
- wiki 原生的搜索/获取/应用/lint 工具
- 可选的桥接模式，可从当前激活的记忆插件导入公开产物
- 可选的对 Obsidian 友好的渲染模式和 CLI 集成

## 它如何与记忆协同工作

可以这样理解这个划分：

| 层 | 负责 |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| 当前激活的记忆插件（`memory-core`、QMD、Honcho 等） | 召回、语义搜索、提升、梦境生成、记忆运行时 |
| `memory-wiki` | 编译后的 wiki 页面、富含来源溯源的综合内容、仪表板、wiki 专用搜索/获取/应用 |

如果当前激活的记忆插件暴露了共享的召回产物，OpenClaw 就可以通过 `memory_search corpus=all` 在一次检索中同时搜索这两层。

当你需要 wiki 专用排序、来源溯源或直接页面访问时，请改用 wiki 原生工具。

## 资料库模式

`memory-wiki` 支持三种资料库模式：

### `isolated`

独立资料库、独立来源，不依赖 `memory-core`。

当你希望 wiki 成为它自己的策展式知识存储时，使用此模式。

### `bridge`

通过公开的插件 SDK 接口，从当前激活的记忆插件读取公开的记忆产物和记忆事件。

当你希望 wiki 在不触及私有插件内部实现的前提下，对记忆插件导出的产物进行编译和组织时，使用此模式。

桥接模式可以索引：

- 导出的记忆产物
- 梦境报告
- 每日日志
- 记忆根文件
- 记忆事件日志

### `unsafe-local`

用于本地私有路径的、显式的同机逃生口。

此模式有意保持实验性且不可移植。只有在你理解其信任边界，并且确实需要桥接模式无法提供的本地文件系统访问时，才应使用它。

## 资料库布局

该插件会初始化如下资料库：

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/
  concepts/
  syntheses/
  sources/
  reports/
  _attachments/
  _views/
  .openclaw-wiki/
```

受管理的内容会保留在生成块内。人工笔记块会被保留。

主要页面分组如下：

- `sources/` 用于导入的原始材料和由桥接支持的页面
- `entities/` 用于持久对象、人物、系统、项目和实物
- `concepts/` 用于思想、抽象概念、模式和策略
- `syntheses/` 用于编译后的摘要和维护中的汇总
- `reports/` 用于生成的仪表板

## 结构化声明和证据

页面可以携带结构化的 `claims` frontmatter，而不只是自由文本。

每条声明可以包含：

- `id`
- `text`
- `status`
- `confidence`
- `evidence[]`
- `updatedAt`

证据条目可以包含：

- `sourceId`
- `path`
- `lines`
- `weight`
- `note`
- `updatedAt`

这使 wiki 更像一个信念层，而不是被动的笔记转储。声明可以被跟踪、评分、质疑，并回溯解析到来源。

## 编译流水线

编译步骤会读取 wiki 页面、规范化摘要，并在以下位置输出稳定的面向机器的产物：

- `.openclaw-wiki/cache/agent-digest.json`
- `.openclaw-wiki/cache/claims.jsonl`

这些摘要的存在，是为了让智能体和运行时代码不必去抓取 Markdown 页面。

编译输出还支持：

- 搜索/获取流程的首轮 wiki 索引
- 从 claim id 回查所属页面
- 紧凑的提示词补充
- 报告/仪表板生成

## 仪表板和健康报告

启用 `render.createDashboards` 时，编译过程会在 `reports/` 下维护仪表板。

内置报告包括：

- `reports/open-questions.md`
- `reports/contradictions.md`
- `reports/low-confidence.md`
- `reports/claim-health.md`
- `reports/stale-pages.md`

这些报告会跟踪如下内容：

- 矛盾备注簇
- 相互竞争的声明簇
- 缺少结构化证据的声明
- 低置信度页面和声明
- 过时或新鲜度未知的内容
- 存在未解决问题的页面

## 搜索和检索

`memory-wiki` 支持两种搜索后端：

- `shared`：在可用时使用共享记忆搜索流程
- `local`：在本地搜索 wiki

它还支持三种语料范围：

- `wiki`
- `memory`
- `all`

重要行为：

- `wiki_search` 和 `wiki_get` 会在可能时优先使用编译摘要进行首轮处理
- claim id 可以解析回其所属页面
- 有争议/过时/新鲜的声明会影响排序
- 来源溯源标签可以保留到结果中

实用规则：

- 使用 `memory_search corpus=all` 进行一次广泛的召回检索
- 当你关心 wiki 专用排序、来源溯源或页面级信念结构时，使用 `wiki_search` + `wiki_get`

## 智能体工具

该插件会注册以下工具：

- `wiki_status`
- `wiki_search`
- `wiki_get`
- `wiki_apply`
- `wiki_lint`

它们的作用：

- `wiki_status`：当前资料库模式、健康状态、Obsidian CLI 可用性
- `wiki_search`：搜索 wiki 页面，并在已配置时搜索共享记忆语料
- `wiki_get`：按 id/path 读取 wiki 页面，或回退到共享记忆语料
- `wiki_apply`：执行窄范围的综合内容/元数据变更，而不是自由形式的页面编辑
- `wiki_lint`：执行结构检查、来源溯源缺口检查、矛盾检查和待解问题检查

该插件还会注册一个非独占的记忆语料补充，因此当当前激活的记忆插件支持语料选择时，共享的 `memory_search` 和 `memory_get` 也可以访问 wiki。

## 提示词和上下文行为

启用 `context.includeCompiledDigestPrompt` 时，记忆提示词部分会附加来自 `agent-digest.json` 的紧凑编译快照。

该快照有意保持小而高信号：

- 仅包含顶部页面
- 仅包含顶部声明
- 矛盾计数
- 问题计数
- 置信度/新鲜度限定词

这是一个可选项，因为它会改变提示词形状，主要适用于显式消费记忆补充内容的上下文引擎或旧版提示词组装流程。

## 配置

将配置放在 `plugins.entries.memory-wiki.config` 下：

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian",
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false,
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true,
          },
          search: {
            backend: "shared",
            corpus: "wiki",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true,
          },
        },
      },
    },
  },
}
```

关键开关：

- `vaultMode`：`isolated`、`bridge`、`unsafe-local`
- `vault.renderMode`：`native` 或 `obsidian`
- `bridge.readMemoryArtifacts`：导入当前激活的记忆插件公开产物
- `bridge.followMemoryEvents`：在桥接模式下包含事件日志
- `search.backend`：`shared` 或 `local`
- `search.corpus`：`wiki`、`memory` 或 `all`
- `context.includeCompiledDigestPrompt`：将紧凑摘要快照附加到记忆提示词部分
- `render.createBacklinks`：生成确定性的相关块
- `render.createDashboards`：生成仪表板页面

## CLI

`memory-wiki` 还暴露了一个顶层 CLI 接口：

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki get entity.alpha
openclaw wiki apply synthesis "Alpha Summary" --body "..." --source-id source.alpha
openclaw wiki bridge import
openclaw wiki obsidian status
```

完整命令参考请参见 [CLI：wiki](/cli/wiki)。

## Obsidian 支持

当 `vault.renderMode` 为 `obsidian` 时，该插件会写出对 Obsidian 友好的 Markdown，并且可以选择性使用官方 `obsidian` CLI。

支持的工作流包括：

- 状态探测
- 资料库搜索
- 打开页面
- 调用 Obsidian 命令
- 跳转到每日日志

这是可选功能。即使不使用 Obsidian，wiki 仍然可以在原生模式下工作。

## 推荐工作流

1. 保留你当前激活的记忆插件，用于召回/提升/梦境生成。
2. 启用 `memory-wiki`。
3. 除非你明确需要桥接模式，否则从 `isolated` 模式开始。
4. 当来源溯源很重要时，使用 `wiki_search` / `wiki_get`。
5. 对于窄范围的综合内容或元数据更新，使用 `wiki_apply`。
6. 在有意义的更改之后运行 `wiki_lint`。
7. 如果你想查看过时内容/矛盾情况的可视化，请开启仪表板。

## 相关文档

- [Memory 概览](/zh-CN/concepts/memory)
- [CLI：memory](/cli/memory)
- [CLI：wiki](/cli/wiki)
- [插件 SDK 概览](/zh-CN/plugins/sdk-overview)
