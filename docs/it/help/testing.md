---
read_when:
    - Esecuzione dei test in locale o in CI
    - Aggiunta di regressioni per bug di modelli/provider
    - Debug del comportamento di gateway + agent
summary: 'Kit di test: suite unit/e2e/live, runner Docker e cosa copre ciascun test'
title: Testing
x-i18n:
    generated_at: "2026-04-06T03:09:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: cfa174e565df5fdf957234b7909beaf1304aa026e731cc2c433ca7d931681b56
    source_path: help/testing.md
    workflow: 15
---

# Testing

OpenClaw ha tre suite Vitest (unit/integration, e2e, live) e un piccolo insieme di runner Docker.

Questa documentazione è una guida su “come testiamo”:

- Cosa copre ciascuna suite (e cosa deliberatamente _non_ copre)
- Quali comandi eseguire per i flussi di lavoro comuni (locale, pre-push, debug)
- Come i test live individuano le credenziali e selezionano modelli/provider
- Come aggiungere regressioni per problemi reali di modelli/provider

## Guida rapida

Nella maggior parte dei giorni:

- Gate completo (atteso prima del push): `pnpm build && pnpm check && pnpm test`
- Esecuzione locale più rapida della suite completa su una macchina capiente: `pnpm test:max`
- Loop diretto di watch Vitest (config projects moderna): `pnpm test:watch`
- Il targeting diretto dei file ora instrada anche i percorsi extension/channel: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`

Quando tocchi i test o vuoi maggiore sicurezza:

- Gate di copertura: `pnpm test:coverage`
- Suite E2E: `pnpm test:e2e`

Quando fai debug di provider/modelli reali (richiede credenziali reali):

- Suite live (sonde modelli + gateway tool/image): `pnpm test:live`
- Esegui in modo silenzioso un solo file live: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

Suggerimento: quando ti serve solo un caso in errore, preferisci restringere i test live tramite le variabili env di allowlist descritte sotto.

## Suite di test (cosa viene eseguito e dove)

Pensa alle suite come a “realismo crescente” (e crescente instabilità/costo):

### Unit / integration (predefinita)

- Comando: `pnpm test`
- Config: `projects` Vitest nativi tramite `vitest.config.ts`
- File: inventari core/unit in `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` e i test node `ui` in allowlist coperti da `vitest.unit.config.ts`
- Ambito:
  - Test unitari puri
  - Test di integrazione in-process (auth gateway, routing, tooling, parsing, config)
  - Regressioni deterministiche per bug noti
- Aspettative:
  - Viene eseguita in CI
  - Non richiede chiavi reali
  - Dovrebbe essere veloce e stabile
- Nota sui projects:
  - `pnpm test`, `pnpm test:watch` e `pnpm test:changed` usano ora tutti la stessa config root `projects` nativa di Vitest.
  - I filtri diretti sui file passano in modo nativo attraverso il grafo dei progetti root, quindi `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` funziona senza un wrapper personalizzato.
- Nota sull'embedded runner:
  - Quando modifichi gli input di discovery dei message-tool o il contesto di runtime della compattazione,
    mantieni entrambi i livelli di copertura.
  - Aggiungi regressioni mirate degli helper per confini puri di routing/normalizzazione.
  - Mantieni anche in buono stato le suite di integrazione dell'embedded runner:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts` e
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - Queste suite verificano che gli id con scope e il comportamento di compattazione continuino a fluire
    attraverso i veri percorsi `run.ts` / `compact.ts`; i soli test degli helper non sono un
    sostituto sufficiente di questi percorsi di integrazione.
- Nota sul pool:
  - La config Vitest di base ora usa `threads` come predefinito.
  - La config Vitest condivisa imposta anche `isolate: false` e usa il runner non isolato per i progetti root, e2e e live.
  - La lane UI root mantiene il suo setup `jsdom` e l'optimizer, ma ora gira anch'essa sul runner condiviso non isolato.
  - `pnpm test` eredita gli stessi valori predefiniti `threads` + `isolate: false` dalla config `projects` di root in `vitest.config.ts`.
  - Il launcher condiviso `scripts/run-vitest.mjs` ora aggiunge anche `--no-maglev` per impostazione predefinita ai processi child Node di Vitest per ridurre il churn di compilazione V8 durante grandi esecuzioni locali. Imposta `OPENCLAW_VITEST_ENABLE_MAGLEV=1` se devi confrontare il comportamento V8 standard.
- Nota sull'iterazione locale veloce:
  - `pnpm test:changed` esegue la config nativa dei projects con `--changed origin/main`.
  - `pnpm test:max` e `pnpm test:changed:max` mantengono la stessa config nativa dei projects, solo con un limite di worker più alto.
  - L'auto-scaling locale dei worker ora è intenzionalmente conservativo e riduce anche quando il load average dell'host è già alto, così più esecuzioni concorrenti di Vitest fanno meno danni per impostazione predefinita.
  - La config Vitest di base contrassegna i file di progetto/config come `forceRerunTriggers` così i rerun in modalità changed restano corretti quando cambia il wiring dei test.
  - La config mantiene `OPENCLAW_VITEST_FS_MODULE_CACHE` abilitato sugli host supportati; imposta `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` se vuoi una posizione cache esplicita per il profiling diretto.
- Nota sul debug delle prestazioni:
  - `pnpm test:perf:imports` abilita la reportistica della durata di import di Vitest più l'output di dettaglio degli import.
  - `pnpm test:perf:imports:changed` limita la stessa vista di profiling ai file modificati rispetto a `origin/main`.
  - `pnpm test:perf:profile:main` scrive un profilo CPU del thread principale per l'overhead di avvio e trasformazione di Vitest/Vite.
  - `pnpm test:perf:profile:runner` scrive profili CPU+heap del runner per la suite unit con parallelismo dei file disabilitato.

### E2E (smoke del gateway)

- Comando: `pnpm test:e2e`
- Config: `vitest.e2e.config.ts`
- File: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- Valori predefiniti di runtime:
  - Usa Vitest `threads` con `isolate: false`, in linea con il resto del repo.
  - Usa worker adattivi (CI: fino a 2, locale: 1 per impostazione predefinita).
  - Viene eseguita in modalità silent per impostazione predefinita per ridurre l'overhead di I/O su console.
- Override utili:
  - `OPENCLAW_E2E_WORKERS=<n>` per forzare il numero di worker (massimo 16).
  - `OPENCLAW_E2E_VERBOSE=1` per riabilitare output dettagliato su console.
- Ambito:
  - Comportamento end-to-end del gateway multi-instance
  - Superfici WebSocket/HTTP, pairing dei node e networking più pesante
- Aspettative:
  - Viene eseguita in CI (quando abilitata nella pipeline)
  - Non richiede chiavi reali
  - Ha più parti in movimento rispetto ai test unitari (può essere più lenta)

### E2E: smoke del backend OpenShell

- Comando: `pnpm test:e2e:openshell`
- File: `test/openshell-sandbox.e2e.test.ts`
- Ambito:
  - Avvia sull'host un gateway OpenShell isolato tramite Docker
  - Crea una sandbox da un Dockerfile locale temporaneo
  - Esegue il backend OpenShell di OpenClaw tramite veri `sandbox ssh-config` + exec SSH
  - Verifica il comportamento del filesystem canonico remoto attraverso il bridge fs della sandbox
- Aspettative:
  - Solo opt-in; non fa parte dell'esecuzione predefinita `pnpm test:e2e`
  - Richiede una CLI locale `openshell` più un demone Docker funzionante
  - Usa `HOME` / `XDG_CONFIG_HOME` isolati, poi distrugge il gateway e la sandbox di test
- Override utili:
  - `OPENCLAW_E2E_OPENSHELL=1` per abilitare il test quando esegui manualmente la suite e2e più ampia
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` per puntare a una CLI binaria non predefinita o a uno script wrapper

### Live (provider reali + modelli reali)

- Comando: `pnpm test:live`
- Config: `vitest.live.config.ts`
- File: `src/**/*.live.test.ts`
- Predefinita: **abilitata** da `pnpm test:live` (imposta `OPENCLAW_LIVE_TEST=1`)
- Ambito:
  - “Questo provider/modello funziona davvero _oggi_ con credenziali reali?”
  - Individua cambiamenti di formato del provider, particolarità del tool-calling, problemi di auth e comportamento dei rate limit
- Aspettative:
  - Per definizione non è stabile in CI (reti reali, policy reali dei provider, quote, outage)
  - Costa denaro / usa rate limit
  - Preferisci eseguire sottoinsiemi ristretti invece di “tutto”
- Le esecuzioni live usano `~/.profile` per recuperare eventuali chiavi API mancanti.
- Per impostazione predefinita, le esecuzioni live isolano comunque `HOME` e copiano config/materiale auth in una home di test temporanea, così i fixture unit non possono mutare il tuo vero `~/.openclaw`.
- Imposta `OPENCLAW_LIVE_USE_REAL_HOME=1` solo quando hai intenzionalmente bisogno che i test live usino la tua vera home directory.
- `pnpm test:live` ora usa per impostazione predefinita una modalità più silenziosa: mantiene l'output di avanzamento `[live] ...`, ma sopprime l'avviso extra su `~/.profile` e silenzia i log di bootstrap del gateway / il rumore Bonjour. Imposta `OPENCLAW_LIVE_TEST_QUIET=0` se vuoi riavere i log completi di avvio.
- Rotazione delle API key (specifica per provider): imposta `*_API_KEYS` in formato virgola/punto e virgola o `*_API_KEY_1`, `*_API_KEY_2` (ad esempio `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`) oppure usa l'override per-live `OPENCLAW_LIVE_*_KEY`; i test ritentano sulle risposte di rate limit.
- Output di avanzamento/heartbeat:
  - Le suite live ora emettono righe di avanzamento su stderr così le chiamate lunghe ai provider risultano visibilmente attive anche quando il capture della console di Vitest è silenzioso.
  - `vitest.live.config.ts` disabilita l'intercettazione della console di Vitest così le righe di avanzamento provider/gateway vengono trasmesse immediatamente durante le esecuzioni live.
  - Regola gli heartbeat dei modelli diretti con `OPENCLAW_LIVE_HEARTBEAT_MS`.
  - Regola gli heartbeat di gateway/sonde con `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`.

## Quale suite devo eseguire?

Usa questa tabella decisionale:

- Modifica di logica/test: esegui `pnpm test` (e `pnpm test:coverage` se hai cambiato molto)
- Tocco del networking del gateway / protocollo WS / pairing: aggiungi `pnpm test:e2e`
- Debug di “il mio bot è giù” / errori specifici del provider / tool calling: esegui un `pnpm test:live` ristretto

## Live: sweep delle capability del node Android

- Test: `src/gateway/android-node.capabilities.live.test.ts`
- Script: `pnpm android:test:integration`
- Obiettivo: invocare **ogni comando attualmente pubblicizzato** da un node Android connesso e verificare il comportamento del contratto del comando.
- Ambito:
  - Setup precondizionato/manuale (la suite non installa/esegue/abbina l'app).
  - Validazione `node.invoke` del gateway comando per comando per il node Android selezionato.
- Pre-setup richiesto:
  - App Android già connessa + abbinata al gateway.
  - App mantenuta in foreground.
  - Permessi/consenso alla cattura concessi per le capability che ti aspetti passino.
- Override facoltativi del target:
  - `OPENCLAW_ANDROID_NODE_ID` o `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- Dettagli completi della configurazione Android: [Android App](/it/platforms/android)

## Live: smoke dei modelli (chiavi profilo)

I test live sono divisi in due livelli così possiamo isolare gli errori:

- “Modello diretto” ci dice se provider/modello riescono almeno a rispondere con la chiave fornita.
- “Smoke del gateway” ci dice se l'intera pipeline gateway+agent funziona per quel modello (sessioni, cronologia, strumenti, criteri sandbox, ecc.).

### Livello 1: completamento diretto del modello (senza gateway)

- Test: `src/agents/models.profiles.live.test.ts`
- Obiettivo:
  - Enumerare i modelli individuati
  - Usare `getApiKeyForModel` per selezionare i modelli per cui hai credenziali
  - Eseguire un piccolo completamento per modello (e regressioni mirate dove necessario)
- Come abilitarlo:
  - `pnpm test:live` (oppure `OPENCLAW_LIVE_TEST=1` se invochi Vitest direttamente)
- Imposta `OPENCLAW_LIVE_MODELS=modern` (o `all`, alias di modern) per eseguire davvero questa suite; altrimenti viene saltata per mantenere `pnpm test:live` focalizzato sullo smoke del gateway
- Come selezionare i modelli:
  - `OPENCLAW_LIVE_MODELS=modern` per eseguire l'allowlist moderna (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_MODELS=all` è un alias dell'allowlist moderna
  - oppure `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."` (allowlist separata da virgole)
- Come selezionare i provider:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity"` (allowlist separata da virgole)
- Da dove arrivano le chiavi:
  - Per impostazione predefinita: store dei profili e fallback env
  - Imposta `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` per imporre **solo lo store dei profili**
- Perché esiste:
  - Separa “l'API del provider è rotta / la chiave non è valida” da “la pipeline agent del gateway è rotta”
  - Contiene regressioni piccole e isolate (esempio: replay del ragionamento OpenAI Responses/Codex Responses + flussi di tool-call)

### Livello 2: smoke del gateway + agent dev (quello che fa davvero "@openclaw")

- Test: `src/gateway/gateway-models.profiles.live.test.ts`
- Obiettivo:
  - Avviare un gateway in-process
  - Creare/patchare una sessione `agent:dev:*` (override del modello per esecuzione)
  - Iterare i modelli-con-chiavi e verificare:
    - risposta “significativa” (senza strumenti)
    - una vera invocazione di strumento funziona (sonda read)
    - sonde di strumenti extra facoltative (sonda exec+read)
    - i percorsi di regressione OpenAI (solo tool-call → follow-up) continuano a funzionare
- Dettagli delle sonde (così puoi spiegare rapidamente gli errori):
  - sonda `read`: il test scrive un file nonce nel workspace e chiede all'agent di `read` leggerlo e restituire il nonce.
  - sonda `exec+read`: il test chiede all'agent di scrivere via `exec` un nonce in un file temporaneo, poi di `read` rileggerlo.
  - sonda image: il test allega un PNG generato (gatto + codice randomizzato) e si aspetta che il modello restituisca `cat <CODE>`.
  - Riferimento implementativo: `src/gateway/gateway-models.profiles.live.test.ts` e `src/gateway/live-image-probe.ts`.
- Come abilitarlo:
  - `pnpm test:live` (oppure `OPENCLAW_LIVE_TEST=1` se invochi Vitest direttamente)
- Come selezionare i modelli:
  - Predefinito: allowlist moderna (Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` è un alias dell'allowlist moderna
  - Oppure imposta `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (o lista separata da virgole) per restringere
- Come selezionare i provider (evitare “tutto OpenRouter”):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,openai,anthropic,zai,minimax"` (allowlist separata da virgole)
- Le sonde tool + image sono sempre attive in questo test live:
  - sonda `read` + sonda `exec+read` (stress dei tool)
  - la sonda image viene eseguita quando il modello dichiara supporto per input image
  - Flusso (alto livello):
    - Il test genera un piccolo PNG con “CAT” + codice casuale (`src/gateway/live-image-probe.ts`)
    - Lo invia tramite `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]`
    - Il gateway analizza gli allegati in `images[]` (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - L'embedded agent inoltra al modello un messaggio utente multimodale
    - Verifica: la risposta contiene `cat` + il codice (tolleranza OCR: piccoli errori consentiti)

Suggerimento: per vedere cosa puoi testare sulla tua macchina (e gli id esatti `provider/model`), esegui:

```bash
openclaw models list
openclaw models list --json
```

## Live: smoke ACP bind (`/acp spawn ... --bind here`)

- Test: `src/gateway/gateway-acp-bind.live.test.ts`
- Obiettivo: validare il vero flusso di conversation-bind ACP con un agent ACP live:
  - inviare `/acp spawn <agent> --bind here`
  - associare sul posto una conversazione sintetica di message-channel
  - inviare un normale follow-up sulla stessa conversazione
  - verificare che il follow-up arrivi nella trascrizione della sessione ACP associata
- Abilitazione:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- Valori predefiniti:
  - Agent ACP: `claude`
  - Canale sintetico: contesto conversazione in stile DM Slack
  - Backend ACP: `acpx`
- Override:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- Note:
  - Questa lane usa la superficie `chat.send` del gateway con campi di originating-route sintetici riservati agli admin così i test possono allegare il contesto di un message-channel senza fingere una consegna esterna.
  - Quando `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` non è impostato, il test usa il registro agent integrato del plugin `acpx` embedded per l'agent ACP harness selezionato.

Esempio:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Ricetta Docker:

```bash
pnpm test:docker:live-acp-bind
```

Note Docker:

- Il runner Docker si trova in `scripts/test-live-acp-bind-docker.sh`.
- Usa `~/.profile`, prepara nel container il materiale auth CLI corrispondente, installa `acpx` in un prefisso npm scrivibile, quindi installa la CLI live richiesta (`@anthropic-ai/claude-code` oppure `@openai/codex`) se manca.
- Dentro Docker, il runner imposta `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx` così acpx mantiene disponibili alla CLI harness figlia le env var del provider provenienti dal profilo caricato.

### Ricette live consigliate

Allowlist ristrette ed esplicite sono le più veloci e meno instabili:

- Modello singolo, diretto (senza gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- Modello singolo, smoke del gateway:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Tool calling su più provider:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Focus Google (chiave API Gemini + Antigravity):
  - Gemini (chiave API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

Note:

- `google/...` usa l'API Gemini (chiave API).
- `google-antigravity/...` usa il bridge OAuth Antigravity (endpoint agent in stile Cloud Code Assist).

## Live: matrice dei modelli (cosa copriamo)

Non esiste un “elenco modelli CI” fisso (live è opt-in), ma questi sono i modelli **consigliati** da coprire regolarmente su una macchina di sviluppo con chiavi.

### Set smoke moderno (tool calling + image)

Questa è l'esecuzione dei “modelli comuni” che ci aspettiamo di mantenere funzionante:

- OpenAI (non-Codex): `openai/gpt-5.4` (facoltativo: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6` (oppure `anthropic/claude-sonnet-4-6`)
- Google (API Gemini): `google/gemini-3.1-pro-preview` e `google/gemini-3-flash-preview` (evita i vecchi modelli Gemini 2.x)
- Google (Antigravity): `google-antigravity/claude-opus-4-6-thinking` e `google-antigravity/gemini-3-flash`
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Esegui lo smoke del gateway con tools + image:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### Baseline: tool calling (Read + Exec facoltativo)

Scegline almeno uno per famiglia di provider:

- OpenAI: `openai/gpt-5.4` (oppure `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6` (oppure `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview` (oppure `google/gemini-3.1-pro-preview`)
- Z.AI (GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

Copertura aggiuntiva facoltativa (utile da avere):

- xAI: `xai/grok-4` (o l'ultima disponibile)
- Mistral: `mistral/`… (scegli un modello “tools” che hai abilitato)
- Cerebras: `cerebras/`… (se hai accesso)
- LM Studio: `lmstudio/`… (locale; il tool calling dipende dalla modalità API)

### Vision: invio image (allegato → messaggio multimodale)

Includi almeno un modello con capacità image in `OPENCLAW_LIVE_GATEWAY_MODELS` (varianti Claude/Gemini/OpenAI con supporto vision, ecc.) per esercitare la sonda image.

### Aggregatori / gateway alternativi

Se hai chiavi abilitate, supportiamo anche i test tramite:

- OpenRouter: `openrouter/...` (centinaia di modelli; usa `openclaw models scan` per trovare candidati con tool+image)
- OpenCode: `opencode/...` per Zen e `opencode-go/...` per Go (auth tramite `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

Altri provider che puoi includere nella matrice live (se hai credenziali/config):

- Integrati: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- Tramite `models.providers` (endpoint personalizzati): `minimax` (cloud/API), più qualsiasi proxy compatibile OpenAI/Anthropic (LM Studio, vLLM, LiteLLM, ecc.)

Suggerimento: non cercare di codificare in modo rigido “tutti i modelli” nella documentazione. L'elenco autorevole è ciò che `discoverModels(...)` restituisce sulla tua macchina + le chiavi disponibili.

## Credenziali (non fare mai commit)

I test live individuano le credenziali nello stesso modo della CLI. Implicazioni pratiche:

- Se la CLI funziona, i test live dovrebbero trovare le stesse chiavi.
- Se un test live dice “nessuna credenziale”, fai debug come faresti con `openclaw models list` / selezione del modello.

- Profili auth per-agent: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (questo è il significato di “chiavi profilo” nei test live)
- Config: `~/.openclaw/openclaw.json` (oppure `OPENCLAW_CONFIG_PATH`)
- Directory legacy di stato: `~/.openclaw/credentials/` (copiata nella home live temporanea quando presente, ma non è lo store principale delle chiavi profilo)
- Le esecuzioni locali live copiano per impostazione predefinita la config attiva, i file `auth-profiles.json` per-agent, la directory legacy `credentials/` e le directory auth CLI esterne supportate in una home di test temporanea; gli override di percorso `agents.*.workspace` / `agentDir` vengono rimossi in quella config temporanea così le sonde restano fuori dal tuo vero workspace host.

Se vuoi affidarti alle chiavi env (ad esempio esportate in `~/.profile`), esegui i test locali dopo `source ~/.profile`, oppure usa i runner Docker qui sotto (possono montare `~/.profile` nel container).

## Live Deepgram (trascrizione audio)

- Test: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- Abilitazione: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## Live BytePlus coding plan

- Test: `src/agents/byteplus.live.test.ts`
- Abilitazione: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- Override facoltativo del modello: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## Live media workflow ComfyUI

- Test: `extensions/comfy/comfy.live.test.ts`
- Abilitazione: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- Ambito:
  - Esercita i percorsi image, video e `music_generate` del comfy bundle integrato
  - Salta ogni capability a meno che `models.providers.comfy.<capability>` non sia configurato
  - Utile dopo modifiche all'invio dei workflow comfy, polling, download o registrazione plugin

## Live image generation

- Test: `src/image-generation/runtime.live.test.ts`
- Comando: `pnpm test:live src/image-generation/runtime.live.test.ts`
- Ambito:
  - Enumera ogni plugin provider di image-generation registrato
  - Carica le variabili env del provider mancanti dalla tua shell di login (`~/.profile`) prima delle sonde
  - Usa per impostazione predefinita le API key live/env prima dei profili auth memorizzati, così chiavi di test obsolete in `auth-profiles.json` non mascherano credenziali shell reali
  - Salta i provider senza auth/profilo/modello utilizzabili
  - Esegue le varianti stock di image-generation attraverso la capability runtime condivisa:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- Provider integrati attualmente coperti:
  - `openai`
  - `google`
- Restrizione facoltativa:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- Comportamento auth facoltativo:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` per forzare auth dallo store profili e ignorare gli override solo env

## Live music generation

- Test: `extensions/music-generation-providers.live.test.ts`
- Abilitazione: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- Ambito:
  - Esercita il percorso condiviso del provider music-generation integrato
  - Attualmente copre Google e MiniMax
  - Carica le variabili env del provider dalla tua shell di login (`~/.profile`) prima delle sonde
  - Salta i provider senza auth/profilo/modello utilizzabili
- Restrizione facoltativa:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`

## Runner Docker (controlli facoltativi "funziona su Linux")

Questi runner Docker sono divisi in due gruppi:

- Runner live-model: `test:docker:live-models` e `test:docker:live-gateway` eseguono solo il file live con chiavi profilo corrispondente dentro l'immagine Docker del repo (`src/agents/models.profiles.live.test.ts` e `src/gateway/gateway-models.profiles.live.test.ts`), montando la tua directory config e workspace locali (e usando `~/.profile` se montato). Gli entrypoint locali corrispondenti sono `test:live:models-profiles` e `test:live:gateway-profiles`.
- I runner live Docker usano per impostazione predefinita un limite smoke più piccolo così una sweep Docker completa resta pratica:
  `test:docker:live-models` usa come predefinito `OPENCLAW_LIVE_MAX_MODELS=12`, e
  `test:docker:live-gateway` usa come predefiniti `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` e
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`. Sostituisci queste variabili env quando
  vuoi esplicitamente la scansione esaustiva più ampia.
- `test:docker:all` costruisce l'immagine Docker live una sola volta tramite `test:docker:live-build`, poi la riutilizza per le due lane live Docker.
- Runner smoke per container: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels` e `test:docker:plugins` avviano uno o più container reali e verificano percorsi di integrazione di livello più alto.

I runner Docker live-model montano inoltre in bind solo le home auth CLI necessarie (o tutte quelle supportate quando l'esecuzione non è ristretta), poi le copiano nella home del container prima dell'esecuzione così l'OAuth della CLI esterna può aggiornare i token senza mutare lo store auth dell'host:

- Modelli diretti: `pnpm test:docker:live-models` (script: `scripts/test-live-models-docker.sh`)
- Smoke ACP bind: `pnpm test:docker:live-acp-bind` (script: `scripts/test-live-acp-bind-docker.sh`)
- Gateway + agent dev: `pnpm test:docker:live-gateway` (script: `scripts/test-live-gateway-models-docker.sh`)
- Smoke live Open WebUI: `pnpm test:docker:openwebui` (script: `scripts/e2e/openwebui-docker.sh`)
- Wizard di onboarding (TTY, scaffolding completo): `pnpm test:docker:onboard` (script: `scripts/e2e/onboard-docker.sh`)
- Networking del gateway (due container, auth WS + health): `pnpm test:docker:gateway-network` (script: `scripts/e2e/gateway-network-docker.sh`)
- Bridge canale MCP (Gateway seedato + bridge stdio + smoke raw Claude notification-frame): `pnpm test:docker:mcp-channels` (script: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins (smoke di installazione + alias `/plugin` + semantica di riavvio del bundle Claude): `pnpm test:docker:plugins` (script: `scripts/e2e/plugins-docker.sh`)

I runner Docker live-model montano inoltre in bind la checkout corrente in sola lettura e
la preparano in uno workdir temporaneo dentro il container. Questo mantiene snella l'immagine
runtime pur eseguendo Vitest esattamente contro il tuo sorgente/config locale.
Il passaggio di staging salta grandi cache solo locali e output di build delle app come
`.pnpm-store`, `.worktrees`, `__openclaw_vitest__` e directory di output locali `.build` o
Gradle così le esecuzioni live Docker non passano minuti a copiare artefatti
specifici della macchina.
Impostano anche `OPENCLAW_SKIP_CHANNELS=1` così le sonde live del gateway non avviano
veri worker di canale Telegram/Discord/ecc. dentro il container.
`test:docker:live-models` esegue comunque `pnpm test:live`, quindi passa anche
`OPENCLAW_LIVE_GATEWAY_*` quando devi restringere o escludere la copertura
live del gateway da quella lane Docker.
`test:docker:openwebui` è uno smoke di compatibilità di livello superiore: avvia un
container gateway OpenClaw con gli endpoint HTTP compatibili OpenAI abilitati,
avvia un container Open WebUI fissato contro quel gateway, effettua il login tramite
Open WebUI, verifica che `/api/models` esponga `openclaw/default`, poi invia una
vera richiesta chat tramite il proxy `/api/chat/completions` di Open WebUI.
La prima esecuzione può essere sensibilmente più lenta perché Docker potrebbe dover scaricare l'immagine
Open WebUI e Open WebUI potrebbe dover completare il proprio setup cold-start.
Questa lane si aspetta una chiave di modello live utilizzabile, e `OPENCLAW_PROFILE_FILE`
(`~/.profile` per impostazione predefinita) è il modo principale per fornirla nelle esecuzioni Docker.
Le esecuzioni riuscite stampano un piccolo payload JSON come `{ "ok": true, "model":
"openclaw/default", ... }`.
`test:docker:mcp-channels` è intenzionalmente deterministico e non richiede un
vero account Telegram, Discord o iMessage. Avvia un container Gateway seedato,
avvia un secondo container che lancia `openclaw mcp serve`, poi
verifica discovery delle conversazioni instradate, letture della trascrizione, metadati
degli allegati, comportamento della coda eventi live, instradamento degli invii in uscita e notifiche di canale +
permessi in stile Claude sul vero bridge MCP stdio. Il controllo delle notifiche
ispeziona direttamente i frame MCP stdio grezzi così lo smoke valida ciò che il
bridge emette davvero, non solo ciò che un determinato SDK client espone.

Smoke manuale ACP plain-language thread (non CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- Mantieni questo script per i flussi di lavoro di regressione/debug. Potrebbe servire di nuovo per la validazione del routing dei thread ACP, quindi non eliminarlo.

Variabili env utili:

- `OPENCLAW_CONFIG_DIR=...` (predefinita: `~/.openclaw`) montata su `/home/node/.openclaw`
- `OPENCLAW_WORKSPACE_DIR=...` (predefinita: `~/.openclaw/workspace`) montata su `/home/node/.openclaw/workspace`
- `OPENCLAW_PROFILE_FILE=...` (predefinita: `~/.profile`) montata su `/home/node/.profile` e caricata prima di eseguire i test
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (predefinita: `~/.cache/openclaw/docker-cli-tools`) montata su `/home/node/.npm-global` per installazioni CLI cache dentro Docker
- Le directory/file auth della CLI esterna sotto `$HOME` vengono montati in sola lettura sotto `/host-auth...`, poi copiati in `/home/node/...` prima dell'avvio dei test
  - Directory predefinite: `.minimax`
  - File predefiniti: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - Le esecuzioni ristrette per provider montano solo le directory/file necessari dedotti da `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`
  - Override manuale con `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none` o una lista separata da virgole come `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex`
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` per restringere l'esecuzione
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` per filtrare i provider dentro il container
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` per garantire che le credenziali provengano dallo store profili (non da env)
- `OPENCLAW_OPENWEBUI_MODEL=...` per scegliere il modello esposto dal gateway per lo smoke Open WebUI
- `OPENCLAW_OPENWEBUI_PROMPT=...` per sovrascrivere il prompt di controllo nonce usato dallo smoke Open WebUI
- `OPENWEBUI_IMAGE=...` per sovrascrivere il tag immagine Open WebUI fissato

## Sanity check della documentazione

Esegui i controlli docs dopo modifiche alla documentazione: `pnpm check:docs`.
Esegui la validazione completa degli anchor Mintlify quando ti servono anche i controlli dei titoli in-page: `pnpm docs:check-links:anchors`.

## Regressione offline (sicura per CI)

Queste sono regressioni della “pipeline reale” senza provider reali:

- Tool calling del gateway (mock OpenAI, vero loop gateway + agent): `src/gateway/gateway.test.ts` (caso: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Wizard del gateway (WS `wizard.start`/`wizard.next`, scrittura di config + auth obbligatoria): `src/gateway/gateway.test.ts` (caso: "runs wizard over ws and writes auth token config")

## Valutazioni di affidabilità agent (Skills)

Abbiamo già alcuni test sicuri per CI che si comportano come “valutazioni di affidabilità agent”:

- Mock tool-calling attraverso il vero loop gateway + agent (`src/gateway/gateway.test.ts`).
- Flussi wizard end-to-end che validano wiring di sessione ed effetti sulla config (`src/gateway/gateway.test.ts`).

Cosa manca ancora per le Skills (vedi [Skills](/it/tools/skills)):

- **Decisioning:** quando le Skills sono elencate nel prompt, l'agent sceglie la Skill giusta (o evita quelle irrilevanti)?
- **Compliance:** l'agent legge `SKILL.md` prima dell'uso e segue i passaggi/argomenti richiesti?
- **Workflow contracts:** scenari multi-turno che verificano ordine dei tool, carryover della cronologia di sessione e confini della sandbox.

Le valutazioni future dovrebbero rimanere prima di tutto deterministiche:

- Un runner di scenari che usa provider mock per verificare tool call + ordine, letture dei file Skill e wiring della sessione.
- Una piccola suite di scenari focalizzati sulle Skills (usa vs evita, gate, prompt injection).
- Valutazioni live facoltative (opt-in, protette da env) solo dopo che la suite sicura per CI sarà pronta.

## Test di contratto (forma di plugin e channel)

I test di contratto verificano che ogni plugin e channel registrato sia conforme al
proprio contratto di interfaccia. Iterano su tutti i plugin individuati ed eseguono una suite di
verifiche su forma e comportamento. La lane unit predefinita `pnpm test`
salta intenzionalmente questi file smoke e seam condivisi; esegui esplicitamente
i comandi contract quando tocchi superfici condivise di channel o provider.

### Comandi

- Tutti i contratti: `pnpm test:contracts`
- Solo contratti channel: `pnpm test:contracts:channels`
- Solo contratti provider: `pnpm test:contracts:plugins`

### Contratti channel

Situati in `src/channels/plugins/contracts/*.contract.test.ts`:

- **plugin** - Forma base del plugin (id, nome, capability)
- **setup** - Contratto del wizard di setup
- **session-binding** - Comportamento del binding di sessione
- **outbound-payload** - Struttura del payload del messaggio
- **inbound** - Gestione dei messaggi in ingresso
- **actions** - Handler delle azioni del channel
- **threading** - Gestione dell'ID del thread
- **directory** - API directory/roster
- **group-policy** - Applicazione della group policy

### Contratti di stato provider

Situati in `src/plugins/contracts/*.contract.test.ts`.

- **status** - Sonde di stato del channel
- **registry** - Forma del registro plugin

### Contratti provider

Situati in `src/plugins/contracts/*.contract.test.ts`:

- **auth** - Contratto del flusso auth
- **auth-choice** - Scelta/selezione auth
- **catalog** - API del catalogo modelli
- **discovery** - Discovery del plugin
- **loader** - Caricamento del plugin
- **runtime** - Runtime del provider
- **shape** - Forma/interfaccia del plugin
- **wizard** - Wizard di setup

### Quando eseguirli

- Dopo aver modificato export o sottopercorsi del plugin-sdk
- Dopo aver aggiunto o modificato un plugin channel o provider
- Dopo aver rifattorizzato registrazione o discovery dei plugin

I test di contratto vengono eseguiti in CI e non richiedono chiavi API reali.

## Aggiunta di regressioni (linee guida)

Quando correggi un problema di provider/modello scoperto in live:

- Aggiungi una regressione sicura per CI se possibile (provider mock/stub, oppure cattura l'esatta trasformazione della forma della richiesta)
- Se è intrinsecamente solo live (rate limit, policy auth), mantieni il test live ristretto e opt-in tramite variabili env
- Preferisci mirare al livello più piccolo che intercetta il bug:
  - bug di conversione/riproduzione della richiesta provider → test modelli diretti
  - bug della pipeline sessione/cronologia/tool del gateway → smoke live del gateway o test mock del gateway sicuro per CI
- Guardrail di attraversamento SecretRef:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` ricava un target campionato per ogni classe SecretRef dai metadati del registro (`listSecretTargetRegistryEntries()`), quindi verifica che gli id exec dei segmenti di attraversamento vengano rifiutati.
  - Se aggiungi una nuova famiglia di target SecretRef `includeInPlan` in `src/secrets/target-registry-data.ts`, aggiorna `classifyTargetClass` in quel test. Il test fallisce intenzionalmente sugli id target non classificati così le nuove classi non possono essere saltate in silenzio.
