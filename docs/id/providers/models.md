---
read_when:
    - Anda ingin memilih provider model
    - Anda menginginkan contoh penyiapan cepat untuk auth LLM + pemilihan model
summary: Provider model (LLM) yang didukung oleh OpenClaw
title: Quickstart Provider Model
x-i18n:
    generated_at: "2026-04-06T03:10:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: c0314fb1c754171e5fc252d30f7ba9bb6acdbb978d97e9249264d90351bac2e7
    source_path: providers/models.md
    workflow: 15
---

# Provider Model

OpenClaw dapat menggunakan banyak provider LLM. Pilih satu, autentikasi, lalu atur
model default sebagai `provider/model`.

## Memulai cepat (dua langkah)

1. Autentikasi dengan provider (biasanya melalui `openclaw onboard`).
2. Atur model default:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Provider yang didukung (set awal)

- [Alibaba Model Studio](/providers/alibaba)
- [Anthropic (API + Claude CLI)](/id/providers/anthropic)
- [Amazon Bedrock](/id/providers/bedrock)
- [BytePlus (Internasional)](/id/concepts/model-providers#byteplus-international)
- [Chutes](/id/providers/chutes)
- [ComfyUI](/providers/comfy)
- [Cloudflare AI Gateway](/id/providers/cloudflare-ai-gateway)
- [fal](/providers/fal)
- [Fireworks](/id/providers/fireworks)
- [Model GLM](/id/providers/glm)
- [MiniMax](/id/providers/minimax)
- [Mistral](/id/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/id/providers/moonshot)
- [OpenAI (API + Codex)](/id/providers/openai)
- [OpenCode (Zen + Go)](/id/providers/opencode)
- [OpenRouter](/id/providers/openrouter)
- [Qianfan](/id/providers/qianfan)
- [Qwen](/id/providers/qwen)
- [Runway](/providers/runway)
- [StepFun](/id/providers/stepfun)
- [Synthetic](/id/providers/synthetic)
- [Vercel AI Gateway](/id/providers/vercel-ai-gateway)
- [Venice (Venice AI)](/id/providers/venice)
- [xAI](/id/providers/xai)
- [Z.AI](/id/providers/zai)

## Varian provider bawaan tambahan

- `anthropic-vertex` - dukungan Anthropic implisit di Google Vertex saat kredensial Vertex tersedia; tidak ada pilihan auth onboarding terpisah
- `copilot-proxy` - bridge Copilot Proxy VS Code lokal; gunakan `openclaw onboard --auth-choice copilot-proxy`

Untuk katalog provider lengkap (xAI, Groq, Mistral, dll.) dan konfigurasi lanjutan,
lihat [Provider model](/id/concepts/model-providers).
