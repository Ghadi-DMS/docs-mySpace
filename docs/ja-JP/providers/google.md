---
read_when:
    - OpenClaw で Google Gemini モデルを使用したい場合
    - API キーまたは OAuth 認証フローが必要な場合
summary: Google Gemini のセットアップ（API キー + OAuth、画像生成、メディア理解、Web 検索）
title: Google（Gemini）
x-i18n:
    generated_at: "2026-04-12T23:31:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 64b848add89061b208a5d6b19d206c433cace5216a0ca4b63d56496aecbde452
    source_path: providers/google.md
    workflow: 15
---

# Google（Gemini）

Google Plugin は、Google AI Studio を通じた Gemini モデルへのアクセスに加えて、
画像生成、メディア理解（image/audio/video）、および Gemini Grounding による Web 検索を提供します。

- Provider: `google`
- Auth: `GEMINI_API_KEY` または `GOOGLE_API_KEY`
- API: Google Gemini API
- Alternative provider: `google-gemini-cli`（OAuth）

## はじめに

希望する認証方法を選び、セットアップ手順に従ってください。

<Tabs>
  <Tab title="API key">
    **最適な用途:** Google AI Studio を通じた標準的な Gemini API アクセス。

    <Steps>
      <Step title="Run onboarding">
        ```bash
        openclaw onboard --auth-choice gemini-api-key
        ```

        または、キーを直接渡します:

        ```bash
        openclaw onboard --non-interactive \
          --mode local \
          --auth-choice gemini-api-key \
          --gemini-api-key "$GEMINI_API_KEY"
        ```
      </Step>
      <Step title="Set a default model">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "google/gemini-3.1-pro-preview" },
            },
          },
        }
        ```
      </Step>
      <Step title="Verify the model is available">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    <Tip>
    環境変数 `GEMINI_API_KEY` と `GOOGLE_API_KEY` はどちらも使用できます。すでに設定してある方を使ってください。
    </Tip>

  </Tab>

  <Tab title="Gemini CLI (OAuth)">
    **最適な用途:** 別の API キーではなく、既存の Gemini CLI ログインを PKCE OAuth 経由で再利用したい場合。

    <Warning>
    `google-gemini-cli` provider は非公式の統合です。この方法で OAuth を使用すると、アカウント制限がかかったという報告があります。自己責任で使用してください。
    </Warning>

    <Steps>
      <Step title="Install the Gemini CLI">
        ローカルの `gemini` コマンドが `PATH` 上で利用可能である必要があります。

        ```bash
        # Homebrew
        brew install gemini-cli

        # or npm
        npm install -g @google/gemini-cli
        ```

        OpenClaw は、Homebrew インストールとグローバル npm インストールの両方をサポートしており、一般的な Windows/npm レイアウトも含みます。
      </Step>
      <Step title="Log in via OAuth">
        ```bash
        openclaw models auth login --provider google-gemini-cli --set-default
        ```
      </Step>
      <Step title="Verify the model is available">
        ```bash
        openclaw models list --provider google-gemini-cli
        ```
      </Step>
    </Steps>

    - Default model: `google-gemini-cli/gemini-3-flash-preview`
    - Alias: `gemini-cli`

    **環境変数:**

    - `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
    - `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

    （または `GEMINI_CLI_*` バリアント。）

    <Note>
    ログイン後に Gemini CLI OAuth リクエストが失敗する場合は、gateway host で `GOOGLE_CLOUD_PROJECT` または `GOOGLE_CLOUD_PROJECT_ID` を設定して再試行してください。
    </Note>

    <Note>
    ブラウザフローが始まる前にログインが失敗する場合は、ローカルの `gemini` コマンドがインストールされており、`PATH` 上にあることを確認してください。
    </Note>

    OAuth 専用の `google-gemini-cli` provider は、別個の text-inference
    サーフェスです。画像生成、メディア理解、Gemini Grounding は `google` provider ID のままです。

  </Tab>
</Tabs>

## 機能

| Capability             | Supported         |
| ---------------------- | ----------------- |
| Chat completions       | はい              |
| Image generation       | はい              |
| Music generation       | はい              |
| Image understanding    | はい              |
| Audio transcription    | はい              |
| Video understanding    | はい              |
| Web search (Grounding) | はい              |
| Thinking/reasoning     | はい（Gemini 3.1+）|
| Gemma 4 models         | はい              |

<Tip>
Gemma 4 モデル（例: `gemma-4-26b-a4b-it`）は thinking mode をサポートします。OpenClaw は、Gemma 4 向けに `thinkingBudget` をサポートされる Google の `thinkingLevel` に書き換えます。thinking を `off` に設定した場合は、`MINIMAL` にマッピングせず、thinking 無効のまま維持されます。
</Tip>

## 画像生成

バンドルされた `google` 画像生成 provider のデフォルトは
`google/gemini-3.1-flash-image-preview` です。

- `google/gemini-3-pro-image-preview` もサポート
- 生成: 1 リクエストあたり最大 4 枚の画像
- 編集モード: 有効、最大 5 枚の入力画像
- ジオメトリ制御: `size`、`aspectRatio`、`resolution`

Google をデフォルトの画像 provider として使用するには:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

<Note>
共有 tool パラメーター、provider 選択、フェイルオーバー動作については、[Image Generation](/ja-JP/tools/image-generation) を参照してください。
</Note>

## 動画生成

バンドルされた `google` Plugin は、共有の
`video_generate` tool を通じて動画生成も登録します。

- Default video model: `google/veo-3.1-fast-generate-preview`
- モード: text-to-video、image-to-video、single-video reference フロー
- `aspectRatio`、`resolution`、`audio` をサポート
- 現在の duration 上限: **4〜8 秒**

Google をデフォルトの video provider として使用するには:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

<Note>
共有 tool パラメーター、provider 選択、フェイルオーバー動作については、[Video Generation](/ja-JP/tools/video-generation) を参照してください。
</Note>

## 音楽生成

バンドルされた `google` Plugin は、共有の
`music_generate` tool を通じて音楽生成も登録します。

- Default music model: `google/lyria-3-clip-preview`
- `google/lyria-3-pro-preview` もサポート
- prompt 制御: `lyrics` と `instrumental`
- 出力形式: デフォルトは `mp3`、加えて `google/lyria-3-pro-preview` では `wav`
- 参照入力: 最大 10 枚の画像
- session ベースの実行は、`action: "status"` を含む共有 task/status フローを通じて分離実行されます

Google をデフォルトの音楽 provider として使用するには:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

<Note>
共有 tool パラメーター、provider 選択、フェイルオーバー動作については、[Music Generation](/ja-JP/tools/music-generation) を参照してください。
</Note>

## 詳細設定

<AccordionGroup>
  <Accordion title="Direct Gemini cache reuse">
    直接の Gemini API 実行（`api: "google-generative-ai"`）では、OpenClaw は設定された `cachedContent` ハンドルを Gemini リクエストにそのまま渡します。

    - `cachedContent` または旧来の `cached_content` のいずれかで、モデルごとまたはグローバルな params を設定します
    - 両方存在する場合は `cachedContent` が優先されます
    - 値の例: `cachedContents/prebuilt-context`
    - Gemini の cache hit 使用量は、上流の `cachedContentTokenCount` から OpenClaw の `cacheRead` に正規化されます

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "google/gemini-2.5-pro": {
              params: {
                cachedContent: "cachedContents/prebuilt-context",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Gemini CLI JSON usage notes">
    `google-gemini-cli` OAuth provider を使用する場合、OpenClaw は CLI JSON 出力を次のように正規化します:

    - 返信テキストは CLI JSON の `response` フィールドから取得されます。
    - CLI の `usage` が空の場合、使用量は `stats` にフォールバックします。
    - `stats.cached` は OpenClaw の `cacheRead` に正規化されます。
    - `stats.input` がない場合、OpenClaw は `stats.input_tokens - stats.cached` から入力トークン数を導出します。

  </Accordion>

  <Accordion title="Environment and daemon setup">
    Gateway が daemon（launchd/systemd）として実行される場合は、`GEMINI_API_KEY` がそのプロセスから利用可能であることを確認してください（たとえば `~/.openclaw/.env` または `env.shellEnv` 内）。
  </Accordion>
</AccordionGroup>

## 関連

<CardGroup cols={2}>
  <Card title="Model selection" href="/ja-JP/concepts/model-providers" icon="layers">
    provider、model ref、フェイルオーバー動作の選び方。
  </Card>
  <Card title="Image generation" href="/ja-JP/tools/image-generation" icon="image">
    共有画像 tool パラメーターと provider 選択。
  </Card>
  <Card title="Video generation" href="/ja-JP/tools/video-generation" icon="video">
    共有 video tool パラメーターと provider 選択。
  </Card>
  <Card title="Music generation" href="/ja-JP/tools/music-generation" icon="music">
    共有音楽 tool パラメーターと provider 選択。
  </Card>
</CardGroup>
