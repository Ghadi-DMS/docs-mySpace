---
read_when:
    - 你需要检查原始模型输出中的推理泄漏
    - 你想在迭代时以监视模式运行 Gateway 网关
    - 你需要一个可重复的调试工作流
summary: 调试工具：监视模式、原始模型流，以及推理泄漏追踪
title: 调试
x-i18n:
    generated_at: "2026-04-05T22:30:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4bc72e8d6cad3a1acaad066f381c82309583fabf304c589e63885f2685dc704e
    source_path: help/debugging.md
    workflow: 15
---

# 调试

本页介绍了用于流式输出的调试辅助工具，尤其适用于提供商将推理内容混入普通文本的情况。

## 运行时调试覆盖

在聊天中使用 `/debug` 设置**仅运行时**的配置覆盖（保存在内存中，而非磁盘）。
`/debug` 默认禁用；使用 `commands.debug: true` 启用。
当你需要切换一些不常用设置，而又不想编辑 `openclaw.json` 时，这会很方便。

示例：

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug unset messages.responsePrefix
/debug reset
```

`/debug reset` 会清除所有覆盖，并恢复到磁盘上的配置。

## Gateway 网关监视模式

为了快速迭代，可在文件监视器下运行 Gateway 网关：

```bash
pnpm gateway:watch
```

它映射到：

```bash
node scripts/watch-node.mjs gateway --force
```

监视器会在 `src/` 下与构建相关的文件、扩展源码文件、扩展的 `package.json` 和 `openclaw.plugin.json` 元数据、`tsconfig.json`、`package.json` 以及 `tsdown.config.ts` 发生变化时重启。
扩展元数据变更会在不强制执行 `tsdown` 重建的情况下重启 Gateway 网关；源码和配置变更仍会先重建 `dist`。

在 `gateway:watch` 之后添加任何 Gateway 网关 CLI 标志，它们都会在每次重启时透传。
现在，对于同一仓库和相同标志组合，重复运行相同的监视命令会替换旧的监视器，而不会留下重复的监视器父进程。

## 开发配置文件 + 开发 Gateway 网关（`--dev`）

使用开发配置文件来隔离状态，并为调试启动一个安全、可随时丢弃的环境。这里有**两个** `--dev` 标志：

- **全局 `--dev`（配置文件）：** 将状态隔离到 `~/.openclaw-dev` 下，并将 Gateway 网关端口默认设为 `19001`（派生端口也会随之变化）。
- **`gateway --dev`：** 告诉 Gateway 网关在缺失时自动创建默认配置 + 工作区**（并跳过 BOOTSTRAP.md）**。

推荐流程（开发配置文件 + 开发引导）：

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

如果你还没有全局安装，请通过 `pnpm openclaw ...` 运行 CLI。

它会执行以下操作：

1. **配置文件隔离**（全局 `--dev`）
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001`（浏览器/canvas 端口也会相应变化）

2. **开发引导**（`gateway --dev`）
   - 如果缺失则写入最小化配置（`gateway.mode=local`，绑定到 loopback）。
   - 将 `agent.workspace` 设为开发工作区。
   - 设置 `agent.skipBootstrap=true`（不使用 BOOTSTRAP.md）。
   - 如果缺失，则为工作区填充以下文件：
     `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`。
   - 默认身份：**C3‑PO**（礼仪机器人）。
   - 在开发模式下跳过渠道提供商（`OPENCLAW_SKIP_CHANNELS=1`）。

重置流程（全新开始）：

```bash
pnpm gateway:dev:reset
```

注意：`--dev` 是一个**全局**配置文件标志，有些运行器会吞掉它。
如果你需要显式写出它，请使用环境变量形式：

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

`--reset` 会清除配置、凭证、会话以及开发工作区（使用 `trash`，而不是 `rm`），然后重新创建默认的开发环境。

提示：如果一个非开发模式的 Gateway 网关已经在运行（`launchd`/`systemd`），请先停止它：

```bash
openclaw gateway stop
```

## 原始流日志（OpenClaw）

OpenClaw 可以在任何过滤/格式化之前记录**原始 assistant 流**。
这是查看推理内容是否以纯文本增量形式到达（或作为独立 thinking 块到达）的最佳方式。

通过 CLI 启用：

```bash
pnpm gateway:watch --raw-stream
```

可选路径覆盖：

```bash
pnpm gateway:watch --raw-stream --raw-stream-path ~/.openclaw/logs/raw-stream.jsonl
```

等价的环境变量：

```bash
OPENCLAW_RAW_STREAM=1
OPENCLAW_RAW_STREAM_PATH=~/.openclaw/logs/raw-stream.jsonl
```

默认文件：

`~/.openclaw/logs/raw-stream.jsonl`

## 原始分块日志（pi-mono）

要在块被解析之前捕获**原始 OpenAI 兼容分块**，pi-mono 提供了一个单独的日志记录器：

```bash
PI_RAW_STREAM=1
```

可选路径：

```bash
PI_RAW_STREAM_PATH=~/.pi-mono/logs/raw-openai-completions.jsonl
```

默认文件：

`~/.pi-mono/logs/raw-openai-completions.jsonl`

> 注意：这仅由使用 pi-mono 的
> `openai-completions` 提供商的进程输出。

## 安全注意事项

- 原始流日志可能包含完整提示词、工具输出和用户数据。
- 请将日志保留在本地，并在调试后删除。
- 如果你要分享日志，请先清除密钥和个人身份信息（PII）。
