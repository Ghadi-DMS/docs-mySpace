---
read_when:
    - エージェントのワークスペースまたはそのファイルレイアウトを説明する必要があります
    - エージェントのワークスペースをバックアップまたは移行したい場合があります
summary: 'エージェントのワークスペース: 保存場所、レイアウト、バックアップ戦略'
title: エージェントのワークスペース
x-i18n:
    generated_at: "2026-04-18T04:40:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: dd2e74614d8d45df04b1bbda48e2224e778b621803d774d38e4b544195eb234e
    source_path: concepts/agent-workspace.md
    workflow: 15
---

# エージェントのワークスペース

ワークスペースはエージェントのホームです。これは、ファイルツールとワークスペースコンテキストで使用される唯一の作業ディレクトリです。
これを非公開に保ち、メモリとして扱ってください。

これは、設定、認証情報、セッションを保存する `~/.openclaw/` とは別です。

**重要:** ワークスペースは**デフォルトの cwd**であり、厳格なサンドボックスではありません。ツールはワークスペースを基準に相対パスを解決しますが、サンドボックス化が有効でない限り、絶対パスは引き続きホスト上の別の場所へ到達できます。分離が必要な場合は、[`agents.defaults.sandbox`](/ja-JP/gateway/sandboxing)（および/またはエージェントごとのサンドボックス設定）を使用してください。サンドボックス化が有効で、`workspaceAccess` が `"rw"` ではない場合、ツールはホストのワークスペースではなく、`~/.openclaw/sandboxes` 配下のサンドボックスワークスペース内で動作します。

## デフォルトの保存場所

- デフォルト: `~/.openclaw/workspace`
- `OPENCLAW_PROFILE` が設定されていて `"default"` ではない場合、デフォルトは `~/.openclaw/workspace-<profile>` になります。
- `~/.openclaw/openclaw.json` で上書きします:

```json5
{
  agent: {
    workspace: "~/.openclaw/workspace",
  },
}
```

`openclaw onboard`、`openclaw configure`、または `openclaw setup` は、ワークスペースが存在しない場合にそれを作成し、ブートストラップファイルを初期投入します。
サンドボックスのシードコピーは、ワークスペース内の通常ファイルのみを受け入れます。ソースワークスペースの外部を指す symlink/hardlink のエイリアスは無視されます。

すでにワークスペースファイルを自分で管理している場合は、ブートストラップファイルの作成を無効にできます:

```json5
{ agent: { skipBootstrap: true } }
```

## 追加のワークスペースフォルダー

古いインストールでは `~/openclaw` が作成されていることがあります。複数のワークスペースディレクトリを残しておくと、同時に有効なのは1つのワークスペースだけなので、認証や状態の食い違いによって混乱を招くことがあります。

**推奨:** 有効なワークスペースは1つだけにしてください。追加フォルダーをもう使わない場合は、アーカイブするかゴミ箱へ移動してください（例: `trash ~/openclaw`）。意図的に複数のワークスペースを保持する場合は、`agents.defaults.workspace` が有効なものを指していることを確認してください。

`openclaw doctor` は、追加のワークスペースディレクトリを検出すると警告します。

## ワークスペースのファイルマップ（各ファイルの意味）

これらは、OpenClaw がワークスペース内にあることを想定している標準ファイルです:

- `AGENTS.md`
  - エージェント向けの運用指示と、メモリの使い方。
  - 毎セッションの開始時に読み込まれます。
  - ルール、優先順位、「どう振る舞うか」の詳細を書くのに適しています。

- `SOUL.md`
  - ペルソナ、トーン、境界。
  - 毎セッションで読み込まれます。
  - ガイド: [SOUL.md Personality Guide](/ja-JP/concepts/soul)

- `USER.md`
  - ユーザーが誰か、どう呼びかけるか。
  - 毎セッションで読み込まれます。

- `IDENTITY.md`
  - エージェントの名前、雰囲気、絵文字。
  - ブートストラップ儀式の間に作成または更新されます。

- `TOOLS.md`
  - ローカルツールと慣習に関するメモ。
  - ツールの利用可否を制御するものではなく、あくまでガイダンスです。

- `HEARTBEAT.md`
  - Heartbeat 実行用の任意の小さなチェックリスト。
  - トークン消費を避けるため、短く保ってください。

- `BOOT.md`
  - 内部フックが有効なとき、Gateway 再起動時に実行される任意の起動チェックリスト。
  - 短く保ち、外向き送信には message ツールを使用してください。

- `BOOTSTRAP.md`
  - 初回実行時だけの儀式。
  - 新規ワークスペースに対してのみ作成されます。
  - 儀式が完了したら削除してください。

- `memory/YYYY-MM-DD.md`
  - 日次メモリログ（1日1ファイル）。
  - セッション開始時に今日と昨日のファイルを読むことを推奨します。

- `MEMORY.md`（任意）
  - キュレーションされた長期メモリ。
  - メインの非公開セッションでのみ読み込んでください（共有/グループコンテキストではありません）。

ワークフローと自動メモリフラッシュについては [Memory](/ja-JP/concepts/memory) を参照してください。

- `skills/`（任意）
  - ワークスペース固有の Skills。
  - そのワークスペースにおける最優先の Skills 保存場所です。
  - 名前が衝突した場合、プロジェクトのエージェント Skills、個人用エージェント Skills、管理された Skills、同梱 Skills、および `skills.load.extraDirs` より優先されます。

- `canvas/`（任意）
  - Node 表示用の Canvas UI ファイル（例: `canvas/index.html`）。

ブートストラップファイルのいずれかが欠けている場合、OpenClaw は「missing file」マーカーをセッションに注入して継続します。大きなブートストラップファイルは注入時に切り詰められます。制限は `agents.defaults.bootstrapMaxChars`（デフォルト: 12000）と `agents.defaults.bootstrapTotalMaxChars`（デフォルト: 60000）で調整してください。
`openclaw setup` は、既存ファイルを上書きせずに不足しているデフォルトを再作成できます。

## ワークスペースに含まれないもの

以下は `~/.openclaw/` 配下にあり、ワークスペースのリポジトリにコミットすべきではありません:

- `~/.openclaw/openclaw.json`（設定）
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`（モデル認証プロファイル: OAuth + API keys）
- `~/.openclaw/credentials/`（チャネル/プロバイダーの状態と旧OAuthインポートデータ）
- `~/.openclaw/agents/<agentId>/sessions/`（セッショントランスクリプト + メタデータ）
- `~/.openclaw/skills/`（管理された Skills）

セッションや設定を移行する必要がある場合は、それらを別途コピーし、バージョン管理には含めないでください。

## Git バックアップ（推奨、非公開）

ワークスペースは非公開のメモリとして扱ってください。バックアップと復旧ができるよう、**非公開**の git リポジトリに置いてください。

これらの手順は Gateway が動作しているマシン上で実行してください（ワークスペースはそこにあります）。

### 1) リポジトリを初期化する

git がインストールされている場合、まったく新しいワークスペースは自動的に初期化されます。このワークスペースがまだリポジトリでない場合は、次を実行してください:

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
git commit -m "Add agent workspace"
```

### 2) 非公開リモートを追加する（初心者向けオプション）

オプションA: GitHub web UI

1. GitHub で新しい**非公開**リポジトリを作成します。
2. README で初期化しないでください（マージ競合を避けるため）。
3. HTTPS リモートURLをコピーします。
4. リモートを追加して push します:

```bash
git branch -M main
git remote add origin <https-url>
git push -u origin main
```

オプションB: GitHub CLI (`gh`)

```bash
gh auth login
gh repo create openclaw-workspace --private --source . --remote origin --push
```

オプションC: GitLab web UI

1. GitLab で新しい**非公開**リポジトリを作成します。
2. README で初期化しないでください（マージ競合を避けるため）。
3. HTTPS リモートURLをコピーします。
4. リモートを追加して push します:

```bash
git branch -M main
git remote add origin <https-url>
git push -u origin main
```

### 3) 継続的な更新

```bash
git status
git add .
git commit -m "Update memory"
git push
```

## シークレットをコミットしないでください

非公開リポジトリであっても、ワークスペースにシークレットを保存するのは避けてください:

- API keys、OAuth tokens、passwords、または private credentials。
- `~/.openclaw/` 配下のものすべて。
- チャットや機密添付ファイルの生ダンプ。

機密参照を保存しなければならない場合は、プレースホルダーを使い、本物のシークレットは別の場所に保管してください（パスワードマネージャー、環境変数、または `~/.openclaw/`）。

推奨される `.gitignore` の初期例:

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## ワークスペースを新しいマシンへ移す

1. 希望するパス（デフォルトは `~/.openclaw/workspace`）にリポジトリを clone します。
2. `~/.openclaw/openclaw.json` で `agents.defaults.workspace` をそのパスに設定します。
3. `openclaw setup --workspace <path>` を実行して、不足しているファイルを初期投入します。
4. セッションが必要な場合は、古いマシンから `~/.openclaw/agents/<agentId>/sessions/` を別途コピーしてください。

## 補足

- マルチエージェントのルーティングでは、エージェントごとに異なるワークスペースを使えます。ルーティング設定については [Channel routing](/ja-JP/channels/channel-routing) を参照してください。
- `agents.defaults.sandbox` が有効な場合、非メインセッションでは `agents.defaults.sandbox.workspaceRoot` 配下のセッションごとのサンドボックスワークスペースを使用できます。

## 関連

- [Standing Orders](/ja-JP/automation/standing-orders) — ワークスペースファイル内の永続的な指示
- [Heartbeat](/ja-JP/gateway/heartbeat) — HEARTBEAT.md ワークスペースファイル
- [Session](/ja-JP/concepts/session) — セッション保存パス
- [Sandboxing](/ja-JP/gateway/sandboxing) — サンドボックス環境におけるワークスペースアクセス
