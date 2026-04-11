---
read_when:
    - エージェント経由で動画を生成する場合
    - 動画生成プロバイダーとモデルを設定する場合
    - '`video_generate`ツールのパラメータを理解する場合'
summary: 12個のプロバイダーバックエンドを使って、テキスト、画像、または既存の動画から動画を生成する
title: 動画生成
x-i18n:
    generated_at: "2026-04-11T02:48:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6848d03ef578181902517d068e8d9fe2f845e572a90481bbdf7bd9f1c591f245
    source_path: tools/video-generation.md
    workflow: 15
---

# 動画生成

OpenClawエージェントは、テキストプロンプト、参照画像、または既存の動画から動画を生成できます。12個のプロバイダーバックエンドがサポートされており、それぞれ異なるモデルオプション、入力モード、機能セットを持っています。エージェントは、設定と利用可能なAPIキーに基づいて適切なプロバイダーを自動的に選択します。

<Note>
少なくとも1つの動画生成プロバイダーが利用可能な場合にのみ、`video_generate`ツールが表示されます。エージェントツール内に表示されない場合は、プロバイダーAPIキーを設定するか、`agents.defaults.videoGenerationModel`を構成してください。
</Note>

OpenClawは、動画生成を3つのランタイムモードとして扱います。

- 参照メディアなしのtext-to-videoリクエストには`generate`
- リクエストに1つ以上の参照画像が含まれる場合は`imageToVideo`
- リクエストに1つ以上の参照動画が含まれる場合は`videoToVideo`

プロバイダーはこれらのモードの任意の部分集合をサポートできます。ツールは送信前にアクティブなモードを検証し、`action=list`でサポートされているモードを報告します。

## クイックスタート

1. サポートされている任意のプロバイダーのAPIキーを設定します:

```bash
export GEMINI_API_KEY="your-key"
```

2. 必要に応じてデフォルトモデルを固定します:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
```

3. エージェントに依頼します:

> 夕焼けの中、やさしいロブスターがサーフィンしている5秒のシネマティックな動画を生成して。

エージェントは自動的に`video_generate`を呼び出します。ツールallowlistは不要です。

## 動画を生成すると何が起こるか

動画生成は非同期です。セッション内でエージェントが`video_generate`を呼び出すと:

1. OpenClawはリクエストをプロバイダーに送信し、すぐにタスクIDを返します。
2. プロバイダーはバックグラウンドでジョブを処理します（通常はプロバイダーや解像度に応じて30秒から5分）。
3. 動画の準備ができると、OpenClawは内部完了イベントで同じセッションを起こします。
4. エージェントは完成した動画を元の会話に投稿します。

ジョブが進行中の間、同じセッション内で重複する`video_generate`呼び出しは、新しい生成を開始する代わりに現在のタスク状態を返します。CLIから進行状況を確認するには、`openclaw tasks list`または`openclaw tasks show <taskId>`を使ってください。

セッションに紐づくエージェント実行以外（たとえば直接ツール呼び出し）では、ツールはインライン生成へフォールバックし、同じターン内で最終メディアパスを返します。

### タスクライフサイクル

各`video_generate`リクエストは4つの状態を移動します。

1. **queued** -- タスクが作成され、プロバイダーが受け付けるのを待っている状態。
2. **running** -- プロバイダーが処理中（通常はプロバイダーや解像度に応じて30秒から5分）。
3. **succeeded** -- 動画が準備完了。エージェントが起きて会話に投稿します。
4. **failed** -- プロバイダーエラーまたはタイムアウト。エージェントがエラー詳細付きで起きます。

CLIから状態を確認します:

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

重複防止: 現在のセッションですでに動画タスクが`queued`または`running`の場合、`video_generate`は新しいタスクを開始する代わりに既存タスクの状態を返します。新しい生成をトリガーせず明示的に確認するには、`action: "status"`を使ってください。

## サポートされているプロバイダー

| プロバイダー | デフォルトモデル                | テキスト | 画像参照          | 動画参照         | APIキー                                  |
| ------------ | ------------------------------- | -------- | ----------------- | ---------------- | ---------------------------------------- |
| Alibaba      | `wan2.6-t2v`                    | Yes      | Yes（リモートURL） | Yes（リモートURL） | `MODELSTUDIO_API_KEY`                    |
| BytePlus     | `seedance-1-0-lite-t2v-250428`  | Yes      | 1 image           | No               | `BYTEPLUS_API_KEY`                       |
| ComfyUI      | `workflow`                      | Yes      | 1 image           | No               | `COMFY_API_KEY` or `COMFY_CLOUD_API_KEY` |
| fal          | `fal-ai/minimax/video-01-live`  | Yes      | 1 image           | No               | `FAL_KEY`                                |
| Google       | `veo-3.1-fast-generate-preview` | Yes      | 1 image           | 1 video          | `GEMINI_API_KEY`                         |
| MiniMax      | `MiniMax-Hailuo-2.3`            | Yes      | 1 image           | No               | `MINIMAX_API_KEY`                        |
| OpenAI       | `sora-2`                        | Yes      | 1 image           | 1 video          | `OPENAI_API_KEY`                         |
| Qwen         | `wan2.6-t2v`                    | Yes      | Yes（リモートURL） | Yes（リモートURL） | `QWEN_API_KEY`                           |
| Runway       | `gen4.5`                        | Yes      | 1 image           | 1 video          | `RUNWAYML_API_SECRET`                    |
| Together     | `Wan-AI/Wan2.2-T2V-A14B`        | Yes      | 1 image           | No               | `TOGETHER_API_KEY`                       |
| Vydra        | `veo3`                          | Yes      | 1 image（`kling`） | No               | `VYDRA_API_KEY`                          |
| xAI          | `grok-imagine-video`            | Yes      | 1 image           | 1 video          | `XAI_API_KEY`                            |

一部のプロバイダーは、追加または代替のAPIキーenv varも受け付けます。詳細は個別の[プロバイダーページ](#related)を参照してください。

利用可能なプロバイダー、モデル、ランタイムモードをランタイムで確認するには、`video_generate action=list`を実行してください。

### 宣言済みcapabilityマトリクス

これは、`video_generate`、契約テスト、および共有live sweepで使用される明示的なモード契約です。

| プロバイダー | `generate` | `imageToVideo` | `videoToVideo` | 現在の共有live lane                                                                                                                       |
| ------------ | ---------- | -------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba      | Yes        | Yes            | Yes            | `generate`、`imageToVideo`; このプロバイダーはリモート`http(s)`動画URLを必要とするため`videoToVideo`はスキップ                           |
| BytePlus     | Yes        | Yes            | No             | `generate`、`imageToVideo`                                                                                                                |
| ComfyUI      | Yes        | Yes            | No             | 共有sweepには含まれない; workflow固有のカバレッジはComfyテスト側にある                                                                   |
| fal          | Yes        | Yes            | No             | `generate`、`imageToVideo`                                                                                                                |
| Google       | Yes        | Yes            | Yes            | `generate`、`imageToVideo`; 現在のbufferベースGemini/Veo sweepではその入力を受け付けないため、共有`videoToVideo`はスキップ              |
| MiniMax      | Yes        | Yes            | No             | `generate`、`imageToVideo`                                                                                                                |
| OpenAI       | Yes        | Yes            | Yes            | `generate`、`imageToVideo`; このorg/入力パスは現在プロバイダー側のinpaint/remixアクセスを必要とするため、共有`videoToVideo`はスキップ    |
| Qwen         | Yes        | Yes            | Yes            | `generate`、`imageToVideo`; このプロバイダーはリモート`http(s)`動画URLを必要とするため`videoToVideo`はスキップ                           |
| Runway       | Yes        | Yes            | Yes            | `generate`、`imageToVideo`; `videoToVideo`は選択モデルが`runway/gen4_aleph`のときのみ実行                                               |
| Together     | Yes        | Yes            | No             | `generate`、`imageToVideo`                                                                                                                |
| Vydra        | Yes        | Yes            | No             | `generate`; 同梱の`veo3`はtext専用で、同梱の`kling`はリモート画像URLを必要とするため、共有`imageToVideo`はスキップ                      |
| xAI          | Yes        | Yes            | Yes            | `generate`、`imageToVideo`; このプロバイダーは現在リモートMP4 URLを必要とするため`videoToVideo`はスキップ                                |

## ツールパラメータ

### 必須

| パラメータ | 型     | 説明                                                           |
| ---------- | ------ | -------------------------------------------------------------- |
| `prompt`   | string | 生成する動画のテキスト説明（`action: "generate"`で必須）      |

### コンテンツ入力

| パラメータ | 型       | 説明                                 |
| ---------- | -------- | ------------------------------------ |
| `image`    | string   | 単一の参照画像（パスまたはURL）      |
| `images`   | string[] | 複数の参照画像（最大5件）            |
| `video`    | string   | 単一の参照動画（パスまたはURL）      |
| `videos`   | string[] | 複数の参照動画（最大4件）            |

### スタイル制御

| パラメータ        | 型      | 説明                                                                     |
| ----------------- | ------- | ------------------------------------------------------------------------ |
| `aspectRatio`     | string  | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` |
| `resolution`      | string  | `480P`, `720P`, `768P`, または`1080P`                                   |
| `durationSeconds` | number  | 目標の長さ（秒）。最も近いプロバイダー対応値に丸められる                |
| `size`            | string  | プロバイダーが対応している場合のサイズヒント                            |
| `audio`           | boolean | 対応時に生成音声を有効化                                                |
| `watermark`       | boolean | 対応時にプロバイダー透かしを切り替え                                    |

### 高度な設定

| パラメータ | 型     | 説明                                              |
| ---------- | ------ | ------------------------------------------------- |
| `action`   | string | `"generate"`（デフォルト）、`"status"`、または`"list"` |
| `model`    | string | プロバイダー/モデルのoverride（例: `runway/gen4.5`） |
| `filename` | string | 出力ファイル名ヒント                              |

すべてのプロバイダーがすべてのパラメータをサポートしているわけではありません。OpenClawはすでにdurationを最も近いプロバイダー対応値へ正規化し、フォールバック先プロバイダーが異なる制御サーフェスを公開している場合は、sizeからaspect-ratioへのような変換済みgeometry hintも再対応付けします。本当に未対応のoverrideはbest-effortで無視され、ツール結果内のwarningとして報告されます。厳格なcapability制限（参照入力が多すぎるなど）は送信前に失敗します。

ツール結果は適用された設定を報告します。OpenClawがプロバイダーフォールバック中にdurationやgeometryを再対応付けした場合、返される`durationSeconds`、`size`、`aspectRatio`、`resolution`値は送信された内容を反映し、`details.normalization`には要求値から適用値への変換が記録されます。

参照入力はランタイムモードも選択します。

- 参照メディアなし: `generate`
- 何らかの画像参照あり: `imageToVideo`
- 何らかの動画参照あり: `videoToVideo`

画像参照と動画参照の混在は、安定した共有capabilityサーフェスではありません。1リクエストにつき1種類の参照タイプを推奨します。

## アクション

- **generate**（デフォルト） -- 与えられたプロンプトと任意の参照入力から動画を生成します。
- **status** -- 現在のセッションで進行中の動画タスク状態を、新しい生成を開始せずに確認します。
- **list** -- 利用可能なプロバイダー、モデル、およびそのcapabilityを表示します。

## モデル選択

動画生成時、OpenClawは次の順序でモデルを解決します。

1. **`model`ツールパラメータ** -- エージェントが呼び出し時に指定した場合。
2. **`videoGenerationModel.primary`** -- 設定から。
3. **`videoGenerationModel.fallbacks`** -- 順番に試行。
4. **自動検出** -- 有効な認証を持つプロバイダーを使用し、現在のデフォルトプロバイダーから始め、その後は残りのプロバイダーをアルファベット順で試します。

プロバイダーが失敗した場合、次の候補が自動的に試されます。すべての候補が失敗した場合、エラーには各試行の詳細が含まれます。

動画生成で明示的な`model`、`primary`、`fallbacks`エントリだけを使いたい場合は、`agents.defaults.mediaGenerationAutoProviderFallback: false`を設定してください。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
      },
    },
  },
}
```

fal上のHeyGen video-agentは次のように固定できます。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/fal-ai/heygen/v2/video-agent",
      },
    },
  },
}
```

fal上のSeedance 2.0は次のように固定できます。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
      },
    },
  },
}
```

## プロバイダーメモ

| プロバイダー | メモ                                                                                                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba      | DashScope/Model Studioの非同期エンドポイントを使用します。参照画像と参照動画はリモート`http(s)` URLである必要があります。                                                |
| BytePlus     | 単一の画像参照のみ。                                                                                                                                                      |
| ComfyUI      | workflow駆動のローカルまたはクラウド実行。構成されたグラフを通じてtext-to-videoとimage-to-videoをサポートします。                                                        |
| fal          | 長時間実行ジョブにはqueueベースのフローを使用します。単一の画像参照のみ。HeyGen video-agentとSeedance 2.0のtext-to-videoおよびimage-to-videoモデル参照を含みます。       |
| Google       | Gemini/Veoを使用します。1つの画像または1つの動画参照をサポートします。                                                                                                    |
| MiniMax      | 単一の画像参照のみ。                                                                                                                                                      |
| OpenAI       | `size` overrideのみが転送されます。その他のスタイルoverride（`aspectRatio`、`resolution`、`audio`、`watermark`）はwarning付きで無視されます。                            |
| Qwen         | Alibabaと同じDashScopeバックエンドです。参照入力はリモート`http(s)` URLである必要があり、ローカルファイルは事前に拒否されます。                                           |
| Runway       | data URI経由でローカルファイルをサポートします。video-to-videoには`runway/gen4_aleph`が必要です。text-only実行では`16:9`および`9:16`のaspect ratioを公開します。          |
| Together     | 単一の画像参照のみ。                                                                                                                                                      |
| Vydra        | 認証が落ちるリダイレクトを避けるため、`https://www.vydra.ai/api/v1`を直接使用します。`veo3`はtext-to-video専用として同梱され、`kling`にはリモート画像URLが必要です。      |
| xAI          | text-to-video、image-to-video、およびリモート動画edit/extendフローをサポートします。                                                                                     |

## プロバイダーcapabilityモード

共有動画生成契約では、フラットな集計制限だけでなく、プロバイダーがモード固有のcapabilityを宣言できるようになりました。新しいプロバイダー実装では、明示的なモードブロックを優先してください。

```typescript
capabilities: {
  generate: {
    maxVideos: 1,
    maxDurationSeconds: 10,
    supportsResolution: true,
  },
  imageToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputImages: 1,
    maxDurationSeconds: 5,
  },
  videoToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputVideos: 1,
    maxDurationSeconds: 5,
  },
}
```

`maxInputImages`や`maxInputVideos`のようなフラットな集計フィールドだけでは、transform-modeサポートを告知するには不十分です。liveテスト、契約テスト、および共有`video_generate`ツールがモードサポートを決定的に検証できるよう、プロバイダーは`generate`、`imageToVideo`、`videoToVideo`を明示的に宣言するべきです。

## liveテスト

共有の同梱プロバイダー向けオプトインliveカバレッジ:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts
```

リポジトリラッパー:

```bash
pnpm test:live:media video
```

このliveファイルは、不足しているプロバイダーenv varを`~/.profile`から読み込み、デフォルトで保存済み認証プロファイルよりlive/env APIキーを優先し、ローカルメディアで安全に実行できる宣言済みモードを実行します。

- sweep内のすべてのプロバイダーに対する`generate`
- `capabilities.imageToVideo.enabled`の場合の`imageToVideo`
- `capabilities.videoToVideo.enabled`で、かつプロバイダー/モデルが共有sweep内でbufferベースのローカル動画入力を受け付ける場合の`videoToVideo`

現在、共有`videoToVideo` live laneがカバーするのは次です。

- `runway`（`runway/gen4_aleph`を選択した場合のみ）

## 設定

OpenClaw設定でデフォルトの動画生成モデルを設定します。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

またはCLI経由:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "qwen/wan2.6-t2v"
```

## 関連

- [ツール概要](/ja-JP/tools)
- [バックグラウンドタスク](/ja-JP/automation/tasks) -- 非同期動画生成のタスク追跡
- [Alibaba Model Studio](/ja-JP/providers/alibaba)
- [BytePlus](/ja-JP/concepts/model-providers#byteplus-international)
- [ComfyUI](/ja-JP/providers/comfy)
- [fal](/ja-JP/providers/fal)
- [Google (Gemini)](/ja-JP/providers/google)
- [MiniMax](/ja-JP/providers/minimax)
- [OpenAI](/ja-JP/providers/openai)
- [Qwen](/ja-JP/providers/qwen)
- [Runway](/ja-JP/providers/runway)
- [Together AI](/ja-JP/providers/together)
- [Vydra](/ja-JP/providers/vydra)
- [xAI](/ja-JP/providers/xai)
- [設定リファレンス](/ja-JP/gateway/configuration-reference#agent-defaults)
- [モデル](/ja-JP/concepts/models)
