---
read_when:
    - Configurare Matrix in OpenClaw
    - Configurare E2EE e la verifica di Matrix
summary: Stato del supporto Matrix, configurazione e esempi di configurazione
title: Matrix
x-i18n:
    generated_at: "2026-04-06T03:08:12Z"
    model: gpt-5.4
    provider: openai
    source_hash: 3e2d84c08d7d5b96db14b914e54f08d25334401cdd92eb890bc8dfb37b0ca2dc
    source_path: channels/matrix.md
    workflow: 15
---

# Matrix

Matrix è il plugin di canale Matrix incluso per OpenClaw.
Usa il `matrix-js-sdk` ufficiale e supporta DM, stanze, thread, contenuti multimediali, reazioni, sondaggi, posizione ed E2EE.

## Plugin incluso

Matrix è incluso come plugin incluso nelle versioni correnti di OpenClaw, quindi le
build pacchettizzate normali non richiedono un'installazione separata.

Se stai usando una build meno recente o un'installazione personalizzata che esclude Matrix, installalo
manualmente:

Installa da npm:

```bash
openclaw plugins install @openclaw/matrix
```

Installa da un checkout locale:

```bash
openclaw plugins install ./path/to/local/matrix-plugin
```

Vedi [Plugins](/it/tools/plugin) per il comportamento dei plugin e le regole di installazione.

## Configurazione

1. Assicurati che il plugin Matrix sia disponibile.
   - Le versioni pacchettizzate correnti di OpenClaw lo includono già.
   - Le installazioni meno recenti/personalizzate possono aggiungerlo manualmente con i comandi sopra.
2. Crea un account Matrix sul tuo homeserver.
3. Configura `channels.matrix` con uno dei seguenti:
   - `homeserver` + `accessToken`, oppure
   - `homeserver` + `userId` + `password`.
4. Riavvia il gateway.
5. Avvia un DM con il bot oppure invitalo in una stanza.

Percorsi di configurazione interattiva:

```bash
openclaw channels add
openclaw configure --section channels
```

Cosa chiede effettivamente la procedura guidata Matrix:

- URL dell'homeserver
- metodo di autenticazione: access token o password
- ID utente solo quando scegli l'autenticazione con password
- nome dispositivo facoltativo
- se abilitare E2EE
- se configurare ora l'accesso alla stanza Matrix

Comportamenti importanti della procedura guidata:

- Se le variabili d'ambiente di autenticazione Matrix esistono già per l'account selezionato e quell'account non ha già l'autenticazione salvata nella configurazione, la procedura guidata offre una scorciatoia tramite env e scrive solo `enabled: true` per quell'account.
- Quando aggiungi interattivamente un altro account Matrix, il nome account inserito viene normalizzato nell'ID account usato nella configurazione e nelle variabili env. Per esempio, `Ops Bot` diventa `ops-bot`.
- I prompt della allowlist DM accettano immediatamente valori completi `@user:server`. I nomi visualizzati funzionano solo quando la ricerca live nella directory trova una corrispondenza esatta; altrimenti la procedura guidata ti chiede di riprovare con un ID Matrix completo.
- I prompt della allowlist delle stanze accettano direttamente ID stanza e alias. Possono anche risolvere in tempo reale i nomi delle stanze a cui si è iscritti, ma i nomi non risolti vengono mantenuti solo come digitati durante la configurazione e vengono ignorati in seguito dalla risoluzione della allowlist a runtime. Preferisci `!room:server` o `#alias:server`.
- L'identità di stanza/sessione a runtime usa l'ID stabile della stanza Matrix. Gli alias dichiarati dalla stanza vengono usati solo come input di ricerca, non come chiave di sessione a lungo termine o identità stabile del gruppo.
- Per risolvere i nomi delle stanze prima di salvarli, usa `openclaw channels resolve --channel matrix "Project Room"`.

Configurazione minima basata su token:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Configurazione basata su password (il token viene memorizzato nella cache dopo il login):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

Matrix memorizza le credenziali nella cache in `~/.openclaw/credentials/matrix/`.
L'account predefinito usa `credentials.json`; gli account con nome usano `credentials-<account>.json`.
Quando lì esistono credenziali nella cache, OpenClaw considera Matrix configurato per setup, doctor e rilevamento dello stato del canale, anche se l'autenticazione corrente non è impostata direttamente nella configurazione.

Equivalenti come variabili d'ambiente (usati quando la chiave di configurazione non è impostata):

- `MATRIX_HOMESERVER`
- `MATRIX_ACCESS_TOKEN`
- `MATRIX_USER_ID`
- `MATRIX_PASSWORD`
- `MATRIX_DEVICE_ID`
- `MATRIX_DEVICE_NAME`

Per account non predefiniti, usa variabili env con ambito account:

- `MATRIX_<ACCOUNT_ID>_HOMESERVER`
- `MATRIX_<ACCOUNT_ID>_ACCESS_TOKEN`
- `MATRIX_<ACCOUNT_ID>_USER_ID`
- `MATRIX_<ACCOUNT_ID>_PASSWORD`
- `MATRIX_<ACCOUNT_ID>_DEVICE_ID`
- `MATRIX_<ACCOUNT_ID>_DEVICE_NAME`

Esempio per l'account `ops`:

- `MATRIX_OPS_HOMESERVER`
- `MATRIX_OPS_ACCESS_TOKEN`

Per l'ID account normalizzato `ops-bot`, usa:

- `MATRIX_OPS_X2D_BOT_HOMESERVER`
- `MATRIX_OPS_X2D_BOT_ACCESS_TOKEN`

Matrix esegue l'escape della punteggiatura negli ID account per mantenere le variabili env con ambito prive di collisioni.
Per esempio, `-` diventa `_X2D_`, quindi `ops-prod` viene mappato a `MATRIX_OPS_X2D_PROD_*`.

La procedura guidata interattiva offre la scorciatoia tramite variabile env solo quando quelle variabili env di autenticazione sono già presenti e l'account selezionato non ha già l'autenticazione Matrix salvata nella configurazione.

## Esempio di configurazione

Questa è una configurazione di base pratica con pairing DM, allowlist delle stanze ed E2EE abilitato:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: "partial",
    },
  },
}
```

## Anteprime in streaming

Lo streaming delle risposte Matrix è opt-in.

Imposta `channels.matrix.streaming` su `"partial"` quando vuoi che OpenClaw invii una singola anteprima live
della risposta, modifichi quell'anteprima sul posto mentre il modello genera il testo e poi la
finalizzi quando la risposta è completata:

```json5
{
  channels: {
    matrix: {
      streaming: "partial",
    },
  },
}
```

- `streaming: "off"` è l'impostazione predefinita. OpenClaw attende la risposta finale e la invia una sola volta.
- `streaming: "partial"` crea un messaggio di anteprima modificabile per il blocco corrente dell'assistente usando normali messaggi di testo Matrix. Questo preserva il comportamento legacy di Matrix di notifica basata sulla prima anteprima, quindi i client standard possono notificare sul primo testo in streaming invece che sul blocco completato.
- `streaming: "quiet"` crea un'anteprima modificabile silenziosa per il blocco corrente dell'assistente. Usalo solo quando configuri anche regole push lato destinatario per le modifiche finalizzate dell'anteprima.
- `blockStreaming: true` abilita messaggi di avanzamento Matrix separati. Con lo streaming di anteprima abilitato, Matrix mantiene la bozza live per il blocco corrente e conserva i blocchi completati come messaggi separati.
- Quando lo streaming di anteprima è attivo e `blockStreaming` è disattivato, Matrix modifica la bozza live sul posto e finalizza quello stesso evento quando il blocco o il turno termina.
- Se l'anteprima non entra più in un singolo evento Matrix, OpenClaw interrompe lo streaming di anteprima e torna alla consegna finale normale.
- Le risposte multimediali continuano a inviare normalmente gli allegati. Se un'anteprima obsoleta non può più essere riutilizzata in sicurezza, OpenClaw la redige prima di inviare la risposta multimediale finale.
- Le modifiche di anteprima comportano chiamate API Matrix aggiuntive. Lascia disattivato lo streaming se vuoi il comportamento più conservativo rispetto ai limiti di frequenza.

`blockStreaming` da solo non abilita le anteprime bozza.
Usa `streaming: "partial"` o `streaming: "quiet"` per le modifiche di anteprima; poi aggiungi `blockStreaming: true` solo se vuoi anche che i blocchi completati dell'assistente rimangano visibili come messaggi di avanzamento separati.

Se hai bisogno delle notifiche standard Matrix senza regole push personalizzate, usa `streaming: "partial"` per il comportamento con anteprima iniziale oppure lascia `streaming` disattivato per la consegna solo finale. Con `streaming: "off"`:

- `blockStreaming: true` invia ogni blocco completato come un normale messaggio Matrix che genera notifiche.
- `blockStreaming: false` invia solo la risposta finale completata come un normale messaggio Matrix che genera notifiche.

### Regole push self-hosted per anteprime finalizzate silenziose

Se gestisci la tua infrastruttura Matrix e vuoi che le anteprime silenziose notifichino solo quando un blocco o la
risposta finale è pronta, imposta `streaming: "quiet"` e aggiungi una regola push per utente per le modifiche finalizzate dell'anteprima.

Di solito questa è una configurazione dell'utente destinatario, non una modifica di configurazione globale dell'homeserver:

Mappa rapida prima di iniziare:

- utente destinatario = la persona che deve ricevere la notifica
- utente bot = l'account Matrix OpenClaw che invia la risposta
- usa l'access token dell'utente destinatario per le chiamate API qui sotto
- fai corrispondere `sender` nella regola push con l'MXID completo dell'utente bot

1. Configura OpenClaw per usare anteprime silenziose:

```json5
{
  channels: {
    matrix: {
      streaming: "quiet",
    },
  },
}
```

2. Assicurati che l'account destinatario riceva già le normali notifiche push Matrix. Le regole
   per anteprima silenziosa funzionano solo se quell'utente ha già pusher/dispositivi funzionanti.

3. Ottieni l'access token dell'utente destinatario.
   - Usa il token dell'utente che riceve, non quello del bot.
   - Riutilizzare un token di sessione client esistente è in genere la soluzione più semplice.
   - Se devi generare un token nuovo, puoi effettuare il login tramite la normale API Client-Server Matrix:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "@alice:example.org"
    },
    "password": "REDACTED"
  }'
```

4. Verifica che l'account destinatario abbia già pusher:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

Se questo restituisce zero pusher/dispositivi attivi, correggi prima le normali notifiche Matrix prima di aggiungere la
regola OpenClaw qui sotto.

OpenClaw contrassegna le modifiche finalizzate di anteprima solo testuale con:

```json
{
  "com.openclaw.finalized_preview": true
}
```

5. Crea una regola push di override per ogni account destinatario che deve ricevere queste notifiche:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

Sostituisci questi valori prima di eseguire il comando:

- `https://matrix.example.org`: URL base del tuo homeserver
- `$USER_ACCESS_TOKEN`: l'access token dell'utente che riceve
- `openclaw-finalized-preview-botname`: un ID regola univoco per questo bot per questo utente ricevente
- `@bot:example.org`: l'MXID del tuo bot Matrix OpenClaw, non l'MXID dell'utente ricevente

Importante per le configurazioni multi-bot:

- Le regole push sono indicizzate da `ruleId`. Rieseguire `PUT` sullo stesso ID regola aggiorna quella singola regola.
- Se un utente ricevente deve ricevere notifiche per più account bot Matrix OpenClaw, crea una regola per bot con un ID regola univoco per ogni corrispondenza `sender`.
- Un pattern semplice è `openclaw-finalized-preview-<botname>`, per esempio `openclaw-finalized-preview-ops` o `openclaw-finalized-preview-support`.

La regola viene valutata rispetto al mittente dell'evento:

- autentica usando il token dell'utente ricevente
- fai corrispondere `sender` con l'MXID del bot OpenClaw

6. Verifica che la regola esista:

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

7. Prova una risposta in streaming. In modalità silenziosa, la stanza dovrebbe mostrare una bozza di anteprima silenziosa e la
   modifica finale sul posto dovrebbe notificare una volta terminato il blocco o il turno.

Se devi rimuovere la regola in seguito, elimina quello stesso ID regola con il token dell'utente ricevente:

```bash
curl -sS -X DELETE \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

Note:

- Crea la regola con l'access token dell'utente ricevente, non con quello del bot.
- Le nuove regole `override` definite dall'utente vengono inserite prima delle regole di soppressione predefinite, quindi non serve alcun parametro di ordinamento aggiuntivo.
- Questo influisce solo sulle modifiche di anteprima solo testuale che OpenClaw può finalizzare in sicurezza sul posto. I fallback per contenuti multimediali e i fallback per anteprime obsolete usano comunque la normale consegna Matrix.
- Se `GET /_matrix/client/v3/pushers` non mostra pusher, l'utente non ha ancora una consegna push Matrix funzionante per questo account/dispositivo.

#### Synapse

Per Synapse, la configurazione sopra di solito è sufficiente da sola:

- Non è richiesta alcuna modifica speciale a `homeserver.yaml` per le notifiche di anteprima OpenClaw finalizzate.
- Se il deployment Synapse invia già normali notifiche push Matrix, il token utente + la chiamata `pushrules` sopra sono il principale passaggio di configurazione.
- Se esegui Synapse dietro un reverse proxy o worker, assicurati che `/_matrix/client/.../pushrules/` raggiunga correttamente Synapse.
- Se esegui worker Synapse, assicurati che i pusher siano integri. La consegna push è gestita dal processo principale o da `synapse.app.pusher` / dai worker pusher configurati.

#### Tuwunel

Per Tuwunel, usa lo stesso flusso di configurazione e la chiamata API `pushrules` mostrata sopra:

- Non è richiesta alcuna configurazione specifica di Tuwunel per il marcatore di anteprima finalizzata stesso.
- Se le normali notifiche Matrix funzionano già per quell'utente, il token utente + la chiamata `pushrules` sopra sono il principale passaggio di configurazione.
- Se le notifiche sembrano scomparire mentre l'utente è attivo su un altro dispositivo, verifica se `suppress_push_when_active` è abilitato. Tuwunel ha aggiunto questa opzione in Tuwunel 1.4.2 il 12 settembre 2025 e può sopprimere intenzionalmente le notifiche push verso altri dispositivi mentre un dispositivo è attivo.

## Crittografia e verifica

Nelle stanze crittografate (E2EE), gli eventi immagine in uscita usano `thumbnail_file` così le anteprime immagine vengono crittografate insieme all'allegato completo. Le stanze non crittografate continuano a usare `thumbnail_url` semplice. Non è necessaria alcuna configurazione: il plugin rileva automaticamente lo stato E2EE.

### Stanze bot-to-bot

Per impostazione predefinita, i messaggi Matrix provenienti da altri account Matrix OpenClaw configurati vengono ignorati.

Usa `allowBots` quando vuoi intenzionalmente traffico Matrix inter-agent:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` accetta messaggi da altri account bot Matrix configurati in stanze e DM consentiti.
- `allowBots: "mentions"` accetta quei messaggi solo quando menzionano visibilmente questo bot nelle stanze. I DM restano comunque consentiti.
- `groups.<room>.allowBots` sovrascrive l'impostazione a livello account per una singola stanza.
- OpenClaw continua a ignorare i messaggi provenienti dallo stesso ID utente Matrix per evitare cicli di auto-risposta.
- Qui Matrix non espone un flag bot nativo; OpenClaw considera "creato da bot" come "inviato da un altro account Matrix configurato su questo gateway OpenClaw".

Usa allowlist rigorose delle stanze e requisiti di menzione quando abiliti traffico bot-to-bot in stanze condivise.

Abilita la crittografia:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

Controlla lo stato della verifica:

```bash
openclaw matrix verify status
```

Stato dettagliato (diagnostica completa):

```bash
openclaw matrix verify status --verbose
```

Includi la recovery key memorizzata nell'output leggibile dalla macchina:

```bash
openclaw matrix verify status --include-recovery-key --json
```

Inizializza lo stato di cross-signing e verifica:

```bash
openclaw matrix verify bootstrap
```

Supporto multi-account: usa `channels.matrix.accounts` con credenziali per account e `name` facoltativo. Vedi [Configuration reference](/it/gateway/configuration-reference#multi-account-all-channels) per il pattern condiviso.

Diagnostica dettagliata del bootstrap:

```bash
openclaw matrix verify bootstrap --verbose
```

Forza un nuovo reset dell'identità di cross-signing prima del bootstrap:

```bash
openclaw matrix verify bootstrap --force-reset-cross-signing
```

Verifica questo dispositivo con una recovery key:

```bash
openclaw matrix verify device "<your-recovery-key>"
```

Dettagli verbosi della verifica del dispositivo:

```bash
openclaw matrix verify device "<your-recovery-key>" --verbose
```

Controlla lo stato di salute del backup delle chiavi stanza:

```bash
openclaw matrix verify backup status
```

Diagnostica dettagliata dello stato di salute del backup:

```bash
openclaw matrix verify backup status --verbose
```

Ripristina le chiavi stanza dal backup sul server:

```bash
openclaw matrix verify backup restore
```

Diagnostica dettagliata del ripristino:

```bash
openclaw matrix verify backup restore --verbose
```

Elimina il backup server corrente e crea una nuova baseline di backup. Se la chiave di backup
memorizzata non può essere caricata correttamente, questo reset può anche ricreare il secret storage così che
i futuri avvii a freddo possano caricare la nuova chiave di backup:

```bash
openclaw matrix verify backup reset --yes
```

Tutti i comandi `verify` sono concisi per impostazione predefinita (incluso il logging interno SDK silenzioso) e mostrano una diagnostica dettagliata solo con `--verbose`.
Usa `--json` per output completi leggibili dalla macchina negli script.

Nelle configurazioni multi-account, i comandi CLI Matrix usano l'account predefinito Matrix implicito a meno che tu non passi `--account <id>`.
Se configuri più account con nome, imposta prima `channels.matrix.defaultAccount` oppure queste operazioni CLI implicite si fermeranno e ti chiederanno di scegliere esplicitamente un account.
Usa `--account` ogni volta che vuoi che operazioni di verifica o sui dispositivi puntino esplicitamente a un account con nome:

```bash
openclaw matrix verify status --account assistant
openclaw matrix verify backup restore --account assistant
openclaw matrix devices list --account assistant
```

Quando la crittografia è disabilitata o non disponibile per un account con nome, gli avvisi Matrix e gli errori di verifica puntano alla chiave di configurazione di quell'account, per esempio `channels.matrix.accounts.assistant.encryption`.

### Cosa significa "verified"

OpenClaw considera questo dispositivo Matrix verificato solo quando viene verificato dalla tua identità di cross-signing.
In pratica, `openclaw matrix verify status --verbose` espone tre segnali di attendibilità:

- `Locally trusted`: questo dispositivo è attendibile solo dal client corrente
- `Cross-signing verified`: l'SDK riporta il dispositivo come verificato tramite cross-signing
- `Signed by owner`: il dispositivo è firmato dalla tua stessa chiave self-signing

`Verified by owner` diventa `yes` solo quando è presente la verifica cross-signing o la firma del proprietario.
La fiducia locale da sola non basta perché OpenClaw tratti il dispositivo come completamente verificato.

### Cosa fa il bootstrap

`openclaw matrix verify bootstrap` è il comando di riparazione e configurazione per gli account Matrix crittografati.
Esegue tutto quanto segue, in questo ordine:

- inizializza il secret storage, riutilizzando una recovery key esistente quando possibile
- inizializza il cross-signing e carica le chiavi pubbliche di cross-signing mancanti
- tenta di contrassegnare e firmare tramite cross-signing il dispositivo corrente
- crea un nuovo backup lato server delle chiavi stanza se non ne esiste già uno

Se l'homeserver richiede autenticazione interattiva per caricare le chiavi di cross-signing, OpenClaw prova prima il caricamento senza autenticazione, poi con `m.login.dummy`, poi con `m.login.password` quando `channels.matrix.password` è configurato.

Usa `--force-reset-cross-signing` solo quando vuoi intenzionalmente scartare l'identità di cross-signing corrente e crearne una nuova.

Se vuoi intenzionalmente scartare il backup corrente delle chiavi stanza e avviare una nuova
baseline di backup per i messaggi futuri, usa `openclaw matrix verify backup reset --yes`.
Fallo solo se accetti che la cronologia crittografata precedente non recuperabile rimanga
indisponibile e che OpenClaw possa ricreare il secret storage se il secret di backup corrente
non può essere caricato in sicurezza.

### Nuova baseline di backup

Se vuoi mantenere funzionanti i futuri messaggi crittografati e accetti di perdere la vecchia cronologia non recuperabile, esegui questi comandi in ordine:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

Aggiungi `--account <id>` a ogni comando quando vuoi puntare esplicitamente a un account Matrix con nome.

### Comportamento all'avvio

Quando `encryption: true`, Matrix imposta per default `startupVerification` su `"if-unverified"`.
All'avvio, se questo dispositivo non è ancora verificato, Matrix richiederà l'autoverifica in un altro client Matrix,
salterà le richieste duplicate quando una è già in sospeso e applicherà un cooldown locale prima di riprovare dopo i riavvii.
Per impostazione predefinita, i tentativi di richiesta falliti vengono ritentati prima rispetto alla creazione riuscita della richiesta.
Imposta `startupVerification: "off"` per disabilitare le richieste automatiche all'avvio, oppure regola `startupVerificationCooldownHours`
se desideri una finestra di nuovo tentativo più breve o più lunga.

All'avvio viene eseguito automaticamente anche un passaggio conservativo di bootstrap crittografico.
Quel passaggio prova prima a riutilizzare il secret storage e l'identità di cross-signing correnti ed evita di reimpostare il cross-signing a meno che tu non esegua un flusso esplicito di riparazione bootstrap.

Se all'avvio viene rilevato uno stato bootstrap danneggiato e `channels.matrix.password` è configurato, OpenClaw può tentare un percorso di riparazione più rigoroso.
Se il dispositivo corrente è già firmato dal proprietario, OpenClaw preserva quell'identità invece di reimpostarla automaticamente.

Aggiornamento dal precedente plugin Matrix pubblico:

- OpenClaw riutilizza automaticamente lo stesso account Matrix, access token e identità del dispositivo quando possibile.
- Prima che venga eseguita qualsiasi modifica di migrazione Matrix utilizzabile, OpenClaw crea o riutilizza uno snapshot di recupero in `~/Backups/openclaw-migrations/`.
- Se usi più account Matrix, imposta `channels.matrix.defaultAccount` prima di aggiornare dal vecchio layout flat-store così OpenClaw sa quale account deve ricevere quello stato legacy condiviso.
- Se il plugin precedente aveva memorizzato localmente una chiave di decrittazione del backup delle chiavi stanza Matrix, l'avvio o `openclaw doctor --fix` la importeranno automaticamente nel nuovo flusso recovery-key.
- Se l'access token Matrix è cambiato dopo che la migrazione era stata preparata, all'avvio ora viene eseguita la scansione delle root di storage sibling basate sull'hash del token per cercare stato legacy di ripristino in sospeso prima di rinunciare al ripristino automatico del backup.
- Se l'access token Matrix cambia in seguito per lo stesso account, homeserver e utente, OpenClaw ora preferisce riutilizzare la root di storage esistente più completa basata sull'hash del token invece di iniziare da una directory di stato Matrix vuota.
- Al successivo avvio del gateway, le chiavi stanza sottoposte a backup vengono ripristinate automaticamente nel nuovo crypto store.
- Se il vecchio plugin aveva chiavi stanza solo locali che non sono mai state sottoposte a backup, OpenClaw mostrerà un avviso chiaro. Queste chiavi non possono essere esportate automaticamente dal precedente rust crypto store, quindi una parte della vecchia cronologia crittografata potrebbe rimanere non disponibile finché non viene recuperata manualmente.
- Vedi [Matrix migration](/it/install/migrating-matrix) per il flusso completo di aggiornamento, i limiti, i comandi di recupero e i messaggi di migrazione comuni.

Lo stato runtime crittografato è organizzato in root per account, utente e hash del token in
`~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/`.
Quella directory contiene il sync store (`bot-storage.json`), il crypto store (`crypto/`),
il file della recovery key (`recovery-key.json`), lo snapshot IndexedDB (`crypto-idb-snapshot.json`),
i binding dei thread (`thread-bindings.json`) e lo stato di verifica all'avvio (`startup-verification.json`)
quando queste funzionalità sono in uso.
Quando il token cambia ma l'identità dell'account rimane la stessa, OpenClaw riutilizza la migliore root esistente
per quella tupla account/homeserver/utente così lo stato di sincronizzazione precedente, lo stato crittografico, i binding dei thread
e lo stato di verifica all'avvio restano visibili.

### Modello del crypto store Node

L'E2EE Matrix in questo plugin usa il percorso Rust crypto ufficiale di `matrix-js-sdk` in Node.
Quel percorso si aspetta persistenza basata su IndexedDB quando vuoi che lo stato crittografico sopravviva ai riavvii.

Attualmente OpenClaw lo fornisce in Node tramite:

- uso di `fake-indexeddb` come shim API IndexedDB richiesto dall'SDK
- ripristino del contenuto IndexedDB del Rust crypto da `crypto-idb-snapshot.json` prima di `initRustCrypto`
- persistenza del contenuto IndexedDB aggiornato di nuovo in `crypto-idb-snapshot.json` dopo l'inizializzazione e durante il runtime
- serializzazione del ripristino e della persistenza dello snapshot rispetto a `crypto-idb-snapshot.json` con un lock file advisory così la persistenza del runtime del gateway e la manutenzione CLI non entrano in competizione sullo stesso file snapshot

Si tratta di infrastruttura di compatibilità/storage, non di un'implementazione crittografica personalizzata.
Il file snapshot è uno stato runtime sensibile ed è memorizzato con permessi di file restrittivi.
Nel modello di sicurezza di OpenClaw, l'host gateway e la directory di stato locale di OpenClaw sono già all'interno del confine dell'operatore fidato, quindi questa è principalmente una questione di durabilità operativa piuttosto che un confine di fiducia remoto separato.

Miglioramento pianificato:

- aggiungere il supporto SecretRef per il materiale di chiave Matrix persistente così recovery key e relativi secret di crittografia dello store possano provenire dai provider di secret OpenClaw invece che solo da file locali

## Gestione del profilo

Aggiorna l'auto-profilo Matrix per l'account selezionato con:

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Aggiungi `--account <id>` quando vuoi puntare esplicitamente a un account Matrix con nome.

Matrix accetta direttamente URL avatar `mxc://`. Quando passi un URL avatar `http://` o `https://`, OpenClaw lo carica prima su Matrix e memorizza l'URL `mxc://` risolto in `channels.matrix.avatarUrl` (o nell'override dell'account selezionato).

## Avvisi automatici di verifica

Matrix ora pubblica gli avvisi del ciclo di vita della verifica direttamente nella stanza DM stretta di verifica come messaggi `m.notice`.
Questo include:

- avvisi di richiesta di verifica
- avvisi di verifica pronta (con istruzioni esplicite "Verifica tramite emoji")
- avvisi di inizio e completamento della verifica
- dettagli SAS (emoji e decimali) quando disponibili

Le richieste di verifica in arrivo da un altro client Matrix vengono tracciate e accettate automaticamente da OpenClaw.
Per i flussi di autoverifica, OpenClaw avvia automaticamente anche il flusso SAS quando la verifica tramite emoji diventa disponibile e conferma il proprio lato.
Per le richieste di verifica da un altro utente/dispositivo Matrix, OpenClaw accetta automaticamente la richiesta e poi aspetta che il flusso SAS proceda normalmente.
Devi comunque confrontare l'emoji o il SAS decimale nel tuo client Matrix e confermare lì "Corrispondono" per completare la verifica.

OpenClaw non accetta automaticamente alla cieca i flussi duplicati auto-avviati. All'avvio viene evitata la creazione di una nuova richiesta quando è già presente una richiesta di autoverifica in sospeso.

Gli avvisi di protocollo/sistema della verifica non vengono inoltrati alla pipeline di chat dell'agente, quindi non producono `NO_REPLY`.

### Igiene dei dispositivi

I vecchi dispositivi Matrix gestiti da OpenClaw possono accumularsi sull'account e rendere più difficile interpretare l'attendibilità delle stanze crittografate.
Elencali con:

```bash
openclaw matrix devices list
```

Rimuovi i dispositivi obsoleti gestiti da OpenClaw con:

```bash
openclaw matrix devices prune-stale
```

### Riparazione diretta della stanza

Se lo stato dei messaggi diretti va fuori sincronia, OpenClaw può ritrovarsi con mapping `m.direct` obsoleti che puntano a vecchie stanze singole invece che al DM attivo. Esamina il mapping corrente per un peer con:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Riparalo con:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

La riparazione mantiene nel plugin la logica specifica di Matrix:

- preferisce un DM stretto 1:1 che è già mappato in `m.direct`
- altrimenti ricorre a qualsiasi DM stretto 1:1 attualmente unito con quell'utente
- se non esiste alcun DM integro, crea una nuova stanza diretta e riscrive `m.direct` perché punti a essa

Il flusso di riparazione non elimina automaticamente le vecchie stanze. Seleziona solo il DM integro e aggiorna il mapping così nuovi invii Matrix, avvisi di verifica e altri flussi di messaggi diretti puntino di nuovo alla stanza corretta.

## Thread

Matrix supporta i thread Matrix nativi sia per le risposte automatiche sia per gli invii con strumenti di messaggistica.

- `dm.sessionScope: "per-user"` (predefinito) mantiene l'instradamento dei DM Matrix con ambito mittente, quindi più stanze DM possono condividere una sessione quando vengono risolte sullo stesso peer.
- `dm.sessionScope: "per-room"` isola ogni stanza DM Matrix nella propria chiave di sessione pur continuando a usare i normali controlli di autenticazione DM e allowlist.
- I binding espliciti di conversazione Matrix hanno comunque la precedenza su `dm.sessionScope`, quindi stanze e thread associati mantengono la sessione di destinazione scelta.
- `threadReplies: "off"` mantiene le risposte a livello superiore e mantiene i messaggi in ingresso nei thread sulla sessione padre.
- `threadReplies: "inbound"` risponde dentro un thread solo quando il messaggio in ingresso era già in quel thread.
- `threadReplies: "always"` mantiene le risposte delle stanze in un thread con radice nel messaggio attivante e instrada quella conversazione attraverso la sessione con ambito thread corrispondente a partire dal primo messaggio attivante.
- `dm.threadReplies` sovrascrive l'impostazione di livello superiore solo per i DM. Per esempio, puoi mantenere isolati i thread delle stanze mantenendo piatti i DM.
- I messaggi in ingresso nei thread includono il messaggio radice del thread come contesto agente aggiuntivo.
- Gli invii con strumenti di messaggistica ora ereditano automaticamente il thread Matrix corrente quando la destinazione è la stessa stanza, o lo stesso utente DM di destinazione, a meno che non venga fornito un `threadId` esplicito.
- Il riutilizzo della stessa sessione con destinazione utente DM si attiva solo quando i metadati della sessione corrente dimostrano lo stesso peer DM sullo stesso account Matrix; altrimenti OpenClaw torna al normale instradamento con ambito utente.
- Quando OpenClaw rileva che una stanza DM Matrix entra in collisione con un'altra stanza DM sulla stessa sessione DM Matrix condivisa, pubblica un `m.notice` una tantum in quella stanza con la via di fuga `/focus` quando i binding dei thread sono abilitati e con l'indicazione `dm.sessionScope`.
- I binding runtime dei thread sono supportati per Matrix. `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` e `/acp spawn` associato a thread ora funzionano in stanze e DM Matrix.
- `/focus` a livello superiore in una stanza/DM Matrix crea un nuovo thread Matrix e lo associa alla sessione di destinazione quando `threadBindings.spawnSubagentSessions=true`.
- Eseguire `/focus` o `/acp spawn --thread here` all'interno di un thread Matrix esistente associa invece quel thread corrente.

## Binding di conversazione ACP

Stanze, DM e thread Matrix esistenti possono essere trasformati in workspace ACP persistenti senza cambiare la superficie di chat.

Flusso rapido per operatori:

- Esegui `/acp spawn codex --bind here` nel DM Matrix, nella stanza o nel thread esistente che vuoi continuare a usare.
- In un DM o stanza Matrix di primo livello, il DM/stanza corrente rimane la superficie di chat e i messaggi futuri vengono instradati alla sessione ACP generata.
- All'interno di un thread Matrix esistente, `--bind here` associa sul posto quel thread corrente.
- `/new` e `/reset` reimpostano sul posto la stessa sessione ACP associata.
- `/acp close` chiude la sessione ACP e rimuove il binding.

Note:

- `--bind here` non crea un thread Matrix figlio.
- `threadBindings.spawnAcpSessions` è richiesto solo per `/acp spawn --thread auto|here`, quando OpenClaw deve creare o associare un thread Matrix figlio.

### Configurazione dei binding dei thread

Matrix eredita i valori predefiniti globali da `session.threadBindings` e supporta anche override per canale:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSubagentSessions`
- `threadBindings.spawnAcpSessions`

I flag di generazione associata a thread Matrix sono opt-in:

- Imposta `threadBindings.spawnSubagentSessions: true` per consentire a `/focus` di primo livello di creare e associare nuovi thread Matrix.
- Imposta `threadBindings.spawnAcpSessions: true` per consentire a `/acp spawn --thread auto|here` di associare sessioni ACP ai thread Matrix.

## Reazioni

Matrix supporta azioni di reazione in uscita, notifiche di reazione in ingresso e reazioni di ack in ingresso.

- Gli strumenti di reazione in uscita sono controllati da `channels["matrix"].actions.reactions`.
- `react` aggiunge una reazione a uno specifico evento Matrix.
- `reactions` elenca il riepilogo corrente delle reazioni per uno specifico evento Matrix.
- `emoji=""` rimuove le reazioni dell'account bot su quell'evento.
- `remove: true` rimuove solo la reazione con l'emoji specificata dall'account bot.

L'ambito delle reazioni di ack viene risolto nell'ordine standard di OpenClaw:

- `channels["matrix"].accounts.<accountId>.ackReaction`
- `channels["matrix"].ackReaction`
- `messages.ackReaction`
- fallback all'emoji di identità dell'agente

L'ambito della reazione di ack viene risolto in questo ordine:

- `channels["matrix"].accounts.<accountId>.ackReactionScope`
- `channels["matrix"].ackReactionScope`
- `messages.ackReactionScope`

La modalità di notifica delle reazioni viene risolta in questo ordine:

- `channels["matrix"].accounts.<accountId>.reactionNotifications`
- `channels["matrix"].reactionNotifications`
- predefinito: `own`

Comportamento attuale:

- `reactionNotifications: "own"` inoltra gli eventi `m.reaction` aggiunti quando puntano a messaggi Matrix creati dal bot.
- `reactionNotifications: "off"` disabilita gli eventi di sistema delle reazioni.
- Le rimozioni delle reazioni non vengono ancora sintetizzate in eventi di sistema perché Matrix le espone come redazioni, non come rimozioni autonome di `m.reaction`.

## Contesto della cronologia

- `channels.matrix.historyLimit` controlla quanti messaggi recenti della stanza sono inclusi come `InboundHistory` quando un messaggio di una stanza Matrix attiva l'agente.
- Come fallback usa `messages.groupChat.historyLimit`. Imposta `0` per disabilitare.
- La cronologia delle stanze Matrix è solo di stanza. I DM continuano a usare la normale cronologia della sessione.
- La cronologia delle stanze Matrix è solo pending: OpenClaw mette in buffer i messaggi della stanza che non hanno ancora attivato una risposta, poi cattura uno snapshot di quella finestra quando arriva una menzione o un altro trigger.
- Il messaggio trigger corrente non è incluso in `InboundHistory`; rimane nel corpo principale in ingresso per quel turno.
- I retry dello stesso evento Matrix riutilizzano lo snapshot originale della cronologia invece di avanzare verso messaggi della stanza più recenti.

## Visibilità del contesto

Matrix supporta il controllo condiviso `contextVisibility` per il contesto supplementare della stanza come testo di risposta recuperato, radici dei thread e cronologia pending.

- `contextVisibility: "all"` è il valore predefinito. Il contesto supplementare viene mantenuto così come ricevuto.
- `contextVisibility: "allowlist"` filtra il contesto supplementare ai mittenti consentiti dai controlli attivi di allowlist della stanza/utente.
- `contextVisibility: "allowlist_quote"` si comporta come `allowlist`, ma mantiene comunque una risposta citata esplicita.

Questa impostazione influisce sulla visibilità del contesto supplementare, non sulla possibilità che il messaggio in ingresso stesso attivi una risposta.
L'autorizzazione del trigger continua a derivare da `groupPolicy`, `groups`, `groupAllowFrom` e dalle impostazioni di policy DM.

## Esempio di policy per DM e stanze

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

Vedi [Groups](/it/channels/groups) per il comportamento di gating delle menzioni e della allowlist.

Esempio di pairing per DM Matrix:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Se un utente Matrix non approvato continua a scriverti prima dell'approvazione, OpenClaw riutilizza lo stesso codice di pairing in sospeso e può inviare di nuovo una risposta di promemoria dopo un breve cooldown invece di generare un nuovo codice.

Vedi [Pairing](/it/channels/pairing) per il flusso condiviso di pairing DM e il layout di storage.

## Approvazioni exec

Matrix può agire come client di approvazione exec per un account Matrix.

- `channels.matrix.execApprovals.enabled`
- `channels.matrix.execApprovals.approvers` (facoltativo; come fallback usa `channels.matrix.dm.allowFrom`)
- `channels.matrix.execApprovals.target` (`dm` | `channel` | `both`, predefinito: `dm`)
- `channels.matrix.execApprovals.agentFilter`
- `channels.matrix.execApprovals.sessionFilter`

Gli approvatori devono essere ID utente Matrix come `@owner:example.org`. Matrix abilita automaticamente le approvazioni exec native quando `enabled` non è impostato oppure è `"auto"` e può essere risolto almeno un approvatore, da `execApprovals.approvers` oppure da `channels.matrix.dm.allowFrom`. Imposta `enabled: false` per disabilitare esplicitamente Matrix come client di approvazione nativo. In caso contrario, le richieste di approvazione ricadono su altre route di approvazione configurate o sulla policy di fallback di approvazione exec.

L'instradamento nativo Matrix oggi è solo per exec:

- `channels.matrix.execApprovals.*` controlla l'instradamento nativo DM/canale solo per le approvazioni exec.
- Le approvazioni dei plugin continuano a usare `/approve` condiviso nella stessa chat più qualsiasi inoltro configurato in `approvals.plugin`.
- Matrix può comunque riutilizzare `channels.matrix.dm.allowFrom` per l'autorizzazione delle approvazioni plugin quando può dedurre in sicurezza gli approvatori, ma non espone un percorso separato nativo DM/canale per il fanout delle approvazioni plugin.

Regole di consegna:

- `target: "dm"` invia i prompt di approvazione ai DM degli approvatori
- `target: "channel"` invia il prompt alla stanza Matrix o al DM di origine
- `target: "both"` invia ai DM degli approvatori e alla stanza Matrix o al DM di origine

I prompt di approvazione Matrix inizializzano scorciatoie tramite reazione sul messaggio di approvazione principale:

- `✅` = consenti una volta
- `❌` = nega
- `♾️` = consenti sempre quando quella decisione è permessa dalla policy exec effettiva

Gli approvatori possono reagire su quel messaggio oppure usare i comandi slash di fallback: `/approve <id> allow-once`, `/approve <id> allow-always` oppure `/approve <id> deny`.

Solo gli approvatori risolti possono approvare o negare. La consegna sul canale include il testo del comando, quindi abilita `channel` o `both` solo in stanze fidate.

I prompt di approvazione Matrix riutilizzano il planner condiviso di approvazione core. La superficie nativa specifica di Matrix è solo di trasporto per le approvazioni exec: instradamento stanza/DM e comportamento di invio/aggiornamento/eliminazione dei messaggi.

Override per account:

- `channels.matrix.accounts.<account>.execApprovals`

Documentazione correlata: [Exec approvals](/it/tools/exec-approvals)

## Esempio multi-account

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

I valori di primo livello `channels.matrix` agiscono come predefiniti per gli account con nome a meno che un account non li sovrascriva.
Puoi assegnare una voce stanza ereditata a un solo account Matrix con `groups.<room>.account` (o il legacy `rooms.<room>.account`).
Le voci senza `account` rimangono condivise tra tutti gli account Matrix, e le voci con `account: "default"` continuano a funzionare quando l'account predefinito è configurato direttamente in `channels.matrix.*` di primo livello.
I predefiniti di autenticazione condivisa parziale non creano da soli un account predefinito implicito separato. OpenClaw sintetizza l'account `default` di primo livello solo quando quel predefinito ha autenticazione fresca (`homeserver` più `accessToken`, oppure `homeserver` più `userId` e `password`); gli account con nome possono comunque rimanere individuabili da `homeserver` più `userId` quando le credenziali nella cache soddisfano in seguito l'autenticazione.
Se Matrix ha già esattamente un account con nome, oppure `defaultAccount` punta a una chiave account con nome esistente, la promozione di riparazione/configurazione da account singolo a multi-account preserva quell'account invece di creare una nuova voce `accounts.default`. Solo le chiavi di autenticazione/bootstrap Matrix vengono spostate in quell'account promosso; le chiavi condivise di policy di consegna restano al livello superiore.
Imposta `defaultAccount` quando vuoi che OpenClaw preferisca un account Matrix con nome per instradamento implicito, probing e operazioni CLI.
Se configuri più account con nome, imposta `defaultAccount` oppure passa `--account <id>` per i comandi CLI che dipendono dalla selezione implicita dell'account.
Passa `--account <id>` a `openclaw matrix verify ...` e `openclaw matrix devices ...` quando vuoi sovrascrivere quella selezione implicita per un comando.

## Homeserver privati/LAN

Per impostazione predefinita, OpenClaw blocca gli homeserver Matrix privati/interni per la protezione SSRF a meno che tu
non scelga esplicitamente l'opt-in per account.

Se il tuo homeserver è in esecuzione su localhost, su un IP LAN/Tailscale o su un hostname interno, abilita
`allowPrivateNetwork` per quell'account Matrix:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      allowPrivateNetwork: true,
      accessToken: "syt_internal_xxx",
    },
  },
}
```

Esempio di configurazione CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Questo opt-in consente solo target privati/interni fidati. Gli homeserver pubblici in chiaro come
`http://matrix.example.org:8008` restano bloccati. Preferisci `https://` quando possibile.

## Instradamento del traffico Matrix tramite proxy

Se il tuo deployment Matrix richiede un proxy HTTP(S) in uscita esplicito, imposta `channels.matrix.proxy`:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Gli account con nome possono sovrascrivere il valore predefinito di primo livello con `channels.matrix.accounts.<id>.proxy`.
OpenClaw usa la stessa impostazione proxy per il traffico Matrix a runtime e per i probe dello stato dell'account.

## Risoluzione del target

Matrix accetta queste forme di target ovunque OpenClaw ti chieda un target stanza o utente:

- Utenti: `@user:server`, `user:@user:server` o `matrix:user:@user:server`
- Stanze: `!room:server`, `room:!room:server` o `matrix:room:!room:server`
- Alias: `#alias:server`, `channel:#alias:server` o `matrix:channel:#alias:server`

La ricerca live nella directory usa l'account Matrix autenticato:

- Le ricerche utente interrogano la directory utenti Matrix su quell'homeserver.
- Le ricerche stanza accettano direttamente ID stanza e alias espliciti, poi ripiegano sulla ricerca dei nomi delle stanze a cui quell'account ha aderito.
- La ricerca per nome nelle stanze aderite è best-effort. Se un nome stanza non può essere risolto in un ID o alias, viene ignorato dalla risoluzione della allowlist a runtime.

## Riferimento della configurazione

- `enabled`: abilita o disabilita il canale.
- `name`: etichetta facoltativa per l'account.
- `defaultAccount`: ID account preferito quando sono configurati più account Matrix.
- `homeserver`: URL dell'homeserver, per esempio `https://matrix.example.org`.
- `allowPrivateNetwork`: consente a questo account Matrix di connettersi a homeserver privati/interni. Abilitalo quando l'homeserver viene risolto in `localhost`, un IP LAN/Tailscale o un host interno come `matrix-synapse`.
- `proxy`: URL facoltativo del proxy HTTP(S) per il traffico Matrix. Gli account con nome possono sovrascrivere il valore predefinito di primo livello con il proprio `proxy`.
- `userId`: ID utente Matrix completo, per esempio `@bot:example.org`.
- `accessToken`: access token per autenticazione basata su token. Sono supportati valori in chiaro e valori SecretRef per `channels.matrix.accessToken` e `channels.matrix.accounts.<id>.accessToken` tramite provider env/file/exec. Vedi [Secrets Management](/it/gateway/secrets).
- `password`: password per il login basato su password. Sono supportati valori in chiaro e valori SecretRef.
- `deviceId`: ID dispositivo Matrix esplicito.
- `deviceName`: nome visualizzato del dispositivo per il login con password.
- `avatarUrl`: URL dell'avatar del profilo memorizzato per la sincronizzazione del profilo e gli aggiornamenti `set-profile`.
- `initialSyncLimit`: limite degli eventi di sincronizzazione all'avvio.
- `encryption`: abilita E2EE.
- `allowlistOnly`: forza comportamento solo allowlist per DM e stanze.
- `allowBots`: consente messaggi da altri account Matrix OpenClaw configurati (`true` o `"mentions"`).
- `groupPolicy`: `open`, `allowlist` o `disabled`.
- `contextVisibility`: modalità di visibilità del contesto supplementare della stanza (`all`, `allowlist`, `allowlist_quote`).
- `groupAllowFrom`: allowlist di ID utente per il traffico delle stanze.
- Le voci `groupAllowFrom` devono essere ID utente Matrix completi. I nomi non risolti vengono ignorati a runtime.
- `historyLimit`: numero massimo di messaggi della stanza da includere come contesto della cronologia di gruppo. Come fallback usa `messages.groupChat.historyLimit`. Imposta `0` per disabilitare.
- `replyToMode`: `off`, `first` o `all`.
- `markdown`: configurazione facoltativa del rendering Markdown per il testo Matrix in uscita.
- `streaming`: `off` (predefinito), `partial`, `quiet`, `true` o `false`. `partial` e `true` abilitano aggiornamenti di bozza con anteprima iniziale usando normali messaggi di testo Matrix. `quiet` usa avvisi di anteprima senza notifica per configurazioni push-rule self-hosted.
- `blockStreaming`: `true` abilita messaggi di avanzamento separati per i blocchi completati dell'assistente mentre è attivo lo streaming della bozza di anteprima.
- `threadReplies`: `off`, `inbound` o `always`.
- `threadBindings`: override per canale per instradamento e ciclo di vita delle sessioni associate ai thread.
- `startupVerification`: modalità automatica di richiesta di autoverifica all'avvio (`if-unverified`, `off`).
- `startupVerificationCooldownHours`: cooldown prima di ritentare richieste automatiche di verifica all'avvio.
- `textChunkLimit`: dimensione dei chunk dei messaggi in uscita.
- `chunkMode`: `length` o `newline`.
- `responsePrefix`: prefisso facoltativo del messaggio per le risposte in uscita.
- `ackReaction`: override facoltativo della reazione di ack per questo canale/account.
- `ackReactionScope`: override facoltativo dell'ambito della reazione di ack (`group-mentions`, `group-all`, `direct`, `all`, `none`, `off`).
- `reactionNotifications`: modalità di notifica delle reazioni in ingresso (`own`, `off`).
- `mediaMaxMb`: limite di dimensione dei contenuti multimediali in MB per la gestione Matrix. Si applica agli invii in uscita e all'elaborazione dei contenuti multimediali in ingresso.
- `autoJoin`: policy di auto-join agli inviti (`always`, `allowlist`, `off`). Predefinito: `off`.
- `autoJoinAllowlist`: stanze/alias consentiti quando `autoJoin` è `allowlist`. Le voci alias vengono risolte in ID stanza durante la gestione dell'invito; OpenClaw non si fida dello stato alias dichiarato dalla stanza invitante.
- `dm`: blocco di policy DM (`enabled`, `policy`, `allowFrom`, `sessionScope`, `threadReplies`).
- Le voci `dm.allowFrom` devono essere ID utente Matrix completi a meno che tu non le abbia già risolte tramite ricerca live nella directory.
- `dm.sessionScope`: `per-user` (predefinito) o `per-room`. Usa `per-room` quando vuoi che ogni stanza DM Matrix mantenga un contesto separato anche se il peer è lo stesso.
- `dm.threadReplies`: override della policy dei thread solo per DM (`off`, `inbound`, `always`). Sovrascrive l'impostazione `threadReplies` di primo livello sia per il posizionamento delle risposte sia per l'isolamento delle sessioni nei DM.
- `execApprovals`: consegna nativa delle approvazioni exec Matrix (`enabled`, `approvers`, `target`, `agentFilter`, `sessionFilter`).
- `execApprovals.approvers`: ID utente Matrix autorizzati ad approvare richieste exec. Facoltativo quando `dm.allowFrom` identifica già gli approvatori.
- `execApprovals.target`: `dm | channel | both` (predefinito: `dm`).
- `accounts`: override con nome per account. I valori di primo livello `channels.matrix` agiscono come predefiniti per queste voci.
- `groups`: mappa di policy per stanza. Preferisci ID stanza o alias; i nomi stanza non risolti vengono ignorati a runtime. L'identità di sessione/gruppo usa l'ID stanza stabile dopo la risoluzione, mentre le etichette leggibili restano basate sui nomi stanza.
- `groups.<room>.account`: limita una voce stanza ereditata a uno specifico account Matrix nelle configurazioni multi-account.
- `groups.<room>.allowBots`: override a livello stanza per mittenti bot configurati (`true` o `"mentions"`).
- `groups.<room>.users`: allowlist dei mittenti per stanza.
- `groups.<room>.tools`: override per stanza di consenti/nega per gli strumenti.
- `groups.<room>.autoReply`: override a livello stanza del gating delle menzioni. `true` disabilita i requisiti di menzione per quella stanza; `false` li forza nuovamente.
- `groups.<room>.skills`: filtro facoltativo delle Skills a livello stanza.
- `groups.<room>.systemPrompt`: snippet facoltativo di system prompt a livello stanza.
- `rooms`: alias legacy per `groups`.
- `actions`: gating per strumento per azione (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).

## Correlati

- [Channels Overview](/it/channels) — tutti i canali supportati
- [Pairing](/it/channels/pairing) — autenticazione DM e flusso di pairing
- [Groups](/it/channels/groups) — comportamento della chat di gruppo e gating delle menzioni
- [Channel Routing](/it/channels/channel-routing) — instradamento della sessione per i messaggi
- [Security](/it/gateway/security) — modello di accesso e hardening
