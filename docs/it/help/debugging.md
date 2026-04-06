---
read_when:
    - Devi ispezionare l'output grezzo del modello per rilevare fuoriuscite del ragionamento
    - Vuoi eseguire il Gateway in modalità watch durante l'iterazione
    - Hai bisogno di un flusso di lavoro di debug ripetibile
summary: 'Strumenti di debug: modalità watch, stream grezzi del modello e tracciamento della fuoriuscita del ragionamento'
title: Debugging
x-i18n:
    generated_at: "2026-04-06T03:07:32Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4bc72e8d6cad3a1acaad066f381c82309583fabf304c589e63885f2685dc704e
    source_path: help/debugging.md
    workflow: 15
---

# Debugging

Questa pagina copre gli strumenti di supporto per il debug dell'output in streaming, soprattutto quando un
provider mescola il ragionamento nel testo normale.

## Override di debug del runtime

Usa `/debug` in chat per impostare override di configurazione **solo runtime** (in memoria, non su disco).
`/debug` è disabilitato per impostazione predefinita; abilitalo con `commands.debug: true`.
Questo è utile quando devi attivare o disattivare impostazioni poco comuni senza modificare `openclaw.json`.

Esempi:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug unset messages.responsePrefix
/debug reset
```

`/debug reset` cancella tutti gli override e torna alla configurazione su disco.

## Modalità watch del Gateway

Per iterare rapidamente, esegui il gateway sotto il file watcher:

```bash
pnpm gateway:watch
```

Questo corrisponde a:

```bash
node scripts/watch-node.mjs gateway --force
```

Il watcher riavvia sui file rilevanti per la build sotto `src/`, sui file sorgente delle estensioni,
sui metadati `package.json` e `openclaw.plugin.json` delle estensioni, su `tsconfig.json`,
`package.json` e `tsdown.config.ts`. Le modifiche ai metadati delle estensioni riavviano il
gateway senza forzare una ricostruzione `tsdown`; le modifiche a sorgenti e configurazione ricostruiscono comunque prima `dist`.

Aggiungi eventuali flag CLI del gateway dopo `gateway:watch` e verranno passati a ogni
riavvio. Rieseguire lo stesso comando watch per lo stesso set di repo/flag ora
sostituisce il watcher precedente invece di lasciare processi watcher parent duplicati.

## Profilo dev + gateway dev (--dev)

Usa il profilo dev per isolare lo stato e avviare una configurazione sicura e usa e getta per il
debug. Esistono **due** flag `--dev`:

- **`--dev` globale (profilo):** isola lo stato sotto `~/.openclaw-dev` e
  imposta per default la porta del gateway su `19001` (anche le porte derivate si spostano con essa).
- **`gateway --dev`:** indica al Gateway di creare automaticamente una configurazione + workspace predefiniti
  se mancanti (e di saltare `BOOTSTRAP.md`).

Flusso consigliato (profilo dev + bootstrap dev):

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

Se non hai ancora un'installazione globale, esegui la CLI tramite `pnpm openclaw ...`.

Cosa fa:

1. **Isolamento del profilo** (`--dev` globale)
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001` (browser/canvas si spostano di conseguenza)

2. **Bootstrap dev** (`gateway --dev`)
   - Scrive una configurazione minima se mancante (`gateway.mode=local`, bind loopback).
   - Imposta `agent.workspace` sul workspace dev.
   - Imposta `agent.skipBootstrap=true` (niente `BOOTSTRAP.md`).
   - Inizializza i file del workspace se mancanti:
     `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`.
   - Identità predefinita: **C3‑PO** (droide protocollare).
   - Salta i provider di canale in modalità dev (`OPENCLAW_SKIP_CHANNELS=1`).

Flusso di reset (nuovo avvio pulito):

```bash
pnpm gateway:dev:reset
```

Nota: `--dev` è un flag di profilo **globale** e viene intercettato da alcuni runner.
Se devi esplicitarlo, usa la forma con variabile env:

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

`--reset` cancella configurazione, credenziali, sessioni e workspace dev (usando
`trash`, non `rm`), poi ricrea la configurazione dev predefinita.

Suggerimento: se è già in esecuzione un gateway non dev (launchd/systemd), arrestalo prima:

```bash
openclaw gateway stop
```

## Logging dello stream grezzo (OpenClaw)

OpenClaw può registrare lo **stream grezzo dell'assistente** prima di qualsiasi filtro/formattazione.
Questo è il modo migliore per vedere se il ragionamento arriva come delta di testo semplice
(o come blocchi di thinking separati).

Abilitalo tramite CLI:

```bash
pnpm gateway:watch --raw-stream
```

Override opzionale del percorso:

```bash
pnpm gateway:watch --raw-stream --raw-stream-path ~/.openclaw/logs/raw-stream.jsonl
```

Variabili env equivalenti:

```bash
OPENCLAW_RAW_STREAM=1
OPENCLAW_RAW_STREAM_PATH=~/.openclaw/logs/raw-stream.jsonl
```

File predefinito:

`~/.openclaw/logs/raw-stream.jsonl`

## Logging dei chunk grezzi (pi-mono)

Per acquisire i **chunk grezzi compatibili con OpenAI** prima che vengano analizzati in blocchi,
pi-mono espone un logger separato:

```bash
PI_RAW_STREAM=1
```

Percorso opzionale:

```bash
PI_RAW_STREAM_PATH=~/.pi-mono/logs/raw-openai-completions.jsonl
```

File predefinito:

`~/.pi-mono/logs/raw-openai-completions.jsonl`

> Nota: questo viene emesso solo dai processi che usano il
> provider `openai-completions` di pi-mono.

## Note di sicurezza

- I log degli stream grezzi possono includere prompt completi, output degli strumenti e dati utente.
- Mantieni i log in locale ed eliminali dopo il debug.
- Se condividi i log, rimuovi prima secret e PII.
