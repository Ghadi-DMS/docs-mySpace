---
read_when:
    - Lavorando sulle funzionalità del canale Discord
summary: Stato del supporto del bot Discord, capacità e configurazione
title: Discord
x-i18n:
    generated_at: "2026-04-06T03:07:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 54af2176a1b4fa1681e3f07494def0c652a2730165058848000e71a59e2a9d08
    source_path: channels/discord.md
    workflow: 15
---

# Discord (Bot API)

Stato: pronto per DM e canali guild tramite il gateway ufficiale di Discord.

<CardGroup cols={3}>
  <Card title="Associazione" icon="link" href="/it/channels/pairing">
    I DM di Discord usano per impostazione predefinita la modalità di associazione.
  </Card>
  <Card title="Comandi slash" icon="terminal" href="/it/tools/slash-commands">
    Comportamento dei comandi nativo e catalogo dei comandi.
  </Card>
  <Card title="Risoluzione dei problemi del canale" icon="wrench" href="/it/channels/troubleshooting">
    Diagnostica e flusso di riparazione tra canali.
  </Card>
</CardGroup>

## Configurazione rapida

Dovrai creare una nuova applicazione con un bot, aggiungere il bot al tuo server e associarlo a OpenClaw. Ti consigliamo di aggiungere il bot al tuo server privato. Se non ne hai ancora uno, [creane prima uno](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server) (scegli **Create My Own > For me and my friends**).

<Steps>
  <Step title="Crea un'applicazione Discord e un bot">
    Vai al [Discord Developer Portal](https://discord.com/developers/applications) e fai clic su **New Application**. Assegna un nome come "OpenClaw".

    Fai clic su **Bot** nella barra laterale. Imposta **Username** con il nome che usi per il tuo agente OpenClaw.

  </Step>

  <Step title="Abilita gli intent privilegiati">
    Sempre nella pagina **Bot**, scorri fino a **Privileged Gateway Intents** e abilita:

    - **Message Content Intent** (obbligatorio)
    - **Server Members Intent** (consigliato; obbligatorio per allowlist basate sui ruoli e corrispondenza nome-ID)
    - **Presence Intent** (facoltativo; necessario solo per gli aggiornamenti di presenza)

  </Step>

  <Step title="Copia il token del tuo bot">
    Torna in alto nella pagina **Bot** e fai clic su **Reset Token**.

    <Note>
    Nonostante il nome, questo genera il tuo primo token — non viene “reimpostato” nulla.
    </Note>

    Copia il token e salvalo da qualche parte. Questo è il tuo **Bot Token** e ti servirà tra poco.

  </Step>

  <Step title="Genera un URL di invito e aggiungi il bot al tuo server">
    Fai clic su **OAuth2** nella barra laterale. Genererai un URL di invito con i permessi corretti per aggiungere il bot al tuo server.

    Scorri fino a **OAuth2 URL Generator** e abilita:

    - `bot`
    - `applications.commands`

    Sotto comparirà una sezione **Bot Permissions**. Abilita:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (facoltativo)

    Copia l'URL generato in fondo, incollalo nel browser, seleziona il tuo server e fai clic su **Continue** per connetterti. Ora dovresti vedere il tuo bot nel server Discord.

  </Step>

  <Step title="Abilita la Developer Mode e raccogli i tuoi ID">
    Tornando nell'app Discord, devi abilitare la Developer Mode per poter copiare gli ID interni.

    1. Fai clic su **User Settings** (icona a ingranaggio accanto al tuo avatar) → **Advanced** → attiva **Developer Mode**
    2. Fai clic con il pulsante destro sull'**icona del server** nella barra laterale → **Copy Server ID**
    3. Fai clic con il pulsante destro sul tuo **avatar** → **Copy User ID**

    Salva il tuo **Server ID** e **User ID** insieme al tuo Bot Token — nel passaggio successivo invierai tutti e tre a OpenClaw.

  </Step>

  <Step title="Consenti i DM dai membri del server">
    Per far funzionare l'associazione, Discord deve consentire al tuo bot di inviarti DM. Fai clic con il pulsante destro sull'**icona del server** → **Privacy Settings** → attiva **Direct Messages**.

    Questo permette ai membri del server (inclusi i bot) di inviarti DM. Lascia questa opzione abilitata se vuoi usare i DM di Discord con OpenClaw. Se prevedi di usare solo i canali guild, puoi disabilitare i DM dopo l'associazione.

  </Step>

  <Step title="Imposta il token del tuo bot in modo sicuro (non inviarlo in chat)">
    Il token del tuo bot Discord è un segreto (come una password). Impostalo sulla macchina che esegue OpenClaw prima di inviare messaggi al tuo agente.

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN --dry-run
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set channels.discord.enabled true --strict-json
openclaw gateway
```

    Se OpenClaw è già in esecuzione come servizio in background, riavvialo tramite l'app Mac di OpenClaw oppure arrestando e riavviando il processo `openclaw gateway run`.

  </Step>

  <Step title="Configura OpenClaw ed esegui l'associazione">

    <Tabs>
      <Tab title="Chiedi al tuo agente">
        Chatta con il tuo agente OpenClaw su qualsiasi canale esistente (ad esempio Telegram) e diglielo. Se Discord è il tuo primo canale, usa invece la scheda CLI / config.

        > "Ho già impostato il token del mio bot Discord nella config. Completa la configurazione di Discord con User ID `<user_id>` e Server ID `<server_id>`."
      </Tab>
      <Tab title="CLI / config">
        Se preferisci una config basata su file, imposta:

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: {
        source: "env",
        provider: "default",
        id: "DISCORD_BOT_TOKEN",
      },
    },
  },
}
```

        Fallback env per l'account predefinito:

```bash
DISCORD_BOT_TOKEN=...
```

        Sono supportati valori `token` in testo semplice. Sono supportati anche valori SecretRef per `channels.discord.token` tra provider env/file/exec. Vedi [Secrets Management](/it/gateway/secrets).

      </Tab>
    </Tabs>

  </Step>

  <Step title="Approva la prima associazione DM">
    Attendi che il gateway sia in esecuzione, poi invia un DM al tuo bot in Discord. Ti risponderà con un codice di associazione.

    <Tabs>
      <Tab title="Chiedi al tuo agente">
        Invia il codice di associazione al tuo agente sul canale esistente:

        > "Approva questo codice di associazione Discord: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    I codici di associazione scadono dopo 1 ora.

    Ora dovresti essere in grado di chattare con il tuo agente in Discord tramite DM.

  </Step>
</Steps>

<Note>
La risoluzione del token è consapevole dell'account. I valori token nella config hanno priorità sul fallback env. `DISCORD_BOT_TOKEN` viene usato solo per l'account predefinito.
Per chiamate in uscita avanzate (strumento messaggi/azioni canale), viene usato un `token` esplicito per quella chiamata. Questo vale per azioni di invio e di lettura/sonda (ad esempio read/search/fetch/thread/pins/permissions). Le impostazioni di policy/retry dell'account provengono comunque dall'account selezionato nello snapshot runtime attivo.
</Note>

## Consigliato: configura uno spazio di lavoro guild

Una volta che i DM funzionano, puoi configurare il tuo server Discord come spazio di lavoro completo in cui ogni canale ottiene la propria sessione agente con il proprio contesto. Questa è l'opzione consigliata per i server privati dove ci siete solo tu e il tuo bot.

<Steps>
  <Step title="Aggiungi il tuo server alla allowlist delle guild">
    Questo consente al tuo agente di rispondere in qualsiasi canale del tuo server, non solo nei DM.

    <Tabs>
      <Tab title="Chiedi al tuo agente">
        > "Aggiungi il mio Discord Server ID `<server_id>` alla allowlist delle guild"
      </Tab>
      <Tab title="Config">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Consenti risposte senza @mention">
    Per impostazione predefinita, il tuo agente risponde nei canali guild solo quando viene menzionato con @. Per un server privato, probabilmente vorrai che risponda a ogni messaggio.

    <Tabs>
      <Tab title="Chiedi al tuo agente">
        > "Consenti al mio agente di rispondere su questo server senza dover essere menzionato con @"
      </Tab>
      <Tab title="Config">
        Imposta `requireMention: false` nella config della tua guild:

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="Pianifica la memoria nei canali guild">
    Per impostazione predefinita, la memoria a lungo termine (MEMORY.md) viene caricata solo nelle sessioni DM. I canali guild non caricano automaticamente MEMORY.md.

    <Tabs>
      <Tab title="Chiedi al tuo agente">
        > "Quando faccio domande nei canali Discord, usa memory_search o memory_get se hai bisogno del contesto a lungo termine da MEMORY.md."
      </Tab>
      <Tab title="Manuale">
        Se hai bisogno di contesto condiviso in ogni canale, inserisci le istruzioni stabili in `AGENTS.md` o `USER.md` (vengono iniettati in ogni sessione). Mantieni le note a lungo termine in `MEMORY.md` e accedivi su richiesta con gli strumenti di memoria.
      </Tab>
    </Tabs>

  </Step>
</Steps>

Ora crea alcuni canali sul tuo server Discord e inizia a chattare. Il tuo agente può vedere il nome del canale e ogni canale ottiene la propria sessione isolata — così puoi configurare `#coding`, `#home`, `#research` o qualsiasi altra cosa si adatti al tuo flusso di lavoro.

## Modello di runtime

- Il gateway gestisce la connessione Discord.
- Il routing delle risposte è deterministico: le risposte in ingresso da Discord tornano su Discord.
- Per impostazione predefinita (`session.dmScope=main`), le chat dirette condividono la sessione principale dell'agente (`agent:main:main`).
- I canali guild sono chiavi di sessione isolate (`agent:<agentId>:discord:channel:<channelId>`).
- I DM di gruppo vengono ignorati per impostazione predefinita (`channels.discord.dm.groupEnabled=false`).
- I comandi slash nativi vengono eseguiti in sessioni di comando isolate (`agent:<agentId>:discord:slash:<userId>`), mantenendo comunque `CommandTargetSessionKey` verso la sessione di conversazione instradata.

## Canali forum

I canali forum e media di Discord accettano solo post in thread. OpenClaw supporta due modi per crearli:

- Invia un messaggio al parent del forum (`channel:<forumId>`) per creare automaticamente un thread. Il titolo del thread usa la prima riga non vuota del tuo messaggio.
- Usa `openclaw message thread create` per creare direttamente un thread. Non passare `--message-id` per i canali forum.

Esempio: invia al parent del forum per creare un thread

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

Esempio: crea esplicitamente un thread del forum

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

I parent del forum non accettano componenti Discord. Se hai bisogno di componenti, invia al thread stesso (`channel:<threadId>`).

## Componenti interattivi

OpenClaw supporta i container Discord components v2 per i messaggi dell'agente. Usa lo strumento messaggi con un payload `components`. I risultati delle interazioni vengono instradati di nuovo all'agente come normali messaggi in ingresso e seguono le impostazioni Discord `replyToMode` esistenti.

Blocchi supportati:

- `text`, `section`, `separator`, `actions`, `media-gallery`, `file`
- Le righe di azione consentono fino a 5 pulsanti oppure un singolo menu di selezione
- Tipi di selezione: `string`, `user`, `role`, `mentionable`, `channel`

Per impostazione predefinita, i componenti sono monouso. Imposta `components.reusable=true` per consentire a pulsanti, selezioni e moduli di essere usati più volte fino alla loro scadenza.

Per limitare chi può fare clic su un pulsante, imposta `allowedUsers` su quel pulsante (ID utente Discord, tag o `*`). Quando configurato, gli utenti non corrispondenti ricevono un rifiuto effimero.

I comandi slash `/model` e `/models` aprono un selettore di modelli interattivo con menu a discesa per provider e modello più un passaggio Submit. La risposta del selettore è effimera e solo l'utente che ha invocato il comando può usarla.

Allegati file:

- I blocchi `file` devono puntare a un riferimento allegato (`attachment://<filename>`)
- Fornisci l'allegato tramite `media`/`path`/`filePath` (file singolo); usa `media-gallery` per più file
- Usa `filename` per sovrascrivere il nome di upload quando deve corrispondere al riferimento allegato

Moduli modali:

- Aggiungi `components.modal` con fino a 5 campi
- Tipi di campo: `text`, `checkbox`, `radio`, `select`, `role-select`, `user-select`
- OpenClaw aggiunge automaticamente un pulsante di attivazione

Esempio:

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Optional fallback text",
  components: {
    reusable: true,
    text: "Choose a path",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Approve",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Decline", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Pick an option",
          options: [
            { label: "Option A", value: "a" },
            { label: "Option B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Details",
      triggerLabel: "Open form",
      fields: [
        { type: "text", label: "Requester" },
        {
          type: "select",
          label: "Priority",
          options: [
            { label: "Low", value: "low" },
            { label: "High", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## Controllo degli accessi e routing

<Tabs>
  <Tab title="Policy DM">
    `channels.discord.dmPolicy` controlla l'accesso ai DM (legacy: `channels.discord.dm.policy`):

    - `pairing` (predefinito)
    - `allowlist`
    - `open` (richiede che `channels.discord.allowFrom` includa `"*"`; legacy: `channels.discord.dm.allowFrom`)
    - `disabled`

    Se la policy DM non è open, gli utenti sconosciuti vengono bloccati (oppure viene richiesto l'abbinamento in modalità `pairing`).

    Precedenza multi-account:

    - `channels.discord.accounts.default.allowFrom` si applica solo all'account `default`.
    - Gli account nominati ereditano `channels.discord.allowFrom` quando il proprio `allowFrom` non è impostato.
    - Gli account nominati non ereditano `channels.discord.accounts.default.allowFrom`.

    Formato target DM per la consegna:

    - `user:<id>`
    - menzione `<@id>`

    Gli ID numerici semplici sono ambigui e vengono rifiutati a meno che non venga fornito un tipo target utente/canale esplicito.

  </Tab>

  <Tab title="Policy guild">
    La gestione delle guild è controllata da `channels.discord.groupPolicy`:

    - `open`
    - `allowlist`
    - `disabled`

    La baseline sicura quando esiste `channels.discord` è `allowlist`.

    Comportamento di `allowlist`:

    - la guild deve corrispondere a `channels.discord.guilds` (`id` preferito, slug accettato)
    - allowlist facoltative del mittente: `users` (si consigliano ID stabili) e `roles` (solo ID ruolo); se una delle due è configurata, i mittenti sono consentiti quando corrispondono a `users` O `roles`
    - la corrispondenza diretta nome/tag è disabilitata per impostazione predefinita; abilita `channels.discord.dangerouslyAllowNameMatching: true` solo come modalità di compatibilità di emergenza
    - nomi/tag sono supportati per `users`, ma gli ID sono più sicuri; `openclaw security audit` avvisa quando vengono usate voci nome/tag
    - se una guild ha `channels` configurati, i canali non elencati vengono negati
    - se una guild non ha un blocco `channels`, tutti i canali in quella guild in allowlist sono consentiti

    Esempio:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    Se imposti solo `DISCORD_BOT_TOKEN` e non crei un blocco `channels.discord`, il fallback runtime è `groupPolicy="allowlist"` (con un avviso nei log), anche se `channels.defaults.groupPolicy` è `open`.

  </Tab>

  <Tab title="Menzioni e DM di gruppo">
    I messaggi guild sono protetti da menzione per impostazione predefinita.

    Il rilevamento delle menzioni include:

    - menzione esplicita del bot
    - pattern di menzione configurati (`agents.list[].groupChat.mentionPatterns`, fallback `messages.groupChat.mentionPatterns`)
    - comportamento implicito reply-to-bot nei casi supportati

    `requireMention` viene configurato per guild/canale (`channels.discord.guilds...`).
    `ignoreOtherMentions` elimina facoltativamente i messaggi che menzionano un altro utente/ruolo ma non il bot (escludendo @everyone/@here).

    DM di gruppo:

    - predefinito: ignorati (`dm.groupEnabled=false`)
    - allowlist facoltativa tramite `dm.groupChannels` (ID canale o slug)

  </Tab>
</Tabs>

### Routing dell'agente basato sui ruoli

Usa `bindings[].match.roles` per instradare i membri delle guild Discord a diversi agenti in base all'ID del ruolo. I binding basati sui ruoli accettano solo ID ruolo e vengono valutati dopo i binding peer o parent-peer e prima dei binding solo guild. Se un binding imposta anche altri campi di corrispondenza (ad esempio `peer` + `guildId` + `roles`), tutti i campi configurati devono corrispondere.

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Configurazione del Developer Portal

<AccordionGroup>
  <Accordion title="Crea app e bot">

    1. Discord Developer Portal -> **Applications** -> **New Application**
    2. **Bot** -> **Add Bot**
    3. Copia il token del bot

  </Accordion>

  <Accordion title="Intent privilegiati">
    In **Bot -> Privileged Gateway Intents**, abilita:

    - Message Content Intent
    - Server Members Intent (consigliato)

    Presence intent è facoltativo ed è richiesto solo se vuoi ricevere aggiornamenti di presenza. Impostare la presenza del bot (`setPresence`) non richiede di abilitare gli aggiornamenti di presenza per i membri.

  </Accordion>

  <Accordion title="Scope OAuth e permessi di base">
    Generatore URL OAuth:

    - scope: `bot`, `applications.commands`

    Permessi di base tipici:

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions (facoltativo)

    Evita `Administrator` a meno che non sia esplicitamente necessario.

  </Accordion>

  <Accordion title="Copia gli ID">
    Abilita Discord Developer Mode, poi copia:

    - ID server
    - ID canale
    - ID utente

    Preferisci ID numerici nella config di OpenClaw per audit e probe affidabili.

  </Accordion>
</AccordionGroup>

## Comandi nativi e autorizzazione dei comandi

- `commands.native` è impostato per impostazione predefinita su `"auto"` ed è abilitato per Discord.
- Override per canale: `channels.discord.commands.native`.
- `commands.native=false` rimuove esplicitamente i comandi nativi Discord registrati in precedenza.
- L'autorizzazione dei comandi nativi usa le stesse allowlist/policy Discord della normale gestione dei messaggi.
- I comandi potrebbero comunque essere visibili nell'interfaccia Discord per utenti non autorizzati; l'esecuzione continua a far rispettare l'autorizzazione OpenClaw e restituisce "not authorized".

Vedi [Slash commands](/it/tools/slash-commands) per catalogo e comportamento dei comandi.

Impostazioni predefinite dei comandi slash:

- `ephemeral: true`

## Dettagli delle funzionalità

<AccordionGroup>
  <Accordion title="Tag di risposta e risposte native">
    Discord supporta tag di risposta nell'output dell'agente:

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    Controllato da `channels.discord.replyToMode`:

    - `off` (predefinito)
    - `first`
    - `all`
    - `batched`

    Nota: `off` disabilita il threading implicito delle risposte. I tag espliciti `[[reply_to_*]]` vengono comunque rispettati.
    `first` allega sempre il riferimento di risposta nativa implicita al primo messaggio Discord in uscita del turno.
    `batched` allega il riferimento di risposta nativa implicita di Discord solo quando il
    turno in ingresso era un batch sottoposto a debounce di più messaggi. Questo è utile
    quando vuoi risposte native soprattutto per chat bursty ambigue, non per ogni
    singolo turno a messaggio singolo.

    Gli ID dei messaggi emergono in contesto/storico così gli agenti possono indirizzare messaggi specifici.

  </Accordion>

  <Accordion title="Anteprima streaming live">
    OpenClaw può trasmettere in streaming bozze di risposta inviando un messaggio temporaneo e modificandolo man mano che arriva il testo.

    - `channels.discord.streaming` controlla lo streaming di anteprima (`off` | `partial` | `block` | `progress`, predefinito: `off`).
    - Il valore predefinito resta `off` perché le modifiche di anteprima Discord possono raggiungere rapidamente i limiti di velocità, specialmente quando più bot o gateway condividono lo stesso account o traffico guild.
    - `progress` è accettato per coerenza cross-channel e viene mappato a `partial` su Discord.
    - `channels.discord.streamMode` è un alias legacy e viene migrato automaticamente.
    - `partial` modifica un singolo messaggio di anteprima man mano che arrivano i token.
    - `block` emette blocchi di dimensione bozza (usa `draftChunk` per regolare dimensione e punti di interruzione).

    Esempio:

```json5
{
  channels: {
    discord: {
      streaming: "partial",
    },
  },
}
```

    Valori predefiniti del chunking in modalità `block` (limitati da `channels.discord.textChunkLimit`):

```json5
{
  channels: {
    discord: {
      streaming: "block",
      draftChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph",
      },
    },
  },
}
```

    Lo streaming di anteprima è solo testo; le risposte multimediali tornano alla consegna normale.

    Nota: lo streaming di anteprima è separato dallo streaming a blocchi. Quando lo streaming a blocchi viene esplicitamente
    abilitato per Discord, OpenClaw salta il flusso di anteprima per evitare il doppio streaming.

  </Accordion>

  <Accordion title="Cronologia, contesto e comportamento dei thread">
    Contesto della cronologia guild:

    - `channels.discord.historyLimit` predefinito `20`
    - fallback: `messages.groupChat.historyLimit`
    - `0` disabilita

    Controlli cronologia DM:

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    Comportamento dei thread:

    - I thread Discord vengono instradati come sessioni canale
    - i metadati del thread parent possono essere usati per il collegamento alla sessione parent
    - la config del thread eredita la config del canale parent a meno che non esista una voce specifica per il thread

    Gli argomenti del canale vengono iniettati come contesto **non affidabile** (non come system prompt).
    Il contesto della risposta e del messaggio citato attualmente rimane così come ricevuto.
    Le allowlist Discord limitano principalmente chi può attivare l'agente, non rappresentano un confine completo di redazione del contesto supplementare.

  </Accordion>

  <Accordion title="Sessioni legate al thread per i subagent">
    Discord può associare un thread a una destinazione di sessione così che i messaggi successivi in quel thread continuino a essere instradati alla stessa sessione (incluse le sessioni di subagent).

    Comandi:

    - `/focus <target>` associa il thread corrente/nuovo a una destinazione subagent/sessione
    - `/unfocus` rimuove l'associazione del thread corrente
    - `/agents` mostra le esecuzioni attive e lo stato dell'associazione
    - `/session idle <duration|off>` controlla/aggiorna la rimozione automatica dell'associazione per inattività per i binding focalizzati
    - `/session max-age <duration|off>` controlla/aggiorna l'età massima rigida per i binding focalizzati

    Config:

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
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // opt-in
      },
    },
  },
}
```

    Note:

    - `session.threadBindings.*` imposta i valori predefiniti globali.
    - `channels.discord.threadBindings.*` sovrascrive il comportamento Discord.
    - `spawnSubagentSessions` deve essere true per creare/associare automaticamente thread per `sessions_spawn({ thread: true })`.
    - `spawnAcpSessions` deve essere true per creare/associare automaticamente thread per ACP (`/acp spawn ... --thread ...` o `sessions_spawn({ runtime: "acp", thread: true })`).
    - Se i thread binding sono disabilitati per un account, `/focus` e le relative operazioni di thread binding non sono disponibili.

    Vedi [Sub-agents](/it/tools/subagents), [ACP Agents](/it/tools/acp-agents) e [Configuration Reference](/it/gateway/configuration-reference).

  </Accordion>

  <Accordion title="Binding persistenti dei canali ACP">
    Per spazi di lavoro ACP stabili "sempre attivi", configura binding ACP tipizzati di livello superiore destinati a conversazioni Discord.

    Percorso config:

    - `bindings[]` con `type: "acp"` e `match.channel: "discord"`

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
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    Note:

    - `/acp spawn codex --bind here` associa il canale o thread Discord corrente sul posto e mantiene i messaggi futuri instradati alla stessa sessione ACP.
    - Questo può comunque significare "avvia una nuova sessione ACP Codex", ma non crea di per sé un nuovo thread Discord. Il canale esistente rimane la superficie di chat.
    - Codex può comunque essere eseguito nel proprio `cwd` o spazio di lavoro backend su disco. Quello spazio di lavoro è stato runtime, non un thread Discord.
    - I messaggi del thread possono ereditare il binding ACP del canale parent.
    - In un canale o thread associato, `/new` e `/reset` reimpostano la stessa sessione ACP sul posto.
    - I thread binding temporanei continuano a funzionare e possono sovrascrivere la risoluzione della destinazione mentre sono attivi.
    - `spawnAcpSessions` è richiesto solo quando OpenClaw deve creare/associare un thread figlio tramite `--thread auto|here`. Non è richiesto per `/acp spawn ... --bind here` nel canale corrente.

    Vedi [ACP Agents](/it/tools/acp-agents) per i dettagli sul comportamento dei binding.

  </Accordion>

  <Accordion title="Notifiche di reazione">
    Modalità di notifica delle reazioni per guild:

    - `off`
    - `own` (predefinito)
    - `all`
    - `allowlist` (usa `guilds.<id>.users`)

    Gli eventi di reazione vengono trasformati in eventi di sistema e allegati alla sessione Discord instradata.

  </Accordion>

  <Accordion title="Reazioni di conferma">
    `ackReaction` invia un'emoji di conferma mentre OpenClaw sta elaborando un messaggio in ingresso.

    Ordine di risoluzione:

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - fallback emoji identità agente (`agents.list[].identity.emoji`, altrimenti "👀")

    Note:

    - Discord accetta emoji Unicode o nomi di emoji personalizzate.
    - Usa `""` per disabilitare la reazione per un canale o un account.

  </Accordion>

  <Accordion title="Scritture config">
    Le scritture config avviate dal canale sono abilitate per impostazione predefinita.

    Questo influisce sui flussi `/config set|unset` (quando le funzionalità dei comandi sono abilitate).

    Disabilita:

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="Proxy gateway">
    Instrada il traffico WebSocket del gateway Discord e le lookup REST di avvio (ID applicazione + risoluzione allowlist) attraverso un proxy HTTP(S) con `channels.discord.proxy`.

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    Override per account:

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="Supporto PluralKit">
    Abilita la risoluzione PluralKit per mappare i messaggi proxy all'identità del membro del sistema:

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // facoltativo; necessario per sistemi privati
      },
    },
  },
}
```

    Note:

    - le allowlist possono usare `pk:<memberId>`
    - i nomi visualizzati dei membri corrispondono per nome/slug solo quando `channels.discord.dangerouslyAllowNameMatching: true`
    - le lookup usano l'ID messaggio originale e sono vincolate da una finestra temporale
    - se la lookup fallisce, i messaggi proxy vengono trattati come messaggi bot e scartati a meno che `allowBots=true`

  </Accordion>

  <Accordion title="Configurazione della presenza">
    Gli aggiornamenti di presenza vengono applicati quando imposti un campo di stato o attività, oppure quando abiliti la presenza automatica.

    Esempio solo stato:

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    Esempio attività (lo stato personalizzato è il tipo di attività predefinito):

```json5
{
  channels: {
    discord: {
      activity: "Focus time",
      activityType: 4,
    },
  },
}
```

    Esempio streaming:

```json5
{
  channels: {
    discord: {
      activity: "Live coding",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    Mappa dei tipi di attività:

    - 0: Playing
    - 1: Streaming (richiede `activityUrl`)
    - 2: Listening
    - 3: Watching
    - 4: Custom (usa il testo dell'attività come stato; l'emoji è facoltativa)
    - 5: Competing

    Esempio di presenza automatica (segnale di salute runtime):

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token exhausted",
      },
    },
  },
}
```

    La presenza automatica mappa la disponibilità runtime allo stato Discord: healthy => online, degraded o unknown => idle, exhausted o unavailable => dnd. Override testuali facoltativi:

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText` (supporta il placeholder `{reason}`)

  </Accordion>

  <Accordion title="Approvals in Discord">
    Discord supporta la gestione delle approvazioni basata su pulsanti nei DM e può facoltativamente pubblicare prompt di approvazione nel canale di origine.

    Percorso config:

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers` (facoltativo; usa `commands.ownerAllowFrom` come fallback quando possibile)
    - `channels.discord.execApprovals.target` (`dm` | `channel` | `both`, predefinito: `dm`)
    - `agentFilter`, `sessionFilter`, `cleanupAfterResolve`

    Discord abilita automaticamente le approvazioni exec native quando `enabled` non è impostato oppure è `"auto"` e può essere risolto almeno un approvatore, da `execApprovals.approvers` o da `commands.ownerAllowFrom`. Discord non deduce gli approvatori exec da `allowFrom` del canale, dal legacy `dm.allowFrom` o da `defaultTo` dei messaggi diretti. Imposta `enabled: false` per disabilitare esplicitamente Discord come client di approvazione nativo.

    Quando `target` è `channel` o `both`, il prompt di approvazione è visibile nel canale. Solo gli approvatori risolti possono usare i pulsanti; gli altri utenti ricevono un rifiuto effimero. I prompt di approvazione includono il testo del comando, quindi abilita la consegna nel canale solo in canali attendibili. Se l'ID canale non può essere derivato dalla chiave di sessione, OpenClaw torna alla consegna DM.

    Discord rende anche i pulsanti di approvazione condivisi usati da altri canali chat. L'adapter Discord nativo aggiunge principalmente il routing DM degli approvatori e il fanout verso il canale.
    Quando questi pulsanti sono presenti, rappresentano l'UX primaria di approvazione; OpenClaw
    dovrebbe includere un comando manuale `/approve` solo quando il risultato dello strumento dice
    che le approvazioni chat non sono disponibili o che l'approvazione manuale è l'unico percorso.

    L'autenticazione del gateway per questo gestore usa lo stesso contratto condiviso di risoluzione delle credenziali degli altri client Gateway:

    - autenticazione locale env-first (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` poi `gateway.auth.*`)
    - in modalità locale, `gateway.remote.*` può essere usato come fallback solo quando `gateway.auth.*` non è impostato; SecretRef locali configurati ma non risolti falliscono in modalità chiusa
    - supporto modalità remota tramite `gateway.remote.*` quando applicabile
    - gli override URL sono sicuri per gli override: gli override CLI non riutilizzano credenziali implicite e gli override env usano solo credenziali env

    Comportamento della risoluzione delle approvazioni:

    - Gli ID con prefisso `plugin:` vengono risolti tramite `plugin.approval.resolve`.
    - Gli altri ID vengono risolti tramite `exec.approval.resolve`.
    - Discord qui non esegue un ulteriore fallback exec-to-plugin; il
      prefisso dell'id decide quale metodo gateway chiamare.

    Le approvazioni exec scadono dopo 30 minuti per impostazione predefinita. Se le approvazioni falliscono con
    ID approvazione sconosciuti, verifica la risoluzione degli approvatori, l'abilitazione della funzionalità e
    che il tipo di id di approvazione consegnato corrisponda alla richiesta in sospeso.

    Documentazione correlata: [Exec approvals](/it/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## Strumenti e gate delle azioni

Le azioni dei messaggi Discord includono messaggistica, amministrazione del canale, moderazione, presenza e azioni sui metadati.

Esempi principali:

- messaggistica: `sendMessage`, `readMessages`, `editMessage`, `deleteMessage`, `threadReply`
- reazioni: `react`, `reactions`, `emojiList`
- moderazione: `timeout`, `kick`, `ban`
- presenza: `setPresence`

I gate delle azioni si trovano sotto `channels.discord.actions.*`.

Comportamento predefinito dei gate:

| Gruppo di azioni                                                                                                                                                         | Predefinito |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | abilitato   |
| roles                                                                                                                                                                    | disabilitato |
| moderation                                                                                                                                                               | disabilitato |
| presence                                                                                                                                                                 | disabilitato |

## UI Components v2

OpenClaw usa Discord components v2 per exec approvals e marcatori cross-context. Le azioni dei messaggi Discord possono anche accettare `components` per UI personalizzate (avanzato; richiede la costruzione di un payload di componenti tramite lo strumento discord), mentre i legacy `embeds` restano disponibili ma non sono consigliati.

- `channels.discord.ui.components.accentColor` imposta il colore di accento usato dai container dei componenti Discord (hex).
- Imposta per account con `channels.discord.accounts.<id>.ui.components.accentColor`.
- Gli `embeds` vengono ignorati quando sono presenti components v2.

Esempio:

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## Canali vocali

OpenClaw può unirsi ai canali vocali Discord per conversazioni continue in tempo reale. Questo è separato dagli allegati dei messaggi vocali.

Requisiti:

- Abilita i comandi nativi (`commands.native` o `channels.discord.commands.native`).
- Configura `channels.discord.voice`.
- Il bot deve avere i permessi Connect + Speak nel canale vocale di destinazione.

Usa il comando nativo solo Discord `/vc join|leave|status` per controllare le sessioni. Il comando usa l'agente predefinito dell'account e segue le stesse regole di allowlist e group policy degli altri comandi Discord.

Esempio di auto-join:

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
    },
  },
}
```

Note:

- `voice.tts` sovrascrive `messages.tts` solo per la riproduzione vocale.
- I turni di trascrizione vocale derivano lo stato del proprietario da Discord `allowFrom` (o `dm.allowFrom`); gli speaker non proprietari non possono accedere agli strumenti riservati al proprietario (ad esempio `gateway` e `cron`).
- La voce è abilitata per impostazione predefinita; imposta `channels.discord.voice.enabled=false` per disabilitarla.
- `voice.daveEncryption` e `voice.decryptionFailureTolerance` vengono passati alle opzioni di join di `@discordjs/voice`.
- I valori predefiniti di `@discordjs/voice` sono `daveEncryption=true` e `decryptionFailureTolerance=24` se non impostati.
- OpenClaw monitora anche gli errori di decrittazione in ricezione e si ripristina automaticamente uscendo e rientrando nel canale vocale dopo errori ripetuti in una breve finestra di tempo.
- Se i log di ricezione mostrano ripetutamente `DecryptionFailed(UnencryptedWhenPassthroughDisabled)`, questo potrebbe essere il bug di ricezione upstream di `@discordjs/voice` tracciato in [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419).

## Messaggi vocali

I messaggi vocali Discord mostrano un'anteprima della forma d'onda e richiedono audio OGG/Opus più metadati. OpenClaw genera automaticamente la forma d'onda, ma richiede `ffmpeg` e `ffprobe` disponibili sull'host gateway per ispezionare e convertire i file audio.

Requisiti e vincoli:

- Fornisci un **percorso file locale** (gli URL vengono rifiutati).
- Ometti il contenuto testuale (Discord non consente testo + messaggio vocale nello stesso payload).
- Qualsiasi formato audio è accettato; OpenClaw converte in OGG/Opus quando necessario.

Esempio:

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## Risoluzione dei problemi

<AccordionGroup>
  <Accordion title="Sono stati usati intent non consentiti o il bot non vede messaggi guild">

    - abilita Message Content Intent
    - abilita Server Members Intent quando dipendi dalla risoluzione utente/membro
    - riavvia il gateway dopo aver modificato gli intent

  </Accordion>

  <Accordion title="Messaggi guild bloccati inaspettatamente">

    - verifica `groupPolicy`
    - verifica la allowlist guild in `channels.discord.guilds`
    - se esiste una mappa `guild.channels`, sono consentiti solo i canali elencati
    - verifica il comportamento di `requireMention` e i pattern di menzione

    Controlli utili:

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="Require mention false ma ancora bloccato">
    Cause comuni:

    - `groupPolicy="allowlist"` senza allowlist guild/canale corrispondente
    - `requireMention` configurato nel posto sbagliato (deve trovarsi sotto `channels.discord.guilds` o nella voce del canale)
    - mittente bloccato dalla allowlist `users` della guild/del canale

  </Accordion>

  <Accordion title="Gli handler di lunga durata vanno in timeout o duplicano le risposte">

    Log tipici:

    - `Listener DiscordMessageListener timed out after 30000ms for event MESSAGE_CREATE`
    - `Slow listener detected ...`
    - `discord inbound worker timed out after ...`

    Parametro budget del listener:

    - account singolo: `channels.discord.eventQueue.listenerTimeout`
    - multi-account: `channels.discord.accounts.<accountId>.eventQueue.listenerTimeout`

    Parametro timeout esecuzione worker:

    - account singolo: `channels.discord.inboundWorker.runTimeoutMs`
    - multi-account: `channels.discord.accounts.<accountId>.inboundWorker.runTimeoutMs`
    - predefinito: `1800000` (30 minuti); imposta `0` per disabilitare

    Baseline consigliata:

```json5
{
  channels: {
    discord: {
      accounts: {
        default: {
          eventQueue: {
            listenerTimeout: 120000,
          },
          inboundWorker: {
            runTimeoutMs: 1800000,
          },
        },
      },
    },
  },
}
```

    Usa `eventQueue.listenerTimeout` per una configurazione lenta del listener e `inboundWorker.runTimeoutMs`
    solo se vuoi una valvola di sicurezza separata per i turni agente in coda.

  </Accordion>

  <Accordion title="Incongruenze nell'audit dei permessi">
    I controlli dei permessi di `channels status --probe` funzionano solo per ID canale numerici.

    Se usi chiavi slug, la corrispondenza runtime può comunque funzionare, ma il probe non può verificare completamente i permessi.

  </Accordion>

  <Accordion title="Problemi con DM e associazione">

    - DM disabilitato: `channels.discord.dm.enabled=false`
    - policy DM disabilitata: `channels.discord.dmPolicy="disabled"` (legacy: `channels.discord.dm.policy`)
    - approvazione dell'associazione in attesa in modalità `pairing`

  </Accordion>

  <Accordion title="Loop bot-to-bot">
    Per impostazione predefinita, i messaggi scritti dai bot vengono ignorati.

    Se imposti `channels.discord.allowBots=true`, usa regole severe di menzione e allowlist per evitare comportamenti a loop.
    Preferisci `channels.discord.allowBots="mentions"` per accettare solo messaggi bot che menzionano il bot.

  </Accordion>

  <Accordion title="La STT vocale perde con DecryptionFailed(...)">

    - mantieni OpenClaw aggiornato (`openclaw update`) in modo che la logica di recupero della ricezione vocale Discord sia presente
    - conferma `channels.discord.voice.daveEncryption=true` (predefinito)
    - parti da `channels.discord.voice.decryptionFailureTolerance=24` (predefinito upstream) e regola solo se necessario
    - osserva i log per:
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - se gli errori continuano dopo il rejoin automatico, raccogli i log e confrontali con [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419)

  </Accordion>
</AccordionGroup>

## Riferimenti alla configurazione

Riferimento principale:

- [Configuration reference - Discord](/it/gateway/configuration-reference#discord)

Campi Discord ad alto segnale:

- avvio/auth: `enabled`, `token`, `accounts.*`, `allowBots`
- policy: `groupPolicy`, `dm.*`, `guilds.*`, `guilds.*.channels.*`
- comando: `commands.native`, `commands.useAccessGroups`, `configWrites`, `slashCommand.*`
- coda eventi: `eventQueue.listenerTimeout` (budget listener), `eventQueue.maxQueueSize`, `eventQueue.maxConcurrency`
- worker inbound: `inboundWorker.runTimeoutMs`
- risposta/storico: `replyToMode`, `historyLimit`, `dmHistoryLimit`, `dms.*.historyLimit`
- consegna: `textChunkLimit`, `chunkMode`, `maxLinesPerMessage`
- streaming: `streaming` (alias legacy: `streamMode`), `draftChunk`, `blockStreaming`, `blockStreamingCoalesce`
- media/retry: `mediaMaxMb`, `retry`
  - `mediaMaxMb` limita gli upload Discord in uscita (predefinito: `100MB`)
- azioni: `actions.*`
- presenza: `activity`, `status`, `activityType`, `activityUrl`
- UI: `ui.components.accentColor`
- funzionalità: `threadBindings`, `bindings[]` di livello superiore (`type: "acp"`), `pluralkit`, `execApprovals`, `intents`, `agentComponents`, `heartbeat`, `responsePrefix`

## Sicurezza e operazioni

- Tratta i token del bot come segreti (`DISCORD_BOT_TOKEN` preferito in ambienti supervisionati).
- Concedi permessi Discord con privilegio minimo.
- Se il deploy/stato dei comandi è obsoleto, riavvia il gateway e ricontrolla con `openclaw channels status --probe`.

## Correlati

- [Pairing](/it/channels/pairing)
- [Groups](/it/channels/groups)
- [Channel routing](/it/channels/channel-routing)
- [Security](/it/gateway/security)
- [Multi-agent routing](/it/concepts/multi-agent)
- [Troubleshooting](/it/channels/troubleshooting)
- [Slash commands](/it/tools/slash-commands)
