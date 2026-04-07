---
read_when:
    - Vous voulez utiliser les modèles Google Gemini avec OpenClaw
    - Vous avez besoin du flux d’authentification par clé API ou par OAuth
summary: Configuration de Google Gemini (clé API + OAuth, génération d’images, compréhension des médias, recherche web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-07T06:53:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 36cc7c7d8d19f6d4a3fb223af36c8402364fc309d14ffe922bd004203ceb1754
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Le plugin Google fournit un accès aux modèles Gemini via Google AI Studio, ainsi que
la génération d’images, la compréhension des médias (image/audio/vidéo) et la recherche web via
Gemini Grounding.

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

Un fournisseur alternatif, `google-gemini-cli`, utilise OAuth PKCE au lieu d’une
clé API. Il s’agit d’une intégration non officielle ; certains utilisateurs signalent des
restrictions de compte. Utilisez-la à vos propres risques.

- Modèle par défaut : `google-gemini-cli/gemini-3.1-pro-preview`
- Alias : `gemini-cli`
- Prérequis d’installation : la CLI Gemini locale doit être disponible comme `gemini`
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

Si les requêtes OAuth de Gemini CLI échouent après la connexion, définissez
`GOOGLE_CLOUD_PROJECT` ou `GOOGLE_CLOUD_PROJECT_ID` sur l’hôte gateway puis
réessayez.

Si la connexion échoue avant le démarrage du flux navigateur, assurez-vous que la commande locale `gemini`
est installée et présente dans `PATH`. OpenClaw prend en charge à la fois les installations Homebrew
et les installations npm globales, y compris les dispositions Windows/npm courantes.

Remarques sur l’utilisation JSON de Gemini CLI :

- Le texte de réponse provient du champ JSON `response` de la CLI.
- L’utilisation revient à `stats` lorsque la CLI laisse `usage` vide.
- `stats.cached` est normalisé en `cacheRead` dans OpenClaw.
- Si `stats.input` est absent, OpenClaw dérive les jetons d’entrée à partir de
  `stats.input_tokens - stats.cached`.

## Capacités

| Capacité               | Pris en charge      |
| ---------------------- | ------------------- |
| Complétions de chat    | Oui                 |
| Génération d’images    | Oui                 |
| Génération musicale    | Oui                 |
| Compréhension d’image  | Oui                 |
| Transcription audio    | Oui                 |
| Compréhension vidéo    | Oui                 |
| Recherche web (Grounding) | Oui              |
| Réflexion/raisonnement | Oui (Gemini 3.1+)   |

## Réutilisation directe du cache Gemini

Pour les exécutions directes de l’API Gemini (`api: "google-generative-ai"`), OpenClaw
transmet désormais un handle `cachedContent` configuré aux requêtes Gemini.

- Configurez des paramètres par modèle ou globaux avec
  `cachedContent` ou l’ancien `cached_content`
- Si les deux sont présents, `cachedContent` l’emporte
- Exemple de valeur : `cachedContents/prebuilt-context`
- L’utilisation de cache Gemini lors d’un cache hit est normalisée en `cacheRead` dans OpenClaw à partir de
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

Le fournisseur intégré de génération d’images `google` utilise par défaut
`google/gemini-3.1-flash-image-preview`.

- Prend aussi en charge `google/gemini-3-pro-image-preview`
- Génération : jusqu’à 4 images par requête
- Mode édition : activé, jusqu’à 5 images d’entrée
- Contrôles de géométrie : `size`, `aspectRatio` et `resolution`

Le fournisseur `google-gemini-cli`, uniquement OAuth, constitue une surface d’inférence texte distincte. La génération d’images, la compréhension des médias et Gemini Grounding restent sur
l’id de fournisseur `google`.

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

Voir [Génération d’images](/fr/tools/image-generation) pour les
paramètres d’outils partagés, la sélection du fournisseur et le comportement de basculement.

## Génération vidéo

Le plugin intégré `google` enregistre aussi la génération vidéo via l’outil partagé
`video_generate`.

- Modèle vidéo par défaut : `google/veo-3.1-fast-generate-preview`
- Modes : texte-vers-vidéo, image-vers-vidéo et flux à référence vidéo unique
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

Voir [Génération vidéo](/fr/tools/video-generation) pour les paramètres
d’outils partagés, la sélection du fournisseur et le comportement de basculement.

## Génération musicale

Le plugin intégré `google` enregistre aussi la génération musicale via l’outil partagé
`music_generate`.

- Modèle musical par défaut : `google/lyria-3-clip-preview`
- Prend aussi en charge `google/lyria-3-pro-preview`
- Contrôles de prompt : `lyrics` et `instrumental`
- Format de sortie : `mp3` par défaut, avec `wav` en plus sur `google/lyria-3-pro-preview`
- Entrées de référence : jusqu’à 10 images
- Les exécutions adossées à une session se détachent via le flux partagé de tâche/statut, y compris `action: "status"`

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

Voir [Génération musicale](/fr/tools/music-generation) pour les paramètres
d’outils partagés, la sélection du fournisseur et le comportement de basculement.

## Remarque sur l’environnement

Si la Gateway s’exécute comme démon (launchd/systemd), assurez-vous que `GEMINI_API_KEY`
est disponible pour ce processus (par exemple, dans `~/.openclaw/.env` ou via
`env.shellEnv`).
