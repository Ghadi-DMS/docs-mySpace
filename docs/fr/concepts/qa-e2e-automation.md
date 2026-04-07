---
read_when:
    - Étendre qa-lab ou qa-channel
    - Ajouter des scénarios QA adossés au dépôt
    - Créer une automatisation QA plus réaliste autour du tableau de bord Gateway
summary: Structure de l’automatisation QA privée pour qa-lab, qa-channel, les scénarios initiaux et les rapports de protocole
title: Automatisation QA E2E
x-i18n:
    generated_at: "2026-04-07T06:49:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 113e89d8d3ee8ef3058d95b9aea9a1c2335b07794446be2d231c0faeb044b23b
    source_path: concepts/qa-e2e-automation.md
    workflow: 15
---

# Automatisation QA E2E

La pile QA privée est conçue pour exercer OpenClaw de manière plus réaliste,
avec une forme proche d’un canal, qu’un simple test unitaire ne peut le faire.

Éléments actuels :

- `extensions/qa-channel` : canal de messages synthétique avec surfaces de MP, de canal, de fil,
  de réaction, de modification et de suppression.
- `extensions/qa-lab` : interface de débogage et bus QA pour observer la transcription,
  injecter des messages entrants et exporter un rapport Markdown.
- `qa/` : ressources initiales adossées au dépôt pour la tâche de lancement et les
  scénarios QA de base.

Le flux opérateur QA actuel est un site QA à deux volets :

- Gauche : tableau de bord Gateway (Control UI) avec l’agent.
- Droite : QA Lab, affichant la transcription de style Slack et le plan de scénario.

Lancez-le avec :

```bash
pnpm qa:lab:up
```

Cela construit le site QA, démarre la voie Gateway adossée à Docker et expose la
page QA Lab où un opérateur ou une boucle d’automatisation peut confier à l’agent une
mission QA, observer le comportement réel du canal et consigner ce qui a fonctionné,
échoué ou est resté bloqué.

## Ressources initiales adossées au dépôt

Les ressources initiales se trouvent dans `qa/` :

- `qa/QA_KICKOFF_TASK.md`
- `qa/seed-scenarios.json`

Elles sont volontairement conservées dans git afin que le plan QA soit visible à la fois pour les humains et pour l’agent. La liste de base doit rester assez large pour couvrir :

- discussion en MP et dans les canaux
- comportement des fils
- cycle de vie des actions sur les messages
- rappels cron
- rappel de mémoire
- changement de modèle
- transfert à un sous-agent
- lecture du dépôt et de la documentation
- une petite tâche de build telle que Lobster Invaders

## Rapports

`qa-lab` exporte un rapport de protocole Markdown à partir de la chronologie observée du bus.
Le rapport doit répondre à ces questions :

- Ce qui a fonctionné
- Ce qui a échoué
- Ce qui est resté bloqué
- Quels scénarios de suivi valent la peine d’être ajoutés

## Documentation connexe

- [Tests](/fr/help/testing)
- [QA Channel](/fr/channels/qa-channel)
- [Tableau de bord](/web/dashboard)
