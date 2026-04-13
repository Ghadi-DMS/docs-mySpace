---
read_when:
    - 查找公开发布渠道的定义
    - 查找版本命名和发布节奏
summary: 公开发布渠道、版本命名和发布节奏
title: 发布策略
x-i18n:
    generated_at: "2026-04-13T13:00:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 402c7f8c38323f735df79611ae3272ffc3816dfd39dd73048b9cab2715fdc334
    source_path: reference/RELEASING.md
    workflow: 15
---

# 发布策略

OpenClaw 有三个公开发布渠道：

- stable：带标签的发布版本，默认发布到 npm `beta`，或在明确指定时发布到 npm `latest`
- beta：预发布标签，发布到 npm `beta`
- dev：`main` 的持续更新头部版本

## 版本命名

- 稳定版发布版本：`YYYY.M.D`
  - Git 标签：`vYYYY.M.D`
- 稳定版修正版发布版本：`YYYY.M.D-N`
  - Git 标签：`vYYYY.M.D-N`
- Beta 预发布版本：`YYYY.M.D-beta.N`
  - Git 标签：`vYYYY.M.D-beta.N`
- 月份和日期不要补零
- `latest` 表示当前已提升为正式版的稳定 npm 发布版本
- `beta` 表示当前的 beta 安装目标
- 稳定版和稳定版修正版默认发布到 npm `beta`；发布操作人员可以明确指定目标为 `latest`，或者稍后再提升已验证的 beta 构建版本
- 每个 OpenClaw 发布都会同时交付 npm 包和 macOS 应用

## 发布节奏

- 发布遵循 beta-first 流程
- 只有在最新 beta 完成验证后，才会跟进 stable
- 详细的发布流程、审批、凭证和恢复说明仅对维护者开放

## 发布前检查

- 在运行 `pnpm release:check` 之前，先运行 `pnpm build && pnpm ui:build`，以便在打包校验步骤中生成预期的 `dist/*` 发布产物和 Control UI bundle
- 每次带标签发布之前都要运行 `pnpm release:check`
- 发布检查现在在单独的手动工作流中运行：
  `OpenClaw Release Checks`
- 这种拆分是有意设计的：让真正的 npm 发布路径保持简短、可预测，并聚焦于产物，而较慢的实时检查则保留在它自己的流程中，避免拖慢或阻塞发布
- 发布检查必须从 `main` 工作流引用发起，这样工作流逻辑和 secrets 才能保持权威一致
- 该工作流接受现有发布标签，或者当前完整的 40 字符 `main` 提交 SHA
- 在提交 SHA 模式下，它只接受当前 `origin/main` 的 HEAD；如果要验证更早的发布提交，请使用发布标签
- `OpenClaw NPM Release` 的仅验证 preflight 也接受当前完整的 40 字符 `main` 提交 SHA，而不要求先推送标签
- 该 SHA 路径仅用于验证，不能提升为真实发布
- 在 SHA 模式下，该工作流只会为包元数据检查合成 `v<package.json version>`；真实发布仍然需要真实的发布标签
- 这两个工作流都会将真实的发布和提升路径保留在 GitHub 托管 runner 上，而非变更性的验证路径则可以使用更大的 Blacksmith Linux runner
- 该工作流会运行
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  并使用工作流 secrets `OPENAI_API_KEY` 和 `ANTHROPIC_API_KEY`
- npm 发布 preflight 不再等待单独的发布检查流程完成
- 在审批之前，运行 `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  （或对应的 beta/修正版标签）
- npm 发布后，运行
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  （或对应的 beta/修正版版本），以在全新的临时前缀中验证已发布注册表安装路径
- 维护者发布自动化现在使用“先 preflight 再 promote”的方式：
  - 真实的 npm 发布必须通过成功的 npm `preflight_run_id`
  - 稳定版 npm 发布默认目标是 `beta`
  - 稳定版 npm 发布可以通过工作流输入明确指定目标为 `latest`
  - 仍然支持通过可信的 `OpenClaw NPM Release` 工作流，以明确的手动模式将稳定版 npm 从 `beta` 提升到 `latest`
  - 直接稳定版发布也可以运行显式的 dist-tag 同步模式，将 `latest` 和 `beta` 都指向已发布的稳定版版本
  - 这些 dist-tag 模式仍然需要在 `npm-release` 环境中提供有效的 `NPM_TOKEN`，因为 npm `dist-tag` 管理独立于可信发布
  - 公开的 `macOS Release` 仅用于验证
  - 真实的私有 mac 发布必须通过成功的私有 mac
    `preflight_run_id` 和 `validate_run_id`
  - 真实发布路径会提升已准备好的产物，而不是再次重新构建它们
- 对于 `YYYY.M.D-N` 这样的稳定版修正版发布，发布后验证器还会检查同一个临时前缀下从 `YYYY.M.D` 升级到 `YYYY.M.D-N` 的路径，这样发布修正就不会悄悄让旧的全局安装仍停留在基础稳定版产物上
- npm 发布 preflight 采用失败即关闭的策略，除非 tarball 同时包含 `dist/control-ui/index.html` 和非空的 `dist/control-ui/assets/` 内容，否则会失败，这样我们就不会再次发布一个空的浏览器仪表板
- 如果此次发布工作涉及 CI 规划、扩展时序清单或扩展测试矩阵，请在审批前从 `.github/workflows/ci.yml` 重新生成并审查由规划器维护的 `checks-node-extensions` 工作流矩阵输出，以免发布说明描述的是过时的 CI 布局
- 稳定版 macOS 发布就绪还包括更新器相关表面：
  - GitHub 发布最终必须包含打包后的 `.zip`、`.dmg` 和 `.dSYM.zip`
  - 发布后，`main` 上的 `appcast.xml` 必须指向新的稳定版 zip
  - 打包后的应用必须保持非调试 bundle id、非空的 Sparkle feed URL，以及对应该发布版本规范 Sparkle 构建下限及以上的 `CFBundleVersion`

## NPM 工作流输入

`OpenClaw NPM Release` 接受这些由操作人员控制的输入：

- `tag`：必填的发布标签，例如 `v2026.4.2`、`v2026.4.2-1` 或 `v2026.4.2-beta.1`；当 `preflight_only=true` 时，也可以是当前完整的 40 字符 `main` 提交 SHA，用于仅验证的 preflight
- `preflight_only`：`true` 表示仅做验证/构建/打包，`false` 表示执行真实发布路径
- `preflight_run_id`：真实发布路径中必填，这样工作流才能复用成功 preflight 运行中准备好的 tarball
- `npm_dist_tag`：发布路径的 npm 目标标签；默认为 `beta`
- `promote_beta_to_latest`：`true` 表示跳过发布，并将已发布的稳定版 `beta` 构建移动到 `latest`
- `sync_stable_dist_tags`：`true` 表示跳过发布，并将 `latest` 和 `beta` 都指向已发布的稳定版版本

`OpenClaw Release Checks` 接受这些由操作人员控制的输入：

- `ref`：要验证的现有发布标签，或当前完整的 40 字符 `main` 提交 SHA

规则：

- 稳定版和修正版标签可以发布到 `beta` 或 `latest`
- Beta 预发布标签只能发布到 `beta`
- 完整提交 SHA 输入仅在 `preflight_only=true` 时允许
- 发布检查的提交 SHA 模式也要求必须是当前 `origin/main` HEAD
- 真实发布路径必须使用与 preflight 相同的 `npm_dist_tag`；工作流会在继续发布之前校验该元数据
- 提升模式必须使用稳定版或修正版标签、`preflight_only=false`、空的 `preflight_run_id`，以及 `npm_dist_tag=beta`
- dist-tag 同步模式必须使用稳定版或修正版标签、`preflight_only=false`、空的 `preflight_run_id`、`npm_dist_tag=latest`，以及 `promote_beta_to_latest=false`
- 提升模式和 dist-tag 同步模式也都需要在 `npm-release` 环境中提供有效的 `NPM_TOKEN`，因为 `npm dist-tag add` 仍然需要常规 npm 身份验证

## 稳定版 npm 发布顺序

在发布稳定版 npm 版本时：

1. 使用 `preflight_only=true` 运行 `OpenClaw NPM Release`
   - 在标签尚不存在时，你可以使用当前完整的 `main` 提交 SHA，对 preflight 工作流进行一次仅验证的 dry run
2. 选择 `npm_dist_tag=beta` 以采用常规的 beta-first 流程，或仅在你明确希望直接发布稳定版时选择 `latest`
3. 使用相同的标签运行单独的 `OpenClaw Release Checks`，或者当你想获得实时 prompt cache 覆盖时，使用当前完整的 `main` 提交 SHA
   - 这样单独拆分是有意为之，这样实时覆盖就能继续可用，而不会再次把耗时长或不稳定的检查耦合回发布工作流
4. 保存成功的 `preflight_run_id`
5. 再次运行 `OpenClaw NPM Release`，并设置 `preflight_only=false`、相同的 `tag`、相同的 `npm_dist_tag` 以及保存的 `preflight_run_id`
6. 如果该发布先落在 `beta`，那么之后当你希望将该已发布构建移动到 `latest` 时，使用相同的稳定版 `tag`、`promote_beta_to_latest=true`、`preflight_only=false`、空的 `preflight_run_id` 和 `npm_dist_tag=beta` 再运行一次 `OpenClaw NPM Release`
7. 如果该发布是有意直接发布到 `latest`，并且也希望 `beta` 跟随同一个稳定版构建，那么使用相同的稳定版 `tag`、`sync_stable_dist_tags=true`、`promote_beta_to_latest=false`、`preflight_only=false`、空的 `preflight_run_id` 和 `npm_dist_tag=latest` 运行 `OpenClaw NPM Release`

提升模式和 dist-tag 同步模式仍然需要 `npm-release` 环境审批，以及该环境中的有效 `NPM_TOKEN`。

这让直接发布路径和 beta-first 提升路径都保持有文档记录，并且对操作人员可见。

## 公开参考

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

维护者会使用私有发布文档
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
作为实际操作手册。
