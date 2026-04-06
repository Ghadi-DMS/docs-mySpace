---
read_when:
    - Vuoi capire quali strumenti fornisce OpenClaw
    - Devi configurare, consentire o negare strumenti
    - Stai decidendo tra strumenti integrati, Skills e plugin
summary: 'Panoramica di strumenti e plugin di OpenClaw: cosa può fare l''agent e come estenderlo'
title: Strumenti e plugin
x-i18n:
    generated_at: "2026-04-06T03:12:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: b2371239316997b0fe389bfa2ec38404e1d3e177755ad81ff8035ac583d9adeb
    source_path: tools/index.md
    workflow: 15
---

# Strumenti e plugin

Tutto ciò che l'agent fa oltre a generare testo avviene tramite gli **strumenti**.
Gli strumenti sono il modo in cui l'agent legge file, esegue comandi, naviga sul web, invia
messaggi e interagisce con i dispositivi.

## Strumenti, Skills e plugin

OpenClaw ha tre livelli che lavorano insieme:

<Steps>
  <Step title="Gli strumenti sono ciò che l'agent richiama">
    Uno strumento è una funzione tipizzata che l'agent può invocare (ad esempio `exec`, `browser`,
    `web_search`, `message`). OpenClaw include un insieme di **strumenti integrati** e
    i plugin possono registrarne altri.

    L'agent vede gli strumenti come definizioni strutturate di funzione inviate all'API del modello.

  </Step>

  <Step title="Le Skills insegnano all'agent quando e come">
    Una Skill è un file markdown (`SKILL.md`) iniettato nel prompt di sistema.
    Le Skills forniscono all'agent contesto, vincoli e indicazioni passo passo per
    usare gli strumenti in modo efficace. Le Skills si trovano nel tuo workspace, in cartelle condivise,
    oppure vengono distribuite all'interno dei plugin.

    [Riferimento Skills](/it/tools/skills) | [Creare Skills](/it/tools/creating-skills)

  </Step>

  <Step title="I plugin impacchettano tutto insieme">
    Un plugin è un pacchetto che può registrare qualsiasi combinazione di capability:
    canali, provider di modelli, strumenti, Skills, speech, realtime transcription,
    realtime voice, media understanding, image generation, video generation,
    web fetch, web search e altro. Alcuni plugin sono **core** (distribuiti con
    OpenClaw), altri sono **esterni** (pubblicati su npm dalla community).

    [Installare e configurare plugin](/it/tools/plugin) | [Crea il tuo](/it/plugins/building-plugins)

  </Step>
</Steps>

## Strumenti integrati

Questi strumenti sono inclusi in OpenClaw e sono disponibili senza installare alcun plugin:

| Strumento                                  | Cosa fa                                                              | Pagina                                      |
| ------------------------------------------ | -------------------------------------------------------------------- | ------------------------------------------- |
| `exec` / `process`                         | Esegue comandi shell, gestisce processi in background                | [Exec](/it/tools/exec)                         |
| `code_execution`                           | Esegue analisi Python remote in sandbox                              | [Code Execution](/it/tools/code-execution)     |
| `browser`                                  | Controlla un browser Chromium (naviga, clicca, screenshot)           | [Browser](/it/tools/browser)                   |
| `web_search` / `x_search` / `web_fetch`    | Cerca nel web, cerca post su X, recupera contenuto di pagine         | [Web](/it/tools/web)                           |
| `read` / `write` / `edit`                  | I/O di file nel workspace                                            |                                             |
| `apply_patch`                              | Patch di file multi-hunk                                             | [Apply Patch](/it/tools/apply-patch)           |
| `message`                                  | Invia messaggi su tutti i canali                                     | [Agent Send](/it/tools/agent-send)             |
| `canvas`                                   | Guida il Canvas a nodi (present, eval, snapshot)                     |                                             |
| `nodes`                                    | Rileva e seleziona dispositivi associati                             |                                             |
| `cron` / `gateway`                         | Gestisce job pianificati; ispeziona, modifica, riavvia o aggiorna il gateway |                                             |
| `image` / `image_generate`                 | Analizza o genera immagini                                           | [Generazione di immagini](/it/tools/image-generation) |
| `music_generate`                           | Genera tracce musicali                                               | [Generazione di musica](/tools/music-generation) |
| `video_generate`                           | Genera video                                                         | [Generazione di video](/tools/video-generation) |
| `tts`                                      | Conversione text-to-speech singola                                   | [TTS](/it/tools/tts)                           |
| `sessions_*` / `subagents` / `agents_list` | Gestione sessioni, stato e orchestrazione di sub-agent               | [Sub-agent](/it/tools/subagents)               |
| `session_status`                           | Lettura leggera in stile `/status` e override del modello di sessione | [Strumenti di sessione](/it/concepts/session-tool)     |

Per il lavoro sulle immagini, usa `image` per l'analisi e `image_generate` per la generazione o la modifica. Se scegli come target `openai/*`, `google/*`, `fal/*` o un altro provider image non predefinito, configura prima l'autenticazione/la chiave API di quel provider.

Per il lavoro sulla musica, usa `music_generate`. Se scegli come target `google/*`, `minimax/*` o un altro provider music non predefinito, configura prima l'autenticazione/la chiave API di quel provider.

Per il lavoro sui video, usa `video_generate`. Se scegli come target `qwen/*` o un altro provider video non predefinito, configura prima l'autenticazione/la chiave API di quel provider.

Per la generazione audio guidata da workflow, usa `music_generate` quando un plugin come
ComfyUI lo registra. Questo è separato da `tts`, che è text-to-speech.

`session_status` è lo strumento leggero di stato/lettura nel gruppo sessions.
Risponde a domande in stile `/status` sulla sessione corrente e può
facoltativamente impostare un override del modello per sessione; `model=default` cancella tale
override. Come `/status`, può completare in retrospettiva contatori sparsi di token/cache e l'etichetta
del modello runtime attivo dall'ultima voce di utilizzo nella trascrizione.

`gateway` è lo strumento runtime riservato al proprietario per le operazioni del gateway:

- `config.schema.lookup` per un sottoalbero della configurazione limitato a un percorso prima delle modifiche
- `config.get` per lo snapshot + hash della configurazione corrente
- `config.patch` per aggiornamenti parziali della configurazione con riavvio
- `config.apply` solo per la sostituzione completa della configurazione
- `update.run` per auto-aggiornamento esplicito + riavvio

Per le modifiche parziali, preferisci `config.schema.lookup` e poi `config.patch`. Usa
`config.apply` solo quando intendi sostituire l'intera configurazione.
Lo strumento rifiuta anche di modificare `tools.exec.ask` o `tools.exec.security`;
gli alias legacy `tools.bash.*` vengono normalizzati agli stessi percorsi exec protetti.

### Strumenti forniti dai plugin

I plugin possono registrare strumenti aggiuntivi. Alcuni esempi:

- [Lobster](/it/tools/lobster) — runtime di workflow tipizzato con approvazioni riprendibili
- [LLM Task](/it/tools/llm-task) — passaggio LLM solo JSON per output strutturato
- [Generazione di musica](/tools/music-generation) — strumento condiviso `music_generate` con provider supportati da workflow
- [Diffs](/it/tools/diffs) — visualizzatore e renderer di diff
- [OpenProse](/it/prose) — orchestrazione di workflow markdown-first

## Configurazione degli strumenti

### Liste di autorizzazione e negazione

Controlla quali strumenti l'agent può richiamare tramite `tools.allow` / `tools.deny` nella
configurazione. La negazione ha sempre la precedenza sull'autorizzazione.

```json5
{
  tools: {
    allow: ["group:fs", "browser", "web_search"],
    deny: ["exec"],
  },
}
```

### Profili degli strumenti

`tools.profile` imposta una allowlist di base prima che vengano applicati `allow`/`deny`.
Override per-agent: `agents.list[].tools.profile`.

| Profilo     | Cosa include                                                                                                                                   |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `full`      | Nessuna restrizione (uguale a non impostato)                                                                                                   |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `image`, `image_generate`, `music_generate`, `video_generate` |
| `messaging` | `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`                                                     |
| `minimal`   | Solo `session_status`                                                                                                                          |

### Gruppi di strumenti

Usa le abbreviazioni `group:*` nelle liste allow/deny:

| Gruppo             | Strumenti                                                                                                   |
| ------------------ | ----------------------------------------------------------------------------------------------------------- |
| `group:runtime`    | exec, process, code_execution (`bash` è accettato come alias di `exec`)                                     |
| `group:fs`         | read, write, edit, apply_patch                                                                              |
| `group:sessions`   | sessions_list, sessions_history, sessions_send, sessions_spawn, sessions_yield, subagents, session_status  |
| `group:memory`     | memory_search, memory_get                                                                                   |
| `group:web`        | web_search, x_search, web_fetch                                                                             |
| `group:ui`         | browser, canvas                                                                                             |
| `group:automation` | cron, gateway                                                                                               |
| `group:messaging`  | message                                                                                                     |
| `group:nodes`      | nodes                                                                                                       |
| `group:agents`     | agents_list                                                                                                 |
| `group:media`      | image, image_generate, music_generate, video_generate, tts                                                  |
| `group:openclaw`   | Tutti gli strumenti integrati di OpenClaw (esclude gli strumenti dei plugin)                                |

`sessions_history` restituisce una vista di richiamo limitata e filtrata per sicurezza. Rimuove
tag di thinking, scaffolding `<relevant-memories>`, payload XML in testo semplice delle chiamate agli strumenti
(inclusi `<tool_call>...</tool_call>`,
`<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`,
`<function_calls>...</function_calls>` e blocchi di chiamata agli strumenti troncati),
scaffolding degradato delle chiamate agli strumenti, token di controllo del modello ASCII/full-width trapelati
e XML malformato delle chiamate agli strumenti MiniMax dal testo dell'assistente, quindi applica
redazione/troncamento e possibili placeholder per righe troppo grandi invece di comportarsi
come dump grezzo della trascrizione.

### Restrizioni specifiche del provider

Usa `tools.byProvider` per limitare gli strumenti per provider specifici senza
modificare i valori predefiniti globali:

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
    },
  },
}
```
