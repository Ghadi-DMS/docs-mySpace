---
read_when:
    - OpenClaw에서 Runway 비디오 생성을 사용하고 싶을 때
    - Runway API 키/env 설정이 필요할 때
    - Runway를 기본 비디오 provider로 설정하고 싶을 때
summary: OpenClaw에서 Runway 비디오 생성 설정
title: Runway
x-i18n:
    generated_at: "2026-04-06T03:11:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: bc615d1a26f7a4b890d29461e756690c858ecb05024cf3c4d508218022da6e76
    source_path: providers/runway.md
    workflow: 15
---

# Runway

OpenClaw는 호스팅된 비디오 생성을 위한 번들 `runway` provider를 제공합니다.

- Provider id: `runway`
- 인증: `RUNWAYML_API_SECRET`(정식) 또는 `RUNWAY_API_KEY`
- API: Runway 작업 기반 비디오 생성(`GET /v1/tasks/{id}` 폴링)

## 빠른 시작

1. API 키 설정:

```bash
openclaw onboard --auth-choice runway-api-key
```

2. Runway를 기본 비디오 provider로 설정:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "runway/gen4.5"
```

3. 에이전트에게 비디오 생성을 요청하세요. 그러면 자동으로 Runway가 사용됩니다.

## 지원 모드

| 모드            | 모델               | 참조 입력                |
| --------------- | ------------------ | ------------------------ |
| 텍스트-비디오   | `gen4.5`(기본값)   | 없음                     |
| 이미지-비디오   | `gen4.5`           | 로컬 또는 원격 이미지 1개 |
| 비디오-비디오   | `gen4_aleph`       | 로컬 또는 원격 비디오 1개 |

- 로컬 이미지 및 비디오 참조는 data URI를 통해 지원됩니다.
- 비디오-비디오는 현재 구체적으로 `runway/gen4_aleph`가 필요합니다.
- 텍스트 전용 실행은 현재 `16:9`와 `9:16` 종횡비를 지원합니다.

## 구성

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## 관련 문서

- [Video Generation](/tools/video-generation) -- 공유 도구 파라미터, provider 선택, 비동기 동작
- [Configuration Reference](/ko/gateway/configuration-reference#agent-defaults)
