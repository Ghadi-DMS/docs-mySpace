---
x-i18n:
    generated_at: "2026-04-08T06:01:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4a9066b2a939c5a9ba69141d75405f0e8097997b523164340e2f0e9a0d5060dd
    source_path: refactor/qa.md
    workflow: 15
---

# Refactorisation de la QA

Statut : migration fondamentale terminée.

## Objectif

Faire évoluer la QA d’OpenClaw d’un modèle à définition répartie vers une source de vérité unique :

- métadonnées des scénarios
- prompts envoyés au modèle
- configuration et nettoyage
- logique du harnais
- assertions et critères de réussite
- artefacts et indications pour les rapports

L’état final souhaité est un harnais QA générique qui charge de puissants fichiers de définition de scénario au lieu de coder en dur la majeure partie du comportement en TypeScript.

## État actuel

La source principale de vérité se trouve désormais dans `qa/scenarios/index.md` plus un fichier par scénario sous `qa/scenarios/*.md`.

Implémenté :

- `qa/scenarios/index.md`
  - métadonnées canoniques du pack QA
  - identité de l’opérateur
  - mission de lancement
- `qa/scenarios/*.md`
  - un fichier Markdown par scénario
  - métadonnées du scénario
  - liaisons de handlers
  - configuration d’exécution spécifique au scénario
- `extensions/qa-lab/src/scenario-catalog.ts`
  - parseur de pack Markdown + validation zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - rendu du plan à partir du pack Markdown
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - génère les fichiers de compatibilité initiaux plus `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - sélectionne les scénarios exécutables via des liaisons de handlers définies en Markdown
- Protocole QA bus + UI
  - pièces jointes inline génériques pour le rendu image/vidéo/audio/fichier

Surfaces encore réparties :

- `extensions/qa-lab/src/suite.ts`
  - possède encore la majeure partie de la logique des handlers exécutables personnalisés
- `extensions/qa-lab/src/report.ts`
  - dérive encore la structure du rapport à partir des sorties d’exécution

Ainsi, la répartition de la source de vérité est corrigée, mais l’exécution reste encore majoritairement appuyée sur des handlers plutôt que pleinement déclarative.

## À quoi ressemble réellement la surface des scénarios

La lecture de la suite actuelle montre quelques classes de scénarios distinctes.

### Interaction simple

- référence de base du canal
- référence de base en message direct
- suivi dans un fil
- changement de modèle
- suivi après approbation
- réaction/modification/suppression

### Mutation de configuration et d’exécution

- désactivation de skill par patch de config
- config apply restart wake-up
- inversion de capacité après redémarrage de configuration
- vérification de dérive de l’inventaire d’exécution

### Assertions sur système de fichiers et dépôt

- rapport de découverte source/docs
- build Lobster Invaders
- recherche d’artefact d’image générée

### Orchestration de la mémoire

- rappel de mémoire
- outils de mémoire dans un contexte de canal
- repli en cas d’échec de la mémoire
- classement de la mémoire de session
- isolation de la mémoire par fil
- memory dreaming sweep

### Intégration d’outils et de plugins

- appel MCP plugin-tools
- visibilité des Skills
- installation à chaud de skill
- génération d’image native
- aller-retour d’image
- compréhension d’image à partir d’une pièce jointe

### Multi-tour et multi-acteur

- transfert vers un sous-agent
- synthèse par fanout de sous-agents
- flux de style reprise après redémarrage

Ces catégories sont importantes parce qu’elles pilotent les exigences du DSL. Une simple liste de prompt + texte attendu ne suffit pas.

## Orientation

### Source de vérité unique

Utiliser `qa/scenarios/index.md` plus `qa/scenarios/*.md` comme source de vérité rédigée.

Le pack doit rester :

- lisible par un humain en revue
- analysable par machine
- suffisamment riche pour piloter :
  - l’exécution de la suite
  - l’initialisation de l’espace de travail QA
  - les métadonnées de l’UI QA Lab
  - les prompts de documentation/découverte
  - la génération de rapports

### Format de rédaction préféré

Utiliser Markdown comme format de premier niveau, avec du YAML structuré à l’intérieur.

Forme recommandée :

- frontmatter YAML
  - id
  - title
  - surface
  - tags
  - docs refs
  - code refs
  - surcharges de modèle/fournisseur
  - prérequis
- sections en prose
  - objectif
  - notes
  - indications de débogage
- blocs YAML délimités
  - setup
  - steps
  - assertions
  - cleanup

Cela apporte :

- une meilleure lisibilité en PR qu’un énorme JSON
- un contexte plus riche qu’un simple YAML
- une analyse stricte et une validation zod

Le JSON brut n’est acceptable qu’en tant que forme intermédiaire générée.

## Forme proposée pour un fichier de scénario

Exemple :

````md
---
id: image-generation-roundtrip
title: Image generation roundtrip
surface: image
tags: [media, image, roundtrip]
models:
  primary: openai/gpt-5.4
requires:
  tools: [image_generate]
  plugins: [openai, qa-channel]
docsRefs:
  - docs/help/testing.md
  - docs/concepts/model-providers.md
codeRefs:
  - extensions/qa-lab/src/suite.ts
  - src/gateway/chat-attachments.ts
---

# Objective

Verify generated media is reattached on the follow-up turn.

# Setup

```yaml scenario.setup
- action: config.patch
  patch:
    agents:
      defaults:
        imageGenerationModel:
          primary: openai/gpt-image-1
- action: session.create
  key: agent:qa:image-roundtrip
```

# Steps

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Image generation check: generate a QA lighthouse image and summarize it in one short sentence.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Image generation check
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Roundtrip image inspection check: describe the generated lighthouse attachment in one short sentence.
  attachments:
    - fromArtifact: lighthouseImage
```

# Expect

```yaml scenario.expect
- assert: outbound.textIncludes
  value: lighthouse
- assert: requestLog.matches
  where:
    promptIncludes: Roundtrip image inspection check
  imageInputCountGte: 1
- assert: artifact.exists
  ref: lighthouseImage
```
````

## Capacités du runner que le DSL doit couvrir

D’après la suite actuelle, le runner générique a besoin de plus que de l’exécution de prompts.

### Actions d’environnement et de configuration

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### Actions de tour d’agent

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### Actions de configuration et d’exécution

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### Actions sur fichiers et artefacts

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### Actions de mémoire et de cron

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### Actions MCP

- `mcp.callTool`

### Assertions

- `outbound.textIncludes`
- `outbound.inThread`
- `outbound.notInRoot`
- `tool.called`
- `tool.notPresent`
- `skill.visible`
- `skill.disabled`
- `file.contains`
- `memory.contains`
- `requestLog.matches`
- `sessionStore.matches`
- `cron.managedPresent`
- `artifact.exists`

## Variables et références d’artefacts

Le DSL doit prendre en charge les sorties enregistrées et les références ultérieures.

Exemples issus de la suite actuelle :

- créer un fil, puis réutiliser `threadId`
- créer une session, puis réutiliser `sessionKey`
- générer une image, puis joindre le fichier au tour suivant
- générer une chaîne marqueur de réveil, puis vérifier qu’elle apparaît plus tard

Capacités nécessaires :

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- références typées pour les chemins, clés de session, identifiants de fil, marqueurs, sorties d’outils

Sans prise en charge des variables, le harnais continuera à faire fuiter la logique des scénarios vers le TypeScript.

## Ce qui doit rester comme échappatoires

Un runner purement déclaratif n’est pas réaliste en phase 1.

Certains scénarios sont intrinsèquement lourds en orchestration :

- memory dreaming sweep
- config apply restart wake-up
- config restart capability flip
- résolution d’artefact d’image générée par horodatage/chemin
- évaluation du rapport de découverte

Ils devraient pour l’instant utiliser des handlers personnalisés explicites.

Règle recommandée :

- 85-90 % déclaratif
- étapes `customHandler` explicites pour le reste difficile
- uniquement des handlers personnalisés nommés et documentés
- aucun code inline anonyme dans le fichier de scénario

Cela garde le moteur générique propre tout en permettant d’avancer.

## Changement d’architecture

### Actuel

Le Markdown des scénarios est déjà la source de vérité pour :

- l’exécution de la suite
- les fichiers d’initialisation de l’espace de travail
- le catalogue de scénarios de l’UI QA Lab
- les métadonnées de rapport
- les prompts de découverte

Compatibilité générée :

- l’espace de travail initial inclut toujours `QA_KICKOFF_TASK.md`
- l’espace de travail initial inclut toujours `QA_SCENARIO_PLAN.md`
- l’espace de travail initial inclut désormais aussi `QA_SCENARIOS.md`

## Plan de refactorisation

### Phase 1 : chargeur et schéma

Terminé.

- ajout de `qa/scenarios/index.md`
- séparation des scénarios dans `qa/scenarios/*.md`
- ajout d’un parseur pour le contenu YAML Markdown nommé du pack
- validation avec zod
- basculement des consommateurs vers le pack analysé
- suppression de `qa/seed-scenarios.json` et `qa/QA_KICKOFF_TASK.md` au niveau du dépôt

### Phase 2 : moteur générique

- diviser `extensions/qa-lab/src/suite.ts` en :
  - chargeur
  - moteur
  - registre d’actions
  - registre d’assertions
  - handlers personnalisés
- conserver les fonctions utilitaires existantes comme opérations du moteur

Livrable :

- le moteur exécute des scénarios déclaratifs simples

Commencer par les scénarios qui sont principalement prompt + attente + assertion :

- suivi dans un fil
- compréhension d’image à partir d’une pièce jointe
- visibilité et invocation de Skills
- référence de base du canal

Livrable :

- premiers vrais scénarios définis en Markdown livrés via le moteur générique

### Phase 4 : migrer les scénarios intermédiaires

- aller-retour de génération d’image
- outils de mémoire dans un contexte de canal
- classement de la mémoire de session
- transfert vers un sous-agent
- synthèse par fanout de sous-agents

Livrable :

- variables, artefacts, assertions d’outils, assertions de journal de requêtes validés

### Phase 5 : conserver les scénarios difficiles sur des handlers personnalisés

- memory dreaming sweep
- config apply restart wake-up
- config restart capability flip
- dérive d’inventaire d’exécution

Livrable :

- même format de rédaction, mais avec des blocs d’étapes personnalisées explicites là où nécessaire

### Phase 6 : supprimer la map de scénarios codée en dur

Quand la couverture du pack sera suffisamment bonne :

- supprimer la majeure partie du branchement TypeScript spécifique aux scénarios dans `extensions/qa-lab/src/suite.ts`

## Faux Slack / prise en charge des médias riches

Le QA bus actuel est centré sur le texte.

Fichiers concernés :

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

Aujourd’hui, le QA bus prend en charge :

- texte
- réactions
- fils

Il ne modélise pas encore les pièces jointes média inline.

### Contrat de transport nécessaire

Ajouter un modèle générique de pièce jointe QA bus :

```ts
type QaBusAttachment = {
  id: string;
  kind: "image" | "video" | "audio" | "file";
  mimeType: string;
  fileName?: string;
  inline?: boolean;
  url?: string;
  contentBase64?: string;
  width?: number;
  height?: number;
  durationMs?: number;
  altText?: string;
  transcript?: string;
};
```

Puis ajouter `attachments?: QaBusAttachment[]` à :

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### Pourquoi commencer par du générique

Ne construisez pas un modèle média réservé à Slack.

À la place :

- un modèle de transport QA générique
- plusieurs moteurs de rendu par-dessus
  - le chat actuel de QA Lab
  - un futur faux Slack web
  - toute autre vue de faux transport

Cela évite la logique dupliquée et permet aux scénarios média de rester indépendants du transport.

### Travail UI nécessaire

Mettre à jour l’UI QA pour afficher :

- aperçu d’image inline
- lecteur audio inline
- lecteur vidéo inline
- puce de pièce jointe de fichier

L’UI actuelle peut déjà afficher les fils et les réactions, donc le rendu des pièces jointes devrait pouvoir se superposer au même modèle de carte de message.

### Travail sur les scénarios rendu possible par le transport média

Une fois que les pièces jointes circuleront dans le QA bus, nous pourrons ajouter des scénarios de faux chat plus riches :

- réponse avec image inline dans un faux Slack
- compréhension d’une pièce jointe audio
- compréhension d’une pièce jointe vidéo
- ordre mixte des pièces jointes
- réponse dans un fil avec conservation du média

## Recommandation

Le prochain bloc d’implémentation devrait être :

1. ajouter un chargeur de scénarios Markdown + un schéma zod
2. générer le catalogue actuel à partir du Markdown
3. migrer d’abord quelques scénarios simples
4. ajouter la prise en charge générique des pièces jointes QA bus
5. afficher une image inline dans l’UI QA
6. puis étendre à l’audio et à la vidéo

C’est le plus petit chemin qui prouve les deux objectifs :

- QA générique définie en Markdown
- surfaces de messagerie simulées plus riches

## Questions ouvertes

- si les fichiers de scénario doivent autoriser des modèles de prompt Markdown embarqués avec interpolation de variables
- si setup/cleanup doivent être des sections nommées ou simplement des listes d’actions ordonnées
- si les références d’artefacts doivent être fortement typées dans le schéma ou basées sur des chaînes
- si les handlers personnalisés doivent vivre dans un seul registre ou dans des registres par surface
- si le fichier de compatibilité JSON généré doit rester versionné pendant la migration
