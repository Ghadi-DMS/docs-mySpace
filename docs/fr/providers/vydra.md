---
read_when:
    - Vous voulez la génération de médias Vydra dans OpenClaw
    - Vous avez besoin d'instructions pour configurer la clé API Vydra
summary: Utiliser la génération d'images, de vidéos et la synthèse vocale Vydra dans OpenClaw
title: Vydra
x-i18n:
    generated_at: "2026-04-07T06:53:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 24006a687ed6f9792e7b2b10927cc7ad71c735462a92ce03d5fa7c2b2ee2fcc2
    source_path: providers/vydra.md
    workflow: 15
---

# Vydra

Le plugin Vydra intégré ajoute :

- la génération d'images via `vydra/grok-imagine`
- la génération de vidéos via `vydra/veo3` et `vydra/kling`
- la synthèse vocale via la route TTS de Vydra basée sur ElevenLabs

OpenClaw utilise la même `VYDRA_API_KEY` pour ces trois capacités.

## URL de base importante

Utilisez `https://www.vydra.ai/api/v1`.

L'hôte apex de Vydra (`https://vydra.ai/api/v1`) redirige actuellement vers `www`. Certains clients HTTP abandonnent `Authorization` lors de cette redirection inter-hôtes, ce qui transforme une clé API valide en échec d'authentification trompeur. Le plugin intégré utilise directement l'URL de base `www` pour éviter cela.

## Configuration

Onboarding interactif :

```bash
openclaw onboard --auth-choice vydra-api-key
```

Ou définissez directement la variable d'environnement :

```bash
export VYDRA_API_KEY="vydra_live_..."
```

## Génération d'images

Modèle d'image par défaut :

- `vydra/grok-imagine`

Définissez-le comme fournisseur d'images par défaut :

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "vydra/grok-imagine",
      },
    },
  },
}
```

La prise en charge intégrée actuelle se limite au texte-vers-image. Les routes d'édition hébergées de Vydra attendent des URL d'image distantes, et OpenClaw n'ajoute pas encore de pont de téléversement spécifique à Vydra dans le plugin intégré.

Voir [Génération d'images](/fr/tools/image-generation) pour le comportement partagé de l'outil.

## Génération de vidéos

Modèles vidéo enregistrés :

- `vydra/veo3` pour le texte-vers-vidéo
- `vydra/kling` pour l'image-vers-vidéo

Définissez Vydra comme fournisseur vidéo par défaut :

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "vydra/veo3",
      },
    },
  },
}
```

Remarques :

- `vydra/veo3` est intégré uniquement en mode texte-vers-vidéo.
- `vydra/kling` exige actuellement une référence d'URL d'image distante. Les téléversements de fichiers locaux sont rejetés immédiatement.
- La route HTTP `kling` actuelle de Vydra a été incohérente quant au fait d'exiger `image_url` ou `video_url` ; le fournisseur intégré mappe la même URL d'image distante vers les deux champs.
- Le plugin intégré reste prudent et ne transmet pas de paramètres de style non documentés tels que le ratio d'aspect, la résolution, le filigrane ou l'audio généré.

Couverture live spécifique au fournisseur :

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_LIVE_VYDRA_VIDEO=1 \
pnpm test:live -- extensions/vydra/vydra.live.test.ts
```

Le fichier live Vydra intégré couvre maintenant :

- `vydra/veo3` texte-vers-vidéo
- `vydra/kling` image-vers-vidéo à l'aide d'une URL d'image distante

Remplacez la fixture d'image distante si nécessaire :

```bash
export OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL="https://example.com/reference.png"
```

Voir [Génération de vidéos](/fr/tools/video-generation) pour le comportement partagé de l'outil.

## Synthèse vocale

Définissez Vydra comme fournisseur de parole :

```json5
{
  messages: {
    tts: {
      provider: "vydra",
      providers: {
        vydra: {
          apiKey: "${VYDRA_API_KEY}",
          voiceId: "21m00Tcm4TlvDq8ikWAM",
        },
      },
    },
  },
}
```

Valeurs par défaut :

- modèle : `elevenlabs/tts`
- ID de voix : `21m00Tcm4TlvDq8ikWAM`

Le plugin intégré expose actuellement une seule voix par défaut connue comme fiable et renvoie des fichiers audio MP3.

## Liens associés

- [Répertoire des fournisseurs](/fr/providers/index)
- [Génération d'images](/fr/tools/image-generation)
- [Génération de vidéos](/fr/tools/video-generation)
