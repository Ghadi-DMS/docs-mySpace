---
read_when:
    - 回答常见的设置、安装、新手引导或运行时支持问题
    - 在深入调试前分诊用户报告的问题
summary: 关于 OpenClaw 设置、配置和使用的常见问题
title: 常见问题
x-i18n:
    generated_at: "2026-04-07T09:43:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: b2ad2fc35fb5b1d6c8fb75e7c0c409089dff033a9810c863c3f7ef64834a9b77
    source_path: help/faq.md
    workflow: 15
---

# 常见问题

面向真实环境设置的快速解答和更深入的故障排除（本地开发、VPS、多智能体、OAuth/API 密钥、模型故障切换）。如需运行时诊断，请参阅 [故障排除](/zh-CN/gateway/troubleshooting)。如需完整的配置参考，请参阅 [Configuration](/zh-CN/gateway/configuration)。

## 最初的六十秒：如果哪里坏了

1. **快速状态（第一项检查）**

   ```bash
   openclaw status
   ```

   快速本地摘要：操作系统 + 更新、Gateway 网关/服务可达性、智能体/会话、提供商配置 + 运行时问题（当 Gateway 网关可达时）。

2. **可粘贴报告（可安全分享）**

   ```bash
   openclaw status --all
   ```

   只读诊断，附带日志尾部（令牌已脱敏）。

3. **守护进程 + 端口状态**

   ```bash
   openclaw gateway status
   ```

   显示 supervisor 运行状态与 RPC 可达性的区别、探测目标 URL，以及服务很可能使用了哪个配置。

4. **深度探测**

   ```bash
   openclaw status --deep
   ```

   执行实时 Gateway 网关健康探测，包括在支持时执行渠道探测
   （需要可达的 Gateway 网关）。参见 [Health](/zh-CN/gateway/health)。

5. **跟随最新日志**

   ```bash
   openclaw logs --follow
   ```

   如果 RPC 不可用，则退回到：

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   文件日志与服务日志彼此独立；参见 [Logging](/zh-CN/logging) 和 [故障排除](/zh-CN/gateway/troubleshooting)。

6. **运行 Doctor（修复）**

   ```bash
   openclaw doctor
   ```

   修复/迁移配置与状态 + 执行健康检查。参见 [Doctor](/zh-CN/gateway/doctor)。

7. **Gateway 网关快照**

   ```bash
   openclaw health --json
   openclaw health --verbose   # 出错时显示目标 URL + 配置路径
   ```

   向正在运行的 Gateway 网关请求完整快照（仅 WS）。参见 [Health](/zh-CN/gateway/health)。

## 快速开始与首次运行设置

<AccordionGroup>
  <Accordion title="我卡住了，最快怎么解决问题">
    使用一个能**看到你的机器**的本地 AI 智能体。这远比在 Discord 里提问更有效，
    因为大多数“我卡住了”的情况都是**本地配置或环境问题**，
    远程协助者无法直接检查。

    - **Claude Code**： [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**： [https://openai.com/codex/](https://openai.com/codex/)

    这些工具可以读取仓库、运行命令、检查日志，并帮助修复你的机器级问题
    （PATH、服务、权限、认证文件）。通过可修改的（git）安装方式，
    将**完整源码检出**交给它们：

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    这会**从 git 检出安装** OpenClaw，因此智能体可以读取代码 + 文档，
    并基于你正在运行的准确版本进行推理。你之后随时都可以
    通过不带 `--install-method git` 重新运行安装器来切回稳定版。

    提示：让智能体先**规划并监督**修复过程（一步一步），然后只执行必要的命令。
    这样改动会更小，也更容易审计。

    如果你发现了真实 bug 或修复，请提交 GitHub issue 或发送 PR：
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    从这些命令开始（求助时请分享输出）：

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    它们的作用：

    - `openclaw status`：Gateway 网关/智能体健康状态 + 基础配置的快速快照。
    - `openclaw models status`：检查提供商认证 + 模型可用性。
    - `openclaw doctor`：验证并修复常见配置/状态问题。

    其他有用的 CLI 检查：`openclaw status --all`、`openclaw logs --follow`、
    `openclaw gateway status`、`openclaw health --verbose`。

    快速调试循环：[最初的六十秒：如果哪里坏了](#first-60-seconds-if-something-is-broken)。
    安装文档：[Install](/zh-CN/install)、[Installer flags](/zh-CN/install/installer)、[Updating](/zh-CN/install/updating)。

  </Accordion>

  <Accordion title="Heartbeat 一直跳过。这些跳过原因是什么意思？">
    常见的 Heartbeat 跳过原因：

    - `quiet-hours`：当前时间不在配置的 active-hours 窗口内
    - `empty-heartbeat-file`：`HEARTBEAT.md` 存在，但只包含空白/仅标题的脚手架内容
    - `no-tasks-due`：`HEARTBEAT.md` 任务模式已启用，但当前没有任何任务间隔到期
    - `alerts-disabled`：所有 Heartbeat 可见性都已禁用（`showOk`、`showAlerts` 和 `useIndicator` 全部关闭）

    在任务模式下，只有在真实的 Heartbeat 运行
    完成后，才会推进到期时间戳。被跳过的运行不会将任务标记为已完成。

    文档：[Heartbeat](/zh-CN/gateway/heartbeat)、[Automation & Tasks](/zh-CN/automation)。

  </Accordion>

  <Accordion title="安装并设置 OpenClaw 的推荐方式">
    仓库推荐从源码运行并使用新手引导：

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    向导还可以自动构建 UI 资源。完成新手引导后，你通常会在 **18789** 端口运行 Gateway 网关。

    从源码开始（贡献者/开发者）：

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # 首次运行时自动安装 UI 依赖
    openclaw onboard
    ```

    如果你还没有全局安装，可以通过 `pnpm openclaw onboard` 运行它。

  </Accordion>

  <Accordion title="完成新手引导后，如何打开仪表盘？">
    向导会在新手引导完成后立即用浏览器打开一个干净的（不带令牌的）仪表盘 URL，并且也会在摘要中打印该链接。请保持该标签页打开；如果它没有自动启动，请在同一台机器上复制/粘贴打印出的 URL。
  </Accordion>

  <Accordion title="如何在 localhost 和远程环境中认证仪表盘？">
    **localhost（同一台机器）：**

    - 打开 `http://127.0.0.1:18789/`。
    - 如果它要求 shared-secret 认证，请将已配置的 token 或 password 粘贴到 Control UI 设置中。
    - Token 来源：`gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）。
    - Password 来源：`gateway.auth.password`（或 `OPENCLAW_GATEWAY_PASSWORD`）。
    - 如果还没有配置 shared secret，可通过 `openclaw doctor --generate-gateway-token` 生成 token。

    **不在 localhost：**

    - **Tailscale Serve**（推荐）：保持 bind loopback，运行 `openclaw gateway --tailscale serve`，打开 `https://<magicdns>/`。如果 `gateway.auth.allowTailscale` 为 `true`，身份标头将满足 Control UI/WebSocket 认证要求（无需粘贴 shared secret，假定 gateway host 可信）；除非你有意使用 private-ingress `none` 或 trusted-proxy HTTP 认证，否则 HTTP API 仍然需要 shared-secret 认证。
      来自同一客户端的并发错误 Serve 认证尝试，会在 failed-auth limiter 记录之前被串行化，因此第二次错误重试可能已经显示 `retry later`。
    - **Tailnet bind**：运行 `openclaw gateway --bind tailnet --token "<token>"`（或配置 password 认证），打开 `http://<tailscale-ip>:18789/`，然后在仪表盘设置中粘贴匹配的 shared secret。
    - **具备身份感知的反向代理**：将 Gateway 网关放在非 loopback 的 trusted proxy 后面，配置 `gateway.auth.mode: "trusted-proxy"`，然后打开代理 URL。
    - **SSH 隧道**：`ssh -N -L 18789:127.0.0.1:18789 user@host`，然后打开 `http://127.0.0.1:18789/`。shared-secret 认证在隧道中仍然适用；如有提示，请粘贴已配置的 token 或 password。

    参见 [Dashboard](/web/dashboard) 和 [Web surfaces](/web) 了解 bind 模式与认证细节。

  </Accordion>

  <Accordion title="为什么聊天审批有两个 exec approval 配置？">
    它们控制的是不同层级：

    - `approvals.exec`：将审批提示转发到聊天目标
    - `channels.<channel>.execApprovals`：让该渠道充当 exec 审批的原生审批客户端

    主机 exec 策略仍然是真正的审批关卡。聊天配置只控制审批
    提示出现在哪里，以及人们如何进行回应。

    在大多数设置中，你**不**需要同时启用两者：

    - 如果聊天本身已经支持命令和回复，那么同一聊天中的 `/approve` 会通过共享路径工作。
    - 如果某个受支持的原生渠道能够安全推断审批人，OpenClaw 现在会在 `channels.<channel>.execApprovals.enabled` 未设置或为 `"auto"` 时，自动启用“优先私信”的原生审批。
    - 当存在原生审批卡片/按钮时，该原生 UI 是主要路径；只有当工具结果说明聊天审批不可用，或手动审批是唯一可用路径时，智能体才应包含手动 `/approve` 命令。
    - 只有在提示还必须转发到其他聊天或明确的运维房间时，才使用 `approvals.exec`。
    - 仅当你明确希望把审批提示发回原始房间/话题时，才使用 `channels.<channel>.execApprovals.target: "channel"` 或 `"both"`。
    - 插件审批又是另一套机制：默认使用同一聊天内的 `/approve`，可选 `approvals.plugin` 转发，并且只有部分原生渠道会在此之上保留原生插件审批处理。

    简单来说：转发用于路由，原生客户端配置用于更丰富的渠道专属 UX。
    参见 [Exec Approvals](/zh-CN/tools/exec-approvals)。

  </Accordion>

  <Accordion title="我需要什么运行时？">
    需要 Node **>= 22**。推荐使用 `pnpm`。Gateway 网关**不推荐**使用 Bun。
  </Accordion>

  <Accordion title="它能在 Raspberry Pi 上运行吗？">
    可以。Gateway 网关很轻量——文档列出的个人使用要求为 **512MB-1GB RAM**、**1 个核心**，
    以及大约 **500MB** 磁盘，并注明 **Raspberry Pi 4 可以运行它**。

    如果你想有更多余量（日志、媒体、其他服务），建议使用 **2GB**，
    但这不是硬性最低要求。

    提示：一台小型 Pi/VPS 可以托管 Gateway 网关，而你可以在笔记本/手机上配对 **nodes**，
    以便访问本地屏幕/摄像头/canvas 或执行命令。参见 [Nodes](/zh-CN/nodes)。

  </Accordion>

  <Accordion title="Raspberry Pi 安装有什么建议吗？">
    简短回答：可以运行，但预计会有一些边角问题。

    - 使用 **64-bit** 操作系统，并保持 Node >= 22。
    - 优先选择**可修改的（git）安装**，这样你能查看日志并快速更新。
    - 一开始不要启用 channels/Skills，然后一个一个添加。
    - 如果遇到奇怪的二进制问题，通常是 **ARM 兼容性**问题。

    文档：[Linux](/zh-CN/platforms/linux)、[Install](/zh-CN/install)。

  </Accordion>

  <Accordion title="卡在 wake up my friend / onboarding hatch 不出来。现在怎么办？">
    这个界面依赖 Gateway 网关可达且已认证。TUI 也会在首次 hatch 时自动发送
    “Wake up, my friend!”。如果你看到这行但**没有回复**，
    并且 token 仍为 0，说明智能体根本没有运行。

    1. 重启 Gateway 网关：

    ```bash
    openclaw gateway restart
    ```

    2. 检查状态 + 认证：

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. 如果仍然卡住，请运行：

    ```bash
    openclaw doctor
    ```

    如果 Gateway 网关是远程的，请确保隧道/Tailscale 连接正常，并且 UI
    指向了正确的 Gateway 网关。参见 [Remote access](/zh-CN/gateway/remote)。

  </Accordion>

  <Accordion title="我能把现有设置迁移到新机器（Mac mini）而不重新做新手引导吗？">
    可以。复制**状态目录**和**工作区**，然后运行一次 Doctor。这样可以
    保持你的机器人“**完全一样**”（记忆、会话历史、认证和渠道状态），
    前提是你同时复制**这两个**位置：

    1. 在新机器上安装 OpenClaw。
    2. 从旧机器复制 `$OPENCLAW_STATE_DIR`（默认：`~/.openclaw`）。
    3. 复制你的工作区（默认：`~/.openclaw/workspace`）。
    4. 运行 `openclaw doctor` 并重启 Gateway 网关服务。

    这样会保留配置、认证档案、WhatsApp 凭据、会话和记忆。如果你处于
    远程模式，请记住 session store 和工作区归 gateway host 所有。

    **重要：**如果你只是把工作区 commit/push 到 GitHub，
    你备份的是**记忆 + bootstrap 文件**，但**不包括**会话历史或认证。
    这些内容保存在 `~/.openclaw/` 下（例如 `~/.openclaw/agents/<agentId>/sessions/`）。

    相关内容：[Migrating](/zh-CN/install/migrating)、[磁盘上的文件位置](#where-things-live-on-disk)、
    [Agent workspace](/zh-CN/concepts/agent-workspace)、[Doctor](/zh-CN/gateway/doctor)、
    [Remote mode](/zh-CN/gateway/remote)。

  </Accordion>

  <Accordion title="我在哪里查看最新版本有哪些新内容？">
    查看 GitHub 变更日志：
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    最新条目在顶部。如果顶部部分标记为 **Unreleased**，则下一个带日期的
    部分就是最近发布的版本。条目按 **Highlights**、**Changes** 和
    **Fixes** 分组（必要时还会有 docs/other 部分）。

  </Accordion>

  <Accordion title="无法访问 docs.openclaw.ai（SSL 错误）">
    某些 Comcast/Xfinity 连接会因 Xfinity
    Advanced Security 错误地拦截 `docs.openclaw.ai`。请禁用它或将 `docs.openclaw.ai` 加入 allowlist，然后重试。
    也请通过这里报告，帮助我们解除拦截：[https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status)。

    如果你仍然无法访问该站点，文档在 GitHub 上也有镜像：
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="稳定版和 beta 的区别">
    **稳定版**和 **beta** 是 **npm dist-tags**，不是不同的代码线：

    - `latest` = 稳定版
    - `beta` = 用于测试的早期构建

    通常，一个稳定版本会先发布到 **beta**，然后再通过显式的
    promotion 步骤将同一版本移动到 `latest`。维护者在需要时也可以
    直接发布到 `latest`。这就是为什么 beta 和稳定版在 promotion 后
    可能会指向**同一个版本**。

    查看变更内容：
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    关于安装单行命令以及 beta 和 dev 的区别，请参阅下方的折叠项。

  </Accordion>

  <Accordion title="如何安装 beta 版本？beta 和 dev 有什么区别？">
    **Beta** 是 npm dist-tag `beta`（promotion 后可能与 `latest` 相同）。
    **Dev** 是 `main` 的移动头部（git）；发布时使用 npm dist-tag `dev`。

    单行命令（macOS/Linux）：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Windows 安装器（PowerShell）：
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    更多细节：[Development channels](/zh-CN/install/development-channels) 和 [Installer flags](/zh-CN/install/installer)。

  </Accordion>

  <Accordion title="如何试用最新内容？">
    有两个选项：

    1. **Dev 渠道（git 检出）：**

    ```bash
    openclaw update --channel dev
    ```

    这会切换到 `main` 分支并从源码更新。

    2. **可修改安装（从安装站点）：**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    这样你会得到一个可编辑的本地仓库，然后可通过 git 更新。

    如果你更喜欢手动进行干净克隆，请使用：

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    文档：[Update](/cli/update)、[Development channels](/zh-CN/install/development-channels)、
    [Install](/zh-CN/install)。

  </Accordion>

  <Accordion title="安装和新手引导通常需要多久？">
    大致参考：

    - **安装：** 2-5 分钟
    - **新手引导：** 5-15 分钟，取决于你配置了多少个渠道/模型

    如果卡住了，请使用 [Installer stuck](#quick-start-and-first-run-setup)
    以及 [我卡住了](#quick-start-and-first-run-setup) 中的快速调试循环。

  </Accordion>

  <Accordion title="安装器卡住了？如何获得更多反馈？">
    用**详细输出**重新运行安装器：

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    带详细输出的 beta 安装：

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    对于可修改的（git）安装：

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    Windows（PowerShell）等效方式：

    ```powershell
    # install.ps1 目前还没有专用的 -Verbose 标志。
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    更多选项：[Installer flags](/zh-CN/install/installer)。

  </Accordion>

  <Accordion title="Windows 安装提示 git not found 或 openclaw not recognized">
    两个常见的 Windows 问题：

    **1) npm 错误 spawn git / git not found**

    - 安装 **Git for Windows**，并确保 `git` 已加入你的 PATH。
    - 关闭并重新打开 PowerShell，然后重新运行安装器。

    **2) 安装后 openclaw is not recognized**

    - 你的 npm 全局 bin 目录没有在 PATH 中。
    - 检查该路径：

      ```powershell
      npm config get prefix
      ```

    - 将该目录添加到你的用户 PATH 中（Windows 上不需要 `\bin` 后缀；在大多数系统上它是 `%AppData%\npm`）。
    - 更新 PATH 后，关闭并重新打开 PowerShell。

    如果你想要最顺畅的 Windows 设置，请使用 **WSL2**，而不是原生 Windows。
    文档：[Windows](/zh-CN/platforms/windows)。

  </Accordion>

  <Accordion title="Windows exec 输出显示乱码中文，我该怎么办？">
    这通常是原生 Windows shell 的控制台代码页不匹配。

    症状：

    - `system.run`/`exec` 输出中的中文显示为乱码
    - 同一个命令在另一个终端配置中显示正常

    PowerShell 中的快速解决办法：

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    然后重启 Gateway 网关并重试你的命令：

    ```powershell
    openclaw gateway restart
    ```

    如果你在最新的 OpenClaw 上仍能复现此问题，请在这里跟踪/报告：

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="文档没有回答我的问题——如何获得更好的答案？">
    使用**可修改的（git）安装**，这样你就拥有完整的本地源码和文档，然后
    在该文件夹中向你的机器人（或 Claude/Codex）提问，
    这样它就能读取仓库并给出准确回答。

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    更多细节：[Install](/zh-CN/install) 和 [Installer flags](/zh-CN/install/installer)。

  </Accordion>

  <Accordion title="如何在 Linux 上安装 OpenClaw？">
    简短回答：按照 Linux 指南操作，然后运行新手引导。

    - Linux 快速路径 + 服务安装：[Linux](/zh-CN/platforms/linux)。
    - 完整演练：[入门指南](/zh-CN/start/getting-started)。
    - 安装器 + 更新：[Install & updates](/zh-CN/install/updating)。

  </Accordion>

  <Accordion title="如何在 VPS 上安装 OpenClaw？">
    任意 Linux VPS 都可以。在服务器上安装，然后使用 SSH/Tailscale 访问 Gateway 网关。

    指南：[exe.dev](/zh-CN/install/exe-dev)、[Hetzner](/zh-CN/install/hetzner)、[Fly.io](/zh-CN/install/fly)。
    远程访问：[Gateway remote](/zh-CN/gateway/remote)。

  </Accordion>

  <Accordion title="云端/VPS 安装指南在哪里？">
    我们提供了一个**托管中心**，汇总常见提供商。选择一个并按指南操作：

    - [VPS hosting](/zh-CN/vps)（所有提供商集中在一处）
    - [Fly.io](/zh-CN/install/fly)
    - [Hetzner](/zh-CN/install/hetzner)
    - [exe.dev](/zh-CN/install/exe-dev)

    在云中它的工作方式是：**Gateway 网关运行在服务器上**，而你通过
    Control UI（或 Tailscale/SSH）从笔记本/手机访问它。你的状态 + 工作区
    都保存在服务器上，因此应将该主机视为事实来源并做好备份。

    你可以将 **nodes**（Mac/iOS/Android/headless）配对到这个云端 Gateway 网关，
    以访问本地屏幕/摄像头/canvas，或在你的笔记本上运行命令，同时
    把 Gateway 网关保留在云中。

    中心：[Platforms](/zh-CN/platforms)。远程访问：[Gateway remote](/zh-CN/gateway/remote)。
    Nodes：[Nodes](/zh-CN/nodes)、[Nodes CLI](/cli/nodes)。

  </Accordion>

  <Accordion title="我可以让 OpenClaw 自己更新自己吗？">
    简短回答：**可以，但不推荐**。更新流程可能会重启
    Gateway 网关（从而中断当前会话），可能需要干净的 git 检出，
    并且可能会要求确认。更安全的做法是由操作员在 shell 中执行更新。

    使用 CLI：

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    如果你必须从智能体自动化执行：

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    文档：[Update](/cli/update)、[Updating](/zh-CN/install/updating)。

  </Accordion>

  <Accordion title="新手引导实际上会做什么？">
    `openclaw onboard` 是推荐的设置路径。在**本地模式**下，它会引导你完成：

    - **模型/认证设置**（提供商 OAuth、API 密钥、Anthropic setup-token，以及 LM Studio 等本地模型选项）
    - **工作区**位置 + bootstrap 文件
    - **Gateway 网关设置**（bind/port/auth/tailscale）
    - **渠道**（WhatsApp、Telegram、Discord、Mattermost、Signal、iMessage，以及像 QQ Bot 这样的内置渠道插件）
    - **Daemon 安装**（macOS 上的 LaunchAgent；Linux/WSL2 上的 systemd user unit）
    - **健康检查**和 **Skills** 选择

    如果你配置的模型未知或缺少认证，它也会给出警告。

  </Accordion>

  <Accordion title="运行这个需要 Claude 或 OpenAI 订阅吗？">
    不需要。你可以使用 **API 密钥**（Anthropic/OpenAI/其他）运行 OpenClaw，
    也可以使用**纯本地模型**，让你的数据保留在本机。订阅（Claude
    Pro/Max 或 OpenAI Codex）只是这些提供商的一种可选认证方式。

    在 OpenClaw 中，对于 Anthropic，实际区别如下：

    - **Anthropic API key**：标准 Anthropic API 计费
    - **OpenClaw 中的 Claude CLI / Claude subscription auth**：Anthropic 员工
      告诉我们这种用法再次被允许，OpenClaw 因此将 `claude -p`
      的用法视为该集成的获准方式，除非 Anthropic 发布新的
      政策

    对于长期运行的 gateway host，Anthropic API key
    仍然是更可预测的设置方式。OpenAI Codex OAuth 也明确支持像 OpenClaw 这样的外部
    工具使用。

    OpenClaw 还支持其他托管型订阅选项，包括
    **Qwen Cloud Coding Plan**、**MiniMax Coding Plan** 和
    **Z.AI / GLM Coding Plan**。

    文档：[Anthropic](/zh-CN/providers/anthropic)、[OpenAI](/zh-CN/providers/openai)、
    [Qwen Cloud](/zh-CN/providers/qwen)、
    [MiniMax](/zh-CN/providers/minimax)、[GLM Models](/zh-CN/providers/glm)、
    [Local models](/zh-CN/gateway/local-models)、[Models](/zh-CN/concepts/models)。

  </Accordion>

  <Accordion title="我可以在没有 API key 的情况下使用 Claude Max 订阅吗？">
    可以。

    Anthropic 员工告诉我们，OpenClaw 风格的 Claude CLI 用法再次被允许，因此
    OpenClaw 将 Claude subscription auth 和 `claude -p` 的用法视为该集成中获准的
    方式，除非 Anthropic 发布新政策。如果你希望
    拥有最可预测的服务端设置，请改用 Anthropic API key。

  </Accordion>

  <Accordion title="你们支持 Claude subscription auth（Claude Pro 或 Max）吗？">
    支持。

    Anthropic 员工告诉我们，这种用法再次被允许，因此 OpenClaw 将
    Claude CLI 复用和 `claude -p` 的使用视为该集成中获准的
    方式，除非 Anthropic 发布新的政策。

    Anthropic setup-token 仍然是 OpenClaw 支持的一种 token 路径，但 OpenClaw 现在在可用时更倾向于 Claude CLI 复用和 `claude -p`。
    对于生产环境或多用户工作负载，Anthropic API key auth 仍然是
    更安全、更可预测的选择。如果你想在 OpenClaw 中使用其他订阅式托管
    选项，请参见 [OpenAI](/zh-CN/providers/openai)、[Qwen / Model
    Cloud](/zh-CN/providers/qwen)、[MiniMax](/zh-CN/providers/minimax) 和 [GLM
    Models](/zh-CN/providers/glm)。

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="为什么我会看到来自 Anthropic 的 HTTP 429 rate_limit_error？">
这表示你当前时间窗口中的 **Anthropic 配额/速率限制** 已耗尽。如果你
使用 **Claude CLI**，请等待窗口重置或升级你的套餐。如果你
使用 **Anthropic API key**，请检查 Anthropic Console
中的使用量/账单，并按需提高限额。

    如果消息具体为：
    `Extra usage is required for long context requests`，说明请求尝试使用
    Anthropic 的 1M context beta（`context1m: true`）。这只有在你的
    凭证有资格进行长上下文计费时才有效（API key 计费或
    启用了 Extra Usage 的 OpenClaw Claude 登录路径）。

    提示：设置一个**后备模型**，这样当某个提供商被限流时，OpenClaw 仍能继续回复。
    参见 [Models](/cli/models)、[OAuth](/zh-CN/concepts/oauth) 和
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/zh-CN/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context)。

  </Accordion>

  <Accordion title="支持 AWS Bedrock 吗？">
    支持。OpenClaw 内置了 **Amazon Bedrock（Converse）** 提供商。当 AWS 环境标记存在时，OpenClaw 可以自动发现流式/文本 Bedrock 目录，并将其合并为隐式 `amazon-bedrock` 提供商；否则你也可以显式启用 `plugins.entries.amazon-bedrock.config.discovery.enabled` 或添加手动提供商条目。参见 [Amazon Bedrock](/zh-CN/providers/bedrock) 和 [Model providers](/zh-CN/providers/models)。如果你更喜欢托管密钥流程，在 Bedrock 前使用 OpenAI 兼容代理也仍然是有效选项。
  </Accordion>

  <Accordion title="Codex auth 是如何工作的？">
    OpenClaw 通过 OAuth（ChatGPT 登录）支持 **OpenAI Code（Codex）**。新手引导可以执行 OAuth 流程，并会在合适时将默认模型设置为 `openai-codex/gpt-5.4`。参见 [Model providers](/zh-CN/concepts/model-providers) 和 [新手引导（CLI）](/zh-CN/start/wizard)。
  </Accordion>

  <Accordion title="为什么 ChatGPT GPT-5.4 不会在 OpenClaw 中解锁 openai/gpt-5.4？">
    OpenClaw 将这两条路径分开处理：

    - `openai-codex/gpt-5.4` = ChatGPT/Codex OAuth
    - `openai/gpt-5.4` = 直接 OpenAI Platform API

    在 OpenClaw 中，ChatGPT/Codex 登录连接的是 `openai-codex/*` 路径，
    而不是直接的 `openai/*` 路径。如果你想在 OpenClaw 中使用
    直接 API 路径，请设置 `OPENAI_API_KEY`（或等效的 OpenAI 提供商配置）。
    如果你想在 OpenClaw 中使用 ChatGPT/Codex 登录，请使用 `openai-codex/*`。

  </Accordion>

  <Accordion title="为什么 Codex OAuth 限额会与 ChatGPT 网页版不同？">
    `openai-codex/*` 使用 Codex OAuth 路径，其可用配额窗口由
    OpenAI 管理，并取决于你的套餐。实际使用中，这些限额可能与
    ChatGPT 网站/应用体验不同，即使二者绑定的是同一个账户。

    OpenClaw 可以在
    `openclaw models status` 中显示当前可见的提供商使用量/配额窗口，但它不会把 ChatGPT 网页版的
    权限“发明”或归一化为直接 API 访问。如果你想使用直接的 OpenAI Platform
    计费/限额路径，请使用带 API key 的 `openai/*`。

  </Accordion>

  <Accordion title="你们支持 OpenAI subscription auth（Codex OAuth）吗？">
    支持。OpenClaw 完全支持 **OpenAI Code（Codex）subscription OAuth**。
    OpenAI 明确允许在像 OpenClaw 这样的外部工具/工作流中
    使用 subscription OAuth。新手引导也可以帮你执行 OAuth 流程。

    参见 [OAuth](/zh-CN/concepts/oauth)、[Model providers](/zh-CN/concepts/model-providers) 和 [新手引导（CLI）](/zh-CN/start/wizard)。

  </Accordion>

  <Accordion title="如何设置 Gemini CLI OAuth？">
    Gemini CLI 使用的是**插件认证流程**，而不是在 `openclaw.json` 中填写 client id 或 secret。

    步骤：

    1. 在本地安装 Gemini CLI，确保 `gemini` 在 `PATH` 中
       - Homebrew：`brew install gemini-cli`
       - npm：`npm install -g @google/gemini-cli`
    2. 启用插件：`openclaw plugins enable google`
    3. 登录：`openclaw models auth login --provider google-gemini-cli --set-default`
    4. 登录后的默认模型：`google-gemini-cli/gemini-3-flash-preview`
    5. 如果请求失败，请在 gateway host 上设置 `GOOGLE_CLOUD_PROJECT` 或 `GOOGLE_CLOUD_PROJECT_ID`

    这会将 OAuth token 存储在 gateway host 上的认证档案中。详情参见：[Model providers](/zh-CN/concepts/model-providers)。

  </Accordion>

  <Accordion title="本地模型适合日常闲聊吗？">
    通常不适合。OpenClaw 需要大上下文 + 强安全性；小模型容易截断并泄露内容。如果你一定要用，请在本地（LM Studio）运行你能使用的**最大**模型构建，并参见 [/gateway/local-models](/zh-CN/gateway/local-models)。更小/量化后的模型会增加 prompt injection 风险——参见 [Security](/zh-CN/gateway/security)。
  </Accordion>

  <Accordion title="如何让托管模型流量固定在特定地区？">
    选择锁定地区的 endpoint。OpenRouter 为 MiniMax、Kimi 和 GLM 提供 US 托管选项；选择 US 托管变体即可将数据保留在该区域内。你仍然可以通过使用 `models.mode: "merge"` 将 Anthropic/OpenAI 与这些模型一同列出，以便在遵守你所选地区化提供商的前提下保留后备能力。
  </Accordion>

  <Accordion title="安装这个必须买一台 Mac Mini 吗？">
    不需要。OpenClaw 可运行于 macOS 或 Linux（Windows 可通过 WSL2）。Mac mini 只是可选项——有些人
    买它作为常开主机，但小型 VPS、家庭服务器或 Raspberry Pi 级别的机器也可以。

    只有在你需要 **仅限 macOS 的工具**时才必须用 Mac。对于 iMessage，请使用 [BlueBubbles](/zh-CN/channels/bluebubbles)（推荐）——BlueBubbles server 可运行在任意 Mac 上，而 Gateway 网关可以运行在 Linux 或其他地方。如果你还想使用其他仅限 macOS 的工具，请在 Mac 上运行 Gateway 网关，或配对一个 macOS node。

    文档：[BlueBubbles](/zh-CN/channels/bluebubbles)、[Nodes](/zh-CN/nodes)、[Mac remote mode](/zh-CN/platforms/mac/remote)。

  </Accordion>

  <Accordion title="要支持 iMessage，我需要 Mac mini 吗？">
    你需要**某种已登录 Messages 的 macOS 设备**。它**不一定**是 Mac mini——
    任何 Mac 都可以。对于 iMessage，**请使用 [BlueBubbles](/zh-CN/channels/bluebubbles)**（推荐）——BlueBubbles server 运行在 macOS 上，而 Gateway 网关可以运行在 Linux 或其他地方。

    常见设置：

    - 在 Linux/VPS 上运行 Gateway 网关，并在任意已登录 Messages 的 Mac 上运行 BlueBubbles server。
    - 如果你想要最简单的单机设置，就把所有东西都运行在这台 Mac 上。

    文档：[BlueBubbles](/zh-CN/channels/bluebubbles)、[Nodes](/zh-CN/nodes)、
    [Mac remote mode](/zh-CN/platforms/mac/remote)。

  </Accordion>

  <Accordion title="如果我买一台 Mac mini 来运行 OpenClaw，我可以把它连接到我的 MacBook Pro 吗？">
    可以。**Mac mini 可以运行 Gateway 网关**，而你的 MacBook Pro 可以作为
    **node**（配套设备）连接。Nodes 不运行 Gateway 网关——它们提供额外
    能力，例如该设备上的 screen/camera/canvas 和 `system.run`。

    常见模式：

    - Gateway 网关在 Mac mini 上（常开）。
    - MacBook Pro 运行 macOS 应用或 node host，并与 Gateway 网关配对。
    - 使用 `openclaw nodes status` / `openclaw nodes list` 查看它。

    文档：[Nodes](/zh-CN/nodes)、[Nodes CLI](/cli/nodes)。

  </Accordion>

  <Accordion title="我可以用 Bun 吗？">
    **不推荐**使用 Bun。我们观察到运行时 bug，尤其是在 WhatsApp 和 Telegram 上。
    想要稳定的 Gateway 网关，请使用 **Node**。

    如果你仍然想试验 Bun，请只在非生产 Gateway 网关上使用，
    并且不要启用 WhatsApp/Telegram。

  </Accordion>

  <Accordion title="Telegram：allowFrom 里该填什么？">
    `channels.telegram.allowFrom` 指的是**真人发送者的 Telegram 用户 ID**（数字）。
    它不是 bot 用户名。

    新手引导接受 `@username` 输入，并会将其解析为数字 ID，但 OpenClaw 授权只使用数字 ID。

    更安全的方式（不依赖第三方 bot）：

    - 给你的 bot 发私信，然后运行 `openclaw logs --follow` 并读取 `from.id`。

    官方 Bot API：

    - 给你的 bot 发私信，然后调用 `https://api.telegram.org/bot<bot_token>/getUpdates` 并读取 `message.from.id`。

    第三方方式（隐私性较差）：

    - 给 `@userinfobot` 或 `@getidsbot` 发私信。

    参见 [/channels/telegram](/zh-CN/channels/telegram#access-control-and-activation)。

  </Accordion>

  <Accordion title="多个不同的 OpenClaw 实例能共用一个 WhatsApp 号码吗？">
    可以，通过**多智能体路由**实现。将每个发送者的 WhatsApp **私信**（peer `kind: "direct"`，发送者 E.164 如 `+15551234567`）绑定到不同的 `agentId`，这样每个人都会拥有自己的工作区和 session store。回复仍然来自**同一个 WhatsApp 账户**，而私信访问控制（`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`）对每个 WhatsApp 账户来说是全局的。参见 [Multi-Agent Routing](/zh-CN/concepts/multi-agent) 和 [WhatsApp](/zh-CN/channels/whatsapp)。
  </Accordion>

  <Accordion title='我能同时运行一个“快速聊天”智能体和一个“用于编程的 Opus”智能体吗？'>
    可以。使用多智能体路由：给每个智能体设置独立的默认模型，然后将入站路由（提供商账户或特定 peers）绑定到各自的智能体。示例配置见 [Multi-Agent Routing](/zh-CN/concepts/multi-agent)。另见 [Models](/zh-CN/concepts/models) 和 [Configuration](/zh-CN/gateway/configuration)。
  </Accordion>

  <Accordion title="Homebrew 能在 Linux 上使用吗？">
    可以。Homebrew 支持 Linux（Linuxbrew）。快速设置：

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    如果你通过 systemd 运行 OpenClaw，请确保服务的 PATH 包含 `/home/linuxbrew/.linuxbrew/bin`（或你的 brew prefix），以便 `brew` 安装的工具能在非登录 shell 中解析。
    近期版本还会在 Linux systemd 服务中预先添加常见用户 bin 目录（例如 `~/.local/bin`、`~/.npm-global/bin`、`~/.local/share/pnpm`、`~/.bun/bin`），并在设置时识别 `PNPM_HOME`、`NPM_CONFIG_PREFIX`、`BUN_INSTALL`、`VOLTA_HOME`、`ASDF_DATA_DIR`、`NVM_DIR` 和 `FNM_DIR`。

  </Accordion>

  <Accordion title="可修改的 git 安装和 npm install 有什么区别">
    - **可修改的（git）安装：**完整源码检出，可编辑，最适合贡献者。
      你需要在本地执行构建，并且可以修改代码/文档。
    - **npm install：**全局 CLI 安装，不包含仓库，最适合“只想直接运行”。
      更新来自 npm dist-tags。

    文档：[入门指南](/zh-CN/start/getting-started)、[Updating](/zh-CN/install/updating)。

  </Accordion>

  <Accordion title="之后还能在 npm 安装和 git 安装之间切换吗？">
    可以。安装另一种形式后，运行 Doctor，使 gateway 服务指向新的入口点。
    这**不会删除你的数据**——它只会更改 OpenClaw 的代码安装方式。你的状态
    （`~/.openclaw`）和工作区（`~/.openclaw/workspace`）不会受到影响。

    从 npm 切换到 git：

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    openclaw doctor
    openclaw gateway restart
    ```

    从 git 切换到 npm：

    ```bash
    npm install -g openclaw@latest
    openclaw doctor
    openclaw gateway restart
    ```

    Doctor 会检测 gateway 服务入口点不匹配，并提供将服务配置重写为匹配当前安装的选项（在自动化中使用 `--repair`）。

    备份建议：参见 [备份策略](#where-things-live-on-disk)。

  </Accordion>

  <Accordion title="我应该把 Gateway 网关运行在笔记本上，还是 VPS 上？">
    简短回答：**如果你想要 24/7 可靠性，请使用 VPS**。如果你想获得
    最低摩擦，并且不介意睡眠/重启，就在本地运行。

    **笔记本（本地 Gateway 网关）**

    - **优点：**无服务器成本，直接访问本地文件，可见的浏览器窗口。
    - **缺点：**睡眠/网络中断 = 断连，操作系统更新/重启会打断，必须保持唤醒。

    **VPS / 云端**

    - **优点：**常开、网络稳定、不受笔记本休眠影响、更容易长期运行。
    - **缺点：**通常是无头运行（需使用截图），只能远程访问文件，你必须通过 SSH 更新。

    **OpenClaw 特别说明：**WhatsApp/Telegram/Slack/Mattermost/Discord 在 VPS 上都能正常工作。唯一真正的取舍是**无头浏览器**还是可见窗口。参见 [Browser](/zh-CN/tools/browser)。

    **推荐默认方案：**如果你之前遇到 Gateway 网关断开连接，用 VPS。若你经常主动使用 Mac，并希望访问本地文件或进行带可见浏览器的 UI 自动化，本地运行也非常合适。

  </Accordion>

  <Accordion title="让 OpenClaw 运行在专用机器上有多重要？">
    不是必须，但**为了可靠性和隔离性，推荐这样做**。

    - **专用主机（VPS/Mac mini/Pi）：**常开、更少睡眠/重启中断、更清晰的权限、更容易长期运行。
    - **共享笔记本/台式机：**非常适合测试和主动使用，但机器睡眠或更新时会出现暂停。

    如果你想兼顾两者，把 Gateway 网关放在专用主机上，再将你的笔记本配对为 **node**，用于本地 screen/camera/exec 工具。参见 [Nodes](/zh-CN/nodes)。
    安全指南请阅读 [Security](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="最低 VPS 要求和推荐操作系统是什么？">
    OpenClaw 很轻量。对于基础的 Gateway 网关 + 一个聊天渠道：

    - **绝对最低配置：**1 vCPU、1GB RAM、约 500MB 磁盘。
    - **推荐配置：**1-2 vCPU、2GB RAM 或更多余量（日志、媒体、多渠道）。Node 工具和浏览器自动化可能比较吃资源。

    操作系统：使用 **Ubuntu LTS**（或任意现代 Debian/Ubuntu）。Linux 安装路径在这些系统上测试最充分。

    文档：[Linux](/zh-CN/platforms/linux)、[VPS hosting](/zh-CN/vps)。

  </Accordion>

  <Accordion title="我可以在 VM 中运行 OpenClaw 吗？需要什么配置？">
    可以。把 VM 当作 VPS 看待：它需要始终开启、可访问，并且有足够的
    RAM 供 Gateway 网关和你启用的各个渠道使用。

    基础建议：

    - **绝对最低配置：**1 vCPU、1GB RAM。
    - **推荐配置：**如果你运行多个渠道、浏览器自动化或媒体工具，使用 2GB RAM 或更多。
    - **操作系统：**Ubuntu LTS 或其他现代 Debian/Ubuntu。

    如果你使用的是 Windows，**WSL2 是最容易上手的 VM 风格设置**，并且工具兼容性最好。
    参见 [Windows](/zh-CN/platforms/windows)、[VPS hosting](/zh-CN/vps)。
    如果你在 VM 中运行 macOS，请参见 [macOS VM](/zh-CN/install/macos-vm)。

  </Accordion>
</AccordionGroup>

## 什么是 OpenClaw？

<AccordionGroup>
  <Accordion title="用一段话介绍，什么是 OpenClaw？">
    OpenClaw 是一个运行在你自己设备上的个人 AI 助手。它会在你已经使用的消息界面中回复你（WhatsApp、Telegram、Slack、Mattermost、Discord、Google Chat、Signal、iMessage、WebChat，以及像 QQ Bot 这样的内置渠道插件），并且在支持的平台上还能提供语音 + 实时 Canvas。**Gateway 网关**是始终在线的控制平面；这个助手本身才是产品。
  </Accordion>

  <Accordion title="价值主张">
    OpenClaw 不是“只是一个 Claude 包装器”。它是一个**本地优先的控制平面**，让你可以在**自己的硬件上**
    运行一个强大的助手，通过你已经在使用的聊天应用访问，
    并具备有状态会话、记忆和工具——而无需将你的工作流控制权交给托管式
    SaaS。

    亮点：

    - **你的设备，你的数据：**在你想要的任何地方运行 Gateway 网关（Mac、Linux、VPS），并将
      工作区 + 会话历史保存在本地。
    - **真实渠道，而不是 Web 沙箱：**WhatsApp/Telegram/Slack/Discord/Signal/iMessage 等，
      外加在支持平台上的移动语音和 Canvas。
    - **模型无关：**使用 Anthropic、OpenAI、MiniMax、OpenRouter 等，并支持按智能体路由
      和故障切换。
    - **纯本地选项：**运行本地模型，这样如果你愿意，**所有数据都可以留在你的设备上**。
    - **多智能体路由：**按渠道、账户或任务分离智能体，每个智能体都有自己的
      工作区和默认设置。
    - **开源且可修改：**无需厂商锁定即可检查、扩展和自托管。

    文档：[Gateway 网关](/zh-CN/gateway)、[Channels](/zh-CN/channels)、[Multi-agent](/zh-CN/concepts/multi-agent)、
    [Memory](/zh-CN/concepts/memory)。

  </Accordion>

  <Accordion title="我刚设置好——首先应该做什么？">
    适合作为开场的项目：

    - 建一个网站（WordPress、Shopify，或简单的静态站点）。
    - 做一个移动应用原型（大纲、界面、API 计划）。
    - 整理文件和文件夹（清理、命名、打标签）。
    - 连接 Gmail 并自动化摘要或后续跟进。

    它可以处理大型任务，但如果你将任务拆分成多个阶段，
    并使用 sub agents 并行处理，效果会更好。

  </Accordion>

  <Accordion title="OpenClaw 最常见的五个日常用例是什么？">
    日常收益通常体现在：

    - **个人简报：**总结你关心的收件箱、日历和新闻。
    - **调研与起草：**快速调研、摘要，以及邮件或文档的初稿。
    - **提醒与跟进：**由 cron 或 heartbeat 驱动的提醒和清单。
    - **浏览器自动化：**填写表单、收集数据、重复执行 Web 任务。
    - **跨设备协作：**从手机发送任务，让 Gateway 网关在服务器上执行，然后在聊天中收到结果。

  </Accordion>

  <Accordion title="OpenClaw 能帮助做 SaaS 的获客、外联、广告和博客吗？">
    对于**研究、筛选和起草**，可以。它可以扫描站点、建立候选清单、
    总结潜在客户，并撰写外联或广告文案草稿。

    对于**外联或广告投放**，请让人类参与审批。避免垃圾信息，遵守当地法律和
    平台政策，并在发送前审查所有内容。最安全的模式是让
    OpenClaw 起草、由你批准。

    文档：[Security](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="相比 Claude Code，它在 Web 开发上的优势是什么？">
    OpenClaw 是一个**个人助手**和协作层，不是 IDE 替代品。想在仓库内获得
    最快的直接编码循环，请使用 Claude Code 或 Codex。想获得
    持久记忆、跨设备访问和工具编排时，请使用 OpenClaw。

    优势：

    - **跨会话持久记忆 + 工作区**
    - **多平台访问**（WhatsApp、Telegram、TUI、WebChat）
    - **工具编排**（浏览器、文件、调度、hooks）
    - **始终在线的 Gateway 网关**（运行在 VPS 上，随时随地交互）
    - 用于本地 browser/screen/camera/exec 的 **Nodes**

    展示：[https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills 和自动化

<AccordionGroup>
  <Accordion title="如何自定义 skills 而不把仓库弄脏？">
    使用托管覆盖，而不是直接编辑仓库中的副本。将你的改动放到 `~/.openclaw/skills/<name>/SKILL.md` 中（或通过 `~/.openclaw/openclaw.json` 里的 `skills.load.extraDirs` 添加一个文件夹）。优先级顺序为 `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → 内置 → `skills.load.extraDirs`，因此托管覆盖仍会优先于内置 skills，而无需修改 git。如果你需要全局安装这个 skill，但只想让某些智能体可见，请将共享副本保存在 `~/.openclaw/skills` 中，并使用 `agents.defaults.skills` 和 `agents.list[].skills` 控制可见性。只有值得上游合并的修改，才应该放在仓库中并以 PR 形式提交。
  </Accordion>

  <Accordion title="可以从自定义文件夹加载 skills 吗？">
    可以。通过 `~/.openclaw/openclaw.json` 中的 `skills.load.extraDirs` 添加额外目录（最低优先级）。默认优先级为 `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → 内置 → `skills.load.extraDirs`。`clawhub` 默认会安装到 `./skills`，OpenClaw 会在下一个会话中将其视为 `<workspace>/skills`。如果该 skill 只应对某些智能体可见，请结合 `agents.defaults.skills` 或 `agents.list[].skills` 一起使用。
  </Accordion>

  <Accordion title="如何为不同任务使用不同模型？">
    当前支持的模式有：

    - **Cron jobs**：隔离作业可以为每个作业设置 `model` 覆盖。
    - **Sub-agents**：将任务路由到具有不同默认模型的独立智能体。
    - **按需切换**：随时使用 `/model` 切换当前会话模型。

    参见 [Cron jobs](/zh-CN/automation/cron-jobs)、[Multi-Agent Routing](/zh-CN/concepts/multi-agent) 和 [Slash commands](/zh-CN/tools/slash-commands)。

  </Accordion>

  <Accordion title="机器人在执行重任务时会卡住。如何把这些任务卸载出去？">
    对于长时间运行或并行任务，请使用 **sub-agents**。Sub-agents 在它们自己的会话中运行，
    返回摘要，并保持你的主聊天响应迅速。

    你可以让机器人“为此任务生成一个 sub-agent”，或使用 `/subagents`。
    在聊天中使用 `/status` 可以查看 Gateway 网关当前正在做什么（以及它是否忙碌）。

    Token 提示：长任务和 sub-agents 都会消耗 token。如果你担心成本，可以通过
    `agents.defaults.subagents.model` 为 sub-agents 设置更便宜的模型。

    文档：[Sub-agents](/zh-CN/tools/subagents)、[Background Tasks](/zh-CN/automation/tasks)。

  </Accordion>

  <Accordion title="Discord 上基于线程绑定的 subagent 会话如何工作？">
    使用线程绑定。你可以将一个 Discord 线程绑定到某个 subagent 或 session 目标，这样该线程中的后续消息就会保持在该绑定的会话上。

    基本流程：

    - 使用 `sessions_spawn` 并设置 `thread: true` 启动（可选设置 `mode: "session"` 以支持持久后续跟进）。
    - 或者用 `/focus <target>` 手动绑定。
    - 使用 `/agents` 检查绑定状态。
    - 使用 `/session idle <duration|off>` 和 `/session max-age <duration|off>` 控制自动取消聚焦。
    - 使用 `/unfocus` 解除线程绑定。

    所需配置：

    - 全局默认值：`session.threadBindings.enabled`、`session.threadBindings.idleHours`、`session.threadBindings.maxAgeHours`。
    - Discord 覆盖：`channels.discord.threadBindings.enabled`、`channels.discord.threadBindings.idleHours`、`channels.discord.threadBindings.maxAgeHours`。
    - 启动时自动绑定：设置 `channels.discord.threadBindings.spawnSubagentSessions: true`。

    文档：[Sub-agents](/zh-CN/tools/subagents)、[Discord](/zh-CN/channels/discord)、[Configuration Reference](/zh-CN/gateway/configuration-reference)、[Slash commands](/zh-CN/tools/slash-commands)。

  </Accordion>

  <Accordion title="一个 subagent 已完成，但完成更新发到了错误位置，或者根本没发出来。我该检查什么？">
    先检查解析后的 requester route：

    - 完成模式的 subagent 投递会优先使用任何已绑定的线程或会话路由（如果存在）。
    - 如果完成来源只携带了一个渠道，OpenClaw 会退回到 requester 会话中存储的路由（`lastChannel` / `lastTo` / `lastAccountId`），这样直接投递仍然可以成功。
    - 如果既没有绑定路由，也没有可用的已存储路由，直接投递可能失败，结果会退回到队列式会话投递，而不是立即发到聊天中。
    - 无效或陈旧的目标仍可能导致回退到队列，或最终投递失败。
    - 如果子会话最后一个可见的 assistant 回复正好是静默 token `NO_REPLY` / `no_reply`，或恰好是 `ANNOUNCE_SKIP`，OpenClaw 会有意抑制 announce，而不是发出陈旧的早期进度。
    - 如果子会话只执行了工具调用就超时，announce 可能会将其压缩成简短的部分进度摘要，而不是重放原始工具输出。

    调试：

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    文档：[Sub-agents](/zh-CN/tools/subagents)、[Background Tasks](/zh-CN/automation/tasks)、[Session Tools](/zh-CN/concepts/session-tool)。

  </Accordion>

  <Accordion title="Cron 或提醒没有触发。我该检查什么？">
    Cron 在 Gateway 网关进程内部运行。如果 Gateway 网关没有持续运行，
    定时作业就不会执行。

    检查清单：

    - 确认 cron 已启用（`cron.enabled`），并且未设置 `OPENCLAW_SKIP_CRON`。
    - 检查 Gateway 网关是否持续 24/7 运行（没有睡眠/重启）。
    - 验证作业的时区设置（`--tz` 与主机时区）。

    调试：

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    文档：[Cron jobs](/zh-CN/automation/cron-jobs)、[Automation & Tasks](/zh-CN/automation)。

  </Accordion>

  <Accordion title="Cron 触发了，但什么都没有发送到渠道。为什么？">
    先检查投递模式：

    - `--no-deliver` / `delivery.mode: "none"` 表示预期不会有外部消息。
    - 缺失或无效的 announce 目标（`channel` / `to`）意味着运行器跳过了出站投递。
    - 渠道认证失败（`unauthorized`、`Forbidden`）意味着运行器尝试投递了，但凭证阻止了它。
    - 一个静默的隔离结果（只有 `NO_REPLY` / `no_reply`）会被视为有意不投递，因此运行器也会抑制队列式回退投递。

    对于隔离的 cron 作业，运行器负责最终投递。智能体应当
    返回一个纯文本摘要，供运行器发送。`--no-deliver` 会将
    结果保留在内部；它并不会让智能体改为通过
    message 工具直接发送。

    调试：

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    文档：[Cron jobs](/zh-CN/automation/cron-jobs)、[Background Tasks](/zh-CN/automation/tasks)。

  </Accordion>

  <Accordion title="为什么一个隔离的 cron 运行会切换模型或重试一次？">
    这通常是实时模型切换路径，而不是重复调度。

    隔离 cron 在活动运行抛出 `LiveSessionModelSwitchError` 时，
    可以持久化一个运行时模型切换并重试。重试会保留切换后的
    provider/model；如果切换还携带了新的 auth profile override，cron
    也会在重试前一并持久化它。

    相关选择规则：

    - 如果适用，Gmail hook 模型覆盖优先级最高。
    - 然后是每个作业的 `model`。
    - 然后是任何已存储的 cron-session 模型覆盖。
    - 最后才是正常的智能体/默认模型选择。

    重试循环是有上限的。在初次尝试加上 2 次切换重试之后，
    cron 会中止，而不是无限循环。

    调试：

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    文档：[Cron jobs](/zh-CN/automation/cron-jobs)、[cron CLI](/cli/cron)。

  </Accordion>

  <Accordion title="如何在 Linux 上安装 Skills？">
    使用原生 `openclaw skills` 命令，或将 skills 放入你的工作区。macOS Skills UI 在 Linux 上不可用。
    浏览 skills： [https://clawhub.ai](https://clawhub.ai)。

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    原生 `openclaw skills install` 会写入当前工作区的 `skills/`
    目录。只有当你想发布或同步自己的 skills 时，才需要单独安装 `clawhub` CLI。
    对于多个智能体共享安装，请将 skill 放在 `~/.openclaw/skills` 下，并使用 `agents.defaults.skills` 或
    `agents.list[].skills`（如果你想限制哪些智能体可以看到它）。

  </Accordion>

  <Accordion title="OpenClaw 能按计划运行任务，或持续在后台运行吗？">
    可以。使用 Gateway 网关调度器：

    - **Cron jobs**：用于计划任务或周期性任务（跨重启持久化）。
    - **Heartbeat**：用于“主会话”的周期性检查。
    - **Isolated jobs**：用于自主智能体，发送摘要或投递到聊天。

    文档：[Cron jobs](/zh-CN/automation/cron-jobs)、[Automation & Tasks](/zh-CN/automation)、
    [Heartbeat](/zh-CN/gateway/heartbeat)。

  </Accordion>

  <Accordion title="我能在 Linux 上运行仅限 Apple macOS 的 skills 吗？">
    不能直接运行。macOS skills 受 `metadata.openclaw.os` 和所需二进制文件共同控制，且只有当这些 skill 在 **Gateway 网关主机**上符合条件时，才会出现在 system prompt 中。在 Linux 上，仅限 `darwin` 的 skills（例如 `apple-notes`、`apple-reminders`、`things-mac`）不会加载，除非你覆盖这层条件判断。

    你有三种受支持的模式：

    **选项 A - 在 Mac 上运行 Gateway 网关（最简单）。**
    在存在这些 macOS 二进制文件的地方运行 Gateway 网关，然后从 Linux 通过 [remote mode](#gateway-ports-already-running-and-remote-mode) 或 Tailscale 连接。由于 Gateway 网关主机是 macOS，这些 skills 会正常加载。

    **选项 B - 使用 macOS node（无需 SSH）。**
    在 Linux 上运行 Gateway 网关，配对一个 macOS node（菜单栏应用），并将 Mac 上的 **Node Run Commands** 设为“Always Ask”或“Always Allow”。当 node 上存在所需二进制文件时，OpenClaw 可以将仅限 macOS 的 skills 视为符合条件。智能体会通过 `nodes` 工具运行这些 skills。如果你选择“Always Ask”，在提示中批准“Always Allow”会将该命令加入 allowlist。

    **选项 C - 通过 SSH 代理 macOS 二进制文件（高级）。**
    让 Gateway 网关继续运行在 Linux 上，但把所需 CLI 二进制文件解析为 SSH 包装器，并在 Mac 上执行。然后覆盖该 skill，使其允许 Linux，以保持其可用。

    1. 为二进制文件创建 SSH 包装器（示例：用于 Apple Notes 的 `memo`）：

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. 将该包装器放在 Linux 主机的 `PATH` 中（例如 `~/bin/memo`）。
    3. 覆盖该 skill 的元数据（工作区中或 `~/.openclaw/skills` 中）以允许 Linux：

       ```markdown
       ---
       name: apple-notes
       description: 通过 macOS 上的 memo CLI 管理 Apple Notes。
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. 开始一个新会话，以刷新 skills 快照。

  </Accordion>

  <Accordion title="你们有 Notion 或 HeyGen 集成吗？">
    目前没有内置。

    可选方案：

    - **自定义 skill / plugin：**最适合稳定可靠的 API 访问（Notion/HeyGen 都有 API）。
    - **浏览器自动化：**无需写代码，但速度更慢，也更脆弱。

    如果你希望按客户保留上下文（代理机构工作流），一个简单模式是：

    - 每个客户一个 Notion 页面（上下文 + 偏好 + 当前工作）。
    - 让智能体在会话开始时获取该页面。

    如果你想要原生集成，请提交功能请求，或构建一个面向这些 API 的
    skill。

    安装 skills：

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    原生安装会落到当前工作区的 `skills/` 目录。对于跨智能体共享的 skills，请将它们放在 `~/.openclaw/skills/<name>/SKILL.md` 中。如果只想让某些智能体看到共享安装，请配置 `agents.defaults.skills` 或 `agents.list[].skills`。有些 skills 依赖通过 Homebrew 安装的二进制文件；在 Linux 上这意味着 Linuxbrew（见上方的 Homebrew Linux 常见问题条目）。参见 [Skills](/zh-CN/tools/skills)、[Skills 配置](/zh-CN/tools/skills-config) 和 [ClawHub](/zh-CN/tools/clawhub)。

  </Accordion>

  <Accordion title="如何让 OpenClaw 使用我当前已登录的 Chrome？">
    使用内置的 `user` 浏览器 profile，它通过 Chrome DevTools MCP 进行附着：

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    如果你想用自定义名称，请创建一个显式 MCP profile：

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    这条路径仅限主机本地。如果 Gateway 网关运行在别处，你需要在浏览器所在机器上运行一个 node host，或者改用远程 CDP。

    `existing-session` / `user` 当前限制：

    - 操作基于 ref 驱动，而不是基于 CSS selector
    - 上传需要 `ref` / `inputRef`，目前一次只支持一个文件
    - `responsebody`、PDF 导出、下载拦截和批量操作仍需要托管浏览器或原始 CDP profile

  </Accordion>
</AccordionGroup>

## 沙箱隔离和记忆

<AccordionGroup>
  <Accordion title="有专门的沙箱隔离文档吗？">
    有。参见 [沙箱隔离](/zh-CN/gateway/sandboxing)。对于 Docker 专用设置（在 Docker 中运行完整 Gateway 网关或沙箱镜像），参见 [Docker](/zh-CN/install/docker)。
  </Accordion>

  <Accordion title="Docker 感觉功能有限——如何启用完整功能？">
    默认镜像以安全优先方式构建，并以 `node` 用户身份运行，因此它
    不包含系统包、Homebrew 或内置浏览器。若要获得更完整的设置：

    - 使用 `OPENCLAW_HOME_VOLUME` 持久化 `/home/node`，以便缓存保留。
    - 使用 `OPENCLAW_DOCKER_APT_PACKAGES` 将系统依赖烘焙进镜像。
    - 通过内置 CLI 安装 Playwright 浏览器：
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - 设置 `PLAYWRIGHT_BROWSERS_PATH`，并确保该路径已持久化。

    文档：[Docker](/zh-CN/install/docker)、[Browser](/zh-CN/tools/browser)。

  </Accordion>

  <Accordion title="我能否让私信保持个人化，同时让群组公开/沙箱隔离，并只用一个智能体？">
    可以——前提是你的私密流量是**私信**，公开流量是**群组**。

    使用 `agents.defaults.sandbox.mode: "non-main"`，这样群组/渠道会话（非 main 键）会在 Docker 中运行，而主私信会话保持在主机上运行。然后通过 `tools.sandbox.tools` 限制沙箱隔离会话中可用的工具。

    设置演练 + 示例配置：[群组：个人私信 + 公开群组](/zh-CN/channels/groups#pattern-personal-dms-public-groups-single-agent)

    关键配置参考：[Gateway configuration](/zh-CN/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="如何将主机文件夹绑定进沙箱？">
    将 `agents.defaults.sandbox.docker.binds` 设置为 `["host:path:mode"]`（例如 `"/home/user/src:/src:ro"`）。全局与每智能体 binds 会合并；当 `scope: "shared"` 时，每智能体 bind 会被忽略。对任何敏感内容都使用 `:ro`，并记住 bind 会绕过沙箱文件系统边界。

    OpenClaw 会同时根据规范化路径和通过最深现有祖先解析得到的 canonical path 验证 bind 源。这意味着即使最后一个路径段尚不存在，symlink 父级逃逸也仍会以关闭方式失败，并且在 symlink 解析后仍会应用 allowed-root 检查。

    示例和安全说明请参见 [沙箱隔离](/zh-CN/gateway/sandboxing#custom-bind-mounts) 和 [Sandbox vs Tool Policy vs Elevated](/zh-CN/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check)。

  </Accordion>

  <Accordion title="记忆是如何工作的？">
    OpenClaw 的记忆就是智能体工作区中的 Markdown 文件：

    - `memory/YYYY-MM-DD.md` 中的每日日志
    - `MEMORY.md` 中整理后的长期笔记（仅 main/private 会话）

    OpenClaw 还会执行一次**静默的压缩前记忆刷新**，提醒模型
    在自动压缩前写下持久笔记。只有当工作区
    可写时才会执行；只读沙箱会跳过它。参见 [Memory](/zh-CN/concepts/memory)。

  </Accordion>

  <Accordion title="记忆老是忘事。如何让它记住？">
    让机器人**把这个事实写入记忆**。长期笔记应该写入 `MEMORY.md`，
    短期上下文写入 `memory/YYYY-MM-DD.md`。

    这是我们仍在持续改进的领域。提醒模型去存储记忆会很有帮助；
    它会知道该怎么做。如果它还是忘记，请确认 Gateway 网关每次运行
    使用的是同一个工作区。

    文档：[Memory](/zh-CN/concepts/memory)、[Agent workspace](/zh-CN/concepts/agent-workspace)。

  </Accordion>

  <Accordion title="记忆会永久保存吗？有什么限制？">
    记忆文件保存在磁盘上，除非你删除它们，否则会一直存在。限制来自你的
    存储空间，而不是模型。**会话上下文**仍然受模型
    上下文窗口限制，因此长对话可能会压缩或截断。这就是
    记忆搜索存在的原因——它只会把相关部分重新拉回上下文。

    文档：[Memory](/zh-CN/concepts/memory)、[Context](/zh-CN/concepts/context)。

  </Accordion>

  <Accordion title="语义记忆搜索需要 OpenAI API key 吗？">
    只有在你使用 **OpenAI embeddings** 时才需要。Codex OAuth 只覆盖 chat/completions，
    **不**授予 embeddings 访问权限，所以**使用 Codex 登录（OAuth 或
    Codex CLI 登录）**并不能帮助启用语义记忆搜索。OpenAI embeddings
    仍然需要真实的 API key（`OPENAI_API_KEY` 或 `models.providers.openai.apiKey`）。

    如果你没有显式设置 provider，OpenClaw 会在
    可以解析到 API key 时自动选择一个 provider（auth profiles、`models.providers.*.apiKey` 或环境变量）。
    如果解析到 OpenAI key，它会优先使用 OpenAI；否则如果解析到 Gemini key
    则使用 Gemini；接着是 Voyage，然后是 Mistral。如果没有可用的远程 key，记忆
    搜索会保持禁用，直到你完成配置。如果你配置了本地模型路径
    且该路径存在，OpenClaw
    会优先选择 `local`。当你显式设置
    `memorySearch.provider = "ollama"` 时，也支持 Ollama。

    如果你更希望保持本地，请设置 `memorySearch.provider = "local"`（并可选
    设置 `memorySearch.fallback = "none"`）。如果你想使用 Gemini embeddings，请设置
    `memorySearch.provider = "gemini"` 并提供 `GEMINI_API_KEY`（或
    `memorySearch.remote.apiKey`）。我们支持 **OpenAI、Gemini、Voyage、Mistral、Ollama 或 local** embedding
    模型——设置细节请参见 [Memory](/zh-CN/concepts/memory)。

  </Accordion>
</AccordionGroup>

## 磁盘上的文件位置

<AccordionGroup>
  <Accordion title="OpenClaw 使用的所有数据都保存在本地吗？">
    不是——**OpenClaw 的状态是本地的**，但**外部服务仍会看到你发送给它们的内容**。

    - **默认本地：**会话、记忆文件、配置和工作区保存在 gateway host 上
      （`~/.openclaw` + 你的工作区目录）。
    - **必须远程：**你发送给模型提供商（Anthropic/OpenAI 等）的消息会进入
      它们的 API，而聊天平台（WhatsApp/Telegram/Slack 等）会在它们自己的
      服务器上存储消息数据。
    - **你可以控制暴露范围：**使用本地模型可以让 prompts 保留在你的机器上，但渠道
      流量仍然会经过各自渠道的服务器。

    相关内容：[Agent workspace](/zh-CN/concepts/agent-workspace)、[Memory](/zh-CN/concepts/memory)。

  </Accordion>

  <Accordion title="OpenClaw 将数据存储在哪里？">
    所有内容都位于 `$OPENCLAW_STATE_DIR` 下（默认：`~/.openclaw`）：

    | 路径                                                            | 用途                                                               |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | 主配置（JSON5）                                                    |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | 旧版 OAuth 导入（首次使用时复制到 auth profiles）                  |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | 认证档案（OAuth、API 密钥，以及可选的 `keyRef`/`tokenRef`）        |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | `file` SecretRef provider 使用的可选文件后端 secret 负载           |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | 旧版兼容文件（静态 `api_key` 条目已清理）                          |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | 提供商状态（例如 `whatsapp/<accountId>/creds.json`）               |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | 每个智能体的状态（agentDir + sessions）                            |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | 对话历史与状态（按智能体划分）                                     |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | 会话元数据（按智能体划分）                                         |

    旧版单智能体路径：`~/.openclaw/agent/*`（由 `openclaw doctor` 迁移）。

    你的**工作区**（AGENTS.md、记忆文件、skills 等）是分开的，通过 `agents.defaults.workspace` 配置（默认：`~/.openclaw/workspace`）。

  </Accordion>

  <Accordion title="AGENTS.md / SOUL.md / USER.md / MEMORY.md 应该放在哪里？">
    这些文件位于**智能体工作区**中，而不是 `~/.openclaw`。

    - **工作区（每个智能体）**：`AGENTS.md`、`SOUL.md`、`IDENTITY.md`、`USER.md`、
      `MEMORY.md`（如果没有 `MEMORY.md`，则回退到旧版 `memory.md`）、
      `memory/YYYY-MM-DD.md`、可选的 `HEARTBEAT.md`。
    - **状态目录（`~/.openclaw`）**：配置、渠道/提供商状态、认证档案、会话、日志，
      以及共享 skills（`~/.openclaw/skills`）。

    默认工作区是 `~/.openclaw/workspace`，可通过以下方式配置：

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    如果机器人在重启后“忘事”，请确认 Gateway 网关每次启动
    使用的是同一个工作区（并记住：远程模式使用的是**gateway host**
    的工作区，而不是你本地笔记本的工作区）。

    提示：如果你希望某种行为或偏好长期保留，请让机器人**把它写进
    AGENTS.md 或 MEMORY.md**，而不要只依赖聊天历史。

    参见 [Agent workspace](/zh-CN/concepts/agent-workspace) 和 [Memory](/zh-CN/concepts/memory)。

  </Accordion>

  <Accordion title="推荐的备份策略">
    将你的**智能体工作区**放进一个**私有** git 仓库，并将其备份到某个
    私有位置（例如 GitHub private）。这样可以保存记忆 + AGENTS/SOUL/USER
    文件，并让你之后恢复这个助手的“心智”。

    **不要**提交 `~/.openclaw` 下的任何内容（credentials、sessions、tokens 或加密后的 secrets payloads）。
    如果你需要完整恢复，请分别备份工作区和状态目录
    （参见上方关于迁移的问题）。

    文档：[Agent workspace](/zh-CN/concepts/agent-workspace)。

  </Accordion>

  <Accordion title="如何彻底卸载 OpenClaw？">
    请参见专门指南：[Uninstall](/zh-CN/install/uninstall)。
  </Accordion>

  <Accordion title="智能体可以在工作区之外工作吗？">
    可以。工作区是**默认 cwd** 和记忆锚点，而不是硬性沙箱。
    相对路径会在工作区内解析，但绝对路径仍可访问主机上的其他
    位置，除非启用了沙箱隔离。如果你需要隔离，请使用
    [`agents.defaults.sandbox`](/zh-CN/gateway/sandboxing) 或每个智能体单独的沙箱设置。如果你
    想让某个仓库成为默认工作目录，请把该智能体的
    `workspace` 指向仓库根目录。OpenClaw 仓库本身只是源码；除非你明确想让智能体在其中工作，否则请让
    工作区与其分开。

    示例（将仓库作为默认 cwd）：

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Remote mode：session store 在哪里？">
    会话状态由**gateway host** 持有。如果你处于 remote mode，你关心的 session store 位于远程机器上，而不是本地笔记本。参见 [Session management](/zh-CN/concepts/session)。
  </Accordion>
</AccordionGroup>

## 配置基础

<AccordionGroup>
  <Accordion title="配置是什么格式？在哪里？">
    OpenClaw 会从 `$OPENCLAW_CONFIG_PATH` 读取可选的 **JSON5** 配置（默认：`~/.openclaw/openclaw.json`）：

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    如果文件缺失，它会使用相对安全的默认值（包括默认工作区 `~/.openclaw/workspace`）。

  </Accordion>

  <Accordion title='我设置了 gateway.bind: "lan"（或 "tailnet"），现在没有任何监听 / UI 显示 unauthorized'>
    非 loopback 绑定**需要有效的 gateway auth 路径**。实际意味着：

    - shared-secret auth：token 或 password
    - 在正确配置的非 loopback、具备身份感知的反向代理后使用 `gateway.auth.mode: "trusted-proxy"`

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    说明：

    - `gateway.remote.token` / `.password` 本身**不会**启用本地 gateway auth。
    - 只有当 `gateway.auth.*` 未设置时，本地调用路径才会把 `gateway.remote.*` 用作回退。
    - 对于 password auth，请改为设置 `gateway.auth.mode: "password"` 加上 `gateway.auth.password`（或 `OPENCLAW_GATEWAY_PASSWORD`）。
    - 如果 `gateway.auth.token` / `gateway.auth.password` 通过 SecretRef 显式配置但未解析成功，解析将以关闭方式失败（不会由 remote 回退悄悄掩盖）。
    - shared-secret 的 Control UI 设置通过 `connect.params.auth.token` 或 `connect.params.auth.password` 进行认证（保存在 app/UI 设置中）。像 Tailscale Serve 或 `trusted-proxy` 这样的具身份模式则使用请求标头。避免把 shared secrets 放进 URL。
    - 当使用 `gateway.auth.mode: "trusted-proxy"` 时，同主机 loopback 反向代理**仍然不能**满足 trusted-proxy auth。trusted proxy 必须是已配置的非 loopback 来源。

  </Accordion>

  <Accordion title="为什么现在在 localhost 上也需要 token？">
    OpenClaw 默认强制启用 gateway auth，包括 loopback。按正常默认路径，这意味着 token auth：如果没有显式配置 auth 路径，gateway 启动时会解析为 token 模式并自动生成一个 token，将其保存到 `gateway.auth.token` 中，因此**本地 WS 客户端也必须认证**。这可以阻止同一台机器上的其他本地进程调用 Gateway 网关。

    如果你更喜欢其他 auth 路径，可以显式选择 password 模式（或在非 loopback 的身份感知反向代理场景中使用 `trusted-proxy`）。如果你**确实**想开放 loopback，请在配置中显式设置 `gateway.auth.mode: "none"`。Doctor 也可以随时为你生成 token：`openclaw doctor --generate-gateway-token`。

  </Accordion>

  <Accordion title="修改配置后需要重启吗？">
    Gateway 网关会监控配置并支持热重载：

    - `gateway.reload.mode: "hybrid"`（默认）：安全改动热应用，关键改动则重启
    - 也支持 `hot`、`restart`、`off`

  </Accordion>

  <Accordion title="如何禁用 CLI 里那些有趣的标语？">
    在配置中设置 `cli.banner.taglineMode`：

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`：隐藏标语文本，但保留横幅标题/版本行。
    - `default`：每次都使用 `All your chats, one OpenClaw.`。
    - `random`：轮换显示有趣/季节性标语（默认行为）。
    - 如果你想完全不显示横幅，请设置环境变量 `OPENCLAW_HIDE_BANNER=1`。

  </Accordion>

  <Accordion title="如何启用 Web 搜索（以及 Web 抓取）？">
    `web_fetch` 无需 API key。`web_search` 取决于你选择的
    provider：

    - Brave、Exa、Firecrawl、Gemini、Grok、Kimi、MiniMax Search、Perplexity 和 Tavily 等基于 API 的 provider，需要按其常规方式配置 API key。
    - Ollama Web 搜索 不需要 key，但它会使用你配置的 Ollama host，并且要求执行 `ollama signin`。
    - DuckDuckGo 不需要 key，但它是一个非官方的基于 HTML 的集成。
    - SearXNG 不需要 key/可自托管；配置 `SEARXNG_BASE_URL` 或 `plugins.entries.searxng.config.webSearch.baseUrl`。

    **推荐方式：**运行 `openclaw configure --section web` 并选择一个 provider。
    环境变量替代方案：

    - Brave：`BRAVE_API_KEY`
    - Exa：`EXA_API_KEY`
    - Firecrawl：`FIRECRAWL_API_KEY`
    - Gemini：`GEMINI_API_KEY`
    - Grok：`XAI_API_KEY`
    - Kimi：`KIMI_API_KEY` 或 `MOONSHOT_API_KEY`
    - MiniMax Search：`MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY` 或 `MINIMAX_API_KEY`
    - Perplexity：`PERPLEXITY_API_KEY` 或 `OPENROUTER_API_KEY`
    - SearXNG：`SEARXNG_BASE_URL`
    - Tavily：`TAVILY_API_KEY`

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
        },
        tools: {
          web: {
            search: {
              enabled: true,
              provider: "brave",
              maxResults: 5,
            },
            fetch: {
              enabled: true,
              provider: "firecrawl", // 可选；省略则自动检测
            },
          },
        },
    }
    ```

    现在，provider 专属的 Web 搜索配置位于 `plugins.entries.<plugin>.config.webSearch.*`。
    旧版 `tools.web.search.*` provider 路径仍会临时加载以兼容旧配置，但不应再用于新配置。
    Firecrawl 的 web-fetch 回退配置位于 `plugins.entries.firecrawl.config.webFetch.*`。

    说明：

    - 如果你使用 allowlists，请添加 `web_search`/`web_fetch`/`x_search` 或 `group:web`。
    - `web_fetch` 默认启用（除非显式禁用）。
    - 如果省略 `tools.web.fetch.provider`，OpenClaw 会从可用凭证中自动检测第一个已就绪的 fetch 回退 provider。当前内置 provider 是 Firecrawl。
    - 守护进程会从 `~/.openclaw/.env`（或服务环境）读取环境变量。

    文档：[Web tools](/zh-CN/tools/web)。

  </Accordion>

  <Accordion title="config.apply 把我的配置清空了。如何恢复并避免再次发生？">
    `config.apply` 会替换**整个配置**。如果你发送的是部分对象，其他所有内容
    都会被移除。

    恢复方式：

    - 从备份恢复（git 或复制的 `~/.openclaw/openclaw.json`）。
    - 如果没有备份，请重新运行 `openclaw doctor` 并重新配置 channels/models。
    - 如果这不是你预期的行为，请提交 bug，并附上你最后已知的配置或任何备份。
    - 本地编码智能体通常可以根据日志或历史重建出一份可用配置。

    避免方式：

    - 小改动请使用 `openclaw config set`。
    - 交互式编辑请使用 `openclaw configure`。
    - 当你不确定确切路径或字段结构时，先使用 `config.schema.lookup`；它会返回一个浅层 schema 节点以及直接子项摘要，方便逐层深入。
    - 对于部分 RPC 编辑，使用 `config.patch`；仅在需要完整替换配置时才使用 `config.apply`。
    - 如果你是从智能体运行中使用仅限 owner 的 `gateway` 工具，它仍会拒绝写入 `tools.exec.ask` / `tools.exec.security`（包括会规范化到同一受保护 exec 路径的旧版 `tools.bash.*` 别名）。

    文档：[Config](/cli/config)、[Configure](/cli/configure)、[Doctor](/zh-CN/gateway/doctor)。

  </Accordion>

  <Accordion title="如何运行一个中心化的 Gateway 网关，并在多台设备上配专门的工作节点？">
    常见模式是**一个 Gateway 网关**（例如 Raspberry Pi）+ **nodes** + **agents**：

    - **Gateway 网关（中心）：**持有 channels（Signal/WhatsApp）、路由和会话。
    - **Nodes（设备）：**Mac/iOS/Android 作为外设连接，并暴露本地工具（`system.run`、`canvas`、`camera`）。
    - **Agents（工作节点）：**为专门角色准备的独立大脑/工作区（例如“Hetzner ops”“Personal data”）。
    - **Sub-agents：**当你想并行处理时，从主智能体派生后台工作。
    - **TUI：**连接到 Gateway 网关并切换 agents/sessions。

    文档：[Nodes](/zh-CN/nodes)、[Remote access](/zh-CN/gateway/remote)、[Multi-Agent Routing](/zh-CN/concepts/multi-agent)、[Sub-agents](/zh-CN/tools/subagents)、[TUI](/web/tui)。

  </Accordion>

  <Accordion title="OpenClaw 浏览器可以无头运行吗？">
    可以。这是一个配置选项：

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    默认值是 `false`（有头模式）。无头模式在某些网站上更容易触发反机器人检测。参见 [Browser](/zh-CN/tools/browser)。

    无头模式使用**同一个 Chromium 引擎**，适用于大多数自动化场景（表单、点击、抓取、登录）。主要区别是：

    - 没有可见的浏览器窗口（如果你需要视觉信息，请使用截图）。
    - 某些网站在无头模式下对自动化更严格（CAPTCHA、反机器人）。
      例如，X/Twitter 通常会阻止无头会话。

  </Accordion>

  <Accordion title="如何使用 Brave 进行浏览器控制？">
    将 `browser.executablePath` 设置为你的 Brave 二进制文件（或任意基于 Chromium 的浏览器），然后重启 Gateway 网关。
    完整配置示例请参见 [Browser](/zh-CN/tools/browser#use-brave-or-another-chromium-based-browser)。
  </Accordion>
</AccordionGroup>

## 远程 Gateway 网关和 nodes

<AccordionGroup>
  <Accordion title="命令如何在 Telegram、gateway 和 nodes 之间传播？">
    Telegram 消息由 **gateway** 处理。Gateway 网关运行智能体，
    然后只在需要 node 工具时，才会通过 **Gateway WebSocket** 调用 nodes：

    Telegram → Gateway 网关 → 智能体 → `node.*` → Node → Gateway 网关 → Telegram

    Nodes 不会看到入站 provider 流量；它们只接收 node RPC 调用。

  </Accordion>

  <Accordion title="如果 Gateway 网关托管在远程，我的智能体如何访问我的电脑？">
    简短回答：**把你的电脑配对成一个 node**。Gateway 网关运行在别处，但它可以
    通过 Gateway WebSocket 在你的本地机器上调用 `node.*` 工具（screen、camera、system）。

    典型设置：

    1. 在常开主机（VPS/家庭服务器）上运行 Gateway 网关。
    2. 让 gateway host 和你的电脑加入同一个 tailnet。
    3. 确保 Gateway WS 可达（tailnet bind 或 SSH tunnel）。
    4. 在本地打开 macOS 应用，并以 **Remote over SSH** 模式（或直接 tailnet）
       连接，以便它注册为一个 node。
    5. 在 Gateway 网关上批准该 node：

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    不需要单独的 TCP bridge；nodes 会通过 Gateway WebSocket 连接。

    安全提醒：配对 macOS node 允许在该机器上执行 `system.run`。只
    配对你信任的设备，并阅读 [Security](/zh-CN/gateway/security)。

    文档：[Nodes](/zh-CN/nodes)、[Gateway protocol](/zh-CN/gateway/protocol)、[macOS remote mode](/zh-CN/platforms/mac/remote)、[Security](/zh-CN/gateway/security)。

  </Accordion>

  <Accordion title="Tailscale 已连接，但我收不到回复。现在怎么办？">
    先检查基础项：

    - Gateway 网关正在运行：`openclaw gateway status`
    - Gateway 网关健康状态：`openclaw status`
    - 渠道健康状态：`openclaw channels status`

    然后验证认证和路由：

    - 如果你使用 Tailscale Serve，请确保 `gateway.auth.allowTailscale` 设置正确。
    - 如果你通过 SSH 隧道连接，请确认本地隧道已建立并且指向正确端口。
    - 确认你的 allowlists（私信或群组）包含你的账户。

    文档：[Tailscale](/zh-CN/gateway/tailscale)、[Remote access](/zh-CN/gateway/remote)、[Channels](/zh-CN/channels)。

  </Accordion>

  <Accordion title="两个 OpenClaw 实例之间可以互相对话吗（本地 + VPS）？">
    可以。虽然没有内置的“bot-to-bot”桥接，但你可以通过几种
    可靠方式实现：

    **最简单：**使用一个两个 bot 都能访问的普通聊天渠道（Telegram/Slack/WhatsApp）。
    让 Bot A 给 Bot B 发送消息，然后让 Bot B 像平常一样回复。

    **CLI bridge（通用）：**运行一个脚本，通过
    `openclaw agent --message ... --deliver` 调用另一个 Gateway 网关，并将目标设为另一个 bot
    正在监听的聊天。如果其中一个 bot 在远程 VPS 上，请让你的 CLI 通过 SSH/Tailscale
    指向那个远程 Gateway 网关（参见 [Remote access](/zh-CN/gateway/remote)）。

    示例模式（从能访问目标 Gateway 网关的机器上运行）：

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    提示：添加一层防护，避免两个 bot 无休止循环（仅回应 mention、使用渠道
    allowlists，或设置“不要回复 bot 消息”规则）。

    文档：[Remote access](/zh-CN/gateway/remote)、[Agent CLI](/cli/agent)、[Agent send](/zh-CN/tools/agent-send)。

  </Accordion>

  <Accordion title="多个智能体需要分别使用不同的 VPS 吗？">
    不需要。一个 Gateway 网关可以托管多个智能体，每个智能体都有自己的工作区、模型默认值
    和路由。这是最常见的设置，比分别为每个智能体运行
    一个 VPS 更便宜也更简单。

    只有在你需要硬隔离（安全边界）或完全不同、且不想共享的
    配置时，才使用多个 VPS。否则，请保留一个 Gateway 网关，
    使用多个智能体或 sub-agents 即可。

  </Accordion>

  <Accordion title="相比从 VPS 用 SSH 访问，在我的个人笔记本上使用 node 有什么好处吗？">
    有——nodes 是从远程 Gateway 网关访问你的笔记本的第一等方式，而且
    它们提供的不仅仅是 shell 访问。Gateway 网关运行于 macOS/Linux（Windows 通过 WSL2），并且
    很轻量（小型 VPS 或 Raspberry Pi 级别设备就够用；4 GB RAM 足够），所以常见
    设置是一个常开主机，加上你的笔记本作为 node。

    - **无需入站 SSH。** Nodes 会主动连接到 Gateway WebSocket，并使用设备配对。
    - **更安全的执行控制。** 该笔记本上的 `system.run` 受 node allowlists/approvals 保护。
    - **更多设备工具。** Nodes 除了 `system.run`，还暴露 `canvas`、`camera` 和 `screen`。
    - **本地浏览器自动化。** 让 Gateway 网关运行在 VPS 上，但通过笔记本上的 node host 在本地运行 Chrome，或者通过 Chrome MCP 附着到主机上的本地 Chrome。

    SSH 适合临时 shell 访问，但对于持续性的智能体工作流和
    设备自动化，nodes 更简单。

    文档：[Nodes](/zh-CN/nodes)、[Nodes CLI](/cli/nodes)、[Browser](/zh-CN/tools/browser)。

  </Accordion>

  <Accordion title="nodes 会运行 gateway 服务吗？">
    不会。每台主机上通常只应运行**一个 gateway**，除非你有意运行隔离的 profiles（参见 [Multiple gateways](/zh-CN/gateway/multiple-gateways)）。Nodes 是连接到
    gateway 的外设（iOS/Android nodes，或菜单栏应用中的 macOS “node mode”）。关于无头 node
    host 和 CLI 控制，参见 [Node host CLI](/cli/node)。

    对于 `gateway`、`discovery` 和 `canvasHost` 的变更，需要执行完整重启。

  </Accordion>

  <Accordion title="有没有通过 API / RPC 应用配置的方式？">
    有。

    - `config.schema.lookup`：在写入前检查某个配置子树及其浅层 schema 节点、匹配的 UI hint 和直接子项摘要
    - `config.get`：获取当前快照 + hash
    - `config.patch`：安全的部分更新（大多数 RPC 编辑的首选）
    - `config.apply`：校验 + 替换完整配置，然后重启
    - 仅限 owner 的 `gateway` 运行时工具仍会拒绝重写 `tools.exec.ask` / `tools.exec.security`；旧版 `tools.bash.*` 别名会规范化到同一受保护 exec 路径

  </Accordion>

  <Accordion title="首次安装的最小合理配置">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    这会设置你的工作区，并限制谁可以触发机器人。

  </Accordion>

  <Accordion title="如何在 VPS 上设置 Tailscale 并从 Mac 连接？">
    最简步骤：

    1. **在 VPS 上安装并登录**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **在你的 Mac 上安装并登录**
       - 使用 Tailscale 应用并登录到同一个 tailnet。
    3. **启用 MagicDNS（推荐）**
       - 在 Tailscale 管理控制台中启用 MagicDNS，使 VPS 拥有稳定名称。
    4. **使用 tailnet 主机名**
       - SSH：`ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS：`ws://your-vps.tailnet-xxxx.ts.net:18789`

    如果你想在不使用 SSH 的情况下访问 Control UI，请在 VPS 上使用 Tailscale Serve：

    ```bash
    openclaw gateway --tailscale serve
    ```

    这会让 gateway 保持绑定在 loopback 上，并通过 Tailscale 暴露 HTTPS。参见 [Tailscale](/zh-CN/gateway/tailscale)。

  </Accordion>

  <Accordion title="如何将 Mac node 连接到远程 Gateway 网关（Tailscale Serve）？">
    Serve 会暴露 **Gateway Control UI + WS**。Nodes 通过同一个 Gateway WS endpoint 连接。

    推荐设置：

    1. **确保 VPS 和 Mac 在同一个 tailnet 中**。
    2. **在远程模式中使用 macOS 应用**（SSH 目标可以是 tailnet 主机名）。
       应用会隧道化 Gateway 端口，并以 node 身份连接。
    3. **在 gateway 上批准该 node：**

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    文档：[Gateway protocol](/zh-CN/gateway/protocol)、[设备发现](/zh-CN/gateway/discovery)、[macOS remote mode](/zh-CN/platforms/mac/remote)。

  </Accordion>

  <Accordion title="我应该在第二台笔记本上安装，还是只添加一个 node？">
    如果你只需要第二台笔记本上的**本地工具**（screen/camera/exec），请将其添加为
    **node**。这样可以保留单一 Gateway 网关，避免重复配置。目前本地 node 工具
    仅支持 macOS，但我们计划扩展到其他操作系统。

    只有在你需要**硬隔离**或两个完全独立的 bot 时，才安装第二个 Gateway 网关。

    文档：[Nodes](/zh-CN/nodes)、[Nodes CLI](/cli/nodes)、[Multiple gateways](/zh-CN/gateway/multiple-gateways)。

  </Accordion>
</AccordionGroup>

## 环境变量和 .env 加载

<AccordionGroup>
  <Accordion title="OpenClaw 如何加载环境变量？">
    OpenClaw 会从父进程（shell、launchd/systemd、CI 等）读取环境变量，并额外加载：

    - 当前工作目录中的 `.env`
    - 来自 `~/.openclaw/.env`（也就是 `$OPENCLAW_STATE_DIR/.env`）的全局回退 `.env`

    这两个 `.env` 文件都不会覆盖已有环境变量。

    你也可以在配置中定义内联环境变量（仅在进程环境中缺失时才应用）：

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    完整优先级和来源请参见 [/environment](/zh-CN/help/environment)。

  </Accordion>

  <Accordion title="我通过服务启动了 Gateway 网关，结果环境变量丢失了。怎么办？">
    两个常见修复方法：

    1. 将缺失的 key 放入 `~/.openclaw/.env`，这样即使服务没有继承你的 shell 环境，也能被读取。
    2. 启用 shell 导入（可选的便捷功能）：

    ```json5
    {
      env: {
        shellEnv: {
          enabled: true,
          timeoutMs: 15000,
        },
      },
    }
    ```

    这会运行你的登录 shell，并只导入缺失的预期键名（永不覆盖）。对应的环境变量：
    `OPENCLAW_LOAD_SHELL_ENV=1`、`OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`。

  </Accordion>

  <Accordion title='我设置了 COPILOT_GITHUB_TOKEN，但 models status 显示 “Shell env: off.”。为什么？'>
    `openclaw models status` 报告的是**shell 环境导入**是否启用。“Shell env: off”
    **并不**表示你的环境变量缺失——它只是表示 OpenClaw 不会自动加载
    你的登录 shell。

    如果 Gateway 网关作为服务运行（launchd/systemd），它不会继承你的 shell
    环境。请用以下任一方式修复：

    1. 将 token 放入 `~/.openclaw/.env`：

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. 或启用 shell 导入（`env.shellEnv.enabled: true`）。
    3. 或把它加到配置中的 `env` 块里（仅在缺失时应用）。

    然后重启 gateway 并重新检查：

    ```bash
    openclaw models status
    ```

    Copilot token 会从 `COPILOT_GITHUB_TOKEN` 读取（也支持 `GH_TOKEN` / `GITHUB_TOKEN`）。
    参见 [/concepts/model-providers](/zh-CN/concepts/model-providers) 和 [/environment](/zh-CN/help/environment)。

  </Accordion>
</AccordionGroup>

## 会话和多个聊天

<AccordionGroup>
  <Accordion title="如何开始一段全新的对话？">
    发送 `/new` 或 `/reset` 作为单独消息。参见 [Session management](/zh-CN/concepts/session)。
  </Accordion>

  <Accordion title="如果我从不发送 /new，会话会自动重置吗？">
    会话可以在 `session.idleMinutes` 之后过期，但这**默认是关闭的**（默认 **0**）。
    将其设为正值即可启用空闲过期。启用后，空闲期之后的**下一条**
    消息会为该聊天键启动一个新的会话 id。
    这不会删除转录内容——只是开启一个新会话。

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="有没有办法做一个 OpenClaw 实例团队（一个 CEO 和很多智能体）？">
    有，可以通过**多智能体路由**和**sub-agents**。你可以创建一个协调者
    智能体和多个拥有各自工作区与模型的工作智能体。

    不过，这更适合作为一种**有趣的实验**。它会消耗大量 token，而且
    通常不如使用一个 bot 配合多个会话来得高效。我们典型设想的模型是：
    你与一个 bot 对话，同时用不同会话来并行工作。这个
    bot 在需要时也可以生成 sub-agents。

    文档：[Multi-agent routing](/zh-CN/concepts/multi-agent)、[Sub-agents](/zh-CN/tools/subagents)、[Agents CLI](/cli/agents)。

  </Accordion>

  <Accordion title="为什么上下文会在任务中途被截断？如何防止？">
    会话上下文受模型窗口限制。长聊天、大量工具输出或过多
    文件都可能触发压缩或截断。

    有帮助的做法：

    - 让机器人总结当前状态并写入文件。
    - 在长任务前使用 `/compact`，切换话题时使用 `/new`。
    - 将重要上下文保存在工作区中，并让机器人重新读取。
    - 对长任务或并行工作使用 sub-agents，以保持主聊天更小。
    - 如果这种情况经常发生，请选择上下文窗口更大的模型。

  </Accordion>

  <Accordion title="如何彻底重置 OpenClaw 但保留安装？">
    使用 reset 命令：

    ```bash
    openclaw reset
    ```

    非交互式完整重置：

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    然后重新运行设置：

    ```bash
    openclaw onboard --install-daemon
    ```

    说明：

    - 如果新手引导检测到已有配置，也会提供 **Reset**。参见 [新手引导（CLI）](/zh-CN/start/wizard)。
    - 如果你使用了 profiles（`--profile` / `OPENCLAW_PROFILE`），请重置每个状态目录（默认是 `~/.openclaw-<profile>`）。
    - 开发重置：`openclaw gateway --dev --reset`（仅开发模式；会清除开发配置 + credentials + sessions + workspace）。

  </Accordion>

  <Accordion title='我收到 “context too large” 错误——如何重置或压缩？'>
    使用以下任一方式：

    - **压缩**（保留对话，但总结较早的轮次）：

      ```
      /compact
      ```

      或使用 `/compact <instructions>` 指导摘要内容。

    - **重置**（为同一聊天键创建新的会话 ID）：

      ```
      /new
      /reset
      ```

    如果这种情况总是发生：

    - 启用或调整**会话修剪**（`agents.defaults.contextPruning`）来裁剪旧的工具输出。
    - 使用上下文窗口更大的模型。

    文档：[Compaction](/zh-CN/concepts/compaction)、[Session pruning](/zh-CN/concepts/session-pruning)、[Session management](/zh-CN/concepts/session)。

  </Accordion>

  <Accordion title='为什么我会看到 “LLM request rejected: messages.content.tool_use.input field required”？'>
    这是一个 provider 校验错误：模型输出了一个缺少必需
    `input` 的 `tool_use` 块。这通常意味着会话历史陈旧或已损坏（经常发生在长线程
    或工具/schema 变更之后）。

    修复方法：使用 `/new`（单独消息）开启一个全新会话。

  </Accordion>

  <Accordion title="为什么我每 30 分钟就会收到一次 heartbeat 消息？">
    Heartbeat 默认每 **30m** 运行一次（使用 OAuth auth 时为 **1h**）。可以调整或禁用它们：

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // 或 "0m" 以禁用
          },
        },
      },
    }
    ```

    如果 `HEARTBEAT.md` 存在但实际为空（只有空行和 markdown
    标题，如 `# Heading`），OpenClaw 会跳过 heartbeat 运行，以节省 API 调用。
    如果该文件不存在，heartbeat 仍然会运行，由模型决定如何处理。

    每智能体覆盖使用 `agents.list[].heartbeat`。文档：[Heartbeat](/zh-CN/gateway/heartbeat)。

  </Accordion>

  <Accordion title='我需要把“bot 账号”加进 WhatsApp 群组吗？'>
    不需要。OpenClaw 运行在**你自己的账号**上，所以只要你在群里，OpenClaw 就能看到它。
    默认情况下，在你允许发送者之前，群组回复会被阻止（`groupPolicy: "allowlist"`）。

    如果你希望只有**你自己**能触发群组回复：

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="如何获取 WhatsApp 群组的 JID？">
    选项 1（最快）：跟随日志，然后在群里发送一条测试消息：

    ```bash
    openclaw logs --follow --json
    ```

    查找以 `@g.us` 结尾的 `chatId`（或 `from`），例如：
    `1234567890-1234567890@g.us`。

    选项 2（如果已配置/加入 allowlist）：从配置列出群组：

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    文档：[WhatsApp](/zh-CN/channels/whatsapp)、[Directory](/cli/directory)、[Logs](/cli/logs)。

  </Accordion>

  <Accordion title="为什么 OpenClaw 在群组中不回复？">
    两个常见原因：

    - mention gating 已启用（默认）。你必须 @ 提及 bot（或匹配 `mentionPatterns`）。
    - 你配置了 `channels.whatsapp.groups` 但没有包含 `"*"`，而该群组不在 allowlist 中。

    参见 [Groups](/zh-CN/channels/groups) 和 [Group messages](/zh-CN/channels/group-messages)。

  </Accordion>

  <Accordion title="群组/线程会与私信共享上下文吗？">
    直接聊天默认会折叠到主会话。群组/渠道有自己的会话键，而 Telegram 话题 / Discord 线程是独立会话。参见 [Groups](/zh-CN/channels/groups) 和 [Group messages](/zh-CN/channels/group-messages)。
  </Accordion>

  <Accordion title="我能创建多少个工作区和智能体？">
    没有硬性限制。几十个（甚至上百个）都没问题，但请注意：

    - **磁盘增长：**sessions + transcripts 保存在 `~/.openclaw/agents/<agentId>/sessions/` 下。
    - **Token 成本：**更多智能体意味着更多并发模型调用。
    - **运维开销：**每智能体 auth profiles、工作区和渠道路由。

    提示：

    - 为每个智能体保留一个**活跃**工作区（`agents.defaults.workspace`）。
    - 如果磁盘增长明显，请修剪旧会话（删除 JSONL 或 store 条目）。
    - 使用 `openclaw doctor` 查找游离工作区和 profile 不匹配。

  </Accordion>

  <Accordion title="我可以同时运行多个 bot 或多个聊天（Slack）吗？应该如何设置？">
    可以。使用 **Multi-Agent Routing** 运行多个隔离的智能体，并按
    渠道/账户/peer 路由入站消息。Slack 支持作为渠道，并可绑定到特定智能体。

    浏览器访问能力很强，但并不意味着“人能做的它都能做”——反机器人机制、CAPTCHA 和 MFA
    仍然可能阻止自动化。若要获得最可靠的浏览器控制，请在主机上使用本地 Chrome MCP，
    或在实际运行浏览器的机器上使用 CDP。

    最佳实践设置：

    - 始终在线的 Gateway 网关主机（VPS/Mac mini）。
    - 每个角色一个智能体（bindings）。
    - 将 Slack 渠道绑定到这些智能体。
    - 在需要时通过 Chrome MCP 或 node 使用本地浏览器。

    文档：[Multi-Agent Routing](/zh-CN/concepts/multi-agent)、[Slack](/zh-CN/channels/slack)、
    [Browser](/zh-CN/tools/browser)、[Nodes](/zh-CN/nodes)。

  </Accordion>
</AccordionGroup>

## 模型：默认值、选择、别名、切换

<AccordionGroup>
  <Accordion title='什么是“默认模型”？'>
    OpenClaw 的默认模型就是你设置在：

    ```
    agents.defaults.model.primary
    ```

    模型以 `provider/model` 的形式引用（例如：`openai/gpt-5.4`）。如果你省略 provider，OpenClaw 会先尝试别名，然后尝试对该精确模型 id 进行唯一的已配置 provider 匹配，只有最后才会回退到已配置的默认 provider，这是一条已弃用的兼容路径。如果该 provider 已不再暴露配置中的默认模型，OpenClaw 会回退到第一个已配置的 provider/model，而不是继续保留一个陈旧的、已移除 provider 的默认值。你仍然应该**显式**设置 `provider/model`。

  </Accordion>

  <Accordion title="你推荐什么模型？">
    **推荐默认值：**使用你提供商栈中可用的最强最新一代模型。
    **对于启用工具或会处理不可信输入的智能体：**优先考虑模型能力，而不是成本。
    **对于日常/低风险聊天：**使用更便宜的后备模型，并按智能体角色进行路由。

    MiniMax 有单独文档：[MiniMax](/zh-CN/providers/minimax) 和
    [Local models](/zh-CN/gateway/local-models)。

    经验法则：对于高风险工作，使用**你负担得起的最佳模型**；对于日常
    聊天或摘要，使用更便宜的模型。你可以按智能体路由模型，并使用 sub-agents
    并行长任务（每个 sub-agent 都会消耗 token）。参见 [Models](/zh-CN/concepts/models) 和
    [Sub-agents](/zh-CN/tools/subagents)。

    强烈警告：能力较弱/过度量化的模型更容易受到 prompt
    injection 和不安全行为影响。参见 [Security](/zh-CN/gateway/security)。

    更多背景：[Models](/zh-CN/concepts/models)。

  </Accordion>

  <Accordion title="如何在不清空配置的情况下切换模型？">
    使用**模型命令**，或只编辑**模型**相关字段。避免完整替换配置。

    安全方式：

    - 在聊天中使用 `/model`（快捷、按会话生效）
    - 使用 `openclaw models set ...`（只更新模型配置）
    - 使用 `openclaw configure --section model`（交互式）
    - 编辑 `~/.openclaw/openclaw.json` 中的 `agents.defaults.model`

    除非你确实想替换整个配置，否则避免用部分对象调用 `config.apply`。
    对于 RPC 编辑，先用 `config.schema.lookup` 检查，并优先使用 `config.patch`。lookup 负载会给你规范化路径、浅层 schema 文档/约束以及直接子项摘要。
    用于部分更新。
    如果你已经覆盖了配置，请从备份恢复，或重新运行 `openclaw doctor` 进行修复。

    文档：[Models](/zh-CN/concepts/models)、[Configure](/cli/configure)、[Config](/cli/config)、[Doctor](/zh-CN/gateway/doctor)。

  </Accordion>

  <Accordion title="我可以使用自托管模型（llama.cpp、vLLM、Ollama）吗？">
    可以。对于本地模型来说，Ollama 是最容易上手的路径。

    最快设置方式：

    1. 从 `https://ollama.com/download` 安装 Ollama
    2. 拉取一个本地模型，例如 `ollama pull glm-4.7-flash`
    3. 如果你也想使用云模型，请运行 `ollama signin`
    4. 运行 `openclaw onboard` 并选择 `Ollama`
    5. 选择 `Local` 或 `Cloud + Local`

    说明：

    - `Cloud + Local` 会提供云模型以及你的本地 Ollama 模型
    - 像 `kimi-k2.5:cloud` 这样的云模型不需要本地 pull
    - 若要手动切换，请使用 `openclaw models list` 和 `openclaw models set ollama/<model>`

    安全说明：更小或高度量化的模型更容易受到 prompt
    injection 影响。对于任何可以使用工具的 bot，我们强烈建议使用**大模型**。
    如果你仍想使用小模型，请启用沙箱隔离并使用严格的工具 allowlists。

    文档：[Ollama](/zh-CN/providers/ollama)、[Local models](/zh-CN/gateway/local-models)、
    [Model providers](/zh-CN/concepts/model-providers)、[Security](/zh-CN/gateway/security)、
    [沙箱隔离](/zh-CN/gateway/sandboxing)。

  </Accordion>

  <Accordion title="OpenClaw、Flawd 和 Krill 使用什么模型？">
    - 这些部署可能彼此不同，而且会随时间变化；没有固定的提供商推荐。
    - 请在各自的 gateway 上通过 `openclaw models status` 检查当前运行时设置。
    - 对于安全敏感/启用工具的智能体，请使用可用的最强最新一代模型。
  </Accordion>

  <Accordion title="如何动态切换模型（无需重启）？">
    发送 `/model` 命令作为单独消息：

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    这些是内置别名。你也可以通过 `agents.defaults.models` 添加自定义别名。

    可用模型可以通过 `/model`、`/model list` 或 `/model status` 列出。

    `/model`（以及 `/model list`）会显示一个紧凑的编号选择器。按编号选择：

    ```
    /model 3
    ```

    你也可以为该 provider 强制指定一个特定 auth profile（按会话生效）：

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    提示：`/model status` 会显示当前活跃的是哪个智能体、使用的是哪个 `auth-profiles.json` 文件，以及接下来会尝试哪个 auth profile。
    它还会在可用时显示已配置的 provider endpoint（`baseUrl`）和 API 模式（`api`）。

    **如何取消用 @profile 设定的固定 profile？**

    重新执行 `/model`，但**不要**带 `@profile` 后缀：

    ```
    /model anthropic/claude-opus-4-6
    ```

    如果你想回到默认值，请在 `/model` 中选择默认项（或发送 `/model <default provider/model>`）。
    使用 `/model status` 确认当前活跃的 auth profile。

  </Accordion>

  <Accordion title="我可以用 GPT 5.2 处理日常任务、用 Codex 5.3 编程吗？">
    可以。设置其中一个为默认值，并在需要时切换：

    - **快速切换（按会话）：**日常任务用 `/model gpt-5.4`，编程时用 `/model openai-codex/gpt-5.4` 走 Codex OAuth。
    - **默认值 + 切换：**将 `agents.defaults.model.primary` 设为 `openai/gpt-5.4`，编程时切换到 `openai-codex/gpt-5.4`（或者反过来）。
    - **Sub-agents：**将编码任务路由到默认模型不同的 sub-agents。

    参见 [Models](/zh-CN/concepts/models) 和 [Slash commands](/zh-CN/tools/slash-commands)。

  </Accordion>

  <Accordion title="如何为 GPT 5.4 配置 fast mode？">
    你可以使用会话级开关，也可以设置配置默认值：

    - **按会话：**当会话使用 `openai/gpt-5.4` 或 `openai-codex/gpt-5.4` 时，发送 `/fast on`。
    - **按模型默认值：**将 `agents.defaults.models["openai/gpt-5.4"].params.fastMode` 设为 `true`。
    - **Codex OAuth 同样适用：**如果你也使用 `openai-codex/gpt-5.4`，请对它设置同样的标志。

    示例：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
            "openai-codex/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    对于 OpenAI，fast mode 在支持的原生 Responses 请求中会映射为 `service_tier = "priority"`。会话级 `/fast` 覆盖优先于配置默认值。

    参见 [Thinking and fast mode](/zh-CN/tools/thinking) 和 [OpenAI fast mode](/zh-CN/providers/openai#openai-fast-mode)。

  </Accordion>

  <Accordion title='为什么我会看到 “Model ... is not allowed”，然后没有回复？'>
    如果设置了 `agents.defaults.models`，它就会成为 `/model` 和任何
    会话覆盖的**allowlist**。选择不在该列表中的模型会返回：

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    该错误会**替代**正常回复返回。修复方法：将该模型加入
    `agents.defaults.models`，移除 allowlist，或从 `/model list` 中选择一个模型。

  </Accordion>

  <Accordion title='为什么我会看到 “Unknown model: minimax/MiniMax-M2.7”？'>
    这表示**provider 未配置**（未找到 MiniMax provider 配置或 auth
    profile），因此无法解析该模型。

    修复检查清单：

    1. 升级到当前 OpenClaw 版本（或直接从源码 `main` 运行），然后重启 gateway。
    2. 确保 MiniMax 已配置（向导或 JSON），或者环境变量/auth profiles 中存在 MiniMax 认证，
       这样对应 provider 才能被注入
       （`minimax` 使用 `MINIMAX_API_KEY`，`minimax-portal` 使用 `MINIMAX_OAUTH_TOKEN` 或已保存的 MiniMax
       OAuth）。
    3. 使用与你的 auth 路径精确匹配的模型 id（区分大小写）：
       API key 设置使用 `minimax/MiniMax-M2.7` 或 `minimax/MiniMax-M2.7-highspeed`，
       OAuth 设置使用 `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed`。
    4. 运行：

       ```bash
       openclaw models list
       ```

       然后从列表中选择（或在聊天中使用 `/model list`）。

    参见 [MiniMax](/zh-CN/providers/minimax) 和 [Models](/zh-CN/concepts/models)。

  </Accordion>

  <Accordion title="我可以把 MiniMax 作为默认模型，把 OpenAI 用于复杂任务吗？">
    可以。将 **MiniMax 设为默认模型**，并在需要时**按会话**切换模型。
    Fallback 是为**错误**准备的，而不是为“难任务”准备的，所以请使用 `/model` 或单独的智能体。

    **选项 A：按会话切换**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M2.7" },
          models: {
            "minimax/MiniMax-M2.7": { alias: "minimax" },
            "openai/gpt-5.4": { alias: "gpt" },
          },
        },
      },
    }
    ```

    然后：

    ```
    /model gpt
    ```

    **选项 B：独立智能体**

    - 智能体 A 默认：MiniMax
    - 智能体 B 默认：OpenAI
    - 按智能体路由，或使用 `/agent` 切换

    文档：[Models](/zh-CN/concepts/models)、[Multi-Agent Routing](/zh-CN/concepts/multi-agent)、[MiniMax](/zh-CN/providers/minimax)、[OpenAI](/zh-CN/providers/openai)。

  </Accordion>

  <Accordion title="opus / sonnet / gpt 是内置快捷方式吗？">
    是的。OpenClaw 提供了一些默认简写（仅当该模型存在于 `agents.defaults.models` 中时才生效）：

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3-flash-preview`
    - `gemini-flash-lite` → `google/gemini-3.1-flash-lite-preview`

    如果你设置了同名的自定义别名，以你的值为准。

  </Accordion>

  <Accordion title="如何定义/覆盖模型快捷方式（别名）？">
    别名来自 `agents.defaults.models.<modelId>.alias`。示例：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
            "anthropic/claude-haiku-4-5": { alias: "haiku" },
          },
        },
      },
    }
    ```

    然后 `/model sonnet`（或在支持时使用 `/<alias>`）就会解析到对应模型 ID。

  </Accordion>

  <Accordion title="如何添加 OpenRouter 或 Z.AI 等其他提供商的模型？">
    OpenRouter（按 token 付费；模型很多）：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI（GLM 模型）：

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5" },
          models: { "zai/glm-5": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    如果你引用了某个 provider/model，但缺少所需 provider key，就会收到运行时 auth 错误（例如 `No API key found for provider "zai"`）。

    **添加新智能体后提示 No API key found for provider**

    这通常表示**新智能体**的 auth store 是空的。认证是按智能体隔离的，保存在：

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    修复选项：

    - 运行 `openclaw agents add <id>` 并在向导中配置认证。
    - 或将主智能体 `agentDir` 中的 `auth-profiles.json` 复制到新智能体的 `agentDir` 中。

    **不要**在多个智能体之间复用 `agentDir`；这会导致 auth/session 冲突。

  </Accordion>
</AccordionGroup>

## 模型故障切换和 “All models failed”

<AccordionGroup>
  <Accordion title="故障切换如何工作？">
    故障切换分两个阶段：

    1. 同一 provider 内的 **Auth profile rotation**。
    2. 切换到 `agents.defaults.model.fallbacks` 中的下一个模型，即 **Model fallback**。

    对失败的 profiles 会应用冷却时间（指数退避），因此即使 provider 被限流或暂时失败，OpenClaw 仍能继续响应。

    速率限制桶不只包含普通的 `429` 响应。OpenClaw
    还会将诸如 `Too many concurrent requests`、
    `ThrottlingException`、`concurrency limit reached`、
    `workers_ai ... quota limit exceeded`、`resource exhausted` 以及周期性的
    使用窗口限制（`weekly/monthly limit reached`）视为值得进行故障切换的
    rate limits。

    某些看起来像计费问题的响应并不是 `402`，而某些 HTTP `402`
    响应也会保留在那个瞬时故障桶中。如果某个 provider 在
    `401` 或 `403` 上返回了明确的计费文本，OpenClaw 仍会把它
    归入 billing 通道，但 provider 专属的文本匹配器仍只作用于拥有它们的
    provider（例如 OpenRouter 的 `Key limit exceeded`）。如果某个 `402`
    消息看起来像可重试的使用窗口限制或
    organization/workspace 支出限制（`daily limit reached, resets tomorrow`、
    `organization spending limit exceeded`），OpenClaw 会将其视为
    `rate_limit`，而不是长期计费停用。

    上下文溢出错误不同：像
    `request_too_large`、`input exceeds the maximum number of tokens`、
    `input token count exceeds the maximum number of input tokens`、
    `input is too long for the model` 或 `ollama error: context length
    exceeded` 这样的特征会保留在压缩/重试路径上，而不会推进模型
    fallback。

    通用服务端错误文本的判断标准，刻意比“任何带 unknown/error 的内容”
    更窄。OpenClaw 确实会把 provider 范围内的瞬时故障形态视为
    值得故障切换的 timeout/overloaded 信号，例如 Anthropic 的裸
    `An unknown error occurred`、OpenRouter 的裸
    `Provider returned error`、停止原因错误如 `Unhandled stop reason:
    error`、带瞬时服务端文本的 JSON `api_error` 负载
    （`internal server error`、`unknown error, 520`、`upstream error`、`backend
    error`），以及 provider 繁忙错误如 `ModelNotReadyException`，前提是 provider 上下文
    匹配。
    像 `LLM request failed with an unknown
    error.` 这样的通用内部回退文本则保持保守，仅靠它本身不会触发模型 fallback。

  </Accordion>

  <Accordion title='“No credentials found for profile anthropic:default” 是什么意思？'>
    这表示系统尝试使用 auth profile ID `anthropic:default`，但在预期的 auth store 中找不到与之对应的 credentials。

    **修复检查清单：**

    - **确认 auth profiles 的存放位置**（新路径与旧路径）
      - 当前路径：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
      - 旧路径：`~/.openclaw/agent/*`（由 `openclaw doctor` 迁移）
    - **确认 Gateway 网关能加载到你的环境变量**
      - 如果你在 shell 中设置了 `ANTHROPIC_API_KEY`，但 Gateway 网关是通过 systemd/launchd 运行的，它可能不会继承该变量。请把它放到 `~/.openclaw/.env`，或启用 `env.shellEnv`。
    - **确保你编辑的是正确的智能体**
      - 多智能体设置意味着可能有多个 `auth-profiles.json` 文件。
    - **做一个 model/auth 状态检查**
      - 使用 `openclaw models status` 查看已配置模型，以及 providers 是否已认证。

    **针对 “No credentials found for profile anthropic” 的修复检查清单**

    这表示当前运行被固定到某个 Anthropic auth profile，但 Gateway 网关
    无法在其 auth store 中找到它。

    - **使用 Claude CLI**
      - 在 gateway host 上运行 `openclaw models auth login --provider anthropic --method cli --set-default`。
    - **如果你想改用 API key**
      - 将 `ANTHROPIC_API_KEY` 放进 **gateway host** 上的 `~/.openclaw/.env`。
      - 清除任何强制使用缺失 profile 的固定顺序：

        ```bash
        openclaw models auth order clear --provider anthropic
        ```

    - **确认你是在 gateway host 上运行命令**
      - 在 remote mode 下，auth profiles 位于 gateway 机器上，而不是你的笔记本上。

  </Accordion>

  <Accordion title="为什么它