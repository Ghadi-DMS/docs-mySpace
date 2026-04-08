---
read_when:
    - Exécution des tests en local ou en CI
    - Ajout de régressions pour des bugs de modèle/fournisseur
    - Débogage du comportement de la gateway et de l’agent
summary: 'Kit de test : suites unitaires/e2e/live, exécuteurs Docker et ce que couvre chaque test'
title: Tests
x-i18n:
    generated_at: "2026-04-08T02:17:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: ace2c19bfc350780475f4348264a4b55be2b4ccbb26f0d33b4a6af34510943b5
    source_path: help/testing.md
    workflow: 15
---

# Tests

OpenClaw dispose de trois suites Vitest (unit/integration, e2e, live) et d’un petit ensemble d’exécuteurs Docker.

Ce document est un guide « comment nous testons » :

- Ce que couvre chaque suite (et ce qu’elle ne couvre délibérément _pas_)
- Quelles commandes exécuter pour les workflows courants (local, avant push, débogage)
- Comment les tests live découvrent les identifiants et sélectionnent les modèles/fournisseurs
- Comment ajouter des régressions pour des problèmes réels de modèle/fournisseur

## Démarrage rapide

La plupart du temps :

- Gate complet (attendu avant push) : `pnpm build && pnpm check && pnpm test`
- Exécution locale plus rapide de toute la suite sur une machine bien dimensionnée : `pnpm test:max`
- Boucle de surveillance Vitest directe : `pnpm test:watch`
- Le ciblage direct de fichier route désormais aussi les chemins d’extension/canal : `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Site QA avec support Docker : `pnpm qa:lab:up`

Quand vous modifiez des tests ou souhaitez plus de confiance :

- Gate de couverture : `pnpm test:coverage`
- Suite E2E : `pnpm test:e2e`

Quand vous déboguez de vrais fournisseurs/modèles (nécessite de vrais identifiants) :

- Suite live (modèles + sondes gateway outil/image) : `pnpm test:live`
- Cibler silencieusement un seul fichier live : `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Astuce : lorsque vous n’avez besoin que d’un seul cas en échec, préférez réduire les tests live via les variables d’environnement de liste d’autorisation décrites ci-dessous.

## Suites de test (ce qui s’exécute où)

Voyez les suites comme un continuum de « réalisme croissant » (et de fragilité/coût croissants) :

### Unit / integration (par défaut)

- Commande : `pnpm test`
- Configuration : dix exécutions séquentielles de fragments (`vitest.full-*.config.ts`) sur les projets Vitest à portée existants
- Fichiers : inventaires core/unit sous `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts`, et les tests node `ui` autorisés couverts par `vitest.unit.config.ts`
- Portée :
  - Tests unitaires purs
  - Tests d’intégration dans le processus (auth gateway, routage, outils, parsing, config)
  - Régressions déterministes pour des bugs connus
- Attentes :
  - S’exécute en CI
  - Aucune vraie clé requise
  - Doit être rapide et stable
- Remarque sur les projets :
  - `pnpm test` sans cible exécute maintenant onze petites configurations fragmentées (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-support-boundary`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) au lieu d’un seul grand processus root-project natif. Cela réduit le RSS maximal sur les machines chargées et évite que le travail auto-reply/extension n’affame des suites sans rapport.
  - `pnpm test --watch` utilise toujours le graphe de projets natif de `vitest.config.ts`, parce qu’une boucle de surveillance multi-fragments n’est pas pratique.
  - `pnpm test`, `pnpm test:watch` et `pnpm test:perf:imports` routent d’abord les cibles explicites de fichier/répertoire vers des voies à portée, de sorte que `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` évite le coût de démarrage complet du root project.
  - `pnpm test:changed` étend les chemins git modifiés vers ces mêmes voies à portée quand le diff ne touche que des fichiers source/test routables ; les modifications de config/setup reviennent toujours à une relance large du root project.
  - Certains tests `plugin-sdk` et `commands` sont aussi routés via des voies légères dédiées qui évitent `test/setup-openclaw-runtime.ts` ; les fichiers lourds en état/runtime restent sur les voies existantes.
  - Certains fichiers source utilitaires `plugin-sdk` et `commands` mappent aussi les exécutions en mode changed vers des tests frères explicites dans ces voies légères, pour que les modifications d’utilitaires évitent de relancer toute la suite lourde du répertoire.
  - `auto-reply` a maintenant trois groupes dédiés : assistants core de premier niveau, tests d’intégration `reply.*` de premier niveau, et le sous-arbre `src/auto-reply/reply/**`. Cela garde le travail du harnais de réponse le plus lourd à l’écart des tests bon marché de statut/chunk/token.
- Remarque sur l’exécuteur embarqué :
  - Lorsque vous modifiez les entrées de découverte des outils de message ou le contexte d’exécution de compaction,
    conservez les deux niveaux de couverture.
  - Ajoutez des régressions ciblées sur les assistants pour les frontières pures de routage/normalisation.
  - Gardez aussi saines les suites d’intégration de l’exécuteur embarqué :
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`, et
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Ces suites vérifient que les ids à portée et le comportement de compaction continuent bien de passer
    par les vrais chemins `run.ts` / `compact.ts` ; des tests utilitaires seuls ne
    remplacent pas suffisamment ces chemins d’intégration.
- Remarque sur le pool :
  - La configuration Vitest de base utilise maintenant `threads` par défaut.
  - La configuration Vitest partagée fixe aussi `isolate: false` et utilise l’exécuteur non isolé sur les root projects, e2e et live.
  - La voie UI racine conserve sa configuration `jsdom` et son optimiseur, mais s’exécute désormais aussi sur l’exécuteur partagé non isolé.
  - Chaque fragment `pnpm test` hérite des mêmes valeurs par défaut `threads` + `isolate: false` depuis la configuration Vitest partagée.
  - Le lanceur partagé `scripts/run-vitest.mjs` ajoute aussi maintenant `--no-maglev` par défaut pour les processus Node enfants de Vitest afin de réduire la surcharge de compilation V8 lors des grosses exécutions locales. Définissez `OPENCLAW_VITEST_ENABLE_MAGLEV=1` si vous devez comparer au comportement V8 standard.
- Remarque sur l’itération locale rapide :
  - `pnpm test:changed` route via des voies à portée quand les chemins modifiés correspondent proprement à une suite plus petite.
  - `pnpm test:max` et `pnpm test:changed:max` conservent le même comportement de routage, simplement avec une limite de workers plus élevée.
  - La mise à l’échelle automatique des workers en local est désormais volontairement prudente et ralentit aussi quand la moyenne de charge de l’hôte est déjà élevée, afin que plusieurs exécutions Vitest concurrentes causent moins de dégâts par défaut.
  - La configuration Vitest de base marque les projets/fichiers de config comme `forceRerunTriggers` afin que les relances en mode changed restent correctes lorsque le câblage des tests change.
  - La configuration laisse `OPENCLAW_VITEST_FS_MODULE_CACHE` activé sur les hôtes pris en charge ; définissez `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` si vous voulez un emplacement de cache explicite unique pour un profilage direct.
- Remarque de débogage de performance :
  - `pnpm test:perf:imports` active le rapport de durée d’importation Vitest ainsi qu’une sortie de ventilation des imports.
  - `pnpm test:perf:imports:changed` limite cette même vue de profilage aux fichiers modifiés depuis `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` compare `test:changed` routé avec le chemin root-project natif pour ce diff validé et affiche le temps total ainsi que le RSS maximal sur macOS.
- `pnpm test:perf:changed:bench -- --worktree` mesure l’arbre de travail modifié actuel en routant la liste des fichiers modifiés via `scripts/test-projects.mjs` et la config root Vitest.
  - `pnpm test:perf:profile:main` écrit un profil CPU du thread principal pour le démarrage Vitest/Vite et la surcharge de transformation.
  - `pnpm test:perf:profile:runner` écrit des profils CPU+heap de l’exécuteur pour la suite unitaire avec le parallélisme de fichiers désactivé.

### E2E (smoke gateway)

- Commande : `pnpm test:e2e`
- Configuration : `vitest.e2e.config.ts`
- Fichiers : `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Valeurs d’exécution par défaut :
  - Utilise Vitest `threads` avec `isolate: false`, comme le reste du dépôt.
  - Utilise des workers adaptatifs (CI : jusqu’à 2, local : 1 par défaut).
  - S’exécute en mode silencieux par défaut pour réduire la surcharge d’E/S console.
- Remplacements utiles :
  - `OPENCLAW_E2E_WORKERS=<n>` pour forcer le nombre de workers (plafonné à 16).
  - `OPENCLAW_E2E_VERBOSE=1` pour réactiver la sortie console verbeuse.
- Portée :
  - Comportement end-to-end de gateway multi-instances
  - Surfaces WebSocket/HTTP, appairage de nœuds et réseau plus lourd
- Attentes :
  - S’exécute en CI (quand activé dans le pipeline)
  - Aucune vraie clé requise
  - Plus d’éléments mobiles que les tests unitaires (peut être plus lent)

### E2E : smoke du backend OpenShell

- Commande : `pnpm test:e2e:openshell`
- Fichier : `test/openshell-sandbox.e2e.test.ts`
- Portée :
  - Démarre une gateway OpenShell isolée sur l’hôte via Docker
  - Crée une sandbox à partir d’un Dockerfile local temporaire
  - Exerce le backend OpenShell d’OpenClaw via de vrais `sandbox ssh-config` + exécution SSH
  - Vérifie le comportement du système de fichiers canonique distant via le pont fs de la sandbox
- Attentes :
  - Uniquement sur opt-in ; ne fait pas partie de l’exécution par défaut `pnpm test:e2e`
  - Nécessite une CLI `openshell` locale et un démon Docker fonctionnel
  - Utilise `HOME` / `XDG_CONFIG_HOME` isolés, puis détruit la gateway et la sandbox de test
- Remplacements utiles :
  - `OPENCLAW_E2E_OPENSHELL=1` pour activer le test lors d’une exécution manuelle plus large de la suite e2e
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` pour pointer vers un binaire CLI non standard ou un script wrapper

### Live (vrais fournisseurs + vrais modèles)

- Commande : `pnpm test:live`
- Configuration : `vitest.live.config.ts`
- Fichiers : `src/**/*.live.test.ts`
- Par défaut : **activé** par `pnpm test:live` (définit `OPENCLAW_LIVE_TEST=1`)
- Portée :
  - « Ce fournisseur/modèle fonctionne-t-il réellement _aujourd’hui_ avec de vrais identifiants ? »
  - Détecter les changements de format des fournisseurs, les particularités d’appel d’outils, les problèmes d’authentification, et le comportement de rate limit
- Attentes :
  - Pas stable en CI par conception (réseaux réels, politiques fournisseurs réelles, quotas, pannes)
  - Coûte de l’argent / consomme des rate limits
  - Préférer des sous-ensembles limités plutôt que « tout »
- Les exécutions live chargent `~/.profile` pour récupérer les clés API manquantes.
- Par défaut, les exécutions live isolent toujours `HOME` et copient le matériel de config/auth vers un répertoire home temporaire de test afin que les fixtures unitaires ne modifient pas votre vrai `~/.openclaw`.
- Définissez `OPENCLAW_LIVE_USE_REAL_HOME=1` uniquement lorsque vous voulez volontairement que les tests live utilisent votre vrai répertoire home.
- `pnpm test:live` utilise maintenant par défaut un mode plus silencieux : il conserve la sortie de progression `[live] ...`, mais supprime l’avis supplémentaire `~/.profile` et coupe les logs de bootstrap de gateway / le bruit Bonjour. Définissez `OPENCLAW_LIVE_TEST_QUIET=0` si vous voulez récupérer les logs de démarrage complets.
- Rotation de clés API (spécifique au fournisseur) : définissez `*_API_KEYS` au format virgule/point-virgule ou `*_API_KEY_1`, `*_API_KEY_2` (par exemple `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) ou remplacez par live via `OPENCLAW_LIVE_*_KEY` ; les tests réessaient en cas de réponses de rate limit.
- Sortie de progression/heartbeat :
  - Les suites live émettent maintenant des lignes de progression vers stderr afin que les longs appels fournisseur restent visiblement actifs même quand la capture console Vitest est silencieuse.
  - `vitest.live.config.ts` désactive l’interception console de Vitest afin que les lignes de progression fournisseur/gateway soient diffusées immédiatement pendant les exécutions live.
  - Ajustez les heartbeats de modèle direct avec `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Ajustez les heartbeats de gateway/sonde avec `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Quelle suite dois-je exécuter ?

Utilisez ce tableau de décision :

- Modification de logique/tests : exécutez `pnpm test` (et `pnpm test:coverage` si vous avez beaucoup changé)
- Modification du réseau gateway / protocole WS / appairage : ajoutez `pnpm test:e2e`
- Débogage de « mon bot est en panne » / échecs spécifiques à un fournisseur / appel d’outils : exécutez un `pnpm test:live` réduit

## Live : balayage des capacités de nœud Android

- Test : `src/gateway/android-node.capabilities.live.test.ts`
- Script : `pnpm android:test:integration`
- Objectif : invoquer **chaque commande actuellement annoncée** par un nœud Android connecté et vérifier le comportement du contrat de commande.
- Portée :
  - Préconditionnée / configuration manuelle (la suite n’installe pas, n’exécute pas et n’appaire pas l’application).
  - Validation `node.invoke` gateway commande par commande pour le nœud Android sélectionné.
- Préconfiguration requise :
  - Application Android déjà connectée et appairée à la gateway.
  - Application gardée au premier plan.
  - Autorisations / consentements de capture accordés pour les capacités que vous attendez voir réussir.
- Remplacements de cible facultatifs :
  - `OPENCLAW_ANDROID_NODE_ID` ou `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Détails complets de configuration Android : [Application Android](/fr/platforms/android)

## Live : smoke de modèle (clés de profil)

Les tests live sont séparés en deux couches afin de pouvoir isoler les échecs :

- « Modèle direct » nous indique si le fournisseur/modèle peut répondre tout court avec la clé donnée.
- « Smoke gateway » nous indique si tout le pipeline gateway+agent fonctionne pour ce modèle (sessions, historique, outils, politique sandbox, etc.).

### Couche 1 : complétion de modèle direct (sans gateway)

- Test : `src/agents/models.profiles.live.test.ts`
- Objectif :
  - Énumérer les modèles découverts
  - Utiliser `getApiKeyForModel` pour sélectionner les modèles pour lesquels vous avez des identifiants
  - Exécuter une petite complétion par modèle (et des régressions ciblées si nécessaire)
- Comment activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
- Définissez `OPENCLAW_LIVE_MODELS=modern` (ou `all`, alias de modern) pour réellement exécuter cette suite ; sinon elle est ignorée afin de garder `pnpm test:live` centré sur le smoke gateway
- Comment sélectionner les modèles :
  - `OPENCLAW_LIVE_MODELS=modern` pour exécuter la liste d’autorisation moderne (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` est un alias de la liste d’autorisation moderne
  - ou `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (liste d’autorisation séparée par virgules)
- Comment sélectionner les fournisseurs :
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (liste d’autorisation séparée par virgules)
- D’où viennent les clés :
  - Par défaut : magasin de profils et replis env
  - Définissez `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour imposer **uniquement le magasin de profils**
- Pourquoi cela existe :
  - Sépare « l’API fournisseur est cassée / la clé est invalide » de « le pipeline d’agent gateway est cassé »
  - Contient de petites régressions isolées (exemple : replay du raisonnement OpenAI Responses/Codex Responses + flux d’appel d’outils)

### Couche 2 : smoke gateway + agent dev (ce que fait réellement "@openclaw")

- Test : `src/gateway/gateway-models.profiles.live.test.ts`
- Objectif :
  - Démarrer une gateway dans le processus
  - Créer/corriger une session `agent:dev:*` (remplacement de modèle à chaque exécution)
  - Itérer sur les modèles avec clés et vérifier :
    - réponse « utile » (sans outils)
    - qu’une véritable invocation d’outil fonctionne (sonde read)
    - des sondes d’outil supplémentaires facultatives (sonde exec+read)
    - que les chemins de régression OpenAI (tool-call-only → suivi) continuent de fonctionner
- Détails des sondes (pour expliquer rapidement les échecs) :
  - Sonde `read` : le test écrit un fichier nonce dans l’espace de travail et demande à l’agent de le `read` puis de renvoyer le nonce.
  - Sonde `exec+read` : le test demande à l’agent d’écrire via `exec` un nonce dans un fichier temporaire, puis de le relire avec `read`.
  - Sonde image : le test joint un PNG généré (chat + code aléatoire) et attend que le modèle renvoie `cat <CODE>`.
  - Référence d’implémentation : `src/gateway/gateway-models.profiles.live.test.ts` et `src/gateway/live-image-probe.ts`.
- Comment activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
- Comment sélectionner les modèles :
  - Par défaut : liste d’autorisation moderne (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` est un alias de la liste d’autorisation moderne
  - Ou définissez `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (ou une liste séparée par virgules) pour réduire
- Comment sélectionner les fournisseurs (éviter « tout OpenRouter ») :
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (liste d’autorisation séparée par virgules)
- Les sondes d’outil + image sont toujours activées dans ce test live :
  - sonde `read` + sonde `exec+read` (stress outils)
  - la sonde image s’exécute lorsque le modèle annonce la prise en charge des entrées image
  - Flux (haut niveau) :
    - Le test génère un petit PNG avec « CAT » + un code aléatoire (`src/gateway/live-image-probe.ts`)
    - L’envoie via `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - La gateway analyse les pièces jointes en `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - L’agent embarqué transmet un message utilisateur multimodal au modèle
    - Assertion : la réponse contient `cat` + le code (tolérance OCR : erreurs mineures autorisées)

Astuce : pour voir ce que vous pouvez tester sur votre machine (et les ids exacts `provider/model`), exécutez :

```bash
openclaw models list
openclaw models list --json
```

## Live : smoke du backend CLI (Claude, Codex, Gemini ou autres CLI locales)

- Test : `src/gateway/gateway-cli-backend.live.test.ts`
- Objectif : valider le pipeline Gateway + agent en utilisant un backend CLI local, sans toucher à votre configuration par défaut.
- Les valeurs par défaut de smoke spécifiques au backend vivent avec la définition `cli-backend.ts` de l’extension propriétaire.
- Activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Valeurs par défaut :
  - Fournisseur/modèle par défaut : `claude-cli/claude-sonnet-4-6`
  - Le comportement commande/args/image vient des métadonnées du plugin backend CLI propriétaire.
- Remplacements facultatifs :
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` pour envoyer une vraie pièce jointe image (les chemins sont injectés dans le prompt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` pour passer les chemins de fichiers image comme args CLI au lieu de l’injection dans le prompt.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (ou `"list"`) pour contrôler la manière dont les args image sont passés lorsque `IMAGE_ARG` est défini.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` pour envoyer un deuxième tour et valider le flux de reprise.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=0` pour désactiver la sonde de continuité de session par défaut Claude Sonnet -> Opus dans une même session (définissez `1` pour la forcer lorsque le modèle sélectionné prend en charge une cible de bascule).

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

Recettes Docker mono-fournisseur :

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:codex
pnpm test:docker:live-cli-backend:gemini
```

Notes :

- L’exécuteur Docker se trouve dans `scripts/test-live-cli-backend-docker.sh`.
- Il exécute le smoke live du backend CLI dans l’image Docker du dépôt en tant qu’utilisateur non root `node`.
- Il résout les métadonnées de smoke CLI depuis l’extension propriétaire, puis installe le package CLI Linux correspondant (`@anthropic-ai/claude-code`, `@openai/codex`, ou `@google/gemini-cli`) dans un préfixe inscriptible mis en cache à `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (par défaut : `~/.cache/openclaw/docker-cli-tools`).
- Le smoke live du backend CLI exerce maintenant le même flux end-to-end pour Claude, Codex et Gemini : tour texte, tour de classification d’image, puis appel d’outil MCP `cron` vérifié via la CLI gateway.
- Le smoke par défaut de Claude corrige aussi la session de Sonnet vers Opus et vérifie que la session reprise se souvient toujours d’une note antérieure.

## Live : smoke de liaison ACP (`/acp spawn ... --bind here`)

- Test : `src/gateway/gateway-acp-bind.live.test.ts`
- Objectif : valider le vrai flux de liaison de conversation ACP avec un agent ACP live :
  - envoyer `/acp spawn <agent> --bind here`
  - lier sur place une conversation synthétique de canal de message
  - envoyer un suivi normal dans cette même conversation
  - vérifier que le suivi arrive dans la transcription de la session ACP liée
- Activer :
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Valeurs par défaut :
  - Agents ACP dans Docker : `claude,codex,gemini`
  - Agent ACP pour `pnpm test:live ...` direct : `claude`
  - Canal synthétique : contexte de conversation de type DM Slack
  - Backend ACP : `acpx`
- Remplacements :
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Notes :
  - Cette voie utilise la surface gateway `chat.send` avec des champs de route d’origine synthétiques réservés à l’administrateur afin que les tests puissent attacher un contexte de canal de message sans prétendre livrer à l’extérieur.
  - Lorsque `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` n’est pas défini, le test utilise le registre d’agents intégré du plugin embarqué `acpx` pour l’agent de harnais ACP sélectionné.

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

Recettes Docker mono-agent :

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:gemini
```

Notes Docker :

- L’exécuteur Docker se trouve dans `scripts/test-live-acp-bind-docker.sh`.
- Par défaut, il exécute le smoke de liaison ACP sur tous les agents CLI live pris en charge, en séquence : `claude`, `codex`, puis `gemini`.
- Utilisez `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`, `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`, ou `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` pour réduire la matrice.
- Il charge `~/.profile`, prépare le matériel d’auth CLI correspondant dans le conteneur, installe `acpx` dans un préfixe npm inscriptible, puis installe la CLI live demandée (`@anthropic-ai/claude-code`, `@openai/codex`, ou `@google/gemini-cli`) si elle manque.
- Dans Docker, l’exécuteur définit `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` afin que acpx conserve pour la CLI harnais enfant les variables d’environnement fournisseur issues du profile chargé.

### Recettes live recommandées

Des listes d’autorisation explicites et réduites sont les plus rapides et les moins fragiles :

- Modèle unique, direct (sans gateway) :
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Modèle unique, smoke gateway :
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Appel d’outils sur plusieurs fournisseurs :
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Focus Google (clé API Gemini + Antigravity) :
  - Gemini (clé API) : `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth) : `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Notes :

- `google/...` utilise l’API Gemini (clé API).
- `google-antigravity/...` utilise le pont OAuth Antigravity (endpoint d’agent de type Cloud Code Assist).
- `google-gemini-cli/...` utilise la CLI Gemini locale sur votre machine (auth distincte + particularités d’outillage).
- API Gemini vs CLI Gemini :
  - API : OpenClaw appelle l’API Gemini hébergée de Google via HTTP (clé API / auth de profil) ; c’est ce que la plupart des utilisateurs entendent par « Gemini ».
  - CLI : OpenClaw exécute un binaire local `gemini` ; il a sa propre auth et peut se comporter différemment (streaming/prise en charge des outils/décalage de version).

## Live : matrice de modèles (ce que nous couvrons)

Il n’existe pas de « liste de modèles CI » fixe (live est sur opt-in), mais ce sont les modèles **recommandés** à couvrir régulièrement sur une machine de développement disposant de clés.

### Ensemble de smoke moderne (appel d’outils + image)

C’est l’exécution « modèles courants » que nous attendons de garder fonctionnelle :

- OpenAI (hors Codex) : `openai/gpt-5.4` (facultatif : `openai/gpt-5.4-mini`)
- OpenAI Codex : `openai-codex/gpt-5.4`
- Anthropic : `anthropic/claude-opus-4-6` (ou `anthropic/claude-sonnet-4-6`)
- Google (API Gemini) : `google/gemini-3.1-pro-preview` et `google/gemini-3-flash-preview` (évitez les anciens modèles Gemini 2.x)
- Google (Antigravity) : `google-antigravity/claude-opus-4-6-thinking` et `google-antigravity/gemini-3-flash`
- Z.AI (GLM) : `zai/glm-4.7`
- MiniMax : `minimax/MiniMax-M2.7`

Exécuter le smoke gateway avec outils + image :
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Référence : appel d’outils (Read + Exec facultatif)

Choisissez-en au moins un par famille de fournisseur :

- OpenAI : `openai/gpt-5.4` (ou `openai/gpt-5.4-mini`)
- Anthropic : `anthropic/claude-opus-4-6` (ou `anthropic/claude-sonnet-4-6`)
- Google : `google/gemini-3-flash-preview` (ou `google/gemini-3.1-pro-preview`)
- Z.AI (GLM) : `zai/glm-4.7`
- MiniMax : `minimax/MiniMax-M2.7`

Couverture supplémentaire facultative (agréable à avoir) :

- xAI : `xai/grok-4` (ou la dernière version disponible)
- Mistral : `mistral/`… (choisissez un modèle « tools » activé)
- Cerebras : `cerebras/`… (si vous y avez accès)
- LM Studio : `lmstudio/`… (local ; l’appel d’outils dépend du mode API)

### Vision : envoi d’image (pièce jointe → message multimodal)

Incluez au moins un modèle capable de traiter des images dans `OPENCLAW_LIVE_GATEWAY_MODELS` (variants Claude/Gemini/OpenAI compatibles vision, etc.) pour exercer la sonde image.

### Agrégateurs / passerelles alternatives

Si vous avez activé les clés, nous prenons aussi en charge les tests via :

- OpenRouter : `openrouter/...` (des centaines de modèles ; utilisez `openclaw models scan` pour trouver des candidats compatibles outils+image)
- OpenCode : `opencode/...` pour Zen et `opencode-go/...` pour Go (auth via `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Autres fournisseurs que vous pouvez inclure dans la matrice live (si vous avez les identifiants/la config) :

- Intégrés : `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Via `models.providers` (endpoints personnalisés) : `minimax` (cloud/API), plus tout proxy compatible OpenAI/Anthropic (LM Studio, vLLM, LiteLLM, etc.)

Astuce : n’essayez pas de figer « tous les modèles » dans la documentation. La liste de référence est ce que `discoverModels(...)` renvoie sur votre machine + les clés réellement disponibles.

## Identifiants (ne jamais commit)

Les tests live découvrent les identifiants de la même manière que la CLI. Conséquences pratiques :

- Si la CLI fonctionne, les tests live devraient trouver les mêmes clés.
- Si un test live indique « pas d’identifiants », déboguez de la même façon que pour `openclaw models list` / la sélection de modèle.

- Profils d’auth par agent : `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (c’est ce que « profile keys » signifie dans les tests live)
- Configuration : `~/.openclaw/openclaw.json` (ou `OPENCLAW_CONFIG_PATH`)
- Ancien répertoire d’état : `~/.openclaw/credentials/` (copié dans le home live préparé lorsqu’il est présent, mais pas le magasin principal de clés de profil)
- Les exécutions live locales copient par défaut la config active, les fichiers `auth-profiles.json` par agent, l’ancien répertoire `credentials/`, ainsi que les répertoires d’auth CLI externes pris en charge dans un home de test temporaire ; les homes live préparés ignorent `workspace/` et `sandboxes/`, et les remplacements de chemin `agents.*.workspace` / `agentDir` sont supprimés afin que les sondes n’utilisent pas votre véritable espace de travail hôte.

Si vous voulez vous appuyer sur des clés env (par ex. exportées dans votre `~/.profile`), exécutez les tests locaux après `source ~/.profile`, ou utilisez les exécuteurs Docker ci-dessous (ils peuvent monter `~/.profile` dans le conteneur).

## Live Deepgram (transcription audio)

- Test : `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Activer : `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## Live plan de codage BytePlus

- Test : `src/agents/byteplus.live.test.ts`
- Activer : `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Remplacement facultatif de modèle : `BYTEPLUS_CODING_MODEL=ark-code-latest`

## Live média de workflow ComfyUI

- Test : `extensions/comfy/comfy.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Portée :
  - Exerce les chemins image, vidéo et `music_generate` du bundle comfy
  - Ignore chaque capacité sauf si `models.providers.comfy.<capability>` est configuré
  - Utile après avoir modifié la soumission de workflow comfy, le polling, les téléchargements ou l’enregistrement du plugin

## Live génération d’image

- Test : `src/image-generation/runtime.live.test.ts`
- Commande : `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harnais : `pnpm test:live:media image`
- Portée :
  - Énumère chaque plugin fournisseur de génération d’image enregistré
  - Charge les variables d’environnement fournisseur manquantes depuis votre shell de connexion (`~/.profile`) avant les sondes
  - Utilise par défaut les clés API live/env avant les profils d’auth stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les vrais identifiants du shell
  - Ignore les fournisseurs sans auth/profil/modèle utilisable
  - Exécute les variantes standard de génération d’image via la capacité runtime partagée :
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Fournisseurs intégrés actuellement couverts :
  - `openai`
  - `google`
- Réduction facultative :
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Comportement d’auth facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l’auth via le magasin de profils et ignorer les remplacements uniquement env

## Live génération musicale

- Test : `extensions/music-generation-providers.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harnais : `pnpm test:live:media music`
- Portée :
  - Exerce le chemin partagé des fournisseurs de génération musicale intégrés
  - Couvre actuellement Google et MiniMax
  - Charge les variables d’environnement fournisseur depuis votre shell de connexion (`~/.profile`) avant les sondes
  - Utilise par défaut les clés API live/env avant les profils d’auth stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les vrais identifiants du shell
  - Ignore les fournisseurs sans auth/profil/modèle utilisable
  - Exécute les deux modes runtime déclarés lorsqu’ils sont disponibles :
    - `generate` avec entrée uniquement prompt
    - `edit` lorsque le fournisseur déclare `capabilities.edit.enabled`
  - Couverture actuelle de la voie partagée :
    - `google` : `generate`, `edit`
    - `minimax` : `generate`
    - `comfy` : fichier live Comfy séparé, pas ce balayage partagé
- Réduction facultative :
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Comportement d’auth facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l’auth via le magasin de profils et ignorer les remplacements uniquement env

## Live génération vidéo

- Test : `extensions/video-generation-providers.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harnais : `pnpm test:live:media video`
- Portée :
  - Exerce le chemin partagé des fournisseurs de génération vidéo intégrés
  - Charge les variables d’environnement fournisseur depuis votre shell de connexion (`~/.profile`) avant les sondes
  - Utilise par défaut les clés API live/env avant les profils d’auth stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les vrais identifiants du shell
  - Ignore les fournisseurs sans auth/profil/modèle utilisable
  - Exécute les deux modes runtime déclarés lorsqu’ils sont disponibles :
    - `generate` avec entrée uniquement prompt
    - `imageToVideo` lorsque le fournisseur déclare `capabilities.imageToVideo.enabled` et que le fournisseur/modèle sélectionné accepte en balayage partagé une entrée image locale basée sur buffer
    - `videoToVideo` lorsque le fournisseur déclare `capabilities.videoToVideo.enabled` et que le fournisseur/modèle sélectionné accepte en balayage partagé une entrée vidéo locale basée sur buffer
  - Fournisseurs `imageToVideo` actuellement déclarés mais ignorés dans le balayage partagé :
    - `vydra` parce que le bundle `veo3` est texte uniquement et que le bundle `kling` exige une URL d’image distante
  - Couverture Vydra spécifique au fournisseur :
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - ce fichier exécute le texte-vers-vidéo `veo3` plus une voie `kling` qui utilise par défaut une fixture d’URL d’image distante
  - Couverture live actuelle de `videoToVideo` :
    - `runway` uniquement lorsque le modèle sélectionné est `runway/gen4_aleph`
  - Fournisseurs `videoToVideo` actuellement déclarés mais ignorés dans le balayage partagé :
    - `alibaba`, `qwen`, `xai` parce que ces chemins exigent actuellement des URL de référence distantes `http(s)` / MP4
    - `google` parce que la voie partagée Gemini/Veo actuelle utilise une entrée locale basée sur buffer et que ce chemin n’est pas accepté dans le balayage partagé
    - `openai` parce que la voie partagée actuelle ne garantit pas l’accès vidéo inpaint/remix spécifique à l’organisation
- Réduction facultative :
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- Comportement d’auth facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l’auth via le magasin de profils et ignorer les remplacements uniquement env

## Harnais live média

- Commande : `pnpm test:live:media`
- Objectif :
  - Exécute les suites live partagées image, musique et vidéo via un point d’entrée natif du dépôt
  - Charge automatiquement les variables d’environnement fournisseur manquantes depuis `~/.profile`
  - Réduit automatiquement chaque suite aux fournisseurs qui disposent actuellement d’une auth utilisable par défaut
  - Réutilise `scripts/test-live.mjs`, pour que le comportement heartbeat et mode silencieux reste cohérent
- Exemples :
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Exécuteurs Docker (vérifications facultatives « fonctionne sous Linux »)

Ces exécuteurs Docker se répartissent en deux groupes :

- Exécuteurs live de modèles : `test:docker:live-models` et `test:docker:live-gateway` n’exécutent que leur fichier live de clés de profil correspondant dans l’image Docker du dépôt (`src/agents/models.profiles.live.test.ts` et `src/gateway/gateway-models.profiles.live.test.ts`), en montant votre répertoire de config local et votre espace de travail (et en chargeant `~/.profile` s’il est monté). Les points d’entrée locaux correspondants sont `test:live:models-profiles` et `test:live:gateway-profiles`.
- Les exécuteurs live Docker utilisent par défaut une limite de smoke plus petite afin qu’un balayage Docker complet reste pratique :
  `test:docker:live-models` utilise par défaut `OPENCLAW_LIVE_MAX_MODELS=12`, et
  `test:docker:live-gateway` utilise par défaut `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`, et
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Remplacez ces variables d’environnement lorsque vous
  voulez explicitement le balayage exhaustif plus large.
- `test:docker:all` construit l’image Docker live une seule fois via `test:docker:live-build`, puis la réutilise pour les deux voies Docker live.
- Les exécuteurs smoke conteneur : `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels`, et `test:docker:plugins` démarrent un ou plusieurs vrais conteneurs et vérifient des chemins d’intégration de plus haut niveau.

Les exécuteurs live Docker de modèles montent aussi en bind uniquement les homes d’auth CLI nécessaires (ou tous ceux pris en charge quand l’exécution n’est pas réduite), puis les copient dans le home du conteneur avant l’exécution afin que l’OAuth des CLI externes puisse rafraîchir les jetons sans modifier le magasin d’auth de l’hôte :

- Modèles directs : `pnpm test:docker:live-models` (script : `scripts/test-live-models-docker.sh`)
- Smoke de liaison ACP : `pnpm test:docker:live-acp-bind` (script : `scripts/test-live-acp-bind-docker.sh`)
- Smoke backend CLI : `pnpm test:docker:live-cli-backend` (script : `scripts/test-live-cli-backend-docker.sh`)
- Gateway + agent dev : `pnpm test:docker:live-gateway` (script : `scripts/test-live-gateway-models-docker.sh`)
- Smoke live Open WebUI : `pnpm test:docker:openwebui` (script : `scripts/e2e/openwebui-docker.sh`)
- Assistant d’onboarding (TTY, scaffolding complet) : `pnpm test:docker:onboard` (script : `scripts/e2e/onboard-docker.sh`)
- Réseau gateway (deux conteneurs, auth WS + santé) : `pnpm test:docker:gateway-network` (script : `scripts/e2e/gateway-network-docker.sh`)
- Pont de canal MCP (Gateway préparée + pont stdio + smoke brut de trame de notification Claude) : `pnpm test:docker:mcp-channels` (script : `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (smoke d’installation + alias `/plugin` + sémantique de redémarrage du bundle Claude) : `pnpm test:docker:plugins` (script : `scripts/e2e/plugins-docker.sh`)

Les exécuteurs live Docker de modèles montent aussi en lecture seule l’extraction courante et
la préparent dans un répertoire de travail temporaire dans le conteneur. Cela garde l’image d’exécution
légère tout en exécutant Vitest sur votre source/configuration locale exacte.
L’étape de préparation ignore les gros caches locaux et les sorties de build d’application comme
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__`, et les répertoires `.build` ou
de sortie Gradle locaux aux applications, afin que les exécutions live Docker ne passent pas des minutes à copier
des artefacts spécifiques à la machine.
Ils définissent aussi `OPENCLAW_SKIP_CHANNELS=1` afin que les sondes live gateway ne démarrent pas
de vrais workers de canal Telegram/Discord/etc. dans le conteneur.
`test:docker:live-models` exécute toujours `pnpm test:live`, donc faites aussi passer
`OPENCLAW_LIVE_GATEWAY_*` lorsque vous devez réduire ou exclure la couverture
live gateway de cette voie Docker.
`test:docker:openwebui` est un smoke de compatibilité de plus haut niveau : il démarre une
gateway OpenClaw en conteneur avec les endpoints HTTP compatibles OpenAI activés,
démarre un conteneur Open WebUI épinglé contre cette gateway, se connecte via
Open WebUI, vérifie que `/api/models` expose `openclaw/default`, puis envoie une
vraie requête de chat via le proxy `/api/chat/completions` d’Open WebUI.
La première exécution peut être sensiblement plus lente parce que Docker peut devoir récupérer
l’image Open WebUI et qu’Open WebUI peut devoir terminer sa propre configuration de démarrage à froid.
Cette voie attend une clé de modèle live utilisable, et `OPENCLAW_PROFILE_FILE`
(`~/.profile` par défaut) est le moyen principal de la fournir dans les exécutions Docker.
Les exécutions réussies affichent une petite charge utile JSON comme `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` est volontairement déterministe et n’a pas besoin d’un
vrai compte Telegram, Discord ou iMessage. Il démarre un conteneur Gateway
préparé, démarre un deuxième conteneur qui lance `openclaw mcp serve`, puis
vérifie la découverte de conversation routée, les lectures de transcription, les métadonnées
de pièce jointe, le comportement de file d’événements live, le routage d’envoi sortant, et les notifications de type canal + permission de style Claude sur le vrai pont stdio MCP. La vérification de notification
inspecte directement les trames MCP stdio brutes afin que le smoke valide ce que le
pont émet réellement, et pas seulement ce qu’un SDK client particulier choisit de faire remonter.

Smoke manuel ACP en langage naturel sur thread (pas CI) :

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Conservez ce script pour les workflows de régression/débogage. Il pourra être nécessaire à nouveau pour valider le routage ACP sur thread, donc ne le supprimez pas.

Variables d’environnement utiles :

- `OPENCLAW_CONFIG_DIR=...` (par défaut : `~/.openclaw`) monté sur `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (par défaut : `~/.openclaw/workspace`) monté sur `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (par défaut : `~/.profile`) monté sur `/home/node/.profile` et chargé avant l’exécution des tests
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (par défaut : `~/.cache/openclaw/docker-cli-tools`) monté sur `/home/node/.npm-global` pour les installations CLI mises en cache dans Docker
- Les répertoires/fichiers d’auth CLI externes sous `$HOME` sont montés en lecture seule sous `/host-auth...`, puis copiés dans `/home/node/...` avant le démarrage des tests
  - Répertoires par défaut : `.minimax`
  - Fichiers par défaut : `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Les exécutions réduites par fournisseur ne montent que les répertoires/fichiers nécessaires déduits depuis `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Remplacement manuel possible avec `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none`, ou une liste séparée par virgules comme `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` pour réduire l’exécution
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` pour filtrer les fournisseurs dans le conteneur
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour s’assurer que les identifiants proviennent du magasin de profils (pas de env)
- `OPENCLAW_OPENWEBUI_MODEL=...` pour choisir le modèle exposé par la gateway pour le smoke Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` pour remplacer le prompt de vérification par nonce utilisé par le smoke Open WebUI
- `OPENWEBUI_IMAGE=...` pour remplacer le tag d’image Open WebUI épinglé

## Vérification rapide de la documentation

Exécutez les vérifications de docs après des modifications de documentation : `pnpm check:docs`.
Exécutez la validation complète des ancres Mintlify quand vous avez aussi besoin de vérifier les titres dans la page : `pnpm docs:check-links:anchors`.

## Régression hors ligne (compatible CI)

Ce sont des régressions de « pipeline réel » sans vrais fournisseurs :

- Appel d’outils gateway (mock OpenAI, vraie boucle gateway + agent) : `src/gateway/gateway.test.ts` (cas : "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Assistant gateway (WS `wizard.start`/`wizard.next`, écrit config + auth imposée) : `src/gateway/gateway.test.ts` (cas : "runs wizard over ws and writes auth token config")

## Évaluations de fiabilité d’agent (Skills)

Nous avons déjà quelques tests compatibles CI qui se comportent comme des « évaluations de fiabilité d’agent » :

- Mock d’appel d’outils via la vraie boucle gateway + agent (`src/gateway/gateway.test.ts`).
- Flux de wizard end-to-end qui valident le câblage des sessions et les effets de configuration (`src/gateway/gateway.test.ts`).

Ce qui manque encore pour les Skills (voir [Skills](/fr/tools/skills)) :

- **Prise de décision :** lorsque les Skills sont listés dans le prompt, l’agent choisit-il le bon Skill (ou évite-t-il les Skills non pertinents) ?
- **Conformité :** l’agent lit-il `SKILL.md` avant utilisation et suit-il les étapes/args requis ?
- **Contrats de workflow :** scénarios multi-tours qui vérifient l’ordre des outils, le report de l’historique de session, et les limites de sandbox.

Les futures évaluations doivent rester déterministes d’abord :

- Un exécuteur de scénarios utilisant des fournisseurs mock pour vérifier les appels d’outils + leur ordre, les lectures de fichiers de Skill, et le câblage de session.
- Une petite suite de scénarios centrés sur les Skills (utiliser vs éviter, gating, injection de prompt).
- Des évaluations live facultatives (opt-in, protégées par env) seulement une fois la suite compatible CI en place.

## Tests de contrat (forme des plugins et des canaux)

Les tests de contrat vérifient que chaque plugin et canal enregistré respecte son
contrat d’interface. Ils itèrent sur tous les plugins découverts et exécutent une suite
d’assertions de forme et de comportement. La voie unitaire par défaut `pnpm test`
ignore volontairement ces fichiers de seam partagé et de smoke ; exécutez explicitement
les commandes de contrat lorsque vous modifiez des surfaces partagées de canal ou de fournisseur.

### Commandes

- Tous les contrats : `pnpm test:contracts`
- Contrats de canal uniquement : `pnpm test:contracts:channels`
- Contrats de fournisseur uniquement : `pnpm test:contracts:plugins`

### Contrats de canal

Situés dans `src/channels/plugins/contracts/*.contract.test.ts` :

- **plugin** - Forme de base du plugin (id, nom, capacités)
- **setup** - Contrat de l’assistant de configuration
- **session-binding** - Comportement de liaison de session
- **outbound-payload** - Structure de la charge utile du message
- **inbound** - Gestion des messages entrants
- **actions** - Gestionnaires d’actions de canal
- **threading** - Gestion des ids de thread
- **directory** - API d’annuaire/de roster
- **group-policy** - Application de la politique de groupe

### Contrats de statut des fournisseurs

Situés dans `src/plugins/contracts/*.contract.test.ts`.

- **status** - Sondes de statut de canal
- **registry** - Forme du registre des plugins

### Contrats de fournisseur

Situés dans `src/plugins/contracts/*.contract.test.ts` :

- **auth** - Contrat du flux d’authentification
- **auth-choice** - Choix/sélection d’authentification
- **catalog** - API de catalogue de modèles
- **discovery** - Découverte de plugins
- **loader** - Chargement de plugin
- **runtime** - Runtime fournisseur
- **shape** - Forme/interface du plugin
- **wizard** - Assistant de configuration

### Quand les exécuter

- Après modification des exports ou sous-chemins `plugin-sdk`
- Après ajout ou modification d’un plugin de canal ou de fournisseur
- Après refactorisation de l’enregistrement ou de la découverte de plugins

Les tests de contrat s’exécutent en CI et ne nécessitent pas de vraies clés API.

## Ajout de régressions (recommandations)

Lorsque vous corrigez un problème de fournisseur/modèle découvert en live :

- Ajoutez si possible une régression compatible CI (fournisseur mock/stub, ou capture de la transformation exacte de la forme de requête)
- Si le problème est intrinsèquement live-only (rate limits, politiques d’auth), gardez le test live réduit et activé via des variables d’environnement
- Préférez cibler la plus petite couche qui attrape le bug :
  - bug de conversion/replay de requête fournisseur → test de modèles directs
  - bug de pipeline gateway session/historique/outils → smoke live gateway ou test mock gateway compatible CI
- Garde-fou de traversée SecretRef :
  - `src/secrets/exec-secret-ref-id-parity.test.ts` dérive une cible échantillon par classe de SecretRef depuis les métadonnées du registre (`listSecretTargetRegistryEntries()`), puis vérifie que les ids exec de segment de traversée sont rejetés.
  - Si vous ajoutez une nouvelle famille de cibles SecretRef `includeInPlan` dans `src/secrets/target-registry-data.ts`, mettez à jour `classifyTargetClass` dans ce test. Le test échoue volontairement sur les ids de cible non classés afin qu’aucune nouvelle classe ne puisse être ignorée silencieusement.
