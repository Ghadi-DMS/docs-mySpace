---
read_when:
    - 调试模型认证或 OAuth 过期问题
    - 编写关于认证或凭证存储的文档
summary: 模型认证：OAuth、API 密钥以及旧版 Anthropic setup-token
title: 认证
x-i18n:
    generated_at: "2026-04-05T17:47:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: f59ede3fcd7e692ad4132287782a850526acf35474b5bfcea29e0e23610636c2
    source_path: gateway/authentication.md
    workflow: 15
---

# 认证（模型提供商）

<Note>
本页介绍的是**模型提供商**认证（API 密钥、OAuth，以及旧版 Anthropic setup-token）。关于**Gateway 网关连接**认证（token、password、trusted-proxy），请参阅 [配置](/zh-CN/gateway/configuration) 和 [Trusted Proxy Auth](/zh-CN/gateway/trusted-proxy-auth)。
</Note>

OpenClaw 支持模型提供商使用 OAuth 和 API 密钥。对于始终在线的 Gateway 网关主机，API 密钥通常是最可预测的选择。当它与你的提供商账户模型匹配时，也支持订阅制 / OAuth 流程。

完整的 OAuth 流程和存储布局，请参阅 [/concepts/oauth](/zh-CN/concepts/oauth)。
关于基于 SecretRef 的认证（`env`/`file`/`exec` 提供商），请参阅 [Secrets Management](/zh-CN/gateway/secrets)。
关于 `models status --probe` 使用的凭证适用性 / 原因代码规则，请参阅
[Auth Credential Semantics](/zh-CN/auth-credential-semantics)。

## 推荐设置（API 密钥，任意提供商）

如果你正在运行长期存活的 Gateway 网关，请先为你选择的提供商配置 API 密钥。
对于 Anthropic，API 密钥认证是稳妥方案。OpenClaw 中 Anthropic 的订阅式认证属于旧版 setup-token 路径，应被视为**额外用量**路径，而不是套餐限额路径。

1. 在你的提供商控制台中创建一个 API 密钥。
2. 将其放到**Gateway 网关主机**上（也就是运行 `openclaw gateway` 的机器）。

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. 如果 Gateway 网关运行在 systemd/launchd 下，建议将密钥放入
   `~/.openclaw/.env`，这样守护进程就可以读取：

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

然后重启守护进程（或重启你的 Gateway 网关进程），并再次检查：

```bash
openclaw models status
openclaw doctor
```

如果你不想自行管理环境变量，新手引导也可以为守护进程存储
API 密钥：`openclaw onboard`。

有关环境继承（`env.shellEnv`、`~/.openclaw/.env`、systemd/launchd）的详细信息，请参阅 [帮助](/zh-CN/help)。

## Anthropic：旧版令牌兼容性

Anthropic setup-token 认证在 OpenClaw 中仍可作为旧版 / 手动路径使用。Anthropic 的公开 Claude Code 文档仍然涵盖 Claude 套餐下直接在终端中使用 Claude Code，但 Anthropic 也单独告知 OpenClaw 用户，**OpenClaw** 的 Claude 登录路径会被视为第三方 harness 用法，并且需要按**额外用量**单独计费，而不包含在订阅内。

若要采用最清晰的设置路径，请使用 Anthropic API 密钥。如果你必须在 OpenClaw 中保留 Anthropic 的订阅式路径，请使用旧版 setup-token 路径，并预期 Anthropic 会将其视为**额外用量**。

手动输入令牌（任意提供商；写入 `auth-profiles.json` 并更新配置）：

```bash
openclaw models auth paste-token --provider openrouter
```

静态凭证也支持 auth profile 引用：

- `api_key` 凭证可以使用 `keyRef: { source, provider, id }`
- `token` 凭证可以使用 `tokenRef: { source, provider, id }`
- OAuth 模式的 profile 不支持 SecretRef 凭证；如果 `auth.profiles.<id>.mode` 被设置为 `"oauth"`，则会拒绝该 profile 使用由 SecretRef 支持的 `keyRef`/`tokenRef` 输入。

适合自动化的检查（过期 / 缺失时退出码为 `1`，即将过期时为 `2`）：

```bash
openclaw models status --check
```

实时认证探测：

```bash
openclaw models status --probe
```

说明：

- 探测结果行可以来自 auth profile、环境变量凭证或 `models.json`。
- 如果显式的 `auth.order.<provider>` 省略了某个已存储 profile，探测会为该 profile 报告
  `excluded_by_auth_order`，而不是尝试使用它。
- 如果认证存在，但 OpenClaw 无法为该提供商解析出可探测的模型候选项，探测会报告 `status: no_model`。
- 限流冷却可以按模型范围生效。某个 profile 对一个模型处于冷却状态时，仍可能可用于同一提供商下的兄弟模型。

可选的运维脚本文档（systemd/Termux）见这里：
[认证监控脚本](/zh-CN/help/scripts#auth-monitoring-scripts)

## Anthropic 说明

Anthropic `claude-cli` 后端已被移除。

- 在 OpenClaw 中处理 Anthropic 流量时，请使用 Anthropic API 密钥。
- Anthropic setup-token 仍保留为旧版 / 手动路径，使用时应预期按照 Anthropic 向 OpenClaw 用户说明的**额外用量**计费。
- `openclaw doctor` 现在会检测已移除但残留的 Anthropic Claude CLI 状态。如果存储的凭证字节仍然存在，doctor 会将它们转换回
  Anthropic token/OAuth profile。否则，doctor 会移除陈旧的 Claude CLI
  配置，并引导你使用 API 密钥或 setup-token 进行恢复。

## 检查模型认证状态

```bash
openclaw models status
openclaw doctor
```

## API 密钥轮换行为（Gateway 网关）

某些提供商支持在 API 调用触发提供商限流时，使用其他密钥重试请求。

- 优先级顺序：
  - `OPENCLAW_LIVE_<PROVIDER>_KEY`（单个覆盖）
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- Google 提供商还会将 `GOOGLE_API_KEY` 作为额外回退项。
- 同一个密钥列表在使用前会先去重。
- OpenClaw 仅会在限流错误时使用下一个密钥重试（例如
  `429`、`rate_limit`、`quota`、`resource exhausted`、`Too many concurrent requests`、`ThrottlingException`、`concurrency limit reached`，或
  `workers_ai ... quota limit exceeded`）。
- 非限流错误不会使用备用密钥重试。
- 如果所有密钥都失败，则返回最后一次尝试的最终错误。

## 控制使用哪个凭证

### 按会话控制（聊天命令）

使用 `/model <alias-or-id>@<profileId>` 为当前会话固定指定某个提供商凭证（示例 profile id：`anthropic:default`、`anthropic:work`）。

使用 `/model`（或 `/model list`）可打开精简选择器；使用 `/model status` 可查看完整视图（候选项 + 下一个 auth profile，以及在已配置时显示的提供商端点详情）。

### 按智能体控制（CLI 覆盖）

为某个智能体设置显式的 auth profile 顺序覆盖（存储在该智能体的 `auth-profiles.json` 中）：

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

使用 `--agent <id>` 可指定某个智能体；省略时则使用已配置的默认智能体。
调试顺序问题时，`openclaw models status --probe` 会将被省略的已存储
profile 显示为 `excluded_by_auth_order`，而不是静默跳过。
调试冷却问题时，请记住限流冷却可能只绑定到某个模型 id，而不是整个提供商 profile。

## 故障排除

### “未找到凭证”

如果 Anthropic profile 缺失，请在**Gateway 网关主机**上配置 Anthropic API 密钥，或设置旧版 Anthropic setup-token 路径，然后重新检查：

```bash
openclaw models status
```

### 令牌即将过期 / 已过期

运行 `openclaw models status` 以确认是哪个 profile 即将过期。如果旧版
Anthropic token profile 缺失或已过期，请通过
setup-token 刷新该设置，或迁移到 Anthropic API 密钥。

如果这台机器仍保留旧构建遗留的、已移除的 Anthropic Claude CLI 状态，请运行：

```bash
openclaw doctor --yes
```

如果存储的凭证字节仍然存在，Doctor 会将 `anthropic:claude-cli` 转换回 Anthropic token/OAuth。否则，它会移除陈旧的 Claude CLI
profile/config/model 引用，并保留下一步操作指引。
