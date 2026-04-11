---
read_when:
    - Étendre qa-lab ou qa-channel
    - Ajout de scénarios QA adossés au dépôt
    - Création d’une automatisation QA plus réaliste autour du tableau de bord Gateway
summary: Forme de l’automatisation QA privée pour qa-lab, qa-channel, les scénarios initialisés et les rapports de protocole
title: Automatisation QA E2E
x-i18n:
    generated_at: "2026-04-11T02:44:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 5427b505e26bfd542e984e3920c3f7cb825473959195ba9737eff5da944c60d0
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatisation QA E2E

La pile QA privée est conçue pour exercer OpenClaw d’une manière plus réaliste,
façonnée par les canaux, qu’un simple test unitaire ne peut le faire.

Éléments actuels :

- `extensions/qa-channel` : canal de messages synthétique avec surfaces de MP, canal, fil,
  réaction, modification et suppression.
- `extensions/qa-lab` : interface utilisateur de débogage et bus QA pour observer la transcription,
  injecter des messages entrants et exporter un rapport Markdown.
- `qa/` : ressources d’initialisation adossées au dépôt pour la tâche de lancement et les scénarios QA
  de référence.

Le flux opérateur QA actuel est un site QA à deux volets :

- Gauche : tableau de bord Gateway (Control UI) avec l’agent.
- Droite : QA Lab, affichant la transcription de style Slack et le plan de scénario.

Exécutez-le avec :

```bash
pnpm qa:lab:up
```

Cela construit le site QA, démarre la voie gateway adossée à Docker et expose la
page QA Lab où un opérateur ou une boucle d’automatisation peut donner à l’agent une
mission QA, observer le comportement réel du canal et consigner ce qui a fonctionné, échoué ou
est resté bloqué.

Pour une itération plus rapide sur l’interface QA Lab sans reconstruire l’image Docker à chaque fois,
démarrez la pile avec un bundle QA Lab monté par liaison :

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` conserve les services Docker sur une image préconstruite et monte par liaison
`extensions/qa-lab/web/dist` dans le conteneur `qa-lab`. `qa:lab:watch`
reconstruit ce bundle lors des modifications, et le navigateur se recharge automatiquement lorsque le hachage
des ressources QA Lab change.

Pour une voie de smoke test Matrix avec transport réel, exécutez :

```bash
pnpm openclaw qa matrix
```

Cette voie provisionne un homeserver Tuwunel jetable dans Docker, enregistre
des utilisateurs temporaires driver, SUT et observateur, crée un salon privé, puis exécute
le plugin Matrix réel dans un processus enfant gateway QA. La voie avec transport actif conserve la
configuration enfant limitée au transport testé, de sorte que Matrix s’exécute sans
`qa-channel` dans la configuration enfant.

Pour une voie de smoke test Telegram avec transport réel, exécutez :

```bash
pnpm openclaw qa telegram
```

Cette voie cible un vrai groupe privé Telegram au lieu de provisionner un
serveur jetable. Elle nécessite `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` et
`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`, ainsi que deux bots distincts dans le même
groupe privé. Le bot SUT doit avoir un nom d’utilisateur Telegram, et l’observation
bot à bot fonctionne le mieux lorsque les deux bots ont le mode de communication bot à bot
activé dans `@BotFather`.

Les voies de transport actives partagent désormais un contrat plus petit au lieu que chacune
invente sa propre forme de liste de scénarios :

`qa-channel` reste la suite large de comportements produit synthétiques et ne fait pas partie
de la matrice de couverture des transports actifs.

| Voie     | Canary | Filtrage des mentions | Blocage par liste d’autorisation | Réponse de niveau supérieur | Reprise après redémarrage | Suivi de fil | Isolation des fils | Observation des réactions | Commande help |
| -------- | ------ | --------------------- | -------------------------------- | --------------------------- | ------------------------- | ------------ | ------------------ | ------------------------- | ------------- |
| Matrix   | x      | x                     | x                                | x                           | x                         | x            | x                  | x                         |               |
| Telegram | x      |                       |                                  |                             |                           |              |                    |                           | x             |

Cela permet de conserver `qa-channel` comme suite large de comportements produit, tandis que Matrix,
Telegram et les futurs transports actifs partagent une liste de vérification explicite
du contrat de transport.

Pour une voie VM Linux jetable sans introduire Docker dans le parcours QA, exécutez :

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

Cela démarre un invité Multipass neuf, installe les dépendances, construit OpenClaw
dans l’invité, exécute `qa suite`, puis recopie le rapport QA normal et le
résumé dans `.artifacts/qa-e2e/...` sur l’hôte.
Cela réutilise le même comportement de sélection de scénarios que `qa suite` sur l’hôte.
Les exécutions de suite sur l’hôte et dans Multipass exécutent plusieurs scénarios sélectionnés en parallèle
avec des workers gateway isolés par défaut, jusqu’à 64 workers ou au nombre
de scénarios sélectionnés. Utilisez `--concurrency <count>` pour ajuster le nombre de workers, ou
`--concurrency 1` pour une exécution en série.
Les exécutions actives transmettent les entrées d’auth QA prises en charge et pratiques pour
l’invité : clés de fournisseur basées sur l’environnement, chemin de configuration du fournisseur QA actif, et
`CODEX_HOME` lorsqu’il est présent. Conservez `--output-dir` sous la racine du dépôt afin que l’invité
puisse écrire en retour via l’espace de travail monté.

## Ressources d’initialisation adossées au dépôt

Les ressources d’initialisation se trouvent dans `qa/` :

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Elles sont volontairement dans git afin que le plan QA soit visible à la fois pour les humains et pour
l’agent. La liste de référence doit rester suffisamment large pour couvrir :

- chat en MP et en canal
- comportement des fils
- cycle de vie des actions sur les messages
- rappels cron
- rappel de mémoire
- changement de modèle
- transmission à un sous-agent
- lecture du dépôt et de la documentation
- une petite tâche de build comme Lobster Invaders

## Rapports

`qa-lab` exporte un rapport de protocole Markdown à partir de la chronologie observée du bus.
Le rapport doit répondre à :

- Ce qui a fonctionné
- Ce qui a échoué
- Ce qui est resté bloqué
- Quels scénarios de suivi valent la peine d’être ajoutés

Pour les vérifications de caractère et de style, exécutez le même scénario sur plusieurs références de modèles actifs
et rédigez un rapport Markdown évalué :

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.4,thinking=xhigh \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-6,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.4,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-6,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

La commande exécute des processus enfants gateway QA locaux, pas Docker. Les scénarios d’évaluation de caractère
doivent définir la persona via `SOUL.md`, puis exécuter des tours utilisateur ordinaires
comme le chat, l’aide sur l’espace de travail et de petites tâches sur des fichiers. Le modèle candidat
ne doit pas être informé qu’il est en cours d’évaluation. La commande conserve chaque transcription complète,
enregistre des statistiques d’exécution de base, puis demande aux modèles juges en mode rapide avec
un raisonnement `xhigh` de classer les exécutions selon le naturel, l’ambiance et l’humour.
Utilisez `--blind-judge-models` lors de la comparaison de fournisseurs : l’invite du juge reçoit toujours
chaque transcription et statut d’exécution, mais les références candidates sont remplacées par des
étiquettes neutres telles que `candidate-01` ; le rapport remappe les classements vers les vraies références après
analyse.
Les exécutions candidates utilisent par défaut le niveau de réflexion `high`, avec `xhigh` pour les modèles OpenAI qui
le prennent en charge. Remplacez un candidat spécifique en ligne avec
`--model provider/model,thinking=<level>`. `--thinking <level>` définit toujours un
secours global, et l’ancienne forme `--model-thinking <provider/model=level>` est
conservée pour compatibilité.
Les références candidates OpenAI utilisent par défaut le mode rapide afin que le traitement prioritaire soit utilisé là
où le fournisseur le prend en charge. Ajoutez `,fast`, `,no-fast` ou `,fast=false` en ligne lorsqu’un
candidat ou juge unique a besoin d’un remplacement. Passez `--fast` uniquement si vous souhaitez
forcer le mode rapide pour chaque modèle candidat. Les durées des candidats et des juges sont
enregistrées dans le rapport pour l’analyse de benchmark, mais les invites des juges indiquent explicitement
de ne pas classer selon la vitesse.
Les exécutions des modèles candidats et juges utilisent toutes deux par défaut une concurrence de 16. Réduisez
`--concurrency` ou `--judge-concurrency` lorsque les limites du fournisseur ou la pression sur la gateway locale
rendent une exécution trop bruyante.
Lorsqu’aucun `--model` candidat n’est fourni, l’évaluation de caractère utilise par défaut
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` et
`google/gemini-3.1-pro-preview` lorsqu’aucun `--model` n’est fourni.
Lorsqu’aucun `--judge-model` n’est fourni, les juges utilisent par défaut
`openai/gpt-5.4,thinking=xhigh,fast` et
`anthropic/claude-opus-4-6,thinking=high`.

## Documentation associée

- [Testing](/fr/help/testing)
- [QA Channel](/fr/channels/qa-channel)
- [Dashboard](/web/dashboard)
