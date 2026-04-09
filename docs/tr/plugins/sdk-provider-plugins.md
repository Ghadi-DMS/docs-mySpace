---
read_when:
    - Yeni bir model sağlayıcısı eklentisi oluşturuyorsunuz
    - OpenClaw'a OpenAI uyumlu bir proxy veya özel bir LLM eklemek istiyorsunuz
    - Sağlayıcı auth, kataloglar ve çalışma zamanı kancalarını anlamanız gerekiyor
sidebarTitle: Provider Plugins
summary: OpenClaw için bir model sağlayıcısı eklentisi oluşturma adım adım kılavuzu
title: Sağlayıcı Eklentileri Oluşturma
x-i18n:
    generated_at: "2026-04-09T01:31:06Z"
    model: gpt-5.4
    provider: openai
    source_hash: 38d9af522dc19e49c81203a83a4096f01c2398b1df771c848a30ad98f251e9e1
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# Sağlayıcı Eklentileri Oluşturma

Bu kılavuz, OpenClaw'a bir model sağlayıcısı
(LLM) ekleyen bir sağlayıcı eklentisi oluşturmayı adım adım açıklar. Sonunda bir model kataloğu,
API anahtarı auth'u ve dinamik model çözümlemesi olan bir sağlayıcınız olacak.

<Info>
  Daha önce hiç OpenClaw eklentisi oluşturmadıysanız, temel paket
  yapısı ve manifest kurulumu için önce
  [Başlangıç](/tr/plugins/building-plugins) bölümünü okuyun.
</Info>

## Yol gösterimi

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paket ve manifest">
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

    Manifest, OpenClaw'ın eklenti çalışma zamanınızı yüklemeden
    kimlik bilgilerini algılayabilmesi için `providerAuthEnvVars` tanımlar.
    Bir sağlayıcı varyantının başka bir sağlayıcı kimliğinin auth'unu yeniden kullanması gerekiyorsa `providerAuthAliases`
    ekleyin. `modelSupport`
    isteğe bağlıdır ve çalışma zamanı kancaları daha oluşmadan önce OpenClaw'ın `acme-large` gibi
    kısa model kimliklerinden sağlayıcı eklentinizi otomatik yüklemesine olanak tanır.
    Sağlayıcıyı ClawHub üzerinde yayımlarsanız, bu `openclaw.compat` ve `openclaw.build` alanları
    `package.json` içinde zorunludur.

  </Step>

  <Step title="Sağlayıcıyı kaydet">
    En küçük bir sağlayıcı için `id`, `label`, `auth` ve `catalog` gerekir:

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

    Bu çalışan bir sağlayıcıdır. Kullanıcılar artık
    `openclaw onboard --acme-ai-api-key <key>` çalıştırabilir ve model olarak
    `acme-ai/acme-large` seçebilir.

    Yalnızca API anahtarı
    auth'u artı katalog destekli tek bir çalışma zamanı kaydeden paketlenmiş sağlayıcılar için, daha dar kapsamlı
    `defineSingleProviderPluginEntry(...)` yardımcısını tercih edin:

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

    Auth akışınız onboarding sırasında `models.providers.*`, takma adlar ve
    ajanın varsayılan modelini de yamalamak zorundaysa,
    `openclaw/plugin-sdk/provider-onboard` içindeki hazır yardımcılardan yararlanın. En dar kapsamlı yardımcılar
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` ve
    `createModelCatalogPresetAppliers(...)` biçimindedir.

    Bir sağlayıcının yerel uç noktası normal
    `openai-completions` taşımasında akışlı kullanım bloklarını destekliyorsa, sağlayıcı kimliği denetimlerini sabit kodlamak yerine
    `openclaw/plugin-sdk/provider-catalog-shared` içindeki paylaşılan katalog yardımcılarını tercih edin.
    `supportsNativeStreamingUsageCompat(...)` ve
    `applyProviderNativeStreamingUsageCompat(...)`, uç nokta yetenek haritasından desteği algılar;
    böylece yerel Moonshot/DashScope tarzı uç noktalar, eklenti özel bir sağlayıcı kimliği kullandığında bile
    katılım gösterebilir.

  </Step>

  <Step title="Dinamik model çözümlemesi ekle">
    Sağlayıcınız keyfi model kimliklerini kabul ediyorsa (bir proxy veya yönlendirici gibi),
    `resolveDynamicModel` ekleyin:

    ```typescript
    api.registerProvider({
      // ... yukarıdaki id, label, auth, catalog

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

    Çözümleme bir ağ çağrısı gerektiriyorsa, eşzamanlı olmayan ön ısınma için `prepareDynamicModel` kullanın —
    tamamlandıktan sonra `resolveDynamicModel` yeniden çalıştırılır.

  </Step>

  <Step title="Çalışma zamanı kancalarını ekle (gerektikçe)">
    Çoğu sağlayıcı yalnızca `catalog` + `resolveDynamicModel` gerektirir. Sağlayıcınız ihtiyaç duydukça
    kancaları aşamalı olarak ekleyin.

    Paylaşılan yardımcı oluşturucular artık en yaygın replay/araç uyumluluğu
    ailelerini kapsıyor; bu yüzden eklentilerin genellikle her kancayı tek tek elle bağlaması gerekmez:

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

    Günümüzde mevcut replay aileleri:

    | Aile | Bağladığı şey |
    | --- | --- |
    | `openai-compatible` | OpenAI uyumlu taşımalar için paylaşılan OpenAI tarzı replay politikası; buna araç çağrısı kimliği temizleme, assistant-first sıralama düzeltmeleri ve taşımanın buna ihtiyaç duyduğu durumlarda genel Gemini dönüş doğrulaması dahildir |
    | `anthropic-by-model` | `modelId` temelinde seçilen Claude farkındalıklı replay politikası; böylece Anthropic-message taşımaları yalnızca çözümlenen model gerçekten bir Claude kimliğiyse Claude'a özgü thinking-block temizliğini alır |
    | `google-gemini` | Yerel Gemini replay politikası artı önyükleme replay temizliği ve etiketli akıl yürütme çıktısı modu |
    | `passthrough-gemini` | OpenAI uyumlu proxy taşımaları üzerinden çalışan Gemini modelleri için Gemini thought-signature temizliği; yerel Gemini replay doğrulamasını veya önyükleme yeniden yazımlarını etkinleştirmez |
    | `hybrid-anthropic-openai` | Tek bir eklenti içinde Anthropic-message ve OpenAI uyumlu model yüzeylerini karıştıran sağlayıcılar için hibrit politika; isteğe bağlı yalnızca Claude thinking-block bırakma davranışı Anthropic tarafıyla sınırlı kalır |

    Gerçek paketlenmiş örnekler:

    - `google` ve `google-gemini-cli`: `google-gemini`
    - `openrouter`, `kilocode`, `opencode` ve `opencode-go`: `passthrough-gemini`
    - `amazon-bedrock` ve `anthropic-vertex`: `anthropic-by-model`
    - `minimax`: `hybrid-anthropic-openai`
    - `moonshot`, `ollama`, `xai` ve `zai`: `openai-compatible`

    Günümüzde mevcut akış aileleri:

    | Aile | Bağladığı şey |
    | --- | --- |
    | `google-thinking` | Paylaşılan akış yolunda Gemini düşünme yükü normalleştirmesi |
    | `kilocode-thinking` | Paylaşılan proxy akış yolunda Kilo akıl yürütme sarmalayıcısı; `kilo/auto` ve desteklenmeyen proxy akıl yürütme kimlikleri enjekte edilen düşünmeyi atlar |
    | `moonshot-thinking` | Yapılandırma + `/think` düzeyinden Moonshot ikili yerel düşünme yükü eşlemesi |
    | `minimax-fast-mode` | Paylaşılan akış yolunda MiniMax fast-mode model yeniden yazımı |
    | `openai-responses-defaults` | Paylaşılan yerel OpenAI/Codex Responses sarmalayıcıları: atıf başlıkları, `/fast`/`serviceTier`, metin ayrıntı düzeyi, yerel Codex web araması, akıl yürütme uyumluluğu yük şekillendirmesi ve Responses bağlam yönetimi |
    | `openrouter-thinking` | Proxy rotaları için OpenRouter akıl yürütme sarmalayıcısı; desteklenmeyen model/`auto` atlamaları merkezi olarak ele alınır |
    | `tool-stream-default-on` | Z.AI gibi açıkça devre dışı bırakılmadıkça araç akışını isteyen sağlayıcılar için varsayılan olarak açık `tool_stream` sarmalayıcısı |

    Gerçek paketlenmiş örnekler:

    - `google` ve `google-gemini-cli`: `google-thinking`
    - `kilocode`: `kilocode-thinking`
    - `moonshot`: `moonshot-thinking`
    - `minimax` ve `minimax-portal`: `minimax-fast-mode`
    - `openai` ve `openai-codex`: `openai-responses-defaults`
    - `openrouter`: `openrouter-thinking`
    - `zai`: `tool-stream-default-on`

    `openclaw/plugin-sdk/provider-model-shared`, replay-family
    enum'unu ve bu ailelerin üzerine kurulduğu paylaşılan yardımcıları da dışa aktarır. Sık kullanılan genel
    dışa aktarımlar şunlardır:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - `buildOpenAICompatibleReplayPolicy(...)`,
      `buildAnthropicReplayPolicyForModel(...)`,
      `buildGoogleGeminiReplayPolicy(...)` ve
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)` gibi paylaşılan replay oluşturucuları
    - `sanitizeGoogleGeminiReplayHistory(...)`
      ve `resolveTaggedReasoningOutputMode()` gibi Gemini replay yardımcıları
    - `resolveProviderEndpoint(...)`,
      `normalizeProviderId(...)`, `normalizeGooglePreviewModelId(...)` ve
      `normalizeNativeXaiModelId(...)` gibi uç nokta/model yardımcıları

    `openclaw/plugin-sdk/provider-stream`, hem aile oluşturucuyu hem de bu ailelerin yeniden kullandığı
    genel sarmalayıcı yardımcılarını sunar. Sık kullanılan genel dışa aktarımlar
    şunlardır:

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - `createOpenAIAttributionHeadersWrapper(...)`,
      `createOpenAIFastModeWrapper(...)`,
      `createOpenAIServiceTierWrapper(...)`,
      `createOpenAIResponsesContextManagementWrapper(...)` ve
      `createCodexNativeWebSearchWrapper(...)` gibi paylaşılan OpenAI/Codex sarmalayıcıları
    - `createOpenRouterWrapper(...)`,
      `createToolStreamWrapper(...)` ve `createMinimaxFastModeWrapper(...)` gibi paylaşılan proxy/sağlayıcı sarmalayıcıları

    Bazı akış yardımcıları bilerek sağlayıcıya yerel tutulur. Mevcut paketlenmiş
    örnek: `@openclaw/anthropic-provider`
    `api.ts` / `contract-api.ts` yüzeyinden
    `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` ve alt düzey Anthropic sarmalayıcı
    oluşturucularını dışa aktarır. Bu yardımcılar Anthropic'e özgü kalır çünkü
    Claude OAuth beta işleme ve `context1m` kapılamasını da kodlarlar.

    Diğer paketlenmiş sağlayıcılar da, davranış
    aileler arasında temiz biçimde paylaşılamadığında taşımaya özgü sarmalayıcıları yerel tutar. Güncel örnek:
    paketlenmiş xAI eklentisi, `/fast` takma ad yeniden yazımları, varsayılan `tool_stream`,
    desteklenmeyen strict-tool temizliği ve xAI'ya özgü akıl yürütme yükü
    kaldırma dahil olmak üzere yerel xAI Responses şekillendirmesini kendi
    `wrapStreamFn` içinde tutar.

    `openclaw/plugin-sdk/provider-tools`, şu anda bir paylaşılan
    araç şeması ailesi ile paylaşılan şema/uyumluluk yardımcılarını sunar:

    - `ProviderToolCompatFamily`, günümüzdeki paylaşılan aile envanterini belgeler.
    - `buildProviderToolCompatFamilyHooks("gemini")`, Gemini güvenli araç şemalarına ihtiyaç duyan sağlayıcılar için Gemini şema
      temizliği + tanılamayı bağlar.
    - `normalizeGeminiToolSchemas(...)` ve `inspectGeminiToolSchemas(...)`
      temel genel Gemini şema yardımcılarıdır.
    - `resolveXaiModelCompatPatch()`, paketlenmiş xAI uyumluluk yamasını döndürür:
      `toolSchemaProfile: "xai"`, desteklenmeyen şema anahtar sözcükleri, yerel
      `web_search` desteği ve HTML entity araç çağrısı argümanı çözme.
    - `applyXaiModelCompat(model)`, aynı xAI uyumluluk yamasını
      çözümlenen bir modele çalıştırıcıya ulaşmadan önce uygular.

    Gerçek paketlenmiş örnek: xAI eklentisi, bu uyumluluk meta verisinin
    sağlayıcıya ait kalması için `normalizeResolvedModel` ile
    `contributeResolvedModelCompat` kullanır; böylece çekirdekte xAI kuralları sabit kodlanmaz.

    Aynı paket kökü deseni diğer paketlenmiş sağlayıcıların da temelini oluşturur:

    - `@openclaw/openai-provider`: `api.ts`, sağlayıcı oluşturucularını,
      varsayılan model yardımcılarını ve gerçek zamanlı sağlayıcı oluşturucularını dışa aktarır
    - `@openclaw/openrouter-provider`: `api.ts`, sağlayıcı oluşturucuyu
      artı onboarding/yapılandırma yardımcılarını dışa aktarır

    <Tabs>
      <Tab title="Belirteç değişimi">
        Her çıkarım çağrısından önce belirteç değişimi gerektiren sağlayıcılar için:

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
      <Tab title="Özel başlıklar">
        Özel istek başlıkları veya gövde değişiklikleri gerektiren sağlayıcılar için:

        ```typescript
        // wrapStreamFn, ctx.streamFn'den türetilmiş bir StreamFn döndürür
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
      <Tab title="Yerel taşıma kimliği">
        Genel HTTP veya WebSocket taşımalarında yerel istek/oturum başlıkları ya da meta veri gerektiren sağlayıcılar için:

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
      <Tab title="Kullanım ve faturalama">
        Kullanım/faturalama verilerini açığa çıkaran sağlayıcılar için:

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

    <Accordion title="Kullanılabilir tüm sağlayıcı kancaları">
      OpenClaw, kancaları şu sırada çağırır. Çoğu sağlayıcı yalnızca 2-3 tanesini kullanır:

      | # | Kanca | Ne zaman kullanılmalı |
      | --- | --- | --- |
      | 1 | `catalog` | Model kataloğu veya temel URL varsayılanları |
      | 2 | `applyConfigDefaults` | Yapılandırma somutlaştırması sırasında sağlayıcıya ait genel varsayılanlar |
      | 3 | `normalizeModelId` | Aramadan önce eski/önizleme model kimliği takma adlarını temizleme |
      | 4 | `normalizeTransport` | Genel model derlemesinden önce sağlayıcı ailesi `api` / `baseUrl` temizliği |
      | 5 | `normalizeConfig` | `models.providers.<id>` yapılandırmasını normalize et |
      | 6 | `applyNativeStreamingUsageCompat` | Yapılandırma sağlayıcıları için yerel akış-kullanımı uyumluluk yeniden yazımları |
      | 7 | `resolveConfigApiKey` | Sağlayıcıya ait env işaretleyici auth çözümlemesi |
      | 8 | `resolveSyntheticAuth` | Yerel/kendi barındırılan veya yapılandırma destekli sentetik auth |
      | 9 | `shouldDeferSyntheticProfileAuth` | Sentetik saklanan profil yer tutucularını env/yapılandırma auth'unun arkasına düşür |
      | 10 | `resolveDynamicModel` | Keyfi yukarı akış model kimliklerini kabul et |
      | 11 | `prepareDynamicModel` | Çözümlemeden önce eşzamanlı olmayan meta veri getirme |
      | 12 | `normalizeResolvedModel` | Çalıştırıcıdan önce taşıma yeniden yazımları |

    Çalışma zamanı yedek notları:

    - `normalizeConfig`, önce eşleşen sağlayıcıyı, ardından
      yalnızca yapılandırmayı gerçekten değiştiren bir tane bulunana kadar diğer
      kanca destekli sağlayıcı eklentilerini denetler.
      Hiçbir sağlayıcı kancası desteklenen Google ailesi yapılandırma girdisini yeniden yazmazsa,
      paketlenmiş Google yapılandırma normalleştiricisi yine de uygulanır.
    - `resolveConfigApiKey`, sunulduğunda sağlayıcı kancasını kullanır. Paketlenmiş
      `amazon-bedrock` yolu da burada yerleşik bir AWS env işaretleyici çözücüye sahiptir,
      her ne kadar Bedrock çalışma zamanı auth'unun kendisi hâlâ AWS SDK varsayılan
      zincirini kullansa da.
      | 13 | `contributeResolvedModelCompat` | Başka bir uyumlu taşımanın arkasındaki üretici modelleri için uyumluluk işaretleri |
      | 14 | `capabilities` | Eski statik yetenek paketi; yalnızca uyumluluk için |
      | 15 | `normalizeToolSchemas` | Kayıttan önce sağlayıcıya ait araç şeması temizliği |
      | 16 | `inspectToolSchemas` | Sağlayıcıya ait araç şeması tanılaması |
      | 17 | `resolveReasoningOutputMode` | Etiketli ve yerel akıl yürütme çıktısı sözleşmesi |
      | 18 | `prepareExtraParams` | Varsayılan istek parametreleri |
      | 19 | `createStreamFn` | Tamamen özel StreamFn taşıması |
      | 20 | `wrapStreamFn` | Normal akış yolunda özel başlık/gövde sarmalayıcıları |
      | 21 | `resolveTransportTurnState` | Yerel tur başına başlıklar/meta veri |
      | 22 | `resolveWebSocketSessionPolicy` | Yerel WS oturum başlıkları/bekleme süresi |
      | 23 | `formatApiKey` | Özel çalışma zamanı belirteç şekli |
      | 24 | `refreshOAuth` | Özel OAuth yenileme |
      | 25 | `buildAuthDoctorHint` | Auth onarım rehberliği |
      | 26 | `matchesContextOverflowError` | Sağlayıcıya ait taşma algılama |
      | 27 | `classifyFailoverReason` | Sağlayıcıya ait oran sınırı/aşırı yük sınıflandırması |
      | 28 | `isCacheTtlEligible` | İstem önbelleği TTL kapılaması |
      | 29 | `buildMissingAuthMessage` | Özel eksik auth ipucu |
      | 30 | `suppressBuiltInModel` | Eski yukarı akış satırlarını gizle |
      | 31 | `augmentModelCatalog` | Sentetik ileri uyumluluk satırları |
      | 32 | `isBinaryThinking` | İkili düşünme aç/kapat |
      | 33 | `supportsXHighThinking` | `xhigh` akıl yürütme desteği |
      | 34 | `resolveDefaultThinkingLevel` | Varsayılan `/think` politikası |
      | 35 | `isModernModelRef` | Canlı/smoke model eşleştirmesi |
      | 36 | `prepareRuntimeAuth` | Çıkarımdan önce belirteç değişimi |
      | 37 | `resolveUsageAuth` | Özel kullanım kimlik bilgisi ayrıştırma |
      | 38 | `fetchUsageSnapshot` | Özel kullanım uç noktası |
      | 39 | `createEmbeddingProvider` | Bellek/arama için sağlayıcıya ait embedding bağdaştırıcısı |
      | 40 | `buildReplayPolicy` | Özel transkript replay/sıkıştırma politikası |
      | 41 | `sanitizeReplayHistory` | Genel temizlikten sonra sağlayıcıya özgü replay yeniden yazımları |
      | 42 | `validateReplayTurns` | Gömülü çalıştırıcıdan önce katı replay dönüş doğrulaması |
      | 43 | `onModelSelected` | Seçim sonrası geri çağırım (ör. telemetri) |

      Prompt ayarlama notu:

      - `resolveSystemPromptContribution`, bir sağlayıcının
        bir model ailesi için önbellek farkındalıklı sistem prompt rehberliği enjekte etmesine olanak tanır.
        Davranış tek bir sağlayıcı/model
        ailesine aitse ve kararlı/dinamik önbellek ayrımını koruması gerekiyorsa bunu
        `before_prompt_build` yerine tercih edin.

      Ayrıntılı açıklamalar ve gerçek dünya örnekleri için
      [İç Yapı: Sağlayıcı Çalışma Zamanı Kancaları](/tr/plugins/architecture#provider-runtime-hooks) bölümüne bakın.
    </Accordion>

  </Step>

  <Step title="Ek yetenekler ekle (isteğe bağlı)">
    <a id="step-5-add-extra-capabilities"></a>
    Bir sağlayıcı eklentisi, metin çıkarımının yanında konuşma, gerçek zamanlı transkripsiyon, gerçek zamanlı
    ses, medya anlama, görüntü oluşturma, video oluşturma, web getirme
    ve web araması kaydedebilir:

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

    OpenClaw bunu bir **hybrid-capability** eklentisi olarak sınıflandırır. Bu,
    şirket eklentileri için önerilen desendir (üretici başına bir eklenti). Bkz.
    [İç Yapı: Yetenek Sahipliği](/tr/plugins/architecture#capability-ownership-model).

    Video oluşturma için yukarıda gösterilen mod farkındalıklı yetenek biçimini tercih edin:
    `generate`, `imageToVideo` ve `videoToVideo`. Düz toplu alanlar,
    örneğin `maxInputImages`, `maxInputVideos` ve `maxDurationSeconds`,
    dönüşüm modu desteğini veya devre dışı bırakılmış modları temiz biçimde duyurmak için
    yeterli değildir.

    Müzik oluşturma sağlayıcıları da aynı deseni izlemelidir:
    yalnızca prompt tabanlı oluşturma için `generate`, referans görsel tabanlı
    oluşturma için `edit`. `maxInputImages`,
    `supportsLyrics` ve `supportsFormat` gibi düz toplu alanlar, düzenleme
    desteğini duyurmak için yeterli değildir; beklenen sözleşme açık
    `generate` / `edit` bloklarıdır.

  </Step>

  <Step title="Test et">
    <a id="step-6-test"></a>
    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // Sağlayıcı yapılandırma nesnenizi index.ts veya özel bir dosyadan dışa aktarın
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

## ClawHub'a yayımla

Sağlayıcı eklentileri, diğer tüm harici kod eklentileriyle aynı şekilde yayımlanır:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

Burada eski yalnızca-skill yayımlama takma adını kullanmayın; eklenti paketleri
`clawhub package publish` kullanmalıdır.

## Dosya yapısı

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers meta verisi
├── openclaw.plugin.json      # Sağlayıcı auth meta verili manifest
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Testler
    └── usage.ts              # Kullanım uç noktası (isteğe bağlı)
```

## Katalog sırası başvurusu

`catalog.order`, kataloğunuzun yerleşik
sağlayıcılara göre ne zaman birleştirileceğini denetler:

| Sıra     | Ne zaman      | Kullanım durumu                                |
| --------- | ------------- | ---------------------------------------------- |
| `simple`  | İlk geçiş     | Düz API anahtarı sağlayıcıları                 |
| `profile` | `simple` sonrası | Auth profilleriyle kapılanan sağlayıcılar   |
| `paired`  | `profile` sonrası | Birden çok ilişkili girdiyi sentezleme     |
| `late`    | Son geçiş     | Mevcut sağlayıcıları geçersiz kılma (çakışmada kazanır) |

## Sonraki adımlar

- [Channel Plugins](/tr/plugins/sdk-channel-plugins) — eklentiniz bir kanal da sağlıyorsa
- [SDK Runtime](/tr/plugins/sdk-runtime) — `api.runtime` yardımcıları (TTS, arama, alt ajan)
- [SDK Overview](/tr/plugins/sdk-overview) — tam alt yol içe aktarma başvurusu
- [Plugin Internals](/tr/plugins/architecture#provider-runtime-hooks) — kanca ayrıntıları ve paketlenmiş örnekler
