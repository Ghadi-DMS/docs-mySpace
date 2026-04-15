---
read_when:
    - 你希望记忆提升自动运行
    - 你想了解每个做梦阶段的作用
    - 你想在不污染 `MEMORY.md` 的情况下调整巩固过程
summary: 通过浅睡、深睡和 REM 阶段进行后台记忆巩固，并配有梦境日记
title: 做梦（实验性）
x-i18n:
    generated_at: "2026-04-15T00:30:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5882a5068f2eabe54ca9893184e5385330a432b921870c38626399ce11c31e25
    source_path: concepts/dreaming.md
    workflow: 15
---

# 做梦（实验性）

做梦是 `memory-core` 中的后台记忆巩固系统。
它帮助 OpenClaw 将强烈的短期信号转移到持久记忆中，同时
让整个过程保持可解释且可审查。

做梦为**选择启用**功能，默认禁用。

## 做梦会写入什么

做梦会保留两类输出：

- `memory/.dreams/` 中的**机器状态**（召回存储、阶段信号、摄取检查点、锁）。
- `DREAMS.md`（或现有的 `dreams.md`）中的**人类可读输出**，以及可选的 `memory/dreaming/<phase>/YYYY-MM-DD.md` 阶段报告文件。

长期提升仍然只会写入 `MEMORY.md`。

## 阶段模型

做梦使用三个协作阶段：

| 阶段 | 目的 | 持久写入 |
| ----- | ----------------------------------------- | ----------------- |
| 浅睡 | 对近期短期材料进行整理和暂存 | 否 |
| 深睡 | 对持久候选项进行评分并提升 | 是（`MEMORY.md`） |
| REM | 反思主题和反复出现的想法 | 否 |

这些阶段是内部实现细节，不是单独的用户可配置“模式”。

### 浅睡阶段

浅睡阶段会摄取近期的每日记忆信号和召回轨迹，对其去重，
并暂存候选条目。

- 在可用时，从短期召回状态、近期每日记忆文件以及已脱敏的会话转录中读取。
- 当存储包含内联输出时，写入受管控的 `## 浅睡` 区块。
- 记录强化信号，供后续深睡排序使用。
- 绝不会写入 `MEMORY.md`。

### 深睡阶段

深睡阶段决定哪些内容会变成长期开启的记忆。

- 使用加权评分和阈值门槛对候选项进行排序。
- 要求通过 `minScore`、`minRecallCount` 和 `minUniqueQueries`。
- 在写入前从实时每日文件中重新提取片段，因此过时或已删除的片段会被跳过。
- 将提升后的条目追加到 `MEMORY.md`。
- 将 `## 深睡` 摘要写入 `DREAMS.md`，并可选写入 `memory/dreaming/deep/YYYY-MM-DD.md`。

### REM 阶段

REM 阶段会提取模式和反思信号。

- 从近期短期轨迹中构建主题和反思摘要。
- 当存储包含内联输出时，写入受管控的 `## REM Sleep` 区块。
- 记录供深睡排序使用的 REM 强化信号。
- 绝不会写入 `MEMORY.md`。

## 会话转录摄取

做梦可以将已脱敏的会话转录摄取到做梦语料中。当
转录可用时，它们会与每日记忆信号和召回轨迹一起送入浅睡阶段。个人和敏感内容会在摄取前被脱敏。

## 梦境日记

做梦还会在 `DREAMS.md` 中保留一份叙事性的**梦境日记**。
每个阶段积累了足够材料后，`memory-core` 会尽力运行一次后台
子智能体轮次（使用默认运行时模型），并追加一条简短的日记条目。

这份日记用于人在 Dreams UI 中阅读，而不是作为提升来源。
由做梦生成的日记/报告工件会被排除在短期
提升之外。只有有依据的记忆片段才有资格被提升到
`MEMORY.md` 中。

此外，还有一条基于依据的历史回填通道，用于审查和恢复工作：

- `memory rem-harness --path ... --grounded` 会从历史 `YYYY-MM-DD.md` 笔记中预览基于依据的日记输出。
- `memory rem-backfill --path ...` 会将可逆的、基于依据的日记条目写入 `DREAMS.md`。
- `memory rem-backfill --path ... --stage-short-term` 会将基于依据的持久候选项暂存到与常规深睡阶段已使用的同一短期证据存储中。
- `memory rem-backfill --rollback` 和 `--rollback-short-term` 会移除这些已暂存的回填工件，而不会触及普通日记条目或实时短期召回。

Control UI 也提供同样的日记回填/重置流程，因此你可以先在 Dreams 场景中检查
结果，再决定这些基于依据的候选项是否值得提升。
该场景还会显示一条独立的基于依据通道，这样你就能看到
哪些已暂存的短期条目来自历史回放、哪些已提升项目由依据驱动，并且只清除仅基于依据的已暂存条目，而不会影响普通的实时短期状态。

## 深睡排序信号

深睡排序使用六个加权基础信号，加上阶段强化：

| 信号 | 权重 | 描述 |
| ------------------- | ------ | ------------------------------------------------- |
| 频率 | 0.24 | 条目积累了多少短期信号 |
| 相关性 | 0.30 | 条目的平均检索质量 |
| 查询多样性 | 0.15 | 使其浮现的不同查询/日期上下文 |
| 时效性 | 0.15 | 随时间衰减的新鲜度评分 |
| 巩固度 | 0.10 | 跨日重复出现的强度 |
| 概念丰富度 | 0.06 | 来自片段/路径的概念标签密度 |

浅睡和 REM 阶段命中会从
`memory/.dreams/phase-signals.json` 增加一个随时间衰减的小幅提升。

## 调度

启用后，`memory-core` 会自动管理一个 cron 任务，用于执行完整的做梦
扫描。每次扫描都会按顺序运行各阶段：浅睡 -> REM -> 深睡。

默认节奏行为：

| 设置 | 默认值 |
| -------------------- | ----------- |
| `dreaming.frequency` | `0 3 * * *` |

## 快速开始

启用做梦：

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

使用自定义扫描节奏启用做梦：

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "timezone": "America/Los_Angeles",
            "frequency": "0 */6 * * *"
          }
        }
      }
    }
  }
}
```

## 斜杠命令

```
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

## CLI 工作流

使用 CLI 提升进行预览或手动应用：

```bash
openclaw memory promote
openclaw memory promote --apply
openclaw memory promote --limit 5
openclaw memory status --deep
```

手动 `memory promote` 默认使用深睡阶段阈值，除非通过
CLI 标志覆盖。

解释某个特定候选项为什么会或不会被提升：

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

预览 REM 反思、候选事实和深睡提升输出，
且不写入任何内容：

```bash
openclaw memory rem-harness
openclaw memory rem-harness --json
```

## 关键默认值

所有设置都位于 `plugins.entries.memory-core.config.dreaming` 下。

| 键 | 默认值 |
| ----------- | ----------- |
| `enabled`   | `false`     |
| `frequency` | `0 3 * * *` |

阶段策略、阈值和存储行为都是内部实现
细节（不是面向用户的配置）。

完整键名列表请参阅 [记忆配置参考](/zh-CN/reference/memory-config#dreaming-experimental)。

## Dreams UI

启用后，Gateway 网关中的 **Dreams** 选项卡会显示：

- 当前做梦启用状态
- 阶段级状态和受管控扫描是否存在
- 短期、基于依据、信号以及今日已提升计数
- 下一次计划运行时间
- 一条用于已暂存历史回放条目的独立 grounded 场景通道
- 由 `doctor.memory.dreamDiary` 支持的可展开梦境日记阅读器

## 相关内容

- [记忆](/zh-CN/concepts/memory)
- [记忆搜索](/zh-CN/concepts/memory-search)
- [memory CLI](/cli/memory)
- [记忆配置参考](/zh-CN/reference/memory-config)
