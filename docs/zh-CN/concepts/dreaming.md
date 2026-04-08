---
read_when:
    - 你希望记忆提升自动运行
    - 你希望了解每个 dreaming 阶段的作用
    - 你希望调整整合过程而不污染 `MEMORY.md`
summary: 带有浅睡、深睡和 REM 阶段以及 Dream Diary 的后台记忆整合
title: Dreaming（实验性）
x-i18n:
    generated_at: "2026-04-08T06:28:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0254f3b0949158264e583c12f36f2b1a83d1b44dc4da01a1b272422d38e8655d
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming（实验性）

Dreaming 是 `memory-core` 中的后台记忆整合系统。
它帮助 OpenClaw 将强烈的短期信号转入持久记忆，同时
保持过程可解释且可审查。

Dreaming 为**可选启用**，默认禁用。

## Dreaming 会写入什么

Dreaming 会保留两类输出：

- `memory/.dreams/` 中的**机器状态**（召回存储、阶段信号、摄取检查点、锁）。
- `DREAMS.md`（或现有的 `dreams.md`）中的**人类可读输出**，以及位于 `memory/dreaming/<phase>/YYYY-MM-DD.md` 下的可选阶段报告文件。

长期提升仍然只会写入 `MEMORY.md`。

## 阶段模型

Dreaming 使用三个协作阶段：

| 阶段 | 目的 | 持久写入 |
| ----- | ----------------------------------------- | ----------------- |
| 浅睡 | 整理并暂存近期短期材料 | 否 |
| 深睡 | 为持久候选项评分并提升 | 是（`MEMORY.md`） |
| REM   | 反思主题和反复出现的想法 | 否 |

这些阶段是内部实现细节，不是单独的用户可配置“模式”。

### 浅睡阶段

浅睡阶段会摄取近期的每日记忆信号和召回痕迹，对其去重，
并暂存候选条目。

- 从短期召回状态、近期每日记忆文件以及脱敏后的会话转录中读取（如果可用）。
- 当存储包含内联输出时，会写入一个受管理的 `## Light Sleep` 区块。
- 记录强化信号，供后续深度排序使用。
- 绝不会写入 `MEMORY.md`。

### 深睡阶段

深睡阶段决定哪些内容会成为长期记忆。

- 使用加权评分和阈值门槛对候选项进行排序。
- 要求通过 `minScore`、`minRecallCount` 和 `minUniqueQueries`。
- 在写入前会从实时每日文件中重新提取片段，因此过期或已删除的片段会被跳过。
- 将提升后的条目追加到 `MEMORY.md`。
- 将 `## Deep Sleep` 摘要写入 `DREAMS.md`，并可选择写入 `memory/dreaming/deep/YYYY-MM-DD.md`。

### REM 阶段

REM 阶段提取模式和反思信号。

- 根据近期短期痕迹构建主题和反思摘要。
- 当存储包含内联输出时，会写入一个受管理的 `## REM Sleep` 区块。
- 记录用于深度排序的 REM 强化信号。
- 绝不会写入 `MEMORY.md`。

## 会话转录摄取

Dreaming 可以将脱敏后的会话转录摄取到 dreaming 语料中。当
转录可用时，它们会与每日记忆信号和召回痕迹一起输入浅睡
阶段。个人内容和敏感内容会在摄取前被脱敏。

## Dream Diary

Dreaming 还会在 `DREAMS.md` 中保留一份叙事性的**Dream Diary**。
每个阶段积累了足够材料后，`memory-core` 会以尽力而为的方式运行一次后台
子智能体轮次（使用默认运行时模型），并追加一段简短的日记条目。

这份日记用于在 Dreams UI 中供人阅读，而不是作为提升来源。

## 深度排序信号

深度排序使用六个加权基础信号以及阶段强化：

| 信号 | 权重 | 描述 |
| ------------------- | ------ | ------------------------------------------------- |
| 频率 | 0.24   | 该条目积累了多少短期信号 |
| 相关性 | 0.30   | 该条目的平均检索质量 |
| 查询多样性 | 0.15   | 使其浮现的不同查询 / 日期上下文 |
| 近期性 | 0.15   | 随时间衰减的新鲜度分数 |
| 整合度 | 0.10   | 跨天重复出现的强度 |
| 概念丰富度 | 0.06   | 来自片段 / 路径的概念标签密度 |

浅睡和 REM 阶段命中会从
`memory/.dreams/phase-signals.json` 中增加一个小幅的、按近期性衰减的加成。

## 调度

启用后，`memory-core` 会自动管理一个 cron 任务来执行一次完整的 dreaming
扫描。每次扫描会按顺序运行各阶段：light -> REM -> deep。

默认频率行为：

| 设置 | 默认值 |
| -------------------- | ----------- |
| `dreaming.frequency` | `0 3 * * *` |

## 快速开始

启用 dreaming：

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

使用自定义扫描频率启用 dreaming：

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

手动执行 `memory promote` 默认使用深睡阶段阈值，除非通过
CLI 标志覆盖。

解释某个特定候选项为何会或不会被提升：

```bash
openclaw memory promote-explain "router vlan"
openclaw memory promote-explain "router vlan" --json
```

预览 REM 反思、候选事实和深度提升输出，而不写入任何内容：

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

完整键名列表请参阅 [Memory 配置参考](/zh-CN/reference/memory-config#dreaming-experimental)。

## Dreams UI

启用后，Gateway 网关的 **Dreams** 选项卡会显示：

- 当前 dreaming 启用状态
- 阶段级状态和受管扫描是否存在
- 短期、长期和今日已提升数量
- 下次计划运行时间
- 一个可展开的 Dream Diary 阅读器，由 `doctor.memory.dreamDiary` 提供支持

## 相关

- [Memory](/zh-CN/concepts/memory)
- [Memory Search](/zh-CN/concepts/memory-search)
- [memory CLI](/cli/memory)
- [Memory 配置参考](/zh-CN/reference/memory-config)
