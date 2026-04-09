---
read_when:
    - 새 모델 제공자 플러그인을 구축하고 있습니다
    - OpenAI 호환 프록시나 커스텀 LLM을 OpenClaw에 추가하려고 합니다
    - 제공자 인증, 카탈로그, 런타임 hook을 이해해야 합니다
sidebarTitle: Provider Plugins
summary: OpenClaw용 모델 제공자 플러그인을 구축하는 단계별 가이드
title: 제공자 플러그인 구축하기
x-i18n:
    generated_at: "2026-04-09T01:31:20Z"
    model: gpt-5.4
    provider: openai
    source_hash: 38d9af522dc19e49c81203a83a4096f01c2398b1df771c848a30ad98f251e9e1
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# 제공자 플러그인 구축하기

이 가이드는 OpenClaw에 모델 제공자
(LLM)를 추가하는 제공자 플러그인을 구축하는 과정을 설명합니다. 끝까지 따라가면 모델 카탈로그,
API 키 인증, 동적 모델 해석을 갖춘 제공자를 만들 수 있습니다.

<Info>
  아직 OpenClaw 플러그인을 한 번도 만들어 본 적이 없다면,
  먼저 [시작하기](/ko/plugins/building-plugins)를 읽고 기본 패키지
  구조와 매니페스트 설정을 확인하세요.
</Info>

## 따라 하기

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="패키지와 매니페스트">
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

    매니페스트는 `providerAuthEnvVars`를 선언하므로 OpenClaw가
    플러그인 런타임을 로드하지 않고도 자격 증명을 감지할 수 있습니다. 제공자 변형이 다른 제공자 id의 인증을 재사용해야 하는 경우
    `providerAuthAliases`를 추가하세요. `modelSupport`는 선택 사항이며, 런타임 hook이 생기기 전에
    `acme-large` 같은 축약형 모델 id에서 OpenClaw가 제공자 플러그인을 자동 로드하도록 해줍니다.
    제공자를 ClawHub에 게시하는 경우 `package.json`의
    `openclaw.compat` 및 `openclaw.build` 필드는 필수입니다.

  </Step>

  <Step title="제공자 등록">
    최소한의 제공자에는 `id`, `label`, `auth`, `catalog`가 필요합니다:

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

    이것으로 동작하는 제공자가 완성됩니다. 이제 사용자는
    `openclaw onboard --acme-ai-api-key <key>`를 실행하고
    모델로 `acme-ai/acme-large`를 선택할 수 있습니다.

    API 키 인증이 있는 텍스트 제공자 하나와 카탈로그 기반 런타임 하나만 등록하는 번들 제공자의 경우,
    더 좁은 범위의
    `defineSingleProviderPluginEntry(...)` 헬퍼를 사용하는 것이 좋습니다:

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

    인증 흐름이 온보딩 중 `models.providers.*`, 별칭, 에이전트 기본 모델도 함께 수정해야 한다면
    `openclaw/plugin-sdk/provider-onboard`의 프리셋 헬퍼를 사용하세요. 가장 범위가 좁은 헬퍼는
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)`, 그리고
    `createModelCatalogPresetAppliers(...)`입니다.

    제공자의 네이티브 엔드포인트가 일반 `openai-completions` 전송에서 스트리밍된 사용량 블록을 지원한다면,
    제공자 id 검사를 하드코딩하는 대신
    `openclaw/plugin-sdk/provider-catalog-shared`의 공유 카탈로그 헬퍼를 사용하는 것이 좋습니다.
    `supportsNativeStreamingUsageCompat(...)`와
    `applyProviderNativeStreamingUsageCompat(...)`는 엔드포인트 capability 맵에서 지원 여부를 감지하므로,
    플러그인이 커스텀 제공자 id를 사용하더라도 네이티브 Moonshot/DashScope 스타일 엔드포인트를 계속
    opt in할 수 있습니다.

  </Step>

  <Step title="동적 모델 해석 추가">
    제공자가 프록시나 라우터처럼 임의의 모델 ID를 허용한다면
    `resolveDynamicModel`을 추가하세요:

    ```typescript
    api.registerProvider({
      // ... 위의 id, label, auth, catalog

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

    해석에 네트워크 호출이 필요하다면 비동기 워밍업을 위해
    `prepareDynamicModel`을 사용하세요. 완료되면 `resolveDynamicModel`이 다시 실행됩니다.

  </Step>

  <Step title="런타임 hook 추가(필요한 경우)">
    대부분의 제공자는 `catalog` + `resolveDynamicModel`만 있으면 됩니다. 제공자에 필요해질 때만
    점진적으로 hook을 추가하세요.

    이제 공유 헬퍼 빌더가 가장 일반적인 replay/tool-compat
    계열을 다루므로, 보통 플러그인에서 각 hook을 하나씩 직접 연결할 필요가 없습니다:

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

    현재 사용 가능한 replay 계열:

    | Family | 연결되는 항목 |
    | --- | --- |
    | `openai-compatible` | 도구 호출 ID 정리, assistant-first 순서 수정, 전송에 필요할 때의 일반 Gemini 턴 검증을 포함한 공유 OpenAI 스타일 replay 정책 |
    | `anthropic-by-model` | `modelId`로 선택되는 Claude 인식 replay 정책으로, Anthropic-message 전송이 실제 해석된 모델이 Claude id일 때만 Claude 전용 thinking 블록 정리를 받도록 함 |
    | `google-gemini` | 네이티브 Gemini replay 정책, bootstrap replay 정리, 태그형 추론 출력 모드 |
    | `passthrough-gemini` | OpenAI 호환 프록시 전송을 통해 실행되는 Gemini 모델의 Gemini thought-signature 정리; 네이티브 Gemini replay 검증이나 bootstrap 재작성을 활성화하지 않음 |
    | `hybrid-anthropic-openai` | 하나의 플러그인 안에서 Anthropic-message와 OpenAI 호환 모델 표면을 혼합하는 제공자를 위한 하이브리드 정책; 선택적인 Claude 전용 thinking 블록 제거는 Anthropic 쪽에만 한정됨 |

    실제 번들 예시:

    - `google` 및 `google-gemini-cli`: `google-gemini`
    - `openrouter`, `kilocode`, `opencode`, `opencode-go`: `passthrough-gemini`
    - `amazon-bedrock` 및 `anthropic-vertex`: `anthropic-by-model`
    - `minimax`: `hybrid-anthropic-openai`
    - `moonshot`, `ollama`, `xai`, `zai`: `openai-compatible`

    현재 사용 가능한 stream 계열:

    | Family | 연결되는 항목 |
    | --- | --- |
    | `google-thinking` | 공유 stream 경로에서 Gemini thinking 페이로드 정규화 |
    | `kilocode-thinking` | 공유 프록시 stream 경로에서 Kilo 추론 래퍼를 적용하며, `kilo/auto`와 지원되지 않는 프록시 추론 id는 주입된 thinking을 건너뜀 |
    | `moonshot-thinking` | config + `/think` 수준에서 Moonshot 이진 네이티브 thinking 페이로드 매핑 |
    | `minimax-fast-mode` | 공유 stream 경로에서 MiniMax fast-mode 모델 재작성 |
    | `openai-responses-defaults` | 공유 네이티브 OpenAI/Codex Responses 래퍼: attribution 헤더, `/fast`/`serviceTier`, 텍스트 verbosity, 네이티브 Codex 웹 검색, 추론 호환 페이로드 형성, Responses 컨텍스트 관리 |
    | `openrouter-thinking` | 프록시 경로를 위한 OpenRouter 추론 래퍼로, 지원되지 않는 모델/`auto` 건너뛰기를 중앙에서 처리 |
    | `tool-stream-default-on` | Z.AI 같은 제공자를 위한 기본 활성화 `tool_stream` 래퍼로, 명시적으로 비활성화되지 않는 한 도구 스트리밍을 사용함 |

    실제 번들 예시:

    - `google` 및 `google-gemini-cli`: `google-thinking`
    - `kilocode`: `kilocode-thinking`
    - `moonshot`: `moonshot-thinking`
    - `minimax` 및 `minimax-portal`: `minimax-fast-mode`
    - `openai` 및 `openai-codex`: `openai-responses-defaults`
    - `openrouter`: `openrouter-thinking`
    - `zai`: `tool-stream-default-on`

    `openclaw/plugin-sdk/provider-model-shared`는 replay-family
    enum과 해당 계열이 기반으로 하는 공유 헬퍼도 내보냅니다. 일반적인 공개 export에는 다음이 포함됩니다:

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - `buildOpenAICompatibleReplayPolicy(...)`,
      `buildAnthropicReplayPolicyForModel(...)`,
      `buildGoogleGeminiReplayPolicy(...)`, 그리고
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)` 같은 공유 replay 빌더
    - `sanitizeGoogleGeminiReplayHistory(...)`
      및 `resolveTaggedReasoningOutputMode()` 같은 Gemini replay 헬퍼
    - `resolveProviderEndpoint(...)`,
      `normalizeProviderId(...)`, `normalizeGooglePreviewModelId(...)`, 그리고
      `normalizeNativeXaiModelId(...)` 같은 엔드포인트/모델 헬퍼

    `openclaw/plugin-sdk/provider-stream`은 family 빌더와
    해당 family가 재사용하는 공개 래퍼 헬퍼를 모두 제공합니다. 일반적인 공개 export에는 다음이 포함됩니다:

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - 다음과 같은 공유 OpenAI/Codex 래퍼:
      `createOpenAIAttributionHeadersWrapper(...)`,
      `createOpenAIFastModeWrapper(...)`,
      `createOpenAIServiceTierWrapper(...)`,
      `createOpenAIResponsesContextManagementWrapper(...)`, 그리고
      `createCodexNativeWebSearchWrapper(...)`
    - `createOpenRouterWrapper(...)`,
      `createToolStreamWrapper(...)`, `createMinimaxFastModeWrapper(...)` 같은
      공유 프록시/제공자 래퍼

    일부 stream 헬퍼는 의도적으로 provider-local로 유지됩니다. 현재 번들
    예시: `@openclaw/anthropic-provider`는
    공개 `api.ts` /
    `contract-api.ts` 경계에서 `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, 그리고
    더 낮은 수준의 Anthropic 래퍼 빌더를 export합니다. 이 헬퍼들은
    Claude OAuth 베타 처리와 `context1m` 게이팅도 함께 인코딩하므로
    Anthropic 전용으로 유지됩니다.

    다른 번들 제공자들도 동작이 계열 전체에 깔끔하게 공유되지 않는 경우
    전송 전용 래퍼를 로컬에 유지합니다. 현재 예시: 번들 xAI 플러그인은
    네이티브 xAI Responses 형성을 자체
    `wrapStreamFn` 안에 유지하며, 여기에는 `/fast` 별칭 재작성, 기본 `tool_stream`,
    지원되지 않는 strict-tool 정리, xAI 전용 추론 페이로드
    제거가 포함됩니다.

    `openclaw/plugin-sdk/provider-tools`는 현재 하나의 공유
    tool-schema 계열과 공유 스키마/호환성 헬퍼를 노출합니다:

    - `ProviderToolCompatFamily`는 현재의 공유 계열 목록을 문서화합니다.
    - `buildProviderToolCompatFamilyHooks("gemini")`는 Gemini 안전 도구 스키마가 필요한 제공자를 위해
      Gemini 스키마 정리 + 진단을 연결합니다.
    - `normalizeGeminiToolSchemas(...)`와 `inspectGeminiToolSchemas(...)`는
      그 기반이 되는 공개 Gemini 스키마 헬퍼입니다.
    - `resolveXaiModelCompatPatch()`는 번들 xAI 호환성 패치를 반환합니다:
      `toolSchemaProfile: "xai"`, 지원되지 않는 스키마 키워드, 네이티브
      `web_search` 지원, HTML 엔터티 도구 호출 인자 디코딩.
    - `applyXaiModelCompat(model)`은 실행기에 도달하기 전에
      동일한 xAI 호환성 패치를 해석된 모델에 적용합니다.

    실제 번들 예시: xAI 플러그인은
    코어에 xAI 규칙을 하드코딩하는 대신
    `normalizeResolvedModel`과 `contributeResolvedModelCompat`를 사용해 해당 호환성 메타데이터의 소유권을 제공자에 둡니다.

    동일한 패키지 루트 패턴은 다른 번들 제공자에도 적용됩니다:

    - `@openclaw/openai-provider`: `api.ts`가 제공자 빌더,
      기본 모델 헬퍼, realtime 제공자 빌더를 export
    - `@openclaw/openrouter-provider`: `api.ts`가 제공자 빌더와
      온보딩/config 헬퍼를 export

    <Tabs>
      <Tab title="토큰 교환">
        각 추론 호출 전에 토큰 교환이 필요한 제공자의 경우:

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
      <Tab title="커스텀 헤더">
        커스텀 요청 헤더나 본문 수정이 필요한 제공자의 경우:

        ```typescript
        // wrapStreamFn은 ctx.streamFn에서 파생된 StreamFn을 반환합니다
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
      <Tab title="네이티브 전송 ID">
        일반 HTTP 또는 WebSocket 전송에서 네이티브 요청/세션 헤더나 메타데이터가 필요한 제공자의 경우:

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
      <Tab title="사용량 및 과금">
        사용량/과금 데이터를 노출하는 제공자의 경우:

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

    <Accordion title="사용 가능한 모든 제공자 hook">
      OpenClaw는 다음 순서로 hook을 호출합니다. 대부분의 제공자는 2~3개만 사용합니다:

      | # | Hook | 사용 시점 |
      | --- | --- | --- |
      | 1 | `catalog` | 모델 카탈로그 또는 base URL 기본값 |
      | 2 | `applyConfigDefaults` | config materialization 중 제공자 소유 전역 기본값 |
      | 3 | `normalizeModelId` | 조회 전 레거시/프리뷰 모델 id 별칭 정리 |
      | 4 | `normalizeTransport` | 일반 모델 조립 전 제공자 계열 `api` / `baseUrl` 정리 |
      | 5 | `normalizeConfig` | `models.providers.<id>` config 정규화 |
      | 6 | `applyNativeStreamingUsageCompat` | config 제공자용 네이티브 스트리밍 사용량 호환성 재작성 |
      | 7 | `resolveConfigApiKey` | 제공자 소유 env-marker 인증 해석 |
      | 8 | `resolveSyntheticAuth` | 로컬/셀프호스트형 또는 config 기반 synthetic 인증 |
      | 9 | `shouldDeferSyntheticProfileAuth` | synthetic 저장 프로필 플레이스홀더를 env/config 인증보다 뒤로 낮춤 |
      | 10 | `resolveDynamicModel` | 임의의 업스트림 모델 ID 허용 |
      | 11 | `prepareDynamicModel` | 해석 전 비동기 메타데이터 조회 |
      | 12 | `normalizeResolvedModel` | 실행기 이전의 전송 재작성 |

    런타임 폴백 참고:

    - `normalizeConfig`는 먼저 일치한 제공자를 확인한 다음,
      실제로 config를 변경할 때까지 다른 hook 지원 제공자 플러그인을 확인합니다.
      지원되는 Google 계열 config 항목을 어떤 제공자 hook도 재작성하지 않으면
      번들된 Google config 정규화기가 계속 적용됩니다.
    - `resolveConfigApiKey`는 노출된 경우 제공자 hook을 사용합니다. 번들된
      `amazon-bedrock` 경로도 여기에서 내장 AWS env-marker 해석기를 가지지만,
      Bedrock 런타임 인증 자체는 여전히 AWS SDK 기본 체인을 사용합니다.
      | 13 | `contributeResolvedModelCompat` | 다른 호환 전송 뒤에 있는 공급업체 모델용 호환성 플래그 |
      | 14 | `capabilities` | 레거시 정적 capability bag; 호환성 전용 |
      | 15 | `normalizeToolSchemas` | 등록 전 제공자 소유 도구 스키마 정리 |
      | 16 | `inspectToolSchemas` | 제공자 소유 도구 스키마 진단 |
      | 17 | `resolveReasoningOutputMode` | 태그형 대 네이티브 추론 출력 계약 |
      | 18 | `prepareExtraParams` | 기본 요청 파라미터 |
      | 19 | `createStreamFn` | 완전히 커스텀 StreamFn 전송 |
      | 20 | `wrapStreamFn` | 일반 stream 경로의 커스텀 헤더/본문 래퍼 |
      | 21 | `resolveTransportTurnState` | 네이티브 턴별 헤더/메타데이터 |
      | 22 | `resolveWebSocketSessionPolicy` | 네이티브 WS 세션 헤더/쿨다운 |
      | 23 | `formatApiKey` | 커스텀 런타임 토큰 형태 |
      | 24 | `refreshOAuth` | 커스텀 OAuth 갱신 |
      | 25 | `buildAuthDoctorHint` | 인증 복구 안내 |
      | 26 | `matchesContextOverflowError` | 제공자 소유 오버플로 감지 |
      | 27 | `classifyFailoverReason` | 제공자 소유 rate-limit/overload 분류 |
      | 28 | `isCacheTtlEligible` | 프롬프트 캐시 TTL 게이팅 |
      | 29 | `buildMissingAuthMessage` | 커스텀 누락 인증 힌트 |
      | 30 | `suppressBuiltInModel` | 오래된 업스트림 행 숨기기 |
      | 31 | `augmentModelCatalog` | synthetic 순방향 호환 행 |
      | 32 | `isBinaryThinking` | 이진 thinking on/off |
      | 33 | `supportsXHighThinking` | `xhigh` 추론 지원 |
      | 34 | `resolveDefaultThinkingLevel` | 기본 `/think` 정책 |
      | 35 | `isModernModelRef` | live/smoke 모델 매칭 |
      | 36 | `prepareRuntimeAuth` | 추론 전 토큰 교환 |
      | 37 | `resolveUsageAuth` | 커스텀 사용량 자격 증명 파싱 |
      | 38 | `fetchUsageSnapshot` | 커스텀 사용량 엔드포인트 |
      | 39 | `createEmbeddingProvider` | 메모리/검색용 제공자 소유 임베딩 어댑터 |
      | 40 | `buildReplayPolicy` | 커스텀 전사 기록 replay/압축 정책 |
      | 41 | `sanitizeReplayHistory` | 일반 정리 후 제공자 전용 replay 재작성 |
      | 42 | `validateReplayTurns` | 내장 실행기 전 엄격한 replay-turn 검증 |
      | 43 | `onModelSelected` | 선택 후 콜백(예: 텔레메트리) |

      프롬프트 튜닝 참고:

      - `resolveSystemPromptContribution`을 사용하면 제공자가 모델 계열에 대한 캐시 인식
        시스템 프롬프트 지침을 주입할 수 있습니다. 동작이 특정 제공자/모델
        계열에 속하고 안정/동적 캐시 분리를 유지해야 한다면
        `before_prompt_build`보다 이것을 사용하는 것이 좋습니다.

      자세한 설명과 실제 예시는
      [내부 구조: 제공자 런타임 Hooks](/ko/plugins/architecture#provider-runtime-hooks)를 참조하세요.
    </Accordion>

  </Step>

  <Step title="추가 capability 추가(선택 사항)">
    <a id="step-5-add-extra-capabilities"></a>
    제공자 플러그인은 텍스트 추론과 함께 음성, realtime 전사, realtime
    음성, 미디어 이해, 이미지 생성, 비디오 생성, 웹 가져오기,
    웹 검색을 등록할 수 있습니다:

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

    OpenClaw는 이를 **하이브리드 capability** 플러그인으로 분류합니다. 이것이
    회사 플러그인(공급업체당 플러그인 하나)에 권장되는 패턴입니다.
    [내부 구조: Capability 소유권](/ko/plugins/architecture#capability-ownership-model)을 참조하세요.

    비디오 생성의 경우 위에 나온 모드 인식 capability 형태를 사용하는 것이 좋습니다:
    `generate`, `imageToVideo`, `videoToVideo`. `maxInputImages`, `maxInputVideos`, `maxDurationSeconds` 같은
    평평한 집계 필드만으로는
    변환 모드 지원이나 비활성화된 모드를 깔끔하게 알리기에 충분하지 않습니다.

    음악 생성 제공자도 같은 패턴을 따라야 합니다:
    프롬프트 전용 생성에는 `generate`, 참조 이미지 기반
    생성에는 `edit`를 사용합니다. `maxInputImages`,
    `supportsLyrics`, `supportsFormat` 같은 평평한 집계 필드만으로는 편집
    지원을 알리기에 충분하지 않으며, 명시적인 `generate` / `edit` 블록이
    기대되는 계약입니다.

  </Step>

  <Step title="테스트">
    <a id="step-6-test"></a>
    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // index.ts 또는 전용 파일에서 제공자 config 객체를 export하세요
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

## ClawHub에 게시

제공자 플러그인은 다른 외부 코드 플러그인과 같은 방식으로 게시합니다:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

여기서는 레거시 skill-only 게시 별칭을 사용하지 마세요. 플러그인 패키지는
`clawhub package publish`를 사용해야 합니다.

## 파일 구조

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers 메타데이터
├── openclaw.plugin.json      # 제공자 인증 메타데이터가 있는 매니페스트
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # 테스트
    └── usage.ts              # 사용량 엔드포인트(선택 사항)
```

## 카탈로그 순서 참고

`catalog.order`는 기본 제공
제공자에 비해 카탈로그가 언제 병합되는지를 제어합니다:

| Order     | 시점          | 사용 사례                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | 첫 번째 패스    | 일반 API 키 제공자                         |
| `profile` | simple 이후  | 인증 프로필에 의해 제한되는 제공자                |
| `paired`  | profile 이후 | 여러 관련 항목 합성             |
| `late`    | 마지막 패스     | 기존 제공자 재정의(충돌 시 우선 적용) |

## 다음 단계

- [채널 플러그인](/ko/plugins/sdk-channel-plugins) — 플러그인이 채널도 제공하는 경우
- [SDK Runtime](/ko/plugins/sdk-runtime) — `api.runtime` 헬퍼(TTS, 검색, 하위 에이전트)
- [SDK 개요](/ko/plugins/sdk-overview) — 전체 하위 경로 import 참고
- [플러그인 내부 구조](/ko/plugins/architecture#provider-runtime-hooks) — hook 세부 사항 및 번들 예시
