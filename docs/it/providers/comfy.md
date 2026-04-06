---
read_when:
    - Vuoi usare workflow ComfyUI locali con OpenClaw
    - Vuoi usare Comfy Cloud con workflow di immagini, video o musica
    - Hai bisogno delle chiavi di configurazione del plugin comfy bundled
summary: Configurazione della generazione di immagini, video e musica con workflow ComfyUI in OpenClaw
title: ComfyUI
x-i18n:
    generated_at: "2026-04-06T03:10:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: e645f32efdffdf4cd498684f1924bb953a014d3656b48f4b503d64e38c61ba9c
    source_path: providers/comfy.md
    workflow: 15
---

# ComfyUI

OpenClaw include un plugin bundled `comfy` per esecuzioni ComfyUI guidate da workflow.

- Provider: `comfy`
- Modelli: `comfy/workflow`
- Superfici condivise: `image_generate`, `video_generate`, `music_generate`
- Autenticazione: nessuna per ComfyUI locale; `COMFY_API_KEY` o `COMFY_CLOUD_API_KEY` per Comfy Cloud
- API: ComfyUI `/prompt` / `/history` / `/view` e Comfy Cloud `/api/*`

## Cosa supporta

- Generazione di immagini da un workflow JSON
- Modifica di immagini con 1 immagine di riferimento caricata
- Generazione di video da un workflow JSON
- Generazione di video con 1 immagine di riferimento caricata
- Generazione di musica o audio tramite lo strumento condiviso `music_generate`
- Download dell'output da un nodo configurato o da tutti i nodi di output corrispondenti

Il plugin bundled è guidato da workflow, quindi OpenClaw non prova a mappare
controlli generici come `size`, `aspectRatio`, `resolution`, `durationSeconds` o controlli in stile TTS
sul tuo grafo.

## Layout di configurazione

Comfy supporta impostazioni di connessione condivise di primo livello più sezioni workflow
per capability specifiche:

```json5
{
  models: {
    providers: {
      comfy: {
        mode: "local",
        baseUrl: "http://127.0.0.1:8188",
        image: {
          workflowPath: "./workflows/flux-api.json",
          promptNodeId: "6",
          outputNodeId: "9",
        },
        video: {
          workflowPath: "./workflows/video-api.json",
          promptNodeId: "12",
          outputNodeId: "21",
        },
        music: {
          workflowPath: "./workflows/music-api.json",
          promptNodeId: "3",
          outputNodeId: "18",
        },
      },
    },
  },
}
```

Chiavi condivise:

- `mode`: `local` o `cloud`
- `baseUrl`: valore predefinito `http://127.0.0.1:8188` per local o `https://cloud.comfy.org` per cloud
- `apiKey`: alternativa inline opzionale alle variabili env
- `allowPrivateNetwork`: consente un `baseUrl` privato/LAN in modalità cloud

Chiavi per capability specifiche sotto `image`, `video` o `music`:

- `workflow` o `workflowPath`: obbligatorio
- `promptNodeId`: obbligatorio
- `promptInputName`: valore predefinito `text`
- `outputNodeId`: opzionale
- `pollIntervalMs`: opzionale
- `timeoutMs`: opzionale

Le sezioni image e video supportano anche:

- `inputImageNodeId`: obbligatorio quando passi un'immagine di riferimento
- `inputImageInputName`: valore predefinito `image`

## Compatibilità retroattiva

La configurazione image esistente di primo livello continua a funzionare:

```json5
{
  models: {
    providers: {
      comfy: {
        workflowPath: "./workflows/flux-api.json",
        promptNodeId: "6",
        outputNodeId: "9",
      },
    },
  },
}
```

OpenClaw tratta questa forma legacy come configurazione del workflow image.

## Workflow image

Imposta il modello image predefinito:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "comfy/workflow",
      },
    },
  },
}
```

Esempio di modifica con immagine di riferimento:

```json5
{
  models: {
    providers: {
      comfy: {
        image: {
          workflowPath: "./workflows/edit-api.json",
          promptNodeId: "6",
          inputImageNodeId: "7",
          inputImageInputName: "image",
          outputNodeId: "9",
        },
      },
    },
  },
}
```

## Workflow video

Imposta il modello video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "comfy/workflow",
      },
    },
  },
}
```

I workflow video Comfy attualmente supportano text-to-video e image-to-video tramite
il grafo configurato. OpenClaw non passa video di input nei workflow Comfy.

## Workflow music

Il plugin bundled registra un provider di generazione musicale per
output audio o musicali definiti dal workflow, esposti tramite lo strumento condiviso `music_generate`:

```text
/tool music_generate prompt="Warm ambient synth loop with soft tape texture"
```

Usa la sezione di configurazione `music` per puntare al JSON del tuo workflow audio e al
nodo di output.

## Comfy Cloud

Usa `mode: "cloud"` più uno tra:

- `COMFY_API_KEY`
- `COMFY_CLOUD_API_KEY`
- `models.providers.comfy.apiKey`

La modalità cloud continua a usare le stesse sezioni workflow `image`, `video` e `music`.

## Test live

Esiste una copertura live opzionale per il plugin bundled:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

Il test live salta i singoli casi image, video o music a meno che la sezione workflow
Comfy corrispondente non sia configurata.

## Correlati

- [Generazione di immagini](/it/tools/image-generation)
- [Generazione di video](/tools/video-generation)
- [Generazione di musica](/tools/music-generation)
- [Directory provider](/it/providers/index)
- [Riferimento della configurazione](/it/gateway/configuration-reference#agent-defaults)
