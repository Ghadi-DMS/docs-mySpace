---
read_when:
    - doctor 마이그레이션을 추가하거나 수정하고 있습니다
    - 호환되지 않는 config 변경을 도입하고 있습니다
summary: 'Doctor command: 상태 점검, config 마이그레이션, 복구 단계'
title: Doctor
x-i18n:
    generated_at: "2026-04-06T03:08:42Z"
    model: gpt-5.4
    provider: openai
    source_hash: 6c0a15c522994552a1eef39206bed71fc5bf45746776372f24f31c101bfbd411
    source_path: gateway/doctor.md
    workflow: 15
---

# Doctor

`openclaw doctor`는 OpenClaw용 복구 + 마이그레이션 도구입니다. 오래된
config/state를 수정하고, 상태를 점검하며, 실행 가능한 복구 단계를 제공합니다.

## 빠른 시작

```bash
openclaw doctor
```

### 헤드리스 / 자동화

```bash
openclaw doctor --yes
```

프롬프트 없이 기본값을 수락합니다(해당되는 경우 restart/service/sandbox 복구 단계 포함).

```bash
openclaw doctor --repair
```

프롬프트 없이 권장 복구를 적용합니다(안전한 경우 복구 + restart 포함).

```bash
openclaw doctor --repair --force
```

공격적인 복구도 적용합니다(사용자 지정 supervisor config를 덮어씀).

```bash
openclaw doctor --non-interactive
```

프롬프트 없이 실행하고 안전한 마이그레이션만 적용합니다(config 정규화 + 디스크 상 state 이동). 사람의 확인이 필요한 restart/service/sandbox 작업은 건너뜁니다.
레거시 state 마이그레이션은 감지되면 자동으로 실행됩니다.

```bash
openclaw doctor --deep
```

추가 gateway 설치를 위해 시스템 서비스(launchd/systemd/schtasks)를 검사합니다.

쓰기 전에 변경 사항을 검토하고 싶다면 먼저 config 파일을 여세요:

```bash
cat ~/.openclaw/openclaw.json
```

## 수행 작업(요약)

- git 설치에 대한 선택적 사전 업데이트(대화형 전용).
- UI 프로토콜 최신성 점검(프로토콜 스키마가 더 최신이면 Control UI를 다시 빌드).
- 상태 점검 + restart 프롬프트.
- Skills 상태 요약(적격/누락/차단) 및 plugin 상태.
- 레거시 값에 대한 config 정규화.
- 레거시 평면 `talk.*` 필드에서 `talk.provider` + `talk.providers.<provider>`로의 Talk config 마이그레이션.
- 레거시 Chrome extension config 및 Chrome MCP 준비 상태에 대한 browser 마이그레이션 점검.
- OpenCode provider override 경고(`models.providers.opencode` / `models.providers.opencode-go`).
- OpenAI Codex OAuth 프로필에 대한 OAuth TLS 전제 조건 점검.
- 레거시 디스크 상 state 마이그레이션(sessions/agent dir/WhatsApp auth).
- 레거시 plugin manifest contract key 마이그레이션(`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
- 레거시 cron 저장소 마이그레이션(`jobId`, `schedule.cron`, 최상위 delivery/payload 필드, payload `provider`, 단순 `notify: true` webhook 폴백 작업).
- 세션 lock 파일 검사 및 오래된 lock 정리.
- state 무결성 및 권한 점검(sessions, transcripts, state dir).
- 로컬에서 실행할 때 config 파일 권한 점검(`chmod 600`).
- 모델 auth 상태: OAuth 만료를 점검하고, 만료가 임박한 토큰을 갱신할 수 있으며, auth-profile cooldown/비활성화 상태를 보고.
- 추가 workspace dir 감지(`~/openclaw`).
- sandboxing이 활성화되어 있을 때 sandbox image 복구.
- 레거시 서비스 마이그레이션 및 추가 gateway 감지.
- Matrix 채널 레거시 state 마이그레이션(`--fix` / `--repair` 모드에서).
- Gateway 런타임 점검(서비스가 설치되었지만 실행되지 않음, 캐시된 launchd label).
- 채널 상태 경고(실행 중인 gateway에서 프로브).
- Supervisor config 감사(launchd/systemd/schtasks) 및 선택적 복구.
- Gateway 런타임 모범 사례 점검(Node vs Bun, version-manager 경로).
- Gateway 포트 충돌 진단(기본값 `18789`).
- 열린 DM 정책에 대한 보안 경고.
- 로컬 토큰 모드에 대한 gateway auth 점검(토큰 소스가 없을 때 토큰 생성을 제안하며, 토큰 SecretRef config는 덮어쓰지 않음).
- Linux에서 systemd linger 점검.
- Workspace bootstrap 파일 크기 점검(컨텍스트 파일의 잘림/한계 근접 경고).
- 셸 completion 상태 점검 및 자동 설치/업그레이드.
- 메모리 검색 임베딩 provider 준비 상태 점검(로컬 모델, 원격 API key 또는 QMD 바이너리).
- 소스 설치 점검(pnpm workspace 불일치, 누락된 UI assets, 누락된 tsx 바이너리).
- 업데이트된 config + wizard 메타데이터 쓰기.

## 상세 동작과 근거

### 0) 선택적 업데이트(git 설치)

이것이 git checkout이고 doctor가 대화형으로 실행 중이면 doctor를 실행하기 전에
업데이트(fetch/rebase/build)를 제안합니다.

### 1) Config 정규화

config에 레거시 값 형태가 포함되어 있으면(예: 채널별 override가 없는
`messages.ackReaction`), doctor가 이를 현재 스키마로 정규화합니다.

여기에는 레거시 Talk 평면 필드도 포함됩니다. 현재 공개 Talk config는
`talk.provider` + `talk.providers.<provider>`입니다. Doctor는 이전
`talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` /
`talk.apiKey` 형태를 provider 맵으로 다시 작성합니다.

### 2) 레거시 config 키 마이그레이션

config에 더 이상 사용되지 않는 키가 포함되어 있으면 다른 command는 실행을 거부하고
`openclaw doctor`를 실행하라고 요청합니다.

Doctor는 다음을 수행합니다:

- 어떤 레거시 키가 발견되었는지 설명합니다.
- 적용한 마이그레이션을 보여줍니다.
- 업데이트된 스키마로 `~/.openclaw/openclaw.json`을 다시 작성합니다.

Gateway도 레거시 config 형식을 감지하면 시작 시 doctor 마이그레이션을 자동 실행하므로
오래된 config가 수동 개입 없이 복구됩니다.
Cron 작업 저장소 마이그레이션은 `openclaw doctor --fix`가 처리합니다.

현재 마이그레이션:

- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.requireMention` → `channels.whatsapp/telegram/imessage.groups."*".requireMention`
- `routing.groupChat.historyLimit` → `messages.groupChat.historyLimit`
- `routing.groupChat.mentionPatterns` → `messages.groupChat.mentionPatterns`
- `routing.queue` → `messages.queue`
- `routing.bindings` → 최상위 `bindings`
- `routing.agents`/`routing.defaultAgentId` → `agents.list` + `agents.list[].default`
- 레거시 `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` → `talk.provider` + `talk.providers.<provider>`
- `routing.agentToAgent` → `tools.agentToAgent`
- `routing.transcribeAudio` → `tools.media.audio.models`
- `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `messages.tts.providers.<provider>`
- `channels.discord.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.voice.tts.providers.<provider>`
- `channels.discord.accounts.<id>.voice.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `channels.discord.accounts.<id>.voice.tts.providers.<provider>`
- `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`) → `plugins.entries.voice-call.config.tts.providers.<provider>`
- `plugins.entries.voice-call.config.provider: "log"` → `"mock"`
- `plugins.entries.voice-call.config.twilio.from` → `plugins.entries.voice-call.config.fromNumber`
- `plugins.entries.voice-call.config.streaming.sttProvider` → `plugins.entries.voice-call.config.streaming.provider`
- `plugins.entries.voice-call.config.streaming.openaiApiKey|sttModel|silenceDurationMs|vadThreshold`
  → `plugins.entries.voice-call.config.streaming.providers.openai.*`
- `bindings[].match.accountID` → `bindings[].match.accountId`
- 이름이 지정된 `accounts`가 있는 채널에서 단일 계정용 최상위 채널 값이 남아 있는 경우, 해당 채널에 대해 선택된 승격 계정으로 그 계정 범위 값을 이동(대부분의 채널은 `accounts.default`, Matrix는 기존의 일치하는 이름 있는/기본 대상을 유지할 수 있음)
- `identity` → `agents.list[].identity`
- `agent.*` → `agents.defaults` + `tools.*` (tools/elevated/exec/sandbox/subagents)
- `agent.model`/`allowedModels`/`modelAliases`/`modelFallbacks`/`imageModelFallbacks`
  → `agents.defaults.models` + `agents.defaults.model.primary/fallbacks` + `agents.defaults.imageModel.primary/fallbacks`
- `browser.ssrfPolicy.allowPrivateNetwork` → `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`
- `browser.profiles.*.driver: "extension"` → `"existing-session"`
- `browser.relayBindHost` 제거(레거시 extension relay 설정)

Doctor 경고에는 다중 계정 채널에 대한 기본 계정 안내도 포함됩니다:

- `channels.<channel>.accounts` 항목이 두 개 이상 구성되었는데 `channels.<channel>.defaultAccount` 또는 `accounts.default`가 없으면, doctor는 폴백 라우팅이 예상치 못한 계정을 선택할 수 있다고 경고합니다.
- `channels.<channel>.defaultAccount`가 알 수 없는 계정 ID로 설정되어 있으면, doctor는 경고하고 구성된 계정 ID 목록을 표시합니다.

### 2b) OpenCode provider override

`models.providers.opencode`, `opencode-zen`, 또는 `opencode-go`를 수동으로 추가한 경우
`@mariozechner/pi-ai`의 builtin OpenCode 카탈로그를 override하게 됩니다.
이로 인해 모델이 잘못된 API에 강제로 연결되거나 비용이 0으로 설정될 수 있습니다. Doctor는
override를 제거하고 모델별 API 라우팅 + 비용을 복원할 수 있도록 경고합니다.

### 2c) Browser 마이그레이션 및 Chrome MCP 준비 상태

browser config가 여전히 제거된 Chrome extension 경로를 가리키는 경우, doctor는
이를 현재의 호스트 로컬 Chrome MCP 연결 모델로 정규화합니다:

- `browser.profiles.*.driver: "extension"`은 `"existing-session"`이 됩니다
- `browser.relayBindHost`는 제거됩니다

또한 doctor는 `defaultProfile:
"user"` 또는 구성된 `existing-session` 프로필을 사용할 때 호스트 로컬 Chrome MCP 경로를 감사합니다:

- 기본 자동 연결 프로필에 대해 동일한 호스트에 Google Chrome이 설치되어 있는지 확인
- 감지된 Chrome 버전을 확인하고 Chrome 144 미만이면 경고
- browser inspect 페이지에서 원격 디버깅을 활성화하라고 안내
  (예: `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging`,
  또는 `edge://inspect/#remote-debugging`)

Doctor는 Chrome 측 설정을 대신 활성화할 수 없습니다. 호스트 로컬 Chrome MCP에는
여전히 다음이 필요합니다:

- gateway/node 호스트의 Chromium 기반 browser 144+
- 로컬에서 실행 중인 browser
- 해당 browser에서 원격 디버깅 활성화
- browser에서 최초 연결 동의 프롬프트 승인

여기서의 준비 상태는 로컬 연결 전제 조건에 관한 것입니다. Existing-session은
현재 Chrome MCP 경로 제한을 그대로 유지합니다. `responsebody`, PDF
내보내기, 다운로드 가로채기, 배치 작업 같은 고급 경로에는 여전히 관리형
browser 또는 raw CDP 프로필이 필요합니다.

이 점검은 Docker, sandbox, remote-browser 또는 기타
헤드리스 흐름에는 **적용되지 않습니다**. 그런 경우는 계속 raw CDP를 사용합니다.

### 2d) OAuth TLS 전제 조건

OpenAI Codex OAuth 프로필이 구성되어 있으면 doctor는 OpenAI
인가 엔드포인트를 프로브하여 로컬 Node/OpenSSL TLS 스택이 인증서 체인을
검증할 수 있는지 확인합니다. 인증서 오류(예:
`UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, 만료된 인증서, self-signed cert)로 프로브가 실패하면,
doctor는 플랫폼별 수정 안내를 출력합니다. Homebrew Node를 사용하는 macOS에서는
대개 `brew postinstall ca-certificates`가 해결책입니다. `--deep`를 사용하면
gateway가 정상이어도 프로브를 실행합니다.

### 3) 레거시 state 마이그레이션(디스크 레이아웃)

Doctor는 이전 디스크 레이아웃을 현재 구조로 마이그레이션할 수 있습니다:

- Sessions 저장소 + transcripts:
  - `~/.openclaw/sessions/`에서 `~/.openclaw/agents/<agentId>/sessions/`로
- Agent dir:
  - `~/.openclaw/agent/`에서 `~/.openclaw/agents/<agentId>/agent/`로
- WhatsApp auth state (Baileys):
  - 레거시 `~/.openclaw/credentials/*.json` (`oauth.json` 제외)에서
  - `~/.openclaw/credentials/whatsapp/<accountId>/...`로 (기본 계정 ID: `default`)

이 마이그레이션은 최선의 노력 기준이며 멱등적입니다. doctor는
백업으로 남겨 둔 레거시 폴더가 있으면 경고를 출력합니다. Gateway/CLI도 시작 시
레거시 sessions + agent dir를 자동 마이그레이션하므로 수동 doctor 실행 없이도
기록/auth/models가 에이전트별 경로로 이동합니다. WhatsApp auth는 의도적으로
`openclaw doctor`를 통해서만 마이그레이션됩니다. 이제 Talk provider/provider-map 정규화는
구조적 동등성으로 비교하므로 키 순서만 다른 차이로는 반복적인
no-op `doctor --fix` 변경이 더 이상 발생하지 않습니다.

### 3a) 레거시 plugin manifest 마이그레이션

Doctor는 설치된 모든 plugin manifest를 검사해 더 이상 사용되지 않는 최상위 capability
키(`speechProviders`, `realtimeTranscriptionProviders`,
`realtimeVoiceProviders`, `mediaUnderstandingProviders`,
`imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`,
`webSearchProviders`)를 찾습니다. 발견되면 이를 `contracts`
객체로 이동하고 manifest 파일을 제자리에서 다시 쓰도록 제안합니다. 이 마이그레이션은 멱등적입니다.
`contracts` 키에 이미 같은 값이 있으면 데이터를 중복하지 않고
레거시 키만 제거합니다.

### 3b) 레거시 cron 저장소 마이그레이션

Doctor는 또한 cron 작업 저장소(기본값 `~/.openclaw/cron/jobs.json`,
또는 override된 경우 `cron.store`)에서 스케줄러가 호환성을 위해 여전히
허용하는 이전 작업 형태를 점검합니다.

현재 cron 정리 항목은 다음과 같습니다:

- `jobId` → `id`
- `schedule.cron` → `schedule.expr`
- 최상위 payload 필드(`message`, `model`, `thinking`, ...) → `payload`
- 최상위 delivery 필드(`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
- payload `provider` delivery 별칭 → 명시적 `delivery.channel`
- 단순 레거시 `notify: true` webhook 폴백 작업 → `delivery.to=cron.webhook`를 사용하는 명시적 `delivery.mode="webhook"`

Doctor는 동작을 변경하지 않고 수행할 수 있을 때만 `notify: true` 작업을 자동 마이그레이션합니다.
작업이 레거시 notify 폴백과 기존의
non-webhook delivery 모드를 함께 사용하면 doctor는 경고하고 해당 작업은 수동 검토용으로 남겨 둡니다.

### 3c) 세션 lock 정리

Doctor는 모든 에이전트 세션 디렉터리를 검사해 오래된 쓰기 lock 파일을 찾습니다 —
세션이 비정상 종료될 때 남겨진 파일입니다. 발견된 각 lock 파일에 대해 다음을 보고합니다:
경로, PID, 해당 PID가 아직 살아 있는지, lock 경과 시간, 그리고
오래된 것으로 간주되는지 여부(죽은 PID 또는 30분 이상 경과). `--fix` / `--repair`
모드에서는 doctor가 오래된 lock 파일을 자동으로 제거합니다. 그렇지 않으면 메모를 출력하고
`--fix`로 다시 실행하라고 안내합니다.

### 4) State 무결성 점검(세션 지속성, 라우팅, 안전성)

state 디렉터리는 운영의 핵심입니다. 이것이 사라지면
sessions, credentials, logs, config를 잃게 됩니다(다른 곳에 백업이 없는 한).

Doctor는 다음을 점검합니다:

- **State dir 누락**: 치명적인 state 손실을 경고하고, 디렉터리 재생성을 제안하며,
  누락된 데이터를 복구할 수 없음을 상기시킵니다.
- **State dir 권한**: 쓰기 가능 여부를 확인하고, 권한 복구를 제안합니다
  (소유자/그룹 불일치가 감지되면 `chown` 힌트도 출력).
- **macOS 클라우드 동기화 state dir**: state가 iCloud Drive
  (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) 또는
  `~/Library/CloudStorage/...` 아래로 해석되면 경고합니다. 동기화 기반 경로는 I/O가 느려지고
  lock/동기화 경합을 유발할 수 있기 때문입니다.
- **Linux SD 또는 eMMC state dir**: state가 `mmcblk*`
  마운트 소스로 해석되면 경고합니다. SD 또는 eMMC 기반 랜덤 I/O는 세션 및 자격 증명 쓰기에서 더 느리고
  마모가 더 빠를 수 있기 때문입니다.
- **Session dir 누락**: `sessions/`와 세션 저장소 디렉터리는
  기록을 유지하고 `ENOENT` 충돌을 피하는 데 필요합니다.
- **Transcript 불일치**: 최근 session 항목에 누락된
  transcript 파일이 있으면 경고합니다.
- **기본 session “1줄 JSONL”**: 기본 transcript에 줄이 하나뿐인 경우
  (기록이 누적되지 않음) 이를 표시합니다.
- **여러 state dir**: 여러 홈 디렉터리에
  여러 `~/.openclaw` 폴더가 있거나 `OPENCLAW_STATE_DIR`이 다른 위치를 가리키면 경고합니다(기록이 설치 간에 분리될 수 있음).
- **원격 모드 알림**: `gateway.mode=remote`인 경우 doctor는
  원격 호스트에서 실행하라고 상기시킵니다(state는 그곳에 있음).
- **Config 파일 권한**: `~/.openclaw/openclaw.json`이
  그룹/전체 읽기 가능이면 경고하고 `600`으로 강화할 것을 제안합니다.

### 5) 모델 auth 상태(OAuth 만료)

Doctor는 auth 저장소의 OAuth 프로필을 검사하고, 토큰이
곧 만료되거나 이미 만료되었을 때 경고하며, 안전한 경우 갱신할 수 있습니다. Anthropic
OAuth/token 프로필이 오래되었으면 Anthropic API key 또는 레거시
Anthropic setup-token 경로를 제안합니다.
갱신 프롬프트는 대화형(TTY)으로 실행할 때만 표시되며, `--non-interactive`는
갱신 시도를 건너뜁니다.

Doctor는 또한 오래된 제거된 Anthropic Claude CLI state도 감지합니다. 이전
`anthropic:claude-cli` 자격 증명 바이트가 여전히 `auth-profiles.json`에 있으면,
doctor는 이를 Anthropic token/OAuth 프로필로 다시 변환하고 오래된
`claude-cli/...` 모델 ref를 다시 작성합니다.
바이트가 사라졌으면 doctor는 오래된 config를 제거하고 대신 복구
command를 출력합니다.

또한 doctor는 다음 때문에 일시적으로 사용할 수 없는 auth 프로필도 보고합니다:

- 짧은 cooldowns(속도 제한/시간 초과/auth 실패)
- 더 긴 비활성화(billing/credit 실패)

### 6) Hooks 모델 검증

`hooks.gmail.model`이 설정되어 있으면 doctor는 모델 ref를
카탈로그 및 allowlist와 대조해 검증하고, 해석되지 않거나 허용되지 않을 경우 경고합니다.

### 7) Sandbox image 복구

sandboxing이 활성화되어 있으면 doctor는 Docker image를 점검하고
현재 image가 없을 때 빌드하거나 레거시 이름으로 전환할 것을 제안합니다.

### 7b) 번들 plugin 런타임 종속성

Doctor는 번들 plugin 런타임 종속성(예: Discord plugin 런타임 패키지)이
OpenClaw 설치 루트에 존재하는지 확인합니다.
누락된 항목이 있으면 doctor는 패키지를 보고하고
`openclaw doctor --fix` / `openclaw doctor --repair` 모드에서 설치합니다.

### 8) Gateway 서비스 마이그레이션 및 정리 힌트

Doctor는 레거시 gateway 서비스(launchd/systemd/schtasks)를 감지하고
이를 제거하고 현재 gateway 포트를 사용하는 OpenClaw 서비스를 설치하도록 제안합니다.
추가 gateway 유사 서비스를 검사하고 정리 힌트를 출력할 수도 있습니다.
프로필 이름이 붙은 OpenClaw gateway 서비스는 정식으로 취급되며 "추가"로 표시되지 않습니다.

### 8b) 시작 Matrix 마이그레이션

Matrix 채널 계정에 보류 중이거나 실행 가능한 레거시 state 마이그레이션이 있으면,
doctor는(`--fix` / `--repair` 모드에서) 마이그레이션 전 스냅샷을 생성한 다음
최선의 노력 기준 마이그레이션 단계를 실행합니다: 레거시 Matrix state 마이그레이션과
레거시 encrypted-state 준비. 두 단계 모두 치명적이지 않으며, 오류는 기록되고
시작은 계속됩니다. 읽기 전용 모드(`--fix` 없는 `openclaw doctor`)에서는 이 점검을
완전히 건너뜁니다.

### 9) 보안 경고

Doctor는 allowlist 없이 provider가 DM에 열려 있거나
정책이 위험한 방식으로 구성된 경우 경고를 출력합니다.

### 10) systemd linger(Linux)

systemd 사용자 서비스로 실행 중이면 doctor는
로그아웃 후에도 gateway가 살아 있도록 lingering이 활성화되어 있는지 확인합니다.

### 11) Workspace 상태(Skills, plugins, 레거시 디렉터리)

Doctor는 기본 에이전트에 대한 workspace 상태 요약을 출력합니다:

- **Skills 상태**: 적격, 요구 사항 누락, allowlist 차단 Skills 수를 집계합니다.
- **레거시 workspace dir**: `~/openclaw` 또는 기타 레거시 workspace 디렉터리가
  현재 workspace와 함께 존재하면 경고합니다.
- **Plugin 상태**: 로드됨/비활성화됨/오류 plugin 수를 집계하고, 오류가 있는
  plugin의 plugin ID를 나열하며, 번들 plugin capability를 보고합니다.
- **Plugin 호환성 경고**: 현재 런타임과 호환성 문제가 있는
  plugin을 표시합니다.
- **Plugin 진단**: plugin registry가 로드 시 출력한 경고나 오류를
  표시합니다.

### 11b) Bootstrap 파일 크기

Doctor는 workspace bootstrap 파일(예: `AGENTS.md`,
`CLAUDE.md`, 또는 기타 주입된 컨텍스트 파일)이 구성된
문자 예산에 근접했거나 초과했는지 점검합니다. 파일별 원시 문자 수와 주입된 문자 수,
잘림 비율, 잘림 원인(`max/file` 또는 `max/total`), 총 예산 대비
총 주입 문자 수를 보고합니다. 파일이 잘렸거나 한계에 가까우면
doctor는 `agents.defaults.bootstrapMaxChars`
및 `agents.defaults.bootstrapTotalMaxChars` 조정을 위한 팁을 출력합니다.

### 11c) 셸 completion

Doctor는 현재 셸에 대해 탭 completion이 설치되어 있는지 점검합니다
(zsh, bash, fish, 또는 PowerShell):

- 셸 프로필이 느린 동적 completion 패턴을 사용하는 경우
  (`source <(openclaw completion ...)`), doctor는 이를 더 빠른
  캐시 파일 방식으로 업그레이드합니다.
- 프로필에 completion이 구성되어 있지만 캐시 파일이 없으면,
  doctor는 캐시를 자동으로 다시 생성합니다.
- completion이 전혀 구성되어 있지 않으면 doctor는 설치를 제안합니다
  (대화형 모드에서만, `--non-interactive`에서는 건너뜀).

캐시를 수동으로 다시 생성하려면 `openclaw completion --write-state`를 실행하세요.

### 12) Gateway auth 점검(로컬 토큰)

Doctor는 로컬 gateway 토큰 auth 준비 상태를 점검합니다.

- 토큰 모드에 토큰이 필요하고 토큰 소스가 없으면 doctor는 하나를 생성할지 제안합니다.
- `gateway.auth.token`이 SecretRef로 관리되지만 사용할 수 없으면 doctor는 경고하고 이를 평문으로 덮어쓰지 않습니다.
- `openclaw doctor --generate-gateway-token`은 토큰 SecretRef가 구성되지 않았을 때만 생성을 강제합니다.

### 12b) 읽기 전용 SecretRef 인식 복구

일부 복구 흐름은 런타임의 즉시 실패 동작을 약화시키지 않고 구성된 자격 증명을 검사해야 합니다.

- 이제 `openclaw doctor --fix`는 대상 config 복구를 위해 status 계열 command와 동일한 읽기 전용 SecretRef 요약 모델을 사용합니다.
- 예: Telegram `allowFrom` / `groupAllowFrom` `@username` 복구는 가능하면 구성된 bot 자격 증명을 사용하려고 시도합니다.
- Telegram bot 토큰이 SecretRef로 구성되었지만 현재 command 경로에서 사용할 수 없으면, doctor는 자격 증명이 구성되었지만 현재 사용할 수 없다고 보고하고 누락된 토큰으로 잘못 보고하거나 충돌하는 대신 자동 해결을 건너뜁니다.

### 13) Gateway 상태 점검 + restart

Doctor는 상태 점검을 실행하고 gateway가 비정상적으로 보이면
restart를 제안합니다.

### 13b) 메모리 검색 준비 상태

Doctor는 구성된 메모리 검색 임베딩 provider가 기본 에이전트에 대해 준비되었는지 점검합니다.
동작은 구성된 backend와 provider에 따라 달라집니다:

- **QMD backend**: `qmd` 바이너리가 사용 가능하고 시작 가능한지 프로브합니다.
  그렇지 않으면 npm 패키지와 수동 바이너리 경로 옵션을 포함한 수정 안내를 출력합니다.
- **명시적 로컬 provider**: 로컬 모델 파일 또는 인식 가능한
  원격/다운로드 가능한 모델 URL을 점검합니다. 없으면 원격 provider로 전환할 것을 제안합니다.
- **명시적 원격 provider** (`openai`, `voyage` 등): API key가
  환경 또는 auth 저장소에 있는지 확인합니다. 없으면 실행 가능한 수정 힌트를 출력합니다.
- **자동 provider**: 먼저 로컬 모델 가용성을 점검한 다음 자동 선택 순서대로 각 원격
  provider를 시도합니다.

gateway 프로브 결과를 사용할 수 있으면(gateway가 점검 시점에 정상이었을 때),
doctor는 이를 CLI에서 보이는 config와 교차 검증하고
불일치가 있으면 알려줍니다.

런타임에서 임베딩 준비 상태를 확인하려면 `openclaw memory status --deep`를 사용하세요.

### 14) 채널 상태 경고

gateway가 정상이면 doctor는 채널 상태 프로브를 실행하고
권장 수정과 함께 경고를 보고합니다.

### 15) Supervisor config 감사 + 복구

Doctor는 설치된 supervisor config(launchd/systemd/schtasks)를 점검해
누락되었거나 오래된 기본값(예: systemd network-online 종속성과
restart 지연)을 확인합니다. 불일치를 찾으면 업데이트를 권장하고
서비스 파일/작업을 현재 기본값으로 다시 쓸 수 있습니다.

참고:

- `openclaw doctor`는 supervisor config를 다시 쓰기 전에 프롬프트를 표시합니다.
- `openclaw doctor --yes`는 기본 복구 프롬프트를 수락합니다.
- `openclaw doctor --repair`는 프롬프트 없이 권장 수정을 적용합니다.
- `openclaw doctor --repair --force`는 사용자 지정 supervisor config를 덮어씁니다.
- 토큰 auth에 토큰이 필요하고 `gateway.auth.token`이 SecretRef로 관리되는 경우, doctor 서비스 설치/복구는 SecretRef를 검증하지만 해석된 평문 토큰 값을 supervisor 서비스 환경 메타데이터에 지속 저장하지 않습니다.
- 토큰 auth에 토큰이 필요하고 구성된 토큰 SecretRef가 해석되지 않으면, doctor는 실행 가능한 안내와 함께 설치/복구 경로를 차단합니다.
- `gateway.auth.token`과 `gateway.auth.password`가 모두 구성되어 있고 `gateway.auth.mode`가 설정되지 않은 경우, doctor는 mode가 명시적으로 설정될 때까지 설치/복구를 차단합니다.
- Linux user-systemd unit의 경우, doctor의 토큰 드리프트 점검은 이제 서비스 auth 메타데이터 비교 시 `Environment=`와 `EnvironmentFile=` 소스를 모두 포함합니다.
- 언제든 `openclaw gateway install --force`를 통해 전체 다시 쓰기를 강제할 수 있습니다.

### 16) Gateway 런타임 + 포트 진단

Doctor는 서비스 런타임(PID, 마지막 종료 상태)을 검사하고
서비스가 설치되어 있지만 실제로 실행 중이 아닐 때 경고합니다. 또한 gateway 포트
(기본값 `18789`)의 충돌 여부를 점검하고 가능한 원인(gateway가 이미 실행 중,
SSH 터널)을 보고합니다.

### 17) Gateway 런타임 모범 사례

Doctor는 gateway 서비스가 Bun 또는 버전 관리형 Node 경로
(`nvm`, `fnm`, `volta`, `asdf` 등)에서 실행될 때 경고합니다. WhatsApp + Telegram 채널에는 Node가 필요하며,
버전 관리 경로는 서비스가 셸 초기화를 로드하지 않기 때문에 업그레이드 후 깨질 수 있습니다.
Doctor는 사용 가능할 경우 시스템 Node 설치(Homebrew/apt/choco)로
마이그레이션할 것을 제안합니다.

### 18) Config 쓰기 + wizard 메타데이터

Doctor는 모든 config 변경 사항을 저장하고 doctor 실행을 기록하기 위해
wizard 메타데이터를 기록합니다.

### 19) Workspace 팁(백업 + 메모리 시스템)

Doctor는 누락된 경우 workspace 메모리 시스템을 제안하고 workspace가
이미 git 아래에 있지 않으면 백업 팁을 출력합니다.

workspace 구조와 git 백업(권장: 비공개 GitHub 또는 GitLab)에 대한 전체 가이드는
[/concepts/agent-workspace](/ko/concepts/agent-workspace)를 참조하세요.
