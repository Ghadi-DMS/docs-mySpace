---
x-i18n:
    generated_at: "2026-04-18T06:44:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: dbb2c70c82da7f6f12d90e25666635ff4147c52e8a94135e902d1de4f5cbccca
    source_path: refactor/qa.md
    workflow: 15
---

# Refactorisation QA

Statut : la migration de base a été intégrée.

## Objectif

Faire évoluer la QA d’OpenClaw d’un modèle à définitions séparées vers une source de vérité unique :

- métadonnées des scénarios
- prompts envoyés au modèle
- configuration et nettoyage
- logique du harnais
- assertions et critères de réussite
- artefacts et indications de rapport

L’état final souhaité est un harnais QA générique qui charge des fichiers de définition de scénarios puissants au lieu de coder en dur la majeure partie du comportement en TypeScript.

## État actuel

La source de vérité principale vit désormais dans `qa/scenarios/index.md` plus un fichier par
scénario sous `qa/scenarios/<theme>/*.md`.

Implémenté :

- `qa/scenarios/index.md`
  - métadonnées canoniques du pack QA
  - identité de l’opérateur
  - mission de lancement
- `qa/scenarios/<theme>/*.md`
  - un fichier Markdown par scénario
  - métadonnées du scénario
  - associations de handlers
  - configuration d’exécution propre au scénario
- `extensions/qa-lab/src/scenario-catalog.ts`
  - parseur de pack Markdown + validation zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - rendu du plan à partir du pack Markdown
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - alimente les fichiers de compatibilité générés plus `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - sélectionne les scénarios exécutables via des associations de handlers définies en Markdown
- Protocole de bus QA + interface utilisateur
  - pièces jointes inline génériques pour le rendu image/vidéo/audio/fichier

Surfaces encore séparées :

- `extensions/qa-lab/src/suite.ts`
  - possède encore la majeure partie de la logique de handlers personnalisés exécutables
- `extensions/qa-lab/src/report.ts`
  - dérive encore la structure du rapport à partir des sorties d’exécution

La séparation de la source de vérité est donc corrigée, mais l’exécution reste encore majoritairement pilotée par des handlers plutôt que pleinement déclarative.

## À quoi ressemble réellement la surface des scénarios

La lecture de la suite actuelle montre quelques classes de scénarios distinctes.

### Interaction simple

- référence de canal
- référence DM
- suivi en fil
- changement de modèle
- poursuite après approbation
- réaction/édition/suppression

### Mutation de configuration et d’exécution

- correctif de config avec désactivation de skill
- réveil après redémarrage suite à application de config
- bascule de capacité après redémarrage de config
- vérification de dérive de l’inventaire d’exécution

### Assertions sur le système de fichiers et le dépôt

- rapport de découverte source/docs
- build de Lobster Invaders
- recherche d’artefact d’image généré

### Orchestration de la mémoire

- rappel mémoire
- outils mémoire dans le contexte du canal
- repli en cas d’échec mémoire
- classement de la mémoire de session
- isolation mémoire par fil
- balayage Dreaming de la mémoire

### Intégration d’outils et de plugins

- appel MCP plugin-tools
- visibilité des skill
- installation à chaud de skill
- génération d’image native
- aller-retour d’image
- compréhension d’image à partir d’une pièce jointe

### Multi-tour et multi-acteur

- transfert vers un sous-agent
- synthèse par répartition sur plusieurs sous-agents
- flux de type récupération après redémarrage

Ces catégories sont importantes car elles déterminent les exigences du DSL. Une liste plate de prompt + texte attendu ne suffit pas.

## Orientation

### Source de vérité unique

Utiliser `qa/scenarios/index.md` plus `qa/scenarios/<theme>/*.md` comme
source de vérité rédigée.

Le pack doit rester :

- lisible par un humain en revue
- analysable par la machine
- assez riche pour piloter :
  - l’exécution de la suite
  - le bootstrap de l’espace de travail QA
  - les métadonnées de l’interface QA Lab
  - les prompts de docs/découverte
  - la génération de rapports

### Format de rédaction préféré

Utiliser Markdown comme format de haut niveau, avec du YAML structuré à l’intérieur.

Forme recommandée :

- frontmatter YAML
  - id
  - title
  - surface
  - tags
  - docs refs
  - code refs
  - remplacements de modèle/provider
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
- un contexte plus riche que du YAML pur
- un parsing strict et une validation zod

Le JSON brut n’est acceptable qu’en tant que forme intermédiaire générée.

## Forme proposée du fichier de scénario

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

D’après la suite actuelle, le runner générique a besoin de plus que l’exécution de prompts.

### Actions d’environnement et de configuration

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### Actions de tour agent

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

### Actions sur les fichiers et artefacts

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### Actions mémoire et Cron

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
- générer une chaîne de marqueur de réveil, puis vérifier qu’elle apparaît plus tard

Capacités nécessaires :

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- références typées pour les chemins, clés de session, ids de fil, marqueurs, sorties d’outils

Sans prise en charge des variables, le harnais continuera à faire fuir la logique des scénarios vers TypeScript.

## Ce qui doit rester comme échappatoires

Un runner entièrement déclaratif pur n’est pas réaliste en phase 1.

Certains scénarios sont intrinsèquement lourds en orchestration :

- balayage Dreaming de la mémoire
- réveil après redémarrage suite à application de config
- bascule de capacité après redémarrage de config
- résolution d’artefact d’image généré par horodatage/chemin
- évaluation du rapport de découverte

Ceux-ci devraient utiliser pour l’instant des handlers personnalisés explicites.

Règle recommandée :

- 85-90 % déclaratif
- étapes `customHandler` explicites pour le reste difficile
- seulement des handlers personnalisés nommés et documentés
- aucun code inline anonyme dans le fichier de scénario

Cela garde le moteur générique propre tout en permettant d’avancer.

## Changement d’architecture

### Actuel

Le Markdown de scénario est déjà la source de vérité pour :

- l’exécution de la suite
- les fichiers de bootstrap de l’espace de travail
- le catalogue de scénarios de l’interface QA Lab
- les métadonnées de rapport
- les prompts de découverte

Compatibilité générée :

- l’espace de travail initialisé inclut encore `QA_KICKOFF_TASK.md`
- l’espace de travail initialisé inclut encore `QA_SCENARIO_PLAN.md`
- l’espace de travail initialisé inclut désormais aussi `QA_SCENARIOS.md`

## Plan de refactorisation

### Phase 1 : chargeur et schéma

Terminé.

- ajout de `qa/scenarios/index.md`
- découpage des scénarios dans `qa/scenarios/<theme>/*.md`
- ajout d’un parseur pour le contenu de pack YAML Markdown nommé
- validation avec zod
- bascule des consommateurs vers le pack parsé
- suppression de `qa/seed-scenarios.json` et `qa/QA_KICKOFF_TASK.md` au niveau du dépôt

### Phase 2 : moteur générique

- découper `extensions/qa-lab/src/suite.ts` en :
  - loader
  - moteur
  - registre d’actions
  - registre d’assertions
  - handlers personnalisés
- conserver les fonctions utilitaires existantes comme opérations du moteur

Livrable :

- le moteur exécute des scénarios déclaratifs simples

Commencer par des scénarios qui sont principalement prompt + attente + assertion :

- suivi en fil
- compréhension d’image à partir d’une pièce jointe
- visibilité et invocation de skill
- référence de canal

Livrable :

- premiers vrais scénarios définis en Markdown livrés via le moteur générique

### Phase 4 : migrer les scénarios intermédiaires

- aller-retour de génération d’image
- outils mémoire dans le contexte du canal
- classement de la mémoire de session
- transfert vers un sous-agent
- synthèse par répartition sur plusieurs sous-agents

Livrable :

- variables, artefacts, assertions d’outils, assertions de journal de requêtes validés

### Phase 5 : conserver les scénarios difficiles sur des handlers personnalisés

- balayage Dreaming de la mémoire
- réveil après redémarrage suite à application de config
- bascule de capacité après redémarrage de config
- dérive de l’inventaire d’exécution

Livrable :

- même format de rédaction, mais avec des blocs d’étapes personnalisées explicites quand nécessaire

### Phase 6 : supprimer la map de scénarios codée en dur

Une fois que la couverture du pack est suffisamment bonne :

- supprimer la majeure partie des branchements TypeScript spécifiques aux scénarios de `extensions/qa-lab/src/suite.ts`

## Prise en charge du faux Slack / des médias enrichis

Le bus QA actuel est centré sur le texte.

Fichiers concernés :

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

Aujourd’hui, le bus QA prend en charge :

- texte
- réactions
- fils

Il ne modélise pas encore les pièces jointes média inline.

### Contrat de transport nécessaire

Ajouter un modèle générique de pièce jointe au bus QA :

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

### Pourquoi d’abord générique

Ne pas construire un modèle de média propre à Slack uniquement.

À la place :

- un modèle de transport QA générique unique
- plusieurs moteurs de rendu au-dessus
  - le chat QA Lab actuel
  - un futur faux web Slack
  - toute autre vue de faux transport

Cela évite la duplication de logique et permet aux scénarios média de rester indépendants du transport.

### Travail d’interface nécessaire

Mettre à jour l’interface QA pour afficher :

- aperçu d’image inline
- lecteur audio inline
- lecteur vidéo inline
- puce de pièce jointe fichier

L’interface actuelle sait déjà afficher les fils et les réactions, donc le rendu des pièces jointes devrait se superposer au même modèle de carte de message.

### Travail de scénario rendu possible par le transport média

Une fois que les pièces jointes circulent via le bus QA, nous pouvons ajouter des scénarios de faux chat plus riches :

- réponse avec image inline dans un faux Slack
- compréhension de pièce jointe audio
- compréhension de pièce jointe vidéo
- ordre mixte des pièces jointes
- réponse dans un fil avec conservation du média

## Recommandation

Le prochain lot d’implémentation devrait être :

1. ajouter un chargeur de scénarios Markdown + schéma zod
2. générer le catalogue actuel à partir du Markdown
3. migrer d’abord quelques scénarios simples
4. ajouter la prise en charge générique des pièces jointes du bus QA
5. afficher l’image inline dans l’interface QA
6. puis étendre à l’audio et à la vidéo

C’est le plus petit chemin qui valide les deux objectifs :

- QA générique définie par Markdown
- surfaces de messagerie simulées plus riches

## Questions ouvertes

- si les fichiers de scénario doivent autoriser des modèles de prompt Markdown embarqués avec interpolation de variables
- si setup/cleanup doivent être des sections nommées ou simplement des listes d’actions ordonnées
- si les références d’artefacts doivent être fortement typées dans le schéma ou basées sur des chaînes
- si les handlers personnalisés doivent vivre dans un registre unique ou dans des registres par surface
- si le fichier de compatibilité JSON généré doit rester versionné pendant la migration
