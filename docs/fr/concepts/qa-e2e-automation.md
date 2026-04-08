---
read_when:
    - Extension de qa-lab ou de qa-channel
    - Ajout de scénarios QA adossés au dépôt
    - Création d’une automatisation QA plus réaliste autour du tableau de bord Gateway
summary: Structure de l’automatisation QA privée pour qa-lab, qa-channel, les scénarios préconfigurés et les rapports de protocole
title: Automatisation E2E QA
x-i18n:
    generated_at: "2026-04-08T06:00:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 57da147dc06abf9620290104e01a83b42182db1806514114fd9e8467492cda99
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatisation E2E QA

La pile QA privée est conçue pour tester OpenClaw d’une manière plus réaliste,
structurée autour des canaux, qu’un simple test unitaire ne peut le faire.

Éléments actuels :

- `extensions/qa-channel` : canal de messages synthétiques avec des surfaces pour les DM, les canaux, les fils, les réactions, les modifications et les suppressions.
- `extensions/qa-lab` : interface de débogage et bus QA pour observer la transcription, injecter des messages entrants et exporter un rapport Markdown.
- `qa/` : ressources préconfigurées adossées au dépôt pour la tâche de démarrage et les scénarios QA de référence.

Le flux opérateur QA actuel repose sur un site QA à deux volets :

- Gauche : tableau de bord Gateway (Control UI) avec l’agent.
- Droite : QA Lab, affichant la transcription de type Slack et le plan du scénario.

Exécutez-le avec :

```bash
pnpm qa:lab:up
```

Cela construit le site QA, démarre la voie Gateway adossée à Docker et expose la
page QA Lab où un opérateur ou une boucle d’automatisation peut confier à
l’agent une mission QA, observer le comportement réel du canal et consigner ce
qui a fonctionné, échoué ou est resté bloqué.

Pour des itérations plus rapides sur l’interface de QA Lab sans reconstruire
l’image Docker à chaque fois, démarrez la pile avec un bundle QA Lab monté par
liaison :

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` conserve les services Docker sur une image préconstruite et
monte par liaison `extensions/qa-lab/web/dist` dans le conteneur `qa-lab`.
`qa:lab:watch` reconstruit ce bundle à chaque modification, et le navigateur se
recharge automatiquement lorsque le hachage des ressources de QA Lab change.

## Ressources préconfigurées adossées au dépôt

Les ressources préconfigurées se trouvent dans `qa/` :

- `qa/scenarios/index.md`
- `qa/scenarios/*.md`

Elles sont volontairement versionnées dans git afin que le plan QA soit visible
à la fois pour les humains et pour l’agent. La liste de référence doit rester
assez large pour couvrir :

- chat en DM et en canal
- comportement des fils
- cycle de vie des actions sur les messages
- rappels cron
- rappel de mémoire
- changement de modèle
- transfert à un sous-agent
- lecture du dépôt et de la documentation
- une petite tâche de build comme Lobster Invaders

## Rapports

`qa-lab` exporte un rapport de protocole Markdown à partir de la chronologie du
bus observé.
Le rapport doit répondre aux questions suivantes :

- Ce qui a fonctionné
- Ce qui a échoué
- Ce qui est resté bloqué
- Les scénarios de suivi qu’il vaut la peine d’ajouter

## Documentation associée

- [Tests](/fr/help/testing)
- [Canal QA](/fr/channels/qa-channel)
- [Tableau de bord](/web/dashboard)
