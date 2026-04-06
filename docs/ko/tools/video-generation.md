---
read_when:
    - 에이전트를 통해 비디오를 생성하는 경우
    - 비디오 생성 provider 및 모델을 구성하는 경우
    - '`video_generate` 도구 파라미터를 이해하려는 경우'
summary: 12개 provider 백엔드를 사용해 텍스트, 이미지, 기존 비디오로부터 비디오 생성하기
title: 비디오 생성
x-i18n:
    generated_at: "2026-04-06T03:13:48Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4afec87368232221db1aa5a3980254093d6a961b17271b2dcbf724e6bd455b16
    source_path: tools/video-generation.md
    workflow: 15
---

# 비디오 생성

OpenClaw 에이전트는 텍스트 프롬프트, 참조 이미지, 기존 비디오로부터 비디오를 생성할 수 있습니다. 12개의 provider 백엔드가 지원되며, 각각 서로 다른 모델 옵션, 입력 모드, 기능 세트를 제공합니다. 에이전트는 구성과 사용 가능한 API 키를 기준으로 적절한 provider를 자동으로 선택합니다.

<Note>
`video_generate` 도구는 하나 이상의 비디오 생성 provider를 사용할 수 있을 때만 표시됩니다. 에이전트 도구에서 이것이 보이지 않는다면 provider API 키를 설정하거나 `agents.defaults.videoGenerationModel`을 구성하세요.
</Note>

## 빠른 시작

1. 지원되는 provider 중 하나에 대한 API 키를 설정합니다:

```bash
export GEMINI_API_KEY="your-key"
```

2. 선택 사항으로 기본 모델을 고정합니다:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "google/veo-3.1-fast-generate-preview"
```

3. 에이전트에게 요청합니다:

> 해질녘에 친근한 바닷가재가 서핑하는 5초짜리 영화풍 비디오를 생성해줘.

에이전트가 `video_generate`를 자동으로 호출합니다. 도구 allowlist 설정은 필요하지 않습니다.

## 비디오를 생성하면 어떤 일이 일어나는가

비디오 생성은 비동기 방식입니다. 세션에서 에이전트가 `video_generate`를 호출하면:

1. OpenClaw는 요청을 provider에 제출하고 즉시 task ID를 반환합니다.
2. provider는 백그라운드에서 작업을 처리합니다(보통 provider와 해상도에 따라 30초에서 5분 정도 소요).
3. 비디오가 준비되면 OpenClaw는 내부 완료 이벤트로 동일한 세션을 다시 깨웁니다.
4. 에이전트가 완성된 비디오를 원래 대화에 다시 게시합니다.

작업이 진행 중인 동안 같은 세션에서 중복된 `video_generate` 호출은 새 생성을 시작하는 대신 현재 task 상태를 반환합니다. CLI에서 진행 상황을 확인하려면 `openclaw tasks list` 또는 `openclaw tasks show <taskId>`를 사용하세요.

세션 기반 에이전트 실행이 아닌 경우(예: 직접 도구 호출), 도구는 인라인 생성으로 폴백하고 같은 턴에서 최종 미디어 경로를 반환합니다.

## 지원되는 provider

| Provider | 기본 모델                        | 텍스트 | 이미지 참조       | 비디오 참조       | API 키                                   |
| -------- | -------------------------------- | ------ | ----------------- | ----------------- | ---------------------------------------- |
| Alibaba  | `wan2.6-t2v`                     | 예     | 예 (원격 URL)     | 예 (원격 URL)     | `MODELSTUDIO_API_KEY`                    |
| BytePlus | `seedance-1-0-lite-t2v-250428`   | 예     | 이미지 1개        | 아니요            | `BYTEPLUS_API_KEY`                       |
| ComfyUI  | `workflow`                       | 예     | 이미지 1개        | 아니요            | `COMFY_API_KEY` 또는 `COMFY_CLOUD_API_KEY` |
| fal      | `fal-ai/minimax/video-01-live`   | 예     | 이미지 1개        | 아니요            | `FAL_KEY`                                |
| Google   | `veo-3.1-fast-generate-preview`  | 예     | 이미지 1개        | 비디오 1개        | `GEMINI_API_KEY`                         |
| MiniMax  | `MiniMax-Hailuo-2.3`             | 예     | 이미지 1개        | 아니요            | `MINIMAX_API_KEY`                        |
| OpenAI   | `sora-2`                         | 예     | 이미지 1개        | 비디오 1개        | `OPENAI_API_KEY`                         |
| Qwen     | `wan2.6-t2v`                     | 예     | 예 (원격 URL)     | 예 (원격 URL)     | `QWEN_API_KEY`                           |
| Runway   | `gen4.5`                         | 예     | 이미지 1개        | 비디오 1개        | `RUNWAYML_API_SECRET`                    |
| Together | `Wan-AI/Wan2.2-T2V-A14B`         | 예     | 이미지 1개        | 아니요            | `TOGETHER_API_KEY`                       |
| Vydra    | `veo3`                           | 예     | 이미지 1개 (`kling`) | 아니요         | `VYDRA_API_KEY`                          |
| xAI      | `grok-imagine-video`             | 예     | 이미지 1개        | 비디오 1개        | `XAI_API_KEY`                            |

일부 provider는 추가 또는 대체 API 키 env var도 허용합니다. 자세한 내용은 각 [provider 페이지](#related)를 참조하세요.

런타임에 사용 가능한 provider와 모델을 확인하려면 `video_generate action=list`를 실행하세요.

## 도구 파라미터

### 필수

| 파라미터 | 타입   | 설명                                                                      |
| -------- | ------ | ------------------------------------------------------------------------- |
| `prompt` | string | 생성할 비디오의 텍스트 설명(`action: "generate"`에 필수)                  |

### 콘텐츠 입력

| 파라미터 | 타입     | 설명                              |
| -------- | -------- | --------------------------------- |
| `image`  | string   | 단일 참조 이미지(경로 또는 URL)   |
| `images` | string[] | 다중 참조 이미지(최대 5개)        |
| `video`  | string   | 단일 참조 비디오(경로 또는 URL)   |
| `videos` | string[] | 다중 참조 비디오(최대 4개)        |

### 스타일 제어

| 파라미터          | 타입    | 설명                                                                   |
| ----------------- | ------- | ---------------------------------------------------------------------- |
| `aspectRatio`     | string  | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` |
| `resolution`      | string  | `480P`, `720P`, 또는 `1080P`                                           |
| `durationSeconds` | number  | 목표 길이(초). 가장 가까운 provider 지원 값으로 반올림됨              |
| `size`            | string  | provider가 지원할 때의 크기 힌트                                      |
| `audio`           | boolean | 지원되는 경우 생성 오디오 활성화                                      |
| `watermark`       | boolean | 지원되는 경우 provider 워터마크 토글                                  |

### 고급

| 파라미터 | 타입   | 설명                                         |
| -------- | ------ | -------------------------------------------- |
| `action` | string | `"generate"`(기본값), `"status"`, 또는 `"list"` |
| `model`  | string | provider/model override(예: `runway/gen4.5`) |
| `filename` | string | 출력 파일명 힌트                           |

모든 provider가 모든 파라미터를 지원하는 것은 아닙니다. 지원되지 않는 override는 가능한 범위에서 무시되며 도구 결과에 경고로 보고됩니다. 하드 capability 제한(예: 참조 입력이 너무 많음)은 제출 전에 실패합니다.

## 액션

- **generate** (기본값) -- 지정된 프롬프트와 선택적 참조 입력으로 비디오를 생성합니다.
- **status** -- 새 생성을 시작하지 않고 현재 세션의 진행 중인 비디오 task 상태를 확인합니다.
- **list** -- 사용 가능한 provider, 모델, capability를 표시합니다.

## 모델 선택

비디오를 생성할 때 OpenClaw는 다음 순서로 모델을 해석합니다:

1. **`model` 도구 파라미터** -- 에이전트가 호출에서 이를 지정한 경우.
2. **`videoGenerationModel.primary`** -- config에서 가져옴.
3. **`videoGenerationModel.fallbacks`** -- 순서대로 시도.
4. **자동 탐지** -- 유효한 auth가 있는 provider를 사용하며, 현재 기본 provider부터 시작하고 이후 나머지 provider를 알파벳 순으로 시도.

어떤 provider가 실패하면 다음 후보를 자동으로 시도합니다. 모든 후보가 실패하면 오류에 각 시도의 세부 정보가 포함됩니다.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
      },
    },
  },
}
```

## Provider 참고

| Provider | 참고 |
| -------- | ---- |
| Alibaba  | DashScope/Model Studio 비동기 엔드포인트를 사용합니다. 참조 이미지와 비디오는 원격 `http(s)` URL이어야 합니다. |
| BytePlus | 단일 이미지 참조만 지원합니다. |
| ComfyUI  | 워크플로 기반 로컬 또는 클라우드 실행입니다. 구성된 그래프를 통해 텍스트-비디오 및 이미지-비디오를 지원합니다. |
| fal      | 장시간 작업에 queue 기반 흐름을 사용합니다. 단일 이미지 참조만 지원합니다. |
| Google   | Gemini/Veo를 사용합니다. 이미지 1개 또는 비디오 1개 참조를 지원합니다. |
| MiniMax  | 단일 이미지 참조만 지원합니다. |
| OpenAI   | `size` override만 전달됩니다. 다른 스타일 override(`aspectRatio`, `resolution`, `audio`, `watermark`)는 경고와 함께 무시됩니다. |
| Qwen     | Alibaba와 동일한 DashScope 백엔드를 사용합니다. 참조 입력은 원격 `http(s)` URL이어야 하며 로컬 파일은 사전에 거부됩니다. |
| Runway   | data URI를 통해 로컬 파일을 지원합니다. 비디오-투-비디오에는 `runway/gen4_aleph`가 필요합니다. 텍스트 전용 실행은 `16:9` 및 `9:16` 종횡비를 노출합니다. |
| Together | 단일 이미지 참조만 지원합니다. |
| Vydra    | auth가 제거되는 리다이렉트를 피하기 위해 `https://www.vydra.ai/api/v1`를 직접 사용합니다. `veo3`는 번들에서 텍스트-비디오 전용이며, `kling`은 원격 이미지 URL이 필요합니다. |
| xAI      | 텍스트-비디오, 이미지-비디오, 원격 비디오 편집/확장 흐름을 지원합니다. |

## 구성

OpenClaw config에서 기본 비디오 생성 모델을 설정합니다:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

또는 CLI로 설정:

```bash
openclaw config set agents.defaults.videoGenerationModel.primary "qwen/wan2.6-t2v"
```

## 관련 항목

- [Tools Overview](/ko/tools)
- [Background Tasks](/ko/automation/tasks) -- 비동기 비디오 생성을 위한 task 추적
- [Alibaba Model Studio](/providers/alibaba)
- [BytePlus](/providers/byteplus)
- [ComfyUI](/providers/comfy)
- [fal](/providers/fal)
- [Google (Gemini)](/ko/providers/google)
- [MiniMax](/ko/providers/minimax)
- [OpenAI](/ko/providers/openai)
- [Qwen](/ko/providers/qwen)
- [Runway](/providers/runway)
- [Together AI](/ko/providers/together)
- [Vydra](/providers/vydra)
- [xAI](/ko/providers/xai)
- [Configuration Reference](/ko/gateway/configuration-reference#agent-defaults)
- [Models](/ko/concepts/models)
