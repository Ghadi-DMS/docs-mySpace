---
read_when:
    - Tworzysz nowy plugin dostawcy modeli
    - Chcesz dodać do OpenClaw proxy zgodne z OpenAI lub własny LLM
    - Musisz zrozumieć uwierzytelnianie dostawców, katalogi i hooki runtime
sidebarTitle: Provider Plugins
summary: Przewodnik krok po kroku po tworzeniu pluginu dostawcy modeli dla OpenClaw
title: Tworzenie pluginów dostawców
x-i18n:
    generated_at: "2026-04-09T01:30:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: 38d9af522dc19e49c81203a83a4096f01c2398b1df771c848a30ad98f251e9e1
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# Tworzenie pluginów dostawców

Ten przewodnik prowadzi przez tworzenie pluginu dostawcy, który dodaje dostawcę modeli
(LLM) do OpenClaw. Na końcu będziesz mieć dostawcę z katalogiem modeli,
uwierzytelnianiem kluczem API i dynamicznym rozwiązywaniem modeli.

<Info>
  Jeśli nie tworzyłeś wcześniej żadnego pluginu OpenClaw, najpierw przeczytaj
  [Getting Started](/pl/plugins/building-plugins), aby poznać podstawową strukturę
  pakietu i konfigurację manifestu.
</Info>

## Przewodnik

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Pakiet i manifest">
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
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
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

    Manifest deklaruje `providerAuthEnvVars`, aby OpenClaw mógł wykrywać
    poświadczenia bez ładowania runtime Twojego pluginu. Dodaj `providerAuthAliases`,
    gdy wariant dostawcy ma ponownie używać uwierzytelniania innego identyfikatora dostawcy. `modelSupport`
    jest opcjonalne i pozwala OpenClaw automatycznie ładować plugin dostawcy na podstawie skróconych
    identyfikatorów modeli, takich jak `acme-large`, zanim hooki runtime zaczną istnieć. Jeśli publikujesz
    dostawcę w ClawHub, pola `openclaw.compat` i `openclaw.build`
    są wymagane w `package.json`.

  </Step>

  <Step title="Zarejestruj dostawcę">
    Minimalny dostawca potrzebuje `id`, `label`, `auth` i `catalog`:

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

    To jest działający dostawca. Użytkownicy mogą teraz
    użyć `openclaw onboard --acme-ai-api-key <key>` i wybrać
    `acme-ai/acme-large` jako swój model.

    W przypadku bundlowanych dostawców, którzy rejestrują tylko jednego dostawcę tekstowego z
    uwierzytelnianiem kluczem API oraz pojedynczym runtime opartym na katalogu, wybierz raczej
    węższy helper `defineSingleProviderPluginEntry(...)`:

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

    Jeśli przepływ uwierzytelniania wymaga także modyfikacji `models.providers.*`, aliasów oraz
    domyślnego modelu agenta podczas onboardingu, użyj preset helperów z
    `openclaw/plugin-sdk/provider-onboard`. Najwęższe helpery to
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` oraz
    `createModelCatalogPresetAppliers(...)`.

    Gdy natywny endpoint dostawcy obsługuje strumieniowane bloki użycia na
    zwykłym transporcie `openai-completions`, preferuj współdzielone helpery katalogu z
    `openclaw/plugin-sdk/provider-catalog-shared` zamiast wpisywania na sztywno kontroli identyfikatora
    dostawcy. `supportsNativeStreamingUsageCompat(...)` i
    `applyProviderNativeStreamingUsageCompat(...)` wykrywają obsługę na podstawie mapy możliwości endpointu,
    dzięki czemu natywne endpointy w stylu Moonshot/DashScope nadal się kwalifikują, nawet jeśli
    plugin używa niestandardowego identyfikatora dostawcy.

  </Step>

  <Step title="Dodaj dynamiczne rozwiązywanie modeli">
    Jeśli dostawca akceptuje dowolne identyfikatory modeli (jak proxy lub router),
    dodaj `resolveDynamicModel`:

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

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

    Jeśli rozwiązywanie wymaga wywołania sieciowego, użyj `prepareDynamicModel` do asynchronicznego
    warm-upu — `resolveDynamicModel` uruchomi się ponownie po jego zakończeniu.

  </Step>

  <Step title="Dodaj hooki runtime (w razie potrzeby)">
    Większość dostawców potrzebuje tylko `catalog` + `resolveDynamicModel`. Dodawaj hooki
    stopniowo, w miarę jak dostawca ich wymaga.

    Współdzielone buildery helperów obejmują teraz najczęstsze rodziny zgodności replay/tool,
    więc pluginy zwykle nie muszą ręcznie podłączać każdego hooka osobno:

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

    Dostępne dziś rodziny replay:

    | Family | Co podłącza |
    | --- | --- |
    | `openai-compatible` | Współdzielona polityka replay w stylu OpenAI dla transportów zgodnych z OpenAI, w tym sanityzacja `tool-call-id`, poprawki kolejności assistant-first oraz ogólna walidacja tur Gemini tam, gdzie transport tego wymaga |
    | `anthropic-by-model` | Polityka replay świadoma Claude, wybierana według `modelId`, dzięki czemu transporty wiadomości Anthropic dostają czyszczenie bloków thinking specyficzne dla Claude tylko wtedy, gdy rozwiązany model rzeczywiście ma identyfikator Claude |
    | `google-gemini` | Natywna polityka replay Gemini oraz sanityzacja replay bootstrap i tryb oznaczonego wyjścia reasoning |
    | `passthrough-gemini` | Sanityzacja sygnatur myśli Gemini dla modeli Gemini uruchamianych przez transporty proxy zgodne z OpenAI; nie włącza natywnej walidacji replay Gemini ani przepisania bootstrap |
    | `hybrid-anthropic-openai` | Hybrydowa polityka dla dostawców, którzy łączą powierzchnie modeli wiadomości Anthropic i kompatybilnych z OpenAI w jednym pluginie; opcjonalne usuwanie bloków thinking tylko dla Claude pozostaje ograniczone do strony Anthropic |

    Rzeczywiste przykłady bundlowane:

    - `google` i `google-gemini-cli`: `google-gemini`
    - `openrouter`, `kilocode`, `opencode` i `opencode-go`: `passthrough-gemini`
    - `amazon-bedrock` i `anthropic-vertex`: `anthropic-by-model`
    - `minimax`: `hybrid-anthropic-openai`
    - `moonshot`, `ollama`, `xai` i `zai`: `openai-compatible`

    Dostępne dziś rodziny stream:

    | Family | Co podłącza |
    | --- | --- |
    | `google-thinking` | Normalizacja payload thinking Gemini na współdzielonej ścieżce stream |
    | `kilocode-thinking` | Opakowanie reasoning Kilo na współdzielonej ścieżce stream proxy, z pomijaniem wstrzykiwanego thinking dla `kilo/auto` i nieobsługiwanych identyfikatorów reasoning proxy |
    | `moonshot-thinking` | Mapowanie natywnego binarnego payload thinking Moonshot z konfiguracji + poziomu `/think` |
    | `minimax-fast-mode` | Przepisanie modelu MiniMax fast-mode na współdzielonej ścieżce stream |
    | `openai-responses-defaults` | Współdzielone natywne opakowania OpenAI/Codex Responses: nagłówki atrybucji, `/fast`/`serviceTier`, szczegółowość tekstu, natywne wyszukiwanie w sieci Codex, kształtowanie payload zgodności reasoning oraz zarządzanie kontekstem Responses |
    | `openrouter-thinking` | Opakowanie reasoning OpenRouter dla tras proxy, z centralnie obsługiwanym pomijaniem modeli nieobsługiwanych/`auto` |
    | `tool-stream-default-on` | Domyślnie włączone opakowanie `tool_stream` dla dostawców takich jak Z.AI, którzy chcą strumieniowania narzędzi, chyba że zostanie to jawnie wyłączone |

    Rzeczywiste przykłady bundlowane:

    - `google` i `google-gemini-cli`: `google-thinking`
    - `kilocode`: `kilocode-thinking`
    - `moonshot`: `moonshot-thinking`
    - `minimax` i `minimax-portal`: `minimax-fast-mode`
    - `openai` i `openai-codex`: `openai-responses-defaults`
    - `openrouter`: `openrouter-thinking`
    - `zai`: `tool-stream-default-on`

    `openclaw/plugin-sdk/provider-model-shared` eksportuje także enum rodziny replay
    oraz współdzielone helpery, z których te rodziny są zbudowane. Typowe publiczne
    eksporty obejmują:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - współdzielone buildery replay, takie jak `buildOpenAICompatibleReplayPolicy(...)`,
      `buildAnthropicReplayPolicyForModel(...)`,
      `buildGoogleGeminiReplayPolicy(...)` oraz
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)`
    - helpery replay Gemini, takie jak `sanitizeGoogleGeminiReplayHistory(...)`
      i `resolveTaggedReasoningOutputMode()`
    - helpery endpointów/modeli, takie jak `resolveProviderEndpoint(...)`,
      `normalizeProviderId(...)`, `normalizeGooglePreviewModelId(...)` oraz
      `normalizeNativeXaiModelId(...)`

    `openclaw/plugin-sdk/provider-stream` udostępnia zarówno builder rodziny, jak i
    publiczne helpery wrapperów ponownie używane przez te rodziny. Typowe publiczne eksporty
    obejmują:

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - współdzielone wrappery OpenAI/Codex, takie jak
      `createOpenAIAttributionHeadersWrapper(...)`,
      `createOpenAIFastModeWrapper(...)`,
      `createOpenAIServiceTierWrapper(...)`,
      `createOpenAIResponsesContextManagementWrapper(...)` oraz
      `createCodexNativeWebSearchWrapper(...)`
    - współdzielone wrappery proxy/dostawców, takie jak `createOpenRouterWrapper(...)`,
      `createToolStreamWrapper(...)` i `createMinimaxFastModeWrapper(...)`

    Niektóre helpery stream celowo pozostają lokalne dla dostawcy. Obecny
    bundlowany przykład: `@openclaw/anthropic-provider` eksportuje
    `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` oraz
    wrappery Anthropic niższego poziomu przez swój publiczny punkt styku `api.ts` /
    `contract-api.ts`. Te helpery pozostają specyficzne dla Anthropic, ponieważ
    kodują także obsługę beta Claude OAuth i bramkowanie `context1m`.

    Inni bundlowani dostawcy także zachowują wrappery specyficzne dla transportu lokalnie, gdy
    zachowanie nie jest współdzielone w czysty sposób między rodzinami. Obecny przykład: bundlowany
    plugin xAI zachowuje natywne kształtowanie xAI Responses we własnym
    `wrapStreamFn`, w tym przepisania aliasów `/fast`, domyślne `tool_stream`,
    czyszczenie nieobsługiwanych strict-tool oraz usuwanie payload reasoning
    specyficznego dla xAI.

    `openclaw/plugin-sdk/provider-tools` udostępnia obecnie jedną współdzieloną
    rodzinę schematu narzędzi oraz współdzielone helpery schematów/zgodności:

    - `ProviderToolCompatFamily` dokumentuje dzisiejszy współdzielony zbiór rodzin.
    - `buildProviderToolCompatFamilyHooks("gemini")` podłącza czyszczenie schematów Gemini
      + diagnostykę dla dostawców, którzy potrzebują schematów narzędzi bezpiecznych dla Gemini.
    - `normalizeGeminiToolSchemas(...)` i `inspectGeminiToolSchemas(...)`
      to bazowe publiczne helpery schematu Gemini.
    - `resolveXaiModelCompatPatch()` zwraca bundlowany patch zgodności xAI:
      `toolSchemaProfile: "xai"`, nieobsługiwane słowa kluczowe schematu, natywne
      wsparcie `web_search` oraz dekodowanie argumentów wywołań narzędzi z encjami HTML.
    - `applyXaiModelCompat(model)` stosuje ten sam patch zgodności xAI do
      rozwiązanego modelu, zanim trafi on do runnera.

    Rzeczywisty bundlowany przykład: plugin xAI używa `normalizeResolvedModel` oraz
    `contributeResolvedModelCompat`, aby zachować metadane zgodności po stronie
    dostawcy zamiast wpisywać reguły xAI na sztywno w core.

    Ten sam wzorzec katalogu głównego pakietu stanowi też podstawę dla innych bundlowanych dostawców:

    - `@openclaw/openai-provider`: `api.ts` eksportuje buildery dostawców,
      helpery modeli domyślnych oraz buildery dostawców realtime
    - `@openclaw/openrouter-provider`: `api.ts` eksportuje builder dostawcy
      oraz helpery onboardingu/konfiguracji

    <Tabs>
      <Tab title="Wymiana tokena">
        Dla dostawców, którzy wymagają wymiany tokena przed każdym wywołaniem inferencji:

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
      <Tab title="Niestandardowe nagłówki">
        Dla dostawców, którzy wymagają niestandardowych nagłówków żądań lub modyfikacji treści żądania:

        ```typescript
        // wrapStreamFn returns a StreamFn derived from ctx.streamFn
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
      <Tab title="Tożsamość natywnego transportu">
        Dla dostawców, którzy potrzebują natywnych nagłówków lub metadanych żądania/sesji w
        ogólnych transportach HTTP lub WebSocket:

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
      <Tab title="Użycie i rozliczenia">
        Dla dostawców, którzy udostępniają dane o użyciu/rozliczeniach:

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

    <Accordion title="Wszystkie dostępne hooki dostawcy">
      OpenClaw wywołuje hooki w tej kolejności. Większość dostawców używa tylko 2-3:

      | # | Hook | Kiedy używać |
      | --- | --- | --- |
      | 1 | `catalog` | Katalog modeli lub domyślne wartości `baseUrl` |
      | 2 | `applyConfigDefaults` | Globalne wartości domyślne należące do dostawcy podczas materializacji konfiguracji |
      | 3 | `normalizeModelId` | Czyszczenie aliasów starszych/podglądowych identyfikatorów modeli przed wyszukiwaniem |
      | 4 | `normalizeTransport` | Czyszczenie `api` / `baseUrl` dla rodziny dostawców przed ogólnym składaniem modelu |
      | 5 | `normalizeConfig` | Normalizacja konfiguracji `models.providers.<id>` |
      | 6 | `applyNativeStreamingUsageCompat` | Przepisania zgodności natywnego strumieniowanego użycia dla dostawców konfiguracyjnych |
      | 7 | `resolveConfigApiKey` | Rozwiązywanie uwierzytelniania markerów środowiska należące do dostawcy |
      | 8 | `resolveSyntheticAuth` | Syntetyczne uwierzytelnianie lokalne/self-hosted lub oparte na konfiguracji |
      | 9 | `shouldDeferSyntheticProfileAuth` | Obniżanie priorytetu syntetycznych placeholderów profili zapisanych względem uwierzytelniania env/config |
      | 10 | `resolveDynamicModel` | Akceptowanie dowolnych nadrzędnych identyfikatorów modeli |
      | 11 | `prepareDynamicModel` | Asynchroniczne pobieranie metadanych przed rozwiązaniem |
      | 12 | `normalizeResolvedModel` | Przepisania transportu przed runnerem |

      Uwagi o fallbacku runtime:

      - `normalizeConfig` najpierw sprawdza dopasowanego dostawcę, a potem innych
        dostawców z hookami, aż któryś faktycznie zmieni konfigurację.
        Jeśli żaden hook dostawcy nie przepisze obsługiwanego wpisu konfiguracji rodziny Google,
        nadal zastosowany zostanie bundlowany normalizator konfiguracji Google.
      - `resolveConfigApiKey` używa hooka dostawcy, gdy jest udostępniony. Bundlowana
        ścieżka `amazon-bedrock` ma tutaj również wbudowany resolver markerów środowiska AWS,
        mimo że samo uwierzytelnianie runtime Bedrock nadal używa domyślnego
        łańcucha AWS SDK.
      | 13 | `contributeResolvedModelCompat` | Flagi zgodności dla modeli dostawców działających przez inny zgodny transport |
      | 14 | `capabilities` | Starszy statyczny zbiór możliwości; tylko dla zgodności |
      | 15 | `normalizeToolSchemas` | Czyszczenie schematów narzędzi należące do dostawcy przed rejestracją |
      | 16 | `inspectToolSchemas` | Diagnostyka schematów narzędzi należąca do dostawcy |
      | 17 | `resolveReasoningOutputMode` | Kontrakt oznaczonego vs natywnego wyjścia reasoning |
      | 18 | `prepareExtraParams` | Domyślne parametry żądania |
      | 19 | `createStreamFn` | W pełni niestandardowy transport StreamFn |
      | 20 | `wrapStreamFn` | Wrappery niestandardowych nagłówków/treści na zwykłej ścieżce stream |
      | 21 | `resolveTransportTurnState` | Natywne nagłówki/metadane per turn |
      | 22 | `resolveWebSocketSessionPolicy` | Natywne nagłówki sesji WS / cooldown |
      | 23 | `formatApiKey` | Niestandardowy kształt tokena runtime |
      | 24 | `refreshOAuth` | Niestandardowe odświeżanie OAuth |
      | 25 | `buildAuthDoctorHint` | Wskazówki naprawy uwierzytelniania |
      | 26 | `matchesContextOverflowError` | Wykrywanie overflow należące do dostawcy |
      | 27 | `classifyFailoverReason` | Klasyfikacja rate-limit/overload należąca do dostawcy |
      | 28 | `isCacheTtlEligible` | Bramka TTL dla cache promptów |
      | 29 | `buildMissingAuthMessage` | Niestandardowa wskazówka brakującego uwierzytelniania |
      | 30 | `suppressBuiltInModel` | Ukrywanie nieaktualnych wierszy upstream |
      | 31 | `augmentModelCatalog` | Syntetyczne wiersze forward-compat |
      | 32 | `isBinaryThinking` | Binarne włączanie/wyłączanie thinking |
      | 33 | `supportsXHighThinking` | Obsługa reasoning `xhigh` |
      | 34 | `resolveDefaultThinkingLevel` | Domyślna polityka `/think` |
      | 35 | `isModernModelRef` | Dopasowanie modeli live/smoke |
      | 36 | `prepareRuntimeAuth` | Wymiana tokena przed inferencją |
      | 37 | `resolveUsageAuth` | Niestandardowe parsowanie poświadczeń użycia |
      | 38 | `fetchUsageSnapshot` | Niestandardowy endpoint użycia |
      | 39 | `createEmbeddingProvider` | Adapter embeddingów należący do dostawcy dla memory/search |
      | 40 | `buildReplayPolicy` | Niestandardowa polityka replay/kompaktowania transkryptu |
      | 41 | `sanitizeReplayHistory` | Przepisania replay specyficzne dla dostawcy po ogólnym czyszczeniu |
      | 42 | `validateReplayTurns` | Ścisła walidacja tur replay przed osadzonym runnerem |
      | 43 | `onModelSelected` | Callback po wyborze modelu (np. telemetry) |

      Uwaga o dostrajaniu promptów:

      - `resolveSystemPromptContribution` pozwala dostawcy wstrzykiwać wskazówki
        do system promptu świadome cache dla rodziny modeli. Preferuj je zamiast
        `before_prompt_build`, gdy zachowanie należy do jednej rodziny dostawcy/modeli
        i powinno zachować stabilny/dynamiczny podział cache.

      Szczegółowe opisy i przykłady z rzeczywistego użycia znajdziesz w
      [Internals: Provider Runtime Hooks](/pl/plugins/architecture#provider-runtime-hooks).
    </Accordion>

  </Step>

  <Step title="Dodaj dodatkowe możliwości (opcjonalnie)">
    <a id="step-5-add-extra-capabilities"></a>
    Plugin dostawcy może rejestrować dostawców speech, realtime transcription, realtime
    voice, media understanding, image generation, video generation, web fetch
    i web search równolegle z inferencją tekstową:

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
          generate: {
            maxVideos: 1,
            maxDurationSeconds: 10,
            supportsResolution: true,
          },
          imageToVideo: {
            enabled: true,
            maxVideos: 1,
            maxInputImages: 1,
            maxDurationSeconds: 5,
          },
          videoToVideo: {
            enabled: false,
          },
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

    OpenClaw klasyfikuje to jako plugin **hybrid-capability**. To
    zalecany wzorzec dla pluginów firmowych (jeden plugin na dostawcę). Zobacz
    [Internals: Capability Ownership](/pl/plugins/architecture#capability-ownership-model).

    W przypadku generowania wideo preferuj pokazany powyżej kształt możliwości świadomy trybu:
    `generate`, `imageToVideo` i `videoToVideo`. Płaskie pola zbiorcze, takie
    jak `maxInputImages`, `maxInputVideos` i `maxDurationSeconds`, nie
    wystarczają, aby w czytelny sposób reklamować obsługę trybów transformacji lub tryby wyłączone.

    Dostawcy generowania muzyki powinni stosować ten sam wzorzec:
    `generate` dla generowania tylko z promptu oraz `edit` dla generowania
    opartego na obrazie referencyjnym. Płaskie pola zbiorcze, takie jak `maxInputImages`,
    `supportsLyrics` i `supportsFormat`, nie wystarczają do reklamowania obsługi edycji;
    oczekiwanym kontraktem są jawne bloki `generate` / `edit`.

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

## Publikowanie w ClawHub

Pluginy dostawców publikuje się tak samo jak każdy inny zewnętrzny plugin kodu:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

Nie używaj tutaj starszego aliasu publikowania tylko dla Skills; pakiety pluginów powinny używać
`clawhub package publish`.

## Struktura plików

```
<bundled-plugin-root>/acme-ai/
├── package.json              # metadane openclaw.providers
├── openclaw.plugin.json      # Manifest z metadanymi uwierzytelniania dostawcy
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Testy
    └── usage.ts              # Endpoint użycia (opcjonalnie)
```

## Informacje o kolejności katalogu

`catalog.order` kontroluje, kiedy katalog jest scalany względem wbudowanych
dostawców:

| Order     | Kiedy        | Przypadek użycia                                |
| --------- | ------------ | ----------------------------------------------- |
| `simple`  | Pierwsze przejście | Zwykli dostawcy z kluczem API                  |
| `profile` | Po `simple`  | Dostawcy zależni od profili uwierzytelniania    |
| `paired`  | Po `profile` | Syntezowanie wielu powiązanych wpisów           |
| `late`    | Ostatnie przejście | Nadpisywanie istniejących dostawców (wygrywa przy kolizji) |

## Kolejne kroki

- [Channel Plugins](/pl/plugins/sdk-channel-plugins) — jeśli plugin udostępnia także kanał
- [SDK Runtime](/pl/plugins/sdk-runtime) — helpery `api.runtime` (TTS, search, subagent)
- [SDK Overview](/pl/plugins/sdk-overview) — pełne odwołanie do importów podścieżek
- [Plugin Internals](/pl/plugins/architecture#provider-runtime-hooks) — szczegóły hooków i bundlowane przykłady
