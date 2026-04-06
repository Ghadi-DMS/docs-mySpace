---
read_when:
    - Aggiunta o modifica delle migrazioni doctor
    - Introduzione di modifiche incompatibili alla configurazione
summary: 'Comando Doctor: controlli di integrità, migrazioni della configurazione e passaggi di riparazione'
title: Doctor
x-i18n:
    generated_at: "2026-04-06T03:08:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6c0a15c522994552a1eef39206bed71fc5bf45746776372f24f31c101bfbd411
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor` è lo strumento di riparazione + migrazione per OpenClaw. Corregge
configurazione/stato obsoleti, controlla l'integrità e fornisce passaggi di riparazione attuabili.

## Avvio rapido

```bash
openclaw doctor
```

### Headless / automazione

```bash
openclaw doctor --yes
```

Accetta i valori predefiniti senza chiedere conferma (inclusi i passaggi di riparazione di riavvio/servizio/sandbox quando applicabili).

```bash
openclaw doctor --repair
```

Applica le riparazioni consigliate senza chiedere conferma (riparazioni + riavvii dove sicuro).

```bash
openclaw doctor --repair --force
```

Applica anche riparazioni aggressive (sovrascrive configurazioni supervisor personalizzate).

```bash
openclaw doctor --non-interactive
```

Esegue senza prompt e applica solo migrazioni sicure (normalizzazione della configurazione + spostamenti dello stato su disco). Salta le azioni di riavvio/servizio/sandbox che richiedono conferma umana.
Le migrazioni dello stato legacy vengono eseguite automaticamente quando rilevate.

```bash
openclaw doctor --deep
```

Analizza i servizi di sistema per installazioni gateway aggiuntive (launchd/systemd/schtasks).

Se vuoi rivedere le modifiche prima della scrittura, apri prima il file di configurazione:

```bash
cat ~/.openclaw/openclaw.json
```

## Cosa fa (riepilogo)

- Aggiornamento pre-flight facoltativo per installazioni git (solo interattivo).
- Controllo di aggiornamento del protocollo UI (ricompila la Control UI quando lo schema del protocollo è più recente).
- Controllo di integrità + prompt di riavvio.
- Riepilogo dello stato di Skills (idonee/mancanti/bloccate) e stato dei plugin.
- Normalizzazione della configurazione per valori legacy.
- Migrazione della configurazione Talk dai campi flat legacy `talk.*` a `talk.provider` + `talk.providers.<provider>`.
- Controlli di migrazione browser per configurazioni legacy dell'estensione Chrome e verifica della readiness di Chrome MCP.
- Avvisi sugli override del provider OpenCode (`models.providers.opencode` / `models.providers.opencode-go`).
- Controllo dei prerequisiti TLS OAuth per i profili OpenAI Codex OAuth.
- Migrazione legacy dello stato su disco (sessioni/dir agente/auth WhatsApp).
- Migrazione delle chiavi legacy dei contratti del manifesto plugin (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- Migrazione legacy dell'archivio cron (`jobId`, `schedule.cron`, campi delivery/payload di primo livello, payload `provider`, job fallback webhook semplici `notify: true`).
- Ispezione dei file di lock di sessione e pulizia dei lock obsoleti.
- Controlli di integrità e permessi dello stato (sessioni, transcript, directory di stato).
- Controlli dei permessi del file di configurazione (`chmod 600`) quando viene eseguito in locale.
- Integrità dell'autenticazione del modello: controlla la scadenza OAuth, può aggiornare token in scadenza e segnala gli stati cooldown/disabilitato dei profili auth.
- Rilevamento di directory workspace aggiuntive (`~/openclaw`).
- Riparazione dell'immagine sandbox quando il sandboxing è abilitato.
- Migrazione dei servizi legacy e rilevamento di gateway aggiuntivi.
- Migrazione legacy dello stato del canale Matrix (in modalità `--fix` / `--repair`).
- Controlli runtime del gateway (servizio installato ma non in esecuzione; etichetta launchd memorizzata nella cache).
- Avvisi sullo stato dei canali (sondati dal gateway in esecuzione).
- Audit della configurazione supervisor (launchd/systemd/schtasks) con riparazione facoltativa.
- Controlli delle best practice del runtime gateway (Node vs Bun, percorsi dei version manager).
- Diagnostica dei conflitti di porta del gateway (predefinita `18789`).
- Avvisi di sicurezza per policy DM aperte.
- Controlli di autenticazione del gateway per la modalità token locale (offre la generazione del token quando non esiste alcuna sorgente di token; non sovrascrive configurazioni token SecretRef).
- Controllo `systemd linger` su Linux.
- Controllo della dimensione dei file bootstrap del workspace (avvisi di troncamento/quasi limite per i file di contesto).
- Controllo dello stato del completamento della shell e installazione/aggiornamento automatici.
- Controllo della readiness del provider di embedding per memory search (modello locale, chiave API remota o binario QMD).
- Controlli di installazione da sorgente (mismatch del workspace pnpm, asset UI mancanti, binario tsx mancante).
- Scrive la configurazione aggiornata + i metadati del wizard.

## Comportamento dettagliato e motivazione

### 0) Aggiornamento facoltativo (installazioni git)

Se si tratta di un checkout git e doctor è in esecuzione in modalità interattiva, offre di
aggiornare (fetch/rebase/build) prima di eseguire doctor.

### 1) Normalizzazione della configurazione

Se la configurazione contiene forme di valore legacy (ad esempio `messages.ackReaction`
senza override specifico per canale), doctor le normalizza nello schema
corrente.

Questo include i campi flat legacy di Talk. La configurazione pubblica attuale di Talk è
`talk.provider` + `talk.providers.<provider>`. Doctor riscrive le vecchie forme
`talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` nella mappa provider.

### 2) Migrazioni delle chiavi di configurazione legacy

Quando la configurazione contiene chiavi deprecate, gli altri comandi si rifiutano di eseguire e chiedono
di eseguire `openclaw doctor`.

Doctor farà quanto segue:

- Spiegherà quali chiavi legacy sono state trovate.
- Mostrerà la migrazione applicata.
- Riscriverà `~/.openclaw/openclaw.json` con lo schema aggiornato.

Il Gateway esegue automaticamente anche le migrazioni doctor all'avvio quando rileva un
formato di configurazione legacy, quindi le configurazioni obsolete vengono riparate senza intervento manuale.
Le migrazioni dell'archivio dei job cron sono gestite da `openclaw doctor --fix`.

Migrazioni attuali:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → `bindings` di primo livello
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- legacy `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- Per i canali con `accounts` nominati ma con valori di canale di primo livello ancora relativi a un singolo account, sposta quei valori scoped all'account nell'account promosso scelto per quel canale (`accounts.default` per la maggior parte dei canali; Matrix può preservare un target nominato/predefinito esistente corrispondente)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- rimuove `browser.relayBindHost` (impostazione legacy del relay dell'estensione)

Gli avvisi di doctor includono anche indicazioni sui valori predefiniti degli account per i canali multi-account:

- Se sono configurate due o più voci `channels.<channel>.accounts` senza `channels.<channel>.defaultAccount` o `accounts.default`, doctor avvisa che il routing di fallback può scegliere un account inatteso.
- Se `channels.<channel>.defaultAccount` è impostato su un ID account sconosciuto, doctor avvisa ed elenca gli ID account configurati.

### 2b) Override del provider OpenCode

Se hai aggiunto manualmente `models.providers.opencode`, `opencode-zen` o `opencode-go`,
questo sovrascrive il catalogo OpenCode integrato da `@mariozechner/pi-ai`.
Ciò può forzare i modelli sull'API sbagliata o azzerare i costi. Doctor avvisa in modo che tu
possa rimuovere l'override e ripristinare l'instradamento API per modello + i costi.

### 2c) Migrazione browser e readiness di Chrome MCP

Se la tua configurazione browser punta ancora al percorso rimosso dell'estensione Chrome, doctor
la normalizza all'attuale modello di attach Chrome MCP locale all'host:

- `browser.profiles.*.driver: "extension"` diventa `"existing-session"`
- `browser.relayBindHost` viene rimosso

Doctor esegue anche un audit del percorso Chrome MCP locale all'host quando usi `defaultProfile:
"user"` o un profilo `existing-session` configurato:

- controlla se Google Chrome è installato sullo stesso host per i profili
  di connessione automatica predefiniti
- controlla la versione di Chrome rilevata e avvisa quando è inferiore a Chrome 144
- ricorda di abilitare il debug remoto nella pagina inspect del browser (per
  esempio `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  o `edge://inspect/#remote-debugging`)

Doctor non può abilitare per te l'impostazione lato Chrome. Chrome MCP locale all'host
richiede comunque:

- un browser basato su Chromium 144+ sull'host gateway/node
- il browser in esecuzione localmente
- il debug remoto abilitato in quel browser
- l'approvazione del primo prompt di consenso attach nel browser

La readiness qui riguarda solo i prerequisiti dell'attach locale. Existing-session mantiene
gli attuali limiti di instradamento Chrome MCP; instradamenti avanzati come `responsebody`, esportazione
PDF, intercettazione dei download e azioni batch richiedono comunque un
browser gestito o un profilo CDP raw.

Questo controllo **non** si applica a Docker, sandbox, remote-browser o altri
flussi headless. Questi continuano a usare CDP raw.

### 2d) Prerequisiti TLS OAuth

Quando è configurato un profilo OpenAI Codex OAuth, doctor sonda l'endpoint di autorizzazione
OpenAI per verificare che lo stack TLS locale Node/OpenSSL possa
convalidare la catena dei certificati. Se la sonda fallisce con un errore di certificato (per
esempio `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, certificato scaduto o certificato autofirmato),
doctor stampa indicazioni di correzione specifiche per piattaforma. Su macOS con un Node Homebrew, la
correzione è di solito `brew postinstall ca-certificates`. Con `--deep`, la sonda viene eseguita
anche se il gateway è integro.

### 3) Migrazioni legacy dello stato (layout su disco)

Doctor può migrare vecchi layout su disco nella struttura attuale:

- Archivio sessioni + transcript:
  - da `~/.openclaw/sessions/` a `~/.openclaw/agents/<agentId>/sessions/`
- Directory agente:
  - da `~/.openclaw/agent/` a `~/.openclaw/agents/<agentId>/agent/`
- Stato auth WhatsApp (Baileys):
  - dal legacy `~/.openclaw/credentials/*.json` (eccetto `oauth.json`)
  - a `~/.openclaw/credentials/whatsapp/<accountId>/...` (ID account predefinito: `default`)

Queste migrazioni sono best-effort e idempotenti; doctor emetterà avvisi quando
lascia eventuali cartelle legacy come backup. Anche Gateway/CLI eseguono l'auto-migrazione
delle sessioni legacy + directory agente all'avvio così che cronologia/auth/modelli finiscano nel
percorso per agente senza un'esecuzione manuale di doctor. L'auth WhatsApp viene intenzionalmente
migrata solo tramite `openclaw doctor`. La normalizzazione della mappa provider/provider di Talk ora
confronta per uguaglianza strutturale, quindi le differenze dovute solo all'ordine delle chiavi non attivano
più cambiamenti ripetuti e nulli di `doctor --fix`.

### 3a) Migrazioni legacy del manifesto plugin

Doctor analizza tutti i manifesti plugin installati per trovare chiavi di capability
di primo livello deprecate (`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`). Quando vengono trovate, propone di spostarle nell'oggetto `contracts`
e riscrivere il file manifesto sul posto. Questa migrazione è idempotente;
se la chiave `contracts` contiene già gli stessi valori, la chiave legacy viene rimossa
senza duplicare i dati.

### 3b) Migrazioni legacy dell'archivio cron

Doctor controlla anche l'archivio dei job cron (`~/.openclaw/cron/jobs.json` per impostazione predefinita,
o `cron.store` se ridefinito) per trovare vecchie forme di job che lo scheduler continua
ad accettare per compatibilità.

Le attuali pulizie cron includono:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- campi payload di primo livello (`message`, `model`, `thinking`, ...) → `payload`
- campi delivery di primo livello (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- alias delivery `provider` del payload → `delivery.channel` esplicito
- job fallback webhook legacy semplici `notify: true` → `delivery.mode="webhook"` esplicito con `delivery.to=cron.webhook`

Doctor esegue automaticamente la migrazione dei job `notify: true` solo quando può farlo senza
modificare il comportamento. Se un job combina il fallback notify legacy con una modalità
delivery non-webhook esistente, doctor avvisa e lascia quel job alla revisione manuale.

### 3c) Pulizia dei lock di sessione

Doctor analizza ogni directory di sessione agente per trovare file di write-lock obsoleti — file lasciati
indietro quando una sessione è terminata in modo anomalo. Per ogni file lock trovato segnala:
il percorso, il PID, se il PID è ancora attivo, l'età del lock e se è
considerato obsoleto (PID non attivo o più vecchio di 30 minuti). In modalità `--fix` / `--repair`
rimuove automaticamente i file lock obsoleti; altrimenti stampa una nota e
ti indica di rieseguire con `--fix`.

### 4) Controlli di integrità dello stato (persistenza della sessione, routing e sicurezza)

La directory di stato è il tronco encefalico operativo. Se scompare, perdi
sessioni, credenziali, log e configurazione (a meno che tu non abbia backup altrove).

Doctor controlla:

- **Directory di stato mancante**: avvisa della perdita catastrofica dello stato, chiede di ricreare
  la directory e ricorda che non può recuperare i dati mancanti.
- **Permessi della directory di stato**: verifica la scrivibilità; offre di riparare i permessi
  (ed emette un suggerimento `chown` quando viene rilevata una mancata corrispondenza di owner/group).
- **Directory di stato macOS sincronizzata nel cloud**: avvisa quando lo stato risolve sotto iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) o
  `~/Library/CloudStorage/...` perché i percorsi supportati dalla sincronizzazione possono causare I/O più lento
  e race di lock/sincronizzazione.
- **Directory di stato Linux su SD o eMMC**: avvisa quando lo stato risolve a una sorgente di mount `mmcblk*`,
  perché l'I/O casuale supportato da SD o eMMC può essere più lento e usurarsi
  più velocemente sotto le scritture di sessione e credenziali.
- **Directory di sessione mancanti**: `sessions/` e la directory dell'archivio sessioni sono
  necessarie per persistere la cronologia ed evitare crash `ENOENT`.
- **Mancata corrispondenza transcript**: avvisa quando voci di sessione recenti hanno file
  transcript mancanti.
- **Sessione principale “JSONL a 1 riga”**: segnala quando il transcript principale ha una sola
  riga (la cronologia non si accumula).
- **Più directory di stato**: avvisa quando esistono più cartelle `~/.openclaw` tra
  diverse home directory o quando `OPENCLAW_STATE_DIR` punta altrove (la cronologia può
  dividersi tra installazioni).
- **Promemoria modalità remota**: se `gateway.mode=remote`, doctor ricorda di eseguirlo
  sull'host remoto (lo stato si trova lì).
- **Permessi del file di configurazione**: avvisa se `~/.openclaw/openclaw.json` è
  leggibile da gruppo/world e offre di restringerli a `600`.

### 5) Integrità dell'autenticazione del modello (scadenza OAuth)

Doctor ispeziona i profili OAuth nell'archivio auth, avvisa quando i token stanno
scadendo/sono scaduti e può aggiornarli quando è sicuro. Se il profilo
OAuth/token Anthropic è obsoleto, suggerisce una chiave API Anthropic o il percorso legacy
Anthropic setup-token.
I prompt di refresh vengono mostrati solo in esecuzione interattiva (TTY); `--non-interactive`
salta i tentativi di refresh.

Doctor rileva anche stato Anthropic Claude CLI rimosso ma obsoleto. Se i vecchi
byte delle credenziali `anthropic:claude-cli` esistono ancora in `auth-profiles.json`,
doctor li riconverte in profili token/OAuth Anthropic e riscrive i riferimenti modello
obsoleti `claude-cli/...`.
Se i byte non ci sono più, doctor rimuove la configurazione obsoleta e stampa invece
comandi di ripristino.

Doctor segnala anche i profili auth temporaneamente inutilizzabili a causa di:

- cooldown brevi (limiti di frequenza/timeout/errori auth)
- disabilitazioni più lunghe (errori di fatturazione/credito)

### 6) Validazione del modello hooks

Se `hooks.gmail.model` è impostato, doctor convalida il riferimento al modello rispetto al
catalogo e alla allowlist e avvisa quando non può essere risolto o non è consentito.

### 7) Riparazione dell'immagine sandbox

Quando il sandboxing è abilitato, doctor controlla le immagini Docker e offre di compilarle o
passare a nomi legacy se l'immagine attuale manca.

### 7b) Dipendenze runtime dei plugin bundled

Doctor verifica che le dipendenze runtime dei plugin bundled (per esempio i
pacchetti runtime del plugin Discord) siano presenti nella root di installazione di OpenClaw.
Se ne manca qualcuna, doctor segnala i pacchetti e li installa in
modalità `openclaw doctor --fix` / `openclaw doctor --repair`.

### 8) Migrazioni dei servizi gateway e indicazioni di pulizia

Doctor rileva i servizi gateway legacy (launchd/systemd/schtasks) e
offre di rimuoverli e installare il servizio OpenClaw usando l'attuale porta gateway.
Può anche analizzare servizi simili a gateway aggiuntivi e stampare indicazioni per la pulizia.
I servizi gateway OpenClaw con nome profilo sono considerati di prima classe e non
vengono segnalati come "aggiuntivi".

### 8b) Migrazione Matrix all'avvio

Quando un account del canale Matrix ha una migrazione dello stato legacy in sospeso o azionabile,
doctor (in modalità `--fix` / `--repair`) crea uno snapshot pre-migrazione e poi
esegue i passaggi di migrazione best-effort: migrazione dello stato Matrix legacy e preparazione
dello stato legacy cifrato. Entrambi i passaggi non sono fatali; gli errori vengono registrati e
l'avvio continua. In modalità sola lettura (`openclaw doctor` senza `--fix`) questo controllo
viene saltato completamente.

### 9) Avvisi di sicurezza

Doctor emette avvisi quando un provider è aperto ai DM senza allowlist, oppure
quando una policy è configurata in modo pericoloso.

### 10) systemd linger (Linux)

Se è in esecuzione come servizio utente systemd, doctor si assicura che linger sia abilitato in modo che il
gateway resti attivo dopo il logout.

### 11) Stato del workspace (Skills, plugin e directory legacy)

Doctor stampa un riepilogo dello stato del workspace per l'agente predefinito:

- **Stato Skills**: conta Skills idonee, con requisiti mancanti e bloccate dalla allowlist.
- **Directory workspace legacy**: avvisa quando `~/openclaw` o altre directory workspace legacy
  esistono insieme al workspace corrente.
- **Stato plugin**: conta plugin caricati/disabilitati/in errore; elenca gli ID plugin per eventuali
  errori; riporta le capability dei plugin bundle.
- **Avvisi di compatibilità plugin**: segnala plugin che hanno problemi di compatibilità con
  il runtime corrente.
- **Diagnostica plugin**: espone eventuali avvisi o errori emessi in fase di caricamento dal
  registro plugin.

### 11b) Dimensione del file bootstrap

Doctor controlla se i file bootstrap del workspace (per esempio `AGENTS.md`,
`CLAUDE.md` o altri file di contesto iniettati) sono vicini o oltre il budget
di caratteri configurato. Segnala per-file i conteggi di caratteri grezzi vs. iniettati, la percentuale di
troncamento, la causa del troncamento (`max/file` o `max/total`) e il totale dei caratteri
iniettati come frazione del budget totale. Quando i file sono troncati o vicini
al limite, doctor stampa suggerimenti per regolare `agents.defaults.bootstrapMaxChars`
e `agents.defaults.bootstrapTotalMaxChars`.

### 11c) Completamento della shell

Doctor controlla se il completamento tramite tab è installato per la shell corrente
(zsh, bash, fish o PowerShell):

- Se il profilo shell usa un pattern di completamento dinamico lento
  (`source <(openclaw completion ...)`), doctor lo aggiorna alla variante più veloce
  con file in cache.
- Se il completamento è configurato nel profilo ma il file cache manca,
  doctor rigenera automaticamente la cache.
- Se il completamento non è configurato affatto, doctor chiede di installarlo
  (solo modalità interattiva; saltato con `--non-interactive`).

Esegui `openclaw completion --write-state` per rigenerare manualmente la cache.

### 12) Controlli auth del gateway (token locale)

Doctor controlla la readiness dell'autenticazione token del gateway locale.

- Se la modalità token ha bisogno di un token e non esiste alcuna sorgente di token, doctor offre di generarne uno.
- Se `gateway.auth.token` è gestito da SecretRef ma non disponibile, doctor avvisa e non lo sovrascrive con testo in chiaro.
- `openclaw doctor --generate-gateway-token` forza la generazione solo quando non è configurato alcun token SecretRef.

### 12b) Riparazioni in sola lettura compatibili con SecretRef

Alcuni flussi di riparazione devono ispezionare le credenziali configurate senza indebolire il comportamento fail-fast del runtime.

- `openclaw doctor --fix` ora usa lo stesso modello di riepilogo SecretRef in sola lettura dei comandi della famiglia status per riparazioni mirate della configurazione.
- Esempio: la riparazione di `allowFrom` / `groupAllowFrom` Telegram con `@username` prova a usare le credenziali del bot configurate quando disponibili.
- Se il token bot Telegram è configurato tramite SecretRef ma non disponibile nel percorso del comando corrente, doctor segnala che la credenziale è configurata-ma-non-disponibile e salta la risoluzione automatica invece di bloccarsi o segnalare erroneamente il token come mancante.

### 13) Controllo di integrità gateway + riavvio

Doctor esegue un controllo di integrità e offre di riavviare il gateway quando sembra
non integro.

### 13b) Readiness di memory search

Doctor controlla se il provider di embedding configurato per memory search è pronto
per l'agente predefinito. Il comportamento dipende dal backend e dal provider configurati:

- **Backend QMD**: sonda se il binario `qmd` è disponibile e avviabile.
  In caso contrario, stampa indicazioni di correzione inclusi il pacchetto npm e un'opzione manuale per il percorso del binario.
- **Provider locale esplicito**: controlla la presenza di un file modello locale o di un URL modello remoto/scaricabile riconosciuto. Se manca, suggerisce di passare a un provider remoto.
- **Provider remoto esplicito** (`openai`, `voyage`, ecc.): verifica che una chiave API sia
  presente nell'ambiente o nell'archivio auth. Stampa suggerimenti di correzione attuabili se manca.
- **Provider automatico**: controlla prima la disponibilità del modello locale, poi prova ciascun provider remoto nell'ordine di selezione automatica.

Quando è disponibile un risultato di sonda del gateway (il gateway era integro al momento del
controllo), doctor confronta il risultato con la configurazione visibile dalla CLI e segnala
eventuali discrepanze.

Usa `openclaw memory status --deep` per verificare la readiness degli embedding a runtime.

### 14) Avvisi sullo stato dei canali

Se il gateway è integro, doctor esegue una sonda dello stato dei canali e segnala
avvisi con correzioni suggerite.

### 15) Audit della configurazione supervisor + riparazione

Doctor controlla la configurazione supervisor installata (launchd/systemd/schtasks) per
valori predefiniti mancanti o obsoleti (ad es. dipendenze systemd network-online e
ritardo di riavvio). Quando trova una mancata corrispondenza, consiglia un aggiornamento e può
riscrivere il file di servizio/task con i valori predefiniti correnti.

Note:

- `openclaw doctor` chiede conferma prima di riscrivere la configurazione supervisor.
- `openclaw doctor --yes` accetta i prompt di riparazione predefiniti.
- `openclaw doctor --repair` applica le correzioni consigliate senza prompt.
- `openclaw doctor --repair --force` sovrascrive configurazioni supervisor personalizzate.
- Se l'autenticazione token richiede un token e `gateway.auth.token` è gestito da SecretRef, l'installazione/riparazione del servizio doctor convalida il SecretRef ma non persiste valori token risolti in testo in chiaro nei metadati dell'ambiente del servizio supervisor.
- Se l'autenticazione token richiede un token e il token SecretRef configurato non è risolto, doctor blocca il percorso di installazione/riparazione con indicazioni attuabili.
- Se sono configurati sia `gateway.auth.token` sia `gateway.auth.password` e `gateway.auth.mode` non è impostato, doctor blocca installazione/riparazione finché la modalità non viene impostata esplicitamente.
- Per le unità user-systemd Linux, i controlli doctor di deriva del token ora includono sia le sorgenti `Environment=` sia `EnvironmentFile=` quando confrontano i metadati auth del servizio.
- Puoi sempre forzare una riscrittura completa tramite `openclaw gateway install --force`.

### 16) Diagnostica del runtime gateway + porta

Doctor ispeziona il runtime del servizio (PID, ultimo stato di uscita) e avvisa quando il
servizio è installato ma in realtà non è in esecuzione. Controlla anche i conflitti
di porta sulla porta gateway (predefinita `18789`) e segnala le cause probabili (gateway già
in esecuzione, tunnel SSH).

### 17) Best practice del runtime gateway

Doctor avvisa quando il servizio gateway viene eseguito su Bun o su un percorso Node gestito da version manager
(`nvm`, `fnm`, `volta`, `asdf`, ecc.). I canali WhatsApp + Telegram richiedono Node,
e i percorsi dei version manager possono rompersi dopo gli aggiornamenti perché il servizio non
carica l'inizializzazione della shell. Doctor offre di migrare a un'installazione Node di sistema quando
disponibile (Homebrew/apt/choco).

### 18) Scrittura della configurazione + metadati del wizard

Doctor persiste qualsiasi modifica alla configurazione e registra i metadati del wizard per annotare l'esecuzione
di doctor.

### 19) Suggerimenti per il workspace (backup + sistema di memoria)

Doctor suggerisce un sistema di memoria del workspace quando manca e stampa un suggerimento di backup
se il workspace non è già sotto git.

Vedi [/concepts/agent-workspace](/it/concepts/agent-workspace) per una guida completa alla
struttura del workspace e al backup git (consigliato GitHub o GitLab privato).
