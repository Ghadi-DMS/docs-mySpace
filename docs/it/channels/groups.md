---
read_when:
    - Modifica del comportamento delle chat di gruppo o del controllo tramite menzione
summary: Comportamento delle chat di gruppo tra le varie superfici (Discord/iMessage/Matrix/Microsoft Teams/Signal/Slack/Telegram/WhatsApp/Zalo)
title: Gruppi
x-i18n:
    generated_at: "2026-04-06T03:06:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: 8620de6f7f0b866bf43a307fdbec3399790f09f22a87703704b0522caba80b18
    source_path: channels/groups.md
    workflow: 15
---

# Gruppi

OpenClaw gestisce le chat di gruppo in modo coerente tra le varie superfici: Discord, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo.

## Introduzione per principianti (2 minuti)

OpenClaw “vive” sui tuoi account di messaggistica. Non esiste un utente bot WhatsApp separato.
Se **tu** sei in un gruppo, OpenClaw può vedere quel gruppo e rispondere lì.

Comportamento predefinito:

- I gruppi sono limitati (`groupPolicy: "allowlist"`).
- Le risposte richiedono una menzione, a meno che tu non disattivi esplicitamente il controllo tramite menzione.

In altre parole: i mittenti presenti nella allowlist possono attivare OpenClaw menzionandolo.

> In breve
>
> - L'accesso ai **messaggi diretti** è controllato da `*.allowFrom`.
> - L'accesso ai **gruppi** è controllato da `*.groupPolicy` + allowlist (`*.groups`, `*.groupAllowFrom`).
> - L'**attivazione della risposta** è controllata dal controllo tramite menzione (`requireMention`, `/activation`).

Flusso rapido (cosa succede a un messaggio di gruppo):

```
groupPolicy? disabled -> scarta
groupPolicy? allowlist -> gruppo consentito? no -> scarta
requireMention? yes -> menzionato? no -> memorizza solo per il contesto
otherwise -> rispondi
```

## Visibilità del contesto e allowlist

Nella sicurezza dei gruppi sono coinvolti due controlli diversi:

- **Autorizzazione all'attivazione**: chi può attivare l'agente (`groupPolicy`, `groups`, `groupAllowFrom`, allowlist specifiche del canale).
- **Visibilità del contesto**: quale contesto supplementare viene inserito nel modello (testo di risposta, citazioni, cronologia del thread, metadati inoltrati).

Per impostazione predefinita, OpenClaw dà priorità al normale comportamento della chat e mantiene il contesto per lo più così come ricevuto. Questo significa che le allowlist decidono principalmente chi può attivare azioni, non un confine universale di redazione per ogni frammento citato o storico.

Il comportamento attuale dipende dal canale:

- Alcuni canali applicano già il filtro basato sul mittente per il contesto supplementare in percorsi specifici (ad esempio inizializzazione dei thread Slack, ricerche di risposte/thread Matrix).
- Altri canali continuano a inoltrare il contesto di citazione/risposta/inoltro così come ricevuto.

Direzione dell'irrobustimento (pianificata):

- `contextVisibility: "all"` (predefinito) mantiene l'attuale comportamento così come ricevuto.
- `contextVisibility: "allowlist"` filtra il contesto supplementare ai mittenti presenti nella allowlist.
- `contextVisibility: "allowlist_quote"` è `allowlist` più un'eccezione esplicita per citazioni/risposte.

Finché questo modello di irrobustimento non sarà implementato in modo coerente tra i canali, aspettati differenze a seconda della superficie.

![Flusso dei messaggi di gruppo](/images/groups-flow.svg)

Se vuoi...

| Obiettivo                                    | Cosa impostare                                             |
| -------------------------------------------- | ---------------------------------------------------------- |
| Consentire tutti i gruppi ma rispondere solo ai @mention | `groups: { "*": { requireMention: true } }`                |
| Disattivare tutte le risposte nei gruppi     | `groupPolicy: "disabled"`                                  |
| Solo gruppi specifici                        | `groups: { "<group-id>": { ... } }` (nessuna chiave `"*"`) |
| Solo tu puoi attivare nei gruppi             | `groupPolicy: "allowlist"`, `groupAllowFrom: ["+1555..."]` |

## Chiavi di sessione

- Le sessioni di gruppo usano chiavi di sessione `agent:<agentId>:<channel>:group:<id>` (le stanze/canali usano `agent:<agentId>:<channel>:channel:<id>`).
- Gli argomenti del forum Telegram aggiungono `:topic:<threadId>` all'id del gruppo, così ogni argomento ha la propria sessione.
- Le chat dirette usano la sessione principale (o una per mittente se configurato).
- Gli heartbeat vengono saltati per le sessioni di gruppo.

<a id="pattern-personal-dms-public-groups-single-agent"></a>

## Modello: messaggi diretti personali + gruppi pubblici (agente singolo)

Sì — funziona bene se il tuo traffico “personale” sono i **messaggi diretti** e il tuo traffico “pubblico” sono i **gruppi**.

Perché: in modalità agente singolo, i messaggi diretti di solito finiscono nella chiave di sessione **principale** (`agent:main:main`), mentre i gruppi usano sempre chiavi di sessione **non principali** (`agent:main:<channel>:group:<id>`). Se abiliti il sandboxing con `mode: "non-main"`, quelle sessioni di gruppo vengono eseguite in Docker mentre la tua sessione principale dei messaggi diretti rimane sull'host.

Questo ti offre un unico “cervello” dell'agente (workspace + memoria condivisi), ma due modalità di esecuzione:

- **Messaggi diretti**: strumenti completi (host)
- **Gruppi**: sandbox + strumenti limitati (Docker)

> Se ti servono workspace/persona davvero separati (“personale” e “pubblico” non devono mai mescolarsi), usa un secondo agente + bindings. Vedi [Instradamento multi-agente](/it/concepts/multi-agent).

Esempio (messaggi diretti sull'host, gruppi in sandbox + strumenti solo di messaggistica):

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // groups/channels are non-main -> sandboxed
        scope: "session", // strongest isolation (one container per group/channel)
        workspaceAccess: "none",
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        // If allow is non-empty, everything else is blocked (deny still wins).
        allow: ["group:messaging", "group:sessions"],
        deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"],
      },
    },
  },
}
```

Vuoi che “i gruppi possano vedere solo la cartella X” invece di “nessun accesso all'host”? Mantieni `workspaceAccess: "none"` e monta nel sandbox solo i percorsi presenti nella allowlist:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
        docker: {
          binds: [
            // hostPath:containerPath:mode
            "/home/user/FriendsShared:/data:ro",
          ],
        },
      },
    },
  },
}
```

Correlati:

- Chiavi di configurazione e valori predefiniti: [Configurazione del gateway](/it/gateway/configuration-reference#agentsdefaultssandbox)
- Debug del motivo per cui uno strumento è bloccato: [Sandbox vs Tool Policy vs Elevated](/it/gateway/sandbox-vs-tool-policy-vs-elevated)
- Dettagli sui bind mount: [Sandboxing](/it/gateway/sandboxing#custom-bind-mounts)

## Etichette di visualizzazione

- Le etichette dell'interfaccia usano `displayName` quando disponibile, con formato `<channel>:<token>`.
- `#room` è riservato a stanze/canali; le chat di gruppo usano `g-<slug>` (minuscolo, spazi -> `-`, mantiene `#@+._-`).

## Politica dei gruppi

Controlla come vengono gestiti i messaggi di gruppo/stanza per canale:

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"],
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789"], // numeric Telegram user id (wizard can resolve @username)
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"],
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"],
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"],
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        GUILD_ID: { channels: { help: { allow: true } } },
      },
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { allow: true } },
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { allow: true },
        "#alias:example.org": { allow: true },
      },
    },
  },
}
```

| Politica      | Comportamento                                               |
| ------------- | ----------------------------------------------------------- |
| `"open"`      | I gruppi bypassano le allowlist; il controllo tramite menzione si applica comunque. |
| `"disabled"`  | Blocca completamente tutti i messaggi di gruppo.            |
| `"allowlist"` | Consente solo gruppi/stanze che corrispondono alla allowlist configurata. |

Note:

- `groupPolicy` è separato dal controllo tramite menzione (che richiede @mention).
- WhatsApp/Telegram/Signal/iMessage/Microsoft Teams/Zalo: usa `groupAllowFrom` (fallback: `allowFrom` esplicito).
- Le approvazioni di abbinamento DM (voci salvate in `*-allowFrom`) si applicano solo all'accesso DM; l'autorizzazione del mittente nei gruppi resta esplicita tramite allowlist di gruppo.
- Discord: la allowlist usa `channels.discord.guilds.<id>.channels`.
- Slack: la allowlist usa `channels.slack.channels`.
- Matrix: la allowlist usa `channels.matrix.groups`. Preferisci gli ID stanza o gli alias; la ricerca del nome delle stanze unite è best-effort e i nomi non risolti vengono ignorati in fase di esecuzione. Usa `channels.matrix.groupAllowFrom` per limitare i mittenti; sono supportate anche allowlist `users` per stanza.
- I DM di gruppo sono controllati separatamente (`channels.discord.dm.*`, `channels.slack.dm.*`).
- La allowlist Telegram può corrispondere a ID utente (`"123456789"`, `"telegram:123456789"`, `"tg:123456789"`) o nomi utente (`"@alice"` o `"alice"`); i prefissi non distinguono tra maiuscole e minuscole.
- Il valore predefinito è `groupPolicy: "allowlist"`; se la tua allowlist di gruppo è vuota, i messaggi di gruppo vengono bloccati.
- Sicurezza di runtime: quando un blocco provider manca completamente (`channels.<provider>` assente), la politica dei gruppi torna a una modalità fail-closed (tipicamente `allowlist`) invece di ereditare `channels.defaults.groupPolicy`.

Modello mentale rapido (ordine di valutazione per i messaggi di gruppo):

1. `groupPolicy` (open/disabled/allowlist)
2. allowlist di gruppo (`*.groups`, `*.groupAllowFrom`, allowlist specifica del canale)
3. controllo tramite menzione (`requireMention`, `/activation`)

## Controllo tramite menzione (predefinito)

I messaggi di gruppo richiedono una menzione, a meno che non venga sovrascritto per gruppo. I valori predefiniti si trovano per sottosistema in `*.groups."*"`.

Rispondere a un messaggio del bot conta come una menzione implicita (quando il canale supporta i metadati di risposta). Questo si applica a Telegram, WhatsApp, Slack, Discord e Microsoft Teams.

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false },
      },
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false },
      },
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false },
      },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw", "\\+15555550123"],
          historyLimit: 50,
        },
      },
    ],
  },
}
```

Note:

- `mentionPatterns` sono pattern regex sicuri e case-insensitive; i pattern non validi e le forme non sicure con ripetizione annidata vengono ignorati.
- Le superfici che forniscono menzioni esplicite continuano a passare; i pattern sono un fallback.
- Override per agente: `agents.list[].groupChat.mentionPatterns` (utile quando più agenti condividono un gruppo).
- Il controllo tramite menzione viene applicato solo quando è possibile rilevare la menzione (menzioni native o `mentionPatterns` configurati).
- I valori predefiniti di Discord si trovano in `channels.discord.guilds."*"` (sovrascrivibile per server/canale).
- Il contesto della cronologia di gruppo è racchiuso in modo uniforme tra i canali ed è **solo pending** (messaggi saltati a causa del controllo tramite menzione); usa `messages.groupChat.historyLimit` per il valore predefinito globale e `channels.<channel>.historyLimit` (o `channels.<channel>.accounts.*.historyLimit`) per gli override. Imposta `0` per disattivarlo.

## Restrizioni degli strumenti per gruppo/canale (facoltativo)

Alcune configurazioni dei canali supportano la limitazione degli strumenti disponibili **all'interno di uno specifico gruppo/stanza/canale**.

- `tools`: consenti/nega strumenti per l'intero gruppo.
- `toolsBySender`: override per mittente all'interno del gruppo.
  Usa prefissi di chiave espliciti:
  `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` e il carattere jolly `"*"`.
  Le chiavi legacy senza prefisso sono ancora accettate e corrispondono solo come `id:`.

Ordine di risoluzione (vince il più specifico):

1. corrispondenza `toolsBySender` del gruppo/canale
2. `tools` del gruppo/canale
3. corrispondenza `toolsBySender` predefinita (`"*"`)
4. `tools` predefinito (`"*"`)

Esempio (Telegram):

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { tools: { deny: ["exec"] } },
        "-1001234567890": {
          tools: { deny: ["exec", "read", "write"] },
          toolsBySender: {
            "id:123456789": { alsoAllow: ["exec"] },
          },
        },
      },
    },
  },
}
```

Note:

- Le restrizioni degli strumenti per gruppo/canale vengono applicate in aggiunta alla politica globale/di agente degli strumenti (deny ha comunque la precedenza).
- Alcuni canali usano nidificazioni diverse per stanze/canali (ad esempio Discord `guilds.*.channels.*`, Slack `channels.*`, Microsoft Teams `teams.*.channels.*`).

## Allowlist di gruppo

Quando `channels.whatsapp.groups`, `channels.telegram.groups` o `channels.imessage.groups` è configurato, le chiavi agiscono come una allowlist di gruppo. Usa `"*"` per consentire tutti i gruppi continuando comunque a impostare il comportamento predefinito delle menzioni.

Confusione comune: l'approvazione dell'abbinamento DM non è la stessa cosa dell'autorizzazione dei gruppi.
Per i canali che supportano l'abbinamento DM, l'archivio degli abbinamenti sblocca solo i DM. I comandi di gruppo richiedono comunque un'autorizzazione esplicita del mittente del gruppo dalle allowlist di configurazione come `groupAllowFrom` o dal fallback di configurazione documentato per quel canale.

Intenti comuni (copia/incolla):

1. Disattivare tutte le risposte nei gruppi

```json5
{
  channels: { whatsapp: { groupPolicy: "disabled" } },
}
```

2. Consentire solo gruppi specifici (WhatsApp)

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "123@g.us": { requireMention: true },
        "456@g.us": { requireMention: false },
      },
    },
  },
}
```

3. Consentire tutti i gruppi ma richiedere la menzione (esplicito)

```json5
{
  channels: {
    whatsapp: {
      groups: { "*": { requireMention: true } },
    },
  },
}
```

4. Solo il proprietario può attivare nei gruppi (WhatsApp)

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
      groups: { "*": { requireMention: true } },
    },
  },
}
```

## Attivazione (solo proprietario)

I proprietari del gruppo possono attivare o disattivare l'attivazione per gruppo:

- `/activation mention`
- `/activation always`

Il proprietario è determinato da `channels.whatsapp.allowFrom` (o dall'E.164 del bot stesso se non impostato). Invia il comando come messaggio autonomo. Le altre superfici al momento ignorano `/activation`.

## Campi del contesto

I payload in ingresso dei gruppi impostano:

- `ChatType=group`
- `GroupSubject` (se noto)
- `GroupMembers` (se noto)
- `WasMentioned` (risultato del controllo tramite menzione)
- Gli argomenti del forum Telegram includono anche `MessageThreadId` e `IsForum`.

Note specifiche del canale:

- BlueBubbles può opzionalmente arricchire i partecipanti senza nome dei gruppi macOS dal database locale dei Contatti prima di popolare `GroupMembers`. Questa opzione è disattivata per impostazione predefinita e viene eseguita solo dopo il normale superamento del controllo del gruppo.

Il prompt di sistema dell'agente include un'introduzione al gruppo al primo turno di una nuova sessione di gruppo. Ricorda al modello di rispondere come una persona, evitare tabelle Markdown, ridurre al minimo le righe vuote e seguire la normale spaziatura della chat, oltre a evitare di digitare sequenze letterali `\n`.

## Specifiche iMessage

- Preferisci `chat_id:<id>` quando esegui routing o allowlisting.
- Elenca le chat: `imsg chats --limit 20`.
- Le risposte nei gruppi tornano sempre allo stesso `chat_id`.

## Specifiche WhatsApp

Vedi [Messaggi di gruppo](/it/channels/group-messages) per il comportamento specifico di WhatsApp (iniezione della cronologia, dettagli sulla gestione delle menzioni).
