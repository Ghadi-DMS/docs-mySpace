---
read_when:
    - 你想要容器化的 Gateway 网关，而不是本地安装
    - 你正在验证 Docker 流程
summary: OpenClaw 的可选基于 Docker 的设置和新手引导
title: Docker
x-i18n:
    generated_at: "2026-04-05T21:33:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: d6aa0453340d7683b4954316274ba6dd1aa7c0ce2483e9bd8ae137ff4efd4c3c
    source_path: install/docker.md
    workflow: 15
---

# Docker（可选）

Docker **可选**。只有当你想要容器化的 Gateway 网关，或想验证 Docker 流程时，才使用它。

## Docker 适合我吗？

- **是**：你想要一个隔离的、可随时丢弃的 Gateway 网关环境，或者想在没有本地安装的主机上运行 OpenClaw。
- **否**：你是在自己的机器上运行，只想要最快的开发循环。请改用常规安装流程。
- **沙箱注意事项**：智能体沙箱隔离也使用 Docker，但它**不**要求整个 Gateway 网关都运行在 Docker 中。参见 [沙箱隔离](/zh-CN/gateway/sandboxing)。

## 前提条件

- Docker Desktop（或 Docker Engine）+ Docker Compose v2
- 镜像构建至少需要 2 GB 内存（在 1 GB 主机上，`pnpm install` 可能因 OOM 被终止并返回退出码 137）
- 足够存放镜像和日志的磁盘空间
- 如果在 VPS/公网主机上运行，请查看
  [网络暴露的安全加固](/zh-CN/gateway/security)，
  尤其是 Docker `DOCKER-USER` 防火墙策略。

## 容器化 Gateway 网关

<Steps>
  <Step title="构建镜像">
    在仓库根目录运行设置脚本：

    ```bash
    ./scripts/docker/setup.sh
    ```

    这会在本地构建 Gateway 网关镜像。若要改用预构建镜像：

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    预构建镜像发布在
    [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw)。
    常见标签：`main`、`latest`、`<version>`（例如 `2026.2.26`）。

  </Step>

  <Step title="完成新手引导">
    设置脚本会自动运行新手引导。它将会：

    - 提示你输入提供商 API 密钥
    - 生成 Gateway 网关令牌并将其写入 `.env`
    - 通过 Docker Compose 启动 Gateway 网关

    在设置过程中，启动前的新手引导和配置写入会直接通过
    `openclaw-gateway` 运行。`openclaw-cli` 用于网关容器已经存在之后你要运行的命令。

  </Step>

  <Step title="打开 Control UI">
    在浏览器中打开 `http://127.0.0.1:18789/`，并将已配置的共享密钥粘贴到 Settings 中。设置脚本默认会将令牌写入 `.env`；如果你将容器配置切换为密码认证，请改用该密码。

    想再次获取 URL？

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="配置渠道（可选）">
    使用 CLI 容器添加消息渠道：

    ```bash
    # WhatsApp（二维码）
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    文档：[WhatsApp](/zh-CN/channels/whatsapp)、[Telegram](/zh-CN/channels/telegram)、[Discord](/zh-CN/channels/discord)

  </Step>
</Steps>

### 手动流程

如果你更希望自己逐步执行，而不是使用设置脚本：

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

<Note>
从仓库根目录运行 `docker compose`。如果你启用了 `OPENCLAW_EXTRA_MOUNTS`
或 `OPENCLAW_HOME_VOLUME`，设置脚本会写入 `docker-compose.extra.yml`；
请使用 `-f docker-compose.yml -f docker-compose.extra.yml` 将其包含进来。
</Note>

<Note>
由于 `openclaw-cli` 与 `openclaw-gateway` 共享网络命名空间，它是一个
启动后的工具。在执行 `docker compose up -d openclaw-gateway` 之前，请通过带有
`--no-deps --entrypoint node` 的 `openclaw-gateway` 运行新手引导和设置阶段的配置写入。
</Note>

### 环境变量

设置脚本接受以下可选环境变量：

| Variable                       | Purpose                                                          |
| ------------------------------ | ---------------------------------------------------------------- |
| `OPENCLAW_IMAGE`               | 使用远程镜像，而不是在本地构建                                   |
| `OPENCLAW_DOCKER_APT_PACKAGES` | 在构建期间安装额外的 apt 软件包（以空格分隔）                    |
| `OPENCLAW_EXTENSIONS`          | 在构建时预安装扩展依赖（名称以空格分隔）                         |
| `OPENCLAW_EXTRA_MOUNTS`        | 额外的主机 bind 挂载（以逗号分隔的 `source:target[:opts]`）      |
| `OPENCLAW_HOME_VOLUME`         | 将 `/home/node` 持久化到具名 Docker volume 中                    |
| `OPENCLAW_SANDBOX`             | 选择启用沙箱引导（`1`、`true`、`yes`、`on`）                     |
| `OPENCLAW_DOCKER_SOCKET`       | 覆盖 Docker socket 路径                                          |

### 健康检查

容器探测端点（无需认证）：

```bash
curl -fsS http://127.0.0.1:18789/healthz   # 存活检查
curl -fsS http://127.0.0.1:18789/readyz     # 就绪检查
```

Docker 镜像内置了一个会 ping `/healthz` 的 `HEALTHCHECK`。
如果检查持续失败，Docker 会将容器标记为 `unhealthy`，
编排系统就可以重启或替换它。

需要认证的深度健康快照：

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN 与 loopback

`scripts/docker/setup.sh` 默认将 `OPENCLAW_GATEWAY_BIND=lan`，这样借助 Docker 端口发布，主机访问
`http://127.0.0.1:18789` 就能正常工作。

- `lan`（默认）：主机浏览器和主机 CLI 都可以访问已发布的 Gateway 网关端口。
- `loopback`：只有容器网络命名空间内的进程可以直接访问
  Gateway 网关。

<Note>
在 `gateway.bind` 中使用 bind 模式值（`lan` / `loopback` / `custom` /
`tailnet` / `auto`），不要使用像 `0.0.0.0` 或 `127.0.0.1` 这样的主机别名。
</Note>

### 存储与持久化

Docker Compose 会将 `OPENCLAW_CONFIG_DIR` bind 挂载到 `/home/node/.openclaw`，并将
`OPENCLAW_WORKSPACE_DIR` 挂载到 `/home/node/.openclaw/workspace`，因此这些路径在
容器被替换后仍会保留。

该挂载的配置目录是 OpenClaw 存放以下内容的位置：

- 用于行为配置的 `openclaw.json`
- 用于存储提供商 OAuth/API 密钥认证的 `agents/<agentId>/agent/auth-profiles.json`
- 用于保存基于环境变量的运行时密钥（例如 `OPENCLAW_GATEWAY_TOKEN`）的 `.env`

有关 VM 部署中持久化的完整细节，请参见
[Docker VM Runtime - 数据持久化位置](/zh-CN/install/docker-vm-runtime#what-persists-where)。

**磁盘增长热点：**请关注 `media/`、会话 JSONL 文件、`cron/runs/*.jsonl`，
以及 `/tmp/openclaw/` 下的滚动文件日志。

### Shell 帮助脚本（可选）

为了更方便地进行日常 Docker 管理，请安装 `ClawDock`：

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

如果你之前是通过旧的原始路径 `scripts/shell-helpers/clawdock-helpers.sh` 安装的 ClawDock，请重新运行上面的安装命令，以便让你的本地帮助脚本文件跟随新路径。

然后使用 `clawdock-start`、`clawdock-stop`、`clawdock-dashboard` 等命令。运行
`clawdock-help` 查看所有命令。
完整帮助指南请参见 [ClawDock](/zh-CN/install/clawdock)。

<AccordionGroup>
  <Accordion title="为 Docker Gateway 网关启用智能体沙箱">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    自定义 socket 路径（例如 rootless Docker）：

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    脚本只有在沙箱前提条件通过后才会挂载 `docker.sock`。如果
    沙箱设置无法完成，脚本会将 `agents.defaults.sandbox.mode`
    重置为 `off`。

  </Accordion>

  <Accordion title="自动化 / CI（非交互式）">
    使用 `-T` 禁用 Compose 伪 TTY 分配：

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="共享网络安全说明">
    `openclaw-cli` 使用 `network_mode: "service:openclaw-gateway"`，因此 CLI
    命令可以通过 `127.0.0.1` 访问 Gateway 网关。请将其视为一个共享的
    信任边界。compose 配置会移除 `NET_RAW`/`NET_ADMIN`，并为
    `openclaw-cli` 启用 `no-new-privileges`。
  </Accordion>

  <Accordion title="权限和 EACCES">
    镜像以 `node`（uid 1000）身份运行。如果你在
    `/home/node/.openclaw` 上看到权限错误，请确保你的主机 bind 挂载归 uid 1000 所有：

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

  </Accordion>

  <Accordion title="更快的重建">
    请按依赖层可缓存的方式组织你的 Dockerfile。这样除非 lockfile 发生变化，
    否则无需重新运行 `pnpm install`：

    ```dockerfile
    FROM node:24-bookworm
    RUN curl -fsSL https://bun.sh/install | bash
    ENV PATH="/root/.bun/bin:${PATH}"
    RUN corepack enable
    WORKDIR /app
    COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
    COPY ui/package.json ./ui/package.json
    COPY scripts ./scripts
    RUN pnpm install --frozen-lockfile
    COPY . .
    RUN pnpm build
    RUN pnpm ui:install
    RUN pnpm ui:build
    ENV NODE_ENV=production
    CMD ["node","dist/index.js"]
    ```

  </Accordion>

  <Accordion title="高级用户容器选项">
    默认镜像以安全优先为设计目标，并以非 root 的 `node` 用户运行。若要使用
    功能更完整的容器：

    1. **持久化 `/home/node`**：`export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **预装系统依赖**：`export OPENCLAW_DOCKER_APT_PACKAGES="git curl jq"`
    3. **安装 Playwright 浏览器**：
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    4. **持久化浏览器下载内容**：设置
       `PLAYWRIGHT_BROWSERS_PATH=/home/node/.cache/ms-playwright`，并使用
       `OPENCLAW_HOME_VOLUME` 或 `OPENCLAW_EXTRA_MOUNTS`。

  </Accordion>

  <Accordion title="OpenAI Codex OAuth（无头 Docker）">
    如果你在向导中选择 OpenAI Codex OAuth，它会打开一个浏览器 URL。在
    Docker 或无头环境中，请复制你最终跳转到的完整重定向 URL，并将其粘贴回
    向导以完成认证。
  </Accordion>

  <Accordion title="基础镜像元数据">
    主 Docker 镜像使用 `node:24-bookworm`，并发布 OCI 基础镜像注释，
    包括 `org.opencontainers.image.base.name`、
    `org.opencontainers.image.source` 等。参见
    [OCI 镜像注释](https://github.com/opencontainers/image-spec/blob/main/annotations.md)。
  </Accordion>
</AccordionGroup>

### 在 VPS 上运行？

有关共享 VM 部署步骤，包括二进制预构建、持久化和更新，请参见
[Hetzner（Docker VPS）](/zh-CN/install/hetzner) 和
[Docker VM Runtime](/zh-CN/install/docker-vm-runtime)。

## 智能体沙箱

当启用 `agents.defaults.sandbox` 时，Gateway 网关会在隔离的 Docker 容器中运行智能体工具执行
（shell、文件读写等），而网关本身仍然运行在主机上。这样你就可以在不将整个
Gateway 网关容器化的前提下，为不受信任或多租户的智能体会话提供一道硬隔离边界。

沙箱范围可以是每个智能体（默认）、每个会话，或共享。每种范围都会获得
各自挂载到 `/workspace` 的工作区。你还可以配置
工具允许/拒绝策略、网络隔离、资源限制和浏览器容器。

有关完整配置、镜像、安全说明和多智能体配置文件，请参见：

- [沙箱隔离](/zh-CN/gateway/sandboxing) -- 完整沙箱参考
- [OpenShell](/zh-CN/gateway/openshell) -- 对沙箱容器的交互式 shell 访问
- [多智能体沙箱和工具](/zh-CN/tools/multi-agent-sandbox-tools) -- 每个智能体的覆盖配置

### 快速启用

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        scope: "agent", // session | agent | shared
      },
    },
  },
}
```

构建默认沙箱镜像：

```bash
scripts/sandbox-setup.sh
```

## 故障排除

<AccordionGroup>
  <Accordion title="镜像缺失或沙箱容器无法启动">
    请使用
    [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh)
    构建沙箱镜像，或将 `agents.defaults.sandbox.docker.image` 设置为你的自定义镜像。
    容器会在每个会话按需自动创建。
  </Accordion>

  <Accordion title="沙箱中的权限错误">
    将 `docker.user` 设置为与你挂载工作区所有权匹配的 UID:GID，
    或对工作区文件夹执行 chown。
  </Accordion>

  <Accordion title="在沙箱中找不到自定义工具">
    OpenClaw 使用 `sh -lc`（登录 shell）运行命令，这会读取
    `/etc/profile`，并可能重置 PATH。请设置 `docker.env.PATH` 来预先添加你的
    自定义工具路径，或在你的 Dockerfile 中向 `/etc/profile.d/` 添加脚本。
  </Accordion>

  <Accordion title="镜像构建期间因 OOM 被终止（退出码 137）">
    VM 至少需要 2 GB 内存。请改用更大的机器规格后重试。
  </Accordion>

  <Accordion title="Control UI 中显示 Unauthorized 或需要配对">
    获取一个新的 dashboard 链接并批准该浏览器设备：

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    更多细节：[Dashboard](/web/dashboard)、[Devices](/cli/devices)。

  </Accordion>

  <Accordion title="Gateway 目标显示 ws://172.x.x.x，或从 Docker CLI 出现配对错误">
    重置网关模式和 bind：

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## 相关内容

- [安装概览](/zh-CN/install) — 所有安装方式
- [Podman](/zh-CN/install/podman) — Docker 的 Podman 替代方案
- [ClawDock](/zh-CN/install/clawdock) — 基于 Docker Compose 的社区设置
- [更新](/zh-CN/install/updating) — 让 OpenClaw 保持最新
- [配置](/zh-CN/gateway/configuration) — 安装后的 Gateway 网关配置
