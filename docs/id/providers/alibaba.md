---
read_when:
    - Anda ingin menggunakan pembuatan video Alibaba Wan di OpenClaw
    - Anda memerlukan penyiapan API key Model Studio atau DashScope untuk pembuatan video
summary: Pembuatan video Wan Alibaba Model Studio di OpenClaw
title: Alibaba Model Studio
x-i18n:
    generated_at: "2026-04-06T03:09:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: 97a1eddc7cbd816776b9368f2a926b5ef9ee543f08d151a490023736f67dc635
    source_path: providers/alibaba.md
    workflow: 15
---

# Alibaba Model Studio

OpenClaw menyediakan provider pembuatan video `alibaba` bawaan untuk model Wan di
Alibaba Model Studio / DashScope.

- Provider: `alibaba`
- Auth yang direkomendasikan: `MODELSTUDIO_API_KEY`
- Juga diterima: `DASHSCOPE_API_KEY`, `QWEN_API_KEY`
- API: pembuatan video asinkron DashScope / Model Studio

## Mulai cepat

1. Setel API key:

```bash
openclaw onboard --auth-choice qwen-standard-api-key
```

2. Setel model video default:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "alibaba/wan2.6-t2v",
      },
    },
  },
}
```

## Model Wan bawaan

Provider `alibaba` bawaan saat ini mendaftarkan:

- `alibaba/wan2.6-t2v`
- `alibaba/wan2.6-i2v`
- `alibaba/wan2.6-r2v`
- `alibaba/wan2.6-r2v-flash`
- `alibaba/wan2.7-r2v`

## Batas saat ini

- Hingga **1** video keluaran per permintaan
- Hingga **1** gambar masukan
- Hingga **4** video masukan
- Durasi hingga **10 detik**
- Mendukung `size`, `aspectRatio`, `resolution`, `audio`, dan `watermark`
- Mode gambar/video referensi saat ini memerlukan **URL http(s) remote**

## Hubungan dengan Qwen

Provider `qwen` bawaan juga menggunakan endpoint DashScope yang dihosting Alibaba untuk
pembuatan video Wan. Gunakan:

- `qwen/...` saat Anda menginginkan permukaan provider Qwen yang kanonis
- `alibaba/...` saat Anda menginginkan permukaan video Wan langsung milik vendor

## Terkait

- [Video Generation](/tools/video-generation)
- [Qwen](/id/providers/qwen)
- [Configuration Reference](/id/gateway/configuration-reference#agent-defaults)
