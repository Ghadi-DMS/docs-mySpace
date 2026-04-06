---
read_when:
    - 모델 인증 또는 OAuth 만료를 디버깅하는 경우
    - 인증 또는 자격 증명 저장소를 문서화하는 경우
summary: '모델 인증: OAuth, API 키, 그리고 레거시 Anthropic setup-token'
title: 인증
x-i18n:
    generated_at: "2026-04-06T03:07:25Z"
    model: gpt-5.4
    provider: openai
    source_hash: f59ede3fcd7e692ad4132287782a850526acf35474b5bfcea29e0e23610636c2
    source_path: gateway/authentication.md
    workflow: 15
---

# 인증(모델 provider)

<Note>
이 페이지는 **모델 provider** 인증(API 키, OAuth, 레거시 Anthropic setup-token)을 다룹니다. **gateway 연결** 인증(token, password, trusted-proxy)은 [구성](/ko/gateway/configuration) 및 [Trusted Proxy Auth](/ko/gateway/trusted-proxy-auth)를 참조하세요.
</Note>

OpenClaw는 모델 provider에 대해 OAuth와 API 키를 지원합니다. 항상 켜져 있는 gateway 호스트에서는 API 키가 보통 가장 예측 가능한 선택지입니다. provider 계정 모델에 맞는 경우 구독/OAuth 흐름도 지원됩니다.

전체 OAuth 흐름과 저장소 레이아웃은 [/concepts/oauth](/ko/concepts/oauth)를 참조하세요.
SecretRef 기반 인증(`env`/`file`/`exec` providers)은 [Secrets Management](/ko/gateway/secrets)를 참조하세요.
`models status --probe`에서 사용하는 자격 증명 적격성/사유 코드 규칙은
[Auth Credential Semantics](/ko/auth-credential-semantics)를 참조하세요.

## 권장 설정(API 키, 모든 provider)

장기간 실행되는 gateway를 운영 중이라면, 선택한 provider의 API 키로 시작하세요.
특히 Anthropic의 경우 API 키 인증이 안전한 경로입니다. OpenClaw 내부의 Anthropic 구독 스타일 인증은 레거시 setup-token 경로이며, 플랜 한도 경로가 아니라 **Extra Usage** 경로로 취급해야 합니다.

1. provider 콘솔에서 API 키를 생성합니다.
2. 이를 **gateway host**(`openclaw gateway`를 실행하는 머신)에 설정합니다.

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Gateway가 systemd/launchd에서 실행된다면, 데몬이 읽을 수 있도록 키를 `~/.openclaw/.env`에 넣는 것을 권장합니다.

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

그런 다음 데몬을 재시작하거나 Gateway 프로세스를 재시작한 뒤 다시 확인하세요.

```bash
openclaw models status
openclaw doctor
```

환경 변수를 직접 관리하고 싶지 않다면, 온보딩에서 API 키를 데몬용으로 저장할 수 있습니다: `openclaw onboard`.

`env.shellEnv`, `~/.openclaw/.env`, systemd/launchd의 환경 상속에 대한 자세한 내용은 [도움말](/ko/help)을 참조하세요.

## Anthropic: 레거시 토큰 호환성

Anthropic setup-token 인증은 여전히 OpenClaw에서
레거시/수동 경로로 사용할 수 있습니다. Anthropic의 공개 Claude Code 문서는 여전히 Claude 플랜에서 Claude Code 터미널을 직접 사용하는 방법을 다루고 있지만, Anthropic은 별도로 OpenClaw 사용자에게 **OpenClaw** Claude 로그인 경로는 서드파티 하네스 사용으로 간주되며 구독과 별도로 청구되는 **Extra Usage**가 필요하다고 안내했습니다.

가장 명확한 설정 경로는 Anthropic API 키를 사용하는 것입니다. OpenClaw에서 Anthropic 구독 스타일 경로를 유지해야 한다면, Anthropic이 이를 **Extra Usage**로 취급한다는 전제하에 레거시 setup-token 경로를 사용하세요.

수동 토큰 입력(모든 provider; `auth-profiles.json`에 기록하고 config 업데이트):

```bash
openclaw models auth paste-token --provider openrouter
```

정적 자격 증명에는 인증 프로필 참조도 지원됩니다.

- `api_key` 자격 증명은 `keyRef: { source, provider, id }`를 사용할 수 있습니다.
- `token` 자격 증명은 `tokenRef: { source, provider, id }`를 사용할 수 있습니다.
- OAuth 모드 프로필은 SecretRef 자격 증명을 지원하지 않습니다. `auth.profiles.<id>.mode`가 `"oauth"`로 설정된 경우, 해당 프로필에 대한 SecretRef 기반 `keyRef`/`tokenRef` 입력은 거부됩니다.

자동화 친화적 확인(만료/누락 시 종료 코드 `1`, 만료 임박 시 `2`):

```bash
openclaw models status --check
```

라이브 인증 프로브:

```bash
openclaw models status --probe
```

참고:

- 프로브 행은 인증 프로필, 환경 자격 증명 또는 `models.json`에서 올 수 있습니다.
- 명시적 `auth.order.<provider>`에 저장된 프로필이 빠져 있으면, 프로브는 해당 프로필을 시도하는 대신 `excluded_by_auth_order`를 보고합니다.
- 인증은 존재하지만 OpenClaw가 해당 provider에 대해 프로브 가능한 모델 후보를 확인할 수 없으면, 프로브는 `status: no_model`을 보고합니다.
- 속도 제한 cooldown은 모델 범위일 수 있습니다. 한 모델에 대해 cooldown 중인 프로필도 같은 provider의 형제 모델에는 여전히 사용할 수 있을 수 있습니다.

선택적 운영 스크립트(systemd/Termux)는 여기 문서화되어 있습니다:
[인증 모니터링 스크립트](/ko/help/scripts#auth-monitoring-scripts)

## Anthropic 참고

Anthropic `claude-cli` 백엔드는 제거되었습니다.

- OpenClaw에서 Anthropic 트래픽에는 Anthropic API 키를 사용하세요.
- Anthropic setup-token은 레거시/수동 경로로 남아 있으며, Anthropic이 OpenClaw 사용자에게 안내한 Extra Usage 과금 전제하에 사용해야 합니다.
- `openclaw doctor`는 이제 오래된 제거된 Anthropic Claude CLI 상태를 감지합니다. 저장된 자격 증명 바이트가 여전히 존재하면 doctor가 이를 Anthropic token/OAuth 프로필로 다시 변환합니다. 그렇지 않으면 doctor는 오래된 Claude CLI 구성 요소를 제거하고 API 키 또는 setup-token 복구 경로를 안내합니다.

## 모델 인증 상태 확인

```bash
openclaw models status
openclaw doctor
```

## API 키 순환 동작(gateway)

일부 provider는 API 호출이 provider 속도 제한에 걸렸을 때 대체 키로 요청을 재시도하는 것을 지원합니다.

- 우선순위 순서:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY`(단일 재정의)
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- Google providers는 추가 폴백으로 `GOOGLE_API_KEY`도 포함합니다.
- 동일한 키 목록은 사용 전에 중복 제거됩니다.
- OpenClaw는 속도 제한 오류에 대해서만 다음 키로 재시도합니다(예: `429`, `rate_limit`, `quota`, `resource exhausted`, `Too many concurrent requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai ... quota limit exceeded`).
- 속도 제한이 아닌 오류는 대체 키로 재시도하지 않습니다.
- 모든 키가 실패하면 마지막 시도의 최종 오류가 반환됩니다.

## 사용할 자격 증명 제어

### 세션별(채팅 명령어)

현재 세션에 특정 provider 자격 증명을 고정하려면 `/model <alias-or-id>@<profileId>`를 사용하세요(예시 프로필 ID: `anthropic:default`, `anthropic:work`).

간단한 선택기는 `/model`(또는 `/model list`)를 사용하고, 전체 보기에는 `/model status`를 사용하세요(후보 + 다음 인증 프로필, 그리고 구성된 경우 provider 엔드포인트 세부 정보 포함).

### 에이전트별(CLI 재정의)

에이전트에 대해 명시적인 인증 프로필 순서 재정의를 설정합니다(해당 에이전트의 `auth-profiles.json`에 저장됨).

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

특정 에이전트를 대상으로 하려면 `--agent <id>`를 사용하고, 생략하면 구성된 기본 에이전트를 사용합니다.
순서 문제를 디버깅할 때 `openclaw models status --probe`는 생략된 저장 프로필을 조용히 건너뛰는 대신 `excluded_by_auth_order`로 표시합니다.
cooldown 문제를 디버깅할 때는 속도 제한 cooldown이 provider 프로필 전체가 아니라 하나의 모델 ID에 묶여 있을 수 있다는 점을 기억하세요.

## 문제 해결

### "No credentials found"

Anthropic 프로필이 없으면 **gateway host**에 Anthropic API 키를 구성하거나 레거시 Anthropic setup-token 경로를 설정한 다음 다시 확인하세요.

```bash
openclaw models status
```

### 토큰 만료 임박/만료됨

어떤 프로필이 만료 중인지 확인하려면 `openclaw models status`를 실행하세요. 레거시 Anthropic 토큰 프로필이 누락되었거나 만료되었다면, setup-token으로 해당 설정을 새로 고치거나 Anthropic API 키로 마이그레이션하세요.

머신에 이전 빌드의 오래된 제거된 Anthropic Claude CLI 상태가 여전히 남아 있다면 다음을 실행하세요.

```bash
openclaw doctor --yes
```

저장된 자격 증명 바이트가 여전히 존재하면 doctor는 `anthropic:claude-cli`를 Anthropic token/OAuth로 다시 변환합니다. 그렇지 않으면 오래된 Claude CLI 프로필/config/model 참조를 제거하고 다음 단계 안내를 남깁니다.
