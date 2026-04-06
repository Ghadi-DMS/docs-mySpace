---
read_when:
    - OpenClaw와 함께 로컬 ComfyUI 워크플로를 사용하고 싶을 때
    - 이미지, 비디오 또는 음악 워크플로와 함께 Comfy Cloud를 사용하고 싶을 때
    - 번들 comfy plugin config 키가 필요할 때
summary: OpenClaw에서의 ComfyUI 워크플로 이미지, 비디오, 음악 생성 설정
title: ComfyUI
x-i18n:
    generated_at: "2026-04-06T03:11:11Z"
    model: gpt-5.4
    provider: openai
    source_hash: e645f32efdffdf4cd498684f1924bb953a014d3656b48f4b503d64e38c61ba9c
    source_path: providers/comfy.md
    workflow: 15
---

# ComfyUI

OpenClaw는 워크플로 기반 ComfyUI 실행을 위한 번들 `comfy` plugin을 제공합니다.

- Provider: `comfy`
- 모델: `comfy/workflow`
- 공용 표면: `image_generate`, `video_generate`, `music_generate`
- Auth: 로컬 ComfyUI는 없음, Comfy Cloud는 `COMFY_API_KEY` 또는 `COMFY_CLOUD_API_KEY`
- API: ComfyUI `/prompt` / `/history` / `/view` 및 Comfy Cloud `/api/*`

## 지원 내용

- 워크플로 JSON으로부터의 이미지 생성
- 업로드된 참조 이미지 1개를 사용하는 이미지 편집
- 워크플로 JSON으로부터의 비디오 생성
- 업로드된 참조 이미지 1개를 사용하는 비디오 생성
- 공용 `music_generate` 도구를 통한 음악 또는 오디오 생성
- 설정된 노드 또는 일치하는 모든 출력 노드에서의 출력 다운로드

번들 plugin은 워크플로 기반이므로, OpenClaw는 일반적인
`size`, `aspectRatio`, `resolution`, `durationSeconds`, 또는 TTS 스타일 제어를
사용자 그래프에 매핑하려고 하지 않습니다.

## Config 레이아웃

Comfy는 공용 최상위 연결 설정과 capability별 워크플로
섹션을 지원합니다.

```json5
{
  models: {
    providers: {
      comfy: {
        mode: "local",
        baseUrl: "http://127.0.0.1:8188",
        image: {
          workflowPath: "./workflows/flux-api.json",
          promptNodeId: "6",
          outputNodeId: "9",
        },
        video: {
          workflowPath: "./workflows/video-api.json",
          promptNodeId: "12",
          outputNodeId: "21",
        },
        music: {
          workflowPath: "./workflows/music-api.json",
          promptNodeId: "3",
          outputNodeId: "18",
        },
      },
    },
  },
}
```

공용 키:

- `mode`: `local` 또는 `cloud`
- `baseUrl`: 로컬은 기본값 `http://127.0.0.1:8188`, 클라우드는 `https://cloud.comfy.org`
- `apiKey`: env var 대신 사용할 수 있는 선택적 인라인 key
- `allowPrivateNetwork`: cloud 모드에서 private/LAN `baseUrl` 허용

`image`, `video`, 또는 `music` 아래 capability별 키:

- `workflow` 또는 `workflowPath`: 필수
- `promptNodeId`: 필수
- `promptInputName`: 기본값 `text`
- `outputNodeId`: 선택 사항
- `pollIntervalMs`: 선택 사항
- `timeoutMs`: 선택 사항

이미지 및 비디오 섹션은 다음도 지원합니다.

- `inputImageNodeId`: 참조 이미지를 전달할 때 필수
- `inputImageInputName`: 기본값 `image`

## 하위 호환성

기존 최상위 이미지 config도 계속 동작합니다.

```json5
{
  models: {
    providers: {
      comfy: {
        workflowPath: "./workflows/flux-api.json",
        promptNodeId: "6",
        outputNodeId: "9",
      },
    },
  },
}
```

OpenClaw는 이 legacy 형태를 이미지 워크플로 config로 취급합니다.

## 이미지 워크플로

기본 이미지 모델 설정:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "comfy/workflow",
      },
    },
  },
}
```

참조 이미지 편집 예시:

```json5
{
  models: {
    providers: {
      comfy: {
        image: {
          workflowPath: "./workflows/edit-api.json",
          promptNodeId: "6",
          inputImageNodeId: "7",
          inputImageInputName: "image",
          outputNodeId: "9",
        },
      },
    },
  },
}
```

## 비디오 워크플로

기본 비디오 모델 설정:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "comfy/workflow",
      },
    },
  },
}
```

Comfy 비디오 워크플로는 현재 설정된 그래프를 통해 텍스트-투-비디오와 이미지-투-비디오를 지원합니다. OpenClaw는 입력 비디오를 Comfy 워크플로에 전달하지 않습니다.

## 음악 워크플로

번들 plugin은 워크플로로 정의된 오디오 또는 음악 출력을 위한 music-generation provider를 등록하며, 공용 `music_generate` 도구를 통해 노출됩니다.

```text
/tool music_generate prompt="Warm ambient synth loop with soft tape texture"
```

오디오 워크플로 JSON과 출력 노드를 가리키도록 `music` config 섹션을 사용하세요.

## Comfy Cloud

`mode: "cloud"`와 다음 중 하나를 사용하세요.

- `COMFY_API_KEY`
- `COMFY_CLOUD_API_KEY`
- `models.providers.comfy.apiKey`

Cloud 모드도 동일한 `image`, `video`, `music` 워크플로 섹션을 사용합니다.

## 라이브 테스트

번들 plugin에 대한 opt-in 라이브 커버리지가 있습니다.

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

일치하는 Comfy 워크플로 섹션이 설정되지 않은 경우, 라이브 테스트는 개별 이미지, 비디오 또는 음악 케이스를 건너뜁니다.

## 관련 문서

- [이미지 생성](/ko/tools/image-generation)
- [비디오 생성](/tools/video-generation)
- [음악 생성](/tools/music-generation)
- [Provider 디렉터리](/ko/providers/index)
- [Configuration Reference](/ko/gateway/configuration-reference#agent-defaults)
