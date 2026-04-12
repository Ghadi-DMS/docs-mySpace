---
read_when:
    - Extension de qa-lab ou qa-channel
    - Ajout de scénarios QA adossés au dépôt
    - Création d’une automatisation QA plus réaliste autour du tableau de bord Gateway
summary: Forme de l’automatisation QA privée pour qa-lab, qa-channel, les scénarios préconfigurés et les rapports de protocole
title: Automatisation QA E2E
x-i18n:
    generated_at: "2026-04-12T23:28:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: b9fe27dc049823d5e3eb7ae1eac6aad21ed9e917425611fb1dbcb28ab9210d5e
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatisation QA E2E

La pile QA privée a pour objectif de tester OpenClaw d’une manière plus réaliste,
calquée sur les canaux, qu’un simple test unitaire ne peut le faire.

Éléments actuels :

- `extensions/qa-channel` : canal de messages synthétique avec surfaces de MP, de canal, de fil,
  de réaction, de modification et de suppression.
- `extensions/qa-lab` : interface de débogage et bus QA pour observer la transcription,
  injecter des messages entrants et exporter un rapport Markdown.
- `qa/` : ressources d’amorçage adossées au dépôt pour la tâche de démarrage et les scénarios QA
  de base.

Le flux opérateur QA actuel repose sur un site QA à deux volets :

- Gauche : tableau de bord Gateway (Control UI) avec l’agent.
- Droite : QA Lab, affichant la transcription de type Slack et le plan de scénario.

Exécutez-le avec :

```bash
pnpm qa:lab:up
```

Cela construit le site QA, démarre la voie Gateway adossée à Docker et expose la
page QA Lab où un opérateur ou une boucle d’automatisation peut donner à l’agent une mission QA,
observer le comportement réel du canal et consigner ce qui a fonctionné, échoué ou
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
reconstruit ce bundle à chaque modification, et le navigateur se recharge automatiquement lorsque le hachage des ressources de QA Lab change.

Pour une voie de test rapide Matrix avec transport réel, exécutez :

```bash
pnpm openclaw qa matrix
```

Cette voie provisionne un homeserver Tuwunel jetable dans Docker, enregistre
des utilisateurs temporaires pilote, SUT et observateur, crée un salon privé, puis exécute
le Plugin Matrix réel dans un processus enfant QA gateway. La voie de transport en direct conserve
la configuration enfant limitée au transport testé, de sorte que Matrix s’exécute sans
`qa-channel` dans la configuration enfant.

Pour une voie de test rapide Telegram avec transport réel, exécutez :

```bash
pnpm openclaw qa telegram
```

Cette voie cible un véritable groupe Telegram privé au lieu de provisionner un
serveur jetable. Elle nécessite `OPENCLAW_QA_TELEGRAM_GROUP_ID`,
`OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` et
`OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`, ainsi que deux bots distincts dans le même
groupe privé. Le bot SUT doit avoir un nom d’utilisateur Telegram, et l’observation
bot à bot fonctionne au mieux lorsque les deux bots ont le mode Bot-to-Bot Communication activé
dans `@BotFather`.

Les voies de transport en direct partagent désormais un contrat plus restreint au lieu que chacune
invente sa propre forme de liste de scénarios :

`qa-channel` reste la suite étendue de comportements produit synthétiques et ne fait pas partie
de la matrice de couverture des transports en direct.

| Voie     | Canary | Filtrage des mentions | Blocage par liste d’autorisation | Réponse de premier niveau | Reprise après redémarrage | Suivi dans le fil | Isolation du fil | Observation des réactions | Commande d’aide |
| -------- | ------ | --------------------- | -------------------------------- | ------------------------- | ------------------------- | ----------------- | ---------------- | ------------------------- | --------------- |
| Matrix   | x      | x                     | x                                | x                         | x                         | x                 | x                | x                         |                 |
| Telegram | x      |                       |                                  |                           |                           |                   |                  |                           | x               |

Cela permet à `qa-channel` de rester la suite étendue de comportements produit, tandis que Matrix,
Telegram et les futurs transports en direct partagent une liste de vérification explicite du contrat de transport.

Pour une voie VM Linux jetable sans introduire Docker dans le parcours QA, exécutez :

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

Cela démarre un nouvel invité Multipass, installe les dépendances, construit OpenClaw
dans l’invité, exécute `qa suite`, puis copie le rapport QA normal et le résumé
dans `.artifacts/qa-e2e/...` sur l’hôte.
Cela réutilise le même comportement de sélection de scénarios que `qa suite` sur l’hôte.
Les exécutions de suite sur l’hôte et sur Multipass exécutent plusieurs scénarios sélectionnés en parallèle
avec des workers gateway isolés par défaut, jusqu’à 64 workers ou au nombre de scénarios sélectionnés. Utilisez `--concurrency <count>` pour ajuster le nombre de workers, ou
`--concurrency 1` pour une exécution en série.
Les exécutions en direct transmettent les entrées d’auth QA prises en charge qui sont pratiques pour
l’invité : clés de fournisseur basées sur l’environnement, chemin de configuration du fournisseur QA live, et
`CODEX_HOME` lorsqu’il est présent. Gardez `--output-dir` sous la racine du dépôt afin que l’invité
puisse réécrire via l’espace de travail monté.

## Ressources d’amorçage adossées au dépôt

Les ressources d’amorçage se trouvent dans `qa/` :

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Celles-ci sont volontairement conservées dans git afin que le plan QA soit visible à la fois pour les humains et pour l’agent.

`qa-lab` doit rester un exécuteur Markdown générique. Chaque fichier Markdown de scénario est
la source de vérité pour une exécution de test et doit définir :

- les métadonnées du scénario
- les références de documentation et de code
- les exigences optionnelles de Plugin
- le correctif optionnel de configuration Gateway
- le `qa-flow` exécutable

La liste de base doit rester assez large pour couvrir :

- les MP et la discussion en canal
- le comportement des fils
- le cycle de vie des actions sur les messages
- les rappels Cron
- le rappel de mémoire
- le changement de modèle
- le transfert à un sous-agent
- la lecture du dépôt et de la documentation
- une petite tâche de build comme Lobster Invaders

## Adaptateurs de transport

`qa-lab` gère une jonction de transport générique pour les scénarios QA Markdown.
`qa-channel` est le premier adaptateur sur cette jonction, mais l’objectif de conception est plus large :
les futurs canaux réels ou synthétiques doivent se brancher sur le même exécuteur de suite
au lieu d’ajouter un exécuteur QA spécifique à un transport.

Au niveau de l’architecture, la répartition est la suivante :

- `qa-lab` gère l’exécution générique des scénarios, la concurrence des workers, l’écriture des artefacts et le reporting.
- l’adaptateur de transport gère la configuration Gateway, l’état de préparation, l’observation entrante et sortante, les actions de transport et l’état de transport normalisé.
- les fichiers de scénario Markdown sous `qa/scenarios/` définissent l’exécution de test ; `qa-lab` fournit la surface d’exécution réutilisable qui les exécute.

Les consignes d’adoption destinées aux mainteneurs pour les nouveaux adaptateurs de canal se trouvent dans
[Testing](/fr/help/testing#adding-a-channel-to-qa).

## Rapports

`qa-lab` exporte un rapport de protocole Markdown à partir de la chronologie observée du bus.
Le rapport doit répondre à :

- Ce qui a fonctionné
- Ce qui a échoué
- Ce qui est resté bloqué
- Quels scénarios de suivi méritent d’être ajoutés

Pour les vérifications de caractère et de style, exécutez le même scénario sur plusieurs références de modèles en direct
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

La commande exécute des processus enfants QA gateway locaux, et non Docker. Les scénarios d’évaluation du caractère
doivent définir la personnalité via `SOUL.md`, puis exécuter des tours utilisateur ordinaires
comme la discussion, l’aide sur l’espace de travail et de petites tâches sur les fichiers. Le modèle candidat
ne doit pas être informé qu’il est en cours d’évaluation. La commande conserve chaque transcription
complète, enregistre les statistiques de base de l’exécution, puis demande aux modèles juges en mode rapide avec
un raisonnement `xhigh` de classer les exécutions selon le naturel, l’ambiance et l’humour.
Utilisez `--blind-judge-models` lors de la comparaison de fournisseurs : l’invite du juge reçoit toujours
chaque transcription et l’état d’exécution, mais les références candidates sont remplacées par des
étiquettes neutres telles que `candidate-01` ; le rapport remappe les classements vers les références réelles après
l’analyse.
Les exécutions candidates utilisent par défaut le niveau de réflexion `high`, avec `xhigh` pour les modèles OpenAI qui
le prennent en charge. Remplacez un candidat spécifique en ligne avec
`--model provider/model,thinking=<level>`. `--thinking <level>` définit toujours une valeur de repli
globale, et l’ancien format `--model-thinking <provider/model=level>` est
conservé pour compatibilité.
Les références candidates OpenAI utilisent par défaut le mode rapide afin d’employer le traitement prioritaire là où
le fournisseur le prend en charge. Ajoutez `,fast`, `,no-fast` ou `,fast=false` en ligne lorsqu’un
candidat ou juge unique a besoin d’une exception. Passez `--fast` uniquement si vous souhaitez
forcer le mode rapide pour chaque modèle candidat. Les durées des candidats et des juges sont
enregistrées dans le rapport pour l’analyse comparative, mais les invites de jugement indiquent explicitement
de ne pas classer selon la vitesse.
Les exécutions des modèles candidats et des modèles juges utilisent toutes deux par défaut une concurrence de 16. Réduisez
`--concurrency` ou `--judge-concurrency` lorsque les limites du fournisseur ou la pression sur la gateway locale
rendent une exécution trop bruyante.
Lorsqu’aucun candidat `--model` n’est transmis, l’évaluation du caractère utilise par défaut
`openai/gpt-5.4`, `openai/gpt-5.2`, `openai/gpt-5`, `anthropic/claude-opus-4-6`,
`anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` et
`google/gemini-3.1-pro-preview` lorsqu’aucun `--model` n’est transmis.
Lorsqu’aucun `--judge-model` n’est transmis, les juges utilisent par défaut
`openai/gpt-5.4,thinking=xhigh,fast` et
`anthropic/claude-opus-4-6,thinking=high`.

## Documentation associée

- [Testing](/fr/help/testing)
- [QA Channel](/fr/channels/qa-channel)
- [Dashboard](/web/dashboard)
