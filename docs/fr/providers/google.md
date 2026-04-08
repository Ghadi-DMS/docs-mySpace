---
read_when:
    - Vous souhaitez utiliser les modèles Google Gemini avec OpenClaw
    - Vous avez besoin du flux d’authentification par clé API ou OAuth
summary: Configuration de Google Gemini (clé API + OAuth, génération d’images, compréhension des médias, recherche Web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-08T06:51:01Z"
    model: gpt-5.4
    provider: openai
    source_hash: fad2ff68987301bd86145fa6e10de8c7b38d5bd5dbcd13db9c883f7f5b9a4e01
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Le plugin Google fournit l’accès aux modèles Gemini via Google AI Studio, ainsi que la génération d’images, la compréhension des médias (image/audio/vidéo) et la recherche Web via Gemini Grounding.

- Fournisseur : `google`
- Authentification : `GEMINI_API_KEY` ou `GOOGLE_API_KEY`
- API : API Google Gemini
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

Un fournisseur alternatif `google-gemini-cli` utilise OAuth PKCE au lieu d’une clé API. Il s’agit d’une intégration non officielle ; certains utilisateurs signalent des restrictions de compte. Utilisez-la à vos propres risques.

- Modèle par défaut : `google-gemini-cli/gemini-3-flash-preview`
- Alias : `gemini-cli`
- Prérequis d’installation : la Gemini CLI locale doit être disponible sous le nom `gemini`
  - Homebrew : `brew install gemini-cli`
  - npm : `npm install -g @google/gemini-cli`
- Connexion :

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

Variables d’environnement :

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(Ou les variantes `GEMINI_CLI_*`.)

Si les requêtes OAuth de Gemini CLI échouent après la connexion, définissez `GOOGLE_CLOUD_PROJECT` ou `GOOGLE_CLOUD_PROJECT_ID` sur l’hôte de la passerelle, puis réessayez.

Si la connexion échoue avant le démarrage du flux dans le navigateur, assurez-vous que la commande locale `gemini` est installée et présente dans le `PATH`. OpenClaw prend en charge les installations Homebrew et les installations npm globales, y compris les dispositions Windows/npm courantes.

Remarques sur l’utilisation du JSON de Gemini CLI :

- Le texte de réponse provient du champ JSON `response` de la CLI.
- L’utilisation se rabat sur `stats` lorsque la CLI laisse `usage` vide.
- `stats.cached` est normalisé en `cacheRead` dans OpenClaw.
- Si `stats.input` est absent, OpenClaw dérive les jetons d’entrée à partir de `stats.input_tokens - stats.cached`.

## Capacités

| Capacité                | Pris en charge    |
| ----------------------- | ----------------- |
| Complétions de chat     | Oui               |
| Génération d’images     | Oui               |
| Génération de musique   | Oui               |
| Compréhension d’images  | Oui               |
| Transcription audio     | Oui               |
| Compréhension vidéo     | Oui               |
| Recherche Web (Grounding) | Oui             |
| Thinking/reasoning      | Oui (Gemini 3.1+) |
| Modèles Gemma 4         | Oui               |

Les modèles Gemma 4 (par exemple `gemma-4-26b-a4b-it`) prennent en charge le mode thinking. OpenClaw réécrit `thinkingBudget` vers un `thinkingLevel` Google pris en charge pour Gemma 4. Définir thinking sur `off` conserve la désactivation de thinking au lieu d’un mappage vers `MINIMAL`.

## Réutilisation directe du cache Gemini

Pour les exécutions directes de l’API Gemini (`api: "google-generative-ai"`), OpenClaw transmet désormais un handle `cachedContent` configuré aux requêtes Gemini.

- Configurez des paramètres par modèle ou globaux avec
  `cachedContent` ou l’ancien `cached_content`
- Si les deux sont présents, `cachedContent` est prioritaire
- Exemple de valeur : `cachedContents/prebuilt-context`
- L’utilisation en cas de réussite du cache Gemini est normalisée en `cacheRead` dans OpenClaw à partir de `cachedContentTokenCount` en amont

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

Le fournisseur de génération d’images `google` inclus utilise par défaut `google/gemini-3.1-flash-image-preview`.

- Prend aussi en charge `google/gemini-3-pro-image-preview`
- Génération : jusqu’à 4 images par requête
- Mode édition : activé, jusqu’à 5 images d’entrée
- Contrôles de géométrie : `size`, `aspectRatio` et `resolution`

Le fournisseur `google-gemini-cli`, uniquement OAuth, constitue une surface d’inférence de texte distincte. La génération d’images, la compréhension des médias et Gemini Grounding restent associées à l’identifiant de fournisseur `google`.

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

Consultez [Image Generation](/fr/tools/image-generation) pour les paramètres d’outil partagés, la sélection du fournisseur et le comportement de basculement.

## Génération de vidéo

Le plugin `google` inclus enregistre également la génération de vidéo via l’outil partagé `video_generate`.

- Modèle vidéo par défaut : `google/veo-3.1-fast-generate-preview`
- Modes : texte vers vidéo, image vers vidéo et flux à référence vidéo unique
- Prend en charge `aspectRatio`, `resolution` et `audio`
- Limitation actuelle de durée : **4 à 8 secondes**

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

Consultez [Video Generation](/fr/tools/video-generation) pour les paramètres d’outil partagés, la sélection du fournisseur et le comportement de basculement.

## Génération de musique

Le plugin `google` inclus enregistre également la génération de musique via l’outil partagé `music_generate`.

- Modèle musical par défaut : `google/lyria-3-clip-preview`
- Prend aussi en charge `google/lyria-3-pro-preview`
- Contrôles d’invite : `lyrics` et `instrumental`
- Format de sortie : `mp3` par défaut, ainsi que `wav` sur `google/lyria-3-pro-preview`
- Entrées de référence : jusqu’à 10 images
- Les exécutions avec session se détachent via le flux partagé de tâche/statut, y compris `action: "status"`

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

Consultez [Music Generation](/fr/tools/music-generation) pour les paramètres d’outil partagés, la sélection du fournisseur et le comportement de basculement.

## Remarque sur l’environnement

Si la passerelle s’exécute en tant que démon (launchd/systemd), assurez-vous que `GEMINI_API_KEY` est disponible pour ce processus (par exemple dans `~/.openclaw/.env` ou via `env.shellEnv`).
