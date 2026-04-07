---
read_when:
    - Vous intégrez le transport QA synthétique dans une exécution de test locale ou CI
    - Vous avez besoin de la surface de configuration du `qa-channel` intégré
    - Vous itérez sur l'automatisation QA de bout en bout
summary: Plugin de canal synthétique de type Slack pour des scénarios QA OpenClaw déterministes
title: Canal QA
x-i18n:
    generated_at: "2026-04-07T06:48:34Z"
    model: gpt-5.4
    provider: openai
    source_hash: 65c2c908d3ec27c827087616c4ea278f10686810091058321ff26f68296a1782
    source_path: channels/qa-channel.md
    workflow: 15
---

# Canal QA

`qa-channel` est un transport de messages synthétique intégré pour la QA automatisée d'OpenClaw.

Ce n'est pas un canal de production. Il existe pour exercer la même frontière
de plugin de canal que celle utilisée par les transports réels, tout en gardant
un état déterministe et entièrement inspectable.

## Ce qu'il fait aujourd'hui

- Grammaire de cible de type Slack :
  - `dm:<user>`
  - `channel:<room>`
  - `thread:<room>/<thread>`
- Bus synthétique adossé à HTTP pour :
  - l'injection de messages entrants
  - la capture des transcriptions sortantes
  - la création de fils
  - les réactions
  - les modifications
  - les suppressions
  - les actions de recherche et de lecture
- Exécuteur de vérification automatique intégré côté hôte qui écrit un rapport Markdown

## Configuration

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

Clés de compte prises en charge :

- `baseUrl`
- `botUserId`
- `botDisplayName`
- `pollTimeoutMs`
- `allowFrom`
- `defaultTo`
- `actions.messages`
- `actions.reactions`
- `actions.search`
- `actions.threads`

## Exécuteur

Tranche verticale actuelle :

```bash
pnpm qa:e2e
```

Cela passe désormais par l'extension `qa-lab` intégrée. Elle démarre le bus QA
du dépôt, lance la tranche d'exécution `qa-channel` intégrée, exécute une
vérification automatique déterministe, et écrit un rapport Markdown dans
`.artifacts/qa-e2e/`.

Interface de débogage privée :

```bash
pnpm qa:lab:up
```

Cette commande unique construit le site QA, démarre la pile gateway + QA Lab
adossée à Docker, et affiche l'URL de QA Lab. Depuis ce site, vous pouvez
sélectionner des scénarios, choisir la voie de modèle, lancer des exécutions
individuelles, et suivre les résultats en direct.

Suite QA complète adossée au dépôt :

```bash
pnpm openclaw qa suite
```

Elle lance le débogueur QA privé sur une URL locale, séparée du bundle de
Control UI livré.

## Portée

La portée actuelle est volontairement limitée :

- bus + transport de plugin
- grammaire de routage par fil
- actions de message gérées par le canal
- rapports Markdown
- site QA adossé à Docker avec contrôles d'exécution

Les travaux de suivi ajouteront :

- l'exécution d'une matrice fournisseur/modèle
- une découverte de scénarios plus riche
- une orchestration native OpenClaw plus tard
