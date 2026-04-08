---
x-i18n:
    generated_at: "2026-04-08T02:18:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0e156cc8e2fe946a0423862f937754a7caa1fe7e6863b50a80bff49a1c86e1e8
    source_path: refactor/qa.md
    workflow: 15
---

# Refactorisation QA

Statut : la migration fondamentale est finalisée.

## Objectif

Faire évoluer la QA d'OpenClaw d'un modèle à définition répartie vers une source de vérité unique :

- métadonnées du scénario
- prompts envoyés au modèle
- configuration et nettoyage
- logique du harnais
- assertions et critères de réussite
- artefacts et indications de rapport

L'état final souhaité est un harnais QA générique qui charge des fichiers de définition de scénario puissants au lieu de coder en dur la majeure partie du comportement en TypeScript.

## État actuel

La source de vérité principale se trouve maintenant dans `qa/scenarios.md`.

Implémenté :

- `qa/scenarios.md`
  - pack QA canonique
  - identité de l'opérateur
  - mission de lancement
  - métadonnées du scénario
  - liaisons de gestionnaires
- `extensions/qa-lab/src/scenario-catalog.ts`
  - parseur de pack Markdown + validation zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - rendu du plan à partir du pack Markdown
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - initialise les fichiers de compatibilité générés ainsi que `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - sélectionne les scénarios exécutables via des liaisons de gestionnaires définies en Markdown
- Protocole de bus QA + UI
  - pièces jointes inline génériques pour le rendu d'image/vidéo/audio/fichier

Surfaces encore réparties :

- `extensions/qa-lab/src/suite.ts`
  - possède encore la majeure partie de la logique de gestionnaire personnalisé exécutable
- `extensions/qa-lab/src/report.ts`
  - dérive encore la structure du rapport à partir des sorties runtime

Ainsi, la répartition de la source de vérité est corrigée, mais l'exécution reste encore principalement adossée à des gestionnaires plutôt qu'entièrement déclarative.

## À quoi ressemble la vraie surface des scénarios

La lecture de la suite actuelle montre quelques classes de scénarios distinctes.

### Interaction simple

- base de référence du canal
- base de référence des messages privés
- suivi en fil
- changement de modèle
- poursuite après approbation
- réaction/modification/suppression

### Mutation de configuration et de runtime

- désactivation de skill via patch de configuration
- réveil après redémarrage suite à l'application de la configuration
- bascule de capacité après redémarrage de configuration
- vérification de dérive de l'inventaire runtime

### Assertions sur le système de fichiers et le dépôt

- rapport de découverte source/docs
- build de Lobster Invaders
- recherche d'artefact d'image générée

### Orchestration de la mémoire

- rappel mémoire
- outils mémoire dans le contexte du canal
- secours en cas d'échec mémoire
- classement de la mémoire de session
- isolation mémoire de fil
- balayage memory dreaming

### Intégration d'outils et de plugins

- appel MCP plugin-tools
- visibilité des skills
- installation à chaud de skill
- génération d'image native
- aller-retour d'image
- compréhension d'image depuis une pièce jointe

### Multi-tour et multi-acteur

- transfert à un sous-agent
- synthèse par fanout de sous-agent
- flux de type récupération après redémarrage

Ces catégories sont importantes parce qu'elles déterminent les exigences de la DSL. Une simple liste de prompt + texte attendu n'est pas suffisante.

## Direction

### Source de vérité unique

Utiliser `qa/scenarios.md` comme source de vérité rédigée.

Le pack doit rester :

- lisible par un humain en revue
- analysable par une machine
- suffisamment riche pour piloter :
  - l'exécution de la suite
  - l'initialisation du workspace QA
  - les métadonnées de l'UI QA Lab
  - les prompts de documentation/découverte
  - la génération de rapports

### Format de rédaction préféré

Utiliser Markdown comme format de premier niveau, avec du YAML structuré à l'intérieur.

Forme recommandée :

- frontmatter YAML
  - id
  - title
  - surface
  - tags
  - docs refs
  - code refs
  - remplacements de modèle/fournisseur
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

- une meilleure lisibilité en PR qu'un énorme JSON
- un contexte plus riche que du pur YAML
- une analyse stricte et une validation zod

Le JSON brut n'est acceptable que comme forme intermédiaire générée.

## Forme de fichier de scénario proposée

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

# Objectif

Vérifier que les médias générés sont rattachés au tour suivant.

# Configuration

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

# Étapes

```yaml scenario.steps
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Contrôle de génération d'image : génère une image de phare QA et résume-la en une phrase courte.
- action: artifact.capture
  kind: generated-image
  promptSnippet: Image generation check
  saveAs: lighthouseImage
- action: agent.send
  session: agent:qa:image-roundtrip
  message: |
    Contrôle d'inspection d'image en aller-retour : décris la pièce jointe générée du phare en une phrase courte.
  attachments:
    - fromArtifact: lighthouseImage
```

# Attendu

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

## Capacités du runner que la DSL doit couvrir

D'après la suite actuelle, le runner générique doit faire plus qu'exécuter des prompts.

### Actions d'environnement et de configuration

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### Actions de tour d'agent

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### Actions de configuration et de runtime

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

### Actions mémoire et cron

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

## Variables et références d'artefacts

La DSL doit prendre en charge les sorties enregistrées et leurs références ultérieures.

Exemples issus de la suite actuelle :

- créer un fil, puis réutiliser `threadId`
- créer une session, puis réutiliser `sessionKey`
- générer une image, puis joindre le fichier au tour suivant
- générer une chaîne de marqueur de réveil, puis vérifier qu'elle apparaît plus tard

Capacités nécessaires :

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- références typées pour les chemins, clés de session, identifiants de fil, marqueurs, sorties d'outil

Sans prise en charge des variables, le harnais continuera à faire fuir la logique de scénario vers TypeScript.

## Ce qui doit rester comme échappatoires

Un runner entièrement déclaratif pur n'est pas réaliste en phase 1.

Certains scénarios sont intrinsèquement lourds en orchestration :

- balayage memory dreaming
- réveil après redémarrage suite à l'application de la configuration
- bascule de capacité après redémarrage de configuration
- résolution d'artefact d'image générée par horodatage/chemin
- évaluation de rapport de découverte

Pour l'instant, ceux-ci doivent utiliser des gestionnaires personnalisés explicites.

Règle recommandée :

- 85-90 % déclaratif
- étapes `customHandler` explicites pour le reste difficile
- gestionnaires personnalisés nommés et documentés uniquement
- aucun code inline anonyme dans le fichier de scénario

Cela garde le moteur générique propre tout en permettant d'avancer.

## Changement d'architecture

### Actuel

Le Markdown de scénario est déjà la source de vérité pour :

- l'exécution de la suite
- les fichiers bootstrap du workspace
- le catalogue de scénarios de l'UI QA Lab
- les métadonnées de rapport
- les prompts de découverte

Compatibilité générée :

- le workspace initialisé inclut encore `QA_KICKOFF_TASK.md`
- le workspace initialisé inclut encore `QA_SCENARIO_PLAN.md`
- le workspace initialisé inclut maintenant aussi `QA_SCENARIOS.md`

## Plan de refactorisation

### Phase 1 : chargeur et schéma

Fait.

- ajout de `qa/scenarios.md`
- ajout d'un parseur pour le contenu du pack YAML Markdown nommé
- validation avec zod
- basculement des consommateurs vers le pack analysé
- suppression de `qa/seed-scenarios.json` et `qa/QA_KICKOFF_TASK.md` au niveau du dépôt

### Phase 2 : moteur générique

- découper `extensions/qa-lab/src/suite.ts` en :
  - chargeur
  - moteur
  - registre d'actions
  - registre d'assertions
  - gestionnaires personnalisés
- conserver les fonctions helper existantes comme opérations du moteur

Livrable :

- le moteur exécute des scénarios déclaratifs simples

Commencer par les scénarios qui sont surtout prompt + attente + assertion :

- suivi en fil
- compréhension d'image depuis une pièce jointe
- visibilité et invocation de skill
- base de référence du canal

Livrable :

- premiers vrais scénarios définis en Markdown livrés via le moteur générique

### Phase 4 : migrer les scénarios intermédiaires

- aller-retour de génération d'image
- outils mémoire dans le contexte du canal
- classement de la mémoire de session
- transfert à un sous-agent
- synthèse par fanout de sous-agent

Livrable :

- variables, artefacts, assertions sur outils, assertions sur journaux de requêtes validés en pratique

### Phase 5 : conserver les scénarios difficiles sur des gestionnaires personnalisés

- balayage memory dreaming
- réveil après redémarrage suite à l'application de la configuration
- bascule de capacité après redémarrage de configuration
- dérive d'inventaire runtime

Livrable :

- même format de rédaction, mais avec des blocs d'étapes personnalisées explicites lorsque nécessaire

### Phase 6 : supprimer la map de scénarios codée en dur

Une fois que la couverture du pack sera suffisante :

- supprimer la majeure partie du branchement TypeScript spécifique aux scénarios dans `extensions/qa-lab/src/suite.ts`

## Prise en charge du faux Slack / des médias enrichis

Le bus QA actuel est d'abord orienté texte.

Fichiers concernés :

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

Aujourd'hui, le bus QA prend en charge :

- texte
- réactions
- fils

Il ne modélise pas encore les pièces jointes média inline.

### Contrat de transport nécessaire

Ajouter un modèle générique de pièce jointe de bus QA :

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

### Pourquoi générique d'abord

Ne construisez pas un modèle média réservé à Slack.

À la place :

- un modèle de transport QA générique
- plusieurs moteurs de rendu au-dessus
  - le chat QA Lab actuel
  - un futur faux web Slack
  - toute autre vue de faux transport

Cela évite la duplication de logique et permet aux scénarios média de rester indépendants du transport.

### Travail UI nécessaire

Mettre à jour l'UI QA pour afficher :

- aperçu d'image inline
- lecteur audio inline
- lecteur vidéo inline
- puce de pièce jointe de fichier

L'UI actuelle peut déjà afficher les fils et les réactions, donc le rendu des pièces jointes devrait se superposer au même modèle de carte de message.

### Travail de scénario rendu possible par le transport média

Une fois que les pièces jointes circulent via le bus QA, nous pouvons ajouter des scénarios de faux chat plus riches :

- réponse avec image inline dans le faux Slack
- compréhension de pièce jointe audio
- compréhension de pièce jointe vidéo
- ordre mixte des pièces jointes
- réponse en fil avec média conservé

## Recommandation

Le prochain lot d'implémentation devrait être :

1. ajouter un chargeur de scénario Markdown + schéma zod
2. générer le catalogue actuel à partir du Markdown
3. migrer d'abord quelques scénarios simples
4. ajouter la prise en charge générique des pièces jointes du bus QA
5. afficher une image inline dans l'UI QA
6. puis étendre à l'audio et à la vidéo

C'est le plus petit chemin qui prouve les deux objectifs :

- QA générique définie en Markdown
- surfaces de fausse messagerie plus riches

## Questions ouvertes

- faut-il autoriser dans les fichiers de scénario des modèles de prompt Markdown embarqués avec interpolation de variables
- faut-il que setup/cleanup soient des sections nommées ou simplement des listes d'actions ordonnées
- faut-il que les références d'artefacts soient fortement typées dans le schéma ou basées sur des chaînes
- faut-il que les gestionnaires personnalisés vivent dans un seul registre ou dans des registres par surface
- faut-il que le fichier de compatibilité JSON généré reste versionné pendant la migration
