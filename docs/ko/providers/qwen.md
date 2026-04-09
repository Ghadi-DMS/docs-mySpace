---
read_when:
    - OpenClaw에서 Qwen을 사용하려는 경우
    - 이전에 Qwen OAuth를 사용한 적이 있는 경우
summary: OpenClaw의 번들된 qwen provider를 통해 Qwen Cloud 사용
title: Qwen
x-i18n:
    generated_at: "2026-04-09T01:30:59Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4786df2cb6ec1ab29d191d012c61dcb0e5468bf0f8561fbbb50eed741efad325
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**Qwen OAuth는 제거되었습니다.** `portal.qwen.ai` 엔드포인트를 사용하던
무료 등급 OAuth 통합(`qwen-portal`)은 더 이상 사용할 수 없습니다.
배경은 [Issue #49557](https://github.com/openclaw/openclaw/issues/49557)를
참고하세요.

</Warning>

## 권장: Qwen Cloud

OpenClaw는 이제 Qwen을 정식 canonical id
`qwen`을 가진 1급 번들 provider로 취급합니다. 번들 provider는 Qwen Cloud / Alibaba DashScope 및
Coding Plan 엔드포인트를 대상으로 하며, 레거시 `modelstudio` id도
호환성 별칭으로 계속 작동합니다.

- Provider: `qwen`
- 권장 env var: `QWEN_API_KEY`
- 호환성을 위해 추가 지원: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- API 스타일: OpenAI 호환

`qwen3.6-plus`를 사용하려면 **Standard (pay-as-you-go)** 엔드포인트를
권장합니다. Coding Plan 지원은 공개 카탈로그보다 늦을 수 있습니다.

```bash
# Global Coding Plan endpoint
openclaw onboard --auth-choice qwen-api-key

# China Coding Plan endpoint
openclaw onboard --auth-choice qwen-api-key-cn

# Global Standard (pay-as-you-go) endpoint
openclaw onboard --auth-choice qwen-standard-api-key

# China Standard (pay-as-you-go) endpoint
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

레거시 `modelstudio-*` auth-choice id와 `modelstudio/...` 모델 참조도
호환성 별칭으로 계속 작동하지만, 새 설정 흐름에서는 canonical
`qwen-*` auth-choice id와 `qwen/...` 모델 참조를 사용하는 것이 좋습니다.

온보딩 후 기본 모델을 설정하세요.

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## 플랜 유형 및 엔드포인트

| 플랜                       | 지역   | Auth choice                | 엔드포인트                                      |
| -------------------------- | ------ | -------------------------- | ----------------------------------------------- |
| Standard (pay-as-you-go)   | China  | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (pay-as-you-go)   | Global | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (subscription) | China  | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (subscription) | Global | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

provider는 auth choice에 따라 엔드포인트를 자동 선택합니다. canonical
선택지는 `qwen-*` 계열을 사용하며, `modelstudio-*`는 호환성 전용입니다.
config에서 커스텀 `baseUrl`로 재정의할 수 있습니다.

기본 Model Studio 엔드포인트는 공용
`openai-completions` 전송에서 스트리밍 사용량 호환성을 알립니다. OpenClaw는 이제 이를 엔드포인트 기능 기준으로 처리하므로, 동일한 기본 호스트를 대상으로 하는 DashScope 호환 커스텀 provider id도
내장 `qwen` provider id를 특별히 요구하지 않고 동일한 스트리밍 사용량 동작을 상속합니다.

## API 키 받기

- **키 관리**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **문서**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## 내장 카탈로그

OpenClaw는 현재 다음 번들 Qwen 카탈로그를 제공합니다. 구성된 카탈로그는
엔드포인트 인식형입니다. Coding Plan config에서는 Standard 엔드포인트에서만
작동하는 것으로 알려진 모델이 제외됩니다.

| 모델 참조                  | 입력        | 컨텍스트  | 참고                                               |
| -------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus`         | text, image | 1,000,000 | 기본 모델                                          |
| `qwen/qwen3.6-plus`         | text, image | 1,000,000 | 이 모델이 필요하면 Standard 엔드포인트 권장        |
| `qwen/qwen3-max-2026-01-23` | text        | 262,144   | Qwen Max 계열                                      |
| `qwen/qwen3-coder-next`     | text        | 262,144   | 코딩                                               |
| `qwen/qwen3-coder-plus`     | text        | 1,000,000 | 코딩                                               |
| `qwen/MiniMax-M2.5`         | text        | 1,000,000 | reasoning 활성화됨                                 |
| `qwen/glm-5`                | text        | 202,752   | GLM                                                |
| `qwen/glm-4.7`              | text        | 202,752   | GLM                                                |
| `qwen/kimi-k2.5`            | text, image | 262,144   | Alibaba를 통한 Moonshot AI                         |

모델이 번들 카탈로그에 있더라도, 실제 사용 가능 여부는 엔드포인트와 결제 플랜에 따라 달라질 수 있습니다.

기본 스트리밍 사용량 호환성은 Coding Plan 호스트와
Standard DashScope 호환 호스트 모두에 적용됩니다.

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Qwen 3.6 Plus 사용 가능 여부

`qwen3.6-plus`는 Standard (pay-as-you-go) Model Studio
엔드포인트에서 사용할 수 있습니다.

- China: `dashscope.aliyuncs.com/compatible-mode/v1`
- Global: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

Coding Plan 엔드포인트에서 `qwen3.6-plus`에 대해
"unsupported model" 오류가 반환되면, Coding Plan
엔드포인트/키 조합 대신 Standard (pay-as-you-go)로 전환하세요.

## 기능 계획

`qwen` 확장은 단순한 코딩/텍스트 모델뿐 아니라 전체 Qwen
Cloud 표면의 벤더 홈으로 자리 잡고 있습니다.

- 텍스트/채팅 모델: 현재 번들 제공
- Tool calling, structured output, thinking: OpenAI 호환 전송에서 상속
- 이미지 생성: provider-plugin 계층에서 예정
- 이미지/비디오 이해: 현재 Standard 엔드포인트에서 번들 제공
- 음성/오디오: provider-plugin 계층에서 예정
- 메모리 임베딩/reranking: 임베딩 어댑터 표면을 통해 예정
- 비디오 생성: 공용 비디오 생성 기능을 통해 현재 번들 제공

## 멀티모달 추가 기능

`qwen` 확장은 이제 다음도 제공합니다.

- `qwen-vl-max-latest`를 통한 비디오 이해
- 다음을 통한 Wan 비디오 생성:
  - `wan2.6-t2v` (기본값)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

이 멀티모달 표면은 Coding Plan 엔드포인트가 아니라
**Standard** DashScope 엔드포인트를 사용합니다.

- Global/Intl Standard base URL: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- China Standard base URL: `https://dashscope.aliyuncs.com/compatible-mode/v1`

비디오 생성의 경우, OpenClaw는 작업 제출 전에 구성된 Qwen 지역을 일치하는
DashScope AIGC 호스트에 매핑합니다.

- Global/Intl: `https://dashscope-intl.aliyuncs.com`
- China: `https://dashscope.aliyuncs.com`

즉, Coding Plan 또는 Standard Qwen 호스트를 가리키는 일반적인
`models.providers.qwen.baseUrl`을 사용하더라도 비디오 생성은 올바른
지역 DashScope 비디오 엔드포인트를 계속 사용합니다.

비디오 생성의 경우 기본 모델을 명시적으로 설정하세요.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

현재 번들 Qwen 비디오 생성 제한:

- 요청당 최대 **1**개 출력 비디오
- 최대 **1**개 입력 이미지
- 최대 **4**개 입력 비디오
- 최대 **10초** duration
- `size`, `aspectRatio`, `resolution`, `audio`, `watermark` 지원
- 참조 이미지/비디오 모드는 현재 **원격 http(s) URL**이 필요합니다. 로컬
  파일 경로는 DashScope 비디오 엔드포인트가 해당 참조에 대해 업로드된 로컬 버퍼를
  허용하지 않기 때문에 사전에 거부됩니다.

공용 도구 매개변수,
provider 선택, 장애 조치 동작은 [Video Generation](/ko/tools/video-generation)을 참고하세요.

## 환경 참고

Gateway가 데몬(launchd/systemd)으로 실행되는 경우, `QWEN_API_KEY`가
해당 프로세스에서 사용 가능해야 합니다(예: `~/.openclaw/.env` 또는
`env.shellEnv`를 통해).
