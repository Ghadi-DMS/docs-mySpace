---
read_when:
    - 扩展 qa-lab 或 qa-channel
    - 添加由仓库支持的 QA 场景
    - 围绕 Gateway 网关仪表板构建更高真实性的 QA 自动化
summary: qa-lab、qa-channel、种子场景和协议报告的私有 QA 自动化形态
title: QA 端到端自动化
x-i18n:
    generated_at: "2026-04-13T02:43:02Z"
    model: gpt-5.4
    provider: openai
    source_hash: a4a4f5c765163565c95c2a071f201775fd9d8d60cad4ff25d71e4710559c1570
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA 端到端自动化

私有 QA 栈旨在以比单个单元测试更贴近真实、更加符合渠道形态的方式来演练 OpenClaw。

当前组成部分：

- `extensions/qa-channel`：合成消息渠道，具备私信、渠道、线程、表情回应、编辑和删除等交互面。
- `extensions/qa-lab`：调试器 UI 和 QA 总线，用于观察对话记录、注入入站消息，以及导出 Markdown 报告。
- `qa/`：由仓库支持的种子资源，用于启动任务和基础 QA 场景。

当前的 QA 操作流程是一个双窗格 QA 站点：

- 左侧：带有智能体的 Gateway 网关仪表板（Control UI）。
- 右侧：QA Lab，显示类似 Slack 的对话记录和场景计划。

使用以下命令运行：

```bash
pnpm qa:lab:up
```

该命令会构建 QA 站点、启动基于 Docker 的 Gateway 网关测试通道，并暴露 QA Lab 页面，供操作员或自动化循环向智能体下达 QA 任务、观察真实渠道行为，并记录哪些内容成功、失败或仍然受阻。

为了更快地迭代 QA Lab UI，而不是每次都重新构建 Docker 镜像，可以使用绑定挂载的 QA Lab bundle 启动整个栈：

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` 会让 Docker 服务保持在一个预构建镜像上运行，并将 `extensions/qa-lab/web/dist` 绑定挂载到 `qa-lab` 容器中。`qa:lab:watch` 会在发生变更时重新构建该 bundle，当 QA Lab 资源哈希变化时，浏览器会自动重新加载。

对于一个传输协议真实的 Matrix 冒烟测试通道，请运行：

```bash
pnpm openclaw qa matrix
```

该通道会在 Docker 中配置一个一次性的 Tuwunel homeserver，注册临时的驱动、SUT 和观察者用户，创建一个私有房间，然后在 QA Gateway 网关子进程中运行真实的 Matrix 插件。实时传输协议通道会将子配置限制在正在测试的传输协议范围内，因此 Matrix 会在子配置中不包含 `qa-channel` 的情况下运行。

对于一个传输协议真实的 Telegram 冒烟测试通道，请运行：

```bash
pnpm openclaw qa telegram
```

该通道会针对一个真实的私有 Telegram 群组，而不是配置一个一次性服务器。它需要 `OPENCLAW_QA_TELEGRAM_GROUP_ID`、`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` 和 `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`，并且要求两个不同的机器人位于同一个私有群组中。SUT 机器人必须有一个 Telegram 用户名，并且当两个机器人都在 `@BotFather` 中启用了 “Bot-to-Bot Communication Mode” 时，机器人对机器人的观察效果最佳。

实时传输协议通道现在共享一个更小的统一契约，而不是各自定义自己的场景列表结构：

`qa-channel` 仍然是宽泛的合成产品行为测试套件，不属于实时传输协议覆盖矩阵的一部分。

| 通道     | Canary | 提及门控 | Allowlist 拦截 | 顶层回复 | 重启恢复 | 线程后续跟进 | 线程隔离 | 表情回应观察 | 帮助命令 |
| -------- | ------ | -------- | -------------- | -------- | -------- | ------------ | -------- | ------------ | -------- |
| Matrix   | x      | x        | x              | x        | x        | x            | x        | x            |          |
| Telegram | x      |          |                |          |          |              |          |              | x        |

这样可以让 `qa-channel` 保持为宽泛的产品行为测试套件，而 Matrix、Telegram 和未来的实时传输协议则共享一份明确的传输协议契约检查清单。

对于一个无需将 Docker 引入 QA 路径的一次性 Linux VM 通道，请运行：

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

这会启动一个全新的 Multipass 来宾机，在来宾机内安装依赖、构建 OpenClaw、运行 `qa suite`，然后将常规 QA 报告和摘要复制回宿主机的 `.artifacts/qa-e2e/...` 中。
它复用了与宿主机上 `qa suite` 相同的场景选择行为。
宿主机和 Multipass 的 suite 运行默认都会以隔离的 Gateway 网关工作进程并行执行多个被选中的场景，最多可达 64 个工作进程或所选场景数量。使用 `--concurrency <count>` 调整工作进程数量，或使用 `--concurrency 1` 串行执行。
实时运行会转发适合来宾机使用的、受支持的 QA 认证输入：基于环境变量的提供商密钥、QA 实时提供商配置路径，以及在存在时的 `CODEX_HOME`。请将 `--output-dir` 保持在仓库根目录下，以便来宾机可以通过挂载的工作区回写结果。

## 由仓库支持的种子资源

种子资源位于 `qa/`：

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

这些内容有意保存在 git 中，以便 QA 计划对人类和智能体都可见。

`qa-lab` 应保持为一个通用的 Markdown 运行器。每个场景 Markdown 文件都是一次测试运行的事实来源，应定义以下内容：

- 场景元数据
- 文档和代码引用
- 可选插件要求
- 可选 Gateway 网关配置补丁
- 可执行的 `qa-flow`

支撑 `qa-flow` 的可复用运行时表面允许保持通用且跨领域。例如，Markdown 场景可以将传输协议侧辅助工具与浏览器侧辅助工具结合起来，通过 Gateway 网关 `browser.request` 接缝驱动嵌入式 Control UI，而无需添加特定场景的专用运行器。

基础列表应保持足够宽泛，以覆盖：

- 私信和渠道聊天
- 线程行为
- 消息动作生命周期
- cron 回调
- 记忆召回
- 模型切换
- 子智能体移交
- 读取仓库和读取文档
- 一个小型构建任务，例如 Lobster Invaders

## 传输协议适配器

`qa-lab` 拥有面向 Markdown QA 场景的通用传输协议接缝。
`qa-channel` 是该接缝上的第一个适配器，但设计目标更广：
未来无论是真实还是合成渠道，都应接入同一个 suite 运行器，而不是新增一个特定于传输协议的 QA 运行器。

在架构层面，划分如下：

- `qa-lab` 负责通用场景执行、工作进程并发、制品写入和报告。
- 传输协议适配器负责 Gateway 网关配置、就绪状态、入站和出站观察、传输协议动作，以及规范化的传输协议状态。
- `qa/scenarios/` 下的 Markdown 场景文件定义测试运行；`qa-lab` 提供执行它们的可复用运行时表面。

面向维护者的新渠道适配器接入指南见
[测试](/zh-CN/help/testing#adding-a-channel-to-qa)。

## 报告

`qa-lab` 会根据观察到的总线时间线导出一份 Markdown 协议报告。
该报告应回答：

- 哪些有效
- 哪些失败
- 哪些仍然受阻
- 值得添加哪些后续场景

对于角色和风格检查，请在多个实时模型引用上运行相同场景，并写出一份经过评审的 Markdown 报告：

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

该命令运行的是本地 QA Gateway 网关子进程，而不是 Docker。角色评估场景应通过 `SOUL.md` 设置 persona，然后运行普通用户轮次，例如聊天、工作区帮助和小型文件任务。候选模型不应被告知自己正在被评估。该命令会保留每份完整对话记录、记录基本运行统计信息，然后以快速模式请求评审模型使用 `xhigh` 推理，根据自然度、氛围和幽默感对运行结果进行排序。
在比较不同提供商时，请使用 `--blind-judge-models`：评审提示仍会获得每份对话记录和运行状态，但候选引用会被替换为中性标签，例如 `candidate-01`；报告会在解析后将排名映射回真实引用。
候选运行默认使用 `high` thinking，而支持的 OpenAI 模型则使用 `xhigh`。如果要覆盖某个特定候选项，请内联使用
`--model provider/model,thinking=<level>`。`--thinking <level>` 仍然会设置全局回退值，较早的 `--model-thinking <provider/model=level>` 形式也会为兼容性保留。
OpenAI 候选引用默认使用快速模式，这样在提供商支持时会启用优先处理。若单个候选项或评审项需要覆盖，请内联添加 `,fast`、`,no-fast` 或 `,fast=false`。只有当你希望为每个候选模型强制开启快速模式时，才传入 `--fast`。报告中会记录候选和评审的持续时间以供基准分析，但评审提示会明确说明不要按速度进行排序。
候选和评审模型运行默认都使用 16 的并发度。当提供商限制或本地 Gateway 网关压力导致运行噪声过大时，可降低 `--concurrency` 或 `--judge-concurrency`。
当未传入候选 `--model` 时，角色评估默认使用
`openai/gpt-5.4`、`openai/gpt-5.2`、`openai/gpt-5`、`anthropic/claude-opus-4-6`、
`anthropic/claude-sonnet-4-6`、`zai/glm-5.1`、
`moonshot/kimi-k2.5` 和
`google/gemini-3.1-pro-preview`。
当未传入 `--judge-model` 时，评审默认使用
`openai/gpt-5.4,thinking=xhigh,fast` 和
`anthropic/claude-opus-4-6,thinking=high`。

## 相关文档

- [测试](/zh-CN/help/testing)
- [QA Channel](/zh-CN/channels/qa-channel)
- [仪表板](/web/dashboard)
