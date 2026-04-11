---
read_when:
    - OpenClawプラグインを構築しています
    - プラグイン設定スキーマを提供する必要がある、またはプラグイン検証エラーをデバッグする必要があります
summary: プラグインマニフェスト + JSONスキーマ要件（厳格な設定検証）
title: プラグインマニフェスト
x-i18n:
    generated_at: "2026-04-11T02:46:37Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6b254c121d1eb5ea19adbd4148243cf47339c960442ab1ca0e0bfd52e0154c88
    source_path: plugins/manifest.md
    workflow: 15
---

# プラグインマニフェスト（openclaw.plugin.json）

このページは、**ネイティブなOpenClawプラグインマニフェスト**のみを対象としています。

互換性のあるバンドルレイアウトについては、[Plugin bundles](/ja-JP/plugins/bundles)を参照してください。

互換バンドル形式では、異なるマニフェストファイルを使用します。

- Codexバンドル: `.codex-plugin/plugin.json`
- Claudeバンドル: `.claude-plugin/plugin.json` またはマニフェストなしのデフォルトClaudeコンポーネントレイアウト
- Cursorバンドル: `.cursor-plugin/plugin.json`

OpenClawはそれらのバンドルレイアウトも自動検出しますが、ここで説明する`openclaw.plugin.json`スキーマに対しては検証されません。

互換バンドルについては、OpenClawは現在、レイアウトがOpenClawランタイムの期待に一致している場合に、バンドルメタデータに加えて、宣言されたskill root、Claude command root、Claudeバンドルの`settings.json`デフォルト、ClaudeバンドルのLSPデフォルト、およびサポートされるhook packを読み取ります。

すべてのネイティブOpenClawプラグインは、**plugin root**に`openclaw.plugin.json`ファイルを含める**必要があります**。OpenClawはこのマニフェストを使って、**プラグインコードを実行せずに**設定を検証します。マニフェストが欠落している、または不正な場合はプラグインエラーとして扱われ、設定検証をブロックします。

完全なプラグインシステムガイドについては[Plugins](/ja-JP/tools/plugin)を参照してください。  
ネイティブなcapabilityモデルと現在の外部互換性ガイダンスについては、[Capability model](/ja-JP/plugins/architecture#public-capability-model)を参照してください。

## このファイルの役割

`openclaw.plugin.json` は、OpenClawがプラグインコードを読み込む前に読むメタデータです。

用途:

- プラグインID
- 設定検証
- プラグインランタイムを起動せずに利用可能であるべき認証およびオンボーディングメタデータ
- プラグインランタイムの読み込み前に解決されるべきエイリアスおよび自動有効化メタデータ
- ランタイム読み込み前にプラグインを自動有効化するための短縮形モデルファミリー所有メタデータ
- バンドル互換ワイヤリングおよびコントラクトカバレッジに使われる静的なcapability所有スナップショット
- ランタイムを読み込まずにcatalogおよび検証サーフェスへマージされるべきチャネル固有の設定メタデータ
- 設定UIヒント

使ってはいけない用途:

- ランタイム動作の登録
- コードentrypointの宣言
- npm installメタデータ

これらはプラグインコードと`package.json`に属します。

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
  "description": "OpenRouterプロバイダープラグイン",
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
      "choiceLabel": "OpenRouter APIキー",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter APIキー",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "APIキー",
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

## トップレベルフィールドリファレンス

| Field | Required | Type | What it means |
| ----------------------------------- | -------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id` | Yes | `string` | 正式なplugin idです。これは`plugins.entries.<id>`で使われるidです。 |
| `configSchema` | Yes | `object` | このプラグイン設定用のインラインJSONスキーマです。 |
| `enabledByDefault` | No | `true` | バンドルされたプラグインをデフォルトで有効にすることを示します。省略するか、`true`以外の値を設定すると、プラグインはデフォルトで無効のままになります。 |
| `legacyPluginIds` | No | `string[]` | この正式plugin idに正規化されるレガシーidです。 |
| `autoEnableWhenConfiguredProviders` | No | `string[]` | 認証、設定、またはモデル参照でそれらのprovider idに言及されたときに、このプラグインを自動有効化すべきprovider idです。 |
| `kind` | No | `"memory"` \| `"context-engine"` | `plugins.slots.*`で使われる排他的なplugin kindを宣言します。 |
| `channels` | No | `string[]` | このプラグインが所有するchannel idです。検出と設定検証に使われます。 |
| `providers` | No | `string[]` | このプラグインが所有するprovider idです。 |
| `modelSupport` | No | `object` | ランタイムの前にプラグインを自動読み込みするために使われる、マニフェスト所有の短縮形モデルファミリーメタデータです。 |
| `cliBackends` | No | `string[]` | このプラグインが所有するCLI推論バックエンドidです。明示的な設定参照からの起動時自動有効化に使われます。 |
| `commandAliases` | No | `object[]` | このプラグインが所有するコマンド名で、ランタイム読み込み前にプラグイン対応の設定およびCLI診断を生成すべきものです。 |
| `providerAuthEnvVars` | No | `Record<string, string[]>` | OpenClawがプラグインコードを読み込まずに調べられる、軽量なprovider認証envメタデータです。 |
| `providerAuthAliases` | No | `Record<string, string>` | 認証参照のために別のprovider idを再利用すべきprovider idです。たとえば、ベースproviderのAPIキーや認証プロファイルを共有するcoding providerなどです。 |
| `channelEnvVars` | No | `Record<string, string[]>` | OpenClawがプラグインコードを読み込まずに調べられる、軽量なchannel envメタデータです。env駆動のchannelセットアップや、汎用の起動/設定ヘルパーが認識すべき認証サーフェスに使用します。 |
| `providerAuthChoices` | No | `object[]` | オンボーディングピッカー、優先provider解決、単純なCLIフラグ配線のための軽量な認証選択メタデータです。 |
| `contracts` | No | `object` | speech、realtime transcription、realtime voice、media-understanding、image-generation、music-generation、video-generation、web-fetch、web search、およびtool ownership向けの静的なバンドルcapabilityスナップショットです。 |
| `channelConfigs` | No | `Record<string, object>` | ランタイム読み込み前に検出および検証サーフェスへマージされる、マニフェスト所有のchannel設定メタデータです。 |
| `skills` | No | `string[]` | plugin rootからの相対パスで指定する、読み込むSkillsディレクトリです。 |
| `name` | No | `string` | 人が読めるプラグイン名です。 |
| `description` | No | `string` | プラグインサーフェスに表示される短い要約です。 |
| `version` | No | `string` | 情報提供用のプラグインバージョンです。 |
| `uiHints` | No | `Record<string, object>` | 設定フィールド用のUIラベル、プレースホルダー、およびsensitivityヒントです。 |

## providerAuthChoicesリファレンス

各`providerAuthChoices`エントリーは、1つのオンボーディングまたは認証の選択肢を記述します。  
OpenClawはこれをproviderランタイムの読み込み前に読み取ります。

| Field | Required | Type | What it means |
| --------------------- | -------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `provider` | Yes | `string` | この選択肢が属するprovider id。 |
| `method` | Yes | `string` | ディスパッチ先の認証method id。 |
| `choiceId` | Yes | `string` | オンボーディングおよびCLIフローで使われる安定した認証選択id。 |
| `choiceLabel` | No | `string` | ユーザー向けラベル。省略した場合、OpenClawは`choiceId`にフォールバックします。 |
| `choiceHint` | No | `string` | ピッカー向けの短い補助テキスト。 |
| `assistantPriority` | No | `number` | assistant主導の対話型ピッカーで、値が小さいほど先に並びます。 |
| `assistantVisibility` | No | `"visible"` \| `"manual-only"` | assistantピッカーではこの選択肢を非表示にしつつ、手動CLI選択は引き続き許可します。 |
| `deprecatedChoiceIds` | No | `string[]` | この置き換え用選択肢へユーザーをリダイレクトすべきレガシーchoice id。 |
| `groupId` | No | `string` | 関連する選択肢をグループ化するためのオプションのgroup id。 |
| `groupLabel` | No | `string` | そのグループのユーザー向けラベル。 |
| `groupHint` | No | `string` | グループ向けの短い補助テキスト。 |
| `optionKey` | No | `string` | 単一フラグの単純な認証フロー向けの内部option key。 |
| `cliFlag` | No | `string` | `--openrouter-api-key`のようなCLIフラグ名。 |
| `cliOption` | No | `string` | `--openrouter-api-key <key>`のような完全なCLIオプション形状。 |
| `cliDescription` | No | `string` | CLIヘルプで使われる説明。 |
| `onboardingScopes` | No | `Array<"text-inference" \| "image-generation">` | この選択肢をどのオンボーディングサーフェスに表示すべきか。省略した場合、デフォルトは`["text-inference"]`です。 |

## commandAliasesリファレンス

`commandAliases` は、プラグインが所有するランタイムコマンド名を、ユーザーが誤って`plugins.allow`に入れたり、ルートCLIコマンドとして実行しようとしたりする可能性がある場合に使用します。OpenClawはこのメタデータを、プラグインランタイムコードをimportせずに診断へ使用します。

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

| Field | Required | Type | What it means |
| ------------ | -------- | ----------------- | ----------------------------------------------------------------------- |
| `name` | Yes | `string` | このプラグインに属するコマンド名。 |
| `kind` | No | `"runtime-slash"` | エイリアスがルートCLIコマンドではなく、チャットスラッシュコマンドであることを示します。 |
| `cliCommand` | No | `string` | 存在する場合、CLI操作向けに提案する関連ルートCLIコマンド。 |

## uiHintsリファレンス

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

| Field | Type | What it means |
| ------------- | ---------- | --------------------------------------- |
| `label` | `string` | ユーザー向けフィールドラベル。 |
| `help` | `string` | 短い補助テキスト。 |
| `tags` | `string[]` | オプションのUIタグ。 |
| `advanced` | `boolean` | フィールドを高度な項目として扱います。 |
| `sensitive` | `boolean` | フィールドをシークレットまたはセンシティブとして扱います。 |
| `placeholder` | `string` | フォーム入力用のプレースホルダーテキスト。 |

## contractsリファレンス

`contracts` は、OpenClawがプラグインランタイムをimportせずに読める、静的なcapability所有メタデータにのみ使用してください。

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

各リストはオプションです。

| Field | Type | What it means |
| -------------------------------- | ---------- | -------------------------------------------------------------- |
| `speechProviders` | `string[]` | このプラグインが所有するspeech provider id。 |
| `realtimeTranscriptionProviders` | `string[]` | このプラグインが所有するrealtime-transcription provider id。 |
| `realtimeVoiceProviders` | `string[]` | このプラグインが所有するrealtime-voice provider id。 |
| `mediaUnderstandingProviders` | `string[]` | このプラグインが所有するmedia-understanding provider id。 |
| `imageGenerationProviders` | `string[]` | このプラグインが所有するimage-generation provider id。 |
| `videoGenerationProviders` | `string[]` | このプラグインが所有するvideo-generation provider id。 |
| `webFetchProviders` | `string[]` | このプラグインが所有するweb-fetch provider id。 |
| `webSearchProviders` | `string[]` | このプラグインが所有するweb-search provider id。 |
| `tools` | `string[]` | バンドルされたコントラクトチェック向けに、このプラグインが所有するagentツール名。 |

## channelConfigsリファレンス

`channelConfigs` は、チャネルプラグインがランタイム読み込み前に軽量な設定メタデータを必要とする場合に使用します。

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

各チャネルエントリーには次を含めることができます。

| Field | Type | What it means |
| ------------- | ------------------------ | ----------------------------------------------------------------------------------------- |
| `schema` | `object` | `channels.<id>`用のJSONスキーマ。宣言された各チャネル設定エントリーで必須です。 |
| `uiHints` | `Record<string, object>` | そのチャネル設定セクション用のオプションのUIラベル/プレースホルダー/sensitiveヒント。 |
| `label` | `string` | ランタイムメタデータが未準備のときに、ピッカーおよびinspectサーフェスへマージされるチャネルラベル。 |
| `description` | `string` | inspectおよびcatalogサーフェス向けの短いチャネル説明。 |
| `preferOver` | `string[]` | 選択サーフェスでこのチャネルが優先されるべき、レガシーまたは低優先度のplugin id。 |

## modelSupportリファレンス

`modelSupport` は、プラグインランタイムの読み込み前に、`gpt-5.4` や `claude-sonnet-4.6` のような短縮形モデルidからOpenClawがproviderプラグインを推測すべき場合に使用します。

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClawは次の優先順位を適用します。

- 明示的な`provider/model`参照は、所有する`providers`マニフェストメタデータを使用する
- `modelPatterns` は `modelPrefixes` より優先される
- 非バンドルプラグイン1つとバンドルプラグイン1つの両方が一致する場合、非バンドルプラグインが勝つ
- 残る曖昧さは、ユーザーまたは設定がproviderを指定するまで無視される

フィールド:

| Field | Type | What it means |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | 短縮形モデルidに対して`startsWith`で一致させるプレフィックス。 |
| `modelPatterns` | `string[]` | プロファイル接尾辞を除去した後の短縮形モデルidに対して一致させる正規表現ソース。 |

レガシーなトップレベルcapabilityキーは非推奨です。`openclaw doctor --fix` を使用して、`speechProviders`、`realtimeTranscriptionProviders`、`realtimeVoiceProviders`、`mediaUnderstandingProviders`、`imageGenerationProviders`、`videoGenerationProviders`、`webFetchProviders`、`webSearchProviders` を `contracts` 配下へ移動してください。通常のマニフェスト読み込みでは、これらのトップレベルフィールドをcapability所有としてはもう扱いません。

## マニフェストとpackage.jsonの違い

この2つのファイルは異なる役割を持ちます。

| File | Use it for |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | 検出、設定検証、認証選択メタデータ、およびプラグインコード実行前に存在している必要があるUIヒント |
| `package.json` | npmメタデータ、依存関係のインストール、およびentrypoint、インストールゲーティング、セットアップ、またはcatalogメタデータに使われる`openclaw`ブロック |

どこにどのメタデータを置くべきか迷った場合は、次のルールを使ってください。

- OpenClawがプラグインコードを読み込む前に知っている必要がある場合は、`openclaw.plugin.json` に置く
- パッケージング、entry file、またはnpm install動作に関するものであれば、`package.json` に置く

### 検出に影響するpackage.jsonフィールド

一部のランタイム前プラグインメタデータは、`openclaw.plugin.json` ではなく、意図的に`package.json`の`openclaw`ブロック配下に置かれます。

重要な例:

| Field | What it means |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions` | ネイティブプラグインのentrypointを宣言します。 |
| `openclaw.setupEntry` | オンボーディングおよび遅延チャネル起動時に使われる、軽量なセットアップ専用entrypointです。 |
| `openclaw.channel` | ラベル、ドキュメントパス、エイリアス、選択時コピーなどの軽量なチャネルcatalogメタデータです。 |
| `openclaw.channel.configuredState` | 「envのみのセットアップがすでに存在するか」を、完全なチャネルランタイムを読み込まずに判定できる軽量なconfigured-state checkerメタデータです。 |
| `openclaw.channel.persistedAuthState` | 「すでに何かログイン済みか」を、完全なチャネルランタイムを読み込まずに判定できる軽量なpersisted-auth checkerメタデータです。 |
| `openclaw.install.npmSpec` / `openclaw.install.localPath` | バンドルプラグインおよび外部公開プラグイン向けのインストール/更新ヒントです。 |
| `openclaw.install.defaultChoice` | 複数のインストール元が利用可能なときの優先インストール経路です。 |
| `openclaw.install.minHostVersion` | `>=2026.3.22` のようなsemver floorを使う、サポートされる最小OpenClawホストバージョンです。 |
| `openclaw.install.allowInvalidConfigRecovery` | 設定が不正な場合に、限定的なバンドルプラグイン再インストール回復経路を許可します。 |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | 起動時に、完全なチャネルプラグインの前にセットアップ専用チャネルサーフェスを読み込めるようにします。 |

`openclaw.install.minHostVersion` は、インストール時およびマニフェストレジストリ読み込み時に適用されます。不正な値は拒否され、有効ではあるが新しすぎる値の場合、古いホストではそのプラグインはスキップされます。

`openclaw.install.allowInvalidConfigRecovery` は意図的に限定的です。これによって任意の壊れた設定がインストール可能になるわけではありません。現時点では、特定の古いバンドルプラグインのアップグレード失敗、たとえば欠落したバンドルプラグインパスや、同じバンドルプラグインに対する古い `channels.<id>` エントリーなどから、インストールフローが回復できるようにするだけです。無関係な設定エラーは引き続きインストールをブロックし、運用者を `openclaw doctor --fix` へ誘導します。

`openclaw.channel.persistedAuthState` は、小さなcheckerモジュール用のpackageメタデータです。

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

これは、セットアップ、doctor、configured-stateフローが、完全なチャネルプラグインを読み込む前に、軽量なyes/no認証probeを必要とする場合に使います。対象のexportは、永続化された状態だけを読む小さな関数にしてください。完全なチャネルランタイムbarrel経由にしないでください。

`openclaw.channel.configuredState` も、軽量なenv-only configuredチェック向けに同じ形状に従います。

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

これは、チャネルがenvまたはその他の小さな非ランタイム入力からconfigured-stateを判定できる場合に使います。チェックに完全な設定解決や実際のチャネルランタイムが必要な場合は、そのロジックを代わりにプラグインの`config.hasConfiguredState`フックに置いてください。

## JSONスキーマ要件

- **すべてのプラグインはJSONスキーマを含める必要があります**。設定を受け付けない場合でも同様です。
- 空のスキーマでも問題ありません（例: `{ "type": "object", "additionalProperties": false }`）。
- スキーマはランタイム時ではなく、設定の読み取り/書き込み時に検証されます。

## 検証の動作

- 未知の`channels.*`キーは、チャネルidがプラグインマニフェストで宣言されていない限り、**エラー**です。
- `plugins.entries.<id>`、`plugins.allow`、`plugins.deny`、`plugins.slots.*` は、**検出可能な**plugin idを参照している必要があります。未知のidは**エラー**です。
- プラグインがインストールされていても、マニフェストまたはスキーマが壊れている、あるいは欠落している場合、検証は失敗し、Doctorがそのプラグインエラーを報告します。
- プラグイン設定が存在するがプラグインが**無効**な場合、設定は保持され、Doctorとログに**警告**が表示されます。

完全な`plugins.*`スキーマについては、[Configuration reference](/ja-JP/gateway/configuration)を参照してください。

## 注意事項

- マニフェストは、ローカルファイルシステム読み込みを含む**ネイティブOpenClawプラグインで必須**です。
- ランタイムは引き続きプラグインモジュールを別途読み込みます。マニフェストは検出と検証専用です。
- ネイティブマニフェストはJSON5として解析されるため、最終値がオブジェクトである限り、コメント、末尾カンマ、引用符なしキーを使用できます。
- マニフェストローダーが読み取るのは文書化されたマニフェストフィールドだけです。ここに独自のトップレベルキーを追加しないでください。
- `providerAuthEnvVars` は、認証probe、env-marker検証、および同様の、env名を調べるためだけにプラグインランタイムを起動すべきでないprovider認証サーフェス向けの軽量メタデータ経路です。
- `providerAuthAliases` を使うと、providerバリアントが別のproviderの認証env var、認証プロファイル、設定ベース認証、およびAPIキーのオンボーディング選択肢を再利用できます。この関係をcoreにハードコードする必要はありません。
- `channelEnvVars` は、shell-envフォールバック、セットアッププロンプト、および同様の、env名を調べるためだけにチャネルランタイムを起動すべきでないチャネルサーフェス向けの軽量メタデータ経路です。
- `providerAuthChoices` は、認証選択ピッカー、`--auth-choice` 解決、preferred-providerマッピング、およびproviderランタイム読み込み前の単純なオンボーディングCLIフラグ登録向けの軽量メタデータ経路です。providerコードを必要とするランタイムのウィザードメタデータについては、[Provider runtime hooks](/ja-JP/plugins/architecture#provider-runtime-hooks)を参照してください。
- 排他的なplugin kindは `plugins.slots.*` を通じて選択されます。
  - `kind: "memory"` は `plugins.slots.memory` で選択されます。
  - `kind: "context-engine"` は `plugins.slots.contextEngine` で選択されます（デフォルト: 組み込みの`legacy`）。
- プラグインに不要であれば、`channels`、`providers`、`cliBackends`、`skills` は省略できます。
- プラグインがネイティブモジュールに依存している場合は、ビルド手順と必要なpackage managerのallowlist要件（例: pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`）を文書化してください。

## 関連

- [Building Plugins](/ja-JP/plugins/building-plugins) — プラグインのはじめに
- [Plugin Architecture](/ja-JP/plugins/architecture) — 内部アーキテクチャ
- [SDK Overview](/ja-JP/plugins/sdk-overview) — Plugin SDKリファレンス
