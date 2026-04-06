---
read_when:
    - Eseguire coding harness tramite ACP
    - Configurare sessioni ACP associate a conversazioni sui canali di messaggistica
    - Associare una conversazione di un canale di messaggistica a una sessione ACP persistente
    - Risolvere problemi del backend ACP e del wiring del plugin
    - Usare i comandi /acp dalla chat
summary: Usa sessioni runtime ACP per Codex, Claude Code, Cursor, Gemini CLI, OpenClaw ACP e altri agent harness
title: Agenti ACP
x-i18n:
    generated_at: "2026-04-06T03:13:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 302f3fe25b1ffe0576592b6e0ad9e8a5781fa5702b31d508d9ba8908f7df33bd
    source_path: tools/acp-agents.md
    workflow: 15
---

# Agenti ACP

Le sessioni [Agent Client Protocol (ACP)](https://agentclientprotocol.com/) consentono a OpenClaw di eseguire harness di coding esterni (per esempio Pi, Claude Code, Codex, Cursor, Copilot, OpenClaw ACP, OpenCode, Gemini CLI e altri harness ACPX supportati) tramite un plugin backend ACP.

Se chiedi a OpenClaw in linguaggio naturale di "eseguire questo in Codex" o "avviare Claude Code in un thread", OpenClaw dovrebbe instradare quella richiesta al runtime ACP (non al runtime nativo dei sub-agent). Ogni spawn di sessione ACP viene tracciato come [attività in background](/it/automation/tasks).

Se vuoi che Codex o Claude Code si connettano direttamente come client MCP esterni
a conversazioni di canale OpenClaw esistenti, usa [`openclaw mcp serve`](/cli/mcp)
invece di ACP.

## Quale pagina mi serve?

Ci sono tre superfici vicine che è facile confondere:

| Vuoi...                                                                            | Usa questo                 | Note                                                                                                             |
| ---------------------------------------------------------------------------------- | -------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Eseguire Codex, Claude Code, Gemini CLI o un altro harness esterno _tramite_ OpenClaw | Questa pagina: agenti ACP  | Sessioni associate alla chat, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, attività in background, controlli runtime |
| Esporre una sessione OpenClaw Gateway _come_ server ACP per un editor o un client | [`openclaw acp`](/cli/acp) | Modalità bridge. L'IDE/client parla ACP con OpenClaw tramite stdio/WebSocket                                     |

## Funziona subito?

Di solito sì.

- Le installazioni nuove includono ora il plugin runtime incluso `acpx` abilitato per impostazione predefinita.
- Il plugin incluso `acpx` preferisce il proprio binario `acpx` locale al plugin e bloccato a una versione precisa.
- All'avvio, OpenClaw verifica quel binario e lo autoripara se necessario.
- Inizia con `/acp doctor` se vuoi un rapido controllo di disponibilità.

Cosa può ancora succedere al primo utilizzo:

- Un adapter del harness di destinazione può essere recuperato on demand con `npx` la prima volta che usi quel harness.
- L'autenticazione del vendor deve comunque esistere sull'host per quel harness.
- Se l'host non ha accesso npm/rete, i recuperi degli adapter al primo avvio possono fallire finché le cache non vengono preriscaldate o l'adapter non viene installato in un altro modo.

Esempi:

- `/acp spawn codex`: OpenClaw dovrebbe essere pronto per inizializzare `acpx`, ma l'adapter ACP di Codex potrebbe comunque aver bisogno di un recupero al primo avvio.
- `/acp spawn claude`: stessa situazione per l'adapter ACP di Claude, più l'autenticazione lato Claude su quell'host.

## Flusso rapido per operatori

Usa questo quando vuoi un runbook pratico di `/acp`:

1. Genera una sessione:
   - `/acp spawn codex --bind here`
   - `/acp spawn codex --mode persistent --thread auto`
2. Lavora nella conversazione o nel thread associato (oppure punta esplicitamente a quella chiave di sessione).
3. Controlla lo stato runtime:
   - `/acp status`
4. Regola le opzioni runtime se necessario:
   - `/acp model <provider/model>`
   - `/acp permissions <profile>`
   - `/acp timeout <seconds>`
5. Dai una correzione a una sessione attiva senza sostituire il contesto:
   - `/acp steer tighten logging and continue`
6. Interrompi il lavoro:
   - `/acp cancel` (ferma il turno corrente), oppure
   - `/acp close` (chiude la sessione + rimuove le associazioni)

## Avvio rapido per persone

Esempi di richieste in linguaggio naturale:

- "Associa questo canale Discord a Codex."
- "Avvia una sessione Codex persistente in un thread qui e mantienila focalizzata."
- "Esegui questo come sessione ACP Claude Code one-shot e riassumi il risultato."
- "Associa questa chat iMessage a Codex e mantieni i messaggi successivi nello stesso workspace."
- "Usa Gemini CLI per questo compito in un thread, poi mantieni i messaggi successivi nello stesso thread."

Cosa dovrebbe fare OpenClaw:

1. Scegliere `runtime: "acp"`.
2. Risolvere il target harness richiesto (`agentId`, per esempio `codex`).
3. Se viene richiesta l'associazione alla conversazione corrente e il canale attivo la supporta, associare la sessione ACP a quella conversazione.
4. Altrimenti, se viene richiesta l'associazione a un thread e il canale corrente la supporta, associare la sessione ACP al thread.
5. Instradare i messaggi successivi associati alla stessa sessione ACP finché non viene tolto il focus/chiusa/scaduta.

## ACP rispetto ai sub-agent

Usa ACP quando vuoi un runtime harness esterno. Usa i sub-agent quando vuoi esecuzioni delegate native di OpenClaw.

| Area            | Sessione ACP                           | Esecuzione sub-agent                 |
| --------------- | -------------------------------------- | ------------------------------------ |
| Runtime         | Plugin backend ACP (per esempio acpx)  | Runtime nativo sub-agent OpenClaw    |
| Chiave sessione | `agent:<agentId>:acp:<uuid>`           | `agent:<agentId>:subagent:<uuid>`    |
| Comandi principali | `/acp ...`                          | `/subagents ...`                     |
| Strumento di spawn | `sessions_spawn` con `runtime:"acp"` | `sessions_spawn` (runtime predefinito) |

Vedi anche [Sub-agents](/it/tools/subagents).

## Come ACP esegue Claude Code

Per Claude Code tramite ACP, lo stack è:

1. Control plane della sessione ACP di OpenClaw
2. plugin runtime incluso `acpx`
3. adapter ACP Claude
4. meccanismo runtime/sessione lato Claude

Distinzione importante:

- ACP Claude è una sessione harness con controlli ACP, ripresa della sessione, tracciamento delle attività in background e associazione facoltativa a conversazioni/thread.
  Per gli operatori, la regola pratica è:

- se vuoi `/acp spawn`, sessioni associabili, controlli runtime o lavoro persistente con harness: usa ACP

## Sessioni associate

### Associazioni alla conversazione corrente

Usa `/acp spawn <harness> --bind here` quando vuoi che la conversazione corrente diventi un workspace ACP durevole senza creare un thread figlio.

Comportamento:

- OpenClaw continua a possedere il trasporto del canale, l'autenticazione, la sicurezza e la consegna.
- La conversazione corrente viene fissata alla chiave di sessione ACP generata.
- I messaggi successivi in quella conversazione vengono instradati alla stessa sessione ACP.
- `/new` e `/reset` reimpostano sul posto la stessa sessione ACP associata.
- `/acp close` chiude la sessione e rimuove l'associazione alla conversazione corrente.

Cosa significa in pratica:

- `--bind here` mantiene la stessa superficie di chat. Su Discord, il canale corrente resta il canale corrente.
- `--bind here` può comunque creare una nuova sessione ACP se stai avviando nuovo lavoro. L'associazione collega quella sessione alla conversazione corrente.
- `--bind here` non crea da solo un thread figlio Discord o un topic Telegram.
- Il runtime ACP può comunque avere la propria directory di lavoro (`cwd`) o workspace su disco gestito dal backend. Quel workspace runtime è separato dalla superficie di chat e non implica un nuovo thread di messaggistica.
- Se effettui uno spawn verso un altro agente ACP e non passi `--cwd`, OpenClaw eredita per impostazione predefinita il workspace dell'**agente di destinazione**, non quello del richiedente.
- Se il percorso del workspace ereditato manca (`ENOENT`/`ENOTDIR`), OpenClaw ripiega sul cwd predefinito del backend invece di riutilizzare silenziosamente l'albero sbagliato.
- Se il workspace ereditato esiste ma non è accessibile (per esempio `EACCES`), lo spawn restituisce il vero errore di accesso invece di eliminare `cwd`.

Modello mentale:

- superficie di chat: dove le persone continuano a parlare (`canale Discord`, `topic Telegram`, `chat iMessage`)
- sessione ACP: lo stato runtime durevole di Codex/Claude/Gemini verso cui OpenClaw instrada
- thread/topic figlio: una superficie di messaggistica aggiuntiva facoltativa creata solo da `--thread ...`
- workspace runtime: la posizione del filesystem in cui viene eseguito l'harness (`cwd`, checkout del repo, workspace del backend)

Esempi:

- `/acp spawn codex --bind here`: mantieni questa chat, genera o collega una sessione ACP Codex e instrada qui i messaggi futuri verso di essa
- `/acp spawn codex --thread auto`: OpenClaw può creare un thread/topic figlio e associare lì la sessione ACP
- `/acp spawn codex --bind here --cwd /workspace/repo`: stessa associazione alla chat di prima, ma Codex viene eseguito in `/workspace/repo`

Supporto per l'associazione alla conversazione corrente:

- I canali di chat/messaggistica che pubblicizzano il supporto per l'associazione alla conversazione corrente possono usare `--bind here` tramite il percorso condiviso di conversation-binding.
- I canali con semantiche personalizzate di thread/topic possono comunque fornire una canonicalizzazione specifica del canale dietro la stessa interfaccia condivisa.
- `--bind here` significa sempre "associa sul posto la conversazione corrente".
- Le associazioni generiche alla conversazione corrente usano lo store di binding condiviso di OpenClaw e sopravvivono ai normali riavvii del gateway.

Note:

- `--bind here` e `--thread ...` si escludono a vicenda in `/acp spawn`.
- Su Discord, `--bind here` associa sul posto il canale o thread corrente. `spawnAcpSessions` è richiesto solo quando OpenClaw deve creare un thread figlio per `--thread auto|here`.
- Se il canale attivo non espone binding ACP per la conversazione corrente, OpenClaw restituisce un chiaro messaggio di non supporto.
- `resume` e le domande su "nuova sessione" riguardano la sessione ACP, non il canale. Puoi riutilizzare o sostituire lo stato runtime senza cambiare la superficie di chat corrente.

### Sessioni associate a thread

Quando i thread binding sono abilitati per un channel adapter, le sessioni ACP possono essere associate ai thread:

- OpenClaw associa un thread a una sessione ACP di destinazione.
- I messaggi successivi in quel thread vengono instradati alla sessione ACP associata.
- L'output ACP viene consegnato di nuovo allo stesso thread.
- Unfocus/close/archive/timeout per inattività o scadenza max-age rimuove l'associazione.

Il supporto ai thread binding è specifico dell'adapter. Se l'adapter del canale attivo non supporta i thread binding, OpenClaw restituisce un chiaro messaggio di non supporto/non disponibilità.

Flag di funzionalità richiesti per ACP associato a thread:

- `acp.enabled=true`
- `acp.dispatch.enabled` è attivo per impostazione predefinita (imposta `false` per sospendere il dispatch ACP)
- Flag di spawn thread ACP del channel adapter abilitato (specifico dell'adapter)
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`

### Canali che supportano thread

- Qualsiasi channel adapter che esponga la capacità di binding sessione/thread.
- Supporto built-in attuale:
  - thread/canali Discord
  - topic Telegram (topic forum in gruppi/supergruppi e topic DM)
- I plugin di canale possono aggiungere supporto tramite la stessa interfaccia di binding.

## Impostazioni specifiche del canale

Per flussi di lavoro non effimeri, configura binding ACP persistenti nelle voci di primo livello `bindings[]`.

### Modello di binding

- `bindings[].type="acp"` contrassegna un binding persistente di conversazione ACP.
- `bindings[].match` identifica la conversazione di destinazione:
  - Canale o thread Discord: `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
  - Topic forum Telegram: `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
  - Chat DM/di gruppo BlueBubbles: `match.channel="bluebubbles"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:* >"`
    Preferisci `chat_id:*` o `chat_identifier:*` per binding di gruppo stabili.
  - Chat DM/di gruppo iMessage: `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:* >"`
    Preferisci `chat_id:*` per binding di gruppo stabili.
- `bindings[].agentId` è l'id dell'agente OpenClaw proprietario.
- Gli override ACP facoltativi si trovano in `bindings[].acp`:
  - `mode` (`persistent` o `oneshot`)
  - `label`
  - `cwd`
  - `backend`

### Valori predefiniti runtime per agente

Usa `agents.list[].runtime` per definire una volta i valori predefiniti ACP per agente:

- `agents.list[].runtime.type="acp"`
- `agents.list[].runtime.acp.agent` (id harness, per esempio `codex` o `claude`)
- `agents.list[].runtime.acp.backend`
- `agents.list[].runtime.acp.mode`
- `agents.list[].runtime.acp.cwd`

Precedenza degli override per sessioni ACP associate:

1. `bindings[].acp.*`
2. `agents.list[].runtime.acp.*`
3. valori predefiniti ACP globali (per esempio `acp.backend`)

Esempio:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

Comportamento:

- OpenClaw assicura che la sessione ACP configurata esista prima dell'uso.
- I messaggi in quel canale o topic vengono instradati alla sessione ACP configurata.
- Nelle conversazioni associate, `/new` e `/reset` reimpostano sul posto la stessa chiave di sessione ACP.
- I binding runtime temporanei (per esempio creati dai flussi di focus su thread) continuano comunque ad applicarsi dove presenti.
- Per spawn ACP cross-agent senza un `cwd` esplicito, OpenClaw eredita il workspace dell'agente di destinazione dalla configurazione dell'agente.
- I percorsi di workspace ereditati mancanti ripiegano sul cwd predefinito del backend; i veri errori di accesso a percorsi esistenti emergono come errori di spawn.

## Avviare sessioni ACP (interfacce)

### Da `sessions_spawn`

Usa `runtime: "acp"` per avviare una sessione ACP da un turno dell'agente o da una chiamata di strumento.

```json
{
  "task": "Open the repo and summarize failing tests",
  "runtime": "acp",
  "agentId": "codex",
  "thread": true,
  "mode": "session"
}
```

Note:

- `runtime` per impostazione predefinita è `subagent`, quindi imposta esplicitamente `runtime: "acp"` per le sessioni ACP.
- Se `agentId` viene omesso, OpenClaw usa `acp.defaultAgent` quando configurato.
- `mode: "session"` richiede `thread: true` per mantenere una conversazione persistente associata.

Dettagli dell'interfaccia:

- `task` (obbligatorio): prompt iniziale inviato alla sessione ACP.
- `runtime` (obbligatorio per ACP): deve essere `"acp"`.
- `agentId` (facoltativo): id dell'harness ACP di destinazione. Usa come fallback `acp.defaultAgent` se impostato.
- `thread` (facoltativo, predefinito `false`): richiede il flusso di binding al thread dove supportato.
- `mode` (facoltativo): `run` (one-shot) o `session` (persistente).
  - il valore predefinito è `run`
  - se `thread: true` e mode viene omesso, OpenClaw può usare come default il comportamento persistente a seconda del percorso runtime
  - `mode: "session"` richiede `thread: true`
- `cwd` (facoltativo): directory di lavoro runtime richiesta (validata dalla policy backend/runtime). Se omesso, lo spawn ACP eredita il workspace dell'agente di destinazione quando configurato; i percorsi ereditati mancanti ripiegano sui valori predefiniti del backend, mentre i veri errori di accesso vengono restituiti.
- `label` (facoltativo): etichetta rivolta all'operatore usata nel testo di sessione/banner.
- `resumeSessionId` (facoltativo): riprende una sessione ACP esistente invece di crearne una nuova. L'agente ricarica la cronologia della conversazione tramite `session/load`. Richiede `runtime: "acp"`.
- `streamTo` (facoltativo): `"parent"` trasmette riepiloghi del progresso della sessione ACP iniziale alla sessione richiedente come eventi di sistema.
  - Quando disponibile, le risposte accettate includono `streamLogPath` che punta a un log JSONL con ambito sessione (`<sessionId>.acp-stream.jsonl`) che puoi seguire per avere la cronologia completa del relay.

### Riprendere una sessione esistente

Usa `resumeSessionId` per continuare una sessione ACP precedente invece di ripartire da zero. L'agente ricarica la cronologia della conversazione tramite `session/load`, così riprende con il contesto completo di quanto accaduto prima.

```json
{
  "task": "Continue where we left off — fix the remaining test failures",
  "runtime": "acp",
  "agentId": "codex",
  "resumeSessionId": "<previous-session-id>"
}
```

Casi d'uso comuni:

- Passare una sessione Codex dal tuo laptop al tuo telefono — chiedi al tuo agente di riprendere da dove avevi lasciato
- Continuare una sessione di coding iniziata interattivamente nella CLI, ora in modalità headless tramite il tuo agente
- Riprendere lavoro interrotto da un riavvio del gateway o da un timeout per inattività

Note:

- `resumeSessionId` richiede `runtime: "acp"` — restituisce un errore se usato con il runtime sub-agent.
- `resumeSessionId` ripristina la cronologia della conversazione ACP upstream; `thread` e `mode` continuano ad applicarsi normalmente alla nuova sessione OpenClaw che stai creando, quindi `mode: "session"` richiede comunque `thread: true`.
- L'agente di destinazione deve supportare `session/load` (Codex e Claude Code lo supportano).
- Se l'id di sessione non viene trovato, lo spawn fallisce con un errore chiaro — nessun fallback silenzioso a una nuova sessione.

### Smoke test per operatori

Usa questo dopo un deploy del gateway quando vuoi un rapido controllo live che lo spawn ACP
funzioni davvero end-to-end, non solo che superi i test unitari.

Gate consigliato:

1. Verifica la versione/commit del gateway distribuito sull'host di destinazione.
2. Conferma che il sorgente distribuito includa l'accettazione del lineage ACP in
   `src/gateway/sessions-patch.ts` (`subagent:* or acp:* sessions`).
3. Apri una sessione bridge ACPX temporanea verso un agente live (per esempio
   `razor(main)` su `jpclawhq`).
4. Chiedi a quell'agente di chiamare `sessions_spawn` con:
   - `runtime: "acp"`
   - `agentId: "codex"`
   - `mode: "run"`
   - task: `Reply with exactly LIVE-ACP-SPAWN-OK`
5. Verifica che l'agente riporti:
   - `accepted=yes`
   - un vero `childSessionKey`
   - nessun errore di validazione
6. Ripulisci la sessione bridge ACPX temporanea.

Esempio di prompt per l'agente live:

```text
Use the sessions_spawn tool now with runtime: "acp", agentId: "codex", and mode: "run".
Set the task to: "Reply with exactly LIVE-ACP-SPAWN-OK".
Then report only: accepted=<yes/no>; childSessionKey=<value or none>; error=<exact text or none>.
```

Note:

- Mantieni questo smoke test su `mode: "run"` a meno che tu non stia testando intenzionalmente
  sessioni ACP persistenti associate a thread.
- Non richiedere `streamTo: "parent"` per il gate di base. Quel percorso dipende dalle
  capacità della sessione/richiedente ed è un controllo di integrazione separato.
- Tratta il test `mode: "session"` associato a thread come un secondo passaggio di integrazione più ricco
  da un vero thread Discord o topic Telegram.

## Compatibilità con il sandbox

Le sessioni ACP attualmente vengono eseguite sul runtime host, non all'interno del sandbox OpenClaw.

Limitazioni attuali:

- Se la sessione richiedente è in sandbox, gli spawn ACP vengono bloccati sia per `sessions_spawn({ runtime: "acp" })` sia per `/acp spawn`.
  - Errore: `Sandboxed sessions cannot spawn ACP sessions because runtime="acp" runs on the host. Use runtime="subagent" from sandboxed sessions.`
- `sessions_spawn` con `runtime: "acp"` non supporta `sandbox: "require"`.
  - Errore: `sessions_spawn sandbox="require" is unsupported for runtime="acp" because ACP sessions run outside the sandbox. Use runtime="subagent" or sandbox="inherit".`

Usa `runtime: "subagent"` quando hai bisogno di esecuzione imposta dal sandbox.

### Dal comando `/acp`

Usa `/acp spawn` per un controllo esplicito da operatore dalla chat quando necessario.

```text
/acp spawn codex --mode persistent --thread auto
/acp spawn codex --mode oneshot --thread off
/acp spawn codex --bind here
/acp spawn codex --thread here
```

Flag principali:

- `--mode persistent|oneshot`
- `--bind here|off`
- `--thread auto|here|off`
- `--cwd <absolute-path>`
- `--label <name>`

Vedi [Slash Commands](/it/tools/slash-commands).

## Risoluzione del target della sessione

La maggior parte delle azioni `/acp` accetta un target di sessione facoltativo (`session-key`, `session-id` o `session-label`).

Ordine di risoluzione:

1. Argomento target esplicito (o `--session` per `/acp steer`)
   - prova prima la chiave
   - poi l'id di sessione in formato UUID
   - poi l'etichetta
2. Binding del thread corrente (se questa conversazione/thread è associato a una sessione ACP)
3. Fallback alla sessione del richiedente corrente

I binding alla conversazione corrente e quelli ai thread partecipano entrambi al passaggio 2.

Se nessun target viene risolto, OpenClaw restituisce un errore chiaro (`Unable to resolve session target: ...`).

## Modalità di binding dello spawn

`/acp spawn` supporta `--bind here|off`.

| Modalità | Comportamento                                                           |
| -------- | ----------------------------------------------------------------------- |
| `here`   | Associa sul posto la conversazione attiva corrente; fallisce se non ce n'è una attiva. |
| `off`    | Non crea un binding della conversazione corrente.                       |

Note:

- `--bind here` è il percorso operativo più semplice per "rendere questo canale o chat supportato da Codex."
- `--bind here` non crea un thread figlio.
- `--bind here` è disponibile solo sui canali che espongono il supporto per il binding alla conversazione corrente.
- `--bind` e `--thread` non possono essere combinati nella stessa chiamata `/acp spawn`.

## Modalità thread dello spawn

`/acp spawn` supporta `--thread auto|here|off`.

| Modalità | Comportamento                                                                                           |
| -------- | -------------------------------------------------------------------------------------------------------- |
| `auto`   | In un thread attivo: associa quel thread. Fuori da un thread: crea/associa un thread figlio dove supportato. |
| `here`   | Richiede il thread attivo corrente; fallisce se non sei in un thread.                                   |
| `off`    | Nessun binding. La sessione si avvia non associata.                                                     |

Note:

- Sulle superfici senza binding ai thread, il comportamento predefinito è di fatto `off`.
- Lo spawn associato a thread richiede supporto dalla policy del canale:
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`
  - Telegram: `channels.telegram.threadBindings.spawnAcpSessions=true`
- Usa `--bind here` quando vuoi fissare la conversazione corrente senza creare un thread figlio.

## Controlli ACP

Famiglia di comandi disponibili:

- `/acp spawn`
- `/acp cancel`
- `/acp steer`
- `/acp close`
- `/acp status`
- `/acp set-mode`
- `/acp set`
- `/acp cwd`
- `/acp permissions`
- `/acp timeout`
- `/acp model`
- `/acp reset-options`
- `/acp sessions`
- `/acp doctor`
- `/acp install`

`/acp status` mostra le opzioni runtime effettive e, quando disponibili, sia gli identificatori di sessione a livello runtime sia quelli a livello backend.

Alcuni controlli dipendono dalle capacità del backend. Se un backend non supporta un controllo, OpenClaw restituisce un chiaro errore di controllo non supportato.

## Ricettario dei comandi ACP

| Comando              | Cosa fa                                                        | Esempio                                                       |
| -------------------- | -------------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | Crea una sessione ACP; binding corrente o binding a thread facoltativi. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Annulla il turno in corso per la sessione di destinazione.     | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Invia un'istruzione di steering alla sessione in esecuzione.   | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Chiude la sessione e sgancia i target di thread associati.     | `/acp close`                                                  |
| `/acp status`        | Mostra backend, modalità, stato, opzioni runtime, capacità.    | `/acp status`                                                 |
| `/acp set-mode`      | Imposta la modalità runtime per la sessione di destinazione.   | `/acp set-mode plan`                                          |
| `/acp set`           | Scrittura generica di opzione di configurazione runtime.       | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Imposta l'override della directory di lavoro runtime.          | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Imposta il profilo della policy di approvazione.               | `/acp permissions strict`                                     |
| `/acp timeout`       | Imposta il timeout runtime (secondi).                          | `/acp timeout 120`                                            |
| `/acp model`         | Imposta l'override del modello runtime.                        | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Rimuove gli override delle opzioni runtime della sessione.     | `/acp reset-options`                                          |
| `/acp sessions`      | Elenca le sessioni ACP recenti dallo store.                    | `/acp sessions`                                               |
| `/acp doctor`        | Salute del backend, capacità, correzioni utilizzabili.         | `/acp doctor`                                                 |
| `/acp install`       | Stampa passaggi deterministici di installazione e abilitazione. | `/acp install`                                               |

`/acp sessions` legge lo store per la sessione corrente associata o la sessione del richiedente. I comandi che accettano token `session-key`, `session-id` o `session-label` risolvono i target tramite la discovery delle sessioni del gateway, incluse root personalizzate `session.store` per agente.

## Mappatura delle opzioni runtime

`/acp` ha comandi di convenienza e un setter generico.

Operazioni equivalenti:

- `/acp model <id>` corrisponde alla chiave di configurazione runtime `model`.
- `/acp permissions <profile>` corrisponde alla chiave di configurazione runtime `approval_policy`.
- `/acp timeout <seconds>` corrisponde alla chiave di configurazione runtime `timeout`.
- `/acp cwd <path>` aggiorna direttamente l'override del cwd runtime.
- `/acp set <key> <value>` è il percorso generico.
  - Caso speciale: `key=cwd` usa il percorso di override cwd.
- `/acp reset-options` cancella tutti gli override runtime per la sessione di destinazione.

## Supporto harness acpx (attuale)

Alias harness built-in attuali di acpx:

- `claude`
- `codex`
- `copilot`
- `cursor` (Cursor CLI: `cursor-agent acp`)
- `droid`
- `gemini`
- `iflow`
- `kilocode`
- `kimi`
- `kiro`
- `openclaw`
- `opencode`
- `pi`
- `qwen`

Quando OpenClaw usa il backend acpx, preferisci questi valori per `agentId` a meno che la tua configurazione acpx non definisca alias agente personalizzati.
Se la tua installazione locale di Cursor espone ancora ACP come `agent acp`, sovrascrivi il comando dell'agente `cursor` nella tua configurazione acpx invece di cambiare il valore predefinito built-in.

L'uso diretto della CLI acpx può anche puntare ad adapter arbitrari tramite `--agent <command>`, ma questa via di fuga raw è una funzionalità della CLI acpx (non il normale percorso `agentId` di OpenClaw).

## Configurazione richiesta

Baseline core ACP:

```json5
{
  acp: {
    enabled: true,
    // Facoltativo. Il valore predefinito è true; imposta false per sospendere il dispatch ACP mantenendo i controlli /acp.
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "pi",
      "qwen",
    ],
    maxConcurrentSessions: 8,
    stream: {
      coalesceIdleMs: 300,
      maxChunkChars: 1200,
    },
    runtime: {
      ttlMinutes: 120,
    },
  },
}
```

La configurazione del thread binding è specifica del channel adapter. Esempio per Discord:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        spawnAcpSessions: true,
      },
    },
  },
}
```

Se lo spawn ACP associato a thread non funziona, verifica prima il flag di funzionalità dell'adapter:

- Discord: `channels.discord.threadBindings.spawnAcpSessions=true`

I binding alla conversazione corrente non richiedono la creazione di thread figli. Richiedono un contesto di conversazione attivo e un channel adapter che esponga i conversation binding ACP.

Vedi [Configuration Reference](/it/gateway/configuration-reference).

## Configurazione del plugin per il backend acpx

Le installazioni nuove includono il plugin runtime incluso `acpx` abilitato per impostazione predefinita, quindi ACP
di solito funziona senza un passaggio manuale di installazione del plugin.

Inizia con:

```text
/acp doctor
```

Se hai disabilitato `acpx`, lo hai negato tramite `plugins.allow` / `plugins.deny`, oppure vuoi
passare a un checkout locale di sviluppo, usa il percorso esplicito del plugin:

```bash
openclaw plugins install acpx
openclaw config set plugins.entries.acpx.enabled true
```

Installazione da workspace locale durante lo sviluppo:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

Poi verifica lo stato del backend:

```text
/acp doctor
```

### Configurazione del comando e della versione di acpx

Per impostazione predefinita, il plugin backend acpx incluso (`acpx`) usa il binario locale al plugin bloccato a una versione precisa:

1. Il comando usa per impostazione predefinita `node_modules/.bin/acpx` locale al plugin dentro il package plugin ACPX.
2. La versione attesa usa per impostazione predefinita il pin dell'estensione.
3. All'avvio, OpenClaw registra immediatamente il backend ACP come non pronto.
4. Un job di ensure in background verifica `acpx --version`.
5. Se il binario locale al plugin manca o non corrisponde, esegue:
   `npm install --omit=dev --no-save acpx@<pinned>` e riverifica.

Puoi sovrascrivere comando/versione nella configurazione del plugin:

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "command": "../acpx/dist/cli.js",
          "expectedVersion": "any"
        }
      }
    }
  }
}
```

Note:

- `command` accetta un percorso assoluto, un percorso relativo o un nome comando (`acpx`).
- I percorsi relativi vengono risolti dalla directory del workspace OpenClaw.
- `expectedVersion: "any"` disabilita il controllo rigoroso della versione.
- Quando `command` punta a un binario/percorso personalizzato, l'installazione automatica locale al plugin viene disabilitata.
- L'avvio di OpenClaw rimane non bloccante mentre viene eseguito il controllo dello stato del backend.

Vedi [Plugins](/it/tools/plugin).

### Installazione automatica delle dipendenze

Quando installi OpenClaw globalmente con `npm install -g openclaw`, le dipendenze runtime di acpx
(binari specifici della piattaforma) vengono installate automaticamente
tramite un hook postinstall. Se l'installazione automatica fallisce, il gateway si avvia comunque
normalmente e segnala la dipendenza mancante tramite `openclaw acp doctor`.

### Bridge MCP per gli strumenti del plugin

Per impostazione predefinita, le sessioni ACPX **non** espongono all'harness ACP gli strumenti registrati dai plugin OpenClaw.

Se vuoi che agenti ACP come Codex o Claude Code possano chiamare gli strumenti
dei plugin OpenClaw installati, come memory recall/store, abilita il bridge dedicato:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

Cosa fa:

- Inietta un server MCP built-in chiamato `openclaw-plugin-tools` nel bootstrap della sessione ACPX.
- Espone gli strumenti plugin già registrati dai plugin OpenClaw installati e abilitati.
- Mantiene la funzionalità esplicita e disattivata per impostazione predefinita.

Note su sicurezza e fiducia:

- Questo amplia la superficie degli strumenti dell'harness ACP.
- Gli agenti ACP ottengono accesso solo agli strumenti dei plugin già attivi nel gateway.
- Trattalo come lo stesso confine di fiducia che concedi permettendo a quei plugin di essere eseguiti in
  OpenClaw stesso.
- Esamina i plugin installati prima di abilitarlo.

I `mcpServers` personalizzati continuano a funzionare come prima. Il bridge built-in degli strumenti plugin è una comodità aggiuntiva opt-in, non un sostituto della configurazione generica dei server MCP.

## Configurazione dei permessi

Le sessioni ACP vengono eseguite in modalità non interattiva — non esiste una TTY per approvare o negare prompt di permesso per scrittura di file ed esecuzione di shell. Il plugin acpx fornisce due chiavi di configurazione che controllano come vengono gestiti i permessi:

Questi permessi harness ACPX sono separati dalle approvazioni exec di OpenClaw e separati dai flag di bypass del vendor del backend CLI come Claude CLI `--permission-mode bypassPermissions`. ACPX `approve-all` è l'interruttore break-glass a livello harness per le sessioni ACP.

### `permissionMode`

Controlla quali operazioni l'agente harness può eseguire senza prompt.

| Valore          | Comportamento                                              |
| --------------- | ---------------------------------------------------------- |
| `approve-all`   | Approva automaticamente tutte le scritture di file e i comandi shell. |
| `approve-reads` | Approva automaticamente solo le letture; scritture ed exec richiedono prompt. |
| `deny-all`      | Nega tutti i prompt di permesso.                           |

### `nonInteractivePermissions`

Controlla cosa succede quando verrebbe mostrato un prompt di permesso ma non è disponibile una TTY interattiva (cosa che per le sessioni ACP avviene sempre).

| Valore | Comportamento                                                         |
| ------ | --------------------------------------------------------------------- |
| `fail` | Interrompe la sessione con `AcpRuntimeError`. **(predefinito)**       |
| `deny` | Nega silenziosamente il permesso e continua (degradazione controllata). |

### Configurazione

Imposta tramite la configurazione del plugin:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

Riavvia il gateway dopo aver modificato questi valori.

> **Importante:** OpenClaw attualmente usa come valori predefiniti `permissionMode=approve-reads` e `nonInteractivePermissions=fail`. Nelle sessioni ACP non interattive, qualsiasi scrittura o exec che attiva un prompt di permesso può fallire con `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`.
>
> Se devi limitare i permessi, imposta `nonInteractivePermissions` su `deny` così le sessioni degradano in modo controllato invece di andare in crash.

## Risoluzione dei problemi

| Sintomo                                                                     | Causa probabile                                                                 | Correzione                                                                                                                                                         |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ACP runtime backend is not configured`                                     | Plugin backend mancante o disabilitato.                                         | Installa e abilita il plugin backend, poi esegui `/acp doctor`.                                                                                                   |
| `ACP is disabled by policy (acp.enabled=false)`                             | ACP disabilitato globalmente.                                                   | Imposta `acp.enabled=true`.                                                                                                                                         |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`           | Il dispatch dai normali messaggi dei thread è disabilitato.                     | Imposta `acp.dispatch.enabled=true`.                                                                                                                                |
| `ACP agent "<id>" is not allowed by policy`                                 | L'agente non è nella allowlist.                                                 | Usa un `agentId` consentito oppure aggiorna `acp.allowedAgents`.                                                                                                   |
| `Unable to resolve session target: ...`                                     | Token key/id/label non valido.                                                  | Esegui `/acp sessions`, copia la key/label esatta, riprova.                                                                                                        |
| `--bind here requires running /acp spawn inside an active ... conversation` | `--bind here` usato senza una conversazione attiva associabile.                 | Spostati nella chat/canale di destinazione e riprova, oppure usa uno spawn non associato.                                                                         |
| `Conversation bindings are unavailable for <channel>.`                      | L'adapter non ha capacità ACP di binding alla conversazione corrente.           | Usa `/acp spawn ... --thread ...` dove supportato, configura `bindings[]` di primo livello oppure spostati su un canale supportato.                               |
| `--thread here requires running /acp spawn inside an active ... thread`     | `--thread here` usato fuori da un contesto thread.                              | Spostati nel thread di destinazione oppure usa `--thread auto`/`off`.                                                                                              |
| `Only <user-id> can rebind this channel/conversation/thread.`               | Un altro utente possiede il target di binding attivo.                           | Riesegui il binding come proprietario oppure usa una conversazione o thread diverso.                                                                               |
| `Thread bindings are unavailable for <channel>.`                            | L'adapter non ha capacità di thread binding.                                    | Usa `--thread off` oppure spostati su un adapter/canale supportato.                                                                                                |
| `Sandboxed sessions cannot spawn ACP sessions ...`                          | Il runtime ACP è lato host; la sessione richiedente è in sandbox.               | Usa `runtime="subagent"` da sessioni in sandbox, oppure esegui lo spawn ACP da una sessione non in sandbox.                                                       |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`     | `sandbox="require"` richiesto per il runtime ACP.                               | Usa `runtime="subagent"` per il sandbox obbligatorio, oppure usa ACP con `sandbox="inherit"` da una sessione non in sandbox.                                      |
| Metadati ACP mancanti per la sessione associata                             | Metadati della sessione ACP obsoleti/eliminati.                                 | Ricrea con `/acp spawn`, poi riassocia/rifocalizza il thread.                                                                                                      |
| `AcpRuntimeError: Permission prompt unavailable in non-interactive mode`    | `permissionMode` blocca scritture/exec nella sessione ACP non interattiva.      | Imposta `plugins.entries.acpx.config.permissionMode` su `approve-all` e riavvia il gateway. Vedi [Configurazione dei permessi](#configurazione-dei-permessi).    |
| La sessione ACP fallisce presto con poco output                             | I prompt di permesso sono bloccati da `permissionMode`/`nonInteractivePermissions`. | Controlla i log del gateway per `AcpRuntimeError`. Per permessi completi, imposta `permissionMode=approve-all`; per degradazione controllata, imposta `nonInteractivePermissions=deny`. |
| La sessione ACP resta bloccata indefinitamente dopo aver completato il lavoro | Il processo harness è terminato ma la sessione ACP non ha segnalato il completamento. | Monitora con `ps aux \| grep acpx`; termina manualmente i processi obsoleti.                                                                                       |
