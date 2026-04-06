---
read_when:
    - Vuoi scegliere un provider di modelli
    - Vuoi esempi di configurazione rapida per auth LLM + selezione del modello
summary: Provider di modelli (LLM) supportati da OpenClaw
title: Guida rapida ai provider di modelli
x-i18n:
    generated_at: "2026-04-06T03:11:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: c0314fb1c754171e5fc252d30f7ba9bb6acdbb978d97e9249264d90351bac2e7
    source_path: providers/models.md
    workflow: 15
---

# Provider di modelli

OpenClaw può usare molti provider LLM. Scegline uno, autenticati, quindi imposta il modello predefinito
come `provider/model`.

## Guida rapida (due passaggi)

1. Autenticati con il provider (di solito tramite `openclaw onboard`).
2. Imposta il modello predefinito:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Provider supportati (set iniziale)

- [Alibaba Model Studio](/providers/alibaba)
- [Anthropic (API + Claude CLI)](/it/providers/anthropic)
- [Amazon Bedrock](/it/providers/bedrock)
- [BytePlus (International)](/it/concepts/model-providers#byteplus-international)
- [Chutes](/it/providers/chutes)
- [ComfyUI](/providers/comfy)
- [Cloudflare AI Gateway](/it/providers/cloudflare-ai-gateway)
- [fal](/providers/fal)
- [Fireworks](/it/providers/fireworks)
- [Modelli GLM](/it/providers/glm)
- [MiniMax](/it/providers/minimax)
- [Mistral](/it/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/it/providers/moonshot)
- [OpenAI (API + Codex)](/it/providers/openai)
- [OpenCode (Zen + Go)](/it/providers/opencode)
- [OpenRouter](/it/providers/openrouter)
- [Qianfan](/it/providers/qianfan)
- [Qwen](/it/providers/qwen)
- [Runway](/providers/runway)
- [StepFun](/it/providers/stepfun)
- [Synthetic](/it/providers/synthetic)
- [Vercel AI Gateway](/it/providers/vercel-ai-gateway)
- [Venice (Venice AI)](/it/providers/venice)
- [xAI](/it/providers/xai)
- [Z.AI](/it/providers/zai)

## Varianti aggiuntive dei provider integrati

- `anthropic-vertex` - supporto Anthropic implicito su Google Vertex quando sono disponibili credenziali Vertex; nessuna scelta di autenticazione onboarding separata
- `copilot-proxy` - bridge locale VS Code Copilot Proxy; usa `openclaw onboard --auth-choice copilot-proxy`

Per il catalogo completo dei provider (xAI, Groq, Mistral, ecc.) e la configurazione avanzata,
vedi [Provider di modelli](/it/concepts/model-providers).
