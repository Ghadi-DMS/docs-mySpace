---
read_when:
    - 公開リリースチャネルの定義を探しています
    - バージョン命名とリリース頻度を探しています
summary: 公開リリースチャネル、バージョン命名、およびリリース頻度
title: リリースポリシー
x-i18n:
    generated_at: "2026-04-11T02:48:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: ca613d094c93670c012f0b79720fad0d5d85be802f54b0acb7a8f22aca5bde12
    source_path: reference/RELEASING.md
    workflow: 15
---

# リリースポリシー

OpenClawには3つの公開リリースレーンがあります。

- stable: デフォルトでnpmの`beta`へ公開されるタグ付きリリース。明示的に要求された場合はnpmの`latest`へ公開
- beta: npmの`beta`へ公開されるプレリリースタグ
- dev: `main`の移動する先頭

## バージョン命名

- Stableリリースバージョン: `YYYY.M.D`
  - Git tag: `vYYYY.M.D`
- Stable修正版リリースバージョン: `YYYY.M.D-N`
  - Git tag: `vYYYY.M.D-N`
- Betaプレリリースバージョン: `YYYY.M.D-beta.N`
  - Git tag: `vYYYY.M.D-beta.N`
- 月や日はゼロ埋めしない
- `latest` は現在昇格済みのstableなnpmリリースを意味する
- `beta` は現在のbetaインストール対象を意味する
- Stableおよびstable修正版リリースは、デフォルトでnpmの`beta`へ公開される。リリース運用者は明示的に`latest`を対象にすることも、検証済みのbetaビルドを後で昇格させることもできる
- すべてのOpenClawリリースはnpmパッケージとmacOSアプリを一緒に出荷する

## リリース頻度

- リリースはbeta-firstで進む
- Stableは、最新のbetaが検証された後にのみ続く
- 詳細なリリース手順、承認、認証情報、リカバリーノートはメンテナー専用

## リリース事前確認

- `pnpm release:check` の前に `pnpm build && pnpm ui:build` を実行して、pack検証ステップに必要な `dist/*` リリース成果物とControl UIバンドルを用意する
- すべてのタグ付きリリース前に `pnpm release:check` を実行する
- mainブランチのnpm事前確認では、tarballをパッケージ化する前に、`OPENAI_API_KEY` と `ANTHROPIC_API_KEY` の両workflow secretを使って `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache` も実行する
- 承認前に `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`（または対応するbeta/修正版タグ）を実行する
- npm公開後に、公開済みレジストリのインストール経路を新しい一時prefixで検証するため、`node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`（または対応するbeta/修正版バージョン）を実行する
- Maintainerのリリース自動化は現在、preflight-then-promoteを使う:
  - 実際のnpm公開は、成功したnpm `preflight_run_id` を通過していなければならない
  - stableなnpmリリースはデフォルトで`beta`
  - stableなnpm公開は、workflow inputで明示的に`latest`を対象にできる
  - stableなnpmの`beta`から`latest`への昇格は、信頼された`OpenClaw NPM Release` workflow上で引き続き明示的な手動モードとして利用可能
  - その昇格モードでも、npmの`dist-tag`管理はtrusted publishingとは別であるため、`npm-release` environment内に有効な`NPM_TOKEN`が必要
  - 公開の`macOS Release`は検証専用
  - 実際の非公開mac公開は、成功した非公開macの `preflight_run_id` と `validate_run_id` を通過していなければならない
  - 実際の公開経路では、再ビルドではなく準備済み成果物を昇格させる
- `YYYY.M.D-N` のようなstable修正版リリースでは、post-publish verifierが `YYYY.M.D` から `YYYY.M.D-N` への同じ一時prefixアップグレード経路も確認するため、リリース修正によって古いグローバルインストールがベースstableペイロードのまま静かに残ることはない
- npmリリース事前確認は、tarballに `dist/control-ui/index.html` と空でない `dist/control-ui/assets/` ペイロードの両方が含まれていない限りclosed failになる。これにより空のブラウザーダッシュボードを再び出荷しないようにする
- リリース作業でCI計画、拡張タイミングマニフェスト、または拡張テストマトリクスに触れた場合は、承認前に `.github/workflows/ci.yml` からplanner所有の `checks-node-extensions` workflow matrix出力を再生成して確認する。これによりリリースノートが古いCIレイアウトを記述しないようにする
- stableなmacOSリリース準備には、updaterサーフェスも含まれる:
  - GitHub releaseには、パッケージ化された `.zip`、`.dmg`、`.dSYM.zip` が最終的に含まれていなければならない
  - `main` 上の `appcast.xml` は、公開後に新しいstable zipを指していなければならない
  - パッケージ化されたアプリは、デバッグでないbundle id、空でないSparkle feed URL、およびそのリリースバージョンに対する正式なSparkle build floor以上の `CFBundleVersion` を維持しなければならない

## NPM workflow入力

`OpenClaw NPM Release` は、運用者が制御する次の入力を受け付けます。

- `tag`: `v2026.4.2`、`v2026.4.2-1`、`v2026.4.2-beta.1` のような必須リリースタグ
- `preflight_only`: 検証/ビルド/パッケージのみの場合は `true`、実際の公開経路の場合は `false`
- `preflight_run_id`: 実際の公開経路で必須。workflowが成功した事前確認実行から準備済みtarballを再利用するために使う
- `npm_dist_tag`: 公開経路用のnpm対象タグ。デフォルトは `beta`
- `promote_beta_to_latest`: 公開をスキップして、すでに公開済みのstableな`beta`ビルドを`latest`へ移動する場合は `true`

ルール:

- Stableおよび修正版タグは、`beta` または `latest` のどちらにも公開できる
- Betaプレリリースタグは `beta` にのみ公開できる
- 実際の公開経路では、事前確認時に使ったものと同じ `npm_dist_tag` を使わなければならない。workflowは公開続行前にそのメタデータを検証する
- 昇格モードでは、stableまたは修正版タグ、`preflight_only=false`、空の `preflight_run_id`、および `npm_dist_tag=beta` を使わなければならない
- 昇格モードでも、`npm dist-tag add` には通常のnpm認証が必要なため、`npm-release` environment内に有効な `NPM_TOKEN` が必要

## Stable npmリリース手順

stableなnpmリリースを切るとき:

1. `preflight_only=true` で `OpenClaw NPM Release` を実行する
2. 通常のbeta-firstフローでは `npm_dist_tag=beta` を選び、意図的に直接stable公開したい場合にのみ `latest` を選ぶ
3. 成功した `preflight_run_id` を保存する
4. `preflight_only=false`、同じ `tag`、同じ `npm_dist_tag`、保存した `preflight_run_id` で再度 `OpenClaw NPM Release` を実行する
5. リリースが `beta` に入った場合、その公開済みビルドを `latest` に移したいタイミングで、同じstableな `tag`、`promote_beta_to_latest=true`、`preflight_only=false`、空の `preflight_run_id`、`npm_dist_tag=beta` で後から `OpenClaw NPM Release` を実行する

昇格モードでも、`npm-release` environmentの承認と、そのenvironment内の有効な `NPM_TOKEN` が必要です。

これにより、直接公開経路とbeta-first昇格経路の両方が、文書化され、運用者から見える状態に保たれます。

## 公開リファレンス

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

実際のランブックについては、メンテナーは
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
の非公開リリースドキュメントを使用します。
