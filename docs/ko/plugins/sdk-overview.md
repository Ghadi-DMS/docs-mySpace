---
read_when:
    - 어떤 SDK 하위 경로에서 import해야 하는지 알아야 합니다
    - OpenClawPluginApi의 모든 등록 메서드에 대한 참조가 필요합니다
    - 특정 SDK export를 찾고 있습니다
sidebarTitle: SDK Overview
summary: import 맵, 등록 API 참조, 그리고 SDK 아키텍처
title: Plugin SDK 개요
x-i18n:
    generated_at: "2026-04-06T03:11:00Z"
    model: gpt-5.4
    provider: openai
    source_hash: d801641f26f39dc21490d2a69a337ff1affb147141360916b8b58a267e9f822a
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Plugin SDK 개요

plugin SDK는 plugins와 코어 사이의 타입 지정 계약입니다. 이 페이지는
**무엇을 import할지**와 **무엇을 등록할 수 있는지**에 대한 참조입니다.

<Tip>
  **방법 안내를 찾고 있나요?**
  - 첫 plugin이라면? [Getting Started](/ko/plugins/building-plugins)부터 시작하세요
  - Channel plugin이라면? [Channel Plugins](/ko/plugins/sdk-channel-plugins)를 참조하세요
  - Provider plugin이라면? [Provider Plugins](/ko/plugins/sdk-provider-plugins)를 참조하세요
</Tip>

## import 규칙

항상 특정 하위 경로에서 import하세요:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

각 하위 경로는 작고 독립적인 모듈입니다. 이렇게 하면 시작 속도가 빨라지고
순환 종속성 문제가 방지됩니다. 채널별 entry/build helpers에는
`openclaw/plugin-sdk/channel-core`를 우선 사용하고, 더 넓은 우산형 표면과
`buildChannelConfigSchema` 같은 공유 helpers에는 `openclaw/plugin-sdk/core`를 유지하세요.

`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` 같은
provider 이름 기반 편의 seam이나 채널 브랜딩 helper seam을
추가하거나 이에 의존하지 마세요. 번들 plugins는
자체 `api.ts` 또는 `runtime-api.ts` 배럴 내부에서 일반적인
SDK 하위 경로를 조합해야 하며, 코어는 실제로 크로스 채널 필요가 있을 때에만
그 plugin 로컬 배럴을 사용하거나 좁은 일반 SDK
계약을 추가해야 합니다.

생성된 export 맵에는 여전히 `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`,
`plugin-sdk/zalo`, `plugin-sdk/zalo-setup`, `plugin-sdk/matrix*` 같은
소수의 번들 plugin helper seam이 포함되어 있습니다. 이러한
하위 경로는 번들 plugin 유지보수와 호환성만을 위해 존재하며,
아래의 일반 표에서는 의도적으로 생략되었고 새로운 서드파티 plugins에 권장되는
import 경로가 아닙니다.

## 하위 경로 참조

용도별로 그룹화한 가장 일반적으로 사용되는 하위 경로입니다. 생성된 전체 목록 200개 이상의
하위 경로는 `scripts/lib/plugin-sdk-entrypoints.json`에 있습니다.

예약된 번들 plugin helper 하위 경로도 여전히 생성된 목록에 나타납니다.
문서 페이지가 하나를 공개용으로 명시적으로 홍보하지 않는 한,
이것들은 구현 세부 사항/호환성 표면으로 취급하세요.

### Plugin entry

| 하위 경로                  | 주요 exports                                                                                                                          |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                   |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                      |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                     |

<AccordionGroup>
  <Accordion title="채널 하위 경로">
    | 하위 경로 | 주요 exports |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | 루트 `openclaw.json` Zod 스키마 export (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, 그리고 `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | 공유 setup wizard helpers, allowlist 프롬프트, setup 상태 빌더 |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | 다중 계정 config/action-gate helpers, 기본 계정 폴백 helpers |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, account-id 정규화 helpers |
    | `plugin-sdk/account-resolution` | 계정 조회 + 기본 폴백 helpers |
    | `plugin-sdk/account-helpers` | 좁은 범위의 account-list/account-action helpers |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | 채널 config 스키마 타입 |
    | `plugin-sdk/telegram-command-config` | 번들 계약 폴백이 있는 Telegram 사용자 지정 command 정규화/검증 helpers |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | 공유 인바운드 라우트 + envelope 빌더 helpers |
    | `plugin-sdk/inbound-reply-dispatch` | 공유 인바운드 기록 및 디스패치 helpers |
    | `plugin-sdk/messaging-targets` | 대상 파싱/매칭 helpers |
    | `plugin-sdk/outbound-media` | 공유 아웃바운드 미디어 로드 helpers |
    | `plugin-sdk/outbound-runtime` | 아웃바운드 ID/send delegate helpers |
    | `plugin-sdk/thread-bindings-runtime` | 스레드 바인딩 수명 주기 및 adapter helpers |
    | `plugin-sdk/agent-media-payload` | 레거시 agent 미디어 payload 빌더 |
    | `plugin-sdk/conversation-runtime` | 대화/스레드 바인딩, pairing, 구성된 바인딩 helpers |
    | `plugin-sdk/runtime-config-snapshot` | 런타임 config 스냅샷 helper |
    | `plugin-sdk/runtime-group-policy` | 런타임 그룹 정책 해석 helpers |
    | `plugin-sdk/channel-status` | 공유 채널 상태 스냅샷/요약 helpers |
    | `plugin-sdk/channel-config-primitives` | 좁은 범위의 채널 config-schema 기본 요소 |
    | `plugin-sdk/channel-config-writes` | 채널 config 쓰기 권한 부여 helpers |
    | `plugin-sdk/channel-plugin-common` | 공유 채널 plugin prelude exports |
    | `plugin-sdk/allowlist-config-edit` | allowlist config 편집/읽기 helpers |
    | `plugin-sdk/group-access` | 공유 그룹 접근 결정 helpers |
    | `plugin-sdk/direct-dm` | 공유 direct-DM auth/guard helpers |
    | `plugin-sdk/interactive-runtime` | 대화형 reply payload 정규화/축소 helpers |
    | `plugin-sdk/channel-inbound` | debounce, 멘션 매칭, envelope helpers |
    | `plugin-sdk/channel-send-result` | reply 결과 타입 |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | 대상 파싱/매칭 helpers |
    | `plugin-sdk/channel-contract` | 채널 계약 타입 |
    | `plugin-sdk/channel-feedback` | 피드백/반응 wiring |
  </Accordion>

  <Accordion title="Provider 하위 경로">
    | 하위 경로 | 주요 exports |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | 엄선된 로컬/셀프호스팅 provider setup helpers |
    | `plugin-sdk/self-hosted-provider-setup` | 집중된 OpenAI 호환 셀프호스팅 provider setup helpers |
    | `plugin-sdk/provider-auth-runtime` | provider plugins용 런타임 API-key 해석 helpers |
    | `plugin-sdk/provider-auth-api-key` | API-key onboarding/profile 쓰기 helpers |
    | `plugin-sdk/provider-auth-result` | 표준 OAuth auth-result 빌더 |
    | `plugin-sdk/provider-auth-login` | provider plugins용 공유 대화형 로그인 helpers |
    | `plugin-sdk/provider-env-vars` | provider auth env-var 조회 helpers |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, 공유 replay-policy 빌더, provider-endpoint helpers, 그리고 `normalizeNativeXaiModelId` 같은 model-id 정규화 helpers |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | 일반적인 provider HTTP/endpoint capability helpers |
    | `plugin-sdk/provider-web-fetch` | 웹 fetch provider 등록/캐시 helpers |
    | `plugin-sdk/provider-web-search` | 웹 검색 provider 등록/캐시/config helpers |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini 스키마 정리 + 진단, 그리고 `resolveXaiModelCompatPatch` / `applyXaiModelCompat` 같은 xAI compat helpers |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` 및 유사한 helpers |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, 스트림 wrapper 타입, 그리고 공유 Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot wrapper helpers |
    | `plugin-sdk/provider-onboard` | 온보딩 config 패치 helpers |
    | `plugin-sdk/global-singleton` | 프로세스 로컬 singleton/map/cache helpers |
  </Accordion>

  <Accordion title="Auth 및 보안 하위 경로">
    | 하위 경로 | 주요 exports |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, command registry helpers, 발신자 인증 helpers |
    | `plugin-sdk/approval-auth-runtime` | 승인자 해석 및 same-chat action-auth helpers |
    | `plugin-sdk/approval-client-runtime` | 네이티브 exec approval profile/filter helpers |
    | `plugin-sdk/approval-delivery-runtime` | 네이티브 approval capability/delivery adapters |
    | `plugin-sdk/approval-native-runtime` | 네이티브 approval target + account-binding helpers |
    | `plugin-sdk/approval-reply-runtime` | exec/plugin approval reply payload helpers |
    | `plugin-sdk/command-auth-native` | 네이티브 command auth + 네이티브 session-target helpers |
    | `plugin-sdk/command-detection` | 공유 command 감지 helpers |
    | `plugin-sdk/command-surface` | command-body 정규화 및 command-surface helpers |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/security-runtime` | 공유 신뢰, DM gating, 외부 콘텐츠, secret 수집 helpers |
    | `plugin-sdk/ssrf-policy` | 호스트 allowlist 및 사설 네트워크 SSRF 정책 helpers |
    | `plugin-sdk/ssrf-runtime` | pinned-dispatcher, SSRF 보호 fetch, 그리고 SSRF 정책 helpers |
    | `plugin-sdk/secret-input` | secret 입력 파싱 helpers |
    | `plugin-sdk/webhook-ingress` | webhook 요청/대상 helpers |
    | `plugin-sdk/webhook-request-guards` | 요청 본문 크기/시간 초과 helpers |
  </Accordion>

  <Accordion title="런타임 및 저장소 하위 경로">
    | 하위 경로 | 주요 exports |
    | --- | --- |
    | `plugin-sdk/runtime` | 넓은 범위의 런타임/로깅/백업/plugin 설치 helpers |
    | `plugin-sdk/runtime-env` | 좁은 범위의 런타임 env, logger, timeout, retry, backoff helpers |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | 공유 plugin command/hook/http/interactive helpers |
    | `plugin-sdk/hook-runtime` | 공유 webhook/internal hook 파이프라인 helpers |
    | `plugin-sdk/lazy-runtime` | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeSurface` 같은 지연 런타임 import/binding helpers |
    | `plugin-sdk/process-runtime` | 프로세스 exec helpers |
    | `plugin-sdk/cli-runtime` | CLI 서식, 대기, 버전 helpers |
    | `plugin-sdk/gateway-runtime` | gateway client 및 channel-status 패치 helpers |
    | `plugin-sdk/config-runtime` | config 로드/쓰기 helpers |
    | `plugin-sdk/telegram-command-config` | 번들 Telegram 계약 표면을 사용할 수 없을 때도 Telegram command 이름/설명 정규화 및 중복/충돌 검사 |
    | `plugin-sdk/approval-runtime` | exec/plugin approval helpers, approval-capability 빌더, auth/profile helpers, 네이티브 라우팅/런타임 helpers |
    | `plugin-sdk/reply-runtime` | 공유 인바운드/reply 런타임 helpers, chunking, dispatch, heartbeat, reply planner |
    | `plugin-sdk/reply-dispatch-runtime` | 좁은 범위의 reply dispatch/finalize helpers |
    | `plugin-sdk/reply-history` | `buildHistoryContext`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` 같은 공유 단기 창 reply-history helpers |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | 좁은 범위의 텍스트/Markdown chunking helpers |
    | `plugin-sdk/session-store-runtime` | 세션 저장소 경로 + updated-at helpers |
    | `plugin-sdk/state-paths` | state/OAuth dir 경로 helpers |
    | `plugin-sdk/routing` | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId` 같은 라우트/session-key/account 바인딩 helpers |
    | `plugin-sdk/status-helpers` | 공유 채널/계정 상태 요약 helpers, 런타임 state 기본값, 이슈 메타데이터 helpers |
    | `plugin-sdk/target-resolver-runtime` | 공유 대상 해석 helpers |
    | `plugin-sdk/string-normalization-runtime` | slug/string 정규화 helpers |
    | `plugin-sdk/request-url` | fetch/request 유사 입력에서 문자열 URL 추출 |
    | `plugin-sdk/run-command` | 정규화된 stdout/stderr 결과를 가진 시간 제한 command 실행기 |
    | `plugin-sdk/param-readers` | 공통 tool/CLI 파라미터 읽기 helpers |
    | `plugin-sdk/tool-send` | tool 인수에서 정식 send 대상 필드 추출 |
    | `plugin-sdk/temp-path` | 공유 임시 다운로드 경로 helpers |
    | `plugin-sdk/logging-core` | 서브시스템 logger 및 redaction helpers |
    | `plugin-sdk/markdown-table-runtime` | Markdown 테이블 모드 helpers |
    | `plugin-sdk/json-store` | 작은 JSON state 읽기/쓰기 helpers |
    | `plugin-sdk/file-lock` | 재진입 가능한 파일 lock helpers |
    | `plugin-sdk/persistent-dedupe` | 디스크 기반 dedupe 캐시 helpers |
    | `plugin-sdk/acp-runtime` | ACP 런타임/세션 및 reply-dispatch helpers |
    | `plugin-sdk/agent-config-primitives` | 좁은 범위의 agent 런타임 config-schema 기본 요소 |
    | `plugin-sdk/boolean-param` | 느슨한 불리언 파라미터 읽기 |
    | `plugin-sdk/dangerous-name-runtime` | 위험한 이름 매칭 해석 helpers |
    | `plugin-sdk/device-bootstrap` | 디바이스 bootstrap 및 pairing 토큰 helpers |
    | `plugin-sdk/extension-shared` | 공유 passive-channel 및 상태 helper 기본 요소 |
    | `plugin-sdk/models-provider-runtime` | `/models` command/provider reply helpers |
    | `plugin-sdk/skill-commands-runtime` | Skills command 목록 helpers |
    | `plugin-sdk/native-command-registry` | 네이티브 command registry/build/serialize helpers |
    | `plugin-sdk/provider-zai-endpoint` | Z.AI 엔드포인트 감지 helpers |
    | `plugin-sdk/infra-runtime` | 시스템 이벤트/heartbeat helpers |
    | `plugin-sdk/collection-runtime` | 작은 bounded cache helpers |
    | `plugin-sdk/diagnostic-runtime` | 진단 플래그 및 이벤트 helpers |
    | `plugin-sdk/error-runtime` | 오류 그래프, 서식 지정, 공유 오류 분류 helpers, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | 래핑된 fetch, 프록시, pinned lookup helpers |
    | `plugin-sdk/host-runtime` | 호스트명 및 SCP 호스트 정규화 helpers |
    | `plugin-sdk/retry-runtime` | retry config 및 retry 실행 helpers |
    | `plugin-sdk/agent-runtime` | agent dir/ID/workspace helpers |
    | `plugin-sdk/directory-runtime` | config 기반 디렉터리 조회/dedup |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Capability 및 테스트 하위 경로">
    | 하위 경로 | 주요 exports |
    | --- | --- |
    | `plugin-sdk/media-runtime` | 공유 미디어 fetch/transform/store helpers와 미디어 payload 빌더 |
    | `plugin-sdk/media-understanding` | 미디어 이해 provider 타입과 provider용 이미지/오디오 helper exports |
    | `plugin-sdk/text-runtime` | assistant-visible-text 제거, Markdown 렌더/chunking/테이블 helpers, redaction helpers, directive-tag helpers, safe-text 유틸리티 같은 공유 텍스트/Markdown/로깅 helpers |
    | `plugin-sdk/text-chunking` | 아웃바운드 텍스트 chunking helper |
    | `plugin-sdk/speech` | speech provider 타입과 provider용 directive, registry, 검증 helpers |
    | `plugin-sdk/speech-core` | 공유 speech provider 타입, registry, directive, 정규화 helpers |
    | `plugin-sdk/realtime-transcription` | realtime transcription provider 타입 및 registry helpers |
    | `plugin-sdk/realtime-voice` | realtime voice provider 타입 및 registry helpers |
    | `plugin-sdk/image-generation` | 이미지 생성 provider 타입 |
    | `plugin-sdk/image-generation-core` | 공유 image-generation 타입, failover, auth, registry helpers |
    | `plugin-sdk/music-generation` | 음악 생성 provider/request/result 타입 |
    | `plugin-sdk/music-generation-core` | 공유 music-generation 타입, failover helpers, provider 조회, model-ref 파싱 |
    | `plugin-sdk/video-generation` | 비디오 생성 provider/request/result 타입 |
    | `plugin-sdk/video-generation-core` | 공유 video-generation 타입, failover helpers, provider 조회, model-ref 파싱 |
    | `plugin-sdk/webhook-targets` | webhook 대상 registry 및 route-install helpers |
    | `plugin-sdk/webhook-path` | webhook 경로 정규화 helpers |
    | `plugin-sdk/web-media` | 공유 원격/로컬 미디어 로드 helpers |
    | `plugin-sdk/zod` | plugin SDK 소비자용으로 재export된 `zod` |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="메모리 하위 경로">
    | 하위 경로 | 주요 exports |
    | --- | --- |
    | `plugin-sdk/memory-core` | manager/config/file/CLI helpers용 번들 memory-core helper 표면 |
    | `plugin-sdk/memory-core-engine-runtime` | 메모리 index/search 런타임 파사드 |
    | `plugin-sdk/memory-core-host-engine-foundation` | 메모리 호스트 기반 엔진 exports |
    | `plugin-sdk/memory-core-host-engine-embeddings` | 메모리 호스트 임베딩 엔진 exports |
    | `plugin-sdk/memory-core-host-engine-qmd` | 메모리 호스트 QMD 엔진 exports |
    | `plugin-sdk/memory-core-host-engine-storage` | 메모리 호스트 저장 엔진 exports |
    | `plugin-sdk/memory-core-host-multimodal` | 메모리 호스트 멀티모달 helpers |
    | `plugin-sdk/memory-core-host-query` | 메모리 호스트 쿼리 helpers |
    | `plugin-sdk/memory-core-host-secret` | 메모리 호스트 secret helpers |
    | `plugin-sdk/memory-core-host-status` | 메모리 호스트 상태 helpers |
    | `plugin-sdk/memory-core-host-runtime-cli` | 메모리 호스트 CLI 런타임 helpers |
    | `plugin-sdk/memory-core-host-runtime-core` | 메모리 호스트 코어 런타임 helpers |
    | `plugin-sdk/memory-core-host-runtime-files` | 메모리 호스트 파일/런타임 helpers |
    | `plugin-sdk/memory-lancedb` | 번들 memory-lancedb helper 표면 |
  </Accordion>

  <Accordion title="예약된 bundled-helper 하위 경로">
    | 계열 | 현재 하위 경로 | 의도된 용도 |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | 번들 browser plugin 지원 helpers (`browser-support`는 호환성 배럴로 유지) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | 번들 Matrix helper/런타임 표면 |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | 번들 LINE helper/런타임 표면 |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | 번들 IRC helper 표면 |
    | 채널별 helpers | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | 번들 채널 호환성/helper seam |
    | Auth/plugin별 helpers | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | 번들 기능/plugin helper seam; `plugin-sdk/github-copilot-token`은 현재 `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken`, `resolveCopilotApiToken`을 export함 |
  </Accordion>
</AccordionGroup>

## 등록 API

`register(api)` 콜백은 다음
메서드를 가진 `OpenClawPluginApi` 객체를 받습니다:

### Capability 등록

| 메서드                                           | 등록하는 항목                   |
| ------------------------------------------------ | ------------------------------- |
| `api.registerProvider(...)`                      | 텍스트 추론(LLM)                |
| `api.registerChannel(...)`                       | 메시징 채널                     |
| `api.registerSpeechProvider(...)`                | 텍스트 음성 변환 / STT 합성     |
| `api.registerRealtimeTranscriptionProvider(...)` | 스트리밍 realtime transcription |
| `api.registerRealtimeVoiceProvider(...)`         | 양방향 realtime voice 세션      |
| `api.registerMediaUnderstandingProvider(...)`    | 이미지/오디오/비디오 분석       |
| `api.registerImageGenerationProvider(...)`       | 이미지 생성                     |
| `api.registerMusicGenerationProvider(...)`       | 음악 생성                       |
| `api.registerVideoGenerationProvider(...)`       | 비디오 생성                     |
| `api.registerWebFetchProvider(...)`              | 웹 fetch / 스크래핑 provider    |
| `api.registerWebSearchProvider(...)`             | 웹 검색                         |

### 도구와 commands

| 메서드                          | 등록하는 항목                                |
| ------------------------------- | -------------------------------------------- |
| `api.registerTool(tool, opts?)` | agent 도구(필수 또는 `{ optional: true }`)   |
| `api.registerCommand(def)`      | 사용자 지정 command(LLM 우회)                |

### 인프라

| 메서드                                         | 등록하는 항목          |
| ---------------------------------------------- | ---------------------- |
| `api.registerHook(events, handler, opts?)`     | 이벤트 hook            |
| `api.registerHttpRoute(params)`                | Gateway HTTP 엔드포인트 |
| `api.registerGatewayMethod(name, handler)`     | Gateway RPC 메서드     |
| `api.registerCli(registrar, opts?)`            | CLI 하위 명령          |
| `api.registerService(service)`                 | 백그라운드 서비스      |
| `api.registerInteractiveHandler(registration)` | 대화형 핸들러          |

예약된 코어 admin 네임스페이스(`config.*`, `exec.approvals.*`, `wizard.*`,
`update.*`)는 plugin이 더 좁은 gateway 메서드 범위를 할당하려 해도
항상 `operator.admin`으로 유지됩니다. plugin 소유 메서드에는
plugin별 접두사를 사용하세요.

### CLI 등록 메타데이터

`api.registerCli(registrar, opts?)`는 두 종류의 최상위 메타데이터를 받습니다:

- `commands`: registrar가 소유한 명시적 command 루트
- `descriptors`: 루트 CLI 도움말,
  라우팅, 지연 plugin CLI 등록에 사용되는 파싱 시점 command descriptor

plugin command를 일반 루트 CLI 경로에서 계속 지연 로드 상태로 유지하려면,
해당 registrar가 노출하는 모든 최상위 command 루트를 포괄하는 `descriptors`를 제공하세요.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Manage Matrix accounts, verification, devices, and profile state",
        hasSubcommands: true,
      },
    ],
  },
);
```

지연 루트 CLI 등록이 필요하지 않을 때만 `commands`를 단독으로 사용하세요.
이 eager 호환 경로는 계속 지원되지만, 파싱 시점 지연 로딩을 위한
descriptor 기반 placeholder는 설치하지 않습니다.

### 배타적 슬롯

| 메서드                                     | 등록하는 항목                         |
| ------------------------------------------ | ------------------------------------- |
| `api.registerContextEngine(id, factory)`   | 컨텍스트 엔진(한 번에 하나만 활성)    |
| `api.registerMemoryPromptSection(builder)` | 메모리 프롬프트 섹션 빌더             |
| `api.registerMemoryFlushPlan(resolver)`    | 메모리 flush 계획 해석기              |
| `api.registerMemoryRuntime(runtime)`       | 메모리 런타임 adapter                 |

### 메모리 임베딩 adapters

| 메서드                                         | 등록하는 항목                               |
| ---------------------------------------------- | ------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | 활성 plugin용 메모리 임베딩 adapter         |

- `registerMemoryPromptSection`, `registerMemoryFlushPlan`, 그리고
  `registerMemoryRuntime`는 메모리 plugins 전용입니다.
- `registerMemoryEmbeddingProvider`는 활성 메모리 plugin이 하나 이상의
  임베딩 adapter ID(예: `openai`, `gemini`, 또는 사용자 지정 plugin 정의 ID)를 등록할 수 있게 합니다.
- `agents.defaults.memorySearch.provider` 및
  `agents.defaults.memorySearch.fallback` 같은 사용자 config는
  등록된 해당 adapter ID에 대해 해석됩니다.

### 이벤트 및 수명 주기

| 메서드                                       | 수행하는 작업            |
| -------------------------------------------- | ------------------------ |
| `api.on(hookName, handler, opts?)`           | 타입 지정 수명 주기 hook |
| `api.onConversationBindingResolved(handler)` | 대화 바인딩 콜백         |

### Hook 결정 의미론

- `before_tool_call`: `{ block: true }`를 반환하면 종결 결정입니다. 어떤 핸들러든 이것을 설정하면 더 낮은 우선순위 핸들러는 건너뜁니다.
- `before_tool_call`: `{ block: false }`를 반환하면 override가 아니라 결정 없음(`block` 생략과 동일)으로 처리됩니다.
- `before_install`: `{ block: true }`를 반환하면 종결 결정입니다. 어떤 핸들러든 이것을 설정하면 더 낮은 우선순위 핸들러는 건너뜁니다.
- `before_install`: `{ block: false }`를 반환하면 override가 아니라 결정 없음(`block` 생략과 동일)으로 처리됩니다.
- `reply_dispatch`: `{ handled: true, ... }`를 반환하면 종결 결정입니다. 어떤 핸들러든 dispatch를 차지하면 더 낮은 우선순위 핸들러와 기본 모델 dispatch 경로는 건너뜁니다.
- `message_sending`: `{ cancel: true }`를 반환하면 종결 결정입니다. 어떤 핸들러든 이것을 설정하면 더 낮은 우선순위 핸들러는 건너뜁니다.
- `message_sending`: `{ cancel: false }`를 반환하면 override가 아니라 결정 없음(`cancel` 생략과 동일)으로 처리됩니다.

### API 객체 필드

| 필드                     | 타입                      | 설명                                                                                       |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------ |
| `api.id`                 | `string`                  | plugin ID                                                                                  |
| `api.name`               | `string`                  | 표시 이름                                                                                  |
| `api.version`            | `string?`                 | plugin 버전(선택 사항)                                                                     |
| `api.description`        | `string?`                 | plugin 설명(선택 사항)                                                                     |
| `api.source`             | `string`                  | plugin 소스 경로                                                                           |
| `api.rootDir`            | `string?`                 | plugin 루트 디렉터리(선택 사항)                                                            |
| `api.config`             | `OpenClawConfig`          | 현재 config 스냅샷(가능한 경우 활성 인메모리 런타임 스냅샷)                                |
| `api.pluginConfig`       | `Record<string, unknown>` | `plugins.entries.<id>.config`의 plugin별 config                                            |
| `api.runtime`            | `PluginRuntime`           | [Runtime helpers](/ko/plugins/sdk-runtime)                                                    |
| `api.logger`             | `PluginLogger`            | 범위가 지정된 logger (`debug`, `info`, `warn`, `error`)                                    |
| `api.registrationMode`   | `PluginRegistrationMode`  | 현재 로드 모드; `"setup-runtime"`은 전체 entry 전의 경량 시작/setup 창입니다               |
| `api.resolvePath(input)` | `(string) => string`      | plugin 루트를 기준으로 경로 해석                                                           |

## 내부 모듈 규칙

plugin 내부에서는 내부 import에 로컬 배럴 파일을 사용하세요:

```
my-plugin/
  api.ts            # 외부 소비자를 위한 공개 exports
  runtime-api.ts    # 내부 전용 런타임 exports
  index.ts          # Plugin entry point
  setup-entry.ts    # 경량 setup 전용 entry(선택 사항)
```

<Warning>
  프로덕션 코드에서 `openclaw/plugin-sdk/<your-plugin>`을 통해
  자신의 plugin을 import하지 마세요. 내부 import는 `./api.ts` 또는
  `./runtime-api.ts`로 연결하세요. SDK 경로는 외부 계약 전용입니다.
</Warning>

파사드 로드 방식의 번들 plugin 공개 표면(`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` 및 유사한 공개 entry 파일)은 이제
OpenClaw가 이미 실행 중일 때 활성 런타임 config 스냅샷을 우선 사용합니다.
아직 런타임 스냅샷이 없으면 디스크에 있는 해석된 config 파일로 폴백합니다.

Provider plugins는 helper가 의도적으로 provider 전용이고
아직 일반적인 SDK 하위 경로에 속하지 않을 때 좁은 plugin 로컬 계약 배럴을 노출할 수도 있습니다.
현재 번들 예시: Anthropic provider는 Claude
stream helpers를 일반적인
`plugin-sdk/*` 계약으로 승격하는 대신 자체 공개 `api.ts` / `contract-api.ts` seam에
유지합니다.

현재의 다른 번들 예시:

- `@openclaw/openai-provider`: `api.ts`는 provider builders,
  기본 모델 helpers, 그리고 realtime provider builders를 export합니다
- `@openclaw/openrouter-provider`: `api.ts`는 provider builder와
  onboarding/config helpers를 export합니다

<Warning>
  extension 프로덕션 코드도 `openclaw/plugin-sdk/<other-plugin>`
  import를 피해야 합니다. helper가 정말 공유되어야 한다면, 두 plugin을 결합하는 대신
  `openclaw/plugin-sdk/speech`, `.../provider-model-shared` 또는 다른
  capability 지향 표면 같은 중립적인 SDK 하위 경로로 승격하세요.
</Warning>

## 관련

- [Entry Points](/ko/plugins/sdk-entrypoints) — `definePluginEntry` 및 `defineChannelPluginEntry` 옵션
- [Runtime Helpers](/ko/plugins/sdk-runtime) — 전체 `api.runtime` 네임스페이스 참조
- [Setup and Config](/ko/plugins/sdk-setup) — 패키징, manifests, config 스키마
- [Testing](/ko/plugins/sdk-testing) — 테스트 유틸리티 및 lint 규칙
- [SDK Migration](/ko/plugins/sdk-migration) — deprecated 표면에서의 마이그레이션
- [Plugin Internals](/ko/plugins/architecture) — 심층 아키텍처 및 capability 모델
