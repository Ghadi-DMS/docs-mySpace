---
read_when:
    - Ollama를 통해 클라우드 또는 로컬 모델로 OpenClaw를 실행하려는 경우
    - Ollama 설정 및 구성 가이드가 필요한 경우
summary: Ollama로 OpenClaw 실행하기(클라우드 및 로컬 모델)
title: Ollama
x-i18n:
    generated_at: "2026-04-09T01:31:04Z"
    model: gpt-5.4
    provider: openai
    source_hash: d3295a7c879d3636a2ffdec05aea6e670e54a990ef52bd9b0cae253bc24aa3f7
    source_path: providers/ollama.md
    workflow: 15
---

# Ollama

Ollama는 머신에서 오픈 소스 모델을 쉽게 실행할 수 있게 해 주는 로컬 LLM 런타임입니다. OpenClaw는 Ollama의 네이티브 API(`/api/chat`)와 통합되며, 스트리밍과 tool calling을 지원하고, `OLLAMA_API_KEY`(또는 auth profile)로 옵트인하고 명시적인 `models.providers.ollama` 항목을 정의하지 않으면 로컬 Ollama 모델을 자동으로 검색할 수 있습니다.

<Warning>
**원격 Ollama 사용자**: OpenClaw와 함께 `/v1` OpenAI 호환 URL(`http://host:11434/v1`)을 사용하지 마세요. 이렇게 하면 tool calling이 깨지고 모델이 원시 tool JSON을 일반 텍스트로 출력할 수 있습니다. 대신 네이티브 Ollama API URL을 사용하세요: `baseUrl: "http://host:11434"` (`/v1` 없음).
</Warning>

## 빠른 시작

### 온보딩(권장)

Ollama를 설정하는 가장 빠른 방법은 온보딩을 사용하는 것입니다:

```bash
openclaw onboard
```

provider 목록에서 **Ollama**를 선택하세요. 온보딩은 다음을 수행합니다:

1. 인스턴스에 접근할 수 있는 Ollama 기본 URL을 묻습니다(기본값 `http://127.0.0.1:11434`).
2. **Cloud + Local**(클라우드 모델 및 로컬 모델) 또는 **Local**(로컬 모델만)을 선택하게 합니다.
3. **Cloud + Local**을 선택했고 `ollama.com`에 로그인되어 있지 않으면 브라우저 로그인 흐름을 엽니다.
4. 사용 가능한 모델을 검색하고 기본값을 제안합니다.
5. 선택한 모델이 로컬에 없으면 자동으로 pull합니다.

비대화형 모드도 지원됩니다:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

선택적으로 사용자 지정 base URL 또는 모델을 지정할 수 있습니다:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

### 수동 설정

1. Ollama 설치: [https://ollama.com/download](https://ollama.com/download)

2. 로컬 추론을 원하면 로컬 모델을 pull합니다:

```bash
ollama pull gemma4
# 또는
ollama pull gpt-oss:20b
# 또는
ollama pull llama3.3
```

3. 클라우드 모델도 사용하려면 로그인합니다:

```bash
ollama signin
```

4. 온보딩을 실행하고 `Ollama`를 선택합니다:

```bash
openclaw onboard
```

- `Local`: 로컬 모델만
- `Cloud + Local`: 로컬 모델과 클라우드 모델
- `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud` 같은 클라우드 모델은 로컬 `ollama pull`이 **필요하지 않습니다**

OpenClaw는 현재 다음을 제안합니다:

- 로컬 기본값: `gemma4`
- 클라우드 기본값: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`

5. 수동 설정을 선호한다면 OpenClaw에서 Ollama를 직접 활성화하세요(아무 값이나 동작하며, Ollama는 실제 키가 필요하지 않습니다):

```bash
# 환경 변수 설정
export OLLAMA_API_KEY="ollama-local"

# 또는 config 파일에 설정
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. 모델을 확인하거나 전환합니다:

```bash
openclaw models list
openclaw models set ollama/gemma4
```

7. 또는 config에서 기본값을 설정합니다:

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/gemma4" },
    },
  },
}
```

## 모델 검색(암시적 provider)

`OLLAMA_API_KEY`(또는 auth profile)를 설정하고 `models.providers.ollama`를 **정의하지 않으면**, OpenClaw는 `http://127.0.0.1:11434`의 로컬 Ollama 인스턴스에서 모델을 검색합니다:

- `/api/tags` 조회
- 가능할 때 `contextWindow`를 읽고 기능(비전 포함)을 감지하기 위해 최선의 노력 방식으로 `/api/show` 조회 사용
- `/api/show`가 보고한 `vision` 기능이 있는 모델은 이미지 입력 가능 모델(`input: ["text", "image"]`)로 표시되므로 OpenClaw가 해당 모델의 프롬프트에 이미지를 자동 주입함
- 모델 이름 휴리스틱(`r1`, `reasoning`, `think`)으로 `reasoning` 표시
- OpenClaw가 사용하는 기본 Ollama max-token 제한으로 `maxTokens` 설정
- 모든 비용을 `0`으로 설정

이렇게 하면 수동 모델 항목 없이도 카탈로그를 로컬 Ollama 인스턴스와 맞춘 상태로 유지할 수 있습니다.

사용 가능한 모델을 확인하려면:

```bash
ollama list
openclaw models list
```

새 모델을 추가하려면 Ollama로 pull하기만 하면 됩니다:

```bash
ollama pull mistral
```

새 모델은 자동으로 검색되어 사용할 수 있게 됩니다.

`models.providers.ollama`를 명시적으로 설정하면 자동 검색은 건너뛰며 모델을 수동으로 정의해야 합니다(아래 참조).

## 구성

### 기본 설정(암시적 검색)

Ollama를 활성화하는 가장 간단한 방법은 환경 변수를 사용하는 것입니다:

```bash
export OLLAMA_API_KEY="ollama-local"
```

### 명시적 설정(수동 모델)

다음 경우에는 명시적 config를 사용하세요:

- Ollama가 다른 host/port에서 실행되는 경우
- 특정 컨텍스트 윈도우나 모델 목록을 강제하고 싶은 경우
- 완전히 수동으로 모델을 정의하고 싶은 경우

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "gpt-oss:20b",
            name: "GPT-OSS 20B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192 * 10
          }
        ]
      }
    }
  }
}
```

`OLLAMA_API_KEY`가 설정되어 있으면 provider 항목에서 `apiKey`를 생략해도 되며 OpenClaw가 가용성 확인을 위해 이를 채웁니다.

### 사용자 지정 base URL(명시적 config)

Ollama가 다른 host 또는 port에서 실행 중인 경우(명시적 config는 자동 검색을 비활성화하므로 모델을 수동 정의해야 함):

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // /v1 없음 - 네이티브 Ollama API URL 사용
        api: "ollama", // 네이티브 tool-calling 동작을 보장하려면 명시적으로 설정
      },
    },
  },
}
```

<Warning>
URL에 `/v1`을 추가하지 마세요. `/v1` 경로는 OpenAI 호환 모드를 사용하며 tool calling이 신뢰할 수 없습니다. 경로 접미사가 없는 기본 Ollama URL을 사용하세요.
</Warning>

### 모델 선택

구성되면 모든 Ollama 모델을 사용할 수 있습니다:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## 클라우드 모델

클라우드 모델을 사용하면 로컬 모델과 함께 클라우드 호스팅 모델(예: `kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`)을 실행할 수 있습니다.

클라우드 모델을 사용하려면 설정 중에 **Cloud + Local** 모드를 선택하세요. 마법사는 로그인 상태를 확인하고 필요하면 브라우저 로그인 흐름을 엽니다. 인증을 확인할 수 없으면 마법사는 로컬 모델 기본값으로 대체합니다.

[ollama.com/signin](https://ollama.com/signin)에서 직접 로그인할 수도 있습니다.

## Ollama Web Search

OpenClaw는 번들된 `web_search` provider로 **Ollama Web Search**도 지원합니다.

- 구성된 Ollama host를 사용합니다(`models.providers.ollama.baseUrl`가 설정된 경우 그 값, 아니면 `http://127.0.0.1:11434`).
- 키가 필요 없습니다.
- Ollama가 실행 중이어야 하고 `ollama signin`으로 로그인되어 있어야 합니다.

`openclaw onboard` 또는 `openclaw configure --section web` 중에 **Ollama Web Search**를 선택하거나 다음을 설정하세요:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

전체 설정 및 동작 세부 정보는 [Ollama Web Search](/ko/tools/ollama-search)를 참조하세요.

## 고급

### reasoning 모델

OpenClaw는 기본적으로 `deepseek-r1`, `reasoning`, `think` 같은 이름의 모델을 reasoning 가능 모델로 취급합니다:

```bash
ollama pull deepseek-r1:32b
```

### 모델 비용

Ollama는 무료이며 로컬에서 실행되므로 모든 모델 비용은 $0으로 설정됩니다.

### 스트리밍 구성

OpenClaw의 Ollama 통합은 기본적으로 **네이티브 Ollama API**(`/api/chat`)를 사용하며, 스트리밍과 tool calling을 동시에 완전히 지원합니다. 별도의 특별한 설정은 필요하지 않습니다.

#### 레거시 OpenAI 호환 모드

<Warning>
**OpenAI 호환 모드에서는 tool calling이 신뢰할 수 없습니다.** 프록시 때문에 OpenAI 형식이 필요하고 네이티브 tool calling 동작에 의존하지 않는 경우에만 이 모드를 사용하세요.
</Warning>

대신 OpenAI 호환 엔드포인트를 사용해야 하는 경우(예: OpenAI 형식만 지원하는 프록시 뒤에 있는 경우) `api: "openai-completions"`를 명시적으로 설정하세요:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: true, // 기본값: true
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

이 모드는 스트리밍과 tool calling을 동시에 지원하지 않을 수 있습니다. 모델 config에서 `params: { streaming: false }`로 스트리밍을 비활성화해야 할 수 있습니다.

Ollama와 함께 `api: "openai-completions"`를 사용하면 OpenClaw는 Ollama가 조용히 4096 컨텍스트 윈도우로 되돌아가지 않도록 기본적으로 `options.num_ctx`를 주입합니다. 프록시/업스트림이 알 수 없는 `options` 필드를 거부하면 이 동작을 비활성화하세요:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: false,
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

### 컨텍스트 윈도우

자동 검색된 모델의 경우 OpenClaw는 가능한 경우 Ollama가 보고한 컨텍스트 윈도우를 사용하고, 그렇지 않으면 OpenClaw가 사용하는 기본 Ollama 컨텍스트 윈도우로 대체합니다. 명시적 provider config에서 `contextWindow`와 `maxTokens`를 재정의할 수 있습니다.

## 문제 해결

### Ollama가 감지되지 않음

Ollama가 실행 중인지, `OLLAMA_API_KEY`(또는 auth profile)를 설정했는지, 그리고 명시적인 `models.providers.ollama` 항목을 **정의하지 않았는지** 확인하세요:

```bash
ollama serve
```

또한 API에 접근 가능한지 확인하세요:

```bash
curl http://localhost:11434/api/tags
```

### 사용 가능한 모델이 없음

모델이 목록에 없으면 다음 중 하나입니다:

- 모델을 로컬에 pull하거나
- `models.providers.ollama`에 모델을 명시적으로 정의하세요.

모델 추가 방법:

```bash
ollama list  # 설치된 항목 확인
ollama pull gemma4
ollama pull gpt-oss:20b
ollama pull llama3.3     # 또는 다른 모델
```

### 연결 거부됨

Ollama가 올바른 port에서 실행 중인지 확인하세요:

```bash
# Ollama 실행 여부 확인
ps aux | grep ollama

# 또는 Ollama 재시작
ollama serve
```

## 함께 보기

- [Model Providers](/ko/concepts/model-providers) - 모든 provider 개요
- [Model Selection](/ko/concepts/models) - 모델 선택 방법
- [Configuration](/ko/gateway/configuration) - 전체 config 참조
