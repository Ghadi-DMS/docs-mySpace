---
read_when:
    - ネイティブなOpenClaw Plugin を構築またはデバッグしている場合
    - Plugin のケイパビリティモデルや所有境界を理解したい場合
    - Plugin のロードパイプラインやレジストリに取り組んでいる場合
    - プロバイダのランタイムフックやチャネル Plugin を実装している場合
sidebarTitle: Internals
summary: 'Plugin の内部: ケイパビリティモデル、所有権、コントラクト、ロードパイプライン、ランタイムヘルパー'
title: Plugin の内部
x-i18n:
    generated_at: "2026-04-12T23:28:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 37361c1e9d2da57c77358396f19dfc7f749708b66ff68f1bf737d051b5d7675d
    source_path: plugins/architecture.md
    workflow: 15
---

# Plugin の内部

<Info>
  これは**詳細なアーキテクチャリファレンス**です。実践的なガイドについては、以下を参照してください:
  - [Plugin のインストールと使用](/ja-JP/tools/plugin) — ユーザーガイド
  - [はじめに](/ja-JP/plugins/building-plugins) — 最初の Plugin チュートリアル
  - [チャネル Plugin](/ja-JP/plugins/sdk-channel-plugins) — メッセージングチャネルを構築する
  - [プロバイダ Plugin](/ja-JP/plugins/sdk-provider-plugins) — モデルプロバイダを構築する
  - [SDK 概要](/ja-JP/plugins/sdk-overview) — インポートマップと登録 API
</Info>

このページでは、OpenClaw Plugin システムの内部アーキテクチャを扱います。

## 公開ケイパビリティモデル

ケイパビリティは、OpenClaw 内における公開された**ネイティブ Plugin**モデルです。すべての
ネイティブ OpenClaw Plugin は、1つ以上のケイパビリティタイプに対して登録します:

| Capability             | 登録方法                                         | 例の Plugin                          |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| テキスト推論           | `api.registerProvider(...)`                      | `openai`, `anthropic`                |
| CLI 推論バックエンド   | `api.registerCliBackend(...)`                    | `openai`, `anthropic`                |
| 音声                   | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`            |
| リアルタイム文字起こし | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| リアルタイム音声       | `api.registerRealtimeVoiceProvider(...)`         | `openai`                             |
| メディア理解           | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                   |
| 画像生成               | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| 音楽生成               | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                  |
| 動画生成               | `api.registerVideoGenerationProvider(...)`       | `qwen`                               |
| Web 取得               | `api.registerWebFetchProvider(...)`              | `firecrawl`                          |
| Web 検索               | `api.registerWebSearchProvider(...)`             | `google`                             |
| チャネル / メッセージング | `api.registerChannel(...)`                    | `msteams`, `matrix`                  |

ケイパビリティを1つも登録せず、フック、ツール、サービスを提供する
Plugin は、**レガシーな hook-only** Plugin です。このパターンは現在も完全にサポートされています。

### 外部互換性に関する方針

ケイパビリティモデルはすでにコアに導入されており、現在はバンドルされた/ネイティブな Plugin
で使われていますが、外部 Plugin 互換性については、「エクスポートされているのだから固定されている」
よりも厳格な基準が依然として必要です。

現在のガイダンス:

- **既存の外部 Plugin:** フックベースの統合を動作し続けるようにする。これを
  互換性の基準線として扱う
- **新しいバンドル/ネイティブ Plugin:** ベンダー固有の直接到達や新しい hook-only 設計ではなく、
  明示的なケイパビリティ登録を優先する
- **ケイパビリティ登録を採用する外部 Plugin:** 許可されるが、ドキュメントで明示的に安定した
  コントラクトと示されない限り、ケイパビリティ固有のヘルパーサーフェスは進化中とみなす

実践的なルール:

- ケイパビリティ登録 API は意図された方向性である
- 移行期間中、レガシーフックは外部 Plugin にとって最も破壊のない安全な経路であり続ける
- エクスポートされたヘルパーのサブパスはすべて同等ではない。偶発的なヘルパーエクスポートではなく、
  文書化された狭いコントラクトを優先する

### Plugin の形態

OpenClaw は、読み込まれた各 Plugin を、実際の登録動作に基づいて形態分類します
（静的メタデータだけではありません）:

- **plain-capability** -- ちょうど1種類のケイパビリティだけを登録する（たとえば
  `mistral` のような provider-only Plugin）
- **hybrid-capability** -- 複数のケイパビリティタイプを登録する（たとえば
  `openai` はテキスト推論、音声、メディア理解、画像
  生成を所有する）
- **hook-only** -- フックのみを登録し（型付きまたはカスタム）、ケイパビリティ、
  ツール、コマンド、サービスは登録しない
- **non-capability** -- ツール、コマンド、サービス、またはルートを登録するが、
  ケイパビリティは登録しない

`openclaw plugins inspect <id>` を使うと、Plugin の形態とケイパビリティの
内訳を確認できます。詳細は [CLI リファレンス](/cli/plugins#inspect) を参照してください。

### レガシーフック

`before_agent_start` フックは、hook-only Plugin のための互換性経路として引き続きサポートされます。
現実のレガシー Plugin は今もこれに依存しています。

方向性:

- 動作し続けるようにする
- レガシーとして文書化する
- モデル/プロバイダのオーバーライド作業には `before_model_resolve` を優先する
- プロンプト変更作業には `before_prompt_build` を優先する
- 実際の利用が減少し、フィクスチャカバレッジで移行の安全性が証明されてから初めて削除する

### 互換性シグナル

`openclaw doctor` または `openclaw plugins inspect <id>` を実行すると、
次のいずれかのラベルが表示されることがあります:

| Signal                     | 意味                                                         |
| -------------------------- | ------------------------------------------------------------ |
| **config valid**           | 設定は正常にパースされ、Plugin も解決される                 |
| **compatibility advisory** | Plugin がサポートされているが古いパターンを使っている（例: `hook-only`） |
| **legacy warning**         | Plugin が `before_agent_start` を使っており、これは非推奨   |
| **hard error**             | 設定が不正、または Plugin の読み込みに失敗した              |

`hook-only` も `before_agent_start` も、現時点ではあなたの Plugin を壊しません --
`hook-only` は助言であり、`before_agent_start` は警告を出すだけです。これらの
シグナルは `openclaw status --all` と `openclaw plugins doctor` にも表示されます。

## アーキテクチャ概要

OpenClaw の Plugin システムは4つのレイヤーを持ちます:

1. **マニフェスト + ディスカバリ**
   OpenClaw は、設定されたパス、ワークスペースルート、
   グローバル拡張ルート、およびバンドルされた拡張から候補 Plugin を見つけます。
   ディスカバリでは、まずネイティブの `openclaw.plugin.json` マニフェストと
   サポートされるバンドルマニフェストを読み取ります。
2. **有効化 + バリデーション**
   コアは、発見された Plugin が有効、無効、ブロック済み、または
   memory のような排他的スロットに選択されているかどうかを判断します。
3. **ランタイム読み込み**
   ネイティブ OpenClaw Plugin は jiti を通じてプロセス内で読み込まれ、
   中央レジストリにケイパビリティを登録します。互換性のあるバンドルは
   ランタイムコードをインポートせずにレジストリレコードへ正規化されます。
4. **サーフェス消費**
   OpenClaw の残りの部分は、レジストリを読み取ってツール、チャネル、プロバイダ
   設定、フック、HTTP ルート、CLI コマンド、サービスを公開します。

特に Plugin CLI については、ルートコマンドのディスカバリは2段階に分かれています:

- パース時メタデータは `registerCli(..., { descriptors: [...] })` から取得される
- 実際の Plugin CLI モジュールは遅延読み込みのままにでき、最初の呼び出し時に登録される

これにより、Plugin 所有の CLI コードを Plugin 内に保ちながらも、OpenClaw は
パース前にルートコマンド名を確保できます。

重要な設計上の境界:

- ディスカバリ + 設定バリデーションは、Plugin コードを実行せずに
  **manifest/schema メタデータ**から動作できるべきである
- ネイティブのランタイム動作は、Plugin モジュールの `register(api)` パスから来る

この分離により、OpenClaw は、完全なランタイムが有効になる前に、
設定を検証し、不足/無効な Plugin を説明し、UI/schema のヒントを構築できます。

### チャネル Plugin と共有 message ツール

チャネル Plugin は、通常のチャットアクションのために別個の send/edit/react ツールを
登録する必要はありません。OpenClaw はコアに1つの共有 `message` ツールを保持し、
チャネル Plugin はその背後にあるチャネル固有のディスカバリと実行を所有します。

現在の境界は次のとおりです:

- コアは共有 `message` ツールホスト、プロンプト配線、セッション/スレッドの
  台帳管理、および実行ディスパッチを所有する
- チャネル Plugin はスコープ付きアクションディスカバリ、ケイパビリティディスカバリ、
  およびチャネル固有のスキーマ断片を所有する
- チャネル Plugin は、会話 ID がどのようにスレッド ID をエンコードするか、
  または親会話から継承するかといった、プロバイダ固有のセッション会話文法を所有する
- チャネル Plugin は、そのアクションアダプタを通じて最終アクションを実行する

チャネル Plugin に対する SDK サーフェスは
`ChannelMessageActionAdapter.describeMessageTool(...)` です。この統合されたディスカバリ
呼び出しにより、Plugin は可視アクション、ケイパビリティ、およびスキーマへの貢献を
まとめて返せるため、それらの要素がばらばらになりません。

コアはそのディスカバリ手順にランタイムスコープを渡します。重要なフィールドには次が含まれます:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- 信頼された受信 `requesterSenderId`

これはコンテキスト依存の Plugin にとって重要です。チャネルは、アクティブなアカウント、
現在のルーム/スレッド/メッセージ、または信頼された要求者 ID に基づいて、
コアの `message` ツールにチャネル固有の分岐をハードコードすることなく、
メッセージアクションを隠したり公開したりできます。

このため、embedded-runner のルーティング変更も依然として Plugin 側の作業です。ランナーは、
現在のチャット/セッション ID を Plugin ディスカバリ境界へ転送し、
共有 `message` ツールが現在のターンに適したチャネル所有のサーフェスを
公開できるようにする責任を持ちます。

チャネル所有の実行ヘルパーについては、バンドルされた Plugin は実行ランタイムを
自分たちの拡張モジュール内に保持すべきです。コアはもはや Discord、
Slack、Telegram、WhatsApp の message-action ランタイムを `src/agents/tools` 配下で所有しません。
別個の `plugin-sdk/*-action-runtime` サブパスは公開しておらず、バンドルされた
Plugin は自分たちのローカルなランタイムコードを、拡張所有モジュールから
直接インポートすべきです。

同じ境界は、一般にプロバイダ名付き SDK シームにも適用されます。コアは Slack、
Discord、Signal、WhatsApp、または類似拡張向けのチャネル固有 convenience barrel を
インポートすべきではありません。コアがある動作を必要とする場合は、バンドルされた
Plugin 自身の `api.ts` / `runtime-api.ts` barrel を利用するか、必要性を共有 SDK の
狭い汎用ケイパビリティへ昇格させてください。

poll については、特に2つの実行経路があります:

- `outbound.sendPoll` は、共通の poll モデルに適合するチャネル向けの共有ベースライン
- `actions.handleAction("poll")` は、チャネル固有の poll セマンティクスや追加の
  poll パラメータ向けの推奨経路

コアは現在、Plugin 側の poll ディスパッチがアクションを拒否した後まで共有 poll パースを
遅延させています。これにより、Plugin 所有の poll ハンドラは、汎用 poll パーサに先に
ブロックされることなく、チャネル固有の poll フィールドを受け入れられます。

完全な起動シーケンスについては、[ロードパイプライン](#load-pipeline) を参照してください。

## ケイパビリティ所有モデル

OpenClaw は、ネイティブ Plugin を、無関係な統合の寄せ集めではなく、
**企業**または**機能**の所有境界として扱います。

これは次を意味します:

- 企業 Plugin は、通常、その企業に属する OpenClaw 向けのサーフェス全体を所有すべきである
- 機能 Plugin は、通常、自らが導入する機能サーフェス全体を所有すべきである
- チャネルは、プロバイダの動作を場当たり的に再実装するのではなく、
  共有コアケイパビリティを利用すべきである

例:

- バンドルされた `openai` Plugin は、OpenAI モデルプロバイダ動作と、OpenAI の
  音声 + リアルタイム音声 + メディア理解 + 画像生成動作を所有する
- バンドルされた `elevenlabs` Plugin は、ElevenLabs の音声動作を所有する
- バンドルされた `microsoft` Plugin は、Microsoft の音声動作を所有する
- バンドルされた `google` Plugin は、Google モデルプロバイダ動作に加えて、
  Google のメディア理解 + 画像生成 + Web 検索動作を所有する
- バンドルされた `firecrawl` Plugin は、Firecrawl の Web 取得動作を所有する
- バンドルされた `minimax`、`mistral`、`moonshot`、`zai` Plugin は、
  それぞれのメディア理解バックエンドを所有する
- バンドルされた `qwen` Plugin は、Qwen テキストプロバイダ動作に加えて、
  メディア理解および動画生成動作を所有する
- `voice-call` Plugin は機能 Plugin です。通話トランスポート、ツール、
  CLI、ルート、および Twilio メディアストリームブリッジを所有しますが、
  ベンダー Plugin を直接インポートする代わりに、共有の音声、
  リアルタイム文字起こし、およびリアルタイム音声ケイパビリティを利用します

意図された最終状態は次のとおりです:

- OpenAI は、テキストモデル、音声、画像、将来の
  動画にまたがっていても、1つの Plugin に存在する
- 別のベンダーも、自身のサーフェス領域に対して同じことができる
- チャネルは、どのベンダー Plugin がそのプロバイダを所有しているかを気にしない。コアが公開する
  共有ケイパビリティコントラクトを利用する

これが重要な区別です:

- **plugin** = 所有境界
- **capability** = 複数の Plugin が実装または利用できるコアコントラクト

したがって、OpenClaw が動画のような新しい領域を追加する場合、最初の問いは
「どのプロバイダが動画処理をハードコードすべきか」ではありません。最初の問いは「コアの
動画ケイパビリティコントラクトとは何か」です。そのコントラクトが存在すれば、
ベンダー Plugin はそれに対して登録でき、チャネル/機能 Plugin はそれを利用できます。

そのケイパビリティがまだ存在しない場合、通常は次のように進めるのが正しい動きです:

1. コアで不足しているケイパビリティを定義する
2. それを型付きで Plugin API/ランタイムから公開する
3. そのケイパビリティに対してチャネル/機能を接続する
4. ベンダー Plugin に実装を登録させる

これにより、所有権を明示したまま、単一ベンダーや単発の Plugin 固有コードパスに依存する
コア動作を避けられます。

### ケイパビリティのレイヤリング

コードをどこに置くべきか判断するときは、次のメンタルモデルを使ってください:

- **コアケイパビリティ層**: 共有のオーケストレーション、ポリシー、フォールバック、設定
  マージルール、配信セマンティクス、型付きコントラクト
- **ベンダー Plugin 層**: ベンダー固有 API、認証、モデルカタログ、音声
  合成、画像生成、将来の動画バックエンド、使用量エンドポイント
- **チャネル/機能 Plugin 層**: Slack/Discord/voice-call などの統合で、
  コアケイパビリティを利用し、それをあるサーフェスとして提示する

たとえば、TTS は次の形に従います:

- コアは返信時 TTS のポリシー、フォールバック順序、設定、チャネル配信を所有する
- `openai`、`elevenlabs`、`microsoft` は合成実装を所有する
- `voice-call` は電話向け TTS ランタイムヘルパーを利用する

将来のケイパビリティについても、同じパターンを優先すべきです。

### 複数ケイパビリティを持つ企業 Plugin の例

企業 Plugin は、外から見たときに一貫性があるべきです。OpenClaw に、モデル、音声、
リアルタイム文字起こし、リアルタイム音声、メディア理解、画像生成、
動画生成、Web 取得、Web 検索の共有コントラクトがあるなら、
ベンダーは自身のサーフェスを1か所でまとめて所有できます:

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // auth/model catalog/runtime hooks
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // vendor speech config — implement the SpeechProviderPlugin interface directly
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // credential + fetch logic
      }),
    );
  },
};

export default plugin;
```

重要なのは、正確なヘルパー名ではありません。形が重要です:

- 1つの Plugin がそのベンダーサーフェスを所有する
- それでもコアはケイパビリティコントラクトを所有する
- チャネルと機能 Plugin は、ベンダーコードではなく `api.runtime.*` ヘルパーを利用する
- コントラクトテストで、その Plugin が自身が所有すると主張するケイパビリティを
  登録したことを検証できる

### ケイパビリティの例: 動画理解

OpenClaw はすでに、画像/音声/動画理解を1つの共有
ケイパビリティとして扱っています。そこでも同じ所有モデルが適用されます:

1. コアが media-understanding コントラクトを定義する
2. ベンダー Plugin が、必要に応じて `describeImage`、`transcribeAudio`、および
   `describeVideo` を登録する
3. チャネルと機能 Plugin は、ベンダーコードへ直接接続するのではなく、
   共有コア動作を利用する

これにより、あるプロバイダの動画に関する前提をコアへ焼き付けずに済みます。Plugin が
ベンダーサーフェスを所有し、コアがケイパビリティコントラクトとフォールバック動作を
所有します。

動画生成もすでに同じ流れを使っています。コアが型付き
ケイパビリティコントラクトとランタイムヘルパーを所有し、ベンダー Plugin が
`api.registerVideoGenerationProvider(...)` 実装をそれに対して登録します。

具体的なロールアウト用チェックリストが必要ですか? 参照:
[Capability Cookbook](/ja-JP/plugins/architecture)。

## コントラクトと強制

Plugin API サーフェスは、意図的に型付きかつ `OpenClawPluginApi` に
集約されています。このコントラクトが、サポートされる登録ポイントと、
Plugin が依存してよいランタイムヘルパーを定義します。

これが重要な理由:

- Plugin 作者は、1つの安定した内部標準を得られる
- コアは、2つの Plugin が同じ
  provider id を登録するような重複所有を拒否できる
- 起動時に、不正な登録に対して実行可能な診断を表示できる
- コントラクトテストで、バンドルされた Plugin の所有権を強制し、静かなドリフトを防げる

強制には2つのレイヤーがあります:

1. **ランタイム登録の強制**
   Plugin レジストリは、Plugin の読み込み時に登録内容を検証します。例:
   重複した provider id、重複した speech provider id、不正な
   登録は、未定義動作ではなく Plugin 診断を生成します。
2. **コントラクトテスト**
   バンドルされた Plugin は、テスト実行中にコントラクトレジストリへ取り込まれるため、
   OpenClaw は所有権を明示的に検証できます。現在これは、モデル
   プロバイダ、speech プロバイダ、Web 検索プロバイダ、およびバンドル登録の
   所有権に使われています。

実際の効果として、OpenClaw は、どの Plugin がどの
サーフェスを所有しているかをあらかじめ把握できます。これにより、所有権が暗黙ではなく
宣言され、型付けされ、テスト可能であるため、コアとチャネルはシームレスに構成できます。

### コントラクトに含めるべきもの

良い Plugin コントラクトは次の特徴を持ちます:

- 型付き
- 小さい
- ケイパビリティ固有
- コアが所有する
- 複数の Plugin で再利用できる
- ベンダー知識なしでチャネル/機能が利用できる

悪い Plugin コントラクトは次のようなものです:

- コア内に隠されたベンダー固有ポリシー
- レジストリを迂回する単発の Plugin 用抜け道
- ベンダー実装へ直接到達するチャネルコード
- `OpenClawPluginApi` や
  `api.runtime` の一部ではない、その場しのぎのランタイムオブジェクト

迷ったら、抽象度を1段引き上げてください。まずケイパビリティを定義し、その後で
Plugin がそこへ接続できるようにします。

## 実行モデル

ネイティブ OpenClaw Plugin は、Gateway と**同一プロセス内**で実行されます。サンドボックス化は
されていません。読み込まれたネイティブ Plugin は、コアコードと同じ
プロセスレベルの信頼境界を持ちます。

含意:

- ネイティブ Plugin は、ツール、ネットワークハンドラ、フック、サービスを登録できる
- ネイティブ Plugin のバグで gateway がクラッシュしたり不安定化したりする可能性がある
- 悪意あるネイティブ Plugin は、OpenClaw プロセス内での任意コード実行と等価である

互換バンドルは、OpenClaw が現在それらを
メタデータ/コンテンツパックとして扱っているため、デフォルトではより安全です。現行リリースでは、これは主に
バンドルされた Skills を意味します。

バンドルされていない Plugin には、allowlist と明示的な install/load パスを使用してください。ワークスペース
Plugin は本番デフォルトではなく、開発時コードとして扱ってください。

バンドルされたワークスペース package 名では、Plugin id を npm 名に
固定してください。デフォルトは `@openclaw/<id>`、またはその package が意図的により狭い
Plugin ロールを公開する場合に限り、承認済みの型付き接尾辞
`-provider`、`-plugin`、`-speech`、`-sandbox`、`-media-understanding` を使います。

重要な信頼上の注意:

- `plugins.allow` が信頼するのは**plugin id**であり、ソースの来歴ではありません。
- バンドルされた Plugin と同じ id を持つワークスペース Plugin は、そのワークスペース Plugin が
  有効化/allowlist されている場合、意図的にバンドル版をシャドーします。
- これは正常であり、ローカル開発、パッチテスト、hotfix に有用です。

## エクスポート境界

OpenClaw がエクスポートするのは、実装上の convenience ではなくケイパビリティです。

ケイパビリティ登録は公開のままにし、非コントラクトのヘルパーエクスポートは削減してください:

- バンドル Plugin 固有のヘルパーサブパス
- 公開 API を意図していないランタイム配管サブパス
- ベンダー固有の convenience ヘルパー
- 実装詳細である setup/onboarding ヘルパー

一部のバンドル Plugin ヘルパーサブパスは、互換性とバンドル Plugin 保守のために、
生成された SDK エクスポートマップにまだ残っています。現在の例には
`plugin-sdk/feishu`、`plugin-sdk/feishu-setup`、`plugin-sdk/zalo`、
`plugin-sdk/zalo-setup`、およびいくつかの `plugin-sdk/matrix*` シームが含まれます。これらは、
新しいサードパーティ Plugin に推奨される SDK パターンではなく、予約された実装詳細エクスポートとして
扱ってください。

## ロードパイプライン

起動時、OpenClaw はおおむね次の処理を行います:

1. 候補 Plugin ルートを発見する
2. ネイティブまたは互換バンドルのマニフェストと package メタデータを読む
3. 安全でない候補を拒否する
4. Plugin 設定（`plugins.enabled`、`allow`、`deny`、`entries`、
   `slots`、`load.paths`）を正規化する
5. 各候補の有効化可否を決定する
6. 有効なネイティブモジュールを jiti 経由で読み込む
7. ネイティブの `register(api)`（またはレガシーエイリアスである `activate(api)`）フックを呼び出し、登録内容を Plugin レジストリへ収集する
8. そのレジストリをコマンド/ランタイムサーフェスに公開する

<Note>
`activate` は `register` のレガシーエイリアスです — ローダーは存在する方（`def.register ?? def.activate`）を解決し、同じタイミングで呼び出します。すべてのバンドル Plugin は `register` を使っています。新しい Plugin では `register` を優先してください。
</Note>

安全ゲートは、ランタイム実行**前**に行われます。エントリが Plugin ルート外へ
エスケープする場合、パスが world-writable の場合、またはバンドルされていない Plugin について
パス所有権が不審に見える場合、その候補はブロックされます。

### マニフェストファーストの動作

マニフェストは、コントロールプレーンの信頼できる情報源です。OpenClaw はこれを使って次を行います:

- Plugin を識別する
- 宣言されたチャネル/Skills/設定スキーマまたはバンドルケイパビリティを発見する
- `plugins.entries.<id>.config` を検証する
- Control UI のラベル/プレースホルダを補強する
- install/catalog メタデータを表示する
- Plugin ランタイムを読み込まずに、軽量な activation と setup 記述子を保持する

ネイティブ Plugin では、ランタイムモジュールがデータプレーン部分です。そこではフック、ツール、コマンド、
またはプロバイダフローのような実際の動作が登録されます。

オプションのマニフェスト `activation` および `setup` ブロックは、コントロールプレーンに留まります。
これらは activation 計画と setup ディスカバリのためのメタデータ専用記述子であり、
ランタイム登録、`register(...)`、または `setupEntry` を置き換えるものではありません。
最初のライブ activation コンシューマは現在、マニフェストの command、channel、provider ヒントを使って、
より広いレジストリ実体化の前に Plugin ロードを絞り込みます:

- CLI ロードは、要求された主要コマンドを所有する Plugin に絞り込まれる
- チャネル setup/Plugin 解決は、要求された
  channel id を所有する Plugin に絞り込まれる
- 明示的なプロバイダ setup/ランタイム解決は、要求された
  provider id を所有する Plugin に絞り込まれる

setup ディスカバリは現在、まず `setup.providers` や
`setup.cliBackends` のような descriptor 所有 id を優先して候補 Plugin を絞り込み、その後で
なお setup 時ランタイムフックを必要とする Plugin に対して `setup-api` へフォールバックします。複数の
発見済み Plugin が同じ正規化済み setup provider または CLI backend
id を主張する場合、setup 参照はディスカバリ順序に依存せず、その曖昧な所有者を拒否します。

### ローダーがキャッシュするもの

OpenClaw は、短期間のプロセス内キャッシュを次の対象に対して保持します:

- ディスカバリ結果
- マニフェストレジストリデータ
- 読み込まれた Plugin レジストリ

これらのキャッシュは、突発的な起動や繰り返しコマンドのオーバーヘッドを減らします。これらは
永続化ではなく、短命なパフォーマンスキャッシュとして考えると安全です。

パフォーマンス上の注意:

- `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` または
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` を設定すると、これらのキャッシュを無効化できます。
- キャッシュウィンドウは `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` と
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS` で調整します。

## レジストリモデル

読み込まれた Plugin は、ランダムなコアのグローバル状態を直接変更しません。代わりに、
中央の Plugin レジストリへ登録します。

レジストリが追跡するもの:

- Plugin レコード（識別情報、ソース、オリジン、ステータス、診断）
- ツール
- レガシーフックと型付きフック
- チャネル
- プロバイダ
- Gateway RPC ハンドラ
- HTTP ルート
- CLI レジストラ
- バックグラウンドサービス
- Plugin 所有のコマンド

その後、コア機能は Plugin モジュールへ直接話しかける代わりに、そのレジストリから読み取ります。
これにより、読み込み方向は一方向に保たれます:

- Plugin モジュール -> レジストリ登録
- コアランタイム -> レジストリ利用

この分離は保守性にとって重要です。これは、ほとんどのコアサーフェスが必要とする統合点が
「レジストリを読むこと」1つで済み、「すべての Plugin モジュールを特別扱いすること」では
ないことを意味します。

## 会話バインディングコールバック

会話をバインドする Plugin は、承認が解決されたときに反応できます。

バインド要求が承認または拒否された後にコールバックを受け取るには、
`api.onConversationBindingResolved(...)` を使用します:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // A binding now exists for this plugin + conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // The request was denied; clear any local pending state.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

コールバックのペイロードフィールド:

- `status`: `"approved"` または `"denied"`
- `decision`: `"allow-once"`、`"allow-always"`、または `"deny"`
- `binding`: 承認済み要求に対して解決されたバインディング
- `request`: 元の要求サマリ、detach ヒント、sender id、および
  会話メタデータ

このコールバックは通知専用です。これは会話をバインドできる主体を変更せず、
コアの承認処理が完了した後に実行されます。

## プロバイダランタイムフック

プロバイダ Plugin には現在2つのレイヤーがあります:

- マニフェストメタデータ: ランタイム読み込み前に軽量なプロバイダ env 認証参照を行うための `providerAuthEnvVars`、
  認証を共有するプロバイダバリアント向けの `providerAuthAliases`、
  ランタイム読み込み前に軽量なチャネル env/setup 参照を行うための `channelEnvVars`、
  さらに、ランタイム読み込み前に軽量なオンボーディング/認証選択ラベルと
  CLI フラグメタデータを扱うための `providerAuthChoices`
- 設定時フック: `catalog` / レガシーな `discovery` と `applyConfigDefaults`
- ランタイムフック: `normalizeModelId`、`normalizeTransport`、
  `normalizeConfig`、
  `applyNativeStreamingUsageCompat`、`resolveConfigApiKey`、
  `resolveSyntheticAuth`、`resolveExternalAuthProfiles`、
  `shouldDeferSyntheticProfileAuth`、
  `resolveDynamicModel`、`prepareDynamicModel`、`normalizeResolvedModel`、
  `contributeResolvedModelCompat`、`capabilities`、
  `normalizeToolSchemas`、`inspectToolSchemas`、
  `resolveReasoningOutputMode`、`prepareExtraParams`、`createStreamFn`、
  `wrapStreamFn`、`resolveTransportTurnState`、
  `resolveWebSocketSessionPolicy`、`formatApiKey`、`refreshOAuth`、
  `buildAuthDoctorHint`、`matchesContextOverflowError`、
  `classifyFailoverReason`、`isCacheTtlEligible`、
  `buildMissingAuthMessage`、`suppressBuiltInModel`、`augmentModelCatalog`、
  `isBinaryThinking`、`supportsXHighThinking`、
  `resolveDefaultThinkingLevel`、`isModernModelRef`、`prepareRuntimeAuth`、
  `resolveUsageAuth`、`fetchUsageSnapshot`、`createEmbeddingProvider`、
  `buildReplayPolicy`、
  `sanitizeReplayHistory`、`validateReplayTurns`、`onModelSelected`

OpenClaw は依然として汎用エージェントループ、フェイルオーバー、トランスクリプト処理、および
ツールポリシーを所有します。これらのフックは、プロバイダ固有の振る舞いを拡張するためのサーフェスであり、
推論トランスポート全体をカスタム実装しなくても済みます。

プロバイダが env ベースの認証情報を持ち、汎用の auth/status/model-picker 経路から
Plugin ランタイムを読み込まずに見える必要がある場合は、マニフェスト `providerAuthEnvVars` を使います。
ある provider id が別の provider id の env vars、auth profiles、
設定ベース認証、および API キーオンボーディング選択を再利用すべき場合は、
マニフェスト `providerAuthAliases` を使います。オンボーディング/認証選択の
CLI サーフェスが、そのプロバイダの選択 id、グループラベル、および単純な
単一フラグ認証配線を、プロバイダランタイムを読み込まずに知る必要がある場合は、
マニフェスト `providerAuthChoices` を使います。プロバイダランタイムの
`envVars` は、オンボーディングラベルや OAuth の
client-id/client-secret セットアップ変数のような、オペレーター向けヒント用に保持してください。

チャネルに env 駆動の認証または setup があり、汎用 shell-env フォールバック、
config/status チェック、または setup プロンプトから、チャネルランタイムを読み込まずに
見える必要がある場合は、マニフェスト `channelEnvVars` を使います。

### フック順序と使い分け

モデル/プロバイダ Plugin では、OpenClaw はおおむね次の順序でフックを呼び出します。
「When to use」列は、素早く判断するためのガイドです。

| #   | Hook                              | 役割                                                                                                           | 使うべきタイミング                                                                                                                          |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | `models.json` 生成中に、プロバイダ設定を `models.providers` へ公開する                                         | プロバイダがカタログまたは base URL のデフォルトを所有している場合                                                                          |
| 2   | `applyConfigDefaults`             | 設定の具体化中に、プロバイダ所有のグローバル設定デフォルトを適用する                                          | デフォルトが認証モード、env、またはプロバイダのモデルファミリのセマンティクスに依存する場合                                               |
| --  | _(built-in model lookup)_         | OpenClaw はまず通常のレジストリ/カタログ経路を試す                                                            | _(Plugin フックではない)_                                                                                                                   |
| 3   | `normalizeModelId`                | 参照前に、レガシーまたはプレビューの model-id エイリアスを正規化する                                          | 正規のモデル解決の前に、プロバイダがエイリアス整理を所有している場合                                                                       |
| 4   | `normalizeTransport`              | 汎用的なモデル組み立ての前に、プロバイダファミリの `api` / `baseUrl` を正規化する                             | 同じトランスポートファミリ内のカスタム provider id に対するトランスポート整理を、プロバイダが所有している場合                           |
| 5   | `normalizeConfig`                 | ランタイム/プロバイダ解決前に `models.providers.<id>` を正規化する                                             | Plugin とともに置くべき設定整理がプロバイダに必要な場合。バンドルされた Google ファミリのヘルパーは、サポートされる Google 設定エントリも補完する |
| 6   | `applyNativeStreamingUsageCompat` | 設定プロバイダにネイティブ streaming-usage 互換リライトを適用する                                              | プロバイダに、エンドポイント駆動の native streaming usage メタデータ修正が必要な場合                                                      |
| 7   | `resolveConfigApiKey`             | ランタイム認証読み込み前に、設定プロバイダ向けの env-marker 認証を解決する                                     | プロバイダ所有の env-marker API キー解決がある場合。`amazon-bedrock` には、ここに組み込みの AWS env-marker リゾルバもある                |
| 8   | `resolveSyntheticAuth`            | 平文を永続化せずに、ローカル/セルフホスト型または設定ベースの認証を公開する                                   | プロバイダが synthetic/local な資格情報マーカーで動作できる場合                                                                            |
| 9   | `resolveExternalAuthProfiles`     | プロバイダ所有の外部認証プロファイルをオーバーレイする。デフォルトの `persistence` は CLI/アプリ所有資格情報向けに `runtime-only` | コピーしたリフレッシュトークンを永続化せずに、プロバイダが外部認証資格情報を再利用する場合                                                 |
| 10  | `shouldDeferSyntheticProfileAuth` | 保存済みの synthetic プロファイルプレースホルダを env/config ベース認証より後順位に下げる                     | プロバイダが、優先されるべきでない synthetic プレースホルダプロファイルを保存する場合                                                     |
| 11  | `resolveDynamicModel`             | まだローカルレジストリにないプロバイダ所有 model id に対して同期フォールバックを行う                          | プロバイダが任意の上流 model id を受け入れる場合                                                                                           |
| 12  | `prepareDynamicModel`             | 非同期ウォームアップを行い、その後 `resolveDynamicModel` を再度実行する                                        | 未知の id を解決する前に、プロバイダがネットワークメタデータを必要とする場合                                                               |
| 13  | `normalizeResolvedModel`          | embedded runner が解決済みモデルを使う前の最終リライト                                                         | プロバイダがトランスポートのリライトを必要とするが、なおコアのトランスポートを使う場合                                                    |
| 14  | `contributeResolvedModelCompat`   | 別の互換トランスポートの背後にあるベンダーモデル向けの互換フラグを提供する                                     | プロバイダを引き継がずに、プロバイダがプロキシトランスポート上の自前モデルを認識する場合                                                  |
| 15  | `capabilities`                    | 共有コアロジックで使われる、プロバイダ所有の transcript/tooling メタデータ                                     | プロバイダに transcript/プロバイダファミリ固有の癖がある場合                                                                               |
| 16  | `normalizeToolSchemas`            | embedded runner が見る前に、ツールスキーマを正規化する                                                         | プロバイダにトランスポートファミリのスキーマ整理が必要な場合                                                                               |
| 17  | `inspectToolSchemas`              | 正規化後に、プロバイダ所有のスキーマ診断を公開する                                                             | コアへプロバイダ固有ルールを教え込まずに、プロバイダがキーワード警告を出したい場合                                                       |
| 18  | `resolveReasoningOutputMode`      | ネイティブまたはタグ付きの reasoning-output コントラクトを選択する                                             | プロバイダがネイティブフィールドではなく、タグ付きの reasoning/final 出力を必要とする場合                                                 |
| 19  | `prepareExtraParams`              | 汎用ストリームオプションラッパーの前に、リクエストパラメータを正規化する                                       | プロバイダにデフォルトのリクエストパラメータ、またはプロバイダごとのパラメータ整理が必要な場合                                           |
| 20  | `createStreamFn`                  | 通常のストリーム経路を完全に置き換え、カスタムトランスポートを使う                                             | プロバイダがラッパーだけではなく、カスタムの wire protocol を必要とする場合                                                               |
| 21  | `wrapStreamFn`                    | 汎用ラッパー適用後にストリームをラップする                                                                     | プロバイダに、カスタムトランスポートなしでリクエストヘッダー/ボディ/モデル互換ラッパーが必要な場合                                       |
| 22  | `resolveTransportTurnState`       | ネイティブなターン単位トランスポートヘッダーまたはメタデータを付加する                                         | 汎用トランスポートでプロバイダネイティブなターン識別情報を送信したい場合                                                                   |
| 23  | `resolveWebSocketSessionPolicy`   | ネイティブ WebSocket ヘッダーまたはセッションクールダウンポリシーを付加する                                    | 汎用 WS トランスポートで、プロバイダがセッションヘッダーやフォールバックポリシーを調整したい場合                                         |
| 24  | `formatApiKey`                    | auth-profile フォーマッタ: 保存済みプロファイルをランタイムの `apiKey` 文字列へ変換する                       | プロバイダが追加の認証メタデータを保存し、カスタムのランタイムトークン形式を必要とする場合                                                |
| 25  | `refreshOAuth`                    | カスタムのリフレッシュエンドポイントまたはリフレッシュ失敗ポリシー向けの OAuth リフレッシュ上書き             | プロバイダが共有の `pi-ai` リフレッシャーに適合しない場合                                                                                  |
| 26  | `buildAuthDoctorHint`             | OAuth リフレッシュ失敗時に追記される修復ヒント                                                                 | リフレッシュ失敗後に、プロバイダ所有の認証修復ガイダンスが必要な場合                                                                       |
| 27  | `matchesContextOverflowError`     | プロバイダ所有のコンテキストウィンドウ超過エラーマッチャ                                                       | 汎用ヒューリスティックでは拾えない生の超過エラーを、プロバイダが持っている場合                                                           |
| 28  | `classifyFailoverReason`          | プロバイダ所有のフェイルオーバー理由分類                                                                       | 生の API/トランスポートエラーを、プロバイダがレート制限/過負荷などへ分類できる場合                                                        |
| 29  | `isCacheTtlEligible`              | プロキシ/バックホール型プロバイダ向けのプロンプトキャッシュポリシー                                            | プロバイダにプロキシ固有のキャッシュ TTL ゲーティングが必要な場合                                                                          |
| 30  | `buildMissingAuthMessage`         | 汎用 missing-auth リカバリメッセージの代替                                                                     | プロバイダ固有の missing-auth リカバリヒントが必要な場合                                                                                  |
| 31  | `suppressBuiltInModel`            | 古い上流モデルの抑制と、オプションのユーザー向けエラーヒント                                                   | 古い上流行を非表示にする、またはベンダーヒントに置き換える必要がある場合                                                                   |
| 32  | `augmentModelCatalog`             | discovery 後に synthetic/final なカタログ行を追加する                                                          | `models list` や picker に、synthetic な forward-compat 行をプロバイダが必要とする場合                                                    |
| 33  | `isBinaryThinking`                | binary-thinking プロバイダ向けの on/off 推論トグル                                                             | プロバイダが on/off の二値 thinking だけを公開する場合                                                                                     |
| 34  | `supportsXHighThinking`           | 選択されたモデル向けの `xhigh` 推論サポート                                                                     | プロバイダがモデルの一部サブセットにのみ `xhigh` を提供したい場合                                                                          |
| 35  | `resolveDefaultThinkingLevel`     | 特定のモデルファミリ向けのデフォルト `/think` レベル                                                           | モデルファミリに対するデフォルト `/think` ポリシーを、プロバイダが所有する場合                                                             |
| 36  | `isModernModelRef`                | ライブプロファイルフィルタおよびスモーク選択向けの modern-model マッチャ                                       | ライブ/スモーク用の優先モデルマッチングを、プロバイダが所有する場合                                                                        |
| 37  | `prepareRuntimeAuth`              | 推論直前に、設定済み資格情報を実際のランタイムトークン/キーへ交換する                                          | プロバイダがトークン交換または短命なリクエスト資格情報を必要とする場合                                                                     |
| 38  | `resolveUsageAuth`                | `/usage` および関連するステータスサーフェス向けの使用量/課金資格情報を解決する                                | プロバイダに、カスタムの使用量/クォータトークン解析、または異なる使用量資格情報が必要な場合                                              |
| 39  | `fetchUsageSnapshot`              | 認証解決後に、プロバイダ固有の使用量/クォータスナップショットを取得して正規化する                             | プロバイダに、プロバイダ固有の使用量エンドポイントまたはペイロードパーサが必要な場合                                                     |
| 40  | `createEmbeddingProvider`         | memory/search 向けの、プロバイダ所有の埋め込みアダプタを構築する                                               | memory の埋め込み動作がプロバイダ Plugin に属する場合                                                                                      |
| 41  | `buildReplayPolicy`               | そのプロバイダ向けの transcript 処理を制御するリプレイポリシーを返す                                           | プロバイダにカスタム transcript ポリシー（たとえば thinking ブロックの除去）が必要な場合                                                 |
| 42  | `sanitizeReplayHistory`           | 汎用 transcript クリーンアップ後に、リプレイ履歴を書き換える                                                   | 共有 Compaction ヘルパーを超える、プロバイダ固有のリプレイ書き換えが必要な場合                                                           |
| 43  | `validateReplayTurns`             | embedded runner の前に、最終的なリプレイターン検証または整形を行う                                             | プロバイダのトランスポートが、汎用サニタイズ後により厳密なターン検証を必要とする場合                                                     |
| 44  | `onModelSelected`                 | モデルがアクティブになったときに、プロバイダ所有の選択後副作用を実行する                                       | モデルが有効化されたときに、プロバイダがテレメトリまたはプロバイダ所有状態を必要とする場合                                               |

`normalizeModelId`、`normalizeTransport`、`normalizeConfig` は、まず
一致したプロバイダ Plugin を確認し、その後、実際に model id または transport/config を変更する
フック対応プロバイダ Plugin が見つかるまで、他のプロバイダ Plugin へ順にフォールスルーします。
これにより、呼び出し側がどのバンドル Plugin がそのリライトを所有しているかを知らなくても、
alias/互換プロバイダ shim が動作し続けます。サポートされている
Google ファミリの設定エントリをどのプロバイダフックもリライトしない場合でも、
バンドルされた Google 設定ノーマライザがその互換性クリーンアップを適用します。

プロバイダが完全にカスタムな wire protocol やカスタムのリクエスト実行器を必要とする場合、
それは別種の拡張です。これらのフックは、なお OpenClaw の通常の推論ループ上で動作する
プロバイダ挙動のためのものです。

### プロバイダの例

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### 組み込みの例

- Anthropic は `resolveDynamicModel`、`capabilities`、`buildAuthDoctorHint`、
  `resolveUsageAuth`、`fetchUsageSnapshot`、`isCacheTtlEligible`、
  `resolveDefaultThinkingLevel`、`applyConfigDefaults`、`isModernModelRef`、
  `wrapStreamFn` を使います。これは、Claude 4.6 の forward-compat、
  プロバイダファミリのヒント、認証修復ガイダンス、使用量エンドポイント統合、
  プロンプトキャッシュ適格性、認証対応の設定デフォルト、Claude の
  デフォルト/適応 thinking ポリシー、およびベータヘッダー、`/fast` / `serviceTier`、
  `context1m` 向けの Anthropic 固有のストリーム整形を所有しているためです。
- Anthropic の Claude 固有ストリームヘルパーは、現時点ではバンドルされた Plugin 自身の
  公開 `api.ts` / `contract-api.ts` シームに留まっています。その package サーフェスは、
  汎用 SDK を1つのプロバイダのベータヘッダールール向けに広げる代わりに、
  `wrapAnthropicProviderStream`、`resolveAnthropicBetas`、
  `resolveAnthropicFastMode`、`resolveAnthropicServiceTier`、およびより低レベルの
  Anthropic ラッパービルダをエクスポートします。
- OpenAI は `resolveDynamicModel`、`normalizeResolvedModel`、
  `capabilities`、さらに `buildMissingAuthMessage`、`suppressBuiltInModel`、
  `augmentModelCatalog`、`supportsXHighThinking`、`isModernModelRef`
  を使います。これは GPT-5.4 の forward-compat、直接の OpenAI
  `openai-completions` -> `openai-responses` 正規化、Codex 認識の認証
  ヒント、Spark 抑制、synthetic な OpenAI リスト行、および GPT-5 の thinking /
  ライブモデルポリシーを所有しているためです。また、`openai-responses-defaults` ストリームファミリは、
  attribution ヘッダー、`/fast`/`serviceTier`、テキスト冗長度、ネイティブ Codex Web 検索、
  reasoning-compat ペイロード整形、および Responses のコンテキスト管理向けの
  共有ネイティブ OpenAI Responses ラッパーを所有します。
- OpenRouter は `catalog`、`resolveDynamicModel`、
  `prepareDynamicModel` を使います。これは、そのプロバイダがパススルー型であり、
  OpenClaw の静的カタログ更新前に新しい
  model id を公開する可能性があるためです。また、
  `capabilities`、`wrapStreamFn`、`isCacheTtlEligible` も使い、
  プロバイダ固有のリクエストヘッダー、ルーティングメタデータ、reasoning パッチ、
  プロンプトキャッシュポリシーをコアの外に保ちます。そのリプレイポリシーは
  `passthrough-gemini` ファミリから来ており、`openrouter-thinking` ストリームファミリは
  プロキシ reasoning 注入と未サポートモデル / `auto` スキップを所有します。
- GitHub Copilot は `catalog`、`auth`、`resolveDynamicModel`、
  `capabilities`、さらに `prepareRuntimeAuth` と `fetchUsageSnapshot` を使います。これは、
  プロバイダ所有のデバイスログイン、モデルフォールバック動作、Claude の transcript
  の癖、GitHub トークン -> Copilot トークン交換、およびプロバイダ所有の使用量エンドポイントを必要とするためです。
- OpenAI Codex は `catalog`、`resolveDynamicModel`、
  `normalizeResolvedModel`、`refreshOAuth`、`augmentModelCatalog`、
  さらに `prepareExtraParams`、`resolveUsageAuth`、`fetchUsageSnapshot` を使います。これは、
  依然としてコアの OpenAI トランスポート上で動作しつつも、その transport/base URL
  正規化、OAuth リフレッシュフォールバックポリシー、デフォルトトランスポート選択、
  synthetic な Codex カタログ行、および ChatGPT 使用量エンドポイント統合を所有しているためです。
  これは直接の OpenAI と同じ `openai-responses-defaults` ストリームファミリを共有します。
- Google AI Studio と Gemini CLI OAuth は `resolveDynamicModel`、
  `buildReplayPolicy`、`sanitizeReplayHistory`、
  `resolveReasoningOutputMode`、`wrapStreamFn`、`isModernModelRef` を使います。これは、
  `google-gemini` リプレイファミリが Gemini 3.1 の forward-compat フォールバック、
  ネイティブ Gemini リプレイ検証、bootstrap リプレイサニテーション、タグ付き
  reasoning-output モード、および modern-model マッチングを所有し、
  `google-thinking` ストリームファミリが Gemini thinking ペイロード正規化を所有するためです。
  Gemini CLI OAuth はさらに、トークン整形、トークン解析、およびクォータ
  エンドポイント接続のために `formatApiKey`、`resolveUsageAuth`、`fetchUsageSnapshot` も使います。
- Anthropic Vertex は、
  `anthropic-by-model` リプレイファミリを通じて `buildReplayPolicy` を使います。これにより、
  Claude 固有のリプレイクリーンアップが、すべての `anthropic-messages` トランスポートではなく
  Claude id にスコープされます。
- Amazon Bedrock は `buildReplayPolicy`、`matchesContextOverflowError`、
  `classifyFailoverReason`、`resolveDefaultThinkingLevel` を使います。これは、
  Anthropic-on-Bedrock トラフィック向けに Bedrock 固有の
  throttle/not-ready/context-overflow エラー分類を所有しているためです。そのリプレイポリシーは
  依然として同じ Claude 専用 `anthropic-by-model` ガードを共有します。
- OpenRouter、Kilocode、Opencode、Opencode Go は `buildReplayPolicy`
  を `passthrough-gemini` リプレイファミリ経由で使います。これは、それらが Gemini
  モデルを OpenAI 互換トランスポート経由でプロキシしており、ネイティブ Gemini リプレイ検証や
  bootstrap リライトなしで、Gemini の thought-signature サニテーションを必要とするためです。
- MiniMax は、
  `hybrid-anthropic-openai` リプレイファミリ経由で `buildReplayPolicy` を使います。これは、
  1つのプロバイダが Anthropic-message と OpenAI 互換の両方のセマンティクスを所有しているためです。
  これにより、Anthropic 側では Claude 専用の thinking ブロック削除を維持しつつ、
  reasoning 出力モードをネイティブへ戻し、`minimax-fast-mode` ストリームファミリは
  共有ストリーム経路で fast-mode モデルリライトを所有します。
- Moonshot は `catalog` と `wrapStreamFn` を使います。これは、依然として共有
  OpenAI トランスポートを使いながら、プロバイダ所有の thinking ペイロード正規化を必要とするためです。
  `moonshot-thinking` ストリームファミリは、設定と `/think` 状態を
  そのネイティブな binary thinking ペイロードへマッピングします。
- Kilocode は `catalog`、`capabilities`、`wrapStreamFn`、
  `isCacheTtlEligible` を使います。これは、プロバイダ所有のリクエストヘッダー、
  reasoning ペイロード正規化、Gemini transcript ヒント、および Anthropic
  cache-TTL ゲーティングを必要とするためです。`kilocode-thinking` ストリームファミリは、
  `kilo/auto` や、明示的な reasoning ペイロードをサポートしない他のプロキシ model id を
  スキップしつつ、Kilo thinking 注入を共有プロキシストリーム経路上に保持します。
- Z.AI は `resolveDynamicModel`、`prepareExtraParams`、`wrapStreamFn`、
  `isCacheTtlEligible`、`isBinaryThinking`、`isModernModelRef`、
  `resolveUsageAuth`、`fetchUsageSnapshot` を使います。これは、GLM-5 フォールバック、
  `tool_stream` のデフォルト、binary thinking UX、modern-model マッチング、
  および使用量認証とクォータ取得の両方を所有しているためです。`tool-stream-default-on` ストリームファミリは、
  デフォルトで有効な `tool_stream` ラッパーを、プロバイダごとの手書き glue から切り離して保ちます。
- xAI は `normalizeResolvedModel`、`normalizeTransport`、
  `contributeResolvedModelCompat`、`prepareExtraParams`、`wrapStreamFn`、
  `resolveSyntheticAuth`、`resolveDynamicModel`、`isModernModelRef`
  を使います。これは、ネイティブ xAI Responses トランスポート正規化、Grok fast-mode
  エイリアスリライト、デフォルトの `tool_stream`、strict-tool / reasoning-payload
  整理、Plugin 所有ツール向けのフォールバック認証再利用、forward-compat な Grok
  モデル解決、および xAI の tool-schema
  プロファイル、未サポートのスキーマキーワード、ネイティブ `web_search`、HTML エンティティ化された
  tool-call 引数デコードのようなプロバイダ所有の互換パッチを所有しているためです。
- Mistral、OpenCode Zen、OpenCode Go は、
  transcript/tooling の癖をコアの外に保つために `capabilities` のみを使います。
- `byteplus`、`cloudflare-ai-gateway`、
  `huggingface`、`kimi-coding`、`nvidia`、`qianfan`、
  `synthetic`、`together`、`venice`、`vercel-ai-gateway`、`volcengine` のような
  catalog-only のバンドルプロバイダは、`catalog` のみを使います。
- Qwen は、テキストプロバイダ向けの `catalog` と、そのマルチモーダルサーフェス向けの
  共有 media-understanding および video-generation 登録を使います。
- MiniMax と Xiaomi は、推論自体は共有トランスポート経由で動作するにもかかわらず、
  `/usage` の動作が Plugin 所有であるため、`catalog` と使用量フックを使います。

## ランタイムヘルパー

Plugin は、`api.runtime` を通じて選択されたコアヘルパーへアクセスできます。TTS の場合:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

注記:

- `textToSpeech` は、ファイル/voice-note サーフェス向けの通常のコア TTS 出力ペイロードを返します。
- コアの `messages.tts` 設定とプロバイダ選択を使います。
- PCM 音声バッファ + サンプルレートを返します。Plugin 側でプロバイダ向けにリサンプリング/エンコードする必要があります。
- `listVoices` はプロバイダごとに任意です。ベンダー所有の voice picker や setup フローに使ってください。
- 音声一覧には、ロケール、性別、パーソナリティタグなどの、より豊富なメタデータを含めることができ、プロバイダ認識型 picker に使えます。
- 現時点で電話対応しているのは OpenAI と ElevenLabs です。Microsoft は未対応です。

Plugin は `api.registerSpeechProvider(...)` で speech プロバイダも登録できます。

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

注記:

- TTS ポリシー、フォールバック、返信配信はコアに維持してください。
- ベンダー所有の合成動作には speech プロバイダを使ってください。
- レガシーな Microsoft の `edge` 入力は `microsoft` プロバイダ id に正規化されます。
- 推奨される所有モデルは企業志向です。OpenClaw がそうした
  ケイパビリティコントラクトを追加していくにつれ、1つのベンダー Plugin が
  テキスト、音声、画像、将来のメディアプロバイダを所有できます。

画像/音声/動画理解については、Plugin は汎用 key/value bag ではなく、
1つの型付き media-understanding プロバイダを登録します:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

注記:

- オーケストレーション、フォールバック、設定、チャネル配線はコアに維持する
- ベンダーの振る舞いはプロバイダ Plugin に維持する
- 加算的な拡張は型付きのまま維持する: 新しい任意メソッド、新しい任意の
  結果フィールド、新しい任意ケイパビリティ
- 動画生成もすでに同じパターンに従っています:
  - コアがケイパビリティコントラクトとランタイムヘルパーを所有する
  - ベンダー Plugin が `api.registerVideoGenerationProvider(...)` を登録する
  - 機能/チャネル Plugin が `api.runtime.videoGeneration.*` を利用する

media-understanding のランタイムヘルパーでは、Plugin は次のように呼び出せます:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

音声文字起こしについては、Plugin は media-understanding ランタイム
または古い STT エイリアスのどちらかを使用できます:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

注記:

- `api.runtime.mediaUnderstanding.*` は、
  画像/音声/動画理解のための推奨される共有サーフェスです。
- コアの media-understanding 音声設定（`tools.media.audio`）とプロバイダのフォールバック順序を使います。
- 文字起こし出力が生成されない場合（たとえば入力がスキップ/未サポートの場合）は `{ text: undefined }` を返します。
- `api.runtime.stt.transcribeAudioFile(...)` は互換性エイリアスとして残っています。

Plugin は `api.runtime.subagent` を通じてバックグラウンドのサブエージェント実行も開始できます:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

注記:

- `provider` と `model` は、セッションに永続化される変更ではなく、実行ごとのオーバーライドです。
- OpenClaw は、信頼された呼び出し元に対してのみ、それらのオーバーライドフィールドを尊重します。
- Plugin 所有のフォールバック実行では、オペレーターが `plugins.entries.<id>.subagent.allowModelOverride: true` で明示的にオプトインする必要があります。
- 信頼された Plugin を特定の正規 `provider/model` ターゲットに制限するには `plugins.entries.<id>.subagent.allowedModels` を使い、任意のターゲットを明示的に許可するには `"*"` を使います。
- 信頼されていない Plugin のサブエージェント実行も動作しますが、オーバーライド要求は黙ってフォールバックされるのではなく拒否されます。

Web 検索については、Plugin はエージェントツール配線へ直接到達する代わりに、
共有ランタイムヘルパーを利用できます:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Plugin は `api.registerWebSearchProvider(...)` を通じて
Web 検索プロバイダも登録できます。

注記:

- プロバイダ選択、資格情報解決、共有リクエストセマンティクスはコアに維持してください。
- ベンダー固有の検索トランスポートには Web 検索プロバイダを使ってください。
- `api.runtime.webSearch.*` は、エージェントツールラッパーに依存せずに検索動作を必要とする機能/チャネル Plugin 向けの推奨共有サーフェスです。

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: 設定された画像生成プロバイダチェーンを使って画像を生成します。
- `listProviders(...)`: 利用可能な画像生成プロバイダとそのケイパビリティを一覧表示します。

## Gateway HTTP ルート

Plugin は `api.registerHttpRoute(...)` で HTTP エンドポイントを公開できます。

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

ルートフィールド:

- `path`: gateway HTTP サーバー配下のルートパス。
- `auth`: 必須。通常の gateway 認証を要求するには `"gateway"` を使い、Plugin 管理の認証/Webhook 検証には `"plugin"` を使います。
- `match`: 任意。`"exact"`（デフォルト）または `"prefix"`。
- `replaceExisting`: 任意。同じ Plugin が自分自身の既存ルート登録を置き換えることを許可します。
- `handler`: ルートがリクエストを処理した場合は `true` を返します。

注記:

- `api.registerHttpHandler(...)` は削除されており、Plugin 読み込みエラーの原因になります。代わりに `api.registerHttpRoute(...)` を使ってください。
- Plugin ルートでは `auth` を明示的に宣言する必要があります。
- 正確に同じ `path + match` の競合は、`replaceExisting: true` でない限り拒否され、ある Plugin が別の Plugin のルートを置き換えることはできません。
- `auth` レベルが異なる重複ルートは拒否されます。`exact`/`prefix` のフォールスルーチェーンは同じ auth レベルの中だけに保ってください。
- `auth: "plugin"` ルートは、オペレーターのランタイムスコープを自動では受け取り**ません**。これは、特権的な Gateway ヘルパー呼び出しではなく、Plugin 管理の Webhook/署名検証用です。
- `auth: "gateway"` ルートは Gateway リクエストのランタイムスコープ内で実行されますが、そのスコープは意図的に保守的です:
  - 共有シークレット bearer 認証（`gateway.auth.mode = "token"` / `"password"`）では、呼び出し元が `x-openclaw-scopes` を送っても、plugin-route のランタイムスコープは `operator.write` に固定されます
  - 信頼された ID 付き HTTP モード（たとえば `trusted-proxy` や、プライベート ingress 上の `gateway.auth.mode = "none"`）では、`x-openclaw-scopes` ヘッダーが明示的に存在する場合にのみそれを尊重します
  - それらの ID 付き plugin-route リクエストで `x-openclaw-scopes` が存在しない場合、ランタイムスコープは `operator.write` にフォールバックします
- 実践的なルール: gateway-auth の Plugin ルートを暗黙の管理者サーフェスだと想定しないでください。ルートに管理者専用動作が必要なら、ID 付き認証モードを要求し、明示的な `x-openclaw-scopes` ヘッダーコントラクトを文書化してください。

## Plugin SDK インポートパス

Plugin を作成するときは、一枚岩の `openclaw/plugin-sdk` インポートではなく
SDK サブパスを使ってください:

- Plugin 登録プリミティブには `openclaw/plugin-sdk/plugin-entry`。
- 汎用の共有 Plugin 向けコントラクトには `openclaw/plugin-sdk/core`。
- ルート `openclaw.json` Zod スキーマ
  エクスポート（`OpenClawSchema`）には `openclaw/plugin-sdk/config-schema`。
- 共有 setup/auth/reply/Webhook
  配線には、`openclaw/plugin-sdk/channel-setup`、
  `openclaw/plugin-sdk/setup-runtime`、
  `openclaw/plugin-sdk/setup-adapter-runtime`、
  `openclaw/plugin-sdk/setup-tools`、
  `openclaw/plugin-sdk/channel-pairing`、
  `openclaw/plugin-sdk/channel-contract`、
  `openclaw/plugin-sdk/channel-feedback`、
  `openclaw/plugin-sdk/channel-inbound`、
  `openclaw/plugin-sdk/channel-lifecycle`、
  `openclaw/plugin-sdk/channel-reply-pipeline`、
  `openclaw/plugin-sdk/command-auth`、
  `openclaw/plugin-sdk/secret-input`、および
  `openclaw/plugin-sdk/webhook-ingress` のような安定したチャネルプリミティブを使います。
  `channel-inbound` は、debounce、mention マッチング、
  受信 mention-policy ヘルパー、envelope フォーマット、および受信 envelope
  コンテキストヘルパーの共有ホームです。
  `channel-setup` は狭い optional-install setup シームです。
  `setup-runtime` は、`setupEntry` /
  遅延起動で使われるランタイム安全な setup サーフェスであり、
  import-safe な setup patch adapter を含みます。
  `setup-adapter-runtime` は env 認識の account-setup adapter シームです。
  `setup-tools` は小さな CLI/archive/docs ヘルパーシーム（`formatCliCommand`、
  `detectBinary`、`extractArchive`、`resolveBrewExecutable`、`formatDocsLink`、
  `CONFIG_DIR`）です。
- 共有ランタイム/設定ヘルパーには、
  `openclaw/plugin-sdk/channel-config-helpers`、
  `openclaw/plugin-sdk/allow-from`、
  `openclaw/plugin-sdk/channel-config-schema`、
  `openclaw/plugin-sdk/telegram-command-config`、
  `openclaw/plugin-sdk/channel-policy`、
  `openclaw/plugin-sdk/approval-gateway-runtime`、
  `openclaw/plugin-sdk/approval-handler-adapter-runtime`、
  `openclaw/plugin-sdk/approval-handler-runtime`、
  `openclaw/plugin-sdk/approval-runtime`、
  `openclaw/plugin-sdk/config-runtime`、
  `openclaw/plugin-sdk/infra-runtime`、
  `openclaw/plugin-sdk/agent-runtime`、
  `openclaw/plugin-sdk/lazy-runtime`、
  `openclaw/plugin-sdk/reply-history`、
  `openclaw/plugin-sdk/routing`、
  `openclaw/plugin-sdk/status-helpers`、
  `openclaw/plugin-sdk/text-runtime`、
  `openclaw/plugin-sdk/runtime-store`、および
  `openclaw/plugin-sdk/directory-runtime` のようなドメインサブパスを使います。
  `telegram-command-config` は、Telegram カスタム
  コマンドの正規化/検証のための狭い公開シームであり、バンドルされた
  Telegram コントラクトサーフェスが一時的に利用できない場合でも引き続き利用可能です。
  `text-runtime` は共有の text/markdown/logging シームであり、
  assistant-visible-text の除去、markdown のレンダリング/チャンク化ヘルパー、redaction
  ヘルパー、directive-tag ヘルパー、安全なテキストユーティリティを含みます。
- 承認固有のチャネルシームでは、Plugin 上の1つの `approvalCapability`
  コントラクトを優先すべきです。そうすればコアは、承認の auth、配信、レンダリング、
  ネイティブルーティング、遅延ネイティブハンドラ動作を、無関係な Plugin フィールドへ承認動作を混在させる代わりに、
  その1つのケイパビリティを通じて読み取れます。
- `openclaw/plugin-sdk/channel-runtime` は非推奨であり、古い Plugin 向けの
  互換 shim としてのみ残っています。新しいコードでは、代わりにより狭い
  汎用プリミティブをインポートすべきであり、リポジトリコードでもこの
  shim への新規インポートを追加すべきではありません。
- バンドル拡張の内部実装は非公開のままです。外部 Plugin は
  `openclaw/plugin-sdk/*` サブパスのみを使うべきです。OpenClaw のコア/テストコードは、
  `index.js`、`api.js`、
  `runtime-api.js`、`setup-entry.js`、`login-qr-api.js` のような狭くスコープされたファイルなど、
  Plugin package ルート配下のリポジトリ公開エントリポイントを使えます。
  コアや別の拡張から、Plugin package の `src/*` を決してインポートしないでください。
- リポジトリエントリポイントの分割:
  `<plugin-package-root>/api.js` はヘルパー/型の barrel、
  `<plugin-package-root>/runtime-api.js` はランタイム専用の barrel、
  `<plugin-package-root>/index.js` はバンドルされた Plugin エントリ、
  `<plugin-package-root>/setup-entry.js` は setup Plugin エントリです。
- 現在のバンドルプロバイダの例:
  - Anthropic は、`wrapAnthropicProviderStream`、ベータヘッダーヘルパー、
    `service_tier` 解析のような Claude ストリームヘルパーに `api.js` / `contract-api.js` を使います。
  - OpenAI は、プロバイダビルダー、デフォルトモデルヘルパー、リアルタイム
    プロバイダビルダーに `api.js` を使います。
  - OpenRouter は、プロバイダビルダーに加えてオンボーディング/設定
    ヘルパーに `api.js` を使い、`register.runtime.js` は依然としてリポジトリ内利用向けに
    汎用 `plugin-sdk/provider-stream` ヘルパーを再エクスポートできます。
- ファサード読み込みされる公開エントリポイントは、存在する場合はアクティブなランタイム設定スナップショットを優先し、
  OpenClaw がまだランタイムスナップショットを提供していない場合は、ディスク上の解決済み設定ファイルにフォールバックします。
- 汎用の共有プリミティブは、引き続き推奨される公開 SDK コントラクトです。
  ただし、バンドルされたチャネルブランド付きヘルパーシームの小規模な予約済み互換セットは
  依然として存在します。これらは、新しいサードパーティのインポート先ではなく、
  バンドル保守/互換性用のシームとして扱ってください。新しいクロスチャネルコントラクトは、
  引き続き汎用 `plugin-sdk/*` サブパスまたは Plugin ローカルの `api.js` /
  `runtime-api.js` barrel に置くべきです。

互換性に関する注記:

- 新しいコードではルートの `openclaw/plugin-sdk` barrel を避けてください。
- まずは狭く安定したプリミティブを優先してください。新しい
  setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool サブパスが、新しい
  バンドル Plugin と外部 Plugin 作業向けの意図されたコントラクトです。
  ターゲットのパース/マッチングは `openclaw/plugin-sdk/channel-targets` に置くべきです。
  message action ゲートと reaction message-id ヘルパーは
  `openclaw/plugin-sdk/channel-actions` に置くべきです。
- バンドル拡張固有の helper barrel は、デフォルトでは安定していません。ある
  ヘルパーがバンドル拡張だけに必要なら、それを
  `openclaw/plugin-sdk/<extension>` に昇格させるのではなく、
  その拡張のローカルな `api.js` または `runtime-api.js` シームの背後に置いてください。
- 新しい共有 helper シームは、チャネル名付きではなく汎用であるべきです。共有ターゲット
  パースは `openclaw/plugin-sdk/channel-targets` に置くべきであり、チャネル固有の
  内部実装は所有する Plugin のローカル `api.js` または `runtime-api.js`
  シームの背後に留めてください。
- `image-generation`、
  `media-understanding`、`speech` のようなケイパビリティ固有サブパスは、
  現在バンドル/ネイティブ Plugin がそれらを使っているため存在します。それらが存在すること自体は、
  エクスポートされたすべてのヘルパーが長期的に固定された外部コントラクトであることを意味しません。

## Message ツールスキーマ

Plugin は、チャネル固有の `describeMessageTool(...)` スキーマへの寄与を
所有すべきです。プロバイダ固有のフィールドは、共有コアではなく Plugin 内に置いてください。

共有で移植可能なスキーマ断片には、
`openclaw/plugin-sdk/channel-actions` からエクスポートされる汎用ヘルパーを再利用してください:

- ボタングリッド形式のペイロードには `createMessageToolButtonsSchema()`
- 構造化カード形式のペイロードには `createMessageToolCardSchema()`

あるスキーマ形状が1つのプロバイダにしか意味を持たない場合は、それを共有 SDK に昇格させるのではなく、
その Plugin 自身のソース内で定義してください。

## チャネルターゲット解決

チャネル Plugin は、チャネル固有のターゲットセマンティクスを所有すべきです。共有
outbound ホストは汎用のままにし、プロバイダルールには messaging adapter サーフェスを使ってください:

- `messaging.inferTargetChatType({ to })` は、正規化されたターゲットを
  ディレクトリ参照前に `direct`、`group`、`channel` のどれとして扱うべきかを判断します。
- `messaging.targetResolver.looksLikeId(raw, normalized)` は、ある入力を
  ディレクトリ検索ではなく id らしい解決へ直接進めるべきかどうかをコアへ伝えます。
- `messaging.targetResolver.resolveTarget(...)` は、正規化後または
  ディレクトリミス後に、コアが最終的なプロバイダ所有解決を必要とするときの Plugin 側フォールバックです。
- `messaging.resolveOutboundSessionRoute(...)` は、ターゲット解決後のプロバイダ固有
  セッションルート構築を所有します。

推奨される分割:

- ピア/グループ検索前に行うべきカテゴリ判断には `inferTargetChatType` を使う
- 「これを明示的/ネイティブなターゲット id として扱う」判定には `looksLikeId` を使う
- `resolveTarget` は、広範なディレクトリ検索ではなく、プロバイダ固有の正規化フォールバックに使う
- chat id、thread id、JID、handle、room id のようなプロバイダネイティブ id は、
  汎用 SDK フィールドではなく、`target` 値またはプロバイダ固有パラメータ内に保持する

## 設定ベースのディレクトリ

設定からディレクトリエントリを導出する Plugin は、そのロジックを Plugin 内に保持し、
`openclaw/plugin-sdk/directory-runtime` の共有ヘルパーを再利用すべきです。

これは、チャネルが次のような設定ベースのピア/グループを必要とするときに使います:

- allowlist 駆動の DM ピア
- 設定済みのチャネル/グループマップ
- アカウント単位の静的ディレクトリフォールバック

`directory-runtime` の共有ヘルパーは、汎用操作のみを扱います:

- クエリフィルタリング
- 上限適用
- 重複除去/正規化ヘルパー
- `ChannelDirectoryEntry[]` の構築

チャネル固有のアカウント検査と id 正規化は、Plugin 実装内に留めてください。

## プロバイダカタログ

プロバイダ Plugin は、
`registerProvider({ catalog: { run(...) { ... } } })` を使って推論用モデルカタログを定義できます。

`catalog.run(...)` は、OpenClaw が
`models.providers` に書き込むのと同じ形を返します:

- 1つのプロバイダエントリに対しては `{ provider }`
- 複数のプロバイダエントリに対しては `{ providers }`

Plugin がプロバイダ固有の model id、base URL
デフォルト、または認証ゲート付きモデルメタデータを所有する場合は、`catalog` を使ってください。

`catalog.order` は、Plugin のカタログが OpenClaw の
組み込み暗黙プロバイダに対していつマージされるかを制御します:

- `simple`: プレーンな API キーまたは env 駆動プロバイダ
- `profile`: auth profile が存在するときに現れるプロバイダ
- `paired`: 複数の関連プロバイダエントリを合成するプロバイダ
- `late`: 他の暗黙プロバイダの後の最終パス

後から来たプロバイダがキー競合時に勝つため、Plugin は同じ provider id を持つ
組み込みプロバイダエントリを意図的に上書きできます。

互換性:

- `discovery` はレガシーエイリアスとして引き続き動作します
- `catalog` と `discovery` の両方が登録されている場合、OpenClaw は `catalog` を使います

## 読み取り専用のチャネル検査

Plugin がチャネルを登録する場合は、`resolveAccount(...)` とあわせて
`plugin.config.inspectAccount(cfg, accountId)` の実装を優先してください。

理由:

- `resolveAccount(...)` はランタイム経路です。資格情報が
  完全に具体化されている前提でよく、必要なシークレットが不足していれば即座に失敗して構いません。
- `openclaw status`、`openclaw status --all`、
  `openclaw channels status`、`openclaw channels resolve`、および doctor/config
  修復フローのような読み取り専用コマンド経路では、設定を説明するだけのために
  ランタイム資格情報を具体化する必要があるべきではありません。

推奨される `inspectAccount(...)` の動作:

- 説明的なアカウント状態のみを返す。
- `enabled` と `configured` を維持する。
- 関連する場合は、資格情報のソース/ステータスフィールドを含める。たとえば:
  - `tokenSource`、`tokenStatus`
  - `botTokenSource`、`botTokenStatus`
  - `appTokenSource`、`appTokenStatus`
  - `signingSecretSource`、`signingSecretStatus`
- 読み取り専用の利用可否を報告するだけなら、生のトークン値を返す必要はありません。
  ステータス系コマンドには `tokenStatus: "available"`（および対応するソースフィールド）を返せば十分です。
- 資格情報が SecretRef 経由で設定されているが、現在のコマンド経路では利用できない場合は
  `configured_unavailable` を使います。

これにより、読み取り専用コマンドは、クラッシュしたり、アカウントを未設定と誤報したりする代わりに、
「設定済みだがこのコマンド経路では利用不可」と報告できます。

## Package pack

Plugin ディレクトリには、`openclaw.extensions` を持つ `package.json` を含められます:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

各エントリは1つの Plugin になります。pack が複数の拡張を列挙している場合、Plugin id は
`name/<fileBase>` になります。

Plugin が npm 依存関係をインポートする場合は、そのディレクトリ内に
`node_modules` が利用できるようインストールしてください（`npm install` / `pnpm install`）。

セキュリティガードレール: すべての `openclaw.extensions` エントリは、symlink 解決後も
Plugin ディレクトリ内に留まっている必要があります。package ディレクトリ外へ
エスケープするエントリは拒否されます。

セキュリティ上の注意: `openclaw plugins install` は Plugin 依存関係を
`npm install --omit=dev --ignore-scripts` でインストールします（ライフサイクルスクリプトなし、本番時に dev dependencies なし）。Plugin の依存関係
ツリーは「pure JS/TS」に保ち、`postinstall` ビルドを必要とする package は避けてください。

任意: `openclaw.setupEntry` は軽量な setup 専用モジュールを指せます。
OpenClaw が無効なチャネル Plugin の setup サーフェスを必要とする場合、または
チャネル Plugin が有効でもまだ未設定の場合、完全な Plugin エントリの代わりに `setupEntry`
を読み込みます。これにより、メイン Plugin エントリがツール、フック、または他のランタイム専用
コードも配線している場合でも、起動と setup を軽量に保てます。

任意: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
を使うと、チャネルがすでに設定済みであっても、gateway の
listen 前起動フェーズ中にチャネル Plugin を同じ `setupEntry` 経路へオプトインできます。

これは、gateway が listen を開始する前に存在していなければならない起動サーフェスを
`setupEntry` が完全にカバーしている場合にのみ使ってください。実際には、setup entry が
起動時に依存するすべてのチャネル所有ケイパビリティを登録する必要があります。たとえば:

- チャネル登録そのもの
- gateway が listen を開始する前に利用可能でなければならない HTTP ルート
- 同じ時間帯に存在していなければならない Gateway メソッド、ツール、またはサービス

完全なエントリが依然として何らかの必須起動ケイパビリティを所有している場合は、
このフラグを有効にしないでください。デフォルト動作のままにして、
OpenClaw に起動中に完全なエントリを読み込ませてください。

バンドルされたチャネルは、完全なチャネルランタイムが読み込まれる前にコアが参照できる、
setup 専用のコントラクトサーフェスヘルパーも公開できます。現在の setup
昇格サーフェスは次のとおりです:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

コアは、完全な Plugin エントリを読み込まずに、レガシーな単一アカウントチャネル設定を
`channels.<id>.accounts.*` へ昇格する必要があるときにそのサーフェスを使います。
Matrix は現在のバンドル例です。名前付きアカウントがすでに存在する場合、
認証/bootstrap キーだけを名前付き昇格アカウントへ移し、
常に `accounts.default` を作るのではなく、設定済みの非標準なデフォルトアカウントキーを保持できます。

これらの setup patch adapter により、バンドルされたコントラクトサーフェスのディスカバリは遅延されたままです。
import 時間は軽く保たれ、昇格サーフェスはモジュール import 時に
バンドルチャネルの起動へ再突入する代わりに、最初に使われたときだけ読み込まれます。

それらの起動サーフェスに Gateway RPC メソッドが含まれる場合は、
Plugin 固有の接頭辞に保ってください。コアの管理名前空間（`config.*`、
`exec.approvals.*`、`wizard.*`、`update.*`）は予約済みであり、Plugin がより狭いスコープを要求しても、
常に `operator.admin` に解決されます。

例:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### チャネルカタログメタデータ

チャネル Plugin は `openclaw.channel` を通じて setup/ディスカバリ用メタデータを、
`openclaw.install` を通じて install ヒントを公開できます。これにより、コアのカタログはデータフリーに保たれます。

例:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Nextcloud Talk Webhook bot によるセルフホスト型チャット。",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

最小例以外で有用な `openclaw.channel` フィールド:

- `detailLabel`: より豊かなカタログ/ステータスサーフェス向けの二次ラベル
- `docsLabel`: ドキュメントリンク用のリンクテキストを上書き
- `preferOver`: このカタログエントリが優先順位で上回るべき、低優先度の Plugin/チャネル id
- `selectionDocsPrefix`、`selectionDocsOmitLabel`、`selectionExtras`: 選択サーフェス上の文言制御
- `markdownCapable`: outbound フォーマット判断のために、そのチャネルを markdown 対応として示す
- `exposure.configured`: `false` に設定すると、設定済みチャネル一覧サーフェスからそのチャネルを隠す
- `exposure.setup`: `false` に設定すると、対話型 setup/configure picker からそのチャネルを隠す
- `exposure.docs`: ドキュメントナビゲーションサーフェス上で、そのチャネルを internal/private として示す
- `showConfigured` / `showInSetup`: 互換性のためにレガシーエイリアスも受け付けるが、`exposure` を優先する
- `quickstartAllowFrom`: 標準クイックスタート `allowFrom` フローへそのチャネルをオプトインする
- `forceAccountBinding`: アカウントが1つしかなくても明示的なアカウントバインディングを要求する
- `preferSessionLookupForAnnounceTarget`: announce ターゲット解決時にセッション参照を優先する

OpenClaw は **外部チャネルカタログ**（たとえば MPM
レジストリエクスポート）もマージできます。次のいずれかに JSON ファイルを配置してください:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

または、`OPENCLAW_PLUGIN_CATALOG_PATHS`（または `OPENCLAW_MPM_CATALOG_PATHS`）で、
1つ以上の JSON ファイルを指定してください（カンマ/セミコロン/`PATH` 区切り）。
各ファイルには `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }` を含める必要があります。パーサは `"entries"` キーのレガシーエイリアスとして `"packages"` または `"plugins"` も受け付けます。

## コンテキストエンジン Plugin

コンテキストエンジン Plugin は、取り込み、組み立て、
Compaction のためのセッションコンテキストオーケストレーションを所有します。Plugin から
`api.registerContextEngine(id, factory)` で登録し、アクティブなエンジンは
`plugins.slots.contextEngine` で選択します。

これは、Plugin が単に memory 検索やフックを追加するだけではなく、デフォルトのコンテキスト
パイプラインを置き換えたり拡張したりする必要がある場合に使います。

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

エンジンが compaction アルゴリズムを**所有しない**場合でも、`compact()` は
実装したままにし、明示的に委譲してください:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## 新しいケイパビリティの追加

Plugin が現在の API に当てはまらない振る舞いを必要とする場合は、非公開の直接到達で
Plugin システムを迂回しないでください。不足しているケイパビリティを追加してください。

推奨される手順:

1. コアコントラクトを定義する
   コアが所有すべき共有動作を決めます: ポリシー、フォールバック、設定マージ、
   ライフサイクル、チャネル向けセマンティクス、ランタイムヘルパー形状。
2. 型付きの Plugin 登録/ランタイムサーフェスを追加する
   最小限で有用な型付きケイパビリティサーフェスで `OpenClawPluginApi` や
   `api.runtime` を拡張します。
3. コア + チャネル/機能コンシューマを接続する
   チャネルと機能 Plugin は、新しいケイパビリティをコア経由で利用すべきであり、
   ベンダー実装を直接インポートすべきではありません。
4. ベンダー実装を登録する
   その後、ベンダー Plugin がそのケイパビリティに対してバックエンドを登録します。
5. コントラクトカバレッジを追加する
   所有権と登録形状が時間とともに明示的に保たれるよう、テストを追加します。

これが、OpenClaw が1つの
プロバイダの世界観にハードコードされることなく、意見を持ち続ける方法です。具体的な
ファイルチェックリストと実例については [Capability Cookbook](/ja-JP/plugins/architecture)
を参照してください。

### ケイパビリティのチェックリスト

新しいケイパビリティを追加するとき、実装は通常これらの
サーフェスをまとめて触るべきです:

- `src/<capability>/types.ts` のコアコントラクト型
- `src/<capability>/runtime.ts` のコアランナー/ランタイムヘルパー
- `src/plugins/types.ts` の Plugin API 登録サーフェス
- `src/plugins/registry.ts` の Plugin レジストリ配線
- 機能/チャネル
  Plugin がそれを使う必要がある場合の `src/plugins/runtime/*` の Plugin ランタイム公開
- `src/test-utils/plugin-registration.ts` の capture/test ヘルパー
- `src/plugins/contracts/registry.ts` の所有権/コントラクトアサーション
- `docs/` のオペレーター/Plugin ドキュメント

これらのサーフェスのいずれかが欠けている場合、それは通常、そのケイパビリティが
まだ完全には統合されていない兆候です。

### ケイパビリティテンプレート

最小パターン:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

コントラクトテストパターン:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

これにより、ルールはシンプルに保たれます:

- コアがケイパビリティコントラクト + オーケストレーションを所有する
- ベンダー Plugin がベンダー実装を所有する
- 機能/チャネル Plugin がランタイムヘルパーを利用する
- コントラクトテストが所有権を明示的に保つ
