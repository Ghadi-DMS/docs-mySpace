---
read_when:
    - Générer de la musique ou de l'audio via l'agent
    - Configurer les fournisseurs et modèles de génération musicale
    - Comprendre les paramètres de l'outil music_generate
summary: Générez de la musique avec des fournisseurs partagés, y compris des plugins adossés à des workflows
title: Génération musicale
x-i18n:
    generated_at: "2026-04-07T06:55:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: ce8da8dfc188efe8593ca5cbec0927dd1d18d2861a1a828df89c8541ccf1cb25
    source_path: tools/music-generation.md
    workflow: 15
---

# Génération musicale

L'outil `music_generate` permet à l'agent de créer de la musique ou de l'audio via la
capacité partagée de génération musicale avec des fournisseurs configurés tels que Google,
MiniMax et ComfyUI configuré par workflow.

Pour les sessions d'agent adossées à des fournisseurs partagés, OpenClaw lance la génération musicale en tant que
tâche d'arrière-plan, la suit dans le registre des tâches, puis réveille à nouveau l'agent lorsque
la piste est prête afin que l'agent puisse republier l'audio terminé dans le
canal d'origine.

<Note>
L'outil partagé intégré n'apparaît que lorsqu'au moins un fournisseur de génération musicale est disponible. Si vous ne voyez pas `music_generate` dans les outils de votre agent, configurez `agents.defaults.musicGenerationModel` ou définissez une clé API de fournisseur.
</Note>

## Démarrage rapide

### Génération adossée à un fournisseur partagé

1. Définissez une clé API pour au moins un fournisseur, par exemple `GEMINI_API_KEY` ou
   `MINIMAX_API_KEY`.
2. Définissez éventuellement votre modèle préféré :

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

3. Demandez à l'agent : _« Génère une piste synthpop entraînante sur une virée de nuit
   dans une ville néon. »_

L'agent appelle automatiquement `music_generate`. Aucune liste d'autorisation d'outil n'est nécessaire.

Pour les contextes synchrones directs sans exécution d'agent adossée à une session, l'outil intégré
revient toujours à une génération inline et renvoie le chemin final du média dans
le résultat de l'outil.

Exemples de prompts :

```text
Generate a cinematic piano track with soft strings and no vocals.
```

```text
Generate an energetic chiptune loop about launching a rocket at sunrise.
```

### Génération Comfy pilotée par workflow

Le plugin `comfy` intégré se branche sur l'outil partagé `music_generate` via
le registre de fournisseurs de génération musicale.

1. Configurez `models.providers.comfy.music` avec un JSON de workflow et des
   nœuds de prompt/sortie.
2. Si vous utilisez Comfy Cloud, définissez `COMFY_API_KEY` ou `COMFY_CLOUD_API_KEY`.
3. Demandez de la musique à l'agent ou appelez directement l'outil.

Exemple :

```text
/tool music_generate prompt="Warm ambient synth loop with soft tape texture"
```

## Prise en charge intégrée des fournisseurs partagés

| Fournisseur | Modèle par défaut      | Entrées de référence | Contrôles pris en charge                                 | Clé API                                |
| ----------- | ---------------------- | -------------------- | -------------------------------------------------------- | -------------------------------------- |
| ComfyUI     | `workflow`             | Jusqu'à 1 image      | Musique ou audio définis par le workflow                 | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| Google      | `lyria-3-clip-preview` | Jusqu'à 10 images    | `lyrics`, `instrumental`, `format`                       | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax     | `music-2.5+`           | Aucune               | `lyrics`, `instrumental`, `durationSeconds`, `format=mp3` | `MINIMAX_API_KEY`                      |

### Matrice de capacités déclarées

Il s'agit du contrat de mode explicite utilisé par `music_generate`, les tests de contrat
et le balayage live partagé.

| Fournisseur | `generate` | `edit` | Limite d'édition | Pistes live partagées                                                    |
| ----------- | ---------- | ------ | ---------------- | ------------------------------------------------------------------------ |
| ComfyUI     | Oui        | Oui    | 1 image          | Pas dans le balayage partagé ; couvert par `extensions/comfy/comfy.live.test.ts` |
| Google      | Oui        | Oui    | 10 images        | `generate`, `edit`                                                       |
| MiniMax     | Oui        | Non    | Aucune           | `generate`                                                               |

Utilisez `action: "list"` pour inspecter les fournisseurs et modèles partagés disponibles à
l'exécution :

```text
/tool music_generate action=list
```

Utilisez `action: "status"` pour inspecter la tâche musicale active adossée à la session :

```text
/tool music_generate action=status
```

Exemple de génération directe :

```text
/tool music_generate prompt="Dreamy lo-fi hip hop with vinyl texture and gentle rain" instrumental=true
```

## Paramètres de l'outil intégré

| Paramètre         | Type     | Description                                                                                           |
| ----------------- | -------- | ----------------------------------------------------------------------------------------------------- |
| `prompt`          | string   | Prompt de génération musicale (obligatoire pour `action: "generate"`)                                 |
| `action`          | string   | `"generate"` (par défaut), `"status"` pour la tâche de la session actuelle, ou `"list"` pour inspecter les fournisseurs |
| `model`           | string   | Remplacement fournisseur/modèle, par ex. `google/lyria-3-pro-preview` ou `comfy/workflow`            |
| `lyrics`          | string   | Paroles facultatives lorsque le fournisseur prend en charge une entrée explicite de paroles           |
| `instrumental`    | boolean  | Demande une sortie instrumentale uniquement lorsque le fournisseur la prend en charge                 |
| `image`           | string   | Chemin ou URL d'une image de référence unique                                                         |
| `images`          | string[] | Plusieurs images de référence (jusqu'à 10)                                                            |
| `durationSeconds` | number   | Durée cible en secondes lorsque le fournisseur prend en charge les indications de durée               |
| `format`          | string   | Indication de format de sortie (`mp3` ou `wav`) lorsque le fournisseur la prend en charge            |
| `filename`        | string   | Indication de nom de fichier                                                                          |

Tous les fournisseurs ne prennent pas en charge tous les paramètres. OpenClaw valide tout de même les limites strictes
comme le nombre d'entrées avant soumission. Lorsqu'un fournisseur prend en charge la durée mais
utilise une durée maximale plus courte que la valeur demandée, OpenClaw la limite automatiquement
à la durée prise en charge la plus proche. Les indications facultatives réellement non prises en charge sont ignorées
avec un avertissement lorsque le fournisseur ou le modèle sélectionné ne peut pas les respecter.

Les résultats de l'outil indiquent les paramètres appliqués. Lorsque OpenClaw limite la durée lors d'un repli de fournisseur, le `durationSeconds` renvoyé reflète la valeur soumise et `details.normalization.durationSeconds` montre la correspondance entre la valeur demandée et la valeur appliquée.

## Comportement asynchrone pour le chemin adossé au fournisseur partagé

- Exécutions d'agent adossées à une session : `music_generate` crée une tâche d'arrière-plan, renvoie immédiatement une réponse de type démarrée/tâche, puis publie la piste terminée plus tard dans un message de suivi de l'agent.
- Prévention des doublons : tant que cette tâche d'arrière-plan est encore `queued` ou `running`, les appels ultérieurs à `music_generate` dans la même session renvoient l'état de la tâche au lieu de lancer une autre génération.
- Recherche d'état : utilisez `action: "status"` pour inspecter la tâche musicale active adossée à la session sans en lancer une nouvelle.
- Suivi des tâches : utilisez `openclaw tasks list` ou `openclaw tasks show <taskId>` pour inspecter les états en file d'attente, en cours et terminaux de la génération.
- Réveil à la fin : OpenClaw injecte un événement interne de fin dans la même session afin que le modèle puisse lui-même rédiger le message de suivi visible par l'utilisateur.
- Indication de prompt : les tours utilisateur/manuels ultérieurs dans la même session reçoivent une petite indication d'exécution lorsqu'une tâche musicale est déjà en cours afin que le modèle n'appelle pas aveuglément `music_generate` à nouveau.
- Repli sans session : les contextes directs/locaux sans véritable session d'agent s'exécutent toujours inline et renvoient le résultat audio final dans le même tour.

### Cycle de vie de la tâche

Chaque requête `music_generate` passe par quatre états :

1. **queued** -- tâche créée, en attente que le fournisseur l'accepte.
2. **running** -- le fournisseur traite la demande (généralement de 30 secondes à 3 minutes selon le fournisseur et la durée).
3. **succeeded** -- la piste est prête ; l'agent se réveille et la publie dans la conversation.
4. **failed** -- erreur ou délai d'attente du fournisseur ; l'agent se réveille avec les détails de l'erreur.

Vérifiez l'état depuis la CLI :

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

Prévention des doublons : si une tâche musicale est déjà `queued` ou `running` pour la session actuelle, `music_generate` renvoie l'état de la tâche existante au lieu d'en démarrer une nouvelle. Utilisez `action: "status"` pour vérifier explicitement sans déclencher une nouvelle génération.

## Configuration

### Sélection du modèle

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["minimax/music-2.5+"],
      },
    },
  },
}
```

### Ordre de sélection des fournisseurs

Lors de la génération de musique, OpenClaw essaie les fournisseurs dans cet ordre :

1. Le paramètre `model` de l'appel d'outil, si l'agent en spécifie un
2. `musicGenerationModel.primary` depuis la configuration
3. `musicGenerationModel.fallbacks` dans l'ordre
4. Détection automatique à l'aide des seules valeurs par défaut de fournisseur adossées à l'authentification :
   - le fournisseur par défaut actuel d'abord
   - les autres fournisseurs de génération musicale enregistrés restants dans l'ordre des ID de fournisseur

Si un fournisseur échoue, le candidat suivant est essayé automatiquement. Si tous échouent, l'erreur
inclut les détails de chaque tentative.

Définissez `agents.defaults.mediaGenerationAutoProviderFallback: false` si vous voulez que
la génération musicale n'utilise que les entrées explicites `model`, `primary` et `fallbacks`.

## Notes sur les fournisseurs

- Google utilise la génération par lots Lyria 3. Le flux intégré actuel prend en charge
  le prompt, le texte de paroles facultatif et des images de référence facultatives.
- MiniMax utilise le point de terminaison par lots `music_generation`. Le flux intégré actuel
  prend en charge le prompt, les paroles facultatives, le mode instrumental, l'orientation de durée et
  la sortie mp3.
- La prise en charge de ComfyUI est pilotée par workflow et dépend du graphe configuré ainsi que du
  mappage de nœuds pour les champs de prompt/sortie.

## Modes de capacité des fournisseurs

Le contrat partagé de génération musicale prend désormais en charge des déclarations de mode explicites :

- `generate` pour la génération à partir du seul prompt
- `edit` lorsque la requête inclut une ou plusieurs images de référence

Les nouvelles implémentations de fournisseurs doivent privilégier des blocs de mode explicites :

```typescript
capabilities: {
  generate: {
    maxTracks: 1,
    supportsLyrics: true,
    supportsFormat: true,
  },
  edit: {
    enabled: true,
    maxTracks: 1,
    maxInputImages: 1,
    supportsFormat: true,
  },
}
```

Les champs plats hérités tels que `maxInputImages`, `supportsLyrics` et
`supportsFormat` ne suffisent pas à annoncer la prise en charge du mode édition. Les fournisseurs doivent
déclarer explicitement `generate` et `edit` afin que les tests live, les tests de contrat et
l'outil partagé `music_generate` puissent valider la prise en charge des modes de manière déterministe.

## Choisir le bon chemin

- Utilisez le chemin partagé adossé à un fournisseur lorsque vous voulez la sélection de modèle, le repli entre fournisseurs et le flux intégré asynchrone de tâche/état.
- Utilisez un chemin de plugin tel que ComfyUI lorsque vous avez besoin d'un graphe de workflow personnalisé ou d'un fournisseur qui ne fait pas partie de la capacité musicale intégrée partagée.
- Si vous déboguez un comportement spécifique à ComfyUI, voir [ComfyUI](/fr/providers/comfy). Si vous déboguez le comportement d'un fournisseur partagé, commencez par [Google (Gemini)](/fr/providers/google) ou [MiniMax](/fr/providers/minimax).

## Tests live

Couverture live opt-in pour les fournisseurs intégrés partagés :

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

Wrapper du dépôt :

```bash
pnpm test:live:media music
```

Ce fichier live charge les variables d'environnement de fournisseur manquantes depuis `~/.profile`, privilégie
par défaut les clés API live/env par rapport aux profils d'authentification stockés, et exécute à la fois
la couverture `generate` et la couverture `edit` déclarée lorsque le fournisseur active le mode édition.

Aujourd'hui, cela signifie :

- `google` : `generate` plus `edit`
- `minimax` : `generate` uniquement
- `comfy` : couverture live Comfy séparée, hors balayage des fournisseurs partagés

Couverture live opt-in pour le chemin musical ComfyUI intégré :

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

Le fichier live Comfy couvre aussi les workflows d'image et de vidéo comfy lorsque ces
sections sont configurées.

## Liens associés

- [Tâches d'arrière-plan](/fr/automation/tasks) - suivi des tâches pour les exécutions `music_generate` détachées
- [Référence de configuration](/fr/gateway/configuration-reference#agent-defaults) - configuration `musicGenerationModel`
- [ComfyUI](/fr/providers/comfy)
- [Google (Gemini)](/fr/providers/google)
- [MiniMax](/fr/providers/minimax)
- [Modèles](/fr/concepts/models) - configuration des modèles et repli
- [Vue d'ensemble des outils](/fr/tools)
