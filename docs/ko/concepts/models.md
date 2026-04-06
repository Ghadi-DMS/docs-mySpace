---
read_when:
    - models CLI(models list/set/scan/aliases/fallbacks)를 추가하거나 수정할 때
    - 모델 폴백 동작 또는 선택 UX를 변경할 때
    - 모델 스캔 프로브(tools/images)를 업데이트할 때
summary: 'Models CLI: 목록, 설정, 별칭, 폴백, 스캔, 상태'
title: Models CLI
x-i18n:
    generated_at: "2026-04-06T03:07:14Z"
    model: gpt-5.4
    provider: openai
    source_hash: 299602ccbe0c3d6bbdb2deab22bc60e1300ef6843ed0b8b36be574cc0213c155
    source_path: concepts/models.md
    workflow: 15
---

# Models CLI

인증 프로필
로테이션, 쿨다운, 그리고 이것이 폴백과 어떻게 상호작용하는지는 [/concepts/model-failover](/ko/concepts/model-failover)를 참고하세요.
빠른 공급자 개요와 예시는 [/concepts/model-providers](/ko/concepts/model-providers)를 참고하세요.

## 모델 선택 방식

OpenClaw는 다음 순서로 모델을 선택합니다.

1. **Primary** 모델(`agents.defaults.model.primary` 또는 `agents.defaults.model`)
2. `agents.defaults.model.fallbacks`의 **Fallbacks**(순서대로)
3. **공급자 인증 페일오버**는 다음 모델로 이동하기 전에 공급자 내부에서 발생

관련 항목:

- `agents.defaults.models`는 OpenClaw가 사용할 수 있는 모델의 허용 목록/카탈로그입니다(별칭 포함).
- `agents.defaults.imageModel`은 **Primary 모델이 이미지를 받을 수 없을 때만** 사용됩니다.
- `agents.defaults.pdfModel`은 `pdf` 도구에서 사용됩니다. 생략하면 이 도구는 `agents.defaults.imageModel`, 그다음 해석된 세션/기본 모델로 폴백됩니다.
- `agents.defaults.imageGenerationModel`은 공용 이미지 생성 기능에서 사용됩니다. 생략해도 `image_generate`는 인증이 설정된 공급자 기본값을 추론할 수 있습니다. 현재 기본 공급자를 먼저 시도한 다음, 남아 있는 등록된 이미지 생성 공급자를 공급자 ID 순서대로 시도합니다. 특정 공급자/모델을 설정하는 경우 해당 공급자의 인증/API 키도 함께 구성하세요.
- `agents.defaults.musicGenerationModel`은 공용 음악 생성 기능에서 사용됩니다. 생략해도 `music_generate`는 인증이 설정된 공급자 기본값을 추론할 수 있습니다. 현재 기본 공급자를 먼저 시도한 다음, 남아 있는 등록된 음악 생성 공급자를 공급자 ID 순서대로 시도합니다. 특정 공급자/모델을 설정하는 경우 해당 공급자의 인증/API 키도 함께 구성하세요.
- `agents.defaults.videoGenerationModel`은 공용 비디오 생성 기능에서 사용됩니다. 생략해도 `video_generate`는 인증이 설정된 공급자 기본값을 추론할 수 있습니다. 현재 기본 공급자를 먼저 시도한 다음, 남아 있는 등록된 비디오 생성 공급자를 공급자 ID 순서대로 시도합니다. 특정 공급자/모델을 설정하는 경우 해당 공급자의 인증/API 키도 함께 구성하세요.
- 에이전트별 기본값은 `agents.list[].model`과 바인딩을 통해 `agents.defaults.model`을 재정의할 수 있습니다([/concepts/multi-agent](/ko/concepts/multi-agent) 참고).

## 빠른 모델 정책

- Primary는 사용 가능한 것 중 가장 강력한 최신 세대 모델로 설정하세요.
- 폴백은 비용/지연 시간에 민감한 작업과 중요도가 낮은 채팅에 사용하세요.
- 도구가 활성화된 에이전트나 신뢰할 수 없는 입력의 경우, 오래되었거나 약한 모델 티어는 피하세요.

## 온보딩(권장)

구성을 직접 수정하고 싶지 않다면 온보딩을 실행하세요.

```bash
openclaw onboard
```

이 명령은 **OpenAI Code (Codex) subscription**(OAuth)와 **Anthropic**(API 키 또는 Claude CLI)을 포함한 일반적인 공급자의 모델 + 인증을 설정할 수 있습니다.

## 구성 키(개요)

- `agents.defaults.model.primary` 및 `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel.primary` 및 `agents.defaults.imageModel.fallbacks`
- `agents.defaults.pdfModel.primary` 및 `agents.defaults.pdfModel.fallbacks`
- `agents.defaults.imageGenerationModel.primary` 및 `agents.defaults.imageGenerationModel.fallbacks`
- `agents.defaults.videoGenerationModel.primary` 및 `agents.defaults.videoGenerationModel.fallbacks`
- `agents.defaults.models`(허용 목록 + 별칭 + 공급자 파라미터)
- `models.providers`(`models.json`에 기록되는 사용자 지정 공급자)

모델 ref는 소문자로 정규화됩니다. `z.ai/*` 같은 공급자 별칭은 `zai/*`로 정규화됩니다.

공급자 구성 예시(OpenCode 포함)는 [/providers/opencode](/ko/providers/opencode)에 있습니다.

## "Model is not allowed"가 표시되는 경우(그리고 왜 응답이 멈추는지)

`agents.defaults.models`가 설정되어 있으면 `/model`과 세션 재정의에 대한 **허용 목록**이 됩니다. 사용자가 그 허용 목록에 없는 모델을 선택하면 OpenClaw는 다음을 반환합니다.

```
Model "provider/model" is not allowed. Use /model to list available models.
```

이것은 일반 응답이 생성되기 **전에** 발생하므로, 메시지에 “응답하지 않은 것처럼” 느껴질 수 있습니다. 해결 방법은 다음 중 하나입니다.

- 모델을 `agents.defaults.models`에 추가
- 허용 목록 비우기(`agents.defaults.models` 제거)
- `/model list`에서 모델 선택

허용 목록 구성 예시:

```json5
{
  agent: {
    model: { primary: "anthropic/claude-sonnet-4-6" },
    models: {
      "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
      "anthropic/claude-opus-4-6": { alias: "Opus" },
    },
  },
}
```

## 채팅에서 모델 전환(`/model`)

다시 시작하지 않고 현재 세션의 모델을 전환할 수 있습니다.

```
/model
/model list
/model 3
/model openai/gpt-5.4
/model status
```

참고:

- `/model`(및 `/model list`)은 간결한 번호형 선택기입니다(모델 패밀리 + 사용 가능한 공급자).
- Discord에서는 `/model`과 `/models`가 공급자 및 모델 드롭다운과 Submit 단계가 있는 대화형 선택기를 엽니다.
- `/model <#>`는 해당 선택기에서 선택합니다.
- `/model`은 새로운 세션 선택을 즉시 저장합니다.
- 에이전트가 유휴 상태이면 다음 실행에서 바로 새 모델을 사용합니다.
- 이미 실행이 진행 중이면 OpenClaw는 실시간 전환을 보류 상태로 표시하고, 깔끔한 재시도 지점에서만 새 모델로 다시 시작합니다.
- 도구 활동이나 응답 출력이 이미 시작되었다면, 보류 중인 전환은 나중 재시도 기회나 다음 사용자 턴까지 대기할 수 있습니다.
- `/model status`는 상세 보기입니다(인증 후보, 그리고 구성된 경우 공급자 엔드포인트 `baseUrl` + `api` 모드).
- 모델 ref는 **첫 번째** `/`를 기준으로 분리하여 파싱합니다. `/model <ref>`를 입력할 때는 `provider/model`을 사용하세요.
- 모델 ID 자체에 `/`가 포함되어 있으면(OpenRouter 스타일), 공급자 접두사를 포함해야 합니다(예: `/model openrouter/moonshotai/kimi-k2`).
- 공급자를 생략하면 OpenClaw는 다음 순서로 입력을 해석합니다.
  1. 별칭 일치
  2. 정확히 동일한 공급자 없는 모델 ID에 대한 고유한 configured-provider 일치
  3. 구성된 기본 공급자로의 deprecated 폴백  
     해당 공급자가 더 이상 구성된 기본 모델을 제공하지 않는 경우, OpenClaw는 오래된 제거된 공급자 기본값이 노출되지 않도록 대신 첫 번째 구성된 공급자/모델로 폴백합니다.

전체 명령 동작/구성: [Slash commands](/ko/tools/slash-commands)

## CLI 명령

```bash
openclaw models list
openclaw models status
openclaw models set <provider/model>
openclaw models set-image <provider/model>

openclaw models aliases list
openclaw models aliases add <alias> <provider/model>
openclaw models aliases remove <alias>

openclaw models fallbacks list
openclaw models fallbacks add <provider/model>
openclaw models fallbacks remove <provider/model>
openclaw models fallbacks clear

openclaw models image-fallbacks list
openclaw models image-fallbacks add <provider/model>
openclaw models image-fallbacks remove <provider/model>
openclaw models image-fallbacks clear
```

`openclaw models`(하위 명령 없음)는 `models status`의 바로가기입니다.

### `models list`

기본적으로 구성된 모델을 표시합니다. 유용한 플래그:

- `--all`: 전체 카탈로그
- `--local`: 로컬 공급자만
- `--provider <name>`: 공급자별 필터링
- `--plain`: 한 줄에 모델 하나
- `--json`: 기계 판독 가능 출력

### `models status`

해석된 Primary 모델, 폴백, 이미지 모델, 그리고 구성된 공급자의 인증 개요를 표시합니다. 또한 인증 저장소에서 찾은 프로필의 OAuth 만료 상태도 표시합니다(기본적으로 24시간 이내면 경고). `--plain`은 해석된 Primary 모델만 출력합니다.
OAuth 상태는 항상 표시되며(`--json` 출력에도 포함됨), 구성된 공급자에 자격 증명이 없으면 `models status`는 **Missing auth** 섹션을 출력합니다.
JSON에는 `auth.oauth`(경고 기간 + 프로필)와 `auth.providers`(환경 변수 기반 자격 증명을 포함한 공급자별 유효 인증)가 포함됩니다. `auth.oauth`는 인증 저장소 프로필 상태만 나타내며, 환경 변수만 사용하는 공급자는 여기에 나타나지 않습니다.
자동화에는 `--check`를 사용하세요(누락/만료 시 종료 코드 `1`, 만료 임박 시 `2`).
실시간 인증 검사에는 `--probe`를 사용하세요. 프로브 행은 인증 프로필, 환경 변수 자격 증명 또는 `models.json`에서 올 수 있습니다.
명시적 `auth.order.<provider>`에 저장된 프로필이 빠져 있으면, 프로브는 시도하는 대신 `excluded_by_auth_order`를 보고합니다. 인증은 있지만 해당 공급자에 대해 프로브 가능한 모델을 해석할 수 없으면, 프로브는 `status: no_model`을 보고합니다.

인증 선택은 공급자/계정에 따라 다릅니다. 항상 켜져 있는 gateway host의 경우 API 키가 보통 가장 예측 가능하며, Claude CLI 재사용 및 기존 Anthropic OAuth/토큰 프로필도 지원됩니다.

예시(Claude CLI):

```bash
claude auth login
openclaw models status
```

## 스캔(OpenRouter 무료 모델)

`openclaw models scan`은 OpenRouter의 **무료 모델 카탈로그**를 검사하고, 필요에 따라 도구 및 이미지 지원 여부를 프로브할 수 있습니다.

주요 플래그:

- `--no-probe`: 실시간 프로브 건너뛰기(메타데이터만)
- `--min-params <b>`: 최소 파라미터 크기(십억 단위)
- `--max-age-days <days>`: 오래된 모델 건너뛰기
- `--provider <name>`: 공급자 접두사 필터
- `--max-candidates <n>`: 폴백 목록 크기
- `--set-default`: `agents.defaults.model.primary`를 첫 번째 선택으로 설정
- `--set-image`: `agents.defaults.imageModel.primary`를 첫 번째 이미지 선택으로 설정

프로빙에는 OpenRouter API 키(인증 프로필 또는 `OPENROUTER_API_KEY`)가 필요합니다. 키가 없으면 `--no-probe`를 사용해 후보만 나열하세요.

스캔 결과 순위 기준:

1. 이미지 지원
2. 도구 지연 시간
3. 컨텍스트 크기
4. 파라미터 수

입력

- OpenRouter `/models` 목록(`:free` 필터)
- 인증 프로필 또는 `OPENROUTER_API_KEY`의 OpenRouter API 키 필요([/environment](/ko/help/environment) 참고)
- 선택적 필터: `--max-age-days`, `--min-params`, `--provider`, `--max-candidates`
- 프로브 제어: `--timeout`, `--concurrency`

TTY에서 실행하면 대화형으로 폴백을 선택할 수 있습니다. 비대화형 모드에서는 `--yes`를 전달해 기본값을 수락하세요.

## 모델 레지스트리(`models.json`)

`models.providers`의 사용자 지정 공급자는 에이전트 디렉터리(기본값 `~/.openclaw/agents/<agentId>/agent/models.json`) 아래 `models.json`에 기록됩니다. `models.mode`가 `replace`로 설정되지 않는 한 이 파일은 기본적으로 병합됩니다.

일치하는 공급자 ID에 대한 병합 모드 우선순위:

- 에이전트 `models.json`에 이미 있는 비어 있지 않은 `baseUrl`이 우선합니다.
- 에이전트 `models.json`의 비어 있지 않은 `apiKey`는 현재 구성/인증 프로필 컨텍스트에서 해당 공급자가 SecretRef로 관리되지 않을 때만 우선합니다.
- SecretRef로 관리되는 공급자 `apiKey` 값은 해석된 시크릿을 저장하는 대신 소스 마커(`ENV_VAR_NAME`은 env ref, `secretref-managed`는 file/exec ref)로부터 새로 고쳐집니다.
- SecretRef로 관리되는 공급자 헤더 값은 소스 마커(`secretref-env:ENV_VAR_NAME`은 env ref, `secretref-managed`는 file/exec ref)로부터 새로 고쳐집니다.
- 비어 있거나 없는 에이전트 `apiKey`/`baseUrl`은 구성 `models.providers`로 폴백됩니다.
- 다른 공급자 필드는 구성과 정규화된 카탈로그 데이터로 새로 고쳐집니다.

마커 저장은 소스 권위를 따릅니다. OpenClaw는 해석된 런타임 시크릿 값이 아니라 활성 소스 구성 스냅샷(해석 전)에서 마커를 기록합니다.
이는 `openclaw agent` 같은 명령 기반 경로를 포함하여 OpenClaw가 `models.json`을 다시 생성할 때마다 적용됩니다.

## 관련 항목

- [Model Providers](/ko/concepts/model-providers) — 공급자 라우팅 및 인증
- [Model Failover](/ko/concepts/model-failover) — 폴백 체인
- [Image Generation](/ko/tools/image-generation) — 이미지 모델 구성
- [Music Generation](/tools/music-generation) — 음악 모델 구성
- [Video Generation](/tools/video-generation) — 비디오 모델 구성
- [Configuration Reference](/ko/gateway/configuration-reference#agent-defaults) — 모델 구성 키
