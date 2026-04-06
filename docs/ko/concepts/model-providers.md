---
read_when:
    - provider별 모델 설정 참고 자료가 필요할 때
    - 모델 provider용 예시 구성이나 CLI 온보딩 명령어를 원할 때
summary: 예시 구성과 CLI 흐름이 포함된 모델 provider 개요
title: 모델 Providers
x-i18n:
    generated_at: "2026-04-06T03:08:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 15e4b82e07221018a723279d309e245bb4023bc06e64b3c910ef2cae3dfa2599
    source_path: concepts/model-providers.md
    workflow: 15
---

# 모델 providers

이 페이지는 **LLM/모델 providers**를 다룹니다(WhatsApp/Telegram 같은 채팅 채널이 아님).
모델 선택 규칙은 [/concepts/models](/ko/concepts/models)을 참조하세요.

## 빠른 규칙

- 모델 ref는 `provider/model` 형식을 사용합니다(예: `opencode/claude-opus-4-6`).
- `agents.defaults.models`를 설정하면 allowlist가 됩니다.
- CLI helper: `openclaw onboard`, `openclaw models list`, `openclaw models set <provider/model>`.
- 폴백 런타임 규칙, cooldown probe, 세션 override 유지 방식은
  [/concepts/model-failover](/ko/concepts/model-failover)에 문서화되어 있습니다.
- `models.providers.*.models[].contextWindow`는 네이티브 모델 메타데이터이고,
  `models.providers.*.models[].contextTokens`는 유효 런타임 상한입니다.
- Provider plugin은 `registerProvider({ catalog })`를 통해 모델 catalog를 주입할 수 있으며,
  OpenClaw는 `models.json`을 쓰기 전에 그 출력을 `models.providers`에 병합합니다.
- Provider manifest는 `providerAuthEnvVars`를 선언할 수 있으므로 일반적인 env 기반
  auth probe가 plugin 런타임을 로드할 필요가 없습니다. 남아 있는 core env-var
  맵은 이제 non-plugin/core providers와 Anthropic API-key-first 온보딩 같은 일부
  일반 우선순위 케이스에만 사용됩니다.
- Provider plugin은 다음을 통해 provider 런타임 동작도 소유할 수 있습니다.
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
  `prepareRuntimeAuth`, `resolveUsageAuth`, `fetchUsageSnapshot`, 그리고
  `onModelSelected`.
- 참고: provider 런타임 `capabilities`는 공유 runner 메타데이터(provider
  family, transcript/tooling 특이사항, transport/cache 힌트)입니다. 이것은
  plugin이 무엇을 등록하는지(텍스트 추론, speech 등)를 설명하는
  [public capability model](/ko/plugins/architecture#public-capability-model)과는 다릅니다.

## plugin이 소유하는 provider 동작

Provider plugin은 이제 대부분의 provider별 로직을 소유할 수 있고, OpenClaw는
일반 추론 루프를 유지합니다.

일반적인 분리 방식:

- `auth[].run` / `auth[].runNonInteractive`: provider가 `openclaw onboard`, `openclaw models auth`, 헤드리스 설정을 위한 온보딩/로그인 흐름을 소유
- `wizard.setup` / `wizard.modelPicker`: provider가 auth-choice 라벨,
  legacy alias, 온보딩 allowlist 힌트, 온보딩/모델 picker의 설정 항목을 소유
- `catalog`: provider가 `models.providers`에 표시됨
- `normalizeModelId`: provider가 조회 또는 canonicalization 전에 legacy/preview 모델 id를 정규화
- `normalizeTransport`: provider가 일반 모델 조립 전에 transport-family `api` / `baseUrl`을 정규화하며,
  OpenClaw는 먼저 일치하는 provider를 확인한 뒤, 실제로 transport를 변경하는
  hook-capable provider plugin이 나올 때까지 다른 plugin도 확인합니다
- `normalizeConfig`: provider가 런타임 사용 전에 `models.providers.<id>` config를 정규화하며,
  OpenClaw는 먼저 일치하는 provider를 확인한 뒤, 실제로 config를 변경하는
  hook-capable provider plugin이 나올 때까지 다른 plugin도 확인합니다. 어떤
  provider hook도 config를 다시 쓰지 않으면, 번들된 Google-family helper가 여전히
  지원되는 Google provider 항목을 정규화합니다.
- `applyNativeStreamingUsageCompat`: provider가 config provider에 대해 endpoint 기반 네이티브 streaming-usage 호환성 재작성 적용
- `resolveConfigApiKey`: provider가 전체 런타임 auth 로딩을 강제하지 않고
  config provider용 env-marker auth를 해결. `amazon-bedrock`은 Bedrock 런타임 auth가
  AWS SDK 기본 체인을 사용하더라도 여기에 내장 AWS env-marker resolver도 포함합니다.
- `resolveSyntheticAuth`: provider가 평문 secret 저장 없이 local/self-hosted 또는 기타
  config 기반 auth 가용성을 노출 가능
- `shouldDeferSyntheticProfileAuth`: provider가 저장된 synthetic profile placeholder를
  env/config 기반 auth보다 낮은 우선순위로 표시 가능
- `resolveDynamicModel`: provider가 아직 로컬 정적 catalog에 없는 모델 id를 수용
- `prepareDynamicModel`: provider가 동적 해결을 다시 시도하기 전에 메타데이터 새로고침 필요
- `normalizeResolvedModel`: provider가 transport 또는 base URL 재작성 필요
- `contributeResolvedModelCompat`: provider가 다른 호환 transport를 통해 들어온 경우에도
  자사 vendor 모델에 대한 compat 플래그를 기여
- `capabilities`: provider가 transcript/tooling/provider-family 특이사항을 게시
- `normalizeToolSchemas`: provider가 embedded runner가 보기 전에 tool schema를 정리
- `inspectToolSchemas`: provider가 정규화 후 transport별 schema 경고를 표시
- `resolveReasoningOutputMode`: provider가 native 대 tagged reasoning-output 계약 중 선택
- `prepareExtraParams`: provider가 모델별 요청 params를 기본 설정 또는 정규화
- `createStreamFn`: provider가 일반 stream 경로를 완전한 custom transport로 교체
- `wrapStreamFn`: provider가 요청 header/body/model compat wrapper 적용
- `resolveTransportTurnState`: provider가 턴별 네이티브 transport header 또는 metadata 제공
- `resolveWebSocketSessionPolicy`: provider가 네이티브 WebSocket 세션
  header 또는 세션 cool-down 정책 제공
- `createEmbeddingProvider`: provider가 core embedding switchboard 대신 provider plugin에 속하는
  메모리 embedding 동작을 소유
- `formatApiKey`: provider가 저장된 auth profile을 transport가 기대하는 런타임
  `apiKey` 문자열로 포맷
- `refreshOAuth`: 공유 `pi-ai`
  refresher만으로 부족할 때 provider가 OAuth refresh를 소유
- `buildAuthDoctorHint`: provider가 OAuth refresh 실패 시 복구 가이드 추가
- `matchesContextOverflowError`: provider가 일반 heuristic이 놓치는
  provider별 context-window overflow 오류를 인식
- `classifyFailoverReason`: provider가 provider별 원시 transport/API
  오류를 rate limit 또는 overload 같은 failover 이유로 매핑
- `isCacheTtlEligible`: provider가 어떤 업스트림 모델 id가 prompt-cache TTL을 지원하는지 결정
- `buildMissingAuthMessage`: provider가 일반 auth-store 오류를
  provider별 복구 힌트로 교체
- `suppressBuiltInModel`: provider가 오래된 업스트림 행을 숨기고 직접 해결 실패에 대해
  vendor 소유 오류를 반환 가능
- `augmentModelCatalog`: provider가 discovery 및 config 병합 후
  synthetic/final catalog 행 추가
- `isBinaryThinking`: provider가 binary on/off thinking UX를 소유
- `supportsXHighThinking`: provider가 선택된 모델에서 `xhigh`를 활성화
- `resolveDefaultThinkingLevel`: provider가 모델 family의 기본 `/think` 정책을 소유
- `applyConfigDefaults`: provider가 auth 모드, env, 모델 family를 기반으로
  config materialization 중 provider별 전역 기본값 적용
- `isModernModelRef`: provider가 live/smoke preferred-model 매칭을 소유
- `prepareRuntimeAuth`: provider가 설정된 자격 증명을 짧은 수명의 런타임 토큰으로 변환
- `resolveUsageAuth`: provider가 `/usage` 및 관련 상태/보고 화면용
  usage/quota 자격 증명을 해결
- `fetchUsageSnapshot`: provider가 usage endpoint fetch/parsing을 소유하고,
  core는 요약 셸과 포맷팅을 계속 소유
- `onModelSelected`: provider가 telemetry 또는 provider 소유 세션 bookkeeping 같은
  선택 후 부수 효과를 실행

현재 번들 예시:

- `anthropic`: Claude 4.6 forward-compat fallback, auth 복구 힌트, usage
  endpoint fetch, cache-TTL/provider-family metadata, auth 인식 전역
  config 기본값
- `amazon-bedrock`: Bedrock별 throttle/not-ready 오류에 대한 provider 소유
  context-overflow 매칭 및 failover 이유 분류, 그리고 Anthropic 트래픽에서
  Claude 전용 replay-policy 가드를 위한 공유 `anthropic-by-model` replay family
- `anthropic-vertex`: Anthropic-message
  트래픽에서 Claude 전용 replay-policy 가드
- `openrouter`: pass-through 모델 id, 요청 wrapper, provider capability
  힌트, proxy Gemini 트래픽에서의 Gemini thought-signature 정리,
  `openrouter-thinking` stream family를 통한 proxy reasoning 주입, routing
  metadata 전달, cache-TTL 정책
- `github-copilot`: 온보딩/device login, forward-compat 모델 fallback,
  Claude-thinking transcript 힌트, 런타임 토큰 교환, usage endpoint
  fetch
- `openai`: GPT-5.4 forward-compat fallback, direct OpenAI transport
  정규화, Codex 인식 missing-auth 힌트, Spark 숨김 처리, synthetic
  OpenAI/Codex catalog 행, thinking/live-model 정책, usage-token alias
  정규화(`input` / `output` 및 `prompt` / `completion` family), 네이티브 OpenAI/Codex
  wrapper를 위한 공유 `openai-responses-defaults` stream family,
  provider-family metadata, `gpt-image-1`용 번들 image-generation provider
  등록, `sora-2`용 번들 video-generation provider 등록
- `google`: Gemini 3.1 forward-compat fallback, 네이티브 Gemini replay
  검증, bootstrap replay 정리, tagged reasoning-output 모드,
  modern-model 매칭, Gemini image-preview 모델용 번들 image-generation provider 등록,
  Veo 모델용 번들 video-generation provider 등록
- `moonshot`: 공유 transport, plugin 소유 thinking payload 정규화
- `kilocode`: 공유 transport, plugin 소유 요청 header, reasoning payload
  정규화, proxy-Gemini thought-signature 정리, cache-TTL
  정책
- `zai`: GLM-5 forward-compat fallback, `tool_stream` 기본값, cache-TTL
  정책, binary-thinking/live-model 정책, usage auth + quota fetch;
  알 수 없는 `glm-5*` id는 번들된 `glm-4.7` 템플릿에서 합성됨
- `xai`: 네이티브 Responses transport 정규화, Grok fast variant를 위한 `/fast` alias 재작성,
  기본 `tool_stream`, xAI 전용 tool-schema /
  reasoning-payload 정리, `grok-imagine-video`용 번들 video-generation provider
  등록
- `mistral`: plugin 소유 capability metadata
- `opencode` 및 `opencode-go`: plugin 소유 capability metadata와
  proxy-Gemini thought-signature 정리
- `alibaba`: `alibaba/wan2.6-t2v` 같은 direct Wan model ref를 위한
  plugin 소유 video-generation catalog
- `byteplus`: plugin 소유 catalog와 Seedance text-to-video/image-to-video 모델용
  번들 video-generation provider 등록
- `fal`: 호스팅된 서드파티용 번들 video-generation provider 등록,
  FLUX 이미지 모델용 image-generation provider 등록, 그리고 호스팅된
  서드파티 비디오 모델용 번들 video-generation provider 등록
- `cloudflare-ai-gateway`, `huggingface`, `kimi`, `nvidia`, `qianfan`,
  `stepfun`, `synthetic`, `venice`, `vercel-ai-gateway`, `volcengine`:
  plugin 소유 catalog만 제공
- `qwen`: 텍스트 모델용 plugin 소유 catalog와 멀티모달 표면을 위한 공유
  media-understanding 및 video-generation provider 등록;
  Qwen 비디오 생성은 번들된 `wan2.6-t2v` 및 `wan2.7-r2v` 같은 모델과 함께
  Standard DashScope 비디오 endpoint를 사용
- `runway`: `gen4.5` 같은 네이티브 Runway task 기반 모델용
  plugin 소유 video-generation provider 등록
- `minimax`: plugin 소유 catalog, Hailuo 비디오 모델용 번들 video-generation provider
  등록, `image-01`용 번들 image-generation provider
  등록, 하이브리드 Anthropic/OpenAI replay-policy
  선택, usage auth/snapshot 로직
- `together`: plugin 소유 catalog와 Wan 비디오 모델용 번들
  video-generation provider 등록
- `xiaomi`: plugin 소유 catalog와 usage auth/snapshot 로직

번들된 `openai` plugin은 이제 두 provider id `openai`와
`openai-codex`를 모두 소유합니다.

여기까지는 OpenClaw의 일반 transport에 여전히 맞는 providers입니다. 완전히 custom 요청 실행기가 필요한 provider는 별도의 더 깊은 extension 표면입니다.

## API key 순환

- 선택된 providers에 대해 일반적인 provider key 순환을 지원합니다.
- 다음을 통해 여러 key를 설정합니다.
  - `OPENCLAW_LIVE_<PROVIDER>_KEY`(단일 live override, 최우선)
  - `<PROVIDER>_API_KEYS`(쉼표 또는 세미콜론 목록)
  - `<PROVIDER>_API_KEY`(기본 key)
  - `<PROVIDER>_API_KEY_*`(번호가 붙은 목록, 예: `<PROVIDER>_API_KEY_1`)
- Google providers의 경우 `GOOGLE_API_KEY`도 fallback으로 포함됩니다.
- key 선택 순서는 우선순위를 유지하면서 중복 값을 제거합니다.
- 요청은 rate-limit 응답에서만 다음 key로 재시도됩니다(예:
  `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many
concurrent requests`, `ThrottlingException`, `concurrency limit reached`,
  `workers_ai ... quota limit exceeded`, 또는 주기적인 usage-limit 메시지).
- rate-limit이 아닌 실패는 즉시 실패하며 key 순환을 시도하지 않습니다.
- 모든 후보 key가 실패하면 마지막 시도의 최종 오류가 반환됩니다.

## 내장 providers (`pi-ai` catalog)

OpenClaw는 pi‑ai catalog와 함께 제공됩니다. 이 providers는
`models.providers` config가 **필요 없습니다**. auth를 설정하고 모델만 선택하면 됩니다.

### OpenAI

- Provider: `openai`
- Auth: `OPENAI_API_KEY`
- 선택적 순환: `OPENAI_API_KEYS`, `OPENAI_API_KEY_1`, `OPENAI_API_KEY_2`, 그리고 `OPENCLAW_LIVE_OPENAI_KEY`(단일 override)
- 예시 모델: `openai/gpt-5.4`, `openai/gpt-5.4-pro`
- CLI: `openclaw onboard --auth-choice openai-api-key`
- 기본 transport는 `auto`입니다(WebSocket 우선, SSE fallback)
- 모델별 override: `agents.defaults.models["openai/<model>"].params.transport` (`"sse"`, `"websocket"`, 또는 `"auto"`)
- OpenAI Responses WebSocket warm-up은 `params.openaiWsWarmup` (`true`/`false`)을 통해 기본 활성화됨
- OpenAI priority processing은 `agents.defaults.models["openai/<model>"].params.serviceTier`를 통해 활성화 가능
- `/fast`와 `params.fastMode`는 direct `openai/*` Responses 요청을 `api.openai.com`의 `service_tier=priority`로 매핑
- 공유 `/fast` 토글 대신 명시적인 티어를 원하면 `params.serviceTier`를 사용하세요
- 숨겨진 OpenClaw attribution header(`originator`, `version`,
  `User-Agent`)는 generic OpenAI-compatible proxy가 아니라 `api.openai.com`으로 가는
  네이티브 OpenAI 트래픽에만 적용됩니다
- 네이티브 OpenAI 경로는 Responses `store`, prompt-cache 힌트,
  OpenAI reasoning-compat payload shaping도 유지하며,
  proxy 경로는 그렇지 않습니다
- `openai/gpt-5.3-codex-spark`는 live OpenAI API가 이를 거부하므로 OpenClaw에서 의도적으로 숨겨져 있습니다. Spark는 Codex 전용으로 취급됩니다

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

### Anthropic

- Provider: `anthropic`
- Auth: `ANTHROPIC_API_KEY`
- 선택적 순환: `ANTHROPIC_API_KEYS`, `ANTHROPIC_API_KEY_1`, `ANTHROPIC_API_KEY_2`, 그리고 `OPENCLAW_LIVE_ANTHROPIC_KEY`(단일 override)
- 예시 모델: `anthropic/claude-opus-4-6`
- CLI: `openclaw onboard --auth-choice apiKey`
- direct public Anthropic 요청은 `api.anthropic.com`으로 전송되는 API-key 및 OAuth 인증 트래픽을 포함해
  공유 `/fast` 토글과 `params.fastMode`를 지원하며,
  OpenClaw는 이를 Anthropic `service_tier` (`auto` vs `standard_only`)로 매핑합니다
- 과금 참고: OpenClaw에서 Anthropic의 실질적인 구분은 **API key** 또는
  **Extra Usage가 포함된 Claude subscription**입니다. Anthropic은 **2026년 4월 4일
  오후 12:00 PT / 오후 8:00 BST**에 OpenClaw 사용자에게 **OpenClaw**
  Claude 로그인 경로가 서드파티 harness 사용으로 간주되며 subscription과 별도로 청구되는
  **Extra Usage**가 필요하다고 알렸습니다. 로컬 재현에서도 OpenClaw를 식별하는
  프롬프트 문자열은 Anthropic SDK + API-key 경로에서 재현되지 않음을 확인했습니다.
- Anthropic setup-token은 다시 legacy/manual OpenClaw 경로로 사용할 수 있습니다.
  Anthropic이 이 경로에 **Extra Usage**가 필요하다고 OpenClaw 사용자에게 알렸다는 점을
  전제로 사용하세요.

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

### OpenAI Code (Codex)

- Provider: `openai-codex`
- Auth: OAuth (ChatGPT)
- 예시 모델: `openai-codex/gpt-5.4`
- CLI: `openclaw onboard --auth-choice openai-codex` 또는 `openclaw models auth login --provider openai-codex`
- 기본 transport는 `auto`입니다(WebSocket 우선, SSE fallback)
- 모델별 override: `agents.defaults.models["openai-codex/<model>"].params.transport` (`"sse"`, `"websocket"`, 또는 `"auto"`)
- `params.serviceTier`는 네이티브 Codex Responses 요청(`chatgpt.com/backend-api`)에서도 전달됩니다
- 숨겨진 OpenClaw attribution header(`originator`, `version`,
  `User-Agent`)는 generic OpenAI-compatible proxy가 아니라
  `chatgpt.com/backend-api`로 가는 네이티브 Codex 트래픽에만 첨부됩니다
- direct `openai/*`와 동일한 `/fast` 토글 및 `params.fastMode` config를 공유하며,
  OpenClaw는 이를 `service_tier=priority`로 매핑합니다
- `openai-codex/gpt-5.3-codex-spark`는 Codex OAuth catalog에서 노출될 때 계속 사용 가능하며,
  entitlement 의존적입니다
- `openai-codex/gpt-5.4`는 네이티브 `contextWindow = 1050000`과 기본 런타임
  `contextTokens = 272000`을 유지합니다. 런타임 상한은 `models.providers.openai-codex.models[].contextTokens`로 override하세요
- 정책 참고: OpenAI Codex OAuth는 OpenClaw 같은 외부 도구/워크플로용으로 명시적으로 지원됩니다.

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

### 기타 subscription 스타일 호스팅 옵션

- [Qwen Cloud](/ko/providers/qwen): Qwen Cloud provider 표면과 Alibaba DashScope 및 Coding Plan endpoint 매핑
- [MiniMax](/ko/providers/minimax): MiniMax Coding Plan OAuth 또는 API key 접근
- [GLM Models](/ko/providers/glm): Z.AI Coding Plan 또는 일반 API endpoint

### OpenCode

- Auth: `OPENCODE_API_KEY`(또는 `OPENCODE_ZEN_API_KEY`)
- Zen 런타임 provider: `opencode`
- Go 런타임 provider: `opencode-go`
- 예시 모델: `opencode/claude-opus-4-6`, `opencode-go/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice opencode-zen` 또는 `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (API key)

- Provider: `google`
- Auth: `GEMINI_API_KEY`
- 선택적 순환: `GEMINI_API_KEYS`, `GEMINI_API_KEY_1`, `GEMINI_API_KEY_2`, `GOOGLE_API_KEY` fallback, 그리고 `OPENCLAW_LIVE_GEMINI_KEY`(단일 override)
- 예시 모델: `google/gemini-3.1-pro-preview`, `google/gemini-3-flash-preview`
- 호환성: `google/gemini-3.1-flash-preview`를 사용하는 legacy OpenClaw config는 `google/gemini-3-flash-preview`로 정규화됩니다
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- direct Gemini 실행은 `agents.defaults.models["google/<model>"].params.cachedContent`
  (또는 legacy `cached_content`)도 허용하며, provider 네이티브
  `cachedContents/...` 핸들을 전달합니다. Gemini cache hit는 OpenClaw `cacheRead`로 표시됩니다

### Google Vertex

- Provider: `google-vertex`
- Auth: gcloud ADC
  - Gemini CLI JSON 응답은 `response`에서 파싱되며, usage는
    `stats`로 fallback하고, `stats.cached`는 OpenClaw `cacheRead`로 정규화됩니다.

### Z.AI (GLM)

- Provider: `zai`
- Auth: `ZAI_API_KEY`
- 예시 모델: `zai/glm-5`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - alias: `z.ai/*`와 `z-ai/*`는 `zai/*`로 정규화됩니다
  - `zai-api-key`는 일치하는 Z.AI endpoint를 자동 감지하고, `zai-coding-global`, `zai-coding-cn`, `zai-global`, `zai-cn`은 특정 표면을 강제합니다

### Vercel AI Gateway

- Provider: `vercel-ai-gateway`
- Auth: `AI_GATEWAY_API_KEY`
- 예시 모델: `vercel-ai-gateway/anthropic/claude-opus-4.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### Kilo Gateway

- Provider: `kilocode`
- Auth: `KILOCODE_API_KEY`
- 예시 모델: `kilocode/kilo/auto`
- CLI: `openclaw onboard --auth-choice kilocode-api-key`
- Base URL: `https://api.kilo.ai/api/gateway/`
- 정적 fallback catalog에는 `kilocode/kilo/auto`가 포함되며,
  live `https://api.kilo.ai/api/gateway/models` discovery가 런타임
  catalog를 더 확장할 수 있습니다.
- `kilocode/kilo/auto` 뒤의 정확한 업스트림 라우팅은 OpenClaw에 하드코딩되어 있지 않고
  Kilo Gateway가 소유합니다.

설정 세부 정보는 [/providers/kilocode](/ko/providers/kilocode)를 참조하세요.

### 기타 번들 provider plugins

- OpenRouter: `openrouter` (`OPENROUTER_API_KEY`)
- 예시 모델: `openrouter/auto`
- OpenClaw는 요청이 실제로 `openrouter.ai`를 대상으로 할 때만
  OpenRouter 문서화된 app-attribution header를 적용합니다
- OpenRouter 전용 Anthropic `cache_control` marker도 임의 proxy URL이 아니라
  검증된 OpenRouter 경로에서만 적용됩니다
- OpenRouter는 proxy 스타일 OpenAI-compatible 경로에 그대로 머무르므로,
  네이티브 OpenAI 전용 요청 shaping(`serviceTier`, Responses `store`,
  prompt-cache 힌트, OpenAI reasoning-compat payload)은 전달되지 않습니다
- Gemini 기반 OpenRouter ref는 proxy-Gemini thought-signature
  정리만 유지하며, 네이티브 Gemini replay 검증 및 bootstrap 재작성은 비활성화됩니다
- Kilo Gateway: `kilocode` (`KILOCODE_API_KEY`)
- 예시 모델: `kilocode/kilo/auto`
- Gemini 기반 Kilo ref도 동일한 proxy-Gemini thought-signature
  정리 경로를 유지하며, `kilocode/kilo/auto` 및 기타 proxy-reasoning-unsupported
  힌트는 proxy reasoning 주입을 건너뜁니다
- MiniMax: `minimax` (API key) 및 `minimax-portal` (OAuth)
- Auth: `minimax`는 `MINIMAX_API_KEY`, `minimax-portal`은 `MINIMAX_OAUTH_TOKEN` 또는 `MINIMAX_API_KEY`
- 예시 모델: `minimax/MiniMax-M2.7` 또는 `minimax-portal/MiniMax-M2.7`
- MiniMax 온보딩/API-key 설정은 `input: ["text", "image"]`가 있는
  명시적 M2.7 모델 정의를 기록하며, 번들 provider catalog는 해당 provider config가
  materialize될 때까지 채팅 ref를 text-only로 유지합니다
- Moonshot: `moonshot` (`MOONSHOT_API_KEY`)
- 예시 모델: `moonshot/kimi-k2.5`
- Kimi Coding: `kimi` (`KIMI_API_KEY` 또는 `KIMICODE_API_KEY`)
- 예시 모델: `kimi/kimi-code`
- Qianfan: `qianfan` (`QIANFAN_API_KEY`)
- 예시 모델: `qianfan/deepseek-v3.2`
- Qwen Cloud: `qwen` (`QWEN_API_KEY`, `MODELSTUDIO_API_KEY`, 또는 `DASHSCOPE_API_KEY`)
- 예시 모델: `qwen/qwen3.5-plus`
- NVIDIA: `nvidia` (`NVIDIA_API_KEY`)
- 예시 모델: `nvidia/nvidia/llama-3.1-nemotron-70b-instruct`
- StepFun: `stepfun` / `stepfun-plan` (`STEPFUN_API_KEY`)
- 예시 모델: `stepfun/step-3.5-flash`, `stepfun-plan/step-3.5-flash-2603`
- Together: `together` (`TOGETHER_API_KEY`)
- 예시 모델: `together/moonshotai/Kimi-K2.5`
- Venice: `venice` (`VENICE_API_KEY`)
- Xiaomi: `xiaomi` (`XIAOMI_API_KEY`)
- 예시 모델: `xiaomi/mimo-v2-flash`
- Vercel AI Gateway: `vercel-ai-gateway` (`AI_GATEWAY_API_KEY`)
- Hugging Face Inference: `huggingface` (`HUGGINGFACE_HUB_TOKEN` 또는 `HF_TOKEN`)
- Cloudflare AI Gateway: `cloudflare-ai-gateway` (`CLOUDFLARE_AI_GATEWAY_API_KEY`)
- Volcengine: `volcengine` (`VOLCANO_ENGINE_API_KEY`)
- 예시 모델: `volcengine-plan/ark-code-latest`
- BytePlus: `byteplus` (`BYTEPLUS_API_KEY`)
- 예시 모델: `byteplus-plan/ark-code-latest`
- xAI: `xai` (`XAI_API_KEY`)
  - 네이티브 번들 xAI 요청은 xAI Responses 경로를 사용합니다
  - `/fast` 또는 `params.fastMode: true`는 `grok-3`, `grok-3-mini`,
    `grok-4`, `grok-4-0709`를 해당 `*-fast` variant로 재작성합니다
  - `tool_stream`은 기본 활성화되어 있습니다.
    비활성화하려면 `agents.defaults.models["xai/<model>"].params.tool_stream`를 `false`로
    설정하세요
- Mistral: `mistral` (`MISTRAL_API_KEY`)
- 예시 모델: `mistral/mistral-large-latest`
- CLI: `openclaw onboard --auth-choice mistral-api-key`
- Groq: `groq` (`GROQ_API_KEY`)
- Cerebras: `cerebras` (`CEREBRAS_API_KEY`)
  - Cerebras의 GLM 모델은 `zai-glm-4.7` 및 `zai-glm-4.6` id를 사용합니다.
  - OpenAI-compatible base URL: `https://api.cerebras.ai/v1`.
- GitHub Copilot: `github-copilot` (`COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN`)
- Hugging Face Inference 예시 모델: `huggingface/deepseek-ai/DeepSeek-R1`; CLI: `openclaw onboard --auth-choice huggingface-api-key`. [Hugging Face (Inference)](/ko/providers/huggingface)를 참조하세요.

## `models.providers`를 통한 providers (custom/base URL)

**custom** providers나 OpenAI/Anthropic‑compatible proxy를 추가하려면
`models.providers`(또는 `models.json`)를 사용하세요.

아래의 많은 번들 provider plugin은 이미 기본 catalog를 게시합니다.
기본 `baseUrl`, header, 모델 목록을 override하려는 경우에만
명시적 `models.providers.<id>` 항목을 사용하세요.

### Moonshot AI (Kimi)

Moonshot은 번들 provider plugin으로 제공됩니다. 기본적으로 내장 provider를 사용하고,
base URL 또는 모델 metadata를 override해야 할 때만 명시적 `models.providers.moonshot`
항목을 추가하세요.

- Provider: `moonshot`
- Auth: `MOONSHOT_API_KEY`
- 예시 모델: `moonshot/kimi-k2.5`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` 또는 `openclaw onboard --auth-choice moonshot-api-key-cn`

Kimi K2 모델 ID:

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

Kimi Coding은 Moonshot AI의 Anthropic-compatible endpoint를 사용합니다.

- Provider: `kimi`
- Auth: `KIMI_API_KEY`
- 예시 모델: `kimi/kimi-code`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-code" } },
  },
}
```

legacy `kimi/k2p5`도 호환성 모델 id로 계속 허용됩니다.

### Volcano Engine (Doubao)

Volcano Engine(화산엔진)은 중국에서 Doubao 및 기타 모델에 대한 접근을 제공합니다.

- Provider: `volcengine` (coding: `volcengine-plan`)
- Auth: `VOLCANO_ENGINE_API_KEY`
- 예시 모델: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

온보딩은 기본적으로 coding 표면을 사용하지만, 일반 `volcengine/*`
catalog도 동시에 등록됩니다.

온보딩/모델 설정 picker에서 Volcengine auth choice는
`volcengine/*`와 `volcengine-plan/*` 행을 모두 우선 사용합니다. 해당 모델이 아직 로드되지 않았다면,
OpenClaw는 빈 provider 범위 picker를 표시하는 대신 필터링되지 않은 catalog로 fallback합니다.

사용 가능한 모델:

- `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
- `volcengine/doubao-seed-code-preview-251028`
- `volcengine/kimi-k2-5-260127` (Kimi K2.5)
- `volcengine/glm-4-7-251222` (GLM 4.7)
- `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2 128K)

코딩 모델(`volcengine-plan`):

- `volcengine-plan/ark-code-latest`
- `volcengine-plan/doubao-seed-code`
- `volcengine-plan/kimi-k2.5`
- `volcengine-plan/kimi-k2-thinking`
- `volcengine-plan/glm-4.7`

### BytePlus (International)

BytePlus ARK는 국제 사용자를 위해 Volcano Engine과 동일한 모델에 대한 접근을 제공합니다.

- Provider: `byteplus` (coding: `byteplus-plan`)
- Auth: `BYTEPLUS_API_KEY`
- 예시 모델: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

온보딩은 기본적으로 coding 표면을 사용하지만, 일반 `byteplus/*`
catalog도 동시에 등록됩니다.

온보딩/모델 설정 picker에서 BytePlus auth choice는
`byteplus/*`와 `byteplus-plan/*` 행을 모두 우선 사용합니다. 해당 모델이 아직 로드되지 않았다면,
OpenClaw는 빈 provider 범위 picker를 표시하는 대신 필터링되지 않은 catalog로 fallback합니다.

사용 가능한 모델:

- `byteplus/seed-1-8-251228` (Seed 1.8)
- `byteplus/kimi-k2-5-260127` (Kimi K2.5)
- `byteplus/glm-4-7-251222` (GLM 4.7)

코딩 모델(`byteplus-plan`):

- `byteplus-plan/ark-code-latest`
- `byteplus-plan/doubao-seed-code`
- `byteplus-plan/kimi-k2.5`
- `byteplus-plan/kimi-k2-thinking`
- `byteplus-plan/glm-4.7`

### Synthetic

Synthetic는 `synthetic` provider 뒤에 Anthropic-compatible 모델을 제공합니다.

- Provider: `synthetic`
- Auth: `SYNTHETIC_API_KEY`
- 예시 모델: `synthetic/hf:MiniMaxAI/MiniMax-M2.5`
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

MiniMax는 custom endpoint를 사용하므로 `models.providers`를 통해 구성됩니다.

- MiniMax OAuth (Global): `--auth-choice minimax-global-oauth`
- MiniMax OAuth (CN): `--auth-choice minimax-cn-oauth`
- MiniMax API key (Global): `--auth-choice minimax-global-api`
- MiniMax API key (CN): `--auth-choice minimax-cn-api`
- Auth: `minimax`는 `MINIMAX_API_KEY`, `minimax-portal`은
  `MINIMAX_OAUTH_TOKEN` 또는 `MINIMAX_API_KEY`

설정 세부 정보, 모델 옵션, config 스니펫은 [/providers/minimax](/ko/providers/minimax)를 참조하세요.

MiniMax의 Anthropic-compatible streaming 경로에서 OpenClaw는
명시적으로 설정하지 않는 한 기본적으로 thinking을 비활성화하며,
`/fast on`은 `MiniMax-M2.7`을 `MiniMax-M2.7-highspeed`로 재작성합니다.

plugin이 소유하는 capability 분리:

- 텍스트/채팅 기본값은 `minimax/MiniMax-M2.7` 유지
- 이미지 생성은 `minimax/image-01` 또는 `minimax-portal/image-01`
- 이미지 이해는 두 MiniMax auth 경로 모두에서 plugin 소유 `MiniMax-VL-01`
- 웹 검색은 provider id `minimax` 유지

### Ollama

Ollama는 번들 provider plugin으로 제공되며 Ollama의 네이티브 API를 사용합니다.

- Provider: `ollama`
- Auth: 필요 없음(로컬 서버)
- 예시 모델: `ollama/llama3.3`
- 설치: [https://ollama.com/download](https://ollama.com/download)

```bash
# Ollama를 설치한 다음 모델을 pull하세요:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

Ollama는 `OLLAMA_API_KEY`로 opt-in하면 로컬 `http://127.0.0.1:11434`에서 감지되며,
번들 provider plugin이 Ollama를 직접
`openclaw onboard`와 모델 picker에 추가합니다. 온보딩, cloud/local 모드, custom config는 [/providers/ollama](/ko/providers/ollama)를 참조하세요.

### vLLM

vLLM은 local/self-hosted OpenAI-compatible
서버를 위한 번들 provider plugin으로 제공됩니다.

- Provider: `vllm`
- Auth: 선택 사항(서버에 따라 다름)
- 기본 base URL: `http://127.0.0.1:8000/v1`

로컬에서 자동 탐지를 opt in하려면(서버가 auth를 강제하지 않으면 어떤 값이든 동작):

```bash
export VLLM_API_KEY="vllm-local"
```

그다음 모델을 설정하세요(`/v1/models`가 반환한 ID 중 하나로 교체):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

자세한 내용은 [/providers/vllm](/ko/providers/vllm)를 참조하세요.

### SGLang

SGLang은 빠른 self-hosted
OpenAI-compatible 서버를 위한 번들 provider plugin으로 제공됩니다.

- Provider: `sglang`
- Auth: 선택 사항(서버에 따라 다름)
- 기본 base URL: `http://127.0.0.1:30000/v1`

로컬에서 자동 탐지를 opt in하려면(서버가 auth를 강제하지
않으면 어떤 값이든 동작):

```bash
export SGLANG_API_KEY="sglang-local"
```

그다음 모델을 설정하세요(`/v1/models`가 반환한 ID 중 하나로 교체):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

자세한 내용은 [/providers/sglang](/ko/providers/sglang)를 참조하세요.

### 로컬 proxy (LM Studio, vLLM, LiteLLM 등)

예시(OpenAI‑compatible):

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

참고:

- custom providers의 경우 `reasoning`, `input`, `cost`, `contextWindow`, `maxTokens`는 선택 사항입니다.
  생략하면 OpenClaw는 기본적으로 다음을 사용합니다.
  - `reasoning: false`
  - `input: ["text"]`
  - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
  - `contextWindow: 200000`
  - `maxTokens: 8192`
- 권장: proxy/모델 한도에 맞는 명시적인 값을 설정하세요.
- non-native endpoint(`api.openai.com`이 아닌 호스트를 가진 비어 있지 않은 `baseUrl`)에서 `api: "openai-completions"`를 사용할 경우, OpenClaw는 지원되지 않는 `developer` role로 인한 provider 400 오류를 피하기 위해 `compat.supportsDeveloperRole: false`를 강제합니다.
- proxy 스타일 OpenAI-compatible 경로는 네이티브 OpenAI 전용 요청
  shaping도 건너뜁니다. 즉 `service_tier` 없음, Responses `store` 없음,
  prompt-cache 힌트 없음, OpenAI reasoning-compat payload shaping 없음,
  숨겨진 OpenClaw attribution header도 없음.
- `baseUrl`이 비어 있거나 생략되면 OpenClaw는 기본 OpenAI 동작(`api.openai.com`으로 해석됨)을 유지합니다.
- 안전을 위해 non-native `openai-completions` endpoint에서는 명시적 `compat.supportsDeveloperRole: true`도 여전히 override됩니다.

## CLI 예시

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

전체 config 예시는 [/gateway/configuration](/ko/gateway/configuration)을 참조하세요.

## 관련 문서

- [Models](/ko/concepts/models) — 모델 구성 및 alias
- [Model Failover](/ko/concepts/model-failover) — fallback 체인과 재시도 동작
- [Configuration Reference](/ko/gateway/configuration-reference#agent-defaults) — 모델 config 키
- [Providers](/ko/providers) — provider별 설정 가이드
