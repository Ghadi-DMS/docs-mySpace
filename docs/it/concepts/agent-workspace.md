---
read_when:
    - Devi spiegare l'area di lavoro dell'agente o il layout dei suoi file.
    - Vuoi eseguire il backup o migrare un'area di lavoro dell'agente.
summary: 'Area di lavoro dell''agente: posizione, layout e strategia di backup'
title: Area di lavoro dell'agente
x-i18n:
    generated_at: "2026-04-18T08:05:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: dd2e74614d8d45df04b1bbda48e2224e778b621803d774d38e4b544195eb234e
    source_path: concepts/agent-workspace.md
    workflow: 15
---

# Area di lavoro dell'agente

L'area di lavoro è la casa dell'agente. È l'unica directory di lavoro usata per
gli strumenti sui file e per il contesto dell'area di lavoro. Mantienila privata e trattala come memoria.

Questa è separata da `~/.openclaw/`, che archivia configurazione, credenziali e
sessioni.

**Importante:** l'area di lavoro è la **cwd predefinita**, non una sandbox rigida. Gli strumenti
risolvono i percorsi relativi rispetto all'area di lavoro, ma i percorsi assoluti possono comunque raggiungere
altre posizioni sull'host a meno che il sandboxing non sia abilitato. Se hai bisogno di isolamento, usa
[`agents.defaults.sandbox`](/it/gateway/sandboxing) (e/o la configurazione sandbox per agente).
Quando il sandboxing è abilitato e `workspaceAccess` non è `"rw"`, gli strumenti operano
all'interno di un'area di lavoro sandbox sotto `~/.openclaw/sandboxes`, non nella tua area di lavoro host.

## Posizione predefinita

- Predefinita: `~/.openclaw/workspace`
- Se `OPENCLAW_PROFILE` è impostato e non è `"default"`, il valore predefinito diventa
  `~/.openclaw/workspace-<profile>`.
- Sostituisci in `~/.openclaw/openclaw.json`:

```json5
{
  agent: {
    workspace: "~/.openclaw/workspace",
  },
}
```

`openclaw onboard`, `openclaw configure` o `openclaw setup` creeranno
l'area di lavoro e inizializzeranno i file bootstrap se mancanti.
Le copie seed della sandbox accettano solo file normali interni all'area di lavoro; gli alias
symlink/hardlink che si risolvono all'esterno dell'area di lavoro sorgente vengono ignorati.

Se gestisci già tu stesso i file dell'area di lavoro, puoi disabilitare la creazione dei file bootstrap:

```json5
{ agent: { skipBootstrap: true } }
```

## Cartelle aggiuntive dell'area di lavoro

Le installazioni meno recenti potrebbero aver creato `~/openclaw`. Mantenere più directory di area di lavoro
può causare derive confuse di autenticazione o stato, perché solo un'area di lavoro è attiva alla volta.

**Raccomandazione:** mantieni una sola area di lavoro attiva. Se non usi più le
cartelle aggiuntive, archiviale o spostale nel Cestino (ad esempio `trash ~/openclaw`).
Se mantieni intenzionalmente più aree di lavoro, assicurati che
`agents.defaults.workspace` punti a quella attiva.

`openclaw doctor` avvisa quando rileva directory aggiuntive dell'area di lavoro.

## Mappa dei file dell'area di lavoro (significato di ciascun file)

Questi sono i file standard che OpenClaw si aspetta all'interno dell'area di lavoro:

- `AGENTS.md`
  - Istruzioni operative per l'agente e su come dovrebbe usare la memoria.
  - Caricato all'inizio di ogni sessione.
  - Buon posto per regole, priorità e dettagli su "come comportarsi".

- `SOUL.md`
  - Persona, tono e limiti.
  - Caricato a ogni sessione.
  - Guida: [Guida alla personalità di SOUL.md](/it/concepts/soul)

- `USER.md`
  - Chi è l'utente e come rivolgersi a lui.
  - Caricato a ogni sessione.

- `IDENTITY.md`
  - Nome, stile ed emoji dell'agente.
  - Creato/aggiornato durante il rituale bootstrap.

- `TOOLS.md`
  - Note sui tuoi strumenti locali e sulle convenzioni.
  - Non controlla la disponibilità degli strumenti; è solo una guida.

- `HEARTBEAT.md`
  - Piccola checklist facoltativa per le esecuzioni Heartbeat.
  - Mantienila breve per evitare consumo di token.

- `BOOT.md`
  - Checklist di avvio facoltativa eseguita al riavvio del Gateway quando gli hook interni sono abilitati.
  - Mantienila breve; usa lo strumento message per gli invii in uscita.

- `BOOTSTRAP.md`
  - Rituale una tantum per la prima esecuzione.
  - Creato solo per un'area di lavoro nuova.
  - Eliminalo dopo il completamento del rituale.

- `memory/YYYY-MM-DD.md`
  - Log di memoria giornaliero (un file per giorno).
  - Si consiglia di leggere oggi + ieri all'inizio della sessione.

- `MEMORY.md` (facoltativo)
  - Memoria a lungo termine curata.
  - Caricalo solo nella sessione principale e privata (non nei contesti condivisi/di gruppo).

Vedi [Memory](/it/concepts/memory) per il flusso di lavoro e lo svuotamento automatico della memoria.

- `skills/` (facoltativa)
  - Skills specifiche dell'area di lavoro.
  - Posizione delle Skills con precedenza più alta per quell'area di lavoro.
  - Sovrascrive le skills dell'agente di progetto, le skills dell'agente personale, le skills gestite, le skills incluse e `skills.load.extraDirs` quando i nomi coincidono.

- `canvas/` (facoltativa)
  - File UI canvas per le visualizzazioni dei nodi (ad esempio `canvas/index.html`).

Se manca un file bootstrap, OpenClaw inserisce un marker di "file mancante" nella
sessione e continua. I file bootstrap grandi vengono troncati durante l'inserimento;
regola i limiti con `agents.defaults.bootstrapMaxChars` (predefinito: 12000) e
`agents.defaults.bootstrapTotalMaxChars` (predefinito: 60000).
`openclaw setup` può ricreare i valori predefiniti mancanti senza sovrascrivere i
file esistenti.

## Cosa NON si trova nell'area di lavoro

Questi elementi si trovano sotto `~/.openclaw/` e NON devono essere sottoposti a commit nel repo dell'area di lavoro:

- `~/.openclaw/openclaw.json` (configurazione)
- `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (profili di autenticazione dei modelli: OAuth + chiavi API)
- `~/.openclaw/credentials/` (stato di canali/provider più dati legacy di importazione OAuth)
- `~/.openclaw/agents/<agentId>/sessions/` (trascrizioni delle sessioni + metadati)
- `~/.openclaw/skills/` (Skills gestite)

Se devi migrare sessioni o configurazione, copiale separatamente e tienile
fuori dal controllo di versione.

## Backup git (consigliato, privato)

Tratta l'area di lavoro come memoria privata. Inseriscila in un repo git **privato** in modo che sia
sottoposta a backup e recuperabile.

Esegui questi passaggi sulla macchina in cui gira il Gateway (è lì che si trova
l'area di lavoro).

### 1) Inizializza il repo

Se git è installato, le aree di lavoro nuove vengono inizializzate automaticamente. Se questa
area di lavoro non è già un repo, esegui:

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
git commit -m "Add agent workspace"
```

### 2) Aggiungi un remote privato (opzioni semplici per principianti)

Opzione A: interfaccia web di GitHub

1. Crea un nuovo repository **privato** su GitHub.
2. Non inizializzarlo con un README (evita conflitti di merge).
3. Copia l'URL HTTPS del remote.
4. Aggiungi il remote ed esegui il push:

```bash
git branch -M main
git remote add origin <https-url>
git push -u origin main
```

Opzione B: GitHub CLI (`gh`)

```bash
gh auth login
gh repo create openclaw-workspace --private --source . --remote origin --push
```

Opzione C: interfaccia web di GitLab

1. Crea un nuovo repository **privato** su GitLab.
2. Non inizializzarlo con un README (evita conflitti di merge).
3. Copia l'URL HTTPS del remote.
4. Aggiungi il remote ed esegui il push:

```bash
git branch -M main
git remote add origin <https-url>
git push -u origin main
```

### 3) Aggiornamenti continui

```bash
git status
git add .
git commit -m "Update memory"
git push
```

## Non eseguire il commit dei segreti

Anche in un repo privato, evita di archiviare segreti nell'area di lavoro:

- Chiavi API, token OAuth, password o credenziali private.
- Qualsiasi elemento sotto `~/.openclaw/`.
- Dump grezzi di chat o allegati sensibili.

Se devi archiviare riferimenti sensibili, usa segnaposto e conserva il segreto reale
altrove (gestore password, variabili d'ambiente o `~/.openclaw/`).

Esempio iniziale di `.gitignore` consigliato:

```gitignore
.DS_Store
.env
**/*.key
**/*.pem
**/secrets*
```

## Spostare l'area di lavoro su una nuova macchina

1. Clona il repo nel percorso desiderato (predefinito `~/.openclaw/workspace`).
2. Imposta `agents.defaults.workspace` su quel percorso in `~/.openclaw/openclaw.json`.
3. Esegui `openclaw setup --workspace <path>` per inizializzare eventuali file mancanti.
4. Se ti servono le sessioni, copia separatamente `~/.openclaw/agents/<agentId>/sessions/` dalla
   vecchia macchina.

## Note avanzate

- Il routing multi-agente può usare aree di lavoro diverse per agente. Vedi
  [Instradamento dei canali](/it/channels/channel-routing) per la configurazione del routing.
- Se `agents.defaults.sandbox` è abilitato, le sessioni non principali possono usare
  aree di lavoro sandbox per sessione sotto `agents.defaults.sandbox.workspaceRoot`.

## Correlati

- [Standing Orders](/it/automation/standing-orders) — istruzioni persistenti nei file dell'area di lavoro
- [Heartbeat](/it/gateway/heartbeat) — file dell'area di lavoro HEARTBEAT.md
- [Session](/it/concepts/session) — percorsi di archiviazione delle sessioni
- [Sandboxing](/it/gateway/sandboxing) — accesso all'area di lavoro in ambienti sandboxizzati
