---
read_when:
    - Créer ou déboguer des plugins OpenClaw natifs
    - Comprendre le modèle de capacités des plugins ou les limites de propriété
    - Travailler sur le pipeline de chargement des plugins ou le registre
    - Implémenter des hooks de runtime de fournisseur ou des plugins de canal
sidebarTitle: Internals
summary: 'Internes des plugins : modèle de capacités, propriété, contrats, pipeline de chargement et helpers de runtime'
title: Internes des plugins
x-i18n:
    generated_at: "2026-04-08T02:20:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: c40ecf14e2a0b2b8d332027aed939cd61fb4289a489f4cd4c076c96d707d1138
    source_path: plugins/architecture.md
    workflow: 15
---

# Internes des plugins

<Info>
  Il s’agit de la **référence d’architecture approfondie**. Pour des guides pratiques, voir :
  - [Install and use plugins](/fr/tools/plugin) — guide utilisateur
  - [Getting Started](/fr/plugins/building-plugins) — premier tutoriel de plugin
  - [Channel Plugins](/fr/plugins/sdk-channel-plugins) — créer un canal de messagerie
  - [Provider Plugins](/fr/plugins/sdk-provider-plugins) — créer un fournisseur de modèles
  - [SDK Overview](/fr/plugins/sdk-overview) — carte des imports et API d’enregistrement
</Info>

Cette page couvre l’architecture interne du système de plugins d’OpenClaw.

## Modèle public de capacités

Les capacités constituent le modèle public des **plugins natifs** dans OpenClaw. Chaque
plugin OpenClaw natif s’enregistre sur un ou plusieurs types de capacités :

| Capability             | Registration method                              | Example plugins                      |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| Inférence de texte     | `api.registerProvider(...)`                      | `openai`, `anthropic`                |
| Backend d’inférence CLI  | `api.registerCliBackend(...)`                    | `openai`, `anthropic`                |
| Parole                 | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`            |
| Transcription en temps réel | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| Voix en temps réel     | `api.registerRealtimeVoiceProvider(...)`         | `openai`                             |
| Compréhension des médias    | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                   |
| Génération d’images    | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Génération de musique  | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                  |
| Génération de vidéo    | `api.registerVideoGenerationProvider(...)`       | `qwen`                               |
| Récupération web              | `api.registerWebFetchProvider(...)`              | `firecrawl`                          |
| Recherche web             | `api.registerWebSearchProvider(...)`             | `google`                             |
| Canal / messagerie    | `api.registerChannel(...)`                       | `msteams`, `matrix`                  |

Un plugin qui enregistre zéro capacité mais fournit des hooks, outils ou
services est un plugin **legacy hook-only**. Ce modèle reste entièrement pris en charge.

### Position de compatibilité externe

Le modèle de capacités est intégré au core et utilisé aujourd’hui par les plugins
intégrés/natifs, mais la compatibilité des plugins externes exige encore une barre plus stricte que « c’est
exporté, donc c’est figé ».

Directive actuelle :

- **plugins externes existants :** conserver le fonctionnement des intégrations basées sur des hooks ; traitez
  cela comme la base de compatibilité
- **nouveaux plugins intégrés/natifs :** préférez un enregistrement explicite des capacités plutôt que
  des accès spécifiques à un fournisseur ou de nouveaux designs hook-only
- **plugins externes adoptant l’enregistrement de capacités :** autorisé, mais considérez les surfaces de helpers
  spécifiques aux capacités comme évolutives tant que la documentation ne marque pas explicitement
  un contrat comme stable

Règle pratique :

- les API d’enregistrement des capacités sont la direction prévue
- les hooks legacy restent la voie la plus sûre et la moins risquée pour les plugins externes pendant
  la transition
- toutes les sous-routes de helpers exportées ne se valent pas ; préférez le contrat étroit documenté,
  et non des exports de helpers accidentels

### Formes de plugins

OpenClaw classe chaque plugin chargé dans une forme selon son comportement
d’enregistrement réel (et pas seulement selon les métadonnées statiques) :

- **plain-capability** -- enregistre exactement un type de capacité (par exemple un
  plugin uniquement fournisseur comme `mistral`)
- **hybrid-capability** -- enregistre plusieurs types de capacités (par exemple
  `openai` possède l’inférence de texte, la parole, la compréhension des médias et la génération
  d’images)
- **hook-only** -- enregistre uniquement des hooks (typés ou personnalisés), sans capacités,
  outils, commandes ou services
- **non-capability** -- enregistre des outils, commandes, services ou routes, mais sans
  capacités

Utilisez `openclaw plugins inspect <id>` pour voir la forme d’un plugin et le détail
de ses capacités. Voir [CLI reference](/cli/plugins#inspect) pour les détails.

### Hooks legacy

Le hook `before_agent_start` reste pris en charge comme voie de compatibilité pour les
plugins hook-only. Des plugins legacy utilisés dans le monde réel en dépendent encore.

Orientation :

- le conserver fonctionnel
- le documenter comme legacy
- préférer `before_model_resolve` pour le travail de substitution de modèle/fournisseur
- préférer `before_prompt_build` pour le travail de mutation de prompt
- ne le retirer qu’une fois l’usage réel en baisse et lorsque la couverture des fixtures démontre la sécurité de la migration

### Signaux de compatibilité

Lorsque vous exécutez `openclaw doctor` ou `openclaw plugins inspect <id>`, vous pouvez voir
l’une de ces étiquettes :

| Signal                     | Meaning                                                      |
| -------------------------- | ------------------------------------------------------------ |
| **config valid**           | La configuration est analysée correctement et les plugins sont résolus                       |
| **compatibility advisory** | Le plugin utilise un modèle pris en charge mais plus ancien (p. ex. `hook-only`) |
| **legacy warning**         | Le plugin utilise `before_agent_start`, qui est obsolète        |
| **hard error**             | La configuration est invalide ou le plugin n’a pas pu être chargé                   |

Ni `hook-only` ni `before_agent_start` ne casseront votre plugin aujourd’hui --
`hook-only` est informatif, et `before_agent_start` ne déclenche qu’un avertissement. Ces
signaux apparaissent aussi dans `openclaw status --all` et `openclaw plugins doctor`.

## Vue d’ensemble de l’architecture

Le système de plugins d’OpenClaw comporte quatre couches :

1. **Manifest + découverte**
   OpenClaw trouve les plugins candidats à partir des chemins configurés, des racines de workspace,
   des racines globales d’extensions et des extensions intégrées. La découverte lit d’abord les manifests natifs
   `openclaw.plugin.json` ainsi que les manifests de bundles pris en charge.
2. **Activation + validation**
   Le core décide si un plugin découvert est activé, désactivé, bloqué ou
   sélectionné pour un emplacement exclusif comme la mémoire.
3. **Chargement du runtime**
   Les plugins OpenClaw natifs sont chargés en processus via jiti et enregistrent
   des capacités dans un registre central. Les bundles compatibles sont normalisés en
   enregistrements de registre sans import du code de runtime.
4. **Consommation de surface**
   Le reste d’OpenClaw lit le registre pour exposer outils, canaux, configuration
   des fournisseurs, hooks, routes HTTP, commandes CLI et services.

Pour le CLI des plugins en particulier, la découverte des commandes racine est divisée en deux phases :

- les métadonnées à l’analyse proviennent de `registerCli(..., { descriptors: [...] })`
- le vrai module CLI du plugin peut rester paresseux et s’enregistrer à la première invocation

Cela permet de garder le code CLI du plugin dans le plugin tout en laissant OpenClaw
réserver les noms de commandes racine avant l’analyse.

La limite de conception importante :

- la découverte + validation de configuration doivent fonctionner à partir des **métadonnées de manifest/schéma**
  sans exécuter le code du plugin
- le comportement natif au runtime provient du chemin `register(api)` du module du plugin

Cette séparation permet à OpenClaw de valider la configuration, d’expliquer les plugins absents/désactivés, et
de construire des indices d’interface/schéma avant que le runtime complet ne soit actif.

### Plugins de canal et outil de message partagé

Les plugins de canal n’ont pas besoin d’enregistrer un outil séparé d’envoi/modification/réaction pour les
actions de chat normales. OpenClaw conserve un outil `message` partagé dans le core, et
les plugins de canal possèdent derrière lui la découverte et l’exécution spécifiques au canal.

La limite actuelle est la suivante :

- le core possède l’hôte de l’outil `message` partagé, le câblage du prompt, la gestion
  de session/fil et le dispatch d’exécution
- les plugins de canal possèdent la découverte d’actions à portée, la découverte de capacités et tout fragment
  de schéma spécifique au canal
- les plugins de canal possèdent la grammaire de conversation de session spécifique au fournisseur, par exemple
  la façon dont les ids de conversation encodent les ids de fil ou héritent des conversations parentes
- les plugins de canal exécutent l’action finale via leur adaptateur d’actions

Pour les plugins de canal, la surface SDK est
`ChannelMessageActionAdapter.describeMessageTool(...)`. Cet appel unifié de découverte
permet à un plugin de renvoyer ensemble ses actions visibles, ses capacités et ses
contributions au schéma afin d’éviter toute dérive entre ces éléments.

Le core transmet la portée de runtime à cette étape de découverte. Les champs importants incluent :

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- `requesterSenderId` entrant de confiance

C’est important pour les plugins sensibles au contexte. Un canal peut masquer ou exposer
des actions de message selon le compte actif, la salle/le fil/le message courant ou
l’identité du demandeur de confiance, sans coder en dur des branches spécifiques à un canal dans l’outil
`message` du core.

C’est pourquoi les changements de routage d’embedded-runner restent du travail de plugin : le runner
est responsable de transmettre l’identité actuelle de chat/session à la limite de découverte
du plugin afin que l’outil `message` partagé expose la bonne surface possédée par le canal
pour le tour en cours.

Pour les helpers d’exécution possédés par un canal, les plugins intégrés doivent conserver le runtime
d’exécution dans leurs propres modules d’extension. Le core ne possède plus les runtimes
d’actions de message Discord, Slack, Telegram ou WhatsApp sous `src/agents/tools`.
Nous ne publions pas de sous-routes séparées `plugin-sdk/*-action-runtime`, et les plugins intégrés
doivent importer directement leur propre code de runtime local depuis leurs
modules possédés par l’extension.

La même limite s’applique aux seams SDK nommés par fournisseur en général : le core ne doit
pas importer de barrels de commodité spécifiques à un canal pour Slack, Discord, Signal,
WhatsApp ou des extensions similaires. Si le core a besoin d’un comportement, il doit soit
consommer le barrel `api.ts` / `runtime-api.ts` du plugin intégré lui-même, soit promouvoir le besoin
en capacité générique étroite dans le SDK partagé.

Pour les sondages en particulier, il existe deux chemins d’exécution :

- `outbound.sendPoll` est la base partagée pour les canaux compatibles avec le modèle
  commun de sondage
- `actions.handleAction("poll")` est le chemin préféré pour les sémantiques de sondage spécifiques à un canal
  ou des paramètres de sondage supplémentaires

Le core diffère désormais l’analyse partagée des sondages jusqu’à ce que le dispatch du sondage du plugin
refuse l’action, de sorte que les gestionnaires de sondage possédés par un plugin puissent accepter
des champs de sondage spécifiques au canal sans être d’abord bloqués par l’analyseur générique.

Voir [Load pipeline](#load-pipeline) pour la séquence complète de démarrage.

## Modèle de propriété des capacités

OpenClaw considère un plugin natif comme la limite de propriété d’une **entreprise** ou d’une
**fonctionnalité**, et non comme un fourre-tout d’intégrations sans lien.

Cela signifie :

- un plugin d’entreprise doit généralement posséder toutes les surfaces OpenClaw de cette entreprise
- un plugin de fonctionnalité doit généralement posséder toute la surface de fonctionnalité qu’il introduit
- les canaux doivent consommer les capacités partagées du core au lieu de réimplémenter
  de façon ad hoc le comportement des fournisseurs

Exemples :

- le plugin intégré `openai` possède le comportement de fournisseur de modèles OpenAI ainsi que le comportement OpenAI
  de parole + voix en temps réel + compréhension des médias + génération
  d’images
- le plugin intégré `elevenlabs` possède le comportement de parole ElevenLabs
- le plugin intégré `microsoft` possède le comportement de parole Microsoft
- le plugin intégré `google` possède le comportement de fournisseur de modèles Google ainsi que
  le comportement Google de compréhension des médias + génération d’images + recherche web
- le plugin intégré `firecrawl` possède le comportement de récupération web Firecrawl
- les plugins intégrés `minimax`, `mistral`, `moonshot` et `zai` possèdent leurs
  backends de compréhension des médias
- le plugin `voice-call` est un plugin de fonctionnalité : il possède le transport d’appel, les outils,
  le CLI, les routes et le pont Twilio media-stream, mais il consomme la parole partagée ainsi que
  les capacités de transcription en temps réel et de voix en temps réel au lieu d’importer directement des plugins fournisseurs

L’état final visé est le suivant :

- OpenAI vit dans un seul plugin même si cela couvre les modèles de texte, la parole, les images et
  la future vidéo
- un autre fournisseur peut faire de même pour sa propre surface
- les canaux ne se soucient pas du plugin fournisseur qui possède le provider ; ils consomment le
  contrat de capacité partagé exposé par le core

Voici la distinction clé :

- **plugin** = limite de propriété
- **capacité** = contrat du core que plusieurs plugins peuvent implémenter ou consommer

Ainsi, si OpenClaw ajoute un nouveau domaine comme la vidéo, la première question n’est pas
« quel fournisseur doit coder en dur la gestion de la vidéo ? » La première question est « quel est
le contrat de capacité vidéo du core ? ». Une fois ce contrat en place, les plugins fournisseurs
peuvent s’y enregistrer et les plugins de canal/fonctionnalité peuvent le consommer.

Si la capacité n’existe pas encore, la bonne démarche est généralement :

1. définir la capacité manquante dans le core
2. l’exposer via l’API/runtime du plugin de manière typée
3. raccorder les canaux/fonctionnalités à cette capacité
4. laisser les plugins fournisseurs enregistrer des implémentations

Cela maintient une propriété explicite tout en évitant un comportement du core qui dépend d’un
seul fournisseur ou d’un chemin de code spécifique à un plugin ponctuel.

### Superposition des capacités

Utilisez ce modèle mental pour décider où le code doit se trouver :

- **couche de capacités du core** : orchestration partagée, stratégie, repli, règles de fusion
  de configuration, sémantiques de livraison et contrats typés
- **couche de plugin fournisseur** : API spécifiques au fournisseur, authentification, catalogues de modèles, synthèse vocale,
  génération d’images, futurs backends vidéo, endpoints d’usage
- **couche de plugin de canal/fonctionnalité** : intégration Slack/Discord/voice-call/etc.
  qui consomme les capacités du core et les présente sur une surface

Par exemple, la synthèse vocale suit cette forme :

- le core possède la stratégie TTS à la réponse, l’ordre de repli, les préférences et la livraison sur canal
- `openai`, `elevenlabs` et `microsoft` possèdent les implémentations de synthèse
- `voice-call` consomme le helper de runtime TTS de téléphonie

Ce même modèle doit être privilégié pour les futures capacités.

### Exemple de plugin d’entreprise multi-capacités

Un plugin d’entreprise doit sembler cohérent de l’extérieur. Si OpenClaw a des
contrats partagés pour les modèles, la parole, la transcription en temps réel, la voix en temps réel, la compréhension des médias,
la génération d’images, la génération de vidéo, la récupération web et la recherche web,
un fournisseur peut posséder toutes ses surfaces au même endroit :

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // auth/model catalog/runtime hooks
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // vendor speech config — implement the SpeechProviderPlugin interface directly
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // credential + fetch logic
      }),
    );
  },
};

export default plugin;
```

Ce qui importe, ce ne sont pas les noms exacts des helpers. C’est la forme qui importe :

- un plugin possède la surface du fournisseur
- le core possède toujours les contrats de capacité
- les plugins de canal et de fonctionnalité consomment les helpers `api.runtime.*`, et non le code fournisseur
- les tests de contrat peuvent affirmer que le plugin a enregistré les capacités qu’il
  prétend posséder

### Exemple de capacité : compréhension vidéo

OpenClaw traite déjà la compréhension image/audio/vidéo comme une seule
capacité partagée. Le même modèle de propriété s’y applique :

1. le core définit le contrat de compréhension des médias
2. les plugins fournisseurs enregistrent `describeImage`, `transcribeAudio` et
   `describeVideo` selon le cas
3. les plugins de canal et de fonctionnalité consomment le comportement partagé du core au lieu de
   se raccorder directement au code fournisseur

Cela évite d’intégrer dans le core les hypothèses vidéo d’un seul fournisseur. Le plugin possède
la surface du fournisseur ; le core possède le contrat de capacité et le comportement de repli.

La génération de vidéo suit déjà cette même séquence : le core possède le
contrat de capacité typé et le helper de runtime, et les plugins fournisseurs enregistrent
des implémentations `api.registerVideoGenerationProvider(...)` sur celui-ci.

Besoin d’une checklist de déploiement concrète ? Voir
[Capability Cookbook](/fr/plugins/architecture).

## Contrats et application

La surface de l’API des plugins est volontairement typée et centralisée dans
`OpenClawPluginApi`. Ce contrat définit les points d’enregistrement pris en charge et
les helpers de runtime sur lesquels un plugin peut s’appuyer.

Pourquoi c’est important :

- les auteurs de plugins obtiennent une norme interne stable
- le core peut rejeter une propriété dupliquée, par exemple deux plugins enregistrant le même id de
  fournisseur
- le démarrage peut exposer des diagnostics exploitables pour un enregistrement mal formé
- les tests de contrat peuvent appliquer la propriété des plugins intégrés et empêcher une dérive silencieuse

Il existe deux couches d’application :

1. **application au runtime de l’enregistrement**
   Le registre des plugins valide les enregistrements au chargement des plugins. Exemples :
   ids de fournisseur dupliqués, ids de fournisseur de parole dupliqués et
   enregistrements mal formés produisent des diagnostics de plugin au lieu d’un comportement indéfini.
2. **tests de contrat**
   Les plugins intégrés sont capturés dans des registres de contrat pendant les exécutions de test afin
   qu’OpenClaw puisse affirmer explicitement la propriété. Aujourd’hui, cela est utilisé pour les
   fournisseurs de modèles, les fournisseurs de parole, les fournisseurs de recherche web et la propriété
   des enregistrements intégrés.

L’effet pratique est qu’OpenClaw sait à l’avance quel plugin possède quelle
surface. Cela permet au core et aux canaux de composer sans friction, car la propriété est
déclarée, typée et testable plutôt qu’implicite.

### Ce qui doit appartenir à un contrat

Les bons contrats de plugin sont :

- typés
- petits
- spécifiques à une capacité
- possédés par le core
- réutilisables par plusieurs plugins
- consommables par les canaux/fonctionnalités sans connaissance fournisseur

Les mauvais contrats de plugin sont :

- une stratégie spécifique à un fournisseur cachée dans le core
- des échappatoires ponctuelles de plugin qui contournent le registre
- du code de canal qui atteint directement une implémentation fournisseur
- des objets de runtime ad hoc qui ne font pas partie de `OpenClawPluginApi` ni de
  `api.runtime`

En cas de doute, montez le niveau d’abstraction : définissez d’abord la capacité, puis
laissez les plugins s’y brancher.

## Modèle d’exécution

Les plugins OpenClaw natifs s’exécutent **en processus** avec la Gateway. Ils ne sont pas
sandboxés. Un plugin natif chargé partage la même limite de confiance au niveau du processus que le code du core.

Implications :

- un plugin natif peut enregistrer des outils, des gestionnaires réseau, des hooks et des services
- un bogue dans un plugin natif peut faire planter ou déstabiliser la gateway
- un plugin natif malveillant équivaut à une exécution de code arbitraire dans le
  processus OpenClaw

Les bundles compatibles sont plus sûrs par défaut, car OpenClaw les traite actuellement
comme des packs de métadonnées/contenu. Dans les versions actuelles, cela signifie surtout des
Skills intégrées.

Utilisez des listes d’autorisation et des chemins explicites d’installation/chargement pour les plugins non intégrés.
Traitez les plugins de workspace comme du code de développement, et non comme des valeurs par défaut de production.

Pour les noms de packages de workspace intégrés, conservez l’id du plugin ancré dans le
nom npm : `@openclaw/<id>` par défaut, ou un suffixe typé approuvé tel que
`-provider`, `-plugin`, `-speech`, `-sandbox` ou `-media-understanding` lorsque
le package expose intentionnellement un rôle de plugin plus étroit.

Note de confiance importante :

- `plugins.allow` fait confiance aux **ids de plugin**, pas à la provenance de la source.
- Un plugin de workspace ayant le même id qu’un plugin intégré masque intentionnellement
  la copie intégrée lorsque ce plugin de workspace est activé/sur liste d’autorisation.
- C’est normal et utile pour le développement local, les tests de correctif et les hotfixes.

## Limite d’export

OpenClaw exporte des capacités, pas des commodités d’implémentation.

Conservez l’enregistrement des capacités public. Réduisez les exports de helpers hors contrat :

- sous-routes de helpers spécifiques à des plugins intégrés
- sous-routes de plomberie de runtime non destinées à l’API publique
- helpers de commodité spécifiques à un fournisseur
- helpers de setup/onboarding qui sont des détails d’implémentation

Certaines sous-routes de helpers de plugins intégrés restent encore dans la carte d’export
SDK générée pour des raisons de compatibilité et de maintenance des plugins intégrés. Exemples actuels :
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` et plusieurs seams `plugin-sdk/matrix*`. Traitez-les comme
des exports réservés de détail d’implémentation, et non comme le modèle SDK recommandé pour
de nouveaux plugins tiers.

## Pipeline de chargement

Au démarrage, OpenClaw fait approximativement ceci :

1. découvre les racines candidates de plugins
2. lit les manifests natifs ou de bundles compatibles ainsi que les métadonnées de package
3. rejette les candidats non sûrs
4. normalise la configuration des plugins (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. décide de l’activation pour chaque candidat
6. charge les modules natifs activés via jiti
7. appelle les hooks natifs `register(api)` (ou `activate(api)` — alias legacy) et collecte les enregistrements dans le registre des plugins
8. expose le registre aux surfaces de commandes/runtime

<Note>
`activate` est un alias legacy de `register` — le chargeur résout celui qui est présent (`def.register ?? def.activate`) et l’appelle au même moment. Tous les plugins intégrés utilisent `register` ; préférez `register` pour les nouveaux plugins.
</Note>

Les barrières de sécurité se produisent **avant** l’exécution au runtime. Les candidats sont bloqués
lorsque le point d’entrée s’échappe de la racine du plugin, que le chemin est inscriptible par tous, ou que la propriété du chemin semble suspecte pour les plugins non intégrés.

### Comportement manifest-first

Le manifest est la source de vérité du plan de contrôle. OpenClaw l’utilise pour :

- identifier le plugin
- découvrir les canaux/Skills/schémas de configuration déclarés ou les capacités du bundle
- valider `plugins.entries.<id>.config`
- enrichir les libellés/placeholders de Control UI
- afficher les métadonnées d’installation/catalogue

Pour les plugins natifs, le module de runtime est la partie plan de données. Il enregistre le
comportement réel tel que hooks, outils, commandes ou flux de fournisseurs.

### Ce que le chargeur met en cache

OpenClaw conserve de courts caches en processus pour :

- les résultats de découverte
- les données de registre de manifests
- les registres de plugins chargés

Ces caches réduisent les surcharges au démarrage en rafale et sur les commandes répétées. Ils peuvent être considérés
comme des caches de performance à courte durée de vie, pas comme une persistance.

Note de performance :

- Définissez `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` ou
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` pour désactiver ces caches.
- Ajustez les fenêtres de cache avec `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` et
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Modèle de registre

Les plugins chargés ne mutent pas directement des globales aléatoires du core. Ils s’enregistrent dans un
registre central de plugins.

Le registre suit :

- les enregistrements de plugin (identité, source, origine, statut, diagnostics)
- les outils
- les hooks legacy et les hooks typés
- les canaux
- les fournisseurs
- les gestionnaires RPC de gateway
- les routes HTTP
- les registrars CLI
- les services en arrière-plan
- les commandes possédées par des plugins

Les fonctionnalités du core lisent ensuite ce registre au lieu de parler directement aux modules de plugins.
Cela maintient un chargement unidirectionnel :

- module de plugin -> enregistrement dans le registre
- runtime du core -> consommation du registre

Cette séparation est importante pour la maintenabilité. Elle signifie que la plupart des surfaces du core n’ont
besoin que d’un seul point d’intégration : « lire le registre », et non « gérer spécialement chaque module de plugin ».

## Callbacks de liaison de conversation

Les plugins qui lient une conversation peuvent réagir lorsqu’une approbation est résolue.

Utilisez `api.onConversationBindingResolved(...)` pour recevoir un callback après qu’une demande de liaison a été approuvée ou refusée :

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // A binding now exists for this plugin + conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // The request was denied; clear any local pending state.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Champs du payload du callback :

- `status`: `"approved"` ou `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` ou `"deny"`
- `binding`: la liaison résolue pour les demandes approuvées
- `request`: le résumé de la demande d’origine, l’indice de détachement, l’id de l’expéditeur et
  les métadonnées de conversation

Ce callback est uniquement une notification. Il ne change pas les permissions de liaison d’une
conversation, et il s’exécute après la fin de la gestion d’approbation du core.

## Hooks de runtime des fournisseurs

Les plugins fournisseurs ont maintenant deux couches :

- métadonnées de manifest : `providerAuthEnvVars` pour une recherche légère
  de l’authentification fournisseur par variable d’environnement avant le chargement du runtime, `channelEnvVars` pour une recherche légère
  de la configuration/authentification de canal avant le chargement du runtime, ainsi que `providerAuthChoices` pour des libellés légers
  d’onboarding/de choix d’authentification et des métadonnées de drapeaux CLI avant le chargement du runtime
- hooks au moment de la configuration : `catalog` / legacy `discovery` ainsi que `applyConfigDefaults`
- hooks de runtime : `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `resolveExternalAuthProfiles`,
  `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`, `normalizeResolvedModel`,
  `contributeResolvedModelCompat`, `capabilities`,
  `normalizeToolSchemas`, `inspectToolSchemas`,
  `resolveReasoningOutputMode`, `prepareExtraParams`, `createStreamFn`,
  `wrapStreamFn`, `resolveTransportTurnState`,
  `resolveWebSocketSessionPolicy`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `isCacheTtlEligible`,
  `buildMissingAuthMessage`, `suppressBuiltInModel`, `augmentModelCatalog`,
  `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `isModernModelRef`, `prepareRuntimeAuth`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `createEmbeddingProvider`,
  `buildReplayPolicy`,
  `sanitizeReplayHistory`, `validateReplayTurns`, `onModelSelected`

OpenClaw possède toujours la boucle d’agent générique, le repli, la gestion de transcription et la
stratégie d’outils. Ces hooks sont la surface d’extension pour le comportement spécifique aux fournisseurs sans
nécessiter tout un transport d’inférence personnalisé.

Utilisez le manifest `providerAuthEnvVars` lorsque le fournisseur possède des identifiants basés sur l’environnement
que les chemins génériques d’authentification/statut/sélecteur de modèle doivent voir sans charger le runtime du plugin.
Utilisez le manifest `providerAuthChoices` lorsque les surfaces CLI d’onboarding/de choix d’authentification
doivent connaître l’id de choix du fournisseur, les libellés de groupe et le câblage d’authentification simple
à un seul drapeau sans charger le runtime du fournisseur. Conservez `envVars` dans le runtime du fournisseur
pour les indices orientés opérateur tels que les libellés d’onboarding ou les
variables de configuration OAuth client-id/client-secret.

Utilisez le manifest `channelEnvVars` lorsqu’un canal possède une authentification ou un setup piloté par l’environnement
que les chemins génériques de repli shell-env, les vérifications config/statut ou les invites de setup
doivent voir sans charger le runtime du canal.

### Ordre des hooks et utilisation

Pour les plugins de modèle/fournisseur, OpenClaw appelle les hooks dans cet ordre approximatif.
La colonne « When to use » est le guide de décision rapide.

| #   | Hook                              | What it does                                                                                                   | When to use                                                                                                                                 |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Publie la configuration du fournisseur dans `models.providers` pendant la génération de `models.json`                                | Le fournisseur possède un catalogue ou des valeurs par défaut de base URL                                                                                                |
| 2   | `applyConfigDefaults`             | Applique des valeurs par défaut globales possédées par le fournisseur pendant la matérialisation de la configuration                                      | Les valeurs par défaut dépendent du mode d’authentification, de l’environnement ou de la sémantique de famille de modèles du fournisseur                                                                       |
| --  | _(built-in model lookup)_         | OpenClaw essaie d’abord le chemin normal registre/catalogue                                                          | _(pas un hook de plugin)_                                                                                                                       |
| 3   | `normalizeModelId`                | Normalise les alias legacy ou preview d’id de modèle avant la recherche                                                     | Le fournisseur possède le nettoyage des alias avant la résolution du modèle canonique                                                                               |
| 4   | `normalizeTransport`              | Normalise `api` / `baseUrl` de la famille du fournisseur avant l’assemblage générique du modèle                                      | Le fournisseur possède le nettoyage du transport pour des ids de fournisseur personnalisés dans la même famille de transport                                                        |
| 5   | `normalizeConfig`                 | Normalise `models.providers.<id>` avant la résolution runtime/fournisseur                                           | Le fournisseur a besoin d’un nettoyage de configuration qui doit vivre avec le plugin ; les helpers intégrés de famille Google assurent aussi le backstop des entrées de configuration Google prises en charge |
| 6   | `applyNativeStreamingUsageCompat` | Applique des réécritures de compatibilité native de streaming-usage aux fournisseurs de configuration                                               | Le fournisseur a besoin de correctifs de métadonnées de streaming-usage pilotés par l’endpoint                                                                        |
| 7   | `resolveConfigApiKey`             | Résout l’authentification env-marker pour les fournisseurs de configuration avant le chargement de l’authentification runtime                                       | Le fournisseur possède une résolution de clé API env-marker ; `amazon-bedrock` a également ici un résolveur env-marker AWS intégré                |
| 8   | `resolveSyntheticAuth`            | Expose une authentification locale/self-hosted ou adossée à la configuration sans persister de texte brut                                   | Le fournisseur peut fonctionner avec un marqueur d’identifiant synthétique/local                                                                               |
| 9   | `resolveExternalAuthProfiles`     | Superpose des profils d’authentification externes possédés par le fournisseur ; `persistence` par défaut est `runtime-only` pour les identifiants possédés par CLI/app | Le fournisseur réutilise des identifiants d’authentification externes sans persister des refresh tokens copiés                                                          |
| 10  | `shouldDeferSyntheticProfileAuth` | Abaisse les placeholders de profils synthétiques stockés derrière l’authentification adossée à env/config                                      | Le fournisseur stocke des profils placeholders synthétiques qui ne doivent pas gagner en priorité                                                               |
| 11  | `resolveDynamicModel`             | Repli synchrone pour les ids de modèle possédés par le fournisseur qui ne sont pas encore dans le registre local                                       | Le fournisseur accepte des ids de modèle amont arbitraires                                                                                               |
| 12  | `prepareDynamicModel`             | Échauffement asynchrone, puis `resolveDynamicModel` s’exécute à nouveau                                                           | Le fournisseur a besoin de métadonnées réseau avant de résoudre des ids inconnus                                                                                |
| 13  | `normalizeResolvedModel`          | Réécriture finale avant que l’embedded runner utilise le modèle résolu                                               | Le fournisseur a besoin de réécritures de transport tout en utilisant un transport du core                                                                           |
| 14  | `contributeResolvedModelCompat`   | Contribue des drapeaux de compatibilité pour les modèles fournisseurs derrière un autre transport compatible                                  | Le fournisseur reconnaît ses propres modèles sur des transports proxy sans prendre la main sur le fournisseur                                                     |
| 15  | `capabilities`                    | Métadonnées de transcription/outillage possédées par le fournisseur utilisées par la logique partagée du core                                           | Le fournisseur a besoin de particularités de transcription/famille de fournisseur                                                                                            |
| 16  | `normalizeToolSchemas`            | Normalise les schémas d’outils avant que l’embedded runner ne les voie                                                    | Le fournisseur a besoin d’un nettoyage des schémas pour la famille de transport                                                                                              |
| 17  | `inspectToolSchemas`              | Expose des diagnostics de schéma possédés par le fournisseur après normalisation                                                  | Le fournisseur veut des avertissements de mots-clés sans apprendre au core des règles spécifiques à un fournisseur                                                               |
| 18  | `resolveReasoningOutputMode`      | Sélectionne le contrat de sortie de raisonnement natif ou balisé                                                              | Le fournisseur a besoin d’une sortie raisonnement/finale balisée au lieu de champs natifs                                                                       |
| 19  | `prepareExtraParams`              | Normalisation des paramètres de requête avant les wrappers génériques d’options de stream                                              | Le fournisseur a besoin de paramètres de requête par défaut ou d’un nettoyage spécifique                                                                         |
| 20  | `createStreamFn`                  | Remplace entièrement le chemin normal du stream par un transport personnalisé                                                   | Le fournisseur a besoin d’un protocole filaire personnalisé, pas seulement d’un wrapper                                                                                   |
| 21  | `wrapStreamFn`                    | Wrapper de stream après application des wrappers génériques                                                              | Le fournisseur a besoin de wrappers de compatibilité sur en-têtes/corps/modèle de requête sans transport personnalisé                                                        |
| 22  | `resolveTransportTurnState`       | Attache des en-têtes ou métadonnées natifs par tour                                                           | Le fournisseur veut que les transports génériques envoient une identité de tour native au fournisseur                                                                     |
| 23  | `resolveWebSocketSessionPolicy`   | Attache des en-têtes WebSocket natifs ou une stratégie de refroidissement de session                                                    | Le fournisseur veut que les transports WS génériques ajustent les en-têtes de session ou la stratégie de repli                                                             |
| 24  | `formatApiKey`                    | Formateur de profil d’authentification : le profil stocké devient la chaîne `apiKey` du runtime                                     | Le fournisseur stocke des métadonnées d’authentification supplémentaires et a besoin d’un format de jeton runtime personnalisé                                                                  |
| 25  | `refreshOAuth`                    | Surcharge du refresh OAuth pour des endpoints de refresh personnalisés ou une stratégie de panne du refresh                                  | Le fournisseur ne correspond pas aux refreshers partagés `pi-ai`                                                                                         |
| 26  | `buildAuthDoctorHint`             | Indice de réparation ajouté lorsqu’un refresh OAuth échoue                                                                  | Le fournisseur a besoin d’indications de réparation d’authentification possédées par le fournisseur après échec du refresh                                                                    |
| 27  | `matchesContextOverflowError`     | Matcher de dépassement de fenêtre de contexte possédé par le fournisseur                                                                 | Le fournisseur a des erreurs brutes de dépassement que les heuristiques génériques manqueraient                                                                              |
| 28  | `classifyFailoverReason`          | Classification de raison de repli possédée par le fournisseur                                                                  | Le fournisseur peut mapper des erreurs API/transport brutes à rate-limit/surcharge/etc.                                                                        |
| 29  | `isCacheTtlEligible`              | Stratégie de cache de prompt pour les fournisseurs proxy/backhaul                                                               | Le fournisseur a besoin d’une gestion spécifique au proxy du TTL de cache                                                                                              |
| 30  | `buildMissingAuthMessage`         | Remplacement du message générique de récupération d’authentification manquante                                                      | Le fournisseur a besoin d’un indice de récupération spécifique à un fournisseur                                                                               |
| 31  | `suppressBuiltInModel`            | Suppression de modèles amont obsolètes avec indice optionnel orienté utilisateur                                          | Le fournisseur doit masquer des lignes amont obsolètes ou les remplacer par un indice fournisseur                                                               |
| 32  | `augmentModelCatalog`             | Lignes de catalogue synthétiques/finales ajoutées après découverte                                                          | Le fournisseur a besoin de lignes synthétiques forward-compat dans `models list` et les sélecteurs                                                                   |
| 33  | `isBinaryThinking`                | Bascule on/off du raisonnement pour les fournisseurs binary-thinking                                                          | Le fournisseur n’expose qu’un raisonnement binaire activé/désactivé                                                                                                |
| 34  | `supportsXHighThinking`           | Prise en charge du raisonnement `xhigh` pour des modèles sélectionnés                                                                  | Le fournisseur veut `xhigh` uniquement sur un sous-ensemble de modèles                                                                                           |
| 35  | `resolveDefaultThinkingLevel`     | Niveau `/think` par défaut pour une famille de modèles spécifique                                                             | Le fournisseur possède la stratégie `/think` par défaut d’une famille de modèles                                                                                    |
| 36  | `isModernModelRef`                | Matcher de modèle moderne pour les filtres de profils live et la sélection smoke                                              | Le fournisseur possède la correspondance de modèles préférés live/smoke                                                                                           |
| 37  | `prepareRuntimeAuth`              | Échange un identifiant configuré contre le vrai jeton/clé de runtime juste avant l’inférence                       | Le fournisseur a besoin d’un échange de jeton ou d’un identifiant de requête à courte durée de vie                                                                           |
| 38  | `resolveUsageAuth`                | Résout les identifiants d’usage/facturation pour `/usage` et les surfaces de statut associées                                     | Le fournisseur a besoin d’une analyse personnalisée du jeton d’usage/quota ou d’un autre identifiant d’usage                                                             |
| 39  | `fetchUsageSnapshot`              | Récupère et normalise les snapshots d’usage/quota spécifiques au fournisseur une fois l’authentification résolue                             | Le fournisseur a besoin d’un endpoint ou d’un parseur de payload spécifique à l’usage                                                                         |
| 40  | `createEmbeddingProvider`         | Construit un adaptateur d’embeddings possédé par le fournisseur pour mémoire/recherche                                                     | Le comportement d’embeddings mémoire appartient au plugin fournisseur                                                                                  |
| 41  | `buildReplayPolicy`               | Renvoie une stratégie de replay contrôlant la gestion de la transcription pour le fournisseur                                        | Le fournisseur a besoin d’une stratégie de transcription personnalisée (par exemple suppression de blocs de raisonnement)                                                             |
| 42  | `sanitizeReplayHistory`           | Réécrit l’historique de replay après le nettoyage générique de transcription                                                        | Le fournisseur a besoin de réécritures spécifiques au replay au-delà des helpers partagés de compaction                                                           |
| 43  | `validateReplayTurns`             | Validation ou remodelage final des tours rejoués avant l’embedded runner                                           | Le transport du fournisseur a besoin d’une validation plus stricte après la sanitation générique                                                                  |
| 44  | `onModelSelected`                 | Exécute des effets de bord possédés par le fournisseur après la sélection                                                                 | Le fournisseur a besoin de télémétrie ou d’état possédé lorsque un modèle devient actif                                                                |

`normalizeModelId`, `normalizeTransport` et `normalizeConfig` vérifient d’abord le
plugin fournisseur correspondant, puis parcourent les autres plugins fournisseurs capables de hooks
jusqu’à ce que l’un d’eux modifie réellement l’id du modèle ou le transport/la configuration. Cela permet
aux shims de fournisseur d’alias/compatibilité de fonctionner sans imposer à l’appelant de savoir quel
plugin intégré possède la réécriture. Si aucun hook fournisseur ne réécrit une entrée de configuration
prise en charge de la famille Google, le normaliseur de configuration Google intégré applique tout de même
ce nettoyage de compatibilité.

Si le fournisseur a besoin d’un protocole filaire entièrement personnalisé ou d’un exécuteur de requêtes personnalisé,
il s’agit d’une autre classe d’extension. Ces hooks servent au comportement fournisseur
qui s’exécute toujours sur la boucle d’inférence normale d’OpenClaw.

### Exemple de fournisseur

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Exemples intégrés

- Anthropic utilise `resolveDynamicModel`, `capabilities`, `buildAuthDoctorHint`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `isCacheTtlEligible`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  et `wrapStreamFn` parce qu’il possède la forward-compat de Claude 4.6,
  les indices de famille de fournisseurs, les indications de réparation d’authentification, l’intégration
  d’endpoint d’usage, l’éligibilité du cache de prompt, les valeurs par défaut de configuration sensibles à l’authentification, la stratégie
  de pensée par défaut/adaptative de Claude, et la mise en forme spécifique à Anthropic du stream pour
  les en-têtes bêta, `/fast` / `serviceTier`, et `context1m`.
- Les helpers de stream spécifiques à Claude chez Anthropic restent pour l’instant dans le seam public
  `api.ts` / `contract-api.ts` du plugin intégré lui-même. Cette surface de package
  exporte `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, ainsi que des builders de wrappers Anthropic plus bas niveau au lieu d’élargir le SDK générique autour des règles
  d’en-têtes bêta d’un seul fournisseur.
- OpenAI utilise `resolveDynamicModel`, `normalizeResolvedModel` et
  `capabilities` ainsi que `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` et `isModernModelRef`
  parce qu’il possède la forward-compat de GPT-5.4, la normalisation directe OpenAI
  `openai-completions` -> `openai-responses`, les
  indices d’authentification tenant compte de Codex, la suppression de Spark, les lignes synthétiques de liste OpenAI, et la stratégie de pensée / modèle live de GPT-5 ; la famille de stream `openai-responses-defaults`
  possède les wrappers partagés natifs OpenAI Responses pour les en-têtes
  d’attribution, `/fast`/`serviceTier`, la verbosité du texte, la recherche web Codex native,
  la mise en forme du payload de compatibilité reasoning, et la gestion du contexte Responses.
- OpenRouter utilise `catalog` ainsi que `resolveDynamicModel` et
  `prepareDynamicModel` parce que le fournisseur est pass-through et peut exposer de nouveaux
  ids de modèle avant la mise à jour du catalogue statique d’OpenClaw ; il utilise aussi
  `capabilities`, `wrapStreamFn` et `isCacheTtlEligible` pour conserver hors du core les en-têtes de requête spécifiques au fournisseur, les métadonnées de routage, les correctifs de raisonnement et la stratégie de cache de prompt. Sa stratégie de replay vient de la
  famille `passthrough-gemini`, tandis que la famille de stream `openrouter-thinking`
  possède l’injection de raisonnement proxy et les contournements des modèles non pris en charge / `auto`.
- GitHub Copilot utilise `catalog`, `auth`, `resolveDynamicModel` et
  `capabilities` ainsi que `prepareRuntimeAuth` et `fetchUsageSnapshot` parce qu’il
  a besoin d’une connexion d’appareil possédée par le fournisseur, d’un comportement de repli des modèles, de particularités de transcription Claude, d’un échange jeton GitHub -> jeton Copilot, et d’un endpoint d’usage possédé par le fournisseur.
- OpenAI Codex utilise `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` et `augmentModelCatalog` ainsi que
  `prepareExtraParams`, `resolveUsageAuth` et `fetchUsageSnapshot` parce qu’il
  fonctionne encore sur les transports OpenAI du core mais possède sa normalisation
  transport/base URL, sa stratégie de repli du refresh OAuth, son choix de transport par défaut,
  ses lignes synthétiques de catalogue Codex, et l’intégration de l’endpoint d’usage ChatGPT ; il
  partage la même famille de stream `openai-responses-defaults` que l’OpenAI direct.
- Google AI Studio et Gemini CLI OAuth utilisent `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` et `isModernModelRef` parce que la
  famille de replay `google-gemini` possède le repli forward-compat de Gemini 3.1,
  la validation native de replay Gemini, la sanitation du replay bootstrap, le
  mode de sortie reasoning balisé, et la correspondance de modèles modernes, tandis que la
  famille de stream `google-thinking` possède la normalisation du payload thinking de Gemini ;
  Gemini CLI OAuth utilise aussi `formatApiKey`, `resolveUsageAuth` et
  `fetchUsageSnapshot` pour le formatage des jetons, l’analyse des jetons et le câblage de l’endpoint de quota.
- Anthropic Vertex utilise `buildReplayPolicy` via la
  famille de replay `anthropic-by-model`, afin que le nettoyage de replay spécifique à Claude reste
  limité aux ids Claude au lieu de tous les transports `anthropic-messages`.
- Amazon Bedrock utilise `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` et `resolveDefaultThinkingLevel` parce qu’il possède
  la classification spécifique à Bedrock des erreurs de throttle/not-ready/context-overflow
  pour le trafic Anthropic-on-Bedrock ; sa stratégie de replay partage encore la même garde
  `anthropic-by-model` limitée à Claude.
- OpenRouter, Kilocode, Opencode et Opencode Go utilisent `buildReplayPolicy`
  via la famille de replay `passthrough-gemini` parce qu’ils proxifient des modèles Gemini
  via des transports compatibles OpenAI et ont besoin de la sanitation des thought-signatures Gemini
  sans validation native de replay Gemini ni réécritures bootstrap.
- MiniMax utilise `buildReplayPolicy` via la
  famille de replay `hybrid-anthropic-openai` parce qu’un seul fournisseur possède à la fois
  des sémantiques Anthropic-message et OpenAI-compatible ; cela conserve la suppression
  des blocs de pensée limités à Claude côté Anthropic tout en remettant le mode de sortie de raisonnement sur natif, et la famille de stream `minimax-fast-mode` possède les réécritures de modèles fast-mode sur le chemin de stream partagé.
- Moonshot utilise `catalog` ainsi que `wrapStreamFn` parce qu’il utilise encore le transport
  OpenAI partagé mais a besoin d’une normalisation du payload thinking possédée par le fournisseur ; la
  famille de stream `moonshot-thinking` mappe la configuration et l’état `/think` sur son payload thinking binaire natif.
- Kilocode utilise `catalog`, `capabilities`, `wrapStreamFn` et
  `isCacheTtlEligible` parce qu’il a besoin d’en-têtes de requête possédés par le fournisseur,
  d’une normalisation du payload reasoning, d’indices de transcription Gemini, et d’une gestion du TTL de cache Anthropic ; la famille de stream `kilocode-thinking` conserve l’injection de pensée Kilo sur le chemin de stream proxy partagé tout en ignorant `kilo/auto` et d’autres ids de modèles proxy qui ne prennent pas en charge des payloads de raisonnement explicites.
- Z.AI utilise `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` et `fetchUsageSnapshot` parce qu’il possède le repli GLM-5,
  les valeurs par défaut `tool_stream`, l’UX de pensée binaire, la correspondance de modèles modernes, ainsi que l’authentification d’usage + la récupération du quota ; la famille de stream `tool-stream-default-on` conserve hors de la colle manuscrite par fournisseur le wrapper `tool_stream` activé par défaut.
- xAI utilise `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` et `isModernModelRef`
  parce qu’il possède la normalisation native du transport xAI Responses, les réécritures d’alias fast-mode de Grok, la valeur par défaut `tool_stream`, le nettoyage strict-tool / payload reasoning, la réutilisation d’authentification de repli pour les outils possédés par le plugin, la résolution forward-compat des modèles Grok, et les correctifs de compatibilité possédés par le fournisseur tels que le profil de schéma d’outils xAI, les mots-clés de schéma non pris en charge, `web_search` natif, et le décodage HTML-entity des arguments d’appel d’outil.
- Mistral, OpenCode Zen et OpenCode Go utilisent uniquement `capabilities` pour conserver hors du core les particularités de transcription/outillage.
- Les fournisseurs intégrés à catalogue seul tels que `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` et `volcengine` utilisent
  uniquement `catalog`.
- Qwen utilise `catalog` pour son fournisseur de texte ainsi que des enregistrements partagés de compréhension des médias et de génération de vidéo pour ses surfaces multimodales.
- MiniMax et Xiaomi utilisent `catalog` ainsi que des hooks d’usage parce que leur comportement `/usage`
  appartient au plugin, même si l’inférence s’exécute encore via les transports partagés.

## Helpers de runtime

Les plugins peuvent accéder à certains helpers du core via `api.runtime`. Pour la TTS :

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Remarques :

- `textToSpeech` renvoie le payload de sortie TTS normal du core pour les surfaces de fichier/note vocale.
- Utilise la configuration du core `messages.tts` et la sélection du fournisseur.
- Renvoie un buffer audio PCM + fréquence d’échantillonnage. Les plugins doivent rééchantillonner/encoder selon les fournisseurs.
- `listVoices` est facultatif selon le fournisseur. Utilisez-le pour les sélecteurs de voix possédés par le fournisseur ou les flux de setup.
- Les listes de voix peuvent inclure des métadonnées plus riches telles que la locale, le genre et des tags de personnalité pour des sélecteurs sensibles au fournisseur.
- OpenAI et ElevenLabs prennent aujourd’hui en charge la téléphonie. Microsoft non.

Les plugins peuvent aussi enregistrer des fournisseurs de parole via `api.registerSpeechProvider(...)`.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Remarques :

- Conservez dans le core la stratégie TTS, le repli et la livraison de réponse.
- Utilisez les fournisseurs de parole pour le comportement de synthèse possédé par le fournisseur.
- L’entrée legacy Microsoft `edge` est normalisée vers l’id de fournisseur `microsoft`.
- Le modèle de propriété préféré est orienté entreprise : un seul plugin fournisseur peut posséder
  le texte, la parole, l’image et de futurs fournisseurs média à mesure qu’OpenClaw ajoute ces
  contrats de capacité.

Pour la compréhension image/audio/vidéo, les plugins enregistrent un fournisseur typé unique
de compréhension des médias au lieu d’un sac générique clé/valeur :

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Remarques :

- Conservez dans le core l’orchestration, le repli, la configuration et le câblage de canal.
- Conservez le comportement fournisseur dans le plugin fournisseur.
- L’expansion additive doit rester typée : nouvelles méthodes facultatives, nouveaux champs de résultat facultatifs, nouvelles capacités facultatives.
- La génération de vidéo suit déjà le même modèle :
  - le core possède le contrat de capacité et le helper de runtime
  - les plugins fournisseurs enregistrent `api.registerVideoGenerationProvider(...)`
  - les plugins de fonctionnalité/canal consomment `api.runtime.videoGeneration.*`

Pour les helpers de runtime media-understanding, les plugins peuvent appeler :

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

Pour la transcription audio, les plugins peuvent utiliser soit le runtime media-understanding,
soit l’alias STT plus ancien :

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Remarques :

- `api.runtime.mediaUnderstanding.*` est la surface partagée préférée pour la
  compréhension image/audio/vidéo.
- Utilise la configuration audio du core media-understanding (`tools.media.audio`) et l’ordre de repli du fournisseur.
- Renvoie `{ text: undefined }` lorsqu’aucune sortie de transcription n’est produite (par exemple entrée ignorée/non prise en charge).
- `api.runtime.stt.transcribeAudioFile(...)` reste un alias de compatibilité.

Les plugins peuvent aussi lancer des exécutions de sous-agent en arrière-plan via `api.runtime.subagent` :

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Remarques :

- `provider` et `model` sont des surcharges facultatives par exécution, pas des changements de session persistants.
- OpenClaw n’honore ces champs de surcharge que pour les appelants de confiance.
- Pour les exécutions de repli possédées par un plugin, les opérateurs doivent activer explicitement `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Utilisez `plugins.entries.<id>.subagent.allowedModels` pour limiter les plugins de confiance à des cibles canoniques spécifiques `provider/model`, ou `"*"` pour autoriser explicitement toute cible.
- Les exécutions de sous-agent depuis des plugins non fiables fonctionnent toujours, mais les demandes de surcharge sont rejetées au lieu de revenir silencieusement au repli.

Pour la recherche web, les plugins peuvent consommer le helper de runtime partagé au lieu de
faire des accès directs au câblage de l’outil d’agent :

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Les plugins peuvent aussi enregistrer des fournisseurs de recherche web via
`api.registerWebSearchProvider(...)`.

Remarques :

- Conservez dans le core la sélection du fournisseur, la résolution des identifiants et les sémantiques partagées de requête.
- Utilisez les fournisseurs de recherche web pour les transports de recherche spécifiques à un fournisseur.
- `api.runtime.webSearch.*` est la surface partagée préférée pour les plugins de fonctionnalité/canal qui ont besoin d’un comportement de recherche sans dépendre du wrapper d’outil d’agent.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)` : génère une image à l’aide de la chaîne configurée de fournisseurs de génération d’images.
- `listProviders(...)` : liste les fournisseurs disponibles de génération d’images et leurs capacités.

## Routes HTTP de Gateway

Les plugins peuvent exposer des endpoints HTTP avec `api.registerHttpRoute(...)`.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Champs de route :

- `path` : chemin de route sous le serveur HTTP de la gateway.
- `auth` : requis. Utilisez `"gateway"` pour exiger l’authentification normale de la gateway, ou `"plugin"` pour une authentification/validation de webhook gérée par le plugin.
- `match` : facultatif. `"exact"` (par défaut) ou `"prefix"`.
- `replaceExisting` : facultatif. Permet au même plugin de remplacer son propre enregistrement de route existant.
- `handler` : renvoie `true` lorsque la route a traité la requête.

Remarques :

- `api.registerHttpHandler(...)` a été supprimé et provoquera une erreur de chargement du plugin. Utilisez `api.registerHttpRoute(...)` à la place.
- Les routes de plugin doivent déclarer `auth` explicitement.
- Les conflits exacts `path + match` sont rejetés sauf si `replaceExisting: true`, et un plugin ne peut pas remplacer la route d’un autre plugin.
- Les routes qui se chevauchent avec des niveaux `auth` différents sont rejetées. Gardez les chaînes de retombée `exact`/`prefix` au même niveau d’authentification uniquement.
- Les routes `auth: "plugin"` ne reçoivent **pas** automatiquement les portées de runtime opérateur. Elles servent aux webhooks gérés par le plugin / à la vérification de signature, pas aux appels privilégiés de helpers Gateway.
- Les routes `auth: "gateway"` s’exécutent dans une portée de runtime de requête Gateway, mais cette portée est volontairement prudente :
  - l’authentification bearer par secret partagé (`gateway.auth.mode = "token"` / `"password"`) maintient les portées de runtime des routes de plugin fixées à `operator.write`, même si l’appelant envoie `x-openclaw-scopes`
  - les modes HTTP de confiance porteurs d’identité (par exemple `trusted-proxy` ou `gateway.auth.mode = "none"` sur une entrée privée) n’honorent `x-openclaw-scopes` que lorsque l’en-tête est explicitement présent
  - si `x-openclaw-scopes` est absent sur ces requêtes de route de plugin porteuses d’identité, la portée de runtime revient à `operator.write`
- Règle pratique : ne supposez pas qu’une route de plugin avec auth gateway est implicitement une surface admin. Si votre route a besoin d’un comportement réservé à l’admin, exigez un mode d’authentification porteur d’identité et documentez le contrat explicite de l’en-tête `x-openclaw-scopes`.

## Chemins d’import du Plugin SDK

Utilisez les sous-routes SDK au lieu de l’import monolithique `openclaw/plugin-sdk`
lorsque vous écrivez des plugins :

- `openclaw/plugin-sdk/plugin-entry` pour les primitives d’enregistrement de plugin.
- `openclaw/plugin-sdk/core` pour le contrat générique partagé côté plugin.
- `openclaw/plugin-sdk/config-schema` pour l’export du schéma Zod racine `openclaw.json`
  (`OpenClawSchema`).
- Primitives stables de canal telles que `openclaw/plugin-sdk/channel-setup`,
  `openclaw/plugin-sdk/setup-runtime`,
  `openclaw/plugin-sdk/setup-adapter-runtime`,
  `openclaw/plugin-sdk/setup-tools`,
  `openclaw/plugin-sdk/channel-pairing`,
  `openclaw/plugin-sdk/channel-contract`,
  `openclaw/plugin-sdk/channel-feedback`,
  `openclaw/plugin-sdk/channel-inbound`,
  `openclaw/plugin-sdk/channel-lifecycle`,
  `openclaw/plugin-sdk/channel-reply-pipeline`,
  `openclaw/plugin-sdk/command-auth`,
  `openclaw/plugin-sdk/secret-input`, et
  `openclaw/plugin-sdk/webhook-ingress` pour le câblage partagé setup/auth/réponse/webhook.
  `channel-inbound` est la maison partagée pour l’anti-rebond, la correspondance de mentions,
  les helpers de stratégie de mentions entrantes, le formatage d’enveloppe et les helpers
  de contexte d’enveloppe entrante.
  `channel-setup` est le seam étroit de setup à installation facultative.
  `setup-runtime` est la surface de setup sûre au runtime utilisée par `setupEntry` /
  le démarrage différé, y compris les adaptateurs de patch de setup sûrs à l’import.
  `setup-adapter-runtime` est le seam d’adaptateur de setup de compte sensible à l’environnement.
  `setup-tools` est le petit seam de helpers CLI/archive/docs (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Sous-routes de domaine telles que `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-gateway-runtime`,
  `openclaw/plugin-sdk/approval-handler-adapter-runtime`,
  `openclaw/plugin-sdk/approval-handler-runtime`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store`, et
  `openclaw/plugin-sdk/directory-runtime` pour les helpers partagés de runtime/configuration.
  `telegram-command-config` est le seam public étroit pour la normalisation/validation
  de commandes personnalisées Telegram et reste disponible même si la surface de contrat Telegram intégrée est temporairement indisponible.
  `text-runtime` est le seam partagé texte/Markdown/journalisation, y compris la
  suppression du texte visible par l’assistant, les helpers de rendu/fragmentation Markdown, les helpers de masquage,
  les helpers de balises de directives et les utilitaires safe-text.
- Les seams de canal spécifiques à l’approbation doivent préférer un unique contrat `approvalCapability`
  sur le plugin. Le core lit ensuite l’authentification, la livraison, le rendu,
  le routage natif et le comportement du gestionnaire natif paresseux à travers cette seule
  capacité au lieu de mélanger le comportement d’approbation dans des champs de plugin sans lien.
- `openclaw/plugin-sdk/channel-runtime` est obsolète et ne reste que comme shim de
  compatibilité pour les anciens plugins. Le nouveau code doit importer des primitives génériques plus étroites à la place, et le code du dépôt ne doit pas ajouter de nouveaux imports du shim.
- Les internes des extensions intégrées restent privées. Les plugins externes doivent utiliser uniquement les sous-routes `openclaw/plugin-sdk/*`. Le code core/test d’OpenClaw peut utiliser les points d’entrée publics du dépôt sous une racine de package de plugin tels que `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js`, et des fichiers à portée étroite tels que
  `login-qr-api.js`. N’importez jamais le `src/*` d’un package de plugin depuis le core ou depuis une autre extension.
- Séparation des points d’entrée du dépôt :
  `<plugin-package-root>/api.js` est le barrel de helpers/types,
  `<plugin-package-root>/runtime-api.js` est le barrel runtime-only,
  `<plugin-package-root>/index.js` est le point d’entrée du plugin intégré,
  et `<plugin-package-root>/setup-entry.js` est le point d’entrée du plugin de setup.
- Exemples actuels de fournisseurs intégrés :
  - Anthropic utilise `api.js` / `contract-api.js` pour les helpers de stream Claude tels que
    `wrapAnthropicProviderStream`, les helpers d’en-têtes bêta, et l’analyse de `service_tier`.
  - OpenAI utilise `api.js` pour les builders de fournisseur, les helpers de modèles par défaut, et les builders de fournisseurs temps réel.
  - OpenRouter utilise `api.js` pour son builder de fournisseur ainsi que pour les
    helpers d’onboarding/configuration, tandis que `register.runtime.js` peut encore réexporter les helpers génériques `plugin-sdk/provider-stream` pour un usage local au dépôt.
- Les points d’entrée publics chargés via façade préfèrent le snapshot de configuration runtime actif
  lorsqu’il existe, puis reviennent au fichier de configuration résolu sur disque lorsqu’OpenClaw
  ne sert pas encore de snapshot runtime.
- Les primitives partagées génériques restent le contrat public préféré du SDK. Un petit ensemble réservé
  de compatibilité de seams de helpers de canal marqués existe encore. Traitez-les comme
  des seams de maintenance/compatibilité pour les plugins intégrés, pas comme de nouvelles cibles d’import pour des tiers ; les nouveaux contrats inter-canaux doivent toujours atterrir sur des sous-routes génériques `plugin-sdk/*` ou sur les barrels locaux du plugin `api.js` /
  `runtime-api.js`.

Note de compatibilité :

- Évitez le barrel racine `openclaw/plugin-sdk` pour le nouveau code.
- Préférez d’abord les primitives stables étroites. Les nouvelles sous-routes setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool sont le contrat visé pour le nouveau travail sur les plugins
  intégrés et externes.
  L’analyse/la correspondance des cibles appartient à `openclaw/plugin-sdk/channel-targets`.
  Les portes d’actions de message et les helpers d’id de message de réaction appartiennent à
  `openclaw/plugin-sdk/channel-actions`.
- Les barrels de helpers spécifiques aux extensions intégrées ne sont pas stables par défaut. Si un
  helper n’est nécessaire que pour une extension intégrée, gardez-le derrière le seam local
  `api.js` ou `runtime-api.js` de l’extension au lieu de le promouvoir dans
  `openclaw/plugin-sdk/<extension>`.
- Les nouveaux seams de helpers partagés doivent être génériques, et non marqués par canal. L’analyse
  partagée des cibles appartient à `openclaw/plugin-sdk/channel-targets` ; les
  internes spécifiques à un canal restent derrière le seam local `api.js` ou `runtime-api.js`
  du plugin propriétaire.
- Des sous-routes spécifiques à une capacité telles que `image-generation`,
  `media-understanding` et `speech` existent parce que les plugins intégrés/natifs les utilisent aujourd’hui.
  Leur présence ne signifie pas à elle seule que chaque helper exporté constitue un
  contrat externe figé à long terme.

## Schémas d’outil de message

Les plugins doivent posséder les contributions de schéma `describeMessageTool(...)` spécifiques au canal.
Conservez dans le plugin les champs spécifiques au fournisseur, et non dans le core partagé.

Pour les fragments de schéma partagés portables, réutilisez les helpers génériques exportés via
`openclaw/plugin-sdk/channel-actions` :

- `createMessageToolButtonsSchema()` pour les payloads de type grille de boutons
- `createMessageToolCardSchema()` pour les payloads de carte structurée

Si une forme de schéma n’a de sens que pour un seul fournisseur, définissez-la dans les propres
sources de ce plugin au lieu de la promouvoir dans le SDK partagé.

## Résolution de cible de canal

Les plugins de canal doivent posséder les sémantiques de cible spécifiques au canal. Gardez l’hôte
sortant partagé générique et utilisez la surface d’adaptateur de messagerie pour les règles du fournisseur :

- `messaging.inferTargetChatType({ to })` décide si une cible normalisée
  doit être traitée comme `direct`, `group` ou `channel` avant la recherche dans l’annuaire.
- `messaging.targetResolver.looksLikeId(raw, normalized)` indique au core si une
  entrée doit passer directement à une résolution de type id plutôt qu’à une recherche dans l’annuaire.
- `messaging.targetResolver.resolveTarget(...)` est le repli du plugin lorsque le
  core a besoin d’une résolution finale possédée par le fournisseur après normalisation ou après un échec de l’annuaire.
- `messaging.resolveOutboundSessionRoute(...)` possède la construction de route de session sortante spécifique au fournisseur une fois une cible résolue.

Répartition recommandée :

- Utilisez `inferTargetChatType` pour les décisions de catégorie qui doivent se produire avant
  la recherche de pairs/groupes.
- Utilisez `looksLikeId` pour les vérifications « traitez ceci comme un id de cible explicite/natif ».
- Utilisez `resolveTarget` pour le repli de normalisation spécifique au fournisseur, et non pour une recherche large dans l’annuaire.
- Conservez les ids natifs du fournisseur tels que chat ids, thread ids, JIDs, handles et room ids dans les valeurs `target` ou dans des paramètres spécifiques au fournisseur, pas dans des champs génériques du SDK.

## Annuaires adossés à la configuration

Les plugins qui dérivent des entrées d’annuaire de la configuration doivent conserver cette logique dans le
plugin et réutiliser les helpers partagés de
`openclaw/plugin-sdk/directory-runtime`.

Utilisez cela lorsqu’un canal a besoin de pairs/groupes adossés à la configuration tels que :

- pairs de MP pilotés par liste d’autorisation
- cartes configurées de canaux/groupes
- replis statiques d’annuaire à portée de compte

Les helpers partagés dans `directory-runtime` ne gèrent que les opérations génériques :

- filtrage des requêtes
- application des limites
- helpers de déduplication/normalisation
- construction de `ChannelDirectoryEntry[]`

L’inspection de compte spécifique au canal et la normalisation des ids doivent rester dans l’implémentation du plugin.

## Catalogues de fournisseurs

Les plugins fournisseurs peuvent définir des catalogues de modèles pour l’inférence avec
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` renvoie la même forme que celle écrite par OpenClaw dans
`models.providers` :

- `{ provider }` pour une entrée de fournisseur
- `{ providers }` pour plusieurs entrées de fournisseurs

Utilisez `catalog` lorsque le plugin possède des ids de modèle spécifiques au fournisseur, des valeurs
par défaut de base URL ou des métadonnées de modèles protégées par authentification.

`catalog.order` contrôle quand le catalogue d’un plugin fusionne par rapport aux fournisseurs implicites intégrés d’OpenClaw :

- `simple` : fournisseurs simples à clé API ou pilotés par l’environnement
- `profile` : fournisseurs qui apparaissent lorsque des profils d’authentification existent
- `paired` : fournisseurs qui synthétisent plusieurs entrées de fournisseurs liées
- `late` : dernier passage, après les autres fournisseurs implicites

Les fournisseurs plus tardifs gagnent en cas de collision de clé, afin que les plugins puissent intentionnellement remplacer une entrée de fournisseur intégrée avec le même id de fournisseur.

Compatibilité :

- `discovery` fonctionne toujours comme alias legacy
- si `catalog` et `discovery` sont tous deux enregistrés, OpenClaw utilise `catalog`

## Inspection en lecture seule de canal

Si votre plugin enregistre un canal, préférez implémenter
`plugin.config.inspectAccount(cfg, accountId)` en plus de `resolveAccount(...)`.

Pourquoi :

- `resolveAccount(...)` est le chemin de runtime. Il peut supposer que les identifiants
  sont entièrement matérialisés et peut échouer rapidement lorsque des secrets requis manquent.
- Les chemins de commandes en lecture seule tels que `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve`, et les flux doctor/config
  de réparation ne devraient pas avoir besoin de matérialiser des identifiants de runtime juste pour
  décrire la configuration.

Comportement recommandé de `inspectAccount(...)` :

- Renvoyer uniquement un état descriptif du compte.
- Préserver `enabled` et `configured`.
- Inclure les champs de source/statut d’identifiants lorsque c’est pertinent, tels que :
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Il n’est pas nécessaire de renvoyer les valeurs brutes des jetons pour simplement signaler
  une disponibilité en lecture seule. Renvoyer `tokenStatus: "available"` (et le champ de source correspondant)
  suffit pour les commandes de type status.
- Utilisez `configured_unavailable` lorsqu’un identifiant est configuré via SecretRef mais
  indisponible dans le chemin de commande actuel.

Cela permet aux commandes en lecture seule de signaler « configuré mais indisponible dans ce chemin de commande » au lieu de planter ou de signaler à tort le compte comme non configuré.

## Packs de packages

Un répertoire de plugin peut inclure un `package.json` avec `openclaw.extensions` :

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Chaque entrée devient un plugin. Si le pack liste plusieurs extensions, l’id du plugin
devient `name/<fileBase>`.

Si votre plugin importe des dépendances npm, installez-les dans ce répertoire afin que
`node_modules` soit disponible (`npm install` / `pnpm install`).

Garde-fou de sécurité : chaque entrée `openclaw.extensions` doit rester dans le répertoire du plugin
après résolution des liens symboliques. Les entrées qui s’échappent du répertoire du package sont
rejetées.

Note de sécurité : `openclaw plugins install` installe les dépendances du plugin avec
`npm install --omit=dev --ignore-scripts` (pas de scripts de cycle de vie, pas de dépendances de développement au runtime). Gardez les arbres de dépendances du plugin en
« pur JS/TS » et évitez les packages qui nécessitent des builds `postinstall`.

Facultatif : `openclaw.setupEntry` peut pointer vers un module léger réservé au setup.
Lorsque OpenClaw a besoin de surfaces de setup pour un plugin de canal désactivé, ou
lorsqu’un plugin de canal est activé mais encore non configuré, il charge `setupEntry`
au lieu du point d’entrée complet du plugin. Cela garde le démarrage et le setup plus légers
lorsque le point d’entrée principal du plugin câble aussi des outils, hooks ou autre code réservé au runtime.

Facultatif : `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
peut faire opter un plugin de canal pour le même chemin `setupEntry` pendant la phase
de démarrage pre-listen de la gateway, même lorsque le canal est déjà configuré.

Utilisez cela uniquement si `setupEntry` couvre entièrement la surface de démarrage qui doit exister
avant que la gateway ne commence à écouter. En pratique, cela signifie que l’entrée de setup
doit enregistrer chaque capacité possédée par le canal dont le démarrage dépend, telle que :

- l’enregistrement du canal lui-même
- toutes les routes HTTP qui doivent être disponibles avant que la gateway ne commence à écouter
- toutes les méthodes, outils ou services Gateway qui doivent exister pendant cette même fenêtre

Si votre entrée complète possède encore une capacité de démarrage requise, n’activez pas
ce drapeau. Conservez le comportement par défaut du plugin et laissez OpenClaw charger l’entrée complète au démarrage.

Les canaux intégrés peuvent aussi publier des helpers de surface de contrat réservés au setup
que le core peut consulter avant le chargement du runtime complet du canal. La surface
actuelle de promotion de setup est :

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Le core utilise cette surface lorsqu’il doit promouvoir une configuration legacy de canal à compte unique
vers `channels.<id>.accounts.*` sans charger l’entrée complète du plugin.
Matrix est l’exemple intégré actuel : il ne déplace que les clés auth/bootstrap vers un
compte promu nommé lorsque des comptes nommés existent déjà, et il peut préserver une
clé de compte par défaut non canonique configurée au lieu de toujours créer
`accounts.default`.

Ces adaptateurs de patch de setup gardent paresseuse la découverte de la surface de contrat intégrée.
Le temps d’import reste léger ; la surface de promotion n’est chargée qu’au premier usage au lieu de
rentrer à nouveau dans le démarrage du canal intégré lors de l’import de module.

Lorsque ces surfaces de démarrage incluent des méthodes RPC Gateway, conservez-les sur un
préfixe spécifique au plugin. Les espaces de noms admin du core (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) restent réservés et se résolvent toujours
en `operator.admin`, même si un plugin demande une portée plus étroite.

Exemple :

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Métadonnées de catalogue de canal

Les plugins de canal peuvent annoncer des métadonnées de setup/découverte via `openclaw.channel` et
des indices d’installation via `openclaw.install`. Cela évite au core de contenir des données de catalogue.

Exemple :

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Champs `openclaw.channel` utiles au-delà de l’exemple minimal :

- `detailLabel` : libellé secondaire pour des surfaces plus riches de catalogue/statut
- `docsLabel` : remplace le texte du lien docs
- `preferOver` : ids de plugin/canal moins prioritaires que cette entrée de catalogue doit dépasser
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras` : contrôles de copie de la surface de sélection
- `markdownCapable` : marque le canal comme compatible Markdown pour les décisions de formatage sortant
- `exposure.configured` : masque le canal des surfaces listant les canaux configurés lorsqu’il vaut `false`
- `exposure.setup` : masque le canal des sélecteurs interactifs de setup/configuration lorsqu’il vaut `false`
- `exposure.docs` : marque le canal comme interne/privé pour les surfaces de navigation docs
- `showConfigured` / `showInSetup` : alias legacy encore acceptés pour compatibilité ; préférez `exposure`
- `quickstartAllowFrom` : fait opter le canal dans le flux standard quickstart `allowFrom`
- `forceAccountBinding` : exige une liaison explicite de compte même lorsqu’un seul compte existe
- `preferSessionLookupForAnnounceTarget` : préfère la recherche de session lors de la résolution des cibles d’annonce

OpenClaw peut aussi fusionner des **catalogues de canaux externes** (par exemple un export de registre
MPM). Déposez un fichier JSON dans l’un de ces emplacements :

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Ou pointez `OPENCLAW_PLUGIN_CATALOG_PATHS` (ou `OPENCLAW_MPM_CATALOG_PATHS`) vers
un ou plusieurs fichiers JSON (délimités par virgules/points-virgules/`PATH`). Chaque fichier doit
contenir `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. L’analyseur accepte aussi `"packages"` ou `"plugins"` comme alias legacy de la clé `"entries"`.

## Plugins de moteur de contexte

Les plugins de moteur de contexte possèdent l’orchestration du contexte de session pour l’ingestion, l’assemblage
et la compaction. Enregistrez-les depuis votre plugin avec
`api.registerContextEngine(id, factory)`, puis sélectionnez le moteur actif avec
`plugins.slots.contextEngine`.

Utilisez cela lorsque votre plugin doit remplacer ou étendre le pipeline de contexte par défaut
au lieu de simplement ajouter une recherche mémoire ou des hooks.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Si votre moteur ne possède **pas** l’algorithme de compaction, gardez `compact()`
implémenté et déléguez-le explicitement :

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Ajouter une nouvelle capacité

Lorsqu’un plugin a besoin d’un comportement qui ne correspond pas à l’API actuelle, ne contournez pas
le système de plugins avec un accès privé direct. Ajoutez la capacité manquante.

Séquence recommandée :

1. définir le contrat du core
   Décidez quel comportement partagé le core doit posséder : stratégie, repli, fusion de configuration,
   cycle de vie, sémantiques côté canal et forme du helper de runtime.
2. ajouter des surfaces typées d’enregistrement/runtime de plugin
   Étendez `OpenClawPluginApi` et/ou `api.runtime` avec la plus petite surface de capacité typée utile.
3. raccorder les consommateurs du core + canal/fonctionnalité
   Les canaux et plugins de fonctionnalité doivent consommer la nouvelle capacité via le core,
   pas en important directement une implémentation fournisseur.
4. enregistrer les implémentations fournisseurs
   Les plugins fournisseurs enregistrent ensuite leurs backends sur cette capacité.
5. ajouter une couverture de contrat
   Ajoutez des tests afin que la propriété et la forme de l’enregistrement restent explicites dans le temps.

C’est ainsi qu’OpenClaw reste opiné sans devenir codé en dur pour la vision du monde
d’un seul fournisseur. Voir le [Capability Cookbook](/fr/plugins/architecture)
pour une checklist concrète de fichiers et un exemple détaillé.

### Checklist de capacité

Lorsque vous ajoutez une nouvelle capacité, l’implémentation doit généralement toucher ensemble
ces surfaces :

- types de contrat du core dans `src/<capability>/types.ts`
- runner/helper de runtime du core dans `src/<capability>/runtime.ts`
- surface d’enregistrement dans l’API des plugins dans `src/plugins/types.ts`
- câblage du registre des plugins dans `src/plugins/registry.ts`
- exposition du runtime des plugins dans `src/plugins/runtime/*` lorsque les plugins de fonctionnalité/canal
  doivent la consommer
- helpers de capture/test dans `src/test-utils/plugin-registration.ts`
- assertions de propriété/contrat dans `src/plugins/contracts/registry.ts`
- documentation opérateur/plugin dans `docs/`

Si l’une de ces surfaces manque, c’est généralement le signe que la capacité n’est pas encore
entièrement intégrée.

### Modèle de capacité

Modèle minimal :

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Modèle de test de contrat :

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Cela garde une règle simple :

- le core possède le contrat de capacité + l’orchestration
- les plugins fournisseurs possèdent les implémentations fournisseurs
- les plugins de fonctionnalité/canal consomment les helpers de runtime
- les tests de contrat gardent la propriété explicite
