---
read_when:
    - Anda ingin memilih provider model
    - Anda memerlukan ikhtisar cepat tentang backend LLM yang didukung
summary: Provider model (LLM) yang didukung oleh OpenClaw
title: Direktori Provider
x-i18n:
    generated_at: "2026-04-06T03:10:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7271157a6ab5418672baff62bfd299572fd010f75aad529267095c6e55903882
    source_path: providers/index.md
    workflow: 15
---

# Provider Model

OpenClaw dapat menggunakan banyak provider LLM. Pilih provider, lakukan autentikasi, lalu atur
model default sebagai `provider/model`.

Mencari dokumen channel chat (WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/dll.)? Lihat [Channels](/id/channels).

## Mulai cepat

1. Lakukan autentikasi dengan provider (biasanya melalui `openclaw onboard`).
2. Atur model default:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Dokumen provider

- [Alibaba Model Studio](/providers/alibaba)
- [Amazon Bedrock](/id/providers/bedrock)
- [Anthropic (API + Claude CLI)](/id/providers/anthropic)
- [BytePlus (Internasional)](/id/concepts/model-providers#byteplus-international)
- [Chutes](/id/providers/chutes)
- [ComfyUI](/providers/comfy)
- [Cloudflare AI Gateway](/id/providers/cloudflare-ai-gateway)
- [DeepSeek](/id/providers/deepseek)
- [fal](/providers/fal)
- [Fireworks](/id/providers/fireworks)
- [GitHub Copilot](/id/providers/github-copilot)
- [Model GLM](/id/providers/glm)
- [Google (Gemini)](/id/providers/google)
- [Groq (inferensi LPU)](/id/providers/groq)
- [Hugging Face (Inference)](/id/providers/huggingface)
- [Kilocode](/id/providers/kilocode)
- [LiteLLM (gateway terpadu)](/id/providers/litellm)
- [MiniMax](/id/providers/minimax)
- [Mistral](/id/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/id/providers/moonshot)
- [NVIDIA](/id/providers/nvidia)
- [Ollama (model cloud + lokal)](/id/providers/ollama)
- [OpenAI (API + Codex)](/id/providers/openai)
- [OpenCode](/id/providers/opencode)
- [OpenCode Go](/id/providers/opencode-go)
- [OpenRouter](/id/providers/openrouter)
- [Perplexity (pencarian web)](/id/providers/perplexity-provider)
- [Qianfan](/id/providers/qianfan)
- [Qwen Cloud](/id/providers/qwen)
- [Runway](/providers/runway)
- [SGLang (model lokal)](/id/providers/sglang)
- [StepFun](/id/providers/stepfun)
- [Synthetic](/id/providers/synthetic)
- [Together AI](/id/providers/together)
- [Venice (Venice AI, berfokus pada privasi)](/id/providers/venice)
- [Vercel AI Gateway](/id/providers/vercel-ai-gateway)
- [Vydra](/providers/vydra)
- [vLLM (model lokal)](/id/providers/vllm)
- [Volcengine (Doubao)](/id/providers/volcengine)
- [xAI](/id/providers/xai)
- [Xiaomi](/id/providers/xiaomi)
- [Z.AI](/id/providers/zai)

## Halaman ikhtisar bersama

- [Varian bundled tambahan](/id/providers/models#additional-bundled-provider-variants) - Anthropic Vertex, Copilot Proxy, dan Gemini CLI OAuth
- [Image Generation](/id/tools/image-generation) - Tool bersama `image_generate`, pemilihan provider, dan failover
- [Music Generation](/tools/music-generation) - Tool bersama `music_generate`, pemilihan provider, dan failover
- [Video Generation](/tools/video-generation) - Tool bersama `video_generate`, pemilihan provider, dan failover

## Provider transkripsi

- [Deepgram (transkripsi audio)](/id/providers/deepgram)

## Tool komunitas

- [Claude Max API Proxy](/id/providers/claude-max-api-proxy) - Proxy komunitas untuk kredensial langganan Claude (verifikasi kebijakan/persyaratan Anthropic sebelum digunakan)

Untuk katalog provider lengkap (xAI, Groq, Mistral, dll.) dan konfigurasi lanjutan,
lihat [Model providers](/id/concepts/model-providers).
