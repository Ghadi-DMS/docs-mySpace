---
read_when:
    - Vous voulez utiliser la génération d’images fal dans OpenClaw
    - Vous avez besoin du flux d’authentification FAL_KEY
    - Vous voulez des valeurs par défaut fal pour `image_generate` ou `video_generate`
summary: Configuration de la génération d’images et de vidéos fal dans OpenClaw
title: fal
x-i18n:
    generated_at: "2026-04-11T02:47:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9bfe4f69124e922a79a516a1bd78f0c00f7a45f3c6f68b6d39e0d196fa01beb3
    source_path: providers/fal.md
    workflow: 15
---

# fal

OpenClaw inclut un provider `fal` bundle pour la génération d’images et de vidéos hébergée.

- Provider : `fal`
- Authentification : `FAL_KEY` (canonique ; `FAL_API_KEY` fonctionne aussi comme solution de secours)
- API : endpoints de modèles fal

## Démarrage rapide

1. Définissez la clé API :

```bash
openclaw onboard --auth-choice fal-api-key
```

2. Définissez un modèle d’image par défaut :

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## Génération d’images

Le provider de génération d’images `fal` bundle utilise par défaut
`fal/fal-ai/flux/dev`.

- Génération : jusqu’à 4 images par requête
- Mode édition : activé, 1 image de référence
- Prend en charge `size`, `aspectRatio` et `resolution`
- Limitation actuelle de l’édition : l’endpoint d’édition d’image fal ne prend **pas** en charge les remplacements de `aspectRatio`

Pour utiliser fal comme provider d’image par défaut :

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## Génération de vidéos

Le provider de génération de vidéos `fal` bundle utilise par défaut
`fal/fal-ai/minimax/video-01-live`.

- Modes : texte-vers-vidéo et flux à image de référence unique
- Runtime : flux submit/status/result adossé à une file pour les tâches de longue durée
- Référence de modèle d’agent vidéo HeyGen :
  - `fal/fal-ai/heygen/v2/video-agent`
- Références de modèle Seedance 2.0 :
  - `fal/bytedance/seedance-2.0/fast/text-to-video`
  - `fal/bytedance/seedance-2.0/fast/image-to-video`
  - `fal/bytedance/seedance-2.0/text-to-video`
  - `fal/bytedance/seedance-2.0/image-to-video`

Pour utiliser Seedance 2.0 comme modèle vidéo par défaut :

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
      },
    },
  },
}
```

Pour utiliser l’agent vidéo HeyGen comme modèle vidéo par défaut :

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/fal-ai/heygen/v2/video-agent",
      },
    },
  },
}
```

## Liens associés

- [Génération d’images](/fr/tools/image-generation)
- [Génération de vidéos](/fr/tools/video-generation)
- [Référence de configuration](/fr/gateway/configuration-reference#agent-defaults)
