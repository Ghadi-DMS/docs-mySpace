---
read_when:
    - Configurazione di approvazioni exec o allowlist
    - Implementazione della UX delle approvazioni exec nell'app macOS
    - Revisione dei prompt di uscita dalla sandbox e delle relative implicazioni
summary: Approvazioni exec, allowlist e prompt di uscita dalla sandbox
title: Approvazioni exec
x-i18n:
    generated_at: "2026-04-06T03:14:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 39e91cd5c7615bdb9a6b201a85bde7514327910f6f12da5a4b0532bceb229c22
    source_path: tools/exec-approvals.md
    workflow: 15
---

# Approvazioni exec

Le approvazioni exec sono il **guardrail dell'app companion / host node** per consentire a un agente in sandbox di eseguire
comandi su un host reale (`gateway` o `node`). Considerale come un interblocco di sicurezza:
i comandi sono consentiti solo quando policy + allowlist + (eventuale) approvazione dell'utente concordano tutte.
Le approvazioni exec sono **in aggiunta** alla policy degli strumenti e al gating elevated (a meno che elevated non sia impostato su `full`, che salta le approvazioni).
La policy effettiva è la **più restrittiva** tra `tools.exec.*` e i valori predefiniti delle approvazioni; se un campo delle approvazioni viene omesso, viene usato il valore di `tools.exec`.
L'exec host usa anche lo stato locale delle approvazioni su quella macchina. Un valore host-local
`ask: "always"` in `~/.openclaw/exec-approvals.json` continua a mostrare prompt anche se
i valori predefiniti della sessione o della configurazione richiedono `ask: "on-miss"`.
Usa `openclaw approvals get`, `openclaw approvals get --gateway` o
`openclaw approvals get --node <id|name|ip>` per ispezionare la policy richiesta,
le sorgenti della policy host e il risultato effettivo.

Se la UI dell'app companion **non è disponibile**, qualsiasi richiesta che richiede un prompt viene
risolta tramite il **fallback ask** (predefinito: deny).

I client di approvazione chat nativi possono anche esporre affordance specifiche del canale sul
messaggio di approvazione in sospeso. Per esempio, Matrix può inizializzare shortcut di reazione sul
prompt di approvazione (`✅` consenti una volta, `❌` nega e `♾️` consenti sempre quando disponibile)
lasciando comunque i comandi `/approve ...` nel messaggio come fallback.

## Dove si applica

Le approvazioni exec sono applicate localmente sull'host di esecuzione:

- **gateway host** → processo `openclaw` sulla macchina gateway
- **node host** → runner del node (app companion macOS o host node headless)

Nota sul modello di fiducia:

- I chiamanti autenticati dal gateway sono operatori fidati per quel Gateway.
- I node abbinati estendono questa capacità di operatore fidato all'host node.
- Le approvazioni exec riducono il rischio di esecuzioni accidentali, ma non costituiscono un confine di autenticazione per utente.
- Le esecuzioni approvate sull'host node vincolano il contesto canonico di esecuzione: cwd canonico, argv esatto, binding
  dell'env quando presente e percorso dell'eseguibile fissato quando applicabile.
- Per shell script e invocazioni dirette di file tramite interprete/runtime, OpenClaw prova anche a vincolare
  un operando file locale concreto. Se quel file vincolato cambia dopo l'approvazione ma prima dell'esecuzione,
  l'esecuzione viene negata invece di eseguire contenuto modificato.
- Questo vincolo del file è intenzionalmente best-effort, non un modello semantico completo di ogni
  percorso di caricamento di interpreti/runtime. Se la modalità di approvazione non riesce a identificare esattamente un file locale concreto
  da vincolare, rifiuta di emettere un'esecuzione supportata da approvazione invece di fingere copertura completa.

Suddivisione macOS:

- **servizio host node** inoltra `system.run` alla **app macOS** tramite IPC locale.
- **app macOS** applica le approvazioni + esegue il comando nel contesto UI.

## Impostazioni e archiviazione

Le approvazioni vivono in un file JSON locale sull'host di esecuzione:

`~/.openclaw/exec-approvals.json`

Schema di esempio:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

## Modalità "YOLO" senza approvazioni

Se vuoi che l'exec host venga eseguito senza prompt di approvazione, devi aprire **entrambi** i livelli di policy:

- policy exec richiesta nella configurazione OpenClaw (`tools.exec.*`)
- policy di approvazione host-local in `~/.openclaw/exec-approvals.json`

Questo ora è il comportamento host predefinito, a meno che tu non lo restringa esplicitamente:

- `tools.exec.security`: `full` su `gateway`/`node`
- `tools.exec.ask`: `off`
- host `askFallback`: `full`

Distinzione importante:

- `tools.exec.host=auto` sceglie dove viene eseguito exec: sandbox quando disponibile, altrimenti gateway.
- YOLO sceglie come viene approvato l'exec host: `security=full` più `ask=off`.
- In modalità YOLO, OpenClaw non aggiunge un gate separato di approvazione euristica per l'offuscamento dei comandi sopra la policy exec host configurata.
- `auto` non rende l'instradamento gateway un override libero da una sessione in sandbox. Una richiesta per chiamata `host=node` è consentita da `auto`, e `host=gateway` è consentito da `auto` solo quando non è attivo alcun runtime sandbox. Se vuoi un valore predefinito stabile non-auto, imposta `tools.exec.host` o usa `/exec host=...` esplicitamente.

Se vuoi una configurazione più conservativa, restringi di nuovo uno dei due livelli a `allowlist` / `on-miss`
oppure `deny`.

Configurazione persistente "mai chiedere" per gateway-host:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
openclaw gateway restart
```

Poi imposta il file delle approvazioni host in modo corrispondente:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Per un host node, applica invece lo stesso file di approvazioni su quel node:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

Scorciatoia solo-sessione:

- `/exec security=full ask=off` cambia solo la sessione corrente.
- `/elevated full` è una scorciatoia break-glass che salta anche le approvazioni exec per quella sessione.

Se il file delle approvazioni host resta più restrittivo della configurazione, continua comunque a prevalere la policy host più restrittiva.

## Manopole della policy

### Sicurezza (`exec.security`)

- **deny**: blocca tutte le richieste exec host.
- **allowlist**: consente solo i comandi in allowlist.
- **full**: consente tutto (equivalente a elevated).

### Ask (`exec.ask`)

- **off**: non mostra mai prompt.
- **on-miss**: mostra un prompt solo quando l'allowlist non corrisponde.
- **always**: mostra un prompt per ogni comando.
- La fiducia durevole `allow-always` non sopprime i prompt quando la modalità ask effettiva è `always`

### Ask fallback (`askFallback`)

Se è richiesto un prompt ma nessuna UI è raggiungibile, fallback decide:

- **deny**: blocca.
- **allowlist**: consente solo se l'allowlist corrisponde.
- **full**: consente.

### Hardening delle eval inline dell'interprete (`tools.exec.strictInlineEval`)

Quando `tools.exec.strictInlineEval=true`, OpenClaw tratta le forme inline di code-eval come solo-approvazione anche se il binario dell'interprete stesso è in allowlist.

Esempi:

- `python -c`
- `node -e`, `node --eval`, `node -p`
- `ruby -e`
- `perl -e`, `perl -E`
- `php -r`
- `lua -e`
- `osascript -e`

Questa è una difesa in profondità per loader di interpreti che non si mappano pulitamente su un solo operando file stabile. In modalità strict:

- questi comandi richiedono comunque approvazione esplicita;
- `allow-always` non rende persistenti automaticamente nuove voci di allowlist per essi.

## Allowlist (per agente)

Le allowlist sono **per agente**. Se esistono più agenti, cambia l'agente che stai
modificando nell'app macOS. I pattern sono **glob case-insensitive**.
I pattern dovrebbero risolversi in **percorsi di binari** (le voci con solo basename vengono ignorate).
Le voci legacy `agents.default` vengono migrate in `agents.main` al caricamento.
Le catene shell come `echo ok && pwd` richiedono comunque che ogni segmento di primo livello soddisfi le regole della allowlist.

Esempi:

- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

Ogni voce di allowlist tiene traccia di:

- **id** UUID stabile usato per l'identità nella UI (facoltativo)
- **ultimo utilizzo** timestamp
- **ultimo comando usato**
- **ultimo percorso risolto**

## Auto-allow delle CLI delle Skills

Quando **Auto-allow skill CLIs** è abilitato, gli eseguibili referenziati da Skills note
sono trattati come presenti in allowlist sui node (node macOS o host node headless). Questo usa
`skills.bins` tramite il Gateway RPC per recuperare l'elenco dei binari delle skill. Disabilitalo se vuoi allowlist manuali rigorose.

Note importanti sul modello di fiducia:

- Questa è una **allowlist implicita di comodità**, separata dalle voci manuali della allowlist dei percorsi.
- È pensata per ambienti con operatori fidati in cui Gateway e node si trovano nello stesso confine di fiducia.
- Se richiedi fiducia esplicita rigorosa, mantieni `autoAllowSkills: false` e usa solo voci manuali della allowlist dei percorsi.

## Safe bins (solo stdin)

`tools.exec.safeBins` definisce un piccolo elenco di binari **solo stdin** (per esempio `cut`)
che possono essere eseguiti in modalità allowlist **senza** voci esplicite di allowlist. I safe bins rifiutano
argomenti posizionali di file e token simili a percorsi, quindi possono operare solo sul flusso in ingresso.
Consideralo un percorso veloce ristretto per filtri di stream, non un elenco generale di fiducia.
**Non** aggiungere binari di interpreti o runtime (per esempio `python3`, `node`, `ruby`, `bash`, `sh`, `zsh`) a `safeBins`.
Se un comando può valutare codice, eseguire sottocomandi o leggere file per progettazione, preferisci voci esplicite di allowlist e mantieni abilitati i prompt di approvazione.
I safe bins personalizzati devono definire un profilo esplicito in `tools.exec.safeBinProfiles.<bin>`.
La validazione è deterministica solo dalla forma di argv (nessun controllo dell'esistenza del filesystem host), il che
impedisce comportamenti da oracolo di esistenza file dovuti a differenze allow/deny.
Le opzioni orientate ai file sono negate per i safe bins predefiniti (per esempio `sort -o`, `sort --output`,
`sort --files0-from`, `sort --compress-program`, `sort --random-source`,
`sort --temporary-directory`/`-T`, `wc --files0-from`, `jq -f/--from-file`,
`grep -f/--file`).
I safe bins impongono anche una policy esplicita per binario sui flag per opzioni che rompono il
comportamento solo stdin (per esempio `sort -o/--output/--compress-program` e i flag ricorsivi di grep).
Le opzioni lunghe sono validate in modalità fail-closed per i safe bins: flag sconosciuti e
abbreviazioni ambigue vengono rifiutati.
Flag negati per profilo safe-bin:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

I safe bins forzano inoltre che i token argv siano trattati come **testo letterale** in fase di esecuzione (niente globbing
e niente espansione di `$VARS`) per i segmenti solo stdin, così pattern come `*` o `$HOME/...` non possono essere
usati per introdurre letture di file di nascosto.
I safe bins devono anche risolversi da directory di binari fidate (predefinite di sistema più eventuali
`tools.exec.safeBinTrustedDirs`). Le voci di `PATH` non sono mai fidate automaticamente.
Le directory safe-bin fidate predefinite sono intenzionalmente minime: `/bin`, `/usr/bin`.
Se il tuo eseguibile safe-bin vive in percorsi di package manager/utente (per esempio
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), aggiungili esplicitamente
a `tools.exec.safeBinTrustedDirs`.
Le catene shell e i redirect non sono consentiti automaticamente in modalità allowlist.

Le catene shell (`&&`, `||`, `;`) sono consentite quando ogni segmento di primo livello soddisfa la allowlist
(inclusi safe bins o auto-allow delle skill). I redirect restano non supportati in modalità allowlist.
La sostituzione di comando (`$()` / backtick) viene rifiutata durante il parsing della allowlist, anche all'interno
di virgolette doppie; usa virgolette singole se ti serve testo letterale `$()`.
Nelle approvazioni dell'app companion macOS, testo shell grezzo contenente sintassi di controllo o espansione shell
(`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) è trattato come mancata corrispondenza con l'allowlist a meno che
il binario shell stesso non sia in allowlist.
Per wrapper shell (`bash|sh|zsh ... -c/-lc`), gli override env con scope richiesta sono ridotti a una
piccola allowlist esplicita (`TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`).
Per le decisioni `allow-always` in modalità allowlist, i wrapper di dispatch noti
(`env`, `nice`, `nohup`, `stdbuf`, `timeout`) rendono persistenti i percorsi degli eseguibili interni invece dei percorsi
dei wrapper. Anche i multiplexer shell (`busybox`, `toybox`) vengono unwrap per le applet shell (`sh`, `ash`,
ecc.) così vengono resi persistenti gli eseguibili interni invece dei binari del multiplexer. Se un wrapper o
multiplexer non può essere unwrap in sicurezza, nessuna voce di allowlist viene resa persistente automaticamente.
Se metti in allowlist interpreti come `python3` o `node`, preferisci `tools.exec.strictInlineEval=true` così l'eval inline richiede comunque un'approvazione esplicita. In modalità strict, `allow-always` può ancora rendere persistenti invocazioni innocue di interpreti/script, ma i vettori di inline-eval non vengono resi persistenti automaticamente.

Safe bins predefiniti:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` e `sort` non fanno parte dell'elenco predefinito. Se abiliti il loro uso, mantieni voci esplicite di allowlist per
i loro workflow non-stdin.
Per `grep` in modalità safe-bin, fornisci il pattern con `-e`/`--regexp`; la forma con pattern posizionale viene
rifiutata così gli operandi file non possono essere introdotti di nascosto come posizionali ambigui.

### Safe bins rispetto alla allowlist

| Argomento | `tools.exec.safeBins` | Allowlist (`exec-approvals.json`) |
| --- | --- | --- |
| Obiettivo | Consentire automaticamente filtri stdin ristretti | Affidare esplicitamente eseguibili specifici |
| Tipo di corrispondenza | Nome dell'eseguibile + policy argv del safe-bin | Pattern glob del percorso dell'eseguibile risolto |
| Ambito degli argomenti | Limitato dal profilo safe-bin e dalle regole sui token letterali | Solo corrispondenza del percorso; gli argomenti sono altrimenti tua responsabilità |
| Esempi tipici | `head`, `tail`, `tr`, `wc` | `jq`, `python3`, `node`, `ffmpeg`, CLI personalizzate |
| Uso migliore | Trasformazioni testuali a basso rischio nelle pipeline | Qualsiasi strumento con comportamento più ampio o effetti collaterali |

Posizione della configurazione:

- `safeBins` proviene dalla configurazione (`tools.exec.safeBins` o per agente `agents.list[].tools.exec.safeBins`).
- `safeBinTrustedDirs` proviene dalla configurazione (`tools.exec.safeBinTrustedDirs` o per agente `agents.list[].tools.exec.safeBinTrustedDirs`).
- `safeBinProfiles` proviene dalla configurazione (`tools.exec.safeBinProfiles` o per agente `agents.list[].tools.exec.safeBinProfiles`). Le chiavi del profilo per agente sovrascrivono quelle globali.
- Le voci della allowlist vivono nel file host-local `~/.openclaw/exec-approvals.json` sotto `agents.<id>.allowlist` (o tramite Control UI / `openclaw approvals allowlist ...`).
- `openclaw security audit` avvisa con `tools.exec.safe_bins_interpreter_unprofiled` quando binari di interpreti/runtime compaiono in `safeBins` senza profili espliciti.
- `openclaw doctor --fix` può creare scaffolding per voci mancanti `safeBinProfiles.<bin>` come `{}` (rivedile e restringile in seguito). I binari di interpreti/runtime non vengono scaffoldati automaticamente.

Esempio di profilo personalizzato:
__OC_I18N_900004__
Se abiliti esplicitamente `jq` in `safeBins`, OpenClaw rifiuta comunque il builtin `env` in modalità safe-bin
così `jq -n env` non può esporre l'ambiente del processo host senza un percorso esplicito in allowlist
o un prompt di approvazione.

## Modifica nella Control UI

Usa la scheda **Control UI → Nodes → Exec approvals** per modificare valori predefiniti, override
per agente e allowlist. Scegli uno scope (Defaults o un agente), regola la policy,
aggiungi/rimuovi pattern di allowlist, poi **Save**. La UI mostra metadati di **ultimo utilizzo**
per pattern così puoi mantenere l'elenco ordinato.

Il selettore di target sceglie **Gateway** (approvazioni locali) oppure un **Node**. I node
devono pubblicizzare `system.execApprovals.get/set` (app macOS o host node headless).
Se un node non pubblicizza ancora le approvazioni exec, modifica direttamente il suo file locale
`~/.openclaw/exec-approvals.json`.

CLI: `openclaw approvals` supporta la modifica di gateway o node (vedi [Approvals CLI](/cli/approvals)).

## Flusso di approvazione

Quando è richiesto un prompt, il gateway trasmette `exec.approval.requested` ai client operatore.
La Control UI e l'app macOS lo risolvono tramite `exec.approval.resolve`, poi il gateway inoltra la
richiesta approvata all'host node.

Per `host=node`, le richieste di approvazione includono un payload canonico `systemRunPlan`. Il gateway usa
quel piano come contesto autorevole di comando/cwd/sessione quando inoltra richieste approvate di `system.run`.

Questo è importante per la latenza delle approvazioni asincrone:

- il percorso exec del node prepara in anticipo un piano canonico
- il record di approvazione memorizza quel piano e i relativi metadati di binding
- una volta approvata, la chiamata finale `system.run` inoltrata riusa il piano memorizzato
  invece di fidarsi di modifiche successive del chiamante
- se il chiamante cambia `command`, `rawCommand`, `cwd`, `agentId` o
  `sessionKey` dopo la creazione della richiesta di approvazione, il gateway rifiuta
  l'esecuzione inoltrata come mancata corrispondenza dell'approvazione

## Comandi di interpreti/runtime

Le esecuzioni di interpreti/runtime supportate da approvazione sono intenzionalmente conservative:

- Il contesto esatto di argv/cwd/env è sempre vincolato.
- Le forme dirette di shell script e file runtime sono vincolate in modo best-effort a uno snapshot concreto di un file locale.
- Le comuni forme wrapper dei package manager che si risolvono comunque in un solo file locale diretto (per esempio
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`) vengono unwrap prima del binding.
- Se OpenClaw non riesce a identificare esattamente un singolo file locale concreto per un comando di interprete/runtime
  (per esempio script di package, forme eval, catene di loader specifiche del runtime o forme ambigue con più file),
  l'esecuzione supportata da approvazione viene negata invece di rivendicare una copertura semantica che non
  possiede.
- Per questi workflow, preferisci sandboxing, un confine host separato oppure un workflow esplicito trusted
  con allowlist/full in cui l'operatore accetta la semantica runtime più ampia.

Quando sono richieste approvazioni, lo strumento exec restituisce immediatamente un id di approvazione. Usa quell'id per
correlare eventi di sistema successivi (`Exec finished` / `Exec denied`). Se nessuna decisione arriva prima del
timeout, la richiesta viene trattata come timeout di approvazione e riportata come motivo di negazione.

### Comportamento del recapito di followup

Dopo che un exec asincrono approvato termina, OpenClaw invia un turno di followup `agent` alla stessa sessione.

- Se esiste un target di recapito esterno valido (canale recapitabile più target `to`), il recapito del followup usa quel canale.
- Nei flussi solo webchat o sessione interna senza target esterno, il recapito del followup resta solo-sessione (`deliver: false`).
- Se un chiamante richiede esplicitamente un recapito esterno rigoroso senza alcun canale esterno risolvibile, la richiesta fallisce con `INVALID_REQUEST`.
- Se `bestEffortDeliver` è abilitato e nessun canale esterno può essere risolto, il recapito viene declassato a solo-sessione invece di fallire.

La finestra di dialogo di conferma include:

- comando + argomenti
- cwd
- id agente
- percorso dell'eseguibile risolto
- host + metadati della policy

Azioni:

- **Allow once** → esegui ora
- **Always allow** → aggiungi alla allowlist + esegui
- **Deny** → blocca

## Inoltro delle approvazioni ai canali chat

Puoi inoltrare i prompt di approvazione exec a qualsiasi canale chat (inclusi i plugin di canale) e approvarli
con `/approve`. Questo usa la normale pipeline di recapito outbound.

Configurazione:
__OC_I18N_900005__
Rispondi in chat:
__OC_I18N_900006__
Il comando `/approve` gestisce sia le approvazioni exec sia quelle dei plugin. Se l'ID non corrisponde a un'approvazione exec in sospeso, controlla automaticamente invece le approvazioni dei plugin.

### Inoltro delle approvazioni dei plugin

L'inoltro delle approvazioni dei plugin usa la stessa pipeline di recapito delle approvazioni exec ma ha una
configurazione indipendente sotto `approvals.plugin`. Abilitare o disabilitare una delle due non influisce sull'altra.
__OC_I18N_900007__
La forma della configurazione è identica a `approvals.exec`: `enabled`, `mode`, `agentFilter`,
`sessionFilter` e `targets` funzionano allo stesso modo.

I canali che supportano risposte interattive condivise renderizzano gli stessi pulsanti di approvazione sia per le approvazioni exec sia per quelle dei plugin. I canali senza UI interattiva condivisa ripiegano su testo semplice con istruzioni `/approve`.

### Approvazioni nella stessa chat su qualsiasi canale

Quando una richiesta di approvazione exec o plugin ha origine da una superficie chat recapitabile, la stessa chat
può ora approvarla con `/approve` per impostazione predefinita. Questo vale per canali come Slack, Matrix e
Microsoft Teams oltre ai flussi esistenti di Web UI e terminal UI.

Questo percorso condiviso a comando testuale usa il normale modello auth del canale per quella conversazione. Se la
chat di origine può già inviare comandi e ricevere risposte, le richieste di approvazione non hanno più bisogno di un
adattatore nativo di recapito separato solo per restare in sospeso.

Discord e Telegram supportano anche `/approve` nella stessa chat, ma quei canali usano comunque il loro
elenco approver risolto per l'autorizzazione anche quando il recapito nativo delle approvazioni è disabilitato.

Per Telegram e altri client di approvazione nativi che chiamano direttamente il Gateway,
questo fallback è intenzionalmente limitato ai fallimenti "approval not found". Un vero
rifiuto/errore di approvazione exec non viene ritentato silenziosamente come approvazione plugin.

### Recapito nativo delle approvazioni

Alcuni canali possono anche agire come client di approvazione nativi. I client nativi aggiungono DM degli approver, fanout
alla chat di origine e UX interattiva delle approvazioni specifica del canale sopra il flusso condiviso `/approve`
nella stessa chat.

Quando sono disponibili card/pulsanti nativi di approvazione, quella UI nativa è il percorso principale
rivolto all'agente. L'agente non dovrebbe anche ripetere un duplicato in chiaro del comando chat
`/approve` a meno che il risultato dello strumento dica che le approvazioni chat non sono disponibili o che
l'approvazione manuale è l'unico percorso rimasto.

Modello generico:

- la policy exec host decide comunque se è richiesta approvazione exec
- `approvals.exec` controlla l'inoltro dei prompt di approvazione verso altre destinazioni chat
- `channels.<channel>.execApprovals` controlla se quel canale agisce come client di approvazione nativo

I client di approvazione nativi abilitano automaticamente il recapito DM-first quando tutte queste condizioni sono vere:

- il canale supporta il recapito nativo delle approvazioni
- gli approver possono essere risolti da `execApprovals.approvers` espliciti o dalle sorgenti di fallback documentate di quel
  canale
- `channels.<channel>.execApprovals.enabled` non è impostato oppure è `"auto"`

Imposta `enabled: false` per disabilitare esplicitamente un client di approvazione nativo. Imposta `enabled: true` per forzarne
l'attivazione quando gli approver si risolvono. Il recapito pubblico alla chat di origine resta esplicito tramite
`channels.<channel>.execApprovals.target`.

FAQ: [Perché esistono due configurazioni di approvazione exec per le approvazioni chat?](/help/faq#why-are-there-two-exec-approval-configs-for-chat-approvals)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`

Questi client di approvazione nativi aggiungono instradamento DM e fanout facoltativo del canale sopra il flusso condiviso
`/approve` nella stessa chat e i pulsanti di approvazione condivisi.

Comportamento condiviso:

- Slack, Matrix, Microsoft Teams e chat recapitabili simili usano il normale modello auth del canale
  per `/approve` nella stessa chat
- quando un client di approvazione nativo si autoabilita, il target di recapito nativo predefinito sono i DM degli approver
- per Discord e Telegram, solo gli approver risolti possono approvare o negare
- gli approver Discord possono essere espliciti (`execApprovals.approvers`) o dedotti da `commands.ownerAllowFrom`
- gli approver Telegram possono essere espliciti (`execApprovals.approvers`) o dedotti dalla configurazione owner esistente (`allowFrom`, più il `defaultTo` dei messaggi diretti dove supportato)
- gli approver Slack possono essere espliciti (`execApprovals.approvers`) o dedotti da `commands.ownerAllowFrom`
- i pulsanti nativi Slack preservano il tipo di id di approvazione, quindi gli id `plugin:` possono risolvere le approvazioni dei plugin
  senza un secondo livello di fallback locale a Slack
- l'instradamento nativo DM/canale di Matrix è solo exec; le approvazioni plugin di Matrix restano sul percorso condiviso
  `/approve` nella stessa chat e sui percorsi facoltativi di inoltro `approvals.plugin`
- il richiedente non deve necessariamente essere un approver
- la chat di origine può approvare direttamente con `/approve` quando supporta già comandi e risposte
- i pulsanti nativi di approvazione Discord instradano in base al tipo di id di approvazione: gli id `plugin:` vanno
  direttamente alle approvazioni plugin, tutto il resto va alle approvazioni exec
- i pulsanti nativi di approvazione Telegram seguono lo stesso fallback limitato da exec a plugin di `/approve`
- quando `target` nativo abilita il recapito alla chat di origine, i prompt di approvazione includono il testo del comando
- le approvazioni exec in sospeso scadono dopo 30 minuti per impostazione predefinita
- se nessuna UI operatore o client di approvazione configurato può accettare la richiesta, il prompt ricade su `askFallback`

Telegram usa per impostazione predefinita i DM degli approver (`target: "dm"`). Puoi passare a `channel` o `both` quando
vuoi che i prompt di approvazione compaiano anche nella chat/topic Telegram di origine. Per i topic forum di Telegram,
OpenClaw preserva il topic per il prompt di approvazione e per il follow-up post-approvazione.

Vedi:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)

### Flusso IPC macOS
__OC_I18N_900008__
Note sulla sicurezza:

- Modalità socket Unix `0600`, token memorizzato in `exec-approvals.json`.
- Controllo del peer con stesso UID.
- Challenge/response (nonce + token HMAC + hash della richiesta) + TTL breve.

## Eventi di sistema

Il ciclo di vita exec è esposto come messaggi di sistema:

- `Exec running` (solo se il comando supera la soglia dell'avviso di esecuzione)
- `Exec finished`
- `Exec denied`

Questi vengono pubblicati nella sessione dell'agente dopo che il node riporta l'evento.
Le approvazioni exec gateway-host emettono gli stessi eventi di ciclo di vita quando il comando termina (e facoltativamente quando è in esecuzione oltre la soglia).
Gli exec protetti da approvazione riutilizzano l'id di approvazione come `runId` in questi messaggi per una facile correlazione.

## Comportamento delle approvazioni negate

Quando un'approvazione exec asincrona viene negata, OpenClaw impedisce all'agente di riutilizzare
l'output di qualsiasi esecuzione precedente dello stesso comando nella sessione. Il motivo della negazione
viene passato con indicazioni esplicite che nessun output del comando è disponibile, il che impedisce
all'agente di affermare che esiste nuovo output o di ripetere il comando negato con
risultati obsoleti di una precedente esecuzione riuscita.

## Implicazioni

- **full** è potente; preferisci le allowlist quando possibile.
- **ask** ti mantiene nel loop pur consentendo approvazioni rapide.
- Le allowlist per agente impediscono che le approvazioni di un agente si riversino negli altri.
- Le approvazioni si applicano solo alle richieste exec host provenienti da **mittenti autorizzati**. I mittenti non autorizzati non possono emettere `/exec`.
- `/exec security=full` è una comodità a livello di sessione per operatori autorizzati e per progettazione salta le approvazioni.
  Per bloccare rigidamente l'exec host, imposta la sicurezza delle approvazioni su `deny` o nega lo strumento `exec` tramite tool policy.

Correlati:

- [Strumento exec](/it/tools/exec)
- [Modalità elevated](/it/tools/elevated)
- [Skills](/it/tools/skills)

## Correlati

- [Exec](/it/tools/exec) — strumento di esecuzione di comandi shell
- [Sandboxing](/it/gateway/sandboxing) — modalità sandbox e accesso al workspace
- [Security](/it/gateway/security) — modello di sicurezza e hardening
- [Sandbox vs Tool Policy vs Elevated](/it/gateway/sandbox-vs-tool-policy-vs-elevated) — quando usare ciascuno
