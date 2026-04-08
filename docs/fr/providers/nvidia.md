---
read_when:
    - Vous voulez utiliser gratuitement des modèles ouverts dans OpenClaw
    - Vous avez besoin de configurer NVIDIA_API_KEY
summary: Utiliser l'API compatible OpenAI de NVIDIA dans OpenClaw
title: NVIDIA
x-i18n:
    generated_at: "2026-04-08T02:17:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: b00f8cedaf223a33ba9f6a6dd8cf066d88cebeea52d391b871e435026182228a
    source_path: providers/nvidia.md
    workflow: 15
---

# NVIDIA

NVIDIA fournit une API compatible OpenAI à l'adresse `https://integrate.api.nvidia.com/v1` pour des modèles ouverts gratuitement. Authentifiez-vous avec une clé API de [build.nvidia.com](https://build.nvidia.com/settings/api-keys).

## Configuration CLI

Exportez la clé une fois, puis lancez l'onboarding et définissez un modèle NVIDIA :

```bash
export NVIDIA_API_KEY="nvapi-..."
openclaw onboard --auth-choice skip
openclaw models set nvidia/nvidia/nemotron-3-super-120b-a12b
```

Si vous passez encore `--token`, rappelez-vous qu'il se retrouve dans l'historique du shell et dans la sortie de `ps` ; préférez la variable d'environnement lorsque c'est possible.

## Extrait de configuration

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/nemotron-3-super-120b-a12b" },
    },
  },
}
```

## Identifiants de modèle

| Référence de modèle                         | Nom                          | Contexte | Sortie max |
| ------------------------------------------ | ---------------------------- | -------- | ---------- |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | NVIDIA Nemotron 3 Super 120B | 262,144  | 8,192      |
| `nvidia/moonshotai/kimi-k2.5`              | Kimi K2.5                    | 262,144  | 8,192      |
| `nvidia/minimaxai/minimax-m2.5`            | Minimax M2.5                 | 196,608  | 8,192      |
| `nvidia/z-ai/glm5`                         | GLM 5                        | 202,752  | 8,192      |

## Remarques

- Endpoint `/v1` compatible OpenAI ; utilisez une clé API de [build.nvidia.com](https://build.nvidia.com/).
- Le fournisseur s'active automatiquement lorsque `NVIDIA_API_KEY` est défini.
- Le catalogue groupé est statique ; les coûts prennent la valeur par défaut `0` dans le code source.
