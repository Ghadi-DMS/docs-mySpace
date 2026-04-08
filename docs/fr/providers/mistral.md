---
read_when:
    - Vous voulez utiliser des modèles Mistral dans OpenClaw
    - Vous avez besoin de l'onboarding par clé API Mistral et des références de modèles
summary: Utiliser les modèles Mistral et la transcription Voxtral avec OpenClaw
title: Mistral
x-i18n:
    generated_at: "2026-04-08T02:17:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e32a0eb2a37dba6383ba338b06a8d0be600e7443aa916225794ccb0fdf46aee
    source_path: providers/mistral.md
    workflow: 15
---

# Mistral

OpenClaw prend en charge Mistral à la fois pour le routage des modèles texte/image (`mistral/...`) et
la transcription audio via Voxtral dans l'analyse des médias.
Mistral peut également être utilisé pour les embeddings mémoire (`memorySearch.provider = "mistral"`).

## Configuration CLI

```bash
openclaw onboard --auth-choice mistral-api-key
# or non-interactive
openclaw onboard --mistral-api-key "$MISTRAL_API_KEY"
```

## Extrait de configuration (fournisseur LLM)

```json5
{
  env: { MISTRAL_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
}
```

## Catalogue LLM intégré

OpenClaw inclut actuellement ce catalogue Mistral intégré :

| Référence de modèle              | Entrée      | Contexte | Sortie max | Remarques                                                       |
| -------------------------------- | ----------- | -------- | ---------- | ---------------------------------------------------------------- |
| `mistral/mistral-large-latest`   | texte, image | 262,144 | 16,384     | Modèle par défaut                                                |
| `mistral/mistral-medium-2508`    | texte, image | 262,144 | 8,192      | Mistral Medium 3.1                                               |
| `mistral/mistral-small-latest`   | texte, image | 128,000 | 16,384     | Mistral Small 4 ; raisonnement ajustable via l'API `reasoning_effort` |
| `mistral/pixtral-large-latest`   | texte, image | 128,000 | 32,768     | Pixtral                                                          |
| `mistral/codestral-latest`       | texte        | 256,000 | 4,096      | Codage                                                           |
| `mistral/devstral-medium-latest` | texte        | 262,144 | 32,768     | Devstral 2                                                       |
| `mistral/magistral-small`        | texte        | 128,000 | 40,000     | Raisonnement activé                                              |

## Extrait de configuration (transcription audio avec Voxtral)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", model: "voxtral-mini-latest" }],
      },
    },
  },
}
```

## Raisonnement ajustable (`mistral-small-latest`)

`mistral/mistral-small-latest` correspond à Mistral Small 4 et prend en charge le [raisonnement ajustable](https://docs.mistral.ai/capabilities/reasoning/adjustable) sur l'API Chat Completions via `reasoning_effort` (`none` minimise la réflexion supplémentaire dans la sortie ; `high` affiche les traces complètes de réflexion avant la réponse finale).

OpenClaw mappe le niveau de **thinking** de la session vers l'API de Mistral :

- **off** / **minimal** → `none`
- **low** / **medium** / **high** / **xhigh** / **adaptive** → `high`

Les autres modèles du catalogue Mistral intégré n'utilisent pas ce paramètre ; continuez à utiliser les modèles `magistral-*` lorsque vous voulez le comportement natif de Mistral orienté raisonnement en premier.

## Remarques

- L'authentification Mistral utilise `MISTRAL_API_KEY`.
- L'URL de base du fournisseur est par défaut `https://api.mistral.ai/v1`.
- Le modèle par défaut de l'onboarding est `mistral/mistral-large-latest`.
- Le modèle audio par défaut de l'analyse des médias pour Mistral est `voxtral-mini-latest`.
- Le chemin de transcription média utilise `/v1/audio/transcriptions`.
- Le chemin des embeddings mémoire utilise `/v1/embeddings` (modèle par défaut : `mistral-embed`).
