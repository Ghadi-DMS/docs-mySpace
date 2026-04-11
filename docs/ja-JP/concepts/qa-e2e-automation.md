---
read_when:
    - qa-labまたはqa-channelの拡張
    - リポジトリ連動のQAシナリオの追加
    - Gatewayダッシュボードを中心とした、より現実に近いQA自動化の構築
summary: qa-lab、qa-channel、シード済みシナリオ、プロトコルレポート向けの非公開QA自動化の構成
title: QA E2E自動化
x-i18n:
    generated_at: "2026-04-11T02:44:18Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5427b505e26bfd542e984e3920c3f7cb825473959195ba9737eff5da944c60d0
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# QA E2E自動化

非公開のQAスタックは、単一のユニットテストよりも現実のチャネルに近い形でOpenClawを検証することを目的としています。

現在の構成要素:

- `extensions/qa-channel`: DM、チャネル、スレッド、リアクション、編集、削除の各サーフェスを備えた合成メッセージチャネル。
- `extensions/qa-lab`: トランスクリプトの観察、受信メッセージの注入、Markdownレポートのエクスポートを行うためのデバッガUIとQAバス。
- `qa/`: キックオフタスクとベースラインQAシナリオ用の、リポジトリ連動のシード資産。

現在のQAオペレーターのフローは、2ペインのQAサイトです:

- 左: エージェントを表示するGatewayダッシュボード（Control UI）。
- 右: Slack風のトランスクリプトとシナリオ計画を表示するQA Lab。

次のコマンドで実行します:

```bash
pnpm qa:lab:up
```

これによりQAサイトがビルドされ、DockerベースのGatewayレーンが起動し、オペレーターまたは自動化ループがエージェントにQAミッションを与え、実際のチャネルの挙動を観察し、何が機能し、何が失敗し、何がブロックされたままかを記録できるQA Labページが公開されます。

Dockerイメージを毎回再ビルドせずにQA Lab UIをより高速に反復したい場合は、QA Labバンドルをバインドマウントした状態でスタックを起動します:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` は、Dockerサービスを事前ビルド済みイメージ上で維持しつつ、`extensions/qa-lab/web/dist` を `qa-lab` コンテナにバインドマウントします。`qa:lab:watch` は変更時にそのバンドルを再ビルドし、QA Labのアセットハッシュが変わるとブラウザが自動再読み込みされます。

実際のトランスポートを使うMatrixスモークレーンを実行するには、次を実行します:

```bash
pnpm openclaw qa matrix
```

このレーンはDocker内に使い捨てのTuwunelホームサーバーをプロビジョニングし、一時的なドライバー、SUT、オブザーバーユーザーを登録し、1つのプライベートルームを作成したうえで、実際のMatrixプラグインをQA Gateway子プロセス内で実行します。ライブトランスポートレーンでは、子設定をテスト対象トランスポートに限定するため、Matrixは子設定内で `qa-channel` なしで実行されます。

実際のトランスポートを使うTelegramスモークレーンを実行するには、次を実行します:

```bash
pnpm openclaw qa telegram
```

このレーンは使い捨てサーバーをプロビジョニングする代わりに、実在する1つのプライベートTelegramグループを対象にします。`OPENCLAW_QA_TELEGRAM_GROUP_ID`、`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`、`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` が必要で、同じプライベートグループ内に2つの異なるボットが必要です。SUTボットにはTelegramユーザー名が必要で、ボット同士の観察は、両方のボットで `@BotFather` のBot-to-Bot Communication Modeが有効になっていると最もうまく機能します。

ライブトランスポートレーンは、それぞれが独自のシナリオリスト形状を考案する代わりに、より小さな1つの共通契約を共有するようになりました:

`qa-channel` は引き続き広範な合成プロダクト挙動スイートであり、ライブトランスポートのカバレッジマトリクスには含まれません。

| レーン | Canary | メンションのゲーティング | 許可リストのブロック | トップレベル返信 | 再起動後の再開 | スレッドでのフォローアップ | スレッドの分離 | リアクションの観察 | ヘルプコマンド |
| ------- | ------ | ------------------------ | -------------------- | ---------------- | -------------- | -------------------------- | -------------- | ------------------ | -------------- |
| Matrix   | x      | x                        | x                    | x                | x              | x                          | x              | x                  |                |
| Telegram | x      |                          |                      |                  |                |                            |                |                    | x              |

これにより、`qa-channel` は広範なプロダクト挙動スイートとして維持されつつ、Matrix、Telegram、および今後のライブトランスポートが、明示的なトランスポート契約チェックリストを共有できます。

DockerをQAパスに持ち込まずに使い捨てのLinux VMレーンを実行するには、次を実行します:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

これにより新しいMultipassゲストが起動し、そのゲスト内で依存関係のインストールとOpenClawのビルドが行われ、`qa suite` が実行された後、通常のQAレポートとサマリーがホスト側の `.artifacts/qa-e2e/...` にコピーされます。  
シナリオ選択の挙動は、ホスト上の `qa suite` と同じものを再利用します。  
ホストとMultipassのスイート実行は、デフォルトで分離されたGatewayワーカーを使って複数の選択シナリオを並列実行し、最大64ワーカーまたは選択シナリオ数のいずれか小さい方まで利用します。ワーカー数を調整するには `--concurrency <count>` を使用し、直列実行には `--concurrency 1` を使用します。  
ライブ実行では、ゲストにとって実用的な対応QA認証入力が転送されます。具体的には、環境変数ベースのプロバイダーキー、QAライブプロバイダー設定パス、存在する場合の `CODEX_HOME` です。ゲストがマウントされたワークスペース経由で書き戻せるように、`--output-dir` はリポジトリルート配下に維持してください。

## リポジトリ連動のシード

シード資産は `qa/` にあります:

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

これらは、QA計画が人間とエージェントの両方から見えるように、意図的にgitに含められています。ベースライン一覧は、次をカバーできる程度に十分広く保つ必要があります:

- DMとチャネルチャット
- スレッド動作
- メッセージアクションのライフサイクル
- cronコールバック
- メモリー想起
- モデル切り替え
- サブエージェントのハンドオフ
- リポジトリ読み取りとドキュメント読み取り
- Lobster Invadersのような小さなビルドタスク1つ

## レポート

`qa-lab` は、観察されたバスタイムラインからMarkdown形式のプロトコルレポートをエクスポートします。  
レポートでは、次の点に答える必要があります:

- 何が機能したか
- 何が失敗したか
- 何がブロックされたままだったか
- どのフォローアップシナリオを追加する価値があるか

キャラクターとスタイルのチェックを行うには、同じシナリオを複数のライブモデル参照で実行し、評価済みMarkdownレポートを書き出します:

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

このコマンドはDockerではなく、ローカルのQA Gateway子プロセスを実行します。キャラクター評価シナリオでは、`SOUL.md` を通じてペルソナを設定し、その後、チャット、ワークスペースヘルプ、小さなファイルタスクのような通常のユーザーターンを実行する必要があります。候補モデルには、評価されていることを伝えないでください。このコマンドは各完全トランスクリプトを保持し、基本的な実行統計を記録したうえで、ジャッジモデルに高速モードかつ `xhigh` 推論で、自然さ、雰囲気、ユーモアに基づいて各実行を順位付けするよう求めます。  
プロバイダー比較時には `--blind-judge-models` を使用してください。ジャッジプロンプトには引き続きすべてのトランスクリプトと実行ステータスが渡されますが、候補参照は `candidate-01` のような中立ラベルに置き換えられ、レポートは解析後に順位を実際の参照へ対応付け直します。  
候補実行の思考レベルはデフォルトで `high` であり、それをサポートするOpenAIモデルでは `xhigh` が使われます。特定の候補を個別に上書きするには、`--model provider/model,thinking=<level>` をインラインで指定します。`--thinking <level>` は引き続きグローバルなフォールバックを設定し、旧形式の `--model-thinking <provider/model=level>` も互換性のために維持されています。  
OpenAI候補参照は、プロバイダーが対応している場合に優先処理が使われるよう、デフォルトで高速モードになります。個別の候補またはジャッジで上書きしたい場合は、`,fast`、`,no-fast`、または `,fast=false` をインラインで追加してください。すべての候補モデルで高速モードを強制的に有効にしたい場合にのみ `--fast` を渡してください。候補とジャッジの所要時間はベンチマーク分析のためにレポートへ記録されますが、ジャッジプロンプトでは速度で順位付けしないよう明示的に指示されています。  
候補モデル実行とジャッジモデル実行はいずれも、デフォルトで並列数16です。プロバイダー制限やローカルGateway負荷によって実行がうるさくなりすぎる場合は、`--concurrency` または `--judge-concurrency` を下げてください。  
候補の `--model` が渡されない場合、キャラクター評価ではデフォルトで `openai/gpt-5.4`、`openai/gpt-5.2`、`openai/gpt-5`、`anthropic/claude-opus-4-6`、`anthropic/claude-sonnet-4-6`、`zai/glm-5.1`、`moonshot/kimi-k2.5`、`google/gemini-3.1-pro-preview` が使用されます。  
`--judge-model` が渡されない場合、ジャッジはデフォルトで `openai/gpt-5.4,thinking=xhigh,fast` と `anthropic/claude-opus-4-6,thinking=high` になります。

## 関連ドキュメント

- [Testing](/ja-JP/help/testing)
- [QA Channel](/ja-JP/channels/qa-channel)
- [Dashboard](/web/dashboard)
