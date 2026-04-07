---
read_when:
    - Exécuter des tests localement ou en CI
    - Ajouter des régressions pour des bugs de modèle/fournisseur
    - Déboguer le comportement de la gateway et de l’agent
summary: 'Kit de test : suites unitaires/e2e/live, exécuteurs Docker et couverture de chaque test'
title: Tests
x-i18n:
    generated_at: "2026-04-07T06:52:43Z"
    model: gpt-5.4
    provider: openai
    source_hash: 77c61126344d03c7b04ccf1f9aba0381cf8c7c73042d69b2d9f3f07a5eba70d3
    source_path: help/testing.md
    workflow: 15
---

# Tests

OpenClaw dispose de trois suites Vitest (unitaires/intégration, e2e, live) et d’un petit ensemble d’exécuteurs Docker.

Cette documentation est un guide « comment nous testons » :

- Ce que couvre chaque suite (et ce qu’elle ne couvre délibérément _pas_)
- Quelles commandes exécuter pour les flux courants (local, avant push, débogage)
- Comment les tests live découvrent les identifiants et sélectionnent les modèles/fournisseurs
- Comment ajouter des régressions pour des problèmes réels de modèle/fournisseur

## Démarrage rapide

La plupart du temps :

- Barrière complète (attendue avant push) : `pnpm build && pnpm check && pnpm test`
- Exécution locale plus rapide de la suite complète sur une machine bien dotée : `pnpm test:max`
- Boucle directe de surveillance Vitest : `pnpm test:watch`
- Le ciblage direct de fichier passe désormais aussi par les chemins d’extension/canal : `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- Site QA adossé à Docker : `pnpm qa:lab:up`

Quand vous modifiez des tests ou voulez plus de confiance :

- Barrière de couverture : `pnpm test:coverage`
- Suite E2E : `pnpm test:e2e`

Lors du débogage de vrais fournisseurs/modèles (nécessite de vrais identifiants) :

- Suite live (modèles + sondes de gateway d’outils/images) : `pnpm test:live`
- Cibler silencieusement un seul fichier live : `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Conseil : lorsque vous n’avez besoin que d’un seul cas en échec, préférez restreindre les tests live via les variables d’environnement de liste d’autorisation décrites ci-dessous.

## Suites de tests (ce qui s’exécute où)

Considérez les suites comme « de plus en plus réalistes » (et de plus en plus fragiles/coûteuses) :

### Unitaire / intégration (par défaut)

- Commande : `pnpm test`
- Config : dix exécutions de fragments séquentielles (`vitest.full-*.config.ts`) sur les projets Vitest ciblés existants
- Fichiers : inventaires core/unit sous `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts`, ainsi que les tests Node `ui` en liste d’autorisation couverts par `vitest.unit.config.ts`
- Portée :
  - Tests unitaires purs
  - Tests d’intégration en processus (authentification gateway, routage, outils, parsing, config)
  - Régressions déterministes pour des bugs connus
- Attentes :
  - S’exécute en CI
  - Aucune vraie clé requise
  - Doit être rapide et stable
- Remarque sur les projets :
  - `pnpm test` sans ciblage exécute désormais dix configurations de fragments plus petites (`core-unit-src`, `core-unit-security`, `core-unit-ui`, `core-unit-support`, `core-contracts`, `core-bundled`, `core-runtime`, `agentic`, `auto-reply`, `extensions`) au lieu d’un seul énorme processus projet racine natif. Cela réduit le RSS maximal sur les machines chargées et évite que le travail auto-reply/extension n’affame les suites non liées.
  - `pnpm test --watch` utilise toujours le graphe de projets racine natif `vitest.config.ts`, car une boucle de surveillance multi-fragments n’est pas pratique.
  - `pnpm test`, `pnpm test:watch` et `pnpm test:perf:imports` acheminent d’abord les cibles explicites fichier/répertoire vers des voies ciblées, donc `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` évite le coût de démarrage complet du projet racine.
  - `pnpm test:changed` étend les chemins git modifiés vers ces mêmes voies ciblées lorsque le diff ne touche que des fichiers source/test routables ; les modifications de config/setup reviennent toujours à une réexécution large du projet racine.
  - Certains tests `plugin-sdk` et `commands` passent aussi par des voies dédiées légères qui sautent `test/setup-openclaw-runtime.ts` ; les fichiers avec état/runtime lourd restent sur les voies existantes.
  - Certains fichiers source d’assistance `plugin-sdk` et `commands` mappent aussi les exécutions en mode changed vers des tests frères explicites dans ces voies légères, afin que les modifications d’helpers évitent de relancer la suite lourde complète pour ce répertoire.
  - `auto-reply` dispose maintenant de trois catégories dédiées : helpers core de niveau supérieur, tests d’intégration `reply.*` de niveau supérieur et sous-arborescence `src/auto-reply/reply/**`. Cela garde le travail le plus lourd du harness reply hors des tests bon marché de statut/chunk/token.
- Remarque sur l’exécuteur intégré :
  - Lorsque vous modifiez les entrées de découverte des outils de message ou le contexte runtime de compaction,
    conservez les deux niveaux de couverture.
  - Ajoutez des régressions ciblées d’helpers pour les frontières pures de routage/normalisation.
  - Conservez aussi en bon état les suites d’intégration de l’exécuteur intégré :
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`, et
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Ces suites vérifient que les ids ciblés et le comportement de compaction continuent de passer
    par les véritables chemins `run.ts` / `compact.ts` ; des tests d’helpers seuls ne
    constituent pas un substitut suffisant à ces chemins d’intégration.
- Remarque sur le pool :
  - La configuration Vitest de base utilise désormais `threads` par défaut.
  - La configuration Vitest partagée fixe aussi `isolate: false` et utilise l’exécuteur non isolé sur les projets racine, les configs e2e et live.
  - La voie UI racine conserve sa configuration `jsdom` et son optimiseur, mais s’exécute maintenant elle aussi sur l’exécuteur partagé non isolé.
  - Chaque fragment `pnpm test` hérite des mêmes valeurs par défaut `threads` + `isolate: false` depuis la configuration Vitest partagée.
  - Le lanceur partagé `scripts/run-vitest.mjs` ajoute aussi désormais `--no-maglev` par défaut pour les processus Node enfants de Vitest afin de réduire le churn de compilation V8 lors des grosses exécutions locales. Définissez `OPENCLAW_VITEST_ENABLE_MAGLEV=1` si vous devez comparer avec le comportement V8 standard.
- Remarque sur l’itération locale rapide :
  - `pnpm test:changed` passe par des voies ciblées lorsque les chemins modifiés correspondent clairement à une suite plus petite.
  - `pnpm test:max` et `pnpm test:changed:max` conservent le même comportement de routage, simplement avec un plafond de workers plus élevé.
  - L’auto-dimensionnement local des workers est désormais volontairement prudent et réduit aussi l’allure lorsque la charge moyenne de l’hôte est déjà élevée, de sorte que plusieurs exécutions Vitest concurrentes fassent moins de dégâts par défaut.
  - La configuration Vitest de base marque les projets/fichiers de config comme `forceRerunTriggers` afin que les réexécutions en mode changed restent correctes lorsque le câblage des tests change.
  - La config garde `OPENCLAW_VITEST_FS_MODULE_CACHE` activé sur les hôtes pris en charge ; définissez `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` si vous voulez un emplacement de cache explicite pour le profilage direct.
- Remarque sur le débogage des performances :
  - `pnpm test:perf:imports` active le rapport de durée d’import Vitest ainsi que la ventilation des imports.
  - `pnpm test:perf:imports:changed` limite cette même vue de profilage aux fichiers modifiés depuis `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` compare le `test:changed` routé avec le chemin natif du projet racine pour ce diff validé et affiche le temps mur ainsi que le RSS maximal macOS.
- `pnpm test:perf:changed:bench -- --worktree` mesure l’arbre de travail sale courant en routant la liste des fichiers modifiés via `scripts/test-projects.mjs` et la config Vitest racine.
  - `pnpm test:perf:profile:main` écrit un profil CPU du thread principal pour la surcharge de démarrage et de transformation Vitest/Vite.
  - `pnpm test:perf:profile:runner` écrit des profils CPU+heap de l’exécuteur pour la suite unitaire avec le parallélisme de fichiers désactivé.

### E2E (smoke gateway)

- Commande : `pnpm test:e2e`
- Config : `vitest.e2e.config.ts`
- Fichiers : `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Valeurs runtime par défaut :
  - Utilise Vitest `threads` avec `isolate: false`, comme le reste du dépôt.
  - Utilise des workers adaptatifs (CI : jusqu’à 2, local : 1 par défaut).
  - S’exécute en mode silencieux par défaut pour réduire la surcharge d’E/S console.
- Remplacements utiles :
  - `OPENCLAW_E2E_WORKERS=<n>` pour forcer le nombre de workers (plafonné à 16).
  - `OPENCLAW_E2E_VERBOSE=1` pour réactiver la sortie console détaillée.
- Portée :
  - Comportement end-to-end gateway multi-instances
  - Surfaces WebSocket/HTTP, appairage de nœuds et réseau plus lourd
- Attentes :
  - S’exécute en CI (lorsqu’activé dans le pipeline)
  - Aucune vraie clé requise
  - Plus d’éléments mobiles que les tests unitaires (peut être plus lent)

### E2E : smoke backend OpenShell

- Commande : `pnpm test:e2e:openshell`
- Fichier : `test/openshell-sandbox.e2e.test.ts`
- Portée :
  - Démarre une gateway OpenShell isolée sur l’hôte via Docker
  - Crée un sandbox à partir d’un Dockerfile local temporaire
  - Exerce le backend OpenShell d’OpenClaw via de vrais `sandbox ssh-config` + exécution SSH
  - Vérifie le comportement du système de fichiers canonique distant via le pont fs du sandbox
- Attentes :
  - Opt-in uniquement ; ne fait pas partie de l’exécution par défaut `pnpm test:e2e`
  - Nécessite une CLI locale `openshell` ainsi qu’un démon Docker opérationnel
  - Utilise `HOME` / `XDG_CONFIG_HOME` isolés, puis détruit la gateway de test et le sandbox
- Remplacements utiles :
  - `OPENCLAW_E2E_OPENSHELL=1` pour activer le test lors d’une exécution manuelle de la suite e2e plus large
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` pour pointer vers un binaire CLI non par défaut ou un script wrapper

### Live (vrais fournisseurs + vrais modèles)

- Commande : `pnpm test:live`
- Config : `vitest.live.config.ts`
- Fichiers : `src/**/*.live.test.ts`
- Par défaut : **activé** par `pnpm test:live` (définit `OPENCLAW_LIVE_TEST=1`)
- Portée :
  - « Ce fournisseur/modèle fonctionne-t-il réellement _aujourd’hui_ avec de vrais identifiants ? »
  - Détecter les changements de format des fournisseurs, les particularités d’appel d’outils, les problèmes d’authentification et le comportement de limitation de débit
- Attentes :
  - Par conception, pas stable en CI (vrais réseaux, vraies politiques fournisseur, quotas, pannes)
  - Coûte de l’argent / consomme des limites de débit
  - Préférez exécuter des sous-ensembles restreints plutôt que « tout »
- Les exécutions live chargent `~/.profile` pour récupérer les clés API manquantes.
- Par défaut, les exécutions live isolent toujours `HOME` et copient la configuration/le matériel d’authentification dans un home de test temporaire afin que les fixtures unitaires ne puissent pas modifier votre vrai `~/.openclaw`.
- Définissez `OPENCLAW_LIVE_USE_REAL_HOME=1` uniquement lorsque vous voulez intentionnellement que les tests live utilisent votre vrai répertoire home.
- `pnpm test:live` utilise désormais par défaut un mode plus silencieux : il conserve les sorties de progression `[live] ...`, mais supprime l’avis supplémentaire sur `~/.profile` et coupe les logs de bootstrap gateway / le bruit Bonjour. Définissez `OPENCLAW_LIVE_TEST_QUIET=0` si vous voulez récupérer les logs de démarrage complets.
- Rotation de clés API (spécifique fournisseur) : définissez `*_API_KEYS` au format virgule/point-virgule ou `*_API_KEY_1`, `*_API_KEY_2` (par exemple `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) ou un remplacement par live via `OPENCLAW_LIVE_*_KEY` ; les tests réessaient sur les réponses de limitation de débit.
- Sortie progression/heartbeat :
  - Les suites live émettent désormais les lignes de progression sur stderr afin que les appels fournisseur longs restent visiblement actifs même lorsque la capture console de Vitest est silencieuse.
  - `vitest.live.config.ts` désactive l’interception console de Vitest afin que les lignes de progression fournisseur/gateway soient diffusées immédiatement pendant les exécutions live.
  - Réglez les heartbeats modèle direct avec `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Réglez les heartbeats gateway/sonde avec `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Quelle suite dois-je exécuter ?

Utilisez ce tableau de décision :

- Modification de logique/tests : exécutez `pnpm test` (et `pnpm test:coverage` si vous avez beaucoup modifié)
- Modification du réseau gateway / protocole WS / appairage : ajoutez `pnpm test:e2e`
- Débogage « mon bot est indisponible » / pannes spécifiques fournisseur / appel d’outils : exécutez un `pnpm test:live` restreint

## Live : balayage des capacités de nœud Android

- Test : `src/gateway/android-node.capabilities.live.test.ts`
- Script : `pnpm android:test:integration`
- Objectif : invoquer **chaque commande actuellement annoncée** par un nœud Android connecté et vérifier le comportement contractuel des commandes.
- Portée :
  - Préconditionnement/configuration manuelle (la suite n’installe, n’exécute ni n’apparie l’application).
  - Validation commande par commande de `node.invoke` gateway pour le nœud Android sélectionné.
- Préconfiguration requise :
  - Application Android déjà connectée et appairée à la gateway.
  - Application maintenue au premier plan.
  - Permissions/consentement de capture accordés pour les capacités que vous attendez comme réussies.
- Remplacements de cible facultatifs :
  - `OPENCLAW_ANDROID_NODE_ID` ou `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Détails complets de configuration Android : [Application Android](/fr/platforms/android)

## Live : smoke modèle (clés de profil)

Les tests live sont divisés en deux couches afin de pouvoir isoler les défaillances :

- « Modèle direct » nous indique si le fournisseur/modèle peut répondre tout court avec la clé donnée.
- « Smoke gateway » nous indique si toute la chaîne gateway+agent fonctionne pour ce modèle (sessions, historique, outils, politique de sandbox, etc.).

### Couche 1 : complétion de modèle directe (sans gateway)

- Test : `src/agents/models.profiles.live.test.ts`
- Objectif :
  - Énumérer les modèles découverts
  - Utiliser `getApiKeyForModel` pour sélectionner les modèles pour lesquels vous avez des identifiants
  - Exécuter une petite complétion par modèle (et des régressions ciblées si nécessaire)
- Comment l’activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
- Définissez `OPENCLAW_LIVE_MODELS=modern` (ou `all`, alias de modern) pour exécuter réellement cette suite ; sinon elle est ignorée afin de garder `pnpm test:live` centré sur le smoke gateway
- Comment sélectionner des modèles :
  - `OPENCLAW_LIVE_MODELS=modern` pour exécuter la liste d’autorisation moderne (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` est un alias pour la liste d’autorisation moderne
  - ou `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (liste d’autorisation séparée par des virgules)
- Comment sélectionner des fournisseurs :
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (liste d’autorisation séparée par des virgules)
- D’où viennent les clés :
  - Par défaut : magasin de profils et remplacements env
  - Définissez `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer **uniquement** le magasin de profils
- Pourquoi cela existe :
  - Sépare « l’API fournisseur est cassée / la clé est invalide » de « la chaîne agent gateway est cassée »
  - Contient de petites régressions isolées (exemple : relecture reasoning OpenAI Responses/Codex Responses + flux d’appel d’outils)

### Couche 2 : smoke gateway + agent dev (ce que fait réellement "@openclaw")

- Test : `src/gateway/gateway-models.profiles.live.test.ts`
- Objectif :
  - Démarrer une gateway en processus
  - Créer/patcher une session `agent:dev:*` (remplacement de modèle à chaque exécution)
  - Itérer sur les modèles disposant de clés et vérifier :
    - réponse « significative » (sans outils)
    - qu’une vraie invocation d’outil fonctionne (sonde read)
    - sondes d’outils supplémentaires facultatives (sonde exec+read)
    - que les chemins de régression OpenAI (appel d’outil seul → suivi) continuent de fonctionner
- Détails des sondes (pour expliquer rapidement les échecs) :
  - Sonde `read` : le test écrit un fichier nonce dans l’espace de travail et demande à l’agent de le `read` puis de renvoyer la nonce.
  - Sonde `exec+read` : le test demande à l’agent d’écrire via `exec` une nonce dans un fichier temporaire, puis de la `read`.
  - Sonde image : le test joint un PNG généré (chat + code aléatoire) et attend que le modèle renvoie `cat <CODE>`.
  - Référence d’implémentation : `src/gateway/gateway-models.profiles.live.test.ts` et `src/gateway/live-image-probe.ts`.
- Comment l’activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
- Comment sélectionner des modèles :
  - Par défaut : liste d’autorisation moderne (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` est un alias pour la liste d’autorisation moderne
  - Ou définissez `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (ou une liste séparée par des virgules) pour restreindre
- Comment sélectionner des fournisseurs (éviter « tout OpenRouter ») :
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (liste d’autorisation séparée par des virgules)
- Les sondes d’outils + image sont toujours actives dans ce test live :
  - sonde `read` + sonde `exec+read` (stress sur les outils)
  - la sonde image s’exécute lorsque le modèle annonce la prise en charge des entrées image
  - Flux (haut niveau) :
    - Le test génère un petit PNG avec « CAT » + code aléatoire (`src/gateway/live-image-probe.ts`)
    - L’envoie via `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - La gateway analyse les pièces jointes en `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - L’agent intégré transmet un message utilisateur multimodal au modèle
    - Vérification : la réponse contient `cat` + le code (tolérance OCR : erreurs mineures autorisées)

Conseil : pour voir ce que vous pouvez tester sur votre machine (et les ids exacts `provider/model`), exécutez :

```bash
openclaw models list
openclaw models list --json
```

## Live : smoke backend CLI (Codex CLI ou autres CLI locales)

- Test : `src/gateway/gateway-cli-backend.live.test.ts`
- Objectif : valider la chaîne gateway + agent à l’aide d’un backend CLI local, sans toucher à votre config par défaut.
- Activer :
  - `pnpm test:live` (ou `OPENCLAW_LIVE_TEST=1` si vous invoquez Vitest directement)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- Valeurs par défaut :
  - Modèle : `codex-cli/gpt-5.4`
  - Commande : `codex`
  - Arguments : `["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]`
- Remplacements (facultatifs) :
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.4"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/codex"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["exec","--json","--color","never","--sandbox","read-only","--skip-git-repo-check"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` pour envoyer une vraie pièce jointe image (les chemins sont injectés dans le prompt).
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` pour transmettre les chemins de fichiers image comme arguments CLI au lieu de les injecter dans le prompt.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (ou `"list"`) pour contrôler la manière dont les arguments image sont transmis lorsque `IMAGE_ARG` est défini.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` pour envoyer un second tour et valider le flux de reprise.

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

Remarques :

- L’exécuteur Docker se trouve dans `scripts/test-live-cli-backend-docker.sh`.
- Il exécute le smoke live de backend CLI dans l’image Docker du dépôt en tant qu’utilisateur non root `node`.
- Pour `codex-cli`, il installe le package Linux `@openai/codex` dans un préfixe inscriptible mis en cache à `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (par défaut : `~/.cache/openclaw/docker-cli-tools`).

## Live : smoke de liaison ACP (`/acp spawn ... --bind here`)

- Test : `src/gateway/gateway-acp-bind.live.test.ts`
- Objectif : valider le véritable flux de liaison de conversation ACP avec un agent ACP live :
  - envoyer `/acp spawn <agent> --bind here`
  - lier en place une conversation synthétique de canal de messages
  - envoyer un suivi normal sur cette même conversation
  - vérifier que le suivi arrive dans la transcription de session ACP liée
- Activer :
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Valeurs par défaut :
  - Agents ACP dans Docker : `claude,codex`
  - Agent ACP pour `pnpm test:live ...` direct : `claude`
  - Canal synthétique : contexte de conversation type MP Slack
  - Backend ACP : `acpx`
- Remplacements :
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Remarques :
  - Cette voie utilise la surface gateway `chat.send` avec des champs de route d’origine synthétiques réservés à l’admin afin que les tests puissent attacher un contexte de canal de messages sans prétendre effectuer une livraison externe.
  - Lorsque `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` n’est pas défini, le test utilise le registre d’agents intégré du plugin `acpx` embarqué pour l’agent de harness ACP sélectionné.

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
```

Remarques Docker :

- L’exécuteur Docker se trouve dans `scripts/test-live-acp-bind-docker.sh`.
- Par défaut, il exécute le smoke de liaison ACP contre les deux agents CLI live pris en charge en séquence : `claude`, puis `codex`.
- Utilisez `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude` ou `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex` pour restreindre la matrice.
- Il charge `~/.profile`, prépare dans le conteneur le matériel d’authentification CLI correspondant, installe `acpx` dans un préfixe npm inscriptible, puis installe la CLI live demandée (`@anthropic-ai/claude-code` ou `@openai/codex`) si elle manque.
- Dans Docker, l’exécuteur définit `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` afin que acpx garde disponibles pour le harnais enfant CLI les variables d’environnement fournisseur issues du profil chargé.

### Recettes live recommandées

Des listes d’autorisation restreintes et explicites sont les plus rapides et les moins fragiles :

- Modèle unique, direct (sans gateway) :
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Modèle unique, smoke gateway :
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Appel d’outils sur plusieurs fournisseurs :
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Focus Google (clé API Gemini + Antigravity) :
  - Gemini (clé API) : `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth) : `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Remarques :

- `google/...` utilise l’API Gemini (clé API).
- `google-antigravity/...` utilise le pont OAuth Antigravity (point de terminaison d’agent de style Cloud Code Assist).
- `google-gemini-cli/...` utilise la CLI Gemini locale sur votre machine (authentification distincte + particularités d’outillage).
- API Gemini vs CLI Gemini :
  - API : OpenClaw appelle l’API Gemini hébergée par Google via HTTP (clé API / authentification de profil) ; c’est généralement ce que les utilisateurs entendent par « Gemini ».
  - CLI : OpenClaw exécute un binaire local `gemini` ; elle a sa propre authentification et peut se comporter différemment (streaming/prise en charge des outils/décalage de version).

## Live : matrice de modèles (ce que nous couvrons)

Il n’existe pas de « liste de modèles CI » fixe (live est opt-in), mais voici les modèles **recommandés** à couvrir régulièrement sur une machine de développement avec des clés.

### Ensemble smoke moderne (appel d’outils + image)

Il s’agit de l’exécution « modèles courants » que nous nous attendons à maintenir en fonctionnement :

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

Choisissez au moins un modèle par famille de fournisseurs :

- OpenAI : `openai/gpt-5.4` (ou `openai/gpt-5.4-mini`)
- Anthropic : `anthropic/claude-opus-4-6` (ou `anthropic/claude-sonnet-4-6`)
- Google : `google/gemini-3-flash-preview` (ou `google/gemini-3.1-pro-preview`)
- Z.AI (GLM) : `zai/glm-4.7`
- MiniMax : `minimax/MiniMax-M2.7`

Couverture supplémentaire facultative (utile mais non indispensable) :

- xAI : `xai/grok-4` (ou la dernière version disponible)
- Mistral : `mistral/`… (choisissez un modèle compatible « tools » que vous avez activé)
- Cerebras : `cerebras/`… (si vous y avez accès)
- LM Studio : `lmstudio/`… (local ; l’appel d’outils dépend du mode API)

### Vision : envoi d’image (pièce jointe → message multimodal)

Incluez au moins un modèle compatible image dans `OPENCLAW_LIVE_GATEWAY_MODELS` (Claude/Gemini/variantes OpenAI compatibles vision, etc.) pour exercer la sonde image.

### Agrégateurs / passerelles alternatives

Si vous avez des clés activées, nous prenons aussi en charge les tests via :

- OpenRouter : `openrouter/...` (des centaines de modèles ; utilisez `openclaw models scan` pour trouver des candidats compatibles outils+image)
- OpenCode : `opencode/...` pour Zen et `opencode-go/...` pour Go (authentification via `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Autres fournisseurs que vous pouvez inclure dans la matrice live (si vous avez les identifiants/la config) :

- Intégrés : `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `google-gemini-cli`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Via `models.providers` (points de terminaison personnalisés) : `minimax` (cloud/API), plus tout proxy compatible OpenAI/Anthropic (LM Studio, vLLM, LiteLLM, etc.)

Conseil : n’essayez pas de coder en dur « tous les modèles » dans la documentation. La liste faisant autorité est celle que renvoie `discoverModels(...)` sur votre machine + les clés disponibles.

## Identifiants (ne jamais commit)

Les tests live découvrent les identifiants de la même manière que la CLI. Conséquences pratiques :

- Si la CLI fonctionne, les tests live devraient trouver les mêmes clés.
- Si un test live indique « pas d’identifiants », déboguez-le de la même manière que `openclaw models list` / la sélection de modèle.

- Profils d’authentification par agent : `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (c’est ce que signifient les « clés de profil » dans les tests live)
- Config : `~/.openclaw/openclaw.json` (ou `OPENCLAW_CONFIG_PATH`)
- Répertoire d’état legacy : `~/.openclaw/credentials/` (copié dans le home live intermédiaire lorsqu’il est présent, mais ce n’est pas le magasin principal de clés de profil)
- Les exécutions live locales copient par défaut la config active, les fichiers `auth-profiles.json` par agent, le répertoire legacy `credentials/` et les répertoires d’authentification CLI externes pris en charge dans un home de test temporaire ; les remplacements de chemin `agents.*.workspace` / `agentDir` sont supprimés de cette config intermédiaire afin que les sondes n’utilisent pas votre véritable espace de travail hôte.

Si vous voulez vous appuyer sur des clés d’environnement (par exemple exportées dans votre `~/.profile`), exécutez les tests locaux après `source ~/.profile`, ou utilisez les exécuteurs Docker ci-dessous (ils peuvent monter `~/.profile` dans le conteneur).

## Deepgram live (transcription audio)

- Test : `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Activer : `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- Test : `src/agents/byteplus.live.test.ts`
- Activer : `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Remplacement de modèle facultatif : `BYTEPLUS_CODING_MODEL=ark-code-latest`

## Workflow média ComfyUI live

- Test : `extensions/comfy/comfy.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Portée :
  - Exerce les chemins image, vidéo et `music_generate` comfy intégrés
  - Ignore chaque capacité sauf si `models.providers.comfy.<capability>` est configuré
  - Utile après une modification de la soumission de workflow comfy, du polling, des téléchargements ou de l’enregistrement du plugin

## Génération d’images live

- Test : `src/image-generation/runtime.live.test.ts`
- Commande : `pnpm test:live src/image-generation/runtime.live.test.ts`
- Harness : `pnpm test:live:media image`
- Portée :
  - Énumère tous les plugins fournisseurs de génération d’images enregistrés
  - Charge les variables d’environnement fournisseur manquantes depuis votre shell de connexion (`~/.profile`) avant la sonde
  - Utilise par défaut les clés API live/env avant les profils d’authentification stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les véritables identifiants du shell
  - Ignore les fournisseurs sans auth/profil/modèle utilisable
  - Exécute les variantes standard de génération d’images via la capacité runtime partagée :
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
- Comportement d’authentification facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l’authentification du magasin de profils et ignorer les remplacements env seuls

## Génération musicale live

- Test : `extensions/music-generation-providers.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Harness : `pnpm test:live:media music`
- Portée :
  - Exerce le chemin partagé des fournisseurs intégrés de génération musicale
  - Couvre actuellement Google et MiniMax
  - Charge les variables d’environnement fournisseur depuis votre shell de connexion (`~/.profile`) avant la sonde
  - Utilise par défaut les clés API live/env avant les profils d’authentification stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les véritables identifiants du shell
  - Ignore les fournisseurs sans auth/profil/modèle utilisable
  - Exécute les deux modes runtime déclarés lorsqu’ils sont disponibles :
    - `generate` avec une entrée composée uniquement d’un prompt
    - `edit` lorsque le fournisseur déclare `capabilities.edit.enabled`
  - Couverture actuelle de la voie partagée :
    - `google` : `generate`, `edit`
    - `minimax` : `generate`
    - `comfy` : fichier live Comfy séparé, pas ce balayage partagé
- Restriction facultative :
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`
- Comportement d’authentification facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l’authentification du magasin de profils et ignorer les remplacements env seuls

## Génération vidéo live

- Test : `extensions/video-generation-providers.live.test.ts`
- Activer : `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- Harness : `pnpm test:live:media video`
- Portée :
  - Exerce le chemin partagé des fournisseurs intégrés de génération vidéo
  - Charge les variables d’environnement fournisseur depuis votre shell de connexion (`~/.profile`) avant la sonde
  - Utilise par défaut les clés API live/env avant les profils d’authentification stockés, afin que des clés de test obsolètes dans `auth-profiles.json` ne masquent pas les véritables identifiants du shell
  - Ignore les fournisseurs sans auth/profil/modèle utilisable
  - Exécute les deux modes runtime déclarés lorsqu’ils sont disponibles :
    - `generate` avec une entrée composée uniquement d’un prompt
    - `imageToVideo` lorsque le fournisseur déclare `capabilities.imageToVideo.enabled` et que le fournisseur/modèle sélectionné accepte une entrée image locale adossée à un buffer dans le balayage partagé
    - `videoToVideo` lorsque le fournisseur déclare `capabilities.videoToVideo.enabled` et que le fournisseur/modèle sélectionné accepte une entrée vidéo locale adossée à un buffer dans le balayage partagé
  - Fournisseurs `imageToVideo` actuellement déclarés mais ignorés dans le balayage partagé :
    - `vydra` parce que `veo3` intégré est texte uniquement et que `kling` intégré nécessite une URL d’image distante
  - Couverture spécifique fournisseur Vydra :
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - ce fichier exécute `veo3` texte-vers-vidéo ainsi qu’une voie `kling` utilisant par défaut une fixture d’URL d’image distante
  - Couverture live actuelle `videoToVideo` :
    - `runway` uniquement lorsque le modèle sélectionné est `runway/gen4_aleph`
  - Fournisseurs `videoToVideo` actuellement déclarés mais ignorés dans le balayage partagé :
    - `alibaba`, `qwen`, `xai` parce que ces chemins exigent actuellement des URL de référence distantes `http(s)` / MP4
    - `google` parce que la voie partagée Gemini/Veo actuelle utilise une entrée locale adossée à un buffer et que ce chemin n’est pas accepté dans le balayage partagé
    - `openai` parce que la voie partagée actuelle ne garantit pas l’accès spécifique à l’organisation pour l’inpainting/remix vidéo
- Restriction facultative :
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
- Comportement d’authentification facultatif :
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour forcer l’authentification du magasin de profils et ignorer les remplacements env seuls

## Harness média live

- Commande : `pnpm test:live:media`
- Objectif :
  - Exécute les suites live partagées image, musique et vidéo via un point d’entrée natif du dépôt
  - Charge automatiquement les variables d’environnement fournisseur manquantes depuis `~/.profile`
  - Restreint automatiquement chaque suite, par défaut, aux fournisseurs disposant actuellement d’une auth utilisable
  - Réutilise `scripts/test-live.mjs`, afin que le comportement heartbeat et mode silencieux reste cohérent
- Exemples :
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## Exécuteurs Docker (vérifications facultatives « fonctionne sous Linux »)

Ces exécuteurs Docker se divisent en deux catégories :

- Exécuteurs live-model : `test:docker:live-models` et `test:docker:live-gateway` exécutent uniquement leur fichier live à clés de profil correspondant dans l’image Docker du dépôt (`src/agents/models.profiles.live.test.ts` et `src/gateway/gateway-models.profiles.live.test.ts`), en montant votre répertoire de configuration local et votre espace de travail (et en chargeant `~/.profile` s’il est monté). Les points d’entrée locaux correspondants sont `test:live:models-profiles` et `test:live:gateway-profiles`.
- Les exécuteurs Docker live utilisent par défaut un plafond smoke plus petit afin qu’un balayage Docker complet reste praticable :
  `test:docker:live-models` utilise par défaut `OPENCLAW_LIVE_MAX_MODELS=12`, et
  `test:docker:live-gateway` utilise par défaut `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`, et
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Remplacez ces variables d’environnement lorsque vous
  voulez explicitement le balayage exhaustif plus large.
- `test:docker:all` construit l’image Docker live une seule fois via `test:docker:live-build`, puis la réutilise pour les deux voies Docker live.
- Les exécuteurs smoke de conteneur : `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` et `test:docker:plugins` démarrent un ou plusieurs vrais conteneurs et vérifient des chemins d’intégration de plus haut niveau.

Les exécuteurs Docker live-model montent aussi seulement les homes d’authentification CLI nécessaires (ou tous ceux pris en charge lorsque l’exécution n’est pas restreinte), puis les copient dans le home du conteneur avant l’exécution afin que l’OAuth CLI externe puisse rafraîchir les jetons sans modifier le magasin d’authentification de l’hôte :

- Modèles directs : `pnpm test:docker:live-models` (script : `scripts/test-live-models-docker.sh`)
- Smoke de liaison ACP : `pnpm test:docker:live-acp-bind` (script : `scripts/test-live-acp-bind-docker.sh`)
- Smoke de backend CLI : `pnpm test:docker:live-cli-backend` (script : `scripts/test-live-cli-backend-docker.sh`)
- Gateway + agent dev : `pnpm test:docker:live-gateway` (script : `scripts/test-live-gateway-models-docker.sh`)
- Smoke live Open WebUI : `pnpm test:docker:openwebui` (script : `scripts/e2e/openwebui-docker.sh`)
- Assistant d’onboarding (TTY, scaffolding complet) : `pnpm test:docker:onboard` (script : `scripts/e2e/onboard-docker.sh`)
- Réseau gateway (deux conteneurs, auth WS + santé) : `pnpm test:docker:gateway-network` (script : `scripts/e2e/gateway-network-docker.sh`)
- Pont de canal MCP (Gateway initialisée + pont stdio + smoke de trame de notification Claude brute) : `pnpm test:docker:mcp-channels` (script : `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (smoke d’installation + alias `/plugin` + sémantique de redémarrage du bundle Claude) : `pnpm test:docker:plugins` (script : `scripts/e2e/plugins-docker.sh`)

Les exécuteurs Docker live-model montent également l’extraction courante en lecture seule et
la préparent dans un répertoire de travail temporaire à l’intérieur du conteneur. Cela garde l’image runtime
légère tout en exécutant Vitest sur votre source/config locale exacte.
L’étape de préparation ignore les gros caches locaux et les sorties de build d’application comme
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__`, ainsi que les répertoires `.build` locaux aux applications ou de sortie Gradle, afin que les exécutions Docker live ne passent pas des minutes à copier
des artefacts spécifiques à la machine.
Ils définissent aussi `OPENCLAW_SKIP_CHANNELS=1` afin que les sondes live gateway ne démarrent pas
de vrais workers de canaux Telegram/Discord/etc. à l’intérieur du conteneur.
`test:docker:live-models` exécute toujours `pnpm test:live`, donc transmettez aussi
`OPENCLAW_LIVE_GATEWAY_*` lorsque vous devez restreindre ou exclure la couverture
live gateway de cette voie Docker.
`test:docker:openwebui` est un smoke de compatibilité de plus haut niveau : il démarre une
gateway OpenClaw en conteneur avec les points de terminaison HTTP compatibles OpenAI activés,
démarre un conteneur Open WebUI épinglé contre cette gateway, se connecte via
Open WebUI, vérifie que `/api/models` expose `openclaw/default`, puis envoie une
vraie requête de chat via le proxy `/api/chat/completions` d’Open WebUI.
La première exécution peut être sensiblement plus lente, car Docker peut devoir télécharger l’image
Open WebUI et Open WebUI peut devoir terminer sa propre initialisation à froid.
Cette voie attend une clé de modèle live utilisable, et `OPENCLAW_PROFILE_FILE`
(`~/.profile` par défaut) est le principal moyen de la fournir dans les exécutions Dockerisées.
Les exécutions réussies affichent une petite charge utile JSON comme `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` est volontairement déterministe et ne nécessite pas de
vrai compte Telegram, Discord ou iMessage. Il démarre un conteneur Gateway
initialisé, lance un second conteneur qui démarre `openclaw mcp serve`, puis
vérifie la découverte de conversation routée, les lectures de transcription, les métadonnées
de pièces jointes, le comportement de la file d’événements live, le routage d’envoi sortant, et les notifications de canal +
permissions de style Claude via le véritable pont MCP stdio. La vérification des notifications
inspecte directement les trames MCP stdio brutes afin que le smoke valide ce que le pont
émet réellement, et pas seulement ce qu’un SDK client particulier choisit d’exposer.

Smoke manuel ACP de fil en langage naturel (hors CI) :

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Conservez ce script pour les flux de régression/débogage. Il pourrait redevenir nécessaire pour la validation du routage de fil ACP, donc ne le supprimez pas.

Variables d’environnement utiles :

- `OPENCLAW_CONFIG_DIR=...` (par défaut : `~/.openclaw`) monté sur `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (par défaut : `~/.openclaw/workspace`) monté sur `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (par défaut : `~/.profile`) monté sur `/home/node/.profile` et chargé avant l’exécution des tests
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (par défaut : `~/.cache/openclaw/docker-cli-tools`) monté sur `/home/node/.npm-global` pour les installations CLI mises en cache dans Docker
- Les répertoires/fichiers d’authentification CLI externes sous `$HOME` sont montés en lecture seule sous `/host-auth...`, puis copiés dans `/home/node/...` avant le démarrage des tests
  - Répertoires par défaut : `.minimax`
  - Fichiers par défaut : `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Les exécutions restreintes par fournisseur montent uniquement les répertoires/fichiers nécessaires déduits de `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Remplacement manuel avec `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none`, ou une liste séparée par des virgules telle que `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` pour restreindre l’exécution
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` pour filtrer les fournisseurs dans le conteneur
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` pour garantir que les identifiants viennent du magasin de profils (et non de l’environnement)
- `OPENCLAW_OPENWEBUI_MODEL=...` pour choisir le modèle exposé par la gateway pour le smoke Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` pour remplacer le prompt de vérification de nonce utilisé par le smoke Open WebUI
- `OPENWEBUI_IMAGE=...` pour remplacer le tag d’image Open WebUI épinglé

## Vérification rapide de la documentation

Exécutez les vérifications de docs après des modifications de documentation : `pnpm check:docs`.
Exécutez la validation complète des ancres Mintlify lorsque vous avez aussi besoin des vérifications d’en-têtes intra-page : `pnpm docs:check-links:anchors`.

## Régression hors ligne (compatible CI)

Il s’agit de régressions de « vrai pipeline » sans vrais fournisseurs :

- Appel d’outils gateway (mock OpenAI, vraie boucle gateway + agent) : `src/gateway/gateway.test.ts` (cas : "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Assistant gateway (WS `wizard.start`/`wizard.next`, écriture de config + auth imposée) : `src/gateway/gateway.test.ts` (cas : "runs wizard over ws and writes auth token config")

## Évaluations de fiabilité d’agent (compétences)

Nous avons déjà quelques tests compatibles CI qui se comportent comme des « évaluations de fiabilité d’agent » :

- Appel d’outils simulé via la vraie boucle gateway + agent (`src/gateway/gateway.test.ts`).
- Flux d’assistant end-to-end qui valident le câblage de session et les effets de configuration (`src/gateway/gateway.test.ts`).

Ce qui manque encore pour les compétences (voir [Skills](/fr/tools/skills)) :

- **Prise de décision :** lorsque les compétences sont listées dans le prompt, l’agent choisit-il la bonne compétence (ou évite-t-il celles qui ne sont pas pertinentes) ?
- **Conformité :** l’agent lit-il `SKILL.md` avant utilisation et suit-il les étapes/arguments requis ?
- **Contrats de workflow :** scénarios multi-tours qui vérifient l’ordre des outils, le report de l’historique de session et les limites du sandbox.

Les évaluations futures devraient d’abord rester déterministes :

- Un exécuteur de scénarios utilisant des fournisseurs simulés pour vérifier les appels d’outils + leur ordre, les lectures de fichiers de compétences et le câblage de session.
- Une petite suite de scénarios centrés sur les compétences (utiliser vs éviter, contrôle d’accès, injection de prompt).
- Des évaluations live facultatives (opt-in, contrôlées par env) seulement après la mise en place de la suite compatible CI.

## Tests de contrat (forme des plugins et canaux)

Les tests de contrat vérifient que chaque plugin et canal enregistré respecte son
contrat d’interface. Ils itèrent sur tous les plugins découverts et exécutent une suite d’assertions de
forme et de comportement. La voie unitaire par défaut `pnpm test` ignore volontairement ces fichiers de couture et de smoke partagés ; exécutez explicitement
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
- **outbound-payload** - Structure de la charge utile de message
- **inbound** - Gestion des messages entrants
- **actions** - Gestionnaires d’actions de canal
- **threading** - Gestion des ids de fil
- **directory** - API d’annuaire/liste
- **group-policy** - Application de la politique de groupe

### Contrats d’état des fournisseurs

Situés dans `src/plugins/contracts/*.contract.test.ts`.

- **status** - Sondes d’état de canal
- **registry** - Forme du registre de plugins

### Contrats de fournisseur

Situés dans `src/plugins/contracts/*.contract.test.ts` :

- **auth** - Contrat du flux d’authentification
- **auth-choice** - Choix/sélection d’authentification
- **catalog** - API du catalogue de modèles
- **discovery** - Découverte des plugins
- **loader** - Chargement des plugins
- **runtime** - Runtime du fournisseur
- **shape** - Forme/interface du plugin
- **wizard** - Assistant de configuration

### Quand les exécuter

- Après modification des exports ou sous-chemins de `plugin-sdk`
- Après ajout ou modification d’un plugin de canal ou de fournisseur
- Après refactorisation de l’enregistrement ou de la découverte des plugins

Les tests de contrat s’exécutent en CI et ne nécessitent pas de vraies clés API.

## Ajouter des régressions (recommandations)

Lorsque vous corrigez un problème de fournisseur/modèle découvert en live :

- Ajoutez si possible une régression compatible CI (simulation/stub de fournisseur, ou capture de la transformation exacte de forme de requête)
- Si le problème est intrinsèquement live seulement (limitations de débit, politiques d’authentification), gardez le test live étroit et opt-in via des variables d’environnement
- Préférez cibler la plus petite couche qui détecte le bug :
  - bug de conversion/relecture de requête fournisseur → test de modèles directs
  - bug de chaîne gateway session/historique/outils → smoke live gateway ou test mock gateway compatible CI
- Garde-fou sur la traversée SecretRef :
  - `src/secrets/exec-secret-ref-id-parity.test.ts` dérive une cible échantillon par classe SecretRef à partir des métadonnées du registre (`listSecretTargetRegistryEntries()`), puis vérifie que les ids exec de segment de traversée sont rejetés.
  - Si vous ajoutez une nouvelle famille de cibles SecretRef `includeInPlan` dans `src/secrets/target-registry-data.ts`, mettez à jour `classifyTargetClass` dans ce test. Le test échoue volontairement sur les ids de cible non classés afin qu’aucune nouvelle classe ne puisse être ignorée silencieusement.
