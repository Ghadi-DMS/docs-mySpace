---
read_when:
    - 어떤 SDK 하위 경로에서 import해야 하는지 알아야 합니다
    - OpenClawPluginApi의 모든 등록 메서드에 대한 참조가 필요합니다
    - 특정 SDK export를 찾고 있습니다
sidebarTitle: SDK Overview
summary: import 맵, 등록 API 참조, SDK 아키텍처
title: Plugin SDK 개요
x-i18n:
    generated_at: "2026-04-09T01:31:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: bf205af060971931df97dca4af5110ce173d2b7c12f56ad7c62d664a402f2381
    source_path: plugins/sdk-overview.md
    workflow: 15
---

# Plugin SDK 개요

Plugin SDK는 plugins와 core 사이의 타입이 지정된 계약입니다. 이 페이지는 **무엇을 import할지**와 **무엇을 등록할 수 있는지**에 대한 참조입니다.

<Tip>
  **사용 방법 가이드를 찾고 있나요?**
  - 첫 plugin이라면? [Getting Started](/ko/plugins/building-plugins)부터 시작하세요
  - Channel plugin이라면? [Channel Plugins](/ko/plugins/sdk-channel-plugins)를 참조하세요
  - Provider plugin이라면? [Provider Plugins](/ko/plugins/sdk-provider-plugins)를 참조하세요
</Tip>

## import 규칙

항상 특정 하위 경로에서 import하세요.

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

각 하위 경로는 작고 독립적인 모듈입니다. 이렇게 하면 시작 속도가 빨라지고 순환 의존성 문제를 방지할 수 있습니다. channel 전용 entry/build helper의 경우 `openclaw/plugin-sdk/channel-core`를 우선 사용하고, 더 넓은 umbrella 표면과 `buildChannelConfigSchema` 같은 공용 helper에는 `openclaw/plugin-sdk/core`를 유지하세요.

`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`, `openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp` 또는 channel 브랜드 helper seam 같은 provider 이름 기반 convenience seam을 추가하거나 이에 의존하지 마세요. 번들 plugins는 자체 `api.ts` 또는 `runtime-api.ts` barrel 안에서 일반적인 SDK 하위 경로를 조합해야 하며, core는 해당 plugin 로컬 barrel을 사용하거나 필요가 진짜 cross-channel일 때 좁고 일반적인 SDK 계약을 추가해야 합니다.

생성된 export 맵에는 여전히 `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`, `plugin-sdk/zalo-setup`, `plugin-sdk/matrix*` 같은 소수의 번들 plugin helper seam이 포함되어 있습니다. 이러한 하위 경로는 번들 plugin 유지 관리 및 호환성 전용으로 존재하며, 아래 공통 표에서는 의도적으로 생략되었고 새로운 서드파티 plugins에 권장되는 import 경로도 아닙니다.

## 하위 경로 참조

가장 자주 사용하는 하위 경로를 용도별로 묶었습니다. 200개가 넘는 하위 경로의 전체 생성 목록은 `scripts/lib/plugin-sdk-entrypoints.json`에 있습니다.

예약된 번들 plugin helper 하위 경로도 이 생성 목록에 계속 나타납니다. 문서 페이지에서 명시적으로 public으로 권장하지 않는 한, 이를 구현 세부 사항/호환성 표면으로 취급하세요.

### Plugin entry

| Subpath                     | 주요 exports                                                                                                                          |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`   | `definePluginEntry`                                                                                                                   |
| `plugin-sdk/core`           | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema` |
| `plugin-sdk/config-schema`  | `OpenClawSchema`                                                                                                                      |
| `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry`                                                                                                     |

<AccordionGroup>
  <Accordion title="Channel 하위 경로">
    | Subpath | 주요 exports |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
    | `plugin-sdk/config-schema` | 루트 `openclaw.json` Zod 스키마 export (`OpenClawSchema`) |
    | `plugin-sdk/channel-setup` | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, 그리고 `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
    | `plugin-sdk/setup` | 공용 setup wizard helper, 허용 목록 프롬프트, setup 상태 빌더 |
    | `plugin-sdk/setup-runtime` | `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-adapter-runtime` | `createEnvPatchedAccountSetupAdapter` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
    | `plugin-sdk/account-core` | 다중 계정 config/action-gate helper, 기본 계정 폴백 helper |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`, account-id 정규화 helper |
    | `plugin-sdk/account-resolution` | 계정 조회 + 기본 폴백 helper |
    | `plugin-sdk/account-helpers` | 좁은 범위의 account-list/account-action helper |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | `createChannelReplyPipeline` |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter` |
    | `plugin-sdk/channel-config-schema` | Channel config 스키마 타입 |
    | `plugin-sdk/telegram-command-config` | 번들 계약 폴백이 포함된 Telegram 사용자 지정 command 정규화/검증 helper |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-lifecycle` | `createAccountStatusSink` |
    | `plugin-sdk/inbound-envelope` | 공용 수신 라우트 + envelope 빌더 helper |
    | `plugin-sdk/inbound-reply-dispatch` | 공용 수신 record-and-dispatch helper |
    | `plugin-sdk/messaging-targets` | 대상 파싱/매칭 helper |
    | `plugin-sdk/outbound-media` | 공용 송신 미디어 로딩 helper |
    | `plugin-sdk/outbound-runtime` | 송신 identity/send delegate helper |
    | `plugin-sdk/thread-bindings-runtime` | 스레드 바인딩 lifecycle 및 adapter helper |
    | `plugin-sdk/agent-media-payload` | 레거시 agent 미디어 payload 빌더 |
    | `plugin-sdk/conversation-runtime` | 대화/스레드 바인딩, 페어링, configured-binding helper |
    | `plugin-sdk/runtime-config-snapshot` | 런타임 config 스냅샷 helper |
    | `plugin-sdk/runtime-group-policy` | 런타임 group-policy 해석 helper |
    | `plugin-sdk/channel-status` | 공용 channel 상태 스냅샷/요약 helper |
    | `plugin-sdk/channel-config-primitives` | 좁은 범위의 channel config-schema primitives |
    | `plugin-sdk/channel-config-writes` | Channel config-write 권한 부여 helper |
    | `plugin-sdk/channel-plugin-common` | 공용 channel plugin prelude exports |
    | `plugin-sdk/allowlist-config-edit` | 허용 목록 config 편집/읽기 helper |
    | `plugin-sdk/group-access` | 공용 group-access 결정 helper |
    | `plugin-sdk/direct-dm` | 공용 direct-DM auth/guard helper |
    | `plugin-sdk/interactive-runtime` | 대화형 reply payload 정규화/축소 helper |
    | `plugin-sdk/channel-inbound` | 수신 debounce, mention 매칭, mention-policy helper 및 envelope helper |
    | `plugin-sdk/channel-send-result` | Reply 결과 타입 |
    | `plugin-sdk/channel-actions` | `createMessageToolButtonsSchema`, `createMessageToolCardSchema` |
    | `plugin-sdk/channel-targets` | 대상 파싱/매칭 helper |
    | `plugin-sdk/channel-contract` | Channel 계약 타입 |
    | `plugin-sdk/channel-feedback` | 피드백/reaction wiring |
    | `plugin-sdk/channel-secret-runtime` | `collectSimpleChannelFieldAssignments`, `getChannelSurface`, `pushAssignment` 및 secret 대상 타입 같은 좁은 범위의 secret-contract helper |
  </Accordion>

  <Accordion title="Provider 하위 경로">
    | Subpath | 주요 exports |
    | --- | --- |
    | `plugin-sdk/provider-entry` | `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | 선별된 local/self-hosted provider setup helper |
    | `plugin-sdk/self-hosted-provider-setup` | 집중된 OpenAI 호환 self-hosted provider setup helper |
    | `plugin-sdk/cli-backend` | CLI backend 기본값 + watchdog 상수 |
    | `plugin-sdk/provider-auth-runtime` | provider plugins용 런타임 API 키 해석 helper |
    | `plugin-sdk/provider-auth-api-key` | `upsertApiKeyProfile` 같은 API 키 onboarding/profile-write helper |
    | `plugin-sdk/provider-auth-result` | 표준 OAuth auth-result 빌더 |
    | `plugin-sdk/provider-auth-login` | provider plugins용 공용 대화형 로그인 helper |
    | `plugin-sdk/provider-env-vars` | Provider auth env-var 조회 helper |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`, `ensureApiKeyFromOptionEnvOrPrompt`, `upsertAuthProfile`, `upsertApiKeyProfile`, `writeOAuthCredentials` |
    | `plugin-sdk/provider-model-shared` | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, 공용 replay-policy 빌더, provider-endpoint helper, `normalizeNativeXaiModelId` 같은 model-id 정규화 helper |
    | `plugin-sdk/provider-catalog-shared` | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | 일반적인 provider HTTP/endpoint capability helper |
    | `plugin-sdk/provider-web-fetch-contract` | `enablePluginInConfig` 및 `WebFetchProviderPlugin` 같은 좁은 범위의 web-fetch config/selection 계약 helper |
    | `plugin-sdk/provider-web-fetch` | Web-fetch provider 등록/cache helper |
    | `plugin-sdk/provider-web-search-config-contract` | plugin-enable wiring이 필요 없는 provider를 위한 좁은 범위의 web-search config/credential helper |
    | `plugin-sdk/provider-web-search-contract` | `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, 범위가 지정된 credential setter/getter 같은 좁은 범위의 web-search config/credential 계약 helper |
    | `plugin-sdk/provider-web-search` | Web-search provider 등록/cache/런타임 helper |
    | `plugin-sdk/provider-tools` | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini 스키마 정리 + 진단, `resolveXaiModelCompatPatch` / `applyXaiModelCompat` 같은 xAI 호환 helper |
    | `plugin-sdk/provider-usage` | `fetchClaudeUsage` 등 |
    | `plugin-sdk/provider-stream` | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, stream wrapper 타입, 공용 Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot wrapper helper |
    | `plugin-sdk/provider-onboard` | Onboarding config patch helper |
    | `plugin-sdk/global-singleton` | 프로세스 로컬 singleton/map/cache helper |
  </Accordion>

  <Accordion title="Auth 및 보안 하위 경로">
    | Subpath | 주요 exports |
    | --- | --- |
    | `plugin-sdk/command-auth` | `resolveControlCommandGate`, command 레지스트리 helper, sender-authorization helper |
    | `plugin-sdk/command-status` | `buildCommandsMessagePaginated`, `buildHelpMessage` 같은 command/help 메시지 빌더 |
    | `plugin-sdk/approval-auth-runtime` | 승인자 해석 및 same-chat action-auth helper |
    | `plugin-sdk/approval-client-runtime` | 네이티브 exec 승인 profile/filter helper |
    | `plugin-sdk/approval-delivery-runtime` | 네이티브 승인 capability/delivery adapter |
    | `plugin-sdk/approval-gateway-runtime` | 공용 승인 gateway-resolution helper |
    | `plugin-sdk/approval-handler-adapter-runtime` | 핫 channel entrypoint용 경량 네이티브 승인 adapter 로딩 helper |
    | `plugin-sdk/approval-handler-runtime` | 더 넓은 범위의 승인 handler 런타임 helper. 좁은 adapter/gateway seam으로 충분하면 그것을 우선 사용하세요 |
    | `plugin-sdk/approval-native-runtime` | 네이티브 승인 대상 + account-binding helper |
    | `plugin-sdk/approval-reply-runtime` | Exec/plugin 승인 reply payload helper |
    | `plugin-sdk/command-auth-native` | 네이티브 command auth + 네이티브 session-target helper |
    | `plugin-sdk/command-detection` | 공용 command 감지 helper |
    | `plugin-sdk/command-surface` | Command-body 정규화 및 command-surface helper |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/channel-secret-runtime` | channel/plugin secret 표면용 좁은 범위의 secret-contract 수집 helper |
    | `plugin-sdk/secret-ref-runtime` | secret-contract/config 파싱용 좁은 범위의 `coerceSecretRef` 및 SecretRef 타입 helper |
    | `plugin-sdk/security-runtime` | 공용 신뢰, DM 게이팅, 외부 콘텐츠, secret 수집 helper |
    | `plugin-sdk/ssrf-policy` | 호스트 허용 목록 및 private-network SSRF 정책 helper |
    | `plugin-sdk/ssrf-runtime` | pinned-dispatcher, SSRF 보호 fetch, SSRF 정책 helper |
    | `plugin-sdk/secret-input` | Secret 입력 파싱 helper |
    | `plugin-sdk/webhook-ingress` | Webhook 요청/대상 helper |
    | `plugin-sdk/webhook-request-guards` | 요청 본문 크기/시간 초과 helper |
  </Accordion>

  <Accordion title="런타임 및 저장소 하위 경로">
    | Subpath | 주요 exports |
    | --- | --- |
    | `plugin-sdk/runtime` | 폭넓은 런타임/로깅/백업/plugin 설치 helper |
    | `plugin-sdk/runtime-env` | 좁은 범위의 런타임 env, logger, timeout, retry, backoff helper |
    | `plugin-sdk/channel-runtime-context` | 일반적인 channel runtime-context 등록 및 조회 helper |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | 공용 plugin command/hook/http/interactive helper |
    | `plugin-sdk/hook-runtime` | 공용 webhook/internal hook 파이프라인 helper |
    | `plugin-sdk/lazy-runtime` | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeSurface` 같은 지연 런타임 import/binding helper |
    | `plugin-sdk/process-runtime` | 프로세스 exec helper |
    | `plugin-sdk/cli-runtime` | CLI 서식 지정, 대기, 버전 helper |
    | `plugin-sdk/gateway-runtime` | Gateway client 및 channel-status patch helper |
    | `plugin-sdk/config-runtime` | Config 로드/쓰기 helper |
    | `plugin-sdk/telegram-command-config` | 번들 Telegram 계약 표면을 사용할 수 없을 때도 Telegram command-name/description 정규화 및 중복/충돌 검사 |
    | `plugin-sdk/approval-runtime` | Exec/plugin 승인 helper, 승인-capability 빌더, auth/profile helper, 네이티브 라우팅/런타임 helper |
    | `plugin-sdk/reply-runtime` | 공용 수신/reply 런타임 helper, chunking, dispatch, heartbeat, reply planner |
    | `plugin-sdk/reply-dispatch-runtime` | 좁은 범위의 reply dispatch/finalize helper |
    | `plugin-sdk/reply-history` | `buildHistoryContext`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` 같은 공용 짧은 기간 reply-history helper |
    | `plugin-sdk/reply-reference` | `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | 좁은 범위의 텍스트/markdown chunking helper |
    | `plugin-sdk/session-store-runtime` | 세션 저장소 경로 + updated-at helper |
    | `plugin-sdk/state-paths` | 상태/OAuth 디렉터리 경로 helper |
    | `plugin-sdk/routing` | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId` 같은 route/session-key/account 바인딩 helper |
    | `plugin-sdk/status-helpers` | 공용 channel/account 상태 요약 helper, 런타임 상태 기본값, issue 메타데이터 helper |
    | `plugin-sdk/target-resolver-runtime` | 공용 대상 해석 helper |
    | `plugin-sdk/string-normalization-runtime` | slug/string 정규화 helper |
    | `plugin-sdk/request-url` | fetch/request 유사 입력에서 문자열 URL 추출 |
    | `plugin-sdk/run-command` | 정규화된 stdout/stderr 결과를 반환하는 시간 제한 command 실행기 |
    | `plugin-sdk/param-readers` | 공용 tool/CLI 매개변수 리더 |
    | `plugin-sdk/tool-payload` | tool 결과 객체에서 정규화된 payload 추출 |
    | `plugin-sdk/tool-send` | tool args에서 정식 send 대상 필드 추출 |
    | `plugin-sdk/temp-path` | 공용 임시 다운로드 경로 helper |
    | `plugin-sdk/logging-core` | 서브시스템 logger 및 redaction helper |
    | `plugin-sdk/markdown-table-runtime` | Markdown 표 모드 helper |
    | `plugin-sdk/json-store` | 작은 JSON 상태 읽기/쓰기 helper |
    | `plugin-sdk/file-lock` | 재진입 가능한 파일 잠금 helper |
    | `plugin-sdk/persistent-dedupe` | 디스크 기반 dedupe cache helper |
    | `plugin-sdk/acp-runtime` | ACP 런타임/세션 및 reply-dispatch helper |
    | `plugin-sdk/agent-config-primitives` | 좁은 범위의 agent 런타임 config-schema primitives |
    | `plugin-sdk/boolean-param` | 느슨한 boolean 매개변수 리더 |
    | `plugin-sdk/dangerous-name-runtime` | 위험한 이름 매칭 해석 helper |
    | `plugin-sdk/device-bootstrap` | 디바이스 bootstrap 및 페어링 토큰 helper |
    | `plugin-sdk/extension-shared` | 공용 passive-channel, 상태, ambient proxy helper primitives |
    | `plugin-sdk/models-provider-runtime` | `/models` command/provider reply helper |
    | `plugin-sdk/skill-commands-runtime` | Skills command 목록 helper |
    | `plugin-sdk/native-command-registry` | 네이티브 command 레지스트리/build/serialize helper |
    | `plugin-sdk/provider-zai-endpoint` | Z.AI endpoint 감지 helper |
    | `plugin-sdk/infra-runtime` | 시스템 이벤트/heartbeat helper |
    | `plugin-sdk/collection-runtime` | 작은 bounded cache helper |
    | `plugin-sdk/diagnostic-runtime` | 진단 플래그 및 이벤트 helper |
    | `plugin-sdk/error-runtime` | 오류 그래프, 서식 지정, 공용 오류 분류 helper, `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | 래핑된 fetch, proxy, pinned lookup helper |
    | `plugin-sdk/host-runtime` | 호스트명 및 SCP 호스트 정규화 helper |
    | `plugin-sdk/retry-runtime` | Retry config 및 retry 실행 helper |
    | `plugin-sdk/agent-runtime` | agent 디렉터리/식별자/워크스페이스 helper |
    | `plugin-sdk/directory-runtime` | config 기반 디렉터리 질의/dedup |
    | `plugin-sdk/keyed-async-queue` | `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="Capability 및 테스트 하위 경로">
    | Subpath | 주요 exports |
    | --- | --- |
    | `plugin-sdk/media-runtime` | 공용 미디어 fetch/transform/store helper 및 미디어 payload 빌더 |
    | `plugin-sdk/media-generation-runtime` | 공용 media-generation failover helper, 후보 선택, 모델 누락 메시지 처리 |
    | `plugin-sdk/media-understanding` | Media understanding provider 타입 및 provider용 이미지/오디오 helper export |
    | `plugin-sdk/text-runtime` | assistant-visible-text 제거, markdown render/chunking/table helper, redaction helper, directive-tag helper, safe-text 유틸리티 같은 공용 텍스트/markdown/로깅 helper |
    | `plugin-sdk/text-chunking` | 송신 텍스트 chunking helper |
    | `plugin-sdk/speech` | Speech provider 타입 및 provider용 directive, 레지스트리, 검증 helper |
    | `plugin-sdk/speech-core` | 공용 speech provider 타입, 레지스트리, directive, 정규화 helper |
    | `plugin-sdk/realtime-transcription` | Realtime transcription provider 타입 및 레지스트리 helper |
    | `plugin-sdk/realtime-voice` | Realtime voice provider 타입 및 레지스트리 helper |
    | `plugin-sdk/image-generation` | Image generation provider 타입 |
    | `plugin-sdk/image-generation-core` | 공용 image-generation 타입, failover, auth, 레지스트리 helper |
    | `plugin-sdk/music-generation` | Music generation provider/request/result 타입 |
    | `plugin-sdk/music-generation-core` | 공용 music-generation 타입, failover helper, provider 조회, model-ref 파싱 |
    | `plugin-sdk/video-generation` | Video generation provider/request/result 타입 |
    | `plugin-sdk/video-generation-core` | 공용 video-generation 타입, failover helper, provider 조회, model-ref 파싱 |
    | `plugin-sdk/webhook-targets` | Webhook 대상 레지스트리 및 route-install helper |
    | `plugin-sdk/webhook-path` | Webhook 경로 정규화 helper |
    | `plugin-sdk/web-media` | 공용 원격/로컬 미디어 로딩 helper |
    | `plugin-sdk/zod` | Plugin SDK 소비자를 위한 재export된 `zod` |
    | `plugin-sdk/testing` | `installCommonResolveTargetErrorCases`, `shouldAckReaction` |
  </Accordion>

  <Accordion title="Memory 하위 경로">
    | Subpath | 주요 exports |
    | --- | --- |
    | `plugin-sdk/memory-core` | manager/config/file/CLI helper를 위한 번들 memory-core helper 표면 |
    | `plugin-sdk/memory-core-engine-runtime` | Memory index/search 런타임 파사드 |
    | `plugin-sdk/memory-core-host-engine-foundation` | Memory host foundation engine exports |
    | `plugin-sdk/memory-core-host-engine-embeddings` | Memory host embedding engine exports |
    | `plugin-sdk/memory-core-host-engine-qmd` | Memory host QMD engine exports |
    | `plugin-sdk/memory-core-host-engine-storage` | Memory host storage engine exports |
    | `plugin-sdk/memory-core-host-multimodal` | Memory host multimodal helper |
    | `plugin-sdk/memory-core-host-query` | Memory host query helper |
    | `plugin-sdk/memory-core-host-secret` | Memory host secret helper |
    | `plugin-sdk/memory-core-host-events` | Memory host 이벤트 저널 helper |
    | `plugin-sdk/memory-core-host-status` | Memory host 상태 helper |
    | `plugin-sdk/memory-core-host-runtime-cli` | Memory host CLI 런타임 helper |
    | `plugin-sdk/memory-core-host-runtime-core` | Memory host core 런타임 helper |
    | `plugin-sdk/memory-core-host-runtime-files` | Memory host 파일/런타임 helper |
    | `plugin-sdk/memory-host-core` | 메모리 host core 런타임 helper를 위한 vendor-neutral 별칭 |
    | `plugin-sdk/memory-host-events` | 메모리 host 이벤트 저널 helper를 위한 vendor-neutral 별칭 |
    | `plugin-sdk/memory-host-files` | 메모리 host 파일/런타임 helper를 위한 vendor-neutral 별칭 |
    | `plugin-sdk/memory-host-markdown` | 메모리 인접 plugins를 위한 공용 managed-markdown helper |
    | `plugin-sdk/memory-host-search` | search-manager 접근을 위한 활성 메모리 런타임 파사드 |
    | `plugin-sdk/memory-host-status` | 메모리 host 상태 helper를 위한 vendor-neutral 별칭 |
    | `plugin-sdk/memory-lancedb` | 번들 memory-lancedb helper 표면 |
  </Accordion>

  <Accordion title="예약된 bundled-helper 하위 경로">
    | Family | 현재 하위 경로 | 의도된 용도 |
    | --- | --- | --- |
    | Browser | `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support` | 번들 browser plugin 지원 helper(`browser-support`는 호환성 barrel로 유지됨) |
    | Matrix | `plugin-sdk/matrix`, `plugin-sdk/matrix-helper`, `plugin-sdk/matrix-runtime-heavy`, `plugin-sdk/matrix-runtime-shared`, `plugin-sdk/matrix-runtime-surface`, `plugin-sdk/matrix-surface`, `plugin-sdk/matrix-thread-bindings` | 번들 Matrix helper/런타임 표면 |
    | Line | `plugin-sdk/line`, `plugin-sdk/line-core`, `plugin-sdk/line-runtime`, `plugin-sdk/line-surface` | 번들 LINE helper/런타임 표면 |
    | IRC | `plugin-sdk/irc`, `plugin-sdk/irc-surface` | 번들 IRC helper 표면 |
    | Channel 전용 helper | `plugin-sdk/googlechat`, `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles`, `plugin-sdk/bluebubbles-policy`, `plugin-sdk/mattermost`, `plugin-sdk/mattermost-policy`, `plugin-sdk/feishu-conversation`, `plugin-sdk/msteams`, `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`, `plugin-sdk/twitch` | 번들 channel 호환성/helper seam |
    | Auth/plugin 전용 helper | `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`, `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`, `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` | 번들 기능/plugin helper seam; `plugin-sdk/github-copilot-token`은 현재 `DEFAULT_COPILOT_API_BASE_URL`, `deriveCopilotApiBaseUrlFromToken`, `resolveCopilotApiToken`을 export합니다 |
  </Accordion>
</AccordionGroup>

## 등록 API

`register(api)` 콜백은 다음 메서드를 가진 `OpenClawPluginApi` 객체를 받습니다.

### Capability 등록

| Method                                           | 등록되는 항목                    |
| ------------------------------------------------ | -------------------------------- |
| `api.registerProvider(...)`                      | 텍스트 추론(LLM)                 |
| `api.registerCliBackend(...)`                    | 로컬 CLI 추론 backend            |
| `api.registerChannel(...)`                       | 메시징 channel                   |
| `api.registerSpeechProvider(...)`                | 텍스트 음성 변환 / STT 합성      |
| `api.registerRealtimeTranscriptionProvider(...)` | 스트리밍 realtime transcription  |
| `api.registerRealtimeVoiceProvider(...)`         | 양방향 realtime 음성 세션        |
| `api.registerMediaUnderstandingProvider(...)`    | 이미지/오디오/비디오 분석        |
| `api.registerImageGenerationProvider(...)`       | 이미지 생성                      |
| `api.registerMusicGenerationProvider(...)`       | 음악 생성                        |
| `api.registerVideoGenerationProvider(...)`       | 비디오 생성                      |
| `api.registerWebFetchProvider(...)`              | 웹 fetch / scrape provider       |
| `api.registerWebSearchProvider(...)`             | 웹 검색                          |

### 도구 및 commands

| Method                          | 등록되는 항목                                  |
| ------------------------------- | ---------------------------------------------- |
| `api.registerTool(tool, opts?)` | agent 도구(필수 또는 `{ optional: true }`)     |
| `api.registerCommand(def)`      | 사용자 지정 command(LLM 우회)                  |

### 인프라

| Method                                         | 등록되는 항목                            |
| ---------------------------------------------- | ---------------------------------------- |
| `api.registerHook(events, handler, opts?)`     | 이벤트 hook                              |
| `api.registerHttpRoute(params)`                | Gateway HTTP endpoint                    |
| `api.registerGatewayMethod(name, handler)`     | Gateway RPC 메서드                       |
| `api.registerCli(registrar, opts?)`            | CLI 하위 명령                            |
| `api.registerService(service)`                 | 백그라운드 서비스                        |
| `api.registerInteractiveHandler(registration)` | 대화형 handler                           |
| `api.registerMemoryPromptSupplement(builder)`  | 추가형 memory 인접 프롬프트 섹션         |
| `api.registerMemoryCorpusSupplement(adapter)`  | 추가형 memory 검색/읽기 corpus           |

예약된 core 관리자 네임스페이스(`config.*`, `exec.approvals.*`, `wizard.*`, `update.*`)는 plugin이 더 좁은 gateway 메서드 범위를 할당하려고 해도 항상 `operator.admin`으로 유지됩니다. plugin 소유 메서드에는 plugin 전용 접두사를 사용하는 것이 좋습니다.

### CLI 등록 메타데이터

`api.registerCli(registrar, opts?)`는 두 가지 종류의 최상위 메타데이터를 받습니다.

- `commands`: registrar가 소유한 명시적 command 루트
- `descriptors`: 루트 CLI help, 라우팅, 지연 plugin CLI 등록에 사용되는 파싱 시점 command descriptor

plugin command를 일반 루트 CLI 경로에서 지연 로드 상태로 유지하려면, 해당 registrar가 노출하는 모든 최상위 command 루트를 포괄하는 `descriptors`를 제공하세요.

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
        description: "Matrix 계정, 검증, 디바이스 및 프로필 상태 관리",
        hasSubcommands: true,
      },
    ],
  },
);
```

지연 루트 CLI 등록이 필요하지 않을 때만 `commands`를 단독으로 사용하세요. 이 eager 호환성 경로는 여전히 지원되지만, 파싱 시점 지연 로드를 위한 descriptor 기반 placeholder는 설치하지 않습니다.

### CLI backend 등록

`api.registerCliBackend(...)`를 사용하면 plugin이 `codex-cli` 같은 로컬 AI CLI backend의 기본 config를 소유할 수 있습니다.

- backend `id`는 `codex-cli/gpt-5` 같은 model ref의 provider 접두사가 됩니다.
- backend `config`는 `agents.defaults.cliBackends.<id>`와 동일한 형태를 사용합니다.
- 사용자 config가 여전히 우선합니다. OpenClaw는 CLI를 실행하기 전에 plugin 기본값 위에 `agents.defaults.cliBackends.<id>`를 병합합니다.
- 병합 후 backend에 호환성 재작성(예: 이전 플래그 형태 정규화)이 필요하다면 `normalizeConfig`를 사용하세요.

### 배타적 슬롯

| Method                                     | 등록되는 항목                                                                                                                                          |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `api.registerContextEngine(id, factory)`   | 컨텍스트 엔진(한 번에 하나만 활성). `assemble()` 콜백은 `availableTools`와 `citationsMode`를 받아 엔진이 프롬프트 추가 내용을 맞춤 조정할 수 있게 합니다. |
| `api.registerMemoryCapability(capability)` | 통합 memory capability                                                                                                                                 |
| `api.registerMemoryPromptSection(builder)` | Memory 프롬프트 섹션 빌더                                                                                                                              |
| `api.registerMemoryFlushPlan(resolver)`    | Memory flush plan resolver                                                                                                                             |
| `api.registerMemoryRuntime(runtime)`       | Memory 런타임 adapter                                                                                                                                  |

### Memory embedding adapters

| Method                                         | 등록되는 항목                                      |
| ---------------------------------------------- | -------------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | 활성 plugin용 memory embedding adapter             |

- `registerMemoryCapability`가 권장되는 배타적 memory-plugin API입니다.
- `registerMemoryCapability`는 companion plugins가 특정 memory plugin의 private 레이아웃에 직접 접근하지 않고 `openclaw/plugin-sdk/memory-host-core`를 통해 export된 memory artifact를 사용할 수 있도록 `publicArtifacts.listArtifacts(...)`도 노출할 수 있습니다.
- `registerMemoryPromptSection`, `registerMemoryFlushPlan`, `registerMemoryRuntime`는 레거시 호환용 배타적 memory-plugin API입니다.
- `registerMemoryEmbeddingProvider`를 사용하면 활성 memory plugin이 하나 이상의 embedding adapter id(예: `openai`, `gemini`, 또는 plugin이 정의한 사용자 지정 id)를 등록할 수 있습니다.
- `agents.defaults.memorySearch.provider` 및 `agents.defaults.memorySearch.fallback` 같은 사용자 config는 이러한 등록된 adapter id를 기준으로 해석됩니다.

### 이벤트 및 lifecycle

| Method                                       | 동작                         |
| -------------------------------------------- | ---------------------------- |
| `api.on(hookName, handler, opts?)`           | 타입 지정된 lifecycle hook   |
| `api.onConversationBindingResolved(handler)` | 대화 바인딩 콜백             |

### Hook 결정 의미

- `before_tool_call`: `{ block: true }`를 반환하면 종료됩니다. 어떤 handler든 이를 설정하면 더 낮은 우선순위 handler는 건너뜁니다.
- `before_tool_call`: `{ block: false }`를 반환하면 결정 없음(`block` 생략과 동일)으로 처리되며, override가 아닙니다.
- `before_install`: `{ block: true }`를 반환하면 종료됩니다. 어떤 handler든 이를 설정하면 더 낮은 우선순위 handler는 건너뜁니다.
- `before_install`: `{ block: false }`를 반환하면 결정 없음(`block` 생략과 동일)으로 처리되며, override가 아닙니다.
- `reply_dispatch`: `{ handled: true, ... }`를 반환하면 종료됩니다. 어떤 handler든 dispatch를 처리했다고 선언하면 더 낮은 우선순위 handler와 기본 모델 dispatch 경로를 건너뜁니다.
- `message_sending`: `{ cancel: true }`를 반환하면 종료됩니다. 어떤 handler든 이를 설정하면 더 낮은 우선순위 handler는 건너뜁니다.
- `message_sending`: `{ cancel: false }`를 반환하면 결정 없음(`cancel` 생략과 동일)으로 처리되며, override가 아닙니다.

### API 객체 필드

| Field                    | Type                      | 설명                                                                                         |
| ------------------------ | ------------------------- | -------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Plugin id                                                                                    |
| `api.name`               | `string`                  | 표시 이름                                                                                    |
| `api.version`            | `string?`                 | Plugin 버전(선택 사항)                                                                       |
| `api.description`        | `string?`                 | Plugin 설명(선택 사항)                                                                       |
| `api.source`             | `string`                  | Plugin 소스 경로                                                                             |
| `api.rootDir`            | `string?`                 | Plugin 루트 디렉터리(선택 사항)                                                              |
| `api.config`             | `OpenClawConfig`          | 현재 config 스냅샷(사용 가능할 경우 활성 in-memory 런타임 스냅샷)                           |
| `api.pluginConfig`       | `Record<string, unknown>` | `plugins.entries.<id>.config`의 plugin 전용 config                                           |
| `api.runtime`            | `PluginRuntime`           | [Runtime helpers](/ko/plugins/sdk-runtime)                                                      |
| `api.logger`             | `PluginLogger`            | 범위가 지정된 logger(`debug`, `info`, `warn`, `error`)                                      |
| `api.registrationMode`   | `PluginRegistrationMode`  | 현재 로드 모드. `"setup-runtime"`은 가벼운 전체 entry 이전 시작/setup 구간입니다            |
| `api.resolvePath(input)` | `(string) => string`      | plugin 루트를 기준으로 경로 해석                                                             |

## 내부 모듈 규칙

plugin 내부에서는 내부 import에 로컬 barrel 파일을 사용하세요.

```
my-plugin/
  api.ts            # 외부 소비자를 위한 public exports
  runtime-api.ts    # 내부 전용 런타임 exports
  index.ts          # Plugin entry point
  setup-entry.ts    # 가벼운 setup 전용 entry(선택 사항)
```

<Warning>
  프로덕션 코드에서 `openclaw/plugin-sdk/<your-plugin>`을 통해 자신의 plugin을 절대 import하지 마세요. 내부 import는 `./api.ts` 또는 `./runtime-api.ts`를 통해 처리하세요. SDK 경로는 외부 계약 전용입니다.
</Warning>

파사드로 로드되는 번들 plugin public 표면(`api.ts`, `runtime-api.ts`, `index.ts`, `setup-entry.ts` 및 유사한 public entry 파일)은 이제 OpenClaw가 이미 실행 중일 때 활성 런타임 config 스냅샷을 우선 사용합니다. 아직 런타임 스냅샷이 없으면 디스크의 해석된 config 파일로 폴백합니다.

Provider plugins는 helper가 의도적으로 provider 전용이고 아직 일반적인 SDK 하위 경로에 속하지 않을 경우 좁은 범위의 plugin 로컬 계약 barrel도 노출할 수 있습니다. 현재 번들 예시: Anthropic provider는 Anthropic 베타 헤더와 `service_tier` 로직을 일반적인 `plugin-sdk/*` 계약으로 올리는 대신 Claude stream helper를 자체 public `api.ts` / `contract-api.ts` seam에 유지합니다.

그 밖의 현재 번들 예시:

- `@openclaw/openai-provider`: `api.ts`는 provider 빌더, 기본 model helper, realtime provider 빌더를 export합니다
- `@openclaw/openrouter-provider`: `api.ts`는 provider 빌더와 onboarding/config helper를 export합니다

<Warning>
  확장 프로덕션 코드도 `openclaw/plugin-sdk/<other-plugin>` import를 피해야 합니다. helper가 정말 공용이어야 한다면 두 plugins를 결합하는 대신 `openclaw/plugin-sdk/speech`, `.../provider-model-shared` 또는 다른 capability 지향 표면 같은 중립적인 SDK 하위 경로로 승격하세요.
</Warning>

## 관련 문서

- [Entry Points](/ko/plugins/sdk-entrypoints) — `definePluginEntry` 및 `defineChannelPluginEntry` 옵션
- [Runtime Helpers](/ko/plugins/sdk-runtime) — 전체 `api.runtime` 네임스페이스 참조
- [Setup and Config](/ko/plugins/sdk-setup) — 패키징, 매니페스트, config 스키마
- [Testing](/ko/plugins/sdk-testing) — 테스트 유틸리티 및 lint 규칙
- [SDK Migration](/ko/plugins/sdk-migration) — 더 이상 권장되지 않는 표면에서 마이그레이션
- [Plugin Internals](/ko/plugins/architecture) — 심화 아키텍처 및 capability 모델
