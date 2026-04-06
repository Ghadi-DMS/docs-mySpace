---
read_when:
    - Anda ingin menggunakan model Google Gemini dengan OpenClaw
    - Anda memerlukan alur auth API key
summary: Setup Google Gemini (API key, pembuatan image, pemahaman media, pencarian web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-06T03:10:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 358d33a68275b01ebd916a3621dd651619cb9a1d062e2fb6196a7f3c501c015a
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Plugin Google menyediakan akses ke model Gemini melalui Google AI Studio, serta
pembuatan image, pemahaman media (image/audio/video), dan pencarian web melalui
Gemini Grounding.

- Provider: `google`
- Auth: `GEMINI_API_KEY` atau `GOOGLE_API_KEY`
- API: Google Gemini API

## Mulai cepat

1. Tetapkan API key:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Tetapkan model default:

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## Contoh non-interaktif

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## Kapabilitas

| Capability             | Supported         |
| ---------------------- | ----------------- |
| Chat completions       | Ya                |
| Image generation       | Ya                |
| Music generation       | Ya                |
| Image understanding    | Ya                |
| Audio transcription    | Ya                |
| Video understanding    | Ya                |
| Web search (Grounding) | Ya                |
| Thinking/reasoning     | Ya (Gemini 3.1+)  |

## Penggunaan ulang cache Gemini langsung

Untuk run API Gemini langsung (`api: "google-generative-ai"`), OpenClaw kini
meneruskan handle `cachedContent` yang dikonfigurasi ke permintaan Gemini.

- Konfigurasikan param per-model atau global dengan
  `cachedContent` atau `cached_content` legacy
- Jika keduanya ada, `cachedContent` yang diprioritaskan
- Contoh nilai: `cachedContents/prebuilt-context`
- Penggunaan cache-hit Gemini dinormalisasi ke OpenClaw `cacheRead` dari
  `cachedContentTokenCount` upstream

Contoh:

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

## Pembuatan image

Provider pembuatan image `google` bawaan secara default menggunakan
`google/gemini-3.1-flash-image-preview`.

- Juga mendukung `google/gemini-3-pro-image-preview`
- Generate: hingga 4 image per permintaan
- Mode edit: diaktifkan, hingga 5 image input
- Kontrol geometri: `size`, `aspectRatio`, dan `resolution`

Pembuatan image, pemahaman media, dan Gemini Grounding semuanya tetap berada pada
id provider `google`.

Untuk menggunakan Google sebagai provider image default:

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

Lihat [Pembuatan Image](/id/tools/image-generation) untuk parameter tool bersama,
pemilihan provider, dan perilaku failover.

## Pembuatan video

Plugin `google` bawaan juga mendaftarkan pembuatan video melalui tool bersama
`video_generate`.

- Model video default: `google/veo-3.1-fast-generate-preview`
- Mode: text-to-video, image-to-video, dan alur referensi video tunggal
- Mendukung `aspectRatio`, `resolution`, dan `audio`
- Clamp durasi saat ini: **4 hingga 8 detik**

Untuk menggunakan Google sebagai provider video default:

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

Lihat [Pembuatan Video](/tools/video-generation) untuk parameter tool bersama,
pemilihan provider, dan perilaku failover.

## Pembuatan musik

Plugin `google` bawaan juga mendaftarkan pembuatan musik melalui tool bersama
`music_generate`.

- Model musik default: `google/lyria-3-clip-preview`
- Juga mendukung `google/lyria-3-pro-preview`
- Kontrol prompt: `lyrics` dan `instrumental`
- Format output: `mp3` secara default, serta `wav` pada `google/lyria-3-pro-preview`
- Input referensi: hingga 10 image
- Run berbasis sesi dipisahkan melalui alur task/status bersama, termasuk `action: "status"`

Untuk menggunakan Google sebagai provider musik default:

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

Lihat [Pembuatan Musik](/tools/music-generation) untuk parameter tool bersama,
pemilihan provider, dan perilaku failover.

## Catatan lingkungan

Jika Gateway berjalan sebagai daemon (launchd/systemd), pastikan `GEMINI_API_KEY`
tersedia untuk proses tersebut (misalnya, di `~/.openclaw/.env` atau melalui
`env.shellEnv`).
