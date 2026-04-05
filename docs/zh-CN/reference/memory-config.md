---
read_when:
    - 你想配置记忆搜索提供商或 embedding 模型
    - 你想设置 QMD 后端
    - 你想调优混合搜索、MMR 或时间衰减
    - 你想启用多模态记忆索引
summary: 记忆搜索、embedding 提供商、QMD、混合搜索和多模态索引的所有配置项
title: 记忆配置参考
x-i18n:
    generated_at: "2026-04-05T17:49:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 40ad19f65d8c688c9a4ce3275befe9455ed40073e0df0cee2f7f3fc4c0c363c4
    source_path: reference/memory-config.md
    workflow: 15
---

# 记忆配置参考

本页列出了 OpenClaw 记忆搜索的每一个配置项。有关概念性概览，请参见：

- [记忆概览](/zh-CN/concepts/memory) -- 记忆的工作方式
- [内置引擎](/zh-CN/concepts/memory-builtin) -- 默认的 SQLite 后端
- [QMD 引擎](/zh-CN/concepts/memory-qmd) -- 本地优先的 sidecar
- [记忆搜索](/zh-CN/concepts/memory-search) -- 搜索流水线和调优

除非另有说明，所有记忆搜索设置都位于 `openclaw.json` 的
`agents.defaults.memorySearch` 下。

---

## 提供商选择

| 键名       | 类型      | 默认值         | 说明                                                                     |
| ---------- | --------- | -------------- | ------------------------------------------------------------------------ |
| `provider` | `string`  | 自动检测       | Embedding 适配器 ID：`openai`、`gemini`、`voyage`、`mistral`、`ollama`、`local` |
| `model`    | `string`  | 提供商默认值   | Embedding 模型名称                                                       |
| `fallback` | `string`  | `"none"`       | 主适配器失败时使用的回退适配器 ID                                       |
| `enabled`  | `boolean` | `true`         | 启用或禁用记忆搜索                                                       |

### 自动检测顺序

当未设置 `provider` 时，OpenClaw 会选择第一个可用项：

1. `local` -- 如果配置了 `memorySearch.local.modelPath` 且文件存在。
2. `openai` -- 如果可以解析出 OpenAI 密钥。
3. `gemini` -- 如果可以解析出 Gemini 密钥。
4. `voyage` -- 如果可以解析出 Voyage 密钥。
5. `mistral` -- 如果可以解析出 Mistral 密钥。

支持 `ollama`，但不会自动检测（请显式设置）。

### API 密钥解析

远程 embedding 需要 API 密钥。OpenClaw 会从以下来源解析：
auth profiles、`models.providers.*.apiKey` 或环境变量。

| 提供商 | 环境变量                     | 配置键名                          |
| ------ | ---------------------------- | --------------------------------- |
| OpenAI | `OPENAI_API_KEY`             | `models.providers.openai.apiKey`  |
| Gemini | `GEMINI_API_KEY`             | `models.providers.google.apiKey`  |
| Voyage | `VOYAGE_API_KEY`             | `models.providers.voyage.apiKey`  |
| Mistral | `MISTRAL_API_KEY`           | `models.providers.mistral.apiKey` |
| Ollama | `OLLAMA_API_KEY`（占位符）  | --                                |

Codex OAuth 仅覆盖聊天/completions，不满足 embedding 请求。

---

## 远程端点配置

用于自定义 OpenAI 兼容端点或覆盖提供商默认值：

| 键名             | 类型     | 说明                              |
| ---------------- | -------- | --------------------------------- |
| `remote.baseUrl` | `string` | 自定义 API 基础 URL               |
| `remote.apiKey`  | `string` | 覆盖 API 密钥                     |
| `remote.headers` | `object` | 额外的 HTTP 标头（与提供商默认值合并） |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Gemini 专属配置

| 键名                   | 类型     | 默认值                 | 说明                                       |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | 也支持 `gemini-embedding-2-preview`        |
| `outputDimensionality` | `number` | `3072`                 | 对于 Embedding 2：768、1536 或 3072        |

<Warning>
更改模型或 `outputDimensionality` 会触发自动全量重建索引。
</Warning>

---

## 本地 embedding 配置

| 键名                  | 类型     | 默认值                 | 说明                         |
| --------------------- | -------- | ---------------------- | ---------------------------- |
| `local.modelPath`     | `string` | 自动下载               | GGUF 模型文件路径            |
| `local.modelCacheDir` | `string` | node-llama-cpp 默认值  | 已下载模型的缓存目录         |

默认模型：`embeddinggemma-300m-qat-Q8_0.gguf`（约 0.6 GB，自动下载）。
需要本机构建：`pnpm approve-builds` 然后 `pnpm rebuild node-llama-cpp`。

---

## 混合搜索配置

全部位于 `memorySearch.query.hybrid` 下：

| 键名                  | 类型      | 默认值  | 说明                           |
| --------------------- | --------- | ------- | ------------------------------ |
| `enabled`             | `boolean` | `true`  | 启用混合 BM25 + 向量搜索       |
| `vectorWeight`        | `number`  | `0.7`   | 向量分数权重（0-1）            |
| `textWeight`          | `number`  | `0.3`   | BM25 分数权重（0-1）           |
| `candidateMultiplier` | `number`  | `4`     | 候选池大小倍数                 |

### MMR（多样性）

| 键名          | 类型      | 默认值  | 说明                               |
| ------------- | --------- | ------- | ---------------------------------- |
| `mmr.enabled` | `boolean` | `false` | 启用 MMR 重排序                    |
| `mmr.lambda`  | `number`  | `0.7`   | 0 = 最大多样性，1 = 最大相关性     |

### 时间衰减（新近性）

| 键名                         | 类型      | 默认值  | 说明                     |
| ---------------------------- | --------- | ------- | ------------------------ |
| `temporalDecay.enabled`      | `boolean` | `false` | 启用新近性加权           |
| `temporalDecay.halfLifeDays` | `number`  | `30`    | 分数每 N 天减半          |

常青文件（`MEMORY.md`、`memory/` 中未带日期的文件）永不衰减。

### 完整示例

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## 额外记忆路径

| 键名         | 类型       | 说明                           |
| ------------ | ---------- | ------------------------------ |
| `extraPaths` | `string[]` | 要索引的额外目录或文件         |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

路径可以是绝对路径，也可以是相对于工作区的路径。目录会递归扫描
`.md` 文件。符号链接处理取决于当前后端：
内置引擎会忽略符号链接，而 QMD 会遵循底层 QMD 扫描器的行为。

对于按智能体范围的跨智能体 transcript 搜索，请使用
`agents.list[].memorySearch.qmd.extraCollections`，而不是 `memory.qmd.paths`。
这些额外集合遵循相同的 `{ path, name, pattern? }` 结构，但会按智能体合并，并且当路径指向当前工作区之外时，可以保留显式共享名称。
如果同一个解析后的路径同时出现在 `memory.qmd.paths` 和
`memorySearch.qmd.extraCollections` 中，QMD 会保留第一个条目并跳过重复项。

---

## 多模态记忆（Gemini）

使用 Gemini Embedding 2，在 Markdown 之外同时索引图像和音频：

| 键名                      | 类型       | 默认值     | 说明                                 |
| ------------------------- | ---------- | ---------- | ------------------------------------ |
| `multimodal.enabled`      | `boolean`  | `false`    | 启用多模态索引                       |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`、`["audio"]` 或 `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10000000` | 可索引的最大文件大小                 |

仅适用于 `extraPaths` 中的文件。默认记忆根目录仍然只支持 Markdown。
需要 `gemini-embedding-2-preview`。`fallback` 必须为 `"none"`。

支持的格式：`.jpg`、`.jpeg`、`.png`、`.webp`、`.gif`、`.heic`、`.heif`
（图像）；`.mp3`、`.wav`、`.ogg`、`.opus`、`.m4a`、`.aac`、`.flac`（音频）。

---

## Embedding 缓存

| 键名               | 类型      | 默认值  | 说明                             |
| ------------------ | --------- | ------- | -------------------------------- |
| `cache.enabled`    | `boolean` | `false` | 在 SQLite 中缓存分块 embedding   |
| `cache.maxEntries` | `number`  | `50000` | 最大缓存 embedding 数量          |

防止在重建索引或 transcript 更新期间，对未变化的文本重复做 embedding。

---

## 批量索引

| 键名                          | 类型      | 默认值  | 说明                   |
| ----------------------------- | --------- | ------- | ---------------------- |
| `remote.batch.enabled`        | `boolean` | `false` | 启用批量 embedding API |
| `remote.batch.concurrency`    | `number`  | `2`     | 并行批任务数           |
| `remote.batch.wait`           | `boolean` | `true`  | 等待批处理完成         |
| `remote.batch.pollIntervalMs` | `number`  | --      | 轮询间隔               |
| `remote.batch.timeoutMinutes` | `number`  | --      | 批处理超时时间         |

适用于 `openai`、`gemini` 和 `voyage`。对于大型回填任务，OpenAI 批处理通常
最快且最便宜。

---

## 会话记忆搜索（实验性）

对会话 transcript 建立索引，并通过 `memory_search` 暴露：

| 键名                          | 类型       | 默认值       | 说明                                  |
| ----------------------------- | ---------- | ------------ | ------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`      | 启用会话索引                          |
| `sources`                     | `string[]` | `["memory"]` | 添加 `"sessions"` 以包含 transcript   |
| `sync.sessions.deltaBytes`    | `number`   | `100000`     | 触发重建索引的字节阈值                |
| `sync.sessions.deltaMessages` | `number`   | `50`         | 触发重建索引的消息阈值                |

会话索引需要显式启用，并且异步运行。结果可能会略微过时。
会话日志保存在磁盘上，因此应将文件系统访问视为信任边界。

---

## SQLite 向量加速（sqlite-vec）

| 键名                         | 类型      | 默认值   | 说明                           |
| ---------------------------- | --------- | -------- | ------------------------------ |
| `store.vector.enabled`       | `boolean` | `true`   | 使用 sqlite-vec 执行向量查询   |
| `store.vector.extensionPath` | `string`  | 内置     | 覆盖 sqlite-vec 路径           |

当 sqlite-vec 不可用时，OpenClaw 会自动回退到进程内余弦相似度。

---

## 索引存储

| 键名                  | 类型     | 默认值                                | 说明                                 |
| --------------------- | -------- | ------------------------------------- | ------------------------------------ |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | 索引位置（支持 `{agentId}` token）   |
| `store.fts.tokenizer` | `string` | `unicode61`                           | FTS5 tokenizer（`unicode61` 或 `trigram`） |

---

## QMD 后端配置

设置 `memory.backend = "qmd"` 以启用。所有 QMD 设置都位于
`memory.qmd` 下：

| 键名                     | 类型      | 默认值   | 说明                                         |
| ------------------------ | --------- | -------- | -------------------------------------------- |
| `command`                | `string`  | `qmd`    | QMD 可执行文件路径                           |
| `searchMode`             | `string`  | `search` | 搜索命令：`search`、`vsearch`、`query`       |
| `includeDefaultMemory`   | `boolean` | `true`   | 自动索引 `MEMORY.md` + `memory/**/*.md`      |
| `paths[]`                | `array`   | --       | 额外路径：`{ name, path, pattern? }`         |
| `sessions.enabled`       | `boolean` | `false`  | 索引会话 transcript                          |
| `sessions.retentionDays` | `number`  | --       | Transcript 保留天数                          |
| `sessions.exportDir`     | `string`  | --       | 导出目录                                     |

### 更新计划

| 键名                      | 类型      | 默认值   | 说明                           |
| ------------------------- | --------- | -------- | ------------------------------ |
| `update.interval`         | `string`  | `5m`     | 刷新间隔                       |
| `update.debounceMs`       | `number`  | `15000`  | 文件变更防抖                   |
| `update.onBoot`           | `boolean` | `true`   | 启动时刷新                     |
| `update.waitForBootSync`  | `boolean` | `false`  | 阻塞启动直到刷新完成           |
| `update.embedInterval`    | `string`  | --       | 单独的 embedding 节奏          |
| `update.commandTimeoutMs` | `number`  | --       | QMD 命令超时时间               |
| `update.updateTimeoutMs`  | `number`  | --       | QMD 更新操作超时时间           |
| `update.embedTimeoutMs`   | `number`  | --       | QMD embedding 操作超时时间     |

### 限制

| 键名                      | 类型     | 默认值  | 说明                     |
| ------------------------- | -------- | ------- | ------------------------ |
| `limits.maxResults`       | `number` | `6`     | 最大搜索结果数           |
| `limits.maxSnippetChars`  | `number` | --      | 限制片段长度             |
| `limits.maxInjectedChars` | `number` | --      | 限制总注入字符数         |
| `limits.timeoutMs`        | `number` | `4000`  | 搜索超时时间             |

### 范围

控制哪些会话可以接收 QMD 搜索结果。Schema 与
[`session.sendPolicy`](/zh-CN/gateway/configuration-reference#session) 相同：

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

默认仅限私信。`match.keyPrefix` 匹配规范化后的会话键；
`match.rawKeyPrefix` 匹配包含 `agent:<id>:` 的原始键。

### 引用

`memory.citations` 适用于所有后端：

| 值               | 行为                                           |
| ---------------- | ---------------------------------------------- |
| `auto`（默认）   | 在片段中包含 `Source: <path#line>` 页脚         |
| `on`             | 始终包含页脚                                   |
| `off`            | 省略页脚（路径仍会在内部传递给智能体）         |

### 完整 QMD 示例

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming（实验性）

Dreaming 配置位于 `plugins.entries.memory-core.config.dreaming` 下，
而不是 `agents.defaults.memorySearch` 下。Dreaming 使用三个协作阶段
（light、deep、REM），每个阶段都有自己的计划和配置。有关概念细节和聊天命令，请参阅 [Dreaming](/concepts/dreaming)。

### 全局设置

| 键名                      | 类型      | 默认值       | 说明                                         |
| ------------------------- | --------- | ------------ | -------------------------------------------- |
| `enabled`                 | `boolean` | `true`       | 所有阶段的总开关                             |
| `timezone`                | `string`  | 未设置       | 用于计划评估和每日笔记的时区                 |
| `verboseLogging`          | `boolean` | `false`      | 输出详细的每次运行 Dreaming 日志             |
| `storage.mode`            | `string`  | `"inline"`   | `inline`、`separate` 或 `both`               |
| `storage.separateReports` | `boolean` | `false`      | 为每个阶段写入单独的报告文件                 |

### Light 阶段（`phases.light`）

扫描最近的 traces、去重，并将候选内容暂存到每日笔记中。
**不会**写入 `MEMORY.md`。

| 键名               | 类型       | 默认值                          | 说明                      |
| ------------------ | ---------- | ------------------------------- | ------------------------- |
| `enabled`          | `boolean`  | `true`                          | 启用 light 阶段           |
| `cron`             | `string`   | `0 */6 * * *`                   | 计划（每 6 小时一次）     |
| `lookbackDays`     | `number`   | `2`                             | 要扫描的 trace 天数       |
| `limit`            | `number`   | `100`                           | 最大暂存候选数            |
| `dedupeSimilarity` | `number`   | `0.9`                           | 去重的 Jaccard 阈值       |
| `sources`          | `string[]` | `["daily","sessions","recall"]` | 数据源                    |

### Deep 阶段（`phases.deep`）

将符合条件的候选内容提升到 `MEMORY.md` 中。它是**唯一**会写入持久事实的阶段。也负责在记忆稀薄时进行恢复。

| 键名                  | 类型       | 默认值                                          | 说明                           |
| --------------------- | ---------- | ----------------------------------------------- | ------------------------------ |
| `enabled`             | `boolean`  | `true`                                          | 启用 deep 阶段                 |
| `cron`                | `string`   | `0 3 * * *`                                     | 计划（每天凌晨 3 点）          |
| `limit`               | `number`   | `10`                                            | 每轮最大提升候选数             |
| `minScore`            | `number`   | `0.8`                                           | 提升所需的最小加权分数         |
| `minRecallCount`      | `number`   | `3`                                             | 最小 recall 次数阈值           |
| `minUniqueQueries`    | `number`   | `3`                                             | 最小不同查询数                 |
| `recencyHalfLifeDays` | `number`   | `14`                                            | 新近性分数减半所需天数         |
| `maxAgeDays`          | `number`   | `30`                                            | 可提升的每日笔记最大年龄       |
| `sources`             | `string[]` | `["daily","memory","sessions","logs","recall"]` | 数据源                         |

#### Deep 恢复（`phases.deep.recovery`）

| 键名                     | 类型      | 默认值  | 说明                                  |
| ------------------------ | --------- | ------- | ------------------------------------- |
| `enabled`                | `boolean` | `true`  | 启用自动恢复                          |
| `triggerBelowHealth`     | `number`  | `0.35`  | 触发恢复的健康分数阈值                |
| `lookbackDays`           | `number`  | `30`    | 查找恢复材料的回溯天数                |
| `maxRecoveredCandidates` | `number`  | `20`    | 每次运行最大恢复候选数                |
| `minRecoveryConfidence`  | `number`  | `0.9`   | 恢复候选的最小置信度                  |
| `autoWriteMinConfidence` | `number`  | `0.97`  | 自动写入阈值（跳过人工审核）          |

### REM 阶段（`phases.rem`）

将主题、反思和模式笔记写入每日笔记。
**不会**写入 `MEMORY.md`。

| 键名                 | 类型       | 默认值                      | 说明                       |
| -------------------- | ---------- | --------------------------- | -------------------------- |
| `enabled`            | `boolean`  | `true`                      | 启用 REM 阶段              |
| `cron`               | `string`   | `0 5 * * 0`                 | 计划（每周日凌晨 5 点）    |
| `lookbackDays`       | `number`   | `7`                         | 用于反思的材料天数         |
| `limit`              | `number`   | `10`                        | 最多写入的模式或主题数     |
| `minPatternStrength` | `number`   | `0.75`                      | 最小标签共现强度           |
| `sources`            | `string[]` | `["memory","daily","deep"]` | 用于反思的数据源           |

### 执行覆盖

每个阶段都接受一个 `execution` 块。还存在一个全局的
`execution.defaults` 块，供各阶段继承。

| 键名              | 类型     | 默认值        | 说明                         |
| ----------------- | -------- | ------------- | ---------------------------- |
| `speed`           | `string` | `"balanced"`  | `fast`、`balanced` 或 `slow` |
| `thinking`        | `string` | `"medium"`    | `low`、`medium` 或 `high`    |
| `budget`          | `string` | `"medium"`    | `cheap`、`medium` 或 `expensive` |
| `model`           | `string` | 未设置        | 覆盖此阶段的模型             |
| `maxOutputTokens` | `number` | 未设置        | 限制输出 token 数            |
| `temperature`     | `number` | 未设置        | 采样温度（0-2）              |
| `timeoutMs`       | `number` | 未设置        | 阶段超时时间（毫秒）         |

### 示例

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            timezone: "America/New_York",
            phases: {
              light: { cron: "0 */4 * * *", lookbackDays: 3 },
              deep: { minScore: 0.85, recencyHalfLifeDays: 21 },
              rem: { lookbackDays: 14 },
            },
          },
        },
      },
    },
  },
}
```
