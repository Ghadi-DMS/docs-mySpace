---
read_when:
    - Vuoi configurare provider di memory search o modelli di embedding
    - Vuoi configurare il backend QMD
    - Vuoi regolare la ricerca ibrida, MMR o il decadimento temporale
    - Vuoi abilitare l'indicizzazione memory multimodale
summary: Tutte le opzioni di configurazione per memory search, provider di embedding, QMD, ricerca ibrida e indicizzazione multimodale
title: Riferimento della configurazione memory
x-i18n:
    generated_at: "2026-04-06T03:12:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0de0b85125443584f4e575cf673ca8d9bd12ecd849d73c537f4a17545afa93fd
    source_path: reference/memory-config.md
    workflow: 15
---

# Riferimento della configurazione memory

Questa pagina elenca ogni opzione di configurazione per memory search di OpenClaw. Per
le panoramiche concettuali, vedi:

- [Panoramica memory](/it/concepts/memory) -- come funziona memory
- [Motore integrato](/it/concepts/memory-builtin) -- backend SQLite predefinito
- [Motore QMD](/it/concepts/memory-qmd) -- sidecar local-first
- [Memory Search](/it/concepts/memory-search) -- pipeline di ricerca e regolazione

Tutte le impostazioni di memory search si trovano in `agents.defaults.memorySearch` in
`openclaw.json`, salvo diversa indicazione.

---

## Selezione del provider

| Chiave     | Tipo      | Predefinito      | Descrizione                                                                                |
| ---------- | --------- | ---------------- | ------------------------------------------------------------------------------------------ |
| `provider` | `string`  | rilevato automaticamente | ID adattatore di embedding: `openai`, `gemini`, `voyage`, `mistral`, `bedrock`, `ollama`, `local` |
| `model`    | `string`  | predefinito del provider | Nome del modello di embedding                                                              |
| `fallback` | `string`  | `"none"`         | ID adattatore di fallback quando il primario fallisce                                      |
| `enabled`  | `boolean` | `true`           | Abilita o disabilita memory search                                                         |

### Ordine di rilevamento automatico

Quando `provider` non è impostato, OpenClaw seleziona il primo disponibile:

1. `local` -- se `memorySearch.local.modelPath` è configurato e il file esiste.
2. `openai` -- se è possibile risolvere una chiave OpenAI.
3. `gemini` -- se è possibile risolvere una chiave Gemini.
4. `voyage` -- se è possibile risolvere una chiave Voyage.
5. `mistral` -- se è possibile risolvere una chiave Mistral.
6. `bedrock` -- se la catena di credenziali AWS SDK viene risolta (ruolo istanza, chiavi di accesso, profilo, SSO, web identity o configurazione condivisa).

`ollama` è supportato ma non viene rilevato automaticamente (impostalo esplicitamente).

### Risoluzione della chiave API

Gli embedding remoti richiedono una chiave API. Bedrock usa invece la catena predefinita
di credenziali AWS SDK (ruoli istanza, SSO, chiavi di accesso).

| Provider | Variabile d'ambiente         | Chiave di configurazione          |
| -------- | ---------------------------- | --------------------------------- |
| OpenAI   | `OPENAI_API_KEY`             | `models.providers.openai.apiKey`  |
| Gemini   | `GEMINI_API_KEY`             | `models.providers.google.apiKey`  |
| Voyage   | `VOYAGE_API_KEY`             | `models.providers.voyage.apiKey`  |
| Mistral  | `MISTRAL_API_KEY`            | `models.providers.mistral.apiKey` |
| Bedrock  | catena di credenziali AWS    | Nessuna chiave API necessaria     |
| Ollama   | `OLLAMA_API_KEY` (segnaposto) | --                               |

Codex OAuth copre solo chat/completions e non soddisfa le
richieste di embedding.

---

## Configurazione dell'endpoint remoto

Per endpoint personalizzati compatibili con OpenAI o per sovrascrivere i valori predefiniti del provider:

| Chiave            | Tipo     | Descrizione                                          |
| ----------------- | -------- | ---------------------------------------------------- |
| `remote.baseUrl`  | `string` | URL base API personalizzato                          |
| `remote.apiKey`   | `string` | Sovrascrive la chiave API                            |
| `remote.headers`  | `object` | Header HTTP aggiuntivi (uniti ai valori predefiniti del provider) |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Configurazione specifica di Gemini

| Chiave                 | Tipo     | Predefinito            | Descrizione                                |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | Supporta anche `gemini-embedding-2-preview` |
| `outputDimensionality` | `number` | `3072`                 | Per Embedding 2: 768, 1536 o 3072          |

<Warning>
Cambiare il modello o `outputDimensionality` attiva automaticamente una reindicizzazione completa.
</Warning>

---

## Configurazione degli embedding Bedrock

Bedrock usa la catena predefinita di credenziali AWS SDK -- non servono chiavi API.
Se OpenClaw viene eseguito su EC2 con un ruolo di istanza Bedrock-enabled, basta impostare
provider e modello:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0",
      },
    },
  },
}
```

| Chiave                 | Tipo     | Predefinito                    | Descrizione                    |
| ---------------------- | -------- | ------------------------------ | ------------------------------ |
| `model`                | `string` | `amazon.titan-embed-text-v2:0` | Qualsiasi ID modello embedding Bedrock |
| `outputDimensionality` | `number` | predefinito del modello        | Per Titan V2: 256, 512 o 1024  |

### Modelli supportati

Sono supportati i seguenti modelli (con rilevamento della famiglia e valori predefiniti
delle dimensioni):

| ID modello                                 | Provider   | Dim predefinite | Dim configurabili     |
| ------------------------------------------ | ---------- | --------------- | --------------------- |
| `amazon.titan-embed-text-v2:0`             | Amazon     | 1024            | 256, 512, 1024        |
| `amazon.titan-embed-text-v1`               | Amazon     | 1536            | --                    |
| `amazon.titan-embed-g1-text-02`            | Amazon     | 1536            | --                    |
| `amazon.titan-embed-image-v1`              | Amazon     | 1024            | --                    |
| `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024            | 256, 384, 1024, 3072  |
| `cohere.embed-english-v3`                  | Cohere     | 1024            | --                    |
| `cohere.embed-multilingual-v3`             | Cohere     | 1024            | --                    |
| `cohere.embed-v4:0`                        | Cohere     | 1536            | 256-1536              |
| `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512             | --                    |
| `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024            | --                    |

Le varianti con suffisso di throughput (ad esempio `amazon.titan-embed-text-v1:2:8k`) ereditano
la configurazione del modello base.

### Autenticazione

L'autenticazione Bedrock usa l'ordine standard di risoluzione delle credenziali AWS SDK:

1. Variabili d'ambiente (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
2. Cache dei token SSO
3. Credenziali del token web identity
4. File condivisi di credenziali e configurazione
5. Credenziali metadata ECS o EC2

La regione viene risolta da `AWS_REGION`, `AWS_DEFAULT_REGION`, dal
`baseUrl` del provider `amazon-bedrock`, oppure usa `us-east-1` come predefinito.

### Permessi IAM

Il ruolo o l'utente IAM richiede:

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "*"
}
```

Per il principio del privilegio minimo, limita `InvokeModel` al modello specifico:

```
arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
```

---

## Configurazione degli embedding locali

| Chiave                | Tipo     | Predefinito              | Descrizione                   |
| --------------------- | -------- | ------------------------ | ----------------------------- |
| `local.modelPath`     | `string` | scaricato automaticamente | Percorso del file modello GGUF |
| `local.modelCacheDir` | `string` | predefinito di node-llama-cpp | Directory cache per i modelli scaricati |

Modello predefinito: `embeddinggemma-300m-qat-Q8_0.gguf` (~0,6 GB, scaricato automaticamente).
Richiede build nativa: `pnpm approve-builds` poi `pnpm rebuild node-llama-cpp`.

---

## Configurazione della ricerca ibrida

Tutto in `memorySearch.query.hybrid`:

| Chiave                | Tipo      | Predefinito | Descrizione                         |
| --------------------- | --------- | ----------- | ----------------------------------- |
| `enabled`             | `boolean` | `true`      | Abilita la ricerca ibrida BM25 + vettoriale |
| `vectorWeight`        | `number`  | `0.7`       | Peso dei punteggi vettoriali (0-1)  |
| `textWeight`          | `number`  | `0.3`       | Peso dei punteggi BM25 (0-1)        |
| `candidateMultiplier` | `number`  | `4`         | Moltiplicatore della dimensione del pool di candidati |

### MMR (diversità)

| Chiave        | Tipo      | Predefinito | Descrizione                              |
| ------------- | --------- | ----------- | ---------------------------------------- |
| `mmr.enabled` | `boolean` | `false`     | Abilita il riordinamento MMR             |
| `mmr.lambda`  | `number`  | `0.7`       | 0 = massima diversità, 1 = massima rilevanza |

### Decadimento temporale (recency)

| Chiave                       | Tipo      | Predefinito | Descrizione                    |
| ---------------------------- | --------- | ----------- | ------------------------------ |
| `temporalDecay.enabled`      | `boolean` | `false`     | Abilita il boost di recency    |
| `temporalDecay.halfLifeDays` | `number`  | `30`        | Il punteggio si dimezza ogni N giorni |

I file evergreen (`MEMORY.md`, file non datati in `memory/`) non decadono mai.

### Esempio completo

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## Percorsi memory aggiuntivi

| Chiave       | Tipo       | Descrizione                              |
| ------------ | ---------- | ---------------------------------------- |
| `extraPaths` | `string[]` | Directory o file aggiuntivi da indicizzare |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

I percorsi possono essere assoluti o relativi al workspace. Le directory vengono analizzate
ricorsivamente per i file `.md`. La gestione dei symlink dipende dal backend attivo:
il motore integrato ignora i symlink, mentre QMD segue il comportamento dello scanner QMD
sottostante.

Per la ricerca di transcript cross-agent scoped per agente, usa
`agents.list[].memorySearch.qmd.extraCollections` invece di `memory.qmd.paths`.
Queste raccolte aggiuntive seguono la stessa forma `{ path, name, pattern? }`, ma
vengono unite per agente e possono preservare nomi condivisi espliciti quando il percorso
punta fuori dal workspace corrente.
Se lo stesso percorso risolto appare sia in `memory.qmd.paths` sia in
`memorySearch.qmd.extraCollections`, QMD mantiene la prima voce e salta il
duplicato.

---

## Memory multimodale (Gemini)

Indicizza immagini e audio insieme a Markdown usando Gemini Embedding 2:

| Chiave                    | Tipo       | Predefinito | Descrizione                              |
| ------------------------- | ---------- | ----------- | ---------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`     | Abilita l'indicizzazione multimodale     |
| `multimodal.modalities`   | `string[]` | --          | `["image"]`, `["audio"]` o `["all"]`     |
| `multimodal.maxFileBytes` | `number`   | `10000000`  | Dimensione massima del file per l'indicizzazione |

Si applica solo ai file in `extraPaths`. Le radici memory predefinite restano solo Markdown.
Richiede `gemini-embedding-2-preview`. `fallback` deve essere `"none"`.

Formati supportati: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`
(immagini); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac` (audio).

---

## Cache degli embedding

| Chiave             | Tipo      | Predefinito | Descrizione                         |
| ------------------ | --------- | ----------- | ----------------------------------- |
| `cache.enabled`    | `boolean` | `false`     | Memorizza nella cache gli embedding dei chunk in SQLite |
| `cache.maxEntries` | `number`  | `50000`     | Numero massimo di embedding in cache |

Impedisce il ri-embedding del testo invariato durante reindicizzazione o aggiornamenti dei transcript.

---

## Indicizzazione batch

| Chiave                        | Tipo      | Predefinito | Descrizione                      |
| ----------------------------- | --------- | ----------- | -------------------------------- |
| `remote.batch.enabled`        | `boolean` | `false`     | Abilita l'API di embedding batch |
| `remote.batch.concurrency`    | `number`  | `2`         | Job batch paralleli              |
| `remote.batch.wait`           | `boolean` | `true`      | Attende il completamento del batch |
| `remote.batch.pollIntervalMs` | `number`  | --          | Intervallo di polling            |
| `remote.batch.timeoutMinutes` | `number`  | --          | Timeout del batch                |

Disponibile per `openai`, `gemini` e `voyage`. Il batch OpenAI è di solito
il più rapido ed economico per grandi backfill.

---

## Session memory search (sperimentale)

Indicizza i transcript di sessione e li espone tramite `memory_search`:

| Chiave                        | Tipo       | Predefinito   | Descrizione                               |
| ----------------------------- | ---------- | ------------- | ----------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`       | Abilita l'indicizzazione delle sessioni   |
| `sources`                     | `string[]` | `["memory"]`  | Aggiungi `"sessions"` per includere i transcript |
| `sync.sessions.deltaBytes`    | `number`   | `100000`      | Soglia in byte per la reindicizzazione    |
| `sync.sessions.deltaMessages` | `number`   | `50`          | Soglia in messaggi per la reindicizzazione |

L'indicizzazione delle sessioni è opt-in e viene eseguita in modo asincrono. I risultati possono essere leggermente
obsoleti. I log di sessione vivono su disco, quindi considera l'accesso al filesystem come il
confine di trust.

---

## Accelerazione vettoriale SQLite (sqlite-vec)

| Chiave                       | Tipo      | Predefinito | Descrizione                         |
| ---------------------------- | --------- | ----------- | ----------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`      | Usa sqlite-vec per le query vettoriali |
| `store.vector.extensionPath` | `string`  | bundled     | Sovrascrive il percorso sqlite-vec  |

Quando sqlite-vec non è disponibile, OpenClaw ripiega automaticamente sulla
similarità coseno in-process.

---

## Archiviazione dell'indice

| Chiave                | Tipo     | Predefinito                           | Descrizione                                 |
| --------------------- | -------- | ------------------------------------- | ------------------------------------------- |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | Posizione dell'indice (supporta il token `{agentId}`) |
| `store.fts.tokenizer` | `string` | `unicode61`                           | Tokenizer FTS5 (`unicode61` o `trigram`)    |

---

## Configurazione del backend QMD

Imposta `memory.backend = "qmd"` per abilitarlo. Tutte le impostazioni QMD si trovano in
`memory.qmd`:

| Chiave                   | Tipo      | Predefinito | Descrizione                                  |
| ------------------------ | --------- | ----------- | -------------------------------------------- |
| `command`                | `string`  | `qmd`       | Percorso dell'eseguibile QMD                 |
| `searchMode`             | `string`  | `search`    | Comando di ricerca: `search`, `vsearch`, `query` |
| `includeDefaultMemory`   | `boolean` | `true`      | Indicizza automaticamente `MEMORY.md` + `memory/**/*.md` |
| `paths[]`                | `array`   | --          | Percorsi aggiuntivi: `{ name, path, pattern? }` |
| `sessions.enabled`       | `boolean` | `false`     | Indicizza i transcript di sessione           |
| `sessions.retentionDays` | `number`  | --          | Conservazione dei transcript                 |
| `sessions.exportDir`     | `string`  | --          | Directory di esportazione                    |

OpenClaw preferisce le attuali forme di raccolta QMD e query MCP, ma mantiene
funzionanti le versioni QMD meno recenti ripiegando sui flag legacy di raccolta `--mask`
e sui vecchi nomi degli strumenti MCP quando necessario.

Gli override dei modelli QMD restano sul lato QMD, non nella configurazione OpenClaw. Se hai bisogno di
sovrascrivere globalmente i modelli di QMD, imposta variabili d'ambiente come
`QMD_EMBED_MODEL`, `QMD_RERANK_MODEL` e `QMD_GENERATE_MODEL` nell'ambiente runtime del gateway.

### Pianificazione degli aggiornamenti

| Chiave                    | Tipo      | Predefinito | Descrizione                                |
| ------------------------- | --------- | ----------- | ------------------------------------------ |
| `update.interval`         | `string`  | `5m`        | Intervallo di aggiornamento                |
| `update.debounceMs`       | `number`  | `15000`     | Debounce delle modifiche ai file           |
| `update.onBoot`           | `boolean` | `true`      | Aggiorna all'avvio                         |
| `update.waitForBootSync`  | `boolean` | `false`     | Blocca l'avvio finché l'aggiornamento non è completato |
| `update.embedInterval`    | `string`  | --          | Cadenza separata per gli embedding         |
| `update.commandTimeoutMs` | `number`  | --          | Timeout per i comandi QMD                  |
| `update.updateTimeoutMs`  | `number`  | --          | Timeout per le operazioni di aggiornamento QMD |
| `update.embedTimeoutMs`   | `number`  | --          | Timeout per le operazioni di embedding QMD |

### Limiti

| Chiave                    | Tipo     | Predefinito | Descrizione                     |
| ------------------------- | -------- | ----------- | ------------------------------- |
| `limits.maxResults`       | `number` | `6`         | Numero massimo di risultati di ricerca |
| `limits.maxSnippetChars`  | `number` | --          | Limita la lunghezza dello snippet |
| `limits.maxInjectedChars` | `number` | --          | Limita il totale dei caratteri iniettati |
| `limits.timeoutMs`        | `number` | `4000`      | Timeout della ricerca           |

### Scope

Controlla quali sessioni possono ricevere risultati di ricerca QMD. Stesso schema di
[`session.sendPolicy`](/it/gateway/configuration-reference#session):

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

Il valore predefinito è solo DM. `match.keyPrefix` corrisponde alla chiave di sessione normalizzata;
`match.rawKeyPrefix` corrisponde alla chiave raw inclusa `agent:<id>:`.

### Citazioni

`memory.citations` si applica a tutti i backend:

| Valore           | Comportamento                                       |
| ---------------- | --------------------------------------------------- |
| `auto` (predefinito) | Include il footer `Source: <path#line>` negli snippet |
| `on`             | Include sempre il footer                            |
| `off`            | Omette il footer (il percorso viene comunque passato internamente all'agente) |

### Esempio completo QMD

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming (sperimentale)

Dreaming viene configurato in `plugins.entries.memory-core.config.dreaming`,
non in `agents.defaults.memorySearch`.

Dreaming viene eseguito come una scansione pianificata unica e usa fasi interne light/deep/REM come
dettaglio di implementazione.

Per il comportamento concettuale e i comandi slash, vedi [Dreaming](/concepts/dreaming).

### Impostazioni utente

| Chiave      | Tipo      | Predefinito  | Descrizione                                      |
| ----------- | --------- | ------------ | ------------------------------------------------ |
| `enabled`   | `boolean` | `false`      | Abilita o disabilita completamente dreaming      |
| `frequency` | `string`  | `0 3 * * *`  | Cadenza cron facoltativa per la scansione completa di dreaming |

### Esempio

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
          },
        },
      },
    },
  },
}
```

Note:

- Dreaming scrive lo stato della macchina in `memory/.dreams/`.
- Dreaming scrive output narrativo leggibile da esseri umani in `DREAMS.md` (o nell'esistente `dreams.md`).
- La policy e le soglie delle fasi light/deep/REM sono comportamento interno, non configurazione esposta all'utente.
