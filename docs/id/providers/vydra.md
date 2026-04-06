---
read_when:
    - Anda ingin media generation Vydra di OpenClaw
    - Anda memerlukan panduan penyiapan API key Vydra
summary: Gunakan image, video, dan speech Vydra di OpenClaw
title: Vydra
x-i18n:
    generated_at: "2026-04-06T03:10:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0fe999e8a5414b8a31a6d7d127bc6bcfc3b4492b8f438ab17dfa9680c5b079b7
    source_path: providers/vydra.md
    workflow: 15
---

# Vydra

Bundled plugin Vydra menambahkan:

- image generation melalui `vydra/grok-imagine`
- video generation melalui `vydra/veo3` dan `vydra/kling`
- sintesis speech melalui rute TTS Vydra berbasis ElevenLabs

OpenClaw menggunakan `VYDRA_API_KEY` yang sama untuk ketiga kapabilitas tersebut.

## URL dasar penting

Gunakan `https://www.vydra.ai/api/v1`.

Host apex Vydra (`https://vydra.ai/api/v1`) saat ini mengarahkan ke `www`. Beberapa klien HTTP menghapus `Authorization` pada redirect lintas host tersebut, yang mengubah API key valid menjadi kegagalan auth yang menyesatkan. Bundled plugin menggunakan URL dasar `www` secara langsung untuk menghindari hal ini.

## Penyiapan

Onboarding interaktif:

```bash
openclaw onboard --auth-choice vydra-api-key
```

Atau atur env var secara langsung:

```bash
export VYDRA_API_KEY="vydra_live_..."
```

## Image generation

Model image default:

- `vydra/grok-imagine`

Atur sebagai provider image default:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "vydra/grok-imagine",
      },
    },
  },
}
```

Dukungan bundled saat ini hanya untuk teks-ke-gambar. Rute edit hosted Vydra mengharapkan URL gambar remote, dan OpenClaw belum menambahkan bridge upload khusus Vydra di bundled plugin.

Lihat [Image Generation](/id/tools/image-generation) untuk perilaku tool bersama.

## Video generation

Model video yang terdaftar:

- `vydra/veo3` untuk teks-ke-video
- `vydra/kling` untuk gambar-ke-video

Atur Vydra sebagai provider video default:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "vydra/veo3",
      },
    },
  },
}
```

Catatan:

- `vydra/veo3` dibundel hanya sebagai teks-ke-video.
- `vydra/kling` saat ini memerlukan referensi URL gambar remote. Unggahan file lokal ditolak sejak awal.
- Bundled plugin tetap konservatif dan tidak meneruskan knob style yang tidak terdokumentasi seperti rasio aspek, resolusi, watermark, atau audio yang dihasilkan.

Lihat [Video Generation](/tools/video-generation) untuk perilaku tool bersama.

## Sintesis speech

Atur Vydra sebagai provider speech:

```json5
{
  messages: {
    tts: {
      provider: "vydra",
      providers: {
        vydra: {
          apiKey: "${VYDRA_API_KEY}",
          voiceId: "21m00Tcm4TlvDq8ikWAM",
        },
      },
    },
  },
}
```

Default:

- model: `elevenlabs/tts`
- voice id: `21m00Tcm4TlvDq8ikWAM`

Bundled plugin saat ini mengekspos satu suara default yang sudah terbukti baik dan mengembalikan file audio MP3.

## Terkait

- [Direktori Provider](/id/providers/index)
- [Image Generation](/id/tools/image-generation)
- [Video Generation](/tools/video-generation)
