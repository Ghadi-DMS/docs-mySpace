---
read_when:
    - Vuoi usare la generazione video Alibaba Wan in OpenClaw
    - Hai bisogno della configurazione della chiave API di Model Studio o DashScope per la generazione video
summary: Generazione video Wan di Alibaba Model Studio in OpenClaw
title: Alibaba Model Studio
x-i18n:
    generated_at: "2026-04-06T03:10:09Z"
    model: gpt-5.4
    provider: openai
    source_hash: 97a1eddc7cbd816776b9368f2a926b5ef9ee543f08d151a490023736f67dc635
    source_path: providers/alibaba.md
    workflow: 15
---

# Alibaba Model Studio

OpenClaw include un provider bundled `alibaba` per la generazione video dei modelli Wan su
Alibaba Model Studio / DashScope.

- Provider: `alibaba`
- Autenticazione preferita: `MODELSTUDIO_API_KEY`
- Accettate anche: `DASHSCOPE_API_KEY`, `QWEN_API_KEY`
- API: generazione video asincrona DashScope / Model Studio

## Avvio rapido

1. Imposta una chiave API:

```bash
openclaw onboard --auth-choice qwen-standard-api-key
```

2. Imposta un modello video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "alibaba/wan2.6-t2v",
      },
    },
  },
}
```

## Modelli Wan integrati

Il provider bundled `alibaba` attualmente registra:

- `alibaba/wan2.6-t2v`
- `alibaba/wan2.6-i2v`
- `alibaba/wan2.6-r2v`
- `alibaba/wan2.6-r2v-flash`
- `alibaba/wan2.7-r2v`

## Limiti attuali

- Fino a **1** video di output per richiesta
- Fino a **1** immagine di input
- Fino a **4** video di input
- Durata fino a **10 secondi**
- Supporta `size`, `aspectRatio`, `resolution`, `audio` e `watermark`
- La modalità immagine/video di riferimento richiede attualmente **URL http(s) remoti**

## Relazione con Qwen

Anche il provider bundled `qwen` usa endpoint DashScope ospitati da Alibaba per
la generazione video Wan. Usa:

- `qwen/...` quando vuoi la superficie canonica del provider Qwen
- `alibaba/...` quando vuoi la superficie video Wan diretta, di proprietà del vendor

## Correlati

- [Generazione video](/tools/video-generation)
- [Qwen](/it/providers/qwen)
- [Riferimento di configurazione](/it/gateway/configuration-reference#agent-defaults)
