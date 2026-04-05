---
read_when:
    - Quieres elegir un proveedor de modelos
    - Quieres ejemplos rápidos de configuración para autenticación de LLM + selección de modelo
summary: Proveedores de modelos (LLM) compatibles con OpenClaw
title: Inicio rápido de proveedores de modelos
x-i18n:
    generated_at: "2026-04-05T12:51:28Z"
    model: gpt-5.4
    provider: openai
    source_hash: 83e372193b476c7cee6eb9f5c443b03563d863043f47c633ac0096bca642cc6f
    source_path: providers/models.md
    workflow: 15
---

# Proveedores de modelos

OpenClaw puede usar muchos proveedores de LLM. Elige uno, autentícalo y luego configura el
modelo predeterminado como `provider/model`.

## Inicio rápido (dos pasos)

1. Autentícate con el proveedor (normalmente mediante `openclaw onboard`).
2. Configura el modelo predeterminado:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Proveedores compatibles (conjunto inicial)

- [Anthropic (API + Claude CLI)](/providers/anthropic)
- [Amazon Bedrock](/providers/bedrock)
- [BytePlus (internacional)](/concepts/model-providers#byteplus-international)
- [Chutes](/providers/chutes)
- [Cloudflare AI Gateway](/providers/cloudflare-ai-gateway)
- [Fireworks](/providers/fireworks)
- [Modelos GLM](/providers/glm)
- [MiniMax](/providers/minimax)
- [Mistral](/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/providers/moonshot)
- [OpenAI (API + Codex)](/providers/openai)
- [OpenCode (Zen + Go)](/providers/opencode)
- [OpenRouter](/providers/openrouter)
- [Qianfan](/providers/qianfan)
- [Qwen](/providers/qwen)
- [StepFun](/providers/stepfun)
- [Synthetic](/providers/synthetic)
- [Vercel AI Gateway](/providers/vercel-ai-gateway)
- [Venice (Venice AI)](/providers/venice)
- [xAI](/providers/xai)
- [Z.AI](/providers/zai)

## Variantes adicionales de proveedores incluidos

- `anthropic-vertex` - compatibilidad implícita de Anthropic en Google Vertex cuando hay credenciales de Vertex disponibles; no hay una opción de autenticación de onboarding independiente
- `copilot-proxy` - puente local de VS Code Copilot Proxy; usa `openclaw onboard --auth-choice copilot-proxy`
- `google-gemini-cli` - flujo OAuth no oficial de Gemini CLI; requiere una instalación local de `gemini` (`brew install gemini-cli` o `npm install -g @google/gemini-cli`); modelo predeterminado `google-gemini-cli/gemini-3.1-pro-preview`; usa `openclaw onboard --auth-choice google-gemini-cli` o `openclaw models auth login --provider google-gemini-cli --set-default`

Para ver el catálogo completo de proveedores (xAI, Groq, Mistral, etc.) y la configuración avanzada,
consulta [Model providers](/concepts/model-providers).
