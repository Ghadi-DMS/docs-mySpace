---
read_when:
    - Uso o modifica dello strumento exec
    - Debug del comportamento stdin o TTY
summary: Uso dello strumento exec, modalità stdin e supporto TTY
title: Strumento Exec
x-i18n:
    generated_at: "2026-04-06T03:12:58Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28388971c627292dba9bf65ae38d7af8cde49a33bb3b5fc8b20da4f0e350bedd
    source_path: tools/exec.md
    workflow: 15
---

# Strumento Exec

Esegui comandi shell nel workspace. Supporta l'esecuzione in foreground + background tramite `process`.
Se `process` non è consentito, `exec` viene eseguito in modo sincrono e ignora `yieldMs`/`background`.
Le sessioni in background hanno scope per agent; `process` vede solo le sessioni dello stesso agent.

## Parametri

- `command` (obbligatorio)
- `workdir` (predefinito: cwd)
- `env` (override chiave/valore)
- `yieldMs` (predefinito 10000): passaggio automatico in background dopo il ritardo
- `background` (bool): passa subito in background
- `timeout` (secondi, predefinito 1800): termina alla scadenza
- `pty` (bool): esegue in uno pseudo-terminale quando disponibile (CLI solo TTY, coding agent, terminal UI)
- `host` (`auto | sandbox | gateway | node`): dove eseguire
- `security` (`deny | allowlist | full`): modalità di enforcement per `gateway`/`node`
- `ask` (`off | on-miss | always`): prompt di approvazione per `gateway`/`node`
- `node` (string): id/nome node per `host=node`
- `elevated` (bool): richiede la modalità elevated (esce dalla sandbox verso il percorso host configurato); `security=full` viene forzato solo quando elevated si risolve in `full`

Note:

- `host` ha come predefinito `auto`: sandbox quando il runtime sandbox è attivo per la sessione, altrimenti gateway.
- `auto` è la strategia di instradamento predefinita, non un wildcard. `host=node` per chiamata è consentito da `auto`; `host=gateway` per chiamata è consentito solo quando nessun runtime sandbox è attivo.
- Senza configurazioni aggiuntive, `host=auto` continua a “funzionare e basta”: senza sandbox si risolve in `gateway`; con una sandbox attiva resta nella sandbox.
- `elevated` esce dalla sandbox verso il percorso host configurato: `gateway` per impostazione predefinita, oppure `node` quando `tools.exec.host=node` (o il valore predefinito della sessione è `host=node`). È disponibile solo quando l'accesso elevated è abilitato per la sessione/provider corrente.
- Le approvazioni `gateway`/`node` sono controllate da `~/.openclaw/exec-approvals.json`.
- `node` richiede un node associato (app companion o host node headless).
- Se sono disponibili più node, imposta `exec.node` o `tools.exec.node` per selezionarne uno.
- `exec host=node` è l'unico percorso di esecuzione shell per i node; il wrapper legacy `nodes.run` è stato rimosso.
- Sugli host non Windows, exec usa `SHELL` quando impostato; se `SHELL` è `fish`, preferisce `bash` (o `sh`)
  da `PATH` per evitare script incompatibili con fish, poi usa `SHELL` come fallback se nessuno dei due esiste.
- Sugli host Windows, exec preferisce il rilevamento di PowerShell 7 (`pwsh`) (Program Files, ProgramW6432, poi PATH),
  quindi usa Windows PowerShell 5.1 come fallback.
- L'esecuzione host (`gateway`/`node`) rifiuta `env.PATH` e gli override del loader (`LD_*`/`DYLD_*`) per
  prevenire hijacking dei binari o codice iniettato.
- OpenClaw imposta `OPENCLAW_SHELL=exec` nell'ambiente del comando generato (inclusi PTY ed esecuzione in sandbox) così le regole di shell/profilo possono rilevare il contesto dello strumento exec.
- Importante: la sandbox è **disattivata per impostazione predefinita**. Se la sandbox è disattivata, `host=auto`
  implicito si risolve in `gateway`. `host=sandbox` esplicito continua invece a fallire in modo chiuso invece di
  eseguire silenziosamente sull'host gateway. Abilita la sandbox o usa `host=gateway` con approvazioni.
- I controlli preflight degli script (per errori comuni di sintassi shell Python/Node) ispezionano solo i file all'interno del
  confine effettivo di `workdir`. Se il percorso di uno script si risolve fuori da `workdir`, il preflight viene saltato per
  quel file.
- Per lavori di lunga durata che iniziano subito, avviali una volta e affidati al
  risveglio automatico al completamento quando è abilitato e il comando emette output o fallisce.
  Usa `process` per log, stato, input o interventi; non emulare la
  pianificazione con loop di sleep, loop di timeout o polling ripetuto.
- Per lavori che devono avvenire più tardi o secondo una pianificazione, usa cron invece di
  pattern sleep/delay con `exec`.

## Configurazione

- `tools.exec.notifyOnExit` (predefinito: true): quando true, le sessioni exec passate in background accodano un evento di sistema e richiedono un heartbeat all'uscita.
- `tools.exec.approvalRunningNoticeMs` (predefinito: 10000): emette un singolo avviso “running” quando un exec protetto da approvazione dura più di questo valore (0 disabilita).
- `tools.exec.host` (predefinito: `auto`; si risolve in `sandbox` quando il runtime sandbox è attivo, altrimenti in `gateway`)
- `tools.exec.security` (predefinito: `deny` per sandbox, `full` per gateway + node quando non impostato)
- `tools.exec.ask` (predefinito: `off`)
- L'exec host senza approvazione è il comportamento predefinito per gateway + node. Se vuoi il comportamento con approvazioni/allowlist, restringi sia `tools.exec.*` sia il criterio host in `~/.openclaw/exec-approvals.json`; vedi [Approvazioni exec](/it/tools/exec-approvals#no-approval-yolo-mode).
- YOLO deriva dai valori predefiniti del criterio host (`security=full`, `ask=off`), non da `host=auto`. Se vuoi forzare l'instradamento gateway o node, imposta `tools.exec.host` o usa `/exec host=...`.
- In modalità `security=full` più `ask=off`, l'exec host segue direttamente il criterio configurato; non esiste un prefiltro euristico aggiuntivo per l'offuscamento dei comandi.
- `tools.exec.node` (predefinito: non impostato)
- `tools.exec.strictInlineEval` (predefinito: false): quando true, le forme inline di eval degli interpreti come `python -c`, `node -e`, `ruby -e`, `perl -e`, `php -r`, `lua -e` e `osascript -e` richiedono sempre approvazione esplicita. `allow-always` può comunque rendere persistenti invocazioni innocue di interpreti/script, ma le forme inline-eval continuano a chiedere conferma ogni volta.
- `tools.exec.pathPrepend`: elenco di directory da anteporre a `PATH` per le esecuzioni exec (solo gateway + sandbox).
- `tools.exec.safeBins`: binari sicuri solo-stdin che possono essere eseguiti senza voci esplicite nell'allowlist. Per i dettagli del comportamento, vedi [Safe bins](/it/tools/exec-approvals#safe-bins-stdin-only).
- `tools.exec.safeBinTrustedDirs`: directory esplicite aggiuntive considerate attendibili per i controlli del percorso degli eseguibili in `safeBins`. Le voci di `PATH` non sono mai considerate attendibili automaticamente. I valori predefiniti integrati sono `/bin` e `/usr/bin`.
- `tools.exec.safeBinProfiles`: criteri argv personalizzati facoltativi per safe bin (`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`).

Esempio:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### Gestione di PATH

- `host=gateway`: unisce il tuo `PATH` della login shell nell'ambiente exec. Gli override di `env.PATH` vengono
  rifiutati per l'esecuzione host. Il demone stesso continua comunque a essere eseguito con un `PATH` minimo:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
- `host=sandbox`: esegue `sh -lc` (login shell) all'interno del container, quindi `/etc/profile` può reimpostare `PATH`.
  OpenClaw antepone `env.PATH` dopo il sourcing del profilo tramite una variabile env interna (senza interpolazione della shell);
  qui si applica anche `tools.exec.pathPrepend`.
- `host=node`: solo gli override env non bloccati che passi vengono inviati al node. Gli override di `env.PATH` vengono
  rifiutati per l'esecuzione host e ignorati dagli host node. Se hai bisogno di voci PATH aggiuntive su un node,
  configura l'ambiente del servizio host node (systemd/launchd) oppure installa gli strumenti in posizioni standard.

Binding per-agent del node (usa l'indice dell'elenco agent nella configurazione):

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

Control UI: la scheda Nodes include un piccolo pannello “Exec node binding” per le stesse impostazioni.

## Override di sessione (`/exec`)

Usa `/exec` per impostare valori predefiniti **per sessione** per `host`, `security`, `ask` e `node`.
Invia `/exec` senza argomenti per mostrare i valori correnti.

Esempio:

```
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

## Modello di autorizzazione

`/exec` viene rispettato solo per **mittenti autorizzati** (allowlist dei canali/abbinamento più `commands.useAccessGroups`).
Aggiorna **solo lo stato della sessione** e non scrive la configurazione. Per disabilitare completamente exec, negalo tramite i
criteri degli strumenti (`tools.deny: ["exec"]` o per-agent). Le approvazioni host continuano comunque ad applicarsi a meno che tu non imposti esplicitamente
`security=full` e `ask=off`.

## Approvazioni exec (app companion / host node)

Gli agent in sandbox possono richiedere un'approvazione per richiesta prima che `exec` venga eseguito sull'host gateway o node.
Vedi [Approvazioni exec](/it/tools/exec-approvals) per criteri, allowlist e flusso UI.

Quando sono richieste approvazioni, lo strumento exec restituisce subito
`status: "approval-pending"` e un id di approvazione. Una volta approvato (o negato / scaduto),
il Gateway emette eventi di sistema (`Exec finished` / `Exec denied`). Se il comando è ancora
in esecuzione dopo `tools.exec.approvalRunningNoticeMs`, viene emesso un singolo avviso `Exec running`.
Nei canali con card/pulsanti di approvazione nativi, l'agent dovrebbe affidarsi prima a
quell'UI nativa e includere un comando manuale `/approve` solo quando il
risultato dello strumento dice esplicitamente che le approvazioni in chat non sono disponibili o che il
percorso manuale è l'unico possibile.

## Allowlist + safe bins

L'enforcement manuale dell'allowlist corrisponde **solo ai percorsi binari risolti** (nessuna corrispondenza per basename). Quando
`security=allowlist`, i comandi shell sono consentiti automaticamente solo se ogni segmento della pipeline è
in allowlist oppure è un safe bin. Il chaining (`;`, `&&`, `||`) e i reindirizzamenti vengono rifiutati in
modalità allowlist a meno che ogni segmento di livello superiore soddisfi l'allowlist (inclusi i safe bin).
I reindirizzamenti continuano a non essere supportati.
La fiducia persistente `allow-always` non aggira questa regola: un comando concatenato richiede comunque che ogni
segmento di livello superiore corrisponda.

`autoAllowSkills` è un percorso di praticità separato nelle approvazioni exec. Non è la stessa cosa delle
voci manuali dell'allowlist dei percorsi. Per una fiducia esplicita rigorosa, mantieni `autoAllowSkills` disabilitato.

Usa i due controlli per scopi diversi:

- `tools.exec.safeBins`: piccoli filtri stream solo-stdin.
- `tools.exec.safeBinTrustedDirs`: directory esplicite aggiuntive attendibili per i percorsi eseguibili dei safe bin.
- `tools.exec.safeBinProfiles`: criteri argv espliciti per safe bin personalizzati.
- allowlist: fiducia esplicita per i percorsi eseguibili.

Non trattare `safeBins` come un'allowlist generica e non aggiungere binari interprete/runtime (ad esempio `python3`, `node`, `ruby`, `bash`). Se ti servono, usa voci esplicite nell'allowlist e mantieni abilitati i prompt di approvazione.
`openclaw security audit` avvisa quando nelle voci `safeBins` degli interpreti/runtime mancano profili espliciti, e `openclaw doctor --fix` può creare lo scaffold per le voci personalizzate mancanti in `safeBinProfiles`.
`openclaw security audit` e `openclaw doctor` avvisano anche quando aggiungi esplicitamente di nuovo in `safeBins` binari dal comportamento ampio come `jq`.
Se aggiungi esplicitamente interpreti all'allowlist, abilita `tools.exec.strictInlineEval` così le forme inline di code-eval richiedono comunque una nuova approvazione.

Per i dettagli completi dei criteri e gli esempi, vedi [Approvazioni exec](/it/tools/exec-approvals#safe-bins-stdin-only) e [Safe bins versus allowlist](/it/tools/exec-approvals#safe-bins-versus-allowlist).

## Esempi

Foreground:

```json
{ "tool": "exec", "command": "ls -la" }
```

Background + poll:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

Il polling serve per lo stato on-demand, non per loop di attesa. Se il risveglio automatico al completamento
è abilitato, il comando può risvegliare la sessione quando emette output o fallisce.

Invio tasti (stile tmux):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Invio (manda solo CR):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Incolla (delimitato di default):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` è un sotto-strumento di `exec` per modifiche strutturate a più file.
È abilitato per impostazione predefinita per i modelli OpenAI e OpenAI Codex. Usa la configurazione solo
quando vuoi disabilitarlo o limitarlo a modelli specifici:

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.4"] },
    },
  },
}
```

Note:

- Disponibile solo per i modelli OpenAI/OpenAI Codex.
- Continuano ad applicarsi i criteri degli strumenti; `allow: ["write"]` consente implicitamente `apply_patch`.
- La configurazione si trova in `tools.exec.applyPatch`.
- `tools.exec.applyPatch.enabled` ha come predefinito `true`; impostalo su `false` per disabilitare lo strumento per i modelli OpenAI.
- `tools.exec.applyPatch.workspaceOnly` ha come predefinito `true` (limitato al workspace). Impostalo su `false` solo se vuoi intenzionalmente che `apply_patch` scriva/elimini fuori dalla directory del workspace.

## Correlati

- [Approvazioni exec](/it/tools/exec-approvals) — gate di approvazione per i comandi shell
- [Sandboxing](/it/gateway/sandboxing) — esecuzione di comandi in ambienti sandbox
- [Processo in background](/it/gateway/background-process) — exec di lunga durata e strumento process
- [Sicurezza](/it/gateway/security) — criteri degli strumenti e accesso elevated
