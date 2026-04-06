---
read_when:
    - OpenClaw에서 Together AI를 사용하려는 경우
    - API 키 env var 또는 CLI auth 선택이 필요한 경우
summary: Together AI 설정(auth + 모델 선택)
title: Together AI
x-i18n:
    generated_at: "2026-04-06T03:11:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: b68fdc15bfcac8d59e3e0c06a39162abd48d9d41a9a64a0ac622cd8e3f80a595
    source_path: providers/together.md
    workflow: 15
---

# Together AI

[Together AI](https://together.ai)는 통합 API를 통해 Llama, DeepSeek, Kimi 등 주요 오픈 소스 모델에 대한 접근을 제공합니다.

- Provider: `together`
- Auth: `TOGETHER_API_KEY`
- API: OpenAI 호환
- Base URL: `https://api.together.xyz/v1`

## 빠른 시작

1. API 키를 설정합니다(권장: Gateway용으로 저장):

```bash
openclaw onboard --auth-choice together-api-key
```

2. 기본 모델을 설정합니다:

```json5
{
  agents: {
    defaults: {
      model: { primary: "together/moonshotai/Kimi-K2.5" },
    },
  },
}
```

## 비대화형 예시

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

이렇게 하면 `together/moonshotai/Kimi-K2.5`가 기본 모델로 설정됩니다.

## 환경 참고

Gateway가 데몬(launchd/systemd)으로 실행되는 경우, `TOGETHER_API_KEY`가
해당 프로세스에서 사용할 수 있도록 해야 합니다(예: `~/.openclaw/.env` 또는
`env.shellEnv`를 통해).

## 내장 카탈로그

현재 OpenClaw는 다음 번들 Together 카탈로그를 제공합니다:

| Model ref                                                    | 이름                                   | 입력        | 컨텍스트   | 참고                             |
| ------------------------------------------------------------ | -------------------------------------- | ----------- | ---------- | -------------------------------- |
| `together/moonshotai/Kimi-K2.5`                              | Kimi K2.5                              | text, image | 262,144    | 기본 모델; reasoning 활성화됨    |
| `together/zai-org/GLM-4.7`                                   | GLM 4.7 Fp8                            | text        | 202,752    | 범용 텍스트 모델                 |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`           | Llama 3.3 70B Instruct Turbo           | text        | 131,072    | 빠른 instruction 모델            |
| `together/meta-llama/Llama-4-Scout-17B-16E-Instruct`         | Llama 4 Scout 17B 16E Instruct         | text, image | 10,000,000 | 멀티모달                         |
| `together/meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8` | Llama 4 Maverick 17B 128E Instruct FP8 | text, image | 20,000,000 | 멀티모달                         |
| `together/deepseek-ai/DeepSeek-V3.1`                         | DeepSeek V3.1                          | text        | 131,072    | 범용 텍스트 모델                 |
| `together/deepseek-ai/DeepSeek-R1`                           | DeepSeek R1                            | text        | 131,072    | reasoning 모델                   |
| `together/moonshotai/Kimi-K2-Instruct-0905`                  | Kimi K2-Instruct 0905                  | text        | 262,144    | 보조 Kimi 텍스트 모델            |

온보딩 프리셋은 `together/moonshotai/Kimi-K2.5`를 기본 모델로 설정합니다.

## 비디오 생성

번들 `together` plugin은 공통
`video_generate` 도구를 통해 비디오 생성도 등록합니다.

- 기본 비디오 모델: `together/Wan-AI/Wan2.2-T2V-A14B`
- 모드: 텍스트-비디오 및 단일 이미지 참조 흐름
- `aspectRatio`와 `resolution` 지원

Together를 기본 비디오 provider로 사용하려면:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "together/Wan-AI/Wan2.2-T2V-A14B",
      },
    },
  },
}
```

공통 도구
파라미터, provider 선택, 장애 조치 동작은 [Video Generation](/tools/video-generation)을 참조하세요.
