---
read_when:
    - 네이티브 OpenClaw 플러그인을 빌드하거나 디버깅할 때
    - 플러그인 capability 모델이나 소유권 경계를 이해할 때
    - 플러그인 로드 파이프라인이나 레지스트리 작업을 할 때
    - 제공자 런타임 훅이나 채널 플러그인을 구현할 때
sidebarTitle: Internals
summary: '플러그인 내부 구조: capability 모델, 소유권, 계약, 로드 파이프라인, 런타임 헬퍼'
title: 플러그인 내부 구조
x-i18n:
    generated_at: "2026-04-09T01:33:22Z"
    model: gpt-5.4
    provider: openai
    source_hash: 2575791f835990589219bb06d8ca92e16a8c38b317f0bfe50b421682f253ef18
    source_path: plugins/architecture.md
    workflow: 15
---

# 플러그인 내부 구조

<Info>
  이 문서는 **심층 아키텍처 참조**입니다. 실용적인 가이드는 다음을 참조하세요.
  - [플러그인 설치 및 사용](/ko/tools/plugin) — 사용자 가이드
  - [시작하기](/ko/plugins/building-plugins) — 첫 플러그인 튜토리얼
  - [채널 플러그인](/ko/plugins/sdk-channel-plugins) — 메시징 채널 빌드
  - [제공자 플러그인](/ko/plugins/sdk-provider-plugins) — 모델 제공자 빌드
  - [SDK 개요](/ko/plugins/sdk-overview) — import 맵 및 등록 API
</Info>

이 페이지에서는 OpenClaw 플러그인 시스템의 내부 아키텍처를 다룹니다.

## 공개 capability 모델

Capabilities는 OpenClaw 내부의 공개 **네이티브 플러그인** 모델입니다. 모든 네이티브 OpenClaw 플러그인은 하나 이상의 capability 유형에 대해 등록됩니다.

| Capability             | 등록 메서드                                      | 예시 플러그인                        |
| ---------------------- | ------------------------------------------------ | ------------------------------------ |
| 텍스트 추론            | `api.registerProvider(...)`                      | `openai`, `anthropic`                |
| CLI 추론 백엔드        | `api.registerCliBackend(...)`                    | `openai`, `anthropic`                |
| 음성                   | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`            |
| 실시간 전사            | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                             |
| 실시간 음성            | `api.registerRealtimeVoiceProvider(...)`         | `openai`                             |
| 미디어 이해            | `api.registerMediaUnderstandingProvider(...)`    | `openai`, `google`                   |
| 이미지 생성            | `api.registerImageGenerationProvider(...)`       | `openai`, `google`, `fal`, `minimax` |
| 음악 생성              | `api.registerMusicGenerationProvider(...)`       | `google`, `minimax`                  |
| 비디오 생성            | `api.registerVideoGenerationProvider(...)`       | `qwen`                               |
| 웹 가져오기            | `api.registerWebFetchProvider(...)`              | `firecrawl`                          |
| 웹 검색                | `api.registerWebSearchProvider(...)`             | `google`                             |
| 채널 / 메시징          | `api.registerChannel(...)`                       | `msteams`, `matrix`                  |

capability를 하나도 등록하지 않고 훅, 도구 또는 서비스를 제공하는 플러그인은 **레거시 hook-only** 플러그인입니다. 이 패턴도 여전히 완전히 지원됩니다.

### 외부 호환성 입장

capability 모델은 이미 코어에 도입되어 있고 현재 번들/네이티브 플러그인에서 사용되고 있지만, 외부 플러그인 호환성은 여전히 "export되었으니 고정되었다"보다 더 엄격한 기준이 필요합니다.

현재 가이드는 다음과 같습니다.

- **기존 외부 플러그인:** hook 기반 통합이 계속 작동하도록 유지하며, 이를 호환성 기준선으로 취급합니다.
- **새 번들/네이티브 플러그인:** 벤더별 직접 접근이나 새로운 hook-only 설계보다 명시적인 capability 등록을 우선합니다.
- **capability 등록을 도입하는 외부 플러그인:** 허용되지만, 문서에서 계약이 안정적이라고 명시하지 않은 한 capability별 헬퍼 표면은 계속 진화하는 것으로 간주합니다.

실무 규칙:

- capability 등록 API가 의도된 방향입니다.
- 전환 기간 동안 레거시 훅은 외부 플러그인에 가장 안전한 무중단 경로로 남아 있습니다.
- export된 헬퍼 하위 경로는 모두 동일하지 않습니다. 우연히 노출된 헬퍼 export가 아니라, 문서화된 좁은 계약을 우선하세요.

### 플러그인 형태

OpenClaw는 로드된 각 플러그인을 정적 메타데이터가 아니라 실제 등록 동작에 따라 형태로 분류합니다.

- **plain-capability** -- 정확히 하나의 capability 유형만 등록합니다(예: `mistral` 같은 provider-only 플러그인).
- **hybrid-capability** -- 여러 capability 유형을 등록합니다(예: `openai`는 텍스트 추론, 음성, 미디어 이해, 이미지 생성을 담당함).
- **hook-only** -- 훅만 등록하며(typed 또는 custom), capabilities, 도구, 명령, 서비스는 등록하지 않습니다.
- **non-capability** -- 도구, 명령, 서비스, 라우트를 등록하지만 capabilities는 등록하지 않습니다.

플러그인의 형태와 capability 세부 구성을 보려면 `openclaw plugins inspect <id>`를 사용하세요. 자세한 내용은 [CLI reference](/cli/plugins#inspect)를 참조하세요.

### 레거시 훅

`before_agent_start` 훅은 hook-only 플러그인을 위한 호환성 경로로 계속 지원됩니다. 실제 레거시 플러그인들이 여전히 여기에 의존합니다.

방향성:

- 계속 동작하게 유지합니다.
- 레거시로 문서화합니다.
- 모델/제공자 override 작업에는 `before_model_resolve`를 우선합니다.
- 프롬프트 변경 작업에는 `before_prompt_build`를 우선합니다.
- 실제 사용량이 줄고 fixture 커버리지가 마이그레이션 안전성을 입증한 뒤에만 제거합니다.

### 호환성 신호

`openclaw doctor` 또는 `openclaw plugins inspect <id>`를 실행하면 다음 레이블 중 하나가 표시될 수 있습니다.

| Signal                     | 의미                                                         |
| -------------------------- | ------------------------------------------------------------ |
| **config valid**           | 구성이 정상적으로 파싱되고 플러그인이 올바르게 해석됨        |
| **compatibility advisory** | 플러그인이 지원되지만 더 오래된 패턴을 사용함(예: `hook-only`) |
| **legacy warning**         | 플러그인이 `before_agent_start`를 사용하고 있으며, 이는 deprecated됨 |
| **hard error**             | 구성이 잘못되었거나 플러그인 로드에 실패함                   |

현재 `hook-only`나 `before_agent_start` 때문에 플러그인이 깨지지는 않습니다. `hook-only`는 권고 수준이며, `before_agent_start`는 경고만 발생시킵니다. 이 신호들은 `openclaw status --all`과 `openclaw plugins doctor`에도 표시됩니다.

## 아키텍처 개요

OpenClaw의 플러그인 시스템은 네 개의 계층으로 구성됩니다.

1. **매니페스트 + 디스커버리**
   OpenClaw는 구성된 경로, 워크스페이스 루트, 전역 확장 루트, 번들 확장에서 후보 플러그인을 찾습니다. 디스커버리는 네이티브 `openclaw.plugin.json` 매니페스트와 지원되는 번들 매니페스트를 먼저 읽습니다.
2. **활성화 + 검증**
   코어는 디스커버리된 플러그인이 활성화, 비활성화, 차단, 또는 메모리 같은 독점 슬롯에 선택되었는지 결정합니다.
3. **런타임 로딩**
   네이티브 OpenClaw 플러그인은 jiti를 통해 프로세스 내부에서 로드되고 중앙 레지스트리에 capabilities를 등록합니다. 호환 가능한 번들은 런타임 코드를 import하지 않고도 레지스트리 레코드로 정규화됩니다.
4. **표면 소비**
   OpenClaw의 나머지 부분은 레지스트리를 읽어 도구, 채널, 제공자 설정, 훅, HTTP 라우트, CLI 명령, 서비스를 노출합니다.

특히 플러그인 CLI의 경우, 루트 명령 디스커버리는 두 단계로 나뉩니다.

- parse 시점 메타데이터는 `registerCli(..., { descriptors: [...] })`에서 옵니다.
- 실제 플러그인 CLI 모듈은 지연 로딩된 상태로 있다가 첫 호출 시 등록될 수 있습니다.

이렇게 하면 OpenClaw가 파싱 전에 루트 명령 이름을 예약하면서도 플러그인 소유 CLI 코드는 플러그인 내부에 유지할 수 있습니다.

중요한 설계 경계:

- 디스커버리 + config 검증은 플러그인 코드를 실행하지 않고 **매니페스트/스키마 메타데이터**만으로 동작해야 합니다.
- 네이티브 런타임 동작은 플러그인 모듈의 `register(api)` 경로에서 옵니다.

이 분리는 OpenClaw가 전체 런타임이 활성화되기 전에 config를 검증하고, 누락되거나 비활성화된 플러그인을 설명하고, UI/스키마 힌트를 구성할 수 있게 합니다.

### 채널 플러그인과 공유 message 도구

채널 플러그인은 일반적인 채팅 작업을 위해 별도의 send/edit/react 도구를 등록할 필요가 없습니다. OpenClaw는 코어에 하나의 공유 `message` 도구를 유지하고, 채널 플러그인은 그 뒤의 채널별 디스커버리와 실행을 담당합니다.

현재 경계는 다음과 같습니다.

- 코어는 공유 `message` 도구 호스트, 프롬프트 연결, 세션/스레드 bookkeeping, 실행 디스패치를 담당합니다.
- 채널 플러그인은 범위 지정된 액션 디스커버리, capability 디스커버리, 채널별 스키마 조각을 담당합니다.
- 채널 플러그인은 대화 ID가 스레드 ID를 인코딩하거나 상위 대화에서 상속하는 방식처럼, 제공자별 세션 대화 문법을 담당합니다.
- 채널 플러그인은 액션 어댑터를 통해 최종 액션을 실행합니다.

채널 플러그인의 SDK 표면은 `ChannelMessageActionAdapter.describeMessageTool(...)`입니다. 이 통합 디스커버리 호출을 통해 플러그인은 노출되는 액션, capabilities, 스키마 기여를 함께 반환할 수 있으므로 이 요소들이 서로 어긋나지 않습니다.

코어는 런타임 범위를 이 디스커버리 단계에 전달합니다. 중요한 필드는 다음과 같습니다.

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- 신뢰된 inbound `requesterSenderId`

이 점은 컨텍스트 민감형 플러그인에서 중요합니다. 채널은 코어 `message` 도구에 채널별 분기를 하드코딩하지 않고도, 활성 계정, 현재 룸/스레드/메시지, 또는 신뢰된 요청자 ID에 따라 메시지 액션을 숨기거나 노출할 수 있습니다.

그래서 embedded-runner 라우팅 변경은 여전히 플러그인 작업입니다. 러너는 현재 채팅/세션 ID를 플러그인 디스커버리 경계로 전달해, 공유 `message` 도구가 현재 턴에 맞는 채널 소유 표면을 노출하도록 해야 합니다.

채널 소유 실행 헬퍼의 경우, 번들 플러그인은 실행 런타임을 자체 확장 모듈 내부에 유지해야 합니다. 코어는 더 이상 `src/agents/tools` 아래의 Discord, Slack, Telegram, WhatsApp 메시지 액션 런타임을 소유하지 않습니다. 별도의 `plugin-sdk/*-action-runtime` 하위 경로도 공개하지 않으며, 번들 플러그인은 확장 소유 모듈에서 자체 로컬 런타임 코드를 직접 import해야 합니다.

같은 경계는 일반적인 provider 이름 기반 SDK seam에도 적용됩니다. 코어는 Slack, Discord, Signal, WhatsApp 같은 확장의 채널별 편의 barrel을 import해서는 안 됩니다. 코어에 어떤 동작이 필요하다면 번들 플러그인 자체의 `api.ts` / `runtime-api.ts` barrel을 사용하거나, 필요 사항을 공유 SDK의 좁고 일반적인 capability로 승격해야 합니다.

특히 poll의 경우 실행 경로는 두 가지입니다.

- `outbound.sendPoll`은 공통 poll 모델에 맞는 채널을 위한 공유 기준선입니다.
- `actions.handleAction("poll")`은 채널별 poll 의미론이나 추가 poll 파라미터가 있을 때 우선되는 경로입니다.

이제 코어는 플러그인 poll 디스패치가 액션을 거절한 뒤에야 공유 poll 파싱을 수행하므로, 플러그인 소유 poll 핸들러는 먼저 일반 poll 파서에 막히지 않고 채널별 poll 필드를 받을 수 있습니다.

전체 시작 시퀀스는 [로드 파이프라인](#load-pipeline)을 참조하세요.

## capability 소유권 모델

OpenClaw는 네이티브 플러그인을 관련 없는 통합 모음이 아니라 **회사** 또는 **기능**의 소유권 경계로 취급합니다.

즉, 다음을 의미합니다.

- 회사 플러그인은 일반적으로 해당 회사의 OpenClaw 표면을 모두 소유해야 합니다.
- 기능 플러그인은 일반적으로 자신이 도입한 전체 기능 표면을 소유해야 합니다.
- 채널은 제공자 동작을 임시로 재구현하기보다 공유 코어 capabilities를 소비해야 합니다.

예시:

- 번들 `openai` 플러그인은 OpenAI 모델 제공자 동작과 OpenAI 음성 + 실시간 음성 + 미디어 이해 + 이미지 생성 동작을 소유합니다.
- 번들 `elevenlabs` 플러그인은 ElevenLabs 음성 동작을 소유합니다.
- 번들 `microsoft` 플러그인은 Microsoft 음성 동작을 소유합니다.
- 번들 `google` 플러그인은 Google 모델 제공자 동작과 Google 미디어 이해 + 이미지 생성 + 웹 검색 동작을 소유합니다.
- 번들 `firecrawl` 플러그인은 Firecrawl 웹 가져오기 동작을 소유합니다.
- 번들 `minimax`, `mistral`, `moonshot`, `zai` 플러그인은 미디어 이해 백엔드를 소유합니다.
- 번들 `qwen` 플러그인은 Qwen 텍스트 제공자 동작과 미디어 이해 및 비디오 생성 동작을 소유합니다.
- `voice-call` 플러그인은 기능 플러그인입니다. 통화 전송, 도구, CLI, 라우트, Twilio 미디어 스트림 브리징을 소유하지만, 벤더 플러그인을 직접 import하는 대신 공유 음성 + 실시간 전사 + 실시간 음성 capabilities를 소비합니다.

의도된 최종 상태는 다음과 같습니다.

- OpenAI가 텍스트 모델, 음성, 이미지, 미래의 비디오까지 포괄하더라도 하나의 플러그인 안에 존재합니다.
- 다른 벤더도 자기 표면 전체를 같은 방식으로 소유할 수 있습니다.
- 채널은 어떤 벤더 플러그인이 제공자를 소유하는지 신경 쓰지 않고, 코어가 노출하는 공유 capability 계약을 소비합니다.

이것이 핵심 구분입니다.

- **plugin** = 소유권 경계
- **capability** = 여러 플러그인이 구현하거나 소비할 수 있는 코어 계약

따라서 OpenClaw가 비디오 같은 새 도메인을 추가할 때 첫 질문은 "어떤 제공자에 비디오 처리를 하드코딩해야 하는가?"가 아닙니다. 첫 질문은 "코어 비디오 capability 계약이 무엇인가?"입니다. 그 계약이 존재하면, 벤더 플러그인이 여기에 등록하고 채널/기능 플러그인이 이를 소비할 수 있습니다.

아직 capability가 없다면 보통 올바른 순서는 다음과 같습니다.

1. 코어에 누락된 capability를 정의합니다.
2. 플러그인 API/런타임을 통해 이를 typed 방식으로 노출합니다.
3. 채널/기능을 그 capability에 연결합니다.
4. 벤더 플러그인이 구현체를 등록하도록 합니다.

이렇게 하면 소유권은 명확하게 유지하면서도, 단일 벤더나 일회성 플러그인별 코드 경로에 의존하는 코어 동작을 피할 수 있습니다.

### capability 계층화

코드가 어디에 속해야 하는지 결정할 때 다음 사고 모델을 사용하세요.

- **코어 capability 계층**: 공유 orchestration, 정책, fallback, config 병합 규칙, 전달 의미론, typed 계약
- **벤더 플러그인 계층**: 벤더별 API, 인증, 모델 카탈로그, 음성 합성, 이미지 생성, 미래의 비디오 백엔드, 사용량 엔드포인트
- **채널/기능 플러그인 계층**: 코어 capabilities를 소비하고 이를 표면에 노출하는 Slack/Discord/voice-call 등의 통합

예를 들어 TTS는 다음 구조를 따릅니다.

- 코어는 응답 시점 TTS 정책, fallback 순서, 환경설정, 채널 전달을 소유합니다.
- `openai`, `elevenlabs`, `microsoft`는 합성 구현을 소유합니다.
- `voice-call`은 전화용 TTS 런타임 헬퍼를 소비합니다.

이 패턴은 미래의 capabilities에도 동일하게 적용하는 것이 바람직합니다.

### 다중 capability 회사 플러그인 예시

회사 플러그인은 외부에서 볼 때 일관된 하나의 단위처럼 느껴져야 합니다. OpenClaw에 모델, 음성, 실시간 전사, 실시간 음성, 미디어 이해, 이미지 생성, 비디오 생성, 웹 가져오기, 웹 검색을 위한 공유 계약이 있다면, 벤더는 이 모든 표면을 한 곳에서 소유할 수 있습니다.

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

중요한 것은 정확한 헬퍼 이름이 아닙니다. 중요한 것은 구조입니다.

- 하나의 플러그인이 벤더 표면을 소유합니다.
- 코어는 여전히 capability 계약을 소유합니다.
- 채널과 기능 플러그인은 벤더 코드가 아니라 `api.runtime.*` 헬퍼를 소비합니다.
- 계약 테스트는 플러그인이 자신이 소유한다고 주장하는 capabilities를 실제로 등록했는지 검증할 수 있습니다.

### capability 예시: 비디오 이해

OpenClaw는 이미 이미지/오디오/비디오 이해를 하나의 공유 capability로 취급합니다. 동일한 소유권 모델이 여기에 적용됩니다.

1. 코어가 media-understanding 계약을 정의합니다.
2. 벤더 플러그인은 해당되는 경우 `describeImage`, `transcribeAudio`, `describeVideo`를 등록합니다.
3. 채널과 기능 플러그인은 벤더 코드에 직접 연결하지 않고 공유 코어 동작을 소비합니다.

이렇게 하면 특정 제공자의 비디오 가정을 코어에 굳혀 넣지 않게 됩니다. 플러그인은 벤더 표면을 소유하고, 코어는 capability 계약과 fallback 동작을 소유합니다.

비디오 생성도 이미 같은 순서를 따릅니다. 코어는 typed capability 계약과 런타임 헬퍼를 소유하고, 벤더 플러그인은 여기에 대해 `api.registerVideoGenerationProvider(...)` 구현을 등록합니다.

구체적인 rollout 체크리스트가 필요하다면 [Capability Cookbook](/ko/plugins/architecture)를 참조하세요.

## 계약과 강제

플러그인 API 표면은 의도적으로 `OpenClawPluginApi`에 typed되고 중앙 집중화되어 있습니다. 이 계약은 지원되는 등록 지점과 플러그인이 의존할 수 있는 런타임 헬퍼를 정의합니다.

이것이 중요한 이유:

- 플러그인 작성자는 하나의 안정적인 내부 표준을 얻게 됩니다.
- 코어는 두 플러그인이 같은 provider ID를 등록하는 중복 소유권을 거부할 수 있습니다.
- 시작 시 잘못된 등록에 대해 실행 가능한 진단을 표시할 수 있습니다.
- 계약 테스트는 번들 플러그인 소유권을 강제하고 조용한 드리프트를 방지할 수 있습니다.

강제에는 두 계층이 있습니다.

1. **런타임 등록 강제**
   플러그인 레지스트리는 플러그인이 로드되면서 등록을 검증합니다. 예를 들어 중복 provider ID, 중복 음성 provider ID, 잘못된 등록은 정의되지 않은 동작이 아니라 플러그인 진단으로 처리됩니다.
2. **계약 테스트**
   번들 플러그인은 테스트 실행 중 계약 레지스트리에 캡처되므로, OpenClaw가 소유권을 명시적으로 검증할 수 있습니다. 현재는 모델 제공자, 음성 제공자, 웹 검색 제공자, 번들 등록 소유권에 대해 사용됩니다.

실질적인 효과는 OpenClaw가 어떤 플러그인이 어떤 표면을 소유하는지 미리 알고 있다는 점입니다. 덕분에 소유권이 암묵적이지 않고 선언적이며 typed되고 테스트 가능하므로, 코어와 채널이 자연스럽게 조합될 수 있습니다.

### 계약에 포함되어야 하는 것

좋은 플러그인 계약은 다음과 같습니다.

- typed
- 작음
- capability별로 분리됨
- 코어가 소유함
- 여러 플러그인이 재사용 가능함
- 채널/기능이 벤더 지식 없이 소비 가능함

좋지 않은 플러그인 계약은 다음과 같습니다.

- 코어에 숨겨진 벤더별 정책
- 레지스트리를 우회하는 일회성 플러그인 탈출구
- 벤더 구현에 직접 접근하는 채널 코드
- `OpenClawPluginApi` 또는 `api.runtime`의 일부가 아닌 임시 런타임 객체

헷갈릴 때는 추상화 수준을 올리세요. 먼저 capability를 정의하고, 그다음 플러그인이 여기에 연결되게 하세요.

## 실행 모델

네이티브 OpenClaw 플러그인은 Gateway와 **같은 프로세스 내부**에서 실행됩니다. 샌드박스되지 않습니다. 로드된 네이티브 플러그인은 코어 코드와 같은 프로세스 수준 신뢰 경계를 가집니다.

의미하는 바:

- 네이티브 플러그인은 도구, 네트워크 핸들러, 훅, 서비스를 등록할 수 있습니다.
- 네이티브 플러그인 버그는 gateway를 크래시시키거나 불안정하게 만들 수 있습니다.
- 악의적인 네이티브 플러그인은 OpenClaw 프로세스 내부의 임의 코드 실행과 동등합니다.

호환 가능한 번들은 OpenClaw가 현재 이를 메타데이터/콘텐츠 팩으로 취급하므로 기본적으로 더 안전합니다. 현재 릴리스에서는 대체로 번들 Skills가 이에 해당합니다.

번들되지 않은 플러그인에는 allowlist와 명시적 install/load 경로를 사용하세요. 워크스페이스 플러그인은 프로덕션 기본값이 아니라 개발 시점 코드로 취급하세요.

번들 워크스페이스 패키지 이름의 경우, 플러그인 ID를 npm 이름에 고정하세요. 기본적으로 `@openclaw/<id>`를 사용하거나, 패키지가 더 좁은 플러그인 역할을 의도적으로 노출할 때는 승인된 typed 접미사인 `-provider`, `-plugin`, `-speech`, `-sandbox`, `-media-understanding`를 사용합니다.

중요한 신뢰 메모:

- `plugins.allow`는 소스 출처가 아니라 **플러그인 ID**를 신뢰합니다.
- 번들 플러그인과 같은 ID를 가진 워크스페이스 플러그인이 활성화되거나 allowlist에 들어가면 의도적으로 번들 사본을 가립니다.
- 이는 로컬 개발, 패치 테스트, 핫픽스에 유용한 정상 동작입니다.

## export 경계

OpenClaw는 구현 편의성이 아니라 capabilities를 export합니다.

capability 등록은 공개 상태로 유지하고, 계약이 아닌 헬퍼 export는 줄이세요.

- 번들 플러그인별 헬퍼 하위 경로
- 공개 API로 의도되지 않은 런타임 plumbing 하위 경로
- 벤더별 편의 헬퍼
- 구현 세부 사항인 setup/onboarding 헬퍼

일부 번들 플러그인 헬퍼 하위 경로는 호환성과 번들 플러그인 유지보수를 위해 생성된 SDK export 맵에 여전히 남아 있습니다. 현재 예로는 `plugin-sdk/feishu`, `plugin-sdk/feishu-setup`, `plugin-sdk/zalo`, `plugin-sdk/zalo-setup`, 그리고 여러 `plugin-sdk/matrix*` seam이 있습니다. 이를 새 서드파티 플러그인을 위한 권장 SDK 패턴이 아니라, 예약된 구현 세부 export로 취급하세요.

## 로드 파이프라인

시작 시 OpenClaw는 대략 다음을 수행합니다.

1. 후보 플러그인 루트를 디스커버리합니다.
2. 네이티브 또는 호환 번들 매니페스트와 패키지 메타데이터를 읽습니다.
3. 안전하지 않은 후보를 거부합니다.
4. 플러그인 config를 정규화합니다(`plugins.enabled`, `allow`, `deny`, `entries`, `slots`, `load.paths`).
5. 각 후보의 활성화 여부를 결정합니다.
6. 활성화된 네이티브 모듈을 jiti로 로드합니다.
7. 네이티브 `register(api)`(또는 레거시 별칭인 `activate(api)`) 훅을 호출하고 등록 내용을 플러그인 레지스트리에 수집합니다.
8. 레지스트리를 명령/런타임 표면에 노출합니다.

<Note>
`activate`는 `register`의 레거시 별칭입니다. 로더는 존재하는 쪽(`def.register ?? def.activate`)을 해석하여 같은 시점에 호출합니다. 모든 번들 플러그인은 `register`를 사용합니다. 새 플러그인에는 `register`를 우선하세요.
</Note>

안전 게이트는 런타임 실행 **이전**에 발생합니다. 엔트리가 플러그인 루트 밖으로 벗어나거나, 경로가 world-writable이거나, 번들되지 않은 플러그인에 대해 경로 소유권이 의심스러운 경우 후보는 차단됩니다.

### 매니페스트 우선 동작

매니페스트는 제어 평면의 기준 source of truth입니다. OpenClaw는 이를 사용해 다음을 수행합니다.

- 플러그인을 식별합니다.
- 선언된 채널/Skills/config 스키마 또는 번들 capability를 디스커버리합니다.
- `plugins.entries.<id>.config`를 검증합니다.
- Control UI 레이블/placeholder를 보강합니다.
- 설치/카탈로그 메타데이터를 표시합니다.

네이티브 플러그인의 경우 런타임 모듈은 데이터 평면 부분입니다. 여기서 훅, 도구, 명령, 또는 provider 흐름 같은 실제 동작을 등록합니다.

### 로더가 캐시하는 것

OpenClaw는 다음에 대해 짧은 프로세스 내 캐시를 유지합니다.

- 디스커버리 결과
- 매니페스트 레지스트리 데이터
- 로드된 플러그인 레지스트리

이 캐시들은 급격한 시작 부하와 반복 명령 오버헤드를 줄여 줍니다. 이를 지속성이 아니라 짧게 유지되는 성능 캐시로 생각하면 됩니다.

성능 메모:

- 캐시를 비활성화하려면 `OPENCLAW_DISABLE_PLUGIN_DISCOVERY_CACHE=1` 또는 `OPENCLAW_DISABLE_PLUGIN_MANIFEST_CACHE=1`을 설정하세요.
- 캐시 기간은 `OPENCLAW_PLUGIN_DISCOVERY_CACHE_MS`와 `OPENCLAW_PLUGIN_MANIFEST_CACHE_MS`로 조정합니다.

## 레지스트리 모델

로드된 플러그인은 임의의 코어 전역 상태를 직접 변경하지 않습니다. 중앙 플러그인 레지스트리에 등록합니다.

레지스트리는 다음을 추적합니다.

- 플러그인 레코드(ID, 소스, origin, 상태, 진단)
- 도구
- 레거시 훅과 typed 훅
- 채널
- 제공자
- gateway RPC 핸들러
- HTTP 라우트
- CLI registrar
- 백그라운드 서비스
- 플러그인 소유 명령

그다음 코어 기능은 플러그인 모듈과 직접 대화하는 대신 이 레지스트리를 읽습니다. 이로써 로딩은 단방향으로 유지됩니다.

- 플러그인 모듈 -> 레지스트리 등록
- 코어 런타임 -> 레지스트리 소비

이 분리는 유지보수성에 중요합니다. 대부분의 코어 표면은 "모든 플러그인 모듈을 특수 처리"하는 대신 "레지스트리를 읽기"라는 하나의 통합 지점만 가지면 되기 때문입니다.

## 대화 바인딩 콜백

대화를 바인딩하는 플러그인은 승인이 해결될 때 반응할 수 있습니다.

바인드 요청이 승인되거나 거부된 후 콜백을 받으려면 `api.onConversationBindingResolved(...)`를 사용하세요.

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
- `decision`: `"allow-once"`, `"allow-always"`, 또는 `"deny"`
- `binding`: 승인된 요청에 대한 해결된 바인딩
- `request`: 원래 요청 요약, detach 힌트, 발신자 ID, 대화 메타데이터

이 콜백은 알림 전용입니다. 누가 대화를 바인딩할 수 있는지는 변경하지 않으며, 코어 승인 처리가 끝난 뒤 실행됩니다.

## 제공자 런타임 훅

이제 제공자 플러그인에는 두 계층이 있습니다.

- 매니페스트 메타데이터: 런타임 로드 전 저비용 provider env-auth 조회를 위한 `providerAuthEnvVars`, 인증을 공유하는 provider variant를 위한 `providerAuthAliases`, 런타임 로드 전 저비용 채널 env/setup 조회를 위한 `channelEnvVars`, 그리고 런타임 로드 전 저비용 onboarding/auth-choice 레이블 및 CLI 플래그 메타데이터를 위한 `providerAuthChoices`
- config 시점 훅: `catalog` / 레거시 `discovery`, 그리고 `applyConfigDefaults`
- 런타임 훅: `normalizeModelId`, `normalizeTransport`,
  `normalizeConfig`,
  `applyNativeStreamingUsageCompat`, `resolveConfigApiKey`,
  `resolveSyntheticAuth`, `resolveExternalAuthProfiles`,
  `shouldDeferSyntheticProfileAuth`,
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

OpenClaw는 여전히 일반적인 에이전트 루프, failover, transcript 처리, 도구 정책을 소유합니다. 이 훅들은 전체 custom 추론 transport 없이도 제공자별 동작을 확장할 수 있는 표면입니다.

제공자에 env 기반 자격 증명이 있어 일반 auth/status/model-picker 경로가 플러그인 런타임을 로드하지 않고도 이를 알아야 한다면 매니페스트 `providerAuthEnvVars`를 사용하세요. 하나의 provider ID가 다른 provider ID의 env vars, auth profiles, config 기반 auth, API-key onboarding 선택지를 재사용해야 한다면 매니페스트 `providerAuthAliases`를 사용하세요. onboarding/auth-choice CLI 표면이 플러그인 런타임을 로드하지 않고도 provider의 choice ID, 그룹 레이블, 단일 플래그 인증 연결을 알아야 한다면 매니페스트 `providerAuthChoices`를 사용하세요. 제공자 런타임 `envVars`는 operator-facing 힌트(예: onboarding 레이블 또는 OAuth client-id/client-secret 설정 변수)용으로 유지하세요.

채널에 env 기반 auth 또는 setup이 있어 일반 shell-env fallback, config/status 검사, setup 프롬프트가 채널 런타임을 로드하지 않고도 이를 알아야 한다면 매니페스트 `channelEnvVars`를 사용하세요.

### 훅 순서와 사용법

모델/제공자 플러그인의 경우 OpenClaw는 대략 다음 순서로 훅을 호출합니다.
"사용 시점" 열은 빠른 판단 가이드입니다.

| #   | Hook                              | 역할                                                                                                           | 사용 시점                                                                                                                                   |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `catalog`                         | `models.json` 생성 중 provider 구성을 `models.providers`에 게시                                               | 제공자가 카탈로그 또는 base URL 기본값을 소유할 때                                                                                          |
| 2   | `applyConfigDefaults`             | config materialization 중 provider 소유 전역 config 기본값을 적용                                              | 기본값이 auth 모드, env, 또는 provider 모델 패밀리 의미론에 따라 달라질 때                                                                 |
| --  | _(내장 모델 조회)_                | OpenClaw가 먼저 일반 레지스트리/카탈로그 경로를 시도                                                          | _(플러그인 훅 아님)_                                                                                                                        |
| 3   | `normalizeModelId`                | 조회 전에 레거시 또는 preview model-id 별칭을 정규화                                                           | 제공자가 정식 모델 해석 전에 별칭 정리를 소유할 때                                                                                          |
| 4   | `normalizeTransport`              | 일반 모델 조립 전에 provider 패밀리의 `api` / `baseUrl`을 정규화                                               | 제공자가 같은 transport 패밀리 내 custom provider ID의 transport 정리를 소유할 때                                                           |
| 5   | `normalizeConfig`                 | 런타임/provider 해석 전에 `models.providers.<id>`를 정규화                                                     | 플러그인에 속해야 하는 config 정리가 필요할 때; 번들 Google 계열 헬퍼도 지원되는 Google config 엔트리를 보완함                            |
| 6   | `applyNativeStreamingUsageCompat` | config providers에 native streaming-usage 호환 재작성 적용                                                     | 제공자가 엔드포인트 기반 native streaming usage 메타데이터 수정을 필요로 할 때                                                              |
| 7   | `resolveConfigApiKey`             | 런타임 auth 로드 전에 config providers의 env-marker auth를 해석                                                | 제공자가 자체 env-marker API-key 해석을 소유할 때; `amazon-bedrock`에는 내장 AWS env-marker resolver도 여기 포함됨                        |
| 8   | `resolveSyntheticAuth`            | 평문을 저장하지 않고 로컬/self-hosted 또는 config 기반 auth를 표면화                                            | 제공자가 synthetic/local 자격 증명 marker로 동작할 수 있을 때                                                                               |
| 9   | `resolveExternalAuthProfiles`     | 제공자 소유 외부 auth profile을 overlay; 기본 `persistence`는 CLI/앱 소유 자격 증명에 대해 `runtime-only`    | 제공자가 copied refresh token을 저장하지 않고 외부 auth 자격 증명을 재사용할 때                                                             |
| 10  | `shouldDeferSyntheticProfileAuth` | 저장된 synthetic profile placeholder를 env/config 기반 auth 뒤로 낮춤                                           | 제공자가 우선 순위를 가져서는 안 되는 synthetic placeholder profile을 저장할 때                                                              |
| 11  | `resolveDynamicModel`             | 아직 로컬 레지스트리에 없는 provider 소유 model ID에 대한 동기 fallback                                        | 제공자가 임의의 업스트림 model ID를 허용할 때                                                                                               |
| 12  | `prepareDynamicModel`             | 비동기 워밍업 후 `resolveDynamicModel`를 다시 실행                                                              | 제공자가 알 수 없는 ID를 해석하기 전에 네트워크 메타데이터를 필요로 할 때                                                                   |
| 13  | `normalizeResolvedModel`          | embedded runner가 해석된 모델을 사용하기 전 최종 재작성                                                        | 제공자가 transport 재작성이 필요하지만 여전히 코어 transport를 사용할 때                                                                     |
| 14  | `contributeResolvedModelCompat`   | 다른 호환 transport 뒤에 있는 벤더 모델에 대한 compat 플래그 기여                                              | 제공자가 provider를 직접 가져가지 않고도 proxy transport에서 자기 모델을 인식할 때                                                          |
| 15  | `capabilities`                    | 공유 코어 로직이 사용하는 제공자 소유 transcript/tooling 메타데이터                                            | 제공자가 transcript/provider-family 특이사항을 필요로 할 때                                                                                 |
| 16  | `normalizeToolSchemas`            | embedded runner가 보기 전에 도구 스키마를 정규화                                                               | 제공자가 transport-family 스키마 정리를 필요로 할 때                                                                                        |
| 17  | `inspectToolSchemas`              | 정규화 후 제공자 소유 스키마 진단 노출                                                                         | 제공자가 코어에 provider별 규칙을 가르치지 않고 키워드 경고를 표시하고 싶을 때                                                              |
| 18  | `resolveReasoningOutputMode`      | native vs tagged reasoning-output 계약 선택                                                                    | 제공자가 native 필드 대신 tagged reasoning/final output을 필요로 할 때                                                                       |
| 19  | `prepareExtraParams`              | 일반 스트림 옵션 wrapper 전에 요청 파라미터 정규화                                                             | 제공자가 기본 요청 파라미터 또는 provider별 파라미터 정리를 필요로 할 때                                                                     |
| 20  | `createStreamFn`                  | 일반 스트림 경로를 완전히 교체하여 custom transport 사용                                                       | 제공자가 wrapper만이 아니라 custom wire protocol을 필요로 할 때                                                                              |
| 21  | `wrapStreamFn`                    | 일반 wrapper 적용 후 스트림 wrapper                                                                             | custom transport 없이 요청 header/body/model compat wrapper가 필요할 때                                                                      |
| 22  | `resolveTransportTurnState`       | native 턴별 transport header 또는 메타데이터 연결                                                              | 제공자가 일반 transport가 provider-native 턴 ID를 전송하길 원할 때                                                                           |
| 23  | `resolveWebSocketSessionPolicy`   | native WebSocket header 또는 세션 cool-down 정책 연결                                                          | 제공자가 일반 WS transport에서 세션 header 또는 fallback 정책 조정을 원할 때                                                                 |
| 24  | `formatApiKey`                    | auth-profile formatter: 저장된 profile을 런타임 `apiKey` 문자열로 변환                                         | 제공자가 추가 auth 메타데이터를 저장하고 custom 런타임 토큰 형식이 필요할 때                                                                 |
| 25  | `refreshOAuth`                    | custom refresh 엔드포인트 또는 refresh 실패 정책을 위한 OAuth refresh override                                 | 제공자가 공유 `pi-ai` refresher에 맞지 않을 때                                                                                               |
| 26  | `buildAuthDoctorHint`             | OAuth refresh 실패 시 추가되는 복구 힌트                                                                       | 제공자가 refresh 실패 후 제공자 소유 auth 복구 가이드를 필요로 할 때                                                                          |
| 27  | `matchesContextOverflowError`     | 제공자 소유 컨텍스트 윈도 overflow matcher                                                                     | 제공자에 일반 heuristic이 놓치는 raw overflow 오류가 있을 때                                                                                 |
| 28  | `classifyFailoverReason`          | 제공자 소유 failover 이유 분류                                                                                 | 제공자가 raw API/transport 오류를 rate-limit/overload 등으로 매핑할 수 있을 때                                                               |
| 29  | `isCacheTtlEligible`              | proxy/backhaul provider를 위한 프롬프트 캐시 정책                                                              | 제공자가 proxy별 캐시 TTL 게이팅을 필요로 할 때                                                                                              |
| 30  | `buildMissingAuthMessage`         | 일반 missing-auth 복구 메시지를 대체                                                                           | 제공자가 제공자별 missing-auth 복구 힌트를 필요로 할 때                                                                                      |
| 31  | `suppressBuiltInModel`            | 오래된 업스트림 모델 숨김과 선택적 사용자 대상 오류 힌트                                                       | 제공자가 오래된 업스트림 행을 숨기거나 벤더 힌트로 대체해야 할 때                                                                            |
| 32  | `augmentModelCatalog`             | 디스커버리 후 synthetic/final 카탈로그 행 추가                                                                 | 제공자가 `models list`와 picker에 synthetic forward-compat 행을 필요로 할 때                                                                 |
| 33  | `isBinaryThinking`                | binary-thinking provider를 위한 on/off reasoning 토글                                                          | 제공자가 binary thinking on/off만 노출할 때                                                                                                  |
| 34  | `supportsXHighThinking`           | 선택된 모델에 대한 `xhigh` reasoning 지원                                                                      | 제공자가 일부 모델에만 `xhigh`를 원할 때                                                                                                     |
| 35  | `resolveDefaultThinkingLevel`     | 특정 모델 패밀리의 기본 `/think` 레벨 해석                                                                     | 제공자가 모델 패밀리의 기본 `/think` 정책을 소유할 때                                                                                        |
| 36  | `isModernModelRef`                | 라이브 profile 필터 및 smoke 선택을 위한 modern-model matcher                                                  | 제공자가 live/smoke 선호 모델 매칭을 소유할 때                                                                                               |
| 37  | `prepareRuntimeAuth`              | 추론 직전 구성된 자격 증명을 실제 런타임 토큰/키로 교환                                                       | 제공자가 토큰 교환 또는 짧은 수명의 요청 자격 증명을 필요로 할 때                                                                            |
| 38  | `resolveUsageAuth`                | `/usage` 및 관련 상태 표면을 위한 사용량/과금 자격 증명 해석                                                  | 제공자가 custom usage/quota 토큰 파싱 또는 다른 usage 자격 증명을 필요로 할 때                                                               |
| 39  | `fetchUsageSnapshot`              | auth 해석 후 제공자별 사용량/quota 스냅샷 조회 및 정규화                                                       | 제공자가 제공자별 usage 엔드포인트 또는 payload parser를 필요로 할 때                                                                        |
| 40  | `createEmbeddingProvider`         | 메모리/검색용 제공자 소유 임베딩 어댑터 생성                                                                   | 메모리 임베딩 동작이 제공자 플러그인과 함께 있어야 할 때                                                                                    |
| 41  | `buildReplayPolicy`               | 제공자의 transcript 처리 방식을 제어하는 replay 정책 반환                                                     | 제공자가 custom transcript 정책(예: thinking 블록 제거)을 필요로 할 때                                                                       |
| 42  | `sanitizeReplayHistory`           | 일반 transcript 정리 후 replay 기록 재작성                                                                     | 제공자가 공유 compaction 헬퍼를 넘어서는 replay 재작성을 필요로 할 때                                                                        |
| 43  | `validateReplayTurns`             | embedded runner 직전 최종 replay 턴 검증 또는 형태 조정                                                       | 제공자 transport가 일반 정리 이후 더 엄격한 턴 검증을 필요로 할 때                                                                           |
| 44  | `onModelSelected`                 | 모델이 활성화될 때 제공자 소유 후속 효과 실행                                                                  | 제공자가 telemetry 또는 제공자 소유 상태 처리를 필요로 할 때                                                                                 |

`normalizeModelId`, `normalizeTransport`, `normalizeConfig`는 먼저 일치하는 provider 플러그인을 확인한 다음, 실제로 model ID나 transport/config를 변경하는 플러그인이 나올 때까지 다른 hook-capable provider 플러그인으로 넘깁니다. 이렇게 하면 호출자가 어느 번들 플러그인이 재작성을 소유하는지 몰라도 alias/compat provider shim이 동작합니다. 어떤 provider 훅도 지원되는 Google 계열 config 엔트리를 재작성하지 않으면, 번들 Google config normalizer가 여전히 그 호환성 정리를 적용합니다.

제공자에 완전히 custom wire protocol 또는 custom 요청 실행기가 필요하다면, 이는 다른 종류의 확장입니다. 이 훅들은 OpenClaw의 일반 추론 루프에서 여전히 실행되는 provider 동작을 위한 것입니다.

### 제공자 예시

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
  `wrapStreamFn`을 사용합니다. Claude 4.6 forward-compat, provider-family 힌트, auth 복구 가이드, usage 엔드포인트 통합, 프롬프트 캐시 적격성, auth 인지 config 기본값, Claude 기본/적응형 thinking 정책, beta headers, `/fast` / `serviceTier`, `context1m`을 위한 Anthropic별 스트림 shaping을 소유하기 때문입니다.
- Anthropic의 Claude 전용 스트림 헬퍼는 당분간 번들 플러그인 자체의 공개 `api.ts` / `contract-api.ts` seam에 유지됩니다. 이 패키지 표면은 한 provider의 beta-header 규칙을 위해 일반 SDK를 확장하지 않고, `wrapAnthropicProviderStream`, `resolveAnthropicBetas`, `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`, 하위 수준 Anthropic wrapper builder를 export합니다.
- OpenAI는 `resolveDynamicModel`, `normalizeResolvedModel`, `capabilities`와 함께 `buildMissingAuthMessage`, `suppressBuiltInModel`, `augmentModelCatalog`, `supportsXHighThinking`, `isModernModelRef`를 사용합니다. GPT-5.4 forward-compat, 직접 OpenAI의 `openai-completions` -> `openai-responses` 정규화, Codex 인지 auth 힌트, Spark 숨김, synthetic OpenAI 목록 행, GPT-5 thinking / live-model 정책을 소유하기 때문입니다. `openai-responses-defaults` 스트림 패밀리는 attribution headers, `/fast`/`serviceTier`, 텍스트 verbosity, native Codex 웹 검색, reasoning-compat payload shaping, Responses context 관리를 위한 공유 native OpenAI Responses wrapper를 소유합니다.
- OpenRouter는 `catalog`와 함께 `resolveDynamicModel`, `prepareDynamicModel`을 사용합니다. 이 provider는 pass-through이어서 OpenClaw의 정적 카탈로그가 업데이트되기 전에 새로운 model ID를 노출할 수 있기 때문입니다. 또한 provider별 요청 headers, 라우팅 메타데이터, reasoning 패치, 프롬프트 캐시 정책을 코어 밖에 유지하기 위해 `capabilities`, `wrapStreamFn`, `isCacheTtlEligible`를 사용합니다. replay 정책은 `passthrough-gemini` 패밀리에서 오고, `openrouter-thinking` 스트림 패밀리는 proxy reasoning 주입과 미지원 모델 / `auto` 건너뛰기를 소유합니다.
- GitHub Copilot은 `catalog`, `auth`, `resolveDynamicModel`, `capabilities`와 함께 `prepareRuntimeAuth`, `fetchUsageSnapshot`을 사용합니다. provider 소유 device login, 모델 fallback 동작, Claude transcript 특이사항, GitHub token -> Copilot token 교환, provider 소유 usage 엔드포인트가 필요하기 때문입니다.
- OpenAI Codex는 `catalog`, `resolveDynamicModel`,
  `normalizeResolvedModel`, `refreshOAuth`, `augmentModelCatalog`와 함께
  `prepareExtraParams`, `resolveUsageAuth`, `fetchUsageSnapshot`을 사용합니다. 여전히 코어 OpenAI transport 위에서 실행되지만 transport/base URL 정규화, OAuth refresh fallback 정책, 기본 transport 선택, synthetic Codex 카탈로그 행, ChatGPT usage 엔드포인트 통합을 소유하기 때문입니다. direct OpenAI와 같은 `openai-responses-defaults` 스트림 패밀리를 공유합니다.
- Google AI Studio와 Gemini CLI OAuth는 `resolveDynamicModel`,
  `buildReplayPolicy`, `sanitizeReplayHistory`,
  `resolveReasoningOutputMode`, `wrapStreamFn`, `isModernModelRef`를 사용합니다. `google-gemini` replay 패밀리가 Gemini 3.1 forward-compat fallback, native Gemini replay 검증, bootstrap replay sanitation, tagged reasoning-output 모드, modern-model 매칭을 소유하고, `google-thinking` 스트림 패밀리가 Gemini thinking payload 정규화를 소유하기 때문입니다. Gemini CLI OAuth는 token formatting, token parsing, quota 엔드포인트 연결을 위해 `formatApiKey`, `resolveUsageAuth`, `fetchUsageSnapshot`도 사용합니다.
- Anthropic Vertex는 `anthropic-by-model` replay 패밀리를 통해 `buildReplayPolicy`를 사용하므로, Claude별 replay 정리는 모든 `anthropic-messages` transport가 아니라 Claude ID에만 적용됩니다.
- Amazon Bedrock은 `buildReplayPolicy`, `matchesContextOverflowError`,
  `classifyFailoverReason`, `resolveDefaultThinkingLevel`을 사용합니다. Anthropic-on-Bedrock 트래픽에 대한 Bedrock별 throttle/not-ready/context-overflow 오류 분류를 소유하기 때문입니다. replay 정책은 여전히 같은 Claude 전용 `anthropic-by-model` 가드를 공유합니다.
- OpenRouter, Kilocode, Opencode, Opencode Go는 `passthrough-gemini` replay 패밀리를 통해 `buildReplayPolicy`를 사용합니다. OpenAI 호환 transport를 통해 Gemini 모델을 프록시하고 native Gemini replay 검증이나 bootstrap 재작성 없이 Gemini thought-signature sanitation이 필요하기 때문입니다.
- MiniMax는 `hybrid-anthropic-openai` replay 패밀리를 통해 `buildReplayPolicy`를 사용합니다. 하나의 provider가 Anthropic-message와 OpenAI 호환 의미론을 모두 소유하기 때문입니다. Anthropic 쪽에서는 Claude 전용 thinking-block 제거를 유지하면서 reasoning output 모드를 다시 native로 override하고, `minimax-fast-mode` 스트림 패밀리는 공유 스트림 경로에서 fast-mode 모델 재작성을 소유합니다.
- Moonshot은 `catalog`와 `wrapStreamFn`을 사용합니다. 여전히 공유 OpenAI transport를 사용하지만 provider 소유 thinking payload 정규화가 필요하기 때문입니다. `moonshot-thinking` 스트림 패밀리는 config와 `/think` 상태를 native binary thinking payload에 매핑합니다.
- Kilocode는 `catalog`, `capabilities`, `wrapStreamFn`,
  `isCacheTtlEligible`를 사용합니다. provider 소유 요청 headers, reasoning payload 정규화, Gemini transcript 힌트, Anthropic cache-TTL 게이팅이 필요하기 때문입니다. `kilocode-thinking` 스트림 패밀리는 공유 proxy 스트림 경로에서 Kilo thinking 주입을 유지하면서 명시적 reasoning payload를 지원하지 않는 `kilo/auto` 및 기타 proxy model ID를 건너뜁니다.
- Z.AI는 `resolveDynamicModel`, `prepareExtraParams`, `wrapStreamFn`,
  `isCacheTtlEligible`, `isBinaryThinking`, `isModernModelRef`,
  `resolveUsageAuth`, `fetchUsageSnapshot`을 사용합니다. GLM-5 fallback, `tool_stream` 기본값, binary thinking UX, modern-model 매칭, usage auth + quota 조회를 모두 소유하기 때문입니다. `tool-stream-default-on` 스트림 패밀리는 기본 활성 `tool_stream` wrapper를 provider별 수기 연결 코드 밖에 유지합니다.
- xAI는 `normalizeResolvedModel`, `normalizeTransport`,
  `contributeResolvedModelCompat`, `prepareExtraParams`, `wrapStreamFn`,
  `resolveSyntheticAuth`, `resolveDynamicModel`, `isModernModelRef`를 사용합니다. native xAI Responses transport 정규화, Grok fast-mode alias 재작성, 기본 `tool_stream`, strict-tool / reasoning-payload 정리, plugin 소유 도구를 위한 fallback auth 재사용, forward-compat Grok 모델 해석, xAI 도구 스키마 profile, 미지원 스키마 키워드, native `web_search`, HTML 엔터티 도구 호출 인자 디코딩 같은 provider 소유 compat 패치를 소유하기 때문입니다.
- Mistral, OpenCode Zen, OpenCode Go는 transcript/tooling 특이사항을 코어 밖에 유지하기 위해 `capabilities`만 사용합니다.
- `byteplus`, `cloudflare-ai-gateway`, `huggingface`, `kimi-coding`, `nvidia`, `qianfan`, `synthetic`, `together`, `venice`, `vercel-ai-gateway`, `volcengine` 같은 카탈로그 전용 번들 provider는 `catalog`만 사용합니다.
- Qwen은 텍스트 provider를 위해 `catalog`를 사용하고, 멀티모달 표면을 위해 공유 media-understanding 및 video-generation 등록도 수행합니다.
- MiniMax와 Xiaomi는 추론은 여전히 공유 transport를 통해 실행되더라도 `/usage` 동작이 플러그인 소유이므로 `catalog`와 usage 훅을 함께 사용합니다.

## 런타임 헬퍼

플러그인은 `api.runtime`를 통해 선택된 코어 헬퍼에 접근할 수 있습니다. TTS의 경우:

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

- `textToSpeech`는 파일/voice-note 표면을 위한 일반 코어 TTS 출력 payload를 반환합니다.
- 코어 `messages.tts` 구성과 provider 선택을 사용합니다.
- PCM 오디오 버퍼 + 샘플 레이트를 반환합니다. 플러그인은 provider에 맞게 resample/encode해야 합니다.
- `listVoices`는 provider별로 선택 사항입니다. 벤더 소유 voice picker 또는 setup 흐름에 사용하세요.
- 음성 목록에는 provider 인식 picker를 위한 locale, 성별, personality 태그 같은 더 풍부한 메타데이터가 포함될 수 있습니다.
- 현재 전화용 TTS는 OpenAI와 ElevenLabs를 지원합니다. Microsoft는 지원하지 않습니다.

플러그인은 `api.registerSpeechProvider(...)`를 통해 음성 provider도 등록할 수 있습니다.

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

- TTS 정책, fallback, 응답 전달은 코어에 유지하세요.
- 벤더 소유 합성 동작에는 음성 provider를 사용하세요.
- 레거시 Microsoft `edge` 입력은 `microsoft` provider ID로 정규화됩니다.
- 선호되는 소유권 모델은 회사 중심입니다. OpenClaw가 capability 계약을 추가함에 따라 하나의 벤더 플러그인이 텍스트, 음성, 이미지, 미래의 미디어 provider를 모두 소유할 수 있습니다.

이미지/오디오/비디오 이해의 경우, 플러그인은 generic key/value bag 대신 하나의 typed media-understanding provider를 등록합니다.

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

- orchestration, fallback, config, 채널 연결은 코어에 유지하세요.
- 벤더 동작은 provider 플러그인에 유지하세요.
- 점진적 확장은 typed 상태를 유지해야 합니다. 새 optional 메서드, 새 optional 결과 필드, 새 optional capabilities처럼 확장하세요.
- 비디오 생성도 이미 같은 패턴을 따릅니다.
  - 코어가 capability 계약과 런타임 헬퍼를 소유합니다.
  - 벤더 플러그인이 `api.registerVideoGenerationProvider(...)`를 등록합니다.
  - 기능/채널 플러그인이 `api.runtime.videoGeneration.*`를 소비합니다.

media-understanding 런타임 헬퍼의 경우, 플러그인은 다음을 호출할 수 있습니다.

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

오디오 전사의 경우, 플러그인은 media-understanding 런타임 또는 이전 STT 별칭을 사용할 수 있습니다.

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

참고:

- `api.runtime.mediaUnderstanding.*`가 이미지/오디오/비디오 이해를 위한 선호 공유 표면입니다.
- 코어 media-understanding 오디오 구성(`tools.media.audio`)과 provider fallback 순서를 사용합니다.
- 전사 출력이 생성되지 않으면(예: 입력이 건너뛰어졌거나 지원되지 않음) `{ text: undefined }`를 반환합니다.
- `api.runtime.stt.transcribeAudioFile(...)`는 호환성 별칭으로 남아 있습니다.

플러그인은 `api.runtime.subagent`를 통해 백그라운드 subagent 실행도 시작할 수 있습니다.

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

- `provider`와 `model`은 영구 세션 변경이 아니라 실행별 override입니다.
- OpenClaw는 신뢰된 호출자에 대해서만 이 override 필드를 허용합니다.
- 플러그인 소유 fallback 실행의 경우, operator는 `plugins.entries.<id>.subagent.allowModelOverride: true`로 명시적으로 opt-in해야 합니다.
- 신뢰된 플러그인을 특정 정식 `provider/model` 대상으로 제한하려면 `plugins.entries.<id>.subagent.allowedModels`를 사용하고, 모든 대상을 명시적으로 허용하려면 `"*"`를 사용하세요.
- 신뢰되지 않은 플러그인의 subagent 실행도 동작하지만, override 요청은 조용히 fallback되지 않고 거부됩니다.

웹 검색의 경우, 플러그인은 에이전트 도구 연결에 직접 접근하는 대신 공유 런타임 헬퍼를 소비할 수 있습니다.

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

플러그인은 `api.registerWebSearchProvider(...)`를 통해 웹 검색 provider를 등록할 수도 있습니다.

참고:

- provider 선택, 자격 증명 해석, 공유 요청 의미론은 코어에 유지하세요.
- 벤더별 검색 transport에는 웹 검색 provider를 사용하세요.
- `api.runtime.webSearch.*`는 에이전트 도구 wrapper에 의존하지 않고 검색 동작이 필요한 기능/채널 플러그인을 위한 선호 공유 표면입니다.

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
- `listProviders(...)`: 사용 가능한 이미지 생성 provider와 그 capabilities를 나열합니다.

## Gateway HTTP 라우트

플러그인은 `api.registerHttpRoute(...)`로 HTTP 엔드포인트를 노출할 수 있습니다.

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

라우트 필드:

- `path`: gateway HTTP 서버 아래의 라우트 경로
- `auth`: 필수입니다. 일반 gateway auth가 필요하면 `"gateway"`를, 플러그인 관리 auth/webhook 검증이 필요하면 `"plugin"`을 사용하세요.
- `match`: 선택 사항입니다. `"exact"`(기본값) 또는 `"prefix"`.
- `replaceExisting`: 선택 사항입니다. 동일한 플러그인이 자신의 기존 라우트 등록을 대체할 수 있게 합니다.
- `handler`: 라우트가 요청을 처리했으면 `true`를 반환합니다.

참고:

- `api.registerHttpHandler(...)`는 제거되었으며 플러그인 로드 오류를 발생시킵니다. 대신 `api.registerHttpRoute(...)`를 사용하세요.
- 플러그인 라우트는 `auth`를 명시적으로 선언해야 합니다.
- 정확히 같은 `path + match` 충돌은 `replaceExisting: true`가 아닌 한 거부되며, 한 플러그인이 다른 플러그인의 라우트를 대체할 수는 없습니다.
- `auth` 수준이 다른 겹치는 라우트는 거부됩니다. `exact`/`prefix` fallthrough 체인은 같은 auth 수준에서만 유지하세요.
- `auth: "plugin"` 라우트는 operator 런타임 scope를 자동으로 받지 않습니다. 이는 권한 있는 Gateway 헬퍼 호출이 아니라, 플러그인 관리 webhook/서명 검증용입니다.
- `auth: "gateway"` 라우트는 Gateway 요청 런타임 scope 내부에서 실행되지만, 그 scope는 의도적으로 보수적입니다.
  - 공유 시크릿 bearer auth(`gateway.auth.mode = "token"` / `"password"`)는 호출자가 `x-openclaw-scopes`를 보내더라도 plugin-route 런타임 scope를 `operator.write`에 고정합니다.
  - 신뢰된 ID 포함 HTTP 모드(예: `trusted-proxy` 또는 private ingress의 `gateway.auth.mode = "none"`)는 헤더가 명시적으로 있을 때만 `x-openclaw-scopes`를 존중합니다.
  - 그런 ID 포함 plugin-route 요청에 `x-openclaw-scopes`가 없으면 런타임 scope는 `operator.write`로 fallback됩니다.
- 실무 규칙: gateway-auth 플러그인 라우트를 암묵적 관리자 표면으로 가정하지 마세요. 라우트에 관리자 전용 동작이 필요하면, ID 포함 auth 모드를 요구하고 명시적인 `x-openclaw-scopes` 헤더 계약을 문서화하세요.

## Plugin SDK import 경로

플러그인을 작성할 때는 단일 `openclaw/plugin-sdk` import 대신 SDK 하위 경로를 사용하세요.

- 플러그인 등록 기본 요소에는 `openclaw/plugin-sdk/plugin-entry`
- 일반적인 공유 플러그인 대상 계약에는 `openclaw/plugin-sdk/core`
- 루트 `openclaw.json` Zod 스키마 export(`OpenClawSchema`)에는 `openclaw/plugin-sdk/config-schema`
- `openclaw/plugin-sdk/channel-setup`,
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
  `openclaw/plugin-sdk/secret-input`,
  `openclaw/plugin-sdk/webhook-ingress` 같은 안정적인 채널 기본 요소는 공유 setup/auth/reply/webhook 연결용입니다. `channel-inbound`는 debounce, mention 매칭, inbound mention-policy 헬퍼, envelope formatting, inbound envelope context 헬퍼를 위한 공유 홈입니다.
  `channel-setup`은 좁은 optional-install setup seam입니다.
  `setup-runtime`은 `setupEntry` / 지연 시작에서 사용되는 런타임 안전 setup 표면이며 import-safe setup patch adapter를 포함합니다.
  `setup-adapter-runtime`은 env 인식 account-setup adapter seam입니다.
  `setup-tools`는 작은 CLI/archive/docs 헬퍼 seam(`formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`)입니다.
- `openclaw/plugin-sdk/channel-config-helpers`,
  `openclaw/plugin-sdk/allow-from`,
  `openclaw/plugin-sdk/channel-config-schema`,
  `openclaw/plugin-sdk/telegram-command-config`,
  `openclaw/plugin-sdk/channel-policy`,
  `openclaw/plugin-sdk/approval-gateway-runtime`,
  `openclaw/plugin-sdk/approval-handler-adapter-runtime`,
  `openclaw/plugin-sdk/approval-handler-runtime`,
  `openclaw/plugin-sdk/approval-runtime`,
  `openclaw/plugin-sdk/config-runtime`,
  `openclaw/plugin-sdk/infra-runtime`,
  `openclaw/plugin-sdk/agent-runtime`,
  `openclaw/plugin-sdk/lazy-runtime`,
  `openclaw/plugin-sdk/reply-history`,
  `openclaw/plugin-sdk/routing`,
  `openclaw/plugin-sdk/status-helpers`,
  `openclaw/plugin-sdk/text-runtime`,
  `openclaw/plugin-sdk/runtime-store`,
  `openclaw/plugin-sdk/directory-runtime` 같은 도메인 하위 경로는 공유 런타임/config 헬퍼용입니다.
  `telegram-command-config`는 Telegram custom command 정규화/검증을 위한 좁은 공개 seam이며 번들 Telegram 계약 표면이 일시적으로 없더라도 계속 사용할 수 있습니다.
  `text-runtime`은 assistant-visible-text 제거, markdown render/chunking 헬퍼, redaction 헬퍼, directive-tag 헬퍼, safe-text 유틸리티를 포함하는 공유 텍스트/markdown/logging seam입니다.
- 승인 전용 채널 seam은 플러그인의 하나의 `approvalCapability` 계약을 우선해야 합니다. 그러면 코어는 승인 auth, 전달, 렌더링, native-routing, 지연 native-handler 동작을 관련 없는 플러그인 필드에 섞는 대신 그 하나의 capability를 통해 읽습니다.
- `openclaw/plugin-sdk/channel-runtime`은 deprecated되었으며 오래된 플러그인을 위한 호환성 shim으로만 남아 있습니다. 새 코드는 더 좁고 일반적인 기본 요소를 import해야 하며, repo 코드에 새 shim import를 추가해서는 안 됩니다.
- 번들 확장 내부 구조는 비공개로 유지됩니다. 외부 플러그인은 `openclaw/plugin-sdk/*` 하위 경로만 사용해야 합니다. OpenClaw 코어/테스트 코드는 `index.js`, `api.js`, `runtime-api.js`, `setup-entry.js`, `login-qr-api.js` 같은 좁은 범위 파일과 같은 플러그인 패키지 루트의 repo 공개 엔트리 포인트를 사용할 수 있습니다. 코어나 다른 확장에서 플러그인 패키지의 `src/*`를 절대 import하지 마세요.
- repo 엔트리 포인트 분리:
  `<plugin-package-root>/api.js`는 헬퍼/타입 barrel,
  `<plugin-package-root>/runtime-api.js`는 runtime-only barrel,
  `<plugin-package-root>/index.js`는 번들 플러그인 엔트리,
  `<plugin-package-root>/setup-entry.js`는 setup 플러그인 엔트리입니다.
- 현재 번들 provider 예시:
  - Anthropic은 `wrapAnthropicProviderStream`, beta-header 헬퍼, `service_tier` 파싱 같은 Claude 스트림 헬퍼를 위해 `api.js` / `contract-api.js`를 사용합니다.
  - OpenAI는 provider builder, 기본 모델 헬퍼, realtime provider builder를 위해 `api.js`를 사용합니다.
  - OpenRouter는 provider builder와 onboarding/config 헬퍼를 위해 `api.js`를 사용하고, `register.runtime.js`는 여전히 repo-local 용도로 일반 `plugin-sdk/provider-stream` 헬퍼를 re-export할 수 있습니다.
- facade로 로드되는 공개 엔트리 포인트는 활성 런타임 config 스냅샷이 있으면 그것을 우선하고, OpenClaw가 아직 런타임 스냅샷을 제공하지 않을 때는 디스크의 해석된 config 파일로 fallback됩니다.
- 일반적인 공유 기본 요소가 여전히 선호되는 공개 SDK 계약입니다. 번들 채널 브랜드가 붙은 일부 예약된 호환성 헬퍼 seam이 여전히 존재합니다. 이를 새 서드파티 import 대상이 아니라 번들 유지보수/호환성 seam으로 취급하고, 새로운 채널 간 계약은 여전히 일반 `plugin-sdk/*` 하위 경로나 플러그인 로컬 `api.js` / `runtime-api.js` barrel에 추가해야 합니다.

호환성 메모:

- 새 코드에서는 루트 `openclaw/plugin-sdk` barrel을 피하세요.
- 먼저 좁고 안정적인 기본 요소를 우선하세요. 더 새로운 setup/pairing/reply/feedback/contract/inbound/threading/command/secret-input/webhook/infra/allowlist/status/message-tool 하위 경로가 새 번들 및 외부 플러그인 작업을 위한 의도된 계약입니다.
  대상 파싱/매칭은 `openclaw/plugin-sdk/channel-targets`에 속합니다.
  메시지 액션 게이트와 reaction message-id 헬퍼는 `openclaw/plugin-sdk/channel-actions`에 속합니다.
- 번들 확장별 헬퍼 barrel은 기본적으로 안정적이지 않습니다. 특정 번들 확장에만 필요한 헬퍼라면 `openclaw/plugin-sdk/<extension>`로 승격하지 말고 해당 확장의 로컬 `api.js` 또는 `runtime-api.js` seam 뒤에 두세요.
- 새 공유 헬퍼 seam은 채널 브랜드가 아니라 일반적이어야 합니다. 공유 대상 파싱은 `openclaw/plugin-sdk/channel-targets`에 속하고, 채널별 내부 구조는 소유 플러그인의 로컬 `api.js` 또는 `runtime-api.js` seam 뒤에 남아 있어야 합니다.
- `image-generation`, `media-understanding`, `speech` 같은 capability별 하위 경로는 현재 번들/네이티브 플러그인이 사용하기 때문에 존재합니다. 그렇다고 모든 export된 헬퍼가 장기적으로 고정된 외부 계약이라는 뜻은 아닙니다.

## 메시지 도구 스키마

플러그인은 채널별 `describeMessageTool(...)` 스키마 기여를 소유해야 합니다. 제공자별 필드는 공유 코어가 아니라 플러그인 내부에 유지하세요.

공유 가능한 이식형 스키마 조각에는 `openclaw/plugin-sdk/channel-actions`를 통해 export되는 일반 헬퍼를 재사용하세요.

- 버튼 그리드 스타일 payload에는 `createMessageToolButtonsSchema()`
- 구조화된 카드 payload에는 `createMessageToolCardSchema()`

어떤 스키마 형태가 한 provider에만 의미가 있다면 공유 SDK로 승격하지 말고 그 플러그인 자체 소스에 정의하세요.

## 채널 대상 해석

채널 플러그인은 채널별 대상 의미론을 소유해야 합니다. 공유 outbound 호스트는 일반적으로 유지하고, provider 규칙에는 메시징 adapter 표면을 사용하세요.

- `messaging.inferTargetChatType({ to })`는 정규화된 대상을 디렉터리 조회 전에 `direct`, `group`, `channel` 중 무엇으로 취급할지 결정합니다.
- `messaging.targetResolver.looksLikeId(raw, normalized)`는 어떤 입력이 디렉터리 검색 대신 바로 ID 형태 해석으로 넘어가야 하는지 코어에 알려 줍니다.
- `messaging.targetResolver.resolveTarget(...)`는 정규화 후 또는 디렉터리 미스 후 코어가 최종 provider 소유 해석이 필요할 때 쓰는 플러그인 fallback입니다.
- `messaging.resolveOutboundSessionRoute(...)`는 대상이 해석된 후 provider별 세션 라우트 구성을 소유합니다.

권장 분리:

- peer/group 검색 전에 일어나야 하는 범주 결정에는 `inferTargetChatType`을 사용하세요.
- "이것을 명시적/native 대상 ID로 취급하라"는 검사에는 `looksLikeId`를 사용하세요.
- `resolveTarget`은 provider별 정규화 fallback용으로 사용하고, 광범위한 디렉터리 검색용으로 사용하지 마세요.
- chat ID, thread ID, JID, handle, room ID 같은 provider-native ID는 일반 SDK 필드가 아니라 `target` 값이나 provider별 파라미터 내부에 유지하세요.

## config 기반 디렉터리

config에서 디렉터리 엔트리를 파생하는 플러그인은 그 로직을 플러그인 내부에 유지하고, `openclaw/plugin-sdk/directory-runtime`의 공유 헬퍼를 재사용해야 합니다.

다음과 같은 config 기반 peer/group이 필요한 채널에서 사용하세요.

- allowlist 기반 DM peer
- 구성된 채널/그룹 맵
- 계정 범위의 정적 디렉터리 fallback

`directory-runtime`의 공유 헬퍼는 일반 작업만 처리합니다.

- 쿼리 필터링
- limit 적용
- dedupe/정규화 헬퍼
- `ChannelDirectoryEntry[]` 빌드

채널별 계정 검사와 ID 정규화는 플러그인 구현에 남겨 두어야 합니다.

## provider 카탈로그

provider 플러그인은 `registerProvider({ catalog: { run(...) { ... } } })`를 통해 추론용 모델 카탈로그를 정의할 수 있습니다.

`catalog.run(...)`은 OpenClaw가 `models.providers`에 기록하는 것과 같은 형태를 반환합니다.

- 하나의 provider 엔트리에는 `{ provider }`
- 여러 provider 엔트리에는 `{ providers }`

provider별 model ID, base URL 기본값, 또는 auth-gated 모델 메타데이터를 플러그인이 소유한다면 `catalog`를 사용하세요.

`catalog.order`는 플러그인의 카탈로그가 OpenClaw의 내장 암시적 provider에 대해 언제 병합되는지 제어합니다.

- `simple`: 일반 API-key 또는 env 기반 provider
- `profile`: auth profile이 있을 때 나타나는 provider
- `paired`: 관련된 여러 provider 엔트리를 합성하는 provider
- `late`: 다른 암시적 provider 이후의 마지막 패스

나중에 오는 provider가 키 충돌 시 우선하므로, 플러그인은 같은 provider ID를 가진 내장 provider 엔트리를 의도적으로 override할 수 있습니다.

호환성:

- `discovery`는 여전히 레거시 별칭으로 동작합니다.
- `catalog`와 `discovery`가 모두 등록되면 OpenClaw는 `catalog`를 사용합니다.

## 읽기 전용 채널 검사

플러그인이 채널을 등록한다면, `resolveAccount(...)`와 함께 `plugin.config.inspectAccount(cfg, accountId)` 구현을 우선하세요.

이유:

- `resolveAccount(...)`는 런타임 경로입니다. 자격 증명이 완전히 materialize되었다고 가정할 수 있고, 필요한 secret이 없으면 빠르게 실패해도 됩니다.
- `openclaw status`, `openclaw status --all`, `openclaw channels status`, `openclaw channels resolve`, doctor/config repair 흐름 같은 읽기 전용 명령 경로는 단순히 구성을 설명하기 위해 런타임 자격 증명을 materialize할 필요가 없어야 합니다.

권장 `inspectAccount(...)` 동작:

- 설명용 계정 상태만 반환합니다.
- `enabled`와 `configured`를 유지합니다.
- 관련 있을 때 자격 증명 source/status 필드를 포함합니다. 예:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- 읽기 전용 가용성을 보고하기 위해 raw token 값을 반환할 필요는 없습니다. `tokenStatus: "available"`(및 해당 source 필드)만으로 상태형 명령에는 충분합니다.
- 자격 증명이 SecretRef로 구성되었지만 현재 명령 경로에서 사용할 수 없으면 `configured_unavailable`을 사용하세요.

이렇게 하면 읽기 전용 명령이 크래시하거나 계정을 미구성으로 잘못 보고하는 대신 "구성되어 있지만 이 명령 경로에서는 사용할 수 없음"을 보고할 수 있습니다.

## 패키지 팩

플러그인 디렉터리에는 `openclaw.extensions`를 포함한 `package.json`이 있을 수 있습니다.

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

각 엔트리는 하나의 플러그인이 됩니다. 팩이 여러 확장을 나열하면 플러그인 ID는 `name/<fileBase>`가 됩니다.

플러그인이 npm 의존성을 import한다면, 해당 디렉터리에 이를 설치하여 `node_modules`를 사용할 수 있도록 하세요(`npm install` / `pnpm install`).

보안 가드레일: 모든 `openclaw.extensions` 엔트리는 심볼릭 링크 해석 후에도 플러그인 디렉터리 내부에 있어야 합니다. 패키지 디렉터리 밖으로 벗어나는 엔트리는 거부됩니다.

보안 메모: `openclaw plugins install`은 `npm install --omit=dev --ignore-scripts`로 플러그인 의존성을 설치합니다(라이프사이클 스크립트 없음, 런타임에 dev dependencies 없음). 플러그인 의존성 트리는 "순수 JS/TS"로 유지하고 `postinstall` 빌드가 필요한 패키지는 피하세요.

선택 사항: `openclaw.setupEntry`는 가벼운 setup 전용 모듈을 가리킬 수 있습니다.
OpenClaw가 비활성화된 채널 플러그인의 setup 표면이 필요할 때, 또는 채널 플러그인이 활성화되었지만 아직 구성되지 않았을 때는 전체 플러그인 엔트리 대신 `setupEntry`를 로드합니다. 이렇게 하면 메인 플러그인 엔트리가 도구, 훅, 기타 runtime-only 코드를 연결하더라도 시작과 setup을 더 가볍게 유지할 수 있습니다.

선택 사항: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`은 채널이 이미 구성된 경우에도 gateway의 pre-listen 시작 단계에서 채널 플러그인을 같은 `setupEntry` 경로에 opt-in시킬 수 있습니다.

이 옵션은 `setupEntry`가 gateway가 수신을 시작하기 전에 반드시 존재해야 하는 시작 표면을 완전히 커버할 때만 사용하세요. 실제로는 setup 엔트리가 시작이 의존하는 모든 채널 소유 capability를 등록해야 한다는 뜻입니다. 예를 들면:

- 채널 등록 자체
- gateway가 수신을 시작하기 전에 사용 가능해야 하는 모든 HTTP 라우트
- 같은 시점에 존재해야 하는 모든 gateway 메서드, 도구, 서비스

전체 엔트리가 여전히 필요한 시작 capability를 소유하고 있다면 이 플래그를 활성화하지 마세요. 기본 동작을 유지하고 OpenClaw가 시작 중 전체 엔트리를 로드하게 두세요.

번들 채널은 전체 채널 런타임이 로드되기 전에 코어가 참조할 수 있는 setup 전용 계약 표면 헬퍼를 게시할 수도 있습니다. 현재 setup 승격 표면은 다음과 같습니다.

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

코어는 레거시 단일 계정 채널 config를 전체 플러그인 엔트리를 로드하지 않고 `channels.<id>.accounts.*`로 승격해야 할 때 이 표면을 사용합니다. 현재 번들 예시는 Matrix입니다. named account가 이미 존재할 때 인증/bootstrap 키만 named 승격 계정으로 이동하고, 항상 `accounts.default`를 만드는 대신 구성된 비정규 default-account 키를 보존할 수 있습니다.

이러한 setup patch adapter는 번들 계약 표면 디스커버리를 지연 상태로 유지합니다. import 시점은 가볍게 유지되고, 승격 표면은 모듈 import 중 번들 채널 시작을 다시 진입하지 않고 첫 사용 시에만 로드됩니다.

이러한 시작 표면에 gateway RPC 메서드가 포함될 때는 플러그인 전용 prefix 아래에 두세요. 코어 관리자 네임스페이스(`config.*`, `exec.approvals.*`, `wizard.*`, `update.*`)는 예약되어 있으며, 플러그인이 더 좁은 scope를 요청하더라도 항상 `operator.admin`으로 해석됩니다.

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

### 채널 카탈로그 메타데이터

채널 플러그인은 `openclaw.channel`을 통해 setup/discovery 메타데이터를, `openclaw.install`을 통해 install 힌트를 광고할 수 있습니다. 이렇게 하면 코어 카탈로그를 data-free로 유지할 수 있습니다.

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

- `detailLabel`: 더 풍부한 카탈로그/상태 표면을 위한 보조 레이블
- `docsLabel`: 문서 링크 텍스트 override
- `preferOver`: 이 카탈로그 엔트리가 더 높은 우선순위를 가져야 하는 낮은 우선순위 플러그인/채널 ID
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: 선택 표면 복사 제어
- `markdownCapable`: outbound formatting 결정용으로 해당 채널이 markdown-capable임을 표시
- `exposure.configured`: `false`이면 구성된 채널 목록 표면에서 채널 숨김
- `exposure.setup`: `false`이면 대화형 setup/configure picker에서 채널 숨김
- `exposure.docs`: 문서 탐색 표면에서 채널을 내부/비공개로 표시
- `showConfigured` / `showInSetup`: 호환성을 위해 여전히 허용되는 레거시 별칭이지만 `exposure`를 우선
- `quickstartAllowFrom`: 채널을 표준 quickstart `allowFrom` 흐름에 opt-in
- `forceAccountBinding`: 계정이 하나뿐이어도 명시적 계정 바인딩 요구
- `preferSessionLookupForAnnounceTarget`: announce 대상 해석 시 세션 조회를 우선

OpenClaw는 **외부 채널 카탈로그**(예: MPM 레지스트리 export)도 병합할 수 있습니다. 다음 위치 중 하나에 JSON 파일을 두세요.

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

또는 `OPENCLAW_PLUGIN_CATALOG_PATHS`(또는 `OPENCLAW_MPM_CATALOG_PATHS`)로 하나 이상의 JSON 파일을 지정하세요(쉼표/세미콜론/`PATH` 구분). 각 파일은 `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }` 형식을 가져야 합니다. 파서는 `"entries"` 키의 레거시 별칭으로 `"packages"`나 `"plugins"`도 허용합니다.

## 컨텍스트 엔진 플러그인

컨텍스트 엔진 플러그인은 ingest, assemble, compaction을 위한 세션 컨텍스트 orchestration을 소유합니다. 플러그인에서 `api.registerContextEngine(id, factory)`로 등록한 다음, 활성 엔진은 `plugins.slots.contextEngine`으로 선택합니다.

기본 컨텍스트 파이프라인을 단순히 확장하는 것이 아니라 이를 교체하거나 확장해야 한다면 이 방식을 사용하세요.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", () => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

엔진이 compaction 알고리즘을 **소유하지 않는다면**, `compact()`를 구현한 채 명시적으로 위임하세요.

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

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
    async assemble({ messages, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## 새 capability 추가하기

플러그인에 현재 API에 맞지 않는 동작이 필요하다면, 비공개 직접 접근으로 플러그인 시스템을 우회하지 마세요. 누락된 capability를 추가하세요.

권장 순서:

1. 코어 계약 정의
   코어가 소유해야 하는 공유 동작을 결정합니다: 정책, fallback, config 병합, 라이프사이클, 채널 대상 의미론, 런타임 헬퍼 형태.
2. typed 플러그인 등록/런타임 표면 추가
   가장 작은 유용한 typed capability 표면으로 `OpenClawPluginApi` 및/또는 `api.runtime`를 확장합니다.
3. 코어 + 채널/기능 소비자 연결
   채널과 기능 플러그인은 벤더 구현을 직접 import하지 말고, 코어를 통해 새 capability를 소비해야 합니다.
4. 벤더 구현 등록
   그다음 벤더 플러그인이 이 capability에 대해 백엔드를 등록합니다.
5. 계약 커버리지 추가
   시간이 지나도 소유권과 등록 형태가 명시적으로 유지되도록 테스트를 추가합니다.

이것이 OpenClaw가 특정 제공자의 세계관에 하드코딩되지 않으면서도 명확한 방향성을 유지하는 방식입니다. 구체적인 파일 체크리스트와 예시는 [Capability Cookbook](/ko/plugins/architecture)를 참조하세요.

### capability 체크리스트

새 capability를 추가할 때 구현은 보통 다음 표면들을 함께 건드려야 합니다.

- `src/<capability>/types.ts`의 코어 계약 타입
- `src/<capability>/runtime.ts`의 코어 runner/런타임 헬퍼
- `src/plugins/types.ts`의 플러그인 API 등록 표면
- `src/plugins/registry.ts`의 플러그인 레지스트리 연결
- 기능/채널 플러그인이 이를 소비해야 할 때의 `src/plugins/runtime/*` 플러그인 런타임 노출
- `src/test-utils/plugin-registration.ts`의 capture/test 헬퍼
- `src/plugins/contracts/registry.ts`의 소유권/계약 assertion
- `docs/`의 operator/플러그인 문서

이 표면들 중 하나가 빠져 있다면, 보통 그 capability가 아직 완전히 통합되지 않았다는 신호입니다.

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

이렇게 규칙은 단순하게 유지됩니다.

- 코어는 capability 계약 + orchestration을 소유합니다.
- 벤더 플러그인은 벤더 구현을 소유합니다.
- 기능/채널 플러그인은 런타임 헬퍼를 소비합니다.
- 계약 테스트는 소유권을 명시적으로 유지합니다.
