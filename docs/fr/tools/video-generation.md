---
read_when:
    - Générer des vidéos via l’agent
    - Configurer les fournisseurs et modèles de génération vidéo
    - Comprendre les paramètres de l’outil `video_generate`
summary: Générez des vidéos à partir de texte, d’images ou de vidéos existantes à l’aide de 12 backends fournisseurs
title: Génération vidéo
x-i18n:
    generated_at: "2026-04-07T06:56:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: bf1224c59a5f1217f56cf2001870aca710a09268677dcd12aad2efbe476e47b7
    source_path: tools/video-generation.md
    workflow: 15
---

# Génération vidéo

Les agents OpenClaw peuvent générer des vidéos à partir de prompts textuels, d’images de référence ou de vidéos existantes. Douze backends fournisseurs sont pris en charge, chacun avec des options de modèle, des modes d’entrée et des ensembles de fonctionnalités différents. L’agent choisit automatiquement le bon fournisseur en fonction de votre configuration et des clés API disponibles.

<Note>
L’outil `video_generate` n’apparaît que lorsqu’au moins un fournisseur de génération vidéo est disponible. Si vous ne le voyez pas dans les outils de votre agent, définissez une clé API de fournisseur ou configurez `agents.defaults.videoGenerationModel`.
</Note>

OpenClaw traite la génération vidéo selon trois modes runtime :

- `generate` pour les requêtes texte-vers-vidéo sans média de référence
- `imageToVideo` lorsque la requête inclut une ou plusieurs images de référence
- `videoToVideo` lorsque la requête inclut une ou plusieurs vidéos de référence

Les fournisseurs peuvent prendre en charge n’importe quel sous-ensemble de ces modes. L’outil valide le
mode actif avant l’envoi et signale les modes pris en charge dans `action=list`.

## Démarrage rapide

1. Définissez une clé API pour n’importe quel fournisseur pris en charge :

```bash
export GEMINI_API_KEY="your-key"
```

2. Épinglez éventuellement un modèle par défaut :

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
```

3. Demandez à l’agent :

> Génère une vidéo cinématographique de 5 secondes d’un homard amical faisant du surf au coucher du soleil.

L’agent appelle automatiquement `video_generate`. Aucune liste d’autorisation d’outil n’est nécessaire.

## Ce qui se passe lorsque vous générez une vidéo

La génération vidéo est asynchrone. Lorsque l’agent appelle `video_generate` dans une session :

1. OpenClaw envoie la requête au fournisseur et renvoie immédiatement un id de tâche.
2. Le fournisseur traite le travail en arrière-plan (généralement de 30 secondes à 5 minutes selon le fournisseur et la résolution).
3. Lorsque la vidéo est prête, OpenClaw réactive la même session avec un événement interne de fin.
4. L’agent publie la vidéo terminée dans la conversation d’origine.

Lorsqu’une tâche est en cours, les appels `video_generate` en doublon dans la même session renvoient l’état de la tâche en cours au lieu de démarrer une nouvelle génération. Utilisez `openclaw tasks list` ou `openclaw tasks show <taskId>` pour vérifier la progression depuis la CLI.

En dehors des exécutions d’agent adossées à une session (par exemple, des invocations directes d’outils), l’outil revient à une génération inline et renvoie le chemin du média final au même tour.

### Cycle de vie d’une tâche

Chaque requête `video_generate` passe par quatre états :

1. **queued** -- tâche créée, en attente d’acceptation par le fournisseur.
2. **running** -- le fournisseur traite la demande (généralement de 30 secondes à 5 minutes selon le fournisseur et la résolution).
3. **succeeded** -- la vidéo est prête ; l’agent se réactive et la publie dans la conversation.
4. **failed** -- erreur du fournisseur ou délai dépassé ; l’agent se réactive avec les détails de l’erreur.

Vérifiez l’état depuis la CLI :

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

Prévention des doublons : si une tâche vidéo est déjà `queued` ou `running` pour la session actuelle, `video_generate` renvoie l’état de la tâche existante au lieu d’en démarrer une nouvelle. Utilisez `action: "status"` pour vérifier explicitement sans déclencher une nouvelle génération.

## Fournisseurs pris en charge

| Fournisseur | Modèle par défaut              | Texte | Réf. image        | Réf. vidéo       | Clé API                                  |
| ----------- | ------------------------------ | ----- | ----------------- | ---------------- | ---------------------------------------- |
| Alibaba     | `wan2.6-t2v`                   | Oui   | Oui (URL distante) | Oui (URL distante) | `MODELSTUDIO_API_KEY`                  |
| BytePlus    | `seedance-1-0-lite-t2v-250428` | Oui   | 1 image           | Non              | `BYTEPLUS_API_KEY`                       |
| ComfyUI     | `workflow`                     | Oui   | 1 image           | Non              | `COMFY_API_KEY` ou `COMFY_CLOUD_API_KEY` |
| fal         | `fal-ai/minimax/video-01-live` | Oui   | 1 image           | Non              | `FAL_KEY`                                |
| Google      | `veo-3.1-fast-generate-preview` | Oui  | 1 image           | 1 vidéo          | `GEMINI_API_KEY`                         |
| MiniMax     | `MiniMax-Hailuo-2.3`           | Oui   | 1 image           | Non              | `MINIMAX_API_KEY`                        |
| OpenAI      | `sora-2`                       | Oui   | 1 image           | 1 vidéo          | `OPENAI_API_KEY`                         |
| Qwen        | `wan2.6-t2v`                   | Oui   | Oui (URL distante) | Oui (URL distante) | `QWEN_API_KEY`                         |
| Runway      | `gen4.5`                       | Oui   | 1 image           | 1 vidéo          | `RUNWAYML_API_SECRET`                    |
| Together    | `Wan-AI/Wan2.2-T2V-A14B`       | Oui   | 1 image           | Non              | `TOGETHER_API_KEY`                       |
| Vydra       | `veo3`                         | Oui   | 1 image (`kling`) | Non              | `VYDRA_API_KEY`                          |
| xAI         | `grok-imagine-video`           | Oui   | 1 image           | 1 vidéo          | `XAI_API_KEY`                            |

Certains fournisseurs acceptent des variables d’environnement de clé API supplémentaires ou alternatives. Consultez les [pages fournisseur](#related) individuelles pour plus de détails.

Exécutez `video_generate action=list` pour inspecter à l’exécution les fournisseurs, modèles et
modes runtime disponibles.

### Matrice des capacités déclarées

Il s’agit du contrat de mode explicite utilisé par `video_generate`, les tests de contrat
et le balayage live partagé.

| Fournisseur | `generate` | `imageToVideo` | `videoToVideo` | Voies live partagées aujourd’hui                                                                                                        |
| ----------- | ---------- | -------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba     | Oui        | Oui            | Oui            | `generate`, `imageToVideo` ; `videoToVideo` ignoré car ce fournisseur nécessite des URL vidéo distantes `http(s)`                      |
| BytePlus    | Oui        | Oui            | Non            | `generate`, `imageToVideo`                                                                                                               |
| ComfyUI     | Oui        | Oui            | Non            | Pas dans le balayage partagé ; la couverture spécifique au workflow se trouve dans les tests Comfy                                      |
| fal         | Oui        | Oui            | Non            | `generate`, `imageToVideo`                                                                                                               |
| Google      | Oui        | Oui            | Oui            | `generate`, `imageToVideo` ; `videoToVideo` partagé ignoré car le balayage Gemini/Veo actuel adossé à un buffer n’accepte pas cette entrée |
| MiniMax     | Oui        | Oui            | Non            | `generate`, `imageToVideo`                                                                                                               |
| OpenAI      | Oui        | Oui            | Oui            | `generate`, `imageToVideo` ; `videoToVideo` partagé ignoré car ce chemin org/entrée nécessite actuellement un accès fournisseur à l’inpaint/remix |
| Qwen        | Oui        | Oui            | Oui            | `generate`, `imageToVideo` ; `videoToVideo` ignoré car ce fournisseur nécessite des URL vidéo distantes `http(s)`                      |
| Runway      | Oui        | Oui            | Oui            | `generate`, `imageToVideo` ; `videoToVideo` ne s’exécute que lorsque le modèle sélectionné est `runway/gen4_aleph`                     |
| Together    | Oui        | Oui            | Non            | `generate`, `imageToVideo`                                                                                                               |
| Vydra       | Oui        | Oui            | Non            | `generate` ; `imageToVideo` partagé ignoré car `veo3` intégré est texte seul et `kling` intégré nécessite une URL d’image distante     |
| xAI         | Oui        | Oui            | Oui            | `generate`, `imageToVideo` ; `videoToVideo` ignoré car ce fournisseur nécessite actuellement une URL MP4 distante                       |

## Paramètres de l’outil

### Obligatoires

| Paramètre | Type   | Description                                                                    |
| --------- | ------ | ------------------------------------------------------------------------------ |
| `prompt`  | string | Description textuelle de la vidéo à générer (obligatoire pour `action: "generate"`) |

### Entrées de contenu

| Paramètre | Type     | Description                            |
| --------- | -------- | -------------------------------------- |
| `image`   | string   | Image de référence unique (chemin ou URL) |
| `images`  | string[] | Plusieurs images de référence (jusqu’à 5) |
| `video`   | string   | Vidéo de référence unique (chemin ou URL) |
| `videos`  | string[] | Plusieurs vidéos de référence (jusqu’à 4) |

### Contrôles de style

| Paramètre         | Type    | Description                                                              |
| ----------------- | ------- | ------------------------------------------------------------------------ |
| `aspectRatio`     | string  | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` |
| `resolution`      | string  | `480P`, `720P`, `768P`, ou `1080P`                                       |
| `durationSeconds` | number  | Durée cible en secondes (arrondie à la valeur prise en charge la plus proche par le fournisseur) |
| `size`            | string  | Indication de taille lorsque le fournisseur la prend en charge           |
| `audio`           | boolean | Active l’audio généré lorsque pris en charge                             |
| `watermark`       | boolean | Active/désactive le filigrane du fournisseur lorsque pris en charge      |

### Avancés

| Paramètre  | Type   | Description                                      |
| ---------- | ------ | ------------------------------------------------ |
| `action`   | string | `"generate"` (par défaut), `"status"` ou `"list"` |
| `model`    | string | Remplacement de fournisseur/modèle (par ex. `runway/gen4.5`) |
| `filename` | string | Indication de nom de fichier de sortie           |

Tous les fournisseurs ne prennent pas en charge tous les paramètres. OpenClaw normalise déjà la durée vers la valeur la plus proche prise en charge par le fournisseur, et remappe aussi les indications de géométrie traduites, par exemple de size vers aspect-ratio, lorsqu’un fournisseur de repli expose une surface de contrôle différente. Les remplacements réellement non pris en charge sont ignorés au mieux et signalés sous forme d’avertissements dans le résultat de l’outil. Les limites strictes de capacité (comme trop d’entrées de référence) échouent avant l’envoi.

Les résultats de l’outil indiquent les paramètres appliqués. Lorsque OpenClaw remappe la durée ou la géométrie lors d’un basculement de fournisseur, les valeurs renvoyées `durationSeconds`, `size`, `aspectRatio` et `resolution` reflètent ce qui a été envoyé, et `details.normalization` capture la traduction entre la demande et l’application.

Les entrées de référence sélectionnent aussi le mode runtime :

- Aucun média de référence : `generate`
- Toute image de référence : `imageToVideo`
- Toute vidéo de référence : `videoToVideo`

Les références d’image et de vidéo mélangées ne constituent pas une surface de capacité partagée stable.
Préférez un seul type de référence par requête.

## Actions

- **generate** (par défaut) -- créer une vidéo à partir du prompt donné et d’éventuelles entrées de référence.
- **status** -- vérifier l’état de la tâche vidéo en cours pour la session actuelle sans démarrer une autre génération.
- **list** -- afficher les fournisseurs, modèles et capacités disponibles.

## Sélection du modèle

Lors de la génération d’une vidéo, OpenClaw résout le modèle dans cet ordre :

1. **Paramètre d’outil `model`** -- si l’agent en spécifie un dans l’appel.
2. **`videoGenerationModel.primary`** -- depuis la config.
3. **`videoGenerationModel.fallbacks`** -- essayés dans l’ordre.
4. **Détection automatique** -- utilise les fournisseurs disposant d’une authentification valide, en commençant par le fournisseur par défaut actuel, puis les autres fournisseurs par ordre alphabétique.

Si un fournisseur échoue, le candidat suivant est essayé automatiquement. Si tous les candidats échouent, l’erreur inclut les détails de chaque tentative.

Définissez `agents.defaults.mediaGenerationAutoProviderFallback: false` si vous voulez
que la génération vidéo utilise uniquement les entrées explicites `model`, `primary` et `fallbacks`.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
      },
    },
  },
}
```

## Remarques sur les fournisseurs

| Fournisseur | Remarques                                                                                                                                                    |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba     | Utilise le point de terminaison asynchrone DashScope/Model Studio. Les images et vidéos de référence doivent être des URL distantes `http(s)`.             |
| BytePlus    | Une seule image de référence.                                                                                                                                |
| ComfyUI     | Exécution locale ou cloud pilotée par workflow. Prend en charge le texte-vers-vidéo et l’image-vers-vidéo via le graphe configuré.                         |
| fal         | Utilise un flux adossé à une file pour les travaux de longue durée. Une seule image de référence.                                                           |
| Google      | Utilise Gemini/Veo. Prend en charge une image ou une vidéo de référence.                                                                                     |
| MiniMax     | Une seule image de référence.                                                                                                                                |
| OpenAI      | Seul le remplacement `size` est transmis. Les autres remplacements de style (`aspectRatio`, `resolution`, `audio`, `watermark`) sont ignorés avec un avertissement. |
| Qwen        | Même backend DashScope qu’Alibaba. Les entrées de référence doivent être des URL distantes `http(s)` ; les fichiers locaux sont rejetés d’emblée.          |
| Runway      | Prend en charge les fichiers locaux via des URI de données. Le mode vidéo-vers-vidéo nécessite `runway/gen4_aleph`. Les exécutions texte seul exposent les rapports d’aspect `16:9` et `9:16`. |
| Together    | Une seule image de référence.                                                                                                                                |
| Vydra       | Utilise directement `https://www.vydra.ai/api/v1` pour éviter les redirections qui perdent l’authentification. `veo3` est intégré comme texte-vers-vidéo uniquement ; `kling` nécessite une URL d’image distante. |
| xAI         | Prend en charge les flux texte-vers-vidéo, image-vers-vidéo et les flux distants d’édition/extension vidéo.                                                |

## Modes de capacité des fournisseurs

Le contrat partagé de génération vidéo permet désormais aux fournisseurs de déclarer des
capacités spécifiques à un mode au lieu de seulement des limites agrégées plates. Les nouvelles
implémentations de fournisseurs devraient privilégier des blocs de mode explicites :

```typescript
capabilities: {
  generate: {
    maxVideos: 1,
    maxDurationSeconds: 10,
    supportsResolution: true,
  },
  imageToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputImages: 1,
    maxDurationSeconds: 5,
  },
  videoToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputVideos: 1,
    maxDurationSeconds: 5,
  },
}
```

Les champs agrégés plats comme `maxInputImages` et `maxInputVideos` ne suffisent
pas à annoncer la prise en charge des modes de transformation. Les fournisseurs devraient déclarer
explicitement `generate`, `imageToVideo` et `videoToVideo` afin que les tests live,
les tests de contrat et l’outil partagé `video_generate` puissent valider la prise en charge des modes
de manière déterministe.

## Tests live

Couverture live opt-in pour les fournisseurs intégrés partagés :

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts
```

Wrapper du dépôt :

```bash
pnpm test:live:media video
```

Ce fichier live charge les variables d’environnement fournisseur manquantes depuis `~/.profile`, privilégie
par défaut les clés API live/env avant les profils d’authentification stockés et exécute les
modes déclarés qu’il peut exercer sans risque avec des médias locaux :

- `generate` pour chaque fournisseur du balayage
- `imageToVideo` lorsque `capabilities.imageToVideo.enabled`
- `videoToVideo` lorsque `capabilities.videoToVideo.enabled` et que le fournisseur/modèle
  accepte une entrée vidéo locale adossée à un buffer dans le balayage partagé

Aujourd’hui, la voie live partagée `videoToVideo` couvre :

- `runway` uniquement lorsque vous sélectionnez `runway/gen4_aleph`

## Configuration

Définissez le modèle de génération vidéo par défaut dans votre config OpenClaw :

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

Ou via la CLI :

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "qwen/wan2.6-t2v"
```

## Lié

- [Vue d’ensemble des outils](/fr/tools)
- [Tâches en arrière-plan](/fr/automation/tasks) -- suivi des tâches pour la génération vidéo asynchrone
- [Alibaba Model Studio](/fr/providers/alibaba)
- [BytePlus](/fr/concepts/model-providers#byteplus-international)
- [ComfyUI](/fr/providers/comfy)
- [fal](/fr/providers/fal)
- [Google (Gemini)](/fr/providers/google)
- [MiniMax](/fr/providers/minimax)
- [OpenAI](/fr/providers/openai)
- [Qwen](/fr/providers/qwen)
- [Runway](/fr/providers/runway)
- [Together AI](/fr/providers/together)
- [Vydra](/fr/providers/vydra)
- [xAI](/fr/providers/xai)
- [Référence de configuration](/fr/gateway/configuration-reference#agent-defaults)
- [Modèles](/fr/concepts/models)
