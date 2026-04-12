---
read_when:
    - Ollama 経由でクラウドモデルまたはローカルモデルを使って OpenClaw を実行したい
    - Ollama のセットアップと設定ガイダンスが必要です
summary: OpenClaw を Ollama で実行する（クラウドモデルとローカルモデル）
title: Ollama
x-i18n:
    generated_at: "2026-04-12T23:32:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: ec796241b884ca16ec7077df4f3f1910e2850487bb3ea94f8fdb37c77e02b219
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama は、マシン上でオープンソースモデルを簡単に実行できるローカル LLM ランタイムです。OpenClaw は Ollama のネイティブ API（`/api/chat`）と統合し、ストリーミングとツール呼び出しをサポートし、`OLLAMA_API_KEY`（または auth profile）でオプトインし、明示的な `models.providers.ollama` エントリーを定義していない場合にはローカル Ollama モデルを自動検出できます。

<Warning>
**リモート Ollama ユーザー向け**: OpenClaw で `/v1` の OpenAI 互換 URL（`http://host:11434/v1`）を使わないでください。これによりツール呼び出しが壊れ、モデルが生のツール JSON をプレーンテキストとして出力することがあります。代わりに、ネイティブ Ollama API URL を使ってください: `baseUrl: "http://host:11434"`（`/v1` なし）。
</Warning>

## はじめに

希望するセットアップ方法とモードを選んでください。

<Tabs>
  <Tab title="オンボーディング（推奨）">
    **最適な用途:** 自動モデル検出付きで動作する Ollama セットアップに最速で到達したい場合。

    <Steps>
      <Step title="オンボーディングを実行する">
        ```bash
        openclaw onboard
        ```

        provider リストから **Ollama** を選択します。
      </Step>
      <Step title="モードを選ぶ">
        - **Cloud + Local** — クラウドホスト型モデルとローカルモデルを一緒に使う
        - **Local** — ローカルモデルのみ

        **Cloud + Local** を選び、まだ ollama.com にサインインしていない場合、オンボーディングはブラウザーのサインインフローを開きます。
      </Step>
      <Step title="モデルを選択する">
        オンボーディングは利用可能なモデルを検出し、デフォルトを提案します。選択したモデルがローカルで利用できない場合は自動で pull します。
      </Step>
      <Step title="モデルが利用可能であることを確認する">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    ### 非対話モード

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --accept-risk
    ```

    必要に応じて、カスタム base URL や model を指定できます。

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

  </Tab>

  <Tab title="手動セットアップ">
    **最適な用途:** インストール、model pull、設定を完全に制御したい場合。

    <Steps>
      <Step title="Ollama をインストールする">
        [ollama.com/download](https://ollama.com/download) からダウンロードします。
      </Step>
      <Step title="ローカルモデルを pull する">
        ```bash
        ollama pull gemma4
        # or
        ollama pull gpt-oss:20b
        # or
        ollama pull llama3.3
        ```
      </Step>
      <Step title="クラウドモデル用にサインインする（任意）">
        クラウドモデルも使いたい場合:

        ```bash
        ollama signin
        ```
      </Step>
      <Step title="OpenClaw で Ollama を有効にする">
        API キーには任意の値を設定してください（Ollama に実際のキーは不要です）。

        ```bash
        # 環境変数を設定
        export OLLAMA_API_KEY="ollama-local"

        # または設定ファイルで構成
        openclaw config set models.providers.ollama.apiKey "ollama-local"
        ```
      </Step>
      <Step title="model を確認して設定する">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        または、設定でデフォルトを指定します。

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## クラウドモデル

<Tabs>
  <Tab title="Cloud + Local">
    クラウドモデルを使うと、ローカルモデルと並行してクラウドホスト型モデルを実行できます。例として `kimi-k2.5:cloud`、`minimax-m2.7:cloud`、`glm-5.1:cloud` があり、これらはローカルでの `ollama pull` を**必要としません**。

    セットアップ中に **Cloud + Local** モードを選択してください。ウィザードはサインイン済みかどうかを確認し、必要に応じてブラウザーのサインインフローを開きます。認証を確認できない場合、ウィザードはローカルモデルのデフォルトにフォールバックします。

    [ollama.com/signin](https://ollama.com/signin) で直接サインインすることもできます。

    OpenClaw は現在、次のクラウドデフォルトを提案します: `kimi-k2.5:cloud`、`minimax-m2.7:cloud`、`glm-5.1:cloud`。

  </Tab>

  <Tab title="ローカルのみ">
    ローカル専用モードでは、OpenClaw はローカル Ollama インスタンスからモデルを検出します。クラウドへのサインインは不要です。

    OpenClaw は現在、ローカルのデフォルトとして `gemma4` を提案します。

  </Tab>
</Tabs>

## モデル検出（暗黙的 provider）

`OLLAMA_API_KEY`（または auth profile）を設定し、**`models.providers.ollama` を定義しない**場合、OpenClaw は `http://127.0.0.1:11434` にあるローカル Ollama インスタンスからモデルを検出します。

| 動作                 | 詳細                                                                                                                                                                   |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| カタログクエリ       | `/api/tags` を問い合わせる                                                                                                                                            |
| 機能検出             | `contextWindow` を読み取り、機能（vision を含む）を検出するために、最善努力の `/api/show` 参照を使う                                                               |
| Vision モデル        | `/api/show` により `vision` 機能が報告されたモデルは、画像対応（`input: ["text", "image"]`）としてマークされるため、OpenClaw は画像をプロンプトに自動注入します |
| Reasoning 検出       | model 名ヒューリスティック（`r1`、`reasoning`、`think`）で `reasoning` をマークする                                                                                  |
| トークン制限         | `maxTokens` は OpenClaw が使用するデフォルトの Ollama 最大トークン上限に設定される                                                                                   |
| コスト               | すべてのコストを `0` に設定する                                                                                                                                        |

これにより、カタログをローカル Ollama インスタンスに合わせたまま、手動の model エントリーを避けられます。

```bash
# 利用可能なモデルを確認する
ollama list
openclaw models list
```

新しい model を追加するには、Ollama で pull するだけです。

```bash
ollama pull mistral
```

新しい model は自動的に検出され、利用可能になります。

<Note>
`models.providers.ollama` を明示的に設定した場合、自動検出はスキップされ、model を手動で定義する必要があります。以下の明示的設定セクションを参照してください。
</Note>

## 設定

<Tabs>
  <Tab title="基本（暗黙的検出）">
    Ollama を有効にする最も簡単な方法は、環境変数を使うことです。

    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    `OLLAMA_API_KEY` が設定されていれば、provider エントリー内の `apiKey` は省略でき、OpenClaw が可用性チェック用に補完します。
    </Tip>

  </Tab>

  <Tab title="明示的設定（手動 model）">
    Ollama が別の host/port で動作している場合、特定のコンテキストウィンドウや model リストを強制したい場合、または完全に手動の model 定義を使いたい場合は、明示的設定を使用します。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            apiKey: "ollama-local",
            api: "ollama",
            models: [
              {
                id: "gpt-oss:20b",
                name: "GPT-OSS 20B",
                reasoning: false,
                input: ["text"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 8192,
                maxTokens: 8192 * 10
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="カスタム base URL">
    Ollama が別の host または port で動作している場合（明示的設定は自動検出を無効にするため、model は手動定義してください）:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // /v1 なし - ネイティブ Ollama API URL を使用
            api: "ollama", // ネイティブのツール呼び出し動作を保証するため明示的に設定
          },
        },
      },
    }
    ```

    <Warning>
    URL に `/v1` を追加しないでください。`/v1` パスは OpenAI 互換モードを使用し、そのモードではツール呼び出しが信頼できません。パス接尾辞のないベースの Ollama URL を使用してください。
    </Warning>

  </Tab>
</Tabs>

### モデル選択

設定後は、すべての Ollama モデルが利用可能になります。

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## Ollama Web Search

OpenClaw は、バンドル済み `web_search` provider として **Ollama Web Search** をサポートしています。

| プロパティ   | 詳細                                                                                                               |
| ----------- | ------------------------------------------------------------------------------------------------------------------ |
| Host        | 設定済みの Ollama host を使用（`models.providers.ollama.baseUrl` が設定されている場合はそれ、そうでなければ `http://127.0.0.1:11434`） |
| Auth        | キー不要                                                                                                           |
| Requirement | Ollama が実行中で、`ollama signin` でサインイン済みであること                                                     |

`openclaw onboard` または `openclaw configure --section web` の間に **Ollama Web Search** を選ぶか、次を設定します。

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

<Note>
完全なセットアップと動作の詳細については、[Ollama Web Search](/ja-JP/tools/ollama-search) を参照してください。
</Note>

## 高度な設定

<AccordionGroup>
  <Accordion title="レガシー OpenAI 互換モード">
    <Warning>
    **OpenAI 互換モードではツール呼び出しは信頼できません。** このモードは、プロキシのために OpenAI 形式が必要で、ネイティブのツール呼び出し動作に依存しない場合にのみ使用してください。
    </Warning>

    代わりに OpenAI 互換エンドポイントを使う必要がある場合（たとえば OpenAI 形式のみをサポートするプロキシの背後など）は、`api: "openai-completions"` を明示的に設定します。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // default: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    このモードでは、ストリーミングとツール呼び出しを同時にサポートできない場合があります。model 設定の `params: { streaming: false }` でストリーミングを無効にする必要があるかもしれません。

    Ollama で `api: "openai-completions"` を使用する場合、OpenClaw はデフォルトで `options.num_ctx` を注入するため、Ollama が黙って 4096 コンテキストウィンドウにフォールバックすることを防ぎます。プロキシまたは上流が未知の `options` フィールドを拒否する場合は、この動作を無効にしてください。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="コンテキストウィンドウ">
    自動検出された model では、OpenClaw は利用可能な場合は Ollama が報告するコンテキストウィンドウを使用し、そうでない場合は OpenClaw が使用するデフォルトの Ollama コンテキストウィンドウにフォールバックします。

    明示的な provider 設定では `contextWindow` と `maxTokens` を上書きできます。

    ```json5
    {
      models: {
        providers: {
          ollama: {
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
              }
            ]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Reasoning モデル">
    OpenClaw は、`deepseek-r1`、`reasoning`、`think` などの名前を持つモデルを、デフォルトで reasoning 対応として扱います。

    ```bash
    ollama pull deepseek-r1:32b
    ```

    追加設定は不要です。OpenClaw が自動的にそれらをマークします。

  </Accordion>

  <Accordion title="モデルコスト">
    Ollama は無料でローカル実行されるため、すべてのモデルコストは $0 に設定されます。これは、自動検出されたモデルにも手動定義されたモデルにも適用されます。
  </Accordion>

  <Accordion title="メモリ埋め込み">
    バンドル済み Ollama Plugin は、[memory search](/ja-JP/concepts/memory) 用のメモリ埋め込み provider を登録します。設定済みの Ollama base URL と API キーを使用します。

    | プロパティ      | 値                    |
    | ------------- | --------------------- |
    | デフォルトモデル | `nomic-embed-text`   |
    | Auto-pull     | はい — 埋め込みモデルがローカルに存在しない場合、自動的に pull されます |

    Ollama を memory search の埋め込み provider として選択するには、次のようにします。

    ```json5
    {
      agents: {
        defaults: {
          memorySearch: { provider: "ollama" },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="ストリーミング設定">
    OpenClaw の Ollama 統合は、デフォルトで **ネイティブ Ollama API**（`/api/chat`）を使用し、ストリーミングとツール呼び出しを同時に完全サポートします。特別な設定は不要です。

    <Tip>
    OpenAI 互換エンドポイントを使う必要がある場合は、上記の「レガシー OpenAI 互換モード」セクションを参照してください。そのモードでは、ストリーミングとツール呼び出しが同時に動作しない場合があります。
    </Tip>

  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="Ollama が検出されない">
    Ollama が実行中であること、`OLLAMA_API_KEY`（または auth profile）を設定したこと、そして明示的な `models.providers.ollama` エントリーを**定義していない**ことを確認してください。

    ```bash
    ollama serve
    ```

    API にアクセスできることを確認します。

    ```bash
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="利用可能なモデルがない">
    model が一覧に表示されない場合は、ローカルで model を pull するか、`models.providers.ollama` で明示的に定義してください。

    ```bash
    ollama list  # インストール済みモデルを確認
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # または別の model
    ```

  </Accordion>

  <Accordion title="接続が拒否される">
    Ollama が正しい port で動作しているか確認してください。

    ```bash
    # Ollama が動作しているか確認
    ps aux | grep ollama

    # または Ollama を再起動
    ollama serve
    ```

  </Accordion>
</AccordionGroup>

<Note>
詳細: [トラブルシューティング](/ja-JP/help/troubleshooting) と [FAQ](/ja-JP/help/faq)。
</Note>

## 関連

<CardGroup cols={2}>
  <Card title="モデル provider" href="/ja-JP/concepts/model-providers" icon="layers">
    すべての provider、model ref、フェイルオーバー動作の概要。
  </Card>
  <Card title="モデル選択" href="/ja-JP/concepts/models" icon="brain">
    モデルの選び方と設定方法。
  </Card>
  <Card title="Ollama Web Search" href="/ja-JP/tools/ollama-search" icon="magnifying-glass">
    Ollama を使った Web Search の完全なセットアップと動作の詳細。
  </Card>
  <Card title="設定" href="/ja-JP/gateway/configuration" icon="gear">
    完全な設定リファレンス。
  </Card>
</CardGroup>
