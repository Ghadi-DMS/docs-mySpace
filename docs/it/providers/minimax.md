---
read_when:
    - Vuoi usare i modelli MiniMax in OpenClaw
    - Hai bisogno di indicazioni per configurare MiniMax
summary: Usa i modelli MiniMax in OpenClaw
title: MiniMax
x-i18n:
    generated_at: "2026-04-06T03:11:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9ca35c43cdde53f6f09d9e12d48ce09e4c099cf8cbe1407ac6dbb45b1422507e
    source_path: providers/minimax.md
    workflow: 15
---

# MiniMax

Il provider MiniMax di OpenClaw usa per impostazione predefinita **MiniMax M2.7**.

MiniMax fornisce anche:

- sintesi vocale bundled tramite T2A v2
- comprensione delle immagini bundled tramite `MiniMax-VL-01`
- generazione musicale bundled tramite `music-2.5+`
- `web_search` bundled tramite l'API di ricerca del MiniMax Coding Plan

Suddivisione del provider:

- `minimax`: provider testuale con chiave API, più generazione di immagini, comprensione delle immagini, voce e ricerca web bundled
- `minimax-portal`: provider testuale OAuth, più generazione di immagini e comprensione delle immagini bundled

## Gamma di modelli

- `MiniMax-M2.7`: modello di reasoning hosted predefinito.
- `MiniMax-M2.7-highspeed`: livello di reasoning M2.7 più veloce.
- `image-01`: modello di generazione di immagini (generazione e modifica image-to-image).

## Generazione di immagini

Il plugin MiniMax registra il modello `image-01` per lo strumento `image_generate`. Supporta:

- **Generazione text-to-image** con controllo del rapporto d'aspetto.
- **Modifica image-to-image** (immagine di riferimento del soggetto) con controllo del rapporto d'aspetto.
- Fino a **9 immagini in output** per richiesta.
- Fino a **1 immagine di riferimento** per richiesta di modifica.
- Rapporti d'aspetto supportati: `1:1`, `16:9`, `4:3`, `3:2`, `2:3`, `3:4`, `9:16`, `21:9`.

Per usare MiniMax per la generazione di immagini, impostalo come provider di generazione immagini:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

Il plugin usa la stessa autenticazione `MINIMAX_API_KEY` o OAuth dei modelli testuali. Non è necessaria alcuna configurazione aggiuntiva se MiniMax è già configurato.

Sia `minimax` sia `minimax-portal` registrano `image_generate` con lo stesso
modello `image-01`. Le configurazioni con chiave API usano `MINIMAX_API_KEY`; le configurazioni OAuth possono invece usare
il percorso auth bundled `minimax-portal`.

Quando l'onboarding o la configurazione con chiave API scrivono voci esplicite `models.providers.minimax`,
OpenClaw materializza `MiniMax-M2.7` e
`MiniMax-M2.7-highspeed` con `input: ["text", "image"]`.

Il catalogo testuale MiniMax bundled integrato rimane invece composto da metadati solo testuali finché
non esiste una configurazione esplicita del provider. La comprensione delle immagini è esposta separatamente
tramite il provider media `MiniMax-VL-01` gestito dal plugin.

Vedi [Image Generation](/it/tools/image-generation) per i parametri condivisi dello
strumento, la selezione del provider e il comportamento di failover.

## Generazione musicale

Il plugin bundled `minimax` registra anche la generazione musicale tramite lo strumento condiviso
`music_generate`.

- Modello musicale predefinito: `minimax/music-2.5+`
- Supporta anche `minimax/music-2.5` e `minimax/music-2.0`
- Controlli del prompt: `lyrics`, `instrumental`, `durationSeconds`
- Formato di output: `mp3`
- Le esecuzioni supportate dalla sessione vengono separate tramite il flusso condiviso di task/stato, incluso `action: "status"`

Per usare MiniMax come provider musicale predefinito:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "minimax/music-2.5+",
      },
    },
  },
}
```

Vedi [Music Generation](/tools/music-generation) per i parametri condivisi dello
strumento, la selezione del provider e il comportamento di failover.

## Generazione video

Il plugin bundled `minimax` registra anche la generazione video tramite lo strumento condiviso
`video_generate`.

- Modello video predefinito: `minimax/MiniMax-Hailuo-2.3`
- Modalità: flussi text-to-video e con singola immagine di riferimento
- Supporta `aspectRatio` e `resolution`

Per usare MiniMax come provider video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "minimax/MiniMax-Hailuo-2.3",
      },
    },
  },
}
```

Vedi [Video Generation](/tools/video-generation) per i parametri condivisi dello
strumento, la selezione del provider e il comportamento di failover.

## Comprensione delle immagini

Il plugin MiniMax registra la comprensione delle immagini separatamente dal catalogo
testuale:

- `minimax`: modello immagine predefinito `MiniMax-VL-01`
- `minimax-portal`: modello immagine predefinito `MiniMax-VL-01`

Per questo motivo l'instradamento automatico dei media può usare la comprensione delle immagini MiniMax anche
quando il catalogo bundled del provider testuale mostra ancora riferimenti chat M2.7 solo testuali.

## Ricerca web

Il plugin MiniMax registra anche `web_search` tramite l'API di ricerca del MiniMax Coding Plan.

- ID provider: `minimax`
- Risultati strutturati: titoli, URL, snippet, query correlate
- Variabile env preferita: `MINIMAX_CODE_PLAN_KEY`
- Alias env accettato: `MINIMAX_CODING_API_KEY`
- Fallback di compatibilità: `MINIMAX_API_KEY` quando punta già a un token coding-plan
- Riutilizzo della regione: `plugins.entries.minimax.config.webSearch.region`, poi `MINIMAX_API_HOST`, poi gli URL base del provider MiniMax
- La ricerca rimane sull'id provider `minimax`; la configurazione OAuth CN/globale può comunque indirizzare indirettamente la regione tramite `models.providers.minimax-portal.baseUrl`

La configurazione si trova sotto `plugins.entries.minimax.config.webSearch.*`.
Vedi [MiniMax Search](/it/tools/minimax-search).

## Scegli una configurazione

### MiniMax OAuth (Coding Plan) - consigliato

**Ideale per:** configurazione rapida con MiniMax Coding Plan tramite OAuth, senza necessità di chiave API.

Autenticati con la scelta OAuth regionale esplicita:

```bash
openclaw onboard --auth-choice minimax-global-oauth
# oppure
openclaw onboard --auth-choice minimax-cn-oauth
```

Mappatura delle scelte:

- `minimax-global-oauth`: utenti internazionali (`api.minimax.io`)
- `minimax-cn-oauth`: utenti in Cina (`api.minimaxi.com`)

Vedi il README del pacchetto plugin MiniMax nel repository OpenClaw per i dettagli.

### MiniMax M2.7 (chiave API)

**Ideale per:** MiniMax hosted con API compatibile con Anthropic.

Configura tramite CLI:

- Onboarding interattivo:

```bash
openclaw onboard --auth-choice minimax-global-api
# oppure
openclaw onboard --auth-choice minimax-cn-api
```

- `minimax-global-api`: utenti internazionali (`api.minimax.io`)
- `minimax-cn-api`: utenti in Cina (`api.minimaxi.com`)

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "minimax/MiniMax-M2.7" } } },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
          {
            id: "MiniMax-M2.7-highspeed",
            name: "MiniMax M2.7 Highspeed",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.6, output: 2.4, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Sul percorso di streaming compatibile con Anthropic, OpenClaw ora disattiva il
thinking MiniMax per impostazione predefinita a meno che tu non imposti esplicitamente `thinking`. L'endpoint
di streaming di MiniMax emette `reasoning_content` in chunk delta in stile OpenAI
invece dei blocchi di thinking nativi Anthropic, il che può esporre reasoning interno
nell'output visibile se lasciato implicitamente abilitato.

### MiniMax M2.7 come fallback (esempio)

**Ideale per:** mantenere come primario il tuo modello più potente di ultima generazione e fare fail over su MiniMax M2.7.
L'esempio sotto usa Opus come primario concreto; sostituiscilo con il tuo modello primario preferito di ultima generazione.

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "primary" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
    },
  },
}
```

## Configura tramite `openclaw configure`

Usa la procedura guidata di configurazione interattiva per impostare MiniMax senza modificare JSON:

1. Esegui `openclaw configure`.
2. Seleziona **Model/auth**.
3. Scegli un'opzione di autenticazione **MiniMax**.
4. Scegli il tuo modello predefinito quando richiesto.

Scelte di autenticazione MiniMax attuali nella procedura guidata/CLI:

- `minimax-global-oauth`
- `minimax-cn-oauth`
- `minimax-global-api`
- `minimax-cn-api`

## Opzioni di configurazione

- `models.providers.minimax.baseUrl`: preferisci `https://api.minimax.io/anthropic` (compatibile con Anthropic); `https://api.minimax.io/v1` è facoltativo per payload compatibili con OpenAI.
- `models.providers.minimax.api`: preferisci `anthropic-messages`; `openai-completions` è facoltativo per payload compatibili con OpenAI.
- `models.providers.minimax.apiKey`: chiave API MiniMax (`MINIMAX_API_KEY`).
- `models.providers.minimax.models`: definisci `id`, `name`, `reasoning`, `contextWindow`, `maxTokens`, `cost`.
- `agents.defaults.models`: assegna alias ai modelli che vuoi nella allowlist.
- `models.mode`: mantieni `merge` se vuoi aggiungere MiniMax accanto ai provider integrati.

## Note

- I riferimenti ai modelli seguono il percorso di autenticazione:
  - configurazione con chiave API: `minimax/<model>`
  - configurazione OAuth: `minimax-portal/<model>`
- Modello chat predefinito: `MiniMax-M2.7`
- Modello chat alternativo: `MiniMax-M2.7-highspeed`
- Su `api: "anthropic-messages"`, OpenClaw inietta
  `thinking: { type: "disabled" }` a meno che il thinking non sia già impostato esplicitamente in
  params/config.
- `/fast on` o `params.fastMode: true` riscrivono `MiniMax-M2.7` in
  `MiniMax-M2.7-highspeed` sul percorso di stream compatibile con Anthropic.
- L'onboarding e la configurazione diretta con chiave API scrivono definizioni esplicite dei modelli con
  `input: ["text", "image"]` per entrambe le varianti M2.7
- Il catalogo provider bundled attualmente espone i riferimenti chat come metadati
  solo testuali finché non esiste una configurazione esplicita del provider MiniMax
- API di utilizzo del Coding Plan: `https://api.minimaxi.com/v1/api/openplatform/coding_plan/remains` (richiede una chiave coding plan).
- OpenClaw normalizza l'utilizzo del coding plan MiniMax nello stesso formato di visualizzazione `% rimanente`
  usato da altri provider. I campi grezzi `usage_percent` / `usagePercent` di MiniMax
  rappresentano la quota residua, non la quota consumata, quindi OpenClaw li inverte.
  I campi basati sul conteggio hanno priorità quando presenti. Quando l'API restituisce `model_remains`,
  OpenClaw preferisce la voce del modello chat, ricava l'etichetta della finestra da
  `start_time` / `end_time` quando necessario e include il nome del modello selezionato
  nell'etichetta del piano in modo che le finestre del coding plan siano più facili da distinguere.
- Gli snapshot di utilizzo trattano `minimax`, `minimax-cn` e `minimax-portal` come la
  stessa superficie di quota MiniMax, e preferiscono l'OAuth MiniMax memorizzato prima di
  ripiegare sulle variabili env della chiave Coding Plan.
- Aggiorna i valori di prezzo in `models.json` se hai bisogno di un tracciamento dei costi preciso.
- Link referral per MiniMax Coding Plan (10% di sconto): [https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)
- Vedi [/concepts/model-providers](/it/concepts/model-providers) per le regole dei provider.
- Usa `openclaw models list` per confermare l'id provider attuale, quindi cambia con
  `openclaw models set minimax/MiniMax-M2.7` oppure
  `openclaw models set minimax-portal/MiniMax-M2.7`.

## Risoluzione dei problemi

### "Unknown model: minimax/MiniMax-M2.7"

Di solito significa che il **provider MiniMax non è configurato** (nessuna
voce provider corrispondente e nessun profilo auth/chiave env MiniMax trovato). Una correzione per questo
rilevamento è presente in **2026.1.12**. Per risolvere:

- Aggiorna a **2026.1.12** (oppure esegui dal sorgente `main`), quindi riavvia il gateway.
- Esegui `openclaw configure` e seleziona un'opzione auth **MiniMax**, oppure
- Aggiungi manualmente il blocco corrispondente `models.providers.minimax` o
  `models.providers.minimax-portal`, oppure
- Imposta `MINIMAX_API_KEY`, `MINIMAX_OAUTH_TOKEN` o un profilo auth MiniMax
  in modo che il provider corrispondente possa essere iniettato.

Assicurati che l'id modello sia **case‑sensitive**:

- percorso chiave API: `minimax/MiniMax-M2.7` o `minimax/MiniMax-M2.7-highspeed`
- percorso OAuth: `minimax-portal/MiniMax-M2.7` o
  `minimax-portal/MiniMax-M2.7-highspeed`

Poi ricontrolla con:

```bash
openclaw models list
```
