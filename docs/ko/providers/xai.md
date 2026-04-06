---
read_when:
    - OpenClaw에서 Grok 모델을 사용하려는 경우
    - xAI auth 또는 모델 ID를 구성하는 경우
summary: OpenClaw에서 xAI Grok 모델 사용하기
title: xAI
x-i18n:
    generated_at: "2026-04-06T03:11:45Z"
    model: gpt-5.4
    provider: openai
    source_hash: 64bc899655427cc10bdc759171c7d1ec25ad9f1e4f9d803f1553d3d586c6d71d
    source_path: providers/xai.md
    workflow: 15
---

# xAI

OpenClaw는 Grok 모델용 번들 `xai` provider plugin을 제공합니다.

## 설정

1. xAI 콘솔에서 API 키를 생성합니다.
2. `XAI_API_KEY`를 설정하거나 다음을 실행합니다:

```bash
openclaw onboard --auth-choice xai-api-key
```

3. 다음과 같은 모델을 선택합니다:

```json5
{
  agents: { defaults: { model: { primary: "xai/grok-4" } } },
}
```

이제 OpenClaw는 번들 xAI 전송 계층으로 xAI Responses API를 사용합니다. 동일한
`XAI_API_KEY`는 Grok 기반 `web_search`, 일급 `x_search`,
원격 `code_execution`에도 사용할 수 있습니다.
`plugins.entries.xai.config.webSearch.apiKey` 아래에 xAI 키를 저장하면,
번들 xAI 모델 provider도 이제 이를 폴백으로 재사용합니다.
`code_execution` 튜닝은 `plugins.entries.xai.config.codeExecution` 아래에 있습니다.

## 현재 번들 모델 카탈로그

이제 OpenClaw에는 다음 xAI 모델 계열이 기본 포함됩니다:

- `grok-3`, `grok-3-fast`, `grok-3-mini`, `grok-3-mini-fast`
- `grok-4`, `grok-4-0709`
- `grok-4-fast`, `grok-4-fast-non-reasoning`
- `grok-4-1-fast`, `grok-4-1-fast-non-reasoning`
- `grok-4.20-beta-latest-reasoning`, `grok-4.20-beta-latest-non-reasoning`
- `grok-code-fast-1`

이 plugin은 동일한 API 형태를 따르는 더 새로운 `grok-4*` 및 `grok-code-fast*` ID도
forward-resolve합니다.

고속 모델 참고:

- `grok-4-fast`, `grok-4-1-fast`, `grok-4.20-beta-*` 변형은
  번들 카탈로그에서 현재 이미지 지원이 가능한 Grok ref입니다.
- `/fast on` 또는 `agents.defaults.models["xai/<model>"].params.fastMode: true`는
  네이티브 xAI 요청을 다음과 같이 다시 씁니다:
  - `grok-3` -> `grok-3-fast`
  - `grok-3-mini` -> `grok-3-mini-fast`
  - `grok-4` -> `grok-4-fast`
  - `grok-4-0709` -> `grok-4-fast`

레거시 호환성 별칭은 여전히 정식 번들 ID로 정규화됩니다. 예를
들면:

- `grok-4-fast-reasoning` -> `grok-4-fast`
- `grok-4-1-fast-reasoning` -> `grok-4-1-fast`
- `grok-4.20-reasoning` -> `grok-4.20-beta-latest-reasoning`
- `grok-4.20-non-reasoning` -> `grok-4.20-beta-latest-non-reasoning`

## 웹 검색

번들 `grok` web-search provider도 `XAI_API_KEY`를 사용합니다:

```bash
openclaw config set tools.web.search.provider grok
```

## 비디오 생성

번들 `xai` plugin은 공통
`video_generate` 도구를 통해 비디오 생성도 등록합니다.

- 기본 비디오 모델: `xai/grok-imagine-video`
- 모드: 텍스트-비디오, 이미지-비디오, 원격 비디오 편집/확장 흐름
- `aspectRatio` 및 `resolution` 지원
- 현재 제한: 로컬 비디오 버퍼는 허용되지 않으며, 비디오 참조/편집 입력에는 원격 `http(s)`
  URL을 사용해야 합니다

xAI를 기본 비디오 provider로 사용하려면:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "xai/grok-imagine-video",
      },
    },
  },
}
```

공통 도구
파라미터, provider 선택, 장애 조치 동작은 [Video Generation](/tools/video-generation)을 참조하세요.

## 알려진 제한 사항

- 현재 auth는 API 키 전용입니다. OpenClaw에는 아직 xAI OAuth/device-code 흐름이 없습니다.
- `grok-4.20-multi-agent-experimental-beta-0304`는 표준 OpenClaw xAI 전송 계층과 다른 업스트림 API 표면이 필요하므로 일반 xAI provider 경로에서는 지원되지 않습니다.

## 참고

- OpenClaw는 공통 러너 경로에서 xAI 전용 도구 스키마 및 도구 호출 호환성 수정을 자동으로 적용합니다.
- 네이티브 xAI 요청은 기본적으로 `tool_stream: true`를 사용합니다.
  이를 비활성화하려면
  `agents.defaults.models["xai/<model>"].params.tool_stream`을 `false`로 설정하세요.
- 번들 xAI 래퍼는 네이티브 xAI 요청을 보내기 전에
  지원되지 않는 엄격한 도구 스키마 플래그와 reasoning payload 키를 제거합니다.
- `web_search`, `x_search`, `code_execution`은 OpenClaw 도구로 노출됩니다. OpenClaw는 각 채팅 턴마다 모든 네이티브 도구를 붙이는 대신, 각 도구 요청 안에서 필요한 특정 xAI 내장 기능만 활성화합니다.
- `x_search`와 `code_execution`은 코어 모델 런타임에 하드코딩된 것이 아니라 번들 xAI plugin이 소유합니다.
- `code_execution`은 로컬 [`exec`](/ko/tools/exec)가 아니라 원격 xAI sandbox 실행입니다.
- 더 넓은 provider 개요는 [Model providers](/ko/providers/index)를 참조하세요.
