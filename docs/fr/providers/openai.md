---
read_when:
    - Vous voulez utiliser des modèles OpenAI dans OpenClaw
    - Vous voulez une authentification par abonnement Codex au lieu de clés API
summary: Utiliser OpenAI via des clés API ou un abonnement Codex dans OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-07T06:54:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6a2ce1ce5f085fe55ec50b8d20359180b9002c9730820cd5b0e011c3bf807b64
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI fournit des API développeur pour les modèles GPT. Codex prend en charge **la connexion ChatGPT** pour un accès par abonnement
ou **la connexion par clé API** pour un accès facturé à l'usage. Codex cloud exige une connexion ChatGPT.
OpenAI prend explicitement en charge l'utilisation d'OAuth par abonnement dans des outils / flux de travail externes comme OpenClaw.

## Style d'interaction par défaut

OpenClaw peut ajouter une petite superposition de prompt spécifique à OpenAI pour les exécutions `openai/*` et
`openai-codex/*`. Par défaut, cette superposition garde l'assistant chaleureux,
collaboratif, concis, direct et un peu plus expressif sur le plan émotionnel,
sans remplacer le prompt système de base d'OpenClaw. La superposition conviviale
autorise aussi des emoji occasionnels lorsqu'ils s'intègrent naturellement, tout en gardant
la sortie globale concise.

Clé de configuration :

`plugins.entries.openai.config.personality`

Valeurs autorisées :

- `"friendly"` : valeur par défaut ; active la superposition spécifique à OpenAI.
- `"on"` : alias de `"friendly"`.
- `"off"` : désactive la superposition et utilise uniquement le prompt de base d'OpenClaw.

Portée :

- S'applique aux modèles `openai/*`.
- S'applique aux modèles `openai-codex/*`.
- N'affecte pas les autres fournisseurs.

Ce comportement est activé par défaut. Conservez explicitement `"friendly"` si vous voulez qu'il
survive à de futures modifications locales de configuration :

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "friendly",
        },
      },
    },
  },
}
```

### Désactiver la superposition de prompt OpenAI

Si vous voulez le prompt de base d'OpenClaw non modifié, définissez la superposition sur `"off"` :

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "off",
        },
      },
    },
  },
}
```

Vous pouvez aussi le définir directement avec la CLI de configuration :

```bash
openclaw config set plugins.entries.openai.config.personality off
```

OpenClaw normalise ce paramètre sans tenir compte de la casse à l'exécution, donc des valeurs comme
`"Off"` désactivent également la superposition conviviale.

## Option A : clé API OpenAI (OpenAI Platform)

**Idéal pour :** accès API direct et facturation à l'usage.
Récupérez votre clé API depuis le tableau de bord OpenAI.

Résumé des routes :

- `openai/gpt-5.4` = route API directe OpenAI Platform
- Nécessite `OPENAI_API_KEY` (ou une configuration équivalente du fournisseur OpenAI)
- Dans OpenClaw, la connexion ChatGPT/Codex passe par `openai-codex/*`, pas `openai/*`

### Configuration via CLI

```bash
openclaw onboard --auth-choice openai-api-key
# ou en non interactif
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Extrait de configuration

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

La documentation actuelle des modèles API d'OpenAI liste `gpt-5.4` et `gpt-5.4-pro` pour un usage API OpenAI direct. OpenClaw transmet les deux via le chemin `openai/*` Responses.
OpenClaw masque volontairement l'entrée obsolète `openai/gpt-5.3-codex-spark`,
car les appels API OpenAI directs la rejettent en trafic réel.

OpenClaw **n'expose pas** `openai/gpt-5.3-codex-spark` sur le chemin API OpenAI direct.
`pi-ai` livre toujours une entrée intégrée pour ce modèle, mais les requêtes API OpenAI réelles
le rejettent actuellement. Spark est traité comme Codex uniquement dans OpenClaw.

## Génération d'images

Le plugin groupé `openai` enregistre aussi la génération d'images via l'outil partagé
`image_generate`.

- Modèle d'image par défaut : `openai/gpt-image-1`
- Génération : jusqu'à 4 images par requête
- Mode édition : activé, jusqu'à 5 images de référence
- Prend en charge `size`
- Limitation actuelle spécifique à OpenAI : OpenClaw ne transmet pas aujourd'hui les surcharges `aspectRatio` ni
  `resolution` à l'API OpenAI Images

Pour utiliser OpenAI comme fournisseur d'images par défaut :

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

Consultez [Image Generation](/fr/tools/image-generation) pour les paramètres
de l'outil partagé, la sélection du fournisseur et le comportement de basculement.

## Génération de vidéo

Le plugin groupé `openai` enregistre aussi la génération de vidéo via l'outil partagé
`video_generate`.

- Modèle vidéo par défaut : `openai/sora-2`
- Modes : texte vers vidéo, image vers vidéo et flux de référence / édition à vidéo unique
- Limites actuelles : 1 image ou 1 vidéo en entrée de référence
- Limitation actuelle spécifique à OpenAI : OpenClaw ne transmet actuellement que les surcharges `size`
  pour la génération vidéo native OpenAI. Les surcharges facultatives non prises en charge
  telles que `aspectRatio`, `resolution`, `audio` et `watermark` sont ignorées
  et signalées sous forme d'avertissement d'outil.

Pour utiliser OpenAI comme fournisseur vidéo par défaut :

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openai/sora-2",
      },
    },
  },
}
```

Consultez [Video Generation](/fr/tools/video-generation) pour les paramètres
de l'outil partagé, la sélection du fournisseur et le comportement de basculement.

## Option B : abonnement OpenAI Code (Codex)

**Idéal pour :** utiliser l'accès par abonnement ChatGPT/Codex au lieu d'une clé API.
Codex cloud exige une connexion ChatGPT, tandis que la CLI Codex prend en charge une connexion ChatGPT ou par clé API.

Résumé des routes :

- `openai-codex/gpt-5.4` = route OAuth ChatGPT/Codex
- Utilise la connexion ChatGPT/Codex, pas une clé API directe OpenAI Platform
- Les limites côté fournisseur pour `openai-codex/*` peuvent différer de l'expérience ChatGPT web / application

### Configuration via CLI (OAuth Codex)

```bash
# Exécuter OAuth Codex dans l'assistant
openclaw onboard --auth-choice openai-codex

# Ou exécuter OAuth directement
openclaw models auth login --provider openai-codex
```

### Extrait de configuration (abonnement Codex)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

La documentation actuelle de Codex d'OpenAI liste `gpt-5.4` comme modèle Codex actuel. OpenClaw
le mappe vers `openai-codex/gpt-5.4` pour l'utilisation OAuth ChatGPT/Codex.

Cette route est volontairement séparée de `openai/gpt-5.4`. Si vous voulez le
chemin API direct OpenAI Platform, utilisez `openai/*` avec une clé API. Si vous voulez
la connexion ChatGPT/Codex, utilisez `openai-codex/*`.

Si l'onboarding réutilise une connexion Codex CLI existante, ces identifiants restent
gérés par Codex CLI. À l'expiration, OpenClaw relit d'abord la source Codex externe
et, lorsque le fournisseur peut les actualiser, réécrit l'identifiant actualisé
dans le stockage Codex au lieu d'en prendre possession dans une copie séparée réservée à OpenClaw.

Si votre compte Codex a droit à Codex Spark, OpenClaw prend aussi en charge :

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw traite Codex Spark comme réservé à Codex. Il n'expose pas de chemin direct
`openai/gpt-5.3-codex-spark` par clé API.

OpenClaw préserve aussi `openai-codex/gpt-5.3-codex-spark` lorsque `pi-ai`
le découvre. Considérez-le comme dépendant des droits et expérimental : Codex Spark est
distinct du `/fast` de GPT-5.4, et sa disponibilité dépend du compte Codex /
ChatGPT connecté.

### Limite de fenêtre de contexte Codex

OpenClaw traite les métadonnées du modèle Codex et la limite de contexte d'exécution comme des
valeurs distinctes.

Pour `openai-codex/gpt-5.4` :

- `contextWindow` natif : `1050000`
- limite par défaut de `contextTokens` à l'exécution : `272000`

Cela permet de garder des métadonnées de modèle fidèles tout en conservant la plus petite
fenêtre d'exécution par défaut qui offre en pratique de meilleures caractéristiques de latence et de qualité.

Si vous voulez une limite effective différente, définissez `models.providers.<provider>.models[].contextTokens` :

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

Utilisez `contextWindow` uniquement lorsque vous déclarez ou remplacez des métadonnées natives de modèle.
Utilisez `contextTokens` lorsque vous voulez limiter le budget de contexte à l'exécution.

### Transport par défaut

OpenClaw utilise `pi-ai` pour le streaming des modèles. Pour `openai/*` et
`openai-codex/*`, le transport par défaut est `"auto"` (WebSocket d'abord, puis repli SSE).

En mode `"auto"`, OpenClaw réessaie également un échec WebSocket précoce et réessayable
avant de se rabattre sur SSE. Le mode forcé `"websocket"` expose toujours
directement les erreurs de transport au lieu de les masquer derrière le repli.

Après un échec WebSocket à la connexion ou au début du tour en mode `"auto"`, OpenClaw marque
le chemin WebSocket de cette session comme dégradé pendant environ 60 secondes et envoie
les tours suivants via SSE pendant ce délai au lieu d'alterner inutilement entre
les transports.

Pour les points de terminaison natifs de la famille OpenAI (`openai/*`, `openai-codex/*` et Azure
OpenAI Responses), OpenClaw attache aussi un état stable d'identité de session et de tour
aux requêtes afin que les nouvelles tentatives, reconnexions et le repli SSE restent alignés sur la même
identité de conversation. Sur les routes natives de la famille OpenAI, cela inclut des en-têtes stables
d'identité de session / de tour ainsi que des métadonnées de transport correspondantes.

OpenClaw normalise aussi les compteurs d'usage OpenAI entre variantes de transport avant
qu'ils n'atteignent les surfaces de session / statut. Le trafic natif OpenAI / Codex Responses peut
signaler l'usage soit sous `input_tokens` / `output_tokens`, soit sous
`prompt_tokens` / `completion_tokens` ; OpenClaw les traite comme les mêmes compteurs d'entrée
et de sortie pour `/status`, `/usage` et les journaux de session. Lorsque le trafic WebSocket natif
omet `total_tokens` (ou signale `0`), OpenClaw se rabat sur le total normalisé entrée + sortie afin que les affichages de session / statut restent renseignés.

Vous pouvez définir `agents.defaults.models.<provider/model>.params.transport` :

- `"sse"` : forcer SSE
- `"websocket"` : forcer WebSocket
- `"auto"` : essayer WebSocket, puis se rabattre sur SSE

Pour `openai/*` (API Responses), OpenClaw active aussi par défaut le préchauffage WebSocket
(`openaiWsWarmup: true`) lorsque le transport WebSocket est utilisé.

Documentation OpenAI connexe :

- [API Realtime avec WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming des réponses API (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### Préchauffage WebSocket OpenAI

La documentation OpenAI décrit le préchauffage comme facultatif. OpenClaw l'active par défaut pour
`openai/*` afin de réduire la latence du premier tour lors de l'utilisation du transport WebSocket.

### Désactiver le préchauffage

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### Activer explicitement le préchauffage

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### Traitement prioritaire OpenAI et Codex

L'API d'OpenAI expose le traitement prioritaire via `service_tier=priority`. Dans
OpenClaw, définissez `agents.defaults.models["<provider>/<model>"].params.serviceTier`
pour transmettre ce champ aux points de terminaison natifs OpenAI / Codex Responses.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Les valeurs prises en charge sont `auto`, `default`, `flex` et `priority`.

OpenClaw transmet `params.serviceTier` à la fois aux requêtes directes `openai/*` Responses
et aux requêtes `openai-codex/*` Codex Responses lorsque ces modèles pointent
vers les points de terminaison natifs OpenAI / Codex.

Comportement important :

- `openai/*` direct doit cibler `api.openai.com`
- `openai-codex/*` doit cibler `chatgpt.com/backend-api`
- si vous faites passer l'un ou l'autre fournisseur par une autre URL de base ou un proxy, OpenClaw laisse `service_tier` inchangé

### Mode rapide OpenAI

OpenClaw expose un basculement partagé de mode rapide pour les sessions `openai/*` et
`openai-codex/*` :

- Discussion / UI : `/fast status|on|off`
- Configuration : `agents.defaults.models["<provider>/<model>"].params.fastMode`

Lorsque le mode rapide est activé, OpenClaw le mappe au traitement prioritaire OpenAI :

- les appels directs `openai/*` Responses vers `api.openai.com` envoient `service_tier = "priority"`
- les appels `openai-codex/*` Responses vers `chatgpt.com/backend-api` envoient aussi `service_tier = "priority"`
- les valeurs `service_tier` déjà présentes dans la charge utile sont préservées
- le mode rapide ne réécrit pas `reasoning` ni `text.verbosity`

Pour GPT 5.4 en particulier, la configuration la plus courante est :

- envoyez `/fast on` dans une session utilisant `openai/gpt-5.4` ou `openai-codex/gpt-5.4`
- ou définissez `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true`
- si vous utilisez aussi OAuth Codex, définissez également `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true`

Exemple :

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

Les surcharges de session priment sur la configuration. Effacer la surcharge de session dans l'UI Sessions
ramène la session à la valeur par défaut configurée.

### Routes natives OpenAI versus routes compatibles OpenAI

OpenClaw traite les points de terminaison directs OpenAI, Codex et Azure OpenAI différemment
des proxys génériques compatibles OpenAI `/v1` :

- les routes natives `openai/*`, `openai-codex/*` et Azure OpenAI conservent
  `reasoning: { effort: "none" }` intact lorsque vous désactivez explicitement le raisonnement
- les routes natives de la famille OpenAI mettent par défaut les schémas d'outil en mode strict
- les en-têtes cachés d'attribution OpenClaw (`originator`, `version` et
  `User-Agent`) ne sont attachés que sur les hôtes natifs OpenAI vérifiés
  (`api.openai.com`) et les hôtes natifs Codex (`chatgpt.com/backend-api`)
- les routes natives OpenAI / Codex conservent la mise en forme de requête propre à OpenAI telle que
  `service_tier`, `store` pour Responses, les charges utiles de compatibilité de raisonnement OpenAI et
  les indications de cache de prompt
- les routes de style proxy compatibles OpenAI conservent le comportement de compatibilité plus souple et
  ne forcent pas les schémas d'outil stricts, la mise en forme de requête réservée au natif ni les
  en-têtes cachés d'attribution OpenAI / Codex

Azure OpenAI reste dans la catégorie de routage natif pour le transport et le comportement de compatibilité, mais il ne reçoit pas les en-têtes cachés d'attribution OpenAI / Codex.

Cela préserve le comportement actuel natif OpenAI Responses sans imposer d'anciens
shims compatibles OpenAI à des backends tiers `/v1`.

### Compactage côté serveur OpenAI Responses

Pour les modèles directs OpenAI Responses (`openai/*` utilisant `api: "openai-responses"` avec
`baseUrl` sur `api.openai.com`), OpenClaw active désormais automatiquement les indications de charge utile de compactage côté serveur OpenAI :

- Force `store: true` (sauf si la compatibilité du modèle définit `supportsStore: false`)
- Injecte `context_management: [{ type: "compaction", compact_threshold: ... }]`

Par défaut, `compact_threshold` vaut `70%` de `contextWindow` du modèle (ou `80000`
lorsqu'il n'est pas disponible).

### Activer explicitement le compactage côté serveur

Utilisez ceci lorsque vous voulez forcer l'injection de `context_management` sur des
modèles Responses compatibles (par exemple Azure OpenAI Responses) :

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### Activer avec un seuil personnalisé

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### Désactiver le compactage côté serveur

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction` ne contrôle que l'injection de `context_management`.
Les modèles directs OpenAI Responses forcent toujours `store: true` sauf si la compatibilité définit
`supportsStore: false`.

## Remarques

- Les références de modèle utilisent toujours `provider/model` (voir [/concepts/models](/fr/concepts/models)).
- Les détails d'authentification + les règles de réutilisation se trouvent dans [/concepts/oauth](/fr/concepts/oauth).
