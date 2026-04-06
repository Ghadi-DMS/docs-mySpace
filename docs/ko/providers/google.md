---
read_when:
    - OpenClaw에서 Google Gemini 모델을 사용하고 싶을 때
    - API 키 인증 흐름이 필요할 때
summary: Google Gemini 설정(API 키, 이미지 생성, 미디어 이해, 웹 검색)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-04-06T03:11:15Z"
    model: gpt-5.4
    provider: openai
    source_hash: 358d33a68275b01ebd916a3621dd651619cb9a1d062e2fb6196a7f3c501c015a
    source_path: providers/google.md
    workflow: 15
---

# Google (Gemini)

Google plugin은 Google AI Studio를 통해 Gemini 모델에 대한 액세스를 제공하며,
Gemini Grounding을 통한 이미지 생성, 미디어 이해(이미지/오디오/비디오), 웹 검색도 지원합니다.

- Provider: `google`
- 인증: `GEMINI_API_KEY` 또는 `GOOGLE_API_KEY`
- API: Google Gemini API

## 빠른 시작

1. API 키 설정:

```bash
openclaw onboard --auth-choice gemini-api-key
```

2. 기본 모델 설정:

```json5
{
  agents: {
    defaults: {
      model: { primary: "google/gemini-3.1-pro-preview" },
    },
  },
}
```

## 비대화형 예시

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY"
```

## 기능

| 기능                   | 지원 여부        |
| ---------------------- | ---------------- |
| 채팅 completions       | 예               |
| 이미지 생성            | 예               |
| 음악 생성              | 예               |
| 이미지 이해            | 예               |
| 오디오 전사            | 예               |
| 비디오 이해            | 예               |
| 웹 검색(Grounding)     | 예               |
| Thinking/reasoning     | 예(Gemini 3.1+)  |

## 직접 Gemini 캐시 재사용

직접 Gemini API 실행(`api: "google-generative-ai"`)의 경우, OpenClaw는 이제
구성된 `cachedContent` 핸들을 Gemini 요청에 그대로 전달합니다.

- 모델별 또는 전역 params에
  `cachedContent` 또는 레거시 `cached_content`를 사용해 구성할 수 있습니다.
- 둘 다 있으면 `cachedContent`가 우선합니다.
- 예시 값: `cachedContents/prebuilt-context`
- Gemini의 캐시 적중 사용량은 업스트림 `cachedContentTokenCount`를 기준으로
  OpenClaw `cacheRead`로 정규화됩니다.

예시:

```json5
{
  agents: {
    defaults: {
      models: {
        "google/gemini-2.5-pro": {
          params: {
            cachedContent: "cachedContents/prebuilt-context",
          },
        },
      },
    },
  },
}
```

## 이미지 생성

번들된 `google` 이미지 생성 provider의 기본값은
`google/gemini-3.1-flash-image-preview`입니다.

- `google/gemini-3-pro-image-preview`도 지원
- 생성: 요청당 최대 4개 이미지
- 편집 모드: 활성화, 입력 이미지 최대 5개
- 기하 제어: `size`, `aspectRatio`, `resolution`

이미지 생성, 미디어 이해, Gemini Grounding은 모두
`google` provider ID에 유지됩니다.

Google을 기본 이미지 provider로 사용하려면:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image-preview",
      },
    },
  },
}
```

공유 도구
파라미터, provider 선택, 장애 조치 동작은 [Image Generation](/ko/tools/image-generation)을 참고하세요.

## 비디오 생성

번들된 `google` plugin은 공유
`video_generate` 도구를 통해 비디오 생성도 등록합니다.

- 기본 비디오 모델: `google/veo-3.1-fast-generate-preview`
- 모드: text-to-video, image-to-video, 단일 비디오 참조 흐름
- `aspectRatio`, `resolution`, `audio` 지원
- 현재 duration 제한: **4~8초**

Google을 기본 비디오 provider로 사용하려면:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

공유 도구
파라미터, provider 선택, 장애 조치 동작은 [Video Generation](/tools/video-generation)을 참고하세요.

## 음악 생성

번들된 `google` plugin은 공유
`music_generate` 도구를 통해 음악 생성도 등록합니다.

- 기본 음악 모델: `google/lyria-3-clip-preview`
- `google/lyria-3-pro-preview`도 지원
- 프롬프트 제어: `lyrics`, `instrumental`
- 출력 형식: 기본 `mp3`, `google/lyria-3-pro-preview`에서는 `wav`도 지원
- 참조 입력: 최대 10개 이미지
- 세션 기반 실행은 `action: "status"`를 포함한 공유 task/status 흐름을 통해 분리 실행됨

Google을 기본 음악 provider로 사용하려면:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

공유 도구
파라미터, provider 선택, 장애 조치 동작은 [Music Generation](/tools/music-generation)을 참고하세요.

## 환경 참고

Gateway가 데몬(launchd/systemd)으로 실행되는 경우 `GEMINI_API_KEY`가
해당 프로세스에서 사용 가능해야 합니다(예: `~/.openclaw/.env` 또는
`env.shellEnv`를 통해).
