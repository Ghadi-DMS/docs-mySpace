---
read_when:
    - '`openclaw onboard`의 자세한 동작이 필요한 경우'
    - 온보딩 결과를 디버깅하거나 온보딩 클라이언트를 통합하는 경우
sidebarTitle: CLI reference
summary: CLI 설정 흐름, auth/모델 설정, 출력, 내부 동작에 대한 전체 참조
title: CLI 설정 참조
x-i18n:
    generated_at: "2026-04-06T03:12:49Z"
    model: gpt-5.4
    provider: openai
    source_hash: 92f379b34a2b48c68335dae4f759117c770f018ec51b275f4f40421c6b3abb23
    source_path: start/wizard-cli-reference.md
    workflow: 15
---

# CLI 설정 참조

이 페이지는 `openclaw onboard`의 전체 참조입니다.
짧은 가이드는 [Onboarding (CLI)](/ko/start/wizard)를 참조하세요.

## 마법사가 하는 일

로컬 모드(기본값)는 다음 과정을 안내합니다:

- 모델 및 auth 설정(OpenAI Code 구독 OAuth, Anthropic Claude CLI 또는 API 키, 그리고 MiniMax, GLM, Ollama, Moonshot, StepFun, AI Gateway 옵션)
- 워크스페이스 위치 및 bootstrap 파일
- Gateway 설정(포트, bind, auth, Tailscale)
- 채널 및 provider(Telegram, WhatsApp, Discord, Google Chat, Mattermost, Signal, BlueBubbles, 기타 번들 채널 plugin)
- 데몬 설치(LaunchAgent, systemd 사용자 유닛, 또는 Startup 폴더 폴백이 있는 네이티브 Windows Scheduled Task)
- 상태 점검
- Skills 설정

원격 모드는 이 컴퓨터가 다른 위치의 gateway에 연결되도록 구성합니다.
원격 호스트에는 아무것도 설치하거나 수정하지 않습니다.

## 로컬 흐름 세부 사항

<Steps>
  <Step title="기존 config 감지">
    - `~/.openclaw/openclaw.json`이 있으면 유지, 수정, 또는 재설정을 선택합니다.
    - 마법사를 다시 실행해도 명시적으로 재설정을 선택하지 않는 한(또는 `--reset`을 전달하지 않는 한) 아무것도 지워지지 않습니다.
    - CLI `--reset`의 기본값은 `config+creds+sessions`이며, 워크스페이스도 제거하려면 `--reset-scope full`을 사용하세요.
    - config가 유효하지 않거나 레거시 키를 포함하고 있으면, 마법사는 중단하고 계속하기 전에 `openclaw doctor`를 실행하라고 요청합니다.
    - 재설정은 `trash`를 사용하며 다음 범위를 제공합니다:
      - config만
      - config + 자격 증명 + 세션
      - 전체 재설정(워크스페이스도 제거)
  </Step>
  <Step title="모델 및 auth">
    - 전체 옵션 매트릭스는 [Auth and model options](#auth-and-model-options)에 있습니다.
  </Step>
  <Step title="워크스페이스">
    - 기본값은 `~/.openclaw/workspace`입니다(변경 가능).
    - 최초 실행 bootstrap ritual에 필요한 워크스페이스 파일을 시드합니다.
    - 워크스페이스 레이아웃: [Agent workspace](/ko/concepts/agent-workspace).
  </Step>
  <Step title="Gateway">
    - 포트, bind, auth 모드, Tailscale 노출 여부를 묻습니다.
    - 권장 사항: loopback에서도 토큰 auth를 활성화해 로컬 WS 클라이언트가 인증하도록 유지하세요.
    - 토큰 모드에서 대화형 설정은 다음을 제공합니다:
      - **일반 텍스트 토큰 생성/저장** (기본값)
      - **SecretRef 사용** (옵트인)
    - 비밀번호 모드에서도 대화형 설정은 일반 텍스트 또는 SecretRef 저장을 지원합니다.
    - 비대화형 토큰 SecretRef 경로: `--gateway-token-ref-env <ENV_VAR>`.
      - 온보딩 프로세스 환경에 비어 있지 않은 env var가 필요합니다.
      - `--gateway-token`과 함께 사용할 수 없습니다.
    - 모든 로컬 프로세스를 완전히 신뢰하는 경우에만 auth를 비활성화하세요.
    - loopback이 아닌 bind에도 여전히 auth가 필요합니다.
  </Step>
  <Step title="채널">
    - [WhatsApp](/ko/channels/whatsapp): 선택적 QR 로그인
    - [Telegram](/ko/channels/telegram): 봇 토큰
    - [Discord](/ko/channels/discord): 봇 토큰
    - [Google Chat](/ko/channels/googlechat): 서비스 계정 JSON + webhook audience
    - [Mattermost](/ko/channels/mattermost): 봇 토큰 + 기본 URL
    - [Signal](/ko/channels/signal): 선택적 `signal-cli` 설치 + 계정 config
    - [BlueBubbles](/ko/channels/bluebubbles): iMessage에 권장; 서버 URL + 비밀번호 + webhook
    - [iMessage](/ko/channels/imessage): 레거시 `imsg` CLI 경로 + DB 접근
    - DM 보안: 기본값은 페어링입니다. 첫 DM은 코드를 보내며,
      `openclaw pairing approve <channel> <code>`로 승인하거나 allowlist를 사용할 수 있습니다.
  </Step>
  <Step title="데몬 설치">
    - macOS: LaunchAgent
      - 로그인된 사용자 세션이 필요합니다. headless 환경에서는 커스텀 LaunchDaemon을 사용하세요(기본 제공되지 않음).
    - Linux 및 WSL2를 통한 Windows: systemd 사용자 유닛
      - 마법사는 로그아웃 후에도 gateway가 유지되도록 `loginctl enable-linger <user>`를 시도합니다.
      - sudo를 요청할 수 있습니다(`/var/lib/systemd/linger`에 기록). 먼저 sudo 없이 시도합니다.
    - 네이티브 Windows: 우선 Scheduled Task
      - 태스크 생성이 거부되면 OpenClaw는 사용자별 Startup 폴더 로그인 항목으로 폴백하고 gateway를 즉시 시작합니다.
      - Scheduled Task는 더 나은 supervisor 상태를 제공하므로 여전히 우선 방식입니다.
    - 런타임 선택: Node(권장; WhatsApp과 Telegram에 필요). Bun은 권장되지 않습니다.
  </Step>
  <Step title="상태 점검">
    - 필요하면 gateway를 시작하고 `openclaw health`를 실행합니다.
    - `openclaw status --deep`는 지원되는 경우 채널 프로브를 포함해 라이브 gateway 상태 프로브를 상태 출력에 추가합니다.
  </Step>
  <Step title="Skills">
    - 사용 가능한 Skills를 읽고 요구 사항을 확인합니다.
    - node manager로 npm, pnpm, 또는 bun을 선택할 수 있습니다.
    - 선택적 의존성을 설치합니다(일부는 macOS에서 Homebrew 사용).
  </Step>
  <Step title="완료">
    - iOS, Android, macOS 앱 옵션을 포함한 요약 및 다음 단계가 표시됩니다.
  </Step>
</Steps>

<Note>
GUI가 감지되지 않으면 마법사는 브라우저를 여는 대신 Control UI용 SSH 포트 포워딩 안내를 출력합니다.
Control UI 자산이 없으면 마법사는 이를 빌드하려고 시도하며, 폴백은 `pnpm ui:build`입니다(UI 의존성 자동 설치).
</Note>

## 원격 모드 세부 사항

원격 모드는 이 컴퓨터가 다른 위치의 gateway에 연결되도록 구성합니다.

<Info>
원격 모드는 원격 호스트에 아무것도 설치하거나 수정하지 않습니다.
</Info>

설정하는 항목:

- 원격 gateway URL (`ws://...`)
- 원격 gateway auth가 필요한 경우 토큰(권장)

<Note>
- gateway가 loopback 전용이면 SSH 터널링 또는 tailnet을 사용하세요.
- 탐지 힌트:
  - macOS: Bonjour (`dns-sd`)
  - Linux: Avahi (`avahi-browse`)
</Note>

## Auth and model options

<AccordionGroup>
  <Accordion title="Anthropic API 키">
    있으면 `ANTHROPIC_API_KEY`를 사용하고, 없으면 키를 입력받은 뒤 데몬에서 사용할 수 있도록 저장합니다.
  </Accordion>
  <Accordion title="OpenAI Code 구독(Codex CLI 재사용)">
    `~/.codex/auth.json`이 있으면 마법사가 이를 재사용할 수 있습니다.
    재사용된 Codex CLI 자격 증명은 계속 Codex CLI가 관리합니다. 만료되면 OpenClaw는
    먼저 해당 소스를 다시 읽고, provider가 이를 갱신할 수 있으면
    소유권을 가져오는 대신 갱신된 자격 증명을 다시 Codex 저장소에 기록합니다.
  </Accordion>
  <Accordion title="OpenAI Code 구독(OAuth)">
    브라우저 흐름; `code#state`를 붙여 넣습니다.

    모델이 설정되지 않았거나 `openai/*`인 경우 `agents.defaults.model`을 `openai-codex/gpt-5.4`로 설정합니다.

  </Accordion>
  <Accordion title="OpenAI API 키">
    있으면 `OPENAI_API_KEY`를 사용하고, 없으면 키를 입력받은 뒤 자격 증명을 auth 프로필에 저장합니다.

    모델이 설정되지 않았거나 `openai/*`, `openai-codex/*`인 경우 `agents.defaults.model`을 `openai/gpt-5.4`로 설정합니다.

  </Accordion>
  <Accordion title="xAI (Grok) API 키">
    `XAI_API_KEY`를 입력받고 xAI를 모델 provider로 구성합니다.
  </Accordion>
  <Accordion title="OpenCode">
    `OPENCODE_API_KEY`(또는 `OPENCODE_ZEN_API_KEY`)를 입력받고 Zen 또는 Go 카탈로그를 선택하게 합니다.
    설정 URL: [opencode.ai/auth](https://opencode.ai/auth).
  </Accordion>
  <Accordion title="API 키(일반)">
    키를 대신 저장합니다.
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    `AI_GATEWAY_API_KEY`를 입력받습니다.
    자세한 내용: [Vercel AI Gateway](/ko/providers/vercel-ai-gateway).
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    account ID, gateway ID, `CLOUDFLARE_AI_GATEWAY_API_KEY`를 입력받습니다.
    자세한 내용: [Cloudflare AI Gateway](/ko/providers/cloudflare-ai-gateway).
  </Accordion>
  <Accordion title="MiniMax">
    config가 자동으로 기록됩니다. 호스팅 기본값은 `MiniMax-M2.7`입니다. API 키 설정은
    `minimax/...`를 사용하고, OAuth 설정은 `minimax-portal/...`를 사용합니다.
    자세한 내용: [MiniMax](/ko/providers/minimax).
  </Accordion>
  <Accordion title="StepFun">
    중국 또는 글로벌 엔드포인트의 StepFun standard 또는 Step Plan용 config가 자동으로 기록됩니다.
    Standard에는 현재 `step-3.5-flash`가 포함되며, Step Plan에는 `step-3.5-flash-2603`도 포함됩니다.
    자세한 내용: [StepFun](/ko/providers/stepfun).
  </Accordion>
  <Accordion title="Synthetic (Anthropic 호환)">
    `SYNTHETIC_API_KEY`를 입력받습니다.
    자세한 내용: [Synthetic](/ko/providers/synthetic).
  </Accordion>
  <Accordion title="Ollama (클라우드 및 로컬 오픈 모델)">
    기본 URL(기본값 `http://127.0.0.1:11434`)을 입력받은 다음 Cloud + Local 또는 Local 모드를 제안합니다.
    사용 가능한 모델을 탐지하고 기본값을 제안합니다.
    자세한 내용: [Ollama](/ko/providers/ollama).
  </Accordion>
  <Accordion title="Moonshot 및 Kimi Coding">
    Moonshot(Kimi K2) 및 Kimi Coding config가 자동으로 기록됩니다.
    자세한 내용: [Moonshot AI (Kimi + Kimi Coding)](/ko/providers/moonshot).
  </Accordion>
  <Accordion title="커스텀 provider">
    OpenAI 호환 및 Anthropic 호환 엔드포인트와 동작합니다.

    대화형 온보딩은 다른 provider API 키 흐름과 동일한 API 키 저장 옵션을 지원합니다:
    - **지금 API 키 붙여넣기** (일반 텍스트)
    - **secret reference 사용** (env ref 또는 구성된 provider ref, 사전 검증 포함)

    비대화형 플래그:
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key` (선택 사항; `CUSTOM_API_KEY`로 폴백)
    - `--custom-provider-id` (선택 사항)
    - `--custom-compatibility <openai|anthropic>` (선택 사항; 기본값 `openai`)

  </Accordion>
  <Accordion title="건너뛰기">
    auth를 구성하지 않은 상태로 둡니다.
  </Accordion>
</AccordionGroup>

모델 동작:

- 탐지된 옵션에서 기본 모델을 선택하거나, provider와 모델을 수동으로 입력합니다.
- 온보딩이 provider auth 선택에서 시작되면 모델 선택기는
  해당 provider를 자동으로 우선 적용합니다. Volcengine과 BytePlus의 경우 이 우선순위는
  해당 coding-plan 변형(`volcengine-plan/*`,
  `byteplus-plan/*`)과도 일치합니다.
- 해당 선호 provider 필터 결과가 비어 있으면, 선택기는 모델이 없다고 표시하는 대신
  전체 카탈로그로 폴백합니다.
- 마법사는 모델 검사를 실행하고 구성된 모델을 알 수 없거나 auth가 없으면 경고합니다.

자격 증명 및 프로필 경로:

- Auth 프로필(API 키 + OAuth): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- 레거시 OAuth import: `~/.openclaw/credentials/oauth.json`

자격 증명 저장 모드:

- 기본 온보딩 동작은 API 키를 auth 프로필에 일반 텍스트 값으로 저장합니다.
- `--secret-input-mode ref`는 일반 텍스트 키 저장 대신 reference 모드를 활성화합니다.
  대화형 설정에서는 다음 중 하나를 선택할 수 있습니다:
  - 환경 변수 ref(예: `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)
  - provider 별칭 + id를 사용하는 구성된 provider ref(`file` 또는 `exec`)
- 대화형 reference 모드는 저장 전에 빠른 사전 검증을 실행합니다.
  - Env ref: 현재 온보딩 환경에서 변수 이름과 비어 있지 않은 값을 검증합니다.
  - Provider ref: provider config를 검증하고 요청된 id를 해석합니다.
  - 사전 검증이 실패하면 온보딩이 오류를 보여 주고 다시 시도할 수 있게 합니다.
- 비대화형 모드에서 `--secret-input-mode ref`는 env 기반만 지원합니다.
  - provider env var를 온보딩 프로세스 환경에 설정하세요.
  - 인라인 키 플래그(예: `--openai-api-key`)는 해당 env var가 설정되어 있어야 하며, 그렇지 않으면 온보딩이 즉시 실패합니다.
  - 커스텀 provider의 경우 비대화형 `ref` 모드는 `models.providers.<id>.apiKey`를 `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`로 저장합니다.
  - 이 커스텀 provider 경우 `--custom-api-key`는 `CUSTOM_API_KEY`가 설정되어 있어야 하며, 그렇지 않으면 온보딩이 즉시 실패합니다.
- Gateway auth 자격 증명은 대화형 설정에서 일반 텍스트와 SecretRef 선택을 지원합니다:
  - 토큰 모드: **일반 텍스트 토큰 생성/저장**(기본값) 또는 **SecretRef 사용**.
  - 비밀번호 모드: 일반 텍스트 또는 SecretRef.
- 비대화형 토큰 SecretRef 경로: `--gateway-token-ref-env <ENV_VAR>`.
- 기존 일반 텍스트 설정은 변경 없이 계속 동작합니다.

<Note>
헤드리스 및 서버 팁: 브라우저가 있는 컴퓨터에서 OAuth를 완료한 다음
해당 에이전트의 `auth-profiles.json`(예:
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, 또는 해당하는
`$OPENCLAW_STATE_DIR/...` 경로)을 gateway 호스트로 복사하세요. `credentials/oauth.json`은
레거시 import 소스일 뿐입니다.
</Note>

## 출력 및 내부 동작

`~/.openclaw/openclaw.json`의 일반적인 필드:

- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers` (MiniMax를 선택한 경우)
- `tools.profile` (로컬 온보딩은 설정되지 않은 경우 기본값으로 `"coding"`을 사용하며, 기존 명시적 값은 유지됩니다)
- `gateway.*` (mode, bind, auth, Tailscale)
- `session.dmScope` (로컬 온보딩은 설정되지 않은 경우 기본값으로 `per-channel-peer`를 사용하며, 기존 명시적 값은 유지됩니다)
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- 프롬프트 중 옵트인하면 채널 allowlist(Slack, Discord, Matrix, Microsoft Teams)도 포함됩니다(가능하면 이름을 ID로 해석)
- `skills.install.nodeManager`
  - `setup --node-manager` 플래그는 `npm`, `pnpm`, 또는 `bun`을 허용합니다.
  - 수동 config에서는 나중에 `skills.install.nodeManager: "yarn"`도 설정할 수 있습니다.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`openclaw agents add`는 `agents.list[]` 및 선택적 `bindings`를 기록합니다.

WhatsApp 자격 증명은 `~/.openclaw/credentials/whatsapp/<accountId>/` 아래에 저장됩니다.
세션은 `~/.openclaw/agents/<agentId>/sessions/` 아래에 저장됩니다.

<Note>
일부 채널은 plugin으로 제공됩니다. 설정 중 선택하면 마법사는
채널 구성을 진행하기 전에 plugin 설치(npm 또는 로컬 경로)를 묻습니다.
</Note>

Gateway 마법사 RPC:

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

클라이언트(macOS 앱 및 Control UI)는 온보딩 로직을 다시 구현하지 않고도 단계를 렌더링할 수 있습니다.

Signal 설정 동작:

- 적절한 릴리스 자산을 다운로드
- `~/.openclaw/tools/signal-cli/<version>/` 아래에 저장
- config에 `channels.signal.cliPath` 기록
- JVM 빌드는 Java 21이 필요
- 가능하면 네이티브 빌드를 사용
- Windows는 WSL2를 사용하며 WSL 내부에서 Linux signal-cli 흐름을 따름

## 관련 문서

- 온보딩 허브: [Onboarding (CLI)](/ko/start/wizard)
- 자동화 및 스크립트: [CLI Automation](/ko/start/wizard-cli-automation)
- 명령 참조: [`openclaw onboard`](/cli/onboard)
