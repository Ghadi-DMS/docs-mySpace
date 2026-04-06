---
read_when:
    - Generazione di musica o audio tramite l'agente
    - Configurazione di provider e modelli per la generazione musicale
    - Comprendere i parametri dello strumento music_generate
summary: Generare musica con provider condivisi, inclusi i plugin supportati da workflow
title: Generazione musicale
x-i18n:
    generated_at: "2026-04-06T03:13:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: a03de8aa75cfb7248eb0c1d969fb2a6da06117967d097e6f6e95771d0f017ae1
    source_path: tools/music-generation.md
    workflow: 15
---

# Generazione musicale

Lo strumento `music_generate` consente all'agente di creare musica o audio tramite la
capacità condivisa di generazione musicale con provider configurati come Google,
MiniMax e ComfyUI configurato tramite workflow.

Per le sessioni agente supportate da provider condivisi, OpenClaw avvia la generazione musicale come
attività in background, la traccia nel registro delle attività, quindi riattiva di nuovo l'agente quando
la traccia è pronta in modo che l'agente possa pubblicare l'audio completato nel
canale originale.

<Note>
Lo strumento condiviso integrato compare solo quando è disponibile almeno un provider di generazione musicale. Se non vedi `music_generate` negli strumenti del tuo agente, configura `agents.defaults.musicGenerationModel` oppure imposta una chiave API del provider.
</Note>

## Avvio rapido

### Generazione supportata da provider condivisi

1. Imposta una chiave API per almeno un provider, ad esempio `GEMINI_API_KEY` oppure
   `MINIMAX_API_KEY`.
2. Facoltativamente imposta il tuo modello preferito:

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

3. Chiedi all'agente: _"Genera una traccia synthpop energica su un viaggio notturno
   attraverso una città al neon."_

L'agente chiama automaticamente `music_generate`. Non serve alcuna allowlist degli strumenti.

Per contesti sincroni diretti senza un'esecuzione dell'agente supportata da sessione, lo
strumento integrato continua a usare la generazione inline come fallback e restituisce il percorso finale del media nel
risultato dello strumento.

Esempi di prompt:

```text
Genera una traccia cinematografica per pianoforte con archi morbidi e senza voce.
```

```text
Genera un loop chiptune energico sul lancio di un razzo all'alba.
```

### Generazione Comfy basata su workflow

Il plugin bundled `comfy` si integra con lo strumento condiviso `music_generate` tramite
il registro dei provider di generazione musicale.

1. Configura `models.providers.comfy.music` con un workflow JSON e
   nodi prompt/output.
2. Se usi Comfy Cloud, imposta `COMFY_API_KEY` oppure `COMFY_CLOUD_API_KEY`.
3. Chiedi musica all'agente oppure chiama direttamente lo strumento.

Esempio:

```text
/tool music_generate prompt="Warm ambient synth loop with soft tape texture"
```

## Supporto bundled condiviso dei provider

| Provider | Modello predefinito    | Input di riferimento | Controlli supportati                                      | Chiave API                              |
| -------- | ---------------------- | -------------------- | --------------------------------------------------------- | --------------------------------------- |
| ComfyUI  | `workflow`             | Fino a 1 immagine    | Musica o audio definiti dal workflow                      | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| Google   | `lyria-3-clip-preview` | Fino a 10 immagini   | `lyrics`, `instrumental`, `format`                        | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax  | `music-2.5+`           | Nessuno              | `lyrics`, `instrumental`, `durationSeconds`, `format=mp3` | `MINIMAX_API_KEY`                      |

Usa `action: "list"` per esaminare i provider e i modelli condivisi disponibili a
runtime:

```text
/tool music_generate action=list
```

Usa `action: "status"` per controllare l'attività musicale attiva supportata da sessione:

```text
/tool music_generate action=status
```

Esempio di generazione diretta:

```text
/tool music_generate prompt="Dreamy lo-fi hip hop with vinyl texture and gentle rain" instrumental=true
```

## Parametri dello strumento integrato

| Parametro         | Tipo     | Descrizione                                                                                      |
| ----------------- | -------- | ------------------------------------------------------------------------------------------------ |
| `prompt`          | string   | Prompt di generazione musicale (obbligatorio per `action: "generate"`)                          |
| `action`          | string   | `"generate"` (predefinito), `"status"` per l'attività della sessione corrente, oppure `"list"` per esaminare i provider |
| `model`           | string   | Override provider/modello, ad esempio `google/lyria-3-pro-preview` o `comfy/workflow`           |
| `lyrics`          | string   | Testo facoltativo del brano quando il provider supporta input esplicito dei testi               |
| `instrumental`    | boolean  | Richiede output solo strumentale quando il provider lo supporta                                 |
| `image`           | string   | Percorso o URL di una singola immagine di riferimento                                            |
| `images`          | string[] | Più immagini di riferimento (fino a 10)                                                          |
| `durationSeconds` | number   | Durata target in secondi quando il provider supporta suggerimenti di durata                      |
| `format`          | string   | Suggerimento sul formato di output (`mp3` o `wav`) quando il provider lo supporta                |
| `filename`        | string   | Suggerimento per il nome file di output                                                          |

Non tutti i provider supportano tutti i parametri. OpenClaw continua comunque a convalidare i limiti rigidi
come il numero di input prima dell'invio, ma i suggerimenti facoltativi non supportati vengono
ignorati con un avviso quando il provider o il modello selezionato non possono rispettarli.

## Comportamento asincrono per il percorso supportato da provider condivisi

- Esecuzioni dell'agente supportate da sessione: `music_generate` crea un'attività in background, restituisce immediatamente una risposta avviata/dell'attività e pubblica la traccia completata più tardi in un messaggio di follow-up dell'agente.
- Prevenzione dei duplicati: mentre quell'attività in background è ancora `queued` oppure `running`, le chiamate successive a `music_generate` nella stessa sessione restituiscono lo stato dell'attività invece di avviare un'altra generazione.
- Lookup dello stato: usa `action: "status"` per controllare l'attività musicale attiva supportata da sessione senza avviarne una nuova.
- Tracciamento delle attività: usa `openclaw tasks list` oppure `openclaw tasks show <taskId>` per controllare lo stato queued, running e terminale della generazione.
- Riattivazione al completamento: OpenClaw inietta un evento interno di completamento nella stessa sessione in modo che il modello possa scrivere autonomamente il follow-up rivolto all'utente.
- Suggerimento del prompt: i turni utente/manuali successivi nella stessa sessione ricevono un piccolo suggerimento runtime quando è già in corso un'attività musicale così il modello non richiama ciecamente `music_generate`.
- Fallback senza sessione: i contesti diretti/locali senza una vera sessione agente continuano a essere eseguiti inline e restituiscono il risultato audio finale nello stesso turno.

## Configurazione

### Selezione del modello

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["minimax/music-2.5+"],
      },
    },
  },
}
```

### Ordine di selezione del provider

Quando genera musica, OpenClaw prova i provider in questo ordine:

1. Parametro `model` della chiamata dello strumento, se l'agente ne specifica uno
2. `musicGenerationModel.primary` dalla configurazione
3. `musicGenerationModel.fallbacks` in ordine
4. Auto-rilevamento usando solo valori predefiniti del provider supportati dall'autenticazione:
   - prima il provider predefinito corrente
   - poi i restanti provider di generazione musicale registrati in ordine di provider-id

Se un provider fallisce, viene provato automaticamente il candidato successivo. Se falliscono tutti,
l'errore include i dettagli di ogni tentativo.

## Note sui provider

- Google usa la generazione batch Lyria 3. L'attuale flusso bundled supporta
  prompt, testo facoltativo dei lyrics e immagini di riferimento facoltative.
- MiniMax usa l'endpoint batch `music_generation`. L'attuale flusso bundled
  supporta prompt, lyrics facoltativi, modalità strumentale, controllo della durata e
  output mp3.
- Il supporto ComfyUI è guidato dal workflow e dipende dal grafo configurato più
  la mappatura dei nodi per i campi prompt/output.

## Scegliere il percorso giusto

- Usa il percorso supportato da provider condivisi quando vuoi selezione del modello, failover del provider e il flusso integrato asincrono di attività/stato.
- Usa un percorso plugin come ComfyUI quando hai bisogno di un grafo workflow personalizzato o di un provider che non fa parte della capacità bundled condivisa di generazione musicale.
- Se stai eseguendo il debug di un comportamento specifico di ComfyUI, vedi [ComfyUI](/providers/comfy). Se stai eseguendo il debug del comportamento dei provider condivisi, inizia da [Google (Gemini)](/it/providers/google) o [MiniMax](/it/providers/minimax).

## Test live

Copertura live opt-in per i provider bundled condivisi:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

Copertura live opt-in per il percorso musicale ComfyUI bundled:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

Il file live Comfy copre anche i workflow di immagini e video comfy quando quelle
sezioni sono configurate.

## Correlati

- [Background Tasks](/it/automation/tasks) - tracciamento delle attività per esecuzioni `music_generate` scollegate
- [Configuration Reference](/it/gateway/configuration-reference#agent-defaults) - configurazione `musicGenerationModel`
- [ComfyUI](/providers/comfy)
- [Google (Gemini)](/it/providers/google)
- [MiniMax](/it/providers/minimax)
- [Models](/it/concepts/models) - configurazione del modello e failover
- [Tools Overview](/it/tools)
