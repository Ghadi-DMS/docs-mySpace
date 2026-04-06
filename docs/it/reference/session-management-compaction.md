---
read_when:
    - Hai bisogno di fare debug degli id sessione, del JSONL della trascrizione o dei campi di sessions.json
    - Stai modificando il comportamento della compattazione automatica o aggiungendo operazioni preliminari “pre-compaction”
    - Vuoi implementare flush della memoria o turni di sistema silenziosi
summary: 'Approfondimento: store delle sessioni + trascrizioni, ciclo di vita e interni della compattazione (automatica)'
title: Approfondimento sulla gestione delle sessioni
x-i18n:
    generated_at: "2026-04-06T03:12:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: e0d8c2d30be773eac0424f7a4419ab055fdd50daac8bc654e7d250c891f2c3b8
    source_path: reference/session-management-compaction.md
    workflow: 15
---

# Gestione delle sessioni e compattazione (approfondimento)

Questo documento spiega come OpenClaw gestisce le sessioni end-to-end:

- **Instradamento della sessione** (come i messaggi in ingresso vengono mappati a un `sessionKey`)
- **Store delle sessioni** (`sessions.json`) e cosa tiene traccia
- **Persistenza della trascrizione** (`*.jsonl`) e la sua struttura
- **Igiene della trascrizione** (fixup specifici del provider prima delle esecuzioni)
- **Limiti di contesto** (finestra di contesto rispetto ai token tracciati)
- **Compattazione** (manuale + automatica) e dove agganciare il lavoro pre-compaction
- **Operazioni silenziose di housekeeping** (ad esempio scritture in memoria che non dovrebbero produrre output visibile all'utente)

Se vuoi prima una panoramica di livello più alto, inizia da:

- [/concepts/session](/it/concepts/session)
- [/concepts/compaction](/it/concepts/compaction)
- [/concepts/memory](/it/concepts/memory)
- [/concepts/memory-search](/it/concepts/memory-search)
- [/concepts/session-pruning](/it/concepts/session-pruning)
- [/reference/transcript-hygiene](/it/reference/transcript-hygiene)

---

## Fonte di verità: il Gateway

OpenClaw è progettato attorno a un singolo **processo Gateway** che possiede lo stato delle sessioni.

- Le UI (app macOS, web Control UI, TUI) dovrebbero interrogare il Gateway per gli elenchi delle sessioni e i conteggi dei token.
- In modalità remota, i file di sessione si trovano sull'host remoto; “controllare i file del tuo Mac locale” non rifletterà ciò che il Gateway sta usando.

---

## Due livelli di persistenza

OpenClaw persiste le sessioni in due livelli:

1. **Store delle sessioni (`sessions.json`)**
   - Mappa chiave/valore: `sessionKey -> SessionEntry`
   - Piccolo, mutabile, sicuro da modificare (o da cui eliminare voci)
   - Tiene traccia dei metadati della sessione (id sessione corrente, ultima attività, toggle, contatori token, ecc.)

2. **Trascrizione (`<sessionId>.jsonl`)**
   - Trascrizione append-only con struttura ad albero (le voci hanno `id` + `parentId`)
   - Memorizza la conversazione effettiva + tool call + riepiloghi di compattazione
   - Usata per ricostruire il contesto del modello per i turni futuri

---

## Posizioni su disco

Per agent, sull'host Gateway:

- Store: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Trascrizioni: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
  - Sessioni dei topic Telegram: `.../<sessionId>-topic-<threadId>.jsonl`

OpenClaw le risolve tramite `src/config/sessions.ts`.

---

## Manutenzione dello store e controlli del disco

La persistenza delle sessioni ha controlli di manutenzione automatica (`session.maintenance`) per `sessions.json` e gli artefatti delle trascrizioni:

- `mode`: `warn` (predefinito) oppure `enforce`
- `pruneAfter`: soglia di età per potare le voci stale (predefinito `30d`)
- `maxEntries`: limite delle voci in `sessions.json` (predefinito `500`)
- `rotateBytes`: ruota `sessions.json` quando è troppo grande (predefinito `10mb`)
- `resetArchiveRetention`: retention per gli archivi di trascrizione `*.reset.<timestamp>` (predefinita: uguale a `pruneAfter`; `false` disabilita la pulizia)
- `maxDiskBytes`: budget facoltativo per la directory delle sessioni
- `highWaterBytes`: target facoltativo dopo la pulizia (predefinito `80%` di `maxDiskBytes`)

Ordine di applicazione per la pulizia del budget disco (`mode: "enforce"`):

1. Rimuovi prima gli artefatti delle trascrizioni archiviati o orfani più vecchi.
2. Se il target è ancora superato, espelli le voci di sessione più vecchie e i relativi file di trascrizione.
3. Continua finché l'utilizzo non è pari o inferiore a `highWaterBytes`.

In `mode: "warn"`, OpenClaw segnala le possibili espulsioni ma non modifica lo store/i file.

Esegui la manutenzione su richiesta:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

---

## Sessioni cron e log di esecuzione

Anche le esecuzioni cron isolate creano voci di sessione/trascrizioni e hanno controlli di retention dedicati:

- `cron.sessionRetention` (predefinito `24h`) elimina dallo store delle sessioni le vecchie sessioni delle esecuzioni cron isolate (`false` disabilita).
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` potano i file `~/.openclaw/cron/runs/<jobId>.jsonl` (predefiniti: `2_000_000` byte e `2000` righe).

---

## Chiavi di sessione (`sessionKey`)

Un `sessionKey` identifica _in quale contenitore di conversazione_ ti trovi (instradamento + isolamento).

Pattern comuni:

- Chat principale/diretta (per agent): `agent:<agentId>:<mainKey>` (predefinito `main`)
- Gruppo: `agent:<agentId>:<channel>:group:<id>`
- Stanza/canale (Discord/Slack): `agent:<agentId>:<channel>:channel:<id>` oppure `...:room:<id>`
- Cron: `cron:<job.id>`
- Webhook: `hook:<uuid>` (salvo override)

Le regole canoniche sono documentate in [/concepts/session](/it/concepts/session).

---

## ID sessione (`sessionId`)

Ogni `sessionKey` punta a un `sessionId` corrente (il file di trascrizione che continua la conversazione).

Regole pratiche:

- **Reset** (`/new`, `/reset`) crea un nuovo `sessionId` per quel `sessionKey`.
- **Reset giornaliero** (predefinito alle 4:00 ora locale sull'host Gateway) crea un nuovo `sessionId` al messaggio successivo dopo il confine di reset.
- **Scadenza per inattività** (`session.reset.idleMinutes` o legacy `session.idleMinutes`) crea un nuovo `sessionId` quando arriva un messaggio dopo la finestra di inattività. Quando sono configurati sia giornaliero sia inattività, vince quello che scade per primo.
- **Guardrail fork del parent thread** (`session.parentForkMaxTokens`, predefinito `100000`) salta il forking della trascrizione parent quando la sessione parent è già troppo grande; il nuovo thread parte da zero. Imposta `0` per disabilitare.

Dettaglio implementativo: la decisione avviene in `initSessionState()` in `src/auto-reply/reply/session.ts`.

---

## Schema dello store delle sessioni (`sessions.json`)

Il tipo valore dello store è `SessionEntry` in `src/config/sessions.ts`.

Campi chiave (non esaustivo):

- `sessionId`: id della trascrizione corrente (il nome file viene derivato da questo salvo se è impostato `sessionFile`)
- `updatedAt`: timestamp dell'ultima attività
- `sessionFile`: override facoltativo del percorso della trascrizione
- `chatType`: `direct | group | room` (aiuta UI e criteri di invio)
- `provider`, `subject`, `room`, `space`, `displayName`: metadati per etichette di gruppi/canali
- Toggle:
  - `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`
  - `sendPolicy` (override per sessione)
- Selezione del modello:
  - `providerOverride`, `modelOverride`, `authProfileOverride`
- Contatori token (best-effort / dipendenti dal provider):
  - `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: quante volte la compattazione automatica è stata completata per questa chiave di sessione
- `memoryFlushAt`: timestamp dell'ultimo flush della memoria pre-compaction
- `memoryFlushCompactionCount`: conteggio di compattazione quando è stato eseguito l'ultimo flush

Lo store è sicuro da modificare, ma il Gateway è l'autorità: può riscrivere o reidratare le voci durante l'esecuzione delle sessioni.

---

## Struttura della trascrizione (`*.jsonl`)

Le trascrizioni sono gestite da `@mariozechner/pi-coding-agent` tramite `SessionManager`.

Il file è JSONL:

- Prima riga: intestazione della sessione (`type: "session"`, include `id`, `cwd`, `timestamp`, `parentSession` facoltativo)
- Poi: voci di sessione con `id` + `parentId` (albero)

Tipi di voce rilevanti:

- `message`: messaggi user/assistant/toolResult
- `custom_message`: messaggi iniettati dall'estensione che _entrano_ nel contesto del modello (possono essere nascosti dalla UI)
- `custom`: stato dell'estensione che _non_ entra nel contesto del modello
- `compaction`: riepilogo di compattazione persistito con `firstKeptEntryId` e `tokensBefore`
- `branch_summary`: riepilogo persistito durante la navigazione di un ramo dell'albero

OpenClaw intenzionalmente **non** “corregge” le trascrizioni; il Gateway usa `SessionManager` per leggerle/scriverle.

---

## Finestre di contesto vs token tracciati

Contano due concetti diversi:

1. **Finestra di contesto del modello**: limite rigido per modello (token visibili al modello)
2. **Contatori dello store delle sessioni**: statistiche scorrevoli scritte in `sessions.json` (usate per /status e dashboard)

Se stai regolando i limiti:

- La finestra di contesto proviene dal catalogo del modello (e può essere sovrascritta tramite config).
- `contextTokens` nello store è un valore di stima/reporting di runtime; non trattarlo come una garanzia rigorosa.

Per maggiori dettagli, vedi [/token-use](/it/reference/token-use).

---

## Compattazione: cos'è

La compattazione riassume la conversazione più vecchia in una voce `compaction` persistita nella trascrizione e mantiene intatti i messaggi recenti.

Dopo la compattazione, i turni futuri vedono:

- Il riepilogo di compattazione
- I messaggi successivi a `firstKeptEntryId`

La compattazione è **persistente** (a differenza della potatura della sessione). Vedi [/concepts/session-pruning](/it/concepts/session-pruning).

## Confini dei blocchi di compattazione e pairing dei tool

Quando OpenClaw divide una lunga trascrizione in blocchi di compattazione, mantiene
accoppiate le tool call dell'assistant con le corrispondenti voci `toolResult`.

- Se la divisione della quota di token cade tra una tool call e il suo risultato, OpenClaw
  sposta il confine al messaggio dell'assistant con la tool call invece di separare
  la coppia.
- Se un blocco finale di tool-result altrimenti spingerebbe il blocco oltre il target,
  OpenClaw conserva quel blocco tool pendente e mantiene intatta la coda non
  riassunta.
- I blocchi di tool-call interrotti/in errore non tengono aperta una divisione pendente.

---

## Quando avviene la compattazione automatica (runtime Pi)

Nell'agent Pi embedded, la compattazione automatica si attiva in due casi:

1. **Recupero da overflow**: il modello restituisce un errore di overflow del contesto
   (`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model`, `ollama error: context length
exceeded` e varianti simili dipendenti dal provider) → compatta → ritenta.
2. **Manutenzione della soglia**: dopo un turno riuscito, quando:

`contextTokens > contextWindow - reserveTokens`

Dove:

- `contextWindow` è la finestra di contesto del modello
- `reserveTokens` è lo spazio riservato per prompt + prossimo output del modello

Queste sono semantiche del runtime Pi (OpenClaw consuma gli eventi, ma è Pi a decidere quando compattare).

---

## Impostazioni di compattazione (`reserveTokens`, `keepRecentTokens`)

Le impostazioni di compattazione di Pi si trovano nelle impostazioni Pi:

```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,
    keepRecentTokens: 20000,
  },
}
```

OpenClaw applica anche una soglia minima di sicurezza per le esecuzioni embedded:

- Se `compaction.reserveTokens < reserveTokensFloor`, OpenClaw lo aumenta.
- La soglia minima predefinita è `20000` token.
- Imposta `agents.defaults.compaction.reserveTokensFloor: 0` per disabilitare la soglia.
- Se è già più alto, OpenClaw lo lascia invariato.

Perché: lasciare spazio sufficiente per operazioni di “housekeeping” multi-turno (come le scritture in memoria) prima che la compattazione diventi inevitabile.

Implementazione: `ensurePiCompactionReserveTokens()` in `src/agents/pi-settings.ts`
(chiamato da `src/agents/pi-embedded-runner.ts`).

---

## Superfici visibili all'utente

Puoi osservare la compattazione e lo stato della sessione tramite:

- `/status` (in qualsiasi sessione chat)
- `openclaw status` (CLI)
- `openclaw sessions` / `sessions --json`
- Modalità verbose: `🧹 Auto-compaction complete` + conteggio della compattazione

---

## Housekeeping silenzioso (`NO_REPLY`)

OpenClaw supporta turni “silenziosi” per attività in background in cui l'utente non dovrebbe vedere output intermedi.

Convenzione:

- L'assistant inizia il suo output con l'esatto token silenzioso `NO_REPLY` /
  `no_reply` per indicare “non inviare una risposta all'utente”.
- OpenClaw lo rimuove/lo sopprime nel livello di consegna.
- La soppressione dell'esatto token silenzioso è case-insensitive, quindi `NO_REPLY` e
  `no_reply` contano entrambi quando l'intero payload è solo il token silenzioso.
- Questo è solo per veri turni in background/senza consegna; non è una scorciatoia per
  normali richieste utente attuabili.

A partire da `2026.1.10`, OpenClaw sopprime anche lo **streaming draft/typing** quando un
blocco parziale inizia con `NO_REPLY`, così le operazioni silenziose non fanno trapelare output parziale a metà turno.

---

## "Memory flush" pre-compaction (implementato)

Obiettivo: prima che avvenga la compattazione automatica, eseguire un turno agentico silenzioso che scriva
stato durevole su disco (ad esempio `memory/YYYY-MM-DD.md` nel workspace dell'agent) così la compattazione non può
cancellare contesto critico.

OpenClaw usa l'approccio del **flush pre-soglia**:

1. Monitora l'utilizzo del contesto della sessione.
2. Quando supera una “soglia morbida” (inferiore alla soglia di compattazione di Pi), esegue una direttiva silenziosa
   “scrivi ora in memoria” verso l'agent.
3. Usa l'esatto token silenzioso `NO_REPLY` / `no_reply` così l'utente non vede
   nulla.

Config (`agents.defaults.compaction.memoryFlush`):

- `enabled` (predefinito: `true`)
- `softThresholdTokens` (predefinito: `4000`)
- `prompt` (messaggio utente per il turno di flush)
- `systemPrompt` (system prompt aggiuntivo aggiunto per il turno di flush)

Note:

- Il prompt/system prompt predefinito include un suggerimento `NO_REPLY` per sopprimere
  la consegna.
- Il flush viene eseguito una volta per ciclo di compattazione (tracciato in `sessions.json`).
- Il flush viene eseguito solo per sessioni Pi embedded.
- Il flush viene saltato quando il workspace della sessione è in sola lettura (`workspaceAccess: "ro"` oppure `"none"`).
- Vedi [Memory](/it/concepts/memory) per il layout dei file nel workspace e i pattern di scrittura.

Pi espone anche un hook `session_before_compact` nell'API dell'estensione, ma oggi la logica di
flush di OpenClaw vive lato Gateway.

---

## Checklist di risoluzione dei problemi

- Chiave di sessione sbagliata? Parti da [/concepts/session](/it/concepts/session) e conferma il `sessionKey` in `/status`.
- Mancata corrispondenza store vs trascrizione? Conferma l'host Gateway e il percorso dello store da `openclaw status`.
- Spam di compattazione? Controlla:
  - finestra di contesto del modello (troppo piccola)
  - impostazioni di compattazione (`reserveTokens` troppo alto per la finestra del modello può causare compattazione anticipata)
  - crescita eccessiva dei tool-result: abilita/regola la potatura della sessione
- Perdite nei turni silenziosi? Conferma che la risposta inizi con `NO_REPLY` (token esatto case-insensitive) e che tu stia usando una build che include la correzione della soppressione dello streaming.
