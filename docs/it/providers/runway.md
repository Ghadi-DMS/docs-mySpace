---
read_when:
    - Vuoi usare la generazione video Runway in OpenClaw
    - Hai bisogno della configurazione della chiave API/env di Runway
    - Vuoi rendere Runway il provider video predefinito
summary: Configurazione della generazione video Runway in OpenClaw
title: Runway
x-i18n:
    generated_at: "2026-04-06T03:11:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: bc615d1a26f7a4b890d29461e756690c858ecb05024cf3c4d508218022da6e76
    source_path: providers/runway.md
    workflow: 15
---

# Runway

OpenClaw include un provider bundled `runway` per la generazione video ospitata.

- ID provider: `runway`
- Autenticazione: `RUNWAYML_API_SECRET` (canonico) o `RUNWAY_API_KEY`
- API: generazione video basata su task di Runway (`GET /v1/tasks/{id}` polling)

## Avvio rapido

1. Imposta la chiave API:

```bash
openclaw onboard --auth-choice runway-api-key
```

2. Imposta Runway come provider video predefinito:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "runway/gen4.5"
```

3. Chiedi all'agente di generare un video. Runway verrà usato automaticamente.

## Modalità supportate

| Modalità       | Modello            | Input di riferimento     |
| -------------- | ------------------ | ------------------------ |
| Text-to-video  | `gen4.5` (predefinito) | Nessuno              |
| Image-to-video | `gen4.5`           | 1 immagine locale o remota |
| Video-to-video | `gen4_aleph`       | 1 video locale o remoto  |

- I riferimenti locali di immagini e video sono supportati tramite URI di dati.
- Video-to-video al momento richiede specificamente `runway/gen4_aleph`.
- Le esecuzioni solo testo al momento espongono i rapporti di aspetto `16:9` e `9:16`.

## Configurazione

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## Correlati

- [Generazione video](/tools/video-generation) -- parametri condivisi dello strumento, selezione del provider e comportamento asincrono
- [Riferimento di configurazione](/it/gateway/configuration-reference#agent-defaults)
