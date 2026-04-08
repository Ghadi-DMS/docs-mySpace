---
read_when:
    - Ви створюєте новий плагін постачальника моделей
    - Ви хочете додати до OpenClaw OpenAI-сумісний проксі або власну LLM
    - Вам потрібно зрозуміти автентифікацію постачальників, каталоги та runtime hooks
sidebarTitle: Provider Plugins
summary: Покроковий посібник зі створення плагіна постачальника моделей для OpenClaw
title: Створення плагінів постачальників
x-i18n:
    generated_at: "2026-04-08T18:09:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: 38d9af522dc19e49c81203a83a4096f01c2398b1df771c848a30ad98f251e9e1
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# Створення плагінів постачальників

Цей посібник покроково пояснює, як створити плагін постачальника, який додає
до OpenClaw постачальника моделей (LLM). Наприкінці у вас буде постачальник із каталогом моделей,
автентифікацією через API key та динамічним визначенням моделей.

<Info>
  Якщо ви раніше не створювали жодного плагіна OpenClaw, спочатку прочитайте
  [Getting Started](/uk/plugins/building-plugins), щоб ознайомитися з базовою
  структурою пакета та налаштуванням маніфесту.
</Info>

## Покроковий приклад

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Пакет і маніфест">
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

    Маніфест оголошує `providerAuthEnvVars`, щоб OpenClaw міг виявляти
    облікові дані без завантаження runtime вашого плагіна. Додайте `providerAuthAliases`,
    якщо варіант постачальника має повторно використовувати автентифікацію іншого ідентифікатора постачальника. `modelSupport`
    необов’язковий і дозволяє OpenClaw автоматично завантажувати ваш плагін постачальника зі скорочених
    ідентифікаторів моделей, таких як `acme-large`, ще до появи runtime hooks. Якщо ви публікуєте
    постачальника у ClawHub, ці поля `openclaw.compat` і `openclaw.build`
    обов’язкові в `package.json`.

  </Step>

  <Step title="Зареєструйте постачальника">
    Мінімальному постачальнику потрібні `id`, `label`, `auth` і `catalog`:

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

    Це вже робочий постачальник. Тепер користувачі можуть виконати
    `openclaw onboard --acme-ai-api-key <key>` і вибрати
    `acme-ai/acme-large` як свою модель.

    Для вбудованих постачальників, які реєструють лише одного текстового постачальника з автентифікацією через API key
    плюс один runtime на основі каталогу, віддавайте перевагу вужчому
    допоміжному засобу `defineSingleProviderPluginEntry(...)`:

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

    Якщо ваш потік автентифікації також має оновлювати `models.providers.*`, aliases і
    модель агента за замовчуванням під час онбордингу, використовуйте готові helper-функції з
    `openclaw/plugin-sdk/provider-onboard`. Найвужчі helper-функції:
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` і
    `createModelCatalogPresetAppliers(...)`.

    Якщо власний endpoint постачальника підтримує потокові блоки використання на
    звичайному транспорті `openai-completions`, віддавайте перевагу спільним helper-функціям каталогу з
    `openclaw/plugin-sdk/provider-catalog-shared` замість жорсткого кодування перевірок
    ідентифікатора постачальника. `supportsNativeStreamingUsageCompat(...)` і
    `applyProviderNativeStreamingUsageCompat(...)` визначають підтримку з карти можливостей endpoint, тож власні endpoint Moonshot/DashScope-типу
    також можуть бути увімкнені, навіть якщо плагін використовує власний ідентифікатор постачальника.

  </Step>

  <Step title="Додайте динамічне визначення моделей">
    Якщо ваш постачальник приймає довільні ідентифікатори моделей (як проксі чи маршрутизатор),
    додайте `resolveDynamicModel`:

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

    Якщо для визначення потрібен мережевий виклик, використовуйте `prepareDynamicModel` для асинхронного
    прогріву — після завершення `resolveDynamicModel` буде запущено знову.

  </Step>

  <Step title="Додайте runtime hooks (за потреби)">
    Більшості постачальників потрібні лише `catalog` + `resolveDynamicModel`. Додавайте hooks
    поступово, коли вони стають потрібними вашому постачальнику.

    Спільні builder-функції helper-ів тепер покривають найпоширеніші сімейства
    replay/tool-compat, тому плагінам зазвичай не потрібно вручну підключати кожен hook окремо:

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

    Доступні сьогодні сімейства replay:

    | Family | Що воно підключає |
    | --- | --- |
    | `openai-compatible` | Спільна політика replay у стилі OpenAI для OpenAI-сумісних транспортів, зокрема очищення tool-call-id, виправлення порядку assistant-first і загальна валідація Gemini-turn там, де цього потребує транспорт |
    | `anthropic-by-model` | Політика replay з урахуванням Claude, що вибирається за `modelId`, тому транспорти Anthropic-message отримують очищення thinking-block, специфічне для Claude, лише коли визначена модель справді є ідентифікатором Claude |
    | `google-gemini` | Власна політика replay Gemini плюс очищення bootstrap replay і режим tagged reasoning-output |
    | `passthrough-gemini` | Очищення thought-signature Gemini для моделей Gemini, що працюють через OpenAI-сумісні проксі-транспорти; не вмикає власну валідацію replay Gemini або переписування bootstrap |
    | `hybrid-anthropic-openai` | Гібридна політика для постачальників, які поєднують поверхні моделей Anthropic-message і OpenAI-compatible в одному плагіні; необов’язкове видалення thinking-block лише для Claude залишається обмеженим стороною Anthropic |

    Реальні вбудовані приклади:

    - `google` і `google-gemini-cli`: `google-gemini`
    - `openrouter`, `kilocode`, `opencode` і `opencode-go`: `passthrough-gemini`
    - `amazon-bedrock` і `anthropic-vertex`: `anthropic-by-model`
    - `minimax`: `hybrid-anthropic-openai`
    - `moonshot`, `ollama`, `xai` і `zai`: `openai-compatible`

    Доступні сьогодні сімейства stream:

    | Family | Що воно підключає |
    | --- | --- |
    | `google-thinking` | Нормалізація payload thinking Gemini на спільному шляху stream |
    | `kilocode-thinking` | Обгортка міркування Kilo на спільному шляху проксі-stream, де `kilo/auto` та непідтримувані ідентифікатори міркування проксі пропускають інжектований thinking |
    | `moonshot-thinking` | Відображення бінарного payload native-thinking Moonshot із config + рівня `/think` |
    | `minimax-fast-mode` | Переписування моделі MiniMax fast-mode на спільному шляху stream |
    | `openai-responses-defaults` | Спільні власні обгортки OpenAI/Codex Responses: заголовки attribution, `/fast`/`serviceTier`, verbosity тексту, власний web search Codex, формування payload для сумісності reasoning і керування контекстом Responses |
    | `openrouter-thinking` | Обгортка reasoning OpenRouter для маршрутів проксі, з централізованою обробкою пропусків для непідтримуваних моделей і `auto` |
    | `tool-stream-default-on` | Обгортка `tool_stream`, увімкнена за замовчуванням, для таких постачальників, як Z.AI, які хочуть потокову передачу інструментів, якщо її явно не вимкнено |

    Реальні вбудовані приклади:

    - `google` і `google-gemini-cli`: `google-thinking`
    - `kilocode`: `kilocode-thinking`
    - `moonshot`: `moonshot-thinking`
    - `minimax` і `minimax-portal`: `minimax-fast-mode`
    - `openai` і `openai-codex`: `openai-responses-defaults`
    - `openrouter`: `openrouter-thinking`
    - `zai`: `tool-stream-default-on`

    `openclaw/plugin-sdk/provider-model-shared` також експортує enum сімейств replay
    разом зі спільними helper-функціями, на основі яких побудовано ці сімейства. Поширені публічні експорти
    включають:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - спільні builder-функції replay, як-от `buildOpenAICompatibleReplayPolicy(...)`,
      `buildAnthropicReplayPolicyForModel(...)`,
      `buildGoogleGeminiReplayPolicy(...)` і
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)`
    - helper-функції replay Gemini, як-от `sanitizeGoogleGeminiReplayHistory(...)`
      і `resolveTaggedReasoningOutputMode()`
    - helper-функції для endpoint/моделей, як-от `resolveProviderEndpoint(...)`,
      `normalizeProviderId(...)`, `normalizeGooglePreviewModelId(...)` і
      `normalizeNativeXaiModelId(...)`

    `openclaw/plugin-sdk/provider-stream` надає як builder сімейств,
    так і публічні helper-обгортки, які ці сімейства повторно використовують. Поширені публічні експорти
    включають:

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - спільні обгортки OpenAI/Codex, як-от
      `createOpenAIAttributionHeadersWrapper(...)`,
      `createOpenAIFastModeWrapper(...)`,
      `createOpenAIServiceTierWrapper(...)`,
      `createOpenAIResponsesContextManagementWrapper(...)` і
      `createCodexNativeWebSearchWrapper(...)`
    - спільні обгортки проксі/постачальників, як-от `createOpenRouterWrapper(...)`,
      `createToolStreamWrapper(...)` і `createMinimaxFastModeWrapper(...)`

    Деякі helper-функції для stream навмисно залишаються локальними для конкретного постачальника. Поточний вбудований
    приклад: `@openclaw/anthropic-provider` експортує
    `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` і
    builder-функції нижчого рівня для обгорток Anthropic зі свого публічного інтерфейсу `api.ts` /
    `contract-api.ts`. Ці helper-функції залишаються специфічними для Anthropic, оскільки
    вони також кодують обробку бета-версій Claude OAuth і обмеження `context1m`.

    Інші вбудовані постачальники також залишають локальними обгортки, специфічні для транспорту, коли
    цю поведінку неможливо чисто розділити між сімействами. Поточний приклад: вбудований
    плагін xAI залишає власне формування Responses xAI у своєму
    `wrapStreamFn`, зокрема переписування alias `/fast`, `tool_stream` за замовчуванням,
    очищення непідтримуваних strict-tool і видалення payload reasoning, специфічного для xAI.

    `openclaw/plugin-sdk/provider-tools` наразі надає одне спільне
    сімейство tool-schema плюс спільні helper-функції schema/compat:

    - `ProviderToolCompatFamily` документує поточний перелік спільних сімейств.
    - `buildProviderToolCompatFamilyHooks("gemini")` підключає очищення schema
      Gemini + діагностику для постачальників, яким потрібні Gemini-safe tool schemas.
    - `normalizeGeminiToolSchemas(...)` і `inspectGeminiToolSchemas(...)`
      — це базові публічні helper-функції schema Gemini.
    - `resolveXaiModelCompatPatch()` повертає вбудований compat patch xAI:
      `toolSchemaProfile: "xai"`, непідтримувані ключові слова schema, власну
      підтримку `web_search` і декодування аргументів виклику інструментів з HTML-entity.
    - `applyXaiModelCompat(model)` застосовує той самий compat patch xAI до
      визначеної моделі, перш ніж вона потрапить до runner.

    Реальний вбудований приклад: плагін xAI використовує `normalizeResolvedModel` плюс
    `contributeResolvedModelCompat`, щоб зберегти ці метадані compat у власності
    постачальника, а не жорстко кодувати правила xAI в core.

    Такий самий шаблон на рівні кореня пакета також лежить в основі інших вбудованих постачальників:

    - `@openclaw/openai-provider`: `api.ts` експортує builder-функції постачальника,
      helper-функції для моделей за замовчуванням і builder-функції постачальника realtime
    - `@openclaw/openrouter-provider`: `api.ts` експортує builder-функцію постачальника
      плюс helper-функції для onboarding/config

    <Tabs>
      <Tab title="Обмін токенами">
        Для постачальників, яким потрібен обмін токенами перед кожним викликом inference:

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
      <Tab title="Користувацькі заголовки">
        Для постачальників, яким потрібні користувацькі заголовки запиту або зміни тіла запиту:

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
      <Tab title="Власна ідентичність транспорту">
        Для постачальників, яким потрібні власні заголовки запиту/сесії або метадані на
        універсальних HTTP- чи WebSocket-транспортах:

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
      <Tab title="Використання та білінг">
        Для постачальників, які надають дані про використання/білінг:

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

    <Accordion title="Усі доступні hooks постачальника">
      OpenClaw викликає hooks у такому порядку. Більшість постачальників використовують лише 2-3:

      | # | Hook | Коли використовувати |
      | --- | --- | --- |
      | 1 | `catalog` | Каталог моделей або значення `baseUrl` за замовчуванням |
      | 2 | `applyConfigDefaults` | Глобальні значення за замовчуванням, що належать постачальнику, під час materialization config |
      | 3 | `normalizeModelId` | Очищення aliases застарілих/preview model-id перед пошуком |
      | 4 | `normalizeTransport` | Очищення `api` / `baseUrl` для сімейства постачальника перед загальним збиранням моделі |
      | 5 | `normalizeConfig` | Нормалізація config `models.providers.<id>` |
      | 6 | `applyNativeStreamingUsageCompat` | Переписування compat для власного streaming-usage для постачальників config |
      | 7 | `resolveConfigApiKey` | Визначення автентифікації за env-marker, що належить постачальнику |
      | 8 | `resolveSyntheticAuth` | Синтетична автентифікація local/self-hosted або на основі config |
      | 9 | `shouldDeferSyntheticProfileAuth` | Знижує пріоритет синтетичних placeholders збереженого профілю відносно env/config auth |
      | 10 | `resolveDynamicModel` | Прийом довільних upstream model IDs |
      | 11 | `prepareDynamicModel` | Асинхронне отримання метаданих перед визначенням |
      | 12 | `normalizeResolvedModel` | Переписування транспорту перед runner |

      Примітки щодо runtime fallback:

      - `normalizeConfig` спочатку перевіряє відповідного постачальника, потім інші
      плагіни постачальників, здатні обробляти hooks, доки один із них справді не змінить config.
      Якщо жоден hook постачальника не перепише підтримуваний запис config сімейства Google,
      усе одно застосовується вбудований нормалізатор config Google.
    - `resolveConfigApiKey` використовує hook постачальника, якщо він доступний. Вбудований
      шлях `amazon-bedrock` також має тут вбудований resolver AWS env-marker,
      хоча сама runtime auth Bedrock усе ще використовує стандартний
      ланцюжок AWS SDK.
      | 13 | `contributeResolvedModelCompat` | Прапори compat для моделей вендора за іншим сумісним транспортом |
      | 14 | `capabilities` | Застарілий статичний набір можливостей; лише для сумісності |
      | 15 | `normalizeToolSchemas` | Очищення tool-schema, що належить постачальнику, перед реєстрацією |
      | 16 | `inspectToolSchemas` | Діагностика tool-schema, що належить постачальнику |
      | 17 | `resolveReasoningOutputMode` | Контракт tagged vs native reasoning-output |
      | 18 | `prepareExtraParams` | Параметри запиту за замовчуванням |
      | 19 | `createStreamFn` | Повністю користувацький транспорт StreamFn |
      | 20 | `wrapStreamFn` | Користувацькі обгортки заголовків/тіла на звичайному шляху stream |
      | 21 | `resolveTransportTurnState` | Власні заголовки/метадані для кожного turn |
      | 22 | `resolveWebSocketSessionPolicy` | Власні заголовки сесії WS / період охолодження |
      | 23 | `formatApiKey` | Користувацька форма runtime token |
      | 24 | `refreshOAuth` | Користувацьке оновлення OAuth |
      | 25 | `buildAuthDoctorHint` | Підказка щодо виправлення автентифікації |
      | 26 | `matchesContextOverflowError` | Визначення переповнення, що належить постачальнику |
      | 27 | `classifyFailoverReason` | Класифікація rate-limit/overload, що належить постачальнику |
      | 28 | `isCacheTtlEligible` | Керування TTL кешу prompt |
      | 29 | `buildMissingAuthMessage` | Користувацька підказка про відсутню автентифікацію |
      | 30 | `suppressBuiltInModel` | Приховати застарілі upstream rows |
      | 31 | `augmentModelCatalog` | Синтетичні rows для forward-compat |
      | 32 | `isBinaryThinking` | Бінарний thinking увімк./вимк. |
      | 33 | `supportsXHighThinking` | Підтримка reasoning `xhigh` |
      | 34 | `resolveDefaultThinkingLevel` | Політика `/think` за замовчуванням |
      | 35 | `isModernModelRef` | Відповідність live/smoke моделей |
      | 36 | `prepareRuntimeAuth` | Обмін токенами перед inference |
      | 37 | `resolveUsageAuth` | Користувацький розбір облікових даних usage |
      | 38 | `fetchUsageSnapshot` | Користувацький endpoint usage |
      | 39 | `createEmbeddingProvider` | Адаптер embedding, що належить постачальнику, для memory/search |
      | 40 | `buildReplayPolicy` | Користувацька політика replay/compaction транскрипту |
      | 41 | `sanitizeReplayHistory` | Специфічні для постачальника переписування replay після загального очищення |
      | 42 | `validateReplayTurns` | Сувора валідація replay-turn перед вбудованим runner |
      | 43 | `onModelSelected` | Колбек після вибору моделі (наприклад, telemetry) |

      Примітка щодо налаштування prompt:

      - `resolveSystemPromptContribution` дозволяє постачальнику додавати cache-aware
        інструкції до system prompt для сімейства моделей. Віддавайте йому перевагу перед
        `before_prompt_build`, якщо поведінка належить одному сімейству постачальника/моделей
        і має зберігати стабільний/динамічний поділ кешу.

      Докладні описи та реальні приклади дивіться в
      [Internals: Provider Runtime Hooks](/uk/plugins/architecture#provider-runtime-hooks).
    </Accordion>

  </Step>

  <Step title="Додайте додаткові можливості (необов’язково)">
    <a id="step-5-add-extra-capabilities"></a>
    Плагін постачальника може реєструвати speech, realtime transcription, realtime
    voice, media understanding, image generation, video generation, web fetch
    і web search поряд із текстовим inference:

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

    OpenClaw класифікує це як плагін **hybrid-capability**. Це
    рекомендований шаблон для корпоративних плагінів (один плагін на вендора). Див.
    [Internals: Capability Ownership](/uk/plugins/architecture#capability-ownership-model).

    Для генерації відео віддавайте перевагу наведеній вище структурі можливостей з урахуванням режимів:
    `generate`, `imageToVideo` і `videoToVideo`. Пласкі агреговані поля, такі
    як `maxInputImages`, `maxInputVideos` і `maxDurationSeconds`, недостатні,
    щоб коректно оголошувати підтримку режимів трансформації або вимкнені режими.

    Постачальники генерації музики мають дотримуватися того самого шаблону:
    `generate` для генерації лише за prompt і `edit` для генерації
    на основі референсного зображення. Пласкі агреговані поля, такі як `maxInputImages`,
    `supportsLyrics` і `supportsFormat`, недостатні, щоб оголосити підтримку edit;
    очікуваним контрактом є явні блоки `generate` / `edit`.

  </Step>

  <Step title="Тестування">
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

## Публікація в ClawHub

Плагіни постачальників публікуються так само, як і будь-які інші зовнішні кодові плагіни:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

Не використовуйте тут застарілий alias публікації лише для Skills; пакети плагінів мають використовувати
`clawhub package publish`.

## Структура файлів

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers metadata
├── openclaw.plugin.json      # Manifest with provider auth metadata
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Tests
    └── usage.ts              # Usage endpoint (optional)
```

## Довідка щодо порядку catalog

`catalog.order` керує тим, коли ваш каталог об’єднується відносно вбудованих
постачальників:

| Order     | Коли          | Випадок використання                          |
| --------- | ------------- | --------------------------------------------- |
| `simple`  | Перший прохід | Звичайні постачальники з API key              |
| `profile` | Після simple  | Постачальники, прив’язані до профілів auth    |
| `paired`  | Після profile | Синтез кількох пов’язаних записів             |
| `late`    | Останній прохід | Перевизначення наявних постачальників (перемагає при конфлікті) |

## Наступні кроки

- [Channel Plugins](/uk/plugins/sdk-channel-plugins) — якщо ваш плагін також надає канал
- [SDK Runtime](/uk/plugins/sdk-runtime) — helper-функції `api.runtime` (TTS, search, subagent)
- [SDK Overview](/uk/plugins/sdk-overview) — повний довідник з імпорту subpath
- [Plugin Internals](/uk/plugins/architecture#provider-runtime-hooks) — деталі hooks і приклади вбудованих реалізацій
