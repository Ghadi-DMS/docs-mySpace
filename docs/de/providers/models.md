---
read_when:
    - Sie möchten einen Modellanbieter auswählen
    - Sie möchten kurze Einrichtungsbeispiele für LLM-Auth + Modellauswahl
summary: Von OpenClaw unterstützte Modellanbieter (LLMs)
title: Schnellstart für Modellanbieter
x-i18n:
    generated_at: "2026-04-08T02:17:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 59ee4c2f993fe0ae05fe34f52bc6f3e0fc9a76b10760f56b20ad251e25ee9f20
    source_path: providers/models.md
    workflow: 15
---

# Modellanbieter

OpenClaw kann viele LLM-Anbieter verwenden. Wählen Sie einen aus, authentifizieren Sie sich und setzen Sie dann das Standard-
modell als `provider/model`.

## Schnellstart (zwei Schritte)

1. Beim Anbieter authentifizieren (normalerweise über `openclaw onboard`).
2. Das Standardmodell festlegen:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Unterstützte Anbieter (Startauswahl)

- [Alibaba Model Studio](/de/providers/alibaba)
- [Anthropic (API + Claude CLI)](/de/providers/anthropic)
- [Amazon Bedrock](/de/providers/bedrock)
- [BytePlus (International)](/de/concepts/model-providers#byteplus-international)
- [Chutes](/de/providers/chutes)
- [ComfyUI](/de/providers/comfy)
- [Cloudflare AI Gateway](/de/providers/cloudflare-ai-gateway)
- [fal](/de/providers/fal)
- [Fireworks](/de/providers/fireworks)
- [GLM models](/de/providers/glm)
- [MiniMax](/de/providers/minimax)
- [Mistral](/de/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/de/providers/moonshot)
- [OpenAI (API + Codex)](/de/providers/openai)
- [OpenCode (Zen + Go)](/de/providers/opencode)
- [OpenRouter](/de/providers/openrouter)
- [Qianfan](/de/providers/qianfan)
- [Qwen](/de/providers/qwen)
- [Runway](/de/providers/runway)
- [StepFun](/de/providers/stepfun)
- [Synthetic](/de/providers/synthetic)
- [Vercel AI Gateway](/de/providers/vercel-ai-gateway)
- [Venice (Venice AI)](/de/providers/venice)
- [xAI](/de/providers/xai)
- [Z.AI](/de/providers/zai)

## Zusätzliche gebündelte Anbietervarianten

- `anthropic-vertex` - implizite Anthropic-Unterstützung auf Google Vertex, wenn Vertex-Credentials verfügbar sind; keine separate Onboarding-Auth-Auswahl
- `copilot-proxy` - lokale VS Code Copilot Proxy-Bridge; verwenden Sie `openclaw onboard --auth-choice copilot-proxy`
- `google-gemini-cli` - inoffizieller Gemini-CLI-OAuth-Flow; erfordert eine lokale `gemini`-Installation (`brew install gemini-cli` oder `npm install -g @google/gemini-cli`); Standardmodell `google-gemini-cli/gemini-3-flash-preview`; verwenden Sie `openclaw onboard --auth-choice google-gemini-cli` oder `openclaw models auth login --provider google-gemini-cli --set-default`

Den vollständigen Anbieterkatalog (xAI, Groq, Mistral usw.) und die erweiterte Konfiguration finden Sie unter [Modellanbieter](/de/concepts/model-providers).
