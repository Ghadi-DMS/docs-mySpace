---
read_when:
    - 기본 OpenClaw 플러그인을 빌드하거나 디버깅할 때
    - 플러그인 capability 모델 또는 소유권 경계를 이해할 때
    - 플러그인 로드 파이프라인 또는 레지스트리 작업 중일 때
    - provider 런타임 hook 또는 채널 플러그인을 구현할 때
sidebarTitle: Internals
summary: '플러그인 내부: capability 모델, 소유권, 계약, 로드 파이프라인, 런타임 헬퍼'
title: 플러그인 내부 구조
x-i18n:
    generated_at: "2026-04-06T03:12:39Z"
    model: gpt-5.4
    provider: openai
    source_hash: d39158455701dedfb75f6c20b8c69fd36ed9841f1d92bed1915f448df57fd47b
    source_path: plugins/architecture.md
    workflow: 15
---

# 플러그인 내부 구조

<Info>
  이것은 **심층 아키텍처 참조 문서**입니다. 실용적인 가이드는 다음을 참고하세요.
  - [플러그인 설치 및 사용](/ko/tools/plugin) — 사용자 가이드
  - [시작하기](/ko/plugins/building-plugins) — 첫 번째 플러그인 튜토리얼
  - [채널 플러그인](/ko/plugins/sdk-channel-plugins) — 메시징 채널 만들기
  - [Provider 플러그인](/ko/plugins/sdk-provider-plugins) — 모델 provider 만들기
  - [SDK 개요](/ko/plugins/sdk-overview) — import 맵과 등록 API
</Info>

이 페이지는 OpenClaw 플러그인 시스템의 내부 아키텍처를 설명합니다.

## 공개 capability 모델

Capabilities는 OpenClaw 내부의 공개 **기본 플러그인** 모델입니다. 모든
기본 OpenClaw 플러그인은 하나 이상의 capability 유형에 대해 등록됩니다.

| Capability             | 등록 메서드                                    | 예시 플러그인                        |
| ---------------------- | ---------------------------------------------- | ------------------------------------ |
| 텍스트 추론            | `api.registerProvider(...)`                    | `openai`, `anthropic`                |
| 음성                   | `api.registerSpeechProvider(...)`              | `elevenlabs`, `microsoft`            |
| 실시간 전사            | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| 실시간 음성            | `api.registerRealtimeVoiceProvider(...)`       | `openai`                             |
| 미디어 이해            | `api.registerMediaUnderstandingProvider(...)`  | `openai`, `google`                   |
| 이미지 생성            | `api.registerImageGenerationProvider(...)`     | `openai`, `google`, `fal`, `minimax` |
| 음악 생성              | `api.registerMusicGenerationProvider(...)`     | `google`, `minimax`                  |
| 비디오 생성            | `api.registerVideoGenerationProvider(...)`     | `qwen`                               |
| 웹 가져오기            | `api.registerWebFetchProvider(...)`            | `firecrawl`                          |
| 웹 검색                | `api.registerWebSearchProvider(...)`           | `google`                             |
| 채널 / 메시징          | `api.registerChannel(...)`                     | `msteams`, `matrix`                  |

capability를 하나도 등록하지 않지만 hooks, tools 또는
services를 제공하는 플러그인은 **레거시 hook-only** 플러그인입니다. 이 패턴도 여전히 완전히 지원됩니다.

### 외부 호환성 입장

capability 모델은 이미 core에 반영되어 있으며 현재 번들/기본 플러그인에서
사용되고 있습니다. 하지만 외부 플러그인 호환성은 "export되었으니 곧 고정된 계약이다"보다 더 엄격한 기준이 필요합니다.

현재 지침:

- **기존 외부 플러그인:** hook 기반 통합이 계속 작동하도록 유지하며,
  이것을 호환성 기준선으로 간주합니다
- **새 번들/기본 플러그인:** vendor별 직접 접근 또는 새로운 hook-only 설계보다
  명시적 capability 등록을 우선합니다
- **capability 등록을 도입하는 외부 플러그인:** 허용되지만, 문서에서 계약이 안정적이라고 명시적으로 표시하지 않는 한
  capability별 helper surface는 진화 중인 것으로 간주하세요

실용적인 규칙:

- capability 등록 API가 의도된 방향입니다
- 전환 기간 동안 외부 플러그인에 대해 레거시 hooks가 가장 안전한 무중단 경로입니다
- export된 helper subpath가 모두 같은 성격은 아닙니다. 우연히 노출된 helper export가 아니라
  문서화된 좁은 계약을 우선하세요

### 플러그인 형태

OpenClaw는 로드된 각 플러그인을 정적 메타데이터가 아니라 실제 등록 동작에 따라 형태로 분류합니다.

- **plain-capability** -- 정확히 하나의 capability 유형만 등록합니다(예: `mistral` 같은
  provider 전용 플러그인)
- **hybrid-capability** -- 여러 capability 유형을 등록합니다(예:
  `openai`는 텍스트 추론, 음성, 미디어 이해, 이미지 생성을 모두 소유함)
- **hook-only** -- hooks(typed 또는 custom)만 등록하고, capabilities,
  tools, commands, services는 등록하지 않음
- **non-capability** -- tools, commands, services, routes는 등록하지만
  capabilities는 등록하지 않음

플러그인의 형태와 capability 구성을 확인하려면 `openclaw plugins inspect <id>`를 사용하세요.
자세한 내용은 [CLI reference](/cli/plugins#inspect)를 참고하세요.

### 레거시 hooks

`before_agent_start` hook은 hook-only 플러그인을 위한 호환 경로로 계속 지원됩니다. 실제 레거시 플러그인들이 여전히 이에 의존합니다.

방향성:

- 계속 작동하게 유지
- 레거시로 문서화
- 모델/provider 재정의 작업에는 `before_model_resolve`를 우선 사용
- 프롬프트 변경 작업에는 `before_prompt_build`를 우선 사용
- 실제 사용량이 줄고 fixture 커버리지가 마이그레이션 안전성을 입증한 뒤에만 제거

### 호환성 신호

`openclaw doctor` 또는 `openclaw plugins inspect <id>`를 실행하면
다음 레이블 중 하나가 표시될 수 있습니다.

| 신호                       | 의미                                                      |
| -------------------------- | --------------------------------------------------------- |
| **config valid**           | Config가 정상적으로 파싱되고 플러그인이 해결됨            |
| **compatibility advisory** | 플러그인이 지원되지만 오래된 패턴을 사용함(예: `hook-only`) |
| **legacy warning**         | 플러그인이 더 이상 권장되지 않는 `before_agent_start`를 사용함 |
| **hard error**             | Config가 잘못되었거나 플러그인 로드에 실패함              |

`hook-only`와 `before_agent_start`는 오늘날 플러그인을 깨뜨리지 않습니다 --
`hook-only`는 권고 수준이고, `before_agent_start`는 경고만 발생시킵니다. 이러한
신호는 `openclaw status --all`과 `openclaw plugins doctor`에도 나타납니다.

## 아키텍처 개요

OpenClaw의 플러그인 시스템은 네 가지 계층으로 구성됩니다.

1. **Manifest + discovery**
   OpenClaw는 구성된 경로, workspace 루트,
   전역 extension 루트, 번들 extension에서 후보 플러그인을 찾습니다.
   discovery는 먼저 기본 `openclaw.plugin.json` manifest와 지원되는 bundle manifest를 읽습니다.
2. **활성화 + 검증**
   Core는 발견된 플러그인이 활성화, 비활성화, 차단 또는
   memory 같은 독점 슬롯에 선택되었는지 판단합니다.
3. **런타임 로딩**
   기본 OpenClaw 플러그인은 jiti를 통해 프로세스 내에서 로드되고
   central registry에 capabilities를 등록합니다. 호환 bundle은
   런타임 코드를 import하지 않고 registry 레코드로 정규화됩니다.
4. **surface 소비**
   OpenClaw의 나머지 부분은 registry를 읽어 tools, channels, provider
   설정, hooks, HTTP routes, CLI commands, services를 노출합니다.

플러그인 CLI의 경우 특히 루트 명령 discovery는 두 단계로 나뉩니다.

- 파싱 시점 메타데이터는 `registerCli(..., { descriptors: [...] })`에서 가져옵니다
- 실제 플러그인 CLI 모듈은 lazy 상태를 유지하다가 첫 호출 시 등록될 수 있습니다

이렇게 하면 OpenClaw가 파싱 전에 루트 명령 이름을 예약할 수 있으면서도
플러그인 소유 CLI 코드를 플러그인 내부에 유지할 수 있습니다.

중요한 설계 경계:

- discovery + config 검증은 플러그인 코드를 실행하지 않고 **manifest/schema metadata**로부터
  작동해야 합니다
- 기본 런타임 동작은 플러그인 모듈의 `register(api)` 경로에서 옵니다

이 분리는 OpenClaw가 전체 런타임이 활성화되기 전에도
config를 검증하고, 누락/비활성화된 플러그인을 설명하고,
UI/schema 힌트를 구성할 수 있게 해줍니다.

### 채널 플러그인과 공유 message tool

채널 플러그인은 일반적인 채팅 작업을 위해 별도의 send/edit/react tool을
등록할 필요가 없습니다. OpenClaw는 core에 하나의 공유 `message` tool을 유지하고,
채널 플러그인은 그 뒤에 있는 채널별 discovery와 실행을 소유합니다.

현재 경계는 다음과 같습니다.

- core는 공유 `message` tool 호스트, 프롬프트 wiring, session/thread
  bookkeeping, 실행 dispatch를 소유합니다
- 채널 플러그인은 범위가 지정된 action discovery, capability discovery, 그리고
  채널별 schema fragment를 소유합니다
- 채널 플러그인은 conversation id가 thread id를 어떻게 인코딩하거나
  부모 conversation을 어떻게 상속하는지 같은 provider별 session conversation grammar를 소유합니다
- 채널 플러그인은 자신의 action adapter를 통해 최종 action을 실행합니다

채널 플러그인용 SDK surface는
`ChannelMessageActionAdapter.describeMessageTool(...)`입니다. 이 통합 discovery
호출을 통해 플러그인은 보이는 actions, capabilities, schema 기여분을
함께 반환할 수 있어 이들이 서로 어긋나지 않게 할 수 있습니다.

Core는 discovery 단계에 런타임 범위를 전달합니다. 중요한 필드는 다음과 같습니다.

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- 신뢰된 인바운드 `requesterSenderId`

이것은 컨텍스트에 민감한 플러그인에 중요합니다. 채널은 core의 `message` tool에
채널별 분기를 하드코딩하지 않고도 활성 계정, 현재 room/thread/message 또는
신뢰된 요청자 identity에 따라 message actions를 숨기거나 노출할 수 있습니다.

이 때문에 embedded-runner 라우팅 변경은 여전히 플러그인 작업입니다. 러너는
현재 채팅/session identity를 플러그인 discovery 경계로 전달하여
공유 `message` tool이 현재 턴에 맞는 채널 소유 surface를 노출하도록 해야 합니다.

채널 소유 실행 helper의 경우 번들 플러그인은 실행 런타임을 자체 extension 모듈 내부에 유지해야 합니다.
Core는 더 이상 `src/agents/tools` 아래에서 Discord,
Slack, Telegram, WhatsApp message-action 런타임을 소유하지 않습니다.
별도의 `plugin-sdk/*-action-runtime` subpath는 공개하지 않으며, 번들
플러그인은 자신의 extension 소유 모듈에서 로컬 런타임 코드를 직접 import해야 합니다.

같은 경계는 일반적인 provider 이름이 붙은 SDK seam에도 적용됩니다. Core는
Slack, Discord, Signal, WhatsApp 또는 유사한 extension용
채널별 convenience barrel을 import해서는 안 됩니다. Core에 어떤 동작이 필요하면
번들 플러그인의 자체 `api.ts` / `runtime-api.ts` barrel을 소비하거나,
필요성을 공유 SDK의 좁은 generic capability로 승격해야 합니다.

특히 polls의 경우 실행 경로가 두 가지입니다.

- `outbound.sendPoll`은 공통 poll 모델에 맞는 채널을 위한 공유 기준선입니다
- `actions.handleAction("poll")`은 채널별 poll 의미론 또는 추가 poll 매개변수에 대해 선호되는 경로입니다

이제 core는 플러그인 poll dispatch가 action을 거절한 뒤에야
공유 poll 파싱을 수행합니다. 따라서 플러그인 소유 poll handler는
generic poll parser에 먼저 막히지 않고 채널별 poll 필드를 받아들일 수 있습니다.

전체 시작 순서는 [Load pipeline](#load-pipeline)을 참고하세요.

## capability 소유권 모델

OpenClaw는 기본 플러그인을 관련 없는 통합들의 묶음이 아니라
**회사** 또는 **기능**의 소유권 경계로 취급합니다.

이는 다음을 의미합니다.

- 회사 플러그인은 일반적으로 그 회사의 OpenClaw 관련 surface를 모두 소유해야 합니다
- 기능 플러그인은 일반적으로 자신이 도입하는 전체 기능 surface를 소유해야 합니다
- 채널은 provider 동작을 임시로 다시 구현하는 대신 공유 core capability를 소비해야 합니다

예시:

- 번들 `openai` 플러그인은 OpenAI 모델 provider 동작과 OpenAI
  speech + realtime-voice + media-understanding + image-generation 동작을 소유합니다
- 번들 `elevenlabs` 플러그인은 ElevenLabs speech 동작을 소유합니다
- 번들 `microsoft` 플러그인은 Microsoft speech 동작을 소유합니다
- 번들 `google` 플러그인은 Google 모델 provider 동작과 Google
  media-understanding + image-generation + web-search 동작을 소유합니다
- 번들 `firecrawl` 플러그인은 Firecrawl web-fetch 동작을 소유합니다
- 번들 `minimax`, `mistral`, `moonshot`, `zai` 플러그인은
  media-understanding backend를 소유합니다
- 번들 `qwen` 플러그인은 Qwen 텍스트 provider 동작과
  media-understanding 및 video-generation 동작을 소유합니다
- `voice-call` 플러그인은 기능 플러그인입니다. 이 플러그인은 call transport, tools,
  CLI, routes, Twilio media-stream bridging을 소유하지만, vendor 플러그인을 직접 import하는 대신
  공유 speech와 realtime-transcription 및 realtime-voice capability를 소비합니다

의도된 최종 상태는 다음과 같습니다.

- OpenAI가 텍스트 모델, speech, 이미지, 미래의 비디오까지 걸쳐 있더라도 하나의 플러그인에 존재
- 다른 vendor도 자신의 surface 영역에 대해 같은 방식 적용 가능
- 채널은 어떤 vendor 플러그인이 provider를 소유하는지 신경 쓰지 않고,
  core가 노출하는 공유 capability 계약을 소비

핵심 구분은 다음과 같습니다.

- **플러그인** = 소유권 경계
- **capability** = 여러 플러그인이 구현하거나 소비할 수 있는 core 계약

따라서 OpenClaw에 video 같은 새 도메인이 추가될 때 첫 번째 질문은
"어떤 provider가 video 처리를 하드코딩해야 하는가?"가 아닙니다.
첫 번째 질문은 "core video capability 계약은 무엇인가?"입니다.
이 계약이 존재하면 vendor 플러그인은 이에 대해 등록할 수 있고,
채널/기능 플러그인은 이를 소비할 수 있습니다.

capability가 아직 없다면 일반적으로 올바른 절차는 다음과 같습니다.

1. core에 누락된 capability를 정의
2. 이를 플러그인 API/런타임을 통해 typed 방식으로 노출
3. 채널/기능을 해당 capability에 연결
4. vendor 플러그인이 구현을 등록하도록 허용

이렇게 하면 소유권을 명확히 유지하면서도
단일 vendor나 일회성 플러그인별 코드 경로에 의존하는 core 동작을 피할 수 있습니다.

### capability 계층화

코드가 어디에 속해야 하는지 판단할 때 다음 사고 모델을 사용하세요.

- **core capability 계층**: 공유 orchestration, policy, fallback, config
  병합 규칙, 전달 의미론, typed 계약
- **vendor 플러그인 계층**: vendor별 API, auth, 모델 카탈로그, speech
  synthesis, image generation, 미래 video backend, usage endpoint
- **채널/기능 플러그인 계층**: 공유 capability를 소비하고 이를 surface에
  제공하는 Slack/Discord/voice-call 등의 통합

예를 들어 TTS는 다음 구조를 따릅니다.

- core는 응답 시점 TTS policy, fallback 순서, prefs, 채널 전달을 소유
- `openai`, `elevenlabs`, `microsoft`는 synthesis 구현을 소유
- `voice-call`은 telephony TTS 런타임 helper를 소비

이와 같은 패턴이 미래 capability에도 선호되어야 합니다.

### 다중 capability 회사 플러그인 예시

회사 플러그인은 외부에서 볼 때 일관된 하나의 덩어리처럼 느껴져야 합니다. OpenClaw가
models, speech, realtime transcription, realtime voice, media
understanding, image generation, video generation, web fetch, web search에 대한 공유 계약을 가진다면,
vendor는 자신의 모든 surface를 한곳에서 소유할 수 있습니다.

```ts
import type { OpenClawPluginDefinition } from "openclaw/plugin-sdk/plugin-entry";
import {
  describeImageWithModel,
  transcribeOpenAiCompatibleAudio,
} from "openclaw/plugin-sdk/media-understanding";

const plugin: OpenClawPluginDefinition = {
  id: "exampleai",
  name: "ExampleAI",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // auth/model catalog/runtime hooks
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // vendor speech config — implement the SpeechProviderPlugin interface directly
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      async describeImage(req) {
        return describeImageWithModel({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
      async transcribeAudio(req) {
        return transcribeOpenAiCompatibleAudio({
          provider: "exampleai",
          model: req.model,
          input: req.input,
        });
      },
    });

    api.registerWebSearchProvider(
      createPluginBackedWebSearchProvider({
        id: "exampleai-search",
        // credential + fetch logic
      }),
    );
  },
};

export default plugin;
```

중요한 것은 정확한 helper 이름이 아닙니다. 중요한 것은 구조입니다.

- 하나의 플러그인이 vendor surface를 소유
- core는 여전히 capability 계약을 소유
- 채널과 기능 플러그인은 vendor 코드가 아니라 `api.runtime.*` helper를 소비
- 계약 테스트는 플러그인이 자신이 소유한다고 주장하는 capability를 등록했는지 검증 가능

### capability 예시: 비디오 이해

OpenClaw는 이미 이미지/오디오/비디오 이해를 하나의 공유
capability로 취급합니다. 같은 소유권 모델이 여기에 적용됩니다.

1. core가 media-understanding 계약을 정의
2. vendor 플러그인이 해당되는 경우 `describeImage`, `transcribeAudio`,
   `describeVideo`를 등록
3. 채널과 기능 플러그인이 vendor 코드에 직접 연결하는 대신 공유 core 동작을 소비

이렇게 하면 특정 provider의 video 가정을 core에 내장하는 일을 피할 수 있습니다.
플러그인은 vendor surface를 소유하고, core는 capability 계약과 fallback 동작을 소유합니다.

video generation도 이미 같은 순서를 따릅니다. core가 typed
capability 계약과 런타임 helper를 소유하고, vendor 플러그인은
`api.registerVideoGenerationProvider(...)` 구현을 등록합니다.

구체적인 출시 체크리스트가 필요하다면
[Capability Cookbook](/ko/plugins/architecture)을 참고하세요.

## 계약 및 강제

플러그인 API surface는 의도적으로 typed이며
`OpenClawPluginApi`에 중앙화되어 있습니다. 이 계약은 지원되는 등록 지점과
플러그인이 의존할 수 있는 런타임 helper를 정의합니다.

이것이 중요한 이유:

- 플러그인 작성자는 하나의 안정적인 내부 표준을 얻음
- core는 두 플러그인이 같은 provider id를 등록하는 것 같은 중복 소유권을 거부할 수 있음
- 시작 시 잘못된 등록에 대해 실행 가능한 진단을 제공할 수 있음
- 계약 테스트는 번들 플러그인의 소유권을 강제하고 조용한 드리프트를 방지할 수 있음

강제는 두 계층으로 이루어집니다.

1. **런타임 등록 강제**
   플러그인 registry는 플러그인이 로드될 때 등록을 검증합니다. 예:
   중복 provider id, 중복 speech provider id, 잘못된
   등록은 정의되지 않은 동작 대신 플러그인 진단을 생성합니다.
2. **계약 테스트**
   번들 플러그인은 테스트 실행 중 계약 registry에 캡처되므로
   OpenClaw는 소유권을 명시적으로 주장할 수 있습니다. 현재는 모델
   providers, speech providers, web search providers, 번들 등록
   소유권에 사용됩니다.

실질적인 효과는 OpenClaw가 어떤 플러그인이 어떤
surface를 소유하는지 시작 시점에 미리 알고 있다는 점입니다. 그래서 core와 채널은
소유권이 선언되고 typed되며 테스트 가능하기 때문에
암묵적으로가 아니라 자연스럽게 조합될 수 있습니다.

### 계약에 들어가야 하는 것

좋은 플러그인 계약은 다음과 같습니다.

- typed
- 작음
- capability별
- core가 소유
- 여러 플러그인이 재사용 가능
- vendor 지식 없이 채널/기능에서 소비 가능

나쁜 플러그인 계약은 다음과 같습니다.

- core 안에 숨겨진 vendor별 policy
- registry를 우회하는 일회성 플러그인 escape hatch
- vendor 구현으로 바로 들어가는 채널 코드
- `OpenClawPluginApi` 또는 `api.runtime`의 일부가 아닌
  ad hoc 런타임 객체

확신이 서지 않으면 추상화 수준을 높이세요. 먼저 capability를 정의한 뒤
플러그인이 그것에 연결되게 하세요.

## 실행 모델

기본 OpenClaw 플러그인은 Gateway와 **동일 프로세스 내**에서 실행됩니다.
샌드박싱되지 않습니다. 로드된 기본 플러그인은 core 코드와 동일한 프로세스 수준의 신뢰 경계를 가집니다.

의미하는 바:

- 기본 플러그인은 tools, network handlers, hooks, services를 등록할 수 있음
- 기본 플러그인의 버그는 gateway를 충돌시키거나 불안정하게 만들 수 있음
- 악성 기본 플러그인은 OpenClaw 프로세스 내부의 임의 코드 실행과 동등함

호환 bundle은 현재 OpenClaw가 이를 메타데이터/콘텐츠 팩으로 취급하기 때문에
기본적으로 더 안전합니다. 현재 릴리스에서는 주로 번들
skills를 의미합니다.

번들이 아닌 플러그인에는 allowlist와 명시적인 install/load 경로를 사용하세요.
workspace 플러그인은 프로덕션 기본값이 아니라 개발 시점 코드로 취급하세요.

번들 workspace 패키지 이름의 경우 플러그인 id를 npm
이름에 고정하세요. 기본적으로 `@openclaw/<id>`이며, 또는
패키지가 의도적으로 더 좁은 플러그인 역할을 노출할 때는
`-provider`, `-plugin`, `-speech`, `-sandbox`, `-media-understanding` 같은 승인된 typed suffix를 사용할 수 있습니다.

중요한 신뢰 참고 사항:

- `plugins.allow`는 소스 provenance가 아니라 **플러그인 id**를 신뢰합니다.
- 번들 플러그인과 같은 id를 가진 workspace 플러그인은
  해당 workspace 플러그인이 활성화/allowlist되면 의도적으로 번들 사본을 가립니다.
- 이는 정상이며 로컬 개발, 패치 테스트, 핫픽스에 유용합니다.

## export 경계

OpenClaw는 구현 convenience가 아니라 capability를 export합니다.

capability 등록은 공개 상태로 유지하고, 비계약 helper export는 줄이세요.

- 번들 플러그인 전용 helper subpath
- 공개 API로 의도되지 않은 런타임 plumbing subpath
- vendor별 convenience helper
- 구현 세부사항인 setup/onboarding helper

일부 번들 플러그인 helper subpath는 호환성과 번들 플러그인 유지 관리를 위해
생성된 SDK export 맵에 아직 남아 있습니다. 현재 예로는
`plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`,
`plugin-sdk/zalo-setup`, 그리고 여러 `plugin-sdk/matrix*` seam이 있습니다.
이들은 새로운 서드파티 플러그인에 권장되는 SDK 패턴이 아니라
예약된 구현 세부 export로 취급하세요.

## 로드 파이프라인

시작 시 OpenClaw는 대략 다음 순서로 동작합니다.

1. 후보 플러그인 루트 검색
2. 기본 또는 호환 bundle manifest와 패키지 metadata 읽기
3. 안전하지 않은 후보 거부
4. 플러그인 config 정규화(`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. 각 후보에 대한 활성화 여부 결정
6. 활성화된 기본 모듈을 jiti로 로드
7. 기본 `register(api)`(또는 레거시 별칭인 `activate(api)`) hook을 호출하고 등록을 플러그인 registry에 수집
8. registry를 commands/런타임 surface에 노출

<Note>
`activate`는 `register`의 레거시 별칭입니다 — 로더는 존재하는 쪽(`def.register ?? def.activate`)을 해결하여 같은 시점에 호출합니다. 모든 번들 플러그인은 `register`를 사용하므로, 새 플러그인에는 `register`를 사용하세요.
</Note>

안전 게이트는 런타임 실행 **이전**에 발생합니다. 후보는
entry가 플러그인 루트 밖으로 벗어나거나, 경로가 world-writable이거나,
번들이 아닌 플러그인에서 경로 소유권이 의심스러우면 차단됩니다.

### Manifest 우선 동작

manifest는 control-plane의 기준 진실 공급원입니다. OpenClaw는 이것을 사용해 다음을 수행합니다.

- 플러그인 식별
- 선언된 channels/skills/config schema 또는 bundle capability 검색
- `plugins.entries.<id>.config` 검증
- Control UI label/placeholder 보강
- install/catalog metadata 표시

기본 플러그인의 경우 런타임 모듈은 data-plane 부분입니다. 실제 동작인
hooks, tools, commands, provider 흐름 등을 등록합니다.

### 로더가 캐시하는 것

OpenClaw는 프로세스 내 단기 캐시를 다음에 대해 유지합니다.

- discovery 결과
- manifest registry 데이터
- 로드된 플러그인 registry

이 캐시는 시작 시의 급격한 부하와 반복 명령 오버헤드를 줄입니다.
이를 지속 저장소가 아닌 수명 짧은 성능 캐시로 생각하면 됩니다.

성능 참고:

- 이 캐시를 비활성화하려면 `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` 또는
  `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1`을 설정하세요.
- 캐시 기간은 `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS`와
  `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`로 조정합니다.

## registry 모델

로드된 플러그인은 임의의 core 전역 상태를 직접 변경하지 않습니다. 대신
central plugin registry에 등록합니다.

registry는 다음을 추적합니다.

- 플러그인 레코드(identity, source, origin, status, diagnostics)
- tools
- 레거시 hooks 및 typed hooks
- channels
- providers
- gateway RPC handlers
- HTTP routes
- CLI registrars
- background services
- 플러그인 소유 commands

그런 다음 core 기능은 플러그인 모듈과 직접 통신하는 대신
이 registry를 읽습니다. 이렇게 하면 로딩이 한 방향으로 유지됩니다.

- 플러그인 모듈 -> registry 등록
- core 런타임 -> registry 소비

이 분리는 유지보수성에 중요합니다. 대부분의 core surface가
"registry를 읽기"라는 하나의 통합 지점만 필요하게 하며,
"모든 플러그인 모듈을 특별 처리"할 필요가 없어집니다.

## conversation 바인딩 콜백

conversation을 바인딩하는 플러그인은 승인이 해결되었을 때 반응할 수 있습니다.

바인드 요청이 승인 또는 거부된 뒤 콜백을 받으려면
`api.onConversationBindingResolved(...)`를 사용하세요.

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // A binding now exists for this plugin + conversation.
        console.log(event.binding?.conversationId);
        return;
      }

      // The request was denied; clear any local pending state.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

콜백 payload 필드:

- `status`: `"approved"` 또는 `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` 또는 `"deny"`
- `binding`: 승인된 요청의 해결된 바인딩
- `request`: 원래 요청 요약, detach 힌트, sender id, 그리고
  conversation metadata

이 콜백은 알림 전용입니다. 누가 conversation을 바인딩할 수 있는지는 바꾸지 않으며,
core 승인 처리가 끝난 뒤에 실행됩니다.

## Provider 런타임 hooks

Provider 플러그인은 이제 두 계층을 가집니다.

- manifest metadata: 런타임 로드 전에 가벼운 env-auth 조회를 위한 `providerAuthEnvVars`,
  그리고 런타임 로드 전에 가벼운 onboarding/auth-choice
  label과 CLI flag metadata를 위한 `providerAuthChoices`
- config 시점 hooks: `catalog` / 레거시 `discovery`와 `applyConfigDefaults`
- 런타임 hooks: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `shouldDeferSyntheticProfileAuth`,
  `resolveDynamicModel`, `prepareDynamicModel`, `normalizeResolvedModel`,
  `contributeResolvedModelCompat`, `capabilities`,
  `normalizeToolSchemas`, `inspectToolSchemas`,
  `resolveReasoningOutputMode`, `prepareExtraParams`, `createStreamFn`,
  `wrapStreamFn`, `resolveTransportTurnState`,
  `resolveWebSocketSessionPolicy`, `formatApiKey`, `refreshOAuth`,
  `buildAuthDoctorHint`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `isCacheTtlEligible`,
  `buildMissingAuthMessage`, `suppressBuiltInModel`, `augmentModelCatalog`,
  `isBinaryThinking`, `supportsXHighThinking`,
  `resolveDefaultThinkingLevel`, `isModernModelRef`, `prepareRuntimeAuth`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `createEmbeddingProvider`,
  `buildReplayPolicy`,
  `sanitizeReplayHistory`, `validateReplayTurns`, `onModelSelected`

OpenClaw는 여전히 generic agent loop, failover, transcript 처리, 그리고
tool policy를 소유합니다. 이 hooks는 전체 custom inference transport 없이
provider별 동작을 확장하기 위한 surface입니다.

provider에 런타임 로드 없이도 generic auth/status/model-picker 경로가 볼 수 있어야 하는
env 기반 자격 증명이 있다면 manifest `providerAuthEnvVars`를 사용하세요.
온보딩/auth-choice CLI surface가 provider의 choice id,
group label, 단일 flag auth wiring을 런타임 로드 없이 알아야 한다면
manifest `providerAuthChoices`를 사용하세요. provider 런타임
`envVars`는 온보딩 label이나 OAuth
client-id/client-secret 설정 변수 같은 운영자용 힌트에 유지하세요.

### Hook 순서와 사용법

모델/provider 플러그인의 경우 OpenClaw는 대략 다음 순서로 hooks를 호출합니다.
"사용 시점" 열은 빠른 판단 가이드입니다.

| #   | Hook                              | 역할                                                                                     | 사용 시점                                                                                                                                   |
| --- | --------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | `models.json` 생성 중 provider config를 `models.providers`에 게시                       | Provider가 카탈로그 또는 base URL 기본값을 소유할 때                                                                                        |
| 2   | `applyConfigDefaults`             | config 구체화 중 provider 소유 전역 config 기본값 적용                                  | 기본값이 auth 모드, env 또는 provider 모델 계열 의미론에 따라 달라질 때                                                                     |
| --  | _(내장 모델 조회)_                | OpenClaw가 먼저 일반 registry/catalog 경로를 시도                                      | _(플러그인 hook 아님)_                                                                                                                      |
| 3   | `normalizeModelId`                | 조회 전 레거시 또는 preview model-id 별칭 정규화                                        | Provider가 정식 모델 확인 전에 별칭 정리를 소유할 때                                                                                        |
| 4   | `normalizeTransport`              | generic 모델 조립 전 provider 계열 `api` / `baseUrl` 정규화                            | 같은 transport 계열 안에서 custom provider id용 transport 정리를 provider가 소유할 때                                                      |
| 5   | `normalizeConfig`                 | 런타임/provider 해결 전 `models.providers.<id>` 정규화                                  | 플러그인에 있어야 하는 config 정리가 필요할 때; 번들 Google 계열 helper도 지원되는 Google config 항목을 보완함                            |
| 6   | `applyNativeStreamingUsageCompat` | config provider에 기본 스트리밍 usage 호환성 재작성 적용                                | endpoint 기반 기본 스트리밍 usage metadata 보정이 필요할 때                                                                                |
| 7   | `resolveConfigApiKey`             | 런타임 auth 로드 전 config provider의 env-marker auth 해결                              | provider 소유 env-marker API 키 해결이 필요할 때; `amazon-bedrock`도 여기서 내장 AWS env-marker resolver를 가짐                           |
| 8   | `resolveSyntheticAuth`            | 일반 텍스트를 저장하지 않고 local/self-hosted 또는 config 기반 auth 노출                | provider가 synthetic/local credential marker로 동작할 수 있을 때                                                                            |
| 9   | `shouldDeferSyntheticProfileAuth` | env/config 기반 auth 뒤로 저장된 synthetic profile placeholder 우선순위 낮춤            | provider가 우선권을 가져오면 안 되는 synthetic placeholder profile을 저장할 때                                                              |
| 10  | `resolveDynamicModel`             | 아직 local registry에 없는 provider 소유 model id에 대한 동기 fallback                  | provider가 임의의 upstream model id를 허용할 때                                                                                             |
| 11  | `prepareDynamicModel`             | 비동기 준비 후 `resolveDynamicModel`을 다시 실행                                        | 알 수 없는 id를 해결하기 전에 network metadata가 필요할 때                                                                                  |
| 12  | `normalizeResolvedModel`          | embedded runner가 해결된 모델을 사용하기 전 최종 재작성                                 | core transport를 계속 사용하지만 transport 재작성이 필요할 때                                                                               |
| 13  | `contributeResolvedModelCompat`   | 다른 호환 transport 뒤의 vendor 모델에 compat 플래그 기여                              | provider를 직접 인수하지 않으면서 proxy transport 위 자신의 모델을 인식할 때                                                                |
| 14  | `capabilities`                    | 공유 core logic이 사용하는 provider 소유 transcript/tooling metadata                    | provider가 transcript/provider-family 특이사항을 가질 때                                                                                    |
| 15  | `normalizeToolSchemas`            | embedded runner가 보기 전 tool schema 정규화                                            | provider에 transport-family schema 정리가 필요할 때                                                                                        |
| 16  | `inspectToolSchemas`              | 정규화 후 provider 소유 schema 진단 노출                                                | core에 provider별 규칙을 가르치지 않고 keyword 경고를 표시하고 싶을 때                                                                      |
| 17  | `resolveReasoningOutputMode`      | 기본 출력 계약과 tagged reasoning-output 계약 중 선택                                   | provider가 기본 필드 대신 tagged reasoning/final output을 필요로 할 때                                                                      |
| 18  | `prepareExtraParams`              | generic stream option wrapper 적용 전 요청 매개변수 정규화                              | provider별 기본 요청 매개변수 또는 정리가 필요할 때                                                                                        |
| 19  | `createStreamFn`                  | 일반 stream 경로를 custom transport로 완전히 대체                                       | 단순 wrapper가 아닌 custom wire protocol이 필요할 때                                                                                       |
| 20  | `wrapStreamFn`                    | generic wrapper 적용 후 stream wrapper                                                  | custom transport 없이 요청 header/body/model compat wrapper가 필요할 때                                                                    |
| 21  | `resolveTransportTurnState`       | 기본 턴별 transport header 또는 metadata 첨부                                           | generic transport가 provider 기본 턴 identity를 보내야 할 때                                                                                |
| 22  | `resolveWebSocketSessionPolicy`   | 기본 WebSocket header 또는 세션 cool-down policy 첨부                                   | generic WS transport의 session header 또는 fallback policy 조정이 필요할 때                                                                 |
| 23  | `formatApiKey`                    | auth-profile formatter: 저장된 profile을 런타임 `apiKey` 문자열로 변환                  | provider가 추가 auth metadata를 저장하고 custom 런타임 token 형태가 필요할 때                                                              |
| 24  | `refreshOAuth`                    | custom refresh endpoint 또는 refresh-failure policy를 위한 OAuth refresh 재정의         | provider가 공유 `pi-ai` refresher에 맞지 않을 때                                                                                            |
| 25  | `buildAuthDoctorHint`             | OAuth refresh 실패 시 추가되는 복구 힌트 구성                                           | refresh 실패 후 provider 소유 auth 복구 안내가 필요할 때                                                                                   |
| 26  | `matchesContextOverflowError`     | provider 소유 context-window overflow 매처                                              | generic 휴리스틱이 놓치는 raw overflow 오류가 있을 때                                                                                       |
| 27  | `classifyFailoverReason`          | provider 소유 failover 사유 분류                                                        | raw API/transport 오류를 rate-limit/overload 등으로 매핑할 수 있을 때                                                                       |
| 28  | `isCacheTtlEligible`              | proxy/backhaul provider용 프롬프트 캐시 policy                                          | proxy별 cache TTL 게이팅이 필요할 때                                                                                                       |
| 29  | `buildMissingAuthMessage`         | generic missing-auth 복구 메시지 대체                                                   | provider별 missing-auth 복구 힌트가 필요할 때                                                                                              |
| 30  | `suppressBuiltInModel`            | 오래된 upstream 모델 억제 및 선택적 사용자 대상 오류 힌트                               | 오래된 upstream 항목을 숨기거나 vendor 힌트로 대체해야 할 때                                                                                |
| 31  | `augmentModelCatalog`             | discovery 후 synthetic/final catalog 행 추가                                            | `models list` 및 picker에 forward-compat synthetic 행이 필요할 때                                                                           |
| 32  | `isBinaryThinking`                | binary-thinking provider의 on/off 추론 토글                                             | provider가 이진 on/off 추론만 노출할 때                                                                                                    |
| 33  | `supportsXHighThinking`           | 선택된 모델에 대한 `xhigh` 추론 지원                                                    | provider가 일부 모델에만 `xhigh`를 허용하고 싶을 때                                                                                        |
| 34  | `resolveDefaultThinkingLevel`     | 특정 모델 계열의 기본 `/think` 수준                                                     | 모델 계열의 기본 `/think` policy를 provider가 소유할 때                                                                                     |
| 35  | `isModernModelRef`                | live profile 필터 및 smoke selection용 modern-model matcher                             | live/smoke 선호 모델 매칭을 provider가 소유할 때                                                                                            |
| 36  | `prepareRuntimeAuth`              | 추론 직전 구성된 자격 증명을 실제 런타임 token/key로 교환                              | token 교환 또는 짧은 수명의 요청 자격 증명이 필요할 때                                                                                      |
| 37  | `resolveUsageAuth`                | `/usage` 및 관련 status surface용 usage/billing 자격 증명 해결                         | custom usage/quota token 파싱 또는 다른 usage 자격 증명이 필요할 때                                                                         |
| 38  | `fetchUsageSnapshot`              | auth 해결 후 provider별 usage/quota snapshot 가져오기 및 정규화                         | provider별 usage endpoint 또는 payload parser가 필요할 때                                                                                   |
| 39  | `createEmbeddingProvider`         | memory/search용 provider 소유 embedding adapter 생성                                    | memory embedding 동작이 provider 플러그인에 속할 때                                                                                        |
| 40  | `buildReplayPolicy`               | provider용 transcript 처리 replay policy 반환                                          | provider에 custom transcript policy가 필요할 때(예: thinking-block 제거)                                                                    |
| 41  | `sanitizeReplayHistory`           | generic transcript 정리 후 replay history 재작성                                        | 공유 compaction helper를 넘는 provider별 replay 재작성이 필요할 때                                                                          |
| 42  | `validateReplayTurns`             | embedded runner 전 최종 replay-turn 검증 또는 재구성                                   | provider transport가 generic sanitation 이후 더 엄격한 turn 검증을 필요로 할 때                                                            |
| 43  | `onModelSelected`                 | provider 소유 post-selection 부작용 실행                                                | 모델이 활성화될 때 telemetry 또는 provider 소유 상태 처리가 필요할 때                                                                      |

`normalizeModelId`, `normalizeTransport`, `normalizeConfig`는 먼저
일치하는 provider 플러그인을 확인한 다음, 실제로 모델 id나
transport/config를 바꾸는 플러그인이 나올 때까지 다른 hook 가능 provider 플러그인으로 넘어갑니다.
이렇게 하면 호출자가 어떤 번들 플러그인이 재작성을 소유하는지 알 필요 없이
별칭/호환 provider shim이 작동합니다. 어떤 provider hook도 지원되는
Google 계열 config 항목을 재작성하지 않으면, 번들 Google config normalizer가
그 호환성 정리를 계속 적용합니다.

provider가 완전히 custom wire protocol이나 custom request executor를 필요로 한다면,
그것은 다른 종류의 extension입니다. 이 hooks는
여전히 OpenClaw의 일반 inference loop 위에서 동작하는 provider 동작을 위한 것입니다.

### Provider 예시

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### 내장 예시

- Anthropic은 `resolveDynamicModel`, `capabilities`, `buildAuthDoctorHint`,
  `resolveUsageAuth`, `fetchUsageSnapshot`, `isCacheTtlEligible`,
  `resolveDefaultThinkingLevel`, `applyConfigDefaults`, `isModernModelRef`,
  그리고 `wrapStreamFn`을 사용합니다. 이유는 Claude 4.6 forward-compat,
  provider-family 힌트, auth 복구 안내, usage endpoint 통합,
  prompt-cache 적격성, auth 인식 config 기본값, Claude
  기본/적응형 thinking policy, 그리고 beta header,
  `/fast` / `serviceTier`, `context1m`을 위한 Anthropic 전용 stream shaping을 소유하기 때문입니다.
- Anthropic의 Claude 전용 stream helper는 현재 번들 플러그인의 자체
  공개 `api.ts` / `contract-api.ts` seam에 남아 있습니다. 이 패키지 surface는
  generic SDK를 한 provider의 beta-header 규칙에 맞춰 넓히는 대신
  `wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
  `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, 그리고 더 저수준의
  Anthropic wrapper builder를 export합니다.
- OpenAI는 `resolveDynamicModel`, `normalizeResolvedModel`, `capabilities`와 함께
  `buildMissingAuthMessage`, `suppressBuiltInModel`,
  `augmentModelCatalog`, `supportsXHighThinking`, `isModernModelRef`를 사용합니다.
  이유는 GPT-5.4 forward-compat, 직접 OpenAI의
  `openai-completions` -> `openai-responses` 정규화, Codex 인식 auth
  힌트, Spark 억제, synthetic OpenAI list 행, GPT-5 thinking /
  live-model policy를 소유하기 때문입니다. `openai-responses-defaults` stream family는
  attribution header,
  `/fast`/`serviceTier`, 텍스트 verbosity, 기본 Codex 웹 검색,
  reasoning-compat payload shaping, Responses context management를 위한
  공유 기본 OpenAI Responses wrapper를 소유합니다.
- OpenRouter는 `catalog`와 함께 `resolveDynamicModel`,
  `prepareDynamicModel`을 사용합니다. 이 provider는 pass-through이므로
  OpenClaw의 정적 카탈로그가 갱신되기 전에 새 model id를 노출할 수 있기 때문입니다.
  또한 provider별 request header, routing metadata, reasoning patch,
  prompt-cache policy를 core 밖에 두기 위해 `capabilities`, `wrapStreamFn`,
  `isCacheTtlEligible`를 사용합니다. replay policy는
  `passthrough-gemini` family에서 오며, `openrouter-thinking` stream family는
  proxy reasoning 주입과 unsupported-model / `auto` 건너뛰기를 소유합니다.
- GitHub Copilot은 `catalog`, `auth`, `resolveDynamicModel`, `capabilities`와 함께
  `prepareRuntimeAuth`, `fetchUsageSnapshot`을 사용합니다. 이유는
  provider 소유 device login, 모델 fallback 동작, Claude transcript 특이사항,
  GitHub token -> Copilot token 교환, provider 소유 usage endpoint가 필요하기 때문입니다.
- OpenAI Codex는 `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth`, `augmentModelCatalog`와 함께
  `prepareExtraParams`, `resolveUsageAuth`, `fetchUsageSnapshot`을 사용합니다.
  이유는 여전히 core OpenAI transport 위에서 동작하지만 transport/base URL
  정규화, OAuth refresh fallback policy, 기본 transport 선택,
  synthetic Codex 카탈로그 행, ChatGPT usage endpoint 통합을 소유하기 때문입니다.
  이 플러그인은 direct OpenAI와 같은 `openai-responses-defaults` stream family를 공유합니다.
- Google AI Studio와 Gemini CLI OAuth는 `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn`, `isModernModelRef`를 사용합니다. 이유는
  `google-gemini` replay family가 Gemini 3.1 forward-compat fallback,
  기본 Gemini replay 검증, bootstrap replay 정리, tagged
  reasoning-output mode, modern-model 매칭을 소유하고,
  `google-thinking` stream family가 Gemini thinking payload 정규화를 소유하기 때문입니다.
  Gemini CLI OAuth는 또한 token formatting, token parsing, quota endpoint
  wiring을 위해 `formatApiKey`, `resolveUsageAuth`, `fetchUsageSnapshot`을 사용합니다.
- Anthropic Vertex는 `anthropic-by-model` replay family를 통해
  `buildReplayPolicy`를 사용하여 Claude 전용 replay 정리가
  모든 `anthropic-messages` transport가 아니라 Claude id에만 적용되도록 합니다.
- Amazon Bedrock은 `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `resolveDefaultThinkingLevel`을 사용합니다. 이유는
  Bedrock의 Anthropic 트래픽에 대한 Bedrock 전용 throttle/not-ready/context-overflow 오류 분류를
  소유하기 때문입니다. replay policy는 여전히 같은
  Claude 전용 `anthropic-by-model` guard를 공유합니다.
- OpenRouter, Kilocode, Opencode, Opencode Go는
  `passthrough-gemini` replay family를 통해 `buildReplayPolicy`를 사용합니다.
  이유는 Gemini 모델을 OpenAI 호환 transport로 프록시하며
  기본 Gemini replay 검증이나 bootstrap 재작성 없이도
  Gemini thought-signature 정리가 필요하기 때문입니다.
- MiniMax는 `hybrid-anthropic-openai` replay family를 통해
  `buildReplayPolicy`를 사용합니다. 이유는 하나의 provider가 Anthropic-message와
  OpenAI 호환 의미론을 모두 소유하기 때문입니다. Anthropic 측에서는 Claude 전용
  thinking-block 제거를 유지하면서 reasoning output mode를 기본 모드로 다시 재정의하고,
  `minimax-fast-mode` stream family는 공유 stream 경로에서
  fast-mode 모델 재작성을 소유합니다.
- Moonshot은 `catalog`와 함께 `wrapStreamFn`을 사용합니다. 이유는 여전히 공유
  OpenAI transport를 사용하지만 provider 소유 thinking payload 정규화가 필요하기 때문입니다.
  `moonshot-thinking` stream family는 config와 `/think` 상태를
  기본 binary thinking payload에 매핑합니다.
- Kilocode는 `catalog`, `capabilities`, `wrapStreamFn`,
  `isCacheTtlEligible`를 사용합니다. 이유는 provider 소유 request header,
  reasoning payload 정규화, Gemini transcript 힌트, Anthropic
  cache-TTL 게이팅이 필요하기 때문입니다. `kilocode-thinking` stream family는
  공유 proxy stream 경로에서 Kilo thinking 주입을 유지하면서 `kilo/auto` 및
  명시적 reasoning payload를 지원하지 않는 다른 proxy model id는 건너뜁니다.
- Z.AI는 `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth`, `fetchUsageSnapshot`을 사용합니다. 이유는 GLM-5 fallback,
  `tool_stream` 기본값, binary thinking UX, modern-model 매칭,
  usage auth와 quota 조회를 모두 소유하기 때문입니다. `tool-stream-default-on` stream family는
  기본 활성화된 `tool_stream` wrapper를 provider별 수작업 glue 밖으로 유지합니다.
- xAI는 `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel`, `isModernModelRef`를 사용합니다.
  이유는 기본 xAI Responses transport 정규화, Grok fast-mode
  별칭 재작성, 기본 `tool_stream`, strict-tool / reasoning-payload
  정리, 플러그인 소유 tool의 fallback auth 재사용, forward-compat Grok
  모델 해결, 그리고 xAI tool-schema 프로필, 지원되지 않는 schema keyword,
  기본 `web_search`, HTML-entity
  tool-call argument 디코딩 같은 provider 소유 compat patch를 소유하기 때문입니다.
- Mistral, OpenCode Zen, OpenCode Go는 transcript/tooling 특이사항을 core 밖에
  유지하기 위해 `capabilities`만 사용합니다.
- `byteplus`, `cloudflare-ai-gateway`,
  `huggingface`, `kimi-coding`, `nvidia`, `qianfan`,
  `synthetic`, `together`, `venice`, `vercel-ai-gateway`, `volcengine` 같은
  카탈로그 전용 번들 provider는 `catalog`만 사용합니다.
- Qwen은 텍스트 provider용 `catalog`와 함께 multimodal surface를 위한
  공유 media-understanding 및 video-generation 등록을 사용합니다.
- MiniMax와 Xiaomi는 추론은 여전히 공유 transport를 통해 실행되더라도
  `/usage` 동작이 플러그인 소유이기 때문에 `catalog`와 usage hooks를 함께 사용합니다.

## 런타임 helpers

플러그인은 `api.runtime`를 통해 선택된 core helper에 접근할 수 있습니다. TTS의 경우:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

참고:

- `textToSpeech`는 파일/voice-note surface용 일반 core TTS 출력 payload를 반환합니다.
- core `messages.tts` config와 provider 선택을 사용합니다.
- PCM 오디오 buffer + sample rate를 반환합니다. 플러그인은 provider용으로 resample/encode해야 합니다.
- `listVoices`는 provider별 선택 사항입니다. vendor 소유 voice picker 또는 setup 흐름에 사용하세요.
- voice 목록에는 provider 인식 picker용 locale, gender, personality tag 같은 더 풍부한 metadata가 포함될 수 있습니다.
- 현재 telephony는 OpenAI와 ElevenLabs가 지원합니다. Microsoft는 지원하지 않습니다.

플러그인은 `api.registerSpeechProvider(...)`를 통해 speech provider를 등록할 수도 있습니다.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

참고:

- TTS policy, fallback, 응답 전달은 core에 두세요.
- vendor 소유 synthesis 동작에는 speech provider를 사용하세요.
- 레거시 Microsoft `edge` 입력은 `microsoft` provider id로 정규화됩니다.
- 선호되는 소유권 모델은 회사 중심입니다. OpenClaw가 capability 계약을 추가함에 따라
  하나의 vendor 플러그인이 텍스트, 음성, 이미지, 미래의 미디어 provider까지 소유할 수 있습니다.

이미지/오디오/비디오 이해의 경우 플러그인은 generic key/value bag 대신
하나의 typed media-understanding provider를 등록합니다.

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

참고:

- orchestration, fallback, config, 채널 wiring은 core에 두세요.
- vendor 동작은 provider 플러그인에 두세요.
- 추가적 확장은 typed 상태를 유지해야 합니다. 새 optional method,
  새 optional result field, 새 optional capability 형태를 사용하세요.
- video generation도 이미 같은 패턴을 따릅니다.
  - core가 capability 계약과 runtime helper를 소유
  - vendor 플러그인이 `api.registerVideoGenerationProvider(...)`를 등록
  - 기능/채널 플러그인이 `api.runtime.videoGeneration.*`를 소비

media-understanding 런타임 helper의 경우 플러그인은 다음을 호출할 수 있습니다.

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});
```

오디오 전사의 경우 플러그인은 media-understanding 런타임 또는
이전 STT 별칭을 사용할 수 있습니다.

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

참고:

- `api.runtime.mediaUnderstanding.*`는
  이미지/오디오/비디오 이해를 위한 선호되는 공유 surface입니다.
- core media-understanding 오디오 config(`tools.media.audio`)와 provider fallback 순서를 사용합니다.
- 전사 결과가 생성되지 않으면(예: 건너뜀/지원되지 않는 입력) `{ text: undefined }`를 반환합니다.
- `api.runtime.stt.transcribeAudioFile(...)`는 호환성 별칭으로 계속 남아 있습니다.

플러그인은 또한 `api.runtime.subagent`를 통해 백그라운드 subagent 실행을 시작할 수 있습니다.

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

참고:

- `provider`와 `model`은 영구 session 변경이 아니라 실행별 optional 재정의입니다.
- OpenClaw는 신뢰된 호출자에 대해서만 이 재정의 필드를 허용합니다.
- 플러그인 소유 fallback 실행에는 운영자가 `plugins.entries.<id>.subagent.allowModelOverride: true`로 옵트인해야 합니다.
- 신뢰된 플러그인을 특정 정식 `provider/model` 대상에 제한하려면 `plugins.entries.<id>.subagent.allowedModels`를 사용하세요. 모든 대상을 명시적으로 허용하려면 `"*"`를 사용합니다.
- 신뢰되지 않은 플러그인 subagent 실행도 여전히 작동하지만, 재정의 요청은 조용히 fallback되지 않고 거부됩니다.

웹 검색의 경우 플러그인은 agent tool wiring 내부로 들어가는 대신
공유 런타임 helper를 소비할 수 있습니다.

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

플러그인은 또한
`api.registerWebSearchProvider(...)`를 통해 web-search provider를 등록할 수 있습니다.

참고:

- provider 선택, 자격 증명 해결, 공유 요청 의미론은 core에 두세요.
- vendor별 검색 transport에는 web-search provider를 사용하세요.
- `api.runtime.webSearch.*`는 agent tool wrapper에 의존하지 않고 검색 동작이 필요한
  기능/채널 플러그인에 권장되는 공유 surface입니다.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: 구성된 이미지 생성 provider 체인을 사용해 이미지를 생성합니다.
- `listProviders(...)`: 사용 가능한 이미지 생성 provider와 그 capability를 나열합니다.

## Gateway HTTP routes

플러그인은 `api.registerHttpRoute(...)`를 사용해 HTTP endpoint를 노출할 수 있습니다.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

route 필드:

- `path`: gateway HTTP 서버 아래의 route 경로
- `auth`: 필수. 일반 gateway auth를 요구하려면 `"gateway"`를, 플러그인 관리 auth/webhook 검증에는 `"plugin"`을 사용합니다.
- `match`: 선택 사항. `"exact"`(기본값) 또는 `"prefix"`.
- `replaceExisting`: 선택 사항. 같은 플러그인이 기존 route 등록을 교체할 수 있게 합니다.
- `handler`: route가 요청을 처리했다면 `true`를 반환합니다.

참고:

- `api.registerHttpHandler(...)`는 제거되었으며 플러그인 로드 오류를 발생시킵니다. 대신 `api.registerHttpRoute(...)`를 사용하세요.
- 플러그인 route는 `auth`를 명시적으로 선언해야 합니다.
- 정확히 같은 `path + match` 충돌은 `replaceExisting: true`가 없는 한 거부되며, 한 플러그인은 다른 플러그인의 route를 교체할 수 없습니다.
- 서로 다른 `auth` 수준의 겹치는 route는 거부됩니다. `exact`/`prefix` fallthrough 체인은 같은 auth 수준에서만 유지하세요.
- `auth: "plugin"` route는 운영자 런타임 scope를 자동으로 받지 않습니다. 이것은 권한 있는 Gateway helper 호출이 아니라
  플러그인 관리 webhook/서명 검증용입니다.
- `auth: "gateway"` route는 Gateway 요청 런타임 scope 안에서 실행되지만,
  그 scope는 의도적으로 보수적입니다.
  - shared-secret bearer auth(`gateway.auth.mode = "token"` / `"password"`)는
    호출자가 `x-openclaw-scopes`를 보내더라도 플러그인 route 런타임 scope를 `operator.write`에 고정합니다
  - 신뢰된 identity 보유 HTTP 모드(예: `trusted-proxy` 또는 비공개 ingress에서의 `gateway.auth.mode = "none"`)는
    헤더가 명시적으로 존재할 때만 `x-openclaw-scopes`를 따릅니다
  - 그런 identity 보유 플러그인 route 요청에서 `x-openclaw-scopes`가 없으면
    런타임 scope는 `operator.write`로 대체됩니다
- 실용적인 규칙: gateway-auth 플러그인 route가 암묵적인 admin surface라고 가정하지 마세요.
  route에 admin 전용 동작이 필요하다면 identity를 보유한 auth 모드를 요구하고
  명시적 `x-openclaw-scopes` header 계약을 문서화하세요.

## Plugin SDK import 경로

플러그인을 작성할 때는 단일 `openclaw/plugin-sdk` import 대신
SDK subpath를 사용하세요.

- 플러그인 등록 기본 요소에는 `openclaw/plugin-sdk/plugin-entry`
- generic 공유 플러그인 대상 계약에는 `openclaw/plugin-sdk/core`
- 루트 `openclaw.json` Zod schema export(`OpenClawSchema`)에는 `openclaw/plugin-sdk/config-schema`
- 안정적인 채널 기본 요소로는 `openclaw/plugin-sdk/channel-setup`,
  `openclaw/plugin-sdk/setup-runtime`,
  `openclaw/plugin-sdk/setup-adapter-runtime`,
  `openclaw/plugin-sdk/setup-tools`,
  `openclaw/plugin-sdk/channel-pairing`,
  `openclaw/plugin-sdk/channel-contract`,
  `openclaw/plugin-sdk/channel-feedback`,
  `openclaw/plugin-sdk/channel-inbound`,
  `openclaw/plugin-sdk/channel-lifecycle`,
  `openclaw/plugin-sdk/channel-reply-pipeline`,
  `openclaw/plugin-sdk/command-auth`,
  `openclaw/plugin-sdk/secret-input`, 그리고
  공유 setup/auth/reply/webhook wiring용 `openclaw/plugin-sdk/webhook-ingress`
  가 있습니다. `channel-inbound`는 debounce, mention matching,
  envelope formatting, inbound envelope context helper의 공유 홈입니다.
  `channel-setup`은 좁은 optional-install setup seam입니다.
  `setup-runtime`은 `setupEntry` /
  지연 시작에서 사용되는 런타임 안전 setup surface이며, import-safe setup patch adapter를 포함합니다.
  `setup-adapter-runtime`은 env 인식 account-setup adapter seam입니다.
  `setup-tools`는 작은 CLI/archive/docs helper seam입니다(`formatCliCommand`,
  `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`,
  `CONFIG_DIR`).
- 도메인 subpath로는 `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store`, 그리고
  공유 런타임/config helper용 `openclaw/plugin-sdk/directory-runtime`
  가 있습니다. `telegram-command-config`는 Telegram custom
  command 정규화/검증을 위한 좁은 공개 seam이며 번들
  Telegram 계약 surface를 일시적으로 사용할 수 없을 때도 계속 제공됩니다.
  `text-runtime`은 assistant-visible-text 제거,
  markdown 렌더/청킹 helper, redaction
  helper, directive-tag helper, safe-text utility를 포함하는 공유 text/markdown/logging seam입니다.
- 승인 전용 채널 seam은 플러그인의 하나의 `approvalCapability`
  계약을 우선 사용해야 합니다. 그러면 core는 승인 동작을 관련 없는 플러그인 필드에 섞는 대신
  그 하나의 capability를 통해 approval auth, delivery, render, native-routing 동작을 읽습니다.
- `openclaw/plugin-sdk/channel-runtime`은 더 이상 권장되지 않으며
  오래된 플러그인을 위한 호환 shim으로만 남아 있습니다. 새 코드는 더 좁은
  generic primitive를 import해야 하며, repo 코드도 shim에 대한 새 import를 추가해서는 안 됩니다.
- 번들 extension 내부 구조는 비공개로 유지됩니다. 외부 플러그인은 오직
  `openclaw/plugin-sdk/*` subpath만 사용해야 합니다. OpenClaw core/test 코드는
  `index.js`, `api.js`,
  `runtime-api.js`, `setup-entry.js`와
  `login-qr-api.js` 같은 좁은 파일 등 플러그인 패키지 루트 아래의 repo 공개 엔트리 포인트를 사용할 수 있습니다.
  core 또는 다른 extension에서 플러그인 패키지의 `src/*`를 import하지 마세요.
- repo 엔트리 포인트 분리:
  `<plugin-package-root>/api.js`는 helper/types barrel,
  `<plugin-package-root>/runtime-api.js`는 runtime 전용 barrel,
  `<plugin-package-root>/index.js`는 번들 플러그인 엔트리,
  `<plugin-package-root>/setup-entry.js`는 setup 플러그인 엔트리입니다.
- 현재 번들 provider 예시:
  - Anthropic은 `wrapAnthropicProviderStream`, beta-header helper,
    `service_tier` 파싱 같은 Claude stream helper를 위해
    `api.js` / `contract-api.js`를 사용합니다.
  - OpenAI는 provider builder, default-model helper,
    realtime provider builder를 위해 `api.js`를 사용합니다.
  - OpenRouter는 provider builder와 onboarding/config
    helper를 위해 `api.js`를 사용하며, `register.runtime.js`는 여전히 repo 로컬 사용을 위해
    generic `plugin-sdk/provider-stream` helper를 다시 export할 수 있습니다.
- facade로 로드되는 공개 엔트리 포인트는 가능한 경우 활성 런타임 config snapshot을 우선 사용하고,
  OpenClaw가 아직 런타임 snapshot을 제공하지 않을 때는 디스크의 해결된 config 파일을 fallback으로 사용합니다.
- generic 공유 primitive가 여전히 선호되는 공개 SDK 계약입니다. 번들 채널 브랜드가 붙은
  소수의 예약된 호환 helper seam은 아직 존재합니다. 이를 새
  서드파티 import 대상이 아니라 번들 유지관리/호환성 seam으로 취급하세요.
  새로운 교차 채널 계약은 여전히 generic `plugin-sdk/*` subpath 또는
  플러그인 로컬 `api.js` / `runtime-api.js` barrel에 위치해야 합니다.

호환성 참고:

- 새 코드에서는 루트 `openclaw/plugin-sdk` barrel을 피하세요.
- 먼저 좁고 안정적인 primitive를 우선하세요. newer setup/pairing/reply/
  feedback/contract/inbound/threading/command/secret-input/webhook/infra/
  allowlist/status/message-tool subpath가 새 번들 및 외부 플러그인 작업을 위한
  의도된 계약입니다.
  대상 파싱/매칭은 `openclaw/plugin-sdk/channel-targets`에 두어야 합니다.
  message action gate와 reaction message-id helper는
  `openclaw/plugin-sdk/channel-actions`에 두어야 합니다.
- 번들 extension 전용 helper barrel은 기본적으로 안정적이지 않습니다.
  helper가 번들 extension에만 필요하다면
  `openclaw/plugin-sdk/<extension>`로 승격하는 대신 해당 extension의
  로컬 `api.js` 또는 `runtime-api.js` seam 뒤에 두세요.
- 새로운 공유 helper seam은 generic이어야 하며 채널 브랜드가 붙어서는 안 됩니다. 공유 대상
  파싱은 `openclaw/plugin-sdk/channel-targets`에 두고, 채널별 내부 구조는
  소유 플러그인의 로컬 `api.js` 또는 `runtime-api.js`
  seam 뒤에 유지하세요.
- `image-generation`,
  `media-understanding`, `speech` 같은 capability별 subpath는
  번들/기본 플러그인이 현재 사용하고 있기 때문에 존재합니다. 하지만 존재 자체가
  export된 모든 helper가 장기적으로 고정된 외부 계약이라는 뜻은 아닙니다.

## Message tool schema

플러그인은 채널별 `describeMessageTool(...)` schema 기여분을
소유해야 합니다. provider별 필드는 공유 core가 아니라 플러그인에 두세요.

공유 가능한 이식성 schema fragment에는
`openclaw/plugin-sdk/channel-actions`를 통해 export되는 generic helper를 재사용하세요.

- 버튼 그리드 스타일 payload용 `createMessageToolButtonsSchema()`
- 구조화된 card payload용 `createMessageToolCardSchema()`

어떤 schema 형태가 한 provider에만 의미가 있다면,
그것을 공유 SDK로 승격하는 대신 해당 플러그인의 자체 소스에 정의하세요.

## 채널 대상 해결

채널 플러그인은 채널별 대상 의미론을 소유해야 합니다. 공유
outbound host는 generic하게 유지하고, provider 규칙에는 messaging adapter surface를 사용하세요.

- `messaging.inferTargetChatType({ to })`는 정규화된 대상을 directory 조회 전에
  `direct`, `group`, `channel` 중 무엇으로 취급할지 결정합니다.
- `messaging.targetResolver.looksLikeId(raw, normalized)`는
  입력이 directory 검색 대신 id와 유사한 해결 경로로 바로 가야 하는지 core에 알려줍니다.
- `messaging.targetResolver.resolveTarget(...)`는 정규화 후 또는
  directory 미스 이후 core가 최종 provider 소유 해결이 필요할 때 사용하는 플러그인 fallback입니다.
- `messaging.resolveOutboundSessionRoute(...)`는 대상이 해결된 뒤
  provider별 session route 구성을 소유합니다.

권장 분리:

- peer/group 검색 전에 일어나야 하는 범주 결정에는 `inferTargetChatType`을 사용하세요.
- "이것을 명시적/기본 대상 id로 취급" 판단에는 `looksLikeId`를 사용하세요.
- broad directory 검색이 아니라 provider별 정규화 fallback에는 `resolveTarget`을 사용하세요.
- chat id, thread id, JID, handle, room id 같은 provider 기본 id는
  generic SDK 필드가 아니라 `target` 값 또는 provider별 params 안에 두세요.

## Config 기반 directory

config에서 directory 항목을 파생하는 플러그인은 그 로직을 플러그인에 두고,
`openclaw/plugin-sdk/directory-runtime`의 공유 helper를 재사용해야 합니다.

채널에 다음과 같은 config 기반 peer/group이 필요할 때 사용하세요.

- allowlist 기반 DM peer
- 구성된 채널/group 맵
- account 범위의 정적 directory fallback

`directory-runtime`의 공유 helper는 generic 작업만 처리합니다.

- query 필터링
- limit 적용
- deduping/정규화 helper
- `ChannelDirectoryEntry[]` 생성

채널별 account 검사와 id 정규화는 플러그인 구현에 남겨 두세요.

## Provider 카탈로그

Provider 플러그인은
`registerProvider({ catalog: { run(...) { ... } } })`를 사용해 추론용 모델 카탈로그를 정의할 수 있습니다.

`catalog.run(...)`은 OpenClaw가
`models.providers`에 기록하는 것과 동일한 형태를 반환합니다.

- 하나의 provider 항목에는 `{ provider }`
- 여러 provider 항목에는 `{ providers }`

provider별 model id, base URL
기본값 또는 auth 게이트 model metadata를 플러그인이 소유할 때 `catalog`를 사용하세요.

`catalog.order`는 플러그인의 카탈로그가 OpenClaw의
내장 암시적 provider에 대해 언제 병합될지를 제어합니다.

- `simple`: 일반 API 키 또는 env 기반 provider
- `profile`: auth profile이 있을 때 나타나는 provider
- `paired`: 여러 관련 provider 항목을 합성하는 provider
- `late`: 다른 암시적 provider 이후 마지막 단계

나중 provider가 키 충돌 시 우선하므로,
플러그인은 같은 provider id를 가진 내장 provider 항목을 의도적으로 재정의할 수 있습니다.

호환성:

- `discovery`는 여전히 레거시 별칭으로 작동합니다
- `catalog`와 `discovery`가 모두 등록되면 OpenClaw는 `catalog`를 사용합니다

## 읽기 전용 채널 검사

플러그인이 채널을 등록한다면,
`resolveAccount(...)`와 함께 `plugin.config.inspectAccount(cfg, accountId)`를 구현하는 것을 권장합니다.

이유:

- `resolveAccount(...)`는 런타임 경로입니다. 자격 증명이 완전히 구체화되어 있다고 가정할 수 있으며,
  필요한 secret가 없으면 빠르게 실패해도 됩니다.
- `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve`, doctor/config
  복구 흐름 같은 읽기 전용 명령 경로는
  단지 구성을 설명하기 위해 런타임 자격 증명을 구체화할 필요가 없어야 합니다.

권장 `inspectAccount(...)` 동작:

- 설명적인 account 상태만 반환
- `enabled`와 `configured` 유지
- 관련 있는 경우 자격 증명 source/status 필드 포함:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- 읽기 전용 가용성을 보고하기 위해 raw token 값을 반환할 필요는 없습니다.
  `tokenStatus: "available"`(및 대응하는 source 필드)만 반환해도 status류 명령에는 충분합니다.
- SecretRef를 통해 자격 증명이 구성되었지만 현재 명령 경로에서 사용할 수 없다면
  `configured_unavailable`을 사용하세요.

이렇게 하면 읽기 전용 명령이 충돌하거나 account를 미구성 상태로 잘못 표시하는 대신
"이 명령 경로에서는 구성되었지만 사용할 수 없음"을 보고할 수 있습니다.

## 패키지 팩

플러그인 디렉터리에는 `openclaw.extensions`가 포함된 `package.json`이 있을 수 있습니다.

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

각 항목은 하나의 플러그인이 됩니다. 팩에 여러 extension이 나열되면 플러그인 id는
`name/<fileBase>`가 됩니다.

플러그인이 npm dependency를 import한다면,
해당 디렉터리에 `node_modules`가 있도록 그 디렉터리에서 설치하세요(`npm install` / `pnpm install`).

보안 가드레일: 모든 `openclaw.extensions` 항목은 symlink 해석 후에도 플러그인
디렉터리 내부에 있어야 합니다. 패키지 디렉터리 밖으로 벗어나는 항목은
거부됩니다.

보안 참고: `openclaw plugins install`은 플러그인 dependency를
`npm install --omit=dev --ignore-scripts`로 설치합니다(런타임에서 lifecycle script 없음, dev dependency 없음). 플러그인 dependency
트리는 "순수 JS/TS"로 유지하고 `postinstall` 빌드를 요구하는 패키지는 피하세요.

선택 사항: `openclaw.setupEntry`는 가벼운 setup 전용 모듈을 가리킬 수 있습니다.
OpenClaw가 비활성화된 채널 플러그인에 대해 setup surface가 필요하거나,
채널 플러그인이 활성화되어 있지만 아직 구성되지 않은 경우,
전체 플러그인 엔트리 대신 `setupEntry`를 로드합니다. 이렇게 하면
메인 플러그인 엔트리가 tools, hooks 또는 다른 런타임 전용
코드도 연결하는 경우 시작과 setup을 더 가볍게 유지할 수 있습니다.

선택 사항: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`은
채널 플러그인이 이미 구성된 경우에도 gateway의
pre-listen 시작 단계에서 같은 `setupEntry` 경로를 사용하도록 옵트인할 수 있게 합니다.

이 옵션은 `setupEntry`가 gateway가 수신을 시작하기 전에 존재해야 하는
시작 surface를 완전히 포함할 때만 사용하세요. 실제로 이것은
setup entry가 시작이 의존하는 모든 채널 소유 capability를 등록해야 함을 의미합니다. 예를 들어:

- 채널 등록 자체
- gateway가 수신을 시작하기 전에 사용 가능해야 하는 모든 HTTP route
- 같은 기간 중 존재해야 하는 gateway 메서드, tools, services

전체 엔트리가 여전히 필수 시작 capability를 소유한다면 이 플래그를 활성화하지 마세요.
기본 동작을 유지하고 OpenClaw가 시작 시 전체 엔트리를 로드하게 하세요.

번들 채널은 또한 전체 채널 런타임이 로드되기 전에 core가 참조할 수 있는
setup 전용 계약 surface helper를 게시할 수 있습니다. 현재 setup 승격 surface는 다음과 같습니다.

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Core는 레거시 단일 account 채널 config를
전체 플러그인 엔트리를 로드하지 않고 `channels.<id>.accounts.*`로 승격해야 할 때 이 surface를 사용합니다.
Matrix가 현재 번들 예시입니다. named account가 이미 존재할 때
auth/bootstrap key만 named 승격 account로 이동하고,
항상 `accounts.default`를 생성하는 대신 구성된 비정규 default-account key를 보존할 수 있습니다.

이 setup patch adapter는 번들 계약 surface discovery를 lazy하게 유지합니다.
import 시점은 가볍게 유지되고, 승격 surface는 모듈 import에서 다시 채널 시작으로 진입하는 대신
처음 사용할 때만 로드됩니다.

이러한 시작 surface에 gateway RPC 메서드가 포함된다면,
플러그인 전용 prefix 아래에 두세요. Core admin namespace(`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`)는 예약되어 있으며
플러그인이 더 좁은 scope를 요청하더라도 항상 `operator.admin`으로 해결됩니다.

예시:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### 채널 카탈로그 metadata

채널 플러그인은 `openclaw.channel`을 통해 setup/discovery metadata를,
`openclaw.install`을 통해 install 힌트를 광고할 수 있습니다. 이렇게 하면 core 카탈로그를 데이터 없이 유지할 수 있습니다.

예시:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

최소 예시 외에 유용한 `openclaw.channel` 필드:

- `detailLabel`: 더 풍부한 카탈로그/status surface를 위한 보조 label
- `docsLabel`: docs 링크 텍스트 재정의
- `preferOver`: 이 카탈로그 항목이 더 높은 우선순위를 가져야 하는 낮은 우선순위 플러그인/채널 id
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: selection-surface 문구 제어
- `markdownCapable`: 아웃바운드 formatting 판단을 위해 채널이 markdown 지원 가능함을 표시
- `exposure.configured`: `false`로 설정 시 구성된 채널 목록 surface에서 채널 숨김
- `exposure.setup`: `false`로 설정 시 인터랙티브 setup/configure picker에서 채널 숨김
- `exposure.docs`: docs 탐색 surface에서 채널을 내부/비공개로 표시
- `showConfigured` / `showInSetup`: 호환성을 위해 여전히 허용되는 레거시 별칭; `exposure`를 권장
- `quickstartAllowFrom`: 채널을 표준 quickstart `allowFrom` 흐름에 옵트인
- `forceAccountBinding`: account가 하나뿐이어도 명시적 account 바인딩 요구
- `preferSessionLookupForAnnounceTarget`: announce 대상 해결 시 session lookup 우선

OpenClaw는 **외부 채널 카탈로그**도 병합할 수 있습니다(예: MPM
registry export). 다음 중 한 위치에 JSON 파일을 두세요.

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

또는 `OPENCLAW_PLUGIN_CATALOG_PATHS`(또는 `OPENCLAW_MPM_CATALOG_PATHS`)를
하나 이상의 JSON 파일로 지정하세요(쉼표/세미콜론/`PATH` 구분). 각 파일은
`{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`
형태를 포함해야 합니다. 파서는 `"entries"` 키의 레거시 별칭으로 `"packages"` 또는 `"plugins"`도 허용합니다.

## 컨텍스트 엔진 플러그인

컨텍스트 엔진 플러그인은 세션 컨텍스트 orchestration의 ingest, assemble,
compaction을 소유합니다. 플러그인에서
`api.registerContextEngine(id, factory)`로 등록한 뒤, 활성 엔진은
`plugins.slots.contextEngine`으로 선택하세요.

기본 컨텍스트 파이프라인을 단지 memory search나 hooks로 확장하는 수준이 아니라
대체하거나 확장해야 할 때 사용하세요.

```ts
export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

엔진이 compaction 알고리즘을 **직접 소유하지 않는다면**,
`compact()`를 구현하고 명시적으로 위임하세요.

```ts
import { delegateCompactionToRuntime } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", () => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages }) {
      return { messages, estimatedTokens: 0 };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## 새 capability 추가

플러그인에 현재 API에 맞지 않는 동작이 필요하다면,
비공개 직접 접근으로 플러그인 시스템을 우회하지 마세요. 누락된 capability를 추가하세요.

권장 순서:

1. core 계약 정의
   core가 어떤 공유 동작을 소유해야 하는지 결정합니다: policy, fallback, config 병합,
   lifecycle, 채널 대상 의미론, 런타임 helper 형태.
2. typed 플러그인 등록/런타임 surface 추가
   가장 작은 유용한 typed capability surface로 `OpenClawPluginApi` 및/또는 `api.runtime`을 확장합니다.
3. core + 채널/기능 소비자 연결
   채널과 기능 플러그인은 vendor 구현을 직접 import하는 대신
   core를 통해 새 capability를 소비해야 합니다.
4. vendor 구현 등록
   그런 다음 vendor 플러그인이 해당 capability에 backend를 등록합니다.
5. 계약 커버리지 추가
   시간이 지나도 소유권과 등록 형태가 명확하게 유지되도록 테스트를 추가합니다.

이것이 OpenClaw가 특정 provider의 세계관에 하드코딩되지 않으면서도
의견 있는 시스템으로 남는 방식입니다. 구체적인 파일 체크리스트와 예시는
[Capability Cookbook](/ko/plugins/architecture)을 참고하세요.

### capability 체크리스트

새 capability를 추가할 때 구현은 일반적으로 다음 surface를 함께 수정해야 합니다.

- `src/<capability>/types.ts`의 core 계약 타입
- `src/<capability>/runtime.ts`의 core runner/runtime helper
- `src/plugins/types.ts`의 플러그인 API 등록 surface
- `src/plugins/registry.ts`의 플러그인 registry wiring
- 기능/채널 플러그인이 소비해야 할 때 `src/plugins/runtime/*`의
  플러그인 런타임 노출
- `src/test-utils/plugin-registration.ts`의 capture/test helper
- `src/plugins/contracts/registry.ts`의 소유권/계약 assertion
- `docs/`의 운영자/플러그인 문서

이 surface 중 하나라도 빠져 있다면, 대개 그 capability가
아직 완전히 통합되지 않았다는 신호입니다.

### capability 템플릿

최소 패턴:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

계약 테스트 패턴:

```ts
expect(findVideoGenerationProviderIdsForPlugin("openai")).toEqual(["openai"]);
```

이렇게 하면 규칙이 단순해집니다.

- core는 capability 계약 + orchestration을 소유
- vendor 플러그인은 vendor 구현을 소유
- 기능/채널 플러그인은 runtime helper를 소비
- 계약 테스트는 소유권을 명시적으로 유지
