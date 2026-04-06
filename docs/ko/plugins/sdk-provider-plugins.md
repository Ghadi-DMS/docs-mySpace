---
read_when:
    - 새 모델 공급자 plugin을 구축하고 있을 때
    - OpenAI 호환 프록시 또는 사용자 지정 LLM을 OpenClaw에 추가하고 싶을 때
    - 공급자 인증, 카탈로그, 런타임 훅을 이해해야 할 때
sidebarTitle: Provider Plugins
summary: OpenClaw용 모델 공급자 plugin을 단계별로 구축하는 가이드
title: Provider Plugins 구축하기
x-i18n:
    generated_at: "2026-04-06T03:10:55Z"
    model: gpt-5.4
    provider: openai
    source_hash: 69500f46aa2cfdfe16e85b0ed9ee3c0032074be46f2d9c9d2940d18ae1095f47
    source_path: plugins/sdk-provider-plugins.md
    workflow: 15
---

# Provider Plugins 구축하기

이 가이드는 OpenClaw에 모델 공급자
(LLM)를 추가하는 provider plugin 구축 과정을 안내합니다. 이 가이드를 마치면 모델 카탈로그,
API 키 인증, 동적 모델 해석 기능을 갖춘 공급자를 갖게 됩니다.

<Info>
  아직 OpenClaw plugin을 한 번도 만들어본 적이 없다면 기본 패키지
  구조와 매니페스트 설정을 먼저 이해하기 위해
  [Getting Started](/ko/plugins/building-plugins)를 먼저 읽으세요.
</Info>

## 단계별 안내

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
    plugin 런타임을 로드하지 않고도 자격 증명을 감지할 수 있습니다. `modelSupport`는 선택 사항이며
    런타임 훅이 존재하기 전에 `acme-large` 같은 축약형 모델 id에서
    OpenClaw가 자동으로 공급자 plugin을 로드할 수 있게 해줍니다. 공급자를
    ClawHub에 게시하려면 `package.json`에 있는 해당 `openclaw.compat` 및 `openclaw.build` 필드가
    필요합니다.

  </Step>

  <Step title="공급자 등록">
    최소한의 공급자에는 `id`, `label`, `auth`, `catalog`가 필요합니다.

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

    이것으로 동작하는 공급자가 됩니다. 이제 사용자는
    `openclaw onboard --acme-ai-api-key <key>`를 실행하고
    `acme-ai/acme-large`를 모델로 선택할 수 있습니다.

    API 키 인증이 있는 텍스트 공급자 하나와 단일 catalog 기반 런타임만 등록하는 번들 공급자의 경우,
    더 좁은 범위의
    `defineSingleProviderPluginEntry(...)` 헬퍼를 사용하는 것이 좋습니다.

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

    인증 흐름에서 온보딩 중 `models.providers.*`, 별칭, 에이전트 기본 모델도
    함께 패치해야 한다면
    `openclaw/plugin-sdk/provider-onboard`의 preset 헬퍼를 사용하세요. 가장 좁은 범위의 헬퍼는
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)`, 그리고
    `createModelCatalogPresetAppliers(...)`입니다.

    공급자의 네이티브 엔드포인트가 일반 `openai-completions` 전송에서 스트리밍 사용량 블록을
    지원하는 경우, 공급자 id 확인을 하드코딩하지 말고
    `openclaw/plugin-sdk/provider-catalog-shared`의 공유 catalog 헬퍼를 사용하는 것이 좋습니다.
    `supportsNativeStreamingUsageCompat(...)`와
    `applyProviderNativeStreamingUsageCompat(...)`는 엔드포인트 capability map에서 지원 여부를 감지하므로,
    사용자 지정 공급자 id를 사용하는 plugin에서도 네이티브 Moonshot/DashScope 스타일 엔드포인트가
    계속 opt-in할 수 있습니다.

  </Step>

  <Step title="동적 모델 해석 추가">
    공급자가 임의의 모델 ID(프록시나 라우터처럼)를 허용한다면
    `resolveDynamicModel`을 추가하세요.

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

    해석에 네트워크 호출이 필요하다면 비동기
    워밍업을 위해 `prepareDynamicModel`을 사용하세요. 완료 후 `resolveDynamicModel`이 다시 실행됩니다.

  </Step>

  <Step title="런타임 훅 추가(필요한 경우)">
    대부분의 공급자는 `catalog` + `resolveDynamicModel`만 필요합니다. 공급자에 필요한 경우에만
    점진적으로 훅을 추가하세요.

    이제 공유 헬퍼 빌더가 가장 일반적인 replay/tool-compat
    계열을 지원하므로, plugin이 각 훅을 하나씩 직접 연결할 필요는 대체로 없습니다.

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

    | Family | 연결되는 내용 |
    | --- | --- |
    | `openai-compatible` | OpenAI 호환 전송을 위한 공유 OpenAI 스타일 replay 정책. tool-call-id 정리, assistant-first 순서 수정, 전송 계층에 필요한 일반 Gemini 턴 검증 포함 |
    | `anthropic-by-model` | `modelId`로 선택되는 Claude 인지 replay 정책. Anthropic-message 전송은 해석된 모델이 실제 Claude id일 때만 Claude 전용 thinking-block 정리를 적용 |
    | `google-gemini` | 네이티브 Gemini replay 정책과 bootstrap replay 정리, 태그된 reasoning-output 모드 |
    | `passthrough-gemini` | OpenAI 호환 프록시 전송을 거치는 Gemini 모델용 Gemini thought-signature 정리. 네이티브 Gemini replay 검증이나 bootstrap 재작성은 활성화하지 않음 |
    | `hybrid-anthropic-openai` | 하나의 plugin 안에서 Anthropic-message와 OpenAI 호환 모델 표면을 함께 섞는 공급자를 위한 하이브리드 정책. 선택적인 Claude 전용 thinking-block 제거는 Anthropic 쪽으로만 범위가 제한됨 |

    실제 번들 예시:

    - `google`: `google-gemini`
    - `openrouter`, `kilocode`, `opencode`, `opencode-go`: `passthrough-gemini`
    - `amazon-bedrock` 및 `anthropic-vertex`: `anthropic-by-model`
    - `minimax`: `hybrid-anthropic-openai`
    - `moonshot`, `ollama`, `xai`, `zai`: `openai-compatible`

    현재 사용 가능한 stream 계열:

    | Family | 연결되는 내용 |
    | --- | --- |
    | `google-thinking` | 공유 stream 경로의 Gemini thinking 페이로드 정규화 |
    | `kilocode-thinking` | 공유 프록시 stream 경로의 Kilo reasoning 래퍼. `kilo/auto`와 지원되지 않는 프록시 reasoning id는 주입된 thinking을 건너뜀 |
    | `moonshot-thinking` | config + `/think` 레벨의 Moonshot 바이너리 native-thinking 페이로드 매핑 |
    | `minimax-fast-mode` | 공유 stream 경로의 MiniMax fast-mode 모델 재작성 |
    | `openai-responses-defaults` | 공유 네이티브 OpenAI/Codex Responses 래퍼: attribution 헤더, `/fast`/`serviceTier`, 텍스트 verbosity, 네이티브 Codex web search, reasoning-compat 페이로드 shaping, Responses 컨텍스트 관리 |
    | `openrouter-thinking` | 프록시 경로용 OpenRouter reasoning 래퍼. 지원되지 않는 모델/`auto` 건너뛰기는 중앙에서 처리 |
    | `tool-stream-default-on` | 명시적으로 비활성화되지 않는 한 tool streaming을 원할 때 사용하는 Z.AI 같은 공급자용 기본 활성 `tool_stream` 래퍼 |

    실제 번들 예시:

    - `google`: `google-thinking`
    - `kilocode`: `kilocode-thinking`
    - `moonshot`: `moonshot-thinking`
    - `minimax` 및 `minimax-portal`: `minimax-fast-mode`
    - `openai` 및 `openai-codex`: `openai-responses-defaults`
    - `openrouter`: `openrouter-thinking`
    - `zai`: `tool-stream-default-on`

    `openclaw/plugin-sdk/provider-model-shared`는 replay-family
    enum과 해당 계열이 기반으로 삼는 공유 헬퍼도 export합니다. 일반적인 공개 export는 다음과 같습니다.

    - `ProviderReplayFamily`
    - `buildProviderReplayFamilyHooks(...)`
    - `buildOpenAICompatibleReplayPolicy(...)`,
      `buildAnthropicReplayPolicyForModel(...)`,
      `buildGoogleGeminiReplayPolicy(...)`,
      `buildHybridAnthropicOrOpenAIReplayPolicy(...)` 같은 공유 replay 빌더
    - `sanitizeGoogleGeminiReplayHistory(...)`
      및 `resolveTaggedReasoningOutputMode()` 같은 Gemini replay 헬퍼
    - `resolveProviderEndpoint(...)`,
      `normalizeProviderId(...)`, `normalizeGooglePreviewModelId(...)`,
      `normalizeNativeXaiModelId(...)` 같은 endpoint/model 헬퍼

    `openclaw/plugin-sdk/provider-stream`은 family builder와
    해당 계열이 재사용하는 공개 래퍼 헬퍼 둘 다 노출합니다. 일반적인 공개 export는
    다음과 같습니다.

    - `ProviderStreamFamily`
    - `buildProviderStreamFamilyHooks(...)`
    - `composeProviderStreamWrappers(...)`
    - `createOpenAIAttributionHeadersWrapper(...)`,
      `createOpenAIFastModeWrapper(...)`,
      `createOpenAIServiceTierWrapper(...)`,
      `createOpenAIResponsesContextManagementWrapper(...)`,
      `createCodexNativeWebSearchWrapper(...)` 같은 공유 OpenAI/Codex 래퍼
    - `createOpenRouterWrapper(...)`,
      `createToolStreamWrapper(...)`, `createMinimaxFastModeWrapper(...)` 같은 공유 프록시/공급자 래퍼

    일부 stream 헬퍼는 의도적으로 공급자 로컬로 유지됩니다. 현재 번들
    예시: `@openclaw/anthropic-provider`는
    `api.ts` / `contract-api.ts` seam의 공개 경로를 통해
    `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, 그리고 더 낮은 수준의 Anthropic 래퍼 빌더를 export합니다.
    이러한 헬퍼는 Claude OAuth 베타 처리와 `context1m` 게이팅도 인코딩하기 때문에
    Anthropic 전용으로 남아 있습니다.

    다른 번들 공급자도 동작이 계열 전반에 깔끔하게 공유되지 않을 경우
    전송별 래퍼를 로컬에 유지합니다. 현재 예시: 번들 xAI plugin은
    `/fast` 별칭 재작성, 기본 `tool_stream`,
    지원되지 않는 strict-tool 정리, xAI 전용 reasoning-payload 제거를 포함한
    네이티브 xAI Responses shaping을 자체 `wrapStreamFn` 안에 유지합니다.

    `openclaw/plugin-sdk/provider-tools`는 현재 하나의 공유
    tool-schema 계열과 공유 schema/compat 헬퍼를 노출합니다.

    - `ProviderToolCompatFamily`는 현재의 공유 계열 목록을 문서화합니다.
    - `buildProviderToolCompatFamilyHooks("gemini")`는 Gemini 안전 tool schema가 필요한 공급자를 위해
      Gemini schema 정리 + 진단을 연결합니다.
    - `normalizeGeminiToolSchemas(...)`와 `inspectGeminiToolSchemas(...)`는
      그 기반이 되는 공개 Gemini schema 헬퍼입니다.
    - `resolveXaiModelCompatPatch()`는 번들 xAI compat patch를 반환합니다:
      `toolSchemaProfile: "xai"`, 지원되지 않는 schema 키워드, 네이티브
      `web_search` 지원, HTML 엔터티 tool-call 인수 디코딩.
    - `applyXaiModelCompat(model)`은 러너에 도달하기 전에
      해석된 모델에 동일한 xAI compat patch를 적용합니다.

    실제 번들 예시: xAI plugin은 `normalizeResolvedModel`과
    `contributeResolvedModelCompat`를 사용하여 xAI 규칙을 core에 하드코딩하는 대신,
    compat 메타데이터의 소유권을 공급자 쪽에 유지합니다.

    같은 package-root 패턴은 다른 번들 공급자에도 적용됩니다.

    - `@openclaw/openai-provider`: `api.ts`가 공급자 빌더,
      기본 모델 헬퍼, realtime 공급자 빌더를 export
    - `@openclaw/openrouter-provider`: `api.ts`가 공급자 빌더와
      onboarding/config 헬퍼를 export

    <Tabs>
      <Tab title="토큰 교환">
        각 추론 호출 전에 토큰 교환이 필요한 공급자의 경우:

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
      <Tab title="사용자 지정 헤더">
        사용자 지정 요청 헤더나 본문 수정이 필요한 공급자의 경우:

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
      <Tab title="네이티브 전송 식별성">
        일반 HTTP 또는 WebSocket 전송에서 네이티브 요청/세션 헤더나 메타데이터가 필요한 공급자의 경우:

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
      <Tab title="사용량과 과금">
        사용량/과금 데이터를 노출하는 공급자의 경우:

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

    <Accordion title="사용 가능한 모든 공급자 훅">
      OpenClaw는 이 순서로 훅을 호출합니다. 대부분의 공급자는 2~3개만 사용합니다.

      | # | Hook | 사용 시점 |
      | --- | --- | --- |
      | 1 | `catalog` | 모델 카탈로그 또는 기본 base URL |
      | 2 | `applyConfigDefaults` | config 구체화 중 공급자 소유 전역 기본값 |
      | 3 | `normalizeModelId` | 조회 전 레거시/프리뷰 모델 id 별칭 정리 |
      | 4 | `normalizeTransport` | 일반 모델 조립 전 공급자 계열 `api` / `baseUrl` 정리 |
      | 5 | `normalizeConfig` | `models.providers.<id>` config 정규화 |
      | 6 | `applyNativeStreamingUsageCompat` | config 공급자를 위한 네이티브 streaming-usage compat 재작성 |
      | 7 | `resolveConfigApiKey` | 공급자 소유 env-marker 인증 해석 |
      | 8 | `resolveSyntheticAuth` | 로컬/셀프 호스팅 또는 config 기반 synthetic auth |
      | 9 | `shouldDeferSyntheticProfileAuth` | synthetic 저장 프로필 placeholder를 env/config auth 뒤로 내림 |
      | 10 | `resolveDynamicModel` | 임의의 업스트림 모델 ID 허용 |
      | 11 | `prepareDynamicModel` | 해석 전 비동기 메타데이터 가져오기 |
      | 12 | `normalizeResolvedModel` | 러너 전송 전 transport 재작성 |

    런타임 폴백 참고:

    - `normalizeConfig`는 먼저 일치하는 공급자를 확인한 다음,
      실제로 config를 변경하는 훅 가능한 다른 공급자 plugin들을 확인합니다.
      어떤 공급자 훅도 지원되는 Google 계열 config 항목을 재작성하지 않으면,
      번들 Google config normalizer가 여전히 적용됩니다.
    - `resolveConfigApiKey`는 노출된 경우 공급자 훅을 사용합니다. 번들
      `amazon-bedrock` 경로에도 여기서 내장 AWS env-marker resolver가 있지만,
      Bedrock 런타임 인증 자체는 여전히 AWS SDK 기본 체인을 사용합니다.
      | 13 | `contributeResolvedModelCompat` | 다른 호환 전송 뒤의 벤더 모델용 compat 플래그 |
      | 14 | `capabilities` | 레거시 정적 capability bag, 호환성 전용 |
      | 15 | `normalizeToolSchemas` | 등록 전 공급자 소유 tool-schema 정리 |
      | 16 | `inspectToolSchemas` | 공급자 소유 tool-schema 진단 |
      | 17 | `resolveReasoningOutputMode` | 태그형 vs 네이티브 reasoning-output 계약 |
      | 18 | `prepareExtraParams` | 기본 요청 파라미터 |
      | 19 | `createStreamFn` | 완전한 사용자 지정 StreamFn 전송 |
      | 20 | `wrapStreamFn` | 일반 stream 경로의 사용자 지정 헤더/본문 래퍼 |
      | 21 | `resolveTransportTurnState` | 네이티브 턴별 헤더/메타데이터 |
      | 22 | `resolveWebSocketSessionPolicy` | 네이티브 WS 세션 헤더/쿨다운 |
      | 23 | `formatApiKey` | 사용자 지정 런타임 토큰 형식 |
      | 24 | `refreshOAuth` | 사용자 지정 OAuth 갱신 |
      | 25 | `buildAuthDoctorHint` | 인증 복구 가이드 |
      | 26 | `matchesContextOverflowError` | 공급자 소유 overflow 감지 |
      | 27 | `classifyFailoverReason` | 공급자 소유 rate-limit/overload 분류 |
      | 28 | `isCacheTtlEligible` | 프롬프트 캐시 TTL 게이팅 |
      | 29 | `buildMissingAuthMessage` | 사용자 지정 missing-auth 힌트 |
      | 30 | `suppressBuiltInModel` | 오래된 업스트림 행 숨기기 |
      | 31 | `augmentModelCatalog` | synthetic forward-compat 행 |
      | 32 | `isBinaryThinking` | 바이너리 thinking on/off |
      | 33 | `supportsXHighThinking` | `xhigh` reasoning 지원 |
      | 34 | `resolveDefaultThinkingLevel` | 기본 `/think` 정책 |
      | 35 | `isModernModelRef` | live/smoke 모델 매칭 |
      | 36 | `prepareRuntimeAuth` | 추론 전 토큰 교환 |
      | 37 | `resolveUsageAuth` | 사용자 지정 사용량 자격 증명 파싱 |
      | 38 | `fetchUsageSnapshot` | 사용자 지정 사용량 엔드포인트 |
      | 39 | `createEmbeddingProvider` | memory/search용 공급자 소유 embedding adapter |
      | 40 | `buildReplayPolicy` | 사용자 지정 transcript replay/compaction 정책 |
      | 41 | `sanitizeReplayHistory` | 일반 정리 후 공급자별 replay 재작성 |
      | 42 | `validateReplayTurns` | embedded runner 전 엄격한 replay-turn 검증 |
      | 43 | `onModelSelected` | 선택 후 콜백(예: telemetry) |

      프롬프트 튜닝 참고:

      - `resolveSystemPromptContribution`은 공급자가 모델 계열을 위한 캐시 인지형
        시스템 프롬프트 가이드를 주입할 수 있게 합니다. 동작이 특정 공급자/모델
        계열에 속하고 안정/동적 캐시 분할을 유지해야 한다면
        `before_prompt_build`보다 이 방식을 권장합니다.

      자세한 설명과 실제 예시는
      [Internals: Provider Runtime Hooks](/ko/plugins/architecture#provider-runtime-hooks)를 참고하세요.
    </Accordion>

  </Step>

  <Step title="추가 capability 추가(선택 사항)">
    <a id="step-5-add-extra-capabilities"></a>
    provider plugin은 텍스트 추론 외에도 speech, realtime transcription, realtime
    voice, media understanding, image generation, video generation, web fetch,
    web search를 등록할 수 있습니다.

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

    OpenClaw는 이를 **hybrid-capability** plugin으로 분류합니다. 이것이
    회사 plugin에 권장되는 패턴입니다(벤더당 plugin 하나). 자세한 내용은
    [Internals: Capability Ownership](/ko/plugins/architecture#capability-ownership-model)을 참고하세요.

  </Step>

  <Step title="테스트">
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

## ClawHub에 게시

Provider plugin은 다른 외부 코드 plugin과 같은 방식으로 게시합니다.

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

여기서는 레거시 skill 전용 publish 별칭을 사용하지 마세요. plugin 패키지는
`clawhub package publish`를 사용해야 합니다.

## 파일 구조

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers metadata
├── openclaw.plugin.json      # providerAuthEnvVars가 포함된 매니페스트
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # 테스트
    └── usage.ts              # 사용량 엔드포인트(선택 사항)
```

## Catalog order 참조

`catalog.order`는 내장
공급자에 대해 카탈로그가 언제 병합되는지 제어합니다.

| Order     | 시점          | 사용 사례                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | 첫 번째 패스    | 단순 API 키 공급자                         |
| `profile` | `simple` 이후  | auth profile에 의해 게이트되는 공급자                |
| `paired`  | `profile` 이후 | 서로 관련된 여러 항목을 synthetic하게 생성             |
| `late`    | 마지막 패스     | 기존 공급자 재정의(충돌 시 우선) |

## 다음 단계

- [Channel Plugins](/ko/plugins/sdk-channel-plugins) — plugin이 채널도 제공하는 경우
- [SDK Runtime](/ko/plugins/sdk-runtime) — `api.runtime` 헬퍼(TTS, search, subagent)
- [SDK Overview](/ko/plugins/sdk-overview) — 전체 subpath import 참조
- [Plugin Internals](/ko/plugins/architecture#provider-runtime-hooks) — 훅 세부 사항과 번들 예시
