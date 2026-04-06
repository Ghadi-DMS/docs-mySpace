---
read_when:
    - 특정 온보딩 단계나 플래그를 찾아볼 때
    - 비대화형 모드로 온보딩을 자동화할 때
    - 온보딩 동작을 디버깅할 때
sidebarTitle: Onboarding Reference
summary: 'CLI 온보딩 전체 참조: 모든 단계, 플래그, 구성 필드'
title: 온보딩 참조
x-i18n:
    generated_at: "2026-04-06T03:12:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: e02a4da4a39ba335199095723f5d3b423671eb12efc2d9e4f9e48c1e8ee18419
    source_path: reference/wizard.md
    workflow: 15
---

# 온보딩 참조

이 문서는 `openclaw onboard`의 전체 참조입니다.
상위 수준 개요는 [Onboarding (CLI)](/ko/start/wizard)를 참고하세요.

## 흐름 세부 정보(로컬 모드)

<Steps>
  <Step title="기존 구성 감지">
    - `~/.openclaw/openclaw.json`이 있으면 **Keep / Modify / Reset** 중에서 선택합니다.
    - 온보딩을 다시 실행해도 명시적으로 **Reset**을 선택하지 않는 한
      (또는 `--reset`을 전달하지 않는 한) 아무것도 지워지지 않습니다.
    - CLI `--reset`의 기본값은 `config+creds+sessions`이며, 작업공간까지 제거하려면 `--reset-scope full`
      을 사용하세요.
    - 구성이 유효하지 않거나 레거시 키를 포함하고 있으면, wizard는 중지되고
      계속하기 전에 `openclaw doctor`를 실행하라고 안내합니다.
    - Reset은 `trash`를 사용하며(`rm`은 사용하지 않음) 다음 범위를 제공합니다:
      - 구성만
      - 구성 + 자격 증명 + 세션
      - 전체 초기화(작업공간도 제거)
  </Step>
  <Step title="Model/Auth">
    - **Anthropic API key**: `ANTHROPIC_API_KEY`가 있으면 이를 사용하고, 없으면 키를 묻고, 이후 daemon에서 사용할 수 있도록 저장합니다.
    - **Anthropic API key**: 온보딩/구성에서 권장되는 Anthropic assistant 선택지입니다.
    - **Anthropic setup-token (legacy/manual)**: 온보딩/구성에서 다시 사용할 수 있지만, Anthropic은 OpenClaw 사용자에게 OpenClaw Claude-login 경로가 서드파티 하네스 사용으로 간주되므로 Claude 계정에 **Extra Usage**가 필요하다고 안내했습니다.
    - **OpenAI Code (Codex) subscription (Codex CLI)**: `~/.codex/auth.json`이 있으면 온보딩에서 이를 재사용할 수 있습니다. 재사용된 Codex CLI 자격 증명은 계속 Codex CLI가 관리합니다. 만료되면 OpenClaw는 먼저 해당 소스를 다시 읽고, 공급자가 이를 갱신할 수 있는 경우 자격 증명의 소유권을 가져오는 대신 갱신된 자격 증명을 다시 Codex 저장소에 기록합니다.
    - **OpenAI Code (Codex) subscription (OAuth)**: 브라우저 흐름이며 `code#state`를 붙여넣습니다.
      - 모델이 설정되지 않았거나 `openai/*`인 경우 `agents.defaults.model`을 `openai-codex/gpt-5.4`로 설정합니다.
    - **OpenAI API key**: `OPENAI_API_KEY`가 있으면 이를 사용하고, 없으면 키를 묻고, 인증 프로필에 저장합니다.
      - 모델이 설정되지 않았거나 `openai/*` 또는 `openai-codex/*`이면 `agents.defaults.model`을 `openai/gpt-5.4`로 설정합니다.
    - **xAI (Grok) API key**: `XAI_API_KEY`를 묻고 xAI를 모델 공급자로 구성합니다.
    - **OpenCode**: `OPENCODE_API_KEY`(또는 `OPENCODE_ZEN_API_KEY`, 발급 위치: https://opencode.ai/auth)를 묻고 Zen 또는 Go 카탈로그를 선택하게 합니다.
    - **Ollama**: Ollama base URL을 묻고, **Cloud + Local** 또는 **Local** 모드를 제공하며, 사용 가능한 모델을 탐지하고, 필요하면 선택한 로컬 모델을 자동으로 pull합니다.
    - 자세한 내용: [Ollama](/ko/providers/ollama)
    - **API key**: 키를 대신 저장합니다.
    - **Vercel AI Gateway (multi-model proxy)**: `AI_GATEWAY_API_KEY`를 묻습니다.
    - 자세한 내용: [Vercel AI Gateway](/ko/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: Account ID, Gateway ID, `CLOUDFLARE_AI_GATEWAY_API_KEY`를 묻습니다.
    - 자세한 내용: [Cloudflare AI Gateway](/ko/providers/cloudflare-ai-gateway)
    - **MiniMax**: 구성이 자동으로 기록되며, 호스팅 기본값은 `MiniMax-M2.7`입니다.
      API 키 설정은 `minimax/...`를 사용하고, OAuth 설정은
      `minimax-portal/...`를 사용합니다.
    - 자세한 내용: [MiniMax](/ko/providers/minimax)
    - **StepFun**: 중국 또는 글로벌 엔드포인트의 StepFun standard 또는 Step Plan용 구성이 자동으로 기록됩니다.
    - Standard는 현재 `step-3.5-flash`를 포함하며, Step Plan은 `step-3.5-flash-2603`도 포함합니다.
    - 자세한 내용: [StepFun](/ko/providers/stepfun)
    - **Synthetic (Anthropic-compatible)**: `SYNTHETIC_API_KEY`를 묻습니다.
    - 자세한 내용: [Synthetic](/ko/providers/synthetic)
    - **Moonshot (Kimi K2)**: 구성이 자동으로 기록됩니다.
    - **Kimi Coding**: 구성이 자동으로 기록됩니다.
    - 자세한 내용: [Moonshot AI (Kimi + Kimi Coding)](/ko/providers/moonshot)
    - **Skip**: 아직 인증을 구성하지 않습니다.
    - 감지된 옵션에서 기본 모델을 선택합니다(또는 `provider/model`을 직접 입력). 최상의 품질과 더 낮은 프롬프트 인젝션 위험을 위해 공급자 스택에서 사용할 수 있는 가장 강력한 최신 세대 모델을 선택하세요.
    - 온보딩은 모델 검사를 실행하고 구성된 모델을 알 수 없거나 인증이 없으면 경고합니다.
    - API 키 저장 모드의 기본값은 평문 인증 프로필 값입니다. 대신 env 기반 ref를 저장하려면 `--secret-input-mode ref`를 사용하세요(예: `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`).
    - 인증 프로필은 `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`에 있습니다(API 키 + OAuth). `~/.openclaw/credentials/oauth.json`은 레거시 가져오기 전용입니다.
    - 자세한 내용: [/concepts/oauth](/ko/concepts/oauth)
    <Note>
    헤드리스/서버 팁: 브라우저가 있는 머신에서 OAuth를 완료한 다음,
    해당 에이전트의 `auth-profiles.json`(예:
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` 또는 해당하는
    `$OPENCLAW_STATE_DIR/...` 경로)을 gateway host로 복사하세요. `credentials/oauth.json`
    은 레거시 가져오기 소스일 뿐입니다.
    </Note>
  </Step>
  <Step title="작업공간">
    - 기본값은 `~/.openclaw/workspace`입니다(구성 가능).
    - 에이전트 bootstrap ritual에 필요한 작업공간 파일을 초기 배치합니다.
    - 전체 작업공간 레이아웃 + 백업 가이드: [Agent workspace](/ko/concepts/agent-workspace)
  </Step>
  <Step title="Gateway">
    - 포트, 바인드, 인증 모드, Tailscale 노출.
    - 인증 권장 사항: loopback에서도 로컬 WS 클라이언트가 인증해야 하도록 **Token**을 유지하세요.
    - token 모드에서 대화형 설정은 다음을 제공합니다:
      - **평문 토큰 생성/저장**(기본값)
      - **SecretRef 사용**(선택 사항)
      - Quickstart는 온보딩 probe/dashboard bootstrap을 위해 `env`, `file`, `exec` 공급자 전반의 기존 `gateway.auth.token` SecretRef를 재사용합니다.
      - 해당 SecretRef가 구성되어 있지만 해석할 수 없으면, 온보딩은 런타임 인증을 조용히 약화시키는 대신 명확한 수정 메시지와 함께 조기에 실패합니다.
    - password 모드에서도 대화형 설정은 평문 또는 SecretRef 저장을 지원합니다.
    - 비대화형 token SecretRef 경로: `--gateway-token-ref-env <ENV_VAR>`.
      - 온보딩 프로세스 환경에 비어 있지 않은 env var가 있어야 합니다.
      - `--gateway-token`과 함께 사용할 수 없습니다.
    - 모든 로컬 프로세스를 완전히 신뢰하는 경우에만 인증을 비활성화하세요.
    - non‑loopback 바인드에는 여전히 인증이 필요합니다.
  </Step>
  <Step title="채널">
    - [WhatsApp](/ko/channels/whatsapp): 선택적 QR 로그인
    - [Telegram](/ko/channels/telegram): 봇 토큰
    - [Discord](/ko/channels/discord): 봇 토큰
    - [Google Chat](/ko/channels/googlechat): 서비스 계정 JSON + webhook audience
    - [Mattermost](/ko/channels/mattermost) (plugin): 봇 토큰 + base URL
    - [Signal](/ko/channels/signal): 선택적 `signal-cli` 설치 + 계정 구성
    - [BlueBubbles](/ko/channels/bluebubbles): **iMessage에 권장**; 서버 URL + 비밀번호 + webhook
    - [iMessage](/ko/channels/imessage): 레거시 `imsg` CLI 경로 + DB 접근
    - DM 보안: 기본값은 페어링입니다. 첫 번째 DM은 코드를 보내며, `openclaw pairing approve <channel> <code>`로 승인하거나 허용 목록을 사용할 수 있습니다.
  </Step>
  <Step title="웹 검색">
    - Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG, Tavily 같은 지원 공급자를 선택하거나 건너뛸 수 있습니다.
    - API 기반 공급자는 빠른 설정을 위해 env var 또는 기존 구성을 사용할 수 있고, 키가 없는 공급자는 해당 공급자별 사전 요구 사항을 사용합니다.
    - `--skip-search`로 건너뜁니다.
    - 나중에 구성: `openclaw configure --section web`
  </Step>
  <Step title="Daemon 설치">
    - macOS: LaunchAgent
      - 로그인된 사용자 세션이 필요합니다. 헤드리스 환경에서는 사용자 지정 LaunchDaemon을 사용하세요(기본 제공되지 않음).
    - Linux(및 WSL2를 통한 Windows): systemd 사용자 유닛
      - 온보딩은 로그아웃 후에도 Gateway가 계속 실행되도록 `loginctl enable-linger <user>` 활성화를 시도합니다.
      - sudo를 요청할 수 있습니다(`/var/lib/systemd/linger`에 기록). 먼저 sudo 없이 시도합니다.
    - **런타임 선택:** Node(권장; WhatsApp/Telegram에 필요). Bun은 **권장되지 않습니다**.
    - token auth에 토큰이 필요하고 `gateway.auth.token`이 SecretRef로 관리되는 경우, daemon 설치는 이를 검증하지만 해석된 평문 토큰 값을 supervisor 서비스 환경 메타데이터에 지속 저장하지 않습니다.
    - token auth에 토큰이 필요하고 구성된 token SecretRef가 해석되지 않으면, daemon 설치는 실행 가능한 안내와 함께 차단됩니다.
    - `gateway.auth.token`과 `gateway.auth.password`가 모두 구성되어 있고 `gateway.auth.mode`가 설정되지 않은 경우, 모드를 명시적으로 설정할 때까지 daemon 설치가 차단됩니다.
  </Step>
  <Step title="상태 점검">
    - 필요하면 Gateway를 시작하고 `openclaw health`를 실행합니다.
    - 팁: `openclaw status --deep`는 상태 출력에 live gateway 상태 probe를 추가하며, 지원되는 경우 채널 probe도 포함합니다(접근 가능한 gateway 필요).
  </Step>
  <Step title="Skills(권장)">
    - 사용 가능한 Skills를 읽고 요구 사항을 확인합니다.
    - 노드 관리자를 선택하게 합니다: **npm / pnpm** (bun은 권장되지 않음).
    - 선택적 의존성을 설치합니다(일부는 macOS에서 Homebrew 사용).
  </Step>
  <Step title="완료">
    - 추가 기능을 위한 iOS/Android/macOS 앱을 포함한 요약 + 다음 단계
  </Step>
</Steps>

<Note>
GUI가 감지되지 않으면, 온보딩은 브라우저를 여는 대신 Control UI용 SSH 포트 포워딩 지침을 출력합니다.
Control UI 자산이 없으면, 온보딩은 이를 빌드하려고 시도합니다. 폴백은 `pnpm ui:build`입니다(UI 의존성 자동 설치).
</Note>

## 비대화형 모드

온보딩을 자동화하거나 스크립트화하려면 `--non-interactive`를 사용하세요.

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

기계 판독 가능한 요약이 필요하면 `--json`을 추가하세요.

비대화형 모드에서의 Gateway token SecretRef:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token`과 `--gateway-token-ref-env`는 함께 사용할 수 없습니다.

<Note>
`--json`은 **비대화형 모드를 의미하지 않습니다**. 스크립트에서는 `--non-interactive`(및 `--workspace`)를 사용하세요.
</Note>

공급자별 명령 예시는 [CLI Automation](/ko/start/wizard-cli-automation#provider-specific-examples)에 있습니다.
플래그 의미와 단계 순서는 이 참조 페이지를 사용하세요.

### 에이전트 추가(비대화형)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.4 \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

## Gateway wizard RPC

Gateway는 RPC(`wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status`)를 통해 온보딩 흐름을 노출합니다.
클라이언트(macOS 앱, Control UI)는 온보딩 로직을 다시 구현하지 않고도 단계를 렌더링할 수 있습니다.

## Signal 설정(signal-cli)

온보딩은 GitHub 릴리스에서 `signal-cli`를 설치할 수 있습니다.

- 적절한 릴리스 자산을 다운로드합니다.
- 이를 `~/.openclaw/tools/signal-cli/<version>/` 아래에 저장합니다.
- 구성에 `channels.signal.cliPath`를 기록합니다.

참고:

- JVM 빌드에는 **Java 21**이 필요합니다.
- 가능한 경우 네이티브 빌드를 사용합니다.
- Windows는 WSL2를 사용하며, signal-cli 설치는 WSL 내부에서 Linux 흐름을 따릅니다.

## wizard가 기록하는 내용

`~/.openclaw/openclaw.json`에 기록되는 일반적인 필드:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers`(Minimax를 선택한 경우)
- `tools.profile`(로컬 온보딩은 설정되지 않은 경우 기본값으로 `"coding"`을 사용하며, 기존의 명시적 값은 유지됨)
- `gateway.*`(mode, bind, auth, tailscale)
- `session.dmScope`(동작 세부 사항: [CLI Setup Reference](/ko/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- 프롬프트 중에 선택하면 채널 허용 목록(Slack/Discord/Matrix/Microsoft Teams)(가능한 경우 이름은 ID로 해석됨)
- `skills.install.nodeManager`
  - `setup --node-manager`는 `npm`, `pnpm`, `bun`을 허용합니다.
  - 수동 구성은 `skills.install.nodeManager`를 직접 설정하여 여전히 `yarn`을 사용할 수 있습니다.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`openclaw agents add`는 `agents.list[]`와 선택적 `bindings`를 기록합니다.

WhatsApp 자격 증명은 `~/.openclaw/credentials/whatsapp/<accountId>/` 아래에 저장됩니다.
세션은 `~/.openclaw/agents/<agentId>/sessions/` 아래에 저장됩니다.

일부 채널은 plugin으로 제공됩니다. 설정 중에 이를 선택하면,
구성하기 전에 온보딩이 이를 설치하라고 안내합니다(npm 또는 로컬 경로).

## 관련 문서

- 온보딩 개요: [Onboarding (CLI)](/ko/start/wizard)
- macOS 앱 온보딩: [Onboarding](/ko/start/onboarding)
- 구성 참조: [Gateway configuration](/ko/gateway/configuration)
- 공급자: [WhatsApp](/ko/channels/whatsapp), [Telegram](/ko/channels/telegram), [Discord](/ko/channels/discord), [Google Chat](/ko/channels/googlechat), [Signal](/ko/channels/signal), [BlueBubbles](/ko/channels/bluebubbles) (iMessage), [iMessage](/ko/channels/imessage) (legacy)
- Skills: [Skills](/ko/tools/skills), [Skills config](/ko/tools/skills-config)
