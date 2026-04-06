---
read_when:
    - Uso o configurazione dei comandi chat
    - Debug dell'instradamento dei comandi o dei permessi
summary: 'Comandi slash: testo vs nativi, configurazione e comandi supportati'
title: Comandi slash
x-i18n:
    generated_at: "2026-04-06T03:13:56Z"
    model: gpt-5.4
    provider: openai
    source_hash: 417e35b9ddd87f25f6c019111b55b741046ea11039dde89210948185ced5696d
    source_path: tools/slash-commands.md
    workflow: 15
---

# Comandi slash

I comandi sono gestiti dal Gateway. La maggior parte dei comandi deve essere inviata come messaggio **autonomo** che inizia con `/`.
Il comando chat bash solo host usa `! <cmd>` (con `/bash <cmd>` come alias).

Esistono due sistemi correlati:

- **Comandi**: messaggi autonomi `/...`.
- **Direttive**: `/think`, `/fast`, `/verbose`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`.
  - Le direttive vengono rimosse dal messaggio prima che il modello lo veda.
  - Nei normali messaggi di chat (non composti solo da direttive), vengono trattate come “suggerimenti inline” e **non** rendono persistenti le impostazioni della sessione.
  - Nei messaggi composti solo da direttive (il messaggio contiene solo direttive), diventano persistenti per la sessione e rispondono con una conferma.
  - Le direttive vengono applicate solo per i **mittenti autorizzati**. Se `commands.allowFrom` è impostato, è l'unica
    allowlist usata; altrimenti l'autorizzazione deriva dalle allowlist/pairing del canale più `commands.useAccessGroups`.
    I mittenti non autorizzati vedono le direttive trattate come testo normale.

Esistono anche alcune **scorciatoie inline** (solo per mittenti in allowlist/autorizzati): `/help`, `/commands`, `/status`, `/whoami` (`/id`).
Vengono eseguite immediatamente, vengono rimosse prima che il modello veda il messaggio e il testo rimanente continua nel flusso normale.

## Configurazione

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: false,
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text` (predefinito `true`) abilita il parsing di `/...` nei messaggi chat.
  - Sulle superfici senza comandi nativi (WhatsApp/WebChat/Signal/iMessage/Google Chat/Microsoft Teams), i comandi testuali continuano a funzionare anche se imposti questo valore su `false`.
- `commands.native` (predefinito `"auto"`) registra i comandi nativi.
  - Auto: attivo per Discord/Telegram; disattivo per Slack (finché non aggiungi slash command); ignorato per i provider senza supporto nativo.
  - Imposta `channels.discord.commands.native`, `channels.telegram.commands.native` o `channels.slack.commands.native` per fare override per provider (bool o `"auto"`).
  - `false` cancella i comandi registrati in precedenza su Discord/Telegram all'avvio. I comandi Slack sono gestiti nell'app Slack e non vengono rimossi automaticamente.
- `commands.nativeSkills` (predefinito `"auto"`) registra i comandi **Skill** in modo nativo quando supportato.
  - Auto: attivo per Discord/Telegram; disattivo per Slack (Slack richiede la creazione di uno slash command per ogni Skill).
  - Imposta `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills` o `channels.slack.commands.nativeSkills` per fare override per provider (bool o `"auto"`).
- `commands.bash` (predefinito `false`) abilita `! <cmd>` per eseguire comandi shell host (`/bash <cmd>` è un alias; richiede allowlist `tools.elevated`).
- `commands.bashForegroundMs` (predefinito `2000`) controlla per quanto tempo bash attende prima di passare alla modalità background (`0` manda subito in background).
- `commands.config` (predefinito `false`) abilita `/config` (legge/scrive `openclaw.json`).
- `commands.mcp` (predefinito `false`) abilita `/mcp` (legge/scrive la configurazione MCP gestita da OpenClaw sotto `mcp.servers`).
- `commands.plugins` (predefinito `false`) abilita `/plugins` (discovery/stato dei plugin più controlli di installazione + abilitazione/disabilitazione).
- `commands.debug` (predefinito `false`) abilita `/debug` (override solo runtime).
- `commands.allowFrom` (opzionale) imposta una allowlist per provider per l'autorizzazione ai comandi. Quando è configurata, è l'unica origine di autorizzazione per comandi e direttive (le allowlist/pairing del canale e `commands.useAccessGroups`
  vengono ignorati). Usa `"*"` come valore predefinito globale; le chiavi specifiche del provider lo sovrascrivono.
- `commands.useAccessGroups` (predefinito `true`) applica allowlist/policy per i comandi quando `commands.allowFrom` non è impostato.

## Elenco dei comandi

Testo + nativi (quando abilitati):

- `/help`
- `/commands`
- `/tools [compact|verbose]` (mostra cosa l'agent corrente può usare in questo momento; `verbose` aggiunge descrizioni)
- `/skill <name> [input]` (esegue una Skill per nome)
- `/status` (mostra lo stato corrente; include utilizzo/quota del provider per il provider del modello corrente quando disponibile)
- `/tasks` (elenca le attività in background per la sessione corrente; mostra dettagli delle attività attive e recenti con conteggi di fallback locali all'agent)
- `/allowlist` (elenca/aggiunge/rimuove voci di allowlist)
- `/approve <id> <decision>` (risolve i prompt di approvazione exec; usa il messaggio di approvazione in sospeso per le decisioni disponibili)
- `/context [list|detail|json]` (spiega il “contesto”; `detail` mostra dimensioni per file + per strumento + per Skill + prompt di sistema)
- `/btw <question>` (fa una domanda laterale effimera sulla sessione corrente senza modificare il contesto futuro della sessione; consulta [/tools/btw](/it/tools/btw))
- `/export-session [path]` (alias: `/export`) (esporta la sessione corrente in HTML con il prompt di sistema completo)
- `/whoami` (mostra il tuo sender id; alias: `/id`)
- `/session idle <duration|off>` (gestisce il disallineamento automatico per inattività per i binding dei thread focalizzati)
- `/session max-age <duration|off>` (gestisce il disallineamento automatico hard max-age per i binding dei thread focalizzati)
- `/subagents list|kill|log|info|send|steer|spawn` (ispeziona, controlla o avvia esecuzioni di sub-agent per la sessione corrente)
- `/acp spawn|cancel|steer|close|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|sessions` (ispeziona e controlla le sessioni runtime ACP)
- `/agents` (elenca gli agent associati ai thread per questa sessione)
- `/focus <target>` (Discord: associa questo thread, o un nuovo thread, a una destinazione session/subagent)
- `/unfocus` (Discord: rimuove l'associazione corrente del thread)
- `/kill <id|#|all>` (interrompe immediatamente uno o tutti i sub-agent in esecuzione per questa sessione; nessun messaggio di conferma)
- `/steer <id|#> <message>` (indirizza immediatamente un sub-agent in esecuzione: durante l'esecuzione quando possibile, altrimenti interrompe il lavoro corrente e riavvia sul messaggio di indirizzamento)
- `/tell <id|#> <message>` (alias per `/steer`)
- `/config show|get|set|unset` (rende persistente la configurazione su disco, solo proprietario; richiede `commands.config: true`)
- `/mcp show|get|set|unset` (gestisce la configurazione del server MCP OpenClaw, solo proprietario; richiede `commands.mcp: true`)
- `/plugins list|show|get|install|enable|disable` (ispeziona i plugin rilevati, installa nuovi plugin e attiva/disattiva l'abilitazione; solo proprietario per le scritture; richiede `commands.plugins: true`)
  - `/plugin` è un alias per `/plugins`.
  - `/plugin install <spec>` accetta le stesse specifiche di plugin di `openclaw plugins install`: percorso locale/archivio, pacchetto npm o `clawhub:<pkg>`.
  - Le scritture di abilitazione/disabilitazione rispondono comunque con un suggerimento di riavvio. Su un gateway foreground monitorato, OpenClaw può eseguire automaticamente quel riavvio subito dopo la scrittura.
- `/debug show|set|unset|reset` (override runtime, solo proprietario; richiede `commands.debug: true`)
- `/usage off|tokens|full|cost` (footer di utilizzo per risposta o riepilogo locale dei costi)
- `/tts off|always|inbound|tagged|status|provider|limit|summary|audio` (controlla TTS; consulta [/tts](/it/tools/tts))
  - Discord: il comando nativo è `/voice` (Discord riserva `/tts`); il testo `/tts` continua a funzionare.
- `/stop`
- `/restart`
- `/dock-telegram` (alias: `/dock_telegram`) (sposta le risposte su Telegram)
- `/dock-discord` (alias: `/dock_discord`) (sposta le risposte su Discord)
- `/dock-slack` (alias: `/dock_slack`) (sposta le risposte su Slack)
- `/activation mention|always` (solo gruppi)
- `/send on|off|inherit` (solo proprietario)
- `/reset` o `/new [model]` (suggerimento modello opzionale; il resto viene inoltrato)
- `/think <off|minimal|low|medium|high|xhigh>` (scelte dinamiche per modello/provider; alias: `/thinking`, `/t`)
- `/fast status|on|off` (omettere l'argomento mostra lo stato effettivo corrente della modalità fast)
- `/verbose on|full|off` (alias: `/v`)
- `/reasoning on|off|stream` (alias: `/reason`; quando è attivo, invia un messaggio separato prefissato `Reasoning:`; `stream` = solo bozza Telegram)
- `/elevated on|off|ask|full` (alias: `/elev`; `full` salta le approvazioni exec)
- `/exec host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` (invia `/exec` per mostrare il valore corrente)
- `/model <name>` (alias: `/models`; oppure `/<alias>` da `agents.defaults.models.*.alias`)
- `/queue <mode>` (più opzioni come `debounce:2s cap:25 drop:summarize`; invia `/queue` per vedere le impostazioni correnti)
- `/bash <command>` (solo host; alias per `! <command>`; richiede `commands.bash: true` + allowlist `tools.elevated`)
- `/dreaming [on|off|status|help]` (attiva/disattiva dreaming globale o mostra lo stato; consulta [Dreaming](/concepts/dreaming))

Solo testo:

- `/compact [instructions]` (consulta [/concepts/compaction](/it/concepts/compaction))
- `! <command>` (solo host; uno alla volta; usa `!poll` + `!stop` per job di lunga durata)
- `!poll` (controlla output / stato; accetta `sessionId` opzionale; funziona anche `/bash poll`)
- `!stop` (interrompe il job bash in esecuzione; accetta `sessionId` opzionale; funziona anche `/bash stop`)

Note:

- I comandi accettano un `:` opzionale tra comando e argomenti (ad esempio `/think: high`, `/send: on`, `/help:`).
- `/new <model>` accetta un alias modello, `provider/model` o un nome provider (corrispondenza fuzzy); se non c'è corrispondenza, il testo viene trattato come corpo del messaggio.
- Per il dettaglio completo dell'utilizzo per provider, usa `openclaw status --usage`.
- `/allowlist add|remove` richiede `commands.config=true` e rispetta `configWrites` del canale.
- Nei canali multi-account, anche `/allowlist --account <id>` con target di configurazione e `/config set channels.<provider>.accounts.<id>...` rispettano `configWrites` dell'account di destinazione.
- `/usage` controlla il footer di utilizzo per risposta; `/usage cost` stampa un riepilogo locale dei costi dai log di sessione OpenClaw.
- `/restart` è abilitato per impostazione predefinita; imposta `commands.restart: false` per disabilitarlo.
- Comando nativo solo Discord: `/vc join|leave|status` controlla i canali vocali (richiede `channels.discord.voice` e comandi nativi; non disponibile come testo).
- I comandi di binding dei thread Discord (`/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age`) richiedono che i binding effettivi dei thread siano abilitati (`session.threadBindings.enabled` e/o `channels.discord.threadBindings.enabled`).
- Riferimento dei comandi ACP e comportamento runtime: [ACP Agents](/it/tools/acp-agents).
- `/verbose` è pensato per il debug e per una visibilità aggiuntiva; mantienilo **disattivato** nell'uso normale.
- `/fast on|off` rende persistente un override di sessione. Usa l'opzione `inherit` nell'interfaccia Sessions per cancellarlo e tornare ai valori predefiniti della configurazione.
- `/fast` è specifico del provider: OpenAI/OpenAI Codex lo mappano a `service_tier=priority` sugli endpoint nativi Responses, mentre le richieste Anthropic pubbliche dirette, incluso il traffico autenticato OAuth inviato a `api.anthropic.com`, lo mappano a `service_tier=auto` o `standard_only`. Consulta [OpenAI](/it/providers/openai) e [Anthropic](/it/providers/anthropic).
- I riepiloghi dei guasti degli strumenti vengono comunque mostrati quando rilevanti, ma il testo dettagliato del guasto viene incluso solo quando `/verbose` è `on` o `full`.
- `/reasoning` (e `/verbose`) sono rischiosi in contesti di gruppo: possono rivelare reasoning interno o output degli strumenti che non intendevi esporre. È preferibile lasciarli disattivati, soprattutto nelle chat di gruppo.
- `/model` rende persistente immediatamente il nuovo modello di sessione.
- Se l'agent è inattivo, l'esecuzione successiva lo usa subito.
- Se è già attiva un'esecuzione, OpenClaw contrassegna un cambio live come in sospeso e riavvia nel nuovo modello solo in un punto di retry pulito.
- Se è già iniziata l'attività degli strumenti o l'output della risposta, il cambio in sospeso può restare in coda fino a una successiva opportunità di retry o al turno utente successivo.
- **Percorso rapido:** i messaggi composti solo da comandi provenienti da mittenti in allowlist vengono gestiti immediatamente (saltano coda + modello).
- **Controllo delle menzioni nei gruppi:** i messaggi composti solo da comandi provenienti da mittenti in allowlist bypassano i requisiti di menzione.
- **Scorciatoie inline (solo mittenti in allowlist):** alcuni comandi funzionano anche quando sono incorporati in un normale messaggio e vengono rimossi prima che il modello veda il testo rimanente.
  - Esempio: `hey /status` attiva una risposta di stato e il testo rimanente continua nel flusso normale.
- Attualmente: `/help`, `/commands`, `/status`, `/whoami` (`/id`).
- I messaggi composti solo da comandi non autorizzati vengono ignorati silenziosamente e i token inline `/...` vengono trattati come testo normale.
- **Comandi Skill:** le Skills `user-invocable` vengono esposte come slash command. I nomi vengono sanitizzati in `a-z0-9_` (massimo 32 caratteri); le collisioni ricevono suffissi numerici (ad esempio `_2`).
  - `/skill <name> [input]` esegue una Skill per nome (utile quando i limiti dei comandi nativi impediscono comandi per-Skill).
  - Per impostazione predefinita, i comandi Skill vengono inoltrati al modello come richiesta normale.
  - Le Skills possono facoltativamente dichiarare `command-dispatch: tool` per instradare il comando direttamente a uno strumento (deterministico, senza modello).
  - Esempio: `/prose` (plugin OpenProse) — consulta [OpenProse](/it/prose).
- **Argomenti dei comandi nativi:** Discord usa autocomplete per le opzioni dinamiche (e menu a pulsanti quando ometti gli argomenti richiesti). Telegram e Slack mostrano un menu a pulsanti quando un comando supporta scelte e ometti l'argomento.

## `/tools`

`/tools` risponde a una domanda runtime, non a una domanda di configurazione: **cosa può usare questo agent in questo momento in
questa conversazione**.

- Il valore predefinito di `/tools` è compatto e ottimizzato per una scansione rapida.
- `/tools verbose` aggiunge brevi descrizioni.
- Le superfici di comando nativo che supportano argomenti espongono lo stesso selettore di modalità `compact|verbose`.
- I risultati hanno ambito di sessione, quindi cambiare agent, canale, thread, autorizzazione del mittente o modello può
  cambiare l'output.
- `/tools` include gli strumenti realmente raggiungibili a runtime, inclusi strumenti core, strumenti
  di plugin connessi e strumenti posseduti dal canale.

Per modificare profili e override, usa il pannello Tools della Control UI o le superfici di configurazione/catalogo invece
di trattare `/tools` come un catalogo statico.

## Superfici di utilizzo (cosa appare dove)

- **Utilizzo/quota del provider** (esempio: “Claude 80% rimasto”) appare in `/status` per il provider del modello corrente quando il tracciamento dell'utilizzo è abilitato. OpenClaw normalizza le finestre dei provider in `% rimasto`; per MiniMax, i campi percentuali solo-rimanente vengono invertiti prima della visualizzazione, e le risposte `model_remains` privilegiano la voce del modello chat più un'etichetta del piano taggata sul modello.
- Le righe **token/cache** in `/status` possono fare fallback all'ultima voce di utilizzo nella trascrizione quando lo snapshot live della sessione è scarso. I valori live esistenti diversi da zero continuano comunque ad avere la precedenza e il fallback della trascrizione può anche recuperare l'etichetta del modello runtime attivo più un totale più grande orientato al prompt quando i totali archiviati mancano o sono inferiori.
- **Token/costo per risposta** è controllato da `/usage off|tokens|full` (aggiunto alle normali risposte).
- `/model status` riguarda **modelli/autenticazione/endpoint**, non l'utilizzo.

## Selezione del modello (`/model`)

`/model` è implementato come direttiva.

Esempi:

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model opus@anthropic:default
/model status
```

Note:

- `/model` e `/model list` mostrano un selettore compatto numerato (famiglia di modelli + provider disponibili).
- Su Discord, `/model` e `/models` aprono un selettore interattivo con menu a discesa di provider e modello più un passaggio Submit.
- `/model <#>` seleziona da quel selettore (e privilegia il provider corrente quando possibile).
- `/model status` mostra la vista dettagliata, inclusi endpoint provider configurato (`baseUrl`) e modalità API (`api`) quando disponibili.

## Override di debug

`/debug` consente di impostare override di configurazione **solo runtime** (in memoria, non su disco). Solo proprietario. Disabilitato per impostazione predefinita; abilitalo con `commands.debug: true`.

Esempi:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

Note:

- Gli override si applicano immediatamente alle nuove letture della configurazione, ma **non** scrivono in `openclaw.json`.
- Usa `/debug reset` per cancellare tutti gli override e tornare alla configurazione su disco.

## Aggiornamenti di configurazione

`/config` scrive nella configurazione su disco (`openclaw.json`). Solo proprietario. Disabilitato per impostazione predefinita; abilitalo con `commands.config: true`.

Esempi:

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

Note:

- La configurazione viene convalidata prima della scrittura; le modifiche non valide vengono rifiutate.
- Gli aggiornamenti `/config` persistono tra i riavvii.

## Aggiornamenti MCP

`/mcp` scrive le definizioni dei server MCP gestite da OpenClaw sotto `mcp.servers`. Solo proprietario. Disabilitato per impostazione predefinita; abilitalo con `commands.mcp: true`.

Esempi:

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

Note:

- `/mcp` archivia la configurazione nella configurazione OpenClaw, non nelle impostazioni di progetto possedute da Pi.
- Gli adapter runtime decidono quali trasporti sono effettivamente eseguibili.

## Aggiornamenti dei plugin

`/plugins` consente agli operatori di ispezionare i plugin rilevati e di attivare/disattivare l'abilitazione nella configurazione. I flussi in sola lettura possono usare `/plugin` come alias. Disabilitato per impostazione predefinita; abilitalo con `commands.plugins: true`.

Esempi:

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
```

Note:

- `/plugins list` e `/plugins show` usano la reale discovery dei plugin rispetto al workspace corrente più la configurazione su disco.
- `/plugins enable|disable` aggiorna solo la configurazione del plugin; non installa né disinstalla i plugin.
- Dopo modifiche di abilitazione/disabilitazione, riavvia il gateway per applicarle.

## Note sulle superfici

- **Comandi testuali** vengono eseguiti nella normale sessione chat (i DM condividono `main`, i gruppi hanno la propria sessione).
- **Comandi nativi** usano sessioni isolate:
  - Discord: `agent:<agentId>:discord:slash:<userId>`
  - Slack: `agent:<agentId>:slack:slash:<userId>` (prefisso configurabile tramite `channels.slack.slashCommand.sessionPrefix`)
  - Telegram: `telegram:slash:<userId>` (indirizza la sessione chat tramite `CommandTargetSessionKey`)
- **`/stop`** prende di mira la sessione chat attiva così può interrompere l'esecuzione corrente.
- **Slack:** `channels.slack.slashCommand` è ancora supportato per un singolo comando in stile `/openclaw`. Se abiliti `commands.native`, devi creare uno slash command Slack per ogni comando integrato (stessi nomi di `/help`). I menu degli argomenti dei comandi per Slack vengono forniti come pulsanti Block Kit effimeri.
  - Eccezione nativa Slack: registra `/agentstatus` (non `/status`) perché Slack riserva `/status`. Il testo `/status` continua a funzionare nei messaggi Slack.

## Domande laterali BTW

`/btw` è una rapida **domanda laterale** sulla sessione corrente.

A differenza della normale chat:

- usa la sessione corrente come contesto di sfondo,
- viene eseguita come chiamata one-shot separata **senza strumenti**,
- non modifica il contesto futuro della sessione,
- non viene scritta nella cronologia della trascrizione,
- viene consegnata come risultato laterale live invece che come normale messaggio dell'assistente.

Questo rende `/btw` utile quando vuoi un chiarimento temporaneo mentre il task principale
continua.

Esempio:

```text
/btw cosa stiamo facendo in questo momento?
```

Consulta [BTW Side Questions](/it/tools/btw) per il comportamento completo e i
dettagli UX del client.
