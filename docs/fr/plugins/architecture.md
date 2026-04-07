---
read_when:
    - Création ou débogage de plugins OpenClaw natifs
    - Comprendre le modèle de capacités des plugins ou les limites de propriété
    - Travail sur le pipeline de chargement des plugins ou le registre
    - Implémentation de hooks d’exécution de fournisseur ou de plugins de canal
sidebarTitle: Internals
summary: 'Éléments internes des plugins : modèle de capacités, propriété, contrats, pipeline de chargement et assistants d’exécution'
title: Éléments internes des plugins
x-i18n:
    generated_at: "2026-04-07T06:55:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9c4b0602df12965a29881eab33b0885f991aeefa2a3fdf3cefc1a7770d6dabe0
    source_path: plugins/architecture.md
    workflow: 15
---

# Éléments internes des plugins

<Info>
  Il s’agit de la **référence d’architecture approfondie**. Pour des guides pratiques, voir :
  - [Installer et utiliser des plugins](/fr/tools/plugin) — guide utilisateur
  - [Premiers pas](/fr/plugins/building-plugins) — premier tutoriel de plugin
  - [Plugins de canal](/fr/plugins/sdk-channel-plugins) — créer un canal de messagerie
  - [Plugins de fournisseur](/fr/plugins/sdk-provider-plugins) — créer un fournisseur de modèles
  - [Vue d’ensemble du SDK](/fr/plugins/sdk-overview) — table d’importation et API d’enregistrement
</Info>

Cette page couvre l’architecture interne du système de plugins d’OpenClaw.

## Modèle de capacités public

Les capacités constituent le modèle public de **plugin natif** dans OpenClaw. Chaque
plugin OpenClaw natif s’enregistre auprès d’un ou plusieurs types de capacités :

| Capability             | Registration method                              | Example plugins                      |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| Inférence de texte     | `api.registerProvider(...)`                      | `openai`, `anthropic`                |
| Backend d’inférence CLI | `api.registerCliBackend(...)`                    | `openai`, `anthropic`                |
| Parole                 | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`            |
| Transcription en temps réel | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| Voix en temps réel     | `api.registerRealtimeVoiceProvider(...)`         | `openai`                             |
| Compréhension multimédia | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                   |
| Génération d’images    | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Génération musicale    | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                  |
| Génération vidéo       | `api.registerVideoGenerationProvider(...)`       | `qwen`                               |
| Récupération Web       | `api.registerWebFetchProvider(...)`              | `firecrawl`                          |
| Recherche Web          | `api.registerWebSearchProvider(...)`             | `google`                             |
| Canal / messagerie     | `api.registerChannel(...)`                       | `msteams`, `matrix`                  |

Un plugin qui n’enregistre aucune capacité mais fournit des hooks, outils ou
services est un plugin **hérité à hooks uniquement**. Ce modèle reste entièrement pris en charge.

### Position actuelle sur la compatibilité externe

Le modèle de capacités est intégré au cœur et utilisé aujourd’hui par les plugins groupés/natifs,
mais la compatibilité des plugins externes nécessite encore une exigence plus stricte que « c’est exporté, donc c’est figé ».

Consignes actuelles :

- **plugins externes existants :** maintenir le fonctionnement des intégrations basées sur des hooks ; traiter
  cela comme la base de compatibilité
- **nouveaux plugins groupés/natifs :** préférer un enregistrement explicite des capacités aux
  accès spécifiques à un fournisseur ou aux nouveaux designs à hooks uniquement
- **plugins externes adoptant l’enregistrement de capacités :** autorisé, mais considérer les surfaces d’assistance
  spécifiques aux capacités comme évolutives tant que la documentation ne marque pas explicitement un contrat comme stable

Règle pratique :

- les API d’enregistrement de capacités constituent la direction voulue
- les hooks hérités restent le chemin le plus sûr pour éviter les ruptures pour les plugins externes pendant
  la transition
- toutes les sous-voies d’assistance exportées ne se valent pas ; préférez le contrat étroit documenté,
  pas des exports d’assistance accessoires

### Formes de plugins

OpenClaw classe chaque plugin chargé dans une forme selon son comportement réel
d’enregistrement (et pas seulement des métadonnées statiques) :

- **plain-capability** -- enregistre exactement un type de capacité (par exemple un
  plugin fournisseur uniquement comme `mistral`)
- **hybrid-capability** -- enregistre plusieurs types de capacités (par exemple
  `openai` possède l’inférence de texte, la parole, la compréhension multimédia et la génération
  d’images)
- **hook-only** -- enregistre uniquement des hooks (typés ou personnalisés), sans capacités,
  outils, commandes ni services
- **non-capability** -- enregistre des outils, commandes, services ou routes mais aucune
  capacité

Utilisez `openclaw plugins inspect <id>` pour voir la forme d’un plugin et la
répartition de ses capacités. Voir [référence CLI](/cli/plugins#inspect) pour plus de détails.

### Hooks hérités

Le hook `before_agent_start` reste pris en charge comme chemin de compatibilité pour
les plugins à hooks uniquement. Des plugins hérités du monde réel en dépendent encore.

Orientation :

- le maintenir fonctionnel
- le documenter comme hérité
- préférer `before_model_resolve` pour le travail de remplacement de modèle/fournisseur
- préférer `before_prompt_build` pour le travail de mutation de prompt
- ne le retirer qu’après une baisse de l’usage réel et après que la couverture par fixtures prouve la sécurité de la migration

### Signaux de compatibilité

Quand vous exécutez `openclaw doctor` ou `openclaw plugins inspect <id>`, vous pouvez voir
l’un de ces libellés :

| Signal                     | Meaning                                                      |
| -------------------------- | ------------------------------------------------------------ |
| **config valide**          | La configuration est analysée correctement et les plugins sont résolus |
| **avis de compatibilité**  | Le plugin utilise un modèle pris en charge mais plus ancien (par ex. `hook-only`) |
| **avertissement hérité**   | Le plugin utilise `before_agent_start`, qui est obsolète     |
| **erreur bloquante**       | La configuration est invalide ou le plugin n’a pas pu être chargé |

Ni `hook-only` ni `before_agent_start` ne casseront votre plugin aujourd’hui --
`hook-only` est un avis, et `before_agent_start` ne déclenche qu’un avertissement. Ces
signaux apparaissent aussi dans `openclaw status --all` et `openclaw plugins doctor`.

## Vue d’ensemble de l’architecture

Le système de plugins d’OpenClaw comporte quatre couches :

1. **Manifeste + découverte**
   OpenClaw trouve les plugins candidats à partir de chemins configurés, de racines d’espace de travail,
   de racines globales d’extension et d’extensions groupées. La découverte lit d’abord les manifestes natifs
   `openclaw.plugin.json` ainsi que les manifestes de bundle pris en charge.
2. **Activation + validation**
   Le cœur décide si un plugin découvert est activé, désactivé, bloqué ou
   sélectionné pour un emplacement exclusif tel que la mémoire.
3. **Chargement d’exécution**
   Les plugins OpenClaw natifs sont chargés dans le processus via jiti et enregistrent
   leurs capacités dans un registre central. Les bundles compatibles sont normalisés en
   enregistrements de registre sans importer le code d’exécution.
4. **Consommation des surfaces**
   Le reste d’OpenClaw lit le registre pour exposer les outils, canaux, configuration
   de fournisseurs, hooks, routes HTTP, commandes CLI et services.

Pour le CLI des plugins en particulier, la découverte des commandes racines est divisée en deux phases :

- les métadonnées au moment de l’analyse proviennent de `registerCli(..., { descriptors: [...] })`
- le vrai module CLI du plugin peut rester paresseux et s’enregistrer lors du premier appel

Cela permet de garder le code CLI possédé par le plugin à l’intérieur du plugin tout en laissant OpenClaw
réserver les noms de commandes racines avant l’analyse.

La limite de conception importante :

- la découverte + la validation de configuration doivent fonctionner à partir des **métadonnées de manifeste/schéma**
  sans exécuter le code du plugin
- le comportement d’exécution natif provient du chemin `register(api)` du module du plugin

Cette séparation permet à OpenClaw de valider la configuration, d’expliquer les plugins manquants/désactivés et
de construire des indications UI/schéma avant que l’exécution complète ne soit active.

### Plugins de canal et outil de message partagé

Les plugins de canal n’ont pas besoin d’enregistrer un outil séparé d’envoi/édition/réaction pour
les actions de discussion normales. OpenClaw conserve un outil `message` partagé dans le cœur, et
les plugins de canal possèdent la découverte et l’exécution spécifiques au canal derrière celui-ci.

La limite actuelle est la suivante :

- le cœur possède l’hôte partagé de l’outil `message`, le câblage des prompts, la tenue des
  sessions/threads et la répartition de l’exécution
- les plugins de canal possèdent la découverte d’actions limitée, la découverte de capacités et tous les
  fragments de schéma spécifiques au canal
- les plugins de canal possèdent la grammaire de conversation de session spécifique au fournisseur, par exemple
  la manière dont les identifiants de conversation encodent les identifiants de thread ou héritent des conversations parentes
- les plugins de canal exécutent l’action finale via leur adaptateur d’action

Pour les plugins de canal, la surface SDK est
`ChannelMessageActionAdapter.describeMessageTool(...)`. Cet appel de découverte unifié
permet à un plugin de renvoyer ensemble ses actions visibles, capacités et contributions de schéma
afin que ces éléments ne divergent pas.

Le cœur transmet la portée d’exécution dans cette étape de découverte. Les champs importants incluent :

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- `requesterSenderId` entrant approuvé

C’est important pour les plugins sensibles au contexte. Un canal peut masquer ou exposer des
actions de message selon le compte actif, la salle/le thread/le message actuel, ou
l’identité approuvée du demandeur, sans coder en dur des branches spécifiques au canal dans
l’outil `message` du cœur.

C’est pourquoi les changements de routage de l’embedded runner restent du travail de plugin : le moteur d’exécution est
responsable de transmettre l’identité de discussion/session actuelle à la limite de découverte du plugin
afin que l’outil `message` partagé expose la bonne surface possédée par le canal pour le tour courant.

Pour les assistants d’exécution possédés par le canal, les plugins groupés doivent conserver le runtime
d’exécution dans leurs propres modules d’extension. Le cœur ne possède plus les runtimes
d’action de message Discord, Slack, Telegram ou WhatsApp sous `src/agents/tools`.
Nous ne publions pas de sous-voies séparées `plugin-sdk/*-action-runtime`, et les plugins groupés
doivent importer directement leur propre code d’exécution local depuis leurs
modules possédés par l’extension.

La même limite s’applique en général aux seams SDK nommés selon un fournisseur : le cœur ne doit
pas importer de barrels utilitaires spécifiques à un canal pour Slack, Discord, Signal,
WhatsApp ou des extensions similaires. Si le cœur a besoin d’un comportement, il doit soit consommer le
barrel `api.ts` / `runtime-api.ts` du plugin groupé lui-même, soit promouvoir le besoin
en capacité générique étroite dans le SDK partagé.

Pour les sondages en particulier, il existe deux chemins d’exécution :

- `outbound.sendPoll` est la base partagée pour les canaux qui correspondent au modèle
  commun de sondage
- `actions.handleAction("poll")` est le chemin préféré pour une sémantique de sondage spécifique au canal
  ou des paramètres de sondage supplémentaires

Le cœur diffère désormais l’analyse des sondages partagés jusqu’à ce que la répartition du sondage par plugin refuse
l’action, afin que les gestionnaires de sondage possédés par le plugin puissent accepter des champs de sondage
spécifiques au canal sans être bloqués d’abord par l’analyseur générique de sondage.

Voir [Pipeline de chargement](#load-pipeline) pour la séquence complète de démarrage.

## Modèle de propriété des capacités

OpenClaw traite un plugin natif comme la limite de propriété d’une **entreprise** ou d’une
**fonctionnalité**, et non comme un ensemble disparate d’intégrations sans lien.

Cela signifie :

- un plugin d’entreprise doit généralement posséder toutes les surfaces OpenClaw de cette
  entreprise
- un plugin de fonctionnalité doit généralement posséder toute la surface de la fonctionnalité qu’il introduit
- les canaux doivent consommer les capacités partagées du cœur au lieu de réimplémenter
  un comportement de fournisseur de manière ad hoc

Exemples :

- le plugin groupé `openai` possède le comportement de fournisseur de modèles OpenAI et le comportement OpenAI
  de parole + voix temps réel + compréhension multimédia + génération d’images
- le plugin groupé `elevenlabs` possède le comportement de parole ElevenLabs
- le plugin groupé `microsoft` possède le comportement de parole Microsoft
- le plugin groupé `google` possède le comportement de fournisseur de modèles Google ainsi que le comportement Google
  de compréhension multimédia + génération d’images + recherche Web
- le plugin groupé `firecrawl` possède le comportement de récupération Web Firecrawl
- les plugins groupés `minimax`, `mistral`, `moonshot` et `zai` possèdent leurs
  backends de compréhension multimédia
- le plugin groupé `qwen` possède le comportement de fournisseur de texte Qwen ainsi que le
  comportement de compréhension multimédia et de génération vidéo
- le plugin `voice-call` est un plugin de fonctionnalité : il possède le transport d’appel, les outils,
  le CLI, les routes et le pont de flux multimédia Twilio, mais il consomme les capacités partagées de parole
  ainsi que de transcription temps réel et de voix temps réel au lieu d’importer directement des plugins de fournisseur

L’état final visé est :

- OpenAI vit dans un seul plugin même s’il couvre les modèles texte, la parole, les images et
  de futures vidéos
- un autre fournisseur peut faire la même chose pour sa propre surface
- les canaux ne se soucient pas du plugin fournisseur qui possède le fournisseur ; ils consomment le
  contrat de capacité partagé exposé par le cœur

C’est la distinction clé :

- **plugin** = limite de propriété
- **capacité** = contrat du cœur que plusieurs plugins peuvent implémenter ou consommer

Donc si OpenClaw ajoute un nouveau domaine tel que la vidéo, la première question n’est pas
« quel fournisseur doit coder en dur la gestion vidéo ? ». La première question est « quel est
le contrat de capacité vidéo du cœur ? ». Une fois que ce contrat existe, les plugins fournisseur
peuvent s’y enregistrer et les plugins de canal/fonctionnalité peuvent le consommer.

Si la capacité n’existe pas encore, la bonne approche est généralement :

1. définir la capacité manquante dans le cœur
2. l’exposer via l’API/runtime du plugin de manière typée
3. raccorder les canaux/fonctionnalités à cette capacité
4. laisser les plugins fournisseur enregistrer des implémentations

Cela garde la propriété explicite tout en évitant un comportement du cœur dépendant d’un
fournisseur unique ou d’un chemin de code spécifique à un plugin isolé.

### Superposition des capacités

Utilisez ce modèle mental pour décider où le code doit se trouver :

- **couche de capacité du cœur** : orchestration partagée, politique, repli, règles de fusion de configuration,
  sémantique de livraison et contrats typés
- **couche de plugin fournisseur** : API spécifiques au fournisseur, authentification, catalogues de modèles, synthèse vocale,
  génération d’images, futurs backends vidéo, points de terminaison d’utilisation
- **couche de plugin canal/fonctionnalité** : intégration Slack/Discord/voice-call/etc.
  qui consomme les capacités du cœur et les présente sur une surface

Par exemple, le TTS suit cette forme :

- le cœur possède la politique TTS au moment de la réponse, l’ordre de repli, les préférences et la livraison par canal
- `openai`, `elevenlabs` et `microsoft` possèdent les implémentations de synthèse
- `voice-call` consomme l’assistant d’exécution TTS de téléphonie

Ce même modèle doit être privilégié pour les futures capacités.

### Exemple de plugin d’entreprise à capacités multiples

Un plugin d’entreprise doit sembler cohérent vu de l’extérieur. Si OpenClaw a des
contrats partagés pour les modèles, la parole, la transcription temps réel, la voix temps réel, la compréhension multimédia,
la génération d’images, la génération vidéo, la récupération Web et la recherche Web,
un fournisseur peut posséder toutes ses surfaces au même endroit :

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
      // hooks d’authentification / catalogue de modèles / runtime
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // config vocale du fournisseur — implémente directement l’interface SpeechProviderPlugin
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
        // logique d’identifiants + récupération
      }),
    );
  },
};

export default plugin;
```

Ce qui compte n’est pas le nom exact des assistants. C’est la forme qui compte :

- un plugin possède la surface du fournisseur
- le cœur possède toujours les contrats de capacité
- les canaux et plugins de fonctionnalité consomment `api.runtime.*`, pas le code du fournisseur
- les tests de contrat peuvent vérifier que le plugin a enregistré les capacités qu’il
  affirme posséder

### Exemple de capacité : compréhension vidéo

OpenClaw traite déjà la compréhension d’images/audio/vidéo comme une seule capacité
partagée. Le même modèle de propriété s’y applique :

1. le cœur définit le contrat de compréhension multimédia
2. les plugins fournisseur enregistrent `describeImage`, `transcribeAudio` et
   `describeVideo` selon le cas
3. les canaux et plugins de fonctionnalité consomment le comportement partagé du cœur au lieu de
   se raccorder directement au code du fournisseur

Cela évite d’intégrer dans le cœur les hypothèses vidéo d’un seul fournisseur. Le plugin possède
la surface du fournisseur ; le cœur possède le contrat de capacité et le comportement de repli.

La génération vidéo suit déjà cette même séquence : le cœur possède le
contrat de capacité typé et l’assistant d’exécution, et les plugins fournisseur enregistrent des
implémentations `api.registerVideoGenerationProvider(...)` contre celui-ci.

Besoin d’une checklist de déploiement concrète ? Voir
[Capability Cookbook](/fr/plugins/architecture).

## Contrats et application

La surface API des plugins est volontairement typée et centralisée dans
`OpenClawPluginApi`. Ce contrat définit les points d’enregistrement pris en charge et
les assistants d’exécution sur lesquels un plugin peut s’appuyer.

Pourquoi c’est important :

- les auteurs de plugins disposent d’un standard interne stable
- le cœur peut rejeter les propriétés dupliquées, par exemple deux plugins enregistrant le même
  identifiant de fournisseur
- le démarrage peut exposer des diagnostics exploitables pour les enregistrements mal formés
- les tests de contrat peuvent imposer la propriété des plugins groupés et éviter les dérives silencieuses

Il existe deux couches d’application :

1. **application lors de l’enregistrement à l’exécution**
   Le registre de plugins valide les enregistrements à mesure que les plugins se chargent. Exemples :
   identifiants de fournisseur dupliqués, identifiants de fournisseur de parole dupliqués et
   enregistrements mal formés produisent des diagnostics de plugin plutôt qu’un comportement non défini.
2. **tests de contrat**
   Les plugins groupés sont capturés dans des registres de contrat pendant les tests afin qu’OpenClaw
   puisse affirmer explicitement la propriété. Aujourd’hui, cela est utilisé pour les
   fournisseurs de modèles, fournisseurs de parole, fournisseurs de recherche Web et la propriété
   des enregistrements groupés.

L’effet pratique est qu’OpenClaw sait, dès le départ, quel plugin possède quelle
surface. Cela permet au cœur et aux canaux de se composer sans friction parce que la propriété est
déclarée, typée et testable plutôt qu’implicite.

### Ce qui doit relever d’un contrat

Les bons contrats de plugin sont :

- typés
- petits
- spécifiques à une capacité
- possédés par le cœur
- réutilisables par plusieurs plugins
- consommables par les canaux/fonctionnalités sans connaissance du fournisseur

Les mauvais contrats de plugin sont :

- une politique spécifique à un fournisseur cachée dans le cœur
- des échappatoires ponctuelles de plugin qui contournent le registre
- du code de canal qui accède directement à une implémentation de fournisseur
- des objets d’exécution ad hoc qui ne font pas partie de `OpenClawPluginApi` ni de
  `api.runtime`

En cas de doute, augmentez le niveau d’abstraction : définissez d’abord la capacité, puis
laissez les plugins s’y brancher.

## Modèle d’exécution

Les plugins OpenClaw natifs s’exécutent **dans le processus** avec la Gateway. Ils ne sont pas
sandboxés. Un plugin natif chargé partage la même limite de confiance au niveau du processus que
le code du cœur.

Implications :

- un plugin natif peut enregistrer des outils, gestionnaires réseau, hooks et services
- un bogue dans un plugin natif peut faire planter ou déstabiliser la gateway
- un plugin natif malveillant équivaut à une exécution de code arbitraire à l’intérieur du
  processus OpenClaw

Les bundles compatibles sont plus sûrs par défaut car OpenClaw les traite actuellement
comme des packs de métadonnées/contenu. Dans les versions actuelles, cela signifie surtout des
Skills groupées.

Utilisez des listes d’autorisation et des chemins explicites d’installation/chargement pour les plugins non groupés. Traitez
les plugins d’espace de travail comme du code de développement, et non comme des valeurs par défaut de production.

Pour les noms de packages d’espace de travail groupés, gardez l’identifiant du plugin ancré dans le nom npm :
`@openclaw/<id>` par défaut, ou un suffixe typé approuvé tel que
`-provider`, `-plugin`, `-speech`, `-sandbox` ou `-media-understanding` lorsque
le package expose intentionnellement un rôle de plugin plus étroit.

Remarque importante sur la confiance :

- `plugins.allow` accorde la confiance aux **identifiants de plugin**, pas à la provenance de la source.
- Un plugin d’espace de travail ayant le même identifiant qu’un plugin groupé masque intentionnellement
  la copie groupée lorsque ce plugin d’espace de travail est activé / en liste d’autorisation.
- C’est normal et utile pour le développement local, les tests de correctifs et les correctifs urgents.

## Limite d’export

OpenClaw exporte des capacités, pas des commodités d’implémentation.

Gardez l’enregistrement des capacités public. Supprimez les exports d’assistance hors contrat :

- sous-voies d’assistance spécifiques à un plugin groupé
- sous-voies de plomberie d’exécution non destinées à être une API publique
- assistants de confort spécifiques à un fournisseur
- assistants de configuration/onboarding qui sont des détails d’implémentation

Certaines sous-voies d’assistance de plugins groupés restent encore dans la table d’export générée du SDK
pour des raisons de compatibilité et de maintenance des plugins groupés. Exemples actuels :
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` et plusieurs seams `plugin-sdk/matrix*`. Traitez-les comme
des exports réservés de détail d’implémentation, et non comme le modèle SDK recommandé pour de
nouveaux plugins tiers.

## Pipeline de chargement

Au démarrage, OpenClaw fait grosso modo ceci :

1. découvre les racines de plugins candidates
2. lit les manifestes natifs ou de bundle compatible et les métadonnées de package
3. rejette les candidats dangereux
4. normalise la configuration des plugins (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. décide de l’activation pour chaque candidat
6. charge les modules natifs activés via jiti
7. appelle les hooks natifs `register(api)` (ou `activate(api)` — un alias hérité) et collecte les enregistrements dans le registre des plugins
8. expose le registre aux surfaces de commandes/d’exécution

<Note>
`activate` est un alias hérité de `register` — le chargeur résout celui qui est présent (`def.register ?? def.activate`) et l’appelle au même moment. Tous les plugins groupés utilisent `register` ; préférez `register` pour les nouveaux plugins.
</Note>

Les barrières de sécurité s’appliquent **avant** l’exécution du runtime. Les candidats sont bloqués
lorsque le point d’entrée sort de la racine du plugin, que le chemin est inscriptible par tout le monde, ou que
la propriété du chemin semble suspecte pour les plugins non groupés.

### Comportement manifeste d’abord

Le manifeste est la source de vérité du plan de contrôle. OpenClaw l’utilise pour :

- identifier le plugin
- découvrir les canaux/Skills/schéma de configuration ou capacités de bundle déclarés
- valider `plugins.entries.<id>.config`
- enrichir les libellés/placeholders de la Control UI
- afficher les métadonnées d’installation/catalogue

Pour les plugins natifs, le module runtime est la partie du plan de données. Il enregistre le
comportement réel tel que hooks, outils, commandes ou flux de fournisseur.

### Ce que le chargeur met en cache

OpenClaw conserve de courts caches en processus pour :

- les résultats de découverte
- les données de registre de manifestes
- les registres de plugins chargés

Ces caches réduisent les démarrages saccadés et la surcharge des commandes répétées. Il faut les considérer
comme des caches de performance de courte durée, pas comme de la persistance.

Remarque sur les performances :

- Définissez `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` ou
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1` pour désactiver ces caches.
- Réglez les fenêtres de cache avec `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS` et
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`.

## Modèle de registre

Les plugins chargés ne modifient pas directement des globals arbitraires du cœur. Ils s’enregistrent dans un
registre central de plugins.

Le registre suit :

- les enregistrements de plugin (identité, source, origine, statut, diagnostics)
- les outils
- les hooks hérités et hooks typés
- les canaux
- les fournisseurs
- les gestionnaires RPC gateway
- les routes HTTP
- les enregistreurs CLI
- les services en arrière-plan
- les commandes possédées par le plugin

Les fonctionnalités du cœur lisent ensuite ce registre au lieu de parler directement aux modules des plugins.
Cela maintient un chargement à sens unique :

- module du plugin -> enregistrement dans le registre
- runtime du cœur -> consommation du registre

Cette séparation est importante pour la maintenabilité. Elle signifie que la plupart des surfaces du cœur n’ont
besoin que d’un seul point d’intégration : « lire le registre », et non « traiter chaque module de plugin
comme un cas particulier ».

## Callbacks de liaison de conversation

Les plugins qui lient une conversation peuvent réagir lorsqu’une approbation est résolue.

Utilisez `api.onConversationBindingResolved(...)` pour recevoir un callback après l’approbation ou le refus d’une demande de liaison :

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // Une liaison existe maintenant pour ce plugin + cette conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // La demande a été refusée ; effacer tout état local en attente.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Champs de la charge utile du callback :

- `status` : `"approved"` ou `"denied"`
- `decision` : `"allow-once"`, `"allow-always"` ou `"deny"`
- `binding` : la liaison résolue pour les demandes approuvées
- `request` : le résumé de la demande d’origine, indication de détachement, identifiant de l’expéditeur et
  métadonnées de conversation

Ce callback est uniquement une notification. Il ne change pas qui est autorisé à lier une
conversation, et il s’exécute une fois la gestion de l’approbation du cœur terminée.

## Hooks d’exécution de fournisseur

Les plugins de fournisseur ont désormais deux couches :

- métadonnées de manifeste : `providerAuthEnvVars` pour une recherche économique de l’authentification fournisseur par variable d’environnement
  avant le chargement du runtime, `channelEnvVars` pour une recherche économique de l’environnement/configuration de canal
  avant le chargement du runtime, plus `providerAuthChoices` pour des
  libellés économiques de choix d’onboarding/authentification et les métadonnées d’indicateurs CLI avant le chargement du runtime
- hooks au moment de la configuration : `catalog` / `discovery` hérité plus `applyConfigDefaults`
- hooks d’exécution : `normalizeModelId`, `normalizeTransport`,
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

OpenClaw possède toujours la boucle générique d’agent, le failover, la gestion des transcriptions et la
politique d’outils. Ces hooks constituent la surface d’extension pour le comportement spécifique au fournisseur sans
nécessiter un transport d’inférence entièrement personnalisé.

Utilisez le manifeste `providerAuthEnvVars` lorsque le fournisseur a des identifiants basés sur l’environnement
que les chemins génériques d’authentification/statut/sélecteur de modèle doivent voir sans charger le runtime du plugin.
Utilisez le manifeste `providerAuthChoices` lorsque les
surfaces CLI d’onboarding/de choix d’authentification doivent connaître l’identifiant de choix du fournisseur, les
libellés de groupe et le câblage simple d’authentification à un indicateur sans charger le runtime du fournisseur. Conservez les `envVars`
du runtime du fournisseur pour les indications orientées opérateur telles que les libellés d’onboarding ou les
variables de configuration d’identifiant client / secret client OAuth.

Utilisez le manifeste `channelEnvVars` lorsqu’un canal a une authentification ou une configuration pilotée par l’environnement que
le repli générique d’environnement shell, les vérifications de config/statut ou les invites de configuration doivent voir
sans charger le runtime du canal.

### Ordre des hooks et usage

Pour les plugins de modèle/fournisseur, OpenClaw appelle les hooks dans cet ordre approximatif.
La colonne « Quand l’utiliser » est le guide de décision rapide.

| #   | Hook                              | What it does                                                                                                   | When to use                                                                                                                                 |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Publie la configuration du fournisseur dans `models.providers` pendant la génération de `models.json`                                | Le fournisseur possède un catalogue ou des valeurs par défaut d’URL de base                                                                                                |
| 2   | `applyConfigDefaults`             | Applique des valeurs globales de configuration par défaut possédées par le fournisseur pendant la matérialisation de la configuration                                      | Les valeurs par défaut dépendent du mode d’authentification, de l’environnement ou de la sémantique de la famille de modèles du fournisseur                                                                       |
| --  | _(recherche de modèle intégrée)_  | OpenClaw essaie d’abord le chemin normal du registre/catalogue                                                          | _(pas un hook de plugin)_                                                                                                                       |
| 3   | `normalizeModelId`                | Normalise les alias d’identifiant de modèle hérités ou preview avant la recherche                                                     | Le fournisseur possède le nettoyage des alias avant la résolution du modèle canonique                                                                               |
| 4   | `normalizeTransport`              | Normalise `api` / `baseUrl` de la famille du fournisseur avant l’assemblage générique du modèle                                      | Le fournisseur possède le nettoyage du transport pour des identifiants de fournisseur personnalisés dans la même famille de transport                                                        |
| 5   | `normalizeConfig`                 | Normalise `models.providers.<id>` avant la résolution runtime/fournisseur                                           | Le fournisseur a besoin d’un nettoyage de configuration qui doit vivre avec le plugin ; les assistants groupés de la famille Google servent aussi de filet de sécurité pour les entrées de configuration Google prises en charge |
| 6   | `applyNativeStreamingUsageCompat` | Applique des réécritures de compatibilité d’utilisation du streaming natif aux fournisseurs de configuration                                               | Le fournisseur a besoin de correctifs de métadonnées d’utilisation du streaming natif pilotés par le point de terminaison                                                                        |
| 7   | `resolveConfigApiKey`             | Résout l’authentification par marqueur d’environnement pour les fournisseurs de configuration avant le chargement de l’authentification runtime                                       | Le fournisseur possède une résolution de clé API par marqueur d’environnement ; `amazon-bedrock` a aussi ici un résolveur intégré de marqueur d’environnement AWS                |
| 8   | `resolveSyntheticAuth`            | Expose une authentification locale/auto-hébergée ou basée sur la configuration sans conserver de texte en clair                                   | Le fournisseur peut fonctionner avec un marqueur d’identifiant synthétique/local                                                                               |
| 9   | `resolveExternalAuthProfiles`     | Superpose des profils d’authentification externes possédés par le fournisseur ; la `persistence` par défaut est `runtime-only` pour les identifiants possédés par CLI/app | Le fournisseur réutilise des identifiants d’authentification externes sans conserver de jetons de rafraîchissement copiés                                                          |
| 10  | `shouldDeferSyntheticProfileAuth` | Relègue les espaces réservés de profils synthétiques stockés derrière l’authentification basée sur l’environnement/la configuration                                      | Le fournisseur stocke des profils de remplacement synthétiques qui ne doivent pas être prioritaires                                                               |
| 11  | `resolveDynamicModel`             | Repli synchrone pour les identifiants de modèle possédés par le fournisseur qui ne figurent pas encore dans le registre local                                       | Le fournisseur accepte des identifiants de modèle amont arbitraires                                                                                               |
| 12  | `prepareDynamicModel`             | Préparation asynchrone, puis `resolveDynamicModel` s’exécute de nouveau                                                           | Le fournisseur a besoin de métadonnées réseau avant de résoudre des identifiants inconnus                                                                                |
| 13  | `normalizeResolvedModel`          | Réécriture finale avant que l’embedded runner n’utilise le modèle résolu                                               | Le fournisseur a besoin de réécritures de transport mais utilise toujours un transport du cœur                                                                           |
| 14  | `contributeResolvedModelCompat`   | Contribue des indicateurs de compatibilité pour des modèles fournisseur derrière un autre transport compatible                                  | Le fournisseur reconnaît ses propres modèles sur des transports proxy sans reprendre la main sur le fournisseur                                                     |
| 15  | `capabilities`                    | Métadonnées de transcription/outillage possédées par le fournisseur et utilisées par la logique partagée du cœur                                           | Le fournisseur a besoin de particularités de transcription/famille de fournisseur                                                                                            |
| 16  | `normalizeToolSchemas`            | Normalise les schémas d’outils avant que l’embedded runner ne les voie                                                    | Le fournisseur a besoin d’un nettoyage de schéma propre à la famille de transport                                                                                              |
| 17  | `inspectToolSchemas`              | Expose des diagnostics de schéma possédés par le fournisseur après normalisation                                                  | Le fournisseur veut des avertissements de mots-clés sans enseigner au cœur des règles spécifiques au fournisseur                                                               |
| 18  | `resolveReasoningOutputMode`      | Sélectionne le contrat de sortie de raisonnement natif ou balisé                                                              | Le fournisseur a besoin d’un raisonnement/sortie finale balisé plutôt que de champs natifs                                                                       |
| 19  | `prepareExtraParams`              | Normalisation des paramètres de requête avant les enveloppes génériques d’options de flux                                              | Le fournisseur a besoin de paramètres de requête par défaut ou d’un nettoyage de paramètres par fournisseur                                                                         |
| 20  | `createStreamFn`                  | Remplace complètement le chemin normal de flux par un transport personnalisé                                                   | Le fournisseur a besoin d’un protocole filaire personnalisé, pas seulement d’une enveloppe                                                                                   |
| 21  | `wrapStreamFn`                    | Enveloppe de flux après application des enveloppes génériques                                                              | Le fournisseur a besoin d’enveloppes de compatibilité d’en-têtes/corps/modèle de requête sans transport personnalisé                                                        |
| 22  | `resolveTransportTurnState`       | Attache des en-têtes ou métadonnées natives de transport par tour                                                           | Le fournisseur veut que les transports génériques envoient une identité de tour native au fournisseur                                                                     |
| 23  | `resolveWebSocketSessionPolicy`   | Attache des en-têtes WebSocket natifs ou une politique de refroidissement de session                                                    | Le fournisseur veut que les transports WS génériques ajustent les en-têtes de session ou la politique de repli                                                             |
| 24  | `formatApiKey`                    | Formateur de profil d’authentification : le profil stocké devient la chaîne `apiKey` d’exécution                                     | Le fournisseur stocke des métadonnées d’authentification supplémentaires et a besoin d’une forme de jeton d’exécution personnalisée                                                                  |
| 25  | `refreshOAuth`                    | Remplacement du rafraîchissement OAuth pour des points de terminaison de rafraîchissement personnalisés ou une politique d’échec de rafraîchissement                                  | Le fournisseur ne correspond pas aux rafraîchisseurs partagés `pi-ai`                                                                                         |
| 26  | `buildAuthDoctorHint`             | Indication de réparation ajoutée quand le rafraîchissement OAuth échoue                                                                  | Le fournisseur a besoin d’une indication de réparation d’authentification possédée par le fournisseur après l’échec du rafraîchissement                                                                    |
| 27  | `matchesContextOverflowError`     | Détecteur de dépassement de fenêtre de contexte possédé par le fournisseur                                                                 | Le fournisseur a des erreurs brutes de dépassement que les heuristiques génériques rateraient                                                                              |
| 28  | `classifyFailoverReason`          | Classification des raisons de failover possédée par le fournisseur                                                                  | Le fournisseur peut mapper les erreurs brutes API/transport vers limitation de débit/surcharge/etc.                                                                        |
| 29  | `isCacheTtlEligible`              | Politique de cache de prompt pour les fournisseurs proxy/backhaul                                                               | Le fournisseur a besoin d’un filtrage TTL du cache spécifique au proxy                                                                                              |
| 30  | `buildMissingAuthMessage`         | Remplacement du message générique de récupération en cas d’authentification manquante                                                      | Le fournisseur a besoin d’une indication de récupération spécifique au fournisseur pour une authentification manquante                                                                               |
| 31  | `suppressBuiltInModel`            | Suppression de modèle amont obsolète plus indication d’erreur optionnelle orientée utilisateur                                          | Le fournisseur a besoin de masquer des lignes amont obsolètes ou de les remplacer par une indication du fournisseur                                                               |
| 32  | `augmentModelCatalog`             | Lignes synthétiques/finales du catalogue ajoutées après la découverte                                                          | Le fournisseur a besoin de lignes synthétiques de compatibilité future dans `models list` et les sélecteurs                                                                   |
| 33  | `isBinaryThinking`                | Bascule de raisonnement activé/désactivé pour les fournisseurs de réflexion binaire                                                          | Le fournisseur n’expose qu’un raisonnement binaire activé/désactivé                                                                                                |
| 34  | `supportsXHighThinking`           | Prise en charge du raisonnement `xhigh` pour certains modèles                                                                  | Le fournisseur veut `xhigh` seulement sur un sous-ensemble de modèles                                                                                           |
| 35  | `resolveDefaultThinkingLevel`     | Niveau `/think` par défaut pour une famille de modèles spécifique                                                             | Le fournisseur possède la politique `/think` par défaut pour une famille de modèles                                                                                    |
| 36  | `isModernModelRef`                | Correspondance des modèles modernes pour les filtres de profils en direct et la sélection smoke                                              | Le fournisseur possède la correspondance des modèles préférés live/smoke                                                                                           |
| 37  | `prepareRuntimeAuth`              | Échange un identifiant configuré contre le jeton/la clé d’exécution réel juste avant l’inférence                       | Le fournisseur a besoin d’un échange de jeton ou d’un identifiant de requête à courte durée de vie                                                                           |
| 38  | `resolveUsageAuth`                | Résout les identifiants d’utilisation/facturation pour `/usage` et les surfaces de statut associées                                     | Le fournisseur a besoin d’une analyse personnalisée du jeton d’utilisation/quota ou d’un identifiant d’utilisation différent                                                             |
| 39  | `fetchUsageSnapshot`              | Récupère et normalise des instantanés d’utilisation/quota spécifiques au fournisseur une fois l’authentification résolue                             | Le fournisseur a besoin d’un point de terminaison d’utilisation spécifique au fournisseur ou d’un parseur de charge utile                                                                         |
| 40  | `createEmbeddingProvider`         | Construit un adaptateur d’embeddings possédé par le fournisseur pour la mémoire/la recherche                                                     | Le comportement des embeddings mémoire relève du plugin fournisseur                                                                                  |
| 41  | `buildReplayPolicy`               | Renvoie une politique de rejeu contrôlant la gestion de transcription pour le fournisseur                                        | Le fournisseur a besoin d’une politique de transcription personnalisée (par exemple suppression de blocs de réflexion)                                                             |
| 42  | `sanitizeReplayHistory`           | Réécrit l’historique de rejeu après le nettoyage générique de transcription                                                        | Le fournisseur a besoin de réécritures de rejeu spécifiques au fournisseur au-delà des assistants partagés de compaction                                                           |
| 43  | `validateReplayTurns`             | Validation ou remodelage final des tours de rejeu avant l’embedded runner                                           | Le transport du fournisseur a besoin d’une validation plus stricte des tours après assainissement générique                                                                  |
| 44  | `onModelSelected`                 | Exécute des effets secondaires après sélection possédés par le fournisseur                                                                 | Le fournisseur a besoin de télémétrie ou d’un état possédé par le fournisseur lorsqu’un modèle devient actif                                                                |

`normalizeModelId`, `normalizeTransport` et `normalizeConfig` vérifient d’abord le
plugin fournisseur correspondant, puis passent aux autres plugins fournisseurs capables de hooks
jusqu’à ce que l’un change effectivement l’identifiant de modèle ou le transport/la configuration. Cela permet aux
shims d’alias/compatibilité de fonctionner sans obliger l’appelant à savoir quel
plugin groupé possède la réécriture. Si aucun hook de fournisseur ne réécrit une
entrée de configuration Google compatible, le normaliseur de configuration Google groupé applique tout de même
ce nettoyage de compatibilité.

Si le fournisseur a besoin d’un protocole filaire totalement personnalisé ou d’un exécuteur de requêtes personnalisé,
il s’agit d’une autre classe d’extension. Ces hooks concernent un comportement fournisseur
qui s’exécute toujours sur la boucle normale d’inférence d’OpenClaw.

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
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`
  et `wrapStreamFn` parce qu’il possède la compatibilité future de Claude 4.6,
  les indications de famille de fournisseur, les conseils de réparation d’authentification, l’intégration
  du point de terminaison d’utilisation, l’éligibilité au cache de prompt, les valeurs par défaut de configuration sensibles à l’authentification, la politique de réflexion
  par défaut/adaptative de Claude, ainsi que la mise en forme de flux propre à Anthropic pour
  les en-têtes bêta, `/fast` / `serviceTier` et `context1m`.
- Les assistants de flux spécifiques à Claude pour Anthropic restent pour l’instant dans le
  seam public propre `api.ts` / `contract-api.ts` du plugin groupé. Cette surface de package
  exporte `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` et les builders d’enveloppes
  Anthropic de niveau inférieur au lieu d’élargir le SDK générique autour des règles
  d’en-têtes bêta d’un seul fournisseur.
- OpenAI utilise `resolveDynamicModel`, `normalizeResolvedModel` et
  `capabilities` ainsi que `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking` et `isModernModelRef`
  parce qu’il possède la compatibilité future de GPT-5.4, la normalisation directe
  `openai-completions` -> `openai-responses`, les indications d’authentification
  tenant compte de Codex, la suppression de Spark, les lignes synthétiques de liste OpenAI, ainsi que la politique de réflexion GPT-5 /
  de modèles live ; la famille de flux `openai-responses-defaults` possède les
  enveloppes partagées OpenAI Responses natives pour les en-têtes d’attribution,
  `/fast`/`serviceTier`, la verbosité du texte, la recherche Web Codex native,
  la mise en forme de charge utile de compatibilité de raisonnement, et la gestion du contexte Responses.
- OpenRouter utilise `catalog` ainsi que `resolveDynamicModel` et
  `prepareDynamicModel` parce que le fournisseur est pass-through et peut exposer de nouveaux
  identifiants de modèle avant la mise à jour du catalogue statique d’OpenClaw ; il utilise aussi
  `capabilities`, `wrapStreamFn` et `isCacheTtlEligible` pour garder
  hors du cœur les en-têtes de requête spécifiques au fournisseur, les métadonnées de routage, les correctifs de raisonnement et
  la politique de cache de prompt. Sa politique de rejeu provient de la
  famille `passthrough-gemini`, tandis que la famille de flux `openrouter-thinking`
  possède l’injection de raisonnement proxy et les sauts des modèles non pris en charge / `auto`.
- GitHub Copilot utilise `catalog`, `auth`, `resolveDynamicModel` et
  `capabilities` ainsi que `prepareRuntimeAuth` et `fetchUsageSnapshot` parce qu’il
  a besoin d’une connexion d’appareil possédée par le fournisseur, d’un comportement de repli de modèle, de
  particularités de transcription Claude, d’un échange de jeton GitHub -> Copilot, et d’un
  point de terminaison d’utilisation possédé par le fournisseur.
- OpenAI Codex utilise `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth` et `augmentModelCatalog` ainsi que
  `prepareExtraParams`, `resolveUsageAuth` et `fetchUsageSnapshot` parce qu’il
  fonctionne toujours sur les transports OpenAI du cœur mais possède sa normalisation de
  transport/URL de base, sa politique de repli de rafraîchissement OAuth, son choix de transport par défaut,
  ses lignes synthétiques de catalogue Codex et l’intégration du point de terminaison d’utilisation ChatGPT ; il
  partage la même famille de flux `openai-responses-defaults` qu’OpenAI direct.
- Google AI Studio et Gemini CLI OAuth utilisent `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn` et `isModernModelRef` parce que la
  famille de rejeu `google-gemini` possède le repli de compatibilité future Gemini 3.1,
  la validation native du rejeu Gemini, l’assainissement du rejeu d’amorçage, le
  mode de sortie de raisonnement balisé et la correspondance des modèles modernes, tandis que la
  famille de flux `google-thinking` possède la normalisation de charge utile de réflexion Gemini ;
  Gemini CLI OAuth utilise aussi `formatApiKey`, `resolveUsageAuth` et
  `fetchUsageSnapshot` pour le formatage de jeton, l’analyse de jeton et le câblage
  du point de terminaison de quota.
- Anthropic Vertex utilise `buildReplayPolicy` via la
  famille de rejeu `anthropic-by-model` afin que le nettoyage de rejeu spécifique à Claude reste
  limité aux identifiants Claude au lieu de tous les transports `anthropic-messages`.
- Amazon Bedrock utilise `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason` et `resolveDefaultThinkingLevel` parce qu’il possède
  la classification Bedrock spécifique des erreurs de limitation / non-prêt / dépassement de contexte
  pour le trafic Anthropic sur Bedrock ; sa politique de rejeu partage toujours le même
  garde-fou `anthropic-by-model` réservé à Claude.
- OpenRouter, Kilocode, Opencode et Opencode Go utilisent `buildReplayPolicy`
  via la famille de rejeu `passthrough-gemini` parce qu’ils proxifient des modèles Gemini
  via des transports compatibles OpenAI et ont besoin d’un
  assainissement des signatures de réflexion Gemini sans validation native du rejeu Gemini ni
  réécritures d’amorçage.
- MiniMax utilise `buildReplayPolicy` via la
  famille de rejeu `hybrid-anthropic-openai` parce qu’un fournisseur possède à la fois des
  sémantiques de type message Anthropic et compatibles OpenAI ; il conserve la
  suppression des blocs de réflexion réservés à Claude côté Anthropic tout en remplaçant le
  mode de sortie de raisonnement par le mode natif, et la famille de flux `minimax-fast-mode` possède
  les réécritures de modèle fast-mode sur le chemin de flux partagé.
- Moonshot utilise `catalog` plus `wrapStreamFn` parce qu’il utilise toujours le transport partagé
  OpenAI mais a besoin d’une normalisation de charge utile de réflexion possédée par le fournisseur ; la
  famille de flux `moonshot-thinking` mappe la configuration plus l’état `/think` sur sa
  charge utile native binaire de réflexion.
- Kilocode utilise `catalog`, `capabilities`, `wrapStreamFn` et
  `isCacheTtlEligible` parce qu’il a besoin d’en-têtes de requête possédés par le fournisseur,
  de normalisation de charge utile de raisonnement, d’indications de transcription Gemini et de
  filtrage Anthropic du TTL du cache ; la famille de flux `kilocode-thinking` conserve l’injection de réflexion Kilo
  sur le chemin de flux proxy partagé tout en ignorant `kilo/auto` et les
  autres identifiants de modèle proxy qui ne prennent pas en charge les charges utiles explicites de raisonnement.
- Z.AI utilise `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth` et `fetchUsageSnapshot` parce qu’il possède le repli GLM-5,
  les valeurs par défaut `tool_stream`, l’UX de réflexion binaire, la correspondance des modèles modernes et à la fois
  l’authentification d’utilisation + la récupération de quota ; la famille de flux `tool-stream-default-on` garde
  l’enveloppe `tool_stream` activée par défaut hors du câblage manuscrit par fournisseur.
- xAI utilise `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel` et `isModernModelRef`
  parce qu’il possède la normalisation native du transport xAI Responses, les réécritures d’alias fast-mode Grok,
  `tool_stream` par défaut, le nettoyage strict d’outils / charges utiles de raisonnement,
  la réutilisation d’authentification de repli pour les outils possédés par le plugin, la résolution de
  modèle Grok compatible avec l’avenir, et les correctifs de compatibilité possédés par le fournisseur tels que le
  profil de schéma d’outils xAI, les mots-clés de schéma non pris en charge, `web_search` natif et le
  décodage des arguments d’appel d’outil en entités HTML.
- Mistral, OpenCode Zen et OpenCode Go utilisent uniquement `capabilities` pour garder
  hors du cœur les particularités de transcription/outillage.
- Les fournisseurs groupés à catalogue seul comme `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway` et `volcengine` utilisent
  uniquement `catalog`.
- Qwen utilise `catalog` pour son fournisseur de texte ainsi que des enregistrements partagés de compréhension multimédia et
  de génération vidéo pour ses surfaces multimodales.
- MiniMax et Xiaomi utilisent `catalog` plus des hooks d’utilisation parce que leur comportement `/usage`
  est possédé par le plugin même si l’inférence passe toujours par les transports partagés.

## Assistants d’exécution

Les plugins peuvent accéder à certains assistants du cœur via `api.runtime`. Pour le TTS :

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

Remarques :

- `textToSpeech` renvoie la charge utile normale du cœur pour les surfaces de fichier/note vocale.
- Utilise la configuration du cœur `messages.tts` et la sélection de fournisseur.
- Renvoie un tampon audio PCM + fréquence d’échantillonnage. Les plugins doivent rééchantillonner/encoder pour les fournisseurs.
- `listVoices` est facultatif selon le fournisseur. Utilisez-le pour les sélecteurs de voix ou flux de configuration possédés par le fournisseur.
- Les listes de voix peuvent inclure des métadonnées plus riches comme la locale, le genre et des tags de personnalité pour des sélecteurs tenant compte du fournisseur.
- OpenAI et ElevenLabs prennent en charge la téléphonie aujourd’hui. Microsoft non.

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

Remarques :

- Gardez la politique TTS, le repli et la livraison des réponses dans le cœur.
- Utilisez des fournisseurs de parole pour le comportement de synthèse possédé par le fournisseur.
- L’entrée héritée Microsoft `edge` est normalisée vers l’identifiant de fournisseur `microsoft`.
- Le modèle de propriété préféré est orienté entreprise : un plugin fournisseur peut posséder
  du texte, de la parole, de l’image et de futurs fournisseurs multimédias à mesure qu’OpenClaw ajoute ces
  contrats de capacité.

Pour la compréhension image/audio/vidéo, les plugins enregistrent un
fournisseur de compréhension multimédia typé au lieu d’un sac générique clé/valeur :

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Remarques :

- Gardez l’orchestration, le repli, la configuration et le câblage des canaux dans le cœur.
- Gardez le comportement fournisseur dans le plugin fournisseur.
- L’expansion additive doit rester typée : nouvelles méthodes facultatives, nouveaux champs de résultat facultatifs, nouvelles capacités facultatives.
- La génération vidéo suit déjà le même modèle :
  - le cœur possède le contrat de capacité et l’assistant d’exécution
  - les plugins fournisseur enregistrent `api.registerVideoGenerationProvider(...)`
  - les plugins de fonctionnalité/canal consomment `api.runtime.videoGeneration.*`

Pour les assistants d’exécution de compréhension multimédia, les plugins peuvent appeler :

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

Pour la transcription audio, les plugins peuvent utiliser soit le runtime de compréhension multimédia
soit l’ancien alias STT :

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Facultatif quand le MIME ne peut pas être déduit de manière fiable :
  mime: "audio/ogg",
});
```

Remarques :

- `api.runtime.mediaUnderstanding.*` est la surface partagée préférée pour la
  compréhension image/audio/vidéo.
- Utilise la configuration audio de compréhension multimédia du cœur (`tools.media.audio`) et l’ordre de repli des fournisseurs.
- Renvoie `{ text: undefined }` lorsqu’aucune sortie de transcription n’est produite (par exemple entrée ignorée / non prise en charge).
- `api.runtime.stt.transcribeAudioFile(...)` reste disponible comme alias de compatibilité.

Les plugins peuvent aussi lancer des exécutions de sous-agent en arrière-plan via `api.runtime.subagent` :

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Remarques :

- `provider` et `model` sont des remplacements facultatifs par exécution, et non des changements de session persistants.
- OpenClaw n’honore ces champs de remplacement que pour les appelants de confiance.
- Pour les exécutions de repli possédées par un plugin, les opérateurs doivent activer cette possibilité avec `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Utilisez `plugins.entries.<id>.subagent.allowedModels` pour restreindre les plugins de confiance à des cibles canoniques `provider/model` spécifiques, ou `"*"` pour autoriser explicitement toute cible.
- Les exécutions de sous-agent de plugins non fiables fonctionnent toujours, mais les demandes de remplacement sont rejetées au lieu de retomber silencieusement sur une valeur par défaut.

Pour la recherche Web, les plugins peuvent consommer l’assistant d’exécution partagé au lieu
d’accéder au câblage de l’outil de l’agent :

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

Les plugins peuvent aussi enregistrer des fournisseurs de recherche Web via
`api.registerWebSearchProvider(...)`.

Remarques :

- Gardez la sélection du fournisseur, la résolution des identifiants et la sémantique partagée des requêtes dans le cœur.
- Utilisez des fournisseurs de recherche Web pour les transports de recherche spécifiques au fournisseur.
- `api.runtime.webSearch.*` est la surface partagée préférée pour les plugins de fonctionnalité/canal qui ont besoin d’un comportement de recherche sans dépendre de l’enveloppe de l’outil de l’agent.

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

- `generate(...)` : génère une image en utilisant la chaîne configurée de fournisseurs de génération d’images.
- `listProviders(...)` : liste les fournisseurs de génération d’images disponibles et leurs capacités.

## Routes HTTP de la Gateway

Les plugins peuvent exposer des points de terminaison HTTP avec `api.registerHttpRoute(...)`.

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

Champs de route :

- `path` : chemin de route sous le serveur HTTP de la gateway.
- `auth` : requis. Utilisez `"gateway"` pour exiger l’authentification normale de la gateway, ou `"plugin"` pour une authentification/validation de webhook gérée par le plugin.
- `match` : facultatif. `"exact"` (par défaut) ou `"prefix"`.
- `replaceExisting` : facultatif. Autorise le même plugin à remplacer son propre enregistrement de route existant.
- `handler` : renvoie `true` lorsque la route a géré la requête.

Remarques :

- `api.registerHttpHandler(...)` a été supprimé et provoquera une erreur de chargement du plugin. Utilisez `api.registerHttpRoute(...)` à la place.
- Les routes de plugin doivent déclarer explicitement `auth`.
- Les conflits exacts `path + match` sont rejetés sauf si `replaceExisting: true`, et un plugin ne peut pas remplacer la route d’un autre plugin.
- Les routes qui se chevauchent avec des niveaux `auth` différents sont rejetées. Gardez les chaînes de retombée `exact`/`prefix` au même niveau d’authentification uniquement.
- Les routes `auth: "plugin"` ne reçoivent **pas** automatiquement les portées d’exécution opérateur. Elles servent aux webhooks / à la validation de signature gérés par le plugin, pas à des appels d’assistance Gateway privilégiés.
- Les routes `auth: "gateway"` s’exécutent dans une portée d’exécution de requête Gateway, mais cette portée est volontairement prudente :
  - l’authentification bearer par secret partagé (`gateway.auth.mode = "token"` / `"password"`) maintient les portées d’exécution des routes de plugin figées sur `operator.write`, même si l’appelant envoie `x-openclaw-scopes`
  - les modes HTTP de confiance portant une identité (par exemple `trusted-proxy` ou `gateway.auth.mode = "none"` sur une entrée privée) n’honorent `x-openclaw-scopes` que lorsque l’en-tête est explicitement présent
  - si `x-openclaw-scopes` est absent sur ces requêtes de route de plugin portant une identité, la portée d’exécution retombe sur `operator.write`
- Règle pratique : ne supposez pas qu’une route de plugin authentifiée par gateway est implicitement une surface d’administration. Si votre route a besoin d’un comportement réservé à l’administration, exigez un mode d’authentification porteur d’identité et documentez explicitement le contrat de l’en-tête `x-openclaw-scopes`.

## Chemins d’importation du SDK plugin

Utilisez les sous-chemins SDK plutôt que l’import monolithique `openclaw/plugin-sdk` lorsque
vous développez des plugins :

- `openclaw/plugin-sdk/plugin-entry` pour les primitives d’enregistrement de plugin.
- `openclaw/plugin-sdk/core` pour le contrat générique partagé côté plugin.
- `openclaw/plugin-sdk/config-schema` pour l’export du schéma Zod racine `openclaw.json`
  (`OpenClawSchema`).
- Primitives de canal stables telles que `openclaw/plugin-sdk/channel-setup`,
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
  `openclaw/plugin-sdk/secret-input` et
  `openclaw/plugin-sdk/webhook-ingress` pour le câblage partagé
  de configuration/authentification/réponse/webhook. `channel-inbound` est la maison partagée pour le debounce, la mise en correspondance des mentions,
  le formatage des enveloppes et les assistants de contexte des enveloppes entrantes.
  `channel-setup` est le seam étroit de configuration d’installation facultative.
  `setup-runtime` est la surface de configuration sûre à l’exécution utilisée par `setupEntry` /
  le démarrage différé, y compris les adaptateurs de patch de configuration sûrs à l’importation.
  `setup-adapter-runtime` est le seam d’adaptateur de configuration de compte sensible à l’environnement.
  `setup-tools` est le petit seam d’assistance CLI/archive/documentation (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Sous-chemins de domaine tels que `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store` et
  `openclaw/plugin-sdk/directory-runtime` pour les assistants partagés d’exécution/configuration.
  `telegram-command-config` est le seam public étroit pour la
  normalisation/validation des commandes personnalisées Telegram et reste disponible même si la surface de contrat Telegram groupée
  est temporairement indisponible.
  `text-runtime` est le seam partagé texte/markdown/logging, y compris
  la suppression du texte visible par l’assistant, les assistants de rendu/découpage markdown, les assistants de rédaction,
  les assistants de balises de directive et les utilitaires de texte sûr.
- Les seams de canal spécifiques à l’approbation doivent préférer un seul contrat `approvalCapability`
  sur le plugin. Le cœur lit ensuite l’authentification d’approbation, la livraison, le rendu et le
  comportement de routage natif à travers cette seule capacité au lieu de mélanger
  le comportement d’approbation à des champs non liés du plugin.
- `openclaw/plugin-sdk/channel-runtime` est obsolète et ne reste présent que comme
  shim de compatibilité pour les anciens plugins. Le nouveau code doit importer les primitives génériques plus étroites à la place,
  et le code du dépôt ne doit pas ajouter de nouveaux imports du shim.
- Les éléments internes des extensions groupées restent privés. Les plugins externes doivent utiliser uniquement les sous-chemins
  `openclaw/plugin-sdk/*`. Le code du cœur/des tests OpenClaw peut utiliser les
  points d’entrée publics du dépôt sous la racine d’un package de plugin tels que `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js` et des fichiers ciblés étroits tels que
  `login-qr-api.js`. N’importez jamais `src/*` d’un package de plugin depuis le cœur ni depuis
  une autre extension.
- Séparation des points d’entrée du dépôt :
  `<plugin-package-root>/api.js` est le barrel d’assistance/types,
  `<plugin-package-root>/runtime-api.js` est le barrel runtime-only,
  `<plugin-package-root>/index.js` est l’entrée du plugin groupé,
  et `<plugin-package-root>/setup-entry.js` est l’entrée du plugin de configuration.
- Exemples actuels de fournisseurs groupés :
  - Anthropic utilise `api.js` / `contract-api.js` pour les assistants de flux Claude tels que
    `wrapAnthropicProviderStream`, les assistants d’en-têtes bêta et l’analyse de `service_tier`.
  - OpenAI utilise `api.js` pour les builders de fournisseurs, les assistants de modèle par défaut et
    les builders de fournisseurs temps réel.
  - OpenRouter utilise `api.js` pour son builder de fournisseur ainsi que les assistants
    d’onboarding/configuration, tandis que `register.runtime.js` peut toujours réexporter des assistants génériques
    `plugin-sdk/provider-stream` pour une utilisation locale au dépôt.
- Les points d’entrée publics chargés par façade préfèrent l’instantané de configuration d’exécution actif
  lorsqu’il existe, puis retombent sur le fichier de configuration résolu sur disque lorsque
  OpenClaw ne sert pas encore d’instantané d’exécution.
- Les primitives partagées génériques restent le contrat public préféré du SDK. Un petit
  ensemble réservé de compatibilité de seams utilitaires marqués par canal groupé existe encore. Traitez-les comme
  des seams de maintenance/compatibilité des plugins groupés, et non comme de nouvelles cibles d’import tierces ; les
  nouveaux contrats transversaux doivent toujours être placés sur des sous-chemins génériques `plugin-sdk/*` ou sur les barrels locaux `api.js` /
  `runtime-api.js` du plugin.

Remarque de compatibilité :

- Évitez le barrel racine `openclaw/plugin-sdk` pour le nouveau code.
- Préférez d’abord les primitives stables et étroites. Les sous-chemins plus récents de configuration/appairage/réponse/
  feedback/contrat/inbound/threading/commande/secret-input/webhook/infra/
  allowlist/statut/message-tool constituent le contrat visé pour les nouveaux travaux de plugins
  groupés et externes.
  L’analyse/la correspondance des cibles relève de `openclaw/plugin-sdk/channel-targets`.
  Les gardes d’action de message et les assistants d’identifiant de message de réaction relèvent de
  `openclaw/plugin-sdk/channel-actions`.
- Les barrels utilitaires spécifiques à une extension groupée ne sont pas stables par défaut. Si un
  assistant n’est nécessaire que pour une extension groupée, gardez-le derrière le seam local
  `api.js` ou `runtime-api.js` de l’extension au lieu de le promouvoir dans
  `openclaw/plugin-sdk/<extension>`.
- Les nouveaux seams d’assistance partagés doivent être génériques, pas marqués par canal. L’analyse partagée
  des cibles relève de `openclaw/plugin-sdk/channel-targets` ; les éléments internes propres à un canal
  restent derrière le seam local `api.js` ou `runtime-api.js` du plugin propriétaire.
- Les sous-chemins spécifiques aux capacités tels que `image-generation`,
  `media-understanding` et `speech` existent parce que les plugins groupés/natifs les utilisent
  aujourd’hui. Leur présence ne signifie pas à elle seule que chaque assistant exporté constitue un
  contrat externe figé à long terme.

## Schémas de l’outil de message

Les plugins doivent posséder les contributions de schéma `describeMessageTool(...)` spécifiques au canal.
Gardez les champs spécifiques au fournisseur dans le plugin, pas dans le cœur partagé.

Pour les fragments de schéma portables partagés, réutilisez les assistants génériques exportés via
`openclaw/plugin-sdk/channel-actions` :

- `createMessageToolButtonsSchema()` pour les charges utiles de type grille de boutons
- `createMessageToolCardSchema()` pour les charges utiles de type carte structurée

Si une forme de schéma n’a de sens que pour un seul fournisseur, définissez-la dans les
sources de ce plugin au lieu de la promouvoir dans le SDK partagé.

## Résolution des cibles de canal

Les plugins de canal doivent posséder la sémantique de cible spécifique au canal. Gardez l’hôte sortant
partagé générique et utilisez la surface de l’adaptateur de messagerie pour les règles du fournisseur :

- `messaging.inferTargetChatType({ to })` décide si une cible normalisée
  doit être traitée comme `direct`, `group` ou `channel` avant la recherche dans l’annuaire.
- `messaging.targetResolver.looksLikeId(raw, normalized)` indique au cœur si une
  entrée doit passer directement à une résolution de type identifiant au lieu d’une recherche dans l’annuaire.
- `messaging.targetResolver.resolveTarget(...)` est le repli du plugin lorsque le
  cœur a besoin d’une résolution finale possédée par le fournisseur après normalisation ou après un échec de l’annuaire.
- `messaging.resolveOutboundSessionRoute(...)` possède la construction de route de session spécifique au fournisseur une fois la cible résolue.

Répartition recommandée :

- Utilisez `inferTargetChatType` pour les décisions de catégorie qui doivent intervenir avant
  la recherche de pairs/groupes.
- Utilisez `looksLikeId` pour les vérifications « traiter ceci comme un identifiant de cible explicite/natif ».
- Utilisez `resolveTarget` comme repli de normalisation spécifique au fournisseur, pas pour une
  recherche large dans l’annuaire.
- Gardez les identifiants natifs du fournisseur comme identifiants de chat, de thread, JID, handles et identifiants de salle
  dans les valeurs `target` ou dans des paramètres spécifiques au fournisseur, pas dans des champs SDK génériques.

## Annuaires basés sur la configuration

Les plugins qui dérivent des entrées d’annuaire de la configuration doivent conserver cette logique dans le
plugin et réutiliser les assistants partagés de
`openclaw/plugin-sdk/directory-runtime`.

Utilisez ceci lorsqu’un canal a besoin de pairs/groupes basés sur la configuration, tels que :

- pairs DM pilotés par allowlist
- cartes configurées de canaux/groupes
- replis d’annuaire statiques liés au compte

Les assistants partagés de `directory-runtime` ne gèrent que des opérations génériques :

- filtrage des requêtes
- application des limites
- assistants de déduplication/normalisation
- construction de `ChannelDirectoryEntry[]`

L’inspection de compte spécifique au canal et la normalisation des identifiants doivent rester dans
l’implémentation du plugin.

## Catalogues de fournisseurs

Les plugins de fournisseur peuvent définir des catalogues de modèles pour l’inférence avec
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` renvoie la même forme que celle qu’OpenClaw écrit dans
`models.providers` :

- `{ provider }` pour une entrée de fournisseur
- `{ providers }` pour plusieurs entrées de fournisseur

Utilisez `catalog` lorsque le plugin possède des identifiants de modèle spécifiques au fournisseur, des valeurs par
défaut d’URL de base ou des métadonnées de modèle dépendantes de l’authentification.

`catalog.order` contrôle le moment où le catalogue d’un plugin est fusionné par rapport aux
fournisseurs implicites intégrés d’OpenClaw :

- `simple` : fournisseurs simples pilotés par clé API ou environnement
- `profile` : fournisseurs qui apparaissent lorsque des profils d’authentification existent
- `paired` : fournisseurs qui synthétisent plusieurs entrées de fournisseur liées
- `late` : dernier passage, après les autres fournisseurs implicites

Les fournisseurs ultérieurs l’emportent en cas de collision de clé, donc les plugins peuvent intentionnellement remplacer
une entrée de fournisseur intégrée avec le même identifiant.

Compatibilité :

- `discovery` fonctionne toujours comme alias hérité
- si `catalog` et `discovery` sont tous deux enregistrés, OpenClaw utilise `catalog`

## Inspection de canal en lecture seule

Si votre plugin enregistre un canal, il est préférable d’implémenter
`plugin.config.inspectAccount(cfg, accountId)` en parallèle de `resolveAccount(...)`.

Pourquoi :

- `resolveAccount(...)` est le chemin d’exécution. Il peut supposer que les identifiants
  sont entièrement matérialisés et peut échouer immédiatement si les secrets requis sont absents.
- Les chemins de commande en lecture seule tels que `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` et les flux de réparation doctor/config
  ne doivent pas avoir besoin de matérialiser les identifiants d’exécution juste pour
  décrire la configuration.

Comportement recommandé de `inspectAccount(...)` :

- Renvoyer uniquement un état descriptif du compte.
- Préserver `enabled` et `configured`.
- Inclure les champs de source/statut des identifiants lorsque c’est pertinent, comme :
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Vous n’avez pas besoin de renvoyer les valeurs brutes des jetons pour simplement signaler une disponibilité en lecture seule. Renvoyer `tokenStatus: "available"` (et le champ source correspondant) suffit pour les commandes de type statut.
- Utilisez `configured_unavailable` lorsqu’un identifiant est configuré via SecretRef mais
  indisponible dans le chemin de commande courant.

Cela permet aux commandes en lecture seule de signaler « configuré mais indisponible dans ce chemin de commande »
au lieu de planter ou d’indiquer à tort que le compte n’est pas configuré.

## Packs de package

Un répertoire de plugin peut inclure un `package.json` avec `openclaw.extensions` :

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Chaque entrée devient un plugin. Si le pack liste plusieurs extensions, l’identifiant du plugin
devient `name/<fileBase>`.

Si votre plugin importe des dépendances npm, installez-les dans ce répertoire afin que
`node_modules` soit disponible (`npm install` / `pnpm install`).

Barrière de sécurité : chaque entrée `openclaw.extensions` doit rester à l’intérieur du répertoire de plugin
après résolution des liens symboliques. Les entrées qui sortent du répertoire du package sont
rejetées.

Remarque de sécurité : `openclaw plugins install` installe les dépendances du plugin avec
`npm install --omit=dev --ignore-scripts` (pas de scripts de cycle de vie, pas de dépendances de développement à l’exécution). Gardez les arbres de dépendances de plugin en « JS/TS pur » et évitez les packages qui nécessitent des builds `postinstall`.

Facultatif : `openclaw.setupEntry` peut pointer vers un module léger réservé à la configuration.
Quand OpenClaw a besoin des surfaces de configuration pour un plugin de canal désactivé, ou
lorsqu’un plugin de canal est activé mais encore non configuré, il charge `setupEntry`
au lieu de l’entrée complète du plugin. Cela rend le démarrage et la configuration plus légers
lorsque l’entrée principale du plugin raccorde aussi des outils, hooks ou autre code réservé à l’exécution.

Facultatif : `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
peut faire opter un plugin de canal pour ce même chemin `setupEntry` pendant la phase de
démarrage avant écoute de la gateway, même lorsque le canal est déjà configuré.

Utilisez cela uniquement si `setupEntry` couvre entièrement la surface de démarrage qui doit exister
avant que la gateway ne commence à écouter. En pratique, cela signifie que l’entrée de configuration
doit enregistrer toutes les capacités possédées par le canal dont le démarrage dépend, comme :

- l’enregistrement du canal lui-même
- toute route HTTP qui doit être disponible avant que la gateway ne commence à écouter
- toute méthode gateway, outil ou service qui doit exister pendant cette même fenêtre

Si votre entrée complète possède encore une capacité de démarrage requise, n’activez pas
ce drapeau. Conservez le comportement par défaut du plugin et laissez OpenClaw charger l’entrée
complète pendant le démarrage.

Les canaux groupés peuvent aussi publier des assistants de surface de contrat réservés à la configuration que le cœur
peut consulter avant le chargement du runtime complet du canal. La surface actuelle de promotion de configuration
est :

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Le cœur utilise cette surface lorsqu’il doit promouvoir une configuration de canal héritée à compte unique
dans `channels.<id>.accounts.*` sans charger l’entrée complète du plugin.
Matrix est l’exemple groupé actuel : il ne déplace que les clés d’authentification/d’amorçage dans un
compte promu nommé lorsque des comptes nommés existent déjà, et il peut préserver une
clé de compte par défaut configurée non canonique au lieu de toujours créer
`accounts.default`.

Ces adaptateurs de patch de configuration maintiennent une découverte paresseuse de la surface de contrat groupée. Le temps
d’import reste léger ; la surface de promotion n’est chargée qu’au premier usage au lieu de
réentrer dans le démarrage du canal groupé à l’import du module.

Lorsque ces surfaces de démarrage incluent des méthodes RPC gateway, conservez-les sur un
préfixe spécifique au plugin. Les espaces de noms d’administration du cœur (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) restent réservés et se résolvent toujours
en `operator.admin`, même si un plugin demande une portée plus étroite.

Exemple :

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

Les plugins de canal peuvent annoncer des métadonnées de configuration/découverte via `openclaw.channel` ainsi que des
indications d’installation via `openclaw.install`. Cela évite toute donnée de catalogue dans le cœur.

Exemple :

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

Champs utiles de `openclaw.channel` au-delà de l’exemple minimal :

- `detailLabel` : libellé secondaire pour des surfaces de catalogue/statut plus riches
- `docsLabel` : remplace le texte du lien vers la documentation
- `preferOver` : identifiants de plugin/canal de priorité plus basse que cette entrée de catalogue doit surpasser
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras` : contrôles de texte pour les surfaces de sélection
- `markdownCapable` : marque le canal comme compatible markdown pour les décisions de formatage sortant
- `exposure.configured` : masque le canal des surfaces listant les canaux configurés si défini à `false`
- `exposure.setup` : masque le canal des sélecteurs interactifs de configuration si défini à `false`
- `exposure.docs` : marque le canal comme interne/privé pour les surfaces de navigation de documentation
- `showConfigured` / `showInSetup` : alias hérités encore acceptés pour compatibilité ; préférez `exposure`
- `quickstartAllowFrom` : fait entrer le canal dans le flux standard quickstart `allowFrom`
- `forceAccountBinding` : exige une liaison explicite de compte même lorsqu’un seul compte existe
- `preferSessionLookupForAnnounceTarget` : préfère la recherche de session pour résoudre les cibles d’annonce

OpenClaw peut aussi fusionner des **catalogues de canaux externes** (par exemple une
exportation de registre MPM). Déposez un fichier JSON à l’un des emplacements suivants :

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Ou pointez `OPENCLAW_PLUGIN_CATALOG_PATHS` (ou `OPENCLAW_MPM_CATALOG_PATHS`) vers
un ou plusieurs fichiers JSON (délimités par virgules/points-virgules/`PATH`). Chaque fichier doit
contenir `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. L’analyseur accepte aussi `"packages"` ou `"plugins"` comme alias hérités de la clé `"entries"`.

## Plugins de moteur de contexte

Les plugins de moteur de contexte possèdent l’orchestration du contexte de session pour l’ingestion, l’assemblage
et la compaction. Enregistrez-les depuis votre plugin avec
`api.registerContextEngine(id, factory)`, puis sélectionnez le moteur actif avec
`plugins.slots.contextEngine`.

Utilisez ceci lorsque votre plugin doit remplacer ou étendre le pipeline de contexte par défaut plutôt que simplement ajouter une recherche mémoire ou des hooks.

```ts
export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Si votre moteur ne possède **pas** l’algorithme de compaction, gardez `compact()`
implémenté et déléguez-le explicitement :

```ts
import { delegateCompactionToRuntime } from "openclaw/plugin-sdk/core";

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
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Ajouter une nouvelle capacité

Lorsqu’un plugin a besoin d’un comportement qui n’entre pas dans l’API actuelle, ne contournez pas
le système de plugins par un accès privé. Ajoutez la capacité manquante.

Séquence recommandée :

1. définir le contrat du cœur
   Décidez quel comportement partagé le cœur doit posséder : politique, repli, fusion de configuration,
   cycle de vie, sémantique côté canal, et forme de l’assistant d’exécution.
2. ajouter des surfaces typées d’enregistrement/runtime pour les plugins
   Étendez `OpenClawPluginApi` et/ou `api.runtime` avec la surface de capacité typée
   la plus petite utile.
3. raccorder les consommateurs du cœur + canal/fonctionnalité
   Les canaux et plugins de fonctionnalité doivent consommer la nouvelle capacité via le cœur,
   pas en important directement une implémentation de fournisseur.
4. enregistrer les implémentations des fournisseurs
   Les plugins fournisseur enregistrent ensuite leurs backends contre la capacité.
5. ajouter une couverture de contrat
   Ajoutez des tests afin que la propriété et la forme d’enregistrement restent explicites avec le temps.

C’est ainsi qu’OpenClaw reste opiniâtre sans être codé en dur pour la vision du monde d’un
seul fournisseur. Voir le [Capability Cookbook](/fr/plugins/architecture)
pour une checklist concrète de fichiers et un exemple détaillé.

### Checklist de capacité

Quand vous ajoutez une nouvelle capacité, l’implémentation doit généralement toucher
ensemble ces surfaces :

- types de contrat du cœur dans `src/<capability>/types.ts`
- moteur du cœur / assistant d’exécution dans `src/<capability>/runtime.ts`
- surface d’enregistrement de l’API plugin dans `src/plugins/types.ts`
- câblage du registre de plugins dans `src/plugins/registry.ts`
- exposition du runtime de plugin dans `src/plugins/runtime/*` lorsque les plugins de fonctionnalité/canal
  doivent la consommer
- assistants de capture/test dans `src/test-utils/plugin-registration.ts`
- assertions de propriété/contrat dans `src/plugins/contracts/registry.ts`
- documentation opérateur/plugin dans `docs/`

Si l’une de ces surfaces manque, c’est généralement le signe que la capacité
n’est pas encore entièrement intégrée.

### Modèle de capacité

Modèle minimal :

```ts
// contrat du cœur
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// API de plugin
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// assistant d’exécution partagé pour les plugins de fonctionnalité/canal
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Modèle de test de contrat :

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Cela garde la règle simple :

- le cœur possède le contrat de capacité + l’orchestration
- les plugins fournisseur possèdent les implémentations fournisseur
- les plugins de fonctionnalité/canal consomment les assistants d’exécution
- les tests de contrat gardent la propriété explicite
