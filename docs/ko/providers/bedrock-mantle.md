---
read_when:
    - OpenClaw에서 Bedrock Mantle이 호스팅하는 OSS 모델을 사용하려는 경우
    - GPT-OSS, Qwen, Kimi 또는 GLM용 Mantle OpenAI 호환 엔드포인트가 필요한 경우
summary: OpenClaw에서 Amazon Bedrock Mantle(OpenAI 호환) 모델 사용하기
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-04-06T03:10:36Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4e5b33ede4067fb7de02a046f3e375cbd2af4bf68e7751c8dd687447f1a78c86
    source_path: providers/bedrock-mantle.md
    workflow: 15
---

# Amazon Bedrock Mantle

OpenClaw에는 Mantle OpenAI 호환 엔드포인트에 연결되는 번들 **Amazon Bedrock Mantle** provider가 포함되어 있습니다. Mantle은 Bedrock 인프라를 기반으로 하는 표준 `/v1/chat/completions` 표면을 통해 오픈 소스 및 서드파티 모델(GPT-OSS, Qwen, Kimi, GLM 등)을 호스팅합니다.

## OpenClaw 지원 사항

- Provider: `amazon-bedrock-mantle`
- API: `openai-completions` (OpenAI 호환)
- Auth: 명시적 `AWS_BEARER_TOKEN_BEDROCK` 또는 IAM 자격 증명 체인 기반 bearer token 생성
- 리전: `AWS_REGION` 또는 `AWS_DEFAULT_REGION` (기본값: `us-east-1`)

## 자동 모델 탐지

`AWS_BEARER_TOKEN_BEDROCK`가 설정되어 있으면 OpenClaw는 이를 직접 사용합니다. 그렇지 않으면 OpenClaw는 공유 자격 증명/config 프로필, SSO, 웹 ID, 인스턴스 또는 태스크 역할을 포함한 AWS 기본 자격 증명 체인에서 Mantle bearer token 생성을 시도합니다. 그런 다음 해당 리전의 `/v1/models` 엔드포인트를 질의하여 사용 가능한 Mantle 모델을 탐지합니다. 탐지 결과는 1시간 동안 캐시되며, IAM에서 파생된 bearer token은 매시간 갱신됩니다.

지원 리전: `us-east-1`, `us-east-2`, `us-west-2`, `ap-northeast-1`,
`ap-south-1`, `ap-southeast-3`, `eu-central-1`, `eu-west-1`, `eu-west-2`,
`eu-south-1`, `eu-north-1`, `sa-east-1`.

## 온보딩

1. **게이트웨이 호스트**에서 auth 경로 하나를 선택합니다:

명시적 bearer token:

```bash
export AWS_BEARER_TOKEN_BEDROCK="..."
# 선택 사항(기본값은 us-east-1):
export AWS_REGION="us-west-2"
```

IAM 자격 증명:

```bash
# 예를 들어 어떤 AWS SDK 호환 auth 소스도 여기서 동작합니다:
export AWS_PROFILE="default"
export AWS_REGION="us-west-2"
```

2. 모델이 탐지되는지 확인합니다:

```bash
openclaw models list
```

탐지된 모델은 `amazon-bedrock-mantle` provider 아래에 표시됩니다. 기본값을 재정의하려는 경우가 아니라면 추가 config는 필요하지 않습니다.

## 수동 구성

자동 탐지 대신 명시적 config를 선호한다면:

```json5
{
  models: {
    providers: {
      "amazon-bedrock-mantle": {
        baseUrl: "https://bedrock-mantle.us-east-1.api.aws/v1",
        api: "openai-completions",
        auth: "api-key",
        apiKey: "env:AWS_BEARER_TOKEN_BEDROCK",
        models: [
          {
            id: "gpt-oss-120b",
            name: "GPT-OSS 120B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

## 참고

- `AWS_BEARER_TOKEN_BEDROCK`가 설정되지 않은 경우, OpenClaw는 AWS SDK 호환 IAM 자격 증명에서 Mantle bearer token을 대신 발급할 수 있습니다.
- bearer token은 표준 [Amazon Bedrock](/ko/providers/bedrock) provider에서 사용하는 동일한 `AWS_BEARER_TOKEN_BEDROCK`입니다.
- reasoning 지원은 `thinking`, `reasoner`, `gpt-oss-120b` 같은 패턴을 포함하는 모델 ID에서 추론됩니다.
- Mantle 엔드포인트를 사용할 수 없거나 모델을 반환하지 않으면, 해당 provider는 조용히 건너뜁니다.
