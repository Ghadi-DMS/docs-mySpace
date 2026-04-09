---
read_when:
    - 扩展 qa-lab 或 qa-channel 时
    - 添加仓库支持的 QA 场景时
    - 围绕 Gateway 网关仪表板构建更高拟真度的 QA 自动化时
summary: 用于 qa-lab、qa-channel、种子场景和协议报告的私有 QA 自动化形态
title: QA 端到端自动化
x-i18n:
    generated_at: "2026-04-09T00:06:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6ee5f55b98e46d1cd0b7ffdea96b0c8b5c1681803c69bdbcbe10dae14e62182b
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA 端到端自动化

私有 QA 栈旨在以比单个单元测试更贴近真实渠道形态的方式来验证 OpenClaw。

当前包含的部分：

- `extensions/qa-channel`：合成消息渠道，具有私信、channel、thread、reaction、edit 和 delete 界面。
- `extensions/qa-lab`：调试器 UI 和 QA 总线，用于观察对话记录、注入入站消息以及导出 Markdown 报告。
- `qa/`：为启动任务和基线 QA 场景提供仓库支持的种子资源。

当前的 QA 操作流程是一个双窗格 QA 站点：

- 左侧：带有智能体的 Gateway 网关仪表板（Control UI）。
- 右侧：QA Lab，显示类 Slack 的对话记录和场景计划。

使用以下命令运行：

```bash
pnpm qa:lab:up
```

这会构建 QA 站点，启动基于 Docker 的 Gateway 网关通道，并公开 QA Lab 页面，操作人员或自动化循环可以在此为智能体分配 QA 任务、观察真实的渠道行为，并记录哪些内容有效、哪些失败了，或哪些仍然受阻。

如果你想更快地迭代 QA Lab UI，而不必每次都重建 Docker 镜像，可通过绑定挂载的 QA Lab bundle 启动该栈：

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` 会让 Docker 服务基于预构建镜像运行，并将 `extensions/qa-lab/web/dist` 绑定挂载到 `qa-lab` 容器中。`qa:lab:watch` 会在变更时重建该 bundle，当 QA Lab 资源哈希发生变化时，浏览器会自动重新加载。

## 仓库支持的种子

种子资源位于 `qa/` 中：

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

这些内容有意保存在 git 中，以便人类和智能体都能看到 QA 计划。基线列表应足够广泛，以覆盖：

- 私信和 channel 聊天
- thread 行为
- 消息操作生命周期
- cron 回调
- memory recall
- 模型切换
- subagent 交接
- 读取仓库和读取文档
- 一个小型构建任务，例如 Lobster Invaders

## 报告

`qa-lab` 会根据观察到的总线时间线导出 Markdown 协议报告。
报告应回答：

- 哪些内容有效
- 哪些内容失败
- 哪些内容仍然受阻
- 哪些后续场景值得添加

对于角色与风格检查，可在多个实时模型引用上运行同一个场景，并编写一份经过评判的 Markdown 报告：

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model minimax/MiniMax-M2.7,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model qwen/qwen3.5-plus,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

该命令运行的是本地 QA Gateway 网关子进程，而不是 Docker。character eval 场景应通过 `SOUL.md` 设置 persona，然后运行普通用户轮次，例如聊天、工作区帮助和小型文件任务。不应告知候选模型它正在被评估。该命令会保留每份完整对话记录、记录基本运行统计信息，然后让评审模型以 fast 模式和 `xhigh` 推理来按自然度、氛围和幽默感对这些运行进行排序。
在比较提供商时，请使用 `--blind-judge-models`：评审提示仍会获得每份对话记录和运行状态，但候选引用会被替换为中性标签，例如 `candidate-01`；报告会在解析完成后将排序结果映射回真实引用。
候选运行默认使用 `high` thinking，而支持该级别的 OpenAI 模型则默认使用 `xhigh`。可通过 `--model provider/model,thinking=<level>` 内联覆盖特定候选项。`--thinking <level>` 仍会设置全局回退值，而较旧的 `--model-thinking <provider/model=level>` 形式则保留用于兼容。
OpenAI 候选引用默认使用 fast 模式，以便在提供商支持的情况下启用优先处理。当单个候选项或评审需要覆盖时，请内联添加 `,fast`、`,no-fast` 或 `,fast=false`。只有当你想为每个候选模型都强制启用 fast 模式时，才传递 `--fast`。候选和评审的持续时间都会记录在报告中，以便进行基准分析，但评审提示会明确说明不要按速度排序。
候选模型运行和评审模型运行默认都使用并发数 16。当提供商限制或本地 Gateway 网关压力使运行噪声过大时，请降低 `--concurrency` 或 `--judge-concurrency`。
当未传递候选 `--model` 时，character eval 默认使用
`openai/gpt-5.4`、`openai/gpt-5.2`、`anthropic/claude-opus-4-6`、
`anthropic/claude-sonnet-4-6`、`minimax/MiniMax-M2.7`、`zai/glm-5.1`、
`moonshot/kimi-k2.5`、`qwen/qwen3.5-plus` 和
`google/gemini-3.1-pro-preview`。
当未传递 `--judge-model` 时，评审默认使用
`openai/gpt-5.4,thinking=xhigh,fast` 和
`anthropic/claude-opus-4-6,thinking=high`。

## 相关文档

- [测试](/zh-CN/help/testing)
- [QA 渠道](/zh-CN/channels/qa-channel)
- [仪表板](/web/dashboard)
