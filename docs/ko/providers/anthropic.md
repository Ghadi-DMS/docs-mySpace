---
read_when:
    - OpenClaw에서 Anthropic 모델을 사용하고 싶을 때
summary: OpenClaw에서 API key를 통해 Anthropic Claude 사용하기
title: Anthropic
x-i18n:
    generated_at: "2026-04-06T03:10:52Z"
    model: gpt-5.4
    provider: openai
    source_hash: bbc6c4938674aedf20ff944bc04e742c9a7e77a5ff10ae4f95b5718504c57c2d
    source_path: providers/anthropic.md
    workflow: 15
---

# Anthropic (Claude)

Anthropic은 **Claude** 모델 계열을 개발하며 API를 통해 접근을 제공합니다.
OpenClaw에서 새로운 Anthropic 설정은 API key를 사용해야 합니다. 기존의 legacy
Anthropic token profile은 이미 설정되어 있다면 런타임에서 계속
인정됩니다.

<Warning>
OpenClaw에서 Anthropic의 과금 구분은 다음과 같습니다.

- **Anthropic API key**: 일반 Anthropic API 과금
- **OpenClaw 내부의 Claude subscription auth**: Anthropic은 **2026년 4월 4일 오후 12:00 PT / 오후 8:00 BST**에 OpenClaw 사용자에게 이것이
  서드파티 harness 사용으로 간주되며 **Extra Usage**(종량제,
  subscription과 별도 청구)가 필요하다고 알렸습니다.

로컬 재현 결과도 이 구분과 일치합니다.

- direct `claude -p`는 여전히 동작할 수 있음
- `claude -p --append-system-prompt ...`는 프롬프트가 OpenClaw를 식별하면
  Extra Usage 가드를 트리거할 수 있음
- 같은 OpenClaw 유사 시스템 프롬프트도
  Anthropic SDK + `ANTHROPIC_API_KEY` 경로에서는 차단이 재현되지 않음

따라서 실질적인 규칙은 다음과 같습니다. **Anthropic API key, 또는 Extra Usage가 있는 Claude subscription**.
가장 명확한 프로덕션 경로를 원한다면 Anthropic API
key를 사용하세요.

Anthropic의 현재 공개 문서:

- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Claude Agent SDK overview](https://platform.claude.com/docs/en/agent-sdk/overview)

- [Using Claude Code with your Pro or Max plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [Using Claude Code with your Team or Enterprise plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)

가장 명확한 과금 경로를 원한다면 Anthropic API key를 대신 사용하세요.
OpenClaw는 [OpenAI
Codex](/ko/providers/openai), [Qwen Cloud Coding Plan](/ko/providers/qwen),
[MiniMax Coding Plan](/ko/providers/minimax), [Z.AI / GLM Coding
Plan](/ko/providers/glm)을 포함한 다른 subscription 스타일 옵션도 지원합니다.
</Warning>

## 옵션 A: Anthropic API key

**가장 적합한 경우:** 표준 API 접근 및 사용량 기반 과금.
Anthropic Console에서 API key를 생성하세요.

### CLI 설정

```bash
openclaw onboard
# choose: Anthropic API key

# 또는 비대화형
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### Anthropic config 스니펫

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## thinking 기본값 (Claude 4.6)

- Anthropic Claude 4.6 모델은 명시적인 thinking 수준이 설정되지 않으면 OpenClaw에서 기본적으로 `adaptive` thinking을 사용합니다.
- 메시지별로(`/think:<level>`) 또는 모델 params에서 override할 수 있습니다:
  `agents.defaults.models["anthropic/<model>"].params.thinking`.
- 관련 Anthropic 문서:
  - [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
  - [Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## 빠른 모드 (Anthropic API)

OpenClaw의 공용 `/fast` 토글은 direct public Anthropic 트래픽도 지원하며, `api.anthropic.com`으로 전송되는 API-key 및 OAuth 인증 요청도 포함됩니다.

- `/fast on`은 `service_tier: "auto"`로 매핑됩니다
- `/fast off`는 `service_tier: "standard_only"`로 매핑됩니다
- Config 기본값:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-sonnet-4-6": {
          params: { fastMode: true },
        },
      },
    },
  },
}
```

중요한 제한 사항:

- OpenClaw는 direct `api.anthropic.com` 요청에 대해서만 Anthropic service tier를 주입합니다. `anthropic/*`를 proxy나 gateway를 통해 라우팅하면 `/fast`는 `service_tier`를 변경하지 않습니다.
- 명시적인 Anthropic `serviceTier` 또는 `service_tier` 모델 params가 설정되어 있으면, 둘 다 있을 때 `/fast` 기본값보다 우선합니다.
- Anthropic은 응답의 `usage.service_tier` 아래에 실제 적용된 tier를 보고합니다. Priority Tier 용량이 없는 계정에서는 `service_tier: "auto"`가 여전히 `standard`로 해석될 수 있습니다.

## 프롬프트 캐싱 (Anthropic API)

OpenClaw는 Anthropic의 프롬프트 캐싱 기능을 지원합니다. 이는 **API 전용**이며, legacy Anthropic token auth는 cache 설정을 반영하지 않습니다.

### 구성

모델 config에서 `cacheRetention` 파라미터를 사용하세요.

| Value   | Cache Duration | Description              |
| ------- | -------------- | ------------------------ |
| `none`  | 캐싱 없음      | 프롬프트 캐싱 비활성화   |
| `short` | 5분            | API Key auth의 기본값    |
| `long`  | 1시간          | 확장 캐시                |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

### 기본값

Anthropic API Key 인증을 사용할 때 OpenClaw는 모든 Anthropic 모델에 대해 `cacheRetention: "short"`(5분 캐시)를 자동으로 적용합니다. config에서 `cacheRetention`을 명시적으로 설정하면 이를 override할 수 있습니다.

### 에이전트별 `cacheRetention` override

모델 수준 params를 기준선으로 사용한 다음, 특정 에이전트는 `agents.list[].params`로 override하세요.

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" }, // 대부분의 에이전트를 위한 기준선
        },
      },
    },
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } }, // 이 에이전트만 override
    ],
  },
}
```

캐시 관련 params의 config 병합 순서:

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params` (`id`가 일치하는 항목, 키별로 override)

이렇게 하면 같은 모델을 사용하는 한 에이전트는 장시간 캐시를 유지하고, 다른 에이전트는 bursty/재사용이 낮은 트래픽에서 쓰기 비용을 피하기 위해 캐싱을 비활성화할 수 있습니다.

### Bedrock Claude 참고

- Bedrock의 Anthropic Claude 모델(`amazon-bedrock/*anthropic.claude*`)은 설정된 경우 `cacheRetention` pass-through를 허용합니다.
- Anthropic이 아닌 Bedrock 모델은 런타임에서 강제로 `cacheRetention: "none"`이 됩니다.
- Anthropic API-key 스마트 기본값은 명시적 값이 없을 때 Claude-on-Bedrock 모델 ref에도 `cacheRetention: "short"`를 기본 적용합니다.

## 1M 컨텍스트 윈도우 (Anthropic beta)

Anthropic의 1M 컨텍스트 윈도우는 beta 게이트로 제공됩니다. OpenClaw에서는 지원되는 Opus/Sonnet 모델별로 `params.context1m: true`를 설정해 활성화합니다.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { context1m: true },
        },
      },
    },
  },
}
```

OpenClaw는 이를 Anthropic 요청의 `anthropic-beta: context-1m-2025-08-07`로 매핑합니다.

이 기능은 해당 모델에 대해 `params.context1m`이 명시적으로 `true`로 설정된 경우에만 활성화됩니다.

요구사항: Anthropic이 해당 자격 증명에서 long-context 사용을 허용해야 합니다
(일반적으로 API key 과금, 또는 Extra Usage가 활성화된 OpenClaw의 Claude-login 경로 / legacy token auth). 그렇지 않으면 Anthropic은 다음을 반환합니다:
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

참고: Anthropic은 현재 legacy Anthropic token auth(`sk-ant-oat-*`)를 사용할 때
`context-1m-*` beta 요청을 거부합니다. 이 legacy auth 모드에서
`context1m: true`를 설정하면 OpenClaw는 경고를 기록하고,
필수 OAuth beta는 유지한 채 context1m beta
header를 건너뛰어 표준 컨텍스트 윈도우로 fallback합니다.

## 제거됨: Claude CLI 백엔드

번들 Anthropic `claude-cli` 백엔드는 제거되었습니다.

- Anthropic의 2026년 4월 4일 공지에 따르면 OpenClaw가 구동하는 Claude-login 트래픽은
  서드파티 harness 사용이며 **Extra Usage**가 필요합니다.
- 로컬 재현에서도 direct
  `claude -p --append-system-prompt ...`가 추가된
  프롬프트가 OpenClaw를 식별하면 같은 가드에 걸릴 수 있음을 확인했습니다.
- 같은 OpenClaw 유사 시스템 프롬프트는
  Anthropic SDK + `ANTHROPIC_API_KEY` 경로에서는 그 가드에 걸리지 않습니다.
- OpenClaw에서 Anthropic 트래픽에는 Anthropic API key를 사용하세요.

## 참고

- Anthropic의 공개 Claude Code 문서는 여전히
  `claude -p` 같은 direct CLI 사용을 문서화하고 있지만, Anthropic이 OpenClaw 사용자에게 별도로 보낸 공지에 따르면
  **OpenClaw** Claude-login 경로는 서드파티 harness 사용이며
  **Extra Usage**(subscription과 별도로 청구되는 종량제)가 필요합니다.
  로컬 재현에서도 direct
  `claude -p --append-system-prompt ...`가 추가된
  프롬프트가 OpenClaw를 식별하면 같은 가드에 걸릴 수 있는 반면, 동일한 프롬프트 형태는
  Anthropic SDK + `ANTHROPIC_API_KEY` 경로에서는 재현되지 않습니다. 프로덕션에서는
  대신 Anthropic API key를 권장합니다.
- Anthropic setup-token은 OpenClaw에서 legacy/manual 경로로 다시 제공됩니다. Anthropic의 OpenClaw 전용 과금 공지는 여전히 적용되므로, 이 경로에는 Anthropic이 **Extra Usage**를 요구한다는 전제하에 사용하세요.
- Auth 세부 정보 및 재사용 규칙은 [/concepts/oauth](/ko/concepts/oauth)에 있습니다.

## 문제 해결

**401 오류 / 토큰이 갑자기 무효화됨**

- Legacy Anthropic token auth는 만료되거나 취소될 수 있습니다.
- 새 설정에서는 Anthropic API key로 마이그레이션하세요.

**provider "anthropic"에 대한 API key를 찾을 수 없음**

- Auth는 **에이전트별**입니다. 새 에이전트는 메인 에이전트의 key를 상속하지 않습니다.
- 해당 에이전트에 대해 온보딩을 다시 실행하거나, gateway
  host에 API key를 설정한 뒤 `openclaw models status`로 확인하세요.

**profile `anthropic:default`에 대한 자격 증명을 찾을 수 없음**

- `openclaw models status`를 실행해 어떤 auth profile이 활성 상태인지 확인하세요.
- 온보딩을 다시 실행하거나, 해당 profile 경로에 API key를 설정하세요.

**사용 가능한 auth profile이 없음(모두 cooldown/unavailable 상태)**

- `openclaw models status --json`에서 `auth.unusableProfiles`를 확인하세요.
- Anthropic rate-limit cooldown은 모델 범위일 수 있으므로, 현재 모델이 cooldown 중이어도 같은 Anthropic의 다른
  모델은 여전히 사용 가능할 수 있습니다.
- 다른 Anthropic profile을 추가하거나 cooldown이 끝날 때까지 기다리세요.

추가 정보: [/gateway/troubleshooting](/ko/gateway/troubleshooting) 및 [/help/faq](/ko/help/faq).
