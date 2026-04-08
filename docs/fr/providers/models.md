---
read_when:
    - Vous voulez choisir un fournisseur de modèles
    - Vous voulez des exemples de configuration rapide pour l'authentification LLM et la sélection de modèle
summary: Fournisseurs de modèles (LLM) pris en charge par OpenClaw
title: Démarrage rapide des fournisseurs de modèles
x-i18n:
    generated_at: "2026-04-08T02:17:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: 59ee4c2f993fe0ae05fe34f52bc6f3e0fc9a76b10760f56b20ad251e25ee9f20
    source_path: providers/models.md
    workflow: 15
---

# Fournisseurs de modèles

OpenClaw peut utiliser de nombreux fournisseurs de LLM. Choisissez-en un, authentifiez-vous, puis définissez le
modèle par défaut sous la forme `provider/model`.

## Démarrage rapide (deux étapes)

1. Authentifiez-vous auprès du fournisseur (généralement via `openclaw onboard`).
2. Définissez le modèle par défaut :

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Fournisseurs pris en charge (ensemble de départ)

- [Alibaba Model Studio](/fr/providers/alibaba)
- [Anthropic (API + Claude CLI)](/fr/providers/anthropic)
- [Amazon Bedrock](/fr/providers/bedrock)
- [BytePlus (International)](/fr/concepts/model-providers#byteplus-international)
- [Chutes](/fr/providers/chutes)
- [ComfyUI](/fr/providers/comfy)
- [Cloudflare AI Gateway](/fr/providers/cloudflare-ai-gateway)
- [fal](/fr/providers/fal)
- [Fireworks](/fr/providers/fireworks)
- [GLM models](/fr/providers/glm)
- [MiniMax](/fr/providers/minimax)
- [Mistral](/fr/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/fr/providers/moonshot)
- [OpenAI (API + Codex)](/fr/providers/openai)
- [OpenCode (Zen + Go)](/fr/providers/opencode)
- [OpenRouter](/fr/providers/openrouter)
- [Qianfan](/fr/providers/qianfan)
- [Qwen](/fr/providers/qwen)
- [Runway](/fr/providers/runway)
- [StepFun](/fr/providers/stepfun)
- [Synthetic](/fr/providers/synthetic)
- [Vercel AI Gateway](/fr/providers/vercel-ai-gateway)
- [Venice (Venice AI)](/fr/providers/venice)
- [xAI](/fr/providers/xai)
- [Z.AI](/fr/providers/zai)

## Variantes supplémentaires de fournisseurs groupés

- `anthropic-vertex` - prise en charge implicite d'Anthropic sur Google Vertex lorsque des identifiants Vertex sont disponibles ; aucun choix d'authentification d'onboarding distinct
- `copilot-proxy` - pont proxy local VS Code Copilot ; utilisez `openclaw onboard --auth-choice copilot-proxy`
- `google-gemini-cli` - flux OAuth Gemini CLI non officiel ; nécessite une installation locale de `gemini` (`brew install gemini-cli` ou `npm install -g @google/gemini-cli`) ; modèle par défaut `google-gemini-cli/gemini-3-flash-preview` ; utilisez `openclaw onboard --auth-choice google-gemini-cli` ou `openclaw models auth login --provider google-gemini-cli --set-default`

Pour le catalogue complet des fournisseurs (xAI, Groq, Mistral, etc.) et la configuration avancée,
voir [Fournisseurs de modèles](/fr/concepts/model-providers).
