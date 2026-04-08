---
read_when:
    - Vuoi usare i modelli Google Gemini con OpenClaw
    - Hai bisogno del flusso di autenticazione con chiave API o OAuth
summary: Configurazione di Google Gemini (chiave API + OAuth, generazione di immagini, comprensione dei contenuti multimediali, ricerca web)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-08T08:12:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: fad2ff68987301bd86145fa6e10de8c7b38d5bd5dbcd13db9c883f7f5b9a4e01
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Il plugin Google fornisce accesso ai modelli Gemini tramite Google AI Studio, oltre a
generazione di immagini, comprensione dei contenuti multimediali (immagini/audio/video) e ricerca web tramite
Gemini Grounding.

- Provider: `google`
- Autenticazione: `GEMINI_API_KEY` o `GOOGLE_API_KEY`
- API: API Google Gemini
- Provider alternativo: `google-gemini-cli` (OAuth)

## Avvio rapido

1. Imposta la chiave API:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. Imposta un modello predefinito:

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## Esempio non interattivo

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## OAuth (Gemini CLI)

Un provider alternativo `google-gemini-cli` usa OAuth PKCE invece di una chiave API.
Si tratta di un'integrazione non ufficiale; alcuni utenti segnalano restrizioni
dell'account. Usala a tuo rischio.

- Modello predefinito: `google-gemini-cli/gemini-3-flash-preview`
- Alias: `gemini-cli`
- Prerequisito di installazione: Gemini CLI locale disponibile come `gemini`
  - Homebrew: `brew install gemini-cli`
  - npm: `npm install -g @google/gemini-cli`
- Accesso:

```bash
openclaw models auth login --provider google-gemini-cli --set-default
```

Variabili di ambiente:

- `OPENCLAW_GEMINI_OAUTH_CLIENT_ID`
- `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET`

(O le varianti `GEMINI_CLI_*`.)

Se le richieste OAuth di Gemini CLI falliscono dopo l'accesso, imposta
`GOOGLE_CLOUD_PROJECT` o `GOOGLE_CLOUD_PROJECT_ID` sull'host del gateway e
riprova.

Se l'accesso fallisce prima che inizi il flusso nel browser, assicurati che il comando locale `gemini`
sia installato e presente nel `PATH`. OpenClaw supporta sia le installazioni tramite Homebrew
sia le installazioni npm globali, inclusi i comuni layout Windows/npm.

Note sull'uso di Gemini CLI JSON:

- Il testo della risposta proviene dal campo JSON `response` della CLI.
- Le informazioni di utilizzo usano `stats` come fallback quando la CLI lascia vuoto `usage`.
- `stats.cached` viene normalizzato in `cacheRead` di OpenClaw.
- Se `stats.input` è mancante, OpenClaw ricava i token di input da
  `stats.input_tokens - stats.cached`.

## Capacità

| Capacità               | Supportato        |
| ---------------------- | ----------------- |
| Completamenti chat     | Sì                |
| Generazione di immagini| Sì                |
| Generazione musicale   | Sì                |
| Comprensione immagini  | Sì                |
| Trascrizione audio     | Sì                |
| Comprensione video     | Sì                |
| Ricerca web (Grounding)| Sì                |
| Thinking/reasoning     | Sì (Gemini 3.1+)  |
| Modelli Gemma 4        | Sì                |

I modelli Gemma 4 (ad esempio `gemma-4-26b-a4b-it`) supportano la modalità thinking. OpenClaw riscrive `thinkingBudget` in un `thinkingLevel` Google supportato per Gemma 4. Impostare thinking su `off` mantiene il thinking disabilitato invece di mapparlo a `MINIMAL`.

## Riutilizzo diretto della cache Gemini

Per le esecuzioni API Gemini dirette (`api: "google-generative-ai"`), OpenClaw ora
passa un handle `cachedContent` configurato alle richieste Gemini.

- Configura i parametri per modello o globali con
  `cachedContent` o il legacy `cached_content`
- Se sono presenti entrambi, `cachedContent` ha la precedenza
- Valore di esempio: `cachedContents/prebuilt-context`
- L'utilizzo con cache hit di Gemini viene normalizzato in `cacheRead` di OpenClaw da
  `cachedContentTokenCount` upstream

Esempio:

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## Generazione di immagini

Il provider di generazione immagini `google` incluso usa per impostazione predefinita
`google/gemini-3.1-flash-image-preview`.

- Supporta anche `google/gemini-3-pro-image-preview`
- Generazione: fino a 4 immagini per richiesta
- Modalità modifica: abilitata, fino a 5 immagini di input
- Controlli geometrici: `size`, `aspectRatio` e `resolution`

Il provider `google-gemini-cli`, solo OAuth, è una superficie separata
per l'inferenza testuale. La generazione di immagini, la comprensione dei contenuti multimediali e Gemini Grounding restano sul
provider id `google`.

Per usare Google come provider di immagini predefinito:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

Vedi [Image Generation](/it/tools/image-generation) per i parametri
condivisi dello strumento, la selezione del provider e il comportamento di failover.

## Generazione video

Il plugin `google` incluso registra anche la generazione video tramite lo strumento condiviso
`video_generate`.

- Modello video predefinito: `google/veo-3.1-fast-generate-preview`
- Modalità: text-to-video, image-to-video e flussi con riferimento a un singolo video
- Supporta `aspectRatio`, `resolution` e `audio`
- Limite attuale della durata: **da 4 a 8 secondi**

Per usare Google come provider video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

Vedi [Video Generation](/it/tools/video-generation) per i parametri
condivisi dello strumento, la selezione del provider e il comportamento di failover.

## Generazione musicale

Il plugin `google` incluso registra anche la generazione musicale tramite lo strumento condiviso
`music_generate`.

- Modello musicale predefinito: `google/lyria-3-clip-preview`
- Supporta anche `google/lyria-3-pro-preview`
- Controlli del prompt: `lyrics` e `instrumental`
- Formato di output: `mp3` per impostazione predefinita, più `wav` su `google/lyria-3-pro-preview`
- Input di riferimento: fino a 10 immagini
- Le esecuzioni supportate da sessione vengono separate tramite il flusso condiviso di task/stato, incluso `action: "status"`

Per usare Google come provider musicale predefinito:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

Vedi [Music Generation](/it/tools/music-generation) per i parametri
condivisi dello strumento, la selezione del provider e il comportamento di failover.

## Nota sull'ambiente

Se il Gateway viene eseguito come demone (launchd/systemd), assicurati che `GEMINI_API_KEY`
sia disponibile per quel processo (ad esempio, in `~/.openclaw/.env` o tramite
`env.shellEnv`).
