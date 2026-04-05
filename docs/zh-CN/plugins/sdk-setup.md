---
read_when:
    - 你正在为插件添加设置向导
    - 你需要理解 `setup-entry.ts` 与 `index.ts` 的区别
    - 你正在定义插件配置模式或 `package.json` 中的 openclaw 元数据
sidebarTitle: Setup and Config
summary: 设置向导、`setup-entry.ts`、配置模式，以及 `package.json` 元数据
title: 插件设置和配置
x-i18n:
    generated_at: "2026-04-05T22:30:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: eac2586516d27bcd94cc4c259fe6274c792b3f9938c7ddd6dbf04a6dbb988dc9
    source_path: plugins/sdk-setup.md
    workflow: 15
---

# 插件设置和配置

有关插件打包（`package.json` 元数据）、清单
（`openclaw.plugin.json`）、设置入口和配置模式的参考文档。

<Tip>
  **想看操作演练？** 操作指南会在具体上下文中介绍打包：
  [渠道插件](/zh-CN/plugins/sdk-channel-plugins#step-1-package-and-manifest) 和
  [提供商插件](/zh-CN/plugins/sdk-provider-plugins#step-1-package-and-manifest)。
</Tip>

## 包元数据

你的 `package.json` 需要一个 `openclaw` 字段，用来告诉插件系统
你的插件提供了什么：

**渠道插件：**

```json
{
  "name": "@myorg/openclaw-my-channel",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "blurb": "Short description of the channel."
    }
  }
}
```

**提供商插件 / ClawHub 发布基线：**

```json openclaw-clawhub-package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

如果你要将插件作为外部插件发布到 ClawHub，这些 `compat` 和 `build`
字段是必填的。规范的发布代码片段位于
`docs/snippets/plugin-publish/`。

### `openclaw` 字段

| 字段         | 类型       | 说明                                                                                                  |
| ------------ | ---------- | ----------------------------------------------------------------------------------------------------- |
| `extensions` | `string[]` | 入口点文件（相对于包根目录）                                                                          |
| `setupEntry` | `string`   | 仅用于设置的轻量入口（可选）                                                                          |
| `channel`    | `object`   | 用于设置、选择器、快速开始和状态界面的渠道目录元数据                                                  |
| `providers`  | `string[]` | 该插件注册的提供商 id                                                                                 |
| `install`    | `object`   | 安装提示：`npmSpec`、`localPath`、`defaultChoice`、`minHostVersion`、`allowInvalidConfigRecovery` |
| `startup`    | `object`   | 启动行为标记                                                                                          |

### `openclaw.channel`

`openclaw.channel` 是用于渠道发现和设置界面的轻量包元数据，
可在运行时加载前使用。

| 字段                                   | 类型       | 含义                                                                          |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | 规范的渠道 id。                                                               |
| `label`                                | `string`   | 主要渠道标签。                                                                |
| `selectionLabel`                       | `string`   | 当需要不同于 `label` 时，用于选择器/设置的标签。                              |
| `detailLabel`                          | `string`   | 用于更丰富渠道目录和状态界面的次级详情标签。                                  |
| `docsPath`                             | `string`   | 用于设置和选择链接的文档路径。                                                |
| `docsLabel`                            | `string`   | 当需要不同于渠道 id 时，用于文档链接的覆盖标签。                              |
| `blurb`                                | `string`   | 简短的新手引导/目录描述。                                                     |
| `order`                                | `number`   | 渠道目录中的排序顺序。                                                        |
| `aliases`                              | `string[]` | 用于渠道选择的额外查找别名。                                                  |
| `preferOver`                           | `string[]` | 该渠道应高于其优先级的较低插件/渠道 id。                                      |
| `systemImage`                          | `string`   | 渠道 UI 目录中可选的图标/系统图像名称。                                       |
| `selectionDocsPrefix`                  | `string`   | 选择界面中文档链接前的前缀文本。                                              |
| `selectionDocsOmitLabel`               | `boolean`  | 在选择文案中直接显示文档路径，而不是带标签的文档链接。                        |
| `selectionExtras`                      | `string[]` | 附加到选择文案中的额外短文本。                                                |
| `markdownCapable`                      | `boolean`  | 将该渠道标记为支持 Markdown，以便进行出站格式决策。                           |
| `exposure`                             | `object`   | 用于设置、已配置列表和文档界面的渠道可见性控制。                              |
| `quickstartAllowFrom`                  | `boolean`  | 让该渠道启用标准快速开始 `allowFrom` 设置流程。                               |
| `forceAccountBinding`                  | `boolean`  | 即使仅存在一个账户，也要求显式账户绑定。                                      |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | 为该渠道解析公告目标时，优先使用会话查找。                                    |

示例：

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "selectionLabel": "My Channel (self-hosted)",
      "detailLabel": "My Channel Bot",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-based self-hosted chat integration.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guide:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` 支持：

- `configured`：在已配置/状态类列表界面中包含该渠道
- `setup`：在交互式设置/配置选择器中包含该渠道
- `docs`：将该渠道标记为在文档/导航界面中面向公开

`showConfigured` 和 `showInSetup` 仍作为旧版别名受支持。优先使用
`exposure`。

### `openclaw.install`

`openclaw.install` 是包元数据，不是清单元数据。

| 字段                         | 类型                 | 含义                                                                            |
| ---------------------------- | -------------------- | ------------------------------------------------------------------------------- |
| `npmSpec`                    | `string`             | 用于安装/更新流程的规范 npm 规格。                                              |
| `localPath`                  | `string`             | 本地开发或内置安装路径。                                                        |
| `defaultChoice`              | `"npm"` \| `"local"` | 当两者都可用时，优先的安装来源。                                                |
| `minHostVersion`             | `string`             | 最低支持的 OpenClaw 版本，格式为 `>=x.y.z`。                                    |
| `allowInvalidConfigRecovery` | `boolean`            | 允许内置插件重装流程从特定的过期配置故障中恢复。                                |

如果设置了 `minHostVersion`，安装和清单注册表加载都会强制执行
该值。较旧的宿主会跳过该插件；无效的版本字符串会被拒绝。

`allowInvalidConfigRecovery` 并不是对损坏配置的通用绕过方式。它只用于
狭义的内置插件恢复场景，因此重装/设置可以修复已知的升级遗留问题，比如缺失的内置插件路径或同一插件对应的过期
`channels.<id>` 条目。如果配置因不相关原因损坏，安装仍会默认拒绝，并提示操作员运行 `openclaw doctor --fix`。

### 延迟完整加载

渠道插件可以通过以下方式选择延迟加载：

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

启用后，即使是已配置的渠道，OpenClaw 在监听前的启动阶段也只会加载
`setupEntry`。完整入口会在 Gateway 网关开始监听后再加载。

<Warning>
  仅当你的 `setupEntry` 在 Gateway 网关开始监听前注册了所需的一切
  （渠道注册、HTTP 路由、Gateway 网关方法）时，才应启用延迟加载。如果完整入口拥有必需的启动能力，请保留默认行为。
</Warning>

如果你的设置/完整入口会注册 Gateway 网关 RPC 方法，请将它们保留在
插件专属前缀下。保留的核心管理员命名空间（`config.*`、
`exec.approvals.*`、`wizard.*`、`update.*`）仍由核心拥有，并始终解析为
`operator.admin`。

## 插件清单

每个原生插件都必须在包根目录中提供一个 `openclaw.plugin.json`。
OpenClaw 使用它在不执行插件代码的情况下验证配置。

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

对于渠道插件，请添加 `kind` 和 `channels`：

```json
{
  "id": "my-channel",
  "kind": "channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

即使插件没有任何配置，也必须提供一个模式。空模式是有效的：

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

完整模式参考请见 [插件清单](/zh-CN/plugins/manifest)。

## ClawHub 发布

对于插件包，请使用面向包的 ClawHub 专用命令：

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

旧版仅面向 Skills 的发布别名适用于 Skills。插件包应始终使用
`clawhub package publish`。

## 设置入口

`setup-entry.ts` 文件是 `index.ts` 的轻量替代项，
当 OpenClaw 只需要设置界面时（新手引导、配置修复、
已禁用渠道检查），会加载它。

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

这样可避免在设置流程中加载沉重的运行时代码（加密库、CLI 注册、
后台服务）。

**OpenClaw 在以下情况下会使用 `setupEntry` 而不是完整入口：**

- 渠道已禁用，但需要设置/新手引导界面
- 渠道已启用，但尚未配置
- 已启用延迟加载（`deferConfiguredChannelFullLoadUntilAfterListen`）

**`setupEntry` 必须注册的内容：**

- 渠道插件对象（通过 `defineSetupPluginEntry`）
- 任何在 Gateway 网关监听前所需的 HTTP 路由
- 启动期间所需的任何 Gateway 网关方法

这些启动期的 Gateway 网关方法仍应避免使用保留的核心管理员
命名空间，例如 `config.*` 或 `update.*`。

**`setupEntry` 不应包含的内容：**

- CLI 注册
- 后台服务
- 沉重的运行时导入（加密、SDK）
- 仅在启动后才需要的 Gateway 网关方法

### 更窄的设置辅助导入

对于只涉及设置的高频路径，如果你只需要设置界面的一部分，请优先使用更窄的设置辅助接缝，而不是更宽泛的
`plugin-sdk/setup` 总入口：

| 导入路径                           | 适用场景                                                                                  | 关键导出                                                                                                                                                                                                                                                                                     |
| ---------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime`         | 在 `setupEntry` / 延迟渠道启动中仍可用的设置期运行时辅助工具                              | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-adapter-runtime` | 具备环境感知能力的账户设置适配器                                                          | `createEnvPatchedAccountSetupAdapter`                                                                                                                                                                                                                                                        |
| `plugin-sdk/setup-tools`           | 设置/安装 CLI/归档/文档辅助工具                                                           | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                              |

当你需要完整的共享设置工具箱时，请使用更宽泛的
`plugin-sdk/setup` 接缝，其中包括配置补丁辅助工具，例如
`moveSingleAccountChannelSectionToDefaultAccount(...)`。

这些设置补丁适配器在导入时仍适合高频路径使用。它们针对内置
单账户提升合同界面查找采用惰性加载，因此导入
`plugin-sdk/setup-runtime` 不会在适配器真正使用前，主动加载内置合同界面发现逻辑。

### 渠道自有的单账户提升

当某个渠道从单账户顶层配置升级到
`channels.<id>.accounts.*` 时，默认的共享行为是将提升后的
账户作用域值移动到 `accounts.default` 中。

内置渠道可以通过其设置合同界面来收窄或覆盖这一提升行为：

- `singleAccountKeysToMove`：应移动到提升后账户中的额外顶层键
- `namedAccountPromotionKeys`：当已存在命名账户时，仅这些
  键会移动到提升后的账户中；共享的策略/投递键会保留在渠道根部
- `resolveSingleAccountPromotionTarget(...)`：选择哪个现有账户
  接收提升后的值

Matrix 是当前的内置示例。如果恰好已经存在一个命名的 Matrix 账户，
或者 `defaultAccount` 指向现有的非规范键（例如 `Ops`），提升过程会保留该账户，而不是创建新的
`accounts.default` 条目。

## 配置模式

插件配置会根据清单中的 JSON Schema 进行验证。用户通过以下方式
配置插件：

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

在注册期间，你的插件会通过 `api.pluginConfig` 接收此配置。

对于渠道专属配置，请改用渠道配置段：

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### 构建渠道配置模式

使用 `openclaw/plugin-sdk/core` 中的 `buildChannelConfigSchema`，将
Zod 模式转换为 OpenClaw 用于验证的 `ChannelConfigSchema`
包装器：

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/core";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

## 设置向导

渠道插件可以为 `openclaw onboard` 提供交互式设置向导。
该向导是 `ChannelPlugin` 上的一个 `ChannelSetupWizard` 对象：

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` 类型支持 `credentials`、`textInputs`、
`dmPolicy`、`allowFrom`、`groupAccess`、`prepare`、`finalize` 等等。
完整示例请参见内置插件包（例如 Discord 插件的 `src/channel.setup.ts`）。

对于只需要标准
`note -> prompt -> parse -> merge -> patch` 流程的私信白名单提示，请优先使用
`openclaw/plugin-sdk/setup` 中的共享设置辅助工具：`createPromptParsedAllowFromForAccount(...)`、
`createTopLevelChannelParsedAllowFromPrompt(...)` 和
`createNestedChannelParsedAllowFromPrompt(...)`。

对于只在标签、分数和可选附加行上有所不同的渠道设置状态块，请优先使用
`openclaw/plugin-sdk/setup` 中的 `createStandardChannelSetupStatus(...)`，
而不是在每个插件中手动编写相同的 `status` 对象。

对于只应在特定上下文中出现的可选设置界面，请使用
`openclaw/plugin-sdk/channel-setup` 中的
`createOptionalChannelSetupSurface`：

```typescript
import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

const setupSurface = createOptionalChannelSetupSurface({
  channel: "my-channel",
  label: "My Channel",
  npmSpec: "@myorg/openclaw-my-channel",
  docsPath: "/channels/my-channel",
});
// Returns { setupAdapter, setupWizard }
```

`plugin-sdk/channel-setup` 还暴露了更底层的
`createOptionalChannelSetupAdapter(...)` 和
`createOptionalChannelSetupWizard(...)` 构建器，适用于你只需要
该可选安装界面其中一半的情况。

生成的可选适配器/向导在真实配置写入时会默认拒绝。它们会在
`validateInput`、`applyAccountConfig` 和 `finalize` 之间复用同一条需要安装的消息，并在设置了 `docsPath` 时追加文档链接。

对于基于二进制程序的设置 UI，请优先使用共享的委托辅助工具，而不是在每个渠道中复制相同的二进制/状态胶水逻辑：

- `createDetectedBinaryStatus(...)`：适用于仅在标签、
  提示、分数和二进制检测上有所变化的状态块
- `createCliPathTextInput(...)`：适用于基于路径的文本输入
- `createDelegatedSetupWizardStatusResolvers(...)`、
  `createDelegatedPrepare(...)`、`createDelegatedFinalize(...)` 和
  `createDelegatedResolveConfigured(...)`：适用于 `setupEntry` 需要
  惰性转发到更重型完整向导的场景
- `createDelegatedTextInputShouldPrompt(...)`：适用于 `setupEntry` 只需要
  委托一个 `textInputs[*].shouldPrompt` 判断的场景

## 发布和安装

**外部插件：** 发布到 [ClawHub](/zh-CN/tools/clawhub) 或 npm，然后安装：

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

OpenClaw 会先尝试 ClawHub，然后自动回退到 npm。你也可以显式
强制使用 ClawHub：

```bash
openclaw plugins install clawhub:@myorg/openclaw-my-plugin   # 仅 ClawHub
```

没有对应的 `npm:` 覆盖方式。当你希望在 ClawHub 回退后走 npm 路径时，
请使用普通的 npm 包规格：

```bash
openclaw plugins install @myorg/openclaw-my-plugin
```

**仓库内插件：** 放置在内置插件工作区树下，构建期间会自动发现。

**用户可以安装：**

```bash
openclaw plugins install <package-name>
```

<Info>
  对于来自 npm 的安装，`openclaw plugins install` 会运行
  `npm install --ignore-scripts`（不执行生命周期脚本）。请保持插件依赖树为纯 JS/TS，并避免使用需要 `postinstall` 构建的包。
</Info>

## 相关内容

- [插件入口点](/zh-CN/plugins/sdk-entrypoints) -- `definePluginEntry` 和 `defineChannelPluginEntry`
- [插件清单](/zh-CN/plugins/manifest) -- 完整清单模式参考
- [构建插件](/zh-CN/plugins/building-plugins) -- 分步式入门指南
