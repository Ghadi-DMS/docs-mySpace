---
read_when:
    - OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED 경고가 표시되는 경우
    - OPENCLAW_EXTENSION_API_DEPRECATED 경고가 표시되는 경우
    - plugin을 최신 plugin 아키텍처로 업데이트하는 경우
    - 외부 OpenClaw plugin을 유지 관리하는 경우
sidebarTitle: Migrate to SDK
summary: 기존 하위 호환성 계층에서 최신 plugin SDK로 마이그레이션
title: Plugin SDK 마이그레이션
x-i18n:
    generated_at: "2026-04-09T01:30:38Z"
    model: gpt-5.4
    provider: openai
    source_hash: 60cbb6c8be30d17770887d490c14e3a4538563339a5206fb419e51e0558bbc07
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Plugin SDK 마이그레이션

OpenClaw는 광범위한 하위 호환성 계층에서 집중적이고 문서화된 import를 사용하는 최신 plugin
아키텍처로 전환했습니다. plugin이 새 아키텍처 이전에 만들어졌다면,
이 가이드는 마이그레이션에 도움이 됩니다.

## 변경되는 내용

기존 plugin 시스템은 plugin이 단일 진입점에서 필요한 거의 모든 것을 import할 수 있도록 하는
두 개의 광범위한 표면을 제공했습니다.

- **`openclaw/plugin-sdk/compat`** — 수십 개의
  helper를 다시 내보내는 단일 import입니다. 새 plugin 아키텍처가 구축되는 동안
  기존 hook 기반 plugins가 계속 동작하도록 도입되었습니다.
- **`openclaw/extension-api`** — plugin이 내장 agent runner 같은
  호스트 측 helper에 직접 접근할 수 있게 해주는 브리지입니다.

이제 두 표면 모두 **사용 중단됨** 상태입니다. 런타임에서는 여전히 동작하지만, 새
plugins는 이를 사용해서는 안 되며, 기존 plugins도 다음 메이저 릴리스에서 제거되기 전에
마이그레이션해야 합니다.

<Warning>
  하위 호환성 계층은 향후 메이저 릴리스에서 제거될 예정입니다.
  여전히 이 표면에서 import하는 plugins는 그 시점에 동작하지 않게 됩니다.
</Warning>

## 왜 변경되었나요

기존 접근 방식은 다음과 같은 문제를 일으켰습니다.

- **느린 시작 속도** — helper 하나를 import해도 관련 없는 수십 개 모듈이 로드됨
- **순환 의존성** — 광범위한 재내보내기로 인해 import cycle이 쉽게 생김
- **불명확한 API 표면** — 어떤 export가 안정적이고 어떤 것이 내부용인지 구분할 수 없음

최신 plugin SDK는 이를 해결합니다. 각 import 경로(`openclaw/plugin-sdk/\<subpath\>`)는
작고 독립적인 모듈이며, 명확한 목적과 문서화된 계약을 가집니다.

번들 채널용 기존 provider 편의 seam도 제거되었습니다. `openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`, `openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
채널 브랜딩 helper seam, 그리고
`openclaw/plugin-sdk/telegram-core` 같은 import는 안정적인 plugin 계약이 아닌
비공개 mono-repo 지름길이었습니다. 대신 더 좁고 일반적인 SDK 하위 경로를 사용하세요. 번들 plugin workspace 내부에서는 provider 소유 helper를 해당 plugin의
자체 `api.ts` 또는 `runtime-api.ts`에 두세요.

현재 번들 provider 예시:

- Anthropic은 Claude 전용 stream helper를 자체 `api.ts` /
  `contract-api.ts` seam에 유지합니다
- OpenAI는 provider builder, 기본 모델 helper, realtime provider
  builder를 자체 `api.ts`에 유지합니다
- OpenRouter는 provider builder와 onboarding/config helper를 자체
  `api.ts`에 유지합니다

## 마이그레이션 방법

<Steps>
  <Step title="승인 네이티브 핸들러를 capability fact로 마이그레이션">
    승인 가능한 채널 plugins는 이제
    `approvalCapability.nativeRuntime`와 공유 runtime-context registry를 통해
    네이티브 승인 동작을 노출합니다.

    주요 변경 사항:

    - `approvalCapability.handler.loadRuntime(...)`를
      `approvalCapability.nativeRuntime`로 교체
    - 승인 관련 auth/delivery를 기존 `plugin.auth` /
      `plugin.approvals` 연결에서 `approvalCapability`로 이동
    - `ChannelPlugin.approvals`는 공개 채널 plugin
      계약에서 제거되었으므로, delivery/native/render 필드를 `approvalCapability`로 이동
    - `plugin.auth`는 채널 login/logout 흐름에만 남아 있으며, 그 안의 승인 auth
      hook은 더 이상 core에서 읽지 않음
    - 클라이언트, 토큰 또는 Bolt
      앱 같은 채널 소유 runtime 객체는 `openclaw/plugin-sdk/channel-runtime-context`를 통해 등록
    - 네이티브 승인 핸들러에서 plugin 소유의 재라우팅 알림을 보내지 마세요.
      core가 이제 실제 delivery 결과에서 발생한 다른 경로 알림을 소유합니다
    - `channelRuntime`을 `createChannelManager(...)`에 전달할 때는,
      실제 `createPluginRuntime().channel` 표면을 제공하세요. 부분 stub은 거부됩니다.

    현재 승인 capability
    레이아웃은 `/plugins/sdk-channel-plugins`를 참조하세요.

  </Step>

  <Step title="Windows wrapper 대체 동작 감사">
    plugin이 `openclaw/plugin-sdk/windows-spawn`을 사용하는 경우,
    해결되지 않는 Windows `.cmd`/`.bat` wrapper는 이제 명시적으로
    `allowShellFallback: true`를 전달하지 않으면 닫힌 상태로 실패합니다.

    ```typescript
    // 이전
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // 이후
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // 신뢰된 호환성 호출자만 이 값을 설정하세요. 이러한 호출자는 의도적으로
      // 셸을 통한 대체를 허용합니다.
      allowShellFallback: true,
    });
    ```

    호출자가 의도적으로 셸 대체에 의존하지 않는다면, `allowShellFallback`을 설정하지 말고
    대신 발생한 오류를 처리하세요.

  </Step>

  <Step title="사용 중단된 import 찾기">
    plugin에서 사용 중단된 두 표면 중 하나에서의 import를 검색하세요.

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="집중된 import로 교체">
    기존 표면의 각 export는 특정 최신 import 경로에 매핑됩니다.

    ```typescript
    // 이전 (사용 중단된 하위 호환성 계층)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // 이후 (최신 집중형 import)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    호스트 측 helper의 경우 직접 import하지 말고,
    주입된 plugin runtime을 사용하세요.

    ```typescript
    // 이전 (사용 중단된 extension-api 브리지)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // 이후 (주입된 runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    동일한 패턴이 다른 기존 브리지 helper에도 적용됩니다.

    | 기존 import | 최신 대응 항목 |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | session 저장소 helper | `api.runtime.agent.session.*` |

  </Step>

  <Step title="빌드 및 테스트">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## import 경로 참조

<Accordion title="일반적인 import 경로 표">
  | import 경로 | 목적 | 주요 export |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | 정식 plugin 진입 helper | `definePluginEntry` |
  | `plugin-sdk/core` | 채널 진입 정의/빌더를 위한 기존 umbrella 재내보내기 | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | 루트 config 스키마 export | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | 단일 provider 진입 helper | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | 집중된 채널 진입 정의 및 빌더 | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | 공유 setup wizard helper | 허용 목록 프롬프트, setup 상태 빌더 |
  | `plugin-sdk/setup-runtime` | setup 시점 runtime helper | import-safe setup patch adapter, lookup-note helper, `promptResolvedAllowFrom`, `splitSetupEntries`, 위임 setup proxy |
  | `plugin-sdk/setup-adapter-runtime` | setup adapter helper | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | setup 도구 helper | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | 다중 계정 helper | 계정 목록/config/action-gate helper |
  | `plugin-sdk/account-id` | account-id helper | `DEFAULT_ACCOUNT_ID`, account-id 정규화 |
  | `plugin-sdk/account-resolution` | 계정 조회 helper | 계정 조회 + 기본값 대체 helper |
  | `plugin-sdk/account-helpers` | 좁은 계정 helper | 계정 목록/계정 작업 helper |
  | `plugin-sdk/channel-setup` | setup wizard adapter | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, 그리고 `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | DM 페어링 기본 구성 요소 | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | 답장 접두사 + 타이핑 연결 | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | config adapter 팩토리 | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | config 스키마 빌더 | 채널 config 스키마 타입 |
  | `plugin-sdk/telegram-command-config` | Telegram 명령 config helper | 명령 이름 정규화, 설명 잘라내기, 중복/충돌 검증 |
  | `plugin-sdk/channel-policy` | 그룹/DM 정책 해석 | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | 계정 상태 추적 | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | 인바운드 envelope helper | 공유 route + envelope 빌더 helper |
  | `plugin-sdk/inbound-reply-dispatch` | 인바운드 답장 helper | 공유 record-and-dispatch helper |
  | `plugin-sdk/messaging-targets` | 메시징 대상 구문 분석 | 대상 구문 분석/매칭 helper |
  | `plugin-sdk/outbound-media` | 아웃바운드 미디어 helper | 공유 아웃바운드 미디어 로딩 |
  | `plugin-sdk/outbound-runtime` | 아웃바운드 runtime helper | 아웃바운드 identity/send delegate helper |
  | `plugin-sdk/thread-bindings-runtime` | 스레드 바인딩 helper | 스레드 바인딩 수명 주기 및 adapter helper |
  | `plugin-sdk/agent-media-payload` | 기존 미디어 payload helper | 기존 필드 레이아웃용 agent 미디어 payload 빌더 |
  | `plugin-sdk/channel-runtime` | 사용 중단된 호환성 shim | 기존 채널 runtime 유틸리티 전용 |
  | `plugin-sdk/channel-send-result` | 전송 결과 타입 | 답장 결과 타입 |
  | `plugin-sdk/runtime-store` | 영구 plugin 저장소 | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | 광범위한 runtime helper | runtime/logging/backup/plugin-install helper |
  | `plugin-sdk/runtime-env` | 좁은 runtime env helper | logger/runtime env, timeout, retry, backoff helper |
  | `plugin-sdk/plugin-runtime` | 공유 plugin runtime helper | plugin commands/hooks/http/interactive helper |
  | `plugin-sdk/hook-runtime` | hook 파이프라인 helper | 공유 webhook/internal hook 파이프라인 helper |
  | `plugin-sdk/lazy-runtime` | 지연 runtime helper | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | 프로세스 helper | 공유 exec helper |
  | `plugin-sdk/cli-runtime` | CLI runtime helper | 명령 포맷팅, 대기, 버전 helper |
  | `plugin-sdk/gateway-runtime` | Gateway helper | Gateway client 및 channel-status patch helper |
  | `plugin-sdk/config-runtime` | Config helper | config 로드/기록 helper |
  | `plugin-sdk/telegram-command-config` | Telegram 명령 helper | 번들 Telegram 계약 표면을 사용할 수 없을 때를 위한 안정적인 대체 Telegram 명령 검증 helper |
  | `plugin-sdk/approval-runtime` | 승인 프롬프트 helper | exec/plugin 승인 payload, 승인 capability/profile helper, 네이티브 승인 라우팅/runtime helper |
  | `plugin-sdk/approval-auth-runtime` | 승인 auth helper | approver 해석, 동일 채팅 action auth |
  | `plugin-sdk/approval-client-runtime` | 승인 client helper | 네이티브 exec 승인 profile/filter helper |
  | `plugin-sdk/approval-delivery-runtime` | 승인 delivery helper | 네이티브 승인 capability/delivery adapter |
  | `plugin-sdk/approval-gateway-runtime` | 승인 gateway helper | 공유 승인 gateway-resolution helper |
  | `plugin-sdk/approval-handler-adapter-runtime` | 승인 adapter helper | 핫 채널 진입점용 경량 네이티브 승인 adapter 로딩 helper |
  | `plugin-sdk/approval-handler-runtime` | 승인 핸들러 helper | 더 광범위한 승인 핸들러 runtime helper. 더 좁은 adapter/gateway seam으로 충분하다면 그것을 우선 사용하세요 |
  | `plugin-sdk/approval-native-runtime` | 승인 대상 helper | 네이티브 승인 대상/account binding helper |
  | `plugin-sdk/approval-reply-runtime` | 승인 답장 helper | exec/plugin 승인 답장 payload helper |
  | `plugin-sdk/channel-runtime-context` | 채널 runtime-context helper | 일반 채널 runtime-context register/get/watch helper |
  | `plugin-sdk/security-runtime` | 보안 helper | 공유 trust, DM gating, external-content, secret-collection helper |
  | `plugin-sdk/ssrf-policy` | SSRF 정책 helper | 호스트 허용 목록 및 private-network 정책 helper |
  | `plugin-sdk/ssrf-runtime` | SSRF runtime helper | pinned-dispatcher, guarded fetch, SSRF 정책 helper |
  | `plugin-sdk/collection-runtime` | 경계가 있는 캐시 helper | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | 진단 게이팅 helper | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | 오류 포맷팅 helper | `formatUncaughtError`, `isApprovalNotFoundError`, 오류 그래프 helper |
  | `plugin-sdk/fetch-runtime` | 래핑된 fetch/proxy helper | `resolveFetch`, proxy helper |
  | `plugin-sdk/host-runtime` | 호스트 정규화 helper | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | 재시도 helper | `RetryConfig`, `retryAsync`, 정책 실행기 |
  | `plugin-sdk/allow-from` | 허용 목록 포맷팅 | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | 허용 목록 입력 매핑 | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | 명령 게이팅 및 명령 표면 helper | `resolveControlCommandGate`, sender authorization helper, 명령 registry helper |
  | `plugin-sdk/command-status` | 명령 상태/help 렌더러 | `buildCommandsMessage`, `buildCommandsMessagePaginated`, `buildHelpMessage` |
  | `plugin-sdk/secret-input` | 비밀 입력 구문 분석 | 비밀 입력 helper |
  | `plugin-sdk/webhook-ingress` | webhook 요청 helper | webhook 대상 유틸리티 |
  | `plugin-sdk/webhook-request-guards` | webhook 본문 가드 helper | 요청 본문 읽기/제한 helper |
  | `plugin-sdk/reply-runtime` | 공유 답장 runtime | 인바운드 디스패치, heartbeat, 답장 planner, chunking |
  | `plugin-sdk/reply-dispatch-runtime` | 좁은 답장 디스패치 helper | finalize + provider dispatch helper |
  | `plugin-sdk/reply-history` | 답장 기록 helper | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | 답장 참조 계획 | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | 답장 청크 helper | 텍스트/markdown chunking helper |
  | `plugin-sdk/session-store-runtime` | 세션 저장소 helper | 저장소 경로 + updated-at helper |
  | `plugin-sdk/state-paths` | 상태 경로 helper | 상태 및 OAuth 디렉터리 helper |
  | `plugin-sdk/routing` | 라우팅/세션 키 helper | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, 세션 키 정규화 helper |
  | `plugin-sdk/status-helpers` | 채널 상태 helper | 채널/계정 상태 요약 빌더, runtime-state 기본값, issue 메타데이터 helper |
  | `plugin-sdk/target-resolver-runtime` | 대상 resolver helper | 공유 대상 resolver helper |
  | `plugin-sdk/string-normalization-runtime` | 문자열 정규화 helper | slug/문자열 정규화 helper |
  | `plugin-sdk/request-url` | 요청 URL helper | request 유사 입력에서 문자열 URL 추출 |
  | `plugin-sdk/run-command` | 시간 제한 명령 helper | 정규화된 stdout/stderr를 포함한 시간 제한 명령 실행기 |
  | `plugin-sdk/param-readers` | 파라미터 리더 | 공통 tool/CLI 파라미터 리더 |
  | `plugin-sdk/tool-payload` | tool payload 추출 | tool 결과 객체에서 정규화된 payload 추출 |
  | `plugin-sdk/tool-send` | tool send 추출 | tool 인수에서 정식 send 대상 필드 추출 |
  | `plugin-sdk/temp-path` | 임시 경로 helper | 공유 임시 다운로드 경로 helper |
  | `plugin-sdk/logging-core` | logging helper | 서브시스템 logger 및 민감 정보 제거 helper |
  | `plugin-sdk/markdown-table-runtime` | Markdown 표 helper | Markdown 표 모드 helper |
  | `plugin-sdk/reply-payload` | 메시지 답장 타입 | 답장 payload 타입 |
  | `plugin-sdk/provider-setup` | 엄선된 로컬/self-hosted provider setup helper | self-hosted provider 검색/config helper |
  | `plugin-sdk/self-hosted-provider-setup` | 집중된 OpenAI 호환 self-hosted provider setup helper | 동일한 self-hosted provider 검색/config helper |
  | `plugin-sdk/provider-auth-runtime` | provider runtime auth helper | runtime API 키 해석 helper |
  | `plugin-sdk/provider-auth-api-key` | provider API 키 setup helper | API 키 온보딩/profile-write helper |
  | `plugin-sdk/provider-auth-result` | provider auth-result helper | 표준 OAuth auth-result 빌더 |
  | `plugin-sdk/provider-auth-login` | provider 대화형 login helper | 공유 대화형 login helper |
  | `plugin-sdk/provider-env-vars` | provider env-var helper | provider auth env-var 조회 helper |
  | `plugin-sdk/provider-model-shared` | 공유 provider 모델/replay helper | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, 공유 replay-policy 빌더, provider-endpoint helper, 모델 ID 정규화 helper |
  | `plugin-sdk/provider-catalog-shared` | 공유 provider 카탈로그 helper | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | provider 온보딩 패치 | 온보딩 config helper |
  | `plugin-sdk/provider-http` | provider HTTP helper | 일반 provider HTTP/endpoint capability helper |
  | `plugin-sdk/provider-web-fetch` | provider web-fetch helper | web-fetch provider 등록/캐시 helper |
  | `plugin-sdk/provider-web-search-config-contract` | provider web-search config helper | plugin-enable 연결이 필요하지 않은 providers를 위한 좁은 web-search config/자격 증명 helper |
  | `plugin-sdk/provider-web-search-contract` | provider web-search 계약 helper | `createWebSearchProviderContractFields`, `enablePluginInConfig`, `resolveProviderWebSearchPluginConfig`, 범위가 지정된 자격 증명 setter/getter 같은 좁은 web-search config/자격 증명 계약 helper |
  | `plugin-sdk/provider-web-search` | provider web-search helper | web-search provider 등록/캐시/runtime helper |
  | `plugin-sdk/provider-tools` | provider tool/schema 호환 helper | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini 스키마 정리 + 진단, 그리고 `resolveXaiModelCompatPatch` / `applyXaiModelCompat` 같은 xAI 호환 helper |
  | `plugin-sdk/provider-usage` | provider 사용량 helper | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage` 및 기타 provider 사용량 helper |
  | `plugin-sdk/provider-stream` | provider stream wrapper helper | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, stream wrapper 타입, 그리고 공유 Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot wrapper helper |
  | `plugin-sdk/keyed-async-queue` | 순서가 보장된 async 큐 | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | 공유 미디어 helper | 미디어 fetch/transform/store helper와 미디어 payload 빌더 |
  | `plugin-sdk/media-generation-runtime` | 공유 media-generation helper | 공유 failover helper, 후보 선택, 이미지/비디오/음악 생성을 위한 모델 누락 메시지 |
  | `plugin-sdk/media-understanding` | media-understanding helper | media understanding provider 타입과 provider 대상 image/audio helper export |
  | `plugin-sdk/text-runtime` | 공유 텍스트 helper | assistant-visible-text 제거, markdown 렌더/chunking/표 helper, 민감 정보 제거 helper, directive-tag helper, safe-text 유틸리티, 기타 관련 text/logging helper |
  | `plugin-sdk/text-chunking` | 텍스트 chunking helper | 아웃바운드 텍스트 chunking helper |
  | `plugin-sdk/speech` | speech helper | speech provider 타입과 provider 대상 directive, registry, validation helper |
  | `plugin-sdk/speech-core` | 공유 speech core | speech provider 타입, registry, directive, 정규화 |
  | `plugin-sdk/realtime-transcription` | realtime transcription helper | provider 타입 및 registry helper |
  | `plugin-sdk/realtime-voice` | realtime voice helper | provider 타입 및 registry helper |
  | `plugin-sdk/image-generation-core` | 공유 image-generation core | image-generation 타입, failover, auth, registry helper |
  | `plugin-sdk/music-generation` | music-generation helper | music-generation provider/request/result 타입 |
  | `plugin-sdk/music-generation-core` | 공유 music-generation core | music-generation 타입, failover helper, provider 조회, model-ref 구문 분석 |
  | `plugin-sdk/video-generation` | video-generation helper | video-generation provider/request/result 타입 |
  | `plugin-sdk/video-generation-core` | 공유 video-generation core | video-generation 타입, failover helper, provider 조회, model-ref 구문 분석 |
  | `plugin-sdk/interactive-runtime` | interactive 답장 helper | interactive 답장 payload 정규화/축소 |
  | `plugin-sdk/channel-config-primitives` | 채널 config 기본 구성 요소 | 좁은 채널 config-schema 기본 구성 요소 |
  | `plugin-sdk/channel-config-writes` | 채널 config-write helper | 채널 config-write authorization helper |
  | `plugin-sdk/channel-plugin-common` | 공유 채널 prelude | 공유 채널 plugin prelude export |
  | `plugin-sdk/channel-status` | 채널 상태 helper | 공유 채널 상태 스냅샷/요약 helper |
  | `plugin-sdk/allowlist-config-edit` | 허용 목록 config helper | 허용 목록 config edit/read helper |
  | `plugin-sdk/group-access` | 그룹 접근 helper | 공유 group-access 결정 helper |
  | `plugin-sdk/direct-dm` | 직접 DM helper | 공유 직접 DM auth/guard helper |
  | `plugin-sdk/extension-shared` | 공유 extension helper | passive-channel/status 및 ambient proxy helper 기본 구성 요소 |
  | `plugin-sdk/webhook-targets` | webhook 대상 helper | webhook 대상 registry 및 route-install helper |
  | `plugin-sdk/webhook-path` | webhook 경로 helper | webhook 경로 정규화 helper |
  | `plugin-sdk/web-media` | 공유 web media helper | 원격/로컬 미디어 로딩 helper |
  | `plugin-sdk/zod` | Zod 재내보내기 | plugin SDK 소비자를 위한 재내보낸 `zod` |
  | `plugin-sdk/memory-core` | 번들 memory-core helper | 메모리 관리자/config/파일/CLI helper 표면 |
  | `plugin-sdk/memory-core-engine-runtime` | 메모리 엔진 runtime 파사드 | 메모리 인덱스/검색 runtime 파사드 |
  | `plugin-sdk/memory-core-host-engine-foundation` | 메모리 호스트 기반 엔진 | 메모리 호스트 기반 엔진 export |
  | `plugin-sdk/memory-core-host-engine-embeddings` | 메모리 호스트 임베딩 엔진 | 메모리 호스트 임베딩 엔진 export |
  | `plugin-sdk/memory-core-host-engine-qmd` | 메모리 호스트 QMD 엔진 | 메모리 호스트 QMD 엔진 export |
  | `plugin-sdk/memory-core-host-engine-storage` | 메모리 호스트 스토리지 엔진 | 메모리 호스트 스토리지 엔진 export |
  | `plugin-sdk/memory-core-host-multimodal` | 메모리 호스트 멀티모달 helper | 메모리 호스트 멀티모달 helper |
  | `plugin-sdk/memory-core-host-query` | 메모리 호스트 쿼리 helper | 메모리 호스트 쿼리 helper |
  | `plugin-sdk/memory-core-host-secret` | 메모리 호스트 비밀 helper | 메모리 호스트 비밀 helper |
  | `plugin-sdk/memory-core-host-events` | 메모리 호스트 이벤트 저널 helper | 메모리 호스트 이벤트 저널 helper |
  | `plugin-sdk/memory-core-host-status` | 메모리 호스트 상태 helper | 메모리 호스트 상태 helper |
  | `plugin-sdk/memory-core-host-runtime-cli` | 메모리 호스트 CLI runtime | 메모리 호스트 CLI runtime helper |
  | `plugin-sdk/memory-core-host-runtime-core` | 메모리 호스트 core runtime | 메모리 호스트 core runtime helper |
  | `plugin-sdk/memory-core-host-runtime-files` | 메모리 호스트 파일/runtime helper | 메모리 호스트 파일/runtime helper |
  | `plugin-sdk/memory-host-core` | 메모리 호스트 core runtime 별칭 | 메모리 호스트 core runtime helper를 위한 vendor-neutral 별칭 |
  | `plugin-sdk/memory-host-events` | 메모리 호스트 이벤트 저널 별칭 | 메모리 호스트 이벤트 저널 helper를 위한 vendor-neutral 별칭 |
  | `plugin-sdk/memory-host-files` | 메모리 호스트 파일/runtime 별칭 | 메모리 호스트 파일/runtime helper를 위한 vendor-neutral 별칭 |
  | `plugin-sdk/memory-host-markdown` | 관리형 markdown helper | memory 인접 plugins를 위한 공유 관리형 markdown helper |
  | `plugin-sdk/memory-host-search` | 활성 메모리 검색 파사드 | 지연 로드되는 활성 메모리 search-manager runtime 파사드 |
  | `plugin-sdk/memory-host-status` | 메모리 호스트 상태 별칭 | 메모리 호스트 상태 helper를 위한 vendor-neutral 별칭 |
  | `plugin-sdk/memory-lancedb` | 번들 memory-lancedb helper | memory-lancedb helper 표면 |
  | `plugin-sdk/testing` | 테스트 유틸리티 | 테스트 helper 및 mock |
</Accordion>

이 표는 전체 SDK
표면이 아니라 의도적으로 일반적인 마이그레이션 하위 집합만 포함합니다. 200개가 넘는 전체 entrypoint 목록은
`scripts/lib/plugin-sdk-entrypoints.json`에 있습니다.

해당 목록에는 여전히
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, `plugin-sdk/matrix*` 같은 일부 번들-plugin helper seam도 포함되어 있습니다. 이들은 번들-plugin 유지 관리와 호환성을 위해 계속 export되지만,
의도적으로 일반 마이그레이션 표에서는 제외되어 있으며
새 plugin 코드에 권장되는 대상이 아닙니다.

같은 규칙은 다음과 같은 다른 번들-helper 계열에도 적용됩니다.

- 브라우저 지원 helper: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
- Matrix: `plugin-sdk/matrix*`
- LINE: `plugin-sdk/line*`
- IRC: `plugin-sdk/irc*`
- `plugin-sdk/googlechat`,
  `plugin-sdk/zalouser`, `plugin-sdk/bluebubbles*`,
  `plugin-sdk/mattermost*`, `plugin-sdk/msteams`,
  `plugin-sdk/nextcloud-talk`, `plugin-sdk/nostr`, `plugin-sdk/tlon`,
  `plugin-sdk/twitch`,
  `plugin-sdk/github-copilot-login`, `plugin-sdk/github-copilot-token`,
  `plugin-sdk/diagnostics-otel`, `plugin-sdk/diffs`, `plugin-sdk/llm-task`,
  `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` 같은 번들 helper/plugin 표면

`plugin-sdk/github-copilot-token`은 현재
`DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken`, `resolveCopilotApiToken`이라는 좁은 토큰-helper
표면을 노출합니다.

작업에 맞는 가장 좁은 import를 사용하세요. export를 찾을 수 없다면,
`src/plugin-sdk/`의 소스를 확인하거나 Discord에서 문의하세요.

## 제거 일정

| 시점 | 발생하는 일 |
| ---------------------- | ----------------------------------------------------------------------- |
| **지금** | 사용 중단된 표면이 런타임 경고를 출력함 |
| **다음 메이저 릴리스** | 사용 중단된 표면이 제거되며, 여전히 이를 사용하는 plugins는 실패함 |

모든 core plugins는 이미 마이그레이션되었습니다. 외부 plugins도 다음 메이저 릴리스 전에
마이그레이션해야 합니다.

## 경고를 일시적으로 숨기기

마이그레이션 작업 중에는 다음 환경 변수를 설정하세요.

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

이것은 임시 탈출구일 뿐, 영구적인 해결책은 아닙니다.

## 관련 문서

- [시작하기](/ko/plugins/building-plugins) — 첫 plugin 만들기
- [SDK 개요](/ko/plugins/sdk-overview) — 전체 하위 경로 import 참조
- [채널 Plugins](/ko/plugins/sdk-channel-plugins) — 채널 plugins 만들기
- [Provider Plugins](/ko/plugins/sdk-provider-plugins) — provider plugins 만들기
- [Plugin 내부 구조](/ko/plugins/architecture) — 아키텍처 심화 설명
- [Plugin Manifest](/ko/plugins/manifest) — manifest 스키마 참조
