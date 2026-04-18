---
read_when:
    - ローカルまたはCIでテストを実行する
    - モデル/プロバイダーのバグに対するリグレッションテストを追加する
    - Gateway + agentの動作をデバッグする
summary: 'テストキット: unit/e2e/liveスイート、Dockerランナー、および各テストがカバーする内容'
title: テスト
x-i18n:
    generated_at: "2026-04-18T04:40:08Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7cdd2048ba58e606fd68703977c2b33000abdb1826b6589ce25a35c53468726a
    source_path: help/testing.md
    workflow: 15
---

# テスト

OpenClawには3つのVitestスイート（unit/integration、e2e、live）と、少数のDockerランナーがあります。

このドキュメントは「OpenClawでどのようにテストするか」のガイドです。

- 各スイートが何をカバーするか（そして意図的に _何をカバーしないか_）
- 一般的なワークフロー（ローカル、push前、デバッグ）でどのコマンドを実行するか
- liveテストがどのように認証情報を見つけ、モデル/プロバイダーを選択するか
- 実際のモデル/プロバイダーの問題に対するリグレッションをどのように追加するか

## クイックスタート

たいていの日は次のとおりです。

- フルゲート（push前に期待される）: `pnpm build && pnpm check && pnpm test`
- 余裕のあるマシンでの、より高速なローカル全スイート実行: `pnpm test:max`
- 直接のVitest watchループ: `pnpm test:watch`
- 直接のファイル指定は、extension/channelパスも対象になりました: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- 単一の失敗を反復修正しているときは、まず対象を絞った実行を優先してください。
- DockerベースのQAサイト: `pnpm qa:lab:up`
- Linux VMベースのQAレーン: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

テストに変更を加えたとき、または追加の確信がほしいとき:

- カバレッジゲート: `pnpm test:coverage`
- E2Eスイート: `pnpm test:e2e`

実際のプロバイダー/モデルをデバッグするとき（実際の認証情報が必要）:

- Liveスイート（モデル + Gatewayのtool/imageプローブ）: `pnpm test:live`
- 1つのliveファイルを静かに対象指定: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

ヒント: 失敗している1ケースだけが必要な場合は、後述のallowlist環境変数を使ってliveテストを絞り込むことを優先してください。

## QA専用ランナー

これらのコマンドは、QA-labの現実性が必要なときに、メインのテストスイートと並んで使います。

- `pnpm openclaw qa suite`
  - リポジトリベースのQAシナリオをホスト上で直接実行します。
  - デフォルトでは、分離されたgatewayワーカーを使って複数の選択シナリオを並列実行し、最大64ワーカーまたは選択シナリオ数まで使用します。`--concurrency <count>`でワーカー数を調整するか、古い直列レーンには`--concurrency 1`を使います。
  - プロバイダーモード `live-frontier`、`mock-openai`、`aimock` をサポートします。`aimock`は、シナリオ対応の `mock-openai` レーンを置き換えることなく、実験的なフィクスチャおよびプロトコルモックのカバレッジのためにローカルのAIMockベースプロバイダーサーバーを起動します。
- `pnpm openclaw qa suite --runner multipass`
  - 同じQAスイートを使い捨てのMultipass Linux VM内で実行します。
  - ホスト上の `qa suite` と同じシナリオ選択動作を維持します。
  - `qa suite` と同じプロバイダー/モデル選択フラグを再利用します。
  - live実行では、ゲストにとって実用的な対応済みQA認証入力を転送します:
    環境変数ベースのプロバイダーキー、QA live provider configパス、および存在する場合は `CODEX_HOME`。
  - 出力ディレクトリは、ゲストがマウントされたワークスペース経由で書き戻せるよう、リポジトリルート配下に置く必要があります。
  - 通常のQAレポート + サマリーに加えて、Multipassログを `.artifacts/qa-e2e/...` 配下に書き込みます。
- `pnpm qa:lab:up`
  - オペレーター形式のQA作業のために、DockerベースのQAサイトを起動します。
- `pnpm openclaw qa aimock`
  - 直接のプロトコルスモークテストのために、ローカルのAIMockプロバイダーサーバーのみを起動します。
- `pnpm openclaw qa matrix`
  - 使い捨てのDockerベースTuwunel homeserverに対して、Matrix live QAレーンを実行します。
  - このQAホストは現時点ではrepo/dev専用です。パッケージ化されたOpenClawインストールには `qa-lab` が含まれないため、`openclaw qa` は公開されません。
  - リポジトリのチェックアウトでは、同梱ランナーを直接読み込みます。別個のPluginインストール手順は不要です。
  - 3人の一時的なMatrixユーザー（`driver`、`sut`、`observer`）と1つのプライベートルームをプロビジョニングし、その後で実際のMatrix PluginをSUTトランスポートとして使うQA gateway子プロセスを起動します。
  - デフォルトでは、固定された安定版Tuwunelイメージ `ghcr.io/matrix-construct/tuwunel:v1.5.1` を使用します。別のイメージをテストする必要がある場合は `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` で上書きしてください。
  - Matrixは、レーンがローカルで使い捨てユーザーをプロビジョニングするため、共有credential-sourceフラグを公開しません。
  - Matrix QAレポート、サマリー、observed-eventsアーティファクト、および結合されたstdout/stderr出力ログを `.artifacts/qa-e2e/...` 配下に書き込みます。
- `pnpm openclaw qa telegram`
  - env内のdriverおよびSUT botトークンを使って、実際のプライベートグループに対するTelegram live QAレーンを実行します。
  - `OPENCLAW_QA_TELEGRAM_GROUP_ID`、`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` が必要です。group idは数値のTelegram chat idである必要があります。
  - 共有プール認証情報向けに `--credential-source convex` をサポートします。通常はenvモードを使い、プール済みリースを使う場合は `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` を設定してください。
  - 同じプライベートグループ内にある2つの異なるbotが必要で、SUT botはTelegram usernameを公開している必要があります。
  - bot同士の安定した観測のため、両方のbotで `@BotFather` のBot-to-Bot Communication Modeを有効にし、driver botがグループ内のbotトラフィックを観測できることを確認してください。
  - Telegram QAレポート、サマリー、およびobserved-messagesアーティファクトを `.artifacts/qa-e2e/...` 配下に書き込みます。

liveトランスポートレーンは、追加される新しいトランスポートが逸脱しないよう、1つの標準契約を共有します。

`qa-channel` は引き続き広範な合成QAスイートであり、live
transport coverage matrixには含まれません。

| Lane     | Canary | Mention gating | Allowlist block | Top-level reply | Restart resume | Thread follow-up | Thread isolation | Reaction observation | Help command |
| -------- | ------ | -------------- | --------------- | --------------- | -------------- | ---------------- | ---------------- | -------------------- | ------------ |
| Matrix   | x      | x              | x               | x               | x              | x                | x                | x                    |              |
| Telegram | x      |                |                 |                 |                |                  |                  |                      | x            |

### Convex経由の共有Telegram認証情報（v1）

`openclaw qa telegram` に対して `--credential-source convex`（または `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`）が有効な場合、
QA labはConvexベースのプールから排他的リースを取得し、レーン実行中はそのリースにHeartbeatを送り、終了時にリースを解放します。

参照用Convexプロジェクトスキャフォールド:

- `qa/convex-credential-broker/`

必須の環境変数:

- `OPENCLAW_QA_CONVEX_SITE_URL`（例: `https://your-deployment.convex.site`）
- 選択したロールに応じた1つのシークレット:
  - `maintainer` 用の `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
  - `ci` 用の `OPENCLAW_QA_CONVEX_SECRET_CI`
- credentialロール選択:
  - CLI: `--credential-role maintainer|ci`
  - 環境変数のデフォルト: `OPENCLAW_QA_CREDENTIAL_ROLE`（デフォルトは `maintainer`）

任意の環境変数:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS`（デフォルト `1200000`）
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS`（デフォルト `30000`）
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS`（デフォルト `90000`）
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS`（デフォルト `15000`）
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`（デフォルト `/qa-credentials/v1`）
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID`（任意のトレースid）
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` は、ローカル専用開発のためにloopback `http://` Convex URLを許可します。

通常運用では `OPENCLAW_QA_CONVEX_SITE_URL` は `https://` を使う必要があります。

maintainer用の管理コマンド（pool add/remove/list）には、
特に `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` が必要です。

maintainer向けCLIヘルパー:

```bash
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

スクリプトやCIユーティリティで機械可読な出力が必要な場合は `--json` を使用してください。

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
- `POST /admin/add`（maintainerシークレットのみ）
  - リクエスト: `{ kind, actorId, payload, note?, status? }`
  - 成功: `{ status: "ok", credential }`
- `POST /admin/remove`（maintainerシークレットのみ）
  - リクエスト: `{ credentialId, actorId }`
  - 成功: `{ status: "ok", changed, credential }`
  - アクティブリース保護: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list`（maintainerシークレットのみ）
  - リクエスト: `{ kind?, status?, includePayload?, limit? }`
  - 成功: `{ status: "ok", credentials, count }`

Telegram kind用のペイロード形式:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` は数値のTelegram chat id文字列である必要があります。
- `admin/add` は `kind: "telegram"` に対してこの形式を検証し、不正なペイロードを拒否します。

### QAにchannelを追加する

markdown QAシステムにchannelを追加するには、必要なものは正確に2つだけです。

1. そのchannel用のtransport adapter。
2. channel契約を検証するscenario pack。

共有 `qa-lab` ホストがフローを管理できる場合、新しいトップレベルQAコマンドrootを追加しないでください。

`qa-lab` は共有ホストの仕組みを管理します:

- `openclaw qa` コマンドroot
- スイートの起動と終了
- ワーカー並列数
- アーティファクト書き込み
- レポート生成
- シナリオ実行
- 古い `qa-channel` シナリオ向けの互換エイリアス

runner Pluginはtransport契約を管理します:

- `openclaw qa <runner>` が共有 `qa` root配下でどのようにマウントされるか
- そのtransport向けにgatewayがどのように設定されるか
- readinessをどのように確認するか
- inbound eventをどのように注入するか
- outbound messageをどのように観測するか
- transcriptおよび正規化されたtransport stateをどのように公開するか
- transportを使ったactionをどのように実行するか
- transport固有のリセットまたはクリーンアップをどのように処理するか

新しいchannelを採用するための最小基準は次のとおりです。

1. 共有 `qa` rootの管理者は `qa-lab` のままにする。
2. 共有 `qa-lab` ホスト境界上にtransport runnerを実装する。
3. transport固有の仕組みはrunner Pluginまたはchannel harness内に保持する。
4. 競合するrootコマンドを登録するのではなく、runnerを `openclaw qa <runner>` としてマウントする。
   runner Pluginは `openclaw.plugin.json` で `qaRunners` を宣言し、`runtime-api.ts` から対応する `qaRunnerCliRegistrations` 配列をエクスポートする必要があります。
   `runtime-api.ts` は軽量に保ってください。遅延CLIおよびrunner実行は、別個のentrypointの背後に置く必要があります。
5. テーマ化された `qa/scenarios/` ディレクトリ配下にmarkdownシナリオを作成または適応する。
6. 新しいシナリオには汎用scenario helperを使用する。
7. リポジトリが意図的な移行を行っている場合を除き、既存の互換エイリアスは動作したままにする。

判断ルールは厳格です。

- 振る舞いを `qa-lab` で一度だけ表現できるなら、`qa-lab` に置く。
- 振る舞いが1つのchannel transportに依存するなら、そのrunner Pluginまたはplugin harness内に保持する。
- シナリオが複数のchannelで使える新しい機能を必要とするなら、`suite.ts` にchannel固有の分岐を追加するのではなく、汎用helperを追加する。
- 振る舞いが1つのtransportにしか意味を持たないなら、そのシナリオをtransport固有のものとして保持し、それをシナリオ契約で明示する。

新しいシナリオ向けの推奨汎用helper名は次のとおりです。

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

既存シナリオ向けには、次を含む互換エイリアスが引き続き利用できます。

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

新しいchannel作業では、汎用helper名を使用する必要があります。
互換エイリアスは一斉移行を避けるために存在しており、
新しいシナリオ作成のモデルではありません。

## テストスイート（どこで何が実行されるか）

スイートは「現実性が増していくもの」（そして不安定さ/コストも増していくもの）として考えてください。

### Unit / integration（デフォルト）

- コマンド: `pnpm test`
- 設定: 既存のスコープ付きVitest projectに対する10個の逐次shard実行（`vitest.full-*.config.ts`）
- ファイル: `src/**/*.test.ts`、`packages/**/*.test.ts`、`test/**/*.test.ts` 配下のcore/unitインベントリと、`vitest.unit.config.ts` でカバーされる許可済み `ui` nodeテスト
- 範囲:
  - 純粋なunitテスト
  - プロセス内integrationテスト（gateway auth、routing、tooling、parsing、config）
  - 既知のバグに対する決定的なリグレッション
- 想定:
  - CIで実行される
  - 実際のキーは不要
  - 高速で安定しているべき
- Projectsメモ:
  - 対象指定なしの `pnpm test` は、1つの巨大なネイティブroot-projectプロセスではなく、11個のより小さいshard設定（`core-unit-src`、`core-unit-security`、`core-unit-ui`、`core-unit-support`、`core-support-boundary`、`core-contracts`、`core-bundled`、`core-runtime`、`agentic`、`auto-reply`、`extensions`）を実行するようになりました。これにより、負荷の高いマシンでのピークRSSが削減され、auto-reply/extensionの処理が無関係なスイートを圧迫するのを防ぎます。
  - `pnpm test --watch` は引き続きネイティブrootの `vitest.config.ts` project graphを使用します。multi-shardのwatchループは現実的ではないためです。
  - `pnpm test`、`pnpm test:watch`、`pnpm test:perf:imports` は、明示的なファイル/ディレクトリ指定をまずスコープ付きレーン経由にルーティングするため、`pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` は完全なroot project起動コストを払わずに済みます。
  - `pnpm test:changed` は、差分がルーティング可能なsource/testファイルだけに触れている場合、変更されたgitパスを同じスコープ付きレーンに展開します。config/setupの編集は引き続き広いroot-project再実行にフォールバックします。
  - agents、commands、plugins、auto-reply helper、`plugin-sdk`、および同様の純粋なutility領域からのimportが軽いunitテストは、`test/setup-openclaw-runtime.ts` をスキップする `unit-fast` レーン経由にルーティングされます。stateful/runtime-heavyなファイルは既存レーンに残ります。
  - 一部の `plugin-sdk` と `commands` helper sourceファイルも、changed-mode実行をこれらの軽量レーン内の明示的な隣接テストへマップするため、helper編集時にそのディレクトリ全体の重いスイートを再実行せずに済みます。
  - `auto-reply` は現在、3つの専用バケットを持ちます: トップレベルcore helper、トップレベル `reply.*` integrationテスト、および `src/auto-reply/reply/**` サブツリーです。これにより、最も重いreply harness処理が軽量なstatus/chunk/tokenテストに乗らないようにしています。
- Embedded runnerメモ:
  - message-tool discovery入力またはCompaction runtime contextを変更する場合は、両レベルのカバレッジを維持してください。
  - 純粋なrouting/normalization境界に対して、焦点を絞ったhelperリグレッションを追加してください。
  - さらに、embedded runner integrationスイートも健全な状態に保ってください:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`、
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`、および
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`。
  - これらのスイートは、スコープ付きidとCompactionの振る舞いが実際の `run.ts` / `compact.ts` パスを通って引き続き流れることを検証します。helperのみのテストは、これらのintegrationパスの十分な代替にはなりません。
- Poolメモ:
  - ベースVitest設定は現在デフォルトで `threads` を使用します。
  - 共有Vitest設定は `isolate: false` も固定し、root projects、e2e、live設定全体で非分離runnerを使用します。
  - root UIレーンは `jsdom` setupとoptimizerを維持していますが、現在は共有の非分離runner上で実行されます。
  - 各 `pnpm test` shardは、共有Vitest設定から同じ `threads` + `isolate: false` のデフォルトを継承します。
  - 共有 `scripts/run-vitest.mjs` launcherは、Vitest子Nodeプロセスに対してデフォルトで `--no-maglev` も追加するようになり、大規模なローカル実行時のV8コンパイル負荷を減らします。標準のV8挙動と比較したい場合は `OPENCLAW_VITEST_ENABLE_MAGLEV=1` を設定してください。
- Fast-local iterationメモ:
  - `pnpm test:changed` は、変更パスがより小さいスイートにきれいにマップされる場合、スコープ付きレーン経由にルーティングされます。
  - `pnpm test:max` と `pnpm test:changed:max` も同じルーティング挙動を維持しつつ、より高いworker上限を使います。
  - ローカルworkerの自動スケーリングは現在意図的に保守的で、ホストのload averageがすでに高い場合にも抑制されるため、複数のVitest実行が同時に走ってもデフォルトで与える影響が小さくなります。
  - ベースVitest設定は、テスト配線が変わったときにchanged-mode再実行の正しさを保つため、projects/configファイルを `forceRerunTriggers` としてマークします。
  - この設定は、対応ホスト上では `OPENCLAW_VITEST_FS_MODULE_CACHE` を有効のままにします。直接プロファイリング用に明示的なキャッシュ場所を1つ使いたい場合は `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` を設定してください。
- Perf-debugメモ:
  - `pnpm test:perf:imports` は、Vitestのimport所要時間レポートとimport-breakdown出力を有効にします。
  - `pnpm test:perf:imports:changed` は、同じプロファイリング表示を `origin/main` 以降に変更されたファイルに絞ります。
- `pnpm test:perf:changed:bench -- --ref <git-ref>` は、そのコミット済み差分について、ルーティングされた `test:changed` とネイティブroot-projectパスを比較し、wall timeとmacOS max RSSを出力します。
- `pnpm test:perf:changed:bench -- --worktree` は、変更済みファイル一覧を `scripts/test-projects.mjs` とroot Vitest設定に通して、現在のdirty treeをベンチマークします。
  - `pnpm test:perf:profile:main` は、Vitest/Viteの起動およびtransformオーバーヘッドのmain-thread CPU profileを書き出します。
  - `pnpm test:perf:profile:runner` は、unitスイートに対してファイル並列を無効化したrunner CPU+heap profileを書き出します。

### E2E（gateway smoke）

- コマンド: `pnpm test:e2e`
- 設定: `vitest.e2e.config.ts`
- ファイル: `src/**/*.e2e.test.ts`、`test/**/*.e2e.test.ts`
- Runtimeのデフォルト:
  - リポジトリの他の部分と同様に、Vitest `threads` を `isolate: false` で使用します。
  - 適応的workerを使用します（CI: 最大2、ローカル: デフォルト1）。
  - console I/Oオーバーヘッドを減らすため、デフォルトでsilent modeで実行します。
- 便利な上書き:
  - worker数を強制する `OPENCLAW_E2E_WORKERS=<n>`（上限16）
  - 詳細なconsole出力を再有効化する `OPENCLAW_E2E_VERBOSE=1`
- 範囲:
  - 複数インスタンスgatewayのエンドツーエンド動作
  - WebSocket/HTTP surface、node pairing、およびより重いネットワーキング
- 想定:
  - CIで実行される（パイプラインで有効な場合）
  - 実際のキーは不要
  - unitテストよりも可動部分が多い（遅くなることがある）

### E2E: OpenShell backend smoke

- コマンド: `pnpm test:e2e:openshell`
- ファイル: `test/openshell-sandbox.e2e.test.ts`
- 範囲:
  - Docker経由でホスト上に分離されたOpenShell gatewayを起動する
  - 一時的なローカルDockerfileからsandboxを作成する
  - 実際の `sandbox ssh-config` + SSH execを介してOpenClawのOpenShell backendを検証する
  - sandbox fs bridgeを通じてremote-canonical filesystemの挙動を検証する
- 想定:
  - オプトインのみ。デフォルトの `pnpm test:e2e` 実行には含まれない
  - ローカルの `openshell` CLIと動作するDocker daemonが必要
  - 分離された `HOME` / `XDG_CONFIG_HOME` を使用し、その後テストgatewayとsandboxを破棄する
- 便利な上書き:
  - より広いe2eスイートを手動実行する際にこのテストを有効化する `OPENCLAW_E2E_OPENSHELL=1`
  - デフォルト以外のCLIバイナリまたはwrapper scriptを指す `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`

### Live（実際のプロバイダー + 実際のモデル）

- コマンド: `pnpm test:live`
- 設定: `vitest.live.config.ts`
- ファイル: `src/**/*.live.test.ts`
- デフォルト: `pnpm test:live` により **有効**（`OPENCLAW_LIVE_TEST=1` を設定）
- 範囲:
  - 「このプロバイダー/モデルは、実際の認証情報で _今日_ 本当に動くか？」
  - プロバイダー形式変更、tool-callingの癖、authの問題、rate limitの挙動を検出する
- 想定:
  - 設計上CIで安定しない（実ネットワーク、実プロバイダーポリシー、quota、障害）
  - 費用がかかる / rate limitを消費する
  - 「全部」よりも、絞り込んだサブセットを実行することを推奨
- live実行は、不足しているAPIキーを拾うために `~/.profile` を読み込みます。
- デフォルトでは、live実行は依然として `HOME` を分離し、config/auth素材を一時的なテストhomeにコピーするため、unit fixtureが実際の `~/.openclaw` を変更できません。
- liveテストに意図的に実際のhome directoryを使わせたい場合にのみ `OPENCLAW_LIVE_USE_REAL_HOME=1` を設定してください。
- `pnpm test:live` は現在、より静かなモードがデフォルトです。`[live] ...` の進捗出力は維持しつつ、追加の `~/.profile` 通知を抑制し、gateway bootstrapログやBonjourの雑音をミュートします。完全な起動ログを再表示したい場合は `OPENCLAW_LIVE_TEST_QUIET=0` を設定してください。
- APIキーのローテーション（プロバイダー別）: カンマ/セミコロン形式の `*_API_KEYS`、または `*_API_KEY_1`、`*_API_KEY_2`（例: `OPENAI_API_KEYS`、`ANTHROPIC_API_KEYS`、`GEMINI_API_KEYS`）、あるいはlive専用の上書きとして `OPENCLAW_LIVE_*_KEY` を設定します。テストはrate limit応答時に再試行します。
- 進捗/Heartbeat出力:
  - liveスイートは現在、長いプロバイダー呼び出し中でもVitestのconsole captureが静かなときに動作中であることが見えるよう、進捗行をstderrに出力します。
  - `vitest.live.config.ts` はVitestのconsole interceptionを無効にするため、プロバイダー/gatewayの進捗行がlive実行中に即座にストリームされます。
  - 直接モデルのHeartbeatは `OPENCLAW_LIVE_HEARTBEAT_MS` で調整します。
  - gateway/probeのHeartbeatは `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` で調整します。

## どのスイートを実行すべきか？

この判断表を使ってください。

- ロジック/テストを編集している: `pnpm test` を実行する（大きく変更したなら `pnpm test:coverage` も）
- gateway networking / WS protocol / pairingに触れている: `pnpm test:e2e` も追加する
- 「自分のbotが落ちている」/ プロバイダー固有の失敗 / tool callingをデバッグしている: 絞り込んだ `pnpm test:live` を実行する

## Live: Android Node capability sweep

- テスト: `src/gateway/android-node.capabilities.live.test.ts`
- スクリプト: `pnpm android:test:integration`
- 目的: 接続されたAndroid Nodeが現在通知している **すべてのコマンド** を呼び出し、コマンド契約の挙動を検証する。
- 範囲:
  - 前提条件付き/手動セットアップ（このスイートはアプリのインストール/実行/ペアリングを行いません）。
  - 選択したAndroid Nodeに対するコマンド単位のgateway `node.invoke` 検証。
- 必須の事前セットアップ:
  - Androidアプリがすでにgatewayに接続済みかつペアリング済みであること。
  - アプリがフォアグラウンドに維持されていること。
  - 成功を期待するcapabilityに必要な権限/キャプチャ同意が付与されていること。
- 任意の対象上書き:
  - `OPENCLAW_ANDROID_NODE_ID` または `OPENCLAW_ANDROID_NODE_NAME`。
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`。
- Androidの完全なセットアップ詳細: [Android App](/ja-JP/platforms/android)

## Live: model smoke（profile keys）

liveテストは、失敗を切り分けられるよう2つの層に分かれています。

- 「Direct model」は、そのキーでプロバイダー/モデルが少なくとも応答できることを示します。
- 「Gateway smoke」は、そのモデルに対してgateway+agentパイプライン全体（sessions、history、tools、sandbox policyなど）が機能することを示します。

### レイヤー1: 直接モデル完了（gatewayなし）

- テスト: `src/agents/models.profiles.live.test.ts`
- 目的:
  - 検出されたモデルを列挙する
  - `getApiKeyForModel` を使って、認証情報を持っているモデルを選択する
  - モデルごとに小さなcompletionを実行する（必要に応じて対象を絞ったリグレッションも実行する）
- 有効化方法:
  - `pnpm test:live`（またはVitestを直接呼び出す場合は `OPENCLAW_LIVE_TEST=1`）
- 実際にこのスイートを実行するには `OPENCLAW_LIVE_MODELS=modern`（または `all`、`modern` のエイリアス）を設定します。そうしない場合、このスイートはskipされ、`pnpm test:live` はgateway smokeに集中したままになります。
- モデルの選択方法:
  - `OPENCLAW_LIVE_MODELS=modern` でmodern allowlistを実行する（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_MODELS=all` はmodern allowlistのエイリアス
  - または `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."`（カンマ区切りallowlist）
  - modern/allスイープはデフォルトで厳選された高シグナル上限を使います。網羅的なmodernスイープには `OPENCLAW_LIVE_MAX_MODELS=0` を、より小さい上限には正の数を設定してください。
- プロバイダーの選択方法:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（カンマ区切りallowlist）
- キーの取得元:
  - デフォルト: profile storeとenvフォールバック
  - **profile store** のみを強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` を設定する
- これが存在する理由:
  - 「provider APIが壊れている / キーが無効である」と「gateway agentパイプラインが壊れている」を切り分けるため
  - 小さく分離されたリグレッションを収めるため（例: OpenAI Responses/Codex Responsesのreasoning replay + tool-callフロー）

### レイヤー2: Gateway + dev agent smoke（`@openclaw` が実際に行うこと）

- テスト: `src/gateway/gateway-models.profiles.live.test.ts`
- 目的:
  - プロセス内gatewayを起動する
  - `agent:dev:*` sessionを作成/patchする（実行ごとにモデル上書き）
  - キーを持つモデルを反復し、次を検証する:
    - 「意味のある」応答（toolsなし）
    - 実際のtool呼び出しが機能すること（read probe）
    - 任意の追加tool probe（exec+read probe）
    - OpenAIのリグレッションパス（tool-call-only → follow-up）が引き続き機能すること
- Probeの詳細（失敗を素早く説明できるようにするため）:
  - `read` probe: テストはworkspaceにnonceファイルを書き込み、agentにそれを `read` してnonceを返答でそのまま返すよう求めます。
  - `exec+read` probe: テストはagentに `exec` でnonceを一時ファイルに書かせ、その後それを `read` で読み返させます。
  - image probe: テストは生成したPNG（cat + ランダム化コード）を添付し、モデルが `cat <CODE>` を返すことを期待します。
  - 実装参照: `src/gateway/gateway-models.profiles.live.test.ts` および `src/gateway/live-image-probe.ts`。
- 有効化方法:
  - `pnpm test:live`（またはVitestを直接呼び出す場合は `OPENCLAW_LIVE_TEST=1`）
- モデルの選択方法:
  - デフォルト: modern allowlist（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` はmodern allowlistのエイリアス
  - または `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`（またはカンマ区切りリスト）で絞り込む
  - modern/allのgatewayスイープはデフォルトで厳選された高シグナル上限を使います。網羅的なmodernスイープには `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` を、より小さい上限には正の数を設定してください。
- プロバイダーの選択方法（「OpenRouterの全部」を避ける）:
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（カンマ区切りallowlist）
- Tool + image probeはこのliveテストでは常に有効です:
  - `read` probe + `exec+read` probe（toolストレス）
  - image入力サポートをモデルが通知している場合はimage probeが実行されます
  - フロー（高レベル）:
    - テストは「CAT」+ ランダムコードを含む小さなPNGを生成します（`src/gateway/live-image-probe.ts`）
    - それを `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]` 経由で送信します
    - Gatewayはattachmentを `images[]` に解析します（`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`）
    - embedded agentはmultimodal user messageをモデルに転送します
    - 検証: 返答に `cat` + そのコードが含まれること（OCRの許容: 軽微な誤りは可）

ヒント: 自分のマシンで何をテストできるか（および正確な `provider/model` id）を確認するには、次を実行してください。

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI backend smoke（Claude、Codex、Gemini、またはその他のローカルCLI）

- テスト: `src/gateway/gateway-cli-backend.live.test.ts`
- 目的: デフォルトconfigに触れずに、ローカルCLI backendを使ってGateway + agentパイプラインを検証する。
- backend固有のsmokeデフォルトは、所有するextensionの `cli-backend.ts` 定義にあります。
- 有効化:
  - `pnpm test:live`（またはVitestを直接呼び出す場合は `OPENCLAW_LIVE_TEST=1`）
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- デフォルト:
  - デフォルトのprovider/model: `claude-cli/claude-sonnet-4-6`
  - command/args/imageの挙動は、所有するCLI backend Plugin metadataから取得します。
- 上書き（任意）:
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - 実際のimage attachmentを送るには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`（パスはpromptに注入されます）。
  - prompt注入の代わりにimage file pathをCLI引数として渡すには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`。
  - `IMAGE_ARG` が設定されているときにimage引数の渡し方を制御するには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（または `"list"`）。
  - 2ターン目を送信してresumeフローを検証するには `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`。
  - デフォルトのClaude Sonnet -> Opus同一session継続probeを無効化するには `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`（選択したモデルがswitch targetをサポートしているときに強制的に有効化するには `1` を設定）。

例:

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Dockerレシピ:

```bash
pnpm test:docker:live-cli-backend
```

単一プロバイダーのDockerレシピ:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

メモ:

- Dockerランナーは `scripts/test-live-cli-backend-docker.sh` にあります。
- これは、リポジトリのDocker image内で、非rootの `node` ユーザーとしてlive CLI-backend smokeを実行します。
- 所有するextensionからCLI smoke metadataを解決し、その後、一致するLinux CLIパッケージ（`@anthropic-ai/claude-code`、`@openai/codex`、または `@google/gemini-cli`）を、キャッシュ可能で書き込み可能なprefix `OPENCLAW_DOCKER_CLI_TOOLS_DIR`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）にインストールします。
- `pnpm test:docker:live-cli-backend:claude-subscription` では、`~/.claude/.credentials.json` に `claudeAiOauth.subscriptionType` があるか、または `claude setup-token` 由来の `CLAUDE_CODE_OAUTH_TOKEN` を通じたポータブルClaude Code subscription OAuthが必要です。まずDocker内で直接 `claude -p` を検証し、その後、Anthropic API-key env varsを保持せずに2つのGateway CLI-backendターンを実行します。このsubscriptionレーンでは、Claudeが現在、通常のsubscription plan制限ではなく追加利用課金経由でサードパーティアプリ利用をルーティングするため、Claude MCP/toolおよびimage probeがデフォルトで無効になります。
- live CLI-backend smokeは現在、Claude、Codex、Geminiに対して同じエンドツーエンドフローを実行します: テキストターン、image classificationターン、その後gateway CLI経由で検証されるMCP `cron` tool呼び出し。
- Claudeのデフォルトsmokeでは、sessionをSonnetからOpusにpatchし、resumeしたsessionが以前のメモを引き続き覚えていることも検証します。

## Live: ACP bind smoke（`/acp spawn ... --bind here`）

- テスト: `src/gateway/gateway-acp-bind.live.test.ts`
- 目的: live ACP agentを使った実際のACP conversation-bindフローを検証する:
  - `/acp spawn <agent> --bind here` を送信する
  - 合成message-channel conversationをその場でbindする
  - 同じconversation上で通常のfollow-upを送信する
  - そのfollow-upがbind済みACP session transcriptに到達することを検証する
- 有効化:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- デフォルト:
  - Docker内のACP agent: `claude,codex,gemini`
  - 直接の `pnpm test:live ...` 用ACP agent: `claude`
  - 合成channel: Slack DM形式のconversation context
  - ACP backend: `acpx`
- 上書き:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- メモ:
  - このレーンは、admin専用の合成originating-routeフィールドを持つgateway `chat.send` surfaceを使うため、外部配信を装わずにmessage-channel contextをテストが付与できます。
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` が未設定の場合、テストは選択したACP harness agentに対して、組み込み `acpx` Pluginの内蔵agent registryを使用します。

例:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Dockerレシピ:

```bash
pnpm test:docker:live-acp-bind
```

単一agentのDockerレシピ:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Dockerメモ:

- Dockerランナーは `scripts/test-live-acp-bind-docker.sh` にあります。
- デフォルトでは、対応するすべてのlive CLI agentに対してACP bind smokeを順に実行します: `claude`、`codex`、次に `gemini`。
- マトリクスを絞るには `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`、または `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` を使用してください。
- これは `~/.profile` を読み込み、一致するCLI auth素材をコンテナに配置し、`acpx` を書き込み可能なnpm prefixにインストールしてから、必要なら要求されたlive CLI（`@anthropic-ai/claude-code`、`@openai/codex`、または `@google/gemini-cli`）をインストールします。
- Docker内では、ランナーは `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` を設定し、`acpx` が読み込んだprofile由来のprovider env varsを子harness CLIで使えるようにします。

## Live: Codex app-server harness smoke

- 目的: 通常のgateway
  `agent` メソッドを通じて、Plugin所有のCodex harnessを検証する:
  - 同梱の `codex` Pluginを読み込む
  - `OPENCLAW_AGENT_RUNTIME=codex` を選択する
  - `codex/gpt-5.4` に対して最初のgateway agentターンを送信する
  - 同じOpenClaw sessionに2ターン目を送信し、app-server
    threadがresumeできることを検証する
  - 同じgateway command
    パスを通して `/codex status` と `/codex models` を実行する
- テスト: `src/gateway/gateway-codex-harness.live.test.ts`
- 有効化: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- デフォルトモデル: `codex/gpt-5.4`
- 任意のimage probe: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- 任意のMCP/tool probe: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- このsmokeは `OPENCLAW_AGENT_HARNESS_FALLBACK=none` を設定するため、壊れたCodex
  harnessがPIへのサイレントフォールバックで通過することはできません。
- 認証: shell/profile由来の `OPENAI_API_KEY` と、任意でコピーされる
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

Dockerレシピ:

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

Dockerメモ:

- Dockerランナーは `scripts/test-live-codex-harness-docker.sh` にあります。
- これはマウントされた `~/.profile` を読み込み、`OPENAI_API_KEY` を渡し、存在する場合はCodex CLIの
  authファイルをコピーし、`@openai/codex` を書き込み可能でマウントされたnpm
  prefixにインストールし、source treeを配置した後、Codex-harness liveテストのみを実行します。
- DockerではimageおよびMCP/tool probeがデフォルトで有効です。より狭いデバッグ実行が必要な場合は
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` または
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` を設定してください。
- Dockerは `OPENCLAW_AGENT_HARNESS_FALLBACK=none` もエクスポートし、live
  テスト設定に合わせるため、`openai-codex/*` やPIへのフォールバックでCodex harness
  のリグレッションが隠れることはありません。

### 推奨liveレシピ

狭く明示的なallowlistが、最も高速で不安定さも最小です。

- 単一モデル、direct（gatewayなし）:
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- 単一モデル、gateway smoke:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 複数プロバイダーにまたがるtool calling:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google重視（Gemini APIキー + Antigravity）:
  - Gemini（APIキー）: `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）: `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

メモ:

- `google/...` はGemini API（APIキー）を使用します。
- `google-antigravity/...` はAntigravity OAuth bridge（Cloud Code Assist形式のagent endpoint）を使用します。
- `google-gemini-cli/...` はマシン上のローカルGemini CLIを使用します（別個のauth + toolingの癖があります）。
- Gemini APIとGemini CLI:
  - API: OpenClawはGoogleのホスト型Gemini APIをHTTP経由で呼び出します（APIキー / profile auth）。大半のユーザーが「Gemini」と言うときはこれを指します。
  - CLI: OpenClawはローカルの `gemini` バイナリをshell実行します。独自のauthがあり、挙動も異なることがあります（streaming/tool support/version skew）。

## Live: model matrix（何をカバーするか）

固定の「CI model list」はありません（liveはオプトイン）が、キーを持つ開発マシンで定期的にカバーすることを**推奨**するモデルは次のとおりです。

### Modern smoke set（tool calling + image）

これは、動作を維持することが期待される「一般的なモデル」実行です。

- OpenAI（非Codex）: `openai/gpt-5.4`（任意: `openai/gpt-5.4-mini`）
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6`（または `anthropic/claude-sonnet-4-6`）
- Google（Gemini API）: `google/gemini-3.1-pro-preview` および `google/gemini-3-flash-preview`（古いGemini 2.xモデルは避ける）
- Google（Antigravity）: `google-antigravity/claude-opus-4-6-thinking` および `google-antigravity/gemini-3-flash`
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

tools + image付きでgateway smokeを実行:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### ベースライン: tool calling（Read + 任意のExec）

少なくともプロバイダーファミリーごとに1つ選んでください。

- OpenAI: `openai/gpt-5.4`（または `openai/gpt-5.4-mini`）
- Anthropic: `anthropic/claude-opus-4-6`（または `anthropic/claude-sonnet-4-6`）
- Google: `google/gemini-3-flash-preview`（または `google/gemini-3.1-pro-preview`）
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

任意の追加カバレッジ（あると望ましい）:

- xAI: `xai/grok-4`（または利用可能な最新版）
- Mistral: `mistral/`…（有効化済みの「tools」対応モデルを1つ選ぶ）
- Cerebras: `cerebras/`…（アクセスがある場合）
- LM Studio: `lmstudio/`…（ローカル。tool callingはAPI modeに依存）

### Vision: image send（attachment → multimodal message）

image probeを実行するために、少なくとも1つのimage対応モデルを `OPENCLAW_LIVE_GATEWAY_MODELS` に含めてください（Claude/Gemini/OpenAIのvision対応バリアントなど）。

### Aggregators / alternate gateways

キーが有効であれば、次経由のテストもサポートしています。

- OpenRouter: `openrouter/...`（数百のモデル。tool+image対応候補を見つけるには `openclaw models scan` を使ってください）
- OpenCode: Zen向けの `opencode/...` とGo向けの `opencode-go/...`（authは `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`）

live matrixに含められる他のプロバイダー（認証情報/configがある場合）:

- 組み込み: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- `models.providers` 経由（custom endpoint）: `minimax`（cloud/API）、および任意のOpenAI/Anthropic互換proxy（LM Studio、vLLM、LiteLLMなど）

ヒント: ドキュメントに「すべてのモデル」をハードコードしようとしないでください。権威ある一覧は、あなたのマシン上で `discoverModels(...)` が返すものと、利用可能なキーの組み合わせです。

## 認証情報（絶対にコミットしない）

liveテストは、CLIと同じ方法で認証情報を検出します。実用上の意味は次のとおりです。

- CLIが動くなら、liveテストも同じキーを見つけられるはずです。
- liveテストが「認証情報なし」と言うなら、`openclaw models list` / モデル選択をデバッグするときと同じ方法でデバッグしてください。

- エージェントごとのauth profile: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（liveテストで「profile keys」と呼んでいるのはこれです）
- Config: `~/.openclaw/openclaw.json`（または `OPENCLAW_CONFIG_PATH`）
- レガシーstateディレクトリ: `~/.openclaw/credentials/`（存在する場合はstaged live homeにコピーされますが、メインのprofile-key storeではありません）
- ローカルlive実行は、デフォルトで有効なconfig、エージェントごとの `auth-profiles.json` ファイル、レガシー `credentials/`、およびサポートされている外部CLI authディレクトリを一時的なテストhomeにコピーします。staged live homeでは `workspace/` と `sandboxes/` はスキップされ、`agents.*.workspace` / `agentDir` のパス上書きは除去されるため、probeが実際のホストworkspaceに触れません。

envキー（たとえば `~/.profile` にexportされたもの）に依存したい場合は、`source ~/.profile` の後にローカルテストを実行するか、以下のDockerランナーを使ってください（これらは `~/.profile` をコンテナにマウントできます）。

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
- 範囲:
  - 同梱comfyのimage、video、および `music_generate` パスを検証する
  - `models.providers.comfy.<capability>` が設定されていない場合は各capabilityをskipする
  - comfy workflow送信、polling、download、またはPlugin登録を変更した後に有用

## Image generation live

- テスト: `src/image-generation/runtime.live.test.ts`
- コマンド: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness: `pnpm test:live:media image`
- 範囲:
  - 登録されているすべてのimage-generation provider Pluginを列挙する
  - probe前にlogin shell（`~/.profile`）から不足しているprovider env varsを読み込む
  - デフォルトでは保存済みauth profileよりもlive/env APIキーを優先して使うため、`auth-profiles.json` 内の古いテストキーが実際のshell認証情報を隠しません
  - 使用可能なauth/profile/modelがないプロバイダーはskipする
  - 共有runtime capabilityを通じて、標準のimage-generationバリアントを実行する:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- 現在カバーされている同梱プロバイダー:
  - `openai`
  - `google`
- 任意の絞り込み:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- 任意のauth挙動:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` でprofile-store authを強制し、envのみの上書きを無視する

## Music generation live

- テスト: `extensions/music-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media music`
- 範囲:
  - 共有の同梱music-generation providerパスを検証する
  - 現在はGoogleとMiniMaxをカバーする
  - probe前にlogin shell（`~/.profile`）からprovider env varsを読み込む
  - デフォルトでは保存済みauth profileよりもlive/env APIキーを優先して使うため、`auth-profiles.json` 内の古いテストキーが実際のshell認証情報を隠しません
  - 使用可能なauth/profile/modelがないプロバイダーはskipする
  - 利用可能な場合は、宣言された両方のruntime modeを実行する:
    - prompt-only入力による `generate`
    - プロバイダーが `capabilities.edit.enabled` を宣言している場合の `edit`
  - 現在の共有レーンカバレッジ:
    - `google`: `generate`、`edit`
    - `minimax`: `generate`
    - `comfy`: 別個のComfy liveファイルであり、この共有スイープではない
- 任意の絞り込み:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- 任意のauth挙動:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` でprofile-store authを強制し、envのみの上書きを無視する

## Video generation live

- テスト: `extensions/video-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness: `pnpm test:live:media video`
- 範囲:
  - 共有の同梱video-generation providerパスを検証する
  - デフォルトでは、release-safeなsmokeパスを使用する: FAL以外のプロバイダー、プロバイダーごとに1件のtext-to-videoリクエスト、1秒のlobster prompt、そして `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` 由来のプロバイダーごとの操作上限（デフォルト `180000`）
  - プロバイダー側のキュー待ち遅延がrelease時間を支配することがあるため、FALはデフォルトでskipされる。明示的に実行するには `--video-providers fal` または `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"` を指定する
  - probe前にlogin shell（`~/.profile`）からprovider env varsを読み込む
  - デフォルトでは保存済みauth profileよりもlive/env APIキーを優先して使うため、`auth-profiles.json` 内の古いテストキーが実際のshell認証情報を隠さない
  - 使用可能なauth/profile/modelがないプロバイダーはskipする
  - デフォルトでは `generate` のみを実行する
  - 利用可能な場合に宣言済みtransform modeも実行するには `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` を設定する:
    - プロバイダーが `capabilities.imageToVideo.enabled` を宣言し、選択したプロバイダー/モデルが共有スイープ内でbuffer-backedなローカルimage入力を受け付ける場合の `imageToVideo`
    - プロバイダーが `capabilities.videoToVideo.enabled` を宣言し、選択したプロバイダー/モデルが共有スイープ内でbuffer-backedなローカルvideo入力を受け付ける場合の `videoToVideo`
  - 共有スイープで現在は宣言済みだがskipされる `imageToVideo` プロバイダー:
    - `vydra`。同梱の `veo3` はtext-onlyであり、同梱の `kling` はリモートimage URLを必要とするため
  - プロバイダー固有のVydraカバレッジ:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - このファイルは、`veo3` のtext-to-videoと、デフォルトでリモートimage URL fixtureを使う `kling` レーンを実行する
  - 現在の `videoToVideo` liveカバレッジ:
    - 選択したモデルが `runway/gen4_aleph` のときのみ `runway`
  - 共有スイープで現在は宣言済みだがskipされる `videoToVideo` プロバイダー:
    - `alibaba`、`qwen`、`xai`。これらのパスは現在リモート `http(s)` / MP4 reference URLを必要とするため
    - `google`。現在の共有Gemini/Veoレーンはローカルのbuffer-backed入力を使っており、そのパスは共有スイープでは受け付けられないため
    - `openai`。現在の共有レーンには組織固有のvideo inpaint/remixアクセス保証がないため
- 任意の絞り込み:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - FALを含むデフォルトスイープ内のすべてのプロバイダーを含めるには `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""`
  - 攻めたsmoke実行のために各プロバイダーの操作上限を短くするには `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000`
- 任意のauth挙動:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` でprofile-store authを強制し、envのみの上書きを無視する

## Media live harness

- コマンド: `pnpm test:live:media`
- 目的:
  - 共有のimage、music、video liveスイートを、1つのリポジトリネイティブentrypoint経由で実行する
  - `~/.profile` から不足しているprovider env varsを自動読み込みする
  - デフォルトで、現在使用可能なauthを持つプロバイダーに各スイートを自動で絞り込む
  - `scripts/test-live.mjs` を再利用するため、Heartbeatおよびquiet-modeの挙動が一貫する
- 例:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Dockerランナー（任意の「Linuxで動く」チェック）

これらのDockerランナーは2つのバケットに分かれます。

- Live-modelランナー: `test:docker:live-models` と `test:docker:live-gateway` は、対応するprofile-key liveファイルのみをリポジトリDocker image内で実行します（`src/agents/models.profiles.live.test.ts` と `src/gateway/gateway-models.profiles.live.test.ts`）。ローカルconfigディレクトリとworkspaceをマウントし（マウントされていれば `~/.profile` も読み込みます）。対応するローカルentrypointは `test:live:models-profiles` と `test:live:gateway-profiles` です。
- Docker liveランナーは、完全なDockerスイープを現実的に保つため、デフォルトでより小さいsmoke上限を使います:
  `test:docker:live-models` はデフォルトで `OPENCLAW_LIVE_MAX_MODELS=12`、および
  `test:docker:live-gateway` はデフォルトで `OPENCLAW_LIVE_GATEWAY_SMOKE=1`、
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`、
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`、および
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` を使用します。より大きい網羅スキャンを明示的に行いたい場合は、これらのenv varを上書きしてください。
- `test:docker:all` は、まず `test:docker:live-build` 経由でlive Docker imageを一度ビルドし、その後2つのlive Dockerレーンでそれを再利用します。
- Container smokeランナー: `test:docker:openwebui`、`test:docker:onboard`、`test:docker:gateway-network`、`test:docker:mcp-channels`、および `test:docker:plugins` は、1つ以上の実コンテナを起動し、より高レベルのintegrationパスを検証します。

live-model Dockerランナーは、必要なCLI auth homeだけをbind-mountし（実行が絞り込まれていない場合はサポート対象すべて）、その後、外部CLI OAuthがホストauth storeを変更せずにトークンを更新できるよう、実行前にそれらをコンテナhomeにコピーします。

- Direct models: `pnpm test:docker:live-models`（スクリプト: `scripts/test-live-models-docker.sh`）
- ACP bind smoke: `pnpm test:docker:live-acp-bind`（スクリプト: `scripts/test-live-acp-bind-docker.sh`）
- CLI backend smoke: `pnpm test:docker:live-cli-backend`（スクリプト: `scripts/test-live-cli-backend-docker.sh`）
- Codex app-server harness smoke: `pnpm test:docker:live-codex-harness`（スクリプト: `scripts/test-live-codex-harness-docker.sh`）
- Gateway + dev agent: `pnpm test:docker:live-gateway`（スクリプト: `scripts/test-live-gateway-models-docker.sh`）
- Open WebUI live smoke: `pnpm test:docker:openwebui`（スクリプト: `scripts/e2e/openwebui-docker.sh`）
- オンボーディングウィザード（TTY、完全なscaffolding）: `pnpm test:docker:onboard`（スクリプト: `scripts/e2e/onboard-docker.sh`）
- Gateway networking（2コンテナ、WS auth + health）: `pnpm test:docker:gateway-network`（スクリプト: `scripts/e2e/gateway-network-docker.sh`）
- MCP channel bridge（seed済みGateway + stdio bridge + 生のClaude notification-frame smoke）: `pnpm test:docker:mcp-channels`（スクリプト: `scripts/e2e/mcp-channels-docker.sh`）
- Plugins（install smoke + `/plugin` エイリアス + Claude-bundle restart semantics）: `pnpm test:docker:plugins`（スクリプト: `scripts/e2e/plugins-docker.sh`）

live-model Dockerランナーは、現在のcheckoutもread-onlyでbind-mountし、
コンテナ内の一時workdirにそれを配置します。これによりruntime
imageをスリムに保ちつつ、正確にあなたのローカルsource/configに対してVitestを実行できます。
この配置ステップでは、大きなローカル専用キャッシュやアプリbuild出力、たとえば
`.pnpm-store`、`.worktrees`、`__openclaw_vitest__`、およびアプリローカルの `.build` や
Gradle出力ディレクトリをskipするため、Docker live実行が
マシン固有アーティファクトのコピーに何分も費やしません。
また、`OPENCLAW_SKIP_CHANNELS=1` も設定するため、gateway live probeが
コンテナ内で実際のTelegram/Discordなどのchannel workerを起動しません。
`test:docker:live-models` は引き続き `pnpm test:live` を実行するため、
そのDockerレーンでgateway liveカバレッジを絞り込んだり除外したりする必要がある場合は
`OPENCLAW_LIVE_GATEWAY_*` も渡してください。
`test:docker:openwebui` はより高レベルの互換smokeです。これは、
OpenAI互換HTTP endpointを有効にしたOpenClaw gatewayコンテナを起動し、
そのgatewayに対して固定版Open WebUIコンテナを起動し、
Open WebUI経由でサインインし、
`/api/models` が `openclaw/default` を公開していることを検証し、その後
Open WebUIの `/api/chat/completions` proxyを通して実際のchatリクエストを送信します。
初回実行は、Dockerが
Open WebUI imageをpullする必要がある場合や、Open WebUI自身のcold-start setupを完了する必要がある場合があるため、目に見えて遅くなることがあります。
このレーンは使用可能なlive modelキーを必要とし、Docker化された実行では
`OPENCLAW_PROFILE_FILE`
（デフォルト `~/.profile`）がそれを提供する主な方法です。
成功した実行では `{ "ok": true, "model":
"openclaw/default", ... }` のような小さなJSON payloadが出力されます。
`test:docker:mcp-channels` は意図的に決定的であり、実際の
Telegram、Discord、または iMessage アカウントを必要としません。これはseed済みGateway
コンテナを起動し、次に `openclaw mcp serve` を起動する2つ目のコンテナを開始し、
その後、実際のstdio MCP bridge上で、ルーティングされたconversation discovery、transcript読み取り、attachment metadata、
live event queueの挙動、outbound send routing、そしてClaude形式のchannel +
permission notificationを検証します。notificationチェックでは
生のstdio MCP frameを直接検査するため、このsmokeは
特定のclient SDKがたまたま表面化するものだけでなく、bridgeが実際に何を出力するかを検証します。

手動ACP平文thread smoke（CIではない）:

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- このスクリプトはリグレッション/デバッグワークフロー用に保持してください。ACP thread routing検証で再度必要になる可能性があるため、削除しないでください。

便利なenv var:

- `OPENCLAW_CONFIG_DIR=...`（デフォルト: `~/.openclaw`）を `/home/node/.openclaw` にマウント
- `OPENCLAW_WORKSPACE_DIR=...`（デフォルト: `~/.openclaw/workspace`）を `/home/node/.openclaw/workspace` にマウント
- `OPENCLAW_PROFILE_FILE=...`（デフォルト: `~/.profile`）を `/home/node/.profile` にマウントし、テスト実行前に読み込む
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1` で、`OPENCLAW_PROFILE_FILE` から読み込まれたenv varのみを検証し、一時的なconfig/workspaceディレクトリを使い、外部CLI authマウントは行わない
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）を `/home/node/.npm-global` にマウントし、Docker内のCLI installキャッシュに使う
- `$HOME` 配下の外部CLI authディレクトリ/ファイルは `/host-auth...` 配下にread-onlyでマウントされ、その後テスト開始前に `/home/node/...` にコピーされる
  - デフォルトディレクトリ: `.minimax`
  - デフォルトファイル: `~/.codex/auth.json`、`~/.codex/config.toml`、`.claude.json`、`~/.claude/.credentials.json`、`~/.claude/settings.json`、`~/.claude/settings.local.json`
  - 絞り込み済みプロバイダー実行では、`OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` から推定された必要なディレクトリ/ファイルのみをマウントする
  - 手動で上書きするには `OPENCLAW_DOCKER_AUTH_DIRS=all`、`OPENCLAW_DOCKER_AUTH_DIRS=none`、または `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` のようなカンマ区切りリストを使う
- 実行を絞るには `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- コンテナ内でプロバイダーを絞り込むには `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- rebuildが不要な再実行で既存の `openclaw:local-live` imageを再利用するには `OPENCLAW_SKIP_DOCKER_BUILD=1`
- 認証情報がprofile store由来であることを保証するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`（envではない）
- Open WebUI smoke向けにgatewayが公開するモデルを選ぶには `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI smokeで使うnonce-check promptを上書きするには `OPENCLAW_OPENWEBUI_PROMPT=...`
- 固定されたOpen WebUI image tagを上書きするには `OPENWEBUI_IMAGE=...`

## ドキュメントの健全性確認

ドキュメント編集後はdocsチェックを実行してください: `pnpm check:docs`。
ページ内見出しチェックも必要な場合は、完全なMintlify anchor検証を実行してください: `pnpm docs:check-links:anchors`。

## オフラインリグレッション（CI-safe）

これらは、実際のプロバイダーなしでの「実際のパイプライン」リグレッションです。

- Gateway tool calling（mock OpenAI、実際のgateway + agent loop）: `src/gateway/gateway.test.ts`（ケース: "runs a mock OpenAI tool call end-to-end via gateway agent loop"）
- Gatewayウィザード（WS `wizard.start`/`wizard.next`、config + auth enforcedを書き込む）: `src/gateway/gateway.test.ts`（ケース: "runs wizard over ws and writes auth token config"）

## Agent reliability evals（Skills）

すでに、次のようなCI-safeテストがいくつかあり、「agent reliability evals」のように振る舞います。

- 実際のgateway + agent loopを通したmock tool-calling（`src/gateway/gateway.test.ts`）。
- session wiringとconfig効果を検証するend-to-endウィザードフロー（`src/gateway/gateway.test.ts`）。

Skillsに関してまだ不足しているもの（[Skills](/ja-JP/tools/skills) を参照）:

- **Decisioning:** promptにSkillsが列挙されているとき、agentは適切なskillを選ぶか（または無関係なものを避けるか）？
- **Compliance:** agentは使用前に `SKILL.md` を読み、必要な手順/引数に従うか？
- **Workflow contracts:** tool順序、session履歴の引き継ぎ、sandbox境界を検証するマルチターンシナリオ。

将来のevalは、まず決定的であることを維持すべきです。

- mock providerを使ってtool呼び出し + 順序、skillファイル読み取り、session配線を検証するscenario runner。
- skillに焦点を当てた小規模シナリオスイート（使う vs 使わない、gating、prompt injection）。
- CI-safeスイートが整った後にのみ、任意のlive eval（オプトイン、env-gated）。

## 契約テスト（Pluginおよびchannelの形状）

契約テストは、登録されたすべてのPluginおよびchannelが、それぞれの
interface契約に準拠していることを検証します。検出されたすべてのPluginを反復し、
形状と振る舞いに関する一連の検証を実行します。デフォルトの `pnpm test` unitレーンは、
これらの共有seamおよびsmokeファイルを意図的にskipします。共有channelまたはprovider surfaceに触れた場合は、
契約コマンドを明示的に実行してください。

### コマンド

- すべての契約: `pnpm test:contracts`
- channel契約のみ: `pnpm test:contracts:channels`
- provider契約のみ: `pnpm test:contracts:plugins`

### Channel契約

`src/channels/plugins/contracts/*.contract.test.ts` にあります:

- **plugin** - 基本的なPlugin形状（id、name、capabilities）
- **setup** - セットアップウィザード契約
- **session-binding** - session bindingの挙動
- **outbound-payload** - message payload構造
- **inbound** - inbound message処理
- **actions** - channel action handler
- **threading** - thread ID処理
- **directory** - directory/roster API
- **group-policy** - group policyの強制

### Provider status契約

`src/plugins/contracts/*.contract.test.ts` にあります。

- **status** - channel status probe
- **registry** - Plugin registry形状

### Provider契約

`src/plugins/contracts/*.contract.test.ts` にあります:

- **auth** - auth flow契約
- **auth-choice** - auth choice/selection
- **catalog** - model catalog API
- **discovery** - Plugin discovery
- **loader** - Plugin loading
- **runtime** - provider runtime
- **shape** - Plugin shape/interface
- **wizard** - セットアップウィザード

### 実行するタイミング

- plugin-sdkのexportまたはsubpathを変更した後
- channelまたはprovider Pluginを追加または変更した後
- Plugin登録またはdiscoveryをリファクタリングした後

契約テストはCIで実行され、実際のAPIキーは不要です。

## リグレッションを追加する（ガイダンス）

liveで見つかったprovider/modelの問題を修正するとき:

- 可能ならCI-safeなリグレッションを追加する（mock/stub provider、または正確なrequest-shape変換をキャプチャする）
- 本質的にlive-onlyな場合（rate limit、auth policy）は、liveテストを狭く保ち、env var経由のオプトインにする
- バグを捕捉できる最小の層を対象にすることを優先する:
  - provider request conversion/replay bug → direct modelsテスト
  - gateway session/history/tool pipeline bug → gateway live smokeまたはCI-safeなgateway mockテスト
- SecretRef traversal guardrail:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` は、registry metadata（`listSecretTargetRegistryEntries()`）からSecretRefクラスごとに1つのサンプルtargetを導出し、その後、traversal-segment exec idが拒否されることを検証します。
  - `src/secrets/target-registry-data.ts` に新しい `includeInPlan` SecretRef targetファミリーを追加した場合は、そのテスト内の `classifyTargetClass` を更新してください。このテストは、未分類のtarget idに対して意図的に失敗するため、新しいクラスが黙ってskipされることはありません。
