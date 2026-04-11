---
read_when:
    - エージェントループまたはライフサイクルイベントの正確なウォークスルーが必要です
summary: エージェントループのライフサイクル、ストリーム、および待機セマンティクス
title: エージェントループ
x-i18n:
    generated_at: "2026-04-11T02:44:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: b6831a5b11e9100e49f650feca51ab44a2bef242ce1b5db2766d0b3b5c5ba729
    source_path: concepts/agent-loop.md
    workflow: 15
---

# エージェントループ（OpenClaw）

エージェントループは、エージェントの完全な「実際の」実行です: 受信 → コンテキストの組み立て → モデル推論 →
ツール実行 → 返信のストリーミング → 永続化。これは、セッション状態の整合性を保ちながら、
メッセージをアクションと最終返信に変換する正式な経路です。

OpenClawでは、ループはセッションごとに単一の直列化された実行であり、モデルが思考し、ツールを呼び出し、出力をストリーミングする間、
ライフサイクルイベントとストリームイベントを発行します。このドキュメントでは、その本物のループがエンドツーエンドでどのように構成されているかを説明します。

## エントリーポイント

- Gateway RPC: `agent` および `agent.wait`。
- CLI: `agent` コマンド。

## 仕組み（概要）

1. `agent` RPCはパラメータを検証し、セッション（sessionKey/sessionId）を解決し、セッションメタデータを永続化し、すぐに `{ runId, acceptedAt }` を返します。
2. `agentCommand` がエージェントを実行します:
   - モデルと thinking/verbose のデフォルト値を解決
   - Skills スナップショットを読み込み
   - `runEmbeddedPiAgent`（pi-agent-coreランタイム）を呼び出し
   - 埋め込みループが発行しない場合は **lifecycle end/error** を発行
3. `runEmbeddedPiAgent`:
   - セッション単位 + グローバルキューを通じて実行を直列化
   - モデル + auth profile を解決し、piセッションを構築
   - piイベントを購読し、assistant/tool の差分をストリーミング
   - タイムアウトを強制し、超過時は実行を中断
   - ペイロード + 使用量メタデータを返す
4. `subscribeEmbeddedPiSession` が pi-agent-core イベントを OpenClaw の `agent` ストリームに橋渡しします:
   - ツールイベント => `stream: "tool"`
   - assistant 差分 => `stream: "assistant"`
   - ライフサイクルイベント => `stream: "lifecycle"` (`phase: "start" | "end" | "error"`)
5. `agent.wait` は `waitForAgentRun` を使用します:
   - `runId` の **lifecycle end/error** を待機
   - `{ status: ok|error|timeout, startedAt, endedAt, error? }` を返す

## キューイング + 並行性

- 実行はセッションキーごと（セッションレーン）に直列化され、必要に応じてグローバルレーンも通ります。
- これによりツール/セッションの競合を防ぎ、セッション履歴の整合性を保ちます。
- メッセージングチャネルは、このレーンシステムに流し込まれるキューモード（collect/steer/followup）を選択できます。
  詳しくは [Command Queue](/ja-JP/concepts/queue) を参照してください。

## セッション + ワークスペースの準備

- ワークスペースは解決されて作成されます。サンドボックス実行では、サンドボックス用ワークスペースルートにリダイレクトされることがあります。
- Skills が読み込まれ（またはスナップショットから再利用され）、env とプロンプトに注入されます。
- bootstrap/context ファイルが解決され、システムプロンプトレポートに注入されます。
- セッション書き込みロックが取得され、`SessionManager` がストリーミング前に開かれて準備されます。

## プロンプトの組み立て + システムプロンプト

- システムプロンプトは、OpenClaw のベースプロンプト、Skills プロンプト、bootstrap コンテキスト、および実行ごとのオーバーライドから構築されます。
- モデル固有の制限と compaction 用の予約トークンが適用されます。
- モデルが何を見るかについては、[System prompt](/ja-JP/concepts/system-prompt) を参照してください。

## フックポイント（介入できる場所）

OpenClaw には2つのフックシステムがあります:

- **内部フック**（Gateway フック）: コマンドおよびライフサイクルイベント用のイベント駆動スクリプト。
- **プラグインフック**: エージェント/ツールのライフサイクルおよび Gateway パイプライン内の拡張ポイント。

### 内部フック（Gateway フック）

- **`agent:bootstrap`**: システムプロンプトが確定する前に bootstrap ファイルを構築している間に実行されます。
  これを使って bootstrap コンテキストファイルを追加/削除します。
- コマンドフック: `/new`、`/reset`、`/stop`、およびその他のコマンドイベント（Hooks ドキュメントを参照）。

セットアップと例については [Hooks](/ja-JP/automation/hooks) を参照してください。

### プラグインフック（エージェント + Gateway ライフサイクル）

これらはエージェントループ内または Gateway パイプライン内で実行されます:

- **`before_model_resolve`**: モデル解決前に、provider/model を決定論的に上書きするために、セッション前（`messages` なし）で実行されます。
- **`before_prompt_build`**: セッション読み込み後（`messages` あり）に実行され、プロンプト送信前に `prependContext`、`systemPrompt`、`prependSystemContext`、または `appendSystemContext` を注入します。ターンごとの動的テキストには `prependContext` を使用し、システムプロンプト空間に置くべき安定したガイダンスには system-context フィールドを使用します。
- **`before_agent_start`**: レガシー互換性フックで、どちらのフェーズでも実行される可能性があります。できるだけ上記の明示的なフックを使用してください。
- **`before_agent_reply`**: インラインアクションの後、LLM 呼び出しの前に実行され、プラグインがそのターンを引き受けて synthetic reply を返したり、そのターンを完全に無言にしたりできます。
- **`agent_end`**: 完了後の最終メッセージ一覧と実行メタデータを検査します。
- **`before_compaction` / `after_compaction`**: compaction サイクルを監視または注釈付けします。
- **`before_tool_call` / `after_tool_call`**: ツールのパラメータ/結果に介入します。
- **`before_install`**: 組み込みスキャン結果を検査し、必要に応じて skill または plugin のインストールをブロックします。
- **`tool_result_persist`**: ツール結果がセッショントランスクリプトに書き込まれる前に、同期的に変換します。
- **`message_received` / `message_sending` / `message_sent`**: 受信および送信メッセージフック。
- **`session_start` / `session_end`**: セッションライフサイクルの境界。
- **`gateway_start` / `gateway_stop`**: Gateway ライフサイクルイベント。

送信/ツールガードのフック判定ルール:

- `before_tool_call`: `{ block: true }` は終端であり、より低い優先度のハンドラーを停止します。
- `before_tool_call`: `{ block: false }` は no-op であり、以前の block を解除しません。
- `before_install`: `{ block: true }` は終端であり、より低い優先度のハンドラーを停止します。
- `before_install`: `{ block: false }` は no-op であり、以前の block を解除しません。
- `message_sending`: `{ cancel: true }` は終端であり、より低い優先度のハンドラーを停止します。
- `message_sending`: `{ cancel: false }` は no-op であり、以前の cancel を解除しません。

フック API と登録の詳細は [Plugin hooks](/ja-JP/plugins/architecture#provider-runtime-hooks) を参照してください。

## ストリーミング + 部分返信

- assistant 差分は pi-agent-core からストリーミングされ、`assistant` イベントとして発行されます。
- ブロックストリーミングは、`text_end` または `message_end` で部分返信を発行できます。
- reasoning ストリーミングは、別ストリームとして、またはブロック返信として発行できます。
- チャンク化とブロック返信の挙動については [Streaming](/ja-JP/concepts/streaming) を参照してください。

## ツール実行 + メッセージングツール

- ツールの start/update/end イベントは `tool` ストリームで発行されます。
- ツール結果は、ログ出力/発行の前に、サイズおよび画像ペイロードに関してサニタイズされます。
- メッセージングツールの送信は、assistant による重複確認を抑制するために追跡されます。

## 返信の整形 + 抑制

- 最終ペイロードは次から組み立てられます:
  - assistant テキスト（および任意の reasoning）
  - インラインツール要約（verbose かつ許可されている場合）
  - モデルエラー時の assistant エラーテキスト
- 正確な silent token `NO_REPLY` / `no_reply` は送信ペイロードから除外されます。
- メッセージングツールの重複は最終ペイロード一覧から削除されます。
- 描画可能なペイロードが何も残っておらず、かつツールでエラーが発生した場合は、
  フォールバックのツールエラー返信が発行されます
  （ただし、メッセージングツールがすでにユーザーに見える返信を送信している場合を除きます）。

## compaction + リトライ

- 自動 compaction は `compaction` ストリームイベントを発行し、リトライを引き起こすことがあります。
- リトライ時には、重複出力を避けるため、インメモリバッファとツール要約がリセットされます。
- compaction パイプラインについては [Compaction](/ja-JP/concepts/compaction) を参照してください。

## イベントストリーム（現時点）

- `lifecycle`: `subscribeEmbeddedPiSession` によって発行されます（およびフォールバックとして `agentCommand` からも発行されます）
- `assistant`: pi-agent-core からのストリーミング差分
- `tool`: pi-agent-core からのストリーミングツールイベント

## チャットチャネルの処理

- assistant 差分はチャット `delta` メッセージにバッファされます。
- チャット `final` は **lifecycle end/error** で発行されます。

## タイムアウト

- `agent.wait` のデフォルト: 30秒（待機のみ）。`timeoutMs` パラメータで上書きできます。
- エージェント実行時: `agents.defaults.timeoutSeconds` のデフォルトは 172800秒（48時間）で、`runEmbeddedPiAgent` の abort タイマーで強制されます。
- LLM アイドルタイムアウト: `agents.defaults.llm.idleTimeoutSeconds` は、アイドルウィンドウ内にレスポンスチャンクが到着しない場合にモデルリクエストを中断します。遅いローカルモデルや reasoning/ツール呼び出し provider には明示的に設定してください。無効化するには 0 に設定します。設定されていない場合、OpenClaw は `agents.defaults.timeoutSeconds` が設定されていればそれを使用し、そうでなければ 120秒を使用します。明示的な LLM またはエージェントタイムアウトのない cron トリガー実行では、アイドルウォッチドッグは無効化され、cron の外側のタイムアウトに依存します。

## 途中で早期終了する可能性がある場所

- エージェントタイムアウト（abort）
- AbortSignal（キャンセル）
- Gateway 切断または RPC タイムアウト
- `agent.wait` タイムアウト（待機のみで、エージェント自体は停止しません）

## 関連

- [Tools](/ja-JP/tools) — 利用可能なエージェントツール
- [Hooks](/ja-JP/automation/hooks) — エージェントライフサイクルイベントによってトリガーされるイベント駆動スクリプト
- [Compaction](/ja-JP/concepts/compaction) — 長い会話がどのように要約されるか
- [Exec Approvals](/ja-JP/tools/exec-approvals) — シェルコマンドの承認ゲート
- [Thinking](/ja-JP/tools/thinking) — thinking/reasoning レベルの設定
