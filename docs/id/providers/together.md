---
read_when:
    - Anda ingin menggunakan Together AI dengan OpenClaw
    - Anda memerlukan env var API key atau pilihan auth CLI
summary: Penyiapan Together AI (auth + pemilihan model)
title: Together AI
x-i18n:
    generated_at: "2026-04-06T03:10:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: b68fdc15bfcac8d59e3e0c06a39162abd48d9d41a9a64a0ac622cd8e3f80a595
    source_path: providers/together.md
    workflow: 15
---

# Together AI

[Together AI](https://together.ai) menyediakan akses ke model open-source terkemuka termasuk Llama, DeepSeek, Kimi, dan lainnya melalui API terpadu.

- Provider: `together`
- Auth: `TOGETHER_API_KEY`
- API: kompatibel dengan OpenAI
- Base URL: `https://api.together.xyz/v1`

## Memulai cepat

1. Tetapkan API key (disarankan: simpan untuk Gateway):

```bash
openclaw onboard --auth-choice together-api-key
```

2. Tetapkan model default:

```json5
{
  agents: {
    defaults: {
      model: { primary: "together/moonshotai/Kimi-K2.5" },
    },
  },
}
```

## Contoh non-interaktif

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

Ini akan menetapkan `together/moonshotai/Kimi-K2.5` sebagai model default.

## Catatan environment

Jika Gateway berjalan sebagai daemon (launchd/systemd), pastikan `TOGETHER_API_KEY`
tersedia untuk proses tersebut (misalnya, di `~/.openclaw/.env` atau melalui
`env.shellEnv`).

## Katalog bawaan

OpenClaw saat ini dikirim dengan katalog Together bawaan berikut:

| Model ref                                                    | Nama                                   | Input       | Context    | Catatan                         |
| ------------------------------------------------------------ | -------------------------------------- | ----------- | ---------- | ------------------------------- |
| `together/moonshotai/Kimi-K2.5`                              | Kimi K2.5                              | text, image | 262,144    | Model default; reasoning aktif |
| `together/zai-org/GLM-4.7`                                   | GLM 4.7 Fp8                            | text        | 202,752    | Model teks serbaguna            |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`           | Llama 3.3 70B Instruct Turbo           | text        | 131,072    | Model instruksi cepat           |
| `together/meta-llama/Llama-4-Scout-17B-16E-Instruct`         | Llama 4 Scout 17B 16E Instruct         | text, image | 10,000,000 | Multimodal                      |
| `together/meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8` | Llama 4 Maverick 17B 128E Instruct FP8 | text, image | 20,000,000 | Multimodal                      |
| `together/deepseek-ai/DeepSeek-V3.1`                         | DeepSeek V3.1                          | text        | 131,072    | Model teks umum                 |
| `together/deepseek-ai/DeepSeek-R1`                           | DeepSeek R1                            | text        | 131,072    | Model reasoning                 |
| `together/moonshotai/Kimi-K2-Instruct-0905`                  | Kimi K2-Instruct 0905                  | text        | 262,144    | Model teks Kimi sekunder        |

Preset onboarding menetapkan `together/moonshotai/Kimi-K2.5` sebagai model default.

## Pembuatan video

Plugin `together` bawaan juga mendaftarkan pembuatan video melalui
tool `video_generate` bersama.

- Model video default: `together/Wan-AI/Wan2.2-T2V-A14B`
- Mode: text-to-video dan alur referensi gambar tunggal
- Mendukung `aspectRatio` dan `resolution`

Untuk menggunakan Together sebagai provider video default:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "together/Wan-AI/Wan2.2-T2V-A14B",
      },
    },
  },
}
```

Lihat [Pembuatan Video](/tools/video-generation) untuk parameter
tool bersama, pemilihan provider, dan perilaku failover.
