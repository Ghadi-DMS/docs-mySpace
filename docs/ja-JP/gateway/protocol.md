---
read_when:
    - Gateway WSクライアントの実装または更新
    - プロトコルの不一致や接続失敗のデバッグ
    - プロトコルスキーマ／モデルの再生成
summary: Gateway WebSocketプロトコル：ハンドシェイク、フレーム、バージョニング
title: Gatewayプロトコル
x-i18n:
    generated_at: "2026-04-18T04:40:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4f0eebcfdd8c926c90b4753a6d96c59e3134ddb91740f65478f11eb75be85e41
    source_path: gateway/protocol.md
    workflow: 15
---

# Gatewayプロトコル（WebSocket）

Gateway WSプロトコルは、OpenClawの**単一のコントロールプレーン + Nodeトランスポート**です。すべてのクライアント（CLI、Web UI、macOSアプリ、iOS/Android Node、ヘッドレスNode）はWebSocket経由で接続し、ハンドシェイク時に自身の**role** + **scope**を宣言します。

## トランスポート

- WebSocket、JSONペイロードを持つテキストフレーム。
- 最初のフレームは**必ず**`connect`リクエストである必要があります。

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
  "payload": {
    "type": "hello-ok",
    "protocol": 3,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

`server`、`features`、`snapshot`、`policy`はすべてスキーマ（`src/gateway/protocol/schema/frames.ts`）で必須です。`canvasHostUrl`は任意です。`auth`は利用可能な場合にネゴシエートされたrole/scopesを報告し、Gatewayが発行した場合は`deviceToken`も含みます。

デバイストークンが発行されない場合でも、`hello-ok.auth`はネゴシエートされた権限を報告できます。

```json
{
  "auth": {
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

デバイストークンが発行された場合、`hello-ok`には次も含まれます。

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

信頼されたブートストラップのハンドオフ中、`hello-ok.auth`には`deviceTokens`内に追加の境界付きroleエントリが含まれることもあります。

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

組み込みのnode/operatorブートストラップフローでは、プライマリNodeトークンは`scopes: []`のままで、ハンドオフされたoperatorトークンはブートストラップoperator許可リスト（`operator.approvals`、`operator.read`、`operator.talk.secrets`、`operator.write`）に制限されたままです。ブートストラップscopeチェックは引き続きroleプレフィックス付きです。operatorエントリはoperatorリクエストのみを満たし、非operatorロールは引き続き自身のroleプレフィックス配下のscopeを必要とします。

### Nodeの例

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

副作用を伴うメソッドには**冪等性キー**が必要です（スキーマを参照）。

## Roles + scopes

### Roles

- `operator` = コントロールプレーンクライアント（CLI/UI/自動化）。
- `node` = 機能ホスト（camera/screen/canvas/system.run）。

### Scopes（operator）

一般的なscope:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`includeSecrets: true`を伴う`talk.config`には`operator.talk.secrets`（または`operator.admin`）が必要です。

Plugin登録されたGateway RPCメソッドは独自のoperator scopeを要求できますが、予約済みのコア管理プレフィックス（`config.*`、`exec.approvals.*`、`wizard.*`、`update.*`）は常に`operator.admin`に解決されます。

メソッドscopeは最初のゲートにすぎません。`chat.send`経由で到達する一部のスラッシュコマンドには、その上により厳しいコマンドレベルのチェックが適用されます。たとえば、永続的な`/config set`および`/config unset`の書き込みには`operator.admin`が必要です。

`node.pair.approve`にも、ベースメソッドscopeに加えて追加の承認時scopeチェックがあります。

- コマンドなしのリクエスト: `operator.pairing`
- non-exec Nodeコマンドを含むリクエスト: `operator.pairing` + `operator.write`
- `system.run`、`system.run.prepare`、または`system.which`を含むリクエスト: `operator.pairing` + `operator.admin`

### Caps/commands/permissions（node）

Nodeは接続時に機能クレームを宣言します。

- `caps`: 高レベルな機能カテゴリ。
- `commands`: invoke用のコマンド許可リスト。
- `permissions`: 細粒度のトグル（例: `screen.record`、`camera.capture`）。

Gatewayはこれらを**クレーム**として扱い、サーバー側の許可リストを強制します。

## プレゼンス

- `system-presence`はデバイスIDをキーとするエントリを返します。
- プレゼンスエントリには`deviceId`、`roles`、`scopes`が含まれるため、UIは**operator**と**node**の両方として接続している場合でも、デバイスごとに1行で表示できます。

## 一般的なRPCメソッド群

このページは生成された完全なダンプではありませんが、公開WSサーフェスは上記のハンドシェイク/認証の例よりも広範です。以下は、Gatewayが現在公開している主なメソッド群です。

`hello-ok.features.methods`は、`src/gateway/server-methods-list.ts`とロードされたplugin/channelのメソッドexportから構築された保守的な検出リストです。これを機能検出として扱ってください。`src/gateway/server-methods/*.ts`で実装されているすべての呼び出し可能ヘルパーの生成ダンプではありません。

### システムとID

- `health`は、キャッシュされた、または新たにプローブされたGatewayヘルススナップショットを返します。
- `status`は`/status`スタイルのGatewayサマリーを返します。機微なフィールドは、admin scopeを持つoperatorクライアントにのみ含まれます。
- `gateway.identity.get`は、relayおよびpairingフローで使用されるGatewayデバイスIDを返します。
- `system-presence`は、接続中のoperator/Nodeデバイスの現在のプレゼンススナップショットを返します。
- `system-event`はシステムイベントを追加し、プレゼンスコンテキストを更新/ブロードキャストできます。
- `last-heartbeat`は、最新の永続化されたHeartbeatイベントを返します。
- `set-heartbeats`は、Gateway上でHeartbeat処理を切り替えます。

### モデルと使用状況

- `models.list`は、ランタイムで許可されたモデルカタログを返します。
- `usage.status`は、プロバイダー使用量ウィンドウ/残りクォータのサマリーを返します。
- `usage.cost`は、日付範囲の集計コスト使用量サマリーを返します。
- `doctor.memory.status`は、アクティブなデフォルトエージェントワークスペースのベクトルメモリ/埋め込み準備状況を返します。
- `sessions.usage`は、セッションごとの使用量サマリーを返します。
- `sessions.usage.timeseries`は、1つのセッションの時系列使用量を返します。
- `sessions.usage.logs`は、1つのセッションの使用量ログエントリを返します。

### Channelsとログインヘルパー

- `channels.status`は、組み込み + バンドルされたchannel/pluginのステータスサマリーを返します。
- `channels.logout`は、そのchannelがログアウトをサポートしている場合に、特定のchannel/accountをログアウトします。
- `web.login.start`は、現在のQR対応web channel providerに対するQR/webログインフローを開始します。
- `web.login.wait`は、そのQR/webログインフローの完了を待機し、成功時にchannelを開始します。
- `push.test`は、登録済みのiOS NodeにテストAPNsプッシュを送信します。
- `voicewake.get`は、保存されているウェイクワードトリガーを返します。
- `voicewake.set`は、ウェイクワードトリガーを更新し、変更をブロードキャストします。

### メッセージングとログ

- `send`は、chat runnerの外側でchannel/account/threadターゲット宛て送信を行うための直接の送信RPCです。
- `logs.tail`は、設定されたGatewayファイルログの末尾を、cursor/limitおよびmax-byte制御付きで返します。

### TalkとTTS

- `talk.config`は、有効なTalk設定ペイロードを返します。`includeSecrets`には`operator.talk.secrets`（または`operator.admin`）が必要です。
- `talk.mode`は、WebChat/Control UIクライアント向けの現在のTalk mode状態を設定/ブロードキャストします。
- `talk.speak`は、アクティブなTalk speech providerを通じて音声を合成します。
- `tts.status`は、TTS有効状態、アクティブprovider、フォールバックprovider、およびprovider設定状態を返します。
- `tts.providers`は、表示可能なTTS providerインベントリを返します。
- `tts.enable`と`tts.disable`は、TTS設定状態を切り替えます。
- `tts.setProvider`は、優先TTS providerを更新します。
- `tts.convert`は、単発のtext-to-speech変換を実行します。

### Secrets、config、update、wizard

- `secrets.reload`は、アクティブなSecretRefを再解決し、完全に成功した場合にのみランタイムのシークレット状態を入れ替えます。
- `secrets.resolve`は、特定のcommand/targetセットに対するコマンド対象シークレット割り当てを解決します。
- `config.get`は、現在の設定スナップショットとハッシュを返します。
- `config.set`は、検証済みの設定ペイロードを書き込みます。
- `config.patch`は、部分的な設定更新をマージします。
- `config.apply`は、完全な設定ペイロードを検証して置き換えます。
- `config.schema`は、Control UIとCLIツールで使用されるライブ設定スキーマペイロードを返します。スキーマ、`uiHints`、バージョン、生成メタデータを含み、ランタイムがロードできる場合はplugin + channelのスキーマメタデータも含みます。このスキーマには、同じラベルとヘルプテキストから導出された`title` / `description`メタデータが含まれ、ネストされたオブジェクト、ワイルドカード、配列項目、および一致するフィールドドキュメントが存在する場合の`anyOf` / `oneOf` / `allOf`構成ブランチも含まれます。
- `config.schema.lookup`は、1つの設定パスに対するパススコープのlookupペイロードを返します。正規化されたパス、浅いスキーマノード、一致したヒント + `hintPath`、およびUI/CLIドリルダウン用の直下の子サマリーを含みます。
  - Lookupスキーマノードは、ユーザー向けドキュメントと一般的な検証フィールドを保持します。`title`、`description`、`type`、`enum`、`const`、`format`、`pattern`、数値/文字列/配列/オブジェクトの境界、および`additionalProperties`、`deprecated`、`readOnly`、`writeOnly`などの真偽値フラグです。
  - 子サマリーは、`key`、正規化された`path`、`type`、`required`、`hasChildren`に加えて、一致した`hint` / `hintPath`を公開します。
- `update.run`は、Gateway更新フローを実行し、更新自体が成功した場合にのみ再起動をスケジュールします。
- `wizard.start`、`wizard.next`、`wizard.status`、`wizard.cancel`は、WS RPC経由でオンボーディングウィザードを公開します。

### 既存の主要ファミリー

#### Agentとワークスペースのヘルパー

- `agents.list`は、設定済みagentエントリを返します。
- `agents.create`、`agents.update`、`agents.delete`は、agentレコードとワークスペース配線を管理します。
- `agents.files.list`、`agents.files.get`、`agents.files.set`は、agentに公開されるブートストラップワークスペースファイルを管理します。
- `agent.identity.get`は、agentまたはセッションに対する有効なアシスタントIDを返します。
- `agent.wait`は、実行の完了を待機し、利用可能な場合は終端スナップショットを返します。

#### セッション制御

- `sessions.list`は、現在のセッションインデックスを返します。
- `sessions.subscribe`と`sessions.unsubscribe`は、現在のWSクライアントに対するセッション変更イベント購読を切り替えます。
- `sessions.messages.subscribe`と`sessions.messages.unsubscribe`は、1つのセッションに対するtranscript/messageイベント購読を切り替えます。
- `sessions.preview`は、特定のセッションキーに対する境界付きtranscriptプレビューを返します。
- `sessions.resolve`は、セッションターゲットを解決または正規化します。
- `sessions.create`は、新しいセッションエントリを作成します。
- `sessions.send`は、既存のセッションにメッセージを送信します。
- `sessions.steer`は、アクティブなセッションに対する割り込みとsteerのバリアントです。
- `sessions.abort`は、セッションのアクティブな作業を中止します。
- `sessions.patch`は、セッションメタデータ/オーバーライドを更新します。
- `sessions.reset`、`sessions.delete`、`sessions.compact`は、セッションメンテナンスを実行します。
- `sessions.get`は、完全な保存済みセッション行を返します。
- chat実行では引き続き`chat.history`、`chat.send`、`chat.abort`、`chat.inject`を使用します。
- `chat.history`は、UIクライアント向けに表示正規化されています。インラインdirectiveタグは可視テキストから除去され、プレーンテキストのtool-call XMLペイロード（`<tool_call>...</tool_call>`、`<function_call>...</function_call>`、`<tool_calls>...</tool_calls>`、`<function_calls>...</function_calls>`、および切り詰められたtool-callブロックを含む）と漏出したASCII/全角のモデル制御トークンは除去され、正確な`NO_REPLY` / `no_reply`のような純粋なsilent-token assistant行は省略され、サイズの大きすぎる行はプレースホルダーに置き換えられることがあります。

#### デバイスペアリングとデバイストークン

- `device.pair.list`は、保留中および承認済みのペア済みデバイスを返します。
- `device.pair.approve`、`device.pair.reject`、`device.pair.remove`は、デバイスペアリングレコードを管理します。
- `device.token.rotate`は、承認済みのroleおよびscope境界内でペア済みデバイストークンをローテーションします。
- `device.token.revoke`は、ペア済みデバイストークンを失効させます。

#### Nodeのペアリング、invoke、保留中の作業

- `node.pair.request`、`node.pair.list`、`node.pair.approve`、`node.pair.reject`、`node.pair.verify`は、Nodeペアリングとブートストラップ検証を扱います。
- `node.list`と`node.describe`は、既知の/接続中のNode状態を返します。
- `node.rename`は、ペア済みNodeラベルを更新します。
- `node.invoke`は、接続中のNodeにコマンドを転送します。
- `node.invoke.result`は、invokeリクエストの結果を返します。
- `node.event`は、Node起点のイベントをGatewayに戻します。
- `node.canvas.capability.refresh`は、スコープ付きcanvas-capabilityトークンを更新します。
- `node.pending.pull`と`node.pending.ack`は、接続中NodeキューAPIです。
- `node.pending.enqueue`と`node.pending.drain`は、オフライン/切断中のNode向けの永続的な保留作業を管理します。

#### 承認ファミリー

- `exec.approval.request`、`exec.approval.get`、`exec.approval.list`、`exec.approval.resolve`は、単発のexec承認リクエストと、保留中承認のlookup/replayを扱います。
- `exec.approval.waitDecision`は、1つの保留中exec承認を待機し、最終決定を返します（タイムアウト時は`null`）。
- `exec.approvals.get`と`exec.approvals.set`は、Gateway exec承認ポリシースナップショットを管理します。
- `exec.approvals.node.get`と`exec.approvals.node.set`は、Node relayコマンド経由でNodeローカルexec承認ポリシーを管理します。
- `plugin.approval.request`、`plugin.approval.list`、`plugin.approval.waitDecision`、`plugin.approval.resolve`は、Plugin定義の承認フローを扱います。

#### その他の主要ファミリー

- automation:
  - `wake`は、即時または次回Heartbeat時のwakeテキスト注入をスケジュールします
  - `cron.list`、`cron.status`、`cron.add`、`cron.update`、`cron.remove`、`cron.run`、`cron.runs`
- Skills/tools: `commands.list`、`skills.*`、`tools.catalog`、`tools.effective`

### 一般的なイベントファミリー

- `chat`: `chat.inject`やその他のtranscript専用chatイベントなどのUI chat更新。
- `session.message`と`session.tool`: 購読中セッション向けのtranscript/イベントストリーム更新。
- `sessions.changed`: セッションインデックスまたはメタデータが変更されました。
- `presence`: システムプレゼンススナップショット更新。
- `tick`: 定期的なkeepalive / livenessイベント。
- `health`: Gatewayヘルススナップショット更新。
- `heartbeat`: Heartbeatイベントストリーム更新。
- `cron`: Cron実行/ジョブ変更イベント。
- `shutdown`: Gatewayシャットダウン通知。
- `node.pair.requested` / `node.pair.resolved`: Nodeペアリングのライフサイクル。
- `node.invoke.request`: Node invokeリクエストのブロードキャスト。
- `device.pair.requested` / `device.pair.resolved`: ペア済みデバイスのライフサイクル。
- `voicewake.changed`: ウェイクワードトリガー設定が変更されました。
- `exec.approval.requested` / `exec.approval.resolved`: exec承認のライフサイクル。
- `plugin.approval.requested` / `plugin.approval.resolved`: Plugin承認のライフサイクル。

### Nodeヘルパーメソッド

- Nodeは、自動許可チェック用に現在のskill実行ファイル一覧を取得するために`skills.bins`を呼び出せます。

### Operatorヘルパーメソッド

- Operatorは、agentのランタイムコマンドインベントリを取得するために`commands.list`（`operator.read`）を呼び出せます。
  - `agentId`は任意です。省略するとデフォルトagentワークスペースを読み取ります。
  - `scope`は、主`name`がどのサーフェスを対象にするかを制御します。
    - `text`は、先頭の`/`を除いた主textコマンドトークンを返します
    - `native`およびデフォルトの`both`パスは、利用可能な場合にprovider対応native名を返します
  - `textAliases`は、`/model`や`/m`のような正確なスラッシュ別名を保持します。
  - `nativeName`は、存在する場合にprovider対応nativeコマンド名を保持します。
  - `provider`は任意で、native名付けとnative Pluginコマンド可用性にのみ影響します。
  - `includeArgs=false`は、レスポンスからシリアライズ済み引数メタデータを省略します。
- Operatorは、agentのランタイムtoolカタログを取得するために`tools.catalog`（`operator.read`）を呼び出せます。レスポンスには、グループ化されたtoolsとprovenanceメタデータが含まれます。
  - `source`: `core`または`plugin`
  - `pluginId`: `source="plugin"`のときのPlugin所有者
  - `optional`: Plugin toolが任意かどうか
- Operatorは、セッションのランタイムで有効なtoolインベントリを取得するために`tools.effective`（`operator.read`）を呼び出せます。
  - `sessionKey`は必須です。
  - Gatewayは、呼び出し元から供給されたauthやdeliveryコンテキストを受け入れる代わりに、セッションから信頼できるランタイムコンテキストをサーバー側で導出します。
  - レスポンスはセッションスコープであり、core、Plugin、channel toolsを含めて、アクティブな会話が現在使用できるものを反映します。
- Operatorは、agentの可視なskillインベントリを取得するために`skills.status`（`operator.read`）を呼び出せます。
  - `agentId`は任意です。省略するとデフォルトagentワークスペースを読み取ります。
  - レスポンスには、適格性、不足している要件、configチェック、生のシークレット値を公開しないサニタイズ済みinstallオプションが含まれます。
- Operatorは、ClawHub検出メタデータのために`skills.search`と`skills.detail`（`operator.read`）を呼び出せます。
- Operatorは、`skills.install`（`operator.admin`）を2つのモードで呼び出せます。
  - ClawHubモード: `{ source: "clawhub", slug, version?, force? }`は、デフォルトagentワークスペースの`skills/`ディレクトリにskillフォルダーをインストールします。
  - Gatewayインストーラーモード: `{ name, installId, dangerouslyForceUnsafeInstall?, timeoutMs? }`は、Gatewayホスト上で宣言された`metadata.openclaw.install`アクションを実行します。
- Operatorは、`skills.update`（`operator.admin`）を2つのモードで呼び出せます。
  - ClawHubモードは、1つの追跡対象slug、またはデフォルトagentワークスペース内のすべての追跡対象ClawHubインストールを更新します。
  - Configモードは、`enabled`、`apiKey`、`env`などの`skills.entries.<skillKey>`値をパッチします。

## Exec承認

- execリクエストに承認が必要な場合、Gatewayは`exec.approval.requested`をブロードキャストします。
- Operatorクライアントは、`exec.approval.resolve`を呼び出して解決します（`operator.approvals` scopeが必要です）。
- `host=node`の場合、`exec.approval.request`には`systemRunPlan`（正規化された`argv`/`cwd`/`rawCommand`/セッションメタデータ）が含まれている必要があります。`systemRunPlan`がないリクエストは拒否されます。
- 承認後、転送された`node.invoke system.run`呼び出しは、その正規の`systemRunPlan`を権威あるcommand/cwd/sessionコンテキストとして再利用します。
- 呼び出し元がprepareと最終承認済み`system.run`転送の間で`command`、`rawCommand`、`cwd`、`agentId`、`sessionKey`を変更した場合、Gatewayは変更されたペイロードを信用せず、その実行を拒否します。

## Agent配信フォールバック

- `agent`リクエストには、送信先配信を要求するための`deliver=true`を含めることができます。
- `bestEffortDeliver=false`は厳密な動作を維持します。未解決または内部専用の配信先ターゲットは`INVALID_REQUEST`を返します。
- `bestEffortDeliver=true`は、外部配信可能ルートを解決できない場合（たとえば内部/webchatセッションや曖昧なマルチchannel設定）に、セッション専用実行へのフォールバックを許可します。

## バージョニング

- `PROTOCOL_VERSION`は`src/gateway/protocol/schema/protocol-schemas.ts`にあります。
- クライアントは`minProtocol` + `maxProtocol`を送信し、サーバーは不一致を拒否します。
- スキーマ + モデルはTypeBox定義から生成されます。
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

### クライアント定数

`src/gateway/client.ts`のリファレンスクライアントは、これらのデフォルト値を使用します。値はプロトコルv3全体で安定しており、サードパーティクライアントに期待されるベースラインです。

| 定数 | デフォルト | ソース |
| ----------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| `PROTOCOL_VERSION` | `3` | `src/gateway/protocol/schema/protocol-schemas.ts` |
| リクエストタイムアウト（RPCごと） | `30_000` ms | `src/gateway/client.ts` (`requestTimeoutMs`) |
| preauth / connect-challengeタイムアウト | `10_000` ms | `src/gateway/handshake-timeouts.ts`（クランプ `250`–`10_000`） |
| 初期再接続バックオフ | `1_000` ms | `src/gateway/client.ts` (`backoffMs`) |
| 最大再接続バックオフ | `30_000` ms | `src/gateway/client.ts` (`scheduleReconnect`) |
| device-token close後の高速リトライクランプ | `250` ms | `src/gateway/client.ts` |
| `terminate()`前の強制停止猶予 | `250` ms | `FORCE_STOP_TERMINATE_GRACE_MS` |
| `stopAndWait()`デフォルトタイムアウト | `1_000` ms | `STOP_AND_WAIT_TIMEOUT_MS` |
| デフォルトtick間隔（`hello-ok`前） | `30_000` ms | `src/gateway/client.ts` |
| tickタイムアウトclose | 無通信が`tickIntervalMs * 2`を超えるとコード`4000` | `src/gateway/client.ts` |
| `MAX_PAYLOAD_BYTES` | `25 * 1024 * 1024`（25 MB） | `src/gateway/server-constants.ts` |

サーバーは、有効な`policy.tickIntervalMs`、`policy.maxPayload`、`policy.maxBufferedBytes`を`hello-ok`で通知します。クライアントは、ハンドシェイク前のデフォルト値ではなく、それらの値に従うべきです。

## 認証

- 共有シークレットGateway認証は、設定された認証モードに応じて`connect.params.auth.token`または`connect.params.auth.password`を使用します。
- Tailscale Serve（`gateway.auth.allowTailscale: true`）や非loopbackの`gateway.auth.mode: "trusted-proxy"`のようなID保持モードでは、`connect.params.auth.*`ではなくリクエストヘッダーからconnect認証チェックを満たします。
- プライベートingressの`gateway.auth.mode: "none"`は共有シークレットconnect認証を完全にスキップします。このモードを公開/信頼できないingressで公開しないでください。
- ペアリング後、Gatewayは接続のrole + scopesにスコープされた**device token**を発行します。これは`hello-ok.auth.deviceToken`で返され、クライアントは将来の接続のためにそれを永続化する必要があります。
- クライアントは、成功したconnectの後に常にプライマリ`hello-ok.auth.deviceToken`を永続化する必要があります。
- その**保存済み**device tokenで再接続する場合、そのトークンに対して保存済みの承認scopeセットも再利用する必要があります。これにより、すでに付与されたread/probe/statusアクセスが保持され、再接続がより狭い暗黙のadmin専用scopeへ静かに縮小されることを防ぎます。
- クライアント側のconnect認証組み立て（`src/gateway/client.ts`の`selectConnectAuth`）:
  - `auth.password`は直交しており、設定されている場合は常に転送されます。
  - `auth.token`は優先順位順に設定されます。最初に明示的な共有トークン、次に明示的な`deviceToken`、最後に保存済みのデバイス単位トークン（`deviceId` + `role`でキー付け）。
  - `auth.bootstrapToken`は、上記のいずれでも`auth.token`が解決されなかった場合にのみ送信されます。共有トークンまたは解決済みのdevice tokenがあれば送信されません。
  - 保存済みdevice tokenの自動昇格は、ワンショットの`AUTH_TOKEN_MISMATCH`リトライ時でも**信頼されたエンドポイントのみ**で有効です — loopback、または固定された`tlsFingerprint`を持つ`wss://`です。ピン留めなしの公開`wss://`は該当しません。
- 追加の`hello-ok.auth.deviceTokens`エントリはブートストラップ引き継ぎトークンです。`wss://`またはloopback/local pairingのような信頼されたトランスポート上でブートストラップ認証を使った接続の場合にのみ永続化してください。
- クライアントが明示的な`deviceToken`または明示的な`scopes`を指定した場合、その呼び出し元要求のscopeセットが引き続き権威を持ちます。キャッシュ済みscopeが再利用されるのは、クライアントが保存済みのデバイス単位トークンを再利用している場合だけです。
- Device tokenは`device.token.rotate`と`device.token.revoke`でローテーション/失効できます（`operator.pairing` scopeが必要です）。
- トークンの発行/ローテーションは、そのデバイスのペアリングエントリに記録された承認済みroleセットの範囲内に制限されます。トークンのローテーションによって、ペアリング承認が一度も許可していないroleへそのデバイスを拡張することはできません。
- ペア済みデバイストークンセッションでは、呼び出し元が`operator.admin`も持っていない限り、デバイス管理は自身のスコープに限定されます。非admin呼び出し元は、自分**自身**のデバイスエントリのみをremove/revoke/rotateできます。
- `device.token.rotate`は、要求されたoperator scopeセットが呼び出し元の現在のセッションscopeに対して適切かどうかも確認します。非admin呼び出し元は、自分が現在保持しているより広いoperator scopeセットへトークンをローテーションできません。
- 認証失敗には、`error.details.code`と回復ヒントが含まれます:
  - `error.details.canRetryWithDeviceToken`（boolean）
  - `error.details.recommendedNextStep`（`retry_with_device_token`、`update_auth_configuration`、`update_auth_credentials`、`wait_then_retry`、`review_auth_configuration`）
- `AUTH_TOKEN_MISMATCH`に対するクライアント動作:
  - 信頼されたクライアントは、キャッシュ済みのデバイス単位トークンで1回だけ制限付きリトライを試行できます。
  - そのリトライが失敗した場合、クライアントは自動再接続ループを停止し、オペレーターの対応ガイダンスを表示する必要があります。

## デバイスID + ペアリング

- Nodeは、キーペアフィンガープリントから導出された安定したデバイスID（`device.id`）を含める必要があります。
- Gatewayは、デバイス + roleごとにトークンを発行します。
- ローカル自動承認が有効でない限り、新しいデバイスIDにはペアリング承認が必要です。
- ペアリング自動承認は、直接のlocal loopback接続を中心にしています。
- OpenClawには、信頼された共有シークレットヘルパーフロー向けの限定的なバックエンド/コンテナローカル自己接続パスもあります。
- 同一ホストのtailnetまたはLAN接続は、依然としてpairing上はリモートとして扱われ、承認が必要です。
- すべてのWSクライアントは、`connect`中に`device` IDを含める必要があります（operator + node）。
  Control UIがこれを省略できるのは、次のモードのみです:
  - localhost専用の安全でないHTTP互換性のための`gateway.controlUi.allowInsecureAuth=true`。
  - 成功した`gateway.auth.mode: "trusted-proxy"` operator Control UI認証。
  - `gateway.controlUi.dangerouslyDisableDeviceAuth=true`（非常時用、深刻なセキュリティ低下）。
- すべての接続は、サーバー提供の`connect.challenge` nonceに署名する必要があります。

### デバイス認証移行診断

従来のchallenge前署名動作をまだ使用しているレガシークライアント向けに、`connect`は現在、安定した`error.details.reason`の下で`error.details.code`に`DEVICE_AUTH_*`詳細コードを返します。

一般的な移行失敗:

| メッセージ | details.code | details.reason | 意味 |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required` | `DEVICE_AUTH_NONCE_REQUIRED` | `device-nonce-missing` | クライアントが`device.nonce`を省略した（または空で送信した）。 |
| `device nonce mismatch` | `DEVICE_AUTH_NONCE_MISMATCH` | `device-nonce-mismatch` | クライアントが古い/誤ったnonceで署名した。 |
| `device signature invalid` | `DEVICE_AUTH_SIGNATURE_INVALID` | `device-signature` | 署名ペイロードがv2ペイロードと一致しない。 |
| `device signature expired` | `DEVICE_AUTH_SIGNATURE_EXPIRED` | `device-signature-stale` | 署名済みタイムスタンプが許容されるスキュー範囲外。 |
| `device identity mismatch` | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch` | `device.id`が公開鍵フィンガープリントと一致しない。 |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key` | 公開鍵の形式/正規化に失敗した。 |

移行目標:

- 常に`connect.challenge`を待機する。
- サーバーnonceを含むv2ペイロードに署名する。
- 同じnonceを`connect.params.device.nonce`で送信する。
- 推奨される署名ペイロードは`v3`で、device/client/role/scopes/token/nonceフィールドに加えて`platform`と`deviceFamily`を束縛します。
- レガシー`v2`署名も互換性のため引き続き受け入れられますが、ペア済みデバイスメタデータのピン留めは再接続時のコマンドポリシーを引き続き制御します。

## TLS + ピン留め

- WS接続ではTLSがサポートされます。
- クライアントは任意でGateway証明書フィンガープリントをピン留めできます（`gateway.tls` configおよび`gateway.remote.tlsFingerprint`またはCLI `--tls-fingerprint`を参照）。

## スコープ

このプロトコルは**完全なGateway API**（status、channels、models、chat、agent、sessions、nodes、approvalsなど）を公開します。正確なサーフェスは`src/gateway/protocol/schema.ts`のTypeBoxスキーマで定義されています。
