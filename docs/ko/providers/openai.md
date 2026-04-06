---
read_when:
    - OpenClaw에서 OpenAI 모델을 사용하고 싶을 때
    - API key 대신 Codex subscription auth를 사용하고 싶을 때
summary: OpenClaw에서 API key 또는 Codex subscription으로 OpenAI 사용하기
title: OpenAI
x-i18n:
    generated_at: "2026-04-06T03:12:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9e04db5787f6ed7b1eda04d965c10febae10809fc82ae4d9769e7163234471f5
    source_path: providers/openai.md
    workflow: 15
---

# OpenAI

OpenAI는 GPT 모델용 개발자 API를 제공합니다. Codex는 subscription
접근을 위한 **ChatGPT 로그인** 또는 사용량 기반 접근을 위한 **API key** 로그인을 지원합니다. Codex cloud는 ChatGPT 로그인이 필요합니다.
OpenAI는 OpenClaw 같은 외부 도구/워크플로에서 subscription OAuth 사용을 명시적으로 지원합니다.

## 기본 상호작용 스타일

OpenClaw는 `openai/*`와
`openai-codex/*` 실행 모두에 대해 작은 OpenAI 전용 프롬프트 오버레이를 추가할 수 있습니다. 기본적으로 이 오버레이는 어시스턴트를 따뜻하고,
협력적이며, 간결하고, 직접적이고, 약간 더 감정적으로 표현되도록 유지하면서도
기본 OpenClaw 시스템 프롬프트를 대체하지는 않습니다. 이 친화적 오버레이는
전체 출력은 간결하게 유지하면서, 자연스럽게 어울릴 때 가끔 emoji도 허용합니다.

Config 키:

`plugins.entries.openai.config.personality`

허용 값:

- `"friendly"`: 기본값, OpenAI 전용 오버레이 활성화
- `"off"`: 오버레이를 비활성화하고 기본 OpenClaw 프롬프트만 사용

범위:

- `openai/*` 모델에 적용됩니다.
- `openai-codex/*` 모델에 적용됩니다.
- 다른 provider에는 영향을 주지 않습니다.

이 동작은 기본적으로 활성화되어 있습니다. 향후 로컬 config 변경에도 이를 유지하려면
명시적으로 `"friendly"`를 유지하세요.

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "friendly",
        },
      },
    },
  },
}
```

### OpenAI 프롬프트 오버레이 비활성화

수정되지 않은 기본 OpenClaw 프롬프트를 원한다면 오버레이를 `"off"`로 설정하세요.

```json5
{
  plugins: {
    entries: {
      openai: {
        config: {
          personality: "off",
        },
      },
    },
  },
}
```

Config CLI로 직접 설정할 수도 있습니다.

```bash
openclaw config set plugins.entries.openai.config.personality off
```

## 옵션 A: OpenAI API key (OpenAI Platform)

**가장 적합한 경우:** direct API 접근 및 사용량 기반 과금.
OpenAI 대시보드에서 API key를 발급받으세요.

### CLI 설정

```bash
openclaw onboard --auth-choice openai-api-key
# 또는 비대화형
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### Config 스니펫

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

OpenAI의 현재 API 모델 문서는 direct
OpenAI API 사용을 위해 `gpt-5.4`와 `gpt-5.4-pro`를 나열합니다. OpenClaw는
둘 다 `openai/*` Responses 경로를 통해 전달합니다.
OpenClaw는 오래된 `openai/gpt-5.3-codex-spark` 행을 의도적으로 숨기는데,
live OpenAI API 호출에서 현재 이 모델이 거부되기 때문입니다.

OpenClaw는 direct OpenAI
API 경로에서 `openai/gpt-5.3-codex-spark`를 **노출하지 않습니다**. `pi-ai`에는 여전히 해당 모델의 내장 행이 있지만, live OpenAI API
요청은 현재 이를 거부합니다. Spark는 OpenClaw에서 Codex 전용으로 취급됩니다.

## 이미지 생성

번들 `openai` plugin은 공용
`image_generate` 도구를 통한 이미지 생성도 등록합니다.

- 기본 이미지 모델: `openai/gpt-image-1`
- 생성: 요청당 최대 4개 이미지
- 편집 모드: 활성화됨, 최대 5개 참조 이미지
- `size` 지원
- 현재 OpenAI 전용 주의사항: OpenClaw는 현재 `aspectRatio` 또는
  `resolution` override를 OpenAI Images API로 전달하지 않습니다

OpenAI를 기본 이미지 provider로 사용하려면:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-1",
      },
    },
  },
}
```

공용 도구
파라미터, provider 선택, failover 동작은 [Image Generation](/ko/tools/image-generation)을 참조하세요.

## 비디오 생성

번들 `openai` plugin은 공용
`video_generate` 도구를 통한 비디오 생성도 등록합니다.

- 기본 비디오 모델: `openai/sora-2`
- 모드: 텍스트-투-비디오, 이미지-투-비디오, 단일 비디오 참조/편집 흐름
- 현재 제한: 이미지 1개 또는 비디오 참조 입력 1개
- 현재 OpenAI 전용 주의사항: OpenClaw는 현재 네이티브 OpenAI 비디오 생성에 대해 `size`
  override만 전달합니다. `aspectRatio`, `resolution`, `audio`, `watermark` 같은 지원되지 않는 선택적 override는 무시되며
  도구 경고로 다시 보고됩니다.

OpenAI를 기본 비디오 provider로 사용하려면:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openai/sora-2",
      },
    },
  },
}
```

공용 도구
파라미터, provider 선택, failover 동작은 [Video Generation](/tools/video-generation)을 참조하세요.

## 옵션 B: OpenAI Code (Codex) subscription

**가장 적합한 경우:** API key 대신 ChatGPT/Codex subscription 접근을 사용하려는 경우.
Codex cloud는 ChatGPT 로그인이 필요하며, Codex CLI는 ChatGPT 또는 API key 로그인을 지원합니다.

### CLI 설정 (Codex OAuth)

```bash
# wizard에서 Codex OAuth 실행
openclaw onboard --auth-choice openai-codex

# 또는 OAuth 직접 실행
openclaw models auth login --provider openai-codex
```

### Config 스니펫 (Codex subscription)

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

OpenAI의 현재 Codex 문서는 현재 Codex 모델로 `gpt-5.4`를 나열합니다. OpenClaw는
이를 ChatGPT/Codex OAuth 사용을 위한 `openai-codex/gpt-5.4`로 매핑합니다.

온보딩이 기존 Codex CLI 로그인을 재사용하면 해당 자격 증명은 계속
Codex CLI가 관리합니다. 만료 시 OpenClaw는 먼저 외부 Codex 소스를 다시 읽고,
provider가 이를 갱신할 수 있으면 별도의 OpenClaw 전용 사본을 소유하는 대신
갱신된 자격 증명을 다시 Codex 저장소에 기록합니다.

Codex Spark entitlement가 있는 Codex 계정이라면 OpenClaw는 다음도 지원합니다.

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw는 Codex Spark를 Codex 전용으로 취급합니다. direct
`openai/gpt-5.3-codex-spark` API-key 경로는 노출하지 않습니다.

OpenClaw는 또한 `pi-ai`가 이를
발견했을 때 `openai-codex/gpt-5.3-codex-spark`를 유지합니다. 이는 entitlement 의존적이며 실험적이라고 보세요. Codex Spark는
GPT-5.4 `/fast`와 별개이며, 가용성은 로그인한 Codex /
ChatGPT 계정에 따라 달라집니다.

### Codex 컨텍스트 윈도우 상한

OpenClaw는 Codex 모델 metadata와 런타임 컨텍스트 상한을 별도의
값으로 취급합니다.

`openai-codex/gpt-5.4`의 경우:

- 네이티브 `contextWindow`: `1050000`
- 기본 런타임 `contextTokens` 상한: `272000`

이렇게 하면 모델 metadata의 정확성을 유지하면서도, 실제로 더 나은 지연 시간과 품질 특성을 보이는 더 작은 기본 런타임
윈도우를 유지할 수 있습니다.

다른 유효 상한을 원하면 `models.providers.<provider>.models[].contextTokens`를 설정하세요.

```json5
{
  models: {
    providers: {
      "openai-codex": {
        models: [
          {
            id: "gpt-5.4",
            contextTokens: 160000,
          },
        ],
      },
    },
  },
}
```

`contextWindow`는 네이티브 모델
metadata를 선언하거나 override할 때만 사용하세요. 런타임 컨텍스트 예산을 제한하려면 `contextTokens`를 사용하세요.

### 기본 transport

OpenClaw는 모델 스트리밍에 `pi-ai`를 사용합니다. `openai/*`와
`openai-codex/*` 모두에서 기본 transport는 `"auto"`입니다(WebSocket 우선, 이후 SSE
fallback).

`"auto"` 모드에서 OpenClaw는 SSE로 fallback하기 전에 초기의 재시도 가능한 WebSocket 실패도
한 번 재시도합니다. 강제 `"websocket"` 모드는 transport 오류를
fallback 뒤에 숨기지 않고 직접 표시합니다.

`"auto"` 모드에서 연결 또는 초기 턴 WebSocket 실패가 발생하면, OpenClaw는
해당 세션의 WebSocket 경로를 약 60초 동안 성능 저하 상태로 표시하고
transport 사이를 계속 오가는 대신, cool-down 동안 이후 턴을
SSE로 전송합니다.

네이티브 OpenAI 계열 endpoint(`openai/*`, `openai-codex/*`, 그리고 Azure
OpenAI Responses)의 경우 OpenClaw는 요청에 안정적인 세션 및 턴 식별 상태도
첨부하여 재시도, 재연결, SSE fallback이 동일한
대화 식별성에 맞춰 유지되도록 합니다. 네이티브 OpenAI 계열 경로에서는 여기에는
안정적인 세션/턴 요청 식별 header와 일치하는 transport metadata가 포함됩니다.

OpenClaw는 또한 OpenAI 사용량 카운터가 세션/상태 표면에 도달하기 전에 transport 변형 간에
정규화합니다. 네이티브 OpenAI/Codex Responses 트래픽은
`input_tokens` / `output_tokens` 또는
`prompt_tokens` / `completion_tokens`로 usage를 보고할 수 있으며, OpenClaw는 이를 `/status`, `/usage`, 세션 로그를 위한 동일한 입력
및 출력 카운터로 취급합니다. 네이티브
WebSocket 트래픽에서 `total_tokens`가 생략되거나(`0`으로 보고되거나) 하면, OpenClaw는
정규화된 입력 + 출력 합계로 fallback하여 세션/상태 표시가 계속 채워지도록 합니다.

`agents.defaults.models.<provider/model>.params.transport`를 설정할 수 있습니다.

- `"sse"`: SSE 강제
- `"websocket"`: WebSocket 강제
- `"auto"`: WebSocket 시도 후 SSE로 fallback

`openai/*`(Responses API)의 경우 OpenClaw는 WebSocket transport 사용 시
기본적으로 WebSocket warm-up도 활성화합니다(`openaiWsWarmup: true`).

관련 OpenAI 문서:

- [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### OpenAI WebSocket warm-up

OpenAI 문서에서는 warm-up을 선택 사항으로 설명합니다. OpenClaw는
WebSocket transport 사용 시 첫 번째 턴 지연 시간을 줄이기 위해
`openai/*`에 대해 기본적으로 이를 활성화합니다.

### warm-up 비활성화

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### warm-up 명시적 활성화

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### OpenAI와 Codex 우선 처리

OpenAI의 API는 `service_tier=priority`를 통해 우선 처리를 노출합니다. OpenClaw에서는 네이티브 OpenAI/Codex Responses endpoint에 이 필드를 전달하려면
`agents.defaults.models["<provider>/<model>"].params.serviceTier`
를 설정하세요.

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

지원되는 값은 `auto`, `default`, `flex`, `priority`입니다.

OpenClaw는 `params.serviceTier`를 direct `openai/*` Responses
요청과 `openai-codex/*` Codex Responses 요청 양쪽에 전달하지만, 이는 해당 모델이
네이티브 OpenAI/Codex endpoint를 가리킬 때만 해당합니다.

중요한 동작:

- direct `openai/*`는 `api.openai.com`을 대상으로 해야 합니다
- `openai-codex/*`는 `chatgpt.com/backend-api`를 대상으로 해야 합니다
- 두 provider 중 어느 쪽이든 다른 base URL이나 proxy를 통해 라우팅하면, OpenClaw는 `service_tier`를 그대로 둡니다

### OpenAI 빠른 모드

OpenClaw는 `openai/*`와
`openai-codex/*` 세션 모두에 대해 공용 빠른 모드 토글을 제공합니다.

- Chat/UI: `/fast status|on|off`
- Config: `agents.defaults.models["<provider>/<model>"].params.fastMode`

빠른 모드가 활성화되면, OpenClaw는 이를 OpenAI 우선 처리로 매핑합니다.

- `api.openai.com`으로 가는 direct `openai/*` Responses 호출은 `service_tier = "priority"`를 전송합니다
- `chatgpt.com/backend-api`로 가는 `openai-codex/*` Responses 호출도 `service_tier = "priority"`를 전송합니다
- 기존 payload `service_tier` 값은 유지됩니다
- 빠른 모드는 `reasoning` 또는 `text.verbosity`를 재작성하지 않습니다

특히 GPT 5.4의 경우 가장 일반적인 설정은 다음과 같습니다.

- `openai/gpt-5.4` 또는 `openai-codex/gpt-5.4`를 사용하는 세션에서 `/fast on` 전송
- 또는 `agents.defaults.models["openai/gpt-5.4"].params.fastMode = true` 설정
- Codex OAuth도 사용한다면 `agents.defaults.models["openai-codex/gpt-5.4"].params.fastMode = true`도 함께 설정

예시:

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

세션 override가 config보다 우선합니다. Sessions UI에서 세션 override를 지우면
세션은 설정된 기본값으로 돌아갑니다.

### 네이티브 OpenAI와 OpenAI-compatible 경로의 차이

OpenClaw는 direct OpenAI, Codex, Azure OpenAI endpoint를
일반적인 OpenAI-compatible `/v1` proxy와 다르게 취급합니다.

- 네이티브 `openai/*`, `openai-codex/*`, Azure OpenAI 경로는
  reasoning을 명시적으로 비활성화할 때 `reasoning: { effort: "none" }`를 그대로 유지합니다
- 네이티브 OpenAI 계열 경로는 기본적으로 tool schema를 strict 모드로 설정합니다
- 숨겨진 OpenClaw attribution header(`originator`, `version`, 그리고
  `User-Agent`)는 검증된 네이티브 OpenAI 호스트
  (`api.openai.com`) 및 네이티브 Codex 호스트(`chatgpt.com/backend-api`)에만 첨부됩니다
- 네이티브 OpenAI/Codex 경로는
  `service_tier`, Responses `store`, OpenAI reasoning-compat payload, prompt-cache 힌트 같은
  OpenAI 전용 요청 shaping을 유지합니다
- proxy 스타일 OpenAI-compatible 경로는 더 느슨한 compat 동작을 유지하며
  strict tool schema, 네이티브 전용 요청 shaping, 숨겨진
  OpenAI/Codex attribution header를 강제하지 않습니다

Azure OpenAI는 transport 및 compat 동작 측면에서는 네이티브 라우팅 범주에 남아 있지만, 숨겨진 OpenAI/Codex attribution header는 받지 않습니다.

이렇게 하면 현재 네이티브 OpenAI Responses 동작을 유지하면서도
서드파티 `/v1` 백엔드에 오래된 OpenAI-compatible shim을 강요하지 않게 됩니다.

### OpenAI Responses 서버 측 압축

direct OpenAI Responses 모델(`api.openai.com`의 `baseUrl`과 함께 `api: "openai-responses"`를 사용하는 `openai/*`)의 경우, OpenClaw는 이제 OpenAI 서버 측
압축 payload 힌트를 자동 활성화합니다.

- `store: true`를 강제합니다(단, 모델 compat가 `supportsStore: false`로 설정한 경우 제외)
- `context_management: [{ type: "compaction", compact_threshold: ... }]`를 주입합니다

기본적으로 `compact_threshold`는 모델 `contextWindow`의 `70%`이며
(사용할 수 없을 때는 `80000`).

### 서버 측 압축 명시적 활성화

호환되는 Responses 모델(예: Azure OpenAI Responses)에서
`context_management` 주입을 강제하려면 이를 사용하세요.

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### 사용자 지정 임계값으로 활성화

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### 서버 측 압축 비활성화

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction`은 `context_management` 주입만 제어합니다.
direct OpenAI Responses 모델은 compat가
`supportsStore: false`로 설정하지 않는 한 계속 `store: true`를 강제합니다.

## 참고

- 모델 ref는 항상 `provider/model` 형식을 사용합니다([/concepts/models](/ko/concepts/models) 참조).
- Auth 세부 정보 및 재사용 규칙은 [/concepts/oauth](/ko/concepts/oauth)에 있습니다.
