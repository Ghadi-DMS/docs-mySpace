---
read_when:
    - メモリー検索プロバイダーまたは埋め込みモデルを設定したい場合
    - QMD バックエンドをセットアップしたい場合
    - ハイブリッド検索、MMR、または時間減衰を調整したい場合
    - マルチモーダルなメモリーインデックス作成を有効にしたい場合
summary: メモリー検索、埋め込みプロバイダー、QMD、ハイブリッド検索、マルチモーダルインデックス作成に関するすべての設定項目
title: メモリー設定リファレンス
x-i18n:
    generated_at: "2026-04-12T23:34:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 299ca9b69eea292ea557a2841232c637f5c1daf2bc0f73c0a42f7c0d8d566ce2
    source_path: reference/memory-config.md
    workflow: 15
---

# メモリー設定リファレンス

このページでは、OpenClaw のメモリー検索に関するすべての設定項目を一覧化しています。概念的な概要については、以下を参照してください。

- [Memory Overview](/ja-JP/concepts/memory) -- メモリーの仕組み
- [Builtin Engine](/ja-JP/concepts/memory-builtin) -- デフォルトの SQLite バックエンド
- [QMD Engine](/ja-JP/concepts/memory-qmd) -- ローカルファーストのサイドカー
- [Memory Search](/ja-JP/concepts/memory-search) -- 検索パイプラインとチューニング
- [Active Memory](/ja-JP/concepts/active-memory) -- 対話セッション向けのメモリーサブエージェントの有効化

特に明記がない限り、すべてのメモリー検索設定は `openclaw.json` の
`agents.defaults.memorySearch` 配下にあります。

**Active Memory** の機能トグルとサブエージェント設定を探している場合は、
`memorySearch` ではなく `plugins.entries.active-memory` 配下にあります。

Active Memory は 2 段階のゲートモデルを使用します。

1. Plugin が有効であり、現在のエージェント ID を対象にしていること
2. リクエストが、対象となる対話型の永続チャットセッションであること

アクティベーションモデル、
Plugin が所有する設定、トランスクリプトの永続化、安全なロールアウトパターンについては、
[Active Memory](/ja-JP/concepts/active-memory) を参照してください。

---

## プロバイダー選択

| Key        | Type      | Default          | Description                                                                                 |
| ---------- | --------- | ---------------- | ------------------------------------------------------------------------------------------- |
| `provider` | `string`  | 自動検出    | 埋め込みアダプター ID: `openai`、`gemini`、`voyage`、`mistral`、`bedrock`、`ollama`、`local` |
| `model`    | `string`  | プロバイダーのデフォルト | 埋め込みモデル名                                                                        |
| `fallback` | `string`  | `"none"`         | プライマリーが失敗したときのフォールバックアダプター ID                                                  |
| `enabled`  | `boolean` | `true`           | メモリー検索を有効または無効にする                                                             |

### 自動検出順序

`provider` が設定されていない場合、OpenClaw は利用可能な最初のものを選択します。

1. `local` -- `memorySearch.local.modelPath` が設定されており、そのファイルが存在する場合。
2. `openai` -- OpenAI キーを解決できる場合。
3. `gemini` -- Gemini キーを解決できる場合。
4. `voyage` -- Voyage キーを解決できる場合。
5. `mistral` -- Mistral キーを解決できる場合。
6. `bedrock` -- AWS SDK の認証情報チェーンが解決される場合（インスタンスロール、アクセスキー、プロファイル、SSO、web identity、または共有 config）。

`ollama` はサポートされていますが自動検出はされません（明示的に設定してください）。

### API キー解決

リモート埋め込みには API キーが必要です。Bedrock は代わりに AWS SDK のデフォルト
認証情報チェーンを使用します（インスタンスロール、SSO、アクセスキー）。

| Provider | Env var                        | Config key                        |
| -------- | ------------------------------ | --------------------------------- |
| OpenAI   | `OPENAI_API_KEY`               | `models.providers.openai.apiKey`  |
| Gemini   | `GEMINI_API_KEY`               | `models.providers.google.apiKey`  |
| Voyage   | `VOYAGE_API_KEY`               | `models.providers.voyage.apiKey`  |
| Mistral  | `MISTRAL_API_KEY`              | `models.providers.mistral.apiKey` |
| Bedrock  | AWS 認証情報チェーン           | API キー不要                 |
| Ollama   | `OLLAMA_API_KEY`（プレースホルダー） | --                                |

Codex OAuth は chat/completions のみを対象としており、埋め込み
リクエストには使用できません。

---

## リモートエンドポイント設定

カスタム OpenAI 互換エンドポイントの使用や、プロバイダーのデフォルトを上書きする場合:

| Key              | Type     | Description                                        |
| ---------------- | -------- | -------------------------------------------------- |
| `remote.baseUrl` | `string` | カスタム API ベース URL                                |
| `remote.apiKey`  | `string` | API キーを上書き                                   |
| `remote.headers` | `object` | 追加の HTTP ヘッダー（プロバイダーのデフォルトとマージされる） |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Gemini 固有の設定

| Key                    | Type     | Default                | Description                                |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | `gemini-embedding-2-preview` もサポート |
| `outputDimensionality` | `number` | `3072`                 | Embedding 2 では 768、1536、または 3072        |

<Warning>
モデルまたは `outputDimensionality` を変更すると、自動的に完全再インデックスがトリガーされます。
</Warning>

---

## Bedrock 埋め込み設定

Bedrock は AWS SDK のデフォルト認証情報チェーンを使用します -- API キーは不要です。
OpenClaw が Bedrock 対応のインスタンスロールを持つ EC2 上で動作している場合は、
プロバイダーとモデルを設定するだけです。

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0",
      },
    },
  },
}
```

| Key                    | Type     | Default                        | Description                     |
| ---------------------- | -------- | ------------------------------ | ------------------------------- |
| `model`                | `string` | `amazon.titan-embed-text-v2:0` | 任意の Bedrock 埋め込みモデル ID  |
| `outputDimensionality` | `number` | モデルのデフォルト                  | Titan V2 では 256、512、または 1024 |

### サポートされるモデル

以下のモデルがサポートされています（ファミリー検出と次元数の
デフォルト付き）。

| Model ID                                   | Provider   | Default Dims | Configurable Dims    |
| ------------------------------------------ | ---------- | ------------ | -------------------- |
| `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256、512、1024       |
| `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                   |
| `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                   |
| `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                   |
| `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256、384、1024、3072 |
| `cohere.embed-english-v3`                  | Cohere     | 1024         | --                   |
| `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                   |
| `cohere.embed-v4:0`                        | Cohere     | 1536         | 256-1536             |
| `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                   |
| `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                   |

スループット接尾辞付きのバリアント（例: `amazon.titan-embed-text-v1:2:8k`）は、
ベースモデルの設定を継承します。

### 認証

Bedrock 認証は、標準の AWS SDK 認証情報解決順序を使用します。

1. 環境変数（`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`）
2. SSO トークンキャッシュ
3. Web identity token 認証情報
4. 共有認証情報ファイルと config ファイル
5. ECS または EC2 メタデータ認証情報

リージョンは `AWS_REGION`、`AWS_DEFAULT_REGION`、`amazon-bedrock`
プロバイダーの `baseUrl` から解決されるか、デフォルトで `us-east-1` になります。

### IAM 権限

IAM ロールまたはユーザーには以下が必要です。

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "*"
}
```

最小権限にするには、`InvokeModel` を特定のモデルに限定してください。

```
arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
```

---

## ローカル埋め込み設定

| Key                   | Type     | Default                | Description                     |
| --------------------- | -------- | ---------------------- | ------------------------------- |
| `local.modelPath`     | `string` | 自動ダウンロード        | GGUF モデルファイルへのパス         |
| `local.modelCacheDir` | `string` | node-llama-cpp のデフォルト | ダウンロード済みモデルのキャッシュディレクトリ |

デフォルトモデル: `embeddinggemma-300m-qat-Q8_0.gguf`（約 0.6 GB、自動ダウンロード）。
ネイティブビルドが必要です: `pnpm approve-builds` を実行し、その後 `pnpm rebuild node-llama-cpp` を実行してください。

---

## ハイブリッド検索設定

すべて `memorySearch.query.hybrid` 配下です。

| Key                   | Type      | Default | Description                        |
| --------------------- | --------- | ------- | ---------------------------------- |
| `enabled`             | `boolean` | `true`  | ハイブリッド BM25 + ベクトル検索を有効にする |
| `vectorWeight`        | `number`  | `0.7`   | ベクトルスコアの重み（0-1）     |
| `textWeight`          | `number`  | `0.3`   | BM25 スコアの重み（0-1）       |
| `candidateMultiplier` | `number`  | `4`     | 候補プールサイズの倍率     |

### MMR（多様性）

| Key           | Type      | Default | Description                          |
| ------------- | --------- | ------- | ------------------------------------ |
| `mmr.enabled` | `boolean` | `false` | MMR 再ランキングを有効にする                |
| `mmr.lambda`  | `number`  | `0.7`   | 0 = 最大多様性、1 = 最大関連性 |

### 時間減衰（新しさ）

| Key                          | Type      | Default | Description               |
| ---------------------------- | --------- | ------- | ------------------------- |
| `temporalDecay.enabled`      | `boolean` | `false` | 新しさブーストを有効にする      |
| `temporalDecay.halfLifeDays` | `number`  | `30`    | N 日ごとにスコアが半減する |

エバーグリーンファイル（`MEMORY.md`、`memory/` 内の日付なしファイル）は減衰しません。

### 完全な例

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## 追加のメモリーパス

| Key          | Type       | Description                              |
| ------------ | ---------- | ---------------------------------------- |
| `extraPaths` | `string[]` | インデックス対象にする追加のディレクトリまたはファイル |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

パスには絶対パスまたは workspace 相対パスを指定できます。ディレクトリは
`.md` ファイルを再帰的にスキャンします。symlink の扱いはアクティブなバックエンドに依存します。
builtin engine は symlink を無視し、QMD は基盤となる QMD
scanner の動作に従います。

エージェントスコープのクロスエージェントトランスクリプト検索には、
`memory.qmd.paths` ではなく `agents.list[].memorySearch.qmd.extraCollections` を使用してください。
これらの追加コレクションは同じ `{ path, name, pattern? }` 形式に従いますが、
エージェントごとにマージされ、パスが現在の workspace 外を指す場合でも、
明示的な共有名を保持できます。
同じ解決済みパスが `memory.qmd.paths` と
`memorySearch.qmd.extraCollections` の両方に現れた場合、QMD は最初のエントリを保持し、
重複はスキップします。

---

## マルチモーダルメモリー（Gemini）

Gemini Embedding 2 を使用して、Markdown と一緒に画像や音声もインデックスします。

| Key                       | Type       | Default    | Description                            |
| ------------------------- | ---------- | ---------- | -------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | マルチモーダルインデックス作成を有効にする             |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`、`["audio"]`、または `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10000000` | インデックス作成対象の最大ファイルサイズ             |

`extraPaths` 内のファイルにのみ適用されます。デフォルトのメモリールートは引き続き Markdown のみです。
`gemini-embedding-2-preview` が必要です。`fallback` は `"none"` でなければなりません。

サポートされる形式: `.jpg`、`.jpeg`、`.png`、`.webp`、`.gif`、`.heic`、`.heif`
（画像）、`.mp3`、`.wav`、`.ogg`、`.opus`、`.m4a`、`.aac`、`.flac`（音声）。

---

## 埋め込みキャッシュ

| Key                | Type      | Default | Description                      |
| ------------------ | --------- | ------- | -------------------------------- |
| `cache.enabled`    | `boolean` | `false` | SQLite にチャンク埋め込みをキャッシュする |
| `cache.maxEntries` | `number`  | `50000` | キャッシュする埋め込みの最大数            |

再インデックス時やトランスクリプト更新時に、変更されていないテキストの再埋め込みを防ぎます。

---

## バッチインデックス作成

| Key                           | Type      | Default | Description                |
| ----------------------------- | --------- | ------- | -------------------------- |
| `remote.batch.enabled`        | `boolean` | `false` | バッチ埋め込み API を有効にする |
| `remote.batch.concurrency`    | `number`  | `2`     | 並列バッチジョブ数        |
| `remote.batch.wait`           | `boolean` | `true`  | バッチ完了を待機する  |
| `remote.batch.pollIntervalMs` | `number`  | --      | ポーリング間隔              |
| `remote.batch.timeoutMinutes` | `number`  | --      | バッチのタイムアウト              |

`openai`、`gemini`、`voyage` で利用できます。OpenAI のバッチは通常、
大規模なバックフィルで最も高速かつ低コストです。

---

## セッションメモリー検索（experimental）

セッショントランスクリプトをインデックスし、`memory_search` 経由で表示します。

| Key                           | Type       | Default      | Description                             |
| ----------------------------- | ---------- | ------------ | --------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`      | セッションインデックス作成を有効にする                 |
| `sources`                     | `string[]` | `["memory"]` | トランスクリプトを含めるには `"sessions"` を追加する |
| `sync.sessions.deltaBytes`    | `number`   | `100000`     | 再インデックスのバイト閾値              |
| `sync.sessions.deltaMessages` | `number`   | `50`         | 再インデックスのメッセージ閾値           |

セッションインデックス作成は opt-in で、非同期に実行されます。結果は多少
古くなる場合があります。セッションログはディスク上にあるため、ファイルシステムアクセスを信頼境界として扱ってください。

---

## SQLite ベクトル高速化（sqlite-vec）

| Key                          | Type      | Default | Description                       |
| ---------------------------- | --------- | ------- | --------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | ベクトルクエリに sqlite-vec を使用する |
| `store.vector.extensionPath` | `string`  | bundled | sqlite-vec パスを上書きする          |

sqlite-vec が利用できない場合、OpenClaw は自動的にインプロセスのコサイン
類似度へフォールバックします。

---

## インデックスストレージ

| Key                   | Type     | Default                               | Description                                 |
| --------------------- | -------- | ------------------------------------- | ------------------------------------------- |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | インデックスの場所（`{agentId}` トークンをサポート） |
| `store.fts.tokenizer` | `string` | `unicode61`                           | FTS5 トークナイザー（`unicode61` または `trigram`）   |

---

## QMD バックエンド設定

有効にするには `memory.backend = "qmd"` を設定します。すべての QMD 設定は
`memory.qmd` 配下にあります。

| Key                      | Type      | Default  | Description                                  |
| ------------------------ | --------- | -------- | -------------------------------------------- |
| `command`                | `string`  | `qmd`    | QMD 実行ファイルパス                          |
| `searchMode`             | `string`  | `search` | 検索コマンド: `search`、`vsearch`、`query` |
| `includeDefaultMemory`   | `boolean` | `true`   | `MEMORY.md` + `memory/**/*.md` を自動インデックスする    |
| `paths[]`                | `array`   | --       | 追加パス: `{ name, path, pattern? }`      |
| `sessions.enabled`       | `boolean` | `false`  | セッショントランスクリプトをインデックスする                    |
| `sessions.retentionDays` | `number`  | --       | トランスクリプト保持期間                         |
| `sessions.exportDir`     | `string`  | --       | エクスポートディレクトリ                             |

OpenClaw は現在の QMD コレクションと MCP クエリ形式を優先しますが、
必要に応じてレガシーの `--mask` コレクションフラグや古い MCP ツール名にフォールバックすることで、
古い QMD リリースも引き続き動作させます。

QMD モデルの上書きは OpenClaw config ではなく QMD 側に留まります。QMD のモデルを
グローバルに上書きする必要がある場合は、Gateway
ランタイム環境で `QMD_EMBED_MODEL`、`QMD_RERANK_MODEL`、`QMD_GENERATE_MODEL` などの環境変数を設定してください。

### 更新スケジュール

| Key                       | Type      | Default | Description                           |
| ------------------------- | --------- | ------- | ------------------------------------- |
| `update.interval`         | `string`  | `5m`    | 更新間隔                      |
| `update.debounceMs`       | `number`  | `15000` | ファイル変更のデバウンス                 |
| `update.onBoot`           | `boolean` | `true`  | 起動時に更新する                    |
| `update.waitForBootSync`  | `boolean` | `false` | 更新完了まで起動をブロックする |
| `update.embedInterval`    | `string`  | --      | 埋め込み用の別個の実行間隔                |
| `update.commandTimeoutMs` | `number`  | --      | QMD コマンドのタイムアウト              |
| `update.updateTimeoutMs`  | `number`  | --      | QMD 更新処理のタイムアウト     |
| `update.embedTimeoutMs`   | `number`  | --      | QMD 埋め込み処理のタイムアウト      |

### 制限

| Key                       | Type     | Default | Description                |
| ------------------------- | -------- | ------- | -------------------------- |
| `limits.maxResults`       | `number` | `6`     | 最大検索結果数         |
| `limits.maxSnippetChars`  | `number` | --      | スニペット長の上限制限       |
| `limits.maxInjectedChars` | `number` | --      | 注入する総文字数の上限制限 |
| `limits.timeoutMs`        | `number` | `4000`  | 検索タイムアウト             |

### スコープ

どのセッションが QMD 検索結果を受け取れるかを制御します。スキーマは
[`session.sendPolicy`](/ja-JP/gateway/configuration-reference#session) と同じです。

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

同梱のデフォルトでは、グループを引き続き拒否しつつ、
ダイレクトセッションとチャネルセッションを許可します。

デフォルトは DM のみです。`match.keyPrefix` は正規化された session key に一致し、
`match.rawKeyPrefix` は `agent:<id>:` を含む生のキーに一致します。

### 引用

`memory.citations` はすべてのバックエンドに適用されます。

| Value            | Behavior                                            |
| ---------------- | --------------------------------------------------- |
| `auto`（デフォルト） | スニペットに `Source: <path#line>` フッターを含める    |
| `on`             | 常にフッターを含める                               |
| `off`            | フッターを省略する（パスは引き続き内部的にエージェントへ渡される） |

### 完全な QMD の例

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming（experimental）

Dreaming は `agents.defaults.memorySearch` ではなく、
`plugins.entries.memory-core.config.dreaming` 配下で設定します。

Dreaming は 1 回のスケジュールされたスイープとして実行され、内部の light / deep / REM フェーズを
実装詳細として使用します。

概念上の動作とスラッシュコマンドについては、[Dreaming](/ja-JP/concepts/dreaming) を参照してください。

### ユーザー設定

| Key         | Type      | Default     | Description                                       |
| ----------- | --------- | ----------- | ------------------------------------------------- |
| `enabled`   | `boolean` | `false`     | Dreaming 全体を有効または無効にする               |
| `frequency` | `string`  | `0 3 * * *` | 完全な Dreaming スイープの任意の Cron 実行間隔 |

### 例

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
          },
        },
      },
    },
  },
}
```

注:

- Dreaming はマシン state を `memory/.dreams/` に書き込みます。
- Dreaming は人間が読めるナラティブ出力を `DREAMS.md`（または既存の `dreams.md`）に書き込みます。
- light / deep / REM フェーズのポリシーと閾値は内部動作であり、ユーザー向け設定ではありません。
