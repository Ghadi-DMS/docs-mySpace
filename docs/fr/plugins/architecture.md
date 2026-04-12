---
read_when:
    - Créer ou déboguer des plugins OpenClaw natifs
    - Comprendre le modèle de capacités des plugins ou les limites de responsabilité
    - Travailler sur le pipeline de chargement des plugins ou le registre
    - Implémenter des hooks d’exécution de fournisseur ou des plugins de canal
sidebarTitle: Internals
summary: 'Internes du Plugin : modèle de capacités, propriété, contrats, pipeline de chargement et helpers d’exécution'
title: Internes du Plugin
x-i18n:
    generated_at: "2026-04-12T23:28:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: 37361c1e9d2da57c77358396f19dfc7f749708b66ff68f1bf737d051b5d7675d
    source_path: plugins/architecture.md
    workflow: 15
---

# Internes du Plugin

<Info>
  Il s’agit de la **référence d’architecture approfondie**. Pour des guides pratiques, voir :
  - [Installer et utiliser des plugins](/fr/tools/plugin) — guide utilisateur
  - [Premiers pas](/fr/plugins/building-plugins) — premier tutoriel de plugin
  - [Plugins de canal](/fr/plugins/sdk-channel-plugins) — créer un canal de messagerie
  - [Plugins de fournisseur](/fr/plugins/sdk-provider-plugins) — créer un fournisseur de modèles
  - [Vue d’ensemble du SDK](/fr/plugins/sdk-overview) — carte des imports et API d’enregistrement
</Info>

Cette page couvre l’architecture interne du système de plugins d’OpenClaw.

## Modèle public de capacités

Les capacités constituent le modèle public de **plugin natif** dans OpenClaw. Chaque
plugin OpenClaw natif s’enregistre par rapport à un ou plusieurs types de capacités :

| Capability             | Registration method                              | Example plugins                      |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| Inférence de texte     | `api.registerProvider(...)`                      | `openai`, `anthropic`                |
| Backend d’inférence CLI  | `api.registerCliBackend(...)`                    | `openai`, `anthropic`                |
| Voix                  | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`            |
| Transcription en temps réel | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| Voix en temps réel         | `api.registerRealtimeVoiceProvider(...)`         | `openai`                             |
| Compréhension des médias    | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                   |
| Génération d’images       | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| Génération musicale       | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                  |
| Génération de vidéos       | `api.registerVideoGenerationProvider(...)`       | `qwen`                               |
| Récupération web              | `api.registerWebFetchProvider(...)`              | `firecrawl`                          |
| Recherche web             | `api.registerWebSearchProvider(...)`             | `google`                             |
| Canal / messagerie    | `api.registerChannel(...)`                       | `msteams`, `matrix`                  |

Un plugin qui enregistre zéro capacité mais fournit des hooks, des outils ou
des services est un plugin **legacy hook-only**. Ce modèle reste entièrement pris en charge.

### Position de compatibilité externe

Le modèle de capacités est intégré au cœur et utilisé aujourd’hui par les plugins
bundled/natifs, mais la compatibilité des plugins externes exige toujours un critère
plus strict que « c’est exporté, donc c’est figé ».

Consignes actuelles :

- **plugins externes existants :** continuer à faire fonctionner les intégrations basées sur des hooks ; traiter
  cela comme base de compatibilité
- **nouveaux plugins bundled/natifs :** préférer un enregistrement explicite des capacités plutôt que
  des accès spécifiques à un fournisseur ou de nouvelles conceptions hook-only
- **plugins externes adoptant l’enregistrement de capacités :** autorisés, mais traiter les surfaces d’helpers
  spécifiques aux capacités comme évolutives tant que la documentation ne marque pas explicitement
  un contrat comme stable

Règle pratique :

- les API d’enregistrement de capacités sont la direction visée
- les hooks legacy restent la voie la plus sûre pour éviter les ruptures pour les plugins externes pendant
  la transition
- tous les sous-chemins d’helpers exportés ne se valent pas ; préférez le contrat documenté étroit,
  pas des exports d’helpers incidentels

### Formes de plugins

OpenClaw classe chaque plugin chargé dans une forme selon son comportement réel
d’enregistrement (et pas seulement selon des métadonnées statiques) :

- **plain-capability** -- enregistre exactement un type de capacité (par exemple un
  plugin uniquement fournisseur comme `mistral`)
- **hybrid-capability** -- enregistre plusieurs types de capacités (par exemple
  `openai` possède l’inférence de texte, la voix, la compréhension des médias et la génération
  d’images)
- **hook-only** -- enregistre uniquement des hooks (typés ou personnalisés), sans
  capacités, outils, commandes ou services
- **non-capability** -- enregistre des outils, commandes, services ou routes, mais aucune
  capacité

Utilisez `openclaw plugins inspect <id>` pour voir la forme d’un plugin et la
ventilation de ses capacités. Voir la [référence CLI](/cli/plugins#inspect) pour plus de détails.

### Hooks legacy

Le hook `before_agent_start` reste pris en charge comme voie de compatibilité pour les
plugins hook-only. Des plugins legacy réels l’utilisent encore.

Orientation :

- le laisser fonctionner
- le documenter comme legacy
- préférer `before_model_resolve` pour le travail de substitution de modèle/fournisseur
- préférer `before_prompt_build` pour le travail de mutation du prompt
- ne le retirer qu’une fois que l’usage réel aura diminué et que la couverture par fixtures prouvera la sûreté de la migration

### Signaux de compatibilité

Lorsque vous exécutez `openclaw doctor` ou `openclaw plugins inspect <id>`, vous pouvez voir
l’un de ces libellés :

| Signal                     | Signification                                                      |
| -------------------------- | ------------------------------------------------------------ |
| **config valid**           | La configuration est analysée correctement et les plugins se résolvent                       |
| **compatibility advisory** | Le plugin utilise un modèle pris en charge mais plus ancien (par ex. `hook-only`) |
| **legacy warning**         | Le plugin utilise `before_agent_start`, qui est obsolète        |
| **hard error**             | La configuration est invalide ou le plugin n’a pas pu être chargé                   |

Ni `hook-only` ni `before_agent_start` ne casseront votre plugin aujourd’hui --
`hook-only` est indicatif, et `before_agent_start` ne déclenche qu’un avertissement. Ces
signaux apparaissent aussi dans `openclaw status --all` et `openclaw plugins doctor`.

## Vue d’ensemble de l’architecture

Le système de plugins d’OpenClaw comporte quatre couches :

1. **Manifeste + découverte**
   OpenClaw trouve les plugins candidats à partir des chemins configurés, des
   racines d’espace de travail, des racines globales d’extensions et des extensions bundled. La découverte lit d’abord les
   manifestes natifs `openclaw.plugin.json` ainsi que les manifestes de bundle pris en charge.
2. **Activation + validation**
   Le cœur décide si un plugin découvert est activé, désactivé, bloqué ou
   sélectionné pour un slot exclusif tel que la mémoire.
3. **Chargement à l’exécution**
   Les plugins OpenClaw natifs sont chargés dans le processus via jiti et enregistrent
   des capacités dans un registre central. Les bundles compatibles sont normalisés en
   enregistrements de registre sans importer de code d’exécution.
4. **Consommation des surfaces**
   Le reste d’OpenClaw lit le registre pour exposer les outils, canaux, configuration
   des fournisseurs, hooks, routes HTTP, commandes CLI et services.

Pour la CLI des plugins en particulier, la découverte des commandes racines est scindée en deux phases :

- les métadonnées d’analyse proviennent de `registerCli(..., { descriptors: [...] })`
- le véritable module CLI du plugin peut rester paresseux et s’enregistrer à la première invocation

Cela permet de conserver le code CLI appartenant au plugin dans le plugin tout en laissant OpenClaw
réserver les noms de commandes racines avant l’analyse.

La limite de conception importante :

- la découverte + la validation de configuration doivent fonctionner à partir des **métadonnées de manifeste/schéma**
  sans exécuter le code du plugin
- le comportement d’exécution natif provient du chemin `register(api)` du module du plugin

Cette séparation permet à OpenClaw de valider la configuration, d’expliquer les plugins manquants/désactivés et
de construire des indications d’interface utilisateur/schéma avant que l’exécution complète ne soit active.

### Plugins de canal et outil de message partagé

Les plugins de canal n’ont pas besoin d’enregistrer un outil distinct d’envoi/édition/réaction pour
les actions de chat normales. OpenClaw conserve un outil `message` partagé dans le cœur, et les
plugins de canal possèdent la découverte et l’exécution spécifiques au canal derrière celui-ci.

La limite actuelle est la suivante :

- le cœur possède l’hôte de l’outil `message` partagé, le câblage du prompt, la tenue des
  sessions/threads et la répartition de l’exécution
- les plugins de canal possèdent la découverte des actions à portée, la découverte des capacités et
  tous les fragments de schéma spécifiques au canal
- les plugins de canal possèdent la grammaire de conversation de session spécifique au fournisseur, comme
  la façon dont les identifiants de conversation encodent les identifiants de thread ou héritent des conversations parentes
- les plugins de canal exécutent l’action finale via leur adaptateur d’action

Pour les plugins de canal, la surface SDK est
`ChannelMessageActionAdapter.describeMessageTool(...)`. Cet appel de découverte unifié
permet à un plugin de retourner ensemble ses actions visibles, ses capacités et ses contributions de schéma afin
que ces éléments ne divergent pas.

Le cœur transmet la portée d’exécution à cette étape de découverte. Les champs importants incluent :

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- `requesterSenderId` entrant approuvé

Cela compte pour les plugins sensibles au contexte. Un canal peut masquer ou exposer
des actions de message en fonction du compte actif, de la room/du thread/du message courant, ou
de l’identité approuvée du demandeur sans coder en dur de branches spécifiques au canal dans l’outil
`message` du cœur.

C’est pourquoi les changements de routage embedded-runner restent du travail de plugin : le runner est
responsable de transmettre l’identité actuelle de chat/session à la limite de découverte du plugin afin
que l’outil `message` partagé expose la bonne surface possédée par le canal pour le tour actuel.

Pour les helpers d’exécution appartenant au canal, les plugins bundled doivent conserver l’exécution
runtime dans leurs propres modules d’extension. Le cœur ne possède plus les runtimes d’action de message
Discord, Slack, Telegram ou WhatsApp sous `src/agents/tools`.
Nous ne publions pas de sous-chemins `plugin-sdk/*-action-runtime` séparés, et les plugins bundled
doivent importer directement leur propre code runtime local depuis leurs
modules possédés par l’extension.

La même limite s’applique aux coutures SDK nommées par fournisseur en général : le cœur ne doit
pas importer de barrels de commodité spécifiques à un canal pour Slack, Discord, Signal,
WhatsApp ou des extensions similaires. Si le cœur a besoin d’un comportement, il doit soit
consommer le barrel `api.ts` / `runtime-api.ts` propre au plugin bundled, soit faire
remonter ce besoin en une capacité générique étroite dans le SDK partagé.

Pour les sondages en particulier, il existe deux chemins d’exécution :

- `outbound.sendPoll` est la base partagée pour les canaux qui correspondent au modèle commun
  de sondage
- `actions.handleAction("poll")` est le chemin préféré pour une sémantique de sondage spécifique au canal ou des paramètres
  de sondage supplémentaires

Le cœur reporte désormais l’analyse partagée des sondages jusqu’à ce que la répartition du sondage du plugin refuse
l’action, afin que les gestionnaires de sondage appartenant au plugin puissent accepter des champs de sondage spécifiques au canal
sans être bloqués d’abord par l’analyseur de sondage générique.

Voir [Pipeline de chargement](#load-pipeline) pour la séquence complète de démarrage.

## Modèle de propriété des capacités

OpenClaw traite un plugin natif comme la limite de responsabilité pour une **entreprise** ou une
**fonctionnalité**, et non comme un fourre-tout d’intégrations sans lien.

Cela signifie :

- un plugin d’entreprise doit généralement posséder toutes les surfaces OpenClaw de cette entreprise
- un plugin de fonctionnalité doit généralement posséder toute la surface de fonctionnalité qu’il introduit
- les canaux doivent consommer les capacités partagées du cœur au lieu de réimplémenter
  de façon ad hoc le comportement des fournisseurs

Exemples :

- le plugin bundled `openai` possède le comportement de fournisseur de modèles OpenAI ainsi que le comportement OpenAI
  de voix + voix en temps réel + compréhension des médias + génération d’images
- le plugin bundled `elevenlabs` possède le comportement de voix ElevenLabs
- le plugin bundled `microsoft` possède le comportement de voix Microsoft
- le plugin bundled `google` possède le comportement de fournisseur de modèles Google ainsi que
  le comportement Google de compréhension des médias + génération d’images + recherche web
- le plugin bundled `firecrawl` possède le comportement de récupération web Firecrawl
- les plugins bundled `minimax`, `mistral`, `moonshot` et `zai` possèdent leurs
  backends de compréhension des médias
- le plugin bundled `qwen` possède le comportement de fournisseur de texte Qwen ainsi que
  la compréhension des médias et la génération de vidéos
- le plugin `voice-call` est un plugin de fonctionnalité : il possède le transport d’appel, les outils,
  la CLI, les routes et le pontage de flux média Twilio, mais il consomme les capacités partagées
  de voix + transcription en temps réel + voix en temps réel au lieu d’importer directement des plugins fournisseurs

L’état final visé est :

- OpenAI vit dans un seul plugin même si cela couvre les modèles de texte, la voix, les images et
  de futures capacités vidéo
- un autre fournisseur peut faire de même pour sa propre surface fonctionnelle
- les canaux ne se soucient pas de savoir quel plugin fournisseur possède le provider ; ils consomment le
  contrat de capacité partagée exposé par le cœur

C’est la distinction clé :

- **plugin** = limite de responsabilité
- **capability** = contrat du cœur que plusieurs plugins peuvent implémenter ou consommer

Ainsi, si OpenClaw ajoute un nouveau domaine comme la vidéo, la première question n’est pas
« quel fournisseur devrait coder en dur la gestion de la vidéo ? » La première question est « quel est
le contrat de capacité vidéo du cœur ? » Une fois ce contrat en place, les plugins fournisseurs
peuvent s’y enregistrer et les plugins de canal/fonctionnalité peuvent le consommer.

Si la capacité n’existe pas encore, la bonne démarche est généralement :

1. définir la capacité manquante dans le cœur
2. l’exposer via l’API/runtime du plugin de manière typée
3. raccorder les canaux/fonctionnalités à cette capacité
4. laisser les plugins fournisseurs enregistrer les implémentations

Cela garde la responsabilité explicite tout en évitant un comportement du cœur dépendant d’un
seul fournisseur ou d’un chemin de code spécifique à un plugin unique.

### Superposition des capacités

Utilisez ce modèle mental pour décider où le code doit se trouver :

- **couche de capacité du cœur** : orchestration partagée, politique, repli, règles de fusion
  de configuration, sémantique de livraison et contrats typés
- **couche de plugin fournisseur** : API spécifiques au fournisseur, authentification, catalogues de modèles, synthèse vocale,
  génération d’images, futurs backends vidéo, points de terminaison d’usage
- **couche de plugin de canal/fonctionnalité** : intégration Slack/Discord/voice-call/etc.
  qui consomme les capacités du cœur et les présente sur une surface

Par exemple, le TTS suit cette forme :

- le cœur possède la politique TTS au moment de la réponse, l’ordre de repli, les préférences et la livraison par canal
- `openai`, `elevenlabs` et `microsoft` possèdent les implémentations de synthèse
- `voice-call` consomme le helper runtime TTS pour la téléphonie

Ce même modèle doit être préféré pour les capacités futures.

### Exemple de plugin d’entreprise multi-capacités

Un plugin d’entreprise doit sembler cohérent vu de l’extérieur. Si OpenClaw dispose de
contrats partagés pour les modèles, la voix, la transcription en temps réel, la voix en temps réel, la
compréhension des médias, la génération d’images, la génération de vidéos, la récupération web et la recherche web,
un fournisseur peut posséder toutes ses surfaces en un seul endroit :

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

Ce qui compte n’est pas le nom exact des helpers. C’est la forme qui compte :

- un seul plugin possède la surface du fournisseur
- le cœur possède toujours les contrats de capacité
- les canaux et plugins de fonctionnalité consomment les helpers `api.runtime.*`, pas du code fournisseur
- les tests de contrat peuvent vérifier que le plugin a enregistré les capacités qu’il
  prétend posséder

### Exemple de capacité : compréhension vidéo

OpenClaw traite déjà la compréhension d’image/audio/vidéo comme une seule
capacité partagée. Le même modèle de responsabilité s’y applique :

1. le cœur définit le contrat de compréhension des médias
2. les plugins fournisseurs enregistrent `describeImage`, `transcribeAudio` et
   `describeVideo` selon le cas
3. les plugins de canal et de fonctionnalité consomment le comportement partagé du cœur au lieu de
   se raccorder directement au code fournisseur

Cela évite d’intégrer dans le cœur les hypothèses vidéo d’un fournisseur donné. Le plugin possède
la surface fournisseur ; le cœur possède le contrat de capacité et le comportement de repli.

La génération vidéo suit déjà cette même séquence : le cœur possède le contrat de
capacité typé et le helper runtime, et les plugins fournisseurs enregistrent des
implémentations `api.registerVideoGenerationProvider(...)` sur cette base.

Besoin d’une checklist concrète de déploiement ? Voir
[Capability Cookbook](/fr/plugins/architecture).

## Contrats et application

La surface de l’API des plugins est volontairement typée et centralisée dans
`OpenClawPluginApi`. Ce contrat définit les points d’enregistrement pris en charge et
les helpers runtime sur lesquels un plugin peut s’appuyer.

Pourquoi c’est important :

- les auteurs de plugins disposent d’une norme interne stable
- le cœur peut rejeter une propriété dupliquée, par exemple deux plugins enregistrant le même
  id de fournisseur
- le démarrage peut faire remonter des diagnostics exploitables pour des enregistrements mal formés
- les tests de contrat peuvent faire respecter la responsabilité des plugins bundled et empêcher une dérive silencieuse

Il existe deux couches d’application :

1. **application à l’enregistrement runtime**
   Le registre des plugins valide les enregistrements au chargement des plugins. Exemples :
   ids de fournisseur dupliqués, ids de fournisseur de voix dupliqués et enregistrements
   mal formés produisent des diagnostics de plugin au lieu d’un comportement indéfini.
2. **tests de contrat**
   Les plugins bundled sont capturés dans des registres de contrat pendant les exécutions de test afin
   qu’OpenClaw puisse vérifier explicitement la responsabilité. Aujourd’hui cela est utilisé pour les
   fournisseurs de modèles, les fournisseurs de voix, les fournisseurs de recherche web et la responsabilité
   d’enregistrement bundled.

L’effet pratique est qu’OpenClaw sait, dès le départ, quel plugin possède quelle
surface. Cela permet au cœur et aux canaux de se composer sans friction, car la responsabilité est
déclarée, typée et testable plutôt qu’implicite.

### Ce qui doit figurer dans un contrat

Les bons contrats de plugin sont :

- typés
- petits
- spécifiques à une capacité
- possédés par le cœur
- réutilisables par plusieurs plugins
- consommables par des canaux/fonctionnalités sans connaissance du fournisseur

Les mauvais contrats de plugin sont :

- une politique spécifique à un fournisseur cachée dans le cœur
- des échappatoires ponctuelles pour plugin qui contournent le registre
- du code de canal accédant directement à une implémentation fournisseur
- des objets runtime ad hoc qui ne font pas partie de `OpenClawPluginApi` ou
  `api.runtime`

En cas de doute, montez le niveau d’abstraction : définissez d’abord la capacité, puis
laissez les plugins s’y brancher.

## Modèle d’exécution

Les plugins OpenClaw natifs s’exécutent **dans le processus** avec la Gateway. Ils ne sont
pas sandboxés. Un plugin natif chargé a la même limite de confiance au niveau du processus que le code du cœur.

Implications :

- un plugin natif peut enregistrer des outils, gestionnaires réseau, hooks et services
- un bogue de plugin natif peut faire planter ou déstabiliser la gateway
- un plugin natif malveillant équivaut à une exécution de code arbitraire dans le processus OpenClaw

Les bundles compatibles sont plus sûrs par défaut, car OpenClaw les traite actuellement
comme des packs de métadonnées/contenu. Dans les versions actuelles, cela signifie surtout des
Skills bundled.

Utilisez des listes d’autorisation et des chemins explicites d’installation/chargement pour les plugins non bundled. Considérez
les plugins d’espace de travail comme du code de développement, pas comme des valeurs par défaut de production.

Pour les noms de paquets bundled d’espace de travail, gardez l’id du plugin ancré dans le nom npm :
`@openclaw/<id>` par défaut, ou un suffixe typé approuvé tel que
`-provider`, `-plugin`, `-speech`, `-sandbox` ou `-media-understanding` lorsque
le paquet expose intentionnellement un rôle de plugin plus étroit.

Note de confiance importante :

- `plugins.allow` fait confiance aux **ids de plugin**, pas à la provenance de la source.
- Un plugin d’espace de travail ayant le même id qu’un plugin bundled masque intentionnellement
  la copie bundled lorsque ce plugin d’espace de travail est activé/sur liste d’autorisation.
- C’est normal et utile pour le développement local, les tests de correctifs et les hotfixes.

## Limite d’export

OpenClaw exporte des capacités, pas des commodités d’implémentation.

Gardez public l’enregistrement des capacités. Réduisez les exports d’helpers hors contrat :

- sous-chemins d’helpers spécifiques à un plugin bundled
- sous-chemins de plomberie runtime non destinés à être une API publique
- helpers de commodité spécifiques à un fournisseur
- helpers de configuration/intégration qui sont des détails d’implémentation

Certains sous-chemins d’helpers de plugins bundled restent encore dans la carte d’exports SDK générée
pour la compatibilité et la maintenance des plugins bundled. Exemples actuels :
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup` et plusieurs coutures `plugin-sdk/matrix*`. Traitez-les comme des
exports réservés de détail d’implémentation, et non comme le modèle SDK recommandé pour de nouveaux plugins tiers.

## Pipeline de chargement

Au démarrage, OpenClaw fait approximativement ceci :

1. découvrir les racines de plugins candidates
2. lire les manifestes natifs ou de bundles compatibles et les métadonnées de paquet
3. rejeter les candidats non sûrs
4. normaliser la configuration des plugins (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. décider de l’activation pour chaque candidat
6. charger les modules natifs activés via jiti
7. appeler les hooks natifs `register(api)` (ou `activate(api)` — un alias legacy) et collecter les enregistrements dans le registre des plugins
8. exposer le registre aux surfaces de commandes/runtime

<Note>
`activate` est un alias legacy de `register` — le chargeur résout celui qui est présent (`def.register ?? def.activate`) et l’appelle au même moment. Tous les plugins bundled utilisent `register` ; préférez `register` pour les nouveaux plugins.
</Note>

Les garde-fous de sécurité s’appliquent **avant** l’exécution runtime. Les candidats sont bloqués
lorsque le point d’entrée sort de la racine du plugin, que le chemin est accessible en écriture à tous, ou que la propriété du chemin paraît suspecte pour des plugins non bundled.

### Comportement manifest-first

Le manifeste est la source de vérité du plan de contrôle. OpenClaw l’utilise pour :

- identifier le plugin
- découvrir les canaux/Skills/schéma de configuration déclarés ou les capacités du bundle
- valider `plugins.entries.<id>.config`
- enrichir les libellés/placeholders de la Control UI
- afficher les métadonnées d’installation/catalogue
- préserver des descripteurs bon marché d’activation et de configuration sans charger le runtime du plugin

Pour les plugins natifs, le module runtime est la partie plan de données. Il enregistre le
comportement réel tel que hooks, outils, commandes ou flux de fournisseur.

Les blocs optionnels `activation` et `setup` du manifeste restent sur le plan de contrôle.
Ce sont des descripteurs de métadonnées uniquement pour la planification de l’activation et la découverte de configuration ;
ils ne remplacent pas l’enregistrement runtime, `register(...)` ni `setupEntry`.
Les premiers consommateurs d’activation réelle utilisent maintenant les indications de commande, de canal et de fournisseur du manifeste
pour restreindre le chargement des plugins avant une matérialisation plus large du registre :

- le chargement CLI se limite aux plugins qui possèdent la commande primaire demandée
- la résolution de configuration/plugin de canal se limite aux plugins qui possèdent l’id
  de canal demandé
- la résolution explicite de configuration/runtime du fournisseur se limite aux plugins qui possèdent l’id
  de fournisseur demandé

La découverte de configuration privilégie maintenant les ids possédés par des descripteurs tels que `setup.providers` et
`setup.cliBackends` pour restreindre les plugins candidats avant de retomber sur
`setup-api` pour les plugins qui ont encore besoin de hooks runtime au moment de la configuration. Si plusieurs
plugins découverts revendiquent le même id normalisé de fournisseur de configuration ou de backend CLI, la recherche de configuration refuse ce propriétaire ambigu au lieu de s’appuyer sur l’ordre de découverte.

### Ce que le chargeur met en cache

OpenClaw conserve de courts caches dans le processus pour :

- les résultats de découverte
- les données de registre de manifestes
- les registres de plugins chargés

Ces caches réduisent les pointes de démarrage et la surcharge des commandes répétées. Il faut les considérer
comme des caches de performance à courte durée de vie, pas comme de la persistance.

Remarque de performance :

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
- les hooks legacy et hooks typés
- les canaux
- les fournisseurs
- les gestionnaires RPC Gateway
- les routes HTTP
- les enregistreurs CLI
- les services d’arrière-plan
- les commandes appartenant au plugin

Les fonctionnalités du cœur lisent ensuite depuis ce registre au lieu de communiquer
directement avec les modules de plugin. Cela maintient un chargement à sens unique :

- module de plugin -> enregistrement dans le registre
- runtime du cœur -> consommation du registre

Cette séparation est importante pour la maintenabilité. Elle signifie que la plupart des surfaces du cœur n’ont
besoin que d’un seul point d’intégration : « lire le registre », et non « gérer spécialement chaque module de plugin ».

## Callbacks de liaison de conversation

Les plugins qui lient une conversation peuvent réagir lorsqu’une approbation est résolue.

Utilisez `api.onConversationBindingResolved(...)` pour recevoir un callback après qu’une demande de liaison a été approuvée ou refusée :

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

Champs de la charge utile du callback :

- `status` : `"approved"` ou `"denied"`
- `decision` : `"allow-once"`, `"allow-always"` ou `"deny"`
- `binding` : la liaison résolue pour les demandes approuvées
- `request` : le résumé de la demande d’origine, l’indication de détachement, l’id de l’expéditeur et
  les métadonnées de conversation

Ce callback est uniquement une notification. Il ne change pas qui est autorisé à lier une
conversation, et il s’exécute une fois que le traitement d’approbation du cœur est terminé.

## Hooks d’exécution de fournisseur

Les plugins de fournisseur ont maintenant deux couches :

- métadonnées du manifeste : `providerAuthEnvVars` pour une recherche légère de l’authentification fournisseur par variable d’environnement
  avant le chargement du runtime, `providerAuthAliases` pour les variantes de fournisseur qui partagent
  l’authentification, `channelEnvVars` pour une recherche légère de la configuration/authentification de canal par variable d’environnement avant le chargement du runtime,
  plus `providerAuthChoices` pour des libellés légers d’intégration/choix d’authentification et
  des métadonnées de drapeau CLI avant le chargement du runtime
- hooks au moment de la configuration : `catalog` / `discovery` legacy plus `applyConfigDefaults`
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

OpenClaw possède toujours la boucle d’agent générique, le failover, la gestion de transcription et
la politique d’outils. Ces hooks constituent la surface d’extension pour le comportement spécifique au fournisseur sans
nécessiter tout un transport d’inférence personnalisé.

Utilisez le manifeste `providerAuthEnvVars` lorsque le fournisseur possède des identifiants basés sur l’environnement
que les chemins génériques d’authentification/statut/sélecteur de modèle doivent voir sans charger le runtime du plugin. Utilisez le manifeste `providerAuthAliases` lorsqu’un id de fournisseur doit réutiliser
les variables d’environnement, profils d’authentification, authentification basée sur la configuration et choix d’intégration de clé API d’un autre id de fournisseur. Utilisez le manifeste `providerAuthChoices` lorsque les surfaces CLI
d’intégration/choix d’authentification doivent connaître l’id de choix du fournisseur, les libellés de groupe et le câblage d’authentification simple à un seul drapeau sans charger le runtime du fournisseur. Conservez `envVars` dans le runtime du fournisseur pour des indications destinées à l’opérateur, comme les libellés d’intégration ou les
variables de configuration OAuth client-id/client-secret.

Utilisez le manifeste `channelEnvVars` lorsqu’un canal possède une authentification ou une configuration pilotée par variables d’environnement que les retombées génériques d’environnement shell, les vérifications de configuration/statut ou les invites de configuration doivent voir
sans charger le runtime du canal.

### Ordre des hooks et utilisation

Pour les plugins de modèle/fournisseur, OpenClaw appelle les hooks dans cet ordre approximatif.
La colonne « Quand l’utiliser » est le guide rapide de décision.

| #   | Hook                              | Ce qu’il fait                                                                                                   | Quand l’utiliser                                                                                                                                 |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | Publie la configuration du fournisseur dans `models.providers` lors de la génération de `models.json`                                | Le fournisseur possède un catalogue ou des valeurs par défaut de base URL                                                                                                |
| 2   | `applyConfigDefaults`             | Applique les valeurs par défaut globales de configuration appartenant au fournisseur lors de la matérialisation de la configuration                                      | Les valeurs par défaut dépendent du mode d’authentification, de l’environnement ou de la sémantique de famille de modèles du fournisseur                                                                       |
| --  | _(built-in model lookup)_         | OpenClaw essaie d’abord le chemin normal de registre/catalogue                                                          | _(pas un hook de plugin)_                                                                                                                       |
| 3   | `normalizeModelId`                | Normalise les alias d’id de modèle legacy ou de préversion avant la recherche                                                     | Le fournisseur possède le nettoyage des alias avant la résolution canonique du modèle                                                                               |
| 4   | `normalizeTransport`              | Normalise `api` / `baseUrl` de la famille de fournisseurs avant l’assemblage générique du modèle                                      | Le fournisseur possède le nettoyage du transport pour des ids de fournisseur personnalisés dans la même famille de transport                                                        |
| 5   | `normalizeConfig`                 | Normalise `models.providers.<id>` avant la résolution runtime/fournisseur                                           | Le fournisseur a besoin d’un nettoyage de configuration qui doit vivre avec le plugin ; les helpers bundled de la famille Google renforcent également les entrées de configuration Google prises en charge |
| 6   | `applyNativeStreamingUsageCompat` | Applique des réécritures de compatibilité d’usage de streaming natif aux fournisseurs de configuration                                               | Le fournisseur a besoin de correctifs de métadonnées d’usage de streaming natif pilotés par le point de terminaison                                                                        |
| 7   | `resolveConfigApiKey`             | Résout l’authentification par marqueur d’environnement pour les fournisseurs de configuration avant le chargement de l’authentification runtime                                       | Le fournisseur possède sa propre résolution de clé API par marqueur d’environnement ; `amazon-bedrock` a également ici un résolveur intégré de marqueur d’environnement AWS                |
| 8   | `resolveSyntheticAuth`            | Expose une authentification locale/autohébergée ou basée sur la configuration sans persister de texte brut                                   | Le fournisseur peut fonctionner avec un marqueur d’identifiant synthétique/local                                                                               |
| 9   | `resolveExternalAuthProfiles`     | Superpose des profils d’authentification externes appartenant au fournisseur ; la `persistence` par défaut est `runtime-only` pour les identifiants détenus par CLI/app | Le fournisseur réutilise des identifiants d’authentification externes sans persister de jetons d’actualisation copiés                                                          |
| 10  | `shouldDeferSyntheticProfileAuth` | Abaisse la priorité des espaces réservés synthétiques stockés de profil derrière l’authentification basée sur l’environnement/la configuration                                      | Le fournisseur stocke des profils espaces réservés synthétiques qui ne doivent pas avoir la priorité                                                               |
| 11  | `resolveDynamicModel`             | Repli synchrone pour des ids de modèle appartenant au fournisseur mais pas encore présents dans le registre local                                       | Le fournisseur accepte des ids de modèle amont arbitraires                                                                                               |
| 12  | `prepareDynamicModel`             | Préchauffage asynchrone, puis `resolveDynamicModel` s’exécute à nouveau                                                           | Le fournisseur a besoin de métadonnées réseau avant de résoudre des ids inconnus                                                                                |
| 13  | `normalizeResolvedModel`          | Réécriture finale avant que l’embedded runner utilise le modèle résolu                                               | Le fournisseur a besoin de réécritures de transport tout en utilisant un transport du cœur                                                                           |
| 14  | `contributeResolvedModelCompat`   | Ajoute des indicateurs de compatibilité pour les modèles fournisseur derrière un autre transport compatible                                  | Le fournisseur reconnaît ses propres modèles sur des transports proxy sans prendre le contrôle du fournisseur                                                     |
| 15  | `capabilities`                    | Métadonnées de transcription/outillage appartenant au fournisseur utilisées par la logique partagée du cœur                                           | Le fournisseur a besoin de particularités liées à la transcription/à la famille de fournisseurs                                                                                            |
| 16  | `normalizeToolSchemas`            | Normalise les schémas d’outils avant que l’embedded runner ne les voie                                                    | Le fournisseur a besoin d’un nettoyage de schéma propre à la famille de transport                                                                                              |
| 17  | `inspectToolSchemas`              | Expose des diagnostics de schéma appartenant au fournisseur après normalisation                                                  | Le fournisseur veut des avertissements sur les mots-clés sans enseigner au cœur des règles spécifiques au fournisseur                                                               |
| 18  | `resolveReasoningOutputMode`      | Sélectionne le contrat de sortie de raisonnement natif ou balisé                                                              | Le fournisseur a besoin d’une sortie raisonnement/finale balisée au lieu de champs natifs                                                                       |
| 19  | `prepareExtraParams`              | Normalisation des paramètres de requête avant les wrappers génériques d’options de flux                                              | Le fournisseur a besoin de paramètres de requête par défaut ou d’un nettoyage de paramètres par fournisseur                                                                         |
| 20  | `createStreamFn`                  | Remplace entièrement le chemin de flux normal par un transport personnalisé                                                   | Le fournisseur a besoin d’un protocole filaire personnalisé, et pas seulement d’un wrapper                                                                                   |
| 21  | `wrapStreamFn`                    | Wrapper de flux après application des wrappers génériques                                                              | Le fournisseur a besoin de wrappers de compatibilité pour en-têtes/corps/modèle de requête sans transport personnalisé                                                        |
| 22  | `resolveTransportTurnState`       | Attache des en-têtes ou métadonnées natives par tour de transport                                                           | Le fournisseur veut que les transports génériques envoient une identité de tour native au fournisseur                                                                     |
| 23  | `resolveWebSocketSessionPolicy`   | Attache des en-têtes WebSocket natifs ou une politique de refroidissement de session                                                    | Le fournisseur veut que les transports WS génériques ajustent les en-têtes de session ou la politique de repli                                                             |
| 24  | `formatApiKey`                    | Formateur de profil d’authentification : le profil stocké devient la chaîne `apiKey` runtime                                     | Le fournisseur stocke des métadonnées d’authentification supplémentaires et a besoin d’une forme de jeton runtime personnalisée                                                                  |
| 25  | `refreshOAuth`                    | Surcharge d’actualisation OAuth pour des points de terminaison d’actualisation personnalisés ou une politique d’échec d’actualisation                                  | Le fournisseur ne correspond pas aux actualisateurs partagés `pi-ai`                                                                                         |
| 26  | `buildAuthDoctorHint`             | Indication de réparation ajoutée lorsque l’actualisation OAuth échoue                                                                  | Le fournisseur a besoin d’une consigne de réparation d’authentification lui appartenant après un échec d’actualisation                                                                    |
| 27  | `matchesContextOverflowError`     | Correspondance d’erreur de dépassement de fenêtre de contexte appartenant au fournisseur                                                                 | Le fournisseur a des erreurs brutes de dépassement que les heuristiques génériques manqueraient                                                                              |
| 28  | `classifyFailoverReason`          | Classification de motif de failover appartenant au fournisseur                                                                  | Le fournisseur peut mapper des erreurs brutes d’API/transport en limite de débit/surcharge/etc.                                                                        |
| 29  | `isCacheTtlEligible`              | Politique de cache de prompt pour les fournisseurs proxy/backhaul                                                               | Le fournisseur a besoin d’un contrôle TTL de cache spécifique au proxy                                                                                              |
| 30  | `buildMissingAuthMessage`         | Remplacement du message générique de récupération en cas d’authentification manquante                                                      | Le fournisseur a besoin d’une indication de récupération spécifique au fournisseur pour une authentification manquante                                                                               |
| 31  | `suppressBuiltInModel`            | Suppression de modèles amont obsolètes avec indication facultative d’erreur orientée utilisateur                                          | Le fournisseur doit masquer des lignes amont obsolètes ou les remplacer par une indication fournisseur                                                               |
| 32  | `augmentModelCatalog`             | Lignes de catalogue synthétiques/finales ajoutées après la découverte                                                          | Le fournisseur a besoin de lignes synthétiques de compatibilité ascendante dans `models list` et les sélecteurs                                                                   |
| 33  | `isBinaryThinking`                | Bascule de raisonnement activé/désactivé pour les fournisseurs à raisonnement binaire                                                          | Le fournisseur n’expose qu’un raisonnement binaire activé/désactivé                                                                                                |
| 34  | `supportsXHighThinking`           | Prise en charge du raisonnement `xhigh` pour certains modèles                                                                  | Le fournisseur veut `xhigh` seulement sur un sous-ensemble de modèles                                                                                           |
| 35  | `resolveDefaultThinkingLevel`     | Niveau `/think` par défaut pour une famille de modèles spécifique                                                             | Le fournisseur possède la politique `/think` par défaut pour une famille de modèles                                                                                    |
| 36  | `isModernModelRef`                | Correspondance de modèle moderne pour les filtres de profil en direct et la sélection smoke                                              | Le fournisseur possède la correspondance de modèle préféré pour les profils en direct/smoke                                                                                           |
| 37  | `prepareRuntimeAuth`              | Échange un identifiant configuré contre le jeton/la clé runtime réel juste avant l’inférence                       | Le fournisseur a besoin d’un échange de jeton ou d’un identifiant de requête à courte durée de vie                                                                           |
| 38  | `resolveUsageAuth`                | Résout les identifiants d’usage/facturation pour `/usage` et les surfaces d’état associées                                     | Le fournisseur a besoin d’une analyse personnalisée des jetons d’usage/quota ou d’un identifiant d’usage différent                                                             |
| 39  | `fetchUsageSnapshot`              | Récupère et normalise des instantanés d’usage/quota spécifiques au fournisseur une fois l’authentification résolue                             | Le fournisseur a besoin d’un point de terminaison d’usage spécifique au fournisseur ou d’un analyseur de charge utile                                                                         |
| 40  | `createEmbeddingProvider`         | Construit un adaptateur d’embedding appartenant au fournisseur pour la mémoire/la recherche                                                     | Le comportement d’embedding de la mémoire appartient au plugin fournisseur                                                                                  |
| 41  | `buildReplayPolicy`               | Retourne une politique de rejeu contrôlant la gestion de transcription pour le fournisseur                                        | Le fournisseur a besoin d’une politique de transcription personnalisée (par exemple, suppression des blocs de réflexion)                                                             |
| 42  | `sanitizeReplayHistory`           | Réécrit l’historique de rejeu après le nettoyage générique de transcription                                                        | Le fournisseur a besoin de réécritures de rejeu spécifiques au fournisseur au-delà des helpers partagés de Compaction                                                           |
| 43  | `validateReplayTurns`             | Validation finale ou remise en forme des tours de rejeu avant l’embedded runner                                           | Le transport fournisseur a besoin d’une validation plus stricte des tours après l’assainissement générique                                                                  |
| 44  | `onModelSelected`                 | Exécute des effets de bord post-sélection appartenant au fournisseur                                                                 | Le fournisseur a besoin de télémétrie ou d’un état appartenant au fournisseur lorsqu’un modèle devient actif                                                                |

`normalizeModelId`, `normalizeTransport` et `normalizeConfig` vérifient d’abord le
plugin fournisseur correspondant, puis passent aux autres plugins fournisseurs capables de hooks
jusqu’à ce que l’un d’eux modifie effectivement l’id du modèle ou le transport/la configuration. Cela permet aux shims
d’alias/compatibilité de fournisseur de continuer à fonctionner sans obliger l’appelant à savoir quel
plugin bundled possède la réécriture. Si aucun hook fournisseur ne réécrit une entrée de configuration
prise en charge de la famille Google, le normaliseur de configuration Google bundled applique tout de même
ce nettoyage de compatibilité.

Si le fournisseur a besoin d’un protocole filaire entièrement personnalisé ou d’un exécuteur de requête
personnalisé, il s’agit d’une autre classe d’extension. Ces hooks sont destinés au comportement
fournisseur qui s’exécute encore sur la boucle d’inférence normale d’OpenClaw.

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
  et `wrapStreamFn` parce qu’il possède la compatibilité ascendante de Claude 4.6,
  les indications de famille de fournisseur, les consignes de réparation d’authentification, l’intégration du point de terminaison d’usage,
  l’éligibilité du cache de prompt, les valeurs par défaut de configuration sensibles à l’authentification, la politique
  de réflexion par défaut/adaptative de Claude, et la mise en forme de flux spécifique à Anthropic pour les
  en-têtes bêta, `/fast` / `serviceTier`, et `context1m`.
- Les helpers de flux spécifiques à Claude d’Anthropic restent pour l’instant dans la
  couture publique propre au plugin bundled `api.ts` / `contract-api.ts`. Cette surface de paquet
  exporte `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, et les builders de wrappers
  Anthropic de plus bas niveau au lieu d’élargir le SDK générique autour des règles d’en-tête bêta d’un
  seul fournisseur.
- OpenAI utilise `resolveDynamicModel`, `normalizeResolvedModel` et
  `capabilities` ainsi que `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking`, et `isModernModelRef`
  parce qu’il possède la compatibilité ascendante de GPT-5.4, la normalisation directe OpenAI
  `openai-completions` -> `openai-responses`, les
  indications d’authentification conscientes de Codex, la suppression de Spark, les lignes de liste OpenAI synthétiques, et la politique
  de réflexion / modèle en direct de GPT-5 ; la famille de flux `openai-responses-defaults` possède les
  wrappers partagés natifs OpenAI Responses pour les en-têtes d’attribution,
  `/fast`/`serviceTier`, la verbosité du texte, la recherche web native Codex,
  la mise en forme de charge utile de compatibilité de raisonnement, et la gestion du contexte Responses.
- OpenRouter utilise `catalog` ainsi que `resolveDynamicModel` et
  `prepareDynamicModel` parce que le fournisseur est en transit et peut exposer de
  nouveaux ids de modèle avant la mise à jour du catalogue statique d’OpenClaw ; il utilise aussi
  `capabilities`, `wrapStreamFn` et `isCacheTtlEligible` pour garder hors du cœur
  les en-têtes de requête spécifiques au fournisseur, les métadonnées de routage, les correctifs de raisonnement et
  la politique de cache de prompt. Sa politique de rejeu provient de la
  famille `passthrough-gemini`, tandis que la famille de flux `openrouter-thinking`
  possède l’injection de raisonnement proxy et les sauts de modèle non pris en charge / `auto`.
- GitHub Copilot utilise `catalog`, `auth`, `resolveDynamicModel`, et
  `capabilities` ainsi que `prepareRuntimeAuth` et `fetchUsageSnapshot` parce qu’il
  a besoin d’une connexion par appareil appartenant au fournisseur, d’un comportement de repli de modèle, de
  particularités de transcription Claude, d’un échange de jeton GitHub -> jeton Copilot, et d’un point de terminaison d’usage possédé par le fournisseur.
- OpenAI Codex utilise `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth`, et `augmentModelCatalog` ainsi que
  `prepareExtraParams`, `resolveUsageAuth`, et `fetchUsageSnapshot` parce qu’il
  fonctionne toujours sur les transports OpenAI du cœur mais possède sa propre normalisation du transport/de la base URL,
  sa politique de repli d’actualisation OAuth, son choix de transport par défaut,
  ses lignes de catalogue Codex synthétiques, et l’intégration du point de terminaison d’usage ChatGPT ; il
  partage la même famille de flux `openai-responses-defaults` qu’OpenAI direct.
- Google AI Studio et Gemini CLI OAuth utilisent `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn`, et `isModernModelRef` parce que la
  famille de rejeu `google-gemini` possède le repli de compatibilité ascendante de Gemini 3.1,
  la validation native de rejeu Gemini, l’assainissement du rejeu bootstrap, le mode de
  sortie de raisonnement balisé, et la correspondance de modèle moderne, tandis que la
  famille de flux `google-thinking` possède la normalisation de charge utile de réflexion Gemini ;
  Gemini CLI OAuth utilise aussi `formatApiKey`, `resolveUsageAuth`, et
  `fetchUsageSnapshot` pour le formatage des jetons, l’analyse des jetons et le raccordement du point de terminaison de quota.
- Anthropic Vertex utilise `buildReplayPolicy` via la
  famille de rejeu `anthropic-by-model` afin que le nettoyage de rejeu spécifique à Claude reste
  limité aux ids Claude au lieu de tous les transports `anthropic-messages`.
- Amazon Bedrock utilise `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason`, et `resolveDefaultThinkingLevel` parce qu’il possède
  la classification spécifique à Bedrock des erreurs de limitation/pas prêt/dépassement de contexte
  pour le trafic Anthropic-sur-Bedrock ; sa politique de rejeu partage toujours la même
  garde `anthropic-by-model` réservée à Claude.
- OpenRouter, Kilocode, Opencode et Opencode Go utilisent `buildReplayPolicy`
  via la famille de rejeu `passthrough-gemini` parce qu’ils proxifient des modèles Gemini
  via des transports compatibles OpenAI et ont besoin de
  l’assainissement de signature de réflexion Gemini sans validation native du rejeu Gemini ni
  réécritures bootstrap.
- MiniMax utilise `buildReplayPolicy` via la
  famille de rejeu `hybrid-anthropic-openai` parce qu’un seul fournisseur possède à la fois
  la sémantique des messages Anthropic et la sémantique compatible OpenAI ; il conserve la suppression des
  blocs de réflexion réservée à Claude du côté Anthropic tout en rétablissant le mode de sortie
  de raisonnement en natif, et la famille de flux `minimax-fast-mode` possède les réécritures de modèle
  fast-mode sur le chemin de flux partagé.
- Moonshot utilise `catalog` ainsi que `wrapStreamFn` parce qu’il utilise toujours le
  transport OpenAI partagé mais a besoin d’une normalisation de charge utile de réflexion appartenant au fournisseur ; la
  famille de flux `moonshot-thinking` mappe la configuration ainsi que l’état `/think` sur sa
  charge utile native de réflexion binaire.
- Kilocode utilise `catalog`, `capabilities`, `wrapStreamFn`, et
  `isCacheTtlEligible` parce qu’il a besoin d’en-têtes de requête appartenant au fournisseur,
  de normalisation de charge utile de raisonnement, d’indications de transcription Gemini, et d’un contrôle
  Anthropic de TTL de cache ; la famille de flux `kilocode-thinking` conserve l’injection de réflexion Kilo
  sur le chemin de flux proxy partagé tout en ignorant `kilo/auto` et les autres ids de modèle proxy
  qui ne prennent pas en charge les charges utiles de raisonnement explicites.
- Z.AI utilise `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth`, et `fetchUsageSnapshot` parce qu’il possède le repli GLM-5,
  les valeurs par défaut `tool_stream`, l’UX de réflexion binaire, la correspondance de modèle moderne, et à la fois
  l’authentification d’usage + la récupération de quota ; la famille de flux `tool-stream-default-on`
  garde le wrapper `tool_stream` activé par défaut hors de la colle manuscrite par fournisseur.
- xAI utilise `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel`, et `isModernModelRef`
  parce qu’il possède la normalisation native du transport xAI Responses, les réécritures d’alias
  fast-mode Grok, `tool_stream` par défaut, le nettoyage strict-tool / charge utile de raisonnement,
  la réutilisation d’authentification de repli pour les outils appartenant au plugin, la résolution ascendante
  des modèles Grok, et les correctifs de compatibilité appartenant au fournisseur comme le profil
  de schéma d’outil xAI, les mots-clés de schéma non pris en charge, `web_search` natif, et le décodage
  des arguments d’appel d’outil avec entités HTML.
- Mistral, OpenCode Zen et OpenCode Go utilisent uniquement `capabilities` pour garder les
  particularités de transcription/outillage hors du cœur.
- Les fournisseurs bundled uniquement catalogue tels que `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway`, et `volcengine` utilisent
  uniquement `catalog`.
- Qwen utilise `catalog` pour son fournisseur de texte ainsi que des enregistrements partagés de compréhension des médias et
  de génération vidéo pour ses surfaces multimodales.
- MiniMax et Xiaomi utilisent `catalog` ainsi que les hooks d’usage parce que leur comportement `/usage`
  appartient au plugin même si l’inférence s’exécute toujours via les transports partagés.

## Helpers runtime

Les plugins peuvent accéder à certains helpers du cœur via `api.runtime`. Pour le TTS :

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

- `textToSpeech` renvoie la charge utile normale de sortie TTS du cœur pour les surfaces de fichier/note vocale.
- Utilise la configuration du cœur `messages.tts` et la sélection du fournisseur.
- Renvoie un buffer audio PCM + fréquence d’échantillonnage. Les plugins doivent rééchantillonner/encoder selon les fournisseurs.
- `listVoices` est facultatif selon le fournisseur. Utilisez-le pour les sélecteurs de voix appartenant au fournisseur ou les flux de configuration.
- Les listes de voix peuvent inclure des métadonnées plus riches comme la langue, le genre et des balises de personnalité pour des sélecteurs sensibles au fournisseur.
- OpenAI et ElevenLabs prennent aujourd’hui en charge la téléphonie. Microsoft non.

Les plugins peuvent aussi enregistrer des fournisseurs de voix via `api.registerSpeechProvider(...)`.

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

- Conservez dans le cœur la politique TTS, le repli et la livraison des réponses.
- Utilisez les fournisseurs de voix pour le comportement de synthèse appartenant au fournisseur.
- L’entrée legacy Microsoft `edge` est normalisée vers l’id de fournisseur `microsoft`.
- Le modèle de responsabilité préféré est orienté entreprise : un seul plugin fournisseur peut posséder
  le texte, la voix, l’image et de futurs fournisseurs de médias à mesure qu’OpenClaw ajoute ces
  contrats de capacité.

Pour la compréhension d’image/audio/vidéo, les plugins enregistrent un fournisseur typé unique
de compréhension des médias au lieu d’un sac clé/valeur générique :

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

- Conservez dans le cœur l’orchestration, le repli, la configuration et le câblage des canaux.
- Conservez le comportement fournisseur dans le plugin fournisseur.
- L’extension additive doit rester typée : nouvelles méthodes facultatives, nouveaux champs de résultat
  facultatifs, nouvelles capacités facultatives.
- La génération vidéo suit déjà le même modèle :
  - le cœur possède le contrat de capacité et le helper runtime
  - les plugins fournisseurs enregistrent `api.registerVideoGenerationProvider(...)`
  - les plugins de fonctionnalité/canal consomment `api.runtime.videoGeneration.*`

Pour les helpers runtime de compréhension des médias, les plugins peuvent appeler :

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

Pour la transcription audio, les plugins peuvent utiliser soit le runtime de compréhension des médias,
soit l’alias STT plus ancien :

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Remarques :

- `api.runtime.mediaUnderstanding.*` est la surface partagée privilégiée pour la
  compréhension d’image/audio/vidéo.
- Utilise la configuration audio de compréhension des médias du cœur (`tools.media.audio`) et l’ordre de repli du fournisseur.
- Renvoie `{ text: undefined }` lorsqu’aucune sortie de transcription n’est produite (par exemple en cas d’entrée ignorée/non prise en charge).
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

- `provider` et `model` sont des surcharges facultatives par exécution, et non des changements persistants de session.
- OpenClaw ne respecte ces champs de surcharge que pour les appelants approuvés.
- Pour les exécutions de repli appartenant au plugin, les opérateurs doivent activer cela explicitement avec `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Utilisez `plugins.entries.<id>.subagent.allowedModels` pour limiter les plugins approuvés à des cibles canoniques `provider/model` spécifiques, ou `"*"` pour autoriser explicitement toute cible.
- Les exécutions de sous-agent de plugins non approuvés fonctionnent toujours, mais les demandes de surcharge sont rejetées au lieu de retomber silencieusement sur autre chose.

Pour la recherche web, les plugins peuvent consommer le helper runtime partagé au lieu
d’accéder au câblage de l’outil d’agent :

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

Remarques :

- Conservez dans le cœur la sélection du fournisseur, la résolution des identifiants et la sémantique de requête partagée.
- Utilisez des fournisseurs de recherche web pour les transports de recherche spécifiques au fournisseur.
- `api.runtime.webSearch.*` est la surface partagée privilégiée pour les plugins de fonctionnalité/canal qui ont besoin d’un comportement de recherche sans dépendre du wrapper de l’outil d’agent.

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

- `generate(...)` : génère une image à l’aide de la chaîne configurée de fournisseurs de génération d’images.
- `listProviders(...)` : liste les fournisseurs de génération d’images disponibles et leurs capacités.

## Routes HTTP Gateway

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
- `auth` : requis. Utilisez `"gateway"` pour exiger l’authentification normale de la gateway, ou `"plugin"` pour une authentification/validation de Webhook gérée par le plugin.
- `match` : facultatif. `"exact"` (par défaut) ou `"prefix"`.
- `replaceExisting` : facultatif. Permet au même plugin de remplacer son propre enregistrement de route existant.
- `handler` : renvoie `true` lorsque la route a traité la requête.

Remarques :

- `api.registerHttpHandler(...)` a été supprimé et provoquera une erreur de chargement du plugin. Utilisez `api.registerHttpRoute(...)` à la place.
- Les routes de plugin doivent déclarer explicitement `auth`.
- Les conflits exacts `path + match` sont rejetés sauf si `replaceExisting: true`, et un plugin ne peut pas remplacer la route d’un autre plugin.
- Les routes qui se chevauchent avec des niveaux `auth` différents sont rejetées. Gardez les chaînes de retombée `exact`/`prefix` au même niveau d’authentification uniquement.
- Les routes `auth: "plugin"` ne reçoivent **pas** automatiquement de portées runtime d’opérateur. Elles servent aux webhooks gérés par le plugin / à la vérification de signature, et non à des appels helper Gateway privilégiés.
- Les routes `auth: "gateway"` s’exécutent dans une portée runtime de requête Gateway, mais cette portée est volontairement prudente :
  - l’authentification bearer par secret partagé (`gateway.auth.mode = "token"` / `"password"`) maintient les portées runtime des routes de plugin épinglées à `operator.write`, même si l’appelant envoie `x-openclaw-scopes`
  - les modes HTTP approuvés portant une identité (par exemple `trusted-proxy` ou `gateway.auth.mode = "none"` sur une ingress privée) ne respectent `x-openclaw-scopes` que lorsque l’en-tête est explicitement présent
  - si `x-openclaw-scopes` est absent sur ces requêtes de route de plugin portant une identité, la portée runtime retombe sur `operator.write`
- Règle pratique : ne supposez pas qu’une route de plugin authentifiée par gateway est implicitement une surface d’administration. Si votre route a besoin d’un comportement réservé à l’administration, exigez un mode d’authentification portant une identité et documentez le contrat explicite de l’en-tête `x-openclaw-scopes`.

## Chemins d’import du Plugin SDK

Utilisez les sous-chemins du SDK au lieu de l’import monolithique `openclaw/plugin-sdk` lorsque
vous créez des plugins :

- `openclaw/plugin-sdk/plugin-entry` pour les primitives d’enregistrement de plugin.
- `openclaw/plugin-sdk/core` pour le contrat générique partagé orienté plugin.
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
  `openclaw/plugin-sdk/secret-input`, et
  `openclaw/plugin-sdk/webhook-ingress` pour le câblage partagé
  de configuration/authentification/réponse/Webhook. `channel-inbound` est l’emplacement partagé pour l’anti-rebond, la correspondance des mentions,
  les helpers de politique de mention entrante, le formatage des enveloppes et les
  helpers de contexte d’enveloppe entrante.
  `channel-setup` est la couture étroite de configuration avec installation facultative.
  `setup-runtime` est la surface de configuration sûre à l’exécution utilisée par `setupEntry` /
  le démarrage différé, y compris les adaptateurs de patch de configuration sûrs à importer.
  `setup-adapter-runtime` est la couture d’adaptateur de configuration de compte sensible à l’environnement.
  `setup-tools` est la petite couture d’helpers CLI/archive/docs (`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- Sous-chemins de domaine tels que `openclaw/plugin-sdk/channel-config-helpers`,
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
  `openclaw/plugin-sdk/directory-runtime` pour les helpers partagés d’exécution/configuration.
  `telegram-command-config` est la couture publique étroite pour la normalisation/validation des commandes personnalisées Telegram et reste disponible même si la surface de contrat Telegram bundled est temporairement indisponible.
  `text-runtime` est la couture partagée de texte/Markdown/journalisation, y compris
  la suppression de texte visible par l’assistant, les helpers de rendu/segmentation Markdown, les helpers de
  rédaction, les helpers de balises de directive et les utilitaires de texte sûr.
- Les coutures de canal spécifiques à l’approbation doivent préférer un seul contrat
  `approvalCapability` sur le plugin. Le cœur lit alors l’authentification d’approbation, la livraison, le rendu,
  le routage natif et le comportement paresseux du gestionnaire natif via cette seule capacité
  au lieu de mélanger le comportement d’approbation dans des champs de plugin non liés.
- `openclaw/plugin-sdk/channel-runtime` est obsolète et ne reste présent que comme
  shim de compatibilité pour les anciens plugins. Le nouveau code doit importer les primitives
  génériques plus étroites à la place, et le code du dépôt ne doit pas ajouter de nouveaux imports du
  shim.
- Les internes d’extensions bundled restent privés. Les plugins externes ne doivent utiliser que
  les sous-chemins `openclaw/plugin-sdk/*`. Le code cœur/test d’OpenClaw peut utiliser les
  points d’entrée publics du dépôt sous une racine de paquet de plugin tels que `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js`, et des fichiers à portée étroite tels que
  `login-qr-api.js`. N’importez jamais le `src/*` d’un paquet de plugin depuis le cœur ou depuis
  une autre extension.
- Découpage des points d’entrée du dépôt :
  `<plugin-package-root>/api.js` est le barrel de helpers/types,
  `<plugin-package-root>/runtime-api.js` est le barrel runtime uniquement,
  `<plugin-package-root>/index.js` est le point d’entrée du plugin bundled,
  et `<plugin-package-root>/setup-entry.js` est le point d’entrée du plugin de configuration.
- Exemples actuels de fournisseurs bundled :
  - Anthropic utilise `api.js` / `contract-api.js` pour les helpers de flux Claude tels
    que `wrapAnthropicProviderStream`, les helpers d’en-tête bêta, et l’analyse de `service_tier`.
  - OpenAI utilise `api.js` pour les builders de fournisseur, les helpers de modèle par défaut, et
    les builders de fournisseur temps réel.
  - OpenRouter utilise `api.js` pour son builder de fournisseur ainsi que pour les helpers
    d’intégration/configuration, tandis que `register.runtime.js` peut encore réexporter des helpers génériques
    `plugin-sdk/provider-stream` pour un usage local au dépôt.
- Les points d’entrée publics chargés par façade préfèrent l’instantané actif de configuration runtime
  lorsqu’il existe, puis retombent sur le fichier de configuration résolu sur disque lorsqu’OpenClaw ne sert pas encore d’instantané runtime.
- Les primitives partagées génériques restent le contrat public privilégié du SDK. Un petit
  ensemble réservé de compatibilité de coutures d’helpers de canal marquées bundled existe encore. Traitez-les comme des coutures de maintenance bundled/compatibilité, et non comme de nouvelles cibles d’import tierces ; les nouveaux contrats inter-canaux doivent toujours être placés sur des sous-chemins génériques `plugin-sdk/*` ou sur les barrels locaux au plugin `api.js` /
  `runtime-api.js`.

Remarque de compatibilité :

- Évitez le barrel racine `openclaw/plugin-sdk` pour le nouveau code.
- Préférez d’abord les primitives stables et étroites. Les nouveaux sous-chemins de configuration/appairage/réponse/
  feedback/contrat/entrant/threading/commande/secret-input/Webhook/infra/
  allowlist/statut/message-tool constituent le contrat visé pour le nouveau
  travail sur les plugins bundled et externes.
  L’analyse/la correspondance des cibles appartient à `openclaw/plugin-sdk/channel-targets`.
  Les garde-fous d’action de message et les helpers d’id de message de réaction appartiennent à
  `openclaw/plugin-sdk/channel-actions`.
- Les barrels d’helpers spécifiques à une extension bundled ne sont pas stables par défaut. Si un
  helper n’est nécessaire que pour une extension bundled, gardez-le derrière la couture locale
  `api.js` ou `runtime-api.js` de l’extension au lieu de le promouvoir dans
  `openclaw/plugin-sdk/<extension>`.
- Les nouvelles coutures d’helpers partagés doivent être génériques, pas marquées par canal. L’analyse
  partagée des cibles appartient à `openclaw/plugin-sdk/channel-targets` ; les internes spécifiques à un canal
  restent derrière la couture locale `api.js` ou `runtime-api.js` du plugin propriétaire.
- Les sous-chemins spécifiques à une capacité tels que `image-generation`,
  `media-understanding` et `speech` existent parce que les plugins bundled/natifs les utilisent
  aujourd’hui. Leur présence ne signifie pas à elle seule que chaque helper exporté est un contrat externe figé à long terme.

## Schémas de l’outil message

Les plugins doivent posséder les contributions de schéma `describeMessageTool(...)`
spécifiques au canal. Conservez les champs spécifiques au fournisseur dans le plugin, pas dans le cœur partagé.

Pour les fragments de schéma portables partagés, réutilisez les helpers génériques exportés via
`openclaw/plugin-sdk/channel-actions` :

- `createMessageToolButtonsSchema()` pour les charges utiles de style grille de boutons
- `createMessageToolCardSchema()` pour les charges utiles de carte structurée

Si une forme de schéma n’a de sens que pour un seul fournisseur, définissez-la dans les
sources propres à ce plugin au lieu de la promouvoir dans le SDK partagé.

## Résolution de cible de canal

Les plugins de canal doivent posséder la sémantique des cibles spécifique au canal. Gardez l’hôte sortant partagé
générique et utilisez la surface d’adaptateur de messagerie pour les règles fournisseur :

- `messaging.inferTargetChatType({ to })` décide si une cible normalisée
  doit être traitée comme `direct`, `group` ou `channel` avant la recherche dans l’annuaire.
- `messaging.targetResolver.looksLikeId(raw, normalized)` indique au cœur si une
  entrée doit passer directement à une résolution de type id au lieu d’une recherche dans l’annuaire.
- `messaging.targetResolver.resolveTarget(...)` est le repli du plugin lorsque le
  cœur a besoin d’une résolution finale appartenant au fournisseur après normalisation ou après un échec de recherche
  dans l’annuaire.
- `messaging.resolveOutboundSessionRoute(...)` possède la construction spécifique au fournisseur de la route de session
  sortante une fois une cible résolue.

Répartition recommandée :

- Utilisez `inferTargetChatType` pour les décisions de catégorie qui doivent être prises avant
  la recherche dans les pairs/groupes.
- Utilisez `looksLikeId` pour les vérifications de type « traiter ceci comme un id de cible explicite/natif ».
- Utilisez `resolveTarget` pour le repli de normalisation spécifique au fournisseur, pas pour une
  recherche large dans l’annuaire.
- Conservez les ids natifs du fournisseur comme les ids de chat, ids de thread, JID, handles et ids de room
  dans les valeurs `target` ou les paramètres spécifiques au fournisseur, et non dans des champs génériques du SDK.

## Annuaires adossés à la configuration

Les plugins qui dérivent des entrées d’annuaire à partir de la configuration doivent garder cette logique dans le
plugin et réutiliser les helpers partagés de
`openclaw/plugin-sdk/directory-runtime`.

Utilisez cela lorsqu’un canal a besoin de pairs/groupes adossés à la configuration, par exemple :

- pairs de message privé pilotés par allowlist
- cartes configurées de canaux/groupes
- replis d’annuaire statiques à portée de compte

Les helpers partagés de `directory-runtime` ne gèrent que les opérations génériques :

- filtrage des requêtes
- application des limites
- helpers de déduplication/normalisation
- construction de `ChannelDirectoryEntry[]`

L’inspection de compte spécifique au canal et la normalisation d’id doivent rester dans l’implémentation du
plugin.

## Catalogues de fournisseurs

Les plugins fournisseurs peuvent définir des catalogues de modèles pour l’inférence avec
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` renvoie la même forme que celle qu’OpenClaw écrit dans
`models.providers` :

- `{ provider }` pour une entrée de fournisseur
- `{ providers }` pour plusieurs entrées de fournisseur

Utilisez `catalog` lorsque le plugin possède des ids de modèle spécifiques au fournisseur, des valeurs par défaut de base URL, ou des métadonnées de modèle protégées par authentification.

`catalog.order` contrôle le moment où le catalogue d’un plugin fusionne par rapport aux
fournisseurs implicites intégrés d’OpenClaw :

- `simple` : fournisseurs simples pilotés par clé API ou environnement
- `profile` : fournisseurs qui apparaissent lorsque des profils d’authentification existent
- `paired` : fournisseurs qui synthétisent plusieurs entrées de fournisseur liées
- `late` : dernier passage, après les autres fournisseurs implicites

Les fournisseurs plus tardifs l’emportent en cas de collision de clé, donc les plugins peuvent intentionnellement remplacer une entrée de fournisseur intégrée avec le même id de fournisseur.

Compatibilité :

- `discovery` fonctionne toujours comme alias legacy
- si `catalog` et `discovery` sont tous deux enregistrés, OpenClaw utilise `catalog`

## Inspection en lecture seule des canaux

Si votre plugin enregistre un canal, préférez l’implémentation de
`plugin.config.inspectAccount(cfg, accountId)` en complément de `resolveAccount(...)`.

Pourquoi :

- `resolveAccount(...)` est le chemin runtime. Il peut supposer que les identifiants
  sont entièrement matérialisés et peut échouer rapidement lorsque les secrets requis sont absents.
- Les chemins de commande en lecture seule tels que `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve`, et les flux doctor/réparation
  de configuration ne devraient pas avoir besoin de matérialiser les identifiants runtime juste pour
  décrire la configuration.

Comportement recommandé pour `inspectAccount(...)` :

- Retournez uniquement un état descriptif du compte.
- Préservez `enabled` et `configured`.
- Incluez les champs source/statut des identifiants lorsque c’est pertinent, par exemple :
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Vous n’avez pas besoin de renvoyer les valeurs brutes des jetons simplement pour signaler une disponibilité en lecture seule. Renvoyer `tokenStatus: "available"` (et le champ source correspondant) suffit pour des commandes de type statut.
- Utilisez `configured_unavailable` lorsqu’un identifiant est configuré via SecretRef mais indisponible dans le chemin de commande courant.

Cela permet aux commandes en lecture seule d’indiquer « configuré mais indisponible dans ce chemin de commande »
au lieu de planter ou de signaler à tort que le compte n’est pas configuré.

## Packs de paquets

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

Chaque entrée devient un plugin. Si le pack liste plusieurs extensions, l’id du plugin
devient `name/<fileBase>`.

Si votre plugin importe des dépendances npm, installez-les dans ce répertoire afin que
`node_modules` soit disponible (`npm install` / `pnpm install`).

Garde-fou de sécurité : chaque entrée `openclaw.extensions` doit rester dans le répertoire du plugin
après résolution des liens symboliques. Les entrées qui sortent du répertoire du paquet sont
rejetées.

Note de sécurité : `openclaw plugins install` installe les dépendances du plugin avec
`npm install --omit=dev --ignore-scripts` (pas de scripts de cycle de vie, pas de dépendances de développement à l’exécution). Gardez les arbres de dépendances de plugin en « JS/TS pur » et évitez les paquets qui nécessitent des builds `postinstall`.

Facultatif : `openclaw.setupEntry` peut pointer vers un module léger réservé à la configuration.
Lorsque OpenClaw a besoin des surfaces de configuration pour un plugin de canal désactivé, ou
lorsqu’un plugin de canal est activé mais pas encore configuré, il charge `setupEntry`
au lieu du point d’entrée complet du plugin. Cela allège le démarrage et la configuration
lorsque votre point d’entrée principal câble aussi des outils, hooks ou autre code réservé au runtime.

Facultatif : `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
peut faire entrer un plugin de canal dans le même chemin `setupEntry` pendant la phase de
démarrage pré-écoute de la gateway, même lorsque le canal est déjà configuré.

N’utilisez cela que si `setupEntry` couvre entièrement la surface de démarrage qui doit exister
avant que la gateway commence à écouter. En pratique, cela signifie que le point d’entrée de configuration
doit enregistrer toutes les capacités appartenant au canal dont le démarrage dépend, telles que :

- l’enregistrement du canal lui-même
- toute route HTTP devant être disponible avant que la gateway commence à écouter
- toutes méthodes Gateway, outils ou services qui doivent exister pendant cette même fenêtre

Si votre point d’entrée complet possède encore une capacité de démarrage requise, n’activez pas
ce drapeau. Conservez le comportement par défaut du plugin et laissez OpenClaw charger le point d’entrée complet au démarrage.

Les canaux bundled peuvent aussi publier des helpers de surface de contrat réservés à la configuration que le cœur
peut consulter avant que le runtime complet du canal ne soit chargé. La surface actuelle de promotion de configuration est :

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Le cœur utilise cette surface lorsqu’il doit promouvoir une configuration legacy de canal à compte unique
dans `channels.<id>.accounts.*` sans charger le point d’entrée complet du plugin.
Matrix est l’exemple bundled actuel : il ne déplace que les clés d’authentification/bootstrap dans un
compte promu nommé lorsque des comptes nommés existent déjà, et il peut préserver une clé de compte par défaut non canonique configurée au lieu de toujours créer
`accounts.default`.

Ces adaptateurs de patch de configuration gardent la découverte de la surface de contrat bundled paresseuse. Le temps d’import reste léger ; la surface de promotion n’est chargée qu’au premier usage au lieu de réentrer dans le démarrage du canal bundled lors de l’import du module.

Lorsque ces surfaces de démarrage incluent des méthodes RPC Gateway, gardez-les sur un préfixe
spécifique au plugin. Les espaces de noms d’administration du cœur (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) restent réservés et se résolvent toujours
vers `operator.admin`, même si un plugin demande une portée plus étroite.

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

Les plugins de canal peuvent annoncer des métadonnées de configuration/découverte via `openclaw.channel` et
des indications d’installation via `openclaw.install`. Cela garde les données du catalogue de cœur vides.

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
- `preferOver` : ids de plugin/canal de priorité plus basse que cette entrée de catalogue doit dépasser
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras` : contrôles de copie de la surface de sélection
- `markdownCapable` : marque le canal comme compatible Markdown pour les décisions de formatage sortant
- `exposure.configured` : masque le canal des surfaces de liste de canaux configurés lorsqu’il est défini à `false`
- `exposure.setup` : masque le canal des sélecteurs interactifs de configuration lorsque défini à `false`
- `exposure.docs` : marque le canal comme interne/privé pour les surfaces de navigation documentaire
- `showConfigured` / `showInSetup` : alias legacy toujours acceptés pour compatibilité ; préférez `exposure`
- `quickstartAllowFrom` : fait entrer le canal dans le flux standard `allowFrom` du démarrage rapide
- `forceAccountBinding` : exige une liaison explicite de compte même lorsqu’un seul compte existe
- `preferSessionLookupForAnnounceTarget` : préfère la recherche de session lors de la résolution des cibles d’annonce

OpenClaw peut aussi fusionner des **catalogues de canaux externes** (par exemple, une exportation de registre MPM). Déposez un fichier JSON à l’un des emplacements suivants :

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Ou faites pointer `OPENCLAW_PLUGIN_CATALOG_PATHS` (ou `OPENCLAW_MPM_CATALOG_PATHS`) vers
un ou plusieurs fichiers JSON (délimités par virgule/point-virgule/`PATH`). Chaque fichier doit
contenir `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. L’analyseur accepte aussi `"packages"` ou `"plugins"` comme alias legacy pour la clé `"entries"`.

## Plugins de moteur de contexte

Les plugins de moteur de contexte possèdent l’orchestration du contexte de session pour l’ingestion, l’assemblage
et la compaction. Enregistrez-les depuis votre plugin avec
`api.registerContextEngine(id, factory)`, puis sélectionnez le moteur actif avec
`plugins.slots.contextEngine`.

Utilisez cela lorsque votre plugin doit remplacer ou étendre le pipeline de contexte par défaut
plutôt que simplement ajouter une recherche mémoire ou des hooks.

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
implémenté et déléguez-le explicitement :

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
le système de plugins avec un accès privé. Ajoutez la capacité manquante.

Séquence recommandée :

1. définir le contrat du cœur
   Décidez quel comportement partagé le cœur doit posséder : politique, repli, fusion de configuration,
   cycle de vie, sémantique orientée canal, et forme du helper runtime.
2. ajouter des surfaces typées d’enregistrement/runtime de plugin
   Étendez `OpenClawPluginApi` et/ou `api.runtime` avec la plus petite surface de capacité typée utile.
3. raccorder les consommateurs cœur + canal/fonctionnalité
   Les canaux et plugins de fonctionnalité doivent consommer la nouvelle capacité via le cœur,
   et non en important directement une implémentation fournisseur.
4. enregistrer les implémentations fournisseur
   Les plugins fournisseurs enregistrent ensuite leurs backends sur la capacité.
5. ajouter une couverture de contrat
   Ajoutez des tests afin que la responsabilité et la forme d’enregistrement restent explicites dans le temps.

C’est ainsi qu’OpenClaw reste affirmé sans devenir codé en dur selon la vision du monde d’un
seul fournisseur. Voir le [Capability Cookbook](/fr/plugins/architecture)
pour une checklist concrète de fichiers et un exemple détaillé.

### Checklist de capacité

Lorsque vous ajoutez une nouvelle capacité, l’implémentation doit généralement toucher ces
surfaces ensemble :

- types de contrat du cœur dans `src/<capability>/types.ts`
- helper runner/runtime du cœur dans `src/<capability>/runtime.ts`
- surface d’enregistrement de l’API des plugins dans `src/plugins/types.ts`
- câblage du registre de plugins dans `src/plugins/registry.ts`
- exposition runtime du plugin dans `src/plugins/runtime/*` lorsque les plugins de fonctionnalité/canal
  doivent la consommer
- helpers de capture/test dans `src/test-utils/plugin-registration.ts`
- assertions de responsabilité/contrat dans `src/plugins/contracts/registry.ts`
- documentation opérateur/plugin dans `docs/`

Si l’une de ces surfaces manque, c’est généralement le signe que la capacité n’est
pas encore entièrement intégrée.

### Modèle de capacité

Modèle minimal :

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

Modèle de test de contrat :

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

Cela garde la règle simple :

- le cœur possède le contrat de capacité + l’orchestration
- les plugins fournisseurs possèdent les implémentations fournisseur
- les plugins de fonctionnalité/canal consomment les helpers runtime
- les tests de contrat gardent la responsabilité explicite
