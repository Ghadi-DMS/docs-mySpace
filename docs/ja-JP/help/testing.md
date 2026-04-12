---
read_when:
    - ローカルまたは CI でのテスト実行
    - モデル/provider のバグに対するリグレッションの追加
    - Gateway + エージェントの動作のデバッグ
summary: 'テストキット: unit/e2e/live スイート、Docker ランナー、および各テストがカバーする内容'
title: テスト
x-i18n:
    generated_at: "2026-04-12T23:28:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: a66ea672c386094ab4a8035a082c8a85d508a14301ad44b628d2a10d9cec3a52
    source_path: help/testing.md
    workflow: 15
---

# テスト

OpenClaw には 3 つの Vitest スイート（unit/integration、e2e、live）と、小規模な Docker ランナー群があります。

このドキュメントは「どのようにテストするか」のガイドです。

- 各スイートが何をカバーするか（そして意図的に _カバーしない_ ものは何か）
- 一般的なワークフロー（ローカル、push 前、デバッグ）でどのコマンドを実行するか
- live テストがどのように認証情報を検出し、model/provider を選択するか
- 実運用の model/provider 問題に対するリグレッションをどのように追加するか

## クイックスタート

ほとんどの日は次のとおりです。

- フルゲート（push 前に想定されるもの）: `pnpm build && pnpm check && pnpm test`
- 十分に余裕のあるマシンでの、より高速なローカル全スイート実行: `pnpm test:max`
- 直接の Vitest watch ループ: `pnpm test:watch`
- 直接のファイル指定は、extension/channel パスにも対応するようになりました: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- 単一の失敗を反復的に調整しているときは、まず対象を絞った実行を優先してください。
- Docker ベースの QA サイト: `pnpm qa:lab:up`
- Linux VM ベースの QA レーン: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

テストに手を入れたとき、または追加の確信が欲しいとき:

- カバレッジゲート: `pnpm test:coverage`
- E2E スイート: `pnpm test:e2e`

実際の provider/model をデバッグするとき（実際の認証情報が必要）:

- Live スイート（models + Gateway tool/image プローブ）: `pnpm test:live`
- 1 つの live ファイルを静かに対象指定: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

ヒント: 失敗している 1 ケースだけが必要な場合は、以下で説明する allowlist 環境変数で live テストを絞り込むことを優先してください。

## QA 専用ランナー

これらのコマンドは、qa-lab の現実性が必要なときに、主要なテストスイートと並んで使います。

- `pnpm openclaw qa suite`
  - リポジトリベースの QA シナリオをホスト上で直接実行します。
  - デフォルトでは、分離された
    Gateway ワーカーを使って複数の選択されたシナリオを並列実行します。最大
    64 ワーカーまたは選択されたシナリオ数までです。ワーカー数を調整するには
    `--concurrency <count>` を使い、従来の直列レーンにするには `--concurrency 1` を使います。
- `pnpm openclaw qa suite --runner multipass`
  - 同じ QA スイートを、使い捨ての Multipass Linux VM 内で実行します。
  - シナリオ選択の挙動はホスト上の `qa suite` と同じです。
  - `qa suite` と同じ provider/model 選択フラグを再利用します。
  - live 実行では、ゲストで実用的なサポート対象の QA 認証入力を転送します:
    環境変数ベースの provider キー、QA live provider 設定パス、
    および存在する場合は `CODEX_HOME`。
  - 出力ディレクトリは、ゲストがマウントされたワークスペース経由で書き戻せるように、
    リポジトリルート配下に置く必要があります。
  - 通常の QA レポート + サマリーに加えて、Multipass ログを
    `.artifacts/qa-e2e/...` 配下に書き込みます。
- `pnpm qa:lab:up`
  - オペレーター形式の QA 作業用に、Docker ベースの QA サイトを起動します。
- `pnpm openclaw qa matrix`
  - 使い捨ての Docker ベース Tuwunel homeserver に対して、Matrix live QA レーンを実行します。
  - 一時的な Matrix ユーザー 3 名（`driver`、`sut`、`observer`）と 1 つのプライベートルームをプロビジョニングし、その後、実際の Matrix Plugin を SUT トランスポートとして使う QA Gateway 子プロセスを起動します。
  - デフォルトでは、固定された安定版 Tuwunel イメージ `ghcr.io/matrix-construct/tuwunel:v1.5.1` を使用します。別のイメージをテストする必要がある場合は、`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` で上書きしてください。
  - Matrix QA レポート、サマリー、および観測イベント成果物を `.artifacts/qa-e2e/...` 配下に書き込みます。
- `pnpm openclaw qa telegram`
  - 環境変数から取得した driver と SUT の bot トークンを使って、実際のプライベートグループに対する Telegram live QA レーンを実行します。
  - `OPENCLAW_QA_TELEGRAM_GROUP_ID`、`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` が必要です。グループ ID は数値の Telegram chat ID である必要があります。
  - 同じプライベートグループ内に 2 つの異なる bot が必要であり、SUT bot は Telegram ユーザー名を公開している必要があります。
  - bot 間観測を安定させるために、両方の bot で `@BotFather` の Bot-to-Bot Communication Mode を有効にし、driver bot がグループ内の bot トラフィックを観測できるようにしてください。
  - Telegram QA レポート、サマリー、および観測メッセージ成果物を `.artifacts/qa-e2e/...` 配下に書き込みます。

live トランスポートレーンは、新しいトランスポートが逸脱しないよう、1 つの標準契約を共有しています。

`qa-channel` は依然として広範な合成 QA スイートであり、live
トランスポートのカバレッジマトリクスには含まれません。

| レーン | Canary | メンションゲーティング | allowlist ブロック | トップレベル返信 | 再起動再開 | スレッド follow-up | スレッド分離 | リアクション観測 | help コマンド |
| -------- | ------ | -------------- | --------------- | --------------- | -------------- | ---------------- | ---------------- | -------------------- | ------------ |
| Matrix   | x      | x              | x               | x               | x              | x                | x                | x                    |              |
| Telegram | x      |                |                 |                 |                |                  |                  |                      | x            |

### QA にチャネルを追加する

Markdown QA システムにチャネルを追加するには、正確に 2 つのものが必要です。

1. そのチャネル用のトランスポートアダプター
2. チャネル契約を実行するシナリオパック

共有の `qa-lab` ランナーでフローを担える場合は、チャネル固有の QA ランナーを追加しないでください。

`qa-lab` は共有メカニズムを担います。

- スイートの起動と終了処理
- ワーカーの並行性
- 成果物の書き込み
- レポート生成
- シナリオ実行
- 旧 `qa-channel` シナリオ向け互換エイリアス

チャネルアダプターはトランスポート契約を担います。

- そのトランスポート向けに Gateway をどう設定するか
- readiness をどう確認するか
- 受信イベントをどう注入するか
- 送信メッセージをどう観測するか
- transcript と正規化済みトランスポート状態をどう公開するか
- トランスポートバックのアクションをどう実行するか
- トランスポート固有のリセットやクリーンアップをどう扱うか

新しいチャネルに対する最低限の採用基準は次のとおりです。

1. 共有 `qa-lab` 境界上にトランスポートアダプターを実装する。
2. トランスポートレジストリにアダプターを登録する。
3. トランスポート固有のメカニズムはアダプターまたはチャネルハーネス内に閉じ込める。
4. `qa/scenarios/` 配下に Markdown シナリオを作成または適応する。
5. 新しいシナリオには汎用シナリオヘルパーを使用する。
6. リポジトリが意図的な移行を行っている場合を除き、既存の互換エイリアスを動作させ続ける。

判断ルールは厳格です。

- 振る舞いを `qa-lab` 内で一度だけ表現できるなら、`qa-lab` に置きます。
- 振る舞いが 1 つのチャネルトランスポートに依存するなら、そのアダプターまたは plugin ハーネス内に置きます。
- シナリオに複数のチャネルで使える新しい機能が必要なら、`suite.ts` にチャネル固有の分岐を追加するのではなく、汎用ヘルパーを追加します。
- 振る舞いが 1 つのトランスポートにしか意味を持たないなら、そのシナリオはトランスポート固有のままにし、それをシナリオ契約内で明示します。

新しいシナリオで推奨される汎用ヘルパー名は次のとおりです。

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

既存シナリオ向けには、以下を含む互換エイリアスを引き続き利用できます。

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

新しいチャネル作業では、汎用ヘルパー名を使うべきです。
互換エイリアスは、一斉移行を避けるために存在しているのであって、
新しいシナリオ作成のモデルではありません。

## テストスイート（何がどこで動くか）

各スイートは「現実性が増していくもの」（そして不安定さ/コストも増していくもの）として考えてください。

### Unit / integration（デフォルト）

- コマンド: `pnpm test`
- 設定: 既存のスコープ付き Vitest プロジェクトに対する、10 個の逐次シャード実行（`vitest.full-*.config.ts`）
- ファイル: `src/**/*.test.ts`、`packages/**/*.test.ts`、`test/**/*.test.ts` 配下の core/unit インベントリと、`vitest.unit.config.ts` でカバーされる許可済みの `ui` node テスト
- スコープ:
  - 純粋な unit テスト
  - プロセス内 integration テスト（Gateway 認証、ルーティング、ツーリング、パース、設定）
  - 既知のバグに対する決定論的リグレッション
- 想定:
  - CI で実行される
  - 実際のキーは不要
  - 高速かつ安定しているべき
- Projects に関する注記:
  - 対象未指定の `pnpm test` は、1 つの巨大なネイティブルートプロジェクトプロセスではなく、11 個のより小さなシャード設定（`core-unit-src`、`core-unit-security`、`core-unit-ui`、`core-unit-support`、`core-support-boundary`、`core-contracts`、`core-bundled`、`core-runtime`、`agentic`、`auto-reply`、`extensions`）を実行するようになりました。これにより、負荷の高いマシンでのピーク RSS を削減し、auto-reply/extension の処理が無関係なスイートを圧迫するのを防ぎます。
  - `pnpm test --watch` は、マルチシャード watch ループが現実的でないため、引き続きネイティブルートの `vitest.config.ts` プロジェクトグラフを使用します。
  - `pnpm test`、`pnpm test:watch`、`pnpm test:perf:imports` は、明示的なファイル/ディレクトリターゲットをまずスコープ付きレーン経由でルーティングするため、`pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` では完全なルートプロジェクト起動コストを払わずに済みます。
  - `pnpm test:changed` は、差分がルーティング可能な source/test ファイルだけに触れている場合、変更された git パスを同じスコープ付きレーンへ展開します。設定/セットアップ編集では、引き続き広範なルートプロジェクト再実行にフォールバックします。
  - agents、commands、plugins、auto-reply ヘルパー、`plugin-sdk`、および同様の純粋なユーティリティ領域の import が軽い unit テストは、`test/setup-openclaw-runtime.ts` をスキップする `unit-fast` レーンを通ります。状態を持つ/ランタイム負荷の高いファイルは既存のレーンに残ります。
  - 一部の `plugin-sdk` および `commands` ヘルパー source ファイルも、changed モード実行をそれらの軽量レーン内の明示的な兄弟テストへマッピングするため、ヘルパー編集でそのディレクトリの重い全スイートを再実行せずに済みます。
  - `auto-reply` には現在 3 つの専用バケットがあります: 最上位の core ヘルパー、最上位の `reply.*` integration テスト、および `src/auto-reply/reply/**` サブツリーです。これにより、最も重い reply ハーネス処理を、軽量な status/chunk/token テストから切り離しています。
- 埋め込みランナーに関する注記:
  - メッセージツールの検出入力または Compaction ランタイムコンテキストを変更する場合は、
    両方のレベルのカバレッジを維持してください。
  - 純粋なルーティング/正規化境界に対する、焦点を絞ったヘルパーリグレッションを追加してください。
  - また、埋め込みランナー integration スイートも健全に保ってください:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`、
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`、および
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`
  - これらのスイートは、スコープ付き ID と Compaction の挙動が引き続き
    実際の `run.ts` / `compact.ts` パスを通って流れることを検証します。ヘルパーのみのテストは、
    これらの integration パスの十分な代替にはなりません。
- プールに関する注記:
  - ベースの Vitest 設定は現在デフォルトで `threads` です。
  - 共有 Vitest 設定では `isolate: false` も固定されており、ルートプロジェクト、e2e、live 設定全体で非分離ランナーを使用します。
  - ルート UI レーンは `jsdom` セットアップと optimizer を維持しますが、現在は共有の非分離ランナー上でも動作します。
  - 各 `pnpm test` シャードは、共有 Vitest 設定から同じ `threads` + `isolate: false` のデフォルトを継承します。
  - 共有 `scripts/run-vitest.mjs` ランチャーは、大規模なローカル実行中の V8 コンパイル負荷を減らすために、Vitest 子 Node プロセスへデフォルトで `--no-maglev` も追加するようになりました。標準の V8 挙動と比較したい場合は `OPENCLAW_VITEST_ENABLE_MAGLEV=1` を設定してください。
- 高速なローカル反復に関する注記:
  - `pnpm test:changed` は、変更パスがより小さなスイートへきれいにマッピングされる場合、スコープ付きレーンを通ります。
  - `pnpm test:max` と `pnpm test:changed:max` は、同じルーティング挙動を維持しつつ、ワーカー上限だけを引き上げます。
  - ローカルのワーカー自動スケーリングは現在意図的に保守的であり、ホストのロードアベレージがすでに高い場合にも抑制するため、複数の同時 Vitest 実行による影響をデフォルトで小さくします。
  - ベースの Vitest 設定では、テスト配線が変わったときにも changed モードの再実行が正しく保たれるよう、projects/config ファイルを `forceRerunTriggers` としてマークしています。
  - 設定は、対応ホスト上では `OPENCLAW_VITEST_FS_MODULE_CACHE` を有効に保ちます。直接プロファイリング用に明示的なキャッシュ場所を 1 つ指定したい場合は、`OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` を設定してください。
- パフォーマンスデバッグに関する注記:
  - `pnpm test:perf:imports` は Vitest の import 時間レポートと import 内訳出力を有効にします。
  - `pnpm test:perf:imports:changed` は、`origin/main` 以降に変更されたファイルに同じプロファイリングビューを絞り込みます。
- `pnpm test:perf:changed:bench -- --ref <git-ref>` は、そのコミット済み差分に対して、ルーティングされた `test:changed` とネイティブルートプロジェクト経路を比較し、壁時計時間と macOS の最大 RSS を表示します。
- `pnpm test:perf:changed:bench -- --worktree` は、変更済みファイルリストを `scripts/test-projects.mjs` とルート Vitest 設定に通すことで、現在の dirty ツリーをベンチマークします。
  - `pnpm test:perf:profile:main` は、Vitest/Vite の起動および transform オーバーヘッドのメインスレッド CPU プロファイルを書き出します。
  - `pnpm test:perf:profile:runner` は、ファイル並列化を無効化した unit スイートに対して、ランナーの CPU+heap プロファイルを書き出します。

### E2E（Gateway スモーク）

- コマンド: `pnpm test:e2e`
- 設定: `vitest.e2e.config.ts`
- ファイル: `src/**/*.e2e.test.ts`、`test/**/*.e2e.test.ts`
- ランタイムデフォルト:
  - リポジトリの他の部分と同様に、Vitest の `threads` と `isolate: false` を使用します。
  - 適応型ワーカーを使用します（CI: 最大 2、ローカル: デフォルトで 1）。
  - コンソール I/O オーバーヘッドを減らすため、デフォルトで silent モードで実行します。
- 便利な上書き:
  - ワーカー数を強制するには `OPENCLAW_E2E_WORKERS=<n>`（上限 16）
  - 詳細なコンソール出力を再有効化するには `OPENCLAW_E2E_VERBOSE=1`
- スコープ:
  - 複数インスタンスの Gateway エンドツーエンド挙動
  - WebSocket/HTTP サーフェス、Node ペアリング、およびより重いネットワーキング
- 想定:
  - CI で実行される（パイプラインで有効な場合）
  - 実際のキーは不要
  - unit テストより可動部分が多い（遅くなる場合がある）

### E2E: OpenShell バックエンドスモーク

- コマンド: `pnpm test:e2e:openshell`
- ファイル: `test/openshell-sandbox.e2e.test.ts`
- スコープ:
  - Docker 経由でホスト上に分離された OpenShell Gateway を起動する
  - 一時的なローカル Dockerfile からサンドボックスを作成する
  - 実際の `sandbox ssh-config` + SSH exec を介して OpenClaw の OpenShell バックエンドを実行する
  - サンドボックス fs ブリッジ経由でリモート正規のファイルシステム挙動を検証する
- 想定:
  - オプトイン専用であり、デフォルトの `pnpm test:e2e` 実行には含まれない
  - ローカルの `openshell` CLI と動作する Docker デーモンが必要
  - 分離された `HOME` / `XDG_CONFIG_HOME` を使用し、その後テスト Gateway とサンドボックスを破棄する
- 便利な上書き:
  - より広い e2e スイートを手動で実行するときにこのテストを有効にするには `OPENCLAW_E2E_OPENSHELL=1`
  - デフォルト以外の CLI バイナリまたはラッパースクリプトを指定するには `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`

### Live（実際の provider + 実際の model）

- コマンド: `pnpm test:live`
- 設定: `vitest.live.config.ts`
- ファイル: `src/**/*.live.test.ts`
- デフォルト: `pnpm test:live` によって **有効**（`OPENCLAW_LIVE_TEST=1` を設定）
- スコープ:
  - 「この provider/model は、実際の認証情報で _今日_ 実際に動作するか？」
  - provider のフォーマット変更、ツールコールの癖、認証問題、レート制限の挙動を検出する
- 想定:
  - 設計上 CI 安定ではない（実ネットワーク、実際の provider ポリシー、クォータ、障害）
  - コストがかかる / レート制限を消費する
  - 「すべて」を実行するより、絞り込んだサブセットの実行を優先する
- Live 実行では、足りない API キーを取得するために `~/.profile` を source します。
- デフォルトでは、live 実行は依然として `HOME` を分離し、設定/認証情報を一時テストホームへコピーするため、unit fixture が実際の `~/.openclaw` を変更できません。
- live テストに意図的に実際のホームディレクトリを使わせる必要がある場合のみ、`OPENCLAW_LIVE_USE_REAL_HOME=1` を設定してください。
- `pnpm test:live` は現在、より静かなモードがデフォルトです: `[live] ...` の進行出力は維持しますが、追加の `~/.profile` 通知を抑制し、Gateway ブートストラップログ/Bonjour の雑音をミュートします。完全な起動ログを再表示したい場合は `OPENCLAW_LIVE_TEST_QUIET=0` を設定してください。
- API キーローテーション（provider 固有）: カンマ/セミコロン形式の `*_API_KEYS` または `*_API_KEY_1`、`*_API_KEY_2` を設定します（例: `OPENAI_API_KEYS`、`ANTHROPIC_API_KEYS`、`GEMINI_API_KEYS`）。または live 専用の上書きとして `OPENCLAW_LIVE_*_KEY` を使います。テストはレート制限応答でリトライします。
- 進行状況/Heartbeat 出力:
  - Live スイートは現在、長い provider 呼び出し中でも Vitest のコンソールキャプチャが静かなときに動作中であることが見えるよう、stderr に進行行を出力します。
  - `vitest.live.config.ts` は Vitest のコンソール介入を無効にしているため、provider/Gateway の進行行は live 実行中に即時ストリーミングされます。
  - 直接 model の Heartbeat は `OPENCLAW_LIVE_HEARTBEAT_MS` で調整します。
  - Gateway/プローブの Heartbeat は `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` で調整します。

## どのスイートを実行すべきか？

次の判断表を使ってください。

- ロジック/テストを編集している: `pnpm test` を実行する（大きく変更した場合は `pnpm test:coverage` も）
- Gateway ネットワーキング / WS プロトコル / ペアリングに触れる: `pnpm test:e2e` も追加する
- 「自分の bot が落ちている」/ provider 固有の失敗 / ツールコールをデバッグしている: 絞り込んだ `pnpm test:live` を実行する

## Live: Android Node capability sweep

- テスト: `src/gateway/android-node.capabilities.live.test.ts`
- スクリプト: `pnpm android:test:integration`
- 目的: 接続済み Android Node が現在公開している **すべてのコマンド** を呼び出し、コマンド契約の挙動を検証すること。
- スコープ:
  - 前提条件付き/手動セットアップ（このスイートはアプリのインストール/実行/ペアリングは行わない）
  - 選択した Android Node に対する、コマンド単位の Gateway `node.invoke` 検証
- 必須の事前セットアップ:
  - Android アプリがすでに Gateway に接続済みかつペアリング済みであること
  - アプリがフォアグラウンドに維持されていること
  - 通過させたい capability に必要な権限/キャプチャ同意が付与されていること
- 任意のターゲット上書き:
  - `OPENCLAW_ANDROID_NODE_ID` または `OPENCLAW_ANDROID_NODE_NAME`
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`
- Android の完全なセットアップ詳細: [Android App](/ja-JP/platforms/android)

## Live: model スモーク（profile keys）

live テストは、失敗を分離できるよう 2 層に分かれています。

- 「直接 model」は、与えられたキーで provider/model が少なくとも応答できるかを示します。
- 「Gateway スモーク」は、その model に対して完全な Gateway+エージェントパイプラインが動作するか（セッション、履歴、ツール、サンドボックスポリシーなど）を示します。

### レイヤー 1: 直接 model completion（Gateway なし）

- テスト: `src/agents/models.profiles.live.test.ts`
- 目的:
  - 検出された model を列挙する
  - `getApiKeyForModel` を使って、認証情報を持っている model を選択する
  - model ごとに小さな completion を実行する（必要に応じて対象を絞ったリグレッションも）
- 有効化方法:
  - `pnpm test:live`（または Vitest を直接呼ぶ場合は `OPENCLAW_LIVE_TEST=1`）
- 実際にこのスイートを実行するには `OPENCLAW_LIVE_MODELS=modern`（または `all`。modern の別名）を設定します。そうしない場合、このスイートは `pnpm test:live` の焦点を Gateway スモークに保つためスキップされます。
- model の選択方法:
  - modern allowlist を実行するには `OPENCLAW_LIVE_MODELS=modern`（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_MODELS=all` は modern allowlist の別名
  - または `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."`（カンマ区切り allowlist）
  - modern/all スイープはデフォルトで厳選された高シグナル上限を使用します。網羅的な modern スイープには `OPENCLAW_LIVE_MAX_MODELS=0`、より小さい上限には正の数を設定してください。
- provider の選択方法:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（カンマ区切り allowlist）
- キーの取得元:
  - デフォルト: profile ストアと環境変数フォールバック
  - **profile ストアのみ** を強制するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` を設定
- これが存在する理由:
  - 「provider API が壊れている / キーが無効」と「Gateway エージェントパイプラインが壊れている」を分離するため
  - 小さく分離されたリグレッションを含めるため（例: OpenAI Responses/Codex Responses の reasoning 再生 + ツールコールフロー）

### レイヤー 2: Gateway + dev エージェントスモーク（`@openclaw` が実際に行うこと）

- テスト: `src/gateway/gateway-models.profiles.live.test.ts`
- 目的:
  - プロセス内 Gateway を起動する
  - `agent:dev:*` セッションを作成/パッチする（実行ごとの model 上書き）
  - キー付きの models を反復し、次を検証する:
    - 「意味のある」応答（ツールなし）
    - 実際のツール呼び出しが機能すること（read プローブ）
    - 任意の追加ツールプローブ（exec+read プローブ）
    - OpenAI のリグレッション経路（ツール呼び出しのみ → follow-up）が引き続き機能すること
- プローブ詳細（失敗を素早く説明できるように）:
  - `read` プローブ: テストはワークスペースに nonce ファイルを書き込み、エージェントにそれを `read` して nonce をそのまま返すよう求めます。
  - `exec+read` プローブ: テストはエージェントに、temp ファイルへ nonce を `exec` で書き込み、その後それを `read` で読み戻すよう求めます。
  - image プローブ: テストは生成した PNG（cat + ランダム化コード）を添付し、model が `cat <CODE>` を返すことを期待します。
  - 実装参照: `src/gateway/gateway-models.profiles.live.test.ts` と `src/gateway/live-image-probe.ts`
- 有効化方法:
  - `pnpm test:live`（または Vitest を直接呼ぶ場合は `OPENCLAW_LIVE_TEST=1`）
- model の選択方法:
  - デフォルト: modern allowlist（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` は modern allowlist の別名
  - または絞り込むために `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`（またはカンマ区切りリスト）を設定
  - modern/all Gateway スイープはデフォルトで厳選された高シグナル上限を使用します。網羅的な modern スイープには `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`、より小さい上限には正の数を設定してください。
- provider の選択方法（「OpenRouter ですべて」を避ける）:
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（カンマ区切り allowlist）
- ツール + image プローブはこの live テストで常に有効です:
  - `read` プローブ + `exec+read` プローブ（ツールストレス）
  - model が image 入力サポートを公開している場合は image プローブも実行
  - フロー（概要）:
    - テストが「CAT」+ ランダムコード入りの小さな PNG を生成する（`src/gateway/live-image-probe.ts`）
    - `agent` の `attachments: [{ mimeType: "image/png", content: "<base64>" }]` 経由で送信する
    - Gateway が添付ファイルを `images[]` にパースする（`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`）
    - 埋め込みエージェントがマルチモーダルなユーザーメッセージを model に転送する
    - 検証: 返信に `cat` + そのコードが含まれること（OCR 許容: 軽微な誤りは可）

ヒント: 自分のマシンで何をテストできるか（および正確な `provider/model` ID）を見るには、次を実行してください。

```bash
openclaw models list
openclaw models list --json
```

## Live: CLI バックエンドスモーク（Claude、Codex、Gemini、またはその他のローカル CLI）

- テスト: `src/gateway/gateway-cli-backend.live.test.ts`
- 目的: デフォルト設定に触れずに、ローカル CLI バックエンドを使って Gateway + エージェントパイプラインを検証すること。
- バックエンド固有のスモークデフォルトは、所有する extension の `cli-backend.ts` 定義内にあります。
- 有効化:
  - `pnpm test:live`（または Vitest を直接呼ぶ場合は `OPENCLAW_LIVE_TEST=1`）
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- デフォルト:
  - デフォルト provider/model: `claude-cli/claude-sonnet-4-6`
  - command/args/image の挙動は、所有する CLI バックエンド Plugin メタデータから取得されます。
- 上書き（任意）:
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - 実際の image 添付を送信するには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`（パスはプロンプトに注入されます）
  - プロンプト注入ではなく image ファイルパスを CLI 引数として渡すには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`
  - `IMAGE_ARG` が設定されているときに image 引数の渡し方を制御するには `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（または `"list"`）
  - 2 回目のターンを送信して resume フローを検証するには `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`
  - デフォルトの Claude Sonnet -> Opus 同一セッション継続性プローブを無効化するには `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`（選択した model がスイッチターゲットをサポートしているときに強制的に有効化するには `1` を設定）

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

単一 provider の Docker レシピ:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

注記:

- Docker ランナーは `scripts/test-live-cli-backend-docker.sh` にあります。
- これは、live CLI-backend スモークをリポジトリ Docker イメージ内で非 root の `node` ユーザーとして実行します。
- 所有する extension から CLI スモークメタデータを解決し、その後、対応する Linux CLI パッケージ（`@anthropic-ai/claude-code`、`@openai/codex`、または `@google/gemini-cli`）を、`OPENCLAW_DOCKER_CLI_TOOLS_DIR`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）にあるキャッシュ可能で書き込み可能な prefix にインストールします。
- `pnpm test:docker:live-cli-backend:claude-subscription` では、`~/.claude/.credentials.json` 内の `claudeAiOauth.subscriptionType`、または `claude setup-token` の `CLAUDE_CODE_OAUTH_TOKEN` のいずれかによる、ポータブルな Claude Code サブスクリプション OAuth が必要です。まず Docker 内で直接の `claude -p` を確認し、その後 Anthropic API キー環境変数を保持せずに 2 回の Gateway CLI-backend ターンを実行します。このサブスクリプションレーンでは、Claude が現在、サードパーティーアプリ利用を通常のサブスクリプションプラン制限ではなく追加利用課金経由でルーティングするため、Claude MCP/ツールおよび image プローブはデフォルトで無効化されています。
- live CLI-backend スモークは現在、Claude、Codex、Gemini に対して同じエンドツーエンドフローを実行します: テキストターン、image 分類ターン、その後 Gateway CLI を通じて検証される MCP `cron` ツールコール。
- Claude のデフォルトスモークでは、セッションを Sonnet から Opus にパッチし、resume したセッションが以前のメモをまだ記憶していることも検証します。

## Live: ACP bind スモーク（`/acp spawn ... --bind here`）

- テスト: `src/gateway/gateway-acp-bind.live.test.ts`
- 目的: live ACP エージェントで実際の ACP conversation-bind フローを検証すること:
  - `/acp spawn <agent> --bind here` を送信する
  - 合成メッセージチャネル会話をその場で bind する
  - 同じ会話上で通常の follow-up を送信する
  - その follow-up が bind 済み ACP セッショントランスクリプトに到達することを検証する
- 有効化:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- デフォルト:
  - Docker 内の ACP エージェント: `claude,codex,gemini`
  - 直接 `pnpm test:live ...` 用の ACP エージェント: `claude`
  - 合成チャネル: Slack DM 形式の会話コンテキスト
  - ACP バックエンド: `acpx`
- 上書き:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- 注記:
  - このレーンは、管理者専用の合成 originating-route フィールド付きの Gateway `chat.send` サーフェスを使うため、テストは外部配信を装うことなくメッセージチャネルコンテキストを付与できます。
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` が未設定の場合、テストは選択した ACP ハーネスエージェントに対して、埋め込み `acpx` Plugin の組み込みエージェントレジストリを使用します。

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
- デフォルトでは、サポートされるすべての live CLI エージェント `claude`、`codex`、`gemini` に対して、ACP bind スモークを順番に実行します。
- マトリクスを絞り込むには、`OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`、または `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` を使います。
- これは `~/.profile` を source し、一致する CLI 認証情報をコンテナにステージし、`acpx` を書き込み可能な npm prefix にインストールし、その後、必要なら要求された live CLI（`@anthropic-ai/claude-code`、`@openai/codex`、または `@google/gemini-cli`）をインストールします。
- Docker 内では、ランナーは `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` を設定するため、acpx は source 済み profile からの provider 環境変数を子ハーネス CLI で利用可能なままにできます。

## Live: Codex app-server ハーネススモーク

- 目的: 通常の Gateway
  `agent` メソッドを通じて Plugin 所有の Codex ハーネスを検証すること:
  - バンドルされた `codex` Plugin を読み込む
  - `OPENCLAW_AGENT_RUNTIME=codex` を選択する
  - `codex/gpt-5.4` に対して最初の Gateway エージェントターンを送信する
  - 同じ OpenClaw セッションへ 2 回目のターンを送信し、app-server
    スレッドが resume できることを検証する
  - 同じ Gateway コマンド
    パスを通じて `/codex status` と `/codex models` を実行する
- テスト: `src/gateway/gateway-codex-harness.live.test.ts`
- 有効化: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- デフォルト model: `codex/gpt-5.4`
- 任意の image プローブ: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- 任意の MCP/ツールプローブ: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- このスモークでは `OPENCLAW_AGENT_HARNESS_FALLBACK=none` を設定するため、壊れた Codex
  ハーネスが PI へのサイレントなフォールバックで通過することはできません。
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

Docker に関する注記:

- Docker ランナーは `scripts/test-live-codex-harness-docker.sh` にあります。
- これは、マウントされた `~/.profile` を source し、`OPENAI_API_KEY` を渡し、存在する場合は Codex CLI
  認証ファイルをコピーし、`@openai/codex` を書き込み可能なマウント済み npm
  prefix にインストールし、ソースツリーをステージした後、Codex-harness live テストのみを実行します。
- Docker ではデフォルトで image および MCP/ツールプローブを有効にします。より狭いデバッグ実行が必要な場合は、
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` または
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` を設定してください。
- Docker でも `OPENCLAW_AGENT_HARNESS_FALLBACK=none` をエクスポートし、live
  テスト設定と一致させるため、`openai-codex/*` や PI フォールバックが Codex ハーネス
  リグレッションを隠すことはできません。

### 推奨 live レシピ

狭く明示的な allowlist が最も高速で、最も不安定さが少ないです。

- 単一 model、直接（Gateway なし）:
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- 単一 model、Gateway スモーク:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 複数 provider にまたがるツールコール:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google フォーカス（Gemini API キー + Antigravity）:
  - Gemini（API キー）: `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）: `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

注記:

- `google/...` は Gemini API（API キー）を使用します。
- `google-antigravity/...` は Antigravity OAuth ブリッジ（Cloud Code Assist 形式のエージェントエンドポイント）を使用します。
- `google-gemini-cli/...` はマシン上のローカル Gemini CLI を使用します（認証とツーリングの癖が別です）。
- Gemini API と Gemini CLI の違い:
  - API: OpenClaw は Google がホストする Gemini API を HTTP 経由で呼び出します（API キー / profile 認証）。ほとんどのユーザーが「Gemini」と言うときは、これを指します。
  - CLI: OpenClaw はローカルの `gemini` バイナリをシェル実行します。独自の認証があり、挙動も異なる場合があります（ストリーミング/ツールサポート/バージョン差異）。

## Live: model マトリクス（何をカバーしているか）

固定の「CI model リスト」はありません（live はオプトイン）が、キーを持つ開発マシンで定期的にカバーする **推奨** model は次のとおりです。

### Modern スモークセット（ツールコール + image）

これは、動作し続けることを期待する「共通 model」実行です。

- OpenAI（非 Codex）: `openai/gpt-5.4`（任意: `openai/gpt-5.4-mini`）
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6`（または `anthropic/claude-sonnet-4-6`）
- Google（Gemini API）: `google/gemini-3.1-pro-preview` と `google/gemini-3-flash-preview`（古い Gemini 2.x model は避けてください）
- Google（Antigravity）: `google-antigravity/claude-opus-4-6-thinking` と `google-antigravity/gemini-3-flash`
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

ツール + image 付きで Gateway スモークを実行:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### ベースライン: ツールコール（Read + 任意の Exec）

provider ファミリーごとに少なくとも 1 つを選んでください。

- OpenAI: `openai/gpt-5.4`（または `openai/gpt-5.4-mini`）
- Anthropic: `anthropic/claude-opus-4-6`（または `anthropic/claude-sonnet-4-6`）
- Google: `google/gemini-3-flash-preview`（または `google/gemini-3.1-pro-preview`）
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

任意の追加カバレッジ（あると望ましい）:

- xAI: `xai/grok-4`（または利用可能な最新版）
- Mistral: `mistral/`…（有効化されている「tools」対応 model を 1 つ選ぶ）
- Cerebras: `cerebras/`…（アクセス権がある場合）
- LM Studio: `lmstudio/`…（ローカル。ツールコールは API モードに依存）

### Vision: image 送信（添付 → マルチモーダルメッセージ）

image プローブを実行するため、少なくとも 1 つの image 対応 model を `OPENCLAW_LIVE_GATEWAY_MODELS` に含めてください（Claude/Gemini/OpenAI の vision 対応バリアントなど）。

### アグリゲーター / 代替 Gateway

キーが有効であれば、次経由のテストもサポートしています。

- OpenRouter: `openrouter/...`（数百の model。ツール + image 対応候補を見つけるには `openclaw models scan` を使ってください）
- OpenCode: Zen 用の `opencode/...` と Go 用の `opencode-go/...`（認証は `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`）

live マトリクスに含められるその他の provider（認証情報/設定がある場合）:

- 組み込み: `openai`、`openai-codex`、`anthropic`、`google`、`google-vertex`、`google-antigravity`、`google-gemini-cli`、`zai`、`openrouter`、`opencode`、`opencode-go`、`xai`、`groq`、`cerebras`、`mistral`、`github-copilot`
- `models.providers` 経由（カスタムエンドポイント）: `minimax`（クラウド/API）、および任意の OpenAI/Anthropic 互換プロキシ（LM Studio、vLLM、LiteLLM など）

ヒント: ドキュメントに「すべての model」をハードコードしようとしないでください。信頼できる正規のリストは、マシン上で `discoverModels(...)` が返すものと、利用可能なキーの組み合わせです。

## 認証情報（絶対にコミットしない）

live テストは、CLI と同じ方法で認証情報を検出します。実際上の意味は次のとおりです。

- CLI が動くなら、live テストでも同じキーを見つけられるはずです。
- live テストが「認証情報なし」と言うなら、`openclaw models list` / model 選択をデバッグするときと同じ方法で調査してください。

- エージェントごとの認証 profile: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（live テストで「profile keys」と言うのはこれのことです）
- 設定: `~/.openclaw/openclaw.json`（または `OPENCLAW_CONFIG_PATH`）
- レガシー state ディレクトリ: `~/.openclaw/credentials/`（存在する場合はステージ済み live ホームへコピーされますが、メインの profile-key ストアではありません）
- ローカルの live 実行では、デフォルトでアクティブ設定、エージェントごとの `auth-profiles.json` ファイル、レガシー `credentials/`、およびサポートされる外部 CLI 認証ディレクトリを一時テストホームへコピーします。ステージ済み live ホームでは `workspace/` と `sandboxes/` はスキップされ、`agents.*.workspace` / `agentDir` のパス上書きも取り除かれるため、プローブが実際のホストワークスペースに触れません。

環境変数キー（たとえば `~/.profile` に export したもの）に依存したい場合は、`source ~/.profile` の後にローカルテストを実行するか、以下の Docker ランナーを使ってください（これらは `~/.profile` をコンテナにマウントできます）。

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
  - comfy workflow の送信、ポーリング、ダウンロード、または Plugin 登録を変更した後に有用

## 画像生成 live

- テスト: `src/image-generation/runtime.live.test.ts`
- コマンド: `pnpm test:live src/image-generation/runtime.live.test.ts`
- ハーネス: `pnpm test:live:media image`
- スコープ:
  - 登録されているすべての画像生成 provider Plugin を列挙する
  - プローブ前に、ログインシェル（`~/.profile`）から不足している provider 環境変数を読み込む
  - デフォルトでは、保存済み認証 profile より live/env API キーを優先するため、`auth-profiles.json` の古いテストキーが実際のシェル認証情報を隠さない
  - 使用可能な認証/profile/model がない provider はスキップする
  - 標準の画像生成バリアントを共有ランタイム capability 経由で実行する:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- 現在カバーされているバンドル済み provider:
  - `openai`
  - `google`
- 任意の絞り込み:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- 任意の認証挙動:
  - profile ストア認証を強制し、env-only 上書きを無視するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## 音楽生成 live

- テスト: `extensions/music-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- ハーネス: `pnpm test:live:media music`
- スコープ:
  - 共有のバンドル済み音楽生成 provider パスを実行する
  - 現在は Google と MiniMax をカバーする
  - プローブ前に、ログインシェル（`~/.profile`）から provider 環境変数を読み込む
  - デフォルトでは、保存済み認証 profile より live/env API キーを優先するため、`auth-profiles.json` の古いテストキーが実際のシェル認証情報を隠さない
  - 使用可能な認証/profile/model がない provider はスキップする
  - 利用可能な場合は、宣言された両方のランタイムモードを実行する:
    - プロンプトのみ入力の `generate`
    - provider が `capabilities.edit.enabled` を宣言している場合の `edit`
  - 現在の共有レーンのカバレッジ:
    - `google`: `generate`、`edit`
    - `minimax`: `generate`
    - `comfy`: 別の Comfy live ファイルであり、この共有スイープではない
- 任意の絞り込み:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- 任意の認証挙動:
  - profile ストア認証を強制し、env-only 上書きを無視するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## 動画生成 live

- テスト: `extensions/video-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- ハーネス: `pnpm test:live:media video`
- スコープ:
  - 共有のバンドル済み動画生成 provider パスを実行する
  - プローブ前に、ログインシェル（`~/.profile`）から provider 環境変数を読み込む
  - デフォルトでは、保存済み認証 profile より live/env API キーを優先するため、`auth-profiles.json` の古いテストキーが実際のシェル認証情報を隠さない
  - 使用可能な認証/profile/model がない provider はスキップする
  - 利用可能な場合は、宣言された両方のランタイムモードを実行する:
    - プロンプトのみ入力の `generate`
    - provider が `capabilities.imageToVideo.enabled` を宣言しており、選択した provider/model が共有スイープでバッファバックのローカル image 入力を受け付ける場合の `imageToVideo`
    - provider が `capabilities.videoToVideo.enabled` を宣言しており、選択した provider/model が共有スイープでバッファバックのローカル video 入力を受け付ける場合の `videoToVideo`
  - 共有スイープで現在は宣言されているがスキップされる `imageToVideo` provider:
    - バンドル済み `veo3` はテキスト専用で、バンドル済み `kling` はリモート image URL を必要とするため、`vydra`
  - provider 固有の Vydra カバレッジ:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - そのファイルは、`veo3` の text-to-video と、デフォルトでリモート image URL fixture を使う `kling` レーンを実行する
  - 現在の `videoToVideo` live カバレッジ:
    - 選択した model が `runway/gen4_aleph` の場合のみ `runway`
  - 共有スイープで現在は宣言されているがスキップされる `videoToVideo` provider:
    - `alibaba`、`qwen`、`xai` は、これらのパスが現在リモート `http(s)` / MP4 参照 URL を必要とするため
    - `google` は、現在の共有 Gemini/Veo レーンがローカルのバッファバック入力を使っており、そのパスは共有スイープで受け付けられないため
    - `openai` は、現在の共有レーンに org 固有の video inpaint/remix アクセス保証がないため
- 任意の絞り込み:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- 任意の認証挙動:
  - profile ストア認証を強制し、env-only 上書きを無視するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## メディア live ハーネス

- コマンド: `pnpm test:live:media`
- 目的:
  - 共有の image、music、video の live スイートを、リポジトリ標準の 1 つのエントリーポイントで実行する
  - `~/.profile` から不足している provider 環境変数を自動読み込みする
  - デフォルトでは、現在使用可能な認証を持つ provider に各スイートを自動的に絞り込む
  - `scripts/test-live.mjs` を再利用するため、Heartbeat と quiet モードの挙動が一貫する
- 例:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Docker ランナー（任意の「Linux で動く」確認）

これらの Docker ランナーは 2 つのカテゴリに分かれます:

- live-model ランナー: `test:docker:live-models` と `test:docker:live-gateway` は、対応する profile-key live ファイルだけをリポジトリ Docker イメージ内で実行します（`src/agents/models.profiles.live.test.ts` と `src/gateway/gateway-models.profiles.live.test.ts`）。ローカルの設定ディレクトリとワークスペースをマウントし（マウントされていれば `~/.profile` も source します）、実行します。対応するローカルエントリーポイントは `test:live:models-profiles` と `test:live:gateway-profiles` です。
- Docker live ランナーは、完全な Docker スイープを実用的に保つため、デフォルトでより小さなスモーク上限を使用します:
  `test:docker:live-models` はデフォルトで `OPENCLAW_LIVE_MAX_MODELS=12` を使用し、
  `test:docker:live-gateway` はデフォルトで `OPENCLAW_LIVE_GATEWAY_SMOKE=1`、
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`、
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`、および
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` を使用します。より大きな網羅的スキャンを
  明示的に行いたい場合は、これらの環境変数を上書きしてください。
- `test:docker:all` は、まず `test:docker:live-build` で live Docker イメージを一度だけビルドし、その後そのイメージを 2 つの live Docker レーンで再利用します。
- コンテナスモークランナー: `test:docker:openwebui`、`test:docker:onboard`、`test:docker:gateway-network`、`test:docker:mcp-channels`、`test:docker:plugins` は、1 つまたは複数の実コンテナを起動し、より高レベルの integration パスを検証します。

live-model Docker ランナーは、必要な CLI 認証ホームだけを bind mount し（または実行が絞り込まれていない場合はサポート対象すべてを bind mount し）、実行前にそれらをコンテナホームへコピーするため、外部 CLI OAuth がホスト認証ストアを変更せずにトークンを更新できます。

- 直接 model: `pnpm test:docker:live-models`（スクリプト: `scripts/test-live-models-docker.sh`）
- ACP bind スモーク: `pnpm test:docker:live-acp-bind`（スクリプト: `scripts/test-live-acp-bind-docker.sh`）
- CLI バックエンドスモーク: `pnpm test:docker:live-cli-backend`（スクリプト: `scripts/test-live-cli-backend-docker.sh`）
- Codex app-server ハーネススモーク: `pnpm test:docker:live-codex-harness`（スクリプト: `scripts/test-live-codex-harness-docker.sh`）
- Gateway + dev エージェント: `pnpm test:docker:live-gateway`（スクリプト: `scripts/test-live-gateway-models-docker.sh`）
- Open WebUI live スモーク: `pnpm test:docker:openwebui`（スクリプト: `scripts/e2e/openwebui-docker.sh`）
- オンボーディング ウィザード（TTY、完全な scaffolding）: `pnpm test:docker:onboard`（スクリプト: `scripts/e2e/onboard-docker.sh`）
- Gateway ネットワーキング（2 つのコンテナ、WS 認証 + ヘルス）: `pnpm test:docker:gateway-network`（スクリプト: `scripts/e2e/gateway-network-docker.sh`）
- MCP チャネルブリッジ（seed 済み Gateway + stdio ブリッジ + 生の Claude 通知フレームスモーク）: `pnpm test:docker:mcp-channels`（スクリプト: `scripts/e2e/mcp-channels-docker.sh`）
- Plugins（インストールスモーク + `/plugin` エイリアス + Claude バンドル再起動セマンティクス）: `pnpm test:docker:plugins`（スクリプト: `scripts/e2e/plugins-docker.sh`）

live-model Docker ランナーは、現在の checkout も読み取り専用で bind mount し、
その中身をコンテナ内の一時 workdir にステージします。これにより、ランタイム
イメージをスリムに保ちながらも、正確にローカルの source/config に対して Vitest を実行できます。
このステージング手順では、大きなローカル専用キャッシュやアプリのビルド出力、たとえば
`.pnpm-store`、`.worktrees`、`__openclaw_vitest__`、およびアプリローカルの `.build` や
Gradle 出力ディレクトリをスキップするため、Docker live 実行で
マシン固有の成果物を何分もかけてコピーせずに済みます。
また、`OPENCLAW_SKIP_CHANNELS=1` も設定するため、Gateway live プローブが
コンテナ内で実際の Telegram/Discord などのチャネルワーカーを起動しません。
`test:docker:live-models` は引き続き `pnpm test:live` を実行するため、
その Docker レーンで Gateway live カバレッジを絞り込んだり除外したりする必要がある場合は、
`OPENCLAW_LIVE_GATEWAY_*` も渡してください。
`test:docker:openwebui` は、より高レベルの互換性スモークです。これは
OpenAI 互換 HTTP エンドポイントを有効にした OpenClaw Gateway コンテナを起動し、
その Gateway に対して固定版の Open WebUI コンテナを起動し、
Open WebUI 経由でサインインし、`/api/models` が `openclaw/default` を公開していることを検証し、その後
Open WebUI の `/api/chat/completions` プロキシを通して
実際のチャットリクエストを送信します。
初回実行は目に見えて遅くなる場合があります。これは Docker が
Open WebUI イメージを pull する必要があったり、Open WebUI 自身のコールドスタート設定完了が必要だったりするためです。
このレーンでは使用可能な live model キーを想定しており、Docker 化実行でそれを提供する主要な方法は
`OPENCLAW_PROFILE_FILE`（デフォルトは `~/.profile`）です。
成功した実行では `{ "ok": true, "model":
"openclaw/default", ... }` のような小さな JSON ペイロードが出力されます。
`test:docker:mcp-channels` は意図的に決定論的であり、実際の
Telegram、Discord、または iMessage アカウントを必要としません。これは seed 済み Gateway
コンテナを起動し、`openclaw mcp serve` を起動する 2 つ目のコンテナを開始し、その後
ルーティングされた会話検出、トランスクリプト読み取り、添付メタデータ、
live イベントキュー挙動、送信ルーティング、および Claude 形式のチャネル +
権限通知を、実際の stdio MCP ブリッジ越しに検証します。通知チェックは
生の stdio MCP フレームを直接検査するため、このスモークは
特定のクライアント SDK がたまたま露出するものではなく、ブリッジが実際に何を発行するかを検証します。

手動 ACP 平文スレッドスモーク（CI ではない）:

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- このスクリプトはリグレッション/デバッグワークフロー用に保持してください。ACP スレッドルーティング検証で再び必要になる可能性があるため、削除しないでください。

便利な環境変数:

- `OPENCLAW_CONFIG_DIR=...`（デフォルト: `~/.openclaw`）は `/home/node/.openclaw` にマウントされます
- `OPENCLAW_WORKSPACE_DIR=...`（デフォルト: `~/.openclaw/workspace`）は `/home/node/.openclaw/workspace` にマウントされます
- `OPENCLAW_PROFILE_FILE=...`（デフォルト: `~/.profile`）は `/home/node/.profile` にマウントされ、テスト実行前に source されます
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）は Docker 内のキャッシュされた CLI インストール用に `/home/node/.npm-global` にマウントされます
- `$HOME` 配下の外部 CLI 認証ディレクトリ/ファイルは、`/host-auth...` 配下に読み取り専用でマウントされ、その後テスト開始前に `/home/node/...` へコピーされます
  - デフォルトのディレクトリ: `.minimax`
  - デフォルトのファイル: `~/.codex/auth.json`、`~/.codex/config.toml`、`.claude.json`、`~/.claude/.credentials.json`、`~/.claude/settings.json`、`~/.claude/settings.local.json`
  - 絞り込まれた provider 実行では、`OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` から推定された必要なディレクトリ/ファイルだけをマウントします
  - 手動で上書きするには `OPENCLAW_DOCKER_AUTH_DIRS=all`、`OPENCLAW_DOCKER_AUTH_DIRS=none`、または `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` のようなカンマ区切りリストを使います
- 実行を絞り込むには `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- コンテナ内で provider を絞り込むには `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- リビルド不要の再実行で既存の `openclaw:local-live` イメージを再利用するには `OPENCLAW_SKIP_DOCKER_BUILD=1`
- 認証情報を profile ストア由来に限定するには `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`（env 由来は使わない）
- Open WebUI スモークで Gateway が公開する model を選ぶには `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI スモークで使う nonce 確認プロンプトを上書きするには `OPENCLAW_OPENWEBUI_PROMPT=...`
- 固定版 Open WebUI イメージタグを上書きするには `OPENWEBUI_IMAGE=...`

## ドキュメントの健全性確認

ドキュメントを編集した後は docs チェックを実行してください: `pnpm check:docs`。
ページ内見出しチェックも含めた完全な Mintlify アンカー検証が必要な場合は、`pnpm docs:check-links:anchors` を実行してください。

## オフラインリグレッション（CI 安全）

これらは、実際の provider を使わない「実パイプライン」リグレッションです。

- Gateway ツールコール（OpenAI をモック、実際の Gateway + エージェントループ）: `src/gateway/gateway.test.ts`（ケース: 「runs a mock OpenAI tool call end-to-end via gateway agent loop」）
- Gateway ウィザード（WS `wizard.start`/`wizard.next`、設定書き込み + 認証強制あり）: `src/gateway/gateway.test.ts`（ケース: 「runs wizard over ws and writes auth token config」）

## エージェント信頼性 eval（Skills）

すでにいくつか、CI 安全で「エージェント信頼性 eval」のように振る舞うテストがあります。

- 実際の Gateway + エージェントループを通したモックツールコール（`src/gateway/gateway.test.ts`）。
- セッション配線と設定効果を検証するエンドツーエンドの ウィザード フロー（`src/gateway/gateway.test.ts`）。

Skills について、まだ不足しているもの（[Skills](/ja-JP/tools/skills) を参照）:

- **意思決定**: Skills がプロンプトに列挙されているとき、エージェントは正しい skill を選ぶか（あるいは無関係なものを避けるか）？
- **準拠**: エージェントは使用前に `SKILL.md` を読み、必須の手順/引数に従うか？
- **ワークフロー契約**: ツール順序、セッション履歴の引き継ぎ、サンドボックス境界を検証するマルチターンシナリオ。

今後の eval は、まず決定論的であるべきです。

- ツールコール + 順序、skill ファイル読み取り、セッション配線を検証する、モック provider を使ったシナリオランナー。
- skill に焦点を当てた小規模シナリオ群（使う vs 避ける、ゲーティング、プロンプトインジェクション）。
- オプションの live eval（オプトイン、env ゲート付き）は、CI 安全スイートが整った後のみ。

## 契約テスト（Plugin およびチャネルの形状）

契約テストは、登録されているすべての Plugin とチャネルが、その
インターフェース契約に準拠していることを検証します。検出されたすべての Plugin を反復し、
形状と挙動に関する一連の検証を実行します。デフォルトの `pnpm test` unit レーンは、
これらの共有 seam およびスモークファイルを意図的にスキップするため、共有チャネルまたは provider サーフェスに触れた場合は
契約コマンドを明示的に実行してください。

### コマンド

- すべての契約: `pnpm test:contracts`
- チャネル契約のみ: `pnpm test:contracts:channels`
- provider 契約のみ: `pnpm test:contracts:plugins`

### チャネル契約

`src/channels/plugins/contracts/*.contract.test.ts` にあります:

- **plugin** - 基本的な Plugin 形状（id、name、capabilities）
- **setup** - セットアップ ウィザード 契約
- **session-binding** - セッション bind の挙動
- **outbound-payload** - メッセージペイロード構造
- **inbound** - 受信メッセージ処理
- **actions** - チャネルアクションハンドラ
- **threading** - スレッド ID 処理
- **directory** - ディレクトリ/ロスター API
- **group-policy** - グループポリシー強制

### provider status 契約

`src/plugins/contracts/*.contract.test.ts` にあります。

- **status** - チャネル status プローブ
- **registry** - Plugin レジストリ形状

### provider 契約

`src/plugins/contracts/*.contract.test.ts` にあります:

- **auth** - 認証フロー契約
- **auth-choice** - 認証の選択/選定
- **catalog** - model catalog API
- **discovery** - Plugin 検出
- **loader** - Plugin ロード
- **runtime** - provider ランタイム
- **shape** - Plugin 形状/インターフェース
- **wizard** - セットアップ ウィザード

### 実行すべきタイミング

- plugin-sdk の export または subpath を変更した後
- チャネルまたは provider Plugin を追加または変更した後
- Plugin 登録または検出をリファクタリングした後

契約テストは CI で実行され、実際の API キーは必要ありません。

## リグレッションの追加（ガイダンス）

live で見つかった provider/model 問題を修正したとき:

- 可能なら CI 安全なリグレッションを追加してください（provider のモック/スタブ、または正確なリクエスト形状変換のキャプチャ）
- 本質的に live 専用の問題（レート制限、認証ポリシー）の場合は、live テストを狭く保ち、環境変数経由でオプトインにしてください
- バグを捕捉できる最小のレイヤーを狙うことを優先してください:
  - provider のリクエスト変換/再生バグ → 直接 model テスト
  - Gateway のセッション/履歴/ツールパイプラインのバグ → Gateway live スモーク、または CI 安全な Gateway モックテスト
- SecretRef トラバーサルガードレール:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` は、レジストリメタデータ（`listSecretTargetRegistryEntries()`）から SecretRef クラスごとに 1 つのサンプルターゲットを導出し、その後トラバーサルセグメント exec ID が拒否されることを検証します。
  - `src/secrets/target-registry-data.ts` に新しい `includeInPlan` SecretRef ターゲットファミリーを追加する場合は、そのテスト内の `classifyTargetClass` を更新してください。このテストは、未分類のターゲット ID があると意図的に失敗するため、新しいクラスを黙ってスキップできません。
