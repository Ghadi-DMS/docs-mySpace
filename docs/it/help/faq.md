---
read_when:
    - Risposta a domande comuni su configurazione, installazione, onboarding o supporto runtime
    - Triage di problemi segnalati dagli utenti prima di un debug più approfondito
summary: Domande frequenti su configurazione, impostazione e utilizzo di OpenClaw
title: FAQ
x-i18n:
    generated_at: "2026-04-06T03:13:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4d6d09621c6033d580cbcf1ff46f81587d69404d6f64c8d8fd8c3f09185bb920
    source_path: help/faq.md
    workflow: 15
---

# FAQ

Risposte rapide più una risoluzione dei problemi più approfondita per configurazioni reali (sviluppo locale, VPS, multi-agent, OAuth/chiavi API, failover dei modelli). Per la diagnostica runtime, vedi [Risoluzione dei problemi](/it/gateway/troubleshooting). Per il riferimento completo della configurazione, vedi [Configurazione](/it/gateway/configuration).

## I primi 60 secondi se qualcosa non funziona

1. **Stato rapido (primo controllo)**

   ```bash
   openclaw status
   ```

   Riepilogo locale rapido: OS + aggiornamento, raggiungibilità del gateway/servizio, agenti/sessioni, configurazione dei provider + problemi runtime (quando il gateway è raggiungibile).

2. **Report copiabile (sicuro da condividere)**

   ```bash
   openclaw status --all
   ```

   Diagnosi in sola lettura con coda dei log (token oscurati).

3. **Daemon + stato della porta**

   ```bash
   openclaw gateway status
   ```

   Mostra il runtime del supervisore rispetto alla raggiungibilità RPC, l'URL di destinazione della probe e quale configurazione probabilmente ha usato il servizio.

4. **Probe approfondite**

   ```bash
   openclaw status --deep
   ```

   Esegue una probe live dello stato del gateway, incluse le probe dei canali quando supportate
   (richiede un gateway raggiungibile). Vedi [Health](/it/gateway/health).

5. **Segui l'ultimo log**

   ```bash
   openclaw logs --follow
   ```

   Se RPC non è disponibile, usa come fallback:

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   I log su file sono separati dai log del servizio; vedi [Logging](/it/logging) e [Risoluzione dei problemi](/it/gateway/troubleshooting).

6. **Esegui doctor (riparazioni)**

   ```bash
   openclaw doctor
   ```

   Ripara/migra configurazione e stato + esegue controlli di integrità. Vedi [Doctor](/it/gateway/doctor).

7. **Snapshot del gateway**

   ```bash
   openclaw health --json
   openclaw health --verbose   # mostra l'URL di destinazione + il percorso della configurazione in caso di errori
   ```

   Chiede al gateway in esecuzione uno snapshot completo (solo WS). Vedi [Health](/it/gateway/health).

## Avvio rapido e configurazione iniziale

<AccordionGroup>
  <Accordion title="Sono bloccato, qual è il modo più rapido per sbloccarmi?">
    Usa un agente AI locale che possa **vedere la tua macchina**. È molto più efficace che chiedere
    su Discord, perché la maggior parte dei casi di "sono bloccato" sono **problemi locali di configurazione o ambiente** che
    gli aiutanti remoti non possono ispezionare.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    Questi strumenti possono leggere il repository, eseguire comandi, ispezionare i log e aiutarti a sistemare la configurazione
    della tua macchina (PATH, servizi, permessi, file di autenticazione). Fornisci loro il **checkout completo del codice sorgente** tramite
    l'installazione hackable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Questo installa OpenClaw **da un checkout git**, così l'agente può leggere il codice + la documentazione e
    ragionare sulla versione esatta che stai eseguendo. Puoi sempre tornare alla versione stabile in seguito
    rieseguendo l'installer senza `--install-method git`.

    Suggerimento: chiedi all'agente di **pianificare e supervisionare** la correzione (passo dopo passo), poi di eseguire solo i
    comandi necessari. In questo modo le modifiche restano piccole e più facili da controllare.

    Se scopri un bug reale o una correzione, apri un issue su GitHub o invia una PR:
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    Inizia con questi comandi (condividi gli output quando chiedi aiuto):

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    Cosa fanno:

    - `openclaw status`: snapshot rapido dello stato di gateway/agente + configurazione di base.
    - `openclaw models status`: controlla l'autenticazione del provider + la disponibilità dei modelli.
    - `openclaw doctor`: convalida e ripara i problemi comuni di configurazione/stato.

    Altri controlli CLI utili: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    Ciclo rapido di debug: [I primi 60 secondi se qualcosa non funziona](#i-primi-60-secondi-se-qualcosa-non-funziona).
    Documentazione di installazione: [Install](/it/install), [Flag dell'installer](/it/install/installer), [Aggiornamento](/it/install/updating).

  </Accordion>

  <Accordion title="Heartbeat continua a essere saltato. Cosa significano i motivi di skip?">
    Motivi comuni di skip di heartbeat:

    - `quiet-hours`: fuori dalla finestra configurata di ore attive
    - `empty-heartbeat-file`: `HEARTBEAT.md` esiste ma contiene solo struttura vuota/intestazioni
    - `no-tasks-due`: la modalità task di `HEARTBEAT.md` è attiva ma nessuno degli intervalli delle attività è ancora scaduto
    - `alerts-disabled`: tutta la visibilità di heartbeat è disabilitata (`showOk`, `showAlerts` e `useIndicator` sono tutti disattivati)

    In modalità task, i timestamp di scadenza avanzano solo dopo il completamento
    di una vera esecuzione di heartbeat. Le esecuzioni saltate non segnano le attività come completate.

    Documentazione: [Heartbeat](/it/gateway/heartbeat), [Automazione e attività](/it/automation).

  </Accordion>

  <Accordion title="Modo consigliato per installare e configurare OpenClaw">
    Il repository consiglia di eseguire dal sorgente e di usare l'onboarding:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    La procedura guidata può anche costruire automaticamente le risorse UI. Dopo l'onboarding, di solito esegui il Gateway sulla porta **18789**.

    Dal sorgente (collaboratori/dev):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # installa automaticamente le dipendenze UI alla prima esecuzione
    openclaw onboard
    ```

    Se non hai ancora un'installazione globale, eseguilo tramite `pnpm openclaw onboard`.

  </Accordion>

  <Accordion title="Come apro la dashboard dopo l'onboarding?">
    La procedura guidata apre il browser con un URL della dashboard pulito (senza token) subito dopo l'onboarding e stampa anche il link nel riepilogo. Tieni aperta quella scheda; se non si è avviata, copia/incolla l'URL stampato sulla stessa macchina.
  </Accordion>

  <Accordion title="Come autentico la dashboard su localhost rispetto a una macchina remota?">
    **Localhost (stessa macchina):**

    - Apri `http://127.0.0.1:18789/`.
    - Se richiede l'autenticazione con secret condiviso, incolla il token o la password configurati nelle impostazioni della Control UI.
    - Origine del token: `gateway.auth.token` (oppure `OPENCLAW_GATEWAY_TOKEN`).
    - Origine della password: `gateway.auth.password` (oppure `OPENCLAW_GATEWAY_PASSWORD`).
    - Se non è ancora configurato alcun secret condiviso, genera un token con `openclaw doctor --generate-gateway-token`.

    **Non su localhost:**

    - **Tailscale Serve** (consigliato): mantieni il bind su loopback, esegui `openclaw gateway --tailscale serve`, apri `https://<magicdns>/`. Se `gateway.auth.allowTailscale` è `true`, gli header di identità soddisfano l'autenticazione di Control UI/WebSocket (nessun secret condiviso da incollare, presuppone un gateway host attendibile); le API HTTP richiedono comunque l'autenticazione con secret condiviso a meno che tu non usi deliberatamente `none` per ingressi privati o l'autenticazione HTTP trusted-proxy.
      I tentativi di autenticazione Serve errati concorrenti dallo stesso client vengono serializzati prima che il limitatore delle autenticazioni fallite li registri, quindi il secondo tentativo errato può già mostrare `retry later`.
    - **Bind tailnet**: esegui `openclaw gateway --bind tailnet --token "<token>"` (oppure configura l'autenticazione con password), apri `http://<tailscale-ip>:18789/`, poi incolla il secret condiviso corrispondente nelle impostazioni della dashboard.
    - **Reverse proxy con riconoscimento dell'identità**: mantieni il Gateway dietro un trusted proxy non-loopback, configura `gateway.auth.mode: "trusted-proxy"`, poi apri l'URL del proxy.
    - **Tunnel SSH**: `ssh -N -L 18789:127.0.0.1:18789 user@host` poi apri `http://127.0.0.1:18789/`. L'autenticazione con secret condiviso si applica ancora sul tunnel; incolla il token o la password configurati se richiesto.

    Vedi [Dashboard](/web/dashboard) e [Superfici web](/web) per i dettagli su modalità di bind e autenticazione.

  </Accordion>

  <Accordion title="Perché ci sono due configurazioni di approvazione exec per le approvazioni in chat?">
    Controllano livelli diversi:

    - `approvals.exec`: inoltra le richieste di approvazione alle destinazioni di chat
    - `channels.<channel>.execApprovals`: fa sì che quel canale agisca come client di approvazione nativo per le approvazioni exec

    La policy exec dell'host resta comunque il vero gate di approvazione. La configurazione chat controlla solo dove
    compaiono le richieste di approvazione e come le persone possono rispondere.

    Nella maggior parte delle configurazioni **non** ti servono entrambe:

    - Se la chat supporta già comandi e risposte, `/approve` nella stessa chat funziona tramite il percorso condiviso.
    - Se un canale nativo supportato può dedurre in sicurezza gli approvatori, OpenClaw ora abilita automaticamente le approvazioni native DM-first quando `channels.<channel>.execApprovals.enabled` non è impostato o è `"auto"`.
    - Quando sono disponibili card/pulsanti di approvazione nativi, quell'interfaccia nativa è il percorso principale; l'agente dovrebbe includere un comando manuale `/approve` solo se il risultato dello strumento indica che le approvazioni in chat non sono disponibili o che l'approvazione manuale è l'unico percorso.
    - Usa `approvals.exec` solo quando le richieste devono essere inoltrate anche ad altre chat o a stanze operative esplicite.
    - Usa `channels.<channel>.execApprovals.target: "channel"` o `"both"` solo quando vuoi esplicitamente che le richieste di approvazione vengano pubblicate di nuovo nella stanza/topic di origine.
    - Le approvazioni dei plugin sono ancora separate: usano per impostazione predefinita `/approve` nella stessa chat, inoltro facoltativo `approvals.plugin`, e solo alcuni canali nativi mantengono una gestione nativa delle approvazioni dei plugin.

    Versione breve: l'inoltro serve per l'instradamento, la configurazione del client nativo serve per una UX di canale più ricca e specifica.
    Vedi [Approvazioni Exec](/it/tools/exec-approvals).

  </Accordion>

  <Accordion title="Di quale runtime ho bisogno?">
    È richiesto Node **>= 22**. `pnpm` è consigliato. Bun **non è consigliato** per il Gateway.
  </Accordion>

  <Accordion title="Funziona su Raspberry Pi?">
    Sì. Il Gateway è leggero - la documentazione indica **512MB-1GB di RAM**, **1 core** e circa **500MB**
    di disco come sufficienti per un uso personale, e segnala che un **Raspberry Pi 4 può eseguirlo**.

    Se vuoi più margine (log, media, altri servizi), **sono consigliati 2GB**, ma
    non è un minimo rigido.

    Suggerimento: un piccolo Pi/VPS può ospitare il Gateway e puoi associare dei **nodi** al tuo laptop/telefono per
    schermo/fotocamera/canvas locali o esecuzione di comandi. Vedi [Nodes](/it/nodes).

  </Accordion>

  <Accordion title="Ci sono suggerimenti per le installazioni su Raspberry Pi?">
    Versione breve: funziona, ma aspettati qualche asperità.

    - Usa un OS **64-bit** e mantieni Node >= 22.
    - Preferisci l'**installazione hackable (git)** così puoi vedere i log e aggiornare rapidamente.
    - Inizia senza canali/Skills, poi aggiungili uno alla volta.
    - Se incontri strani problemi binari, di solito si tratta di un problema di **compatibilità ARM**.

    Documentazione: [Linux](/it/platforms/linux), [Install](/it/install).

  </Accordion>

  <Accordion title="Si blocca su wake up my friend / l'onboarding non si completa. E adesso?">
    Quella schermata dipende dal fatto che il Gateway sia raggiungibile e autenticato. La TUI invia anche
    automaticamente "Wake up, my friend!" al primo avvio. Se vedi quella riga senza **nessuna risposta**
    e i token restano a 0, l'agente non è mai stato eseguito.

    1. Riavvia il Gateway:

    ```bash
    openclaw gateway restart
    ```

    2. Controlla stato + autenticazione:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. Se continua a bloccarsi, esegui:

    ```bash
    openclaw doctor
    ```

    Se il Gateway è remoto, assicurati che la connessione tunnel/Tailscale sia attiva e che l'interfaccia
    punti al Gateway corretto. Vedi [Accesso remoto](/it/gateway/remote).

  </Accordion>

  <Accordion title="Posso migrare la mia configurazione su una nuova macchina (Mac mini) senza rifare l'onboarding?">
    Sì. Copia la **directory di stato** e il **workspace**, poi esegui Doctor una volta. Questo
    mantiene il tuo bot "esattamente uguale" (memoria, cronologia delle sessioni, autenticazione e
    stato dei canali) purché tu copi **entrambe** le posizioni:

    1. Installa OpenClaw sulla nuova macchina.
    2. Copia `$OPENCLAW_STATE_DIR` (predefinito: `~/.openclaw`) dalla vecchia macchina.
    3. Copia il tuo workspace (predefinito: `~/.openclaw/workspace`).
    4. Esegui `openclaw doctor` e riavvia il servizio Gateway.

    In questo modo preservi configurazione, profili di autenticazione, credenziali WhatsApp, sessioni e memoria. Se sei in
    modalità remota, ricorda che l'host del gateway possiede l'archivio delle sessioni e il workspace.

    **Importante:** se esegui solo commit/push del tuo workspace su GitHub, stai facendo
    il backup di **memoria + file bootstrap**, ma **non** della cronologia delle sessioni o dell'autenticazione. Questi elementi si trovano
    in `~/.openclaw/` (per esempio `~/.openclaw/agents/<agentId>/sessions/`).

    Correlati: [Migrazione](/it/install/migrating), [Dove si trovano le cose su disco](#dove-si-trovano-le-cose-su-disco),
    [Workspace dell'agente](/it/concepts/agent-workspace), [Doctor](/it/gateway/doctor),
    [Modalità remota](/it/gateway/remote).

  </Accordion>

  <Accordion title="Dove posso vedere le novità dell'ultima versione?">
    Controlla il changelog su GitHub:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Le voci più recenti sono in alto. Se la sezione in alto è contrassegnata come **Unreleased**, la sezione datata successiva
    è l'ultima versione distribuita. Le voci sono raggruppate in **Highlights**, **Changes** e
    **Fixes** (più sezioni docs/altre quando necessario).

  </Accordion>

  <Accordion title="Impossibile accedere a docs.openclaw.ai (errore SSL)">
    Alcune connessioni Comcast/Xfinity bloccano erroneamente `docs.openclaw.ai` tramite Xfinity
    Advanced Security. Disattivalo o inserisci `docs.openclaw.ai` nella allowlist, poi riprova.
    Aiutaci a sbloccarlo segnalando qui: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    Se non riesci ancora a raggiungere il sito, la documentazione è rispecchiata su GitHub:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="Differenza tra stable e beta">
    **Stable** e **beta** sono **npm dist-tag**, non linee di codice separate:

    - `latest` = stable
    - `beta` = build anticipata per test

    Di solito, una release stabile arriva prima su **beta**, poi un passaggio esplicito
    di promozione sposta quella stessa versione su `latest`. I maintainer possono anche
    pubblicare direttamente su `latest` quando necessario. Per questo beta e stable possono
    puntare alla **stessa versione** dopo la promozione.

    Vedi cosa è cambiato:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    Per i comandi di installazione rapidi e la differenza tra beta e dev, vedi l'accordion seguente.

  </Accordion>

  <Accordion title="Come installo la versione beta e qual è la differenza tra beta e dev?">
    **Beta** è il dist-tag npm `beta` (può coincidere con `latest` dopo la promozione).
    **Dev** è la head in movimento di `main` (git); quando viene pubblicata, usa il dist-tag npm `dev`.

    One-liner (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Installer Windows (PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    Maggiori dettagli: [Canali di sviluppo](/it/install/development-channels) e [Flag dell'installer](/it/install/installer).

  </Accordion>

  <Accordion title="Come posso provare le ultimissime novità?">
    Due opzioni:

    1. **Canale dev (checkout git):**

    ```bash
    openclaw update --channel dev
    ```

    Questo passa al branch `main` e aggiorna dal sorgente.

    2. **Installazione hackable (dal sito dell'installer):**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    In questo modo ottieni un repository locale che puoi modificare e poi aggiornare via git.

    Se preferisci un clone pulito manuale, usa:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    Documentazione: [Update](/cli/update), [Canali di sviluppo](/it/install/development-channels),
    [Install](/it/install).

  </Accordion>

  <Accordion title="Quanto tempo richiedono di solito installazione e onboarding?">
    Guida approssimativa:

    - **Installazione:** 2-5 minuti
    - **Onboarding:** 5-15 minuti a seconda di quanti canali/modelli configuri

    Se si blocca, usa [Installer bloccato](#avvio-rapido-e-configurazione-iniziale)
    e il ciclo rapido di debug in [Sono bloccato](#avvio-rapido-e-configurazione-iniziale).

  </Accordion>

  <Accordion title="Installer bloccato? Come posso ottenere più feedback?">
    Riesegui l'installer con **output verboso**:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    Installazione beta con verbose:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    Per un'installazione hackable (git):

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    Equivalente Windows (PowerShell):

    ```powershell
    # install.ps1 non ha ancora un flag -Verbose dedicato.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    Altre opzioni: [Flag dell'installer](/it/install/installer).

  </Accordion>

  <Accordion title="Su Windows l'installazione dice git not found oppure openclaw not recognized">
    Due problemi comuni su Windows:

    **1) errore npm spawn git / git not found**

    - Installa **Git for Windows** e assicurati che `git` sia nel tuo PATH.
    - Chiudi e riapri PowerShell, poi riesegui l'installer.

    **2) openclaw is not recognized dopo l'installazione**

    - La cartella bin globale di npm non è nel PATH.
    - Controlla il percorso:

      ```powershell
      npm config get prefix
      ```

    - Aggiungi quella directory al PATH utente (su Windows non serve il suffisso `\bin`; sulla maggior parte dei sistemi è `%AppData%\npm`).
    - Chiudi e riapri PowerShell dopo aver aggiornato il PATH.

    Se vuoi la configurazione Windows più fluida, usa **WSL2** invece di Windows nativo.
    Documentazione: [Windows](/it/platforms/windows).

  </Accordion>

  <Accordion title="L'output exec su Windows mostra testo cinese illeggibile: cosa devo fare?">
    Di solito si tratta di una mancata corrispondenza della code page della console nelle shell Windows native.

    Sintomi:

    - l'output di `system.run`/`exec` rende il cinese come mojibake
    - lo stesso comando appare corretto in un altro profilo terminale

    Soluzione rapida in PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    Poi riavvia il Gateway e riprova il comando:

    ```powershell
    openclaw gateway restart
    ```

    Se riesci ancora a riprodurre il problema sull'ultima versione di OpenClaw, tienilo monitorato/segnalalo qui:

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="La documentazione non ha risposto alla mia domanda: come ottengo una risposta migliore?">
    Usa l'**installazione hackable (git)** per avere localmente il codice sorgente e la documentazione completi, poi chiedi
    al tuo bot (o a Claude/Codex) _da quella cartella_ così potrà leggere il repository e rispondere con precisione.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Maggiori dettagli: [Install](/it/install) e [Flag dell'installer](/it/install/installer).

  </Accordion>

  <Accordion title="Come installo OpenClaw su Linux?">
    Risposta breve: segui la guida Linux, poi esegui l'onboarding.

    - Percorso rapido Linux + installazione del servizio: [Linux](/it/platforms/linux).
    - Procedura completa: [Getting Started](/it/start/getting-started).
    - Installer + aggiornamenti: [Installazione e aggiornamenti](/it/install/updating).

  </Accordion>

  <Accordion title="Come installo OpenClaw su un VPS?">
    Va bene qualsiasi VPS Linux. Installa sul server, poi usa SSH/Tailscale per raggiungere il Gateway.

    Guide: [exe.dev](/it/install/exe-dev), [Hetzner](/it/install/hetzner), [Fly.io](/it/install/fly).
    Accesso remoto: [Gateway remoto](/it/gateway/remote).

  </Accordion>

  <Accordion title="Dove sono le guide di installazione cloud/VPS?">
    Manteniamo un **hub di hosting** con i provider più comuni. Scegline uno e segui la guida:

    - [Hosting VPS](/it/vps) (tutti i provider in un unico posto)
    - [Fly.io](/it/install/fly)
    - [Hetzner](/it/install/hetzner)
    - [exe.dev](/it/install/exe-dev)

    Come funziona nel cloud: il **Gateway gira sul server**, e tu vi accedi
    dal laptop/telefono tramite la Control UI (o Tailscale/SSH). Stato + workspace
    vivono sul server, quindi considera l'host come la fonte di verità e fai il backup.

    Puoi associare **nodi** (Mac/iOS/Android/headless) a quel Gateway cloud per accedere a
    schermo/fotocamera/canvas locali o eseguire comandi sul laptop mantenendo
    il Gateway nel cloud.

    Hub: [Piattaforme](/it/platforms). Accesso remoto: [Gateway remoto](/it/gateway/remote).
    Nodi: [Nodes](/it/nodes), [CLI Nodes](/cli/nodes).

  </Accordion>

  <Accordion title="Posso chiedere a OpenClaw di aggiornarsi da solo?">
    Risposta breve: **possibile, ma non consigliato**. Il flusso di aggiornamento può riavviare il
    Gateway (interrompendo la sessione attiva), può richiedere un checkout git pulito e
    può chiedere una conferma. Più sicuro: eseguire gli aggiornamenti da shell come operatore.

    Usa la CLI:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    Se devi automatizzare da un agente:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    Documentazione: [Update](/cli/update), [Aggiornamento](/it/install/updating).

  </Accordion>

  <Accordion title="Cosa fa davvero l'onboarding?">
    `openclaw onboard` è il percorso di configurazione consigliato. In **modalità locale** ti guida attraverso:

    - **Configurazione modello/autenticazione** (OAuth del provider, chiavi API, setup-token legacy di Anthropic, più opzioni di modelli locali come LM Studio)
    - Posizione del **workspace** + file bootstrap
    - **Impostazioni del Gateway** (bind/porta/autenticazione/tailscale)
    - **Canali** (WhatsApp, Telegram, Discord, Mattermost, Signal, iMessage, più plugin di canale inclusi come QQ Bot)
    - **Installazione del daemon** (LaunchAgent su macOS; unità systemd user su Linux/WSL2)
    - **Controlli di integrità** e selezione delle **Skills**

    Avverte anche se il modello configurato è sconosciuto o se manca l'autenticazione.

  </Accordion>

  <Accordion title="Mi serve un abbonamento Claude o OpenAI per usarlo?">
    No. Puoi eseguire OpenClaw con **chiavi API** (Anthropic/OpenAI/altri) o con
    **modelli solo locali**, così i tuoi dati restano sul tuo dispositivo. Gli abbonamenti (Claude
    Pro/Max o OpenAI Codex) sono modi facoltativi per autenticare quei provider.

    Per Anthropic in OpenClaw, la distinzione pratica è:

    - **Chiave API Anthropic**: normale fatturazione Anthropic API
    - **Autenticazione in OpenClaw con abbonamento Claude**: Anthropic ha comunicato agli utenti OpenClaw il
      **4 aprile 2026 alle 12:00 PM PT / 8:00 PM BST** che questo richiede
      **Extra Usage** fatturato separatamente dall'abbonamento

    Le nostre riproduzioni locali mostrano anche che `claude -p --append-system-prompt ...` può
    incontrare la stessa protezione Extra Usage quando il prompt aggiunto identifica
    OpenClaw, mentre la stessa stringa di prompt **non** riproduce quel blocco sul
    percorso Anthropic SDK + chiave API. OpenAI Codex OAuth è esplicitamente
    supportato per strumenti esterni come OpenClaw.

    OpenClaw supporta anche altre opzioni hosted in stile abbonamento, incluse
    **Qwen Cloud Coding Plan**, **MiniMax Coding Plan** e
    **Z.AI / GLM Coding Plan**.

    Documentazione: [Anthropic](/it/providers/anthropic), [OpenAI](/it/providers/openai),
    [Qwen Cloud](/it/providers/qwen),
    [MiniMax](/it/providers/minimax), [GLM Models](/it/providers/glm),
    [Modelli locali](/it/gateway/local-models), [Modelli](/it/concepts/models).

  </Accordion>

  <Accordion title="Posso usare l'abbonamento Claude Max senza una chiave API?">
    Sì, ma trattalo come **autenticazione con abbonamento Claude con Extra Usage**.

    Gli abbonamenti Claude Pro/Max non includono una chiave API. In OpenClaw, questo
    significa che si applica la nota di fatturazione specifica di Anthropic per OpenClaw: il traffico
    in abbonamento richiede **Extra Usage**. Se vuoi traffico Anthropic senza
    quel percorso Extra Usage, usa invece una chiave API Anthropic.

  </Accordion>

  <Accordion title="Supportate l'autenticazione con abbonamento Claude (Claude Pro o Max)?">
    Sì, ma l'interpretazione supportata ora è:

    - Anthropic in OpenClaw con un abbonamento significa **Extra Usage**
    - Anthropic in OpenClaw senza quel percorso significa **chiave API**

    Il setup-token Anthropic è ancora disponibile come percorso OpenClaw legacy/manuale,
    e la nota di fatturazione specifica di Anthropic per OpenClaw si applica ancora in quel caso. Abbiamo
    riprodotto localmente la stessa protezione di fatturazione anche con l'uso diretto di
    `claude -p --append-system-prompt ...` quando il prompt aggiunto
    identifica OpenClaw, mentre la stessa stringa di prompt **non** si riproduce sul
    percorso Anthropic SDK + chiave API.

    Per carichi di lavoro di produzione o multiutente, l'autenticazione con chiave API Anthropic è la
    scelta più sicura e consigliata. Se vuoi altre opzioni hosted
    in stile abbonamento in OpenClaw, vedi [OpenAI](/it/providers/openai), [Qwen / Model
    Cloud](/it/providers/qwen), [MiniMax](/it/providers/minimax) e
    [GLM Models](/it/providers/glm).

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="Perché vedo HTTP 429 rate_limit_error da Anthropic?">
Significa che la tua **quota/limite di velocità Anthropic** è esaurita per la finestra corrente. Se
usi **Claude CLI**, attendi il reset della finestra o aggiorna il tuo piano. Se
usi una **chiave API Anthropic**, controlla Anthropic Console
per utilizzo/fatturazione e aumenta i limiti se necessario.

    Se il messaggio è specificamente:
    `Extra usage is required for long context requests`, la richiesta sta cercando di usare
    la beta di contesto 1M di Anthropic (`context1m: true`). Funziona solo quando la tua
    credenziale è idonea alla fatturazione long-context (fatturazione con chiave API oppure il
    percorso OpenClaw Claude-login con Extra Usage abilitato).

    Suggerimento: imposta un **modello fallback** così OpenClaw può continuare a rispondere mentre un provider è soggetto a rate limit.
    Vedi [Models](/cli/models), [OAuth](/it/concepts/oauth) e
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/it/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context).

  </Accordion>

  <Accordion title="AWS Bedrock è supportato?">
    Sì. OpenClaw ha un provider incluso **Amazon Bedrock (Converse)**. Con i marker env AWS presenti, OpenClaw può rilevare automaticamente il catalogo Bedrock streaming/testuale e unirlo come provider implicito `amazon-bedrock`; altrimenti puoi abilitare esplicitamente `plugins.entries.amazon-bedrock.config.discovery.enabled` o aggiungere una voce provider manuale. Vedi [Amazon Bedrock](/it/providers/bedrock) e [Provider di modelli](/it/providers/models). Se preferisci un flusso con chiavi gestite, anche un proxy compatibile OpenAI davanti a Bedrock è un'opzione valida.
  </Accordion>

  <Accordion title="Come funziona l'autenticazione Codex?">
    OpenClaw supporta **OpenAI Code (Codex)** tramite OAuth (accesso ChatGPT). L'onboarding può eseguire il flusso OAuth e imposterà il modello predefinito su `openai-codex/gpt-5.4` quando appropriato. Vedi [Provider di modelli](/it/concepts/model-providers) e [Onboarding (CLI)](/it/start/wizard).
  </Accordion>

  <Accordion title="Supportate l'autenticazione con abbonamento OpenAI (Codex OAuth)?">
    Sì. OpenClaw supporta pienamente **OpenAI Code (Codex) subscription OAuth**.
    OpenAI consente esplicitamente l'uso dell'OAuth in abbonamento in strumenti/workflow esterni
    come OpenClaw. L'onboarding può eseguire il flusso OAuth per te.

    Vedi [OAuth](/it/concepts/oauth), [Provider di modelli](/it/concepts/model-providers) e [Onboarding (CLI)](/it/start/wizard).

  </Accordion>

  <Accordion title="Come configuro Gemini CLI OAuth?">
    Gemini CLI usa un **flusso di autenticazione plugin**, non un client id o un secret in `openclaw.json`.

    Usa invece il provider Gemini API:

    1. Abilita il plugin: `openclaw plugins enable google`
    2. Esegui `openclaw onboard --auth-choice gemini-api-key`
    3. Imposta un modello Google come `google/gemini-3.1-pro-preview`

  </Accordion>

  <Accordion title="Un modello locale va bene per chat informali?">
    Di solito no. OpenClaw ha bisogno di contesto ampio + sicurezza robusta; modelli piccoli troncano e perdono contenuto. Se proprio devi, esegui localmente la build di modello **più grande** che puoi (LM Studio) e vedi [/gateway/local-models](/it/gateway/local-models). I modelli più piccoli/quantizzati aumentano il rischio di prompt injection - vedi [Security](/it/gateway/security).
  </Accordion>

  <Accordion title="Come faccio a mantenere il traffico verso i modelli hosted in una regione specifica?">
    Scegli endpoint fissati per regione. OpenRouter espone opzioni ospitate negli Stati Uniti per MiniMax, Kimi e GLM; scegli la variante ospitata negli USA per mantenere i dati nella regione. Puoi comunque elencare Anthropic/OpenAI insieme a questi usando `models.mode: "merge"` così i fallback restano disponibili pur rispettando il provider con regione specifica che selezioni.
  </Accordion>

  <Accordion title="Devo comprare un Mac Mini per installarlo?">
    No. OpenClaw gira su macOS o Linux (Windows tramite WSL2). Un Mac mini è facoltativo - alcune persone
    ne acquistano uno come host sempre attivo, ma vanno bene anche un piccolo VPS, un server domestico o una macchina di classe Raspberry Pi.

    Ti serve un Mac **solo per gli strumenti esclusivi di macOS**. Per iMessage, usa [BlueBubbles](/it/channels/bluebubbles) (consigliato) - il server BlueBubbles gira su qualsiasi Mac, e il Gateway può girare su Linux o altrove. Se vuoi altri strumenti solo macOS, esegui il Gateway su un Mac o associa un nodo macOS.

    Documentazione: [BlueBubbles](/it/channels/bluebubbles), [Nodes](/it/nodes), [Mac remote mode](/it/platforms/mac/remote).

  </Accordion>

  <Accordion title="Mi serve un Mac mini per il supporto iMessage?">
    Ti serve **un dispositivo macOS** connesso a Messages. **Non** deve essere per forza un Mac mini -
    va bene qualsiasi Mac. **Usa [BlueBubbles](/it/channels/bluebubbles)** (consigliato) per iMessage - il server BlueBubbles gira su macOS, mentre il Gateway può girare su Linux o altrove.

    Configurazioni comuni:

    - Esegui il Gateway su Linux/VPS ed esegui il server BlueBubbles su qualsiasi Mac connesso a Messages.
    - Esegui tutto sul Mac se vuoi la configurazione più semplice su una sola macchina.

    Documentazione: [BlueBubbles](/it/channels/bluebubbles), [Nodes](/it/nodes),
    [Mac remote mode](/it/platforms/mac/remote).

  </Accordion>

  <Accordion title="Se compro un Mac mini per eseguire OpenClaw, posso collegarlo al mio MacBook Pro?">
    Sì. Il **Mac mini può eseguire il Gateway**, e il tuo MacBook Pro può collegarsi come
    **nodo** (dispositivo companion). I nodi non eseguono il Gateway - forniscono funzionalità extra
    come schermo/fotocamera/canvas e `system.run` su quel dispositivo.

    Schema comune:

    - Gateway sul Mac mini (sempre attivo).
    - Il MacBook Pro esegue l'app macOS o un host nodo e si associa al Gateway.
    - Usa `openclaw nodes status` / `openclaw nodes list` per vederlo.

    Documentazione: [Nodes](/it/nodes), [CLI Nodes](/cli/nodes).

  </Accordion>

  <Accordion title="Posso usare Bun?">
    Bun **non è consigliato**. Osserviamo bug runtime, soprattutto con WhatsApp e Telegram.
    Usa **Node** per gateway stabili.

    Se vuoi comunque sperimentare con Bun, fallo su un gateway non di produzione
    senza WhatsApp/Telegram.

  </Accordion>

  <Accordion title="Telegram: cosa va in allowFrom?">
    `channels.telegram.allowFrom` è **l'ID utente Telegram numerico del mittente umano**. Non è il nome utente del bot.

    L'onboarding accetta input `@username` e lo risolve in un ID numerico, ma l'autorizzazione OpenClaw usa solo ID numerici.

    Più sicuro (nessun bot di terze parti):

    - Invia un DM al tuo bot, poi esegui `openclaw logs --follow` e leggi `from.id`.

    Bot API ufficiale:

    - Invia un DM al tuo bot, poi chiama `https://api.telegram.org/bot<bot_token>/getUpdates` e leggi `message.from.id`.

    Terze parti (meno privato):

    - Invia un DM a `@userinfobot` o `@getidsbot`.

    Vedi [/channels/telegram](/it/channels/telegram#access-control-and-activation).

  </Accordion>

  <Accordion title="Più persone possono usare un numero WhatsApp con istanze OpenClaw diverse?">
    Sì, tramite **instradamento multi-agent**. Associa il **DM** WhatsApp di ogni mittente (peer `kind: "direct"`, mittente E.164 come `+15551234567`) a un diverso `agentId`, così ogni persona ottiene il proprio workspace e archivio delle sessioni. Le risposte continueranno comunque a provenire dallo **stesso account WhatsApp**, e il controllo di accesso ai DM (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) è globale per account WhatsApp. Vedi [Instradamento Multi-Agent](/it/concepts/multi-agent) e [WhatsApp](/it/channels/whatsapp).
  </Accordion>

  <Accordion title='Posso eseguire un agente "chat veloce" e un agente "Opus per coding"?'>
    Sì. Usa l'instradamento multi-agent: assegna a ogni agente il proprio modello predefinito, poi associa le route in ingresso (account provider o peer specifici) a ciascun agente. Una configurazione di esempio si trova in [Instradamento Multi-Agent](/it/concepts/multi-agent). Vedi anche [Modelli](/it/concepts/models) e [Configurazione](/it/gateway/configuration).
  </Accordion>

  <Accordion title="Homebrew funziona su Linux?">
    Sì. Homebrew supporta Linux (Linuxbrew). Configurazione rapida:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    Se esegui OpenClaw tramite systemd, assicurati che il PATH del servizio includa `/home/linuxbrew/.linuxbrew/bin` (o il prefisso brew in uso) così gli strumenti installati con `brew` vengono risolti nelle shell non di login.
    Le build recenti antepongono anche directory bin utente comuni nei servizi Linux systemd (per esempio `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`) e rispettano `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR` e `FNM_DIR` quando impostati.

  </Accordion>

  <Accordion title="Differenza tra l'installazione git hackable e npm install">
    - **Installazione hackable (git):** checkout completo del sorgente, modificabile, ideale per i collaboratori.
      Esegui le build localmente e puoi modificare codice/documentazione.
    - **npm install:** installazione globale della CLI, senza repository, ideale per "eseguirlo e basta".
      Gli aggiornamenti arrivano dai dist-tag npm.

    Documentazione: [Getting started](/it/start/getting-started), [Aggiornamento](/it/install/updating).

  </Accordion>

  <Accordion title="Posso passare più tardi da installazioni npm a git e viceversa?">
    Sì. Installa l'altra variante, poi esegui Doctor in modo che il servizio gateway punti al nuovo entrypoint.
    Questo **non elimina i tuoi dati** - cambia solo l'installazione del codice OpenClaw. Il tuo stato
    (`~/.openclaw`) e il workspace (`~/.openclaw/workspace`) restano intatti.

    Da npm a git:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    openclaw doctor
    openclaw gateway restart
    ```

    Da git a npm:

    ```bash
    npm install -g openclaw@latest
    openclaw doctor
    openclaw gateway restart
    ```

    Doctor rileva una mancata corrispondenza dell'entrypoint del servizio gateway e propone di riscrivere la configurazione del servizio per adattarla all'installazione corrente (usa `--repair` in automazione).

    Suggerimenti per il backup: vedi [Strategia di backup](#dove-si-trovano-le-cose-su-disco).

  </Accordion>

  <Accordion title="Dovrei eseguire il Gateway sul laptop o su un VPS?">
    Risposta breve: **se vuoi affidabilità 24/7, usa un VPS**. Se vuoi il
    minor attrito e ti vanno bene sleep/riavvii, eseguilo in locale.

    **Laptop (Gateway locale)**

    - **Pro:** nessun costo server, accesso diretto ai file locali, finestra browser visibile.
    - **Contro:** sleep/interruzioni di rete = disconnessioni, aggiornamenti/riavvii del sistema operativo interrompono, deve restare sveglio.

    **VPS / cloud**

    - **Pro:** sempre attivo, rete stabile, nessun problema di sleep del laptop, più facile da mantenere in esecuzione.
    - **Contro:** spesso gira headless (usa screenshot), accesso solo remoto ai file, devi usare SSH per gli aggiornamenti.

    **Nota specifica di OpenClaw:** WhatsApp/Telegram/Slack/Mattermost/Discord funzionano tutti bene da un VPS. L'unico vero compromesso è **browser headless** rispetto a una finestra visibile. Vedi [Browser](/it/tools/browser).

    **Scelta predefinita consigliata:** VPS se hai già avuto disconnessioni del gateway. Il locale è ottimo quando stai usando attivamente il Mac e vuoi accesso ai file locali o automazione UI con un browser visibile.

  </Accordion>

  <Accordion title="Quanto è importante eseguire OpenClaw su una macchina dedicata?">
    Non è obbligatorio, ma è **consigliato per affidabilità e isolamento**.

    - **Host dedicato (VPS/Mac mini/Pi):** sempre attivo, meno interruzioni da sleep/riavvio, permessi più puliti, più facile da mantenere attivo.
    - **Laptop/desktop condiviso:** va benissimo per test e uso attivo, ma aspettati pause quando la macchina va in sleep o si aggiorna.

    Se vuoi il meglio dei due mondi, tieni il Gateway su un host dedicato e associa il laptop come **nodo** per strumenti locali di schermo/fotocamera/exec. Vedi [Nodes](/it/nodes).
    Per la guida sulla sicurezza, leggi [Security](/it/gateway/security).

  </Accordion>

  <Accordion title="Quali sono i requisiti minimi di un VPS e il sistema operativo consigliato?">
    OpenClaw è leggero. Per un Gateway di base + un canale chat:

    - **Minimo assoluto:** 1 vCPU, 1GB RAM, ~500MB disco.
    - **Consigliato:** 1-2 vCPU, 2GB RAM o più per avere margine (log, media, più canali). Gli strumenti Node e l'automazione browser possono richiedere molte risorse.

    OS: usa **Ubuntu LTS** (o qualsiasi Debian/Ubuntu moderno). Il percorso di installazione Linux è quello testato meglio.

    Documentazione: [Linux](/it/platforms/linux), [Hosting VPS](/it/vps).

  </Accordion>

  <Accordion title="Posso eseguire OpenClaw in una VM e quali sono i requisiti?">
    Sì. Tratta una VM come un VPS: deve essere sempre attiva, raggiungibile e avere RAM sufficiente
    per il Gateway e gli eventuali canali che abiliti.

    Indicazioni di base:

    - **Minimo assoluto:** 1 vCPU, 1GB RAM.
    - **Consigliato:** 2GB RAM o più se esegui più canali, automazione browser o strumenti media.
    - **OS:** Ubuntu LTS o un altro Debian/Ubuntu moderno.

    Se usi Windows, **WSL2 è la configurazione in stile VM più semplice** e offre la compatibilità
    migliore con gli strumenti. Vedi [Windows](/it/platforms/windows), [Hosting VPS](/it/vps).
    Se stai eseguendo macOS in una VM, vedi [VM macOS](/it/install/macos-vm).

  </Accordion>
</AccordionGroup>

## Cos'è OpenClaw?

<AccordionGroup>
  <Accordion title="Che cos'è OpenClaw, in un paragrafo?">
    OpenClaw è un assistente AI personale che esegui sui tuoi dispositivi. Risponde sulle superfici di messaggistica che già usi (WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat e plugin di canale inclusi come QQ Bot) e può anche gestire voce + un Canvas live sulle piattaforme supportate. Il **Gateway** è il control plane sempre attivo; l'assistente è il prodotto.
  </Accordion>

  <Accordion title="Proposta di valore">
    OpenClaw non è "solo un wrapper per Claude". È un **control plane local-first** che ti permette di eseguire un
    assistente capace sul **tuo hardware**, raggiungibile dalle app di chat che già usi, con
    sessioni con stato, memoria e strumenti - senza cedere il controllo dei tuoi workflow a un
    SaaS hosted.

    Punti salienti:

    - **I tuoi dispositivi, i tuoi dati:** esegui il Gateway dove vuoi (Mac, Linux, VPS) e mantieni
      locali workspace + cronologia delle sessioni.
    - **Canali reali, non una sandbox web:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage/ecc.,
      più voce mobile e Canvas sulle piattaforme supportate.
    - **Agnostico rispetto al modello:** usa Anthropic, OpenAI, MiniMax, OpenRouter, ecc., con instradamento
      per agente e failover.
    - **Opzione solo locale:** esegui modelli locali così **tutti i dati possono restare sul tuo dispositivo** se vuoi.
    - **Instradamento multi-agent:** agenti separati per canale, account o attività, ognuno con il proprio
      workspace e i propri valori predefiniti.
    - **Open source e hackable:** ispeziona, estendi e self-host senza vendor lock-in.

    Documentazione: [Gateway](/it/gateway), [Canali](/it/channels), [Multi-agent](/it/concepts/multi-agent),
    [Memory](/it/concepts/memory).

  </Accordion>

  <Accordion title="L'ho appena configurato - cosa dovrei fare per prima cosa?">
    Buoni primi progetti:

    - Costruire un sito web (WordPress, Shopify o un semplice sito statico).
    - Creare il prototipo di un'app mobile (struttura, schermate, piano API).
    - Organizzare file e cartelle (pulizia, nomi, tag).
    - Collegare Gmail e automatizzare riepiloghi o follow-up.

    Può gestire attività grandi, ma funziona meglio se le dividi in fasi e
    usi sub agent per il lavoro parallelo.

  </Accordion>

  <Accordion title="Quali sono i cinque casi d'uso quotidiani principali di OpenClaw?">
    I vantaggi quotidiani di solito sono:

    - **Briefing personali:** riepiloghi di inbox, calendario e notizie che ti interessano.
    - **Ricerca e stesura:** ricerche rapide, riepiloghi e prime bozze per email o documenti.
    - **Promemoria e follow-up:** solleciti e checklist guidati da cron o heartbeat.
    - **Automazione del browser:** compilazione di moduli, raccolta di dati e ripetizione di attività web.
    - **Coordinamento tra dispositivi:** invia un'attività dal telefono, lascia che il Gateway la esegua su un server e ricevi il risultato in chat.

  </Accordion>

  <Accordion title="OpenClaw può aiutare con lead generation, outreach, annunci e blog per un SaaS?">
    Sì per **ricerca, qualificazione e stesura**. Può analizzare siti, costruire shortlist,
    riassumere prospect e scrivere bozze di outreach o copy per annunci.

    Per **outreach o campagne pubblicitarie**, mantieni una persona nel loop. Evita lo spam, rispetta le leggi locali e
    le policy della piattaforma e controlla tutto prima dell'invio. Il modello più sicuro è lasciare che
    OpenClaw prepari la bozza e che tu approvi.

    Documentazione: [Security](/it/gateway/security).

  </Accordion>

  <Accordion title="Quali sono i vantaggi rispetto a Claude Code per lo sviluppo web?">
    OpenClaw è un **assistente personale** e un livello di coordinamento, non un sostituto dell'IDE. Usa
    Claude Code o Codex per il ciclo di coding diretto più veloce in un repository. Usa OpenClaw quando
    vuoi memoria durevole, accesso cross-device e orchestrazione degli strumenti.

    Vantaggi:

    - **Memoria + workspace persistenti** tra sessioni
    - **Accesso multipiattaforma** (WhatsApp, Telegram, TUI, WebChat)
    - **Orchestrazione degli strumenti** (browser, file, scheduling, hook)
    - **Gateway sempre attivo** (esegui su un VPS, interagisci da ovunque)
    - **Nodes** per browser/schermo/fotocamera/exec locali

    Showcase: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills e automazione

<AccordionGroup>
  <Accordion title="Come posso personalizzare le Skills senza mantenere il repository sporco?">
    Usa override gestiti invece di modificare la copia nel repository. Inserisci le tue modifiche in `~/.openclaw/skills/<name>/SKILL.md` (oppure aggiungi una cartella tramite `skills.load.extraDirs` in `~/.openclaw/openclaw.json`). La precedenza è `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → bundled → `skills.load.extraDirs`, quindi gli override gestiti hanno comunque la precedenza sulle Skills incluse senza toccare git. Se ti serve la Skill installata globalmente ma visibile solo ad alcuni agenti, tieni la copia condivisa in `~/.openclaw/skills` e controlla la visibilità con `agents.defaults.skills` e `agents.list[].skills`. Solo le modifiche degne di upstream dovrebbero vivere nel repository ed essere inviate come PR.
  </Accordion>

  <Accordion title="Posso caricare Skills da una cartella personalizzata?">
    Sì. Aggiungi directory extra tramite `skills.load.extraDirs` in `~/.openclaw/openclaw.json` (precedenza più bassa). La precedenza predefinita è `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → bundled → `skills.load.extraDirs`. `clawhub` installa per impostazione predefinita in `./skills`, che OpenClaw considera come `<workspace>/skills` nella sessione successiva. Se la Skill deve essere visibile solo a determinati agenti, abbinala a `agents.defaults.skills` o `agents.list[].skills`.
  </Accordion>

  <Accordion title="Come posso usare modelli diversi per attività diverse?">
    Oggi i pattern supportati sono:

    - **Cron job**: i job isolati possono impostare un override `model` per job.
    - **Sub-agent**: instrada le attività verso agenti separati con modelli predefiniti diversi.
    - **Cambio su richiesta**: usa `/model` per cambiare il modello della sessione corrente in qualsiasi momento.

    Vedi [Cron job](/it/automation/cron-jobs), [Instradamento Multi-Agent](/it/concepts/multi-agent) e [Comandi slash](/it/tools/slash-commands).

  </Accordion>

  <Accordion title="Il bot si blocca mentre esegue lavori pesanti. Come posso scaricarli altrove?">
    Usa i **sub-agent** per attività lunghe o parallele. I sub-agent vengono eseguiti nella propria sessione,
    restituiscono un riepilogo e mantengono reattiva la chat principale.

    Chiedi al tuo bot di "creare un sub-agent per questa attività" oppure usa `/subagents`.
    Usa `/status` in chat per vedere cosa sta facendo il Gateway in questo momento (e se è occupato).

    Suggerimento sui token: sia le attività lunghe sia i sub-agent consumano token. Se il costo è un problema, imposta un
    modello più economico per i sub-agent tramite `agents.defaults.subagents.model`.

    Documentazione: [Sub-agent](/it/tools/subagents), [Attività in background](/it/automation/tasks).

  </Accordion>

  <Accordion title="Come funzionano su Discord le sessioni subagent associate a un thread?">
    Usa i binding dei thread. Puoi associare un thread Discord a un target subagent o sessione in modo che i messaggi successivi in quel thread restino su quella sessione associata.

    Flusso di base:

    - Crea con `sessions_spawn` usando `thread: true` (e facoltativamente `mode: "session"` per follow-up persistenti).
    - Oppure associa manualmente con `/focus <target>`.
    - Usa `/agents` per ispezionare lo stato del binding.
    - Usa `/session idle <duration|off>` e `/session max-age <duration|off>` per controllare l'auto-unfocus.
    - Usa `/unfocus` per staccare il thread.

    Configurazione richiesta:

    - Valori predefiniti globali: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
    - Override Discord: `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`.
    - Auto-bind alla creazione: imposta `channels.discord.threadBindings.spawnSubagentSessions: true`.

    Documentazione: [Sub-agent](/it/tools/subagents), [Discord](/it/channels/discord), [Riferimento della configurazione](/it/gateway/configuration-reference), [Comandi slash](/it/tools/slash-commands).

  </Accordion>

  <Accordion title="Un subagent ha terminato, ma l'aggiornamento di completamento è andato nel posto sbagliato o non è mai stato pubblicato. Cosa dovrei controllare?">
    Controlla prima la route del richiedente risolta:

    - La consegna del subagent in modalità completamento preferisce qualsiasi thread associato o route di conversazione quando esiste.
    - Se l'origine del completamento porta solo un canale, OpenClaw usa come fallback la route memorizzata della sessione del richiedente (`lastChannel` / `lastTo` / `lastAccountId`) così la consegna diretta può comunque riuscire.
    - Se non esistono né una route associata né una route memorizzata utilizzabile, la consegna diretta può fallire e il risultato ricade sulla consegna in coda alla sessione invece di essere pubblicato subito in chat.
    - Target non validi o obsoleti possono comunque forzare il fallback in coda o il fallimento della consegna finale.
    - Se l'ultima risposta visibile dell'assistente figlio è esattamente il token silenzioso `NO_REPLY` / `no_reply`, oppure esattamente `ANNOUNCE_SKIP`, OpenClaw sopprime intenzionalmente l'annuncio invece di pubblicare progressi precedenti obsoleti.
    - Se il figlio è andato in timeout dopo sole chiamate di strumenti, l'annuncio può comprimere tutto questo in un breve riepilogo del progresso parziale invece di riprodurre l'output grezzo degli strumenti.

    Debug:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentazione: [Sub-agent](/it/tools/subagents), [Attività in background](/it/automation/tasks), [Strumenti di sessione](/it/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron o i promemoria non scattano. Cosa dovrei controllare?">
    Cron gira all'interno del processo Gateway. Se il Gateway non è in esecuzione continua,
    i job pianificati non verranno eseguiti.

    Checklist:

    - Conferma che cron sia abilitato (`cron.enabled`) e che `OPENCLAW_SKIP_CRON` non sia impostato.
    - Verifica che il Gateway sia in esecuzione 24/7 (senza sleep/riavvii).
    - Verifica le impostazioni del fuso orario per il job (`--tz` rispetto al fuso orario dell'host).

    Debug:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Documentazione: [Cron job](/it/automation/cron-jobs), [Automazione e attività](/it/automation).

  </Accordion>

  <Accordion title="Cron è scattato, ma non è stato inviato nulla al canale. Perché?">
    Controlla prima la modalità di consegna:

    - `--no-deliver` / `delivery.mode: "none"` significa che non è previsto alcun messaggio esterno.
    - Target di annuncio mancante o non valido (`channel` / `to`) significa che il runner ha saltato la consegna in uscita.
    - Errori di autenticazione del canale (`unauthorized`, `Forbidden`) significano che il runner ha provato a consegnare ma le credenziali l'hanno bloccato.
    - Un risultato isolato silenzioso (`NO_REPLY` / `no_reply` soltanto) viene trattato come intenzionalmente non consegnabile, quindi il runner sopprime anche la consegna di fallback in coda.

    Per i cron job isolati, il runner possiede la consegna finale. Ci si aspetta che l'agente
    restituisca un riepilogo in testo semplice che il runner possa inviare. `--no-deliver` mantiene
    quel risultato interno; non consente invece all'agente di inviare direttamente con lo
    strumento message.

    Debug:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentazione: [Cron job](/it/automation/cron-jobs), [Attività in background](/it/automation/tasks).

  </Accordion>

  <Accordion title="Perché un'esecuzione cron isolata ha cambiato modello o ha riprovato una volta?">
    Di solito è il percorso di live model-switch, non una pianificazione duplicata.

    Cron isolato può rendere persistente un handoff runtime del modello e riprovare quando l'esecuzione attiva
    genera `LiveSessionModelSwitchError`. Il retry mantiene il provider/modello cambiato
    e, se il cambio includeva un nuovo override del profilo di autenticazione, cron
    rende persistente anche quello prima del retry.

    Regole di selezione correlate:

    - L'override del modello del hook Gmail vince per primo quando applicabile.
    - Poi il `model` per job.
    - Poi qualsiasi override del modello memorizzato nella sessione cron.
    - Poi la normale selezione del modello agent/default.

    Il ciclo di retry è limitato. Dopo il tentativo iniziale più 2 retry di cambio modello,
    cron interrompe invece di continuare all'infinito.

    Debug:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    Documentazione: [Cron job](/it/automation/cron-jobs), [CLI cron](/cli/cron).

  </Accordion>

  <Accordion title="Come installo le Skills su Linux?">
    Usa i comandi nativi `openclaw skills` o inserisci le Skills nel tuo workspace. L'interfaccia macOS Skills UI non è disponibile su Linux.
    Sfoglia le Skills su [https://clawhub.ai](https://clawhub.ai).

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    Il comando nativo `openclaw skills install` scrive nella directory `skills/`
    del workspace attivo. Installa la CLI separata `clawhub` solo se vuoi pubblicare o
    sincronizzare le tue Skills. Per installazioni condivise tra agenti, inserisci la Skill in
    `~/.openclaw/skills` e usa `agents.defaults.skills` oppure
    `agents.list[].skills` se vuoi limitare quali agenti possono vederla.

  </Accordion>

  <Accordion title="OpenClaw può eseguire attività secondo una pianificazione o continuamente in background?">
    Sì. Usa lo scheduler del Gateway:

    - **Cron job** per attività pianificate o ricorrenti (persistono tra i riavvii).
    - **Heartbeat** per controlli periodici della "sessione principale".
    - **Job isolati** per agenti autonomi che pubblicano riepiloghi o consegnano alle chat.

    Documentazione: [Cron job](/it/automation/cron-jobs), [Automazione e attività](/it/automation),
    [Heartbeat](/it/gateway/heartbeat).

  </Accordion>

  <Accordion title="Posso eseguire da Linux Skills Apple/macOS-only?">
    Non direttamente. Le Skills macOS sono controllate da `metadata.openclaw.os` più i binari richiesti, e le Skills compaiono nel system prompt solo quando sono idonee sull'**host Gateway**. Su Linux, le Skills `darwin`-only (come `apple-notes`, `apple-reminders`, `things-mac`) non verranno caricate a meno che tu non sovrascriva il gating.

    Hai tre pattern supportati:

    **Opzione A - eseguire il Gateway su un Mac (più semplice).**
    Esegui il Gateway dove esistono i binari macOS, poi collegati da Linux in [modalità remota](#gateway-porte-già-in-esecuzione-e-modalità-remota) o tramite Tailscale. Le Skills si caricano normalmente perché l'host del Gateway è macOS.

    **Opzione B - usare un nodo macOS (senza SSH).**
    Esegui il Gateway su Linux, associa un nodo macOS (app menubar) e imposta **Node Run Commands** su "Always Ask" o "Always Allow" sul Mac. OpenClaw può trattare le Skills solo-macOS come idonee quando i binari richiesti esistono sul nodo. L'agente esegue quelle Skills tramite lo strumento `nodes`. Se scegli "Always Ask", approvare "Always Allow" nella richiesta aggiunge quel comando alla allowlist.

    **Opzione C - fare proxy dei binari macOS via SSH (avanzato).**
    Tieni il Gateway su Linux, ma fai in modo che i binari CLI richiesti vengano risolti in wrapper SSH che girano su un Mac. Poi sovrascrivi la Skill per consentire Linux in modo che resti idonea.

    1. Crea un wrapper SSH per il binario (esempio: `memo` per Apple Notes):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Inserisci il wrapper nel `PATH` dell'host Linux (per esempio `~/bin/memo`).
    3. Sovrascrivi i metadata della Skill (workspace o `~/.openclaw/skills`) per consentire Linux:

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. Avvia una nuova sessione in modo che lo snapshot delle Skills venga aggiornato.

  </Accordion>

  <Accordion title="Avete un'integrazione Notion o HeyGen?">
    Non integrata nativamente al momento.

    Opzioni:

    - **Skill / plugin personalizzato:** soluzione migliore per un accesso API affidabile (sia Notion sia HeyGen hanno API).
    - **Automazione del browser:** funziona senza codice ma è più lenta e fragile.

    Se vuoi mantenere il contesto per cliente (workflow da agenzia), uno schema semplice è:

    - Una pagina Notion per cliente (contesto + preferenze + lavoro attivo).
    - Chiedere all'agente di recuperare quella pagina all'inizio di una sessione.

    Se vuoi un'integrazione nativa, apri una richiesta di funzionalità o crea una Skill
    mirata a quelle API.

    Installa le Skills:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    Le installazioni native finiscono nella directory `skills/` del workspace attivo. Per Skills condivise tra agenti, inseriscile in `~/.openclaw/skills/<name>/SKILL.md`. Se solo alcuni agenti devono vedere un'installazione condivisa, configura `agents.defaults.skills` o `agents.list[].skills`. Alcune Skills si aspettano binari installati tramite Homebrew; su Linux questo significa Linuxbrew (vedi la voce FAQ Homebrew Linux sopra). Vedi [Skills](/it/tools/skills), [Configurazione Skills](/it/tools/skills-config) e [ClawHub](/it/tools/clawhub).

  </Accordion>

  <Accordion title="Come uso il mio Chrome già autenticato con OpenClaw?">
    Usa il profilo browser integrato `user`, che si collega tramite Chrome DevTools MCP:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Se vuoi un nome personalizzato, crea un profilo MCP esplicito:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Questo percorso è locale rispetto all'host. Se il Gateway gira altrove, esegui un node host sulla macchina del browser oppure usa CDP remoto.

    Limiti attuali di `existing-session` / `user`:

    - le azioni sono basate su ref, non su selettori CSS
    - gli upload richiedono `ref` / `inputRef` e attualmente supportano un solo file alla volta
    - `responsebody`, export PDF, intercettazione dei download e azioni batch richiedono ancora un browser gestito o un profilo CDP grezzo

  </Accordion>
</AccordionGroup>

## Sandboxing e memoria

<AccordionGroup>
  <Accordion title="Esiste una documentazione dedicata al sandboxing?">
    Sì. Vedi [Sandboxing](/it/gateway/sandboxing). Per la configurazione specifica di Docker (gateway completo in Docker o immagini sandbox), vedi [Docker](/it/install/docker).
  </Accordion>

  <Accordion title="Docker sembra limitato - come abilito tutte le funzionalità?">
    L'immagine predefinita privilegia la sicurezza ed esegue come utente `node`, quindi non
    include pacchetti di sistema, Homebrew o browser inclusi. Per una configurazione più completa:

    - Rendi persistente `/home/node` con `OPENCLAW_HOME_VOLUME` così le cache sopravvivono.
    - Inserisci nell'immagine le dipendenze di sistema con `OPENCLAW_DOCKER_APT_PACKAGES`.
    - Installa i browser Playwright tramite la CLI inclusa:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - Imposta `PLAYWRIGHT_BROWSERS_PATH` e assicurati che il percorso sia persistente.

    Documentazione: [Docker](/it/install/docker), [Browser](/it/tools/browser).

  </Accordion>

  <Accordion title="Posso mantenere i DM personali ma rendere pubblici/in sandbox i gruppi con un solo agente?">
    Sì - se il tuo traffico privato è nei **DM** e quello pubblico è nei **gruppi**.

    Usa `agents.defaults.sandbox.mode: "non-main"` così le sessioni gruppo/canale (chiavi non-main) girano in Docker, mentre la sessione DM principale resta sull'host. Poi limita gli strumenti disponibili nelle sessioni sandbox tramite `tools.sandbox.tools`.

    Procedura + configurazione di esempio: [Gruppi: DM personali + gruppi pubblici](/it/channels/groups#pattern-personal-dms-public-groups-single-agent)

    Riferimento chiavi di configurazione: [Configurazione Gateway](/it/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="Come faccio ad associare una cartella host nel sandbox?">
    Imposta `agents.defaults.sandbox.docker.binds` su `["host:path:mode"]` (es. `"/home/user/src:/src:ro"`). I bind globali + per agente vengono uniti; i bind per agente vengono ignorati quando `scope: "shared"`. Usa `:ro` per tutto ciò che è sensibile e ricorda che i bind aggirano le pareti del filesystem sandbox.

    OpenClaw convalida le sorgenti dei bind sia rispetto al percorso normalizzato sia rispetto al percorso canonico risolto tramite l'antenato esistente più profondo. Questo significa che le fughe tramite symlink-parent continuano a fallire in modalità closed anche quando l'ultimo segmento del percorso non esiste ancora, e che i controlli di root consentita continuano ad applicarsi dopo la risoluzione dei symlink.

    Vedi [Sandboxing](/it/gateway/sandboxing#custom-bind-mounts) e [Sandbox vs Tool Policy vs Elevated](/it/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) per esempi e note di sicurezza.

  </Accordion>

  <Accordion title="Come funziona la memoria?">
    La memoria di OpenClaw consiste semplicemente in file Markdown nel workspace dell'agente:

    - Note giornaliere in `memory/YYYY-MM-DD.md`
    - Note curate a lungo termine in `MEMORY.md` (solo sessioni principali/private)

    OpenClaw esegue anche uno **scarico silenzioso della memoria prima della compattazione** per ricordare al modello
    di scrivere note durevoli prima dell'auto-compattazione. Questo avviene solo quando il workspace
    è scrivibile (i sandbox in sola lettura lo saltano). Vedi [Memory](/it/concepts/memory).

  </Accordion>

  <Accordion title="La memoria continua a dimenticare le cose. Come faccio a farle restare?">
    Chiedi al bot di **scrivere il fatto nella memoria**. Le note a lungo termine vanno in `MEMORY.md`,
    il contesto a breve termine in `memory/YYYY-MM-DD.md`.

    È ancora un'area che stiamo migliorando. Aiuta ricordare al modello di archiviare i ricordi;
    saprà cosa fare. Se continua a dimenticare, verifica che il Gateway stia usando lo stesso
    workspace a ogni esecuzione.

    Documentazione: [Memory](/it/concepts/memory), [Workspace dell'agente](/it/concepts/agent-workspace).

  </Accordion>

  <Accordion title="La memoria persiste per sempre? Quali sono i limiti?">
    I file di memoria vivono su disco e persistono finché non li elimini. Il limite è il tuo
    spazio di archiviazione, non il modello. Il **contesto di sessione** è comunque limitato dalla
    finestra di contesto del modello, quindi conversazioni molto lunghe possono essere compattate o troncate. Per questo
    esiste la ricerca nella memoria: riporta nel contesto solo le parti pertinenti.

    Documentazione: [Memory](/it/concepts/memory), [Contesto](/it/concepts/context).

  </Accordion>

  <Accordion title="La ricerca semantica nella memoria richiede una chiave API OpenAI?">
    Solo se usi **OpenAI embeddings**. Codex OAuth copre chat/completions e
    **non** concede accesso agli embeddings, quindi **accedere con Codex (OAuth o il
    login della CLI Codex)** non aiuta per la ricerca semantica nella memoria. Gli embeddings OpenAI
    richiedono comunque una vera chiave API (`OPENAI_API_KEY` o `models.providers.openai.apiKey`).

    Se non imposti esplicitamente un provider, OpenClaw seleziona automaticamente un provider quando
    riesce a risolvere una chiave API (profili di autenticazione, `models.providers.*.apiKey` o variabili env).
    Preferisce OpenAI se riesce a risolvere una chiave OpenAI, altrimenti Gemini se una chiave Gemini
    viene risolta, poi Voyage, poi Mistral. Se non è disponibile alcuna chiave remota, la ricerca nella memoria
    resta disabilitata finché non la configuri. Se hai un percorso di modello locale
    configurato e presente, OpenClaw
    preferisce `local`. Ollama è supportato quando imposti esplicitamente
    `memorySearch.provider = "ollama"`.

    Se preferisci restare in locale, imposta `memorySearch.provider = "local"` (e facoltativamente
    `memorySearch.fallback = "none"`). Se vuoi gli embeddings Gemini, imposta
    `memorySearch.provider = "gemini"` e fornisci `GEMINI_API_KEY` (oppure
    `memorySearch.remote.apiKey`). Supportiamo modelli di embedding **OpenAI, Gemini, Voyage, Mistral, Ollama o local**
    - vedi [Memory](/it/concepts/memory) per i dettagli di configurazione.

  </Accordion>
</AccordionGroup>

## Dove si trovano le cose su disco

<AccordionGroup>
  <Accordion title="Tutti i dati usati con OpenClaw vengono salvati localmente?">
    No - **lo stato di OpenClaw è locale**, ma **i servizi esterni vedono comunque ciò che invii loro**.

    - **Locale per impostazione predefinita:** sessioni, file di memoria, configurazione e workspace vivono sull'host Gateway
      (`~/.openclaw` + la directory del tuo workspace).
    - **Remoto per necessità:** i messaggi che invii ai provider di modelli (Anthropic/OpenAI/ecc.) vanno
      alle loro API e le piattaforme chat (WhatsApp/Telegram/Slack/ecc.) archiviano i dati dei messaggi sui propri
      server.
    - **Controlli tu l'impronta:** usando modelli locali, i prompt restano sulla tua macchina, ma il
      traffico dei canali passa comunque attraverso i server del canale.

    Correlati: [Workspace dell'agente](/it/concepts/agent-workspace), [Memory](/it/concepts/memory).

  </Accordion>

  <Accordion title="Dove salva i propri dati OpenClaw?">
    Tutto vive sotto `$OPENCLAW_STATE_DIR` (predefinito: `~/.openclaw`):

    | Path                                                            | Scopo                                                              |
    | --------------------------------------------------------------- | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | Configurazione principale (JSON5)                                  |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | Import OAuth legacy (copiato nei profili auth al primo utilizzo)   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | Profili di autenticazione (OAuth, chiavi API e `keyRef`/`tokenRef` facoltativi) |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | Payload secret facoltativo supportato da file per provider SecretRef `file` |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | File di compatibilità legacy (voci statiche `api_key` ripulite)    |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | Stato del provider (es. `whatsapp/<accountId>/creds.json`)         |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | Stato per agente (agentDir + sessioni)                             |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | Cronologia e stato delle conversazioni (per agente)                |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | Metadati delle sessioni (per agente)                               |

    Percorso legacy single-agent: `~/.openclaw/agent/*` (migrato da `openclaw doctor`).

    Il tuo **workspace** (`AGENTS.md`, file di memoria, Skills, ecc.) è separato e configurato tramite `agents.defaults.workspace` (predefinito: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="Dove dovrebbero trovarsi AGENTS.md / SOUL.md / USER.md / MEMORY.md?">
    Questi file vivono nel **workspace dell'agente**, non in `~/.openclaw`.

    - **Workspace (per agente)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md` (o il fallback legacy `memory.md` quando `MEMORY.md` è assente),
      `memory/YYYY-MM-DD.md`, facoltativamente `HEARTBEAT.md`.
    - **Directory di stato (`~/.openclaw`)**: configurazione, stato di canali/provider, profili di autenticazione, sessioni, log
      e Skills condivise (`~/.openclaw/skills`).

    Il workspace predefinito è `~/.openclaw/workspace`, configurabile tramite:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Se il bot "dimentica" dopo un riavvio, conferma che il Gateway stia usando lo stesso
    workspace a ogni avvio (e ricorda: la modalità remota usa il workspace dell'**host gateway**,
    non quello del tuo laptop locale).

    Suggerimento: se vuoi un comportamento o una preferenza durevoli, chiedi al bot di **scriverli in
    AGENTS.md o MEMORY.md** invece di fare affidamento sulla cronologia della chat.

    Vedi [Workspace dell'agente](/it/concepts/agent-workspace) e [Memory](/it/concepts/memory).

  </Accordion>

  <Accordion title="Strategia di backup consigliata">
    Metti il tuo **workspace dell'agente** in un repository git **privato** e fai il backup in un luogo
    privato (per esempio GitHub privato). In questo modo acquisisci memoria + file AGENTS/SOUL/USER
    e puoi ripristinare in seguito la "mente" dell'assistente.

    **Non** fare commit di nulla sotto `~/.openclaw` (credenziali, sessioni, token o payload di secret crittografati).
    Se hai bisogno di un ripristino completo, esegui separatamente il backup sia del workspace sia della directory di stato
    (vedi la domanda sulla migrazione sopra).

    Documentazione: [Workspace dell'agente](/it/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Come disinstallo completamente OpenClaw?">
    Vedi la guida dedicata: [Disinstallazione](/it/install/uninstall).
  </Accordion>

  <Accordion title="Gli agenti possono lavorare fuori dal workspace?">
    Sì. Il workspace è la **cwd predefinita** e l'ancora della memoria, non una sandbox rigida.
    I percorsi relativi vengono risolti nel workspace, ma i percorsi assoluti possono accedere ad altre
    posizioni dell'host a meno che il sandboxing non sia abilitato. Se hai bisogno di isolamento, usa
    [`agents.defaults.sandbox`](/it/gateway/sandboxing) o impostazioni sandbox per agente. Se vuoi che un
    repository sia la directory di lavoro predefinita, punta il `workspace` di
    quell'agente alla root del repository. Il repository OpenClaw è solo codice sorgente; tieni il
    workspace separato a meno che tu non voglia deliberatamente che l'agente lavori al suo interno.

    Esempio (repository come cwd predefinita):

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Modalità remota: dov'è l'archivio delle sessioni?">
    Lo stato delle sessioni è posseduto dall'**host gateway**. Se sei in modalità remota, l'archivio delle sessioni che ti interessa è sulla macchina remota, non sul tuo laptop locale. Vedi [Gestione delle sessioni](/it/concepts/session).
  </Accordion>
</AccordionGroup>

## Fondamenti di configurazione

<AccordionGroup>
  <Accordion title="Che formato ha la configurazione? Dov'è?">
    OpenClaw legge una configurazione facoltativa **JSON5** da `$OPENCLAW_CONFIG_PATH` (predefinito: `~/.openclaw/openclaw.json`):

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    Se il file manca, usa valori predefiniti ragionevolmente sicuri (incluso un workspace predefinito di `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title='Ho impostato gateway.bind: "lan" (o "tailnet") e ora non ascolta nulla / la UI dice unauthorized'>
    I bind non-loopback **richiedono un percorso di autenticazione gateway valido**. In pratica significa:

    - autenticazione con secret condiviso: token o password
    - `gateway.auth.mode: "trusted-proxy"` dietro un reverse proxy non-loopback con riconoscimento dell'identità configurato correttamente

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    Note:

    - `gateway.remote.token` / `.password` **non** abilitano da soli l'autenticazione del gateway locale.
    - I percorsi di chiamata locale possono usare `gateway.remote.*` come fallback solo quando `gateway.auth.*` non è impostato.
    - Per l'autenticazione con password, imposta invece `gateway.auth.mode: "password"` più `gateway.auth.password` (oppure `OPENCLAW_GATEWAY_PASSWORD`).
    - Se `gateway.auth.token` / `gateway.auth.password` è configurato esplicitamente tramite SecretRef e non risolto, la risoluzione fallisce in modalità closed (nessun fallback remoto che mascheri il problema).
    - Le configurazioni Control UI con secret condiviso si autenticano tramite `connect.params.auth.token` o `connect.params.auth.password` (memorizzati nelle impostazioni app/UI). Le modalità basate sull'identità come Tailscale Serve o `trusted-proxy` usano invece header di richiesta. Evita di inserire secret condivisi negli URL.
    - Con `gateway.auth.mode: "trusted-proxy"`, anche i reverse proxy loopback sullo stesso host **non** soddisfano l'autenticazione trusted-proxy. Il trusted proxy deve essere una sorgente non-loopback configurata.

  </Accordion>

  <Accordion title="Perché ora ho bisogno di un token anche su localhost?">
    OpenClaw impone per impostazione predefinita l'autenticazione del gateway, incluso loopback. Nel normale percorso predefinito questo significa autenticazione via token: se non è configurato alcun percorso di autenticazione esplicito, l'avvio del gateway risolve la modalità token e ne genera automaticamente uno, salvandolo in `gateway.auth.token`, quindi **i client WS locali devono autenticarsi**. Questo impedisce ad altri processi locali di chiamare il Gateway.

    Se preferisci un percorso di autenticazione diverso, puoi scegliere esplicitamente la modalità password (oppure, per reverse proxy non-loopback con riconoscimento dell'identità, `trusted-proxy`). Se **vuoi davvero** un loopback aperto, imposta esplicitamente `gateway.auth.mode: "none"` nella tua configurazione. Doctor può generare un token in qualsiasi momento: `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="Devo riavviare dopo aver cambiato la configurazione?">
    Il Gateway osserva la configurazione e supporta il hot-reload:

    - `gateway.reload.mode: "hybrid"` (predefinito): applica a caldo le modifiche sicure, riavvia per quelle critiche
    - Sono supportati anche `hot`, `restart`, `off`

  </Accordion>

  <Accordion title="Come disabilito i tagline divertenti della CLI?">
    Imposta `cli.banner.taglineMode` nella configurazione:

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`: nasconde il testo della tagline ma mantiene la riga del titolo/banner con la versione.
    - `default`: usa sempre `All your chats, one OpenClaw.`.
    - `random`: tagline divertenti/stagionali a rotazione (comportamento predefinito).
    - Se non vuoi alcun banner, imposta la variabile env `OPENCLAW_HIDE_BANNER=1`.

  </Accordion>

  <Accordion title="Come abilito la ricerca web (e web fetch)?">
    `web_fetch` funziona senza chiave API. `web_search` dipende dal provider
    selezionato:

    - I provider basati su API come Brave, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Perplexity e Tavily richiedono la normale configurazione della chiave API.
    - Ollama Web Search non richiede chiavi, ma usa l'host Ollama configurato e richiede `ollama signin`.
    - DuckDuckGo non richiede chiavi, ma è un'integrazione non ufficiale basata su HTML.
    - SearXNG non richiede chiavi/può essere self-hosted; configura `SEARXNG_BASE_URL` oppure `plugins.entries.searxng.config.webSearch.baseUrl`.

    **Consigliato:** esegui `openclaw configure --section web` e scegli un provider.
    Alternative tramite variabili ambiente:

    - Brave: `BRAVE_API_KEY`
    - Exa: `EXA_API_KEY`
    - Firecrawl: `FIRECRAWL_API_KEY`
    - Gemini: `GEMINI_API_KEY`
    - Grok: `XAI_API_KEY`
    - Kimi: `KIMI_API_KEY` oppure `MOONSHOT_API_KEY`
    - MiniMax Search: `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` oppure `MINIMAX_API_KEY`
    - Perplexity: `PERPLEXITY_API_KEY` oppure `OPENROUTER_API_KEY`
    - SearXNG: `SEARXNG_BASE_URL`
    - Tavily: `TAVILY_API_KEY`

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
        },
        tools: {
          web: {
            search: {
              enabled: true,
              provider: "brave",
              maxResults: 5,
            },
            fetch: {
              enabled: true,
              provider: "firecrawl", // facoltativo; ometti per rilevamento automatico
            },
          },
        },
    }
    ```

    La configurazione web-search specifica del provider ora vive in `plugins.entries.<plugin>.config.webSearch.*`.
    I percorsi provider legacy `tools.web.search.*` vengono ancora caricati temporaneamente per compatibilità, ma non dovrebbero essere usati nelle nuove configurazioni.
    La configurazione di fallback web-fetch Firecrawl vive in `plugins.entries.firecrawl.config.webFetch.*`.

    Note:

    - Se usi allowlist, aggiungi `web_search`/`web_fetch`/`x_search` oppure `group:web`.
    - `web_fetch` è abilitato per impostazione predefinita (a meno che non venga disabilitato esplicitamente).
    - Se `tools.web.fetch.provider` viene omesso, OpenClaw rileva automaticamente il primo provider di fallback fetch pronto tra le credenziali disponibili. Oggi il provider incluso è Firecrawl.
    - I daemon leggono le variabili env da `~/.openclaw/.env` (oppure dall'ambiente del servizio).

    Documentazione: [Strumenti web](/it/tools/web).

  </Accordion>

  <Accordion title="config.apply ha cancellato la mia configurazione. Come recupero e come lo evito?">
    `config.apply` sostituisce l'**intera configurazione**. Se invii un oggetto parziale, tutto il
    resto viene rimosso.

    Recupero:

    - Ripristina da un backup (git o una copia di `~/.openclaw/openclaw.json`).
    - Se non hai un backup, riesegui `openclaw doctor` e riconfigura canali/modelli.
    - Se è stato inaspettato, segnala un bug e includi l'ultima configurazione nota o eventuali backup.
    - Un agente di coding locale può spesso ricostruire una configurazione funzionante da log o cronologia.

    Per evitarlo:

    - Usa `openclaw config set` per piccole modifiche.
    - Usa `openclaw configure` per modifiche interattive.
    - Usa prima `config.schema.lookup` quando non sei sicuro del percorso esatto o della forma di un campo; restituisce un nodo schema superficiale più riepiloghi immediati dei figli per approfondire.
    - Usa `config.patch` per modifiche RPC parziali; conserva `config.apply` solo per la sostituzione completa della configurazione.
    - Se stai usando lo strumento `gateway` riservato al proprietario da un'esecuzione agente, continuerà comunque a rifiutare scritture su `tools.exec.ask` / `tools.exec.security` (incluse le alias legacy `tools.bash.*` che si normalizzano agli stessi percorsi exec protetti).

    Documentazione: [Config](/cli/config), [Configure](/cli/configure), [Doctor](/it/gateway/doctor).

  </Accordion>

  <Accordion title="Come eseguo un Gateway centrale con worker specializzati su più dispositivi?">
    Il pattern più comune è **un Gateway** (es. Raspberry Pi) più **nodi** e **agenti**:

    - **Gateway (centrale):** possiede canali (Signal/WhatsApp), instradamento e sessioni.
    - **Nodi (dispositivi):** Mac/iOS/Android si connettono come periferiche ed espongono strumenti locali (`system.run`, `canvas`, `camera`).
    - **Agenti (worker):** cervelli/workspace separati per ruoli speciali (es. "Hetzner ops", "Dati personali").
    - **Sub-agent:** avviano lavoro in background da un agente principale quando vuoi parallelismo.
    - **TUI:** si connette al Gateway e cambia agenti/sessioni.

    Documentazione: [Nodes](/it/nodes), [Accesso remoto](/it/gateway/remote), [Instradamento Multi-Agent](/it/concepts/multi-agent), [Sub-agent](/it/tools/subagents), [TUI](/web/tui).

  </Accordion>

  <Accordion title="Il browser di OpenClaw può girare headless?">
    Sì. È un'opzione di configurazione:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    Il valore predefinito è `false` (headful). La modalità headless ha più probabilità di attivare controlli anti-bot su alcuni siti. Vedi [Browser](/it/tools/browser).

    La modalità headless usa **lo stesso motore Chromium** e funziona per la maggior parte delle automazioni (moduli, clic, scraping, login). Le principali differenze:

    - Nessuna finestra browser visibile (usa screenshot se ti servono immagini).
    - Alcuni siti sono più severi riguardo all'automazione in modalità headless (CAPTCHA, anti-bot).
      Per esempio, X/Twitter spesso blocca le sessioni headless.

  </Accordion>

  <Accordion title="Come uso Brave per il controllo del browser?">
    Imposta `browser.executablePath` sul binario Brave (o di qualsiasi browser basato su Chromium) e riavvia il Gateway.
    Vedi gli esempi completi di configurazione in [Browser](/it/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Gateway remoti e nodi

<AccordionGroup>
  <Accordion title="Come si propagano i comandi tra Telegram, il gateway e i nodi?">
    I messaggi Telegram vengono gestiti dal **gateway**. Il gateway esegue l'agente e
    solo allora chiama i nodi tramite il **Gateway WebSocket** quando serve uno strumento del nodo:

    Telegram → Gateway → Agent → `node.*` → Node → Gateway → Telegram

    I nodi non vedono il traffico provider in ingresso; ricevono solo chiamate RPC del nodo.

  </Accordion>

  <Accordion title="Come può il mio agente accedere al mio computer se il Gateway è ospitato in remoto?">
    Risposta breve: **associa il tuo computer come nodo**. Il Gateway gira altrove, ma può
    chiamare gli strumenti `node.*` (schermo, fotocamera, sistema) sulla tua macchina locale tramite il Gateway WebSocket.

    Configurazione tipica:

    1. Esegui il Gateway sull'host sempre attivo (VPS/home server).
    2. Metti l'host Gateway e il tuo computer sulla stessa tailnet.
    3. Assicurati che il WS del Gateway sia raggiungibile (bind tailnet o tunnel SSH).
    4. Apri localmente l'app macOS e collegati in modalità **Remote over SSH** (o tailnet diretto)
       così può registrarsi come nodo.
    5. Approva il nodo sul Gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Non serve alcun bridge TCP separato; i nodi si connettono tramite il Gateway WebSocket.

    Promemoria di sicurezza: associare un nodo macOS consente `system.run` su quella macchina. Associa
    solo dispositivi di cui ti fidi e consulta [Security](/it/gateway/security).

    Documentazione: [Nodes](/it/nodes), [Protocollo Gateway](/it/gateway/protocol), [macOS remote mode](/it/platforms/mac/remote), [Security](/it/gateway/security).

  </Accordion>

  <Accordion title="Tailscale è connesso ma non ricevo risposte. E adesso?">
    Controlla le basi:

    - Il Gateway è in esecuzione: `openclaw gateway status`
    - Stato del Gateway: `openclaw status`
    - Stato dei canali: `openclaw channels status`

    Poi verifica autenticazione e instradamento:

    - Se usi Tailscale Serve, assicurati che `gateway.auth.allowTailscale` sia impostato correttamente.
    - Se ti connetti tramite tunnel SSH, conferma che il tunnel locale sia attivo e punti alla porta giusta.
    - Conferma che le allowlist (DM o gruppo) includano il tuo account.

    Documentazione: [Tailscale](/it/gateway/tailscale), [Accesso remoto](/it/gateway/remote), [Canali](/it/channels).

  </Accordion>

  <Accordion title="Due istanze OpenClaw possono parlare tra loro (locale + VPS)?">
    Sì. Non esiste un bridge "bot-to-bot" integrato, ma puoi collegarli in alcuni
    modi affidabili:

    **Più semplice:** usa un normale canale chat a cui entrambi i bot possono accedere (Telegram/Slack/WhatsApp).
    Fai in modo che Bot A invii un messaggio a Bot B, poi lascia che Bot B risponda normalmente.

    **Bridge CLI (generico):** esegui uno script che chiama l'altro Gateway con
    `openclaw agent --message ... --deliver`, indirizzato a una chat in cui l'altro bot
    ascolta. Se un bot è su un VPS remoto, punta la tua CLI a quel Gateway remoto
    tramite SSH/Tailscale (vedi [Accesso remoto](/it/gateway/remote)).

    Pattern di esempio (eseguito da una macchina che può raggiungere il Gateway di destinazione):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Suggerimento: aggiungi un guardrail per evitare che i due bot finiscano in un loop infinito (solo mention,
    allowlist di canale o una regola "non rispondere ai messaggi dei bot").

    Documentazione: [Accesso remoto](/it/gateway/remote), [CLI Agent](/cli/agent), [Agent send](/it/tools/agent-send).

  </Accordion>

  <Accordion title="Ho bisogno di VPS separati per più agenti?">
    No. Un Gateway può ospitare più agenti, ciascuno con il proprio workspace, modelli predefiniti
    e instradamento. Questa è la configurazione normale ed è molto più economica e semplice che eseguire
    un VPS per agente.

    Usa VPS separati solo quando hai bisogno di isolamento rigido (confini di sicurezza) o di
    configurazioni molto diverse che non vuoi condividere. Altrimenti, mantieni un solo Gateway e
    usa più agenti o sub-agent.

  </Accordion>

  <Accordion title="C'è un vantaggio nell'usare un nodo sul mio laptop personale invece di SSH da un VPS?">
    Sì - i nodi sono il modo di prima classe per raggiungere il tuo laptop da un Gateway remoto, e
    sbloccano più del semplice accesso shell. Il Gateway gira su macOS/Linux (Windows via WSL2) ed è
    leggero (va bene un piccolo VPS o una macchina di classe Raspberry Pi; 4 GB di RAM sono più che sufficienti), quindi una configurazione comune
    è un host sempre attivo più il tuo laptop come nodo.

    - **Nessun SSH in ingresso richiesto.** I nodi si connettono in uscita al Gateway WebSocket e usano il pairing dei dispositivi.
    - **Controlli di esecuzione più sicuri.** `system.run` è controllato da allowlist/approvazioni del nodo su quel laptop.
    - **Più strumenti del dispositivo.** I nodi espongono `canvas`, `camera` e `screen` oltre a `system.run`.
    - **Automazione browser locale.** Mantieni il Gateway su un VPS, ma esegui Chrome in locale tramite un node host sul laptop, oppure collegati a Chrome locale sull'host tramite Chrome MCP.

    SSH va bene per accesso shell occasionale, ma i nodi sono più semplici per workflow continuativi degli agenti e
    l'automazione del dispositivo.

    Documentazione: [Nodes](/it/nodes), [CLI Nodes](/cli/nodes), [Browser](/it/tools/browser).

  </Accordion>

  <Accordion title="I nodi eseguono un servizio gateway?">
    No. Dovrebbe essere eseguito un solo **gateway** per host, a meno che tu non stia deliberatamente eseguendo profili isolati (vedi [Gateway multipli](/it/gateway/multiple-gateways)). I nodi sono periferiche che si collegano
    al gateway (nodi iOS/Android o "node mode" macOS nell'app menubar). Per node
    host headless e controllo CLI, vedi [CLI Node host](/cli/node).

    È richiesto un riavvio completo per modifiche a `gateway`, `discovery` e `canvasHost`.

  </Accordion>

  <Accordion title="Esiste un'API / un modo RPC per applicare la configurazione?">
    Sì.

    - `config.schema.lookup`: ispeziona un sottoalbero di configurazione con il relativo nodo schema superficiale, il suggerimento UI corrispondente e i riepiloghi immediati dei figli prima di scrivere
    - `config.get`: recupera lo snapshot corrente + hash
    - `config.patch`: aggiornamento parziale sicuro (preferito per la maggior parte delle modifiche RPC)
    - `config.apply`: convalida + sostituisce l'intera configurazione, poi riavvia
    - Lo strumento runtime `gateway` riservato al proprietario continua a rifiutare la riscrittura di `tools.exec.ask` / `tools.exec.security`; gli alias legacy `tools.bash.*` si normalizzano agli stessi percorsi exec protetti

  </Accordion>

  <Accordion title="Configurazione minima ragionevole per una prima installazione">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Questo imposta il tuo workspace e limita chi può attivare il bot.

  </Accordion>

  <Accordion title="Come configuro Tailscale su un VPS e mi collego dal mio Mac?">
    Passaggi minimi:

    1. **Installa + accedi sul VPS**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **Installa + accedi sul tuo Mac**
       - Usa l'app Tailscale e accedi alla stessa tailnet.
    3. **Abilita MagicDNS (consigliato)**
       - Nella console di amministrazione Tailscale, abilita MagicDNS così il VPS avrà un nome stabile.
    4. **Usa il nome host della tailnet**
       - SSH: `ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS: `ws://your-vps.tailnet-xxxx.ts.net:18789`

    Se vuoi la Control UI senza SSH, usa Tailscale Serve sul VPS:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Questo mantiene il gateway associato a loopback ed espone HTTPS tramite Tailscale. Vedi [Tailscale](/it/gateway/tailscale).

  </Accordion>

  <Accordion title="Come collego un nodo Mac a un Gateway remoto (Tailscale Serve)?">
    Serve espone la **Control UI + WS del Gateway**. I nodi si connettono tramite lo stesso endpoint WS del Gateway.

    Configurazione consigliata:

    1. **Assicurati che VPS + Mac siano sulla stessa tailnet**.
    2. **Usa l'app macOS in modalità Remote** (il target SSH può essere il nome host tailnet).
       L'app creerà un tunnel sulla porta del Gateway e si connetterà come nodo.
    3. **Approva il nodo** sul gateway:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Documentazione: [Protocollo Gateway](/it/gateway/protocol), [Discovery](/it/gateway/discovery), [macOS remote mode](/it/platforms/mac/remote).

  </Accordion>

  <Accordion title="Dovrei installare su un secondo laptop o aggiungere semplicemente un nodo?">
    Se hai bisogno solo di **strumenti locali** (schermo/fotocamera/exec) sul secondo laptop, aggiungilo come
    **nodo**. In questo modo mantieni un solo Gateway ed eviti configurazioni duplicate. Gli strumenti dei nodi locali
    sono attualmente solo per macOS, ma prevediamo di estenderli ad altri sistemi operativi.

    Installa un secondo Gateway solo quando hai bisogno di **isolamento rigido** o di due bot completamente separati.

    Documentazione: [Nodes](/it/nodes), [CLI Nodes](/cli/nodes), [Gateway multipli](/it/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Variabili env e caricamento .env

<AccordionGroup>
  <Accordion title="Come carica le variabili ambiente OpenClaw?">
    OpenClaw legge le variabili env dal processo genitore (shell, launchd/systemd, CI, ecc.) e in più carica:

    - `.env` dalla directory di lavoro corrente
    - un `.env` globale di fallback da `~/.openclaw/.env` (ovvero `$OPENCLAW_STATE_DIR/.env`)

    Nessuno dei due file `.env` sovrascrive variabili env già esistenti.

    Puoi anche definire variabili env inline nella configurazione (applicate solo se mancanti nel process env):

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Vedi [/environment](/it/help/environment) per precedenza e sorgenti complete.

  </Accordion>

  <Accordion title="Ho avviato il Gateway tramite il servizio e le mie variabili env sono sparite. E adesso?">
    Due correzioni comuni:

    1. Inserisci le chiavi mancanti in `~/.openclaw/.env` così verranno rilevate anche quando il servizio non eredita l'ambiente della shell.
    2. Abilita l'importazione della shell (comodità opt-in):

    ```json5
    {
      env: {
        shellEnv: {
          enabled: true,
          timeoutMs: 15000,
        },
      },
    }
    ```

    Questo esegue la tua login shell e importa solo le chiavi attese mancanti (senza mai sovrascrivere). Equivalenti tramite variabili env:
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='Ho impostato COPILOT_GITHUB_TOKEN, ma models status mostra "Shell env: off." Perché?'>
    `openclaw models status` riporta se l'**importazione dell'ambiente shell** è abilitata. "Shell env: off"
    **non** significa che manchino le tue variabili env - significa solo che OpenClaw non caricherà
    automaticamente la tua login shell.

    Se il Gateway gira come servizio (launchd/systemd), non erediterà l'ambiente della shell.
    Risolvi in uno di questi modi:

    1. Inserisci il token in `~/.openclaw/.env`:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. Oppure abilita l'importazione della shell (`env.shellEnv.enabled: true`).
    3. Oppure aggiungilo al blocco `env` della configurazione (si applica solo se mancante).

    Poi riavvia il gateway e ricontrolla:

    ```bash
    openclaw models status
    ```

    I token Copilot vengono letti da `COPILOT_GITHUB_TOKEN` (anche `GH_TOKEN` / `GITHUB_TOKEN`).
    Vedi [/concepts/model-providers](/it/concepts/model-providers) e [/environment](/it/help/environment).

  </Accordion>
</AccordionGroup>

## Sessioni e più chat

<AccordionGroup>
  <Accordion title="Come avvio una conversazione nuova?">
    Invia `/new` o `/reset` come messaggio autonomo. Vedi [Gestione delle sessioni](/it/concepts/session).
  </Accordion>

  <Accordion title="Le sessioni si azzerano automaticamente se non invio mai /new?">
    Le sessioni possono scadere dopo `session.idleMinutes`, ma questa opzione è **disabilitata per impostazione predefinita** (valore predefinito **0**).
    Impostala su un valore positivo per abilitare la scadenza per inattività. Quando abilitata, il messaggio **successivo**
    dopo il periodo di inattività avvia un nuovo ID sessione per quella chiave di chat.
    Questo non elimina le trascrizioni - avvia solo una nuova sessione.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="Esiste un modo per creare un team di istanze OpenClaw (un CEO e molti agenti)?">
    Sì, tramite **instradamento multi-agent** e **sub-agent**. Puoi creare un agente
    coordinatore e diversi agenti worker con propri workspace e modelli.

    Detto questo, è meglio considerarlo un **esperimento divertente**. Consuma molti token ed è spesso
    meno efficiente che usare un solo bot con sessioni separate. Il modello tipico che
    immaginiamo è un solo bot con cui parli, con sessioni diverse per lavoro parallelo. Quel
    bot può anche avviare sub-agent quando necessario.

    Documentazione: [Instradamento multi-agent](/it/concepts/multi-agent), [Sub-agent](/it/tools/subagents), [CLI Agents](/cli/agents).

  </Accordion>

  <Accordion title="Perché il contesto è stato troncato a metà attività? Come posso evitarlo?">
    Il contesto di sessione è limitato dalla finestra del modello. Chat lunghe, grandi output degli strumenti o molti
    file possono attivare compattazione o troncamento.

    Cosa aiuta:

    - Chiedi al bot di riepilogare lo stato corrente e scriverlo in un file.
    - Usa `/compact` prima di attività lunghe e `/new` quando cambi argomento.
    - Mantieni il contesto importante nel workspace e chiedi al bot di rileggerlo.
    - Usa sub-agent per lavori lunghi o paralleli così la chat principale resta più piccola.
    - Scegli un modello con una finestra di contesto più ampia se succede spesso.

  </Accordion>

  <Accordion title="Come azzero completamente OpenClaw ma mantenendolo installato?">
    Usa il comando reset:

    ```bash
    openclaw reset
    ```

    Reset completo non interattivo:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Poi riesegui la configurazione:

    ```bash
    openclaw onboard --install-daemon
    ```

    Note:

    - Onboarding offre anche **Reset** se rileva una configurazione esistente. Vedi [Onboarding (CLI)](/it/start/wizard).
    - Se hai usato profili (`--profile` / `OPENCLAW_PROFILE`), azzera ogni directory di stato (i valori predefiniti sono `~/.openclaw-<profile>`).
    - Reset dev: `openclaw gateway --dev --reset` (solo dev; cancella configurazione dev + credenziali + sessioni + workspace).

  </Accordion>

  <Accordion title='Ricevo errori "context too large" - come faccio a resettare o compattare?'>
    Usa una di queste opzioni:

    - **Compatta** (mantiene la conversazione ma riassume i turni più vecchi):

      ```
      /compact
      ```

      oppure `/compact <instructions>` per guidare il riepilogo.

    - **Reset** (nuovo ID sessione per la stessa chiave di chat):

      ```
      /new
      /reset
      ```

    Se continua a succedere:

    - Abilita o regola la **session pruning** (`agents.defaults.contextPruning`) per ridurre il vecchio output degli strumenti.
    - Usa un modello con una finestra di contesto più grande.

    Documentazione: [Compattazione](/it/concepts/compaction), [Session pruning](/it/concepts/session-pruning), [Gestione delle sessioni](/it/concepts/session).

  </Accordion>

  <Accordion title='Perché vedo "LLM request rejected: messages.content.tool_use.input field required"?'>
    Questo è un errore di validazione del provider: il modello ha emesso un blocco `tool_use` senza il campo
    `input` richiesto. Di solito significa che la cronologia della sessione è obsoleta o corrotta (spesso dopo thread lunghi
    o una modifica a strumento/schema).

    Correzione: avvia una nuova sessione con `/new` (messaggio autonomo).

  </Accordion>

  <Accordion title="Perché ricevo messaggi heartbeat ogni 30 minuti?">
    Gli heartbeat vengono eseguiti ogni **30m** per impostazione predefinita (**1h** quando si usa autenticazione OAuth). Regolali o disabilitali:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // oppure "0m" per disabilitare
          },
        },
      },
    }
    ```

    Se `HEARTBEAT.md` esiste ma è di fatto vuoto (solo righe vuote e intestazioni markdown
    come `# Heading`), OpenClaw salta l'esecuzione di heartbeat per risparmiare chiamate API.
    Se il file manca, l'heartbeat continua a essere eseguito e il modello decide cosa fare.

    Gli override per agente usano `agents.list[].heartbeat`. Documentazione: [Heartbeat](/it/gateway/heartbeat).

  </Accordion>

  <Accordion title='Devo aggiungere un "bot account" a un gruppo WhatsApp?'>
    No. OpenClaw gira sul **tuo account personale**, quindi se sei nel gruppo, OpenClaw può vederlo.
    Per impostazione predefinita, le risposte nei gruppi sono bloccate finché non consenti i mittenti (`groupPolicy: "allowlist"`).

    Se vuoi che solo **tu** possa attivare risposte nel gruppo:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Come ottengo il JID di un gruppo WhatsApp?">
    Opzione 1 (più veloce): segui i log e invia un messaggio di test nel gruppo:

    ```bash
    openclaw logs --follow --json
    ```

    Cerca `chatId` (oppure `from`) che termina con `@g.us`, come:
    `1234567890-1234567890@g.us`.

    Opzione 2 (se già configurato/in allowlist): elenca i gruppi dalla configurazione:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Documentazione: [WhatsApp](/it/channels/whatsapp), [Directory](/cli/directory), [Logs](/cli/logs).

  </Accordion>

  <Accordion title="Perché OpenClaw non risponde in un gruppo?">
    Due cause comuni:

    - Il gating sulle mention è attivo (predefinito). Devi fare @mention al bot (o corrispondere a `mentionPatterns`).
    - Hai configurato `channels.whatsapp.groups` senza `"*"` e il gruppo non è nella allowlist.

    Vedi [Gruppi](/it/channels/groups) e [Messaggi di gruppo](/it/channels/group-messages).

  </Accordion>

  <Accordion title="Gruppi/thread condividono il contesto con i DM?">
    Le chat dirette collassano per impostazione predefinita nella sessione principale. Gruppi/canali hanno le proprie chiavi di sessione e i topic Telegram / thread Discord sono sessioni separate. Vedi [Gruppi](/it/channels/groups) e [Messaggi di gruppo](/it/channels/group-messages).
  </Accordion>

  <Accordion title="Quanti workspace e agenti posso creare?">
    Nessun limite rigido. Decine (anche centinaia) vanno bene, ma fai attenzione a:

    - **Crescita del disco:** sessioni + trascrizioni vivono sotto `~/.openclaw/agents/<agentId>/sessions/`.
    - **Costo dei token:** più agenti significa maggiore uso concorrente dei modelli.
    - **Sovraccarico operativo:** profili di autenticazione, workspace e instradamento dei canali per agente.

    Suggerimenti:

    - Mantieni un workspace **attivo** per agente (`agents.defaults.workspace`).
    - Pota le vecchie sessioni (elimina JSONL o voci dello store) se il disco cresce.
    - Usa `openclaw doctor` per individuare workspace orfani e mancate corrispondenze dei profili.

  </Accordion>

  <Accordion title="Posso eseguire più bot o chat contemporaneamente (Slack) e come dovrei configurarlo?">
    Sì. Usa **Instradamento Multi-Agent** per eseguire più agenti isolati e instradare i messaggi in ingresso per
    canale/account/peer. Slack è supportato come canale e può essere associato ad agenti specifici.

    L'accesso browser è potente ma non equivale a "fare tutto ciò che può fare un essere umano" - anti-bot, CAPTCHA e MFA possono
    comunque bloccare l'automazione. Per il controllo browser più affidabile, usa Chrome MCP locale sull'host,
    oppure usa CDP sulla macchina che esegue effettivamente il browser.

    Configurazione consigliata:

    - Host Gateway sempre attivo (VPS/Mac mini).
    - Un agente per ruolo (binding).
    - Canale/i Slack associati a quegli agenti.
    - Browser locale via Chrome MCP o un nodo quando necessario.

    Documentazione: [Instradamento Multi-Agent](/it/concepts/multi-agent), [Slack](/it/channels/slack),
    [Browser](/it/tools/browser), [Nodes](/it/nodes).

  </Accordion>
</AccordionGroup>

## Modelli: valori predefiniti, selezione, alias, cambio

<AccordionGroup>
  <Accordion title='Che cos'è il "modello predefinito"?'>
    Il modello predefinito di OpenClaw è qualunque modello imposti come:

    ```
    agents.defaults.model.primary
    ```

    I modelli sono referenziati come `provider/model` (esempio: `openai/gpt-5.4`). Se ometti il provider, OpenClaw prova prima un alias, poi una corrispondenza univoca del provider configurato per quell'esatto ID modello, e solo dopo ricade sul provider predefinito configurato come percorso di compatibilità deprecato. Se quel provider non espone più il modello predefinito configurato, OpenClaw usa come fallback il primo provider/modello configurato invece di mostrare un valore predefinito obsoleto di un provider rimosso. Dovresti comunque impostare **esplicitamente** `provider/model`.

  </Accordion>

  <Accordion title="Quale modello consigliate?">
    **Valore predefinito consigliato:** usa il modello di ultima generazione più potente disponibile nel tuo stack provider.
    **Per agenti con strumenti abilitati o input non attendibili:** dai priorità alla potenza del modello rispetto al costo.
    **Per chat di routine/a bassa criticità:** usa modelli fallback più economici e instradali per ruolo dell'agente.

    MiniMax ha una propria documentazione: [MiniMax](/it/providers/minimax) e
    [Modelli locali](/it/gateway/local-models).

    Regola pratica: usa il **miglior modello che puoi permetterti** per lavori ad alta criticità e un modello più economico
    per chat di routine o riepiloghi. Puoi instradare i modelli per agente e usare sub-agent per
    parallelizzare attività lunghe (ogni sub-agent consuma token). Vedi [Modelli](/it/concepts/models) e
    [Sub-agent](/it/tools/subagents).

    Forte avvertimento: i modelli più deboli o eccessivamente quantizzati sono più vulnerabili a prompt
    injection e comportamenti non sicuri. Vedi [Security](/it/gateway/security).

    Più contesto: [Modelli](/it/concepts/models).

  </Accordion>

  <Accordion title="Come cambio modello senza cancellare la mia configurazione?">
    Usa **comandi del modello** oppure modifica solo i campi del **modello**. Evita sostituzioni complete della configurazione.

    Opzioni sicure:

    - `/model` in chat (rapido, per sessione)
    - `openclaw models set ...` (aggiorna solo la configurazione del modello)
    - `openclaw configure --section model` (interattivo)
    - modifica `agents.defaults.model` in `~/.openclaw/openclaw.json`

    Evita `config.apply` con un oggetto parziale a meno che tu non voglia sostituire l'intera configurazione.
    Per modifiche RPC, ispeziona prima con `config.schema.lookup` e preferisci `config.patch`. Il payload di lookup ti fornisce il percorso normalizzato, documentazione/vincoli dello schema superficiale e riepiloghi immediati dei figli
    per aggiornamenti parziali.
    Se hai sovrascritto la configurazione, ripristina da backup o riesegui `openclaw doctor` per ripararla.

    Documentazione: [Models](/it/concepts/models), [Configure](/cli/configure), [Config](/cli/config), [Doctor](/it/gateway/doctor).

  </Accordion>

  <Accordion title="Posso usare modelli self-hosted (llama.cpp, vLLM, Ollama)?">
    Sì. Ollama è il percorso più semplice per i modelli locali.

    Configurazione più rapida:

    1. Installa Ollama da `https://ollama.com/download`
    2. Scarica un modello locale come `ollama pull glm-4.7-flash`
    3. Se vuoi anche modelli cloud, esegui `ollama signin`
    4. Esegui `openclaw onboard` e scegli `Ollama`
    5. Scegli `Local` oppure `Cloud + Local`

    Note:

    - `Cloud + Local` ti offre modelli cloud più i tuoi modelli Ollama locali
    - i modelli cloud come `kimi-k2.5:cloud` non richiedono un pull locale
    - per il cambio manuale, usa `openclaw models list` e `openclaw models set ollama/<model>`

    Nota di sicurezza: i modelli più piccoli o fortemente quantizzati sono più vulnerabili a prompt
    injection. Consigliamo fortemente **modelli grandi** per qualsiasi bot che possa usare strumenti.
    Se vuoi comunque modelli piccoli, abilita il sandboxing e allowlist rigide degli strumenti.

    Documentazione: [Ollama](/it/providers/ollama), [Modelli locali](/it/gateway/local-models),
    [Provider di modelli](/it/concepts/model-providers), [Security](/it/gateway/security),
    [Sandboxing](/it/gateway/sandboxing).

  </Accordion>

  <Accordion title="Quali modelli usano OpenClaw, Flawd e Krill?">
    - Queste distribuzioni possono differire e cambiare nel tempo; non esiste una raccomandazione fissa sul provider.
    - Controlla l'impostazione runtime corrente su ogni gateway con `openclaw models status`.
    - Per agenti sensibili alla sicurezza/con strumenti abilitati, usa il modello di ultima generazione più potente disponibile.
  </Accordion>

  <Accordion title="Come cambio modello al volo (senza riavviare)?">
    Usa il comando `/model` come messaggio autonomo:

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    Questi sono gli alias integrati. Gli alias personalizzati possono essere aggiunti tramite `agents.defaults.models`.

    Puoi elencare i modelli disponibili con `/model`, `/model list` oppure `/model status`.

    `/model` (e `/model list`) mostra un selettore compatto e numerato. Seleziona per numero:

    ```
    /model 3
    ```

    Puoi anche forzare un profilo di autenticazione specifico per il provider (per sessione):

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Suggerimento: `/model status` mostra quale agente è attivo, quale file `auth-profiles.json` viene usato e quale profilo di autenticazione verrà provato dopo.
    Mostra anche l'endpoint provider configurato (`baseUrl`) e la modalità API (`api`) quando disponibili.

    **Come faccio a rimuovere il pin di un profilo impostato con @profile?**

    Riesegui `/model` **senza** il suffisso `@profile`:

    ```
    /model anthropic/claude-opus-4-6
    ```

    Se vuoi tornare al valore predefinito, sceglilo da `/model` (oppure invia `/model <default provider/model>`).
    Usa `/model status` per confermare quale profilo di autenticazione è attivo.

  </Accordion>

  <Accordion title="Posso usare GPT 5.2 per le attività quotidiane e Codex 5.3 per il coding?">
    Sì. Impostane uno come predefinito e cambia quando serve:

    - **Cambio rapido (per sessione):** `/model gpt-5.4` per attività quotidiane, `/model openai-codex/gpt-5.4` per coding con Codex OAuth.
    - **Predefinito + cambio:** imposta `agents.defaults.model.primary` su `openai/gpt-5.4`, poi passa a `openai-codex/gpt-5.4` quando fai coding (oppure il contrario).
    - **Sub-agent:** instrada i compiti di coding verso sub-agent con un modello predefinito diverso.

    Vedi [Modelli](/it/concepts/models) e [Comandi slash](/it/tools/slash-commands).

  </Accordion>

  <Accordion title="Come configuro la fast mode per GPT 5.4?">
    Usa un toggle di sessione oppure un valore predefinito in configurazione:

    - **Per sessione:** invia `/fast on` mentre la sessione sta usando `openai/gpt-5.4` o `openai-codex/gpt-5.4`.
    - **Valore predefinito per modello:** imposta `agents.defaults.models["openai/gpt-5.4"].params.fastMode` su `true`.
    - **Anche Codex OAuth:** se usi anche `openai-codex/gpt-5.4`, imposta lì lo stesso flag.

    Esempio:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
            "openai-codex/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    Per OpenAI, la fast mode corrisponde a `service_tier = "priority"` nelle richieste native Responses supportate. Gli override di sessione `/fast` hanno la precedenza sui valori predefiniti di configurazione.

    Vedi [Thinking e fast mode](/it/tools/thinking) e [OpenAI fast mode](/it/providers/openai#openai-fast-mode).

  </Accordion>

  <Accordion title='Perché vedo "Model ... is not allowed" e poi nessuna risposta?'>
    Se `agents.defaults.models` è impostato, diventa la **allowlist** per `/model` e per qualsiasi
    override di sessione. Scegliere un modello che non è in quell'elenco restituisce:

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    Quell'errore viene restituito **al posto** di una risposta normale. Correzione: aggiungi il modello a
    `agents.defaults.models`, rimuovi la allowlist oppure scegli un modello da `/model list`.

  </Accordion>

  <Accordion title='Perché vedo "Unknown model: minimax/MiniMax-M2.7"?'>
    Questo significa che il **provider non è configurato** (non è stata trovata alcuna configurazione provider MiniMax o alcun
    profilo di autenticazione), quindi il modello non può essere risolto.

    Checklist di correzione:

    1. Aggiorna a una release OpenClaw corrente (oppure esegui dal sorgente `main`), poi riavvia il gateway.
    2. Assicurati che MiniMax sia configurato (wizard o JSON), oppure che esista autenticazione MiniMax
       in env/profili di autenticazione in modo che il provider corrispondente possa essere inserito
       (`MINIMAX_API_KEY` per `minimax`, `MINIMAX_OAUTH_TOKEN` o OAuth MiniMax
       memorizzato per `minimax-portal`).
    3. Usa l'ID modello esatto (case-sensitive) per il tuo percorso di autenticazione:
       `minimax/MiniMax-M2.7` oppure `minimax/MiniMax-M2.7-highspeed` per configurazione
       con chiave API, oppure `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed` per configurazione OAuth.
    4. Esegui:

       ```bash
       openclaw models list
       ```

       e scegli dall'elenco (oppure `/model list` in chat).

    Vedi [MiniMax](/it/providers/minimax) e [Modelli](/it/concepts/models).

  </Accordion>

  <Accordion title="Posso usare MiniMax come predefinito e OpenAI per attività complesse?">
    Sì. Usa **MiniMax come predefinito** e cambia modello **per sessione** quando serve.
    I fallback sono per gli **errori**, non per i "compiti difficili", quindi usa `/model` o un agente separato.

    **Opzione A: cambio per sessione**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M2.7" },
          models: {
            "minimax/MiniMax-M2.7": { alias: "minimax" },
            "openai/gpt-5.4": { alias: "gpt" },
          },
        },
      },
    }
    ```

    Poi:

    ```
    /model gpt
    ```

    **Opzione B: agenti separati**

    - Agente A predefinito: MiniMax
    - Agente B predefinito: OpenAI
    - Instrada per agente o usa `/agent` per cambiare

    Documentazione: [Modelli](/it/concepts/models), [Instradamento Multi-Agent](/it/concepts/multi-agent), [MiniMax](/it/providers/minimax), [OpenAI](/it/providers/openai).

  </Accordion>

  <Accordion title="opus / sonnet / gpt sono scorciatoie integrate?">
    Sì. OpenClaw include alcune abbreviazioni predefinite (applicate solo quando il modello esiste in `agents.defaults.models`):

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3-flash-preview`
    - `gemini-flash-lite` → `google/gemini-3.1-flash-lite-preview`

    Se imposti un tuo alias con lo stesso nome, il tuo valore ha la precedenza.

  </Accordion>

  <Accordion title="Come definisco/sovrascrivo scorciatoie del modello (alias)?">
    Gli alias provengono da `agents.defaults.models.<modelId>.alias`. Esempio:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
            "anthropic/claude-haiku-4-5": { alias: "haiku" },
          },
        },
      },
    }
    ```

    Poi `/model sonnet` (oppure `/<alias>` quando supportato) viene risolto in quell'ID modello.

  </Accordion>

  <Accordion title="Come aggiungo modelli da altri provider come OpenRouter o Z.AI?">
    OpenRouter (pay-per-token; molti modelli):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (modelli GLM):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5" },
          models: { "zai/glm-5": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    Se fai riferimento a un provider/modello ma manca la chiave richiesta del provider, otterrai un errore di autenticazione runtime (es. `No API key found for provider "zai"`).

    **No API key found for provider dopo aver aggiunto un nuovo agente**

    Di solito significa che il **nuovo agente** ha un archivio di autenticazione vuoto. L'autenticazione è per-agente e
    viene salvata in:

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Opzioni di correzione:

    - Esegui `openclaw agents add <id>` e configura l'autenticazione durante la procedura guidata.
    - Oppure copia `auth-profiles.json` da `agentDir` dell'agente principale in `agentDir` del nuovo agente.

    Non riutilizzare `agentDir` tra agenti; provoca collisioni di autenticazione/sessione.

  </Accordion>
</AccordionGroup>

## Failover dei modelli e "All models failed"

<AccordionGroup>
  <Accordion title="Come funziona il failover?">
    Il failover avviene in due fasi:

    1. **Rotazione del profilo di autenticazione** all'interno dello stesso provider.
    2. **Fallback del modello** al modello successivo in `agents.defaults.model.fallbacks`.

    I cooldown si applicano ai profili in errore (backoff esponenziale), quindi OpenClaw può continuare a rispondere anche quando un provider è soggetto a rate limit o temporaneamente in errore.

    Il bucket dei rate limit include più delle semplici risposte