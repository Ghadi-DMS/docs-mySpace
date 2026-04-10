---
read_when:
    - 实现或更新 Gateway 网关 WS 客户端
    - 调试协议不匹配或连接失败
    - 重新生成协议模式 / 模型
summary: Gateway 网关 WebSocket 协议：握手、帧、版本控制
title: Gateway 网关协议
x-i18n:
    generated_at: "2026-04-10T06:30:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 83c820c46d4803d571c770468fd6782619eaa1dca253e156e8087dec735c127f
    source_path: gateway/protocol.md
    workflow: 15
---

# Gateway 网关协议（WebSocket）

Gateway 网关 WS 协议是 OpenClaw 的**单一控制平面 + 节点传输层**。所有客户端（CLI、Web UI、macOS 应用、iOS/Android 节点、无头节点）都通过 WebSocket 连接，并在握手时声明其**角色** + **作用域**。

## 传输

- WebSocket，使用带有 JSON 载荷的文本帧。
- 第一帧**必须**是 `connect` 请求。

## 握手（connect）

Gateway 网关 → 客户端（连接前质询）：

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

客户端 → Gateway 网关：

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway 网关 → 客户端：

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
}
```

当签发设备令牌时，`hello-ok` 还会包含：

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

在受信任的引导移交流程中，`hello-ok.auth` 还可能在 `deviceTokens` 中包含额外的有界角色条目：

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

对于内置的节点 / operator 引导流程，主节点令牌仍保持 `scopes: []`，而任何移交的 operator 令牌仍受限于引导 operator 允许列表（`operator.approvals`、`operator.read`、`operator.talk.secrets`、`operator.write`）。引导作用域检查仍保持角色前缀规则：operator 条目只满足 operator 请求，而非 operator 角色仍需要其自身角色前缀下的作用域。

### 节点示例

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## 帧格式

- **请求**：`{type:"req", id, method, params}`
- **响应**：`{type:"res", id, ok, payload|error}`
- **事件**：`{type:"event", event, payload, seq?, stateVersion?}`

有副作用的方法需要**幂等键**（参见 schema）。

## 角色 + 作用域

### 角色

- `operator` = 控制平面客户端（CLI / UI / 自动化）。
- `node` = 能力宿主（camera / screen / canvas / system.run）。

### 作用域（operator）

常见作用域：

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

带有 `includeSecrets: true` 的 `talk.config` 需要 `operator.talk.secrets`（或 `operator.admin`）。

由插件注册的 Gateway 网关 RPC 方法可以请求它们自己的 operator 作用域，但保留的核心管理前缀（`config.*`、`exec.approvals.*`、`wizard.*`、`update.*`）始终解析为 `operator.admin`。

方法作用域只是第一道门槛。通过 `chat.send` 触达的一些斜杠命令还会在此基础上应用更严格的命令级检查。例如，持久化的 `/config set` 和 `/config unset` 写入需要 `operator.admin`。

`node.pair.approve` 在基础方法作用域之上还带有额外的审批时作用域检查：

- 无命令请求：`operator.pairing`
- 带有非 exec 节点命令的请求：`operator.pairing` + `operator.write`
- 包含 `system.run`、`system.run.prepare` 或 `system.which` 的请求：`operator.pairing` + `operator.admin`

### Caps / commands / permissions（node）

节点在连接时声明能力声明：

- `caps`：高层能力类别。
- `commands`：用于 invoke 的命令允许列表。
- `permissions`：细粒度开关（例如 `screen.record`、`camera.capture`）。

Gateway 网关将这些视为**声明**，并在服务端强制执行允许列表。

## 在线状态

- `system-presence` 返回按设备身份键控的条目。
- 在线状态条目包含 `deviceId`、`roles` 和 `scopes`，这样 UI 就可以在设备同时以**operator** 和 **node** 身份连接时，仍只显示单行。

## 常见 RPC 方法族

本页不是自动生成的完整导出，但公开的 WS 接口比上面的握手 / 认证示例更广。这些是 Gateway 网关当前公开的主要方法族。

`hello-ok.features.methods` 是基于 `src/gateway/server-methods-list.ts` 和已加载的插件 / 渠道方法导出构建的保守发现列表。请将其视为功能发现，而不是 `src/gateway/server-methods/*.ts` 中每个可调用辅助方法的自动生成导出。

### 系统和身份

- `health` 返回缓存的或新探测的网关健康状态快照。
- `status` 返回 `/status` 风格的网关摘要；敏感字段仅对具有 admin 作用域的 operator 客户端包含。
- `gateway.identity.get` 返回 relay 和配对流程使用的网关设备身份。
- `system-presence` 返回当前已连接 operator / node 设备的在线状态快照。
- `system-event` 追加一条系统事件，并可更新 / 广播在线状态上下文。
- `last-heartbeat` 返回最新持久化的心跳事件。
- `set-heartbeats` 切换网关上的心跳处理。

### 模型和用量

- `models.list` 返回运行时允许的模型目录。
- `usage.status` 返回提供商用量窗口 / 剩余额度摘要。
- `usage.cost` 返回某个日期范围内聚合的成本用量摘要。
- `doctor.memory.status` 返回活动默认智能体工作区的向量记忆 / 嵌入就绪状态。
- `sessions.usage` 返回按会话的用量摘要。
- `sessions.usage.timeseries` 返回单个会话的时间序列用量。
- `sessions.usage.logs` 返回单个会话的用量日志条目。

### 渠道和登录辅助方法

- `channels.status` 返回内置 + 内置插件渠道 / 插件状态摘要。
- `channels.logout` 在渠道支持退出登录时，登出特定渠道 / 账号。
- `web.login.start` 为当前支持 QR 的 Web 渠道提供商启动 QR / Web 登录流程。
- `web.login.wait` 等待该 QR / Web 登录流程完成，并在成功后启动渠道。
- `push.test` 向已注册的 iOS 节点发送一条测试 APNs 推送。
- `voicewake.get` 返回已存储的唤醒词触发器。
- `voicewake.set` 更新唤醒词触发器并广播更改。

### 消息和日志

- `send` 是直接的出站投递 RPC，用于在 chat runner 之外，按渠道 / 账号 / 线程目标发送消息。
- `logs.tail` 返回已配置的网关文件日志尾部，支持 cursor / limit 和最大字节数控制。

### Talk 和 TTS

- `talk.config` 返回生效中的 Talk 配置载荷；`includeSecrets` 需要 `operator.talk.secrets`（或 `operator.admin`）。
- `talk.mode` 为 WebChat / Control UI 客户端设置 / 广播当前 Talk 模式状态。
- `talk.speak` 通过当前活动的 Talk 语音提供商合成语音。
- `tts.status` 返回 TTS 启用状态、当前提供商、回退提供商和提供商配置状态。
- `tts.providers` 返回可见的 TTS 提供商清单。
- `tts.enable` 和 `tts.disable` 切换 TTS 偏好设置状态。
- `tts.setProvider` 更新首选 TTS 提供商。
- `tts.convert` 运行一次性文本转语音转换。

### 密钥、配置、更新和向导

- `secrets.reload` 重新解析活动中的 `SecretRef`，并仅在完全成功时切换运行时密钥状态。
- `secrets.resolve` 为特定命令 / 目标集合解析命令目标密钥分配。
- `config.get` 返回当前配置快照和哈希值。
- `config.set` 写入经过验证的配置载荷。
- `config.patch` 合并部分配置更新。
- `config.apply` 验证并替换完整配置载荷。
- `config.schema` 返回 Control UI 和 CLI 工具使用的实时配置 schema 载荷：schema、`uiHints`、版本和生成元数据，包括运行时能够加载时的插件 + 渠道 schema 元数据。该 schema 包含从 UI 使用的同一套标签和帮助文本派生而来的字段 `title` / `description` 元数据，包括嵌套对象、通配符、数组项，以及在存在匹配字段文档时的 `anyOf` / `oneOf` / `allOf` 组合分支。
- `config.schema.lookup` 返回某个配置路径的路径范围查找载荷：标准化路径、浅层 schema 节点、匹配的提示 + `hintPath`，以及供 UI / CLI 向下钻取的直接子项摘要。
  - 查找 schema 节点保留面向用户的文档和常见验证字段：`title`、`description`、`type`、`enum`、`const`、`format`、`pattern`、数值 / 字符串 / 数组 / 对象边界，以及 `additionalProperties`、`deprecated`、`readOnly`、`writeOnly` 等布尔标志。
  - 子项摘要公开 `key`、标准化 `path`、`type`、`required`、`hasChildren`，以及匹配的 `hint` / `hintPath`。
- `update.run` 运行网关更新流程，并仅在更新本身成功时安排重启。
- `wizard.start`、`wizard.next`、`wizard.status` 和 `wizard.cancel` 通过 WS RPC 暴露新手引导向导。

### 现有主要方法族

#### 智能体和工作区辅助方法

- `agents.list` 返回已配置的智能体条目。
- `agents.create`、`agents.update` 和 `agents.delete` 管理智能体记录和工作区接线。
- `agents.files.list`、`agents.files.get` 和 `agents.files.set` 管理为某个智能体公开的引导工作区文件。
- `agent.identity.get` 返回某个智能体或会话的生效助手身份。
- `agent.wait` 等待某次运行完成，并在可用时返回终态快照。

#### 会话控制

- `sessions.list` 返回当前会话索引。
- `sessions.subscribe` 和 `sessions.unsubscribe` 为当前 WS 客户端切换会话变更事件订阅。
- `sessions.messages.subscribe` 和 `sessions.messages.unsubscribe` 为单个会话切换转录 / 消息事件订阅。
- `sessions.preview` 返回特定会话键的有界转录预览。
- `sessions.resolve` 解析或规范化会话目标。
- `sessions.create` 创建新的会话条目。
- `sessions.send` 向现有会话发送消息。
- `sessions.steer` 是活动会话的中断并引导变体。
- `sessions.abort` 中止某个会话的活动工作。
- `sessions.patch` 更新会话元数据 / 覆盖项。
- `sessions.reset`、`sessions.delete` 和 `sessions.compact` 执行会话维护。
- `sessions.get` 返回完整存储的会话行。
- chat 执行仍使用 `chat.history`、`chat.send`、`chat.abort` 和 `chat.inject`。
- 面向 UI 客户端时，`chat.history` 会进行显示规范化：内联指令标签会从可见文本中移除，纯文本工具调用 XML 载荷（包括 `<tool_call>...</tool_call>`、`<function_call>...</function_call>`、`<tool_calls>...</tool_calls>`、`<function_calls>...</function_calls>` 以及截断的工具调用块）和泄漏的 ASCII / 全角模型控制令牌会被移除，纯静默令牌的 assistant 行（例如精确的 `NO_REPLY` / `no_reply`）会被省略，过大的行可被占位符替换。

#### 设备配对和设备令牌

- `device.pair.list` 返回待处理和已批准的已配对设备。
- `device.pair.approve`、`device.pair.reject` 和 `device.pair.remove` 管理设备配对记录。
- `device.token.rotate` 在已批准的角色和作用域边界内轮换已配对设备令牌。
- `device.token.revoke` 撤销已配对设备令牌。

#### 节点配对、调用和待处理工作

- `node.pair.request`、`node.pair.list`、`node.pair.approve`、`node.pair.reject` 和 `node.pair.verify` 涵盖节点配对和引导验证。
- `node.list` 和 `node.describe` 返回已知 / 已连接的节点状态。
- `node.rename` 更新已配对节点标签。
- `node.invoke` 将命令转发到已连接节点。
- `node.invoke.result` 返回 invoke 请求的结果。
- `node.event` 将节点源事件回传到网关。
- `node.canvas.capability.refresh` 刷新具作用域的 canvas 能力令牌。
- `node.pending.pull` 和 `node.pending.ack` 是已连接节点队列 API。
- `node.pending.enqueue` 和 `node.pending.drain` 管理离线 / 断开连接节点的持久待处理工作。

#### 审批方法族

- `exec.approval.request`、`exec.approval.get`、`exec.approval.list` 和 `exec.approval.resolve` 涵盖一次性 exec 审批请求，以及待处理审批的查找 / 重放。
- `exec.approval.waitDecision` 等待单个待处理 exec 审批，并返回最终决定（或超时时返回 `null`）。
- `exec.approvals.get` 和 `exec.approvals.set` 管理 Gateway 网关 exec 审批策略快照。
- `exec.approvals.node.get` 和 `exec.approvals.node.set` 通过节点 relay 命令管理节点本地 exec 审批策略。
- `plugin.approval.request`、`plugin.approval.list`、`plugin.approval.waitDecision` 和 `plugin.approval.resolve` 涵盖插件定义的审批流程。

#### 其他主要方法族

- 自动化：
  - `wake` 调度立即或下一次心跳时的唤醒文本注入
  - `cron.list`、`cron.status`、`cron.add`、`cron.update`、`cron.remove`、`cron.run`、`cron.runs`
- Skills / 工具：`commands.list`、`skills.*`、`tools.catalog`、`tools.effective`

### 常见事件族

- `chat`：UI chat 更新，例如 `chat.inject` 和其他仅转录的 chat 事件。
- `session.message` 和 `session.tool`：已订阅会话的转录 / 事件流更新。
- `sessions.changed`：会话索引或元数据已变更。
- `presence`：系统在线状态快照更新。
- `tick`：周期性 keepalive / 存活事件。
- `health`：网关健康状态快照更新。
- `heartbeat`：心跳事件流更新。
- `cron`：cron 运行 / 作业变更事件。
- `shutdown`：Gateway 网关关闭通知。
- `node.pair.requested` / `node.pair.resolved`：节点配对生命周期。
- `node.invoke.request`：节点 invoke 请求广播。
- `device.pair.requested` / `device.pair.resolved`：已配对设备生命周期。
- `voicewake.changed`：唤醒词触发器配置已更改。
- `exec.approval.requested` / `exec.approval.resolved`：exec 审批生命周期。
- `plugin.approval.requested` / `plugin.approval.resolved`：插件审批生命周期。

### 节点辅助方法

- 节点可以调用 `skills.bins` 以获取当前 Skills 可执行文件列表，用于自动允许检查。

### Operator 辅助方法

- Operator 可以调用 `commands.list`（`operator.read`）来获取某个智能体的运行时命令清单。
  - `agentId` 是可选的；省略它可读取默认智能体工作区。
  - `scope` 控制主 `name` 针对哪个表面：
    - `text` 返回不带前导 `/` 的主文本命令令牌
    - `native` 和默认的 `both` 路径会在可用时返回提供商感知的原生命名
  - `textAliases` 包含精确的斜杠别名，例如 `/model` 和 `/m`。
  - `nativeName` 包含提供商感知的原生命令名（如果存在）。
  - `provider` 是可选的，仅影响原生命名以及原生插件命令可用性。
  - `includeArgs=false` 会从响应中省略序列化参数元数据。
- Operator 可以调用 `tools.catalog`（`operator.read`）来获取某个智能体的运行时工具目录。响应包含分组后的工具和来源元数据：
  - `source`：`core` 或 `plugin`
  - `pluginId`：当 `source="plugin"` 时的插件所有者
  - `optional`：插件工具是否为可选
- Operator 可以调用 `tools.effective`（`operator.read`）来获取某个会话的运行时生效工具清单。
  - 必须提供 `sessionKey`。
  - Gateway 网关会从会话中在服务端推导受信任的运行时上下文，而不是接受调用方提供的 auth 或投递上下文。
  - 响应按会话作用域返回，并反映当前活动对话此刻可以使用的内容，包括 core、插件和渠道工具。
- Operator 可以调用 `skills.status`（`operator.read`）来获取某个智能体可见的 Skills 清单。
  - `agentId` 是可选的；省略它可读取默认智能体工作区。
  - 响应包含资格、缺失要求、配置检查以及已脱敏的安装选项，不会暴露原始密钥值。
- Operator 可以调用 `skills.search` 和 `skills.detail`（`operator.read`）来获取 ClawHub 发现元数据。
- Operator 可以通过两种模式调用 `skills.install`（`operator.admin`）：
  - ClawHub 模式：`{ source: "clawhub", slug, version?, force? }` 将一个 skill 文件夹安装到默认智能体工作区的 `skills/` 目录。
  - Gateway 网关安装器模式：`{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }` 在 Gateway 网关主机上运行已声明的 `metadata.openclaw.install` 操作。
- Operator 可以通过两种模式调用 `skills.update`（`operator.admin`）：
  - ClawHub 模式更新一个已跟踪的 slug，或更新默认智能体工作区中的所有已跟踪 ClawHub 安装。
  - 配置模式会修补 `skills.entries.<skillKey>` 值，例如 `enabled`、`apiKey` 和 `env`。

## Exec 审批

- 当 exec 请求需要审批时，网关会广播 `exec.approval.requested`。
- Operator 客户端通过调用 `exec.approval.resolve` 来完成处理（需要 `operator.approvals` 作用域）。
- 对于 `host=node`，`exec.approval.request` 必须包含 `systemRunPlan`（规范化的 `argv` / `cwd` / `rawCommand` / 会话元数据）。缺少 `systemRunPlan` 的请求会被拒绝。
- 审批通过后，被转发的 `node.invoke system.run` 调用会复用该规范化的 `systemRunPlan` 作为权威的命令 / `cwd` / 会话上下文。
- 如果调用方在 prepare 和最终获批的 `system.run` 转发之间修改了 `command`、`rawCommand`、`cwd`、`agentId` 或 `sessionKey`，网关会拒绝执行，而不是信任已修改的载荷。

## 智能体投递回退

- `agent` 请求可以包含 `deliver=true` 以请求出站投递。
- `bestEffortDeliver=false` 保持严格行为：未解析或仅内部的投递目标会返回 `INVALID_REQUEST`。
- `bestEffortDeliver=true` 允许在无法解析外部可投递路由时回退为仅会话执行（例如内部 / webchat 会话或多渠道配置不明确的情况）。

## 版本控制

- `PROTOCOL_VERSION` 位于 `src/gateway/protocol/schema.ts`。
- 客户端发送 `minProtocol` + `maxProtocol`；服务器会拒绝不匹配的情况。
- Schemas + models 从 TypeBox 定义生成：
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## 认证

- 共享密钥网关认证使用 `connect.params.auth.token` 或 `connect.params.auth.password`，具体取决于已配置的认证模式。
- 带身份的模式，例如 Tailscale Serve（`gateway.auth.allowTailscale: true`）或非 loopback 的 `gateway.auth.mode: "trusted-proxy"`，会从请求头而不是 `connect.params.auth.*` 满足连接认证检查。
- 私有入口的 `gateway.auth.mode: "none"` 会完全跳过共享密钥连接认证；不要在公共 / 不受信任的入口上暴露该模式。
- 配对完成后，Gateway 网关会签发一个**设备令牌**，其作用域受限于连接角色 + 作用域。它会在 `hello-ok.auth.deviceToken` 中返回，客户端应将其持久化以供未来连接使用。
- 客户端应在任何成功连接后持久化主 `hello-ok.auth.deviceToken`。
- 使用该**已存储**设备令牌重新连接时，也应复用为该令牌存储的已批准作用域集合。这样可以保留先前已授予的读取 / 探测 / 状态访问，并避免重新连接时静默收缩为更窄的隐式仅 admin 作用域。
- 常规连接认证优先级是：显式共享 token / password 优先，然后是显式 `deviceToken`，再然后是存储的每设备令牌，最后是引导令牌。
- 额外的 `hello-ok.auth.deviceTokens` 条目是引导移交令牌。只有当连接使用的是受信任传输（例如 `wss://` 或 loopback / 本地配对）上的引导认证时，才应持久化它们。
- 如果客户端提供了**显式** `deviceToken` 或显式 `scopes`，则该调用方请求的作用域集合仍是权威的；只有当客户端复用已存储的每设备令牌时，才会复用缓存的作用域。
- 设备令牌可以通过 `device.token.rotate` 和 `device.token.revoke` 轮换 / 撤销（需要 `operator.pairing` 作用域）。
- 令牌签发 / 轮换始终受限于该设备配对条目中记录的已批准角色集合；轮换令牌不能将设备扩展到配对审批从未授予的角色。
- 对于已配对设备令牌会话，除非调用方还具有 `operator.admin`，否则设备管理是自作用域的：非 admin 调用方只能移除 / 撤销 / 轮换**自己的**设备条目。
- `device.token.rotate` 还会根据调用方当前会话作用域集合检查请求的 operator 作用域集合。非 admin 调用方不能将令牌轮换到比自己当前持有更宽的 operator 作用域集合。
- 认证失败会包含 `error.details.code` 以及恢复提示：
  - `error.details.canRetryWithDeviceToken`（布尔值）
  - `error.details.recommendedNextStep`（`retry_with_device_token`、`update_auth_configuration`、`update_auth_credentials`、`wait_then_retry`、`review_auth_configuration`）
- 对于 `AUTH_TOKEN_MISMATCH` 的客户端行为：
  - 受信任客户端可以使用缓存的每设备令牌进行一次有界重试。
  - 如果该重试失败，客户端应停止自动重连循环，并向 operator 展示操作指引。

## 设备身份 + 配对

- 节点应包含稳定的设备身份（`device.id`），该身份应从密钥对指纹派生。
- Gateway 网关会按设备 + 角色签发令牌。
- 新的设备 ID 需要配对审批，除非启用了本地自动审批。
- 配对自动审批以直接的 local loopback 连接为中心。
- OpenClaw 还提供了一条受限的后端 / 容器本地自连接路径，用于受信任的共享密钥辅助流程。
- 同主机 tailnet 或 LAN 连接在配对上仍被视为远程连接，并且需要审批。
- 所有 WS 客户端都必须在 `connect` 期间包含 `device` 身份（operator + node）。
  只有在以下模式下，Control UI 才可以省略它：
  - `gateway.controlUi.allowInsecureAuth=true`，用于仅 localhost 的不安全 HTTP 兼容性。
  - 成功通过 `gateway.auth.mode: "trusted-proxy"` 的 operator Control UI 认证。
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true`（紧急兜底，属于严重的安全降级）。
- 所有连接都必须对服务器提供的 `connect.challenge` nonce 进行签名。

### 设备认证迁移诊断

对于仍使用挑战前签名行为的旧版客户端，`connect` 现在会在 `error.details.code` 下返回 `DEVICE_AUTH_*` 详细代码，并在 `error.details.reason` 下返回稳定的原因值。

常见迁移失败：

| Message | details.code | details.reason | 含义 |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required` | `DEVICE_AUTH_NONCE_REQUIRED` | `device-nonce-missing` | 客户端省略了 `device.nonce`（或发送了空值）。 |
| `device nonce mismatch` | `DEVICE_AUTH_NONCE_MISMATCH` | `device-nonce-mismatch` | 客户端使用了过期 / 错误的 nonce 进行签名。 |
| `device signature invalid` | `DEVICE_AUTH_SIGNATURE_INVALID` | `device-signature` | 签名载荷与 v2 载荷不匹配。 |
| `device signature expired` | `DEVICE_AUTH_SIGNATURE_EXPIRED` | `device-signature-stale` | 签名时间戳超出允许的时钟偏移范围。 |
| `device identity mismatch` | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch` | `device.id` 与公钥指纹不匹配。 |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key` | 公钥格式 / 规范化失败。 |

迁移目标：

- 始终等待 `connect.challenge`。
- 对包含服务器 nonce 的 v2 载荷进行签名。
- 在 `connect.params.device.nonce` 中发送相同的 nonce。
- 首选签名载荷为 `v3`，它除了 device / client / role / scopes / token / nonce 字段外，还绑定了 `platform` 和 `deviceFamily`。
- 出于兼容性考虑，旧版 `v2` 签名仍然可被接受，但已配对设备的元数据固定仍会在重连时控制命令策略。

## TLS + 固定

- WS 连接支持 TLS。
- 客户端可以选择固定网关证书指纹（参见 `gateway.tls` 配置，以及 `gateway.remote.tlsFingerprint` 或 CLI `--tls-fingerprint`）。

## 范围

此协议公开了**完整的网关 API**（status、channels、models、chat、agent、sessions、nodes、approvals 等）。确切接口由 `src/gateway/protocol/schema.ts` 中的 TypeBox schema 定义。
