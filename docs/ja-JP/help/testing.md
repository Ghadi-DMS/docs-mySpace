---
read_when:
    - ローカルまたは CI でテストを実行する
    - モデル/プロバイダーのバグに対するリグレッションを追加する
    - Gateway + エージェントの動作をデバッグする
summary: 'テストキット: unit/e2e/live スイート、Docker ランナー、および各テストでカバーされる内容'
title: テスト
x-i18n:
    generated_at: "2026-04-15T14:40:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec3632cafa1f38b27510372391b84af744266df96c58f7fac98aa03763465db8
    source_path: help/testing.md
    workflow: 15
---

# テスト

OpenClaw には 3 つの Vitest スイート（unit/integration、e2e、live）と、少数の Docker ランナーがあります。

このドキュメントは「どのようにテストするか」のガイドです。

- 各スイートが何をカバーするか（そして意図的に何を _カバーしない_ か）
- 一般的なワークフロー（ローカル、プッシュ前、デバッグ）で実行するコマンド
- live テストがどのように認証情報を検出し、モデル/プロバイダーを選択するか
- 実際のモデル/プロバイダーの問題に対するリグレッションをどのように追加するか

## クイックスタート

普段は次のとおりです。

- フルゲート（プッシュ前に期待されるもの）: `pnpm build && pnpm check && pnpm test`
- 余裕のあるマシンでの高速なローカル全スイート実行: `pnpm test:max`
- 直接の Vitest ウォッチループ: `pnpm test:watch`
- 直接のファイル指定は拡張機能/チャネルのパスにも対応するようになりました: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- 単一の失敗を反復的に修正している場合は、まず対象を絞った実行を優先してください。
- Docker ベースの QA サイト: `pnpm qa:lab:up`
- Linux VM ベースの QA レーン: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

テストに触れたときや、より高い確信が欲しいとき:

- カバレッジゲート: `pnpm test:coverage`
- E2E スイート: `pnpm test:e2e`

実際のプロバイダー/モデルをデバッグするとき（実際の認証情報が必要）:

- live スイート（モデル + Gateway のツール/画像プローブ）: `pnpm test:live`
- 1 つの live ファイルを静かに対象指定: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

ヒント: 失敗している 1 ケースだけが必要な場合は、以下で説明する allowlist 環境変数を使って live テストを絞り込むのを優先してください。

## QA 固有のランナー

これらのコマンドは、QA-lab の現実性が必要なときにメインのテストスイートの横にあります。

- `pnpm openclaw qa suite`
  - リポジトリベースの QA シナリオをホスト上で直接実行します。
  - デフォルトでは、分離された Gateway ワーカーを使って、選択された複数のシナリオを並列実行します。最大 64 ワーカー、または選択されたシナリオ数までです。ワーカー数を調整するには `--concurrency <count>` を使い、従来の直列レーンにするには `--concurrency 1` を使います。
- `pnpm openclaw qa suite --runner multipass`
  - 同じ QA スイートを使い捨ての Multipass Linux VM 内で実行します。
  - ホスト上の `qa suite` と同じシナリオ選択動作を維持します。
  - `qa suite` と同じプロバイダー/モデル選択フラグを再利用します。
  - live 実行では、ゲストで実用的な、サポートされている QA 認証入力を転送します:
    環境変数ベースのプロバイダーキー、QA live プロバイダー設定パス、および存在する場合の `CODEX_HOME`。
  - ゲストがマウントされたワークスペース経由で書き戻せるように、出力ディレクトリはリポジトリルート配下に置く必要があります。
  - 通常の QA レポート + サマリーに加えて、Multipass ログを `.artifacts/qa-e2e/...` 配下に書き込みます。
- `pnpm qa:lab:up`
  - オペレーター形式の QA 作業向けに、Docker ベースの QA サイトを起動します。
- `pnpm openclaw qa matrix`
  - 使い捨ての Docker ベース Tuwunel homeserver に対して、Matrix live QA レーンを実行します。
  - この QA ホストは現在、repo/dev 専用です。パッケージ化された OpenClaw インストールには `qa-lab` が含まれないため、`openclaw qa` は公開されません。
  - リポジトリのチェックアウトは、バンドルされたランナーを直接読み込みます。別個の Plugin インストール手順は不要です。
  - 一時的な Matrix ユーザーを 3 つ（`driver`、`sut`、`observer`）と 1 つのプライベートルームをプロビジョニングし、その後、実際の Matrix Plugin を SUT トランスポートとして使う QA Gateway 子プロセスを起動します。
  - デフォルトでは固定された安定版の Tuwunel イメージ `ghcr.io/matrix-construct/tuwunel:v1.5.1` を使います。別のイメージをテストする必要がある場合は `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` で上書きしてください。
  - Matrix はローカルで使い捨てユーザーをプロビジョニングするため、共有認証情報ソースフラグは公開していません。
  - Matrix QA レポート、サマリー、および観測イベントのアーティファクトを `.artifacts/qa-e2e/...` 配下に書き込みます。
- `pnpm openclaw qa telegram`
  - 環境変数から取得した driver と SUT のボットトークンを使って、実際のプライベートグループに対して Telegram live QA レーンを実行します。
  - `OPENCLAW_QA_TELEGRAM_GROUP_ID`、`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` が必要です。グループ id は数値の Telegram chat id である必要があります。
  - 共有プール認証情報には `--credential-source convex` をサポートします。デフォルトでは env モードを使うか、プールされたリースを利用するには `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` を設定してください。
  - 同じプライベートグループ内にある 2 つの別個のボットが必要で、SUT ボットは Telegram ユーザー名を公開している必要があります。
  - 安定した bot-to-bot 観測のために、両方のボットについて `@BotFather` で Bot-to-Bot Communication Mode を有効にし、driver ボットがグループ内の bot トラフィックを観測できることを確認してください。
  - Telegram QA レポート、サマリー、および観測メッセージのアーティファクトを `.artifacts/qa-e2e/...` 配下に書き込みます。

live トランスポートレーンは、標準契約を 1 つ共有しており、新しいトランスポートが逸脱しないようにしています。

`qa-channel` は依然として広範な synthetic QA スイートであり、live
トランスポートのカバレッジマトリクスには含まれません。

| Lane     | Canary | Mention gating | Allowlist block | Top-level reply | Restart resume | Thread follow-up | Thread isolation | Reaction observation | Help command |
| -------- | ------ | -------------- | --------------- | --------------- | -------------- | ---------------- | ---------------- | -------------------- | ------------ |
| Matrix   | x      | x              | x               | x               | x              | x                | x                | x                    |              |
| Telegram | x      |                |                 |                 |                |                  |                  |                      | x            |

### Convex による共有 Telegram 認証情報（v1）

`openclaw qa telegram` に対して `--credential-source convex`（または `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`）が有効な場合、
QA lab は Convex ベースのプールから排他的なリースを取得し、
レーンの実行中はそのリースに Heartbeat を送り、
シャットダウン時にそのリースを解放します。

参照用 Convex プロジェクトのスキャフォールド:

- `qa/convex-credential-broker/`

必要な環境変数:

- `OPENCLAW_QA_CONVEX_SITE_URL`（例: `https://your-deployment.convex.site`）
- 選択されたロールに対応する 1 つのシークレット:
  - `maintainer` には `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
  - `ci` には `OPENCLAW_QA_CONVEX_SECRET_CI`
- 認証情報ロールの選択:
  - CLI: `--credential-role maintainer|ci`
  - 環境変数のデフォルト: `OPENCLAW_QA_CREDENTIAL_ROLE`（デフォルトは `maintainer`）

任意の環境変数:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS`（デフォルト `1200000`）
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS`（デフォルト `30000`）
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS`（デフォルト `90000`）
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS`（デフォルト `15000`）
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`（デフォルト `/qa-credentials/v1`）
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID`（任意のトレース id）
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` は、ローカル専用開発向けに loopback `http://` Convex URL を許可します。

通常運用では、`OPENCLAW_QA_CONVEX_SITE_URL` は `https://` を使う必要があります。

メンテナー向け管理コマンド（プールの追加/削除/一覧表示）には、
特に `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` が必要です。

メンテナー向け CLI ヘルパー:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

スクリプトや CI ユーティリティで機械可読な出力が必要な場合は `--json` を使ってください。

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
  - アクティブリースのガード: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list`（maintainer secret のみ）
  - リクエスト: `{ kind?, status?, includePayload?, limit? }`
  - 成功: `{ status: "ok", credentials, count }`

Telegram kind の payload 形状:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` は数値の Telegram chat id 文字列である必要があります。
- `admin/add` は `kind: "telegram"` に対してこの形状を検証し、不正な payload を拒否します。

### QA にチャネルを追加する

Markdown QA システムにチャネルを追加するには、必要なものは正確に 2 つです。

1. そのチャネル用のトランスポートアダプター。
2. チャネル契約を実行する scenario pack。

共有 `qa-lab` ホストがフローを所有できる場合は、新しいトップレベル QA コマンド root を追加しないでください。

`qa-lab` は共有ホストの仕組みを所有します。

- `openclaw qa` コマンド root
- スイートの起動と終了処理
- ワーカー並行実行
- アーティファクト書き込み
- レポート生成
- シナリオ実行
- 古い `qa-channel` シナリオ向けの互換エイリアス

ランナー Plugin はトランスポート契約を所有します。

- `openclaw qa <runner>` が共有 `qa` root 配下にどのようにマウントされるか
- そのトランスポート向けに Gateway がどのように設定されるか
- readiness がどのようにチェックされるか
- inbound イベントがどのように注入されるか
- outbound メッセージがどのように観測されるか
- transcript と正規化されたトランスポート状態がどのように公開されるか
- トランスポートを使うアクションがどのように実行されるか
- トランスポート固有のリセットやクリーンアップがどのように処理されるか

新しいチャネルの最小導入基準は次のとおりです。

1. 共有 `qa` root の所有者は `qa-lab` のままにする。
2. 共有 `qa-lab` ホストの seam 上にトランスポートランナーを実装する。
3. トランスポート固有の仕組みはランナー Plugin またはチャネル harness 内に留める。
4. 競合する root コマンドを登録するのではなく、ランナーを `openclaw qa <runner>` としてマウントする。
   ランナー Plugin は `openclaw.plugin.json` で `qaRunners` を宣言し、`runtime-api.ts` から対応する `qaRunnerCliRegistrations` 配列をエクスポートする必要があります。
   `runtime-api.ts` は軽量に保ち、遅延 CLI およびランナー実行は別個のエントリーポイントの背後に置いてください。
5. `qa/scenarios/` 配下で Markdown シナリオを作成または調整する。
6. 新しいシナリオには汎用シナリオヘルパーを使う。
7. リポジトリが意図的な移行を行っている場合を除き、既存の互換エイリアスを動作させ続ける。

判断ルールは厳格です。

- 振る舞いを `qa-lab` で一度だけ表現できるなら、`qa-lab` に置いてください。
- 振る舞いが 1 つのチャネルトランスポートに依存するなら、そのランナー Plugin または Plugin harness に留めてください。
- シナリオが複数のチャネルで使える新しい機能を必要とするなら、`suite.ts` にチャネル固有の分岐を追加するのではなく、汎用ヘルパーを追加してください。
- 振る舞いが 1 つのトランスポートにしか意味を持たないなら、そのシナリオはトランスポート固有のままにし、それをシナリオ契約で明示してください。

新しいシナリオに推奨される汎用ヘルパー名は次のとおりです。

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

既存シナリオ向けの互換エイリアスも引き続き利用できます。これには次が含まれます。

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

新しいチャネル作業では、汎用ヘルパー名を使う必要があります。
互換エイリアスはフラグデー移行を避けるために存在しており、
新しいシナリオ作成のモデルではありません。

## テストスイート（どこで何が実行されるか）

スイートは「現実性が増すもの」（そして不安定さ/コストも増すもの）として考えてください。

### Unit / integration（デフォルト）

- コマンド: `pnpm test`
- 設定: 既存のスコープ付き Vitest project に対する 10 個の逐次 shard 実行（`vitest.full-*.config.ts`）
- ファイル: `src/**/*.test.ts`、`packages/**/*.test.ts`、`test/**/*.test.ts` 配下の core/unit インベントリと、`vitest.unit.config.ts` でカバーされる許可済みの `ui` node テスト
- スコープ:
  - 純粋な unit テスト
  - プロセス内 integration テスト（gateway auth、routing、tooling、parsing、config）
  - 既知のバグに対する決定的なリグレッション
- 想定:
  - CI で実行される
  - 実際のキーは不要
  - 高速で安定しているべき
- Projects に関する注記:
  - 対象指定なしの `pnpm test` は、1 つの巨大なネイティブルート project プロセスではなく、11 個のより小さな shard 設定（`core-unit-src`、`core-unit-security`、`core-unit-ui`、`core-unit-support`、`core-support-boundary`、`core-contracts`、`core-bundled`、`core-runtime`、`agentic`、`auto-reply`、`extensions`）を実行するようになりました。これにより、負荷の高いマシンでのピーク RSS が削減され、auto-reply/extension の処理が無関係なスイートを圧迫するのを防ぎます。
  - `pnpm test --watch` は引き続きネイティブルートの `vitest.config.ts` project graph を使います。複数 shard の watch ループは現実的ではないためです。
  - `pnpm test`、`pnpm test:watch`、`pnpm test:perf:imports` は、明示的なファイル/ディレクトリ指定を最初にスコープ付きレーン経由でルーティングするため、`pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` は完全なルート project 起動コストを支払わずに済みます。
  - `pnpm test:changed` は、差分がルーティング可能なソース/テストファイルだけに触れている場合、変更された git パスを同じスコープ付きレーンに展開します。config/setup の編集は、引き続き広範なルート project の再実行にフォールバックします。
  - agents、commands、plugins、auto-reply ヘルパー、`plugin-sdk`、および同様の純粋なユーティリティ領域からの import-light な unit テストは、`test/setup-openclaw-runtime.ts` をスキップする `unit-fast` レーンを通ります。状態を持つファイルや runtime の重いファイルは既存のレーンに残ります。
  - 一部の `plugin-sdk` および `commands` ヘルパーのソースファイルは、変更モード実行をそれらの light レーンにある明示的な隣接テストにもマップするようになったため、ヘルパー編集時にそのディレクトリ全体の重いスイートを再実行せずに済みます。
  - `auto-reply` は現在、3 つの専用バケットを持ちます: トップレベルの core ヘルパー、トップレベルの `reply.*` integration テスト、そして `src/auto-reply/reply/**` サブツリーです。これにより、最も重い reply harness 処理が、軽量な status/chunk/token テストに載らないようにしています。
- Embedded runner に関する注記:
  - メッセージツール検出入力または Compaction runtime コンテキストを変更する場合は、
    両レベルのカバレッジを維持してください。
  - 純粋な routing/normalization 境界には、焦点を絞ったヘルパーリグレッションを追加してください。
  - さらに、embedded runner integration スイートも健全に保ってください:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`、
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`、および
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`。
  - これらのスイートは、scoped id と Compaction の振る舞いが
    実際の `run.ts` / `compact.ts` パスを通って流れ続けることを検証します。ヘルパーのみのテストは、
    これらの integration パスの十分な代替にはなりません。
- Pool に関する注記:
  - ベース Vitest config は現在デフォルトで `threads` を使います。
  - 共有 Vitest config は `isolate: false` も固定しており、ルート projects、e2e、live config 全体で非分離 runner を使用します。
  - ルート UI レーンは `jsdom` の setup と optimizer を維持しますが、現在は共有の非分離 runner 上でも動作します。
  - 各 `pnpm test` shard は、共有 Vitest config から同じ `threads` + `isolate: false` のデフォルトを継承します。
  - 共有の `scripts/run-vitest.mjs` ランチャーは、大規模なローカル実行中の V8 コンパイルの揺れを減らすため、Vitest 子 Node プロセスに対してデフォルトで `--no-maglev` も追加するようになりました。標準の V8 挙動と比較したい場合は `OPENCLAW_VITEST_ENABLE_MAGLEV=1` を設定してください。
- 高速なローカル反復に関する注記:
  - `pnpm test:changed` は、変更されたパスがより小さなスイートにきれいにマップされる場合、スコープ付きレーンを通ります。
  - `pnpm test:max` と `pnpm test:changed:max` は、同じルーティング動作を維持しつつ、ワーカー上限だけを高くします。
  - ローカルのワーカー自動スケーリングは現在、意図的に保守的であり、ホストの load average がすでに高い場合にも抑制されるため、複数の同時 Vitest 実行がデフォルトで与える悪影響が小さくなります。
  - ベース Vitest config は、テスト配線が変わったときに changed-mode の再実行が正しく保たれるよう、projects/config ファイルを `forceRerunTriggers` としてマークします。
  - config は、対応ホスト上では `OPENCLAW_VITEST_FS_MODULE_CACHE` を有効に保ちます。直接プロファイリングのために明示的なキャッシュ場所を 1 つ指定したい場合は `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` を設定してください。
- パフォーマンスデバッグに関する注記:
  - `pnpm test:perf:imports` は Vitest の import-duration レポートと import-breakdown 出力を有効にします。
  - `pnpm test:perf:imports:changed` は、同じプロファイリング表示を `origin/main` 以降に変更されたファイルにスコープします。
- `pnpm test:perf:changed:bench -- --ref <git-ref>` は、そのコミット済み差分に対してルーティングされた `test:changed` をネイティブルート project パスと比較し、wall time と macOS の max RSS を出力します。
- `pnpm test:perf:changed:bench -- --worktree` は、変更されたファイル一覧を `scripts/test-projects.mjs` とルート Vitest config に通すことで、現在の dirty tree をベンチマークします。
  - `pnpm test:perf:profile:main` は、Vitest/Vite の起動および transform オーバーヘッドに対するメインスレッド CPU プロファイルを書き出します。
  - `pnpm test:perf:profile:runner` は、ファイル並列化を無効にした unit スイートに対する runner の CPU+heap プロファイルを書き出します。

### E2E（Gateway スモーク）

- コマンド: `pnpm test:e2e`
- 設定: `vitest.e2e.config.ts`
- ファイル: `src/**/*.e2e.test.ts`、`test/**/*.e2e.test.ts`
- デフォルトの runtime:
  - リポジトリの他の部分と同様に、Vitest の `threads` と `isolate: false` を使います。
  - 適応型ワーカーを使います（CI: 最大 2、ローカル: デフォルトで 1）。
  - コンソール I/O オーバーヘッドを減らすため、デフォルトでは silent モードで実行します。
- 便利な上書き:
  - ワーカー数を強制するには `OPENCLAW_E2E_WORKERS=<n>`（上限は 16）。
  - 詳細なコンソール出力を再有効化するには `OPENCLAW_E2E_VERBOSE=1`。
- スコープ:
  - 複数インスタンスの Gateway の end-to-end 動作
  - WebSocket/HTTP サーフェス、Node ペアリング、およびより重いネットワーク処理
- 想定:
  - （パイプラインで有効な場合は）CI で実行される
  - 実際のキーは不要
  - unit テストより可動部分が多い（遅くなることがある）

### E2E: OpenShell バックエンドスモーク

- コマンド: `pnpm test:e2e:openshell`
- ファイル: `test/openshell-sandbox.e2e.test.ts`
- スコープ:
  - Docker 経由で、ホスト上に分離された OpenShell Gateway を起動する
  - 一時的なローカル Dockerfile から sandbox を作成する
  - 実際の `sandbox ssh-config` + SSH exec を介して OpenClaw の OpenShell バックエンドを実行する
  - sandbox fs bridge を通じて、リモート正準の filesystem 振る舞いを検証する
- 想定:
  - オプトインのみ。デフォルトの `pnpm test:e2e` 実行には含まれない
  - ローカルの `openshell` CLI と動作する Docker daemon が必要
  - 分離された `HOME` / `XDG_CONFIG_HOME` を使い、その後テスト Gateway と sandbox を破棄する
- 便利な上書き:
  - より広い e2e スイートを手動実行する際にこのテストを有効化するには `OPENCLAW_E2E_OPENSHELL=1`
  - デフォルト以外の CLI バイナリまたはラッパースクリプトを指定するには `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`

### Live（実際のプロバイダー + 実際のモデル）

- コマンド: `pnpm test:live`
- 設定: `vitest.live.config.ts`
- ファイル: `src/**/*.live.test.ts`
- デフォルト: `pnpm test:live` によって **有効**（`OPENCLAW_LIVE_TEST=1` を設定）
- スコープ:
  - 「このプロバイダー/モデルは _今日_ 実際の認証情報で本当に動作するか？」
  - プロバイダーのフォーマット変更、tool-calling の癖、認証の問題、rate limit の挙動を検出する
- 想定:
  - 設計上、CI 安定ではない（実ネットワーク、実際のプロバイダーポリシー、クォータ、障害）
  - コストがかかる / rate limit を消費する
  - 「全部」ではなく、対象を絞ったサブセットの実行を優先する
- live 実行は不足している API キーを取得するために `~/.profile` を読み込みます。
- デフォルトでは、live 実行は引き続き `HOME` を分離し、config/auth マテリアルを一時的なテスト home にコピーするため、unit fixture が実際の `~/.openclaw` を変更することはありません。
- live テストで実際の home ディレクトリを使う必要がある場合にのみ、`OPENCLAW_LIVE_USE_REAL_HOME=1` を設定してください。
- `pnpm test:live` は現在、より静かなモードがデフォルトです: `[live] ...` の進捗出力は維持しますが、追加の `~/.profile` 通知を抑制し、Gateway のブートストラップログ/Bonjour の雑音をミュートします。完全な起動ログを再び見たい場合は `OPENCLAW_LIVE_TEST_QUIET=0` を設定してください。
- API キーローテーション（プロバイダー固有）: カンマ/セミコロン形式の `*_API_KEYS`、または `*_API_KEY_1`、`*_API_KEY_2`（例: `OPENAI_API_KEYS`、`ANTHROPIC_API_KEYS`、`GEMINI_API_KEYS`）、もしくは live ごとの上書き `OPENCLAW_LIVE_*_KEY` を設定します。テストは rate limit 応答時に再試行します。
- 進捗/Heartbeat 出力:
  - live スイートは現在、進捗行を stderr に出力するため、Vitest のコンソールキャプチャが静かでも、長いプロバイダー呼び出しが視覚的にアクティブであることがわかります。
  - `vitest.live.config.ts` は Vitest のコンソールインターセプトを無効にするため、プロバイダー/Gateway の進捗行は live 実行中に即座にストリームされます。
  - 直接モデルの Heartbeat を調整するには `OPENCLAW_LIVE_HEARTBEAT_MS` を使います。
  - Gateway/プローブの Heartbeat を調整するには `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` を使います。

## どのスイートを実行すべきか？

次の判断表を使ってください。

- ロジック/テストを編集している: `pnpm test` を実行（大きく変更した場合は `pnpm test:coverage` も）
- Gateway ネットワーク / WS protocol / pairing に触れている: `pnpm test:e2e` を追加
- 「自分の bot がダウンしている」/ プロバイダー固有の障害 / tool calling をデバッグしている: 対象を絞った `pnpm test:live` を実行

## Live: Android Node capability sweep

- テスト: `src/gateway/android-node.capabilities.live.test.ts`
- スクリプト: `pnpm android:test:integration`
- 目的: 接続された Android Node が現在広告している **すべてのコマンド** を呼び出し、コマンド契約の振る舞いを検証する。
- スコープ:
  - 前提条件付き/手動セットアップ（このスイートはアプリをインストール/実行/ペアリングしません）。
  - 選択された Android Node に対する、コマンドごとの Gateway `node.invoke` 検証。
- 必要な事前セットアップ:
  - Android アプリがすでに接続済みかつ Gateway とペアリング済みであること。
  - アプリがフォアグラウンドに保たれていること。
  - 成功を期待する capability に対して、権限/キャプチャ同意が付与されていること。
- 任意のターゲット上書き:
  - `OPENCLAW_ANDROID_NODE_ID` または `OPENCLAW_ANDROID_NODE_NAME`。
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`。
- Android の完全なセットアップ詳細: [Android App](/ja-JP/platforms/android)

## Live: model smoke（profile keys）

live テストは、障害を切り分けられるように 2 つのレイヤーに分かれています。

- 「直接モデル」は、そのキーでプロバイダー/モデルが少なくとも応答できるかを示します。
- 「Gateway スモーク」は、そのモデルに対して Gateway+agent パイプライン全体（セッション、履歴、ツール、sandbox ポリシーなど）が動作することを示します。

### レイヤー 1: 直接モデル completion（Gateway なし）

- テスト: `src/agents/models.profiles.live.test.ts`
- 目的:
  - 検出されたモデルを列挙する
  - `getApiKeyForModel` を使って、認証情報を持つモデルを選択する
  - モデルごとに小さな completion を実行する（必要に応じて対象を絞ったリグレッションも）
- 有効化方法:
  - `pnpm test:live`（または Vitest を直接起動する場合は `OPENCLAW_LIVE_TEST=1`）
- このスイートを実際に実行するには `OPENCLAW_LIVE_MODELS=modern`（または `all`、modern のエイリアス）を設定します。そうしない場合は、`pnpm test:live` の焦点を Gateway スモークに保つためスキップされます
- モデルの選択方法:
  - modern allowlist を実行するには `OPENCLAW_LIVE_MODELS=modern`（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_MODELS=all` は modern allowlist のエイリアスです
  - または `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."`（カンマ区切りの allowlist）
  - modern/all sweep はデフォルトで厳選された高シグナルの上限を使います。網羅的な modern sweep にするには `OPENCLAW_LIVE_MAX_MODELS=0` を設定し、より小さな上限にするには正の数を設定してください。
- プロバイダーの選択方法:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（カンマ区切りの allowlist）
- キーの取得元:
  - デフォルト: profile store と env フォールバック
  - **profile store** のみを強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` を設定
- これが存在する理由:
  - 「プロバイダー API が壊れている / キーが無効」と「Gateway agent パイプラインが壊れている」を切り分ける
  - 小さく分離されたリグレッションを収容する（例: OpenAI Responses/Codex Responses の reasoning 再生 + tool-call フロー）

### レイヤー 2: Gateway + dev agent スモーク（`@openclaw` が実際に行うこと）

- テスト: `src/gateway/gateway-models.profiles.live.test.ts`
- 目的:
  - プロセス内 Gateway を起動する
  - `agent:dev:*` セッションを作成/patch する（実行ごとに model override）
  - キーを持つモデルを反復し、次を検証する:
    - 「意味のある」応答（ツールなし）
    - 実際のツール呼び出しが動作する（read プローブ）
    - 任意の追加ツールプローブ（exec+read プローブ）
    - OpenAI のリグレッションパス（tool-call-only → follow-up）が引き続き動作する
- プローブの詳細（障害をすばやく説明できるように）:
  - `read` プローブ: テストはワークスペースに nonce ファイルを書き込み、エージェントにそれを `read` して nonce をそのまま返すよう依頼します。
  - `exec+read` プローブ: テストはエージェントに、一時ファイルへ nonce を `exec` で書き込んでから、それを `read` で読み戻すよう依頼します。
  - image プローブ: テストは生成した PNG（cat + ランダム化されたコード）を添付し、モデルが `cat <CODE>` を返すことを期待します。
  - 実装参照: `src/gateway/gateway-models.profiles.live.test.ts` および `src/gateway/live-image-probe.ts`。
- 有効化方法:
  - `pnpm test:live`（または Vitest を直接起動する場合は `OPENCLAW_LIVE_TEST=1`）
- モデルの選択方法:
  - デフォルト: modern allowlist（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` は modern allowlist のエイリアスです
  - または、絞り込むために `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`（またはカンマ区切りの一覧）を設定します
  - modern/all の Gateway sweep はデフォルトで厳選された高シグナルの上限を使います。網羅的な modern sweep にするには `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`、より小さな上限にするには正の数を設定してください。
- プロバイダーの選択方法（「OpenRouter 全部」を避ける）:
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（カンマ区切りの allowlist）
- この live テストでは、ツール + image プローブは常に有効です:
  - `read` プローブ + `exec+read` プローブ（ツール負荷テスト）
  - モデルが image input サポートを広告している場合、image プローブが実行されます
  - フロー（概要）:
    - テストは「CAT」+ ランダムコードを含む小さな PNG を生成します（`src/gateway/live-image-probe.ts`）
    - それを `agent` の `attachments: [{ mimeType: "image/png", content: "<base64>" }]` 経由で送信します
    - Gateway は添付を `images[]` に解析します（`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`）
    - Embedded agent はマルチモーダルな user message をモデルに転送します
    - 検証: 返信に `cat` + そのコードが含まれること（OCR 許容: 軽微な誤りは許可）

ヒント: 自分のマシンで何をテストできるか（および正確な `provider/model` id）を確認するには、次を実行してください。

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI バックエンドスモーク（Claude、Codex、Gemini、またはその他のローカル CLI）

- テスト: `src/gateway/gateway-cli-backend.live.test.ts`
- 目的: デフォルト設定に触れずに、ローカル CLI バックエンドを使って Gateway + エージェントパイプラインを検証する。
- バックエンド固有のデフォルトスモークは、所有する拡張機能の `cli-backend.ts` 定義にあります。
- 有効化:
  - `pnpm test:live`（または Vitest を直接起動する場合は `OPENCLAW_LIVE_TEST=1`）
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- デフォルト:
  - デフォルトの provider/model: `claude-cli/claude-sonnet-4-6`
  - command/args/image の振る舞いは、所有する CLI バックエンド Plugin メタデータから取得されます。
- 上書き（任意）:
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - 実際の image 添付を送信するには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`（パスはプロンプトに注入されます）。
  - image ファイルパスをプロンプト注入ではなく CLI 引数として渡すには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`。
  - `IMAGE_ARG` が設定されているときに image 引数の渡し方を制御するには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（または `"list"`）。
  - 2 回目のターンを送って resume フローを検証するには `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`。
  - デフォルトの Claude Sonnet -> Opus 同一セッション継続性プローブを無効にするには `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`（選択したモデルが切り替え先をサポートしている場合に強制的に有効化するには `1` を設定）。

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
- 所有する拡張機能から CLI スモークメタデータを解決し、その後、一致する Linux CLI パッケージ（`@anthropic-ai/claude-code`、`@openai/codex`、または `@google/gemini-cli`）を、`OPENCLAW_DOCKER_CLI_TOOLS_DIR`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）にあるキャッシュ可能な書き込み可能プレフィックスへインストールします。
- `pnpm test:docker:live-cli-backend:claude-subscription` には、`~/.claude/.credentials.json` の `claudeAiOauth.subscriptionType` または `claude setup-token` による `CLAUDE_CODE_OAUTH_TOKEN` のいずれかを通じた、ポータブルな Claude Code subscription OAuth が必要です。最初に Docker 内で直接 `claude -p` を検証し、その後 Anthropic API-key 環境変数を保持せずに 2 回の Gateway CLI-backend ターンを実行します。この subscription レーンは、Claude が現在サードパーティアプリ利用を通常の subscription plan 制限ではなく追加利用課金にルーティングするため、Claude MCP/tool および image プローブをデフォルトで無効にしています。
- live CLI-backend スモークは現在、Claude、Codex、Gemini に対して同じ end-to-end フローを実行します: テキストターン、image 分類ターン、その後 Gateway CLI 経由で検証される MCP `cron` ツール呼び出しです。
- Claude のデフォルトスモークは、セッションを Sonnet から Opus に patch し、再開されたセッションが以前のメモを引き続き覚えていることも検証します。

## Live: ACP bind スモーク（`/acp spawn ... --bind here`）

- テスト: `src/gateway/gateway-acp-bind.live.test.ts`
- 目的: live ACP エージェントを使った実際の ACP conversation-bind フローを検証する:
  - `/acp spawn <agent> --bind here` を送る
  - synthetic な message-channel conversation をその場で bind する
  - 同じ conversation 上で通常の follow-up を送る
  - follow-up が bind 済み ACP セッショントランスクリプトに入ることを検証する
- 有効化:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- デフォルト:
  - Docker 内の ACP エージェント: `claude,codex,gemini`
  - 直接 `pnpm test:live ...` 用の ACP エージェント: `claude`
  - synthetic channel: Slack DM 形式の conversation コンテキスト
  - ACP バックエンド: `acpx`
- 上書き:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- 注記:
  - このレーンは、admin 専用の synthetic originating-route フィールドを持つ Gateway の `chat.send` サーフェスを使うため、外部配信を装わずにテストが message-channel コンテキストを付与できます。
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` が未設定の場合、テストは選択された ACP harness エージェントに対して、埋め込み `acpx` Plugin の組み込みエージェントレジストリを使います。

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

単一エージェントの Docker レシピ:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Docker に関する注記:

- Docker ランナーは `scripts/test-live-acp-bind-docker.sh` にあります。
- デフォルトでは、サポートされているすべての live CLI エージェントに対して ACP bind スモークを順番に実行します: `claude`、`codex`、`gemini`。
- マトリクスを絞り込むには `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`、または `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` を使います。
- これは `~/.profile` を読み込み、一致する CLI 認証マテリアルをコンテナへステージし、`acpx` を書き込み可能な npm プレフィックスへインストールし、その後、要求された live CLI（`@anthropic-ai/claude-code`、`@openai/codex`、または `@google/gemini-cli`）が存在しない場合はインストールします。
- Docker 内では、ランナーは `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` を設定するため、acpx は読み込まれた profile のプロバイダー環境変数を子 harness CLI で利用可能なまま保てます。

## Live: Codex app-server harness スモーク

- 目的: 通常の Gateway
  `agent` メソッドを通じて、Plugin 所有の Codex harness を検証する:
  - バンドルされた `codex` Plugin を読み込む
  - `OPENCLAW_AGENT_RUNTIME=codex` を選択する
  - `codex/gpt-5.4` に対して最初の Gateway agent ターンを送る
  - 同じ OpenClaw セッションに 2 回目のターンを送り、app-server
    thread が resume できることを検証する
  - 同じ Gateway command
    パスを通じて `/codex status` と `/codex models` を実行する
- テスト: `src/gateway/gateway-codex-harness.live.test.ts`
- 有効化: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- デフォルトモデル: `codex/gpt-5.4`
- 任意の image プローブ: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- 任意の MCP/tool プローブ: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- このスモークは `OPENCLAW_AGENT_HARNESS_FALLBACK=none` を設定するため、壊れた Codex
  harness が PI へのサイレントフォールバックで通過することはありません。
- 認証: シェル/profile からの `OPENAI_API_KEY`、加えて任意でコピーされた
  `~/.codex/auth.json` と `~/.codex/config.toml`

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

Docker に関する注記:

- Docker ランナーは `scripts/test-live-codex-harness-docker.sh` にあります。
- これはマウントされた `~/.profile` を読み込み、`OPENAI_API_KEY` を渡し、存在する場合は Codex CLI
  認証ファイルをコピーし、`@openai/codex` を書き込み可能なマウント済み npm
  プレフィックスへインストールし、ソースツリーをステージして、その後 Codex-harness live テストのみを実行します。
- Docker はデフォルトで image および MCP/tool プローブを有効にします。より狭いデバッグ実行が必要な場合は
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` または
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` を設定してください。
- Docker も `OPENCLAW_AGENT_HARNESS_FALLBACK=none` をエクスポートし、live
  テスト設定と一致させるため、`openai-codex/*` または PI フォールバックで Codex harness
  のリグレッションが隠れることはありません。

### 推奨される live レシピ

狭く明示的な allowlist が最も高速で、不安定さも最小です。

- 単一モデル、直接（Gateway なし）:
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- 単一モデル、Gateway スモーク:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 複数プロバイダーにわたる tool calling:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google にフォーカス（Gemini API key + Antigravity）:
  - Gemini（API key）: `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）: `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

注記:

- `google/...` は Gemini API（API キー）を使用します。
- `google-antigravity/...` は Antigravity OAuth ブリッジ（Cloud Code Assist 形式のエージェントエンドポイント）を使用します。
- `google-gemini-cli/...` は、あなたのマシン上のローカル Gemini CLI を使用します（認証や tooling の癖は別です）。
- Gemini API と Gemini CLI:
  - API: OpenClaw は、Google がホストする Gemini API を HTTP 経由で呼び出します（API キー / profile 認証）。多くのユーザーが「Gemini」と言うとき、通常はこちらを指します。
  - CLI: OpenClaw はローカルの `gemini` バイナリをシェル実行します。独自の認証を持ち、挙動が異なる場合があります（streaming/tool サポート/version のずれ）。

## Live: model matrix（何をカバーするか）

固定の「CI model list」はありません（live はオプトインです）が、キーを持つ開発マシンで定期的にカバーする **推奨** モデルは次のとおりです。

### Modern スモークセット（tool calling + image）

これは、継続して動作していることを期待する「一般的なモデル」の実行です。

- OpenAI（非 Codex）: `openai/gpt-5.4`（任意: `openai/gpt-5.4-mini`）
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6`（または `anthropic/claude-sonnet-4-6`）
- Google（Gemini API）: `google/gemini-3.1-pro-preview` および `google/gemini-3-flash-preview`（古い Gemini 2.x モデルは避ける）
- Google（Antigravity）: `google-antigravity/claude-opus-4-6-thinking` および `google-antigravity/gemini-3-flash`
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

ツール + image 付きで Gateway スモークを実行:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### ベースライン: tool calling（Read + 任意の Exec）

プロバイダーファミリーごとに少なくとも 1 つ選んでください。

- OpenAI: `openai/gpt-5.4`（または `openai/gpt-5.4-mini`）
- Anthropic: `anthropic/claude-opus-4-6`（または `anthropic/claude-sonnet-4-6`）
- Google: `google/gemini-3-flash-preview`（または `google/gemini-3.1-pro-preview`）
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

任意の追加カバレッジ（あるとよいもの）:

- xAI: `xai/grok-4`（または利用可能な最新）
- Mistral: `mistral/`…（有効化されている「tools」対応モデルを 1 つ選ぶ）
- Cerebras: `cerebras/`…（アクセス権がある場合）
- LM Studio: `lmstudio/`…（ローカル。tool calling は API モードに依存）

### Vision: image 送信（添付 → マルチモーダルメッセージ）

image プローブを実行するために、少なくとも 1 つの image 対応モデル（Claude/Gemini/OpenAI の vision 対応バリアントなど）を `OPENCLAW_LIVE_GATEWAY_MODELS` に含めてください。

### Aggregators / 代替 Gateway

キーが有効であれば、次経由のテストもサポートしています。

- OpenRouter: `openrouter/...`（数百のモデル。tool+image 対応候補を見つけるには `openclaw models scan` を使ってください）
- OpenCode: Zen 用は `opencode/...`、Go 用は `opencode-go/...`（認証は `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`）

live matrix に含められるその他のプロバイダー（認証情報/config がある場合）:

- 組み込み: `openai`、`openai-codex`、`anthropic`、`google`、`google-vertex`、`google-antigravity`、`google-gemini-cli`、`zai`、`openrouter`、`opencode`、`opencode-go`、`xai`、`groq`、`cerebras`、`mistral`、`github-copilot`
- `models.providers` 経由（カスタムエンドポイント）: `minimax`（cloud/API）、および任意の OpenAI/Anthropic 互換プロキシ（LM Studio、vLLM、LiteLLM など）

ヒント: ドキュメント内に「全モデル」をハードコードしようとしないでください。権威ある一覧は、あなたのマシン上で `discoverModels(...)` が返すものと、利用可能なキーによって決まります。

## 認証情報（絶対にコミットしない）

live テストは CLI と同じ方法で認証情報を検出します。実用上の意味は次のとおりです。

- CLI が動作するなら、live テストも同じキーを見つけられるはずです。
- live テストで「認証情報なし」と出る場合は、`openclaw models list` / モデル選択をデバッグするのと同じ方法でデバッグしてください。

- エージェントごとの認証 profile: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（live テストでいう「profile keys」とはこれを指します）
- config: `~/.openclaw/openclaw.json`（または `OPENCLAW_CONFIG_PATH`）
- レガシー state ディレクトリ: `~/.openclaw/credentials/`（存在する場合はステージされた live home にコピーされますが、メインの profile-key ストアではありません）
- ローカルの live 実行は、デフォルトでアクティブな config、エージェントごとの `auth-profiles.json` ファイル、レガシー `credentials/`、およびサポートされている外部 CLI 認証ディレクトリを一時的なテスト home にコピーします。ステージされた live home では `workspace/` と `sandboxes/` をスキップし、`agents.*.workspace` / `agentDir` のパス上書きは除去されるため、プローブが実際のホスト workspace に触れないようになっています。

env キー（たとえば `~/.profile` で export されたもの）に依存したい場合は、`source ~/.profile` 後にローカルテストを実行するか、以下の Docker ランナーを使ってください（`~/.profile` をコンテナにマウントできます）。

## Deepgram live（音声文字起こし）

- テスト: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- 有効化: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- テスト: `src/agents/byteplus.live.test.ts`
- 有効化: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- 任意のモデル上書き: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- テスト: `extensions/comfy/comfy.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- スコープ:
  - バンドルされた comfy の image、video、および `music_generate` パスを実行する
  - `models.providers.comfy.<capability>` が設定されていない限り、各 capability をスキップする
  - comfy workflow 送信、polling、ダウンロード、または Plugin 登録を変更した後に有用

## Image generation live

- テスト: `src/image-generation/runtime.live.test.ts`
- コマンド: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- スコープ:
  - 登録されているすべての image-generation provider Plugin を列挙する
  - プローブ前に、ログインシェル（`~/.profile`）から不足している provider 環境変数を読み込む
  - デフォルトでは、保存済み認証 profile より live/env API キーを優先して使うため、`auth-profiles.json` 内の古いテストキーが実際のシェル認証情報を覆い隠しません
  - 使用可能な auth/profile/model がないプロバイダーはスキップする
  - 共有 runtime capability を通じて、標準の image-generation バリアントを実行する:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- 現在カバーされているバンドル済みプロバイダー:
  - `openai`
  - `google`
- 任意の絞り込み:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- 任意の認証挙動:
  - env のみの上書きを無視して profile-store 認証を強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Music generation live

- テスト: `extensions/music-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- スコープ:
  - 共有のバンドル済み music-generation provider パスを実行する
  - 現在は Google と MiniMax をカバー
  - プローブ前に、ログインシェル（`~/.profile`）から provider 環境変数を読み込む
  - デフォルトでは、保存済み認証 profile より live/env API キーを優先して使うため、`auth-profiles.json` 内の古いテストキーが実際のシェル認証情報を覆い隠しません
  - 使用可能な auth/profile/model がないプロバイダーはスキップする
  - 利用可能な場合、宣言された両方の runtime モードを実行する:
    - プロンプトのみの入力による `generate`
    - プロバイダーが `capabilities.edit.enabled` を宣言している場合の `edit`
  - 現在の共有レーンのカバレッジ:
    - `google`: `generate`、`edit`
    - `minimax`: `generate`
    - `comfy`: 別個の Comfy live ファイルであり、この共有 sweep ではない
- 任意の絞り込み:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- 任意の認証挙動:
  - env のみの上書きを無視して profile-store 認証を強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Video generation live

- テスト: `extensions/video-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- スコープ:
  - 共有のバンドル済み video-generation provider パスを実行する
  - デフォルトでは、リリース安全なスモークパスを使います: 非 FAL プロバイダー、プロバイダーごとに 1 件の text-to-video リクエスト、1 秒の lobster プロンプト、および `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS`（デフォルト `180000`）によるプロバイダーごとの操作上限
  - FAL は、プロバイダー側のキュー遅延がリリース時間を支配しうるため、デフォルトでスキップされます。明示的に実行するには `--video-providers fal` または `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"` を渡してください
  - プローブ前に、ログインシェル（`~/.profile`）から provider 環境変数を読み込む
  - デフォルトでは、保存済み認証 profile より live/env API キーを優先して使うため、`auth-profiles.json` 内の古いテストキーが実際のシェル認証情報を覆い隠しません
  - 使用可能な auth/profile/model がないプロバイダーはスキップする
  - デフォルトでは `generate` のみを実行する
  - 利用可能な場合に宣言された transform モードも実行するには `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` を設定する:
    - プロバイダーが `capabilities.imageToVideo.enabled` を宣言し、選択されたプロバイダー/モデルが共有 sweep 内で buffer ベースのローカル image 入力を受け付ける場合の `imageToVideo`
    - プロバイダーが `capabilities.videoToVideo.enabled` を宣言し、選択されたプロバイダー/モデルが共有 sweep 内で buffer ベースのローカル video 入力を受け付ける場合の `videoToVideo`
  - 共有 sweep で現在宣言はされているがスキップされる `imageToVideo` プロバイダー:
    - バンドル済み `veo3` は text-only で、バンドル済み `kling` はリモート image URL を必要とするため `vydra`
  - プロバイダー固有の Vydra カバレッジ:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - そのファイルは、デフォルトでリモート image URL fixture を使う `kling` レーンに加えて `veo3` の text-to-video を実行します
  - 現在の `videoToVideo` live カバレッジ:
    - 選択されたモデルが `runway/gen4_aleph` の場合の `runway` のみ
  - 共有 sweep で現在宣言はされているがスキップされる `videoToVideo` プロバイダー:
    - `alibaba`、`qwen`、`xai` は、現在それらのパスでリモート `http(s)` / MP4 参照 URL が必要なため
    - `google` は、現在の共有 Gemini/Veo レーンがローカル buffer ベース入力を使っており、そのパスが共有 sweep では受け入れられないため
    - `openai` は、現在の共有レーンに org 固有の video inpaint/remix アクセス保証がないため
- 任意の絞り込み:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - デフォルト sweep で FAL を含むすべてのプロバイダーを含めるには `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`
  - 積極的なスモーク実行のために、各プロバイダー操作上限を減らすには `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`
- 任意の認証挙動:
  - env のみの上書きを無視して profile-store 認証を強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## Media live harness

- コマンド: `pnpm test:live:media`
- 目的:
  - 共有の image、music、video live スイートを、リポジトリネイティブの単一エントリーポイント経由で実行する
  - `~/.profile` から不足している provider 環境変数を自動読み込みする
  - デフォルトでは、現在使用可能な auth を持つプロバイダーに各スイートを自動で絞り込む
  - `scripts/test-live.mjs` を再利用するため、Heartbeat と quiet-mode の挙動が一貫する
- 例:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker ランナー（任意の「Linux で動作する」チェック）

これらの Docker ランナーは 2 つのバケットに分かれます:

- Live-model ランナー: `test:docker:live-models` と `test:docker:live-gateway` は、リポジトリ Docker イメージ内で対応する profile-key live ファイルのみを実行します（`src/agents/models.profiles.live.test.ts` と `src/gateway/gateway-models.profiles.live.test.ts`）。ローカルの config ディレクトリと workspace をマウントし（マウントされていれば `~/.profile` も読み込みます）、対応するローカルエントリーポイントは `test:live:models-profiles` と `test:live:gateway-profiles` です。
- Docker live ランナーは、完全な Docker sweep を現実的に保つため、デフォルトで小さめのスモーク上限を使います:
  `test:docker:live-models` はデフォルトで `OPENCLAW_LIVE_MAX_MODELS=12`、
  `test:docker:live-gateway` はデフォルトで `OPENCLAW_LIVE_GATEWAY_SMOKE=1`、
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`、
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`、および
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` を使います。より大きい網羅的スキャンを
  明示的に行いたい場合は、それらの環境変数を上書きしてください。
- `test:docker:all` はまず `test:docker:live-build` 経由で live Docker イメージを 1 回ビルドし、その後それを 2 つの live Docker レーンで再利用します。
- コンテナスモークランナー: `test:docker:openwebui`、`test:docker:onboard`、`test:docker:gateway-network`、`test:docker:mcp-channels`、`test:docker:plugins` は、1 つ以上の実コンテナを起動し、より高レベルの integration パスを検証します。

live-model Docker ランナーは、必要な CLI 認証 home のみ（または、実行が絞り込まれていない場合はサポートされているすべて）を bind-mount し、その後、実行前にそれらをコンテナ home にコピーします。これにより、外部 CLI OAuth がホストの認証ストアを変更せずにトークンを更新できます。

- 直接モデル: `pnpm test:docker:live-models`（スクリプト: `scripts/test-live-models-docker.sh`）
- ACP bind スモーク: `pnpm test:docker:live-acp-bind`（スクリプト: `scripts/test-live-acp-bind-docker.sh`）
- CLI バックエンドスモーク: `pnpm test:docker:live-cli-backend`（スクリプト: `scripts/test-live-cli-backend-docker.sh`）
- Codex app-server harness スモーク: `pnpm test:docker:live-codex-harness`（スクリプト: `scripts/test-live-codex-harness-docker.sh`）
- Gateway + dev agent: `pnpm test:docker:live-gateway`（スクリプト: `scripts/test-live-gateway-models-docker.sh`）
- Open WebUI live スモーク: `pnpm test:docker:openwebui`（スクリプト: `scripts/e2e/openwebui-docker.sh`）
- オンボーディング ウィザード（TTY、完全スキャフォールディング）: `pnpm test:docker:onboard`（スクリプト: `scripts/e2e/onboard-docker.sh`）
- Gateway ネットワーク（2 コンテナ、WS auth + health）: `pnpm test:docker:gateway-network`（スクリプト: `scripts/e2e/gateway-network-docker.sh`）
- MCP channel bridge（seed 済み Gateway + stdio bridge + 生の Claude notification-frame スモーク）: `pnpm test:docker:mcp-channels`（スクリプト: `scripts/e2e/mcp-channels-docker.sh`）
- Plugins（インストールスモーク + `/plugin` エイリアス + Claude バンドルの再起動セマンティクス）: `pnpm test:docker:plugins`（スクリプト: `scripts/e2e/plugins-docker.sh`）

live-model Docker ランナーは、現在のチェックアウトも読み取り専用で bind-mount し、
コンテナ内の一時 workdir にステージします。これにより、runtime
イメージをスリムに保ちながらも、正確にあなたのローカルの source/config に対して Vitest を実行できます。
ステージング手順では、`.pnpm-store`、`.worktrees`、`__openclaw_vitest__`、およびアプリローカルの `.build` や
Gradle 出力ディレクトリのような、大きなローカル専用キャッシュやアプリビルド出力をスキップするため、
Docker live 実行がマシン固有のアーティファクトのコピーに何分も費やすことはありません。
また、それらは `OPENCLAW_SKIP_CHANNELS=1` も設定するため、
Gateway live プローブがコンテナ内で実際の Telegram/Discord などのチャネルワーカーを起動しません。
`test:docker:live-models` は引き続き `pnpm test:live` を実行するため、
その Docker レーンから Gateway
live カバレッジを絞り込んだり除外したりする必要がある場合は、`OPENCLAW_LIVE_GATEWAY_*` も渡してください。
`test:docker:openwebui` は、より高レベルの互換性スモークです。これは
OpenAI 互換 HTTP エンドポイントを有効にした OpenClaw Gateway コンテナを起動し、
その Gateway に対して固定版の Open WebUI コンテナを起動し、
Open WebUI 経由でサインインし、
`/api/models` が `openclaw/default` を公開していることを確認した後、
Open WebUI の `/api/chat/completions` プロキシ経由で
実際の chat リクエストを送信します。
初回実行は、Docker が
Open WebUI イメージを pull する必要があったり、Open WebUI 自身のコールドスタートセットアップを完了する必要があるため、目に見えて遅くなることがあります。
このレーンは使用可能な live model key を想定しており、`OPENCLAW_PROFILE_FILE`
（デフォルトでは `~/.profile`）が Docker 化された実行でそれを提供する主な方法です。
成功した実行では `{ "ok": true, "model":
"openclaw/default", ... }` のような小さな JSON payload が出力されます。
`test:docker:mcp-channels` は意図的に決定的であり、実際の
Telegram、Discord、または iMessage アカウントを必要としません。これは seed 済み Gateway
コンテナを起動し、続いて `openclaw mcp serve` を起動する 2 つ目のコンテナを開始し、
ルーティングされた conversation 検出、transcript 読み取り、添付メタデータ、
live event queue の挙動、outbound 送信ルーティング、および Claude 形式の channel +
permission 通知を、実際の stdio MCP bridge 上で検証します。通知チェックは
生の stdio MCP フレームを直接検査するため、このスモークは
特定の client SDK がたまたま表面化するものだけでなく、bridge が実際に何を出力するかを検証します。

手動 ACP 平文スレッドスモーク（CI ではない）:

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- このスクリプトはリグレッション/デバッグワークフロー用に維持してください。ACP スレッドルーティング検証のために再び必要になる可能性があるため、削除しないでください。

便利な環境変数:

- `OPENCLAW_CONFIG_DIR=...`（デフォルト: `~/.openclaw`）は `/home/node/.openclaw` にマウントされます
- `OPENCLAW_WORKSPACE_DIR=...`（デフォルト: `~/.openclaw/workspace`）は `/home/node/.openclaw/workspace` にマウントされます
- `OPENCLAW_PROFILE_FILE=...`（デフォルト: `~/.profile`）は `/home/node/.profile` にマウントされ、テスト実行前に読み込まれます
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1` は、`OPENCLAW_PROFILE_FILE` から読み込まれた env vars のみを検証します。一時的な config/workspace ディレクトリを使用し、外部 CLI 認証マウントは行いません
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）は `/home/node/.npm-global` にマウントされ、Docker 内での CLI インストールキャッシュに使われます
- `$HOME` 配下の外部 CLI 認証ディレクトリ/ファイルは `/host-auth...` 配下に読み取り専用でマウントされ、その後テスト開始前に `/home/node/...` へコピーされます
  - デフォルトディレクトリ: `.minimax`
  - デフォルトファイル: `~/.codex/auth.json`、`~/.codex/config.toml`、`.claude.json`、`~/.claude/.credentials.json`、`~/.claude/settings.json`、`~/.claude/settings.local.json`
  - 絞り込まれたプロバイダー実行では、`OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` から推定される必要なディレクトリ/ファイルのみをマウントします
  - 手動上書きは `OPENCLAW_DOCKER_AUTH_DIRS=all`、`OPENCLAW_DOCKER_AUTH_DIRS=none`、または `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` のようなカンマ区切り一覧で行えます
- 実行を絞り込むには `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- コンテナ内でプロバイダーを絞り込むには `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- 再ビルド不要の再実行で既存の `openclaw:local-live` イメージを再利用するには `OPENCLAW_SKIP_DOCKER_BUILD=1`
- 認証情報が env ではなく profile store から来ることを保証するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`
- Open WebUI スモーク向けに Gateway が公開するモデルを選ぶには `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI スモークで使う nonce チェックプロンプトを上書きするには `OPENCLAW_OPENWEBUI_PROMPT=...`
- 固定された Open WebUI イメージタグを上書きするには `OPENWEBUI_IMAGE=...`

## ドキュメントの健全性

ドキュメント編集後は docs チェックを実行してください: `pnpm check:docs`。
ページ内見出しチェックも必要な場合は、完全な Mintlify アンカー検証を実行してください: `pnpm docs:check-links:anchors`。

## オフラインリグレッション（CI 安全）

これらは実際のプロバイダーなしで行う「実パイプライン」リグレッションです。

- Gateway tool calling（mock OpenAI、実際の Gateway + agent loop）: `src/gateway/gateway.test.ts`（ケース: 「runs a mock OpenAI tool call end-to-end via gateway agent loop」）
- Gateway ウィザード（WS `wizard.start`/`wizard.next`、config + auth の書き込みを強制）: `src/gateway/gateway.test.ts`（ケース: 「runs wizard over ws and writes auth token config」）

## エージェント信頼性 evals（Skills）

CI 安全で「エージェント信頼性 evals」のように振る舞うテストは、すでにいくつかあります。

- 実際の Gateway + agent loop を通る mock tool-calling（`src/gateway/gateway.test.ts`）。
- セッション配線と config 効果を検証する end-to-end のウィザードフロー（`src/gateway/gateway.test.ts`）。

Skills に関してまだ不足しているもの（[Skills](/ja-JP/tools/skills) を参照）:

- **Decisioning:** Skills がプロンプトに列挙されたとき、エージェントは正しい skill を選ぶか（または無関係なものを避けるか）？
- **Compliance:** エージェントは使用前に `SKILL.md` を読み、必要な手順/引数に従うか？
- **Workflow contracts:** ツール順序、セッション履歴の引き継ぎ、sandbox 境界を検証する複数ターンのシナリオ。

将来の evals は、まず決定的であることを優先すべきです。

- mock providers を使ってツール呼び出し + 順序、skill ファイル読み取り、セッション配線を検証する scenario runner。
- スキルに焦点を当てた小さなシナリオスイート（使う vs 避ける、ゲーティング、prompt injection）。
- CI 安全なスイートが整った後に限り、任意の live evals（オプトイン、env ゲート付き）。

## Contract テスト（Plugin とチャネルの形状）

Contract テストは、登録されたすべての Plugin とチャネルがその
インターフェース契約に準拠していることを検証します。これらは検出されたすべての Plugin を反復し、
形状と振る舞いに関する一連のアサーションを実行します。デフォルトの `pnpm test` unit レーンは、
これらの共有 seam およびスモークファイルを意図的にスキップします。共有チャネルまたは provider サーフェスに触れた場合は、
contract コマンドを明示的に実行してください。

### コマンド

- すべての contract: `pnpm test:contracts`
- チャネル contract のみ: `pnpm test:contracts:channels`
- provider contract のみ: `pnpm test:contracts:plugins`

### チャネル contract

`src/channels/plugins/contracts/*.contract.test.ts` にあります:

- **plugin** - 基本的な Plugin の形状（id、name、capabilities）
- **setup** - セットアップ ウィザード契約
- **session-binding** - セッションバインディングの振る舞い
- **outbound-payload** - メッセージ payload 構造
- **inbound** - inbound メッセージ処理
- **actions** - チャネル action ハンドラー
- **threading** - thread ID 処理
- **directory** - ディレクトリ/roster API
- **group-policy** - グループポリシーの強制

### Provider status contract

`src/plugins/contracts/*.contract.test.ts` にあります。

- **status** - チャネル status プローブ
- **registry** - Plugin レジストリの形状

### Provider contract

`src/plugins/contracts/*.contract.test.ts` にあります:

- **auth** - 認証フロー契約
- **auth-choice** - 認証の選択
- **catalog** - モデルカタログ API
- **discovery** - Plugin 検出
- **loader** - Plugin 読み込み
- **runtime** - provider runtime
- **shape** - Plugin の形状/インターフェース
- **wizard** - セットアップ ウィザード

### 実行するタイミング

- plugin-sdk の export または subpath を変更した後
- チャネルまたは provider Plugin を追加または変更した後
- Plugin 登録または検出をリファクタリングした後

Contract テストは CI で実行され、実際の API キーは必要ありません。

## リグレッションを追加する（ガイダンス）

live で見つかった provider/model の問題を修正したとき:

- 可能であれば CI 安全なリグレッションを追加する（provider を mock/stub する、または正確な request-shape 変換をキャプチャする）
- 本質的に live 専用（rate limit、認証ポリシーなど）の場合は、live テストを狭く保ち、env vars によるオプトインにする
- バグを捕まえられる最小のレイヤーを対象にすることを優先する:
  - provider request 変換/再生バグ → 直接モデルテスト
  - gateway session/history/tool パイプラインバグ → Gateway live スモークまたは CI 安全な Gateway mock テスト
- SecretRef トラバーサルのガードレール:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` は、レジストリメタデータ（`listSecretTargetRegistryEntries()`）から SecretRef クラスごとに 1 つのサンプル対象を導出し、その後トラバーサルセグメント exec id が拒否されることを検証します。
  - `src/secrets/target-registry-data.ts` に新しい `includeInPlan` SecretRef 対象ファミリーを追加する場合は、そのテスト内の `classifyTargetClass` を更新してください。このテストは、分類されていない target id に対して意図的に失敗するため、新しいクラスが黙ってスキップされることはありません。
