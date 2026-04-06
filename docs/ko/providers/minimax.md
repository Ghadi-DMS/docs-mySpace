---
read_when:
    - OpenClaw에서 MiniMax 모델을 사용하고 싶을 때
    - MiniMax 설정 가이드가 필요할 때
summary: OpenClaw에서 MiniMax 모델 사용하기
title: MiniMax
x-i18n:
    generated_at: "2026-04-06T03:11:44Z"
    model: gpt-5.4
    provider: openai
    source_hash: 9ca35c43cdde53f6f09d9e12d48ce09e4c099cf8cbe1407ac6dbb45b1422507e
    source_path: providers/minimax.md
    workflow: 15
---

# MiniMax

OpenClaw의 MiniMax 공급자는 기본값으로 **MiniMax M2.7**을 사용합니다.

MiniMax는 다음도 제공합니다.

- T2A v2를 통한 번들 음성 합성
- `MiniMax-VL-01`을 통한 번들 이미지 이해
- `music-2.5+`를 통한 번들 음악 생성
- MiniMax Coding Plan 검색 API를 통한 번들 `web_search`

공급자 분리:

- `minimax`: API 키 텍스트 공급자, 번들 이미지 생성, 이미지 이해, 음성, 웹 검색 포함
- `minimax-portal`: OAuth 텍스트 공급자, 번들 이미지 생성 및 이미지 이해 포함

## 모델 라인업

- `MiniMax-M2.7`: 기본 호스팅 추론 모델
- `MiniMax-M2.7-highspeed`: 더 빠른 M2.7 추론 티어
- `image-01`: 이미지 생성 모델(생성 및 image-to-image 편집)

## 이미지 생성

MiniMax plugin은 `image_generate` 도구용으로 `image-01` 모델을 등록합니다. 지원 항목:

- 종횡비 제어가 가능한 **text-to-image 생성**
- 종횡비 제어가 가능한 **image-to-image 편집**(피사체 참조)
- 요청당 최대 **9개 출력 이미지**
- 편집 요청당 최대 **1개 참조 이미지**
- 지원되는 종횡비: `1:1`, `16:9`, `4:3`, `3:2`, `2:3`, `3:4`, `9:16`, `21:9`

이미지 생성에 MiniMax를 사용하려면 이미지 생성 공급자로 설정하세요.

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

plugin은 텍스트 모델과 동일한 `MINIMAX_API_KEY` 또는 OAuth 인증을 사용합니다. MiniMax가 이미 설정되어 있다면 추가 구성은 필요하지 않습니다.

`minimax`와 `minimax-portal` 모두 동일한
`image-01` 모델로 `image_generate`를 등록합니다. API 키 설정은 `MINIMAX_API_KEY`를 사용하고, OAuth 설정은
번들 `minimax-portal` 인증 경로를 대신 사용할 수 있습니다.

온보딩 또는 API 키 설정이 명시적인 `models.providers.minimax`
항목을 작성하면 OpenClaw는 `MiniMax-M2.7` 및
`MiniMax-M2.7-highspeed`를 `input: ["text", "image"]`와 함께 구체화합니다.

내장 번들 MiniMax 텍스트 카탈로그 자체는 해당 명시적 공급자 config가 존재할 때까지
텍스트 전용 메타데이터로 유지됩니다. 이미지 이해는
plugin 소유 `MiniMax-VL-01` 미디어 공급자를 통해 별도로 노출됩니다.

공유 도구
파라미터, 공급자 선택, 폴백 동작은 [Image Generation](/ko/tools/image-generation)을 참고하세요.

## 음악 생성

번들 `minimax` plugin은 공유
`music_generate` 도구를 통해 음악 생성도 등록합니다.

- 기본 음악 모델: `minimax/music-2.5+`
- `minimax/music-2.5` 및 `minimax/music-2.0`도 지원
- 프롬프트 제어: `lyrics`, `instrumental`, `durationSeconds`
- 출력 형식: `mp3`
- 세션 기반 실행은 `action: "status"`를 포함한 공유 task/status 흐름을 통해 분리 실행됨

기본 음악 공급자로 MiniMax를 사용하려면:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "minimax/music-2.5+",
      },
    },
  },
}
```

공유 도구
파라미터, 공급자 선택, 폴백 동작은 [Music Generation](/tools/music-generation)을 참고하세요.

## 비디오 생성

번들 `minimax` plugin은 공유
`video_generate` 도구를 통해 비디오 생성도 등록합니다.

- 기본 비디오 모델: `minimax/MiniMax-Hailuo-2.3`
- 모드: text-to-video 및 단일 이미지 참조 흐름
- `aspectRatio` 및 `resolution` 지원

기본 비디오 공급자로 MiniMax를 사용하려면:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "minimax/MiniMax-Hailuo-2.3",
      },
    },
  },
}
```

공유 도구
파라미터, 공급자 선택, 폴백 동작은 [Video Generation](/tools/video-generation)을 참고하세요.

## 이미지 이해

MiniMax plugin은 텍스트
카탈로그와 별도로 이미지 이해를 등록합니다.

- `minimax`: 기본 이미지 모델 `MiniMax-VL-01`
- `minimax-portal`: 기본 이미지 모델 `MiniMax-VL-01`

이 때문에 번들 텍스트 공급자 카탈로그가 여전히 텍스트 전용 M2.7 채팅 ref만 보여주더라도
자동 미디어 라우팅은 MiniMax 이미지 이해를 사용할 수 있습니다.

## 웹 검색

MiniMax plugin은 MiniMax Coding Plan
검색 API를 통해 `web_search`도 등록합니다.

- 공급자 id: `minimax`
- 구조화된 결과: 제목, URL, 스니펫, 관련 쿼리
- 권장 env var: `MINIMAX_CODE_PLAN_KEY`
- 허용되는 env 별칭: `MINIMAX_CODING_API_KEY`
- 호환성 폴백: 이미 coding-plan 토큰을 가리키는 경우 `MINIMAX_API_KEY`
- 지역 재사용: `plugins.entries.minimax.config.webSearch.region`, 그다음 `MINIMAX_API_HOST`, 그다음 MiniMax 공급자 base URL
- 검색은 공급자 id `minimax`에 유지되며, OAuth CN/global 설정은 `models.providers.minimax-portal.baseUrl`을 통해 여전히 간접적으로 지역을 조정할 수 있음

config는 `plugins.entries.minimax.config.webSearch.*` 아래에 있습니다.
자세한 내용은 [MiniMax Search](/ko/tools/minimax-search)를 참고하세요.

## 설정 선택

### MiniMax OAuth (Coding Plan) - 권장

**적합한 경우:** API 키 없이 OAuth를 통해 MiniMax Coding Plan을 빠르게 설정하고 싶을 때

명시적인 지역 OAuth 선택으로 인증하세요.

```bash
openclaw onboard --auth-choice minimax-global-oauth
# or
openclaw onboard --auth-choice minimax-cn-oauth
```

선택 매핑:

- `minimax-global-oauth`: 해외 사용자(`api.minimax.io`)
- `minimax-cn-oauth`: 중국 사용자(`api.minimaxi.com`)

자세한 내용은 OpenClaw 저장소의 MiniMax plugin 패키지 README를 참고하세요.

### MiniMax M2.7 (API key)

**적합한 경우:** Anthropic 호환 API를 사용하는 호스팅 MiniMax

CLI로 구성:

- 대화형 온보딩:

```bash
openclaw onboard --auth-choice minimax-global-api
# or
openclaw onboard --auth-choice minimax-cn-api
```

- `minimax-global-api`: 해외 사용자(`api.minimax.io`)
- `minimax-cn-api`: 중국 사용자(`api.minimaxi.com`)

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "minimax/MiniMax-M2.7" } } },
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "MiniMax-M2.7",
            name: "MiniMax M2.7",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
          {
            id: "MiniMax-M2.7-highspeed",
            name: "MiniMax M2.7 Highspeed",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0.6, output: 2.4, cacheRead: 0.06, cacheWrite: 0.375 },
            contextWindow: 204800,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Anthropic 호환 스트리밍 경로에서 OpenClaw는 이제 명시적으로 `thinking`을 설정하지 않는 한
기본적으로 MiniMax thinking을 비활성화합니다. MiniMax의
스트리밍 엔드포인트는 네이티브 Anthropic thinking 블록 대신 OpenAI 스타일 델타 청크로 `reasoning_content`를 내보내므로,
암묵적으로 활성화된 상태로 두면 내부 reasoning이 보이는 출력으로
누출될 수 있습니다.

### 폴백으로 MiniMax M2.7 사용(예시)

**적합한 경우:** 가장 강력한 최신 세대 모델을 Primary로 유지하고, MiniMax M2.7로 페일오버하고 싶을 때
아래 예시는 구체적인 Primary로 Opus를 사용합니다. 선호하는 최신 세대 Primary 모델로 바꿔 사용하세요.

```json5
{
  env: { MINIMAX_API_KEY: "sk-..." },
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "primary" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
    },
  },
}
```

## `openclaw configure`로 구성

JSON을 수정하지 않고 대화형 config wizard로 MiniMax를 설정하세요.

1. `openclaw configure` 실행
2. **Model/auth** 선택
3. **MiniMax** 인증 옵션 선택
4. 메시지가 표시되면 기본 모델 선택

현재 wizard/CLI의 MiniMax 인증 선택 항목:

- `minimax-global-oauth`
- `minimax-cn-oauth`
- `minimax-global-api`
- `minimax-cn-api`

## 구성 옵션

- `models.providers.minimax.baseUrl`: `https://api.minimax.io/anthropic`(Anthropic 호환)을 권장합니다. `https://api.minimax.io/v1`은 OpenAI 호환 페이로드용 선택 사항입니다.
- `models.providers.minimax.api`: `anthropic-messages`를 권장합니다. `openai-completions`는 OpenAI 호환 페이로드용 선택 사항입니다.
- `models.providers.minimax.apiKey`: MiniMax API 키(`MINIMAX_API_KEY`)
- `models.providers.minimax.models`: `id`, `name`, `reasoning`, `contextWindow`, `maxTokens`, `cost` 정의
- `agents.defaults.models`: 허용 목록에 둘 모델에 별칭 부여
- `models.mode`: 내장 공급자와 함께 MiniMax를 추가하려면 `merge` 유지

## 참고

- 모델 ref는 인증 경로를 따릅니다.
  - API 키 설정: `minimax/<model>`
  - OAuth 설정: `minimax-portal/<model>`
- 기본 채팅 모델: `MiniMax-M2.7`
- 대체 채팅 모델: `MiniMax-M2.7-highspeed`
- `api: "anthropic-messages"`에서는 OpenClaw가
  params/config에서 이미 thinking이 명시적으로 설정되지 않은 한
  `thinking: { type: "disabled" }`를 주입합니다.
- `/fast on` 또는 `params.fastMode: true`는 Anthropic 호환 스트림 경로에서
  `MiniMax-M2.7`를 `MiniMax-M2.7-highspeed`로 재작성합니다.
- 온보딩 및 직접 API 키 설정은
  두 M2.7 변형 모두에 대해 `input: ["text", "image"]`가 포함된 명시적 모델 정의를 작성합니다.
- 번들 공급자 카탈로그는 현재 명시적 MiniMax 공급자 config가 존재할 때까지
  채팅 ref를 텍스트 전용 메타데이터로 노출합니다.
- Coding Plan 사용량 API: `https://api.minimaxi.com/v1/api/openplatform/coding_plan/remains`(coding plan 키 필요)
- OpenClaw는 MiniMax coding-plan 사용량을 다른 공급자에서 사용하는 동일한 `% left`
  표시로 정규화합니다. MiniMax의 원시 `usage_percent` / `usagePercent`
  필드는 소비된 할당량이 아니라 남은 할당량이므로 OpenClaw는 이를 반전합니다.
  카운트 기반 필드가 있으면 그것이 우선합니다. API가 `model_remains`를 반환하면,
  OpenClaw는 채팅 모델 항목을 우선 사용하고, 필요 시 `start_time` / `end_time`에서
  기간 레이블을 도출하며, coding-plan 기간을 더 쉽게 구분할 수 있도록
  선택된 모델 이름을 플랜 레이블에 포함합니다.
- 사용량 스냅샷은 `minimax`, `minimax-cn`, `minimax-portal`을
  동일한 MiniMax 할당량 표면으로 처리하며, Coding Plan 키 env var로 폴백하기 전에
  저장된 MiniMax OAuth를 우선 사용합니다.
- 정확한 비용 추적이 필요하다면 `models.json`의 가격 값을 업데이트하세요.
- MiniMax Coding Plan 추천 링크(10% 할인): [https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)
- 공급자 규칙은 [/concepts/model-providers](/ko/concepts/model-providers)를 참고하세요.
- 현재 공급자 id를 확인하려면 `openclaw models list`를 사용한 다음,
  `openclaw models set minimax/MiniMax-M2.7` 또는
  `openclaw models set minimax-portal/MiniMax-M2.7`로 전환하세요.

## 문제 해결

### "Unknown model: minimax/MiniMax-M2.7"

이는 보통 **MiniMax 공급자가 구성되지 않았음**을 의미합니다(일치하는
공급자 항목이 없고 MiniMax 인증 프로필/env 키도 없음). 이
감지에 대한 수정은 **2026.1.12**에 포함되어 있습니다. 다음 방법으로 해결하세요.

- **2026.1.12**로 업그레이드(또는 소스 `main`에서 실행)한 뒤 gateway를 재시작
- `openclaw configure`를 실행하고 **MiniMax** 인증 옵션 선택, 또는
- 일치하는 `models.providers.minimax` 또는
  `models.providers.minimax-portal` 블록을 수동으로 추가, 또는
- `MINIMAX_API_KEY`, `MINIMAX_OAUTH_TOKEN`, 또는 MiniMax 인증 프로필을 설정하여
  일치하는 공급자가 주입될 수 있게 함

모델 id는 **대소문자를 구분**하므로 확인하세요.

- API 키 경로: `minimax/MiniMax-M2.7` 또는 `minimax/MiniMax-M2.7-highspeed`
- OAuth 경로: `minimax-portal/MiniMax-M2.7` 또는
  `minimax-portal/MiniMax-M2.7-highspeed`

그런 다음 다음으로 다시 확인하세요.

```bash
openclaw models list
```
