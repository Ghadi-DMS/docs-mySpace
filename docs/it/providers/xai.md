---
read_when:
    - Vuoi usare i modelli Grok in OpenClaw
    - Stai configurando l'autenticazione xAI o gli ID dei modelli
summary: Usare i modelli Grok di xAI in OpenClaw
title: xAI
x-i18n:
    generated_at: "2026-04-06T03:11:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 64bc899655427cc10bdc759171c7d1ec25ad9f1e4f9d803f1553d3d586c6d71d
    source_path: providers/xai.md
    workflow: 15
---

# xAI

OpenClaw include un plugin provider `xai` bundled per i modelli Grok.

## Configurazione

1. Crea una chiave API nella console xAI.
2. Imposta `XAI_API_KEY`, oppure esegui:

```bash
openclaw onboard --auth-choice xai-api-key
```

3. Scegli un modello come:

```json5
{
  agents: { defaults: { model: { primary: "xai/grok-4" } } },
}
```

OpenClaw ora usa l'API xAI Responses come trasporto xAI bundled. La stessa
`XAI_API_KEY` può anche alimentare `web_search` basato su Grok, `x_search` di prima classe
e `code_execution` remoto.
Se archivi una chiave xAI in `plugins.entries.xai.config.webSearch.apiKey`,
il provider di modelli xAI bundled ora riutilizza anche quella chiave come fallback.
La configurazione di `code_execution` si trova in `plugins.entries.xai.config.codeExecution`.

## Catalogo attuale dei modelli bundled

OpenClaw ora include queste famiglie di modelli xAI pronte all'uso:

- `grok-3`, `grok-3-fast`, `grok-3-mini`, `grok-3-mini-fast`
- `grok-4`, `grok-4-0709`
- `grok-4-fast`, `grok-4-fast-non-reasoning`
- `grok-4-1-fast`, `grok-4-1-fast-non-reasoning`
- `grok-4.20-beta-latest-reasoning`, `grok-4.20-beta-latest-non-reasoning`
- `grok-code-fast-1`

Il plugin risolve in forward anche gli ID più recenti `grok-4*` e `grok-code-fast*` quando
seguono la stessa forma API.

Note sui modelli fast:

- `grok-4-fast`, `grok-4-1-fast` e le varianti `grok-4.20-beta-*` sono gli
  attuali riferimenti Grok con supporto image nel catalogo bundled.
- `/fast on` o `agents.defaults.models["xai/<model>"].params.fastMode: true`
  riscrivono le richieste xAI native come segue:
  - `grok-3` -> `grok-3-fast`
  - `grok-3-mini` -> `grok-3-mini-fast`
  - `grok-4` -> `grok-4-fast`
  - `grok-4-0709` -> `grok-4-fast`

Gli alias legacy di compatibilità continuano a essere normalizzati agli ID bundled canonici. Per
esempio:

- `grok-4-fast-reasoning` -> `grok-4-fast`
- `grok-4-1-fast-reasoning` -> `grok-4-1-fast`
- `grok-4.20-reasoning` -> `grok-4.20-beta-latest-reasoning`
- `grok-4.20-non-reasoning` -> `grok-4.20-beta-latest-non-reasoning`

## Ricerca web

Anche il provider bundled di ricerca web `grok` usa `XAI_API_KEY`:

```bash
openclaw config set tools.web.search.provider grok
```

## Generazione video

Il plugin bundled `xai` registra anche la generazione video tramite lo
strumento condiviso `video_generate`.

- Modello video predefinito: `xai/grok-imagine-video`
- Modalità: flussi text-to-video, image-to-video e flussi remoti di modifica/estensione video
- Supporta `aspectRatio` e `resolution`
- Limite attuale: i buffer video locali non sono accettati; usa URL remoti `http(s)`
  per gli input di riferimento/modifica video

Per usare xAI come provider video predefinito:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "xai/grok-imagine-video",
      },
    },
  },
}
```

Consulta [Generazione video](/tools/video-generation) per i parametri dello
strumento condiviso, la selezione del provider e il comportamento di failover.

## Limiti noti

- L'autenticazione oggi è solo con chiave API. In OpenClaw non esiste ancora un flusso xAI OAuth/device-code.
- `grok-4.20-multi-agent-experimental-beta-0304` non è supportato nel normale percorso del provider xAI perché richiede una superficie API upstream diversa dal trasporto xAI standard di OpenClaw.

## Note

- OpenClaw applica automaticamente correzioni di compatibilità specifiche di xAI per schema degli strumenti e chiamate agli strumenti nel percorso runner condiviso.
- Le richieste xAI native usano per impostazione predefinita `tool_stream: true`. Imposta
  `agents.defaults.models["xai/<model>"].params.tool_stream` su `false` per
  disabilitarlo.
- Il wrapper xAI bundled rimuove i flag di schema degli strumenti strict non supportati e
  le chiavi del payload di reasoning prima di inviare richieste xAI native.
- `web_search`, `x_search` e `code_execution` sono esposti come strumenti OpenClaw. OpenClaw abilita lo specifico built-in xAI di cui ha bisogno all'interno di ogni richiesta dello strumento invece di allegare tutti gli strumenti nativi a ogni turno di chat.
- `x_search` e `code_execution` sono posseduti dal plugin xAI bundled invece di essere codificati in modo statico nel runtime del modello core.
- `code_execution` è esecuzione remota nel sandbox xAI, non [`exec`](/it/tools/exec) locale.
- Per una panoramica più ampia dei provider, consulta [Provider di modelli](/it/providers/index).
