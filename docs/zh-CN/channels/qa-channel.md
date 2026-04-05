---
read_when:
    - 你正在将合成 QA 传输接入本地或 CI 测试运行
    - 你需要内置的 qa-channel 配置界面
    - 你正在迭代端到端 QA 自动化
summary: 用于确定性 OpenClaw QA 场景的合成 Slack 类渠道插件
title: QA 渠道
x-i18n:
    generated_at: "2026-04-05T22:29:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b88cd73df2f61b34ad1eb83c3450f8fe15a51ac69fbb5a9eca0097564d67a06
    source_path: channels/qa-channel.md
    workflow: 15
---

# QA 渠道

`qa-channel` 是一个内置的合成消息传输，用于自动化 OpenClaw QA。

它不是生产渠道。它的存在是为了在保持状态确定且可完全检查的同时，演练真实传输所使用的相同渠道插件边界。

## 当前功能

- Slack 类目标语法：
  - `dm:<user>`
  - `channel:<room>`
  - `thread:<room>/<thread>`
- 基于 HTTP 的合成总线，用于：
  - 注入入站消息
  - 捕获出站转录
  - 创建线程
  - 反应
  - 编辑
  - 删除
  - 搜索和读取操作
- 内置的主机端自检运行器，可写入 Markdown 报告

## 配置

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

支持的账户键名：

- `baseUrl`
- `botUserId`
- `botDisplayName`
- `pollTimeoutMs`
- `allowFrom`
- `defaultTo`
- `actions.messages`
- `actions.reactions`
- `actions.search`
- `actions.threads`

## 运行器

当前的垂直切片：

```bash
pnpm qa:e2e
```

现在它会通过内置的 `qa-lab` 扩展进行路由。它会启动仓库内的 QA 总线，启动内置的 `qa-channel` 运行时切片，运行确定性的自检，并将 Markdown 报告写入 `.artifacts/qa-e2e/`。

私有调试器 UI：

```bash
pnpm qa:lab:build
pnpm openclaw qa ui
```

完整的仓库支持 QA 套件：

```bash
pnpm openclaw qa suite
```

这会在本地 URL 启动私有 QA 调试器，与随附的 Control UI bundle 分开。

## 范围

当前范围有意保持狭窄：

- 总线 + 插件传输
- 线程路由语法
- 渠道自有消息操作
- Markdown 报告

后续工作将添加：

- Docker 化的 OpenClaw 编排
- 提供商/模型矩阵执行
- 更丰富的场景发现
- 后续引入 OpenClaw 原生编排
