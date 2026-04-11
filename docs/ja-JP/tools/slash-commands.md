---
read_when:
    - チャットコマンドを使用または設定すること
    - コマンドのルーティングまたは権限をデバッグすること
summary: 'スラッシュコマンド: テキストとネイティブ、設定、およびサポートされているコマンド'
title: スラッシュコマンド
x-i18n:
    generated_at: "2026-04-11T02:48:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2cc346361c3b1a63aae9ec0f28706f4cb0b866b6c858a3999101f6927b923b4a
    source_path: tools/slash-commands.md
    workflow: 15
---

# スラッシュコマンド

コマンドは Gateway によって処理されます。ほとんどのコマンドは `/` で始まる**単独の**メッセージとして送信する必要があります。
host 専用の bash チャットコマンドは `! <cmd>` を使用します（`/bash <cmd>` はそのエイリアスです）。

関連する 2 つの仕組みがあります。

- **Commands**: 単独の `/...` メッセージ。
- **Directives**: `/think`, `/fast`, `/verbose`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`。
  - Directives は、モデルがメッセージを見る前に取り除かれます。
  - 通常のチャットメッセージ内（directive のみではない場合）では、「インラインヒント」として扱われ、セッション設定は永続化されません。
  - directive のみのメッセージ（メッセージが directives だけを含む場合）では、セッションに永続化され、確認応答が返されます。
  - Directives は**認可された送信者**に対してのみ適用されます。`commands.allowFrom` が設定されている場合、それが唯一の
    allowlist として使われます。そうでない場合、認可は channel allowlists/pairing と `commands.useAccessGroups` に基づきます。
    認可されていない送信者では、directives は通常のテキストとして扱われます。

いくつかの **インラインショートカット** もあります（allowlist 済み/認可済み送信者のみ）: `/help`, `/commands`, `/status`, `/whoami` (`/id`)。
これらは即座に実行され、モデルがメッセージを見る前に取り除かれ、残りのテキストは通常フローで続行されます。

## 設定

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: true,
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw",
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text`（デフォルト `true`）は、チャットメッセージ内での `/...` の解析を有効にします。
  - ネイティブコマンドを持たない surface（WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams）では、これを `false` に設定しても text commands は引き続き動作します。
- `commands.native`（デフォルト `"auto"`）は、ネイティブコマンドを登録します。
  - Auto: Discord/Telegram ではオン、Slack ではオフ（slash commands を追加するまでは）。ネイティブ対応のない provider では無視されます。
  - provider ごとに上書きするには、`channels.discord.commands.native`、`channels.telegram.commands.native`、または `channels.slack.commands.native` を設定します（bool または `"auto"`）。
  - `false` は、起動時に Discord/Telegram で以前登録されたコマンドを消去します。Slack commands は Slack app 側で管理され、自動削除はされません。
- `commands.nativeSkills`（デフォルト `"auto"`）は、対応している場合に **skill** commands をネイティブ登録します。
  - Auto: Discord/Telegram ではオン、Slack ではオフ（Slack は skill ごとに slash command の作成が必要）。
  - provider ごとに上書きするには、`channels.discord.commands.nativeSkills`、`channels.telegram.commands.nativeSkills`、または `channels.slack.commands.nativeSkills` を設定します（bool または `"auto"`）。
- `commands.bash`（デフォルト `false`）は、`! <cmd>` による host shell command 実行を有効にします（`/bash <cmd>` はエイリアス。`tools.elevated` allowlists が必要）。
- `commands.bashForegroundMs`（デフォルト `2000`）は、bash が background mode に切り替わるまでどれだけ待つかを制御します（`0` で即座に background 化）。
- `commands.config`（デフォルト `false`）は `/config` を有効にします（`openclaw.json` の読み書き）。
- `commands.mcp`（デフォルト `false`）は `/mcp` を有効にします（`mcp.servers` 配下の OpenClaw 管理 MCP config の読み書き）。
- `commands.plugins`（デフォルト `false`）は `/plugins` を有効にします（plugin の検出/状態確認、および install + enable/disable 制御）。
- `commands.debug`（デフォルト `false`）は `/debug` を有効にします（runtime 専用の上書き）。
- `commands.restart`（デフォルト `true`）は `/restart` と gateway restart tool actions を有効にします。
- `commands.ownerAllowFrom`（任意）は、owner 専用 command/tool surface 向けの明示的な owner allowlist を設定します。これは `commands.allowFrom` とは別です。
- `commands.ownerDisplay` は、system prompt 内で owner ids をどう表示するかを制御します: `raw` または `hash`。
- `commands.ownerDisplaySecret` は、`commands.ownerDisplay="hash"` の場合に使用する HMAC secret を任意で設定します。
- `commands.allowFrom`（任意）は、command 認可のための provider ごとの allowlist を設定します。これが設定されている場合、
  command と directives の認可元はこれ**のみ**になります（channel allowlists/pairing と `commands.useAccessGroups` は無視されます）。グローバルデフォルトには `"*"` を使い、provider 固有キーがそれを上書きします。
- `commands.useAccessGroups`（デフォルト `true`）は、`commands.allowFrom` が設定されていない場合に、command に対して allowlists/policies を適用します。

## コマンド一覧

現在のソースオブトゥルース:

- core 組み込みコマンドは `src/auto-reply/commands-registry.shared.ts` から来ます
- 生成された dock commands は `src/auto-reply/commands-registry.data.ts` から来ます
- plugin commands は plugin の `registerCommand()` 呼び出しから来ます
- 実際にあなたの gateway で利用可能かどうかは、引き続き config flags、channel surface、およびインストール/有効化された plugins に依存します

### Core 組み込みコマンド

現在利用可能な組み込みコマンド:

- `/new [model]` は新しいセッションを開始します。`/reset` は reset のエイリアスです。
- `/compact [instructions]` はセッションコンテキストを compact します。[/concepts/compaction](/ja-JP/concepts/compaction) を参照してください。
- `/stop` は現在の run を中止します。
- `/session idle <duration|off>` と `/session max-age <duration|off>` は thread-binding の有効期限を管理します。
- `/think <off|minimal|low|medium|high|xhigh>` は thinking level を設定します。エイリアス: `/thinking`, `/t`。
- `/verbose on|off|full` は verbose 出力を切り替えます。エイリアス: `/v`。
- `/fast [status|on|off]` は fast mode を表示または設定します。
- `/reasoning [on|off|stream]` は reasoning の可視性を切り替えます。エイリアス: `/reason`。
- `/elevated [on|off|ask|full]` は elevated mode を切り替えます。エイリアス: `/elev`。
- `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` は exec defaults を表示または設定します。
- `/model [name|#|status]` は model を表示または設定します。
- `/models [provider] [page] [limit=<n>|size=<n>|all]` は provider 一覧、または provider の models を表示します。
- `/queue <mode>` は queue 動作（`steer`, `interrupt`, `followup`, `collect`, `steer-backlog`）と、`debounce:2s cap:25 drop:summarize` のようなオプションを管理します。
- `/help` は短い help 要約を表示します。
- `/commands` は生成された command catalog を表示します。
- `/tools [compact|verbose]` は、現在の agent が今使えるものを表示します。
- `/status` は runtime status を表示します。利用可能な場合は provider の usage/quota も含みます。
- `/tasks` は現在のセッションのアクティブ/最近の background tasks を一覧表示します。
- `/context [list|detail|json]` は、context がどのように組み立てられるかを説明します。
- `/export-session [path]` は現在のセッションを HTML にエクスポートします。エイリアス: `/export`。
- `/whoami` はあなたの sender id を表示します。エイリアス: `/id`。
- `/skill <name> [input]` は skill を名前で実行します。
- `/allowlist [list|add|remove] ...` は allowlist エントリを管理します。text-only。
- `/approve <id> <decision>` は exec approval prompt を解決します。
- `/btw <question>` は、将来のセッションコンテキストを変更せずに脇道の質問をします。[/tools/btw](/ja-JP/tools/btw) を参照してください。
- `/subagents list|kill|log|info|send|steer|spawn` は現在のセッションの sub-agent runs を管理します。
- `/acp spawn|cancel|steer|close|sessions|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|help` は ACP セッションと runtime options を管理します。
- `/focus <target>` は現在の Discord thread または Telegram topic/conversation をセッション target にバインドします。
- `/unfocus` は現在の binding を削除します。
- `/agents` は現在のセッションに thread-bound された agents を一覧表示します。
- `/kill <id|#|all>` は 1 つまたはすべての実行中 sub-agent を中止します。
- `/steer <id|#> <message>` は実行中の sub-agent にステアリングを送ります。エイリアス: `/tell`。
- `/config show|get|set|unset` は `openclaw.json` を読み書きします。owner-only。`commands.config: true` が必要です。
- `/mcp show|get|set|unset` は `mcp.servers` 配下の OpenClaw 管理 MCP server config を読み書きします。owner-only。`commands.mcp: true` が必要です。
- `/plugins list|inspect|show|get|install|enable|disable` は plugin 状態を確認または変更します。`/plugin` はエイリアスです。書き込みは owner-only。`commands.plugins: true` が必要です。
- `/debug show|set|unset|reset` は runtime 専用 config overrides を管理します。owner-only。`commands.debug: true` が必要です。
- `/usage off|tokens|full|cost` はレスポンスごとの usage footer を制御するか、ローカル cost summary を表示します。
- `/tts on|off|status|provider|limit|summary|audio|help` は TTS を制御します。[/tools/tts](/ja-JP/tools/tts) を参照してください。
- `/restart` は、有効な場合に OpenClaw を再起動します。デフォルトは有効です。無効にするには `commands.restart: false` を設定します。
- `/activation mention|always` は group activation mode を設定します。
- `/send on|off|inherit` は send policy を設定します。owner-only。
- `/bash <command>` は host shell command を実行します。text-only。エイリアス: `! <command>`。`commands.bash: true` と `tools.elevated` allowlists が必要です。
- `!poll [sessionId]` は background bash job を確認します。
- `!stop [sessionId]` は background bash job を停止します。

### 生成された dock commands

Dock commands は、native-command をサポートする channel plugins から生成されます。現在の bundled セット:

- `/dock-discord`（エイリアス: `/dock_discord`）
- `/dock-mattermost`（エイリアス: `/dock_mattermost`）
- `/dock-slack`（エイリアス: `/dock_slack`）
- `/dock-telegram`（エイリアス: `/dock_telegram`）

### Bundled plugin commands

bundled plugins はさらに slash commands を追加できます。このリポジトリにある現在の bundled commands:

- `/dreaming [on|off|status|help]` は memory dreaming を切り替えます。[Dreaming](/ja-JP/concepts/dreaming) を参照してください。
- `/pair [qr|status|pending|approve|cleanup|notify]` はデバイスの pairing/setup flow を管理します。[Pairing](/ja-JP/channels/pairing) を参照してください。
- `/phone status|arm <camera|screen|writes|all> [duration]|disarm` は高リスク phone node commands を一時的に arm します。
- `/voice status|list [limit]|set <voiceId|name>` は Talk voice config を管理します。Discord では、ネイティブ command 名は `/talkvoice` です。
- `/card ...` は LINE rich card プリセットを送信します。[LINE](/ja-JP/channels/line) を参照してください。
- `/codex status|models|threads|resume|compact|review|account|mcp|skills` は bundled Codex app-server harness を確認および制御します。[Codex Harness](/ja-JP/plugins/codex-harness) を参照してください。
- QQBot 専用コマンド:
  - `/bot-ping`
  - `/bot-version`
  - `/bot-help`
  - `/bot-upgrade`
  - `/bot-logs`

### 動的 skill commands

ユーザーが呼び出せる skills も slash commands として公開されます。

- `/skill <name> [input]` は、汎用エントリポイントとして常に使えます。
- skills は、skill/plugin が登録していれば `/prose` のような直接コマンドとして現れることもあります。
- ネイティブ skill-command 登録は `commands.nativeSkills` と `channels.<provider>.commands.nativeSkills` によって制御されます。

注記:

- コマンドは、コマンドと引数の間に任意で `:` を受け付けます（例: `/think: high`, `/send: on`, `/help:`）。
- `/new <model>` は model alias、`provider/model`、または provider 名（あいまい一致）を受け付けます。一致しない場合、そのテキストはメッセージ本文として扱われます。
- provider usage の完全な内訳を見るには、`openclaw status --usage` を使用してください。
- `/allowlist add|remove` は `commands.config=true` を必要とし、channel の `configWrites` に従います。
- マルチアカウント channel では、config 対象の `/allowlist --account <id>` と `/config set channels.<provider>.accounts.<id>...` も、対象 account の `configWrites` に従います。
- `/usage` はレスポンスごとの usage footer を制御します。`/usage cost` は OpenClaw session logs からローカル cost summary を表示します。
- `/restart` はデフォルトで有効です。無効にするには `commands.restart: false` を設定します。
- `/plugins install <spec>` は `openclaw plugins install` と同じ plugin spec を受け付けます: ローカル path/archive、npm package、または `clawhub:<pkg>`。
- `/plugins enable|disable` は plugin config を更新し、再起動を促す場合があります。
- Discord 専用ネイティブコマンド: `/vc join|leave|status` は voice channels を制御します（`channels.discord.voice` と native commands が必要。text としては利用不可）。
- Discord の thread-binding コマンド（`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`）は、実効的に thread bindings が有効である必要があります（`session.threadBindings.enabled` および/または `channels.discord.threadBindings.enabled`）。
- ACP コマンドのリファレンスと runtime 動作: [ACP Agents](/ja-JP/tools/acp-agents)。
- `/verbose` はデバッグや追加の可視化を目的としています。通常利用では **off** のままにしてください。
- `/fast on|off` はセッション上書きを永続化します。解除して config デフォルトに戻すには、Sessions UI の `inherit` オプションを使用してください。
- `/fast` は provider 固有です。OpenAI/OpenAI Codex ではネイティブ Responses endpoint 上で `service_tier=priority` にマップされ、direct public Anthropic リクエストでは、`api.anthropic.com` に送られる OAuth 認証トラフィックを含めて `service_tier=auto` または `standard_only` にマップされます。[OpenAI](/ja-JP/providers/openai) と [Anthropic](/ja-JP/providers/anthropic) を参照してください。
- tool failure の要約は関係がある場合は引き続き表示されますが、詳細な failure テキストが含まれるのは `/verbose` が `on` または `full` のときだけです。
- `/reasoning`（および `/verbose`）は group 設定ではリスクがあります。意図しない internal reasoning や tool 出力を露出する可能性があります。特に group chats では、off のままにしておくことを推奨します。
- `/model` は新しい session model を即座に永続化します。
- agent が idle なら、次の run ですぐにそれが使われます。
- すでに run がアクティブな場合、OpenClaw は live switch を保留中としてマークし、クリーンな retry point でのみ新しい model へ再起動します。
- tool activity または reply output がすでに始まっている場合、その保留中スイッチは後の retry 機会または次の user turn までキューされたままになることがあります。
- **Fast path:** allowlist 済み送信者からのコマンドのみのメッセージは即座に処理されます（queue + model をバイパス）。
- **Group mention gating:** allowlist 済み送信者からのコマンドのみのメッセージは mention 要件をバイパスします。
- **Inline shortcuts（allowlist 済み送信者のみ）:** 一部のコマンドは通常メッセージに埋め込まれていても動作し、残りのテキストをモデルが見る前に取り除かれます。
  - 例: `hey /status` は status reply をトリガーし、残りのテキストは通常フローで続行されます。
- 現在の対象: `/help`, `/commands`, `/status`, `/whoami` (`/id`)。
- 認可されていないコマンドのみのメッセージは黙って無視され、インライン `/...` トークンは通常テキストとして扱われます。
- **Skill commands:** `user-invocable` な Skills は slash commands として公開されます。名前は `a-z0-9_` にサニタイズされ（最大 32 文字）、衝突時は数値サフィックスが付きます（例: `_2`）。
  - `/skill <name> [input]` は skill を名前で実行します（ネイティブ command 制限により skill ごとの commands が使えない場合に便利です）。
  - デフォルトでは、skill commands は通常のリクエストとして model に転送されます。
  - Skills はオプションで `command-dispatch: tool` を宣言でき、その場合 command を tool に直接ルーティングします（決定的で、model なし）。
  - 例: `/prose`（OpenProse plugin）— [OpenProse](/ja-JP/prose) を参照してください。
- **Native command arguments:** Discord は動的オプションに autocomplete を使います（必須引数を省略した場合は button menus も使用）。Telegram と Slack は、コマンドが選択肢をサポートしていて引数を省略した場合に button menu を表示します。

## `/tools`

`/tools` が答えるのは config の質問ではなく、runtime の質問です: **この会話でこの agent が今すぐ使えるものは何か**。

- デフォルトの `/tools` は compact で、すばやく確認できるよう最適化されています。
- `/tools verbose` は短い説明を追加します。
- 引数をサポートする native-command surface では、同じ `compact|verbose` モード切り替えが公開されます。
- 結果は session スコープであるため、agent、channel、thread、sender 認可、または model を変更すると出力も変わることがあります。
- `/tools` には、core tools、接続された plugin tools、channel 所有の tools を含む、runtime で実際に到達可能な tools が含まれます。

profile や override の編集には、`/tools` を静的 catalog として扱うのではなく、Control UI の Tools パネルまたは config/catalog surface を使用してください。

## Usage surface（どこに何が表示されるか）

- **Provider usage/quota**（例: 「Claude 80% left」）は、usage tracking が有効なとき、現在の model provider に対して `/status` に表示されます。OpenClaw は provider の window を `% left` に正規化します。MiniMax では remaining-only の percent フィールドは表示前に反転され、`model_remains` レスポンスでは model-tagged な plan label とともに chat-model エントリが優先されます。
- `/status` 内の **Token/cache 行** は、live session snapshot が疎な場合、最新の transcript usage エントリにフォールバックできます。既存の非ゼロ live 値が引き続き優先され、保存済み total が欠けているか小さい場合には、transcript フォールバックによりアクティブな runtime model label と、より大きい prompt 指向 total も復元できます。
- **レスポンスごとの tokens/cost** は `/usage off|tokens|full` で制御されます（通常の replies に付加されます）。
- `/model status` は **models/auth/endpoints** に関するものであり、usage ではありません。

## モデル選択（`/model`）

`/model` は directive として実装されています。

例:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

注記:

- `/model` と `/model list` は、compact で番号付きの picker（model family + 利用可能な providers）を表示します。
- Discord では、`/model` と `/models` は provider と model の dropdown、および Submit ステップを持つ対話型 picker を開きます。
- `/model <#>` はその picker から選択します（可能であれば現在の provider を優先します）。
- `/model status` は、設定された provider endpoint（`baseUrl`）と API mode（`api`）を含む詳細ビューを表示します（利用可能な場合）。

## デバッグ上書き

`/debug` では **runtime のみ** の config 上書き（ディスクではなくメモリ）を設定できます。owner-only。デフォルトでは無効で、`commands.debug: true` で有効化します。

例:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

注記:

- 上書きは新しい config 読み取りに即時適用されますが、`openclaw.json` には書き込まれません。
- すべての上書きを消してディスク上の config に戻るには `/debug reset` を使用してください。

## Config 更新

`/config` はオンディスクの config（`openclaw.json`）に書き込みます。owner-only。デフォルトでは無効で、`commands.config: true` で有効化します。

例:

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

注記:

- Config は書き込み前に検証され、不正な変更は拒否されます。
- `/config` の更新は再起動後も保持されます。

## MCP 更新

`/mcp` は `mcp.servers` 配下の OpenClaw 管理 MCP server 定義を書き込みます。owner-only。デフォルトでは無効で、`commands.mcp: true` で有効化します。

例:

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

注記:

- `/mcp` は OpenClaw config に保存され、Pi 所有の project settings には保存されません。
- 実際にどの transport が実行可能かは runtime adapters が決定します。

## Plugin 更新

`/plugins` では operator が検出済み plugins を確認し、config 内の有効化状態を切り替えられます。読み取り専用フローでは `/plugin` をエイリアスとして使えます。デフォルトでは無効で、`commands.plugins: true` で有効化します。

例:

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

注記:

- `/plugins list` と `/plugins show` は、現在の workspace とオンディスク config に対する実際の plugin discovery を使用します。
- `/plugins enable|disable` は plugin config のみを更新し、plugin の install や uninstall は行いません。
- enable/disable の変更後は、適用のために gateway を再起動してください。

## Surface に関する注記

- **Text commands** は通常の chat session で実行されます（DM は `main` を共有し、groups は独自の session を持ちます）。
- **Native commands** は分離された sessions を使用します:
  - Discord: `agent:<agentId>:discord:slash:<userId>`
  - Slack: `agent:<agentId>:slack:slash:<userId>`（prefix は `channels.slack.slashCommand.sessionPrefix` で設定可能）
  - Telegram: `telegram:slash:<userId>`（`CommandTargetSessionKey` を通じて chat session を対象にします）
- **`/stop`** はアクティブな chat session を対象にして、現在の run を中止できるようにします。
- **Slack:** `channels.slack.slashCommand` は、単一の `/openclaw` 形式 command 用として引き続きサポートされています。`commands.native` を有効にする場合は、組み込み command ごとに 1 つの Slack slash command を作成する必要があります（名前は `/help` などと同じ）。Slack 向けの command 引数メニューは ephemeral Block Kit buttons として配信されます。
  - Slack のネイティブ例外: Slack が `/status` を予約しているため、`/status` ではなく `/agentstatus` を登録します。text の `/status` は Slack メッセージ内では引き続き動作します。

## BTW の脇道質問

`/btw` は、現在の session に関する素早い **脇道の質問** です。

通常の chat と異なり、これは:

- 現在の session を背景コンテキストとして使用し、
- tool なしの独立した one-shot call として実行され、
- 将来の session context を変更せず、
- transcript history に書き込まれず、
- 通常の assistant message ではなく、ライブの side result として配信されます。

そのため、メインタスクを進めたまま一時的な確認をしたいときに `/btw` が役立ちます。

例:

```text
/btw 今何をしているんだっけ？
```

完全な動作と client UX の詳細については [BTW Side Questions](/ja-JP/tools/btw) を参照してください。
