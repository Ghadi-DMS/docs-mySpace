---
read_when:
    - Installazione o configurazione dei plugin
    - Comprensione del rilevamento dei plugin e delle regole di caricamento
    - Lavoro con bundle di plugin compatibili con Codex/Claude
sidebarTitle: Install and Configure
summary: Installa, configura e gestisci i plugin OpenClaw
title: Plugin
x-i18n:
    generated_at: "2026-04-06T03:13:21Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e2472a3023f3c1c6ee05b0cdc228f6b713cc226a08695b327de8a3ad6973c83
    source_path: tools/plugin.md
    workflow: 15
---

# Plugin

I plugin estendono OpenClaw con nuove capability: canali, provider di modelli,
strumenti, Skills, speech, trascrizione realtime, voce realtime,
media-understanding, generazione di immagini, generazione video, web fetch, web
search e altro ancora. Alcuni plugin sono **core** (forniti con OpenClaw), altri
sono **esterni** (pubblicati su npm dalla community).

## Avvio rapido

<Steps>
  <Step title="Vedi cosa è caricato">
    ```bash
    openclaw plugins list
    ```
  </Step>

  <Step title="Installa un plugin">
    ```bash
    # Da npm
    openclaw plugins install @openclaw/voice-call

    # Da una directory o archivio locale
    openclaw plugins install ./my-plugin
    openclaw plugins install ./my-plugin.tgz
    ```

  </Step>

  <Step title="Riavvia il Gateway">
    ```bash
    openclaw gateway restart
    ```

    Poi configura sotto `plugins.entries.\<id\>.config` nel tuo file di configurazione.

  </Step>
</Steps>

Se preferisci il controllo nativo in chat, abilita `commands.plugins: true` e usa:

```text
/plugin install clawhub:@openclaw/voice-call
/plugin show voice-call
/plugin enable voice-call
```

Il percorso di installazione usa lo stesso resolver della CLI: percorso/archivio locale, `clawhub:<pkg>` esplicito o specifica di pacchetto bare (prima ClawHub, poi fallback su npm).

Se la configurazione non è valida, normalmente l'installazione fallisce in modalità fail-closed e ti indirizza a
`openclaw doctor --fix`. L'unica eccezione di ripristino è un percorso ristretto di
reinstallazione di plugin bundled per plugin che scelgono di usare
`openclaw.install.allowInvalidConfigRecovery`.

## Tipi di plugin

OpenClaw riconosce due formati di plugin:

| Formato    | Come funziona                                                   | Esempi                                                 |
| ---------- | --------------------------------------------------------------- | ------------------------------------------------------ |
| **Nativo** | `openclaw.plugin.json` + modulo runtime; viene eseguito in-process | Plugin ufficiali, pacchetti npm della community     |
| **Bundle** | Layout compatibile con Codex/Claude/Cursor; mappato alle funzionalità OpenClaw | `.codex-plugin/`, `.claude-plugin/`, `.cursor-plugin/` |

Entrambi compaiono sotto `openclaw plugins list`. Vedi [Bundle di plugin](/it/plugins/bundles) per i dettagli sui bundle.

Se stai scrivendo un plugin nativo, inizia con [Creazione di plugin](/it/plugins/building-plugins)
e la [Panoramica del Plugin SDK](/it/plugins/sdk-overview).

## Plugin ufficiali

### Installabili (npm)

| Plugin          | Pacchetto              | Documentazione                       |
| --------------- | ---------------------- | ------------------------------------ |
| Matrix          | `@openclaw/matrix`     | [Matrix](/it/channels/matrix)           |
| Microsoft Teams | `@openclaw/msteams`    | [Microsoft Teams](/it/channels/msteams) |
| Nostr           | `@openclaw/nostr`      | [Nostr](/it/channels/nostr)             |
| Voice Call      | `@openclaw/voice-call` | [Voice Call](/it/plugins/voice-call)    |
| Zalo            | `@openclaw/zalo`       | [Zalo](/it/channels/zalo)               |
| Zalo Personal   | `@openclaw/zalouser`   | [Zalo Personal](/it/plugins/zalouser)   |

### Core (forniti con OpenClaw)

<AccordionGroup>
  <Accordion title="Provider di modelli (abilitati per impostazione predefinita)">
    `anthropic`, `byteplus`, `cloudflare-ai-gateway`, `github-copilot`, `google`,
    `huggingface`, `kilocode`, `kimi-coding`, `minimax`, `mistral`, `qwen`,
    `moonshot`, `nvidia`, `openai`, `opencode`, `opencode-go`, `openrouter`,
    `qianfan`, `synthetic`, `together`, `venice`,
    `vercel-ai-gateway`, `volcengine`, `xiaomi`, `zai`
  </Accordion>

  <Accordion title="Plugin memory">
    - `memory-core` — memory search bundled (predefinito tramite `plugins.slots.memory`)
    - `memory-lancedb` — memory a lungo termine installata on-demand con auto-recall/capture (imposta `plugins.slots.memory = "memory-lancedb"`)
  </Accordion>

  <Accordion title="Provider speech (abilitati per impostazione predefinita)">
    `elevenlabs`, `microsoft`
  </Accordion>

  <Accordion title="Altro">
    - `browser` — plugin browser bundled per lo strumento browser, la CLI `openclaw browser`, il metodo gateway `browser.request`, il runtime browser e il servizio di controllo browser predefinito (abilitato per impostazione predefinita; disabilitalo prima di sostituirlo)
    - `copilot-proxy` — bridge VS Code Copilot Proxy (disabilitato per impostazione predefinita)
  </Accordion>
</AccordionGroup>

Cerchi plugin di terze parti? Vedi [Plugin della community](/it/plugins/community).

## Configurazione

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-extension"] },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

| Campo           | Descrizione                                            |
| ---------------- | ------------------------------------------------------ |
| `enabled`        | Interruttore principale (predefinito: `true`)         |
| `allow`          | Allowlist dei plugin (facoltativa)                    |
| `deny`           | Denylist dei plugin (facoltativa; deny ha la precedenza) |
| `load.paths`     | File/directory di plugin aggiuntivi                   |
| `slots`          | Selettori di slot esclusivi (ad es. `memory`, `contextEngine`) |
| `entries.\<id\>` | Toggle + configurazione per plugin                    |

Le modifiche di configurazione **richiedono un riavvio del gateway**. Se il Gateway è in esecuzione con
config watch + riavvio in-process abilitato (il percorso predefinito `openclaw gateway`),
quel riavvio di solito viene eseguito automaticamente poco dopo la scrittura della configurazione.

<Accordion title="Stati del plugin: disabilitato vs mancante vs non valido">
  - **Disabilitato**: il plugin esiste ma le regole di abilitazione lo hanno disattivato. La configurazione viene preservata.
  - **Mancante**: la configurazione fa riferimento a un ID plugin che il rilevamento non ha trovato.
  - **Non valido**: il plugin esiste ma la sua configurazione non corrisponde allo schema dichiarato.
</Accordion>

## Rilevamento e precedenza

OpenClaw analizza i plugin in questo ordine (vince la prima corrispondenza):

<Steps>
  <Step title="Percorsi di configurazione">
    `plugins.load.paths` — percorsi espliciti di file o directory.
  </Step>

  <Step title="Estensioni del workspace">
    `\<workspace\>/.openclaw/<plugin-root>/*.ts` e `\<workspace\>/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="Estensioni globali">
    `~/.openclaw/<plugin-root>/*.ts` e `~/.openclaw/<plugin-root>/*/index.ts`.
  </Step>

  <Step title="Plugin bundled">
    Forniti con OpenClaw. Molti sono abilitati per impostazione predefinita (provider di modelli, speech).
    Altri richiedono un'abilitazione esplicita.
  </Step>
</Steps>

### Regole di abilitazione

- `plugins.enabled: false` disabilita tutti i plugin
- `plugins.deny` ha sempre la precedenza su allow
- `plugins.entries.\<id\>.enabled: false` disabilita quel plugin
- I plugin originati dal workspace sono **disabilitati per impostazione predefinita** (devono essere esplicitamente abilitati)
- I plugin bundled seguono l'insieme built-in abilitato per impostazione predefinita, salvo override
- Gli slot esclusivi possono forzare l'abilitazione del plugin selezionato per quello slot

## Slot dei plugin (categorie esclusive)

Alcune categorie sono esclusive (una sola attiva alla volta):

```json5
{
  plugins: {
    slots: {
      memory: "memory-core", // oppure "none" per disabilitare
      contextEngine: "legacy", // oppure un ID plugin
    },
  },
}
```

| Slot            | Cosa controlla         | Predefinito         |
| --------------- | ---------------------- | ------------------- |
| `memory`        | Plugin memory attivo   | `memory-core`       |
| `contextEngine` | Context engine attivo  | `legacy` (built-in) |

## Riferimento CLI

```bash
openclaw plugins list                       # inventario compatto
openclaw plugins list --enabled            # solo plugin caricati
openclaw plugins list --verbose            # righe di dettaglio per plugin
openclaw plugins list --json               # inventario leggibile dalla macchina
openclaw plugins inspect <id>              # dettaglio approfondito
openclaw plugins inspect <id> --json       # leggibile dalla macchina
openclaw plugins inspect --all             # tabella dell'intera flotta
openclaw plugins info <id>                 # alias di inspect
openclaw plugins doctor                    # diagnostica

openclaw plugins install <package>         # installa (prima ClawHub, poi npm)
openclaw plugins install clawhub:<pkg>     # installa solo da ClawHub
openclaw plugins install <spec> --force    # sovrascrive l'installazione esistente
openclaw plugins install <path>            # installa da percorso locale
openclaw plugins install -l <path>         # collega (senza copia) per sviluppo
openclaw plugins install <plugin> --marketplace <source>
openclaw plugins install <plugin> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <spec> --pin      # registra la specifica npm esatta risolta
openclaw plugins install <spec> --dangerously-force-unsafe-install
openclaw plugins update <id>             # aggiorna un plugin
openclaw plugins update <id> --dangerously-force-unsafe-install
openclaw plugins update --all            # aggiorna tutti
openclaw plugins uninstall <id>          # rimuove record di configurazione/installazione
openclaw plugins uninstall <id> --keep-files
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json

openclaw plugins enable <id>
openclaw plugins disable <id>
```

I plugin bundled vengono forniti con OpenClaw. Molti sono abilitati per impostazione predefinita (per esempio
provider di modelli bundled, provider speech bundled e il plugin browser
bundled). Altri plugin bundled richiedono comunque `openclaw plugins enable <id>`.

`--force` sovrascrive sul posto un plugin installato o hook pack esistente.
Non è supportato con `--link`, che riutilizza il percorso sorgente invece di
copiare su una destinazione di installazione gestita.

`--pin` è solo per npm. Non è supportato con `--marketplace`, perché
le installazioni da marketplace persistono i metadati della sorgente del marketplace invece di una specifica npm.

`--dangerously-force-unsafe-install` è un override di emergenza per falsi
positivi dallo scanner integrato di codice pericoloso. Consente ai flussi di installazione e aggiornamento
dei plugin di continuare oltre i rilevamenti integrati `critical`, ma non
aggira comunque i blocchi di policy `before_install` del plugin o il blocco per errori di scansione.

Questo flag CLI si applica solo ai flussi di installazione/aggiornamento dei plugin. Le installazioni di dipendenze
Skills supportate dal Gateway usano invece l'override della richiesta corrispondente `dangerouslyForceUnsafeInstall`, mentre `openclaw skills install` resta il flusso separato di download/installazione Skills da ClawHub.

I bundle compatibili partecipano allo stesso flusso di lista/inspect/enable/disable dei plugin. Il supporto runtime attuale include Skills bundle, command-Skills Claude,
valori predefiniti di Claude `settings.json`, valori predefiniti di Claude `.lsp.json` e `lspServers` dichiarati nel manifest, command-Skills Cursor e directory hook Codex compatibili.

`openclaw plugins inspect <id>` segnala anche le capability di bundle rilevate più le voci di server MCP e LSP supportate o non supportate per i plugin supportati da bundle.

Le sorgenti marketplace possono essere un nome noto di marketplace Claude da
`~/.claude/plugins/known_marketplaces.json`, una root locale di marketplace o un percorso
`marketplace.json`, una scorciatoia GitHub come `owner/repo`, un URL di repo GitHub
o un URL git. Per i marketplace remoti, le voci del plugin devono rimanere all'interno del
repo marketplace clonato e usare solo sorgenti di percorso relative.

Vedi [riferimento CLI `openclaw plugins`](/cli/plugins) per tutti i dettagli.

## Panoramica dell'API plugin

I plugin nativi esportano un oggetto entry che espone `register(api)`. I plugin più vecchi
possono ancora usare `activate(api)` come alias legacy, ma i nuovi plugin dovrebbero
usare `register`.

```typescript
export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  register(api) {
    api.registerProvider({
      /* ... */
    });
    api.registerTool({
      /* ... */
    });
    api.registerChannel({
      /* ... */
    });
  },
});
```

OpenClaw carica l'oggetto entry e chiama `register(api)` durante l'attivazione
del plugin. Il loader continua a usare `activate(api)` come fallback per i plugin più vecchi,
ma i plugin bundled e i nuovi plugin esterni dovrebbero considerare `register` come il
contratto pubblico.

Metodi comuni di registrazione:

| Metodo                                  | Cosa registra              |
| --------------------------------------- | -------------------------- |
| `registerProvider`                      | Provider di modelli (LLM)  |
| `registerChannel`                       | Canale chat                |
| `registerTool`                          | Strumento agente           |
| `registerHook` / `on(...)`              | Hook del ciclo di vita     |
| `registerSpeechProvider`                | Text-to-speech / STT       |
| `registerRealtimeTranscriptionProvider` | STT in streaming           |
| `registerRealtimeVoiceProvider`         | Voce realtime duplex       |
| `registerMediaUnderstandingProvider`    | Analisi immagini/audio     |
| `registerImageGenerationProvider`       | Generazione di immagini    |
| `registerMusicGenerationProvider`       | Generazione musicale       |
| `registerVideoGenerationProvider`       | Generazione video          |
| `registerWebFetchProvider`              | Provider web fetch / scrape |
| `registerWebSearchProvider`             | Web search                 |
| `registerHttpRoute`                     | Endpoint HTTP              |
| `registerCommand` / `registerCli`       | Comandi CLI                |
| `registerContextEngine`                 | Context engine             |
| `registerService`                       | Servizio in background     |

Comportamento delle guard hook per gli hook tipizzati del ciclo di vita:

- `before_tool_call`: `{ block: true }` è terminale; gli handler a priorità inferiore vengono saltati.
- `before_tool_call`: `{ block: false }` è un no-op e non annulla un blocco precedente.
- `before_install`: `{ block: true }` è terminale; gli handler a priorità inferiore vengono saltati.
- `before_install`: `{ block: false }` è un no-op e non annulla un blocco precedente.
- `message_sending`: `{ cancel: true }` è terminale; gli handler a priorità inferiore vengono saltati.
- `message_sending`: `{ cancel: false }` è un no-op e non annulla una cancellazione precedente.

Per il comportamento completo degli hook tipizzati, vedi [Panoramica SDK](/it/plugins/sdk-overview#hook-decision-semantics).

## Correlati

- [Creazione di plugin](/it/plugins/building-plugins) — crea il tuo plugin
- [Bundle di plugin](/it/plugins/bundles) — compatibilità con bundle Codex/Claude/Cursor
- [Manifesto plugin](/it/plugins/manifest) — schema del manifesto
- [Registrazione degli strumenti](/it/plugins/building-plugins#registering-agent-tools) — aggiungi strumenti agente in un plugin
- [Interni dei plugin](/it/plugins/architecture) — modello delle capability e pipeline di caricamento
- [Plugin della community](/it/plugins/community) — elenchi di terze parti
