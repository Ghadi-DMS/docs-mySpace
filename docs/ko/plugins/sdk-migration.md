---
read_when:
    - OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED 경고가 표시될 때
    - OPENCLAW_EXTENSION_API_DEPRECATED 경고가 표시될 때
    - plugin을 최신 plugin 아키텍처로 업데이트할 때
    - 외부 OpenClaw plugin을 유지 관리할 때
sidebarTitle: Migrate to SDK
summary: 레거시 하위 호환 계층에서 최신 plugin SDK로 마이그레이션
title: Plugin SDK 마이그레이션
x-i18n:
    generated_at: "2026-04-06T03:10:23Z"
    model: gpt-5.4
    provider: openai
    source_hash: b71ce69b30c3bb02da1b263b1d11dc3214deae5f6fc708515e23b5a1c7bb7c8f
    source_path: plugins/sdk-migration.md
    workflow: 15
---

# Plugin SDK 마이그레이션

OpenClaw는 광범위한 하위 호환 계층에서, 집중된 문서화된 import를 사용하는 최신 plugin
아키텍처로 이동했습니다. plugin이 새 아키텍처 이전에 작성되었다면,
이 가이드가 마이그레이션에 도움이 됩니다.

## 변경 사항

기존 plugin 시스템은 plugin이 단일 진입점에서 필요한 모든 것을 import할 수 있게 해 주는
두 개의 광범위한 표면을 제공했습니다:

- **`openclaw/plugin-sdk/compat`** — 수십 개의
  헬퍼를 재내보내는 단일 import입니다. 새 plugin 아키텍처가 구축되는 동안 오래된 훅 기반 plugin이 계속 동작하도록 도입되었습니다.
- **`openclaw/extension-api`** — plugin에
  내장 에이전트 러너 같은 호스트 측 헬퍼에 대한 직접 접근을 제공한 브리지입니다.

이제 두 표면 모두 **deprecated** 상태입니다. 런타임에서는 여전히 동작하지만, 새
plugin은 이를 사용해서는 안 되며, 기존 plugin도 다음 주요 릴리스에서 제거되기 전에 마이그레이션해야 합니다.

<Warning>
  하위 호환 계층은 향후 주요 릴리스에서 제거됩니다.
  이 표면들에서 계속 import하는 plugin은 그 시점에 동작이 중단됩니다.
</Warning>

## 왜 바뀌었나요

기존 접근 방식은 다음과 같은 문제를 일으켰습니다:

- **느린 시작** — 하나의 헬퍼를 import하면 관련 없는 수십 개의 모듈이 함께 로드됨
- **순환 의존성** — 광범위한 재내보내기로 import 순환을 쉽게 만들 수 있었음
- **불명확한 API 표면** — 어떤 export가 안정적이고 어떤 것이 내부용인지 구분할 수 없었음

최신 plugin SDK는 이를 해결합니다. 각 import 경로(`openclaw/plugin-sdk/\<subpath\>`)는
명확한 목적과 문서화된 계약을 가진 작고 독립적인 모듈입니다.

번들 채널을 위한 레거시 provider 편의 seam도 제거되었습니다. 예를 들어
`openclaw/plugin-sdk/slack`, `openclaw/plugin-sdk/discord`,
`openclaw/plugin-sdk/signal`, `openclaw/plugin-sdk/whatsapp`,
채널 브랜드 헬퍼 seam, 그리고
`openclaw/plugin-sdk/telegram-core` 같은 import는 안정적인 plugin 계약이 아니라
비공개 mono-repo 단축 경로였습니다. 대신 더 좁고 일반적인 SDK subpath를 사용하세요. 번들
plugin 워크스페이스 내부에서는 provider 소유 헬퍼를 해당 plugin의 자체
`api.ts` 또는 `runtime-api.ts`에 유지하세요.

현재 번들 provider 예시:

- Anthropic은 Claude 전용 스트림 헬퍼를 자체 `api.ts` /
  `contract-api.ts` seam에 유지합니다
- OpenAI는 provider 빌더, 기본 모델 헬퍼, realtime provider
  빌더를 자체 `api.ts`에 유지합니다
- OpenRouter는 provider 빌더와 온보딩/config 헬퍼를 자체
  `api.ts`에 유지합니다

## 마이그레이션 방법

<Steps>
  <Step title="Windows wrapper 폴백 동작 감사">
    plugin이 `openclaw/plugin-sdk/windows-spawn`을 사용한다면, 해석되지 않는 Windows
    `.cmd`/`.bat` wrapper는 이제 `allowShellFallback: true`를
    명시적으로 전달하지 않는 한 안전하게 실패합니다.

    ```typescript
    // 이전
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // 이후
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // 셸 매개 폴백을 의도적으로 허용하는 신뢰된 호환 호출자에만
      // 이 값을 설정하세요.
      allowShellFallback: true,
    });
    ```

    호출자가 셸 폴백에 의도적으로 의존하지 않는다면
    `allowShellFallback`을 설정하지 말고 대신 발생한 오류를 처리하세요.

  </Step>

  <Step title="deprecated import 찾기">
    plugin에서 다음 deprecated 표면 중 하나로부터의 import를 검색하세요:

    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```

  </Step>

  <Step title="집중된 import로 교체">
    기존 표면의 각 export는 특정 최신 import 경로에 매핑됩니다:

    ```typescript
    // 이전 (deprecated 하위 호환 계층)
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

    호스트 측 헬퍼의 경우 직접 import하는 대신
    주입된 plugin runtime을 사용하세요:

    ```typescript
    // 이전 (deprecated extension-api 브리지)
    import { runEmbeddedPiAgent } from "openclaw/extension-api";
    const result = await runEmbeddedPiAgent({ sessionId, prompt });

    // 이후 (주입된 runtime)
    const result = await api.runtime.agent.runEmbeddedPiAgent({ sessionId, prompt });
    ```

    동일한 패턴이 다른 레거시 브리지 헬퍼에도 적용됩니다:

    | 기존 import | 최신 대체 항목 |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | session store helpers | `api.runtime.agent.session.*` |

  </Step>

  <Step title="빌드 및 테스트">
    ```bash
    pnpm build
    pnpm test -- my-plugin/
    ```
  </Step>
</Steps>

## Import 경로 참조

<Accordion title="일반적인 import 경로 표">
  | Import path | 용도 | 주요 export |
  | --- | --- | --- |
  | `plugin-sdk/plugin-entry` | 정식 plugin 진입 헬퍼 | `definePluginEntry` |
  | `plugin-sdk/core` | 채널 진입 정의/빌더를 위한 레거시 umbrella 재내보내기 | `defineChannelPluginEntry`, `createChatChannelPlugin` |
  | `plugin-sdk/config-schema` | 루트 config 스키마 export | `OpenClawSchema` |
  | `plugin-sdk/provider-entry` | 단일 provider 진입 헬퍼 | `defineSingleProviderPluginEntry` |
  | `plugin-sdk/channel-core` | 집중형 채널 진입 정의 및 빌더 | `defineChannelPluginEntry`, `defineSetupPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase` |
  | `plugin-sdk/setup` | 공통 설정 마법사 헬퍼 | Allowlist 프롬프트, 설정 상태 빌더 |
  | `plugin-sdk/setup-runtime` | 설정 시점 runtime 헬퍼 | import-safe 설정 패치 어댑터, lookup-note 헬퍼, `promptResolvedAllowFrom`, `splitSetupEntries`, 위임된 설정 프록시 |
  | `plugin-sdk/setup-adapter-runtime` | 설정 어댑터 헬퍼 | `createEnvPatchedAccountSetupAdapter` |
  | `plugin-sdk/setup-tools` | 설정 도구 헬퍼 | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR` |
  | `plugin-sdk/account-core` | 다중 계정 헬퍼 | 계정 목록/config/액션 게이트 헬퍼 |
  | `plugin-sdk/account-id` | account-id 헬퍼 | `DEFAULT_ACCOUNT_ID`, account-id 정규화 |
  | `plugin-sdk/account-resolution` | 계정 조회 헬퍼 | 계정 조회 + 기본값 폴백 헬퍼 |
  | `plugin-sdk/account-helpers` | 좁은 범위의 계정 헬퍼 | 계정 목록/계정 액션 헬퍼 |
  | `plugin-sdk/channel-setup` | 설정 마법사 어댑터 | `createOptionalChannelSetupSurface`, `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`, 그리고 `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`, `setSetupChannelEnabled`, `splitSetupEntries` |
  | `plugin-sdk/channel-pairing` | DM 페어링 기본 요소 | `createChannelPairingController` |
  | `plugin-sdk/channel-reply-pipeline` | 답변 접두사 + typing 연결 | `createChannelReplyPipeline` |
  | `plugin-sdk/channel-config-helpers` | Config 어댑터 팩토리 | `createHybridChannelConfigAdapter` |
  | `plugin-sdk/channel-config-schema` | Config 스키마 빌더 | 채널 config 스키마 타입 |
  | `plugin-sdk/telegram-command-config` | Telegram 명령 config 헬퍼 | 명령 이름 정규화, 설명 자르기, 중복/충돌 검증 |
  | `plugin-sdk/channel-policy` | 그룹/DM 정책 해석 | `resolveChannelGroupRequireMention` |
  | `plugin-sdk/channel-lifecycle` | 계정 상태 추적 | `createAccountStatusSink` |
  | `plugin-sdk/inbound-envelope` | 인바운드 envelope 헬퍼 | 공통 route + envelope 빌더 헬퍼 |
  | `plugin-sdk/inbound-reply-dispatch` | 인바운드 답변 헬퍼 | 공통 기록 및 디스패치 헬퍼 |
  | `plugin-sdk/messaging-targets` | 메시징 대상 파싱 | 대상 파싱/매칭 헬퍼 |
  | `plugin-sdk/outbound-media` | 아웃바운드 미디어 헬퍼 | 공통 아웃바운드 미디어 로드 |
  | `plugin-sdk/outbound-runtime` | 아웃바운드 runtime 헬퍼 | 아웃바운드 ID/전송 위임 헬퍼 |
  | `plugin-sdk/thread-bindings-runtime` | 스레드 바인딩 헬퍼 | 스레드 바인딩 수명 주기 및 어댑터 헬퍼 |
  | `plugin-sdk/agent-media-payload` | 레거시 미디어 payload 헬퍼 | 레거시 필드 레이아웃용 에이전트 미디어 payload 빌더 |
  | `plugin-sdk/channel-runtime` | deprecated 호환성 shim | 레거시 채널 runtime 유틸리티 전용 |
  | `plugin-sdk/channel-send-result` | 전송 결과 타입 | 답변 결과 타입 |
  | `plugin-sdk/runtime-store` | 영속 plugin 저장소 | `createPluginRuntimeStore` |
  | `plugin-sdk/runtime` | 광범위한 runtime 헬퍼 | runtime/로깅/백업/plugin 설치 헬퍼 |
  | `plugin-sdk/runtime-env` | 좁은 범위의 runtime env 헬퍼 | 로거/runtime env, timeout, retry, backoff 헬퍼 |
  | `plugin-sdk/plugin-runtime` | 공통 plugin runtime 헬퍼 | plugin 명령/hooks/http/interactive 헬퍼 |
  | `plugin-sdk/hook-runtime` | hook 파이프라인 헬퍼 | 공통 webhook/internal hook 파이프라인 헬퍼 |
  | `plugin-sdk/lazy-runtime` | 지연 runtime 헬퍼 | `createLazyRuntimeModule`, `createLazyRuntimeMethod`, `createLazyRuntimeMethodBinder`, `createLazyRuntimeNamedExport`, `createLazyRuntimeSurface` |
  | `plugin-sdk/process-runtime` | 프로세스 헬퍼 | 공통 exec 헬퍼 |
  | `plugin-sdk/cli-runtime` | CLI runtime 헬퍼 | 명령 포매팅, 대기, 버전 헬퍼 |
  | `plugin-sdk/gateway-runtime` | 게이트웨이 헬퍼 | 게이트웨이 클라이언트 및 채널 상태 패치 헬퍼 |
  | `plugin-sdk/config-runtime` | Config 헬퍼 | config 로드/쓰기 헬퍼 |
  | `plugin-sdk/telegram-command-config` | Telegram 명령 헬퍼 | 번들 Telegram 계약 표면을 사용할 수 없을 때의 폴백 안정형 Telegram 명령 검증 헬퍼 |
  | `plugin-sdk/approval-runtime` | 승인 프롬프트 헬퍼 | exec/plugin 승인 payload, 승인 capability/profile 헬퍼, 네이티브 승인 라우팅/runtime 헬퍼 |
  | `plugin-sdk/approval-auth-runtime` | 승인 auth 헬퍼 | 승인자 해석, same-chat 액션 auth |
  | `plugin-sdk/approval-client-runtime` | 승인 클라이언트 헬퍼 | 네이티브 exec 승인 프로필/필터 헬퍼 |
  | `plugin-sdk/approval-delivery-runtime` | 승인 전송 헬퍼 | 네이티브 승인 capability/전송 어댑터 |
  | `plugin-sdk/approval-native-runtime` | 승인 대상 헬퍼 | 네이티브 승인 대상/계정 바인딩 헬퍼 |
  | `plugin-sdk/approval-reply-runtime` | 승인 답변 헬퍼 | exec/plugin 승인 답변 payload 헬퍼 |
  | `plugin-sdk/security-runtime` | 보안 헬퍼 | 공통 trust, DM 게이팅, external-content, secret-collection 헬퍼 |
  | `plugin-sdk/ssrf-policy` | SSRF 정책 헬퍼 | 호스트 allowlist 및 private-network 정책 헬퍼 |
  | `plugin-sdk/ssrf-runtime` | SSRF runtime 헬퍼 | pinned-dispatcher, guarded fetch, SSRF 정책 헬퍼 |
  | `plugin-sdk/collection-runtime` | 제한된 캐시 헬퍼 | `pruneMapToMaxSize` |
  | `plugin-sdk/diagnostic-runtime` | 진단 게이팅 헬퍼 | `isDiagnosticFlagEnabled`, `isDiagnosticsEnabled` |
  | `plugin-sdk/error-runtime` | 오류 포매팅 헬퍼 | `formatUncaughtError`, `isApprovalNotFoundError`, 오류 그래프 헬퍼 |
  | `plugin-sdk/fetch-runtime` | 래핑된 fetch/프록시 헬퍼 | `resolveFetch`, 프록시 헬퍼 |
  | `plugin-sdk/host-runtime` | 호스트 정규화 헬퍼 | `normalizeHostname`, `normalizeScpRemoteHost` |
  | `plugin-sdk/retry-runtime` | 재시도 헬퍼 | `RetryConfig`, `retryAsync`, 정책 실행기 |
  | `plugin-sdk/allow-from` | Allowlist 포매팅 | `formatAllowFromLowercase` |
  | `plugin-sdk/allowlist-resolution` | Allowlist 입력 매핑 | `mapAllowlistResolutionInputs` |
  | `plugin-sdk/command-auth` | 명령 게이팅 및 명령 표면 헬퍼 | `resolveControlCommandGate`, 발신자 권한 부여 헬퍼, 명령 레지스트리 헬퍼 |
  | `plugin-sdk/secret-input` | secret 입력 파싱 | secret 입력 헬퍼 |
  | `plugin-sdk/webhook-ingress` | webhook 요청 헬퍼 | webhook 대상 유틸리티 |
  | `plugin-sdk/webhook-request-guards` | webhook 본문 가드 헬퍼 | 요청 본문 읽기/제한 헬퍼 |
  | `plugin-sdk/reply-runtime` | 공통 답변 runtime | 인바운드 디스패치, heartbeat, 답변 planner, 청킹 |
  | `plugin-sdk/reply-dispatch-runtime` | 좁은 범위의 답변 디스패치 헬퍼 | finalize + provider 디스패치 헬퍼 |
  | `plugin-sdk/reply-history` | 답변 기록 헬퍼 | `buildHistoryContext`, `buildPendingHistoryContextFromMap`, `recordPendingHistoryEntry`, `clearHistoryEntriesIfEnabled` |
  | `plugin-sdk/reply-reference` | 답변 참조 계획 | `createReplyReferencePlanner` |
  | `plugin-sdk/reply-chunking` | 답변 청크 헬퍼 | 텍스트/markdown 청킹 헬퍼 |
  | `plugin-sdk/session-store-runtime` | 세션 저장소 헬퍼 | 저장소 경로 + updated-at 헬퍼 |
  | `plugin-sdk/state-paths` | 상태 경로 헬퍼 | 상태 및 OAuth 디렉터리 헬퍼 |
  | `plugin-sdk/routing` | 라우팅/세션 키 헬퍼 | `resolveAgentRoute`, `buildAgentSessionKey`, `resolveDefaultAgentBoundAccountId`, 세션 키 정규화 헬퍼 |
  | `plugin-sdk/status-helpers` | 채널 상태 헬퍼 | 채널/계정 상태 요약 빌더, runtime-state 기본값, 이슈 메타데이터 헬퍼 |
  | `plugin-sdk/target-resolver-runtime` | 대상 해석기 헬퍼 | 공통 대상 해석기 헬퍼 |
  | `plugin-sdk/string-normalization-runtime` | 문자열 정규화 헬퍼 | slug/문자열 정규화 헬퍼 |
  | `plugin-sdk/request-url` | 요청 URL 헬퍼 | request 유사 입력에서 문자열 URL 추출 |
  | `plugin-sdk/run-command` | 시간 측정 명령 헬퍼 | 정규화된 stdout/stderr를 갖는 시간 측정 명령 실행기 |
  | `plugin-sdk/param-readers` | 파라미터 리더 | 공통 도구/CLI 파라미터 리더 |
  | `plugin-sdk/tool-send` | 도구 전송 추출 | 도구 인수에서 정규 send 대상 필드 추출 |
  | `plugin-sdk/temp-path` | 임시 경로 헬퍼 | 공통 임시 다운로드 경로 헬퍼 |
  | `plugin-sdk/logging-core` | 로깅 헬퍼 | 서브시스템 로거 및 redaction 헬퍼 |
  | `plugin-sdk/markdown-table-runtime` | Markdown 표 헬퍼 | Markdown 표 모드 헬퍼 |
  | `plugin-sdk/reply-payload` | 메시지 답변 타입 | 답변 payload 타입 |
  | `plugin-sdk/provider-setup` | 선별된 로컬/자체 호스팅 provider 설정 헬퍼 | 자체 호스팅 provider 탐지/config 헬퍼 |
  | `plugin-sdk/self-hosted-provider-setup` | 집중형 OpenAI 호환 자체 호스팅 provider 설정 헬퍼 | 동일한 자체 호스팅 provider 탐지/config 헬퍼 |
  | `plugin-sdk/provider-auth-runtime` | provider runtime auth 헬퍼 | runtime API 키 해석 헬퍼 |
  | `plugin-sdk/provider-auth-api-key` | provider API 키 설정 헬퍼 | API 키 온보딩/프로필 쓰기 헬퍼 |
  | `plugin-sdk/provider-auth-result` | provider auth-result 헬퍼 | 표준 OAuth auth-result 빌더 |
  | `plugin-sdk/provider-auth-login` | provider 대화형 로그인 헬퍼 | 공통 대화형 로그인 헬퍼 |
  | `plugin-sdk/provider-env-vars` | provider 환경 변수 헬퍼 | provider auth 환경 변수 조회 헬퍼 |
  | `plugin-sdk/provider-model-shared` | 공통 provider 모델/리플레이 헬퍼 | `ProviderReplayFamily`, `buildProviderReplayFamilyHooks`, `normalizeModelCompat`, 공통 replay-policy 빌더, provider-endpoint 헬퍼, 모델 ID 정규화 헬퍼 |
  | `plugin-sdk/provider-catalog-shared` | 공통 provider 카탈로그 헬퍼 | `findCatalogTemplate`, `buildSingleProviderApiKeyCatalog`, `supportsNativeStreamingUsageCompat`, `applyProviderNativeStreamingUsageCompat` |
  | `plugin-sdk/provider-onboard` | provider 온보딩 패치 | 온보딩 config 헬퍼 |
  | `plugin-sdk/provider-http` | provider HTTP 헬퍼 | 일반 provider HTTP/endpoint capability 헬퍼 |
  | `plugin-sdk/provider-web-fetch` | provider web-fetch 헬퍼 | web-fetch provider 등록/캐시 헬퍼 |
  | `plugin-sdk/provider-web-search` | provider web-search 헬퍼 | web-search provider 등록/캐시/config 헬퍼 |
  | `plugin-sdk/provider-tools` | provider 도구/스키마 호환성 헬퍼 | `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks`, Gemini 스키마 정리 + diagnostics, 그리고 `resolveXaiModelCompatPatch` / `applyXaiModelCompat` 같은 xAI 호환성 헬퍼 |
  | `plugin-sdk/provider-usage` | provider 사용량 헬퍼 | `fetchClaudeUsage`, `fetchGeminiUsage`, `fetchGithubCopilotUsage`, 기타 provider 사용량 헬퍼 |
  | `plugin-sdk/provider-stream` | provider 스트림 래퍼 헬퍼 | `ProviderStreamFamily`, `buildProviderStreamFamilyHooks`, `composeProviderStreamWrappers`, 스트림 래퍼 타입, 공통 Anthropic/Bedrock/Google/Kilocode/Moonshot/OpenAI/OpenRouter/Z.A.I/MiniMax/Copilot 래퍼 헬퍼 |
  | `plugin-sdk/keyed-async-queue` | 순서 보장 async 대기열 | `KeyedAsyncQueue` |
  | `plugin-sdk/media-runtime` | 공통 미디어 헬퍼 | 미디어 fetch/변환/저장 헬퍼와 미디어 payload 빌더 |
  | `plugin-sdk/media-understanding` | media-understanding 헬퍼 | media understanding provider 타입과 provider 측 이미지/오디오 헬퍼 export |
  | `plugin-sdk/text-runtime` | 공통 텍스트 헬퍼 | assistant 표시 텍스트 제거, markdown 렌더/청킹/표 헬퍼, redaction 헬퍼, directive-tag 헬퍼, safe-text 유틸리티, 관련 텍스트/로깅 헬퍼 |
  | `plugin-sdk/text-chunking` | 텍스트 청킹 헬퍼 | 아웃바운드 텍스트 청킹 헬퍼 |
  | `plugin-sdk/speech` | 음성 헬퍼 | 음성 provider 타입과 provider 측 directive, 레지스트리, 검증 헬퍼 |
  | `plugin-sdk/speech-core` | 공통 음성 코어 | 음성 provider 타입, 레지스트리, directive, 정규화 |
  | `plugin-sdk/realtime-transcription` | realtime 전사 헬퍼 | provider 타입 및 레지스트리 헬퍼 |
  | `plugin-sdk/realtime-voice` | realtime 음성 헬퍼 | provider 타입 및 레지스트리 헬퍼 |
  | `plugin-sdk/image-generation-core` | 공통 이미지 생성 코어 | 이미지 생성 타입, 장애 조치, auth, 레지스트리 헬퍼 |
  | `plugin-sdk/music-generation` | 음악 생성 헬퍼 | 음악 생성 provider/요청/결과 타입 |
  | `plugin-sdk/music-generation-core` | 공통 음악 생성 코어 | 음악 생성 타입, 장애 조치 헬퍼, provider 조회, model-ref 파싱 |
  | `plugin-sdk/video-generation` | 비디오 생성 헬퍼 | 비디오 생성 provider/요청/결과 타입 |
  | `plugin-sdk/video-generation-core` | 공통 비디오 생성 코어 | 비디오 생성 타입, 장애 조치 헬퍼, provider 조회, model-ref 파싱 |
  | `plugin-sdk/interactive-runtime` | interactive 답변 헬퍼 | interactive 답변 payload 정규화/축소 |
  | `plugin-sdk/channel-config-primitives` | 채널 config 기본 요소 | 좁은 범위의 채널 config-schema 기본 요소 |
  | `plugin-sdk/channel-config-writes` | 채널 config-write 헬퍼 | 채널 config-write 권한 부여 헬퍼 |
  | `plugin-sdk/channel-plugin-common` | 공통 채널 프렐류드 | 공통 채널 plugin 프렐류드 export |
  | `plugin-sdk/channel-status` | 채널 상태 헬퍼 | 공통 채널 상태 스냅샷/요약 헬퍼 |
  | `plugin-sdk/allowlist-config-edit` | Allowlist config 헬퍼 | Allowlist config 편집/읽기 헬퍼 |
  | `plugin-sdk/group-access` | 그룹 접근 헬퍼 | 공통 그룹 접근 결정 헬퍼 |
  | `plugin-sdk/direct-dm` | Direct-DM 헬퍼 | 공통 direct-DM auth/가드 헬퍼 |
  | `plugin-sdk/extension-shared` | 공통 extension 헬퍼 | 수동 채널/상태 헬퍼 기본 요소 |
  | `plugin-sdk/webhook-targets` | webhook 대상 헬퍼 | webhook 대상 레지스트리 및 route 설치 헬퍼 |
  | `plugin-sdk/webhook-path` | webhook 경로 헬퍼 | webhook 경로 정규화 헬퍼 |
  | `plugin-sdk/web-media` | 공통 웹 미디어 헬퍼 | 원격/로컬 미디어 로드 헬퍼 |
  | `plugin-sdk/zod` | Zod 재내보내기 | plugin SDK 소비자를 위한 `zod` 재내보내기 |
  | `plugin-sdk/memory-core` | 번들 memory-core 헬퍼 | Memory manager/config/file/CLI 헬퍼 표면 |
  | `plugin-sdk/memory-core-engine-runtime` | Memory 엔진 runtime facade | Memory index/search runtime facade |
  | `plugin-sdk/memory-core-host-engine-foundation` | Memory host foundation 엔진 | Memory host foundation 엔진 export |
  | `plugin-sdk/memory-core-host-engine-embeddings` | Memory host embedding 엔진 | Memory host embedding 엔진 export |
  | `plugin-sdk/memory-core-host-engine-qmd` | Memory host QMD 엔진 | Memory host QMD 엔진 export |
  | `plugin-sdk/memory-core-host-engine-storage` | Memory host storage 엔진 | Memory host storage 엔진 export |
  | `plugin-sdk/memory-core-host-multimodal` | Memory host 멀티모달 헬퍼 | Memory host 멀티모달 헬퍼 |
  | `plugin-sdk/memory-core-host-query` | Memory host 쿼리 헬퍼 | Memory host 쿼리 헬퍼 |
  | `plugin-sdk/memory-core-host-secret` | Memory host secret 헬퍼 | Memory host secret 헬퍼 |
  | `plugin-sdk/memory-core-host-status` | Memory host 상태 헬퍼 | Memory host 상태 헬퍼 |
  | `plugin-sdk/memory-core-host-runtime-cli` | Memory host CLI runtime | Memory host CLI runtime 헬퍼 |
  | `plugin-sdk/memory-core-host-runtime-core` | Memory host core runtime | Memory host core runtime 헬퍼 |
  | `plugin-sdk/memory-core-host-runtime-files` | Memory host 파일/runtime 헬퍼 | Memory host 파일/runtime 헬퍼 |
  | `plugin-sdk/memory-lancedb` | 번들 memory-lancedb 헬퍼 | Memory-lancedb 헬퍼 표면 |
  | `plugin-sdk/testing` | 테스트 유틸리티 | 테스트 헬퍼 및 mocks |
</Accordion>

이 표는 전체 SDK
표면이 아니라 의도적으로 일반적인 마이그레이션 하위 집합만 담고 있습니다. 200개가 넘는 전체 entrypoint 목록은
`scripts/lib/plugin-sdk-entrypoints.json`에 있습니다.

이 목록에는 여전히
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, `plugin-sdk/matrix*` 같은 번들 plugin 헬퍼 seam이 포함되어 있습니다. 이들은
번들 plugin 유지보수와 호환성을 위해 계속 export되지만, 의도적으로
일반 마이그레이션 표에서는 제외되어 있으며 새 plugin 코드의 권장 대상은 아닙니다.

같은 규칙은 다음과 같은 다른 번들 헬퍼 계열에도 적용됩니다:

- 브라우저 지원 헬퍼: `plugin-sdk/browser-cdp`, `plugin-sdk/browser-config-runtime`, `plugin-sdk/browser-config-support`, `plugin-sdk/browser-control-auth`, `plugin-sdk/browser-node-runtime`, `plugin-sdk/browser-profiles`, `plugin-sdk/browser-security-runtime`, `plugin-sdk/browser-setup-tools`, `plugin-sdk/browser-support`
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
  `plugin-sdk/thread-ownership`, `plugin-sdk/voice-call` 같은
  번들 헬퍼/plugin 표면

`plugin-sdk/github-copilot-token`은 현재 다음과 같은 좁은 범위의 토큰 헬퍼
표면을 노출합니다:
`DEFAULT_COPILOT_API_BASE_URL`,
`deriveCopilotApiBaseUrlFromToken`, 그리고 `resolveCopilotApiToken`.

작업에 맞는 가장 좁은 import를 사용하세요. export를 찾을 수 없다면
`src/plugin-sdk/`의 소스를 확인하거나 Discord에서 문의하세요.

## 제거 일정

| 시점 | 발생 사항 |
| ---------------------- | ----------------------------------------------------------------------- |
| **지금** | deprecated 표면이 런타임 경고를 출력함 |
| **다음 주요 릴리스** | deprecated 표면이 제거되며, 여전히 이를 사용하는 plugin은 실패함 |

모든 core plugin은 이미 마이그레이션되었습니다. 외부 plugin도
다음 주요 릴리스 전에 마이그레이션해야 합니다.

## 경고를 임시로 억제하기

마이그레이션 작업 중에는 다음 환경 변수를 설정하세요:

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

이는 임시 탈출구일 뿐, 영구적인 해결책이 아닙니다.

## 관련 항목

- [Getting Started](/ko/plugins/building-plugins) — 첫 plugin 만들기
- [SDK Overview](/ko/plugins/sdk-overview) — 전체 subpath import 참조
- [Channel Plugins](/ko/plugins/sdk-channel-plugins) — 채널 plugin 만들기
- [Provider Plugins](/ko/plugins/sdk-provider-plugins) — provider plugin 만들기
- [Plugin Internals](/ko/plugins/architecture) — 아키텍처 심화 설명
- [Plugin Manifest](/ko/plugins/manifest) — manifest 스키마 참조
