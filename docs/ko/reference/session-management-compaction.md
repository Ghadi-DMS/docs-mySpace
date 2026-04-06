---
read_when:
    - 세션 ID, transcript JSONL, 또는 sessions.json 필드를 디버그해야 합니다
    - auto-compaction 동작을 변경하거나 “pre-compaction” 하우스키핑을 추가하고 있습니다
    - 메모리 flush 또는 조용한 시스템 턴을 구현하려고 합니다
summary: '심층 분석: 세션 저장소 + transcripts, 수명 주기, 그리고 (자동) compaction 내부 구조'
title: 세션 관리 심층 분석
x-i18n:
    generated_at: "2026-04-06T03:12:30Z"
    model: gpt-5.4
    provider: openai
    source_hash: e0d8c2d30be773eac0424f7a4419ab055fdd50daac8bc654e7d250c891f2c3b8
    source_path: reference/session-management-compaction.md
    workflow: 15
---

# 세션 관리 및 Compaction(심층 분석)

이 문서는 OpenClaw가 세션을 엔드 투 엔드로 관리하는 방식을 설명합니다:

- **세션 라우팅** (`sessionKey`에 인바운드 메시지가 어떻게 매핑되는지)
- **세션 저장소** (`sessions.json`)와 추적하는 항목
- **Transcript 지속성** (`*.jsonl`)과 그 구조
- **Transcript 위생** (실행 전 provider별 보정)
- **컨텍스트 한계** (컨텍스트 윈도우 대 추적된 토큰)
- **Compaction** (수동 + auto-compaction)과 pre-compaction 작업을 연결할 위치
- **조용한 하우스키핑** (예: 사용자에게 보이는 출력을 만들지 않아야 하는 메모리 쓰기)

먼저 더 높은 수준의 개요가 필요하다면 다음부터 시작하세요:

- [/concepts/session](/ko/concepts/session)
- [/concepts/compaction](/ko/concepts/compaction)
- [/concepts/memory](/ko/concepts/memory)
- [/concepts/memory-search](/ko/concepts/memory-search)
- [/concepts/session-pruning](/ko/concepts/session-pruning)
- [/reference/transcript-hygiene](/ko/reference/transcript-hygiene)

---

## 단일 진실 공급원: Gateway

OpenClaw는 세션 state를 소유하는 단일 **Gateway 프로세스**를 중심으로 설계되었습니다.

- UI(macOS 앱, 웹 Control UI, TUI)는 Gateway에 세션 목록과 토큰 수를 조회해야 합니다.
- 원격 모드에서는 세션 파일이 원격 호스트에 있으므로 “로컬 Mac 파일 확인”은 Gateway가 실제로 사용하는 내용을 반영하지 않습니다.

---

## 두 가지 지속성 계층

OpenClaw는 세션을 두 계층으로 저장합니다:

1. **세션 저장소 (`sessions.json`)**
   - 키/값 맵: `sessionKey -> SessionEntry`
   - 작고, 변경 가능하며, 편집(또는 항목 삭제)해도 안전함
   - 세션 메타데이터(현재 세션 ID, 마지막 활동, 토글, 토큰 카운터 등)를 추적함

2. **Transcript (`<sessionId>.jsonl`)**
   - 트리 구조를 가진 append-only transcript(항목에는 `id` + `parentId`가 있음)
   - 실제 대화 + tool 호출 + compaction 요약을 저장함
   - 향후 턴의 모델 컨텍스트를 재구성하는 데 사용됨

---

## 디스크 상 위치

Gateway 호스트에서, 에이전트별:

- 저장소: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- Transcripts: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
  - Telegram topic 세션: `.../<sessionId>-topic-<threadId>.jsonl`

OpenClaw는 `src/config/sessions.ts`를 통해 이를 해석합니다.

---

## 저장소 유지 관리와 디스크 제어

세션 지속성에는 `sessions.json`과 transcript 아티팩트를 위한 자동 유지 관리 제어(`session.maintenance`)가 있습니다:

- `mode`: `warn`(기본값) 또는 `enforce`
- `pruneAfter`: 오래된 항목의 보관 기간 기준(기본값 `30d`)
- `maxEntries`: `sessions.json` 항목 수 상한(기본값 `500`)
- `rotateBytes`: `sessions.json`이 너무 커지면 회전(기본값 `10mb`)
- `resetArchiveRetention`: `*.reset.<timestamp>` transcript 아카이브의 보관 기간(기본값: `pruneAfter`와 동일, `false`면 정리 비활성화)
- `maxDiskBytes`: 선택적 sessions 디렉터리 예산
- `highWaterBytes`: 정리 후 목표값(기본값 `maxDiskBytes`의 `80%`)

디스크 예산 정리의 적용 순서(`mode: "enforce"`):

1. 가장 오래된 보관된 transcript 또는 orphan 아티팩트를 먼저 제거합니다.
2. 그래도 목표를 초과하면 가장 오래된 세션 항목과 해당 transcript 파일을 제거합니다.
3. 사용량이 `highWaterBytes` 이하가 될 때까지 계속합니다.

`mode: "warn"`에서는 OpenClaw가 가능한 제거를 보고만 하고 저장소/파일은 변경하지 않습니다.

필요할 때 유지 관리를 실행하세요:

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

---

## Cron 세션과 실행 로그

격리된 cron 실행도 세션 항목/transcripts를 생성하며, 전용 보관 제어가 있습니다:

- `cron.sessionRetention`(기본값 `24h`)은 세션 저장소에서 오래된 격리 cron 실행 세션을 정리합니다(`false`면 비활성화).
- `cron.runLog.maxBytes` + `cron.runLog.keepLines`는 `~/.openclaw/cron/runs/<jobId>.jsonl` 파일을 정리합니다(기본값: `2_000_000` 바이트와 `2000`줄).

---

## 세션 키(`sessionKey`)

`sessionKey`는 _어떤 대화 버킷_ 에 있는지를 식별합니다(라우팅 + 격리).

일반적인 패턴:

- 메인/direct chat(에이전트별): `agent:<agentId>:<mainKey>` (기본값 `main`)
- 그룹: `agent:<agentId>:<channel>:group:<id>`
- Room/channel (Discord/Slack): `agent:<agentId>:<channel>:channel:<id>` 또는 `...:room:<id>`
- Cron: `cron:<job.id>`
- Webhook: `hook:<uuid>` (override되지 않은 경우)

정식 규칙은 [/concepts/session](/ko/concepts/session)에 문서화되어 있습니다.

---

## 세션 ID(`sessionId`)

각 `sessionKey`는 현재 `sessionId`를 가리킵니다(대화를 이어가는 transcript 파일).

경험칙:

- **Reset** (`/new`, `/reset`)은 해당 `sessionKey`에 대해 새 `sessionId`를 생성합니다.
- **일일 reset** (기본값: gateway 호스트 로컬 시간 기준 오전 4:00)는 reset 경계를 지난 뒤 다음 메시지에서 새 `sessionId`를 생성합니다.
- **유휴 만료** (`session.reset.idleMinutes` 또는 레거시 `session.idleMinutes`)는 유휴 창 이후 메시지가 도착하면 새 `sessionId`를 생성합니다. 일일 + 유휴가 둘 다 구성되면 먼저 만료되는 쪽이 우선합니다.
- **스레드 부모 포크 가드** (`session.parentForkMaxTokens`, 기본값 `100000`)는 부모 세션이 이미 너무 클 때 부모 transcript 포킹을 건너뜁니다. 새 스레드는 새로 시작합니다. 비활성화하려면 `0`으로 설정하세요.

구현 세부 사항: 결정은 `src/auto-reply/reply/session.ts`의 `initSessionState()`에서 이루어집니다.

---

## 세션 저장소 스키마 (`sessions.json`)

저장소 값 타입은 `src/config/sessions.ts`의 `SessionEntry`입니다.

주요 필드(전체 목록은 아님):

- `sessionId`: 현재 transcript ID(`sessionFile`이 설정되지 않은 한 파일명은 여기서 파생됨)
- `updatedAt`: 마지막 활동 타임스탬프
- `sessionFile`: 선택적 명시적 transcript 경로 override
- `chatType`: `direct | group | room` (UI와 전송 정책에 도움)
- `provider`, `subject`, `room`, `space`, `displayName`: 그룹/채널 라벨링용 메타데이터
- 토글:
  - `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`
  - `sendPolicy` (세션별 override)
- 모델 선택:
  - `providerOverride`, `modelOverride`, `authProfileOverride`
- 토큰 카운터(최선의 노력 / provider 종속):
  - `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
- `compactionCount`: 이 세션 키에서 auto-compaction이 완료된 횟수
- `memoryFlushAt`: 마지막 pre-compaction 메모리 flush 타임스탬프
- `memoryFlushCompactionCount`: 마지막 flush가 실행되었을 때의 compaction 횟수

저장소는 편집해도 안전하지만, 권한은 Gateway에 있습니다. 세션이 실행되면서 항목을 다시 쓰거나 다시 채울 수 있습니다.

---

## Transcript 구조 (`*.jsonl`)

transcript는 `@mariozechner/pi-coding-agent`의 `SessionManager`가 관리합니다.

파일 형식은 JSONL입니다:

- 첫 줄: 세션 헤더(`type: "session"`, `id`, `cwd`, `timestamp`, 선택적 `parentSession` 포함)
- 이후: `id` + `parentId`를 가진 세션 항목들(트리)

주목할 만한 항목 타입:

- `message`: user/assistant/toolResult 메시지
- `custom_message`: 모델 컨텍스트에 _포함되는_ extension 주입 메시지(UI에서 숨길 수 있음)
- `custom`: 모델 컨텍스트에 _포함되지 않는_ extension state
- `compaction`: `firstKeptEntryId` 및 `tokensBefore`를 가진 저장된 compaction 요약
- `branch_summary`: 트리 브랜치를 탐색할 때 저장되는 요약

OpenClaw는 의도적으로 transcript를 “보정”하지 않습니다. Gateway는 `SessionManager`를 사용해 이를 읽고 씁니다.

---

## 컨텍스트 윈도우 대 추적된 토큰

두 가지 다른 개념이 중요합니다:

1. **모델 컨텍스트 윈도우**: 모델별 하드 한도(모델에 보이는 토큰)
2. **세션 저장소 카운터**: `sessions.json`에 기록되는 롤링 통계(`/status` 및 대시보드에 사용)

한계를 조정하고 있다면:

- 컨텍스트 윈도우는 모델 카탈로그에서 오며(config로 override 가능)
- 저장소의 `contextTokens`는 런타임 추정/보고 값이므로 엄격한 보장으로 취급하지 마세요

자세한 내용은 [/token-use](/ko/reference/token-use)를 참조하세요.

---

## Compaction: 무엇인가

Compaction은 오래된 대화를 transcript에 저장된 `compaction` 항목으로 요약하고 최근 메시지는 그대로 유지합니다.

compaction 후 향후 턴에는 다음이 보입니다:

- compaction 요약
- `firstKeptEntryId` 이후의 메시지

Compaction은 **영속적**입니다(세션 pruning과 다름). [/concepts/session-pruning](/ko/concepts/session-pruning)을 참조하세요.

## Compaction 청크 경계와 tool 페어링

OpenClaw가 긴 transcript를 compaction 청크로 나눌 때,
assistant tool 호출과 그에 대응하는 `toolResult` 항목을 함께 유지합니다.

- 토큰 분할 지점이 tool 호출과 결과 사이에 걸리면, OpenClaw는
  페어를 분리하는 대신 경계를 assistant tool-call 메시지로 이동합니다.
- 뒤따르는 tool-result 블록 때문에 청크가 목표를 초과하게 되더라도,
  OpenClaw는 해당 대기 중 tool 블록을 보존하고 요약되지 않은 꼬리 부분을 그대로 유지합니다.
- 중단되었거나 오류가 있는 tool-call 블록은 대기 중인 분할을 계속 열어 두지 않습니다.

---

## auto-compaction이 발생하는 시점(Pi 런타임)

내장 Pi agent에서 auto-compaction은 두 경우에 트리거됩니다:

1. **오버플로 복구**: 모델이 컨텍스트 오버플로 오류를 반환하는 경우
   (`request_too_large`, `context length exceeded`, `input exceeds the maximum
number of tokens`, `input token count exceeds the maximum number of input
tokens`, `input is too long for the model`, `ollama error: context length
exceeded`, 그리고 유사한 provider 형태 변형) → compact → 재시도.
2. **임계값 유지 관리**: 성공적인 턴 후 다음일 때:

`contextTokens > contextWindow - reserveTokens`

여기서:

- `contextWindow`는 모델의 컨텍스트 윈도우
- `reserveTokens`는 프롬프트 + 다음 모델 출력용으로 예약된 헤드룸

이것은 Pi 런타임 의미론입니다(OpenClaw는 이벤트를 소비하지만, compact 시점 결정은 Pi가 합니다).

---

## Compaction 설정 (`reserveTokens`, `keepRecentTokens`)

Pi의 compaction 설정은 Pi settings에 있습니다:

```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,
    keepRecentTokens: 20000,
  },
}
```

OpenClaw는 내장 실행에 대해 안전 하한도 적용합니다:

- `compaction.reserveTokens < reserveTokensFloor`이면 OpenClaw가 값을 올립니다.
- 기본 하한은 `20000` 토큰입니다.
- 하한을 비활성화하려면 `agents.defaults.compaction.reserveTokensFloor: 0`으로 설정하세요.
- 이미 더 높으면 OpenClaw는 그대로 둡니다.

이유: compaction이 불가피해지기 전에 메모리 쓰기 같은 다중 턴 “하우스키핑”을 위한 충분한 헤드룸을 남기기 위함입니다.

구현: `src/agents/pi-settings.ts`의 `ensurePiCompactionReserveTokens()`
(`src/agents/pi-embedded-runner.ts`에서 호출).

---

## 사용자에게 보이는 표면

다음을 통해 compaction과 세션 state를 관찰할 수 있습니다:

- `/status` (모든 채팅 세션에서)
- `openclaw status` (CLI)
- `openclaw sessions` / `sessions --json`
- 상세 모드: `🧹 Auto-compaction complete` + compaction 횟수

---

## 조용한 하우스키핑 (`NO_REPLY`)

OpenClaw는 사용자가 중간 출력을 보지 않아야 하는 백그라운드 작업을 위해 “조용한” 턴을 지원합니다.

규칙:

- assistant는 정확한 조용한 토큰 `NO_REPLY` /
  `no_reply`로 출력을 시작해 “사용자에게 응답을 전달하지 말라”는 뜻을 나타냅니다.
- OpenClaw는 전달 계층에서 이를 제거/억제합니다.
- 정확한 조용한 토큰 억제는 대소문자를 구분하지 않으므로, 전체 payload가 조용한 토큰 하나뿐일 때 `NO_REPLY`와
  `no_reply`가 모두 인정됩니다.
- 이것은 진정한 백그라운드/비전달 턴에만 사용하기 위한 것이며, 일반적인 실행 가능한 사용자 요청의 지름길이 아닙니다.

`2026.1.10`부터 OpenClaw는 부분 청크가 `NO_REPLY`로 시작할 때 **draft/typing streaming**도 억제하므로,
조용한 작업이 턴 중간에 부분 출력을 유출하지 않습니다.

---

## Pre-compaction "memory flush"(구현됨)

목표: auto-compaction이 발생하기 전에 영속적인
state를 디스크에 쓰는 조용한 agent 턴(예: 에이전트 workspace의 `memory/YYYY-MM-DD.md`)을 실행하여
compaction이 중요한 컨텍스트를 지우지 못하게 합니다.

OpenClaw는 **사전 임계값 flush** 접근 방식을 사용합니다:

1. 세션 컨텍스트 사용량을 모니터링합니다.
2. “소프트 임계값”(Pi의 compaction 임계값보다 낮은 값)을 넘으면 agent에 조용한
   “지금 메모리를 기록하라” 지시를 실행합니다.
3. 정확한 조용한 토큰 `NO_REPLY` / `no_reply`를 사용하여 사용자가
   아무것도 보지 않게 합니다.

구성(`agents.defaults.compaction.memoryFlush`):

- `enabled` (기본값: `true`)
- `softThresholdTokens` (기본값: `4000`)
- `prompt` (flush 턴용 user 메시지)
- `systemPrompt` (flush 턴에 추가되는 추가 system prompt)

참고:

- 기본 prompt/system prompt에는 전달 억제를 위한 `NO_REPLY` 힌트가 포함되어 있습니다.
- flush는 compaction 주기당 한 번만 실행됩니다(`sessions.json`에서 추적).
- flush는 내장 Pi 세션에서만 실행됩니다.
- 세션 workspace가 읽기 전용일 때(`workspaceAccess: "ro"` 또는 `"none"`)는 flush를 건너뜁니다.
- workspace 파일 레이아웃과 쓰기 패턴은 [Memory](/ko/concepts/memory)를 참조하세요.

Pi는 extension API에서 `session_before_compact` hook도 노출하지만, OpenClaw의
flush 로직은 현재 Gateway 측에 있습니다.

---

## 문제 해결 체크리스트

- 세션 키가 잘못되었나요? [/concepts/session](/ko/concepts/session)부터 시작하고 `/status`의 `sessionKey`를 확인하세요.
- 저장소와 transcript가 일치하지 않나요? Gateway 호스트와 `openclaw status`의 저장소 경로를 확인하세요.
- compaction이 너무 자주 발생하나요? 다음을 확인하세요:
  - 모델 컨텍스트 윈도우(너무 작음)
  - compaction 설정(모델 윈도우에 비해 `reserveTokens`가 너무 크면 더 이른 compaction이 발생할 수 있음)
  - tool-result 비대화: 세션 pruning을 활성화/조정하세요
- 조용한 턴이 유출되나요? 응답이 `NO_REPLY`로 시작하는지(대소문자 구분 없는 정확한 토큰)와 streaming 억제 수정이 포함된 빌드인지 확인하세요.
