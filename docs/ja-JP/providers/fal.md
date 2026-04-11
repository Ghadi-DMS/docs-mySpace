---
read_when:
    - OpenClawでfal画像生成を使用したいです
    - '`FAL_KEY`認証フローが必要です'
    - '`image_generate`または`video_generate`向けのfalデフォルト設定が必要です'
summary: OpenClawでのfal画像生成および動画生成の設定
title: fal
x-i18n:
    generated_at: "2026-04-11T02:48:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9bfe4f69124e922a79a516a1bd78f0c00f7a45f3c6f68b6d39e0d196fa01beb3
    source_path: providers/fal.md
    workflow: 15
---

# fal

OpenClawには、ホスト型の画像生成および動画生成向けにバンドル版`fal`プロバイダーが含まれています。

- プロバイダー: `fal`
- 認証: `FAL_KEY`（正規。`FAL_API_KEY`もフォールバックとして動作）
- API: falモデルエンドポイント

## クイックスタート

1. APIキーを設定します:

```bash
openclaw onboard --auth-choice fal-api-key
```

2. デフォルトの画像モデルを設定します:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## 画像生成

バンドル版`fal`画像生成プロバイダーのデフォルトは
`fal/fal-ai/flux/dev`です。

- 生成: 1回のリクエストで最大4枚の画像
- 編集モード: 有効、参照画像は1枚
- `size`、`aspectRatio`、`resolution`をサポート
- 現在の編集時の注意点: fal画像編集エンドポイントは
  `aspectRatio`上書きをサポートしていません

falをデフォルト画像プロバイダーとして使用するには:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## 動画生成

バンドル版`fal`動画生成プロバイダーのデフォルトは
`fal/fal-ai/minimax/video-01-live`です。

- モード: テキストから動画、および単一画像参照フロー
- ランタイム: 長時間実行ジョブ向けのキュー対応submit/status/resultフロー
- HeyGen video-agentモデル参照:
  - `fal/fal-ai/heygen/v2/video-agent`
- Seedance 2.0モデル参照:
  - `fal/bytedance/seedance-2.0/fast/text-to-video`
  - `fal/bytedance/seedance-2.0/fast/image-to-video`
  - `fal/bytedance/seedance-2.0/text-to-video`
  - `fal/bytedance/seedance-2.0/image-to-video`

Seedance 2.0をデフォルト動画モデルとして使用するには:

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

HeyGen video-agentをデフォルト動画モデルとして使用するには:

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

## 関連

- [Image Generation](/ja-JP/tools/image-generation)
- [Video Generation](/ja-JP/tools/video-generation)
- [Configuration Reference](/ja-JP/gateway/configuration-reference#agent-defaults)
