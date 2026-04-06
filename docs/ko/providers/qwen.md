---
read_when:
    - OpenClaw에서 Qwen을 사용하려고 합니다
    - 이전에 Qwen OAuth를 사용했습니다
summary: OpenClaw의 번들 qwen provider를 통해 Qwen Cloud 사용하기
title: Qwen
x-i18n:
    generated_at: "2026-04-06T03:11:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: f175793693ab6a4c3f1f4d42040e673c15faf7603a500757423e9e06977c989d
    source_path: providers/qwen.md
    workflow: 15
---

# Qwen

<Warning>

**Qwen OAuth는 제거되었습니다.** `portal.qwen.ai` 엔드포인트를 사용하던
무료 등급 OAuth 통합(`qwen-portal`)은 더 이상 사용할 수 없습니다.
배경은 [Issue #49557](https://github.com/openclaw/openclaw/issues/49557)를
참조하세요.

</Warning>

## 권장: Qwen Cloud

이제 OpenClaw는 Qwen을 정식 번들 provider로 취급하며 정식 ID는
`qwen`입니다. 번들 provider는 Qwen Cloud / Alibaba DashScope 및
Coding Plan 엔드포인트를 대상으로 하며, 레거시 `modelstudio` ID도
호환성 별칭으로 계속 작동합니다.

- Provider: `qwen`
- 권장 env var: `QWEN_API_KEY`
- 호환성을 위해 다음도 허용: `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY`
- API 스타일: OpenAI 호환

`qwen3.6-plus`를 사용하려면 **Standard (pay-as-you-go)** 엔드포인트를
권장합니다. Coding Plan 지원은 공개 카탈로그보다 늦을 수 있습니다.

```bash
# 전역 Coding Plan 엔드포인트
openclaw onboard --auth-choice qwen-api-key

# 중국 Coding Plan 엔드포인트
openclaw onboard --auth-choice qwen-api-key-cn

# 전역 Standard (pay-as-you-go) 엔드포인트
openclaw onboard --auth-choice qwen-standard-api-key

# 중국 Standard (pay-as-you-go) 엔드포인트
openclaw onboard --auth-choice qwen-standard-api-key-cn
```

레거시 `modelstudio-*` auth-choice ID와 `modelstudio/...` 모델 ref도
호환성 별칭으로 계속 작동하지만, 새로운 setup 흐름에서는 정식
`qwen-*` auth-choice ID와 `qwen/...` 모델 ref를 사용하는 것이 좋습니다.

온보딩 후 기본 모델을 설정하세요:

```json5
{
  agents: {
    defaults: {
      model: { primary: "qwen/qwen3.5-plus" },
    },
  },
}
```

## 플랜 유형과 엔드포인트

| 플랜                        | 리전 | Auth choice                | 엔드포인트                                     |
| --------------------------- | ---- | -------------------------- | ---------------------------------------------- |
| Standard (pay-as-you-go)    | 중국 | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`      |
| Standard (pay-as-you-go)    | 전역 | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Coding Plan (구독)          | 중국 | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`               |
| Coding Plan (구독)          | 전역 | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`          |

provider는 auth choice를 기반으로 엔드포인트를 자동 선택합니다. 정식
choice는 `qwen-*` 계열을 사용하며, `modelstudio-*`는 호환성 전용으로 유지됩니다.
config에서 사용자 지정 `baseUrl`로 재정의할 수 있습니다.

기본 Model Studio 엔드포인트는 공유
`openai-completions` 전송에서 스트리밍 사용량 호환성을 알립니다. OpenClaw는 이제 이를 엔드포인트
capability를 기준으로 처리하므로, 동일한 기본 호스트를 대상으로 하는 DashScope 호환
사용자 지정 provider ID도 내장 `qwen` provider ID를 특별히 요구하지 않고
동일한 스트리밍 사용량 동작을 상속합니다.

## API key 받기

- **키 관리**: [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys)
- **문서**: [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)

## 내장 카탈로그

OpenClaw는 현재 다음 번들 Qwen 카탈로그를 제공합니다:

| Model ref                   | 입력        | 컨텍스트  | 참고사항                                           |
| --------------------------- | ----------- | --------- | -------------------------------------------------- |
| `qwen/qwen3.5-plus`         | text, image | 1,000,000 | 기본 모델                                          |
| `qwen/qwen3.6-plus`         | text, image | 1,000,000 | 이 모델이 필요하면 Standard 엔드포인트 권장        |
| `qwen/qwen3-max-2026-01-23` | text        | 262,144   | Qwen Max 라인                                      |
| `qwen/qwen3-coder-next`     | text        | 262,144   | 코딩                                               |
| `qwen/qwen3-coder-plus`     | text        | 1,000,000 | 코딩                                               |
| `qwen/MiniMax-M2.5`         | text        | 1,000,000 | 추론 활성화                                        |
| `qwen/glm-5`                | text        | 202,752   | GLM                                                |
| `qwen/glm-4.7`              | text        | 202,752   | GLM                                                |
| `qwen/kimi-k2.5`            | text, image | 262,144   | Alibaba를 통한 Moonshot AI                         |

모델이 번들 카탈로그에 있더라도 엔드포인트와 과금 플랜에 따라 사용 가능 여부는 달라질 수 있습니다.

기본 스트리밍 사용량 호환성은 Coding Plan 호스트와
Standard DashScope 호환 호스트 모두에 적용됩니다:

- `https://coding.dashscope.aliyuncs.com/v1`
- `https://coding-intl.dashscope.aliyuncs.com/v1`
- `https://dashscope.aliyuncs.com/compatible-mode/v1`
- `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`

## Qwen 3.6 Plus 가용성

`qwen3.6-plus`는 Standard (pay-as-you-go) Model Studio
엔드포인트에서 사용할 수 있습니다:

- 중국: `dashscope.aliyuncs.com/compatible-mode/v1`
- 전역: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

Coding Plan 엔드포인트가 `qwen3.6-plus`에 대해 "unsupported model" 오류를 반환하면,
Coding Plan 엔드포인트/key 조합 대신 Standard (pay-as-you-go)로 전환하세요.

## capability 계획

`qwen` extension은 단지 coding/text 모델만이 아니라
전체 Qwen Cloud 표면을 위한 벤더 홈으로 자리 잡고 있습니다.

- 텍스트/채팅 모델: 현재 번들 제공
- tool calling, 구조화 출력, thinking: OpenAI 호환 전송에서 상속
- 이미지 생성: provider-plugin 계층에서 계획 중
- 이미지/비디오 이해: 현재 Standard 엔드포인트에서 번들 제공
- speech/오디오: provider-plugin 계층에서 계획 중
- 메모리 임베딩/리랭킹: 임베딩 adapter 표면을 통해 계획 중
- 비디오 생성: 공유 video-generation capability를 통해 현재 번들 제공

## 멀티모달 추가 기능

이제 `qwen` extension은 다음도 제공합니다:

- `qwen-vl-max-latest`를 통한 비디오 이해
- 다음을 통한 Wan 비디오 생성:
  - `wan2.6-t2v` (기본값)
  - `wan2.6-i2v`
  - `wan2.6-r2v`
  - `wan2.6-r2v-flash`
  - `wan2.7-r2v`

이 멀티모달 표면은 Coding Plan 엔드포인트가 아니라
**Standard** DashScope 엔드포인트를 사용합니다.

- 전역/국제 Standard base URL: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
- 중국 Standard base URL: `https://dashscope.aliyuncs.com/compatible-mode/v1`

비디오 생성의 경우 OpenClaw는 작업 제출 전에 구성된 Qwen 리전을
일치하는 DashScope AIGC 호스트로 매핑합니다:

- 전역/국제: `https://dashscope-intl.aliyuncs.com`
- 중국: `https://dashscope.aliyuncs.com`

즉, Coding Plan 또는 Standard Qwen 호스트 중 하나를 가리키는 일반적인
`models.providers.qwen.baseUrl`을 사용해도 비디오 생성은 계속 올바른
리전 DashScope 비디오 엔드포인트를 사용하게 됩니다.

비디오 생성의 경우 기본 모델을 명시적으로 설정하세요:

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

- 요청당 출력 비디오는 최대 **1개**
- 입력 이미지는 최대 **1개**
- 입력 비디오는 최대 **4개**
- 길이는 최대 **10초**
- `size`, `aspectRatio`, `resolution`, `audio`, `watermark` 지원
- 참조 이미지/비디오 모드는 현재 **원격 http(s) URL**이 필요합니다. DashScope 비디오 엔드포인트가
  해당 참조에 대해 업로드된 로컬 버퍼를 허용하지 않기 때문에 로컬
  파일 경로는 초기에 거부됩니다.

공유 도구 파라미터, provider 선택, failover 동작은
[Video Generation](/tools/video-generation)을 참조하세요.

## 환경 참고

Gateway가 데몬(launchd/systemd)으로 실행되는 경우 `QWEN_API_KEY`가
해당 프로세스에서 사용 가능해야 합니다(예: `~/.openclaw/.env` 또는
`env.shellEnv`를 통해).
