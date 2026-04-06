---
read_when:
    - Anda ingin menggunakan video generation Runway di OpenClaw
    - Anda memerlukan penyiapan API key/env Runway
    - Anda ingin menjadikan Runway sebagai provider video default
summary: Penyiapan video generation Runway di OpenClaw
title: Runway
x-i18n:
    generated_at: "2026-04-06T03:10:29Z"
    model: gpt-5.4
    provider: openai
    source_hash: bc615d1a26f7a4b890d29461e756690c858ecb05024cf3c4d508218022da6e76
    source_path: providers/runway.md
    workflow: 15
---

# Runway

OpenClaw menyediakan provider `runway` bawaan untuk video generation yang dihosting.

- ID provider: `runway`
- Auth: `RUNWAYML_API_SECRET` (kanonis) atau `RUNWAY_API_KEY`
- API: video generation berbasis task Runway (polling `GET /v1/tasks/{id}`)

## Mulai cepat

1. Atur API key:

```bash
openclaw onboard --auth-choice runway-api-key
```

2. Atur Runway sebagai provider video default:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "runway/gen4.5"
```

3. Minta agen untuk membuat video. Runway akan digunakan secara otomatis.

## Mode yang didukung

| Mode           | Model              | Input referensi          |
| -------------- | ------------------ | ------------------------ |
| Teks-ke-video  | `gen4.5` (default) | Tidak ada                |
| Gambar-ke-video | `gen4.5`          | 1 gambar lokal atau remote |
| Video-ke-video | `gen4_aleph`       | 1 video lokal atau remote |

- Referensi gambar dan video lokal didukung melalui data URI.
- Video-ke-video saat ini secara khusus memerlukan `runway/gen4_aleph`.
- Run berbasis teks saja saat ini mengekspos rasio aspek `16:9` dan `9:16`.

## Konfigurasi

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## Terkait

- [Video Generation](/tools/video-generation) -- parameter tool bersama, pemilihan provider, dan perilaku async
- [Configuration Reference](/id/gateway/configuration-reference#agent-defaults)
