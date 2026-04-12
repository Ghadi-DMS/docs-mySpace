---
read_when:
    - 公開リリースチャネルの定義を探している
    - バージョン命名とリリース頻度を探している
summary: 公開リリースチャネル、バージョン命名、およびリリース頻度
title: リリースポリシー
x-i18n:
    generated_at: "2026-04-12T23:34:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: dffc1ee5fdbb20bd1bf4b3f817d497fc0d87f70ed6c669d324fea66dc01d0b0b
    source_path: reference/RELEASING.md
    workflow: 15
---

# リリースポリシー

OpenClaw には 3 つの公開リリースレーンがあります。

- stable: タグ付きリリースで、デフォルトでは npm `beta` に公開され、明示的に指定された場合は npm `latest` に公開される
- beta: npm `beta` に公開されるプレリリースタグ
- dev: `main` の移動する先頭

## バージョン命名

- stable リリースバージョン: `YYYY.M.D`
  - Git タグ: `vYYYY.M.D`
- stable 修正リリースバージョン: `YYYY.M.D-N`
  - Git タグ: `vYYYY.M.D-N`
- beta プレリリースバージョン: `YYYY.M.D-beta.N`
  - Git タグ: `vYYYY.M.D-beta.N`
- 月や日はゼロ埋めしない
- `latest` は現在昇格済みの stable npm リリースを意味する
- `beta` は現在の beta インストール対象を意味する
- stable および stable 修正リリースは、デフォルトで npm `beta` に公開される。リリース運用者は明示的に `latest` を指定することも、後で検証済み beta ビルドを昇格させることもできる
- すべての OpenClaw リリースは npm パッケージと macOS アプリを一緒に出荷する

## リリース頻度

- リリースは beta-first で進む
- stable は最新 beta の検証後にのみ続く
- 詳細なリリース手順、承認、認証情報、復旧メモは
  maintainer 専用

## リリース事前確認

- パック検証ステップに必要な
  `dist/*` リリース成果物と Control UI バンドルを用意するため、
  `pnpm release:check` の前に `pnpm build && pnpm ui:build` を実行する
- すべてのタグ付きリリースの前に `pnpm release:check` を実行する
- リリースチェックは現在、別の手動ワークフローで実行される:
  `OpenClaw Release Checks`
- この分離は意図的なものであり、実際の npm リリース経路を短く、
  決定的で、成果物中心に保ちつつ、低速なライブチェックは独自の
  レーンに残して公開を遅らせたりブロックしたりしないようにする
- リリースチェックは、ワークフローロジックと secrets を正準に保つため、
  `main` ワークフロー ref から実行しなければならない
- そのワークフローは、既存のリリースタグまたは現在の完全な
  40 文字の `main` commit SHA のいずれかを受け付ける
- commit-SHA モードでは、現在の `origin/main` HEAD のみを受け付ける。古いリリース commit には
  リリースタグを使う
- `OpenClaw NPM Release` の検証専用事前確認も、プッシュ済みタグを必要とせずに
  現在の完全な 40 文字 `main` commit SHA を受け付ける
- その SHA 経路は検証専用であり、実際の公開には昇格できない
- SHA モードでは、ワークフローはパッケージメタデータ確認のためだけに
  `v<package.json version>` を合成する。実際の公開には依然として実際のリリースタグが必要
- 両方のワークフローは、実際の公開と昇格経路を GitHub ホスト型
  runner 上に保ちつつ、非破壊の検証経路ではより大きい
  Blacksmith Linux runner を使えるようにする
- そのワークフローは、
  `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache`
  を、`OPENAI_API_KEY` と `ANTHROPIC_API_KEY` の両方の workflow secrets を使って実行する
- npm リリース事前確認は、もはや別のリリースチェックレーンを待たない
- 承認前に
  `RELEASE_TAG=vYYYY.M.D node --import tsx scripts/openclaw-npm-release-check.ts`
  （または対応する beta/修正タグ）を実行する
- npm 公開後、公開されたレジストリの
  インストール経路を新しい一時 prefix で検証するために、
  `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.D`
  （または対応する beta/修正バージョン）を実行する
- maintainer のリリース自動化は現在、事前確認してから昇格する方式を使う:
  - 実際の npm 公開は、成功した npm `preflight_run_id` を通過しなければならない
  - stable npm リリースはデフォルトで `beta`
  - stable npm 公開は、ワークフロー入力で明示的に `latest` を指定できる
  - stable npm の `beta` から `latest` への昇格も、信頼された `OpenClaw NPM Release` ワークフロー上の明示的な手動モードとして引き続き利用可能
  - その昇格モードでも、npm の `dist-tag` 管理は信頼された公開とは別であるため、`npm-release` 環境に有効な `NPM_TOKEN` が引き続き必要
  - 公開の `macOS Release` は検証専用
  - 実際の非公開 mac 公開は、成功した非公開 mac の
    `preflight_run_id` と `validate_run_id` を通過しなければならない
  - 実際の公開経路は、成果物を再ビルドするのではなく、準備済み成果物を昇格する
- `YYYY.M.D-N` のような stable 修正リリースでは、公開後検証器は
  `YYYY.M.D` から `YYYY.M.D-N` への同じ temp-prefix アップグレード経路も確認するため、
  リリース修正によって古いグローバルインストールが基準 stable ペイロードのまま
  気づかれずに残ることを防ぐ
- npm リリース事前確認は、tarball に
  `dist/control-ui/index.html` と空でない `dist/control-ui/assets/` ペイロードの両方が含まれていない限り
  fail closed する。これにより空のブラウザダッシュボードを再び出荷しないようにする
- リリース作業が CI 計画、extension タイミング manifest、または
  extension テストマトリクスに触れた場合は、承認前に `.github/workflows/ci.yml` から
  planner 所有の `checks-node-extensions` ワークフローマトリクス出力を再生成して確認し、
  リリースノートが古い CI レイアウトを説明しないようにする
- stable macOS リリース準備には updater 関連サーフェスも含まれる:
  - GitHub リリースには、パッケージ化された `.zip`、`.dmg`、`.dSYM.zip` が最終的に含まれていなければならない
  - `main` 上の `appcast.xml` は、公開後に新しい stable zip を指していなければならない
  - パッケージ化されたアプリは、非デバッグ bundle id、空でない Sparkle feed
    URL、およびそのリリースバージョンに対する正準 Sparkle build floor 以上の `CFBundleVersion` を維持しなければならない

## NPM ワークフロー入力

`OpenClaw NPM Release` は、運用者が制御する次の入力を受け付けます。

- `tag`: `v2026.4.2`、`v2026.4.2-1`、または
  `v2026.4.2-beta.1` のような必須のリリースタグ。`preflight_only=true` の場合は、検証専用事前確認用として
  現在の完全な 40 文字 `main` commit SHA でもよい
- `preflight_only`: 検証/ビルド/パッケージのみなら `true`、実際の公開経路なら `false`
- `preflight_run_id`: 実際の公開経路で必須。ワークフローが成功した事前確認実行の準備済み tarball を再利用するため
- `npm_dist_tag`: 公開経路用の npm 対象タグ。デフォルトは `beta`
- `promote_beta_to_latest`: `true` の場合、公開をスキップし、すでに公開済みの
  stable `beta` ビルドを `latest` に移動する

`OpenClaw Release Checks` は、運用者が制御する次の入力を受け付けます。

- `ref`: 検証対象の既存リリースタグ、または現在の完全な 40 文字 `main` commit
  SHA

ルール:

- stable および修正タグは `beta` または `latest` のどちらにも公開できる
- beta プレリリースタグは `beta` にのみ公開できる
- 完全な commit SHA 入力は `preflight_only=true` の場合にのみ許可される
- リリースチェックの commit-SHA モードでも、現在の `origin/main` HEAD が必要
- 実際の公開経路は、事前確認で使用したものと同じ `npm_dist_tag` を使わなければならず、
  ワークフローは公開継続前にそのメタデータを検証する
- 昇格モードでは、stable または修正タグ、`preflight_only=false`、
  空の `preflight_run_id`、および `npm_dist_tag=beta` を使用しなければならない
- 昇格モードでも、`npm dist-tag add` には通常の npm auth が必要なため、
  `npm-release` 環境に有効な `NPM_TOKEN` が必要

## Stable npm リリース手順

stable npm リリースを作成する場合:

1. `preflight_only=true` で `OpenClaw NPM Release` を実行する
   - タグがまだ存在しない場合、事前確認ワークフローの検証専用 dry run として
     現在の完全な `main` commit SHA を使用できる
2. 通常の beta-first フローでは `npm_dist_tag=beta` を選び、意図的に
   stable を直接公開したい場合にのみ `latest` を選ぶ
3. ライブのプロンプトキャッシュ検証が必要な場合は、同じタグまたは
   現在の完全な `main` commit SHA で `OpenClaw Release Checks` を別途実行する
   - これは、長時間かかるまたは不安定なチェックを公開ワークフローに再結合せず、
     ライブ検証を利用可能なままにするため、意図的に分離されている
4. 成功した `preflight_run_id` を保存する
5. `preflight_only=false`、同じ `tag`、同じ `npm_dist_tag`、
   保存した `preflight_run_id` で再度 `OpenClaw NPM Release` を実行する
6. リリースが `beta` に着地した場合、その公開済みビルドを `latest` に移動したいときは、
   後で同じ stable `tag`、`promote_beta_to_latest=true`、`preflight_only=false`、
   `preflight_run_id` は空、`npm_dist_tag=beta` で `OpenClaw NPM Release` を実行する

昇格モードでも、`npm-release` 環境の承認と、その環境内の有効な
`NPM_TOKEN` が必要です。

これにより、直接公開経路と beta-first 昇格経路の両方が文書化され、
運用者から見える状態に保たれます。

## 公開参照

- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

maintainer は、実際のランブックについて
[`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md)
にある非公開リリースドキュメントを使用します。
