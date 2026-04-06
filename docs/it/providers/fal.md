---
read_when:
    - Vuoi usare la generazione di immagini fal in OpenClaw
    - Hai bisogno del flusso di autenticazione FAL_KEY
    - Vuoi i valori predefiniti fal per image_generate o video_generate
summary: Configurazione di generazione di immagini e video fal in OpenClaw
title: fal
x-i18n:
    generated_at: "2026-04-06T03:10:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1922907d2c8360c5877a56495323d54bd846d47c27a801155e3d11e3f5706fbd
    source_path: providers/fal.md
    workflow: 15
---

# fal

OpenClaw include un provider `fal` integrato per la generazione ospitata di immagini e video.

- Provider: `fal`
- Auth: `FAL_KEY` (canonico; anche `FAL_API_KEY` funziona come fallback)
- API: endpoint modello fal

## Guida rapida

1. Imposta la chiave API:

```bash
openclaw onboard --auth-choice fal-api-key
```

2. Imposta un modello di immagini predefinito:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## Generazione di immagini

Il provider integrato di image-generation `fal` usa come valore predefinito
`fal/fal-ai/flux/dev`.

- Generate: fino a 4 immagini per richiesta
- Modalità edit: abilitata, 1 immagine di riferimento
- Supporta `size`, `aspectRatio` e `resolution`
- Limitazione attuale dell'edit: l'endpoint di modifica immagini fal **non** supporta
  override di `aspectRatio`

Per usare fal come provider di immagini predefinito:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## Generazione di video

Il provider integrato di video-generation `fal` usa come valore predefinito
`fal/fal-ai/minimax/video-01-live`.

- Modalità: flussi da testo a video e con immagine singola di riferimento
- Runtime: flusso submit/status/result supportato da coda per job di lunga durata

Per usare fal come provider video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/fal-ai/minimax/video-01-live",
      },
    },
  },
}
```

## Correlati

- [Generazione di immagini](/it/tools/image-generation)
- [Generazione di video](/tools/video-generation)
- [Riferimento configurazione](/it/gateway/configuration-reference#agent-defaults)
