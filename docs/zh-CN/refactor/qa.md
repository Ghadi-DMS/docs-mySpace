---
x-i18n:
    generated_at: "2026-04-17T15:05:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: dbb2c70c82da7f6f12d90e25666635ff4147c52e8a94135e902d1de4f5cbccca
    source_path: refactor/qa.md
    workflow: 15
---

# QA 重构

状态：基础迁移已落地。

## 目标

将 OpenClaw QA 从拆分定义模型迁移为单一事实来源：

- 场景元数据
- 发送给模型的提示词
- 设置与清理
- harness 逻辑
- 断言与成功标准
- 产物与报告提示

期望的最终状态是：一个通用 QA harness 加载功能强大的场景定义文件，而不是将大部分行为硬编码在 TypeScript 中。

## 当前状态

当前的主要事实来源现在位于 `qa/scenarios/index.md`，以及每个场景对应的单独文件 `qa/scenarios/<theme>/*.md`。

已实现：

- `qa/scenarios/index.md`
  - 规范的 QA 包元数据
  - 操作员身份
  - 启动任务
- `qa/scenarios/<theme>/*.md`
  - 每个场景一个 Markdown 文件
  - 场景元数据
  - handler 绑定
  - 场景专用执行配置
- `extensions/qa-lab/src/scenario-catalog.ts`
  - Markdown 包解析器 + zod 校验
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - 从 Markdown 包渲染执行计划
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - 生成兼容性文件种子，以及 `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - 通过 Markdown 定义的 handler 绑定选择可执行场景
- QA 总线协议 + UI
  - 用于图片 / 视频 / 音频 / 文件渲染的通用内联附件

仍然拆分的部分：

- `extensions/qa-lab/src/suite.ts`
  - 仍然承载大多数可执行自定义 handler 逻辑
- `extensions/qa-lab/src/report.ts`
  - 仍然根据运行时输出推导报告结构

因此，事实来源拆分问题已经修复，但执行层仍然主要依赖 handler，而不是完全声明式。

## 实际场景面是什么样的

阅读当前 suite 可以看到几类不同的场景。

### 简单交互

- 渠道基线
- 私信基线
- 线程后续跟进
- 模型切换
- 审批后续执行
- reaction / edit / delete

### 配置与运行时变更

- config patch skill disable
- config apply restart wake-up
- config restart capability flip
- runtime inventory drift check

### 文件系统与仓库断言

- source / docs discovery report
- build Lobster Invaders
- generated image artifact lookup

### Memory 编排

- memory recall
- 渠道上下文中的 memory 工具
- memory failure fallback
- session memory ranking
- 线程 memory 隔离
- memory dreaming sweep

### 工具与插件集成

- MCP plugin-tools call
- Skills 可见性
- Skills 热安装
- 原生图像生成
- 图像往返
- 从附件理解图像

### 多轮与多参与者

- subagent handoff
- subagent fanout synthesis
- restart recovery 风格流程

这些分类很重要，因为它们决定 DSL 的需求。仅靠“提示词 + 预期文本”的平面列表是不够的。

## 方向

### 单一事实来源

使用 `qa/scenarios/index.md` 和 `qa/scenarios/<theme>/*.md` 作为编写时的单一事实来源。

这个包应保持：

- 便于人工在评审中阅读
- 可被机器解析
- 足够丰富，可驱动：
  - suite 执行
  - QA 工作区 bootstrap
  - QA Lab UI 元数据
  - docs / discovery 提示词
  - 报告生成

### 首选编写格式

使用 Markdown 作为顶层格式，并在其中使用结构化 YAML。

推荐结构：

- YAML frontmatter
  - id
  - title
  - surface
  - tags
  - docs refs
  - code refs
  - model / provider overrides
  - prerequisites
- prose sections
  - objective
  - notes
  - debugging hints
- fenced YAML blocks
  - setup
  - steps
  - assertions
  - cleanup

这样可以获得：

- 比庞大的 JSON 更好的 PR 可读性
- 比纯 YAML 更丰富的上下文
- 严格解析与 zod 校验

原始 JSON 仅应作为中间生成形式接受。

## 提议的场景文件结构

示例：

````md
---
id: image-generation-roundtrip
title: Image generation roundtrip
surface: image
tags: [media, image, roundtrip]
models:
  primary: openai/gpt-5.4
requires:
  tools: [image_generate]
  plugins: [openai, qa-channel]
docsRefs:
  - docs/help/testing.md
  - docs/concepts/model-providers.md
codeRefs:
  - extensions/qa-lab/src/suite.ts
  - src/gateway/chat-attachments.ts
---

# Objective

Verify generated media is reattached on the follow-up turn.

# Setup

```yaml scenario.setup
- action: config.patch
  patch:
    agents:
      defaults:
        imageGenerationModel:
          primary: openai/gpt-image-1
- action: session.create
  key: agent:qa:image-roundtrip
```

# Steps

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Image generation check: generate a QA lighthouse image and summarize it in one short sentence.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Image generation check
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Roundtrip image inspection check: describe the generated lighthouse attachment in one short sentence.
  attachments:
    - fromArtifact: lighthouseImage
```

# Expect

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Roundtrip image inspection check
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## DSL 必须覆盖的运行器能力

根据当前 suite，通用运行器需要的不只是提示词执行。

### 环境与设置动作

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### 智能体轮次动作

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### 配置与运行时动作

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### 文件与产物动作

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### Memory 与 cron 动作

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### MCP 动作

- `mcp.callTool`

### 断言

- `outbound.textIncludes`
- `outbound.inThread`
- `outbound.notInRoot`
- `tool.called`
- `tool.notPresent`
- `skill.visible`
- `skill.disabled`
- `file.contains`
- `memory.contains`
- `requestLog.matches`
- `sessionStore.matches`
- `cron.managedPresent`
- `artifact.exists`

## 变量与产物引用

DSL 必须支持保存输出并在后续引用。

当前 suite 中的示例：

- 创建一个线程，然后复用 `threadId`
- 创建一个 session，然后复用 `sessionKey`
- 生成一张图片，然后在下一轮中附加该文件
- 生成一个 wake marker 字符串，然后断言它稍后出现

所需能力：

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- 针对路径、session key、线程 id、marker、工具输出的类型化引用

如果没有变量支持，harness 逻辑就会继续泄漏回 TypeScript 中。

## 哪些内容应保留为逃生口

在第 1 阶段，完全纯声明式的运行器并不现实。

有些场景天然就是重编排型的：

- memory dreaming sweep
- config apply restart wake-up
- config restart capability flip
- 按时间戳 / 路径解析生成图像产物
- discovery-report evaluation

这些目前应继续使用显式自定义 handler。

推荐规则：

- 85-90% 声明式
- 对剩余困难部分使用显式 `customHandler` steps
- 仅允许具名且有文档说明的自定义 handler
- 不允许在场景文件中写匿名内联代码

这样可以让通用引擎保持整洁，同时仍然推进迁移。

## 架构变更

### 当前

场景 Markdown 现在已经是以下内容的事实来源：

- suite 执行
- 工作区 bootstrap 文件
- QA Lab UI 场景目录
- 报告元数据
- discovery 提示词

生成的兼容性内容：

- seed 工作区仍然包含 `QA_KICKOFF_TASK.md`
- seed 工作区仍然包含 `QA_SCENARIO_PLAN.md`
- seed 工作区现在还包含 `QA_SCENARIOS.md`

## 重构计划

### 第 1 阶段：加载器与 schema

已完成。

- 添加了 `qa/scenarios/index.md`
- 将场景拆分到 `qa/scenarios/<theme>/*.md`
- 为具名 Markdown YAML 包内容添加了解析器
- 使用 zod 进行校验
- 将消费者切换到已解析的包
- 删除了仓库级 `qa/seed-scenarios.json` 和 `qa/QA_KICKOFF_TASK.md`

### 第 2 阶段：通用引擎

- 将 `extensions/qa-lab/src/suite.ts` 拆分为：
  - loader
  - engine
  - action registry
  - assertion registry
  - custom handlers
- 保留现有 helper functions 作为引擎操作

交付物：

- 引擎执行简单的声明式场景

先从主要是 prompt + wait + assert 的场景开始：

- threaded follow-up
- 从附件理解图像
- Skills 可见性与调用
- channel baseline

交付物：

- 第一批真正由 Markdown 定义并通过通用引擎交付的场景

### 第 4 阶段：迁移中等复杂度场景

- image generation roundtrip
- 渠道上下文中的 memory 工具
- session memory ranking
- subagent handoff
- subagent fanout synthesis

交付物：

- 变量、产物、工具断言、request-log 断言都得到验证

### 第 5 阶段：将困难场景保留在自定义 handler 中

- memory dreaming sweep
- config apply restart wake-up
- config restart capability flip
- runtime inventory drift

交付物：

- 保持相同的编写格式，但在需要时使用显式 custom-step blocks

### 第 6 阶段：删除硬编码场景映射

当包覆盖率足够高后：

- 删除 `extensions/qa-lab/src/suite.ts` 中大多数按场景分支的 TypeScript 逻辑

## Fake Slack / 富媒体支持

当前 QA 总线仍然是 text-first。

相关文件：

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

目前 QA 总线支持：

- 文本
- reactions
- threads

它还不能建模内联媒体附件。

### 所需的传输契约

添加通用 QA 总线附件模型：

```ts
type QaBusAttachment = {
  id: string;
  kind: "image" | "video" | "audio" | "file";
  mimeType: string;
  fileName?: string;
  inline?: boolean;
  url?: string;
  contentBase64?: string;
  width?: number;
  height?: number;
  durationMs?: number;
  altText?: string;
  transcript?: string;
};
```

然后将 `attachments?: QaBusAttachment[]` 添加到：

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### 为什么要先做通用层

不要构建仅限 Slack 的媒体模型。

而应该：

- 先有一个通用 QA 传输模型
- 再在其上构建多个渲染器
  - 当前 QA Lab chat
  - 未来的 fake Slack web
  - 其他任何 fake transport views

这样可以避免重复逻辑，并让媒体场景保持与传输层无关。

### 所需的 UI 工作

更新 QA UI 以渲染：

- 内联图片预览
- 内联音频播放器
- 内联视频播放器
- 文件附件 chip

当前 UI 已经可以渲染线程和 reactions，因此附件渲染应可以叠加到同一消息卡片模型上。

### 媒体传输启用后的场景工作

一旦附件可以通过 QA 总线流转，我们就可以添加更丰富的 fake-chat 场景：

- fake Slack 中的内联图片回复
- 音频附件理解
- 视频附件理解
- 混合附件顺序
- 保留媒体的线程回复

## 建议

下一块实现工作应是：

1. 添加 Markdown 场景加载器 + zod schema
2. 从 Markdown 生成当前目录
3. 先迁移几个简单场景
4. 添加通用 QA 总线附件支持
5. 在 QA UI 中渲染内联图片
6. 然后再扩展到音频和视频

这是能同时验证两个目标的最小路径：

- 通用的 Markdown 定义 QA
- 更丰富的 fake messaging surfaces

## 开放问题

- 场景文件是否应允许嵌入带变量插值的 Markdown 提示词模板
- setup / cleanup 应该是具名 section，还是仅作为有序动作列表
- 产物引用在 schema 中应采用强类型，还是基于字符串
- 自定义 handler 应放在一个统一 registry 中，还是按 surface 分 registry
- 在迁移期间，生成的 JSON 兼容性文件是否应继续保留为已检入状态
