---
read_when:
    - 你希望记忆提升能够自动运行
    - 你希望了解 Dreaming 的三个阶段
    - 你希望调整整合过程，而不污染 MEMORY.md
summary: 具有三个协作阶段的后台记忆整合：轻度、深度和 REM
title: Dreaming（实验性）
x-i18n:
    generated_at: "2026-04-05T17:47:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: a038e6e010af895abde5524727b11be224aa27d630ae1e9bc056e616777d9714
    source_path: concepts/dreaming.md
    workflow: 15
---

# Dreaming（实验性）

Dreaming 是 `memory-core` 中的后台记忆整合系统。它会重新审视对话中出现的内容，并决定哪些内容值得作为持久上下文保留下来。

Dreaming 使用三个协作的**阶段**，而不是相互竞争的模式。每个阶段都有明确不同的职责、写入到不同的目标，并按照各自的计划运行。

## 三个阶段

### Light

Light Dreaming 会整理最近的杂乱内容。它会扫描最近的记忆痕迹，按 Jaccard 相似度去重，对相关条目进行聚类，并将候选记忆暂存到每日记忆笔记中（`memory/YYYY-MM-DD.md`）。

Light **不会**向 `MEMORY.md` 写入任何内容。它只负责组织和暂存。可以理解为：“今天有哪些内容以后可能会重要？”

### Deep

Deep Dreaming 会决定哪些内容会成为持久记忆。它运行真正的提升逻辑：基于六个信号的加权评分、阈值门槛、召回次数、唯一查询多样性、近期衰减，以及最大年龄过滤。

Deep 是**唯一**允许将持久事实写入 `MEMORY.md` 的阶段。它还负责在记忆稀薄时进行恢复（当健康度低于配置阈值时）。可以理解为：“哪些内容足够真实，值得保留？”

### REM

REM Dreaming 会寻找模式并进行反思。它会检查近期材料，通过概念标签聚类识别重复出现的主题，并将更高层次的笔记和反思写入每日笔记。

REM 会写入每日笔记（`memory/YYYY-MM-DD.md`），**不会**写入 `MEMORY.md`。它的输出是解释性的，而不是规范性的。可以理解为：“我注意到了什么模式？”

## 硬性边界

| 阶段 | 职责 | 写入到 | 不会写入到 |
| ----- | --------- | -------------------------- | ----------------- |
| Light | 整理 | 每日笔记（YYYY-MM-DD.md） | MEMORY.md |
| Deep  | 保留 | MEMORY.md | -- |
| REM   | 解释 | 每日笔记（YYYY-MM-DD.md） | MEMORY.md |

## 快速开始

启用全部三个阶段（推荐）：

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

仅启用深度提升：

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true,
            "phases": {
              "light": { "enabled": false },
              "deep": { "enabled": true },
              "rem": { "enabled": false }
            }
          }
        }
      }
    }
  }
}
```

## 配置

所有 Dreaming 设置都位于 `openclaw.json` 的 `plugins.entries.memory-core.config.dreaming` 下。完整键名列表请参阅 [记忆配置参考](/zh-CN/reference/memory-config#dreaming-experimental)。

### 全局设置

| 键名 | 类型 | 默认值 | 说明 |
| ---------------- | --------- | ---------- | ------------------------------------------------ |
| `enabled` | `boolean` | `true` | 所有阶段的总开关 |
| `timezone` | `string` | 未设置 | 用于计划评估和每日笔记的时区 |
| `verboseLogging` | `boolean` | `false` | 输出详细的每次 Dreaming 运行日志 |
| `storage.mode` | `string` | `"inline"` | `inline`、`separate` 或 `both` |

### Light 阶段配置

| 键名 | 类型 | 默认值 | 说明 |
| ------------------ | ---------- | ------------------------------- | --------------------------------- |
| `enabled` | `boolean` | `true` | 启用 Light 阶段 |
| `cron` | `string` | `0 */6 * * *` | 计划（默认：每 6 小时一次） |
| `lookbackDays` | `number` | `2` | 要扫描多少天的痕迹 |
| `limit` | `number` | `100` | 每次运行最多暂存的候选项数 |
| `dedupeSimilarity` | `number` | `0.9` | 去重的 Jaccard 阈值 |
| `sources` | `string[]` | `["daily","sessions","recall"]` | 要扫描的数据源 |

### Deep 阶段配置

| 键名 | 类型 | 默认值 | 说明 |
| --------------------- | ---------- | ----------------------------------------------- | ------------------------------------ |
| `enabled` | `boolean` | `true` | 启用 Deep 阶段 |
| `cron` | `string` | `0 3 * * *` | 计划（默认：每天凌晨 3 点） |
| `limit` | `number` | `10` | 每个周期最多提升的候选项数 |
| `minScore` | `number` | `0.8` | 提升所需的最低加权分数 |
| `minRecallCount` | `number` | `3` | 最低召回次数阈值 |
| `minUniqueQueries` | `number` | `3` | 最低不同查询数 |
| `recencyHalfLifeDays` | `number` | `14` | 近期分数减半所需天数 |
| `maxAgeDays` | `number` | `30` | 可提升的每日笔记最大年龄 |
| `sources` | `string[]` | `["daily","memory","sessions","logs","recall"]` | 数据源 |

### Deep 恢复配置

当长期记忆健康度低于某个阈值时，会触发恢复。

| 键名 | 类型 | 默认值 | 说明 |
| --------------------------------- | --------- | ------- | ------------------------------------------ |
| `recovery.enabled` | `boolean` | `true` | 启用自动恢复 |
| `recovery.triggerBelowHealth` | `number` | `0.35` | 触发恢复的健康度阈值 |
| `recovery.lookbackDays` | `number` | `30` | 回溯查找恢复材料的时间范围 |
| `recovery.maxRecoveredCandidates` | `number` | `20` | 每次运行最多恢复的候选项数 |
| `recovery.minRecoveryConfidence` | `number` | `0.9` | 恢复候选项所需的最低置信度 |
| `recovery.autoWriteMinConfidence` | `number` | `0.97` | 自动写入阈值（跳过人工审核） |

### REM 阶段配置

| 键名 | 类型 | 默认值 | 说明 |
| -------------------- | ---------- | --------------------------- | --------------------------------------- |
| `enabled` | `boolean` | `true` | 启用 REM 阶段 |
| `cron` | `string` | `0 5 * * 0` | 计划（默认：每周日凌晨 5 点） |
| `lookbackDays` | `number` | `7` | 要反思多少天的材料 |
| `limit` | `number` | `10` | 最多写入的模式或主题数 |
| `minPatternStrength` | `number` | `0.75` | 最低标签共现强度 |
| `sources` | `string[]` | `["memory","daily","deep"]` | 用于反思的数据源 |

### 执行覆盖设置

每个阶段都接受一个 `execution` 块，用于覆盖全局默认值：

| 键名 | 类型 | 默认值 | 说明 |
| ----------------- | -------- | ------------ | ------------------------------ |
| `speed` | `string` | `"balanced"` | `fast`、`balanced` 或 `slow` |
| `thinking` | `string` | `"medium"` | `low`、`medium` 或 `high` |
| `budget` | `string` | `"medium"` | `cheap`、`medium` 或 `expensive` |
| `model` | `string` | 未设置 | 为此阶段覆盖模型 |
| `maxOutputTokens` | `number` | 未设置 | 限制输出 token 数 |
| `temperature` | `number` | 未设置 | 采样温度（0-2） |
| `timeoutMs` | `number` | 未设置 | 阶段超时时间（毫秒） |

## 提升信号（Deep 阶段）

Deep Dreaming 会组合六个加权信号。要完成提升，所有已配置的阈值门槛都必须同时通过。

| 信号 | 权重 | 说明 |
| ------------------- | ------ | -------------------------------------------------- |
| 频率 | 0.24 | 同一条目被召回的频率 |
| 相关性 | 0.30 | 被检索时的平均召回分数 |
| 查询多样性 | 0.15 | 使其浮现出来的不同查询意图数量 |
| 近期性 | 0.15 | 时间衰减（`recencyHalfLifeDays`，默认 14） |
| 整合度 | 0.10 | 奖励跨多天重复出现的召回 |
| 概念丰富度 | 0.06 | 奖励具有更丰富派生概念标签的条目 |

## 聊天命令

```
/dreaming status                 # 显示阶段配置和运行频率
/dreaming on                     # 启用所有阶段
/dreaming off                    # 禁用所有阶段
/dreaming enable light|deep|rem  # 启用特定阶段
/dreaming disable light|deep|rem # 禁用特定阶段
/dreaming help                   # 显示用法指南
```

## CLI 命令

从命令行预览并应用深度提升：

```bash
# 预览提升候选项
openclaw memory promote

# 将提升内容应用到 MEMORY.md
openclaw memory promote --apply

# 限制预览数量
openclaw memory promote --limit 5

# 包含已提升条目
openclaw memory promote --include-promoted

# 检查 Dreaming 状态
openclaw memory status --deep
```

完整 flag 参考请参阅 [memory CLI](/cli/memory)。

## 工作原理

### Light 阶段流水线

1. 从 `memory/.dreams/short-term-recall.json` 读取短期召回条目。
2. 过滤出位于当前时间 `lookbackDays` 范围内的条目。
3. 按 Jaccard 相似度去重（阈值可配置）。
4. 按平均召回分数排序，最多取 `limit` 个条目。
5. 将暂存候选项写入每日笔记中的 `## Light Sleep` 区块。

### Deep 阶段流水线

1. 使用加权信号读取并排序短期召回候选项。
2. 应用阈值门槛：`minScore`、`minRecallCount`、`minUniqueQueries`。
3. 按 `maxAgeDays` 过滤，并应用近期衰减。
4. 在已配置的记忆工作区之间展开处理。
5. 写入前重新读取实时每日笔记（跳过过期或已删除的片段）。
6. 将符合条件的条目追加到 `MEMORY.md`，并附带提升时间戳。
7. 标记已提升条目，以便将来周期中排除它们。
8. 如果健康度低于 `recovery.triggerBelowHealth`，则运行恢复流程。

### REM 阶段流水线

1. 读取 `lookbackDays` 范围内最近的记忆痕迹。
2. 按共现关系对概念标签进行聚类。
3. 按 `minPatternStrength` 过滤模式。
4. 将主题和反思写入每日笔记中的 `## REM Sleep` 区块。

## 调度

每个阶段都会自动管理自己的 cron 作业。启用 Dreaming 后，`memory-core` 会在 Gateway 网关启动时协调受管 cron 作业。你不需要手动创建 cron 条目。

| 阶段 | 默认计划 | 说明 |
| ----- | ---------------- | ------------------- |
| Light | `0 */6 * * *` | 每 6 小时一次 |
| Deep  | `0 3 * * *` | 每天凌晨 3 点 |
| REM   | `0 5 * * 0` | 每周日凌晨 5 点 |

你可以使用阶段的 `cron` 键覆盖任何计划。所有计划都会遵循全局 `timezone` 设置。

## Dreams UI

启用 Dreaming 后，Gateway 网关侧边栏会显示一个 **Dreams** 标签页，其中包含记忆统计信息（短期数量、长期数量、已提升数量）以及下一次计划周期时间。每日计数在设置了 `dreaming.timezone` 时会遵循该配置，否则回退到已配置的用户时区。

手动运行 `openclaw memory promote` 默认使用相同的 Deep 阶段阈值，因此除非你传递 CLI 覆盖参数，否则计划运行与按需提升会保持一致。

## 相关内容

- [Memory](/zh-CN/concepts/memory)
- [Memory Search](/zh-CN/concepts/memory-search)
- [记忆配置参考](/zh-CN/reference/memory-config)
- [memory CLI](/cli/memory)
