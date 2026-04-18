---
read_when:
    - OpenClawにおける「コンテキスト」が何を意味するのかを理解したい
    - モデルがなぜ何かを「知っている」のか（または忘れたのか）をデバッグしている
    - コンテキストのオーバーヘッドを減らしたい（`/context`、`/status`、`/compact`）
summary: コンテキスト：モデルが何を見るか、どのように構築されるか、そしてそれをどのように調べるか
title: コンテキスト
x-i18n:
    generated_at: "2026-04-18T04:40:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 477ccb1d9654968d0e904b6846b32b8c14db6b6c0d3d2ec2b7409639175629f9
    source_path: concepts/context.md
    workflow: 15
---

# コンテキスト

「コンテキスト」とは、**OpenClawが実行のためにモデルへ送信するすべて**です。これは、モデルの**コンテキストウィンドウ**（トークン上限）によって制限されます。

初心者向けのイメージ：

- **システムプロンプト**（OpenClawが構築）: ルール、ツール、Skillsの一覧、時刻/ランタイム、注入されたワークスペースファイル。
- **会話履歴**: このセッションにおけるあなたのメッセージとアシスタントのメッセージ。
- **ツール呼び出し/結果 + 添付ファイル**: コマンド出力、ファイル読み取り、画像/音声など。

コンテキストは「メモリ」とは_同じものではありません_。メモリはディスクに保存して後で再読み込みできますが、コンテキストはモデルの現在のウィンドウ内に入っているものです。

## クイックスタート（コンテキストを確認する）

- `/status` → 「ウィンドウがどれくらい埋まっているか？」のクイック表示 + セッション設定。
- `/context list` → 何が注入されているか + おおよそのサイズ（ファイルごと + 合計）。
- `/context detail` → より詳細な内訳: ファイルごと、ツールスキーマごとのサイズ、skillエントリごとのサイズ、システムプロンプトのサイズ。
- `/usage tokens` → 通常の返信に、返信ごとの使用量フッターを追加。
- `/compact` → 古い履歴を要約してコンパクトなエントリにし、ウィンドウの空きを増やす。

関連項目: [スラッシュコマンド](/ja-JP/tools/slash-commands)、[トークン使用量とコスト](/ja-JP/reference/token-use)、[Compaction](/ja-JP/concepts/compaction)。

## 出力例

値は、モデル、プロバイダー、ツールポリシー、ワークスペース内の内容によって変わります。

### `/context list`

```
🧠 Context breakdown
Workspace: <workspaceDir>
Bootstrap max/file: 12,000 chars
Sandbox: mode=non-main sandboxed=false
System prompt (run): 38,412 chars (~9,603 tok) (Project Context 23,901 chars (~5,976 tok))

Injected workspace files:
- AGENTS.md: OK | raw 1,742 chars (~436 tok) | injected 1,742 chars (~436 tok)
- SOUL.md: OK | raw 912 chars (~228 tok) | injected 912 chars (~228 tok)
- TOOLS.md: TRUNCATED | raw 54,210 chars (~13,553 tok) | injected 20,962 chars (~5,241 tok)
- IDENTITY.md: OK | raw 211 chars (~53 tok) | injected 211 chars (~53 tok)
- USER.md: OK | raw 388 chars (~97 tok) | injected 388 chars (~97 tok)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 chars (~0 tok) | injected 0 chars (~0 tok)

Skills list (system prompt text): 2,184 chars (~546 tok) (12 skills)
Tools: read, edit, write, exec, process, browser, message, sessions_send, …
Tool list (system prompt text): 1,032 chars (~258 tok)
Tool schemas (JSON): 31,988 chars (~7,997 tok) (counts toward context; not shown as text)
Tools: (same as above)

Session tokens (cached): 14,250 total / ctx=32,000
```

### `/context detail`

```
🧠 Context breakdown (detailed)
…
Top skills (prompt entry size):
- frontend-design: 412 chars (~103 tok)
- oracle: 401 chars (~101 tok)
… (+10 more skills)

Top tools (schema size):
- browser: 9,812 chars (~2,453 tok)
- exec: 6,240 chars (~1,560 tok)
… (+N more tools)
```

## コンテキストウィンドウに含まれるもの

モデルが受け取るものはすべて含まれます。たとえば：

- システムプロンプト（全セクション）。
- 会話履歴。
- ツール呼び出し + ツール結果。
- 添付ファイル/文字起こし（画像/音声/ファイル）。
- Compactionの要約と刈り込みアーティファクト。
- プロバイダーの「ラッパー」や隠しヘッダー（見えなくてもカウントされる）。

## OpenClawがシステムプロンプトを構築する方法

システムプロンプトは**OpenClawが管理**しており、実行ごとに再構築されます。含まれる内容は次のとおりです：

- ツール一覧 + 短い説明。
- Skills一覧（メタデータのみ。詳細は後述）。
- ワークスペースの場所。
- 時刻（UTC + 設定されている場合は変換後のユーザー時刻）。
- ランタイムメタデータ（ホスト/OS/モデル/thinking）。
- **Project Context**配下に注入されたワークスペースのブートストラップファイル。

完全な内訳: [システムプロンプト](/ja-JP/concepts/system-prompt)。

## 注入されるワークスペースファイル（Project Context）

デフォルトでは、OpenClawは固定のワークスペースファイル群を注入します（存在する場合）：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（初回実行時のみ）

大きいファイルは、`agents.defaults.bootstrapMaxChars`（デフォルト `12000` 文字）を使ってファイルごとに切り詰められます。OpenClawはさらに、ファイル全体にまたがるブートストラップ注入の合計上限 `agents.defaults.bootstrapTotalMaxChars`（デフォルト `60000` 文字）も適用します。`/context` では、**raw と injected** のサイズ、および切り詰めが発生したかどうかが表示されます。

切り詰めが発生すると、ランタイムはProject Context配下にプロンプト内警告ブロックを注入できます。これは `agents.defaults.bootstrapPromptTruncationWarning`（`off`、`once`、`always`。デフォルトは `once`）で設定します。

## Skills: 注入されるものと必要時ロードされるもの

システムプロンプトには、コンパクトな**Skills一覧**（名前 + 説明 + 場所）が含まれます。この一覧には実際のオーバーヘッドがあります。

skillの指示内容自体は、デフォルトでは含まれません。モデルは、必要なときにだけそのskillの `SKILL.md` を `read` する想定です。

## ツール: コストは2種類ある

ツールは、2つの形でコンテキストに影響します：

1. システムプロンプト内の**ツール一覧テキスト**（「Tooling」として見えるもの）。
2. **ツールスキーマ**（JSON）。モデルがツールを呼び出せるように送信されます。プレーンテキストとしては見えなくても、コンテキストに含まれます。

`/context detail` では、どのツールスキーマが最も大きいかを分解して表示できるため、何が支配的かを確認できます。

## コマンド、ディレクティブ、「インラインショートカット」

スラッシュコマンドはGatewayによって処理されます。動作にはいくつかの種類があります：

- **スタンドアロンコマンド**: `/...` だけのメッセージは、コマンドとして実行されます。
- **ディレクティブ**: `/think`、`/verbose`、`/trace`、`/reasoning`、`/elevated`、`/model`、`/queue` は、モデルがメッセージを見る前に取り除かれます。
  - ディレクティブだけのメッセージは、セッション設定を保持します。
  - 通常メッセージ内のインラインディレクティブは、メッセージ単位のヒントとして動作します。
- **インラインショートカット**（許可リストに載った送信者のみ）: 通常メッセージ内の特定の `/...` トークンは即座に実行できます（例: 「hey /status」）。その後、残りのテキストがモデルに渡される前に取り除かれます。

詳細: [スラッシュコマンド](/ja-JP/tools/slash-commands)。

## セッション、Compaction、刈り込み（何が保持されるか）

メッセージをまたいで何が保持されるかは、仕組みによって異なります：

- **通常の履歴** は、ポリシーにより compact/prune されるまでセッショントランスクリプトに保持されます。
- **Compaction** は、要約をトランスクリプト内に保持し、最近のメッセージはそのまま維持します。
- **刈り込み** は、実行時の_メモリ内_プロンプトから古いツール結果を削除しますが、トランスクリプト自体は書き換えません。

ドキュメント: [Session](/ja-JP/concepts/session)、[Compaction](/ja-JP/concepts/compaction)、[Session pruning](/ja-JP/concepts/session-pruning)。

デフォルトでは、OpenClawは組み立てと compaction に組み込みの `legacy` コンテキストエンジンを使います。`kind: "context-engine"` を提供する plugin をインストールし、`plugins.slots.contextEngine` でそれを選ぶと、OpenClawは代わりにコンテキストの組み立て、`/compact`、および関連する subagent コンテキストライフサイクルフックをそのエンジンに委譲します。`ownsCompaction: false` であっても `legacy` エンジンへの自動フォールバックは行われません。アクティブなエンジンは依然として `compact()` を正しく実装している必要があります。完全なプラガブルインターフェース、ライフサイクルフック、設定については [Context Engine](/ja-JP/concepts/context-engine) を参照してください。

## `/context` が実際に報告するもの

`/context` は、可能であれば最新の**実行時に構築された**システムプロンプトレポートを優先します：

- `System prompt (run)` = 直近の埋め込み済み（ツール利用可能）実行から取得され、セッションストアに保存されたもの。
- `System prompt (estimate)` = 実行レポートが存在しない場合（またはそのレポートを生成しないCLIバックエンド経由で実行している場合）に、その場で計算されたもの。

どちらの場合でも、サイズと主な寄与要因を報告しますが、システムプロンプト全体やツールスキーマそのものは出力しません。

## 関連

- [Context Engine](/ja-JP/concepts/context-engine) — plugin によるカスタムコンテキスト注入
- [Compaction](/ja-JP/concepts/compaction) — 長い会話の要約
- [システムプロンプト](/ja-JP/concepts/system-prompt) — システムプロンプトの構築方法
- [Agent Loop](/ja-JP/concepts/agent-loop) — エージェント実行サイクル全体
