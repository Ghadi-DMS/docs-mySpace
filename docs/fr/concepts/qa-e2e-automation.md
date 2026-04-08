---
read_when:
    - Étendre qa-lab ou qa-channel
    - Ajouter des scénarios QA adossés au dépôt
    - Créer une automatisation QA plus réaliste autour du tableau de bord Gateway
summary: Structure de l’automatisation QA privée pour qa-lab, qa-channel, les scénarios préconfigurés et les rapports de protocole
title: Automatisation QA E2E
x-i18n:
    generated_at: "2026-04-08T02:14:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3b4aa5acc8e77303f4045d4f04372494cae21b89d2fdaba856dbb4855ced9d27
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatisation QA E2E

La pile QA privée est conçue pour exercer OpenClaw d’une manière plus réaliste,
calquée sur les canaux, qu’un simple test unitaire ne le peut.

Éléments actuels :

- `extensions/qa-channel` : canal de messages synthétique avec des surfaces de MP, de canal, de fil,
  de réaction, de modification et de suppression.
- `extensions/qa-lab` : interface de débogage et bus QA pour observer la transcription,
  injecter des messages entrants et exporter un rapport Markdown.
- `qa/` : ressources préconfigurées adossées au dépôt pour la tâche de démarrage et les scénarios QA
  de référence.

Le flux opérateur QA actuel repose sur un site QA à deux panneaux :

- Gauche : tableau de bord Gateway (Control UI) avec l’agent.
- Droite : QA Lab, affichant la transcription de type Slack et le plan de scénario.

Lancez-le avec :

```bash
pnpm qa:lab:up
```

Cela construit le site QA, démarre la voie Gateway adossée à Docker et expose la
page QA Lab où un opérateur ou une boucle d’automatisation peut confier à l’agent une mission QA,
observer le comportement réel du canal et consigner ce qui a fonctionné, échoué ou
est resté bloqué.

Pour itérer plus rapidement sur l’interface de QA Lab sans reconstruire l’image Docker à chaque fois,
démarrez la pile avec un bundle QA Lab monté par liaison :

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` maintient les services Docker sur une image préconstruite et monte par liaison
`extensions/qa-lab/web/dist` dans le conteneur `qa-lab`. `qa:lab:watch`
reconstruit ce bundle à chaque modification, et le navigateur se recharge automatiquement lorsque le hachage
des ressources de QA Lab change.

## Ressources préconfigurées adossées au dépôt

Les ressources préconfigurées se trouvent dans `qa/` :

- `qa/scenarios.md`

Elles sont volontairement versionnées dans git afin que le plan QA soit visible à la fois pour les humains et pour
l’agent. La liste de référence doit rester suffisamment large pour couvrir :

- chat en MP et en canal
- comportement des fils
- cycle de vie des actions sur les messages
- rappels cron
- rappel de mémoire
- changement de modèle
- transfert vers un sous-agent
- lecture du dépôt et de la documentation
- une petite tâche de build comme Lobster Invaders

## Rapports

`qa-lab` exporte un rapport de protocole Markdown à partir de la chronologie observée du bus.
Le rapport doit répondre à :

- Ce qui a fonctionné
- Ce qui a échoué
- Ce qui est resté bloqué
- Quels scénarios de suivi méritent d’être ajoutés

## Documentation associée

- [Testing](/fr/help/testing)
- [QA Channel](/fr/channels/qa-channel)
- [Dashboard](/web/dashboard)
