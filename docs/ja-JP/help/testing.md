---
read_when:
    - ローカルまたは CI でテストを実行する
    - model/provider のバグに対するリグレッションを追加する
    - Gateway + agent の動作をデバッグする
summary: テストキット：unit/e2e/live スイート、Docker ランナー、および各テストでカバーされる内容
title: テスト中
x-i18n:
    generated_at: "2026-04-13T04:46:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3db91b4bc36f626cd014958ec66b08b9cecd9faaa20a5746cd3a49ad4b0b1c38
    source_path: help/testing.md
    workflow: 15
---

# テスト

OpenClaw には 3 つの Vitest スイート（unit/integration、e2e、live）と、少数の Docker ランナーがあります。

このドキュメントは「どのようにテストするか」のガイドです。

- 各スイートが何をカバーするか（そして意図的に _何をカバーしないか_）
- 一般的なワークフロー（ローカル、push 前、デバッグ）でどのコマンドを実行するか
- live テストがどのように認証情報を検出し、モデル/プロバイダーを選択するか
- 実際の model/provider の問題に対するリグレッションをどのように追加するか

## クイックスタート

たいていの日は:

- フルゲート（push 前に想定）: `pnpm build && pnpm check && pnpm test`
- 余裕のあるマシンでの、より高速なローカルフルスイート実行: `pnpm test:max`
- 直接の Vitest watch ループ: `pnpm test:watch`
- 直接のファイル指定は、extension/channel のパスにもルーティングされるようになりました: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- 単一の失敗を反復修正しているときは、まず対象を絞った実行を優先してください。
- Docker ベースの QA サイト: `pnpm qa:lab:up`
- Linux VM ベースの QA レーン: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

テストに触れたときや、さらに確信が欲しいとき:

- カバレッジゲート: `pnpm test:coverage`
- E2E スイート: `pnpm test:e2e`

実際の provider/model をデバッグするとき（実際の認証情報が必要）:

- live スイート（models + Gateway の tool/image プローブ）: `pnpm test:live`
- 単一の live ファイルを静かに対象指定: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

ヒント: 失敗している 1 ケースだけが必要な場合は、以下で説明する allowlist 環境変数を使って live テストを絞り込むことを優先してください。

## QA 固有のランナー

これらのコマンドは、QA-lab レベルの現実性が必要なときに、メインのテストスイートの横に位置づけられます。

- `pnpm openclaw qa suite`
  - リポジトリベースの QA シナリオをホスト上で直接実行します。
  - デフォルトでは、分離された Gateway ワーカーを使って複数の選択シナリオを並列実行し、最大 64 ワーカーまたは選択シナリオ数まで使用します。`--concurrency <count>` でワーカー数を調整するか、以前の直列レーンには `--concurrency 1` を使用してください。
- `pnpm openclaw qa suite --runner multipass`
  - 同じ QA スイートを使い捨ての Multipass Linux VM 内で実行します。
  - ホスト上の `qa suite` と同じシナリオ選択動作を維持します。
  - `qa suite` と同じ provider/model 選択フラグを再利用します。
  - live 実行では、ゲストで実用的なサポート済み QA 認証入力を転送します:
    env ベースの provider key、QA live provider config path、存在する場合の `CODEX_HOME`。
  - 出力ディレクトリは、ゲストがマウントされたワークスペース経由で書き戻せるよう、リポジトリルート配下に置く必要があります。
  - 通常の QA レポート + サマリーに加えて、Multipass ログを `.artifacts/qa-e2e/...` 配下に書き込みます。
- `pnpm qa:lab:up`
  - オペレーター風の QA 作業のために、Docker ベースの QA サイトを起動します。
- `pnpm openclaw qa matrix`
  - 使い捨ての Docker ベース Tuwunel homeserver に対して、Matrix live QA レーンを実行します。
  - 一時的な Matrix ユーザー 3 人（`driver`、`sut`、`observer`）と 1 つのプライベートルームをプロビジョニングし、その後、実際の Matrix Plugin を SUT トランスポートとして使う QA Gateway 子プロセスを起動します。
  - デフォルトでは、固定された stable な Tuwunel イメージ `ghcr.io/matrix-construct/tuwunel:v1.5.1` を使用します。別のイメージをテストする必要がある場合は、`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` で上書きしてください。
  - Matrix は現在、レーンが使い捨てユーザーをローカルでプロビジョニングするため、`--credential-source env` のみをサポートします。
  - Matrix QA レポート、サマリー、および observed-events artifact を `.artifacts/qa-e2e/...` 配下に書き込みます。
- `pnpm openclaw qa telegram`
  - env の driver および SUT bot token を使って、実際のプライベートグループに対して Telegram live QA レーンを実行します。
  - `OPENCLAW_QA_TELEGRAM_GROUP_ID`、`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` が必要です。group id は数値の Telegram chat id である必要があります。
  - 共有プール済み認証情報に対して `--credential-source convex` をサポートします。通常は env モードを使用し、プール済みリースを利用したい場合は `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` を設定してください。
  - 同じプライベートグループ内にある 2 つの異なる bot が必要で、SUT bot は Telegram username を公開している必要があります。
  - 安定した bot-to-bot 観測のために、両方の bot で `@BotFather` の Bot-to-Bot Communication Mode を有効化し、driver bot がグループ内の bot トラフィックを観測できるようにしてください。
  - Telegram QA レポート、サマリー、および observed-messages artifact を `.artifacts/qa-e2e/...` 配下に書き込みます。

live transport レーンは、新しい transport がずれないように、1 つの標準契約を共有します。

`qa-channel` は引き続き広範な synthetic QA スイートであり、live
transport カバレッジマトリクスには含まれません。

| Lane     | Canary | Mention gating | Allowlist block | Top-level reply | Restart resume | Thread follow-up | Thread isolation | Reaction observation | Help command |
| -------- | ------ | -------------- | --------------- | --------------- | -------------- | ---------------- | ---------------- | -------------------- | ------------ |
| Matrix   | x      | x              | x               | x               | x              | x                | x                | x                    |              |
| Telegram | x      |                |                 |                 |                |                  |                  |                      | x            |

### Convex 経由の共有 Telegram 認証情報（v1）

`openclaw qa telegram` に対して `--credential-source convex`（または `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`）を有効にすると、QA lab は Convex ベースのプールから排他的リースを取得し、レーン実行中はそのリースに heartbeat を送り、終了時にリースを解放します。

参照用 Convex project scaffold:

- `qa/convex-credential-broker/`

必要な環境変数:

- `OPENCLAW_QA_CONVEX_SITE_URL`（例: `https://your-deployment.convex.site`）
- 選択したロールに対する 1 つのシークレット:
  - `maintainer` には `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
  - `ci` には `OPENCLAW_QA_CONVEX_SECRET_CI`
- 認証ロール選択:
  - CLI: `--credential-role maintainer|ci`
  - env デフォルト: `OPENCLAW_QA_CREDENTIAL_ROLE`（デフォルトは `maintainer`）

任意の環境変数:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS`（デフォルト `1200000`）
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS`（デフォルト `30000`）
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS`（デフォルト `90000`）
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS`（デフォルト `15000`）
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`（デフォルト `/qa-credentials/v1`）
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID`（任意の trace id）
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` は、ローカル専用開発向けに loopback `http://` Convex URL を許可します。

通常運用では、`OPENCLAW_QA_CONVEX_SITE_URL` は `https://` を使う必要があります。

管理者向け管理コマンド（プールの追加/削除/一覧）には、
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` が特に必要です。

管理者向け CLI ヘルパー:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

スクリプトや CI ユーティリティで機械可読な出力が必要な場合は `--json` を使用してください。

デフォルトのエンドポイント契約（`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`）:

- `POST /acquire`
  - リクエスト: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - 成功: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - 枯渇/再試行可能: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /heartbeat`
  - リクエスト: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - 成功: `{ status: "ok" }`（または空の `2xx`）
- `POST /release`
  - リクエスト: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - 成功: `{ status: "ok" }`（または空の `2xx`）
- `POST /admin/add`（maintainer secret のみ）
  - リクエスト: `{ kind, actorId, payload, note?, status? }`
  - 成功: `{ status: "ok", credential }`
- `POST /admin/remove`（maintainer secret のみ）
  - リクエスト: `{ credentialId, actorId }`
  - 成功: `{ status: "ok", changed, credential }`
  - アクティブなリースのガード: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list`（maintainer secret のみ）
  - リクエスト: `{ kind?, status?, includePayload?, limit? }`
  - 成功: `{ status: "ok", credentials, count }`

Telegram kind の payload 形状:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` は数値の Telegram chat id 文字列である必要があります。
- `admin/add` は `kind: "telegram"` に対してこの形状を検証し、不正な payload を拒否します。

### QA に channel を追加する

markdown QA システムに channel を追加するには、必要なのは正確に 2 つだけです。

1. その channel 用の transport adapter
2. channel 契約を実行する scenario pack

共有の `qa-lab` ランナーがフローを担える場合は、channel 固有の QA ランナーを追加しないでください。

`qa-lab` は共有メカニクスを担います:

- スイートの起動と終了処理
- ワーカー並列実行
- artifact の書き込み
- レポート生成
- シナリオ実行
- 古い `qa-channel` シナリオ向けの互換エイリアス

channel adapter は transport 契約を担います:

- その transport に対して Gateway がどのように設定されるか
- readiness をどのように確認するか
- inbound event をどのように注入するか
- outbound message をどのように観測するか
- transcript と正規化済み transport state をどのように公開するか
- transport ベースの action をどのように実行するか
- transport 固有の reset や cleanup をどのように処理するか

新しい channel を採用する最小条件は次のとおりです。

1. 共有 `qa-lab` seam 上に transport adapter を実装する。
2. transport registry に adapter を登録する。
3. transport 固有のメカニクスは adapter または channel harness の中に閉じ込める。
4. `qa/scenarios/` 配下で markdown シナリオを作成または適応する。
5. 新しいシナリオには汎用 scenario helper を使う。
6. リポジトリが意図的な移行を行っているのでない限り、既存の互換エイリアスを動作させ続ける。

判断ルールは厳格です。

- 振る舞いを `qa-lab` に 1 回だけ表現できるなら、`qa-lab` に置いてください。
- 振る舞いが 1 つの channel transport に依存するなら、その adapter または plugin harness に置いてください。
- シナリオが複数の channel で使える新しい capability を必要とするなら、`suite.ts` に channel 固有の分岐を追加するのではなく、汎用 helper を追加してください。
- 振る舞いが 1 つの transport にしか意味を持たないなら、そのシナリオを transport 固有に保ち、そのことを scenario 契約内で明示してください。

新しいシナリオ向けの推奨される汎用 helper 名は次のとおりです。

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

既存シナリオ向けには、次の互換エイリアスも引き続き利用できます。

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

新しい channel の作業では、汎用 helper 名を使用してください。
互換エイリアスはフラグデー移行を避けるために存在するものであり、
新しいシナリオ作成のモデルではありません。

## テストスイート（どこで何が実行されるか）

これらのスイートは「現実性が増していくもの」（そして flaky さ/コストも増すもの）として考えてください。

### Unit / integration（デフォルト）

- コマンド: `pnpm test`
- 設定: 既存のスコープ付き Vitest project に対する 10 個の順次 shard 実行（`vitest.full-*.config.ts`）
- ファイル: `src/**/*.test.ts`、`packages/**/*.test.ts`、`test/**/*.test.ts` 配下の core/unit inventory と、`vitest.unit.config.ts` でカバーされる許可済みの `ui` node テスト
- スコープ:
  - 純粋な unit テスト
  - プロセス内 integration テスト（Gateway 認証、ルーティング、tooling、パース、config）
  - 既知のバグに対する決定論的なリグレッション
- 期待値:
  - CI で実行される
  - 実際のキーは不要
  - 高速かつ安定しているべき
- Projects 注記:
  - 対象未指定の `pnpm test` は、1 つの巨大な native root-project プロセスではなく、11 個の小さな shard config（`core-unit-src`、`core-unit-security`、`core-unit-ui`、`core-unit-support`、`core-support-boundary`、`core-contracts`、`core-bundled`、`core-runtime`、`agentic`、`auto-reply`、`extensions`）を実行するようになりました。これにより、負荷の高いマシンでのピーク RSS が削減され、auto-reply/extension の処理が無関係なスイートを飢えさせることを防ぎます。
  - `pnpm test --watch` は、マルチ shard の watch ループが現実的ではないため、引き続き native root の `vitest.config.ts` project graph を使用します。
  - `pnpm test`、`pnpm test:watch`、`pnpm test:perf:imports` は、明示的なファイル/ディレクトリ指定をまずスコープ付きレーンにルーティングするため、`pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` では root project 全体の起動コストを支払わずに済みます。
  - `pnpm test:changed` は、差分がルーティング可能な source/test ファイルだけに触れている場合、変更された git パスを同じスコープ付きレーンに展開します。config/setup の編集は、引き続き広い root-project 再実行にフォールバックします。
  - agents、commands、plugins、auto-reply helper、`plugin-sdk`、および類似の純粋なユーティリティ領域からの import-light な unit テストは、`test/setup-openclaw-runtime.ts` をスキップする `unit-fast` レーンにルーティングされます。stateful/runtime-heavy なファイルは既存のレーンに残ります。
  - 選択された `plugin-sdk` および `commands` の helper source ファイルは、changed モードの実行をそれらの light レーンにある明示的な sibling テストにもマップするため、helper 編集時にそのディレクトリの重いフルスイートを再実行せずに済みます。
  - `auto-reply` には現在、3 つの専用バケットがあります: トップレベルの core helper、トップレベルの `reply.*` integration テスト、そして `src/auto-reply/reply/**` サブツリーです。これにより、最も重い reply harness の処理を、軽量な status/chunk/token テストから切り離せます。
- Embedded runner 注記:
  - message-tool の discovery 入力または Compaction の runtime context を変更する場合は、
    両方のレベルのカバレッジを維持してください。
  - 純粋な routing/normalization 境界向けの、焦点を絞った helper リグレッションを追加してください。
  - あわせて、embedded runner の integration スイートも健全に保ってください:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`、
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`、および
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`。
  - これらのスイートは、scoped id と Compaction の挙動が
    実際の `run.ts` / `compact.ts` パスを通って流れ続けることを検証します。helper のみのテストは、
    これらの integration パスの十分な代替にはなりません。
- Pool 注記:
  - ベースの Vitest config は現在デフォルトで `threads` を使用します。
  - 共有 Vitest config は `isolate: false` も固定し、root project、e2e、live config 全体で非分離ランナーを使用します。
  - root UI レーンは `jsdom` の setup と optimizer を維持しますが、現在は共有の非分離ランナー上でも動作します。
  - 各 `pnpm test` shard は、共有 Vitest config から同じ `threads` + `isolate: false` のデフォルトを継承します。
  - 共有の `scripts/run-vitest.mjs` ランチャーは現在、Vitest 子 Node プロセスに対してデフォルトで `--no-maglev` も追加し、大きなローカル実行中の V8 コンパイルのチャーンを減らします。標準の V8 挙動と比較したい場合は、`OPENCLAW_VITEST_ENABLE_MAGLEV=1` を設定してください。
- Fast-local iteration 注記:
  - `pnpm test:changed` は、変更されたパスがより小さいスイートにきれいにマップされる場合、スコープ付きレーンを経由します。
  - `pnpm test:max` と `pnpm test:changed:max` は同じルーティング動作を保ちつつ、worker cap だけを高くします。
  - ローカル worker の自動スケーリングは現在意図的に保守的で、ホストの load average がすでに高い場合にも抑制されるため、複数の並行 Vitest 実行がデフォルトで与えるダメージが小さくなります。
  - ベース Vitest config は project/config ファイルを `forceRerunTriggers` としてマークしており、テスト配線が変わったときにも changed モードの再実行が正しく保たれます。
  - config は、サポートされるホスト上で `OPENCLAW_VITEST_FS_MODULE_CACHE` を有効に保ちます。直接プロファイリング用に明示的なキャッシュ場所を 1 つ使いたい場合は、`OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` を設定してください。
- Perf-debug 注記:
  - `pnpm test:perf:imports` は、Vitest の import-duration レポートと import-breakdown 出力を有効にします。
  - `pnpm test:perf:imports:changed` は、同じプロファイリングビューを `origin/main` 以降に変更されたファイルにスコープします。
- `pnpm test:perf:changed:bench -- --ref <git-ref>` は、そのコミット済み diff に対してルーティングされた `test:changed` を native root-project パスと比較し、wall time と macOS の max RSS を出力します。
- `pnpm test:perf:changed:bench -- --worktree` は、変更済みファイル一覧を `scripts/test-projects.mjs` と root Vitest config にルーティングすることで、現在の dirty tree をベンチマークします。
  - `pnpm test:perf:profile:main` は、Vitest/Vite の起動と transform オーバーヘッドに対する main-thread CPU profile を書き込みます。
  - `pnpm test:perf:profile:runner` は、ファイル並列化を無効にした unit スイートに対する runner CPU+heap profile を書き込みます。

### E2E（Gateway スモーク）

- コマンド: `pnpm test:e2e`
- 設定: `vitest.e2e.config.ts`
- ファイル: `src/**/*.e2e.test.ts`、`test/**/*.e2e.test.ts`
- ランタイムのデフォルト:
  - リポジトリの他の部分に合わせて、Vitest の `threads` と `isolate: false` を使用します。
  - 適応型 worker を使用します（CI: 最大 2、ローカル: デフォルトで 1）。
  - console I/O オーバーヘッドを減らすため、デフォルトで silent mode で実行します。
- 便利な上書き:
  - worker 数を強制するには `OPENCLAW_E2E_WORKERS=<n>`（上限 16）。
  - 詳細な console 出力を再有効化するには `OPENCLAW_E2E_VERBOSE=1`。
- スコープ:
  - 複数インスタンスの Gateway end-to-end 挙動
  - WebSocket/HTTP surface、Node pairing、およびより重いネットワーキング
- 期待値:
  - （パイプラインで有効な場合）CI で実行される
  - 実際のキーは不要
  - unit テストより可動部分が多い（遅くなる可能性がある）

### E2E: OpenShell backend スモーク

- コマンド: `pnpm test:e2e:openshell`
- ファイル: `test/openshell-sandbox.e2e.test.ts`
- スコープ:
  - Docker 経由でホスト上に分離された OpenShell Gateway を起動する
  - 一時的なローカル Dockerfile から sandbox を作成する
  - 実際の `sandbox ssh-config` + SSH exec を通じて OpenClaw の OpenShell backend を実行する
  - sandbox fs bridge を通じて remote-canonical filesystem の挙動を検証する
- 期待値:
  - オプトインのみ。デフォルトの `pnpm test:e2e` 実行には含まれない
  - ローカルの `openshell` CLI と動作する Docker daemon が必要
  - 分離された `HOME` / `XDG_CONFIG_HOME` を使用し、その後テスト用 Gateway と sandbox を破棄する
- 便利な上書き:
  - より広い e2e スイートを手動実行するときにこのテストを有効にするには `OPENCLAW_E2E_OPENSHELL=1`
  - デフォルト以外の CLI バイナリまたは wrapper script を指すには `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`

### Live（実プロバイダー + 実モデル）

- コマンド: `pnpm test:live`
- 設定: `vitest.live.config.ts`
- ファイル: `src/**/*.live.test.ts`
- デフォルト: `pnpm test:live` により **有効**（`OPENCLAW_LIVE_TEST=1` を設定）
- スコープ:
  - 「この provider/model は、実際の認証情報で _今日_ 実際に動くか？」
  - provider のフォーマット変更、tool-calling の癖、認証の問題、rate limit の挙動を捕捉する
- 期待値:
  - 設計上 CI で安定しない（実ネットワーク、実プロバイダーポリシー、クォータ、障害）
  - コストがかかる / rate limit を使用する
  - 「全部」ではなく、対象を絞ったサブセットの実行を優先する
- live 実行は、足りない API key を拾うために `~/.profile` を source します。
- デフォルトでは、live 実行は引き続き `HOME` を分離し、config/auth material を一時テスト用ホームにコピーするため、unit fixture が実際の `~/.openclaw` を変更することはありません。
- live テストで意図的に実際の home directory を使いたい場合のみ、`OPENCLAW_LIVE_USE_REAL_HOME=1` を設定してください。
- `pnpm test:live` は現在、より静かなモードがデフォルトです。`[live] ...` の進捗出力は維持しますが、追加の `~/.profile` 通知を抑制し、Gateway bootstrap ログ/Bonjour の雑音をミュートします。完全な起動ログを再表示したい場合は `OPENCLAW_LIVE_TEST_QUIET=0` を設定してください。
- API key のローテーション（provider 固有）: カンマ/セミコロン形式の `*_API_KEYS` または `*_API_KEY_1`、`*_API_KEY_2`（例: `OPENAI_API_KEYS`、`ANTHROPIC_API_KEYS`、`GEMINI_API_KEYS`）、あるいは live ごとの上書き `OPENCLAW_LIVE_*_KEY` を設定してください。テストは rate limit 応答時に再試行します。
- Progress/heartbeat 出力:
  - live スイートは現在、長い provider 呼び出しが、Vitest の console capture が静かなときでも目に見えて動作中であるよう、進捗行を stderr に出力します。
  - `vitest.live.config.ts` は Vitest の console interception を無効にし、provider/Gateway の進捗行が live 実行中に即座にストリームされるようにします。
  - direct-model の heartbeat は `OPENCLAW_LIVE_HEARTBEAT_MS` で調整します。
  - Gateway/probe の heartbeat は `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` で調整します。

## どのスイートを実行すべきか？

この判断表を使ってください。

- ロジック/テストを編集した: `pnpm test` を実行する（大きく変更した場合は `pnpm test:coverage` も）
- Gateway ネットワーキング / WS protocol / pairing に触れた: `pnpm test:e2e` を追加する
- 「bot が落ちている」/ provider 固有の障害 / tool calling をデバッグしている: 対象を絞った `pnpm test:live` を実行する

## Live: Android Node capability sweep

- テスト: `src/gateway/android-node.capabilities.live.test.ts`
- スクリプト: `pnpm android:test:integration`
- 目的: 接続された Android Node が現在広告している **すべてのコマンド** を呼び出し、コマンド契約の挙動を検証する。
- スコープ:
  - 前提条件付き/手動セットアップ（このスイートはアプリのインストール/起動/pairing を行わない）
  - 選択された Android Node に対する、コマンドごとの Gateway `node.invoke` 検証
- 必要な事前セットアップ:
  - Android アプリがすでに接続され、Gateway と pair 済みであること。
  - アプリが foreground に維持されていること。
  - 合格を期待する capability に対して、権限/キャプチャ同意が付与されていること。
- 任意のターゲット上書き:
  - `OPENCLAW_ANDROID_NODE_ID` または `OPENCLAW_ANDROID_NODE_NAME`。
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`。
- Android の完全なセットアップ詳細: [Android App](/ja-JP/platforms/android)

## Live: model スモーク（profile key）

live テストは、障害を切り分けられるように 2 層に分かれています。

- 「Direct model」は、与えられたキーで provider/model がそもそも応答できるかを教えてくれます。
- 「Gateway smoke」は、その model に対して Gateway+agent の完全なパイプライン（session、history、tools、sandbox policy など）が動作するかを教えてくれます。

### レイヤー 1: 直接 model completion（Gateway なし）

- テスト: `src/agents/models.profiles.live.test.ts`
- 目的:
  - 検出されたモデルを列挙する
  - `getApiKeyForModel` を使って、認証情報を持っているモデルを選択する
  - モデルごとに小さな completion を実行する（必要に応じて対象を絞ったリグレッションも）
- 有効化方法:
  - `pnpm test:live`（または Vitest を直接起動する場合は `OPENCLAW_LIVE_TEST=1`）
- このスイートを実際に実行するには `OPENCLAW_LIVE_MODELS=modern`（または `all`、modern のエイリアス）を設定してください。そうしないと、`pnpm test:live` を Gateway smoke に集中させるためにスキップされます
- モデルの選び方:
  - modern allowlist を実行するには `OPENCLAW_LIVE_MODELS=modern`（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_MODELS=all` は modern allowlist のエイリアス
  - または `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."`（カンマ区切り allowlist）
  - modern/all sweep はデフォルトで、厳選された高シグナル上限を使用します。網羅的な modern sweep には `OPENCLAW_LIVE_MAX_MODELS=0`、より小さい上限には正の数を設定してください。
- プロバイダーの選び方:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（カンマ区切り allowlist）
- キーの取得元:
  - デフォルト: profile store と env fallback
  - **profile store のみ** を強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`
- これが存在する理由:
  - 「provider API が壊れている / キーが無効」なのか、「Gateway agent パイプラインが壊れている」なのかを切り分ける
  - 小さく分離されたリグレッションを収める（例: OpenAI Responses/Codex Responses の reasoning replay + tool-call フロー）

### レイヤー 2: Gateway + dev agent スモーク（`"@openclaw"` が実際に行うこと）

- テスト: `src/gateway/gateway-models.profiles.live.test.ts`
- 目的:
  - プロセス内 Gateway を起動する
  - `agent:dev:*` session を作成/patch する（実行ごとの model override）
  - キー付きモデルを反復し、次を検証する:
    - 「意味のある」応答（tools なし）
    - 実際の tool invocation が動作する（read probe）
    - 任意の追加 tool probe（exec+read probe）
    - OpenAI のリグレッションパス（tool-call-only → follow-up）が動作し続ける
- Probe の詳細（障害をすばやく説明できるように）:
  - `read` probe: テストはワークスペース内に nonce ファイルを書き込み、agent にそれを `read` して nonce をそのまま返すよう依頼します。
  - `exec+read` probe: テストは agent に `exec` で一時ファイルへ nonce を書き込み、その後それを `read` で読み戻すよう依頼します。
  - image probe: テストは生成した PNG（cat + ランダム化されたコード）を添付し、model が `cat <CODE>` を返すことを期待します。
  - 実装参照: `src/gateway/gateway-models.profiles.live.test.ts` および `src/gateway/live-image-probe.ts`。
- 有効化方法:
  - `pnpm test:live`（または Vitest を直接起動する場合は `OPENCLAW_LIVE_TEST=1`）
- モデルの選び方:
  - デフォルト: modern allowlist（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` は modern allowlist のエイリアス
  - または、絞り込むために `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`（またはカンマ区切りリスト）を設定する
  - modern/all の Gateway sweep はデフォルトで厳選された高シグナル上限を使用します。網羅的な modern sweep には `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`、より小さい上限には正の数を設定してください。
- プロバイダーの選び方（「OpenRouter を全部」は避ける）:
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（カンマ区切り allowlist）
- この live テストでは tool + image probe は常に有効です:
  - `read` probe + `exec+read` probe（tool ストレス）
  - image probe は、model が image input のサポートを広告している場合に実行されます
  - フロー（高レベル）:
    - テストは「CAT」+ ランダムコード入りの小さな PNG を生成します（`src/gateway/live-image-probe.ts`）
    - `agent` の `attachments: [{ mimeType: "image/png", content: "<base64>" }]` 経由でそれを送信します
    - Gateway は添付を `images[]` にパースします（`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`）
    - embedded agent は multimodal な user message を model に転送します
    - アサーション: reply に `cat` + そのコードが含まれること（OCR 許容: 小さな誤りは許可）

ヒント: 自分のマシンで何をテストできるか（および正確な `provider/model` id）を確認するには、次を実行してください。

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI backend スモーク（Claude、Codex、Gemini、またはその他のローカル CLI）

- テスト: `src/gateway/gateway-cli-backend.live.test.ts`
- 目的: デフォルト config に触れずに、ローカル CLI backend を使って Gateway + agent パイプラインを検証する。
- backend 固有の smoke デフォルトは、所有 extension の `cli-backend.ts` 定義にあります。
- 有効化:
  - `pnpm test:live`（または Vitest を直接起動する場合は `OPENCLAW_LIVE_TEST=1`）
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- デフォルト:
  - デフォルトの provider/model: `claude-cli/claude-sonnet-4-6`
  - command/args/image の挙動は、所有 CLI backend Plugin metadata から取得されます。
- 上書き（任意）:
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - 実際の image attachment を送るには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`（パスは prompt に注入されます）。
  - prompt 注入ではなく CLI 引数として image file path を渡すには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`。
  - `IMAGE_ARG` が設定されているときに image 引数の渡し方を制御するには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（または `"list"`）。
  - 2 回目の turn を送って resume フローを検証するには `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`。
  - デフォルトの Claude Sonnet -> Opus 同一 session 継続性 probe を無効にするには `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`（選択された model が switch target をサポートするときに強制的に有効にするには `1`）。

例:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Docker レシピ:

```bash
pnpm test:docker:live-cli-backend
```

単一プロバイダーの Docker レシピ:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

注記:

- Docker ランナーは `scripts/test-live-cli-backend-docker.sh` にあります。
- これは、リポジトリ Docker イメージ内で live CLI-backend スモークを非 root の `node` ユーザーとして実行します。
- 所有 extension から CLI smoke metadata を解決し、その後、対応する Linux CLI package（`@anthropic-ai/claude-code`、`@openai/codex`、または `@google/gemini-cli`）を、キャッシュ可能で書き込み可能な prefix `OPENCLAW_DOCKER_CLI_TOOLS_DIR`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）にインストールします。
- `pnpm test:docker:live-cli-backend:claude-subscription` には、`~/.claude/.credentials.json` 内の `claudeAiOauth.subscriptionType`、または `claude setup-token` の `CLAUDE_CODE_OAUTH_TOKEN` のいずれかによる portable な Claude Code subscription OAuth が必要です。これはまず Docker 内で直接の `claude -p` を確認し、その後 Anthropic API-key env vars を保持せずに 2 回の Gateway CLI-backend turn を実行します。この subscription レーンでは、Claude が現在サードパーティアプリ利用を通常の subscription plan 上限ではなく追加利用課金にルーティングするため、Claude MCP/tool および image probe はデフォルトで無効化されます。
- live CLI-backend スモークは現在、Claude、Codex、Gemini に対して同じ end-to-end フローを実行します: text turn、image classification turn、その後 gateway CLI 経由で検証される MCP `cron` tool call。
- Claude のデフォルト smoke では、session を Sonnet から Opus に patch し、resume した session が以前のメモを引き続き覚えていることも検証します。

## Live: ACP bind スモーク（`/acp spawn ... --bind here`）

- テスト: `src/gateway/gateway-acp-bind.live.test.ts`
- 目的: live ACP agent を使って、実際の ACP conversation-bind フローを検証する:
  - `/acp spawn <agent> --bind here` を送信する
  - synthetic な message-channel conversation をその場で bind する
  - 同じ conversation 上で通常の follow-up を送信する
  - follow-up が bind 済み ACP session transcript に到達することを検証する
- 有効化:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- デフォルト:
  - Docker 内の ACP agent: `claude,codex,gemini`
  - 直接の `pnpm test:live ...` 用 ACP agent: `claude`
  - synthetic channel: Slack DM スタイルの conversation context
  - ACP backend: `acpx`
- 上書き:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- 注記:
  - このレーンは、admin 専用の synthetic な originating-route フィールド付きで Gateway の `chat.send` surface を使用するため、テストは外部配信を装わずに message-channel context を付与できます。
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` が未設定の場合、テストは選択された ACP harness agent に対して、埋め込み `acpx` Plugin の組み込み agent registry を使用します。

例:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker レシピ:

```bash
pnpm test:docker:live-acp-bind
```

単一 agent の Docker レシピ:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Docker 注記:

- Docker ランナーは `scripts/test-live-acp-bind-docker.sh` にあります。
- デフォルトでは、サポートされるすべての live CLI agent に対して ACP bind スモークを順番に実行します: `claude`、`codex`、`gemini`。
- マトリクスを絞り込むには `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`、または `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` を使用してください。
- これは `~/.profile` を source し、一致する CLI auth material をコンテナ内にステージし、書き込み可能な npm prefix に `acpx` をインストールし、その後要求された live CLI（`@anthropic-ai/claude-code`、`@openai/codex`、または `@google/gemini-cli`）がなければインストールします。
- Docker 内では、ランナーは `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` を設定するため、acpx は source 済み profile から利用可能な provider env vars を子 harness CLI に保持できます。

## Live: Codex app-server harness スモーク

- 目的: 通常の Gateway
  `agent` メソッドを通じて、Plugin 所有の Codex harness を検証する:
  - バンドルされた `codex` Plugin を読み込む
  - `OPENCLAW_AGENT_RUNTIME=codex` を選択する
  - `codex/gpt-5.4` に最初の Gateway agent turn を送信する
  - 同じ OpenClaw session に 2 回目の turn を送信し、app-server
    thread が resume できることを検証する
  - 同じ Gateway command
    パスを通じて `/codex status` と `/codex models` を実行する
- テスト: `src/gateway/gateway-codex-harness.live.test.ts`
- 有効化: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- デフォルト model: `codex/gpt-5.4`
- 任意の image probe: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- 任意の MCP/tool probe: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- このスモークは `OPENCLAW_AGENT_HARNESS_FALLBACK=none` を設定するため、壊れた Codex
  harness が PI へのサイレントフォールバックによって通過することはできません。
- 認証: シェル/profile からの `OPENAI_API_KEY` と、任意でコピーされる
  `~/.codex/auth.json` および `~/.codex/config.toml`

ローカルレシピ:

```bash
source ~/.profile
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=codex/gpt-5.4 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Docker レシピ:

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

Docker 注記:

- Docker ランナーは `scripts/test-live-codex-harness-docker.sh` にあります。
- これはマウントされた `~/.profile` を source し、`OPENAI_API_KEY` を渡し、存在する場合は Codex CLI
  auth ファイルをコピーし、書き込み可能なマウント済み npm
  prefix に `@openai/codex` をインストールし、source tree をステージした後、Codex-harness live テストのみを実行します。
- Docker では image および MCP/tool probe がデフォルトで有効です。より狭いデバッグ実行が必要な場合は
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` または
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` を設定してください。
- Docker でも `OPENCLAW_AGENT_HARNESS_FALLBACK=none` を export し、live
  テスト設定と一致させることで、`openai-codex/*` や PI へのフォールバックが Codex harness
  のリグレッションを隠せないようにします。

### 推奨される live レシピ

狭く明示的な allowlist が最も高速で、最も flaky になりにくいです。

- 単一 model、direct（Gateway なし）:
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- 単一 model、Gateway スモーク:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 複数プロバイダーにまたがる tool calling:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google フォーカス（Gemini API key + Antigravity）:
  - Gemini（API key）: `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）: `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

注記:

- `google/...` は Gemini API（API key）を使用します。
- `google-antigravity/...` は Antigravity OAuth bridge（Cloud Code Assist スタイルの agent endpoint）を使用します。
- `google-gemini-cli/...` はあなたのマシン上のローカル Gemini CLI を使用します（認証と tooling の癖は別です）。
- Gemini API と Gemini CLI の違い:
  - API: OpenClaw は Google がホストする Gemini API を HTTP 経由で呼び出します（API key / profile auth）。これはほとんどのユーザーが「Gemini」と言うときに意味しているものです。
  - CLI: OpenClaw はローカルの `gemini` バイナリをシェル実行します。これは独自の認証を持ち、挙動も異なる場合があります（streaming/tool サポート/バージョンずれ）。

## Live: model マトリクス（何をカバーするか）

固定の「CI model list」はありません（live はオプトイン）ですが、キーを持つ開発マシンで定期的にカバーすることが**推奨**されるモデルは次のとおりです。

### Modern スモークセット（tool calling + image）

これは、動作し続けることを期待する「一般的なモデル」実行です。

- OpenAI（非 Codex）: `openai/gpt-5.4`（任意: `openai/gpt-5.4-mini`）
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6`（または `anthropic/claude-sonnet-4-6`）
- Google（Gemini API）: `google/gemini-3.1-pro-preview` および `google/gemini-3-flash-preview`（古い Gemini 2.x モデルは避ける）
- Google（Antigravity）: `google-antigravity/claude-opus-4-6-thinking` および `google-antigravity/gemini-3-flash`
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

tools + image 付きで Gateway スモークを実行:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### ベースライン: tool calling（Read + 任意の Exec）

provider ファミリーごとに少なくとも 1 つは選んでください。

- OpenAI: `openai/gpt-5.4`（または `openai/gpt-5.4-mini`）
- Anthropic: `anthropic/claude-opus-4-6`（または `anthropic/claude-sonnet-4-6`）
- Google: `google/gemini-3-flash-preview`（または `google/gemini-3.1-pro-preview`）
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

任意の追加カバレッジ（あると望ましい）:

- xAI: `xai/grok-4`（または利用可能な最新）
- Mistral: `mistral/`…（有効化されている「tools」対応モデルを 1 つ選ぶ）
- Cerebras: `cerebras/`…（アクセス権がある場合）
- LM Studio: `lmstudio/`…（ローカル。tool calling は API mode に依存）

### Vision: image 送信（attachment → multimodal message）

image probe を実行するため、少なくとも 1 つは image 対応 model（Claude/Gemini/OpenAI の vision 対応 variant など）を `OPENCLAW_LIVE_GATEWAY_MODELS` に含めてください。

### Aggregator / 代替 Gateway

キーが有効なら、以下を通じたテストもサポートしています。

- OpenRouter: `openrouter/...`（何百ものモデル。tool+image 対応候補を見つけるには `openclaw models scan` を使用）
- OpenCode: Zen には `opencode/...`、Go には `opencode-go/...`（認証は `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`）

live マトリクスに含められる他の provider（認証情報/config がある場合）:

- Built-in: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- `models.providers` 経由（カスタム endpoint）: `minimax`（cloud/API）、および任意の OpenAI/Anthropic 互換 proxy（LM Studio、vLLM、LiteLLM など）

ヒント: ドキュメントに「すべてのモデル」をハードコードしようとしないでください。権威ある一覧は、あなたのマシン上で `discoverModels(...)` が返すものと、利用可能なキーがあるものです。

## 認証情報（絶対にコミットしない）

live テストは、CLI と同じ方法で認証情報を検出します。実際上の意味は次のとおりです。

- CLI が動作するなら、live テストも同じキーを見つけられるはずです。
- live テストが「認証情報なし」と言うなら、`openclaw models list` / model 選択をデバッグするときと同じ方法でデバッグしてください。

- agent ごとの認証 profile: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（live テストで「profile keys」と言うときはこれを指します）
- Config: `~/.openclaw/openclaw.json`（または `OPENCLAW_CONFIG_PATH`）
- legacy state dir: `~/.openclaw/credentials/`（存在する場合はステージされた live home にコピーされますが、メインの profile-key ストアではありません）
- ローカルの live 実行では、デフォルトでアクティブな config、agent ごとの `auth-profiles.json` ファイル、legacy `credentials/`、およびサポートされる外部 CLI 認証ディレクトリを一時テスト用ホームにコピーします。ステージされた live home では `workspace/` と `sandboxes/` をスキップし、`agents.*.workspace` / `agentDir` のパス上書きは除去されるため、probe が実際のホストワークスペースに触れません。

env key に依存したい場合（例: `~/.profile` で export されているもの）は、ローカルテストの前に `source ~/.profile` を実行するか、以下の Docker ランナーを使用してください（`~/.profile` をコンテナにマウントできます）。

## Deepgram live（音声文字起こし）

- テスト: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- 有効化: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- テスト: `src/agents/byteplus.live.test.ts`
- 有効化: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- 任意の model 上書き: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- テスト: `extensions/comfy/comfy.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- スコープ:
  - バンドルされた comfy の image、video、および `music_generate` パスを実行する
  - `models.providers.comfy.<capability>` が設定されていない限り、各 capability をスキップする
  - comfy workflow の送信、polling、download、または Plugin 登録を変更した後に有用

## Image generation live

- テスト: `src/image-generation/runtime.live.test.ts`
- コマンド: `pnpm test:live src/image-generation/runtime.live.test.ts`
- ハーネス: `pnpm test:live:media image`
- スコープ:
  - 登録されているすべての image-generation provider Plugin を列挙する
  - probing 前に、足りない provider env vars をログインシェル（`~/.profile`）から読み込む
  - デフォルトでは、保存済み auth profile よりも live/env API key を優先して使うため、`auth-profiles.json` 内の古いテスト用キーが実際のシェル認証情報を覆い隠さない
  - 使用可能な auth/profile/model がない provider はスキップする
  - 共有ランタイム capability を通じて、標準の image-generation variant を実行する:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- 現在カバーされている bundled provider:
  - `openai`
  - `google`
- 任意の絞り込み:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- 任意の認証挙動:
  - env-only の上書きを無視し、profile-store 認証を強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Music generation live

- テスト: `extensions/music-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- ハーネス: `pnpm test:live:media music`
- スコープ:
  - 共有 bundled music-generation provider パスを実行する
  - 現在は Google と MiniMax をカバーする
  - probing 前に、provider env vars をログインシェル（`~/.profile`）から読み込む
  - デフォルトでは、保存済み auth profile よりも live/env API key を優先して使うため、`auth-profiles.json` 内の古いテスト用キーが実際のシェル認証情報を覆い隠さない
  - 使用可能な auth/profile/model がない provider はスキップする
  - 利用可能な場合は、宣言された両方のランタイム mode を実行する:
    - prompt-only 入力による `generate`
    - provider が `capabilities.edit.enabled` を宣言している場合の `edit`
  - 現在の共有レーンでのカバレッジ:
    - `google`: `generate`、`edit`
    - `minimax`: `generate`
    - `comfy`: 別の Comfy live ファイルであり、この共有 sweep ではない
- 任意の絞り込み:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- 任意の認証挙動:
  - env-only の上書きを無視し、profile-store 認証を強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Video generation live

- テスト: `extensions/video-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- ハーネス: `pnpm test:live:media video`
- スコープ:
  - 共有 bundled video-generation provider パスを実行する
  - probing 前に、provider env vars をログインシェル（`~/.profile`）から読み込む
  - デフォルトでは、保存済み auth profile よりも live/env API key を優先して使うため、`auth-profiles.json` 内の古いテスト用キーが実際のシェル認証情報を覆い隠さない
  - 使用可能な auth/profile/model がない provider はスキップする
  - 利用可能な場合は、宣言された両方のランタイム mode を実行する:
    - prompt-only 入力による `generate`
    - provider が `capabilities.imageToVideo.enabled` を宣言し、かつ選択された provider/model が共有 sweep で buffer-backed のローカル image 入力を受け付ける場合の `imageToVideo`
    - provider が `capabilities.videoToVideo.enabled` を宣言し、かつ選択された provider/model が共有 sweep で buffer-backed のローカル video 入力を受け付ける場合の `videoToVideo`
  - 共有 sweep で現在宣言済みだがスキップされる `imageToVideo` provider:
    - `vydra`。bundled の `veo3` は text-only で、bundled の `kling` はリモート image URL を必要とするため
  - provider 固有の Vydra カバレッジ:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - このファイルは、デフォルトでリモート image URL fixture を使う `kling` レーンに加え、`veo3` の text-to-video を実行する
  - 現在の `videoToVideo` live カバレッジ:
    - 選択された model が `runway/gen4_aleph` の場合のみ `runway`
  - 共有 sweep で現在宣言済みだがスキップされる `videoToVideo` provider:
    - `alibaba`、`qwen`、`xai`。これらのパスは現在、リモート `http(s)` / MP4 参照 URL を必要とするため
    - `google`。現在の共有 Gemini/Veo レーンはローカルの buffer-backed 入力を使用しており、そのパスは共有 sweep では受け付けられないため
    - `openai`。現在の共有レーンには org 固有の video inpaint/remix アクセス保証がないため
- 任意の絞り込み:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- 任意の認証挙動:
  - env-only の上書きを無視し、profile-store 認証を強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Media live ハーネス

- コマンド: `pnpm test:live:media`
- 目的:
  - 共有の image、music、video live スイートを、リポジトリ標準の 1 つの entrypoint から実行する
  - `~/.profile` から足りない provider env vars を自動読み込みする
  - デフォルトで、現在使用可能な auth を持つ provider に各スイートを自動で絞り込む
  - `scripts/test-live.mjs` を再利用するため、heartbeat と quiet-mode の挙動が一貫する
- 例:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker ランナー（任意の「Linux でも動く」チェック）

これらの Docker ランナーは 2 つのカテゴリに分かれます:

- Live-model ランナー: `test:docker:live-models` と `test:docker:live-gateway` は、それぞれ対応する profile-key live ファイルのみをリポジトリ Docker イメージ内で実行します（`src/agents/models.profiles.live.test.ts` と `src/gateway/gateway-models.profiles.live.test.ts`）。ローカルの config dir と workspace をマウントし（マウントされていれば `~/.profile` も source します）。対応するローカル entrypoint は `test:live:models-profiles` と `test:live:gateway-profiles` です。
- Docker live ランナーは、フル Docker sweep を現実的に保つため、デフォルトでより小さいスモーク上限を使います:
  `test:docker:live-models` はデフォルトで `OPENCLAW_LIVE_MAX_MODELS=12`、
  `test:docker:live-gateway` はデフォルトで `OPENCLAW_LIVE_GATEWAY_SMOKE=1`、
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`、
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`、および
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` を使用します。より大きな網羅的スキャンを
  明示的に望む場合は、これらの env vars を上書きしてください。
- `test:docker:all` は、まず `test:docker:live-build` 経由で live Docker イメージを 1 回ビルドし、その後それを 2 つの live Docker レーンで再利用します。
- コンテナスモークランナー: `test:docker:openwebui`、`test:docker:onboard`、`test:docker:gateway-network`、`test:docker:mcp-channels`、および `test:docker:plugins` は、1 つ以上の実コンテナを起動し、より高レベルな integration パスを検証します。

live-model Docker ランナーはまた、必要な CLI 認証ホームだけを bind-mount し（実行が絞り込まれていない場合はサポートされるものをすべて）、実行前にそれらをコンテナ home にコピーします。これにより、外部 CLI OAuth がホストの認証ストアを変更せずにトークンを更新できます。

- Direct models: `pnpm test:docker:live-models`（スクリプト: `scripts/test-live-models-docker.sh`）
- ACP bind スモーク: `pnpm test:docker:live-acp-bind`（スクリプト: `scripts/test-live-acp-bind-docker.sh`）
- CLI backend スモーク: `pnpm test:docker:live-cli-backend`（スクリプト: `scripts/test-live-cli-backend-docker.sh`）
- Codex app-server harness スモーク: `pnpm test:docker:live-codex-harness`（スクリプト: `scripts/test-live-codex-harness-docker.sh`）
- Gateway + dev agent: `pnpm test:docker:live-gateway`（スクリプト: `scripts/test-live-gateway-models-docker.sh`）
- Open WebUI live スモーク: `pnpm test:docker:openwebui`（スクリプト: `scripts/e2e/openwebui-docker.sh`）
- オンボーディング ウィザード（TTY、完全な scaffolding）: `pnpm test:docker:onboard`（スクリプト: `scripts/e2e/onboard-docker.sh`）
- Gateway ネットワーキング（2 コンテナ、WS auth + health）: `pnpm test:docker:gateway-network`（スクリプト: `scripts/e2e/gateway-network-docker.sh`）
- MCP channel bridge（seed 済み Gateway + stdio bridge + 生の Claude notification-frame スモーク）: `pnpm test:docker:mcp-channels`（スクリプト: `scripts/e2e/mcp-channels-docker.sh`）
- Plugins（install スモーク + `/plugin` エイリアス + Claude bundle の再起動セマンティクス）: `pnpm test:docker:plugins`（スクリプト: `scripts/e2e/plugins-docker.sh`）

live-model Docker ランナーはまた、現在の checkout を読み取り専用で bind-mount し、
コンテナ内の一時 workdir にステージします。これにより、runtime
イメージをスリムに保ちつつ、正確にあなたのローカル source/config に対して Vitest を実行できます。
ステージング手順では、大きなローカル専用キャッシュやアプリ build 出力、
たとえば `.pnpm-store`、`.worktrees`、`__openclaw_vitest__`、およびアプリローカルの `.build` や
Gradle 出力ディレクトリなどをスキップするため、Docker live 実行が
マシン固有 artifact のコピーに何分も費やすことはありません。
また、これらは `OPENCLAW_SKIP_CHANNELS=1` を設定するため、
コンテナ内で Gateway live probe が実際の Telegram/Discord などの channel worker を起動しません。
`test:docker:live-models` は依然として `pnpm test:live` を実行するため、
その Docker レーンから Gateway
live カバレッジを絞り込むまたは除外したい場合は、`OPENCLAW_LIVE_GATEWAY_*` も渡してください。
`test:docker:openwebui` は、より高レベルな互換性スモークです。これは
OpenAI 互換 HTTP endpoint を有効にした OpenClaw Gateway コンテナを起動し、
その Gateway に向けて固定版の Open WebUI コンテナを起動し、Open WebUI 経由でサインインし、
`/api/models` が `openclaw/default` を公開していることを確認し、その後
Open WebUI の `/api/chat/completions` proxy を通じて実際の chat request を送信します。
初回実行は、Docker が
Open WebUI イメージを pull する必要があったり、Open WebUI 自身のコールドスタート設定が完了する必要があるため、目に見えて遅くなることがあります。
このレーンは、使用可能な live model key を想定しており、`OPENCLAW_PROFILE_FILE`
（デフォルトでは `~/.profile`）が Docker 化された実行でそれを提供する主な方法です。
成功した実行では `{ "ok": true, "model":
"openclaw/default", ... }` のような小さな JSON payload が出力されます。
`test:docker:mcp-channels` は意図的に決定論的であり、実際の
Telegram、Discord、または iMessage アカウントを必要としません。これは seed 済み Gateway
コンテナを起動し、`openclaw mcp serve` を起動する 2 つ目のコンテナを開始し、その後
ルーティングされた conversation discovery、transcript 読み取り、attachment metadata、
live event queue の挙動、outbound send routing、そして Claude スタイルの channel +
permission notification を、実際の stdio MCP bridge 上で検証します。notification チェックは
生の stdio MCP frame を直接検査するため、このスモークは、特定の client SDK がたまたま表面化するものではなく、
bridge が実際に出力する内容を検証します。

手動 ACP plain-language thread スモーク（CI ではない）:

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- このスクリプトはリグレッション/デバッグのワークフロー用に維持してください。ACP thread routing の検証のために再度必要になる可能性があるので、削除しないでください。

便利な env vars:

- `OPENCLAW_CONFIG_DIR=...`（デフォルト: `~/.openclaw`）は `/home/node/.openclaw` にマウントされます
- `OPENCLAW_WORKSPACE_DIR=...`（デフォルト: `~/.openclaw/workspace`）は `/home/node/.openclaw/workspace` にマウントされます
- `OPENCLAW_PROFILE_FILE=...`（デフォルト: `~/.profile`）は `/home/node/.profile` にマウントされ、テスト実行前に source されます
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）は Docker 内でキャッシュされた CLI install 用に `/home/node/.npm-global` にマウントされます
- `$HOME` 配下の外部 CLI 認証 dir/file は、`/host-auth...` 配下に読み取り専用でマウントされ、テスト開始前に `/home/node/...` へコピーされます
  - デフォルト dir: `.minimax`
  - デフォルト file: `~/.codex/auth.json`、`~/.codex/config.toml`、`.claude.json`、`~/.claude/.credentials.json`、`~/.claude/settings.json`、`~/.claude/settings.local.json`
  - 絞り込まれた provider 実行では、`OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` から推測された必要な dir/file のみをマウントします
  - 手動で上書きするには `OPENCLAW_DOCKER_AUTH_DIRS=all`、`OPENCLAW_DOCKER_AUTH_DIRS=none`、または `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` のようなカンマ区切りリストを使用してください
- 実行を絞り込むには `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- コンテナ内の provider をフィルタするには `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- 再ビルド不要の再実行で既存の `openclaw:local-live` イメージを再利用するには `OPENCLAW_SKIP_DOCKER_BUILD=1`
- 認証情報が env ではなく profile store 由来であることを保証するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`
- Open WebUI スモーク用に Gateway が公開する model を選ぶには `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI スモークで使用する nonce-check prompt を上書きするには `OPENCLAW_OPENWEBUI_PROMPT=...`
- 固定された Open WebUI image tag を上書きするには `OPENWEBUI_IMAGE=...`

## ドキュメントの健全性確認

ドキュメント編集後は docs チェックを実行してください: `pnpm check:docs`。
インページ見出しチェックも必要な場合は、完全な Mintlify アンカー検証を実行してください: `pnpm docs:check-links:anchors`。

## オフラインリグレッション（CI 安全）

これらは、実 provider なしでの「実際のパイプライン」リグレッションです。

- Gateway tool calling（mock OpenAI、実際の Gateway + agent ループ）: `src/gateway/gateway.test.ts`（ケース: "runs a mock OpenAI tool call end-to-end via gateway agent loop"）
- Gateway ウィザード（WS `wizard.start`/`wizard.next`、config + auth enforced の書き込み）: `src/gateway/gateway.test.ts`（ケース: "runs wizard over ws and writes auth token config"）

## agent 信頼性 evals（Skills）

すでに、次のような「agent 信頼性 evals」のように振る舞う CI 安全なテストがいくつかあります。

- 実際の Gateway + agent ループを通る mock tool-calling（`src/gateway/gateway.test.ts`）。
- session wiring と config 効果を検証する end-to-end の ウィザード フロー（`src/gateway/gateway.test.ts`）。

Skills 向けにまだ不足しているもの（[Skills](/ja-JP/tools/skills) を参照）:

- **Decisioning:** prompt に Skills が列挙されているとき、agent は正しい skill を選ぶか（または無関係なものを避けるか）？
- **Compliance:** agent は使用前に `SKILL.md` を読み、必要な手順/引数に従うか？
- **Workflow contracts:** tool の順序、session history の持ち越し、sandbox 境界を検証する multi-turn シナリオ。

将来の evals は、まず決定論的であるべきです。

- tool call + 順序、skill file の読み取り、session wiring を検証するために mock provider を使う scenario runner。
- skill に焦点を当てた小規模スイート（使う vs 使わない、gating、prompt injection）。
- オプトインで env-gated な live evals は、CI 安全なスイートが整ってからにする。

## Contract テスト（Plugin と channel の形状）

Contract テストは、登録されているすべての Plugin と channel がその
interface contract に準拠していることを検証します。これらは検出されたすべての Plugin を反復し、
形状と挙動に関する一連のアサーションを実行します。デフォルトの `pnpm test` unit レーンは、
これらの共有 seam および smoke ファイルを意図的にスキップします。共有 channel または provider の surface に触れた場合は、
contract コマンドを明示的に実行してください。

### コマンド

- すべての contract: `pnpm test:contracts`
- channel contract のみ: `pnpm test:contracts:channels`
- provider contract のみ: `pnpm test:contracts:plugins`

### Channel contract

`src/channels/plugins/contracts/*.contract.test.ts` にあります:

- **plugin** - 基本的な Plugin 形状（id、name、capabilities）
- **setup** - セットアップ ウィザード 契約
- **session-binding** - session binding の挙動
- **outbound-payload** - message payload 構造
- **inbound** - inbound message 処理
- **actions** - channel action handler
- **threading** - thread ID 処理
- **directory** - directory/roster API
- **group-policy** - group policy 強制

### Provider status contract

`src/plugins/contracts/*.contract.test.ts` にあります。

- **status** - channel status probe
- **registry** - Plugin registry 形状

### Provider contract

`src/plugins/contracts/*.contract.test.ts` にあります:

- **auth** - 認証フロー契約
- **auth-choice** - 認証選択
- **catalog** - model catalog API
- **discovery** - Plugin discovery
- **loader** - Plugin loading
- **runtime** - provider runtime
- **shape** - Plugin 形状/interface
- **wizard** - セットアップ ウィザード

### 実行するタイミング

- plugin-sdk の export または subpath を変更した後
- channel または provider Plugin を追加または変更した後
- Plugin 登録または discovery をリファクタした後

Contract テストは CI で実行され、実際の API key は不要です。

## リグレッションを追加する（ガイダンス）

live で見つかった provider/model の問題を修正するとき:

- 可能なら CI 安全なリグレッションを追加する（mock/stub provider、または正確な request-shape 変換をキャプチャする）
- 本質的に live 専用なら（rate limit、認証ポリシーなど）、live テストは狭く保ち、env vars 経由のオプトインにする
- バグを捕まえられる最小のレイヤーを狙うことを優先する:
  - provider の request conversion/replay バグ → direct models テスト
  - Gateway の session/history/tool パイプラインバグ → Gateway live スモーク、または CI 安全な Gateway mock テスト
- SecretRef トラバーサルのガードレール:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` は、registry metadata（`listSecretTargetRegistryEntries()`）から SecretRef クラスごとに 1 つのサンプル target を導出し、トラバーサルセグメントの exec id が拒否されることを検証します。
  - `src/secrets/target-registry-data.ts` に新しい `includeInPlan` SecretRef target ファミリーを追加する場合は、そのテスト内の `classifyTargetClass` を更新してください。このテストは未分類の target id に対して意図的に失敗するため、新しいクラスが黙ってスキップされることを防ぎます。
