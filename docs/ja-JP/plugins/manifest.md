---
read_when:
    - OpenClaw Plugin を構築している場合
    - Plugin の設定スキーマを提供する必要がある場合、または Plugin の検証エラーをデバッグする必要がある場合
summary: Plugin マニフェスト + JSON スキーマ要件（厳格な設定検証）
title: Plugin マニフェスト
x-i18n:
    generated_at: "2026-04-12T23:28:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 93b57c7373e4ccd521b10945346db67991543bd2bed4cc8b6641e1f215b48579
    source_path: plugins/manifest.md
    workflow: 15
---

# Plugin マニフェスト（`openclaw.plugin.json`）

このページは、**ネイティブ OpenClaw Plugin マニフェスト**のみを対象としています。

互換バンドルレイアウトについては、[Plugin bundles](/ja-JP/plugins/bundles) を参照してください。

互換バンドル形式では、異なるマニフェストファイルを使用します。

- Codex バンドル: `.codex-plugin/plugin.json`
- Claude バンドル: `.claude-plugin/plugin.json` またはマニフェストなしのデフォルト Claude コンポーネントレイアウト
- Cursor バンドル: `.cursor-plugin/plugin.json`

OpenClaw はそれらのバンドルレイアウトも自動検出しますが、ここで説明する `openclaw.plugin.json` スキーマに対しては検証されません。

互換バンドルについては、OpenClaw は現在、バンドルメタデータに加えて、宣言された skill ルート、Claude コマンドルート、Claude バンドルの `settings.json` デフォルト、Claude バンドルの LSP デフォルト、および、そのレイアウトが OpenClaw ランタイムの期待に一致する場合の対応 hook pack を読み取ります。

すべてのネイティブ OpenClaw Plugin は、**plugin ルート**に `openclaw.plugin.json` ファイルを必ず含める必要があります。OpenClaw はこのマニフェストを使用して、**plugin コードを実行せずに**設定を検証します。マニフェストが存在しない、または無効な場合は plugin エラーとして扱われ、設定検証をブロックします。

完全な plugin システムガイドについては、[Plugins](/ja-JP/tools/plugin) を参照してください。
ネイティブ capability モデルと現在の外部互換性ガイダンスについては、
[Capability model](/ja-JP/plugins/architecture#public-capability-model) を参照してください。

## このファイルの役割

`openclaw.plugin.json` は、OpenClaw が plugin コードを読み込む前に読むメタデータです。

用途:

- plugin の識別情報
- 設定検証
- plugin ランタイムを起動しなくても利用できる認証およびオンボーディングのメタデータ
- ランタイム読み込み前にコントロールプレーンの各サーフェスが確認できる軽量なアクティベーションヒント
- ランタイム読み込み前にセットアップ/オンボーディングの各サーフェスが確認できる軽量なセットアップ記述子
- plugin ランタイム読み込み前に解決されるべきエイリアスおよび自動有効化メタデータ
- plugin ランタイム読み込み前に plugin を自動アクティベートすべきモデルファミリー所有権の簡略メタデータ
- バンドル互換配線およびコントラクト網羅に使用される静的 capability 所有スナップショット
- ランタイムを読み込まずにカタログおよび検証サーフェスへマージされるべきチャネル固有の設定メタデータ
- 設定 UI ヒント

用途ではないもの:

- ランタイム動作の登録
- コードのエントリーポイント宣言
- npm install メタデータ

これらは plugin コードと `package.json` に属します。

## 最小例

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## リッチな例

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter provider plugin",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "cliBackends": ["openrouter-cli"],
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "channelEnvVars": {
    "openrouter-chatops": ["OPENROUTER_CHATOPS_TOKEN"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API key",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API key",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## 最上位フィールドのリファレンス

| フィールド | 必須 | 型 | 意味 |
| ----------------------------------- | -------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id` | はい | `string` | 正式な plugin ID。これは `plugins.entries.<id>` で使用される ID です。 |
| `configSchema` | はい | `object` | この plugin の設定に対するインライン JSON Schema。 |
| `enabledByDefault` | いいえ | `true` | バンドルされた plugin をデフォルトで有効としてマークします。省略するか、`true` 以外の値を設定すると、その plugin はデフォルトで無効のままになります。 |
| `legacyPluginIds` | いいえ | `string[]` | この正式な plugin ID に正規化されるレガシー ID。 |
| `autoEnableWhenConfiguredProviders` | いいえ | `string[]` | 認証、設定、またはモデル参照でそれらが言及されたときに、この plugin を自動有効化すべき provider ID。 |
| `kind` | いいえ | `"memory"` \| `"context-engine"` | `plugins.slots.*` で使用される排他的な plugin 種別を宣言します。 |
| `channels` | いいえ | `string[]` | この plugin が所有するチャネル ID。検出と設定検証に使用されます。 |
| `providers` | いいえ | `string[]` | この plugin が所有する provider ID。 |
| `modelSupport` | いいえ | `object` | ランタイム前に plugin を自動ロードするために使用される、マニフェスト所有のモデルファミリー簡略メタデータ。 |
| `cliBackends` | いいえ | `string[]` | この plugin が所有する CLI 推論バックエンド ID。明示的な設定参照からの起動時自動アクティベーションに使用されます。 |
| `commandAliases` | いいえ | `object[]` | ランタイム読み込み前に plugin を認識した設定および CLI 診断を生成すべき、この plugin が所有するコマンド名。 |
| `providerAuthEnvVars` | いいえ | `Record<string, string[]>` | OpenClaw が plugin コードを読み込まずに確認できる、軽量な provider 認証用環境変数メタデータ。 |
| `providerAuthAliases` | いいえ | `Record<string, string>` | 認証検索のために別の provider ID を再利用すべき provider ID。たとえば、ベース provider の API キーと認証プロファイルを共有する coding provider などです。 |
| `channelEnvVars` | いいえ | `Record<string, string[]>` | OpenClaw が plugin コードを読み込まずに確認できる、軽量なチャネル環境変数メタデータ。env 駆動のチャネルセットアップや、汎用の起動/設定ヘルパーが参照すべき認証サーフェスに使用します。 |
| `providerAuthChoices` | いいえ | `object[]` | オンボーディングピッカー、優先 provider 解決、単純な CLI フラグ配線のための軽量な認証選択メタデータ。 |
| `activation` | いいえ | `object` | provider、command、channel、route、および capability トリガー読み込みのための軽量なアクティベーションヒント。メタデータのみであり、実際の動作は引き続き plugin ランタイムが所有します。 |
| `setup` | いいえ | `object` | 検出およびセットアップの各サーフェスが plugin ランタイムを読み込まずに確認できる、軽量なセットアップ/オンボーディング記述子。 |
| `contracts` | いいえ | `object` | speech、realtime transcription、realtime voice、media-understanding、image-generation、music-generation、video-generation、web-fetch、web search、およびツール所有権のための静的なバンドル capability スナップショット。 |
| `channelConfigs` | いいえ | `Record<string, object>` | ランタイム読み込み前に検出および検証サーフェスへマージされる、マニフェスト所有のチャネル設定メタデータ。 |
| `skills` | いいえ | `string[]` | plugin ルートからの相対パスで指定する、読み込む Skills ディレクトリ。 |
| `name` | いいえ | `string` | 人が読むための plugin 名。 |
| `description` | いいえ | `string` | plugin サーフェスに表示される短い要約。 |
| `version` | いいえ | `string` | 情報表示用の plugin バージョン。 |
| `uiHints` | いいえ | `Record<string, object>` | 設定フィールド用の UI ラベル、プレースホルダー、および機密性ヒント。 |

## `providerAuthChoices` リファレンス

各 `providerAuthChoices` エントリは、1 つのオンボーディングまたは認証の選択肢を記述します。
OpenClaw は provider ランタイムを読み込む前にこれを読み取ります。

| フィールド | 必須 | 型 | 意味 |
| --------------------- | -------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `provider` | はい | `string` | この選択肢が属する provider ID。 |
| `method` | はい | `string` | ディスパッチ先の認証方式 ID。 |
| `choiceId` | はい | `string` | オンボーディングおよび CLI フローで使用される安定した認証選択肢 ID。 |
| `choiceLabel` | いいえ | `string` | ユーザー向けラベル。省略した場合、OpenClaw は `choiceId` にフォールバックします。 |
| `choiceHint` | いいえ | `string` | ピッカー用の短い補助テキスト。 |
| `assistantPriority` | いいえ | `number` | アシスタント主導のインタラクティブピッカーでは、値が小さいほど先に並びます。 |
| `assistantVisibility` | いいえ | `"visible"` \| `"manual-only"` | アシスタントのピッカーではこの選択肢を非表示にしつつ、手動 CLI 選択は引き続き許可します。 |
| `deprecatedChoiceIds` | いいえ | `string[]` | ユーザーをこの置き換え先の選択肢にリダイレクトすべきレガシー選択肢 ID。 |
| `groupId` | いいえ | `string` | 関連する選択肢をグループ化するための任意のグループ ID。 |
| `groupLabel` | いいえ | `string` | そのグループのユーザー向けラベル。 |
| `groupHint` | いいえ | `string` | グループ用の短い補助テキスト。 |
| `optionKey` | いいえ | `string` | 単一フラグの単純な認証フロー用の内部オプションキー。 |
| `cliFlag` | いいえ | `string` | `--openrouter-api-key` のような CLI フラグ名。 |
| `cliOption` | いいえ | `string` | `--openrouter-api-key <key>` のような完全な CLI オプション形式。 |
| `cliDescription` | いいえ | `string` | CLI ヘルプで使用される説明。 |
| `onboardingScopes` | いいえ | `Array<"text-inference" \| "image-generation">` | この選択肢をどのオンボーディングサーフェスに表示するか。省略した場合、デフォルトは `["text-inference"]` です。 |

## `commandAliases` リファレンス

`commandAliases` は、ユーザーが誤って `plugins.allow` に入れたり、ルート CLI コマンドとして実行しようとしたりする可能性がある、plugin 所有のランタイムコマンド名がある場合に使用します。OpenClaw は、このメタデータを使用して、plugin ランタイムコードを import せずに診断を行います。

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| フィールド | 必須 | 型 | 意味 |
| ------------ | -------- | ----------------- | ----------------------------------------------------------------------- |
| `name` | はい | `string` | この plugin に属するコマンド名。 |
| `kind` | いいえ | `"runtime-slash"` | このエイリアスを、ルート CLI コマンドではなくチャットのスラッシュコマンドとしてマークします。 |
| `cliCommand` | いいえ | `string` | 存在する場合、CLI 操作向けに提案する関連ルート CLI コマンド。 |

## `activation` リファレンス

`activation` は、その plugin を後でアクティベートすべきコントロールプレーンイベントを低コストで宣言できる場合に使用します。

このブロックはメタデータのみです。ランタイム動作を登録するものではなく、`register(...)`、`setupEntry`、その他のランタイム/plugin エントリーポイントを置き換えるものでもありません。
現在のコンシューマーはこれを、より広い plugin 読み込みの前の絞り込みヒントとして使用しているため、アクティベーションメタデータが欠けていても、通常は性能面のコストが発生するだけです。レガシーなマニフェスト所有権フォールバックがまだ存在する間は、正しさは変わらないはずです。

```json
{
  "activation": {
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| フィールド | 必須 | 型 | 意味 |
| ---------------- | -------- | ---------------------------------------------------- | ----------------------------------------------------------------- |
| `onProviders` | いいえ | `string[]` | 要求されたときにこの plugin をアクティベートすべき provider ID。 |
| `onCommands` | いいえ | `string[]` | この plugin をアクティベートすべきコマンド ID。 |
| `onChannels` | いいえ | `string[]` | この plugin をアクティベートすべきチャネル ID。 |
| `onRoutes` | いいえ | `string[]` | この plugin をアクティベートすべき route 種別。 |
| `onCapabilities` | いいえ | `Array<"provider" \| "channel" \| "tool" \| "hook">` | コントロールプレーンのアクティベーション計画で使用される大まかな capability ヒント。 |

現在のライブコンシューマー:

- コマンドトリガーの CLI 計画は、レガシーな
  `commandAliases[].cliCommand` または `commandAliases[].name` にフォールバックします
- チャネルトリガーの setup/channel 計画は、明示的なチャネルアクティベーションメタデータがない場合、レガシーな `channels[]`
  所有権にフォールバックします
- provider トリガーの setup/runtime 計画は、明示的な provider
  アクティベーションメタデータがない場合、レガシーな
  `providers[]` および最上位の `cliBackends[]` 所有権にフォールバックします

## `setup` リファレンス

`setup` は、セットアップおよびオンボーディングの各サーフェスが、ランタイム読み込み前に低コストな plugin 所有メタデータを必要とする場合に使用します。

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

最上位の `cliBackends` は引き続き有効で、CLI 推論バックエンドを記述し続けます。`setup.cliBackends` は、メタデータのみを維持すべきコントロールプレーン/セットアップフロー向けの、セットアップ固有の記述子サーフェスです。

存在する場合、`setup.providers` と `setup.cliBackends` は、セットアップ検出のための優先される記述子優先のルックアップサーフェスです。記述子が候補 plugin の絞り込みだけを行い、セットアップにさらに豊富なセットアップ時ランタイムフックが必要な場合は、`requiresRuntime: true` を設定し、フォールバック実行パスとして `setup-api` を維持してください。

セットアップのルックアップでは plugin 所有の `setup-api` コードを実行できるため、正規化された `setup.providers[].id` と `setup.cliBackends[]` の値は、検出された plugin 全体で一意である必要があります。所有権が曖昧な場合は、検出順で勝者を選ぶのではなく、クローズドに失敗します。

### `setup.providers` リファレンス

| フィールド | 必須 | 型 | 意味 |
| ------------- | -------- | ---------- | ------------------------------------------------------------------------------------ |
| `id` | はい | `string` | セットアップまたはオンボーディング中に公開される provider ID。正規化された ID はグローバルに一意に保ってください。 |
| `authMethods` | いいえ | `string[]` | 完全なランタイムを読み込まずにこの provider がサポートするセットアップ/認証方式 ID。 |
| `envVars` | いいえ | `string[]` | plugin ランタイム読み込み前に汎用の setup/status サーフェスが確認できる環境変数。 |

### `setup` フィールド

| フィールド | 必須 | 型 | 意味 |
| ------------------ | -------- | ---------- | --------------------------------------------------------------------------------------------------- |
| `providers` | いいえ | `object[]` | セットアップおよびオンボーディング中に公開される provider セットアップ記述子。 |
| `cliBackends` | いいえ | `string[]` | 記述子優先のセットアップルックアップで使用されるセットアップ時バックエンド ID。正規化された ID はグローバルに一意に保ってください。 |
| `configMigrations` | いいえ | `string[]` | この plugin の setup サーフェスが所有する設定マイグレーション ID。 |
| `requiresRuntime` | いいえ | `boolean` | 記述子ルックアップ後も setup に `setup-api` の実行が必要かどうか。 |

## `uiHints` リファレンス

`uiHints` は、設定フィールド名から小さなレンダリングヒントへのマップです。

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API key",
      "help": "Used for OpenRouter requests",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

各フィールドヒントには次を含めることができます。

| フィールド | 型 | 意味 |
| ------------- | ---------- | --------------------------------------- |
| `label` | `string` | ユーザー向けのフィールドラベル。 |
| `help` | `string` | 短い補助テキスト。 |
| `tags` | `string[]` | 任意の UI タグ。 |
| `advanced` | `boolean` | このフィールドを高度な項目としてマークします。 |
| `sensitive` | `boolean` | このフィールドをシークレットまたは機密情報としてマークします。 |
| `placeholder` | `string` | フォーム入力用のプレースホルダーテキスト。 |

## `contracts` リファレンス

`contracts` は、OpenClaw が plugin ランタイムを import せずに読み取れる、静的な capability 所有メタデータにのみ使用してください。

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

各リストは任意です。

| フィールド | 型 | 意味 |
| -------------------------------- | ---------- | -------------------------------------------------------------- |
| `speechProviders` | `string[]` | この plugin が所有する speech provider ID。 |
| `realtimeTranscriptionProviders` | `string[]` | この plugin が所有する realtime-transcription provider ID。 |
| `realtimeVoiceProviders` | `string[]` | この plugin が所有する realtime-voice provider ID。 |
| `mediaUnderstandingProviders` | `string[]` | この plugin が所有する media-understanding provider ID。 |
| `imageGenerationProviders` | `string[]` | この plugin が所有する image-generation provider ID。 |
| `videoGenerationProviders` | `string[]` | この plugin が所有する video-generation provider ID。 |
| `webFetchProviders` | `string[]` | この plugin が所有する web-fetch provider ID。 |
| `webSearchProviders` | `string[]` | この plugin が所有する web-search provider ID。 |
| `tools` | `string[]` | バンドルされたコントラクトチェック用にこの plugin が所有するエージェントツール名。 |

## `channelConfigs` リファレンス

`channelConfigs` は、チャネル Plugin がランタイム読み込み前に低コストな設定メタデータを必要とする場合に使用します。

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix homeserver connection",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

各チャネルエントリには次を含めることができます。

| フィールド | 型 | 意味 |
| ------------- | ------------------------ | ----------------------------------------------------------------------------------------- |
| `schema` | `object` | `channels.<id>` 用の JSON Schema。宣言された各チャネル設定エントリで必須です。 |
| `uiHints` | `Record<string, object>` | そのチャネル設定セクション用の任意の UI ラベル/プレースホルダー/機密性ヒント。 |
| `label` | `string` | ランタイムメタデータの準備ができていない場合に、ピッカーおよび inspect サーフェスへマージされるチャネルラベル。 |
| `description` | `string` | inspect および catalog サーフェス向けの短いチャネル説明。 |
| `preferOver` | `string[]` | 選択サーフェスでこのチャネルが優先されるべき、レガシーまたは低優先度の plugin ID。 |

## `modelSupport` リファレンス

`modelSupport` は、OpenClaw が `gpt-5.4` や `claude-sonnet-4.6` のような短縮モデル ID から、plugin ランタイム読み込み前に provider Plugin を推測すべき場合に使用します。

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw は次の優先順位を適用します。

- 明示的な `provider/model` 参照は、所有する `providers` マニフェストメタデータを使用します
- `modelPatterns` は `modelPrefixes` より優先されます
- 1 つの非バンドル plugin と 1 つのバンドル plugin の両方が一致する場合、非バンドル plugin が優先されます
- 残る曖昧さは、ユーザーまたは設定が provider を指定するまで無視されます

フィールド:

| フィールド | 型 | 意味 |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | 短縮モデル ID に対して `startsWith` で一致させるプレフィックス。 |
| `modelPatterns` | `string[]` | プロファイル接尾辞を除去した後の短縮モデル ID に対して一致させる正規表現ソース。 |

レガシーな最上位 capability キーは非推奨です。`openclaw doctor --fix` を使用して
`speechProviders`、`realtimeTranscriptionProviders`、
`realtimeVoiceProviders`、`mediaUnderstandingProviders`、
`imageGenerationProviders`、`videoGenerationProviders`、
`webFetchProviders`、および `webSearchProviders` を `contracts` 配下へ移動してください。通常の
マニフェスト読み込みでは、これらの最上位フィールドを capability
所有権として扱わなくなっています。

## マニフェストと `package.json` の違い

この 2 つのファイルは異なる役割を持ちます。

| ファイル | 用途 |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | plugin コード実行前に存在している必要がある、検出、設定検証、認証選択メタデータ、および UI ヒント |
| `package.json` | npm メタデータ、依存関係のインストール、およびエントリーポイント、インストール制御、セットアップ、または catalog メタデータに使用される `openclaw` ブロック |

どこにどのメタデータを置くべきか迷った場合は、次のルールを使ってください。

- OpenClaw が plugin コードの読み込み前に知っている必要があるなら、`openclaw.plugin.json` に置きます
- パッケージング、エントリーファイル、または npm install の動作に関するものなら、`package.json` に置きます

### 検出に影響する `package.json` フィールド

一部のランタイム前 plugin メタデータは、意図的に `openclaw.plugin.json` ではなく `package.json` の
`openclaw` ブロックに置かれます。

重要な例:

| フィールド | 意味 |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions` | ネイティブ Plugin エントリーポイントを宣言します。 |
| `openclaw.setupEntry` | オンボーディングおよび遅延チャネル起動時に使用される、軽量なセットアップ専用エントリーポイント。 |
| `openclaw.channel` | ラベル、ドキュメントパス、エイリアス、選択時の文言などの軽量なチャネル catalog メタデータ。 |
| `openclaw.channel.configuredState` | 完全なチャネルランタイムを読み込まずに「env のみのセットアップがすでに存在するか?」に答えられる、軽量な configured-state チェッカーメタデータ。 |
| `openclaw.channel.persistedAuthState` | 完全なチャネルランタイムを読み込まずに「すでに何かにサインインしているか?」に答えられる、軽量な persisted-auth チェッカーメタデータ。 |
| `openclaw.install.npmSpec` / `openclaw.install.localPath` | バンドルされた Plugin および外部公開された Plugin のインストール/更新ヒント。 |
| `openclaw.install.defaultChoice` | 複数のインストール元が利用可能な場合の優先インストールパス。 |
| `openclaw.install.minHostVersion` | `>=2026.3.22` のような semver 下限を使う、サポートされる最小 OpenClaw ホストバージョン。 |
| `openclaw.install.allowInvalidConfigRecovery` | 設定が無効な場合に、限定的なバンドル Plugin の再インストール回復パスを許可します。 |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | 起動中に完全なチャネル Plugin の前に、セットアップ専用チャネルサーフェスを読み込めるようにします。 |

`openclaw.install.minHostVersion` は、インストール時およびマニフェスト
レジストリ読み込み時に適用されます。無効な値は拒否され、有効だがより新しい値は
古いホストではその plugin をスキップします。

`openclaw.install.allowInvalidConfigRecovery` は意図的に限定的です。
任意の壊れた設定をインストール可能にするものではありません。現在は、特定の古いバンドル Plugin
アップグレード失敗、たとえばバンドル Plugin パスの欠落や、その同じバンドル Plugin に対する古い
`channels.<id>` エントリなどから、インストールフローが回復できるようにするだけです。
無関係な設定エラーは引き続きインストールをブロックし、オペレーターを
`openclaw doctor --fix` に誘導します。

`openclaw.channel.persistedAuthState` は、小さなチェッカーモジュール用のパッケージメタデータです。

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

これは、セットアップ、doctor、または configured-state フローが、完全なチャネル Plugin を読み込む前に、低コストな yes/no の認証プローブを必要とする場合に使用します。対象の export は、永続化された状態のみを読む小さな関数にしてください。完全なチャネルランタイム barrel を経由させないでください。

`openclaw.channel.configuredState` も、低コストな env のみの configured チェック用に同じ形式に従います。

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

チャネルが env やその他の小さな非ランタイム入力から configured-state に答えられる場合に使用します。チェックに完全な設定解決または実際のチャネルランタイムが必要な場合は、そのロジックを代わりに plugin `config.hasConfiguredState` hook に置いてください。

## JSON Schema の要件

- **すべての Plugin は JSON Schema を必ず含める必要があります**。設定を受け付けない場合でも同様です。
- 空のスキーマでも問題ありません（例: `{ "type": "object", "additionalProperties": false }`）。
- スキーマはランタイム時ではなく、設定の読み取り/書き込み時に検証されます。

## 検証の動作

- 不明な `channels.*` キーは、チャネル ID が
  plugin マニフェストで宣言されていない限り、**エラー**です。
- `plugins.entries.<id>`、`plugins.allow`、`plugins.deny`、および `plugins.slots.*`
  は、**検出可能な** plugin ID を参照する必要があります。不明な ID は **エラー**です。
- plugin がインストールされていても、マニフェストまたはスキーマが壊れている、あるいは存在しない場合、
  検証は失敗し、Doctor は plugin エラーを報告します。
- plugin 設定が存在していても、その plugin が**無効**の場合、設定は保持され、
  Doctor + ログに **警告** が表示されます。

完全な `plugins.*` スキーマについては、[設定リファレンス](/ja-JP/gateway/configuration) を参照してください。

## 注意

- マニフェストは、ローカルファイルシステムからの読み込みを含め、**ネイティブ OpenClaw Plugin では必須**です。
- ランタイムは引き続き plugin モジュールを別途読み込みます。マニフェストは
  検出 + 検証専用です。
- ネイティブマニフェストは JSON5 で解析されるため、最終的な値がオブジェクトである限り、
  コメント、末尾のカンマ、クォートなしキーを使用できます。
- マニフェストローダーが読み取るのは、文書化されたマニフェストフィールドだけです。ここに
  カスタムの最上位キーを追加するのは避けてください。
- `providerAuthEnvVars` は、認証プローブ、env マーカー検証、および同様の provider 認証サーフェス向けの
  低コストなメタデータパスです。これらは env 名を確認するだけのために plugin
  ランタイムを起動すべきではありません。
- `providerAuthAliases` は、コアにその関係をハードコードすることなく、provider バリアントが別の provider の認証
  env vars、認証プロファイル、設定ベースの認証、および API キーのオンボーディング選択肢を
  再利用できるようにします。
- `channelEnvVars` は、shell-env フォールバック、セットアップ
  プロンプト、および同様のチャネルサーフェス向けの低コストなメタデータパスです。これらは env 名を確認するだけのために plugin
  ランタイムを起動すべきではありません。
- `providerAuthChoices` は、認証選択肢ピッカー、
  `--auth-choice` 解決、優先 provider マッピング、および単純なオンボーディング
  CLI フラグ登録を、provider ランタイム読み込み前に行うための低コストなメタデータパスです。provider コードを必要とするランタイム
  ウィザードのメタデータについては、
  [Provider runtime hooks](/ja-JP/plugins/architecture#provider-runtime-hooks) を参照してください。
- 排他的な plugin 種別は `plugins.slots.*` を通じて選択されます。
  - `kind: "memory"` は `plugins.slots.memory` で選択されます。
  - `kind: "context-engine"` は `plugins.slots.contextEngine`
    で選択されます（デフォルト: 組み込みの `legacy`）。
- `channels`、`providers`、`cliBackends`、および `skills` は、
  plugin がそれらを必要としない場合は省略できます。
- plugin がネイティブモジュールに依存する場合は、ビルド手順と、
  必要なパッケージマネージャーの許可リスト要件（たとえば pnpm の `allow-build-scripts`
  や `pnpm rebuild <package>`）を文書化してください。

## 関連

- [Building Plugins](/ja-JP/plugins/building-plugins) — Plugin のはじめに
- [Plugin Architecture](/ja-JP/plugins/architecture) — 内部アーキテクチャ
- [SDK Overview](/ja-JP/plugins/sdk-overview) — Plugin SDK リファレンス
