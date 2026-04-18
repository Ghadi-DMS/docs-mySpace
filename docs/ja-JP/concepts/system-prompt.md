---
read_when:
    - システムプロンプトのテキスト、ツール一覧、または時刻/Heartbeat セクションの編集
    - ワークスペースのブートストラップや Skills の注入動作の変更
summary: OpenClaw のシステムプロンプトに含まれる内容と、その組み立て方法
title: システムプロンプト
x-i18n:
    generated_at: "2026-04-18T04:40:07Z"
    model: gpt-5.4
    provider: openai
    source_hash: e60705994cebdd9768926168cb1c6d17ab717d7ff02353a5d5e7478ba8191cab
    source_path: concepts/system-prompt.md
    workflow: 15
---

# システムプロンプト

OpenClaw は、各エージェント実行ごとにカスタムのシステムプロンプトを構築します。このプロンプトは **OpenClaw が所有** しており、pi-coding-agent のデフォルトプロンプトは使用しません。

このプロンプトは OpenClaw によって組み立てられ、各エージェント実行に注入されます。

プロバイダ Plugin は、OpenClaw が所有する完全なプロンプトを置き換えることなく、キャッシュを意識したプロンプトガイダンスを提供できます。プロバイダランタイムでは次のことが可能です。

- 名前付きの少数のコアセクション（`interaction_style`、
  `tool_call_style`、`execution_bias`）を置き換える
- プロンプトキャッシュ境界の上に **stable prefix** を注入する
- プロンプトキャッシュ境界の下に **dynamic suffix** を注入する

モデルファミリー固有のチューニングには、プロバイダ所有の貢献を使用してください。従来の
`before_prompt_build` によるプロンプト変更は、互換性のため、または本当にグローバルなプロンプト変更のために維持し、通常のプロバイダ動作には使わないでください。

## 構造

このプロンプトは意図的にコンパクトで、固定セクションを使用します。

- **Tooling**: structured-tool の単一の正しい情報源であることのリマインダーと、ランタイムのツール使用ガイダンス。
- **Safety**: 権力追求的な行動や監督の回避を避けるための短いガードレールのリマインダー。
- **Skills**（利用可能な場合）: 必要に応じて skill の指示を読み込む方法をモデルに伝えます。
- **OpenClaw Self-Update**: `config.schema.lookup` で安全に設定を確認する方法、`config.patch` で設定にパッチを適用する方法、`config.apply` で完全な設定を置き換える方法、そして明示的なユーザー要求がある場合にのみ `update.run` を実行する方法。owner-only の `gateway` ツールも、保護された exec パスに正規化される従来の `tools.bash.*` エイリアスを含め、`tools.exec.ask` / `tools.exec.security` の書き換えを拒否します。
- **Workspace**: 作業ディレクトリ（`agents.defaults.workspace`）。
- **Documentation**: OpenClaw ドキュメントへのローカルパス（リポジトリまたは npm パッケージ）と、それを読むべきタイミング。
- **Workspace Files (injected)**: ブートストラップファイルが以下に含まれていることを示します。
- **Sandbox**（有効な場合）: サンドボックス化されたランタイム、サンドボックスパス、昇格された exec が利用可能かどうかを示します。
- **Current Date & Time**: ユーザーのローカル時刻、タイムゾーン、時刻形式。
- **Reply Tags**: サポートされているプロバイダ向けの任意の返信タグ構文。
- **Heartbeats**: デフォルトエージェントで heartbeat が有効な場合の、heartbeat プロンプトと ack 動作。
- **Runtime**: ホスト、OS、node、モデル、リポジトリルート（検出された場合）、thinking level（1 行）。
- **Reasoning**: 現在の可視性レベルと `/reasoning` 切り替えのヒント。

Tooling セクションには、長時間実行される作業向けのランタイムガイダンスも含まれます。

- 将来のフォローアップ（`check back later`、リマインダー、定期作業）には `exec` の sleep ループ、`yieldMs` の遅延トリック、繰り返しの `process` ポーリングではなく cron を使う
- `exec` / `process` は、今すぐ開始してバックグラウンドで継続実行されるコマンドにのみ使う
- 自動完了 wake が有効な場合は、コマンドを一度だけ開始し、出力の発生または失敗時には push ベースの wake パスに依存する
- 実行中コマンドの確認が必要な場合は、ログ、ステータス、入力、または介入のために `process` を使う
- タスクがより大きい場合は、`sessions_spawn` を優先する。サブエージェントの完了は push ベースで、要求元に自動通知される
- 完了待ちだけのために `subagents list` / `sessions_list` をループでポーリングしない

実験的な `update_plan` ツールが有効な場合、Tooling では、これを自明ではない複数ステップ作業にのみ使用し、`in_progress` ステップを常にちょうど 1 つ維持し、更新のたびに計画全体を繰り返さないようモデルに指示します。

システムプロンプト内の Safety ガードレールは助言的なものです。これらはモデルの動作を導きますが、ポリシーを強制するものではありません。強制にはツールポリシー、exec 承認、サンドボックス化、チャネル allowlist を使用してください。オペレーターは設計上これらを無効化できます。

ネイティブの承認カード/ボタンを持つチャネルでは、ランタイムプロンプトはまずそのネイティブ承認 UI に依存するようエージェントに伝えるようになりました。手動の `/approve` コマンドを含めるのは、ツール結果がチャット承認を利用できないと示す場合、または手動承認が唯一の経路である場合だけにする必要があります。

## プロンプトモード

OpenClaw はサブエージェント向けにより小さなシステムプロンプトをレンダリングできます。ランタイムは各実行に `promptMode` を設定します（ユーザー向け設定ではありません）。

- `full`（デフォルト）: 上記のすべてのセクションを含みます。
- `minimal`: サブエージェント用に使われます。**Skills**、**Memory Recall**、**OpenClaw Self-Update**、**Model Aliases**、**User Identity**、**Reply Tags**、**Messaging**、**Silent Replies**、**Heartbeats** を省略します。Tooling、**Safety**、Workspace、Sandbox、Current Date & Time（既知の場合）、Runtime、および注入されたコンテキストは引き続き利用可能です。
- `none`: ベースの識別行のみを返します。

`promptMode=minimal` の場合、追加の注入プロンプトには **Group Chat Context** ではなく **Subagent Context** というラベルが付けられます。

## ワークスペースブートストラップの注入

ブートストラップファイルはトリミングされ、**Project Context** の下に追加されます。これにより、モデルは明示的な読み取りを必要とせずに、アイデンティティとプロファイルのコンテキストを把握できます。

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（新規ワークスペースのみ）
- `MEMORY.md` が存在する場合はそれ、存在しない場合は小文字のフォールバックとして `memory.md`

これらのファイルはすべて、**ファイル固有のゲートが適用される場合を除き**、毎ターン **コンテキストウィンドウに注入** されます。通常実行時の `HEARTBEAT.md` は、デフォルトエージェントで heartbeat が無効な場合、または
`agents.defaults.heartbeat.includeSystemPromptSection` が false の場合は省略されます。注入されるファイルは簡潔に保ってください。特に `MEMORY.md` は、時間とともに肥大化して、予想外にコンテキスト使用量が増えたり、Compaction がより頻繁に発生したりする原因になります。

> **注:** `memory/*.md` の日次ファイルは、通常のブートストラップ Project Context には **含まれません**。通常のターンでは、これらは `memory_search` と `memory_get` ツールを通じて必要に応じてアクセスされるため、モデルが明示的にそれらを読まない限り、コンテキストウィンドウを消費しません。例外は素の `/new` および `/reset` ターンです。この最初のターンでは、ランタイムが最近の日次メモリをワンショットの起動コンテキストブロックとして前置できる場合があります。

大きなファイルはマーカー付きで切り詰められます。ファイルごとの最大サイズは
`agents.defaults.bootstrapMaxChars`（デフォルト: 12000）で制御されます。ファイル全体で注入されるブートストラップコンテンツの合計は
`agents.defaults.bootstrapTotalMaxChars`（デフォルト: 60000）で上限が設定されます。ファイルが見つからない場合は、短い missing-file マーカーが注入されます。切り詰めが発生した場合、OpenClaw は Project Context に警告ブロックを注入できます。これは
`agents.defaults.bootstrapPromptTruncationWarning`（`off`、`once`、`always`；
デフォルト: `once`）で制御します。

サブエージェントセッションでは `AGENTS.md` と `TOOLS.md` のみが注入されます（サブエージェントのコンテキストを小さく保つため、他のブートストラップファイルは除外されます）。

内部 hook は `agent:bootstrap` を通じてこのステップに介入し、注入されるブートストラップファイルを変更または置き換えることができます（たとえば `SOUL.md` を別の persona に差し替えるなど）。

エージェントの話し方をより一般的でないものにしたい場合は、
[SOUL.md Personality Guide](/ja-JP/concepts/soul) から始めてください。

各注入ファイルの寄与量（生のサイズと注入後のサイズ、切り詰め、さらにツールスキーマのオーバーヘッド）を確認するには、`/context list` または `/context detail` を使用してください。[Context](/ja-JP/concepts/context) を参照してください。

## 時刻処理

システムプロンプトには、ユーザーのタイムゾーンが既知の場合、専用の **Current Date & Time** セクションが含まれます。プロンプトキャッシュを安定させるため、ここには現在 **タイムゾーン** のみが含まれます（動的な時計や時刻形式は含みません）。

エージェントが現在時刻を必要とする場合は `session_status` を使用してください。ステータスカードにはタイムスタンプ行が含まれます。同じツールでは、セッションごとのモデルオーバーライドを任意で設定することもできます（`model=default` でクリアされます）。

設定項目:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat`（`auto` | `12` | `24`）

完全な動作の詳細は [Date & Time](/ja-JP/date-time) を参照してください。

## Skills

対象となる Skills が存在する場合、OpenClaw は **利用可能な Skills 一覧**（`formatSkillsForPrompt`）を簡潔に注入し、各 skill の **ファイルパス** を含めます。プロンプトは、モデルに対して、一覧にある場所（workspace、managed、または bundled）から `read` を使って SKILL.md を読み込むよう指示します。対象となる skill がない場合、Skills セクションは省略されます。

対象判定には、skill メタデータのゲート、ランタイム環境/設定チェック、および `agents.defaults.skills` または
`agents.list[].skills` が設定されている場合の有効なエージェント skill allowlist が含まれます。

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

これにより、ベースプロンプトを小さく保ちながら、対象を絞った skill 利用を可能にします。

skills 一覧の予算は skills サブシステムが管理します。

- グローバルデフォルト: `skills.limits.maxSkillsPromptChars`
- エージェント単位のオーバーライド: `agents.list[].skillsLimits.maxSkillsPromptChars`

一般的な境界付きランタイム抜粋には別のサーフェスが使われます。

- `agents.defaults.contextLimits.*`
- `agents.list[].contextLimits.*`

この分離により、skills のサイズ管理を、`memory_get`、ライブツール結果、Compaction 後の AGENTS.md 再読み込みなどのランタイム読み取り/注入サイズ管理から切り離しています。

## Documentation

利用可能な場合、システムプロンプトには **Documentation** セクションが含まれ、ローカルの OpenClaw ドキュメントディレクトリ（リポジトリワークスペース内の `docs/` または同梱された npm パッケージの docs）を指し示します。また、公開ミラー、ソースリポジトリ、コミュニティ Discord、そして skill 発見のための ClawHub（[https://clawhub.ai](https://clawhub.ai)）についても記載します。プロンプトは、OpenClaw の動作、コマンド、設定、またはアーキテクチャについては、まずローカルドキュメントを参照するようモデルに指示し、可能であれば `openclaw status` を自分で実行し、アクセス権がない場合にのみユーザーに尋ねるようにします。
