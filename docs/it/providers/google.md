---
read_when:
    - Vuoi usare i modelli Google Gemini con OpenClaw
    - Hai bisogno del flusso di autenticazione con chiave API
summary: Configurazione di Google Gemini (chiave API, generazione di immagini, comprensione dei media, web search)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-06T03:11:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 358d33a68275b01ebd916a3621dd651619cb9a1d062e2fb6196a7f3c501c015a
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Il plugin Google fornisce accesso ai modelli Gemini tramite Google AI Studio, oltre a
generazione di immagini, comprensione dei media (immagine/audio/video) e web search tramite
Gemini Grounding.

- Provider: `google`
- Autenticazione: `GEMINI_API_KEY` o `GOOGLE_API_KEY`
- API: Google Gemini API

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

## Capability

| Capability             | Supportata        |
| ---------------------- | ----------------- |
| Chat completions       | Sì                |
| Generazione di immagini | Sì               |
| Generazione musicale   | Sì                |
| Comprensione delle immagini | Sì           |
| Trascrizione audio     | Sì                |
| Comprensione video     | Sì                |
| Web search (Grounding) | Sì                |
| Thinking/reasoning     | Sì (Gemini 3.1+)  |

## Riutilizzo diretto della cache Gemini

Per le esecuzioni dirette dell'API Gemini (`api: "google-generative-ai"`), OpenClaw ora
passa un handle `cachedContent` configurato alle richieste Gemini.

- Configura i parametri per modello o globali con
  `cachedContent` oppure con il legacy `cached_content`
- Se sono presenti entrambi, `cachedContent` ha la precedenza
- Valore di esempio: `cachedContents/prebuilt-context`
- L'uso del cache-hit Gemini viene normalizzato in OpenClaw `cacheRead` a partire da
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

Il provider bundled `google` per la generazione di immagini usa per impostazione predefinita
`google/gemini-3.1-flash-image-preview`.

- Supporta anche `google/gemini-3-pro-image-preview`
- Generazione: fino a 4 immagini per richiesta
- Modalità modifica: abilitata, fino a 5 immagini di input
- Controlli geometrici: `size`, `aspectRatio` e `resolution`

La generazione di immagini, la comprensione dei media e Gemini Grounding restano tutte sul
provider con ID `google`.

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

Vedi [Generazione di immagini](/it/tools/image-generation) per i parametri
condivisi dello strumento, la selezione del provider e il comportamento di failover.

## Generazione video

Il plugin bundled `google` registra anche la generazione video tramite lo strumento condiviso
`video_generate`.

- Modello video predefinito: `google/veo-3.1-fast-generate-preview`
- Modalità: text-to-video, image-to-video e flussi di riferimento con singolo video
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

Vedi [Generazione video](/tools/video-generation) per i parametri
condivisi dello strumento, la selezione del provider e il comportamento di failover.

## Generazione musicale

Il plugin bundled `google` registra anche la generazione musicale tramite lo strumento condiviso
`music_generate`.

- Modello musicale predefinito: `google/lyria-3-clip-preview`
- Supporta anche `google/lyria-3-pro-preview`
- Controlli del prompt: `lyrics` e `instrumental`
- Formato di output: `mp3` per impostazione predefinita, più `wav` su `google/lyria-3-pro-preview`
- Input di riferimento: fino a 10 immagini
- Le esecuzioni supportate da sessione vengono scollegate tramite il flusso condiviso task/status, incluso `action: "status"`

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

Vedi [Generazione musicale](/tools/music-generation) per i parametri
condivisi dello strumento, la selezione del provider e il comportamento di failover.

## Nota sull'ambiente

Se il Gateway viene eseguito come daemon (launchd/systemd), assicurati che `GEMINI_API_KEY`
sia disponibile per quel processo (ad esempio in `~/.openclaw/.env` o tramite
`env.shellEnv`).
