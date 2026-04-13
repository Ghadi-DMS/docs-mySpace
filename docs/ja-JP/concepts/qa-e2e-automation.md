---
read_when:
    - qa-labまたはqa-channelの拡張
    - リポジトリ連動のQAシナリオの追加
    - Gatewayダッシュボードを中心に、より現実に近いQA自動化を構築すること
summary: qa-lab、qa-channel、シード済みシナリオ、およびプロトコルレポート向けの非公開QA自動化の構成
title: QA E2E自動化
x-i18n:
    generated_at: "2026-04-13T04:46:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: a4a4f5c765163565c95c2a071f201775fd9d8d60cad4ff25d71e4710559c1570
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA E2E自動化

非公開QAスタックは、単一のユニットテストよりも、より現実的でチャネルに即した形でOpenClawを検証することを目的としています。

現在の構成要素:

- `extensions/qa-channel`: DM、チャネル、スレッド、リアクション、編集、削除の各サーフェスを備えた合成メッセージチャネル。
- `extensions/qa-lab`: トランスクリプトの観察、受信メッセージの注入、Markdownレポートのエクスポートを行うためのデバッガーUIとQAバス。
- `qa/`: キックオフタスクとベースラインQAシナリオ用の、リポジトリ連動のシードアセット。

現在のQAオペレーターのフローは、2ペインのQAサイトです:

- 左: エージェントを表示するGatewayダッシュボード（Control UI）。
- 右: Slack風のトランスクリプトとシナリオプランを表示するQA Lab。

次のコマンドで実行します:

```bash
pnpm qa:lab:up
```

これによりQAサイトがビルドされ、DockerベースのGatewayレーンが起動し、オペレーターまたは自動化ループがエージェントにQAミッションを与え、実際のチャネルの動作を観察し、何が機能したか、何が失敗したか、何がブロックされたままだったかを記録できるQA Labページが公開されます。

Dockerイメージを毎回再ビルドせずにQA Lab UIをより高速に反復したい場合は、バインドマウントされたQA Labバンドルでスタックを起動します:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` は、事前ビルド済みイメージ上でDockerサービスを維持しつつ、`extensions/qa-lab/web/dist` を `qa-lab` コンテナにバインドマウントします。`qa:lab:watch` は変更時にそのバンドルを再ビルドし、QA Labアセットのハッシュが変わるとブラウザが自動リロードします。

トランスポート実動のMatrixスモークレーンを実行するには、次を実行します:

```bash
pnpm openclaw qa matrix
```

このレーンでは、Docker内に使い捨てのTuwunelホームサーバーをプロビジョニングし、一時的なドライバー、SUT、オブザーバーユーザーを登録し、1つのプライベートルームを作成したうえで、QA Gateway子プロセス内で実際のMatrix Pluginを実行します。ライブトランスポートレーンでは、子プロセスの設定をテスト対象のトランスポートに限定するため、Matrixは子プロセス設定内で `qa-channel` なしに実行されます。

トランスポート実動のTelegramスモークレーンを実行するには、次を実行します:

```bash
pnpm openclaw qa telegram
```

このレーンは、使い捨てサーバーをプロビジョニングする代わりに、1つの実在するプライベートTelegramグループを対象にします。必要なのは `OPENCLAW_QA_TELEGRAM_GROUP_ID`、`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`、および同じプライベートグループ内に存在する2つの異なるボットです。SUTボットにはTelegramユーザー名が必要で、ボット同士の観察は、両方のボットで `@BotFather` のBot-to-Bot Communication Modeを有効にしていると最も安定します。

ライブトランスポートレーンは現在、それぞれが独自のシナリオリスト形状を考案するのではなく、より小さい1つの共通コントラクトを共有しています。

`qa-channel` は引き続き広範な合成プロダクト動作スイートであり、ライブトランスポートのカバレッジマトリクスには含まれません。

| レーン | カナリア | メンションゲーティング | Allowlistブロック | トップレベル返信 | 再起動後の再開 | スレッドのフォローアップ | スレッド分離 | リアクション観察 | helpコマンド |
| -------- | ------ | -------------- | --------------- | --------------- | -------------- | ---------------- | ---------------- | -------------------- | ------------ |
| Matrix   | x      | x              | x               | x               | x              | x                | x                | x                    |              |
| Telegram | x      |                |                 |                 |                |                  |                  |                      | x            |

これにより、`qa-channel` は広範なプロダクト動作スイートとして維持されつつ、Matrix、Telegram、および将来のライブトランスポートは、明示的なトランスポートコントラクトのチェックリストを共有できます。

DockerをQAパスに持ち込まずに、使い捨てのLinux VMレーンを実行するには、次を実行します:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

これにより新しいMultipassゲストが起動し、依存関係がインストールされ、ゲスト内でOpenClawがビルドされ、`qa suite` が実行された後、通常のQAレポートとサマリーがホスト上の `.artifacts/qa-e2e/...` にコピーされます。
シナリオ選択の挙動は、ホスト上の `qa suite` と同じものが再利用されます。
ホストおよびMultipassでのスイート実行は、デフォルトで分離されたGatewayワーカーを使って、選択された複数のシナリオを並列実行します。上限は64ワーカーまたは選択されたシナリオ数です。ワーカー数を調整するには `--concurrency <count>` を使用し、直列実行するには `--concurrency 1` を使用します。
ライブ実行では、ゲストで実用的なサポート済みQA認証入力、つまり環境変数ベースのプロバイダーキー、QAライブプロバイダー設定パス、および存在する場合は `CODEX_HOME` が転送されます。ゲストがマウントされたワークスペース経由で書き戻せるように、`--output-dir` はリポジトリルート配下に維持してください。

## リポジトリ連動のシード

シードアセットは `qa/` にあります:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

これらは、QAプランが人間にもエージェントにも見えるように、意図的にgitに置かれています。

`qa-lab` は汎用的なMarkdownランナーであり続けるべきです。各シナリオMarkdownファイルは1回のテスト実行に対する信頼できる唯一の情報源であり、次を定義する必要があります:

- シナリオメタデータ
- ドキュメントおよびコード参照
- 任意のPlugin要件
- 任意のGateway設定パッチ
- 実行可能な `qa-flow`

`qa-flow` を支える再利用可能なランタイムサーフェスは、汎用的で横断的なままで構いません。たとえばMarkdownシナリオでは、特別扱いのランナーを追加せずに、トランスポート側ヘルパーと、Gatewayの `browser.request` シームを通じて埋め込みControl UIを駆動するブラウザ側ヘルパーを組み合わせることができます。

ベースラインの一覧は、少なくとも次をカバーできるだけの広さを維持する必要があります:

- DMとチャネルチャット
- スレッド動作
- メッセージアクションのライフサイクル
- Cronコールバック
- メモリの想起
- モデル切り替え
- サブエージェントへのハンドオフ
- リポジトリ読み取りとドキュメント読み取り
- Lobster Invadersのような小規模なビルドタスク1件

## トランスポートアダプター

`qa-lab` は、Markdown QAシナリオ向けの汎用トランスポートシームを管理します。
`qa-channel` はそのシーム上の最初のアダプターですが、設計上の目標はより広く、将来の実チャネルまたは合成チャネルも、トランスポート専用のQAランナーを追加するのではなく、同じスイートランナーに接続されるべきです。

アーキテクチャレベルでは、分担は次のとおりです:

- `qa-lab` は、汎用シナリオ実行、ワーカー並列性、アーティファクト書き込み、およびレポートを管理します。
- トランスポートアダプターは、Gateway設定、準備完了、受信および送信の観察、トランスポート操作、および正規化されたトランスポート状態を管理します。
- `qa/scenarios/` 配下のMarkdownシナリオファイルがテスト実行を定義し、それを実行する再利用可能なランタイムサーフェスを `qa-lab` が提供します。

新しいチャネルアダプター向けのメンテナー向け導入ガイダンスは、[Testing](/ja-JP/help/testing#adding-a-channel-to-qa) にあります。

## レポート

`qa-lab` は、観測されたバスタイムラインからMarkdown形式のプロトコルレポートをエクスポートします。
このレポートは、次の問いに答えるべきです:

- 何が機能したか
- 何が失敗したか
- 何がブロックされたままだったか
- どのフォローアップシナリオを追加する価値があるか

キャラクターおよびスタイルのチェックでは、同じシナリオを複数のライブモデル参照で実行し、評価付きのMarkdownレポートを書き出します:

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

このコマンドはDockerではなく、ローカルのQA Gateway子プロセスを実行します。キャラクター評価シナリオでは、`SOUL.md` を通じてペルソナを設定し、その後チャット、ワークスペースヘルプ、小さなファイルタスクなどの通常のユーザーターンを実行する必要があります。候補モデルには、それが評価されていることを伝えてはいけません。このコマンドは各完全トランスクリプトを保持し、基本的な実行統計を記録したうえで、`xhigh` 推論を使う高速モードの判定モデルに対して、自然さ、雰囲気、ユーモアの観点で実行結果を順位付けするよう依頼します。
プロバイダーを比較する場合は `--blind-judge-models` を使用してください。判定プロンプトには引き続きすべてのトランスクリプトと実行状態が渡されますが、候補参照は `candidate-01` のような中立ラベルに置き換えられます。レポートでは、解析後に順位を実際の参照へと再対応付けします。
候補実行の思考レベルはデフォルトで `high` であり、OpenAIモデルで対応している場合は `xhigh` になります。特定の候補を上書きするには、`--model provider/model,thinking=<level>` をインラインで指定します。`--thinking <level>` は引き続きグローバルなフォールバックを設定し、旧来の `--model-thinking <provider/model=level>` 形式も互換性のため維持されています。
OpenAI候補参照はデフォルトで高速モードになっており、プロバイダーが対応している場合は優先処理が使われます。単一の候補または判定モデルで上書きが必要な場合は、`,fast`、`,no-fast`、または `,fast=false` をインラインで追加してください。すべての候補モデルで高速モードを強制的に有効にしたい場合にのみ `--fast` を渡してください。候補と判定モデルの実行時間はベンチマーク分析のためレポートに記録されますが、判定プロンプトでは速度で順位付けしないよう明示されています。
候補および判定モデルの実行はいずれもデフォルトで同時実行数16です。プロバイダー制限やローカルGateway負荷によって実行のノイズが大きすぎる場合は、`--concurrency` または `--judge-concurrency` を下げてください。
候補の `--model` が渡されない場合、キャラクター評価はデフォルトで `openai/gpt-5.4`、`openai/gpt-5.2`、`openai/gpt-5`、`anthropic/claude-opus-4-6`、`anthropic/claude-sonnet-4-6`、`zai/glm-5.1`、`moonshot/kimi-k2.5`、および `google/gemini-3.1-pro-preview` を使用します。
判定用の `--judge-model` が渡されない場合、判定モデルのデフォルトは `openai/gpt-5.4,thinking=xhigh,fast` と `anthropic/claude-opus-4-6,thinking=high` です。

## 関連ドキュメント

- [Testing](/ja-JP/help/testing)
- [QA Channel](/ja-JP/channels/qa-channel)
- [Dashboard](/web/dashboard)
