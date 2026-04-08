---
read_when:
    - Vous souhaitez utiliser les modèles Google Gemini avec OpenClaw
    - Vous avez besoin du flux d’authentification par clé API ou OAuth
summary: Configuration de Google Gemini (clé API + OAuth, génération d’images, compréhension des médias, recherche web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-08T02:17:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: e9e558f5ce35c853e0240350be9a1890460c5f7f7fd30b05813a656497dee516
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Le plugin Google donne accès aux modèles Gemini via Google AI Studio, ainsi qu’à la
génération d’images, à la compréhension des médias (image/audio/vidéo) et à la recherche web via
Gemini Grounding.

- Fournisseur : `google`
- Auth : `GEMINI_API_KEY` ou `GOOGLE_API_KEY`
- API : Google Gemini API
- Fournisseur alternatif : `google-gemini-cli` (OAuth)

## Démarrage rapide

1. Définissez la clé API :

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Définissez un modèle par défaut :

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## Exemple non interactif

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## OAuth (Gemini CLI)

Un fournisseur alternatif, `google-gemini-cli`, utilise OAuth PKCE au lieu d’une
clé API. Il s’agit d’une intégration non officielle ; certains utilisateurs signalent des
restrictions de compte. Utilisez-la à vos propres risques.

- Modèle par défaut : `google-gemini-cli/gemini-3-flash-preview`
- Alias : `gemini-cli`
- Prérequis d’installation : Gemini CLI disponible localement sous le nom `gemini`
  - Homebrew : `brew install gemini-cli`
  - npm : `npm install -g @google/gemini-cli`
- Connexion :

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

Variables d’environnement :

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(ou les variantes `GEMINI_CLI_*`).

Si les requêtes OAuth de Gemini CLI échouent après la connexion, définissez
`GOOGLE_CLOUD_PROJECT` ou `GOOGLE_CLOUD_PROJECT_ID` sur l’hôte de la passerelle et
réessayez.

Si la connexion échoue avant le démarrage du flux du navigateur, assurez-vous que la commande locale `gemini`
est installée et présente dans le `PATH`. OpenClaw prend en charge les installations Homebrew
et les installations npm globales, y compris les dispositions Windows/npm courantes.

Notes sur l’utilisation du JSON de Gemini CLI :

- Le texte de réponse provient du champ JSON `response` du CLI.
- L’utilisation bascule sur `stats` lorsque le CLI laisse `usage` vide.
- `stats.cached` est normalisé en `cacheRead` OpenClaw.
- Si `stats.input` est absent, OpenClaw dérive les jetons d’entrée à partir de
  `stats.input_tokens - stats.cached`.

## Capacités

| Capacité                 | Pris en charge     |
| ------------------------ | ------------------ |
| Complétions de chat      | Oui                |
| Génération d’images      | Oui                |
| Génération musicale      | Oui                |
| Compréhension d’images   | Oui                |
| Transcription audio      | Oui                |
| Compréhension vidéo      | Oui                |
| Recherche web (Grounding) | Oui               |
| Thinking/reasoning       | Oui (Gemini 3.1+)  |

## Réutilisation directe du cache Gemini

Pour les exécutions directes de l’API Gemini (`api: "google-generative-ai"`), OpenClaw
transmet désormais un handle `cachedContent` configuré aux requêtes Gemini.

- Configurez des paramètres par modèle ou globaux avec
  `cachedContent` ou l’ancien `cached_content`
- Si les deux sont présents, `cachedContent` est prioritaire
- Exemple de valeur : `cachedContents/prebuilt-context`
- L’utilisation d’un accès au cache Gemini est normalisée dans `cacheRead` OpenClaw à partir de
  `cachedContentTokenCount` en amont

Exemple :

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## Génération d’images

Le fournisseur groupé de génération d’images `google` utilise par défaut
`google/gemini-3.1-flash-image-preview`.

- Prend aussi en charge `google/gemini-3-pro-image-preview`
- Génération : jusqu’à 4 images par requête
- Mode édition : activé, jusqu’à 5 images d’entrée
- Contrôles de géométrie : `size`, `aspectRatio` et `resolution`

Le fournisseur `google-gemini-cli`, limité à OAuth, constitue une surface distincte
d’inférence de texte. La génération d’images, la compréhension des médias et Gemini Grounding restent sur
l’identifiant de fournisseur `google`.

Pour utiliser Google comme fournisseur d’images par défaut :

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

Voir [Génération d’images](/fr/tools/image-generation) pour les paramètres partagés de l’outil,
la sélection du fournisseur et le comportement de basculement.

## Génération de vidéo

Le plugin groupé `google` enregistre aussi la génération vidéo via l’outil partagé
`video_generate`.

- Modèle vidéo par défaut : `google/veo-3.1-fast-generate-preview`
- Modes : texte vers vidéo, image vers vidéo et flux de référence à vidéo unique
- Prend en charge `aspectRatio`, `resolution` et `audio`
- Limite actuelle de durée : **4 à 8 secondes**

Pour utiliser Google comme fournisseur vidéo par défaut :

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

Voir [Génération de vidéo](/fr/tools/video-generation) pour les paramètres partagés de l’outil,
la sélection du fournisseur et le comportement de basculement.

## Génération musicale

Le plugin groupé `google` enregistre aussi la génération musicale via l’outil partagé
`music_generate`.

- Modèle musical par défaut : `google/lyria-3-clip-preview`
- Prend aussi en charge `google/lyria-3-pro-preview`
- Contrôles du prompt : `lyrics` et `instrumental`
- Format de sortie : `mp3` par défaut, plus `wav` sur `google/lyria-3-pro-preview`
- Entrées de référence : jusqu’à 10 images
- Les exécutions appuyées sur une session se détachent via le flux partagé tâche/état, y compris `action: "status"`

Pour utiliser Google comme fournisseur musical par défaut :

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

Voir [Génération musicale](/fr/tools/music-generation) pour les paramètres partagés de l’outil,
la sélection du fournisseur et le comportement de basculement.

## Remarque sur l’environnement

Si la passerelle s’exécute comme daemon (launchd/systemd), assurez-vous que `GEMINI_API_KEY`
est disponible pour ce processus (par exemple dans `~/.openclaw/.env` ou via
`env.shellEnv`).
