---
read_when:
    - أنت تبني plugin جديدًا لموفر نموذج
    - تريد إضافة proxy متوافق مع OpenAI أو LLM مخصص إلى OpenClaw
    - تحتاج إلى فهم مصادقة الموفّر، والفهارس، وruntime hooks
sidebarTitle: Provider Plugins
summary: دليل خطوة بخطوة لبناء plugin لموفر نموذج لـ OpenClaw
title: بناء Provider Plugins
x-i18n:
    generated_at: "2026-04-06T03:11:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: 69500f46aa2cfdfe16e85b0ed9ee3c0032074be46f2d9c9d2940d18ae1095f47
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# بناء Provider Plugins

يرشدك هذا الدليل خلال بناء plugin لموفّر يضيف موفر نموذج
(LLM) إلى OpenClaw. وبنهاية هذا الدليل، سيكون لديك موفّر يحتوي على فهرس نماذج،
ومصادقة بمفتاح API، وحل ديناميكي للنماذج.

<Info>
  إذا لم تكن قد أنشأت أي OpenClaw plugin من قبل، فاقرأ
  [البدء](/ar/plugins/building-plugins) أولًا للتعرف على بنية الحزمة
  الأساسية وإعداد البيان.
</Info>

## شرح عملي

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="الحزمة والبيان">
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

    يصرّح البيان بالحقل `providerAuthEnvVars` حتى يتمكن OpenClaw من اكتشاف
    بيانات الاعتماد من دون تحميل runtime الخاص بـ plugin لديك. ويُعد `modelSupport` اختياريًا
    ويسمح لـ OpenClaw بتحميل plugin الموفّر تلقائيًا من معرّفات نماذج مختصرة
    مثل `acme-large` قبل وجود runtime hooks. وإذا نشرت
    الموفّر على ClawHub، فإن حقول `openclaw.compat` و`openclaw.build`
    هذه مطلوبة في `package.json`.

  </Step>

  <Step title="تسجيل الموفّر">
    يحتاج الموفّر الأدنى إلى `id` و`label` و`auth` و`catalog`:

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

    هذا موفّر عامل بالفعل. ويمكن للمستخدمين الآن
    تشغيل `openclaw onboard --acme-ai-api-key <key>` واختيار
    `acme-ai/acme-large` كنموذج لهم.

    بالنسبة إلى الموفّرين المضمنين الذين يسجلون فقط موفّرًا نصيًا واحدًا مع
    مصادقة بمفتاح API بالإضافة إلى runtime واحد مدعوم بفهرس، فافضل
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
    النموذج الافتراضي للوكيل أثناء onboarding، فاستخدم المساعدات الجاهزة من
    `openclaw/plugin-sdk/provider-onboard`. وأضيق المساعدات هي
    `createDefaultModelPresetAppliers(...)`,
    و`createDefaultModelsPresetAppliers(...)`، و
    `createModelCatalogPresetAppliers(...)`.

    عندما تدعم نقطة النهاية الأصلية للموفّر كتل الاستخدام المتدفقة على
    ناقل `openai-completions` العادي، فافضّل مساعدات الفهرس المشتركة في
    `openclaw/plugin-sdk/provider-catalog-shared` بدلًا من ترميز
    عمليات التحقق من معرّف الموفّر بشكل ثابت. تكتشف
    `supportsNativeStreamingUsageCompat(...)` و
    `applyProviderNativeStreamingUsageCompat(...)` الدعم من خريطة قدرات نقطة النهاية، بحيث تظل نقاط النهاية الأصلية بنمط Moonshot/DashScope
    قادرة على الاشتراك حتى عندما يستخدم plugin معرّف موفّر مخصصًا.

  </Step>

  <Step title="إضافة حل ديناميكي للنموذج">
    إذا كان الموفّر لديك يقبل معرّفات نماذج عشوائية (مثل proxy أو router)،
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
    الإحماء غير المتزامن — يتم تشغيل `resolveDynamicModel` مرة أخرى بعد اكتماله.

  </Step>

  <Step title="إضافة runtime hooks (عند الحاجة)">
    تحتاج معظم الموفّرات فقط إلى `catalog` و`resolveDynamicModel`. أضف hooks
    تدريجيًا حسب ما يتطلبه الموفّر لديك.

    تغطي أدوات البناء المساعدة المشتركة الآن أكثر عائلات replay/tool-compat
    شيوعًا، لذلك لا تحتاج plugins عادة إلى توصيل كل hook يدويًا واحدًا تلو الآخر:

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
    | `openai-compatible` | سياسة replay مشتركة بنمط OpenAI لناقلات OpenAI-compatible، بما في ذلك تنقية `tool-call-id`، وإصلاحات ترتيب assistant-first، والتحقق العام من أدوار Gemini عندما يحتاجه الناقل |
    | `anthropic-by-model` | سياسة replay واعية بـ Claude يتم اختيارها حسب `modelId`، بحيث لا تحصل نواقل رسائل Anthropic على تنظيف كتل التفكير الخاصة بـ Claude إلا عندما يكون النموذج المحلول فعليًا معرّف Claude |
    | `google-gemini` | سياسة replay أصلية لـ Gemini بالإضافة إلى تنقية bootstrap replay ووضع مخرجات reasoning الموسومة |
    | `passthrough-gemini` | تنقية thought-signature لـ Gemini للنماذج التي تعمل عبر نواقل proxy متوافقة مع OpenAI؛ ولا يفعّل التحقق الأصلي من Gemini replay أو إعادة كتابة bootstrap |
    | `hybrid-anthropic-openai` | سياسة هجينة للموفّرين الذين يخلطون بين أسطح نماذج رسائل Anthropic والأسطح المتوافقة مع OpenAI داخل plugin واحد؛ ويظل إسقاط كتل التفكير الخاصة بـ Claude اختياريًا ومحصورًا في جانب Anthropic |

    أمثلة حقيقية من الموفّرين المضمنين:

    - `google`: ‏`google-gemini`
    - `openrouter` و`kilocode` و`opencode` و`opencode-go`: ‏`passthrough-gemini`
    - `amazon-bedrock` و`anthropic-vertex`: ‏`anthropic-by-model`
    - `minimax`: ‏`hybrid-anthropic-openai`
    - `moonshot` و`ollama` و`xai` و`zai`: ‏`openai-compatible`

    عائلات stream المتاحة اليوم:

    | العائلة | ما الذي توصله |
    | --- | --- |
    | `google-thinking` | تطبيع حمولة التفكير الخاصة بـ Gemini على مسار stream المشترك |
    | `kilocode-thinking` | غلاف reasoning الخاص بـ Kilo على مسار proxy stream المشترك، مع تجاوز `kilo/auto` ومعرّفات reasoning غير المدعومة بدون حقن التفكير |
    | `moonshot-thinking` | تعيين حمولة native-thinking الثنائية الخاصة بـ Moonshot من الإعدادات + مستوى `/think` |
    | `minimax-fast-mode` | إعادة كتابة نموذج MiniMax fast-mode على مسار stream المشترك |
    | `openai-responses-defaults` | مغلفات Responses الأصلية المشتركة لـ OpenAI/Codex: ترويسات الإسناد، و`/fast`/`serviceTier`، وverbosity النص، وCodex web search الأصلية، وتشكيل حمولة reasoning-compat، وإدارة سياق Responses |
    | `openrouter-thinking` | غلاف reasoning الخاص بـ OpenRouter لمسارات proxy، مع التعامل مركزيًا مع تجاوز `auto`/النموذج غير المدعوم |
    | `tool-stream-default-on` | غلاف `tool_stream` مفعّل افتراضيًا للموفّرين مثل Z.AI الذين يريدون تدفق الأدوات ما لم يتم تعطيله صراحة |

    أمثلة حقيقية من الموفّرين المضمنين:

    - `google`: ‏`google-thinking`
    - `kilocode`: ‏`kilocode-thinking`
    - `moonshot`: ‏`moonshot-thinking`
    - `minimax` و`minimax-portal`: ‏`minimax-fast-mode`
    - `openai` و`openai-codex`: ‏`openai-responses-defaults`
    - `openrouter`: ‏`openrouter-thinking`
    - `zai`: ‏`tool-stream-default-on`

    يصدّر `openclaw/plugin-sdk/provider-model-shared` أيضًا تعداد replay-family
    بالإضافة إلى المساعدات المشتركة التي تُبنى منها هذه العائلات. ومن
    الصادرات العامة الشائعة:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - أدوات بناء replay المشتركة مثل `buildOpenAICompatibleReplayPolicy(...)`،
      و`buildAnthropicReplayPolicyForModel(...)`،
      و`buildGoogleGeminiReplayPolicy(...)`، و
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)`
    - مساعدات replay الخاصة بـ Gemini مثل `sanitizeGoogleGeminiReplayHistory(...)`
      و`resolveTaggedReasoningOutputMode()`
    - مساعدات نقطة النهاية/النموذج مثل `resolveProviderEndpoint(...)`،
      و`normalizeProviderId(...)`، و`normalizeGooglePreviewModelId(...)`، و
      `normalizeNativeXaiModelId(...)`

    يكشف `openclaw/plugin-sdk/provider-stream` كلًا من أداة بناء العائلة
    والمساعدات العامة الخاصة بالأغلفة التي تعيد هذه العائلات استخدامها. ومن
    الصادرات العامة الشائعة:

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - أغلفة OpenAI/Codex المشتركة مثل
      `createOpenAIAttributionHeadersWrapper(...)`,
      و`createOpenAIFastModeWrapper(...)`,
      و`createOpenAIServiceTierWrapper(...)`,
      و`createOpenAIResponsesContextManagementWrapper(...)`، و
      `createCodexNativeWebSearchWrapper(...)`
    - أغلفة proxy/provider المشتركة مثل `createOpenRouterWrapper(...)`،
      و`createToolStreamWrapper(...)`، و`createMinimaxFastModeWrapper(...)`

    تبقى بعض مساعدات stream محلية للموفّر عن قصد. المثال المضمّن الحالي:
    يصدّر `@openclaw/anthropic-provider`
    `wrapAnthropicProviderStream` و`resolveAnthropicBetas`،
    و`resolveAnthropicFastMode`، و`resolveAnthropicServiceTier`، وأدوات
    بناء أغلفة Anthropic ذات المستوى الأدنى من وصلة `api.ts` /
    `contract-api.ts` العامة الخاصة به. وتبقى هذه المساعدات خاصة بـ Anthropic
    لأنها ترمّز أيضًا معالجة Claude OAuth beta وقيود `context1m`.

    تحتفظ موفّرات مضمنة أخرى أيضًا بأغلفة خاصة بالنقل محليًا عندما لا يكون
    السلوك مشتركًا بشكل نظيف عبر العائلات. المثال الحالي: يحتفظ
    plugin xAI المضمن بتشكيل xAI Responses الأصلية داخل
    `wrapStreamFn` الخاص به، بما في ذلك إعادة كتابة الأسماء المستعارة لـ `/fast`،
    و`tool_stream` الافتراضي، وتنظيف strict-tool غير المدعوم، وإزالة
    حمولة reasoning الخاصة بـ xAI.

    يكشف `openclaw/plugin-sdk/provider-tools` حاليًا عائلة واحدة مشتركة
    لمخطط الأدوات بالإضافة إلى مساعدات schema/compat المشتركة:

    - يوثق `ProviderToolCompatFamily` قائمة العائلات المشتركة المتاحة اليوم.
    - يوصل `buildProviderToolCompatFamilyHooks("gemini")` عمليات تنظيف
      مخطط Gemini + التشخيصات للموفّرين الذين يحتاجون إلى مخططات أدوات آمنة لـ Gemini.
    - يمثّل `normalizeGeminiToolSchemas(...)` و`inspectGeminiToolSchemas(...)`
      مساعدات مخطط Gemini العامة الأساسية.
    - يعيد `resolveXaiModelCompatPatch()` تصحيح compat المضمن الخاص بـ xAI:
      `toolSchemaProfile: "xai"`، وكلمات schema المفتاحية غير المدعومة، ودعم
      `web_search` الأصلي، وفك ترميز وسائط استدعاء الأدوات ذات كيانات HTML.
    - يطبّق `applyXaiModelCompat(model)` تصحيح compat نفسه على النموذج
      المحلول قبل وصوله إلى المشغّل.

    مثال حقيقي من الموفّرين المضمنين: يستخدم plugin الخاص بـ xAI
    `normalizeResolvedModel` مع `contributeResolvedModelCompat` للإبقاء على
    بيانات compat الوصفية هذه مملوكة للموفّر بدلًا من ترميز قواعد xAI
    بشكل ثابت في core.

    يدعم نمط جذر الحزمة نفسه أيضًا موفّرين مضمنين آخرين:

    - `@openclaw/openai-provider`: يصدّر `api.ts` أدوات بناء الموفّر،
      ومساعدات النموذج الافتراضي، وأدوات بناء realtime provider
    - `@openclaw/openrouter-provider`: يصدّر `api.ts` أداة بناء الموفّر
      بالإضافة إلى مساعدات onboarding/config

    <Tabs>
      <Tab title="تبادل الرمز">
        بالنسبة إلى الموفّرين الذين يحتاجون إلى تبادل رمز قبل كل استدعاء استدلال:

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
      <Tab title="ترويسات مخصصة">
        بالنسبة إلى الموفّرين الذين يحتاجون إلى ترويسات طلب مخصصة أو تعديلات في الجسم:

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
      <Tab title="هوية النقل الأصلية">
        بالنسبة إلى الموفّرين الذين يحتاجون إلى ترويسات/بيانات وصفية أصلية للطلب أو الجلسة على
        نواقل HTTP أو WebSocket العامة:

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
        بالنسبة إلى الموفّرين الذين يكشفون بيانات الاستخدام/الفوترة:

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

    <Accordion title="جميع Provider hooks المتاحة">
      يستدعي OpenClaw الـ hooks بهذا الترتيب. وتستخدم معظم الموفّرات 2-3 فقط:

      | # | Hook | متى تستخدمه |
      | --- | --- | --- |
      | 1 | `catalog` | فهرس النموذج أو إعدادات `baseUrl` الافتراضية |
      | 2 | `applyConfigDefaults` | إعدادات افتراضية عامة يملكها الموفّر أثناء materialization للإعدادات |
      | 3 | `normalizeModelId` | تنظيف الأسماء المستعارة القديمة/المعاينة لـ model-id قبل lookup |
      | 4 | `normalizeTransport` | تنظيف `api` / `baseUrl` الخاص بعائلة الموفّر قبل التجميع العام للنموذج |
      | 5 | `normalizeConfig` | تطبيع إعداد `models.providers.<id>` |
      | 6 | `applyNativeStreamingUsageCompat` | إعادات كتابة compat للاستخدام المتدفق الأصلي لموفّري الإعدادات |
      | 7 | `resolveConfigApiKey` | حل المصادقة لعلامات env الذي يملكه الموفّر |
      | 8 | `resolveSyntheticAuth` | مصادقة synthetic محلية/مستضافة ذاتيًا أو مدعومة بالإعدادات |
      | 9 | `shouldDeferSyntheticProfileAuth` | تأخير عناصر placeholder لملف التعريف المخزّن synthetic إلى ما بعد مصادقة env/config |
      | 10 | `resolveDynamicModel` | قبول معرّفات نماذج upstream عشوائية |
      | 11 | `prepareDynamicModel` | جلب بيانات وصفية غير متزامن قبل الحل |
      | 12 | `normalizeResolvedModel` | إعادات كتابة النقل قبل المشغّل |

      ملاحظات fallback الخاصة بـ runtime:

      - يتحقق `normalizeConfig` من الموفّر المطابق أولًا، ثم من
        plugins الموفّرة الأخرى القادرة على hook حتى يقوم أحدها فعليًا بتغيير الإعدادات.
        وإذا لم تعِد أي hook خاصة بموفّر كتابة إدخال إعدادات مدعوم من عائلة Google،
        فسيظل مطبّع إعدادات Google المضمن مطبقًا.
      - يستخدم `resolveConfigApiKey` hook الخاصة بالموفّر عند كشفها. ويحتوي
        المسار المضمن `amazon-bedrock` أيضًا على محلّل داخلي لعلامات AWS env هنا،
        رغم أن مصادقة runtime الخاصة بـ Bedrock نفسها ما زالت تستخدم سلسلة AWS SDK الافتراضية.
      | 13 | `contributeResolvedModelCompat` | علامات compat لنماذج المورّد خلف ناقل متوافق آخر |
      | 14 | `capabilities` | حقيبة قدرات ثابتة قديمة؛ للتوافق فقط |
      | 15 | `normalizeToolSchemas` | تنظيف مخطط الأدوات الذي يملكه الموفّر قبل التسجيل |
      | 16 | `inspectToolSchemas` | تشخيصات مخطط الأدوات التي يملكها الموفّر |
      | 17 | `resolveReasoningOutputMode` | عقد مخرجات reasoning بين tagged وnative |
      | 18 | `prepareExtraParams` | معاملات الطلب الافتراضية |
      | 19 | `createStreamFn` | ناقل StreamFn مخصص بالكامل |
      | 20 | `wrapStreamFn` | أغلفة ترويسة/جسم مخصصة على مسار stream العادي |
      | 21 | `resolveTransportTurnState` | ترويسات/بيانات وصفية أصلية لكل دور |
      | 22 | `resolveWebSocketSessionPolicy` | ترويسات جلسة WS أصلية/تهدئة |
      | 23 | `formatApiKey` | شكل رمز runtime مخصص |
      | 24 | `refreshOAuth` | تحديث OAuth مخصص |
      | 25 | `buildAuthDoctorHint` | إرشادات إصلاح المصادقة |
      | 26 | `matchesContextOverflowError` | اكتشاف فيضان السياق الذي يملكه الموفّر |
      | 27 | `classifyFailoverReason` | تصنيف تحديد المعدل/التحميل الزائد الذي يملكه الموفّر |
      | 28 | `isCacheTtlEligible` | بوابة TTL لذاكرة التخزين المؤقت للـ prompt |
      | 29 | `buildMissingAuthMessage` | تلميح missing-auth مخصص |
      | 30 | `suppressBuiltInModel` | إخفاء الصفوف القديمة من upstream |
      | 31 | `augmentModelCatalog` | صفوف synthetic للتوافق الأمامي |
      | 32 | `isBinaryThinking` | تشغيل/إيقاف التفكير الثنائي |
      | 33 | `supportsXHighThinking` | دعم reasoning من نوع `xhigh` |
      | 34 | `resolveDefaultThinkingLevel` | سياسة `/think` الافتراضية |
      | 35 | `isModernModelRef` | مطابقة نموذج live/smoke |
      | 36 | `prepareRuntimeAuth` | تبادل الرمز قبل الاستدلال |
      | 37 | `resolveUsageAuth` | تحليل بيانات اعتماد الاستخدام المخصص |
      | 38 | `fetchUsageSnapshot` | نقطة نهاية استخدام مخصصة |
      | 39 | `createEmbeddingProvider` | محول embedding يملكه الموفّر للذاكرة/البحث |
      | 40 | `buildReplayPolicy` | سياسة replay/compaction مخصصة للنص الحواري |
      | 41 | `sanitizeReplayHistory` | إعادات كتابة replay خاصة بالموفّر بعد التنظيف العام |
      | 42 | `validateReplayTurns` | تحقق صارم من أدوار replay قبل embedded runner |
      | 43 | `onModelSelected` | استدعاء لاحق للاختيار (مثل telemetry) |

      ملاحظة ضبط prompt:

      - يتيح `resolveSystemPromptContribution` للموفّر حقن
        إرشادات system-prompt واعية بالكاش لعائلة نموذج. ويفضّل استخدامه بدلًا من
        `before_prompt_build` عندما ينتمي السلوك إلى عائلة موفّر/نموذج واحدة
        ويجب أن يحافظ على تقسيم الكاش الثابت/الديناميكي المستقر.

      للاطلاع على أوصاف مفصلة وأمثلة من العالم الحقيقي، راجع
      [الداخليات: Provider Runtime Hooks](/ar/plugins/architecture#provider-runtime-hooks).
    </Accordion>

  </Step>

  <Step title="إضافة قدرات إضافية (اختياري)">
    <a id="step-5-add-extra-capabilities"></a>
    يمكن لـ provider plugin تسجيل speech، وrealtime transcription، وrealtime
    voice، وفهم الوسائط، وإنشاء الصور، وإنشاء الفيديو، وweb fetch،
    وweb search إلى جانب الاستدلال النصي:

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

    يصنف OpenClaw هذا على أنه plugin **hybrid-capability**. وهذا هو
    النمط الموصى به لإضافات الشركات (plugin واحدة لكل مورّد). راجع
    [الداخليات: ملكية القدرات](/ar/plugins/architecture#capability-ownership-model).

  </Step>

  <Step title="الاختبار">
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

تُنشر Provider Plugins بالطريقة نفسها مثل أي plugin خارجي آخر للشفرة:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

لا تستخدم الاسم المستعار القديم الخاص بالنشر المعتمد على skill هنا؛ إذ يجب أن تستخدم
حزم plugin الأمر `clawhub package publish`.

## بنية الملفات

```
<bundled-plugin-root>/acme-ai/
├── package.json              # بيانات openclaw.providers الوصفية
├── openclaw.plugin.json      # البيان مع providerAuthEnvVars
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # الاختبارات
    └── usage.ts              # نقطة نهاية الاستخدام (اختياري)
```

## مرجع ترتيب الفهرس

يتحكم `catalog.order` في وقت دمج الفهرس الخاص بك نسبةً إلى
الموفّرين المضمنين:

| الترتيب | متى | حالة الاستخدام |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | المرور الأول | موفّرون عاديون بمفتاح API |
| `profile` | بعد simple  | موفّرون محكومون بملفات تعريف المصادقة |
| `paired`  | بعد profile | إنشاء إدخالات متعددة مرتبطة |
| `late`    | المرور الأخير | تجاوز الموفّرين الحاليين (يفوز عند التصادم) |

## الخطوات التالية

- [Channel Plugins](/ar/plugins/sdk-channel-plugins) — إذا كان plugin لديك يوفّر أيضًا channel
- [SDK Runtime](/ar/plugins/sdk-runtime) — مساعدات `api.runtime` ‏(TTS، والبحث، والوكيل الفرعي)
- [نظرة عامة على SDK](/ar/plugins/sdk-overview) — مرجع كامل للاستيراد عبر المسارات الفرعية
- [داخليات Plugin](/ar/plugins/architecture#provider-runtime-hooks) — تفاصيل hook وأمثلة الموفّرين المضمنين
