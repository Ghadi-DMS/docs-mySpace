---
read_when:
    - Stai creando un plugin OpenClaw
    - Devi distribuire uno schema di configurazione del plugin o eseguire il debug degli errori di convalida del plugin
summary: Manifest del plugin + requisiti dello schema JSON (convalida rigorosa della configurazione)
title: Manifest del plugin
x-i18n:
    generated_at: "2026-04-06T03:09:33Z"
    model: gpt-5.4
    provider: openai
    source_hash: f6f915a761cdb5df77eba5d2ccd438c65445bd2ab41b0539d1200e63e8cf2c3a
    source_path: plugins/manifest.md
    workflow: 15
---

# Manifest del plugin (openclaw.plugin.json)

Questa pagina riguarda solo il **manifest nativo del plugin OpenClaw**.

Per i layout di bundle compatibili, consulta [Bundle di plugin](/it/plugins/bundles).

I formati di bundle compatibili usano file manifest diversi:

- Bundle Codex: `.codex-plugin/plugin.json`
- Bundle Claude: `.claude-plugin/plugin.json` oppure il layout predefinito del componente Claude
  senza manifest
- Bundle Cursor: `.cursor-plugin/plugin.json`

OpenClaw rileva automaticamente anche questi layout di bundle, ma non vengono convalidati
rispetto allo schema `openclaw.plugin.json` descritto qui.

Per i bundle compatibili, OpenClaw attualmente legge i metadati del bundle più le root Skills dichiarate, le root dei comandi Claude, i valori predefiniti di `settings.json` del bundle Claude,
i valori predefiniti LSP del bundle Claude e i pacchetti hook supportati quando il layout corrisponde
alle aspettative del runtime OpenClaw.

Ogni plugin OpenClaw nativo **deve** includere un file `openclaw.plugin.json` nella
**root del plugin**. OpenClaw usa questo manifest per convalidare la configurazione
**senza eseguire il codice del plugin**. I manifest mancanti o non validi sono trattati come
errori del plugin e bloccano la convalida della configurazione.

Consulta la guida completa del sistema plugin: [Plugins](/it/tools/plugin).
Per il modello di capability nativo e le indicazioni correnti sulla compatibilità esterna:
[Modello di capability](/it/plugins/architecture#public-capability-model).

## Cosa fa questo file

`openclaw.plugin.json` è il metadato che OpenClaw legge prima di caricare il
codice del tuo plugin.

Usalo per:

- identità del plugin
- convalida della configurazione
- metadati di autenticazione e onboarding che devono essere disponibili senza avviare il
  runtime del plugin
- metadati di alias e attivazione automatica che devono essere risolti prima del caricamento del runtime del plugin
- metadati abbreviati di proprietà della famiglia di modelli che devono attivare automaticamente il
  plugin prima del caricamento del runtime
- snapshot statiche di proprietà delle capability usate per il wiring di compatibilità bundled e
  la copertura dei contratti
- metadati di configurazione specifici del canale che devono essere uniti nelle superfici di catalogo e convalida
  senza caricare il runtime
- suggerimenti UI per la configurazione

Non usarlo per:

- registrare il comportamento del runtime
- dichiarare entrypoint di codice
- metadati di installazione npm

Questi appartengono al codice del plugin e a `package.json`.

## Esempio minimo

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Esempio completo

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "Plugin provider OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "providerAuthEnvVars": {
    "openrouter": ["OPENROUTER_API_KEY"]
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "Chiave API OpenRouter",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "Chiave API OpenRouter",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "Chiave API",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Riferimento dei campi di primo livello

| Campo                               | Obbligatorio | Tipo                             | Significato                                                                                                                                                                                                  |
| ----------------------------------- | ------------ | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                | Sì           | `string`                         | ID canonico del plugin. È l'ID usato in `plugins.entries.<id>`.                                                                                                                                              |
| `configSchema`                      | Sì           | `object`                         | JSON Schema inline per la configurazione di questo plugin.                                                                                                                                                    |
| `enabledByDefault`                  | No           | `true`                           | Contrassegna un plugin bundled come abilitato per impostazione predefinita. Omettilo, oppure imposta qualsiasi valore diverso da `true`, per lasciare il plugin disabilitato per impostazione predefinita. |
| `legacyPluginIds`                   | No           | `string[]`                       | ID legacy che vengono normalizzati a questo ID canonico del plugin.                                                                                                                                          |
| `autoEnableWhenConfiguredProviders` | No           | `string[]`                       | ID provider che devono abilitare automaticamente questo plugin quando autenticazione, configurazione o riferimenti ai modelli li menzionano.                                                                 |
| `kind`                              | No           | `"memory"` \| `"context-engine"` | Dichiara un tipo esclusivo di plugin usato da `plugins.slots.*`.                                                                                                                                             |
| `channels`                          | No           | `string[]`                       | ID dei canali posseduti da questo plugin. Usati per discovery e convalida della configurazione.                                                                                                              |
| `providers`                         | No           | `string[]`                       | ID dei provider posseduti da questo plugin.                                                                                                                                                                   |
| `modelSupport`                      | No           | `object`                         | Metadati abbreviati di famiglia di modelli posseduti dal manifest usati per caricare automaticamente il plugin prima del runtime.                                                                           |
| `providerAuthEnvVars`               | No           | `Record<string, string[]>`       | Metadati env economici per l'autenticazione del provider che OpenClaw può ispezionare senza caricare il codice del plugin.                                                                                  |
| `providerAuthChoices`               | No           | `object[]`                       | Metadati economici delle scelte di autenticazione per selettori di onboarding, risoluzione del provider preferito e semplice wiring dei flag CLI.                                                           |
| `contracts`                         | No           | `object`                         | Snapshot statica delle capability bundled per speech, realtime transcription, realtime voice, media-understanding, image-generation, music-generation, video-generation, web-fetch, web search e proprietà degli strumenti. |
| `channelConfigs`                    | No           | `Record<string, object>`         | Metadati di configurazione del canale posseduti dal manifest uniti nelle superfici di discovery e convalida prima del caricamento del runtime.                                                              |
| `skills`                            | No           | `string[]`                       | Directory Skills da caricare, relative alla root del plugin.                                                                                                                                                  |
| `name`                              | No           | `string`                         | Nome leggibile del plugin.                                                                                                                                                                                    |
| `description`                       | No           | `string`                         | Breve riepilogo mostrato nelle superfici del plugin.                                                                                                                                                         |
| `version`                           | No           | `string`                         | Versione informativa del plugin.                                                                                                                                                                              |
| `uiHints`                           | No           | `Record<string, object>`         | Etichette UI, placeholder e suggerimenti di sensibilità per i campi di configurazione.                                                                                                                       |

## Riferimento `providerAuthChoices`

Ogni voce `providerAuthChoices` descrive una scelta di onboarding o autenticazione.
OpenClaw la legge prima che il runtime del provider venga caricato.

| Campo                 | Obbligatorio | Tipo                                            | Significato                                                                                          |
| --------------------- | ------------ | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `provider`            | Sì           | `string`                                        | ID del provider a cui appartiene questa scelta.                                                      |
| `method`              | Sì           | `string`                                        | ID del metodo di autenticazione a cui instradare.                                                    |
| `choiceId`            | Sì           | `string`                                        | ID stabile della scelta di autenticazione usato dai flussi di onboarding e CLI.                     |
| `choiceLabel`         | No           | `string`                                        | Etichetta visibile all'utente. Se omessa, OpenClaw usa `choiceId` come fallback.                    |
| `choiceHint`          | No           | `string`                                        | Breve testo di supporto per il selettore.                                                            |
| `assistantPriority`   | No           | `number`                                        | I valori più bassi vengono ordinati prima nei selettori interattivi guidati dall'assistente.        |
| `assistantVisibility` | No           | `"visible"` \| `"manual-only"`                  | Nasconde la scelta dai selettori dell'assistente pur consentendo ancora la selezione manuale via CLI. |
| `deprecatedChoiceIds` | No           | `string[]`                                      | ID legacy di scelta che devono reindirizzare gli utenti a questa scelta sostitutiva.                |
| `groupId`             | No           | `string`                                        | ID gruppo opzionale per raggruppare scelte correlate.                                                |
| `groupLabel`          | No           | `string`                                        | Etichetta visibile all'utente per quel gruppo.                                                       |
| `groupHint`           | No           | `string`                                        | Breve testo di supporto per il gruppo.                                                               |
| `optionKey`           | No           | `string`                                        | Chiave opzione interna per flussi di autenticazione semplici a un solo flag.                         |
| `cliFlag`             | No           | `string`                                        | Nome del flag CLI, come `--openrouter-api-key`.                                                      |
| `cliOption`           | No           | `string`                                        | Forma completa dell'opzione CLI, come `--openrouter-api-key <key>`.                                  |
| `cliDescription`      | No           | `string`                                        | Descrizione usata nell'help CLI.                                                                     |
| `onboardingScopes`    | No           | `Array<"text-inference" \| "image-generation">` | Su quali superfici di onboarding deve comparire questa scelta. Se omesso, il valore predefinito è `["text-inference"]`. |

## Riferimento `uiHints`

`uiHints` è una mappa dai nomi dei campi di configurazione a piccoli suggerimenti di rendering.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "Chiave API",
      "help": "Usata per le richieste OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Ogni suggerimento di campo può includere:

| Campo         | Tipo       | Significato                               |
| ------------- | ---------- | ----------------------------------------- |
| `label`       | `string`   | Etichetta del campo visibile all'utente.  |
| `help`        | `string`   | Breve testo di supporto.                  |
| `tags`        | `string[]` | Tag UI opzionali.                         |
| `advanced`    | `boolean`  | Contrassegna il campo come avanzato.      |
| `sensitive`   | `boolean`  | Contrassegna il campo come secret o sensibile. |
| `placeholder` | `string`   | Testo placeholder per gli input del form. |

## Riferimento `contracts`

Usa `contracts` solo per metadati statici di proprietà delle capability che OpenClaw può
leggere senza importare il runtime del plugin.

```json
{
  "contracts": {
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "mediaUnderstandingProviders": ["openai", "openai-codex"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Ogni elenco è opzionale:

| Campo                            | Tipo       | Significato                                                  |
| -------------------------------- | ---------- | ------------------------------------------------------------ |
| `speechProviders`                | `string[]` | ID dei provider speech posseduti da questo plugin.           |
| `realtimeTranscriptionProviders` | `string[]` | ID dei provider di trascrizione realtime posseduti da questo plugin. |
| `realtimeVoiceProviders`         | `string[]` | ID dei provider voice realtime posseduti da questo plugin.   |
| `mediaUnderstandingProviders`    | `string[]` | ID dei provider di media-understanding posseduti da questo plugin. |
| `imageGenerationProviders`       | `string[]` | ID dei provider di image-generation posseduti da questo plugin. |
| `videoGenerationProviders`       | `string[]` | ID dei provider di video-generation posseduti da questo plugin. |
| `webFetchProviders`              | `string[]` | ID dei provider di web-fetch posseduti da questo plugin.     |
| `webSearchProviders`             | `string[]` | ID dei provider di web search posseduti da questo plugin.    |
| `tools`                          | `string[]` | Nomi degli strumenti agent posseduti da questo plugin per i controlli del contratto bundled. |

## Riferimento `channelConfigs`

Usa `channelConfigs` quando un plugin del canale necessita di metadati di configurazione economici prima
del caricamento del runtime.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "URL homeserver",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Connessione homeserver Matrix",
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Ogni voce canale può includere:

| Campo         | Tipo                     | Significato                                                                                  |
| ------------- | ------------------------ | -------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | JSON Schema per `channels.<id>`. Obbligatorio per ogni voce dichiarata di configurazione del canale. |
| `uiHints`     | `Record<string, object>` | Etichette UI/placeholder/suggerimenti di sensibilità opzionali per quella sezione di configurazione del canale. |
| `label`       | `string`                 | Etichetta del canale unita nelle superfici di selezione e ispezione quando i metadati del runtime non sono pronti. |
| `description` | `string`                 | Breve descrizione del canale per le superfici di ispezione e catalogo.                       |
| `preferOver`  | `string[]`               | ID di plugin legacy o a priorità inferiore che questo canale deve superare nelle superfici di selezione. |

## Riferimento `modelSupport`

Usa `modelSupport` quando OpenClaw deve dedurre il plugin del tuo provider da
ID di modello abbreviati come `gpt-5.4` o `claude-sonnet-4.6` prima del caricamento del runtime del plugin.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw applica questa precedenza:

- i riferimenti espliciti `provider/model` usano i metadati `providers` del manifest proprietario
- `modelPatterns` hanno la precedenza su `modelPrefixes`
- se un plugin non bundled e un plugin bundled corrispondono entrambi, vince il plugin non bundled
- l'ambiguità residua viene ignorata finché l'utente o la configurazione non specificano un provider

Campi:

| Campo           | Tipo       | Significato                                                                       |
| --------------- | ---------- | --------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Prefissi confrontati con `startsWith` rispetto agli ID di modello abbreviati.     |
| `modelPatterns` | `string[]` | Sorgenti regex confrontate con gli ID di modello abbreviati dopo la rimozione del suffisso del profilo. |

Le chiavi legacy di capability di primo livello sono deprecate. Usa `openclaw doctor --fix` per
spostare `speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`,
`webFetchProviders` e `webSearchProviders` sotto `contracts`; il normale
caricamento del manifest non tratta più quei campi di primo livello come
proprietà delle capability.

## Manifest rispetto a package.json

I due file hanno funzioni diverse:

| File                   | Usalo per                                                                                                                           |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Discovery, convalida della configurazione, metadati delle scelte di autenticazione e suggerimenti UI che devono esistere prima dell'esecuzione del codice del plugin |
| `package.json`         | Metadati npm, installazione delle dipendenze e blocco `openclaw` usato per entrypoint, gating dell'installazione, setup o metadati di catalogo |

Se non sei sicuro di dove debba stare un metadato, usa questa regola:

- se OpenClaw deve conoscerlo prima di caricare il codice del plugin, inseriscilo in `openclaw.plugin.json`
- se riguarda packaging, file di entry o comportamento di installazione npm, inseriscilo in `package.json`

### Campi di package.json che influenzano la discovery

Alcuni metadati del plugin pre-runtime vivono intenzionalmente in `package.json` sotto il
blocco `openclaw` invece che in `openclaw.plugin.json`.

Esempi importanti:

| Campo                                                             | Significato                                                                                                                                   |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                             | Dichiara gli entrypoint del plugin nativo.                                                                                                    |
| `openclaw.setupEntry`                                             | Entrypoint leggero solo per il setup usato durante onboarding e avvio differito dei canali.                                                  |
| `openclaw.channel`                                                | Metadati economici del catalogo canali come etichette, percorsi della documentazione, alias e testo di selezione.                           |
| `openclaw.channel.configuredState`                                | Metadati leggeri del controllo dello stato configurato che possono rispondere a "esiste già una configurazione solo env?" senza caricare il runtime completo del canale. |
| `openclaw.channel.persistedAuthState`                             | Metadati leggeri del controllo dell'autenticazione persistita che possono rispondere a "c'è già qualcosa con login effettuato?" senza caricare il runtime completo del canale. |
| `openclaw.install.npmSpec` / `openclaw.install.localPath`         | Suggerimenti di installazione/aggiornamento per plugin bundled e pubblicati esternamente.                                                    |
| `openclaw.install.defaultChoice`                                  | Percorso di installazione preferito quando sono disponibili più origini di installazione.                                                     |
| `openclaw.install.minHostVersion`                                 | Versione minima supportata dell'host OpenClaw, usando una soglia semver come `>=2026.3.22`.                                                  |
| `openclaw.install.allowInvalidConfigRecovery`                     | Consente un percorso limitato di recupero tramite reinstallazione del plugin bundled quando la configurazione non è valida.                  |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen` | Consente il caricamento delle superfici del canale solo setup prima del plugin canale completo durante l'avvio.                              |

`openclaw.install.minHostVersion` viene applicato durante l'installazione e il
caricamento del registro dei manifest. I valori non validi vengono rifiutati; i valori validi ma più recenti saltano il
plugin sugli host più vecchi.

`openclaw.install.allowInvalidConfigRecovery` è intenzionalmente limitato. Non
rende installabili configurazioni arbitrariamente rotte. Oggi consente solo ai flussi di installazione
di recuperare da specifici errori obsoleti di aggiornamento del plugin bundled, come un
percorso mancante del plugin bundled o una voce `channels.<id>` obsoleta per quello stesso
plugin bundled. Errori di configurazione non correlati continuano a bloccare l'installazione e indirizzano gli operatori
a `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` è un metadato package per un piccolo
modulo di controllo:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Usalo quando i flussi di setup, doctor o stato configurato hanno bisogno di un probe di autenticazione sì/no economico prima del caricamento del plugin canale completo. L'export di destinazione deve essere una piccola
funzione che legge solo lo stato persistito; non instradarla attraverso il barrel completo del runtime del canale.

`openclaw.channel.configuredState` segue la stessa forma per controlli economici dello stato configurato solo env:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "specifier": "./configured-state",
        "exportName": "hasTelegramConfiguredState"
      }
    }
  }
}
```

Usalo quando un canale può rispondere allo stato configurato da env o da altri piccoli
input non runtime. Se il controllo richiede la risoluzione completa della configurazione o il vero
runtime del canale, mantieni invece quella logica nell'hook `config.hasConfiguredState` del plugin.

## Requisiti JSON Schema

- **Ogni plugin deve includere un JSON Schema**, anche se non accetta alcuna configurazione.
- Uno schema vuoto è accettabile (ad esempio `{ "type": "object", "additionalProperties": false }`).
- Gli schemi vengono convalidati in fase di lettura/scrittura della configurazione, non in fase di runtime.

## Comportamento della convalida

- Le chiavi `channels.*` sconosciute sono **errori**, a meno che l'ID canale non sia dichiarato da
  un manifest di plugin.
- `plugins.entries.<id>`, `plugins.allow`, `plugins.deny` e `plugins.slots.*`
  devono fare riferimento a ID plugin **rilevabili**. Gli ID sconosciuti sono **errori**.
- Se un plugin è installato ma ha un manifest o uno schema mancante o non valido,
  la convalida fallisce e Doctor segnala l'errore del plugin.
- Se esiste una configurazione del plugin ma il plugin è **disabilitato**, la configurazione viene mantenuta e
  viene mostrato un **avviso** in Doctor + nei log.

Consulta [Configuration reference](/it/gateway/configuration) per lo schema completo di `plugins.*`.

## Note

- Il manifest è **obbligatorio per i plugin OpenClaw nativi**, inclusi i caricamenti dal filesystem locale.
- Il runtime continua comunque a caricare separatamente il modulo del plugin; il manifest serve solo per
  discovery + convalida.
- I manifest nativi vengono analizzati con JSON5, quindi commenti, virgole finali e
  chiavi senza virgolette sono accettati purché il valore finale sia comunque un oggetto.
- Solo i campi del manifest documentati vengono letti dal loader del manifest. Evita di aggiungere
  qui chiavi di primo livello personalizzate.
- `providerAuthEnvVars` è il percorso di metadati economici per probe di autenticazione, convalida dei marker env
  e superfici simili di autenticazione del provider che non devono avviare il runtime del plugin
  solo per ispezionare i nomi env.
- `providerAuthChoices` è il percorso di metadati economici per selettori di scelte di autenticazione,
  risoluzione di `--auth-choice`, mappatura del provider preferito e semplice onboarding
  di registrazione dei flag CLI prima del caricamento del runtime del provider. Per i metadati della procedura guidata runtime
  che richiedono il codice del provider, consulta
  [Hook runtime del provider](/it/plugins/architecture#provider-runtime-hooks).
- I tipi esclusivi di plugin vengono selezionati tramite `plugins.slots.*`.
  - `kind: "memory"` viene selezionato da `plugins.slots.memory`.
  - `kind: "context-engine"` viene selezionato da `plugins.slots.contextEngine`
    (predefinito: `legacy` integrato).
- `channels`, `providers` e `skills` possono essere omessi quando un
  plugin non ne ha bisogno.
- Se il tuo plugin dipende da moduli nativi, documenta i passaggi di build e qualsiasi
  requisito di allowlist del package manager (ad esempio, pnpm `allow-build-scripts`
  - `pnpm rebuild <package>`).

## Correlati

- [Creare plugin](/it/plugins/building-plugins) — primi passi con i plugin
- [Architettura dei plugin](/it/plugins/architecture) — architettura interna
- [Panoramica SDK](/it/plugins/sdk-overview) — riferimento del Plugin SDK
