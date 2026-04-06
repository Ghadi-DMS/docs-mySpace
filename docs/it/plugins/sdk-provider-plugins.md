---
read_when:
    - Stai creando un nuovo plugin provider di modelli
    - Vuoi aggiungere a OpenClaw un proxy compatibile con OpenAI o un LLM personalizzato
    - Hai bisogno di comprendere auth provider, cataloghi e hook di runtime
sidebarTitle: Provider Plugins
summary: Guida passo passo per creare un plugin provider di modelli per OpenClaw
title: Creare plugin provider
x-i18n:
    generated_at: "2026-04-06T03:10:47Z"
    model: gpt-5.4
    provider: openai
    source_hash: 69500f46aa2cfdfe16e85b0ed9ee3c0032074be46f2d9c9d2940d18ae1095f47
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# Creare plugin provider

Questa guida ti accompagna nella creazione di un plugin provider che aggiunge un provider di modelli
(LLM) a OpenClaw. Alla fine avrai un provider con un catalogo di modelli,
auth con API key e risoluzione dinamica dei modelli.

<Info>
  Se non hai mai creato prima un plugin OpenClaw, leggi prima
  [Getting Started](/it/plugins/building-plugins) per la struttura di base del package
  e la configurazione del manifest.
</Info>

## Procedura guidata

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Package e manifest">
    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Acme AI model provider",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "providerAuthEnvVars": {
        "acme-ai": ["ACME_AI_API_KEY"]
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "Acme AI API key",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Acme AI API key"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    Il manifest dichiara `providerAuthEnvVars` in modo che OpenClaw possa rilevare
    le credenziali senza caricare il runtime del tuo plugin. `modelSupport` è facoltativo
    e permette a OpenClaw di caricare automaticamente il tuo plugin provider da model id abbreviati
    come `acme-large` prima che esistano hook di runtime. Se pubblichi il
    provider su ClawHub, quei campi `openclaw.compat` e `openclaw.build`
    sono obbligatori in `package.json`.

  </Step>

  <Step title="Registra il provider">
    Un provider minimo ha bisogno di `id`, `label`, `auth` e `catalog`:

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });
      },
    });
    ```

    Questo è un provider funzionante. Gli utenti ora possono
    `openclaw onboard --acme-ai-api-key <key>` e selezionare
    `acme-ai/acme-large` come modello.

    Per i provider integrati che registrano solo un provider testuale con
    auth tramite API key più un singolo runtime basato su catalogo, preferisci
    l'helper più ristretto
    `defineSingleProviderPluginEntry(...)`:

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI API key",
            hint: "API key from your Acme AI dashboard",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Enter your Acme AI API key",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    Se il tuo flusso di auth deve anche modificare `models.providers.*`, alias e
    il modello predefinito dell'agent durante l'onboarding, usa gli helper preset di
    `openclaw/plugin-sdk/provider-onboard`. Gli helper più ristretti sono
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` e
    `createModelCatalogPresetAppliers(...)`.

    Quando l'endpoint nativo di un provider supporta blocchi di utilizzo in streaming sul
    normale trasporto `openai-completions`, preferisci gli helper di catalogo condivisi in
    `openclaw/plugin-sdk/provider-catalog-shared` invece di hardcodare controlli sugli id dei provider.
    `supportsNativeStreamingUsageCompat(...)` e
    `applyProviderNativeStreamingUsageCompat(...)` rilevano il supporto dalla mappa delle capability dell'endpoint,
    così anche gli endpoint nativi in stile Moonshot/DashScope possono aderire
    anche quando un plugin usa un id provider personalizzato.

  </Step>

  <Step title="Aggiungi la risoluzione dinamica dei modelli">
    Se il tuo provider accetta model ID arbitrari (come un proxy o un router),
    aggiungi `resolveDynamicModel`:

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog sopra

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    Se la risoluzione richiede una chiamata di rete, usa `prepareDynamicModel` per un warm-up
    asincrono: `resolveDynamicModel` viene eseguito di nuovo dopo il completamento.

  </Step>

  <Step title="Aggiungi hook di runtime (se necessario)">
    La maggior parte dei provider richiede solo `catalog` + `resolveDynamicModel`. Aggiungi hook
    in modo incrementale in base alle esigenze del tuo provider.

    I builder helper condivisi ora coprono le famiglie più comuni di replay/tool-compat,
    quindi i plugin di solito non devono collegare manualmente ogni hook uno per uno:

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    Famiglie di replay disponibili oggi:

    | Family | Cosa collega |
    | --- | --- |
    | `openai-compatible` | Criteri di replay condivisi in stile OpenAI per trasporti compatibili con OpenAI, inclusa la sanitizzazione di `tool-call-id`, correzioni dell'ordine assistant-first e validazione generica dei turni Gemini dove il trasporto lo richiede |
    | `anthropic-by-model` | Criteri di replay compatibili con Claude scelti da `modelId`, così i trasporti Anthropic-message ricevono la pulizia dei thinking-block specifica di Claude solo quando il modello risolto è effettivamente un id Claude |
    | `google-gemini` | Criteri di replay Gemini nativi più sanitizzazione del replay bootstrap e modalità tagged reasoning-output |
    | `passthrough-gemini` | Sanitizzazione della thought-signature Gemini per modelli Gemini eseguiti tramite trasporti proxy compatibili con OpenAI; non abilita la validazione nativa del replay Gemini né le riscritture bootstrap |
    | `hybrid-anthropic-openai` | Criteri ibridi per provider che combinano superfici di modello Anthropic-message e compatibili con OpenAI in un unico plugin; l'eliminazione facoltativa dei thinking-block solo-Claude resta limitata al lato Anthropic |

    Esempi reali integrati:

    - `google`: `google-gemini`
    - `openrouter`, `kilocode`, `opencode` e `opencode-go`: `passthrough-gemini`
    - `amazon-bedrock` e `anthropic-vertex`: `anthropic-by-model`
    - `minimax`: `hybrid-anthropic-openai`
    - `moonshot`, `ollama`, `xai` e `zai`: `openai-compatible`

    Famiglie di stream disponibili oggi:

    | Family | Cosa collega |
    | --- | --- |
    | `google-thinking` | Normalizzazione del payload di thinking Gemini sul percorso stream condiviso |
    | `kilocode-thinking` | Wrapper del reasoning Kilo sul percorso stream proxy condiviso, con `kilo/auto` e id reasoning proxy non supportati che saltano il thinking iniettato |
    | `moonshot-thinking` | Mappatura del payload native-thinking binario Moonshot da config + livello `/think` |
    | `minimax-fast-mode` | Riscrittura del modello MiniMax fast-mode sul percorso stream condiviso |
    | `openai-responses-defaults` | Wrapper nativi condivisi OpenAI/Codex Responses: header di attribuzione, `/fast`/`serviceTier`, verbosità del testo, ricerca web nativa Codex, shaping del payload reasoning-compat e gestione del contesto Responses |
    | `openrouter-thinking` | Wrapper reasoning OpenRouter per percorsi proxy, con skip per modelli non supportati/`auto` gestiti centralmente |
    | `tool-stream-default-on` | Wrapper `tool_stream` abilitato di default per provider come Z.AI che vogliono il tool streaming salvo disabilitazione esplicita |

    Esempi reali integrati:

    - `google`: `google-thinking`
    - `kilocode`: `kilocode-thinking`
    - `moonshot`: `moonshot-thinking`
    - `minimax` e `minimax-portal`: `minimax-fast-mode`
    - `openai` e `openai-codex`: `openai-responses-defaults`
    - `openrouter`: `openrouter-thinking`
    - `zai`: `tool-stream-default-on`

    `openclaw/plugin-sdk/provider-model-shared` esporta anche l'enum replay-family
    più gli helper condivisi su cui quelle famiglie sono costruite. Gli export pubblici
    comuni includono:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - builder replay condivisi come `buildOpenAICompatibleReplayPolicy(...)`,
      `buildAnthropicReplayPolicyForModel(...)`,
      `buildGoogleGeminiReplayPolicy(...)` e
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)`
    - helper replay Gemini come `sanitizeGoogleGeminiReplayHistory(...)`
      e `resolveTaggedReasoningOutputMode()`
    - helper endpoint/model come `resolveProviderEndpoint(...)`,
      `normalizeProviderId(...)`, `normalizeGooglePreviewModelId(...)` e
      `normalizeNativeXaiModelId(...)`

    `openclaw/plugin-sdk/provider-stream` espone sia il builder di family sia
    i wrapper helper pubblici riutilizzati da quelle family. Gli export pubblici comuni
    includono:

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - wrapper condivisi OpenAI/Codex come
      `createOpenAIAttributionHeadersWrapper(...)`,
      `createOpenAIFastModeWrapper(...)`,
      `createOpenAIServiceTierWrapper(...)`,
      `createOpenAIResponsesContextManagementWrapper(...)` e
      `createCodexNativeWebSearchWrapper(...)`
    - wrapper condivisi proxy/provider come `createOpenRouterWrapper(...)`,
      `createToolStreamWrapper(...)` e `createMinimaxFastModeWrapper(...)`

    Alcuni helper stream restano volutamente locali al provider. Esempio
    integrato attuale: `@openclaw/anthropic-provider` esporta
    `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` e i
    builder wrapper Anthropic di livello inferiore dalla sua seam pubblica `api.ts` /
    `contract-api.ts`. Quegli helper restano specifici di Anthropic perché
    codificano anche la gestione beta Claude OAuth e il gate `context1m`.

    Anche altri provider integrati mantengono wrapper specifici del trasporto in locale quando
    il comportamento non è condivisibile in modo pulito tra le family. Esempio attuale: il
    plugin xAI integrato mantiene nel proprio
    `wrapStreamFn` lo shaping nativo delle Responses xAI, inclusi rewrite degli alias `/fast`,
    `tool_stream` predefinito, pulizia strict-tool non supportata e rimozione del payload
    reasoning specifica di xAI.

    `openclaw/plugin-sdk/provider-tools` al momento espone una family condivisa di
    tool-schema più helper condivisi per schema/compat:

    - `ProviderToolCompatFamily` documenta oggi l'inventario delle family condivise.
    - `buildProviderToolCompatFamilyHooks("gemini")` collega pulizia +
      diagnostica dello schema Gemini per provider che richiedono tool schema sicuri per Gemini.
    - `normalizeGeminiToolSchemas(...)` e `inspectGeminiToolSchemas(...)`
      sono i sottostanti helper pubblici dello schema Gemini.
    - `resolveXaiModelCompatPatch()` restituisce la patch compat integrata di xAI:
      `toolSchemaProfile: "xai"`, keyword schema non supportate, supporto nativo
      `web_search` e decodifica degli argomenti delle tool-call con entità HTML.
    - `applyXaiModelCompat(model)` applica la stessa patch compat xAI a un
      modello risolto prima che arrivi al runner.

    Esempio reale integrato: il plugin xAI usa `normalizeResolvedModel` più
    `contributeResolvedModelCompat` per mantenere questi metadati compat di proprietà del
    provider invece di hardcodare regole xAI nel core.

    Lo stesso pattern package-root supporta anche altri provider integrati:

    - `@openclaw/openai-provider`: `api.ts` esporta builder provider,
      helper per i modelli predefiniti e builder realtime provider
    - `@openclaw/openrouter-provider`: `api.ts` esporta il builder del provider
      più helper di onboarding/config

    <Tabs>
      <Tab title="Scambio di token">
        Per i provider che richiedono uno scambio di token prima di ogni chiamata di inferenza:

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="Header personalizzati">
        Per i provider che richiedono header di richiesta personalizzati o modifiche al body:

        ```typescript
        // wrapStreamFn restituisce uno StreamFn derivato da ctx.streamFn
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="Identità del trasporto nativo">
        Per i provider che richiedono header o metadati nativi di richiesta/sessione su
        trasporti HTTP o WebSocket generici:

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="Utilizzo e fatturazione">
        Per i provider che espongono dati di utilizzo/fatturazione:

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```
      </Tab>
    </Tabs>

    <Accordion title="Tutti gli hook provider disponibili">
      OpenClaw richiama gli hook in questo ordine. La maggior parte dei provider ne usa solo 2-3:

      | # | Hook | Quando usarlo |
      | --- | --- | --- |
      | 1 | `catalog` | Catalogo modelli o valori predefiniti `baseUrl` |
      | 2 | `applyConfigDefaults` | Valori predefiniti globali di proprietà del provider durante la materializzazione della config |
      | 3 | `normalizeModelId` | Pulizia degli alias legacy/preview dei model-id prima della ricerca |
      | 4 | `normalizeTransport` | Pulizia di `api` / `baseUrl` della family provider prima dell'assemblaggio generico del modello |
      | 5 | `normalizeConfig` | Normalizza la config `models.providers.<id>` |
      | 6 | `applyNativeStreamingUsageCompat` | Riscritture compat per l'utilizzo in native streaming per provider di config |
      | 7 | `resolveConfigApiKey` | Risoluzione auth dei marker env di proprietà del provider |
      | 8 | `resolveSyntheticAuth` | Auth sintetica locale/self-hosted o basata sulla config |
      | 9 | `shouldDeferSyntheticProfileAuth` | Abbassa di priorità i placeholder sintetici dei profili memorizzati rispetto all'auth env/config |
      | 10 | `resolveDynamicModel` | Accetta model ID upstream arbitrari |
      | 11 | `prepareDynamicModel` | Recupero asincrono dei metadati prima della risoluzione |
      | 12 | `normalizeResolvedModel` | Riscritture di trasporto prima del runner |

    Note sul fallback di runtime:

    - `normalizeConfig` controlla prima il provider corrispondente, poi gli altri
      plugin provider con capability di hook finché uno non modifica davvero la config.
      Se nessun hook provider riscrive una voce di config supportata della famiglia Google,
      continua comunque ad applicarsi il normalizzatore di config Google integrato.
    - `resolveConfigApiKey` usa l'hook provider quando esposto. Il percorso integrato
      `amazon-bedrock` ha qui anche un resolver integrato per marker env AWS,
      anche se l'auth di runtime Bedrock continua comunque a usare la catena predefinita AWS SDK.
      | 13 | `contributeResolvedModelCompat` | Flag compat per modelli vendor dietro un altro trasporto compatibile |
      | 14 | `capabilities` | Bag statico legacy delle capability; solo per compatibilità |
      | 15 | `normalizeToolSchemas` | Pulizia dello schema tool di proprietà del provider prima della registrazione |
      | 16 | `inspectToolSchemas` | Diagnostica dello schema tool di proprietà del provider |
      | 17 | `resolveReasoningOutputMode` | Contratto reasoning-output tagged vs nativo |
      | 18 | `prepareExtraParams` | Parametri di richiesta predefiniti |
      | 19 | `createStreamFn` | Trasporto StreamFn completamente personalizzato |
      | 20 | `wrapStreamFn` | Wrapper personalizzati di header/body sul normale percorso stream |
      | 21 | `resolveTransportTurnState` | Header/metadati nativi per turno |
      | 22 | `resolveWebSocketSessionPolicy` | Header di sessione WS nativi/cool-down |
      | 23 | `formatApiKey` | Forma personalizzata del token di runtime |
      | 24 | `refreshOAuth` | Refresh OAuth personalizzato |
      | 25 | `buildAuthDoctorHint` | Guida di riparazione auth |
      | 26 | `matchesContextOverflowError` | Rilevamento overflow di proprietà del provider |
      | 27 | `classifyFailoverReason` | Classificazione di rate-limit/sovraccarico di proprietà del provider |
      | 28 | `isCacheTtlEligible` | Gate TTL della cache prompt |
      | 29 | `buildMissingAuthMessage` | Suggerimento personalizzato per auth mancante |
      | 30 | `suppressBuiltInModel` | Nasconde righe upstream obsolete |
      | 31 | `augmentModelCatalog` | Righe sintetiche di forward-compat |
      | 32 | `isBinaryThinking` | Thinking binario on/off |
      | 33 | `supportsXHighThinking` | Supporto del reasoning `xhigh` |
      | 34 | `resolveDefaultThinkingLevel` | Criteri predefiniti `/think` |
      | 35 | `isModernModelRef` | Matching del modello live/smoke |
      | 36 | `prepareRuntimeAuth` | Scambio di token prima dell'inferenza |
      | 37 | `resolveUsageAuth` | Parsing personalizzato delle credenziali di utilizzo |
      | 38 | `fetchUsageSnapshot` | Endpoint di utilizzo personalizzato |
      | 39 | `createEmbeddingProvider` | Adapter embedding di proprietà del provider per memory/search |
      | 40 | `buildReplayPolicy` | Criteri personalizzati di replay/compattazione della trascrizione |
      | 41 | `sanitizeReplayHistory` | Riscritture replay specifiche del provider dopo la pulizia generica |
      | 42 | `validateReplayTurns` | Validazione rigorosa dei turni replay prima dell'embedded runner |
      | 43 | `onModelSelected` | Callback post-selezione (ad esempio telemetria) |

      Nota sulla regolazione dei prompt:

      - `resolveSystemPromptContribution` consente a un provider di iniettare
        linee guida del system prompt sensibili alla cache per una family di modelli. Preferiscilo a
        `before_prompt_build` quando il comportamento appartiene a una provider/model
        family e deve preservare la divisione stable/dynamic della cache.

      Per descrizioni dettagliate ed esempi reali, vedi
      [Internals: Provider Runtime Hooks](/it/plugins/architecture#provider-runtime-hooks).
    </Accordion>

  </Step>

  <Step title="Aggiungi capability extra (facoltativo)">
    <a id="step-5-add-extra-capabilities"></a>
    Un plugin provider può registrare speech, trascrizione realtime, voce realtime,
    media understanding, image generation, video generation, web fetch
    e web search insieme all'inferenza testuale:

    ```typescript
    register(api) {
      api.registerProvider({ id: "acme-ai", /* ... */ });

      api.registerSpeechProvider({
        id: "acme-ai",
        label: "Acme Speech",
        isConfigured: ({ config }) => Boolean(config.messages?.tts),
        synthesize: async (req) => ({
          audioBuffer: Buffer.from(/* PCM data */),
          outputFormat: "mp3",
          fileExtension: ".mp3",
          voiceCompatible: false,
        }),
      });

      api.registerRealtimeTranscriptionProvider({
        id: "acme-ai",
        label: "Acme Realtime Transcription",
        isConfigured: () => true,
        createSession: (req) => ({
          connect: async () => {},
          sendAudio: () => {},
          close: () => {},
          isConnected: () => true,
        }),
      });

      api.registerRealtimeVoiceProvider({
        id: "acme-ai",
        label: "Acme Realtime Voice",
        isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
        createBridge: (req) => ({
          connect: async () => {},
          sendAudio: () => {},
          setMediaTimestamp: () => {},
          submitToolResult: () => {},
          acknowledgeMark: () => {},
          close: () => {},
          isConnected: () => true,
        }),
      });

      api.registerMediaUnderstandingProvider({
        id: "acme-ai",
        capabilities: ["image", "audio"],
        describeImage: async (req) => ({ text: "A photo of..." }),
        transcribeAudio: async (req) => ({ text: "Transcript..." }),
      });

      api.registerImageGenerationProvider({
        id: "acme-ai",
        label: "Acme Images",
        generate: async (req) => ({ /* image result */ }),
      });

      api.registerVideoGenerationProvider({
        id: "acme-ai",
        label: "Acme Video",
        capabilities: {
          maxVideos: 1,
          maxDurationSeconds: 10,
          supportsResolution: true,
        },
        generateVideo: async (req) => ({ videos: [] }),
      });

      api.registerWebFetchProvider({
        id: "acme-ai-fetch",
        label: "Acme Fetch",
        hint: "Fetch pages through Acme's rendering backend.",
        envVars: ["ACME_FETCH_API_KEY"],
        placeholder: "acme-...",
        signupUrl: "https://acme.example.com/fetch",
        credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
        getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
        setCredentialValue: (fetchConfigTarget, value) => {
          const acme = (fetchConfigTarget.acme ??= {});
          acme.apiKey = value;
        },
        createTool: () => ({
          description: "Fetch a page through Acme Fetch.",
          parameters: {},
          execute: async (args) => ({ content: [] }),
        }),
      });

      api.registerWebSearchProvider({
        id: "acme-ai-search",
        label: "Acme Search",
        search: async (req) => ({ content: [] }),
      });
    }
    ```

    OpenClaw lo classifica come plugin **hybrid-capability**. Questo è il
    pattern consigliato per i plugin aziendali (un plugin per vendor). Vedi
    [Internals: Capability Ownership](/it/plugins/architecture#capability-ownership-model).

  </Step>

  <Step title="Test">
    <a id="step-6-test"></a>
    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // Export your provider config object from index.ts or a dedicated file
    import { acmeProvider } from "./provider.js";

    describe("acme-ai provider", () => {
      it("resolves dynamic models", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("returns catalog when key is available", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("returns null catalog when no key", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## Pubblica su ClawHub

I plugin provider vengono pubblicati allo stesso modo di qualsiasi altro plugin di codice esterno:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

Non usare qui il legacy alias di pubblicazione solo-Skills; i package plugin devono usare
`clawhub package publish`.

## Struttura dei file

```
<bundled-plugin-root>/acme-ai/
├── package.json              # metadata openclaw.providers
├── openclaw.plugin.json      # Manifest con providerAuthEnvVars
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Test
    └── usage.ts              # Endpoint di utilizzo (facoltativo)
```

## Riferimento ordine del catalogo

`catalog.order` controlla quando il tuo catalogo viene unito rispetto ai
provider integrati:

| Order     | Quando        | Caso d'uso                                     |
| --------- | ------------- | ---------------------------------------------- |
| `simple`  | Primo passaggio | Provider semplici con API key                |
| `profile` | Dopo simple   | Provider protetti da auth profile              |
| `paired`  | Dopo profile  | Sintetizza più voci correlate                  |
| `late`    | Ultimo passaggio | Sovrascrive i provider esistenti (vince in caso di collisione) |

## Passaggi successivi

- [Plugin channel](/it/plugins/sdk-channel-plugins) — se il tuo plugin fornisce anche un channel
- [SDK Runtime](/it/plugins/sdk-runtime) — helper `api.runtime` (TTS, search, subagent)
- [Panoramica SDK](/it/plugins/sdk-overview) — riferimento completo agli import dei sottopercorsi
- [Internals dei plugin](/it/plugins/architecture#provider-runtime-hooks) — dettagli degli hook ed esempi integrati
