---
read_when:
    - 正在查找公开发布渠道的定义
    - 正在查找版本命名和发布节奏
summary: 公开发布渠道、版本命名和发布节奏
title: 发布策略
x-i18n:
    generated_at: "2026-04-10T23:44:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: ca613d094c93670c012f0b79720fad0d5d85be802f54b0acb7a8f22aca5bde12
    source_path: reference/RELEASING.md
    workflow: 15
---

# 发布策略

OpenClaw 有三条公开发布渠道：

- stable：带标签的发布版本，默认发布到 npm `beta`，或在明确要求时发布到 npm `latest`
- beta：预发布标签，发布到 npm `beta`
- dev：`main` 的持续更新头部版本

## 版本命名

- Stable 发布版本：`YYYY.M.D`
  - Git 标签：`vYYYY.M.D`
- Stable 修正版发布版本：`YYYY.M.D-N`
  - Git 标签：`vYYYY.M.D-N`
- Beta 预发布版本：`YYYY.M.D-beta.N`
  - Git 标签：`vYYYY.M.D-beta.N`
- 月份和日期不要补零
- `latest` 表示当前已提升为正式版的 stable npm 发布版本
- `beta` 表示当前 beta 安装目标
- Stable 和 stable 修正版发布默认发布到 npm `beta`；发布操作人员也可以显式指定发布到 `latest`，或者稍后再将已验证的 beta 构建提升过去
- 每个 OpenClaw 发布都会同时交付 npm 包和 macOS 应用

## 发布节奏

- 发布先走 beta
- 只有在最新 beta 完成验证后，才会进入 stable
- 详细的发布流程、审批、凭证和恢复说明仅限维护者查看

## 发布前检查

- 在运行 `pnpm release:check` 之前，先运行 `pnpm build && pnpm ui:build`，这样 pack 校验步骤所需的 `dist/*` 发布产物和 Control UI bundle 才会存在
- 每次带标签发布前都要运行 `pnpm release:check`
- `main` 分支上的 npm 发布前检查还会在打包 tarball 之前运行 `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`，并使用 `OPENAI_API_KEY` 和 `ANTHROPIC_API_KEY` 这两个工作流密钥
- 在批准前运行 `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`（或对应的 beta / 修正版标签）
- npm 发布后，运行 `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`（或对应的 beta / 修正版版本），以在一个全新的临时前缀中验证已发布注册表安装路径
- 维护者发布自动化现在使用“先预检，再提升”的流程：
  - 真正的 npm 发布必须依赖一次成功的 npm `preflight_run_id`
  - stable npm 发布默认目标为 `beta`
  - stable npm 发布可以通过工作流输入显式指定目标为 `latest`
  - 也仍然支持在受信任的 `OpenClaw NPM Release` 工作流中，以显式手动模式将 stable 从 `beta` 提升到 `latest`
  - 该提升模式仍然需要 `npm-release` 环境中存在有效的 `NPM_TOKEN`，因为 npm `dist-tag` 管理与受信任发布是分开的
  - 公开的 `macOS Release` 仅用于验证
  - 真正的私有 mac 发布必须依赖成功的私有 mac `preflight_run_id` 和 `validate_run_id`
  - 真正的发布路径会提升已准备好的产物，而不是再次重新构建它们
- 对于像 `YYYY.M.D-N` 这样的 stable 修正版发布，发布后验证器还会检查从 `YYYY.M.D` 升级到 `YYYY.M.D-N` 的同一临时前缀升级路径，这样修正版发布就不能在无声无息中让旧的全局安装仍停留在基础 stable 负载上
- npm 发布前检查默认采用失败即关闭的策略，除非 tarball 同时包含 `dist/control-ui/index.html` 和非空的 `dist/control-ui/assets/` 内容，否则检查不会通过，以避免再次发布一个空的浏览器仪表盘
- 如果发布工作涉及 CI 规划、扩展时序清单或扩展测试矩阵，请在批准前重新生成并审查来自 `.github/workflows/ci.yml` 的规划器维护的 `checks-node-extensions` 工作流矩阵输出，以免发布说明描述过时的 CI 布局
- Stable macOS 发布就绪检查还包括更新器相关表面：
  - GitHub release 最终必须包含打包后的 `.zip`、`.dmg` 和 `.dSYM.zip`
  - 发布后，`main` 上的 `appcast.xml` 必须指向新的 stable zip
  - 打包后的应用必须保持非调试 bundle id、非空的 Sparkle feed URL，以及不低于该发布版本规范 Sparkle 构建底线的 `CFBundleVersion`

## NPM 工作流输入

`OpenClaw NPM Release` 接受以下由操作人员控制的输入：

- `tag`：必填发布标签，例如 `v2026.4.2`、`v2026.4.2-1` 或 `v2026.4.2-beta.1`
- `preflight_only`：`true` 表示仅执行验证 / 构建 / 打包，`false` 表示执行真实发布路径
- `preflight_run_id`：真实发布路径中必填，这样工作流可以复用成功预检运行中准备好的 tarball
- `npm_dist_tag`：发布路径的 npm 目标标签；默认是 `beta`
- `promote_beta_to_latest`：`true` 表示跳过发布，并将一个已经发布的 stable `beta` 构建移动到 `latest`

规则：

- Stable 和修正版标签可以发布到 `beta` 或 `latest`
- Beta 预发布标签只能发布到 `beta`
- 真实发布路径必须使用与预检时相同的 `npm_dist_tag`；工作流会在继续发布前校验该元数据
- 提升模式必须使用 stable 或修正版标签、`preflight_only=false`、空的 `preflight_run_id`，以及 `npm_dist_tag=beta`
- 提升模式还要求 `npm-release` 环境中存在有效的 `NPM_TOKEN`，因为 `npm dist-tag add` 仍然需要常规 npm 认证

## Stable npm 发布顺序

在发布 stable npm 版本时：

1. 以 `preflight_only=true` 运行 `OpenClaw NPM Release`
2. 正常的先 beta 后 stable 流程请选择 `npm_dist_tag=beta`；只有在你明确想直接发布 stable 时才选择 `latest`
3. 保存成功的 `preflight_run_id`
4. 再次运行 `OpenClaw NPM Release`，这次使用 `preflight_only=false`，并传入相同的 `tag`、相同的 `npm_dist_tag`，以及上一步保存的 `preflight_run_id`
5. 如果该发布先落在 `beta`，之后当你想把该已发布构建移动到 `latest` 时，再用相同的 stable `tag`、`promote_beta_to_latest=true`、`preflight_only=false`、空的 `preflight_run_id` 和 `npm_dist_tag=beta` 运行 `OpenClaw NPM Release`

提升模式仍然需要 `npm-release` 环境审批以及该环境中的有效 `NPM_TOKEN`。

这样一来，直接发布路径和先 beta 后提升路径都会被记录下来，并且对操作人员清晰可见。

## 公开参考

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

维护者会使用私有发布文档 [`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md) 作为实际操作手册。
