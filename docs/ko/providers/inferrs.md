---
read_when:
    - 로컬 inferrs 서버에 대해 OpenClaw를 실행하려는 경우
    - inferrs를 통해 Gemma 또는 다른 모델을 제공하는 경우
    - inferrs용 정확한 OpenClaw 호환 플래그가 필요한 경우
summary: inferrs(OpenAI 호환 로컬 서버)를 통해 OpenClaw 실행하기
title: inferrs
x-i18n:
    generated_at: "2026-04-09T01:30:27Z"
    model: gpt-5.4
    provider: openai
    source_hash: 03b9d5a9935c75fd369068bacb7807a5308cd0bd74303b664227fb664c3a2098
    source_path: providers/inferrs.md
    workflow: 15
---

# inferrs

[inferrs](https://github.com/ericcurtin/inferrs)는 OpenAI 호환 `/v1` API 뒤에서 로컬 모델을 제공할 수 있습니다. OpenClaw는 일반 `openai-completions` 경로를 통해 `inferrs`와 함께 동작합니다.

현재 `inferrs`는 전용 OpenClaw provider plugin이 아니라 사용자 지정 자체 호스팅 OpenAI 호환 백엔드로 취급하는 것이 가장 적절합니다.

## 빠른 시작

1. 모델로 `inferrs`를 시작합니다.

예:

```bash
inferrs serve google/gemma-4-E2B-it \
  --host 127.0.0.1 \
  --port 8080 \
  --device metal
```

2. 서버에 연결할 수 있는지 확인합니다.

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/v1/models
```

3. 명시적인 OpenClaw provider 항목을 추가하고 기본 모델이 이를 가리키도록 설정합니다.

## 전체 구성 예제

이 예제는 로컬 `inferrs` 서버에서 Gemma 4를 사용합니다.

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
      models: {
        "inferrs/google/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## `requiresStringContent`가 중요한 이유

일부 `inferrs` Chat Completions 경로는 구조화된 content-part 배열이 아니라 문자열 `messages[].content`만 허용합니다.

OpenClaw 실행 시 다음과 같은 오류가 발생하면:

```text
messages[1].content: invalid type: sequence, expected a string
```

다음을 설정하세요:

```json5
compat: {
  requiresStringContent: true
}
```

OpenClaw는 요청을 보내기 전에 순수 텍스트 content part를 일반 문자열로 평탄화합니다.

## Gemma 및 tool-schema 주의 사항

현재 일부 `inferrs` + Gemma 조합은 작은 직접 `/v1/chat/completions` 요청은 수락하지만 전체 OpenClaw agent-runtime 턴에서는 여전히 실패합니다.

이 경우 먼저 다음을 시도하세요:

```json5
compat: {
  requiresStringContent: true,
  supportsTools: false
}
```

이 설정은 해당 모델에 대한 OpenClaw의 tool schema 표면을 비활성화하며, 더 엄격한 로컬 백엔드에서 prompt 부담을 줄일 수 있습니다.

작은 직접 요청은 여전히 동작하지만 일반 OpenClaw agent 턴이 `inferrs` 내부에서 계속 충돌한다면, 남아 있는 문제는 보통 OpenClaw의 전송 계층보다는 업스트림 모델/서버 동작에 있습니다.

## 수동 스모크 테스트

구성한 후에는 두 계층을 모두 테스트하세요:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"google/gemma-4-E2B-it","messages":[{"role":"user","content":"What is 2 + 2?"}],"stream":false}'

openclaw infer model run \
  --model inferrs/google/gemma-4-E2B-it \
  --prompt "What is 2 + 2? Reply with one short sentence." \
  --json
```

첫 번째 명령은 동작하지만 두 번째가 실패하면 아래 문제 해결 참고 사항을 사용하세요.

## 문제 해결

- `curl /v1/models` 실패: `inferrs`가 실행 중이 아니거나, 연결할 수 없거나, 예상한 host/port에 바인딩되지 않았습니다.
- `messages[].content ... expected a string`: `compat.requiresStringContent: true`를 설정하세요.
- 직접적인 작은 `/v1/chat/completions` 호출은 통과하지만 `openclaw infer model run`은 실패: `compat.supportsTools: false`를 시도하세요.
- OpenClaw에서 더 이상 스키마 오류는 발생하지 않지만 `inferrs`가 더 큰 agent 턴에서 여전히 충돌: 이를 업스트림 `inferrs` 또는 모델의 제한으로 보고 prompt 부담을 줄이거나 로컬 백엔드/모델을 전환하세요.

## 프록시 스타일 동작

`inferrs`는 네이티브 OpenAI 엔드포인트가 아니라 프록시 스타일 OpenAI 호환 `/v1` 백엔드로 취급됩니다.

- 네이티브 OpenAI 전용 요청 형태 조정은 여기에는 적용되지 않음
- `service_tier`, Responses `store`, prompt-cache 힌트, OpenAI reasoning-compat 페이로드 형태 조정이 없음
- 숨겨진 OpenClaw attribution 헤더(`originator`, `version`, `User-Agent`)는 사용자 지정 `inferrs` base URL에 주입되지 않음

## 함께 보기

- [Local models](/ko/gateway/local-models)
- [Gateway troubleshooting](/ko/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail)
- [Model providers](/ko/concepts/model-providers)
