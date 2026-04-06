---
read_when:
    - 에이전트를 통해 음악 또는 오디오를 생성하는 경우
    - 음악 생성 provider 및 모델을 구성하는 경우
    - '`music_generate` 도구 파라미터를 이해하려는 경우'
summary: 워크플로 기반 plugin을 포함한 공통 provider로 음악 생성하기
title: 음악 생성
x-i18n:
    generated_at: "2026-04-06T03:13:17Z"
    model: gpt-5.4
    provider: openai
    source_hash: a03de8aa75cfb7248eb0c1d969fb2a6da06117967d097e6f6e95771d0f017ae1
    source_path: tools/music-generation.md
    workflow: 15
---

# 음악 생성

`music_generate` 도구를 사용하면 에이전트가 Google,
MiniMax, 워크플로 기반으로 구성된 ComfyUI 같은 provider를 통해 공통 음악 생성 capability를 사용하여 음악이나 오디오를 만들 수 있습니다.

공통 provider 기반 에이전트 세션의 경우, OpenClaw는 음악 생성을
백그라운드 작업으로 시작하고, 이를 task ledger에 추적한 뒤,
트랙이 준비되면 에이전트를 다시 깨워 완성된 오디오를 원래
채널에 다시 게시할 수 있게 합니다.

<Note>
하나 이상의 음악 생성 provider를 사용할 수 있을 때만 내장 공통 도구가 표시됩니다. 에이전트 도구에서 `music_generate`가 보이지 않는다면 `agents.defaults.musicGenerationModel`을 구성하거나 provider API 키를 설정하세요.
</Note>

## 빠른 시작

### 공통 provider 기반 생성

1. 예를 들어 `GEMINI_API_KEY` 또는
   `MINIMAX_API_KEY`처럼 최소 하나의 provider에 대한 API 키를 설정합니다.
2. 선택 사항으로 선호 모델을 설정합니다:

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

3. 에이전트에게 다음과 같이 요청합니다: _"네온 도시를 가로지르는 야간 드라이브에 대한 경쾌한 신스팝 트랙을 생성해줘."_

에이전트가 `music_generate`를 자동으로 호출합니다. 도구 allow-list 설정은 필요하지 않습니다.

세션 기반 에이전트 실행이 없는 직접 동기 컨텍스트의 경우에도, 내장
도구는 여전히 인라인 생성으로 폴백하고 도구 결과에 최종 미디어 경로를 반환합니다.

예시 프롬프트:

```text
보컬 없이 부드러운 스트링이 들어간 영화풍 피아노 트랙을 생성해줘.
```

```text
일출에 로켓을 발사하는 내용을 담은 에너지 넘치는 칩튠 루프를 생성해줘.
```

### 워크플로 기반 Comfy 생성

번들 `comfy` plugin은
음악 생성 provider registry를 통해 공통 `music_generate` 도구에 연결됩니다.

1. 워크플로 JSON과
   prompt/output 노드로 `models.providers.comfy.music`를 구성합니다.
2. Comfy Cloud를 사용하는 경우 `COMFY_API_KEY` 또는 `COMFY_CLOUD_API_KEY`를 설정합니다.
3. 에이전트에게 음악을 요청하거나 도구를 직접 호출합니다.

예시:

```text
/tool music_generate prompt="부드러운 테이프 질감이 있는 따뜻한 앰비언트 신스 루프"
```

## 공통 번들 provider 지원

| Provider | 기본 모델             | 참조 입력      | 지원되는 제어 항목                                      | API 키                                 |
| -------- | --------------------- | -------------- | ------------------------------------------------------- | -------------------------------------- |
| ComfyUI  | `workflow`            | 최대 이미지 1개 | 워크플로 정의 음악 또는 오디오                          | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| Google   | `lyria-3-clip-preview` | 최대 이미지 10개 | `lyrics`, `instrumental`, `format`                      | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax  | `music-2.5+`          | 없음           | `lyrics`, `instrumental`, `durationSeconds`, `format=mp3` | `MINIMAX_API_KEY`                    |

런타임에 사용 가능한 공통 provider와 모델을 확인하려면
`action: "list"`를 사용하세요:

```text
/tool music_generate action=list
```

활성 세션 기반 음악 task를 확인하려면
`action: "status"`를 사용하세요:

```text
/tool music_generate action=status
```

직접 생성 예시:

```text
/tool music_generate prompt="비닐 질감과 잔잔한 빗소리가 있는 몽환적인 로파이 힙합" instrumental=true
```

## 내장 도구 파라미터

| 파라미터          | 타입     | 설명                                                                                           |
| ----------------- | -------- | ---------------------------------------------------------------------------------------------- |
| `prompt`          | string   | 음악 생성 프롬프트(`action: "generate"`에 필수)                                                |
| `action`          | string   | `"generate"`(기본값), 현재 세션 task용 `"status"`, 또는 provider를 확인하는 `"list"`           |
| `model`           | string   | provider/model override. 예: `google/lyria-3-pro-preview` 또는 `comfy/workflow`                |
| `lyrics`          | string   | provider가 명시적 가사 입력을 지원할 때 사용하는 선택적 가사                                   |
| `instrumental`    | boolean  | provider가 지원할 때 반주 전용 출력을 요청                                                     |
| `image`           | string   | 단일 참조 이미지 경로 또는 URL                                                                 |
| `images`          | string[] | 다중 참조 이미지(최대 10개)                                                                    |
| `durationSeconds` | number   | provider가 길이 힌트를 지원할 때의 목표 길이(초)                                               |
| `format`          | string   | provider가 지원할 때의 출력 형식 힌트(`mp3` 또는 `wav`)                                        |
| `filename`        | string   | 출력 파일명 힌트                                                                               |

모든 provider가 모든 파라미터를 지원하는 것은 아닙니다. OpenClaw는 제출 전 입력 개수 같은 하드 제한은 여전히 검증하지만, 선택한 provider 또는 모델이 지원하지 않는 선택적 힌트는 경고와 함께 무시됩니다.

## 공통 provider 기반 경로의 비동기 동작

- 세션 기반 에이전트 실행: `music_generate`는 백그라운드 task를 생성하고, 즉시 시작됨/task 응답을 반환한 뒤, 나중에 후속 에이전트 메시지에서 완성된 트랙을 게시합니다.
- 중복 방지: 해당 백그라운드 task가 같은 세션에서 아직 `queued` 또는 `running` 상태이면, 이후 `music_generate` 호출은 새 생성을 시작하는 대신 task 상태를 반환합니다.
- 상태 조회: 새 작업을 시작하지 않고 활성 세션 기반 음악 task를 확인하려면 `action: "status"`를 사용하세요.
- task 추적: 생성의 대기 중, 실행 중, 종료 상태를 확인하려면 `openclaw tasks list` 또는 `openclaw tasks show <taskId>`를 사용하세요.
- 완료 후 재개: OpenClaw는 동일한 세션에 내부 완료 이벤트를 다시 주입하여 모델이 사용자에게 보일 후속 메시지를 직접 작성할 수 있게 합니다.
- 프롬프트 힌트: 같은 세션에서 이후 사용자/수동 턴에는 음악 task가 이미 진행 중일 때 작은 런타임 힌트가 제공되어, 모델이 `music_generate`를 무작정 다시 호출하지 않게 합니다.
- 세션 없는 폴백: 실제 에이전트 세션이 없는 직접/로컬 컨텍스트에서는 여전히 인라인으로 실행되며 같은 턴에 최종 오디오 결과를 반환합니다.

## 구성

### 모델 선택

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["minimax/music-2.5+"],
      },
    },
  },
}
```

### Provider 선택 순서

음악 생성 시 OpenClaw는 다음 순서로 provider를 시도합니다:

1. 에이전트가 지정한 경우 도구 호출의 `model` 파라미터
2. config의 `musicGenerationModel.primary`
3. 순서대로 `musicGenerationModel.fallbacks`
4. auth 기반 provider 기본값만 사용한 자동 탐지:
   - 현재 기본 provider 먼저
   - 나머지 등록된 음악 생성 provider를 provider-id 순서로

어떤 provider가 실패하면 다음 후보를 자동으로 시도합니다. 모두 실패하면
오류에는 각 시도의 세부 정보가 포함됩니다.

## Provider 참고

- Google은 Lyria 3 배치 생성을 사용합니다. 현재 번들 흐름은
  프롬프트, 선택적 가사 텍스트, 선택적 참조 이미지를 지원합니다.
- MiniMax는 배치 `music_generation` 엔드포인트를 사용합니다. 현재 번들 흐름은
  프롬프트, 선택적 가사, 반주 모드, 길이 조정,
  mp3 출력을 지원합니다.
- ComfyUI 지원은 워크플로 기반이며 구성된 그래프와
  prompt/output 필드에 대한 노드 매핑에 따라 달라집니다.

## 올바른 경로 선택하기

- 모델 선택, provider 장애 조치, 내장 비동기 task/상태 흐름이 필요하다면 공통 provider 기반 경로를 사용하세요.
- 커스텀 워크플로 그래프가 필요하거나 공통 번들 음악 capability에 포함되지 않은 provider가 필요하다면 ComfyUI 같은 plugin 경로를 사용하세요.
- ComfyUI 전용 동작을 디버깅하는 경우 [ComfyUI](/providers/comfy)를 참조하세요. 공통 provider 동작을 디버깅하는 경우 [Google (Gemini)](/ko/providers/google) 또는 [MiniMax](/ko/providers/minimax)부터 시작하세요.

## 라이브 테스트

공통 번들 provider에 대한 옵트인 라이브 커버리지:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

번들 ComfyUI 음악 경로에 대한 옵트인 라이브 커버리지:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

Comfy 라이브 파일은 해당 섹션이 구성되어 있으면 comfy 이미지 및 비디오 워크플로도 함께 다룹니다.

## 관련 항목

- [Background Tasks](/ko/automation/tasks) - 분리된 `music_generate` 실행을 위한 task 추적
- [Configuration Reference](/ko/gateway/configuration-reference#agent-defaults) - `musicGenerationModel` config
- [ComfyUI](/providers/comfy)
- [Google (Gemini)](/ko/providers/google)
- [MiniMax](/ko/providers/minimax)
- [Models](/ko/concepts/models) - 모델 구성 및 장애 조치
- [Tools Overview](/ko/tools)
