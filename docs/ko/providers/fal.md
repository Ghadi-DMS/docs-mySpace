---
read_when:
    - OpenClaw에서 fal 이미지 생성을 사용하고 싶을 때
    - FAL_KEY 인증 흐름이 필요할 때
    - image_generate 또는 video_generate에 fal 기본값을 사용하고 싶을 때
summary: OpenClaw에서 fal 이미지 및 비디오 생성 설정
title: fal
x-i18n:
    generated_at: "2026-04-06T03:11:03Z"
    model: gpt-5.4
    provider: openai
    source_hash: 1922907d2c8360c5877a56495323d54bd846d47c27a801155e3d11e3f5706fbd
    source_path: providers/fal.md
    workflow: 15
---

# fal

OpenClaw는 호스팅 이미지 및 비디오 생성을 위한 번들 `fal` 공급자를 제공합니다.

- 공급자: `fal`
- 인증: `FAL_KEY`(정식; `FAL_API_KEY`도 폴백으로 동작)
- API: fal 모델 엔드포인트

## 빠른 시작

1. API 키 설정:

```bash
openclaw onboard --auth-choice fal-api-key
```

2. 기본 이미지 모델 설정:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## 이미지 생성

번들 `fal` 이미지 생성 공급자의 기본값은
`fal/fal-ai/flux/dev`입니다.

- 생성: 요청당 최대 4개 이미지
- 편집 모드: 활성화됨, 참조 이미지 1개
- `size`, `aspectRatio`, `resolution` 지원
- 현재 편집 관련 주의 사항: fal 이미지 편집 엔드포인트는 `aspectRatio` 재정의를 지원하지 않습니다.

fal을 기본 이미지 공급자로 사용하려면:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/fal-ai/flux/dev",
      },
    },
  },
}
```

## 비디오 생성

번들 `fal` 비디오 생성 공급자의 기본값은
`fal/fal-ai/minimax/video-01-live`입니다.

- 모드: text-to-video 및 단일 이미지 참조 흐름
- 런타임: 장시간 실행 작업을 위한 큐 기반 submit/status/result 흐름

fal을 기본 비디오 공급자로 사용하려면:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "fal/fal-ai/minimax/video-01-live",
      },
    },
  },
}
```

## 관련 항목

- [Image Generation](/ko/tools/image-generation)
- [Video Generation](/tools/video-generation)
- [Configuration Reference](/ko/gateway/configuration-reference#agent-defaults)
