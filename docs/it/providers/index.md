---
read_when:
    - Vuoi scegliere un provider di modelli
    - Hai bisogno di una rapida panoramica dei backend LLM supportati
summary: Provider di modelli (LLM) supportati da OpenClaw
title: Directory provider
x-i18n:
    generated_at: "2026-04-06T03:11:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7271157a6ab5418672baff62bfd299572fd010f75aad529267095c6e55903882
    source_path: providers/index.md
    workflow: 15
---

# Provider di modelli

OpenClaw può usare molti provider LLM. Scegli un provider, autenticati, poi imposta il
modello predefinito come `provider/model`.

Cerchi la documentazione dei canali chat (WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/ecc.)? Consulta [Channels](/it/channels).

## Avvio rapido

1. Autenticati con il provider (di solito tramite `openclaw onboard`).
2. Imposta il modello predefinito:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Documentazione provider

- [Alibaba Model Studio](/providers/alibaba)
- [Amazon Bedrock](/it/providers/bedrock)
- [Anthropic (API + Claude CLI)](/it/providers/anthropic)
- [BytePlus (internazionale)](/it/concepts/model-providers#byteplus-international)
- [Chutes](/it/providers/chutes)
- [ComfyUI](/providers/comfy)
- [Cloudflare AI Gateway](/it/providers/cloudflare-ai-gateway)
- [DeepSeek](/it/providers/deepseek)
- [fal](/providers/fal)
- [Fireworks](/it/providers/fireworks)
- [GitHub Copilot](/it/providers/github-copilot)
- [Modelli GLM](/it/providers/glm)
- [Google (Gemini)](/it/providers/google)
- [Groq (inferenza LPU)](/it/providers/groq)
- [Hugging Face (inferenza)](/it/providers/huggingface)
- [Kilocode](/it/providers/kilocode)
- [LiteLLM (gateway unificato)](/it/providers/litellm)
- [MiniMax](/it/providers/minimax)
- [Mistral](/it/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/it/providers/moonshot)
- [NVIDIA](/it/providers/nvidia)
- [Ollama (modelli cloud + locali)](/it/providers/ollama)
- [OpenAI (API + Codex)](/it/providers/openai)
- [OpenCode](/it/providers/opencode)
- [OpenCode Go](/it/providers/opencode-go)
- [OpenRouter](/it/providers/openrouter)
- [Perplexity (ricerca web)](/it/providers/perplexity-provider)
- [Qianfan](/it/providers/qianfan)
- [Qwen Cloud](/it/providers/qwen)
- [Runway](/providers/runway)
- [SGLang (modelli locali)](/it/providers/sglang)
- [StepFun](/it/providers/stepfun)
- [Synthetic](/it/providers/synthetic)
- [Together AI](/it/providers/together)
- [Venice (Venice AI, focalizzato sulla privacy)](/it/providers/venice)
- [Vercel AI Gateway](/it/providers/vercel-ai-gateway)
- [Vydra](/providers/vydra)
- [vLLM (modelli locali)](/it/providers/vllm)
- [Volcengine (Doubao)](/it/providers/volcengine)
- [xAI](/it/providers/xai)
- [Xiaomi](/it/providers/xiaomi)
- [Z.AI](/it/providers/zai)

## Pagine panoramiche condivise

- [Varianti bundled aggiuntive](/it/providers/models#additional-bundled-provider-variants) - Anthropic Vertex, Copilot Proxy e Gemini CLI OAuth
- [Generazione di immagini](/it/tools/image-generation) - strumento condiviso `image_generate`, selezione del provider e failover
- [Generazione di musica](/tools/music-generation) - strumento condiviso `music_generate`, selezione del provider e failover
- [Generazione di video](/tools/video-generation) - strumento condiviso `video_generate`, selezione del provider e failover

## Provider di trascrizione

- [Deepgram (trascrizione audio)](/it/providers/deepgram)

## Strumenti della community

- [Claude Max API Proxy](/it/providers/claude-max-api-proxy) - Proxy della community per credenziali di abbonamento Claude (verifica i criteri/termini di Anthropic prima dell'uso)

Per il catalogo completo dei provider (xAI, Groq, Mistral, ecc.) e la configurazione avanzata,
consulta [Provider di modelli](/it/concepts/model-providers).
