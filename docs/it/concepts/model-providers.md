---
read_when:
    - Ti serve un riferimento di configurazione dei modelli provider per provider
    - Vuoi configurazioni di esempio o comandi CLI di onboarding per i provider di modelli
summary: Panoramica dei provider di modelli con configurazioni di esempio + flussi CLI
title: Provider di modelli
x-i18n:
    generated_at: "2026-04-06T03:08:40Z"
    model: gpt-5.4
    provider: openai
    source_hash: 15e4b82e07221018a723279d309e245bb4023bc06e64b3c910ef2cae3dfa2599
    source_path: concepts/model-providers.md
    workflow: 15
---

# Provider di modelli

Questa pagina copre i **provider LLM/modelli** (non i canali chat come WhatsApp/Telegram).
Per le regole di selezione dei modelli, vedi [/concepts/models](/it/concepts/models).

## Regole rapide

- I riferimenti ai modelli usano `provider/model` (esempio: `opencode/claude-opus-4-6`).
- Se imposti `agents.defaults.models`, diventa la allowlist.
- Helper CLI: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- Le regole di runtime per il fallback, le probe di cooldown e la persistenza
  degli override di sessione sono documentate in
  [/concepts/model-failover](/it/concepts/model-failover).
- `models.providers.*.models[].contextWindow` è il metadato nativo del modello;
  `models.providers.*.models[].contextTokens` è il limite effettivo del runtime.
- I plugin provider possono iniettare cataloghi di modelli tramite `registerProvider({ catalog })`;
  OpenClaw unisce quell'output in `models.providers` prima di scrivere
  `models.json`.
- I manifest dei provider possono dichiarare `providerAuthEnvVars` in modo che
  le probe di autenticazione generiche basate su variabili d'ambiente non debbano caricare il runtime del plugin. La mappa rimanente delle variabili d'ambiente nel core
  ora serve solo per provider core/non-plugin e per alcuni casi di precedenza generica
  come l'onboarding Anthropic con priorità alla chiave API.
- I plugin provider possono anche gestire il comportamento runtime del provider tramite
  `normalizeModelId`, `normalizeTransport`, `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`,
  `normalizeResolvedModel`, `contributeResolvedModelCompat`,
  `capabilities`, `normalizeToolSchemas`,
  `inspectToolSchemas`, `resolveReasoningOutputMode`,
  `prepareExtraParams`, `createStreamFn`, `wrapStreamFn`,
  `resolveTransportTurnState`, `resolveWebSocketSessionPolicy`,
  `createEmbeddingProvider`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`,
  `matchesContextOverflowError`, `classifyFailoverReason`,
  `isCacheTtlEligible`, `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot`, e
  `onModelSelected`.
- Nota: `capabilities` del runtime del provider è un metadato condiviso del runner (famiglia del provider,
  particolarità di trascrizione/tooling, suggerimenti su transport/cache). Non è la
  stessa cosa del [modello di capability pubblico](/it/plugins/architecture#public-capability-model)
  che descrive ciò che un plugin registra (inferenza testuale, voce, ecc.).

## Comportamento del provider gestito dal plugin

I plugin provider possono ora gestire la maggior parte della logica specifica del provider, mentre OpenClaw mantiene
il ciclo di inferenza generico.

Suddivisione tipica:

- `auth[].run` / `auth[].runNonInteractive`: il provider gestisce i flussi di onboarding/login
  per `openclaw onboard`, `openclaw models auth` e la configurazione headless
- `wizard.setup` / `wizard.modelPicker`: il provider gestisce etichette di scelta dell'autenticazione,
  alias legacy, suggerimenti per l'allowlist di onboarding e voci di configurazione nei selettori di onboarding/modello
- `catalog`: il provider compare in `models.providers`
- `normalizeModelId`: il provider normalizza gli id di modelli legacy/preview prima della
  ricerca o canonizzazione
- `normalizeTransport`: il provider normalizza `api` / `baseUrl` della famiglia di transport
  prima dell'assemblaggio generico del modello; OpenClaw controlla prima il provider corrispondente,
  poi altri plugin provider compatibili con hook finché uno non modifica davvero il
  transport
- `normalizeConfig`: il provider normalizza la configurazione `models.providers.<id>` prima che
  il runtime la usi; OpenClaw controlla prima il provider corrispondente, poi altri
  plugin provider compatibili con hook finché uno non modifica davvero la configurazione. Se nessun
  hook del provider riscrive la configurazione, gli helper Google-family inclusi
  normalizzano comunque le voci supportate dei provider Google.
- `applyNativeStreamingUsageCompat`: il provider applica riscritture di compatibilità sull'uso nativo in streaming guidate dall'endpoint per i provider di configurazione
- `resolveConfigApiKey`: il provider risolve l'autenticazione con marker d'ambiente per i provider di configurazione
  senza forzare il caricamento dell'autenticazione completa del runtime. `amazon-bedrock` ha anche un
  resolver integrato per marker AWS in questa fase, anche se l'autenticazione runtime di Bedrock usa
  la catena predefinita dell'SDK AWS.
- `resolveSyntheticAuth`: il provider può esporre la disponibilità di autenticazione
  locale/self-hosted o di altro tipo basata su configurazione senza salvare segreti in chiaro
- `shouldDeferSyntheticProfileAuth`: il provider può contrassegnare i placeholder di profilo sintetico memorizzati
  con precedenza inferiore rispetto all'autenticazione basata su env/config
- `resolveDynamicModel`: il provider accetta id modello non ancora presenti nel
  catalogo statico locale
- `prepareDynamicModel`: il provider richiede un aggiornamento dei metadati prima di ritentare
  la risoluzione dinamica
- `normalizeResolvedModel`: il provider richiede riscritture di transport o base URL
- `contributeResolvedModelCompat`: il provider contribuisce flag di compatibilità per i propri
  modelli vendor anche quando arrivano tramite un altro transport compatibile
- `capabilities`: il provider pubblica particolarità di trascrizione/tooling/famiglia provider
- `normalizeToolSchemas`: il provider ripulisce gli schemi degli strumenti prima che il runner
  embedded li veda
- `inspectToolSchemas`: il provider espone avvisi specifici del transport sugli schemi
  dopo la normalizzazione
- `resolveReasoningOutputMode`: il provider sceglie contratti di output di reasoning
  nativi o taggati
- `prepareExtraParams`: il provider imposta valori predefiniti o normalizza parametri di richiesta per modello
- `createStreamFn`: il provider sostituisce il normale percorso di stream con un
  transport completamente personalizzato
- `wrapStreamFn`: il provider applica wrapper di compatibilità a header/body/modello della richiesta
- `resolveTransportTurnState`: il provider fornisce header o metadati nativi del transport
  per turno
- `resolveWebSocketSessionPolicy`: il provider fornisce header nativi della sessione WebSocket
  o una policy di cooldown della sessione
- `createEmbeddingProvider`: il provider gestisce il comportamento di embedding della memoria quando
  appartiene al plugin provider invece che allo switchboard core degli embedding
- `formatApiKey`: il provider formatta i profili di autenticazione memorizzati nella
  stringa `apiKey` di runtime attesa dal transport
- `refreshOAuth`: il provider gestisce il refresh OAuth quando i refresher condivisi di `pi-ai`
  non sono sufficienti
- `buildAuthDoctorHint`: il provider aggiunge indicazioni di riparazione quando il refresh OAuth
  fallisce
- `matchesContextOverflowError`: il provider riconosce errori di overflow della finestra di contesto
  specifici del provider che le euristiche generiche non rileverebbero
- `classifyFailoverReason`: il provider mappa errori grezzi specifici del provider dal transport/API
  a motivi di failover come rate limit o overload
- `isCacheTtlEligible`: il provider decide quali id modello upstream supportano il TTL della prompt cache
- `buildMissingAuthMessage`: il provider sostituisce l'errore generico dell'auth store
  con un suggerimento di recupero specifico del provider
- `suppressBuiltInModel`: il provider nasconde righe upstream obsolete e può restituire un
  errore gestito dal vendor per i fallimenti di risoluzione diretta
- `augmentModelCatalog`: il provider aggiunge righe sintetiche/finali del catalogo dopo
  discovery e merge della configurazione
- `isBinaryThinking`: il provider gestisce l'esperienza utente del thinking binario on/off
- `supportsXHighThinking`: il provider abilita `xhigh` per modelli selezionati
- `resolveDefaultThinkingLevel`: il provider gestisce la policy predefinita di `/think` per una
  famiglia di modelli
- `applyConfigDefaults`: il provider applica valori predefiniti globali specifici del provider
  durante la materializzazione della configurazione in base a modalità di autenticazione, env o famiglia di modelli
- `isModernModelRef`: il provider gestisce il matching del modello preferito live/smoke
- `prepareRuntimeAuth`: il provider trasforma una credenziale configurata in un token runtime
  di breve durata
- `resolveUsageAuth`: il provider risolve credenziali di utilizzo/quota per `/usage`
  e superfici correlate di stato/reporting
- `fetchUsageSnapshot`: il provider gestisce fetch/parsing dell'endpoint di utilizzo mentre
  il core mantiene comunque la shell del riepilogo e la formattazione
- `onModelSelected`: il provider esegue effetti collaterali post-selezione come
  telemetria o bookkeeping di sessione gestito dal provider

Esempi bundled attuali:

- `anthropic`: fallback forward-compat per Claude 4.6, suggerimenti di riparazione auth, fetch
  dell'endpoint di utilizzo, metadati cache-TTL/famiglia provider e valori predefiniti globali
  della configurazione consapevoli dell'autenticazione
- `amazon-bedrock`: matching gestito dal provider per l'overflow del contesto e classificazione del
  motivo del failover per errori Bedrock specifici di throttle/not-ready, oltre
  alla famiglia di replay condivisa `anthropic-by-model` per le policy di replay solo-Claude
  sul traffico Anthropic
- `anthropic-vertex`: guardie della policy di replay solo-Claude sul traffico
  `anthropic-message`
- `openrouter`: id modello pass-through, wrapper di richiesta, hint sulle capability del provider,
  sanificazione della thought-signature Gemini sul traffico Gemini proxato, iniezione del reasoning del proxy tramite la famiglia di stream `openrouter-thinking`,
  inoltro dei metadati di routing e policy cache-TTL
- `github-copilot`: onboarding/login da dispositivo, fallback del modello forward-compat,
  hint di trascrizione per il thinking di Claude, scambio di token runtime e fetch
  dell'endpoint di utilizzo
- `openai`: fallback forward-compat per GPT-5.4, normalizzazione diretta del transport OpenAI,
  hint di auth mancante consapevoli di Codex, soppressione di Spark, righe sintetiche del catalogo OpenAI/Codex,
  policy di thinking/modello live, normalizzazione degli alias dei token di utilizzo
  (`input` / `output` e famiglie `prompt` / `completion`), la famiglia di stream condivisa
  `openai-responses-defaults` per wrapper nativi OpenAI/Codex,
  metadati della famiglia provider, registrazione bundled del provider di generazione immagini
  per `gpt-image-1` e registrazione bundled del provider di generazione video
  per `sora-2`
- `google`: fallback forward-compat per Gemini 3.1, validazione nativa del replay Gemini,
  sanificazione del replay bootstrap, modalità di output di reasoning taggata,
  matching di modelli moderni, registrazione bundled del provider di generazione immagini per
  i modelli Gemini image-preview e registrazione bundled del provider di generazione video
  per i modelli Veo
- `moonshot`: transport condiviso, normalizzazione del payload di thinking gestita dal plugin
- `kilocode`: transport condiviso, header di richiesta gestiti dal plugin, normalizzazione del payload di reasoning,
  sanificazione della thought-signature Gemini su proxy-Gemini e policy cache-TTL
- `zai`: fallback forward-compat per GLM-5, valori predefiniti `tool_stream`, policy cache-TTL,
  policy di binary-thinking/modello live e auth di utilizzo + fetch della quota;
  gli id sconosciuti `glm-5*` vengono sintetizzati dal template bundled `glm-4.7`
- `xai`: normalizzazione nativa del transport Responses, riscritture degli alias `/fast` per
  le varianti rapide di Grok, `tool_stream` predefinito, pulizia di schemi strumenti /
  payload di reasoning specifica xAI e registrazione bundled del provider di generazione video
  per `grok-imagine-video`
- `mistral`: metadati di capability gestiti dal plugin
- `opencode` e `opencode-go`: metadati di capability gestiti dal plugin più
  sanificazione della thought-signature Gemini su proxy-Gemini
- `alibaba`: catalogo di generazione video gestito dal plugin per riferimenti diretti a modelli Wan
  come `alibaba/wan2.6-t2v`
- `byteplus`: cataloghi gestiti dal plugin più registrazione bundled del provider di generazione video
  per i modelli Seedance text-to-video/image-to-video
- `fal`: registrazione bundled del provider di generazione video per provider hosted di terze parti
  e registrazione bundled del provider di generazione immagini per modelli immagine FLUX, più registrazione bundled
  del provider di generazione video per modelli video hosted di terze parti
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway` e `volcengine`:
  solo cataloghi gestiti dal plugin
- `qwen`: cataloghi gestiti dal plugin per modelli testuali più registrazioni condivise dei provider
  di comprensione dei media e generazione video per le sue superfici multimodali;
  la generazione video Qwen usa gli endpoint video Standard DashScope con modelli Wan bundled
  come `wan2.6-t2v` e `wan2.7-r2v`
- `runway`: registrazione del provider di generazione video gestita dal plugin per modelli nativi
  Runway basati su task come `gen4.5`
- `minimax`: cataloghi gestiti dal plugin, registrazione bundled del provider di generazione video
  per modelli video Hailuo, registrazione bundled del provider di generazione immagini
  per `image-01`, selezione ibrida della policy di replay Anthropic/OpenAI e logica auth/snapshot dell'utilizzo
- `together`: cataloghi gestiti dal plugin più registrazione bundled del provider di generazione video
  per modelli video Wan
- `xiaomi`: cataloghi gestiti dal plugin più logica auth/snapshot dell'utilizzo

Il plugin bundled `openai` ora gestisce entrambi gli id provider: `openai` e
`openai-codex`.

Questo copre i provider che rientrano ancora nei normali transport di OpenClaw. Un provider
che richiede un esecutore di richieste completamente personalizzato è una superficie di estensione
separata e più profonda.

## Rotazione delle chiavi API

- Supporta la rotazione generica del provider per provider selezionati.
- Configura più chiavi tramite:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY` (override live singolo, priorità massima)
  - `<PROVIDER>_API_KEYS` (lista separata da virgole o punti e virgola)
  - `<PROVIDER>_API_KEY` (chiave primaria)
  - `<PROVIDER>_API_KEY_*` (lista numerata, ad esempio `<PROVIDER>_API_KEY_1`)
- Per i provider Google, `GOOGLE_API_KEY` è incluso anche come fallback.
- L'ordine di selezione delle chiavi preserva la priorità e deduplica i valori.
- Le richieste vengono ritentate con la chiave successiva solo in caso di risposte di rate limit (ad
  esempio `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded` o messaggi periodici di limite di utilizzo).
- I fallimenti non dovuti a rate limit falliscono immediatamente; non viene tentata alcuna rotazione delle chiavi.
- Quando tutte le chiavi candidate falliscono, viene restituito l'errore finale dell'ultimo tentativo.

## Provider integrati (catalogo pi-ai)

OpenClaw include il catalogo pi‑ai. Questi provider non richiedono
alcuna configurazione `models.providers`; basta impostare l'autenticazione + scegliere un modello.

### OpenAI

- Provider: `openai`
- Auth: `OPENAI_API_KEY`
- Rotazione facoltativa: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, più `OPENCLAW_LIVE_OPENAI_KEY` (override singolo)
- Modelli di esempio: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- Il transport predefinito è `auto` (prima WebSocket, fallback SSE)
- Override per modello tramite `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"` o `"auto"`)
- Il warm-up WebSocket di OpenAI Responses è abilitato per impostazione predefinita tramite `params.openaiWsWarmup` (`true`/`false`)
- L'elaborazione prioritaria OpenAI può essere abilitata tramite `agents.defaults.models["openai/<model>"].params.serviceTier`
- `/fast` e `params.fastMode` mappano le richieste Responses dirette `openai/*` a `service_tier=priority` su `api.openai.com`
- Usa `params.serviceTier` quando vuoi un livello esplicito invece del toggle condiviso `/fast`
- Gli header di attribuzione OpenClaw nascosti (`originator`, `version`,
  `User-Agent`) si applicano solo al traffico OpenAI nativo verso `api.openai.com`, non
  ai proxy generici compatibili con OpenAI
- I percorsi OpenAI nativi mantengono anche `store` di Responses, hint di prompt-cache e
  modellazione del payload di compatibilità reasoning OpenAI; i percorsi proxy no
- `openai/gpt-5.3-codex-spark` è intenzionalmente soppresso in OpenClaw perché l'API OpenAI live lo rifiuta; Spark è trattato come solo Codex

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Provider: `anthropic`
- Auth: `ANTHROPIC_API_KEY`
- Rotazione facoltativa: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, più `OPENCLAW_LIVE_ANTHROPIC_KEY` (override singolo)
- Modello di esempio: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- Le richieste Anthropic pubbliche dirette supportano anche il toggle condiviso `/fast` e `params.fastMode`, inclusi traffico autenticato con chiave API e OAuth inviato a `api.anthropic.com`; OpenClaw lo mappa a `service_tier` di Anthropic (`auto` vs `standard_only`)
- Nota di fatturazione: per Anthropic in OpenClaw, la distinzione pratica è **chiave API** oppure **abbonamento Claude con Extra Usage**. Anthropic ha informato gli utenti OpenClaw il **4 aprile 2026 alle 12:00 PM PT / 8:00 PM BST** che il percorso di login Claude di **OpenClaw** conta come utilizzo di harness di terze parti e richiede **Extra Usage** fatturato separatamente dall'abbonamento. Le nostre riproduzioni locali mostrano anche che la stringa del prompt identificativa di OpenClaw non si riproduce nel percorso Anthropic SDK + chiave API.
- Il setup-token Anthropic è di nuovo disponibile come percorso OpenClaw legacy/manuale. Usalo sapendo che Anthropic ha detto agli utenti OpenClaw che questo percorso richiede **Extra Usage**.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- Provider: `openai-codex`
- Auth: OAuth (ChatGPT)
- Modello di esempio: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex` o `openclaw models auth login --provider openai-codex`
- Il transport predefinito è `auto` (prima WebSocket, fallback SSE)
- Override per modello tramite `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"` o `"auto"`)
- `params.serviceTier` viene inoltrato anche nelle richieste native Codex Responses (`chatgpt.com/backend-api`)
- Gli header di attribuzione OpenClaw nascosti (`originator`, `version`,
  `User-Agent`) vengono allegati solo al traffico Codex nativo verso
  `chatgpt.com/backend-api`, non ai proxy generici compatibili con OpenAI
- Condivide lo stesso toggle `/fast` e la stessa configurazione `params.fastMode` di `openai/*` diretto; OpenClaw lo mappa a `service_tier=priority`
- `openai-codex/gpt-5.3-codex-spark` resta disponibile quando il catalogo OAuth Codex lo espone; dipende dai diritti
- `openai-codex/gpt-5.4` mantiene `contextWindow = 1050000` nativo e un `contextTokens = 272000` predefinito a runtime; override del limite runtime con `models.providers.openai-codex.models[].contextTokens`
- Nota sulla policy: OpenAI Codex OAuth è esplicitamente supportato per strumenti/workflow esterni come OpenClaw.

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [{ id: "gpt-5.4", contextTokens: 160000 }],
      },
    },
  },
}
```

### Altre opzioni hosted in stile abbonamento

- [Qwen Cloud](/it/providers/qwen): superficie provider Qwen Cloud più mapping degli endpoint Alibaba DashScope e Coding Plan
- [MiniMax](/it/providers/minimax): accesso MiniMax Coding Plan con OAuth o chiave API
- [GLM Models](/it/providers/glm): endpoint Z.AI Coding Plan o API generiche

### OpenCode

- Auth: `OPENCODE_API_KEY` (o `OPENCODE_ZEN_API_KEY`)
- Provider runtime Zen: `opencode`
- Provider runtime Go: `opencode-go`
- Modelli di esempio: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen` o `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (chiave API)

- Provider: `google`
- Auth: `GEMINI_API_KEY`
- Rotazione facoltativa: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, fallback `GOOGLE_API_KEY` e `OPENCLAW_LIVE_GEMINI_KEY` (override singolo)
- Modelli di esempio: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- Compatibilità: la configurazione legacy di OpenClaw che usa `google/gemini-3.1-flash-preview` viene normalizzata in `google/gemini-3-flash-preview`
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- Le esecuzioni Gemini dirette accettano anche `agents.defaults.models["google/<model>"].params.cachedContent`
  (o il legacy `cached_content`) per inoltrare un handle nativo del provider
  `cachedContents/...`; i cache hit Gemini emergono come `cacheRead` di OpenClaw

### Google Vertex

- Provider: `google-vertex`
- Auth: gcloud ADC
  - Le risposte JSON della CLI Gemini vengono parse da `response`; l'utilizzo ricade su
    `stats`, con `stats.cached` normalizzato in `cacheRead` di OpenClaw.

### Z.AI (GLM)

- Provider: `zai`
- Auth: `ZAI_API_KEY`
- Modello di esempio: `zai/glm-5`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - Alias: `z.ai/*` e `z-ai/*` vengono normalizzati in `zai/*`
  - `zai-api-key` rileva automaticamente l'endpoint Z.AI corrispondente; `zai-coding-global`, `zai-coding-cn`, `zai-global` e `zai-cn` forzano una superficie specifica

### Vercel AI Gateway

- Provider: `vercel-ai-gateway`
- Auth: `AI_GATEWAY_API_KEY`
- Modello di esempio: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Provider: `kilocode`
- Auth: `KILOCODE_API_KEY`
- Modello di esempio: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- URL base: `https://api.kilo.ai/api/gateway/`
- Il catalogo statico di fallback include `kilocode/kilo/auto`; la discovery live
  di `https://api.kilo.ai/api/gateway/models` può espandere ulteriormente il catalogo
  runtime.
- L'instradamento upstream esatto dietro `kilocode/kilo/auto` è gestito da Kilo Gateway,
  non codificato rigidamente in OpenClaw.

Vedi [/providers/kilocode](/it/providers/kilocode) per i dettagli di configurazione.

### Altri plugin provider bundled

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- Modello di esempio: `openrouter/auto`
- OpenClaw applica gli header di attribuzione dell'app documentati da OpenRouter solo quando
  la richiesta punta effettivamente a `openrouter.ai`
- I marker `cache_control` specifici di Anthropic per OpenRouter sono analogamente limitati
  ai percorsi OpenRouter verificati, non a URL proxy arbitrari
- OpenRouter resta sul percorso compatibile con OpenAI in stile proxy, quindi la modellazione nativa delle richieste solo-OpenAI (`serviceTier`, `store` di Responses,
  hint di prompt-cache, payload di compatibilità reasoning OpenAI) non viene inoltrata
- I riferimenti OpenRouter basati su Gemini mantengono solo la sanificazione della thought-signature per proxy-Gemini;
  la validazione nativa del replay Gemini e le riscritture bootstrap restano disattivate
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- Modello di esempio: `kilocode/kilo/auto`
- I riferimenti Kilo basati su Gemini mantengono lo stesso percorso di sanificazione della thought-signature
  per proxy-Gemini; `kilocode/kilo/auto` e altri hint in cui il proxy non supporta il reasoning
  saltano l'iniezione del reasoning del proxy
- MiniMax: `minimax` (chiave API) e `minimax-portal` (OAuth)
- Auth: `MINIMAX_API_KEY` per `minimax`; `MINIMAX_OAUTH_TOKEN` o `MINIMAX_API_KEY` per `minimax-portal`
- Modello di esempio: `minimax/MiniMax-M2.7` o `minimax-portal/MiniMax-M2.7`
- L'onboarding/configurazione con chiave API di MiniMax scrive definizioni esplicite del modello M2.7 con
  `input: ["text", "image"]`; il catalogo bundled del provider mantiene i riferimenti chat
  solo testo finché quella configurazione provider non viene materializzata
- Moonshot: `moonshot` (`MOONSHOT_API_KEY`)
- Modello di esempio: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi` (`KIMI_API_KEY` o `KIMICODE_API_KEY`)
- Modello di esempio: `kimi/kimi-code`
- Qianfan: `qianfan` (`QIANFAN_API_KEY`)
- Modello di esempio: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen` (`QWEN_API_KEY`, `MODELSTUDIO_API_KEY` o `DASHSCOPE_API_KEY`)
- Modello di esempio: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia` (`NVIDIA_API_KEY`)
- Modello di esempio: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- Modelli di esempio: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: `together` (`TOGETHER_API_KEY`)
- Modello di esempio: `together/moonshotai/Kimi-K2.5`
- Venice: `venice` (`VENICE_API_KEY`)
- Xiaomi: `xiaomi` (`XIAOMI_API_KEY`)
- Modello di esempio: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: `huggingface` (`HUGGINGFACE_HUB_TOKEN` o `HF_TOKEN`)
- Cloudflare AI Gateway: `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- Modello di esempio: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus` (`BYTEPLUS_API_KEY`)
- Modello di esempio: `byteplus-plan/ark-code-latest`
- xAI: `xai` (`XAI_API_KEY`)
  - Le richieste xAI native bundled usano il percorso xAI Responses
  - `/fast` o `params.fastMode: true` riscrivono `grok-3`, `grok-3-mini`,
    `grok-4` e `grok-4-0709` nelle rispettive varianti `*-fast`
  - `tool_stream` è attivo per impostazione predefinita; imposta
    `agents.defaults.models["xai/<model>"].params.tool_stream` su `false` per
    disattivarlo
- Mistral: `mistral` (`MISTRAL_API_KEY`)
- Modello di esempio: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq` (`GROQ_API_KEY`)
- Cerebras: `cerebras` (`CEREBRAS_API_KEY`)
  - I modelli GLM su Cerebras usano gli id `zai-glm-4.7` e `zai-glm-4.6`.
  - URL base compatibile con OpenAI: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Modello di esempio di Hugging Face Inference: `huggingface/deepseek-ai/DeepSeek-R1`; CLI: `openclaw onboard --auth-choice huggingface-api-key`. Vedi [Hugging Face (Inference)](/it/providers/huggingface).

## Provider tramite `models.providers` (URL personalizzato/base)

Usa `models.providers` (o `models.json`) per aggiungere provider **personalizzati** o
proxy compatibili con OpenAI/Anthropic.

Molti dei plugin provider bundled qui sotto pubblicano già un catalogo predefinito.
Usa voci esplicite `models.providers.<id>` solo quando vuoi sovrascrivere
URL base, header o elenco dei modelli predefiniti.

### Moonshot AI (Kimi)

Moonshot è incluso come plugin provider bundled. Usa il provider integrato per
impostazione predefinita e aggiungi una voce esplicita `models.providers.moonshot` solo quando
devi sovrascrivere l'URL base o i metadati del modello:

- Provider: `moonshot`
- Auth: `MOONSHOT_API_KEY`
- Modello di esempio: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` o `openclaw onboard --auth-choice moonshot-api-key-cn`

ID modello Kimi K2:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.5`
- `moonshot/kimi-k2-thinking`
- `moonshot/kimi-k2-thinking-turbo`
- `moonshot/kimi-k2-turbo`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.5", name: "Kimi K2.5" }],
      },
    },
  },
}
```

### Kimi Coding

Kimi Coding usa l'endpoint compatibile con Anthropic di Moonshot AI:

- Provider: `kimi`
- Auth: `KIMI_API_KEY`
- Modello di esempio: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

Il legacy `kimi/k2p5` resta accettato come id modello di compatibilità.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎) fornisce accesso a Doubao e ad altri modelli in Cina.

- Provider: `volcengine` (coding: `volcengine-plan`)
- Auth: `VOLCANO_ENGINE_API_KEY`
- Modello di esempio: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

L'onboarding usa per impostazione predefinita la superficie coding, ma il catalogo generale `volcengine/*`
viene registrato nello stesso momento.

Nei selettori di onboarding/configurazione del modello, la scelta di autenticazione Volcengine preferisce entrambe le
righe `volcengine/*` e `volcengine-plan/*`. Se quei modelli non sono ancora caricati,
OpenClaw torna al catalogo non filtrato invece di mostrare un selettore vuoto
limitato al provider.

Modelli disponibili:

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

Modelli coding (`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (internazionale)

BytePlus ARK fornisce accesso agli stessi modelli di Volcano Engine per gli utenti internazionali.

- Provider: `byteplus` (coding: `byteplus-plan`)
- Auth: `BYTEPLUS_API_KEY`
- Modello di esempio: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

L'onboarding usa per impostazione predefinita la superficie coding, ma il catalogo generale `byteplus/*`
viene registrato nello stesso momento.

Nei selettori di onboarding/configurazione del modello, la scelta di autenticazione BytePlus preferisce entrambe le
righe `byteplus/*` e `byteplus-plan/*`. Se quei modelli non sono ancora caricati,
OpenClaw torna al catalogo non filtrato invece di mostrare un selettore vuoto
limitato al provider.

Modelli disponibili:

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

Modelli coding (`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Synthetic fornisce modelli compatibili con Anthropic dietro il provider `synthetic`:

- Provider: `synthetic`
- Auth: `SYNTHETIC_API_KEY`
- Modello di esempio: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M2.5" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M2.5", name: "MiniMax M2.5" }],
      },
    },
  },
}
```

### MiniMax

MiniMax è configurato tramite `models.providers` perché usa endpoint personalizzati:

- MiniMax OAuth (globale): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax chiave API (globale): `--auth-choice minimax-global-api`
- MiniMax chiave API (CN): `--auth-choice minimax-cn-api`
- Auth: `MINIMAX_API_KEY` per `minimax`; `MINIMAX_OAUTH_TOKEN` o
  `MINIMAX_API_KEY` per `minimax-portal`

Vedi [/providers/minimax](/it/providers/minimax) per dettagli di configurazione, opzioni dei modelli e snippet di configurazione.

Sul percorso di streaming compatibile con Anthropic di MiniMax, OpenClaw disattiva il thinking per
impostazione predefinita a meno che non venga impostato esplicitamente, e `/fast on` riscrive
`MiniMax-M2.7` in `MiniMax-M2.7-highspeed`.

Suddivisione delle capability gestite dal plugin:

- I valori predefiniti per testo/chat restano su `minimax/MiniMax-M2.7`
- La generazione di immagini è `minimax/image-01` o `minimax-portal/image-01`
- La comprensione delle immagini è `MiniMax-VL-01`, gestita dal plugin, su entrambi i percorsi auth MiniMax
- La ricerca web resta sull'id provider `minimax`

### Ollama

Ollama è incluso come plugin provider bundled e usa l'API nativa di Ollama:

- Provider: `ollama`
- Auth: nessuna richiesta (server locale)
- Modello di esempio: `ollama/llama3.3`
- Installazione: [https://ollama.com/download](https://ollama.com/download)

```bash
# Installa Ollama, poi scarica un modello:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama viene rilevato localmente su `http://127.0.0.1:11434` quando abiliti l'opt-in con
`OLLAMA_API_KEY`, e il plugin provider bundled aggiunge Ollama direttamente a
`openclaw onboard` e al selettore del modello. Vedi [/providers/ollama](/it/providers/ollama)
per onboarding, modalità cloud/locale e configurazione personalizzata.

### vLLM

vLLM è incluso come plugin provider bundled per server
compatibili con OpenAI locali/self-hosted:

- Provider: `vllm`
- Auth: facoltativa (dipende dal tuo server)
- URL base predefinito: `http://127.0.0.1:8000/v1`

Per abilitare l'auto-discovery in locale (qualsiasi valore va bene se il tuo server non impone l'autenticazione):

```bash
export VLLM_API_KEY="vllm-local"
```

Poi imposta un modello (sostituisci con uno degli ID restituiti da `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

Vedi [/providers/vllm](/it/providers/vllm) per i dettagli.

### SGLang

SGLang è incluso come plugin provider bundled per server
compatibili con OpenAI self-hosted veloci:

- Provider: `sglang`
- Auth: facoltativa (dipende dal tuo server)
- URL base predefinito: `http://127.0.0.1:30000/v1`

Per abilitare l'auto-discovery in locale (qualsiasi valore va bene se il tuo server non
impone l'autenticazione):

```bash
export SGLANG_API_KEY="sglang-local"
```

Poi imposta un modello (sostituisci con uno degli ID restituiti da `/v1/models`):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

Vedi [/providers/sglang](/it/providers/sglang) per i dettagli.

### Proxy locali (LM Studio, vLLM, LiteLLM, ecc.)

Esempio (compatibile con OpenAI):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "LMSTUDIO_KEY",
        api: "openai-completions",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Note:

- Per i provider personalizzati, `reasoning`, `input`, `cost`, `contextWindow` e `maxTokens` sono facoltativi.
  Se omessi, OpenClaw usa i valori predefiniti:
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- Consigliato: imposta valori espliciti che corrispondano ai limiti del tuo proxy/modello.
- Per `api: "openai-completions"` su endpoint non nativi (qualsiasi `baseUrl` non vuoto il cui host non sia `api.openai.com`), OpenClaw forza `compat.supportsDeveloperRole: false` per evitare errori 400 del provider dovuti a ruoli `developer` non supportati.
- I percorsi compatibili con OpenAI in stile proxy saltano anche la modellazione nativa delle richieste solo-OpenAI: niente `service_tier`, niente `store` di Responses, niente hint di prompt-cache, niente modellazione del payload di compatibilità reasoning OpenAI e niente header di attribuzione OpenClaw nascosti.
- Se `baseUrl` è vuoto/omesso, OpenClaw mantiene il comportamento OpenAI predefinito (che risolve in `api.openai.com`).
- Per sicurezza, un `compat.supportsDeveloperRole: true` esplicito viene comunque sovrascritto sugli endpoint `openai-completions` non nativi.

## Esempi CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

Vedi anche: [/gateway/configuration](/it/gateway/configuration) per esempi di configurazione completi.

## Correlati

- [Models](/it/concepts/models) — configurazione dei modelli e alias
- [Model Failover](/it/concepts/model-failover) — catene di fallback e comportamento dei retry
- [Riferimento configurazione](/it/gateway/configuration-reference#agent-defaults) — chiavi di configurazione del modello
- [Providers](/it/providers) — guide di configurazione per provider
