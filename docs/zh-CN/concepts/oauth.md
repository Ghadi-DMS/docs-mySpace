---
read_when:
    - 你想端到端了解 OpenClaw OAuth
    - 你遇到了令牌失效 / 登出问题
    - 你想使用 Claude CLI 或 OAuth 认证流程
    - 你想使用多个账户或配置文件路由
summary: OpenClaw 中的 OAuth：令牌交换、存储与多账户模式
title: OAuth
x-i18n:
    generated_at: "2026-04-05T17:46:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 402e20dfeb6ae87a90cba5824a56a7ba3b964f3716508ea5cc48a47e5affdd73
    source_path: concepts/oauth.md
    workflow: 15
---

# OAuth

OpenClaw 支持通过 OAuth 使用“订阅认证”，适用于提供此能力的提供商
（尤其是 **OpenAI Codex（ChatGPT OAuth）**）。对于 Anthropic，目前实际情况
是：

- **Anthropic API 密钥**：标准 Anthropic API 计费
- **OpenClaw 中的 Anthropic 订阅认证**：Anthropic 已于 **2026 年 4 月 4 日太平洋时间下午 12:00 / 英国夏令时晚上 8:00** 通知 OpenClaw
  用户，这现在需要 **Extra Usage**

OpenAI Codex OAuth 明确支持在 OpenClaw 这类外部工具中使用。本文说明：

对于生产环境中的 Anthropic，API 密钥认证是更安全、推荐的方案。

- OAuth **令牌交换**如何工作（PKCE）
- 令牌**存储**在哪里（以及原因）
- 如何处理**多个账户**（配置文件 + 每会话覆盖）

OpenClaw 还支持自带 OAuth 或 API 密钥流程的**提供商插件**。可通过以下命令运行：

```bash
openclaw models auth login --provider <id>
```

## 令牌汇聚点（为什么存在）

OAuth 提供商通常会在登录 / 刷新流程中签发**新的刷新令牌**。某些提供商（或 OAuth 客户端）可能会在为同一用户 / 应用签发新令牌时使旧的刷新令牌失效。

实际症状：

- 你同时通过 OpenClaw _以及_ Claude Code / Codex CLI 登录 → 之后其中一个会随机“被登出”

为减少这种情况，OpenClaw 将 `auth-profiles.json` 视为一个**令牌汇聚点**：

- 运行时从**同一个位置**读取凭证
- 我们可以保留多个配置文件并以确定性的方式进行路由
- 当凭证复用自 Codex CLI 之类的外部 CLI 时，OpenClaw
  会带着来源信息镜像这些凭证，并重新读取该外部来源，而不是
  自己轮换刷新令牌

## 存储（令牌存放位置）

密钥按**每个智能体**存储：

- 认证配置文件（OAuth + API 密钥 + 可选的值级别引用）：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- 旧版兼容文件：`~/.openclaw/agents/<agentId>/agent/auth.json`
  （发现静态 `api_key` 条目时会清理掉）

仅用于旧版导入的文件（仍受支持，但不是主存储）：

- `~/.openclaw/credentials/oauth.json`（首次使用时会导入到 `auth-profiles.json` 中）

以上所有路径也都遵循 `$OPENCLAW_STATE_DIR`（状态目录覆盖）。完整参考：[/gateway/configuration](/zh-CN/gateway/configuration-reference#auth-storage)

关于静态 SecretRef 和运行时快照激活行为，请参见[密钥管理](/zh-CN/gateway/secrets)。

## Anthropic 旧版令牌兼容性

<Warning>
Anthropic 的公开 Claude Code 文档说明，直接使用 Claude Code 会保持在
Claude 订阅限额内。另一个层面上，Anthropic 已于 **2026 年 4 月 4 日太平洋时间下午 12:00 / 英国夏令时晚上 8:00** 告知 OpenClaw 用户，**OpenClaw 被视为第三方封装工具**。现有的 Anthropic 令牌配置文件在 OpenClaw 中在技术上仍可使用，但 Anthropic 表示，OpenClaw 路径现在要求该流量使用 **Extra
Usage**（按量付费，独立于订阅单独计费）。

关于 Anthropic 当前的直接 Claude Code 套餐文档，请参见 [Using Claude Code
with your Pro or Max
plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
和 [Using Claude Code with your Team or Enterprise
plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)。

如果你想在 OpenClaw 中使用其他订阅式选项，请参见 [OpenAI
Codex](/zh-CN/providers/openai)、[Qwen Cloud Coding
Plan](/zh-CN/providers/qwen)、[MiniMax Coding Plan](/zh-CN/providers/minimax)
和 [Z.AI / GLM Coding Plan](/zh-CN/providers/glm)。
</Warning>

OpenClaw 现在再次开放 Anthropic setup-token，作为旧版 / 手动路径。
Anthropic 针对 OpenClaw 的计费通知仍适用于该路径，因此
使用时请预期 Anthropic 会对
OpenClaw 驱动的 Claude 登录流量要求 **Extra Usage**。

## Anthropic Claude CLI 迁移

Anthropic 在 OpenClaw 中已不再提供受支持的本地 Claude CLI 迁移路径。
对于 Anthropic 流量，请使用 Anthropic API 密钥；或者仅在
已经配置好的情况下继续使用旧版基于令牌的认证，并接受 Anthropic 将该 OpenClaw 路径视为 **Extra Usage** 的前提。

## OAuth 交换（登录如何工作）

OpenClaw 的交互式登录流程由 `@mariozechner/pi-ai` 实现，并接入各类向导 / 命令。

### Anthropic setup-token

流程形态：

1. 从 OpenClaw 启动 Anthropic setup-token 或 paste-token
2. OpenClaw 将生成的 Anthropic 凭证存储到认证配置文件中
3. 模型选择保持为 `anthropic/...`
4. 现有的 Anthropic 认证配置文件仍可用于回滚 / 顺序控制

### OpenAI Codex（ChatGPT OAuth）

OpenAI Codex OAuth 明确支持在 Codex CLI 之外使用，包括 OpenClaw 工作流。

流程形态（PKCE）：

1. 生成 PKCE verifier / challenge + 随机 `state`
2. 打开 `https://auth.openai.com/oauth/authorize?...`
3. 尝试在 `http://127.0.0.1:1455/auth/callback` 捕获回调
4. 如果回调无法绑定（或者你处于远程 / 无头环境），则粘贴重定向 URL / 代码
5. 在 `https://auth.openai.com/oauth/token` 执行交换
6. 从访问令牌中提取 `accountId`，并存储 `{ access, refresh, expires, accountId }`

向导路径为 `openclaw onboard` → 认证选项 `openai-codex`。

## 刷新 + 过期

配置文件会存储一个 `expires` 时间戳。

在运行时：

- 如果 `expires` 在未来 → 使用已存储的访问令牌
- 如果已过期 → 刷新（在文件锁保护下）并覆盖已存储的凭证
- 例外：复用的外部 CLI 凭证仍由外部管理；OpenClaw
  会重新读取 CLI 认证存储，而不会自行使用复制来的刷新令牌

刷新流程是自动的；通常你不需要手动管理令牌。

## 多个账户（配置文件）+ 路由

有两种模式：

### 1）首选：分离智能体

如果你希望“个人”和“工作”永不互相影响，请使用隔离的智能体（独立会话 + 凭证 + 工作区）：

```bash
openclaw agents add work
openclaw agents add personal
```

然后按每个智能体配置认证（通过向导），并将聊天路由到正确的智能体。

### 2）高级：在一个智能体中使用多个配置文件

`auth-profiles.json` 支持同一提供商的多个配置文件 ID。

选择使用哪个配置文件：

- 通过配置顺序全局选择（`auth.order`）
- 通过 `/model ...@<profileId>` 按会话选择

示例（会话覆盖）：

- `/model Opus@anthropic:work`

如何查看有哪些配置文件 ID：

- `openclaw channels list --json`（显示 `auth[]`）

相关文档：

- [/concepts/model-failover](/zh-CN/concepts/model-failover)（轮换 + 冷却规则）
- [/tools/slash-commands](/zh-CN/tools/slash-commands)（命令界面）

## 相关内容

- [认证](/zh-CN/gateway/authentication) — 模型提供商认证概览
- [密钥](/zh-CN/gateway/secrets) — 凭证存储与 SecretRef
- [配置参考](/zh-CN/gateway/configuration-reference#auth-storage) — 认证配置键
