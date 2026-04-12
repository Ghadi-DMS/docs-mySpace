---
read_when:
    - 思考、fast モード、または verbose ディレクティブの解析やデフォルトを調整する場合
summary: '`/think`、`/fast`、`/verbose`、`/trace`、および推論可視性のためのディレクティブ構文'
title: 思考レベル
x-i18n:
    generated_at: "2026-04-12T23:34:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4f3b1341281f07ba4e9061e3355845dca234be04cc0d358594312beeb7676e68
    source_path: tools/thinking.md
    workflow: 15
---

# 思考レベル（`/think` ディレクティブ）

## できること

- 任意の受信本文で使えるインラインディレクティブ: `/t <level>`、`/think:<level>`、または `/thinking <level>`。
- レベル（エイリアス）: `off | minimal | low | medium | high | xhigh | adaptive`
  - minimal → 「think」
  - low → 「think hard」
  - medium → 「think harder」
  - high → 「ultrathink」（最大予算）
  - xhigh → 「ultrathink+」（GPT-5.2 + Codex モデルのみ）
  - adaptive → provider が管理する適応型推論予算（Anthropic Claude 4.6 モデルファミリーでサポート）
  - `x-high`、`x_high`、`extra-high`、`extra high`、`extra_high` は `xhigh` にマップされます。
  - `highest`、`max` は `high` にマップされます。
- provider に関する注意:
  - Anthropic Claude 4.6 モデルは、明示的な thinking レベルが設定されていない場合、デフォルトで `adaptive` になります。
  - Anthropic 互換ストリーミング経路上の MiniMax（`minimax/*`）は、モデル params またはリクエスト params で明示的に thinking を設定しない限り、デフォルトで `thinking: { type: "disabled" }` になります。これにより、MiniMax のネイティブではない Anthropic ストリーム形式から `reasoning_content` delta が漏れるのを防ぎます。
  - Z.AI（`zai/*`）はバイナリ thinking（`on`/`off`）のみをサポートします。`off` 以外のレベルはすべて `on` として扱われます（`low` にマップ）。
  - Moonshot（`moonshot/*`）は `/think off` を `thinking: { type: "disabled" }` に、`off` 以外の任意のレベルを `thinking: { type: "enabled" }` にマップします。thinking が有効な場合、Moonshot は `tool_choice` として `auto|none` のみを受け付けます。OpenClaw は互換性のない値を `auto` に正規化します。

## 解決順序

1. メッセージ上のインラインディレクティブ（そのメッセージにのみ適用）。
2. セッションオーバーライド（ディレクティブのみのメッセージを送って設定）。
3. エージェントごとのデフォルト（設定の `agents.list[].thinkingDefault`）。
4. グローバルデフォルト（設定の `agents.defaults.thinkingDefault`）。
5. フォールバック: Anthropic Claude 4.6 モデルは `adaptive`、その他の推論対応モデルは `low`、それ以外は `off`。

## セッションデフォルトを設定する

- **ディレクティブだけ**のメッセージを送ります（空白は可）。例: `/think:medium` または `/t high`。
- これは現在のセッションに固定されます（デフォルトでは送信者ごと）。`/think:off` またはセッションのアイドルリセットで解除されます。
- 確認返信が送られます（`Thinking level set to high.` / `Thinking disabled.`）。レベルが無効な場合（例: `/thinking big`）、コマンドはヒント付きで拒否され、セッション状態は変更されません。
- 現在の thinking レベルを確認するには、引数なしで `/think`（または `/think:`）を送ります。

## エージェントごとの適用

- **Embedded Pi**: 解決されたレベルは、プロセス内の Pi エージェントランタイムに渡されます。

## Fast モード（`/fast`）

- レベル: `on|off`。
- ディレクティブのみのメッセージは、セッションの fast モードオーバーライドを切り替え、`Fast mode enabled.` / `Fast mode disabled.` と返信します。
- 現在有効な fast モード状態を確認するには、モードなしで `/fast`（または `/fast status`）を送ります。
- OpenClaw は、次の順序で fast モードを解決します:
  1. インライン/ディレクティブのみの `/fast on|off`
  2. セッションオーバーライド
  3. エージェントごとのデフォルト（`agents.list[].fastModeDefault`）
  4. モデルごとの設定: `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. フォールバック: `off`
- `openai/*` では、fast モードは対応する Responses リクエストで `service_tier=priority` を送信することで、OpenAI の優先処理にマップされます。
- `openai-codex/*` では、fast モードは Codex Responses でも同じ `service_tier=priority` フラグを送信します。OpenClaw は、両方の認証経路で 1 つの共有 `/fast` トグルを維持します。
- `api.anthropic.com` に送信される OAuth 認証済みトラフィックを含む直接の公開 `anthropic/*` リクエストでは、fast モードは Anthropic の service tier にマップされます: `/fast on` は `service_tier=auto` を設定し、`/fast off` は `service_tier=standard_only` を設定します。
- Anthropic 互換経路上の `minimax/*` では、`/fast on`（または `params.fastMode: true`）は `MiniMax-M2.7` を `MiniMax-M2.7-highspeed` に書き換えます。
- 明示的な Anthropic `serviceTier` / `service_tier` モデル params は、両方が設定されている場合、fast モードのデフォルトを上書きします。OpenClaw は、Anthropic 以外のプロキシ base URL に対しては引き続き Anthropic の service-tier 注入をスキップします。

## Verbose ディレクティブ（`/verbose` または `/v`）

- レベル: `on`（最小） | `full` | `off`（デフォルト）。
- ディレクティブのみのメッセージは、セッション verbose を切り替え、`Verbose logging enabled.` / `Verbose logging disabled.` と返信します。無効なレベルは、状態を変更せずにヒントを返します。
- `/verbose off` は明示的なセッションオーバーライドを保存します。Sessions UI で `inherit` を選ぶと解除できます。
- インラインディレクティブはそのメッセージにのみ影響し、それ以外はセッション/グローバルデフォルトが適用されます。
- 現在の verbose レベルを確認するには、引数なしで `/verbose`（または `/verbose:`）を送ります。
- verbose が有効な場合、構造化ツール結果を出力するエージェント（Pi、その他の JSON エージェント）は、各ツール呼び出しを個別のメタデータ専用メッセージとして返します。利用可能な場合、`<emoji> <tool-name>: <arg>`（パス/コマンド）の形式でプレフィックスが付きます。これらのツール要約は、各ツールの開始時にすぐ送信されます（別々のバブル）。ストリーミング delta としては送信されません。
- ツール失敗の要約は通常モードでも表示されたままですが、生のエラー詳細サフィックスは verbose が `on` または `full` の場合にのみ表示されます。
- verbose が `full` の場合、ツール出力も完了後に転送されます（別バブル、長さは安全な範囲に切り詰め）。実行中に `/verbose on|full|off` を切り替えた場合、後続のツールバブルは新しい設定に従います。

## Plugin trace ディレクティブ（`/trace`）

- レベル: `on` | `off`（デフォルト）。
- ディレクティブのみのメッセージは、セッションの Plugin trace 出力を切り替え、`Plugin trace enabled.` / `Plugin trace disabled.` と返信します。
- インラインディレクティブはそのメッセージにのみ影響し、それ以外はセッション/グローバルデフォルトが適用されます。
- 現在の trace レベルを確認するには、引数なしで `/trace`（または `/trace:`）を送ります。
- `/trace` は `/verbose` より狭い範囲です。Active Memory のデバッグサマリーのような、Plugin 所有の trace/debug 行だけを公開します。
- trace 行は `/status` に表示されることがあり、通常のアシスタント返信の後続診断メッセージとして現れることもあります。

## 推論の可視性（`/reasoning`）

- レベル: `on|off|stream`。
- ディレクティブのみのメッセージは、返信で thinking ブロックを表示するかどうかを切り替えます。
- 有効にすると、推論は `Reasoning:` で始まる**別メッセージ**として送信されます。
- `stream`（Telegram のみ）: 返信生成中に Telegram のドラフトバブルへ推論をストリーミングし、その後、推論なしの最終回答を送信します。
- エイリアス: `/reason`。
- 現在の reasoning レベルを確認するには、引数なしで `/reasoning`（または `/reasoning:`）を送ります。
- 解決順序: インラインディレクティブ、セッションオーバーライド、エージェントごとのデフォルト（`agents.list[].reasoningDefault`）、フォールバック（`off`）。

## 関連

- Elevated mode のドキュメントは [Elevated mode](/ja-JP/tools/elevated) にあります。

## Heartbeat

- Heartbeat プローブ本文は、設定された heartbeat プロンプトです（デフォルト: `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`）。heartbeat メッセージ内のインラインディレクティブは通常どおり適用されます（ただし、heartbeat からセッションデフォルトを変更するのは避けてください）。
- Heartbeat 配信はデフォルトで最終ペイロードのみです。利用可能な場合に別の `Reasoning:` メッセージも送信するには、`agents.defaults.heartbeat.includeReasoning: true` またはエージェントごとの `agents.list[].heartbeat.includeReasoning: true` を設定してください。

## Web チャット UI

- Web チャットの thinking セレクターは、ページ読み込み時に受信セッションストア/設定から、そのセッションに保存されたレベルを反映します。
- 別のレベルを選ぶと、`sessions.patch` を通じてセッションオーバーライドが即座に書き込まれます。次の送信までは待たず、単発の `thinkingOnce` オーバーライドでもありません。
- 最初のオプションは常に `Default (<resolved level>)` で、解決されたデフォルトはアクティブなセッションモデルから決まります: Anthropic/Bedrock 上の Claude 4.6 では `adaptive`、その他の推論対応モデルでは `low`、それ以外では `off` です。
- ピッカーは provider 対応のままです:
  - ほとんどの provider は `off | minimal | low | medium | high | adaptive` を表示します
  - Z.AI はバイナリの `off | on` を表示します
- `/think:<level>` も引き続き機能し、同じ保存済みセッションレベルを更新するため、チャットディレクティブとピッカーは同期されたままです。
