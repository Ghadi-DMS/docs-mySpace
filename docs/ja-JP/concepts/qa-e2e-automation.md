---
read_when:
    - qa-labまたはqa-channelの拡張
    - リポジトリ連動のQAシナリオの追加
    - Gatewayダッシュボードを中心に、より現実に近いQA自動化を構築する
summary: qa-lab、qa-channel、シード済みシナリオ、プロトコルレポート向けの非公開QA自動化の構成
title: QA E2E自動化
x-i18n:
    generated_at: "2026-04-18T04:40:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: adf8c5f74e8fabdc8e9fd7ecd41afce8b60354c7dd24d92ac926d3c527927cd4
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA E2E自動化

非公開のQAスタックは、単一のユニットテストよりも現実的で、
チャネルに近い形でOpenClawを検証することを目的としています。

現在の構成要素:

- `extensions/qa-channel`: DM、チャネル、スレッド、
  リアクション、編集、削除の各操作面を備えた合成メッセージチャネル。
- `extensions/qa-lab`: トランスクリプトの観測、
  受信メッセージの注入、Markdownレポートのエクスポートを行うための
  デバッガーUIとQAバス。
- `qa/`: キックオフタスクとベースラインQA
  シナリオ向けの、リポジトリ連動のシードアセット。

現在のQAオペレーターのフローは、2ペイン構成のQAサイトです:

- 左: エージェントを表示するGatewayダッシュボード（Control UI）。
- 右: Slack風のトランスクリプトとシナリオプランを表示するQA Lab。

次のコマンドで実行します:

```bash
pnpm qa:lab:up
```

これによりQAサイトがビルドされ、Dockerベースのgatewayレーンが起動し、
オペレーターまたは自動化ループがエージェントにQA
ミッションを与え、実際のチャネル動作を観察し、何が機能したか、
何が失敗したか、何がブロックされたままだったかを記録できる
QA Labページが公開されます。

Dockerイメージを毎回再ビルドせずにQA Lab UIをより高速に反復開発するには、
バインドマウントされたQA Labバンドルでスタックを起動します:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` は事前ビルド済みイメージ上でDockerサービスを維持し、
`extensions/qa-lab/web/dist` を `qa-lab` コンテナにバインドマウントします。`qa:lab:watch`
は変更時にそのバンドルを再ビルドし、QA Labアセットのハッシュが変わると
ブラウザは自動的にリロードされます。

実際のトランスポートを使うMatrixスモークレーンを実行するには、次を使います:

```bash
pnpm openclaw qa matrix
```

このレーンは、使い捨てのTuwunel homeserverをDockerで用意し、
一時的なdriver、SUT、observerユーザーを登録し、1つのプライベートルームを作成してから、
実際のMatrix pluginをQA gateway child内で実行します。ライブトランスポートレーンは、
child設定をテスト対象のトランスポートに限定して保つため、
child設定内では `qa-channel` なしでMatrixが動作します。構造化レポートアーティファクトと、
stdout/stderrをまとめたログを、選択したMatrix QA出力ディレクトリに書き出します。
外側の `scripts/run-node.mjs` のビルド/ランチャー出力も取得するには、
`OPENCLAW_RUN_NODE_OUTPUT_LOG=<path>` をリポジトリ内のログファイルに設定します。

実際のトランスポートを使うTelegramスモークレーンを実行するには、次を使います:

```bash
pnpm openclaw qa telegram
```

このレーンは、使い捨てサーバーを用意する代わりに、
実際の1つのプライベートTelegramグループを対象にします。必要なのは
`OPENCLAW_QA_TELEGRAM_GROUP_ID`、
`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、
`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` と、
同じプライベートグループ内にいる2つの異なるボットです。
SUTボットにはTelegramユーザー名が必要で、
ボット同士の観測は、両方のボットで
`@BotFather` の Bot-to-Bot Communication Mode が有効になっていると
最も安定して動作します。

ライブトランスポートレーンは現在、それぞれが独自のシナリオ一覧形式を発明する代わりに、
より小さな1つの共通コントラクトを共有しています:

`qa-channel` は依然として広範な合成プロダクト動作スイートであり、
ライブトランスポートのカバレッジマトリクスには含まれません。

| レーン   | Canary | メンションゲーティング | 許可リストブロック | トップレベル返信 | 再起動後の再開 | スレッドフォローアップ | スレッド分離 | リアクション観測 | ヘルプコマンド |
| -------- | ------ | ---------------------- | ------------------ | ---------------- | -------------- | ---------------------- | ------------ | ---------------- | -------------- |
| Matrix   | x      | x                      | x                  | x                | x              | x                      | x            | x                |                |
| Telegram | x      |                        |                    |                  |                |                        |              |                  | x              |

これにより、`qa-channel` は広範なプロダクト動作スイートとして維持されつつ、
Matrix、Telegram、将来のライブトランスポートが、
明示的な1つのトランスポートコントラクトのチェックリストを共有できます。

QAパスにDockerを持ち込まずに、使い捨てのLinux VMレーンを実行するには、
次を使います:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

これにより新しいMultipass guestが起動し、依存関係をインストールし、
guest内でOpenClawをビルドし、`qa suite` を実行してから、
通常のQAレポートとサマリーをホスト上の `.artifacts/qa-e2e/...` にコピーして戻します。
ホスト上の `qa suite` と同じシナリオ選択動作を再利用します。
ホストとMultipassのsuite実行は、デフォルトで分離されたgateway workerを使って、
選択された複数のシナリオを並列実行し、
最大64 workerまたは選択シナリオ数のいずれか少ない方まで動作します。
worker数を調整するには `--concurrency <count>` を使い、
直列実行するには `--concurrency 1` を使います。
ライブ実行では、guestに対して実用的なサポート済みQA認証入力が転送されます:
envベースのprovider key、QA live provider設定パス、
および存在する場合は `CODEX_HOME` です。guestが
マウントされたワークスペース経由で書き戻せるように、
`--output-dir` はリポジトリルート配下に保ってください。

## リポジトリ連動のシード

シードアセットは `qa/` にあります:

- `qa/scenarios/index.md`
- `qa/scenarios/<theme>/*.md`

これらは意図的にgitに置かれており、QAプランが人間にも
エージェントにも見えるようになっています。

`qa-lab` は汎用的なmarkdownランナーとして維持するべきです。各シナリオmarkdownファイルは
1回のテスト実行における信頼できる唯一の情報源であり、次を定義する必要があります:

- シナリオメタデータ
- 任意のカテゴリ、ケイパビリティ、レーン、リスクのメタデータ
- docsとcodeの参照
- 任意のplugin要件
- 任意のgateway設定パッチ
- 実行可能な `qa-flow`

`qa-flow` を支える再利用可能なランタイム面は、
汎用的かつ横断的なままで構いません。たとえば、markdownシナリオは、
埋め込みControl UIをGatewayの `browser.request` seam経由で駆動する
ブラウザ側ヘルパーと、トランスポート側ヘルパーを組み合わせてもよく、
そのために特別扱いのランナーを追加する必要はありません。

シナリオファイルは、ソースツリーのフォルダではなく、
プロダクト機能ごとにグループ化するべきです。ファイルを移動しても
シナリオIDは安定したままにし、実装トレーサビリティには
`docsRefs` と `codeRefs` を使ってください。

ベースライン一覧は、次をカバーできる程度に広く保つべきです:

- DMとチャネルチャット
- スレッド動作
- メッセージアクションのライフサイクル
- Cronコールバック
- メモリの想起
- モデル切り替え
- subagent handoff
- リポジトリ読み取りとdocs読み取り
- Lobster Invadersのような小さなビルドタスク1件

## Providerモックレーン

`qa suite` には2つのローカルproviderモックレーンがあります:

- `mock-openai` はシナリオ認識型のOpenClawモックです。
  リポジトリ連動QAとパリティゲート向けの、
  デフォルトの決定的モックレーンのままです。
- `aimock` は実験的なプロトコル、
  フィクスチャ、record/replay、chaosカバレッジ向けに
  AIMockベースのproviderサーバーを起動します。これは追加的なものであり、
  `mock-openai` シナリオディスパッチャーを置き換えるものではありません。

Providerレーン実装は `extensions/qa-lab/src/providers/` 配下にあります。
各providerは、自身のデフォルト値、ローカルサーバー起動、
gateway model設定、auth-profileステージング要件、
およびlive/mockケイパビリティフラグを管理します。
共有suiteコードとgatewayコードは、provider名で分岐するのではなく、
provider registryを経由してルーティングするべきです。

## トランスポートアダプター

`qa-lab` はmarkdown QAシナリオ向けの汎用トランスポートseamを所有します。
`qa-channel` はそのseam上の最初のアダプターですが、
設計目標はより広いものです:
将来の実チャネルまたは合成チャネルは、
トランスポート固有のQAランナーを追加するのではなく、
同じsuite runnerに接続できるようにするべきです。

アーキテクチャレベルでの分担は次のとおりです:

- `qa-lab` は汎用シナリオ実行、worker並列性、アーティファクト書き出し、レポートを所有します。
- トランスポートアダプターはgateway設定、readiness、受信および送信の観測、トランスポートアクション、正規化されたトランスポート状態を所有します。
- `qa/scenarios/` 配下のmarkdownシナリオファイルがテスト実行を定義し、それを実行する再利用可能なランタイム面は `qa-lab` が提供します。

新しいチャネルアダプター向けのメンテナー向け導入ガイダンスは
[Testing](/ja-JP/help/testing#adding-a-channel-to-qa) にあります。

## レポート

`qa-lab` は、観測されたバスタイムラインからMarkdownのプロトコルレポートを
エクスポートします。レポートは次に答えるべきです:

- 何が機能したか
- 何が失敗したか
- 何がブロックされたままだったか
- どのフォローアップシナリオを追加する価値があるか

文字やスタイルのチェックについては、同じシナリオを複数のライブモデル参照で実行し、
判定付きMarkdownレポートを書き出します:

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

このコマンドはDockerではなく、ローカルのQA gateway child processを実行します。
character evalシナリオでは `SOUL.md` を通じてペルソナを設定し、
その後チャット、ワークスペース支援、小さなファイルタスクのような
通常のユーザーターンを実行するべきです。
候補モデルには、それが評価対象であることを伝えてはいけません。
このコマンドは各完全トランスクリプトを保持し、基本的な実行統計を記録したうえで、
judge modelにfast modeと `xhigh` reasoningで、
自然さ、雰囲気、ユーモアに基づいて実行結果を順位付けさせます。
providerを比較する際は `--blind-judge-models` を使ってください:
judgeプロンプトには依然としてすべてのトランスクリプトと実行ステータスが渡されますが、
候補参照は `candidate-01` のような中立ラベルに置き換えられます。
レポートは、解析後に順位を実際の参照へ対応付けて戻します。
候補実行はデフォルトで `high` thinking、
それをサポートするOpenAIモデルでは `xhigh` になります。
特定の候補を上書きするには
`--model provider/model,thinking=<level>` をインラインで使います。
`--thinking <level>` は引き続きグローバルなフォールバックを設定し、
古い `--model-thinking <provider/model=level>` 形式も
互換性のため保持されています。
OpenAI候補参照はデフォルトでfast modeとなるため、
providerが対応している場合はpriority processingが使われます。
単一の候補またはjudgeに上書きが必要な場合は、
`,fast`、`,no-fast`、または `,fast=false` をインラインで追加してください。
すべての候補モデルに対してfast modeを強制的に有効にしたい場合にのみ
`--fast` を渡します。候補とjudgeの所要時間は
ベンチマーク分析のためレポートに記録されますが、
judgeプロンプトでは速度で順位付けしないよう明示されています。
候補実行とjudge model実行はどちらもデフォルトで並列数16です。
providerの制限やローカルgatewayの負荷で実行が不安定になる場合は、
`--concurrency` または `--judge-concurrency` を下げてください。
候補 `--model` が指定されない場合、character evalのデフォルトは
`openai/gpt-5.4`、`openai/gpt-5.2`、`openai/gpt-5`、`anthropic/claude-opus-4-6`、
`anthropic/claude-sonnet-4-6`、`zai/glm-5.1`、
`moonshot/kimi-k2.5`、
`google/gemini-3.1-pro-preview` になります。
judge `--judge-model` が指定されない場合、judgeのデフォルトは
`openai/gpt-5.4,thinking=xhigh,fast` と
`anthropic/claude-opus-4-6,thinking=high` です。

## 関連ドキュメント

- [Testing](/ja-JP/help/testing)
- [QA Channel](/ja-JP/channels/qa-channel)
- [Dashboard](/web/dashboard)
