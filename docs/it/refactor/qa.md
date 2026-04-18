---
x-i18n:
    generated_at: "2026-04-18T08:05:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: dbb2c70c82da7f6f12d90e25666635ff4147c52e8a94135e902d1de4f5cbccca
    source_path: refactor/qa.md
    workflow: 15
---

# Refactor QA

Stato: migrazione fondamentale completata.

## Obiettivo

Spostare il QA di OpenClaw da un modello a definizione divisa a un'unica fonte di verità:

- metadati dello scenario
- prompt inviati al modello
- setup e teardown
- logica dell'harness
- asserzioni e criteri di successo
- artefatti e suggerimenti per il report

Lo stato finale desiderato è un harness QA generico che carica file di definizione degli scenari potenti invece di codificare in TypeScript la maggior parte del comportamento.

## Stato attuale

La fonte primaria di verità ora si trova in `qa/scenarios/index.md` più un file per
scenario sotto `qa/scenarios/<theme>/*.md`.

Implementato:

- `qa/scenarios/index.md`
  - metadati canonici del pacchetto QA
  - identità dell'operatore
  - missione di avvio
- `qa/scenarios/<theme>/*.md`
  - un file markdown per scenario
  - metadati dello scenario
  - binding degli handler
  - configurazione di esecuzione specifica dello scenario
- `extensions/qa-lab/src/scenario-catalog.ts`
  - parser del pacchetto markdown + validazione zod
- `extensions/qa-lab/src/qa-agent-bootstrap.ts`
  - rendering del piano a partire dal pacchetto markdown
- `extensions/qa-lab/src/qa-agent-workspace.ts`
  - inizializza file di compatibilità generati più `QA_SCENARIOS.md`
- `extensions/qa-lab/src/suite.ts`
  - seleziona gli scenari eseguibili tramite binding degli handler definiti nel markdown
- Protocollo QA bus + UI
  - allegati inline generici per il rendering di immagini/video/audio/file

Superfici ancora divise:

- `extensions/qa-lab/src/suite.ts`
  - possiede ancora la maggior parte della logica eseguibile degli handler personalizzati
- `extensions/qa-lab/src/report.ts`
  - deriva ancora la struttura del report dagli output runtime

Quindi la divisione della fonte di verità è stata risolta, ma l'esecuzione è ancora in gran parte basata su handler invece di essere completamente dichiarativa.

## Che aspetto ha davvero la superficie degli scenari

Leggendo la suite attuale si vedono alcune classi distinte di scenari.

### Interazione semplice

- baseline del canale
- baseline DM
- follow-up in thread
- cambio modello
- completamento dell'approvazione
- reaction/edit/delete

### Mutazione di configurazione e runtime

- config patch skill disable
- config apply restart wake-up
- config restart capability flip
- runtime inventory drift check

### Asserzioni su filesystem e repo

- source/docs discovery report
- build Lobster Invaders
- generated image artifact lookup

### Orchestrazione della memoria

- memory recall
- memory tools in channel context
- memory failure fallback
- session memory ranking
- thread memory isolation
- memory dreaming sweep

### Integrazione di strumenti e Plugin

- MCP plugin-tools call
- skill visibility
- skill hot install
- native image generation
- image roundtrip
- image understanding from attachment

### Multi-turno e multi-attore

- subagent handoff
- subagent fanout synthesis
- restart recovery style flows

Queste categorie sono importanti perché guidano i requisiti del DSL. Un elenco piatto di prompt + testo atteso non basta.

## Direzione

### Unica fonte di verità

Usare `qa/scenarios/index.md` più `qa/scenarios/<theme>/*.md` come
fonte di verità creata manualmente.

Il pacchetto deve rimanere:

- leggibile dagli esseri umani in review
- parsabile dalle macchine
- abbastanza ricco da guidare:
  - esecuzione della suite
  - bootstrap dell'area di lavoro QA
  - metadati della UI di QA Lab
  - prompt di docs/discovery
  - generazione dei report

### Formato di authoring preferito

Usare markdown come formato di livello superiore, con YAML strutturato al suo interno.

Forma consigliata:

- frontmatter YAML
  - id
  - title
  - surface
  - tags
  - riferimenti docs
  - riferimenti code
  - override di model/provider
  - prerequisiti
- sezioni in prosa
  - objective
  - notes
  - debugging hints
- blocchi YAML fenced
  - setup
  - steps
  - assertions
  - cleanup

Questo offre:

- migliore leggibilità nelle PR rispetto a enormi file JSON
- contesto più ricco rispetto a puro YAML
- parsing rigoroso e validazione zod

Il JSON grezzo è accettabile solo come forma intermedia generata.

## Forma proposta del file di scenario

Esempio:

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

## Capacità del runner che il DSL deve coprire

In base alla suite attuale, il runner generico ha bisogno di più della semplice esecuzione dei prompt.

### Azioni di ambiente e setup

- `bus.reset`
- `gateway.waitHealthy`
- `channel.waitReady`
- `session.create`
- `thread.create`
- `workspace.writeSkill`

### Azioni di turno dell'agente

- `agent.send`
- `agent.wait`
- `bus.injectInbound`
- `bus.injectOutbound`

### Azioni di configurazione e runtime

- `config.get`
- `config.patch`
- `config.apply`
- `gateway.restart`
- `tools.effective`
- `skills.status`

### Azioni su file e artefatti

- `file.write`
- `file.read`
- `file.delete`
- `file.touchTime`
- `artifact.captureGeneratedImage`
- `artifact.capturePath`

### Azioni su memoria e Cron

- `memory.indexForce`
- `memory.searchCli`
- `doctor.memory.status`
- `cron.list`
- `cron.run`
- `cron.waitCompletion`
- `sessionTranscript.write`

### Azioni MCP

- `mcp.callTool`

### Asserzioni

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

## Variabili e riferimenti ad artefatti

Il DSL deve supportare output salvati e riferimenti successivi.

Esempi dalla suite attuale:

- creare un thread, poi riusare `threadId`
- creare una sessione, poi riusare `sessionKey`
- generare un'immagine, poi allegare il file nel turno successivo
- generare una stringa marcatore di wake, poi verificare che compaia più tardi

Capacità necessarie:

- `saveAs`
- `${vars.name}`
- `${artifacts.name}`
- riferimenti tipizzati per percorsi, chiavi sessione, id thread, marcatori, output degli strumenti

Senza supporto per le variabili, l'harness continuerà a far trapelare la logica degli scenari in TypeScript.

## Cosa dovrebbe restare come via di fuga

Un runner completamente puro e dichiarativo non è realistico nella fase 1.

Alcuni scenari sono intrinsecamente pesanti dal punto di vista dell'orchestrazione:

- memory dreaming sweep
- config apply restart wake-up
- config restart capability flip
- risoluzione dell'artefatto immagine generato per timestamp/percorso
- valutazione del discovery-report

Questi dovrebbero usare per ora handler personalizzati espliciti.

Regola consigliata:

- 85-90% dichiarativo
- `customHandler` espliciti per la parte difficile restante
- solo handler personalizzati nominati e documentati
- nessun codice inline anonimo nel file di scenario

Questo mantiene pulito il motore generico e consente comunque di progredire.

## Modifica architetturale

### Attuale

Il markdown degli scenari è già la fonte di verità per:

- esecuzione della suite
- file bootstrap dell'area di lavoro
- catalogo degli scenari nella UI di QA Lab
- metadati del report
- prompt di discovery

Compatibilità generata:

- l'area di lavoro inizializzata include ancora `QA_KICKOFF_TASK.md`
- l'area di lavoro inizializzata include ancora `QA_SCENARIO_PLAN.md`
- l'area di lavoro inizializzata ora include anche `QA_SCENARIOS.md`

## Piano di refactor

### Fase 1: loader e schema

Completata.

- aggiunto `qa/scenarios/index.md`
- suddivisi gli scenari in `qa/scenarios/<theme>/*.md`
- aggiunto parser per contenuti pack markdown YAML con nome
- validato con zod
- spostati i consumer al pack parsato
- rimossi `qa/seed-scenarios.json` e `qa/QA_KICKOFF_TASK.md` a livello repo

### Fase 2: motore generico

- dividere `extensions/qa-lab/src/suite.ts` in:
  - loader
  - engine
  - registry delle azioni
  - registry delle asserzioni
  - handler personalizzati
- mantenere le funzioni helper esistenti come operazioni del motore

Deliverable:

- il motore esegue scenari dichiarativi semplici

Iniziare con scenari che sono soprattutto prompt + attesa + asserzione:

- threaded follow-up
- image understanding from attachment
- skill visibility and invocation
- channel baseline

Deliverable:

- primi scenari reali definiti in markdown distribuiti tramite il motore generico

### Fase 4: migrare gli scenari di complessità media

- image generation roundtrip
- memory tools in channel context
- session memory ranking
- subagent handoff
- subagent fanout synthesis

Deliverable:

- variabili, artefatti, asserzioni sugli strumenti, asserzioni sul request-log comprovate

### Fase 5: mantenere gli scenari difficili su handler personalizzati

- memory dreaming sweep
- config apply restart wake-up
- config restart capability flip
- runtime inventory drift

Deliverable:

- stesso formato di authoring, ma con blocchi custom-step espliciti dove necessario

### Fase 6: eliminare la mappa di scenari hardcoded

Una volta che la copertura del pack è abbastanza buona:

- rimuovere la maggior parte del branching TypeScript specifico per scenario da `extensions/qa-lab/src/suite.ts`

## Supporto Fake Slack / Rich Media

L'attuale QA bus è incentrato sul testo.

File rilevanti:

- `extensions/qa-channel/src/protocol.ts`
- `extensions/qa-lab/src/bus-state.ts`
- `extensions/qa-lab/src/bus-queries.ts`
- `extensions/qa-lab/src/bus-server.ts`
- `extensions/qa-lab/web/src/ui-render.ts`

Oggi il QA bus supporta:

- testo
- reaction
- thread

Non modella ancora gli allegati multimediali inline.

### Contratto di trasporto necessario

Aggiungere un modello generico di allegato QA bus:

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

Poi aggiungere `attachments?: QaBusAttachment[]` a:

- `QaBusMessage`
- `QaBusInboundMessageInput`
- `QaBusOutboundMessageInput`

### Perché prima generico

Non costruire un modello media solo per Slack.

Invece:

- un unico modello di trasporto QA generico
- più renderer sopra di esso
  - l'attuale chat di QA Lab
  - una futura finta web Slack
  - qualsiasi altra vista di trasporto fittizio

Questo evita logica duplicata e permette agli scenari media di restare agnostici rispetto al trasporto.

### Lavoro UI necessario

Aggiornare la UI QA per renderizzare:

- anteprima immagine inline
- player audio inline
- player video inline
- chip allegato file

L'attuale UI può già renderizzare thread e reaction, quindi il rendering degli allegati dovrebbe potersi stratificare sullo stesso modello di card messaggio.

### Lavoro sugli scenari abilitato dal trasporto media

Una volta che gli allegati scorrono attraverso il QA bus, possiamo aggiungere scenari fake-chat più ricchi:

- risposta con immagine inline in fake Slack
- comprensione di allegato audio
- comprensione di allegato video
- ordinamento misto degli allegati
- risposta in thread con media mantenuti

## Raccomandazione

Il prossimo blocco di implementazione dovrebbe essere:

1. aggiungere loader di scenari markdown + schema zod
2. generare il catalogo attuale dal markdown
3. migrare prima alcuni scenari semplici
4. aggiungere supporto generico agli allegati del QA bus
5. renderizzare immagini inline nella UI QA
6. poi estendere ad audio e video

Questo è il percorso più piccolo che dimostra entrambi gli obiettivi:

- QA generico definito in markdown
- superfici di messaggistica fittizia più ricche

## Domande aperte

- se i file di scenario debbano consentire template di prompt markdown incorporati con interpolazione di variabili
- se setup/cleanup debbano essere sezioni nominate o semplicemente elenchi ordinati di azioni
- se i riferimenti agli artefatti debbano essere fortemente tipizzati nello schema o basati su stringhe
- se gli handler personalizzati debbano trovarsi in un unico registry o in registry per superficie
- se il file di compatibilità JSON generato debba rimanere sotto controllo di versione durante la migrazione
