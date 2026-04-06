---
read_when:
    - Vuoi usare i modelli OpenAI in OpenClaw
    - Vuoi l'autenticazione con abbonamento Codex invece delle chiavi API
summary: Usare OpenAI tramite chiavi API o abbonamento Codex in OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-04-06T03:11:50Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e04db5787f6ed7b1eda04d965c10febae10809fc82ae4d9769e7163234471f5
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI fornisce API per sviluppatori per i modelli GPT. Codex supporta l'**accesso con ChatGPT** per l'accesso
in abbonamento oppure l'accesso con **chiave API** per l'accesso basato sul consumo. Codex cloud richiede l'accesso con ChatGPT.
OpenAI supporta esplicitamente l'uso di OAuth in abbonamento in strumenti/workflow esterni come OpenClaw.

## Stile di interazione predefinito

OpenClaw può aggiungere un piccolo overlay di prompt specifico per OpenAI sia per le esecuzioni `openai/*` sia per
`openai-codex/*`. Per impostazione predefinita, l'overlay mantiene l'assistente cordiale,
collaborativo, conciso, diretto e un po' più espressivo emotivamente
senza sostituire il system prompt di base di OpenClaw. L'overlay amichevole
consente anche l'uso occasionale di emoji quando si adatta in modo naturale, mantenendo
comunque l'output complessivamente conciso.

Chiave di configurazione:

`plugins.entries.openai.config.personality`

Valori consentiti:

- `"friendly"`: predefinito; abilita l'overlay specifico per OpenAI.
- `"off"`: disabilita l'overlay e usa solo il prompt di base di OpenClaw.

Ambito:

- Si applica ai modelli `openai/*`.
- Si applica ai modelli `openai-codex/*`.
- Non influisce sugli altri provider.

Questo comportamento è attivo per impostazione predefinita. Mantieni `"friendly"` esplicitamente se vuoi che
sopravviva a futuri cambiamenti locali della config:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "friendly",
        },
      },
    },
  },
}
```

### Disabilitare l'overlay di prompt OpenAI

Se vuoi il prompt di base di OpenClaw non modificato, imposta l'overlay su `"off"`:

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "off",
        },
      },
    },
  },
}
```

Puoi anche impostarlo direttamente con la CLI di configurazione:

```bash
openclaw config set plugins.entries.openai.config.personality off
```

## Opzione A: chiave API OpenAI (OpenAI Platform)

**Ideale per:** accesso diretto alle API e fatturazione basata sul consumo.
Ottieni la tua chiave API dalla dashboard OpenAI.

### Configurazione CLI

```bash
openclaw onboard --auth-choice openai-api-key
# oppure non interattivo
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Snippet di configurazione

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

La documentazione attuale dei modelli API di OpenAI elenca `gpt-5.4` e `gpt-5.4-pro` per l'uso diretto
delle API OpenAI. OpenClaw inoltra entrambi tramite il percorso `openai/*` Responses.
OpenClaw sopprime intenzionalmente la riga obsoleta `openai/gpt-5.3-codex-spark`,
perché le chiamate dirette alle API OpenAI la rifiutano nel traffico reale.

OpenClaw **non** espone `openai/gpt-5.3-codex-spark` sul percorso diretto delle API OpenAI.
`pi-ai` continua a includere una riga integrata per quel modello, ma le richieste reali alle API OpenAI
attualmente la rifiutano. Spark viene trattato come esclusivo di Codex in OpenClaw.

## Generazione di immagini

Il plugin bundled `openai` registra anche la generazione di immagini tramite lo strumento condiviso
`image_generate`.

- Modello immagine predefinito: `openai/gpt-image-1`
- Generazione: fino a 4 immagini per richiesta
- Modalità modifica: abilitata, fino a 5 immagini di riferimento
- Supporta `size`
- Limitazione attuale specifica di OpenAI: OpenClaw oggi non inoltra gli override `aspectRatio` o
  `resolution` alla OpenAI Images API

Per usare OpenAI come provider di immagini predefinito:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

Vedi [Image Generation](/it/tools/image-generation) per i parametri condivisi dello
strumento, la selezione del provider e il comportamento di failover.

## Generazione video

Il plugin bundled `openai` registra anche la generazione video tramite lo strumento condiviso
`video_generate`.

- Modello video predefinito: `openai/sora-2`
- Modalità: da testo a video, da immagine a video e flussi di riferimento/modifica con singolo video
- Limiti attuali: 1 immagine o 1 input video di riferimento
- Limitazione attuale specifica di OpenAI: OpenClaw attualmente inoltra solo gli override `size`
  per la generazione video nativa OpenAI. Gli override opzionali non supportati
  come `aspectRatio`, `resolution`, `audio` e `watermark` vengono ignorati
  e restituiti come avviso dello strumento.

Per usare OpenAI come provider video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openai/sora-2",
      },
    },
  },
}
```

Vedi [Video Generation](/tools/video-generation) per i parametri condivisi dello
strumento, la selezione del provider e il comportamento di failover.

## Opzione B: abbonamento OpenAI Code (Codex)

**Ideale per:** usare l'accesso in abbonamento ChatGPT/Codex invece di una chiave API.
Codex cloud richiede l'accesso con ChatGPT, mentre la CLI Codex supporta l'accesso con ChatGPT o con chiave API.

### Configurazione CLI (Codex OAuth)

```bash
# Esegui Codex OAuth nella procedura guidata
openclaw onboard --auth-choice openai-codex

# Oppure esegui direttamente OAuth
openclaw models auth login --provider openai-codex
```

### Snippet di configurazione (abbonamento Codex)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

La documentazione attuale di Codex di OpenAI elenca `gpt-5.4` come modello Codex corrente. OpenClaw
lo mappa a `openai-codex/gpt-5.4` per l'uso con OAuth ChatGPT/Codex.

Se l'onboarding riusa un accesso esistente della CLI Codex, tali credenziali restano
gestite dalla CLI Codex. Alla scadenza, OpenClaw rilegge prima la sorgente Codex esterna
e, quando il provider può aggiornarla, riscrive la credenziale aggiornata
nello storage di Codex invece di assumerne la gestione in una copia separata solo OpenClaw.

Se il tuo account Codex ha diritto a Codex Spark, OpenClaw supporta anche:

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw tratta Codex Spark come esclusivo di Codex. Non espone un percorso API-key diretto
`openai/gpt-5.3-codex-spark`.

OpenClaw conserva anche `openai-codex/gpt-5.3-codex-spark` quando `pi-ai`
lo rileva. Trattalo come dipendente dall'idoneità e sperimentale: Codex Spark è
separato da GPT-5.4 `/fast`, e la disponibilità dipende dall'account Codex /
ChatGPT connesso.

### Limite della finestra di contesto di Codex

OpenClaw tratta i metadati del modello Codex e il limite di contesto runtime come
valori separati.

Per `openai-codex/gpt-5.4`:

- `contextWindow` nativo: `1050000`
- limite predefinito runtime `contextTokens`: `272000`

Questo mantiene veritieri i metadati del modello preservando al contempo la
finestra runtime predefinita più piccola, che in pratica ha caratteristiche migliori di latenza e qualità.

Se vuoi un limite effettivo diverso, imposta `models.providers.<provider>.models[].contextTokens`:

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

Usa `contextWindow` solo quando stai dichiarando o sovrascrivendo metadati nativi
del modello. Usa `contextTokens` quando vuoi limitare il budget di contesto runtime.

### Trasporto predefinito

OpenClaw usa `pi-ai` per lo streaming del modello. Sia per `openai/*` sia per
`openai-codex/*`, il trasporto predefinito è `"auto"` (prima WebSocket, poi fallback
SSE).

In modalità `"auto"`, OpenClaw ritenta anche un errore WebSocket iniziale e ripetibile
prima di passare a SSE. La modalità `"websocket"` forzata continua invece a esporre direttamente
gli errori di trasporto invece di nasconderli dietro il fallback.

Dopo un errore WebSocket alla connessione o nei primi turni in modalità `"auto"`, OpenClaw contrassegna
il percorso WebSocket di quella sessione come degradato per circa 60 secondi e invia i
turni successivi tramite SSE durante il raffreddamento invece di oscillare continuamente tra
trasporti.

Per gli endpoint nativi della famiglia OpenAI (`openai/*`, `openai-codex/*` e Azure
OpenAI Responses), OpenClaw allega anche stato stabile di identità di sessione e turno
alle richieste in modo che retry, reconnessioni e fallback SSE restino allineati alla stessa
identità di conversazione. Sui percorsi nativi della famiglia OpenAI questo include header
stabili di identità della richiesta di sessione/turno più metadati di trasporto corrispondenti.

OpenClaw normalizza anche i contatori di utilizzo OpenAI tra le varianti di trasporto prima
che raggiungano le superfici di sessione/stato. Il traffico nativo OpenAI/Codex Responses può
segnalare l'uso sia come `input_tokens` / `output_tokens` sia come
`prompt_tokens` / `completion_tokens`; OpenClaw li tratta come gli stessi contatori di input
e output per `/status`, `/usage` e i log di sessione. Quando il traffico WebSocket nativo
omette `total_tokens` (o segnala `0`), OpenClaw usa come fallback il totale normalizzato input + output
così le visualizzazioni di sessione/stato restano popolate.

Puoi impostare `agents.defaults.models.<provider/model>.params.transport`:

- `"sse"`: forza SSE
- `"websocket"`: forza WebSocket
- `"auto"`: prova WebSocket, poi passa a SSE

Per `openai/*` (Responses API), OpenClaw abilita anche il warm-up WebSocket per impostazione
predefinita (`openaiWsWarmup: true`) quando viene usato il trasporto WebSocket.

Documentazione OpenAI correlata:

- [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### Warm-up WebSocket OpenAI

La documentazione OpenAI descrive il warm-up come facoltativo. OpenClaw lo abilita per impostazione predefinita per
`openai/*` per ridurre la latenza del primo turno quando si usa il trasporto WebSocket.

### Disabilitare il warm-up

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### Abilitare esplicitamente il warm-up

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### Elaborazione prioritaria OpenAI e Codex

L'API OpenAI espone l'elaborazione prioritaria tramite `service_tier=priority`. In
OpenClaw, imposta `agents.defaults.models["<provider>/<model>"].params.serviceTier`
per inoltrare quel campo sugli endpoint nativi OpenAI/Codex Responses.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

I valori supportati sono `auto`, `default`, `flex` e `priority`.

OpenClaw inoltra `params.serviceTier` sia alle richieste dirette `openai/*` Responses
sia alle richieste `openai-codex/*` Codex Responses quando quei modelli puntano
agli endpoint nativi OpenAI/Codex.

Comportamento importante:

- `openai/*` diretto deve puntare a `api.openai.com`
- `openai-codex/*` deve puntare a `chatgpt.com/backend-api`
- se instradi uno dei due provider tramite un altro URL base o proxy, OpenClaw lascia `service_tier` invariato

### Modalità fast di OpenAI

OpenClaw espone un toggle condiviso della modalità fast sia per le sessioni `openai/*` sia per
`openai-codex/*`:

- Chat/UI: `/fast status|on|off`
- Config: `agents.defaults.models["<provider>/<model>"].params.fastMode`

Quando la modalità fast è abilitata, OpenClaw la mappa all'elaborazione prioritaria OpenAI:

- le chiamate dirette `openai/*` Responses a `api.openai.com` inviano `service_tier = "priority"`
- anche le chiamate `openai-codex/*` Responses a `chatgpt.com/backend-api` inviano `service_tier = "priority"`
- i valori `service_tier` già presenti nel payload vengono preservati
- la modalità fast non riscrive `reasoning` o `text.verbosity`

Per GPT 5.4 nello specifico, la configurazione più comune è:

- inviare `/fast on` in una sessione che usa `openai/gpt-5.4` o `openai-codex/gpt-5.4`
- oppure impostare `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true`
- se usi anche Codex OAuth, imposta anche `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true`

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

Gli override di sessione hanno priorità sulla config. Cancellando l'override di sessione nella UI Sessions
la sessione torna al valore predefinito configurato.

### Percorsi nativi OpenAI rispetto a quelli compatibili con OpenAI

OpenClaw tratta gli endpoint diretti OpenAI, Codex e Azure OpenAI in modo diverso
rispetto ai proxy generici compatibili OpenAI `/v1`:

- i percorsi nativi `openai/*`, `openai-codex/*` e Azure OpenAI mantengono
  `reasoning: { effort: "none" }` intatto quando disabiliti esplicitamente il ragionamento
- i percorsi nativi della famiglia OpenAI impostano per default gli schemi degli strumenti in modalità strict
- gli header nascosti di attribuzione OpenClaw (`originator`, `version` e
  `User-Agent`) vengono allegati solo sugli host nativi OpenAI verificati
  (`api.openai.com`) e sugli host Codex nativi (`chatgpt.com/backend-api`)
- i percorsi nativi OpenAI/Codex mantengono il request shaping specifico di OpenAI come
  `service_tier`, `store` di Responses, payload di compatibilità del ragionamento OpenAI e
  hint della cache del prompt
- i percorsi compatibili con OpenAI in stile proxy mantengono il comportamento compatibile più permissivo e non
  forzano schemi degli strumenti strict, request shaping solo nativo o header di attribuzione nascosti
  OpenAI/Codex

Azure OpenAI resta nel gruppo di routing nativo per il comportamento di trasporto e compatibilità,
ma non riceve gli header nascosti di attribuzione OpenAI/Codex.

Questo preserva l'attuale comportamento nativo di OpenAI Responses senza imporre vecchie
shim compatibili con OpenAI su backend `/v1` di terze parti.

### Compaction lato server di OpenAI Responses

Per i modelli OpenAI Responses diretti (`openai/*` che usano `api: "openai-responses"` con
`baseUrl` su `api.openai.com`), OpenClaw ora abilita automaticamente gli hint di payload per la
compaction lato server di OpenAI:

- Forza `store: true` (a meno che la compat del modello imposti `supportsStore: false`)
- Inietta `context_management: [{ type: "compaction", compact_threshold: ... }]`

Per impostazione predefinita, `compact_threshold` è il `70%` del `contextWindow` del modello (oppure `80000`
quando non disponibile).

### Abilitare esplicitamente la compaction lato server

Usa questa opzione quando vuoi forzare l'iniezione di `context_management` su modelli
Responses compatibili (per esempio Azure OpenAI Responses):

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### Abilitare con una soglia personalizzata

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### Disabilitare la compaction lato server

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction` controlla solo l'iniezione di `context_management`.
I modelli OpenAI Responses diretti continuano a forzare `store: true` a meno che la compatibilità imposti
`supportsStore: false`.

## Note

- I riferimenti ai modelli usano sempre `provider/model` (vedi [/concepts/models](/it/concepts/models)).
- I dettagli auth + le regole di riuso sono in [/concepts/oauth](/it/concepts/oauth).
