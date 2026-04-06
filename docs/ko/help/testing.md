---
read_when:
    - 로컬 또는 CI에서 테스트를 실행할 때
    - 모델/공급자 버그에 대한 회귀를 추가할 때
    - gateway + agent 동작을 디버깅할 때
summary: '테스트 키트: unit/e2e/live 스위트, Docker 러너, 그리고 각 테스트가 다루는 내용'
title: 테스트
x-i18n:
    generated_at: "2026-04-06T03:09:35Z"
    model: gpt-5.4
    provider: openai
    source_hash: cfa174e565df5fdf957234b7909beaf1304aa026e731cc2c433ca7d931681b56
    source_path: help/testing.md
    workflow: 15
---

# 테스트

OpenClaw에는 세 가지 Vitest 스위트(unit/integration, e2e, live)와 소수의 Docker 러너가 있습니다.

이 문서는 “우리가 어떻게 테스트하는지”에 대한 가이드입니다.

- 각 스위트가 다루는 것(그리고 의도적으로 다루지 않는 것)
- 일반적인 워크플로(로컬, 푸시 전, 디버깅)에 어떤 명령을 실행할지
- live 테스트가 자격 증명을 찾고 모델/공급자를 선택하는 방식
- 실제 모델/공급자 문제에 대한 회귀를 추가하는 방법

## 빠른 시작

대부분의 날에는:

- 전체 게이트(푸시 전 기대됨): `pnpm build && pnpm check && pnpm test`
- 여유 있는 머신에서 더 빠른 로컬 전체 스위트 실행: `pnpm test:max`
- 직접 Vitest watch 루프(최신 projects config): `pnpm test:watch`
- 직접 파일 타기팅은 이제 extension/channel 경로도 라우팅합니다: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`

테스트를 수정했거나 추가 확신이 필요할 때:

- 커버리지 게이트: `pnpm test:coverage`
- E2E 스위트: `pnpm test:e2e`

실제 공급자/모델을 디버깅할 때(실제 자격 증명 필요):

- Live 스위트(모델 + gateway tool/image probe): `pnpm test:live`
- 하나의 live 파일만 조용히 타기팅: `pnpm test:live -- src/agents/models.profiles.live.test.ts`

팁: 실패하는 케이스 하나만 필요할 때는 아래에 설명된 허용 목록 환경 변수를 사용해 live 테스트 범위를 좁히는 것이 좋습니다.

## 테스트 스위트(무엇이 어디서 실행되는지)

스위트는 “현실성이 점점 높아지는 단계”(그리고 불안정성/비용도 증가)라고 생각하면 됩니다.

### Unit / integration(기본값)

- 명령: `pnpm test`
- 구성: `vitest.config.ts`를 통한 네이티브 Vitest `projects`
- 파일: `src/**/*.test.ts`, `packages/**/*.test.ts`, `test/**/*.test.ts` 아래의 core/unit 인벤토리와 `vitest.unit.config.ts`가 다루는 허용된 `ui` node 테스트
- 범위:
  - 순수 unit 테스트
  - 프로세스 내 integration 테스트(gateway auth, routing, tooling, parsing, config)
  - 알려진 버그에 대한 결정론적 회귀
- 기대 사항:
  - CI에서 실행됨
  - 실제 키 필요 없음
  - 빠르고 안정적이어야 함
- Projects 참고:
  - `pnpm test`, `pnpm test:watch`, `pnpm test:changed`는 이제 모두 동일한 네이티브 Vitest 루트 `projects` config를 사용합니다.
  - 직접 파일 필터는 루트 프로젝트 그래프를 통해 네이티브로 라우팅되므로, `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`가 사용자 지정 래퍼 없이 동작합니다.
- Embedded runner 참고:
  - 메시지 도구 탐색 입력이나 compaction 런타임 컨텍스트를 변경할 때는 두 수준의 커버리지를 모두 유지하세요.
  - 순수 routing/normalization 경계에 대해서는 집중된 helper 회귀를 추가하세요.
  - 다음 embedded runner integration 스위트도 건강하게 유지해야 합니다:
    `src/agents/pi-embedded-runner/compact.hooks.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.test.ts`,
    `src/agents/pi-embedded-runner/run.overflow-compaction.loop.test.ts`.
  - 이 스위트들은 범위 지정된 id와 compaction 동작이 실제 `run.ts` / `compact.ts` 경로를 통해 계속 흐르는지 검증합니다. helper 전용 테스트만으로는 이러한 integration 경로를 충분히 대체할 수 없습니다.
- Pool 참고:
  - 기본 Vitest config는 이제 기본적으로 `threads`를 사용합니다.
  - 공유 Vitest config는 `isolate: false`도 고정하며, 루트 projects, e2e, live config 전반에서 비격리 러너를 사용합니다.
  - 루트 UI 레인은 자체 `jsdom` 설정과 optimizer를 유지하지만, 이제 공유 비격리 러너에서도 실행됩니다.
  - `pnpm test`는 루트 `vitest.config.ts` projects config로부터 동일한 `threads` + `isolate: false` 기본값을 상속합니다.
  - 공유 `scripts/run-vitest.mjs` 런처는 큰 로컬 실행에서 V8 컴파일 churn을 줄이기 위해 이제 기본적으로 Vitest child Node 프로세스에 `--no-maglev`도 추가합니다. 기본 V8 동작과 비교해야 하면 `OPENCLAW_VITEST_ENABLE_MAGLEV=1`을 설정하세요.
- 빠른 로컬 반복 참고:
  - `pnpm test:changed`는 `--changed origin/main`과 함께 네이티브 projects config를 실행합니다.
  - `pnpm test:max`와 `pnpm test:changed:max`는 같은 네이티브 projects config를 유지하되 worker cap만 더 높습니다.
  - 로컬 worker 자동 스케일링은 이제 의도적으로 더 보수적이며, 호스트 load average가 이미 높을 때도 물러나므로 여러 Vitest 실행이 동시에 돌 때 기본적으로 피해를 덜 줍니다.
  - 기본 Vitest config는 테스트 wiring이 바뀔 때 changed-mode 재실행이 정확하게 유지되도록 projects/config 파일을 `forceRerunTriggers`로 표시합니다.
  - 지원되는 호스트에서는 config가 `OPENCLAW_VITEST_FS_MODULE_CACHE`를 계속 활성화합니다. 직접 프로파일링을 위해 명시적 캐시 위치 하나를 쓰고 싶다면 `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path`를 설정하세요.
- 성능 디버그 참고:
  - `pnpm test:perf:imports`는 Vitest import-duration 보고와 import-breakdown 출력을 활성화합니다.
  - `pnpm test:perf:imports:changed`는 동일한 프로파일링 보기를 `origin/main` 이후 변경된 파일로 제한합니다.
  - `pnpm test:perf:profile:main`은 Vitest/Vite 시작 및 transform 오버헤드에 대한 메인 스레드 CPU 프로파일을 기록합니다.
  - `pnpm test:perf:profile:runner`는 파일 병렬화를 비활성화한 unit 스위트에 대해 runner CPU+heap 프로파일을 기록합니다.

### E2E(gateway 스모크)

- 명령: `pnpm test:e2e`
- 구성: `vitest.e2e.config.ts`
- 파일: `src/**/*.e2e.test.ts`, `test/**/*.e2e.test.ts`
- 런타임 기본값:
  - 나머지 저장소와 동일하게 Vitest `threads`와 `isolate: false`를 사용합니다.
  - 적응형 worker를 사용합니다(CI: 최대 2, 로컬: 기본 1).
  - 콘솔 I/O 오버헤드를 줄이기 위해 기본적으로 silent 모드로 실행됩니다.
- 유용한 재정의:
  - worker 수를 강제하려면 `OPENCLAW_E2E_WORKERS=<n>`(최대 16)
  - 자세한 콘솔 출력을 다시 활성화하려면 `OPENCLAW_E2E_VERBOSE=1`
- 범위:
  - 다중 인스턴스 gateway end-to-end 동작
  - WebSocket/HTTP 표면, node 페어링, 더 무거운 네트워킹
- 기대 사항:
  - CI에서 실행됨(파이프라인에서 활성화된 경우)
  - 실제 키 필요 없음
  - unit 테스트보다 움직이는 부분이 많음(더 느릴 수 있음)

### E2E: OpenShell 백엔드 스모크

- 명령: `pnpm test:e2e:openshell`
- 파일: `test/openshell-sandbox.e2e.test.ts`
- 범위:
  - Docker를 통해 host에서 격리된 OpenShell gateway를 시작
  - 임시 로컬 Dockerfile로부터 샌드박스를 생성
  - 실제 `sandbox ssh-config` + SSH exec를 통해 OpenClaw의 OpenShell 백엔드를 수행
  - 샌드박스 fs bridge를 통해 remote-canonical 파일시스템 동작 검증
- 기대 사항:
  - opt-in 전용이며 기본 `pnpm test:e2e` 실행에는 포함되지 않음
  - 로컬 `openshell` CLI와 동작하는 Docker daemon 필요
  - 격리된 `HOME` / `XDG_CONFIG_HOME`을 사용한 뒤 테스트 gateway와 샌드박스를 제거
- 유용한 재정의:
  - 더 넓은 e2e 스위트를 수동 실행할 때 테스트를 활성화하려면 `OPENCLAW_E2E_OPENSHELL=1`
  - 기본이 아닌 CLI 바이너리 또는 래퍼 스크립트를 가리키려면 `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell`

### Live(실제 공급자 + 실제 모델)

- 명령: `pnpm test:live`
- 구성: `vitest.live.config.ts`
- 파일: `src/**/*.live.test.ts`
- 기본값: `pnpm test:live`가 **활성화**함(`OPENCLAW_LIVE_TEST=1` 설정)
- 범위:
  - “이 공급자/모델이 _오늘_ 실제 자격 증명으로 실제로 동작하는가?”
  - 공급자 형식 변경, tool-calling 특이점, 인증 문제, rate limit 동작 포착
- 기대 사항:
  - 설계상 CI에서 안정적이지 않음(실제 네트워크, 실제 공급자 정책, 할당량, 장애)
  - 비용이 들고 rate limit을 사용함
  - “전부”보다 범위를 좁힌 부분집합 실행을 선호
- Live 실행은 누락된 API 키를 가져오기 위해 `~/.profile`을 source합니다.
- 기본적으로 live 실행은 여전히 `HOME`을 격리하고 config/auth 자료를 임시 테스트 홈으로 복사하여 unit 픽스처가 실제 `~/.openclaw`를 수정하지 못하게 합니다.
- live 테스트가 실제 홈 디렉터리를 사용해야 하는 경우에만 `OPENCLAW_LIVE_USE_REAL_HOME=1`을 설정하세요.
- `pnpm test:live`는 이제 더 조용한 모드가 기본입니다. `[live] ...` 진행 출력은 유지하지만 추가 `~/.profile` 알림과 gateway bootstrap 로그/Bonjour chatter는 숨깁니다. 전체 시작 로그가 필요하면 `OPENCLAW_LIVE_TEST_QUIET=0`을 설정하세요.
- API 키 로테이션(공급자별): 쉼표/세미콜론 형식의 `*_API_KEYS` 또는 `*_API_KEY_1`, `*_API_KEY_2`를 설정하세요(예: `OPENAI_API_KEYS`, `ANTHROPIC_API_KEYS`, `GEMINI_API_KEYS`). 또는 live 전용 재정의로 `OPENCLAW_LIVE_*_KEY`를 사용할 수 있습니다. 테스트는 rate limit 응답 시 재시도합니다.
- 진행/heartbeat 출력:
  - Live 스위트는 이제 stderr에 진행 라인을 출력하므로 Vitest 콘솔 캡처가 조용해도 긴 공급자 호출이 실제로 진행 중임을 볼 수 있습니다.
  - `vitest.live.config.ts`는 Vitest 콘솔 가로채기를 비활성화하므로 공급자/gateway 진행 라인이 live 실행 중 즉시 스트리밍됩니다.
  - 직접 모델 heartbeat는 `OPENCLAW_LIVE_HEARTBEAT_MS`로 조정하세요.
  - gateway/probe heartbeat는 `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS`로 조정하세요.

## 어떤 스위트를 실행해야 하나요?

다음 결정표를 사용하세요.

- 로직/테스트 수정: `pnpm test` 실행(많이 바꿨다면 `pnpm test:coverage`도)
- gateway 네트워킹 / WS 프로토콜 / 페어링 수정: `pnpm test:e2e` 추가
- “내 봇이 죽어 있다” / 공급자별 실패 / tool calling 디버그: 범위를 좁힌 `pnpm test:live` 실행

## Live: Android node capability 스윕

- 테스트: `src/gateway/android-node.capabilities.live.test.ts`
- 스크립트: `pnpm android:test:integration`
- 목표: 연결된 Android node가 현재 광고하는 **모든 명령**을 호출하고 명령 계약 동작을 검증
- 범위:
  - 사전 조건이 있는 수동 설정(스위트는 앱을 설치/실행/페어링하지 않음)
  - 선택한 Android node에 대한 명령별 gateway `node.invoke` 검증
- 필요한 사전 설정:
  - Android 앱이 이미 gateway에 연결되고 페어링되어 있어야 함
  - 앱을 전경 상태로 유지
  - 통과를 기대하는 capability에 필요한 권한/캡처 동의 부여
- 선택적 대상 재정의:
  - `OPENCLAW_ANDROID_NODE_ID` 또는 `OPENCLAW_ANDROID_NODE_NAME`
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`
- 전체 Android 설정 세부 정보: [Android App](/ko/platforms/android)

## Live: 모델 스모크(프로필 키)

실패를 분리할 수 있도록 live 테스트는 두 계층으로 나뉩니다.

- “직접 모델”은 주어진 키로 공급자/모델이 최소한 응답할 수 있는지를 알려줍니다.
- “Gateway 스모크”는 해당 모델에 대해 전체 gateway+agent 파이프라인이 동작하는지를 알려줍니다(세션, 기록, 도구, 샌드박스 정책 등).

### 계층 1: 직접 모델 completion(gateway 없음)

- 테스트: `src/agents/models.profiles.live.test.ts`
- 목표:
  - 발견된 모델 열거
  - `getApiKeyForModel`을 사용해 자격 증명이 있는 모델 선택
  - 모델별로 작은 completion 실행(필요 시 대상 회귀 포함)
- 활성화 방법:
  - `pnpm test:live`(또는 Vitest를 직접 호출할 경우 `OPENCLAW_LIVE_TEST=1`)
- 이 스위트를 실제로 실행하려면 `OPENCLAW_LIVE_MODELS=modern`(또는 modern의 별칭인 `all`)을 설정하세요. 그렇지 않으면 `pnpm test:live`를 gateway 스모크에 집중시키기 위해 건너뜁니다.
- 모델 선택 방법:
  - 최신 허용 목록(Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)을 실행하려면 `OPENCLAW_LIVE_MODELS=modern`
  - `OPENCLAW_LIVE_MODELS=all`은 최신 허용 목록의 별칭
  - 또는 `OPENCLAW_LIVE_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,..."`(쉼표 허용 목록)
- 공급자 선택 방법:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity"`(쉼표 허용 목록)
- 키 출처:
  - 기본값: 프로필 저장소와 환경 변수 폴백
  - **프로필 저장소만** 강제하려면 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`
- 존재 이유:
  - “공급자 API가 깨졌거나 키가 잘못되었는가”와 “gateway agent 파이프라인이 깨졌는가”를 분리
  - 작고 격리된 회귀를 포함(예: OpenAI Responses/Codex Responses reasoning replay + tool-call 흐름)

### 계층 2: Gateway + dev agent 스모크(실제 "@openclaw"가 하는 일)

- 테스트: `src/gateway/gateway-models.profiles.live.test.ts`
- 목표:
  - 프로세스 내 gateway 실행
  - `agent:dev:*` 세션 생성/패치(실행별 모델 재정의)
  - 키가 있는 모델들을 순회하며 다음을 검증:
    - “의미 있는” 응답(도구 없음)
    - 실제 도구 호출이 동작함(read probe)
    - 선택적 추가 도구 probe(exec+read probe)
    - OpenAI 회귀 경로(tool-call-only → follow-up)가 계속 동작
- Probe 세부 사항(실패를 빠르게 설명할 수 있도록):
  - `read` probe: 테스트가 작업공간에 nonce 파일을 쓰고, agent에게 그것을 `read`하고 nonce를 다시 말하도록 요청합니다.
  - `exec+read` probe: 테스트가 agent에게 `exec`로 nonce를 임시 파일에 쓰고, 다시 `read`하도록 요청합니다.
  - image probe: 테스트가 생성된 PNG(고양이 + 무작위 코드)를 첨부하고, 모델이 `cat <CODE>`를 반환할 것으로 기대합니다.
  - 구현 참조: `src/gateway/gateway-models.profiles.live.test.ts` 및 `src/gateway/live-image-probe.ts`
- 활성화 방법:
  - `pnpm test:live`(또는 Vitest를 직접 호출할 경우 `OPENCLAW_LIVE_TEST=1`)
- 모델 선택 방법:
  - 기본값: 최신 허용 목록(Opus/Sonnet 4.6+, GPT-5.x + Codex, Gemini 3, GLM 4.7, MiniMax M2.7, Grok 4)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all`은 최신 허용 목록의 별칭
  - 또는 범위를 좁히려면 `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"`(또는 쉼표 목록) 설정
- 공급자 선택 방법(“OpenRouter 전부” 피하기):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,openai,anthropic,zai,minimax"`(쉼표 허용 목록)
- 이 live 테스트에서는 tool + image probe가 항상 켜져 있습니다:
  - `read` probe + `exec+read` probe(tool stress)
  - 모델이 이미지 입력 지원을 광고하면 image probe 실행
  - 흐름(상위 수준):
    - 테스트가 “CAT” + 무작위 코드를 가진 작은 PNG를 생성(`src/gateway/live-image-probe.ts`)
    - 이를 `agent`의 `attachments: [{ mimeType: "image/png", content: "<base64>" }]`로 전송
    - Gateway가 첨부 파일을 `images[]`로 파싱(`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - Embedded agent가 멀티모달 사용자 메시지를 모델에 전달
    - 검증: 응답에 `cat` + 코드가 포함됨(OCR 허용 오차: 작은 실수 허용)

팁: 자신의 머신에서 무엇을 테스트할 수 있는지(그리고 정확한 `provider/model` id)를 확인하려면 다음을 실행하세요.

```bash
openclaw models list
openclaw models list --json
```

## Live: ACP bind 스모크(`/acp spawn ... --bind here`)

- 테스트: `src/gateway/gateway-acp-bind.live.test.ts`
- 목표: live ACP agent와 함께 실제 ACP 대화 바인드 흐름을 검증
  - `/acp spawn <agent> --bind here` 전송
  - synthetic message-channel 대화를 그 자리에서 바인드
  - 같은 대화에서 일반 후속 메시지 전송
  - 후속 메시지가 바인드된 ACP 세션 transcript에 도착하는지 검증
- 활성화:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- 기본값:
  - ACP agent: `claude`
  - Synthetic channel: Slack DM 스타일 대화 컨텍스트
  - ACP 백엔드: `acpx`
- 재정의:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
- 참고:
  - 이 레인은 gateway `chat.send` 표면과 admin 전용 synthetic originating-route 필드를 사용하므로, 테스트가 외부 전달을 가장하지 않고 message-channel 컨텍스트를 붙일 수 있습니다.
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND`가 설정되지 않으면, 테스트는 선택한 ACP harness agent에 대해 내장 `acpx` plugin의 내장 agent registry를 사용합니다.

예시:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

Docker 레시피:

```bash
pnpm test:docker:live-acp-bind
```

Docker 참고:

- Docker 러너는 `scripts/test-live-acp-bind-docker.sh`에 있습니다.
- `~/.profile`을 source하고, 일치하는 CLI auth 자료를 컨테이너에 준비하고, 쓰기 가능한 npm prefix에 `acpx`를 설치한 뒤, 요청한 live CLI(`@anthropic-ai/claude-code` 또는 `@openai/codex`)가 없으면 설치합니다.
- Docker 내부에서 러너는 `OPENCLAW_LIVE_ACP_BIND_ACPX_COMMAND=$HOME/.npm-global/bin/acpx`를 설정하여, source된 프로필의 공급자 환경 변수를 acpx가 자식 harness CLI에 사용할 수 있게 유지합니다.

### 권장 live 레시피

좁고 명시적인 허용 목록이 가장 빠르고 가장 덜 불안정합니다.

- 단일 모델, 직접(gateway 없음):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.4" pnpm test:live src/agents/models.profiles.live.test.ts`

- 단일 모델, gateway 스모크:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 여러 공급자에 걸친 tool calling:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3-flash-preview,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google 집중(Gemini API 키 + Antigravity):
  - Gemini(API 키): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity(OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

참고:

- `google/...`는 Gemini API(API 키)를 사용합니다.
- `google-antigravity/...`는 Antigravity OAuth 브리지(Cloud Code Assist 스타일 agent 엔드포인트)를 사용합니다.

## Live: 모델 매트릭스(무엇을 다루는지)

고정된 “CI 모델 목록”은 없습니다(live는 opt-in). 하지만 키가 있는 개발 머신에서 정기적으로 다루기를 권장하는 모델은 다음과 같습니다.

### 최신 스모크 세트(tool calling + image)

이것이 우리가 계속 동작하길 기대하는 “일반 모델” 실행입니다.

- OpenAI(non-Codex): `openai/gpt-5.4`(선택 사항: `openai/gpt-5.4-mini`)
- OpenAI Codex: `openai-codex/gpt-5.4`
- Anthropic: `anthropic/claude-opus-4-6`(또는 `anthropic/claude-sonnet-4-6`)
- Google(Gemini API): `google/gemini-3.1-pro-preview` 및 `google/gemini-3-flash-preview`(구형 Gemini 2.x 모델은 피하세요)
- Google(Antigravity): `google-antigravity/claude-opus-4-6-thinking` 및 `google-antigravity/gemini-3-flash`
- Z.AI(GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

도구 + 이미지로 gateway 스모크 실행:
`OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.4,openai-codex/gpt-5.4,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/MiniMax-M2.7" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### 기준선: tool calling(Read + 선택적 Exec)

공급자 패밀리별로 최소 하나는 선택하세요.

- OpenAI: `openai/gpt-5.4`(또는 `openai/gpt-5.4-mini`)
- Anthropic: `anthropic/claude-opus-4-6`(또는 `anthropic/claude-sonnet-4-6`)
- Google: `google/gemini-3-flash-preview`(또는 `google/gemini-3.1-pro-preview`)
- Z.AI(GLM): `zai/glm-4.7`
- MiniMax: `minimax/MiniMax-M2.7`

선택적 추가 커버리지(있으면 좋음):

- xAI: `xai/grok-4`(또는 사용 가능한 최신 버전)
- Mistral: `mistral/`…(활성화된 “tools” 지원 모델 하나 선택)
- Cerebras: `cerebras/`…(접근 권한이 있는 경우)
- LM Studio: `lmstudio/`…(로컬; tool calling은 API 모드에 따라 다름)

### Vision: 이미지 전송(첨부 파일 → 멀티모달 메시지)

image probe를 수행하려면 `OPENCLAW_LIVE_GATEWAY_MODELS`에 이미지 가능 모델 하나 이상(Claude/Gemini/OpenAI 비전 가능 변형 등)을 포함하세요.

### 집계자 / 대체 gateway

키가 활성화되어 있다면 다음을 통한 테스트도 지원합니다.

- OpenRouter: `openrouter/...`(수백 개 모델; tool+image 가능 후보를 찾으려면 `openclaw models scan` 사용)
- OpenCode: Zen용 `opencode/...`, Go용 `opencode-go/...`(인증: `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

live 매트릭스에 포함할 수 있는 더 많은 공급자(자격 증명/구성이 있는 경우):

- 내장: `openai`, `openai-codex`, `anthropic`, `google`, `google-vertex`, `google-antigravity`, `zai`, `openrouter`, `opencode`, `opencode-go`, `xai`, `groq`, `cerebras`, `mistral`, `github-copilot`
- `models.providers`를 통한 사용자 지정 엔드포인트: `minimax`(cloud/API), 그리고 OpenAI/Anthropic 호환 프록시(LM Studio, vLLM, LiteLLM 등)

팁: 문서에 “모든 모델”을 하드코딩하려 하지 마세요. 권위 있는 목록은 사용자의 머신에서 `discoverModels(...)`가 반환하는 것 + 사용 가능한 키입니다.

## 자격 증명(절대 커밋 금지)

live 테스트는 CLI와 동일한 방식으로 자격 증명을 찾습니다. 실용적으로는 다음을 뜻합니다.

- CLI가 동작하면 live 테스트도 같은 키를 찾아야 합니다.
- live 테스트가 “자격 증명 없음”이라고 하면, `openclaw models list` / 모델 선택을 디버깅하듯 똑같이 디버깅하세요.

- 에이전트별 인증 프로필: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`(live 테스트에서 말하는 “프로필 키”가 이것입니다)
- 구성: `~/.openclaw/openclaw.json`(또는 `OPENCLAW_CONFIG_PATH`)
- 레거시 상태 디렉터리: `~/.openclaw/credentials/`(존재하면 준비된 live 홈으로 복사되지만, 기본 프로필 키 저장소는 아님)
- 로컬 live 실행은 기본적으로 활성 config, 에이전트별 `auth-profiles.json` 파일, 레거시 `credentials/`, 지원되는 외부 CLI auth 디렉터리를 임시 테스트 홈으로 복사합니다. 이 준비된 config에서는 `agents.*.workspace` / `agentDir` 경로 재정의가 제거되므로 probe가 실제 host 작업공간을 건드리지 않습니다.

환경 변수 키(예: `~/.profile`에 export된 것)를 사용하려면, `source ~/.profile` 후 로컬 테스트를 실행하거나 아래의 Docker 러너를 사용하세요(이들은 `~/.profile`을 컨테이너에 마운트할 수 있습니다).

## Deepgram live(오디오 전사)

- 테스트: `src/media-understanding/providers/deepgram/audio.live.test.ts`
- 활성화: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## BytePlus coding plan live

- 테스트: `src/agents/byteplus.live.test.ts`
- 활성화: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live src/agents/byteplus.live.test.ts`
- 선택적 모델 재정의: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## ComfyUI workflow media live

- 테스트: `extensions/comfy/comfy.live.test.ts`
- 활성화: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- 범위:
  - 번들된 comfy 이미지, 비디오, `music_generate` 경로 수행
  - `models.providers.comfy.<capability>`가 구성되지 않은 capability는 각각 건너뜀
  - comfy workflow 제출, polling, downloads, plugin 등록을 변경한 뒤 유용함

## 이미지 생성 live

- 테스트: `src/image-generation/runtime.live.test.ts`
- 명령: `pnpm test:live src/image-generation/runtime.live.test.ts`
- 범위:
  - 등록된 모든 이미지 생성 공급자 plugin을 열거
  - probe 전에 로그인 셸(`~/.profile`)에서 누락된 공급자 환경 변수를 로드
  - 기본적으로 저장된 인증 프로필보다 live/env API 키를 우선 사용하므로 `auth-profiles.json`의 오래된 테스트 키가 실제 셸 자격 증명을 가리지 않음
  - 사용 가능한 auth/profile/model이 없는 공급자는 건너뜀
  - 공유 런타임 capability를 통해 기본 이미지 생성 변형 실행:
    - `google:flash-generate`
    - `google:pro-generate`
    - `google:pro-edit`
    - `openai:default-generate`
- 현재 다루는 번들 공급자:
  - `openai`
  - `google`
- 선택적 범위 좁히기:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-1,google/gemini-3.1-flash-image-preview"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit"`
- 선택적 인증 동작:
  - 프로필 저장소 인증을 강제하고 env 전용 재정의를 무시하려면 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`

## 음악 생성 live

- 테스트: `extensions/music-generation-providers.live.test.ts`
- 활성화: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- 범위:
  - 공유 번들 음악 생성 공급자 경로 수행
  - 현재 Google과 MiniMax를 다룸
  - probe 전에 로그인 셸(`~/.profile`)에서 공급자 환경 변수를 로드
  - 사용 가능한 auth/profile/model이 없는 공급자는 건너뜀
- 선택적 범위 좁히기:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.5+"`

## Docker 러너(선택적 "Linux에서도 동작" 검사)

이 Docker 러너는 두 부류로 나뉩니다.

- Live-model 러너: `test:docker:live-models`와 `test:docker:live-gateway`는 저장소 Docker 이미지 안에서 일치하는 프로필 키 live 파일(`src/agents/models.profiles.live.test.ts`와 `src/gateway/gateway-models.profiles.live.test.ts`)만 실행합니다. 로컬 config 디렉터리와 작업공간을 마운트하고(마운트된 경우 `~/.profile`도 source), 이에 대응하는 로컬 엔트리포인트는 `test:live:models-profiles`와 `test:live:gateway-profiles`입니다.
- Docker live 러너는 전체 Docker 스윕을 실용적으로 유지하기 위해 기본적으로 더 작은 스모크 cap을 사용합니다:
  `test:docker:live-models`는 기본값으로 `OPENCLAW_LIVE_MAX_MODELS=12`,
  `test:docker:live-gateway`는 기본값으로 `OPENCLAW_LIVE_GATEWAY_SMOKE=1`,
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`,
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000`,
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000`를 사용합니다. 더 큰 exhaustive scan을 명시적으로 원할 때는 해당 환경 변수를 재정의하세요.
- `test:docker:all`은 먼저 `test:docker:live-build`를 통해 live Docker 이미지를 한 번 빌드한 뒤, 두 live Docker 레인에서 재사용합니다.
- 컨테이너 스모크 러너: `test:docker:openwebui`, `test:docker:onboard`, `test:docker:gateway-network`, `test:docker:mcp-channels`, `test:docker:plugins`는 하나 이상의 실제 컨테이너를 부팅하고 더 높은 수준의 integration 경로를 검증합니다.

live-model Docker 러너는 필요한 CLI auth 홈만 bind-mount하고(또는 실행이 좁혀지지 않았으면 지원되는 모든 항목), 실행 전에 이를 컨테이너 홈으로 복사하므로 외부 CLI OAuth가 host auth 저장소를 수정하지 않고 토큰을 새로 고칠 수 있습니다.

- 직접 모델: `pnpm test:docker:live-models`(스크립트: `scripts/test-live-models-docker.sh`)
- ACP bind 스모크: `pnpm test:docker:live-acp-bind`(스크립트: `scripts/test-live-acp-bind-docker.sh`)
- Gateway + dev agent: `pnpm test:docker:live-gateway`(스크립트: `scripts/test-live-gateway-models-docker.sh`)
- Open WebUI live 스모크: `pnpm test:docker:openwebui`(스크립트: `scripts/e2e/openwebui-docker.sh`)
- Onboarding wizard(TTY, 전체 스캐폴딩): `pnpm test:docker:onboard`(스크립트: `scripts/e2e/onboard-docker.sh`)
- Gateway networking(두 컨테이너, WS auth + health): `pnpm test:docker:gateway-network`(스크립트: `scripts/e2e/gateway-network-docker.sh`)
- MCP 채널 bridge(seeded Gateway + stdio bridge + raw Claude notification-frame 스모크): `pnpm test:docker:mcp-channels`(스크립트: `scripts/e2e/mcp-channels-docker.sh`)
- Plugins(설치 스모크 + `/plugin` 별칭 + Claude 번들 재시작 시맨틱): `pnpm test:docker:plugins`(스크립트: `scripts/e2e/plugins-docker.sh`)

live-model Docker 러너는 현재 체크아웃도 읽기 전용으로 bind-mount하고,
이를 컨테이너 내부의 임시 workdir에 준비합니다. 이렇게 하면 런타임
이미지는 슬림하게 유지하면서도 정확히 로컬 소스/config에 대해 Vitest를 실행할 수 있습니다.
준비 단계는 `.pnpm-store`, `.worktrees`, `__openclaw_vitest__`, 앱 로컬 `.build` 또는
Gradle 출력 디렉터리처럼 큰 로컬 전용 캐시와 앱 빌드 출력을 건너뛰므로,
Docker live 실행이 머신 전용 아티팩트를 복사하느라 몇 분씩 쓰지 않게 됩니다.
또한 `OPENCLAW_SKIP_CHANNELS=1`을 설정하므로 gateway live probe가 컨테이너 안에서
실제 Telegram/Discord 등의 채널 worker를 시작하지 않습니다.
`test:docker:live-models`는 여전히 `pnpm test:live`를 실행하므로,
해당 Docker 레인에서 gateway live 커버리지를 좁히거나 제외해야 할 때는
`OPENCLAW_LIVE_GATEWAY_*`도 함께 전달하세요.
`test:docker:openwebui`는 더 높은 수준의 호환성 스모크입니다. 이는
OpenAI 호환 HTTP 엔드포인트가 활성화된 OpenClaw gateway 컨테이너를 시작하고,
그 gateway를 대상으로 고정된 Open WebUI 컨테이너를 시작한 다음, Open WebUI를 통해 로그인하고,
`/api/models`가 `openclaw/default`를 노출하는지 확인한 뒤,
Open WebUI의 `/api/chat/completions` 프록시를 통해 실제 채팅 요청을 보냅니다.
첫 실행은 Docker가 Open WebUI 이미지를 풀해야 하거나
Open WebUI가 자체 콜드 스타트 설정을 완료해야 할 수 있으므로 눈에 띄게 느릴 수 있습니다.
이 레인은 사용 가능한 live 모델 키를 기대하며, Dockerized 실행에서 이를 제공하는
주요 방법은 `OPENCLAW_PROFILE_FILE`(기본값 `~/.profile`)입니다.
성공적인 실행은 `{ "ok": true, "model":
"openclaw/default", ... }` 같은 작은 JSON 페이로드를 출력합니다.
`test:docker:mcp-channels`는 의도적으로 결정론적이며 실제
Telegram, Discord, iMessage 계정이 필요하지 않습니다. seeded Gateway
컨테이너를 부팅하고, 두 번째 컨테이너에서 `openclaw mcp serve`를 실행한 뒤,
실제 stdio MCP bridge를 통해 라우팅된 대화 탐색, transcript 읽기, 첨부 파일 메타데이터,
live event queue 동작, outbound send routing, Claude 스타일 채널 +
권한 알림을 검증합니다. 알림 검사는 원시 stdio MCP 프레임을 직접 검사하므로,
이 스모크는 특정 client SDK가 우연히 surface하는 것만이 아니라
bridge가 실제로 내보내는 내용을 검증합니다.

수동 ACP plain-language 스레드 스모크(CI 아님):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- 이 스크립트는 회귀/디버그 워크플로를 위해 유지하세요. ACP 스레드 라우팅 검증에 다시 필요할 수 있으므로 삭제하지 마세요.

유용한 환경 변수:

- `OPENCLAW_CONFIG_DIR=...`(기본값: `~/.openclaw`)를 `/home/node/.openclaw`에 마운트
- `OPENCLAW_WORKSPACE_DIR=...`(기본값: `~/.openclaw/workspace`)를 `/home/node/.openclaw/workspace`에 마운트
- `OPENCLAW_PROFILE_FILE=...`(기본값: `~/.profile`)를 `/home/node/.profile`에 마운트하고 테스트 실행 전에 source
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...`(기본값: `~/.cache/openclaw/docker-cli-tools`)를 Docker 내부 캐시된 CLI 설치용 `/home/node/.npm-global`에 마운트
- `$HOME` 아래의 외부 CLI auth 디렉터리/파일은 `/host-auth...` 아래 읽기 전용으로 마운트된 뒤 테스트 시작 전에 `/home/node/...`로 복사됨
  - 기본 디렉터리: `.minimax`
  - 기본 파일: `~/.codex/auth.json`, `~/.codex/config.toml`, `.claude.json`, `~/.claude/.credentials.json`, `~/.claude/settings.json`, `~/.claude/settings.local.json`
  - 범위를 좁힌 공급자 실행은 `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS`에서 추론한 필요한 디렉터리/파일만 마운트
  - 수동 재정의: `OPENCLAW_DOCKER_AUTH_DIRS=all`, `OPENCLAW_DOCKER_AUTH_DIRS=none`, 또는 `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` 같은 쉼표 목록
- 실행 범위를 좁히려면 `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...`
- 컨테이너 내부 공급자 필터링에는 `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...`
- 자격 증명이 프로필 저장소에서만 오도록 보장하려면 `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1`
- Open WebUI 스모크용 gateway 노출 모델을 선택하려면 `OPENCLAW_OPENWEBUI_MODEL=...`
- Open WebUI 스모크에서 nonce-check 프롬프트를 재정의하려면 `OPENCLAW_OPENWEBUI_PROMPT=...`
- 고정된 Open WebUI 이미지 태그를 재정의하려면 `OPENWEBUI_IMAGE=...`

## 문서 sanity

문서를 수정한 뒤 문서 검사를 실행하세요: `pnpm check:docs`.
페이지 내 heading 검사도 필요하면 전체 Mintlify anchor 검증을 실행하세요: `pnpm docs:check-links:anchors`.

## 오프라인 회귀(CI 안전)

다음은 실제 공급자 없이도 “실제 파이프라인”을 검증하는 회귀입니다.

- Gateway tool calling(mock OpenAI, 실제 gateway + agent loop): `src/gateway/gateway.test.ts`(케이스: "runs a mock OpenAI tool call end-to-end via gateway agent loop")
- Gateway wizard(WS `wizard.start`/`wizard.next`, config + auth 쓰기 강제): `src/gateway/gateway.test.ts`(케이스: "runs wizard over ws and writes auth token config")

## 에이전트 신뢰성 평가(Skills)

이미 “에이전트 신뢰성 평가”처럼 동작하는 몇 가지 CI 안전 테스트가 있습니다.

- 실제 gateway + agent loop를 통한 mock tool-calling(`src/gateway/gateway.test.ts`)
- 세션 wiring과 config 효과를 검증하는 end-to-end wizard 흐름(`src/gateway/gateway.test.ts`)

Skills에 대해 아직 부족한 것([Skills](/ko/tools/skills) 참고):

- **Decisioning:** 프롬프트에 skills가 나열될 때 agent가 올바른 skill을 선택하는가(또는 무관한 skill을 피하는가)?
- **Compliance:** 사용 전에 agent가 `SKILL.md`를 읽고 필요한 단계/인수를 따르는가?
- **Workflow contracts:** 도구 순서, 세션 기록 이월, 샌드박스 경계를 단언하는 multi-turn 시나리오

향후 평가는 우선 결정론적으로 유지되어야 합니다.

- mock provider를 사용하는 시나리오 러너로 tool call + 순서, skill 파일 읽기, 세션 wiring을 단언
- skill 중심 시나리오의 소규모 스위트(사용 vs 회피, 게이팅, 프롬프트 인젝션)
- CI 안전 스위트가 준비된 뒤에만 선택적으로 env 게이트된 live 평가

## 계약 테스트(plugin 및 channel 형태)

계약 테스트는 등록된 모든 plugin과 channel이 해당 인터페이스 계약을 준수하는지 검증합니다. 발견된 모든 plugin을 순회하며 형태 및 동작 단언 스위트를 실행합니다. 기본 `pnpm test` unit 레인은 이러한 공유 seam 및 스모크 파일을 의도적으로 건너뛰므로, 공유 channel 또는 provider 표면을 수정할 때는 계약 명령을 명시적으로 실행하세요.

### 명령

- 모든 계약: `pnpm test:contracts`
- channel 계약만: `pnpm test:contracts:channels`
- provider 계약만: `pnpm test:contracts:plugins`

### Channel 계약

`src/channels/plugins/contracts/*.contract.test.ts`에 있습니다.

- **plugin** - 기본 plugin 형태(id, name, capabilities)
- **setup** - setup wizard 계약
- **session-binding** - 세션 바인딩 동작
- **outbound-payload** - 메시지 페이로드 구조
- **inbound** - 인바운드 메시지 처리
- **actions** - channel action 핸들러
- **threading** - 스레드 ID 처리
- **directory** - 디렉터리/roster API
- **group-policy** - 그룹 정책 강제

### Provider status 계약

`src/plugins/contracts/*.contract.test.ts`에 있습니다.

- **status** - channel 상태 probe
- **registry** - plugin registry 형태

### Provider 계약

`src/plugins/contracts/*.contract.test.ts`에 있습니다.

- **auth** - 인증 흐름 계약
- **auth-choice** - 인증 선택/선정
- **catalog** - 모델 카탈로그 API
- **discovery** - plugin 탐색
- **loader** - plugin 로딩
- **runtime** - provider 런타임
- **shape** - plugin 형태/인터페이스
- **wizard** - setup wizard

### 실행 시점

- plugin-sdk export 또는 subpath를 변경한 뒤
- channel 또는 provider plugin을 추가하거나 수정한 뒤
- plugin 등록 또는 탐색을 리팩터링한 뒤

계약 테스트는 CI에서 실행되며 실제 API 키가 필요하지 않습니다.

## 회귀 추가(가이드)

live에서 발견된 provider/model 문제를 수정할 때:

- 가능하면 CI 안전 회귀를 추가하세요(mock/stub provider 또는 정확한 request-shape transformation 캡처)
- 본질적으로 live 전용이라면(rate limit, auth 정책), live 테스트를 좁고 env 변수 기반 opt-in으로 유지하세요
- 버그를 잡을 수 있는 가장 작은 계층을 타기팅하는 것이 좋습니다.
  - provider request conversion/replay 버그 → 직접 모델 테스트
  - gateway session/history/tool 파이프라인 버그 → gateway live 스모크 또는 CI 안전 gateway mock 테스트
- SecretRef traversal 가드레일:
  - `src/secrets/exec-secret-ref-id-parity.test.ts`는 registry 메타데이터(`listSecretTargetRegistryEntries()`)에서 SecretRef 클래스별 샘플 타깃 하나를 도출한 뒤, traversal-segment exec id가 거부되는지 단언합니다.
  - `src/secrets/target-registry-data.ts`에 새 `includeInPlan` SecretRef 타깃 패밀리를 추가한다면, 해당 테스트의 `classifyTargetClass`를 업데이트하세요. 이 테스트는 분류되지 않은 타깃 id에서 의도적으로 실패하므로 새 클래스가 조용히 건너뛰어질 수 없습니다.
