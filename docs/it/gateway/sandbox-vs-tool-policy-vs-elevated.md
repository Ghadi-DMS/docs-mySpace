---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'Perché uno strumento è bloccato: runtime sandbox, criteri di allow/deny degli strumenti e gate exec elevati'
title: Sandbox vs criteri degli strumenti vs Elevated
x-i18n:
    generated_at: "2026-04-06T03:07:26Z"
    model: gpt-5.4
    provider: openai
    source_hash: 331f5b2f0d5effa1320125d9f29948e16d0deaffa59eb1e4f25a63481cbe22d6
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 15
---

# Sandbox vs criteri degli strumenti vs Elevated

OpenClaw ha tre controlli correlati, ma diversi:

1. **Sandbox** (`agents.defaults.sandbox.*` / `agents.list[].sandbox.*`) decide **dove vengono eseguiti gli strumenti** (Docker o host).
2. **Criteri degli strumenti** (`tools.*`, `tools.sandbox.tools.*`, `agents.list[].tools.*`) decidono **quali strumenti sono disponibili/consentiti**.
3. **Elevated** (`tools.elevated.*`, `agents.list[].tools.elevated.*`) è una **via di fuga solo per exec** per eseguire fuori dalla sandbox quando sei in sandbox (`gateway` per impostazione predefinita, oppure `node` quando il target exec è configurato su `node`).

## Debug rapido

Usa l'inspector per vedere cosa OpenClaw sta _effettivamente_ facendo:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

Stampa:

- modalità/ambito/accesso workspace effettivi della sandbox
- se la sessione è attualmente in sandbox (main vs non-main)
- allow/deny effettivi degli strumenti nella sandbox (e se provengono da agent/global/default)
- gate elevated e percorsi chiave per la correzione

## Sandbox: dove vengono eseguiti gli strumenti

La sandbox è controllata da `agents.defaults.sandbox.mode`:

- `"off"`: tutto viene eseguito sull'host.
- `"non-main"`: solo le sessioni non-main sono in sandbox (la comune “sorpresa” per gruppi/canali).
- `"all"`: tutto è in sandbox.

Vedi [Sandboxing](/it/gateway/sandboxing) per la matrice completa (ambito, mount del workspace, immagini).

### Bind mount (controllo rapido di sicurezza)

- `docker.binds` _buca_ il filesystem della sandbox: tutto ciò che monti è visibile all'interno del container con la modalità che imposti (`:ro` o `:rw`).
- Il valore predefinito è lettura-scrittura se ometti la modalità; preferisci `:ro` per codice sorgente/segreti.
- `scope: "shared"` ignora i bind per-agent (si applicano solo i bind globali).
- OpenClaw convalida le sorgenti bind due volte: prima sul percorso sorgente normalizzato, poi di nuovo dopo la risoluzione tramite l'antenato esistente più profondo. Le evasioni tramite symlink-parent non aggirano i controlli dei percorsi bloccati o delle radici consentite.
- I percorsi leaf inesistenti vengono comunque controllati in modo sicuro. Se `/workspace/alias-out/new-file` viene risolto tramite un parent con symlink verso un percorso bloccato o fuori dalle radici consentite configurate, il bind viene rifiutato.
- Fare bind di `/var/run/docker.sock` consegna di fatto il controllo dell'host alla sandbox; fallo solo intenzionalmente.
- L'accesso al workspace (`workspaceAccess: "ro"`/`"rw"`) è indipendente dalle modalità di bind.

## Criteri degli strumenti: quali strumenti esistono/sono invocabili

Contano due livelli:

- **Profilo strumenti**: `tools.profile` e `agents.list[].tools.profile` (allowlist di base)
- **Profilo strumenti del provider**: `tools.byProvider[provider].profile` e `agents.list[].tools.byProvider[provider].profile`
- **Criteri strumenti globali/per-agent**: `tools.allow`/`tools.deny` e `agents.list[].tools.allow`/`agents.list[].tools.deny`
- **Criteri strumenti del provider**: `tools.byProvider[provider].allow/deny` e `agents.list[].tools.byProvider[provider].allow/deny`
- **Criteri strumenti della sandbox** (si applicano solo in sandbox): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` e `agents.list[].tools.sandbox.tools.*`

Regole pratiche:

- `deny` vince sempre.
- Se `allow` non è vuoto, tutto il resto viene trattato come bloccato.
- I criteri degli strumenti sono il blocco rigido: `/exec` non può aggirare uno strumento `exec` negato.
- `/exec` cambia solo i valori predefiniti della sessione per i mittenti autorizzati; non concede accesso agli strumenti.
  Le chiavi degli strumenti del provider accettano sia `provider` (ad esempio `google-antigravity`) sia `provider/model` (ad esempio `openai/gpt-5.4`).

### Gruppi di strumenti (shorthand)

I criteri degli strumenti (globali, agent, sandbox) supportano voci `group:*` che si espandono in più strumenti:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

Gruppi disponibili:

- `group:runtime`: `exec`, `process`, `code_execution` (`bash` è accettato come
  alias di `exec`)
- `group:fs`: `read`, `write`, `edit`, `apply_patch`
- `group:sessions`: `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`
- `group:memory`: `memory_search`, `memory_get`
- `group:web`: `web_search`, `x_search`, `web_fetch`
- `group:ui`: `browser`, `canvas`
- `group:automation`: `cron`, `gateway`
- `group:messaging`: `message`
- `group:nodes`: `nodes`
- `group:agents`: `agents_list`
- `group:media`: `image`, `image_generate`, `video_generate`, `tts`
- `group:openclaw`: tutti gli strumenti OpenClaw integrati (esclude i plugin provider)

## Elevated: "esegui sull'host" solo per exec

Elevated **non** concede strumenti aggiuntivi; influisce solo su `exec`.

- Se sei in sandbox, `/elevated on` (o `exec` con `elevated: true`) viene eseguito fuori dalla sandbox (le approvazioni possono comunque applicarsi).
- Usa `/elevated full` per saltare le approvazioni exec per la sessione.
- Se stai già eseguendo direttamente, elevated è di fatto un no-op (comunque soggetto a gate).
- Elevated **non** è limitato allo scope delle Skills e **non** sovrascrive allow/deny degli strumenti.
- Elevated non concede override arbitrari cross-host da `host=auto`; segue le normali regole del target exec e preserva `node` solo quando il target configurato/di sessione è già `node`.
- `/exec` è separato da elevated. Regola solo i valori predefiniti exec per sessione per i mittenti autorizzati.

Gate:

- Abilitazione: `tools.elevated.enabled` (e facoltativamente `agents.list[].tools.elevated.enabled`)
- Allowlist mittenti: `tools.elevated.allowFrom.<provider>` (e facoltativamente `agents.list[].tools.elevated.allowFrom.<provider>`)

Vedi [Elevated Mode](/it/tools/elevated).

## Correzioni comuni per la "prigione della sandbox"

### "Lo strumento X è bloccato dai criteri degli strumenti della sandbox"

Chiavi di correzione (scegline una):

- Disabilita la sandbox: `agents.defaults.sandbox.mode=off` (o per-agent `agents.list[].sandbox.mode=off`)
- Consenti lo strumento all'interno della sandbox:
  - rimuovilo da `tools.sandbox.tools.deny` (o per-agent `agents.list[].tools.sandbox.tools.deny`)
  - oppure aggiungilo a `tools.sandbox.tools.allow` (o all'allow per-agent)

### "Pensavo che fosse main, perché è in sandbox?"

In modalità `"non-main"`, le chiavi gruppo/canale _non_ sono main. Usa la chiave della sessione main (mostrata da `sandbox explain`) oppure cambia la modalità in `"off"`.

## Vedi anche

- [Sandboxing](/it/gateway/sandboxing) -- riferimento completo della sandbox (modalità, ambiti, backend, immagini)
- [Sandbox e strumenti multi-agent](/it/tools/multi-agent-sandbox-tools) -- override per-agent e precedenza
- [Elevated Mode](/it/tools/elevated)
