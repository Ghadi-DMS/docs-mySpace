---
read_when:
    - Generazione di video tramite l'agent
    - Configurazione dei provider e dei modelli di generazione video
    - Comprensione dei parametri dello strumento video_generate
summary: Genera video da testo, immagini o video esistenti usando 12 backend provider
title: Generazione di video
x-i18n:
    generated_at: "2026-04-06T03:13:31Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4afec87368232221db1aa5a3980254093d6a961b17271b2dcbf724e6bd455b16
    source_path: tools/video-generation.md
    workflow: 15
---

# Generazione di video

Gli agent OpenClaw possono generare video da prompt di testo, immagini di riferimento o video esistenti. Sono supportati dodici backend provider, ciascuno con opzioni di modello, modalità di input e set di funzionalità differenti. L'agent sceglie automaticamente il provider corretto in base alla tua configurazione e alle API key disponibili.

<Note>
Lo strumento `video_generate` compare solo quando è disponibile almeno un provider di generazione video. Se non lo vedi tra gli strumenti del tuo agent, imposta una API key del provider o configura `agents.defaults.videoGenerationModel`.
</Note>

## Guida rapida

1. Imposta una API key per un provider supportato:

```bash
export GEMINI_API_KEY="your-key"
```

2. Facoltativamente, fissa un modello predefinito:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
```

3. Chiedi all'agent:

> Genera un video cinematografico di 5 secondi di un'aragosta amichevole che fa surf al tramonto.

L'agent chiama automaticamente `video_generate`. Non è necessaria alcuna allowlist degli strumenti.

## Cosa succede quando generi un video

La generazione video è asincrona. Quando l'agent chiama `video_generate` in una sessione:

1. OpenClaw invia la richiesta al provider e restituisce immediatamente un ID attività.
2. Il provider elabora il job in background (tipicamente da 30 secondi a 5 minuti a seconda del provider e della risoluzione).
3. Quando il video è pronto, OpenClaw riattiva la stessa sessione con un evento interno di completamento.
4. L'agent pubblica il video completato nella conversazione originale.

Mentre un job è in corso, chiamate duplicate a `video_generate` nella stessa sessione restituiscono lo stato corrente dell'attività invece di avviare un'altra generazione. Usa `openclaw tasks list` o `openclaw tasks show <taskId>` per controllare l'avanzamento dalla CLI.

Al di fuori delle esecuzioni dell'agent supportate da sessione (ad esempio invocazioni dirette dello strumento), lo strumento usa come fallback la generazione inline e restituisce il percorso finale del media nello stesso turno.

## Provider supportati

| Provider | Modello predefinito             | Testo | Riferimento immagine | Riferimento video | API key                                  |
| -------- | ------------------------------- | ----- | -------------------- | ----------------- | ---------------------------------------- |
| Alibaba  | `wan2.6-t2v`                    | Sì    | Sì (URL remoto)      | Sì (URL remoto)   | `MODELSTUDIO_API_KEY`                    |
| BytePlus | `seedance-1-0-lite-t2v-250428`  | Sì    | 1 immagine           | No                | `BYTEPLUS_API_KEY`                       |
| ComfyUI  | `workflow`                      | Sì    | 1 immagine           | No                | `COMFY_API_KEY` o `COMFY_CLOUD_API_KEY` |
| fal      | `fal-ai/minimax/video-01-live`  | Sì    | 1 immagine           | No                | `FAL_KEY`                                |
| Google   | `veo-3.1-fast-generate-preview` | Sì    | 1 immagine           | 1 video           | `GEMINI_API_KEY`                         |
| MiniMax  | `MiniMax-Hailuo-2.3`            | Sì    | 1 immagine           | No                | `MINIMAX_API_KEY`                        |
| OpenAI   | `sora-2`                        | Sì    | 1 immagine           | 1 video           | `OPENAI_API_KEY`                         |
| Qwen     | `wan2.6-t2v`                    | Sì    | Sì (URL remoto)      | Sì (URL remoto)   | `QWEN_API_KEY`                           |
| Runway   | `gen4.5`                        | Sì    | 1 immagine           | 1 video           | `RUNWAYML_API_SECRET`                    |
| Together | `Wan-AI/Wan2.2-T2V-A14B`        | Sì    | 1 immagine           | No                | `TOGETHER_API_KEY`                       |
| Vydra    | `veo3`                          | Sì    | 1 immagine (`kling`) | No                | `VYDRA_API_KEY`                          |
| xAI      | `grok-imagine-video`            | Sì    | 1 immagine           | 1 video           | `XAI_API_KEY`                            |

Alcuni provider accettano variabili env API key aggiuntive o alternative. Vedi le singole [pagine provider](#related) per i dettagli.

Esegui `video_generate action=list` per ispezionare i provider e i modelli disponibili a runtime.

## Parametri dello strumento

### Obbligatori

| Parametro | Tipo   | Descrizione                                                                    |
| --------- | ------ | ------------------------------------------------------------------------------ |
| `prompt`  | string | Descrizione testuale del video da generare (obbligatoria per `action: "generate"`) |

### Input di contenuto

| Parametro | Tipo     | Descrizione                              |
| --------- | -------- | ---------------------------------------- |
| `image`   | string   | Singola immagine di riferimento (percorso o URL) |
| `images`  | string[] | Immagini di riferimento multiple (fino a 5) |
| `video`   | string   | Singolo video di riferimento (percorso o URL) |
| `videos`  | string[] | Video di riferimento multipli (fino a 4) |

### Controlli di stile

| Parametro         | Tipo    | Descrizione                                                              |
| ----------------- | ------- | ------------------------------------------------------------------------ |
| `aspectRatio`     | string  | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`  |
| `resolution`      | string  | `480P`, `720P` o `1080P`                                                 |
| `durationSeconds` | number  | Durata target in secondi (arrotondata al valore supportato più vicino dal provider) |
| `size`            | string  | Suggerimento di dimensione quando il provider lo supporta                |
| `audio`           | boolean | Abilita l'audio generato quando supportato                              |
| `watermark`       | boolean | Attiva/disattiva la filigrana del provider quando supportata            |

### Avanzati

| Parametro  | Tipo   | Descrizione                                      |
| ---------- | ------ | ------------------------------------------------ |
| `action`   | string | `"generate"` (predefinito), `"status"` o `"list"` |
| `model`    | string | Override provider/model (ad esempio `runway/gen4.5`) |
| `filename` | string | Suggerimento per il nome del file di output      |

Non tutti i provider supportano tutti i parametri. Gli override non supportati vengono ignorati su base best-effort e segnalati come avvisi nel risultato dello strumento. I limiti rigidi delle capability (come troppi input di riferimento) falliscono prima dell'invio.

## Azioni

- **generate** (predefinita) -- crea un video dal prompt fornito e dagli eventuali input di riferimento.
- **status** -- controlla lo stato dell'attività video in corso per la sessione corrente senza avviare un'altra generazione.
- **list** -- mostra provider, modelli disponibili e relative capability.

## Selezione del modello

Quando genera un video, OpenClaw risolve il modello in questo ordine:

1. **Parametro dello strumento `model`** -- se l'agent ne specifica uno nella chiamata.
2. **`videoGenerationModel.primary`** -- dalla configurazione.
3. **`videoGenerationModel.fallbacks`** -- provati in ordine.
4. **Rilevamento automatico** -- usa i provider che hanno auth valida, iniziando dal provider predefinito corrente, poi i provider rimanenti in ordine alfabetico.

Se un provider fallisce, viene provato automaticamente il candidato successivo. Se tutti i candidati falliscono, l'errore include i dettagli di ogni tentativo.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
      },
    },
  },
}
```

## Note sui provider

| Provider | Note                                                                                                                                    |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba  | Usa l'endpoint asincrono DashScope/Model Studio. Le immagini e i video di riferimento devono essere URL remoti `http(s)`.              |
| BytePlus | Solo un'immagine di riferimento.                                                                                                         |
| ComfyUI  | Esecuzione locale o cloud guidata da workflow. Supporta text-to-video e image-to-video tramite il grafo configurato.                   |
| fal      | Usa un flusso supportato da coda per job di lunga durata. Solo un'immagine di riferimento.                                              |
| Google   | Usa Gemini/Veo. Supporta un'immagine o un video di riferimento.                                                                          |
| MiniMax  | Solo un'immagine di riferimento.                                                                                                         |
| OpenAI   | Viene inoltrato solo l'override `size`. Gli altri override di stile (`aspectRatio`, `resolution`, `audio`, `watermark`) vengono ignorati con un avviso. |
| Qwen     | Stesso backend DashScope di Alibaba. Gli input di riferimento devono essere URL remoti `http(s)`; i file locali vengono rifiutati in anticipo. |
| Runway   | Supporta file locali tramite data URI. Video-to-video richiede `runway/gen4_aleph`. Le esecuzioni solo testo espongono rapporti d'aspetto `16:9` e `9:16`. |
| Together | Solo un'immagine di riferimento.                                                                                                         |
| Vydra    | Usa direttamente `https://www.vydra.ai/api/v1` per evitare redirect che perdono auth. `veo3` è integrato solo come text-to-video; `kling` richiede un URL immagine remoto. |
| xAI      | Supporta flussi text-to-video, image-to-video e modifica/estensione di video remoti.                                                    |

## Configurazione

Imposta il modello di generazione video predefinito nella configurazione OpenClaw:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

Oppure tramite la CLI:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "qwen/wan2.6-t2v"
```

## Correlati

- [Panoramica degli strumenti](/it/tools)
- [Attività in background](/it/automation/tasks) -- tracciamento delle attività per la generazione video asincrona
- [Alibaba Model Studio](/providers/alibaba)
- [BytePlus](/providers/byteplus)
- [ComfyUI](/providers/comfy)
- [fal](/providers/fal)
- [Google (Gemini)](/it/providers/google)
- [MiniMax](/it/providers/minimax)
- [OpenAI](/it/providers/openai)
- [Qwen](/it/providers/qwen)
- [Runway](/providers/runway)
- [Together AI](/it/providers/together)
- [Vydra](/providers/vydra)
- [xAI](/it/providers/xai)
- [Riferimento configurazione](/it/gateway/configuration-reference#agent-defaults)
- [Modelli](/it/concepts/models)
