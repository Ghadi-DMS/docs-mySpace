---
read_when:
    - Anda ingin menggunakan alur kerja ComfyUI lokal dengan OpenClaw
    - Anda ingin menggunakan Comfy Cloud dengan alur kerja gambar, video, atau musik
    - Anda memerlukan key config plugin comfy bawaan
summary: Penyiapan pembuatan gambar, video, dan musik dengan alur kerja ComfyUI di OpenClaw
title: ComfyUI
x-i18n:
    generated_at: "2026-04-06T03:10:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: e645f32efdffdf4cd498684f1924bb953a014d3656b48f4b503d64e38c61ba9c
    source_path: providers/comfy.md
    workflow: 15
---

# ComfyUI

OpenClaw menyediakan plugin `comfy` bawaan untuk eksekusi ComfyUI berbasis alur kerja.

- Provider: `comfy`
- Model: `comfy/workflow`
- Permukaan bersama: `image_generate`, `video_generate`, `music_generate`
- Auth: tidak ada untuk ComfyUI lokal; `COMFY_API_KEY` atau `COMFY_CLOUD_API_KEY` untuk Comfy Cloud
- API: ComfyUI `/prompt` / `/history` / `/view` dan Comfy Cloud `/api/*`

## Yang didukung

- Pembuatan gambar dari workflow JSON
- Pengeditan gambar dengan 1 gambar referensi yang diunggah
- Pembuatan video dari workflow JSON
- Pembuatan video dengan 1 gambar referensi yang diunggah
- Pembuatan musik atau audio melalui alat bersama `music_generate`
- Unduhan output dari node yang dikonfigurasi atau semua node output yang cocok

Plugin bawaan ini berbasis alur kerja, jadi OpenClaw tidak mencoba memetakan
`size`, `aspectRatio`, `resolution`, `durationSeconds`, atau kontrol gaya TTS generik
ke graph Anda.

## Tata letak config

Comfy mendukung pengaturan koneksi tingkat atas bersama ditambah bagian alur kerja
per kapabilitas:

```json5
{
  models: {
    providers: {
      comfy: {
        mode: "local",
        baseUrl: "http://127.0.0.1:8188",
        image: {
          workflowPath: "./workflows/flux-api.json",
          promptNodeId: "6",
          outputNodeId: "9",
        },
        video: {
          workflowPath: "./workflows/video-api.json",
          promptNodeId: "12",
          outputNodeId: "21",
        },
        music: {
          workflowPath: "./workflows/music-api.json",
          promptNodeId: "3",
          outputNodeId: "18",
        },
      },
    },
  },
}
```

Key bersama:

- `mode`: `local` atau `cloud`
- `baseUrl`: default ke `http://127.0.0.1:8188` untuk lokal atau `https://cloud.comfy.org` untuk cloud
- `apiKey`: alternatif key inline opsional untuk env vars
- `allowPrivateNetwork`: izinkan `baseUrl` privat/LAN dalam mode cloud

Key per kapabilitas di bawah `image`, `video`, atau `music`:

- `workflow` atau `workflowPath`: wajib
- `promptNodeId`: wajib
- `promptInputName`: default ke `text`
- `outputNodeId`: opsional
- `pollIntervalMs`: opsional
- `timeoutMs`: opsional

Bagian gambar dan video juga mendukung:

- `inputImageNodeId`: wajib saat Anda meneruskan gambar referensi
- `inputImageInputName`: default ke `image`

## Kompatibilitas mundur

Config gambar tingkat atas yang sudah ada tetap berfungsi:

```json5
{
  models: {
    providers: {
      comfy: {
        workflowPath: "./workflows/flux-api.json",
        promptNodeId: "6",
        outputNodeId: "9",
      },
    },
  },
}
```

OpenClaw memperlakukan bentuk lama itu sebagai config alur kerja gambar.

## Alur kerja gambar

Setel model gambar default:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "comfy/workflow",
      },
    },
  },
}
```

Contoh pengeditan dengan gambar referensi:

```json5
{
  models: {
    providers: {
      comfy: {
        image: {
          workflowPath: "./workflows/edit-api.json",
          promptNodeId: "6",
          inputImageNodeId: "7",
          inputImageInputName: "image",
          outputNodeId: "9",
        },
      },
    },
  },
}
```

## Alur kerja video

Setel model video default:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "comfy/workflow",
      },
    },
  },
}
```

Alur kerja video Comfy saat ini mendukung text-to-video dan image-to-video melalui
graph yang dikonfigurasi. OpenClaw tidak meneruskan video input ke alur kerja Comfy.

## Alur kerja musik

Plugin bawaan mendaftarkan provider pembuatan musik untuk output
audio atau musik yang didefinisikan alur kerja, yang ditampilkan melalui alat bersama `music_generate`:

```text
/tool music_generate prompt="Warm ambient synth loop with soft tape texture"
```

Gunakan bagian config `music` untuk menunjuk ke workflow JSON audio Anda dan node
output.

## Comfy Cloud

Gunakan `mode: "cloud"` ditambah salah satu dari:

- `COMFY_API_KEY`
- `COMFY_CLOUD_API_KEY`
- `models.providers.comfy.apiKey`

Mode cloud tetap menggunakan bagian alur kerja `image`, `video`, dan `music` yang sama.

## Pengujian langsung

Cakupan langsung opt-in tersedia untuk plugin bawaan:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

Pengujian langsung melewati kasus gambar, video, atau musik individual kecuali
bagian alur kerja Comfy yang cocok telah dikonfigurasi.

## Terkait

- [Image Generation](/id/tools/image-generation)
- [Video Generation](/tools/video-generation)
- [Music Generation](/tools/music-generation)
- [Provider Directory](/id/providers/index)
- [Configuration Reference](/id/gateway/configuration-reference#agent-defaults)
