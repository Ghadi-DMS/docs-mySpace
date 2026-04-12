---
read_when:
    - Exécuter les tests en local ou dans la CI
    - Ajouter des tests de régression pour les bugs de modèle/fournisseur
    - Déboguer le comportement de la passerelle + de l'agent
summary: 'Kit de test : suites unitaires/e2e/live, exécuteurs Docker et ce que couvre chaque test'
title: Tests
x-i18n:
    generated_at: "2026-04-12T23:28:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: a66ea672c386094ab4a8035a082c8a85d508a14301ad44b628d2a10d9cec3a52
    source_path: help/testing.md
    workflow: 15
---

# Tests

OpenClaw dispose de trois suites Vitest (unitaire/intégration, e2e, live) et d'un petit ensemble d'exécuteurs Docker.

Cette documentation est un guide « comment nous testons » :

- Ce que couvre chaque suite (et ce qu'elle ne couvre délibérément _pas_)
- Quelles commandes exécuter pour les workflows courants (local, avant push, débogage)
- Comment les tests live détectent les identifiants et sélectionnent les modèles/fournisseurs
- Comment ajouter des tests de régression pour des problèmes réels de modèle/fournisseur

## Démarrage rapide

La plupart du temps :

- Gate complet (attendu avant push) : `pnpm build && pnpm check && pnpm test`
- Exécution locale plus rapide de la suite complète sur une machine bien dimensionnée : `pnpm test:max`
- Boucle de surveillance Vitest directe : `pnpm test:watch`
- Le ciblage direct de fichiers prend désormais aussi en charge les chemins d'extension/de canal : `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Préférez d'abord les exécutions ciblées lorsque vous itérez sur un seul échec.
- Site QA adossé à Docker : `pnpm qa:lab:up`
- Voie QA adossée à une VM Linux : `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

Quand vous modifiez les tests ou souhaitez davantage de confiance :

- Gate de couverture : `pnpm test:coverage`
- Suite E2E : `pnpm test:e2e`

Lors du débogage de fournisseurs/modèles réels (nécessite de vrais identifiants) :

- Suite live (modèles + sondes d'outils/images de la passerelle) : `pnpm test:live`
- Cibler silencieusement un seul fichier live : `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Conseil : lorsque vous n'avez besoin que d'un seul cas en échec, préférez restreindre les tests live via les variables d'environnement de liste d'autorisation décrites ci-dessous.

## Exécuteurs spécifiques à la QA

Ces commandes s'ajoutent aux suites de tests principales lorsque vous avez besoin du réalisme de qa-lab :

- `pnpm openclaw qa suite`
  - Exécute directement sur l'hôte des scénarios QA adossés au dépôt.
  - Exécute plusieurs scénarios sélectionnés en parallèle par défaut avec des workers de passerelle isolés, jusqu'à 64 workers ou au nombre de scénarios sélectionnés. Utilisez `--concurrency <count>` pour ajuster le nombre de workers, ou `--concurrency 1` pour l'ancienne voie sérielle.
- `pnpm openclaw qa suite --runner multipass`
  - Exécute la même suite QA dans une VM Linux Multipass jetable.
  - Conserve le même comportement de sélection de scénarios que `qa suite` sur l'hôte.
  - Réutilise les mêmes indicateurs de sélection de fournisseur/modèle que `qa suite`.
  - Les exécutions live transmettent les entrées d'authentification QA prises en charge et pratiques pour l'invité : les clés de fournisseur basées sur l'environnement, le chemin de configuration du fournisseur live QA, et `CODEX_HOME` lorsqu'il est présent.
  - Les répertoires de sortie doivent rester sous la racine du dépôt afin que l'invité puisse réécrire via l'espace de travail monté.
  - Écrit le rapport QA normal + le résumé ainsi que les journaux Multipass sous `.artifacts/qa-e2e/...`.
- `pnpm qa:lab:up`
  - Démarre le site QA adossé à Docker pour un travail QA de type opérateur.
- `pnpm openclaw qa matrix`
  - Exécute la voie QA Matrix live sur un homeserver Tuwunel jetable adossé à Docker.
  - Provisionne trois utilisateurs Matrix temporaires (`driver`, `sut`, `observer`) ainsi qu'une salle privée, puis démarre un processus enfant de passerelle QA avec le vrai Plugin Matrix comme transport SUT.
  - Utilise par défaut l'image Tuwunel stable épinglée `ghcr.io/matrix-construct/tuwunel:v1.5.1`. Remplacez-la avec `OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE` lorsque vous devez tester une autre image.
  - Écrit un rapport QA Matrix, un résumé et un artefact observed-events sous `.artifacts/qa-e2e/...`.
- `pnpm openclaw qa telegram`
  - Exécute la voie QA Telegram live sur un vrai groupe privé à l'aide des jetons de bot driver et SUT issus de l'environnement.
  - Nécessite `OPENCLAW_QA_TELEGRAM_GROUP_ID`, `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` et `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`. L'identifiant du groupe doit être l'identifiant numérique du chat Telegram.
  - Nécessite deux bots distincts dans le même groupe privé, le bot SUT exposant un nom d'utilisateur Telegram.
  - Pour une observation stable bot-à-bot, activez le Bot-to-Bot Communication Mode dans `@BotFather` pour les deux bots et assurez-vous que le bot driver peut observer le trafic des bots du groupe.
  - Écrit un rapport QA Telegram, un résumé et un artefact observed-messages sous `.artifacts/qa-e2e/...`.

Les voies de transport live partagent un contrat standard afin que les nouveaux transports ne divergent pas :

`qa-channel` reste la suite QA synthétique générale et ne fait pas partie de la matrice de couverture des transports live.

| Voie     | Canary | Contrôle des mentions | Blocage par liste d'autorisation | Réponse de premier niveau | Reprise après redémarrage | Suivi de thread | Isolation des threads | Observation des réactions | Commande d'aide |
| -------- | ------ | --------------------- | -------------------------------- | ------------------------- | ------------------------- | --------------- | --------------------- | ------------------------- | --------------- |
| Matrix   | x      | x                     | x                                | x                         | x                         | x               | x                     | x                         |                 |
| Telegram | x      |                       |                                  |                           |                           |                 |                       |                           | x               |

### Ajouter un canal à la QA

L'ajout d'un canal au système QA Markdown nécessite exactement deux éléments :

1. Un adaptateur de transport pour le canal.
2. Un pack de scénarios qui exerce le contrat du canal.

N'ajoutez pas d'exécuteur QA spécifique à un canal lorsque l'exécuteur partagé `qa-lab` peut prendre en charge le flux.

`qa-lab` gère les mécanismes partagés :

- démarrage et arrêt de la suite
- concurrence des workers
- écriture des artefacts
- génération de rapports
- exécution des scénarios
- alias de compatibilité pour les anciens scénarios `qa-channel`

L'adaptateur de canal gère le contrat de transport :

- comment la passerelle est configurée pour ce transport
- comment l'état de préparation est vérifié
- comment les événements entrants sont injectés
- comment les messages sortants sont observés
- comment les transcriptions et l'état de transport normalisé sont exposés
- comment les actions adossées au transport sont exécutées
- comment la réinitialisation ou le nettoyage spécifique au transport est géré

Le seuil minimal d'adoption pour un nouveau canal est le suivant :

1. Implémenter l'adaptateur de transport sur l'interface partagée `qa-lab`.
2. Enregistrer l'adaptateur dans le registre de transport.
3. Conserver les mécanismes spécifiques au transport dans l'adaptateur ou le harnais du canal.
4. Rédiger ou adapter les scénarios Markdown sous `qa/scenarios/`.
5. Utiliser les helpers de scénarios génériques pour les nouveaux scénarios.
6. Maintenir les alias de compatibilité existants, sauf si le dépôt effectue une migration intentionnelle.

La règle de décision est stricte :

- Si un comportement peut être exprimé une seule fois dans `qa-lab`, placez-le dans `qa-lab`.
- Si un comportement dépend d'un transport de canal, conservez-le dans cet adaptateur ou ce harnais de plugin.
- Si un scénario a besoin d'une nouvelle capacité que plus d'un canal peut utiliser, ajoutez un helper générique plutôt qu'une branche spécifique à un canal dans `suite.ts`.
- Si un comportement n'a de sens que pour un seul transport, conservez le scénario spécifique à ce transport et explicitez-le dans le contrat du scénario.

Les noms de helpers génériques préférés pour les nouveaux scénarios sont :

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Des alias de compatibilité restent disponibles pour les scénarios existants, notamment :

- `waitForQaChannelReady`
- `waitForOutboundMessage`
- `waitForNoOutbound`
- `formatConversationTranscript`
- `resetBus`

Les nouveaux travaux sur les canaux doivent utiliser les noms de helpers génériques.
Les alias de compatibilité existent pour éviter une migration en une seule étape, pas comme modèle pour
la rédaction de nouveaux scénarios.

## Suites de tests (ce qui s'exécute où)

Considérez les suites comme offrant un « réalisme croissant » (et une instabilité/un coût croissants) :

### Unitaire / intégration (par défaut)

- Commande : `pnpm test`
- Configuration : dix exécutions de shards séquentielles (`vitest.full-*.config.ts`) sur les projets Vitest ciblés existants
- Fichiers : inventaires core/unit sous `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts`, ainsi que les tests Node `ui` autorisés couverts par `vitest.unit.config.ts`
- Portée :
  - Tests unitaires purs
  - Tests d'intégration en processus (authentification de la passerelle, routage, outillage, parsing, configuration)
  - Régressions déterministes pour des bugs connus
- Attentes :
  - S'exécute dans la CI
  - Aucune vraie clé requise
  - Doit être rapide et stable
- Note sur les projets :
  - `pnpm test` sans ciblage exécute désormais onze configurations de shards plus petites (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) au lieu d'un seul processus géant du projet racine natif. Cela réduit le pic de RSS sur les machines chargées et évite que le travail auto-reply/extension n'affame les suites non liées.
  - `pnpm test --watch` utilise toujours le graphe de projets natif de `vitest.config.ts`, car une boucle watch multi-shard n'est pas pratique.
  - `pnpm test`, `pnpm test:watch` et `pnpm test:perf:imports` dirigent d'abord les cibles explicites de fichier/répertoire via des voies ciblées, de sorte que `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` évite de payer le coût de démarrage complet du projet racine.
  - `pnpm test:changed` développe les chemins git modifiés vers les mêmes voies ciblées lorsque le diff ne touche que des fichiers source/test routables ; les modifications de config/setup reviennent toujours à la réexécution large du projet racine.
  - Les tests unitaires à import léger provenant des agents, commandes, plugins, helpers auto-reply, `plugin-sdk` et zones utilitaires pures similaires passent par la voie `unit-fast`, qui ignore `test/setup-openclaw-runtime.ts` ; les fichiers stateful/lourds à l'exécution restent sur les voies existantes.
  - Certains fichiers source helpers `plugin-sdk` et `commands` sélectionnés mappent aussi les exécutions en mode changed vers des tests frères explicites dans ces voies légères, afin que les modifications de helpers évitent de relancer toute la suite lourde de ce répertoire.
  - `auto-reply` dispose désormais de trois buckets dédiés : les helpers core de premier niveau, les tests d'intégration `reply.*` de premier niveau, et le sous-arbre `src/auto-reply/reply/**`. Cela garde le travail le plus lourd du harnais de réponse hors des tests bon marché de statut/chunk/token.
- Note sur l'embedded runner :
  - Lorsque vous modifiez les entrées de découverte des outils de message ou le contexte d'exécution de Compaction,
    conservez les deux niveaux de couverture.
  - Ajoutez des régressions ciblées sur les helpers pour les frontières pures de routage/normalisation.
  - Maintenez aussi en bon état les suites d'intégration de l'embedded runner :
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`, et
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Ces suites vérifient que les ids ciblés et le comportement de Compaction continuent bien de passer
    par les vrais chemins `run.ts` / `compact.ts` ; des tests sur helpers seuls ne sont pas un
    substitut suffisant à ces chemins d'intégration.
- Note sur le pool :
  - La configuration Vitest de base utilise désormais `threads` par défaut.
  - La configuration Vitest partagée fixe aussi `isolate: false` et utilise l'exécuteur non isolé sur les projets racine, les configurations e2e et live.
  - La voie UI racine conserve sa configuration `jsdom` et son optimiseur, mais s'exécute maintenant aussi sur l'exécuteur partagé non isolé.
  - Chaque shard `pnpm test` hérite des mêmes valeurs par défaut `threads` + `isolate: false` depuis la configuration Vitest partagée.
  - Le lanceur partagé `scripts/run-vitest.mjs` ajoute désormais aussi `--no-maglev` par défaut aux processus Node enfants de Vitest afin de réduire le churn de compilation V8 pendant les grosses exécutions locales. Définissez `OPENCLAW_VITEST_ENABLE_MAGLEV=1` si vous devez comparer avec le comportement V8 standard.
- Note sur l'itération locale rapide :
  - `pnpm test:changed` passe par des voies ciblées lorsque les chemins modifiés correspondent proprement à une suite plus petite.
  - `pnpm test:max` et `pnpm test:changed:max` conservent le même comportement de routage, simplement avec une limite de workers plus élevée.
  - L'auto-scaling local des workers est désormais volontairement conservateur et réduit aussi la cadence lorsque la charge moyenne de l'hôte est déjà élevée, afin que plusieurs exécutions Vitest concurrentes causent moins de dégâts par défaut.
  - La configuration Vitest de base marque les projets/fichiers de configuration comme `forceRerunTriggers` afin que les réexécutions en mode changed restent correctes lorsque le câblage des tests change.
  - La configuration conserve `OPENCLAW_VITEST_FS_MODULE_CACHE` activé sur les hôtes pris en charge ; définissez `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` si vous voulez un emplacement de cache explicite pour un profilage direct.
- Note de débogage des performances :
  - `pnpm test:perf:imports` active les rapports de durée d'import Vitest ainsi que la sortie de détail des imports.
  - `pnpm test:perf:imports:changed` limite la même vue de profilage aux fichiers modifiés depuis `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` compare le `test:changed` routé avec le chemin natif du projet racine pour ce diff validé et affiche le temps mur ainsi que le RSS max sur macOS.
- `pnpm test:perf:changed:bench -- --worktree` mesure l'arbre de travail modifié actuel en routant la liste des fichiers modifiés via `scripts/test-projects.mjs` et la configuration Vitest racine.
  - `pnpm test:perf:profile:main` écrit un profil CPU du thread principal pour le démarrage Vitest/Vite et la surcharge de transformation.
  - `pnpm test:perf:profile:runner` écrit des profils CPU+heap de l'exécuteur pour la suite unitaire avec le parallélisme de fichiers désactivé.

### E2E (smoke de la passerelle)

- Commande : `pnpm test:e2e`
- Configuration : `vitest.e2e.config.ts`
- Fichiers : `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Valeurs par défaut à l'exécution :
  - Utilise Vitest `threads` avec `isolate: false`, en cohérence avec le reste du dépôt.
  - Utilise des workers adaptatifs (CI : jusqu'à 2, local : 1 par défaut).
  - S'exécute en mode silencieux par défaut afin de réduire la surcharge des E/S console.
- Remplacements utiles :
  - `OPENCLAW_E2E_WORKERS=<n>` pour forcer le nombre de workers (plafonné à 16).
  - `OPENCLAW_E2E_VERBOSE=1` pour réactiver une sortie console verbeuse.
- Portée :
  - Comportement end-to-end multi-instance de la passerelle
  - Surfaces WebSocket/HTTP, appairage de nœuds et réseau plus lourd
- Attentes :
  - S'exécute dans la CI (lorsqu'activé dans le pipeline)
  - Aucune vraie clé requise
  - Plus de pièces mobiles que les tests unitaires (peut être plus lent)

### E2E : smoke du backend OpenShell

- Commande : `pnpm test:e2e:openshell`
- Fichier : `test/openshell-sandbox.e2e.test.ts`
- Portée :
  - Démarre sur l'hôte une passerelle OpenShell isolée via Docker
  - Crée un sandbox à partir d'un Dockerfile local temporaire
  - Exerce le backend OpenShell d'OpenClaw via un vrai `sandbox ssh-config` + exécution SSH
  - Vérifie le comportement canonique du système de fichiers distant via le pont fs du sandbox
- Attentes :
  - Opt-in uniquement ; ne fait pas partie de l'exécution par défaut de `pnpm test:e2e`
  - Nécessite un CLI `openshell` local ainsi qu'un démon Docker fonctionnel
  - Utilise un `HOME` / `XDG_CONFIG_HOME` isolé, puis détruit la passerelle de test et le sandbox
- Remplacements utiles :
  - `OPENCLAW_E2E_OPENSHELL=1` pour activer le test lors d'une exécution manuelle de la suite e2e plus large
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` pour pointer vers un binaire CLI non par défaut ou un script wrapper

### Live (vrais fournisseurs + vrais modèles)

- Commande : `pnpm test:live`
- Configuration : `vitest.live.config.ts`
- Fichiers : `src/**/*.live.test.ts`
- Par défaut : **activé** par `pnpm test:live` (définit `OPENCLAW_LIVE_TEST=1`)
- Portée :
  - « Est-ce que ce fournisseur/modèle fonctionne réellement _aujourd'hui_ avec de vrais identifiants ? »
  - Détecter les changements de format des fournisseurs, les particularités d'appel d'outils, les problèmes d'authentification et le comportement de limitation de débit
- Attentes :
  - Pas stable en CI par conception (vrais réseaux, vraies politiques fournisseurs, quotas, pannes)
  - Coûte de l'argent / consomme des limites de débit
  - Préférez exécuter des sous-ensembles restreints plutôt que « tout »
- Les exécutions live sourcent `~/.profile` pour récupérer les clés d'API manquantes.
- Par défaut, les exécutions live isolent toujours `HOME` et copient le matériel de config/authentification dans un répertoire personnel de test temporaire afin que les fixtures unitaires ne puissent pas modifier votre vrai `~/.openclaw`.
- Définissez `OPENCLAW_LIVE_USE_REAL_HOME=1` uniquement lorsque vous avez intentionnellement besoin que les tests live utilisent votre vrai répertoire personnel.
- `pnpm test:live` utilise désormais par défaut un mode plus silencieux : il conserve la sortie de progression `[live] ...`, mais masque l'avis supplémentaire sur `~/.profile` et coupe les journaux de bootstrap de la passerelle / le bruit Bonjour. Définissez `OPENCLAW_LIVE_TEST_QUIET=0` si vous souhaitez retrouver les journaux complets de démarrage.
- Rotation des clés API (spécifique au fournisseur) : définissez `*_API_KEYS` au format virgule/point-virgule ou `*_API_KEY_1`, `*_API_KEY_2` (par exemple `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) ou un remplacement par live via `OPENCLAW_LIVE_*_KEY` ; les tests réessaient en cas de réponses de limitation de débit.
- Sortie de progression/Heartbeat :
  - Les suites live émettent désormais des lignes de progression vers stderr afin que les appels longs aux fournisseurs paraissent actifs même lorsque la capture console de Vitest est silencieuse.
  - `vitest.live.config.ts` désactive l'interception console de Vitest afin que les lignes de progression du fournisseur/de la passerelle soient diffusées immédiatement pendant les exécutions live.
  - Réglez les Heartbeat des modèles directs avec `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Réglez les Heartbeat de la passerelle/des sondes avec `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Quelle suite dois-je exécuter ?

Utilisez ce tableau de décision :

- Modification de logique/tests : exécutez `pnpm test` (et `pnpm test:coverage` si vous avez beaucoup changé)
- Modification du réseau de la passerelle / du protocole WS / de l'appairage : ajoutez `pnpm test:e2e`
- Débogage de « mon bot est en panne » / échecs spécifiques au fournisseur / appel d'outils : exécutez un `pnpm test:live` restreint

## Live : balayage des capacités du nœud Android

- Test : `src/gateway/android-node.capabilities.live.test.ts`
- Script : `pnpm android:test:integration`
- Objectif : invoquer **chaque commande actuellement annoncée** par un nœud Android connecté et vérifier le comportement contractuel des commandes.
- Portée :
  - Préconfiguration/configuration manuelle (la suite n'installe pas, n'exécute pas et n'appaire pas l'application).
  - Validation `node.invoke` de la passerelle commande par commande pour le nœud Android sélectionné.
- Préconfiguration requise :
  - Application Android déjà connectée et appairée à la passerelle.
  - Application maintenue au premier plan.
  - Autorisations/consentement de capture accordés pour les capacités que vous attendez comme réussies.
- Remplacements de cible facultatifs :
  - `OPENCLAW_ANDROID_NODE_ID` ou `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Détails complets de configuration Android : [Android App](/fr/platforms/android)

## Live : smoke des modèles (clés de profil)

Les tests live sont divisés en deux couches afin d'isoler les échecs :

- « Modèle direct » nous indique si le fournisseur/modèle peut répondre tout court avec la clé donnée.
- « Smoke de la passerelle » nous indique si le pipeline complet passerelle+agent fonctionne pour ce modèle (sessions, historique, outils, politique de sandbox, etc.).

### Couche 1 : complétion directe du modèle (sans passerelle)

- Test : `src/agents/models.profiles.live.test.ts`
- Objectif :
  - Énumérer les modèles découverts
  - Utiliser `getApiKeyForModel` pour sélectionner les modèles pour lesquels vous avez des identifiants
  - Exécuter une petite complétion par modèle (et des régressions ciblées si nécessaire)
- Comment activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
- Définissez `OPENCLAW_LIVE_MODELS=modern` (ou `all`, alias de modern) pour réellement exécuter cette suite ; sinon elle est ignorée afin de garder `pnpm test:live` concentré sur le smoke de la passerelle
- Comment sélectionner les modèles :
  - `OPENCLAW_LIVE_MODELS=modern` pour exécuter la liste d'autorisation moderne (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` est un alias pour la liste d'autorisation moderne
  - ou `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (liste d'autorisation séparée par des virgules)
  - Les balayages modern/all utilisent par défaut un plafond organisé à fort signal ; définissez `OPENCLAW_LIVE_MAX_MODELS=0` pour un balayage moderne exhaustif ou une valeur positive pour un plafond plus petit.
- Comment sélectionner les fournisseurs :
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (liste d'autorisation séparée par des virgules)
- D'où viennent les clés :
  - Par défaut : magasin de profils et solutions de secours via l'environnement
  - Définissez `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour imposer **uniquement** le magasin de profils
- Pourquoi cela existe :
  - Sépare « l'API du fournisseur est cassée / la clé est invalide » de « le pipeline d'agent de la passerelle est cassé »
  - Contient de petites régressions isolées (exemple : replay du raisonnement OpenAI Responses/Codex Responses + flux d'appel d'outils)

### Couche 2 : smoke de la passerelle + agent dev (ce que fait réellement "@openclaw")

- Test : `src/gateway/gateway-models.profiles.live.test.ts`
- Objectif :
  - Démarrer une passerelle en processus
  - Créer/modifier une session `agent:dev:*` (remplacement de modèle à chaque exécution)
  - Itérer sur les modèles avec clés et vérifier :
    - une réponse « significative » (sans outils)
    - qu'une vraie invocation d'outil fonctionne (sonde de lecture)
    - des sondes d'outils supplémentaires facultatives (sonde exec+read)
    - que les chemins de régression OpenAI (tool-call-only → suivi) continuent de fonctionner
- Détails des sondes (afin que vous puissiez expliquer rapidement les échecs) :
  - sonde `read` : le test écrit un fichier nonce dans l'espace de travail et demande à l'agent de le `read` puis de renvoyer le nonce.
  - sonde `exec+read` : le test demande à l'agent d'écrire un nonce via `exec` dans un fichier temporaire, puis de le relire avec `read`.
  - sonde d'image : le test joint un PNG généré (chat + code aléatoire) et attend du modèle qu'il renvoie `cat <CODE>`.
  - Référence d'implémentation : `src/gateway/gateway-models.profiles.live.test.ts` et `src/gateway/live-image-probe.ts`.
- Comment activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
- Comment sélectionner les modèles :
  - Par défaut : liste d'autorisation moderne (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` est un alias pour la liste d'autorisation moderne
  - Ou définissez `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (ou une liste séparée par des virgules) pour restreindre
  - Les balayages gateway modern/all utilisent par défaut un plafond organisé à fort signal ; définissez `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` pour un balayage moderne exhaustif ou une valeur positive pour un plafond plus petit.
- Comment sélectionner les fournisseurs (éviter « tout OpenRouter ») :
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (liste d'autorisation séparée par des virgules)
- Les sondes d'outils + d'image sont toujours activées dans ce test live :
  - sonde `read` + sonde `exec+read` (stress sur les outils)
  - la sonde d'image s'exécute lorsque le modèle annonce la prise en charge de l'entrée image
  - Flux (vue d'ensemble) :
    - Le test génère un petit PNG avec « CAT » + un code aléatoire (`src/gateway/live-image-probe.ts`)
    - L'envoie via `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - La passerelle analyse les pièces jointes dans `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - L'agent embarqué transmet au modèle un message utilisateur multimodal
    - Vérification : la réponse contient `cat` + le code (tolérance OCR : de petites erreurs sont autorisées)

Conseil : pour voir ce que vous pouvez tester sur votre machine (et les ids exacts `provider/model`), exécutez :

```bash
openclaw models list
openclaw models list --json
```

## Live : smoke du backend CLI (Claude, Codex, Gemini ou autres CLI locaux)

- Test : `src/gateway/gateway-cli-backend.live.test.ts`
- Objectif : valider le pipeline Gateway + agent en utilisant un backend CLI local, sans toucher à votre configuration par défaut.
- Les valeurs par défaut du smoke spécifiques au backend se trouvent dans la définition `cli-backend.ts` de l'extension propriétaire.
- Activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Valeurs par défaut :
  - Fournisseur/modèle par défaut : `claude-cli/claude-sonnet-4-6`
  - Le comportement commande/args/image provient des métadonnées du Plugin propriétaire du backend CLI.
- Remplacements facultatifs :
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` pour envoyer une vraie pièce jointe image (les chemins sont injectés dans l'invite).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` pour transmettre les chemins de fichiers image comme arguments CLI au lieu d'une injection dans l'invite.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (ou `"list"`) pour contrôler la manière dont les arguments d'image sont transmis lorsque `IMAGE_ARG` est défini.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` pour envoyer un second tour et valider le flux de reprise.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` pour désactiver la sonde de continuité de session par défaut Claude Sonnet -> Opus sur une même session (définissez-la à `1` pour la forcer lorsque le modèle sélectionné prend en charge une cible de bascule).

Exemple :

```bash
OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

Recette Docker :

```bash
pnpm test:docker:live-cli-backend
```

Recettes Docker à fournisseur unique :

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Notes :

- L'exécuteur Docker se trouve dans `scripts/test-live-cli-backend-docker.sh`.
- Il exécute le smoke live du backend CLI à l'intérieur de l'image Docker du dépôt en tant qu'utilisateur non root `node`.
- Il résout les métadonnées du smoke CLI à partir de l'extension propriétaire, puis installe le paquet CLI Linux correspondant (`@anthropic-ai/claude-code`, `@openai/codex` ou `@google/gemini-cli`) dans un préfixe inscriptible mis en cache à `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (par défaut : `~/.cache/openclaw/docker-cli-tools`).
- `pnpm test:docker:live-cli-backend:claude-subscription` nécessite une authentification OAuth d'abonnement portable Claude Code via soit `~/.claude/.credentials.json` avec `claudeAiOauth.subscriptionType`, soit `CLAUDE_CODE_OAUTH_TOKEN` depuis `claude setup-token`. Il prouve d'abord un `claude -p` direct dans Docker, puis exécute deux tours Gateway CLI-backend sans conserver les variables d'environnement de clé API Anthropic. Cette voie d'abonnement désactive par défaut les sondes Claude MCP/outil et image, car Claude achemine actuellement l'utilisation d'applications tierces via une facturation d'usage supplémentaire au lieu des limites normales du plan d'abonnement.
- Le smoke live du backend CLI exerce désormais le même flux end-to-end pour Claude, Codex et Gemini : tour texte, tour de classification d'image, puis appel d'outil MCP `cron` vérifié via le CLI de la passerelle.
- Le smoke par défaut de Claude modifie aussi la session de Sonnet vers Opus et vérifie que la session reprise se souvient toujours d'une note précédente.

## Live : smoke ACP bind (`/acp spawn ... --bind here`)

- Test : `src/gateway/gateway-acp-bind.live.test.ts`
- Objectif : valider le vrai flux de liaison de conversation ACP avec un agent ACP live :
  - envoyer `/acp spawn <agent> --bind here`
  - lier en place une conversation synthétique de canal de messages
  - envoyer un suivi normal sur cette même conversation
  - vérifier que le suivi arrive dans la transcription de session ACP liée
- Activer :
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Valeurs par défaut :
  - Agents ACP dans Docker : `claude,codex,gemini`
  - Agent ACP pour `pnpm test:live ...` direct : `claude`
  - Canal synthétique : contexte de conversation de type message privé Slack
  - Backend ACP : `acpx`
- Remplacements :
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Notes :
  - Cette voie utilise la surface `chat.send` de la passerelle avec des champs synthétiques de route d'origine réservés à l'administration afin que les tests puissent attacher un contexte de canal de messages sans prétendre livrer un message à l'extérieur.
  - Lorsque `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` n'est pas défini, le test utilise le registre d'agents intégré du Plugin `acpx` embarqué pour l'agent de harnais ACP sélectionné.

Exemple :

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Recette Docker :

```bash
pnpm test:docker:live-acp-bind
```

Recettes Docker à agent unique :

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Notes Docker :

- L'exécuteur Docker se trouve dans `scripts/test-live-acp-bind-docker.sh`.
- Par défaut, il exécute le smoke ACP bind sur tous les agents CLI live pris en charge en séquence : `claude`, `codex`, puis `gemini`.
- Utilisez `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` ou `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` pour restreindre la matrice.
- Il source `~/.profile`, prépare dans le conteneur le matériel d'authentification CLI correspondant, installe `acpx` dans un préfixe npm inscriptible, puis installe le CLI live demandé (`@anthropic-ai/claude-code`, `@openai/codex` ou `@google/gemini-cli`) s'il est absent.
- Dans Docker, l'exécuteur définit `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` afin qu'acpx conserve disponibles pour le CLI de harnais enfant les variables d'environnement fournisseur issues du profil sourcé.

## Live : smoke du harnais Codex app-server

- Objectif : valider le harnais Codex appartenant au Plugin via la méthode
  `agent` normale de la passerelle :
  - charger le Plugin `codex` intégré
  - sélectionner `OPENCLAW_AGENT_RUNTIME=codex`
  - envoyer un premier tour agent de la passerelle à `codex/gpt-5.4`
  - envoyer un second tour à la même session OpenClaw et vérifier que le thread app-server
    peut reprendre
  - exécuter `/codex status` et `/codex models` via le même chemin de commande
    de la passerelle
- Test : `src/gateway/gateway-codex-harness.live.test.ts`
- Activer : `OPENCLAW_LIVE_CODEX_HARNESS=1`
- Modèle par défaut : `codex/gpt-5.4`
- Sonde d'image facultative : `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- Sonde MCP/outil facultative : `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- Le smoke définit `OPENCLAW_AGENT_HARNESS_FALLBACK=none` afin qu'un harnais Codex cassé
  ne puisse pas réussir en retombant silencieusement sur PI.
- Authentification : `OPENAI_API_KEY` depuis le shell/profil, plus copie facultative de
  `~/.codex/auth.json` et `~/.codex/config.toml`

Recette locale :

```bash
source ~/.profile
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=codex/gpt-5.4 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

Recette Docker :

```bash
source ~/.profile
pnpm test:docker:live-codex-harness
```

Notes Docker :

- L'exécuteur Docker se trouve dans `scripts/test-live-codex-harness-docker.sh`.
- Il source le `~/.profile` monté, transmet `OPENAI_API_KEY`, copie les fichiers
  d'authentification Codex CLI lorsqu'ils sont présents, installe `@openai/codex` dans un préfixe npm
  monté et inscriptible, prépare l'arborescence source, puis exécute uniquement le test live du harnais Codex.
- Docker active par défaut les sondes d'image et MCP/outil. Définissez
  `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` ou
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` lorsque vous avez besoin d'une exécution de débogage plus restreinte.
- Docker exporte aussi `OPENCLAW_AGENT_HARNESS_FALLBACK=none`, conformément à la configuration du
  test live afin qu'un fallback `openai-codex/*` ou PI ne puisse pas masquer une régression du harnais Codex.

### Recettes live recommandées

Des listes d'autorisation étroites et explicites sont les plus rapides et les moins sujettes aux échecs aléatoires :

- Modèle unique, direct (sans passerelle) :
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Modèle unique, smoke de la passerelle :
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Appel d'outils sur plusieurs fournisseurs :
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Focus Google (clé API Gemini + Antigravity) :
  - Gemini (clé API) : `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth) : `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Notes :

- `google/...` utilise l'API Gemini (clé API).
- `google-antigravity/...` utilise le pont OAuth Antigravity (point de terminaison d'agent de type Cloud Code Assist).
- `google-gemini-cli/...` utilise le CLI Gemini local sur votre machine (authentification distincte + particularités d'outillage).
- API Gemini vs CLI Gemini :
  - API : OpenClaw appelle l'API Gemini hébergée par Google via HTTP (authentification par clé API / profil) ; c'est ce que la plupart des utilisateurs entendent par « Gemini ».
  - CLI : OpenClaw exécute un binaire local `gemini` ; il a sa propre authentification et peut se comporter différemment (streaming/prise en charge des outils/décalage de version).

## Live : matrice de modèles (ce que nous couvrons)

Il n'existe pas de « liste de modèles CI » fixe (le live est opt-in), mais voici les modèles **recommandés** à couvrir régulièrement sur une machine de développement avec des clés.

### Ensemble de smoke moderne (appel d'outils + image)

Il s'agit de l'exécution des « modèles courants » que nous nous attendons à maintenir en état de marche :

- OpenAI (hors Codex) : `openai/gpt-5.4` (facultatif : `openai/gpt-5.4-mini`)
- OpenAI Codex : `openai-codex/gpt-5.4`
- Anthropic : `anthropic/claude-opus-4-6` (ou `anthropic/claude-sonnet-4-6`)
- Google (API Gemini) : `google/gemini-3.1-pro-preview` et `google/gemini-3-flash-preview` (évitez les anciens modèles Gemini 2.x)
- Google (Antigravity) : `google-antigravity/claude-opus-4-6-thinking` et `google-antigravity/gemini-3-flash`
- Z.AI (GLM) : `zai/glm-4.7`
- MiniMax : `minimax/MiniMax-M2.7`

Exécuter le smoke de la passerelle avec outils + image :
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Référence de base : appel d'outils (Read + Exec facultatif)

Choisissez au moins un modèle par famille de fournisseurs :

- OpenAI : `openai/gpt-5.4` (ou `openai/gpt-5.4-mini`)
- Anthropic : `anthropic/claude-opus-4-6` (ou `anthropic/claude-sonnet-4-6`)
- Google : `google/gemini-3-flash-preview` (ou `google/gemini-3.1-pro-preview`)
- Z.AI (GLM) : `zai/glm-4.7`
- MiniMax : `minimax/MiniMax-M2.7`

Couverture supplémentaire facultative (agréable à avoir) :

- xAI : `xai/grok-4` (ou la dernière version disponible)
- Mistral : `mistral/`… (choisissez un modèle compatible « tools » que vous avez activé)
- Cerebras : `cerebras/`… (si vous y avez accès)
- LM Studio : `lmstudio/`… (local ; l'appel d'outils dépend du mode API)

### Vision : envoi d'image (pièce jointe → message multimodal)

Incluez au moins un modèle compatible image dans `OPENCLAW_LIVE_GATEWAY_MODELS` (variants Claude/Gemini/OpenAI compatibles vision, etc.) afin d'exercer la sonde d'image.

### Agrégateurs / passerelles alternatives

Si vous avez les clés activées, nous prenons aussi en charge les tests via :

- OpenRouter : `openrouter/...` (des centaines de modèles ; utilisez `openclaw models scan` pour trouver des candidats compatibles outils+image)
- OpenCode : `opencode/...` pour Zen et `opencode-go/...` pour Go (authentification via `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Davantage de fournisseurs peuvent être inclus dans la matrice live (si vous avez les identifiants/la configuration) :

- Intégrés : `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Via `models.providers` (points de terminaison personnalisés) : `minimax` (cloud/API), ainsi que tout proxy compatible OpenAI/Anthropic (LM Studio, vLLM, LiteLLM, etc.)

Conseil : n'essayez pas de coder en dur « tous les modèles » dans la documentation. La liste faisant autorité est tout ce que `discoverModels(...)` renvoie sur votre machine + toutes les clés disponibles.

## Identifiants (ne jamais commit)

Les tests live découvrent les identifiants de la même manière que le CLI. Conséquences pratiques :

- Si le CLI fonctionne, les tests live devraient trouver les mêmes clés.
- Si un test live dit « aucun identifiant », déboguez-le comme vous débogueriez `openclaw models list` / la sélection de modèle.

- Profils d'authentification par agent : `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (c'est ce que signifie « clés de profil » dans les tests live)
- Configuration : `~/.openclaw/openclaw.json` (ou `OPENCLAW_CONFIG_PATH`)
- Répertoire d'état hérité : `~/.openclaw/credentials/` (copié dans le répertoire personnel live préparé lorsqu'il est présent, mais pas dans le magasin principal de clés de profil)
- Les exécutions live locales copient par défaut la configuration active, les fichiers `auth-profiles.json` par agent, le répertoire hérité `credentials/` et les répertoires d'authentification CLI externes pris en charge dans un répertoire personnel de test temporaire ; les répertoires personnels live préparés ignorent `workspace/` et `sandboxes/`, et les remplacements de chemin `agents.*.workspace` / `agentDir` sont supprimés afin que les sondes restent hors de votre véritable espace de travail hôte.

Si vous souhaitez vous appuyer sur des clés d'environnement (par exemple exportées dans votre `~/.profile`), exécutez les tests locaux après `source ~/.profile`, ou utilisez les exécuteurs Docker ci-dessous (ils peuvent monter `~/.profile` dans le conteneur).

## Live Deepgram (transcription audio)

- Test : `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Activer : `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## Live BytePlus coding plan

- Test : `src/agents/byteplus.live.test.ts`
- Activer : `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Remplacement de modèle facultatif : `BYTEPLUS_CODING_MODEL=ark-code-latest`

## Live ComfyUI workflow media

- Test : `extensions/comfy/comfy.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Portée :
  - Exerce les chemins image, vidéo et `music_generate` du comfy intégré
  - Ignore chaque capacité sauf si `models.providers.comfy.<capability>` est configuré
  - Utile après modification de la soumission de workflow comfy, du polling, des téléchargements ou de l'enregistrement du Plugin

## Live Image generation

- Test : `src/image-generation/runtime.live.test.ts`
- Commande : `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harnais : `pnpm test:live:media image`
- Portée :
  - Énumère chaque Plugin fournisseur de génération d'image enregistré
  - Charge les variables d'environnement fournisseur manquantes depuis votre shell de connexion (`~/.profile`) avant le sondage
  - Utilise par défaut les clés API live/d'environnement avant les profils d'authentification stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les véritables identifiants du shell
  - Ignore les fournisseurs sans authentification/profil/modèle exploitable
  - Exécute les variantes standard de génération d'image via la capacité runtime partagée :
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Fournisseurs intégrés actuellement couverts :
  - `openai`
  - `google`
- Restriction facultative :
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Comportement d'authentification facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l'authentification via le magasin de profils et ignorer les remplacements uniquement présents dans l'environnement

## Live Music generation

- Test : `extensions/music-generation-providers.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harnais : `pnpm test:live:media music`
- Portée :
  - Exerce le chemin partagé de fournisseur intégré de génération musicale
  - Couvre actuellement Google et MiniMax
  - Charge les variables d'environnement fournisseur depuis votre shell de connexion (`~/.profile`) avant le sondage
  - Utilise par défaut les clés API live/d'environnement avant les profils d'authentification stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les véritables identifiants du shell
  - Ignore les fournisseurs sans authentification/profil/modèle exploitable
  - Exécute les deux modes runtime déclarés lorsqu'ils sont disponibles :
    - `generate` avec entrée basée uniquement sur l'invite
    - `edit` lorsque le fournisseur déclare `capabilities.edit.enabled`
  - Couverture actuelle de la voie partagée :
    - `google` : `generate`, `edit`
    - `minimax` : `generate`
    - `comfy` : fichier live Comfy séparé, pas ce balayage partagé
- Restriction facultative :
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Comportement d'authentification facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l'authentification via le magasin de profils et ignorer les remplacements uniquement présents dans l'environnement

## Live Video generation

- Test : `extensions/video-generation-providers.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harnais : `pnpm test:live:media video`
- Portée :
  - Exerce le chemin partagé de fournisseur intégré de génération vidéo
  - Charge les variables d'environnement fournisseur depuis votre shell de connexion (`~/.profile`) avant le sondage
  - Utilise par défaut les clés API live/d'environnement avant les profils d'authentification stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les véritables identifiants du shell
  - Ignore les fournisseurs sans authentification/profil/modèle exploitable
  - Exécute les deux modes runtime déclarés lorsqu'ils sont disponibles :
    - `generate` avec entrée basée uniquement sur l'invite
    - `imageToVideo` lorsque le fournisseur déclare `capabilities.imageToVideo.enabled` et que le fournisseur/modèle sélectionné accepte une entrée image locale basée sur buffer dans le balayage partagé
    - `videoToVideo` lorsque le fournisseur déclare `capabilities.videoToVideo.enabled` et que le fournisseur/modèle sélectionné accepte une entrée vidéo locale basée sur buffer dans le balayage partagé
  - Fournisseurs `imageToVideo` actuellement déclarés mais ignorés dans le balayage partagé :
    - `vydra` parce que le `veo3` intégré est text-only et que le `kling` intégré exige une URL d'image distante
  - Couverture Vydra spécifique au fournisseur :
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - ce fichier exécute `veo3` en texte-vers-vidéo plus une voie `kling` qui utilise par défaut un fixture d'URL d'image distante
  - Couverture live `videoToVideo` actuelle :
    - `runway` uniquement lorsque le modèle sélectionné est `runway/gen4_aleph`
  - Fournisseurs `videoToVideo` actuellement déclarés mais ignorés dans le balayage partagé :
    - `alibaba`, `qwen`, `xai` parce que ces chemins exigent actuellement des URL de référence distantes `http(s)` / MP4
    - `google` parce que la voie Gemini/Veo partagée actuelle utilise une entrée locale basée sur buffer et que ce chemin n'est pas accepté dans le balayage partagé
    - `openai` parce que la voie partagée actuelle ne garantit pas l'accès spécifique à l'organisation pour l'inpainting/remix vidéo
- Restriction facultative :
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- Comportement d'authentification facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l'authentification via le magasin de profils et ignorer les remplacements uniquement présents dans l'environnement

## Harnais live média

- Commande : `pnpm test:live:media`
- Objectif :
  - Exécute les suites live partagées image, musique et vidéo via un seul point d'entrée natif du dépôt
  - Charge automatiquement les variables d'environnement fournisseur manquantes depuis `~/.profile`
  - Restreint automatiquement chaque suite par défaut aux fournisseurs qui disposent actuellement d'une authentification exploitable
  - Réutilise `scripts/test-live.mjs`, de sorte que le comportement Heartbeat et mode silencieux reste cohérent
- Exemples :
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Exécuteurs Docker (vérifications facultatives « fonctionne sous Linux »)

Ces exécuteurs Docker se divisent en deux catégories :

- Exécuteurs live-model : `test:docker:live-models` et `test:docker:live-gateway` n'exécutent que leur fichier live de clés de profil correspondant à l'intérieur de l'image Docker du dépôt (`src/agents/models.profiles.live.test.ts` et `src/gateway/gateway-models.profiles.live.test.ts`), en montant votre répertoire de configuration local et votre espace de travail (et en sourçant `~/.profile` s'il est monté). Les points d'entrée locaux correspondants sont `test:live:models-profiles` et `test:live:gateway-profiles`.
- Les exécuteurs Docker live utilisent par défaut un plafond de smoke plus réduit afin qu'un balayage Docker complet reste praticable :
  `test:docker:live-models` utilise par défaut `OPENCLAW_LIVE_MAX_MODELS=12`, et
  `test:docker:live-gateway` utilise par défaut `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`, et
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Remplacez ces variables d'environnement lorsque vous
  souhaitez explicitement un balayage exhaustif plus large.
- `test:docker:all` construit une fois l'image Docker live via `test:docker:live-build`, puis la réutilise pour les deux voies Docker live.
- Les exécuteurs smoke de conteneur : `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` et `test:docker:plugins` démarrent un ou plusieurs vrais conteneurs et vérifient des chemins d'intégration de plus haut niveau.

Les exécuteurs Docker live-model montent aussi en bind uniquement les répertoires d'authentification CLI nécessaires (ou tous ceux pris en charge lorsque l'exécution n'est pas restreinte), puis les copient dans le répertoire personnel du conteneur avant l'exécution afin que l'OAuth du CLI externe puisse actualiser les jetons sans modifier le magasin d'authentification de l'hôte :

- Modèles directs : `pnpm test:docker:live-models` (script : `scripts/test-live-models-docker.sh`)
- Smoke ACP bind : `pnpm test:docker:live-acp-bind` (script : `scripts/test-live-acp-bind-docker.sh`)
- Smoke du backend CLI : `pnpm test:docker:live-cli-backend` (script : `scripts/test-live-cli-backend-docker.sh`)
- Smoke du harnais Codex app-server : `pnpm test:docker:live-codex-harness` (script : `scripts/test-live-codex-harness-docker.sh`)
- Passerelle + agent dev : `pnpm test:docker:live-gateway` (script : `scripts/test-live-gateway-models-docker.sh`)
- Smoke live Open WebUI : `pnpm test:docker:openwebui` (script : `scripts/e2e/openwebui-docker.sh`)
- Assistant d'onboarding (TTY, échafaudage complet) : `pnpm test:docker:onboard` (script : `scripts/e2e/onboard-docker.sh`)
- Réseau de passerelle (deux conteneurs, auth WS + santé) : `pnpm test:docker:gateway-network` (script : `scripts/e2e/gateway-network-docker.sh`)
- Pont de canal MCP (Gateway pré-initialisée + pont stdio + smoke brut de trame de notification Claude) : `pnpm test:docker:mcp-channels` (script : `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (smoke d'installation + alias `/plugin` + sémantique de redémarrage du bundle Claude) : `pnpm test:docker:plugins` (script : `scripts/e2e/plugins-docker.sh`)

Les exécuteurs Docker live-model montent également en bind la révision courante en lecture seule et
la préparent dans un répertoire de travail temporaire à l'intérieur du conteneur. Cela garde l'image runtime
légère tout en exécutant Vitest sur votre source/configuration locale exacte.
L'étape de préparation ignore les gros caches uniquement locaux et les sorties de build d'application telles que
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__`, ainsi que les répertoires de sortie `.build` locaux à l'application ou Gradle,
afin que les exécutions live Docker ne passent pas des minutes à copier
des artefacts spécifiques à la machine.
Ils définissent aussi `OPENCLAW_SKIP_CHANNELS=1` afin que les sondes live de la passerelle ne démarrent pas
de vrais workers de canal Telegram/Discord/etc. à l'intérieur du conteneur.
`test:docker:live-models` exécute toujours `pnpm test:live`, donc transmettez aussi
`OPENCLAW_LIVE_GATEWAY_*` lorsque vous devez restreindre ou exclure la couverture live de la passerelle
dans cette voie Docker.
`test:docker:openwebui` est un smoke de compatibilité de plus haut niveau : il démarre un
conteneur de passerelle OpenClaw avec les points de terminaison HTTP compatibles OpenAI activés,
démarre un conteneur Open WebUI épinglé face à cette passerelle, se connecte via
Open WebUI, vérifie que `/api/models` expose `openclaw/default`, puis envoie une
vraie requête de chat via le proxy `/api/chat/completions` d'Open WebUI.
La première exécution peut être sensiblement plus lente car Docker peut devoir extraire l'image
Open WebUI et Open WebUI peut avoir besoin de terminer sa propre configuration de démarrage à froid.
Cette voie attend une clé de modèle live exploitable, et `OPENCLAW_PROFILE_FILE`
(`~/.profile` par défaut) est le principal moyen de la fournir dans des exécutions Dockerisées.
Les exécutions réussies affichent une petite charge utile JSON comme `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` est volontairement déterministe et n'a pas besoin d'un
vrai compte Telegram, Discord ou iMessage. Il démarre un conteneur Gateway
pré-initialisé, démarre un second conteneur qui lance `openclaw mcp serve`, puis
vérifie la découverte de conversations routées, les lectures de transcription, les métadonnées de pièces jointes,
le comportement de la file d'événements live, le routage d'envoi sortant, et les notifications de canal +
d'autorisations de type Claude sur le vrai pont stdio MCP. La vérification des notifications
inspecte directement les trames MCP stdio brutes afin que le smoke valide ce que le
pont émet réellement, et pas seulement ce qu'un SDK client particulier expose par hasard.

Smoke manuel ACP en langage naturel sur thread (pas CI) :

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Conservez ce script pour les workflows de régression/débogage. Il pourra être à nouveau nécessaire pour la validation du routage des threads ACP, donc ne le supprimez pas.

Variables d'environnement utiles :

- `OPENCLAW_CONFIG_DIR=...` (par défaut : `~/.openclaw`) monté sur `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (par défaut : `~/.openclaw/workspace`) monté sur `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (par défaut : `~/.profile`) monté sur `/home/node/.profile` et sourcé avant l'exécution des tests
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (par défaut : `~/.cache/openclaw/docker-cli-tools`) monté sur `/home/node/.npm-global` pour les installations CLI mises en cache dans Docker
- Les répertoires/fichiers d'authentification CLI externes sous `$HOME` sont montés en lecture seule sous `/host-auth...`, puis copiés dans `/home/node/...` avant le démarrage des tests
  - Répertoires par défaut : `.minimax`
  - Fichiers par défaut : `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Les exécutions restreintes par fournisseur ne montent que les répertoires/fichiers nécessaires déduits de `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Remplacement manuel avec `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none`, ou une liste séparée par des virgules comme `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` pour restreindre l'exécution
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` pour filtrer les fournisseurs dans le conteneur
- `OPENCLAW_SKIP_DOCKER_BUILD=1` pour réutiliser une image `openclaw:local-live` existante lors de relances qui ne nécessitent pas de reconstruction
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour garantir que les identifiants proviennent du magasin de profils (et non de l'environnement)
- `OPENCLAW_OPENWEBUI_MODEL=...` pour choisir le modèle exposé par la passerelle pour le smoke Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` pour remplacer l'invite de vérification de nonce utilisée par le smoke Open WebUI
- `OPENWEBUI_IMAGE=...` pour remplacer le tag d'image Open WebUI épinglé

## Vérification de la documentation

Exécutez les vérifications de documentation après des modifications de doc : `pnpm check:docs`.
Exécutez la validation complète des ancres Mintlify lorsque vous avez aussi besoin de vérifier les titres dans la page : `pnpm docs:check-links:anchors`.

## Régression hors ligne (compatible CI)

Il s'agit de régressions de « vrai pipeline » sans vrais fournisseurs :

- Appel d'outil de la passerelle (OpenAI simulé, vraie boucle passerelle + agent) : `src/gateway/gateway.test.ts` (cas : "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Assistant de la passerelle (WS `wizard.start`/`wizard.next`, écrit config + auth imposés) : `src/gateway/gateway.test.ts` (cas : "runs wizard over ws and writes auth token config")

## Évaluations de fiabilité des agents (Skills)

Nous avons déjà quelques tests compatibles CI qui se comportent comme des « évaluations de fiabilité des agents » :

- Appel d'outil simulé via la vraie boucle passerelle + agent (`src/gateway/gateway.test.ts`).
- Flux d'assistant end-to-end qui valident le câblage de session et les effets de configuration (`src/gateway/gateway.test.ts`).

Ce qui manque encore pour les Skills (voir [Skills](/fr/tools/skills)) :

- **Prise de décision :** lorsque les Skills sont listées dans l'invite, l'agent choisit-il la bonne Skill (ou évite-t-il celles qui ne sont pas pertinentes) ?
- **Conformité :** l'agent lit-il `SKILL.md` avant utilisation et suit-il les étapes/arguments requis ?
- **Contrats de workflow :** scénarios multi-tours qui vérifient l'ordre des outils, la conservation de l'historique de session et les frontières du sandbox.

Les futures évaluations doivent d'abord rester déterministes :

- Un exécuteur de scénarios utilisant des fournisseurs simulés pour vérifier les appels d'outils + leur ordre, les lectures de fichiers de Skill et le câblage de session.
- Une petite suite de scénarios centrés sur les Skills (utiliser vs éviter, gate, injection d'invite).
- Des évaluations live facultatives (opt-in, contrôlées par variables d'environnement) seulement après la mise en place de la suite compatible CI.

## Tests de contrat (forme des plugins et canaux)

Les tests de contrat vérifient que chaque plugin et canal enregistré respecte son
contrat d'interface. Ils parcourent tous les plugins découverts et exécutent une suite d'assertions
de forme et de comportement. La voie unitaire par défaut `pnpm test` ignore volontairement
ces fichiers partagés de seam et de smoke ; exécutez explicitement les commandes de contrat
lorsque vous modifiez des surfaces partagées de canal ou de fournisseur.

### Commandes

- Tous les contrats : `pnpm test:contracts`
- Contrats de canal uniquement : `pnpm test:contracts:channels`
- Contrats de fournisseur uniquement : `pnpm test:contracts:plugins`

### Contrats de canal

Situés dans `src/channels/plugins/contracts/*.contract.test.ts` :

- **plugin** - Forme de base du Plugin (id, nom, capacités)
- **setup** - Contrat de l'assistant de configuration
- **session-binding** - Comportement de liaison de session
- **outbound-payload** - Structure de charge utile des messages
- **inbound** - Gestion des messages entrants
- **actions** - Gestionnaires d'actions de canal
- **threading** - Gestion des ids de thread
- **directory** - API de répertoire/roster
- **group-policy** - Application de la politique de groupe

### Contrats d'état des fournisseurs

Situés dans `src/plugins/contracts/*.contract.test.ts`.

- **status** - Sondes d'état des canaux
- **registry** - Forme du registre de Plugins

### Contrats de fournisseur

Situés dans `src/plugins/contracts/*.contract.test.ts` :

- **auth** - Contrat du flux d'authentification
- **auth-choice** - Choix/sélection de l'authentification
- **catalog** - API du catalogue de modèles
- **discovery** - Découverte de Plugin
- **loader** - Chargement de Plugin
- **runtime** - Runtime du fournisseur
- **shape** - Forme/interface du Plugin
- **wizard** - Assistant de configuration

### Quand les exécuter

- Après modification des exports ou sous-chemins de `plugin-sdk`
- Après ajout ou modification d'un Plugin de canal ou de fournisseur
- Après refactorisation de l'enregistrement ou de la découverte des Plugins

Les tests de contrat s'exécutent dans la CI et ne nécessitent pas de vraies clés d'API.

## Ajouter des régressions (recommandations)

Lorsque vous corrigez un problème de fournisseur/modèle découvert en live :

- Ajoutez si possible une régression compatible CI (fournisseur simulé/stub, ou capture de la transformation exacte de forme de requête)
- Si le problème est intrinsèquement live-only (limitations de débit, politiques d'authentification), gardez le test live étroit et opt-in via des variables d'environnement
- Préférez cibler la plus petite couche qui intercepte le bug :
  - bug de conversion/rejeu de requête fournisseur → test de modèles directs
  - bug du pipeline de session/historique/outils de la passerelle → smoke live de la passerelle ou test simulé compatible CI de la passerelle
- Garde-fou de traversée SecretRef :
  - `src/secrets/exec-secret-ref-id-parity.test.ts` dérive une cible échantillonnée par classe SecretRef à partir des métadonnées du registre (`listSecretTargetRegistryEntries()`), puis vérifie que les ids exec de segment de traversée sont rejetés.
  - Si vous ajoutez une nouvelle famille de cibles SecretRef `includeInPlan` dans `src/secrets/target-registry-data.ts`, mettez à jour `classifyTargetClass` dans ce test. Le test échoue volontairement sur les ids de cible non classifiés afin que les nouvelles classes ne puissent pas être ignorées silencieusement.
