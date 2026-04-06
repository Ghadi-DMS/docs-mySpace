---
read_when:
    - OpenClaw OAuth의 엔드 투 엔드 동작을 이해하고 싶습니다
    - 토큰 무효화 / 로그아웃 문제를 겪고 있습니다
    - Claude CLI 또는 OAuth 인증 흐름이 필요합니다
    - 여러 계정 또는 프로필 라우팅이 필요합니다
summary: 'OpenClaw의 OAuth: 토큰 교환, 저장소, 다중 계정 패턴'
title: OAuth
x-i18n:
    generated_at: "2026-04-06T03:07:13Z"
    model: gpt-5.4
    provider: openai
    source_hash: 402e20dfeb6ae87a90cba5824a56a7ba3b964f3716508ea5cc48a47e5affdd73
    source_path: concepts/oauth.md
    workflow: 15
---

# OAuth

OpenClaw는 이를 제공하는 provider에 대해 OAuth를 통한 “구독 인증”을 지원합니다
(특히 **OpenAI Codex (ChatGPT OAuth)**). Anthropic의 경우 현재 실질적인 구분은 다음과 같습니다:

- **Anthropic API key**: 일반 Anthropic API 과금
- **OpenClaw 내부의 Anthropic 구독 인증**: Anthropic은 OpenClaw
  사용자에게 **2026년 4월 4일 오후 12:00 PT / 오후 8:00 BST**에 이것이 이제
  **Extra Usage**를 요구한다고 알렸습니다

OpenAI Codex OAuth는 OpenClaw 같은 외부 도구에서 사용하도록 명시적으로 지원됩니다.
이 페이지에서는 다음을 설명합니다:

프로덕션에서 Anthropic의 경우 API key 인증이 더 안전하며 권장되는 경로입니다.

- OAuth **토큰 교환**이 작동하는 방식(PKCE)
- 토큰이 **저장**되는 위치(그리고 그 이유)
- **여러 계정**을 처리하는 방법(프로필 + 세션별 재정의)

OpenClaw는 자체 OAuth 또는 API‑key
흐름을 제공하는 **provider plugins**도 지원합니다. 다음으로 실행하세요:

```bash
openclaw models auth login --provider <id>
```

## 토큰 싱크(존재하는 이유)

OAuth provider는 로그인/갱신 흐름 중에 흔히 **새 refresh token**을 발급합니다. 일부 provider(또는 OAuth 클라이언트)는 같은 사용자/앱에 대해 새 토큰이 발급되면 이전 refresh token을 무효화할 수 있습니다.

실제 증상:

- OpenClaw _및_ Claude Code / Codex CLI를 통해 로그인하면 → 나중에 둘 중 하나가 무작위로 “로그아웃”됩니다

이를 줄이기 위해 OpenClaw는 `auth-profiles.json`을 **토큰 싱크**로 취급합니다:

- 런타임은 **한 곳**에서 자격 증명을 읽습니다
- 여러 프로필을 유지하고 이를 결정론적으로 라우팅할 수 있습니다
- Codex CLI 같은 외부 CLI에서 자격 증명을 재사용하는 경우 OpenClaw는
  출처 정보를 포함해 이를 미러링하고 refresh token 자체를 회전하는 대신
  그 외부 소스를 다시 읽습니다

## 저장소(토큰이 저장되는 위치)

시크릿은 **에이전트별**로 저장됩니다:

- 인증 프로필(OAuth + API keys + 선택적 값 수준 refs): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- 레거시 호환성 파일: `~/.openclaw/agents/<agentId>/agent/auth.json`
  (`api_key` 고정 항목은 발견되면 제거됨)

레거시 가져오기 전용 파일(여전히 지원되지만 기본 저장소는 아님):

- `~/.openclaw/credentials/oauth.json` (첫 사용 시 `auth-profiles.json`으로 가져옴)

위의 모든 항목은 `$OPENCLAW_STATE_DIR`(state dir 재정의)도 따릅니다. 전체 참조: [/gateway/configuration](/ko/gateway/configuration-reference#auth-storage)

정적 secret refs와 런타임 스냅샷 활성화 동작에 대해서는 [Secrets Management](/ko/gateway/secrets)를 참조하세요.

## Anthropic 레거시 토큰 호환성

<Warning>
Anthropic의 공개 Claude Code 문서에서는 Claude Code를 직접 사용하는 경우
Claude 구독 한도 내에 머문다고 설명합니다. 별도로 Anthropic은
**2026년 4월 4일 오후 12:00 PT / 오후 8:00 BST**에 OpenClaw 사용자에게
**OpenClaw를 서드파티 harness로 간주한다**고 알렸습니다.
기존 Anthropic 토큰 프로필은 기술적으로 여전히 OpenClaw에서 사용할 수 있지만,
Anthropic은 이제 OpenClaw 경로의 해당 트래픽에 대해 **Extra
Usage**(구독과 별도로 과금되는 종량제)를 요구한다고 밝히고 있습니다.

Anthropic의 현재 직접 Claude Code 플랜 문서는 [Using Claude Code
with your Pro or Max
plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
및 [Using Claude Code with your Team or Enterprise
plan](https://support.anthropic.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan/)을
참조하세요.

OpenClaw에서 다른 구독형 옵션을 원한다면 [OpenAI
Codex](/ko/providers/openai), [Qwen Cloud Coding
Plan](/ko/providers/qwen), [MiniMax Coding Plan](/ko/providers/minimax),
그리고 [Z.AI / GLM Coding Plan](/ko/providers/glm)을 참조하세요.
</Warning>

OpenClaw는 이제 Anthropic setup-token을 레거시/수동 경로로 다시 노출합니다.
Anthropic의 OpenClaw 전용 과금 고지는 이 경로에도 여전히 적용되므로,
Anthropic이 OpenClaw 기반 Claude 로그인 트래픽에 대해 **Extra Usage**를 요구한다는
전제하에 사용하세요.

## Anthropic Claude CLI 마이그레이션

Anthropic은 더 이상 OpenClaw에서 지원되는 로컬 Claude CLI 마이그레이션 경로를 제공하지 않습니다.
Anthropic 트래픽에는 Anthropic API keys를 사용하거나, 이미 구성된 경우에만
레거시 토큰 기반 인증을 유지하되 Anthropic이 해당 OpenClaw 경로를
**Extra Usage**로 취급한다는 점을 전제로 하세요.

## OAuth 교환(로그인 작동 방식)

OpenClaw의 대화형 로그인 흐름은 `@mariozechner/pi-ai`에 구현되어 있으며 wizard/command에 연결되어 있습니다.

### Anthropic setup-token

흐름 형태:

1. OpenClaw에서 Anthropic setup-token 또는 paste-token 시작
2. OpenClaw가 결과 Anthropic 자격 증명을 인증 프로필에 저장
3. 모델 선택은 `anthropic/...`에 유지
4. 기존 Anthropic 인증 프로필은 롤백/순서 제어를 위해 계속 사용 가능

### OpenAI Codex (ChatGPT OAuth)

OpenAI Codex OAuth는 OpenClaw 워크플로를 포함해 Codex CLI 외부에서 사용할 수 있도록 명시적으로 지원됩니다.

흐름 형태(PKCE):

1. PKCE verifier/challenge + 무작위 `state` 생성
2. `https://auth.openai.com/oauth/authorize?...` 열기
3. `http://127.0.0.1:1455/auth/callback`에서 콜백 캡처 시도
4. 콜백에 바인드할 수 없거나 원격/헤드리스 환경이라면 리디렉션 URL/code를 붙여넣기
5. `https://auth.openai.com/oauth/token`에서 교환
6. access token에서 `accountId`를 추출하고 `{ access, refresh, expires, accountId }` 저장

wizard 경로는 `openclaw onboard` → auth choice `openai-codex`입니다.

## 갱신 + 만료

프로필은 `expires` 타임스탬프를 저장합니다.

런타임 시:

- `expires`가 미래 시점이면 → 저장된 access token 사용
- 만료되었으면 → 갱신(파일 잠금 하에) 후 저장된 자격 증명 덮어쓰기
- 예외: 재사용된 외부 CLI 자격 증명은 외부에서 계속 관리되며 OpenClaw는
  CLI 인증 저장소를 다시 읽고 복사된 refresh token 자체는 절대 사용하지 않습니다

갱신 흐름은 자동이므로 일반적으로 토큰을 수동으로 관리할 필요가 없습니다.

## 여러 계정(프로필) + 라우팅

두 가지 패턴이 있습니다:

### 1) 권장: 별도 에이전트

“개인용”과 “업무용”이 절대로 상호작용하지 않게 하려면 격리된 에이전트(별도 세션 + 자격 증명 + 워크스페이스)를 사용하세요:

```bash
openclaw agents add work
openclaw agents add personal
```

그런 다음 에이전트별로 인증을 구성하고(wizard), 채팅을 올바른 에이전트로 라우팅하세요.

### 2) 고급: 하나의 에이전트에서 여러 프로필

`auth-profiles.json`은 같은 provider에 대해 여러 프로필 ID를 지원합니다.

어떤 프로필을 사용할지 선택하는 방법:

- 전역적으로는 config 순서(`auth.order`)를 통해
- 세션별로는 `/model ...@<profileId>`를 통해

예시(세션 재정의):

- `/model Opus@anthropic:work`

어떤 프로필 ID가 있는지 확인하는 방법:

- `openclaw channels list --json` (`auth[]` 표시)

관련 문서:

- [/concepts/model-failover](/ko/concepts/model-failover) (회전 + 쿨다운 규칙)
- [/tools/slash-commands](/ko/tools/slash-commands) (command 표면)

## 관련

- [Authentication](/ko/gateway/authentication) — 모델 provider 인증 개요
- [Secrets](/ko/gateway/secrets) — 자격 증명 저장소와 SecretRef
- [Configuration Reference](/ko/gateway/configuration-reference#auth-storage) — 인증 config 키
