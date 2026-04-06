---
read_when:
    - Anda ingin menggunakan pembuatan image fal di OpenClaw
    - Anda memerlukan alur auth FAL_KEY
    - Anda menginginkan default fal untuk image_generate atau video_generate
summary: Setup pembuatan image dan video fal di OpenClaw
title: fal
x-i18n:
    generated_at: "2026-04-06T03:10:05Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1922907d2c8360c5877a56495323d54bd846d47c27a801155e3d11e3f5706fbd
    source_path: providers/fal.md
    workflow: 15
---

# fal

OpenClaw menyediakan provider `fal` bawaan untuk pembuatan image dan video yang dihosting.

- Provider: `fal`
- Auth: `FAL_KEY` (kanonis; `FAL_API_KEY` juga berfungsi sebagai fallback)
- API: endpoint model fal

## Mulai cepat

1. Tetapkan API key:

```bash
openclaw onboard --auth-choice fal-api-key
```

2. Tetapkan model image default:

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

## Pembuatan image

Provider pembuatan image `fal` bawaan secara default menggunakan
`fal/fal-ai/flux/dev`.

- Generate: hingga 4 image per permintaan
- Mode edit: diaktifkan, 1 image referensi
- Mendukung `size`, `aspectRatio`, dan `resolution`
- Peringatan edit saat ini: endpoint edit image fal **tidak** mendukung
  override `aspectRatio`

Untuk menggunakan fal sebagai provider image default:

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

## Pembuatan video

Provider pembuatan video `fal` bawaan secara default menggunakan
`fal/fal-ai/minimax/video-01-live`.

- Mode: text-to-video dan alur referensi image tunggal
- Runtime: alur submit/status/result berbasis antrean untuk job yang berjalan lama

Untuk menggunakan fal sebagai provider video default:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/fal-ai/minimax/video-01-live",
      },
    },
  },
}
```

## Terkait

- [Pembuatan Image](/id/tools/image-generation)
- [Pembuatan Video](/tools/video-generation)
- [Referensi Konfigurasi](/id/gateway/configuration-reference#agent-defaults)
