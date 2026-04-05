---
read_when:
    - Quieres elegir un proveedor de modelos
    - Necesitas una vista rápida de los backends LLM compatibles
summary: Proveedores de modelos (LLM) compatibles con OpenClaw
title: Directorio de proveedores
x-i18n:
    generated_at: "2026-04-05T12:51:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 690d17c14576d454ea3cd3dcbc704470da10a2a34adfe681dab7048438f2e193
    source_path: providers/index.md
    workflow: 15
---

# Proveedores de modelos

OpenClaw puede usar muchos proveedores de LLM. Elige un proveedor, autentícate y luego establece el
modelo predeterminado como `provider/model`.

¿Buscas documentación de canales de chat (WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/etc.)? Consulta [Channels](/channels).

## Inicio rápido

1. Autentícate con el proveedor (normalmente mediante `openclaw onboard`).
2. Establece el modelo predeterminado:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Documentación de proveedores

- [Amazon Bedrock](/providers/bedrock)
- [Anthropic (API + Claude CLI)](/providers/anthropic)
- [BytePlus (International)](/concepts/model-providers#byteplus-international)
- [Chutes](/providers/chutes)
- [Cloudflare AI Gateway](/providers/cloudflare-ai-gateway)
- [DeepSeek](/providers/deepseek)
- [Fireworks](/providers/fireworks)
- [GitHub Copilot](/providers/github-copilot)
- [Modelos GLM](/providers/glm)
- [Google (Gemini)](/providers/google)
- [Groq (inferencia LPU)](/providers/groq)
- [Hugging Face (Inference)](/providers/huggingface)
- [Kilocode](/providers/kilocode)
- [LiteLLM (gateway unificado)](/providers/litellm)
- [MiniMax](/providers/minimax)
- [Mistral](/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/providers/moonshot)
- [NVIDIA](/providers/nvidia)
- [Ollama (nube + modelos locales)](/providers/ollama)
- [OpenAI (API + Codex)](/providers/openai)
- [OpenCode](/providers/opencode)
- [OpenCode Go](/providers/opencode-go)
- [OpenRouter](/providers/openrouter)
- [Perplexity (búsqueda web)](/providers/perplexity-provider)
- [Qianfan](/providers/qianfan)
- [Qwen Cloud](/providers/qwen)
- [Qwen / Model Studio (detalle del endpoint; `qwen-*` canónico, `modelstudio-*` heredado)](/providers/qwen_modelstudio)
- [SGLang (modelos locales)](/providers/sglang)
- [StepFun](/providers/stepfun)
- [Synthetic](/providers/synthetic)
- [Together AI](/providers/together)
- [Venice (Venice AI, centrado en la privacidad)](/providers/venice)
- [Vercel AI Gateway](/providers/vercel-ai-gateway)
- [vLLM (modelos locales)](/providers/vllm)
- [Volcengine (Doubao)](/providers/volcengine)
- [xAI](/providers/xai)
- [Xiaomi](/providers/xiaomi)
- [Z.AI](/providers/zai)

## Páginas de resumen compartidas

- [Variantes integradas adicionales](/providers/models#additional-bundled-provider-variants) - Anthropic Vertex, Copilot Proxy y Gemini CLI OAuth

## Proveedores de transcripción

- [Deepgram (transcripción de audio)](/providers/deepgram)

## Herramientas de la comunidad

- [Claude Max API Proxy](/providers/claude-max-api-proxy) - Proxy comunitario para credenciales de suscripción de Claude (verifica la política/los términos de Anthropic antes de usarlo)

Para ver el catálogo completo de proveedores (xAI, Groq, Mistral, etc.) y la configuración avanzada,
consulta [Model providers](/concepts/model-providers).
