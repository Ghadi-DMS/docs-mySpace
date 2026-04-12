---
read_when:
    - qa-lab または qa-channel の拡張
    - リポジトリに裏付けられた QA シナリオの追加
    - Gateway ダッシュボードを中心とした、より現実性の高い QA 自動化の構築
summary: qa-lab、qa-channel、シード済みシナリオ、およびプロトコルレポート向けの非公開 QA 自動化の構成
title: QA E2E 自動化
x-i18n:
    generated_at: "2026-04-12T23:28:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: b9fe27dc049823d5e3eb7ae1eac6aad21ed9e917425611fb1dbcb28ab9210d5e
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA E2E 自動化

この非公開 QA スタックは、単一のユニットテストではできない、より現実的で、
チャネルに沿った形で OpenClaw を検証することを目的としています。

現在の構成要素:

- `extensions/qa-channel`: DM、channel、thread、
  reaction、edit、delete の各サーフェスを備えた合成メッセージチャネル。
- `extensions/qa-lab`: transcript の観察、
  受信メッセージの注入、Markdown レポートのエクスポートのためのデバッガ UI と QA バス。
- `qa/`: キックオフタスクおよびベースライン QA
  シナリオ向けの、リポジトリに裏付けられたシードアセット。

現在の QA オペレーターのフローは、2 ペインの QA サイトです:

- 左: エージェントを表示する Gateway ダッシュボード（Control UI）。
- 右: Slack 風の transcript と scenario plan を表示する QA Lab。

実行方法:

```bash
pnpm qa:lab:up
```

これにより QA サイトがビルドされ、Docker ベースの gateway レーンが起動し、
QA Lab ページが公開されます。ここでオペレーターまたは自動化ループがエージェントに QA
ミッションを与え、実際のチャネル動作を観察し、何が機能したか、何が失敗したか、
何がブロックされたままだったかを記録できます。

Docker イメージを毎回再ビルドせずに QA Lab UI をより高速に反復したい場合は、
バインドマウントされた QA Lab バンドルでスタックを起動します:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` は、Docker サービスをビルド済みイメージ上で維持し、
`extensions/qa-lab/web/dist` を `qa-lab` コンテナにバインドマウントします。
`qa:lab:watch` は変更時にそのバンドルを再ビルドし、
QA Lab のアセットハッシュが変わるとブラウザは自動リロードします。

transport-real な Matrix スモークレーンを実行するには、次を実行します:

```bash
pnpm openclaw qa matrix
```

このレーンは、Docker 内で使い捨ての Tuwunel homeserver をプロビジョニングし、
一時的な driver、SUT、observer ユーザーを登録し、1 つの private room を作成したうえで、
実際の Matrix Plugin を QA gateway child 内で実行します。ライブ transport レーンは、
child config をテスト対象の transport に限定して保持するため、
child config 内で `qa-channel` なしで Matrix が動作します。

transport-real な Telegram スモークレーンを実行するには、次を実行します:

```bash
pnpm openclaw qa telegram
```

このレーンは、使い捨てサーバーをプロビジョニングする代わりに、1 つの実在する
private Telegram group を対象にします。これには `OPENCLAW_QA_TELEGRAM_GROUP_ID`、
`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、
`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` が必要で、
同じ private group 内に 2 つの異なる bot が必要です。SUT bot は Telegram username を
持っている必要があり、両方の bot で `@BotFather` 内の Bot-to-Bot Communication Mode
が有効になっていると、bot 間観察が最もよく機能します。

ライブ transport レーンは現在、それぞれが独自の scenario list 形状を考案する代わりに、
1 つのより小さな contract を共有します:

`qa-channel` は依然として広範な合成 product-behavior スイートであり、
ライブ transport coverage matrix には含まれません。

| レーン   | Canary | Mention gating | Allowlist block | Top-level reply | Restart resume | Thread follow-up | Thread isolation | Reaction observation | Help command |
| -------- | ------ | -------------- | --------------- | --------------- | -------------- | ---------------- | ---------------- | -------------------- | ------------ |
| Matrix   | x      | x              | x               | x               | x              | x                | x                | x                    |              |
| Telegram | x      |                |                 |                 |                |                  |                  |                      | x            |

これにより、`qa-channel` は広範な product-behavior スイートとして維持される一方で、
Matrix、Telegram、および将来のライブ transport は、1 つの明示的な
transport-contract チェックリストを共有します。

Docker を QA パスに持ち込まずに使い捨て Linux VM レーンを実行するには、
次を実行します:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

これにより新しい Multipass guest が起動し、依存関係がインストールされ、
guest 内で OpenClaw がビルドされて `qa suite` が実行され、その後通常の QA レポートと
summary がホスト上の `.artifacts/qa-e2e/...` にコピーされます。
これは、ホスト上の `qa suite` と同じ scenario selection 動作を再利用します。
ホストおよび Multipass の suite 実行では、選択された複数のシナリオを、分離された
gateway worker によってデフォルトで並列実行します。最大 64 worker、
または選択された scenario 数のいずれか少ない方までです。
worker 数を調整するには `--concurrency <count>` を使用し、
直列実行には `--concurrency 1` を使用してください。
ライブ実行では、guest で現実的に扱えるサポート対象の QA auth 入力が転送されます:
env ベースの provider key、QA live provider config path、
および存在する場合の `CODEX_HOME` です。guest がマウントされた workspace を通じて
書き戻せるように、`--output-dir` はリポジトリルート配下に維持してください。

## リポジトリに裏付けられたシード

シードアセットは `qa/` にあります:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

これらは意図的に git に置かれており、QA plan が人間とエージェントの両方から
見えるようになっています。

`qa-lab` は汎用の markdown runner のままであるべきです。各 scenario markdown file は、
1 回のテスト実行に対する source of truth であり、次を定義する必要があります:

- scenario metadata
- docs および code ref
- 任意の Plugin 要件
- 任意の gateway config patch
- 実行可能な `qa-flow`

ベースライン一覧は、少なくとも次をカバーできる十分な広さを保つ必要があります:

- DM と channel chat
- thread の動作
- message action lifecycle
- Cron callback
- memory recall
- model switching
- subagent handoff
- repo 読み取りと docs 読み取り
- Lobster Invaders のような小さなビルドタスク 1 つ

## transport adapter

`qa-lab` は markdown QA シナリオ向けの汎用 transport seam を管理します。
`qa-channel` はその seam 上の最初の adapter ですが、設計上の目標はより広く、
将来の実チャネルまたは合成チャネルも、transport 固有の QA runner を追加するのではなく、
同じ suite runner に接続できるようにすることです。

アーキテクチャレベルでの分割は次のとおりです:

- `qa-lab` は汎用 scenario execution、worker concurrency、artifact writing、reporting を管理します。
- transport adapter は gateway config、readiness、受信および送信の観察、transport action、正規化された transport state を管理します。
- `qa/scenarios/` 配下の markdown scenario file がテスト実行を定義し、それを実行する再利用可能な runtime surface は `qa-lab` が提供します。

新しいチャネル adapter 向けの maintainer 向け導入ガイダンスは
[Testing](/ja-JP/help/testing#adding-a-channel-to-qa) にあります。

## レポート

`qa-lab` は、観察された bus timeline から Markdown protocol report をエクスポートします。
このレポートは次に答えるべきです:

- 何が機能したか
- 何が失敗したか
- 何がブロックされたままだったか
- どの follow-up scenario を追加する価値があるか

キャラクターとスタイルのチェックについては、同じ scenario を複数の live model
ref に対して実行し、判定付きの Markdown レポートを書き出します:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

このコマンドは Docker ではなく、ローカルの QA gateway child process を実行します。
character eval シナリオでは、`SOUL.md` を通じて persona を設定し、
その後 chat、workspace help、小さな file task などの通常の user turn を実行する必要があります。
候補 model には、それが評価中であることを伝えてはいけません。
このコマンドは各完全 transcript を保持し、基本的な run stats を記録したうえで、
judge model に fast mode と `xhigh` reasoning で、
自然さ、雰囲気、ユーモアに基づいて run を順位付けするよう求めます。
provider を比較する際は `--blind-judge-models` を使用してください:
judge prompt は依然としてすべての transcript と run status を受け取りますが、
candidate ref は `candidate-01` のような中立ラベルに置き換えられます。
レポートはパース後に順位を実際の ref に再マッピングします。
candidate run の thinking はデフォルトで `high` であり、
それをサポートする OpenAI model では `xhigh` になります。
特定の candidate をインラインで上書きするには、
`--model provider/model,thinking=<level>` を使ってください。
`--thinking <level>` は引き続きグローバルなフォールバックを設定し、
古い `--model-thinking <provider/model=level>` 形式も互換性のために維持されています。
OpenAI candidate ref は、provider がサポートする場合に priority processing を使うため、
デフォルトで fast mode です。単一の candidate または judge に上書きが必要な場合は、
`,fast`、`,no-fast`、または `,fast=false` をインラインで追加してください。
すべての candidate model に対して fast mode を強制的に有効にしたい場合のみ、
`--fast` を渡してください。candidate と judge の duration はベンチマーク分析のために
レポートに記録されますが、judge prompt では明示的に速度で順位付けしないよう指示します。
candidate と judge の model run はどちらもデフォルトで concurrency 16 です。
provider 制限またはローカル gateway 負荷によって実行がノイジーになりすぎる場合は、
`--concurrency` または `--judge-concurrency` を下げてください。
candidate の `--model` が渡されない場合、character eval のデフォルトは
`openai/gpt-5.4`、`openai/gpt-5.2`、`openai/gpt-5`、`anthropic/claude-opus-4-6`、
`anthropic/claude-sonnet-4-6`、`zai/glm-5.1`、
`moonshot/kimi-k2.5`、`google/gemini-3.1-pro-preview` です。
judge の `--judge-model` が渡されない場合、judge のデフォルトは
`openai/gpt-5.4,thinking=xhigh,fast` と
`anthropic/claude-opus-4-6,thinking=high` です。

## 関連ドキュメント

- [Testing](/ja-JP/help/testing)
- [QA Channel](/ja-JP/channels/qa-channel)
- [Dashboard](/web/dashboard)
