---
read_when:
    - Vous souhaitez utiliser des modèles OpenAI dans OpenClaw
    - Vous souhaitez utiliser l’authentification par abonnement Codex au lieu de clés API
    - Vous avez besoin d’un comportement d’exécution des agents GPT-5 plus strict
summary: Utilisez OpenAI via des clés API ou un abonnement Codex dans OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-12T00:18:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 7aa06fba9ac901e663685a6b26443a2f6aeb6ec3589d939522dc87cbb43497b4
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI fournit des API développeur pour les modèles GPT. Codex prend en charge la **connexion ChatGPT** pour un accès par abonnement
ou la connexion par **clé API** pour un accès facturé à l’usage. Codex cloud nécessite une connexion ChatGPT.
OpenAI prend explicitement en charge l’utilisation d’OAuth d’abonnement dans des outils/workflows externes comme OpenClaw.

## Style d’interaction par défaut

OpenClaw peut ajouter une petite surcouche d’invite spécifique à OpenAI pour les exécutions `openai/*` et
`openai-codex/*`. Par défaut, la surcouche garde l’assistant chaleureux,
collaboratif, concis, direct et un peu plus expressif sur le plan émotionnel
sans remplacer l’invite système de base d’OpenClaw. La surcouche conviviale
autorise aussi l’emoji occasionnel lorsqu’il s’intègre naturellement, tout en gardant
une sortie globale concise.

Clé de configuration :

`plugins.entries.openai.config.personality`

Valeurs autorisées :

- `"friendly"` : valeur par défaut ; active la surcouche spécifique à OpenAI.
- `"on"` : alias de `"friendly"`.
- `"off"` : désactive la surcouche et utilise uniquement l’invite de base d’OpenClaw.

Portée :

- S’applique aux modèles `openai/*`.
- S’applique aux modèles `openai-codex/*`.
- N’affecte pas les autres fournisseurs.

Ce comportement est activé par défaut. Conservez explicitement `"friendly"` si vous voulez que
cela survive à de futurs changements de configuration locale :

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

### Désactiver la surcouche d’invite OpenAI

Si vous voulez l’invite de base OpenClaw non modifiée, définissez la surcouche sur `"off"` :

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

Vous pouvez aussi la définir directement avec la CLI de configuration :

```bash
openclaw config set plugins.entries.openai.config.personality off
```

OpenClaw normalise ce paramètre sans tenir compte de la casse à l’exécution, donc des valeurs comme
`"Off"` désactivent quand même la surcouche conviviale.

## Option A : clé API OpenAI (OpenAI Platform)

**Idéal pour :** l’accès direct à l’API et la facturation à l’usage.
Obtenez votre clé API depuis le tableau de bord OpenAI.

Résumé du routage :

- `openai/gpt-5.4` = route API directe OpenAI Platform
- Nécessite `OPENAI_API_KEY` (ou une configuration équivalente du fournisseur OpenAI)
- Dans OpenClaw, la connexion ChatGPT/Codex passe par `openai-codex/*`, pas par `openai/*`

### Configuration CLI

```bash
openclaw onboard --auth-choice openai-api-key
# ou en mode non interactif
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Extrait de configuration

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

La documentation actuelle des modèles API d’OpenAI liste `gpt-5.4` et `gpt-5.4-pro` pour une utilisation directe
de l’API OpenAI. OpenClaw transmet les deux via le chemin Responses `openai/*`.
OpenClaw masque intentionnellement l’entrée obsolète `openai/gpt-5.3-codex-spark`,
car les appels directs à l’API OpenAI la rejettent dans le trafic réel.

OpenClaw **n’expose pas** `openai/gpt-5.3-codex-spark` sur le chemin direct de l’API OpenAI.
`pi-ai` inclut toujours une entrée intégrée pour ce modèle, mais les requêtes réelles à l’API OpenAI
la rejettent actuellement. Spark est traité comme réservé à Codex dans OpenClaw.

## Génération d’images

Le plugin `openai` inclus enregistre aussi la génération d’images via l’outil partagé
`image_generate`.

- Modèle d’image par défaut : `openai/gpt-image-1`
- Génération : jusqu’à 4 images par requête
- Mode édition : activé, jusqu’à 5 images de référence
- Prend en charge `size`
- Limitation spécifique actuelle à OpenAI : OpenClaw ne transmet pas aujourd’hui les remplacements `aspectRatio` ou
  `resolution` à l’API OpenAI Images

Pour utiliser OpenAI comme fournisseur d’images par défaut :

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

Voir [Image Generation](/fr/tools/image-generation) pour les paramètres
de l’outil partagé, la sélection du fournisseur et le comportement de basculement.

## Génération de vidéos

Le plugin `openai` inclus enregistre aussi la génération de vidéos via l’outil partagé
`video_generate`.

- Modèle vidéo par défaut : `openai/sora-2`
- Modes : texte vers vidéo, image vers vidéo, et flux de référence/édition à vidéo unique
- Limites actuelles : 1 image ou 1 vidéo en entrée de référence
- Limitation spécifique actuelle à OpenAI : OpenClaw ne transmet actuellement que les remplacements
  `size` pour la génération vidéo native OpenAI. Les remplacements optionnels non pris en charge
  comme `aspectRatio`, `resolution`, `audio` et `watermark` sont ignorés
  et renvoyés sous forme d’avertissement d’outil.

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

Voir [Video Generation](/fr/tools/video-generation) pour les paramètres
de l’outil partagé, la sélection du fournisseur et le comportement de basculement.

## Option B : abonnement OpenAI Code (Codex)

**Idéal pour :** utiliser l’accès par abonnement ChatGPT/Codex au lieu d’une clé API.
Codex cloud nécessite une connexion ChatGPT, tandis que la CLI Codex prend en charge la connexion par ChatGPT ou par clé API.

Résumé du routage :

- `openai-codex/gpt-5.4` = route OAuth ChatGPT/Codex
- Utilise la connexion ChatGPT/Codex, pas une clé API directe OpenAI Platform
- Les limites côté fournisseur pour `openai-codex/*` peuvent différer de l’expérience web/app ChatGPT

### Configuration CLI (OAuth Codex)

```bash
# Exécuter OAuth Codex dans l’assistant
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

La documentation actuelle de Codex d’OpenAI liste `gpt-5.4` comme modèle Codex actuel. OpenClaw
l’associe à `openai-codex/gpt-5.4` pour l’utilisation avec OAuth ChatGPT/Codex.

Cette route est volontairement distincte de `openai/gpt-5.4`. Si vous voulez le
chemin direct de l’API OpenAI Platform, utilisez `openai/*` avec une clé API. Si vous voulez
la connexion ChatGPT/Codex, utilisez `openai-codex/*`.

Si l’onboarding réutilise une connexion existante de la CLI Codex, ces identifiants restent
gérés par la CLI Codex. À l’expiration, OpenClaw relit d’abord la source Codex externe
et, lorsque le fournisseur peut l’actualiser, écrit l’identifiant actualisé
dans le stockage Codex au lieu d’en prendre le contrôle dans une copie séparée propre à OpenClaw.

Si votre compte Codex donne droit à Codex Spark, OpenClaw prend aussi en charge :

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw traite Codex Spark comme réservé à Codex. Il n’expose pas de chemin direct
`openai/gpt-5.3-codex-spark` avec clé API.

OpenClaw conserve aussi `openai-codex/gpt-5.3-codex-spark` lorsque `pi-ai`
le détecte. Considérez-le comme dépendant des droits et expérimental : Codex Spark est
distinct de GPT-5.4 `/fast`, et sa disponibilité dépend du compte Codex /
ChatGPT connecté.

### Limite de fenêtre de contexte Codex

OpenClaw traite les métadonnées du modèle Codex et la limite de contexte à l’exécution comme des
valeurs distinctes.

Pour `openai-codex/gpt-5.4` :

- `contextWindow` natif : `1050000`
- plafond `contextTokens` par défaut à l’exécution : `272000`

Cela permet de garder des métadonnées de modèle fidèles tout en conservant la
fenêtre d’exécution par défaut plus petite, qui présente en pratique de meilleures caractéristiques
de latence et de qualité.

Si vous voulez un plafond effectif différent, définissez `models.providers.<provider>.models[].contextTokens` :

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

Utilisez `contextWindow` uniquement lorsque vous déclarez ou remplacez des métadonnées natives
du modèle. Utilisez `contextTokens` lorsque vous voulez limiter le budget de contexte à l’exécution.

### Transport par défaut

OpenClaw utilise `pi-ai` pour le streaming des modèles. Pour `openai/*` comme pour
`openai-codex/*`, le transport par défaut est `"auto"` (WebSocket d’abord, puis repli sur SSE).

En mode `"auto"`, OpenClaw réessaie aussi une défaillance WebSocket précoce et récupérable
avant de basculer sur SSE. Le mode forcé `"websocket"` continue d’exposer directement les erreurs de transport
au lieu de les masquer derrière le repli.

Après un échec WebSocket à la connexion ou au début d’un tour en mode `"auto"`, OpenClaw marque
le chemin WebSocket de cette session comme dégradé pendant environ 60 secondes et envoie
les tours suivants via SSE pendant le temps de refroidissement au lieu d’osciller entre
les transports.

Pour les endpoints natifs de la famille OpenAI (`openai/*`, `openai-codex/*` et Azure
OpenAI Responses), OpenClaw attache aussi un état stable d’identité de session et de tour
aux requêtes afin que les tentatives, reconnexions et replis SSE restent alignés sur la même
identité de conversation. Sur les routes natives de la famille OpenAI, cela inclut des en-têtes
stables d’identité de requête session/tour ainsi que des métadonnées de transport correspondantes.

OpenClaw normalise aussi les compteurs d’usage OpenAI entre les variantes de transport avant
qu’ils n’atteignent les surfaces de session/statut. Le trafic natif OpenAI/Codex Responses peut
signaler l’usage soit comme `input_tokens` / `output_tokens`, soit comme
`prompt_tokens` / `completion_tokens` ; OpenClaw les traite comme les mêmes compteurs d’entrée
et de sortie pour `/status`, `/usage` et les journaux de session. Lorsque le trafic WebSocket natif
omet `total_tokens` (ou signale `0`), OpenClaw revient au total normalisé entrée + sortie
afin que les affichages de session/statut restent renseignés.

Vous pouvez définir `agents.defaults.models.<provider/model>.params.transport` :

- `"sse"` : forcer SSE
- `"websocket"` : forcer WebSocket
- `"auto"` : essayer WebSocket, puis basculer sur SSE

Pour `openai/*` (API Responses), OpenClaw active aussi l’échauffement WebSocket par défaut
(`openaiWsWarmup: true`) lorsque le transport WebSocket est utilisé.

Documentation OpenAI associée :

- [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

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

### Échauffement WebSocket OpenAI

La documentation OpenAI décrit l’échauffement comme facultatif. OpenClaw l’active par défaut pour
`openai/*` afin de réduire la latence du premier tour lors de l’utilisation du transport WebSocket.

### Désactiver l’échauffement

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

### Activer explicitement l’échauffement

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

L’API OpenAI expose le traitement prioritaire via `service_tier=priority`. Dans
OpenClaw, définissez `agents.defaults.models["<provider>/<model>"].params.serviceTier`
pour transmettre ce champ aux endpoints natifs OpenAI/Codex Responses.

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

OpenClaw transmet `params.serviceTier` aux requêtes Responses directes `openai/*`
et aux requêtes Codex Responses `openai-codex/*` lorsque ces modèles pointent
vers les endpoints natifs OpenAI/Codex.

Comportement important :

- `openai/*` direct doit cibler `api.openai.com`
- `openai-codex/*` doit cibler `chatgpt.com/backend-api`
- si vous routez l’un ou l’autre fournisseur via une autre URL de base ou un proxy, OpenClaw laisse `service_tier` inchangé

### Mode rapide OpenAI

OpenClaw expose un basculement partagé de mode rapide pour les sessions `openai/*` et
`openai-codex/*` :

- Chat/UI : `/fast status|on|off`
- Configuration : `agents.defaults.models["<provider>/<model>"].params.fastMode`

Lorsque le mode rapide est activé, OpenClaw l’associe au traitement prioritaire OpenAI :

- les appels directs `openai/*` Responses vers `api.openai.com` envoient `service_tier = "priority"`
- les appels `openai-codex/*` Responses vers `chatgpt.com/backend-api` envoient aussi `service_tier = "priority"`
- les valeurs `service_tier` déjà présentes dans la charge utile sont conservées
- le mode rapide ne réécrit pas `reasoning` ni `text.verbosity`

Pour GPT 5.4 en particulier, la configuration la plus courante est la suivante :

- envoyer `/fast on` dans une session utilisant `openai/gpt-5.4` ou `openai-codex/gpt-5.4`
- ou définir `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true`
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

Les remplacements au niveau de la session priment sur la configuration. Effacer le remplacement de session dans l’interface Sessions
rétablit la valeur par défaut configurée pour la session.

### Routes natives OpenAI versus routes compatibles OpenAI

OpenClaw traite différemment les endpoints directs OpenAI, Codex et Azure OpenAI
par rapport aux proxys génériques compatibles OpenAI `/v1` :

- les routes natives `openai/*`, `openai-codex/*` et Azure OpenAI conservent
  `reasoning: { effort: "none" }` intact lorsque vous désactivez explicitement le raisonnement
- les routes natives de la famille OpenAI définissent par défaut les schémas d’outils en mode strict
- les en-têtes d’attribution OpenClaw cachés (`originator`, `version` et
  `User-Agent`) ne sont ajoutés que sur les hôtes natifs OpenAI vérifiés
  (`api.openai.com`) et les hôtes natifs Codex (`chatgpt.com/backend-api`)
- les routes natives OpenAI/Codex conservent les ajustements de requête propres à OpenAI comme
  `service_tier`, `store` de Responses, les charges utiles de compatibilité de raisonnement OpenAI, et
  les indications de cache d’invite
- les routes compatibles OpenAI de type proxy conservent le comportement de compatibilité plus souple et
  ne forcent pas les schémas d’outils stricts, les ajustements de requête réservés au natif, ni les en-têtes cachés
  d’attribution OpenAI/Codex

Azure OpenAI reste dans le groupe de routage natif pour le comportement de transport et de compatibilité,
mais il ne reçoit pas les en-têtes cachés d’attribution OpenAI/Codex.

Cela préserve le comportement actuel natif d’OpenAI Responses sans imposer d’anciens
adaptateurs compatibles OpenAI à des backends `/v1` tiers.

### Mode GPT agentique strict

Pour les exécutions de la famille GPT-5 sur `openai/*` et `openai-codex/*`, OpenClaw peut utiliser un
contrat d’exécution Pi intégré plus strict :

```json5
{
  agents: {
    defaults: {
      embeddedPi: {
        executionContract: "strict-agentic",
      },
    },
  },
}
```

Avec `strict-agentic`, OpenClaw ne traite plus un tour d’assistant composé uniquement d’un plan comme
une progression réussie lorsqu’une action d’outil concrète est disponible. Il réessaie le
tour avec une directive d’action immédiate, active automatiquement l’outil structuré `update_plan` pour
les travaux importants, et affiche un état de blocage explicite si le modèle continue
à planifier sans agir.

Ce mode est limité aux exécutions de la famille GPT-5 d’OpenAI et OpenAI Codex. Les autres fournisseurs
et les anciennes familles de modèles conservent le comportement Pi intégré par défaut, sauf si vous les faites
opter pour d’autres paramètres d’exécution.

### Compaction côté serveur d’OpenAI Responses

Pour les modèles directs OpenAI Responses (`openai/*` utilisant `api: "openai-responses"` avec
`baseUrl` sur `api.openai.com`), OpenClaw active désormais automatiquement les indications de charge utile
de compaction côté serveur d’OpenAI :

- Force `store: true` (sauf si la compatibilité du modèle définit `supportsStore: false`)
- Injecte `context_management: [{ type: "compaction", compact_threshold: ... }]`

Par défaut, `compact_threshold` vaut `70%` du `contextWindow` du modèle (ou `80000`
s’il n’est pas disponible).

### Activer explicitement la compaction côté serveur

Utilisez ceci si vous voulez forcer l’injection de `context_management` sur des
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

### Désactiver la compaction côté serveur

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

`responsesServerCompaction` contrôle uniquement l’injection de `context_management`.
Les modèles directs OpenAI Responses continuent à forcer `store: true` sauf si la compatibilité définit
`supportsStore: false`.

## Remarques

- Les références de modèle utilisent toujours `provider/model` (voir [/concepts/models](/fr/concepts/models)).
- Les détails d’authentification et les règles de réutilisation se trouvent dans [/concepts/oauth](/fr/concepts/oauth).
