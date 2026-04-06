---
read_when:
    - Generazione di immagini tramite l'agente
    - Configurazione dei provider e dei modelli per la generazione di immagini
    - Comprensione dei parametri dello strumento `image_generate`
summary: Genera e modifica immagini usando i provider configurati (OpenAI, Google Gemini, fal, MiniMax, ComfyUI, Vydra)
title: Generazione di immagini
x-i18n:
    generated_at: "2026-04-06T03:12:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: dde416dd1441a06605db85b5813cf61ccfc525813d6db430b7b7dfa53d6a3134
    source_path: tools/image-generation.md
    workflow: 15
---

# Generazione di immagini

Lo strumento `image_generate` consente all'agente di creare e modificare immagini usando i provider configurati. Le immagini generate vengono recapitate automaticamente come allegati multimediali nella risposta dell'agente.

<Note>
Lo strumento appare solo quando è disponibile almeno un provider di generazione di immagini. Se non vedi `image_generate` tra gli strumenti del tuo agente, configura `agents.defaults.imageGenerationModel` o imposta una chiave API del provider.
</Note>

## Avvio rapido

1. Imposta una chiave API per almeno un provider (ad esempio `OPENAI_API_KEY` o `GEMINI_API_KEY`).
2. Facoltativamente imposta il tuo modello preferito:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

3. Chiedi all'agente: _"Genera un'immagine di una simpatica mascotte aragosta."_

L'agente chiama automaticamente `image_generate`. Non serve alcuna allowlist degli strumenti — è abilitato per impostazione predefinita quando un provider è disponibile.

## Provider supportati

| Provider | Modello predefinito             | Supporto modifica                  | Chiave API                                             |
| -------- | ------------------------------- | ---------------------------------- | ------------------------------------------------------ |
| OpenAI   | `gpt-image-1`                   | Sì (fino a 5 immagini)             | `OPENAI_API_KEY`                                       |
| Google   | `gemini-3.1-flash-image-preview` | Sì                               | `GEMINI_API_KEY` o `GOOGLE_API_KEY`                    |
| fal      | `fal-ai/flux/dev`               | Sì                                 | `FAL_KEY`                                              |
| MiniMax  | `image-01`                      | Sì (riferimento del soggetto)      | `MINIMAX_API_KEY` o OAuth MiniMax (`minimax-portal`)   |
| ComfyUI  | `workflow`                      | Sì (1 immagine, configurata dal workflow) | `COMFY_API_KEY` o `COMFY_CLOUD_API_KEY` per il cloud |
| Vydra    | `grok-imagine`                  | No                                 | `VYDRA_API_KEY`                                        |

Usa `action: "list"` per ispezionare provider e modelli disponibili a runtime:

```
/tool image_generate action=list
```

## Parametri dello strumento

| Parametro     | Tipo     | Descrizione                                                                        |
| ------------- | -------- | ---------------------------------------------------------------------------------- |
| `prompt`      | string   | Prompt di generazione dell'immagine (obbligatorio per `action: "generate"`)       |
| `action`      | string   | `"generate"` (predefinito) o `"list"` per ispezionare i provider                  |
| `model`       | string   | Override provider/modello, ad esempio `openai/gpt-image-1`                         |
| `image`       | string   | Singolo percorso immagine di riferimento o URL per la modalità modifica            |
| `images`      | string[] | Più immagini di riferimento per la modalità modifica (fino a 5)                    |
| `size`        | string   | Suggerimento di dimensione: `1024x1024`, `1536x1024`, `1024x1536`, `1024x1792`, `1792x1024` |
| `aspectRatio` | string   | Rapporto di aspetto: `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` |
| `resolution`  | string   | Suggerimento di risoluzione: `1K`, `2K` o `4K`                                     |
| `count`       | number   | Numero di immagini da generare (1–4)                                               |
| `filename`    | string   | Suggerimento per il nome del file di output                                        |

Non tutti i provider supportano tutti i parametri. Lo strumento passa ciò che ciascun provider supporta, ignora il resto e segnala gli override scartati nel risultato dello strumento.

## Configurazione

### Selezione del modello

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
        fallbacks: ["google/gemini-3.1-flash-image-preview", "fal/fal-ai/flux/dev"],
      },
    },
  },
}
```

### Ordine di selezione del provider

Quando genera un'immagine, OpenClaw prova i provider in questo ordine:

1. Il parametro **`model`** della chiamata allo strumento (se l'agente ne specifica uno)
2. **`imageGenerationModel.primary`** dalla configurazione
3. **`imageGenerationModel.fallbacks`** nell'ordine indicato
4. **Rilevamento automatico** — usa solo i valori predefiniti dei provider supportati dall'autenticazione:
   - prima il provider predefinito corrente
   - poi i restanti provider di generazione di immagini registrati in ordine di ID provider

Se un provider fallisce (errore di autenticazione, rate limit, ecc.), il candidato successivo viene provato automaticamente. Se falliscono tutti, l'errore include i dettagli di ogni tentativo.

Note:

- Il rilevamento automatico è consapevole dell'autenticazione. Un provider predefinito entra nell'elenco dei candidati solo quando OpenClaw può effettivamente autenticare quel provider.
- Usa `action: "list"` per ispezionare i provider attualmente registrati, i loro
  modelli predefiniti e i suggerimenti sulle env var di autenticazione.

### Modifica delle immagini

OpenAI, Google, fal, MiniMax e ComfyUI supportano la modifica di immagini di riferimento. Passa un percorso immagine di riferimento o un URL:

```
"Genera una versione acquerello di questa foto" + image: "/path/to/photo.jpg"
```

OpenAI e Google supportano fino a 5 immagini di riferimento tramite il parametro `images`. fal, MiniMax e ComfyUI ne supportano 1.

La generazione di immagini MiniMax è disponibile tramite entrambi i percorsi auth MiniMax bundled:

- `minimax/image-01` per configurazioni con chiave API
- `minimax-portal/image-01` per configurazioni OAuth

## Capability dei provider

| Capability            | OpenAI               | Google               | fal                 | MiniMax                    | ComfyUI                            | Vydra |
| --------------------- | -------------------- | -------------------- | ------------------- | -------------------------- | ---------------------------------- | ----- |
| Generazione           | Sì (fino a 4)        | Sì (fino a 4)        | Sì (fino a 4)       | Sì (fino a 9)              | Sì (output definiti dal workflow)  | Sì (1) |
| Modifica/riferimento  | Sì (fino a 5 immagini) | Sì (fino a 5 immagini) | Sì (1 immagine)  | Sì (1 immagine, riferimento soggetto) | Sì (1 immagine, configurata dal workflow) | No |
| Controllo dimensione  | Sì                   | Sì                   | Sì                  | No                         | No                                 | No |
| Rapporto di aspetto   | No                   | Sì                   | Sì (solo generazione) | Sì                      | No                                 | No |
| Risoluzione (1K/2K/4K) | No                  | Sì                   | Sì                  | No                         | No                                 | No |

## Correlati

- [Panoramica degli strumenti](/it/tools) — tutti gli strumenti agente disponibili
- [fal](/providers/fal) — configurazione del provider immagini e video fal
- [ComfyUI](/providers/comfy) — configurazione del workflow locale ComfyUI e Comfy Cloud
- [Google (Gemini)](/it/providers/google) — configurazione del provider immagini Gemini
- [MiniMax](/it/providers/minimax) — configurazione del provider immagini MiniMax
- [OpenAI](/it/providers/openai) — configurazione del provider OpenAI Images
- [Vydra](/providers/vydra) — configurazione di immagini, video e speech Vydra
- [Riferimento di configurazione](/it/gateway/configuration-reference#agent-defaults) — configurazione `imageGenerationModel`
- [Modelli](/it/concepts/models) — configurazione dei modelli e failover
