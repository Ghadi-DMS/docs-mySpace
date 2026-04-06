---
read_when:
    - OpenClaw에서 Vydra 미디어 생성을 사용하고 싶을 때
    - Vydra API 키 설정 안내가 필요할 때
summary: OpenClaw에서 Vydra 이미지, 비디오, 음성을 사용하기
title: Vydra
x-i18n:
    generated_at: "2026-04-06T03:11:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0fe999e8a5414b8a31a6d7d127bc6bcfc3b4492b8f438ab17dfa9680c5b079b7
    source_path: providers/vydra.md
    workflow: 15
---

# Vydra

번들된 Vydra plugin은 다음을 추가합니다.

- `vydra/grok-imagine`를 통한 이미지 생성
- `vydra/veo3` 및 `vydra/kling`을 통한 비디오 생성
- Vydra의 ElevenLabs 기반 TTS 경로를 통한 음성 합성

OpenClaw는 세 가지 기능 모두에 동일한 `VYDRA_API_KEY`를 사용합니다.

## 중요한 기본 URL

`https://www.vydra.ai/api/v1`을 사용하세요.

Vydra의 apex 호스트(`https://vydra.ai/api/v1`)는 현재 `www`로 리디렉션됩니다. 일부 HTTP 클라이언트는 이 교차 호스트 리디렉션에서 `Authorization`을 유지하지 않기 때문에, 유효한 API 키가 오해를 부르는 인증 실패처럼 보일 수 있습니다. 번들 plugin은 이를 피하기 위해 `www` 기본 URL을 직접 사용합니다.

## 설정

대화형 온보딩:

```bash
openclaw onboard --auth-choice vydra-api-key
```

또는 env var를 직접 설정:

```bash
export VYDRA_API_KEY="vydra_live_..."
```

## 이미지 생성

기본 이미지 모델:

- `vydra/grok-imagine`

이를 기본 이미지 provider로 설정:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "vydra/grok-imagine",
      },
    },
  },
}
```

현재 번들 지원은 텍스트-이미지 전용입니다. Vydra의 호스팅된 편집 경로는 원격 이미지 URL을 기대하며, OpenClaw는 아직 번들 plugin에 Vydra 전용 업로드 브리지를 추가하지 않았습니다.

공유 도구 동작은 [Image Generation](/ko/tools/image-generation)을 참고하세요.

## 비디오 생성

등록된 비디오 모델:

- 텍스트-비디오용 `vydra/veo3`
- 이미지-비디오용 `vydra/kling`

Vydra를 기본 비디오 provider로 설정:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "vydra/veo3",
      },
    },
  },
}
```

참고:

- `vydra/veo3`는 번들에서 텍스트-비디오 전용으로 제공됩니다.
- `vydra/kling`은 현재 원격 이미지 URL 참조가 필요합니다. 로컬 파일 업로드는 초기 단계에서 거부됩니다.
- 번들 plugin은 보수적으로 동작하며 종횡비, 해상도, 워터마크, 생성된 오디오 같은 문서화되지 않은 스타일 knob는 전달하지 않습니다.

공유 도구 동작은 [Video Generation](/tools/video-generation)을 참고하세요.

## 음성 합성

Vydra를 음성 provider로 설정:

```json5
{
  messages: {
    tts: {
      provider: "vydra",
      providers: {
        vydra: {
          apiKey: "${VYDRA_API_KEY}",
          voiceId: "21m00Tcm4TlvDq8ikWAM",
        },
      },
    },
  },
}
```

기본값:

- 모델: `elevenlabs/tts`
- voice id: `21m00Tcm4TlvDq8ikWAM`

번들 plugin은 현재 하나의 검증된 기본 voice를 노출하며 MP3 오디오 파일을 반환합니다.

## 관련 문서

- [Provider Directory](/ko/providers/index)
- [Image Generation](/ko/tools/image-generation)
- [Video Generation](/tools/video-generation)
