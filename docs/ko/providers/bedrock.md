---
read_when:
    - OpenClaw에서 Amazon Bedrock 모델을 사용하려는 경우
    - 모델 호출을 위한 AWS 자격 증명/리전 설정이 필요한 경우
summary: OpenClaw에서 Amazon Bedrock(Converse API) 모델 사용하기
title: Amazon Bedrock
x-i18n:
    generated_at: "2026-04-06T03:11:10Z"
    model: gpt-5.4
    provider: openai
    source_hash: 70bb29fe9199084b1179ced60935b5908318f5b80ced490bf44a45e0467c4929
    source_path: providers/bedrock.md
    workflow: 15
---

# Amazon Bedrock

OpenClaw는 pi‑ai의 **Bedrock Converse**
스트리밍 provider를 통해 **Amazon Bedrock** 모델을 사용할 수 있습니다. Bedrock auth는 API 키가 아니라
**AWS SDK 기본 자격 증명 체인**을 사용합니다.

## pi-ai 지원 사항

- Provider: `amazon-bedrock`
- API: `bedrock-converse-stream`
- Auth: AWS 자격 증명(env vars, 공유 config, 또는 인스턴스 역할)
- 리전: `AWS_REGION` 또는 `AWS_DEFAULT_REGION` (기본값: `us-east-1`)

## 자동 모델 탐지

OpenClaw는 **스트리밍**과 **텍스트 출력**을 지원하는 Bedrock 모델을 자동으로 탐지할 수 있습니다. 탐지는 `bedrock:ListFoundationModels`와
`bedrock:ListInferenceProfiles`를 사용하며, 결과는 캐시됩니다(기본값: 1시간).

암묵적 provider가 활성화되는 방식:

- `plugins.entries.amazon-bedrock.config.discovery.enabled`가 `true`이면,
  AWS env 마커가 없어도 OpenClaw는 탐지를 시도합니다.
- `plugins.entries.amazon-bedrock.config.discovery.enabled`가 설정되지 않은 경우,
  OpenClaw는 다음 AWS auth 마커 중 하나를 볼 때만
  암묵적 Bedrock provider를 자동 추가합니다:
  `AWS_BEARER_TOKEN_BEDROCK`, `AWS_ACCESS_KEY_ID` +
  `AWS_SECRET_ACCESS_KEY`, 또는 `AWS_PROFILE`.
- 실제 Bedrock 런타임 auth 경로는 여전히 AWS SDK 기본 체인을 사용하므로,
  탐지에 옵트인하려고 `enabled: true`가 필요했던 경우에도
  공유 config, SSO, IMDS 인스턴스 역할 auth는 동작할 수 있습니다.

Config 옵션은 `plugins.entries.amazon-bedrock.config.discovery` 아래에 있습니다:

```json5
{
  plugins: {
    entries: {
      "amazon-bedrock": {
        config: {
          discovery: {
            enabled: true,
            region: "us-east-1",
            providerFilter: ["anthropic", "amazon"],
            refreshInterval: 3600,
            defaultContextWindow: 32000,
            defaultMaxTokens: 4096,
          },
        },
      },
    },
  },
}
```

참고:

- `enabled`의 기본값은 자동 모드입니다. 자동 모드에서 OpenClaw는
  지원되는 AWS env 마커를 볼 때만 암묵적 Bedrock provider를 활성화합니다.
- `region`의 기본값은 `AWS_REGION` 또는 `AWS_DEFAULT_REGION`, 그다음 `us-east-1`입니다.
- `providerFilter`는 Bedrock provider 이름(예: `anthropic`)과 일치합니다.
- `refreshInterval`의 단위는 초이며, 캐시를 비활성화하려면 `0`으로 설정하세요.
- `defaultContextWindow`(기본값: `32000`)와 `defaultMaxTokens`(기본값: `4096`)는
  탐지된 모델에 사용됩니다(모델 한도를 알고 있다면 override하세요).
- 명시적인 `models.providers["amazon-bedrock"]` 항목의 경우에도 OpenClaw는
  전체 런타임 auth 로드를 강제하지 않고
  `AWS_BEARER_TOKEN_BEDROCK` 같은 AWS env 마커에서 Bedrock env-marker auth를 조기에 해석할 수 있습니다. 실제
  모델 호출 auth 경로는 여전히 AWS SDK 기본 체인을 사용합니다.

## 온보딩

1. **게이트웨이 호스트**에서 AWS 자격 증명을 사용할 수 있는지 확인하세요:

```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"
# 선택 사항:
export AWS_SESSION_TOKEN="..."
export AWS_PROFILE="your-profile"
# 선택 사항(Bedrock API key/bearer token):
export AWS_BEARER_TOKEN_BEDROCK="..."
```

2. config에 Bedrock provider와 모델을 추가하세요(`apiKey`는 필요 없음):

```json5
{
  models: {
    providers: {
      "amazon-bedrock": {
        baseUrl: "https://bedrock-runtime.us-east-1.amazonaws.com",
        api: "bedrock-converse-stream",
        auth: "aws-sdk",
        models: [
          {
            id: "us.anthropic.claude-opus-4-6-v1:0",
            name: "Claude Opus 4.6 (Bedrock)",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1:0" },
    },
  },
}
```

## EC2 인스턴스 역할

IAM 역할이 연결된 EC2 인스턴스에서 OpenClaw를 실행하면 AWS SDK는
인스턴스 메타데이터 서비스(IMDS)를 인증에 사용할 수 있습니다. Bedrock
모델 탐지의 경우 OpenClaw는 명시적으로
`plugins.entries.amazon-bedrock.config.discovery.enabled: true`를 설정하지 않는 한
AWS env 마커에서만 암묵적 provider를 자동 활성화합니다.

IMDS 기반 호스트에 권장되는 설정:

- `plugins.entries.amazon-bedrock.config.discovery.enabled`를 `true`로 설정하세요.
- `plugins.entries.amazon-bedrock.config.discovery.region`을 설정하세요(또는 `AWS_REGION` export).
- 가짜 API 키는 **필요하지 않습니다**.
- 자동 모드 또는 상태 표면용 env 마커가 특별히 필요한 경우에만
  `AWS_PROFILE=default`가 필요합니다.

```bash
# 권장: 탐지를 명시적으로 활성화하고 리전 설정
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 선택 사항: 명시적 활성화 없이 자동 모드를 원하면 env 마커 추가
export AWS_PROFILE=default
export AWS_REGION=us-east-1
```

EC2 인스턴스 역할에 필요한 **필수 IAM 권한**:

- `bedrock:InvokeModel`
- `bedrock:InvokeModelWithResponseStream`
- `bedrock:ListFoundationModels` (자동 탐지용)
- `bedrock:ListInferenceProfiles` (추론 프로필 탐지용)

또는 관리형 정책 `AmazonBedrockFullAccess`를 연결하세요.

## 빠른 설정(AWS 경로)

```bash
# 1. IAM 역할 및 인스턴스 프로필 생성
aws iam create-role --role-name EC2-Bedrock-Access \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy --role-name EC2-Bedrock-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonBedrockFullAccess

aws iam create-instance-profile --instance-profile-name EC2-Bedrock-Access
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-Bedrock-Access \
  --role-name EC2-Bedrock-Access

# 2. EC2 인스턴스에 연결
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxx \
  --iam-instance-profile Name=EC2-Bedrock-Access

# 3. EC2 인스턴스에서 탐지를 명시적으로 활성화
openclaw config set plugins.entries.amazon-bedrock.config.discovery.enabled true
openclaw config set plugins.entries.amazon-bedrock.config.discovery.region us-east-1

# 4. 선택 사항: 명시적 활성화 없이 자동 모드를 원하면 env 마커 추가
echo 'export AWS_PROFILE=default' >> ~/.bashrc
echo 'export AWS_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 5. 모델이 탐지되는지 확인
openclaw models list
```

## 추론 프로필

OpenClaw는 foundation 모델과 함께 **리전 및 글로벌 추론 프로필**도 탐지합니다. 프로필이 알려진 foundation 모델에 매핑되면 해당
프로필은 그 모델의 capability(컨텍스트 윈도우, 최대 토큰,
reasoning, 비전)를 상속하고, 올바른 Bedrock 요청 리전도 자동으로
주입됩니다. 즉, 교차 리전 Claude 프로필이 수동
provider override 없이도 동작합니다.

추론 프로필 ID는 `us.anthropic.claude-opus-4-6-v1:0`(리전)
또는 `anthropic.claude-opus-4-6-v1:0`(글로벌)처럼 보입니다. 백엔드 모델이 이미
탐지 결과에 있으면 프로필은 해당 모델의 전체 capability 세트를 상속합니다.
그렇지 않으면 안전한 기본값이 적용됩니다.

추가 config는 필요하지 않습니다. 탐지가 활성화되어 있고 IAM
principal에 `bedrock:ListInferenceProfiles` 권한이 있다면, 프로필은
`openclaw models list`에서 foundation 모델과 함께 표시됩니다.

## 참고

- Bedrock은 AWS 계정/리전에서 **모델 접근**이 활성화되어 있어야 합니다.
- 자동 탐지에는 `bedrock:ListFoundationModels` 및
  `bedrock:ListInferenceProfiles` 권한이 필요합니다.
- 자동 모드에 의존하는 경우 게이트웨이 호스트에 지원되는 AWS auth env 마커 중 하나를 설정하세요.
  env 마커 없이 IMDS/공유 config auth를 선호한다면
  `plugins.entries.amazon-bedrock.config.discovery.enabled: true`를 설정하세요.
- OpenClaw는 자격 증명 소스를 다음 순서로 표시합니다: `AWS_BEARER_TOKEN_BEDROCK`,
  그다음 `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`, 그다음 `AWS_PROFILE`, 마지막으로
  기본 AWS SDK 체인.
- reasoning 지원 여부는 모델에 따라 다르므로,
  현재 capability는 Bedrock 모델 카드를 확인하세요.
- 관리형 키 흐름을 선호한다면 Bedrock 앞에 OpenAI 호환
  프록시를 두고 이를 OpenAI provider로 구성할 수도 있습니다.

## Guardrails

`amazon-bedrock` plugin config에 `guardrail` 객체를 추가하여
모든 Bedrock 모델 호출에 [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)를 적용할 수 있습니다.
Guardrails를 사용하면 콘텐츠 필터링,
주제 차단, 단어 필터, 민감 정보 필터, 컨텍스트 기반
그라운딩 검사를 강제할 수 있습니다.

```json5
{
  plugins: {
    entries: {
      "amazon-bedrock": {
        config: {
          guardrail: {
            guardrailIdentifier: "abc123", // 가드레일 ID 또는 전체 ARN
            guardrailVersion: "1", // 버전 번호 또는 "DRAFT"
            streamProcessingMode: "sync", // 선택 사항: "sync" 또는 "async"
            trace: "enabled", // 선택 사항: "enabled", "disabled", 또는 "enabled_full"
          },
        },
      },
    },
  },
}
```

- `guardrailIdentifier`(필수)는 가드레일 ID(예: `abc123`) 또는
  전체 ARN(예: `arn:aws:bedrock:us-east-1:123456789012:guardrail/abc123`)을 허용합니다.
- `guardrailVersion`(필수)은 사용할 게시된 버전 또는
  작업 중인 초안인 `"DRAFT"`를 지정합니다.
- `streamProcessingMode`(선택 사항)는 스트리밍 중 가드레일 평가를
  동기식(`"sync"`) 또는 비동기식(`"async"`)으로 실행할지 제어합니다. 이를
  생략하면 Bedrock이 기본 동작을 사용합니다.
- `trace`(선택 사항)는 API 응답에서 가드레일 trace 출력을 활성화합니다.
  디버깅에는 `"enabled"` 또는 `"enabled_full"`로 설정하고, 프로덕션에서는
  생략하거나 `"disabled"`로 설정하세요.

게이트웨이가 사용하는 IAM principal에는 표준 invoke 권한 외에도
`bedrock:ApplyGuardrail` 권한이 있어야 합니다.

## 메모리 검색용 임베딩

Bedrock은 [memory search](/ko/concepts/memory-search)의
임베딩 provider로도 사용할 수 있습니다. 이는 추론 provider와 별도로
구성되며, `agents.defaults.memorySearch.provider`를 `"bedrock"`으로 설정하세요:

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "bedrock",
        model: "amazon.titan-embed-text-v2:0", // 기본값
      },
    },
  },
}
```

Bedrock 임베딩은 추론과 동일한 AWS SDK 자격 증명 체인(인스턴스
역할, SSO, 액세스 키, 공유 config, 웹 ID)을 사용합니다. API 키는
필요하지 않습니다. `provider`가 `"auto"`이면 해당
자격 증명 체인이 성공적으로 해석될 때 Bedrock이 자동 탐지됩니다.

지원되는 임베딩 모델에는 Amazon Titan Embed(v1, v2), Amazon Nova
Embed, Cohere Embed(v3, v4), TwelveLabs Marengo가 포함됩니다.
전체 모델 목록과 차원 옵션은
[Memory configuration reference — Bedrock](/ko/reference/memory-config#bedrock-embedding-config)을 참조하세요.
