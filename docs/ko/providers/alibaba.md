---
read_when:
    - OpenClaw에서 Alibaba Wan 비디오 생성을 사용하고 싶을 때
    - 비디오 생성을 위한 Model Studio 또는 DashScope API key 설정이 필요할 때
summary: OpenClaw에서의 Alibaba Model Studio Wan 비디오 생성
title: Alibaba Model Studio
x-i18n:
    generated_at: "2026-04-06T03:10:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 97a1eddc7cbd816776b9368f2a926b5ef9ee543f08d151a490023736f67dc635
    source_path: providers/alibaba.md
    workflow: 15
---

# Alibaba Model Studio

OpenClaw는 Alibaba Model Studio / DashScope의 Wan 모델용 번들 `alibaba`
video-generation provider를 제공합니다.

- Provider: `alibaba`
- 권장 auth: `MODELSTUDIO_API_KEY`
- 추가로 허용됨: `DASHSCOPE_API_KEY`, `QWEN_API_KEY`
- API: DashScope / Model Studio 비동기 비디오 생성

## 빠른 시작

1. API key를 설정합니다.

```bash
openclaw onboard --auth-choice qwen-standard-api-key
```

2. 기본 비디오 모델을 설정합니다.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "alibaba/wan2.6-t2v",
      },
    },
  },
}
```

## 내장 Wan 모델

번들 `alibaba` provider는 현재 다음을 등록합니다.

- `alibaba/wan2.6-t2v`
- `alibaba/wan2.6-i2v`
- `alibaba/wan2.6-r2v`
- `alibaba/wan2.6-r2v-flash`
- `alibaba/wan2.7-r2v`

## 현재 제한 사항

- 요청당 최대 **1개** 출력 비디오
- 최대 **1개** 입력 이미지
- 최대 **4개** 입력 비디오
- 최대 **10초** 길이
- `size`, `aspectRatio`, `resolution`, `audio`, `watermark` 지원
- 참조 이미지/비디오 모드는 현재 **원격 http(s) URL**이 필요함

## Qwen과의 관계

번들 `qwen` provider도 Wan 비디오 생성을 위해 Alibaba 호스팅 DashScope endpoint를 사용합니다. 다음과 같이 사용하세요.

- 정식 Qwen provider 표면을 원하면 `qwen/...`
- vendor가 직접 소유하는 Wan 비디오 표면을 원하면 `alibaba/...`

## 관련 문서

- [비디오 생성](/tools/video-generation)
- [Qwen](/ko/providers/qwen)
- [Configuration Reference](/ko/gateway/configuration-reference#agent-defaults)
