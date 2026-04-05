---
read_when:
    - 构建或调试原生 OpenClaw 插件
    - 理解插件能力模型或归属边界
    - 处理插件加载流水线或注册表
    - 实现提供商运行时钩子或渠道插件
sidebarTitle: Internals
summary: 插件内部机制：能力模型、归属、契约、加载流水线和运行时辅助函数
title: 插件内部机制
x-i18n:
    generated_at: "2026-04-05T22:33:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: d6e53a71b18d12cde81acac040c2110693dd2586b8888ff6104467ddd81ce22e
    source_path: plugins/architecture.md
    workflow: 15
---

# 插件内部机制

<Info>
  这是**深度架构参考**。如需实用指南，请参见：
  - [安装和使用插件](/zh-CN/tools/plugin) —— 用户指南
  - [入门指南](/zh-CN/plugins/building-plugins) —— 第一个插件教程
  - [渠道插件](/zh-CN/plugins/sdk-channel-plugins) —— 构建一个消息渠道
  - [提供商插件](/zh-CN/plugins/sdk-provider-plugins) —— 构建一个模型提供商
  - [SDK 概览](/zh-CN/plugins/sdk-overview) —— 导入映射与注册 API
</Info>

本页介绍 OpenClaw 插件系统的内部架构。

## 公共能力模型

能力是 OpenClaw 内部公开的**原生插件**模型。每个原生 OpenClaw 插件都会针对一种或多种能力类型进行注册：

| 能力 | 注册方法 | 示例插件 |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| 文本推理 | `api.registerProvider(...)` | `openai`, `anthropic` |
| 语音 | `api.registerSpeechProvider(...)` | `elevenlabs`, `microsoft` |
| 实时转录 | `api.registerRealtimeTranscriptionProvider(...)` | `openai` |
| 实时语音 | `api.registerRealtimeVoiceProvider(...)` | `openai` |
| 媒体理解 | `api.registerMediaUnderstandingProvider(...)` | `openai`, `google` |
| 图像生成 | `api.registerImageGenerationProvider(...)` | `openai`, `google`, `fal`, `minimax` |
| 视频生成 | `api.registerVideoGenerationProvider(...)` | `qwen` |
| 网页抓取 | `api.registerWebFetchProvider(...)` | `firecrawl` |
| 网页搜索 | `api.registerWebSearchProvider(...)` | `google` |
| 渠道 / 消息传递 | `api.registerChannel(...)` | `msteams`, `matrix` |

一个注册了零个能力，但提供钩子、工具或服务的插件，是**仅钩子遗留**插件。这种模式仍然受到完全支持。

### 外部兼容性立场

能力模型已经在核心中落地，并且如今已被内置 / 原生插件使用，但外部插件兼容性仍然需要比“它已导出，因此它已冻结”更严格的标准。

当前指导原则：

- **现有外部插件：** 保持基于钩子的集成继续可用；将此视为兼容性的基线
- **新的内置 / 原生插件：** 优先使用显式能力注册，而不是厂商特定的深入访问或新的仅钩子设计
- **采用能力注册的外部插件：** 允许这样做，但除非文档明确将某个契约标记为稳定，否则应将能力专用辅助接口视为仍在演进中

实用规则：

- 能力注册 API 是预期的发展方向
- 在过渡期间，对于外部插件来说，遗留钩子仍然是最安全、最不易破坏的路径
- 导出的辅助子路径并不全都等价；优先使用狭义且有文档说明的契约，而不是偶然导出的辅助接口

### 插件形态

OpenClaw 会根据每个已加载插件的实际注册行为（而不只是静态元数据）将其归类为某种形态：

- **plain-capability** —— 只注册一种能力类型（例如仅提供商插件，如 `mistral`）
- **hybrid-capability** —— 注册多种能力类型（例如 `openai` 同时拥有文本推理、语音、媒体理解和图像生成）
- **hook-only** —— 仅注册钩子（类型化或自定义），不注册能力、工具、命令或服务
- **non-capability** —— 注册工具、命令、服务或路由，但不注册能力

使用 `openclaw plugins inspect <id>` 可查看插件的形态和能力细分。详见 [CLI 参考](/cli/plugins#inspect)。

### 遗留钩子

`before_agent_start` 钩子仍作为仅钩子插件的兼容路径受到支持。现实中的遗留插件仍然依赖它。

方向：

- 保持其可用
- 在文档中将其标记为遗留
- 对于模型 / 提供商覆盖工作，优先使用 `before_model_resolve`
- 对于提示词变更工作，优先使用 `before_prompt_build`
- 只有在真实使用量下降且夹具覆盖证明迁移安全之后，才考虑移除

### 兼容性信号

当你运行 `openclaw doctor` 或 `openclaw plugins inspect <id>` 时，可能会看到以下标签之一：

| 信号 | 含义 |
| -------------------------- | ------------------------------------------------------------ |
| **config valid** | 配置可正常解析，插件可正常解析 |
| **compatibility advisory** | 插件使用受支持但较旧的模式（例如 `hook-only`） |
| **legacy warning** | 插件使用 `before_agent_start`，该用法已弃用 |
| **hard error** | 配置无效或插件加载失败 |

目前，`hook-only` 和 `before_agent_start` 都不会导致你的插件中断 —— `hook-only` 只是提示，而 `before_agent_start` 只会触发警告。这些信号也会显示在 `openclaw status --all` 和 `openclaw plugins doctor` 中。

## 架构概览

OpenClaw 的插件系统分为四层：

1. **清单 + 发现**
   OpenClaw 会从已配置路径、工作区根目录、全局扩展根目录和内置扩展中查找候选插件。发现流程会优先读取原生 `openclaw.plugin.json` 清单以及受支持的 bundle 清单。
2. **启用 + 验证**
   核心决定某个已发现的插件是启用、禁用、阻止，还是被选入某个独占槽位，例如 memory。
3. **运行时加载**
   原生 OpenClaw 插件通过 jiti 在进程内加载，并将能力注册到一个中央注册表中。兼容 bundle 会在不导入运行时代码的情况下被规范化为注册表记录。
4. **接口消费**
   OpenClaw 的其余部分通过读取注册表来暴露工具、渠道、提供商设置、钩子、HTTP 路由、CLI 命令和服务。

对于插件 CLI，根命令发现被拆分为两个阶段：

- 解析时元数据来自 `registerCli(..., { descriptors: [...] })`
- 真正的插件 CLI 模块可以保持惰性，并在首次调用时再注册

这样可以让插件自有的 CLI 代码保留在插件内部，同时仍允许 OpenClaw 在解析前预留根命令名称。

重要的设计边界：

- 发现 + 配置验证应基于**清单 / schema 元数据**完成，而无需执行插件代码
- 原生运行时行为来自插件模块的 `register(api)` 路径

这种拆分使 OpenClaw 能够在完整运行时尚未激活之前，就验证配置、说明缺失 / 禁用的插件，并构建 UI / schema 提示。

### 渠道插件与共享消息工具

对于正常聊天操作，渠道插件不需要分别注册发送 / 编辑 / 反应工具。OpenClaw 在核心中保留一个共享的 `message` 工具，而渠道插件则负责其背后的渠道专属发现与执行。

当前边界如下：

- 核心拥有共享 `message` 工具宿主、提示词接线、会话 / 线程记录以及执行分发
- 渠道插件拥有作用域动作发现、能力发现以及所有渠道专属 schema 片段
- 渠道插件拥有提供商专属的会话对话语法，例如对话 id 如何编码线程 id，或如何从父对话继承
- 渠道插件通过其动作适配器执行最终动作

对于渠道插件，SDK 接口为
`ChannelMessageActionAdapter.describeMessageTool(...)`。这个统一的发现调用允许插件将其可见动作、能力和 schema 贡献一起返回，以避免这些部分彼此漂移。

核心会将运行时作用域传入该发现步骤。重要字段包括：

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- 受信任的入站 `requesterSenderId`

这对于上下文敏感的插件很重要。渠道可以根据当前账户、当前房间 / 线程 / 消息或受信任的请求者身份来隐藏或暴露消息动作，而无需在核心 `message` 工具中硬编码渠道专属分支。

这也是为什么嵌入式运行器路由变更仍然属于插件工作：运行器负责将当前聊天 / 会话身份转发到插件发现边界，使共享 `message` 工具能为当前轮次暴露正确的渠道自有接口。

对于渠道自有执行辅助函数，内置插件应将执行运行时保留在各自的扩展模块中。核心不再拥有位于 `src/agents/tools` 下的 Discord、Slack、Telegram 或 WhatsApp 消息动作运行时。我们不会发布单独的 `plugin-sdk/*-action-runtime` 子路径，内置插件应直接从其扩展自有模块导入本地运行时代码。

相同边界同样适用于一般的提供商命名 SDK 接缝：核心不应导入针对 Slack、Discord、Signal、WhatsApp 或类似扩展的渠道专属便捷 barrel。如果核心需要某种行为，应当消费内置插件自己的 `api.ts` / `runtime-api.ts` barrel，或将需求提升为共享 SDK 中一个狭义、通用的能力。

对于投票，当前有两条执行路径：

- `outbound.sendPoll` 是适用于符合通用投票模型的渠道的共享基线
- `actions.handleAction("poll")` 是用于渠道专属投票语义或额外投票参数的首选路径

现在，核心会在插件投票分发拒绝该动作之后，才推迟执行共享投票解析，因此插件自有的投票处理器可以接受渠道专属投票字段，而不会先被通用投票解析器拦截。

完整启动顺序见[加载流水线](#load-pipeline)。

## 能力归属模型

OpenClaw 将原生插件视为某个**公司**或某个**功能**的归属边界，而不是一堆互不相关集成的杂物袋。

这意味着：

- 公司插件通常应拥有该公司的所有面向 OpenClaw 的接口
- 功能插件通常应拥有其引入的完整功能接口
- 渠道应消费共享核心能力，而不是临时重新实现提供商行为

例如：

- 内置的 `openai` 插件拥有 OpenAI 模型提供商行为，以及 OpenAI 的语音 + 实时语音 + 媒体理解 + 图像生成行为
- 内置的 `elevenlabs` 插件拥有 ElevenLabs 语音行为
- 内置的 `microsoft` 插件拥有 Microsoft 语音行为
- 内置的 `google` 插件拥有 Google 模型提供商行为，以及 Google 的媒体理解 + 图像生成 + 网页搜索行为
- 内置的 `firecrawl` 插件拥有 Firecrawl 网页抓取行为
- 内置的 `minimax`、`mistral`、`moonshot` 和 `zai` 插件拥有它们的媒体理解后端
- 内置的 `qwen` 插件拥有 Qwen 文本提供商行为，以及媒体理解和视频生成行为
- `voice-call` 插件是一个功能插件：它拥有通话传输、工具、CLI、路由和 Twilio 媒体流桥接，但它会消费共享的语音以及实时转录和实时语音能力，而不是直接导入厂商插件

预期的最终状态是：

- 即使 OpenAI 横跨文本模型、语音、图像以及未来的视频，它也应存在于一个插件中
- 其他厂商也可以对其自己的接口范围采用相同方式
- 渠道不关心哪个厂商插件拥有该提供商；它们只消费核心暴露的共享能力契约

这是关键区别：

- **plugin** = 归属边界
- **capability** = 可由多个插件实现或消费的核心契约

因此，如果 OpenClaw 增加一个新领域，例如视频，第一个问题不应是“哪个提供商应该硬编码视频处理？”而应是“核心视频能力契约是什么？”一旦这个契约存在，厂商插件就可以针对它注册，而渠道 / 功能插件也可以消费它。

如果该能力尚不存在，正确做法通常是：

1. 在核心中定义缺失的能力
2. 通过插件 API / 运行时以类型化方式暴露它
3. 让渠道 / 功能基于该能力接线
4. 让厂商插件注册具体实现

这样可以在保持归属明确的同时，避免核心行为依赖单一厂商或一次性的插件专属代码路径。

### 能力分层

决定代码归属时，可使用如下思维模型：

- **核心能力层**：共享编排、策略、回退、配置合并规则、投递语义和类型化契约
- **厂商插件层**：厂商专属 API、认证、模型目录、语音合成、图像生成、未来的视频后端、使用量端点
- **渠道 / 功能插件层**：Slack / Discord / voice-call 等集成，它们消费核心能力并将其呈现在某个接口上

例如，TTS 遵循这种形态：

- 核心拥有回复时 TTS 策略、回退顺序、偏好和渠道投递
- `openai`、`elevenlabs` 和 `microsoft` 拥有合成实现
- `voice-call` 消费电话 TTS 运行时辅助函数

未来能力也应优先采用相同模式。

### 多能力公司插件示例

一个公司插件从外部看应当具有一致性。如果 OpenClaw 为模型、语音、实时转录、实时语音、媒体理解、图像生成、视频生成、网页抓取和网页搜索提供共享契约，那么某个厂商就可以在一个地方拥有其所有接口：

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // auth/model catalog/runtime hooks
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // vendor speech config — implement the SpeechProviderPlugin interface directly
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // credential + fetch logic
      }),
    );
  },
};

export default plugin;
```

关键不在于辅助函数的确切名称，而在于这种形态：

- 一个插件拥有该厂商接口
- 核心仍然拥有能力契约
- 渠道和功能插件消费 `api.runtime.*` 辅助函数，而不是厂商代码
- 契约测试可以断言插件确实注册了它所宣称拥有的能力

### 能力示例：视频理解

OpenClaw 已经将图像 / 音频 / 视频理解视为一种共享能力。相同的归属模型也适用于这里：

1. 核心定义媒体理解契约
2. 厂商插件根据适用情况注册 `describeImage`、`transcribeAudio` 和 `describeVideo`
3. 渠道和功能插件消费共享核心行为，而不是直接接线到厂商代码

这样可以避免将某个提供商的视频假设固化进核心。插件拥有厂商接口；核心拥有能力契约和回退行为。

视频生成也已经采用同样的顺序：核心拥有类型化能力契约和运行时辅助函数，而厂商插件则针对 `api.registerVideoGenerationProvider(...)` 注册实现。

需要一个具体的发布清单？请参见[能力扩展手册](/zh-CN/plugins/architecture)。

## 契约与强制执行

插件 API 接口被有意设计为类型化并集中在 `OpenClawPluginApi` 中。该契约定义了受支持的注册点，以及插件可以依赖的运行时辅助函数。

其重要性在于：

- 插件作者获得一个稳定的内部标准
- 核心可以拒绝重复归属，例如两个插件注册同一个 provider id
- 启动时可以为格式错误的注册暴露可执行的诊断信息
- 契约测试可以强制执行内置插件归属，并防止静默漂移

这里有两层强制执行：

1. **运行时注册强制执行**
   插件注册表会在插件加载时验证注册内容。例如：重复 provider id、重复语音提供商 id，以及格式错误的注册都会产生插件诊断，而不是导致未定义行为。
2. **契约测试**
   在测试运行期间，内置插件会被捕获进契约注册表，以便 OpenClaw 能显式断言归属。目前这用于模型提供商、语音提供商、网页搜索提供商和内置注册归属。

实际效果是，OpenClaw 能够预先知道哪个插件拥有哪些接口。这让核心和渠道可以无缝组合，因为归属是声明式、类型化且可测试的，而不是隐式的。

### 什么内容应该属于契约

好的插件契约应当具备以下特征：

- 类型化
- 小而精
- 能力专属
- 由核心拥有
- 可被多个插件复用
- 可被渠道 / 功能消费，而无需了解厂商细节

不好的插件契约包括：

- 隐藏在核心中的厂商专属策略
- 绕过注册表的一次性插件逃生口
- 直接深入访问厂商实现的渠道代码
- 不属于 `OpenClawPluginApi` 或 `api.runtime` 的临时运行时对象

如果不确定，就提高抽象层级：先定义能力，再让插件接入它。

## 执行模型

原生 OpenClaw 插件与 Gateway 网关**在同一进程内**运行。它们不是沙箱隔离的。已加载的原生插件与核心代码共享相同的进程级信任边界。

这意味着：

- 原生插件可以注册工具、网络处理器、钩子和服务
- 原生插件中的 bug 可能导致 Gateway 网关崩溃或不稳定
- 恶意原生插件等同于在 OpenClaw 进程内执行任意代码

兼容 bundle 默认更安全，因为 OpenClaw 当前将它们视为元数据 / 内容包。在当前版本中，这主要指内置 Skills。

对于非内置插件，请使用 allowlist 和显式安装 / 加载路径。将工作区插件视为开发时代码，而不是生产默认项。

对于内置工作区包名，保持插件 id 锚定在 npm 名称中：默认使用 `@openclaw/<id>`，或在包有意暴露更窄的插件角色时使用获批的类型后缀，例如 `-provider`、`-plugin`、`-speech`、`-sandbox` 或 `-media-understanding`。

重要的信任说明：

- `plugins.allow` 信任的是**插件 id**，而不是来源证明。
- 与内置插件同 id 的工作区插件，在启用 / 加入 allowlist 时，会有意遮蔽内置副本。
- 这是正常且有用的行为，适用于本地开发、补丁测试和热修复。

## 导出边界

OpenClaw 导出的是能力，而不是实现便捷接口。

保持能力注册公开。裁剪非契约辅助导出：

- 内置插件专属辅助子路径
- 非公共 API 的运行时底层子路径
- 厂商专属便捷辅助函数
- 属于实现细节的设置 / 新手引导辅助函数

某些内置插件辅助子路径仍然保留在生成的 SDK 导出映射中，以兼容和维护内置插件。当前示例包括 `plugin-sdk/feishu`、`plugin-sdk/feishu-setup`、`plugin-sdk/zalo`、`plugin-sdk/zalo-setup` 以及若干 `plugin-sdk/matrix*` 接缝。应将它们视为保留的实现细节导出，而不是新第三方插件推荐采用的 SDK 模式。

## 加载流水线

在启动时，OpenClaw 大致会执行以下步骤：

1. 发现候选插件根目录
2. 读取原生或兼容 bundle 的清单与包元数据
3. 拒绝不安全候选项
4. 规范化插件配置（`plugins.enabled`、`allow`、`deny`、`entries`、`slots`、`load.paths`）
5. 决定每个候选项是否启用
6. 通过 jiti 加载已启用的原生模块
7. 调用原生 `register(api)`（或 `activate(api)` —— 一个遗留别名）钩子，并将注册内容收集到插件注册表中
8. 将注册表暴露给命令 / 运行时接口

<Note>
`activate` 是 `register` 的遗留别名 —— 加载器会解析当前存在的那个（`def.register ?? def.activate`），并在相同位置调用它。所有内置插件都使用 `register`；新插件请优先使用 `register`。
</Note>

安全门会发生在**运行时执行之前**。当入口逃出插件根目录、路径可被所有人写入，或者对于非内置插件来说路径归属看起来可疑时，候选项会被阻止。

### 清单优先行为

清单是控制面的真实来源。OpenClaw 用它来：

- 识别插件
- 发现声明的 channels / Skills / 配置 schema 或 bundle 能力
- 验证 `plugins.entries.<id>.config`
- 增强 Control UI 标签 / 占位符
- 显示安装 / 目录元数据

对于原生插件，运行时模块则是数据面部分。它会注册钩子、工具、命令或提供商流程等实际行为。

### 加载器缓存了什么

OpenClaw 会保留一些短生命周期的进程内缓存，用于：

- 发现结果
- 清单注册表数据
- 已加载插件注册表

这些缓存可减少突发启动和重复命令的开销。它们可以被视为短期性能缓存，而不是持久化。

性能说明：

- 设置 `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` 或
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` 可禁用这些缓存。
- 使用 `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` 和
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS` 调整缓存窗口。

## 注册表模型

已加载的插件不会直接修改任意核心全局状态。它们会注册到一个中央插件注册表中。

注册表会跟踪：

- 插件记录（身份、来源、出处、状态、诊断）
- 工具
- 遗留钩子和类型化钩子
- 渠道
- 提供商
- Gateway 网关 RPC 处理器
- HTTP 路由
- CLI 注册器
- 后台服务
- 插件自有命令

随后，核心功能会从该注册表中读取，而不是直接与插件模块交互。这使加载保持单向：

- 插件模块 -> 注册表注册
- 核心运行时 -> 注册表消费

这种分离对于可维护性很重要。它意味着大多数核心接口只需要一个集成点：“读取注册表”，而不是“为每个插件模块做特殊处理”。

## 对话绑定回调

绑定对话的插件可以在审批结果确定后作出响应。

使用 `api.onConversationBindingResolved(...)` 可在绑定请求被批准或拒绝后收到回调：

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // A binding now exists for this plugin + conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // The request was denied; clear any local pending state.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

回调负载字段：

- `status`：`"approved"` 或 `"denied"`
- `decision`：`"allow-once"`、`"allow-always"` 或 `"deny"`
- `binding`：已获批请求的最终绑定
- `request`：原始请求摘要、detach 提示、发送者 id 和对话元数据

该回调仅用于通知。它不会改变谁被允许绑定对话，并且它会在核心审批处理完成后运行。

## 提供商运行时钩子

提供商插件现在分为两层：

- 清单元数据：`providerAuthEnvVars` 用于在运行时加载前进行低成本环境认证查找，`providerAuthChoices` 用于在运行时加载前提供低成本的新手引导 / 认证选择标签和 CLI 标志元数据
- 配置时钩子：`catalog` / 遗留 `discovery` 以及 `applyConfigDefaults`
- 运行时钩子：`normalizeModelId`、`normalizeTransport`、
  `normalizeConfig`、
  `applyNativeStreamingUsageCompat`、`resolveConfigApiKey`、
  `resolveSyntheticAuth`、`shouldDeferSyntheticProfileAuth`、
  `resolveDynamicModel`、`prepareDynamicModel`、`normalizeResolvedModel`、
  `contributeResolvedModelCompat`、`capabilities`、
  `normalizeToolSchemas`、`inspectToolSchemas`、
  `resolveReasoningOutputMode`、`prepareExtraParams`、`createStreamFn`、
  `wrapStreamFn`、`resolveTransportTurnState`、
  `resolveWebSocketSessionPolicy`、`formatApiKey`、`refreshOAuth`、
  `buildAuthDoctorHint`、`matchesContextOverflowError`、
  `classifyFailoverReason`、`isCacheTtlEligible`、
  `buildMissingAuthMessage`、`suppressBuiltInModel`、`augmentModelCatalog`、
  `isBinaryThinking`、`supportsXHighThinking`、
  `resolveDefaultThinkingLevel`、`isModernModelRef`、`prepareRuntimeAuth`、
  `resolveUsageAuth`、`fetchUsageSnapshot`、`createEmbeddingProvider`、
  `buildReplayPolicy`、
  `sanitizeReplayHistory`、`validateReplayTurns`、`onModelSelected`

OpenClaw 仍然拥有通用智能体循环、故障转移、转录处理和工具策略。这些钩子是在不需要整套自定义推理传输的前提下，为提供商专属行为提供的扩展接口。

当提供商具有基于环境变量的凭证，且通用认证 / 状态 / 模型选择器路径应在不加载插件运行时的情况下看到这些凭证时，请使用清单中的 `providerAuthEnvVars`。当新手引导 / 认证选择 CLI 接口需要在不加载提供商运行时的情况下知道提供商的 choice id、分组标签和简单单标志认证接线时，请使用清单中的 `providerAuthChoices`。将提供商运行时的 `envVars` 保留给面向操作员的提示，例如新手引导标签或 OAuth client-id / client-secret 设置变量。

### 钩子顺序与用法

对于模型 / 提供商插件，OpenClaw 大致按以下顺序调用钩子。
“何时使用”列是快速决策指南。

| #   | 钩子 | 作用 | 何时使用 |
| --- | --------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog` | 在生成 `models.json` 期间，将提供商配置发布到 `models.providers` 中 | 提供商拥有目录或 base URL 默认值 |
| 2   | `applyConfigDefaults` | 在配置具体化期间应用提供商自有的全局配置默认值 | 默认值依赖认证模式、环境或提供商模型家族语义 |
| --  | _(内置模型查找)_ | OpenClaw 会先尝试常规注册表 / 目录路径 | _(不是插件钩子)_ |
| 3   | `normalizeModelId` | 在查找前规范化遗留或预览版 model-id 别名 | 提供商在规范模型解析前拥有别名清理逻辑 |
| 4   | `normalizeTransport` | 在通用模型组装前规范化提供商家族的 `api` / `baseUrl` | 提供商在同一传输家族中拥有自定义 provider id 的传输清理逻辑 |
| 5   | `normalizeConfig` | 在运行时 / 提供商解析前规范化 `models.providers.<id>` | 提供商需要把配置清理逻辑放在插件中；内置的 Google 家族辅助函数也会为受支持的 Google 配置项兜底 |
| 6   | `applyNativeStreamingUsageCompat` | 对配置提供商应用原生流式使用量兼容重写 | 提供商需要基于端点的原生流式使用量元数据修复 |
| 7   | `resolveConfigApiKey` | 在加载运行时认证前，为配置提供商解析环境标记认证 | 提供商拥有环境标记 API key 解析逻辑；`amazon-bedrock` 这里也有内置 AWS 环境标记解析器 |
| 8   | `resolveSyntheticAuth` | 在不持久化明文的前提下暴露本地 / 自托管或配置支持的认证 | 提供商可使用合成 / 本地凭证标记运行 |
| 9   | `shouldDeferSyntheticProfileAuth` | 在环境 / 配置支持的认证之后，降低已存储合成配置档占位符的优先级 | 提供商存储了合成占位配置档，且它们不应获得更高优先级 |
| 10  | `resolveDynamicModel` | 为本地注册表中尚不存在的提供商自有 model id 提供同步回退 | 提供商接受任意上游 model id |
| 11  | `prepareDynamicModel` | 异步预热，然后再次运行 `resolveDynamicModel` | 提供商在解析未知 id 之前需要网络元数据 |
| 12  | `normalizeResolvedModel` | 在嵌入式运行器使用已解析模型前做最终重写 | 提供商需要传输重写，但仍使用核心传输 |
| 13  | `contributeResolvedModelCompat` | 为另一种兼容传输背后的厂商模型贡献兼容标志 | 提供商能够在代理传输上识别自己的模型，而无需接管该提供商 |
| 14  | `capabilities` | 被共享核心逻辑使用的提供商自有转录 / 工具元数据 | 提供商需要处理转录 / 提供商家族特性 |
| 15  | `normalizeToolSchemas` | 在嵌入式运行器看到工具 schema 前进行规范化 | 提供商需要传输家族 schema 清理 |
| 16  | `inspectToolSchemas` | 在规范化后暴露提供商自有 schema 诊断 | 提供商希望给出关键字警告，而无需让核心学习提供商专属规则 |
| 17  | `resolveReasoningOutputMode` | 选择原生或带标签的推理输出契约 | 提供商需要使用带标签的推理 / 最终输出，而不是原生字段 |
| 18  | `prepareExtraParams` | 在通用流式选项包装器前进行请求参数规范化 | 提供商需要默认请求参数或按提供商进行参数清理 |
| 19  | `createStreamFn` | 用自定义传输完全替换正常流式路径 | 提供商需要自定义线协议，而不仅仅是包装器 |
| 20  | `wrapStreamFn` | 在通用包装器应用后再包装流式函数 | 提供商需要请求头 / 请求体 / 模型兼容包装器，而非自定义传输 |
| 21  | `resolveTransportTurnState` | 附加原生的逐轮传输头或元数据 | 提供商希望通用传输发送提供商原生轮次身份 |
| 22  | `resolveWebSocketSessionPolicy` | 附加原生 WebSocket 头或会话冷却策略 | 提供商希望通用 WS 传输调整会话头或回退策略 |
| 23  | `formatApiKey` | 认证配置档格式化器：将存储的配置档转换为运行时 `apiKey` 字符串 | 提供商存储额外认证元数据，并需要自定义运行时令牌形态 |
| 24  | `refreshOAuth` | 为自定义刷新端点或刷新失败策略覆盖 OAuth 刷新 | 提供商不适配共享的 `pi-ai` 刷新器 |
| 25  | `buildAuthDoctorHint` | OAuth 刷新失败时附加的修复提示 | 提供商在刷新失败后需要提供商自有的认证修复指导 |
| 26  | `matchesContextOverflowError` | 提供商自有上下文窗口溢出匹配器 | 提供商有通用启发式无法识别的原始溢出错误 |
| 27  | `classifyFailoverReason` | 提供商自有故障转移原因分类 | 提供商可以将原始 API / 传输错误映射为限流 / 过载等 |
| 28  | `isCacheTtlEligible` | 代理 / 回程提供商的提示词缓存策略 | 提供商需要代理专属缓存 TTL 门控 |
| 29  | `buildMissingAuthMessage` | 替换通用的缺失认证恢复消息 | 提供商需要提供商专属的缺失认证恢复提示 |
| 30  | `suppressBuiltInModel` | 过时上游模型抑制，以及可选的面向用户错误提示 | 提供商需要隐藏过时上游条目或以厂商提示替换它们 |
| 31  | `augmentModelCatalog` | 在发现后追加合成 / 最终目录条目 | 提供商需要在 `models list` 和选择器中加入合成前向兼容条目 |
| 32  | `isBinaryThinking` | 为二元思考提供商提供开 / 关推理切换 | 提供商仅暴露二元开 / 关思考 |
| 33  | `supportsXHighThinking` | 为选定模型提供 `xhigh` 推理支持 | 提供商仅希望在一部分模型上启用 `xhigh` |
| 34  | `resolveDefaultThinkingLevel` | 为特定模型家族指定默认 `/think` 级别 | 提供商拥有某个模型家族的默认 `/think` 策略 |
| 35  | `isModernModelRef` | 用于实时配置档筛选和 smoke 选择的现代模型匹配器 | 提供商拥有 live / smoke 首选模型匹配逻辑 |
| 36  | `prepareRuntimeAuth` | 在推理前最后一步，将已配置凭证交换为实际运行时令牌 / 密钥 | 提供商需要令牌交换或短期请求凭证 |
| 37  | `resolveUsageAuth` | 为 `/usage` 及相关状态接口解析使用量 / 计费凭证 | 提供商需要自定义使用量 / 配额令牌解析或不同的使用量凭证 |
| 38  | `fetchUsageSnapshot` | 在认证解析后抓取并规范化提供商专属使用量 / 配额快照 | 提供商需要提供商专属使用量端点或负载解析器 |
| 39  | `createEmbeddingProvider` | 为 memory / 搜索 构建提供商自有 embedding 适配器 | memory embedding 行为应属于提供商插件 |
| 40  | `buildReplayPolicy` | 返回一个控制提供商转录处理方式的重放策略 | 提供商需要自定义转录策略（例如去除 thinking 块） |
| 41  | `sanitizeReplayHistory` | 在通用转录清理后重写重放历史 | 提供商需要超出共享压缩辅助函数之外的提供商专属重放重写 |
| 42  | `validateReplayTurns` | 在嵌入式运行器之前，对重放轮次进行最终验证或重塑 | 提供商传输需要在通用净化后执行更严格的轮次验证 |
| 43  | `onModelSelected` | 运行提供商自有的选择后副作用 | 当模型变为活动状态时，提供商需要遥测或提供商自有状态 |

`normalizeModelId`、`normalizeTransport` 和 `normalizeConfig` 会先检查匹配到的提供商插件，然后再继续检查其他具备相应钩子能力的提供商插件，直到其中某个插件真的修改了 model id 或 transport / config。这样可以让别名 / 兼容提供商 shim 正常工作，而无需让调用方知道哪个内置插件拥有该重写逻辑。如果没有任何提供商钩子重写受支持的 Google 家族配置项，内置 Google 配置规范化器仍会应用该兼容清理。

如果提供商需要完全自定义的线协议或自定义请求执行器，那是另一类扩展。这些钩子适用于仍运行在 OpenClaw 常规推理循环上的提供商行为。

### 提供商示例

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### 内置示例

- Anthropic 使用 `resolveDynamicModel`、`capabilities`、`buildAuthDoctorHint`、
  `resolveUsageAuth`、`fetchUsageSnapshot`、`isCacheTtlEligible`、
  `resolveDefaultThinkingLevel`、`applyConfigDefaults`、`isModernModelRef`
  和 `wrapStreamFn`，因为它拥有 Claude 4.6 前向兼容、
  提供商家族提示、认证修复指导、使用量端点集成、
  提示词缓存资格、带认证感知的配置默认值、Claude
  默认 / 自适应思考策略，以及面向 beta headers、
  `/fast` / `serviceTier` 和 `context1m` 的 Anthropic 专属流式整形。
- Anthropic 的 Claude 专属流式辅助函数目前保留在该内置插件自己的公共 `api.ts` / `contract-api.ts` 接缝中。该包接口
  导出 `wrapAnthropicProviderStream`、`resolveAnthropicBetas`、
  `resolveAnthropicFastMode`、`resolveAnthropicServiceTier`，以及更底层的
  Anthropic 包装器构建函数，而不是围绕某个提供商的 beta-header 规则扩展通用 SDK。
- OpenAI 使用 `resolveDynamicModel`、`normalizeResolvedModel` 和
  `capabilities`，以及 `buildMissingAuthMessage`、`suppressBuiltInModel`、
  `augmentModelCatalog`、`supportsXHighThinking` 和 `isModernModelRef`，
  因为它拥有 GPT-5.4 前向兼容、直接 OpenAI
  `openai-completions` -> `openai-responses` 规范化、Codex 感知认证
  提示、Spark 抑制、合成 OpenAI 列表条目，以及 GPT-5 思考 /
  live-model 策略；`openai-responses-defaults` 流式家族拥有
  共享原生 OpenAI Responses 包装器，用于 attribution headers、
  `/fast`/`serviceTier`、文本冗长度、原生 Codex 网页搜索、
  reasoning-compat 负载整形，以及 Responses 上下文管理。
- OpenRouter 使用 `catalog` 以及 `resolveDynamicModel` 和
  `prepareDynamicModel`，因为该提供商是透传型的，可能会在 OpenClaw 的静态目录更新之前暴露新的 model id；它还使用
  `capabilities`、`wrapStreamFn` 和 `isCacheTtlEligible`，以便将
  提供商专属请求头、路由元数据、推理补丁和提示词缓存策略留在核心之外。它的重放策略来自
  `passthrough-gemini` 家族，而 `openrouter-thinking` 流式家族则拥有代理推理注入和不受支持模型 / `auto` 跳过逻辑。
- GitHub Copilot 使用 `catalog`、`auth`、`resolveDynamicModel` 和
  `capabilities`，以及 `prepareRuntimeAuth` 和 `fetchUsageSnapshot`，因为它
  需要提供商自有设备登录、模型回退行为、Claude 转录特性、
  GitHub token -> Copilot token 交换，以及提供商自有使用量端点。
- OpenAI Codex 使用 `catalog`、`resolveDynamicModel`、
  `normalizeResolvedModel`、`refreshOAuth` 和 `augmentModelCatalog`，以及
  `prepareExtraParams`、`resolveUsageAuth` 和 `fetchUsageSnapshot`，因为它
  仍运行在核心 OpenAI 传输之上，但拥有自己的传输 / base URL
  规范化、OAuth 刷新回退策略、默认传输选择、
  合成 Codex 目录条目，以及 ChatGPT 使用量端点集成；它与直接 OpenAI 共享同一个 `openai-responses-defaults` 流式家族。
- Google AI Studio 和 Gemini CLI OAuth 使用 `resolveDynamicModel`、
  `buildReplayPolicy`、`sanitizeReplayHistory`、
  `resolveReasoningOutputMode`、`wrapStreamFn` 和 `isModernModelRef`，因为
  `google-gemini` 重放家族拥有 Gemini 3.1 前向兼容回退、
  原生 Gemini 重放验证、bootstrap 重放净化、
  带标签的推理输出模式，以及现代模型匹配，而
  `google-thinking` 流式家族拥有 Gemini thinking 负载规范化；
  Gemini CLI OAuth 还使用 `formatApiKey`、`resolveUsageAuth` 和
  `fetchUsageSnapshot` 来处理令牌格式化、令牌解析和配额端点接线。
- Anthropic Vertex 通过
  `anthropic-by-model` 重放家族使用 `buildReplayPolicy`，这样 Claude 专属重放清理就只作用于 Claude id，而不是所有 `anthropic-messages` 传输。
- Amazon Bedrock 使用 `buildReplayPolicy`、`matchesContextOverflowError`、
  `classifyFailoverReason` 和 `resolveDefaultThinkingLevel`，因为它拥有
  针对 Anthropic-on-Bedrock 流量的 Bedrock 专属限流 / 未就绪 / 上下文溢出错误分类；其重放策略仍共享相同的
  仅 Claude `anthropic-by-model` 防护。
- OpenRouter、Kilocode、Opencode 和 Opencode Go 通过 `passthrough-gemini` 重放家族使用 `buildReplayPolicy`，因为它们通过 OpenAI 兼容传输代理 Gemini 模型，并需要 Gemini thought-signature 净化，而不需要原生 Gemini 重放验证或 bootstrap 重写。
- MiniMax 通过
  `hybrid-anthropic-openai` 重放家族使用 `buildReplayPolicy`，因为一个提供商同时拥有 Anthropic-message 和 OpenAI 兼容语义；它会在 Anthropic 一侧保留仅 Claude 的 thinking 块丢弃逻辑，同时将推理输出模式覆盖回原生，而 `minimax-fast-mode` 流式家族拥有共享流式路径上的 fast-mode 模型重写。
- Moonshot 使用 `catalog` 加 `wrapStreamFn`，因为它仍使用共享
  OpenAI 传输，但需要提供商自有 thinking 负载规范化；`moonshot-thinking` 流式家族会把配置和 `/think` 状态映射到其原生二元 thinking 负载上。
- Kilocode 使用 `catalog`、`capabilities`、`wrapStreamFn` 和
  `isCacheTtlEligible`，因为它需要提供商自有请求头、
  推理负载规范化、Gemini 转录提示和 Anthropic
  缓存 TTL 门控；`kilocode-thinking` 流式家族会在共享代理流式路径上保留 Kilo thinking 注入，同时跳过 `kilo/auto` 和其他不支持显式推理负载的代理 model id。
- Z.AI 使用 `resolveDynamicModel`、`prepareExtraParams`、`wrapStreamFn`、
  `isCacheTtlEligible`、`isBinaryThinking`、`isModernModelRef`、
  `resolveUsageAuth` 和 `fetchUsageSnapshot`，因为它拥有 GLM-5 回退、
  `tool_stream` 默认值、二元 thinking 用户体验、现代模型匹配，以及
  使用量认证和配额抓取；`tool-stream-default-on` 流式家族让默认开启的 `tool_stream` 包装器不必散落在各提供商的手写胶水代码中。
- xAI 使用 `normalizeResolvedModel`、`normalizeTransport`、
  `contributeResolvedModelCompat`、`prepareExtraParams`、`wrapStreamFn`、
  `resolveSyntheticAuth`、`resolveDynamicModel` 和 `isModernModelRef`，
  因为它拥有原生 xAI Responses 传输规范化、Grok fast-mode
  别名重写、默认 `tool_stream`、strict-tool / reasoning-payload
  清理、面向插件自有工具的回退认证复用、前向兼容 Grok
  模型解析，以及提供商自有兼容补丁，例如 xAI 工具 schema
  配置档、不支持的 schema 关键字、原生 `web_search` 和 HTML 实体
  工具调用参数解码。
- Mistral、OpenCode Zen 和 OpenCode Go 仅使用 `capabilities`，以便将转录 / 工具特性保留在核心之外。
- 仅目录型内置提供商，如 `byteplus`、`cloudflare-ai-gateway`、
  `huggingface`、`kimi-coding`、`nvidia`、`qianfan`、
  `synthetic`、`together`、`venice`、`vercel-ai-gateway` 和 `volcengine`，仅使用
  `catalog`。
- Qwen 对其文本提供商使用 `catalog`，同时对其多模态接口注册共享的媒体理解和视频生成能力。
- MiniMax 和 Xiaomi 使用 `catalog` 加使用量钩子，因为它们的 `/usage`
  行为属于插件自有，即使推理仍通过共享传输运行。

## 运行时辅助函数

插件可以通过 `api.runtime` 访问部分核心辅助函数。对于 TTS：

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

说明：

- `textToSpeech` 返回用于文件 / 语音笔记接口的常规核心 TTS 输出负载。
- 使用核心 `messages.tts` 配置和提供商选择。
- 返回 PCM 音频缓冲区 + 采样率。插件必须为提供商自行重采样 / 编码。
- `listVoices` 对每个提供商都是可选的。可将它用于厂商自有语音选择器或设置流程。
- 语音列表可以包含更丰富的元数据，例如地区、性别和个性标签，以供提供商感知的选择器使用。
- 目前 OpenAI 和 ElevenLabs 支持电话场景。Microsoft 不支持。

插件也可以通过 `api.registerSpeechProvider(...)` 注册语音提供商。

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

说明：

- 将 TTS 策略、回退和回复投递保留在核心中。
- 使用语音提供商承载厂商自有合成行为。
- 遗留 Microsoft `edge` 输入会被规范化为 `microsoft` provider id。
- 首选归属模型是面向公司的：随着 OpenClaw 增加这些能力契约，一个厂商插件可以同时拥有文本、语音、图像和未来媒体提供商。

对于图像 / 音频 / 视频理解，插件应注册一个类型化的媒体理解提供商，而不是通用键值包：

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

说明：

- 将编排、回退、配置和渠道接线保留在核心中。
- 将厂商行为保留在提供商插件中。
- 增量扩展应保持类型化：新的可选方法、新的可选结果字段、新的可选能力。
- 视频生成已经遵循相同模式：
  - 核心拥有能力契约和运行时辅助函数
  - 厂商插件注册 `api.registerVideoGenerationProvider(...)`
  - 功能 / 渠道插件消费 `api.runtime.videoGeneration.*`

对于媒体理解运行时辅助函数，插件可以调用：

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

对于音频转录，插件可以使用媒体理解运行时，或者使用较旧的 STT 别名：

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

说明：

- `api.runtime.mediaUnderstanding.*` 是图像 / 音频 / 视频理解的首选共享接口。
- 使用核心媒体理解音频配置（`tools.media.audio`）和提供商回退顺序。
- 当未产生转录输出时（例如输入被跳过 / 不受支持），返回 `{ text: undefined }`。
- `api.runtime.stt.transcribeAudioFile(...)` 仍保留为兼容别名。

插件还可以通过 `api.runtime.subagent` 启动后台子智能体运行：

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

说明：

- `provider` 和 `model` 是每次运行的可选覆盖项，不会持久化更改会话。
- OpenClaw 仅对受信任调用方接受这些覆盖字段。
- 对于插件自有回退运行，操作员必须通过 `plugins.entries.<id>.subagent.allowModelOverride: true` 显式选择启用。
- 使用 `plugins.entries.<id>.subagent.allowedModels` 将受信任插件限制为特定规范 `provider/model` 目标，或使用 `"*"` 显式允许任何目标。
- 不受信任的插件子智能体运行仍可工作，但覆盖请求会被拒绝，而不是静默回退。

对于网页搜索，插件可以消费共享运行时辅助函数，而不是深入访问智能体工具接线：

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

插件也可以通过
`api.registerWebSearchProvider(...)` 注册网页搜索提供商。

说明：

- 将提供商选择、凭证解析和共享请求语义保留在核心中。
- 使用网页搜索提供商来承载厂商专属搜索传输。
- `api.runtime.webSearch.*` 是功能 / 渠道插件在需要搜索行为但不依赖智能体工具包装器时的首选共享接口。

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`：使用已配置的图像生成提供商链生成图像。
- `listProviders(...)`：列出可用的图像生成提供商及其能力。

## Gateway 网关 HTTP 路由

插件可以通过 `api.registerHttpRoute(...)` 暴露 HTTP 端点。

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

路由字段：

- `path`：Gateway 网关 HTTP 服务器下的路由路径。
- `auth`：必填。使用 `"gateway"` 表示要求正常 Gateway 网关认证，或使用 `"plugin"` 表示由插件管理认证 / webhook 验签。
- `match`：可选。`"exact"`（默认）或 `"prefix"`。
- `replaceExisting`：可选。允许同一插件替换自己已有的路由注册。
- `handler`：当路由处理了请求时返回 `true`。

说明：

- `api.registerHttpHandler(...)` 已移除，并会导致插件加载错误。请改用 `api.registerHttpRoute(...)`。
- 插件路由必须显式声明 `auth`。
- 精确的 `path + match` 冲突会被拒绝，除非设置 `replaceExisting: true`，并且一个插件不能替换另一个插件的路由。
- 不同 `auth` 级别的重叠路由会被拒绝。仅在相同认证级别内保留 `exact` / `prefix` 贯穿链。
- `auth: "plugin"` 路由**不会**自动收到操作员运行时作用域。它们用于插件管理的 webhook / 签名验证，而不是特权 Gateway 网关辅助调用。
- `auth: "gateway"` 路由会在 Gateway 网关请求运行时作用域内运行，但该作用域是有意保守的：
  - 共享密钥 bearer 认证（`gateway.auth.mode = "token"` / `"password"`）会将插件路由运行时作用域固定为 `operator.write`，即使调用方发送了 `x-openclaw-scopes`
  - 受信任的携带身份 HTTP 模式（例如 `trusted-proxy` 或私有入口上的 `gateway.auth.mode = "none"`）只有在明确存在该请求头时，才会尊重 `x-openclaw-scopes`
  - 如果这些携带身份的插件路由请求上不存在 `x-openclaw-scopes`，运行时作用域会回退到 `operator.write`
- 实用规则：不要假设一个带 gateway 认证的插件路由天然就是管理员接口。如果你的路由需要仅管理员行为，请要求使用携带身份的认证模式，并记录清楚显式 `x-openclaw-scopes` 请求头契约。

## 插件 SDK 导入路径

编写插件时，请使用 SDK 子路径，而不是单体的 `openclaw/plugin-sdk` 导入：

- `openclaw/plugin-sdk/plugin-entry` 用于插件注册原语。
- `openclaw/plugin-sdk/core` 用于通用共享的面向插件契约。
- `openclaw/plugin-sdk/config-schema` 用于根 `openclaw.json` Zod schema
  导出（`OpenClawSchema`）。
- 稳定的渠道原语，例如 `openclaw/plugin-sdk/channel-setup`、
  `openclaw/plugin-sdk/setup-runtime`、
  `openclaw/plugin-sdk/setup-adapter-runtime`、
  `openclaw/plugin-sdk/setup-tools`、
  `openclaw/plugin-sdk/channel-pairing`、
  `openclaw/plugin-sdk/channel-contract`、
  `openclaw/plugin-sdk/channel-feedback`、
  `openclaw/plugin-sdk/channel-inbound`、
  `openclaw/plugin-sdk/channel-lifecycle`、
  `openclaw/plugin-sdk/channel-reply-pipeline`、
  `openclaw/plugin-sdk/command-auth`、
  `openclaw/plugin-sdk/secret-input` 和
  `openclaw/plugin-sdk/webhook-ingress`，用于共享设置 / 认证 / 回复 / webhook
  接线。`channel-inbound` 是防抖、提及匹配、
  envelope 格式化和入站 envelope 上下文辅助函数的共享归属位置。
  `channel-setup` 是狭义的可选安装设置接缝。
  `setup-runtime` 是 `setupEntry` /
  延迟启动所使用、运行时安全的设置接口，包括导入安全的设置补丁适配器。
  `setup-adapter-runtime` 是具备环境感知的账户设置适配器接缝。
  `setup-tools` 是一个小型 CLI / 归档 / 文档辅助接口（`formatCliCommand`、
  `detectBinary`、`extractArchive`、`resolveBrewExecutable`、`formatDocsLink`、
  `CONFIG_DIR`）。
- 领域子路径，例如 `openclaw/plugin-sdk/channel-config-helpers`、
  `openclaw/plugin-sdk/allow-from`、
  `openclaw/plugin-sdk/channel-config-schema`、
  `openclaw/plugin-sdk/telegram-command-config`、
  `openclaw/plugin-sdk/channel-policy`、
  `openclaw/plugin-sdk/approval-runtime`、
  `openclaw/plugin-sdk/config-runtime`、
  `openclaw/plugin-sdk/infra-runtime`、
  `openclaw/plugin-sdk/agent-runtime`、
  `openclaw/plugin-sdk/lazy-runtime`、
  `openclaw/plugin-sdk/reply-history`、
  `openclaw/plugin-sdk/routing`、
  `openclaw/plugin-sdk/status-helpers`、
  `openclaw/plugin-sdk/text-runtime`、
  `openclaw/plugin-sdk/runtime-store` 和
  `openclaw/plugin-sdk/directory-runtime`，用于共享运行时 / 配置辅助函数。
  `telegram-command-config` 是 Telegram 自定义
  命令规范化 / 验证的狭义公共接缝，即使内置
  Telegram 契约接口暂时不可用时也会保留。
  `text-runtime` 是共享文本 / markdown / 日志接缝，包括
  assistant 可见文本剥离、markdown 渲染 / 分块辅助函数、脱敏
  辅助函数、directive-tag 辅助函数以及安全文本工具。
- 审批专用渠道接缝应优先采用插件上的单个 `approvalCapability`
  契约。然后核心通过这一个能力来读取审批认证、投递、渲染和
  原生路由行为，而不是将审批行为混杂到其他无关插件字段中。
- `openclaw/plugin-sdk/channel-runtime` 已弃用，仅保留为
  旧插件的兼容 shim。新代码应改为导入更窄的通用原语，并且仓库代码不应新增对该 shim 的导入。
- 内置扩展内部实现仍然是私有的。外部插件应仅使用
  `openclaw/plugin-sdk/*` 子路径。OpenClaw 核心 / 测试代码可以使用
  插件包根目录下的仓库公共入口点，例如 `index.js`、`api.js`、
  `runtime-api.js`、`setup-entry.js` 和范围更窄的文件，例如
  `login-qr-api.js`。核心或其他扩展绝不要导入某个插件包的 `src/*`。
- 仓库入口点拆分：
  `<plugin-package-root>/api.js` 是辅助函数 / 类型 barrel，
  `<plugin-package-root>/runtime-api.js` 是仅运行时 barrel，
  `<plugin-package-root>/index.js` 是内置插件入口，
  `<plugin-package-root>/setup-entry.js` 是设置插件入口。
- 当前内置提供商示例：
  - Anthropic 使用 `api.js` / `contract-api.js` 提供 Claude 流式辅助函数，例如
    `wrapAnthropicProviderStream`、beta-header 辅助函数和 `service_tier`
    解析。
  - OpenAI 使用 `api.js` 提供提供商构建器、默认模型辅助函数和
    实时提供商构建器。
  - OpenRouter 使用 `api.js` 提供其提供商构建器以及新手引导 / 配置
    辅助函数，而 `register.runtime.js` 仍可为仓库本地用途重新导出通用
    `plugin-sdk/provider-stream` 辅助函数。
- 由 facade 加载的公共入口点会优先使用当前活动的运行时配置快照；
  如果 OpenClaw 尚未提供运行时快照，则回退到磁盘上的已解析配置文件。
- 通用共享原语仍然是首选的公共 SDK 契约。仍保留一小组保留的兼容性内置渠道品牌辅助接缝。应将它们视为内置维护 / 兼容接缝，而不是新的第三方导入目标；新的跨渠道契约仍应落在通用 `plugin-sdk/*` 子路径或插件本地 `api.js` /
  `runtime-api.js` barrel 上。

兼容性说明：

- 新代码请避免使用根 `openclaw/plugin-sdk` barrel。
- 优先使用窄而稳定的原语。较新的 setup / pairing / reply /
  feedback / contract / inbound / threading / command / secret-input / webhook / infra /
  allowlist / status / message-tool 子路径是新内置和外部插件工作的预期契约。
  目标解析 / 匹配属于 `openclaw/plugin-sdk/channel-targets`。
  消息动作门控和反应消息 id 辅助函数属于
  `openclaw/plugin-sdk/channel-actions`。
- 内置扩展专属辅助 barrel 默认并不稳定。如果某个
  辅助函数仅被某个内置扩展需要，应将其保留在该扩展本地
  `api.js` 或 `runtime-api.js` 接缝后面，而不是提升到
  `openclaw/plugin-sdk/<extension>`。
- 新的共享辅助接缝应是通用的，而不是带渠道品牌的。共享目标
  解析属于 `openclaw/plugin-sdk/channel-targets`；渠道专属
  内部实现则保留在所属插件本地的 `api.js` 或 `runtime-api.js`
  接缝之后。
- `image-generation`、
  `media-understanding` 和 `speech` 等能力专用子路径之所以存在，是因为今天内置 / 原生插件确实会使用它们。但这本身并不意味着其中每个导出辅助函数都是长期冻结的外部契约。

## 消息工具 schema

插件应拥有渠道专属的 `describeMessageTool(...)` schema
贡献。将提供商专属字段保留在插件中，而不是共享核心中。

对于共享的可移植 schema 片段，请复用通过
`openclaw/plugin-sdk/channel-actions` 导出的通用辅助函数：

- `createMessageToolButtonsSchema()` 用于按钮网格风格负载
- `createMessageToolCardSchema()` 用于结构化卡片负载

如果某种 schema 形态只适用于一个提供商，就应在该插件自己的源码中定义，而不是将其提升进共享 SDK。

## 渠道目标解析

渠道插件应拥有渠道专属目标语义。保持共享 outbound 宿主的通用性，并通过消息适配器接口承载提供商规则：

- `messaging.inferTargetChatType({ to })` 决定规范化目标在目录查找前应被视为 `direct`、`group` 还是 `channel`。
- `messaging.targetResolver.looksLikeId(raw, normalized)` 告诉核心，某个输入是否应跳过目录搜索，直接进入类 id 解析。
- `messaging.targetResolver.resolveTarget(...)` 是当核心在规范化之后或目录未命中之后需要提供商自有最终解析时的插件回退。
- `messaging.resolveOutboundSessionRoute(...)` 在目标解析完成后拥有提供商专属会话路由构建逻辑。

推荐拆分：

- 使用 `inferTargetChatType` 处理应在搜索 peers / groups 之前完成的类别决策。
- 使用 `looksLikeId` 处理“将其视为显式 / 原生目标 id”的检查。
- 使用 `resolveTarget` 处理提供商专属规范化回退，而不是广义目录搜索。
- 将聊天 id、线程 id、JID、handle 和房间 id 等提供商原生 id 保留在 `target` 值或提供商专属参数中，而不要放进通用 SDK 字段里。

## 由配置支持的目录

从配置派生目录条目的插件，应将该逻辑保留在插件中，并复用来自
`openclaw/plugin-sdk/directory-runtime` 的共享辅助函数。

当某个渠道需要由配置支持的 peers / groups 时，请使用它，例如：

- 由 allowlist 驱动的私信 peers
- 已配置的 channel / group 映射
- 按账户划分的静态目录回退

`directory-runtime` 中的共享辅助函数只处理通用操作：

- 查询过滤
- 限制应用
- 去重 / 规范化辅助函数
- 构建 `ChannelDirectoryEntry[]`

渠道专属账户检查和 id 规范化应保留在插件实现中。

## 提供商目录

提供商插件可以通过
`registerProvider({ catalog: { run(...) { ... } } })` 定义用于推理的模型目录。

`catalog.run(...)` 返回的形态与 OpenClaw 写入
`models.providers` 的形态一致：

- `{ provider }` 用于一个 provider 条目
- `{ providers }` 用于多个 provider 条目

当插件拥有提供商专属 model id、base URL 默认值或受认证控制的模型元数据时，请使用 `catalog`。

`catalog.order` 控制插件目录相对于 OpenClaw
内置隐式提供商的合并时机：

- `simple`：普通 API key 或环境驱动的提供商
- `profile`：当存在认证配置档时出现的提供商
- `paired`：合成多个相关 provider 条目的提供商
- `late`：最后一轮，在其他隐式提供商之后

后出现的 provider 会在键冲突时获胜，因此插件可以有意用相同 provider id 覆盖内置 provider 条目。

兼容性：

- `discovery` 仍可作为遗留别名使用
- 如果同时注册了 `catalog` 和 `discovery`，OpenClaw 会使用 `catalog`

## 只读渠道检查

如果你的插件注册了一个渠道，请优先实现
`plugin.config.inspectAccount(cfg, accountId)`，并与 `resolveAccount(...)` 一起提供。

原因如下：

- `resolveAccount(...)` 是运行时路径。它可以假定凭证已经被完全具体化，并在缺失必需 secret 时快速失败。
- 只读命令路径，例如 `openclaw status`、`openclaw status --all`、
  `openclaw channels status`、`openclaw channels resolve`，以及 doctor / 配置
  修复流程，不应为了描述配置而必须具体化运行时凭证。

推荐的 `inspectAccount(...)` 行为：

- 只返回描述性的账户状态。
- 保留 `enabled` 和 `configured`。
- 在相关时包含凭证来源 / 状态字段，例如：
  - `tokenSource`、`tokenStatus`
  - `botTokenSource`、`botTokenStatus`
  - `appTokenSource`、`appTokenStatus`
  - `signingSecretSource`、`signingSecretStatus`
- 你不需要为了报告只读可用性而返回原始令牌值。返回 `tokenStatus: "available"`（以及匹配的 source 字段）就足以支持状态类命令。
- 当某个凭证通过 SecretRef 配置，但在当前命令路径中不可用时，请使用 `configured_unavailable`。

这样，只读命令就可以报告“已配置，但在当前命令路径中不可用”，而不是崩溃或误报该账户未配置。

## 包 pack

插件目录可以包含一个带有 `openclaw.extensions` 的 `package.json`：

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

每个条目都会变成一个插件。如果 pack 列出了多个扩展，插件 id
将变为 `name/<fileBase>`。

如果你的插件导入了 npm 依赖，请在该目录中安装它们，以便
`node_modules` 可用（`npm install` / `pnpm install`）。

安全护栏：每个 `openclaw.extensions` 条目在解析符号链接后都必须保留在插件目录内。任何逃出包目录的条目都会被拒绝。

安全说明：`openclaw plugins install` 会使用
`npm install --omit=dev --ignore-scripts` 安装插件依赖（无 lifecycle scripts，运行时无 dev dependencies）。请保持插件依赖树为“纯 JS / TS”，并避免需要 `postinstall` 构建的包。

可选：`openclaw.setupEntry` 可以指向一个轻量级、仅设置用的模块。
当 OpenClaw 需要为一个已禁用渠道插件提供设置接口，或者
当渠道插件已启用但仍未配置时，它会加载 `setupEntry`，
而不是完整插件入口。这样在你的主插件入口还会接线工具、钩子或其他仅运行时代码时，可以让启动和设置更轻量。

可选：`openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
可以让渠道插件在 Gateway 网关监听前的启动阶段，也改走同一个 `setupEntry` 路径，即使该渠道已经配置完成。

仅当 `setupEntry` 完全覆盖了启动前必须存在的接口时才应使用此选项。实际来说，这意味着 setup entry
必须注册启动所依赖的每一种渠道自有能力，例如：

- 渠道注册本身
- 在 Gateway 网关开始监听前必须可用的任何 HTTP 路由
- 在同一窗口期内必须存在的任何网关方法、工具或服务

如果你的完整入口仍拥有任何必需的启动能力，就不要启用这个标志。保持插件的默认行为，让 OpenClaw 在启动期间加载完整入口。

内置渠道还可以发布仅设置用途的契约接口辅助函数，以便核心在完整渠道运行时加载前进行查询。当前的设置提升接口为：

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

当核心需要在不加载完整插件入口的情况下，将遗留单账户渠道配置提升到 `channels.<id>.accounts.*` 时，会使用该接口。Matrix 是当前的内置示例：当已存在命名账户时，它只会将认证 / bootstrap 键移动到一个已命名的提升账户中，并且它可以保留一个已配置的非规范 default-account 键，而不是总是创建
`accounts.default`。

这些设置补丁适配器可使内置契约接口发现保持惰性。导入时间因此保持轻量；该提升接口仅在首次使用时加载，而不会在模块导入时重新进入内置渠道启动流程。

当这些启动接口包含 Gateway 网关 RPC 方法时，请将其保留在
插件专属前缀之下。核心管理命名空间（`config.*`、
`exec.approvals.*`、`wizard.*`、`update.*`）始终保留，并且即使某个插件请求更窄作用域，它们也始终解析为 `operator.admin`。

示例：

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### 渠道目录元数据

渠道插件可以通过 `openclaw.channel` 声明设置 / 发现元数据，并通过 `openclaw.install` 提供安装提示。这样可以让核心目录保持无数据化。

示例：

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

除了最小示例之外，有用的 `openclaw.channel` 字段还包括：

- `detailLabel`：用于更丰富目录 / 状态接口的次级标签
- `docsLabel`：覆盖文档链接的链接文本
- `preferOver`：此目录条目应优先于的低优先级插件 / 渠道 id
- `selectionDocsPrefix`、`selectionDocsOmitLabel`、`selectionExtras`：选择界面文案控制
- `markdownCapable`：将该渠道标记为支持 markdown，以用于出站格式决策
- `exposure.configured`：设置为 `false` 时，在已配置渠道列表接口中隐藏该渠道
- `exposure.setup`：设置为 `false` 时，在交互式设置 / 配置选择器中隐藏该渠道
- `exposure.docs`：将该渠道标记为内部 / 私有，以用于文档导航接口
- `showConfigured` / `showInSetup`：遗留别名，出于兼容性仍被接受；优先使用 `exposure`
- `quickstartAllowFrom`：让渠道接入标准快速开始 `allowFrom` 流程
- `forceAccountBinding`：即使只存在一个账户，也要求显式账户绑定
- `preferSessionLookupForAnnounceTarget`：在解析 announce 目标时优先使用会话查找

OpenClaw 还可以合并**外部渠道目录**（例如 MPM
注册表导出）。可将 JSON 文件放在以下任一位置：

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

或者将 `OPENCLAW_PLUGIN_CATALOG_PATHS`（或 `OPENCLAW_MPM_CATALOG_PATHS`）指向一个或多个 JSON 文件（以逗号 / 分号 / `PATH` 分隔）。每个文件应包含 `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`。解析器也接受 `"packages"` 或 `"plugins"` 作为 `"entries"` 键的遗留别名。

## 上下文引擎插件

上下文引擎插件拥有会话上下文编排，用于摄取、组装和压缩。可通过
`api.registerContextEngine(id, factory)` 从你的插件中注册它们，然后通过
`plugins.slots.contextEngine` 选择活动引擎。

当你的插件需要替换或扩展默认上下文流水线，而不仅仅是增加 memory 搜索或钩子时，请使用此方式。

```ts
export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

如果你的引擎**不**拥有压缩算法，请保持 `compact()`
已实现，并显式委托给它：

```ts
import { delegateCompactionToRuntime } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## 添加新能力

当插件需要当前 API 无法覆盖的行为时，不要通过私有深入访问来绕过插件系统。请添加缺失的能力。

推荐顺序：

1. 定义核心契约
   确定核心应拥有哪些共享行为：策略、回退、配置合并、
   生命周期、面向渠道的语义，以及运行时辅助函数形态。
2. 添加类型化插件注册 / 运行时接口
   以最小但有用的类型化能力接口扩展 `OpenClawPluginApi` 和 / 或 `api.runtime`。
3. 接线核心 + 渠道 / 功能消费者
   渠道和功能插件应通过核心消费新能力，
   而不是直接导入某个厂商实现。
4. 注册厂商实现
   然后由厂商插件针对该能力注册其后端。
5. 添加契约覆盖
   添加测试，使归属和注册形态在长期内保持明确。

这正是 OpenClaw 能保持有主张、却不会被硬编码到某个提供商世界观中的方式。具体文件清单和完整示例，请参见[能力扩展手册](/zh-CN/plugins/architecture)。

### 能力清单

当你添加一个新能力时，通常应该同时触及以下接口：

- `src/<capability>/types.ts` 中的核心契约类型
- `src/<capability>/runtime.ts` 中的核心运行器 / 运行时辅助函数
- `src/plugins/types.ts` 中的插件 API 注册接口
- `src/plugins/registry.ts` 中的插件注册表接线
- 当功能 / 渠道插件需要消费该能力时，位于 `src/plugins/runtime/*` 中的插件运行时暴露
- `src/test-utils/plugin-registration.ts` 中的捕获 / 测试辅助函数
- `src/plugins/contracts/registry.ts` 中的归属 / 契约断言
- `docs/` 中面向操作员 / 插件的文档

如果这些接口中缺少某一项，通常说明该能力尚未完全集成。

### 能力模板

最小模式：

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

契约测试模式：

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

这样规则就很简单：

- 核心拥有能力契约 + 编排
- 厂商插件拥有厂商实现
- 功能 / 渠道插件消费运行时辅助函数
- 契约测试使归属保持明确
