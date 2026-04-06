---
read_when:
    - Hai bisogno del comportamento dettagliato di `openclaw onboard`
    - Stai eseguendo il debug dei risultati dell'onboarding o integrando client di onboarding
sidebarTitle: CLI reference
summary: Riferimento completo per il flusso di configurazione CLI, setup auth/modello, output e dettagli interni
title: Riferimento della configurazione CLI
x-i18n:
    generated_at: "2026-04-06T03:12:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 92f379b34a2b48c68335dae4f759117c770f018ec51b275f4f40421c6b3abb23
    source_path: start/wizard-cli-reference.md
    workflow: 15
---

# Riferimento della configurazione CLI

Questa pagina è il riferimento completo per `openclaw onboard`.
Per la guida breve, vedi [Onboarding (CLI)](/it/start/wizard).

## Cosa fa la procedura guidata

La modalità locale (predefinita) ti guida attraverso:

- Configurazione del modello e dell'autenticazione (OAuth con abbonamento OpenAI Code, Anthropic Claude CLI o chiave API, oltre alle opzioni MiniMax, GLM, Ollama, Moonshot, StepFun e AI Gateway)
- Posizione dello workspace e file bootstrap
- Impostazioni del gateway (porta, bind, autenticazione, Tailscale)
- Canali e provider (Telegram, WhatsApp, Discord, Google Chat, Mattermost, Signal, BlueBubbles e altri plugin canale bundled)
- Installazione del daemon (LaunchAgent, unità utente systemd o Scheduled Task nativa di Windows con fallback nella cartella Startup)
- Controllo dello stato
- Configurazione delle Skills

La modalità remota configura questa macchina per connettersi a un gateway altrove.
Non installa né modifica nulla sull'host remoto.

## Dettagli del flusso locale

<Steps>
  <Step title="Rilevamento della configurazione esistente">
    - Se esiste `~/.openclaw/openclaw.json`, scegli Mantieni, Modifica o Reimposta.
    - Rieseguire la procedura guidata non cancella nulla a meno che tu non scelga esplicitamente Reimposta (o passi `--reset`).
    - `--reset` della CLI usa per impostazione predefinita `config+creds+sessions`; usa `--reset-scope full` per rimuovere anche lo workspace.
    - Se la configurazione non è valida o contiene chiavi legacy, la procedura guidata si ferma e ti chiede di eseguire `openclaw doctor` prima di continuare.
    - La reimpostazione usa `trash` e offre questi ambiti:
      - Solo configurazione
      - Configurazione + credenziali + sessioni
      - Reimpostazione completa (rimuove anche lo workspace)
  </Step>
  <Step title="Modello e autenticazione">
    - La matrice completa delle opzioni è in [Opzioni di autenticazione e modello](#opzioni-di-autenticazione-e-modello).
  </Step>
  <Step title="Workspace">
    - Predefinito `~/.openclaw/workspace` (configurabile).
    - Inserisce i file workspace necessari per il rituale di bootstrap del primo avvio.
    - Layout dello workspace: [Agent workspace](/it/concepts/agent-workspace).
  </Step>
  <Step title="Gateway">
    - Richiede porta, bind, modalità auth ed esposizione Tailscale.
    - Consigliato: mantieni abilitata l'autenticazione tramite token anche per loopback in modo che i client WS locali debbano autenticarsi.
    - In modalità token, la configurazione interattiva offre:
      - **Generate/store plaintext token** (predefinito)
      - **Use SecretRef** (opt-in)
    - In modalità password, la configurazione interattiva supporta anche l'archiviazione in testo semplice o SecretRef.
    - Percorso SecretRef del token non interattivo: `--gateway-token-ref-env <ENV_VAR>`.
      - Richiede una variabile env non vuota nell'ambiente del processo di onboarding.
      - Non può essere combinato con `--gateway-token`.
    - Disabilita l'autenticazione solo se ti fidi completamente di ogni processo locale.
    - I bind non-loopback richiedono comunque l'autenticazione.
  </Step>
  <Step title="Canali">
    - [WhatsApp](/it/channels/whatsapp): login QR facoltativo
    - [Telegram](/it/channels/telegram): token del bot
    - [Discord](/it/channels/discord): token del bot
    - [Google Chat](/it/channels/googlechat): JSON dell'account di servizio + audience webhook
    - [Mattermost](/it/channels/mattermost): token del bot + URL base
    - [Signal](/it/channels/signal): installazione facoltativa di `signal-cli` + configurazione account
    - [BlueBubbles](/it/channels/bluebubbles): consigliato per iMessage; URL server + password + webhook
    - [iMessage](/it/channels/imessage): percorso CLI legacy `imsg` + accesso DB
    - Sicurezza DM: l'impostazione predefinita è pairing. Il primo DM invia un codice; approvalo con
      `openclaw pairing approve <channel> <code>` oppure usa le allowlist.
  </Step>
  <Step title="Installazione del daemon">
    - macOS: LaunchAgent
      - Richiede una sessione utente con accesso effettuato; per ambienti headless, usa un LaunchDaemon personalizzato (non incluso).
    - Linux e Windows tramite WSL2: unità utente systemd
      - La procedura guidata prova `loginctl enable-linger <user>` in modo che il gateway resti attivo dopo il logout.
      - Potrebbe richiedere sudo (scrive `/var/lib/systemd/linger`); prima ci prova senza sudo.
    - Windows nativo: Scheduled Task per prima
      - Se la creazione del task viene negata, OpenClaw usa come fallback un elemento di accesso nella cartella Startup per utente e avvia immediatamente il gateway.
      - Le Scheduled Task restano preferite perché forniscono uno stato del supervisore migliore.
    - Selezione del runtime: Node (consigliato; richiesto per WhatsApp e Telegram). Bun non è consigliato.
  </Step>
  <Step title="Controllo dello stato">
    - Avvia il gateway (se necessario) ed esegue `openclaw health`.
    - `openclaw status --deep` aggiunge la probe live dello stato del gateway all'output di stato, incluse le probe dei canali quando supportate.
  </Step>
  <Step title="Skills">
    - Legge le Skills disponibili e controlla i requisiti.
    - Ti consente di scegliere il node manager: npm, pnpm o bun.
    - Installa le dipendenze facoltative (alcune usano Homebrew su macOS).
  </Step>
  <Step title="Fine">
    - Riepilogo e passaggi successivi, incluse le opzioni per app iOS, Android e macOS.
  </Step>
</Steps>

<Note>
Se non viene rilevata alcuna GUI, la procedura guidata stampa istruzioni di port-forward SSH per la Control UI invece di aprire un browser.
Se mancano le risorse della Control UI, la procedura guidata prova a compilarle; il fallback è `pnpm ui:build` (installa automaticamente le dipendenze UI).
</Note>

## Dettagli della modalità remota

La modalità remota configura questa macchina per connettersi a un gateway altrove.

<Info>
La modalità remota non installa né modifica nulla sull'host remoto.
</Info>

Cosa imposti:

- URL del gateway remoto (`ws://...`)
- Token se è richiesta l'autenticazione del gateway remoto (consigliato)

<Note>
- Se il gateway è solo loopback, usa il tunneling SSH o una tailnet.
- Suggerimenti di rilevamento:
  - macOS: Bonjour (`dns-sd`)
  - Linux: Avahi (`avahi-browse`)
</Note>

## Opzioni di autenticazione e modello

<AccordionGroup>
  <Accordion title="Chiave API Anthropic">
    Usa `ANTHROPIC_API_KEY` se presente oppure richiede una chiave, quindi la salva per l'uso da parte del daemon.
  </Accordion>
  <Accordion title="Abbonamento OpenAI Code (riuso della CLI Codex)">
    Se esiste `~/.codex/auth.json`, la procedura guidata può riutilizzarlo.
    Le credenziali della CLI Codex riutilizzate restano gestite dalla CLI Codex; alla scadenza OpenClaw
    rilegge prima quella sorgente e, quando il provider può aggiornarla, scrive
    la credenziale aggiornata di nuovo nello storage Codex invece di prenderne il controllo
    direttamente.
  </Accordion>
  <Accordion title="Abbonamento OpenAI Code (OAuth)">
    Flusso browser; incolla `code#state`.

    Imposta `agents.defaults.model` su `openai-codex/gpt-5.4` quando il modello non è impostato o è `openai/*`.

  </Accordion>
  <Accordion title="Chiave API OpenAI">
    Usa `OPENAI_API_KEY` se presente oppure richiede una chiave, quindi memorizza la credenziale nei profili auth.

    Imposta `agents.defaults.model` su `openai/gpt-5.4` quando il modello non è impostato, è `openai/*` o `openai-codex/*`.

  </Accordion>
  <Accordion title="Chiave API xAI (Grok)">
    Richiede `XAI_API_KEY` e configura xAI come provider del modello.
  </Accordion>
  <Accordion title="OpenCode">
    Richiede `OPENCODE_API_KEY` (oppure `OPENCODE_ZEN_API_KEY`) e ti permette di scegliere il catalogo Zen o Go.
    URL di configurazione: [opencode.ai/auth](https://opencode.ai/auth).
  </Accordion>
  <Accordion title="Chiave API (generica)">
    Memorizza la chiave per te.
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    Richiede `AI_GATEWAY_API_KEY`.
    Maggiori dettagli: [Vercel AI Gateway](/it/providers/vercel-ai-gateway).
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    Richiede account ID, gateway ID e `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    Maggiori dettagli: [Cloudflare AI Gateway](/it/providers/cloudflare-ai-gateway).
  </Accordion>
  <Accordion title="MiniMax">
    La configurazione viene scritta automaticamente. Il valore hosted predefinito è `MiniMax-M2.7`; la configurazione con chiave API usa
    `minimax/...`, mentre la configurazione OAuth usa `minimax-portal/...`.
    Maggiori dettagli: [MiniMax](/it/providers/minimax).
  </Accordion>
  <Accordion title="StepFun">
    La configurazione viene scritta automaticamente per StepFun standard o Step Plan sugli endpoint Cina o globali.
    Standard include attualmente `step-3.5-flash`, e Step Plan include anche `step-3.5-flash-2603`.
    Maggiori dettagli: [StepFun](/it/providers/stepfun).
  </Accordion>
  <Accordion title="Synthetic (compatibile con Anthropic)">
    Richiede `SYNTHETIC_API_KEY`.
    Maggiori dettagli: [Synthetic](/it/providers/synthetic).
  </Accordion>
  <Accordion title="Ollama (Cloud e modelli open locali)">
    Richiede URL base (predefinito `http://127.0.0.1:11434`), poi offre modalità Cloud + Local o Local.
    Rileva i modelli disponibili e suggerisce i predefiniti.
    Maggiori dettagli: [Ollama](/it/providers/ollama).
  </Accordion>
  <Accordion title="Moonshot e Kimi Coding">
    Le configurazioni di Moonshot (Kimi K2) e Kimi Coding vengono scritte automaticamente.
    Maggiori dettagli: [Moonshot AI (Kimi + Kimi Coding)](/it/providers/moonshot).
  </Accordion>
  <Accordion title="Provider personalizzato">
    Funziona con endpoint compatibili con OpenAI e compatibili con Anthropic.

    L'onboarding interattivo supporta le stesse opzioni di archiviazione della chiave API degli altri flussi di chiave API del provider:
    - **Paste API key now** (testo semplice)
    - **Use secret reference** (riferimento env o provider configurato, con validazione preflight)

    Flag non interattivi:
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key` (facoltativo; fallback a `CUSTOM_API_KEY`)
    - `--custom-provider-id` (facoltativo)
    - `--custom-compatibility <openai|anthropic>` (facoltativo; predefinito `openai`)

  </Accordion>
  <Accordion title="Salta">
    Lascia l'autenticazione non configurata.
  </Accordion>
</AccordionGroup>

Comportamento del modello:

- Scegli il modello predefinito tra le opzioni rilevate, oppure inserisci manualmente provider e modello.
- Quando l'onboarding parte da una scelta di autenticazione del provider, il selettore del modello preferisce
  automaticamente quel provider. Per Volcengine e BytePlus, la stessa preferenza
  corrisponde anche alle loro varianti coding-plan (`volcengine-plan/*`,
  `byteplus-plan/*`).
- Se quel filtro del provider preferito sarebbe vuoto, il selettore torna al
  catalogo completo invece di non mostrare modelli.
- La procedura guidata esegue un controllo del modello e avvisa se il modello configurato è sconosciuto o manca l'autenticazione.

Percorsi delle credenziali e dei profili:

- Profili auth (chiavi API + OAuth): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Importazione OAuth legacy: `~/.openclaw/credentials/oauth.json`

Modalità di archiviazione delle credenziali:

- Il comportamento predefinito dell'onboarding salva le chiavi API come valori in testo semplice nei profili auth.
- `--secret-input-mode ref` abilita la modalità riferimento invece dell'archiviazione in testo semplice della chiave.
  Nella configurazione interattiva puoi scegliere tra:
  - riferimento a variabile di ambiente (ad esempio `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)
  - riferimento a provider configurato (`file` o `exec`) con alias provider + id
- La modalità riferimento interattiva esegue una rapida validazione preflight prima del salvataggio.
  - Riferimenti env: convalida il nome della variabile + valore non vuoto nell'ambiente di onboarding corrente.
  - Riferimenti provider: convalida la config del provider e risolve l'id richiesto.
  - Se il preflight fallisce, l'onboarding mostra l'errore e ti consente di riprovare.
- In modalità non interattiva, `--secret-input-mode ref` è supportato solo con env.
  - Imposta la variabile env del provider nell'ambiente del processo di onboarding.
  - I flag inline della chiave (ad esempio `--openai-api-key`) richiedono che tale variabile env sia impostata; altrimenti l'onboarding fallisce immediatamente.
  - Per i provider personalizzati, la modalità `ref` non interattiva memorizza `models.providers.<id>.apiKey` come `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`.
  - In quel caso del provider personalizzato, `--custom-api-key` richiede che `CUSTOM_API_KEY` sia impostato; altrimenti l'onboarding fallisce immediatamente.
- Le credenziali auth del gateway supportano scelte testo semplice e SecretRef nella configurazione interattiva:
  - Modalità token: **Generate/store plaintext token** (predefinito) oppure **Use SecretRef**.
  - Modalità password: testo semplice oppure SecretRef.
- Percorso SecretRef del token non interattivo: `--gateway-token-ref-env <ENV_VAR>`.
- Le configurazioni esistenti in testo semplice continuano a funzionare senza modifiche.

<Note>
Suggerimento per ambienti headless e server: completa OAuth su una macchina con browser, poi copia
`auth-profiles.json` di quell'agente (ad esempio
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, o il corrispondente
percorso `$OPENCLAW_STATE_DIR/...`) sull'host del gateway. `credentials/oauth.json`
è solo una sorgente legacy per l'importazione.
</Note>

## Output e dettagli interni

Campi tipici in `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (se è stato scelto MiniMax)
- `tools.profile` (l'onboarding locale imposta questo valore su `"coding"` quando non è impostato; i valori espliciti esistenti vengono preservati)
- `gateway.*` (mode, bind, auth, tailscale)
- `session.dmScope` (l'onboarding locale imposta questo valore su `per-channel-peer` quando non è impostato; i valori espliciti esistenti vengono preservati)
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Allowlist dei canali (Slack, Discord, Matrix, Microsoft Teams) quando scegli l'opzione durante i prompt (i nomi vengono risolti in ID quando possibile)
- `skills.install.nodeManager`
  - Il flag `setup --node-manager` accetta `npm`, `pnpm` o `bun`.
  - La configurazione manuale può comunque impostare successivamente `skills.install.nodeManager: "yarn"`.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`openclaw agents add` scrive `agents.list[]` e `bindings` facoltativi.

Le credenziali WhatsApp vengono salvate in `~/.openclaw/credentials/whatsapp/<accountId>/`.
Le sessioni vengono archiviate in `~/.openclaw/agents/<agentId>/sessions/`.

<Note>
Alcuni canali vengono distribuiti come plugin. Quando vengono selezionati durante la configurazione, la procedura guidata
richiede di installare il plugin (npm o percorso locale) prima della configurazione del canale.
</Note>

RPC della procedura guidata del gateway:

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

I client (app macOS e Control UI) possono renderizzare i passaggi senza reimplementare la logica di onboarding.

Comportamento della configurazione di Signal:

- Scarica l'asset di release appropriato
- Lo salva in `~/.openclaw/tools/signal-cli/<version>/`
- Scrive `channels.signal.cliPath` nella configurazione
- Le build JVM richiedono Java 21
- Le build native vengono usate quando disponibili
- Windows usa WSL2 e segue il flusso Linux di signal-cli all'interno di WSL

## Documentazione correlata

- Hub onboarding: [Onboarding (CLI)](/it/start/wizard)
- Automazione e script: [CLI Automation](/it/start/wizard-cli-automation)
- Riferimento comandi: [`openclaw onboard`](/cli/onboard)
