---
read_when:
    - バックグラウンドジョブまたはウェイクアップのスケジュール設定
    - 外部トリガー（Webhook、Gmail）をOpenClawに接続する】【。assistant to=functions.read კომენტary  天天送json  天天中彩票是  {"path":"/home/runner/work/docs/docs/source/.agents/skills/security-triage/SKILL.md"}
    - スケジュールされたタスクでheartbeatとcronのどちらを使うかを判断する
summary: Gatewayスケジューラのスケジュール済みジョブ、Webhook、Gmail PubSubトリガー
title: スケジュールされたタスク
x-i18n:
    generated_at: "2026-04-11T02:44:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 04d94baa152de17d78515f7d545f099fe4810363ab67e06b465e489737f54665
    source_path: automation/cron-jobs.md
    workflow: 15
---

# スケジュールされたタスク（Cron）

CronはGatewayの組み込みスケジューラです。ジョブを永続化し、適切なタイミングでエージェントを起動し、出力をチャットチャネルまたはWebhookエンドポイントに返すことができます。

## クイックスタート

```bash
# 1回限りのリマインダーを追加
openclaw cron add \
  --name "Reminder" \
  --at "2026-02-01T16:00:00Z" \
  --session main \
  --system-event "Reminder: check the cron docs draft" \
  --wake now \
  --delete-after-run

# ジョブを確認
openclaw cron list

# 実行履歴を表示
openclaw cron runs --id <job-id>
```

## cronの仕組み

- Cronは**Gatewayプロセス内**で実行されます（モデル内ではありません）。
- ジョブは`~/.openclaw/cron/jobs.json`に永続化されるため、再起動してもスケジュールは失われません。
- すべてのcron実行で[バックグラウンドタスク](/ja-JP/automation/tasks)レコードが作成されます。
- 1回限りのジョブ（`--at`）は、デフォルトで成功後に自動削除されます。
- 分離されたcron実行では、実行完了時にその`cron:<jobId>`セッション用の追跡対象ブラウザタブやプロセスをベストエフォートで閉じるため、切り離されたブラウザ自動化によって孤立プロセスが残りません。
- 分離されたcron実行では、古い確認応答返信も防止されます。最初の結果が単なる中間ステータス更新（`on it`、`pulling everything together`、および同様のヒント）であり、最終回答を担当する子孫subagent実行がまだ存在しない場合、OpenClawは配信前に実際の結果を得るためにもう一度再プロンプトします。

<a id="maintenance"></a>

cronのタスク再調整はランタイム側で管理されます。古い子セッション行がまだ存在していても、cronランタイムがそのジョブを実行中として追跡している間は、アクティブなcronタスクは存続します。
ランタイムがそのジョブの管理をやめ、5分間の猶予ウィンドウが経過すると、メンテナンスによってそのタスクは`lost`とマークされることがあります。

## スケジュールの種類

| 種類    | CLIフラグ | 説明                                                    |
| ------- | --------- | ------------------------------------------------------- |
| `at`    | `--at`    | 1回限りのタイムスタンプ（ISO 8601または`20m`のような相対指定） |
| `every` | `--every` | 固定間隔                                                |
| `cron`  | `--cron`  | `--tz`を省略可能な5フィールドまたは6フィールドのcron式  |

タイムゾーンがないタイムスタンプはUTCとして扱われます。ローカルの壁時計時刻でスケジュールするには`--tz America/New_York`を追加してください。

毎時ちょうどに実行される繰り返し式は、負荷の急増を減らすために最大5分まで自動的にずらされます。正確な時刻に強制するには`--exact`を使用し、明示的なウィンドウを指定するには`--stagger 30s`を使用してください。

## 実行スタイル

| スタイル        | `--session`値       | 実行場所                 | 最適な用途                      |
| --------------- | ------------------- | ------------------------ | ------------------------------- |
| メインセッション | `main`              | 次回heartbeatターン      | リマインダー、システムイベント |
| 分離            | `isolated`          | 専用の`cron:<jobId>`     | レポート、バックグラウンド作業 |
| 現在のセッション | `current`           | 作成時にバインド         | コンテキスト認識の定期作業     |
| カスタムセッション | `session:custom-id` | 永続的な名前付きセッション | 履歴を活用するワークフロー      |

**メインセッション**ジョブはシステムイベントをキューに入れ、必要に応じてheartbeatを起動します（`--wake now`または`--wake next-heartbeat`）。**分離**ジョブは、新しいセッションで専用のエージェントターンを実行します。**カスタムセッション**（`session:xxx`）は実行間でコンテキストを保持するため、以前の要約を基に積み上げていく日次スタンドアップのようなワークフローを可能にします。

分離ジョブでは、ランタイムの後処理にそのcronセッション向けのベストエフォートなブラウザクリーンアップが含まれるようになりました。クリーンアップの失敗は無視されるため、実際のcron結果が優先されます。

分離されたcron実行がsubagentをオーケストレーションする場合、配信でも古い親の中間テキストより、最終的な子孫の出力が優先されます。子孫がまだ実行中であれば、OpenClawはその部分的な親更新を通知せず抑制します。

### 分離ジョブのペイロードオプション

- `--message`: プロンプトテキスト（分離では必須）
- `--model` / `--thinking`: モデルおよびthinkingレベルのオーバーライド
- `--light-context`: ワークスペースのブートストラップファイル挿入をスキップ
- `--tools exec,read`: ジョブが使用できるツールを制限

`--model`はそのジョブで選択された許可済みモデルを使用します。要求されたモデルが許可されていない場合、cronは警告をログに出し、代わりにそのジョブのエージェント/デフォルトのモデル選択にフォールバックします。設定されたフォールバックチェーンは引き続き適用されますが、明示的なジョブ単位フォールバックリストのない単純なモデルオーバーライドでは、隠れた追加リトライ先としてエージェントのprimaryモデルは付加されなくなりました。

分離ジョブのモデル選択の優先順位は次のとおりです。

1. Gmailフックのモデルオーバーライド（実行がGmailから来ており、そのオーバーライドが許可されている場合）
2. ジョブ単位ペイロードの`model`
3. 保存済みcronセッションのモデルオーバーライド
4. エージェント/デフォルトのモデル選択

Fast modeも解決済みのlive選択に従います。選択されたモデル設定に`params.fastMode`がある場合、分離cronはデフォルトでそれを使用します。保存済みセッションの`fastMode`オーバーライドは、どちらの方向でも設定より優先されます。

分離実行がliveのモデル切り替えハンドオフに達した場合、cronは切り替え後のprovider/modelで再試行し、そのlive選択を再試行前に永続化します。切り替えに新しい認証プロファイルも含まれる場合、cronはその認証プロファイルのオーバーライドも永続化します。再試行回数には上限があります。初回試行に加えて2回の切り替え再試行後は、無限ループを避けるためにcronは中止します。

## 配信と出力

| モード    | 動作                                                     |
| --------- | -------------------------------------------------------- |
| `announce` | 要約を対象チャネルに配信（分離のデフォルト）            |
| `webhook`  | 完了イベントのペイロードをURLにPOST                     |
| `none`     | 内部のみ、配信なし                                      |

チャネル配信には`--announce --channel telegram --to "-1001234567890"`を使用します。Telegramフォーラムトピックには`-1001234567890:topic:123`を使用します。Slack/Discord/Mattermostの対象には、明示的なプレフィックス（`channel:<id>`、`user:<id>`）を使用する必要があります。

cronが管理する分離ジョブでは、ランナーが最終配信経路を管理します。エージェントにはプレーンテキストの要約を返すようプロンプトが与えられ、その要約が`announce`、`webhook`を通じて送信されるか、`none`では内部保持されます。`--no-deliver`は配信をエージェントに戻すのではなく、実行を内部のみに保ちます。

元のタスクが特定の外部受信者にメッセージを送ることを明示的に指示している場合、エージェントはそれを直接送信しようとせず、誰に/どこにそのメッセージを送るべきかを出力内に記載する必要があります。

失敗通知は別の宛先経路に従います。

- `cron.failureDestination`は失敗通知のグローバルデフォルトを設定します。
- `job.delivery.failureDestination`はジョブ単位でそれを上書きします。
- どちらも設定されておらず、かつジョブがすでに`announce`で配信している場合、失敗通知はそのprimaryのannounce対象にフォールバックするようになりました。
- `delivery.failureDestination`がサポートされるのは、primary配信モードが`webhook`である場合を除き、`sessionTarget="isolated"`ジョブのみです。

## CLIの例

1回限りのリマインダー（メインセッション）:

```bash
openclaw cron add \
  --name "Calendar check" \
  --at "20m" \
  --session main \
  --system-event "Next heartbeat: check calendar." \
  --wake now
```

配信付きの定期分離ジョブ:

```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize overnight updates." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

モデルおよびthinkingオーバーライド付きの分離ジョブ:

```bash
openclaw cron add \
  --name "Deep analysis" \
  --cron "0 6 * * 1" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Weekly deep analysis of project progress." \
  --model "opus" \
  --thinking high \
  --announce
```

## Webhook

Gatewayは外部トリガー用にHTTP Webhookエンドポイントを公開できます。設定で有効にします。

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
  },
}
```

### 認証

すべてのリクエストには、ヘッダー経由でフックトークンを含める必要があります。

- `Authorization: Bearer <token>`（推奨）
- `x-openclaw-token: <token>`

クエリ文字列のトークンは拒否されます。

### POST /hooks/wake

メインセッション用のシステムイベントをキューに入れます。

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"text":"New email received","mode":"now"}'
```

- `text`（必須）: イベントの説明
- `mode`（省略可能）: `now`（デフォルト）または`next-heartbeat`

### POST /hooks/agent

分離されたエージェントターンを実行します。

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Summarize inbox","name":"Email","model":"openai/gpt-5.4-mini"}'
```

フィールド: `message`（必須）、`name`、`agentId`、`wakeMode`、`deliver`、`channel`、`to`、`model`、`thinking`、`timeoutSeconds`。

### マップされたフック（POST /hooks/\<name\>）

カスタムフック名は、設定内の`hooks.mappings`を介して解決されます。マッピングでは、テンプレートまたはコード変換を使って任意のペイロードを`wake`または`agent`アクションに変換できます。

### セキュリティ

- フックエンドポイントはloopback、tailnet、または信頼できるリバースプロキシの背後に置いてください。
- 専用のフックトークンを使用してください。gateway認証トークンを再利用しないでください。
- `hooks.path`は専用のサブパスにしてください。`/`は拒否されます。
- 明示的な`agentId`ルーティングを制限するには`hooks.allowedAgentIds`を設定してください。
- 呼び出し元がセッションを選択する必要がない限り、`hooks.allowRequestSessionKey=false`を維持してください。
- `hooks.allowRequestSessionKey`を有効にする場合は、許可されるセッションキーの形状を制約するために`hooks.allowedSessionKeyPrefixes`も設定してください。
- フックペイロードはデフォルトで安全境界によりラップされます。

## Gmail PubSub連携

Google PubSubを介してGmail受信トリガーをOpenClawに接続します。

**前提条件**: `gcloud` CLI、`gog`（gogcli）、OpenClaw hooks有効化、公開HTTPSエンドポイント用のTailscale。

### ウィザードセットアップ（推奨）

```bash
openclaw webhooks gmail setup --account openclaw@gmail.com
```

これにより`hooks.gmail`設定が書き込まれ、Gmailプリセットが有効になり、pushエンドポイントにはTailscale Funnelが使用されます。

### Gateway自動起動

`hooks.enabled=true`かつ`hooks.gmail.account`が設定されている場合、Gatewayは起動時に`gog gmail watch serve`を開始し、watchを自動更新します。無効化するには`OPENCLAW_SKIP_GMAIL_WATCHER=1`を設定してください。

### 手動の一回限りセットアップ

1. `gog`が使用するOAuthクライアントを所有するGCPプロジェクトを選択します。

```bash
gcloud auth login
gcloud config set project <project-id>
gcloud services enable gmail.googleapis.com pubsub.googleapis.com
```

2. トピックを作成し、Gmailにpushアクセスを付与します。

```bash
gcloud pubsub topics create gog-gmail-watch
gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
  --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
  --role=roles/pubsub.publisher
```

3. watchを開始します。

```bash
gog gmail watch start \
  --account openclaw@gmail.com \
  --label INBOX \
  --topic projects/<project-id>/topics/gog-gmail-watch
```

### Gmailモデルオーバーライド

```json5
{
  hooks: {
    gmail: {
      model: "openrouter/meta-llama/llama-3.3-70b-instruct:free",
      thinking: "off",
    },
  },
}
```

## ジョブの管理

```bash
# すべてのジョブを一覧表示
openclaw cron list

# ジョブを編集
openclaw cron edit <jobId> --message "Updated prompt" --model "opus"

# ジョブを今すぐ強制実行
openclaw cron run <jobId>

# 期限到来時のみ実行
openclaw cron run <jobId> --due

# 実行履歴を表示
openclaw cron runs --id <jobId> --limit 50

# ジョブを削除
openclaw cron remove <jobId>

# エージェント選択（マルチエージェント構成）
openclaw cron add --name "Ops sweep" --cron "0 6 * * *" --session isolated --message "Check ops queue" --agent ops
openclaw cron edit <jobId> --clear-agent
```

モデルオーバーライドに関する注意:

- `openclaw cron add|edit --model ...`はジョブの選択モデルを変更します。
- モデルが許可されている場合、その正確なprovider/modelが分離エージェント実行に渡されます。
- 許可されていない場合、cronは警告を出し、そのジョブのエージェント/デフォルトのモデル選択にフォールバックします。
- 設定されたフォールバックチェーンは引き続き適用されますが、明示的なジョブ単位フォールバックリストのない単純な`--model`オーバーライドでは、無言の追加リトライ先としてエージェントprimaryにフォールスルーしなくなりました。

## 設定

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1,
    retry: {
      maxAttempts: 3,
      backoffMs: [60000, 120000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "server_error"],
    },
    webhookToken: "専用のWebhookトークンに置き換えてください",
    sessionRetention: "24h",
    runLog: { maxBytes: "2mb", keepLines: 2000 },
  },
}
```

cronを無効にするには: `cron.enabled: false` または `OPENCLAW_SKIP_CRON=1`。

**1回限りのリトライ**: 一時的なエラー（レート制限、過負荷、ネットワーク、サーバーエラー）は指数バックオフで最大3回まで再試行されます。永続的なエラーは即座に無効化されます。

**定期実行のリトライ**: 再試行の間に指数バックオフ（30秒〜60分）が適用されます。次回の実行が成功するとバックオフはリセットされます。

**メンテナンス**: `cron.sessionRetention`（デフォルトは`24h`）は分離実行のセッションエントリを削除します。`cron.runLog.maxBytes` / `cron.runLog.keepLines`は実行ログファイルを自動的に削除します。

## トラブルシューティング

### コマンドの手順

```bash
openclaw status
openclaw gateway status
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
openclaw doctor
```

### cronが実行されない

- `cron.enabled`と`OPENCLAW_SKIP_CRON`環境変数を確認してください。
- Gatewayが継続的に実行されていることを確認してください。
- `cron`スケジュールでは、タイムゾーン（`--tz`）とホストのタイムゾーンを確認してください。
- 実行出力の`reason: not-due`は、手動実行が`openclaw cron run <jobId> --due`で確認され、そのジョブの期限がまだ来ていなかったことを意味します。

### cronは実行されたが配信されない

- 配信モードが`none`の場合、外部メッセージは想定されません。
- 配信先が欠落または無効（`channel`/`to`）の場合、送信はスキップされます。
- チャネル認証エラー（`unauthorized`、`Forbidden`）は、認証情報によって配信がブロックされたことを意味します。
- 分離実行がサイレントトークン（`NO_REPLY` / `no_reply`）のみを返した場合、OpenClawは直接の外部配信を抑制し、フォールバックのキュー済み要約経路も抑制するため、チャットには何も投稿されません。
- cronが管理する分離ジョブでは、フォールバックとしてエージェントがmessageツールを使うことを期待しないでください。ランナーが最終配信を管理し、`--no-deliver`は直接送信を許可する代わりに内部処理のままにします。

### タイムゾーンの注意点

- `--tz`なしのcronはgatewayホストのタイムゾーンを使用します。
- タイムゾーンなしの`at`スケジュールはUTCとして扱われます。
- heartbeatの`activeHours`は設定済みタイムゾーン解決を使用します。

## 関連

- [Automation & Tasks](/ja-JP/automation) — すべての自動化メカニズムの概要
- [Background Tasks](/ja-JP/automation/tasks) — cron実行のタスク台帳
- [Heartbeat](/ja-JP/gateway/heartbeat) — 定期的なメインセッションのターン
- [Timezone](/ja-JP/concepts/timezone) — タイムゾーン設定
