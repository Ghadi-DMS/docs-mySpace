---
read_when:
    - Gateway WSクライアントを実装または更新する場合
    - プロトコルの不一致や接続失敗をデバッグする場合
    - プロトコルスキーマ/モデルを再生成する場合
summary: 'Gateway WebSocketプロトコル: ハンドシェイク、フレーム、バージョニング'
title: Gatewayプロトコル
x-i18n:
    generated_at: "2026-04-11T02:44:57Z"
    model: gpt-5.4
    provider: openai
    source_hash: 83c820c46d4803d571c770468fd6782619eaa1dca253e156e8087dec735c127f
    source_path: gateway/protocol.md
    workflow: 15
---

# Gatewayプロトコル（WebSocket）

Gateway WSプロトコルは、OpenClawの**単一のコントロールプレーン + ノードトランスポート**です。すべてのクライアント（CLI、web UI、macOSアプリ、iOS/Androidノード、ヘッドレスノード）はWebSocket経由で接続し、ハンドシェイク時に自分の**role** + **scope**を宣言します。

## トランスポート

- WebSocket、JSONペイロードを持つテキストフレーム。
- 最初のフレームは**必ず**`connect`リクエストでなければなりません。

## ハンドシェイク（connect）

Gateway → Client（接続前チャレンジ）:

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

Client → Gateway:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Gateway → Client:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": { "type": "hello-ok", "protocol": 3, "policy": { "tickIntervalMs": 15000 } }
}
```

デバイストークンが発行されると、`hello-ok`には次も含まれます。

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

信頼されたbootstrap handoffの間、`hello-ok.auth`には`deviceTokens`内に追加の制限付きroleエントリが含まれることもあります。

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

組み込みのnode/operator bootstrapフローでは、主要なnodeトークンは`scopes: []`のままで、引き渡されるoperatorトークンはbootstrap operator allowlist（`operator.approvals`、`operator.read`、`operator.talk.secrets`、`operator.write`）に制限されたままです。bootstrap scopeチェックは引き続きrole接頭辞付きです。operatorエントリはoperatorリクエストだけを満たし、operator以外のroleは引き続き自分自身のrole接頭辞配下のscopeが必要です。

### ノードの例

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 3,
    "maxProtocol": 3,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

## フレーミング

- **Request**: `{type:"req", id, method, params}`
- **Response**: `{type:"res", id, ok, payload|error}`
- **Event**: `{type:"event", event, payload, seq?, stateVersion?}`

副作用を持つメソッドには**idempotency key**が必要です（スキーマを参照）。

## role + scope

### role

- `operator` = コントロールプレーンクライアント（CLI/UI/automation）。
- `node` = capability host（camera/screen/canvas/system.run）。

### scope（operator）

一般的なscope:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`includeSecrets: true`付きの`talk.config`には`operator.talk.secrets`（または`operator.admin`）が必要です。

プラグイン登録されたGateway RPCメソッドは独自のoperator scopeを要求できますが、予約済みのコア管理者接頭辞（`config.*`、`exec.approvals.*`、`wizard.*`、`update.*`）は常に`operator.admin`に解決されます。

メソッドscopeは最初のゲートにすぎません。`chat.send`経由で到達する一部のスラッシュコマンドには、その上にさらに厳しいコマンドレベルのチェックが適用されます。たとえば、永続的な`/config set`と`/config unset`の書き込みには`operator.admin`が必要です。

`node.pair.approve`にも、ベースメソッドscopeに加えて追加の承認時scopeチェックがあります。

- コマンドなしのリクエスト: `operator.pairing`
- non-execノードコマンド付きのリクエスト: `operator.pairing` + `operator.write`
- `system.run`、`system.run.prepare`、または`system.which`を含むリクエスト:
  `operator.pairing` + `operator.admin`

### caps/commands/permissions（node）

ノードは接続時にcapability claimを宣言します。

- `caps`: 高レベルのcapabilityカテゴリ。
- `commands`: invoke用のコマンドallowlist。
- `permissions`: 粒度の細かいトグル（例: `screen.record`、`camera.capture`）。

Gatewayはこれらを**claim**として扱い、サーバー側allowlistを強制します。

## プレゼンス

- `system-presence`はデバイスIDをキーとするエントリを返します。
- プレゼンスエントリには`deviceId`、`roles`、`scopes`が含まれるため、UIはそのデバイスが**operator**と**node**の両方として接続していても1行で表示できます。

## 一般的なRPCメソッドファミリー

このページは生成された完全ダンプではありませんが、公開WSサーフェスは上記のハンドシェイク/認証の例よりも広範です。以下は、現在Gatewayが公開している主なメソッドファミリーです。

`hello-ok.features.methods`は、`src/gateway/server-methods-list.ts`と、読み込まれたプラグイン/チャンネルのメソッドエクスポートから構築される保守的なディスカバリーリストです。これは機能検出として扱ってください。`src/gateway/server-methods/*.ts`で実装されている、呼び出し可能なすべてのヘルパーの生成ダンプではありません。

### システムとID

- `health`は、キャッシュ済みまたは新たにプローブしたGatewayのhealthスナップショットを返します。
- `status`は`/status`形式のGateway概要を返します。機密フィールドはadmin scopeを持つoperatorクライアントにのみ含まれます。
- `gateway.identity.get`は、relayおよびpairingフローで使用されるGatewayデバイスIDを返します。
- `system-presence`は、接続中のoperator/nodeデバイスの現在のプレゼンススナップショットを返します。
- `system-event`はシステムイベントを追加し、プレゼンスコンテキストを更新/ブロードキャストできます。
- `last-heartbeat`は、最新の永続化されたheartbeatイベントを返します。
- `set-heartbeats`は、Gateway上のheartbeat処理を切り替えます。

### モデルと使用量

- `models.list`は、ランタイムで許可されたモデルカタログを返します。
- `usage.status`は、プロバイダーの使用ウィンドウ/残りクォータの概要を返します。
- `usage.cost`は、指定した日付範囲の集計済みコスト使用量サマリーを返します。
- `doctor.memory.status`は、アクティブなデフォルトエージェントワークスペースにおけるvector-memory / embedding readinessを返します。
- `sessions.usage`は、セッションごとの使用量サマリーを返します。
- `sessions.usage.timeseries`は、1つのセッションの時系列使用量を返します。
- `sessions.usage.logs`は、1つのセッションの使用量ログエントリを返します。

### チャンネルとログインヘルパー

- `channels.status`は、組み込み + 同梱チャンネル/プラグインのステータスサマリーを返します。
- `channels.logout`は、そのチャンネルがlogoutをサポートしている場合、特定のチャンネル/アカウントをログアウトします。
- `web.login.start`は、現在のQR対応webチャンネルプロバイダー向けにQR/webログインフローを開始します。
- `web.login.wait`は、そのQR/webログインフローの完了を待ち、成功したらチャンネルを起動します。
- `push.test`は、登録済みiOSノードにテストAPNs pushを送信します。
- `voicewake.get`は、保存されているウェイクワードトリガーを返します。
- `voicewake.set`は、ウェイクワードトリガーを更新し、変更をブロードキャストします。

### メッセージングとログ

- `send`は、chat runnerの外で、チャンネル/アカウント/スレッド対象への送信を行う直接のアウトバウンド配信RPCです。
- `logs.tail`は、設定済みGatewayファイルログのtailを、cursor/limitおよびmax-byte制御付きで返します。

### TalkとTTS

- `talk.config`は、有効なTalk設定ペイロードを返します。`includeSecrets`には`operator.talk.secrets`（または`operator.admin`）が必要です。
- `talk.mode`は、WebChat/Control UIクライアント向けに現在のTalkモード状態を設定/ブロードキャストします。
- `talk.speak`は、アクティブなTalk speech providerを通じて音声を合成します。
- `tts.status`は、TTSの有効状態、アクティブなプロバイダー、フォールバックプロバイダー、プロバイダー設定状態を返します。
- `tts.providers`は、表示可能なTTSプロバイダー一覧を返します。
- `tts.enable`と`tts.disable`は、TTS設定状態を切り替えます。
- `tts.setProvider`は、優先TTSプロバイダーを更新します。
- `tts.convert`は、単発のtext-to-speech変換を実行します。

### Secret、設定、更新、ウィザード

- `secrets.reload`は、アクティブなSecretRefを再解決し、完全成功時のみランタイムsecret状態を切り替えます。
- `secrets.resolve`は、特定のコマンド/ターゲットセットに対するコマンド対象secret割り当てを解決します。
- `config.get`は、現在の設定スナップショットとハッシュを返します。
- `config.set`は、検証済みの設定ペイロードを書き込みます。
- `config.patch`は、部分的な設定更新をマージします。
- `config.apply`は、完全な設定ペイロードを検証して置き換えます。
- `config.schema`は、Control UIおよびCLIツールで使用されるライブ設定スキーマペイロードを返します。これには、スキーマ、`uiHints`、バージョン、生成メタデータが含まれ、ランタイムで読み込める場合はプラグイン + チャンネルのスキーマメタデータも含まれます。スキーマには、ネストしたオブジェクト、ワイルドカード、配列項目、`anyOf` / `oneOf` / `allOf`の分岐で一致するフィールドドキュメントが存在する場合、UIで使われるのと同じラベルおよびヘルプテキストから導出されたフィールド`title` / `description`メタデータが含まれます。
- `config.schema.lookup`は、1つの設定パスに対するパススコープのlookupペイロードを返します。これには、正規化されたパス、浅いスキーマノード、一致したhint + `hintPath`、およびUI/CLIドリルダウン用の直下の子サマリーが含まれます。
  - Lookupスキーマノードは、ユーザー向けドキュメントと一般的な検証フィールドを保持します: `title`、`description`、`type`、`enum`、`const`、`format`、`pattern`、数値/文字列/配列/オブジェクトの境界、および`additionalProperties`、`deprecated`、`readOnly`、`writeOnly`のようなbooleanフラグ。
  - 子サマリーには、`key`、正規化された`path`、`type`、`required`、`hasChildren`、および一致した`hint` / `hintPath`が含まれます。
- `update.run`は、Gateway更新フローを実行し、更新自体が成功した場合にのみ再起動をスケジュールします。
- `wizard.start`、`wizard.next`、`wizard.status`、`wizard.cancel`は、WS RPC経由でオンボーディングウィザードを公開します。

### 既存の主要ファミリー

#### エージェントとワークスペースヘルパー

- `agents.list`は、設定済みエージェントエントリを返します。
- `agents.create`、`agents.update`、`agents.delete`は、エージェントレコードとワークスペース配線を管理します。
- `agents.files.list`、`agents.files.get`、`agents.files.set`は、エージェント向けに公開されるbootstrapワークスペースファイルを管理します。
- `agent.identity.get`は、エージェントまたはセッションに対する有効なassistant IDを返します。
- `agent.wait`は、実行完了を待ち、利用可能な場合は終端スナップショットを返します。

#### セッション制御

- `sessions.list`は、現在のセッションインデックスを返します。
- `sessions.subscribe`と`sessions.unsubscribe`は、現在のWSクライアントに対するセッション変更イベントのサブスクリプションを切り替えます。
- `sessions.messages.subscribe`と`sessions.messages.unsubscribe`は、1つのセッションに対するtranscript/messageイベントのサブスクリプションを切り替えます。
- `sessions.preview`は、特定のセッションキーに対する制限付きtranscriptプレビューを返します。
- `sessions.resolve`は、セッションターゲットを解決または正規化します。
- `sessions.create`は、新しいセッションエントリを作成します。
- `sessions.send`は、既存のセッションにメッセージを送信します。
- `sessions.steer`は、アクティブなセッションに対するinterrupt-and-steerバリアントです。
- `sessions.abort`は、セッションのアクティブな処理を中止します。
- `sessions.patch`は、セッションメタデータ/overrideを更新します。
- `sessions.reset`、`sessions.delete`、`sessions.compact`は、セッションメンテナンスを実行します。
- `sessions.get`は、保存されている完全なセッション行を返します。
- chat実行では、引き続き`chat.history`、`chat.send`、`chat.abort`、`chat.inject`を使用します。
- `chat.history`はUIクライアント向けに表示用に正規化されます。インラインdirectiveタグは表示テキストから除去され、プレーンテキストのtool-call XMLペイロード（`<tool_call>...</tool_call>`、`<function_call>...</function_call>`、`<tool_calls>...</tool_calls>`、`<function_calls>...</function_calls>`、および切り詰められたtool-callブロックを含む）や漏れたASCII/全角のモデル制御トークンは除去され、正確に`NO_REPLY` / `no_reply`である純粋なsilent-token assistant行は省略され、過大な行はプレースホルダーに置き換えられることがあります。

#### デバイスpairingとデバイストークン

- `device.pair.list`は、保留中および承認済みのペアリング済みデバイスを返します。
- `device.pair.approve`、`device.pair.reject`、`device.pair.remove`は、デバイスペアリングレコードを管理します。
- `device.token.rotate`は、承認済みのroleおよびscopeの範囲内でペアリング済みデバイストークンをローテーションします。
- `device.token.revoke`は、ペアリング済みデバイストークンを失効させます。

#### ノードpairing、invoke、および保留中の作業

- `node.pair.request`、`node.pair.list`、`node.pair.approve`、`node.pair.reject`、`node.pair.verify`は、ノードpairingとbootstrap検証を扱います。
- `node.list`と`node.describe`は、既知/接続済みノードの状態を返します。
- `node.rename`は、ペアリング済みノードのラベルを更新します。
- `node.invoke`は、接続済みノードにコマンドを転送します。
- `node.invoke.result`は、invokeリクエストの結果を返します。
- `node.event`は、ノード起点のイベントをGatewayへ戻します。
- `node.canvas.capability.refresh`は、スコープ付きcanvas-capabilityトークンを更新します。
- `node.pending.pull`と`node.pending.ack`は、接続済みノード向けのキューAPIです。
- `node.pending.enqueue`と`node.pending.drain`は、オフライン/切断中ノード向けの永続的な保留作業を管理します。

#### 承認ファミリー

- `exec.approval.request`、`exec.approval.get`、`exec.approval.list`、`exec.approval.resolve`は、単発のexec承認リクエストと、保留中の承認のlookup/replayを扱います。
- `exec.approval.waitDecision`は、1件の保留中exec承認を待機し、最終決定を返します（タイムアウト時は`null`）。
- `exec.approvals.get`と`exec.approvals.set`は、Gatewayのexec承認ポリシースナップショットを管理します。
- `exec.approvals.node.get`と`exec.approvals.node.set`は、ノードrelayコマンド経由でノードローカルexec承認ポリシーを管理します。
- `plugin.approval.request`、`plugin.approval.list`、`plugin.approval.waitDecision`、`plugin.approval.resolve`は、プラグイン定義の承認フローを扱います。

#### その他の主要ファミリー

- automation:
  - `wake`は、即時または次回heartbeat時のwakeテキスト注入をスケジュールします
  - `cron.list`、`cron.status`、`cron.add`、`cron.update`、`cron.remove`、`cron.run`、`cron.runs`
- skills/tools: `commands.list`、`skills.*`、`tools.catalog`、`tools.effective`

### 一般的なイベントファミリー

- `chat`: `chat.inject`やその他のtranscript専用chatイベントなどのUI chat更新。
- `session.message`と`session.tool`: サブスクライブされたセッションのtranscript/event-stream更新。
- `sessions.changed`: セッションインデックスまたはメタデータが変更されたとき。
- `presence`: システムプレゼンススナップショット更新。
- `tick`: 定期的なkeepalive / livenessイベント。
- `health`: Gateway healthスナップショット更新。
- `heartbeat`: heartbeatイベントストリーム更新。
- `cron`: cron実行/ジョブ変更イベント。
- `shutdown`: Gatewayシャットダウン通知。
- `node.pair.requested` / `node.pair.resolved`: ノードpairingライフサイクル。
- `node.invoke.request`: ノードinvokeリクエストのブロードキャスト。
- `device.pair.requested` / `device.pair.resolved`: ペアリング済みデバイスのライフサイクル。
- `voicewake.changed`: ウェイクワードトリガー設定が変更されたとき。
- `exec.approval.requested` / `exec.approval.resolved`: exec承認ライフサイクル。
- `plugin.approval.requested` / `plugin.approval.resolved`: プラグイン承認ライフサイクル。

### ノードヘルパーメソッド

- ノードは、auto-allowチェックのために、現在のskill実行ファイル一覧を取得する`skills.bins`を呼び出せます。

### operatorヘルパーメソッド

- operatorは、エージェントのランタイムコマンド一覧を取得するために`commands.list`（`operator.read`）を呼び出せます。
  - `agentId`は省略可能です。省略するとデフォルトのエージェントワークスペースを読み取ります。
  - `scope`は、主`name`がどのサーフェスを対象にするかを制御します:
    - `text`は、先頭の`/`を除いた主テキストコマンドトークンを返します
    - `native`およびデフォルトの`both`パスは、利用可能な場合にプロバイダー対応のnative名を返します
  - `textAliases`は、`/model`や`/m`のような正確なスラッシュエイリアスを保持します。
  - `nativeName`は、存在する場合にプロバイダー対応のnativeコマンド名を保持します。
  - `provider`は省略可能で、native命名とnativeプラグインコマンドの可用性にのみ影響します。
  - `includeArgs=false`は、シリアライズされた引数メタデータをレスポンスから省略します。
- operatorは、エージェントのランタイムツールカタログを取得するために`tools.catalog`（`operator.read`）を呼び出せます。レスポンスには、グループ化されたツールとprovenanceメタデータが含まれます。
  - `source`: `core`または`plugin`
  - `pluginId`: `source="plugin"`のときのプラグイン所有者
  - `optional`: プラグインツールがoptionalかどうか
- operatorは、セッションに対するランタイム有効ツール一覧を取得するために`tools.effective`（`operator.read`）を呼び出せます。
  - `sessionKey`は必須です。
  - Gatewayは、呼び出し元から提供された認証や配信コンテキストを受け入れるのではなく、サーバー側でセッションから信頼済みランタイムコンテキストを導出します。
  - レスポンスはセッションスコープで、コア、プラグイン、チャンネルツールを含め、現在アクティブな会話で今使えるものを反映します。
- operatorは、エージェントの表示可能なskill一覧を取得するために`skills.status`（`operator.read`）を呼び出せます。
  - `agentId`は省略可能です。省略するとデフォルトのエージェントワークスペースを読み取ります。
  - レスポンスには、raw secret値を公開せずに、適格性、不足要件、設定チェック、サニタイズ済みインストールオプションが含まれます。
- operatorは、ClawHub discoveryメタデータのために`skills.search`と`skills.detail`（`operator.read`）を呼び出せます。
- operatorは、`skills.install`（`operator.admin`）を2つのモードで呼び出せます。
  - ClawHubモード: `{ source: "clawhub", slug, version?, force? }`は、skillフォルダーをデフォルトのエージェントワークスペース`skills/`ディレクトリにインストールします。
  - Gateway installerモード: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`は、Gatewayホスト上で宣言された`metadata.openclaw.install`アクションを実行します。
- operatorは、`skills.update`（`operator.admin`）を2つのモードで呼び出せます。
  - ClawHubモードは、デフォルトのエージェントワークスペース内で1つの追跡済みslugまたはすべての追跡済みClawHubインストールを更新します。
  - Configモードは、`enabled`、`apiKey`、`env`のような`skills.entries.<skillKey>`値をパッチします。

## Exec承認

- execリクエストに承認が必要な場合、Gatewayは`exec.approval.requested`をブロードキャストします。
- operatorクライアントは`exec.approval.resolve`を呼び出して解決します（`operator.approvals` scopeが必要）。
- `host=node`の場合、`exec.approval.request`には`systemRunPlan`（正規化された`argv`/`cwd`/`rawCommand`/セッションメタデータ）が必須です。`systemRunPlan`が欠けているリクエストは拒否されます。
- 承認後、転送される`node.invoke system.run`呼び出しは、その正規化された`systemRunPlan`を権威あるcommand/cwd/sessionコンテキストとして再利用します。
- 呼び出し元がprepareから最終的な承認済み`system.run`転送までの間に`command`、`rawCommand`、`cwd`、`agentId`、または`sessionKey`を変更した場合、Gatewayは変更されたペイロードを信頼せずに実行を拒否します。

## エージェント配信フォールバック

- `agent`リクエストには、アウトバウンド配信を要求するための`deliver=true`を含めることができます。
- `bestEffortDeliver=false`は厳格な動作を維持します。未解決または内部専用の配信先ターゲットは`INVALID_REQUEST`を返します。
- `bestEffortDeliver=true`は、外部配信可能ルートを解決できない場合（たとえば内部/webchatセッションや曖昧なマルチチャンネル設定）に、セッション専用実行へのフォールバックを許可します。

## バージョニング

- `PROTOCOL_VERSION`は`src/gateway/protocol/schema.ts`にあります。
- クライアントは`minProtocol` + `maxProtocol`を送信し、サーバーは不一致を拒否します。
- スキーマ + モデルはTypeBox定義から生成されます:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

## 認証

- 共有シークレットのGateway認証では、設定された認証モードに応じて`connect.params.auth.token`または`connect.params.auth.password`を使用します。
- Tailscale Serve（`gateway.auth.allowTailscale: true`）や、loopback以外での`gateway.auth.mode: "trusted-proxy"`のようなID付きモードでは、`connect.params.auth.*`ではなくリクエストヘッダーから接続認証チェックを満たします。
- private-ingressの`gateway.auth.mode: "none"`は共有シークレット接続認証を完全にスキップします。このモードを公開/信頼できないingressに公開しないでください。
- pairing後、Gatewayは接続role + scopeにスコープされた**device token**を発行します。これは`hello-ok.auth.deviceToken`で返され、クライアントは今後の接続のために永続化する必要があります。
- クライアントは、接続成功後に主`hello-ok.auth.deviceToken`を永続化する必要があります。
- その**保存済み**device tokenで再接続する場合、そのトークンに対して承認済みの保存scopeセットも再利用する必要があります。これにより、すでに付与されていたread/probe/statusアクセスが保持され、再接続時に暗黙のadmin-only scopeへと静かに縮小することを防げます。
- 通常の接続認証の優先順位は、まず明示的な共有token/password、次に明示的な`deviceToken`、次に保存済みのデバイス単位token、最後にbootstrap tokenです。
- 追加の`hello-ok.auth.deviceTokens`エントリはbootstrap handoff tokenです。`wss://`やloopback/local pairingのような信頼できるトランスポートでbootstrap認証を使用した場合にのみ永続化してください。
- クライアントが明示的な`deviceToken`または明示的な`scopes`を指定した場合、その呼び出し元が要求したscopeセットが引き続き権威を持ちます。キャッシュ済みscopeは、クライアントが保存済みのデバイス単位tokenを再利用している場合にのみ再利用されます。
- デバイストークンは`device.token.rotate`および`device.token.revoke`でローテーション/失効できます（`operator.pairing` scopeが必要）。
- トークンの発行/ローテーションは、そのデバイスのpairingエントリに記録された承認済みroleセットの範囲内に制限されます。トークンのローテーションによって、そのデバイスをpairing承認で一度も許可されていないroleへ拡張することはできません。
- ペアリング済みデバイストークンセッションでは、呼び出し元が`operator.admin`も持っていない限り、デバイス管理は自己スコープです。adminでない呼び出し元は、自分**自身**のデバイスエントリに対してのみremove/revoke/rotateできます。
- `device.token.rotate`は、要求されたoperator scopeセットを呼び出し元の現在のセッションscopeに対してもチェックします。adminでない呼び出し元は、現在保持しているより広いoperator scopeセットへトークンをローテーションできません。
- 認証失敗には、`error.details.code`と回復ヒントが含まれます:
  - `error.details.canRetryWithDeviceToken`（boolean）
  - `error.details.recommendedNextStep`（`retry_with_device_token`、`update_auth_configuration`、`update_auth_credentials`、`wait_then_retry`、`review_auth_configuration`）
- `AUTH_TOKEN_MISMATCH`に対するクライアント動作:
  - 信頼されたクライアントは、キャッシュ済みのデバイス単位tokenで1回だけ制限付き再試行を試みることができます。
  - その再試行が失敗した場合、クライアントは自動再接続ループを停止し、operatorに必要な対応を案内するべきです。

## デバイスID + pairing

- ノードは、キーペアのフィンガープリントから導出された安定したデバイスID（`device.id`）を含める必要があります。
- Gatewayは、デバイスごと + roleごとにトークンを発行します。
- local loopbackの自動承認が有効になっていない限り、新しいデバイスIDにはpairing承認が必要です。
- pairing自動承認は、直接のlocal loopback接続を中心に設計されています。
- OpenClawには、信頼された共有シークレットヘルパーフロー向けの、限定的なバックエンド/コンテナローカル自己接続パスもあります。
- 同一ホストのtailnetまたはLAN接続は、pairingの観点では引き続きリモートとして扱われ、承認が必要です。
- すべてのWSクライアントは、`connect`時に`device` IDを含める必要があります（operator + node）。
  Control UIがこれを省略できるのは、次のモードのみです:
  - localhost専用の安全でないHTTP互換性向けの`gateway.controlUi.allowInsecureAuth=true`
  - 成功した`gateway.auth.mode: "trusted-proxy"`のoperator Control UI認証
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true`（緊急避難用、重大なセキュリティ低下）
- すべての接続は、サーバーから提供された`connect.challenge` nonceに署名する必要があります。

### デバイス認証移行診断

以前のchallenge前署名動作をまだ使っているレガシークライアント向けに、`connect`は`error.details.reason`の安定した値とともに、`error.details.code`配下で`DEVICE_AUTH_*`詳細コードを返すようになりました。

一般的な移行失敗:

| メッセージ                  | details.code                     | details.reason           | 意味                                                   |
| --------------------------- | -------------------------------- | ------------------------ | ------------------------------------------------------ |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | クライアントが`device.nonce`を省略した（または空文字を送信した）。 |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | クライアントが古い/誤ったnonceで署名した。             |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | 署名ペイロードがv2ペイロードと一致しない。             |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | 署名済みタイムスタンプが許容skewの範囲外である。       |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id`が公開鍵フィンガープリントと一致しない。    |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | 公開鍵形式/正規化に失敗した。                          |

移行の目標:

- 常に`connect.challenge`を待機してください。
- サーバーnonceを含むv2ペイロードに署名してください。
- 同じnonceを`connect.params.device.nonce`に送信してください。
- 推奨される署名ペイロードは`v3`で、device/client/role/scopes/token/nonceフィールドに加えて`platform`と`deviceFamily`を束縛します。
- レガシー`v2`署名も互換性のため引き続き受け入れられますが、ペアリング済みデバイスメタデータのpinningは、再接続時のコマンドポリシーを引き続き制御します。

## TLS + pinning

- WS接続ではTLSがサポートされます。
- クライアントは、必要に応じてGateway証明書フィンガープリントをpinできます（`gateway.tls`設定、および`gateway.remote.tlsFingerprint`またはCLIの`--tls-fingerprint`を参照）。

## スコープ

このプロトコルは、**完全なGateway API**（status、channels、models、chat、agent、sessions、nodes、approvalsなど）を公開します。正確なサーフェスは`src/gateway/protocol/schema.ts`内のTypeBoxスキーマで定義されています。
