---
read_when:
    - أنت تبني plugin جديدة لمزوّد نماذج
    - تريد إضافة proxy متوافق مع OpenAI أو LLM مخصص إلى OpenClaw
    - تحتاج إلى فهم مصادقة المزوّد، والفهارس، وhooks وقت التشغيل
sidebarTitle: Provider Plugins
summary: دليل خطوة بخطوة لبناء plugin لمزوّد نماذج في OpenClaw
title: بناء plugins للمزوّدين
x-i18n:
    generated_at: "2026-04-05T12:52:51Z"
    model: gpt-5.4
    provider: openai
    source_hash: e781e5fc436b2189b9f8cc63e7611f49df1fd2526604a0596a0631f49729b085
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# بناء plugins للمزوّدين

يرشدك هذا الدليل خلال بناء plugin لمزوّد تضيف مزوّد نماذج
(LLM) إلى OpenClaw. وبحلول النهاية سيكون لديك مزوّد يحتوي على فهرس نماذج،
ومصادقة عبر مفتاح API، وحل ديناميكي للنماذج.

<Info>
  إذا لم تكن قد بنيت أي plugin لـ OpenClaw من قبل، فاقرأ
  [البدء](/plugins/building-plugins) أولًا لمعرفة بنية الحزمة الأساسية
  وإعداد manifest.
</Info>

## الشرح العملي

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="الحزمة وmanifest">
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

    يعرّف manifest الحقل `providerAuthEnvVars` حتى يتمكن OpenClaw من اكتشاف
    بيانات الاعتماد من دون تحميل runtime الخاص بـ plugin. ويُعد `modelSupport` اختياريًا
    ويسمح لـ OpenClaw بتحميل plugin الخاصة بمزوّدك تلقائيًا من معرّفات نماذج مختصرة
    مثل `acme-large` قبل وجود hooks وقت التشغيل. وإذا نشرت
    المزوّد على ClawHub، فإن حقول `openclaw.compat` و`openclaw.build` هذه
    مطلوبة في `package.json`.

  </Step>

  <Step title="سجّل المزوّد">
    يحتاج المزوّد الأدنى إلى `id` و`label` و`auth` و`catalog`:

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

    هذا مزوّد عامل. ويمكن للمستخدمين الآن تشغيل
    `openclaw onboard --acme-ai-api-key <key>` ثم اختيار
    `acme-ai/acme-large` كنموذج لهم.

    بالنسبة إلى المزوّدين المضمّنين الذين لا يسجلون إلا مزوّد نص واحد مع
    مصادقة مفتاح API بالإضافة إلى runtime واحد يعتمد على catalog، ففضّل
    المساعد الأضيق `defineSingleProviderPluginEntry(...)`:

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

    إذا كان تدفق المصادقة لديك يحتاج أيضًا إلى تصحيح `models.providers.*`، والأسماء المستعارة، و
    النموذج الافتراضي للوكيل أثناء onboarding، فاستخدم مساعدات preset من
    `openclaw/plugin-sdk/provider-onboard`. وأضيق المساعدات هي
    `createDefaultModelPresetAppliers(...)`،
    و`createDefaultModelsPresetAppliers(...)`، و
    `createModelCatalogPresetAppliers(...)`.

    عندما تدعم نقطة النهاية الأصلية للمزوّد كتل usage متدفقة على
    وسيلة النقل العادية `openai-completions`، ففضّل المساعدات المشتركة الخاصة بـ catalog في
    `openclaw/plugin-sdk/provider-catalog-shared` بدلًا من ترميز فحوصات
    معرّف المزوّد يدويًا. تقوم `supportsNativeStreamingUsageCompat(...)` و
    `applyProviderNativeStreamingUsageCompat(...)` باكتشاف الدعم من خريطة قدرات
    نقطة النهاية، بحيث تستمر نقاط النهاية الأصلية بأسلوب Moonshot/DashScope
    في الاشتراك حتى عندما يستخدم plugin معرّف مزوّد مخصصًا.

  </Step>

  <Step title="أضف الحل الديناميكي للنماذج">
    إذا كان مزوّدك يقبل معرّفات نماذج عشوائية (مثل proxy أو موجّه)،
    فأضف `resolveDynamicModel`:

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

    إذا كان الحل يتطلب استدعاء شبكة، فاستخدم `prepareDynamicModel` من أجل
    الإحماء غير المتزامن — إذ يتم تشغيل `resolveDynamicModel` مرة أخرى بعد اكتماله.

  </Step>

  <Step title="أضف hooks وقت التشغيل (عند الحاجة)">
    تحتاج معظم المزوّدات فقط إلى `catalog` + `resolveDynamicModel`. وأضف hooks
    تدريجيًا حسب ما يتطلبه مزوّدك.

    تغطي أدوات البناء المساعدة المشتركة الآن أكثر عائلات توافق replay/tool شيوعًا،
    لذا لا تحتاج plugins عادةً إلى توصيل كل hook يدويًا:

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

    عائلات replay المتاحة اليوم:

    | العائلة | ما الذي توصله |
    | --- | --- |
    | `openai-compatible` | سياسة replay مشتركة بأسلوب OpenAI لوسائل النقل المتوافقة مع OpenAI، بما في ذلك تنقية tool-call-id، وإصلاحات ترتيب assistant-first، والتحقق العام من أدوار Gemini عندما تحتاج وسيلة النقل إلى ذلك |
    | `anthropic-by-model` | سياسة replay مدركة لـ Claude تُختار بواسطة `modelId`، بحيث لا تحصل وسائل نقل Anthropic-message على تنظيف كتل التفكير الخاصة بـ Claude إلا عندما يكون النموذج المحلول في الواقع معرّف Claude |
    | `google-gemini` | سياسة replay أصلية لـ Gemini بالإضافة إلى تنقية bootstrap replay ووضع مخرجات reasoning الموسوم |
    | `passthrough-gemini` | تنقية thought-signature الخاصة بـ Gemini للنماذج التي تعمل عبر وسائل نقل proxy المتوافقة مع OpenAI؛ ولا تفعّل التحقق الأصلي من replay في Gemini أو إعادة كتابة bootstrap |
    | `hybrid-anthropic-openai` | سياسة هجينة للمزوّدين الذين يخلطون بين أسطح نماذج Anthropic-message وOpenAI-compatible في plugin واحدة؛ ويظل إسقاط كتل التفكير الخاصة بـ Claude اختياريًا ومحصورًا بجانب Anthropic |

    أمثلة حقيقية من المزوّدين المضمنين:

    - `google` و`google-gemini-cli`: ‏`google-gemini`
    - `openrouter` و`kilocode` و`opencode` و`opencode-go`: ‏`passthrough-gemini`
    - `amazon-bedrock` و`anthropic-vertex`: ‏`anthropic-by-model`
    - `minimax`: ‏`hybrid-anthropic-openai`
    - `moonshot` و`ollama` و`xai` و`zai`: ‏`openai-compatible`

    عائلات stream المتاحة اليوم:

    | العائلة | ما الذي توصله |
    | --- | --- |
    | `google-thinking` | توحيد حمولة التفكير الخاصة بـ Gemini على مسار stream المشترك |
    | `kilocode-thinking` | wrapper للاستدلال في Kilo على مسار stream المشترك الخاص بالـ proxy، مع تخطي `kilo/auto` ومعرّفات الاستدلال غير المدعومة للتفكير المحقون |
    | `moonshot-thinking` | تعيين حمولة التفكير الثنائية الأصلية في Moonshot من التكوين + مستوى `/think` |
    | `minimax-fast-mode` | إعادة كتابة نموذج الوضع السريع MiniMax على مسار stream المشترك |
    | `openai-responses-defaults` | wrappers مشتركة أصلية لـ OpenAI/Codex Responses: رؤوس الإسناد، و`/fast`/`serviceTier`، وverbosity النص، والبحث الأصلي على الويب في Codex، وتشكيل حمولة reasoning-compat، وإدارة سياق Responses |
    | `openrouter-thinking` | wrapper للاستدلال في OpenRouter لمسارات proxy، مع معالجة مركزية لتخطي النماذج غير المدعومة/`auto` |
    | `tool-stream-default-on` | wrapper افتراضي التفعيل لـ `tool_stream` للمزوّدين مثل Z.AI الذين يريدون بث الأدوات ما لم يتم تعطيله صراحةً |

    أمثلة حقيقية من المزوّدين المضمنين:

    - `google` و`google-gemini-cli`: ‏`google-thinking`
    - `kilocode`: ‏`kilocode-thinking`
    - `moonshot`: ‏`moonshot-thinking`
    - `minimax` و`minimax-portal`: ‏`minimax-fast-mode`
    - `openai` و`openai-codex`: ‏`openai-responses-defaults`
    - `openrouter`: ‏`openrouter-thinking`
    - `zai`: ‏`tool-stream-default-on`

    كما يصدّر `openclaw/plugin-sdk/provider-model-shared` تعداد replay-family
    بالإضافة إلى المساعدات المشتركة التي تُبنى منها تلك العائلات. وتشمل
    الصادرات العامة الشائعة:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - أدوات بناء replay المشتركة مثل `buildOpenAICompatibleReplayPolicy(...)`،
      و`buildAnthropicReplayPolicyForModel(...)`،
      و`buildGoogleGeminiReplayPolicy(...)`، و
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)`
    - مساعدات replay الخاصة بـ Gemini مثل `sanitizeGoogleGeminiReplayHistory(...)`
      و`resolveTaggedReasoningOutputMode()`
    - مساعدات endpoint/model مثل `resolveProviderEndpoint(...)`،
      و`normalizeProviderId(...)`، و`normalizeGooglePreviewModelId(...)`، و
      `normalizeNativeXaiModelId(...)`

    يوفّر `openclaw/plugin-sdk/provider-stream` كلًا من أداة بناء العائلة
    وwrapperات المساعدة العامة التي تعيد هذه العائلات استخدامها. وتشمل
    الصادرات العامة الشائعة:

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - wrappers المشتركة الخاصة بـ OpenAI/Codex مثل
      `createOpenAIAttributionHeadersWrapper(...)`،
      و`createOpenAIFastModeWrapper(...)`،
      و`createOpenAIServiceTierWrapper(...)`،
      و`createOpenAIResponsesContextManagementWrapper(...)`، و
      `createCodexNativeWebSearchWrapper(...)`
    - wrappers المشتركة الخاصة بالـ proxy/المزوّد مثل `createOpenRouterWrapper(...)`،
      و`createToolStreamWrapper(...)`، و`createMinimaxFastModeWrapper(...)`

    تبقى بعض مساعدات stream محلية للمزوّد عن قصد. والمثال
    الحالي من المزوّدين المضمنين: يصدّر `@openclaw/anthropic-provider`
    الدوال `wrapAnthropicProviderStream`، و`resolveAnthropicBetas`،
    و`resolveAnthropicFastMode`، و`resolveAnthropicServiceTier`، و
    أدوات بناء wrapper منخفضة المستوى الخاصة بـ Anthropic من خلال السطح العام `api.ts` /
    `contract-api.ts`. وتبقى هذه المساعدات خاصة بـ Anthropic لأنها
    تتضمن أيضًا معالجة beta الخاصة بـ Claude OAuth وقيود `context1m`.

    كما تحتفظ مزوّدات مضمنة أخرى أيضًا بـ wrappers خاصة بوسيلة النقل محليًا عندما
    لا يكون السلوك مشتركًا بشكل نظيف بين العائلات. والمثال الحالي: يحتفظ
    plugin ‏xAI المضمن بتشكيل xAI Responses الأصلي داخل
    `wrapStreamFn` الخاص به، بما في ذلك إعادة كتابة الاسم المستعار `/fast`، و`tool_stream`
    الافتراضي، وتنظيف strict-tool غير المدعوم، وإزالة
    حمولة الاستدلال الخاصة بـ xAI.

    يعرّض `openclaw/plugin-sdk/provider-tools` حاليًا عائلة واحدة مشتركة
    لمخطط الأدوات بالإضافة إلى مساعدات schema/compat مشتركة:

    - يوثق `ProviderToolCompatFamily` مخزون العائلات المشتركة اليوم.
    - يقوم `buildProviderToolCompatFamilyHooks("gemini")` بتوصيل
      تنظيف مخطط Gemini + التشخيصات للمزوّدين الذين يحتاجون إلى مخططات أدوات آمنة لـ Gemini.
    - تمثل `normalizeGeminiToolSchemas(...)` و`inspectGeminiToolSchemas(...)`
      المساعدات العامة الأساسية لمخطط Gemini.
    - تعيد `resolveXaiModelCompatPatch()` تصحيح compat المضمن لـ xAI:
      ‏`toolSchemaProfile: "xai"`، والكلمات المفتاحية غير المدعومة في schema، ودعم
      `web_search` الأصلي، وفك ترميز وسائط tool-call المشفرة بـ HTML entities.
    - تطبق `applyXaiModelCompat(model)` تصحيح compat نفسه لـ xAI على
      نموذج محلول قبل أن يصل إلى runner.

    مثال حقيقي من المزوّدين المضمنين: يستخدم plugin ‏xAI كلًا من `normalizeResolvedModel` و
    `contributeResolvedModelCompat` للحفاظ على ملكية بيانات compat الوصفية
    لدى المزوّد بدلًا من ترميز قواعد xAI في core.

    كما يدعم نمط جذر الحزمة نفسه مزوّدين مضمنين آخرين:

    - `@openclaw/openai-provider`: يصدّر `api.ts` أدوات بناء المزوّد،
      ومساعدات النموذج الافتراضي، وأدوات بناء مزوّدات realtime
    - `@openclaw/openrouter-provider`: يصدّر `api.ts` أداة بناء المزوّد
      بالإضافة إلى مساعدات onboarding/config

    <Tabs>
      <Tab title="تبادل الرموز المميزة">
        بالنسبة إلى المزوّدين الذين يحتاجون إلى تبادل رمز مميز قبل كل استدعاء استدلال:

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
      <Tab title="رؤوس مخصصة">
        بالنسبة إلى المزوّدين الذين يحتاجون إلى رؤوس طلبات مخصصة أو تعديلات على النص:

        ```typescript
        // تعيد wrapStreamFn قيمة StreamFn مشتقة من ctx.streamFn
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
      <Tab title="هوية النقل الأصلية">
        بالنسبة إلى المزوّدين الذين يحتاجون إلى رؤوس/بيانات وصفية أصلية للطلب أو الجلسة على
        وسائل النقل العامة HTTP أو WebSocket:

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
      <Tab title="الاستخدام والفوترة">
        بالنسبة إلى المزوّدين الذين يكشفون بيانات الاستخدام/الفوترة:

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

    <Accordion title="جميع hooks المزوّد المتاحة">
      يستدعي OpenClaw hooks بهذا الترتيب. وتستخدم معظم المزوّدات 2-3 فقط:

      | # | Hook | متى تستخدمها |
      | --- | --- | --- |
      | 1 | `catalog` | فهرس النماذج أو الإعدادات الافتراضية لـ base URL |
      | 2 | `applyConfigDefaults` | إعدادات افتراضية عامة يملكها المزوّد أثناء تجسيد التكوين |
      | 3 | `normalizeModelId` | تنظيف الأسماء المستعارة القديمة/preview الخاصة بـ model-id قبل البحث |
      | 4 | `normalizeTransport` | تنظيف `api` / `baseUrl` لعائلة المزوّد قبل بناء النموذج العام |
      | 5 | `normalizeConfig` | توحيد تكوين `models.providers.<id>` |
      | 6 | `applyNativeStreamingUsageCompat` | إعادة كتابة compat الخاصة بالاستخدام المتدفق الأصلي لمزوّدي التكوين |
      | 7 | `resolveConfigApiKey` | حل مصادقة env-marker المملوكة للمزوّد |
      | 8 | `resolveSyntheticAuth` | مصادقة synthetic محلية/مستضافة ذاتيًا أو مدعومة بالتكوين |
      | 9 | `shouldDeferSyntheticProfileAuth` | خفض أولوية العناصر النائبة synthetic المخزنة في stored-profile خلف مصادقة env/config |
      | 10 | `resolveDynamicModel` | قبول معرّفات نماذج upstream عشوائية |
      | 11 | `prepareDynamicModel` | جلب بيانات وصفية غير متزامن قبل الحل |
      | 12 | `normalizeResolvedModel` | إعادة كتابة النقل قبل runner |

    ملاحظات الرجوع في وقت التشغيل:

    - يتحقق `normalizeConfig` من المزوّد المطابق أولًا، ثم من plugins المزوّدات الأخرى القادرة على التعامل مع
      hook حتى يغيّر أحدها التكوين فعليًا.
      وإذا لم تعِد كتابة أي hook من hooks المزوّد إدخال تكوين مدعوم من عائلة Google، فإن
      أداة توحيد تكوين Google المضمنة ما زالت تُطبَّق.
    - يستخدم `resolveConfigApiKey` hook الخاص بالمزوّد عند توفره. كما أن
      المسار المضمن `amazon-bedrock` يحتوي أيضًا على محلل مدمج لمحددات بيئة AWS هنا،
      رغم أن مصادقة Bedrock وقت التشغيل نفسها ما زالت تستخدم سلسلة AWS SDK الافتراضية.
      | 13 | `contributeResolvedModelCompat` | أعلام compat لنماذج المزوّد خلف وسيلة نقل متوافقة أخرى |
      | 14 | `capabilities` | حقيبة إمكانات ثابتة قديمة؛ للتوافق فقط |
      | 15 | `normalizeToolSchemas` | تنظيف مخطط الأدوات الخاص بالمزوّد قبل التسجيل |
      | 16 | `inspectToolSchemas` | تشخيصات مخطط الأدوات الخاصة بالمزوّد |
      | 17 | `resolveReasoningOutputMode` | عقد مخرجات reasoning الموسومة مقابل الأصلية |
      | 18 | `prepareExtraParams` | معلمات الطلب الافتراضية |
      | 19 | `createStreamFn` | وسيلة نقل StreamFn مخصصة بالكامل |
      | 20 | `wrapStreamFn` | wrappers مخصصة للرؤوس/النصوص على مسار stream العادي |
      | 21 | `resolveTransportTurnState` | رؤوس/بيانات وصفية أصلية لكل دور |
      | 22 | `resolveWebSocketSessionPolicy` | رؤوس جلسة WS أصلية/فترة تهدئة |
      | 23 | `formatApiKey` | شكل token مخصص في وقت التشغيل |
      | 24 | `refreshOAuth` | تحديث OAuth مخصص |
      | 25 | `buildAuthDoctorHint` | إرشادات إصلاح المصادقة |
      | 26 | `matchesContextOverflowError` | اكتشاف overflow يملكه المزوّد |
      | 27 | `classifyFailoverReason` | تصنيف rate-limit/overload يملكه المزوّد |
      | 28 | `isCacheTtlEligible` | تقييد TTL الخاص بـ prompt cache |
      | 29 | `buildMissingAuthMessage` | تلميح مخصص لغياب المصادقة |
      | 30 | `suppressBuiltInModel` | إخفاء الصفوف القديمة في المصدر |
      | 31 | `augmentModelCatalog` | صفوف synthetic للتوافق المستقبلي |
      | 32 | `isBinaryThinking` | تشغيل/إيقاف التفكير الثنائي |
      | 33 | `supportsXHighThinking` | دعم reasoning من نوع `xhigh` |
      | 34 | `resolveDefaultThinkingLevel` | سياسة `/think` الافتراضية |
      | 35 | `isModernModelRef` | مطابقة النماذج الحية/نماذج smoke |
      | 36 | `prepareRuntimeAuth` | تبادل token قبل الاستدلال |
      | 37 | `resolveUsageAuth` | تحليل بيانات اعتماد الاستخدام المخصصة |
      | 38 | `fetchUsageSnapshot` | نقطة نهاية استخدام مخصصة |
      | 39 | `createEmbeddingProvider` | مهايئ embeddings يملكه المزوّد للذاكرة/البحث |
      | 40 | `buildReplayPolicy` | سياسة replay/compaction مخصصة للنصوص |
      | 41 | `sanitizeReplayHistory` | إعادة كتابة replay خاصة بالمزوّد بعد التنظيف العام |
      | 42 | `validateReplayTurns` | تحقق صارم من أدوار replay قبل embedded runner |
      | 43 | `onModelSelected` | callback بعد الاختيار (مثل telemetry) |

      للحصول على أوصاف مفصلة وأمثلة واقعية، راجع
      [الداخليات: hooks وقت تشغيل المزوّد](/plugins/architecture#provider-runtime-hooks).
    </Accordion>

  </Step>

  <Step title="أضف إمكانات إضافية (اختياري)">
    <a id="step-5-add-extra-capabilities"></a>
    يمكن لـ plugin الخاصة بالمزوّد تسجيل خدمات speech، والنسخ الفوري realtime transcription، والصوت الفوري realtime
    voice، وفهم الوسائط، وتوليد الصور، وتوليد الفيديو، وweb fetch،
    وweb search إلى جانب استدلال النص:

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

    يصنف OpenClaw هذا على أنه plugin ذات **إمكانات هجينة**. وهذا هو
    النمط الموصى به لـ plugins الخاصة بالشركات (plugin واحدة لكل مزود). راجع
    [الداخليات: ملكية الإمكانات](/plugins/architecture#capability-ownership-model).

  </Step>

  <Step title="اختبر">
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

## النشر إلى ClawHub

تُنشر plugins الخاصة بالمزوّدين بالطريقة نفسها التي تُنشر بها أي plugin خارجية أخرى:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

لا تستخدم الاسم المستعار القديم الخاص بالنشر المرتبط بالـ Skills فقط هنا؛ إذ يجب أن تستخدم
حزم plugins الأمر `clawhub package publish`.

## بنية الملفات

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers metadata
├── openclaw.plugin.json      # Manifest with providerAuthEnvVars
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Tests
    └── usage.ts              # Usage endpoint (optional)
```

## مرجع ترتيب catalog

يتحكم `catalog.order` في توقيت دمج catalog الخاصة بك مقارنةً بالمزوّدين
المضمنين:

| الترتيب     | متى          | حالة الاستخدام                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | المرور الأول    | مزوّدات مفاتيح API البسيطة                         |
| `profile` | بعد simple  | مزوّدات تعتمد على ملفات المصادقة الشخصية                |
| `paired`  | بعد profile | توليد عدة إدخالات مترابطة             |
| `late`    | المرور الأخير     | تجاوز المزوّدين الحاليين (يفوز عند التصادم) |

## الخطوات التالية

- [plugins القنوات](/plugins/sdk-channel-plugins) — إذا كانت plugin توفر قناة أيضًا
- [وقت تشغيل SDK](/plugins/sdk-runtime) — مساعدات `api.runtime` ‏(TTS، والبحث، والوكيل الفرعي)
- [نظرة عامة على SDK](/plugins/sdk-overview) — المرجع الكامل للاستيراد عبر subpath
- [داخليات plugin](/plugins/architecture#provider-runtime-hooks) — تفاصيل hooks وأمثلة من المزوّدين المضمنين
