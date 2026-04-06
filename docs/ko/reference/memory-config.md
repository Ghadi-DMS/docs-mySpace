---
read_when:
    - 메모리 검색 provider 또는 임베딩 모델을 구성하고 싶을 때
    - QMD 백엔드를 설정하고 싶을 때
    - 하이브리드 검색, MMR 또는 시간 감쇠를 조정하고 싶을 때
    - 멀티모달 메모리 인덱싱을 활성화하고 싶을 때
summary: 메모리 검색, 임베딩 provider, QMD, 하이브리드 검색, 멀티모달 인덱싱을 위한 모든 구성 항목
title: 메모리 구성 참조
x-i18n:
    generated_at: "2026-04-06T03:12:41Z"
    model: gpt-5.4
    provider: openai
    source_hash: 0de0b85125443584f4e575cf673ca8d9bd12ecd849d73c537f4a17545afa93fd
    source_path: reference/memory-config.md
    workflow: 15
---

# 메모리 구성 참조

이 페이지는 OpenClaw 메모리 검색의 모든 구성 항목을 나열합니다.
개념적 개요는 다음을 참고하세요.

- [Memory Overview](/ko/concepts/memory) -- 메모리가 동작하는 방식
- [Builtin Engine](/ko/concepts/memory-builtin) -- 기본 SQLite 백엔드
- [QMD Engine](/ko/concepts/memory-qmd) -- 로컬 우선 사이드카
- [Memory Search](/ko/concepts/memory-search) -- 검색 파이프라인 및 조정

별도로 언급되지 않는 한 모든 메모리 검색 설정은
`openclaw.json`의 `agents.defaults.memorySearch` 아래에 있습니다.

---

## Provider 선택

| 키         | 타입      | 기본값         | 설명                                                                                   |
| ---------- | --------- | -------------- | -------------------------------------------------------------------------------------- |
| `provider` | `string`  | 자동 감지      | 임베딩 어댑터 ID: `openai`, `gemini`, `voyage`, `mistral`, `bedrock`, `ollama`, `local` |
| `model`    | `string`  | provider 기본값 | 임베딩 모델 이름                                                                       |
| `fallback` | `string`  | `"none"`       | 기본 어댑터가 실패할 때 사용할 대체 어댑터 ID                                          |
| `enabled`  | `boolean` | `true`         | 메모리 검색 활성화 또는 비활성화                                                       |

### 자동 감지 순서

`provider`가 설정되지 않은 경우 OpenClaw는 사용 가능한 첫 항목을 선택합니다.

1. `local` -- `memorySearch.local.modelPath`가 구성되어 있고 파일이 존재하는 경우
2. `openai` -- OpenAI 키를 확인할 수 있는 경우
3. `gemini` -- Gemini 키를 확인할 수 있는 경우
4. `voyage` -- Voyage 키를 확인할 수 있는 경우
5. `mistral` -- Mistral 키를 확인할 수 있는 경우
6. `bedrock` -- AWS SDK 자격 증명 체인이 확인되는 경우(인스턴스 역할, 액세스 키, 프로필, SSO, 웹 아이덴티티 또는 공유 구성)

`ollama`도 지원되지만 자동 감지는 하지 않습니다(명시적으로 설정하세요).

### API 키 확인

원격 임베딩에는 API 키가 필요합니다. 대신 Bedrock은 AWS SDK 기본
자격 증명 체인(인스턴스 역할, SSO, 액세스 키)을 사용합니다.

| Provider | Env var                        | 구성 키                           |
| -------- | ------------------------------ | --------------------------------- |
| OpenAI   | `OPENAI_API_KEY`               | `models.providers.openai.apiKey`  |
| Gemini   | `GEMINI_API_KEY`               | `models.providers.google.apiKey`  |
| Voyage   | `VOYAGE_API_KEY`               | `models.providers.voyage.apiKey`  |
| Mistral  | `MISTRAL_API_KEY`              | `models.providers.mistral.apiKey` |
| Bedrock  | AWS 자격 증명 체인            | API 키 필요 없음                  |
| Ollama   | `OLLAMA_API_KEY` (placeholder) | --                                |

Codex OAuth는 채팅/completions만 지원하며 임베딩
요청에는 사용할 수 없습니다.

---

## 원격 엔드포인트 구성

사용자 지정 OpenAI 호환 엔드포인트를 사용하거나 provider 기본값을 재정의하려면 다음을 사용하세요.

| 키               | 타입     | 설명                                           |
| ---------------- | -------- | ---------------------------------------------- |
| `remote.baseUrl` | `string` | 사용자 지정 API 기본 URL                       |
| `remote.apiKey`  | `string` | API 키 재정의                                  |
| `remote.headers` | `object` | 추가 HTTP 헤더(provider 기본값과 병합됨)       |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small",
        remote: {
          baseUrl: "https://api.example.com/v1/",
          apiKey: "YOUR_KEY",
        },
      },
    },
  },
}
```

---

## Gemini 전용 구성

| 키                     | 타입     | 기본값                 | 설명                                       |
| ---------------------- | -------- | ---------------------- | ------------------------------------------ |
| `model`                | `string` | `gemini-embedding-001` | `gemini-embedding-2-preview`도 지원        |
| `outputDimensionality` | `number` | `3072`                 | Embedding 2의 경우: 768, 1536 또는 3072    |

<Warning>
모델 또는 `outputDimensionality`를 변경하면 자동으로 전체 재인덱싱이 수행됩니다.
</Warning>

---

## Bedrock 임베딩 구성

Bedrock은 AWS SDK 기본 자격 증명 체인을 사용하므로 API 키가 필요하지 않습니다.
OpenClaw가 Bedrock이 활성화된 인스턴스 역할이 있는 EC2에서 실행 중이라면,
provider와 모델만 설정하면 됩니다.

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0",
      },
    },
  },
}
```

| 키                     | 타입     | 기본값                         | 설명                            |
| ---------------------- | -------- | ------------------------------ | ------------------------------- |
| `model`                | `string` | `amazon.titan-embed-text-v2:0` | 모든 Bedrock 임베딩 모델 ID     |
| `outputDimensionality` | `number` | 모델 기본값                    | Titan V2의 경우: 256, 512 또는 1024 |

### 지원 모델

다음 모델이 지원됩니다(패밀리 감지 및 차원 기본값 포함).

| 모델 ID                                    | Provider   | 기본 차원    | 구성 가능한 차원    |
| ------------------------------------------ | ---------- | ------------ | ------------------- |
| `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024      |
| `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                  |
| `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                  |
| `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                  |
| `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072 |
| `cohere.embed-english-v3`                  | Cohere     | 1024         | --                  |
| `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                  |
| `cohere.embed-v4:0`                        | Cohere     | 1536         | 256-1536            |
| `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                  |
| `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                  |

처리량 접미사가 붙은 변형(예: `amazon.titan-embed-text-v1:2:8k`)은
기본 모델의 구성을 그대로 상속합니다.

### 인증

Bedrock 인증은 표준 AWS SDK 자격 증명 확인 순서를 사용합니다.

1. 환경 변수(`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`)
2. SSO 토큰 캐시
3. 웹 아이덴티티 토큰 자격 증명
4. 공유 자격 증명 및 구성 파일
5. ECS 또는 EC2 메타데이터 자격 증명

리전은 `AWS_REGION`, `AWS_DEFAULT_REGION`, `amazon-bedrock`
provider의 `baseUrl`에서 확인되며, 없으면 기본값은 `us-east-1`입니다.

### IAM 권한

IAM 역할 또는 사용자에게 다음 권한이 필요합니다.

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "*"
}
```

최소 권한 원칙을 적용하려면 `InvokeModel`을 특정 모델로 제한하세요.

```
arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
```

---

## 로컬 임베딩 구성

| 키                    | 타입     | 기본값                 | 설명                            |
| --------------------- | -------- | ---------------------- | ------------------------------- |
| `local.modelPath`     | `string` | 자동 다운로드          | GGUF 모델 파일 경로             |
| `local.modelCacheDir` | `string` | node-llama-cpp 기본값  | 다운로드된 모델의 캐시 디렉터리 |

기본 모델: `embeddinggemma-300m-qat-Q8_0.gguf`(~0.6 GB, 자동 다운로드).
네이티브 빌드가 필요합니다: `pnpm approve-builds` 후 `pnpm rebuild node-llama-cpp`.

---

## 하이브리드 검색 구성

모두 `memorySearch.query.hybrid` 아래에 있습니다.

| 키                    | 타입      | 기본값 | 설명                             |
| --------------------- | --------- | ------ | -------------------------------- |
| `enabled`             | `boolean` | `true` | 하이브리드 BM25 + 벡터 검색 활성화 |
| `vectorWeight`        | `number`  | `0.7`  | 벡터 점수 가중치(0-1)            |
| `textWeight`          | `number`  | `0.3`  | BM25 점수 가중치(0-1)            |
| `candidateMultiplier` | `number`  | `4`    | 후보 풀 크기 배수                |

### MMR(다양성)

| 키            | 타입      | 기본값 | 설명                                |
| ------------- | --------- | ------ | ----------------------------------- |
| `mmr.enabled` | `boolean` | `false` | MMR 재순위화 활성화                 |
| `mmr.lambda`  | `number`  | `0.7`  | 0 = 최대 다양성, 1 = 최대 관련성    |

### 시간 감쇠(최신성)

| 키                           | 타입      | 기본값 | 설명                           |
| ---------------------------- | --------- | ------ | ------------------------------ |
| `temporalDecay.enabled`      | `boolean` | `false` | 최신성 부스트 활성화          |
| `temporalDecay.halfLifeDays` | `number`  | `30`   | N일마다 점수가 절반으로 감소   |

상시 유지 파일(`MEMORY.md`, `memory/` 내 날짜가 없는 파일)은 감쇠되지 않습니다.

### 전체 예시

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            vectorWeight: 0.7,
            textWeight: 0.3,
            mmr: { enabled: true, lambda: 0.7 },
            temporalDecay: { enabled: true, halfLifeDays: 30 },
          },
        },
      },
    },
  },
}
```

---

## 추가 메모리 경로

| 키           | 타입       | 설명                              |
| ------------ | ---------- | --------------------------------- |
| `extraPaths` | `string[]` | 인덱싱할 추가 디렉터리 또는 파일  |

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        extraPaths: ["../team-docs", "/srv/shared-notes"],
      },
    },
  },
}
```

경로는 절대 경로 또는 workspace 기준 상대 경로일 수 있습니다. 디렉터리는
`.md` 파일을 재귀적으로 스캔합니다. 심볼릭 링크 처리 방식은 활성 백엔드에 따라 다릅니다.
builtin 엔진은 심볼릭 링크를 무시하고, QMD는 기반 QMD
스캐너 동작을 따릅니다.

에이전트 범위의 교차 에이전트 transcript 검색에는
`memory.qmd.paths` 대신 `agents.list[].memorySearch.qmd.extraCollections`를
사용하세요. 이 추가 컬렉션은 동일한 `{ path, name, pattern? }` 형태를 따르지만,
에이전트별로 병합되며 경로가 현재 workspace 밖을 가리킬 때도
명시적인 공유 이름을 유지할 수 있습니다.
같은 확인된 경로가 `memory.qmd.paths`와
`memorySearch.qmd.extraCollections`에 모두 나타나면 QMD는 첫 번째 항목을 유지하고
중복은 건너뜁니다.

---

## 멀티모달 메모리(Gemini)

Gemini Embedding 2를 사용해 Markdown과 함께 이미지와 오디오도 인덱싱합니다.

| 키                        | 타입       | 기본값     | 설명                                     |
| ------------------------- | ---------- | ---------- | ---------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | 멀티모달 인덱싱 활성화                   |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`, `["audio"]` 또는 `["all"]`  |
| `multimodal.maxFileBytes` | `number`   | `10000000` | 인덱싱할 최대 파일 크기                  |

`extraPaths`의 파일에만 적용됩니다. 기본 메모리 루트는 여전히 Markdown 전용입니다.
`gemini-embedding-2-preview`가 필요합니다. `fallback`은 반드시 `"none"`이어야 합니다.

지원 형식: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`
(이미지); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac`(오디오).

---

## 임베딩 캐시

| 키                 | 타입      | 기본값 | 설명                                |
| ------------------ | --------- | ------ | ----------------------------------- |
| `cache.enabled`    | `boolean` | `false` | SQLite에 청크 임베딩 캐시           |
| `cache.maxEntries` | `number`  | `50000` | 최대 캐시 임베딩 수                 |

재인덱싱 또는 transcript 업데이트 중 변경되지 않은 텍스트를 다시 임베딩하는 일을 방지합니다.

---

## 배치 인덱싱

| 키                            | 타입      | 기본값 | 설명                      |
| ----------------------------- | --------- | ------ | ------------------------- |
| `remote.batch.enabled`        | `boolean` | `false` | 배치 임베딩 API 활성화    |
| `remote.batch.concurrency`    | `number`  | `2`    | 병렬 배치 작업 수         |
| `remote.batch.wait`           | `boolean` | `true` | 배치 완료까지 대기        |
| `remote.batch.pollIntervalMs` | `number`  | --     | 폴링 간격                 |
| `remote.batch.timeoutMinutes` | `number`  | --     | 배치 타임아웃             |

`openai`, `gemini`, `voyage`에서 사용할 수 있습니다. OpenAI 배치는 일반적으로
대규모 백필에서 가장 빠르고 저렴합니다.

---

## 세션 메모리 검색(실험적)

세션 transcript를 인덱싱하고 `memory_search`를 통해 노출합니다.

| 키                            | 타입       | 기본값       | 설명                                      |
| ----------------------------- | ---------- | ------------ | ----------------------------------------- |
| `experimental.sessionMemory`  | `boolean`  | `false`      | 세션 인덱싱 활성화                        |
| `sources`                     | `string[]` | `["memory"]` | transcript를 포함하려면 `"sessions"` 추가 |
| `sync.sessions.deltaBytes`    | `number`   | `100000`     | 재인덱싱 바이트 임계값                    |
| `sync.sessions.deltaMessages` | `number`   | `50`         | 재인덱싱 메시지 임계값                    |

세션 인덱싱은 opt-in이며 비동기적으로 실행됩니다. 결과는 약간
오래되었을 수 있습니다. 세션 로그는 디스크에 있으므로 파일 시스템 접근을
신뢰 경계로 취급하세요.

---

## SQLite 벡터 가속(sqlite-vec)

| 키                           | 타입      | 기본값  | 설명                               |
| ---------------------------- | --------- | ------- | ---------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | 벡터 쿼리에 sqlite-vec 사용        |
| `store.vector.extensionPath` | `string`  | bundled | sqlite-vec 경로 재정의             |

sqlite-vec을 사용할 수 없으면 OpenClaw는 자동으로 프로세스 내 cosine
similarity로 대체합니다.

---

## 인덱스 저장소

| 키                    | 타입     | 기본값                               | 설명                                      |
| --------------------- | -------- | ------------------------------------ | ----------------------------------------- |
| `store.path`          | `string` | `~/.openclaw/memory/{agentId}.sqlite` | 인덱스 위치(`{agentId}` 토큰 지원)        |
| `store.fts.tokenizer` | `string` | `unicode61`                          | FTS5 tokenizer(`unicode61` 또는 `trigram`) |

---

## QMD 백엔드 구성

활성화하려면 `memory.backend = "qmd"`를 설정하세요. 모든 QMD 설정은
`memory.qmd` 아래에 있습니다.

| 키                       | 타입      | 기본값   | 설명                                             |
| ------------------------ | --------- | -------- | ------------------------------------------------ |
| `command`                | `string`  | `qmd`    | QMD 실행 파일 경로                               |
| `searchMode`             | `string`  | `search` | 검색 명령: `search`, `vsearch`, `query`          |
| `includeDefaultMemory`   | `boolean` | `true`   | `MEMORY.md` + `memory/**/*.md` 자동 인덱싱       |
| `paths[]`                | `array`   | --       | 추가 경로: `{ name, path, pattern? }`            |
| `sessions.enabled`       | `boolean` | `false`  | 세션 transcript 인덱싱                           |
| `sessions.retentionDays` | `number`  | --       | transcript 보관 기간                             |
| `sessions.exportDir`     | `string`  | --       | 내보내기 디렉터리                                |

OpenClaw는 현재 QMD 컬렉션 및 MCP 쿼리 형태를 우선 사용하지만,
필요할 경우 레거시 `--mask` 컬렉션 플래그와
이전 MCP 도구 이름으로 대체해 구버전 QMD도 계속 작동하도록 유지합니다.

QMD 모델 재정의는 OpenClaw 구성 쪽이 아니라 QMD 쪽에 유지됩니다.
QMD 모델을 전역적으로 재정의해야 한다면 gateway
런타임 환경에서 `QMD_EMBED_MODEL`, `QMD_RERANK_MODEL`,
`QMD_GENERATE_MODEL` 같은 환경 변수를 설정하세요.

### 업데이트 일정

| 키                        | 타입      | 기본값  | 설명                                  |
| ------------------------- | --------- | ------- | ------------------------------------- |
| `update.interval`         | `string`  | `5m`    | 새로고침 간격                         |
| `update.debounceMs`       | `number`  | `15000` | 파일 변경 디바운스                    |
| `update.onBoot`           | `boolean` | `true`  | 시작 시 새로고침                      |
| `update.waitForBootSync`  | `boolean` | `false` | 새로고침 완료까지 시작 차단           |
| `update.embedInterval`    | `string`  | --      | 별도 임베딩 주기                      |
| `update.commandTimeoutMs` | `number`  | --      | QMD 명령 타임아웃                     |
| `update.updateTimeoutMs`  | `number`  | --      | QMD 업데이트 작업 타임아웃            |
| `update.embedTimeoutMs`   | `number`  | --      | QMD 임베딩 작업 타임아웃              |

### 제한

| 키                        | 타입     | 기본값 | 설명                         |
| ------------------------- | -------- | ------ | ---------------------------- |
| `limits.maxResults`       | `number` | `6`    | 최대 검색 결과 수            |
| `limits.maxSnippetChars`  | `number` | --     | snippet 길이 제한            |
| `limits.maxInjectedChars` | `number` | --     | 총 주입 문자 수 제한         |
| `limits.timeoutMs`        | `number` | `4000` | 검색 타임아웃                |

### 범위

어떤 세션이 QMD 검색 결과를 받을 수 있는지 제어합니다. 스키마는
[`session.sendPolicy`](/ko/gateway/configuration-reference#session)와 같습니다.

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

기본값은 DM 전용입니다. `match.keyPrefix`는 정규화된 세션 키와 일치하고,
`match.rawKeyPrefix`는 `agent:<id>:`를 포함한 원시 키와 일치합니다.

### 인용

`memory.citations`는 모든 백엔드에 적용됩니다.

| 값               | 동작                                               |
| ---------------- | -------------------------------------------------- |
| `auto` (기본값)  | snippet에 `Source: <path#line>` 바닥글 포함        |
| `on`             | 항상 바닥글 포함                                   |
| `off`            | 바닥글 생략(경로는 여전히 내부적으로 에이전트에 전달됨) |

### 전체 QMD 예시

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 6, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming(실험적)

Dreaming은 `agents.defaults.memorySearch` 아래가 아니라
`plugins.entries.memory-core.config.dreaming` 아래에서 구성합니다.

Dreaming은 하나의 예약된 스윕으로 실행되며, 내부 light/deep/REM 단계는
구현 세부 사항으로 사용됩니다.

개념적 동작과 슬래시 명령은 [Dreaming](/concepts/dreaming)을 참고하세요.

### 사용자 설정

| 키          | 타입      | 기본값      | 설명                                        |
| ----------- | --------- | ----------- | ------------------------------------------- |
| `enabled`   | `boolean` | `false`     | Dreaming 전체 활성화 또는 비활성화          |
| `frequency` | `string`  | `0 3 * * *` | 전체 dreaming 스윕에 대한 선택적 cron 주기  |

### 예시

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
          },
        },
      },
    },
  },
}
```

참고:

- Dreaming은 기계 상태를 `memory/.dreams/`에 기록합니다.
- Dreaming은 사람이 읽을 수 있는 서사 출력을 `DREAMS.md`(또는 기존 `dreams.md`)에 기록합니다.
- light/deep/REM 단계 정책과 임계값은 내부 동작이며 사용자 대상 config가 아닙니다.
