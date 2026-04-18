---
x-i18n:
    generated_at: "2026-04-18T04:40:46Z"
    model: gpt-5.4
    provider: openai
    source_hash: dbb2c70c82da7f6f12d90e25666635ff4147c52e8a94135e902d1de4f5cbccca
    source_path: refactor/qa.md
    workflow: 15
---

# QAリファクタリング

ステータス: 基盤となる移行は完了しました。

## 目標

OpenClaw QAを分割定義モデルから単一の信頼できる情報源へ移行します:

- シナリオメタデータ
- モデルに送信されるプロンプト
- セットアップとクリーンアップ
- ハーネスロジック
- アサーションと成功条件
- アーティファクトとレポートヒント

目指す最終状態は、TypeScript内にほとんどの挙動をハードコードするのではなく、強力なシナリオ定義ファイルを読み込む汎用QAハーネスです。

## 現在の状態

現在の主な信頼できる情報源は `qa/scenarios/index.md` と、各シナリオごとの `qa/scenarios/<theme>/*.md` にあります。

実装済み:

- `qa/scenarios/index.md`
  - 正式なQAパックメタデータ
  - オペレーターID
  - キックオフミッション
- `qa/scenarios/<theme>/*.md`
  - シナリオごとに1つのMarkdownファイル
  - シナリオメタデータ
  - ハンドラーバインディング
  - シナリオ固有の実行設定
- `extensions/qa-lab/src/scenario-catalog.ts`
  - Markdownパックパーサー + zod検証
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - Markdownパックからのプランレンダリング
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - 生成された互換ファイルと `QA_SCENARIOS.md` をシード
- `extensions/qa-lab/src/suite.ts`
  - Markdownで定義されたハンドラーバインディングを通じて実行可能なシナリオを選択
- QAバスプロトコル + UI
  - 画像/動画/音声/ファイル表示用の汎用インライン添付

依然として分割されたままの面:

- `extensions/qa-lab/src/suite.ts`
  - 依然として実行可能なカスタムハンドラーロジックの大半を管理
- `extensions/qa-lab/src/report.ts`
  - 依然としてレポート構造を実行時出力から導出

つまり、信頼できる情報源の分割自体は解消されましたが、実行はまだ完全宣言的ではなく、主にハンドラーバックです。

## 実際のシナリオ面はどのようなものか

現在のsuiteを読むと、いくつかの異なるシナリオクラスがあります。

### シンプルなやり取り

- チャネルベースライン
- DMベースライン
- スレッドでのフォローアップ
- モデル切り替え
- 承認フォロースルー
- リアクション/編集/削除

### 設定とランタイムの変更

- config patch skill disable
- config apply restart wake-up
- config restart capability flip
- runtime inventory drift check

### ファイルシステムとリポジトリアサーション

- source/docs discovery report
- build Lobster Invaders
- generated image artifact lookup

### メモリオーケストレーション

- memory recall
- channel context内のmemory tools
- memory failure fallback
- session memory ranking
- thread memory isolation
- memory dreaming sweep

### ツールとPlugin統合

- MCP plugin-tools call
- skill visibility
- skill hot install
- native image generation
- image roundtrip
- 添付からの画像理解

### マルチターンとマルチアクター

- subagent handoff
- subagent fanout synthesis
- restart recovery style flows

これらのカテゴリはDSL要件を左右するため重要です。プロンプト + 期待テキストの単純な一覧だけでは不十分です。

## 方針

### 単一の信頼できる情報源

`qa/scenarios/index.md` と `qa/scenarios/<theme>/*.md` を、作成元の信頼できる情報源として使用します。

このパックは次を維持する必要があります:

- レビュー時に人間が読みやすいこと
- 機械で解析できること
- 次を駆動できるだけの表現力があること:
  - suite実行
  - QAワークスペースのブートストラップ
  - QA Lab UIメタデータ
  - docs/discoveryプロンプト
  - レポート生成

### 推奨する記述形式

トップレベル形式としてMarkdownを使用し、その中に構造化YAMLを埋め込みます。

推奨形状:

- YAML frontmatter
  - id
  - title
  - surface
  - tags
  - docs refs
  - code refs
  - model/provider overrides
  - prerequisites
- prose sections
  - objective
  - notes
  - debugging hints
- fenced YAML blocks
  - setup
  - steps
  - assertions
  - cleanup

これにより次が得られます:

- 巨大なJSONより優れたPR可読性
- 純粋なYAMLより豊かな文脈
- 厳密な解析とzod検証

生のJSONは、中間生成形式としてのみ許容されます。

## 提案するシナリオファイル形状

例:

````md
---
id: image-generation-roundtrip
title: Image generation roundtrip
surface: image
tags: [media, image, roundtrip]
models:
  primary: openai/gpt-5.4
requires:
  tools: [image_generate]
  plugins: [openai, qa-channel]
docsRefs:
  - docs/help/testing.md
  - docs/concepts/model-providers.md
codeRefs:
  - extensions/qa-lab/src/suite.ts
  - src/gateway/chat-attachments.ts
---

# Objective

Verify generated media is reattached on the follow-up turn.

# Setup

```yaml scenario.setup
- action: config.patch
  patch:
    agents:
      defaults:
        imageGenerationModel:
          primary: openai/gpt-image-1
- action: session.create
  key: agent:qa:image-roundtrip
```

# Steps

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Image generation check: generate a QA lighthouse image and summarize it in one short sentence.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Image generation check
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Roundtrip image inspection check: describe the generated lighthouse attachment in one short sentence.
  attachments:
    - fromArtifact: lighthouseImage
```

# Expect

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Roundtrip image inspection check
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## DSLがカバーすべきランナー機能

現在のsuiteに基づくと、汎用ランナーにはプロンプト実行以上の機能が必要です。

### 環境およびセットアップアクション

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### エージェントターンアクション

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### 設定およびランタイムアクション

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### ファイルおよびアーティファクトアクション

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### メモリおよびCronアクション

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### MCPアクション

- `mcp.callTool`

### アサーション

- `outbound.textIncludes`
- `outbound.inThread`
- `outbound.notInRoot`
- `tool.called`
- `tool.notPresent`
- `skill.visible`
- `skill.disabled`
- `file.contains`
- `memory.contains`
- `requestLog.matches`
- `sessionStore.matches`
- `cron.managedPresent`
- `artifact.exists`

## 変数とアーティファクト参照

DSLは、保存済み出力と後続参照をサポートする必要があります。

現在のsuiteでの例:

- スレッドを作成し、その後 `threadId` を再利用する
- セッションを作成し、その後 `sessionKey` を再利用する
- 画像を生成し、次のターンでそのファイルを添付する
- wake marker文字列を生成し、それが後で現れることを検証する

必要な機能:

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- path、session key、thread id、marker、tool outputに対する型付き参照

変数サポートがなければ、ハーネスはシナリオロジックをTypeScriptへ漏らし続けることになります。

## エスケープハッチとして残すべきもの

フェーズ1で完全に純粋な宣言的ランナーを実現するのは現実的ではありません。

いくつかのシナリオは本質的にオーケストレーションが重いものです:

- memory dreaming sweep
- config apply restart wake-up
- config restart capability flip
- timestamp/pathによるgenerated image artifact resolution
- discovery-report evaluation

これらは、当面は明示的なカスタムハンドラーを使うべきです。

推奨ルール:

- 85〜90%は宣言的
- 残りの難しい部分には明示的な `customHandler` ステップ
- カスタムハンドラーは名前付きかつ文書化されていること
- シナリオファイル内に匿名インラインコードを置かないこと

これにより、進捗を維持しながら汎用エンジンをクリーンに保てます。

## アーキテクチャ変更

### 現在

シナリオMarkdownは、すでに次に対する信頼できる情報源です:

- suite実行
- ワークスペースブートストラップファイル
- QA Lab UIシナリオカタログ
- レポートメタデータ
- discoveryプロンプト

生成される互換要素:

- シードされたワークスペースには引き続き `QA_KICKOFF_TASK.md` が含まれる
- シードされたワークスペースには引き続き `QA_SCENARIO_PLAN.md` が含まれる
- シードされたワークスペースには現在 `QA_SCENARIOS.md` も含まれる

## リファクタリング計画

### Phase 1: loaderとschema

完了。

- `qa/scenarios/index.md` を追加
- シナリオを `qa/scenarios/<theme>/*.md` に分割
- 名前付きMarkdown YAMLパック内容用のパーサーを追加
- zodで検証
- 利用側を解析済みパックへ切り替え
- リポジトリレベルの `qa/seed-scenarios.json` と `qa/QA_KICKOFF_TASK.md` を削除

### Phase 2: 汎用エンジン

- `extensions/qa-lab/src/suite.ts` を以下に分割:
  - loader
  - engine
  - action registry
  - assertion registry
  - custom handlers
- 既存のヘルパー関数をエンジン操作として維持

成果物:

- エンジンがシンプルな宣言的シナリオを実行する

まずは、主に prompt + wait + assert で構成されるシナリオから開始します:

- threaded follow-up
- image understanding from attachment
- skill visibility and invocation
- channel baseline

成果物:

- 最初の本格的なMarkdown定義シナリオが汎用エンジン経由で提供される

### Phase 4: 中程度のシナリオを移行

- image generation roundtrip
- memory tools in channel context
- session memory ranking
- subagent handoff
- subagent fanout synthesis

成果物:

- variables、artifacts、tool assertions、request-log assertions が実証される

### Phase 5: 難しいシナリオはカスタムハンドラーに残す

- memory dreaming sweep
- config apply restart wake-up
- config restart capability flip
- runtime inventory drift

成果物:

- 記述形式は同じだが、必要な箇所に明示的なcustom-stepブロックを使う

### Phase 6: ハードコードされたシナリオマップを削除

パックのカバレッジが十分になったら:

- `extensions/qa-lab/src/suite.ts` からシナリオ固有のTypeScript分岐の大半を削除する

## Fake Slack / リッチメディアサポート

現在のQAバスはtext-firstです。

関連ファイル:

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

現在QAバスがサポートしているもの:

- テキスト
- リアクション
- スレッド

まだインラインメディア添付はモデル化されていません。

### 必要なトランスポート契約

汎用QAバス添付モデルを追加します:

```ts
type QaBusAttachment = {
  id: string;
  kind: "image" | "video" | "audio" | "file";
  mimeType: string;
  fileName?: string;
  inline?: boolean;
  url?: string;
  contentBase64?: string;
  width?: number;
  height?: number;
  durationMs?: number;
  altText?: string;
  transcript?: string;
};
```

次に `attachments?: QaBusAttachment[]` を以下へ追加します:

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### なぜ先に汎用化するのか

Slack専用のメディアモデルは作らないでください。

代わりに:

- 1つの汎用QAトランスポートモデル
- その上に複数のレンダラー
  - 現在のQA Lab chat
  - 将来のfake Slack web
  - その他のfake transportビュー

これによりロジックの重複を防ぎ、メディアシナリオをトランスポート非依存に保てます。

### 必要なUI作業

QA UIを更新して以下をレンダリングします:

- インライン画像プレビュー
- インライン音声プレーヤー
- インライン動画プレーヤー
- ファイル添付チップ

現在のUIはすでにスレッドとリアクションをレンダリングできるため、添付レンダリングは同じメッセージカードモデルの上に重ねられるはずです。

### メディアトランスポートで可能になるシナリオ作業

添付がQAバスを流れるようになれば、より豊かなfake-chatシナリオを追加できます:

- fake Slackでのインライン画像返信
- 音声添付の理解
- 動画添付の理解
- 混在添付の順序
- メディアを保持したスレッド返信

## 推奨事項

次に実装すべきまとまりは以下です:

1. Markdownシナリオローダー + zod schemaを追加
2. 現在のカタログをMarkdownから生成
3. まずいくつかのシンプルなシナリオを移行
4. 汎用QAバス添付サポートを追加
5. QA UIでインライン画像をレンダリング
6. その後、音声と動画へ拡張

これは次の2つの目標を実証する最小の道筋です:

- 汎用のMarkdown定義QA
- より豊かなfake messaging surfaces

## 未解決の質問

- シナリオファイルで、変数補間付きの埋め込みMarkdownプロンプトテンプレートを許可すべきかどうか
- setup/cleanupは名前付きセクションにすべきか、それとも単なる順序付きアクションリストにすべきか
- アーティファクト参照はschema内で強く型付けすべきか、それとも文字列ベースにすべきか
- カスタムハンドラーは1つのregistryに置くべきか、それともsurfaceごとのregistryにすべきか
- 移行期間中、生成されたJSON互換ファイルを引き続きチェックインしておくべきかどうか
