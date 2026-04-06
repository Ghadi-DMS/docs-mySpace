---
read_when:
    - 일반적인 설정, 설치, 온보딩 또는 런타임 지원 질문에 답하는 경우
    - 더 깊은 디버깅에 앞서 사용자가 보고한 문제를 분류하는 경우
summary: OpenClaw 설정, 구성, 사용에 관한 자주 묻는 질문
title: FAQ
x-i18n:
    generated_at: "2026-04-06T03:13:53Z"
    model: gpt-5.4
    provider: openai
    source_hash: 4d6d09621c6033d580cbcf1ff46f81587d69404d6f64c8d8fd8c3f09185bb920
    source_path: help/faq.md
    workflow: 15
---

# FAQ

실제 환경 설정(local dev, VPS, multi-agent, OAuth/API 키, 모델 페일오버)을 위한 빠른 답변과 더 깊은 문제 해결을 제공합니다. 런타임 진단은 [문제 해결](/ko/gateway/troubleshooting)을 참조하세요. 전체 구성 참조는 [구성](/ko/gateway/configuration)을 참조하세요.

## 문제가 생겼을 때 처음 60초

1. **빠른 상태 확인(첫 점검)**

   ```bash
   openclaw status
   ```

   빠른 로컬 요약: OS + 업데이트, gateway/service 연결 가능 여부, agents/sessions, provider 구성 + 런타임 문제(gateway에 연결 가능한 경우).

2. **공유 가능한 보고서(안전하게 공유 가능)**

   ```bash
   openclaw status --all
   ```

   읽기 전용 진단과 로그 tail(tokens는 마스킹됨)을 제공합니다.

3. **데몬 + 포트 상태**

   ```bash
   openclaw gateway status
   ```

   supervisor 런타임과 RPC 연결 가능 여부, 프로브 대상 URL, 서비스가 사용했을 가능성이 높은 구성을 보여줍니다.

4. **심층 프로브**

   ```bash
   openclaw status --deep
   ```

   live gateway health probe를 실행하며, 지원되는 경우 채널 프로브도 포함합니다
   (연결 가능한 gateway 필요). [상태 점검](/ko/gateway/health)을 참조하세요.

5. **최신 로그 tail**

   ```bash
   openclaw logs --follow
   ```

   RPC가 내려가 있다면 다음으로 대체하세요.

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

   파일 로그는 서비스 로그와 별개입니다. [로깅](/ko/logging) 및 [문제 해결](/ko/gateway/troubleshooting)을 참조하세요.

6. **doctor 실행(복구)**

   ```bash
   openclaw doctor
   ```

   구성/상태를 복구하거나 마이그레이션하고 상태 점검을 실행합니다. [Doctor](/ko/gateway/doctor)를 참조하세요.

7. **Gateway 스냅샷**

   ```bash
   openclaw health --json
   openclaw health --verbose   # 오류 시 대상 URL + config 경로를 표시
   ```

   실행 중인 gateway에 전체 스냅샷을 요청합니다(WS 전용). [상태 점검](/ko/gateway/health)을 참조하세요.

## 빠른 시작 및 첫 실행 설정

<AccordionGroup>
  <Accordion title="막혔습니다. 가장 빠르게 해결하는 방법은 무엇인가요?">
    **내 컴퓨터를 볼 수 있는** 로컬 AI 에이전트를 사용하세요. 대부분의 "막혔다" 사례는 원격 도우미가 확인할 수 없는 **로컬 구성 또는 환경 문제**이므로 Discord에서 묻는 것보다 훨씬 효과적입니다.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    이런 도구는 저장소를 읽고, 명령을 실행하고, 로그를 검사하며, 머신 수준 설정(PATH, services, permissions, auth files)을 고치는 데 도움을 줄 수 있습니다. 해킹 가능한(git) 설치로 **전체 소스 체크아웃**을 제공하세요.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    이렇게 하면 OpenClaw가 **git 체크아웃으로부터** 설치되므로, 에이전트가 코드 + 문서를 읽고 현재 실행 중인 정확한 버전을 추론할 수 있습니다. 나중에 `--install-method git` 없이 설치 프로그램을 다시 실행하면 언제든 stable로 되돌릴 수 있습니다.

    팁: 에이전트에게 수정 작업을 **계획하고 감독**(단계별)하게 한 뒤, 꼭 필요한 명령만 실행하게 하세요. 이렇게 하면 변경 범위가 작고 감사하기 쉬워집니다.

    실제 버그나 수정 사항을 발견했다면 GitHub issue를 등록하거나 PR을 보내주세요.
    [https://github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
    [https://github.com/openclaw/openclaw/pulls](https://github.com/openclaw/openclaw/pulls)

    도움을 요청할 때는 먼저 다음 명령부터 실행하세요(출력을 공유하세요).

    ```bash
    openclaw status
    openclaw models status
    openclaw doctor
    ```

    각 명령의 역할:

    - `openclaw status`: gateway/agent 상태 + 기본 구성의 빠른 스냅샷
    - `openclaw models status`: provider 인증 + 모델 가용성 점검
    - `openclaw doctor`: 일반적인 구성/상태 문제 검증 및 복구

    그 밖에 유용한 CLI 점검: `openclaw status --all`, `openclaw logs --follow`,
    `openclaw gateway status`, `openclaw health --verbose`.

    빠른 디버그 루프: [문제가 생겼을 때 처음 60초](#문제가-생겼을-때-처음-60초).
    설치 문서: [설치](/ko/install), [설치 프로그램 플래그](/ko/install/installer), [업데이트](/ko/install/updating).

  </Accordion>

  <Accordion title="Heartbeat가 계속 건너뛰어집니다. skip 이유는 무엇을 뜻하나요?">
    일반적인 heartbeat skip 이유:

    - `quiet-hours`: 구성된 active-hours 창 바깥
    - `empty-heartbeat-file`: `HEARTBEAT.md`는 존재하지만 비어 있거나 헤더만 있는 스캐폴딩만 포함
    - `no-tasks-due`: `HEARTBEAT.md` task 모드가 활성화되어 있지만 어떤 작업 간격도 아직 도래하지 않음
    - `alerts-disabled`: heartbeat 가시성이 모두 비활성화됨(`showOk`, `showAlerts`, `useIndicator`가 모두 꺼짐)

    task 모드에서는 실제 heartbeat 실행이 완료된 후에만 due 타임스탬프가 갱신됩니다.
    건너뛴 실행은 작업 완료로 표시되지 않습니다.

    문서: [Heartbeat](/ko/gateway/heartbeat), [자동화 및 작업](/ko/automation).

  </Accordion>

  <Accordion title="OpenClaw를 설치하고 설정하는 권장 방법">
    이 저장소에서는 소스에서 실행하고 온보딩을 사용하는 방식을 권장합니다.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    마법사는 UI 자산도 자동으로 빌드할 수 있습니다. 온보딩 후에는 일반적으로 **18789** 포트에서 Gateway를 실행합니다.

    소스에서 실행(기여자/dev):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build # 첫 실행 시 UI deps를 자동 설치
    openclaw onboard
    ```

    아직 전역 설치가 없다면 `pnpm openclaw onboard`로 실행하세요.

  </Accordion>

  <Accordion title="온보딩 후 대시보드는 어떻게 열나요?">
    마법사는 온보딩 직후 깨끗한(토큰이 포함되지 않은) 대시보드 URL로 브라우저를 열고, 요약에도 링크를 출력합니다. 그 탭을 계속 열어 두세요. 자동으로 열리지 않았다면 같은 머신에서 출력된 URL을 복사해 붙여 넣으세요.
  </Accordion>

  <Accordion title="localhost와 원격 환경에서 대시보드 인증은 어떻게 하나요?">
    **Localhost(같은 머신):**

    - `http://127.0.0.1:18789/`를 엽니다.
    - shared-secret 인증을 요구하면 구성된 token 또는 password를 Control UI 설정에 붙여 넣으세요.
    - token 소스: `gateway.auth.token`(또는 `OPENCLAW_GATEWAY_TOKEN`)
    - password 소스: `gateway.auth.password`(또는 `OPENCLAW_GATEWAY_PASSWORD`)
    - 아직 shared secret이 구성되지 않았다면 `openclaw doctor --generate-gateway-token`으로 token을 생성하세요.

    **localhost가 아닌 경우:**

    - **Tailscale Serve**(권장): bind를 loopback으로 유지하고 `openclaw gateway --tailscale serve`를 실행한 뒤 `https://<magicdns>/`를 엽니다. `gateway.auth.allowTailscale`이 `true`이면 identity headers가 Control UI/WebSocket 인증을 충족합니다(shared secret을 붙여 넣을 필요 없음, trusted gateway host 가정). HTTP APIs는 private-ingress `none` 또는 trusted-proxy HTTP auth를 의도적으로 사용하지 않는 한 여전히 shared-secret 인증이 필요합니다.
      같은 클라이언트에서 잘못된 Serve 인증 시도가 동시에 발생하면 failed-auth limiter가 기록되기 전에 직렬화되므로, 두 번째 잘못된 재시도에서는 이미 `retry later`가 표시될 수 있습니다.
    - **Tailnet bind**: `openclaw gateway --bind tailnet --token "<token>"`을 실행하거나 password 인증을 구성한 뒤, `http://<tailscale-ip>:18789/`를 열고 대시보드 설정에 일치하는 shared secret을 붙여 넣으세요.
    - **Identity-aware reverse proxy**: Gateway를 non-loopback trusted proxy 뒤에 두고 `gateway.auth.mode: "trusted-proxy"`를 구성한 다음 proxy URL을 엽니다.
    - **SSH tunnel**: `ssh -N -L 18789:127.0.0.1:18789 user@host` 후 `http://127.0.0.1:18789/`를 엽니다. 터널 위에서도 shared-secret 인증은 그대로 적용되므로, 요청 시 구성된 token 또는 password를 붙여 넣으세요.

    bind 모드와 인증 상세는 [대시보드](/web/dashboard) 및 [웹 표면](/web)을 참조하세요.

  </Accordion>

  <Accordion title="채팅 승인용 exec approval 구성은 왜 두 가지가 있나요?">
    서로 다른 계층을 제어합니다.

    - `approvals.exec`: 승인 프롬프트를 채팅 대상에 전달
    - `channels.<channel>.execApprovals`: 해당 채널이 exec 승인에 대한 네이티브 승인 클라이언트처럼 동작하도록 설정

    호스트 exec 정책은 여전히 실제 승인 게이트입니다. 채팅 구성은 승인 프롬프트가 어디에 나타나고 사람들이 어떻게 답할 수 있는지만 제어합니다.

    대부분의 설정에서는 둘 다 **필요하지 않습니다**.

    - 채팅이 이미 명령과 응답을 지원하면 같은 채팅의 `/approve`가 공용 경로로 동작합니다.
    - 지원되는 네이티브 채널이 approver를 안전하게 추론할 수 있으면, OpenClaw는 `channels.<channel>.execApprovals.enabled`가 unset 또는 `"auto"`일 때 DM 우선 네이티브 승인을 자동 활성화합니다.
    - 네이티브 승인 카드/버튼을 사용할 수 있을 때는 그 네이티브 UI가 기본 경로입니다. 에이전트는 tool 결과가 채팅 승인을 사용할 수 없다고 말하거나 수동 승인만 가능한 경우에만 수동 `/approve` 명령을 포함해야 합니다.
    - 프롬프트를 다른 채팅이나 명시적 ops room에도 전달해야 할 때만 `approvals.exec`를 사용하세요.
    - 승인 프롬프트를 원래 방/주제로 다시 게시하고 싶을 때만 `channels.<channel>.execApprovals.target: "channel"` 또는 `"both"`를 사용하세요.
    - plugin 승인은 또 별개입니다. 기본값은 같은 채팅의 `/approve`이며, 선택적 `approvals.plugin` 전달을 사용할 수 있고, 일부 네이티브 채널만 그 위에 plugin-approval-native 처리를 유지합니다.

    짧게 말하면: 전달은 라우팅용이고, 네이티브 클라이언트 구성은 더 풍부한 채널별 UX용입니다.
    [Exec 승인](/ko/tools/exec-approvals)을 참조하세요.

  </Accordion>

  <Accordion title="어떤 런타임이 필요한가요?">
    Node **>= 22**가 필요합니다. `pnpm`을 권장합니다. Gateway에는 Bun을 **권장하지 않습니다**.
  </Accordion>

  <Accordion title="Raspberry Pi에서도 실행되나요?">
    예. Gateway는 가볍습니다. 문서에는 개인용으로 **512MB-1GB RAM**, **1 코어**, 약 **500MB** 디스크면 충분하다고 되어 있으며, **Raspberry Pi 4에서도 실행 가능**하다고 안내합니다.

    여유 공간(로그, 미디어, 다른 서비스)이 더 필요하다면 **2GB를 권장**하지만, 엄격한 최소 요구사항은 아닙니다.

    팁: 작은 Pi/VPS가 Gateway를 호스팅하고, 노트북/휴대폰에서 **nodes**를 페어링해
    local screen/camera/canvas 또는 명령 실행을 사용할 수 있습니다. [노드](/ko/nodes)를 참조하세요.

  </Accordion>

  <Accordion title="Raspberry Pi 설치 팁이 있나요?">
    짧게 말하면: 동작은 하지만 거친 부분이 있을 수 있습니다.

    - **64-bit** OS를 사용하고 Node >= 22를 유지하세요.
    - 로그를 보고 빠르게 업데이트할 수 있도록 **해킹 가능한(git) 설치**를 권장합니다.
    - 채널/Skills 없이 시작한 뒤 하나씩 추가하세요.
    - 이상한 바이너리 문제가 생기면 보통 **ARM 호환성** 문제입니다.

    문서: [Linux](/ko/platforms/linux), [설치](/ko/install).

  </Accordion>

  <Accordion title="wake up my friend에서 멈추거나 온보딩이 진행되지 않습니다. 어떻게 해야 하나요?">
    그 화면은 Gateway에 연결 가능하고 인증되었는지에 따라 달라집니다. TUI도 첫 hatch 시
    자동으로 "Wake up, my friend!"를 보냅니다. 이 줄만 보이고 **응답이 전혀 없으며**
    token이 0에 머물면, 에이전트가 전혀 실행되지 않은 것입니다.

    1. Gateway를 재시작하세요.

    ```bash
    openclaw gateway restart
    ```

    2. 상태 + 인증을 확인하세요.

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. 여전히 멈춘다면 다음을 실행하세요.

    ```bash
    openclaw doctor
    ```

    Gateway가 원격이라면 tunnel/Tailscale 연결이 살아 있는지, UI가 올바른 Gateway를 가리키는지 확인하세요. [원격 액세스](/ko/gateway/remote)를 참조하세요.

  </Accordion>

  <Accordion title="설정을 새 머신(Mac mini)으로 옮기면서 온보딩을 다시 하지 않아도 되나요?">
    예. **state 디렉터리**와 **workspace**를 복사한 뒤 Doctor를 한 번 실행하세요. 이렇게 하면 **두 위치를 모두** 복사하는 한 봇을 "완전히 동일하게"(memory, session history, auth, channel state) 유지할 수 있습니다.

    1. 새 머신에 OpenClaw를 설치합니다.
    2. 이전 머신의 `$OPENCLAW_STATE_DIR`(기본값: `~/.openclaw`)을 복사합니다.
    3. workspace(기본값: `~/.openclaw/workspace`)를 복사합니다.
    4. `openclaw doctor`를 실행하고 Gateway 서비스를 재시작합니다.

    그러면 config, auth profiles, WhatsApp creds, sessions, memory가 유지됩니다. remote 모드라면 gateway host가 session store와 workspace를 소유한다는 점을 기억하세요.

    **중요:** workspace만 GitHub에 커밋/푸시하면 **memory + bootstrap files**는 백업되지만 **session history나 auth는 백업되지 않습니다**. 그런 항목은 `~/.openclaw/` 아래(예: `~/.openclaw/agents/<agentId>/sessions/`)에 있습니다.

    관련 문서: [마이그레이션](/ko/install/migrating), [디스크상의 위치](#디스크상의-위치),
    [에이전트 workspace](/ko/concepts/agent-workspace), [Doctor](/ko/gateway/doctor),
    [원격 모드](/ko/gateway/remote).

  </Accordion>

  <Accordion title="최신 버전의 변경 사항은 어디에서 보나요?">
    GitHub changelog를 확인하세요:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    최신 항목이 맨 위에 있습니다. 맨 위 섹션이 **Unreleased**로 표시되어 있으면, 그 다음 날짜가 있는
    섹션이 최근 출시 버전입니다. 항목은 **Highlights**, **Changes**, **Fixes**
    (필요한 경우 docs/other 섹션 추가)로 묶여 있습니다.

  </Accordion>

  <Accordion title="docs.openclaw.ai에 접근할 수 없습니다(SSL 오류)">
    일부 Comcast/Xfinity 연결에서는 Xfinity Advanced Security가 `docs.openclaw.ai`를 잘못 차단합니다.
    이를 비활성화하거나 `docs.openclaw.ai`를 allowlist에 추가한 후 다시 시도하세요.
    다음 링크에서 차단 해제 요청을 도와주세요: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    그래도 사이트에 접근할 수 없다면 문서는 GitHub에도 미러링되어 있습니다:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="stable과 beta의 차이는 무엇인가요?">
    **Stable**과 **beta**는 별도의 코드 라인이 아니라 **npm dist-tags**입니다.

    - `latest` = stable
    - `beta` = 테스트용 초기 빌드

    보통 stable 릴리스는 먼저 **beta**에 올라간 뒤, 별도의 승격 단계에서 같은 버전이 `latest`로 이동합니다.
    필요하면 유지보수자가 바로 `latest`로 게시할 수도 있습니다. 그래서 승격 후에는 beta와 stable이 **같은 버전**을 가리킬 수 있습니다.

    변경 사항 확인:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    설치 원라이너와 beta와 dev의 차이는 아래 아코디언을 참조하세요.

  </Accordion>

  <Accordion title="beta 버전은 어떻게 설치하고, beta와 dev의 차이는 무엇인가요?">
    **Beta**는 npm dist-tag `beta`입니다(승격 후 `latest`와 같을 수 있음).
    **Dev**는 `main`의 이동하는 헤드(git)이며, 게시될 때는 npm dist-tag `dev`를 사용합니다.

    원라이너(macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Windows 설치 프로그램(PowerShell):
    [https://openclaw.ai/install.ps1](https://openclaw.ai/install.ps1)

    자세한 내용: [개발 채널](/ko/install/development-channels) 및 [설치 프로그램 플래그](/ko/install/installer).

  </Accordion>

  <Accordion title="최신 버전을 써 보려면 어떻게 하나요?">
    두 가지 방법이 있습니다.

    1. **Dev 채널(git 체크아웃):**

    ```bash
    openclaw update --channel dev
    ```

    이 명령은 `main` 브랜치로 전환하고 소스에서 업데이트합니다.

    2. **해킹 가능한 설치(설치 사이트에서):**

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    이렇게 하면 수정 가능한 로컬 저장소를 받게 되고, 이후 git로 업데이트할 수 있습니다.

    직접 깔끔하게 clone하고 싶다면 다음을 사용하세요.

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    문서: [업데이트](/cli/update), [개발 채널](/ko/install/development-channels),
    [설치](/ko/install).

  </Accordion>

  <Accordion title="설치와 온보딩은 보통 얼마나 걸리나요?">
    대략적인 가이드:

    - **설치:** 2-5분
    - **온보딩:** 구성하는 채널/모델 수에 따라 5-15분

    멈춘 경우 [설치 프로그램이 멈췄나요?](#빠른-시작-및-첫-실행-설정)와
    [막혔습니다](#빠른-시작-및-첫-실행-설정)의 빠른 디버그 루프를 사용하세요.

  </Accordion>

  <Accordion title="설치 프로그램이 멈췄나요? 더 많은 피드백은 어떻게 받나요?">
    설치 프로그램을 **verbose 출력**으로 다시 실행하세요.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
    ```

    verbose가 포함된 beta 설치:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    ```

    해킹 가능한(git) 설치의 경우:

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    Windows(PowerShell)에서 동일하게 하려면:

    ```powershell
    # install.ps1에는 아직 전용 -Verbose 플래그가 없습니다.
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

    더 많은 옵션: [설치 프로그램 플래그](/ko/install/installer).

  </Accordion>

  <Accordion title="Windows 설치 시 git not found 또는 openclaw not recognized가 표시됩니다">
    Windows에서 흔한 두 가지 문제입니다.

    **1) npm error spawn git / git not found**

    - **Git for Windows**를 설치하고 `git`이 PATH에 있는지 확인하세요.
    - PowerShell을 닫았다가 다시 열고 설치 프로그램을 다시 실행하세요.

    **2) 설치 후 openclaw is not recognized**

    - npm 전역 bin 폴더가 PATH에 없습니다.
    - 경로를 확인하세요:

      ```powershell
      npm config get prefix
      ```

    - 해당 디렉터리를 사용자 PATH에 추가하세요(Windows에서는 `\bin` 접미사가 필요 없으며, 대부분 시스템에서는 `%AppData%\npm`입니다).
    - PATH를 업데이트한 후 PowerShell을 닫았다가 다시 열어야 합니다.

    가장 매끄러운 Windows 환경을 원한다면 네이티브 Windows보다 **WSL2**를 사용하세요.
    문서: [Windows](/ko/platforms/windows).

  </Accordion>

  <Accordion title="Windows exec 출력에서 중국어가 깨집니다. 어떻게 해야 하나요?">
    보통 네이티브 Windows 셸의 콘솔 코드 페이지 불일치 문제입니다.

    증상:

    - `system.run`/`exec` 출력에서 중국어가 mojibake로 표시됨
    - 같은 명령이 다른 터미널 프로필에서는 정상적으로 보임

    PowerShell에서의 빠른 해결 방법:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    그런 다음 Gateway를 재시작하고 명령을 다시 시도하세요.

    ```powershell
    openclaw gateway restart
    ```

    최신 OpenClaw에서도 계속 재현된다면 다음에서 추적/보고하세요.

    - [Issue #30640](https://github.com/openclaw/openclaw/issues/30640)

  </Accordion>

  <Accordion title="문서가 제 질문에 답하지 못했습니다. 더 나은 답을 얻으려면 어떻게 하나요?">
    **해킹 가능한(git) 설치**를 사용해 전체 소스와 문서를 로컬에 두고, 그 폴더에서
    봇(또는 Claude/Codex)에게 질문하세요. 그러면 저장소를 읽고 정확하게 답할 수 있습니다.

    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    자세한 내용: [설치](/ko/install) 및 [설치 프로그램 플래그](/ko/install/installer).

  </Accordion>

  <Accordion title="Linux에 OpenClaw는 어떻게 설치하나요?">
    짧게 답하면: Linux 가이드를 따른 뒤 온보딩을 실행하세요.

    - Linux 빠른 경로 + 서비스 설치: [Linux](/ko/platforms/linux)
    - 전체 안내: [시작하기](/ko/start/getting-started)
    - 설치 프로그램 + 업데이트: [설치 및 업데이트](/ko/install/updating)

  </Accordion>

  <Accordion title="VPS에 OpenClaw는 어떻게 설치하나요?">
    어떤 Linux VPS든 됩니다. 서버에 설치한 뒤 SSH/Tailscale로 Gateway에 접근하세요.

    가이드: [exe.dev](/ko/install/exe-dev), [Hetzner](/ko/install/hetzner), [Fly.io](/ko/install/fly).
    원격 액세스: [Gateway 원격](/ko/gateway/remote).

  </Accordion>

  <Accordion title="클라우드/VPS 설치 가이드는 어디에 있나요?">
    일반적인 provider를 모아둔 **hosting hub**가 있습니다. 하나를 선택해 가이드를 따르세요.

    - [VPS hosting](/ko/vps) (모든 provider를 한곳에)
    - [Fly.io](/ko/install/fly)
    - [Hetzner](/ko/install/hetzner)
    - [exe.dev](/ko/install/exe-dev)

    클라우드에서의 동작 방식: **Gateway는 서버에서 실행**되며, 노트북/휴대폰에서
    Control UI(또는 Tailscale/SSH)를 통해 접근합니다. state + workspace도 서버에 있으므로,
    호스트를 단일 진실 원본으로 취급하고 백업하세요.

    해당 클라우드 Gateway에 **nodes**(Mac/iOS/Android/headless)를 페어링해
    local screen/camera/canvas에 접근하거나, Gateway를 클라우드에 둔 채
    노트북에서 명령을 실행할 수 있습니다.

    허브: [플랫폼](/ko/platforms). 원격 액세스: [Gateway 원격](/ko/gateway/remote).
    노드: [노드](/ko/nodes), [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="OpenClaw가 스스로 업데이트하게 할 수 있나요?">
    짧게 답하면: **가능하지만 권장하지 않습니다**. 업데이트 흐름은
    Gateway를 재시작할 수 있고(현재 세션이 끊김), 깨끗한 git 체크아웃이 필요할 수 있으며,
    확인 프롬프트가 나타날 수 있습니다. 더 안전한 방법은 운영자가 셸에서 업데이트를 실행하는 것입니다.

    CLI 사용:

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    꼭 에이전트에서 자동화해야 한다면:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    문서: [업데이트](/cli/update), [업데이트](/ko/install/updating).

  </Accordion>

  <Accordion title="온보딩은 실제로 무엇을 하나요?">
    `openclaw onboard`는 권장 설정 경로입니다. **local mode**에서는 다음을 안내합니다.

    - **모델/인증 설정**(provider OAuth, API 키, Anthropic 레거시 setup-token, LM Studio 같은 로컬 모델 옵션 포함)
    - **Workspace** 위치 + bootstrap 파일
    - **Gateway 설정**(bind/port/auth/tailscale)
    - **채널**(WhatsApp, Telegram, Discord, Mattermost, Signal, iMessage 및 QQ Bot 같은 번들 채널 plugins)
    - **데몬 설치**(macOS의 LaunchAgent, Linux/WSL2의 systemd user unit)
    - **상태 점검** 및 **Skills** 선택

    또한 구성된 모델이 알 수 없거나 인증이 없는 경우 경고를 표시합니다.

  </Accordion>

  <Accordion title="이걸 실행하려면 Claude 또는 OpenAI 구독이 필요한가요?">
    아니요. OpenClaw는 **API 키**(Anthropic/OpenAI/기타) 또는
    데이터가 기기에 남도록 **로컬 전용 모델**로 실행할 수 있습니다. 구독(Claude
    Pro/Max 또는 OpenAI Codex)은 해당 provider에 인증하는 선택적 방법일 뿐입니다.

    OpenClaw에서 Anthropic은 실제로 다음처럼 나뉩니다.

    - **Anthropic API key**: 일반 Anthropic API 과금
    - **OpenClaw의 Claude subscription auth**: Anthropic이 **2026년 4월 4일 오후 12:00 PT / 오후 8:00 BST**에 OpenClaw 사용자에게
      구독과 별도로 청구되는 **Extra Usage**가 필요하다고 안내함

    로컬 재현 결과, `claude -p --append-system-prompt ...`도
    추가된 프롬프트가 OpenClaw를 식별할 경우 동일한 Extra Usage 가드에
    걸릴 수 있었지만, 같은 프롬프트 문자열이 Anthropic SDK + API-key 경로에서는
    재현되지 않았습니다. OpenAI Codex OAuth는
    OpenClaw 같은 외부 도구에서 명시적으로 지원됩니다.

    OpenClaw는 다음과 같은 다른 호스팅형 구독 스타일 옵션도 지원합니다.
    **Qwen Cloud Coding Plan**, **MiniMax Coding Plan**, 그리고
    **Z.AI / GLM Coding Plan**.

    문서: [Anthropic](/ko/providers/anthropic), [OpenAI](/ko/providers/openai),
    [Qwen Cloud](/ko/providers/qwen),
    [MiniMax](/ko/providers/minimax), [GLM Models](/ko/providers/glm),
    [로컬 모델](/ko/gateway/local-models), [모델](/ko/concepts/models).

  </Accordion>

  <Accordion title="API 키 없이 Claude Max subscription을 사용할 수 있나요?">
    예. 하지만 이를 **Extra Usage가 포함된 Claude subscription auth**로 취급하세요.

    Claude Pro/Max 구독에는 API 키가 포함되지 않습니다. OpenClaw에서는
    Anthropic의 OpenClaw 전용 과금 안내가 적용되므로, subscription
    트래픽에는 **Extra Usage**가 필요합니다. Extra Usage 경로 없이 Anthropic 트래픽을
    사용하고 싶다면 Anthropic API 키를 대신 사용하세요.

  </Accordion>

  <Accordion title="Claude subscription auth(Claude Pro 또는 Max)를 지원하나요?">
    예. 하지만 현재 지원되는 해석은 다음과 같습니다.

    - 구독으로 OpenClaw에서 Anthropic을 쓰는 것은 **Extra Usage**
    - 그 경로 없이 OpenClaw에서 Anthropic을 쓰는 것은 **API key**

    Anthropic setup-token은 여전히 레거시/수동 OpenClaw 경로로 사용 가능하며,
    Anthropic의 OpenClaw 전용 과금 안내도 계속 적용됩니다. 또한
    추가된 프롬프트가 OpenClaw를 식별할 때 직접적인
    `claude -p --append-system-prompt ...` 사용에서도 같은 과금 가드를 로컬에서 재현했지만,
    같은 프롬프트 문자열은 Anthropic SDK + API-key 경로에서는 재현되지 않았습니다.

    프로덕션이나 multi-user 워크로드에서는 Anthropic API key 인증이 더 안전하고
    권장되는 선택입니다. OpenClaw에서 다른 구독 스타일의 호스팅 옵션을 원한다면
    [OpenAI](/ko/providers/openai), [Qwen / Model
    Cloud](/ko/providers/qwen), [MiniMax](/ko/providers/minimax), [GLM Models](/ko/providers/glm)를 참조하세요.

  </Accordion>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>
<Accordion title="Anthropic에서 HTTP 429 rate_limit_error가 보이는 이유는 무엇인가요?">
이는 현재 윈도우에서 **Anthropic quota/rate limit**이 소진되었음을 뜻합니다. **Claude CLI**를
사용한다면 윈도우가 재설정될 때까지 기다리거나 플랜을 업그레이드하세요. **Anthropic API key**를
사용한다면 Anthropic Console에서 사용량/과금을 확인하고 필요시 한도를 올리세요.

    특히 다음 메시지가 표시된다면:
    `Extra usage is required for long context requests`, 요청이
    Anthropic의 1M context beta(`context1m: true`)를 사용하려는 것입니다. 이는
    자격 증명이 긴 컨텍스트 과금(API key 과금 또는
    Extra Usage가 활성화된 OpenClaw Claude-login 경로)을 사용할 수 있을 때만 동작합니다.

    팁: provider가 rate-limited되더라도 OpenClaw가 계속 응답할 수 있도록
    **fallback model**을 설정하세요.
    [모델](/cli/models), [OAuth](/ko/concepts/oauth), 그리고
    [/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context](/ko/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context)를 참조하세요.

  </Accordion>

  <Accordion title="AWS Bedrock를 지원하나요?">
    예. OpenClaw에는 번들된 **Amazon Bedrock (Converse)** provider가 있습니다. AWS env 마커가 있으면 OpenClaw는 streaming/text Bedrock 카탈로그를 자동 발견해 암시적 `amazon-bedrock` provider로 병합할 수 있고, 그렇지 않으면 `plugins.entries.amazon-bedrock.config.discovery.enabled`를 명시적으로 활성화하거나 수동 provider 항목을 추가할 수 있습니다. [Amazon Bedrock](/ko/providers/bedrock) 및 [모델 providers](/ko/providers/models)를 참조하세요. 관리형 키 흐름을 원한다면 Bedrock 앞에 OpenAI 호환 프록시를 두는 것도 여전히 유효한 옵션입니다.
  </Accordion>

  <Accordion title="Codex 인증은 어떻게 동작하나요?">
    OpenClaw는 OAuth(ChatGPT 로그인)를 통해 **OpenAI Code (Codex)**를 지원합니다. 온보딩은 OAuth 흐름을 실행할 수 있으며, 적절한 경우 기본 모델을 `openai-codex/gpt-5.4`로 설정합니다. [모델 providers](/ko/concepts/model-providers) 및 [온보딩(CLI)](/ko/start/wizard)를 참조하세요.
  </Accordion>

  <Accordion title="OpenAI subscription auth(Codex OAuth)를 지원하나요?">
    예. OpenClaw는 **OpenAI Code (Codex) subscription OAuth**를 완전히 지원합니다.
    OpenAI는 OpenClaw 같은 외부 도구/워크플로에서 subscription OAuth 사용을
    명시적으로 허용합니다. 온보딩이 이 OAuth 흐름을 대신 실행할 수 있습니다.

    [OAuth](/ko/concepts/oauth), [모델 providers](/ko/concepts/model-providers), [온보딩(CLI)](/ko/start/wizard)를 참조하세요.

  </Accordion>

  <Accordion title="Gemini CLI OAuth는 어떻게 설정하나요?">
    Gemini CLI는 `openclaw.json`의 client id 또는 secret이 아니라 **plugin auth flow**를 사용합니다.

    대신 Gemini API provider를 사용하세요.

    1. plugin 활성화: `openclaw plugins enable google`
    2. `openclaw onboard --auth-choice gemini-api-key` 실행
    3. `google/gemini-3.1-pro-preview` 같은 Google 모델 설정

  </Accordion>

  <Accordion title="가벼운 잡담에는 로컬 모델도 괜찮나요?">
    보통은 아닙니다. OpenClaw는 긴 컨텍스트와 강한 안전성이 필요하므로 작은 카드는 잘리고 누출됩니다. 꼭 써야 한다면 로컬에서 가능한 **가장 큰** 모델 빌드(LM Studio)를 실행하고 [/gateway/local-models](/ko/gateway/local-models)를 참조하세요. 더 작거나 양자화된 모델은 prompt injection 위험이 증가합니다. [보안](/ko/gateway/security)을 참조하세요.
  </Accordion>

  <Accordion title="호스팅된 모델 트래픽을 특정 지역에만 머무르게 하려면 어떻게 하나요?">
    지역 고정 엔드포인트를 선택하세요. OpenRouter는 MiniMax, Kimi, GLM에 대해 미국 호스팅 옵션을 제공하므로, 데이터를 지역 내에 유지하려면 US-hosted 변형을 선택하세요. 선택한 지역 provider를 존중하면서도 Anthropic/OpenAI를 함께 나열하려면 `models.mode: "merge"`를 사용해 fallback이 계속 가능하도록 할 수 있습니다.
  </Accordion>

  <Accordion title="이걸 설치하려면 Mac Mini를 사야 하나요?">
    아니요. OpenClaw는 macOS 또는 Linux(Windows는 WSL2를 통해)에서 실행됩니다. Mac mini는 선택사항일 뿐입니다. 일부 사용자는 항상 켜 두는 호스트로 구매하지만, 작은 VPS, 홈 서버 또는 Raspberry Pi 급 장비도 충분히 동작합니다.

    Mac은 **macOS 전용 도구**에만 필요합니다. iMessage는 [BlueBubbles](/ko/channels/bluebubbles)를 사용하세요(권장). BlueBubbles 서버는 어떤 Mac에서도 실행되며, Gateway는 Linux나 다른 곳에서 실행할 수 있습니다. 다른 macOS 전용 도구가 필요하다면 Gateway를 Mac에서 실행하거나 macOS node를 페어링하세요.

    문서: [BlueBubbles](/ko/channels/bluebubbles), [노드](/ko/nodes), [Mac 원격 모드](/ko/platforms/mac/remote).

  </Accordion>

  <Accordion title="iMessage를 지원하려면 Mac mini가 필요한가요?">
    **어떤 macOS 장치든** Messages에 로그인되어 있어야 합니다. 반드시 Mac mini일 필요는 없습니다.
    iMessage에는 **[BlueBubbles](/ko/channels/bluebubbles)**를 사용하세요(권장). BlueBubbles 서버는 macOS에서 실행되고, Gateway는 Linux나 다른 곳에서 실행할 수 있습니다.

    일반적인 구성:

    - Linux/VPS에서 Gateway 실행, Messages에 로그인된 어떤 Mac에서든 BlueBubbles 서버 실행
    - 가장 단순한 단일 머신 구성을 원한다면 Mac에서 모든 것을 실행

    문서: [BlueBubbles](/ko/channels/bluebubbles), [노드](/ko/nodes),
    [Mac 원격 모드](/ko/platforms/mac/remote).

  </Accordion>

  <Accordion title="OpenClaw를 돌리려고 Mac mini를 사면, MacBook Pro와 연결할 수 있나요?">
    예. **Mac mini는 Gateway를 실행**하고, MacBook Pro는
    **node**(보조 장치)로 연결할 수 있습니다. node는 Gateway를 실행하지 않고,
    그 장치에서 screen/camera/canvas 및 `system.run` 같은 추가 기능을 제공합니다.

    일반적인 패턴:

    - 항상 켜져 있는 Mac mini에 Gateway
    - MacBook Pro는 macOS 앱 또는 node host를 실행하고 Gateway와 페어링
    - 상태 확인은 `openclaw nodes status` / `openclaw nodes list` 사용

    문서: [노드](/ko/nodes), [Nodes CLI](/cli/nodes).

  </Accordion>

  <Accordion title="Bun을 사용할 수 있나요?">
    Bun은 **권장하지 않습니다**. 특히 WhatsApp과 Telegram에서 런타임 버그를 확인했습니다.
    안정적인 gateway에는 **Node**를 사용하세요.

    그래도 Bun을 실험하고 싶다면 WhatsApp/Telegram 없이
    비프로덕션 gateway에서만 사용하세요.

  </Accordion>

  <Accordion title="Telegram에서 allowFrom에는 무엇을 넣어야 하나요?">
    `channels.telegram.allowFrom`은 **사람 발신자의 Telegram user ID**(숫자)입니다. 봇 사용자명이 아닙니다.

    온보딩은 `@username` 입력을 받아 숫자 ID로 해석하지만, OpenClaw 인증은 숫자 ID만 사용합니다.

    더 안전한 방법(서드파티 봇 없음):

    - 봇에 DM을 보내고 `openclaw logs --follow`를 실행해 `from.id`를 읽으세요.

    공식 Bot API:

    - 봇에 DM을 보내고 `https://api.telegram.org/bot<bot_token>/getUpdates`를 호출한 뒤 `message.from.id`를 읽으세요.

    서드파티(개인정보 측면에서 덜 안전):

    - `@userinfobot` 또는 `@getidsbot`에 DM을 보내세요.

    [/channels/telegram](/ko/channels/telegram#access-control-and-activation)을 참조하세요.

  </Accordion>

  <Accordion title="한 WhatsApp 번호를 여러 OpenClaw 인스턴스가 서로 다른 방식으로 사용할 수 있나요?">
    예. **multi-agent routing**으로 가능합니다. 각 발신자의 WhatsApp **DM**(peer `kind: "direct"`, 발신자 E.164 예: `+15551234567`)을 다른 `agentId`에 바인딩하면 각 사람이 자신의 workspace와 session store를 갖게 됩니다. 응답은 여전히 **같은 WhatsApp account**에서 오며, DM 접근 제어(`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`)는 WhatsApp account 단위로 전역 적용됩니다. [Multi-Agent Routing](/ko/concepts/multi-agent) 및 [WhatsApp](/ko/channels/whatsapp)을 참조하세요.
  </Accordion>

  <Accordion title='`빠른 채팅` 에이전트와 `코딩용 Opus` 에이전트를 따로 둘 수 있나요?'>
    예. multi-agent routing을 사용하세요. 각 에이전트에 자체 기본 모델을 주고, inbound route(provider account 또는 특정 peers)를 각 에이전트에 바인딩하면 됩니다. 예시 구성은 [Multi-Agent Routing](/ko/concepts/multi-agent)에 있습니다. [모델](/ko/concepts/models) 및 [구성](/ko/gateway/configuration)도 참조하세요.
  </Accordion>

  <Accordion title="Linux에서 Homebrew가 동작하나요?">
    예. Homebrew는 Linux(Linuxbrew)를 지원합니다. 빠른 설정:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    systemd로 OpenClaw를 실행한다면 서비스 PATH에 `/home/linuxbrew/.linuxbrew/bin`(또는 brew prefix)이 포함되어 `brew`로 설치한 도구가 non-login shell에서도 해석되도록 하세요.
    최근 빌드는 Linux systemd services에서 일반적인 사용자 bin 디렉터리(예: `~/.local/bin`, `~/.npm-global/bin`, `~/.local/share/pnpm`, `~/.bun/bin`)를 앞에 추가하고, 설정된 경우 `PNPM_HOME`, `NPM_CONFIG_PREFIX`, `BUN_INSTALL`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `NVM_DIR`, `FNM_DIR`도 반영합니다.

  </Accordion>

  <Accordion title="해킹 가능한 git 설치와 npm install의 차이는 무엇인가요?">
    - **해킹 가능한(git) 설치:** 전체 소스 체크아웃, 수정 가능, 기여자에게 가장 적합
      로컬에서 빌드하며 코드/문서를 직접 수정할 수 있습니다.
    - **npm install:** 전역 CLI 설치, 저장소 없음, "그냥 실행"하기에 가장 적합
      업데이트는 npm dist-tags에서 받습니다.

    문서: [시작하기](/ko/start/getting-started), [업데이트](/ko/install/updating).

  </Accordion>

  <Accordion title="나중에 npm 설치와 git 설치를 바꿔도 되나요?">
    예. 다른 방식으로 설치한 뒤 Doctor를 실행하면 gateway service가 새 entrypoint를 가리키도록 바뀝니다.
    이 작업은 **데이터를 삭제하지 않으며** OpenClaw 코드 설치만 변경합니다. state
    (`~/.openclaw`)와 workspace(`~/.openclaw/workspace`)는 그대로 유지됩니다.

    npm에서 git으로:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    openclaw doctor
    openclaw gateway restart
    ```

    git에서 npm으로:

    ```bash
    npm install -g openclaw@latest
    openclaw doctor
    openclaw gateway restart
    ```

    Doctor는 gateway service entrypoint 불일치를 감지하고 현재 설치에 맞게 서비스 구성을 다시 쓰도록 제안합니다(자동화에서는 `--repair` 사용).

    백업 팁: [백업 전략](#디스크상의-위치)을 참조하세요.

  </Accordion>

  <Accordion title="Gateway는 노트북에서 돌리는 게 좋나요, 아니면 VPS에서 돌리는 게 좋나요?">
    짧게 답하면: **24/7 신뢰성이 필요하다면 VPS를 사용하세요**. 가장 적은 마찰을 원하고 sleep/restart를 감수할 수 있다면 로컬에서 실행하세요.

    **노트북(로컬 Gateway)**

    - **장점:** 서버 비용 없음, 로컬 파일 직접 접근, 실시간 브라우저 창
    - **단점:** sleep/네트워크 끊김 = 연결 해제, OS 업데이트/재부팅으로 중단, 깨어 있어야 함

    **VPS / 클라우드**

    - **장점:** 항상 켜짐, 안정적인 네트워크, 노트북 sleep 문제 없음, 계속 실행하기 쉬움
    - **단점:** 대개 headless로 실행됨(스크린샷 사용), 원격 파일 접근만 가능, 업데이트 시 SSH 필요

    **OpenClaw 특이사항:** WhatsApp/Telegram/Slack/Mattermost/Discord는 모두 VPS에서 잘 동작합니다. 진짜 차이는 **headless browser**와 보이는 창뿐입니다. [브라우저](/ko/tools/browser)를 참조하세요.

    **권장 기본값:** 이전에 gateway 연결 끊김을 겪었다면 VPS를 추천합니다. Mac을 적극적으로 사용 중이고 로컬 파일 접근이나 보이는 브라우저를 통한 UI 자동화가 필요하다면 로컬도 매우 좋습니다.

  </Accordion>

  <Accordion title="OpenClaw를 전용 머신에서 실행하는 것이 얼마나 중요한가요?">
    필수는 아니지만, **신뢰성과 격리 측면에서 권장**됩니다.

    - **전용 호스트(VPS/Mac mini/Pi):** 항상 켜짐, sleep/reboot 중단이 적음, 권한이 더 깔끔함, 계속 실행하기 쉬움
    - **공유 노트북/데스크톱:** 테스트와 적극적 사용에는 충분히 괜찮지만, 머신이 sleep하거나 업데이트할 때 일시 정지를 예상해야 함

    두 장점을 모두 원한다면 Gateway는 전용 호스트에 두고, 노트북을 **node**로 페어링해 local screen/camera/exec 도구를 사용하세요. [노드](/ko/nodes)를 참조하세요.
    보안 지침은 [보안](/ko/gateway/security)을 읽어보세요.

  </Accordion>

  <Accordion title="최소 VPS 요구사항과 권장 OS는 무엇인가요?">
    OpenClaw는 가볍습니다. 기본 Gateway + 하나의 채팅 채널 기준:

    - **절대 최소:** 1 vCPU, 1GB RAM, 약 500MB 디스크
    - **권장:** 여유 공간(로그, 미디어, 여러 채널)을 위해 1-2 vCPU, 2GB RAM 이상. Node 도구와 브라우저 자동화는 자원을 많이 사용할 수 있습니다.

    OS는 **Ubuntu LTS**(또는 현대적인 Debian/Ubuntu 계열)를 사용하세요. Linux 설치 경로가 그 환경에서 가장 잘 테스트되어 있습니다.

    문서: [Linux](/ko/platforms/linux), [VPS hosting](/ko/vps).

  </Accordion>

  <Accordion title="VM에서 OpenClaw를 실행할 수 있나요? 요구사항은 무엇인가요?">
    예. VM도 VPS처럼 취급하세요. 항상 켜져 있고, 연결 가능하며,
    Gateway와 활성화한 채널을 수용할 만큼 충분한 RAM이 필요합니다.

    기본 가이드:

    - **절대 최소:** 1 vCPU, 1GB RAM
    - **권장:** 여러 채널, 브라우저 자동화, 미디어 도구를 실행한다면 2GB RAM 이상
    - **OS:** Ubuntu LTS 또는 다른 최신 Debian/Ubuntu

    Windows에서는 **WSL2가 가장 쉬운 VM 스타일 설정**이며 도구 호환성이 가장 좋습니다.
    [Windows](/ko/platforms/windows), [VPS hosting](/ko/vps)을 참조하세요.
    macOS를 VM에서 실행한다면 [macOS VM](/ko/install/macos-vm)을 참조하세요.

  </Accordion>
</AccordionGroup>

## OpenClaw란 무엇인가요?

<AccordionGroup>
  <Accordion title="OpenClaw를 한 문단으로 설명하면 무엇인가요?">
    OpenClaw는 사용자의 장치에서 직접 실행하는 개인용 AI assistant입니다. 이미 사용하는 메시징 표면(WhatsApp, Telegram, Slack, Mattermost, Discord, Google Chat, Signal, iMessage, WebChat 및 QQ Bot 같은 번들 채널 plugins)에서 응답하며, 지원되는 플랫폼에서는 voice + live Canvas도 사용할 수 있습니다. **Gateway**는 항상 켜져 있는 제어 평면이며, assistant가 곧 제품입니다.
  </Accordion>

  <Accordion title="가치 제안">
    OpenClaw는 "그냥 Claude 래퍼"가 아닙니다. **로컬 우선 제어 평면**으로서,
    이미 사용하는 채팅 앱을 통해 접근할 수 있고, 상태 유지 세션, memory, 도구를 제공하는
    강력한 assistant를 **자신의 하드웨어**에서 실행할 수 있게 해 주며,
    워크플로 제어를 호스팅된 SaaS에 넘기지 않아도 됩니다.

    주요 특징:

    - **내 장치, 내 데이터:** 원하는 곳(Mac, Linux, VPS)에서 Gateway를 실행하고
      workspace + session history를 로컬에 유지
    - **웹 샌드박스가 아닌 실제 채널:** WhatsApp/Telegram/Slack/Discord/Signal/iMessage 등,
      지원 플랫폼에서는 모바일 voice와 Canvas까지 제공
    - **모델 불가지론:** Anthropic, OpenAI, MiniMax, OpenRouter 등을 agent별 라우팅과
      failover와 함께 사용 가능
    - **로컬 전용 옵션:** 원하면 로컬 모델을 실행해 **모든 데이터를 기기에만 유지** 가능
    - **multi-agent routing:** 채널, account, 작업별로 에이전트를 분리하고 각자
      workspace와 기본값을 가짐
    - **오픈 소스이자 해킹 가능:** 벤더 종속 없이 직접 검사, 확장, self-host 가능

    문서: [Gateway](/ko/gateway), [채널](/ko/channels), [Multi-agent](/ko/concepts/multi-agent),
    [메모리](/ko/concepts/memory).

  </Accordion>

  <Accordion title="방금 설정했는데, 먼저 무엇을 해보면 좋을까요?">
    좋은 첫 프로젝트 예시:

    - 웹사이트 만들기(WordPress, Shopify, 또는 간단한 정적 사이트)
    - 모바일 앱 프로토타입(개요, 화면, API 계획)
    - 파일과 폴더 정리(cleanup, naming, tagging)
    - Gmail을 연결하고 요약 또는 follow-up 자동화

    큰 작업도 처리할 수 있지만, 보통 단계를 나누고 병렬 작업에 sub agents를 쓸 때
    가장 잘 작동합니다.

  </Accordion>

  <Accordion title="OpenClaw의 일상적인 상위 다섯 가지 사용 사례는 무엇인가요?">
    일상적인 활용은 보통 다음과 같습니다.

    - **개인 브리핑:** 받은편지함, 캘린더, 관심 있는 뉴스 요약
    - **리서치와 초안 작성:** 빠른 조사, 요약, 이메일/문서 초안 작성
    - **리마인더와 후속 조치:** cron 또는 heartbeat 기반 알림과 체크리스트
    - **브라우저 자동화:** 폼 작성, 데이터 수집, 반복적인 웹 작업
    - **기기 간 조율:** 휴대폰에서 작업을 보내고, Gateway가 서버에서 실행한 뒤 결과를 채팅으로 돌려받기

  </Accordion>

  <Accordion title="SaaS용 리드 생성, 아웃리치, 광고, 블로그 작성에 OpenClaw를 사용할 수 있나요?">
    **조사, 선별, 초안 작성**에는 예. 사이트를 스캔하고, shortlists를 만들고,
    잠재 고객을 요약하며, 아웃리치나 광고 문구 초안을 작성할 수 있습니다.

    **아웃리치나 광고 집행**에는 반드시 사람이 개입하도록 하세요. 스팸을 피하고, 현지 법률과
    플랫폼 정책을 따르며, 전송 전에 항상 검토하세요. 가장 안전한 패턴은
    OpenClaw가 초안을 만들고 사람이 승인하는 것입니다.

    문서: [보안](/ko/gateway/security).

  </Accordion>

  <Accordion title="웹 개발에서 Claude Code 대비 장점은 무엇인가요?">
    OpenClaw는 IDE 대체제가 아니라 **개인 assistant**이자 조율 계층입니다. 저장소 안에서
    가장 빠른 직접 코딩 루프가 필요하면 Claude Code 또는 Codex를 사용하세요. 지속적인 memory,
    여러 기기에서의 접근, 도구 오케스트레이션이 필요할 때 OpenClaw를 사용하세요.

    장점:

    - 세션 간 **지속적인 memory + workspace**
    - **다중 플랫폼 접근**(WhatsApp, Telegram, TUI, WebChat)
    - **도구 오케스트레이션**(브라우저, 파일, 스케줄링, hooks)
    - **항상 켜져 있는 Gateway**(VPS에서 실행하고 어디서나 상호작용)
    - local browser/screen/camera/exec를 위한 **nodes**

    쇼케이스: [https://openclaw.ai/showcase](https://openclaw.ai/showcase)

  </Accordion>
</AccordionGroup>

## Skills와 자동화

<AccordionGroup>
  <Accordion title="저장소를 더럽히지 않고 Skills를 커스터마이즈하려면 어떻게 하나요?">
    저장소 사본을 직접 수정하지 말고 관리형 override를 사용하세요. 변경 사항을 `~/.openclaw/skills/<name>/SKILL.md`에 두거나(`~/.openclaw/openclaw.json`의 `skills.load.extraDirs`를 통해 폴더 추가 가능), 우선순위는 `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → 번들 → `skills.load.extraDirs`입니다. 따라서 git을 건드리지 않고도 관리형 override가 번들 Skills보다 우선합니다. Skill을 전역으로 설치하되 일부 에이전트에만 보이게 하고 싶다면 공유 사본은 `~/.openclaw/skills`에 두고 `agents.defaults.skills`와 `agents.list[].skills`로 가시성을 제어하세요. 업스트림에 기여할 만한 수정만 저장소에 두고 PR로 보내야 합니다.
  </Accordion>

  <Accordion title="사용자 지정 폴더에서 Skills를 로드할 수 있나요?">
    예. `~/.openclaw/openclaw.json`의 `skills.load.extraDirs`를 통해 추가 디렉터리를 지정하세요(가장 낮은 우선순위). 기본 우선순위는 `<workspace>/skills` → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → 번들 → `skills.load.extraDirs`입니다. `clawhub`는 기본적으로 `./skills`에 설치하며, OpenClaw는 다음 세션에서 이를 `<workspace>/skills`로 처리합니다. Skill이 특정 에이전트에만 보여야 한다면 `agents.defaults.skills` 또는 `agents.list[].skills`와 함께 사용하세요.
  </Accordion>

  <Accordion title="작업에 따라 서로 다른 모델을 사용하려면 어떻게 하나요?">
    현재 지원되는 패턴은 다음과 같습니다.

    - **Cron jobs**: 격리된 작업에 job별 `model` override 지정 가능
    - **Sub-agents**: 서로 다른 기본 모델을 가진 별도 에이전트로 작업 라우팅
    - **온디맨드 전환**: 현재 세션 모델을 언제든 `/model`로 변경

    [Cron jobs](/ko/automation/cron-jobs), [Multi-Agent Routing](/ko/concepts/multi-agent), [슬래시 명령어](/ko/tools/slash-commands)를 참조하세요.

  </Accordion>

  <Accordion title="무거운 작업을 할 때 봇이 멈춥니다. 어떻게 오프로드하나요?">
    길거나 병렬적인 작업에는 **sub-agents**를 사용하세요. Sub-agents는 자체 세션에서 실행되고,
    요약을 반환하며, 메인 채팅의 응답성을 유지해 줍니다.

    봇에게 "이 작업에 대해 sub-agent를 생성해"라고 요청하거나 `/subagents`를 사용하세요.
    지금 Gateway가 무엇을 하는지(바쁜지도 포함) 보려면 채팅에서 `/status`를 사용하세요.

    토큰 팁: 긴 작업과 sub-agent는 모두 토큰을 소비합니다. 비용이 걱정된다면
    `agents.defaults.subagents.model`을 통해 sub-agent용 더 저렴한 모델을 설정하세요.

    문서: [Sub-agents](/ko/tools/subagents), [백그라운드 작업](/ko/automation/tasks).

  </Accordion>

  <Accordion title="Discord에서 thread에 묶인 subagent 세션은 어떻게 동작하나요?">
    thread 바인딩을 사용하세요. Discord thread를 subagent 또는 session 대상에 바인딩하면 해당 thread의 후속 메시지가 그 바인딩된 세션에 계속 유지됩니다.

    기본 흐름:

    - `sessions_spawn`을 `thread: true`로 실행(지속적인 후속 처리를 원하면 `mode: "session"`도 선택 가능)
    - 또는 `/focus <target>`으로 수동 바인딩
    - `/agents`로 바인딩 상태 확인
    - `/session idle <duration|off>` 및 `/session max-age <duration|off>`로 자동 unfocus 제어
    - `/unfocus`로 thread 분리

    필요한 구성:

    - 전역 기본값: `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`
    - Discord override: `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.idleHours`, `channels.discord.threadBindings.maxAgeHours`
    - spawn 시 자동 바인딩: `channels.discord.threadBindings.spawnSubagentSessions: true` 설정

    문서: [Sub-agents](/ko/tools/subagents), [Discord](/ko/channels/discord), [구성 참조](/ko/gateway/configuration-reference), [슬래시 명령어](/ko/tools/slash-commands).

  </Accordion>

  <Accordion title="subagent가 끝났는데 완료 알림이 잘못된 곳으로 갔거나 전송되지 않았습니다. 무엇을 확인해야 하나요?">
    먼저 해결된 requester route를 확인하세요.

    - 완료 모드의 subagent 전달은 바인딩된 thread 또는 conversation route가 있으면 이를 우선 사용합니다.
    - 완료 origin에 channel만 있을 경우, OpenClaw는 requester session의 저장된 route(`lastChannel` / `lastTo` / `lastAccountId`)로 fallback하여 direct 전달이 가능하도록 합니다.
    - 바인딩된 route도 쓸 수 있는 저장 route도 없으면 direct 전달이 실패할 수 있으며, 결과는 즉시 채팅에 게시되지 않고 queue 기반 session 전달로 fallback됩니다.
    - 잘못되었거나 오래된 대상은 여전히 queue fallback 또는 최종 전달 실패를 유발할 수 있습니다.
    - child의 마지막으로 보이는 assistant reply가 정확히 silent token `NO_REPLY` / `no_reply` 또는 정확히 `ANNOUNCE_SKIP`이면, OpenClaw는 오래된 이전 진행 상태를 게시하지 않기 위해 의도적으로 알림을 억제합니다.
    - child가 tool 호출만 수행한 뒤 timeout되었다면, 알림은 원시 tool 출력을 재생하는 대신 이를 짧은 부분 진행 요약으로 축약할 수 있습니다.

    디버그:

    ```bash
    openclaw tasks show <runId-or-sessionKey>
    ```

    문서: [Sub-agents](/ko/tools/subagents), [백그라운드 작업](/ko/automation/tasks), [세션 도구](/ko/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron 또는 reminder가 실행되지 않습니다. 무엇을 확인해야 하나요?">
    Cron은 Gateway 프로세스 안에서 실행됩니다. Gateway가 계속 실행 중이 아니면,
    예약 작업은 실행되지 않습니다.

    체크리스트:

    - cron이 활성화되어 있는지(`cron.enabled`)와 `OPENCLAW_SKIP_CRON`이 설정되지 않았는지 확인
    - Gateway가 24/7 실행 중인지 확인(no sleep/restarts)
    - 작업의 timezone 설정(`--tz` vs host timezone) 확인

    디버그:

    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    문서: [Cron jobs](/ko/automation/cron-jobs), [자동화 및 작업](/ko/automation).

  </Accordion>

  <Accordion title="Cron은 실행됐지만 채널로 아무것도 보내지지 않았습니다. 이유가 무엇인가요?">
    먼저 전달 모드를 확인하세요.

    - `--no-deliver` / `delivery.mode: "none"`이면 외부 메시지가 예상되지 않습니다.
    - announce 대상(`channel` / `to`)이 없거나 잘못되면 runner가 outbound delivery를 건너뜁니다.
    - 채널 인증 실패(`unauthorized`, `Forbidden`)는 runner가 전달을 시도했지만 자격 증명에 막혔음을 의미합니다.
    - 침묵하는 격리 결과(`NO_REPLY` / `no_reply`만 반환)는 의도적으로 전달 불가로 취급되므로, runner도 queue fallback delivery를 억제합니다.

    격리된 cron jobs에서는 runner가 최종 전달을 담당합니다. 에이전트는
    runner가 보낼 수 있는 일반 텍스트 요약을 반환해야 합니다. `--no-deliver`는
    결과를 내부에만 유지할 뿐, 에이전트가 message tool로 직접 보내도록
    허용하는 것이 아닙니다.

    디버그:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    문서: [Cron jobs](/ko/automation/cron-jobs), [백그라운드 작업](/ko/automation/tasks).

  </Accordion>

  <Accordion title="격리된 cron 실행 중 모델이 바뀌거나 한 번 재시도된 이유는 무엇인가요?">
    보통 중복 스케줄링이 아니라 live model-switch 경로입니다.

    격리된 cron은 활성 실행이 `LiveSessionModelSwitchError`를 던질 때 런타임 모델 handoff를 지속하고 재시도할 수 있습니다. 재시도는 바뀐 provider/model을 유지하며, 전환이 새로운 auth profile override를 포함했다면 cron은 그것도 저장한 뒤 재시도합니다.

    관련 선택 규칙:

    - 적용되는 경우 Gmail hook model override가 가장 먼저 우선
    - 그다음 job별 `model`
    - 그다음 저장된 cron-session model override
    - 그다음 일반적인 agent/default model 선택

    재시도 루프는 제한되어 있습니다. 초기 시도 + 2회의 switch 재시도 후에는
    영원히 반복하지 않고 중단합니다.

    디버그:

    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <runId-or-sessionKey>
    ```

    문서: [Cron jobs](/ko/automation/cron-jobs), [cron CLI](/cli/cron).

  </Accordion>

  <Accordion title="Linux에서 Skills는 어떻게 설치하나요?">
    네이티브 `openclaw skills` 명령을 사용하거나 workspace에 Skills를 넣으세요. macOS Skills UI는 Linux에서 사용할 수 없습니다.
    Skills는 [https://clawhub.ai](https://clawhub.ai)에서 찾아볼 수 있습니다.

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install <skill-slug>
    openclaw skills install <skill-slug> --version <version>
    openclaw skills install <skill-slug> --force
    openclaw skills update --all
    openclaw skills list --eligible
    openclaw skills check
    ```

    네이티브 `openclaw skills install`은 활성 workspace의 `skills/`
    디렉터리에 기록합니다. 자신의 Skills를 게시하거나 동기화하려는 경우에만 별도의 `clawhub` CLI를 설치하세요. 여러 에이전트 간에 공유하려면 Skill을 `~/.openclaw/skills` 아래에 두고, 어떤 에이전트가 볼지 제한하고 싶다면 `agents.defaults.skills` 또는
    `agents.list[].skills`를 사용하세요.

  </Accordion>

  <Accordion title="OpenClaw는 예약 또는 지속적인 백그라운드 작업을 실행할 수 있나요?">
    예. Gateway scheduler를 사용하세요.

    - **Cron jobs**: 예약되거나 반복되는 작업용(재시작 후에도 유지)
    - **Heartbeat**: "메인 세션"의 주기적 점검용
    - **격리된 jobs**: 요약을 게시하거나 채팅으로 전달하는 자율 에이전트용

    문서: [Cron jobs](/ko/automation/cron-jobs), [자동화 및 작업](/ko/automation),
    [Heartbeat](/ko/gateway/heartbeat).

  </Accordion>

  <Accordion title="Linux에서 Apple macOS 전용 Skills를 실행할 수 있나요?">
    직접적으로는 안 됩니다. macOS Skills는 `metadata.openclaw.os`와 필요한 바이너리에 의해 게이트되며, Skill은 **Gateway host**에서 조건을 만족할 때만 system prompt에 나타납니다. Linux에서는 `darwin` 전용 Skills(`apple-notes`, `apple-reminders`, `things-mac` 등)가 게이팅을 override하지 않는 한 로드되지 않습니다.

    지원되는 패턴은 세 가지입니다.

    **옵션 A - Mac에서 Gateway 실행(가장 단순).**
    macOS 바이너리가 있는 곳에서 Gateway를 실행하고 Linux에서는 [원격 모드](#gateway-포트-이미-실행-중-및-원격-모드) 또는 Tailscale로 연결하세요. Gateway host가 macOS이므로 Skills가 정상 로드됩니다.

    **옵션 B - macOS node 사용(SSH 없음).**
    Linux에서 Gateway를 실행하고 macOS node(menubar app)를 페어링한 뒤, Mac에서 **Node Run Commands**를 "Always Ask" 또는 "Always Allow"로 설정하세요. 필요한 바이너리가 node에 존재하면 OpenClaw는 macOS 전용 Skills를 eligible로 취급할 수 있습니다. 에이전트는 `nodes` tool을 통해 그 Skills를 실행합니다. "Always Ask"를 선택한 경우 프롬프트에서 "Always Allow"를 승인하면 해당 명령이 allowlist에 추가됩니다.

    **옵션 C - SSH로 macOS 바이너리 프록시(고급).**
    Gateway는 Linux에 두고, 필요한 CLI 바이너리가 Mac에서 실행되는 SSH wrapper로 해석되게 하세요. 그런 다음 Linux를 허용하도록 Skill을 override해 eligible 상태를 유지합니다.

    1. 바이너리에 대한 SSH wrapper를 만듭니다(예: Apple Notes용 `memo`):

       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```

    2. Linux host의 `PATH`에 wrapper를 넣습니다(예: `~/bin/memo`).
    3. Skill metadata를 override해 Linux를 허용합니다(workspace 또는 `~/.openclaw/skills`):

       ```markdown
       ---
       name: apple-notes
       description: Manage Apple Notes via the memo CLI on macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```

    4. 새로운 세션을 시작해 Skills snapshot이 새로고침되도록 합니다.

  </Accordion>

  <Accordion title="Notion 또는 HeyGen 통합이 있나요?">
    현재 내장되어 있지는 않습니다.

    옵션:

    - **커스텀 Skill / plugin:** 안정적인 API 접근에 가장 적합(Notion/HeyGen 모두 API 제공)
    - **브라우저 자동화:** 코드 없이 동작하지만 더 느리고 취약함

    클라이언트별 컨텍스트(agency 워크플로)를 유지하고 싶다면 간단한 패턴은 다음과 같습니다.

    - 클라이언트마다 Notion 페이지 하나(context + preferences + active work)
    - 세션 시작 시 에이전트에게 그 페이지를 가져오라고 지시

    네이티브 통합이 필요하다면 기능 요청을 열거나 해당 API를 대상으로 하는 Skill을 구축하세요.

    Skills 설치:

    ```bash
    openclaw skills install <skill-slug>
    openclaw skills update --all
    ```

    네이티브 설치는 활성 workspace의 `skills/` 디렉터리에 들어갑니다. 여러 에이전트에서 공유하려면 `~/.openclaw/skills/<name>/SKILL.md`에 두세요. 공유 설치를 일부 에이전트에만 보이게 하려면 `agents.defaults.skills` 또는 `agents.list[].skills`를 구성하세요. 일부 Skills는 Homebrew로 설치된 바이너리를 기대하며, Linux에서는 Linuxbrew를 뜻합니다(위의 Homebrew Linux FAQ 항목 참조). [Skills](/ko/tools/skills), [Skills 구성](/ko/tools/skills-config), [ClawHub](/ko/tools/clawhub)를 참조하세요.

  </Accordion>

  <Accordion title="기존에 로그인된 Chrome을 OpenClaw와 함께 사용하려면 어떻게 하나요?">
    Chrome DevTools MCP를 통해 연결하는 내장 `user` browser profile을 사용하세요.

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    사용자 지정 이름을 원하면 명시적인 MCP profile을 만드세요.

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    이 경로는 host-local입니다. Gateway가 다른 곳에서 실행 중이라면 브라우저 머신에서 node host를 실행하거나 원격 CDP를 사용하세요.

    현재 `existing-session` / `user`의 제한:

    - 액션은 CSS-selector 기반이 아니라 ref 기반
    - 업로드는 `ref` / `inputRef`가 필요하며 현재는 한 번에 한 파일만 지원
    - `responsebody`, PDF 내보내기, 다운로드 가로채기, 배치 액션은 여전히 관리형 browser 또는 raw CDP profile이 필요

  </Accordion>
</AccordionGroup>

## 샌드박싱과 메모리

<AccordionGroup>
  <Accordion title="전용 샌드박싱 문서가 있나요?">
    예. [샌드박싱](/ko/gateway/sandboxing)을 참조하세요. Docker 전용 설정(전체 gateway를 Docker에서 실행하거나 sandbox 이미지 사용)은 [Docker](/ko/install/docker)를 참조하세요.
  </Accordion>

  <Accordion title="Docker가 제한적으로 느껴집니다. 전체 기능을 활성화하려면 어떻게 하나요?">
    기본 이미지는 보안 우선이며 `node` 사용자로 실행되므로
    system packages, Homebrew, 번들 browsers를 포함하지 않습니다. 더 완전한 설정을 원한다면:

    - 캐시가 유지되도록 `/home/node`를 `OPENCLAW_HOME_VOLUME`으로 영속화
    - `OPENCLAW_DOCKER_APT_PACKAGES`로 system deps를 이미지에 bake
    - 번들 CLI로 Playwright browsers 설치:
      `node /app/node_modules/playwright-core/cli.js install chromium`
    - `PLAYWRIGHT_BROWSERS_PATH`를 설정하고 그 경로를 영속화

    문서: [Docker](/ko/install/docker), [브라우저](/ko/tools/browser).

  </Accordion>

  <Accordion title="DM은 개인용으로 유지하고 그룹은 공개/샌드박스 처리하도록 한 에이전트에서 함께 사용할 수 있나요?">
    예. 개인 트래픽이 **DMs**이고 공개 트래픽이 **groups**라면 가능합니다.

    `agents.defaults.sandbox.mode: "non-main"`을 사용하면 group/channel sessions(비-main 키)는 Docker에서 실행되고, 메인 DM 세션은 호스트에서 유지됩니다. 그런 다음 `tools.sandbox.tools`를 통해 샌드박스 세션에서 사용 가능한 도구를 제한하세요.

    설정 안내 + 예시 구성: [그룹: 개인 DMs + 공개 groups](/ko/channels/groups#pattern-personal-dms-public-groups-single-agent)

    주요 구성 참조: [Gateway 구성](/ko/gateway/configuration-reference#agentsdefaultssandbox)

  </Accordion>

  <Accordion title="호스트 폴더를 sandbox에 바인딩하려면 어떻게 하나요?">
    `agents.defaults.sandbox.docker.binds`를 `["host:path:mode"]`로 설정하세요(예: `"/home/user/src:/src:ro"`). 전역 + 에이전트별 바인드는 병합되며, `scope: "shared"`일 때는 에이전트별 바인드가 무시됩니다. 민감한 항목에는 `:ro`를 사용하고, 바인드는 sandbox 파일시스템 장벽을 우회한다는 점을 기억하세요.

    OpenClaw는 정규화된 경로와 가장 깊은 기존 조상을 통해 해결된 canonical 경로 모두에 대해 bind source를 검증합니다. 즉 마지막 경로 세그먼트가 아직 존재하지 않더라도 symlink-parent 탈출은 여전히 fail closed되며, symlink 해석 후에도 allowed-root 검사가 그대로 적용됩니다.

    예시와 안전 참고는 [샌드박싱](/ko/gateway/sandboxing#custom-bind-mounts) 및 [Sandbox vs Tool Policy vs Elevated](/ko/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check)를 참조하세요.

  </Accordion>

  <Accordion title="메모리는 어떻게 동작하나요?">
    OpenClaw의 메모리는 에이전트 workspace 안의 Markdown 파일일 뿐입니다.

    - `memory/YYYY-MM-DD.md`의 일일 메모
    - `MEMORY.md`의 선별된 장기 메모(main/private sessions 전용)

    OpenClaw는 또한 **조용한 pre-compaction memory flush**를 실행해 모델이
    자동 compaction 전에 지속적인 메모를 기록하도록 상기시킵니다. 이 작업은 workspace가
    쓰기 가능할 때만 실행됩니다(읽기 전용 sandbox는 이를 건너뜀). [메모리](/ko/concepts/memory)를 참조하세요.

  </Accordion>

  <Accordion title="메모리가 자꾸 잊어버립니다. 어떻게 하면 유지되게 할 수 있나요?">
    봇에게 **그 사실을 memory에 기록하라고** 요청하세요. 장기 메모는 `MEMORY.md`에,
    단기 컨텍스트는 `memory/YYYY-MM-DD.md`에 들어가야 합니다.

    이 부분은 아직 계속 개선 중입니다. 모델에게 메모리를 저장하라고 상기시키면
    도움이 됩니다. 무엇을 해야 할지 이해할 것입니다. 계속 잊어버린다면 Gateway가
    매번 같은 workspace를 사용하고 있는지 확인하세요.

    문서: [메모리](/ko/concepts/memory), [에이전트 workspace](/ko/concepts/agent-workspace).

  </Accordion>

  <Accordion title="메모리는 영원히 유지되나요? 한계는 무엇인가요?">
    메모리 파일은 디스크에 저장되며, 삭제할 때까지 유지됩니다. 한계는 모델이 아니라
    저장 공간입니다. 다만 **세션 컨텍스트**는 여전히 모델 컨텍스트 창에 제한되므로
    긴 대화는 compact되거나 truncate될 수 있습니다. 그래서
    memory search가 존재합니다. 관련 부분만 다시 컨텍스트로 불러옵니다.

    문서: [메모리](/ko/concepts/memory), [컨텍스트](/ko/concepts/context).

  </Accordion>

  <Accordion title="의미 기반 memory search에 OpenAI API key가 꼭 필요한가요?">
    **OpenAI embeddings**를 사용할 때만 필요합니다. Codex OAuth는 chat/completions를 다루며
    embeddings 접근 권한은 주지 않으므로, **Codex로 로그인하는 것(OAuth 또는
    Codex CLI 로그인)**은 의미 기반 memory search에 도움이 되지 않습니다. OpenAI embeddings에는
    여전히 실제 API key(`OPENAI_API_KEY` 또는 `models.providers.openai.apiKey`)가 필요합니다.

    provider를 명시적으로 설정하지 않으면 OpenClaw는
    API key(auth profiles, `models.providers.*.apiKey`, 또는 env vars)를 해결할 수 있을 때 provider를 자동 선택합니다.
    OpenAI 키가 해결되면 OpenAI를 우선하고, 그렇지 않으면 Gemini, 그다음 Voyage,
    그다음 Mistral을 시도합니다. 원격 키를 사용할 수 없으면 memory
    search는 구성될 때까지 비활성 상태로 남습니다. 로컬 모델 경로가
    구성되어 있고 실제로 존재한다면 OpenClaw는
    `local`을 우선합니다. Ollama는
    `memorySearch.provider = "ollama"`를 명시적으로 설정했을 때 지원됩니다.

    로컬을 유지하고 싶다면 `memorySearch.provider = "local"`(선택적으로
    `memorySearch.fallback = "none"`도 함께)을 설정하세요. Gemini embeddings를 원한다면
    `memorySearch.provider = "gemini"`로 설정하고 `GEMINI_API_KEY`(또는
    `memorySearch.remote.apiKey`)를 제공하세요. OpenClaw는 **OpenAI, Gemini, Voyage, Mistral, Ollama 또는 local** 임베딩 모델을 지원합니다.
    자세한 설정은 [메모리](/ko/concepts/memory)를 참조하세요.

  </Accordion>
</AccordionGroup>

## 디스크상의 위치

<AccordionGroup>
  <Accordion title="OpenClaw와 함께 사용되는 모든 데이터가 로컬에 저장되나요?">
    아니요. **OpenClaw의 상태는 로컬**에 있지만, **외부 서비스는 여전히 당신이 보내는 내용을 봅니다**.

    - **기본적으로 로컬:** sessions, memory files, config, workspace는 Gateway host에 있음
      (`~/.openclaw` + 사용자의 workspace 디렉터리)
    - **필연적으로 원격:** 모델 providers(Anthropic/OpenAI 등)에 보내는 메시지는
      그들의 API로 가고, 채팅 플랫폼(WhatsApp/Telegram/Slack 등)은 메시지 데이터를
      자신들의 서버에 저장
    - **영향 범위는 사용자가 제어:** 로컬 모델을 사용하면 프롬프트는 기기에 남지만,
      채널 트래픽은 여전히 해당 채널 서버를 거침

    관련 문서: [에이전트 workspace](/ko/concepts/agent-workspace), [메모리](/ko/concepts/memory).

  </Accordion>

  <Accordion title="OpenClaw는 데이터를 어디에 저장하나요?">
    모든 것은 `$OPENCLAW_STATE_DIR`(기본값: `~/.openclaw`) 아래에 있습니다.

    | 경로                                                            | 용도 |
    | --------------------------------------------------------------- | ---- |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                             | 메인 config(JSON5) |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                    | 레거시 OAuth import(첫 사용 시 auth profiles로 복사됨) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | Auth profiles(OAuth, API keys, 선택적 `keyRef`/`tokenRef`) |
    | `$OPENCLAW_STATE_DIR/secrets.json`                              | `file` SecretRef providers용 선택적 파일 기반 secret payload |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`          | 레거시 호환 파일(정적 `api_key` 항목은 scrubbed됨) |
    | `$OPENCLAW_STATE_DIR/credentials/`                              | Provider 상태(예: `whatsapp/<accountId>/creds.json`) |
    | `$OPENCLAW_STATE_DIR/agents/`                                   | 에이전트별 상태(agentDir + sessions) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                | 대화 기록 및 상태(에이전트별) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/sessions.json`   | 세션 메타데이터(에이전트별) |

    레거시 single-agent 경로: `~/.openclaw/agent/*`(`openclaw doctor`가 마이그레이션).

    **Workspace**(AGENTS.md, memory files, skills 등)는 별도로 분리되어 있으며 `agents.defaults.workspace`를 통해 구성합니다(기본값: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="AGENTS.md / SOUL.md / USER.md / MEMORY.md는 어디에 있어야 하나요?">
    이 파일들은 `~/.openclaw`가 아니라 **에이전트 workspace**에 있어야 합니다.

    - **Workspace(에이전트별)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`,
      `MEMORY.md`(없으면 레거시 대체 `memory.md`),
      `memory/YYYY-MM-DD.md`, 선택적 `HEARTBEAT.md`
    - **State dir(`~/.openclaw`)**: config, channel/provider 상태, auth profiles, sessions, logs,
      공유 Skills(`~/.openclaw/skills`)

    기본 workspace는 `~/.openclaw/workspace`이며 다음으로 구성할 수 있습니다.

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    재시작 후 봇이 "잊어버리는" 것처럼 보인다면, Gateway가 매번 같은
    workspace를 사용해 시작하는지 확인하세요(그리고 remote mode는 **gateway host의**
    workspace를 사용하며, 로컬 노트북의 것이 아니라는 점도 기억하세요).

    팁: 지속적인 동작이나 선호를 원한다면 채팅 기록에 의존하지 말고
    봇에게 **AGENTS.md 또는 MEMORY.md에 기록하라고** 요청하세요.

    [에이전트 workspace](/ko/concepts/agent-workspace) 및 [메모리](/ko/concepts/memory)를 참조하세요.

  </Accordion>

  <Accordion title="권장 백업 전략">
    **에이전트 workspace**를 **비공개** git 저장소에 넣고 어딘가 사적으로
    백업하세요(예: GitHub private). 이렇게 하면 memory + AGENTS/SOUL/USER
    파일을 포착하고, 나중에 assistant의 "마음"을 복원할 수 있습니다.

    `~/.openclaw` 아래의 어떤 것도 커밋하지 마세요(credentials, sessions, tokens, 암호화된 secrets payload 포함).
    완전 복원이 필요하다면 workspace와 state directory를
    별도로 모두 백업하세요(위의 migration 질문 참조).

    문서: [에이전트 workspace](/ko/concepts/agent-workspace).

  </Accordion>

  <Accordion title="OpenClaw를 완전히 제거하려면 어떻게 하나요?">
    전용 가이드를 참조하세요: [제거](/ko/install/uninstall).
  </Accordion>

  <Accordion title="에이전트가 workspace 밖에서도 작업할 수 있나요?">
    예. workspace는 **기본 cwd**이자 memory anchor이지, 강제 샌드박스는 아닙니다.
    상대 경로는 workspace 안에서 해석되지만, 절대 경로는 sandboxing이 비활성화된 경우
    호스트의 다른 위치에도 접근할 수 있습니다. 격리가 필요하다면
    [`agents.defaults.sandbox`](/ko/gateway/sandboxing) 또는 에이전트별 sandbox 설정을 사용하세요. 저장소를 기본 작업 디렉터리로 사용하고 싶다면 그 에이전트의
    `workspace`를 저장소 루트로 지정하세요. OpenClaw 저장소는 단지 소스 코드일 뿐이므로,
    에이전트가 의도적으로 그 안에서 작업하게 하려는 경우가 아니라면 workspace는 분리해 두세요.

    예시(저장소를 기본 cwd로 사용):

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="원격 모드에서는 session store가 어디에 있나요?">
    세션 상태는 **gateway host**가 소유합니다. remote mode라면, 확인해야 할 session store는 로컬 노트북이 아니라 원격 머신에 있습니다. [세션 관리](/ko/concepts/session)를 참조하세요.
  </Accordion>
</AccordionGroup>

## 기본 구성

<AccordionGroup>
  <Accordion title="config 형식은 무엇이며, 어디에 있나요?">
    OpenClaw는 `$OPENCLAW_CONFIG_PATH`(기본값: `~/.openclaw/openclaw.json`)에서 선택적인 **JSON5** 구성을 읽습니다.

    ```
    $OPENCLAW_CONFIG_PATH
    ```

    파일이 없으면 안전에 가까운 기본값(기본 workspace `~/.openclaw/workspace` 포함)을 사용합니다.

  </Accordion>

  <Accordion title='`gateway.bind: "lan"`(또는 `"tailnet"`)을 설정했더니 아무것도 수신하지 않거나 UI에 unauthorized가 표시됩니다'>
    Non-loopback bind에는 **유효한 gateway auth 경로가 필요합니다**. 실제로는 다음을 의미합니다.

    - shared-secret 인증: token 또는 password
    - 올바르게 구성된 non-loopback identity-aware reverse proxy 뒤의 `gateway.auth.mode: "trusted-proxy"`

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    참고:

    - `gateway.remote.token` / `.password`는 로컬 gateway auth를 자동으로 활성화하지 않습니다.
    - 로컬 호출 경로는 `gateway.auth.*`가 비어 있을 때만 `gateway.remote.*`를 fallback으로 사용할 수 있습니다.
    - password 인증의 경우 `gateway.auth.mode: "password"`와 `gateway.auth.password`(또는 `OPENCLAW_GATEWAY_PASSWORD`)를 설정하세요.
    - `gateway.auth.token` / `gateway.auth.password`가 SecretRef로 명시적으로 구성되어 있는데 해결되지 않으면, fail closed로 처리되며 원격 fallback으로 가려지지 않습니다.
    - Shared-secret Control UI 설정은 `connect.params.auth.token` 또는 `connect.params.auth.password`(앱/UI 설정에 저장됨)로 인증합니다. Tailscale Serve나 `trusted-proxy` 같은 identity-bearing 모드는 대신 request headers를 사용합니다. shared secret을 URL에 넣지 마세요.
    - `gateway.auth.mode: "trusted-proxy"`에서는 같은 호스트의 loopback reverse proxy는 여전히 trusted-proxy auth를 충족하지 않습니다. trusted proxy는 구성된 non-loopback source여야 합니다.

  </Accordion>

  <Accordion title="왜 이제 localhost에서도 token이 필요한가요?">
    OpenClaw는 loopback을 포함해 기본적으로 gateway auth를 강제합니다. 일반적인 기본 경로에서는 token auth를 의미합니다. 명시적인 auth 경로가 구성되지 않으면 gateway 시작 시 token 모드로 해석되고 자동 생성된 token이 `gateway.auth.token`에 저장되므로 **로컬 WS 클라이언트도 인증해야 합니다**. 이렇게 하면 다른 로컬 프로세스가 Gateway를 호출하는 것을 막을 수 있습니다.

    다른 인증 경로를 선호한다면 password 모드(또는 non-loopback identity-aware reverse proxy의 경우 `trusted-proxy`)를 명시적으로 선택할 수 있습니다. **정말로** 열린 loopback을 원한다면 config에서 `gateway.auth.mode: "none"`을 명시적으로 설정하세요. Doctor는 언제든 token을 생성해 줄 수 있습니다: `openclaw doctor --generate-gateway-token`.

  </Accordion>

  <Accordion title="config를 바꾼 뒤 반드시 재시작해야 하나요?">
    Gateway는 config를 감시하며 hot-reload를 지원합니다.

    - `gateway.reload.mode: "hybrid"`(기본값): 안전한 변경은 hot-apply, 중요한 변경은 재시작
    - `hot`, `restart`, `off`도 지원

  </Accordion>

  <Accordion title="재미있는 CLI 태그라인을 끄려면 어떻게 하나요?">
    config에서 `cli.banner.taglineMode`를 설정하세요.

    ```json5
    {
      cli: {
        banner: {
          taglineMode: "off", // random | default | off
        },
      },
    }
    ```

    - `off`: 태그라인 텍스트를 숨기지만 배너 제목/버전 줄은 유지
    - `default`: 항상 `All your chats, one OpenClaw.` 사용
    - `random`: 웃긴/계절성 태그라인 순환(기본 동작)
    - 배너를 아예 원하지 않으면 env `OPENCLAW_HIDE_BANNER=1` 설정

  </Accordion>

  <Accordion title="웹 검색(및 웹 가져오기)은 어떻게 활성화하나요?">
    `web_fetch`는 API key 없이 동작합니다. `web_search`는 선택한
    provider에 따라 다릅니다.

    - Brave, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Perplexity, Tavily 같은 API 기반 providers는 일반적인 API key 설정이 필요합니다.
    - Ollama Web Search는 키가 필요 없지만, 구성된 Ollama host를 사용하며 `ollama signin`이 필요합니다.
    - DuckDuckGo는 키가 필요 없지만 비공식 HTML 기반 통합입니다.
    - SearXNG는 키가 필요 없는/self-hosted 방식이며 `SEARXNG_BASE_URL` 또는 `plugins.entries.searxng.config.webSearch.baseUrl`를 구성하세요.

    **권장:** `openclaw configure --section web`를 실행해 provider를 선택하세요.
    환경 변수 대안:

    - Brave: `BRAVE_API_KEY`
    - Exa: `EXA_API_KEY`
    - Firecrawl: `FIRECRAWL_API_KEY`
    - Gemini: `GEMINI_API_KEY`
    - Grok: `XAI_API_KEY`
    - Kimi: `KIMI_API_KEY` 또는 `MOONSHOT_API_KEY`
    - MiniMax Search: `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, 또는 `MINIMAX_API_KEY`
    - Perplexity: `PERPLEXITY_API_KEY` 또는 `OPENROUTER_API_KEY`
    - SearXNG: `SEARXNG_BASE_URL`
    - Tavily: `TAVILY_API_KEY`

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
        },
        tools: {
          web: {
            search: {
              enabled: true,
              provider: "brave",
              maxResults: 5,
            },
            fetch: {
              enabled: true,
              provider: "firecrawl", // 선택 사항; 자동 감지를 원하면 생략
            },
          },
        },
    }
    ```

    provider별 web-search config는 이제 `plugins.entries.<plugin>.config.webSearch.*` 아래에 있습니다.
    레거시 `tools.web.search.*` provider 경로도 호환을 위해 일시적으로는 로드되지만, 새 config에서는 사용하지 않아야 합니다.
    Firecrawl web-fetch fallback config는 `plugins.entries.firecrawl.config.webFetch.*` 아래에 있습니다.

    참고:

    - allowlist를 사용한다면 `web_search`/`web_fetch`/`x_search` 또는 `group:web`를 추가하세요.
    - `web_fetch`는 기본적으로 활성화되어 있습니다(명시적으로 비활성화하지 않는 한).
    - `tools.web.fetch.provider`를 생략하면 OpenClaw는 사용 가능한 자격 증명에서 준비된 첫 fetch fallback provider를 자동 감지합니다. 현재 번들 provider는 Firecrawl입니다.
    - 데몬은 `~/.openclaw/.env`(또는 서비스 환경)에서 env vars를 읽습니다.

    문서: [웹 도구](/ko/tools/web).

  </Accordion>

  <Accordion title="config.apply가 내 config를 날려버렸습니다. 어떻게 복구하고 방지하나요?">
    `config.apply`는 **전체 config를 대체**합니다. 부분 객체를 보내면 나머지는 모두 제거됩니다.

    복구:

    - 백업(git 또는 복사해 둔 `~/.openclaw/openclaw.json`)에서 복원
    - 백업이 없다면 `openclaw doctor`를 다시 실행하고 채널/모델 재구성
    - 예상치 못한 동작이었다면 버그를 보고하고 마지막으로 알고 있던 config나 백업을 첨부
    - 로컬 코딩 에이전트는 로그나 기록에서 동작 가능한 config를 재구성할 수 있는 경우가 많음

    방지:

    - 작은 변경에는 `openclaw config set` 사용
    - 대화형 편집에는 `openclaw configure` 사용
    - 정확한 경로나 필드 형식이 확실하지 않을 때는 먼저 `config.schema.lookup` 사용. 얕은 schema 노드와 즉시 자식 요약을 반환함
    - 부분 RPC 편집에는 `config.patch` 사용, `config.apply`는 전체 config 교체에만 사용
    - 에이전트 실행에서 owner 전용 `gateway` tool을 사용하는 경우에도 `tools.exec.ask` / `tools.exec.security`에 대한 쓰기는 계속 거부합니다(동일한 보호된 exec 경로로 정규화되는 레거시 `tools.bash.*` alias 포함)

    문서: [Config](/cli/config), [Configure](/cli/configure), [Doctor](/ko/gateway/doctor).

  </Accordion>

  <Accordion title="여러 장치에서 전문화된 worker와 함께 중앙 Gateway를 운영하려면 어떻게 하나요?">
    일반적인 패턴은 **하나의 Gateway**(예: Raspberry Pi)와 **nodes**, **agents**입니다.

    - **Gateway(중앙):** 채널(Signal/WhatsApp), 라우팅, sessions를 소유
    - **Nodes(장치):** Macs/iOS/Android가 주변 장치처럼 연결되어 local tools(`system.run`, `canvas`, `camera`)를 노출
    - **Agents(workers):** 특수 역할(예: "Hetzner ops", "Personal data")을 위한 별도 brain/workspace
    - **Sub-agents:** 메인 agent에서 백그라운드 병렬 작업을 생성하고 싶을 때 사용
    - **TUI:** Gateway에 연결해 agents/sessions 전환

    문서: [노드](/ko/nodes), [원격 액세스](/ko/gateway/remote), [Multi-Agent Routing](/ko/concepts/multi-agent), [Sub-agents](/ko/tools/subagents), [TUI](/web/tui).

  </Accordion>

  <Accordion title="OpenClaw browser는 headless로 실행할 수 있나요?">
    예. config 옵션입니다.

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    기본값은 `false`(headful)입니다. Headless는 일부 사이트에서 anti-bot 검사를 더 자주 유발할 수 있습니다. [브라우저](/ko/tools/browser)를 참조하세요.

    Headless는 **같은 Chromium 엔진**을 사용하며 대부분의 자동화(폼, 클릭, 스크래핑, 로그인)에 동작합니다. 주요 차이점:

    - 보이는 브라우저 창이 없음(시각 자료가 필요하면 스크린샷 사용)
    - 일부 사이트는 headless 모드에서 자동화에 더 엄격함(CAPTCHA, anti-bot)
      예를 들어 X/Twitter는 headless 세션을 자주 차단합니다.

  </Accordion>

  <Accordion title="브라우저 제어에 Brave를 사용하려면 어떻게 하나요?">
    `browser.executablePath`를 Brave 바이너리(또는 다른 Chromium 기반 브라우저)로 설정하고 Gateway를 재시작하세요.
    전체 config 예시는 [브라우저](/ko/tools/browser#use-brave-or-another-chromium-based-browser)를 참조하세요.
  </Accordion>
</AccordionGroup>

## 원격 gateways와 nodes

<AccordionGroup>
  <Accordion title="Telegram, gateway, nodes 사이에서 명령은 어떻게 전달되나요?">
    Telegram 메시지는 **gateway**가 처리합니다. gateway가 agent를 실행한 후,
    node tool이 필요할 때만 **Gateway WebSocket**을 통해 nodes를 호출합니다.

    Telegram → Gateway → Agent → `node.*` → Node → Gateway → Telegram

    Nodes는 inbound provider 트래픽을 보지 못하며, node RPC 호출만 받습니다.

  </Accordion>

  <Accordion title="Gateway가 원격 호스팅되는 경우 에이전트가 내 컴퓨터에 접근하려면 어떻게 하나요?">
    짧게 답하면: **컴퓨터를 node로 페어링**하세요. Gateway는 다른 곳에서 실행되지만,
    Gateway WebSocket을 통해 로컬 머신의 `node.*` tools(screen, camera, system)를
    호출할 수 있습니다.

    일반적인 설정:

    1. 항상 켜져 있는 호스트(VPS/home server)에서 Gateway 실행
    2. Gateway host와 컴퓨터를 같은 tailnet에 연결
    3. Gateway WS에 접근 가능한지 확인(tailnet bind 또는 SSH tunnel)
    4. 로컬에서 macOS 앱을 열고 **Remote over SSH** 모드(또는 직접 tailnet)로 연결해
       node로 등록되도록 함
    5. Gateway에서 node 승인:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    별도의 TCP bridge는 필요하지 않으며, nodes는 Gateway WebSocket을 통해 연결됩니다.

    보안 참고: macOS node를 페어링하면 그 머신에서 `system.run`이 가능해집니다. 신뢰하는 장치만
    페어링하고, [보안](/ko/gateway/security)을 검토하세요.

    문서: [노드](/ko/nodes), [Gateway 프로토콜](/ko/gateway/protocol), [macOS 원격 모드](/ko/platforms/mac/remote), [보안](/ko/gateway/security).

  </Accordion>

  <Accordion title="Tailscale은 연결되어 있지만 응답이 없습니다. 어떻게 해야 하나요?">
    기본 사항을 확인하세요.

    - Gateway가 실행 중인지: `openclaw gateway status`
    - Gateway 상태: `openclaw status`
    - 채널 상태: `openclaw channels status`

    그런 다음 인증과 라우팅을 확인하세요.

    - Tailscale Serve를 사용한다면 `gateway.auth.allowTailscale`이 올바르게 설정되었는지 확인
    - SSH tunnel로 연결한다면 로컬 tunnel이 살아 있고 올바른 포트를 가리키는지 확인
    - allowlists(DM 또는 group)에 내 account가 포함되어 있는지 확인

    문서: [Tailscale](/ko/gateway/tailscale), [원격 액세스](/ko/gateway/remote), [채널](/ko/channels).

  </Accordion>

  <Accordion title="두 OpenClaw 인스턴스(로컬 + VPS)가 서로 대화할 수 있나요?">
    예. 내장된 "bot-to-bot" 브리지는 없지만 몇 가지
    신뢰할 수 있는 방법으로 연결할 수 있습니다.

    **가장 단순한 방법:** 두 봇이 접근 가능한 일반 채팅 채널(Telegram/Slack/WhatsApp)을 사용하세요.
    Bot A가 Bot B에 메시지를 보내고, Bot B가 평소처럼 응답하게 하면 됩니다.

    **CLI 브리지(범용):** 다른 Gateway에
    `openclaw agent --message ... --deliver`를 호출하는 스크립트를 실행하고, 대상 봇이
    수신하는 채팅을 지정하세요. 한 봇이 원격 VPS에 있다면
    SSH/Tailscale을 통해 그 원격 Gateway를 가리키도록 CLI를 설정하세요([원격 액세스](/ko/gateway/remote) 참조).

    예시 패턴(대상 Gateway에 접근 가능한 머신에서 실행):

    ```bash
    openclaw agent --message "Hello from local bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    팁: 두 봇이 끝없이 반복 응답하지 않도록 보호 장치를 두세요(멘션 전용,
    채널 allowlists, 또는 "봇 메시지에는 답하지 말 것" 규칙).

    문서: [원격 액세스](/ko/gateway/remote), [Agent CLI](/cli/agent), [Agent send](/ko/tools/agent-send).

  </Accordion>

  <Accordion title="여러 agent를 위해 별도의 VPS가 필요한가요?">
    아니요. 하나의 Gateway가 여러 agent를 호스팅할 수 있으며, 각 agent는 자신의 workspace, 기본 모델,
    라우팅을 가질 수 있습니다. 이것이 일반적인 설정이며
    agent마다 VPS를 따로 두는 것보다 훨씬 저렴하고 단순합니다.

    보안 경계 같은 강한 격리나,
    공유하고 싶지 않은 매우 다른 config가 필요할 때만 별도 VPS를 사용하세요. 그렇지 않다면
    Gateway 하나에 여러 agents 또는 sub-agents를 사용하세요.

  </Accordion>

  <Accordion title="원격 Gateway에서 SSH로 접근하는 대신 개인 노트북에서 node를 쓰는 이점이 있나요?">
    예. nodes는 원격 Gateway에서 노트북에 접근하는 일급 방식이며,
    셸 접근 이상의 기능을 제공합니다. Gateway는 macOS/Linux(Windows는 WSL2)에서
    실행되며 가볍기 때문에(작은 VPS 또는 Raspberry Pi급 장비면 충분, 4 GB RAM이면 넉넉),
    항상 켜진 호스트와 노트북을 node로 두는 구성이 일반적입니다.

    - **인바운드 SSH 불필요.** Nodes는 Gateway WebSocket으로 outbound 연결하고 장치 페어링을 사용합니다.
    - **더 안전한 실행 제어.** `system.run`은 해당 노트북의 node allowlists/approvals로 제어됩니다.
    - **더 많은 장치 도구.** Nodes는 `system.run` 외에도 `canvas`, `camera`, `screen`을 제공합니다.
    - **로컬 브라우저 자동화.** Gateway는 VPS에 두고, 노트북에서 node host를 통해 로컬 Chrome을 실행하거나, 호스트의 Chrome MCP를 통해 로컬 Chrome에 연결할 수 있습니다.

    SSH는 임시 셸 접근에는 괜찮지만, nodes는 지속적인 agent 워크플로와
    장치 자동화에 더 단순합니다.

    문서: [노드](/ko/nodes), [Nodes CLI](/cli/nodes), [브라우저](/ko/tools/browser).

  </Accordion>

  <Accordion title="nodes는 gateway service를 실행하나요?">
    아니요. **gateway는 호스트당 하나만** 실행해야 합니다. 의도적으로 격리된 profiles를 실행하는 경우는 예외입니다([여러 gateways](/ko/gateway/multiple-gateways) 참조). nodes는 gateway에 연결되는 주변 장치(iOS/Android nodes, 또는 menubar app의 macOS "node mode")입니다. headless node hosts와 CLI 제어는 [Node host CLI](/cli/node)를 참조하세요.

    `gateway`, `discovery`, `canvasHost` 변경에는 전체 재시작이 필요합니다.

  </Accordion>

  <Accordion title="config를 적용하는 API / RPC 방법이 있나요?">
    예.

    - `config.schema.lookup`: 쓰기 전에 하나의 config subtree를 얕은 schema 노드, 일치한 UI 힌트, 즉시 자식 요약과 함께 검사
    - `config.get`: 현재 스냅샷 + 해시 가져오기
    - `config.patch`: 안전한 부분 업데이트(대부분의 RPC 편집에 권장)
    - `config.apply`: 전체 config를 검증하고 교체한 뒤 재시작
    - owner 전용 `gateway` 런타임 도구는 여전히 `tools.exec.ask` / `tools.exec.security` 재작성을 거부하며, 레거시 `tools.bash.*` alias도 같은 보호된 exec 경로로 정규화됨

  </Accordion>

  <Accordion title="첫 설치를 위한 최소한의 정상 구성">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    이 설정은 workspace를 지정하고 누가 봇을 트리거할 수 있는지 제한합니다.

  </Accordion>

  <Accordion title="VPS에 Tailscale을 설정하고 Mac에서 연결하려면 어떻게 하나요?">
    최소 단계:

    1. **VPS에 설치 + 로그인**

       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```

    2. **Mac에 설치 + 로그인**
       - Tailscale 앱을 사용해 같은 tailnet에 로그인하세요.
    3. **MagicDNS 활성화(권장)**
       - Tailscale 관리자 콘솔에서 MagicDNS를 켜서 VPS가 안정적인 이름을 갖도록 하세요.
    4. **tailnet hostname 사용**
       - SSH: `ssh user@your-vps.tailnet-xxxx.ts.net`
       - Gateway WS: `ws://your-vps.tailnet-xxxx.ts.net:18789`

    SSH 없이 Control UI를 쓰려면 VPS에서 Tailscale Serve를 사용하세요.

    ```bash
    openclaw gateway --tailscale serve
    ```

    이렇게 하면 gateway는 loopback에 바인딩된 채로 유지되고, Tailscale을 통해 HTTPS가 노출됩니다. [Tailscale](/ko/gateway/tailscale)을 참조하세요.

  </Accordion>

  <Accordion title="원격 Gateway(Tailscale Serve)에 Mac node를 연결하려면 어떻게 하나요?">
    Serve는 **Gateway Control UI + WS**를 노출합니다. Nodes는 같은 Gateway WS 엔드포인트로 연결합니다.

    권장 설정:

    1. **VPS와 Mac이 같은 tailnet에 있는지 확인**
    2. **macOS 앱을 Remote 모드로 사용**(SSH 대상은 tailnet hostname이어도 됨)
       앱이 Gateway 포트를 터널링하고 node로 연결합니다.
    3. gateway에서 node 승인:

       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    문서: [Gateway 프로토콜](/ko/gateway/protocol), [Discovery](/ko/gateway/discovery), [macOS 원격 모드](/ko/platforms/mac/remote).

  </Accordion>

  <Accordion title="두 번째 노트북에 설치해야 하나요, 아니면 node만 추가하면 되나요?">
    두 번째 노트북에서 **local tools**(screen/camera/exec)만 필요하다면
    **node**로 추가하세요. 그러면 Gateway를 하나만 유지하면서 중복 config를 피할 수 있습니다. Local node tools는
    현재 macOS 전용이지만, 다른 OS로도 확장할 계획입니다.

    강한 격리나 완전히 분리된 두 봇이 필요할 때만 두 번째 Gateway를 설치하세요.

    문서: [노드](/ko/nodes), [Nodes CLI](/cli/nodes), [여러 gateways](/ko/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## 환경 변수와 .env 로딩

<AccordionGroup>
  <Accordion title="OpenClaw는 환경 변수를 어떻게 로드하나요?">
    OpenClaw는 부모 프로세스(셸, launchd/systemd, CI 등)에서 env vars를 읽고, 추가로 다음도 로드합니다.

    - 현재 작업 디렉터리의 `.env`
    - `~/.openclaw/.env`(즉 `$OPENCLAW_STATE_DIR/.env`)의 전역 fallback `.env`

    어느 `.env` 파일도 기존 env vars를 override하지 않습니다.

    config에 inline env vars를 정의할 수도 있습니다(프로세스 env에 없을 때만 적용).

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    전체 우선순위와 소스는 [/environment](/ko/help/environment)를 참조하세요.

  </Accordion>

  <Accordion title="service로 Gateway를 시작했더니 env vars가 사라졌습니다. 어떻게 해야 하나요?">
    흔한 해결책은 두 가지입니다.

    1. 누락된 키를 `~/.openclaw/.env`에 넣어 service가 셸 env를 상속하지 않아도 로드되게 하세요.
    2. shell import를 활성화하세요(선택적 편의 기능).

    ```json5
    {
      env: {
        shellEnv: {
          enabled: true,
          timeoutMs: 15000,
        },
      },
    }
    ```

    이 기능은 로그인 셸을 실행해 예상된 누락 키만 가져오며(절대 override하지 않음), 해당하는 env var는
    `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`입니다.

  </Accordion>

  <Accordion title='`COPILOT_GITHUB_TOKEN`을 설정했는데 models status에는 "Shell env: off."라고 나옵니다. 왜 그런가요?'>
    `openclaw models status`는 **shell env import**가 활성화되어 있는지 보고합니다. "Shell env: off"는
    env vars가 없다는 뜻이 아니라, OpenClaw가 로그인 셸을 자동으로 로드하지
    않는다는 뜻입니다.

    Gateway가 service(launchd/systemd)로 실행되면 셸 환경을 상속하지 않습니다.
    다음 중 하나로 해결하세요.

    1. token을 `~/.openclaw/.env`에 넣기:

       ```
       COPILOT_GITHUB_TOKEN=...
       ```

    2. 또는 shell import(`env.shellEnv.enabled: true`) 활성화
    3. 또는 config의 `env` 블록에 추가(없을 때만 적용)

    그런 다음 gateway를 재시작하고 다시 확인하세요.

    ```bash
    openclaw models status
    ```

    Copilot token은 `COPILOT_GITHUB_TOKEN`(또는 `GH_TOKEN` / `GITHUB_TOKEN`)에서 읽습니다.
    [/concepts/model-providers](/ko/concepts/model-providers) 및 [/environment](/ko/help/environment)를 참조하세요.

  </Accordion>
</AccordionGroup>

## 세션과 여러 채팅

<AccordionGroup>
  <Accordion title="새 대화를 시작하려면 어떻게 하나요?">
    `/new` 또는 `/reset`을 독립된 메시지로 보내세요. [세션 관리](/ko/concepts/session)를 참조하세요.
  </Accordion>

  <Accordion title="/new를 한 번도 보내지 않으면 세션이 자동으로 초기화되나요?">
    세션은 `session.idleMinutes` 이후 만료될 수 있지만, 이 기능은 **기본적으로 비활성화**되어 있습니다(기본값 **0**).
    양수 값을 설정해 idle expiry를 활성화하세요. 활성화되면 idle 기간 이후의 **다음**
    메시지가 해당 채팅 키에 대해 새로운 session id를 시작합니다.
    이는 대화록을 삭제하는 것이 아니라 새 세션을 시작하는 것입니다.

    ```json5
    {
      session: {
        idleMinutes: 240,
      },
    }
    ```

  </Accordion>

  <Accordion title="OpenClaw 인스턴스 팀(CEO 한 명과 여러 agents) 같은 구성을 만들 수 있나요?">
    예. **multi-agent routing**과 **sub-agents**를 통해 가능합니다. 하나의 coordinator
    agent와, 각자 자신의 workspace와 모델을 가진 여러 worker agents를 만들 수 있습니다.

    다만 이것은 **재미있는 실험**으로 보는 편이 좋습니다. 토큰을 많이 사용하며,
    하나의 봇에 여러 세션을 두는 것보다 비효율적인 경우가 많습니다. 우리가
    보통 떠올리는 모델은 하나의 봇과, 병렬 작업을 위한 서로 다른 세션들입니다. 이
    봇은 필요 시 sub-agents도 생성할 수 있습니다.

    문서: [Multi-agent routing](/ko/concepts/multi-agent), [Sub-agents](/ko/tools/subagents), [Agents CLI](/cli/agents).

  </Accordion>

  <Accordion title="작업 도중 컨텍스트가 잘렸습니다. 어떻게 방지하나요?">
    세션 컨텍스트는 모델 창에 의해 제한됩니다. 긴 채팅, 큰 tool 출력, 많은
    파일은 compaction 또는 truncation을 유발할 수 있습니다.

    도움이 되는 방법:

    - 봇에게 현재 상태를 요약해 파일에 기록하게 하세요.
    - 긴 작업 전에 `/compact`, 주제를 바꿀 때는 `/new`를 사용하세요.
    - 중요한 컨텍스트는 workspace에 두고 봇에게 다시 읽게 하세요.
    - 긴 작업이나 병렬 작업에는 sub-agents를 사용해 메인 채팅을 더 작게 유지하세요.
    - 이런 일이 자주 발생하면 더 큰 컨텍스트 창을 가진 모델을 선택하세요.

  </Accordion>

  <Accordion title="OpenClaw를 완전히 초기화하되 설치는 유지하려면 어떻게 하나요?">
    reset 명령을 사용하세요.

    ```bash
    openclaw reset
    ```

    비대화형 전체 초기화:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    그런 다음 설정을 다시 실행하세요.

    ```bash
    openclaw onboard --install-daemon
    ```

    참고:

    - 기존 config가 보이면 온보딩도 **Reset**을 제안합니다. [온보딩(CLI)](/ko/start/wizard)를 참조하세요.
    - profiles(`--profile` / `OPENCLAW_PROFILE`)를 사용했다면 각 state dir를 개별적으로 초기화하세요(기본값은 `~/.openclaw-<profile>`).
    - Dev reset: `openclaw gateway --dev --reset`(dev 전용; dev config + credentials + sessions + workspace를 삭제)

  </Accordion>

  <Accordion title='`context too large` 오류가 발생합니다. 어떻게 reset 또는 compact하나요?'>
    다음 중 하나를 사용하세요.

    - **Compact**(대화는 유지하고 오래된 turn만 요약):

      ```
      /compact
      ```

      또는 요약 방향을 주려면 `/compact <instructions>`.

    - **Reset**(같은 채팅 키에 대해 새 session ID 시작):

      ```
      /new
      /reset
      ```

    계속 발생한다면:

    - **session pruning**(`agents.defaults.contextPruning`)을 활성화하거나 조정해 오래된 tool 출력을 잘라내세요.
    - 더 큰 컨텍스트 창을 가진 모델을 사용하세요.

    문서: [Compaction](/ko/concepts/compaction), [세션 pruning](/ko/concepts/session-pruning), [세션 관리](/ko/concepts/session).

  </Accordion>

  <Accordion title='왜 "LLM request rejected: messages.content.tool_use.input field required"가 보이나요?'>
    이는 provider 검증 오류입니다. 모델이 필수 `input` 없이 `tool_use` 블록을 출력했다는 뜻입니다. 보통 세션 기록이 오래되었거나 손상된 경우에 발생합니다(긴 thread나 tool/schema 변경 후가 흔함).

    해결: `/new`를 독립된 메시지로 보내 새 세션을 시작하세요.

  </Accordion>

  <Accordion title="왜 30분마다 heartbeat 메시지가 오나요?">
    Heartbeat는 기본적으로 **30분**마다 실행됩니다(OAuth 인증을 사용할 때는 **1시간**). 조정하거나 비활성화하려면:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // 또는 비활성화하려면 "0m"
          },
        },
      },
    }
    ```

    `HEARTBEAT.md`가 존재하지만 사실상 비어 있으면(빈 줄과 `# Heading` 같은 markdown
    헤더만 포함), OpenClaw는 API 호출을 아끼기 위해 heartbeat 실행을 건너뜁니다.
    파일이 없어도 heartbeat는 실행되며 모델이 무엇을 할지 결정합니다.

    에이전트별 override는 `agents.list[].heartbeat`를 사용합니다. 문서: [Heartbeat](/ko/gateway/heartbeat).

  </Accordion>

  <Accordion title='WhatsApp 그룹에 "봇 계정"을 추가해야 하나요?'>
    아니요. OpenClaw는 **당신 자신의 account**에서 실행되므로, 당신이 그룹에 있으면 OpenClaw도 그 그룹을 볼 수 있습니다.
    기본적으로 group 응답은 발신자를 허용할 때까지 차단됩니다(`groupPolicy: "allowlist"`).

    그룹 응답을 **자신만** 트리거하게 하고 싶다면:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="WhatsApp 그룹의 JID는 어떻게 얻나요?">
    옵션 1(가장 빠름): 로그를 tail한 상태에서 그룹에 테스트 메시지를 보내세요.

    ```bash
    openclaw logs --follow --json
    ```

    `@g.us`로 끝나는 `chatId`(또는 `from`)를 찾으세요. 예:
    `1234567890-1234567890@g.us`.

    옵션 2(이미 구성/allowlist된 경우): config에서 그룹 목록을 봅니다.

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    문서: [WhatsApp](/ko/channels/whatsapp), [Directory](/cli/directory), [로그](/cli/logs).

  </Accordion>

  <Accordion title="왜 OpenClaw가 그룹에서 응답하지 않나요?">
    흔한 원인은 두 가지입니다.

    - mention gating이 켜져 있습니다(기본값). 봇을 @mention해야 하거나 `mentionPatterns`에 맞아야 합니다.
    - `channels.whatsapp.groups`를 `"*"` 없이 구성했고 그 그룹이 allowlist에 없습니다.

    [Groups](/ko/channels/groups) 및 [그룹 메시지](/ko/channels/group-messages)를 참조하세요.

  </Accordion>

  <Accordion title="그룹/threads는 DM과 컨텍스트를 공유하나요?">
    Direct chats는 기본적으로 main session으로 합쳐집니다. Groups/channels는 자체 session keys를 가지며, Telegram topics / Discord threads는 별도 세션입니다. [Groups](/ko/channels/groups) 및 [그룹 메시지](/ko/channels/group-messages)를 참조하세요.
  </Accordion>

  <Accordion title="workspaces와 agents를 몇 개까지 만들 수 있나요?">
    하드 제한은 없습니다. 수십 개(심지어 수백 개)도 괜찮지만 다음을 주의하세요.

    - **디스크 증가:** sessions + transcripts는 `~/.openclaw/agents/<agentId>/sessions/` 아래에 저장됨
    - **토큰 비용:** agents가 많을수록 동시에 모델을 더 많이 사용
    - **운영 오버헤드:** agent별 auth profiles, workspaces, channel routing

    팁:

    - agent마다 하나의 **활성** workspace(`agents.defaults.workspace`) 유지
    - 디스크가 커지면 오래된 sessions(JSONL 또는 store entries) 정리
    - `openclaw doctor`로 흩어진 workspaces와 profile 불일치 점검

  </Accordion>

  <Accordion title="여러 봇 또는 채팅을 동시에(Slack) 실행할 수 있나요? 어떻게 설정해야 하나요?">
    예. **Multi-Agent Routing**을 사용해 여러 개의 격리된 agents를 실행하고,
    inbound messages를 channel/account/peer별로 라우팅하세요. Slack은 채널로 지원되며
    특정 agents에 바인딩할 수 있습니다.

    브라우저 접근은 강력하지만 "사람이 할 수 있는 모든 것"을 뜻하지는 않습니다. anti-bot, CAPTCHAs, MFA는
    여전히 자동화를 막을 수 있습니다. 가장 신뢰할 수 있는 브라우저 제어를 위해서는 호스트의 local Chrome MCP를 사용하거나,
    실제로 browser를 실행하는 머신에서 CDP를 사용하세요.

    모범 사례 설정:

    - 항상 켜진 Gateway host(VPS/Mac mini)
    - 역할별로 하나의 agent(bindings)
    - 그 agents에 바인딩된 Slack channels
    - 필요할 때는 Chrome MCP 또는 node를 통한 local browser

    문서: [Multi-Agent Routing](/ko/concepts/multi-agent), [Slack](/ko/channels/slack),
    [브라우저](/ko/tools/browser), [노드](/ko/nodes).

  </Accordion>
</AccordionGroup>

## 모델: 기본값, 선택, aliases, 전환

<AccordionGroup>
  <Accordion title='`기본 모델`이란 무엇인가요?'>
    OpenClaw의 기본 모델은 다음으로 설정한 값입니다.

    ```
    agents.defaults.model.primary
    ```

    모델은 `provider/model` 형식으로 참조합니다(예: `openai/gpt-5.4`). provider를 생략하면 OpenClaw는 먼저 alias를 시도하고, 그다음 정확한 model id에 대한 고유한 configured-provider 일치를 찾고, 마지막으로 deprecated 호환 경로로 구성된 기본 provider에 fallback합니다. 해당 provider가 더 이상 구성된 기본 모델을 제공하지 않으면, 오래되어 제거된 provider 기본값을 노출하는 대신 첫 번째로 구성된 provider/model로 fallback합니다. 그래도 **명시적으로** `provider/model`을 설정하는 것이 좋습니다.

  </Accordion>

  <Accordion title="어떤 모델을 권장하나요?">
    **권장 기본값:** provider 스택에서 사용할 수 있는 가장 강력한 최신 세대 모델을 사용하세요.
    **도구를 사용하거나 신뢰할 수 없는 입력을 받는 에이전트:** 비용보다 모델 성능을 우선하세요.
    **일상적이거나 낮은 위험의 채팅:** 더 저렴한 fallback 모델을 사용하고 agent 역할별로 라우팅하세요.

    MiniMax 전용 문서: [MiniMax](/ko/providers/minimax) 및
    [로컬 모델](/ko/gateway/local-models).

    경험칙: 중요한 작업에는 **감당 가능한 가장 좋은 모델**을,
    일상적인 채팅이나 요약에는 더 저렴한 모델을 사용하세요. agent별로 모델을 라우팅하고 sub-agents를 사용해
    긴 작업을 병렬화할 수 있습니다(sub-agent마다 토큰을 사용함). [모델](/ko/concepts/models) 및
    [Sub-agents](/ko/tools/subagents)를 참조하세요.

    강한 경고: 더 약하거나 과도하게 양자화된 모델은 prompt
    injection과 안전하지 않은 동작에 더 취약합니다. [보안](/ko/gateway/security)을 참조하세요.

    더 자세한 내용: [모델](/ko/concepts/models).

  </Accordion>

  <Accordion title="config를 지우지 않고 모델을 바꾸려면 어떻게 하나요?">
    **model 명령**을 사용하거나 **model** 필드만 편집하세요. 전체 config 교체는 피하세요.

    안전한 옵션:

    - 채팅에서 `/model`(빠르고 세션별)
    - `openclaw models set ...`(모델 config만 업데이트)
    - `openclaw configure --section model`(대화형)
    - `~/.openclaw/openclaw.json`의 `agents.defaults.model` 편집

    전체 config를 교체할 의도가 아니라면 부분 객체와 함께 `config.apply`를 사용하지 마세요.
    RPC 편집의 경우 먼저 `config.schema.lookup`으로 검사하고 `config.patch`를 선호하세요. lookup payload는 정규화된 경로, 얕은 schema 문서/제약, 즉시 자식 요약을 제공합니다.
    부분 업데이트에 적합합니다.
    이미 config를 덮어썼다면 백업에서 복원하거나 `openclaw doctor`를 다시 실행해 복구하세요.

    문서: [모델](/ko/concepts/models), [Configure](/cli/configure), [Config](/cli/config), [Doctor](/ko/gateway/doctor).

  </Accordion>

  <Accordion title="self-hosted 모델(llama.cpp, vLLM, Ollama)을 사용할 수 있나요?">
    예. 로컬 모델에는 Ollama가 가장 쉬운 경로입니다.

    가장 빠른 설정:

    1. `https://ollama.com/download`에서 Ollama 설치
    2. `ollama pull glm-4.7-flash` 같은 로컬 모델 가져오기
    3. 클라우드 모델도 쓰고 싶다면 `ollama signin` 실행
    4. `openclaw onboard` 실행 후 `Ollama` 선택
    5. `Local` 또는 `Cloud + Local` 선택

    참고:

    - `Cloud + Local`은 클라우드 모델 + 로컬 Ollama 모델을 함께 제공
    - `kimi-k2.5:cloud` 같은 클라우드 모델은 로컬 pull이 필요 없음
    - 수동 전환에는 `openclaw models list`와 `openclaw models set ollama/<model>` 사용

    보안 참고: 더 작거나 많이 양자화된 모델은 prompt
    injection에 더 취약합니다. 도구를 사용할 수 있는 봇에는 **큰 모델**을 강력히 권장합니다.
    그래도 작은 모델을 쓰고 싶다면 sandboxing과 엄격한 tool allowlists를 활성화하세요.

    문서: [Ollama](/ko/providers/ollama), [로컬 모델](/ko/gateway/local-models),
    [모델 providers](/ko/concepts/model-providers), [보안](/ko/gateway/security),
    [샌드박싱](/ko/gateway/sandboxing).

  </Accordion>

  <Accordion title="OpenClaw, Flawd, Krill은 어떤 모델을 사용하나요?">
    - 이 배포들은 서로 다를 수 있고 시간에 따라 바뀔 수 있으며, 고정된 provider 권장안은 없습니다.
    - 각 gateway의 현재 런타임 설정은 `openclaw models status`로 확인하세요.
    - 보안에 민감하거나 도구를 사용하는 agents에는 사용 가능한 가장 강력한 최신 세대 모델을 사용하세요.
  </Accordion>

  <Accordion title="재시작 없이 즉시 모델을 바꾸려면 어떻게 하나요?">
    `/model` 명령을 독립된 메시지로 사용하세요.

    ```
    /model sonnet
    /model opus
    /model gpt
    /model gpt-mini
    /model gemini
    /model gemini-flash
    /model gemini-flash-lite
    ```

    이것들은 내장 alias입니다. 사용자 정의 alias는 `agents.defaults.models`를 통해 추가할 수 있습니다.

    사용 가능한 모델은 `/model`, `/model list`, `/model status`로 볼 수 있습니다.

    `/model`(및 `/model list`)은 간단한 번호 선택기를 표시합니다. 번호로 선택하세요.

    ```
    /model 3
    ```

    현재 세션에서 provider에 대해 특정 auth profile을 강제할 수도 있습니다.

    ```
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    팁: `/model status`는 현재 어떤 agent가 활성 상태인지, 어떤 `auth-profiles.json` 파일을 쓰는지, 다음에 어떤 auth profile이 시도될지를 보여줍니다.
    사용 가능한 경우 구성된 provider endpoint(`baseUrl`)와 API mode(`api`)도 보여줍니다.

    **`@profile`로 설정한 profile 고정은 어떻게 해제하나요?**

    `@profile` 접미사 없이 `/model`을 다시 실행하세요.

    ```
    /model anthropic/claude-opus-4-6
    ```

    기본값으로 돌아가고 싶다면 `/model`에서 기본값을 선택하세요(또는 `/model <default provider/model>` 전송).
    어떤 auth profile이 활성 상태인지 확인하려면 `/model status`를 사용하세요.

  </Accordion>

  <Accordion title="일상 작업에는 GPT 5.2를, 코딩에는 Codex 5.3을 사용할 수 있나요?">
    예. 하나를 기본값으로 두고 필요할 때 전환하면 됩니다.

    - **빠른 전환(세션별):** 일상 작업에는 `/model gpt-5.4`, Codex OAuth를 사용하는 코딩에는 `/model openai-codex/gpt-5.4`
    - **기본값 + 전환:** `agents.defaults.model.primary`를 `openai/gpt-5.4`로 설정한 뒤, 코딩 시 `openai-codex/gpt-5.4`로 전환(또는 반대로)
    - **Sub-agents:** 코딩 작업을 다른 기본 모델을 가진 sub-agent에 라우팅

    [모델](/ko/concepts/models) 및 [슬래시 명령어](/ko/tools/slash-commands)를 참조하세요.

  </Accordion>

  <Accordion title="GPT 5.4의 fast mode는 어떻게 구성하나요?">
    세션 토글 또는 config 기본값 중 하나를 사용하세요.

    - **세션별:** 세션이 `openai/gpt-5.4` 또는 `openai-codex/gpt-5.4`를 사용하는 동안 `/fast on` 전송
    - **모델별 기본값:** `agents.defaults.models["openai/gpt-5.4"].params.fastMode`를 `true`로 설정
    - **Codex OAuth도 함께:** `openai-codex/gpt-5.4`를 함께 쓴다면 그 항목에도 같은 플래그 설정

    예시:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
            "openai-codex/gpt-5.4": {
              params: {
                fastMode: true,
              },
            },
          },
        },
      },
    }
    ```

    OpenAI의 경우 fast mode는 지원되는 네이티브 Responses 요청에서 `service_tier = "priority"`로 매핑됩니다. 세션의 `/fast` override가 config 기본값보다 우선합니다.

    [Thinking and fast mode](/ko/tools/thinking) 및 [OpenAI fast mode](/ko/providers/openai#openai-fast-mode)를 참조하세요.

  </Accordion>

  <Accordion title='왜 "Model ... is not allowed"가 보인 뒤 응답이 없나요?'>
    `agents.defaults.models`가 설정되면 `/model`과 세션 override에 대한 **allowlist**가 됩니다.
    그 목록에 없는 모델을 선택하면 다음이 반환됩니다.

    ```
    Model "provider/model" is not allowed. Use /model to list available models.
    ```

    이 오류는 정상 응답 **대신** 반환됩니다. 해결 방법: 모델을
    `agents.defaults.models`에 추가하거나, allowlist를 제거하거나, `/model list`에서 모델을 선택하세요.

  </Accordion>

  <Accordion title='왜 "Unknown model: minimax/MiniMax-M2.7"가 보이나요?'>
    이는 **provider가 구성되지 않았음**을 뜻합니다(MiniMax provider config나 auth
    profile을 찾지 못함). 그래서 모델을 해석할 수 없습니다.

    해결 체크리스트:

    1. 현재 OpenClaw 릴리스로 업그레이드하거나 소스 `main`에서 실행한 뒤 gateway 재시작
    2. MiniMax가 구성되어 있는지(wizard 또는 JSON), 또는 MiniMax auth가 env/auth profiles에 있어 해당 provider가 주입될 수 있는지 확인
       (`minimax`용 `MINIMAX_API_KEY`, `minimax-portal`용 `MINIMAX_OAUTH_TOKEN` 또는 저장된 MiniMax
       OAuth)
    3. auth 경로에 맞는 정확한 model id(대소문자 구분)를 사용:
       API key 설정은 `minimax/MiniMax-M2.7` 또는 `minimax/MiniMax-M2.7-highspeed`,
       OAuth 설정은 `minimax-portal/MiniMax-M2.7` /
       `minimax-portal/MiniMax-M2.7-highspeed`
    4. 다음 실행:

       ```bash
       openclaw models list
       ```

       목록에서 고르거나 채팅에서는 `/model list` 사용

    [MiniMax](/ko/providers/minimax) 및 [모델](/ko/concepts/models)을 참조하세요.

  </Accordion>

  <Accordion title="기본값은 MiniMax로, 복잡한 작업에는 OpenAI를 사용할 수 있나요?">
    예. **기본값은 MiniMax**로 두고 필요할 때 **세션별로** 모델을 바꾸면 됩니다.
    Fallback은 **오류용**이지 "어려운 작업"용이 아니므로, `/model` 또는 별도 agent를 사용하세요.

    **옵션 A: 세션별 전환**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M2.7" },
          models: {
            "minimax/MiniMax-M2.7": { alias: "minimax" },
            "openai/gpt-5.4": { alias: "gpt" },
          },
        },
      },
    }
    ```

    그런 다음:

    ```
    /model gpt
    ```

    **옵션 B: 별도 agents**

    - Agent A 기본값: MiniMax
    - Agent B 기본값: OpenAI
    - agent별로 라우팅하거나 `/agent`로 전환

    문서: [모델](/ko/concepts/models), [Multi-Agent Routing](/ko/concepts/multi-agent), [MiniMax](/ko/providers/minimax), [OpenAI](/ko/providers/openai).

  </Accordion>

  <Accordion title="opus / sonnet / gpt는 내장 단축키인가요?">
    예. OpenClaw는 몇 가지 기본 단축키를 제공합니다(`agents.defaults.models`에 해당 모델이 있을 때만 적용).

    - `opus` → `anthropic/claude-opus-4-6`
    - `sonnet` → `anthropic/claude-sonnet-4-6`
    - `gpt` → `openai/gpt-5.4`
    - `gpt-mini` → `openai/gpt-5.4-mini`
    - `gpt-nano` → `openai/gpt-5.4-nano`
    - `gemini` → `google/gemini-3.1-pro-preview`
    - `gemini-flash` → `google/gemini-3-flash-preview`
    - `gemini-flash-lite` → `google/gemini-3.1-flash-lite-preview`

    같은 이름의 alias를 직접 설정하면 사용자 값이 우선합니다.

  </Accordion>

  <Accordion title="모델 단축키(alias)를 정의/override하려면 어떻게 하나요?">
    Alias는 `agents.defaults.models.<modelId>.alias`에서 옵니다. 예시:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
            "anthropic/claude-haiku-4-5": { alias: "haiku" },
          },
        },
      },
    }
    ```

    그러면 `/model sonnet`(또는 지원되는 경우 `/<alias>`)이 해당 model ID로 해석됩니다.

  </Accordion>

  <Accordion title="OpenRouter나 Z.AI 같은 다른 provider의 모델은 어떻게 추가하나요?">
    OpenRouter(토큰당 과금, 다양한 모델):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI(GLM 모델):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5" },
          models: { "zai/glm-5": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    provider/model을 참조했는데 필요한 provider 키가 없으면 런타임 auth 오류가 발생합니다(예: `No API key found for provider "zai"`).

    **새 agent를 추가한 뒤 `No API key found for provider`가 나타나는 경우**

    보통 **새 agent**의 auth store가 비어 있다는 뜻입니다. Auth는 agent별이며 다음에 저장됩니다.

    ```
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    해결 방법:

    - `openclaw agents add <id>`를 실행하고 마법사에서 auth 구성
    - 또는 메인 agent의 `agentDir`에서 `auth-profiles.json`을 새 agent의 `agentDir`로 복사

    agents 간에 `agentDir`를 재사용하지 마세요. auth/session 충돌이 발생합니다.

  </Accordion>
</AccordionGroup>

## 모델 failover와 "All models failed"

<AccordionGroup>
  <Accordion title="failover는 어떻게 동작하나요?">
    Failover는 두 단계로 일어납니다.

    1. 같은 provider 내에서 **auth profile rotation**
    2. `agents.defaults.model.fallbacks`의 다음 모델로 **model fallback**

    실패하는 profiles에는 cooldown(지수 백오프)이 적용되므로 provider가 rate-limited되거나 일시적으로 실패해도 OpenClaw가 계속 응답할 수 있습니다.

    rate-limit 버킷은 단순한 `429` 응답보다 넓습니다. OpenClaw는
    `Too many concurrent requests`,
    `ThrottlingException`, `concurrency limit reached`,
    `workers_ai ... quota limit exceeded`, `resource exhausted`, 그리고
    주기적 사용량 윈도우 제한(`weekly/monthly limit reached`) 같은 메시지도 failover할 만한
    rate limits로 취급합니다.

    일부 billing처럼 보이는 응답은 `402`가 아니고, 일부 HTTP `402`
    응답도 그 일시적 버킷에 남습니다. provider가
    `401` 또는 `403`에 명시적인