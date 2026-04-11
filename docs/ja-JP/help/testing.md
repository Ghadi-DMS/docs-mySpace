---
read_when:
    - ローカルまたはCIでテストを実行する
    - モデル/プロバイダーのバグに対する回帰テストを追加する
    - Gateway + エージェントの動作をデバッグする
summary: 'テストキット: unit/e2e/liveスイート、Dockerランナー、および各テストでカバーされる内容'
title: テスト
x-i18n:
    generated_at: "2026-04-11T02:45:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 55e75d056306a77b0d112a3902c08c7771f53533250847fc3d785b1df3e0e9e7
    source_path: help/testing.md
    workflow: 15
---

# テスト

OpenClawには3つのVitestスイート（unit/integration、e2e、live）と、小規模なDockerランナー群があります。

このドキュメントは「どのようにテストするか」のガイドです。

- 各スイートが何をカバーするか（そして意図的に何をカバーしないか）
- 一般的なワークフロー（ローカル、push前、デバッグ）でどのコマンドを実行するか
- liveテストがどのように認証情報を検出し、モデル/プロバイダーを選択するか
- 実運用で発生したモデル/プロバイダーの問題に対する回帰テストをどう追加するか

## クイックスタート

たいていの日は次を使います。

- フルゲート（push前に想定）: `pnpm build && pnpm check && pnpm test`
- 余裕のあるマシンでのより高速なローカルフルスイート実行: `pnpm test:max`
- 直接のVitestウォッチループ: `pnpm test:watch`
- 直接のファイル指定は、extension/channelパスにも対応するようになりました: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- 単一の失敗を反復修正しているときは、まず対象を絞った実行を優先してください。
- DockerベースのQAサイト: `pnpm qa:lab:up`
- Linux VMベースのQAレーン: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

テストに手を入れたとき、または追加の確信がほしいとき:

- カバレッジゲート: `pnpm test:coverage`
- E2Eスイート: `pnpm test:e2e`

実際のプロバイダー/モデルをデバッグするとき（実際の認証情報が必要）:

- liveスイート（モデル + Gatewayツール/画像プローブ）: `pnpm test:live`
- 1つのliveファイルを静かに対象指定: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

ヒント: 必要なのが1つの失敗ケースだけなら、以下で説明するallowlist環境変数でliveテストを絞り込むことを優先してください。

## QA専用ランナー

これらのコマンドは、QA-labレベルの現実性が必要なときにメインのテストスイートと並んで使います。

- `pnpm openclaw qa suite`
  - リポジトリベースのQAシナリオをホスト上で直接実行します。
  - デフォルトでは、分離されたgatewayワーカーで複数の選択シナリオを並列実行します。最大64ワーカーまたは選択したシナリオ数までです。ワーカー数を調整するには`--concurrency <count>`を使用し、以前の直列レーンにするには`--concurrency 1`を使用してください。
- `pnpm openclaw qa suite --runner multipass`
  - 同じQAスイートを使い捨てのMultipass Linux VM内で実行します。
  - ホスト上の`qa suite`と同じシナリオ選択動作を維持します。
  - `qa suite`と同じプロバイダー/モデル選択フラグを再利用します。
  - live実行では、ゲストで実用的なサポート対象QA認証入力を転送します:
    環境変数ベースのプロバイダーキー、QA liveプロバイダー設定パス、存在する場合は`CODEX_HOME`。
  - 出力ディレクトリは、ゲストがマウントされたワークスペース経由で書き戻せるよう、リポジトリルート配下に保つ必要があります。
  - 通常のQAレポート + サマリーに加え、Multipassログを`.artifacts/qa-e2e/...`配下に書き込みます。
- `pnpm qa:lab:up`
  - オペレーター型QA作業向けにDockerベースのQAサイトを起動します。
- `pnpm openclaw qa matrix`
  - 使い捨てのDockerベースTuwunel homeserverに対して、Matrix live QAレーンを実行します。
  - 一時的な3人のMatrixユーザー（`driver`、`sut`、`observer`）と1つのプライベートルームをプロビジョニングし、その後、実際のMatrix pluginをSUTトランスポートとして使うQA gateway子プロセスを起動します。
  - デフォルトでは固定された安定版Tuwunelイメージ`ghcr.io/matrix-construct/tuwunel:v1.5.1`を使用します。別のイメージをテストする必要がある場合は`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE`で上書きしてください。
  - Matrix QAレポート、サマリー、およびobserved-eventsアーティファクトを`.artifacts/qa-e2e/...`配下に書き込みます。
- `pnpm openclaw qa telegram`
  - 環境変数から取得したdriverおよびSUTボットトークンを使って、実際のプライベートグループに対してTelegram live QAレーンを実行します。
  - `OPENCLAW_QA_TELEGRAM_GROUP_ID`、`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`が必要です。グループIDは数値のTelegramチャットIDである必要があります。
  - 同じプライベートグループ内に2つの異なるボットが必要で、SUTボットはTelegramユーザー名を公開している必要があります。
  - 安定したbot間観測のため、両方のボットで`@BotFather`のBot-to-Bot Communication Modeを有効にし、driverボットがグループ内ボットトラフィックを観測できることを確認してください。
  - Telegram QAレポート、サマリー、およびobserved-messagesアーティファクトを`.artifacts/qa-e2e/...`配下に書き込みます。

liveトランスポートレーンは、新しいトランスポートがずれないように、1つの標準コントラクトを共有します。

`qa-channel`は引き続き広範な合成QAスイートであり、liveトランスポートのカバレッジマトリクスには含まれません。

| レーン    | Canary | Mention gating | Allowlist block | Top-level reply | Restart resume | Thread follow-up | Thread isolation | Reaction observation | Help command |
| --------- | ------ | -------------- | --------------- | --------------- | -------------- | ---------------- | ---------------- | -------------------- | ------------ |
| Matrix    | x      | x              | x               | x               | x              | x                | x                | x                    |              |
| Telegram  | x      |                |                 |                 |                |                  |                  |                      | x            |

## テストスイート（どこで何が実行されるか）

スイートは「現実性が増していくもの」（そして不安定さ/コストも増していくもの）として考えてください。

### Unit / integration（デフォルト）

- コマンド: `pnpm test`
- 設定: 既存のスコープ付きVitestプロジェクトに対する10個の逐次シャード実行（`vitest.full-*.config.ts`）
- ファイル: `src/**/*.test.ts`、`packages/**/*.test.ts`、`test/**/*.test.ts`配下のcore/unitインベントリと、`vitest.unit.config.ts`でカバーされるホワイトリスト化された`ui`のnodeテスト
- 範囲:
  - 純粋なユニットテスト
  - プロセス内統合テスト（gateway認証、ルーティング、ツーリング、パース、設定）
  - 既知のバグに対する決定的な回帰テスト
- 期待事項:
  - CIで実行される
  - 実際のキーは不要
  - 高速かつ安定しているべき
- プロジェクトに関する注記:
  - 対象未指定の`pnpm test`は、1つの巨大なネイティブルートプロジェクトプロセスではなく、11個の小さなシャード設定（`core-unit-src`、`core-unit-security`、`core-unit-ui`、`core-unit-support`、`core-support-boundary`、`core-contracts`、`core-bundled`、`core-runtime`、`agentic`、`auto-reply`、`extensions`）を実行するようになりました。これにより、負荷の高いマシンでのピークRSSが削減され、auto-reply/extensionの作業が無関係なスイートを圧迫するのを防ぎます。
  - `pnpm test --watch`は、マルチシャードのウォッチループが現実的ではないため、引き続きネイティブルート`vitest.config.ts`プロジェクトグラフを使います。
  - `pnpm test`、`pnpm test:watch`、`pnpm test:perf:imports`は、明示的なファイル/ディレクトリ対象をまずスコープ付きレーン経由でルーティングするため、`pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`ではルートプロジェクト全体の起動コストを払わずに済みます。
  - `pnpm test:changed`は、差分がルーティング可能なソース/テストファイルだけに触れている場合、変更されたgitパスを同じスコープ付きレーンに展開します。設定/セットアップの編集は、引き続き広範なルートプロジェクト再実行にフォールバックします。
  - agents、commands、plugins、auto-replyヘルパー、`plugin-sdk`、および同様の純粋なユーティリティ領域からの軽量importユニットテストは、`test/setup-openclaw-runtime.ts`をスキップする`unit-fast`レーンを通ります。状態を持つ/ランタイム負荷の高いファイルは既存レーンに残ります。
  - 一部の`plugin-sdk`および`commands`ヘルパーソースファイルは、変更モード実行をそれらの軽量レーン内の明示的な隣接テストにもマッピングするため、ヘルパー編集でそのディレクトリの重いフルスイートを再実行せずに済みます。
  - `auto-reply`には現在3つの専用バケットがあります: トップレベルcoreヘルパー、トップレベル`reply.*`統合テスト、`src/auto-reply/reply/**`サブツリー。これにより、最も重いreplyハーネス作業を軽量なstatus/chunk/tokenテストから切り離せます。
- 組み込みランナーに関する注記:
  - メッセージツール検出入力またはcompactionランタイムコンテキストを変更する場合は、両方のレベルのカバレッジを維持してください。
  - 純粋なルーティング/正規化境界については、焦点を絞ったヘルパー回帰テストを追加してください。
  - また、組み込みランナーの統合スイートも健全に保ってください:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`、
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`、および
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`。
  - これらのスイートは、スコープ付きIDとcompaction動作が実際の`run.ts` / `compact.ts`経路を通って流れ続けることを検証します。ヘルパーのみのテストは、これらの統合経路の十分な代替にはなりません。
- プールに関する注記:
  - ベースVitest設定は現在デフォルトで`threads`です。
  - 共有Vitest設定は`isolate: false`も固定し、root projects、e2e、live設定全体で非分離ランナーを使用します。
  - ルートUIレーンはその`jsdom`セットアップとオプティマイザーを維持しますが、現在は共有の非分離ランナー上でも動作します。
  - 各`pnpm test`シャードは、共有Vitest設定から同じ`threads` + `isolate: false`デフォルトを継承します。
  - 共有`scripts/run-vitest.mjs`ランチャーは、大規模なローカル実行中のV8コンパイル負荷を減らすため、Vitest子Nodeプロセスに対してデフォルトで`--no-maglev`も追加するようになりました。標準V8動作と比較したい場合は`OPENCLAW_VITEST_ENABLE_MAGLEV=1`を設定してください。
- 高速ローカル反復に関する注記:
  - `pnpm test:changed`は、変更パスがより小さなスイートにきれいに対応する場合、スコープ付きレーンを経由します。
  - `pnpm test:max`と`pnpm test:changed:max`も同じルーティング動作を維持しつつ、ワーカー上限を高くします。
  - ローカルワーカー自動スケーリングは現在意図的に保守的で、ホストのロードアベレージがすでに高い場合にも抑制されるため、複数のVitest実行が同時に走っても通常は影響が小さくなります。
  - ベースVitest設定は、変更モード再実行がテスト配線の変更時にも正しく保たれるよう、projects/configファイルを`forceRerunTriggers`としてマークします。
  - 設定は、サポートされるホストで`OPENCLAW_VITEST_FS_MODULE_CACHE`を有効に保ちます。直接プロファイリング用に明示的なキャッシュ場所を1つ指定したい場合は`OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`を設定してください。
- パフォーマンスデバッグに関する注記:
  - `pnpm test:perf:imports`はVitestのimport所要時間レポートとimport内訳出力を有効にします。
  - `pnpm test:perf:imports:changed`は、`origin/main`以降に変更されたファイルに対して同じプロファイリング表示を絞り込みます。
- `pnpm test:perf:changed:bench -- --ref <git-ref>`は、そのコミット済み差分に対してルーティングされた`test:changed`とネイティブルートプロジェクト経路を比較し、wall timeとmacOS max RSSを表示します。
- `pnpm test:perf:changed:bench -- --worktree`は、変更中の現在ツリーを`script/test-projects.mjs`とルートVitest設定経由で変更ファイル一覧にルーティングしてベンチマークします。
  - `pnpm test:perf:profile:main`は、Vitest/Vite起動とtransformオーバーヘッドのmain-thread CPUプロファイルを書き出します。
  - `pnpm test:perf:profile:runner`は、ファイル並列を無効にしたunitスイート用のrunner CPU+heapプロファイルを書き出します。

### E2E（gatewayスモーク）

- コマンド: `pnpm test:e2e`
- 設定: `vitest.e2e.config.ts`
- ファイル: `src/**/*.e2e.test.ts`、`test/**/*.e2e.test.ts`
- ランタイムのデフォルト:
  - リポジトリの他部分と同様に、Vitest `threads`と`isolate: false`を使用します。
  - 適応型ワーカーを使用します（CI: 最大2、ローカル: デフォルト1）。
  - コンソールI/Oのオーバーヘッドを減らすため、デフォルトでsilent modeで実行されます。
- 便利なオーバーライド:
  - ワーカー数を強制するには`OPENCLAW_E2E_WORKERS=<n>`（上限16）
  - 詳細なコンソール出力を再有効化するには`OPENCLAW_E2E_VERBOSE=1`
- 範囲:
  - 複数インスタンスgatewayのエンドツーエンド動作
  - WebSocket/HTTPサーフェス、ノードペアリング、およびより重いネットワーキング
- 期待事項:
  - CIで実行される（パイプラインで有効な場合）
  - 実際のキーは不要
  - ユニットテストより可動部が多い（遅くなることがある）

### E2E: OpenShellバックエンドスモーク

- コマンド: `pnpm test:e2e:openshell`
- ファイル: `test/openshell-sandbox.e2e.test.ts`
- 範囲:
  - Docker経由でホスト上に分離されたOpenShell gatewayを起動
  - 一時的なローカルDockerfileからsandboxを作成
  - 実際の`sandbox ssh-config` + SSH execを通じてOpenClawのOpenShellバックエンドを実行
  - sandbox fs bridgeを通じてリモート正規filesystem動作を検証
- 期待事項:
  - オプトイン専用であり、デフォルトの`pnpm test:e2e`実行には含まれない
  - ローカルの`openshell` CLIと動作するDockerデーモンが必要
  - 分離された`HOME` / `XDG_CONFIG_HOME`を使用し、その後テストgatewayとsandboxを破棄する
- 便利なオーバーライド:
  - より広いe2eスイートを手動実行するときにテストを有効化するには`OPENCLAW_E2E_OPENSHELL=1`
  - デフォルト以外のCLIバイナリまたはラッパースクリプトを指定するには`OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`

### Live（実際のプロバイダー + 実際のモデル）

- コマンド: `pnpm test:live`
- 設定: `vitest.live.config.ts`
- ファイル: `src/**/*.live.test.ts`
- デフォルト: `pnpm test:live`で**有効**（`OPENCLAW_LIVE_TEST=1`を設定）
- 範囲:
  - 「このprovider/modelは、実際の認証情報で、_今日_実際に動くか？」
  - providerのフォーマット変更、tool-callingの癖、認証の問題、レート制限の挙動を検出
- 期待事項:
  - 設計上、CIで安定するものではない（実ネットワーク、実際のproviderポリシー、クォータ、障害）
  - コストがかかる / レート制限を消費する
  - 「全部」を実行するより、絞り込んだサブセットを優先する
- live実行では、不足しているAPIキーを取得するために`~/.profile`を読み込みます。
- デフォルトでは、live実行は引き続き`HOME`を分離し、設定/認証情報を一時的なテスト用homeにコピーするため、unit用fixtureが実際の`~/.openclaw`を変更できません。
- liveテストで意図的に実際のhomeディレクトリを使う必要がある場合のみ、`OPENCLAW_LIVE_USE_REAL_HOME=1`を設定してください。
- `pnpm test:live`は現在、より静かなモードがデフォルトです。`[live] ...`の進行状況出力は維持しますが、追加の`~/.profile`通知は抑制し、gateway起動ログ/Bonjour chatterをミュートします。完全な起動ログを再表示したい場合は`OPENCLAW_LIVE_TEST_QUIET=0`を設定してください。
- APIキーのローテーション（provider別）: カンマ/セミコロン形式の`*_API_KEYS`、または`*_API_KEY_1`、`*_API_KEY_2`（例: `OPENAI_API_KEYS`、`ANTHROPIC_API_KEYS`、`GEMINI_API_KEYS`）を設定するか、live専用のオーバーライドとして`OPENCLAW_LIVE_*_KEY`を使います。テストはレート制限レスポンス時に再試行します。
- 進行状況/heartbeat出力:
  - liveスイートは現在、stderrに進行状況行を出力するため、Vitestのコンソールキャプチャが静かでも、長いprovider呼び出しが動作中であることが見えるようになっています。
  - `vitest.live.config.ts`はVitestのコンソール割り込みを無効にしているため、provider/gatewayの進行状況行がlive実行中に即座にストリームされます。
  - 直接モデルのheartbeatは`OPENCLAW_LIVE_HEARTBEAT_MS`で調整します。
  - gateway/probeのheartbeatは`OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`で調整します。

## どのスイートを実行すべきか？

この判断表を使ってください。

- ロジック/テストを編集した: `pnpm test`を実行（大きく変更したなら`pnpm test:coverage`も）
- gatewayネットワーキング / WSプロトコル / ペアリングに触れた: `pnpm test:e2e`も追加
- 「botが落ちている」 / provider固有の失敗 / tool callingをデバッグしている: 絞り込んだ`pnpm test:live`を実行

## Live: Androidノード機能スイープ

- テスト: `src/gateway/android-node.capabilities.live.test.ts`
- スクリプト: `pnpm android:test:integration`
- 目的: 接続されたAndroidノードが現在公開している**すべてのコマンド**を呼び出し、コマンドコントラクトの挙動を検証すること。
- 範囲:
  - 前提条件付き/手動セットアップ（このスイートはアプリのインストール/起動/ペアリングは行いません）。
  - 選択したAndroidノードに対するコマンド単位のgateway `node.invoke`検証。
- 必要な事前セットアップ:
  - Androidアプリがすでにgatewayに接続され、ペアリング済みであること。
  - アプリがフォアグラウンドに維持されていること。
  - 通過を期待する機能に対して、権限/キャプチャ同意が付与されていること。
- 任意の対象オーバーライド:
  - `OPENCLAW_ANDROID_NODE_ID`または`OPENCLAW_ANDROID_NODE_NAME`。
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`。
- Androidの完全なセットアップ詳細: [Android App](/ja-JP/platforms/android)

## Live: modelスモーク（profileキー）

liveテストは、失敗を切り分けられるように2つのレイヤーに分かれています。

- 「Direct model」は、そのprovider/modelが与えられたキーで少なくとも応答できるかを示します。
- 「Gateway smoke」は、そのモデルに対して完全なgateway+agentパイプラインが動作するかを示します（sessions、history、tools、sandbox policyなど）。

### レイヤー1: 直接モデル補完（gatewayなし）

- テスト: `src/agents/models.profiles.live.test.ts`
- 目的:
  - 検出されたモデルを列挙する
  - `getApiKeyForModel`を使って、認証情報を持つモデルを選択する
  - モデルごとに小さな補完を実行する（必要に応じて対象回帰も）
- 有効化方法:
  - `pnpm test:live`（またはVitestを直接呼び出す場合は`OPENCLAW_LIVE_TEST=1`）
- このスイートを実際に実行するには`OPENCLAW_LIVE_MODELS=modern`（または`all`、`modern`の別名）を設定します。そうしないと、`pnpm test:live`をgateway smokeに集中させるためにスキップされます。
- モデルの選び方:
  - `OPENCLAW_LIVE_MODELS=modern`でmodern allowlistを実行（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_MODELS=all`はmodern allowlistの別名
  - または`OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."`（カンマ区切りallowlist）
  - modern/allスイープはデフォルトで厳選された高シグナル上限を使います。網羅的なmodernスイープには`OPENCLAW_LIVE_MAX_MODELS=0`を、より小さい上限には正の数を設定してください。
- providerの選び方:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（カンマ区切りallowlist）
- キーの取得元:
  - デフォルト: profile storeとenvフォールバック
  - **profile storeのみ**を強制するには`OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`を設定
- これが存在する理由:
  - 「provider APIが壊れている / キーが無効」と「gateway agentパイプラインが壊れている」を切り分けるため
  - 小さく分離された回帰を含めるため（例: OpenAI Responses/Codex Responsesのreasoning replay + tool-callフロー）

### レイヤー2: Gateway + dev agentスモーク（`@openclaw`が実際に何をするか）

- テスト: `src/gateway/gateway-models.profiles.live.test.ts`
- 目的:
  - プロセス内gatewayを起動する
  - `agent:dev:*`セッションを作成/パッチする（実行ごとにモデルオーバーライド）
  - キー付きモデルを反復し、次を検証する:
    - 「意味のある」応答（toolsなし）
    - 実際のtool呼び出しが動作する（read probe）
    - 任意の追加tool probe（exec+read probe）
    - OpenAIの回帰パス（tool-callのみ → フォローアップ）が機能し続ける
- probeの詳細（失敗を素早く説明できるように）:
  - `read` probe: テストがワークスペースにnonceファイルを書き込み、エージェントにそれを`read`してnonceをそのまま返すよう要求します。
  - `exec+read` probe: テストがエージェントに`exec`で一時ファイルへnonceを書かせ、その後それを`read`で読み戻すよう要求します。
  - image probe: テストが生成したPNG（cat + ランダムコード）を添付し、モデルが`cat <CODE>`を返すことを期待します。
  - 実装参照: `src/gateway/gateway-models.profiles.live.test.ts`および`src/gateway/live-image-probe.ts`。
- 有効化方法:
  - `pnpm test:live`（またはVitestを直接呼び出す場合は`OPENCLAW_LIVE_TEST=1`）
- モデルの選び方:
  - デフォルト: modern allowlist（Opus/Sonnet 4.6+、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.7、Grok 4）
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all`はmodern allowlistの別名
  - または`OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`（またはカンマ区切りリスト）で絞り込み
  - modern/all gatewayスイープはデフォルトで厳選された高シグナル上限を使います。網羅的なmodernスイープには`OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0`を、より小さい上限には正の数を設定してください。
- providerの選び方（「OpenRouter全部」を避ける）:
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（カンマ区切りallowlist）
- このliveテストではtool + image probeは常に有効です:
  - `read` probe + `exec+read` probe`（toolストレス）
  - image input supportをモデルが公開している場合、image probeを実行
  - フロー（高レベル）:
    - テストが「CAT」+ランダムコード入りの小さなPNGを生成（`src/gateway/live-image-probe.ts`）
    - それを`agent`へ`attachments: [{ mimeType: "image/png", content: "<base64>" }]`として送信
    - Gatewayが添付を`images[]`へパース（`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`）
    - 組み込みagentがマルチモーダルなユーザーメッセージをモデルへ転送
    - 検証: 返信に`cat` + そのコードが含まれること（OCR許容: 軽微な誤りは許可）

ヒント: 自分のマシンで何をテストできるか（および正確な`provider/model` ID）を見るには、次を実行してください。
__OC_I18N_900000__
## Live: CLIバックエンドスモーク（Claude、Codex、Gemini、またはその他のローカルCLI）

- テスト: `src/gateway/gateway-cli-backend.live.test.ts`
- 目的: デフォルト設定に触れずに、ローカルCLIバックエンドを使ってGateway + agentパイプラインを検証すること。
- バックエンド固有のスモークデフォルトは、所有するextensionの`cli-backend.ts`定義にあります。
- 有効化:
  - `pnpm test:live`（またはVitestを直接呼び出す場合は`OPENCLAW_LIVE_TEST=1`）
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- デフォルト:
  - デフォルトprovider/model: `claude-cli/claude-sonnet-4-6`
  - command/args/image動作は、所有するCLIバックエンドpluginメタデータから取得されます。
- オーバーライド（任意）:
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - 実際の画像添付を送るには`OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1`（パスはプロンプトに注入されます）。
  - プロンプト注入ではなくCLI引数として画像ファイルパスを渡すには`OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"`
  - `IMAGE_ARG`が設定されているときの画像引数の渡し方を制御するには`OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（または`"list"`）
  - 2ターン目を送り、resumeフローを検証するには`OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1`
  - デフォルトのClaude Sonnet -> Opus同一セッション継続性probeを無効にするには`OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0`（選択モデルが切り替え先をサポートしているときに強制的に有効化するには`1`）

例:
__OC_I18N_900001__
Dockerレシピ:
__OC_I18N_900002__
単一providerのDockerレシピ:
__OC_I18N_900003__
注記:

- Dockerランナーは`scripts/test-live-cli-backend-docker.sh`にあります。
- これは、リポジトリDockerイメージ内で、非rootの`node`ユーザーとしてlive CLI-backendスモークを実行します。
- 所有するextensionからCLIスモークメタデータを解決し、対応するLinux CLIパッケージ（`@anthropic-ai/claude-code`、`@openai/codex`、または`@google/gemini-cli`）を、`OPENCLAW_DOCKER_CLI_TOOLS_DIR`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）にあるキャッシュ可能な書き込み可能prefixへインストールします。
- `pnpm test:docker:live-cli-backend:claude-subscription`は、`~/.claude/.credentials.json`内の`claudeAiOauth.subscriptionType`、または`claude setup-token`由来の`CLAUDE_CODE_OAUTH_TOKEN`のいずれかによるポータブルなClaude Code subscription OAuthを必要とします。これはまずDocker内での直接`claude -p`を検証し、その後Anthropic APIキーenvを保持せずに2回のGateway CLI-backendターンを実行します。このsubscriptionレーンでは、Claudeが現在、サードパーティーアプリ利用を通常のsubscriptionプラン上限ではなく追加利用課金経由で処理しているため、Claude MCP/toolおよびimage probeはデフォルトで無効化されます。
- live CLI-backendスモークは現在、Claude、Codex、Geminiに対して同じエンドツーエンドフローを実行します: テキストターン、画像分類ターン、その後gateway CLI経由で検証されるMCP `cron` tool call。
- Claudeのデフォルトスモークは、セッションをSonnetからOpusへパッチし、再開したセッションが以前のメモを引き続き記憶していることも検証します。

## Live: ACPバインドスモーク（`/acp spawn ... --bind here`）

- テスト: `src/gateway/gateway-acp-bind.live.test.ts`
- 目的: live ACP agentを使った実際のACP conversation-bindフローを検証すること:
  - `/acp spawn <agent> --bind here`を送信する
  - 合成されたmessage-channel会話をその場でバインドする
  - 同じ会話上で通常のフォローアップを送る
  - そのフォローアップが、バインドされたACPセッションのトランスクリプトに到達することを確認する
- 有効化:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- デフォルト:
  - Docker内のACP agent: `claude,codex,gemini`
  - 直接`pnpm test:live ...`用のACP agent: `claude`
  - 合成チャネル: Slack DM風の会話コンテキスト
  - ACPバックエンド: `acpx`
- オーバーライド:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- 注記:
  - このレーンは、admin専用の合成originating-routeフィールドを持つgatewayの`chat.send`サーフェスを使うため、外部配信を装わずにmessage-channelコンテキストをテストへアタッチできます。
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND`が未設定の場合、テストは埋め込み`acpx`pluginの組み込みagent registryを使って選択したACPハーネスagentを解決します。

例:
__OC_I18N_900004__
Dockerレシピ:
__OC_I18N_900005__
単一agentのDockerレシピ:
__OC_I18N_900006__
Dockerに関する注記:

- Dockerランナーは`scripts/test-live-acp-bind-docker.sh`にあります。
- デフォルトでは、サポートされているすべてのlive CLI agentに対してACP bindスモークを順番に実行します: `claude`、`codex`、`gemini`。
- マトリクスを絞り込むには、`OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`、`OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`、または`OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini`を使用してください。
- これは`~/.profile`を読み込み、一致するCLI認証情報をコンテナにステージし、書き込み可能なnpm prefixに`acpx`をインストールした後、必要なら要求されたlive CLI（`@anthropic-ai/claude-code`、`@openai/codex`、または`@google/gemini-cli`）をインストールします。
- Docker内では、ランナーは`OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`を設定し、`acpx`が読み込まれたprofileのprovider env varを子ハーネスCLIで使えるようにします。

## Live: Codex app-serverハーネススモーク

- 目的: pluginが所有するCodexハーネスを通常のgateway
  `agent`メソッド経由で検証すること:
  - バンドルされた`codex`pluginを読み込む
  - `OPENCLAW_AGENT_RUNTIME=codex`を選択する
  - 最初のgateway agentターンを`codex/gpt-5.4`へ送信する
  - 2回目のターンを同じOpenClawセッションへ送信し、app-server
    スレッドが再開できることを確認する
  - 同じgatewayコマンド
    経路を通じて`/codex status`と`/codex models`を実行する
- テスト: `src/gateway/gateway-codex-harness.live.test.ts`
- 有効化: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- デフォルトモデル: `codex/gpt-5.4`
- 任意のimage probe: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- 任意のMCP/tool probe: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- このスモークでは`OPENCLAW_AGENT_HARNESS_FALLBACK=none`を設定するため、壊れたCodex
  ハーネスがPIへ黙ってフォールバックして通過することはできません。
- 認証: シェル/profileからの`OPENAI_API_KEY`に加え、任意でコピーされる
  `~/.codex/auth.json`と`~/.codex/config.toml`

ローカルレシピ:
__OC_I18N_900007__
Dockerレシピ:
__OC_I18N_900008__
Dockerに関する注記:

- Dockerランナーは`scripts/test-live-codex-harness-docker.sh`にあります。
- これはマウントされた`~/.profile`を読み込み、`OPENAI_API_KEY`を渡し、存在する場合はCodex CLIの
  認証ファイルをコピーし、書き込み可能なマウント済みnpm
  prefixに`@openai/codex`をインストールし、ソースツリーをステージした後、Codex-harness liveテストのみを実行します。
- DockerはデフォルトでimageおよびMCP/tool probeを有効にします。より狭いデバッグ実行が必要な場合は
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0`または
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0`を設定してください。
- Dockerは`OPENCLAW_AGENT_HARNESS_FALLBACK=none`もエクスポートし、live
  テスト設定に合わせることで、`openai-codex/*`またはPIフォールバックがCodexハーネスの
  回帰を隠せないようにします。

### 推奨liveレシピ

狭く明示的なallowlistが最も高速で、最も不安定さが少ないです。

- 単一モデル、直接（gatewayなし）:
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- 単一モデル、gatewayスモーク:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 複数providerにまたがるtool calling:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google重視（Gemini API key + Antigravity）:
  - Gemini（API key）: `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）: `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

注記:

- `google/...`はGemini API（API key）を使います。
- `google-antigravity/...`はAntigravity OAuthブリッジ（Cloud Code Assist風のagent endpoint）を使います。
- `google-gemini-cli/...`はローカルマシン上のGemini CLIを使います（別個の認証 + ツーリングの癖があります）。
- Gemini APIとGemini CLI:
  - API: OpenClawがGoogleのホスト型Gemini APIをHTTP経由で呼び出します（API key / profile auth）。これはたいていのユーザーが「Gemini」と言うときに意味するものです。
  - CLI: OpenClawがローカルの`gemini`バイナリをシェル実行します。独自の認証があり、挙動が異なることがあります（streaming/tool support/version skew）。

## Live: modelマトリクス（何をカバーしているか）

固定の「CIモデル一覧」はありません（liveはオプトイン）が、キーを持つ開発マシンで定期的にカバーすることを**推奨**するモデルは次のとおりです。

### Modernスモークセット（tool calling + image）

これは、動作し続けることを期待している「一般的なモデル」の実行です。

- OpenAI（非Codex）: `openai/gpt-5.4`（任意: `openai/gpt-5.4-mini`）
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6`（または`anthropic/claude-sonnet-4-6`）
- Google（Gemini API）: `google/gemini-3.1-pro-preview`および`google/gemini-3-flash-preview`（古いGemini 2.xモデルは避ける）
- Google（Antigravity）: `google-antigravity/claude-opus-4-6-thinking`および`google-antigravity/gemini-3-flash`
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

tools + image付きでgatewayスモークを実行:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### ベースライン: tool calling（Read + 任意のExec）

providerファミリーごとに少なくとも1つ選んでください。

- OpenAI: `openai/gpt-5.4`（または`openai/gpt-5.4-mini`）
- Anthropic: `anthropic/claude-opus-4-6`（または`anthropic/claude-sonnet-4-6`）
- Google: `google/gemini-3-flash-preview`（または`google/gemini-3.1-pro-preview`）
- Z.AI（GLM）: `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

任意の追加カバレッジ（あるとよい）:

- xAI: `xai/grok-4`（または利用可能な最新）
- Mistral: `mistral/`…（有効化済みの「tools」対応モデルを1つ選ぶ）
- Cerebras: `cerebras/`…（アクセス権がある場合）
- LM Studio: `lmstudio/`…（ローカル; tool callingはAPIモードに依存）

### Vision: image送信（添付 → マルチモーダルメッセージ）

image probeを通すために、少なくとも1つのimage対応モデルを`OPENCLAW_LIVE_GATEWAY_MODELS`に含めてください（Claude/Gemini/OpenAIのvision対応バリアントなど）。

### アグリゲーター / 代替gateway

キーが有効であれば、次経由のテストもサポートしています。

- OpenRouter: `openrouter/...`（数百のモデル; tool+image対応候補を見つけるには`openclaw models scan`を使用）
- OpenCode: Zen用の`opencode/...`およびGo用の`opencode-go/...`（認証は`OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`）

liveマトリクスに含められるproviderは他にもあります（認証情報/設定がある場合）:

- 組み込み: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- `models.providers`経由（カスタムエンドポイント）: `minimax`（cloud/API）、および任意のOpenAI/Anthropic互換proxy（LM Studio、vLLM、LiteLLMなど）

ヒント: ドキュメントに「すべてのモデル」をハードコードしようとしないでください。権威ある一覧は、そのマシン上で`discoverModels(...)`が返すものと、利用可能なキーによって決まります。

## 認証情報（絶対にコミットしない）

liveテストはCLIと同じ方法で認証情報を検出します。実際上の意味は次のとおりです。

- CLIが動くなら、liveテストも同じキーを見つけられるはずです。
- liveテストが「認証情報なし」と言うなら、`openclaw models list` / モデル選択のデバッグと同じ方法で調査してください。

- エージェント単位のauth profile: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（これがliveテストにおける「profile keys」の意味です）
- 設定: `~/.openclaw/openclaw.json`（または`OPENCLAW_CONFIG_PATH`）
- 旧stateディレクトリ: `~/.openclaw/credentials/`（存在する場合はステージ済みlive homeへコピーされますが、mainのprofile-key storeではありません）
- ローカルlive実行は、デフォルトでアクティブ設定、エージェント単位の`auth-profiles.json`ファイル、旧`credentials/`、およびサポートされる外部CLI認証ディレクトリを一時テストhomeへコピーします。ステージ済みlive homeでは`workspace/`と`sandboxes/`はスキップされ、`agents.*.workspace` / `agentDir`のパスオーバーライドは削除されるため、probeが実際のホストworkspaceに触れません。

envキーに依存したい場合（たとえば`~/.profile`でexportしている場合）は、`source ~/.profile`の後にローカルテストを実行するか、以下のDockerランナーを使ってください（これらは`~/.profile`をコンテナにマウントできます）。

## Deepgram live（音声文字起こし）

- テスト: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- 有効化: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- テスト: `src/agents/byteplus.live.test.ts`
- 有効化: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- 任意のモデルオーバーライド: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- テスト: `extensions/comfy/comfy.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- 範囲:
  - バンドルされたcomfyの画像、動画、`music_generate`経路を実行
  - `models.providers.comfy.<capability>`が設定されていない限り、各機能はスキップ
  - comfyのworkflow送信、ポーリング、ダウンロード、またはplugin登録を変更した後に有用

## 画像生成live

- テスト: `src/image-generation/runtime.live.test.ts`
- コマンド: `pnpm test:live src/image-generation/runtime.live.test.ts`
- ハーネス: `pnpm test:live:media image`
- 範囲:
  - 登録されているすべての画像生成provider pluginを列挙
  - probe前に、ログインシェル（`~/.profile`）から不足しているprovider env varを読み込む
  - デフォルトでは、保存済みauth profileよりlive/env APIキーを優先して使うため、`auth-profiles.json`内の古いテストキーが実際のシェル認証情報を覆い隠しません
  - 使用可能なauth/profile/modelがないproviderはスキップ
  - 共有ランタイム機能を通じて標準の画像生成バリアントを実行:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- 現在カバーされるバンドル済みprovider:
  - `openai`
  - `google`
- 任意の絞り込み:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- 任意の認証動作:
  - profile-store認証を強制し、envのみのオーバーライドを無視するには`OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## 音楽生成live

- テスト: `extensions/music-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- ハーネス: `pnpm test:live:media music`
- 範囲:
  - 共有のバンドル済み音楽生成provider経路を実行
  - 現在はGoogleとMiniMaxをカバー
  - probe前に、ログインシェル（`~/.profile`）からprovider env varを読み込む
  - デフォルトでは、保存済みauth profileよりlive/env APIキーを優先して使うため、`auth-profiles.json`内の古いテストキーが実際のシェル認証情報を覆い隠しません
  - 使用可能なauth/profile/modelがないproviderはスキップ
  - 利用可能な場合、宣言された両方のランタイムモードを実行:
    - プロンプトのみ入力での`generate`
    - providerが`capabilities.edit.enabled`を宣言している場合の`edit`
  - 現在の共有レーンカバレッジ:
    - `google`: `generate`, `edit`
    - `minimax`: `generate`
    - `comfy`: この共有スイープではなく、別のComfy liveファイル
- 任意の絞り込み:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- 任意の認証動作:
  - profile-store認証を強制し、envのみのオーバーライドを無視するには`OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## 動画生成live

- テスト: `extensions/video-generation-providers.live.test.ts`
- 有効化: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- ハーネス: `pnpm test:live:media video`
- 範囲:
  - 共有のバンドル済み動画生成provider経路を実行
  - probe前に、ログインシェル（`~/.profile`）からprovider env varを読み込む
  - デフォルトでは、保存済みauth profileよりlive/env APIキーを優先して使うため、`auth-profiles.json`内の古いテストキーが実際のシェル認証情報を覆い隠しません
  - 使用可能なauth/profile/modelがないproviderはスキップ
  - 利用可能な場合、宣言された両方のランタイムモードを実行:
    - プロンプトのみ入力での`generate`
    - providerが`capabilities.imageToVideo.enabled`を宣言しており、選択されたprovider/modelが共有スイープでバッファベースのローカル画像入力を受け付ける場合の`imageToVideo`
    - providerが`capabilities.videoToVideo.enabled`を宣言しており、選択されたprovider/modelが共有スイープでバッファベースのローカル動画入力を受け付ける場合の`videoToVideo`
  - 現在の共有スイープで宣言済みだがスキップされる`imageToVideo` provider:
    - バンドル済み`veo3`はテキスト専用で、バンドル済み`kling`はリモート画像URLを必要とするため、`vydra`
  - provider固有のVydraカバレッジ:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - このファイルは、`veo3`のテキストから動画へのレーンと、デフォルトでリモート画像URL fixtureを使う`kling`レーンを実行します
  - 現在の`videoToVideo` liveカバレッジ:
    - 選択モデルが`runway/gen4_aleph`の場合のみ`runway`
  - 現在の共有スイープで宣言済みだがスキップされる`videoToVideo` provider:
    - これらの経路は現在リモート`http(s)` / MP4参照URLを必要とするため、`alibaba`、`qwen`、`xai`
    - 現在の共有Gemini/Veoレーンはローカルのバッファベース入力を使い、その経路は共有スイープでは受け付けられないため、`google`
    - 現在の共有レーンにはorg固有の動画inpaint/remixアクセス保証がないため、`openai`
- 任意の絞り込み:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- 任意の認証動作:
  - profile-store認証を強制し、envのみのオーバーライドを無視するには`OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## メディアliveハーネス

- コマンド: `pnpm test:live:media`
- 目的:
  - 共有の画像、音楽、動画liveスイートを、リポジトリ標準の1つのエントリーポイントから実行
  - `~/.profile`から不足しているprovider env varを自動読み込み
  - デフォルトで、現在使用可能なauthを持つproviderに各スイートを自動的に絞り込む
  - `scripts/test-live.mjs`を再利用するため、heartbeatとquiet-modeの挙動が一貫する
- 例:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Dockerランナー（任意の「Linuxで動く」確認）

これらのDockerランナーは2つのバケットに分かれます。

- live-modelランナー: `test:docker:live-models`と`test:docker:live-gateway`は、それぞれ対応するprofile-key liveファイルのみをリポジトリDockerイメージ内で実行します（`src/agents/models.profiles.live.test.ts`と`src/gateway/gateway-models.profiles.live.test.ts`）。対応するローカルエントリーポイントは`test:live:models-profiles`と`test:live:gateway-profiles`です。
- Docker liveランナーは、完全なDockerスイープを現実的に保つため、デフォルトでより小さいスモーク上限を使います:
  `test:docker:live-models`はデフォルトで`OPENCLAW_LIVE_MAX_MODELS=12`、
  `test:docker:live-gateway`はデフォルトで`OPENCLAW_LIVE_GATEWAY_SMOKE=1`、
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`、
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`、および
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`を設定します。より大きな網羅的スキャンを明示的に行いたい場合は、これらのenv varを上書きしてください。
- `test:docker:all`は、まず`test:docker:live-build`でlive Dockerイメージを一度ビルドし、その後2つのlive Dockerレーンでそれを再利用します。
- コンテナスモークランナー: `test:docker:openwebui`、`test:docker:onboard`、`test:docker:gateway-network`、`test:docker:mcp-channels`、`test:docker:plugins`は、1つ以上の実コンテナを起動し、より高レベルの統合経路を検証します。

live-model Dockerランナーは、必要なCLI認証homeのみ（または実行が絞り込まれていない場合はサポート対象すべて）をbind mountし、実行前にそれらをコンテナhomeへコピーするため、外部CLI OAuthはホストの認証ストアを変更せずにトークンを更新できます。

- 直接モデル: `pnpm test:docker:live-models`（スクリプト: `scripts/test-live-models-docker.sh`）
- ACP bindスモーク: `pnpm test:docker:live-acp-bind`（スクリプト: `scripts/test-live-acp-bind-docker.sh`）
- CLIバックエンドスモーク: `pnpm test:docker:live-cli-backend`（スクリプト: `scripts/test-live-cli-backend-docker.sh`）
- Codex app-serverハーネススモーク: `pnpm test:docker:live-codex-harness`（スクリプト: `scripts/test-live-codex-harness-docker.sh`）
- Gateway + dev agent: `pnpm test:docker:live-gateway`（スクリプト: `scripts/test-live-gateway-models-docker.sh`）
- Open WebUI liveスモーク: `pnpm test:docker:openwebui`（スクリプト: `scripts/e2e/openwebui-docker.sh`）
- オンボーディングウィザード（TTY、完全なscaffolding）: `pnpm test:docker:onboard`（スクリプト: `scripts/e2e/onboard-docker.sh`）
- Gatewayネットワーキング（2コンテナ、WS認証 + health）: `pnpm test:docker:gateway-network`（スクリプト: `scripts/e2e/gateway-network-docker.sh`）
- MCPチャネルブリッジ（seed済みGateway + stdioブリッジ + 生のClaude notification-frameスモーク）: `pnpm test:docker:mcp-channels`（スクリプト: `scripts/e2e/mcp-channels-docker.sh`）
- Plugins（インストールスモーク + `/plugin` alias + Claude-bundle再起動セマンティクス）: `pnpm test:docker:plugins`（スクリプト: `scripts/e2e/plugins-docker.sh`）

live-model Dockerランナーは、現在のチェックアウトも読み取り専用でbind mountし、コンテナ内の一時workdirへステージします。これにより、ランタイムイメージをスリムに保ちつつ、手元の正確なソース/設定に対してVitestを実行できます。
ステージングでは、`.pnpm-store`、`.worktrees`、`__openclaw_vitest__`、およびアプリローカルの`.build`やGradle出力ディレクトリのような、大きなローカル専用キャッシュやアプリビルド出力をスキップするため、Docker live実行でマシン固有アーティファクトのコピーに何分も費やしません。
また、`OPENCLAW_SKIP_CHANNELS=1`も設定するため、gateway live probeがコンテナ内で実際のTelegram/Discordなどのチャネルワーカーを起動しません。
`test:docker:live-models`は引き続き`pnpm test:live`を実行するため、そのDockerレーンからgateway liveカバレッジを絞り込む、または除外したい場合は、`OPENCLAW_LIVE_GATEWAY_*`も渡してください。
`test:docker:openwebui`はより高レベルの互換性スモークです。OpenAI互換HTTPエンドポイントを有効にした
OpenClaw gatewayコンテナを起動し、そのgatewayに対して固定版のOpen WebUIコンテナを起動し、Open WebUI経由でサインインし、`/api/models`が`openclaw/default`を公開していることを確認し、その後Open WebUIの`/api/chat/completions`プロキシ経由で実際のチャットリクエストを送信します。
初回実行は、DockerがOpen WebUIイメージをpullする必要があったり、Open WebUI自身のコールドスタートセットアップ完了が必要だったりするため、目に見えて遅くなることがあります。
このレーンは使用可能なlive modelキーを前提としており、Docker化実行では`OPENCLAW_PROFILE_FILE`
（デフォルトは`~/.profile`）がそれを提供する主な方法です。
成功した実行では、`{ "ok": true, "model":
"openclaw/default", ... }`のような小さなJSONペイロードが出力されます。
`test:docker:mcp-channels`は意図的に決定的であり、実際のTelegram、Discord、iMessageアカウントは必要ありません。これはseed済みGateway
コンテナを起動し、`openclaw mcp serve`を起動する2つ目のコンテナを開始し、その後、ルーティングされた会話検出、トランスクリプト読み取り、添付メタデータ、
live event queue動作、外向き送信ルーティング、および実際のstdio MCPブリッジ上でのClaude風チャネル +
権限通知を検証します。通知チェックでは生のstdio MCPフレームを直接検査するため、このスモークは特定クライアントSDKがたまたま表面化するものではなく、
ブリッジが実際に出力する内容を検証します。

手動ACP平文スレッドスモーク（CIではない）:

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- このスクリプトは回帰/デバッグワークフロー用に維持してください。ACPスレッドルーティングの検証で再び必要になる可能性があるため、削除しないでください。

便利なenv var:

- `OPENCLAW_CONFIG_DIR=...`（デフォルト: `~/.openclaw`）を`/home/node/.openclaw`にマウント
- `OPENCLAW_WORKSPACE_DIR=...`（デフォルト: `~/.openclaw/workspace`）を`/home/node/.openclaw/workspace`にマウント
- `OPENCLAW_PROFILE_FILE=...`（デフォルト: `~/.profile`）を`/home/node/.profile`にマウントし、テスト実行前に読み込み
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`（デフォルト: `~/.cache/openclaw/docker-cli-tools`）を`/home/node/.npm-global`にマウントし、Docker内でCLIインストールをキャッシュ
- `$HOME`配下の外部CLI認証ディレクトリ/ファイルは、`/host-auth...`配下に読み取り専用でマウントされ、その後テスト開始前に`/home/node/...`へコピーされます
  - デフォルトのディレクトリ: `.minimax`
  - デフォルトのファイル: `~/.codex/auth.json`、`~/.codex/config.toml`、`.claude.json`、`~/.claude/.credentials.json`、`~/.claude/settings.json`、`~/.claude/settings.local.json`
  - providerを絞り込んだ実行では、`OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`から推定された必要なディレクトリ/ファイルだけをマウント
  - 手動で上書きするには`OPENCLAW_DOCKER_AUTH_DIRS=all`、`OPENCLAW_DOCKER_AUTH_DIRS=none`、または`OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`のようなカンマ区切りリストを使用
- 実行を絞り込むには`OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- コンテナ内でproviderを絞り込むには`OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- 再ビルド不要の再実行で既存の`openclaw:local-live`イメージを再利用するには`OPENCLAW_SKIP_DOCKER_BUILD=1`
- 認証情報がprofile store由来であることを保証するには`OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`（envではなく）
- Open WebUIスモークでgatewayが公開するモデルを選ぶには`OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUIスモークで使うnonce確認プロンプトを上書きするには`OPENCLAW_OPENWEBUI_PROMPT=...`
- 固定されたOpen WebUIイメージタグを上書きするには`OPENWEBUI_IMAGE=...`

## ドキュメントの健全性確認

ドキュメント編集後は次を実行してください: `pnpm check:docs`。
ページ内見出しチェックも必要な場合は、完全なMintlifyアンカー検証を実行してください: `pnpm docs:check-links:anchors`。

## オフライン回帰（CI対応）

これらは、実際のproviderなしで行う「実パイプライン」回帰です。

- Gateway tool calling（mock OpenAI、実際のgateway + agent loop）: `src/gateway/gateway.test.ts`（ケース: "runs a mock OpenAI tool call end-to-end via gateway agent loop"）
- Gatewayウィザード（WS `wizard.start`/`wizard.next`、設定 + authの書き込みを強制）: `src/gateway/gateway.test.ts`（ケース: "runs wizard over ws and writes auth token config"）

## エージェント信頼性eval（Skills）

すでに、いくつかのCI対応テストが「エージェント信頼性eval」のように振る舞います。

- 実際のgateway + agent loopを通したmock tool-calling（`src/gateway/gateway.test.ts`）。
- セッション配線と設定効果を検証するエンドツーエンドのウィザードフロー（`src/gateway/gateway.test.ts`）。

Skillsについてまだ不足しているもの（[Skills](/tools/skills)を参照）:

- **意思決定:** Skillsがプロンプトに列挙されているとき、エージェントは正しいskillを選ぶか（または無関係なものを避けるか）？
- **準拠性:** エージェントは使用前に`SKILL.md`を読み、必要な手順/引数に従うか？
- **ワークフローコントラクト:** ツール順序、セッション履歴の引き継ぎ、sandbox境界を検証するマルチターンシナリオ。

今後のevalは、まず決定的であるべきです。

- mock providerを使って、tool呼び出し + 順序、skillファイル読み取り、セッション配線を検証するシナリオランナー。
- skillに焦点を当てた小規模シナリオ群（使う/避ける、ゲーティング、プロンプトインジェクション）。
- CI対応スイートが整ってからのみ、任意のlive eval（オプトイン、envでゲート）。

## コントラクトテスト（pluginおよびchannelの形状）

コントラクトテストは、登録されているすべてのpluginとchannelがその
インターフェースコントラクトに準拠していることを検証します。検出されたすべてのpluginを反復し、
形状と動作に関する一連のアサーションを実行します。デフォルトの`pnpm test` unitレーンは、これらの共有seamおよびスモークファイルを意図的に
スキップします。共有channelまたはproviderサーフェスに触れた場合は、コントラクトコマンドを明示的に実行してください。

### コマンド

- すべてのコントラクト: `pnpm test:contracts`
- channelコントラクトのみ: `pnpm test:contracts:channels`
- providerコントラクトのみ: `pnpm test:contracts:plugins`

### Channelコントラクト

`src/channels/plugins/contracts/*.contract.test.ts`にあります。

- **plugin** - 基本plugin形状（id、name、capabilities）
- **setup** - セットアップウィザードのコントラクト
- **session-binding** - セッションバインディングの挙動
- **outbound-payload** - メッセージペイロード構造
- **inbound** - inboundメッセージ処理
- **actions** - channel actionハンドラー
- **threading** - スレッドID処理
- **directory** - ディレクトリ/roster API
- **group-policy** - グループポリシーの適用

### Provider statusコントラクト

`src/plugins/contracts/*.contract.test.ts`にあります。

- **status** - Channel status probe
- **registry** - Plugin registry shape

### Providerコントラクト

`src/plugins/contracts/*.contract.test.ts`にあります。

- **auth** - 認証フローのコントラクト
- **auth-choice** - 認証方式の選択
- **catalog** - モデルカタログAPI
- **discovery** - Plugin discovery
- **loader** - Plugin loading
- **runtime** - Provider runtime
- **shape** - Plugin shape/interface
- **wizard** - セットアップウィザード

### 実行すべきタイミング

- plugin-sdkのexportまたはsubpathを変更した後
- channelまたはprovider pluginを追加または変更した後
- plugin登録またはdiscoveryをリファクタリングした後

コントラクトテストはCIで実行され、実際のAPIキーは不要です。

## 回帰を追加する際のガイダンス

liveで見つかったprovider/modelの問題を修正したとき:

- 可能ならCI対応の回帰を追加してください（mock/stub provider、または正確なrequest-shape変換をキャプチャ）
- 本質的にlive専用の場合（レート制限、認証ポリシー）は、liveテストを狭く保ち、env varでオプトインにしてください
- バグを検出できる最小のレイヤーを対象にすることを優先してください:
  - providerのリクエスト変換/replayバグ → direct modelsテスト
  - gatewayのsession/history/toolパイプラインバグ → gateway liveスモークまたはCI対応gateway mockテスト
- SecretRef走査のガードレール:
  - `src/secrets/exec-secret-ref-id-parity.test.ts`は、registryメタデータ（`listSecretTargetRegistryEntries()`）からSecretRefクラスごとに1つのサンプル対象を導出し、走査セグメントのexec IDが拒否されることを検証します。
  - `src/secrets/target-registry-data.ts`に新しい`includeInPlan` SecretRef target familyを追加する場合は、そのテスト内の`classifyTargetClass`を更新してください。このテストは、分類されていないtarget IDに対して意図的に失敗するため、新しいクラスが黙ってスキップされることを防ぎます。
