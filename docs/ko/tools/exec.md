---
read_when:
    - exec 도구를 사용하거나 수정할 때
    - stdin 또는 TTY 동작을 디버깅할 때
summary: Exec 도구 사용법, stdin 모드, TTY 지원
title: Exec Tool
x-i18n:
    generated_at: "2026-04-06T03:13:16Z"
    model: gpt-5.4
    provider: openai
    source_hash: 28388971c627292dba9bf65ae38d7af8cde49a33bb3b5fc8b20da4f0e350bedd
    source_path: tools/exec.md
    workflow: 15
---

# Exec Tool

작업공간에서 셸 명령을 실행합니다. `process`를 통한 포그라운드 + 백그라운드 실행을 지원합니다.
`process`가 허용되지 않으면 `exec`는 동기적으로 실행되며 `yieldMs`/`background`를 무시합니다.
백그라운드 세션은 에이전트별 범위로 제한되며, `process`는 같은 에이전트의 세션만 볼 수 있습니다.

## 파라미터

- `command`(필수)
- `workdir`(기본값: cwd)
- `env`(키/값 재정의)
- `yieldMs`(기본값 10000): 지연 후 자동 백그라운드 전환
- `background`(bool): 즉시 백그라운드 실행
- `timeout`(초, 기본값 1800): 만료 시 종료
- `pty`(bool): 가능할 때 pseudo-terminal로 실행(TTY 전용 CLI, 코딩 에이전트, 터미널 UI)
- `host`(`auto | sandbox | gateway | node`): 실행 위치
- `security`(`deny | allowlist | full`): `gateway`/`node`용 적용 모드
- `ask`(`off | on-miss | always`): `gateway`/`node`용 승인 프롬프트
- `node`(string): `host=node`용 node id/name
- `elevated`(bool): elevated 모드 요청(샌드박스에서 구성된 host 경로로 탈출); `security=full`은 elevated가 `full`로 해석될 때만 강제됩니다

참고:

- `host` 기본값은 `auto`입니다. 세션에서 샌드박스 런타임이 활성화되어 있으면 sandbox, 아니면 gateway입니다.
- `auto`는 기본 라우팅 전략이지 와일드카드가 아닙니다. 호출별 `host=node`는 `auto`에서 허용되지만, 호출별 `host=gateway`는 샌드박스 런타임이 활성화되어 있지 않을 때만 허용됩니다.
- 추가 구성 없이도 `host=auto`는 여전히 “그냥 동작”합니다. 샌드박스가 없으면 `gateway`로 해석되고, 활성 샌드박스가 있으면 샌드박스에 머뭅니다.
- `elevated`는 샌드박스에서 구성된 host 경로로 탈출합니다. 기본값은 `gateway`이며, `tools.exec.host=node`(또는 세션 기본값이 `host=node`)인 경우 `node`가 됩니다. 현재 세션/공급자에 대해 elevated 접근이 활성화된 경우에만 사용할 수 있습니다.
- `gateway`/`node` 승인은 `~/.openclaw/exec-approvals.json`으로 제어됩니다.
- `node`에는 페어링된 node(컴패니언 앱 또는 헤드리스 node host)가 필요합니다.
- 여러 node를 사용할 수 있는 경우 `exec.node` 또는 `tools.exec.node`를 설정해 하나를 선택하세요.
- `exec host=node`는 node용 유일한 셸 실행 경로이며, 레거시 `nodes.run` 래퍼는 제거되었습니다.
- Windows가 아닌 host에서는 exec가 설정된 경우 `SHELL`을 사용합니다. `SHELL`이 `fish`이면 fish와 호환되지 않는 스크립트를 피하기 위해 `PATH`의 `bash`(또는 `sh`)를 우선 사용하고, 둘 다 없을 때만 `SHELL`로 폴백합니다.
- Windows host에서는 exec가 PowerShell 7(`pwsh`) 탐색(Program Files, ProgramW6432, 그다음 PATH)을 우선 사용하고, 그다음 Windows PowerShell 5.1로 폴백합니다.
- Host 실행(`gateway`/`node`)은 바이너리 하이재킹이나 주입된 코드를 방지하기 위해 `env.PATH`와 로더 재정의(`LD_*`/`DYLD_*`)를 거부합니다.
- OpenClaw는 생성된 명령 환경(PTY 및 샌드박스 실행 포함)에 `OPENCLAW_SHELL=exec`를 설정하므로 셸/프로필 규칙이 exec-tool 컨텍스트를 감지할 수 있습니다.
- 중요: 샌드박싱은 기본적으로 **꺼져 있습니다**. 샌드박싱이 꺼져 있으면 암시적 `host=auto`는 `gateway`로 해석됩니다. 명시적 `host=sandbox`는 조용히 gateway host에서 실행되는 대신 여전히 실패 시 닫힘 방식으로 처리됩니다. 샌드박싱을 활성화하거나 승인과 함께 `host=gateway`를 사용하세요.
- 스크립트 사전 검사(일반적인 Python/Node 셸 문법 실수용)는 유효한 `workdir` 경계 안에 있는 파일만 검사합니다. 스크립트 경로가 `workdir` 밖으로 해석되면 해당 파일에 대한 사전 검사는 건너뜁니다.
- 지금 시작하는 장기 실행 작업은 한 번만 시작하고, 자동 완료 웨이크가 활성화되어 있다면 명령이 출력을 내거나 실패할 때의 자동 완료 웨이크를 활용하세요.
  로그, 상태, 입력, 개입이 필요할 때는 `process`를 사용하세요. sleep 루프, timeout 루프, 반복 폴링으로 스케줄링을 흉내 내지 마세요.
- 나중에 실행되거나 일정에 따라 실행되어야 하는 작업은 `exec`의 sleep/delay 패턴 대신 cron을 사용하세요.

## 구성

- `tools.exec.notifyOnExit`(기본값: true): true이면 백그라운드로 전환된 exec 세션이 종료 시 시스템 이벤트를 큐에 넣고 heartbeat를 요청합니다.
- `tools.exec.approvalRunningNoticeMs`(기본값: 10000): 승인 게이트된 exec가 이 시간보다 오래 실행되면 한 번의 “running” 알림을 보냅니다(0이면 비활성화).
- `tools.exec.host`(기본값: `auto`; 샌드박스 런타임이 활성화되어 있으면 `sandbox`, 아니면 `gateway`로 해석)
- `tools.exec.security`(기본값: sandbox는 `deny`, gateway + node는 설정되지 않은 경우 `full`)
- `tools.exec.ask`(기본값: `off`)
- 무승인 host exec가 gateway + node의 기본값입니다. 승인/allowlist 동작을 원하면 `tools.exec.*`와 host의 `~/.openclaw/exec-approvals.json`을 모두 더 엄격하게 설정하세요. 자세한 내용은 [Exec approvals](/ko/tools/exec-approvals#no-approval-yolo-mode)를 참고하세요.
- YOLO는 `host=auto`가 아니라 host 정책 기본값(`security=full`, `ask=off`)에서 옵니다. gateway 또는 node 라우팅을 강제하려면 `tools.exec.host`를 설정하거나 `/exec host=...`를 사용하세요.
- `security=full`과 `ask=off` 모드에서는 host exec가 구성된 정책을 직접 따르며, 추가적인 휴리스틱 명령 난독화 프리필터는 없습니다.
- `tools.exec.node`(기본값: 설정되지 않음)
- `tools.exec.strictInlineEval`(기본값: false): true이면 `python -c`, `node -e`, `ruby -e`, `perl -e`, `php -r`, `lua -e`, `osascript -e` 같은 인라인 인터프리터 eval 형식은 항상 명시적 승인이 필요합니다. `allow-always`는 여전히 무해한 인터프리터/스크립트 호출을 저장할 수 있지만, 인라인 eval 형식은 매번 계속 프롬프트됩니다.
- `tools.exec.pathPrepend`: exec 실행 시 `PATH` 앞에 붙일 디렉터리 목록(gateway + sandbox 전용)
- `tools.exec.safeBins`: 명시적 allowlist 항목 없이 실행할 수 있는 stdin 전용 안전 바이너리. 동작 세부 정보는 [Safe bins](/ko/tools/exec-approvals#safe-bins-stdin-only)를 참고하세요.
- `tools.exec.safeBinTrustedDirs`: `safeBins` 경로 검사에 대해 추가로 명시적으로 신뢰하는 디렉터리. `PATH` 항목은 자동으로 신뢰되지 않습니다. 내장 기본값은 `/bin`과 `/usr/bin`입니다.
- `tools.exec.safeBinProfiles`: 안전 바이너리별 선택적 사용자 지정 argv 정책(`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`)

예시:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### PATH 처리

- `host=gateway`: 로그인 셸의 `PATH`를 exec 환경에 병합합니다. host 실행에서는 `env.PATH` 재정의가 거부됩니다. daemon 자체는 여전히 최소한의 `PATH`로 실행됩니다:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
- `host=sandbox`: 컨테이너 내부에서 `sh -lc`(로그인 셸)로 실행되므로 `/etc/profile`이 `PATH`를 재설정할 수 있습니다.
  OpenClaw는 profile sourcing 후 내부 env var를 통해 `env.PATH`를 앞에 붙입니다(셸 보간 없음).
  `tools.exec.pathPrepend`도 여기 적용됩니다.
- `host=node`: 전달한 비차단 env 재정의만 node로 전송됩니다. host 실행에서는 `env.PATH` 재정의가 거부되며 node host에서는 무시됩니다. node에 추가 PATH 항목이 필요하다면 node host 서비스 환경(systemd/launchd)을 구성하거나 도구를 표준 위치에 설치하세요.

에이전트별 node 바인딩(config에서 agent list 인덱스 사용):

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

Control UI: Nodes 탭에는 동일한 설정을 위한 작은 “Exec node binding” 패널이 포함되어 있습니다.

## 세션 재정의(`/exec`)

`/exec`를 사용해 `host`, `security`, `ask`, `node`에 대한 **세션별** 기본값을 설정하세요.
현재 값을 보려면 인수 없이 `/exec`를 보내세요.

예시:

```
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

## 권한 부여 모델

`/exec`는 **권한 있는 발신자**에 대해서만 적용됩니다(채널 allowlist/페어링 + `commands.useAccessGroups`).
이는 **세션 상태만** 업데이트하며 config를 기록하지 않습니다. exec를 강제로 비활성화하려면 도구
정책으로 거부하세요(`tools.deny: ["exec"]` 또는 에이전트별). 명시적으로
`security=full`과 `ask=off`를 설정하지 않는 한 host 승인은 계속 적용됩니다.

## Exec approvals(컴패니언 앱 / node host)

샌드박스된 에이전트는 gateway 또는 node host에서 `exec`가 실행되기 전에 요청별 승인을 요구할 수 있습니다.
정책, allowlist, UI 흐름은 [Exec approvals](/ko/tools/exec-approvals)를 참고하세요.

승인이 필요할 때 exec 도구는
`status: "approval-pending"`과 승인 id를 즉시 반환합니다. 승인되거나(또는 거부/시간 초과되면)
Gateway는 시스템 이벤트(`Exec finished` / `Exec denied`)를 발생시킵니다. 명령이 여전히
`tools.exec.approvalRunningNoticeMs` 이후에도 실행 중이면 단일 `Exec running` 알림이 발생합니다.
기본 승인 카드/버튼이 있는 채널에서는 에이전트가 우선 해당
네이티브 UI를 사용해야 하며, 도구
결과가 채팅 승인을 사용할 수 없거나 수동 승인이
유일한 경로라고 명시적으로 말할 때만 수동 `/approve` 명령을 포함해야 합니다.

## Allowlist + safe bins

수동 allowlist 적용은 **해석된 바이너리 경로만** 일치시킵니다(베이스네임 일치 없음). `security=allowlist`일
때 셸 명령은 모든 파이프라인 세그먼트가 allowlist에 있거나 안전 바이너리인 경우에만
자동 허용됩니다. 체이닝(`;`, `&&`, `||`)과 리다이렉션은
최상위 세그먼트 각각이 allowlist를 만족하지 않으면 allowlist 모드에서 거부됩니다(안전 바이너리 포함).
리다이렉션은 여전히 지원되지 않습니다.
지속적인 `allow-always` 신뢰도 이 규칙을 우회하지 않습니다. 체인 명령은 여전히 모든
최상위 세그먼트가 일치해야 합니다.

`autoAllowSkills`는 exec approvals의 별도 편의 경로입니다. 수동 경로 allowlist 항목과는
동일하지 않습니다. 엄격한 명시적 신뢰가 필요하다면 `autoAllowSkills`를 비활성화하세요.

두 제어는 서로 다른 용도로 사용하세요.

- `tools.exec.safeBins`: 작은 stdin 전용 스트림 필터
- `tools.exec.safeBinTrustedDirs`: 안전 바이너리 실행 파일 경로를 위한 명시적 추가 신뢰 디렉터리
- `tools.exec.safeBinProfiles`: 사용자 지정 safe bin용 명시적 argv 정책
- allowlist: 실행 파일 경로에 대한 명시적 신뢰

`safeBins`를 일반적인 allowlist처럼 취급하지 마세요. 또한 인터프리터/런타임 바이너리(예: `python3`, `node`, `ruby`, `bash`)를 추가하지 마세요. 그런 바이너리가 필요하다면 명시적 allowlist 항목을 사용하고 승인 프롬프트는 활성화된 상태로 유지하세요.
`openclaw security audit`는 인터프리터/런타임 `safeBins` 항목에 명시적 프로필이 없으면 경고하며, `openclaw doctor --fix`는 누락된 사용자 지정 `safeBinProfiles` 항목을 스캐폴드할 수 있습니다.
`openclaw security audit`와 `openclaw doctor`는 `jq` 같은 광범위한 동작의 바이너리를 `safeBins`에 명시적으로 다시 추가할 때도 경고합니다.
인터프리터를 명시적으로 allowlist에 추가한다면 `tools.exec.strictInlineEval`을 활성화하여 인라인 코드 eval 형식이 계속 새 승인을 요구하도록 하세요.

전체 정책 세부 정보와 예시는 [Exec approvals](/ko/tools/exec-approvals#safe-bins-stdin-only) 및 [Safe bins versus allowlist](/ko/tools/exec-approvals#safe-bins-versus-allowlist)를 참고하세요.

## 예시

포그라운드:

```json
{ "tool": "exec", "command": "ls -la" }
```

백그라운드 + 폴링:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

폴링은 대기 루프가 아니라 필요할 때의 상태 확인용입니다. 자동 완료 웨이크가
활성화되어 있다면, 명령은 출력을 내거나 실패할 때 세션을 깨울 수 있습니다.

키 전송(tmux 스타일):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Submit(CR만 전송):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Paste(기본적으로 bracketed):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch`는 구조화된 다중 파일 편집을 위한 `exec`의 하위 도구입니다.
기본적으로 OpenAI 및 OpenAI Codex 모델에서 활성화됩니다. 비활성화하거나 특정 모델로 제한하려는 경우에만 config를 사용하세요.

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.4"] },
    },
  },
}
```

참고:

- OpenAI/OpenAI Codex 모델에서만 사용할 수 있습니다.
- 도구 정책은 여전히 적용됩니다. `allow: ["write"]`는 암묵적으로 `apply_patch`를 허용합니다.
- config는 `tools.exec.applyPatch` 아래에 있습니다.
- `tools.exec.applyPatch.enabled`의 기본값은 `true`이며, OpenAI 모델에서 도구를 비활성화하려면 `false`로 설정하세요.
- `tools.exec.applyPatch.workspaceOnly`의 기본값은 `true`(작업공간 내부만)입니다. `apply_patch`가 작업공간 디렉터리 밖에 쓰기/삭제하도록 의도한 경우에만 `false`로 설정하세요.

## 관련 항목

- [Exec Approvals](/ko/tools/exec-approvals) — 셸 명령용 승인 게이트
- [Sandboxing](/ko/gateway/sandboxing) — 샌드박스 환경에서 명령 실행
- [Background Process](/ko/gateway/background-process) — 장기 실행 exec 및 process 도구
- [Security](/ko/gateway/security) — 도구 정책 및 elevated 접근
