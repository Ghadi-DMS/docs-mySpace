---
read_when:
    - Vuoi la generazione media Vydra in OpenClaw
    - Hai bisogno di indicazioni per la configurazione della chiave API Vydra
summary: Usa immagini, video e speech di Vydra in OpenClaw
title: Vydra
x-i18n:
    generated_at: "2026-04-06T03:11:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0fe999e8a5414b8a31a6d7d127bc6bcfc3b4492b8f438ab17dfa9680c5b079b7
    source_path: providers/vydra.md
    workflow: 15
---

# Vydra

Il plugin Vydra bundled aggiunge:

- generazione di immagini tramite `vydra/grok-imagine`
- generazione video tramite `vydra/veo3` e `vydra/kling`
- sintesi vocale tramite il percorso TTS di Vydra supportato da ElevenLabs

OpenClaw usa la stessa `VYDRA_API_KEY` per tutte e tre le capability.

## URL base importante

Usa `https://www.vydra.ai/api/v1`.

L'host apex di Vydra (`https://vydra.ai/api/v1`) al momento reindirizza a `www`. Alcuni client HTTP eliminano `Authorization` in quel reindirizzamento cross-host, trasformando una chiave API valida in un errore auth fuorviante. Il plugin bundled usa direttamente l'URL base `www` per evitarlo.

## Configurazione

Onboarding interattivo:

```bash
openclaw onboard --auth-choice vydra-api-key
```

Oppure imposta direttamente la variabile d'ambiente:

```bash
export VYDRA_API_KEY="vydra_live_..."
```

## Generazione di immagini

Modello immagine predefinito:

- `vydra/grok-imagine`

Impostalo come provider di immagini predefinito:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "vydra/grok-imagine",
      },
    },
  },
}
```

L'attuale supporto bundled è solo text-to-image. I percorsi di modifica ospitati da Vydra si aspettano URL immagine remoti e OpenClaw non aggiunge ancora un bridge di upload specifico per Vydra nel plugin bundled.

Vedi [Generazione di immagini](/it/tools/image-generation) per il comportamento condiviso dello strumento.

## Generazione video

Modelli video registrati:

- `vydra/veo3` per text-to-video
- `vydra/kling` per image-to-video

Imposta Vydra come provider video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "vydra/veo3",
      },
    },
  },
}
```

Note:

- `vydra/veo3` è bundled solo come text-to-video.
- `vydra/kling` al momento richiede un URL immagine remoto di riferimento. I caricamenti di file locali vengono rifiutati in anticipo.
- Il plugin bundled rimane conservativo e non inoltra controlli di stile non documentati come rapporto di aspetto, risoluzione, watermark o audio generato.

Vedi [Generazione video](/tools/video-generation) per il comportamento condiviso dello strumento.

## Sintesi vocale

Imposta Vydra come provider speech:

```json5
{
  messages: {
    tts: {
      provider: "vydra",
      providers: {
        vydra: {
          apiKey: "${VYDRA_API_KEY}",
          voiceId: "21m00Tcm4TlvDq8ikWAM",
        },
      },
    },
  },
}
```

Valori predefiniti:

- modello: `elevenlabs/tts`
- ID voce: `21m00Tcm4TlvDq8ikWAM`

Il plugin bundled al momento espone una voce predefinita affidabile nota e restituisce file audio MP3.

## Correlati

- [Directory dei provider](/it/providers/index)
- [Generazione di immagini](/it/tools/image-generation)
- [Generazione video](/tools/video-generation)
